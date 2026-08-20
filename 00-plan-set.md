# usabl Implementation Plan — Overview and Build Order

> **Location note:** This plan set lives in the planning directory, NOT in the `usabl`
> or `usabl-app` repos. Source of truth for decisions is
> `../usabl-decisions-2026-08-19.md`; the in-repo canonical spec is
> `usabl/docs/ground-truth.md`. Do not commit these plan files to any shared repo.

**Goal:** Build a working prototype of usabl — an accessibility proof engine that,
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
// decides a verdict — it returns Draft[] only. The real CheckRunner (Phase 2) opens a Page
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

**Tech stack:**
- TypeScript (ESM, strict), Node 22.
- Vitest for all tests. Every test runs on in-memory fakes; no network, no real browser
  in unit tests.
- tsup for the build (single `usabl` package, subpath exports, one `usabl` bin).
- Playwright drives the real browser behind the `BrowserDriver` interface (real Deps
  only; never in unit tests).
- axe-core (WCAG baseline provider), `@guidepup/virtual-screen-reader` (in-loop voicing
  provider).

---

## Build order

Each phase is its own plan and produces working, testable software on its own. Later
phases depend only on the frozen contracts and modules of earlier ones, never on their
internals.

### Phase 1 — Core foundation  →  `01-core-foundation.md` (detailed)
Contracts, primitives (stable sort, canonical JSON, hashing, identity), `Deps` +
in-memory fakes, the **gate** (verdict computation, evidence-floor differential,
waivers, dedup), the pure `run()` orchestrator, receipt minting, a CLI slice, and a
golden-oracle harness. **Exit criterion:** all four verdicts plus the idle non-verdict
are reachable by running `run()` over fakes, and the canonical `Result` is
snapshot-stable.

### Phase 2 — Detection engine  →  `02-detection-engine.md`
The `Provider` interface and three deterministic layers: axe-core provider (WCAG
baseline), PatternFly rulepack (design-system semantics only; defers to axe on anything
axe catches generically), and the keyboard walk (role-aware interaction probing that
reports `unverified` rather than guessing). Depends on Phase 1 contracts. **Exit
criterion:** real `Draft[]` produced from a live page through the real `BrowserDriver`.

### Phase 3 — Coverage, guard, and trust  →  `03-coverage-guard-trust.md`
Route-graph + wide-blast coverage (changed files → affected screens; `nothingToCheck`
vs `not_covered`), the git-anchored guard (config-guards-itself, session pinning,
`approval_required` on divergence), receipt binding (three hashes), and the evidence
floor + waivers accept loop. Depends on Phase 1. **Exit criterion:** a policy edit
forces `approval_required`; an accept commit converges the next run to `verified`.

### Phase 4 — Announcement voicing lane  →  `04-voicing-lane.md`
The differentiating layer. `InteractionContract` + `SpeechObligation` authored
contracts; a Virtual Screen Reader provider; the two-tier split — structural tier
(deterministic, gates: missing name/role/state, no live region, unreachable focus,
over-long tab path) and voicing tier (`preview`, advisory: transcript token matching
inside a step's window). Measured promotion via a `promotedObligations` config list,
populated by an offline Orca-on-Fedora match-rate harness. Depends on Phases 1–2 (reuses
the keyboard-walk step runner). **Exit criterion:** voicing findings emit as `preview`
by default; a promoted obligation class emits as `deterministic` and gates.

### Phase 5 — Surfaces  →  `05-surfaces.md`
Full CLI, the Claude Code stop hook (enforcer), the CI GitHub Action + PR comment
(tamper-proof via `--trusted-ref`), the dev-server overlay (Vite plugin, advisory), and
the Playwright helper. Every surface calls the same core and is a projection of
`Result`. Depends on Phases 1–3. **Exit criterion:** the same code state yields the same
verdict across all surfaces.

### Phase 6 — Intake and accessible docs output  →  `06-intake-and-docs.md`
`RequirementBundle` normalize + schema (many shapes in, one bundle out; guarded), and
the docs-output artifacts (alt-text manifest, announcement snippets, keyboard paths)
bound to receipt evidence. Depends on Phases 1–3. **Exit criterion:** an authored
requirement maps to a check; a generated artifact carries an `evidenceRef`.

### Phase 7 — Demo fixture and real-app measurement  →  `07-demo-and-measurement.md`
The `usabl-app` PatternFly fixture with the hero bug (modal focus not returned / toast
not announced / icon button unnamed — all structural), golden-oracle scenarios for all
four verdicts, and a measurement-only pass against Fleet Insights (coverage /
`not_covered` reporting; scan the rendered DOM via Playwright storageState; no CI).
Depends on all prior phases. **Exit criterion:** the broken→fixed loop is reproducible
and the receipt re-verifies.

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
- **DRY / YAGNI:** build only the accessibility verdict now; keep `run()` general so it
  could generalize later, but do not build the general engine.
- **Frequent commits, Conventional Commits** (`feat:`, `fix:`, `test:`, `refactor:`,
  `chore:`, `docs:`).
- **Canonical-first:** any new output is a pure function `Result -> bytes`. Never
  re-derive findings in a formatter.
- **Do not weaken honesty:** a surface that cannot be fully exercised is `not_covered`,
  never a silent pass. `preview`/`model-judgment` never mint `verified`.
