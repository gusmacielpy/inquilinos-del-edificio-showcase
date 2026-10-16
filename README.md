# Inquilinos — Building Management & Monthly Settlement Platform

> 🇬🇧 English (current) · 🇪🇸 [Leer en español](./README.es.md) · → [Architecture deep dive](./ARCHITECTURE.md)

**A Next.js + Supabase SaaS that replaces a paper-and-pen building administration workflow with a domain-rich settlement engine.**

> A single-tenant building of 10 units (2 blocks, 3 floors, Gs. currency) where the most expensive manual task — generating the monthly tenant settlement — required crossing scattered data (rent, guarantee installments, water proration, ANDE electricity bills, late fees, carried debt). Now it runs in seconds.

**Status:** In production for a family building in Paraguay. Single-admin (one user) v1, responsive, deployed on Coolify (self-hosted VPS) with Supabase Postgres.

> This repository is a **case study** of the architecture and engineering decisions. The implementation lives in a private repository. Source access / live demo can be arranged on request for recruiters or technical leads.
>
> The operational system is in Spanish because it serves a Paraguayan admin. This document is in English for broader readability; Spanish domain terms are preserved where relevant (`contrato`, `sumario`, `mora`, `prorrateo`).

---

## TL;DR

- **Domain over CRUD:** ~9.8k LOC of typed domain logic (`sumario.ts` 1.2k lines alone) — period arithmetic, late-fee (`mora`) accrual, water proration (3 modes), and a composable `construirSumario` settlement.
- **Settlement as pure function:** `construirSumario(supabase, contratoId, { mes, fechaRef })` assembles header, current charges, pending installments, carried debt, ANDE invoices, period payments and `totalACobrar` — always live-computed, optionally frozen as immutable `snapshot_sumario` (JSONB + Zod).
- **Billing cycle per contract:** Each `contrato` has its own cut day (`dia_pago_acordado`), so closings are independent — not a global "close the building" month.
- **Money done right:** Guaraníes (`bigint`, no decimals), explicit `pago → pago_aplicado` distribution with `saldo_favor` fallback, and `exclusion_saldo` as a non-payment adjustment — never "floating" money.
- **Operable admin UX:** Print CSS + WhatsApp plain-text export (`sumario-texto.ts`), dashboard alerts, per-contract renewal/finalization lifecycle, all timezone-fixed to `America/Asuncion`.
- **Stack:** Next.js 16 (App Router, RSC), React 19, TypeScript, Tailwind 4, Supabase (Postgres + Auth), Zod, date-fns/date-fns-tz, Coolify + Nixpacks.

---

## Why This Project Exists

The previous workflow was a notebook: the admin transcribed rents, water totals, electricity NIS slips and handwritten agreements, then manually calculated what each tenant owed that month. Errors in proration or late fees were common, and carried debt was hard to audit.

Specific constraints made an off-the-shelf property manager a poor fit:

- Billing cycle is **per contract**, not calendar month — anchored to `dia_pago_acordado`.
- Water cost is split by **configurable proration** (equal / weighted / per-meter), with a building-level factor table.
- Electricity (ANDE) bills are **per NIS** and may be paid directly by the tenant or by the admin — both cases affect the settlement differently.
- Initial obligations (first rent, guarantee, notary) are **installment plans** with per-installment due dates and amounts.
- Mid-month entry must not auto-generate a partial charge without an explicit business rule — the first collection is left to a manual payment/pending.

---

## Key Features

- **Unit catalog.** 10 units `A0Ch, A0Sa, A1Ch, A1Gr, B0Ch, B0Gr, B1Ch, B1Gr, B2Ch, B2Gr` with `tipo` (`chico | grande | salon | servicio`), occupancy state, NIS list and notes. Cannot delete a unit with historical contracts.
- **Tenants.** independent of contracts (a tenant can outlive a contract), supports multiple titulars per contract, search by name/phone.
- **Contracts.** full economic + temporal model: `monto_alquiler`, `dia_pago_acordado` (1–31), `dias_gracia`, `mora_diaria` (nullable = no fee), `umbral_alerta_dias`, parking (`cantidad × monto_unitario`), `cobra_energia_admin`, `monto_basura`, and three `cobra_*_ingreso` forgiveness toggles for the entry month. States are **derived** (`pendiente_firma | vigente | por_vencer | vencido | renovado | finalizado | sin_datos`), never stored.
- **Initial plans.** up to 3 per contract (`primer_alquiler | garantia | escribania`), each with arbitrary per-installment amounts and due dates; UI guards that installments sum to the plan total.
- **Water service.** monthly `servicio_agua_mes` with `monto_total` and `metodo_prorrateo`; proration excludes `tipo=servicio` and vacant/out-of-service units, with deterministic rounding adjustment.
- **ANDE invoices.** per-NIS, per-period ledger with `estado` (`pendiente | pagada | vencida` derived), `pagador` (`inquilino_directo | administracion`), optional `fecha_pago`.
- **Payments.** `pago` + `pago_aplicado` (typed distribution to `alquiler | mora | agua | energia | estacionamiento | basura | cuota_inicial | deuda_inicial | saldo_favor`). Auto-suggest by oldest due installment, editable.
- **Monthly settlement (sumario).** preview any month, print and WhatsApp export, close as immutable snapshot, reopen with audit (`reabierto_en`, `motivo`).
- **Dashboard & alerts.** occupied/vacant/out-of-service counts, pending-signature / expiring / expired / closable contracts, overdue ANDE, total carried debt — all clickable.

---

## High-level Architecture

```
Vercel / Coolify (Next.js 16 App Router)
├─ (app) — authenticated route group (proxy.ts session refresh)
│  ├─ / (dashboard)  /unidades  /inquilinos  /contratos  /servicios  /configuracion
│  └─ /contratos/[id]/sumario, /pagos, /planes, /pendientes, /cierres
│  Server Actions for mutations (create payment, close snapshot, load invoice…)
│  Route handlers for PDF print + WhatsApp text
└───────────────┬──────────────────────────────
                │ @supabase/ssr (service role on server, anon + RLS permissive in v1)
                ▼
        Supabase Postgres
        13+ tables (unidad, inquilino, contrato, plan_inicial, cuota_inicial,
        servicio_agua_mes, consumo_agua_unidad, factura_ande, pago, pago_aplicado,
        snapshot_sumario, configuracion, contrato_pendiente …)
        + JSONB snapshots + triggers (touch_modificado_en)
```

Three surfaces: **Next.js app** (UI + API), **Supabase** (state), **Coolify/Nixpacks** (deploy).

---

## Stack

| Layer | Choice | Notes |
|-------|--------|-------|
| Frontend + Backend | Next.js 16 (App Router) + TypeScript + RSC | `proxy.ts` replaces legacy `middleware.ts` in Next 16 |
| UI | Tailwind 4 | Print CSS for PDF, no heavy PDF lib |
| DB & Auth | Supabase Postgres + Auth (email/password, single user v1) | `sb_publishable_*` key, service role only server-side |
| Validation | Zod | Forms, payloads, snapshot JSONB |
| Dates | date-fns + date-fns-tz | Fixed `America/Asuncion`, no UTC drift |
| Currency | Custom `formatGs` helper | `Gs. 1.500.000`, `bigint`, no decimals |
| Hosting | Coolify (self-hosted VPS) + Nixpacks | `nixpacks.toml` forces `npm install` over `npm ci` for cross-platform locks |
| Types | `supabase gen types` → `database.types.ts` | Regenerated from live schema |

---

## Deep Dive

Architecture decisions, domain rules and trade-offs:

**→ [ARCHITECTURE.md](./ARCHITECTURE.md)** · [Versión en español](./ARCHITECTURE.es.md)

Covers: derived contract states, billing period arithmetic, `mora` accrual, water proration (3 modes), settlement composition, `exclusion_saldo` vs `pago`, snapshot semantics, and known limitations.

---

## Status and Roadmap

**Shipped:**

- Unit/tenant/contract CRUD with derived states, renewal and finalization flows.
- Initial installment plans with per-installment editing and total guard.
- Water monthly entry + proration, ANDE invoice ledger, payment distribution with `saldo_favor`.
- Live-computed settlement with print and WhatsApp export.
- Config page (alert threshold, water weighting factors, building header).

**Next:**

- Persisted `snapshot_sumario` close/reopen (immutable JSONB freeze).
- Consolidated monthly listing (all active contracts) with zip export.
- Dashboard alert aggregation and economic summary.
- ANDE scraping automation (v2).

---

## Repository Access

Source is private due to real tenant PII (names, CI/RUC, phone, NIS) in production data and git history. Access on request for:

- Technical recruiters / hiring managers evaluating senior / product engineering.
- Engineering leads interested in the settlement engine patterns.

Contact: [contacto@gusmaciel.com](mailto:contacto@gusmaciel.com) · [linkedin.com/in/gusmacielpy](https://www.linkedin.com/in/gusmacielpy)

Portfolio: [gusmaciel.com](https://gusmaciel.com)

---

## Credits

Design, implementation and building onboarding by [Gus Maciel](https://gusmaciel.com). Specs co-authored with the building admin from the paper workflow.
