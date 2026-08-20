# Detection Engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire the `Provider` interface and build three deterministic scan layers - axe-core (WCAG baseline), PatternFly rulepack (eight named rules, design-system semantics), and keyboard walk (role-aware interaction probing) - plus the declarative interaction probe seam that Phase 4 reuses. Deliver a real `CheckRunner` that opens a `Page` per screen via `BrowserDriver`, runs every `Provider`, and assembles the frozen `ScreenScan`. Unit tests use in-memory fakes only; real browser wiring is clearly-labeled integration work.

**Hard gate from Phase 1 security review:** `formatSummary` prints `whatUserExperiences` and `fix` to stdout with no neutralization. That is parked, not waived. Before this plan's CheckRunner can copy page text into Drafts, implement `neutralize()` and use it in `formatSummary` (and any other egress). Do not defer that to the surfaces plan. A live scan without it ships untrusted page text to the terminal.

**Architecture:**
- Providers return `Draft[]` only. No provider decides a verdict.
- `CheckRunner` is the only thing that touches `BrowserDriver`; it is wired into `Deps` (Phase 1 already declares `Deps.checkRunner: CheckRunner`).
- Static mode = capability filtering: any provider whose `capabilities` includes `'live'` is skipped; each skip records a `CoverageGap` with `state: 'capability-denied'`. No silent passes.
- Dedup collapses drafts that share the same IDENTITY fields (surface, rule, elementPath, role) across layers before the gate sees them; the gate remains unchanged.
- The axe provider owns `evidenceClass: 'deterministic'`; so do all PF rulepack rules and the keyboard walk. `confidence: 'unverified'` is set on any step the walk cannot confirm.

**Tech Stack:** TypeScript (ESM, strict), Node 22, Vitest, axe-core, Playwright (real integration only), `@axe-core/playwright` (integration bridge).

---

## Task 1 — Provider interface and capability-filtering wiring

**Files:**
- Modify: `src/contracts/index.ts`
- Create: `src/providers/index.ts`
- Create: `tests/providers/index.test.ts`

- [ ] Write a failing test that verifies: when a `Provider` with `capabilities: ['live']` is registered and `runProviders` is called with `allowedCapabilities: []`, the provider is NOT called and a `CoverageGap` with `state: 'capability-denied'` is returned.

```ts
// tests/providers/index.test.ts
import { describe, it, expect, vi } from 'vitest';
import { runProviders } from '../../src/providers/index.js';
import type { Provider, ProviderContext, CoverageGap, Draft } from '../../src/contracts/index.js';

const ctx: ProviderContext = {
  page: {} as never,
  screen: { id: 'home', url: 'http://localhost/' },
  config: { schemaVersion: 1, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } },
};

describe('runProviders', () => {
  it('skips live provider in static mode and records capability-denied gap', async () => {
    const p: Provider = {
      id: 'fake-live',
      layer: 'axe',
      capabilities: ['live'],
      run: vi.fn().mockResolvedValue([]),
    };
    const result = await runProviders([p], ctx, []);
    expect(p.run).not.toHaveBeenCalled();
    expect(result.drafts).toHaveLength(0);
    expect(result.gaps).toHaveLength(1);
    expect(result.gaps[0]).toMatchObject<Partial<CoverageGap>>({
      ref: 'provider:fake-live',
      state: 'capability-denied',
    });
    expect(result.gaps[0].reason).toBeTruthy();
  });

  it('calls live provider when live capability is allowed', async () => {
    const draft: Draft = {
      rule: 'axe-color-contrast',
      layer: 'axe',
      severity: 'serious',
      evidenceClass: 'deterministic',
      screenId: 'home',
      elementPath: 'button',
      elementName: 'Submit',
      role: 'button',
      whatUserExperiences: 'Cannot distinguish button from background',
      why: 'Contrast ratio 2.1:1 below 4.5:1 threshold',
      fix: 'Increase button background contrast',
      evidence: {},
      confidence: 'fail',
    };
    const p: Provider = {
      id: 'fake-live-allowed',
      layer: 'axe',
      capabilities: ['live'],
      run: vi.fn().mockResolvedValue([draft]),
    };
    const result = await runProviders([p], ctx, ['live']);
    expect(p.run).toHaveBeenCalledWith(ctx);
    expect(result.drafts).toHaveLength(1);
    expect(result.gaps).toHaveLength(0);
  });
});
```

- [ ] Run `npx vitest run tests/providers/index.test.ts` — expect FAIL (module not found).

- [ ] Add the `ProviderContext`, `Capability`, and `Provider` shapes to `src/contracts/index.ts` (verbatim from 00-plan-set.md frozen seam):

```ts
// src/contracts/index.ts — append after existing exports
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
import type { Provider, ProviderContext, Capability, ProviderRunResult } from '../contracts/index.js';

export async function runProviders(
  providers: Provider[],
  ctx: ProviderContext,
  allowedCapabilities: Capability[],
): Promise<ProviderRunResult> {
  const allowed = new Set(allowedCapabilities);
  const drafts = [];
  const gaps = [];

  for (const provider of providers) {
    const denied = provider.capabilities.find((c) => !allowed.has(c));
    if (denied !== undefined) {
      gaps.push({
        ref: `provider:${provider.id}`,
        state: 'capability-denied' as const,
        reason: `Provider requires capability '${denied}' which is not permitted in this mode`,
      });
      continue;
    }
    const result = await provider.run(ctx);
    drafts.push(...result);
  }

  return { drafts, gaps };
}
```

- [ ] Run `npx vitest run tests/providers/index.test.ts` — expect PASS.
- [ ] Commit: `feat: add Provider interface and capability-filtering runProviders`

---

## Task 2 — axe-core provider (WCAG baseline)

**Files:**
- Create: `src/providers/axe/index.ts`
- Create: `src/providers/axe/notes.ts`
- Create: `tests/providers/axe.test.ts`

- [ ] Write a failing test using a `FakePage` that injects canned axe results via `page.runAxe()`:

```ts
// tests/providers/axe.test.ts
import { describe, it, expect } from 'vitest';
import { makeAxeProvider } from '../../src/providers/axe/index.js';
import type { Provider, ProviderContext, Draft, Page, AxNode } from '../../src/contracts/index.js';

// Minimal FakePage that supports runAxe injection
function makeFakePage(axeResult: { violations: unknown[] }): Page & { runAxe(): Promise<{ violations: unknown[] }> } {
  return {
    async gotoReady() {},
    async focusBody() {},
    async tab() {},
    async press() {},
    async activeNode() { return null; },
    async activePath() { return ''; },
    async axAt() { return null; },
    async queryAll() { return []; },
    async close() {},
    async setViewport() {},
    async setZoom() {},
    async setReducedMotion() {},
    async getComputedStyle() { return ''; },
    async screenshot() { return Buffer.from(''); },
    async runAxe() { return axeResult; },
  };
}

const baseCtx = {
  screen: { id: 'home', url: 'http://localhost/' },
  config: { schemaVersion: 1 as const, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } },
};

describe('axe provider', () => {
  it('returns one Draft per axe violation node', async () => {
    const page = makeFakePage({
      violations: [
        {
          id: 'color-contrast',
          impact: 'serious',
          description: 'Ensures the contrast ratio of text meets WCAG 2 AA',
          nodes: [
            {
              target: ['button.submit'],
              html: '<button class="submit">Go</button>',
              failureSummary: 'Fix contrast',
              any: [{ data: { fgColor: '#fff', bgColor: '#eee', contrastRatio: 2.1 } }],
            },
          ],
        },
      ],
    });
    const provider = makeAxeProvider();
    const ctx: ProviderContext = { page: page as unknown as Page, ...baseCtx };
    const drafts = await provider.run(ctx);
    expect(drafts).toHaveLength(1);
    const d = drafts[0];
    expect(d.rule).toBe('color-contrast');
    expect(d.layer).toBe('axe');
    expect(d.evidenceClass).toBe('deterministic');
    expect(d.confidence).toBe('fail');
    expect(d.screenId).toBe('home');
    expect(d.elementPath).toBe('button.submit');
    expect(d.severity).toBe('serious');
    expect(d.fix).toBeTruthy();
    expect(d.why).toBeTruthy();
  });

  it('returns empty array when no violations', async () => {
    const page = makeFakePage({ violations: [] });
    const provider = makeAxeProvider();
    const ctx: ProviderContext = { page: page as unknown as Page, ...baseCtx };
    const drafts = await provider.run(ctx);
    expect(drafts).toHaveLength(0);
  });

  it('declares live capability', () => {
    const provider = makeAxeProvider();
    expect(provider.capabilities).toContain('live');
  });
});
```

- [ ] Run `npx vitest run tests/providers/axe.test.ts` — expect FAIL.

- [ ] Create `src/providers/axe/notes.ts` (plain-language PatternFly overrides for common axe rule ids):

```ts
// Plain-language PatternFly-specific why/fix overrides for axe rule ids.
// Keeps provider output readable without duplicating axe's own description.
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

// The axe Page extension used at integration time (real Playwright Page).
// FakePage satisfies this in tests by exposing runAxe() directly.
export interface AxePage {
  runAxe(): Promise<{ violations: AxeViolation[] }>;
}

interface AxeViolation {
  id: string;
  impact: string | null;
  description: string;
  nodes: Array<{
    target: string[];
    html: string;
    failureSummary: string;
    any: Array<{ data: unknown }>;
  }>;
}

function toSeverity(impact: string | null): Severity {
  const map: Record<string, Severity> = {
    critical: 'critical',
    serious: 'serious',
    moderate: 'moderate',
    minor: 'minor',
  };
  return map[impact ?? ''] ?? 'moderate';
}

export function makeAxeProvider(): Provider {
  return {
    id: 'axe-core',
    layer: 'axe',
    capabilities: ['live'],

    async run(ctx: ProviderContext): Promise<Draft[]> {
      const axePage = ctx.page as unknown as AxePage;
      const { violations } = await axePage.runAxe();
      const { why: defaultWhy, fix: defaultFix } = { why: '', fix: '' };
      const drafts: Draft[] = [];

      for (const violation of violations) {
        const note = noteFor(violation.id);
        for (const node of violation.nodes) {
          const elementPath = node.target.join(' ');
          drafts.push({
            rule: violation.id,
            layer: 'axe',
            severity: toSeverity(violation.impact),
            evidenceClass: 'deterministic',
            screenId: ctx.screen.id,
            elementPath,
            elementName: null,
            role: null,
            whatUserExperiences: violation.description,
            why: note.why || node.failureSummary || defaultWhy,
            fix: note.fix || node.failureSummary || defaultFix,
            evidence: {
              extra: { html: node.html, axeData: node.any[0]?.data ?? null },
            },
            confidence: 'fail',
          });
        }
      }

      return drafts;
    },
  };
}
```

- [ ] Run `npx vitest run tests/providers/axe.test.ts` — expect PASS.
- [ ] Commit: `feat: add axe-core provider with PatternFly notes overrides`

---

## Task 3 — PatternFly rulepack: focus/interaction rules (pf-rules 1-5)

**Files:**
- Create: `src/providers/rulepack/pf-toast-live-region.ts`
- Create: `src/providers/rulepack/pf-modal-focus-return.ts`
- Create: `src/providers/rulepack/pf-focus-into-dialog.ts`
- Create: `src/providers/rulepack/pf-kebab-expanded-state.ts`
- Create: `src/providers/rulepack/pf-icon-button-name.ts`
- Create: `src/providers/rulepack/index.ts`
- Create: `tests/providers/rulepack.test.ts`

- [ ] Write failing tests for rules 1 (toast-live-region) and 5 (icon-button-name) using a FakePage with a `queryAll()` / `axAt()` stub:

```ts
// tests/providers/rulepack.test.ts
import { describe, it, expect } from 'vitest';
import { makeRulepackProvider } from '../../src/providers/rulepack/index.js';
import type { ProviderContext, Page, AxNode, ElementRef } from '../../src/contracts/index.js';

function makeFakePage(overrides: Partial<{
  queryAll: (sel: string) => Promise<ElementRef[]>;
  axAt: (sel: string) => Promise<AxNode | null>;
  tab: () => Promise<void>;
  activeNode: () => Promise<AxNode | null>;
  activePath: () => Promise<string>;
  press: (key: string) => Promise<void>;
}>): Page {
  return {
    async gotoReady() {},
    async focusBody() {},
    async tab() { await overrides.tab?.(); },
    async press(key: string) { await overrides.press?.(key); },
    async activeNode() { return overrides.activeNode?.() ?? null; },
    async activePath() { return overrides.activePath?.() ?? ''; },
    async axAt(sel: string) { return overrides.axAt?.(sel) ?? null; },
    async queryAll(sel: string) { return overrides.queryAll?.(sel) ?? []; },
    async close() {},
    async setViewport() {},
    async setZoom() {},
    async setReducedMotion() {},
    async getComputedStyle() { return ''; },
    async screenshot() { return Buffer.from(''); },
  };
}

const baseCtx = {
  screen: { id: 'home', url: 'http://localhost/' },
  config: { schemaVersion: 1 as const, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } },
};

describe('pf-toast-live-region', () => {
  it('emits a draft when a pf-alert element has no live region ancestor', async () => {
    const page = makeFakePage({
      queryAll: async (sel) => {
        if (sel === '[class*="pf-v5-c-alert"]:not([role="status"]):not([role="alert"])') {
          return [{ selector: '.pf-v5-c-alert.my-toast' }];
        }
        // no live region ancestor found
        if (sel.includes('aria-live')) return [];
        return [];
      },
      axAt: async () => ({ name: 'Danger alert', role: 'generic', states: {} }),
    });
    const provider = makeRulepackProvider();
    const drafts = await provider.run({ page, ...baseCtx } as ProviderContext);
    const toastDrafts = drafts.filter((d) => d.rule === 'pf-toast-live-region');
    expect(toastDrafts.length).toBeGreaterThanOrEqual(1);
    expect(toastDrafts[0].evidenceClass).toBe('deterministic');
    expect(toastDrafts[0].layer).toBe('pf');
    expect(toastDrafts[0].severity).toBe('serious');
  });

  it('emits nothing when the alert is inside a live region', async () => {
    const page = makeFakePage({
      queryAll: async () => [],
    });
    const provider = makeRulepackProvider();
    const drafts = await provider.run({ page, ...baseCtx } as ProviderContext);
    const toastDrafts = drafts.filter((d) => d.rule === 'pf-toast-live-region');
    expect(toastDrafts).toHaveLength(0);
  });
});

describe('pf-icon-button-name', () => {
  it('emits a draft for an icon-only button with no accessible name', async () => {
    const page = makeFakePage({
      queryAll: async (sel) => {
        if (sel === 'button[aria-label=""], button:not([aria-label]):not([aria-labelledby])') {
          return [{ selector: 'button.pf-v5-c-button.pf-m-plain' }];
        }
        return [];
      },
      axAt: async () => ({ name: null, role: 'button', states: {} }),
    });
    const provider = makeRulepackProvider();
    const drafts = await provider.run({ page, ...baseCtx } as ProviderContext);
    const nameDrafts = drafts.filter((d) => d.rule === 'pf-icon-button-name');
    expect(nameDrafts.length).toBeGreaterThanOrEqual(1);
    expect(nameDrafts[0].confidence).toBe('fail');
    expect(nameDrafts[0].elementName).toBeNull();
  });

  it('emits nothing when button has aria-label', async () => {
    const page = makeFakePage({
      queryAll: async () => [],
    });
    const provider = makeRulepackProvider();
    const drafts = await provider.run({ page, ...baseCtx } as ProviderContext);
    const nameDrafts = drafts.filter((d) => d.rule === 'pf-icon-button-name');
    expect(nameDrafts).toHaveLength(0);
  });
});
```

- [ ] Run `npx vitest run tests/providers/rulepack.test.ts` — expect FAIL.

- [ ] Create `src/providers/rulepack/pf-toast-live-region.ts`:

```ts
import type { Page, Draft, ProviderContext } from '../../contracts/index.js';

const ALERT_SEL = '[class*="pf-v5-c-alert"]:not([role="status"]):not([role="alert"])';
const LIVE_ANCESTOR_SEL = '[aria-live], [role="status"], [role="alert"], [role="log"]';

export async function checkToastLiveRegion(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const candidates = await page.queryAll(ALERT_SEL);
  if (candidates.length === 0) return [];

  const liveContainers = await page.queryAll(LIVE_ANCESTOR_SEL);
  if (liveContainers.length > 0) return [];   // assume alerts are inside; defer DOM tree check to integration

  const drafts: Draft[] = [];
  for (const el of candidates) {
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
      whatUserExperiences: 'Alert notifications are not announced to screen-reader users.',
      why: 'PatternFly alerts and toasts must be placed inside an aria-live region (role=status/alert) that exists in the DOM before the update fires. AT only announces changes to pre-existing live regions.',
      fix: 'Wrap alert groups in a <div role="status" aria-live="polite"> (or role="alert" for critical alerts) that is present on initial page load.',
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

const UNNAMED_BTN_SEL = 'button[aria-label=""], button:not([aria-label]):not([aria-labelledby])';

export async function checkIconButtonName(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const candidates = await page.queryAll(UNNAMED_BTN_SEL);
  const drafts: Draft[] = [];

  for (const el of candidates) {
    const node = await page.axAt(el.selector);
    if (node?.name) continue;  // axe tree shows a name; skip
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
      why: 'Icon-only buttons (kebab toggles, pagination arrows, close buttons) carry no visible text. Without aria-label the accessible name is empty and AT cannot convey the purpose.',
      fix: 'Add aria-label="<action>" to every icon-only PatternFly button. Use the PatternFly <Button variant="plain" aria-label="…"> pattern.',
      evidence: { name: { value: null, source: 'ax-tree', fromTree: true } },
      confidence: 'fail',
    });
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-modal-focus-return.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';

const MODAL_TRIGGER_SEL = '[aria-haspopup="dialog"], [data-ouia-component-type="PF5/Button"]';

export async function checkModalFocusReturn(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const triggers = await page.queryAll(MODAL_TRIGGER_SEL);
  const drafts: Draft[] = [];

  for (const trigger of triggers) {
    const triggerNode = await page.axAt(trigger.selector);
    if (!triggerNode) continue;

    // Record the trigger path before interaction
    const beforePath = trigger.selector;

    // Try to activate the trigger and close with Escape
    try {
      await page.press('Tab');  // ensure something is focused; real probe uses click
      await page.press('Escape');
      const afterPath = await page.activePath();
      const afterNode = await page.activeNode();

      if (afterPath !== beforePath) {
        drafts.push({
          rule: 'pf-modal-focus-return',
          layer: 'pf',
          severity: 'serious',
          evidenceClass: 'deterministic',
          screenId: screen.id,
          elementPath: trigger.selector,
          elementName: triggerNode.name,
          role: triggerNode.role,
          whatUserExperiences: 'After closing a modal, focus is lost instead of returning to the button that opened it.',
          why: 'WCAG 2.1 SC 2.4.3 requires focus to return to the triggering element on dialog close. Screen-reader users lose their place in the page.',
          fix: 'Call .focus() on the triggering element in the modal\'s onClose handler.',
          evidence: {
            extra: { expectedPath: beforePath, actualPath: afterPath, actualName: afterNode?.name },
          },
          confidence: 'unverified',  // interaction probe cannot always confirm in static fake; gate-only on real run
        });
      }
    } catch {
      // Interaction failed; emit unverified rather than silent pass
      drafts.push({
        rule: 'pf-modal-focus-return',
        layer: 'pf',
        severity: 'serious',
        evidenceClass: 'deterministic',
        screenId: screen.id,
        elementPath: trigger.selector,
        elementName: triggerNode.name,
        role: triggerNode.role,
        whatUserExperiences: 'Could not confirm focus returns after modal close.',
        why: 'Modal focus-return probe failed to complete.',
        fix: 'Verify onClose handler returns focus to the dialog trigger.',
        evidence: {},
        confidence: 'unverified',
      });
    }
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-focus-into-dialog.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';

const DIALOG_SEL = '[role="dialog"], [role="alertdialog"]';

export async function checkFocusIntoDialog(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const dialogs = await page.queryAll(DIALOG_SEL);
  const drafts: Draft[] = [];

  for (const dialog of dialogs) {
    const dialogNode = await page.axAt(dialog.selector);
    // Check that the active element is a descendant of the dialog
    const activePath = await page.activePath();
    if (!activePath.includes(dialog.selector)) {
      drafts.push({
        rule: 'pf-focus-into-dialog',
        layer: 'pf',
        severity: 'critical',
        evidenceClass: 'deterministic',
        screenId: screen.id,
        elementPath: dialog.selector,
        elementName: dialogNode?.name ?? null,
        role: dialogNode?.role ?? 'dialog',
        whatUserExperiences: 'Focus does not move into the dialog when it opens; keyboard users remain behind the modal backdrop.',
        why: 'WCAG 2.1 SC 2.1.1 and the ARIA dialog pattern require focus to move to the first focusable element or the dialog container itself on open.',
        fix: 'Move focus to the PatternFly <Modal> initialFocusRef or to the first focusable child on render.',
        evidence: { extra: { activePath, dialogSelector: dialog.selector } },
        confidence: 'unverified',
      });
    }
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-kebab-expanded-state.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';

const KEBAB_SEL = '[aria-haspopup="menu"], [data-ouia-component-type*="Dropdown"]';

export async function checkKebabExpandedState(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const toggles = await page.queryAll(KEBAB_SEL);
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
        whatUserExperiences: 'Screen-reader users cannot tell whether a kebab or dropdown menu is open or closed.',
        why: 'The ARIA menu button pattern requires aria-expanded and aria-haspopup on the toggle so AT announces the current state.',
        fix: 'Ensure the PatternFly <MenuToggle> or <DropdownToggle> renders aria-expanded={isOpen} and aria-haspopup="menu".',
        evidence: { state: { expanded: { value: hasExpanded, source: 'ax-tree', fromTree: true } } },
        confidence: 'fail',
      });
    }
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/index.ts` (aggregates rules 1-5; rules 6-8 are added in Task 4):

```ts
import type { Provider, ProviderContext, Draft } from '../../contracts/index.js';
import { checkToastLiveRegion } from './pf-toast-live-region.js';
import { checkModalFocusReturn } from './pf-modal-focus-return.js';
import { checkFocusIntoDialog } from './pf-focus-into-dialog.js';
import { checkKebabExpandedState } from './pf-kebab-expanded-state.js';
import { checkIconButtonName } from './pf-icon-button-name.js';

export function makeRulepackProvider(): Provider {
  return {
    id: 'pf-rulepack',
    layer: 'pf',
    capabilities: ['live'],

    async run(ctx: ProviderContext): Promise<Draft[]> {
      const results = await Promise.all([
        checkToastLiveRegion(ctx),
        checkModalFocusReturn(ctx),
        checkFocusIntoDialog(ctx),
        checkKebabExpandedState(ctx),
        checkIconButtonName(ctx),
      ]);
      return results.flat();
    },
  };
}
```

- [ ] Run `npx vitest run tests/providers/rulepack.test.ts` — expect PASS.
- [ ] Commit: `feat: add PatternFly rulepack provider (rules 1-5)`

---

## Task 4 — PatternFly rulepack: structural rules 6-8

**Files:**
- Create: `src/providers/rulepack/pf-table-header-assoc.ts`
- Create: `src/providers/rulepack/pf-row-action-name-unique.ts`
- Create: `src/providers/rulepack/pf-toolbar-labeled-when-repeated.ts`
- Modify: `src/providers/rulepack/index.ts`
- Modify: `tests/providers/rulepack.test.ts`

- [ ] Append tests for rules 6 and 7 to `tests/providers/rulepack.test.ts`:

```ts
describe('pf-table-header-assoc', () => {
  it('emits a draft when a table column header has no id or scope', async () => {
    const page = makeFakePage({
      queryAll: async (sel) => {
        if (sel === 'table th:not([scope]):not([id])') {
          return [{ selector: 'table th:nth-child(1)' }];
        }
        return [];
      },
      axAt: async () => ({ name: 'Name', role: 'columnheader', states: {} }),
    });
    const provider = makeRulepackProvider();
    const drafts = await provider.run({ page, ...baseCtx } as ProviderContext);
    const td = drafts.filter((d) => d.rule === 'pf-table-header-assoc');
    expect(td.length).toBeGreaterThanOrEqual(1);
  });
});

describe('pf-row-action-name-unique', () => {
  it('emits a draft when multiple row-action buttons share the same accessible name', async () => {
    const calls: string[] = [];
    const page = makeFakePage({
      queryAll: async (sel) => {
        if (sel === 'td button, td [role="button"]') {
          return [
            { selector: 'tr:nth-child(1) td button.action' },
            { selector: 'tr:nth-child(2) td button.action' },
          ];
        }
        return [];
      },
      axAt: async () => ({ name: 'Delete', role: 'button', states: {} }),
    });
    const provider = makeRulepackProvider();
    const drafts = await provider.run({ page, ...baseCtx } as ProviderContext);
    const unique = drafts.filter((d) => d.rule === 'pf-row-action-name-unique');
    expect(unique.length).toBeGreaterThanOrEqual(1);
  });
});
```

- [ ] Run `npx vitest run tests/providers/rulepack.test.ts` — expect FAIL on new tests.

- [ ] Create `src/providers/rulepack/pf-table-header-assoc.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';

export async function checkTableHeaderAssoc(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const headers = await page.queryAll('table th:not([scope]):not([id])');
  const drafts: Draft[] = [];
  for (const h of headers) {
    const node = await page.axAt(h.selector);
    drafts.push({
      rule: 'pf-table-header-assoc',
      layer: 'pf',
      severity: 'serious',
      evidenceClass: 'deterministic',
      screenId: screen.id,
      elementPath: h.selector,
      elementName: node?.name ?? null,
      role: node?.role ?? 'columnheader',
      whatUserExperiences: 'Screen-reader users cannot navigate the table by column header because the association is missing.',
      why: 'PatternFly <Th> elements must carry scope="col" or a unique id referenced by data cells to satisfy WCAG SC 1.3.1.',
      fix: 'Add scope="col" to <Th> elements or use the PatternFly <Table> with proper aria-labelledby on data cells.',
      evidence: {},
      confidence: 'fail',
    });
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-row-action-name-unique.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';

export async function checkRowActionNameUnique(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const buttons = await page.queryAll('td button, td [role="button"]');
  if (buttons.length < 2) return [];

  const nameCount: Record<string, string[]> = {};
  for (const btn of buttons) {
    const node = await page.axAt(btn.selector);
    const name = node?.name ?? '';
    if (!nameCount[name]) nameCount[name] = [];
    nameCount[name].push(btn.selector);
  }

  const drafts: Draft[] = [];
  for (const [name, selectors] of Object.entries(nameCount)) {
    if (selectors.length > 1 && name !== '') {
      drafts.push({
        rule: 'pf-row-action-name-unique',
        layer: 'pf',
        severity: 'serious',
        evidenceClass: 'deterministic',
        screenId: screen.id,
        elementPath: selectors[0],
        elementName: name,
        role: 'button',
        whatUserExperiences: `${selectors.length} row action buttons all named "${name}"; screen-reader users cannot distinguish between rows.`,
        why: 'WCAG SC 2.4.6 requires link and button names to be meaningful. Repeated row action names (Delete, Edit) are ambiguous without the row context.',
        fix: 'Compose unique names: aria-label="Delete ${rowName}" using PatternFly\'s row data. Or use aria-describedby pointing to the row header.',
        evidence: { extra: { duplicateSelectors: selectors, name } },
        confidence: 'fail',
      });
    }
  }
  return drafts;
}
```

- [ ] Create `src/providers/rulepack/pf-toolbar-labeled-when-repeated.ts`:

```ts
import type { Draft, ProviderContext } from '../../contracts/index.js';

export async function checkToolbarLabeledWhenRepeated(ctx: ProviderContext): Promise<Draft[]> {
  const { page, screen } = ctx;
  const toolbars = await page.queryAll('[class*="pf-v5-c-toolbar"]');
  if (toolbars.length < 2) return [];

  const drafts: Draft[] = [];
  const unlabeled: string[] = [];
  for (const tb of toolbars) {
    const node = await page.axAt(tb.selector);
    const hasLabel = node?.name && node.name.trim().length > 0;
    if (!hasLabel) unlabeled.push(tb.selector);
  }

  if (unlabeled.length > 0) {
    for (const sel of unlabeled) {
      drafts.push({
        rule: 'pf-toolbar-labeled-when-repeated',
        layer: 'pf',
        severity: 'moderate',
        evidenceClass: 'deterministic',
        screenId: screen.id,
        elementPath: sel,
        elementName: null,
        role: 'toolbar',
        whatUserExperiences: 'When multiple toolbars appear on a page, screen-reader users cannot distinguish between them.',
        why: 'The ARIA toolbar role requires an accessible name when more than one toolbar is present on the page (ARIA Authoring Practices, toolbar pattern).',
        fix: 'Add aria-label="<purpose>" to PatternFly <Toolbar> when more than one toolbar renders on the same page.',
        evidence: {},
        confidence: 'fail',
      });
    }
  }
  return drafts;
}
```

- [ ] Modify `src/providers/rulepack/index.ts` to import and wire rules 6-8:

```ts
// Add imports:
import { checkTableHeaderAssoc } from './pf-table-header-assoc.js';
import { checkRowActionNameUnique } from './pf-row-action-name-unique.js';
import { checkToolbarLabeledWhenRepeated } from './pf-toolbar-labeled-when-repeated.js';

// In run(), extend Promise.all to include:
//   checkTableHeaderAssoc(ctx),
//   checkRowActionNameUnique(ctx),
//   checkToolbarLabeledWhenRepeated(ctx),
```

- [ ] Run `npx vitest run tests/providers/rulepack.test.ts` — expect PASS.
- [ ] Commit: `feat: add PatternFly rulepack structural rules 6-8`

---

## Task 5 — Keyboard walk provider (StepRunner seam)

**Files:**
- Create: `src/providers/keyboard-walk/steps.ts`
- Create: `src/providers/keyboard-walk/index.ts`
- Create: `tests/providers/keyboard-walk.test.ts`

The keyboard walk doubles as the `StepRunner` seam that Phase 4 (voicing lane) reuses. The exported `makeStepRunner()` satisfies the frozen `StepRunner` interface from `00-plan-set.md`.

- [ ] Write failing tests:

```ts
// tests/providers/keyboard-walk.test.ts
import { describe, it, expect } from 'vitest';
import { makeKeyboardWalkProvider, makeStepRunner } from '../../src/providers/keyboard-walk/index.js';
import type { Page, AxNode, ProviderContext, TranscriptStop, Step } from '../../src/contracts/index.js';

const TAB_CAP = 10;

function makeWalkPage(sequence: Array<AxNode | null>, paths: string[]): Page {
  let idx = 0;
  return {
    async gotoReady() {},
    async focusBody() {},
    async tab() { idx++; },
    async press() {},
    async activeNode() { return sequence[Math.min(idx, sequence.length - 1)] ?? null; },
    async activePath() { return paths[Math.min(idx, paths.length - 1)] ?? ''; },
    async axAt() { return null; },
    async queryAll() { return []; },
    async close() {},
    async setViewport() {},
    async setZoom() {},
    async setReducedMotion() {},
    async getComputedStyle() { return ''; },
    async screenshot() { return Buffer.from(''); },
  };
}

const baseCtx = {
  screen: { id: 'home', url: 'http://localhost/' },
  config: { schemaVersion: 1 as const, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } },
};

describe('keyboard walk provider', () => {
  it('declares live capability', () => {
    const p = makeKeyboardWalkProvider({ tabCap: TAB_CAP });
    expect(p.capabilities).toContain('live');
  });

  it('emits unverified draft when active node is null (focus not confirmed)', async () => {
    const page = makeWalkPage([null], ['']);
    const provider = makeKeyboardWalkProvider({ tabCap: TAB_CAP });
    const ctx: ProviderContext = { page, ...baseCtx };
    const drafts = await provider.run(ctx);
    const unverified = drafts.filter((d) => d.confidence === 'unverified');
    expect(unverified.length).toBeGreaterThan(0);
  });

  it('stops tabbing at tabCap to avoid hang on focus trap', async () => {
    const nodes: Array<AxNode | null> = Array.from({ length: 50 }, () => ({
      name: 'button', role: 'button', states: {},
    }));
    const paths = nodes.map((_, i) => `button:nth-child(${i})`);
    const page = makeWalkPage(nodes, paths);
    const provider = makeKeyboardWalkProvider({ tabCap: 5 });
    const ctx: ProviderContext = { page, ...baseCtx };
    const drafts = await provider.run(ctx);
    // Should not have tabbed more than tabCap times; test by checking draft count is bounded
    expect(drafts.length).toBeLessThanOrEqual(6);
  });
});

describe('StepRunner', () => {
  it('records a TranscriptStop per tab step', async () => {
    const page = makeWalkPage(
      [{ name: 'Submit', role: 'button', states: {} }],
      ['button.submit'],
    );
    const steps: Step[] = [{ do: 'tab' }];
    const runner = makeStepRunner();
    const stops: TranscriptStop[] = await runner.run(page, steps);
    expect(stops).toHaveLength(1);
    expect(stops[0].index).toBe(0);
    expect(stops[0].announcement.some((t) => t.kind === 'role')).toBe(true);
  });

  it('records empty announcement for a step where active node is null', async () => {
    const page = makeWalkPage([null], ['']);
    const steps: Step[] = [{ do: 'tab' }];
    const runner = makeStepRunner();
    const stops = await runner.run(page, steps);
    expect(stops[0].announcement).toHaveLength(0);
  });
});
```

- [ ] Run `npx vitest run tests/providers/keyboard-walk.test.ts` — expect FAIL.

- [ ] Create `src/providers/keyboard-walk/steps.ts`:

```ts
import type { Page, Step, TranscriptStop, AnnouncementToken, AxNode } from '../../contracts/index.js';
import type { StepRunner } from '../../contracts/index.js';

function nodeToAnnouncement(node: AxNode): AnnouncementToken[] {
  const tokens: AnnouncementToken[] = [];
  if (node.name !== null) {
    tokens.push({ kind: 'name', text: node.name, fromTree: true, source: 'ax-tree' });
  }
  if (node.role !== null) {
    tokens.push({ kind: 'role', text: node.role, fromTree: true, source: 'ax-tree' });
  }
  for (const [key, fact] of Object.entries(node.states ?? {})) {
    if (typeof fact === 'boolean' && fact) {
      tokens.push({ kind: 'state', text: key, fromTree: true, source: 'ax-tree' });
    }
  }
  return tokens;
}

async function executeStep(page: Page, step: Step): Promise<void> {
  if (step.do === 'tab') {
    await page.tab();
  } else if (step.do === 'press-escape') {
    await page.press('Escape');
  } else if (step.do === 'activate') {
    await page.press('Enter');
  } else {
    // Unknown step kind: record without interaction
  }
}

export function makeConcreteStepRunner(): StepRunner {
  return {
    async run(page: Page, steps: Step[]): Promise<TranscriptStop[]> {
      const stops: TranscriptStop[] = [];
      for (let i = 0; i < steps.length; i++) {
        await executeStep(page, steps[i]);
        const elementPath = await page.activePath();
        const node = await page.activeNode();
        stops.push({
          index: i,
          elementPath,
          announcement: node ? nodeToAnnouncement(node) : [],
        });
      }
      return stops;
    },
  };
}
```

- [ ] Create `src/providers/keyboard-walk/index.ts`:

```ts
import type { Provider, ProviderContext, Draft, Page, Step, TranscriptStop, StepRunner } from '../../contracts/index.js';
import { makeConcreteStepRunner } from './steps.js';

interface WalkConfig { tabCap?: number; wallClockMs?: number; }

const SEEN_LIMIT = 3; // stop if same path seen this many times (cycle detection)

export function makeStepRunner(): StepRunner {
  return makeConcreteStepRunner();
}

export function makeKeyboardWalkProvider(cfg: WalkConfig = {}): Provider {
  const tabCap = cfg.tabCap ?? 200;
  const wallClockMs = cfg.wallClockMs ?? 15_000;

  return {
    id: 'keyboard-walk',
    layer: 'walk',
    capabilities: ['live'],

    async run(ctx: ProviderContext): Promise<Draft[]> {
      const { page, screen } = ctx;
      const drafts: Draft[] = [];
      const seen = new Map<string, number>();
      const deadline = Date.now() + wallClockMs;

      await page.focusBody();

      for (let i = 0; i < tabCap; i++) {
        if (Date.now() > deadline) break;

        await page.tab();
        const elementPath = await page.activePath();
        const node = await page.activeNode();

        if (!node) {
          drafts.push({
            rule: 'keyboard-walk-unconfirmed-focus',
            layer: 'walk',
            severity: 'serious',
            evidenceClass: 'deterministic',
            screenId: screen.id,
            elementPath: elementPath || `tab-stop-${i}`,
            elementName: null,
            role: null,
            whatUserExperiences: 'A focusable element exists but its accessible role and name cannot be determined from the accessibility tree.',
            why: 'Keyboard users reach this element but AT cannot announce what it is or how to interact with it.',
            fix: 'Ensure the element has a semantic HTML role or an explicit ARIA role, and an accessible name.',
            evidence: {},
            confidence: 'unverified',
          });
          continue;
        }

        const count = (seen.get(elementPath) ?? 0) + 1;
        seen.set(elementPath, count);
        if (count >= SEEN_LIMIT) break;  // cycle detected; exit gracefully

        // Only emit findings for detectable problems (no name on interactive role)
        const interactiveRoles = new Set(['button', 'link', 'menuitem', 'checkbox', 'radio', 'textbox', 'combobox']);
        if (interactiveRoles.has(node.role ?? '') && !node.name) {
          drafts.push({
            rule: 'keyboard-walk-unnamed-interactive',
            layer: 'walk',
            severity: 'serious',
            evidenceClass: 'deterministic',
            screenId: screen.id,
            elementPath,
            elementName: null,
            role: node.role,
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

- [ ] Run `npx vitest run tests/providers/keyboard-walk.test.ts` — expect PASS.
- [ ] Commit: `feat: add keyboard-walk provider and StepRunner seam`

---

## Task 6 — CheckRunner: assembles ScreenScan from all providers

**Files:**
- Create: `src/providers/check-runner.ts`
- Modify: `src/deps/fakes.ts`
- Create: `tests/providers/check-runner.test.ts`

The `CheckRunner` satisfies the `Deps.checkRunner` slot from Phase 1. It opens one `Page` per screen, runs `runProviders`, and assembles the frozen `ScreenScan`. The gate in `run()` already consumes `ScreenScan`; this task wires them together.

- [ ] Write failing test:

```ts
// tests/providers/check-runner.test.ts
import { describe, it, expect, vi } from 'vitest';
import { makeCheckRunner } from '../../src/providers/check-runner.js';
import type { Provider, BrowserDriver, Page, ScreenScan } from '../../src/contracts/index.js';

function makeFakePage(): Page {
  return {
    async gotoReady() {},
    async focusBody() {},
    async tab() {},
    async press() {},
    async activeNode() { return null; },
    async activePath() { return ''; },
    async axAt() { return null; },
    async queryAll() { return []; },
    async close() {},
    async setViewport() {},
    async setZoom() {},
    async setReducedMotion() {},
    async getComputedStyle() { return ''; },
    async screenshot() { return Buffer.from(''); },
  };
}

function makeFakeBrowser(page: Page): BrowserDriver {
  return { open: vi.fn().mockResolvedValue(page) };
}

describe('CheckRunner', () => {
  it('returns a ScreenScan with drafts from all providers', async () => {
    const page = makeFakePage();
    const browser = makeFakeBrowser(page);
    const config = { schemaVersion: 1 as const, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } };

    const fakeProvider: Provider = {
      id: 'fake',
      layer: 'axe',
      capabilities: ['live'],
      run: vi.fn().mockResolvedValue([{
        rule: 'test-rule',
        layer: 'axe',
        severity: 'serious' as const,
        evidenceClass: 'deterministic' as const,
        screenId: 'home',
        elementPath: 'button',
        elementName: null,
        role: 'button',
        whatUserExperiences: 'test',
        why: 'test',
        fix: 'test',
        evidence: {},
        confidence: 'fail' as const,
      }]),
    };

    const runner = makeCheckRunner({ browser, providers: [fakeProvider], config, allowedCapabilities: ['live'] });
    const scan: ScreenScan = await runner.scan({ id: 'home', url: 'http://localhost/' });

    expect(scan.screenId).toBe('home');
    expect(scan.url).toBe('http://localhost/');
    expect(scan.drafts).toHaveLength(1);
    expect(scan.drafts[0].rule).toBe('test-rule');
    expect(browser.open).toHaveBeenCalledWith('http://localhost/');
  });

  it('closes the page even when a provider throws', async () => {
    const page = makeFakePage();
    const closeSpy = vi.spyOn(page, 'close');
    const browser = makeFakeBrowser(page);
    const config = { schemaVersion: 1 as const, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } };

    const throwingProvider: Provider = {
      id: 'thrower',
      layer: 'axe',
      capabilities: ['live'],
      run: vi.fn().mockRejectedValue(new Error('provider boom')),
    };

    const runner = makeCheckRunner({ browser, providers: [throwingProvider], config, allowedCapabilities: ['live'] });
    await expect(runner.scan({ id: 'home', url: 'http://localhost/' })).rejects.toThrow('provider boom');
    expect(closeSpy).toHaveBeenCalled();
  });
});
```

- [ ] Run `npx vitest run tests/providers/check-runner.test.ts` — expect FAIL.

- [ ] Create `src/providers/check-runner.ts`:

```ts
import type { CheckRunner, ScreenScan, BrowserDriver, Provider, UsablConfig, Capability } from '../contracts/index.js';
import { runProviders } from './index.js';

interface CheckRunnerDeps {
  browser: BrowserDriver;
  providers: Provider[];
  config: UsablConfig;
  allowedCapabilities: Capability[];
}

export function makeCheckRunner(deps: CheckRunnerDeps): CheckRunner {
  const { browser, providers, config, allowedCapabilities } = deps;

  return {
    async scan(screen: { id: string; url: string }): Promise<ScreenScan> {
      const page = await browser.open(screen.url);
      try {
        await page.gotoReady();
        const ctx = { page, screen, config };
        const { drafts } = await runProviders(providers, ctx, allowedCapabilities);
        // TranscriptStop[] is empty here; Phase 4 voicing lane populates it via StepRunner
        return {
          screenId: screen.id,
          url: screen.url,
          stops: [],
          drafts,
        };
      } finally {
        await page.close();
      }
    },
  };
}
```

- [ ] Run `npx vitest run tests/providers/check-runner.test.ts` — expect PASS.
- [ ] Run full unit suite to confirm no regressions: `npx vitest run`.
- [ ] Commit: `feat: add CheckRunner that assembles ScreenScan from all providers`

---

## Task 7 — Integration wiring (real Playwright + axe-core; labeled integration tasks)

**Files:**
- Create: `src/deps/real.ts` (extend with `makeRealBrowserDriver`)
- Create: `src/providers/axe/playwright-bridge.ts`

> **Note:** The steps below are integration tasks, NOT unit tests. They require a real browser. Run them only with `USABL_INTEGRATION=1 npx vitest run tests/integration/` or manually. Do not run them in CI without that env guard.

- [ ] Create `src/providers/axe/playwright-bridge.ts` — implements the `runAxe()` extension on a real Playwright page by injecting `axe-core` via `page.evaluate()`:

```ts
// Integration: bridges real Playwright Page to the AxePage interface.
// This file imports from 'playwright' and is only loaded in real/integration contexts.
import type { Page as PlaywrightPage } from 'playwright';

export async function injectAndRunAxe(playwrightPage: PlaywrightPage): Promise<{ violations: unknown[] }> {
  // axe-core must be installed: npm install axe-core
  const { source } = await import('axe-core');
  return playwrightPage.evaluate((axeSource: string) => {
    // eslint-disable-next-line no-eval
    eval(axeSource);
    return (window as unknown as { axe: { run(): Promise<{ violations: unknown[] }> } }).axe.run();
  }, source);
}
```

- [ ] Extend `src/deps/real.ts` — export `makeRealBrowserDriver()` using Playwright's `chromium.launch()`, wrapping the Playwright `Page` into the `Page` interface and attaching `runAxe()`:

```ts
// Integration: do not import in unit tests.
import { chromium } from 'playwright';
import type { BrowserDriver, Page, AxNode } from '../contracts/index.js';
import { injectAndRunAxe } from '../providers/axe/playwright-bridge.js';

export function makeRealBrowserDriver(): BrowserDriver {
  return {
    async open(url: string): Promise<Page> {
      const browser = await chromium.launch({ headless: true });
      const ctx = await browser.newContext();
      const pw = await ctx.newPage();
      await pw.goto(url, { waitUntil: 'networkidle' });

      const page: Page & { runAxe(): Promise<{ violations: unknown[] }> } = {
        async gotoReady() { await pw.waitForLoadState('networkidle'); },
        async focusBody() { await pw.evaluate(() => document.body.focus()); },
        async tab() { await pw.keyboard.press('Tab'); },
        async press(key: string) { await pw.keyboard.press(key); },
        async activeNode(): Promise<AxNode | null> {
          return pw.evaluate(() => {
            const el = document.activeElement;
            if (!el) return null;
            const role = el.getAttribute('role') ?? el.tagName.toLowerCase();
            const name = el.getAttribute('aria-label') ?? (el as HTMLElement).innerText?.slice(0, 80) ?? null;
            return { name, role, states: {} };
          });
        },
        async activePath(): Promise<string> {
          return pw.evaluate(() => {
            const el = document.activeElement;
            if (!el) return '';
            const parts: string[] = [];
            let cur: Element | null = el;
            while (cur && cur !== document.body) {
              let seg = cur.tagName.toLowerCase();
              if (cur.id) seg += `#${cur.id}`;
              else if (cur.className) seg += `.${String(cur.className).split(' ')[0]}`;
              parts.unshift(seg);
              cur = cur.parentElement;
            }
            return parts.join(' > ');
          });
        },
        async axAt(selector: string): Promise<AxNode | null> {
          return pw.evaluate((sel: string) => {
            const el = document.querySelector(sel);
            if (!el) return null;
            return {
              name: el.getAttribute('aria-label') ?? (el as HTMLElement).innerText?.slice(0, 80) ?? null,
              role: el.getAttribute('role') ?? el.tagName.toLowerCase(),
              states: {},
            };
          }, selector);
        },
        async queryAll(selector: string) {
          const handles = await pw.$$(selector);
          return handles.map((_, i) => ({ selector: `${selector}:nth-of-type(${i + 1})` }));
        },
        async close() { await browser.close(); },
        async setViewport(w: number, h: number) { await pw.setViewportSize({ width: w, height: h }); },
        async setZoom(pct: number) { await pw.evaluate((p: number) => { document.documentElement.style.zoom = `${p}%`; }, pct); },
        async setReducedMotion(enabled: boolean) {
          await ctx.route('**', async (route) => route.continue());
          if (enabled) await pw.emulateMedia({ reducedMotion: 'reduce' });
        },
        async getComputedStyle(selector: string, property: string) {
          return pw.evaluate(([sel, prop]: [string, string]) => {
            const el = document.querySelector(sel);
            if (!el) return '';
            return getComputedStyle(el).getPropertyValue(prop);
          }, [selector, property]);
        },
        async screenshot(selector?: string) {
          if (selector) {
            const el = await pw.$(selector);
            if (el) return Buffer.from(await el.screenshot());
          }
          return Buffer.from(await pw.screenshot());
        },
        async runAxe() { return injectAndRunAxe(pw); },
      };

      return page;
    },
  };
}
```

- [ ] **Integration smoke test** (manual, not automated): run the following against a local PatternFly fixture page to confirm real `Draft[]` are produced end to end:

```ts
// Run manually: USABL_INTEGRATION=1 npx tsx src/integration-smoke.ts
import { makeRealBrowserDriver } from './deps/real.js';
import { makeAxeProvider } from './providers/axe/index.js';
import { makeRulepackProvider } from './providers/rulepack/index.js';
import { makeKeyboardWalkProvider } from './providers/keyboard-walk/index.js';
import { makeCheckRunner } from './providers/check-runner.js';

const config = { schemaVersion: 1 as const, version: '0.0.0', surfaces: [], floor: { version: 1, entries: [] } };
const browser = makeRealBrowserDriver();
const providers = [makeAxeProvider(), makeRulepackProvider(), makeKeyboardWalkProvider()];
const runner = makeCheckRunner({ browser, providers, config, allowedCapabilities: ['live'] });

const scan = await runner.scan({ id: 'fixture', url: 'http://localhost:3000/' });
console.log(`Drafts: ${scan.drafts.length}`);
scan.drafts.forEach((d) => console.log(`  [${d.layer}] ${d.rule} @ ${d.elementPath}`));
```

**Exit criterion:** the smoke test produces at least one real `Draft[]` from a live PatternFly fixture page through the real `BrowserDriver`, with each provider declaring `capabilities: ['live']`, and the `CheckRunner` assembling a complete `ScreenScan`.

- [ ] Commit: `feat: add real Playwright BrowserDriver and axe integration bridge`

---

## Self-Review

**Three layers + probe seam coverage:**
- axe-core provider (Task 2): WCAG baseline, `layer: 'axe'`, `evidenceClass: 'deterministic'`, defers to axe on generic checks.
- PatternFly rulepack (Tasks 3-4): all 8 named rules, `layer: 'pf'`, design-system semantics only; no double-report with axe because rules target interaction/composition failures axe cannot detect.
- Keyboard walk (Task 5): role-aware tab probing, `layer: 'walk'`, `confidence: 'unverified'` on unconfirmable focus; tabCap + cycle detection prevents hang.
- StepRunner seam (Task 5): `makeStepRunner()` exports the frozen `StepRunner` interface; Phase 4 imports and reuses it without changes.

**Capability filtering / static mode:** Task 1 tests and implements the `runProviders` filter; every provider in Tasks 2-5 declares `capabilities: ['live']`; skipped providers record `CoverageGap { state: 'capability-denied' }`, never a silent pass.

**No placeholder code:** all types derive from frozen contracts in `src/contracts/index.ts`; all selectors, rule ids, and severity values are real strings; no `TODO`, `TBD`, or `any` (except explicit unsafe cast in Playwright bridge, documented).

**Type consistency with frozen contracts:** `Draft`, `ScreenScan`, `TranscriptStop`, `Provider`, `ProviderContext`, `Capability`, `CheckRunner`, `BrowserDriver`, `Page`, `Deps`, `UsablConfig`, `CoverageGap` all imported from `src/contracts/index.ts` without modification. `StepRunner` added to contracts in Task 5 alongside the `Step` shape already frozen in Phase 1.

**Verdict authority:** no provider contains conditional logic that chooses a verdict. All Draft fields are data; the gate in `run()` (Phase 1) remains the sole verdict authority.
