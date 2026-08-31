# usabl v0.2.0 finish, master tracker

Single source of truth for finishing usabl to v0.2.0. Read this first. Update
status here as work lands. The detail lives in the per-workstream chunks; this
file holds the definition of done, the decisions, the checklist, and the order.

Last verified: 2026-08-29.

## Definition of done

v0.2.0 means "ready for the team to pick up." Concretely:

- All engineering complete: the code-review honesty fixes (already merged) plus
  all seven adoption slices (4b, 5, 6a through 6d, 7), and usabl-app main green.
- The hero demo runs and is tested end to end by the finishing engineer, across
  all four surfaces: CLI, overlay, PR comment, stop hook.
- A clean-clone onboarding path works, proven by an agent-run proxy. The human
  onboarding proof follows as v0.2.1.
- All handoff docs current at v0.2.0: the team-orientation site, linked guides,
  READMEs, the ground-truth truth pass, and a CHANGELOG.
- An evidence bundle is captured, the final gate is green, both repos tagged
  v0.2.0.
- The v0.3.0 verifier work (09) is filed as a GitHub issue and held.
- The human follow-on work is catalogued in human-tasks.md for assignment.

Human testing does not gate the v0.2.0 tag. Real-teammate onboarding, NVDA
review, two-operator and recovery rehearsals ship as v0.2.1, v0.2.2.

## How we execute

We build it all here, in this environment. The finishing engineer drives and
orchestrates a builder-reviewer loop: one agent builds a slice test-first (RED
then GREEN), a second agent reviews it, and the loop repeats until the reviewer
is satisfied. Every slice then gets a full security review, at slice granularity,
not per PR. Seven slices means seven security reviews; the install-family PR (A3)
therefore carries four, one for each of 6a, 6b, 6c, 6d. These are the builder,
reviewer, and CISO lanes from roles.md, run as subagents.

Slices integrate as PRs cut from main, one at a time. No stacking. Squash merges.
Honesty invariants never flex. usabl stays lowercase. No em dashes.

## Verified starting state (2026-08-29)

- usabl (engine): main `e84b42b`, clean, version `0.2.0`. Tags `v0.1.0`,
  `v0.2.0-rc.1`. No open PRs. Code-review items 1a (guard-before-untrusted-parse)
  and 1b (waiver ISO-date validation) are merged. Shipped CLI: check, comment,
  bypass, init, baseline, floor prune, enforce accessibility|policy, docs.
- usabl-app (fixture): main `91a041e`, version `0.2.0`. Tag `v0.2.0-rc.1`. CI
  pins engine `e84b42b`. Open PRs: #21 gitignore (mergeable), #20 oracle proof
  (draft, do-not-merge), #17 hero-loop rehearsal (do-not-merge). Known red:
  `src/demo/demo-wiring.test.ts` expects engine pin `caef8a4` but the workflow
  pins `e84b42b`, so main is failing until B1 lands.
- usabl-plans: main `a05ac1c`. `09-verifier-evidence-and-outcomes.md` is
  untracked.

## Decisions (2026-08-29)

1. v0.2.0 = team-pickup ready. Human testing follows as v0.2.1 and later.
2. Build all seven adoption slices in v0.2.0.
3. 09-verifier is v0.3.0. File a GitHub issue, hold the work.
4. Voicing stays frozen for v0.2.0 (built, exported, unwired). The demo's
   announcement finding comes from the PatternFly rulepack, not voicing. Wiring
   and calibration land in v0.3.0.
5. Package distribution: keep the private pinned-checkout for v0.2.0. Defer npm
   publish, which is a public-visibility decision, to a human task.
6. Approval cadence: pre-authorize non-guarded PRs; live-gate any PR touching
   guarded paths (config, CODEOWNERS, workflows, evidence ledgers) and the tag.
7. Plan docs: chunked, in usabl-plans/v0.2.0/, git-backed.

## Scope

In: adoption slices 4b, 5, 6a through 6d, 7; the demo-wiring fix; docs handoff;
demo rehearsal; evidence bundle; tag.

Out, v0.2.1 and later: human onboarding proof, NVDA and keyboard review,
two-operator rehearsal, recovery rehearsals, dogfooding a real project, npm
publish.

Out, v0.3.0: everything in 09-verifier (mutation and sensitivity evidence, Orca
calibration, barrier-days ledger, evidence-bound docs, agent repair study), and
wiring the voicing lane.

## PR checklist

Guarded PRs and the tag require explicit approval before they proceed.
Everything else is pre-authorized and reviewed as it opens.

| PR | Repo | Change | Guarded | Status |
|----|------|--------|---------|--------|
| B1 | usabl-app | fix demo-wiring test expected SHA, unblock main | no | merged (#22) |
| A1 | usabl | slice 4b floor-notice (paidDownCount across surfaces) | no | merged (#89) |
| A2 | usabl | slice 5 routes drift (`usabl drift routes`) | no | merged (#90) |
| A3 | usabl | slices 6a-6d install family + `usabl stop-hook` | no | merged (#91) |
| A4 | usabl | slice 7 `usabl doctor` | no | merged (#92) |
| C1 | usabl | Item 5 docs truth pass + CHANGELOG | no | merged (#93) |
| C2 | (docs site) | team-orientation + guides to v0.2.0 | no | merged (#94) |
| C3 | usabl, usabl-app | READMEs + CONTRIBUTING to v0.2.0 | no | merged (#95, #23) |
| B2 | usabl-app | merge #21 gitignore; close #20 and #17 with evidence | no | done (#21 merged; #20, #17 closed with evidence) |
| B3 | usabl-app | re-pin CI to the frozen engine commit | yes (workflows) | done (#24; admin-override merge, main 09d92aa; suite green) |
| TAG | both | tag v0.2.0 | yes (tag) | done (usabl v0.2.0 -> 51a9ce3; usabl-app v0.2.0 -> 09d92aa) |

Note: A2, A3, A4 each register a subcommand in `src/cli.ts`. Developed on
parallel worktrees they conflict on that one file, so cli.ts registrations are
resolved at integration time in merge order, each PR rebased on main first.

## Sequencing

Day 1, 2026-08-29:
- B1 first, to unblock usabl-app main.
- Fan out A1 through A4 on worktrees.
- Start C1 (truth pass is mostly independent of the adoption code).
- Integrate slices as they pass review, one PR at a time.

Day 2, 2026-08-30:
- Finish slice merges.
- C2 and C3 (docs), after the final command set is settled.
- D (hero rehearsal, clean-clone proxy, green suites).
- B2 (PR dispositions), then B3 (CI re-pin, live-gated).
- E (evidence bundle), then the tag (live-gated).

Buffer, 2026-08-31: tag and 09 issue if anything slips.

## Risks and fallback

Seven test-first slices plus a full docs overhaul plus rehearsal in two days is
aggressive. Parallel subagents make it reachable. If something must give, the
last-to-land priority is drift (5), then doctor (7); the hero demo and onboarding
work without them. The tag can slip a few hours rather than ship untested. Any
slip is recorded here, not hidden.

## Sources reconciled

- usabl-plans phase docs 00 through 08 (the original build).
- Additional plan docs in innovation-days-2026/: usabl-v0.2.0-plan.md,
  usabl-adoption-slices.md, usabl-v0.3.0-plan.md (the last is 0.2.0 adoption
  work despite its filename).
- 09-verifier-evidence-and-outcomes.md, now scoped to v0.3.0.

## Chunks

- A-adoption.md: the seven slices.
- B-fixes.md: red-main fix, PR dispositions, CI re-pin.
- C-docs.md: truth pass, orientation site, guides, READMEs, CHANGELOG.
- D-demo.md: hero rehearsal, clean-clone proxy, green suites, evidence to capture.
- E-release.md: evidence bundle, tag, v0.3.0 issue, package-distribution note.
- human-tasks.md: the v0.2.1 and later handoff catalogue.
- how-we-built-this.md: living build log for the v0.2.0 finish.
