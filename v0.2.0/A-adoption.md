# A, adoption engineering

The seven remaining adoption slices. Source: usabl-adoption-slices.md and
usabl-v0.3.0-plan.md. All confirmed not built as of 2026-08-29 (no `src/install`,
`src/drift`, `src/doctor`; no `paidDownCount`).

Design lines carried from the source plans, they still hold:

- Generators write drafts. Only a reviewed merge trusts them.
- The engine never consumes policy it generated in the same run.
- Every command is idempotent.
- On a wrong or missing inference, refuse and print the manual step. Never emit
  confident wrong coverage. A wrong guess is a gap or a refusal, not a verdict.
- Waivers stay human-authored.
- Repo settings are operator actions. The tool prepares and verifies; a human
  applies.

Each slice is built test-first in a builder-reviewer loop, then gets a full
security review at slice granularity. The install family (6a-6d) ships as one
cohesive PR (A3) because it is one feature with shared scaffolding, but each of
its four slices gets its own security review inside that PR.

## PR A1, slice 4b, floor-notice

Goal: when the evidence floor pays down debt, say so on every surface, so a
re-armed gate is visible rather than silent.

- Add `paidDownCount` to the canonical `Result` (the count of floor entries that
  matched and were carried, made visible when the floor shrinks).
- Surface it in `src/output/summary.ts`, `src/surfaces/pr-comment.ts`,
  `src/surfaces/overlay-client.ts`.
- Keep the gate the only verdict authority; this is projection only.

Acceptance:
- RED tests first for each surface showing the notice absent.
- A floor pay-down shows the count on CLI summary, PR comment, and overlay.
- No pay-down shows nothing (no zero-noise).

Shared-file note: touches three surface files. Low conflict with other slices;
does not touch cli.ts.

## PR A2, slice 5, routes drift

Goal: detect when `usabl.routes.json` no longer matches the app's real routes,
so scan targets cannot silently rot.

- `src/drift/routes.ts`: compare configured routes against discovered routes;
  report added, removed, changed. Report only, never mint a verdict.
- `usabl drift routes` subcommand in `src/cli.ts`.

Acceptance:
- RED tests for added, removed, and matching routes.
- A route present in the app but missing from routes.json is reported as drift.
- Exit behavior follows the disclosure model, not a new verdict.

Shared-file note: registers a subcommand in cli.ts (integration-order conflict
with A3, A4).

## PR A3, slices 6a-6d, install family

Goal: one-command wiring of each usabl surface into a target repo. Generators
write drafts a human then reviews and merges.

- 6a `src/install/overlay.ts` and `usabl install --overlay`: write the Vite
  overlay wiring.
- 6b `src/install/claude.ts` and `usabl install --claude`: write
  `.claude/settings.json` for the stop hook. Add a `usabl stop-hook` subcommand
  so the hook has a stable command entry point rather than a raw path into
  `dist/stop-hook-runner.js`.
- 6c `src/install/ci.ts` and `usabl install --ci`: write the two-job gate
  workflow (accessibility scan plus trusted-ref policy).
- 6d `src/install/branch-rule.ts` and `usabl install --branch-rule`: prepare and
  verify the branch-protection ruleset. Prepare and verify only; a human applies
  the repository setting.

Acceptance:
- RED tests for each generator: output shape, idempotence (running twice is a
  no-op or a clean update), and refusal-with-manual-step on ambiguous input.
- The generated workflow matches the fixture's working two-job pattern.
- `usabl install --branch-rule` never mutates a repo setting itself; it prints
  the exact setting and verifies the current state.
- `usabl stop-hook` runs the same path the fixture wires today.

Shared-file note: new `src/install/` dir plus multiple cli.ts registrations.
When run against a target repo, these commands write guarded files (config,
workflows, CODEOWNERS); that is the command's job and is covered by generator
tests against fixtures, not by editing this repo's guarded paths.

## PR A4, slice 7, doctor

Goal: one command that tells an operator what is wired, what is missing, and the
next honest step. The self-check that makes onboarding legible.

- `src/doctor/index.ts` and `usabl doctor`: report presence and health of config,
  routes, evidence floor, waivers, overlay wiring, stop hook, CI workflow, branch
  rule. Each line states wired, missing, or drifted, and the next step.

Acceptance:
- RED tests for a fully wired repo, a bare repo, and a partially wired repo.
- Never claims wired when a check cannot confirm it; unknown is unknown.
- Output is a projection; doctor mints no verdict.

Shared-file note: new `src/doctor/` dir plus one cli.ts registration.

## Integration order for cli.ts

A2, A3, A4 all edit the cli.ts subcommand allow-list. Merge order: A1 (no cli.ts),
then A2, then A3, then A4, each rebased on main so the allow-list is resolved
once per merge. If a slice is delayed, it rebases behind the ones already in.
