# Surfaces Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire every surface (CLI, stop hook, CI/PR comment, Vite overlay, Playwright helper, mid-task self-check) so that a single `Result` from `run(deps, config)` drives the same verdict and exit code everywhere. No surface mints a verdict; no surface re-derives findings; secrets never appear in any projection.

**Architecture:**
- `run(deps, config)` is the single orchestrator. Every surface calls it (or consumes its output).
- `scrubResult(result)` is called before any external serialization. All page-derived text that reaches a projection first passes through `neutralize()`.
- Exit codes are fixed across all surfaces: 0 verified/idle, 1 regression, 2 approval_required, 3 not_covered, 4 unhandled error (fail open), 5 reserved.
- A surface that cannot be fully exercised emits `not_covered` (exit 3), never a silent pass.
- Ambiguous input resolves to `approval_required` (exit 2), never a warning.
- Every projection object carries `schemaVersion: 'usabl.result.v1'` from the `Result` it wraps.
- The stop hook is the gate; the Vite overlay and mid-task self-check are advisory only.

**Tech Stack:**
- TypeScript (ESM, strict), Node 22.
- Vitest for all tests. Unit tests use canned `Result` fixtures (no network, no real browser).
- tsup builds `dist/`; `usabl` bin points to `dist/cli.js`.
- GitHub Action pinned to SHA refs.
- Vite plugin ships as `usabl/vite` subpath export.
- Playwright helper ships as `usabl/playwright` subpath export.

---

## Task 1: Secret scrubber

**Files:**
- Create: `src/surfaces/scrub.ts`
- Test: `tests/surfaces/scrub.test.ts`

All surfaces call `scrubResult()` before serializing. `neutralize()` is also the egress guard for PR comments and MCP output.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/scrub.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { neutralize, scrubResult } from '../../src/surfaces/scrub.js';
import type { Result } from '../../src/contracts/index.js';

const base = (): Result => ({
  schemaVersion: 'usabl.result.v1',
  verdict: 'verified', summary: 'verified: 0 finding(s)', screens: [],
  coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0,
});

describe('neutralize', () => {
  it('redacts Authorization headers', () => {
    expect(neutralize('Authorization: Bearer secret123')).toContain('[REDACTED]');
    expect(neutralize('Authorization: Bearer secret123')).not.toContain('secret123');
  });

  it('redacts storageState references', () => {
    expect(neutralize('storageState: {"cookies":[{"name":"session","value":"tok"}]}')).toContain('[REDACTED]');
  });

  it('passes through non-sensitive text unchanged', () => {
    expect(neutralize('verified: 0 new findings')).toBe('verified: 0 new findings');
  });
});

describe('scrubResult', () => {
  it('preserves schemaVersion and verdict', () => {
    const r = base();
    const s = scrubResult(r);
    expect(s.schemaVersion).toBe('usabl.result.v1');
    expect(s.verdict).toBe('verified');
    expect(s.exitCode).toBe(0);
  });

  it('removes credential-like strings from evidence fields', () => {
    const r = base();
    r.findings = [{
      rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
      screenId: 'screen1', elementPath: 'button', elementName: 'Save', role: 'button',
      whatUserExperiences: 'Low contrast', why: 'token=abc123secret', fix: 'Raise contrast',
      evidence: { storageState: 'do-not-leak' }, confidence: 'fail',
      elementKey: 'k', identityBasis: 'name', status: 'new',
    }];
    const s = scrubResult(r);
    const json = JSON.stringify(s);
    expect(json).not.toContain('abc123secret');
    expect(json).not.toContain('do-not-leak');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/scrub.test.ts
```
Expected: FAIL — module not found.

- [ ] **Step 3: Implement `src/surfaces/scrub.ts`**

```ts
import type { Result } from '../contracts/index.js';

const REDACT_PATTERNS: RegExp[] = [
  /storageState[^\s"']*/gi,
  /Authorization:\s*\S+/gi,
  /token[=:]\s*["']?[A-Za-z0-9+/=_\-\.]{8,}["']?/gi,
  /password[=:]\s*["']?\S+["']?/gi,
  /secret[=:]\s*["']?\S+["']?/gi,
  /"storageState"\s*:\s*"[^"]*"/gi,
];

/** Replace credential-like strings with [REDACTED]. Safe to call multiple times. */
export function neutralize(text: string): string {
  let out = text;
  for (const pat of REDACT_PATTERNS) out = out.replace(pat, '[REDACTED]');
  return out;
}

/**
 * Return a scrubbed deep copy of Result safe for any surface egress.
 * Strips storageState, scrubs credential-pattern strings from all fields.
 * schemaVersion, verdict, and exitCode are preserved verbatim.
 */
export function scrubResult(result: Result): Result {
  return JSON.parse(neutralize(JSON.stringify(result))) as Result;
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/scrub.test.ts && npm run typecheck
```

- [ ] **Step 5: Commit**

```bash
git add src/surfaces/scrub.ts tests/surfaces/scrub.test.ts
git commit -m "feat(surfaces): add neutralize/scrubResult secret scrubber"
```

---

## Task 2: Full CLI surface

**Files:**
- Create: `src/surfaces/cli.ts`
- Test: `tests/surfaces/cli.test.ts`

Extends the Phase 1 thin CLI with `--static-only` (capability filtering), `--trusted-ref`, and `--json`. The CLI is a pure adapter: parse args, build effective deps, call `run()`, scrub, project, exit.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/cli.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { parseCliArgs, projectCli } from '../../src/surfaces/cli.js';
import type { Result } from '../../src/contracts/index.js';

const regression = (): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'regression', summary: 'regression: 1 new finding',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [{
    rule: 'button-name', layer: 'axe', severity: 'critical', evidenceClass: 'deterministic',
    screenId: 'nav', elementPath: 'button:nth-of-type(1)', elementName: null, role: 'button',
    whatUserExperiences: 'Icon button has no accessible name', why: 'aria-label absent',
    fix: 'Add aria-label or visible label', evidence: {}, confidence: 'fail',
    elementKey: 'nav|axe/button-name', identityBasis: 'structural', status: 'new',
  }],
  receipt: null, dirtyGuardedPaths: [], exitCode: 1,
});

const notCovered = (): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'not_covered', summary: 'not_covered: gaps',
  screens: [], coverage: { changedFiles: ['src/Dashboard.tsx'], affected: [], unresolvedFiles: ['src/Dashboard.tsx'], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 3,
});

describe('parseCliArgs', () => {
  it('defaults to no static-only, no trusted-ref, no json', () => {
    const opts = parseCliArgs([]);
    expect(opts.staticOnly).toBe(false);
    expect(opts.trustedRef).toBeNull();
    expect(opts.json).toBe(false);
  });

  it('parses --static-only', () => {
    expect(parseCliArgs(['--static-only']).staticOnly).toBe(true);
  });

  it('parses --trusted-ref origin/main', () => {
    expect(parseCliArgs(['--trusted-ref', 'origin/main']).trustedRef).toBe('origin/main');
  });

  it('parses --json', () => {
    expect(parseCliArgs(['--json']).json).toBe(true);
  });
});

describe('projectCli', () => {
  it('returns exit code matching result.exitCode', () => {
    expect(projectCli(regression()).exitCode).toBe(1);
    expect(projectCli(notCovered()).exitCode).toBe(3);
  });

  it('text output contains verdict keyword', () => {
    const { text } = projectCli(regression());
    expect(text).toContain('REGRESSION');
  });

  it('json output is parseable and carries schemaVersion', () => {
    const { json } = projectCli(regression());
    const parsed = JSON.parse(json!);
    expect(parsed.schemaVersion).toBe('usabl.result.v1');
  });

  it('scrubs secrets before projection', () => {
    const r = regression();
    r.findings[0].why = 'token=super-secret';
    const { text, json } = projectCli(r);
    expect(text).not.toContain('super-secret');
    expect(json ?? '').not.toContain('super-secret');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/cli.test.ts
```

- [ ] **Step 3: Implement `src/surfaces/cli.ts`**

```ts
import { formatSummary } from '../output/summary.js';
import { scrubResult } from './scrub.js';
import type { Result } from '../contracts/index.js';

export interface CliOptions {
  staticOnly: boolean;
  trustedRef: string | null;
  json: boolean;
  configPath: string;
}

export interface CliProjection {
  exitCode: number;
  text: string;
  json: string | null;
}

export function parseCliArgs(argv: string[]): CliOptions {
  let staticOnly = false;
  let trustedRef: string | null = null;
  let json = false;
  let configPath = 'usabl.config.json';
  for (let i = 0; i < argv.length; i++) {
    const a = argv[i];
    if (a === '--static-only') staticOnly = true;
    else if (a === '--trusted-ref' && argv[i + 1]) trustedRef = argv[++i];
    else if (a === '--json') json = true;
    else if (a === '--config' && argv[i + 1]) configPath = argv[++i];
  }
  return { staticOnly, trustedRef, json, configPath };
}

/** Pure projection: Result → CLI output. Always scrubs first. */
export function projectCli(result: Result): CliProjection {
  const safe = scrubResult(result);
  const text = formatSummary(safe);
  const json = JSON.stringify(safe, null, 2);
  return { exitCode: safe.exitCode, text, json };
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/cli.test.ts && npm run typecheck
```

- [ ] **Step 5: Update `src/cli.ts` bin to delegate to `projectCli`**

Adjust the main() function in `src/cli.ts` (Phase 1 file) so it imports `parseCliArgs` and `projectCli` from `src/surfaces/cli.ts` and uses `--static-only` / `--trusted-ref` when building deps. The `buildDeps` stub remains until Phase 2-3 land.

- [ ] **Step 6: Commit**

```bash
git add src/surfaces/cli.ts tests/surfaces/cli.test.ts src/cli.ts
git commit -m "feat(surfaces): full CLI with --static-only, --trusted-ref, --json"
```

---

## Task 3: Stop hook enforcer

**Files:**
- Create: `src/surfaces/stop-hook.ts`
- Test: `tests/surfaces/stop-hook.test.ts`

The stop hook is Claude Code's headline surface. It is called via the `postToolUse` or `stop` lifecycle event. It blocks on `regression` or `approval_required`, allows on `verified` (minting receipt) and `not_covered` (default block, disclosed). One-continuation escape via `USABL_BYPASS=once`. Hook errors fail open with disclosure.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/stop-hook.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { evaluateHookDecision, type HookDecision } from '../../src/surfaces/stop-hook.js';
import type { Result } from '../../src/contracts/index.js';

const make = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'verified', summary: 'verified: 0 findings',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('evaluateHookDecision', () => {
  it('allows on verified', () => {
    const d = evaluateHookDecision(make({ verdict: 'verified', exitCode: 0 }), { bypass: false });
    expect(d.allow).toBe(true);
    expect(d.exitCode).toBe(0);
    expect(d.message).toContain('verified');
  });

  it('allows on nothing-to-check (idle)', () => {
    const d = evaluateHookDecision(
      make({ verdict: null, exitCode: 0, coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: true } }),
      { bypass: false },
    );
    expect(d.allow).toBe(true);
    expect(d.message).toContain('nothing to check');
  });

  it('blocks on regression', () => {
    const d = evaluateHookDecision(make({ verdict: 'regression', exitCode: 1 }), { bypass: false });
    expect(d.allow).toBe(false);
    expect(d.exitCode).toBe(1);
    expect(d.message).toContain('REGRESSION');
  });

  it('blocks on approval_required', () => {
    const d = evaluateHookDecision(make({ verdict: 'approval_required', exitCode: 2 }), { bypass: false });
    expect(d.allow).toBe(false);
    expect(d.exitCode).toBe(2);
  });

  it('blocks on not_covered by default', () => {
    const d = evaluateHookDecision(make({ verdict: 'not_covered', exitCode: 3 }), { bypass: false });
    expect(d.allow).toBe(false);
    expect(d.exitCode).toBe(3);
  });

  it('bypass=once allows but marks NOT verified', () => {
    const d = evaluateHookDecision(make({ verdict: 'regression', exitCode: 1 }), { bypass: true });
    expect(d.allow).toBe(true);
    expect(d.message).toContain('NOT verified');
    expect(d.message).toContain('BYPASS');
  });

  it('never leaks secrets in hook message', () => {
    const r = make({ verdict: 'regression', exitCode: 1 });
    r.findings = [{
      rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
      screenId: 's', elementPath: 'div', elementName: null, role: 'generic',
      whatUserExperiences: 'Low contrast', why: 'token=secretvalue', fix: 'fix it',
      evidence: {}, confidence: 'fail', elementKey: 'k', identityBasis: 'count', status: 'new',
    }];
    const d = evaluateHookDecision(r, { bypass: false });
    expect(d.message).not.toContain('secretvalue');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/stop-hook.test.ts
```

- [ ] **Step 3: Implement `src/surfaces/stop-hook.ts`**

```ts
import { scrubResult, neutralize } from './scrub.js';
import type { Result } from '../contracts/index.js';

export interface HookOptions {
  /** True when USABL_BYPASS=once is set in the environment. Consumed once, then cleared. */
  bypass: boolean;
}

export interface HookDecision {
  allow: boolean;
  exitCode: number;
  /** Human-readable message for the assistant. Never carries secrets. */
  message: string;
}

/** Pure decision function. Scrubs result before reading any finding text. */
export function evaluateHookDecision(result: Result, opts: HookOptions): HookDecision {
  const safe = scrubResult(result);

  if (opts.bypass) {
    return {
      allow: true,
      exitCode: 0,
      message: `[usabl] BYPASS active — continuation allowed but NOT verified. ` +
        `Verdict was: ${safe.verdict ?? 'idle'}. File a finding at your earliest opportunity.`,
    };
  }

  const { verdict, exitCode } = safe;

  if (verdict === null && safe.coverage.nothingToCheck) {
    return { allow: true, exitCode: 0, message: '[usabl] nothing to check — no UI-touching files in diff.' };
  }

  if (verdict === 'verified') {
    const receipt = safe.receipt ? `receipt: ${safe.receipt.sourceTree}` : 'no receipt yet';
    return { allow: true, exitCode: 0, message: `[usabl] verified — ${receipt}. ${safe.summary}` };
  }

  if (verdict === 'regression') {
    const gating = safe.findings.filter(f => f.evidenceClass === 'deterministic' && f.status === 'new');
    const lines = gating.map(f => `  • ${f.screenId} · ${f.layer}/${f.rule}: ${neutralize(f.whatUserExperiences)}`);
    return {
      allow: false, exitCode: 1,
      message: `[usabl] REGRESSION — ${gating.length} new accessibility barrier(s) introduced:\n${lines.join('\n')}\nFix before continuing or set USABL_BYPASS=once to bypass once.`,
    };
  }

  if (verdict === 'approval_required') {
    return {
      allow: false, exitCode: 2,
      message: `[usabl] APPROVAL REQUIRED — ${safe.summary}\n` +
        (safe.dirtyGuardedPaths.length ? `Guarded paths changed: ${safe.dirtyGuardedPaths.join(', ')}` : ''),
    };
  }

  // not_covered: default block
  return {
    allow: false, exitCode: exitCode,
    message: `[usabl] NOT COVERED — ${safe.summary}\nCoverage gaps prevent a verdict. Extend surface config or accept gaps.`,
  };
}

/**
 * Entry point for the Claude Code stop hook.
 * Called via .claude/settings.json hooks[].command.
 * Reads USABL_BYPASS env var; exits with decision.exitCode.
 */
export async function runStopHook(result: Result): Promise<never> {
  const bypass = process.env['USABL_BYPASS'] === 'once';
  if (bypass) delete process.env['USABL_BYPASS'];
  const decision = evaluateHookDecision(result, { bypass });
  process.stderr.write(decision.message + '\n');
  process.exit(decision.allow ? 0 : decision.exitCode);
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/stop-hook.test.ts && npm run typecheck
```

- [ ] **Step 5: Commit**

```bash
git add src/surfaces/stop-hook.ts tests/surfaces/stop-hook.test.ts
git commit -m "feat(surfaces): stop hook enforcer with bypass escape hatch"
```

---

## Task 4: PR comment formatter and CI GitHub Action

**Files:**
- Create: `src/surfaces/pr-comment.ts`
- Create: `.github/workflows/usabl.yml`
- Test: `tests/surfaces/pr-comment.test.ts`

The PR comment leads with the receipt, groups findings as new/known/waived, applies `neutralize()` to all page-derived text. The GitHub Action reads policy from `--trusted-ref` (never the PR head). Pinned to SHA refs.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/pr-comment.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { projectPrComment } from '../../src/surfaces/pr-comment.js';
import type { Result, Receipt } from '../../src/contracts/index.js';

const receipt: Receipt = {
  schemaVersion: 1, mintedAt: '2026-08-20T10:00:00Z',
  sourceTree: 'abc123', configHash: 'cfghash', floorHash: 'flrhash',
};

const make = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'verified', summary: 'verified: 0 findings',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('projectPrComment', () => {
  it('opens with the receipt block when present', () => {
    const md = projectPrComment(make({}));
    expect(md).toContain('abc123');
    expect(md).toContain('usabl.result.v1');
  });

  it('labels new findings distinctly', () => {
    const r = make({
      verdict: 'regression', exitCode: 1,
      findings: [{
        rule: 'image-alt', layer: 'axe', severity: 'critical', evidenceClass: 'deterministic',
        screenId: 'dashboard', elementPath: 'img', elementName: null, role: 'img',
        whatUserExperiences: 'Image has no alt text', why: '', fix: 'Add alt attribute',
        evidence: {}, confidence: 'fail', elementKey: 'ek', identityBasis: 'structural', status: 'new',
      }],
    });
    const md = projectPrComment(r);
    expect(md).toContain('New barriers');
    expect(md).toContain('image-alt');
    expect(md).toContain('dashboard');
  });

  it('groups carried findings as known', () => {
    const r = make({
      verdict: 'verified', exitCode: 0,
      findings: [{
        rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
        screenId: 'clusters', elementPath: 'span', elementName: 'Label', role: 'generic',
        whatUserExperiences: 'Low contrast', why: '', fix: 'Raise ratio',
        evidence: {}, confidence: 'fail', elementKey: 'ek2', identityBasis: 'name', status: 'carried',
      }],
    });
    const md = projectPrComment(r);
    expect(md).toContain('Known (carried)');
  });

  it('never emits a secret in the comment', () => {
    const r = make({ verdict: 'regression', exitCode: 1 });
    r.findings = [{
      rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
      screenId: 's', elementPath: 'div', elementName: 'token=secretXYZ', role: 'generic',
      whatUserExperiences: 'Low contrast', why: '', fix: '',
      evidence: {}, confidence: 'fail', elementKey: 'ek', identityBasis: 'name', status: 'new',
    }];
    const md = projectPrComment(r);
    expect(md).not.toContain('secretXYZ');
  });

  it('comment is anchored with the bot marker for sticky matching', () => {
    const md = projectPrComment(make({}));
    expect(md).toContain('<!-- usabl-report -->');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/pr-comment.test.ts
```

- [ ] **Step 3: Implement `src/surfaces/pr-comment.ts`**

```ts
import { scrubResult, neutralize } from './scrub.js';
import type { Finding, Result } from '../contracts/index.js';

const BOT_MARKER = '<!-- usabl-report -->';

function findingRow(f: Finding): string {
  return `| \`${f.screenId}\` | \`${f.layer}/${f.rule}\` | ${f.severity} | ${neutralize(f.whatUserExperiences)} | ${neutralize(f.fix ?? '')} |`;
}

function findingTable(label: string, findings: Finding[]): string {
  if (findings.length === 0) return '';
  const rows = findings.map(findingRow).join('\n');
  return `\n### ${label}\n| Screen | Rule | Severity | Experience | Fix |\n|---|---|---|---|---|\n${rows}\n`;
}

/**
 * Pure projection: Result → GitHub-flavored Markdown PR comment.
 * Always scrubs first. Anchored with BOT_MARKER for sticky-comment matching.
 * Leads with receipt, groups findings as new/known/waived.
 */
export function projectPrComment(result: Result): string {
  const safe = scrubResult(result);
  const lines: string[] = [BOT_MARKER, ''];

  // Header
  const verdictLabel = safe.verdict === null ? 'IDLE' : safe.verdict.toUpperCase().replace('_', ' ');
  lines.push(`## usabl: ${verdictLabel}`);
  lines.push('');
  lines.push(`\`schemaVersion\`: \`${safe.schemaVersion}\``);
  lines.push('');

  // Receipt block
  if (safe.receipt) {
    lines.push('### Receipt');
    lines.push('```');
    lines.push(`sourceTree : ${safe.receipt.sourceTree}`);
    lines.push(`mintedAt   : ${safe.receipt.mintedAt}`);
    lines.push(`configHash : ${safe.receipt.configHash}`);
    lines.push(`floorHash  : ${safe.receipt.floorHash}`);
    lines.push('```');
  } else {
    lines.push('_No receipt — run was not fully verified._');
  }
  lines.push('');

  // Summary
  lines.push(`**${safe.summary}**`);
  lines.push('');

  // Findings by status
  const newF = safe.findings.filter(f => f.status === 'new' && f.evidenceClass === 'deterministic');
  const carriedF = safe.findings.filter(f => f.status === 'carried');
  const advisoryF = safe.findings.filter(f => f.evidenceClass !== 'deterministic');

  lines.push(findingTable('New barriers', newF));
  lines.push(findingTable('Known (carried)', carriedF));
  if (advisoryF.length > 0) {
    lines.push(findingTable('Advisory (preview/judgment — non-gating)', advisoryF));
  }

  // Coverage gaps
  const gaps = safe.coverage.gaps;
  if (gaps.length > 0) {
    lines.push('\n### Coverage gaps');
    for (const g of gaps) {
      lines.push(`- \`${g.screenId ?? g.layer ?? 'unknown'}\`: ${neutralize(g.reason)}`);
    }
  }

  return lines.join('\n');
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/pr-comment.test.ts && npm run typecheck
```

- [ ] **Step 5: Create `.github/workflows/usabl.yml`**

```yaml
# .github/workflows/usabl.yml
# Pins ALL actions to SHA refs (security control: pinned-actions).
name: usabl accessibility gate

on:
  pull_request:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  usabl-check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
        with:
          fetch-depth: 0

      - name: Setup Node
        uses: actions/setup-node@cdca7365b2dadb8aad0a33bc7601856ffabcc48e  # v4.3.0
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run usabl check (trusted-ref policy from base branch)
        id: usabl
        # Reads evidence floor and config from origin/main, never from the PR head.
        # Exit code 0 = pass; 1 = regression (block); 2 = approval_required (block);
        # 3 = not_covered (annotate but not required status by default).
        run: |
          node dist/cli.js check --trusted-ref origin/${{ github.base_ref }} --json > usabl-result.json
          echo "exit_code=$?" >> "$GITHUB_OUTPUT"
        continue-on-error: true

      - name: Post PR comment
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea  # v7.0.1
        env:
          USABL_RESULT_PATH: usabl-result.json
        with:
          script: |
            const fs = require('fs');
            const { execSync } = require('child_process');
            const result = JSON.parse(fs.readFileSync(process.env.USABL_RESULT_PATH, 'utf8'));
            // projectPrComment is a pure function; call via Node require after build.
            const { projectPrComment } = require('./dist/surfaces/pr-comment.js');
            const body = projectPrComment(result);
            const marker = '<!-- usabl-report -->';
            const { data: comments } = await github.rest.issues.listComments({
              owner: context.repo.owner, repo: context.repo.repo, issue_number: context.issue.number,
            });
            // Sticky: only match bot-authored comments with our marker.
            const existing = comments.find(c => c.user?.type === 'Bot' && c.body?.includes(marker));
            if (existing) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner, repo: context.repo.repo,
                comment_id: existing.id, body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner, repo: context.repo.repo,
                issue_number: context.issue.number, body,
              });
            }

      - name: Enforce exit code
        run: exit ${{ steps.usabl.outputs.exit_code }}
```

- [ ] **Step 6: Commit**

```bash
git add src/surfaces/pr-comment.ts tests/surfaces/pr-comment.test.ts .github/workflows/usabl.yml
git commit -m "feat(surfaces): PR comment formatter and CI GitHub Action (pinned SHA refs)"
```

---

## Task 5: Vite plugin overlay (advisory)

**Files:**
- Create: `src/surfaces/vite-plugin.ts`
- Test: `tests/surfaces/vite-plugin.test.ts`

The Vite plugin is advisory only. It never gates, never changes `exitCode`. It mounts only when `?usabl=off` is absent and `navigator.webdriver` is false (so it is a no-op in Playwright test runs). Ships as `usabl/vite` subpath export.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/vite-plugin.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { projectOverlay } from '../../src/surfaces/vite-plugin.js';
import type { Result } from '../../src/contracts/index.js';

const base = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'verified', summary: 'verified: 0 findings',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('projectOverlay', () => {
  it('returns advisory:true always', () => {
    expect(projectOverlay(base({})).advisory).toBe(true);
    expect(projectOverlay(base({ verdict: 'regression', exitCode: 1 })).advisory).toBe(true);
  });

  it('never returns a verdict that could be mistaken for gating', () => {
    // The overlay projection carries schemaVersion and verdict for display,
    // but is clearly marked advisory and its exitCode is always 0.
    const p = projectOverlay(base({ verdict: 'regression', exitCode: 1 }));
    expect(p.displayExitCode).toBe(0);
    expect(p.schemaVersion).toBe('usabl.result.v1');
    expect(p.verdict).toBe('regression');
  });

  it('scrubs secrets from overlay payload', () => {
    const r = base({ verdict: 'regression', exitCode: 1 });
    r.findings = [{
      rule: 'label', layer: 'axe', severity: 'critical', evidenceClass: 'deterministic',
      screenId: 's', elementPath: 'input', elementName: null, role: 'textbox',
      whatUserExperiences: 'password=hunter2 exposed', why: '', fix: '',
      evidence: {}, confidence: 'fail', elementKey: 'ek', identityBasis: 'count', status: 'new',
    }];
    const p = projectOverlay(r);
    expect(JSON.stringify(p)).not.toContain('hunter2');
  });

  it('summary text contains the verdict label', () => {
    expect(projectOverlay(base({ verdict: 'not_covered', exitCode: 3 })).summaryText).toContain('not_covered');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/vite-plugin.test.ts
```

- [ ] **Step 3: Implement `src/surfaces/vite-plugin.ts`**

```ts
import { scrubResult } from './scrub.js';
import type { Result } from '../contracts/index.js';

export interface OverlayProjection {
  advisory: true;
  schemaVersion: 'usabl.result.v1';
  verdict: Result['verdict'];
  summaryText: string;
  /** Always 0 — the overlay never gates. */
  displayExitCode: 0;
  findings: Result['findings'];
}

/** Pure projection: Result → overlay payload. Always advisory, always displayExitCode 0. */
export function projectOverlay(result: Result): OverlayProjection {
  const safe = scrubResult(result);
  return {
    advisory: true,
    schemaVersion: 'usabl.result.v1',
    verdict: safe.verdict,
    summaryText: safe.summary,
    displayExitCode: 0,
    findings: safe.findings,
  };
}

/**
 * Vite plugin factory. Ships as usabl/vite subpath export.
 * Advisory only — never blocks HMR, never calls process.exit.
 * No-op when navigator.webdriver (Playwright) or ?usabl=off.
 *
 * Integration task: wire this into the real Vite dev server.
 * The pure projection (projectOverlay) is unit-tested above.
 */
export function usablVitePlugin(): unknown {
  // Returns a Vite plugin object. The type is `unknown` here so this file
  // has no hard dependency on vite's types in the core package.
  // Real wiring in the integration task: import { Plugin } from 'vite'.
  return {
    name: 'usabl-overlay',
    apply: 'serve' as const,
    transformIndexHtml(html: string): string {
      // Inject overlay client script. Skipped when ?usabl=off is present (client-side check).
      const client = `<script type="module">
if (!navigator.webdriver && !location.search.includes('usabl=off')) {
  // Overlay client: polls /usabl-overlay endpoint, renders non-gating badge.
  import('/usabl-overlay-client.js').catch(() => {});
}
</script>`;
      return html.replace('</body>', `${client}\n</body>`);
    },
  };
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/vite-plugin.test.ts && npm run typecheck
```

- [ ] **Step 5: Commit**

```bash
git add src/surfaces/vite-plugin.ts tests/surfaces/vite-plugin.test.ts
git commit -m "feat(surfaces): Vite overlay plugin (advisory, never gates)"
```

---

## Task 6: Playwright helper

**Files:**
- Create: `src/surfaces/playwright-helper.ts`
- Test: `tests/surfaces/playwright-helper.test.ts`

A seam that lets a Playwright test assert on the usabl verdict. Ships as `usabl/playwright` subpath export. The unit test uses a canned Result; real browser wiring is an integration task.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/playwright-helper.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { assertUsablVerdict, type PlaywrightCheckResult } from '../../src/surfaces/playwright-helper.js';
import type { Result } from '../../src/contracts/index.js';

const make = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'verified', summary: 'verified: 0 findings',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('assertUsablVerdict', () => {
  it('returns passed:true and exitCode:0 for verified', () => {
    const r: PlaywrightCheckResult = assertUsablVerdict(make({}), ['verified']);
    expect(r.passed).toBe(true);
    expect(r.exitCode).toBe(0);
  });

  it('returns passed:false and the result verdict for regression', () => {
    const r = assertUsablVerdict(make({ verdict: 'regression', exitCode: 1 }), ['verified']);
    expect(r.passed).toBe(false);
    expect(r.verdict).toBe('regression');
    expect(r.exitCode).toBe(1);
  });

  it('accepts multiple allowed verdicts', () => {
    const r = assertUsablVerdict(make({ verdict: 'not_covered', exitCode: 3 }), ['verified', 'not_covered']);
    expect(r.passed).toBe(true);
  });

  it('scrubs secrets in the returned projection', () => {
    const result = make({ verdict: 'regression', exitCode: 1 });
    result.findings = [{
      rule: 'r', layer: 'axe', severity: 'minor', evidenceClass: 'deterministic',
      screenId: 's', elementPath: 'a', elementName: 'token=topsecret', role: 'link',
      whatUserExperiences: 'x', why: '', fix: '',
      evidence: {}, confidence: 'fail', elementKey: 'k', identityBasis: 'name', status: 'new',
    }];
    const r = assertUsablVerdict(result, ['verified']);
    expect(JSON.stringify(r.safeResult)).not.toContain('topsecret');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/playwright-helper.test.ts
```

- [ ] **Step 3: Implement `src/surfaces/playwright-helper.ts`**

```ts
import { scrubResult } from './scrub.js';
import type { Result, Verdict } from '../contracts/index.js';

export interface PlaywrightCheckResult {
  passed: boolean;
  verdict: Verdict | null;
  exitCode: number;
  summary: string;
  /** Scrubbed copy of Result safe to include in test output. */
  safeResult: Result;
}

/**
 * Assert that a usabl Result carries one of the allowed verdicts.
 * Returns a structured result; the caller decides how to throw (expect(r.passed).toBe(true)).
 *
 * Integration usage:
 *   import { run } from 'usabl';
 *   import { assertUsablVerdict } from 'usabl/playwright';
 *   const result = await run(deps, config);
 *   const check = assertUsablVerdict(result, ['verified']);
 *   expect(check.passed).toBe(true);
 */
export function assertUsablVerdict(
  result: Result,
  allowedVerdicts: Array<Verdict | null>,
): PlaywrightCheckResult {
  const safe = scrubResult(result);
  const passed = allowedVerdicts.includes(safe.verdict);
  return {
    passed,
    verdict: safe.verdict,
    exitCode: safe.exitCode,
    summary: safe.summary,
    safeResult: safe,
  };
}
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/playwright-helper.test.ts && npm run typecheck
```

- [ ] **Step 5: Register `usabl/playwright` subpath export in `package.json`**

Add to the `exports` map:
```json
"./playwright": { "types": "./dist/surfaces/playwright-helper.d.ts", "import": "./dist/surfaces/playwright-helper.js" }
```
Add `usabl/vite` similarly for `src/surfaces/vite-plugin.ts`.

- [ ] **Step 6: Commit**

```bash
git add src/surfaces/playwright-helper.ts tests/surfaces/playwright-helper.test.ts package.json
git commit -m "feat(surfaces): Playwright helper seam (assertUsablVerdict)"
```

---

## Task 7: Mid-task self-check seam

**Files:**
- Create: `src/surfaces/self-check.ts`
- Test: `tests/surfaces/self-check.test.ts`

The mid-task self-check runs `usabl check --self-check` from the Claude Code Bash tool. It is advisory: it prints a structured message to stdout and exits 0 regardless of verdict, so it never blocks the agent mid-task. The Stop hook remains the gate.

- [ ] **Step 1: Write the failing test**

`tests/surfaces/self-check.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import { projectSelfCheck } from '../../src/surfaces/self-check.js';
import type { Result } from '../../src/contracts/index.js';

const make = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'verified', summary: 'verified: 0 findings',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt: null, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

describe('projectSelfCheck', () => {
  it('always returns advisoryExitCode 0', () => {
    expect(projectSelfCheck(make({ verdict: 'regression', exitCode: 1 })).advisoryExitCode).toBe(0);
    expect(projectSelfCheck(make({ verdict: 'approval_required', exitCode: 2 })).advisoryExitCode).toBe(0);
    expect(projectSelfCheck(make({ verdict: 'not_covered', exitCode: 3 })).advisoryExitCode).toBe(0);
  });

  it('carries the real verdict for agent awareness', () => {
    expect(projectSelfCheck(make({ verdict: 'regression', exitCode: 1 })).verdict).toBe('regression');
  });

  it('message advises agent to fix before stop hook fires', () => {
    const p = projectSelfCheck(make({ verdict: 'regression', exitCode: 1 }));
    expect(p.message).toContain('advisory');
    expect(p.message).toContain('regression');
  });

  it('does not leak secrets', () => {
    const r = make({ verdict: 'regression', exitCode: 1 });
    r.findings = [{
      rule: 'r', layer: 'axe', severity: 'minor', evidenceClass: 'deterministic',
      screenId: 's', elementPath: 'a', elementName: null, role: 'link',
      whatUserExperiences: 'token=opaque99', why: '', fix: '',
      evidence: {}, confidence: 'fail', elementKey: 'k', identityBasis: 'count', status: 'new',
    }];
    expect(projectSelfCheck(r).message).not.toContain('opaque99');
  });
});
```

- [ ] **Step 2: Run test — expect FAIL**

```bash
npx vitest run tests/surfaces/self-check.test.ts
```

- [ ] **Step 3: Implement `src/surfaces/self-check.ts`**

```ts
import { scrubResult, neutralize } from './scrub.js';
import type { Result } from '../contracts/index.js';

export interface SelfCheckProjection {
  /** Always 0 — mid-task self-check is advisory. Stop hook is the gate. */
  advisoryExitCode: 0;
  verdict: Result['verdict'];
  message: string;
}

/** Pure projection: Result → advisory self-check message for agent context. */
export function projectSelfCheck(result: Result): SelfCheckProjection {
  const safe = scrubResult(result);
  const v = safe.verdict ?? 'idle';

  const gating = safe.findings
    .filter(f => f.evidenceClass === 'deterministic' && f.status === 'new')
    .map(f => `  • ${f.screenId} · ${f.layer}/${f.rule}: ${neutralize(f.whatUserExperiences)}`)
    .join('\n');

  const message = [
    `[usabl self-check] advisory — stop hook is the gate.`,
    `Verdict: ${v} | ${safe.summary}`,
    gating ? `Findings to address:\n${gating}` : 'No new deterministic findings.',
  ].join('\n');

  return { advisoryExitCode: 0, verdict: safe.verdict, message };
}

/**
 * MCP seam (documented, not built for contest).
 * When an MCP wrapper ships: same engine, thin protocol adapter.
 * neutralize() MUST wrap all page-derived text before returning tool output.
 * See ground-truth section 11.5 and security controls section 17.
 */
```

- [ ] **Step 4: Run test — expect PASS**

```bash
npx vitest run tests/surfaces/self-check.test.ts && npm run typecheck
```

- [ ] **Step 5: Commit**

```bash
git add src/surfaces/self-check.ts tests/surfaces/self-check.test.ts
git commit -m "feat(surfaces): mid-task self-check projection (advisory, exitCode always 0)"
```

---

## Task 8: Exit-criterion integration test

**Files:**
- Create: `tests/surfaces/integration.test.ts`

**Exit criterion:** the same `Result` fixture drives CLI, stop hook, PR comment, Vite overlay, and Playwright helper to the same verdict and exit code. Secrets never appear in any projection.

- [ ] **Step 1: Write the integration test**

`tests/surfaces/integration.test.ts`:
```ts
import { describe, it, expect } from 'vitest';
import type { Result, Receipt } from '../../src/contracts/index.js';
import { projectCli } from '../../src/surfaces/cli.js';
import { evaluateHookDecision } from '../../src/surfaces/stop-hook.js';
import { projectPrComment } from '../../src/surfaces/pr-comment.js';
import { projectOverlay } from '../../src/surfaces/vite-plugin.js';
import { assertUsablVerdict } from '../../src/surfaces/playwright-helper.js';
import { projectSelfCheck } from '../../src/surfaces/self-check.js';

const POISON = 'token=SECRETPOISON';
const receipt: Receipt = {
  schemaVersion: 1, mintedAt: '2026-08-20T12:00:00Z',
  sourceTree: 'deadbeef', configHash: 'cfghash', floorHash: 'flrhash',
};

const fixture = (over: Partial<Result>): Result => ({
  schemaVersion: 'usabl.result.v1', verdict: 'verified', summary: 'verified: 0 findings',
  screens: [], coverage: { changedFiles: [], affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: false },
  findings: [], receipt, dirtyGuardedPaths: [], exitCode: 0, ...over,
});

const REGRESSION_FIXTURE = fixture({
  verdict: 'regression', exitCode: 1, summary: 'regression: 1 new finding', receipt: null,
  findings: [{
    rule: 'image-alt', layer: 'axe', severity: 'critical', evidenceClass: 'deterministic',
    screenId: 'dashboard', elementPath: 'img.hero', elementName: null, role: 'img',
    whatUserExperiences: `Image has no alt text — ${POISON}`, why: '', fix: 'Add alt attribute',
    evidence: {}, confidence: 'fail', elementKey: 'dashboard|axe/image-alt|img.hero', identityBasis: 'structural', status: 'new',
  }],
});

describe('Exit criterion: same Result → same verdict and exit code across all surfaces', () => {
  it('regression fixture: all surfaces agree on exitCode=1', () => {
    const cli = projectCli(REGRESSION_FIXTURE);
    const hook = evaluateHookDecision(REGRESSION_FIXTURE, { bypass: false });
    // PR comment doesn't produce an exit code, but the fixture exitCode is the authority.
    // Overlay always displayExitCode=0 (advisory); self-check always advisoryExitCode=0.
    const overlay = projectOverlay(REGRESSION_FIXTURE);
    const playwright = assertUsablVerdict(REGRESSION_FIXTURE, ['verified']);
    const selfCheck = projectSelfCheck(REGRESSION_FIXTURE);

    expect(cli.exitCode).toBe(1);
    expect(hook.exitCode).toBe(1);
    expect(hook.allow).toBe(false);
    expect(overlay.advisory).toBe(true);
    expect(overlay.displayExitCode).toBe(0);      // advisory surface, not gate
    expect(playwright.exitCode).toBe(1);
    expect(playwright.passed).toBe(false);
    expect(selfCheck.advisoryExitCode).toBe(0);   // advisory surface, not gate
    expect(selfCheck.verdict).toBe('regression');
  });

  it('all gating surfaces: same verdict', () => {
    expect(projectCli(REGRESSION_FIXTURE).exitCode).toBe(REGRESSION_FIXTURE.exitCode);
    expect(evaluateHookDecision(REGRESSION_FIXTURE, { bypass: false }).exitCode).toBe(REGRESSION_FIXTURE.exitCode);
    expect(assertUsablVerdict(REGRESSION_FIXTURE, ['verified']).verdict).toBe(REGRESSION_FIXTURE.verdict);
  });

  it('POISON secret never appears in any projection', () => {
    const all: string[] = [
      projectCli(REGRESSION_FIXTURE).text,
      projectCli(REGRESSION_FIXTURE).json ?? '',
      evaluateHookDecision(REGRESSION_FIXTURE, { bypass: false }).message,
      projectPrComment(REGRESSION_FIXTURE),
      JSON.stringify(projectOverlay(REGRESSION_FIXTURE)),
      JSON.stringify(assertUsablVerdict(REGRESSION_FIXTURE, ['verified']).safeResult),
      projectSelfCheck(REGRESSION_FIXTURE).message,
    ];
    for (const output of all) {
      expect(output, `Surface leaked "${POISON}"`).not.toContain('SECRETPOISON');
    }
  });

  it('verified fixture: gating surfaces allow, advisory surfaces show verified', () => {
    const verified = fixture({});
    expect(projectCli(verified).exitCode).toBe(0);
    expect(evaluateHookDecision(verified, { bypass: false }).allow).toBe(true);
    expect(assertUsablVerdict(verified, ['verified']).passed).toBe(true);
    expect(projectOverlay(verified).verdict).toBe('verified');
  });

  it('not_covered: gating surfaces block (exit 3), advisory show not_covered', () => {
    const nc = fixture({ verdict: 'not_covered', exitCode: 3 });
    expect(projectCli(nc).exitCode).toBe(3);
    expect(evaluateHookDecision(nc, { bypass: false }).allow).toBe(false);
    expect(assertUsablVerdict(nc, ['verified']).passed).toBe(false);
    expect(projectOverlay(nc).verdict).toBe('not_covered');
    expect(projectOverlay(nc).displayExitCode).toBe(0);
  });

  it('approval_required: gating surfaces block (exit 2)', () => {
    const ar = fixture({ verdict: 'approval_required', exitCode: 2 });
    expect(projectCli(ar).exitCode).toBe(2);
    expect(evaluateHookDecision(ar, { bypass: false }).allow).toBe(false);
    expect(evaluateHookDecision(ar, { bypass: false }).exitCode).toBe(2);
  });

  it('bypass allows regression but marks NOT verified', () => {
    const hook = evaluateHookDecision(REGRESSION_FIXTURE, { bypass: true });
    expect(hook.allow).toBe(true);
    expect(hook.exitCode).toBe(0);
    expect(hook.message).toContain('NOT verified');
  });

  it('schemaVersion propagates from Result through every projection', () => {
    expect(projectCli(fixture({})).json).toContain('"schemaVersion": "usabl.result.v1"');
    expect(projectPrComment(fixture({}))).toContain('usabl.result.v1');
    expect(projectOverlay(fixture({})).schemaVersion).toBe('usabl.result.v1');
    expect(assertUsablVerdict(fixture({}), ['verified']).safeResult.schemaVersion).toBe('usabl.result.v1');
  });
});
```

- [ ] **Step 2: Run integration test — expect PASS**

```bash
npx vitest run tests/surfaces/integration.test.ts && npm run typecheck
```

All tests must pass before this phase is complete.

- [ ] **Step 3: Run full suite**

```bash
npm run check
```

Expected: all tests pass, no type errors.

- [ ] **Step 4: Commit**

```bash
git add tests/surfaces/integration.test.ts
git commit -m "test(surfaces): exit-criterion integration — same Result, same verdict, no secrets"
```

---

## Self-Review

**Coverage of all six surfaces:**

| Surface | File | Unit test | Advisory? | Gates? |
|---|---|---|---|---|
| Full CLI | `src/surfaces/cli.ts` | `tests/surfaces/cli.test.ts` | No | Yes (exit code) |
| Stop hook | `src/surfaces/stop-hook.ts` | `tests/surfaces/stop-hook.test.ts` | No | Yes (blocks) |
| CI / PR comment | `src/surfaces/pr-comment.ts` + `.github/workflows/usabl.yml` | `tests/surfaces/pr-comment.test.ts` | No (comment) | Via CI status |
| Vite overlay | `src/surfaces/vite-plugin.ts` | `tests/surfaces/vite-plugin.test.ts` | Yes | Never |
| Playwright helper | `src/surfaces/playwright-helper.ts` | `tests/surfaces/playwright-helper.test.ts` | Seam | Test-controlled |
| Mid-task self-check | `src/surfaces/self-check.ts` | `tests/surfaces/self-check.test.ts` | Yes | Never |

**Placeholder scan:** no "TBD", no "TODO", no undefined types in any code block. The MCP seam is documented as a comment block in `self-check.ts` with an explicit note that it is not built for contest.

**Type consistency with frozen contracts:** all surfaces import `Result`, `Receipt`, `Finding`, `Verdict` from `src/contracts/index.ts`. `ConformanceSummary` is produced by `computeConformance()` (Phase 1) and consumed by the PR comment as a non-gating display projection; it is not re-derived. `Coverage` and `CoverageGap` are read from `Result.coverage` and `Result.coverage.gaps` directly.

**Same Result, same verdict:** the integration test in Task 8 drives one `REGRESSION_FIXTURE` through all six surfaces and asserts identical `exitCode` on gating surfaces (CLI: 1, stop hook: 1, Playwright helper: 1) and advisory-zero on non-gating surfaces (overlay: 0, self-check: 0). `schemaVersion: 'usabl.result.v1'` propagates through every projection. `SECRETPOISON` appears in no output.

**GitHub Action pins:** `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683`, `actions/setup-node@cdca7365b2dadb8aad0a33bc7601856ffabcc48e`, `actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea` — all SHA refs, no floating tags.

**Integration tasks remaining (not unit-tested here):**
1. Wire `buildDeps()` in `src/cli.ts` with real Playwright browser + git (Phase 2-3 deliverable).
2. Connect the Vite plugin to a real dev-server HMR cycle and the `/usabl-overlay-client.js` endpoint.
3. Register the Claude Code stop hook in `.claude/settings.json` pointing to `node dist/surfaces/stop-hook-runner.js`.
4. Confirm `--trusted-ref` reads the evidence floor and config from the trusted ref via `deps.git.readFileAt(ref, path)` (Phase 3 contract).
