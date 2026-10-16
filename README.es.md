# Inquilinos — Plataforma de Gestión de Edificio y Liquidación Mensual

> 🇬🇧 [Read in English](./README.md) · 🇪🇸 Español (actual) · → [Arquitectura detallada](./ARCHITECTURE.es.md)

**SaaS en Next.js + Supabase que reemplaza la administración en papel y bolígrafo por un motor de liquidación con dominio rico.**

> Edificio de 10 unidades (2 bloques, 3 pisos, moneda Gs.) donde la tarea más costosa —generar la liquidación mensual de cada inquilino— exigía cruzar datos dispersos (alquiler, cuotas de garantía/escribanía, prorrateo de agua, facturas ANDE, mora, deuda arrastrada). Ahora sale en segundos.

**Estado:** En producción para un edificio familiar en Paraguay. Versión 1 mono-usuario (un solo administrador), responsive, desplegado en Coolify (VPS propio) con Supabase Postgres.

> Este repositorio es un **case study** de la arquitectura y las decisiones de ingeniería. La implementación está en un repositorio privado. Acceso al código / demo en vivo disponible a pedido para recruiters o líderes técnicos.

---

## Resumen (TL;DR)

- **Dominio antes que CRUD:** ~9,8k LOC de lógica de dominio tipada (`sumario.ts` 1.200 líneas solo) — aritmética de periodos, mora, prorrateo de agua (3 modos) y el ensamblado componible `construirSumario`.
- **Liquidación como función pura:** `construirSumario(supabase, contratoId, { mes, fechaRef })` arma encabezado, cargos del mes, cuotas pendientes, deuda arrastrada, facturas ANDE, pagos del periodo y `totalACobrar` — siempre calculado en vivo, opcionalmente congelado como `snapshot_sumario` inmutable (JSONB + Zod).
- **Ciclo de facturación por contrato:** cada `contrato` tiene su propio día de corte (`dia_pago_acordado`), los cierres son independientes — no hay un "cierre global del edificio".
- **Dinero bien hecho:** guaraníes (`bigint`, sin decimales), distribución explícita `pago → pago_aplicado` con fallback `saldo_favor`, y `exclusion_saldo` como ajuste sin ser pago real — nunca queda dinero "flotando".
- **UX operativa:** exportación Print CSS + texto plano para WhatsApp (`sumario-texto.ts`), alertas en dashboard, ciclo de vida de renovación/finalización por contrato, todo con zona horaria fija `America/Asuncion`.
- **Stack:** Next.js 16 (App Router, RSC), React 19, TypeScript, Tailwind 4, Supabase (Postgres + Auth), Zod, date-fns/date-fns-tz, Coolify + Nixpacks.

---

## Por qué existe este proyecto

El flujo anterior era un cuaderno: la administración transcribía alquileres, totales de agua, boletas de NIS y acuerdos manuscritos, y luego calculaba a mano cuánto debía cada inquilino ese mes. Errores de prorrateo o mora eran comunes y la deuda arrastrada era difícil de auditar.

Restricciones que hacían que un administrador de propiedades genérico no encajara:

- El ciclo de facturación es **por contrato**, no por mes calendario — anclado a `dia_pago_acordado`.
- El agua se reparte con **prorrateo configurable** (partes iguales / ponderado / medidor individual), con tabla de factores a nivel edificio.
- Las facturas de ANDE son **por NIS** y pueden ser pagadas directo por el inquilino o por la administración — ambos casos afectan distinto a la liquidación.
- Las obligaciones iniciales (primer alquiler, garantía, escribanía) son **planes en cuotas** con fecha y monto por cuota.
- El ingreso a mitad de mes no debe generar un cargo parcial automático sin regla de negocio explícita — el primer cobro queda a cargo de un pago/pendiente manual.

---

## Funcionalidades principales

- **Catálogo de unidades.** 10 unidades `A0Ch, A0Sa, A1Ch, A1Gr, B0Ch, B0Gr, B1Ch, B1Gr, B2Ch, B2Gr` con `tipo` (`chico | grande | salon | servicio`), estado de ocupación, lista de NIS y notas. No se puede borrar una unidad con contratos históricos.
- **Inquilinos.** independientes de los contratos (un inquilino puede sobrevivir a un contrato), múltiples titulares por contrato, búsqueda por nombre/teléfono.
- **Contratos.** modelo económico y temporal completo: `monto_alquiler`, `dia_pago_acordado` (1–31), `dias_gracia`, `mora_diaria` (nullable = sin mora), `umbral_alerta_dias`, estacionamiento (`cantidad × monto_unitario`), `cobra_energia_admin`, `monto_basura` y tres toggles `cobra_*_ingreso` de perdón del mes de ingreso. Estados **derivados** (`pendiente_firma | vigente | por_vencer | vencido | renovado | finalizado | sin_datos`), nunca almacenados.
- **Planes iniciales.** hasta 3 por contrato (`primer_alquiler | garantia | escribania`), cada uno con montos y vencimientos arbitrarios por cuota; la UI valida que la suma cuadre con el total del plan.
- **Servicio de agua.** `servicio_agua_mes` mensual con `monto_total` y `metodo_prorrateo`; el prorrateo excluye `tipo=servicio` y unidades vacías/fuera de servicio, con ajuste determinístico por redondeo.
- **Facturas ANDE.** ledger por NIS y por periodo con `estado` (`pendiente | pagada | vencida` derivado), `pagador` (`inquilino_directo | administracion`), `fecha_pago` opcional.
- **Pagos.** `pago` + `pago_aplicado` (distribución tipada a `alquiler | mora | agua | energia | estacionamiento | basura | cuota_inicial | deuda_inicial | saldo_favor`). Sugerencia automática por antigüedad, editable.
- **Liquidación mensual (sumario).** previsualización de cualquier mes, exportación a impresión y WhatsApp, cierre como snapshot inmutable, reapertura auditada (`reabierto_en`, `motivo`).
- **Dashboard y alertas.** conteos ocupado/vacío/fuera de servicio, contratos pendientes de firma / por vencer / vencidos / listos para cerrar, ANDE vencidas, deuda total arrastrada — todo clickeable.

---

## Arquitectura de alto nivel

```
Vercel / Coolify (Next.js 16 App Router)
├─ (app) — route group autenticado (proxy.ts renueva sesión)
│  ├─ / (dashboard)  /unidades  /inquilinos  /contratos  /servicios  /configuracion
│  └─ /contratos/[id]/sumario, /pagos, /planes, /pendientes, /cierres
│  Server Actions para mutaciones (crear pago, cerrar snapshot, cargar factura…)
│  Route handlers para PDF imprimible + texto WhatsApp
└───────────────┬──────────────────────────────
                │ @supabase/ssr (service role en server, anon + RLS permisivo en v1)
                ▼
        Supabase Postgres
        13+ tablas (unidad, inquilino, contrato, plan_inicial, cuota_inicial,
        servicio_agua_mes, consumo_agua_unidad, factura_ande, pago, pago_aplicado,
        snapshot_sumario, configuracion, contrato_pendiente …)
        + snapshots JSONB + triggers (touch_modificado_en)
```

Tres superficies: **app Next.js** (UI + API), **Supabase** (estado), **Coolify/Nixpacks** (deploy).

---

## Stack

| Capa | Elección | Notas |
|------|----------|-------|
| Frontend + Backend | Next.js 16 (App Router) + TypeScript + RSC | `proxy.ts` reemplaza al viejo `middleware.ts` en Next 16 |
| UI | Tailwind 4 | CSS de impresión para PDF, sin librería pesada |
| BD y Auth | Supabase Postgres + Auth (email/password, un usuario en v1) | key `sb_publishable_*`, service role solo en server |
| Validación | Zod | Formularios, payloads, JSONB del snapshot |
| Fechas | date-fns + date-fns-tz | `America/Asuncion` fijo, sin drift UTC |
| Moneda | helper propio `formatGs` | `Gs. 1.500.000`, `bigint`, sin decimales |
| Hosting | Coolify (VPS propio) + Nixpacks | `nixpacks.toml` fuerza `npm install` sobre `npm ci` por locks cruzados |
| Tipos | `supabase gen types` → `database.types.ts` | Regenerado desde el schema vivo |

---

## Deep Dive

Decisiones de arquitectura, reglas de dominio y trade-offs:

**→ [ARCHITECTURE.es.md](./ARCHITECTURE.es.md)** · [Versión en inglés](./ARCHITECTURE.md)

Cubre: estados derivados, aritmética de periodos, mora, prorrateo de agua (3 modos), composición de la liquidación, `exclusion_saldo` vs `pago`, semántica de snapshots y limitaciones conocidas.

---

## Estado y roadmap

**Ya entregado:**

- CRUD de unidades/inquilinos/contratos con estados derivados, flujos de renovación y finalización.
- Planes iniciales con edición por cuota y guarda de total.
- Carga mensual de agua + prorrateo, ledger ANDE, distribución de pagos con `saldo_favor`.
- Liquidación calculada en vivo con exportación a impresión y WhatsApp.
- Página de configuración (umbral de alerta, factores de ponderación, encabezado del edificio).

**Siguiente:**

- Snapshot `snapshot_sumario` persistido (freeze JSONB inmutable) con cierre/reapertura.
- Listado consolidado mensual (todos los contratos activos) con export zip.
- Agregación de alertas y resumen económico en dashboard.
- Scraping automático de ANDE (v2).

---

## Acceso al repositorio

El código es privado por PII real de inquilinos (nombres, CI/RUC, teléfono, NIS) en datos de producción e historial de git. Acceso a pedido para:

- Recruiters / hiring managers evaluando perfil senior / product engineering.
- Líderes técnicos interesados en los patrones del motor de liquidación.

Contacto: [contacto@gusmaciel.com](mailto:contacto@gusmaciel.com) · [linkedin.com/in/gusmacielpy](https://www.linkedin.com/in/gusmacielpy)

Portfolio: [gusmaciel.com](https://gusmaciel.com)

---

## Créditos

Diseño, implementación y onboarding del edificio por [Gus Maciel](https://gusmaciel.com). Specs co-escritos con la administración a partir del flujo en papel.
