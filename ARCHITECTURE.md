# Architecture Deep Dive — Inquilinos

> Paraphrased from the private implementation. Code snippets are shortened for clarity. Spanish domain terms are preserved where they carry business meaning.

---

## 1. Derived Contract State (never stored)

Contract state is not a column; it is a pure function of dates and a building-level threshold.

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

`sin_datos` exists for historical loads where neither entry nor signature date is known — surfaced as a dashboard alert so it is not confused with a real `pendiente_firma`.
`vencido` shows "Vencido hace N días" and remains closable.

Helpers: `diasVencidoContrato`, `diasSinFirmar`, `estadoContratoLabel`, `estadoContratoBadgeClass`.

---

## 2. Billing Period Arithmetic (per contract)

Each contract has its own cycle anchored to `dia_pago_acordado` (D). The building does not have a global "close the month".

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

Edge case: `diaVenc = 31` in February → `diaValidoDelMes` clamps to the real last day of the month. Helper `periodosEntre(entrada, tope)` walks the history with a 400-iteration guard; `mesCalendarioDelPeriodo` bridges per-contract cycles and calendar-month records (water/ANDE are loaded by calendar month).

---

## 3. Late Fee Accrual (`mora`)

Simple, but with an important freeze rule:

```ts
function moraAlquiler({ moraDiaria, saldoAlquiler, fechaLimiteSinMora, fechaRef }): bigint {
  if (!moraDiaria) return 0n;                 // nullable = no fee
  if (saldoAlquiler <= 0) return 0n;          // freezes at 0 the moment it is paid
  const limite = addDays(periodo.inicio, dias_gracia);
  const diasAtraso = Math.max(0, differenceInDays(fechaRef, limite));
  return BigInt(diasAtraso) * moraDiaria;
}
```

`fechaRef` = `hoy` in preview, `fecha_cierre` when the snapshot is frozen. While the period is open the mora is live-recomputed; once closed it is fixed at the closing value. `fechaLimiteSinMora` already accounts for months shorter than `dia_pago_acordado + dias_gracia`.

---

## 4. Water Proration (`prorrateo`)

Three modes, all exclude `unidad.tipo = servicio` (common-area meters like the garage) and units not `ocupada`:

```ts
function prorratearAgua({ montoTotal, metodo, unidadesOcupadas, factoresPonderacion, consumos }) {
  switch (metodo) {
    case "partes_iguales":     return dividir(montoTotal, unidadesOcupadas.length);
    case "ponderado": {        // e.g. grande=1.5, chico=1.0, salon=1.0
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
  // deterministic rounding adjustment: remainder goes to the last unit by codigo
}
```

`prorratear_todas_unidades` flag on `servicio_agua_mes` lets a given month include all units instead of only `ocupada` (useful for building-wide bills). Live recompute warns that historical proration uses current occupancy, so it can drift if occupancy changed since the month was recorded — documented as a v1 approximation.

---

## 5. Settlement Composition (`construirSumario`)

The settlement is a single pure assembly over live data:

```ts
async function construirSumario(supabase, contratoId, { mes, fechaRef }): Promise<Sumario> {
  const datos = await obtenerDatosContrato(supabase, contratoId); // 9 parallel selects + fanouts
  // 1) mesIngreso = fecha_inicio_contrato ?? fecha_entrada
  // 2) deudaArrastrada = all months [mesIngreso .. mes-1] with positive saldos
  // 3) orphan water/energy cycles via calcularCiclos*Pendientes (covers sin_datos contracts)
  // 4) deudaInicial via calcularSaldosDeudaInicial
  // 5) cargos del mes + cuotas pendientes due this month + pendientes abiertos + pagosDelPeriodo
  // 6) totalACobrar = deudaArrastrada + cargosActuales + cuotasPendientes
  return { encabezado, contrato, cargos, cuotasPendientes, deudaArrastrada, pendientes, pagosDelPeriodo, totalACobrar };
}
```

`obtenerDatosContrato` fans out to `factura_ande`, `contrato_pendiente`, `plan_inicial+cuota_inicial`, `deuda_inicial`, `pago`, `unidad(ocupada|todas)`, `servicio_agua_mes`, `configuracion(factores_ponderacion_agua)`, then `pago_aplicado`, `exclusion_saldo`, `consumo_agua_unidad`.

Return type:

```ts
interface Sumario {
  contratoId: string;
  encabezado: { inquilinos: string[]; unidad: string; mesLabel: string; periodoInicio: string; periodoFin: string; fechaGeneracion: string; cerradoEn: string|null };
  contrato: { estado: string; montoAlquiler: number; diaPagoAcordado: number };
  cargos: CargoSumario[];              // alquiler (+ todo incluido), estacionamiento, basura, agua, energia, mora
  cuotasPendientes: CuotaPendienteSumario[]; // due or overdue this month
  deudaArrastrada: DeudaArrastradaItem[];    // all prior months with balance
  pendientes: PendienteSumario[];       // open contract-level todos
  pagosDelPeriodo: PagoSumario[];       // by payment date within month
  totalACobrar: number;
}
```

Design bet: the settlement is **always live-computed**, even when a `mes_cerrado` report exists for that month. The dashboard uses it as a monitoring surface; the frozen record is consulted separately under `/cierres/YYYY-MM` as history. This keeps live truth and audit truth separate.

---

## 6. `cargo_fijo` — All-Inclusive Override

A commercial agreement flag that collapses recurring charges into one line:

```ts
if (c.cargo_fijo_activo && c.cargo_fijo_monto !== null) {
  return {
    alquiler: c.cargo_fijo_monto,
    estacionamiento: 0, basura: 0, agua: 0, energia: 0,
    cicloAgua: cicloAgua, cicloEnergia: cicloEnergia, // retained for traceability
  };
}
```

Debt and initial installments still apply on top (one-time start-of-contract charges). The settlement renders a single row `Alquiler (todo incluido)` and skips the separate parking/garbage/water/energy rows. Payments against it use `cargo_tipo="alquiler"` with the month key.

---

## 7. Entry-Month Forgiveness (`cobra_*_ingreso`)

Three per-contract toggles (`cobra_alquiler_ingreso`, `cobra_basura_ingreso`, `cobra_estacionamiento_ingreso`) handle the mid-month entry case explicitly: the spec deliberately **does not prorate** a partial first month.

```ts
function cargosBrutosDelMes(datos, mes, { esMesDeIngreso, alquilerCubiertoPorPlan }) {
  if (alquilerCubiertoPorPlan) return { alquiler: 0, detalle: "Cubierto por plan Primer alquiler" };
  const alquilerPerdonado = esMesDeIngreso && !c.cobra_alquiler_ingreso;
  return { alquiler: alquilerPerdonado ? 0 : c.monto_alquiler, … };
}
```

If `primer_alquiler` plan exists, it takes precedence over the forgiveness flag — the rent for the entry month is expected to be covered by that plan's installments.

Water and energy cycles labeled `≤ mesDeIngreso` are also skipped, since a cycle key `YYYY-MM-01` represents consumption of the prior month.

---

## 8. `pago` vs `exclusion_saldo` (no floating money)

Every guaraní is accounted for in one of two ledgers:

- **`pago` + `pago_aplicado`**: real money received. A `pago` row holds `fecha, monto, metodo`; its distribution across cargos is stored in `pago_aplicado(cargo_tipo, periodo, monto)` or `cuota_inicial(cargo_ref)`. Creating a payment atomically inserts `pago` + N `pago_aplicado`; if the sum of applications is less than `pago.monto`, the remainder auto-goes to `saldo_favor` so money is never unassigned; if it exceeds, the action rejects. Rollback on `pago_aplicado` failure deletes the parent `pago`.

- **`exclusion_saldo`**: manual adjustment that reduces a balance without being money received (e.g., negotiated forgiveness, billing correction). Same `cargo_tipo + periodo + monto` shape, plus `excluir_total` which forces the balance to 0 regardless of the current gross (important when proration totals move). Kept separate so the payment history stays a clean "money received" ledger.

Both ledgers are consulted together in `saldosDelMes`:

```ts
const saldo = excluidoTotal ? 0
  : Math.max(0, bruto - pagadoPorConcepto - excluidoPorConcepto);
```

Per-installment accounting mirrors this with `pagadoPorCuota: Map<cuota.id, number>` and `cuotaTotalmenteExcluida: Set<cuota.id>`.

---

## 9. Orphan Cycles and `sin_datos` Contracts

Water and energy are stored by cycle key `periodo = YYYY-MM-01` (consumption of the prior month). A second pass via `calcularCiclosAguaPendientes` / `calcularCiclosEnergiaPendientes` catches cycles that have a balance but did not fall into any month walked in the main `deudaArrastrada` loop — typical for `sin_datos` contracts (no entry date, so no month range) or historical load misalignments. They are appended as `Ciclo 09/2026` entries, filtered against future months and pre-entry cycles.

---

## 10. Snapshot Semantics (planned)

`snapshot_sumario` is the immutable freeze of a period:

```
snapshot_sumario(contrato_id, periodo_inicio, periodo_fin, fecha_cierre, datos jsonb, reabierto_en, motivo_reapertura)
```

Rules: idempotent close (closing twice returns the same snapshot), `reabrir` requires a reason and leaves an audit trail, payments dated within a closed period are still accepted but do not mutate the snapshot — they are recorded as post-close receipts. `datos` is validated by Zod on write.

Current status: spec and types exist; persistence is the next milestone (spec tasks 24–25). The live settlement already powers print and WhatsApp export without waiting for the snapshot.

---

## 11. Export Surfaces

- **Print/PDF:** `window.print()` + `@media print` in `globals.css`. No `@react-pdf/renderer` — lighter, sufficient for a single-admin use case where "Save as PDF" from the browser is the target.
- **WhatsApp:** `sumario-texto.ts: sumarioToPlainText(sumario)` — plain text with `*bold*` (native WhatsApp), moderate emoji, clean line breaks; Clipboard API with silent fallback.

---

## 12. Operational Constraints

- **Timezone:** `America/Asuncion` fixed via `date-fns-tz`; every date helper (`hoy()`, `formatFechaHora`) goes through it. Currency is `Gs. 1.500.000` with thousands separator, no decimals, `bigint` in the DB.
- **RLS:** permissive `authenticated_all` in v1 (single user behind Supabase Auth). Fine-grained RLS is a v2 item when multi-user arrives.
- **Deploy:** `nixpacks.toml` forces `npm install` over `npm ci` — the lock is generated on Windows and misses Linux-only optional deps (e.g. `sharp`), which `npm ci` would fail on in Coolify.
- **Next 16 quirk:** `middleware.ts` was renamed to `proxy.ts`; the project uses the new convention with `updateSession()`.

---

## 13. Trade-offs Not Yet Resolved

- **Historical water proration** reuses current occupancy — acknowledged approximation; a `unidad_ocupacion_snapshot` per month would be exact but was deferred.
- **Global building data** (`configuracion` key-value) is simple but not typed at the SQL level — Zod validates on read.
- **Single environment** (production only) — preview/staging to be added if usage grows.
- **No ANDE scraping yet** — the `factura_ande` model is ready for it (`nis, periodo, monto, estado`), but loading is still manual.

The reasoning style — fix the audit surface first, keep money ledgers separate, compute settlements as pure functions — is the more transferable artifact.
