# Intake and Accessible Docs Output Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add two capabilities to the usabl engine: (1) a guarded intake path that accepts design and UX handoffs in multiple shapes and normalizes them to a validated `RequirementBundle`, mapping each requirement to a runtime check or a docs-output trigger; and (2) three docs-output generators that project the canonical `Result` plus the `Receipt` into accessible documentation artifacts, each entry bound via `evidenceRef` to the specific receipt evidence that backs it.

**Architecture:**
- Every intake path calls `normalize(raw, format)` which validates against the Zod `RequirementBundle` schema; any malformed or ambiguous input returns `{ ok: false, verdict: 'approval_required' }`: never a silent default, never a stderr warning.
- Every docs generator is a pure function `(result, receipt, ...) => DocArtifact`. No network, no file I/O; callers supply deps.
- `evidenceRef` is a deterministic string (`receipt:{sourceTree}:{screenId}:{discriminator}` or `stop:{sourceTree}:{screenId}:{index}`) that resolves unambiguously to evidence in the receipt's code state. The honesty rule is enforceable and enforced: an `evidenceRef` is emitted ONLY when the receipt exists AND the surface appears in `receipt.coverage.checked`. A requirement on a surface the run never checked gets no `evidenceRef`, never a fabricated one.
- `boundToReceipt: receipt.sourceTree` on every `DocArtifact`. If the tree changes and the receipt is invalidated, callers can detect stale docs by comparing the stored `boundToReceipt` against the current tree hash.
- AI-generated text (alt text proposals, annotation copy) is `status: 'draft'` until `approved: true` is set in the requirement. Only approved requirements emit `status: 'approved'` entries.
- All contracts (`RequirementBundle`, `Requirement`, `ContentAssertion`, `FlowAssertion`, `DocAssertion`, `DocArtifact`, `Result`, `Receipt`, `ScreenScan`, `TranscriptStop`, `Finding`, `Draft`, `Deps`, `UsablConfig`, `Provider`, `ProviderContext`) are imported from `src/contracts/index.ts`: frozen in Phase 1 (`01-core-foundation.md` Task 2) and Phase 2 (Provider/ProviderContext/StepRunner seams in `00-plan-set.md`).

**Tech Stack:** TypeScript (ESM, strict), Node 22, Vitest (unit tests), Zod pinned to
`^3.23` (the code uses v3 APIs: `discriminatedUnion`, `.passthrough()`; zod v4 changed
them, so an unpinned install fails), `js-yaml` `^4` (YAML parse; pin the major version
because `load` is safe-by-default only in v4; v3 constructs arbitrary types), tsup
(build). No Playwright in unit tests; all tests use in-memory fakes from
`src/deps/fakes.ts` (`makeFakePage`, never hand-rolled `Page` literals).

**Clock discipline:** no `new Date()` anywhere in this phase. `DocArtifact.generatedAt`
is `receipt.mintedAt` when a receipt exists and the empty string otherwise; an artifact
without a receipt has no honest time source and does not invent one.

**Wiring:** Task 8 connects intake to the engine. The requirements directory is already
force-guarded by Phase 3's `buildGuardedSet` (it includes `config.requirements`
unconditionally), so a requirement edit is an `approval_required` event without any
work here.

**Phase dependencies:** Phases 1, 2, and 3 must be complete. Phase 6 reads `Result`, `Receipt`, `Finding`, `ScreenScan`, `TranscriptStop`, `Draft`, `Provider`, `ProviderContext` from already-frozen contracts; it introduces no new contract surface.

---

## Task 1: RequirementBundle Zod schema and validation guard

**Files:**
- `src/intake/schema.ts`
- `test/intake/schema.test.ts`

The `RequirementBundle` type is already frozen in `01-core-foundation.md` Task 2 (lines 387–402). This task adds the Zod runtime schema and the `parseBundle` guard function.

### Steps

- [ ] **1. Write failing test**

```ts
// test/intake/schema.test.ts
import { describe, it, expect } from 'vitest';
import { parseBundle } from '../../src/intake/schema.js';

describe('parseBundle', () => {
  it('accepts a valid content requirement bundle', () => {
    const input = {
      version: 1,
      requirements: [{
        id: 'req-01', kind: 'content', surface: 'clusters',
        description: 'Topology SVG alt text', approved: true,
        assertion: { type: 'content', selector: 'img.topology-svg', expectedText: 'Cluster network topology' },
      }],
    };
    const r = parseBundle(input);
    expect(r.ok).toBe(true);
    if (r.ok) expect(r.bundle.requirements).toHaveLength(1);
  });

  it('returns approval_required on missing required fields', () => {
    const r = parseBundle({ version: 1, requirements: [{ id: 'req-02' }] });
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.verdict).toBe('approval_required');
  });

  it('returns approval_required when assertion type does not match kind', () => {
    const input = {
      version: 1,
      requirements: [{
        id: 'req-03', kind: 'content', surface: 's1', description: 'd', approved: false,
        assertion: { type: 'flow', steps: [] },   // mismatch: kind=content, assertion.type=flow
      }],
    };
    const r = parseBundle(input);
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.verdict).toBe('approval_required');
  });

  it('accepts a doc requirement', () => {
    const input = {
      version: 1,
      requirements: [{
        id: 'req-doc-01', kind: 'doc', surface: 'clusters',
        description: 'Publish alt-text manifest', approved: true,
        assertion: { type: 'doc', artifact: 'alt-text-manifest' },
      }],
    };
    const r = parseBundle(input);
    expect(r.ok).toBe(true);
  });
});
```

- [ ] **2. Run: expect FAIL** (`src/intake/schema.ts` does not exist)

```sh
npx vitest run test/intake/schema.test.ts
```

- [ ] **3. Implement `src/intake/schema.ts`**

```ts
import { z } from 'zod';
import type { RequirementBundle } from '../contracts/index.js';

const contentAssertionSchema = z.object({
  type: z.literal('content'),
  selector: z.string().min(1),
  expectedText: z.string().optional(),
  mustNotBe: z.enum(['decorative', 'empty']).optional(),
});

const flowAssertionSchema = z.object({
  type: z.literal('flow'),
  steps: z.array(z.object({ do: z.string() }).passthrough()),
  expectedAnnouncement: z.string().optional(),
});

const docAssertionSchema = z.object({
  type: z.literal('doc'),
  artifact: z.enum(['alt-text-manifest', 'announcement-snippets', 'keyboard-paths']),
});

const requirementSchema = z
  .object({
    id: z.string().min(1),
    kind: z.enum(['content', 'flow', 'doc']),
    surface: z.string().min(1),
    description: z.string().min(1),
    assertion: z.discriminatedUnion('type', [contentAssertionSchema, flowAssertionSchema, docAssertionSchema]),
    owner: z.string().optional(),
    approved: z.boolean(),
  })
  .superRefine((req, ctx) => {
    if (req.kind !== req.assertion.type) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: `kind '${req.kind}' does not match assertion.type '${req.assertion.type}'`,
      });
    }
  });

const bundleSchema = z.object({
  version: z.literal(1),
  requirements: z.array(requirementSchema).min(1),
});

export type ParseResult =
  | { ok: true; bundle: RequirementBundle }
  | { ok: false; verdict: 'approval_required'; reason: string };

export function parseBundle(raw: unknown): ParseResult {
  const parsed = bundleSchema.safeParse(raw);
  if (!parsed.success) {
    const reason = parsed.error.issues.map(i => i.message).join('; ');
    return { ok: false, verdict: 'approval_required', reason };
  }
  return { ok: true, bundle: parsed.data as RequirementBundle };
}
```

- [ ] **4. Run: expect PASS**

```sh
npx vitest run test/intake/schema.test.ts
```

- [ ] **5. Commit**

```sh
git add src/intake/schema.ts test/intake/schema.test.ts
git commit -m "feat(intake): RequirementBundle Zod schema with approval_required guard"
```

---

## Task 2: Normalize: YAML input shape

**Files:**
- `src/intake/normalize.ts`
- `test/intake/normalize.test.ts`

Primary shape for the contest is hand-authored YAML in the repo. The normalizer is the single entry point for all input shapes; callers never call `parseBundle` directly.

### Steps

- [ ] **1. Write failing test**

```ts
// test/intake/normalize.test.ts
import { describe, it, expect } from 'vitest';
import { normalize } from '../../src/intake/normalize.js';

describe('normalize', () => {
  const validYaml = `
version: 1
requirements:
  - id: req-alt-01
    kind: content
    surface: clusters
    description: Topology SVG alt text
    approved: true
    assertion:
      type: content
      selector: img.topology-svg
      expectedText: "Cluster network topology showing three nodes"
`;

  it('parses valid YAML into a RequirementBundle', () => {
    const r = normalize(validYaml, 'yaml');
    expect(r.ok).toBe(true);
    if (r.ok) {
      expect(r.bundle.version).toBe(1);
      expect(r.bundle.requirements[0].id).toBe('req-alt-01');
    }
  });

  it('returns approval_required on invalid YAML syntax', () => {
    const r = normalize('version: 1\nrequirements: [unclosed', 'yaml');
    expect(r.ok).toBe(false);
    if (!r.ok) expect(r.verdict).toBe('approval_required');
  });

  it('returns approval_required when YAML parses but does not match schema', () => {
    const r = normalize('version: 1\nrequirements:\n  - id: x', 'yaml');
    expect(r.ok).toBe(false);
    if (!r.ok) {
      expect(r.verdict).toBe('approval_required');
      expect(r.reason.length).toBeGreaterThan(0);
    }
  });
});
```

- [ ] **2. Run: expect FAIL**

```sh
npx vitest run test/intake/normalize.test.ts
```

- [ ] **3. Implement `src/intake/normalize.ts`**

```ts
import yaml from 'js-yaml';
import { parseBundle } from './schema.js';
import type { ParseResult } from './schema.js';

export type InputFormat = 'yaml';

export function normalize(raw: string, format: InputFormat = 'yaml'): ParseResult {
  let parsed: unknown;
  try {
    // Intake YAML may be author/contributor-supplied. Restrict to JSON_SCHEMA so no
    // custom YAML tags can construct arbitrary types (fail-closed). The bundle is plain
    // data (version, requirements[]), so JSON_SCHEMA loses nothing we need.
    parsed = yaml.load(raw, { schema: yaml.JSON_SCHEMA });
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err);
    return { ok: false, verdict: 'approval_required', reason: `YAML parse error: ${msg}` };
  }
  return parseBundle(parsed);
}
```

- [ ] **4. Run: expect PASS**

```sh
npx vitest run test/intake/normalize.test.ts
```

- [ ] **5. Commit**

```sh
git add src/intake/normalize.ts test/intake/normalize.test.ts
git commit -m "feat(intake): normalize() converts YAML to RequirementBundle with guard"
```

---

## Task 3: Requirement-to-provider mapping

**Files:**
- `src/intake/map-to-providers.ts`
- `test/intake/map-to-providers.test.ts`

`content` requirements become deterministic content-matcher providers (reuse the `Provider` seam from Phase 2, `00-plan-set.md`). `flow` requirements become declarative walk probes. `doc` requirements feed the output module only: no runtime provider is emitted.

All intake-derived Drafts carry `evidenceClass: 'deterministic'` per ground-truth §13.

### Steps

- [ ] **1. Write failing test**

```ts
// test/intake/map-to-providers.test.ts
import { describe, it, expect } from 'vitest';
import { mapRequirementsToProviders } from '../../src/intake/map-to-providers.js';
import type { RequirementBundle, UsablConfig, ProviderContext, Page } from '../../src/contracts/index.js';

import { makeFakePage } from '../../src/deps/fakes.js';

const fakePage: Page = makeFakePage({
  activeNode: async () => ({ name: 'Wrong label', role: 'img', states: {} }),
  activePath: async () => 'img.topology-svg',
  axAt: async (sel) => sel === 'img.topology-svg'
    ? { name: 'Wrong label', role: 'img', states: {} }
    : null,
});

const fakeConfig: UsablConfig = {
  appBaseUrl: 'http://localhost:3000',
  uiFileGlobs: [],
  discovery: { routerFile: '', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: 'http://localhost:3000/clusters', files: [] }],
  guardedPaths: [],
};

describe('mapRequirementsToProviders', () => {
  it('maps a content requirement to a deterministic provider', () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [{
        id: 'req-alt-01', kind: 'content', surface: 'clusters',
        description: 'Topology SVG alt text', approved: true,
        assertion: { type: 'content', selector: 'img.topology-svg', expectedText: 'Cluster network topology' },
      }],
    };
    const providers = mapRequirementsToProviders(bundle);
    expect(providers).toHaveLength(1);
    expect(providers[0].id).toBe('intake:req-alt-01');
    expect(providers[0].layer).toBe('intake-content');
    expect(providers[0].capabilities).toEqual(['live']);
  });

  it('emits no provider for doc requirements', () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [{
        id: 'req-doc-01', kind: 'doc', surface: 'clusters',
        description: 'Publish alt-text manifest', approved: true,
        assertion: { type: 'doc', artifact: 'alt-text-manifest' },
      }],
    };
    expect(mapRequirementsToProviders(bundle)).toHaveLength(0);
  });

  it('maps a flow requirement to a deterministic provider', () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [{
        id: 'req-flow-01', kind: 'flow', surface: 'clusters',
        description: 'Error announced before focus moves', approved: true,
        assertion: { type: 'flow', steps: [{ do: 'tab' }], expectedAnnouncement: 'Save failed' },
      }],
    };
    const providers = mapRequirementsToProviders(bundle);
    expect(providers).toHaveLength(1);
    expect(providers[0].layer).toBe('intake-flow');
  });

  it('content provider run() emits a deterministic Draft on text mismatch', async () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [{
        id: 'req-alt-01', kind: 'content', surface: 'clusters',
        description: 'Topology SVG alt text', approved: true,
        assertion: { type: 'content', selector: 'img.topology-svg', expectedText: 'Cluster network topology' },
      }],
    };
    const providers = mapRequirementsToProviders(bundle);
    const ctx: ProviderContext = {
      page: fakePage,
      screen: { id: 'clusters', url: 'http://localhost:3000/clusters' },
      config: fakeConfig,
    };
    const drafts = await providers[0].run(ctx);
    expect(drafts).toHaveLength(1);
    expect(drafts[0].evidenceClass).toBe('deterministic');
    expect(drafts[0].rule).toBe('intake:req-alt-01');
    expect(drafts[0].confidence).toBe('fail');
  });

  it('content provider run() emits no drafts when text matches', async () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [{
        id: 'req-alt-02', kind: 'content', surface: 'clusters',
        description: 'Topology SVG alt text', approved: true,
        assertion: { type: 'content', selector: 'img.topology-svg', expectedText: 'Wrong label' },
      }],
    };
    const providers = mapRequirementsToProviders(bundle);
    const ctx: ProviderContext = {
      page: fakePage,
      screen: { id: 'clusters', url: 'http://localhost:3000/clusters' },
      config: fakeConfig,
    };
    const drafts = await providers[0].run(ctx);
    expect(drafts).toHaveLength(0);
  });
});
```

- [ ] **2. Run: expect FAIL**

```sh
npx vitest run test/intake/map-to-providers.test.ts
```

- [ ] **3. Implement `src/intake/map-to-providers.ts`**

```ts
import type {
  Provider, ProviderContext, Draft, RequirementBundle, Requirement,
  ContentAssertion, FlowAssertion,
} from '../contracts/index.js';

function makeContentProvider(req: Requirement, assertion: ContentAssertion): Provider {
  return {
    id: `intake:${req.id}`,
    layer: 'intake-content',
    capabilities: ['live'],
    async run(ctx: ProviderContext): Promise<Draft[]> {
      if (ctx.screen.id !== req.surface) return [];
      const node = await ctx.page.axAt(assertion.selector);
      const drafts: Draft[] = [];

      if (assertion.expectedText !== undefined) {
        const actual = node?.name ?? null;
        if (actual !== assertion.expectedText) {
          drafts.push({
            rule: `intake:${req.id}`,
            layer: 'intake-content',
            severity: 'serious',
            evidenceClass: 'deterministic',
            screenId: req.surface,
            elementPath: assertion.selector,
            elementName: actual,
            role: node?.role ?? null,
            whatUserExperiences: `Screen reader announces "${actual ?? 'nothing'}" instead of the approved text`,
            why: `Content requirement ${req.id} specifies exact accessible name "${assertion.expectedText}"`,
            fix: `Update the element's alt/aria-label to match approved string: "${assertion.expectedText}"`,
            evidence: { name: { value: actual, source: 'ax-tree', fromTree: true } },
            confidence: 'fail',
          });
        }
      }

      if (assertion.mustNotBe === 'empty') {
        const actual = node?.name ?? null;
        if (actual === null || actual.trim() === '') {
          drafts.push({
            rule: `intake:${req.id}`,
            layer: 'intake-content',
            severity: 'serious',
            evidenceClass: 'deterministic',
            screenId: req.surface,
            elementPath: assertion.selector,
            elementName: null,
            role: node?.role ?? null,
            whatUserExperiences: 'Element has no accessible name; screen reader will not describe it',
            why: `Content requirement ${req.id} requires a non-empty accessible name on ${assertion.selector}`,
            fix: 'Add a non-empty alt attribute or aria-label to the element',
            evidence: { name: { value: null, source: 'ax-tree', fromTree: true } },
            confidence: 'fail',
          });
        }
      }

      return drafts;
    },
  };
}

function makeFlowProvider(req: Requirement, assertion: FlowAssertion): Provider {
  return {
    id: `intake:${req.id}`,
    layer: 'intake-flow',
    capabilities: ['live'],
    async run(ctx: ProviderContext): Promise<Draft[]> {
      if (ctx.screen.id !== req.surface) return [];
      await ctx.page.armAnnouncementCapture();
      for (const step of assertion.steps) {
        if (step.do === 'tab') await ctx.page.tab();
        else if (step.do === 'activate') await ctx.page.press('Enter');
        else if (step.do === 'click' && typeof step['selector'] === 'string') {
          await ctx.page.click(step['selector'] as string);
        } else if (step.do === 'press' && typeof step['key'] === 'string') {
          await ctx.page.press(step['key'] as string);
        }
      }
      if (assertion.expectedAnnouncement !== undefined) {
        // The expected text can arrive on the focused node OR through a live region
        // (a toast is never on the focused node). Both count; silence fails.
        const node = await ctx.page.activeNode();
        const focusText = node?.name ?? '';
        const liveTexts = await ctx.page.drainAnnouncements();
        const heard = [focusText, ...liveTexts].some((t) => t.includes(assertion.expectedAnnouncement!));
        if (!heard) {
          return [{
            rule: `intake:${req.id}`,
            layer: 'intake-flow',
            severity: 'serious',
            evidenceClass: 'deterministic',
            screenId: req.surface,
            elementPath: await ctx.page.activePath(),
            elementName: focusText || null,
            role: node?.role ?? null,
            whatUserExperiences: `Screen reader does not announce "${assertion.expectedAnnouncement}" after this flow`,
            why: `Flow requirement ${req.id} expects the announcement to include "${assertion.expectedAnnouncement}"; neither the focused element nor any live region carried it`,
            fix: 'Render the text into a live region that exists before the update, or onto the focused element',
            evidence: { name: { value: focusText || null, source: 'ax-tree', fromTree: true } },
            confidence: 'fail',
          }];
        }
      }
      return [];
    },
  };
}

export function mapRequirementsToProviders(bundle: RequirementBundle): Provider[] {
  const providers: Provider[] = [];
  for (const req of bundle.requirements) {
    if (req.assertion.type === 'content') {
      providers.push(makeContentProvider(req, req.assertion));
    } else if (req.assertion.type === 'flow') {
      providers.push(makeFlowProvider(req, req.assertion));
    }
    // 'doc' assertions feed the output module, not runtime checks: no provider emitted
  }
  return providers;
}
```

- [ ] **4. Run: expect PASS**

```sh
npx vitest run test/intake/map-to-providers.test.ts
```

- [ ] **5. Commit**

```sh
git add src/intake/map-to-providers.ts test/intake/map-to-providers.test.ts
git commit -m "feat(intake): map requirements to deterministic Providers; doc reqs emit no provider"
```

---

## Task 4: Alt-text manifest generator

**Files:**
- `src/docs/alt-text-manifest.ts`
- `test/docs/alt-text-manifest.test.ts`

Pure function of `Result + Receipt + RequirementBundle + surface -> DocArtifact`. Entries are derived from `content` requirements whose selector implies an image or labelled element. `approved: true` in the requirement → `status: 'approved'` in the entry. Receipt backing gives `boundToReceipt` and per-entry `evidenceRef`.

### Steps

- [ ] **1. Write failing test**

```ts
// test/docs/alt-text-manifest.test.ts
import { describe, it, expect } from 'vitest';
import { generateAltTextManifest } from '../../src/docs/alt-text-manifest.js';
import type { Result, Receipt, RequirementBundle, Coverage } from '../../src/contracts/index.js';

const baseCoverage: Coverage = {
  changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false,
};

const baseResult: Result = {
  schemaVersion: 'usabl.result.v1',
  verdict: 'verified',
  summary: '',
  screens: [],
  coverage: baseCoverage,
  findings: [],
  receipt: null,
  dirtyGuardedPaths: [],
  exitCode: 0,
};

const baseReceipt: Receipt = {
  schemaVersion: 1,
  sourceTree: 'abc123treehash',
  baseRevision: null,
  policyHash: 'ph1',
  runnerVersion: '0.0.0-test',
  scannerVersions: { axeCore: '4.10.0', playwright: '1.45.0', chromium: '126' },
  surfaces: ['clusters'],
  coverage: { checked: ['clusters'], notCovered: [] },
  verdict: 'verified',
  findingsSummary: { new: 0, carried: 0, fixed: 0, unverified: 0 },
  activeWaivers: 0,
  mintedAt: '2026-08-20T00:00:00.000Z',
};

const singleContentBundle: RequirementBundle = {
  version: 1,
  requirements: [{
    id: 'req-alt-01', kind: 'content', surface: 'clusters',
    description: 'Topology SVG alt text', approved: true,
    assertion: { type: 'content', selector: 'img.topology-svg', expectedText: 'Cluster network topology showing three nodes' },
  }],
};

describe('generateAltTextManifest', () => {
  it('emits an approved entry with evidenceRef when requirement is approved and receipt is present', () => {
    const artifact = generateAltTextManifest(baseResult, baseReceipt, singleContentBundle, 'clusters');
    expect(artifact.kind).toBe('alt-text-manifest');
    expect(artifact.surface).toBe('clusters');
    expect(artifact.boundToReceipt).toBe('abc123treehash');
    expect(artifact.entries).toHaveLength(1);
    expect(artifact.entries[0].status).toBe('approved');
    expect(artifact.entries[0].content).toBe('Cluster network topology showing three nodes');
    expect(artifact.entries[0].evidenceRef).toContain('abc123treehash');
    expect(artifact.entries[0].evidenceRef).toContain('clusters');
  });

  it('emits a draft entry when requirement is not approved', () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [{
        id: 'req-alt-02', kind: 'content', surface: 'clusters',
        description: 'Close icon label', approved: false,
        assertion: { type: 'content', selector: 'button.close', expectedText: 'Close dialog' },
      }],
    };
    const artifact = generateAltTextManifest(baseResult, baseReceipt, bundle, 'clusters');
    expect(artifact.entries[0].status).toBe('draft');
  });

  it('omits boundToReceipt and evidenceRef when receipt is null', () => {
    const artifact = generateAltTextManifest(baseResult, null, singleContentBundle, 'clusters');
    expect(artifact.boundToReceipt).toBeUndefined();
    expect(artifact.entries[0].evidenceRef).toBeUndefined();
    expect(artifact.generatedAt).toBe(''); // no receipt, no honest time source
  });

  it('omits evidenceRef when the receipt did NOT cover this surface (never fabricated)', () => {
    const uncovered = { ...baseReceipt, coverage: { checked: ['hosts'], notCovered: ['clusters'] } };
    const artifact = generateAltTextManifest(baseResult, uncovered, singleContentBundle, 'clusters');
    expect(artifact.entries[0].evidenceRef).toBeUndefined();
    expect(artifact.boundToReceipt).toBe('abc123treehash'); // binding still recorded; evidence is not claimed
  });

  it('skips requirements for other surfaces', () => {
    const bundle: RequirementBundle = {
      version: 1,
      requirements: [
        {
          id: 'req-a', kind: 'content', surface: 'clusters',
          description: 'Clusters image', approved: true,
          assertion: { type: 'content', selector: 'img.a', expectedText: 'A' },
        },
        {
          id: 'req-b', kind: 'content', surface: 'hosts',
          description: 'Hosts image', approved: true,
          assertion: { type: 'content', selector: 'img.b', expectedText: 'B' },
        },
      ],
    };
    const artifact = generateAltTextManifest(baseResult, baseReceipt, bundle, 'clusters');
    expect(artifact.entries).toHaveLength(1);
    expect(artifact.entries[0].element).toBe('img.a');
  });
});
```

- [ ] **2. Run: expect FAIL**

```sh
npx vitest run test/docs/alt-text-manifest.test.ts
```

- [ ] **3. Implement `src/docs/alt-text-manifest.ts`**

```ts
import type { Result, Receipt, RequirementBundle, DocArtifact, Finding, ContentAssertion } from '../contracts/index.js';

function findingEvidenceRef(f: Finding): string {
  return `receipt:${f.screenId}:finding:${f.elementPath}`;
}

function receiptEvidenceRef(sourceTree: string, screenId: string, selector: string): string {
  return `receipt:${sourceTree}:${screenId}:${selector}`;
}

export function generateAltTextManifest(
  result: Result,
  receipt: Receipt | null,
  bundle: RequirementBundle,
  surface: string,
): DocArtifact {
  const contentReqs = bundle.requirements.filter(
    r => r.surface === surface && r.assertion.type === 'content' && (r.assertion as ContentAssertion).expectedText !== undefined,
  );

  const entries: DocArtifact['entries'] = contentReqs.map(req => {
    const assertion = req.assertion as ContentAssertion;
    const selector = assertion.selector;
    const expectedText = assertion.expectedText!;

    // Look for a finding from this requirement's rule on this surface
    const backing = result.findings.find(
      f => f.screenId === surface && f.rule === `intake:${req.id}` && f.elementPath === selector,
    );

    // HONESTY RULE: an evidenceRef exists only when the receipt covers this surface.
    // A requirement on a surface the run never checked gets NO ref, never a fabricated one.
    const covered = receipt !== null && receipt.coverage.checked.includes(surface);
    let evidenceRef: string | undefined;
    if (covered) {
      evidenceRef = backing
        ? findingEvidenceRef(backing)
        : receiptEvidenceRef(receipt!.sourceTree, surface, selector);
    }

    return {
      element: selector,
      content: expectedText,
      status: req.approved ? 'approved' : 'draft',
      evidenceRef,
    };
  });

  return {
    kind: 'alt-text-manifest',
    surface,
    entries,
    // No receipt means no honest time source; the empty string says so.
    generatedAt: receipt?.mintedAt ?? '',
    boundToReceipt: receipt?.sourceTree,
  };
}
```

- [ ] **4. Run: expect PASS**

```sh
npx vitest run test/docs/alt-text-manifest.test.ts
```

- [ ] **5. Commit**

```sh
git add src/docs/alt-text-manifest.ts test/docs/alt-text-manifest.test.ts
git commit -m "feat(docs): generateAltTextManifest() bound to receipt evidenceRef"
```

---

## Task 5: Announcement snippets generator

**Files:**
- `src/docs/announcement-snippets.ts`
- `test/docs/announcement-snippets.test.ts`

Pure function of `Result + Receipt -> DocArtifact[]`. One artifact per surface. Each entry represents one `TranscriptStop`: what a screen reader would announce at that interactive stop. Entries are always `status: 'draft'` (AI-generated; must be approved in the requirement bundle before publishing). `evidenceRef` encodes the stop index so callers can trace back to the exact transcript.

### Steps

- [ ] **1. Write failing test**

```ts
// test/docs/announcement-snippets.test.ts
import { describe, it, expect } from 'vitest';
import { generateAnnouncementSnippets } from '../../src/docs/announcement-snippets.js';
import type { Result, Receipt, Coverage, ScreenScan } from '../../src/contracts/index.js';

const baseCoverage: Coverage = {
  changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false,
};

const baseReceipt: Receipt = {
  schemaVersion: 1,
  sourceTree: 'treehash456',
  baseRevision: null,
  policyHash: 'ph1',
  runnerVersion: '0.0.0-test',
  scannerVersions: { axeCore: '4.10.0', playwright: '1.45.0', chromium: '126' },
  surfaces: ['clusters'],
  coverage: { checked: ['clusters'], notCovered: [] },
  verdict: 'verified',
  findingsSummary: { new: 0, carried: 0, fixed: 0, unverified: 0 },
  activeWaivers: 0,
  mintedAt: '2026-08-20T00:00:00.000Z',
};

const scanWithStops: ScreenScan = {
  screenId: 'clusters',
  url: 'http://localhost:3000/clusters',
  stops: [
    {
      index: 0,
      elementPath: 'button.create-cluster',
      announcement: [
        { kind: 'name', text: 'Create cluster', fromTree: true, source: 'ax-tree' },
        { kind: 'role', text: 'button', fromTree: true, source: 'ax-tree' },
      ],
    },
    {
      index: 1,
      elementPath: 'a.cluster-link',
      announcement: [
        { kind: 'name', text: 'my-cluster', fromTree: true, source: 'ax-tree' },
        { kind: 'role', text: 'link', fromTree: true, source: 'ax-tree' },
      ],
    },
  ],
  drafts: [],
  gaps: [],
};

function makeResult(screens: ScreenScan[]): Result {
  return {
    schemaVersion: 'usabl.result.v1',
    verdict: 'verified',
    summary: '',
    screens,
    coverage: baseCoverage,
    findings: [],
    receipt: null,
    dirtyGuardedPaths: [],
    exitCode: 0,
  };
}

describe('generateAnnouncementSnippets', () => {
  it('produces one artifact per surface with one entry per stop', () => {
    const artifacts = generateAnnouncementSnippets(makeResult([scanWithStops]), baseReceipt);
    expect(artifacts).toHaveLength(1);
    expect(artifacts[0].kind).toBe('announcement-snippets');
    expect(artifacts[0].surface).toBe('clusters');
    expect(artifacts[0].entries).toHaveLength(2);
    expect(artifacts[0].entries[0].element).toBe('button.create-cluster');
    expect(artifacts[0].entries[0].content).toContain('Create cluster');
  });

  it('each entry carries an evidenceRef containing the sourceTree and stop index', () => {
    const artifacts = generateAnnouncementSnippets(makeResult([scanWithStops]), baseReceipt);
    expect(artifacts[0].entries[0].evidenceRef).toBe('stop:treehash456:clusters:0');
    expect(artifacts[0].entries[1].evidenceRef).toBe('stop:treehash456:clusters:1');
    expect(artifacts[0].boundToReceipt).toBe('treehash456');
  });

  it('entries are status draft (AI-generated, not auto-approved)', () => {
    const artifacts = generateAnnouncementSnippets(makeResult([scanWithStops]), baseReceipt);
    for (const e of artifacts[0].entries) {
      expect(e.status).toBe('draft');
    }
  });

  it('produces an empty entries array when a surface has no stops', () => {
    const emptyScan: ScreenScan = { screenId: 'hosts', url: 'http://x/hosts', stops: [], drafts: [], gaps: [] };
    const artifacts = generateAnnouncementSnippets(makeResult([emptyScan]), baseReceipt);
    expect(artifacts[0].entries).toHaveLength(0);
  });
});
```

- [ ] **2. Run: expect FAIL**

```sh
npx vitest run test/docs/announcement-snippets.test.ts
```

- [ ] **3. Implement `src/docs/announcement-snippets.ts`**

```ts
import type { Result, Receipt, DocArtifact } from '../contracts/index.js';

export function generateAnnouncementSnippets(result: Result, receipt: Receipt | null): DocArtifact[] {
  return result.screens.map(scan => {
    const entries: DocArtifact['entries'] = scan.stops.map(stop => {
      const content = stop.announcement
        .map(t => t.text ?? '')
        .filter(t => t.length > 0)
        .join(', ');

      const covered = receipt !== null && receipt.coverage.checked.includes(scan.screenId);
      const evidenceRef = covered
        ? `stop:${receipt!.sourceTree}:${scan.screenId}:${stop.index}`
        : undefined;

      return {
        element: stop.elementPath,
        content,
        status: 'draft' as const,
        evidenceRef,
      };
    });

    return {
      kind: 'announcement-snippets' as const,
      surface: scan.screenId,
      entries,
      generatedAt: receipt?.mintedAt ?? '',
      boundToReceipt: receipt?.sourceTree,
    };
  });
}
```

- [ ] **4. Run: expect PASS**

```sh
npx vitest run test/docs/announcement-snippets.test.ts
```

- [ ] **5. Commit**

```sh
git add src/docs/announcement-snippets.ts test/docs/announcement-snippets.test.ts
git commit -m "feat(docs): generateAnnouncementSnippets() bound to stop evidenceRef"
```

---

## Task 6: Keyboard paths generator

**Files:**
- `src/docs/keyboard-paths.ts`
- `test/docs/keyboard-paths.test.ts`

Pure function of `Result + Receipt -> DocArtifact[]`. One artifact per surface. Each entry is a Tab-order step showing the element path (prefixed with its 1-based ordinal), the full announcement, and a stable `evidenceRef`. Status is always `'draft'`: the path is a generated snapshot, not an authored contract.

### Steps

- [ ] **1. Write failing test**

```ts
// test/docs/keyboard-paths.test.ts
import { describe, it, expect } from 'vitest';
import { generateKeyboardPaths } from '../../src/docs/keyboard-paths.js';
import type { Result, Receipt, Coverage, ScreenScan } from '../../src/contracts/index.js';

const baseCoverage: Coverage = {
  changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false,
};

const baseReceipt: Receipt = {
  schemaVersion: 1,
  sourceTree: 'kp-tree-789',
  baseRevision: null,
  policyHash: 'ph1',
  runnerVersion: '0.0.0-test',
  scannerVersions: { axeCore: '4.10.0', playwright: '1.45.0', chromium: '126' },
  surfaces: ['clusters'],
  coverage: { checked: ['clusters'], notCovered: [] },
  verdict: 'verified',
  findingsSummary: { new: 0, carried: 0, fixed: 0, unverified: 0 },
  activeWaivers: 0,
  mintedAt: '2026-08-20T00:00:00.000Z',
};

const scan: ScreenScan = {
  screenId: 'clusters',
  url: 'http://localhost:3000/clusters',
  stops: [
    {
      index: 0,
      elementPath: 'button.create-cluster',
      announcement: [
        { kind: 'name', text: 'Create cluster', fromTree: true, source: 'ax-tree' },
        { kind: 'role', text: 'button', fromTree: true, source: 'ax-tree' },
        { kind: 'state', text: null, fromTree: true, source: 'ax-tree' },
      ],
    },
  ],
  drafts: [],
  gaps: [],
};

function makeResult(screens: ScreenScan[]): Result {
  return {
    schemaVersion: 'usabl.result.v1',
    verdict: 'verified',
    summary: '',
    screens,
    coverage: baseCoverage,
    findings: [],
    receipt: null,
    dirtyGuardedPaths: [],
    exitCode: 0,
  };
}

describe('generateKeyboardPaths', () => {
  it('produces one artifact per surface with Tab-order entries', () => {
    const artifacts = generateKeyboardPaths(makeResult([scan]), baseReceipt);
    expect(artifacts).toHaveLength(1);
    expect(artifacts[0].kind).toBe('keyboard-paths');
    expect(artifacts[0].surface).toBe('clusters');
    expect(artifacts[0].entries).toHaveLength(1);
    expect(artifacts[0].entries[0].element).toContain('[1]');
    expect(artifacts[0].entries[0].element).toContain('button.create-cluster');
  });

  it('entry content encodes name:role:state tokens', () => {
    const artifacts = generateKeyboardPaths(makeResult([scan]), baseReceipt);
    const content = artifacts[0].entries[0].content;
    expect(content).toContain('name:Create cluster');
    expect(content).toContain('role:button');
  });

  it('each entry carries a stable stop evidenceRef', () => {
    const artifacts = generateKeyboardPaths(makeResult([scan]), baseReceipt);
    expect(artifacts[0].entries[0].evidenceRef).toBe('stop:kp-tree-789:clusters:0');
    expect(artifacts[0].boundToReceipt).toBe('kp-tree-789');
  });

  it('all entries are status draft', () => {
    const artifacts = generateKeyboardPaths(makeResult([scan]), baseReceipt);
    for (const e of artifacts[0].entries) {
      expect(e.status).toBe('draft');
    }
  });
});
```

- [ ] **2. Run: expect FAIL**

```sh
npx vitest run test/docs/keyboard-paths.test.ts
```

- [ ] **3. Implement `src/docs/keyboard-paths.ts`**

```ts
import type { Result, Receipt, DocArtifact } from '../contracts/index.js';

export function generateKeyboardPaths(result: Result, receipt: Receipt | null): DocArtifact[] {
  return result.screens.map(scan => {
    const entries: DocArtifact['entries'] = scan.stops.map((stop, idx) => {
      const tokens = stop.announcement
        .map(t => `${t.kind}:${t.text ?? 'null'}`)
        .join(' | ');

      const covered = receipt !== null && receipt.coverage.checked.includes(scan.screenId);
      const evidenceRef = covered
        ? `stop:${receipt!.sourceTree}:${scan.screenId}:${stop.index}`
        : undefined;

      return {
        element: `[${idx + 1}] ${stop.elementPath}`,
        content: tokens,
        status: 'draft' as const,
        evidenceRef,
      };
    });

    return {
      kind: 'keyboard-paths' as const,
      surface: scan.screenId,
      entries,
      generatedAt: receipt?.mintedAt ?? '',
      boundToReceipt: receipt?.sourceTree,
    };
  });
}
```

- [ ] **4. Run: expect PASS**

```sh
npx vitest run test/docs/keyboard-paths.test.ts
```

- [ ] **5. Commit**

```sh
git add src/docs/keyboard-paths.ts test/docs/keyboard-paths.test.ts
git commit -m "feat(docs): generateKeyboardPaths() with Tab-order entries and stop evidenceRef"
```

---

## Task 7: Integration: exit criterion verification

**Files:**
- `test/intake/integration.test.ts`

Exercises the three exit criterion invariants end-to-end using in-memory fakes: (1) an authored requirement maps to a check; (2) a generated artifact carries an `evidenceRef` that resolves to real receipt evidence; (3) malformed intake yields `approval_required`.

### Steps

- [ ] **1. Write failing test**

```ts
// test/intake/integration.test.ts
import { describe, it, expect } from 'vitest';
import { normalize } from '../../src/intake/normalize.js';
import { mapRequirementsToProviders } from '../../src/intake/map-to-providers.js';
import { generateAltTextManifest } from '../../src/docs/alt-text-manifest.js';
import { generateAnnouncementSnippets } from '../../src/docs/announcement-snippets.js';
import { generateKeyboardPaths } from '../../src/docs/keyboard-paths.js';
import { makeFakePage } from '../../src/deps/fakes.js';
import type { Result, Receipt, Coverage, ScreenScan, UsablConfig, ProviderContext, Page } from '../../src/contracts/index.js';

const baseCoverage: Coverage = {
  changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false,
};

const receipt: Receipt = {
  schemaVersion: 1,
  sourceTree: 'integration-tree-abc',
  baseRevision: null,
  policyHash: 'ph1',
  runnerVersion: '0.0.0-test',
  scannerVersions: { axeCore: '4.10.0', playwright: '1.45.0', chromium: '126' },
  surfaces: ['clusters'],
  coverage: { checked: ['clusters'], notCovered: [] },
  verdict: 'verified',
  findingsSummary: { new: 0, carried: 0, fixed: 0, unverified: 0 },
  activeWaivers: 0,
  mintedAt: '2026-08-20T00:00:00.000Z',
};

const scanWithStops: ScreenScan = {
  screenId: 'clusters',
  url: 'http://localhost:3000/clusters',
  stops: [
    {
      index: 0,
      elementPath: 'button.create-cluster',
      announcement: [
        { kind: 'name', text: 'Create cluster', fromTree: true, source: 'ax-tree' },
        { kind: 'role', text: 'button', fromTree: true, source: 'ax-tree' },
      ],
    },
  ],
  drafts: [],
  gaps: [],
};

const result: Result = {
  schemaVersion: 'usabl.result.v1',
  verdict: 'verified',
  summary: '',
  screens: [scanWithStops],
  coverage: baseCoverage,
  findings: [],
  receipt,
  dirtyGuardedPaths: [],
  exitCode: 0,
};

describe('Phase 6 integration', () => {
  it('authored requirement maps to a check (provider)', () => {
    const yaml = `
version: 1
requirements:
  - id: req-int-01
    kind: content
    surface: clusters
    description: Topology alt text
    approved: true
    assertion:
      type: content
      selector: img.topology-svg
      expectedText: "Cluster network topology"
`;
    const parseResult = normalize(yaml, 'yaml');
    expect(parseResult.ok).toBe(true);
    if (!parseResult.ok) return;

    const providers = mapRequirementsToProviders(parseResult.bundle);
    expect(providers).toHaveLength(1);
    expect(providers[0].id).toBe('intake:req-int-01');
    expect(providers[0].layer).toBe('intake-content');
    expect(providers[0].capabilities).toContain('live');
  });

  it('generated alt-text artifact carries an evidenceRef that contains the receipt sourceTree', () => {
    const bundle = normalize(`
version: 1
requirements:
  - id: req-int-02
    kind: content
    surface: clusters
    description: Logo alt text
    approved: true
    assertion:
      type: content
      selector: img.logo
      expectedText: "Red Hat OpenShift"
`, 'yaml');
    expect(bundle.ok).toBe(true);
    if (!bundle.ok) return;

    const artifact = generateAltTextManifest(result, receipt, bundle.bundle, 'clusters');
    expect(artifact.boundToReceipt).toBe('integration-tree-abc');
    expect(artifact.entries[0].evidenceRef).toContain('integration-tree-abc');
  });

  it('announcement snippet and keyboard path artifacts carry evidenceRef tied to receipt stops', () => {
    const snippets = generateAnnouncementSnippets(result, receipt);
    const paths = generateKeyboardPaths(result, receipt);

    expect(snippets[0].boundToReceipt).toBe('integration-tree-abc');
    expect(snippets[0].entries[0].evidenceRef).toBe('stop:integration-tree-abc:clusters:0');

    expect(paths[0].boundToReceipt).toBe('integration-tree-abc');
    expect(paths[0].entries[0].evidenceRef).toBe('stop:integration-tree-abc:clusters:0');
  });

  it('malformed intake YAML yields approval_required: never a silent pass or default', () => {
    const r1 = normalize('not: yaml: at: all: [unclosed', 'yaml');
    expect(r1.ok).toBe(false);
    if (!r1.ok) expect(r1.verdict).toBe('approval_required');

    const r2 = normalize('version: 1\nrequirements:\n  - id: only-id', 'yaml');
    expect(r2.ok).toBe(false);
    if (!r2.ok) expect(r2.verdict).toBe('approval_required');

    // kind/assertion.type mismatch → approval_required
    const r3 = normalize(`
version: 1
requirements:
  - id: bad
    kind: content
    surface: s1
    description: d
    approved: true
    assertion:
      type: flow
      steps: []
`, 'yaml');
    expect(r3.ok).toBe(false);
    if (!r3.ok) expect(r3.verdict).toBe('approval_required');
  });

  it('content provider run() on a fake page produces a deterministic Draft on mismatch', async () => {
    const bundle = normalize(`
version: 1
requirements:
  - id: req-int-03
    kind: content
    surface: clusters
    description: Topology alt text
    approved: true
    assertion:
      type: content
      selector: img.topology-svg
      expectedText: "Correct text"
`, 'yaml');
    expect(bundle.ok).toBe(true);
    if (!bundle.ok) return;

    const providers = mapRequirementsToProviders(bundle.bundle);
    const fakePage: Page = makeFakePage({
      activePath: async () => 'img.topology-svg',
      axAt: async () => ({ name: 'Wrong text entirely', role: 'img', states: {} }),
    });
    const fakeConfig: UsablConfig = {
      appBaseUrl: 'http://localhost:3000',
      uiFileGlobs: [],
      discovery: { routerFile: '', wideBlastGlobs: [] },
      surfaces: [{ id: 'clusters', url: 'http://localhost:3000/clusters', files: [] }],
      guardedPaths: [],
    };
    const ctx: ProviderContext = {
      page: fakePage,
      screen: { id: 'clusters', url: 'http://localhost:3000/clusters' },
      config: fakeConfig,
    };
    const drafts = await providers[0].run(ctx);
    expect(drafts).toHaveLength(1);
    expect(drafts[0].evidenceClass).toBe('deterministic');
    expect(drafts[0].confidence).toBe('fail');
  });
});
```

- [ ] **2. Run: expect FAIL** (modules exist but integration wiring is being tested end-to-end)

```sh
npx vitest run test/intake/integration.test.ts
```

- [ ] **3. Fix any integration gaps** (typically none if Tasks 1–6 passed)

Run the full suite to confirm nothing regressed:

```sh
npx vitest run test/intake/ test/docs/
```

- [ ] **4. Run integration test: expect PASS**

```sh
npx vitest run test/intake/integration.test.ts
```

- [ ] **5. Commit**

```sh
git add test/intake/integration.test.ts
git commit -m "test(intake): integration: requirement maps to check, artifact carries evidenceRef, malformed yields approval_required"
```

---

## Task 8: Wire intake into the engine

**Files:**
- Create: `src/intake/load.ts`
- Modify: the CLI's `buildDeps` wiring (provider list construction)
- Test: `test/intake/load.test.ts`

Without this task the intake is a library nothing calls. With it:

1. `loadRequirements(fs, config)`: when `config.requirements` is set, glob
   `<dir>/**/*.yaml` + `<dir>/**/*.yml` through `deps.fs`, `normalize()` each file, and
   merge the bundles. ANY file that fails to parse returns
   `{ ok: false, verdict: 'approval_required', reason }` for the whole load: a broken
   requirement file is a policy problem, never a skipped file.
2. The CLI's CheckRunner construction appends `mapRequirementsToProviders(bundle)` to
   the provider list (after axe/rulepack/walk, before the voicing provider). Intake
   providers are surface-scoped by their own `run()` guards.
3. A failed load surfaces as `approval_required` through the normal gate path: the
   caller converts `{ ok: false }` into a guard-diverged-style blocked Result before
   any browser work (same treatment as a diverged policy file).
4. The requirements directory is already in the guarded set (Phase 3
   `buildGuardedSet` includes `config.requirements` unconditionally), so editing a
   requirement without committing it blocks as `approval_required`. No extra work,
   but Step 1's test asserts it end to end.

- [ ] **Step 1: Write the failing test**

```ts
// test/intake/load.test.ts
import { describe, it, expect } from 'vitest';
import { loadRequirements } from '../../src/intake/load.js';
import { buildGuardedSet } from '../../src/trust/guard.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://localhost:3000', uiFileGlobs: [],
  discovery: { routerFile: '', wideBlastGlobs: [] }, surfaces: [],
  guardedPaths: [], requirements: 'requirements/',
};

const goodYaml = `
version: 1
requirements:
  - id: req-01
    kind: content
    surface: clusters
    description: Topology alt text
    approved: true
    assertion:
      type: content
      selector: img.topology-svg
      expectedText: "Cluster network topology"
`;

describe('loadRequirements', () => {
  it('globs the requirements dir and merges bundles', async () => {
    const deps = makeFakeDeps({ files: { 'requirements/clusters.yaml': goodYaml } });
    const r = await loadRequirements(deps.fs, config);
    expect(r.ok).toBe(true);
    if (r.ok) expect(r.bundle.requirements).toHaveLength(1);
  });

  it('returns ok with an empty bundle when no requirements dir is configured', async () => {
    const deps = makeFakeDeps({});
    const r = await loadRequirements(deps.fs, { ...config, requirements: undefined });
    expect(r.ok).toBe(true);
    if (r.ok) expect(r.bundle.requirements).toHaveLength(0);
  });

  it('ANY malformed file fails the whole load with approval_required', async () => {
    const deps = makeFakeDeps({ files: {
      'requirements/good.yaml': goodYaml,
      'requirements/bad.yaml': 'version: 1\nrequirements:\n  - id: only-id',
    } });
    const r = await loadRequirements(deps.fs, config);
    expect(r.ok).toBe(false);
    if (!r.ok) {
      expect(r.verdict).toBe('approval_required');
      expect(r.reason).toContain('bad.yaml');
    }
  });

  it('the requirements dir sits in the guarded set without any extra wiring', () => {
    expect(buildGuardedSet(config)).toContain('requirements/');
  });
});
```

- [ ] **Step 2: Run: expect FAIL. Step 3: Implement `src/intake/load.ts`** (glob,
  read, `normalize` each, merge `requirements` arrays, first failure wins with the
  file path prefixed to the reason). Then extend the CLI wiring to append
  `mapRequirementsToProviders(bundle)` to the provider list and to convert a failed
  load into the blocked `approval_required` Result before any scan.

- [ ] **Step 4: Run: expect PASS**, then the full suite.
- [ ] **Step 5: Commit** `feat(intake): load requirement bundles from the guarded dir and wire providers into the engine`

---

## Self-Review

Before closing this phase, verify:

**Intake normalize/schema/guard**
- [ ] `parseBundle` rejects every invalid shape with `verdict: 'approval_required'`: no silent defaults, no partial results.
- [ ] `normalize()` handles YAML parse errors (try/catch), schema failures (Zod), and kind/assertion.type mismatches (superRefine): all three resolve to `approval_required`.
- [ ] `mapRequirementsToProviders` emits no provider for `doc` requirements; only `content` and `flow` produce runtime checks.
- [ ] All intake-derived Drafts carry `evidenceClass: 'deterministic'` per ground-truth §13.

**Docs artifacts and evidenceRef binding**
- [ ] All three generators (`generateAltTextManifest`, `generateAnnouncementSnippets`, `generateKeyboardPaths`) are pure functions; no I/O, no side effects, no `new Date()` (`generatedAt` is `receipt.mintedAt` or the empty string).
- [ ] `evidenceRef` is emitted ONLY when the receipt exists AND `receipt.coverage.checked` includes the surface. A surface the run never checked gets no ref. Never fabricated.
- [ ] No entry is emitted with `status: 'approved'` unless `requirement.approved === true`.
- [ ] `boundToReceipt` is `undefined` (not `null`, not `''`) when `receipt` is `null`.

**Engine wiring (Task 8)**
- [ ] `loadRequirements` merges every YAML bundle under `config.requirements`; one malformed file fails the whole load as `approval_required` with the file named in the reason.
- [ ] Intake providers ride the same CheckRunner provider list; the requirements dir is in the guarded set via Phase 3's `buildGuardedSet`.
- [ ] `schemaVersion` is carried via the `DocArtifact` type (enforced by TypeScript, not duplicated at runtime).

**Placeholder / TBD scan**
- [ ] Search `src/intake/ src/docs/` for `TODO`, `FIXME`, `any`, `as any`, `TBD`: address or document each occurrence.
- [ ] Note: `as any` in `makeContentProvider`/`makeFlowProvider` casts are acceptable only if TypeScript's discriminated-union narrowing of `Requirement.assertion` cannot be widened further without changing the frozen contract.

**Type consistency with frozen contracts**
- [ ] `Draft` fields match the Phase 1 frozen interface exactly: `rule`, `layer`, `severity`, `evidenceClass`, `screenId`, `elementPath`, `elementName`, `role`, `whatUserExperiences`, `why`, `fix`, `evidence`, `confidence`.
- [ ] `RequirementBundle`, `Requirement`, `DocArtifact` are imported from `src/contracts/index.ts`: not re-declared.
- [ ] `Provider`, `ProviderContext`, `Capability` are imported from `src/contracts/index.ts` (added by Phase 2 per the frozen seam in `00-plan-set.md`).
- [ ] `Receipt.sourceTree` (the tree hash string) is the canonical key for `boundToReceipt` and `evidenceRef`: not a running ID, not a timestamp.
