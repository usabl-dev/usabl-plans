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
- A2 landed as usabl#90 after a second review round. The reviewer confirmed the
  empty-discovery refusal but found that the same honesty failure survived just
  under the zero-match line: a router mixing parseable and unparseable paths
  discovered only the parseable ones and reported the rest as confident
  removals. The security lane separately found a second egress: the router file
  name, which a pull-request author controls through the committed config, was
  printed to stderr without the neutralizer. Rather than patch the symptoms, the
  rework fixed the root cause both lanes had circled. The fallback parser now
  reads the object-property route form (path: '/home') used by the React Router
  data-router API, so mainstream modern routers are compared instead of refused;
  the router file name is neutralized in both refusal messages; and the report
  carries a plain-English caveat that routes usabl cannot statically parse can
  appear as false removals. The duplicate-path crash was also removed. Both
  lanes then returned resolved, and the exit-code contract, the honesty guards,
  and the new parser branch were each mutation-tested (removing a guard fails its
  test). Final state: 500 tests, full check green. Two residuals are tracked as
  their own single-purpose follow-ups rather than bundled into the feature: bidi
  control characters in the shared neutralizer, and a symmetric added-route
  caveat. Both fail in the safe direction, so neither blocks the honesty bar.

- A3 (install family plus stop-hook, slices 6a through 6d) landed as usabl#91,
  one cohesive PR carrying four generators and a new subcommand. The builder
  landed the family first: `usabl install --overlay|--claude|--ci|--branch-rule`,
  each writing a draft only and refusing with a manual step on ambiguous input,
  plus `usabl stop-hook`, a thin stdin reader that always exits 0 so a wedged
  hook can never block continuation through an exit code. The engineer verified
  the build independently (full check green, diff scope correct, the four core
  guards mutation-proven) before routing it to review.
- The slice carried five review lanes at once, one per integration plus a
  whole-slice reviewer. The CI lane returned resolved on the first pass: it
  reconstructed the emitted gate workflow and confirmed it is byte-faithful to
  the fixture, with all of the security properties intact (fork-secret fence,
  trusted-read policy job, SHA-pinned actions, the numeric pull-request guard,
  the read-only engine snapshot, and a documented sentinel in place of a
  fabricated engine commit). The other four returned concerns, and three of them
  were the same shape: a confident wrong success. Recognition that failed toward
  success was the common root. The overlay wiring check scanned raw text, so a
  commented-out plugin read as already wired. The Claude hook ownership check
  used a substring match, so an operator command that merely mentioned the marker
  (a wrapped or foreign command) was rewritten in place and reported as a
  success while the operator's real command was lost. The branch-rule check
  treated any 404 as proof of no protection, so a permission or wrong-repo 404
  became a fabricated not-applied. The reviewer added test-coverage gaps: the
  glue that routes a target to its generator, and the ordering that lets
  `install --ci` write a workflow rather than trip the check-mode trusted-ref
  requirement, were both correct but unlocked by any test.
- All findings went back to the builder as one consolidated rework. The fix in
  every honesty case was to narrow recognition to exact, anchored, positively
  confirmed forms and let everything else fall to the safe direction: refuse or
  cannot-verify. The overlay check now runs on a string-aware scrubbed copy so a
  commented-out or quoted mention cannot count. Claude ownership is now an exact
  or anchored match, and a wrapped, prefixed, piped, or foreign command refuses
  with the file left byte-for-byte unchanged; a second usabl hook now refuses
  rather than hide a double-run. Branch-rule recognizes absence only by GitHub's
  exact "Branch not protected" signal, and its gh reader was narrowed to a
  read-only getJson port so no caller path can turn it into a write. A shared
  correctness gap, missing the .cts and .cjs Vite config forms, was fixed in
  both the overlay and init candidate lists so a config in those forms is no
  longer shadowed by a fresh draft.
- The engineer re-verified the rework independently, mutation-testing each of the
  three tightened honesty guards: breaking a guard flipped exactly its own tests
  red and nothing else, proving the tests load-bearing. All five lanes then
  returned resolved, the whole-slice reviewer re-derived the load-bearing checks
  itself rather than take the claim on faith, and CI was green. Final state: 549
  tests passing, install suite at 49, full check green. Two tidies were cleared
  to ship by the lanes and tracked as their own low-priority follow-ups rather
  than spun into another cycle: a path-segment boundary on the retired-hook
  regex, and a multi-line-import form for the overlay wiring check. Both fail in
  the safe direction (over-refuse, never a false success), so neither blocks the
  honesty bar.

- A4 (doctor, slice 7) landed as usabl#92, a read-only `usabl doctor` that
  projects eight integration surfaces as wired, missing, drifted, or unknown,
  each with one honest next step. It mints no verdict and always exits 0 when it
  renders, so it never becomes a second verdict authority, and unknown is a
  first-class state for a surface usabl cannot positively confirm. The builder
  reused the install family's recognition predicates rather than re-derive
  weaker checks. The engineer verified the first build independently and, before
  routing it to review, found two honesty inversions the reuse had carried in.
- The first was a confident wrong success on the ci surface. Doctor drove off
  planCi's byte-exact already-wired, but the draft workflow ships with the
  engine ref still set to the placeholder sentinel, so a gate that cannot run
  read as wired while a correctly pinned gate read as drifted. The fix was a new
  pin-aware recognizer, classifyGateWorkflow, added next to the draft it checks:
  wired requires both engine-ref lines to carry the same real 40-character
  commit SHA, the sentinel is unpinned (doctor renders it drifted with a pin
  step), and any structural change or inconsistent pin is drifted. The second
  was a conflation on the stop-hook surface. The install plan's single update
  action covered both a settings file with no usabl hook at all and a usabl hook
  that only needed normalizing, and doctor called both drifted with a false
  claim about a retired path. The fix added an UpdateReason discriminator so
  doctor tells the genuinely missing surface from the drifted one. Both changes
  are additive: planCi, writeCi, and writeClaude are byte-unchanged, so the
  merged install family behaves exactly as before.
- The reviewer and the security lane re-derived the fixes independently against
  the reworked commit. The security lane recorded, for the record, that its
  first pass had accepted planCi's byte-exact already-wired as honest and missed
  the sentinel embedded in the draft; on re-derivation it ran crafted-workflow
  attempts (sentinel intact, half-pinned, a real SHA beside a tampered security
  line, trailing content on a ref line, uppercase hex) and confirmed every one
  falls to not-wired, so no repo state reports an unenforceable gate as green.
  The engineer mutation-tested both new guards: breaking the SHA recognizer
  flips only the real-SHA-to-wired tests, and forcing the stop-hook reason to
  normalize flips only the settings-exists-no-hook test while the install suite
  stays green, proving the new field read-only. Final state: 583 tests, full
  check green. This completes the seven adoption slices (4b, 5, 6a through 6d,
  7). Two residuals are tracked as their own low-priority follow-ups rather than
  spun into another cycle: the unpinned next step's reference to install --ci,
  which does not itself reprint the pin steps for an existing unpinned draft, and
  a guard-test for classifyGateWorkflow's coupling to the draft literally
  carrying the sentinel. Both fail in the safe direction, so neither blocks the
  honesty bar.

- C1 (docs truth pass + CHANGELOG, item 5) landed as usabl#93. It reconciled the
  shipped documentation with the code so no doc claims what the engine does not
  back. Four reconciliations: the how-usabl-works and code-walkthrough pages now
  name all four evidence classes and keep the rule that only deterministic
  evidence can change the verdict, matching the EvidenceClass contract; the docs
  and the receipt docstring now say a receipt binds four things (source tree,
  policy hash, runner version, scanner versions), matching what verifyReceipt
  actually compares; the receipt reference now states base revision is recorded
  for context and is not compared on re-verification; and the walkthrough and
  ground truth now state the voicing lane is built and exported but not wired
  into the v0.2.0 run path, landing in v0.3.0. The CHANGELOG gained the full
  adoption command set and a short note disclosing the foundational commands
  carried from 0.1.0, including the bypass escape hatch that does not verify,
  labeled as pre-existing so it makes no false version attribution. The slice
  also added an offline documentation link checker, run via npm run docs:linkcheck,
  that validates relative links, local files, and heading anchors without touching
  the network.
- The engineer verified the first build independently and found two honesty
  defects. The builder had removed a `npm run lint` line from the usabl-app
  contribution guide, calling it a broken reference, but that command exists in
  usabl-app's package.json; the line was restored to net-zero. The new link
  checker silently blessed four links that resolve on this filesystem but escape
  the repo root and would break in a clean clone. That is the tool's own version
  of recognition that fails toward success, so the fix was to disclose those
  links as their own category and never bless them, added test-first and verified
  load-bearing by mutation: breaking the escape guard flipped exactly the one
  escape test and left the four others green. A missing escaping file still
  reports broken and exits non-zero, so the checker never fails toward success.
- The engineering lane then flagged that the CHANGELOG omitted the shipped bypass
  escape hatch and did not name comment. Before acting, the engineer verified that
  both commands shipped in v0.1.0, not 0.2.0, and that there is no v0.1.0 CHANGELOG
  section. The honest fix was therefore a disclosure note, not an Added entry:
  adding them to the 0.2.0 Added list would have falsely claimed 0.2.0 introduced
  them, turning the truth pass into a new untruth. The note discloses check,
  comment, and bypass as carried from 0.1.0, with bypass described accurately
  against the code as a one-time, next-stop-only marker that skips the next Stop
  hook. Both review lanes returned satisfied, each re-deriving its own concern
  rather than taking the fix on faith; full check green, docs:linkcheck green with
  the four cross-repo escapes disclosed.
- Residuals tracked rather than spun into another cycle. Four research links in
  the docs corpus point at ../../research and escape the repo root; they resolve
  here but would break in a clean clone, so they are a content decision for the
  docs-site and README work, not a code defect. Two security observations on the
  new checker are immaterial for a dev-time read-only tool: it resolves symlinks
  lexically rather than via realpath, and it checks link existence rather than
  active content. One residual does not fail in the safe direction and is tracked
  for exactly that reason: the checker extracts inline markdown links and href
  and src attributes, but not reference-style links or autolinks, so a broken link
  in one of those forms would pass unnoticed. The current corpus uses neither
  form, so the gap is latent, but it is a recognition-that-fails-toward-success
  gap in an honesty tool and belongs on the hardening list, not dismissed as
  safe-direction.

- C2 (team-orientation page to v0.2.0) landed as usabl#94. The page is published
  on public GitHub Pages from main under docs, so it is the first thing a new
  teammate reads. It was still framed as a countdown to the contest with dated
  week-by-week cards. The slice replaced that with a team hand-off section that
  states v0.2.0 is ready to pick up and lays out three date-free onboarding steps,
  each with an explicit "ready when" line, and renamed the nav item and its anchor
  from demo to handoff. It added an accessible table of the six adoption commands
  a teammate actually runs (init, baseline, floor prune, drift routes, install,
  doctor), each description checked against src/cli.ts and the CHANGELOG, framed
  by a sentence that keeps the honesty boundary explicit: none of these commands
  mints a verdict, only the gate does. It pointed teammates at the human-tasks
  catalogue as plain text rather than a link, because that catalogue lives in the
  private plans repo and a public page must not ship a link that 404s for the
  reader. It also carried the claim boundary forward, changing the verified-
  evidence line to name all four receipt bindings (code, policy, runner, and
  scanner stack) so the page matches the receipt truth C1 established.
- The engineer verified the change independently before routing it to review:
  scope confined to the one HTML file, the Content-Security-Policy meta untouched,
  no private-repo link, no dangling demo anchor, and docs:linkcheck green (74
  relative links, zero broken, the four pre-existing cross-repo research escapes
  unchanged). A working-tree slip during that pass (a stray git checkout of a path
  from main overwrote the working copy, so a link count briefly read one low) was
  caught by the count regression and restored from HEAD; the commit itself was
  never touched.
- Two lanes reviewed without pre-shared findings: one for content, honesty, and
  the page's own accessibility, one for security. The content lane confirmed each
  command description against the code (the four receipt bindings at
  receipt.ts, the install target flags and the read-only branch-rule reader at
  cli.ts), that every anchor and aria-labelledby target resolves with no skipped
  heading levels, and that the new command table is a keyboard-reachable labelled
  region with scoped header cells. The security lane confirmed the CSP still
  forbids scripts and connections, that nothing active or external was added, and
  that the private plans repo appears only as plain text. Both returned satisfied.
  Both raised the same one non-blocking nit: the page named the rehearsal
  "two-owner" while the catalogue heading calls it "two-operator." Because the
  page points a reader at that catalogue, the term was aligned to the catalogue so
  the entry is findable rather than shipped as a known drift. Final state: full
  check green in CI, docs:linkcheck green.

- C3 (READMEs, CONTRIBUTING, and guides to v0.2.0) landed as two PRs, usabl#95
  and usabl-app#23. The engine repo gained a README "Command surface" section
  that lists every command the CLI registers, grouped by role and framed so the
  reader knows only the gate decides a verdict. CONTRIBUTING's build process was
  rewritten from a stale vendor-specific description (it named particular model
  families) into three vendor-neutral lanes: implement, review, and security
  review, with the rule that the lane which writes a change does not review it or
  own the merge security gate. The contribution guide and the how-usabl-works
  page now name usabl baseline and usabl floor prune as draft-writing floor
  commands that mint no verdict. The team fixture's runbook gained an "Adopt
  usabl in your own repository" section covering the same adoption and lifecycle
  commands, so a teammate can wire usabl into their own repo, not only drive the
  fixture.
- The engineer verified the first build independently and made two corrections
  the builder had honestly flagged or missed. The README command list omitted
  usabl docs, the twelfth command, so the list did not match the CLI; it was
  added under a Reporting heading as a generator that projects artifacts and
  mints no verdict. The stop-hook entry described the command as a thin stdin
  reader that always exits 0, which inverted its purpose: usabl stop-hook runs
  the gate when the assistant tries to finish and blocks continuation through the
  Stop hook decision, and the exit-0 property exists so a wedged or errored hook
  fails open with disclosure rather than blocking through an exit code. Both the
  engine README and the runbook were corrected to say the command is the gate for
  continuation, not a passthrough.
- Two lanes reviewed without pre-shared findings, one for content, honesty, and
  page accessibility, one for security. The security lane returned clean: both
  changes are documentation only, touch no guarded path, add no script or network
  resource, leave the how-usabl-works Content-Security-Policy intact, introduce no
  secret, and add no link to the private plans repo. The content lane verified
  every one of the twelve command descriptions against the engine source rather
  than the CHANGELOG, confirmed the command list is complete, confirmed the
  CONTRIBUTING rewrite names no vendor, and confirmed the CSP, heading levels, and
  style. It found one nit, in the safe direction: the corrected stop-hook wording
  narrowed the block condition to an unresolved regression, while the engine
  blocks on three verdicts (regression, not covered, and approval required) plus
  guarded policy drift detected within the session. The wording was broadened to
  match, verified against the BLOCKING_VERDICTS set in stop-hook.ts and the
  session-drift path in stop-hook-runner.ts. This nit is itself an instance of the
  slice's own discipline: a doc that understated an enforcement surface was
  corrected to match the code.
- Final state: usabl#95 full check green in CI and locally, docs:linkcheck green
  (74 relative links, zero broken, the four pre-existing cross-repo research
  escapes unchanged); usabl-app#23 gate-comment and policy checks green. Both
  squash-merged. This completes the documentation handoff for v0.2.0 except the
  ground-truth reconciliation already covered in C1. Remaining before the tag: the
  hero rehearsal and clean-clone proxy (D), the pull-request dispositions (B2),
  the guarded CI re-pin (B3), and the evidence bundle (E).

## D, demo readiness (2026-08-30)

- D1, the hero loop, was run end to end against the fixture and captured on all
  four surfaces. To produce a genuine verified verdict rather than an idle
  no-diff result, the loop followed the runbook shape: commit the broken source
  on a throwaway branch so the base is broken, then repair the working tree so
  the diff-aware gate sees a real repair. Broken source returned regression, exit
  1, eight gating findings on the two screens the change reaches through the route
  graph; repaired source returned verified, exit 0, zero findings, with a receipt
  bound to source tree, policy hash, runner version, and scanner stack. The CLI,
  the overlay result endpoint, the pull-request comment, and the stop hook all
  agreed on both states, including the identical receipt sourceTree hash on every
  surface that carries one. The stop hook blocked on regression through a stdout
  decision and allowed on verified with the receipt disclosed on stderr, both at
  exit 0, which matches the fail-open contract documented in C3.
- The run corrected two predictions in the D plan, both in the safe direction.
  The regression exit code is 1, not 2. The predicted pf-toast-live-region
  finding does not appear, and correctly so: a toast enters the DOM only after a
  click, and the rule short-circuits on an empty alert set and proves containment
  only, refusing the temporal announcement half. An absent alert is silence, not
  a false finding. The plan was updated to the verified results.
- D3, the green suites, passed. The engine check is green: 588 tests pass and 7
  skip across 92 files, then typecheck, build, DTS, and the package smoke all
  pass, exit 0. The seven skips are the live integration tests skipped by
  default. The fixture is green: 21 tests across 8 files, typecheck and lint
  clean. One honesty note worth keeping: a leftover git worktree under the
  gitignored .work directory made vitest discover a stale second copy of the
  fixture suite and report a phantom failure on an old engine pin. Gitignore
  hides a path from git, not from the test runner. Removing the worktree restored
  the pristine counts.
- D2, the clean-clone onboarding proxy, passed. It ran as a background agent from
  fresh clones of both repositories, which landed on the exact frozen commits, and
  the finishing engineer then re-read the raw command outputs to confirm the
  report. The full walk, prepare plus eight onboarding commands, took 109 seconds,
  about 1.8 minutes, well under the 30-minute north star. Four of the eight
  commands exited non-zero, and every one is an honest refusal or verdict in the
  safe direction rather than a crash: init refused to overwrite existing config
  without force, install --ci refused to clobber a hand-tuned security workflow,
  install --branch-rule reported it cannot set GitHub branch protection from the
  CLI and printed the exact setting instead, and check asked for approval because
  baseline had dirtied the guarded evidence ledger. Doctor reported seven wired
  and one missing, the branch protection, and restated that it mints no verdict.
  Only two files ended modified. Eleven refusals or manual steps were captured for
  the human onboarding script, two of them hard human-required limits. The result
  is labelled agent-run, not human-confirmed, and feeds the evidence bundle in E.

(append entries as work lands)
