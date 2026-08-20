# How we built this

Living record of how usabl went from an idea to running software. Append a dated
entry when something true changes. Do not rewrite history to look smoother than it
was. The point is a story a judge, a teammate, or a later us can follow.

Roles: [`roles.md`](roles.md). Plans: this directory. Product spec:
`usabl/docs/ground-truth.md`. Decisions that closed open spec forks:
`../usabl-decisions-2026-08-19.md`.

---

## The idea

Most accessibility tools scan and advise. A human may or may not read the list. AI
assistants now write a large share of UI, and they are happy to call the work done
while a screen reader still cannot use the screen.

usabl is a **proof engine**. It checks the surfaces a change actually touched and
returns one of four answers a person can trust: `verified`, `regression`,
`not_covered`, `approval_required`. Idle (nothing UI-touching changed) is not a
fifth verdict. It is `verdict: null`, exit 0, and an explicit "nothing to check."

The AI can suggest fixes. It does not get to grade its own work. The gate is the
only verdict authority. `verified` and receipts are reserved for reproducible
deterministic evidence. Preview and model-judgment surface. They never mint
`verified`. They never write a receipt. A surface we cannot fully exercise is
`not_covered` with a reason, never a silent pass.

PatternFly is the first instantiation. The engine is check-agnostic: any provider
that returns `Draft[]` can plug in.

Tagline: usable by default. Supporting: Don't ship until it's usabl. The product
name is always lowercase.

---

## Spec, then design, then build

We did not start in the compiler.

1. **Thesis and positioning.** What usabl is (proof, not another scanner wrapper),
   who it is for (Priya shipping UI, James using a screen reader, teams with a
   merge gate), and how it compares. Lives in `usabl/docs/positioning.md` and
   related product docs.
2. **Ground truth.** One consolidated design replaced scattered shared-design /
   contest-plan / team-plan drafts. Architecture, four verdicts, evidence classes,
   receipts, guard/trust, surfaces, adoption, WCAG map, demo strategy. Author:
   eparenti, August 2026. This is the product spec.
3. **Settled forks (2026-08-19).** A decisions file closed what ground truth still
   left open: NVDA as hero demo voice and Orca on Fedora as validation/dogfood
   reader; one npm package; Bash-only mid-task check for the contest (MCP is a
   seam); product repo vs fixture vs `usabl-app` consumer vs Fleet Insights as a
   measurement target; voicing as preview unless a class is later promoted by
   measurement.
4. **Implementation plans.** Phases 1 through 7 live here, not in the product repo,
   so plans never ship inside the tool. Phase 1 is the engine kernel: contracts,
   gate, `run()`, receipt, CLI, golden oracle. Later phases add providers, real
   coverage/guard, voicing, surfaces, intake, demo.
5. **Then code.** TDD. Coherence-slice PRs (one sentence a judge can read), not
   one PR per primitive and not one dump of a whole phase.

Contest scoring (Innovation, Feasibility, User Experience, Technical Excellence)
is why the bar goes up, not down. We do not excuse weak design with "it is just a
demo."

---

## How the team builds

Founder (Ed) is chief of product and UX lead. Architect/CTO oversees and does not
write product code. This product is built with multiple models. Codex 5.3 implements
(TDD, commit, report). Opus 4.6 is engineering manager and senior reviewer after each
task. Gemini 3.1 pro is CISO: every PR vs `main` before merge, independent of Opus.

Commits are Conventional Commits, human author. Push in slices after the founder
says yes. Comments teach *why* in the file. They do not cite plan task numbers or
ground-truth section marks.

---

## What shipped on `main` (through 2026-08-20)

GitHub history is the public story. Each merged PR is one idea. Phase 1 is complete.

| Slice | PR | What became true |
| --- | --- | --- |
| 0 | [4](https://github.com/usabl-dev/usabl/pull/4) | Product language is locked: lowercase usabl, on/off (no modes), NVDA/Orca split. |
| 1 | [5](https://github.com/usabl-dev/usabl/pull/5) | The repo is a real TypeScript project. CI runs `npm ci` and `npm run check` on Node 22. |
| 2 | [6](https://github.com/usabl-dev/usabl/pull/6) | Engine vocabulary is frozen in `src/contracts`. |
| 3 | [7](https://github.com/usabl-dev/usabl/pull/7) | Receipts can be ordered and hashed (`sortBy`, canonicalize, sha256). |
| 4 | [8](https://github.com/usabl-dev/usabl/pull/8) | Tests can run the engine in memory (identity + fakes). |
| 5 | [9](https://github.com/usabl-dev/usabl/pull/9) | The gate is the only verdict authority (filter, differential, waivers). |
| 6 | [10](https://github.com/usabl-dev/usabl/pull/10) | You can invoke the engine: guard, receipt, `run()`, CLI, conformance, golden oracle. |

`buildDeps()` still throws. That is honest. Real browser, git, and providers are
later phases. Tests inject `makeFakeDeps`. The golden oracle pins idle, verified
(with receipt), regression, not_covered, and approval_required.

Stacked PRs 8-10 were squash-merged in order. After each squash, the next branch
was rebased onto new `main` (force-with-lease of the feature branch, never
`main`). Retargeting onto `main` does not start Actions unless the workflow
lists the `edited` type; we close/reopened 9 and 10 to start `check`, then
fixed the workflow.

---

## Things we learned in the build (keep)

- A slice is a PR. A task is not. Fifteen task commits on a working branch, three
  PRs a judge can read.
- Stacked PRs plus squash rewrite feature-branch SHAs. Other clones that had
  those branches checked out must `fetch` and `reset --hard` to the remote.
  Phase 2 does not stack: each slice is cut from current `main`.
- `pull_request` without `types: [edited]` does not run when a PR is retargeted
  onto `main`. Close/reopen works; listing `edited` is the real fix.
- Sample config URLs like `http://127.0.0.1:5173` are the fixture app's Vite
  origin, not a hardcoded engine target. The engine reads config.
- Comments that say "see ground-truth §5" rot. Comments that state the invariant
  do not.
- The guard compares file bytes. Listing `src/gate` as a directory in config was a
  lying claim. We list `src/gate/index.ts` until directory expansion exists.
- `formatSummary` is already an egress. Neutralize page-derived text there before
  Playwright can fill `whatUserExperiences` / `fix`. Parked as a hard gate of
  plan 02, written in `usabl/docs/threat-model.md`.
- `verdict: null` means the gate did not mint a verdict. Idle is one case. Fail
  open (exit 4) is another. The old "null iff nothingToCheck" comment was too
  tight.

---

## What is not built yet (honest)

Providers, real `CheckRunner`, route-graph coverage, directory-shaped guarded
paths, CI trusted-ref, voicing lane, stop hook, overlay, MCP. `neutralize()` on
CLI finding text is the first Phase 2 slice; until it lands, live scanning stays
unwired. If a path is not built, it fails honestly or is not wired.

---

## Log

Append here. Newest last. One entry per real turn in the story.

### 2026-08 - Spec and design

Ground truth, positioning, threat model, adoption model, decisions file, plan set
01 through 07.

### 2026-08-20 - Phase 1 kernel in code

Slices 0 through 3 on `main`. Slices 4 through 6 implemented, opened as PRs 8
through 10, CISO reviews run, guard-path honesty fix on PR 10, neutralize parked
for the detection-engine phase. Team roles written (`roles.md`). This file
started.

### 2026-08-20 - Phase 1 on main; Phase 2 sliced

PRs 8, 9, and 10 squash-merged in order (identity, gate, `run()` / CLI / oracle).
Phase 1 kernel is on `main`. `buildDeps()` still throws. Neutralize remains the
hard gate before a live CheckRunner. Phase 2 slice map: [`02-slices.md`](02-slices.md).
Do not stack those PRs.

### 2026-08-20 - Keep the bar

Wrote [`quality-bar.md`](quality-bar.md). Long sessions forget coaching. Subagents
start empty. CTO enforces: paste the bar every dispatch, never skip Opus or CISO,
never self-approve a failed dispatch, experiments allowed, honesty not.
