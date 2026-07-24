# Compenso — MVP Design Spec

**Date:** 2026-07-24 · **Status:** awaiting user review
**Goal:** portfolio-grade payroll operations platform. Built to read "expert developer" on GitHub/CV first; sellable product second.

---

## 1. What Compenso is

A plug-and-play **payroll operations layer** for SMBs with mixed teams (employees + contractors). It ingests work data from wherever it lives (spreadsheets, manual entry, built-in time clock), runs an auditable pay cycle, tracks payouts to "landed," and generates payslips — **without ever touching tax filing**.

Explicit non-goals (MVP): tax calculation/filing, moving money (we track, not transmit), accounting features (invoicing, books), white-label mode, GHL app, multi-currency runs. These are V2/V3 per the roadmap in `mindmap.html`.

## 2. MVP cut-line

**In:**
1. Org setup + team roles (owner / admin / manager / worker)
2. Workers: employees + contractors, versioned compensation history (salary, hourly, per-job, commission, tips)
3. Time & work intake: manual entries, built-in punch clock, **CSV/Excel import with AI column mapping**, manager approval flow
4. Adjustments: bonus, deduction, reimbursement, advance + repayment tracking
5. Pay runs: draft → review (AI pre-run audit) → approved → finalized (immutable) → paid; off-cycle runs supported
6. **Pre-run audit v1**: rule-based checks (ghost worker, duplicate entry, overtime spike, sudden pay jump, missing bank details) surfaced as blocking/warning findings
7. Payout batches: grouped by method, CSV/bank-file export, per-payout status (pending → sent → landed/failed)
8. Payslips: branded PDF per worker per run, private storage, signed URLs
9. Worker portal: own payslips, own time, own profile
10. Append-only audit log on every sensitive action
11. Dashboard: labor cost this period, run status, pending approvals

**Out (fast-follow after MVP ships):** integrations (Clockify/QuickBooks), NL "ask payroll anything," PTO, e-sign onboarding, WhatsApp notifications, profitability reports.

## 3. Stack & repo layout

React 18 + TypeScript + Vite + Tailwind + shadcn/ui · Supabase (Postgres, Auth, RLS, Storage, Edge Functions) · Vercel deploy · pnpm workspace · Vitest · GitHub Actions CI.

**New dedicated Supabase project** (`compenso`) — not shared with LofiHub/Codex/SIPS.

```
compenso/
├── apps/web/                  # React app
│   └── src/
│       ├── pages/             # route-level composition only
│       ├── features/<domain>/ # components + hooks per domain (workers, runs, time, payouts…)
│       ├── services/          # ONLY layer that talks to Supabase; one file per domain
│       ├── lib/               # supabase client, utils, formatting
│       └── types/             # re-exports from @compenso/shared
├── packages/engine/           # pure TS pay-calculation engine — zero deps, zero I/O
├── packages/shared/           # types + zod schemas shared by web, engine, functions
├── supabase/
│   ├── migrations/            # versioned SQL, numbered
│   └── functions/             # edge functions (ai-mapping, payslip-pdf, finalize-run)
├── docs/
│   ├── specs/                 # this file
│   ├── adr/                   # architecture decision records
│   └── architecture.md        # diagrams (mermaid): layering, run lifecycle, RLS model
└── .github/workflows/ci.yml   # typecheck + lint + test on push
```

**Layering rule (hard):** UI → feature hooks → services → Supabase. Components never import the Supabase client. The engine never imports anything from the app. Files ≤300 lines.

**Why a workspace:** `@compenso/engine` as a pure, framework-free, fully-tested package is the centerpiece portfolio signal — money math with 100% test coverage, usable from web, edge functions, and (later) the embed API.

## 4. Data model (Postgres)

All money = **integer minor units (cents) + ISO currency code**. Never floats.

| Table | Purpose / key fields |
|---|---|
| `organizations` | name, base_currency, settings |
| `profiles` | 1:1 auth.users; display fields only — never trust for authz |
| `memberships` | org_id + user_id + role (`owner/admin/manager/worker`) — **the root of trust** |
| `workers` | org-scoped; type `employee/contractor`; payout_method jsonb; optional user_id for portal |
| `compensations` | worker_id, pay_type (`salary/hourly/per_job/commission/tips`), rate_cents, frequency, `effective_from/effective_to` — comp history is versioned, never overwritten |
| `time_entries` | worker, date, minutes, source (`clock/import/manual`), status (`pending/approved/rejected`), approver |
| `adjustments` | type (`bonus/deduction/reimbursement/advance/advance_repayment`), amount_cents, status, optional run link |
| `imports` | uploaded file ref, AI-proposed mapping jsonb, user-confirmed mapping, row results |
| `pay_runs` | period, frequency (`weekly/biweekly/monthly/off_cycle`), status, totals, finalized_at/by, calc_version |
| `pay_run_items` | per worker per run: **inputs snapshot jsonb** + gross/adjustments/net cents. Frozen at finalize |
| `audit_findings` | run_id, rule, severity (`blocker/warning`), worker_id, message, resolved_by |
| `payout_batches` | run_id, method, export file ref, status |
| `payouts` | item_id, amount, status (`pending/sent/landed/failed`), timestamps |
| `payslips` | item_id, storage path, sequential number per org |
| `audit_log` | append-only: actor, action, entity, before/after jsonb. INSERT-only (no UPDATE/DELETE policies) |

**Pay run lifecycle (state machine, enforced by trigger + service layer):**
`draft → in_review → approved → finalized → paying → paid` (+ `cancelled` from any pre-finalized state). Finalize snapshots every input into `pay_run_items` and stamps `calc_version`; after that, run + items are immutable (trigger-rejected updates). Finalize with unresolved `blocker` findings is refused server-side (edge function `finalize-run`), not just hidden in UI.

## 5. Security model

- **Root of trust = `memberships.role`**, checked via `security definer` helper functions (`compenso_role(org_id)`) — never `user_metadata`, never client-asserted (your standing rule).
- RLS on **every** table, org-scoped; workers' policies restrict to `worker.user_id = auth.uid()` rows (own payslips/time only).
- All reads bounded (pagination server-side); counts via `head:true`.
- Payslip PDFs in a **private** bucket; access via short-TTL signed URLs only.
- No email/account enumeration anywhere (generic auth errors).
- `SECURITY.md` in repo documents the whole model — part of the portfolio deliverable.

## 6. Pay engine (`@compenso/engine`)

Pure function core: `calculateRun(inputs: RunInputs): RunResult`.

- Deterministic: same inputs → same output; `calc_version` recorded on every finalized run so historical runs stay reproducible after engine upgrades.
- Handles: hourly (minutes × rate with rounding policy), salary **proration** (mid-period start/end, day-count basis documented), per-job, commission, tips; adjustment application order (gross → additions → deductions → net, floor at zero with explicit shortfall carry).
- Overtime: simple threshold rule per org (e.g. >40h/wk multiplier) — configurable, MVP-simple.
- Rounding: half-up at cent level, applied once at final aggregation per line — documented in an ADR.
- **Test suite is the showpiece:** proration edges, rounding accumulation, mixed comp types, off-cycle overlap, advance-repayment clamping. Target ~100% coverage on this package.

## 7. AI features (MVP scope only)

1. **Import mapping** (`ai-mapping` edge function): sends CSV headers + 5 sample rows to Claude (Haiku 4.5 — cheap, sufficient), returns proposed column→field mapping + per-column confidence; user confirms in UI before any row is written. API key lives server-side only.
2. **Pre-run audit v1 = deterministic rules** (no LLM): ghost worker (no activity but nonzero pay), duplicate time entries, overtime spike vs 4-run trailing average (default: >1.5×), net-pay jump vs last run (default: >50%, org-configurable), missing payout details, unapproved time in period. Rule engine lives in `@compenso/engine` (pure + tested). LLM explanations of findings = fast-follow.

## 8. Payslips

Edge function `payslip-pdf` renders a branded template (org logo/name, period, earnings/adjustments breakdown, YTD totals) → PDF → private bucket → path on `payslips` row. Generated at finalize for all items in the run. Sequential payslip numbers per org (gapless counter table).

## 9. UI surface (MVP screens)

Admin: Dashboard · Workers (list/detail/comp history) · Time (entries + approval queue) · Imports (upload → AI mapping review → commit) · Adjustments · Pay Runs (list + 4-step run wizard: Gather → Review/Audit → Approve → Finalize) · Payouts (batches + status board) · Payslips · Audit log · Settings (org, team, roles).
Worker portal: My payslips · My time (+ punch clock) · My profile.

Design goes through the standard pipeline (`lofistack-frontend-stack`: ui-ux-pro-max → design-taste-frontend → shadcn/Magic → GSAP → UX QA) at build time — award-winning bar applies. Visual direction decided then, not in this spec.

## 10. Testing & CI

- `packages/engine`: Vitest, exhaustive (the money math is the CV artifact).
- `apps/web` services: Vitest with mocked Supabase client for state-machine guards.
- RLS: SQL test script asserting cross-org and worker-scope denials (run against local Supabase).
- CI (GitHub Actions): pnpm install → typecheck → lint → test, badge in README. Live once the remote exists.
- Definition of done per your protocol: tsc + lint clean, tests pass, no file >300 lines, layering intact.

## 11. Docs deliverables (portfolio surface)

`README.md` (pitch, architecture diagram, screenshots, badge) · `docs/architecture.md` (mermaid: layering, run lifecycle, RLS model) · `SECURITY.md` · `docs/adr/` (ADR-001 money-as-integers, ADR-002 run immutability/snapshot, ADR-003 multi-tenant RLS design, ADR-004 pure engine package). No demo deployment in scope (user decision 2026-07-24); seed data exists only for local dev/tests.

## 12. Risks / constraints

- "Payroll" wording sets tax expectations → product copy says **payroll operations**; README states the no-filing boundary explicitly.
- No money movement in MVP — payout "sent/landed" is human-confirmed status, and the UI must be honest about that.
- New Supabase project + Vercel project required (user creates; MCP added after).
- Public repo → no real personal data ever in seeds/fixtures; demo data is synthetic.

## 13. Resolved decisions (2026-07-24)

1. **Currency:** USD default, org-configurable field from day one. No demo deployment (user decision) — build only.
2. **GitHub remote:** deferred — local git only until the user green-lights remote creation (standing rule: ask before remote/push).
3. **Supabase:** engine + skeleton phases run without it; the cloud project gets created (user dashboard action) when the first migration lands.
