# Keep the bar, v0.2.0

Same principle as the v0.1.0 quality bar (../quality-bar.md): quality declines in
long agent sessions. Context fills, coaching thins, reviews get skipped, "good
enough for a contest" shows up. This file is how we refuse that, for the v0.2.0
finish and its build method (builder-reviewer loop, a full security review per
slice, all built here). Roles and model names are in roles.md.

Flexible on process experiments. Not flexible on what we said we would deliver:
an honest proof engine, four verdicts a person can trust, no fake greens, contest
and Monday both respect the work.

## Why this exists

Subagents start empty. They do not remember this session or the plan set. They
only know what the dispatch pastes and the files it names. Treat a forgotten
paste as the default failure, not a surprise.

## What never flexes

- The gate is the only verdict authority. Providers return `Draft[]`.
- `verified` and receipts are reserved for deterministic evidence. Preview and
  model-judgment never mint `verified` and never write a receipt.
- Unbuilt paths fail honestly or stay unwired. `buildDeps` throws until real Deps.
- Page-derived text is untrusted. Neutralize at every egress before a live scan.
- Test-first. Tests assert behavior. A reviewer pass after every slice. A full
  security review of every slice.
- Guarded-path changes and the tag require explicit approval. Other PRs are
  pre-authorized and reviewed as they open.
- No product code from the orchestrator or reviewer lane during review.
- usabl lowercase. Comments teach why. No plan archaeology. No em dashes. No
  personal names and no side-conversation references in committed docs or code.

If a shortcut fights one of these, stop and report it. Do not quietly ship.

## The v0.2.0 build loop

Per slice:

1. The builder builds test-first (RED then GREEN) from the slice brief in
   A-adoption.md.
2. The reviewer reviews: spec match, correctness, types, behavior-asserting
   tests, names, honesty. Not senior means needs fixes, not a merge.
3. Loop until the reviewer is satisfied.
4. A full security review of the slice: concrete paths in the diff (guard bypass,
   untrusted egress, secrets, CI token blast radius, generated-file safety for
   the install family). Critical or Important means do not merge.
5. Integrate as a PR cut from main. Squash. No stacking.

The install family (A3) runs steps 1 through 4 for each of its four slices before
it integrates as one PR.

## Every builder dispatch (paste, do not assume memory)

1. Contest four: Innovation, Feasibility, UX, Technical Excellence. The contest
   raises the bar, it does not lower it.
2. Roles one-liner from roles.md.
3. Builder floor: follow the brief; no freelance design; TDD; no lying stubs;
   self-review against a judge and a Monday maintainer; do not push.
4. Pointers, not dumps: this file, roles.md, the slice section of A-adoption.md.
   Never the whole plan set.
5. Report path and DONE / DONE_WITH_CONCERNS / BLOCKED contract.

If the dispatch omits (1) through (3), it is not a legal dispatch. Rewrite it.

## Every reviewer dispatch

Same contest four and honesty list. Spec match is not enough; needs fixes if it
is not senior. Do not pre-judge findings. Do not self-approve. Do not merge.

## Every security review (per slice)

Independent of the reviewer. CI green and reviewer-approved do not skip it.
Concrete paths only. No medium-or-higher with a real path means it is not
blocking. Critical or Important means do not merge. For the install family,
review the safety of every generated artifact: the workflow, the branch rule, the
hook settings, and the overlay wiring.

## Every slice before asking to integrate

- The ledger names the commits.
- The reviewer approved (spec and senior).
- `npm run check` green in the builder report, TDD RED then GREEN.
- Mechanical scan: no em dashes in touched `src/` and tests; no `Task N`,
  section cites, or ground-truth cites in new comments; product name lowercase;
  no editor co-author trailer on the commit; no personal names.
- The security review is done for the slice.
- One sentence a reviewer can read. Cut from current main. Do not stack.

## Decline signals (reset or stop)

- Skipping the reviewer or the security review because of time or "it is small."
- The orchestrator writing product code "just this once."
- Rubber-stamping a spec-matching shrug.
- Briefs getting shorter while diffs get sloppier.
- Comments citing the plan or ground-truth sections.
- Stacking PRs.
- The same Important finding twice, shipped anyway.

Response: name it, restore the gate, then continue. Do not narrate a recovery
that did not happen.
