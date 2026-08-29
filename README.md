# usabl implementation plans

Planning documents for **usabl**, the accessibility proof engine (Red Hat
Innovation Days 2026).

**This repository holds planning documents only.** The product source lives in
`usabl-dev/usabl`; the demo consumer lives in `usabl-dev/usabl-app`. These plans
are kept out of the product repos on purpose so specs and plans never ship inside
the tool or its pull requests.

## Current work: the v0.2.0 finish

v0.1.0 shipped the engine and the demo consumer. v0.2.0 takes usabl to
team-pickup readiness: the remaining adoption commands, a docs handoff, a
rehearsed demo, and a v0.2.0 tag. Human testing (onboarding, screen-reader
review, two-operator and recovery rehearsals) follows the tag as v0.2.1 and
v0.2.2.

The plan set lives in [`v0.2.0/`](v0.2.0/). Start with the tracker; it is the
single source of truth.

| Doc | Purpose |
|-----|---------|
| [`v0.2.0/00-tracker.md`](v0.2.0/00-tracker.md) | Definition of done, decisions, PR checklist, and sequencing |
| [`v0.2.0/A-adoption.md`](v0.2.0/A-adoption.md) | The remaining adoption slices: floor notice, routes drift, install family, doctor |
| [`v0.2.0/B-fixes.md`](v0.2.0/B-fixes.md) | Fixes, hygiene, and open-PR dispositions |
| [`v0.2.0/C-docs.md`](v0.2.0/C-docs.md) | Docs truth pass and the team-orientation refresh |
| [`v0.2.0/D-demo.md`](v0.2.0/D-demo.md) | Hero-loop and onboarding rehearsals |
| [`v0.2.0/E-release.md`](v0.2.0/E-release.md) | Evidence bundle, final gate, and tag |
| [`v0.2.0/human-tasks.md`](v0.2.0/human-tasks.md) | Human tasks for after onboarding (v0.2.1 and later) |
| [`v0.2.0/roles.md`](v0.2.0/roles.md) | Operating model and model lineup for the v0.2.0 build |
| [`v0.2.0/quality-bar.md`](v0.2.0/quality-bar.md) | The bar the finish holds |
| [`v0.2.0/how-we-built-this.md`](v0.2.0/how-we-built-this.md) | Living build log for the finish |

v0.2.0 is built in one environment with a builder-reviewer loop per slice and a
full security review per slice. See the tracker and `v0.2.0/roles.md` for the
method and the model lineup.

## v0.1.0: the foundation build

The phase-by-phase plans that built the engine and the demo consumer. Each plan
is self-contained and produces working, testable software on its own; later
phases depend only on the frozen contracts of earlier ones, never on their
internals.

Operating model: [`roles.md`](roles.md). Quality bar: [`quality-bar.md`](quality-bar.md).
Build story: [`how-we-built-this.md`](how-we-built-this.md). Phase 2 PR map:
[`02-slices.md`](02-slices.md) (cut from `main`, do not stack). The v0.2.0
versions of the operating model, quality bar, and build log are in
[`v0.2.0/`](v0.2.0/).

| Plan | Phase |
|------|-------|
| [`00-plan-set.md`](00-plan-set.md) | Overview, build order, and cross-phase frozen seams |
| [`01-core-foundation.md`](01-core-foundation.md) | Phase 1: contracts, gate, `run()`, receipt, CLI slice, golden oracle |
| [`02-detection-engine.md`](02-detection-engine.md) | Phase 2: axe-core, PatternFly rulepack, keyboard walk |
| [`03-coverage-guard-trust.md`](03-coverage-guard-trust.md) | Phase 3: coverage planner, git-anchored guard, three-hash receipt |
| [`04-voicing-lane.md`](04-voicing-lane.md) | Phase 4: announcement voicing lane (structural gates, voicing preview) |
| [`05-surfaces.md`](05-surfaces.md) | Phase 5: CLI, stop hook, CI/PR comment, dev overlay, Playwright helper |
| [`06-intake-and-docs.md`](06-intake-and-docs.md) | Phase 6: requirement intake and accessible docs output |
| [`07-demo-and-measurement.md`](07-demo-and-measurement.md) | Phase 7: demo fixture flip and real-app measurement |
| [`08-strong-team-demo.md`](08-strong-team-demo.md) | Phase 8: multi-scenario fixture, accessibility inspector, Claude, CI, and Fleet Insights demo |

`09-verifier-evidence-and-outcomes.md` is future work held for v0.3.0.

## How to execute a plan

The v0.1.0 phase plans are sequences of bite-sized, test-first tasks. Use the
`superpowers:subagent-driven-development` workflow (a fresh subagent per task,
review between tasks) or `superpowers:executing-plans` for inline batch
execution.

The v0.2.0 finish uses a builder-reviewer loop with a full security review per
slice; see [`v0.2.0/00-tracker.md`](v0.2.0/00-tracker.md) and
[`v0.2.0/quality-bar.md`](v0.2.0/quality-bar.md).

## Architecture (non-negotiable)

- One pure `run(deps, config)` over an injected `Deps` object. No hidden I/O.
- Every check is a `Provider` that returns `Draft[]`. Providers never decide anything.
- The gate is the single verdict authority. Verdicts are `verified`, `regression`,
  `not_covered`, `approval_required`, plus an idle non-verdict.
- The canonical `Result` JSON is the single source of truth. Every surface is a pure
  projection of it.
- `verified` and the re-verifiable receipt are reserved for reproducible evidence.
  A surface that cannot be fully exercised is `not_covered`, never a silent pass.
