# Compenso

**Payroll operations platform** — universal work-data intake, auditable pay runs, payout tracking, and payslips for teams that mix employees and contractors.

Compenso deliberately stops at the tax-filing boundary: it is the **operations layer around payroll**, not a tax filer. It plugs into whatever a business already uses — spreadsheets, time clocks, manual entry — and runs the money workflow end-to-end: intake → approval → pay run → payout tracking → payslips → insight.

## Why it exists

Small businesses run payroll across four disconnected places — a shift app, a clock-in tool, an overtime spreadsheet, and a payroll processor — with a human as the integration layer. Compenso closes that gap:

- **Universal intake** — AI-assisted spreadsheet import, built-in time clock, approval chains
- **Pay engine** — salary, hourly (+ overtime), per-job, commission, tips; employees and contractors in one run
- **Pre-run audit** — deterministic rules catch ghost workers, duplicate entries, overtime spikes, and pay anomalies *before* money moves
- **Immutable runs** — finalized pay runs are snapshotted and versioned; history is reproducible forever
- **Payout tracking to "landed"** — batches, exports, and per-payout status (Compenso tracks money; it does not move it)

## Architecture

pnpm workspace:

| Package | Role |
|---|---|
| `@compenso/engine` | Pure, zero-dependency pay calculation + audit rules. All money is integer cents (ADR-001). Deterministic and fully unit-tested. |
| `@compenso/shared` | Domain types shared by engine, app, and edge functions. |
| `apps/web` | React + TypeScript app on Supabase (Phase 3+). |

Non-negotiables: strict layering (UI never touches the database), RLS on every table with server-side roles as the root of trust, append-only audit log, versioned migrations.

Key decisions are recorded in [`docs/adr/`](docs/adr/). Full design spec: [`docs/specs/`](docs/specs/).

## Roadmap

| Phase | Scope | Status |
|---|---|---|
| 1 | Workspace + pay engine (money math, audit rules, CI) | 🔨 in progress |
| 2 | Database: migrations, RLS, security model | planned |
| 3 | Web app core: auth, orgs, workers, time, imports | planned |
| 4 | Pay runs end-to-end, payouts, payslip PDFs | planned |
| 5 | Worker portal, dashboard, docs polish | planned |

Implementation plans live in [`docs/plans/`](docs/plans/).

## Development

```sh
pnpm install
pnpm -r typecheck
pnpm -r test
```
