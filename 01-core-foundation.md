# usabl Core Foundation: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the verdict core of usabl: a pure `run(deps, config)` that produces
a canonical `Result` with all four verdicts (`verified`, `regression`, `not_covered`,
`approval_required`) plus the idle non-verdict, reachable over in-memory fakes, backed
by a re-verifiable receipt.

**Architecture:** Providers hand the gate `Draft[]`; the gate is the sole verdict
authority; the canonical `Result` JSON is the single source of truth and every output is
a pure projection of it. The whole engine is a pure function over an injected `Deps`
object, so every test runs on fakes with no network and no real browser.

**Tech Stack:** TypeScript (ESM, strict), Node 22, Vitest, tsup, `node:crypto` for
hashing. No runtime deps in this phase.

> **Source of truth:** `../usabl-decisions-2026-08-19.md` and `usabl/docs/ground-truth.md`
> (§4 architecture, §5 contracts, §6 deps, §9 gate, §10 receipt, §22 config). Do not
> commit this plan into the `usabl` repo.

---

## File structure (created by this plan)

```
usabl/
  package.json                 npm package "usabl", type module, one bin, subpath exports
  tsconfig.json                strict ESM
  tsup.config.ts               build entrypoints
  vitest.config.ts             test config
  usabl.config.json            sample config (§22 shape)
  src/
    contracts/index.ts         ALL frozen types (single import surface)
    primitives/
      sortKey.ts               sortBy(items, keyFn): stable, pure
      canonical.ts             canonicalize(value), sha256(text), canonicalHash(value)
      slug.ts                  slug(text): stable token for identity keys
      identity.ts              computeIdentity(draft) -> { elementKey, identityBasis }
    deps/
      fakes.ts                 makeFakeDeps() + fake builders for every Deps member
    guard/index.ts             computeGuardDivergence(deps, guardedPaths) (minimal, local)
    gate/index.ts              gate(input) -> GateOutput (the verdict authority)
    evidence/receipt.ts        mintReceipt(...) -> Receipt
    output/summary.ts          formatSummary(result) -> string (pure projection)
      output/conformance.ts      computeConformance(result) -> ConformanceSummary (Fork 1b, projection)
    run.ts                     run(deps, config) -> Result (orchestrator)
    cli.ts                     thin bin: load config, build deps, run, print, exit
    index.ts                   public API re-exports
  fixtures/golden/             canonical Result snapshots (oracle)
  test/                        mirrors src/
```

**Public API (frozen names used across all later phases):**
- `run(deps: Deps, config: UsablConfig, opts?: RunOptions): Promise<Result>`: `opts` is
  optional and additive; two-argument calls keep working. CI passes
  `{ changedFiles, trustedRef }`.
- `gate(input: GateInput): GateOutput`
- `computeIdentity(draft: Draft): { elementKey: string | null; identityBasis: IdentityBasis }`
- `canonicalize(value: unknown): string`, `sha256(text: string): string`,
  `canonicalHash(value: unknown): string`
- `sortBy<T>(items: T[], keyFn: (t: T) => string): T[]`
- `mintReceipt(deps: Deps, config: UsablConfig, args: ReceiptArgs): Promise<Receipt>`
- `computePolicyHash(deps: Deps, guardedPaths: string[], ref?: string): Promise<string>` :
  the ONLY policy hash definition; minting and verification both call it.
- `formatSummary(result: Result): string`
- `computeConformance(result: Result): ConformanceSummary`
- `makeFakeDeps(overrides?: Partial<FakeDepsSpec>): Deps`
- `makeFakePage(overrides?: Partial<Page>): Page`: every test fake Page comes from here;
  no hand-rolled Page literals in tests.

---

## Task 1: Project scaffold

**Files:**
- Create: `package.json`, `tsconfig.json`, `vitest.config.ts`, `tsup.config.ts`
- Create: `test/smoke.test.ts`

- [ ] **Step 1: Write the failing test**

`test/smoke.test.ts`:
```ts
import { describe, it, expect } from 'vitest';

describe('toolchain', () => {
  it('runs vitest and TypeScript', () => {
    const doubled: number = [1, 2, 3].reduce((a, b) => a + b, 0);
    expect(doubled).toBe(6);
  });
});
```

- [ ] **Step 2: Create `package.json`**

```json
{
  "name": "usabl",
  "version": "0.0.0",
  "description": "Accessibility proof engine",
  "license": "Apache-2.0",
  "type": "module",
  "bin": { "usabl": "./dist/cli.js" },
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" }
  },
  "files": ["dist"],
  "engines": { "node": ">=22" },
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "build": "tsup",
    "check": "npm run typecheck && npm run test"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "tsup": "^8.0.0",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0"
  }
}
```

- [ ] **Step 3: Create `tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022"],
    "types": ["node"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noEmit": true,
    "skipLibCheck": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true
  },
  "include": ["src", "test"]
}
```

- [ ] **Step 4: Create `vitest.config.ts`**

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['test/**/*.test.ts'],
    environment: 'node',
  },
});
```

- [ ] **Step 5: Create `tsup.config.ts`**

```ts
import { defineConfig } from 'tsup';

export default defineConfig({
  entry: { index: 'src/index.ts', cli: 'src/cli.ts' },
  format: ['esm'],
  dts: true,
  clean: true,
  target: 'node22',
});
```

- [ ] **Step 6: Install and run the test**

Run: `npm install && npm test`
Expected: PASS (1 test). If `npm install` cannot reach the registry, stop and resolve
the registry/proxy before continuing: do not skip.

- [ ] **Step 7: Commit**

```bash
git checkout -b feat/core-foundation
git add package.json tsconfig.json vitest.config.ts tsup.config.ts test/smoke.test.ts
git commit -m "chore: scaffold TypeScript + Vitest toolchain"
```

---

## Task 2: Frozen contracts

**Files:**
- Create: `src/contracts/index.ts`
- Test: `test/contracts/contracts.test.ts`

- [ ] **Step 1: Write the failing test** (a runtime shape check so drift is caught)

`test/contracts/contracts.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import type {
  Draft, Finding, Result, Verdict, EvidenceClass, Waiver, EvidenceFloor,
  UsablConfig, GateInput, GateOutput, ConformanceSummary,
} from '../../src/contracts/index.js';

describe('contracts', () => {
  it('constructs a minimal Draft and Finding', () => {
    const draft: Draft = {
      rule: 'button-name', layer: 'axe', severity: 'critical',
      evidenceClass: 'deterministic', screenId: 'clusters', elementPath: 'button:nth-of-type(1)',
      elementName: null, role: 'button', whatUserExperiences: 'x', why: 'y', fix: 'z',
      evidence: {}, confidence: 'fail',
    };
    const finding: Finding = { ...draft, elementKey: null, identityBasis: 'count', status: 'new' };
    expect(finding.status).toBe('new');
  });

  it('enumerates all four verdicts and the preview evidence class', () => {
    const verdicts: Verdict[] = ['verified', 'regression', 'not_covered', 'approval_required'];
    const classes: EvidenceClass[] = ['deterministic', 'preview', 'model-judgment', 'human-confirmed'];
    expect(verdicts).toHaveLength(4);
    expect(classes).toContain('preview');
  });

  it('constructs a conformance summary with all buckets shown', () => {
    const summary: ConformanceSummary = {
      verdict: 'verified', blocked: false,
      deterministic: { newFailures: 0, carried: 1, waived: 0, fixed: 2 },
      judged: { modelJudgment: 3, preview: 1 },
      notEvaluated: { unresolvedFiles: 0, gaps: 0 },
    };
    expect(summary.judged.modelJudgment).toBe(3);
    expect(summary.blocked).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/contracts/contracts.test.ts`
Expected: FAIL: cannot find module `src/contracts/index.js`.

- [ ] **Step 3: Write `src/contracts/index.ts`**

```ts
// ---- verdict + evidence vocabulary (ground-truth §5 plus the preview class) ----
export type Severity = 'critical' | 'serious' | 'moderate' | 'minor';
export type EvidenceClass = 'deterministic' | 'preview' | 'model-judgment' | 'human-confirmed';
export type Verdict = 'verified' | 'regression' | 'not_covered' | 'approval_required';
export type FactSource = 'ax-tree' | 'attribute';
export type IdentityBasis = 'name' | 'structural' | 'count';

// ---- evidence + findings ----
export interface Fact<T = string | null> { value: T; source: FactSource; fromTree: boolean; }
export interface EvidenceFacts {
  name?: Fact;
  role?: Fact;
  state?: Record<string, Fact<unknown>>;
  extra?: Record<string, unknown>;
}
export interface Draft {
  rule: string;
  layer: string;                 // contest ids: 'axe' | 'pf' | 'walk'; stays open
  severity: Severity;
  evidenceClass: EvidenceClass;  // provenance taxonomy: only 'deterministic' (or promoted) mints
                                 //   the gate verdict; model-judgment/preview feed the judgment
                                 //   assessment + conformance summary and never gate
  screenId: string;
  elementPath: string;
  elementName: string | null;
  role: string | null;
  whatUserExperiences: string;
  why: string;
  fix: string;
  evidence: EvidenceFacts;
  confidence: 'fail' | 'unverified';
}
export interface Finding extends Draft {
  elementKey: string | null;     // null when identityBasis is 'count'
  identityBasis: IdentityBasis;
  status: 'new' | 'carried' | 'fixed' | 'waived';
}

// ---- announcement / transcript (voicing lane consumes these in Phase 4) ----
// kind 'live' = text that reached an armed live region after a step, captured from DOM
// mutations rather than from the focused node. Toast and status announcements enter the
// transcript only this way; fromTree is false and source is 'attribute' for live tokens.
export interface AnnouncementToken { kind: 'name' | 'role' | 'state' | 'live'; text: string | null; fromTree: boolean; source: FactSource; }
export interface TranscriptStop { index: number; elementPath: string; announcement: AnnouncementToken[]; }
export interface ScreenScan {
  screenId: string;
  url: string;
  stops: TranscriptStop[];
  drafts: Draft[];
  gaps: CoverageGap[];           // capability skips + scan failures for THIS screen; run() merges into coverage.gaps
}

// ---- coverage ----
export interface AffectedScreen { screenId: string; url: string; provenance: 'route-graph' | 'wide-blast' | 'manual'; importChain?: string[]; }
export interface CoverageGap {
  ref: string;                                                    // surface id, url, provider id, or file path this gap concerns
  state: 'unresolved' | 'not-covered' | 'skipped' | 'capability-denied';
  reason: string;                                                 // why it was not exercised; never empty. Never report coverage that was not achieved.
}
export interface Coverage {
  changedFiles: string[];
  affected: AffectedScreen[];
  unresolvedFiles: string[];
  gaps: CoverageGap[];           // each in-scope surface or check that could not be exercised, each with a reason
  nothingToCheck: boolean;       // no UI-touching files; not a verdict
}
// The gate treats ANY gap as blocking from day one: gaps force not_covered exactly like
// unresolvedFiles. Phase 1 gaps arrive via ScreenScan.gaps; the Phase 3 planner adds
// discovery gaps of its own.

// ---- receipt (ground-truth §5, §10) ----
export interface Receipt {
  schemaVersion: 1;
  sourceTree: string;
  baseRevision: string | null;
  policyHash: string;
  runnerVersion: string;
  scannerVersions: { axeCore: string; playwright: string; chromium: string };
  surfaces: string[];
  coverage: { checked: string[]; notCovered: string[] };
  verdict: 'verified';
  findingsSummary: { new: number; carried: number; fixed: number; unverified: number };
  activeWaivers: number;
  signature?: string;            // slot only; OIDC signing is a documented later seam
  mintedAt: string;
}

// ---- the whole serializable output ----
export interface Result {
  schemaVersion: 'usabl.result.v1';   // versioned contract; every projection echoes it
  verdict: Verdict | null;       // null when nothing to check, and on the disclosed error path (exitCode 4)
  summary: string;
  screens: ScreenScan[];
  coverage: Coverage;
  findings: Finding[];
  receipt: Receipt | null;
  dirtyGuardedPaths: string[];
  exitCode: 0 | 1 | 2 | 3 | 4 | 5;
}
// exitCode: 0 verified or nothing-to-check; 1 regression; 2 approval_required;
// 3 not_covered; 4 unhandled error (fail open with disclosure);
// 5 reserved for a future opt-in advisory soft-gate. The default install never emits 5.

// ---- conformance summary: a NON-GATING projection of Result, never a single score ----
// Always shows every bucket side by side; never hides the not-evaluated denominator. The gate
// verdict stays the authority; this is a read-only view for CI comments and the ACCESSIBILITY.md row.
export interface ConformanceSummary {
  verdict: Verdict | null;                 // echoes Result.verdict (the gate is the authority)
  blocked: boolean;                        // any in-scope deterministic new failure caps the summary
  deterministic: { newFailures: number; carried: number; waived: number; fixed: number };
  judged: { modelJudgment: number; preview: number };  // advisory provenance; never gates
  notEvaluated: { unresolvedFiles: number; gaps: number };
}

// ---- evidence floor + waivers ----
export interface FloorEntry {
  screenId: string;
  layer: string;                 // contest: 'axe' | 'pf' | 'walk'
  rule: string;
  elementKey: string | null;
  identityBasis: IdentityBasis;
  count: number;
}
export interface EvidenceFloor { version: 1; entries: FloorEntry[]; }
export interface Waiver {
  rule: string;
  surface: string;               // matches Finding.screenId
  scope: string;                 // matches Finding.elementKey, or '*' for the whole rule on the surface
  reason: string;
  owner: string;
  approvedBy: string;
  created: string;               // ISO
  expires: string;               // ISO; ignored once past
}
export interface WaiverLedger { version: 1; waivers: Waiver[]; }

// ---- interaction contracts (Phase 4 consumes; frozen day 1) ----
export interface Step { do: string; [key: string]: unknown; }
export interface SpeechObligation {
  class: string;                 // promotion keys on this
  afterStep: number;             // index into steps; checked only in that step's window
  requiredTokens: string[];      // semantic name/role/state/consequence tokens
  focusedRole?: string;
  mustAnnounce: boolean;         // true = consequence must reach a live region (structural gate)
}
export interface InteractionContract {
  contractId: string;
  surfaceId: string;
  task: string;
  steps: Step[];
  obligations: SpeechObligation[];
  maxTabPath?: number;
}

// ---- design intake + docs output (Phase 6 consumes) ----
export type RequirementKind = 'content' | 'flow' | 'doc';
export interface ContentAssertion { type: 'content'; selector: string; expectedText?: string; mustNotBe?: 'decorative' | 'empty'; }
export interface FlowAssertion { type: 'flow'; steps: Array<{ do: string; [key: string]: unknown }>; expectedAnnouncement?: string; }
export interface DocAssertion { type: 'doc'; artifact: 'alt-text-manifest' | 'announcement-snippets' | 'keyboard-paths'; }
export interface Requirement {
  id: string; kind: RequirementKind; surface: string; description: string;
  assertion: ContentAssertion | FlowAssertion | DocAssertion; owner?: string; approved: boolean;
}
export interface RequirementBundle { version: 1; requirements: Requirement[]; }
export interface DocArtifact {
  kind: 'alt-text-manifest' | 'announcement-snippets' | 'keyboard-paths';
  surface: string;
  entries: Array<{ element: string; content: string; status: 'draft' | 'approved'; evidenceRef?: string }>;
  generatedAt: string;
  boundToReceipt?: string;
}

// ---- injected dependencies (ground-truth §6) ----
export interface AxNode { name: string | null; role: string | null; states: Record<string, unknown>; }
export interface ElementRef { selector: string; }
export interface Page {
  gotoReady(): Promise<void>;
  focusBody(): Promise<void>;
  tab(): Promise<void>;
  press(key: string): Promise<void>;
  click(selector: string): Promise<void>;
  activeNode(): Promise<AxNode | null>;
  activePath(): Promise<string>;
  // Honest predicates for interaction rules: identity and containment of the
  // focused element, computed in the page (never by comparing selector strings
  // against DOM paths: those are different string spaces).
  activeElementIs(selector: string): Promise<boolean>;
  activeElementWithin(selector: string): Promise<boolean>;
  // Live-region capture: arm installs a MutationObserver over [aria-live] and
  // role=status/alert/log containers; drain returns and clears the text that
  // reached them since the last drain. This is how toast announcements are observed.
  armAnnouncementCapture(): Promise<void>;
  drainAnnouncements(): Promise<string[]>;
  axAt(selector: string): Promise<AxNode | null>;
  queryAll(selector: string): Promise<ElementRef[]>;
  close(): Promise<void>;
  setViewport(width: number, height: number): Promise<void>;
  setZoom(percent: number): Promise<void>;
  setReducedMotion(enabled: boolean): Promise<void>;
  getComputedStyle(selector: string, property: string): Promise<string>;
  screenshot(selector?: string): Promise<Buffer>;
}
// One warm browser per process: open() creates a fresh context + page; close()
// disposes the shared browser at the end of the run.
export interface BrowserDriver { open(url: string): Promise<Page>; close(): Promise<void>; }
export interface GitReader {
  writeTree(): Promise<string>;
  show(ref: string, path: string): Promise<string | null>;
  statusZ(): Promise<Array<{ code: string; path: string }>>;
  lsFiles(ref: string, prefix: string): Promise<string[]>;   // files under prefix at ref; '' = all
  headRef(): Promise<string>;
}
export interface FsGlob { readFile(path: string): Promise<string | null>; glob(patterns: string[]): Promise<string[]>; }
export interface CheckRunner { scan(screen: { id: string; url: string }): Promise<ScreenScan>; }
export interface Deps {
  clock: () => string;
  browser: BrowserDriver;
  git: GitReader;
  fs: FsGlob;
  checkRunner: CheckRunner;
  runnerVersion: string;
  scannerVersions: { axeCore: string; playwright: string; chromium: string };
}

// ---- config (ground-truth §22) ----
export interface SurfaceConfig { id: string; url: string; files: string[]; }
export interface UsablConfig {
  appBaseUrl: string;
  uiFileGlobs: string[];
  discovery: { routerFile: string; wideBlastGlobs: string[] };
  surfaces: SurfaceConfig[];
  requirements?: string;
  guardedPaths: string[];
  promotedObligations?: string[];   // empty by default; populated only by offline calibration
}

// ---- run() options (optional third argument; two-argument calls keep working) ----
export interface RunOptions {
  changedFiles?: string[];       // override the change set (CI passes the PR diff); default = git status
  trustedRef?: string | null;    // when set, floor/waivers (and in Phase 3 the guarded set) are read from this ref, never the working tree
}

// ---- gate I/O ----
export interface GateInput {
  coverage: Coverage;
  guardDivergedPaths: string[];
  drafts: Draft[];
  floor: EvidenceFloor;
  waivers: Waiver[];
  now: string;                       // clock() value, for waiver expiry
}
export interface GateOutput {
  verdict: Verdict | null;
  findings: Finding[];
  exitCode: 0 | 1 | 2 | 3 | 4 | 5;   // 5 reserved for the opt-in judgment soft-gate (Phase 3+)
  summary: string;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/contracts/contracts.test.ts && npm run typecheck`
Expected: PASS and clean typecheck.

- [ ] **Step 5: Commit**

```bash
git add src/contracts/index.ts test/contracts/contracts.test.ts
git commit -m "feat: freeze cross-module contracts (verdicts, drafts, deps, config)"
```

---

## Task 3: Stable sort primitive

**Files:**
- Create: `src/primitives/sortKey.ts`
- Test: `test/primitives/sortKey.test.ts`

- [ ] **Step 1: Write the failing test**

`test/primitives/sortKey.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { sortBy } from '../../src/primitives/sortKey.js';

describe('sortBy', () => {
  it('sorts by the string key using code-unit order', () => {
    const out = sortBy([{ k: 'b' }, { k: 'a' }, { k: 'c' }], (x) => x.k);
    expect(out.map((x) => x.k)).toEqual(['a', 'b', 'c']);
  });

  it('is stable for equal keys and does not mutate the input', () => {
    const input = [{ k: 'a', n: 1 }, { k: 'a', n: 2 }, { k: 'a', n: 3 }];
    const out = sortBy(input, (x) => x.k);
    expect(out.map((x) => x.n)).toEqual([1, 2, 3]);
    expect(input.map((x) => x.n)).toEqual([1, 2, 3]); // input untouched
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/primitives/sortKey.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/primitives/sortKey.ts`**

```ts
/** Stable sort by a string key, using UTF-16 code-unit order. Returns a new array. */
export function sortBy<T>(items: T[], keyFn: (t: T) => string): T[] {
  return items
    .map((value, index) => ({ value, index, key: keyFn(value) }))
    .sort((a, b) => (a.key < b.key ? -1 : a.key > b.key ? 1 : a.index - b.index))
    .map((x) => x.value);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/primitives/sortKey.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/primitives/sortKey.ts test/primitives/sortKey.test.ts
git commit -m "feat: add stable sortBy primitive"
```

---

## Task 4: Canonical JSON + hashing

**Files:**
- Create: `src/primitives/canonical.ts`
- Test: `test/primitives/canonical.test.ts`

- [ ] **Step 1: Write the failing test**

`test/primitives/canonical.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { canonicalize, sha256, canonicalHash } from '../../src/primitives/canonical.js';

describe('canonicalize', () => {
  it('sorts object keys and is insensitive to input key order', () => {
    expect(canonicalize({ b: 1, a: 2 })).toBe('{"a":2,"b":1}');
    expect(canonicalize({ a: 2, b: 1 })).toBe('{"a":2,"b":1}');
  });

  it('recurses into nested objects and arrays (array order preserved)', () => {
    expect(canonicalize({ z: [{ y: 1, x: 2 }], a: null })).toBe('{"a":null,"z":[{"x":2,"y":1}]}');
  });

  it('drops undefined-valued keys like JSON does', () => {
    expect(canonicalize({ a: undefined, b: 1 })).toBe('{"b":1}');
  });

  it('throws on non-finite numbers', () => {
    expect(() => canonicalize({ a: Number.NaN })).toThrow();
  });
});

describe('sha256 / canonicalHash', () => {
  it('hashes deterministically and independently of key order', () => {
    expect(sha256('abc')).toBe('ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad');
    expect(canonicalHash({ a: 1, b: 2 })).toBe(canonicalHash({ b: 2, a: 1 }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/primitives/canonical.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/primitives/canonical.ts`**

```ts
import { createHash } from 'node:crypto';

/**
 * RFC 8785 (JCS)-style canonical JSON: object keys sorted by UTF-16 code unit,
 * no insignificant whitespace, undefined-valued keys dropped (as JSON does).
 * Restricted to JSON-safe values (no floats needing special formatting appear in
 * Result); non-finite numbers are rejected rather than silently coerced.
 */
export function canonicalize(value: unknown): string {
  return serialize(value);
}

function serialize(v: unknown): string {
  if (v === null) return 'null';
  const t = typeof v;
  if (t === 'boolean') return v ? 'true' : 'false';
  if (t === 'number') {
    if (!Number.isFinite(v as number)) throw new Error('canonicalize: non-finite number');
    return JSON.stringify(v);
  }
  if (t === 'string') return JSON.stringify(v);
  if (Array.isArray(v)) return '[' + v.map((x) => serialize(x === undefined ? null : x)).join(',') + ']';
  if (t === 'object') {
    const obj = v as Record<string, unknown>;
    const keys = Object.keys(obj).filter((k) => obj[k] !== undefined).sort();
    return '{' + keys.map((k) => JSON.stringify(k) + ':' + serialize(obj[k])).join(',') + '}';
  }
  throw new Error('canonicalize: unserializable value of type ' + t);
}

export function sha256(text: string): string {
  return createHash('sha256').update(text, 'utf8').digest('hex');
}

export function canonicalHash(value: unknown): string {
  return sha256(canonicalize(value));
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/primitives/canonical.test.ts`
Expected: PASS (the `sha256('abc')` vector is the standard SHA-256 of "abc").

- [ ] **Step 5: Commit**

```bash
git add src/primitives/canonical.ts test/primitives/canonical.test.ts
git commit -m "feat: add canonical JSON and sha256 hashing"
```

---

## Task 5: Identity primitive

**Files:**
- Create: `src/primitives/slug.ts`, `src/primitives/identity.ts`
- Test: `test/primitives/identity.test.ts`

Identity is layer-independent (§9): it identifies the (screen, rule, element), so dedup
can collapse the same defect across layers. Priority: **name > structural > count**.
Identity-weak rules (unnamed-element checks) can never be keyed by name.

- [ ] **Step 1: Write the failing test**

`test/primitives/identity.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { computeIdentity } from '../../src/primitives/identity.js';
import type { Draft } from '../../src/contracts/index.js';

function draft(over: Partial<Draft>): Draft {
  return {
    rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
    screenId: 'clusters', elementPath: 'main > div:nth-of-type(2) > button', elementName: null,
    role: 'button', whatUserExperiences: '', why: '', fix: '', evidence: {}, confidence: 'fail',
    ...over,
  };
}

describe('computeIdentity', () => {
  it('uses the accessible name when present', () => {
    const id = computeIdentity(draft({ evidence: { name: { value: 'Delete cluster', source: 'ax-tree', fromTree: true } } }));
    expect(id.identityBasis).toBe('name');
    expect(id.elementKey).toBe('clusters|color-contrast|name:delete-cluster');
  });

  it('is count-based (no key) for identity-weak unnamed rules', () => {
    const id = computeIdentity(draft({ rule: 'button-name', evidence: {} }));
    expect(id.identityBasis).toBe('count');
    expect(id.elementKey).toBeNull();
  });

  it('falls back to structural (role + neutralized path) with no name', () => {
    const id = computeIdentity(draft({ role: 'button', evidence: {} }));
    expect(id.identityBasis).toBe('structural');
    expect(id.elementKey).toBe('clusters|color-contrast|struct:button:main>div>button');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/primitives/identity.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/primitives/slug.ts`**

```ts
/** Lowercase, hyphenated, ascii-ish token safe for identity keys. */
export function slug(text: string): string {
  return text
    .trim()
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-+|-+$/g, '');
}
```

- [ ] **Step 4: Write `src/primitives/identity.ts`**

```ts
import type { Draft, IdentityBasis } from '../contracts/index.js';
import { slug } from './slug.js';

/** Rules that assert "this element has no accessible name": never keyable by name. */
export const IDENTITY_WEAK = new Set<string>([
  'button-name',
  'pf-icon-button-name',
  'keyboard-walk-unnamed-interactive',
]);

/** Strip volatile positional detail from an elementPath into a stable structural token. */
function neutralizePath(path: string): string {
  return path
    .replace(/:nth-of-type\(\d+\)/g, '')
    .replace(/:nth-child\(\d+\)/g, '')
    .replace(/\s*>\s*/g, '>')
    .replace(/\s+/g, '');
}

/**
 * Layer-independent identity for a Draft. elementKey excludes the layer so dedup can
 * collapse the same defect across axe/pf/walk. Returns null key for count-based rules.
 */
export function computeIdentity(draft: Draft): { elementKey: string | null; identityBasis: IdentityBasis } {
  const base = `${draft.screenId}|${draft.rule}`;
  if (IDENTITY_WEAK.has(draft.rule)) {
    return { elementKey: null, identityBasis: 'count' };
  }
  const name = draft.evidence.name?.value;
  if (name) {
    return { elementKey: `${base}|name:${slug(name)}`, identityBasis: 'name' };
  }
  const role = draft.role ?? 'unknown';
  return { elementKey: `${base}|struct:${role}:${neutralizePath(draft.elementPath)}`, identityBasis: 'structural' };
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run test/primitives/identity.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/primitives/slug.ts src/primitives/identity.ts test/primitives/identity.test.ts
git commit -m "feat: add layer-independent identity primitive"
```

---

## Task 6: Injected deps + in-memory fakes

**Files:**
- Create: `src/deps/fakes.ts`
- Test: `test/deps/fakes.test.ts`

- [ ] **Step 1: Write the failing test**

`test/deps/fakes.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { ScreenScan } from '../../src/contracts/index.js';

const scan: ScreenScan = { screenId: 'clusters', url: 'http://x/clusters', stops: [], drafts: [], gaps: [] };

describe('makeFakeDeps', () => {
  it('provides a fixed clock and version metadata', () => {
    const deps = makeFakeDeps({ now: '2026-08-19T00:00:00.000Z', runnerVersion: '0.0.0-test' });
    expect(deps.clock()).toBe('2026-08-19T00:00:00.000Z');
    expect(deps.runnerVersion).toBe('0.0.0-test');
  });

  it('returns scripted git status, files, and scans', async () => {
    const deps = makeFakeDeps({
      changed: [{ code: 'M', path: 'fixtures/app/src/ClustersPage.tsx' }],
      files: { 'usabl.config.json': '{}' },
      headContents: { 'usabl.config.json': '{}', 'src/gate/index.ts': 'X' },
      writeTree: 'tree-abc',
      scans: { clusters: scan },
    });
    expect(await deps.git.statusZ()).toEqual([{ code: 'M', path: 'fixtures/app/src/ClustersPage.tsx' }]);
    expect(await deps.git.writeTree()).toBe('tree-abc');
    expect(await deps.fs.readFile('usabl.config.json')).toBe('{}');
    expect(await deps.git.lsFiles('HEAD', 'src/')).toEqual(['src/gate/index.ts']);
    expect(await deps.checkRunner.scan({ id: 'clusters', url: 'http://x/clusters' })).toEqual(scan);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/deps/fakes.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/deps/fakes.ts`**

```ts
import type { Deps, ScreenScan, Page } from '../contracts/index.js';

export interface FakeDepsSpec {
  now: string;
  runnerVersion: string;
  scannerVersions: { axeCore: string; playwright: string; chromium: string };
  changed: Array<{ code: string; path: string }>;
  files: Record<string, string>;          // working-tree contents by path
  headContents: Record<string, string>;   // committed contents by path (git.show, lsFiles, policy hash)
  writeTree: string;                        // git write-tree result
  headRef: string;
  scans: Record<string, ScreenScan>;        // by screenId
}

const DEFAULTS: FakeDepsSpec = {
  now: '2026-01-01T00:00:00.000Z',
  runnerVersion: '0.0.0-test',
  scannerVersions: { axeCore: '0.0.0', playwright: '0.0.0', chromium: '0.0.0' },
  changed: [],
  files: {},
  headContents: {},
  writeTree: 'tree-0000',
  headRef: 'HEAD-0000',
  scans: {},
};

const notUsed = (name: string) => async (): Promise<never> => {
  throw new Error(`fake Page.${name} not used in this test`);
};

/** Complete in-memory Page. Override any member per test; never hand-roll Page literals. */
export function makeFakePage(overrides: Partial<Page> = {}): Page {
  return {
    gotoReady: async () => {}, focusBody: async () => {}, tab: async () => {}, press: async () => {},
    click: async () => {},
    activeNode: async () => null, activePath: async () => 'body',
    activeElementIs: async () => false, activeElementWithin: async () => false,
    armAnnouncementCapture: async () => {}, drainAnnouncements: async () => [],
    axAt: async () => null, queryAll: async () => [], close: async () => {},
    setViewport: async () => {}, setZoom: async () => {}, setReducedMotion: async () => {},
    getComputedStyle: notUsed('getComputedStyle'), screenshot: notUsed('screenshot'),
    ...overrides,
  };
}

export function makeFakeDeps(overrides: Partial<FakeDepsSpec> = {}): Deps {
  const spec: FakeDepsSpec = { ...DEFAULTS, ...overrides };
  return {
    clock: () => spec.now,
    runnerVersion: spec.runnerVersion,
    scannerVersions: spec.scannerVersions,
    browser: { open: async (_url: string) => makeFakePage(), close: async () => {} },
    git: {
      writeTree: async () => spec.writeTree,
      show: async (_ref, path) => spec.headContents[path] ?? null,
      statusZ: async () => spec.changed,
      lsFiles: async (_ref, prefix) =>
        Object.keys(spec.headContents).filter((p) => p.startsWith(prefix)).sort(),
      headRef: async () => spec.headRef,
    },
    fs: {
      readFile: async (path) => spec.files[path] ?? null,
      glob: async (patterns) => Object.keys(spec.files).filter((f) => patterns.some((p) => matchGlob(p, f))),
    },
    checkRunner: {
      scan: async ({ id, url }) => spec.scans[id] ?? { screenId: id, url, stops: [], drafts: [], gaps: [] },
    },
  };
}

/** Minimal glob matcher for fakes: supports '**' and '*'. */
function matchGlob(pattern: string, path: string): boolean {
  const rx = new RegExp('^' + pattern
    .replace(/[.+^${}()|[\]\\]/g, '\\$&')
    .replace(/\*\*/g, '\x00')
    .replace(/\*/g, '[^/]*')
    .replace(/\x00/g, '.*') + '$');
  return rx.test(path);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/deps/fakes.test.ts && npm run typecheck`
Expected: PASS and clean typecheck.

- [ ] **Step 5: Commit**

```bash
git add src/deps/fakes.ts test/deps/fakes.test.ts
git commit -m "feat: add in-memory Deps fakes for tests"
```

---

## Task 7: Gate: evidence filter and verdict priority

**Files:**
- Create: `src/gate/index.ts`
- Test: `test/gate/verdict.test.ts`

This task builds the verdict decision assuming findings are already prepared. Tasks 8–9
add identity/differential and waivers. Verdict priority (§9): guard → coverage →
`regression` (new deterministic fail) → `not_covered` (unverifiable) → `verified`.

- [ ] **Step 1: Write the failing test**

`test/gate/verdict.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { gate } from '../../src/gate/index.js';
import type { Coverage, Draft, EvidenceFloor } from '../../src/contracts/index.js';

const emptyFloor: EvidenceFloor = { version: 1, entries: [] };
const covered: Coverage = {
  changedFiles: ['fixtures/app/src/ClustersPage.tsx'],
  affected: [{ screenId: 'clusters', url: 'http://x/clusters', provenance: 'manual' }],
  unresolvedFiles: [], gaps: [], nothingToCheck: false,
};
function d(over: Partial<Draft>): Draft {
  return {
    rule: 'r', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
    screenId: 'clusters', elementPath: 'button', elementName: 'Save', role: 'button',
    whatUserExperiences: '', why: '', fix: '', evidence: { name: { value: 'Save', source: 'ax-tree', fromTree: true } },
    confidence: 'fail', ...over,
  };
}
const base = { guardDivergedPaths: [] as string[], floor: emptyFloor, waivers: [], now: '2026-01-01T00:00:00.000Z' };

describe('gate verdict', () => {
  it('returns approval_required when a guarded path diverged', () => {
    const out = gate({ ...base, coverage: covered, drafts: [], guardDivergedPaths: ['src/gate'] });
    expect(out.verdict).toBe('approval_required');
    expect(out.exitCode).toBe(2);
    expect(out.findings).toEqual([]);
  });

  it('returns null (idle) when nothing to check', () => {
    const coverage: Coverage = { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: true };
    const out = gate({ ...base, coverage, drafts: [] });
    expect(out.verdict).toBeNull();
    expect(out.exitCode).toBe(0);
  });

  it('regresses on a new deterministic failure', () => {
    const out = gate({ ...base, coverage: covered, drafts: [d({ rule: 'color-contrast' })] });
    expect(out.verdict).toBe('regression');
    expect(out.exitCode).toBe(1);
  });

  it('ignores preview and model-judgment findings for the verdict', () => {
    const out = gate({ ...base, coverage: covered, drafts: [
      d({ rule: 'voicing-x', evidenceClass: 'preview' }),
      d({ rule: 'ai-x', evidenceClass: 'model-judgment' }),
    ] });
    expect(out.verdict).toBe('verified');
    expect(out.exitCode).toBe(0);
    expect(out.findings).toHaveLength(2); // surfaced, not gating
  });

  it('returns not_covered when a deterministic finding is unverified', () => {
    const out = gate({ ...base, coverage: covered, drafts: [d({ rule: 'walk-x', confidence: 'unverified' })] });
    expect(out.verdict).toBe('not_covered');
    expect(out.exitCode).toBe(3);
  });

  it('returns not_covered when UI files did not map to a screen', () => {
    const coverage: Coverage = { ...covered, unresolvedFiles: ['fixtures/app/src/Orphan.tsx'] };
    const out = gate({ ...base, coverage, drafts: [] });
    expect(out.verdict).toBe('not_covered');
  });

  it('returns not_covered when a coverage gap exists (capability denied)', () => {
    const coverage: Coverage = { ...covered, gaps: [
      { ref: 'provider:axe-core', state: 'capability-denied', reason: 'static mode: provider needs live' },
    ] };
    const out = gate({ ...base, coverage, drafts: [] });
    expect(out.verdict).toBe('not_covered');
    expect(out.exitCode).toBe(3);
  });

  it('verifies when there are no gating problems', () => {
    const out = gate({ ...base, coverage: covered, drafts: [] });
    expect(out.verdict).toBe('verified');
    expect(out.exitCode).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/gate/verdict.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/gate/index.ts`**

```ts
import type { Draft, Finding, GateInput, GateOutput } from '../contracts/index.js';
import { sortBy } from '../primitives/sortKey.js';
import { computeIdentity } from '../primitives/identity.js';

const GATES = (c: Draft['evidenceClass']): boolean => c === 'deterministic';

export function gate(input: GateInput): GateOutput {
  // 1. Guard first: any diverged guarded path forces approval_required; harness never runs.
  if (input.guardDivergedPaths.length > 0) {
    return { verdict: 'approval_required', findings: [], exitCode: 2,
      summary: `approval required: ${input.guardDivergedPaths.length} guarded path(s) changed` };
  }
  // 2. Coverage: nothing to check is an idle non-verdict.
  if (input.coverage.nothingToCheck) {
    return { verdict: null, findings: [], exitCode: 0, summary: 'nothing to check (no UI-touching files)' };
  }
  // 3. Build findings (identity + differential + waivers added in Tasks 8–9).
  const findings = buildFindings(input);

  // 4. Verdict is computed only from deterministic, still-active findings.
  const gating = findings.filter((f) => GATES(f.evidenceClass) && f.status !== 'waived' && f.status !== 'fixed');
  const hasNewFail = gating.some((f) => f.confidence === 'fail' && f.status === 'new');
  const hasUnverified =
    gating.some((f) => f.confidence === 'unverified') ||
    input.coverage.unresolvedFiles.length > 0 ||
    input.coverage.gaps.length > 0;   // any gap blocks: never verified on a partial scan

  if (hasNewFail) return { verdict: 'regression', findings, exitCode: 1, summary: verdictSummary('regression', gating) };
  if (hasUnverified) return { verdict: 'not_covered', findings, exitCode: 3, summary: verdictSummary('not_covered', gating) };
  return { verdict: 'verified', findings, exitCode: 0, summary: verdictSummary('verified', gating) };
}

/** Task 8 replaces the body with identity + dedup + differential; Task 9 adds waivers. */
export function buildFindings(input: GateInput): Finding[] {
  const findings = input.drafts.map((draft): Finding => {
    const { elementKey, identityBasis } = computeIdentity(draft);
    return { ...draft, elementKey, identityBasis, status: 'new' };
  });
  return sortBy(findings, findingKey);
}

export function findingKey(f: Finding): string {
  return `${f.screenId}|${f.layer}|${f.rule}|${f.elementKey ?? 'count'}`;
}

function verdictSummary(verdict: string, gating: Finding[]): string {
  return `${verdict}: ${gating.length} gating finding(s)`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/gate/verdict.test.ts`
Expected: PASS (all 8 cases).

- [ ] **Step 5: Commit**

```bash
git add src/gate/index.ts test/gate/verdict.test.ts
git commit -m "feat: add gate verdict priority and evidence-class filter"
```

---

## Task 8: Gate: identity differential and dedup

**Files:**
- Modify: `src/gate/index.ts` (replace `buildFindings`)
- Test: `test/gate/differential.test.ts`

The evidence floor is the accepted deterministic finding set. New identity → `new`
(regresses); identity already in the floor → `carried` (does not regress); floor entry
with no current match → `fixed`. The floor accepts only `confidence: 'fail'` findings:
an `unverified` finding can never be accepted into a floor; it must be resolved or the
surface stays `not_covered`.

Dedup has two mechanisms:

1. **Identity dedup:** drafts sharing the layer-independent identity key collapse to one
   Finding, preferring the PatternFly why/fix.
2. **Equivalence suppression for identity-weak rules:** axe `button-name`,
   `pf-icon-button-name`, and `keyboard-walk-unnamed-interactive` report the same defect
   class (unnamed control) but can never share element keys, because unnamed elements
   only have counts. For each (screen, equivalence class), keep ONLY the drafts from the
   highest-priority layer that fired (pf > axe > walk) and drop the rest. One broken
   kebab produces one finding, and the count ratchet stays stable because the floor
   count always comes from a single layer.

- [ ] **Step 1: Write the failing test**

`test/gate/differential.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { gate } from '../../src/gate/index.js';
import type { Coverage, Draft, EvidenceFloor, Finding } from '../../src/contracts/index.js';

const covered: Coverage = {
  changedFiles: ['x'], affected: [{ screenId: 'clusters', url: 'u', provenance: 'manual' }],
  unresolvedFiles: [], gaps: [], nothingToCheck: false,
};
function d(over: Partial<Draft>): Draft {
  return {
    rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
    screenId: 'clusters', elementPath: 'button', elementName: 'Save', role: 'button',
    whatUserExperiences: '', why: '', fix: '',
    evidence: { name: { value: 'Save', source: 'ax-tree', fromTree: true } }, confidence: 'fail', ...over,
  };
}
const base = { guardDivergedPaths: [] as string[], waivers: [], now: '2026-01-01T00:00:00.000Z', coverage: covered };
const find = (out: { findings: Finding[] }, rule: string) => out.findings.filter((f) => f.rule === rule);

describe('gate differential', () => {
  it('marks a finding already in the floor as carried and does not regress', () => {
    const floor: EvidenceFloor = { version: 1, entries: [
      { screenId: 'clusters', layer: 'axe', rule: 'color-contrast', elementKey: 'clusters|color-contrast|name:save', identityBasis: 'name', count: 1 },
    ] };
    const out = gate({ ...base, floor, drafts: [d({})] });
    expect(find(out, 'color-contrast')[0]!.status).toBe('carried');
    expect(out.verdict).toBe('verified');
  });

  it('marks a disappeared floor entry as fixed', () => {
    const floor: EvidenceFloor = { version: 1, entries: [
      { screenId: 'clusters', layer: 'axe', rule: 'gone-rule', elementKey: 'clusters|gone-rule|name:save', identityBasis: 'name', count: 1 },
    ] };
    const out = gate({ ...base, floor, drafts: [] });
    expect(find(out, 'gone-rule')[0]!.status).toBe('fixed');
    expect(out.verdict).toBe('verified');
  });

  it('dedups the same defect across layers by identity, preferring the pf why/fix', () => {
    const floor: EvidenceFloor = { version: 1, entries: [] };
    const axe = d({ layer: 'axe', why: 'axe why', fix: 'axe fix' });
    const pf = d({ layer: 'pf', why: 'pf why', fix: 'pf fix' });
    const out = gate({ ...base, floor, drafts: [axe, pf] });
    const kept = find(out, 'color-contrast');
    expect(kept).toHaveLength(1);
    expect(kept[0]!.why).toBe('pf why');
    expect(kept[0]!.fix).toBe('pf fix');
  });

  it('suppresses duplicate unnamed-control reports, keeping only the strongest layer', () => {
    // One unnamed kebab fires axe button-name, pf-icon-button-name, AND the walk rule.
    // Without suppression the fixture oracle would count one defect three times.
    const floor: EvidenceFloor = { version: 1, entries: [] };
    const drafts = [
      d({ rule: 'button-name', layer: 'axe', evidence: {} }),
      d({ rule: 'pf-icon-button-name', layer: 'pf', evidence: {} }),
      d({ rule: 'keyboard-walk-unnamed-interactive', layer: 'walk', evidence: {} }),
    ];
    const out = gate({ ...base, floor, drafts });
    const unnamed = out.findings.filter((f) => f.identityBasis === 'count' && f.status !== 'fixed');
    expect(unnamed).toHaveLength(1);
    expect(unnamed[0]!.layer).toBe('pf');
    expect(out.verdict).toBe('regression'); // still one real new defect
  });

  it('regresses when a count-based rule increases over the floor', () => {
    const floor: EvidenceFloor = { version: 1, entries: [
      { screenId: 'clusters', layer: 'axe', rule: 'button-name', elementKey: null, identityBasis: 'count', count: 1 },
    ] };
    const drafts = [d({ rule: 'button-name', evidence: {} }), d({ rule: 'button-name', evidence: {}, elementPath: 'button2' })];
    const out = gate({ ...base, floor, drafts });
    expect(out.verdict).toBe('regression');
  });

  it('does not regress when a count-based rule matches the floor count', () => {
    const floor: EvidenceFloor = { version: 1, entries: [
      { screenId: 'clusters', layer: 'axe', rule: 'button-name', elementKey: null, identityBasis: 'count', count: 1 },
    ] };
    const out = gate({ ...base, floor, drafts: [d({ rule: 'button-name', evidence: {} })] });
    expect(out.verdict).toBe('verified');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/gate/differential.test.ts`
Expected: FAIL: carried/fixed/dedup behavior not implemented.

- [ ] **Step 3: Replace `buildFindings` in `src/gate/index.ts`**

Replace the Task-7 `buildFindings` (keep the rest of the file) with:
```ts
import type { Draft, Finding, FloorEntry, GateInput, GateOutput } from '../contracts/index.js';
// (keep the existing sortBy, computeIdentity imports and gate()/findingKey()/verdictSummary())

/** Prefer pf over axe when both fire on the same defect. */
function preferLayer(a: Finding, b: Finding): Finding {
  if (a.layer === 'pf') return a;
  if (b.layer === 'pf') return b;
  return a;
}

/** Identity-weak rules that report the same defect class from different layers. */
const RULE_EQUIV: Record<string, string> = {
  'button-name': 'unnamed-control',
  'pf-icon-button-name': 'unnamed-control',
  'keyboard-walk-unnamed-interactive': 'unnamed-control',
};
const LAYER_PRIORITY: Record<string, number> = { pf: 0, axe: 1, walk: 2 };

/**
 * Equivalence suppression: unnamed elements cannot be keyed, so cross-layer duplicates
 * of the same control are indistinguishable. For each (screen, equivalence class), keep
 * only the drafts from the highest-priority layer that fired. This keeps the finding
 * list and the count ratchet stable: the floor count always comes from one layer.
 */
function suppressEquivalents(findings: Finding[]): Finding[] {
  const bestLayer = new Map<string, number>();
  for (const f of findings) {
    const cls = RULE_EQUIV[f.rule];
    if (!cls || f.identityBasis !== 'count') continue;
    const key = `${f.screenId}|${cls}`;
    const rank = LAYER_PRIORITY[f.layer] ?? 9;
    const cur = bestLayer.get(key);
    if (cur === undefined || rank < cur) bestLayer.set(key, rank);
  }
  return findings.filter((f) => {
    const cls = RULE_EQUIV[f.rule];
    if (!cls || f.identityBasis !== 'count') return true;
    return (LAYER_PRIORITY[f.layer] ?? 9) === bestLayer.get(`${f.screenId}|${cls}`);
  });
}

/** Layer-independent key for cross-layer dedup and floor comparison. */
function identityKey(f: { screenId: string; rule: string; elementKey: string | null }): string {
  return `${f.screenId}|${f.rule}|${f.elementKey ?? 'count'}`;
}

export function buildFindings(input: GateInput): Finding[] {
  // 1. Draft -> Finding with identity, then suppress cross-layer equivalents.
  const raw = suppressEquivalents(input.drafts.map((draft: Draft): Finding => {
    const { elementKey, identityBasis } = computeIdentity(draft);
    return { ...draft, elementKey, identityBasis, status: 'new' };
  }));

  // 2. Dedup across layers by identity, preferring the pf why/fix.
  const byIdentity = new Map<string, Finding>();
  const countByGroup = new Map<string, number>();
  for (const f of raw) {
    const key = identityKey(f);
    if (f.identityBasis === 'count') {
      countByGroup.set(key, (countByGroup.get(key) ?? 0) + 1);
    }
    const existing = byIdentity.get(key);
    byIdentity.set(key, existing ? preferLayer(existing, f) : f);
  }

  // 3. Differential vs the evidence floor.
  const floorByKey = new Map<string, FloorEntry>();
  for (const e of input.floor.entries) floorByKey.set(identityKey(e), e);

  const findings: Finding[] = [];
  for (const [key, f] of byIdentity) {
    const floor = floorByKey.get(key);
    if (!floor) { findings.push({ ...f, status: 'new' }); continue; }
    if (f.identityBasis === 'count') {
      const now = countByGroup.get(key) ?? 0;
      findings.push({ ...f, status: now > floor.count ? 'new' : 'carried' });
    } else {
      findings.push({ ...f, status: 'carried' });
    }
  }

  // 4. Floor entries with no current match are fixed (surfaced, never gate).
  for (const [key, e] of floorByKey) {
    if (!byIdentity.has(key)) {
      findings.push({
        rule: e.rule, layer: e.layer, severity: 'minor', evidenceClass: 'deterministic',
        screenId: e.screenId, elementPath: '', elementName: null, role: null,
        whatUserExperiences: '', why: '', fix: '', evidence: {}, confidence: 'fail',
        elementKey: e.elementKey, identityBasis: e.identityBasis, status: 'fixed',
      });
    }
  }

  return sortBy(findings, findingKey);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/gate/differential.test.ts test/gate/verdict.test.ts`
Expected: PASS (both files: Task 7 behavior still holds).

- [ ] **Step 5: Commit**

```bash
git add src/gate/index.ts test/gate/differential.test.ts
git commit -m "feat: add evidence-floor differential and cross-layer dedup"
```

---

## Task 9: Gate: waivers

**Files:**
- Modify: `src/gate/index.ts` (apply waivers inside `buildFindings`)
- Test: `test/gate/waivers.test.ts`

An active, unexpired waiver matching `(rule, surface, scope)` sets `status: 'waived'`
(so it does not gate). An expired waiver is ignored, so the finding reverts to `new`.

- [ ] **Step 1: Write the failing test**

`test/gate/waivers.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { gate } from '../../src/gate/index.js';
import type { Coverage, Draft, EvidenceFloor, Waiver } from '../../src/contracts/index.js';

const covered: Coverage = {
  changedFiles: ['x'], affected: [{ screenId: 'clusters', url: 'u', provenance: 'manual' }],
  unresolvedFiles: [], gaps: [], nothingToCheck: false,
};
const floor: EvidenceFloor = { version: 1, entries: [] };
const draft: Draft = {
  rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
  screenId: 'clusters', elementPath: 'button', elementName: 'Save', role: 'button',
  whatUserExperiences: '', why: '', fix: '',
  evidence: { name: { value: 'Save', source: 'ax-tree', fromTree: true } }, confidence: 'fail',
};
const waiver = (over: Partial<Waiver>): Waiver => ({
  rule: 'color-contrast', surface: 'clusters', scope: 'clusters|color-contrast|name:save',
  reason: 'tracked', owner: 'team', approvedBy: 'owner',
  created: '2026-01-01T00:00:00.000Z', expires: '2026-12-31T00:00:00.000Z', ...over,
});
const base = { guardDivergedPaths: [] as string[], coverage: covered, floor, drafts: [draft] };

describe('gate waivers', () => {
  it('waives a matching active finding so it does not regress', () => {
    const out = gate({ ...base, waivers: [waiver({})], now: '2026-06-01T00:00:00.000Z' });
    expect(out.findings.find((f) => f.rule === 'color-contrast')!.status).toBe('waived');
    expect(out.verdict).toBe('verified');
  });

  it('ignores an expired waiver so the finding regresses', () => {
    const out = gate({ ...base, waivers: [waiver({ expires: '2026-02-01T00:00:00.000Z' })], now: '2026-06-01T00:00:00.000Z' });
    expect(out.findings.find((f) => f.rule === 'color-contrast')!.status).toBe('new');
    expect(out.verdict).toBe('regression');
  });

  it('matches a wildcard scope for the whole rule on the surface', () => {
    const out = gate({ ...base, waivers: [waiver({ scope: '*' })], now: '2026-06-01T00:00:00.000Z' });
    expect(out.verdict).toBe('verified');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/gate/waivers.test.ts`
Expected: FAIL: waivers not applied yet.

- [ ] **Step 3: Add waiver application to `buildFindings`**

At the end of `buildFindings`, before the final `sortBy`, insert waiver application and
include the import for `Waiver`:
```ts
// (add Waiver to the type import from ../contracts/index.js)

// Apply active, unexpired waivers to already-differential findings.
function applyWaivers(findings: Finding[], waivers: Waiver[], now: string): Finding[] {
  const active = waivers.filter((w) => w.expires > now);
  return findings.map((f) => {
    if (f.status === 'fixed') return f;
    const matched = active.some((w) =>
      w.rule === f.rule && w.surface === f.screenId && (w.scope === '*' || w.scope === f.elementKey),
    );
    return matched ? { ...f, status: 'waived' } : f;
  });
}
```
Then change the final return of `buildFindings` from:
```ts
  return sortBy(findings, findingKey);
```
to:
```ts
  return sortBy(applyWaivers(findings, input.waivers, input.now), findingKey);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/gate/`
Expected: PASS (verdict, differential, waivers all green).

- [ ] **Step 5: Commit**

```bash
git add src/gate/index.ts test/gate/waivers.test.ts
git commit -m "feat: apply active waivers in the gate"
```

---

## Task 10: Minimal local guard

**Files:**
- Create: `src/guard/index.ts`
- Test: `test/guard/guard.test.ts`

The local guard is tamper-evident (§10): each guarded path's working-tree content must
match HEAD. Divergence → `approval_required`. Phase 1 handles FILE paths only; the
sample config lists concrete files. Phase 3 hardens the guard: unconditional config
self-guarding, directory expansion (so `src/gate` guards every file under it), session
pinning, and the CI trusted-ref path.

- [ ] **Step 1: Write the failing test**

`test/guard/guard.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { computeGuardDivergence } from '../../src/guard/index.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';

describe('computeGuardDivergence', () => {
  it('reports no divergence when guarded files match HEAD', async () => {
    const deps = makeFakeDeps({
      files: { 'usabl.config.json': '{"a":1}', 'src/gate/index.ts': 'X' },
      headContents: { 'usabl.config.json': '{"a":1}', 'src/gate/index.ts': 'X' },
    });
    expect(await computeGuardDivergence(deps, ['usabl.config.json', 'src/gate/index.ts'])).toEqual([]);
  });

  it('reports a guarded path whose working tree differs from HEAD', async () => {
    const deps = makeFakeDeps({
      files: { 'usabl.config.json': '{"a":2}' },
      headContents: { 'usabl.config.json': '{"a":1}' },
    });
    expect(await computeGuardDivergence(deps, ['usabl.config.json'])).toEqual(['usabl.config.json']);
  });

  it('treats a guarded path missing from HEAD as diverged', async () => {
    const deps = makeFakeDeps({ files: { 'new.json': '{}' }, headContents: {} });
    expect(await computeGuardDivergence(deps, ['new.json'])).toEqual(['new.json']);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/guard/guard.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/guard/index.ts`**

```ts
import type { Deps } from '../contracts/index.js';

/**
 * Local tamper-evident guard: a guarded path diverges when its working-tree content
 * does not exactly equal its HEAD content (including being absent from HEAD).
 * FILE paths only here. Phase 3 adds unconditional config self-guarding, directory
 * expansion, and session pinning; do not pass directory paths to this function.
 */
export async function computeGuardDivergence(deps: Deps, guardedPaths: string[]): Promise<string[]> {
  const diverged: string[] = [];
  for (const path of guardedPaths) {
    const [working, head] = await Promise.all([deps.fs.readFile(path), deps.git.show('HEAD', path)]);
    if (working === null && head === null) continue; // guarded path not present anywhere: nothing to compare
    if (working !== head) diverged.push(path);
  }
  return diverged.sort();
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/guard/guard.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/guard/index.ts test/guard/guard.test.ts
git commit -m "feat: add minimal local tamper-evident guard"
```

---

## Task 11: Receipt minting

**Files:**
- Create: `src/evidence/receipt.ts`
- Test: `test/evidence/receipt.test.ts`

A receipt is minted only for a `verified` verdict, binding three hashes (§10): the
working-tree hash (`git write-tree`), a `policyHash` over the guarded paths' committed
contents, and the `runnerVersion`. `computePolicyHash` defined here is THE single policy
hash in the product: sha256 over canonicalized, sorted `[path, sha256(committed content)]`
pairs. Phase 3's `verifyReceipt` calls this same function with the same inputs; no second
algorithm may exist. Phase 3 passes the expanded guarded set (directories resolved to
files) by handing `mintReceipt` a config whose `guardedPaths` are already expanded.

- [ ] **Step 1: Write the failing test**

`test/evidence/receipt.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { mintReceipt, computePolicyHash } from '../../src/evidence/receipt.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://127.0.0.1:5173', uiFileGlobs: ['fixtures/app/src/**'],
  discovery: { routerFile: 'x', wideBlastGlobs: [] }, surfaces: [],
  guardedPaths: ['usabl.config.json', 'src/gate/index.ts'],
};

describe('computePolicyHash', () => {
  it('is deterministic and changes only when committed guarded content changes', async () => {
    const deps1 = makeFakeDeps({ headContents: { 'usabl.config.json': 'cfg-v1', 'src/gate/index.ts': 'X' } });
    const deps2 = makeFakeDeps({ headContents: { 'usabl.config.json': 'cfg-v2', 'src/gate/index.ts': 'X' } });
    const a = await computePolicyHash(deps1, config.guardedPaths);
    const b = await computePolicyHash(deps1, config.guardedPaths);
    const c = await computePolicyHash(deps2, config.guardedPaths);
    expect(a).toBe(b);
    expect(a).toMatch(/^[0-9a-f]{64}$/);
    expect(a).not.toBe(c);
  });

  it('does not incorporate the source tree (the hashes stay separate)', async () => {
    const deps1 = makeFakeDeps({ headContents: { 'usabl.config.json': 'cfg' }, writeTree: 'tree-1' });
    const deps2 = makeFakeDeps({ headContents: { 'usabl.config.json': 'cfg' }, writeTree: 'tree-2' });
    expect(await computePolicyHash(deps1, config.guardedPaths))
      .toBe(await computePolicyHash(deps2, config.guardedPaths));
  });
});

describe('mintReceipt', () => {
  it('binds source tree, policy hash, and runner version', async () => {
    const deps = makeFakeDeps({
      now: '2026-08-19T12:00:00.000Z', writeTree: 'tree-abc', runnerVersion: '0.0.0-test',
      headContents: { 'usabl.config.json': 'cfg-v1', 'src/gate/index.ts': 'X' },
    });
    const receipt = await mintReceipt(deps, config, {
      surfaces: ['cli'], checked: ['clusters'], notCovered: [],
      findingsSummary: { new: 0, carried: 1, fixed: 0, unverified: 0 }, activeWaivers: 0,
    });
    expect(receipt.verdict).toBe('verified');
    expect(receipt.sourceTree).toBe('tree-abc');
    expect(receipt.runnerVersion).toBe('0.0.0-test');
    expect(receipt.mintedAt).toBe('2026-08-19T12:00:00.000Z');
    expect(receipt.policyHash).toBe(await computePolicyHash(deps, config.guardedPaths));
    expect(receipt.baseRevision).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/evidence/receipt.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/evidence/receipt.ts`**

```ts
import type { Deps, Receipt, UsablConfig } from '../contracts/index.js';
import { sortBy } from '../primitives/sortKey.js';
import { canonicalHash, sha256 } from '../primitives/canonical.js';

export interface ReceiptArgs {
  surfaces: string[];
  checked: string[];
  notCovered: string[];
  findingsSummary: Receipt['findingsSummary'];
  activeWaivers: number;
  baseRevision?: string | null;
}

/**
 * THE policy hash: sha256 over canonicalized, sorted [path, sha256(committed content)]
 * pairs. A path absent from the ref hashes the empty string. Minting and verification
 * both call this function; no other policy hash definition exists anywhere.
 */
export async function computePolicyHash(
  deps: Deps,
  guardedPaths: string[],
  ref = 'HEAD',
): Promise<string> {
  const pairs = await Promise.all(
    guardedPaths.map(async (path) => {
      const content = await deps.git.show(ref, path);
      return [path, sha256(content ?? '')] as const;
    }),
  );
  return canonicalHash(sortBy([...pairs], ([path]) => path));
}

export async function mintReceipt(deps: Deps, config: UsablConfig, args: ReceiptArgs): Promise<Receipt> {
  const [sourceTree, policy] = await Promise.all([
    deps.git.writeTree(),
    computePolicyHash(deps, config.guardedPaths),
  ]);
  return {
    schemaVersion: 1,
    sourceTree,
    baseRevision: args.baseRevision ?? null,
    policyHash: policy,
    runnerVersion: deps.runnerVersion,
    scannerVersions: deps.scannerVersions,
    surfaces: sortBy([...args.surfaces], (s) => s),
    coverage: { checked: sortBy([...args.checked], (s) => s), notCovered: sortBy([...args.notCovered], (s) => s) },
    verdict: 'verified',
    findingsSummary: args.findingsSummary,
    activeWaivers: args.activeWaivers,
    mintedAt: deps.clock(),
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/evidence/receipt.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/evidence/receipt.ts test/evidence/receipt.test.ts
git commit -m "feat: mint receipts binding source tree, policy hash, runner version"
```

---

## Task 12: The `run()` orchestrator

**Files:**
- Create: `src/run.ts`, `src/index.ts`
- Test: `test/run.test.ts`

`run()` is the pure engine: compute coverage from changed files + config, compute guard
divergence, scan affected surfaces via the injected `CheckRunner`, read floor/waivers
via `fs`, call the gate, mint a receipt on `verified`, and assemble the `Result`.
(Route-graph coverage is Phase 3; here coverage is the direct config mapping.)

- [ ] **Step 1: Write the failing test**

`test/run.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { run } from '../src/run.js';
import { makeFakeDeps } from '../src/deps/fakes.js';
import type { Draft, ScreenScan, UsablConfig } from '../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://127.0.0.1:5173',
  uiFileGlobs: ['fixtures/app/src/**'],
  discovery: { routerFile: 'fixtures/app/src/App.tsx', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: 'http://127.0.0.1:5173/clusters', files: ['fixtures/app/src/ClustersPage.tsx'] }],
  guardedPaths: ['usabl.config.json'],
};
const failDraft: Draft = {
  rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
  screenId: 'clusters', elementPath: 'button', elementName: 'Save', role: 'button',
  whatUserExperiences: '', why: '', fix: '',
  evidence: { name: { value: 'Save', source: 'ax-tree', fromTree: true } }, confidence: 'fail',
};
const scanWith = (drafts: Draft[]): ScreenScan => ({ screenId: 'clusters', url: config.surfaces[0]!.url, stops: [], drafts, gaps: [] });
const guardOk = { files: { 'usabl.config.json': '{}' }, headContents: { 'usabl.config.json': '{}' } };

describe('run', () => {
  it('is idle (verdict null, exit 0) when no UI files changed', async () => {
    const deps = makeFakeDeps({ ...guardOk, changed: [{ code: 'M', path: 'README.md' }] });
    const r = await run(deps, config);
    expect(r.verdict).toBeNull();
    expect(r.exitCode).toBe(0);
    expect(r.coverage.nothingToCheck).toBe(true);
  });

  it('verifies and mints a receipt when an affected surface is clean', async () => {
    const deps = makeFakeDeps({ ...guardOk, writeTree: 'tree-1',
      changed: [{ code: 'M', path: 'fixtures/app/src/ClustersPage.tsx' }],
      scans: { clusters: scanWith([]) } });
    const r = await run(deps, config);
    expect(r.verdict).toBe('verified');
    expect(r.receipt?.sourceTree).toBe('tree-1');
    expect(r.exitCode).toBe(0);
  });

  it('regresses (exit 1) and mints no receipt on a new failure', async () => {
    const deps = makeFakeDeps({ ...guardOk,
      changed: [{ code: 'M', path: 'fixtures/app/src/ClustersPage.tsx' }],
      scans: { clusters: scanWith([failDraft]) } });
    const r = await run(deps, config);
    expect(r.verdict).toBe('regression');
    expect(r.receipt).toBeNull();
    expect(r.exitCode).toBe(1);
  });

  it('is not_covered (exit 3) when a UI file maps to no surface', async () => {
    const deps = makeFakeDeps({ ...guardOk,
      changed: [{ code: 'M', path: 'fixtures/app/src/Orphan.tsx' }],
      scans: {} });
    const r = await run(deps, config);
    expect(r.verdict).toBe('not_covered');
    expect(r.coverage.unresolvedFiles).toContain('fixtures/app/src/Orphan.tsx');
    expect(r.exitCode).toBe(3);
  });

  it('is approval_required (exit 2) when a guarded path diverged', async () => {
    const deps = makeFakeDeps({
      files: { 'usabl.config.json': '{"tampered":true}' }, headContents: { 'usabl.config.json': '{}' },
      changed: [{ code: 'M', path: 'usabl.config.json' }] });
    const r = await run(deps, config);
    expect(r.verdict).toBe('approval_required');
    expect(r.dirtyGuardedPaths).toEqual(['usabl.config.json']);
    expect(r.exitCode).toBe(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/run.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/run.ts`**

```ts
import type {
  Coverage, Deps, EvidenceFloor, Finding, Result, RunOptions, ScreenScan, UsablConfig, Waiver, WaiverLedger,
} from './contracts/index.js';
import { computeGuardDivergence } from './guard/index.js';
import { gate } from './gate/index.js';
import { mintReceipt } from './evidence/receipt.js';

const EMPTY_FLOOR: EvidenceFloor = { version: 1, entries: [] };

function matchGlob(pattern: string, path: string): boolean {
  const rx = new RegExp('^' + pattern
    .replace(/[.+^${}()|[\]\\]/g, '\\$&')
    .replace(/\*\*/g, '\x00').replace(/\*/g, '[^/]*').replace(/\x00/g, '.*') + '$');
  return rx.test(path);
}

/** Direct config-mapping coverage. Phase 3 replaces this with route-graph discovery. */
function computeCoverage(config: UsablConfig, changed: string[]): Coverage {
  const uiFiles = changed.filter((f) => config.uiFileGlobs.some((g) => matchGlob(g, f)));
  if (uiFiles.length === 0) {
    return { changedFiles: changed, affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: true };
  }
  const affected = config.surfaces
    .filter((s) => s.files.some((sf) => uiFiles.includes(sf)))
    .map((s) => ({ screenId: s.id, url: s.url, provenance: 'manual' as const }));
  const mapped = new Set(config.surfaces.flatMap((s) => s.files));
  const unresolvedFiles = uiFiles.filter((f) => !mapped.has(f));
  return { changedFiles: changed, affected, unresolvedFiles, gaps: [], nothingToCheck: false };
}

/**
 * Policy files (floor, waivers) come from the working tree by default. When
 * opts.trustedRef is set (the CI path), they are read from that ref instead, so a
 * PR cannot influence the policy it is judged against.
 */
async function readPolicyJson<T>(deps: Deps, path: string, trustedRef: string | null): Promise<T | null> {
  const raw = trustedRef !== null
    ? await deps.git.show(trustedRef, path)
    : await deps.fs.readFile(path);
  if (raw === null) return null;
  return JSON.parse(raw) as T;
}

export async function run(deps: Deps, config: UsablConfig, opts: RunOptions = {}): Promise<Result> {
  try {
    const trustedRef = opts.trustedRef ?? null;
    const changed = opts.changedFiles ?? (await deps.git.statusZ()).map((c) => c.path);
    const baseCoverage = computeCoverage(config, changed);
    const guardDivergedPaths = await computeGuardDivergence(deps, config.guardedPaths);

    // Scan each affected surface fully (never sample).
    const screens: ScreenScan[] = [];
    if (guardDivergedPaths.length === 0 && !baseCoverage.nothingToCheck) {
      for (const s of baseCoverage.affected) {
        screens.push(await deps.checkRunner.scan({ id: s.screenId, url: s.url }));
      }
    }
    const drafts = screens.flatMap((s) => s.drafts);
    // Per-screen gaps (capability skips, scan failures) merge into coverage before the gate.
    const coverage: Coverage = {
      ...baseCoverage,
      gaps: [...baseCoverage.gaps, ...screens.flatMap((s) => s.gaps)],
    };

    const floor = (await readPolicyJson<EvidenceFloor>(deps, '.usabl-evidence.json', trustedRef)) ?? EMPTY_FLOOR;
    const ledger = await readPolicyJson<WaiverLedger>(deps, '.usabl-waivers.json', trustedRef);
    const waivers: Waiver[] = ledger?.waivers ?? [];

    const gated = gate({ coverage, guardDivergedPaths, drafts, floor, waivers, now: deps.clock() });

    const receipt = gated.verdict === 'verified'
      ? await mintReceipt(deps, config, {
          surfaces: ['cli'],
          checked: coverage.affected.map((a) => a.screenId),
          notCovered: coverage.unresolvedFiles,
          findingsSummary: summarize(gated.findings),
          activeWaivers: gated.findings.filter((f) => f.status === 'waived').length,
        })
      : null;

    return {
      schemaVersion: 'usabl.result.v1',
      verdict: gated.verdict, summary: gated.summary, screens, coverage,
      findings: gated.findings, receipt, dirtyGuardedPaths: guardDivergedPaths, exitCode: gated.exitCode,
    };
  } catch (err) {
    // Fail open with disclosure (§9), never a silent pass. formatSummary renders
    // exitCode 4 as an ERROR headline; verdict null here does NOT mean idle.
    return {
      schemaVersion: 'usabl.result.v1',
      verdict: null, summary: `unhandled error: ${(err as Error).message}`, screens: [],
      coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
      findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 4,
    };
  }
}

function summarize(findings: Finding[]) {
  return {
    new: findings.filter((f) => f.status === 'new').length,
    carried: findings.filter((f) => f.status === 'carried').length,
    fixed: findings.filter((f) => f.status === 'fixed').length,
    unverified: findings.filter((f) => f.confidence === 'unverified').length,
  };
}
```

- [ ] **Step 4: Write `src/index.ts` (public API)**

```ts
export * from './contracts/index.js';
export { run } from './run.js';
export { gate } from './gate/index.js';
export { mintReceipt, computePolicyHash } from './evidence/receipt.js';
export { computeGuardDivergence } from './guard/index.js';
export { computeIdentity, IDENTITY_WEAK } from './primitives/identity.js';
export { canonicalize, sha256, canonicalHash } from './primitives/canonical.js';
export { sortBy } from './primitives/sortKey.js';
export { makeFakeDeps, makeFakePage } from './deps/fakes.js';
export { formatSummary } from './output/summary.js';
export { computeConformance } from './output/conformance.js';
```

Note: `formatSummary` (Task 13) and `computeConformance` (Task 14) do not exist yet; if
running the typecheck before those tasks, temporarily omit those last two export lines and
add them back as each module lands.

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run test/run.test.ts`
Expected: PASS (idle, verified+receipt, regression, not_covered, approval_required).

- [ ] **Step 6: Commit**

```bash
git add src/run.ts src/index.ts test/run.test.ts
git commit -m "feat: add pure run() orchestrator over injected Deps"
```

---

## Task 13: Output projection + CLI

**Files:**
- Create: `src/output/summary.ts`, `src/cli.ts`
- Test: `test/output/summary.test.ts`

`formatSummary` is a pure projection of `Result` (never re-derives findings). The CLI is
a thin wrapper: load config, build deps, `run()`, print, exit with `result.exitCode`.

- [ ] **Step 1: Write the failing test**

`test/output/summary.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { formatSummary } from '../../src/output/summary.js';
import type { Result } from '../../src/contracts/index.js';

const baseResult = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1',
  verdict: 'verified', summary: 'verified: 0 gating finding(s)', screens: [],
  coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('formatSummary', () => {
  it('renders the verdict headline', () => {
    expect(formatSummary(baseResult({}))).toContain('VERIFIED');
  });

  it('lists gating findings for a regression', () => {
    const r = baseResult({
      verdict: 'regression', exitCode: 1,
      findings: [{
        rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
        screenId: 'clusters', elementPath: 'button', elementName: 'Save', role: 'button',
        whatUserExperiences: 'Low contrast text', why: '', fix: 'Raise contrast to 4.5:1',
        evidence: {}, confidence: 'fail', elementKey: 'k', identityBasis: 'name', status: 'new',
      }],
    });
    const out = formatSummary(r);
    expect(out).toContain('REGRESSION');
    expect(out).toContain('color-contrast');
    expect(out).toContain('clusters');
  });

  it('names the idle state distinctly', () => {
    expect(formatSummary(baseResult({ verdict: null, summary: 'nothing to check (no UI-touching files)' })))
      .toContain('nothing to check');
  });

  it('renders an unhandled error distinctly, never as idle', () => {
    const out = formatSummary(baseResult({ verdict: null, exitCode: 4, summary: 'unhandled error: boom' }));
    expect(out).toContain('ERROR');
    expect(out).not.toContain('IDLE');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/output/summary.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/output/summary.ts`**

```ts
import type { Result } from '../contracts/index.js';

const HEADLINE: Record<string, string> = {
  verified: 'VERIFIED', regression: 'REGRESSION', not_covered: 'NOT COVERED', approval_required: 'APPROVAL REQUIRED',
};

/** Pure projection of a Result to a human summary. Never re-derives findings. */
export function formatSummary(result: Result): string {
  const lines: string[] = [];
  // exitCode 4 is the disclosed fail-open path; it must never read as a calm idle.
  const head = result.exitCode === 4
    ? 'ERROR (fail open, disclosed)'
    : result.verdict === null ? 'IDLE' : HEADLINE[result.verdict] ?? result.verdict;
  lines.push(`usabl: ${head} - ${result.summary}`);

  const gating = result.findings.filter(
    (f) => f.evidenceClass === 'deterministic' && (f.status === 'new' || f.status === 'carried'),
  );
  for (const f of gating) {
    lines.push(`  [${f.status}] ${f.screenId} · ${f.layer}/${f.rule} (${f.severity}): ${f.whatUserExperiences}`);
    if (f.fix) lines.push(`      fix: ${f.fix}`);
  }
  if (result.dirtyGuardedPaths.length > 0) {
    lines.push(`  guarded paths changed: ${result.dirtyGuardedPaths.join(', ')}`);
  }
  if (result.receipt) lines.push(`  receipt: sourceTree ${result.receipt.sourceTree} @ ${result.receipt.mintedAt}`);
  return lines.join('\n');
}
```

- [ ] **Step 4: Write `src/cli.ts`**

```ts
#!/usr/bin/env node
import { readFile } from 'node:fs/promises';
import { run } from './run.js';
import { formatSummary } from './output/summary.js';
import type { Deps, UsablConfig } from './contracts/index.js';

async function loadConfig(path = 'usabl.config.json'): Promise<UsablConfig> {
  return JSON.parse(await readFile(path, 'utf8')) as UsablConfig;
}

/**
 * Build real Deps. Full implementations (Playwright browser, git plumbing, axe/pf/walk
 * CheckRunner) land in Phases 2–3. This wiring point stays stable.
 */
async function buildDeps(_config: UsablConfig): Promise<Deps> {
  throw new Error('real Deps are implemented in Phases 2–3; use makeFakeDeps in tests until then');
}

export async function main(argv: string[] = process.argv.slice(2)): Promise<number> {
  const command = argv[0] ?? 'check';
  if (command !== 'check') {
    // Usage errors are the error family (4), never 2: exit 2 means approval_required.
    process.stderr.write(`unknown command: ${command}\n`);
    return 4;
  }
  const config = await loadConfig();
  const deps = await buildDeps(config);
  const result = await run(deps, config);
  process.stdout.write(formatSummary(result) + '\n');
  return result.exitCode;
}

// Only run when invoked directly as the bin.
if (import.meta.url === `file://${process.argv[1]}`) {
  main().then((code) => process.exit(code)).catch((err) => {
    process.stderr.write(`usabl: ${(err as Error).message}\n`);
    process.exit(4);
  });
}
```

- [ ] **Step 5: Run test + typecheck + build**

Run: `npx vitest run test/output/summary.test.ts && npm run typecheck && npm run build`
Expected: tests PASS, clean typecheck, `dist/index.js` and `dist/cli.js` produced.
(Re-add the `formatSummary` export line in `src/index.ts` from Task 12 Step 4 now.)

- [ ] **Step 6: Commit**

```bash
git add src/output/summary.ts src/cli.ts src/index.ts test/output/summary.test.ts
git commit -m "feat: add Result summary projection and CLI entrypoint"
```

---

## Task 14: Conformance summary projection

**Files:**
- Create: `src/output/conformance.ts`
- Test: `test/output/conformance.test.ts`

`computeConformance` is a second pure projection of `Result` (Fork 1b): a non-gating,
three-bucket honest view (verified / judged / not-evaluated) that never collapses into a
single score and never hides the not-evaluated denominator. The gate verdict stays the
authority; this only reshapes what `run()` already produced, so it never re-derives findings.

- [ ] **Step 1: Write the failing test**

`test/output/conformance.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { computeConformance } from '../../src/output/conformance.js';
import type { Finding, Result } from '../../src/contracts/index.js';

function finding(over: Partial<Finding>): Finding {
  return {
    rule: 'r', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
    screenId: 'clusters', elementPath: 'button', elementName: null, role: 'button',
    whatUserExperiences: '', why: '', fix: '', evidence: {}, confidence: 'fail',
    elementKey: null, identityBasis: 'count', status: 'new', ...over,
  };
}
const result = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1',
  verdict: 'verified', summary: '', screens: [],
  coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('computeConformance', () => {
  it('counts deterministic buckets and flags a blocker on a new failure', () => {
    const c = computeConformance(result({
      verdict: 'regression', exitCode: 1,
      findings: [
        finding({ status: 'new', confidence: 'fail' }),
        finding({ status: 'carried' }),
        finding({ status: 'waived' }),
        finding({ status: 'fixed' }),
      ],
    }));
    expect(c.deterministic).toEqual({ newFailures: 1, carried: 1, waived: 1, fixed: 1 });
    expect(c.blocked).toBe(true);
    expect(c.verdict).toBe('regression');
  });

  it('separates judged (model-judgment + preview) from the gating buckets', () => {
    const c = computeConformance(result({
      findings: [
        finding({ evidenceClass: 'model-judgment', status: 'new' }),
        finding({ evidenceClass: 'preview', status: 'new' }),
      ],
    }));
    expect(c.judged).toEqual({ modelJudgment: 1, preview: 1 });
    expect(c.deterministic.newFailures).toBe(0);
    expect(c.blocked).toBe(false); // judged findings never block
  });

  it('surfaces the not-evaluated denominator from coverage', () => {
    const c = computeConformance(result({
      verdict: 'not_covered', exitCode: 3,
      coverage: { changedFiles: ['x'], affected: [], unresolvedFiles: ['a.tsx', 'b.tsx'], gaps: [], nothingToCheck: false },
    }));
    expect(c.notEvaluated.unresolvedFiles).toBe(2);
  });

  it('counts coverage gaps in the not-evaluated bucket', () => {
    const c = computeConformance(result({
      verdict: 'not_covered', exitCode: 3,
      coverage: {
        changedFiles: ['x'], affected: [], unresolvedFiles: [],
        gaps: [
          { ref: 'clusters', state: 'capability-denied', reason: 'static mode: provider needs live' },
          { ref: 'fixtures/app/src/Orphan.tsx', state: 'not-covered', reason: 'file maps to no surface' },
        ],
        nothingToCheck: false,
      },
    }));
    expect(c.notEvaluated.gaps).toBe(2);
    expect(c.notEvaluated.unresolvedFiles).toBe(0);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/output/conformance.test.ts`
Expected: FAIL: module not found.

- [ ] **Step 3: Write `src/output/conformance.ts`**

```ts
import type { ConformanceSummary, Finding, Result } from '../contracts/index.js';

const isDeterministic = (f: Finding): boolean => f.evidenceClass === 'deterministic';

/**
 * Pure projection of Result into the non-gating conformance summary (Fork 1b).
 * Never re-derives findings, never collapses to a single score, never hides
 * not-evaluated. The gate verdict (result.verdict) remains the authority.
 */
export function computeConformance(result: Result): ConformanceSummary {
  const det = result.findings.filter(isDeterministic);
  const newFailures = det.filter((f) => f.status === 'new' && f.confidence === 'fail').length;
  return {
    verdict: result.verdict,
    blocked: newFailures > 0,
    deterministic: {
      newFailures,
      carried: det.filter((f) => f.status === 'carried').length,
      waived: det.filter((f) => f.status === 'waived').length,
      fixed: det.filter((f) => f.status === 'fixed').length,
    },
    judged: {
      modelJudgment: result.findings.filter((f) => f.evidenceClass === 'model-judgment').length,
      preview: result.findings.filter((f) => f.evidenceClass === 'preview').length,
    },
    notEvaluated: { unresolvedFiles: result.coverage.unresolvedFiles.length, gaps: result.coverage.gaps.length },
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/output/conformance.test.ts && npm run typecheck`
Expected: PASS and clean typecheck.

- [ ] **Step 5: Commit**

```bash
git add src/output/conformance.ts test/output/conformance.test.ts
git commit -m "feat: add non-gating conformance summary projection (Fork 1b)"
```

---

## Task 15: Golden oracle

**Files:**
- Create: `usabl.config.json`, `test/golden/oracle.test.ts`, `fixtures/golden/*.json`
- Test: `test/golden/oracle.test.ts`

The oracle pins the canonical `Result` for each verdict scenario over fakes, so any
later change that alters the output is caught. Because fakes give a fixed clock and a
fixed `writeTree`, the canonical `Result` is fully deterministic.

- [ ] **Step 1: Create `usabl.config.json`** (repo-root sample, §22 shape)

```json
{
  "appBaseUrl": "http://127.0.0.1:5173",
  "uiFileGlobs": ["fixtures/app/src/**"],
  "discovery": {
    "routerFile": "fixtures/app/src/App.tsx",
    "wideBlastGlobs": ["fixtures/app/src/main.tsx", "fixtures/app/src/App.tsx", "fixtures/app/index.html"]
  },
  "surfaces": [
    { "id": "clusters", "url": "http://127.0.0.1:5173/clusters", "files": ["fixtures/app/src/ClustersPage.tsx"] }
  ],
  "requirements": "requirements/",
  "guardedPaths": ["usabl.config.json", "src/gate/index.ts", "src/guard/index.ts", ".usabl-evidence.json", ".usabl-waivers.json"],
  "promotedObligations": []
}
```

Phase 1's minimal guard compares file paths only, so the sample lists concrete files.
Phase 3 adds directory expansion and switches these entries to whole directories
(`src/gate`, `src/guard`, `src/providers/rulepack`, `requirements/`).

- [ ] **Step 2: Write the failing test**

`test/golden/oracle.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { readFile, writeFile } from 'node:fs/promises';
import { run } from '../../src/run.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import { canonicalize } from '../../src/primitives/canonical.js';
import type { Draft, ScreenScan, UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://127.0.0.1:5173', uiFileGlobs: ['fixtures/app/src/**'],
  discovery: { routerFile: 'fixtures/app/src/App.tsx', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: 'http://127.0.0.1:5173/clusters', files: ['fixtures/app/src/ClustersPage.tsx'] }],
  guardedPaths: ['usabl.config.json'],
};
const fail: Draft = {
  rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
  screenId: 'clusters', elementPath: 'button', elementName: 'Save', role: 'button',
  whatUserExperiences: 'Low contrast', why: 'ratio 2:1', fix: 'Raise to 4.5:1',
  evidence: { name: { value: 'Save', source: 'ax-tree', fromTree: true } }, confidence: 'fail',
};
const scan = (drafts: Draft[]): ScreenScan => ({ screenId: 'clusters', url: config.surfaces[0]!.url, stops: [], drafts, gaps: [] });
const guardOk = { files: { 'usabl.config.json': '{}' }, headContents: { 'usabl.config.json': '{}' }, writeTree: 'tree-fixed', now: '2026-08-19T00:00:00.000Z' };

const scenarios: Record<string, () => ReturnType<typeof makeFakeDeps>> = {
  idle: () => makeFakeDeps({ ...guardOk, changed: [{ code: 'M', path: 'README.md' }] }),
  verified: () => makeFakeDeps({ ...guardOk, changed: [{ code: 'M', path: 'fixtures/app/src/ClustersPage.tsx' }], scans: { clusters: scan([]) } }),
  regression: () => makeFakeDeps({ ...guardOk, changed: [{ code: 'M', path: 'fixtures/app/src/ClustersPage.tsx' }], scans: { clusters: scan([fail]) } }),
  not_covered: () => makeFakeDeps({ ...guardOk, changed: [{ code: 'M', path: 'fixtures/app/src/Orphan.tsx' }] }),
  approval_required: () => makeFakeDeps({ files: { 'usabl.config.json': '{"x":1}' }, headContents: { 'usabl.config.json': '{}' }, writeTree: 'tree-fixed', now: guardOk.now, changed: [{ code: 'M', path: 'usabl.config.json' }] }),
};

const UPDATE = process.env.UPDATE_GOLDEN === '1';

describe('golden oracle', () => {
  for (const [name, mkDeps] of Object.entries(scenarios)) {
    it(`canonical Result is stable for ${name}`, async () => {
      const result = await run(mkDeps(), config);
      const canonical = canonicalize(result);
      const path = new URL(`../../fixtures/golden/${name}.json`, import.meta.url);
      if (UPDATE) { await writeFile(path, canonical + '\n', 'utf8'); return; }
      const expected = (await readFile(path, 'utf8')).trim();
      expect(canonical).toBe(expected);
    });
  }
});
```

- [ ] **Step 3: Generate the golden files, then verify they lock**

Run: `UPDATE_GOLDEN=1 npx vitest run test/golden/oracle.test.ts`
Then inspect each `fixtures/golden/*.json` and confirm the verdicts are correct:
`idle` → `"verdict":null`; `verified` → `"verdict":"verified"` with a receipt;
`regression` → `"verdict":"regression"`; `not_covered` → `"verdict":"not_covered"`;
`approval_required` → `"verdict":"approval_required"`.
Then run without the flag: `npx vitest run test/golden/oracle.test.ts`
Expected: PASS (all five locked).

- [ ] **Step 4: Full suite green**

Run: `npm run check`
Expected: typecheck clean, all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add usabl.config.json test/golden/oracle.test.ts fixtures/golden
git commit -m "test: pin canonical Result golden oracle for all verdicts"
```

---

## Phase exit criteria

- [ ] `run()` over fakes produces every verdict: `verified` (with receipt), `regression`,
  `not_covered`, `approval_required`, and the idle `verdict: null`.
- [ ] The gate is the only module that constructs a verdict.
- [ ] `preview` and `model-judgment` findings surface and feed the judgment assessment +
  conformance summary, but never change the gate verdict.
- [ ] `computeConformance` projects a Result into the three honest buckets (verified /
  judged / not-evaluated) with a blocker flag, never a single score.
- [ ] The canonical `Result` is snapshot-stable (golden oracle green).
- [ ] `npm run check` passes; `npm run build` emits `dist/index.js` + `dist/cli.js`.

## Self-review notes (done while writing)

- **Spec coverage:** verdict priority, evidence-class filter, identity (name/structural/
  count), dedup preferring pf, evidence-floor ratchet, waivers with expiry, receipt three
  hashes, idle vs not_covered, fail-open exit 4: all mapped to tasks. Coverage discovery
  (route graph), the real browser/git/CheckRunner Deps, and CI trusted-ref are explicitly
  deferred to Phases 2–3 and called out at their wiring points.
- **Type consistency:** `GateInput`/`GateOutput`, `Finding.status`, `FloorEntry`,
  `Waiver`, `ReceiptArgs`, and `findingKey`/`identityKey` names are used identically across
  Tasks 7–12.
- **Deferred-but-named:** `buildDeps()` in the CLI throws with a pointer to Phases 2–3
  rather than pretending to work: honest, not a silent stub.
- **Open reconciliation (not a blocker):** the decision log wrote the floor layer as
  `'keyboard'`; this plan uses `'walk'` per ground-truth §9 and keeps `layer: string`.
  Fold the corrected value into `ground-truth.md` during the docs reconciliation.
- **2026-08-20 reconciliation applied:** gaps gate from day one (any `coverage.gaps`
  entry forces `not_covered`); `computePolicyHash` is the single policy hash and is
  exported; `run()` takes an optional `RunOptions` third argument (changedFiles override
  + trustedRef policy reads) without breaking two-argument callers; equivalence
  suppression collapses cross-layer unnamed-control duplicates; `makeFakePage` is the
  only source of fake Pages; `GitReader.lsFiles` replaces `lsTree`; exitCode 4 renders
  as an ERROR headline, never IDLE; unknown CLI commands exit 4, not 2.
