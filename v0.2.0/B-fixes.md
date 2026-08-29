# B, fixes and hygiene

## B1, unblock usabl-app main (do this first)

`usabl-app/src/demo/demo-wiring.test.ts` asserts the gate workflow pins engine
commit `caef8a4` (an old commit, #80). The workflow now pins `e84b42b` (#88).
The test fails on main until fixed.

Steps:
1. Verify the failure by running the usabl-app test suite. Do not assume; read
   the actual failing output first.
2. Update the expected SHA to `e84b42b` (or to whatever the workflow pins at the
   time), so the test matches the workflow.
3. Note the brittleness: this test hard-codes the engine pin, so every re-pin
   breaks it. Record a follow-up (a less brittle assertion, for example reading
   the pinned SHA from the workflow) as a candidate for v0.2.1. Do not expand
   scope now.

PR B1, usabl-app, non-guarded. Small. Merge before anything else so the rest of
the sprint builds on a green main.

## B2, open PR dispositions

- #21 gitignore hygiene (mergeable): merge. Non-guarded. Confirms `.work/` and
  secrets are ignored.
- #20 oracle proof (draft, do-not-merge): close without merging. First capture
  its RED (8 findings on demo:break) and GREEN (verified on repair) evidence
  links into the bundle (see E-release.md).
- #17 hero-loop rehearsal (do-not-merge): close without merging. Capture its run
  as rehearsal evidence if useful, otherwise close with a note.

## B3, re-pin CI to the frozen engine commit

After all engine (usabl) PRs merge and the final v0.2.0 engine commit exists,
update the two `ref:` pins in `usabl-app/.github/workflows/usabl-gate.yml` to that
commit, and update B1's test expectation to match in the same PR.

PR B3, usabl-app, GUARDED (touches `.github/workflows`). Requires explicit
approval before it proceeds, and satisfies CODEOWNERS approval. This is one of
only two live-gated steps.

## B4, branch cleanup (optional, low priority)

usabl has 26 local branches, most merged. Pruning them is hygiene, not a
blocker. Do only if time allows, and never delete a branch that is not confirmed
merged.
