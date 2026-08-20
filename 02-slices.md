# Phase 2 slices

How plan [`02-detection-engine.md`](02-detection-engine.md) becomes PRs. Tasks stay
in that file. This file is the slice map: one PR, one sentence a judge can read.

Process (learned from Phase 1): **do not stack**. Cut each slice from current
`main`, squash-merge, then cut the next. Stacking plus squash rewrites feature
branches and other clones must reset those branches. `main` stays a fast-forward
story.

Working branch for Codex task commits: `feat/detection-engine`. Cherry-pick onto
a slice branch from `main` when the story is complete. Later slices wait until
their commits exist. Ask the founder before each push.

CISO (`gemini-3.1-pro` security-review vs `main`) on every PR before merge.

---

## Hard gate

Slice 0 lands **before** any `CheckRunner` copies page text into Drafts. Live
scanning without `neutralize()` on `formatSummary` ships untrusted page text to
the terminal. That is parked from Phase 1, not waived. Do not defer it to
surfaces (plan 05).

`buildDeps()` still throws until slice 7 can build real Deps without lying.
Tests keep injecting fakes. A stub that returned empty scans would mint a fake
`verified`.

---

## Map

| Slice | Story | Plan tasks | What becomes true |
| --- | --- | --- | --- |
| 0 | Finding text cannot hijack the terminal | (not in 02 task list; CISO hard gate) | `src/primitives/neutralize.ts` strips C0/C1, ANSI, and OSC. `formatSummary` runs `whatUserExperiences` and `fix` through it. Threat-model checkbox for this egress. Distinct from `neutralizePath` in identity (CSS paths, not page text). |
| 1 | A check is a Provider that returns Draft[] | Task 1 | `Provider` + capability filtering. Denied `live` records a coverage gap, never a silent pass. |
| 2 | axe-core is a WCAG baseline provider | Task 2 | Violations are deterministic drafts. axe `incomplete` maps to `unverified`, not a guessed fail. Unit tests on fakes only. |
| 3 | PatternFly static rules report honestly | Task 3 | Static rulepack rules. Selectors that match nothing stay dead; no pretend coverage. |
| 4 | Interaction-only PF rules probe; they do not guess | Task 4 | Modal, dialog, and menu rules `click()` and key through, then `activeElementIs` / `activeElementWithin`. Sequential after static rules. |
| 5 | Keyboard walk records what would be announced | Task 5 | Walk provider + `StepRunner` seam (Phase 4 reuses it). Unnamed interactive and unverified tab stops; no guessing. |
| 6 | CheckRunner assembles a ScreenScan with gaps | Task 6 | One `CheckRunner` touches `BrowserDriver`. `scan()` never throws. Stops, provider drafts, and gaps on `ScreenScan`. Still fakes. |
| 7 | A real browser reads the accessibility tree | Task 7 | Playwright `BrowserDriver` over CDP. Live-region capture. axe bridge. Transcript is the AX tree, not `aria-label \|\| innerText`. This is the first live page text. Slice 0 must already be on `main`. |

Do not open one PR per primitive. Do not dump Tasks 1-7 into one PR.

---

## Honesty that does not flex

- Providers return `Draft[]`. They never mint a verdict.
- The gate remains the only verdict authority.
- `verified` and receipts stay reserved for deterministic evidence.
- A screen that cannot be scanned is a gap -> `not_covered`, never a silent pass.
- usabl stays lowercase. Comments teach *why*. No plan-task numbers, no
  ground-truth section cites, no em dashes.

---

## Exit criterion (phase)

Real `Draft[]` from a live page through the real `BrowserDriver`, with `stops`
and `gaps` populated. The kernel from Phase 1 is unchanged: `run(deps, config)`
over injected Deps.
