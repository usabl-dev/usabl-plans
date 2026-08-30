# how we built this, v0.2.0

Living build log for the v0.2.0 finish. Append an entry as work lands. The
v0.1.0 log is at ../how-we-built-this.md; this file records v0.2.0, which uses a
different build method.

## build method (v0.2.0)

v0.1.0 was built phase by phase against the plan set (00 through 08). v0.2.0 is
the finish to team-pickup readiness, and it is built differently:

- Built in one environment, not handed between separate processes.
- A builder-reviewer loop per slice: one agent builds test-first, a second
  reviews, and the loop repeats until the reviewer is satisfied.
- A full security review of each slice, at slice granularity, not per PR. The
  install family ships as one PR but each of its four slices is reviewed on its
  own.
- Slices integrate as PRs cut from main, one at a time, squash-merged, no
  stacking.
- Guarded-path PRs and the tag require explicit approval; other PRs are
  pre-authorized and reviewed as they open.
- Test-first throughout. Honesty invariants never flex.

See 00-tracker.md for the definition of done and the checklist, and ../roles.md
for the builder, reviewer, and security lanes.

## what shipped after v0.1.0 (through 2026-08-29)

Landed on top of v0.1.0 before this finish began:

- Honesty and defect fixes: receipt-to-engine binding, announcement honesty, the
  docs-output surface made reachable, Fleet Insights export, the gate's own
  workflow and CODEOWNERS guarded, the guard-before-untrusted-parse ordering,
  waiver ISO-date validation, and the guarded-directory read fix.
- Adoption commands: init, baseline, floor prune (4a), and the split-CI approval
  path.
- Engine and fixture both at version 0.2.0; tags v0.1.0 and v0.2.0-rc.1.

## what is not built yet (honest, as of 2026-08-29)

- Adoption slices: floor-notice (4b), routes drift (5), the install family
  (6a-6d), and doctor (7).
- usabl-app main is red until the demo-wiring test matches the current engine
  pin.
- Docs are at v0.1.0: the orientation site, the linked guides, the READMEs, and
  the ground-truth truth pass.
- No v0.2.0 evidence bundle and no v0.2.0 tag yet.
- Voicing is built and exported but not wired; it lands in v0.3.0.

## log

### 2026-08-29

- Reconciled the plan set against the verified repo state and wrote the v0.2.0
  plan: the tracker, the workstream chunks, and the human-tasks handoff.
- Confirmed code-review items 1a and 1b are already merged, confirmed all seven
  adoption slices are unbuilt, and confirmed the announcement finding comes from
  the PatternFly rulepack rather than the voicing lane.
- Recorded the v0.2.0 build method above.

### 2026-08-30

- A1 (floor-notice, slice 4b) merged as usabl#89. The builder-reviewer loop
  rejected an early build that reported floor pay-down it had not confirmed, then
  landed a gap-guarded version whose honesty test was proven load-bearing by a
  mutation check (removing the guard flips the count and fails the test).
- usabl-app dispositions: confirmed B1 (demo-wiring pin) already merged as #22,
  merged #21 (gitignore for agent work dirs and secrets), and synced main. The
  #20 and #17 rehearsal PRs stay open until the hero rehearsal and evidence
  bundle exist, so their closes can cite real evidence rather than a claim.
- A2 (routes drift, slice 5) went through the full loop. The builder landed the
  URL-only comparison and, after a first review, the exit-code disclosure
  contract as a tested outcome function. The independent reviewer then blocked on
  a real honesty gap: a readable router the fallback regex cannot parse (for
  example React Router's data-router API, which uses a path object key) yields
  zero discovered routes, and the command reported every configured route as
  removed at exit 1 rather than refusing. The security lane separately found that
  file-derived URLs were printed to stdout without the codebase's egress
  neutralizer, which a crafted router path could use to spoof a clean report in
  CI logs. Both went back to the builder as one consolidated rework: refuse with
  a manual step when discovery is empty, make an unreadable router refuse rather
  than crash, and neutralize printed URLs.

(append entries as work lands)
