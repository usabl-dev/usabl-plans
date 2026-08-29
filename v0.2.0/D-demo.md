# D, demo readiness, tested

Tested means actually run, not asserted. All of D is done by the finishing
engineer solo; the human versions are v0.2.1 (see human-tasks.md).

## D1, hero loop end to end

Rehearse the full loop locally and confirm every surface agrees on one Result.

1. `npm run demo:repair` to a clean baseline, run the gate, confirm verified.
2. `npm run demo:break`, run the gate, confirm the expected findings appear:
   - `pf-modal-focus-return` (WCAG 2.4.3) on /clusters, the canonical hero bug.
   - `pf-toast-live-region` on /deployments, the announcement finding, emitted by
     the PatternFly rulepack (this is why voicing is not needed for the demo).
   - the other broken-mode /deployments findings the rulepack surfaces.
3. Confirm the same Result renders consistently on all four surfaces: CLI
   summary, overlay, PR comment, stop hook.
4. `npm run demo:repair`, run the gate, confirm it returns to verified.

Capture: the Result JSON for broken and repaired, and a note that all four
surfaces matched.

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

## D3, green suites

- Full usabl test suite green (458-plus tests, plus the new slice tests).
- Full usabl-app test suite green, including the demo-wiring fix from B1.
- Output pristine, no warnings.

## Evidence to capture (feeds E-release.md)

- Result JSON, broken and repaired.
- The four-surface agreement note.
- The onboarding transcript and timing.
- The oracle RED and GREEN links from PR #20.
- CI run links for the final green gate.
