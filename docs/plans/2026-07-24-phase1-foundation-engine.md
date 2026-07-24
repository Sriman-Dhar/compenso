# Compenso Phase 1 — Foundation & Pay Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** pnpm workspace with `@compenso/shared` (domain types) and `@compenso/engine` (pure, fully-tested pay calculation + audit rules), CI-ready.

**Architecture:** Monorepo (pnpm workspace). The engine is a zero-dependency, zero-I/O TypeScript package — pure functions from typed inputs to typed results. All money is integer cents. The web app (Phase 3) and edge functions (Phase 4) consume it; nothing in this phase touches Supabase.

**Tech Stack:** TypeScript 5, pnpm workspaces, Vitest, GitHub Actions (workflow file only; remote deferred).

## Global Constraints

- All money values are **integer cents** (`number`, validated integer, ≥ 0 where noted). Never floats, never string math.
- No file over 300 lines.
- Engine package: **zero runtime dependencies, zero I/O** — no fetch, no fs, no Date.now() inside calculations.
- TDD strictly: failing test → minimal implementation → pass → commit.
- Commits end with: `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`
- Git identity is repo-local `Sriman-Dhar <Sriman-Dhar@users.noreply.github.com>` (already configured). No remote, no push in this phase.
- Working directory: `/Users/srimandhar/Work/Compenso`

**Phase roadmap (later plans, not this one):** Phase 2 DB+RLS · Phase 3 web app core · Phase 4 pay-run E2E + edge functions · Phase 5 portal/dashboard/docs.

---

### Task 1: Workspace scaffold

**Files:**
- Create: `pnpm-workspace.yaml`, `package.json`, `tsconfig.base.json`, `.gitignore`, `.nvmrc`
- Create: `packages/shared/package.json`, `packages/shared/tsconfig.json`
- Create: `packages/engine/package.json`, `packages/engine/tsconfig.json`, `packages/engine/vitest.config.ts`
- Create: `docs/adr/ADR-004-pure-engine-package.md`

**Interfaces:**
- Produces: workspace where `pnpm -r typecheck` and `pnpm --filter @compenso/engine test` run. Package names `@compenso/shared`, `@compenso/engine`.

- [ ] **Step 1: Verify pnpm is available**

Run: `corepack enable 2>/dev/null; pnpm -v || npm i -g pnpm`
Expected: a version number (9.x or 10.x).

- [ ] **Step 2: Create root files**

`pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
  - "apps/*"
```

`package.json`:
```json
{
  "name": "compenso",
  "private": true,
  "scripts": {
    "typecheck": "pnpm -r typecheck",
    "test": "pnpm -r test"
  },
  "devDependencies": {
    "typescript": "^5.5.0"
  }
}
```

`tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

`.gitignore`:
```
node_modules/
dist/
coverage/
.env
.env.*
*.local
.DS_Store
```

`.nvmrc`:
```
22
```

- [ ] **Step 3: Create package manifests**

`packages/shared/package.json`:
```json
{
  "name": "@compenso/shared",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "echo 'no tests (types only)'"
  }
}
```

`packages/shared/tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "include": ["src"]
}
```

`packages/engine/package.json`:
```json
{
  "name": "@compenso/engine",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "./src/index.ts",
  "scripts": {
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest"
  },
  "dependencies": {
    "@compenso/shared": "workspace:*"
  },
  "devDependencies": {
    "vitest": "^3.0.0"
  }
}
```

`packages/engine/tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "include": ["src", "test"]
}
```

`packages/engine/vitest.config.ts`:
```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["test/**/*.test.ts"],
    coverage: { provider: "v8", include: ["src/**"] },
  },
});
```

- [ ] **Step 4: Write ADR-004**

`docs/adr/ADR-004-pure-engine-package.md`:
```markdown
# ADR-004: Pay calculation lives in a pure, zero-dependency package

**Status:** accepted · 2026-07-24

## Decision
All pay math and audit rules live in `@compenso/engine`: pure functions,
no I/O, no runtime dependencies, no clock access. Callers (web app, edge
functions) assemble inputs; the engine only computes.

## Consequences
- Deterministic: same inputs → same outputs; runs are reproducible via
  `calcVersion` stamped at finalize.
- Testable to ~100% coverage without mocks.
- Reusable from browser, Deno edge functions, and a future embed API.
```

- [ ] **Step 5: Install and verify**

Create `packages/shared/src/index.ts` with placeholder `export {};` so typecheck passes, then:

Run: `pnpm install && pnpm -r typecheck`
Expected: install succeeds, typecheck exits 0 for both packages.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: pnpm workspace scaffold (shared + engine packages)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: Shared domain types

**Files:**
- Modify: `packages/shared/src/index.ts` (replace placeholder)

**Interfaces:**
- Produces (used by every later task — exact names):
  `PayType`, `AdjustmentType`, `Severity`, `CompensationInput`, `AdjustmentInput`, `WorkerRunInput`, `RunInputs`, `PayLine`, `WorkerRunResult`, `RunTotals`, `RunResult`, `AuditWorkerInput`, `AuditFinding`, `AuditThresholds`.

- [ ] **Step 1: Write the types**

`packages/shared/src/index.ts`:
```ts
export type PayType = "salary" | "hourly" | "per_job" | "commission" | "tips";
export type AdjustmentType =
  | "bonus"
  | "deduction"
  | "reimbursement"
  | "advance"
  | "advance_repayment";
export type Severity = "blocker" | "warning";

export interface CompensationInput {
  payType: PayType;
  /** hourly: cents/hour · salary: cents/period · amount-based types: unused (0) */
  rateCents: number;
  currency: string; // ISO 4217, e.g. "USD"
}

export interface AdjustmentInput {
  id: string;
  type: AdjustmentType;
  /** always positive; sign is derived from type */
  amountCents: number;
  label: string;
}

export interface WorkerRunInput {
  workerId: string;
  compensation: CompensationInput;
  /** hourly only: approved minutes in period */
  approvedMinutes?: number;
  /** salary proration; both omitted = full period */
  daysActive?: number;
  daysInPeriod?: number;
  /** per_job / commission / tips: individual amounts */
  amountItemsCents?: number[];
  adjustments: AdjustmentInput[];
}

export interface RunInputs {
  periodStart: string; // ISO date
  periodEnd: string;   // ISO date
  /** optional overtime rule: minutes over threshold paid at multiplierPct (150 = 1.5x) */
  overtime?: { thresholdMinutes: number; multiplierPct: number };
  workers: WorkerRunInput[];
}

export interface PayLine {
  kind: "base" | "overtime" | PayType | AdjustmentType;
  label: string;
  /** signed: negative for deductions */
  amountCents: number;
}

export interface WorkerRunResult {
  workerId: string;
  grossCents: number;
  additionsCents: number;
  deductionsCents: number;
  netCents: number;
  /** deduction amount that could not be applied (net floored at 0) */
  shortfallCents: number;
  lines: PayLine[];
}

export interface RunTotals {
  grossCents: number;
  additionsCents: number;
  deductionsCents: number;
  netCents: number;
  workerCount: number;
}

export interface RunResult {
  calcVersion: string;
  totals: RunTotals;
  items: WorkerRunResult[];
}

export interface AuditWorkerInput {
  workerId: string;
  netCents: number;
  /** most-recent-last nets from prior finalized runs (may be empty) */
  priorNetsCents: number[];
  overtimeMinutes: number;
  /** average overtime minutes across prior runs; 0 if none */
  priorOvertimeAvgMinutes: number;
  hasPayoutMethod: boolean;
  /** worker had any approved time or amount items this period */
  hasActivity: boolean;
  unapprovedMinutes: number;
  timeEntries: { date: string; minutes: number }[];
}

export interface AuditThresholds {
  /** warn when |net - lastNet| exceeds this % of lastNet (default 50) */
  netJumpPct: number;
  /** warn when overtime exceeds this % of prior average (default 150) */
  overtimeSpikePct: number;
}

export interface AuditFinding {
  rule:
    | "missing_payout_method"
    | "ghost_worker"
    | "duplicate_time_entry"
    | "net_jump"
    | "overtime_spike"
    | "unapproved_time";
  severity: Severity;
  workerId: string;
  message: string;
}
```

- [ ] **Step 2: Typecheck**

Run: `pnpm --filter @compenso/shared typecheck`
Expected: exit 0.

- [ ] **Step 3: Commit**

```bash
git add packages/shared
git commit -m "feat(shared): core domain types for engine and app

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: Money primitives

**Files:**
- Create: `packages/engine/src/money.ts`
- Test: `packages/engine/test/money.test.ts`
- Create: `docs/adr/ADR-001-money-as-integer-cents.md`

**Interfaces:**
- Produces: `assertCents(v: number, ctx: string): void` (throws on non-integer/negative/unsafe), `addCents(...vals: number[]): number`, `roundHalfUpDiv(numerator: number, denominator: number): number` (integer half-up division), `payForMinutes(rateCentsPerHour: number, minutes: number): number`, `prorate(amountCents: number, activeUnits: number, totalUnits: number): number`.

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/money.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import {
  addCents,
  assertCents,
  payForMinutes,
  prorate,
  roundHalfUpDiv,
} from "../src/money";

describe("assertCents", () => {
  it("accepts non-negative integers", () => {
    expect(() => assertCents(0, "x")).not.toThrow();
    expect(() => assertCents(123456, "x")).not.toThrow();
  });
  it("rejects floats, negatives, NaN, unsafe", () => {
    expect(() => assertCents(1.5, "rate")).toThrow(/rate/);
    expect(() => assertCents(-1, "rate")).toThrow();
    expect(() => assertCents(NaN, "rate")).toThrow();
    expect(() => assertCents(Number.MAX_SAFE_INTEGER + 1, "rate")).toThrow();
  });
});

describe("roundHalfUpDiv", () => {
  it("rounds half up", () => {
    expect(roundHalfUpDiv(5, 2)).toBe(3); // 2.5 -> 3
    expect(roundHalfUpDiv(4, 2)).toBe(2);
    expect(roundHalfUpDiv(1, 3)).toBe(0); // 0.33 -> 0
    expect(roundHalfUpDiv(2, 3)).toBe(1); // 0.66 -> 1
  });
});

describe("payForMinutes", () => {
  it("computes rate * minutes / 60 with single half-up rounding", () => {
    expect(payForMinutes(1500, 60)).toBe(1500); // $15/h * 1h
    expect(payForMinutes(1500, 90)).toBe(2250);
    expect(payForMinutes(1000, 1)).toBe(17); // 16.67 -> 17
  });
});

describe("prorate", () => {
  it("prorates by unit counts, half-up", () => {
    expect(prorate(300000, 15, 30)).toBe(150000);
    expect(prorate(100000, 1, 3)).toBe(33333); // 33333.33 -> 33333
    expect(prorate(100000, 2, 3)).toBe(66667); // 66666.67 -> 66667
  });
  it("full period returns full amount", () => {
    expect(prorate(300000, 30, 30)).toBe(300000);
  });
});

describe("addCents", () => {
  it("sums validated integers", () => {
    expect(addCents(1, 2, 3)).toBe(6);
    expect(() => addCents(1, 0.5)).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — cannot resolve `../src/money`.

- [ ] **Step 3: Implement**

`packages/engine/src/money.ts`:
```ts
export function assertCents(v: number, ctx: string): void {
  if (!Number.isSafeInteger(v) || v < 0) {
    throw new Error(`${ctx}: expected non-negative integer cents, got ${v}`);
  }
}

export function addCents(...vals: number[]): number {
  let total = 0;
  for (const v of vals) {
    assertCents(v, "addCents");
    total += v;
  }
  assertCents(total, "addCents total");
  return total;
}

/** Integer division rounded half-up. Inputs must be non-negative, d > 0. */
export function roundHalfUpDiv(numerator: number, denominator: number): number {
  if (!Number.isSafeInteger(numerator) || numerator < 0) {
    throw new Error(`roundHalfUpDiv: bad numerator ${numerator}`);
  }
  if (!Number.isSafeInteger(denominator) || denominator <= 0) {
    throw new Error(`roundHalfUpDiv: bad denominator ${denominator}`);
  }
  return Math.floor((2 * numerator + denominator) / (2 * denominator));
}

/** Pay for worked minutes at an hourly rate, rounded once at the end. */
export function payForMinutes(rateCentsPerHour: number, minutes: number): number {
  assertCents(rateCentsPerHour, "payForMinutes rate");
  if (!Number.isSafeInteger(minutes) || minutes < 0) {
    throw new Error(`payForMinutes: bad minutes ${minutes}`);
  }
  return roundHalfUpDiv(rateCentsPerHour * minutes, 60);
}

/** Prorate an amount by active/total units (e.g. days), rounded half-up once. */
export function prorate(
  amountCents: number,
  activeUnits: number,
  totalUnits: number,
): number {
  assertCents(amountCents, "prorate amount");
  if (!Number.isSafeInteger(activeUnits) || activeUnits < 0) {
    throw new Error(`prorate: bad activeUnits ${activeUnits}`);
  }
  if (!Number.isSafeInteger(totalUnits) || totalUnits <= 0) {
    throw new Error(`prorate: bad totalUnits ${totalUnits}`);
  }
  if (activeUnits > totalUnits) {
    throw new Error(`prorate: activeUnits ${activeUnits} > totalUnits ${totalUnits}`);
  }
  return roundHalfUpDiv(amountCents * activeUnits, totalUnits);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS (all money tests green).

- [ ] **Step 5: Write ADR-001**

`docs/adr/ADR-001-money-as-integer-cents.md`:
```markdown
# ADR-001: Money is integer minor units (cents)

**Status:** accepted · 2026-07-24

## Decision
Every monetary value in Compenso — DB columns, engine math, API payloads —
is an integer count of the currency's minor unit plus an ISO 4217 code.
Rounding is half-up and applied exactly once per derived line
(`roundHalfUpDiv`), never on intermediates.

## Why
IEEE floats cannot represent 0.1; accumulated float errors in payroll are
real money. Integer math is exact, portable, and testable.

## Consequences
- `assertCents` guards every entry point; non-integer input is a thrown
  error, not a silent coercion.
- Display formatting is a UI concern only.
```

- [ ] **Step 6: Commit**

```bash
git add packages/engine docs/adr/ADR-001-money-as-integer-cents.md
git commit -m "feat(engine): money primitives with half-up integer rounding

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: Base pay — hourly with overtime

**Files:**
- Create: `packages/engine/src/base-pay.ts`
- Test: `packages/engine/test/base-pay-hourly.test.ts`

**Interfaces:**
- Consumes: `payForMinutes`, `roundHalfUpDiv` from `./money.js`; types from `@compenso/shared`.
- Produces: `basePayLines(worker: WorkerRunInput, overtime?: RunInputs["overtime"]): PayLine[]` — hourly branch. (Salary and amount-based branches added in Tasks 5–6 inside the same function.)

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/base-pay-hourly.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import type { WorkerRunInput } from "@compenso/shared";
import { basePayLines } from "../src/base-pay";

function hourlyWorker(minutes: number, rateCents = 1500): WorkerRunInput {
  return {
    workerId: "w1",
    compensation: { payType: "hourly", rateCents, currency: "USD" },
    approvedMinutes: minutes,
    adjustments: [],
  };
}

describe("basePayLines: hourly", () => {
  it("pays approved minutes at rate", () => {
    const lines = basePayLines(hourlyWorker(600)); // 10h @ $15
    expect(lines).toEqual([
      { kind: "base", label: "Hourly pay (600 min)", amountCents: 15000 },
    ]);
  });

  it("zero minutes -> zero base line", () => {
    expect(basePayLines(hourlyWorker(0))).toEqual([
      { kind: "base", label: "Hourly pay (0 min)", amountCents: 0 },
    ]);
  });

  it("missing approvedMinutes throws", () => {
    const w = hourlyWorker(0);
    delete (w as { approvedMinutes?: number }).approvedMinutes;
    expect(() => basePayLines(w)).toThrow(/approvedMinutes/);
  });

  it("splits overtime above threshold at multiplier", () => {
    // 50h with 40h threshold @ $15/h, 1.5x -> base 36000, OT 13500
    const lines = basePayLines(hourlyWorker(3000), {
      thresholdMinutes: 2400,
      multiplierPct: 150,
    });
    expect(lines).toEqual([
      { kind: "base", label: "Hourly pay (2400 min)", amountCents: 36000 },
      { kind: "overtime", label: "Overtime (600 min @ 150%)", amountCents: 13500 },
    ]);
  });

  it("no overtime line when under threshold", () => {
    const lines = basePayLines(hourlyWorker(1200), {
      thresholdMinutes: 2400,
      multiplierPct: 150,
    });
    expect(lines).toHaveLength(1);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — cannot resolve `../src/base-pay`.

- [ ] **Step 3: Implement (hourly branch only)**

`packages/engine/src/base-pay.ts`:
```ts
import type { PayLine, RunInputs, WorkerRunInput } from "@compenso/shared";
import { payForMinutes, roundHalfUpDiv } from "./money";

type OvertimeRule = NonNullable<RunInputs["overtime"]>;

function hourlyLines(w: WorkerRunInput, overtime?: OvertimeRule): PayLine[] {
  const minutes = w.approvedMinutes;
  if (minutes === undefined) {
    throw new Error(`worker ${w.workerId}: hourly requires approvedMinutes`);
  }
  const rate = w.compensation.rateCents;

  if (!overtime || minutes <= overtime.thresholdMinutes) {
    return [
      {
        kind: "base",
        label: `Hourly pay (${minutes} min)`,
        amountCents: payForMinutes(rate, minutes),
      },
    ];
  }

  const baseMinutes = overtime.thresholdMinutes;
  const otMinutes = minutes - baseMinutes;
  const otPay = roundHalfUpDiv(rate * otMinutes * overtime.multiplierPct, 60 * 100);
  return [
    {
      kind: "base",
      label: `Hourly pay (${baseMinutes} min)`,
      amountCents: payForMinutes(rate, baseMinutes),
    },
    {
      kind: "overtime",
      label: `Overtime (${otMinutes} min @ ${overtime.multiplierPct}%)`,
      amountCents: otPay,
    },
  ];
}

export function basePayLines(
  w: WorkerRunInput,
  overtime?: OvertimeRule,
): PayLine[] {
  switch (w.compensation.payType) {
    case "hourly":
      return hourlyLines(w, overtime);
    default:
      throw new Error(
        `worker ${w.workerId}: unsupported payType ${w.compensation.payType}`,
      );
  }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/engine
git commit -m "feat(engine): hourly base pay with overtime split

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: Base pay — salary proration

**Files:**
- Modify: `packages/engine/src/base-pay.ts` (add salary branch)
- Test: `packages/engine/test/base-pay-salary.test.ts`

**Interfaces:**
- Consumes/Produces: same `basePayLines` signature; salary branch uses `prorate` from `./money.js`.

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/base-pay-salary.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import type { WorkerRunInput } from "@compenso/shared";
import { basePayLines } from "../src/base-pay";

function salaryWorker(over?: Partial<WorkerRunInput>): WorkerRunInput {
  return {
    workerId: "w2",
    compensation: { payType: "salary", rateCents: 300000, currency: "USD" },
    adjustments: [],
    ...over,
  };
}

describe("basePayLines: salary", () => {
  it("full period salary when no proration fields", () => {
    expect(basePayLines(salaryWorker())).toEqual([
      { kind: "base", label: "Salary", amountCents: 300000 },
    ]);
  });

  it("prorates mid-period start", () => {
    expect(
      basePayLines(salaryWorker({ daysActive: 15, daysInPeriod: 30 })),
    ).toEqual([
      { kind: "base", label: "Salary (15/30 days)", amountCents: 150000 },
    ]);
  });

  it("throws when only one proration field is present", () => {
    expect(() => basePayLines(salaryWorker({ daysActive: 10 }))).toThrow(
      /daysActive and daysInPeriod/,
    );
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — "unsupported payType salary".

- [ ] **Step 3: Add salary branch**

In `packages/engine/src/base-pay.ts`, add above `basePayLines`:

```ts
function salaryLines(w: WorkerRunInput): PayLine[] {
  const { daysActive, daysInPeriod } = w;
  const hasActive = daysActive !== undefined;
  const hasTotal = daysInPeriod !== undefined;
  if (hasActive !== hasTotal) {
    throw new Error(
      `worker ${w.workerId}: daysActive and daysInPeriod must be provided together`,
    );
  }
  if (!hasActive || !hasTotal) {
    return [{ kind: "base", label: "Salary", amountCents: w.compensation.rateCents }];
  }
  return [
    {
      kind: "base",
      label: `Salary (${daysActive}/${daysInPeriod} days)`,
      amountCents: prorate(w.compensation.rateCents, daysActive, daysInPeriod),
    },
  ];
}
```

Update the import line to include `prorate`:
```ts
import { payForMinutes, prorate, roundHalfUpDiv } from "./money";
```

Add to the switch in `basePayLines`:
```ts
    case "salary":
      return salaryLines(w);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/engine
git commit -m "feat(engine): salary base pay with day-count proration

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 6: Base pay — amount-based types (per_job / commission / tips)

**Files:**
- Modify: `packages/engine/src/base-pay.ts` (add amount branch)
- Test: `packages/engine/test/base-pay-amounts.test.ts`

**Interfaces:**
- Same `basePayLines` signature. Amount-based types read `amountItemsCents` and emit one line per item with `kind` = the pay type.

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/base-pay-amounts.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import type { PayType, WorkerRunInput } from "@compenso/shared";
import { basePayLines } from "../src/base-pay";

function amountWorker(payType: PayType, items?: number[]): WorkerRunInput {
  return {
    workerId: "w3",
    compensation: { payType, rateCents: 0, currency: "USD" },
    amountItemsCents: items,
    adjustments: [],
  };
}

describe("basePayLines: amount-based", () => {
  it("one line per item, kind = pay type", () => {
    expect(basePayLines(amountWorker("per_job", [50000, 25000]))).toEqual([
      { kind: "per_job", label: "per_job payment 1", amountCents: 50000 },
      { kind: "per_job", label: "per_job payment 2", amountCents: 25000 },
    ]);
  });

  it("commission and tips behave the same", () => {
    expect(basePayLines(amountWorker("commission", [12345]))).toEqual([
      { kind: "commission", label: "commission payment 1", amountCents: 12345 },
    ]);
    expect(basePayLines(amountWorker("tips", [700]))).toEqual([
      { kind: "tips", label: "tips payment 1", amountCents: 700 },
    ]);
  });

  it("missing amountItemsCents throws; empty list allowed (zero pay)", () => {
    expect(() => basePayLines(amountWorker("per_job"))).toThrow(/amountItemsCents/);
    expect(basePayLines(amountWorker("per_job", []))).toEqual([]);
  });

  it("rejects non-integer amounts", () => {
    expect(() => basePayLines(amountWorker("tips", [10.5]))).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — "unsupported payType per_job".

- [ ] **Step 3: Add amount branch**

In `packages/engine/src/base-pay.ts`, update the money import to include `assertCents`:
```ts
import { assertCents, payForMinutes, prorate, roundHalfUpDiv } from "./money";
```

Add:
```ts
function amountLines(w: WorkerRunInput): PayLine[] {
  const items = w.amountItemsCents;
  if (items === undefined) {
    throw new Error(
      `worker ${w.workerId}: ${w.compensation.payType} requires amountItemsCents`,
    );
  }
  return items.map((amount, i) => {
    assertCents(amount, `worker ${w.workerId} amount item ${i + 1}`);
    return {
      kind: w.compensation.payType,
      label: `${w.compensation.payType} payment ${i + 1}`,
      amountCents: amount,
    };
  });
}
```

Replace the switch in `basePayLines` with:
```ts
  switch (w.compensation.payType) {
    case "hourly":
      return hourlyLines(w, overtime);
    case "salary":
      return salaryLines(w);
    case "per_job":
    case "commission":
    case "tips":
      return amountLines(w);
  }
```
(The `default` throw is removed — the switch is now exhaustive; TypeScript enforces it.)

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS (all base-pay suites).

- [ ] **Step 5: Commit**

```bash
git add packages/engine
git commit -m "feat(engine): amount-based pay types (per_job, commission, tips)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 7: Adjustments application

**Files:**
- Create: `packages/engine/src/adjustments.ts`
- Test: `packages/engine/test/adjustments.test.ts`

**Interfaces:**
- Produces: `applyAdjustments(grossCents: number, adjustments: AdjustmentInput[]): { additionsCents: number; deductionsCents: number; netCents: number; shortfallCents: number; lines: PayLine[] }`
- Sign rule: `bonus`, `reimbursement`, `advance` add; `deduction`, `advance_repayment` subtract. Net floors at 0; the unapplied remainder is `shortfallCents`.

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/adjustments.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import type { AdjustmentInput } from "@compenso/shared";
import { applyAdjustments } from "../src/adjustments";

const adj = (
  type: AdjustmentInput["type"],
  amountCents: number,
  id = "a1",
): AdjustmentInput => ({ id, type, amountCents, label: type });

describe("applyAdjustments", () => {
  it("no adjustments: net = gross", () => {
    expect(applyAdjustments(10000, [])).toEqual({
      additionsCents: 0,
      deductionsCents: 0,
      netCents: 10000,
      shortfallCents: 0,
      lines: [],
    });
  });

  it("adds bonus/reimbursement/advance, subtracts deduction/advance_repayment", () => {
    const r = applyAdjustments(10000, [
      adj("bonus", 2000, "b"),
      adj("reimbursement", 500, "r"),
      adj("advance", 1000, "a"),
      adj("deduction", 300, "d"),
      adj("advance_repayment", 700, "ar"),
    ]);
    expect(r.additionsCents).toBe(3500);
    expect(r.deductionsCents).toBe(1000);
    expect(r.netCents).toBe(12500);
    expect(r.shortfallCents).toBe(0);
    expect(r.lines).toEqual([
      { kind: "bonus", label: "bonus", amountCents: 2000 },
      { kind: "reimbursement", label: "reimbursement", amountCents: 500 },
      { kind: "advance", label: "advance", amountCents: 1000 },
      { kind: "deduction", label: "deduction", amountCents: -300 },
      { kind: "advance_repayment", label: "advance_repayment", amountCents: -700 },
    ]);
  });

  it("floors net at zero and reports shortfall", () => {
    const r = applyAdjustments(1000, [adj("advance_repayment", 2500)]);
    expect(r.netCents).toBe(0);
    expect(r.deductionsCents).toBe(1000); // only what was actually applied
    expect(r.shortfallCents).toBe(1500);
  });

  it("rejects non-positive adjustment amounts", () => {
    expect(() => applyAdjustments(1000, [adj("bonus", 0)])).toThrow();
    expect(() => applyAdjustments(1000, [adj("bonus", -5)])).toThrow();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — cannot resolve `../src/adjustments`.

- [ ] **Step 3: Implement**

`packages/engine/src/adjustments.ts`:
```ts
import type { AdjustmentInput, PayLine } from "@compenso/shared";
import { assertCents } from "./money";

const ADDITIVE: ReadonlySet<AdjustmentInput["type"]> = new Set([
  "bonus",
  "reimbursement",
  "advance",
]);

export interface AdjustmentResult {
  additionsCents: number;
  deductionsCents: number;
  netCents: number;
  shortfallCents: number;
  lines: PayLine[];
}

export function applyAdjustments(
  grossCents: number,
  adjustments: AdjustmentInput[],
): AdjustmentResult {
  assertCents(grossCents, "applyAdjustments gross");

  let additions = 0;
  let requestedDeductions = 0;
  const lines: PayLine[] = [];

  for (const a of adjustments) {
    assertCents(a.amountCents, `adjustment ${a.id}`);
    if (a.amountCents === 0) {
      throw new Error(`adjustment ${a.id}: amount must be positive`);
    }
    const isAdd = ADDITIVE.has(a.type);
    if (isAdd) additions += a.amountCents;
    else requestedDeductions += a.amountCents;
    lines.push({
      kind: a.type,
      label: a.label,
      amountCents: isAdd ? a.amountCents : -a.amountCents,
    });
  }

  const payable = grossCents + additions;
  const applied = Math.min(requestedDeductions, payable);
  return {
    additionsCents: additions,
    deductionsCents: applied,
    netCents: payable - applied,
    shortfallCents: requestedDeductions - applied,
    lines,
  };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/engine
git commit -m "feat(engine): adjustment application with net floor and shortfall carry

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 8: calculateRun aggregator

**Files:**
- Create: `packages/engine/src/calculate-run.ts`
- Create: `packages/engine/src/index.ts`
- Test: `packages/engine/test/calculate-run.test.ts`

**Interfaces:**
- Produces: `CALC_VERSION = "engine-0.1.0"`, `calculateRun(inputs: RunInputs): RunResult`.
- `packages/engine/src/index.ts` re-exports: `calculateRun`, `CALC_VERSION`, `basePayLines`, `applyAdjustments`, money helpers, and (after Task 9) `runAudit`.

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/calculate-run.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import type { RunInputs } from "@compenso/shared";
import { CALC_VERSION, calculateRun } from "../src/calculate-run";

const inputs: RunInputs = {
  periodStart: "2026-07-01",
  periodEnd: "2026-07-31",
  overtime: { thresholdMinutes: 9600, multiplierPct: 150 },
  workers: [
    {
      workerId: "hourly-1",
      compensation: { payType: "hourly", rateCents: 2000, currency: "USD" },
      approvedMinutes: 10200, // 170h: 160 base + 10 OT
      adjustments: [{ id: "a1", type: "bonus", amountCents: 5000, label: "Spot bonus" }],
    },
    {
      workerId: "salary-1",
      compensation: { payType: "salary", rateCents: 400000, currency: "USD" },
      adjustments: [
        { id: "a2", type: "advance_repayment", amountCents: 50000, label: "Advance" },
      ],
    },
  ],
};

describe("calculateRun", () => {
  it("computes per-worker items and stamps calc version", () => {
    const r = calculateRun(inputs);
    expect(r.calcVersion).toBe(CALC_VERSION);
    expect(r.items).toHaveLength(2);

    const hourly = r.items[0]!;
    // base 9600min@$20/h = 320000; OT 600min@150% = 30000; +5000 bonus
    expect(hourly.grossCents).toBe(350000);
    expect(hourly.netCents).toBe(355000);

    const salary = r.items[1]!;
    expect(salary.grossCents).toBe(400000);
    expect(salary.netCents).toBe(350000);
  });

  it("totals equal the sum of items", () => {
    const r = calculateRun(inputs);
    const sum = (f: (i: (typeof r.items)[number]) => number) =>
      r.items.reduce((acc, i) => acc + f(i), 0);
    expect(r.totals.grossCents).toBe(sum((i) => i.grossCents));
    expect(r.totals.netCents).toBe(sum((i) => i.netCents));
    expect(r.totals.workerCount).toBe(2);
  });

  it("rejects duplicate workerIds", () => {
    const bad: RunInputs = { ...inputs, workers: [inputs.workers[0]!, inputs.workers[0]!] };
    expect(() => calculateRun(bad)).toThrow(/duplicate workerId/);
  });

  it("is deterministic", () => {
    expect(calculateRun(inputs)).toEqual(calculateRun(inputs));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — cannot resolve `../src/calculate-run`.

- [ ] **Step 3: Implement**

`packages/engine/src/calculate-run.ts`:
```ts
import type { RunInputs, RunResult, WorkerRunResult } from "@compenso/shared";
import { applyAdjustments } from "./adjustments";
import { basePayLines } from "./base-pay";
import { addCents } from "./money";

export const CALC_VERSION = "engine-0.1.0";

export function calculateRun(inputs: RunInputs): RunResult {
  const seen = new Set<string>();
  const items: WorkerRunResult[] = inputs.workers.map((w) => {
    if (seen.has(w.workerId)) {
      throw new Error(`duplicate workerId ${w.workerId}`);
    }
    seen.add(w.workerId);

    const baseLines = basePayLines(w, inputs.overtime);
    const grossCents = addCents(...baseLines.map((l) => l.amountCents));
    const adj = applyAdjustments(grossCents, w.adjustments);
    return {
      workerId: w.workerId,
      grossCents,
      additionsCents: adj.additionsCents,
      deductionsCents: adj.deductionsCents,
      netCents: adj.netCents,
      shortfallCents: adj.shortfallCents,
      lines: [...baseLines, ...adj.lines],
    };
  });

  return {
    calcVersion: CALC_VERSION,
    totals: {
      grossCents: addCents(...items.map((i) => i.grossCents)),
      additionsCents: addCents(...items.map((i) => i.additionsCents)),
      deductionsCents: addCents(...items.map((i) => i.deductionsCents)),
      netCents: addCents(...items.map((i) => i.netCents)),
      workerCount: items.length,
    },
    items,
  };
}
```

`packages/engine/src/index.ts`:
```ts
export { CALC_VERSION, calculateRun } from "./calculate-run";
export { basePayLines } from "./base-pay";
export { applyAdjustments } from "./adjustments";
export type { AdjustmentResult } from "./adjustments";
export {
  addCents,
  assertCents,
  payForMinutes,
  prorate,
  roundHalfUpDiv,
} from "./money";
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add packages/engine
git commit -m "feat(engine): calculateRun aggregator with calc versioning

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 9: Pre-run audit rules

**Files:**
- Create: `packages/engine/src/audit.ts`
- Modify: `packages/engine/src/index.ts` (add export)
- Test: `packages/engine/test/audit.test.ts`

**Interfaces:**
- Consumes: `AuditWorkerInput`, `AuditFinding`, `AuditThresholds` from `@compenso/shared`.
- Produces: `DEFAULT_THRESHOLDS: AuditThresholds = { netJumpPct: 50, overtimeSpikePct: 150 }`, `runAudit(workers: AuditWorkerInput[], thresholds?: AuditThresholds): AuditFinding[]`.
- Severity map: `missing_payout_method` = blocker; all other rules = warning.

- [ ] **Step 1: Write the failing tests**

`packages/engine/test/audit.test.ts`:
```ts
import { describe, expect, it } from "vitest";
import type { AuditWorkerInput } from "@compenso/shared";
import { runAudit } from "../src/audit";

function worker(over: Partial<AuditWorkerInput>): AuditWorkerInput {
  return {
    workerId: "w1",
    netCents: 100000,
    priorNetsCents: [],
    overtimeMinutes: 0,
    priorOvertimeAvgMinutes: 0,
    hasPayoutMethod: true,
    hasActivity: true,
    unapprovedMinutes: 0,
    timeEntries: [],
    ...over,
  };
}

const rules = (w: Partial<AuditWorkerInput>) =>
  runAudit([worker(w)]).map((f) => `${f.rule}:${f.severity}`);

describe("runAudit", () => {
  it("clean worker -> no findings", () => {
    expect(runAudit([worker({})])).toEqual([]);
  });

  it("missing payout method is a blocker (only when being paid)", () => {
    expect(rules({ hasPayoutMethod: false })).toEqual([
      "missing_payout_method:blocker",
    ]);
    expect(rules({ hasPayoutMethod: false, netCents: 0 })).toEqual([]);
  });

  it("ghost worker: paid with no activity", () => {
    expect(rules({ hasActivity: false })).toEqual(["ghost_worker:warning"]);
  });

  it("duplicate time entries: same date+minutes twice", () => {
    expect(
      rules({
        timeEntries: [
          { date: "2026-07-01", minutes: 480 },
          { date: "2026-07-01", minutes: 480 },
        ],
      }),
    ).toEqual(["duplicate_time_entry:warning"]);
  });

  it("net jump beyond 50% of last net", () => {
    expect(rules({ netCents: 160000, priorNetsCents: [100000] })).toEqual([
      "net_jump:warning",
    ]);
    expect(rules({ netCents: 140000, priorNetsCents: [100000] })).toEqual([]);
    // no history -> no rule
    expect(rules({ netCents: 160000, priorNetsCents: [] })).toEqual([]);
  });

  it("overtime spike beyond 150% of prior average", () => {
    expect(
      rules({ overtimeMinutes: 400, priorOvertimeAvgMinutes: 200 }),
    ).toEqual(["overtime_spike:warning"]);
    expect(
      rules({ overtimeMinutes: 250, priorOvertimeAvgMinutes: 200 }),
    ).toEqual([]);
  });

  it("unapproved time in period", () => {
    expect(rules({ unapprovedMinutes: 60 })).toEqual(["unapproved_time:warning"]);
  });

  it("multiple findings accumulate across workers", () => {
    const findings = runAudit([
      worker({ hasPayoutMethod: false }),
      worker({ workerId: "w2", unapprovedMinutes: 30 }),
    ]);
    expect(findings).toHaveLength(2);
    expect(findings.map((f) => f.workerId)).toEqual(["w1", "w2"]);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm --filter @compenso/engine test`
Expected: FAIL — cannot resolve `../src/audit`.

- [ ] **Step 3: Implement**

`packages/engine/src/audit.ts`:
```ts
import type {
  AuditFinding,
  AuditThresholds,
  AuditWorkerInput,
} from "@compenso/shared";

export const DEFAULT_THRESHOLDS: AuditThresholds = {
  netJumpPct: 50,
  overtimeSpikePct: 150,
};

export function runAudit(
  workers: AuditWorkerInput[],
  thresholds: AuditThresholds = DEFAULT_THRESHOLDS,
): AuditFinding[] {
  const findings: AuditFinding[] = [];
  for (const w of workers) {
    findings.push(...auditWorker(w, thresholds));
  }
  return findings;
}

function auditWorker(
  w: AuditWorkerInput,
  t: AuditThresholds,
): AuditFinding[] {
  const out: AuditFinding[] = [];
  const add = (rule: AuditFinding["rule"], severity: AuditFinding["severity"], message: string) =>
    out.push({ rule, severity, workerId: w.workerId, message });

  if (w.netCents > 0 && !w.hasPayoutMethod) {
    add("missing_payout_method", "blocker", "Worker is being paid but has no payout method on file");
  }

  if (w.netCents > 0 && !w.hasActivity) {
    add("ghost_worker", "warning", "Nonzero pay with no recorded activity this period");
  }

  const seen = new Set<string>();
  for (const e of w.timeEntries) {
    const key = `${e.date}|${e.minutes}`;
    if (seen.has(key)) {
      add("duplicate_time_entry", "warning", `Duplicate time entry: ${e.minutes} min on ${e.date}`);
      break;
    }
    seen.add(key);
  }

  const lastNet = w.priorNetsCents.at(-1);
  if (lastNet !== undefined && lastNet > 0) {
    const jump = Math.abs(w.netCents - lastNet) * 100;
    if (jump > lastNet * t.netJumpPct) {
      add("net_jump", "warning", `Net pay changed more than ${t.netJumpPct}% vs last run`);
    }
  }

  if (
    w.priorOvertimeAvgMinutes > 0 &&
    w.overtimeMinutes * 100 > w.priorOvertimeAvgMinutes * t.overtimeSpikePct
  ) {
    add("overtime_spike", "warning", `Overtime exceeds ${t.overtimeSpikePct}% of prior average`);
  }

  if (w.unapprovedMinutes > 0) {
    add("unapproved_time", "warning", `${w.unapprovedMinutes} unapproved minutes in period`);
  }

  return out;
}
```

Add to `packages/engine/src/index.ts`:
```ts
export { DEFAULT_THRESHOLDS, runAudit } from "./audit";
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm --filter @compenso/engine test`
Expected: PASS (all suites).

- [ ] **Step 5: Commit**

```bash
git add packages/engine
git commit -m "feat(engine): deterministic pre-run audit rules

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 10: CI workflow, coverage, README

**Files:**
- Create: `.github/workflows/ci.yml`
- Modify: `README.md` (already exists — update Status only)
- Modify: `packages/engine/package.json` (coverage script)

**Interfaces:**
- Produces: `pnpm --filter @compenso/engine test:coverage` and a CI workflow that runs typecheck + tests on push (activates when a remote exists later).

- [ ] **Step 1: Add coverage script and dependency**

In `packages/engine/package.json` scripts add:
```json
"test:coverage": "vitest run --coverage"
```
Run: `pnpm --filter @compenso/engine add -D @vitest/coverage-v8`

- [ ] **Step 2: Run coverage and record the number**

Run: `pnpm --filter @compenso/engine test:coverage`
Expected: PASS; statement coverage on `src/**` ≥ 95%. If below, add tests for the uncovered branches (they will be error guards) before proceeding.

- [ ] **Step 3: Create CI workflow**

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 10
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm -r typecheck
      - run: pnpm -r test
```

- [ ] **Step 4: Update README status**

`README.md` already exists at the repo root. In its **Roadmap** table, change the Phase 1 row's Status cell from `🔨 in progress` to `✅ done — engine fully tested`. Make no other changes.

- [ ] **Step 5: Full verification**

Run: `pnpm -r typecheck && pnpm -r test`
Expected: both exit 0, all suites pass.

Run: `find packages -name "*.ts" -not -path "*/node_modules/*" | xargs wc -l | sort -rn | head -5`
Expected: no source file over 300 lines.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: CI workflow, coverage tooling, README

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

## Phase 1 exit criteria

- `pnpm -r typecheck` and `pnpm -r test` green; engine coverage ≥ 95%.
- Engine has zero runtime deps (check `packages/engine/package.json` — only `@compenso/shared`).
- ADR-001 and ADR-004 committed; README current.
- No file > 300 lines. No remote created.
