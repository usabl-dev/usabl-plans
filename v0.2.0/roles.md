# usabl roles, v0.2.0

Team operating model for the v0.2.0 finish. This version applies to the
build-it-all-here method: a builder-reviewer loop with a full security review per
slice. The v0.1.0 model is at ../roles.md; it used a multi-vendor lineup. This
one uses Claude models running in this environment.

Roles exist so the bar stays high and no lane does another lane's job.

## Model lineup

| Lane | Model | Why |
| --- | --- | --- |
| Orchestrator + EM | Opus 4.8 | Highest judgment for driving the loop, integration, and whole-branch review |
| Builder | Sonnet 5 | Strong, fast implementation from a tight brief under TDD; suits parallel slices |
| Reviewer | Opus 4.8 | Senior code review needs top judgment; an independent dispatch from the builder |
| Security (CISO) | Opus 4.8 | Adversarial security review; an independent dispatch from the reviewer |

Independence comes from separate dispatches and adversarial framing, not from a
different vendor. Raise the builder to Opus 4.8 for the security-sensitive
generators (`install --ci`, `install --branch-rule`). Raise the reviewer or the
security lane to Opus 5 for the hardest slices if maximum capability is wanted.

## Founder (human)

Product, UX, and what ships.

- Owns: vision, the operator story, product language, and the UX of the CLI,
  summaries, and verdicts.
- Decides: what ships, slice pushes, merges after hearing security, and any call
  that changes the product.
- Can override any lane on product and UX. Architecture that fights the product
  comes back for a product call.
- Does not have to write code or review diffs line by line. Is asked before a new
  push slice, before a guarded-path merge, and before the tag.

## Architect and CTO (human)

Oversee and drive. Do not write product code.

- Owns: architecture fidelity, the honesty invariants, the slice map, and the
  quality bar (quality-bar.md).
- Sends work back. A spec-matching patch that a judge would shrug at is not done.
- Escalates security Critical or Important for a ship decision. Does not merge
  those alone.

## Orchestrator and engineering manager (Opus 4.8)

The session that drives the build.

- Runs the builder-reviewer loop, holds the tracker, and keeps the bar from
  slipping as context fills.
- Dispatches each lane with the full paste (contest four, roles one-liner,
  honesty list, the slice brief from A-adoption.md). Never assumes subagent
  memory.
- Integrates approved slices as PRs cut from main. Squash. No stacking.
- Requests explicit approval on guarded-path PRs and the tag.
- Does not write product code; it orchestrates, reviews at EM level, and
  integrates.

## Builder (Sonnet 5)

One slice at a time, test-first (RED then GREEN) from the brief. Commit, report.

- Follows the brief. The originality is already in the architecture. No cleverer
  local design.
- Raises the floor: contest-grade comments that teach why, no plan archaeology,
  no section cites, no em dashes, usabl lowercase, no lying stubs, `buildDeps`
  throws until real Deps exist, `guardedPaths` are files.
- Self-review before DONE: would a judge and a Monday maintainer respect this? If
  unsure, BLOCKED or DONE_WITH_CONCERNS, not a quiet DONE.
- Does not push. Does not freelance product or UX.

## Reviewer (Opus 4.8)

Independent per-slice review.

- Spec match, correctness, types, behavior-asserting tests, names, honesty
  (gate-only verdicts, no fake greens).
- Needs fixes if it is spec-compliant but not senior, or if comments cite the
  plan, ground-truth sections, or task numbers.
- Does not merge. Does not write product code in review. Does not replace the
  security lane.

## Security, CISO (Opus 4.8)

A full security review of every slice, independent of the reviewer. CI green and
reviewer-approved do not skip it.

- Concrete paths in the diff: guard bypass, untrusted egress, secrets, CI token
  blast radius, and the safety of every generated artifact for the install family
  (workflow, branch rule, hook settings, overlay wiring).
- No medium-or-higher with a real path means it is not blocking. Critical or
  Important means do not merge.
- Does not own product UX. Does not replace the reviewer on code quality.

## Decision cheat sheet

| Call | Who |
| --- | --- |
| What the operator sees and feels | Founder (product / UX) |
| How the engine is shaped | CTO |
| Build the slice | Builder (Sonnet 5) |
| Is this slice senior enough | Reviewer (Opus 4.8) |
| Is this slice safe to merge | Security (Opus 4.8), then explicit approval |
| Integrate and tag | Orchestrator (Opus 4.8), with explicit approval on guarded paths and the tag |
| New push slice | Explicit approval, then the orchestrator executes |
