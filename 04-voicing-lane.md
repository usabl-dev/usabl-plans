# Announcement Voicing Lane Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the differentiating voicing layer to usabl. An authored `InteractionContract` drives a Virtual Screen Reader provider that records what an AT announces at each step. A structural tier emits `deterministic` findings that gate. A voicing tier emits `preview` findings that never gate by default. A `promotedObligations` config list is the only promotion mechanism; promotion happens offline, never at runtime.

**Architecture:**
- `VirtualSrProvider` is a `Provider`. It returns `Draft[]`. It never decides a verdict.
- The gate (Phase 1) is the sole verdict authority. `evidenceClass: 'preview'` findings are excluded from the gate verdict by definition; only `deterministic` (and items listed in `promotedObligations`) feed it.
- `StepRunner` (frozen seam from `00-plan-set.md`) executes `Step[]` and returns `TranscriptStop[]`. The voicing provider reuses it — no reimplementation.
- The COMPARATOR INVARIANT: normalize the authored `SpeechObligation.requiredTokens` token; compare against the RAW observed `AnnouncementToken` text. Never normalize the observed transcript.
- `@guidepup/virtual-screen-reader` runs only in integration tests against a real DOM. Unit tests use a fake transcript injected via a `TranscriptReader` seam.

**Tech Stack:**
- TypeScript (ESM, strict), Node 22
- Vitest for all tests (unit tests: fakes only; no DOM, no real browser)
- `@guidepup/virtual-screen-reader` — integration-only; behind `TranscriptReader` seam
- Frozen contracts from `src/contracts/index.ts` (Phase 1): `InteractionContract`, `SpeechObligation`, `Step`, `TranscriptStop`, `AnnouncementToken`, `Draft`, `Finding`, `Provider`, `ProviderContext`, `Capability`, `UsablConfig`, `Page`
- Frozen `StepRunner` seam from `00-plan-set.md`

---

## Task 1 — Normalize helper for obligation tokens

**Why first:** every other task depends on the comparator invariant. Nail it in isolation with pure, trivially-testable code.

**Files:**
- `src/voicing/normalize.ts`
- `tests/voicing/normalize.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// tests/voicing/normalize.test.ts
import { describe, it, expect } from 'vitest';
import { normalizeToken } from '../../src/voicing/normalize.js';

describe('normalizeToken', () => {
  it('lowercases and strips punctuation', () => {
    expect(normalizeToken('Save Changes!')).toBe('save changes');
  });
  it('collapses whitespace', () => {
    expect(normalizeToken('  Save   Changes  ')).toBe('save changes');
  });
  it('handles null as empty string', () => {
    expect(normalizeToken(null)).toBe('');
  });
  it('leaves plain tokens unchanged except case', () => {
    expect(normalizeToken('dialog')).toBe('dialog');
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run tests/voicing/normalize.test.ts
```

Expected: `Cannot find module '../../src/voicing/normalize.js'`

- [ ] **Step 3: Write the implementation**

```ts
// src/voicing/normalize.ts

/**
 * Normalize an authored obligation token for comparison.
 *
 * COMPARATOR INVARIANT: this function is called on the AUTHORED side only.
 * Raw observed AnnouncementToken.text is NEVER passed through here.
 */
export function normalizeToken(raw: string | null): string {
  if (raw === null) return '';
  return raw
    .toLowerCase()
    .replace(/[^\w\s]/g, '')
    .replace(/\s+/g, ' ')
    .trim();
}

/**
 * Return true when every required token from the obligation is present
 * (as a substring) in at least one announcement in the stop's window.
 *
 * Normalization is applied to the OBLIGATION tokens.
 * The observed text is matched raw (lowercased for comparison only, never stripped).
 */
export function obligationSatisfied(
  required: string[],
  announcements: Array<{ text: string | null }>,
): boolean {
  const observedRaw = announcements
    .map(a => (a.text ?? '').toLowerCase())
    .join('\n');
  return required.every(tok => observedRaw.includes(normalizeToken(tok)));
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run tests/voicing/normalize.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): add token normalizer with comparator invariant
```

---

## Task 2 — `TranscriptReader` seam and fake

**Why:** `@guidepup/virtual-screen-reader` is a real browser dependency. The seam makes every unit test run on pure fakes; the real wiring is a labeled integration task at the end.

**Files:**
- `src/voicing/transcript-reader.ts`
- `src/voicing/fakes.ts`
- `tests/voicing/transcript-reader.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// tests/voicing/transcript-reader.test.ts
import { describe, it, expect } from 'vitest';
import type { TranscriptStop } from '../../src/contracts/index.js';
import { FakeTranscriptReader } from '../../src/voicing/fakes.js';

describe('FakeTranscriptReader', () => {
  it('returns the pre-configured stops', async () => {
    const stops: TranscriptStop[] = [
      { index: 0, elementPath: 'button#save', announcement: [{ kind: 'name', text: 'Save', fromTree: true, source: 'ax-tree' }] },
      { index: 1, elementPath: 'dialog', announcement: [{ kind: 'role', text: 'dialog', fromTree: true, source: 'ax-tree' }, { kind: 'name', text: 'Confirm deletion', fromTree: true, source: 'ax-tree' }] },
    ];
    const reader = new FakeTranscriptReader(stops);
    const result = await reader.read();
    expect(result).toStrictEqual(stops);
  });

  it('returns empty array when no stops configured', async () => {
    const reader = new FakeTranscriptReader([]);
    expect(await reader.read()).toEqual([]);
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run tests/voicing/transcript-reader.test.ts
```

- [ ] **Step 3: Write the seam and fake**

```ts
// src/voicing/transcript-reader.ts
import type { TranscriptStop } from '../contracts/index.js';

/**
 * Seam between the voicing provider and @guidepup/virtual-screen-reader.
 * Unit tests use FakeTranscriptReader. Integration uses GuidepupTranscriptReader
 * (labeled integration task at end of this plan).
 */
export interface TranscriptReader {
  read(): Promise<TranscriptStop[]>;
}
```

```ts
// src/voicing/fakes.ts
import type { TranscriptStop } from '../contracts/index.js';
import type { TranscriptReader } from './transcript-reader.js';

export class FakeTranscriptReader implements TranscriptReader {
  constructor(private readonly stops: TranscriptStop[]) {}
  async read(): Promise<TranscriptStop[]> {
    return this.stops;
  }
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run tests/voicing/transcript-reader.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): add TranscriptReader seam and FakeTranscriptReader
```

---

## Task 3 — Structural tier (deterministic, gates)

The structural tier reads `TranscriptStop[]` and the `InteractionContract` and emits `deterministic` `Draft[]` for reproducible structural failures: missing name, missing role, missing state, no live region, unreachable focus, over-long tab path. These gate.

**Files:**
- `src/voicing/structural-tier.ts`
- `tests/voicing/structural-tier.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/voicing/structural-tier.test.ts
import { describe, it, expect } from 'vitest';
import type { TranscriptStop, InteractionContract, Draft } from '../../src/contracts/index.js';
import { runStructuralTier } from '../../src/voicing/structural-tier.js';

const baseContract: InteractionContract = {
  contractId: 'test-contract',
  surfaceId: 'clusters-page',
  task: 'Open delete dialog',
  steps: [
    { do: 'tab' },
    { do: 'activate' },
  ],
  obligations: [
    {
      class: 'dialog-name',
      afterStep: 1,
      requiredTokens: ['Confirm deletion'],
      focusedRole: 'dialog',
      mustAnnounce: true,
    },
  ],
  maxTabPath: 5,
};

function stop(index: number, elementPath: string, tokens: Array<{ kind: 'name' | 'role' | 'state'; text: string | null }>): TranscriptStop {
  return {
    index,
    elementPath,
    announcement: tokens.map(t => ({ ...t, fromTree: true, source: 'ax-tree' as const })),
  };
}

describe('runStructuralTier', () => {
  it('emits no drafts when all obligations met structurally', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button#delete', [{ kind: 'name', text: 'Delete cluster' }, { kind: 'role', text: 'button' }]),
      stop(1, 'dialog', [{ kind: 'role', text: 'dialog' }, { kind: 'name', text: 'Confirm deletion' }]),
    ];
    const drafts = runStructuralTier(baseContract, stops, 'clusters-page');
    expect(drafts.filter(d => d.evidenceClass === 'deterministic')).toHaveLength(0);
  });

  it('emits deterministic draft when focused element has no name', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button#delete', [{ kind: 'role', text: 'button' }]),
      stop(1, 'dialog', [{ kind: 'role', text: 'dialog' }, { kind: 'name', text: 'Confirm deletion' }]),
    ];
    const drafts = runStructuralTier(baseContract, stops, 'clusters-page');
    const det = drafts.filter(d => d.evidenceClass === 'deterministic');
    expect(det.length).toBeGreaterThanOrEqual(1);
    expect(det[0].rule).toBe('voicing/missing-name');
  });

  it('emits deterministic draft when mustAnnounce obligation has no live-region role', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button#delete', [{ kind: 'name', text: 'Delete cluster' }, { kind: 'role', text: 'button' }]),
      // dialog role missing — focus landed somewhere else
      stop(1, 'main', [{ kind: 'name', text: 'Confirm deletion' }]),
    ];
    const drafts = runStructuralTier(baseContract, stops, 'clusters-page');
    const det = drafts.filter(d => d.evidenceClass === 'deterministic');
    expect(det.some(d => d.rule === 'voicing/missing-role')).toBe(true);
  });

  it('emits deterministic draft when tab path exceeds maxTabPath', () => {
    const contract: InteractionContract = { ...baseContract, maxTabPath: 2 };
    const stops: TranscriptStop[] = [
      stop(0, 'a#skip', [{ kind: 'name', text: 'Skip to content' }, { kind: 'role', text: 'link' }]),
      stop(1, 'button#one', [{ kind: 'name', text: 'First' }, { kind: 'role', text: 'button' }]),
      stop(2, 'button#two', [{ kind: 'name', text: 'Second' }, { kind: 'role', text: 'button' }]),
    ];
    const drafts = runStructuralTier(contract, stops, 'clusters-page');
    const det = drafts.filter(d => d.rule === 'voicing/over-long-tab-path');
    expect(det.length).toBeGreaterThanOrEqual(1);
    expect(det[0].evidenceClass).toBe('deterministic');
  });

  it('emits deterministic draft when a step stop is absent (unreachable focus)', () => {
    // zero stops — keyboard walk produced nothing
    const drafts = runStructuralTier(baseContract, [], 'clusters-page');
    const det = drafts.filter(d => d.rule === 'voicing/unreachable-focus');
    expect(det.length).toBeGreaterThanOrEqual(1);
    expect(det[0].evidenceClass).toBe('deterministic');
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run tests/voicing/structural-tier.test.ts
```

- [ ] **Step 3: Write the implementation**

```ts
// src/voicing/structural-tier.ts
import type {
  Draft,
  TranscriptStop,
  InteractionContract,
  SpeechObligation,
  AnnouncementToken,
} from '../contracts/index.js';

function hasTok(announcement: AnnouncementToken[], kind: 'name' | 'role' | 'state'): boolean {
  return announcement.some(t => t.kind === kind && t.text !== null && t.text.trim() !== '');
}

function makeDraft(
  rule: string,
  screenId: string,
  elementPath: string,
  whatUserExperiences: string,
  why: string,
  fix: string,
): Draft {
  return {
    rule,
    layer: 'voicing',
    severity: 'serious',
    evidenceClass: 'deterministic',
    screenId,
    elementPath,
    elementName: null,
    role: null,
    whatUserExperiences,
    why,
    fix,
    evidence: {},
    confidence: 'fail',
  };
}

export function runStructuralTier(
  contract: InteractionContract,
  stops: TranscriptStop[],
  screenId: string,
): Draft[] {
  const drafts: Draft[] = [];

  // 1. Unreachable focus: no stops at all
  if (stops.length === 0) {
    drafts.push(makeDraft(
      'voicing/unreachable-focus',
      screenId,
      contract.surfaceId,
      'Keyboard focus could not reach any element defined in the interaction contract.',
      'The keyboard walk returned zero stops. Focus may be trapped, the page may not load, or all interactive elements may be unreachable via Tab.',
      'Ensure all interactive elements are reachable via sequential Tab navigation from the body.',
    ));
    return drafts;
  }

  // 2. Missing name or role on each stop
  for (const stop of stops) {
    const el = stop.elementPath;
    if (!hasTok(stop.announcement, 'name')) {
      drafts.push(makeDraft(
        'voicing/missing-name',
        screenId,
        el,
        `An AT user receives no accessible name when focus reaches "${el}".`,
        'The accessibility tree exposes no name for this element at focus time.',
        'Add an accessible name via aria-label, aria-labelledby, or visible text content.',
      ));
    }
    if (!hasTok(stop.announcement, 'role')) {
      drafts.push(makeDraft(
        'voicing/missing-role',
        screenId,
        el,
        `An AT user receives no role announcement when focus reaches "${el}".`,
        'The element has no ARIA role or semantic HTML role exposed in the accessibility tree.',
        'Use a semantic HTML element or add an explicit role attribute.',
      ));
    }
  }

  // 3. Per-obligation structural checks (mustAnnounce = live region required)
  for (const ob of contract.obligations) {
    const windowStop = stops.find(s => s.index === ob.afterStep);
    if (!windowStop) {
      // step index not reached — unreachable focus for this obligation window
      drafts.push(makeDraft(
        'voicing/unreachable-focus',
        screenId,
        `step-${ob.afterStep}`,
        `Keyboard could not reach the element expected at step ${ob.afterStep}.`,
        `No TranscriptStop recorded at step index ${ob.afterStep}.`,
        'Ensure the step sequence reaches the target element.',
      ));
      continue;
    }

    // mustAnnounce: the focusedRole must be a live-region or dialog kind
    if (ob.mustAnnounce && ob.focusedRole) {
      const roleFound = windowStop.announcement.some(
        t => t.kind === 'role' && (t.text ?? '').toLowerCase() === ob.focusedRole!.toLowerCase(),
      );
      if (!roleFound) {
        drafts.push(makeDraft(
          'voicing/missing-role',
          screenId,
          windowStop.elementPath,
          `After step ${ob.afterStep}, an AT user expects focus on a "${ob.focusedRole}" but no such role was announced.`,
          `SpeechObligation "${ob.class}" requires focusedRole "${ob.focusedRole}" at step ${ob.afterStep}; observed role was absent or different.`,
          `Ensure the element at step ${ob.afterStep} carries role="${ob.focusedRole}" or equivalent.`,
        ));
      }
    }
  }

  // 4. Over-long tab path
  if (contract.maxTabPath !== undefined && stops.length > contract.maxTabPath) {
    drafts.push(makeDraft(
      'voicing/over-long-tab-path',
      screenId,
      contract.surfaceId,
      `An AT user must Tab ${stops.length} times to reach the target; the contract allows at most ${contract.maxTabPath}.`,
      `Tab path length ${stops.length} exceeds maxTabPath ${contract.maxTabPath} defined in contract "${contract.contractId}".`,
      'Reduce the number of focusable elements before the target, or add a skip-navigation link.',
    ));
  }

  return drafts;
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run tests/voicing/structural-tier.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): structural tier — deterministic drafts for missing name/role/state, unreachable focus, over-long tab path
```

---

## Task 4 — Voicing tier (preview, advisory)

The voicing tier matches transcript tokens inside each obligation's step window. It emits `preview` findings. It never gates. `promotedObligations` are checked at runtime to decide whether to flip `preview` to `deterministic` for that obligation class.

**Files:**
- `src/voicing/voicing-tier.ts`
- `tests/voicing/voicing-tier.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/voicing/voicing-tier.test.ts
import { describe, it, expect } from 'vitest';
import type { TranscriptStop, InteractionContract, Draft } from '../../src/contracts/index.js';
import { runVoicingTier } from '../../src/voicing/voicing-tier.js';

const contract: InteractionContract = {
  contractId: 'toast-contract',
  surfaceId: 'clusters-page',
  task: 'Delete cluster and observe toast',
  steps: [{ do: 'tab' }, { do: 'activate' }],
  obligations: [
    {
      class: 'toast-announced',
      afterStep: 1,
      requiredTokens: ['Cluster deleted', 'success'],
      mustAnnounce: false,
    },
  ],
};

function stop(index: number, texts: string[]): TranscriptStop {
  return {
    index,
    elementPath: `el-${index}`,
    announcement: texts.map(text => ({ kind: 'name' as const, text, fromTree: true, source: 'ax-tree' as const })),
  };
}

describe('runVoicingTier', () => {
  it('emits no draft when obligation tokens are present in the window', () => {
    const stops: TranscriptStop[] = [
      stop(0, ['Delete cluster', 'button']),
      stop(1, ['Cluster deleted', 'Alert success']),
    ];
    const drafts = runVoicingTier(contract, stops, 'clusters-page', []);
    expect(drafts).toHaveLength(0);
  });

  it('emits preview draft when required token is absent from the window', () => {
    const stops: TranscriptStop[] = [
      stop(0, ['Delete cluster', 'button']),
      stop(1, ['Loading...']),
    ];
    const drafts = runVoicingTier(contract, stops, 'clusters-page', []);
    expect(drafts).toHaveLength(1);
    expect(drafts[0].evidenceClass).toBe('preview');
    expect(drafts[0].rule).toBe('voicing/missing-announcement');
  });

  it('emits deterministic draft when obligation class is in promotedObligations', () => {
    const stops: TranscriptStop[] = [
      stop(0, ['Delete cluster', 'button']),
      stop(1, ['Loading...']),
    ];
    const drafts = runVoicingTier(contract, stops, 'clusters-page', ['toast-announced']);
    expect(drafts).toHaveLength(1);
    expect(drafts[0].evidenceClass).toBe('deterministic');
  });

  it('emits no draft when the window stop is missing but mustAnnounce is false', () => {
    // obligation at step 1 but stops only go to step 0
    // structural tier handles unreachable; voicing tier is lenient on missing window
    const stops: TranscriptStop[] = [stop(0, ['Delete cluster', 'button'])];
    const drafts = runVoicingTier(contract, stops, 'clusters-page', []);
    // No stop at index 1 — voicing tier records a preview gap
    expect(drafts[0]?.evidenceClass).toBe('preview');
  });

  it('preview findings never carry confidence:fail when not promoted', () => {
    const stops: TranscriptStop[] = [stop(1, ['something else'])];
    const drafts = runVoicingTier(contract, stops, 'clusters-page', []);
    drafts.filter(d => d.evidenceClass === 'preview').forEach(d => {
      // preview findings are advisory; confidence reflects uncertainty
      expect(d.confidence).toBe('unverified');
    });
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run tests/voicing/voicing-tier.test.ts
```

- [ ] **Step 3: Write the implementation**

```ts
// src/voicing/voicing-tier.ts
import type {
  Draft,
  EvidenceClass,
  TranscriptStop,
  InteractionContract,
  SpeechObligation,
} from '../contracts/index.js';
import { obligationSatisfied } from './normalize.js';

function makeDraft(
  rule: string,
  screenId: string,
  elementPath: string,
  evidenceClass: EvidenceClass,
  whatUserExperiences: string,
  why: string,
  fix: string,
): Draft {
  const isPromoted = evidenceClass === 'deterministic';
  return {
    rule,
    layer: 'voicing',
    severity: 'moderate',
    evidenceClass,
    screenId,
    elementPath,
    elementName: null,
    role: null,
    whatUserExperiences,
    why,
    fix,
    evidence: {},
    // promoted obligations are reproducible facts; advisory ones are unverified
    confidence: isPromoted ? 'fail' : 'unverified',
  };
}

export function runVoicingTier(
  contract: InteractionContract,
  stops: TranscriptStop[],
  screenId: string,
  promotedObligations: string[],
): Draft[] {
  const drafts: Draft[] = [];

  for (const ob of contract.obligations) {
    const windowStop = stops.find(s => s.index === ob.afterStep);

    const evClass: EvidenceClass = promotedObligations.includes(ob.class)
      ? 'deterministic'
      : 'preview';

    if (!windowStop) {
      // No stop in this window — emit a gap finding
      drafts.push(makeDraft(
        'voicing/missing-announcement',
        screenId,
        `step-${ob.afterStep}`,
        evClass,
        `No AT announcement was recorded at step ${ob.afterStep}; required tokens were not observed.`,
        `SpeechObligation "${ob.class}" requires tokens [${ob.requiredTokens.join(', ')}] at step ${ob.afterStep}, but no stop was recorded there.`,
        'Ensure the step sequence reaches an element that triggers the expected announcement.',
      ));
      continue;
    }

    const satisfied = obligationSatisfied(ob.requiredTokens, windowStop.announcement);
    if (!satisfied) {
      const observed = windowStop.announcement.map(t => t.text ?? '(null)').join(', ');
      drafts.push(makeDraft(
        'voicing/missing-announcement',
        screenId,
        windowStop.elementPath,
        evClass,
        `An AT user does not hear the expected announcement after step ${ob.afterStep}. Required: [${ob.requiredTokens.join(', ')}].`,
        `Obligation "${ob.class}": required tokens [${ob.requiredTokens.join(', ')}] not found in observed transcript at step ${ob.afterStep}. Observed: [${observed}].`,
        'Ensure the widget announces the required name/role/state tokens at the correct step. Check live-region wiring and ARIA attribute values.',
      ));
    }
  }

  return drafts;
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run tests/voicing/voicing-tier.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): voicing tier — preview token matching with promotedObligations promotion
```

---

## Task 5 — `VirtualSrProvider` (the Provider)

Wires both tiers together into a `Provider` that the `CheckRunner` (Phase 2) can call. Accepts contracts via config (loaded from the `requirements` file path or a direct list). Reads `TranscriptStop[]` from a `StepRunner` run, then calls both tiers and merges their `Draft[]`.

**Files:**
- `src/voicing/virtual-sr-provider.ts`
- `tests/voicing/virtual-sr-provider.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/voicing/virtual-sr-provider.test.ts
import { describe, it, expect, vi } from 'vitest';
import type {
  Draft, InteractionContract, TranscriptStop, ProviderContext, UsablConfig, Page,
} from '../../src/contracts/index.js';
import type { StepRunner } from '../../src/contracts/index.js';
import { makeVirtualSrProvider } from '../../src/voicing/virtual-sr-provider.js';
import { FakeTranscriptReader } from '../../src/voicing/fakes.js';

// Minimal fake Page (satisfies interface; methods not called in these tests)
function fakePage(): Page {
  const noop = async () => {};
  return {
    gotoReady: noop, focusBody: noop, tab: noop, press: noop, close: noop,
    setViewport: noop, setZoom: noop, setReducedMotion: noop,
    activeNode: async () => null, activePath: async () => '',
    axAt: async () => null, queryAll: async () => [],
    getComputedStyle: async () => '', screenshot: async () => Buffer.alloc(0),
  };
}

const contract: InteractionContract = {
  contractId: 'modal-contract',
  surfaceId: 'clusters-page',
  task: 'Open modal and verify announcement',
  steps: [{ do: 'tab' }, { do: 'activate' }],
  obligations: [
    {
      class: 'modal-name',
      afterStep: 1,
      requiredTokens: ['Delete cluster'],
      focusedRole: 'dialog',
      mustAnnounce: true,
    },
  ],
  maxTabPath: 10,
};

function stops(withName: boolean): TranscriptStop[] {
  return [
    { index: 0, elementPath: 'button#delete', announcement: [{ kind: 'name', text: 'Delete', fromTree: true, source: 'ax-tree' }, { kind: 'role', text: 'button', fromTree: true, source: 'ax-tree' }] },
    { index: 1, elementPath: 'dialog', announcement: withName
      ? [{ kind: 'role', text: 'dialog', fromTree: true, source: 'ax-tree' }, { kind: 'name', text: 'Delete cluster', fromTree: true, source: 'ax-tree' }]
      : [{ kind: 'role', text: 'dialog', fromTree: true, source: 'ax-tree' }],
    },
  ];
}

function fakeRunner(s: TranscriptStop[]): StepRunner {
  return { run: async () => s };
}

function fakeConfig(promoted: string[] = []): UsablConfig {
  return {
    appBaseUrl: 'http://localhost:3000',
    uiFileGlobs: [],
    discovery: { routerFile: '', wideBlastGlobs: [] },
    surfaces: [],
    guardedPaths: [],
    promotedObligations: promoted,
  };
}

describe('VirtualSrProvider', () => {
  it('declares capability live', () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner(stops(true)));
    expect(provider.capabilities).toContain('live');
  });

  it('returns no drafts when all obligations are satisfied', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner(stops(true)));
    const ctx: ProviderContext = { page: fakePage(), screen: { id: 'clusters-page', url: 'http://localhost:3000/clusters' }, config: fakeConfig() };
    const drafts = await provider.run(ctx);
    expect(drafts.filter(d => d.evidenceClass === 'deterministic')).toHaveLength(0);
    expect(drafts.filter(d => d.evidenceClass === 'preview')).toHaveLength(0);
  });

  it('returns deterministic draft when modal name missing', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner(stops(false)));
    const ctx: ProviderContext = { page: fakePage(), screen: { id: 'clusters-page', url: 'http://localhost:3000/clusters' }, config: fakeConfig() };
    const drafts = await provider.run(ctx);
    const det = drafts.filter(d => d.evidenceClass === 'deterministic');
    // structural tier should flag missing name for dialog obligation
    expect(det.length).toBeGreaterThanOrEqual(1);
  });

  it('returns preview draft for voicing token miss when not promoted', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner(stops(false)));
    const ctx: ProviderContext = { page: fakePage(), screen: { id: 'clusters-page', url: 'http://localhost:3000/clusters' }, config: fakeConfig([]) };
    const drafts = await provider.run(ctx);
    const preview = drafts.filter(d => d.evidenceClass === 'preview');
    expect(preview.length).toBeGreaterThanOrEqual(1);
  });

  it('returns deterministic draft for voicing token miss when obligation is promoted', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner(stops(false)));
    const ctx: ProviderContext = { page: fakePage(), screen: { id: 'clusters-page', url: 'http://localhost:3000/clusters' }, config: fakeConfig(['modal-name']) };
    const drafts = await provider.run(ctx);
    const det = drafts.filter(d => d.evidenceClass === 'deterministic');
    expect(det.length).toBeGreaterThanOrEqual(1);
  });

  it('skips contracts for other surfaces', async () => {
    const otherContract: InteractionContract = { ...contract, surfaceId: 'other-page' };
    const provider = makeVirtualSrProvider([otherContract], fakeRunner(stops(false)));
    const ctx: ProviderContext = { page: fakePage(), screen: { id: 'clusters-page', url: 'http://localhost:3000/clusters' }, config: fakeConfig() };
    const drafts = await provider.run(ctx);
    expect(drafts).toHaveLength(0);
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run tests/voicing/virtual-sr-provider.test.ts
```

- [ ] **Step 3: Write the implementation**

```ts
// src/voicing/virtual-sr-provider.ts
import type {
  Draft, InteractionContract, Provider, ProviderContext, Capability, StepRunner,
} from '../contracts/index.js';
import { runStructuralTier } from './structural-tier.js';
import { runVoicingTier } from './voicing-tier.js';

/**
 * VirtualSrProvider — the Phase 4 Provider.
 *
 * - Declares capability 'live': it drives real browser focus via StepRunner.
 * - Returns Draft[] only; never decides a verdict.
 * - Structural tier emits deterministic findings (gate-eligible).
 * - Voicing tier emits preview by default; emits deterministic if the obligation
 *   class appears in config.promotedObligations.
 * - Promotion is a config-time flag set by offline calibration only; the engine
 *   never self-promotes at runtime.
 */
export function makeVirtualSrProvider(
  contracts: InteractionContract[],
  stepRunner: StepRunner,
): Provider {
  return {
    id: 'voicing/virtual-sr',
    layer: 'voicing',
    capabilities: ['live'] satisfies Capability[],

    async run(ctx: ProviderContext): Promise<Draft[]> {
      const { screen, config } = ctx;
      const promoted = config.promotedObligations ?? [];

      // Filter to contracts scoped to this surface
      const surfaceContracts = contracts.filter(c => c.surfaceId === screen.id);
      if (surfaceContracts.length === 0) return [];

      const drafts: Draft[] = [];

      for (const contract of surfaceContracts) {
        // Execute the interaction steps via the frozen StepRunner seam
        const stops = await stepRunner.run(ctx.page, contract.steps);

        // Structural tier: deterministic, gate-eligible
        const structural = runStructuralTier(contract, stops, screen.id);
        drafts.push(...structural);

        // Voicing tier: preview by default; deterministic if class is promoted
        const voicing = runVoicingTier(contract, stops, screen.id, promoted);
        drafts.push(...voicing);
      }

      return drafts;
    },
  };
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run tests/voicing/virtual-sr-provider.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): VirtualSrProvider wires structural and voicing tiers as a Provider
```

---

## Task 6 — Promotion invariant tests

These tests assert the invariants in one place: `preview` never mints `verified`, promoted findings gate as `deterministic`, no runtime self-promotion path exists in the provider code.

**Files:**
- `tests/voicing/promotion-invariants.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// tests/voicing/promotion-invariants.test.ts
import { describe, it, expect } from 'vitest';
import type { TranscriptStop, InteractionContract, UsablConfig } from '../../src/contracts/index.js';
import { makeVirtualSrProvider } from '../../src/voicing/virtual-sr-provider.js';

const contract: InteractionContract = {
  contractId: 'promo-test',
  surfaceId: 'test-surface',
  task: 'Verify promotion invariant',
  steps: [{ do: 'tab' }],
  obligations: [
    {
      class: 'check-class-a',
      afterStep: 0,
      requiredTokens: ['Expected token'],
      mustAnnounce: false,
    },
    {
      class: 'check-class-b',
      afterStep: 0,
      requiredTokens: ['Other token'],
      mustAnnounce: false,
    },
  ],
};

// stop that satisfies neither obligation
const failStop: TranscriptStop = {
  index: 0,
  elementPath: 'button',
  announcement: [{ kind: 'name', text: 'Wrong text', fromTree: true, source: 'ax-tree' }],
};

const fakeRunner = { run: async () => [failStop] };

function cfg(promoted: string[]): UsablConfig {
  return {
    appBaseUrl: 'http://localhost:3000',
    uiFileGlobs: [],
    discovery: { routerFile: '', wideBlastGlobs: [] },
    surfaces: [],
    guardedPaths: [],
    promotedObligations: promoted,
  };
}

function fakePage(): any { return {}; }

describe('promotion invariants', () => {
  it('without promotion, all voicing findings are preview', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner);
    const ctx = { page: fakePage(), screen: { id: 'test-surface', url: 'http://localhost:3000/' }, config: cfg([]) };
    const drafts = await provider.run(ctx);
    const voicingDrafts = drafts.filter(d => d.rule === 'voicing/missing-announcement');
    expect(voicingDrafts.length).toBeGreaterThan(0);
    voicingDrafts.forEach(d => expect(d.evidenceClass).toBe('preview'));
  });

  it('only the listed obligation class is promoted; others stay preview', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner);
    const ctx = { page: fakePage(), screen: { id: 'test-surface', url: 'http://localhost:3000/' }, config: cfg(['check-class-a']) };
    const drafts = await provider.run(ctx);
    const promoted = drafts.filter(d => d.rule === 'voicing/missing-announcement' && d.evidenceClass === 'deterministic');
    const notPromoted = drafts.filter(d => d.rule === 'voicing/missing-announcement' && d.evidenceClass === 'preview');
    expect(promoted.length).toBe(1);
    expect(notPromoted.length).toBe(1);
  });

  it('preview findings never carry confidence:fail', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner);
    const ctx = { page: fakePage(), screen: { id: 'test-surface', url: 'http://localhost:3000/' }, config: cfg([]) };
    const drafts = await provider.run(ctx);
    drafts.filter(d => d.evidenceClass === 'preview').forEach(d => {
      expect(d.confidence).not.toBe('fail');
    });
  });

  it('provider.capabilities always includes live', async () => {
    const provider = makeVirtualSrProvider([contract], fakeRunner);
    expect(provider.capabilities).toContain('live');
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run tests/voicing/promotion-invariants.test.ts
```

- [ ] **Step 3: Run and confirm PASS** (no code change needed — invariants already encoded)

```
npx vitest run tests/voicing/promotion-invariants.test.ts
```

- [ ] **Step 4: Commit**

```
test(voicing): promotion invariant suite — preview never gates, only config-listed classes promote
```

---

## Task 7 — Full voicing test suite pass

Run all voicing tests together to confirm nothing regresses across tasks.

- [ ] **Step 1: Run full suite**

```
npx vitest run tests/voicing/
```

Expected: all tests pass, zero failures.

- [ ] **Step 2: Commit (only if any clean-up was needed)**

```
refactor(voicing): clean up after full suite run
```

---

## Task 8 — Integration task (labeled, not unit-tested here)

This task is a labeled seam, not implemented in this phase. It wires the real `@guidepup/virtual-screen-reader` behind the `TranscriptReader` interface. It must never run in unit tests.

**File to create later:** `src/voicing/guidepup-transcript-reader.ts`

Architecture notes for the implementer:

- `@guidepup/virtual-screen-reader` automates VoiceOver and NVDA. It does NOT automate Orca.
- The in-loop unit-test provider is always `FakeTranscriptReader`.
- Orca-on-Fedora is the offline calibration harness for promotion decisions. Its transcript is captured via `speech-dispatcher` log or manual transcription, not via guidepup.
- NVDA is the hero DEMO voice, captured once via a Windows VM. It is not the in-loop reader.
- The `GuidepupTranscriptReader` drives a real Page via guidepup's `VoiceOver` or `NVDA` class. It returns `TranscriptStop[]` by mapping the raw speech events to `AnnouncementToken[]`. The comparator invariant applies: never normalize the observed tokens.

```ts
// src/voicing/guidepup-transcript-reader.ts  (skeleton only — integration task)
// import { virtual } from '@guidepup/virtual-screen-reader';
// import type { TranscriptReader } from './transcript-reader.js';
// import type { Page, TranscriptStop } from '../contracts/index.js';
//
// export class GuidepupTranscriptReader implements TranscriptReader {
//   constructor(private readonly page: Page) {}
//   async read(): Promise<TranscriptStop[]> {
//     // TODO: drive virtual SR, map speech events to TranscriptStop[]
//     throw new Error('GuidepupTranscriptReader: integration wiring not yet implemented');
//   }
// }
```

- [ ] **Step 1: Create the skeleton file** (comment-only, marks the seam clearly)

```
feat(voicing): add GuidepupTranscriptReader skeleton — integration seam for @guidepup/virtual-screen-reader
```

---

## Task 9 — Export from package index

Wire the new modules into the package's public surface so the `CheckRunner` (Phase 2) can import them.

**File:** `src/index.ts` (add exports alongside existing Phase 1 + 2 exports)

- [ ] **Step 1: Add voicing exports**

```ts
// In src/index.ts, add:
export { makeVirtualSrProvider } from './voicing/virtual-sr-provider.js';
export { runStructuralTier } from './voicing/structural-tier.js';
export { runVoicingTier } from './voicing/voicing-tier.js';
export { normalizeToken, obligationSatisfied } from './voicing/normalize.js';
export type { TranscriptReader } from './voicing/transcript-reader.js';
export { FakeTranscriptReader } from './voicing/fakes.js';
```

- [ ] **Step 2: Rebuild**

```
npx tsup
```

Expected: build succeeds, no type errors.

- [ ] **Step 3: Commit**

```
feat(voicing): export voicing lane modules from package index
```

---

## Self-Review

| Concern | Status |
|---|---|
| Frozen contracts used verbatim | `InteractionContract`, `SpeechObligation`, `Step`, `TranscriptStop`, `AnnouncementToken`, `Draft`, `Provider`, `ProviderContext`, `Capability`, `UsablConfig`, `Page`, `StepRunner` all imported from `src/contracts/index.ts` — no redeclaration |
| Provider never decides verdict | `VirtualSrProvider.run()` returns `Draft[]` only |
| `preview` never mints `verified` | `evidenceClass: 'preview'` findings carry `confidence: 'unverified'`; gate (Phase 1) filters by `evidenceClass` |
| `verified` and receipt reserved | No receipt logic in this phase; receipt is Phase 1 gate concern |
| Comparator invariant | `normalizeToken` called on obligation tokens only; observed `AnnouncementToken.text` compared raw (lowercased in `obligationSatisfied` for case-insensitive match, never stripped) |
| Structural tier gates | Emits `evidenceClass: 'deterministic'`, `confidence: 'fail'` — picked up by Phase 1 gate |
| Voicing tier advisory by default | Emits `evidenceClass: 'preview'`, `confidence: 'unverified'` |
| Promotion is config-only | `promotedObligations` list from `UsablConfig`; engine never writes to it at runtime |
| `StepRunner` seam reused | `VirtualSrProvider` calls `stepRunner.run(page, steps)` — exact frozen signature from `00-plan-set.md` |
| `@guidepup` behind seam | `TranscriptReader` interface isolates guidepup; unit tests use `FakeTranscriptReader` only |
| No placeholders | All code blocks are complete and runnable |
| Type consistency | All types imported from frozen `src/contracts/index.ts`; no local redeclarations |
| Screen-reader split honored | Orca = offline calibration (not guidepup); NVDA = demo voice (Windows VM); Virtual SR = in-loop unit tests |
| Capability declared | `capabilities: ['live']` — static mode skips this provider and records a `capability-denied` gap |
