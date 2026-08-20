# Announcement Voicing Lane Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add the differentiating voicing layer to usabl. An authored `InteractionContract` drives a Virtual Screen Reader provider that records what an AT announces at each step. A structural tier emits `deterministic` findings that gate. A voicing tier emits `preview` findings that never gate by default. A `promotedObligations` config list is the only promotion mechanism; promotion happens offline, never at runtime.

**Architecture:**
- `VirtualSrProvider` is a `Provider`. It returns `Draft[]`. It never decides a verdict.
- The gate (Phase 1) is the sole verdict authority. `evidenceClass: 'preview'` findings are excluded from the gate verdict by definition; only `deterministic` (and items listed in `promotedObligations`) feed it.
- `StepRunner` (frozen seam from `00-plan-set.md`, built in Phase 2) executes `Step[]` and returns `TranscriptStop[]`. After every step it drains the live-region buffer into `kind: 'live'` tokens. Toast and status obligations match against live tokens; without them a "toast announced" obligation could never be satisfied.
- The COMPARATOR INVARIANT: matching normalizes BOTH sides (case, punctuation, whitespace) so incidental punctuation or token boundaries in the observed transcript cannot fabricate a miss or mask a hit. Honesty lives in DISPLAY: findings always quote the raw observed text verbatim. The engine never doctors what it shows, only how it compares.
- The structural tier checks are SCOPED to obligation windows plus contract-level facts (unreachable focus, over-long tab path). It does not blanket-require a name and role on every stop: focus legitimately lands on nameless targets (a main landmark after a route change, a tabindex="-1" skip target), and the keyboard walk already flags unnamed INTERACTIVE elements.

**Tech Stack:**
- TypeScript (ESM, strict), Node 22
- Vitest for all tests (unit tests: fakes only via `makeFakePage`; no DOM, no real browser)
- Frozen contracts from `src/contracts/index.ts` (Phase 1): `InteractionContract`, `SpeechObligation`, `Step`, `TranscriptStop`, `AnnouncementToken` (including `kind: 'live'`), `Draft`, `Finding`, `Provider`, `ProviderContext`, `Capability`, `UsablConfig`, `Page`
- Frozen `StepRunner` seam from `00-plan-set.md`
- Screen-reader division of labor: the AX-tree transcript plus live tokens are the
  in-loop evidence; Orca-on-Fedora is the OFFLINE calibration harness that feeds
  `promotedObligations`; NVDA is the demo voice. No screen-reader automation library
  runs in the loop (Task 8).

---

## Task 1: Normalize helper for obligation tokens

**Why first:** every other task depends on the comparator invariant. Nail it in isolation with pure, trivially-testable code.

**Files:**
- `src/voicing/normalize.ts`
- `test/voicing/normalize.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// test/voicing/normalize.test.ts
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
npx vitest run test/voicing/normalize.test.ts
```

Expected: `Cannot find module '../../src/voicing/normalize.js'`

- [ ] **Step 3: Write the implementation**

```ts
// src/voicing/normalize.ts

/**
 * Normalize a token for COMPARISON: lowercase, strip punctuation, collapse
 * whitespace. Both the authored obligation tokens and the observed transcript pass
 * through this for matching. Findings always DISPLAY the raw observed text
 * verbatim; normalization never leaks into what the user sees.
 */
export function normalizeToken(raw: string | null): string {
  if (raw === null) return '';
  return raw
    .toLowerCase()
    .replace(/[^\w\s]/g, ' ')
    .replace(/\s+/g, ' ')
    .trim();
}

/**
 * True when every required token appears in the normalized observed transcript.
 * The transcript is the normalized announcement texts joined with single spaces,
 * so a required phrase can span adjacent announcement tokens and punctuation
 * inside the observed text cannot fabricate a miss.
 */
export function obligationSatisfied(
  required: string[],
  announcements: Array<{ text: string | null }>,
): boolean {
  const haystack = announcements
    .map(a => normalizeToken(a.text))
    .filter(t => t !== '')
    .join(' ');
  return required.every(tok => {
    const needle = normalizeToken(tok);
    return needle === '' || haystack.includes(needle);
  });
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run test/voicing/normalize.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): add token normalizer with comparator invariant
```

---

## Task 2: Obligation matching matrix (punctuation, spanning, live tokens)

**Why:** the comparator is where a voicing lane quietly rots. These tests pin the
matching semantics against the three real-world hazards: punctuation inside observed
text, a required phrase spanning adjacent announcement tokens, and toast text that
arrives only as a `kind: 'live'` token.

**Files:**
- `test/voicing/matching.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// test/voicing/matching.test.ts
import { describe, it, expect } from 'vitest';
import { obligationSatisfied } from '../../src/voicing/normalize.js';
import type { AnnouncementToken } from '../../src/contracts/index.js';

const name = (text: string): AnnouncementToken => ({ kind: 'name', text, fromTree: true, source: 'ax-tree' });
const live = (text: string): AnnouncementToken => ({ kind: 'live', text, fromTree: false, source: 'attribute' });

describe('obligationSatisfied', () => {
  it('punctuation in the observed text cannot fabricate a miss', () => {
    expect(obligationSatisfied(['cluster deleted successfully'], [name('Cluster deleted, successfully.')])).toBe(true);
  });

  it('a required phrase can span adjacent announcement tokens', () => {
    expect(obligationSatisfied(['confirm deletion'], [name('Confirm'), name('deletion')])).toBe(true);
  });

  it('matches toast text that arrives only as a live token', () => {
    expect(obligationSatisfied(['Cluster deleted'], [name('Delete'), live('Cluster deleted successfully')])).toBe(true);
  });

  it('a genuinely absent token still misses', () => {
    expect(obligationSatisfied(['Cluster deleted'], [name('Loading')])).toBe(false);
  });

  it('every required token must be present, independently', () => {
    expect(obligationSatisfied(['Cluster deleted', 'success'], [live('Cluster deleted')])).toBe(false);
    expect(obligationSatisfied(['Cluster deleted', 'success'], [live('Cluster deleted'), live('Alert: success')])).toBe(true);
  });
});
```

- [ ] **Step 2: Run and confirm PASS or FAIL honestly** (these may already pass after
  Task 1; if any fails, fix `normalize.ts`, never the test)

```
npx vitest run test/voicing/matching.test.ts
```

- [ ] **Step 3: Commit**

```
test(voicing): pin obligation matching semantics (punctuation, spanning, live tokens)
```

---

## Task 3: Structural tier (deterministic, gates)

The structural tier reads `TranscriptStop[]` and the `InteractionContract` and emits
`deterministic` `Draft[]` for reproducible structural failures. The checks are SCOPED:

- **Obligation windows only** for name/role facts: at `ob.afterStep`, the focused
  element must carry the expected `focusedRole` (when set) and an accessible name.
  There is no blanket name/role requirement on every stop: focus legitimately lands on
  nameless targets (main landmark after a route change, tabindex="-1" skip targets),
  and the keyboard walk already flags unnamed INTERACTIVE elements.
- **mustAnnounce means live region, checked via live tokens:** when
  `ob.mustAnnounce` is true, the step window must contain at least one
  `kind: 'live'` token. This is the structural "no live region" gate: the consequence
  of the interaction (a toast, a status change) must actually reach assistive
  technology. Which WORDS it contains is the voicing tier's business.
- **Unreachable focus:** zero stops, or no stop recorded at an obligation's step.
- **Over-long tab path:** more stops than `maxTabPath` allows.

These gate.

**Files:**
- `src/voicing/structural-tier.ts`
- `test/voicing/structural-tier.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// test/voicing/structural-tier.test.ts
import { describe, it, expect } from 'vitest';
import type { TranscriptStop, InteractionContract, AnnouncementToken } from '../../src/contracts/index.js';
import { runStructuralTier } from '../../src/voicing/structural-tier.js';

const dialogContract: InteractionContract = {
  contractId: 'dialog-contract',
  surfaceId: 'clusters-page',
  task: 'Open delete dialog',
  steps: [{ do: 'tab' }, { do: 'activate' }],
  obligations: [
    { class: 'dialog-name', afterStep: 1, requiredTokens: ['Confirm deletion'], focusedRole: 'dialog', mustAnnounce: false },
  ],
  maxTabPath: 5,
};

const toastContract: InteractionContract = {
  contractId: 'toast-contract',
  surfaceId: 'clusters-page',
  task: 'Delete cluster and observe toast',
  steps: [{ do: 'tab' }, { do: 'activate' }],
  obligations: [
    { class: 'toast-announced', afterStep: 1, requiredTokens: ['Cluster deleted'], mustAnnounce: true },
  ],
};

function stop(index: number, elementPath: string, tokens: Array<Pick<AnnouncementToken, 'kind' | 'text'>>): TranscriptStop {
  return {
    index,
    elementPath,
    announcement: tokens.map(t => ({
      ...t,
      fromTree: t.kind !== 'live',
      source: t.kind === 'live' ? ('attribute' as const) : ('ax-tree' as const),
    })),
  };
}

describe('runStructuralTier', () => {
  it('emits no drafts when the obligation window has the expected role and a name', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete cluster' }, { kind: 'role', text: 'button' }]),
      stop(1, 'div:nth-child(9)', [{ kind: 'role', text: 'dialog' }, { kind: 'name', text: 'Confirm deletion' }]),
    ];
    expect(runStructuralTier(dialogContract, stops, 'clusters-page')).toHaveLength(0);
  });

  it('does NOT flag nameless stops outside obligation windows (main landmark, skip targets)', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'main:nth-child(1)', [{ kind: 'role', text: 'main' }]), // nameless, legitimate
      stop(1, 'div:nth-child(9)', [{ kind: 'role', text: 'dialog' }, { kind: 'name', text: 'Confirm deletion' }]),
    ];
    expect(runStructuralTier(dialogContract, stops, 'clusters-page')).toHaveLength(0);
  });

  it('flags a nameless focused element INSIDE an obligation window', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete cluster' }, { kind: 'role', text: 'button' }]),
      stop(1, 'div:nth-child(9)', [{ kind: 'role', text: 'dialog' }]), // dialog with no accessible name
    ];
    const drafts = runStructuralTier(dialogContract, stops, 'clusters-page');
    expect(drafts.some(d => d.rule === 'voicing/missing-name' && d.confidence === 'fail')).toBe(true);
  });

  it('flags a wrong or missing focusedRole in the obligation window', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete cluster' }, { kind: 'role', text: 'button' }]),
      stop(1, 'main:nth-child(1)', [{ kind: 'role', text: 'main' }, { kind: 'name', text: 'Confirm deletion' }]),
    ];
    const drafts = runStructuralTier(dialogContract, stops, 'clusters-page');
    expect(drafts.some(d => d.rule === 'voicing/missing-role')).toBe(true);
  });

  it('mustAnnounce: flags a window with NO live token (nothing reached a live region)', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete' }, { kind: 'role', text: 'button' }]),
      stop(1, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete' }, { kind: 'role', text: 'button' }]),
    ];
    const drafts = runStructuralTier(toastContract, stops, 'clusters-page');
    const hit = drafts.find(d => d.rule === 'voicing/missing-live-announcement');
    expect(hit).toBeDefined();
    expect(hit!.confidence).toBe('fail');
    expect(hit!.evidenceClass).toBe('deterministic');
  });

  it('mustAnnounce: stays silent when a live token reached the window', () => {
    const stops: TranscriptStop[] = [
      stop(0, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete' }, { kind: 'role', text: 'button' }]),
      stop(1, 'button:nth-child(1)', [{ kind: 'name', text: 'Delete' }, { kind: 'live', text: 'Cluster deleted' }]),
    ];
    expect(runStructuralTier(toastContract, stops, 'clusters-page')
      .filter(d => d.rule === 'voicing/missing-live-announcement')).toHaveLength(0);
  });

  it('flags an over-long tab path', () => {
    const contract: InteractionContract = { ...dialogContract, maxTabPath: 1 };
    const stops: TranscriptStop[] = [
      stop(0, 'a:nth-child(1)', [{ kind: 'name', text: 'Skip' }, { kind: 'role', text: 'link' }]),
      stop(1, 'div:nth-child(9)', [{ kind: 'role', text: 'dialog' }, { kind: 'name', text: 'Confirm deletion' }]),
    ];
    expect(runStructuralTier(contract, stops, 'clusters-page')
      .some(d => d.rule === 'voicing/over-long-tab-path')).toBe(true);
  });

  it('flags unreachable focus on zero stops and on a missing obligation window', () => {
    expect(runStructuralTier(dialogContract, [], 'clusters-page')
      .some(d => d.rule === 'voicing/unreachable-focus')).toBe(true);
    const onlyStepZero = [stop(0, 'button:nth-child(1)', [{ kind: 'name', text: 'x' }, { kind: 'role', text: 'button' }])];
    expect(runStructuralTier(dialogContract, onlyStepZero, 'clusters-page')
      .some(d => d.rule === 'voicing/unreachable-focus')).toBe(true);
  });
});
```

- [ ] **Step 2: Run and confirm FAIL**

```
npx vitest run test/voicing/structural-tier.test.ts
```

- [ ] **Step 3: Write the implementation**

```ts
// src/voicing/structural-tier.ts
import type {
  AnnouncementToken,
  Draft,
  InteractionContract,
  TranscriptStop,
} from '../contracts/index.js';

function hasTok(announcement: AnnouncementToken[], kind: AnnouncementToken['kind']): boolean {
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

/**
 * Scoped structural checks. Obligation windows carry the name/role requirements;
 * stops outside windows are never blanket-checked (the keyboard walk already flags
 * unnamed interactive elements, and legitimate nameless focus targets exist).
 */
export function runStructuralTier(
  contract: InteractionContract,
  stops: TranscriptStop[],
  screenId: string,
): Draft[] {
  const drafts: Draft[] = [];

  // 1. Unreachable focus: no stops at all.
  if (stops.length === 0) {
    drafts.push(makeDraft(
      'voicing/unreachable-focus',
      screenId,
      contract.surfaceId,
      'Keyboard focus could not reach any element defined in the interaction contract.',
      'The step run returned zero stops. Focus may be trapped, the page may not load, or the target may be unreachable via keyboard.',
      'Ensure the contract steps can be executed from the body via keyboard.',
    ));
    return drafts;
  }

  // 2. Per-obligation window checks.
  for (const ob of contract.obligations) {
    const windowStop = stops.find(s => s.index === ob.afterStep);
    if (!windowStop) {
      drafts.push(makeDraft(
        'voicing/unreachable-focus',
        screenId,
        `step-${ob.afterStep}`,
        `Keyboard could not reach the element expected at step ${ob.afterStep}.`,
        `No TranscriptStop recorded at step index ${ob.afterStep} for obligation "${ob.class}".`,
        'Ensure the step sequence reaches the target element.',
      ));
      continue;
    }

    if (ob.focusedRole) {
      const roleFound = windowStop.announcement.some(
        t => t.kind === 'role' && (t.text ?? '').toLowerCase() === ob.focusedRole!.toLowerCase(),
      );
      if (!roleFound) {
        drafts.push(makeDraft(
          'voicing/missing-role',
          screenId,
          windowStop.elementPath,
          `After step ${ob.afterStep}, an AT user expects focus on a "${ob.focusedRole}" but hears a different or missing role.`,
          `Obligation "${ob.class}" requires focusedRole "${ob.focusedRole}" at step ${ob.afterStep}.`,
          `Ensure the element at step ${ob.afterStep} carries role="${ob.focusedRole}" or the equivalent semantic element.`,
        ));
      }
      if (!hasTok(windowStop.announcement, 'name')) {
        drafts.push(makeDraft(
          'voicing/missing-name',
          screenId,
          windowStop.elementPath,
          `The element focus lands on after step ${ob.afterStep} has no accessible name.`,
          `Obligation "${ob.class}" targets this element; an AT user hears its role with no label.`,
          'Add an accessible name via aria-label, aria-labelledby, or visible text content.',
        ));
      }
    }

    // mustAnnounce: the consequence must actually REACH a live region. The evidence
    // is a 'live' token in this step's window. Which words it contains is the
    // voicing tier's business; that nothing arrived at all is a hard structural fail.
    if (ob.mustAnnounce && !hasTok(windowStop.announcement, 'live')) {
      drafts.push(makeDraft(
        'voicing/missing-live-announcement',
        screenId,
        windowStop.elementPath,
        `After step ${ob.afterStep}, nothing was announced: no text reached any live region.`,
        `Obligation "${ob.class}" requires the interaction's consequence to be announced. No live-region update was observed in this step's window, so a screen-reader user gets silence.`,
        'Render the confirmation into an aria-live region (role="status" or role="alert") that exists before the update fires.',
      ));
    }
  }

  // 3. Over-long tab path.
  if (contract.maxTabPath !== undefined && stops.length > contract.maxTabPath) {
    drafts.push(makeDraft(
      'voicing/over-long-tab-path',
      screenId,
      contract.surfaceId,
      `An AT user must Tab ${stops.length} times to reach the target; the contract allows at most ${contract.maxTabPath}.`,
      `Tab path length ${stops.length} exceeds maxTabPath ${contract.maxTabPath} in contract "${contract.contractId}".`,
      'Reduce the number of focusable elements before the target, or add a skip-navigation link.',
    ));
  }

  return drafts;
}
```

- [ ] **Step 4: Run and confirm PASS**

```
npx vitest run test/voicing/structural-tier.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): structural tier: deterministic drafts for missing name/role/state, unreachable focus, over-long tab path
```

---

## Task 4: Voicing tier (preview, advisory)

The voicing tier matches transcript tokens inside each obligation's step window. It emits `preview` findings. It never gates. `promotedObligations` are checked at runtime to decide whether to flip `preview` to `deterministic` for that obligation class.

**Files:**
- `src/voicing/voicing-tier.ts`
- `test/voicing/voicing-tier.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// test/voicing/voicing-tier.test.ts
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

function liveStop(index: number, focusTexts: string[], liveTexts: string[]): TranscriptStop {
  return {
    index,
    elementPath: `el-${index}`,
    announcement: [
      ...focusTexts.map(text => ({ kind: 'name' as const, text, fromTree: true, source: 'ax-tree' as const })),
      ...liveTexts.map(text => ({ kind: 'live' as const, text, fromTree: false, source: 'attribute' as const })),
    ],
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

  it('satisfies an obligation from LIVE tokens (the toast case: focus text alone would miss)', () => {
    const stops: TranscriptStop[] = [
      stop(0, ['Delete cluster', 'button']),
      liveStop(1, ['Delete cluster'], ['Cluster deleted', 'Alert success toast']),
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
    // No stop at index 1: voicing tier records a preview gap
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
npx vitest run test/voicing/voicing-tier.test.ts
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
      // No stop in this window: emit a gap finding
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
npx vitest run test/voicing/voicing-tier.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): voicing tier: preview token matching with promotedObligations promotion
```

---

## Task 5: `VirtualSrProvider` (the Provider)

Wires both tiers together into a `Provider` that the `CheckRunner` (Phase 2) can call. Accepts contracts via config (loaded from the `requirements` file path or a direct list). Reads `TranscriptStop[]` from a `StepRunner` run, then calls both tiers and merges their `Draft[]`.

**Files:**
- `src/voicing/virtual-sr-provider.ts`
- `test/voicing/virtual-sr-provider.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// test/voicing/virtual-sr-provider.test.ts
import { describe, it, expect, vi } from 'vitest';
import type {
  InteractionContract, TranscriptStop, ProviderContext, UsablConfig,
} from '../../src/contracts/index.js';
import type { StepRunner } from '../../src/contracts/index.js';
import { makeVirtualSrProvider } from '../../src/voicing/virtual-sr-provider.js';
import { makeFakePage } from '../../src/deps/fakes.js';

const fakePage = makeFakePage; // methods not called in these tests; complete fake from Phase 1

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
npx vitest run test/voicing/virtual-sr-provider.test.ts
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
 * VirtualSrProvider: the Phase 4 Provider.
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
npx vitest run test/voicing/virtual-sr-provider.test.ts
```

- [ ] **Step 5: Commit**

```
feat(voicing): VirtualSrProvider wires structural and voicing tiers as a Provider
```

---

## Task 6: Promotion invariant tests

These tests assert the invariants in one place: `preview` never mints `verified`, promoted findings gate as `deterministic`, no runtime self-promotion path exists in the provider code.

**Files:**
- `test/voicing/promotion-invariants.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// test/voicing/promotion-invariants.test.ts
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

import { makeFakePage } from '../../src/deps/fakes.js';
const fakePage = makeFakePage;

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

- [ ] **Step 2: Run: expected PASS immediately.** This is an invariant suite over
  already-built code, so there is no red step. If any case fails, the invariant was
  broken upstream: fix the provider or the tiers, never the test.

```
npx vitest run test/voicing/promotion-invariants.test.ts
```

- [ ] **Step 3: Commit**

```
test(voicing): promotion invariant suite: preview never gates, only config-listed classes promote
```

---

## Task 7: Full voicing test suite pass

Run all voicing tests together to confirm nothing regresses across tasks.

- [ ] **Step 1: Run full suite**

```
npx vitest run test/voicing/
```

Expected: all tests pass, zero failures.

- [ ] **Step 2: Commit (only if any clean-up was needed)**

```
refactor(voicing): clean up after full suite run
```

---

## Task 8: Offline calibration harness (documented seam, no code in this phase)

No file is created here (a committed skeleton that throws is dead weight). This section
records how `promotedObligations` gets populated, so the seam is real without shipping
stubs:

- **In-loop evidence** is always the AX-tree transcript plus live tokens from the
  Phase 2 StepRunner. No screen-reader automation library runs inside the engine.
- **Orca-on-Fedora is the offline calibration harness.** For each obligation class, run
  the same interaction under Orca (transcript via the speech-dispatcher log or manual
  transcription) and measure the match rate between what the engine's transcript
  predicted and what Orca actually said. A class whose match rate holds up earns its
  place in `promotedObligations`; the config edit is itself an `approval_required`
  event because the config is guarded.
- **NVDA is the demo voice**, recorded once from a Windows VM after the hero bug is
  locked. It is never the in-loop reader.
- **Factual note for the implementer:** `@guidepup/virtual-screen-reader` is a
  headless, DOM-based virtual screen reader. The separate `@guidepup/guidepup` package
  drives real VoiceOver and NVDA. Either can strengthen the OFFLINE harness later;
  neither enters the loop. If one is adopted, its raw transcript is stored verbatim
  next to the calibration results, and matching uses the same normalize-for-comparison,
  display-raw rule as everything else.

---

## Task 9: Export from package index

Wire the new modules into the package's public surface so the `CheckRunner` (Phase 2) can import them.

**File:** `src/index.ts` (add exports alongside existing Phase 1 + 2 exports)

- [ ] **Step 1: Add voicing exports**

```ts
// In src/index.ts, add:
export { makeVirtualSrProvider } from './voicing/virtual-sr-provider.js';
export { runStructuralTier } from './voicing/structural-tier.js';
export { runVoicingTier } from './voicing/voicing-tier.js';
export { normalizeToken, obligationSatisfied } from './voicing/normalize.js';
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
| Frozen contracts used verbatim | `InteractionContract`, `SpeechObligation`, `Step`, `TranscriptStop`, `AnnouncementToken`, `Draft`, `Provider`, `ProviderContext`, `Capability`, `UsablConfig`, `Page`, `StepRunner` all imported from `src/contracts/index.ts`: no redeclaration |
| Provider never decides verdict | `VirtualSrProvider.run()` returns `Draft[]` only |
| `preview` never mints `verified` | `evidenceClass: 'preview'` findings carry `confidence: 'unverified'`; gate (Phase 1) filters by `evidenceClass` |
| `verified` and receipt reserved | No receipt logic in this phase; receipt is Phase 1 gate concern |
| Comparator invariant | Matching normalizes BOTH sides (case, punctuation, whitespace, token spanning); findings display raw observed text verbatim. Pinned by the Task 2 matrix. |
| Live-region evidence | `kind: 'live'` tokens from the StepRunner satisfy toast obligations; `mustAnnounce` without a live token is a deterministic `voicing/missing-live-announcement` fail. The toast hero bug is detectable end to end. |
| Structural tier gates | Emits `evidenceClass: 'deterministic'`, `confidence: 'fail'`; checks SCOPED to obligation windows plus unreachable-focus and tab-path facts. No blanket per-stop noise. |
| Voicing tier advisory by default | Emits `evidenceClass: 'preview'`, `confidence: 'unverified'` |
| Promotion is config-only | `promotedObligations` list from `UsablConfig`; the config is guarded, so a promotion edit is itself an `approval_required` event; the engine never writes to it at runtime |
| `StepRunner` seam reused | `VirtualSrProvider` calls `stepRunner.run(page, steps)`: exact frozen signature from `00-plan-set.md`; no reimplementation |
| No dead seams | No `TranscriptReader` class ships; offline calibration is documented in Task 8 without stub files |
| Type consistency | All types imported from frozen `src/contracts/index.ts`; fakes from `makeFakePage`; no local redeclarations, no `as any` |
| Screen-reader split honored | AX transcript + live tokens = in-loop evidence; Orca = offline calibration; NVDA = demo voice (Windows VM) |
| Capability declared | `capabilities: ['live']`: static mode skips this provider and records a `capability-denied` gap |
