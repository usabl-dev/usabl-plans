# v0.1.0 team demo (internal)

Audience: usabl-dev teammates, all on Claude Code. This is a team preview they
can clone, run, and file issues against. It is **not** the contest pitch.
The contest script is written after this round of feedback.

Product name is **usabl** (lowercase). Fixture repo is **usabl-app**.

Engine `main` at spec time: `1dc0ed8` (`js-yaml` 4.3.1, #45).

## Goal

A teammate, without the founder in the room, can:

1. Clone `usabl` and `usabl-app` as siblings, install, and build.
2. Feel the hero bug in the browser (modal focus not returned on close).
3. See the same gated Result on four Monday surfaces: CLI, overlay, stop-hook,
   and PR/CI comment.
4. Fix the modal in **code** and watch the ratchet go to `verified`.
5. File feedback with a GitHub issue template.

## What this is not

- Not a judge script. No `usabl demo` command. No npm publish.
- Not Fleet Insights live measurement. Not MCP.
- Not the query-string contest trick as the team ratchet. `?variant=fixed`
  stays in the fixture for later. Teammates do not edit `usabl.config.json`
  to flip variants (that file is guarded and would mint `approval_required`).

## Honesty

- The gate is the only verdict authority. Overlay and PR comment are
  projections of an already-gated `Result`. They never re-derive findings.
- Overlay is advisory (`displayExitCode` stays 0). Stop-hook and CI gate.
- `verified` and receipts stay reserved for deterministic evidence.
- Page-derived text is untrusted. Neutralize at every egress (already true
  on CLI, overlay, PR comment).
- A docs-only PR is idle (`verdict: null`), not a fake green. UI-touching
  PRs against default `/clusters` are `regression` until the modal is fixed.

Frozen names (landed contracts, not plan aliases): `Result`, `verdict`,
`exitCode`, `run()`, `buildDeps`, `UsablConfig` (`appBaseUrl`, `uiFileGlobs`,
`discovery.routerFile`, `wideBlastGlobs`, `surfaces`, `guardedPaths`),
hero rule `pf-modal-focus-return`. CLI: `usabl check --ci --trusted-ref`,
`usabl comment`. Overlay export: `usabl/vite`.

## Layout

```
<parent>/
  usabl/        engine, version 0.1.0, git tag v0.1.0
  usabl-app/    fixture, version 0.1.0
```

Local fixture depends on the engine with `"usabl": "file:../usabl"` after
`npm run build` in `usabl` (needs `dist/`).

## The loop

1. Node 22. Clone both repos as siblings. `npm install` in each.
2. In `usabl`: `npm run build` (CLI, overlay plugin, stop-hook runner).
3. In `usabl-app`: `npm run dev` on `http://127.0.0.1:5173`.
4. Browser: `/clusters`. Trigger is "View cluster details" with
   `aria-haspopup="dialog"`. Open, Escape: focus is lost.
5. From `usabl-app` cwd: `npx usabl check` (or `node ../usabl/dist/cli.js check`).
   Expect `regression`, `pf-modal-focus-return`, `exitCode` 1, `receipt` null.
6. Same browser: overlay badge is advisory and shows that Result. `?usabl=off`
   hides it. Playwright `navigator.webdriver` already skips the loader.
7. Open Claude Code **on `usabl-app`**. Stop hook is already
   `node ../usabl/dist/stop-hook-runner.js`. A stop during a UI-touching
   session blocks on regression (`{"decision":"block"}` on stdout).
8. Fix `DemoModal`: keep the PatternFly trap; restore `triggerEl.focus()` on
   close as the default path (not behind `variant === 'fixed'` only). Next
   `usabl check` is `verified` with a receipt.
9. Open a PR on `usabl-app` that touches `src/pages/Clusters.tsx` (or the
   modal). Unfixed: sticky `<!-- usabl-report -->` comment + failing check.
   Fixed: `VERIFIED`. Docs-only PR: idle.

## Overlay wiring (engine + fixture)

`usablVitePlugin` today takes `{ run: () => Promise<Result> }`. `buildDeps`
and `loadConfig` are not on the public `usabl` barrel.

Add a host factory on the existing `usabl/vite` export so the fixture does
not assemble `Deps`:

```ts
usablVitePluginFromConfig(opts?: { cwd?: string; configPath?: string })
```

It loads `usabl.config.json`, calls `buildDeps`, then `run()`, and closes the
browser after each run. It never mints a verdict. Fixture `vite.config.ts`
adds that plugin next to React.

Keep `usabl=off` and `webdriver` skip so live oracles stay honest.

## PR/CI on usabl-app

Copy the **gate** lessons from `usabl/.github/workflows/usabl-gate.yml`, not
the engine's typecheck job:

- `--ci --trusted-ref "origin/${BASE_REF}"`. Refuse without it.
- Capture exit with `|| code=$?` on the same line. Fail closed if empty.
- `node .../cli.js comment < usabl-result.json` then sticky comment among
  bot comments that carry `<!-- usabl-report -->`.
- Actions pinned by SHA. `github.base_ref` only through `env:`.
- Start the fixture on `127.0.0.1:5173` so committed `usabl.config.json`
  URLs work. Install Chromium from the engine's Playwright.

**Private checkout.** `file:../usabl` does not exist on GitHub Actions.
Checkout `usabl-dev/usabl` at tag **`v0.1.0`** into `usabl-app/.usabl-engine/`
(gitignored) using secret **`USABL_ENGINE_CHECKOUT_TOKEN`** (read-only on
`usabl-dev/usabl`). Install that tree, build it, run its CLI. Never put a
token in YAML.

Founder creates the secret before slice D is useful.

## Feedback

Issue template in `usabl`:

- Surface: `cli` | `overlay` | `stop-hook` | `ci-pr`
- Verdict seen
- Expected
- Repro (commands, URLs, PR link)
- Honest / confusing (free text)

Fixture-only bugs can still use that template. `usabl-app` README points
here. No contest language in the template.

## Docs and version

- `usabl` `package.json` version `0.1.0`. README: team preview status, sibling
  clone, pointer to the runbook. Drop "early development" / `0.0.0` badges.
- Runbook: `usabl/docs/team-preview.md` (internal loop, not a pitch).
- `CONTRIBUTING.md`: link the runbook.
- `usabl-app` README: this is the PF6 fixture, not a Vite template. Sibling
  layout, `npm run build` in usabl first, overlay, stop-hook, CI.
- Git tag `v0.1.0` on usabl after the version PR lands (CI pin).
- Hygiene: gitignore `.pre-commit-cache/` in usabl if it is still untracked
  noise.

## Slice map (do not stack)

Cut each from current `main` of that repo. Squash-merge. Ask before each push.

| Slice | Repo | Story |
| --- | --- | --- |
| A | `usabl` | v0.1.0, team-preview runbook, issue template, `.pre-commit-cache/` gitignore |
| B | `usabl` | `usablVitePluginFromConfig` on `usabl/vite`; tag `v0.1.0` after merge |
| C | `usabl-app` | `file:../usabl`, overlay plugin, README, version 0.1.0 |
| D | `usabl-app` | PR/CI gate workflow + `.usabl-engine/` gitignore |

Slice D needs tag `v0.1.0` and the checkout secret.

Engine work is Codex in `usabl`. Fixture work is Codex in `usabl-app`. Opus
after each task. CISO on every PR vs that repo's `main`.

## Parked

- Contest demo script
- Query-string as the official flip
- Fleet Insights live SSO run
- MCP
- Enable Dependabot alerts on `usabl-app` (separate ops)
- Remaining usabl lockfile alerts (`vitest` / `vite` / `esbuild`)
