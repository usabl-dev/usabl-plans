# E, release

## E1, evidence bundle (Item 10)

Collect, in one place, the proof that v0.2.0 is real:

- Result JSON for the broken and repaired demo states.
- The note that all four surfaces rendered the same Result.
- The onboarding proxy transcript and timing (D2).
- The oracle RED and GREEN evidence from PR #20.
- CI run links for the final green gate on both repos.

Private evidence stays out of git. If any artifact needs to live in a repo, it
is scrubbed first and only with explicit approval. Fleet Insights is measurement
only, no verdict, no receipt.

### Status, 2026-08-30

The evidence bundle is assembled in one place, out of git, in the finishing
engineer's scratch area, with an index. It collects the broken and repaired
Result JSON, the four-surface agreement note, the D2 onboarding transcript and
timing, the oracle RED and GREEN links (verified: RED concluded failure, GREEN
concluded success), and the current green gate on both repos. One item is
honestly pending: the final usabl-app gate link comes after B3, the guarded CI
re-pin, which is held for approval.

## E2, final gate and tag (Item 11)

1. Confirm both repos are green: all suites pass, the gate is green, the CI pin
   points at the final engine commit (B3).
2. Freeze: if the engine advanced past v0.2.0-rc.1, cut v0.2.0-rc.2 or go
   straight to the tag once green. Record which.
3. Tag v0.2.0 on usabl, then on usabl-app.

The tag requires explicit approval. Before tagging, report the exact commits to
tag, both suites green, the gate green, the evidence bundle complete, and the
recovery method (delete the tag and re-cut if wrong). Tag only on explicit
approval.

## E3, 09-verifier to v0.3.0

- Commit `09-verifier-evidence-and-outcomes.md` to usabl-plans so it is preserved
  (it is currently untracked).
- Open a GitHub issue in usabl-dev/usabl titled for the v0.3.0 verifier work,
  summarizing the six mechanisms (mutation and sensitivity evidence, Orca
  calibration, barrier-days ledger, evidence-bound docs, agent repair study) and
  wiring the voicing lane. Label it v0.3.0 and future. Link the plan doc.

### Status, 2026-08-30

Held. Committing 09-verifier into the plans repo was declined by the workspace
safety guard as reversing the standing preference to keep specs and plans out of
shared repositories, together with the earlier decision to leave this doc
untracked and file it as an issue instead. The doc remains untracked and
preserved locally. The v0.3.0 issue is drafted and waits on explicit approval,
alongside B3 and the tag.

## E4, package distribution note (Item 6)

Recommendation for v0.2.0: keep the private pinned-checkout. The CI clones
usabl-dev/usabl at a pinned SHA with a token and snapshots it read-only. This
works for a private team pickup and requires no public exposure.

Defer npm publish. Publishing makes the engine public, an irreversible
visibility change that should not ride along with a team-pickup tag. It becomes a
human decision and task (H7): verify the `usabl` name is available, decide on
visibility, then publish and switch CI to `npm install usabl@0.2.0`.
