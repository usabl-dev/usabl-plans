# usabl Implementation Plan: Overview and Build Order

> **Location note:** This plan set lives in the planning directory, NOT in the `usabl`
> or `usabl-app` repos. Source of truth for decisions is
> `../usabl-decisions-2026-08-19.md`; the in-repo canonical spec is
> `usabl/docs/ground-truth.md`. Do not commit these plan files to any shared repo.

**Goal:** Build a working prototype of usabl: an accessibility proof engine that,
for a given code state, returns exactly one of four verdicts (`verified`,
`regression`, `not_covered`, `approval_required`) plus an idle non-verdict, backed by
a re-verifiable receipt. Once the prototype stands end to end, the team iterates on
detection breadth and surfaces.

**Architecture (non-negotiable, from ground-truth §4):**
- One pure `run(deps, config)` over an injected `Deps` object. No hidden I/O.
- Every check is a `Provider` that returns `Draft[]`. Providers never decide anything.
- The **gate** is the single verdict authority. Nothing else mints a verdict.
- Each finding carries an `evidenceClass`, which is a PROVENANCE taxonomy
  (`deterministic | preview | model-judgment | human-confirmed`), not a caste system.
  The engine produces TWO first-class outputs from one run: (a) the reproducible gate
  verdict, minted only from `deterministic` (or promoted / `human-confirmed`) evidence
  and backed by the re-verifiable receipt; and (b) the judgment assessment, from
  `model-judgment` + `preview`. `verified` and the receipt stay RESERVED for
  reproducible evidence (the moat): raw model-judgment never mints `verified` and never
  writes a receipt. Judgment is first-class in visibility and influence via three honest
  paths (sharpen `not_covered`, opt-in soft-gate on a distinct exit code, calibrated
  promotion), not in the right to mint the reproducible verdict. See decision-log
  Section 8.2 for the full rule.
- The canonical `Result` JSON is the single source of truth. Every surface (CLI, stop
  hook, overlay, CI comment, Playwright helper, and the non-gating conformance summary)
  is a pure projection of it.

**Scope split (decision-log Section 8.1):** this plan set builds the CONTEST slice
(deterministic core + gate + receipt; ONE real model-judgment provider wired end to
end; the promotion seam present but empty; the voicing tier at `preview`; the conformance
summary + a draft ACR + an ACCESSIBILITY.md row). The FULL product (many providers, the
ecosystem provider-wrapping set, the calibrated promotion path populated, ACR
generation) is the spec in `ground-truth.md`, not this plan. What flexes for the contest
is breadth, never the architecture or the honesty invariants above.

**Cross-phase frozen seams (freeze centrally so parallel plans agree):** the contracts in
`01-core-foundation.md` Task 2 are frozen (Draft, Finding, Result, Deps, Page, BrowserDriver,
CheckRunner, ScreenScan, TranscriptStop, Step, InteractionContract, SpeechObligation,
EvidenceFloor, Waiver, UsablConfig, ConformanceSummary, GateInput/Output). Two additional
seams are frozen here because they cross phase boundaries; consumers must use these verbatim,
not reinvent them:

```ts
// Phase 2 defines these; Phases 4, 5, 7 consume them. A Provider is one check and NEVER
// decides a verdict: it returns Draft[] only. The real CheckRunner (Phase 2) opens a Page
// per screen via BrowserDriver, runs every Provider, and collects Draft[] (plus transcript
// stops) into the frozen ScreenScan that run() already consumes.
export interface ProviderContext { page: Page; screen: { id: string; url: string }; config: UsablConfig; }
export type Capability = 'live' | 'network' | 'secrets' | 'filesystem-write';
export interface Provider {
  id: string;
  layer: string;
  capabilities: Capability[];   // declared execution needs; static mode runs only providers whose capabilities exclude 'live'
  run(ctx: ProviderContext): Promise<Draft[]>;
}

// The keyboard-walk step runner (Phase 2), reused by the voicing lane (Phase 4). Executes an
// ordered list of Steps against a Page and records what an AT would announce at each stop.
export interface StepRunner { run(page: Page, steps: Step[]): Promise<TranscriptStop[]>; }
```

Static mode is capability filtering, not a separate code path: a provider skipped for lacking a permitted capability records a coverage gap with reason 'capability-denied', never a silent pass.

**Reconciliation deltas (2026-08-20, applied across all plans).** These are part of the frozen
contract set in `01-core-foundation.md` Task 2. Consumers use them verbatim:

- `ScreenScan` carries `gaps: CoverageGap[]`. Provider capability skips and per-screen scan
  failures land there; `run()` merges every screen's gaps into `coverage.gaps` before the gate.
  The gate treats any gap as blocking (`not_covered`) from day one.
- `AnnouncementToken.kind` gains `'live'`: text that reached an armed live region after a step,
  captured from DOM mutations. Live tokens are how toast and status announcements enter the
  transcript; the voicing lane matches obligations against name/role/state AND live tokens.
  `assert-announced` from ground-truth §7.3 is realized as a `SpeechObligation` over live tokens,
  not as a step kind.
- `Page` gains five methods: `click(selector)`, `activeElementIs(selector)`,
  `activeElementWithin(selector)`, `armAnnouncementCapture()`, `drainAnnouncements()`. These are
  the honest primitives for interaction rules (modal focus return, focus into dialog) and
  live-region capture.
- `BrowserDriver` gains `close()`: one warm browser per process, one context per `open(url)`,
  `close()` disposes the browser.
- `GitReader.lsFiles(ref, prefix)` replaces `lsTree`: list files under a prefix at a ref. Used to
  expand directory guarded paths and to build the policy hash.
- `run(deps, config, opts?)` takes an optional third argument
  `RunOptions { changedFiles?: string[]; trustedRef?: string | null }`. Default behavior is
  unchanged (changed files from `git status`, policy from the working tree). CI passes both.
- There is exactly ONE policy hash definition: `computePolicyHash` in `evidence/receipt.ts`,
  a sha256 over the canonicalized, sorted `[path, sha256(committed content)]` pairs of the
  expanded guarded set. Minting and verification both call it. No second algorithm exists.
- The real `BrowserDriver` reads names, roles, and states from the accessibility tree over CDP.
  A DOM approximation (aria-label || innerText) is forbidden: the product's claim is "what the
  screen reader gets," and the transcript must come from the AX tree.
- The demo fixture targets PatternFly v6 (`pf-v6-*` classes). No `pf-v5` selectors anywhere.
- Rules that only exist through interaction (modal focus return, focus into dialog, menu focus)
  are interaction probes that click and key through the widget. A static one-shot DOM pass cannot
  check them and must not pretend to.

**Tech stack:**
- TypeScript (ESM, strict), Node 22.
- Vitest for all tests. Every test runs on in-memory fakes; no network, no real browser
  in unit tests.
- tsup for the build (single `usabl` package, subpath exports, one `usabl` bin).
- Playwright drives the real browser behind the `BrowserDriver` interface (real Deps
  only; never in unit tests), reading names/roles/states from the AX tree over CDP.
- axe-core + `@axe-core/playwright` (WCAG baseline provider). The voicing lane's
  in-loop evidence is the AX transcript plus live-region tokens; screen-reader
  automation libraries appear only in the offline calibration harness (Phase 4 Task 8).

---

## Build order

Each phase is its own plan and produces working, testable software on its own. Later
phases depend only on the frozen contracts and modules of earlier ones, never on their
internals.

### Phase 1: Core foundation  →  `01-core-foundation.md` (detailed)
Contracts, primitives (stable sort, canonical JSON, hashing, identity), `Deps` +
in-memory fakes, the **gate** (verdict computation, evidence-floor differential,
waivers, dedup), the pure `run()` orchestrator, receipt minting, a CLI slice, and a
golden-oracle harness. **Exit criterion:** all four verdicts plus the idle non-verdict
are reachable by running `run()` over fakes, and the canonical `Result` is
snapshot-stable.

### Phase 2: Detection engine  →  `02-detection-engine.md`
The `Provider` interface and three deterministic layers: axe-core provider (WCAG
baseline, violations gate and axe `incomplete` maps to `unverified`), PatternFly rulepack
(static rules plus interaction probes that click and key through modals and menus), and
the keyboard walk (role-aware tab probing that reports `unverified` rather than
guessing). Plus: live-region capture primitives, the CDP accessibility-tree
`BrowserDriver`, and a `CheckRunner` that records the transcript into `ScreenScan.stops`
and provider gaps into `ScreenScan.gaps`. Depends on Phase 1 contracts. **Exit
criterion:** real `Draft[]` produced from a live page through the real `BrowserDriver`,
with stops and gaps populated.

### Phase 3: Coverage, guard, and trust  →  `03-coverage-guard-trust.md`
Route-graph + wide-blast coverage (changed files → affected screens; `nothingToCheck`
vs `not_covered`), the git-anchored guard (config file unconditionally self-guarded,
directory guarded paths expanded to files, session pin helpers), receipt binding via the
single `computePolicyHash`, the trusted-ref read path (CI reads config, floor, and
waivers from the base ref, never the PR working tree), and the evidence floor + waivers
accept loop. Depends on Phase 1. **Exit criterion:** a policy edit forces
`approval_required`; an accept commit converges the next run to `verified`; a receipt
minted by `run()` re-verifies with the same hash function.

### Phase 4: Announcement voicing lane  →  `04-voicing-lane.md`
The differentiating layer. `InteractionContract` + `SpeechObligation` authored
contracts; a Virtual Screen Reader provider; the two-tier split: structural tier
(deterministic, gates: missing name/role/state, no live region, unreachable focus,
over-long tab path) and voicing tier (`preview`, advisory: transcript token matching
inside a step's window). Measured promotion via a `promotedObligations` config list,
populated by an offline Orca-on-Fedora match-rate harness. Depends on Phases 1–2 (reuses
the keyboard-walk step runner). **Exit criterion:** voicing findings emit as `preview`
by default; a promoted obligation class emits as `deterministic` and gates.

### Phase 5: Surfaces  →  `05-surfaces.md`
Full CLI, the Claude Code stop hook (a real hook runner speaking the documented hook
protocol: stdin JSON in, `{"decision":"block"}` out, `stop_hook_active` one-continuation
cap, receipt fast path from the on-disk receipt store, session-pin check), the CI GitHub
Action + PR comment (policy from `--trusted-ref`, refusal without a base ref, current
announcements from the transcript), the dev-server overlay (Vite plugin + served client
+ result endpoint, advisory), and the Playwright helper. Every surface calls the same
core and is a projection of `Result`. Depends on Phases 1–3. **Exit criterion:** the
same code state yields the same verdict across all surfaces, and the stop hook actually
blocks in a live Claude Code session.

### Phase 6: Intake and accessible docs output  →  `06-intake-and-docs.md`
`RequirementBundle` normalize + schema (many shapes in, one bundle out; guarded), and
the docs-output artifacts (alt-text manifest, announcement snippets, keyboard paths)
bound to receipt evidence. Depends on Phases 1–3. **Exit criterion:** an authored
requirement maps to a check; a generated artifact carries an `evidenceRef`.

### Phase 7: Demo fixture and real-app measurement  →  `07-demo-and-measurement.md`
Scaffold the `usabl-app` PatternFly v6 fixture (the repo does not exist yet), plant the
hero bug (modal focus not returned on close; toast-not-announced and unnamed icon button
follow as fixture growth), prove the clean variant yields zero findings through the full
provider stack, drive the broken→fixed verdict flip end to end through the real engine,
and run a measurement-only pass against Fleet Insights (coverage / `not_covered`
reporting; rendered DOM via Playwright storageState; no CI, no gating). Depends on all
prior phases and uses the Phase 1 contracts verbatim. **Exit criterion:** the
broken→fixed loop is reproducible and the receipt re-verifies.

### Phase 8: Strong team demo  →  `08-strong-team-demo.md`
Expand the proven Phase 7 loop into one team demonstration across the real dev overlay,
Claude mid-session check, stop hook, PR and CI, plus measurement-only Fleet Insights
evidence. Replace the small overlay badge with an accessible inspector, add several
supported accessibility scenarios to a realistic operations workflow, and keep preview
controls separate from source-bound proof. **Exit criterion:** two operators can run and
recover the full sequence, every local and GitHub surface agrees for the same source,
and Fleet Insights findings and gaps have human-reviewed measurement evidence.

---

## Dependency graph

```
                 ┌─────────────────────────┐
                 │ Phase 1: Core foundation │  (contracts, gate, run, receipt)
                 └─────────────┬───────────┘
        ┌──────────────┬───────┴───────┬──────────────┐
        ▼              ▼               ▼              ▼
 ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────┐
 │ Phase 2:   │ │ Phase 3:   │ │ Phase 6:     │ │ Phase 5: │
 │ Detection  │ │ Coverage / │ │ Intake /     │ │ Surfaces │
 │ engine     │ │ guard /    │ │ docs output  │ │ (needs   │
 └─────┬──────┘ │ trust      │ └──────────────┘ │  1,2,3)  │
       │        └─────┬──────┘                   └────┬─────┘
       ▼              │                               │
 ┌────────────┐      │                               │
 │ Phase 4:   │◄─────┘ (reuses keyboard-walk runner)  │
 │ Voicing    │                                       │
 └─────┬──────┘                                       │
       └──────────────────────┬────────────────────── ┘
                               ▼
                   ┌──────────────────────────┐
                   │ Phase 7: Demo + measure   │
                   └──────────────────────────┘
```

## Conventions (all phases)

- **TDD:** write the failing test, watch it fail, write the minimal code, watch it pass,
  commit. Every test runs on fakes.
- **One test root:** all tests live under `test/` (the vitest include is `test/**/*.test.ts`).
  No plan writes to `tests/`.
- **Fakes come from one place:** unit tests build fakes with `makeFakeDeps` / `makeFakePage`
  from `src/deps/fakes.ts`. No hand-rolled `Deps` or `Page` literals in tests, no `as any`.
- **Injected clock only:** `deps.clock()` is the only time source anywhere in the engine and
  its projections. No `Date.now()` or `new Date()` in shipped code.
- **Comment hygiene:** code comments in the product repo use plain language. Decision-log
  section numbers and internal shorthand stay in these plan files, never in committed source.
- **DRY / YAGNI:** build only the accessibility verdict now; keep `run()` general so it
  could generalize later, but do not build the general engine.
- **Frequent commits, Conventional Commits** (`feat:`, `fix:`, `test:`, `refactor:`,
  `chore:`, `docs:`).
- **Canonical-first:** any new output is a pure function `Result -> bytes`. Never
  re-derive findings in a formatter.
- **Do not weaken honesty:** a surface that cannot be fully exercised is `not_covered`,
  never a silent pass. `preview`/`model-judgment` never mint `verified`.
