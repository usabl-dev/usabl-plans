# Detection Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the `Provider` interface and build the deterministic scan layers: axe-core (WCAG baseline; violations gate, axe `incomplete` maps to `unverified`), the PatternFly rulepack (static rules PLUS interaction probes that click and key through modals and menus), and the keyboard walk (role-aware tab probing that reports `unverified` rather than guessing). Deliver the live-region capture primitives, a real `CheckRunner` that opens a `Page` per screen, records the transcript into `ScreenScan.stops`, runs every `Provider`, and collects provider gaps into `ScreenScan.gaps`, and a real `BrowserDriver` that reads names, roles, and states from the accessibility tree over CDP.

**Architecture:**
- Providers return `Draft[]` only. No provider decides a verdict.
- `CheckRunner` is the only thing that touches `BrowserDriver`; it satisfies `Deps.checkRunner` from Phase 1. `scan()` NEVER throws: a screen that cannot be scanned returns a `ScreenScan` whose `gaps` explain why. The gate turns any gap into `not_covered`.
- Static mode = capability filtering: a provider whose `capabilities` include a denied capability is skipped and records a `CoverageGap` with `state: 'capability-denied'`. Gaps flow into `ScreenScan.gaps`; `run()` merges them into `coverage.gaps`. No silent passes.
- Rules that only exist through interaction (modal focus return, focus into dialog, focus into menu) are interaction probes: they `click()` the widget open, key through it, and assert with the honest predicates `activeElementIs` / `activeElementWithin`. A static one-shot DOM pass cannot check these and must not pretend to.
- Rules 2/3/4 mutate page state, so the rulepack runs its checks SEQUENTIALLY, static rules first, probes last. The transcript pass and axe run before the rulepack.
- The real `BrowserDriver` reads the accessibility tree over CDP. A DOM approximation (`aria-label || innerText`) is forbidden: the transcript must be what assistive technology gets.
- The duplicate between axe `button-name`, `pf-icon-button-name`, and the walk's unnamed-interactive rule is resolved by the gate's equivalence suppression (Phase 1 Task 8, pf preferred). Providers report honestly; the gate dedups.
- Every unit test builds fakes with `makeFakePage` / `makeFakeDeps` from `src/deps/fakes.ts`. No hand-rolled `Page` literals, no `as any`.

**Tech Stack:** TypeScript (ESM, strict), Node 22, Vitest, axe-core + `@axe-core/playwright` (integration bridge), Playwright (real integration only).

**Selector note:** the fixture targets PatternFly v6, so component-class selectors use `pf-v6-*`. Selectors that can be expressed through ARIA semantics use ARIA first. Verify every selector against the rendered fixture during Phase 7: if a selector matches nothing on the broken fixture, the rule is dead and the planted-count oracle will catch it.

---

## Task 1: Provider interface and capability-filtering wiring

**Files:**
- Modify: `src/contracts/index.ts`
- Create: `src/providers/index.ts`
- Create: `test/helpers.ts`
- Create: `test/providers/index.test.ts`

- [ ] Create `test/helpers.ts` (the one shared config builder for provider tests; the frozen `UsablConfig` shape, never an invented one):

```ts
// test/helpers.ts
import type { UsablConfig } from '../src/contracts/index.js';

export function testConfig(overrides: Partial<UsablConfig> = {}): UsablConfig {
  return {
    appBaseUrl: 'http://127.0.0.1:5173',
    uiFileGlobs: ['src/**'],
    discovery: { routerFile: 'src/App.tsx', wideBlastGlobs: [] },
    surfaces: [],
    guardedPaths: ['usabl.config.json'],
    ...overrides,
  };
}
```

- [ ] Write a failing test: a `Provider` with `capabilities: ['live']` is NOT called when `allowedCapabilities` is `[]`, and a `CoverageGap` with `state: 'capability-denied'` is recorded.

```ts
// test/providers/index.test.ts
import { describe, it, expect, vi } from 'vitest';
import { runProviders } from '../../src/providers/index.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import { testConfig } from '../helpers.js';
import type { Provider, ProviderContext, Draft } from '../../src/contracts/index.js';

const ctx: ProviderContext = {
  page: makeFakePage(),
  screen: { id: 'home', url: 'http://localhost/' },
  config: testConfig(),
};

const draft: Draft = {
  rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
  screenId: 'home', elementPath: 'button', elementName: 'Submit', role: 'button',
  whatUserExperiences: 'Cannot distinguish button from background',
  why: 'Contrast ratio 2.1:1 below 4.5:1 threshold', fix: 'Increase contrast',
  evidence: {}, confidence: 'fail',
};

describe('runProviders', () => {
  it('skips a live provider in static mode and records a capability-denied gap', async () => {
    const p: Provider = { id: 'fake-live', layer: 'axe', capabilities: ['live'], run: vi.fn().mockResolvedValue([]) };
    const result = await runProviders([p], ctx, []);
    expect(p.run).not.toHaveBeenCalled();
    expect(result.drafts).toHaveLength(0);
    expect(result.gaps).toHaveLength(1);
    expect(result.gaps[0]).toMatchObject({ ref: 'provider:fake-live', state: 'capability-denied' });
    expect(result.gaps[0]!.reason).toBeTruthy();
  });

  it('calls the provider when its capabilities are allowed', async () => {
    const p: Provider = { id: 'fake-live', layer: 'axe', capabilities: ['live'], run: vi.fn().mockResolvedValue([draft]) };
    const result = await runProviders([p], ctx, ['live']);
    expect(p.run).toHaveBeenCalledWith(ctx);
    expect(result.drafts).toHaveLength(1);
    expect(result.gaps).toHaveLength(0);
  });

  it('a throwing provider becomes a gap, never a crash or a silent pass', async () => {
    const p: Provider = { id: 'boom', layer: 'pf', capabilities: [], run: vi.fn().mockRejectedValue(new Error('boom')) };
    const result = await runProviders([p], ctx, []);
    expect(result.gaps).toHaveLength(1);
    expect(result.gaps[0]).toMatchObject({ ref: 'provider:boom', state: 'not-covered' });
    expect(result.gaps[0]!.reason).toContain('boom');
  });
});
```

- [ ] Run `npx vitest run test/providers/index.test.ts`: expect FAIL (module not found).

- [ ] Add the frozen seam shapes to `src/contracts/index.ts` (verbatim from `00-plan-set.md`):

```ts
// src/contracts/index.ts: append after existing exports
export type Capability = 'live' | 'network' | 'secrets' | 'filesystem-write';

export interface ProviderContext {
  page: Page;
  screen: { id: string; url: string };
  config: UsablConfig;
}

export interface Provider {
  id: string;
  layer: string;
  capabilities: Capability[];
  run(ctx: ProviderContext): Promise<Draft[]>;
}

export interface ProviderRunResult {
  drafts: Draft[];
  gaps: CoverageGap[];
}
```

- [ ] Create `src/providers/index.ts`:

```ts
import type { Provider, ProviderContext, Capability, CoverageGap, Draft, ProviderRunResult } from '../contracts/index.js';

/**
 * Runs providers SEQUENTIALLY (interaction probes mutate page state; parallel
 * execution would interleave clicks and key presses). A denied capability or a
 * thrown provider error becomes a CoverageGap, never a crash or a silent pass.
 */
export async function runProviders(
  providers: Provider[],
  ctx: ProviderContext,
  allowedCapabilities: Capability[],
): Promise<ProviderRunResult> {
  const allowed = new Set(allowedCapabilities);
  const drafts: Draft[] = [];
  const gaps: CoverageGap[] = [];

  for (const provider of providers) {
    const denied = provider.capabilities.find((c) => !allowed.has(c));
    if (denied !== undefined) {
      gaps.push({
        ref: `provider:${provider.id}`,
        state: 'capability-denied',
        reason: `provider requires capability '${denied}' which is not permitted in this mode`,
      });
      continue;
    }
    try {
      drafts.push(...(await provider.run(ctx)));
    } catch (err) {
      gaps.push({
        ref: `provider:${provider.id}`,
        state: 'not-covered',
        reason: `provider failed: ${(err as Error).message}`,
      });
    }
  }

  return { drafts, gaps };
}
```

- [ ] Run `npx vitest run test/providers/index.test.ts`: expect PASS.
- [ ] Commit: `feat: add Provider interface and capability-filtering runProviders`

---

## Task 2: axe-core provider (violations gate; incomplete maps to unverified)

**Files:**
- Create: `src/providers/axe/index.ts`
- Create: `src/providers/axe/notes.ts`
- Create: `test/providers/axe.test.ts`

axe reports `violations` (definite failures) and `incomplete` (needs review). Dropping
`incomplete` silently would be a silent pass, so it maps to `confidence: 'unverified'`,
which the gate turns into `not_covered`.

- [ ] Write the failing test:

```ts
// test/providers/axe.test.ts
import { describe, it, expect } from 'vitest';
import { makeAxeProvider, type AxePage } from '../../src/providers/axe/index.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import { testConfig } from '../helpers.js';
import type { Page, ProviderContext } from '../../src/contracts/index.js';

function axePage(result: { violations: unknown[]; incomplete: unknown[] }): Page & AxePage {
  return Object.assign(makeFakePage(), { runAxe: async () => result });
}

const screen = { id: 'home', url: 'http://localhost/' };
const violation = {
  id: 'color-contrast', impact: 'serious', description: 'Contrast below AA',
  nodes: [{ target: ['button.submit'], html: '<button class="submit">Go</button>', failureSummary: 'Fix contrast', any: [] }],
};

describe('axe provider', () => {
  it('maps each violation node to a fail Draft', async () => {
    const page = axePage({ violations: [violation], incomplete: [] });
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    const drafts = await makeAxeProvider().run(ctx);
    expect(drafts).toHaveLength(1);
    const d = drafts[0]!;
    expect(d.rule).toBe('color-contrast');
    expect(d.layer).toBe('axe');
    expect(d.evidenceClass).toBe('deterministic');
    expect(d.confidence).toBe('fail');
    expect(d.screenId).toBe('home');
    expect(d.elementPath).toBe('button.submit');
    expect(d.severity).toBe('serious');
    expect(d.why).toBeTruthy();
    expect(d.fix).toBeTruthy();
  });

  it('maps each incomplete node to an unverified Draft (never dropped)', async () => {
    const page = axePage({ violations: [], incomplete: [violation] });
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    const drafts = await makeAxeProvider().run(ctx);
    expect(drafts).toHaveLength(1);
    expect(drafts[0]!.confidence).toBe('unverified');
  });

  it('returns an empty array on a clean page and declares live capability', async () => {
    const page = axePage({ violations: [], incomplete: [] });
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    expect(await makeAxeProvider().run(ctx)).toHaveLength(0);
    expect(makeAxeProvider().capabilities).toContain('live');
  });
});
```

- [ ] Run `npx vitest run test/providers/axe.test.ts`: expect FAIL.

- [ ] Create `src/providers/axe/notes.ts` (plain-language PatternFly overrides for common axe rule ids; same content as before):

```ts
// Plain-language PatternFly-specific why/fix overrides for axe rule ids.
export const AXE_NOTES: Record<string, { why: string; fix: string }> = {
  'color-contrast': {
    why: 'Low contrast makes text unreadable for low-vision users, especially on PatternFly action buttons and helper text.',
    fix: 'Increase the foreground/background contrast ratio to at least 4.5:1 (AA) or use a PatternFly semantic color token that meets this threshold.',
  },
  'button-name': {
    why: 'Screen readers announce the accessible name of buttons; an unnamed button gives users no context for the action.',
    fix: 'Add aria-label, aria-labelledby, or visible text to the button. PatternFly icon-only buttons must include an aria-label.',
  },
  'image-alt': {
    why: 'Images without alt text are skipped or read as the filename by screen readers.',
    fix: 'Add descriptive alt text, or alt="" if decorative. PatternFly <Brand> requires an alt prop.',
  },
  'label': {
    why: 'Form inputs without labels prevent screen-reader users from understanding what data to enter.',
    fix: 'Associate a <FormLabel> with its input via htmlFor/id, or use aria-labelledby.',
  },
  'aria-required-children': {
    why: 'Missing required ARIA child roles break the structural contract that AT relies on for navigation.',
    fix: 'Ensure the PatternFly composite widget contains the required owned roles (e.g., listbox > option).',
  },
};

export function noteFor(ruleId: string): { why: string; fix: string } {
  return AXE_NOTES[ruleId] ?? { why: '', fix: '' };
}
```

- [ ] Create `src/providers/axe/index.ts`:

```ts
import type { Provider, ProviderContext, Draft, Severity } from '../../contracts/index.js';
import { noteFor } from './notes.js';

// The axe Page extension attached by the real BrowserDriver (Task 7). Test fakes
// attach runAxe directly onto makeFakePage().
export interface AxePage {
  runAxe(): Promise<{ violations: AxeIssue[]; incomplete: AxeIssue[] }>;
}

interface AxeIssue {
  id: string;
  impact: string | null;
  description: string;
  nodes: Array<{ target: string[]; html: string; failureSummary: string; any: Array<{ data?: unknown }> }>;
}

function toSeverity(impact: string | null): Severity {
  const map: Record<string, Severity> = { critical: 'critical', serious: 'serious', moderate: 'moderate', minor: 'minor' };
  return map[impact ?? ''] ?? 'moderate';
}

function toDrafts(issues: AxeIssue[], screenId: string, confidence: 'fail' | 'unverified'): Draft[] {
  const drafts: Draft[] = [];
  for (const issue of issues) {
    const note = noteFor(issue.id);
    for (const node of issue.nodes) {
      drafts.push({
        rule: issue.id,
        layer: 'axe',
        severity: toSeverity(issue.impact),
        evidenceClass: 'deterministic',
        screenId,
        elementPath: node.target.join(' '),
        elementName: null,
        role: null,
        whatUserExperiences: confidence === 'unverified' ? `Needs review: ${issue.description}` : issue.description,
        why: note.why || node.failureSummary,
        fix: note.fix || node.failureSummary,
        evidence: { extra: { html: node.html, axeData: node.any[0]?.data ?? null } },
        confidence,
      });
    }
  }
  return drafts;
}

export function makeAxeProvider(): Provider {
  return {
    id: 'axe-core',
    layer: 'axe',
    capabilities: ['live'],
    async run(ctx: ProviderContext): Promise<Draft[]> {
      const { violations, incomplete } = await (ctx.page as unknown as AxePage).runAxe();
      return [
        ...toDrafts(violations, ctx.screen.id, 'fail'),
        ...toDrafts(incomplete, ctx.screen.id, 'unverified'),
      ];
    },
  };
}
```

- [ ] Run `npx vitest run test/providers/axe.test.ts`: expect PASS.
- [ ] Commit: `feat: add axe-core provider (violations fail, incomplete unverified)`

---

## Task 3: PatternFly rulepack: static rules

**Files:**
- Create: `src/providers/rulepack/selectors.ts`
- Create: `src/providers/rulepack/pf-toast-live-region.ts`
- Create: `src/providers/rulepack/pf-icon-button-name.ts`
- Create: `src/providers/rulepack/pf-kebab-expanded-state.ts`
- Create: `src/providers/rulepack/pf-table-header-assoc.ts`
- Create: `src/providers/rulepack/pf-row-action-name-unique.ts`
- Create: `src/providers/rulepack/pf-toolbar-labeled-when-repeated.ts`
- Create: `src/providers/rulepack/index.ts`
- Create: `test/providers/rulepack.test.ts`

These six checks are honestly static: each asserts a property that exists in the
resting DOM/AX tree. The two interaction rules (modal focus return, focus into
dialog/menu) are Task 4 probes. The toast rule's static half is containment: an alert
component rendered OUTSIDE any live container is a hard fail. The temporal half
(the region must exist BEFORE the update fires) is observed by the voicing lane's
live-token capture during interaction contracts (Phase 4); this rule never claims it.

- [ ] Create `src/providers/rulepack/selectors.ts`:

```ts
// PatternFly 6 selectors, ARIA-first where the semantics allow it.
// VERIFY against the rendered fixture in Phase 7: a selector that matches nothing
// on the broken fixture is a dead rule and the planted-count oracle will catch it.
export const SEL = {
  alert: '[class*="pf-v6-c-alert"]',
  liveContainer: '[aria-live], [role="status"], [role="alert"], [role="log"]',
  unnamedButton: 'button[aria-label=""], button:not([aria-label]):not([aria-labelledby])',
  menuToggle: '[aria-haspopup="menu"], [aria-haspopup="true"]',
  dialogTrigger: '[aria-haspopup="dialog"]',
  dialog: '[role="dialog"], [role="alertdialog"]',
  menu: '[role="menu"], [role="listbox"]',
  toolbar: '[class*="pf-v6-c-toolbar"]',
  rowActionButton: 'td button, td [role="button"]',
  unscopedTh: 'table th:not([scope]):not([id])',
} as const;
```

- [ ] Write failing tests (static rules; fakes from `makeFakePage`):

```ts
// test/providers/rulepack.test.ts
import { describe, it, expect } from 'vitest';
import { makeRulepackProvider } from '../../src/providers/rulepack/index.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import { testConfig } from '../helpers.js';
import type { AxNode, ElementRef, ProviderContext } from '../../src/contracts/index.js';

const screen = { id: 'home', url: 'http://localhost/' };
const ctxWith = (overrides: Parameters<typeof makeFakePage>[0]): ProviderContext =>
  ({ page: makeFakePage(overrides), screen, config: testConfig() });

describe('pf-toast-live-region (static containment)', () => {
  it('flags an alert rendered outside any live container', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> => {
        if (sel.includes('pf-v6-c-alert') && !sel.includes('aria-live')) return [{ selector: 'div:nth-child(3)' }];
        return []; // the "inside a live container" query finds nothing
      },
      axAt: async (): Promise<AxNode | null> => ({ name: 'Saved', role: 'generic', states: {} }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    const hits = drafts.filter((d) => d.rule === 'pf-toast-live-region');
    expect(hits).toHaveLength(1);
    expect(hits[0]!.confidence).toBe('fail');
    expect(hits[0]!.layer).toBe('pf');
  });

  it('stays silent when every alert sits inside a live container', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.includes('pf-v6-c-alert') ? [{ selector: 'div:nth-child(3)' }] : [],
    });
    // Both the all-alerts query and the inside-live query return the same element.
    const drafts = await makeRulepackProvider().run(ctx);
    expect(drafts.filter((d) => d.rule === 'pf-toast-live-region')).toHaveLength(0);
  });
});

describe('pf-icon-button-name', () => {
  it('flags an icon-only button whose AX name is empty', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.startsWith('button[aria-label=""]') ? [{ selector: 'button:nth-child(2)' }] : [],
      axAt: async (): Promise<AxNode | null> => ({ name: null, role: 'button', states: {} }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    const hits = drafts.filter((d) => d.rule === 'pf-icon-button-name');
    expect(hits).toHaveLength(1);
    expect(hits[0]!.elementName).toBeNull();
  });

  it('skips a button whose AX tree resolves a name (text content, labelledby)', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.startsWith('button[aria-label=""]') ? [{ selector: 'button:nth-child(2)' }] : [],
      axAt: async (): Promise<AxNode | null> => ({ name: 'Save', role: 'button', states: {} }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    expect(drafts.filter((d) => d.rule === 'pf-icon-button-name')).toHaveLength(0);
  });
});

describe('pf-kebab-expanded-state', () => {
  it('flags a menu toggle missing expanded/haspopup in the AX tree', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.includes('aria-haspopup') ? [{ selector: 'button:nth-child(4)' }] : [],
      axAt: async (): Promise<AxNode | null> => ({ name: 'Actions', role: 'button', states: {} }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    expect(drafts.filter((d) => d.rule === 'pf-kebab-expanded-state')).toHaveLength(1);
  });

  it('stays silent when the toggle exposes expanded and haspopup states', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.includes('aria-haspopup') ? [{ selector: 'button:nth-child(4)' }] : [],
      axAt: async (): Promise<AxNode | null> =>
        ({ name: 'Actions', role: 'button', states: { expanded: false, haspopup: 'menu' } }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    expect(drafts.filter((d) => d.rule === 'pf-kebab-expanded-state')).toHaveLength(0);
  });
});

describe('pf-row-action-name-unique', () => {
  it('flags row action buttons that all share one accessible name', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.startsWith('td button')
          ? [{ selector: 'tr:nth-child(1) button' }, { selector: 'tr:nth-child(2) button' }]
          : [],
      axAt: async (): Promise<AxNode | null> => ({ name: 'Delete', role: 'button', states: {} }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    expect(drafts.filter((d) => d.rule === 'pf-row-action-name-unique').length).toBeGreaterThanOrEqual(1);
  });
});

describe('pf-table-header-assoc', () => {
  it('flags a column header with neither scope nor id', async () => {
    const ctx = ctxWith({
      queryAll: async (sel: string): Promise<ElementRef[]> =>
        sel.startsWith('table th') ? [{ selector: 'table th:nth-child(1)' }] : [],
      axAt: async (): Promise<AxNode | null> => ({ name: 'Name', role: 'columnheader', states: {} }),
    });
    const drafts = await makeRulepackProvider().run(ctx);
    expect(drafts.filter((d) => d.rule === 'pf-table-header-assoc')).toHaveLength(1);
  });
});
```

- [ ] Run `npx vitest run test/providers/rulepack.test.ts`: expect FAIL.

- [ ] Create `src/providers/rulepack/pf-toast-live-region.ts` (containment by set difference; needs the driver's stable selectors, which are identical across queries for the same element):

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';
import { SEL } from './selectors.js';

/**
 * Static half of the toast rule: an alert component rendered OUTSIDE any live
 * container can never be announced. The temporal half (the region must exist
 * before the update fires) is observed by the voicing lane during interaction
 * contracts; this rule does not claim it.
 */
export async function checkToastLiveRegion(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const all = await page.queryAll(SEL.alert);
  if (all.length === 0) return [];
  const inside = new Set((await page.queryAll(`${SEL.liveContainer} ${SEL.alert}`)).map((e) => e.selector));

  const drafts: Draft[] = [];
  for (const el of all) {
    if (inside.has(el.selector)) continue;
    const node = await page.axAt(el.selector);
    drafts.push({
      rule: 'pf-toast-live-region',
      layer: 'pf',
      severity: 'serious',
      evidenceClass: 'deterministic',
      screenId: screen.id,
      elementPath: el.selector,
      elementName: node?.name ?? null,
      role: node?.role ?? null,
      whatUserExperiences: 'This alert is not announced to screen-reader users.',
      why: 'PatternFly alerts and toasts must render inside an aria-live region (role=status/alert) so assistive technology announces the update.',
      fix: 'Wrap alert groups in a container with role="status" (or role="alert" for critical alerts) that exists on initial page load.',
      evidence: { role: node ? { value: node.role, source: 'ax-tree', fromTree: true } : undefined },
      confidence: 'fail',
    });
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-icon-button-name.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';
import { SEL } from './selectors.js';

export async function checkIconButtonName(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const candidates = await page.queryAll(SEL.unnamedButton);
  const drafts: Draft[] = [];
  for (const el of candidates) {
    const node = await page.axAt(el.selector);
    if (node?.name) continue; // the AX tree resolves a name (text content, labelledby); not a defect
    drafts.push({
      rule: 'pf-icon-button-name',
      layer: 'pf',
      severity: 'serious',
      evidenceClass: 'deterministic',
      screenId: screen.id,
      elementPath: el.selector,
      elementName: null,
      role: node?.role ?? 'button',
      whatUserExperiences: 'Screen-reader users hear only "button" with no label, giving no context for the action.',
      why: 'Icon-only buttons (kebab toggles, pagination arrows, close buttons) carry no visible text. Without aria-label the accessible name is empty.',
      fix: 'Add aria-label="<action>" to every icon-only PatternFly button.',
      evidence: { name: { value: null, source: 'ax-tree', fromTree: true } },
      confidence: 'fail',
    });
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-kebab-expanded-state.ts` (static half of rule 4; the focus-into-menu half is a Task 4 probe):

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';
import { SEL } from './selectors.js';

export async function checkKebabExpandedState(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const toggles = await page.queryAll(SEL.menuToggle);
  const drafts: Draft[] = [];
  for (const toggle of toggles) {
    const node = await page.axAt(toggle.selector);
    if (!node) continue;
    const hasExpanded = 'expanded' in (node.states ?? {});
    const hasHaspopup = 'haspopup' in (node.states ?? {});
    if (!hasExpanded || !hasHaspopup) {
      drafts.push({
        rule: 'pf-kebab-expanded-state',
        layer: 'pf',
        severity: 'serious',
        evidenceClass: 'deterministic',
        screenId: screen.id,
        elementPath: toggle.selector,
        elementName: node.name,
        role: node.role,
        whatUserExperiences: 'Screen-reader users cannot tell whether this menu is open or closed.',
        why: 'The ARIA menu button pattern requires aria-expanded and aria-haspopup on the toggle so AT announces the current state.',
        fix: 'Ensure the PatternFly <MenuToggle> renders aria-expanded={isOpen} and aria-haspopup="menu".',
        evidence: { state: { expanded: { value: hasExpanded, source: 'ax-tree', fromTree: true } } },
        confidence: 'fail',
      });
    }
  }
  return drafts;
}
```

- [ ] Create `pf-table-header-assoc.ts`, `pf-row-action-name-unique.ts`, and
  `pf-toolbar-labeled-when-repeated.ts` with the same shapes as the tests above
  (unchanged logic from the previous revision, `SEL.*` selectors, real AX names via
  `axAt`). Row-action severity `serious`; toolbar severity `moderate`; toolbar rule
  fires only when two or more toolbars render and one lacks an accessible name.

- [ ] Create `src/providers/rulepack/index.ts` (SEQUENTIAL; static rules only until Task 4 appends the probes):

```ts
import type { Provider, ProviderContext, Draft } from '../../contracts/index.js';
import { checkToastLiveRegion } from './pf-toast-live-region.js';
import { checkIconButtonName } from './pf-icon-button-name.js';
import { checkKebabExpandedState } from './pf-kebab-expanded-state.js';
import { checkTableHeaderAssoc } from './pf-table-header-assoc.js';
import { checkRowActionNameUnique } from './pf-row-action-name-unique.js';
import { checkToolbarLabeledWhenRepeated } from './pf-toolbar-labeled-when-repeated.js';

type Check = (ctx: ProviderContext) => Promise<Draft[]>;

// Order is load-bearing: static reads first, interaction probes (Task 4) last,
// because probes click widgets open and mutate page state.
const STATIC_CHECKS: Check[] = [
  checkToastLiveRegion,
  checkIconButtonName,
  checkKebabExpandedState,
  checkTableHeaderAssoc,
  checkRowActionNameUnique,
  checkToolbarLabeledWhenRepeated,
];

export function makeRulepackProvider(extraChecks: Check[] = []): Provider {
  return {
    id: 'pf-rulepack',
    layer: 'pf',
    capabilities: ['live'],
    async run(ctx: ProviderContext): Promise<Draft[]> {
      const drafts: Draft[] = [];
      for (const check of [...STATIC_CHECKS, ...extraChecks]) {
        drafts.push(...(await check(ctx)));
      }
      return drafts;
    },
  };
}
```

- [ ] Run `npx vitest run test/providers/rulepack.test.ts`: expect PASS.
- [ ] Commit: `feat: add PatternFly rulepack static rules (PF6 selectors, AX-tree names)`

---

## Task 4: PatternFly rulepack: interaction probes (modal, dialog, menu)

**Files:**
- Create: `src/providers/rulepack/probes.ts`
- Modify: `src/providers/rulepack/index.ts`
- Create: `test/providers/rulepack-probes.test.ts`

`pf-modal-focus-return` and `pf-focus-into-dialog` only exist through interaction:
open the dialog, check focus went in, press Escape, check focus came back to the
exact trigger. The probes use `click()` plus the honest predicates
`activeElementWithin` / `activeElementIs`. Comparing a CSS selector string against a
DOM path string is forbidden: those are different string spaces and never match.

A trigger that declares `aria-haspopup="dialog"` but opens nothing is `unverified`
(the probe could not establish the fact), never a silent skip.

- [ ] Write the failing test:

```ts
// test/providers/rulepack-probes.test.ts
import { describe, it, expect } from 'vitest';
import { probeDialogs, probeMenus } from '../../src/providers/rulepack/probes.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import { testConfig } from '../helpers.js';
import type { ElementRef, ProviderContext } from '../../src/contracts/index.js';

const screen = { id: 'clusters', url: 'http://localhost/clusters' };

function dialogPage(opts: { opens: boolean; focusIn: boolean; focusBack: boolean }) {
  let open = false;
  return makeFakePage({
    queryAll: async (sel: string): Promise<ElementRef[]> => {
      if (sel.includes('aria-haspopup="dialog"')) return [{ selector: 'button:nth-child(1)' }];
      if (sel.includes('role="dialog"')) return open ? [{ selector: 'div:nth-child(9)' }] : [];
      return [];
    },
    click: async () => { open = opts.opens; },
    press: async (key: string) => { if (key === 'Escape') open = false; },
    activeElementWithin: async () => open && opts.focusIn,
    activeElementIs: async () => !open && opts.focusBack,
    axAt: async () => ({ name: 'View details', role: 'button', states: {} }),
  });
}

describe('probeDialogs', () => {
  it('passes a correct dialog silently', async () => {
    const ctx: ProviderContext = { page: dialogPage({ opens: true, focusIn: true, focusBack: true }), screen, config: testConfig() };
    expect(await probeDialogs(ctx)).toHaveLength(0);
  });

  it('fails pf-focus-into-dialog when focus does not enter the open dialog', async () => {
    const ctx: ProviderContext = { page: dialogPage({ opens: true, focusIn: false, focusBack: true }), screen, config: testConfig() };
    const drafts = await probeDialogs(ctx);
    expect(drafts.some((d) => d.rule === 'pf-focus-into-dialog' && d.confidence === 'fail')).toBe(true);
  });

  it('fails pf-modal-focus-return when Escape does not return focus to the trigger', async () => {
    const ctx: ProviderContext = { page: dialogPage({ opens: true, focusIn: true, focusBack: false }), screen, config: testConfig() };
    const drafts = await probeDialogs(ctx);
    expect(drafts.some((d) => d.rule === 'pf-modal-focus-return' && d.confidence === 'fail')).toBe(true);
  });

  it('emits unverified when a declared dialog trigger opens nothing', async () => {
    const ctx: ProviderContext = { page: dialogPage({ opens: false, focusIn: false, focusBack: false }), screen, config: testConfig() };
    const drafts = await probeDialogs(ctx);
    expect(drafts).toHaveLength(1);
    expect(drafts[0]!.confidence).toBe('unverified');
  });
});

describe('probeMenus', () => {
  it('fails pf-kebab-expanded-state focus half when focus does not enter the menu', async () => {
    let open = false;
    const page = makeFakePage({
      queryAll: async (sel: string): Promise<ElementRef[]> => {
        if (sel.includes('aria-haspopup="menu"')) return [{ selector: 'button:nth-child(4)' }];
        if (sel.includes('role="menu"')) return open ? [{ selector: 'ul:nth-child(5)' }] : [];
        return [];
      },
      click: async () => { open = true; },
      press: async (key: string) => { if (key === 'Escape') open = false; },
      activeElementWithin: async () => false,
      axAt: async () => ({ name: 'Actions', role: 'button', states: { expanded: false, haspopup: 'menu' } }),
    });
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    const drafts = await probeMenus(ctx);
    expect(drafts.some((d) => d.rule === 'pf-kebab-expanded-state' && d.confidence === 'fail')).toBe(true);
  });
});
```

- [ ] Run: expect FAIL.

- [ ] Create `src/providers/rulepack/probes.ts`:

```ts
import type { AxNode, Draft, ProviderContext } from '../../contracts/index.js';
import { SEL } from './selectors.js';

function draft(
  rule: string, screenId: string, elementPath: string, node: AxNode | null,
  confidence: 'fail' | 'unverified',
  whatUserExperiences: string, why: string, fix: string,
): Draft {
  return {
    rule, layer: 'pf', severity: 'serious', evidenceClass: 'deterministic',
    screenId, elementPath, elementName: node?.name ?? null, role: node?.role ?? null,
    whatUserExperiences, why, fix, evidence: {}, confidence,
  };
}

/** Rule 3 (focus into dialog) + rule 2 (focus returns to trigger on Escape). */
export async function probeDialogs(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const drafts: Draft[] = [];
  for (const trigger of await page.queryAll(SEL.dialogTrigger)) {
    const node = await page.axAt(trigger.selector);
    try {
      await page.click(trigger.selector);
      const dialogs = await page.queryAll(SEL.dialog);
      if (dialogs.length === 0) {
        drafts.push(draft('pf-focus-into-dialog', screen.id, trigger.selector, node, 'unverified',
          'Could not verify dialog focus behavior.',
          'The trigger declares aria-haspopup="dialog" but no dialog appeared when activated; the probe could not establish the fact.',
          'Confirm the trigger opens a dialog, or remove aria-haspopup="dialog".'));
        continue;
      }
      const dialogSel = dialogs[0]!.selector;
      if (!(await page.activeElementWithin(dialogSel))) {
        drafts.push(draft('pf-focus-into-dialog', screen.id, dialogSel, node, 'fail',
          'Focus does not move into the dialog when it opens; keyboard users remain behind the backdrop.',
          'The ARIA dialog pattern requires focus to move to the dialog or its first focusable element on open.',
          'Move focus to the PatternFly <Modal> initial focus target on render.'));
      }
      await page.press('Escape');
      if (!(await page.activeElementIs(trigger.selector))) {
        drafts.push(draft('pf-modal-focus-return', screen.id, trigger.selector, node, 'fail',
          'After closing the dialog, focus is lost instead of returning to the button that opened it.',
          'WCAG 2.4.3: focus must return to the triggering element on dialog close, or screen-reader users lose their place.',
          "Call .focus() on the trigger in the modal's onClose handler."));
      }
    } catch (err) {
      drafts.push(draft('pf-modal-focus-return', screen.id, trigger.selector, node, 'unverified',
        'Could not verify dialog focus behavior.',
        `The dialog probe failed: ${(err as Error).message}`,
        'Verify the dialog opens and closes cleanly, then re-run.'));
    }
  }
  return drafts;
}

/** Rule 4, interactive half: activating a menu toggle must move focus into the menu. */
export async function probeMenus(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const drafts: Draft[] = [];
  for (const toggle of await page.queryAll(SEL.menuToggle)) {
    const node = await page.axAt(toggle.selector);
    try {
      await page.click(toggle.selector);
      const menus = await page.queryAll(SEL.menu);
      if (menus.length > 0 && !(await page.activeElementWithin(menus[0]!.selector))) {
        drafts.push(draft('pf-kebab-expanded-state', screen.id, toggle.selector, node, 'fail',
          'Opening this menu leaves keyboard focus behind; users cannot reach the menu items.',
          'The ARIA menu button pattern moves focus into the menu on open.',
          'Focus the first menu item when the PatternFly menu opens.'));
      }
      await page.press('Escape');
    } catch (err) {
      drafts.push(draft('pf-kebab-expanded-state', screen.id, toggle.selector, node, 'unverified',
        'Could not verify menu focus behavior.',
        `The menu probe failed: ${(err as Error).message}`,
        'Verify the menu opens and closes cleanly, then re-run.'));
    }
  }
  return drafts;
}
```

- [ ] Modify `src/providers/rulepack/index.ts`: import `probeDialogs` and `probeMenus`
  and append them AFTER the static checks in the sequential list (probes mutate state,
  so they run last).

- [ ] Run `npx vitest run test/providers/rulepack-probes.test.ts test/providers/rulepack.test.ts`: expect PASS.
- [ ] Commit: `feat: add rulepack interaction probes for dialog and menu focus rules`

---

## Task 5: Keyboard walk provider and the StepRunner seam

**Files:**
- Create: `src/providers/keyboard-walk/steps.ts`
- Create: `src/providers/keyboard-walk/index.ts`
- Create: `test/providers/keyboard-walk.test.ts`

The StepRunner is the frozen seam Phase 4 reuses. Step kinds: `tab`, `press-escape`,
`activate`, `click` (with `selector`). An unknown step kind THROWS (silent no-ops are
silent passes; the voicing provider catches and emits `unverified`). After every step
the runner drains the live-region buffer and appends each drained string as an
`AnnouncementToken` with `kind: 'live'`. This is how toast announcements enter the
transcript; without it, "toast announced" obligations could never be satisfied.

- [ ] Write failing tests:

```ts
// test/providers/keyboard-walk.test.ts
import { describe, it, expect } from 'vitest';
import { makeKeyboardWalkProvider, makeStepRunner } from '../../src/providers/keyboard-walk/index.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import { testConfig } from '../helpers.js';
import type { AxNode, ProviderContext, Step } from '../../src/contracts/index.js';

const screen = { id: 'home', url: 'http://localhost/' };

function walkPage(sequence: Array<AxNode | null>, paths: string[], live: string[][] = []) {
  let idx = -1;
  return makeFakePage({
    tab: async () => { idx++; },
    press: async () => { idx++; },
    activeNode: async () => sequence[Math.min(Math.max(idx, 0), sequence.length - 1)] ?? null,
    activePath: async () => paths[Math.min(Math.max(idx, 0), paths.length - 1)] ?? '',
    drainAnnouncements: async () => live[Math.min(Math.max(idx, 0), live.length - 1)] ?? [],
  });
}

describe('StepRunner', () => {
  it('records one TranscriptStop per step with name and role tokens', async () => {
    const page = walkPage([{ name: 'Submit', role: 'button', states: {} }], ['button:nth-child(1)']);
    const stops = await makeStepRunner().run(page, [{ do: 'tab' }]);
    expect(stops).toHaveLength(1);
    expect(stops[0]!.announcement.some((t) => t.kind === 'name' && t.text === 'Submit')).toBe(true);
    expect(stops[0]!.announcement.some((t) => t.kind === 'role' && t.text === 'button')).toBe(true);
  });

  it('appends drained live-region text as kind "live" tokens', async () => {
    const page = walkPage(
      [{ name: 'Delete', role: 'button', states: {} }],
      ['button:nth-child(1)'],
      [['Cluster deleted successfully']],
    );
    const stops = await makeStepRunner().run(page, [{ do: 'activate' }]);
    const live = stops[0]!.announcement.filter((t) => t.kind === 'live');
    expect(live).toHaveLength(1);
    expect(live[0]!.text).toBe('Cluster deleted successfully');
    expect(live[0]!.fromTree).toBe(false);
  });

  it('throws on an unknown step kind (never a silent no-op)', async () => {
    const page = walkPage([null], ['']);
    await expect(makeStepRunner().run(page, [{ do: 'wave-hands' } as Step])).rejects.toThrow('unknown step kind');
  });
});

describe('keyboard walk provider', () => {
  it('emits an unverified draft when focus cannot be confirmed', async () => {
    const page = walkPage([null], ['div:nth-child(1)']);
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    const drafts = await makeKeyboardWalkProvider({ tabCap: 3 }).run(ctx);
    expect(drafts.some((d) => d.rule === 'keyboard-walk-unconfirmed-focus' && d.confidence === 'unverified')).toBe(true);
  });

  it('flags an unnamed interactive element (link role from the AX tree)', async () => {
    const page = walkPage(
      [{ name: null, role: 'link', states: {} }],
      ['a:nth-child(2)'],
    );
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    const drafts = await makeKeyboardWalkProvider({ tabCap: 1 }).run(ctx);
    expect(drafts.some((d) => d.rule === 'keyboard-walk-unnamed-interactive')).toBe(true);
  });

  it('stops at tabCap and on cycles instead of hanging', async () => {
    const nodes = Array.from({ length: 50 }, () => ({ name: 'x', role: 'button', states: {} }));
    const page = walkPage(nodes, nodes.map(() => 'button:nth-child(1)')); // same path = immediate cycle
    const ctx: ProviderContext = { page, screen, config: testConfig() };
    const drafts = await makeKeyboardWalkProvider({ tabCap: 5 }).run(ctx);
    expect(drafts.length).toBeLessThanOrEqual(5);
  });
});
```

- [ ] Run: expect FAIL.

- [ ] Create `src/providers/keyboard-walk/steps.ts`:

```ts
import type { AnnouncementToken, AxNode, Page, Step, StepRunner, TranscriptStop } from '../../contracts/index.js';

function nodeTokens(node: AxNode): AnnouncementToken[] {
  const tokens: AnnouncementToken[] = [];
  if (node.name !== null) tokens.push({ kind: 'name', text: node.name, fromTree: true, source: 'ax-tree' });
  if (node.role !== null) tokens.push({ kind: 'role', text: node.role, fromTree: true, source: 'ax-tree' });
  for (const [key, value] of Object.entries(node.states ?? {})) {
    if (value === true) tokens.push({ kind: 'state', text: key, fromTree: true, source: 'ax-tree' });
  }
  return tokens;
}

async function executeStep(page: Page, step: Step): Promise<void> {
  switch (step.do) {
    case 'tab': await page.tab(); return;
    case 'press-escape': await page.press('Escape'); return;
    case 'activate': await page.press('Enter'); return;
    case 'click': await page.click(String(step['selector'] ?? '')); return;
    default: throw new Error(`unknown step kind: ${step.do}`);
  }
}

export function makeConcreteStepRunner(): StepRunner {
  return {
    async run(page: Page, steps: Step[]): Promise<TranscriptStop[]> {
      await page.armAnnouncementCapture();
      const stops: TranscriptStop[] = [];
      for (let i = 0; i < steps.length; i++) {
        await executeStep(page, steps[i]!);
        const elementPath = await page.activePath();
        const node = await page.activeNode();
        const announcement = node ? nodeTokens(node) : [];
        // Live-region text that arrived during this step's window becomes 'live' tokens.
        for (const text of await page.drainAnnouncements()) {
          announcement.push({ kind: 'live', text, fromTree: false, source: 'attribute' });
        }
        stops.push({ index: i, elementPath, announcement });
      }
      return stops;
    },
  };
}
```

- [ ] Create `src/providers/keyboard-walk/index.ts`:

```ts
import type { Draft, Provider, ProviderContext, StepRunner } from '../../contracts/index.js';
import { makeConcreteStepRunner } from './steps.js';

export function makeStepRunner(): StepRunner {
  return makeConcreteStepRunner();
}

// now() is injected so unit tests stay deterministic; real wiring passes Date.now.
interface WalkConfig { tabCap?: number; wallClockMs?: number; now?: () => number; }

const INTERACTIVE_ROLES = new Set([
  'button', 'link', 'menuitem', 'checkbox', 'radio', 'textbox', 'combobox', 'tab', 'switch', 'option',
]);

export function makeKeyboardWalkProvider(cfg: WalkConfig = {}): Provider {
  const tabCap = cfg.tabCap ?? 200;

  return {
    id: 'keyboard-walk',
    layer: 'walk',
    capabilities: ['live'],
    async run(ctx: ProviderContext): Promise<Draft[]> {
      const { page, screen } = ctx;
      const drafts: Draft[] = [];
      const seen = new Set<string>();
      const deadline = cfg.now && cfg.wallClockMs ? cfg.now() + cfg.wallClockMs : null;

      await page.focusBody();
      for (let i = 0; i < tabCap; i++) {
        if (deadline !== null && cfg.now!() > deadline) break;
        await page.tab();
        const elementPath = await page.activePath();
        const node = await page.activeNode();

        if (!node) {
          drafts.push({
            rule: 'keyboard-walk-unconfirmed-focus', layer: 'walk', severity: 'serious',
            evidenceClass: 'deterministic', screenId: screen.id,
            elementPath: elementPath || `tab-stop-${i}`, elementName: null, role: null,
            whatUserExperiences: 'A focusable element exists but its accessible role and name cannot be determined.',
            why: 'Keyboard users reach this element but AT cannot announce what it is.',
            fix: 'Give the element a semantic HTML role or an explicit ARIA role, plus an accessible name.',
            evidence: {}, confidence: 'unverified',
          });
          continue;
        }

        if (seen.has(elementPath)) break; // cycle: the tab order wrapped
        seen.add(elementPath);

        if (INTERACTIVE_ROLES.has(node.role ?? '') && !node.name) {
          drafts.push({
            rule: 'keyboard-walk-unnamed-interactive', layer: 'walk', severity: 'serious',
            evidenceClass: 'deterministic', screenId: screen.id,
            elementPath, elementName: null, role: node.role,
            whatUserExperiences: 'An interactive element has no accessible name; screen-reader users hear only the role.',
            why: `A ${node.role} reached via Tab has no accessible name in the AX tree.`,
            fix: 'Add aria-label or visible text to the element.',
            evidence: { role: { value: node.role, source: 'ax-tree', fromTree: true } },
            confidence: 'fail',
          });
        }
      }
      return drafts;
    },
  };
}
```

- [ ] Run: expect PASS.
- [ ] Commit: `feat: add keyboard walk provider and StepRunner with live-token capture`

---

## Task 6: CheckRunner: transcript pass, providers, gaps into ScreenScan

**Files:**
- Create: `src/providers/check-runner.ts`
- Create: `test/providers/check-runner.test.ts`

The `CheckRunner` satisfies `Deps.checkRunner`. Per screen it: opens the page, arms
live-region capture, records a bounded transcript pass into `stops` (this feeds the
announcement views, the docs generators, and the demo), runs the providers, and
returns everything INCLUDING the gaps. `scan()` never throws: a screen that fails to
load returns a `ScreenScan` whose `gaps` say why, and the gate blocks on it.

- [ ] Write the failing test:

```ts
// test/providers/check-runner.test.ts
import { describe, it, expect, vi } from 'vitest';
import { makeCheckRunner } from '../../src/providers/check-runner.js';
import { makeStepRunner } from '../../src/providers/keyboard-walk/index.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import { testConfig } from '../helpers.js';
import type { BrowserDriver, Draft, Provider } from '../../src/contracts/index.js';

const draft: Draft = {
  rule: 'test-rule', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
  screenId: 'home', elementPath: 'button', elementName: null, role: 'button',
  whatUserExperiences: 't', why: 't', fix: 't', evidence: {}, confidence: 'fail',
};

function driverFor(page = makeFakePage()): BrowserDriver {
  return { open: vi.fn().mockResolvedValue(page), close: async () => {} };
}

describe('CheckRunner', () => {
  it('assembles drafts, stops, and gaps into one ScreenScan', async () => {
    const page = makeFakePage({
      activeNode: async () => ({ name: 'Go', role: 'button', states: {} }),
      activePath: async () => 'button:nth-child(1)',
    });
    const ok: Provider = { id: 'ok', layer: 'axe', capabilities: ['live'], run: vi.fn().mockResolvedValue([draft]) };
    const denied: Provider = { id: 'needs-net', layer: 'pf', capabilities: ['network'], run: vi.fn() };
    const runner = makeCheckRunner({
      browser: driverFor(page), providers: [ok, denied], config: testConfig(),
      allowedCapabilities: ['live'], stepRunner: makeStepRunner(), transcriptTabCap: 2,
    });
    const scan = await runner.scan({ id: 'home', url: 'http://localhost/' });
    expect(scan.screenId).toBe('home');
    expect(scan.drafts).toHaveLength(1);
    expect(scan.stops.length).toBeGreaterThan(0);
    expect(scan.gaps).toHaveLength(1);
    expect(scan.gaps[0]).toMatchObject({ ref: 'provider:needs-net', state: 'capability-denied' });
  });

  it('closes the page even when a provider throws, and reports the failure as a gap', async () => {
    const page = makeFakePage();
    const closeSpy = vi.spyOn(page, 'close');
    const thrower: Provider = { id: 'boom', layer: 'axe', capabilities: [], run: vi.fn().mockRejectedValue(new Error('boom')) };
    const runner = makeCheckRunner({
      browser: driverFor(page), providers: [thrower], config: testConfig(),
      allowedCapabilities: [], stepRunner: makeStepRunner(), transcriptTabCap: 1,
    });
    const scan = await runner.scan({ id: 'home', url: 'http://localhost/' });
    expect(scan.gaps.some((g) => g.reason.includes('boom'))).toBe(true);
    expect(closeSpy).toHaveBeenCalled();
  });

  it('turns a screen that fails to open into a not-covered gap, never a throw', async () => {
    const browser: BrowserDriver = { open: vi.fn().mockRejectedValue(new Error('net down')), close: async () => {} };
    const runner = makeCheckRunner({
      browser, providers: [], config: testConfig(),
      allowedCapabilities: [], stepRunner: makeStepRunner(),
    });
    const scan = await runner.scan({ id: 'home', url: 'http://localhost/' });
    expect(scan.drafts).toHaveLength(0);
    expect(scan.gaps).toHaveLength(1);
    expect(scan.gaps[0]).toMatchObject({ ref: 'http://localhost/', state: 'not-covered' });
  });
});
```

- [ ] Run: expect FAIL.

- [ ] Create `src/providers/check-runner.ts`:

```ts
import type {
  BrowserDriver, Capability, CheckRunner, Provider, ScreenScan, Step, StepRunner, UsablConfig,
} from '../contracts/index.js';
import { runProviders } from './index.js';

interface CheckRunnerDeps {
  browser: BrowserDriver;
  providers: Provider[];
  config: UsablConfig;
  allowedCapabilities: Capability[];
  stepRunner: StepRunner;
  transcriptTabCap?: number;
}

const tabSteps = (n: number): Step[] => Array.from({ length: n }, () => ({ do: 'tab' }));

/** Trim the transcript at the first revisited element path (the tab order wrapped). */
function trimCycle(stops: ScreenScan['stops']): ScreenScan['stops'] {
  const seen = new Set<string>();
  const out: ScreenScan['stops'] = [];
  for (const stop of stops) {
    if (seen.has(stop.elementPath)) break;
    seen.add(stop.elementPath);
    out.push(stop);
  }
  return out;
}

export function makeCheckRunner(deps: CheckRunnerDeps): CheckRunner {
  const { browser, providers, config, allowedCapabilities, stepRunner } = deps;
  const cap = deps.transcriptTabCap ?? 50;

  return {
    // NEVER throws: a screen that cannot be scanned reports gaps and the gate blocks.
    async scan(screen: { id: string; url: string }): Promise<ScreenScan> {
      let page;
      try {
        page = await browser.open(screen.url);
      } catch (err) {
        return {
          screenId: screen.id, url: screen.url, stops: [], drafts: [],
          gaps: [{ ref: screen.url, state: 'not-covered', reason: `screen failed to open: ${(err as Error).message}` }],
        };
      }
      try {
        await page.gotoReady();
        await page.armAnnouncementCapture();
        // 1. Transcript pass first (probes mutate state later): bounded tab walk.
        const stops = trimCycle(await stepRunner.run(page, tabSteps(cap)));
        await page.focusBody();
        // 2. Providers, sequentially; capability skips and errors become gaps.
        const { drafts, gaps } = await runProviders(providers, { page, screen, config }, allowedCapabilities);
        return { screenId: screen.id, url: screen.url, stops, drafts, gaps };
      } catch (err) {
        return {
          screenId: screen.id, url: screen.url, stops: [], drafts: [],
          gaps: [{ ref: screen.url, state: 'not-covered', reason: `scan failed: ${(err as Error).message}` }],
        };
      } finally {
        await page.close();
      }
    },
  };
}
```

- [ ] Run the new test, then the full unit suite: `npx vitest run`.
- [ ] Commit: `feat: add CheckRunner with transcript pass and gap collection`

---

## Task 7: Real BrowserDriver: CDP accessibility tree, live-region capture, axe bridge

**Files:**
- Create: `src/deps/real.ts`
- Create: `src/integration-smoke.ts`

> **Integration, not unit tests.** Requires a real browser. Run manually or with
> `USABL_INTEGRATION=1`. Never in the default CI unit path.

- [ ] Install the runtime dependencies:

```bash
npm install playwright axe-core @axe-core/playwright
npx playwright install chromium
```

- [ ] Create `src/deps/real.ts`. Requirements, in order of importance:

1. **AX tree over CDP, never DOM approximation.** `activeNode()` and `axAt()` resolve
   the element to a CDP `backendNodeId` and read `Accessibility.getPartialAXTree`
   (`fetchRelatives: false`): `name.value`, `role.value`, and `properties` (expanded,
   haspopup, checked, disabled, ...) into `AxNode.states`. An ignored node returns
   `null`. This is the product's credibility surface: the transcript must be what AT
   gets, including names computed from `aria-labelledby` and label association.
2. **One warm browser.** The driver lazily launches one Chromium; `open(url)` creates
   a fresh context + page (with `storageState` when provided for the measurement pass);
   `Page.close()` closes the context; `BrowserDriver.close()` disposes the browser.
3. **Live-region capture.** `addInitScript` installs a MutationObserver over
   `[aria-live], [role=status], [role=alert], [role=log]` containers that appends each
   changed container's text to a page-global buffer. `drainAnnouncements()` evaluates
   a read-and-clear of that buffer; `armAnnouncementCapture()` is a no-op (the init
   script arms at document start, BEFORE any update can fire, which is exactly the
   temporal property the toast rule needs).
4. **Stable unique selectors.** `queryAll(selector)` computes one stable CSS path per
   match inside the page (`#id` short-circuit, else a `tag:nth-child(i)` chain up to
   the nearest id or the root). `activePath()` uses the same path function, so paths
   from different calls are comparable. `nth-of-type` appended to a compound selector
   is forbidden (wrong semantics, invalid CSS).
5. **Honest predicates in the page.** `activeElementIs(sel)` evaluates
   `document.querySelector(sel) === document.activeElement`;
   `activeElementWithin(sel)` evaluates `.contains(document.activeElement)`.
   Never compare selector strings to path strings.
6. **axe bridge.** Attach `runAxe()` using `new AxeBuilder({ page }).analyze()` from
   `@axe-core/playwright`, returning `{ violations, incomplete }`. No hand-rolled
   `eval` injection.
7. `gotoReady()` = bounded `networkidle` wait (15 s cap). `click(sel)` = Playwright
   click with a short timeout. Zoom, viewport, reduced motion, computed style, and
   screenshot as before.

Skeleton (fill in the mechanical parts; every method above is required):

```ts
import { chromium, type Browser, type CDPSession, type Page as PwPage } from 'playwright';
import { AxeBuilder } from '@axe-core/playwright';
import type { AxNode, BrowserDriver, Page } from '../contracts/index.js';

const LIVE_OBSERVER = `(() => {
  const LIVE = '[aria-live],[role="status"],[role="alert"],[role="log"]';
  const buf = [];
  window.__usablDrain = () => buf.splice(0, buf.length);
  const push = (t) => { const x = (t ?? '').trim(); if (x) buf.push(x); };
  const obs = new MutationObserver((muts) => {
    for (const m of muts) {
      const el = m.target instanceof Element ? m.target : m.target.parentElement;
      const host = el && el.closest ? el.closest(LIVE) : null;
      if (host) push(host.textContent);
    }
  });
  const start = () => obs.observe(document.documentElement, { subtree: true, childList: true, characterData: true });
  if (document.readyState === 'loading') document.addEventListener('DOMContentLoaded', start); else start();
})();`;

async function axFromBackendId(cdp: CDPSession, backendNodeId: number): Promise<AxNode | null> {
  const { nodes } = await cdp.send('Accessibility.getPartialAXTree', { backendNodeId, fetchRelatives: false });
  const n = nodes?.[0];
  if (!n || n.ignored) return null;
  const states: Record<string, unknown> = {};
  for (const p of n.properties ?? []) states[p.name] = p.value?.value;
  return { name: (n.name?.value as string) ?? null, role: (n.role?.value as string) ?? null, states };
}

export function makeRealBrowserDriver(opts: { storageStatePath?: string } = {}): BrowserDriver {
  let browser: Browser | null = null;
  return {
    async open(url: string): Promise<Page> {
      browser ??= await chromium.launch({ headless: true });
      const ctx = await browser.newContext(opts.storageStatePath ? { storageState: opts.storageStatePath } : {});
      const pw = await ctx.newPage();
      await pw.addInitScript(LIVE_OBSERVER);
      const cdp = await ctx.newCDPSession(pw);
      await cdp.send('Accessibility.enable');
      await pw.goto(url, { waitUntil: 'networkidle', timeout: 15_000 });
      // ... wrap pw + cdp into the Page interface per requirements 1-7 above,
      //     with close() closing the CONTEXT (not the browser).
      return wrapPage(pw, ctx, cdp);
    },
    async close() { await browser?.close(); browser = null; },
  };
}
```

- [ ] Create `src/integration-smoke.ts` (run manually with a fixture dev server up):

```ts
// USABL_INTEGRATION=1 npx tsx src/integration-smoke.ts http://localhost:5173/clusters
import { makeRealBrowserDriver } from './deps/real.js';
import { makeAxeProvider } from './providers/axe/index.js';
import { makeRulepackProvider } from './providers/rulepack/index.js';
import { makeKeyboardWalkProvider, makeStepRunner } from './providers/keyboard-walk/index.js';
import { makeCheckRunner } from './providers/check-runner.js';
import type { UsablConfig } from './contracts/index.js';

const url = process.argv[2] ?? 'http://localhost:5173/';
const config: UsablConfig = {
  appBaseUrl: url, uiFileGlobs: [], discovery: { routerFile: '', wideBlastGlobs: [] },
  surfaces: [], guardedPaths: [],
};
const browser = makeRealBrowserDriver();
const runner = makeCheckRunner({
  browser,
  providers: [makeAxeProvider(), makeRulepackProvider(), makeKeyboardWalkProvider({ now: Date.now, wallClockMs: 15_000 })],
  config, allowedCapabilities: ['live'], stepRunner: makeStepRunner(),
});
const scan = await runner.scan({ id: 'smoke', url });
console.log(`stops: ${scan.stops.length}  drafts: ${scan.drafts.length}  gaps: ${scan.gaps.length}`);
scan.drafts.forEach((d) => console.log(`  [${d.layer}] ${d.rule} (${d.confidence}) @ ${d.elementPath}`));
scan.gaps.forEach((g) => console.log(`  GAP ${g.state}: ${g.reason}`));
await browser.close();
```

**Exit criterion:** against a live PatternFly page, the smoke run produces real
`Draft[]` with AX-tree names (verify one `aria-labelledby` case resolves), a non-empty
transcript with at least one `live` token after an action that fires a toast, and
`gaps` populated when a capability is denied.

- [ ] Commit: `feat: add CDP-based real BrowserDriver with live-region capture and axe bridge`

---

## Self-Review

**Layer coverage:**
- axe provider: violations → `fail`, incomplete → `unverified` (never dropped), `layer: 'axe'`.
- Rulepack: six static rules honestly static; the two focus rules plus the menu focus half are interaction probes using `click` + `activeElementIs`/`activeElementWithin`. No rule compares selector strings to path strings. All selectors PF6/ARIA-first.
- Keyboard walk: AX-tree roles (so links count), `unverified` on unconfirmed focus, cycle break on first revisit, injectable clock for the wall-clock ceiling.
- StepRunner: frozen seam; step kinds tab/press-escape/activate/click; unknown kind throws; drains live-region text into `kind: 'live'` tokens after every step.

**Gap flow:** capability denials and provider/screen failures land in `ScreenScan.gaps` → `run()` merges into `coverage.gaps` → gate forces `not_covered`. Verified by tests in Tasks 1 and 6 plus the Phase 1 gate test.

**Duplicate control:** the unnamed-control overlap across axe/pf/walk is resolved by the gate's equivalence suppression (Phase 1 Task 8), not by pretending the layers do not overlap.

**Known static limits, disclosed:** the toast rule checks containment only; the temporal live-region property is proven by live tokens during interaction contracts (Phase 4). A dialog trigger that opens nothing is `unverified`.

**Fakes:** every unit test uses `makeFakePage`/`makeFakeDeps` overrides. No hand-rolled `Page` literals, no `as any`, and the only cast is the documented `AxePage` attachment.

**Type consistency:** `Draft`, `ScreenScan` (with `gaps`), `TranscriptStop`, `AnnouncementToken` (with `'live'`), `Provider`, `ProviderContext`, `Capability`, `CoverageGap`, `CheckRunner`, `BrowserDriver` (with `close`), `Page` (with the five new methods), `StepRunner`, `Step`: all imported verbatim from `src/contracts/index.ts`.

**Verdict authority:** no provider decides a verdict. Drafts and gaps are data; the gate decides.
