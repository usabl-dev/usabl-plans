# C, docs for handoff

The docs must be current at v0.2.0 so the team can pick usabl up without a guide.
Everything here is non-guarded.

## C1, Item 5 truth pass (usabl)

Reconcile the docs with the code. Known drift to fix, gathered from the code
deep-dive and the v0.2.0 plan:

- ground-truth.md: the EvidenceClass description drifts between three values and
  four values. Fix to the real four: deterministic, preview, model-judgment,
  human-confirmed.
- The receipt docstring says three bindings; `verifyReceipt` checks four
  (sourceTree, policyHash, runnerVersion, scannerVersions). Correct the doc.
- `baseRevision` is described as verified but is not verified anywhere. State
  what is actually checked, or remove the claim.
- The voicing note should say plainly: built and exported, not wired, lands in
  v0.3.0 with its calibration.
- Document the exit-4 stop-hook policy (fail-open discloses, does not block) as
  a deliberate decision, matching run.ts.
- Threat model: note the app-CI same-job risk and how the two-job split answers
  it.
- Run link checks across the docs.
- Write a CHANGELOG entry for v0.2.0 covering every new adoption command
  (drift, install family, stop-hook, doctor) and the floor-notice.

PR C1, usabl, non-guarded. Can start early; the CHANGELOG lines for adoption
commands finalize once those slices land.

## C2, team-orientation site to v0.2.0

The published page at usabl-dev.github.io/usabl/team-orientation.html is at
v0.1.0. First locate the GitHub Pages source (the repo and path that publishes
that site; check usabl for a docs/ or site source, or a separate pages repo).

Updates:
- Drop the "three weeks to the contest" timeline; replace with the current
  handoff framing.
- Add the adoption commands to the workflow section: init, baseline, floor prune,
  drift, install, doctor.
- Refresh the claim boundary to v0.2.0.
- Keep the result model and the four surfaces; verify each still matches.
- Point to human-tasks.md as the "what to do after you onboard" catalogue.

PR C2, docs site, non-guarded.

## C3, READMEs, CONTRIBUTING, linked guides

- usabl README and usabl-app README to v0.2.0, including the new command set.
- CONTRIBUTING current with the PR template and the review lanes.
- The linked guides referenced from orientation (proof loop, how usabl works,
  testing and reporting, contribution, demo rehearsal) refreshed to v0.2.0 and
  the new commands.

PR C3, non-guarded. Do after the command set is final so nothing documents a
command that changed shape at merge.
