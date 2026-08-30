# D, demo readiness, tested

Tested means actually run, not asserted. All of D is done by the finishing
engineer solo; the human versions are v0.2.1 (see human-tasks.md).

## D1, hero loop end to end

Rehearse the full loop locally and confirm every surface agrees on one Result.

1. `npm run demo:repair` to a clean baseline, run the gate, confirm verified.
2. `npm run demo:break`, run the gate, confirm the regression and its findings.
3. Confirm the same Result renders consistently on all four surfaces: CLI
   summary, overlay, PR comment, stop hook.
4. `npm run demo:repair`, run the gate, confirm it returns to verified.

Capture: the Result JSON for broken and repaired, and a note that all four
surfaces matched.

### Verified run, 2026-08-30

Run against the fixture on a throwaway branch (broken base commit, then a
working-tree repair), engine runnerVersion `0.2.0`, dev server on
`127.0.0.1:5173`. The run is ground truth; the earlier prediction in this file
did not match and was corrected here.

Broken source, verdict `regression`, exit `1`, eight gating findings on the two
screens the diff reaches through the route graph:

- `pf-focus-into-dialog` (serious) on clusters.
- `pf-modal-focus-return` (serious) on clusters, the canonical hero bug.
- `axe/button-name` (critical) on deployments.
- `pf-icon-button-name` (serious) on deployments.
- `pf-kebab-expanded-state` (serious) on deployments.
- `pf-row-action-name-unique` (serious) on deployments.
- `pf-toolbar-labeled-when-repeated` (moderate) on deployments, two instances.

Repaired source, verdict `verified`, exit `0`, zero findings, receipt minted and
bound to sourceTree, policyHash, runnerVersion, and the scanner stack.

Two corrections to the original prediction:

- The regression exit code is `1`, not `2`.
- `pf-toast-live-region` does not appear, and correctly so. A toast enters the
  DOM only after a "Start deployment" click; a snapshot scan has no alert
  element, so the rule short-circuits at its empty-alerts guard and returns
  nothing. It proves containment only and refuses the temporal announcement
  half, so an absent alert is silence, not a guess. The announcement barrier
  still lives in the keyboard preview experience; the gate reports only the eight
  barriers it can observe deterministically. Findings carry no WCAG
  success-criterion field, so the `WCAG 2.4.3` label was descriptive only.

All four surfaces agreed on both states, including the same receipt sourceTree
hash on every surface that carries one. Artifacts are in the finishing
engineer's evidence directory and feed E-release.md.

## D2, clean-clone onboarding proxy

Prove the onboarding path from a fresh clone. This is the agent-run proxy for
the human onboarding proof (H1).

1. Fresh clone of usabl-app into a scratch dir.
2. Walk the onboarding: init, baseline, install (overlay, claude, ci,
   branch-rule as prepared drafts), doctor, first verdict.
3. Time it against the under-30-minutes north star. Record the actual time.
4. Record every refusal or manual step the tool printed, so H1 has a script.

Capture: the command transcript, the timing, and the resulting doctor output.
Label it agent-run, not human-confirmed.

### Verified run, 2026-08-30

Run as a background agent from fresh clones of both repositories, then
independently re-read by the finishing engineer against the raw command outputs.
It is agent-run, not human-confirmed.

The clones landed on the exact frozen commits: engine `51a9ce3` and fixture
`1c2404f`, both matching the expected heads. The fixture consumes the engine
through `"usabl": "file:../usabl"`, and after `npm ci` the `node_modules/usabl`
symlink resolved so `npx usabl` ran the frozen engine.

Timing against the under-30-minutes north star: prepare (two `npm ci` runs, the
engine build, and a warm-cache Playwright install) took 57 seconds; the
eight-command onboarding walk took 52 seconds; total 109 seconds, about 1.8
minutes, leaving roughly 28 minutes of headroom. One caveat: the Playwright
install finished in 4 seconds on a warm cache and did no real download, though it
printed an unsupported-OS notice. A cold-cache machine would add download time to
prepare.

The eight-command walk, with each exit code:

- `usabl init` exit 2: refused to overwrite the existing config and route
  manifest without `--force`, attributed all four routes, and flagged three
  drafts for review.
- `usabl baseline` exit 0: wrote zero floor entries as a draft to review and
  merge.
- `usabl install --overlay` exit 0: already wired, no change.
- `usabl install --claude` exit 0: updated the live `.claude/settings.json` to
  run the Stop hook, with a note to review the diff before committing.
- `usabl install --ci` exit 2: refused to overwrite the hand-tuned security
  workflow, which differs from the draft at line 40, and asked for a manual
  reconcile.
- `usabl install --branch-rule` exit 2: reported not applied, because branch
  protection cannot be set from the CLI, and printed the exact GitHub setting for
  a human to apply.
- `usabl doctor` exit 0: seven wired, one missing (branch protection), zero
  drifted, zero unknown, and restated that doctor mints no verdict.
- `usabl check` exit 2: approval required, because `baseline` had dirtied the
  guarded `.usabl-evidence.json`; accessibility itself was idle with no
  UI-touching diff.

Only two files ended modified: `.claude/settings.json` and
`.usabl-evidence.json`. Every non-zero exit is an honest refusal or verdict in
the safe direction, not a crash. The four exit-2 results are the honesty design
made visible: usabl will not clobber hand-tuned config, will not claim a GitHub
setting it cannot make, and the diff-aware gate asks for approval when a guarded
path changes rather than minting a green. Eleven refusals or manual steps were
captured for the H1 script, two of them hard human-required limits: reconcile the
CI workflow by hand, and set branch protection in the GitHub UI. Artifacts are in
the finishing engineer's evidence directory and feed E-release.md.

## D3, green suites

Verified 2026-08-30.

- Full usabl suite green: `npm run check` exit 0, 588 passed and 7 skipped
  across 92 files, then typecheck, build, DTS, and the package smoke all pass.
  The 7 skips are the live integration tests that are skipped by default
  (`test/deps/real.integration.test.ts` and peers), not failures.
- Full usabl-app suite green: 21 passed across 8 files, typecheck clean, lint
  clean, including the demo-wiring fix from B1.
- Output pristine, no warnings. One caveat learned here: a leftover git worktree
  under the gitignored `.work/` dir made vitest discover a second, stale copy of
  the suite and report a phantom failure. Gitignore hides a path from git, not
  from the test runner. The pristine counts above are from a clean tree with no
  agent scratch present.

## Evidence to capture (feeds E-release.md)

- Result JSON, broken and repaired.
- The four-surface agreement note.
- The onboarding transcript and timing.
- The oracle RED and GREEN links from PR #20.
- CI run links for the final green gate.
