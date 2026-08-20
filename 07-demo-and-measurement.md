# Demo Fixture and Real-App Measurement Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the `usabl-app` PatternFly v6 fixture (the repo does not exist yet), plant the hero bug (modal focus not returned on close) behind a `?variant=` switch, prove the CLEAN variant yields zero findings through the full provider stack (the false-positive oracle), drive the broken→fixed verdict flip end to end through the real engine with a receipt that re-verifies, and run a measurement-only pass against Fleet Insights that reports coverage and gaps without gating anything.

**Architecture (all frozen contracts consumed VERBATIM from `usabl` Phase 1):**
- Everything imports `Result`, `Receipt`, `Coverage`, `CoverageGap`, `Finding`, `Draft`, `ScreenScan` (with `gaps`), `Deps`, `UsablConfig`, `RunOptions` from `usabl`'s `src/contracts/index.ts`. This plan invents NO types, NO alternate `gate()` signature, NO `baseFindings` mechanism: the ratchet is the committed evidence floor, exactly as Phases 1 and 3 built it.
- The five-outcome golden oracle over fakes already lives in Phase 1 Task 15 and is not duplicated here. This phase adds the two things fakes cannot give: a real browser flip and a real-app measurement.
- Hero bug: `pf-modal-focus-return`, detected by the Phase 2 interaction probe (click the trigger, Escape, `activeElementIs(trigger)`). WCAG 2.4.3, structural, and felt: a screen-reader user loses their place.
- Broken vs fixed states are `?variant=broken|fixed` in the fixture app. No rebuild between demo states.
- Fleet Insights: measurement-only. No CI, no gating, no receipt. Session via Playwright `storageState` exported once from a real login. Never committed.

**Tech Stack:**
- `usabl-app`: React 18 + Vite + PatternFly 6 + react-router-dom, Vitest + @testing-library/react + jsdom for component tests.
- `usabl`: the engine (Phases 1-6), Playwright + CDP driver from Phase 2.

---

## Task 1: Scaffold `usabl-app` (the repo does not exist yet)

**Files:** the whole `usabl-app` repo skeleton.

> **Operator step first:** creating the `usabl-dev/usabl-app` remote is repo creation
> and stays with the operator. The agent scaffolds LOCALLY (`git init`) and stops;
> pushing happens after the operator creates the private remote.

- [ ] **Step 1: Scaffold the app**

```bash
npm create vite@latest usabl-app -- --template react-ts
cd usabl-app
npm install @patternfly/react-core react-router-dom
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

Vitest config: `environment: 'jsdom'`, include `src/**/*.test.tsx`.

- [ ] **Step 2: App structure** (real routes on a shared shell so coverage fan-out is
  demonstrable; grow toward the full ground-truth fixture as detection breadth lands):

```
usabl-app/
  src/
    App.tsx                  router: /, /clusters, /settings on a shared AppShell
    components/AppShell.tsx  masthead + sidebar (PF6 Page)
    components/StatusBadge.tsx  shared by Overview and Clusters (fan-out proof)
    components/DemoModal.tsx unchanged between variants except the planted defect
    pages/Overview.tsx
    pages/Clusters.tsx       hosts the hero bug
    pages/Settings.tsx       IDENTICAL in both variants (false-positive control)
    lib/variant.ts           readVariant(): 'broken' | 'fixed' from ?variant=
  usabl.config.json          the engine gates THIS repo (see Step 3)
  usabl.routes.json          sidecar route manifest with real entry files
  .claude/settings.json      Stop hook → node ../usabl/dist/stop-hook-runner.js (demo wiring)
  .gitignore                 includes .usabl/ and storageState*.json
```

- [ ] **Step 3: Install usabl into the fixture repo.** `usabl.config.json` (frozen §22
  shape): `appBaseUrl: "http://127.0.0.1:5173"`, `uiFileGlobs: ["src/**"]`, discovery
  routerFile `src/App.tsx` with wide-blast globs (`src/App.tsx`, `src/main.tsx`,
  `src/**/*.css`, `index.html`, `vite.config.ts`), one manual surface entry per page,
  guardedPaths `["usabl.config.json", ".usabl-evidence.json", ".usabl-waivers.json"]`.
  `usabl.routes.json` maps `clusters → src/pages/Clusters.tsx`, etc. Commit empty
  floor and waiver ledgers so the guard has a clean anchor.

- [ ] **Step 4: Commit** `chore: scaffold usabl-app fixture (PF6, routes, usabl installed)`

---

## Task 2: Fixture: broken and fixed modal states

**Files:**
- `usabl-app/src/components/DemoModal.tsx`
- `usabl-app/src/pages/Clusters.tsx`
- `usabl-app/src/lib/variant.ts`
- `usabl-app/src/components/DemoModal.test.tsx`

- [ ] **Step 1: Write the failing component test**

```tsx
// usabl-app/src/components/DemoModal.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { DemoModal } from './DemoModal.js';

function withTrigger(run: (trigger: HTMLButtonElement) => void): void {
  const trigger = document.createElement('button');
  trigger.textContent = 'Open';
  document.body.appendChild(trigger);
  trigger.focus();
  try { run(trigger); } finally { document.body.removeChild(trigger); }
}

describe('DemoModal', () => {
  it('broken: focus does not return to the trigger on close', () => {
    withTrigger((trigger) => {
      render(<DemoModal variant="broken" triggerEl={trigger} defaultOpen />);
      fireEvent.click(screen.getByRole('button', { name: /close/i }));
      expect(document.activeElement).not.toBe(trigger);
    });
  });

  it('fixed: focus returns to the trigger on close', () => {
    withTrigger((trigger) => {
      render(<DemoModal variant="fixed" triggerEl={trigger} defaultOpen />);
      fireEvent.click(screen.getByRole('button', { name: /close/i }));
      expect(document.activeElement).toBe(trigger);
    });
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**, then implement:

`src/lib/variant.ts`:
```ts
export type Variant = 'broken' | 'fixed';
export function readVariant(): Variant {
  return new URLSearchParams(window.location.search).get('variant') === 'fixed' ? 'fixed' : 'broken';
}
```

`src/components/DemoModal.tsx` (PF6 Modal; verify the import surface against the
installed @patternfly/react-core major during the build):
```tsx
import React from 'react';
import { Modal, ModalHeader, ModalBody, ModalFooter, Button } from '@patternfly/react-core';

export interface DemoModalProps {
  variant: 'broken' | 'fixed';
  triggerEl: HTMLElement | null;
  defaultOpen?: boolean;
}

export function DemoModal({ variant, triggerEl, defaultOpen = false }: DemoModalProps) {
  const [isOpen, setIsOpen] = React.useState(defaultOpen);

  const handleClose = () => {
    setIsOpen(false);
    if (variant === 'fixed' && triggerEl) {
      triggerEl.focus(); // FIXED: return focus to the element that opened the modal
    }
    // BROKEN: intentionally omits the focus return. This is the planted hero defect.
  };

  return (
    <Modal isOpen={isOpen} onClose={handleClose} aria-labelledby="demo-modal-title">
      <ModalHeader title="Cluster details" labelId="demo-modal-title" />
      <ModalBody><p>Production cluster: 3 nodes, healthy.</p></ModalBody>
      <ModalFooter>
        <Button variant="primary" onClick={handleClose}>Confirm</Button>
        <Button variant="link" onClick={handleClose}>Close</Button>
      </ModalFooter>
    </Modal>
  );
}
```

`src/pages/Clusters.tsx`: a "View cluster details" trigger button carrying
`aria-haspopup="dialog"` (the Phase 2 probe keys off it), `readVariant()` at page
level, and the modal mounted on open with `triggerEl` from a ref.

- [ ] **Step 3: Run and confirm PASS. Commit** `feat(usabl-app): DemoModal with broken/fixed focus-return variants`

---

## Task 3: False-positive oracle: the fixed variant yields ZERO findings (integration)

**Files:**
- `usabl/test/integration/fixture-clean.test.ts`

> Integration: requires the usabl-app dev server (`npm run dev`, port 5173) and a real
> browser. Guard with `USABL_INTEGRATION=1`.

"Clean means zero findings" is the trust property. If any provider fires on the FIXED
variant (axe defaults, kebab states, walk), that is a harness bug or a fixture bug and
it gets fixed HERE, before the demo flip is attempted. Settings stays identical in
both variants as the control.

- [ ] **Step 1: Write the test**

```ts
// usabl/test/integration/fixture-clean.test.ts
import { describe, it, expect } from 'vitest';
import { run } from '../../src/run.js';
import { buildRealDeps } from '../../src/deps/build.js';   // Task 4a
import type { UsablConfig } from '../../src/contracts/index.js';

const FIXTURE = 'http://127.0.0.1:5173';
const config: UsablConfig = {
  appBaseUrl: FIXTURE,
  uiFileGlobs: ['src/**'],
  discovery: { routerFile: 'src/App.tsx', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: `${FIXTURE}/clusters?variant=fixed`, files: ['src/pages/Clusters.tsx'] }],
  guardedPaths: ['usabl.config.json'],
};

describe.runIf(process.env['USABL_INTEGRATION'] === '1')('fixture false-positive oracle', () => {
  it('the FIXED variant verifies with zero findings through the full provider stack', async () => {
    const deps = await buildRealDeps({ cwd: '../usabl-app' });
    try {
      const result = await run(deps, config, { changedFiles: ['src/pages/Clusters.tsx'] });
      expect(result.findings.filter((f) => f.status !== 'fixed')).toHaveLength(0);
      expect(result.coverage.gaps).toHaveLength(0);
      expect(result.verdict).toBe('verified');
      expect(result.receipt).not.toBeNull();
    } finally {
      await deps.browser.close();
    }
  }, 60_000);
});
```

- [ ] **Step 2: Run against the dev server and drive to green.** Every failure here is
  a real bug in a rule, the driver, or the fixture. Fix the CODE (or the fixture
  markup when PF6 genuinely renders an issue), never the assertion. Document each fix
  in the commit message.

- [ ] **Step 3: Commit** `test(integration): fixed variant is a zero-finding oracle through the full stack`

---

## Task 4: Hero-bug flip: regression → fix → verified, receipt re-verifies

**Files:**
- `usabl/src/deps/real-git.ts`, `usabl/src/deps/real-fs.ts`, `usabl/src/deps/build.ts` (Task 4a)
- `usabl/test/integration/hero-bug-flip.test.ts`

### Task 4a: Real GitReader/FsGlob and the buildRealDeps assembly

The real BrowserDriver exists (Phase 2). This sub-task completes real `Deps`:

- `real-git.ts` implements the frozen `GitReader` with plumbing, NUL-delimited where
  output is parsed (§17): `writeTree()` = `git add -A` into a TEMPORARY index file
  (`GIT_INDEX_FILE`) then `git write-tree` (never touches the user's index);
  `show(ref, path)` = `git show ref:path` (null on nonzero exit); `statusZ()` =
  `git status --porcelain -z`; `lsFiles(ref, prefix)` =
  `git ls-tree -r --name-only ref -- prefix`; `headRef()` = `git rev-parse HEAD`.
- `real-fs.ts` implements `FsGlob` over `node:fs/promises` + a small glob.
- `build.ts` exports `buildRealDeps({ cwd, storageStatePath?, staticOnly? })`:
  assembles clock (`() => new Date().toISOString()` lives HERE, the single real clock),
  the Phase 2 driver, git, fs, and the CheckRunner with the full provider list
  (axe, rulepack incl. probes, walk, voicing when contracts exist, intake when
  requirements exist) and `allowedCapabilities` (`[]` when staticOnly).
  `runnerVersion` = package version + sha256 prefix over the built engine files;
  `scannerVersions` read from installed packages, and an unreadable version becomes a
  `not_covered` gap per §9 (never a guess).
- Wire `src/cli.ts`'s `buildDeps` stub to this module (removes the Phase 1 throw).

- [ ] **Step 1 (4a): implement and unit-test the pure parts** (glob matching, version
  string assembly); plumbing itself is exercised by the integration tests below.

### Task 4b: The flip itself

Preconditions: usabl-app repo committed clean (config + EMPTY floor + empty waivers
committed), dev server running.

- [ ] **Step 2: Write the integration test**

```ts
// usabl/test/integration/hero-bug-flip.test.ts
import { describe, it, expect } from 'vitest';
import { run } from '../../src/run.js';
import { verifyReceipt } from '../../src/evidence/receipt.js';
import { buildRealDeps } from '../../src/deps/build.js';
import type { UsablConfig } from '../../src/contracts/index.js';

const FIXTURE = 'http://127.0.0.1:5173';
const configFor = (variant: 'broken' | 'fixed'): UsablConfig => ({
  appBaseUrl: FIXTURE,
  uiFileGlobs: ['src/**'],
  discovery: { routerFile: 'src/App.tsx', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: `${FIXTURE}/clusters?variant=${variant}`, files: ['src/pages/Clusters.tsx'] }],
  guardedPaths: ['usabl.config.json'],
});
const CHANGED = { changedFiles: ['src/pages/Clusters.tsx'] };

describe.runIf(process.env['USABL_INTEGRATION'] === '1')('hero-bug flip: pf-modal-focus-return', () => {
  it('broken → regression, fixed → verified with a re-verifiable receipt', async () => {
    const deps = await buildRealDeps({ cwd: '../usabl-app' });
    try {
      // 1. Broken state: the interaction probe catches the missing focus return.
      //    The committed floor is EMPTY, so the finding is new: regression.
      const broken = await run(deps, configFor('broken'), CHANGED);
      expect(broken.verdict).toBe('regression');
      expect(broken.exitCode).toBe(1);
      expect(broken.receipt).toBeNull();
      const heroFindings = broken.findings.filter((f) => f.rule === 'pf-modal-focus-return');
      expect(heroFindings.length).toBeGreaterThan(0);
      expect(heroFindings.every((f) => f.status === 'new' && f.confidence === 'fail')).toBe(true);

      // 2. Fixed state: the same probe passes; nothing else fires (Task 3 proved it).
      const fixed = await run(deps, configFor('fixed'), CHANGED);
      expect(fixed.verdict).toBe('verified');
      expect(fixed.exitCode).toBe(0);
      expect(fixed.receipt).not.toBeNull();

      // 3. The receipt re-verifies against the CURRENT tree with the same hash function.
      const tree = await deps.git.writeTree();
      const check = await verifyReceipt(deps, configFor('fixed'), fixed.receipt!, tree);
      expect(check.valid).toBe(true);
      expect(check.failedFields).toEqual([]);

      // 4. Oracle preservation: the scan ran with the overlay plugin active in dev,
      //    and the harness never saw an overlay node (mount guard works).
      const overlayFindings = fixed.findings.filter((f) => f.elementPath.includes('__usabl'));
      expect(overlayFindings).toHaveLength(0);
    } finally {
      await deps.browser.close();
    }
  }, 120_000);
});
```

- [ ] **Step 3: Run and drive to green.** The regression assertion failing means the
  probe or the driver is broken (Phase 2 territory); the verified assertion failing
  means a false positive survived Task 3. Trace to the responsible phase's code and
  fix it there.

- [ ] **Step 4: Commit** `test(integration): hero-bug flip through the real engine with receipt re-verification`

---

## Task 5: Verdict oracle check (no duplication)

The five-outcome golden oracle (idle, verified, regression, not_covered,
approval_required) lives in Phase 1 Task 15 over fakes and stays there: one oracle,
one owner. This task is a check, not new code:

- [ ] Run `npx vitest run test/golden/oracle.test.ts` in `usabl` and confirm all five
  scenarios are green on the integrated codebase.
- [ ] If integration work exposed an outcome the oracle does not pin (it should not),
  add the scenario to PHASE 1's oracle file with `UPDATE_GOLDEN=1` and eyeball the
  new golden JSON before committing it.

---

## Task 6: Fleet Insights measurement harness (measurement-only, never gates)

**Files:**
- `usabl/src/measure/fleet-insights.ts`
- `usabl/scripts/measure-fleet-insights.ts`

Fleet Insights is a client-rendered React + PatternFly SPA behind Red Hat SSO. The
measurement pass answers ONE question honestly: how much of a real app can the engine
exercise, and where are the gaps. It mints no verdict and writes no receipt.

- [ ] **Step 1: Implement `runMeasurementOnly`** over the REAL contracts (no invented
  config fields; it takes a surface list directly):

```ts
// usabl/src/measure/fleet-insights.ts
import type { CheckRunner, CoverageGap, ScreenScan } from '../contracts/index.js';

export interface MeasurementInput { id: string; url: string; }
export interface MeasurementReport {
  screensAttempted: number;
  screensWithFindings: number;
  totalDrafts: number;
  gaps: CoverageGap[];
  screens: Array<{ id: string; drafts: number; stops: number; gaps: number }>;
  note: 'measurement-only: no verdict minted, no receipt written, nothing gated';
}

/** Scan each surface via the normal CheckRunner (scan never throws; failures are gaps). */
export async function runMeasurementOnly(
  checkRunner: CheckRunner,
  surfaces: MeasurementInput[],
): Promise<MeasurementReport> {
  const scans: ScreenScan[] = [];
  for (const s of surfaces) scans.push(await checkRunner.scan(s));
  return {
    screensAttempted: surfaces.length,
    screensWithFindings: scans.filter((x) => x.drafts.length > 0).length,
    totalDrafts: scans.reduce((n, x) => n + x.drafts.length, 0),
    gaps: scans.flatMap((x) => x.gaps),
    screens: scans.map((x) => ({ id: x.screenId, drafts: x.drafts.length, stops: x.stops.length, gaps: x.gaps.length })),
    note: 'measurement-only: no verdict minted, no receipt written, nothing gated',
  };
}
```

- [ ] **Step 2: The runnable script** (`scripts/measure-fleet-insights.ts`): reads
  `FLEET_INSIGHTS_STORAGE_STATE` (exits 1 with instructions when unset), builds real
  deps with `buildRealDeps({ cwd: '.', storageStatePath })`, defines the surface list
  (`/`, `/clusters`, `/workloads`, `/settings` on the Fleet Insights host), calls
  `runMeasurementOnly`, stamps `runAt` (a script may read the clock; the ENGINE may
  not), and prints the JSON report to stdout. `deps.browser.close()` in a finally.

- [ ] **Step 3: Dry check** `npx tsc --noEmit`, then a live run by the operator with a
  fresh session. The report's gap list plus the verified-verdict-rate math feed the
  week-2 `not_covered` recalibration decision (ground-truth §25).

- [ ] **Step 4: Commit** `feat(measure): Fleet Insights measurement-only harness over the real CheckRunner`

---

## Task 7: storageState export and secret-scrub procedure

**Files:**
- `usabl/.gitignore` (add storageState patterns)
- `usabl/scripts/scrub-storage-state.ts`

Unchanged procedure, restated because it is security-load-bearing:

- [ ] `.gitignore` additions:

```gitignore
# Fleet Insights session: never commit, scrubbed or not
storageState*.json
**/storageState*.json
fleet-insights-session/
```

- [ ] `scripts/scrub-storage-state.ts`: reads a Playwright storageState JSON, replaces
  every cookie value matching `/^[A-Za-z0-9_-]{20,}$/` with `[SCRUBBED:<name>]`,
  writes the review copy, prints counts. Even the scrubbed file stays out of git.

- [ ] One-time export procedure (comment block at the top of the script):
  `npx playwright open --save-storage=/tmp/fleet-insights-session.json <host>`,
  complete SSO + MFA, visit the four surfaces, close; scrub for review; export
  `FLEET_INSIGHTS_STORAGE_STATE=/tmp/fleet-insights-session.json`; sessions expire
  with SSO cookie rotation (8-24 h), re-export on 401/302.

- [ ] Verify the ignore: `echo '{}' > storageState-test.json && git check-ignore -v storageState-test.json && rm storageState-test.json`

- [ ] Commit `chore: gitignore storageState and add scrub utility`

---

## Self-Review

**Contract fidelity:** every import in this phase resolves against `usabl`'s frozen
`src/contracts/index.ts`. No `src/types.js`, no `gate({ baseFindings })`, no
`Draft.ruleId`, no `UsablConfig.screens/measurementOnly`. The ratchet in the flip test
is the committed evidence floor, not a passed-in baseline.

**Hero bug detectability:** the trigger carries `aria-haspopup="dialog"`, the Phase 2
probe clicks it, Escapes, and asserts `activeElementIs(trigger)`. Broken variant fails
that predicate; fixed variant passes. The chain from planted defect to `regression` to
`verified` runs through the real driver, the real gate, and the real floor.

**False-positive oracle:** Task 3 proves clean-equals-zero-findings through the FULL
stack before the flip is attempted, with Settings as the untouched control. Failures
there are code bugs by definition and get fixed at the responsible phase.

**Real deps completed:** Task 4a is the single home for real git plumbing
(NUL-delimited parsing, temporary-index write-tree) and the `buildRealDeps` assembly;
the CLI's Phase 1 stub is retired here.

**Measurement honesty:** `runMeasurementOnly` never touches the gate, mints nothing,
and reports gaps verbatim from `ScreenScan.gaps`. The script owns the wall clock;
the engine never does.

**Secrets:** storageState flows only through an env var into the driver; gitignore
plus scrub utility plus expiry note; nothing session-shaped is ever committed.

**Repo hygiene:** `usabl-dev/usabl-app` remote creation is an operator step; the agent
scaffolds locally and stops.
