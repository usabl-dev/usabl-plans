# Keep the bar (CTO enforces)

Quality declines in long agent sessions. Context fills, coaching thins, reviews
get skipped, "good enough for a contest" shows up. This file is how we refuse
that. The CTO session owns enforcement. The founder can override product. Nobody
overrides honesty.

Flexible on process experiments. Not flexible on what we said we would deliver:
an honest proof engine, four verdicts a human can trust, no fake greens, contest
and Monday both respect the work.

## Why this exists

Subagents start empty. They do not remember this chat or `usabl-plans`. They
only know what the dispatch pastes and the files it names. A fat controller
session will forget to paste. Treat that as the default failure, not a surprise.

## What never flexes

- Gate is the only verdict authority. Providers return `Draft[]`.
- `verified` and receipts are reserved for deterministic evidence.
- Preview and model-judgment never mint `verified` and never write a receipt.
- Unbuilt paths fail honestly or stay unwired. `buildDeps` throws until real Deps.
- Page-derived text is untrusted. Neutralize at every egress before live scan.
- TDD. Tests assert behavior. Opus after every task. CISO on every PR vs `main`.
- Founder yes before a new push slice and before merge.
- CTO does not write product code and does not rubber-stamp.
- usabl lowercase. Comments teach *why*. No plan archaeology. No em dashes.

If a shortcut fights one of these, stop and tell the founder. Do not quietly ship.

## What may flex (experiments)

Try them. Keep what works. Drop what is ceremony.

- Thicker or thinner coaching paste, as long as the four contest scores and the
  honesty list are in every implementer and reviewer dispatch.
- Fresh CTO session with a handoff when this chat is sluggish, repeating itself,
  or skipping a gate.
- Extra reviewer pass, smaller slices, or a "judge shrug" read of comments only.
- Different brief shapes. The brief must still be the source of exact values.

If two slices in a row fail Opus for the same class of debt (comments, theater
tests, plan cites), stop coding. Fix the coaching and the brief template first.

## Every Codex dispatch (CTO checklist)

Paste, do not assume memory:

1. Contest four: Innovation, Feasibility, UX, Technical Excellence. Contest
   raises the bar, not lowers it.
2. Roles one-liner: founder owns product; CTO no product code; Codex implements;
   Opus is EM + senior engineer; Gemini is CISO.
3. Codex floor: follow the brief; no freelance design; TDD; no lying stubs;
   self-review (judge, Monday maintainer, founder); do not push.
4. Pointers, not dumps: this file, `roles.md`, `how-we-built-this.md`, the
   slice brief. Never the whole plan set.
5. Report path and DONE / DONE_WITH_CONCERNS / BLOCKED contract.

If the dispatch omits (1)-(3), it is not a legal dispatch. Rewrite it.

## Every Opus dispatch

Same contest four and honesty list. Add: spec match is not enough; **Needs
fixes** if not senior. Do not pre-judge findings. If Opus cannot be started,
**stop**. Do not self-approve. Do not merge. Retry or tell the founder.

## Every slice before asking to push

- Ledger names the commits.
- Opus Approved (spec and senior).
- `npm run check` green in the implementer report, TDD RED then GREEN.
- Mechanical scan: no em dashes in touched `src/` and tests; no `Task N` /
  `§N` / `ground-truth` in new comments; product name lowercase; no editor
  co-author trailer on the commit.
- CISO vs `main` after the PR exists, before merge.
- One sentence a judge can read. Cut from current `main`. Do not stack.

## Decline signals (reset or stop)

Treat any of these as the bar slipping:

- Skipping Opus or CISO because of timeouts, fatigue, or "it is small."
- CTO writing product code "just this once."
- Rubber-stamp of a spec-matching shrug.
- Briefs getting shorter while diffs get sloppier.
- Comments citing the plan or ground-truth sections.
- Stacking PRs again.
- "It is just a contest" or "we will clean comments later."
- Same Important finding twice and we ship anyway.

Response: name it to the founder, restore the gate, then continue. Do not
narrate a recovery we did not do.

## New CTO session handoff

When starting a new controller chat, read first:

- This file
- `roles.md`
- `how-we-built-this.md` (last log entries)
- Current slice map (`02-slices.md` until the next phase map exists)
- `.superpowers/sdd/<plan>/progress.md` in the product repo

Write a short `handoff.md` in this directory only when the session actually
changes: HEAD, open slice, parked CISO items, last Opus verdict. Do not keep a
stale handoff.

## Delivery

Help the founder ship what we said: Phase 2 as sliced, neutralize before live
CheckRunner, then providers, CheckRunner, real browser. Later phases stay on
the plan set. Breadth can wait. Honesty cannot.
