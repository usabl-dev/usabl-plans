# Human tasks, v0.2.1 and later

The work that needs a person, to assign after the team onboards. This is not the
sprint tracker; it is the handoff. It becomes useful once v0.2.0 is tagged and
the tool is the team's.

Prerequisite for every task: complete team-orientation first (the site at
usabl-dev.github.io/usabl/team-orientation.html), so the owner shares the result
model, the four surfaces, and the claim boundary.

Roles are from roles.md: Founder (product and UX), CTO (engine shape), reviewer
(the senior-review lane), CISO (security review), any teammate.

Each task lists why it matters, a suggested owner, prerequisites, acceptance,
and a target version.

## H1, clean-clone onboarding proof (human)

Why: the agent ran a proxy (D2). The real proof is a teammate who has never
wired usabl doing it from a clean clone.
Owner: any teammate who did not build usabl.
Prereq: onboarding docs current (done in v0.2.0).
Acceptance: from a fresh clone to first honest verdict, only review, merge, and
one approved repo setting as human acts, under 30 minutes. Every confusion is
logged.
Target: v0.2.1.

## H2, accessibility review, NVDA and keyboard (human)

Why: usabl issues machine verdicts on accessibility; a human must confirm the
tool itself and the fixture behave for a real assistive-tech user.
Owner: a teammate with NVDA, plus a keyboard-only pass. Orca and VoiceOver
optional.
Acceptance: the hero bug and its repair are confirmed by ear and by keyboard,
not only by the gate. Findings that the automated layer misses are recorded.
Target: v0.2.1.

## H3, two-operator guarded-change rehearsal (human)

Why: the trust model depends on a second owner approving guarded changes. It
must be rehearsed by two people, not asserted.
Owner: an author plus a CODEOWNERS reviewer, two different people.
Acceptance: a guarded change is blocked until a second owner approves on the
current head, the author's own approval does not count, and the gate flips on
approval.
Target: v0.2.1.

## H4, delivery and recovery rehearsals (human)

Why: the team must know what to do when the gate is red, the engine token
expires, CI breaks, or the floor needs re-arming.
Owner: CTO plus a teammate.
Acceptance: each failure mode is triggered on purpose and recovered, with the
steps written down.
Target: v0.2.2.

## H5, dogfood usabl on a real second project (human)

Why: the fixture is built to fail in known ways. Real confidence comes from
wiring usabl into a project it did not grow up with.
Owner: any teammate with a candidate repo.
Acceptance: usabl gates a real project, and the friction is recorded as backlog.
Target: v0.2.2.

## H6, verify and own repo settings (operator)

Why: branch protection, required checks, and CODEOWNERS membership are operator
actions the tool can only prepare and verify.
Owner: Founder or CTO with repo admin.
Acceptance: the "Protect main" ruleset is confirmed active with no bypass
actors, both gate checks required, and CODEOWNERS membership is correct.
Target: v0.2.1.

## H7, npm publish decision and execution (human)

Why: publishing makes the engine public. It is a visibility decision, not an
engineering one.
Owner: Founder.
Prereq: confirm the `usabl` name is available on npm.
Acceptance: a decision is recorded. If yes, the package is published and CI
switches from the pinned private checkout to `npm install usabl@0.2.0`.
Target: v0.2.x.

## H8, contest demo rehearsal and pitch (human)

Why: the demo must land with judges, live, under time.
Owner: Founder plus one presenter.
Acceptance: the hero loop is rehearsed end to end within the time slot, with a
fallback if the network or lab misbehaves.
Target: contest.

## H9, triage the v0.3.0 verifier issue (human)

Why: 09-verifier is genuinely new work held for v0.3.0. It needs scoping before
anyone builds it.
Owner: CTO.
Acceptance: the six mechanisms and the voicing wiring are triaged, sequenced,
and a v0.3.0 slice plan exists.
Target: v0.3.0.
