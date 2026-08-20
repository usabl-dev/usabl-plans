# Coverage, Guard, and Trust Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Populate `Coverage` from real discovery (route manifest + import graph +
wide-blast + manual surfaces); harden the guard (config file unconditionally
self-guarded, directory guarded paths expanded to files, session-pin helpers); bind and
re-verify the receipt through the single `computePolicyHash`; implement the trusted-ref
read path so CI judges a PR against the base ref's policy; and close the evidence floor
accept loop so a policy edit forces `approval_required` and an accept commit converges
the next run to `verified`.

**Architecture:**
- `run(deps, config, opts?)` keeps the frozen Phase 1 signature. This phase swaps the
  internals (planner instead of the direct-mapping stub, hardened guard, expanded
  guarded set for the receipt); it never changes the public API.
- The gate remains the single verdict authority. This phase feeds it
  (`coverage.gaps`, `guardDivergedPaths`) and does not re-implement it.
- The config file is UNCONDITIONALLY self-guarded and checked FIRST. A config that
  edits itself out of its own `guardedPaths` is still caught, because the guard never
  trusts the working-tree config's contents before verifying the config file against
  HEAD. Ordering is load-bearing.
- Directory guarded paths expand to files: the union of committed files under the
  prefix (`git.lsFiles`) and working-tree files under it (`fs.glob`). Editing
  `src/gate/anything.ts` diverges the guard. The engine cannot be silently self-edited.
- ONE policy hash: `computePolicyHash` from Phase 1 (`evidence/receipt.ts`), computed
  over the EXPANDED guarded set. `mintReceipt` and `verifyReceipt` use the same
  function with the same inputs. No second algorithm.
- `opts.trustedRef` (CI): floor and waivers are read from the trusted ref via
  `git.show(ref, path)` (already wired in Phase 1's `readPolicyJson`). A PR cannot
  influence the policy it is judged against.
- The evidence floor accepts only `confidence: 'fail'` findings. Unverified findings
  cannot be accepted; the surface stays `not_covered` until they are resolved.
- Ambiguous coverage resolves to honest gaps (`not_covered`), never a warning.

**Tech Stack:** TypeScript (ESM, strict), Node 22, Vitest (fakes only in unit tests;
`makeFakeDeps` from Phase 1, never hand-rolled `Deps`), tsup.

---

## Task 1: Route manifest loader and import-graph builder

**Files:**
- `src/coverage/route-manifest.ts`
- `src/coverage/import-graph.ts`
- `test/coverage/route-manifest.test.ts`
- `test/coverage/import-graph.test.ts`

The route manifest maps `screenId → { url, entryFile }`. The JSON sidecar
(`usabl.routes.json`) is the real mechanism: it attributes an entry file per route so
the import graph can map a changed file to specific screens. The regex fallback (React
Router `path="..."` extraction) recovers route ids and URLs but CANNOT attribute entry
files; such routes carry `entryFile: null`, are reachable through wide-blast and manual
surfaces, and file-level attribution honestly does not exist for them. Bundler aliases
that cannot be resolved stay unresolvable. No silent passes.

- [ ] **Step 1: Write failing tests**

```ts
// test/coverage/route-manifest.test.ts
import { describe, it, expect } from 'vitest';
import { parseRouteManifest } from '../../src/coverage/route-manifest.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';

const fsOf = (files: Record<string, string>) => makeFakeDeps({ files }).fs;

describe('parseRouteManifest', () => {
  it('loads a JSON sidecar when present', async () => {
    const fs = fsOf({
      'usabl.routes.json': JSON.stringify({
        routes: [{ screenId: 'clusters', url: '/clusters', entryFile: 'src/ClustersPage.tsx' }],
      }),
    });
    const manifest = await parseRouteManifest(fs, { routerFile: 'src/router.tsx', wideBlastGlobs: [] });
    expect(manifest.routes).toHaveLength(1);
    expect(manifest.routes[0]!.screenId).toBe('clusters');
    expect(manifest.routes[0]!.entryFile).toBe('src/ClustersPage.tsx');
  });

  it('falls back to regex extraction with entryFile null (no attribution invented)', async () => {
    const fs = fsOf({
      'src/router.tsx': `
        <Route path="/alerts" element={<AlertsPage />} />
        <Route path="/hosts"  element={<HostsPage />} />
      `,
    });
    const manifest = await parseRouteManifest(fs, { routerFile: 'src/router.tsx', wideBlastGlobs: [] });
    expect(manifest.routes.map((r) => r.screenId).sort()).toEqual(['alerts', 'hosts']);
    expect(manifest.routes.every((r) => r.entryFile === null)).toBe(true);
  });

  it('returns an empty manifest when neither source exists', async () => {
    const manifest = await parseRouteManifest(fsOf({}), { routerFile: 'src/router.tsx', wideBlastGlobs: [] });
    expect(manifest.routes).toHaveLength(0);
  });
});
```

```ts
// test/coverage/import-graph.test.ts
import { describe, it, expect } from 'vitest';
import { buildImportGraph } from '../../src/coverage/import-graph.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';

const fsOf = (files: Record<string, string>) => makeFakeDeps({ files }).fs;

describe('buildImportGraph', () => {
  it('resolves a relative import by probing real extensions (.ts wins when .tsx absent)', async () => {
    const fs = fsOf({
      'src/ClustersPage.tsx': `import { ClusterTable } from './ClusterTable';`,
      'src/ClusterTable.ts': `export const ClusterTable = 1;`,
    });
    const graph = await buildImportGraph(fs, ['src/ClustersPage.tsx']);
    expect(graph.get('src/ClustersPage.tsx')).toContain('src/ClusterTable.ts');
  });

  it('records an alias import as unresolvable', async () => {
    const fs = fsOf({ 'src/Page.tsx': `import { Foo } from '@/components/Foo';` });
    const graph = await buildImportGraph(fs, ['src/Page.tsx']);
    expect(graph.unresolvable.some((u) => u.includes('@/components/Foo'))).toBe(true);
  });

  it('records a relative import with no existing candidate file as unresolvable', async () => {
    const fs = fsOf({ 'src/Page.tsx': `import { Gone } from './Gone';` });
    const graph = await buildImportGraph(fs, ['src/Page.tsx']);
    expect(graph.unresolvable.some((u) => u.includes('./Gone'))).toBe(true);
  });

  it('terminates BFS on cycles', async () => {
    const fs = fsOf({
      'src/A.tsx': `import { B } from './B';`,
      'src/B.tsx': `import { A } from './A';`,
    });
    const graph = await buildImportGraph(fs, ['src/A.tsx']);
    expect(graph.get('src/A.tsx')).toContain('src/B.tsx');
  });
});
```

- [ ] **Step 2: Run: expected FAIL**

```bash
npx vitest run test/coverage/route-manifest.test.ts test/coverage/import-graph.test.ts
```

- [ ] **Step 3: Write implementation**

```ts
// src/coverage/route-manifest.ts
import type { FsGlob } from '../contracts/index.js';

export interface RouteEntry { screenId: string; url: string; entryFile: string | null; }
export interface RouteManifest { routes: RouteEntry[]; }

function urlToScreenId(url: string): string {
  return url.replace(/^\//, '').replace(/\//g, '-') || 'root';
}

export async function parseRouteManifest(
  fs: FsGlob,
  discovery: { routerFile: string; wideBlastGlobs: string[] },
): Promise<RouteManifest> {
  const sidecar = await fs.readFile('usabl.routes.json');
  if (sidecar != null) return JSON.parse(sidecar) as RouteManifest;

  const source = await fs.readFile(discovery.routerFile);
  if (source == null) return { routes: [] };
  // Regex fallback recovers route ids and URLs only. It cannot attribute entry
  // files, so entryFile is null and file-level mapping honestly does not exist
  // for these routes (wide-blast and manual surfaces still cover them).
  const re = /path=["'`](\/[^"'`]*)["'`]/g;
  const routes: RouteEntry[] = [];
  let m: RegExpExecArray | null;
  while ((m = re.exec(source)) !== null) {
    routes.push({ screenId: urlToScreenId(m[1]!), url: m[1]!, entryFile: null });
  }
  return { routes };
}
```

```ts
// src/coverage/import-graph.ts
import * as nodePath from 'node:path';
import type { FsGlob } from '../contracts/index.js';

export interface ImportGraph {
  get(file: string): string[];
  unresolvable: string[];
}

const STATIC_IMPORT = /(?:^|\n)\s*import\s+(?:[^'"]+\s+from\s+)?['"]([^'"]+)['"]/g;
const DYNAMIC_IMPORT = /import\(['"]([^'"]+)['"]\)/g;

function extractSpecifiers(source: string): string[] {
  const specs: string[] = [];
  for (const re of [STATIC_IMPORT, DYNAMIC_IMPORT]) {
    re.lastIndex = 0;
    let m: RegExpExecArray | null;
    while ((m = re.exec(source)) !== null) specs.push(m[1]!);
  }
  return specs;
}

/** Resolve a relative specifier by probing which candidate file actually exists. */
async function resolveSpecifier(fs: FsGlob, base: string, spec: string): Promise<string | null> {
  if (!spec.startsWith('.')) return null; // alias or bare specifier: not resolvable here
  const resolved = nodePath.normalize(nodePath.join(nodePath.dirname(base), spec));
  if (/\.[a-z]+$/i.test(resolved)) return (await fs.readFile(resolved)) !== null ? resolved : null;
  for (const suffix of ['.tsx', '.ts', '.jsx', '.js', '/index.tsx', '/index.ts']) {
    const candidate = resolved + suffix;
    if ((await fs.readFile(candidate)) !== null) return candidate;
  }
  return null; // no candidate exists: unresolvable, never a guess
}

export async function buildImportGraph(fs: FsGlob, entryFiles: string[]): Promise<ImportGraph> {
  const edges = new Map<string, string[]>();
  const unresolvable: string[] = [];
  const queue = [...entryFiles];
  const visited = new Set<string>(queue);

  while (queue.length > 0) {
    const file = queue.shift()!;
    const source = await fs.readFile(file);
    if (source == null) continue;
    const children: string[] = [];
    for (const spec of extractSpecifiers(source)) {
      const resolved = await resolveSpecifier(fs, file, spec);
      if (resolved == null) {
        if (spec.startsWith('.') || spec.startsWith('@/') || spec.startsWith('~/')) {
          unresolvable.push(`${file}:${spec}`);
        }
        continue; // bare package imports are not app surfaces
      }
      children.push(resolved);
      if (!visited.has(resolved)) { visited.add(resolved); queue.push(resolved); }
    }
    edges.set(file, children);
  }

  return { get: (file) => edges.get(file) ?? [], unresolvable };
}
```

- [ ] **Step 4: Run: expected PASS**, then `npm run typecheck`.
- [ ] **Step 5: Commit** `feat: route manifest loader and existence-probing import graph`

---

## Task 2: Coverage planner: changed files → affected screens, gaps populated

**Files:**
- `src/coverage/planner.ts`
- `test/coverage/planner.test.ts`

The planner is the only caller of the route manifest and import graph. It produces a
fully-populated `Coverage`, including a `CoverageGap` with a written reason for every
UI file it cannot map. Only routes with a non-null `entryFile` participate in
route-graph matching.

- [ ] **Step 1: Write failing test**

```ts
// test/coverage/planner.test.ts
import { describe, it, expect } from 'vitest';
import { computeCoverage } from '../../src/coverage/planner.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { UsablConfig } from '../../src/contracts/index.js';

const fsOf = (files: Record<string, string>) => makeFakeDeps({ files }).fs;

const baseConfig: UsablConfig = {
  appBaseUrl: 'http://localhost:3000',
  uiFileGlobs: ['src/**/*.tsx'],
  discovery: { routerFile: 'src/router.tsx', wideBlastGlobs: ['src/App.tsx'] },
  surfaces: [],
  guardedPaths: ['usabl.config.json'],
};

describe('computeCoverage', () => {
  it('returns nothingToCheck when no UI files changed', async () => {
    const cov = await computeCoverage(fsOf({ 'src/router.tsx': '' }), baseConfig, ['docs/README.md']);
    expect(cov.nothingToCheck).toBe(true);
    expect(cov.gaps).toHaveLength(0);
  });

  it('maps a changed UI file to an affected screen via the sidecar route manifest', async () => {
    const fs = fsOf({
      'usabl.routes.json': JSON.stringify({
        routes: [{ screenId: 'clusters', url: '/clusters', entryFile: 'src/ClustersPage.tsx' }],
      }),
      'src/ClustersPage.tsx': `export default function ClustersPage() {}`,
    });
    const cov = await computeCoverage(fs, baseConfig, ['src/ClustersPage.tsx']);
    expect(cov.nothingToCheck).toBe(false);
    expect(cov.affected.some((s) => s.screenId === 'clusters' && s.provenance === 'route-graph')).toBe(true);
    expect(cov.unresolvedFiles).toHaveLength(0);
    expect(cov.gaps).toHaveLength(0);
  });

  it('populates a gap with a reason for a UI file that maps to no screen', async () => {
    const fs = fsOf({
      'usabl.routes.json': JSON.stringify({ routes: [] }),
      'src/Orphan.tsx': `export default function Orphan() {}`,
    });
    const cov = await computeCoverage(fs, baseConfig, ['src/Orphan.tsx']);
    expect(cov.unresolvedFiles).toContain('src/Orphan.tsx');
    expect(cov.gaps).toHaveLength(1);
    expect(cov.gaps[0]!.state).toBe('unresolved');
    expect(cov.gaps[0]!.reason).not.toBe('');
  });

  it('wide-blast: a changed global file touches every known route', async () => {
    const fs = fsOf({
      'usabl.routes.json': JSON.stringify({
        routes: [
          { screenId: 'clusters', url: '/clusters', entryFile: 'src/ClustersPage.tsx' },
          { screenId: 'alerts', url: '/alerts', entryFile: null },
        ],
      }),
      'src/App.tsx': `export default function App() {}`,
    });
    const cov = await computeCoverage(fs, baseConfig, ['src/App.tsx']);
    expect(cov.affected).toHaveLength(2);
    expect(cov.affected.every((s) => s.provenance === 'wide-blast')).toBe(true);
  });

  it('manual surface config is an additive fallback', async () => {
    const cfg: UsablConfig = {
      ...baseConfig,
      surfaces: [{ id: 'login', url: '/login', files: ['src/LoginPage.tsx'] }],
    };
    const fs = fsOf({
      'usabl.routes.json': JSON.stringify({ routes: [] }),
      'src/LoginPage.tsx': `export default function LoginPage() {}`,
    });
    const cov = await computeCoverage(fs, cfg, ['src/LoginPage.tsx']);
    expect(cov.affected.some((s) => s.screenId === 'login' && s.provenance === 'manual')).toBe(true);
  });
});
```

- [ ] **Step 2: Run: expected FAIL**

- [ ] **Step 3: Write implementation**

```ts
// src/coverage/planner.ts
import { parseRouteManifest } from './route-manifest.js';
import { buildImportGraph } from './import-graph.js';
import type { AffectedScreen, Coverage, CoverageGap, FsGlob, UsablConfig } from '../contracts/index.js';

function matchesAnyGlob(file: string, globs: string[]): boolean {
  return globs.some((g) => {
    const re = new RegExp('^' + g.replace(/[.+^${}()|[\]\\]/g, '\\$&').replace(/\*\*/g, '\x00').replace(/\*/g, '[^/]*').replace(/\x00/g, '.*') + '$');
    return re.test(file);
  });
}

function collectClosure(graph: { get(f: string): string[] }, entry: string): Set<string> {
  const seen = new Set<string>();
  const q = [entry];
  while (q.length > 0) {
    const cur = q.shift()!;
    if (seen.has(cur)) continue;
    seen.add(cur);
    for (const child of graph.get(cur)) q.push(child);
  }
  return seen;
}

export async function computeCoverage(
  fs: FsGlob,
  config: UsablConfig,
  changedFiles: string[],
): Promise<Coverage> {
  const uiFiles = changedFiles.filter((f) => matchesAnyGlob(f, config.uiFileGlobs));
  if (uiFiles.length === 0) {
    return { changedFiles, affected: [], unresolvedFiles: [], gaps: [], nothingToCheck: true };
  }

  const manifest = await parseRouteManifest(fs, config.discovery);
  const affected: AffectedScreen[] = [];
  const unresolvedFiles: string[] = [];
  const gaps: CoverageGap[] = [];
  const urlOf = (routeUrl: string): string => config.appBaseUrl.replace(/\/$/, '') + routeUrl;
  const addAffected = (s: AffectedScreen): void => {
    if (!affected.some((a) => a.screenId === s.screenId)) affected.push(s);
  };

  // 1. Wide-blast: shell/CSS/config changes affect every known route.
  const wideBlast = uiFiles.some((f) => matchesAnyGlob(f, config.discovery.wideBlastGlobs));
  if (wideBlast) {
    for (const route of manifest.routes) {
      addAffected({ screenId: route.screenId, url: urlOf(route.url), provenance: 'wide-blast' });
    }
  }

  // 2. Route-graph: only routes that attribute an entry file participate.
  const routed = manifest.routes.filter((r): r is typeof r & { entryFile: string } => r.entryFile !== null);
  const graph = routed.length > 0 ? await buildImportGraph(fs, routed.map((r) => r.entryFile)) : null;

  for (const changed of uiFiles.filter((f) => !matchesAnyGlob(f, config.discovery.wideBlastGlobs))) {
    let matched = false;
    if (graph) {
      for (const route of routed) {
        const closure = collectClosure(graph, route.entryFile);
        if (closure.has(changed)) {
          addAffected({
            screenId: route.screenId, url: urlOf(route.url),
            provenance: 'route-graph', importChain: [route.entryFile, changed],
          });
          matched = true;
        }
      }
    }
    if (!matched) {
      // 3. Manual surfaces: additive fallback, never a replacement.
      const manual = config.surfaces.find((s) => s.files.includes(changed));
      if (manual) {
        addAffected({ screenId: manual.id, url: manual.url, provenance: 'manual' });
      } else {
        unresolvedFiles.push(changed);
        gaps.push({
          ref: changed,
          state: 'unresolved',
          reason: `UI file '${changed}' does not appear in any route entry-file closure, wide-blast glob, or manual surface entry`,
        });
      }
    }
  }

  return { changedFiles, affected, unresolvedFiles, gaps, nothingToCheck: false };
}
```

- [ ] **Step 4: Run: expected PASS**, then `npm run typecheck`.
- [ ] **Step 5: Commit** `feat: coverage planner with gap reasons and honest route attribution`

---

## Task 3: Hardened guard: config-guards-itself first, directory expansion, session pins

**Files:**
- `src/trust/guard.ts`
- `test/trust/guard.test.ts`

The two properties this task must prove:

1. **Config-guards-itself, ordering load-bearing.** The config FILE is checked against
   HEAD before its contents are trusted for anything. A tampered config that removes
   itself (or anything else) from `guardedPaths` still forces `approval_required`,
   because the config check does not depend on the config's own contents.
2. **Directory expansion.** A guarded directory covers every file under it, committed
   or new: the expansion is the union of `git.lsFiles(ref, prefix)` and
   `fs.glob([prefix, prefix + '/**'])`. Editing or adding any file under `src/gate`
   diverges the guard. This closes the engine-self-edit hole.

- [ ] **Step 1: Write failing tests**

```ts
// test/trust/guard.test.ts
import { describe, it, expect } from 'vitest';
import { CONFIG_PATH, buildGuardedSet, expandGuardedSet, checkGuard, computeSessionPins, diffSessionPins } from '../../src/trust/guard.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://localhost:3000',
  uiFileGlobs: ['src/**/*.tsx'],
  discovery: { routerFile: 'src/router.tsx', wideBlastGlobs: [] },
  surfaces: [],
  guardedPaths: ['usabl.config.json', 'src/gate'],
  requirements: 'requirements/',
};

const cleanPolicy = {
  'usabl.config.json': '{"v":1}',
  '.usabl-evidence.json': '{}',
  '.usabl-waivers.json': '{}',
  'src/gate/index.ts': 'GATE',
};

describe('buildGuardedSet', () => {
  it('always includes config, evidence, waivers, and the requirements dir, regardless of guardedPaths', () => {
    const stripped: UsablConfig = { ...config, guardedPaths: [] };
    const guarded = buildGuardedSet(stripped);
    expect(guarded).toContain(CONFIG_PATH);
    expect(guarded).toContain('.usabl-evidence.json');
    expect(guarded).toContain('.usabl-waivers.json');
    expect(guarded).toContain('requirements/');
  });
});

describe('expandGuardedSet', () => {
  it('expands a guarded directory to committed files plus new working-tree files under it', async () => {
    const deps = makeFakeDeps({
      headContents: { ...cleanPolicy },
      files: { ...cleanPolicy, 'src/gate/new-module.ts': 'NEW' },
    });
    const expanded = await expandGuardedSet(deps, buildGuardedSet(config));
    expect(expanded).toContain('src/gate/index.ts');       // committed under the dir
    expect(expanded).toContain('src/gate/new-module.ts');  // new in the working tree
    expect(expanded).toContain(CONFIG_PATH);               // plain file entries survive
  });
});

describe('checkGuard', () => {
  it('reports no divergence when the working tree matches HEAD', async () => {
    const deps = makeFakeDeps({ headContents: cleanPolicy, files: cleanPolicy });
    expect(await checkGuard(deps, config)).toEqual([]);
  });

  it('short-circuits on a tampered config BEFORE trusting its guardedPaths (the bypass test)', async () => {
    // The working-tree config removed itself and everything else from guardedPaths.
    const tampered: UsablConfig = { ...config, guardedPaths: [] };
    const deps = makeFakeDeps({
      headContents: cleanPolicy,
      files: { ...cleanPolicy, 'usabl.config.json': '{"v":2,"guardedPaths":[]}' },
    });
    const diverged = await checkGuard(deps, tampered);
    expect(diverged).toEqual([CONFIG_PATH]);
  });

  it('detects an edit to a file INSIDE a guarded directory', async () => {
    const deps = makeFakeDeps({
      headContents: cleanPolicy,
      files: { ...cleanPolicy, 'src/gate/index.ts': 'EDITED' },
    });
    expect(await checkGuard(deps, config)).toContain('src/gate/index.ts');
  });

  it('detects a NEW file added under a guarded directory', async () => {
    const deps = makeFakeDeps({
      headContents: cleanPolicy,
      files: { ...cleanPolicy, 'src/gate/backdoor.ts': 'X' },
    });
    expect(await checkGuard(deps, config)).toContain('src/gate/backdoor.ts');
  });
});

describe('session pins', () => {
  it('pins every expanded guarded path to a sha of its committed content', async () => {
    const deps = makeFakeDeps({ headContents: cleanPolicy, files: cleanPolicy });
    const pins = await computeSessionPins(deps, config);
    expect(pins['usabl.config.json']).toMatch(/^[0-9a-f]{64}$/);
    expect(pins['src/gate/index.ts']).toMatch(/^[0-9a-f]{64}$/);
  });

  it('diffSessionPins reports paths whose committed content changed mid-session', async () => {
    const before = await computeSessionPins(makeFakeDeps({ headContents: cleanPolicy, files: cleanPolicy }), config);
    const afterDeps = makeFakeDeps({
      headContents: { ...cleanPolicy, 'src/gate/index.ts': 'COMMITTED-TAMPER' },
      files: { ...cleanPolicy, 'src/gate/index.ts': 'COMMITTED-TAMPER' },
    });
    const after = await computeSessionPins(afterDeps, config);
    expect(diffSessionPins(before, after)).toEqual(['src/gate/index.ts']);
  });
});
```

- [ ] **Step 2: Run: expected FAIL**

- [ ] **Step 3: Write implementation**

```ts
// src/trust/guard.ts
import { createHash } from 'node:crypto';
import type { Deps, UsablConfig } from '../contracts/index.js';

export const CONFIG_PATH = 'usabl.config.json';
const ALWAYS_GUARDED = [CONFIG_PATH, '.usabl-evidence.json', '.usabl-waivers.json'];

const sha256 = (content: string): string => createHash('sha256').update(content, 'utf8').digest('hex');

/**
 * The complete guarded set: config, evidence, and waiver files are ALWAYS included,
 * plus the requirements directory when configured, plus config.guardedPaths.
 * A config cannot edit anything out of the unconditional core.
 */
export function buildGuardedSet(config: UsablConfig): string[] {
  const set = new Set<string>([
    ...ALWAYS_GUARDED,
    ...(config.requirements ? [config.requirements] : []),
    ...config.guardedPaths,
  ]);
  return [...set].sort();
}

/**
 * Expand directory entries to files: union of committed files under the prefix
 * (git.lsFiles) and working-tree files under it (fs.glob), so edits AND additions
 * both diverge. Plain file entries pass through. Real lsFiles semantics are git
 * pathspec listing: `git ls-tree -r --name-only HEAD -- <prefix>`.
 */
export async function expandGuardedSet(deps: Deps, guardedSet: string[]): Promise<string[]> {
  const out = new Set<string>();
  for (const entry of guardedSet) {
    const prefix = entry.replace(/\/$/, '');
    const committed = await deps.git.lsFiles('HEAD', prefix);
    const working = await deps.fs.glob([prefix, `${prefix}/**`]);
    if (committed.length === 0 && working.length === 0) { out.add(entry); continue; }
    for (const f of [...committed, ...working]) out.add(f);
  }
  return [...out].sort();
}

async function diverges(deps: Deps, path: string): Promise<boolean> {
  const [working, committed] = await Promise.all([deps.fs.readFile(path), deps.git.show('HEAD', path)]);
  if (working === null && committed === null) return false; // absent everywhere: nothing to compare
  return working !== committed;
}

/**
 * ORDERING IS LOAD-BEARING: the config file is verified against HEAD before its
 * contents are trusted to name the rest of the guarded set. A tampered config
 * short-circuits to approval_required no matter what it says about guardedPaths.
 */
export async function checkGuard(deps: Deps, config: UsablConfig): Promise<string[]> {
  if (await diverges(deps, CONFIG_PATH)) return [CONFIG_PATH];
  const expanded = await expandGuardedSet(deps, buildGuardedSet(config));
  const diverged: string[] = [];
  await Promise.all(expanded.map(async (path) => {
    if (await diverges(deps, path)) diverged.push(path);
  }));
  return diverged.sort();
}

/** path → sha256(committed content) over the EXPANDED guarded set. */
export async function computeSessionPins(deps: Deps, config: UsablConfig): Promise<Record<string, string>> {
  const expanded = await expandGuardedSet(deps, buildGuardedSet(config));
  const pins: Record<string, string> = {};
  await Promise.all(expanded.map(async (path) => {
    pins[path] = sha256((await deps.git.show('HEAD', path)) ?? '');
  }));
  return pins;
}

/** Paths whose pinned committed content changed since the pins were taken. */
export function diffSessionPins(saved: Record<string, string>, current: Record<string, string>): string[] {
  const changed: string[] = [];
  for (const [path, pin] of Object.entries(saved)) {
    if (current[path] !== pin) changed.push(path);
  }
  for (const path of Object.keys(current)) {
    if (!(path in saved)) changed.push(path);
  }
  return changed.sort();
}
```

Pin STORAGE (a tmpdir file keyed by session id, written at the first stop-hook run,
checked on later runs) lives with the stop-hook runner in Phase 5; this module stays
pure.

- [ ] **Step 4: Run: expected PASS**, then `npm run typecheck`.
- [ ] **Step 5: Commit** `feat: hardened guard with config-first ordering, directory expansion, session pins`

---

## Task 4: Receipt verification through the single policy hash

**Files:**
- `src/evidence/receipt.ts` (extend; `computePolicyHash` and `mintReceipt` exist from Phase 1 and are NOT redefined)
- `test/evidence/receipt-verify.test.ts`

`verifyReceipt` recomputes exactly what minting computed: `computePolicyHash` over the
EXPANDED guarded set, the current `writeTree`, and `deps.runnerVersion`. Same function,
same inputs; a receipt minted by `run()` re-verifies bit for bit.

- [ ] **Step 1: Write failing tests**

```ts
// test/evidence/receipt-verify.test.ts
import { describe, it, expect } from 'vitest';
import { mintReceipt, verifyReceipt } from '../../src/evidence/receipt.js';
import { buildGuardedSet, expandGuardedSet } from '../../src/trust/guard.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://localhost:3000',
  uiFileGlobs: ['src/**/*.tsx'],
  discovery: { routerFile: 'src/router.tsx', wideBlastGlobs: [] },
  surfaces: [],
  guardedPaths: ['usabl.config.json', 'src/gate'],
};

const policy = {
  'usabl.config.json': 'cfg', '.usabl-evidence.json': '{}', '.usabl-waivers.json': '{}',
  'src/gate/index.ts': 'GATE',
};

async function mintOn(deps: ReturnType<typeof makeFakeDeps>) {
  const expanded = await expandGuardedSet(deps, buildGuardedSet(config));
  return mintReceipt(deps, { ...config, guardedPaths: expanded }, {
    surfaces: ['cli'], checked: ['clusters'], notCovered: [],
    findingsSummary: { new: 0, carried: 0, fixed: 0, unverified: 0 }, activeWaivers: 0,
  });
}

describe('verifyReceipt', () => {
  it('a receipt minted by the engine re-verifies against the same state', async () => {
    const deps = makeFakeDeps({ headContents: policy, files: policy, writeTree: 'tree-a', runnerVersion: '0.1.0' });
    const receipt = await mintOn(deps);
    const check = await verifyReceipt(deps, config, receipt, 'tree-a');
    expect(check.valid).toBe(true);
    expect(check.failedFields).toEqual([]);
  });

  it('fails on sourceTree divergence only, with policy intact', async () => {
    const deps = makeFakeDeps({ headContents: policy, files: policy, writeTree: 'tree-a', runnerVersion: '0.1.0' });
    const receipt = await mintOn(deps);
    const check = await verifyReceipt(deps, config, receipt, 'tree-CHANGED');
    expect(check.valid).toBe(false);
    expect(check.failedFields).toEqual(['sourceTree']);
  });

  it('fails on policyHash when a guarded file changed at HEAD (committed policy edit)', async () => {
    const deps1 = makeFakeDeps({ headContents: policy, files: policy, writeTree: 'tree-a', runnerVersion: '0.1.0' });
    const receipt = await mintOn(deps1);
    const deps2 = makeFakeDeps({
      headContents: { ...policy, 'src/gate/index.ts': 'EDITED-GATE' },
      files: { ...policy, 'src/gate/index.ts': 'EDITED-GATE' },
      writeTree: 'tree-a', runnerVersion: '0.1.0',
    });
    const check = await verifyReceipt(deps2, config, receipt, 'tree-a');
    expect(check.failedFields).toContain('policyHash');
  });

  it('fails on runnerVersion drift', async () => {
    const deps = makeFakeDeps({ headContents: policy, files: policy, writeTree: 'tree-a', runnerVersion: '0.1.0' });
    const receipt = await mintOn(deps);
    const newer = makeFakeDeps({ headContents: policy, files: policy, writeTree: 'tree-a', runnerVersion: '0.2.0' });
    const check = await verifyReceipt(newer, config, receipt, 'tree-a');
    expect(check.failedFields).toContain('runnerVersion');
  });
});
```

- [ ] **Step 2: Run: expected FAIL**

- [ ] **Step 3: Add `verifyReceipt` to `src/evidence/receipt.ts`** (additions only; do
  not touch `computePolicyHash` or `mintReceipt`):

```ts
// --- addition to src/evidence/receipt.ts ---
import { buildGuardedSet, expandGuardedSet } from '../trust/guard.js';

export interface ReceiptVerification {
  valid: boolean;
  failedFields: string[];
}

/**
 * Re-verify a receipt against the current state, using THE SAME computePolicyHash
 * over THE SAME expanded guarded set that minting used. currentSourceTree is passed
 * in (the caller already ran writeTree).
 */
export async function verifyReceipt(
  deps: Deps,
  config: UsablConfig,
  receipt: Receipt,
  currentSourceTree: string,
): Promise<ReceiptVerification> {
  const expanded = await expandGuardedSet(deps, buildGuardedSet(config));
  const currentPolicy = await computePolicyHash(deps, expanded);
  const failedFields: string[] = [];
  if (receipt.sourceTree !== currentSourceTree) failedFields.push('sourceTree');
  if (receipt.policyHash !== currentPolicy) failedFields.push('policyHash');
  if (receipt.runnerVersion !== deps.runnerVersion) failedFields.push('runnerVersion');
  return { valid: failedFields.length === 0, failedFields };
}
```

- [ ] **Step 4: Run: expected PASS**, then `npm run typecheck`.
- [ ] **Step 5: Commit** `feat: verifyReceipt through the single policy hash over the expanded guarded set`

---

## Task 5: Wire planner, hardened guard, and expanded receipt into `run()`

**Files:**
- `src/run.ts` (modify internals; the `run(deps, config, opts?)` signature does NOT change)
- `test/run-phase3.test.ts`

Changes inside `run()`:
1. Replace the Phase 1 direct-mapping `computeCoverage` with the planner
   (`src/coverage/planner.ts`).
2. Replace `computeGuardDivergence` with `checkGuard` (config-first, expanded).
3. Mint the receipt over the EXPANDED guarded set:
   `mintReceipt(deps, { ...config, guardedPaths: expanded }, args)`.
4. Trusted-ref floor/waiver reads already exist from Phase 1 (`readPolicyJson`).
   CI mints no receipts on PRs, so trusted-ref policy hashing at mint time is a
   documented seam, not contest work.

- [ ] **Step 1: Write failing tests** (note the identity keys: `button-name` is
  identity-weak, so its floor entry is COUNT-based with `elementKey: null`; a
  name-based entry uses the `computeIdentity` key shape `screen|rule|name:slug`):

```ts
// test/run-phase3.test.ts
import { describe, it, expect } from 'vitest';
import { run } from '../../src/run.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { Draft, EvidenceFloor, ScreenScan, UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://localhost:3000',
  uiFileGlobs: ['src/**/*.tsx'],
  discovery: { routerFile: 'src/router.tsx', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: 'http://localhost:3000/clusters', files: ['src/ClustersPage.tsx'] }],
  guardedPaths: ['usabl.config.json'],
};

const cleanPolicy = {
  'usabl.config.json': '{}', '.usabl-evidence.json': '{}', '.usabl-waivers.json': '{}',
};

const unnamedButton: Draft = {
  rule: 'button-name', layer: 'axe', severity: 'critical', evidenceClass: 'deterministic',
  screenId: 'clusters', elementPath: 'button:nth-child(1)', elementName: null, role: 'button',
  whatUserExperiences: 'no accessible name', why: 'icon-only', fix: 'add aria-label',
  evidence: {}, confidence: 'fail',
};

const namedContrast: Draft = {
  rule: 'color-contrast', layer: 'axe', severity: 'serious', evidenceClass: 'deterministic',
  screenId: 'clusters', elementPath: 'h1:nth-child(1)', elementName: 'Clusters', role: 'heading',
  whatUserExperiences: 'low contrast', why: 'ratio 2.5', fix: 'raise contrast',
  evidence: { name: { value: 'Clusters', source: 'ax-tree', fromTree: true } }, confidence: 'fail',
};

const scanOf = (drafts: Draft[]): ScreenScan =>
  ({ screenId: 'clusters', url: 'http://localhost:3000/clusters', stops: [], drafts, gaps: [] });

describe('run() with Phase 3 wiring', () => {
  it('is idle when no UI files changed and policy is clean', async () => {
    const deps = makeFakeDeps({ files: cleanPolicy, headContents: cleanPolicy });
    const r = await run(deps, config, { changedFiles: ['README.md'] });
    expect(r.verdict).toBeNull();
    expect(r.exitCode).toBe(0);
  });

  it('is approval_required when the config diverges, even on a docs-only diff', async () => {
    const deps = makeFakeDeps({
      files: { ...cleanPolicy, 'usabl.config.json': '{"tampered":true}' },
      headContents: cleanPolicy,
    });
    const r = await run(deps, config, { changedFiles: ['README.md'] });
    expect(r.verdict).toBe('approval_required');
    expect(r.dirtyGuardedPaths).toContain('usabl.config.json');
    expect(r.exitCode).toBe(2);
  });

  it('is not_covered with a written gap when a UI file maps to nothing', async () => {
    const deps = makeFakeDeps({
      files: { ...cleanPolicy, 'usabl.routes.json': JSON.stringify({ routes: [] }), 'src/Orphan.tsx': 'x' },
      headContents: cleanPolicy,
    });
    const r = await run(deps, config, { changedFiles: ['src/Orphan.tsx'] });
    expect(r.verdict).toBe('not_covered');
    expect(r.exitCode).toBe(3);
    expect(r.coverage.gaps.length).toBeGreaterThan(0);
  });

  it('converges to verified when the floor carries a count-based accepted finding', async () => {
    const floor: EvidenceFloor = {
      version: 1,
      entries: [{ screenId: 'clusters', layer: 'axe', rule: 'button-name', elementKey: null, identityBasis: 'count', count: 1 }],
    };
    const files = { ...cleanPolicy, '.usabl-evidence.json': JSON.stringify(floor), 'src/ClustersPage.tsx': 'x' };
    const deps = makeFakeDeps({
      files, headContents: files, writeTree: 'tree-accept',
      scans: { clusters: scanOf([unnamedButton]) },
    });
    const r = await run(deps, config, { changedFiles: ['src/ClustersPage.tsx'] });
    expect(r.verdict).toBe('verified');
    expect(r.receipt).not.toBeNull();
    expect(r.receipt!.sourceTree).toBe('tree-accept');
  });

  it('converges to verified when the floor carries a name-keyed accepted finding', async () => {
    const floor: EvidenceFloor = {
      version: 1,
      entries: [{ screenId: 'clusters', layer: 'axe', rule: 'color-contrast', elementKey: 'clusters|color-contrast|name:clusters', identityBasis: 'name', count: 1 }],
    };
    const files = { ...cleanPolicy, '.usabl-evidence.json': JSON.stringify(floor), 'src/ClustersPage.tsx': 'x' };
    const deps = makeFakeDeps({
      files, headContents: files,
      scans: { clusters: scanOf([namedContrast]) },
    });
    const r = await run(deps, config, { changedFiles: ['src/ClustersPage.tsx'] });
    expect(r.verdict).toBe('verified');
    expect(r.findings.find((f) => f.rule === 'color-contrast')!.status).toBe('carried');
  });

  it('a PR cannot self-accept: with trustedRef, the working-tree floor is ignored AND its edit blocks', async () => {
    const acceptedFloor: EvidenceFloor = {
      version: 1,
      entries: [{ screenId: 'clusters', layer: 'axe', rule: 'button-name', elementKey: null, identityBasis: 'count', count: 1 }],
    };
    const head = { ...cleanPolicy, 'src/ClustersPage.tsx': 'x' };
    const deps = makeFakeDeps({
      files: { ...head, '.usabl-evidence.json': JSON.stringify(acceptedFloor) }, // PR tries to self-accept
      headContents: head,                                                        // trusted ref: empty floor
      scans: { clusters: scanOf([unnamedButton]) },
    });
    const r = await run(deps, config, { changedFiles: ['src/ClustersPage.tsx'], trustedRef: 'HEAD' });
    // The uncommitted floor edit diverges the guard; and even without the guard,
    // the trusted-ref floor is empty so the finding would regress. Either way: never verified.
    expect(r.verdict).not.toBe('verified');
    expect(r.receipt).toBeNull();
  });
});
```

- [ ] **Step 2: Run: expected FAIL**

- [ ] **Step 3: Update `src/run.ts` internals**

```ts
// Replace the Phase 1 stub imports:
import { computeCoverage } from './coverage/planner.js';
import { checkGuard, buildGuardedSet, expandGuardedSet } from './trust/guard.js';

// Inside run(): coverage from the planner (async now), guard via checkGuard,
// and the receipt minted over the expanded guarded set:
const baseCoverage = await computeCoverage(deps.fs, config, changed);
const guardDivergedPaths = await checkGuard(deps, config);
// ... scans, gap merge, floor/waivers via readPolicyJson (unchanged from Phase 1) ...
const receipt = gated.verdict === 'verified'
  ? await mintReceipt(
      deps,
      { ...config, guardedPaths: await expandGuardedSet(deps, buildGuardedSet(config)) },
      { surfaces: ['cli'], checked: coverage.affected.map((a) => a.screenId),
        notCovered: coverage.unresolvedFiles, findingsSummary: summarize(gated.findings),
        activeWaivers: gated.findings.filter((f) => f.status === 'waived').length },
    )
  : null;
```

- [ ] **Step 4: Run the new tests, then the FULL suite.** Phase 1's run tests and the
  golden oracle must still pass. The oracle's canonical output changes shape only if a
  fixture was wrong; regenerate with `UPDATE_GOLDEN=1` ONLY after eyeballing the diff.

```bash
npx vitest run test/run-phase3.test.ts && npx vitest run && npm run typecheck
```

- [ ] **Step 5: Commit** `feat: wire planner, hardened guard, and expanded receipt into run()`

---

## Task 6: Exit-criterion integration: policy edit blocks, accept converges, receipt re-verifies, bypass is closed

**Files:**
- `test/integration/phase3-exit-criterion.test.ts`

The full lifecycle over `makeFakeDeps`, including the security regression tests for the
config self-guard bypass and the engine-self-edit hole.

- [ ] **Step 1: Write the integration test**

```ts
// test/integration/phase3-exit-criterion.test.ts
import { describe, it, expect } from 'vitest';
import { run } from '../../src/run.js';
import { verifyReceipt } from '../../src/evidence/receipt.js';
import { makeFakeDeps } from '../../src/deps/fakes.js';
import type { Draft, EvidenceFloor, ScreenScan, UsablConfig } from '../../src/contracts/index.js';

const config: UsablConfig = {
  appBaseUrl: 'http://localhost:3000',
  uiFileGlobs: ['src/**/*.tsx'],
  discovery: { routerFile: 'src/router.tsx', wideBlastGlobs: [] },
  surfaces: [{ id: 'clusters', url: 'http://localhost:3000/clusters', files: ['src/ClustersPage.tsx'] }],
  guardedPaths: ['usabl.config.json', 'src/gate'],
};

const iconDraft: Draft = {
  rule: 'button-name', layer: 'axe', severity: 'critical', evidenceClass: 'deterministic',
  screenId: 'clusters', elementPath: 'button:nth-child(1)', elementName: null, role: 'button',
  whatUserExperiences: 'no accessible name', why: 'icon-only', fix: 'add aria-label',
  evidence: {}, confidence: 'fail',
};
const scanOf = (drafts: Draft[]): ScreenScan =>
  ({ screenId: 'clusters', url: 'http://localhost:3000/clusters', stops: [], drafts, gaps: [] });

const committed = {
  'usabl.config.json': '{"v":1}', '.usabl-evidence.json': '{}', '.usabl-waivers.json': '{}',
  'src/gate/index.ts': 'GATE', 'src/ClustersPage.tsx': 'PAGE',
};

describe('Phase 3 exit criterion', () => {
  it('an uncommitted policy edit forces approval_required and mints nothing', async () => {
    const deps = makeFakeDeps({
      headContents: committed,
      files: { ...committed, '.usabl-waivers.json': '{"version":1,"waivers":[]}' }, // edited, uncommitted
      scans: { clusters: scanOf([iconDraft]) },
    });
    const r = await run(deps, config, { changedFiles: ['src/ClustersPage.tsx'] });
    expect(r.verdict).toBe('approval_required');
    expect(r.receipt).toBeNull();
  });

  it('SECURITY: a config that drops itself from guardedPaths is still caught', async () => {
    const strippedConfig: UsablConfig = { ...config, guardedPaths: [] };
    const deps = makeFakeDeps({
      headContents: committed,
      files: { ...committed, 'usabl.config.json': '{"v":2,"guardedPaths":[]}' },
      scans: { clusters: scanOf([]) },
    });
    const r = await run(deps, strippedConfig, { changedFiles: ['src/ClustersPage.tsx'] });
    expect(r.verdict).toBe('approval_required');
    expect(r.dirtyGuardedPaths).toEqual(['usabl.config.json']);
    expect(r.receipt).toBeNull();
  });

  it('SECURITY: editing engine source under a guarded directory blocks', async () => {
    const deps = makeFakeDeps({
      headContents: committed,
      files: { ...committed, 'src/gate/index.ts': 'PATCHED-GATE' },
      scans: { clusters: scanOf([]) },
    });
    const r = await run(deps, config, { changedFiles: ['src/ClustersPage.tsx'] });
    expect(r.verdict).toBe('approval_required');
    expect(r.dirtyGuardedPaths).toContain('src/gate/index.ts');
  });

  it('the accept commit converges the next run to verified and the receipt re-verifies', async () => {
    const acceptedFloor: EvidenceFloor = {
      version: 1,
      entries: [{ screenId: 'clusters', layer: 'axe', rule: 'button-name', elementKey: null, identityBasis: 'count', count: 1 }],
    };
    const accepted = { ...committed, '.usabl-evidence.json': JSON.stringify(acceptedFloor) };
    const deps = makeFakeDeps({
      headContents: accepted, files: accepted, writeTree: 'tree-accepted', runnerVersion: '0.1.0',
      scans: { clusters: scanOf([iconDraft]) },
    });
    const r = await run(deps, config, { changedFiles: ['src/ClustersPage.tsx'] });
    expect(r.verdict).toBe('verified');
    expect(r.exitCode).toBe(0);
    expect(r.receipt).not.toBeNull();

    const check = await verifyReceipt(deps, config, r.receipt!, 'tree-accepted');
    expect(check.valid).toBe(true);
    expect(check.failedFields).toEqual([]);
  });
});
```

- [ ] **Step 2: Run: fix any wiring gap in `src/run.ts` until all scenarios pass**
- [ ] **Step 3: Full suite + typecheck**

```bash
npx vitest run && npm run typecheck
```

- [ ] **Step 4: Commit** `test: Phase 3 exit criterion incl. config self-guard and directory-guard security cases`

---

## Self-Review

**Coverage:** planner produces route-graph, wide-blast, and manual provenance; every
unmapped UI file carries a `CoverageGap` with a written reason; regex-fallback routes
never pretend to attribute entry files; the import graph probes file existence instead
of guessing extensions.

**Guard:** the config file is checked FIRST and unconditionally; the bypass (config
edits itself out of `guardedPaths`) has a dedicated security test; directory guarded
paths expand to committed plus working-tree files, closing the engine-self-edit hole;
session pins cover the expanded set, and `diffSessionPins` is pure (storage is the
Phase 5 stop-hook runner's job).

**Receipt:** exactly one policy hash (`computePolicyHash`, defined in Phase 1) computed
over the expanded guarded set at mint AND at verify; `verifyReceipt` also checks
`runnerVersion`. A receipt minted by `run()` re-verifies in the integration test.

**run():** the frozen `run(deps, config, opts?)` signature is untouched; trusted-ref
floor/waiver reads come from Phase 1's `readPolicyJson`; the trusted-ref test proves a
PR cannot self-accept its own floor.

**Floors:** accept only `confidence: 'fail'` findings. Stated in Phase 1 Task 8 and
respected here: no test writes an unverified finding into a floor.

**Fakes:** every test uses `makeFakeDeps`. No hand-rolled `Deps`, no `as any`, no
invented `BrowserDriver` shapes.
