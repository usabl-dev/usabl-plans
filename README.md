# usabl implementation plans

Phase-by-phase implementation plans for **usabl**, the accessibility proof engine
(Red Hat Innovation Days 2026).

**This repository holds planning documents only.** The product source lives in
`usabl-dev/usabl`; the demo consumer lives in `usabl-dev/usabl-app`. These plans are
kept out of the product repos on purpose so specs and plans never ship inside the
tool or its pull requests.

Team operating model: [`roles.md`](roles.md) (founder, CTO, Opus as EM, Codex, Gemini as CISO).
Quality bar (CTO enforces): [`quality-bar.md`](quality-bar.md).
Build story (living): [`how-we-built-this.md`](how-we-built-this.md).
Phase 2 PR map: [`02-slices.md`](02-slices.md) (cut from `main`, do not stack).

## Build order

Each plan is self-contained and produces working, testable software on its own. Later
phases depend only on the frozen contracts of earlier ones, never on their internals.

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

## How to execute a plan

Each plan is a sequence of bite-sized, test-first tasks. Use the
`superpowers:subagent-driven-development` workflow (a fresh subagent per task, review
between tasks) or `superpowers:executing-plans` for inline batch execution.

## Architecture (non-negotiable)

- One pure `run(deps, config)` over an injected `Deps` object. No hidden I/O.
- Every check is a `Provider` that returns `Draft[]`. Providers never decide anything.
- The gate is the single verdict authority. Verdicts are `verified`, `regression`,
  `not_covered`, `approval_required`, plus an idle non-verdict.
- The canonical `Result` JSON is the single source of truth. Every surface is a pure
  projection of it.
- `verified` and the re-verifiable receipt are reserved for reproducible evidence.
  A surface that cannot be fully exercised is `not_covered`, never a silent pass.
