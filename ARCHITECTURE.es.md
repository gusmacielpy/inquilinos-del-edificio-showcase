# Arquitectura — Inquilinos

> Parafraseado de la implementación privada. Los snippets están acortados para claridad. Los términos de dominio se preservan en español donde aportan significado de negocio.

---

## 1. Estado de contrato derivado (nunca almacenado)

El estado no es una columna; es una función pura de fechas y un umbral a nivel edificio.

```ts
type Estado = "finalizado" | "renovado" | "sin_datos" | "pendiente_firma"
            | "vencido" | "por_vencer" | "vigente";

function estadoContrato(c: Contrato, hoy: Date, umbralGlobal: number): Estado {
  if (c.finalizado_en) return "finalizado";
  if (c.sucesor_contrato_id) return "renovado";
  if (!c.fecha_inicio_contrato && !c.fecha_entrada) return "sin_datos";
  if (!c.fecha_inicio_contrato) return "pendiente_firma";

  const umbral = c.umbral_alerta_dias ?? umbralGlobal; // default 60
  const diasAlFin = differenceInDays(c.fecha_fin_contrato!, hoy);

  if (diasAlFin < 0) return "vencido";
  if (diasAlFin <= umbral) return "por_vencer";
  return "vigente";
}
```

`sin_datos` existe para cargas históricas donde no se conoce ni entrada ni firma — se muestra como alerta en el dashboard para no confundirlo con un `pendiente_firma` real.
`vencido` muestra "Vencido hace N días" y sigue siendo cerrable.

Helpers: `diasVencidoContrato`, `diasSinFirmar`, `estadoContratoLabel`, `estadoContratoBadgeClass`.

---

## 2. Aritmética de periodos (por contrato)

Cada contrato tiene su propio ciclo anclado a `dia_pago_acordado` (D). No existe un "cierre global del edificio".

```ts
function periodoActual(diaVenc: number, hoy: Date) {
  const d = hoy.getDate();
  const inicio = d >= diaVenc
    ? new Date(hoy.getFullYear(), hoy.getMonth(), diaVenc)
    : new Date(hoy.getFullYear(), hoy.getMonth() - 1, diaVenc);
  const fin = subDays(addMonths(inicio, 1), 1);
  return { inicio, fin };
}
```

Caso borde: `diaVenc = 31` en febrero → `diaValidoDelMes` lo ajusta al último día real del mes. Helper `periodosEntre(entrada, tope)` recorre el historial con guarda de 400 iteraciones; `mesCalendarioDelPeriodo` hace de puente entre ciclos por contrato y registros por mes calendario (agua/ANDE se cargan por mes calendario).

---

## 3. Mora (late fee)

Simple, pero con una regla importante de congelamiento:

```ts
function moraAlquiler({ moraDiaria, saldoAlquiler, fechaLimiteSinMora, fechaRef }): bigint {
  if (!moraDiaria) return 0n;                 // nullable = sin mora
  if (saldoAlquiler <= 0) return 0n;          // se congela en 0 al saldar
  const limite = addDays(periodo.inicio, dias_gracia);
  const diasAtraso = Math.max(0, differenceInDays(fechaRef, limite));
  return BigInt(diasAtraso) * moraDiaria;
}
```

`fechaRef` = `hoy` en preview, `fecha_cierre` cuando se congela el snapshot. Mientras el periodo está abierto la mora se recalcula en vivo; una vez cerrado queda fija. `fechaLimiteSinMora` ya contempla meses más cortos que `dia_pago_acordado + dias_gracia`.

---

## 4. Prorrateo de agua

Tres modos, todos excluyen `unidad.tipo = servicio` (medidores de áreas comunes como el garage) y unidades no `ocupada`:

```ts
function prorratearAgua({ montoTotal, metodo, unidadesOcupadas, factoresPonderacion, consumos }) {
  switch (metodo) {
    case "partes_iguales":     return dividir(montoTotal, unidadesOcupadas.length);
    case "ponderado": {        // ej. grande=1.5, chico=1.0, salon=1.0
      const pesos = unidadesOcupadas.map(u => factoresPonderacion[u.tipo]);
      const total = sum(pesos);
      return unidadesOcupadas.map((u,i) => redondear(montoTotal * pesos[i] / total));
    }
    case "medidor_individual": {
      const total = sum(consumos.map(c => c.consumo));
      return unidadesOcupadas.map(u => {
        const c = consumos.find(x => x.unidad_codigo === u.codigo)?.consumo ?? 0;
        return redondear(montoTotal * c / total);
      });
    }
  }
  // ajuste determinístico por redondeo: el remanente va a la última unidad por código
}
```

El flag `prorratear_todas_unidades` en `servicio_agua_mes` permite incluir todas las unidades en un mes dado en vez de solo `ocupada` (útil para facturas a nivel edificio). El recálculo en vivo advierte que el prorrateo histórico usa la ocupación actual, por lo que puede variar si hubo altas/bajas — documentado como aproximación de v1.

---

## 5. Composición de la liquidación (`construirSumario`)

La liquidación es un único ensamblado puro sobre datos vivos:

```ts
async function construirSumario(supabase, contratoId, { mes, fechaRef }): Promise<Sumario> {
  const datos = await obtenerDatosContrato(supabase, contratoId); // 9 selects paralelos + fanouts
  // 1) mesIngreso = fecha_inicio_contrato ?? fecha_entrada
  // 2) deudaArrastrada = meses [mesIngreso .. mes-1] con saldos positivos
  // 3) ciclos huérfanos agua/energía vía calcularCiclos*Pendientes (cubre contratos sin_datos)
  // 4) deudaInicial vía calcularSaldosDeudaInicial
  // 5) cargos del mes + cuotas pendientes del mes + pendientes abiertos + pagosDelPeriodo
  // 6) totalACobrar = deudaArrastrada + cargosActuales + cuotasPendientes
  return { encabezado, contrato, cargos, cuotasPendientes, deudaArrastrada, pendientes, pagosDelPeriodo, totalACobrar };
}
```

`obtenerDatosContrato` hace fanout a `factura_ande`, `contrato_pendiente`, `plan_inicial+cuota_inicial`, `deuda_inicial`, `pago`, `unidad(ocupada|todas)`, `servicio_agua_mes`, `configuracion(factores_ponderacion_agua)`, luego `pago_aplicado`, `exclusion_saldo`, `consumo_agua_unidad`.

Tipo de retorno:

```ts
interface Sumario {
  contratoId: string;
  encabezado: { inquilinos: string[]; unidad: string; mesLabel: string; periodoInicio: string; periodoFin: string; fechaGeneracion: string; cerradoEn: string|null };
  contrato: { estado: string; montoAlquiler: number; diaPagoAcordado: number };
  cargos: CargoSumario[];              // alquiler (+ todo incluido), estacionamiento, basura, agua, energia, mora
  cuotasPendientes: CuotaPendienteSumario[]; // vencidas o con vencimiento este mes
  deudaArrastrada: DeudaArrastradaItem[];    // meses previos con saldo
  pendientes: PendienteSumario[];       // todos abiertos a nivel contrato
  pagosDelPeriodo: PagoSumario[];       // por fecha de pago dentro del mes
  totalACobrar: number;
}
```

Apuesta de diseño: la liquidación **siempre se calcula en vivo**, aunque exista un reporte `mes_cerrado` para ese mes. El dashboard la usa como superficie de monitoreo; el registro congelado se consulta aparte en `/cierres/YYYY-MM` como historia. Así se mantiene separada la verdad viva de la verdad auditada.

---

## 6. `cargo_fijo` — override todo incluido

Un flag de acuerdo comercial que colapsa los cargos recurrentes en una sola línea:

```ts
if (c.cargo_fijo_activo && c.cargo_fijo_monto !== null) {
  return {
    alquiler: c.cargo_fijo_monto,
    estacionamiento: 0, basura: 0, agua: 0, energia: 0,
    cicloAgua: cicloAgua, cicloEnergia: cicloEnergia, // se retienen para trazabilidad
  };
}
```

La deuda y las cuotas iniciales igual aplican por encima (son cargos únicos de arranque). La liquidación renderiza una sola fila `Alquiler (todo incluido)` y saltea las filas separadas de estacionamiento/basura/agua/energía. Los pagos contra ese cargo usan `cargo_tipo="alquiler"` con la key del mes.

---

## 7. Perdón del mes de ingreso (`cobra_*_ingreso`)

Tres toggles por contrato (`cobra_alquiler_ingreso`, `cobra_basura_ingreso`, `cobra_estacionamiento_ingreso`) resuelven el caso de ingreso a mitad de mes de forma explícita: la spec **no prorratea** un mes parcial de forma deliberada.

```ts
function cargosBrutosDelMes(datos, mes, { esMesDeIngreso, alquilerCubiertoPorPlan }) {
  if (alquilerCubiertoPorPlan) return { alquiler: 0, detalle: "Cubierto por plan Primer alquiler" };
  const alquilerPerdonado = esMesDeIngreso && !c.cobra_alquiler_ingreso;
  return { alquiler: alquilerPerdonado ? 0 : c.monto_alquiler, … };
}
```

Si existe plan `primer_alquiler`, tiene precedencia sobre el flag de perdón — se espera que el alquiler del mes de ingreso esté cubierto por las cuotas de ese plan.

Los ciclos de agua/energía etiquetados `≤ mesDeIngreso` también se saltean, ya que una key `YYYY-MM-01` representa consumo del mes previo.

---

## 8. `pago` vs `exclusion_saldo` (sin dinero flotando)

Cada guaraní está contabilizado en uno de dos ledgers:

- **`pago` + `pago_aplicado`**: dinero real recibido. Una fila `pago` tiene `fecha, monto, metodo`; su distribución se guarda en `pago_aplicado(cargo_tipo, periodo, monto)` o `cuota_inicial(cargo_ref)`. Crear un pago inserta atómicamente `pago` + N `pago_aplicado`; si la suma de aplicaciones es menor que `pago.monto`, el resto va automáticamente a `saldo_favor` para que nunca quede dinero sin asignar; si excede, la action rechaza. Rollback en fallo de `pago_aplicado` borra el `pago` padre.

- **`exclusion_saldo`**: ajuste manual que reduce un saldo sin ser dinero recibido (ej. perdón negociado, corrección de facturación). Misma forma `cargo_tipo + periodo + monto`, más `excluir_total` que fuerza el saldo a 0 sin importar el bruto actual (importante cuando los totales de prorrateo se mueven). Se mantiene aparte para que el historial de pagos quede como ledger limpio de "dinero recibido".

Ambos ledgers se consultan juntos en `saldosDelMes`:

```ts
const saldo = excluidoTotal ? 0
  : Math.max(0, bruto - pagadoPorConcepto - excluidoPorConcepto);
```

La contabilidad por cuota replica esto con `pagadoPorCuota: Map<cuota.id, number>` y `cuotaTotalmenteExcluida: Set<cuota.id>`.

---

## 9. Ciclos huérfanos y contratos `sin_datos`

Agua y energía se guardan por key de ciclo `periodo = YYYY-MM-01` (consumo del mes previo). Una segunda pasada vía `calcularCiclosAguaPendientes` / `calcularCiclosEnergiaPendientes` captura ciclos con saldo que no cayeron en ningún mes recorrido en el loop principal de `deudaArrastrada` — típico en contratos `sin_datos` (sin fecha de ingreso, sin rango) o desalineaciones históricas de carga. Se agregan como entradas `Ciclo 09/2026`, filtradas contra meses futuros y previos al ingreso.

---

## 10. Semántica de snapshots (planeado)

`snapshot_sumario` es el freeze inmutable de un periodo:

```
snapshot_sumario(contrato_id, periodo_inicio, periodo_fin, fecha_cierre, datos jsonb, reabierto_en, motivo_reapertura)
```

Reglas: cierre idempotente (cerrar dos veces devuelve el mismo snapshot), `reabrir` exige motivo y deja rastro de auditoría, los pagos con fecha dentro de un periodo cerrado igual se aceptan pero no mutan el snapshot — quedan como cobros posteriores al cierre. `datos` se valida con Zod al escribir.

Estado actual: spec y tipos existen; la persistencia es el próximo hito (tasks 24–25). La liquidación en vivo ya alimenta impresión y WhatsApp sin esperar al snapshot.

---

## 11. Superficies de exportación

- **Impresión/PDF:** `window.print()` + `@media print` en `globals.css`. Sin `@react-pdf/renderer` — más liviano, suficiente para mono-usuario donde "Guardar como PDF" desde el navegador es el target.
- **WhatsApp:** `sumario-texto.ts: sumarioToPlainText(sumario)` — texto plano con `*negrita*` (nativo de WhatsApp), emoji moderado, saltos de línea limpios; Clipboard API con fallback silencioso.

---

## 12. Restricciones operativas

- **Zona horaria:** `America/Asuncion` fija vía `date-fns-tz`; todo helper de fecha (`hoy()`, `formatFechaHora`) pasa por ahí. Moneda `Gs. 1.500.000` con separador de miles, sin decimales, `bigint` en BD.
- **RLS:** permisivo `authenticated_all` en v1 (un usuario detrás de Supabase Auth). RLS granular queda para v2 cuando llegue multi-usuario.
- **Deploy:** `nixpacks.toml` fuerza `npm install` sobre `npm ci` — el lock se genera en Windows y le faltan deps opcionales de Linux (ej. `sharp`), con las que `npm ci` fallaría en Coolify.
- **Quirk Next 16:** `middleware.ts` pasó a llamarse `proxy.ts`; el proyecto usa la nueva convención con `updateSession()`.

---

## 13. Trade-offs aún abiertos

- **Prorrateo histórico de agua** reutiliza la ocupación actual — aproximación reconocida; un `unidad_ocupacion_snapshot` por mes sería exacto pero se difirió.
- **Datos globales del edificio** (`configuracion` key-value) son simples pero no tipados a nivel SQL — Zod valida en lectura.
- **Un solo entorno** (producción) — preview/staging se agregan si crece el uso.
- **Sin scraping ANDE aún** — el modelo `factura_ande` ya está listo (`nis, periodo, monto, estado`), pero la carga sigue manual.

El estilo de razonamiento —fijar primero la superficie de auditoría, mantener los ledgers de dinero separados, computar liquidaciones como funciones puras— es el artefacto más transferible.
