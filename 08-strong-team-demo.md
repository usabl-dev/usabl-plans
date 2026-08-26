# Strong Team Demo Implementation Plan

**Status:** confirmed for implementation on 2026-08-26.

**Version:** keep `usabl` and `usabl-app` at v0.1.0.

**Current baseline:** planning starts in the middle of Week 2. In `usabl-app`,
PR 1 wired the dev overlay and PR 2 added the trusted PR gate. Both are merged.
Future work uses slice names in this plan. GitHub assigns the real PR number when
each slice opens.

## Outcome

Build one connected team demonstration of usabl across the real development
surfaces:

```text
Broken application
  → dev overlay reports regression
  → Claude runs a mid-session self-check
  → stop hook blocks incomplete work
  → Claude repairs the accessibility barriers
  → overlay and self-check report verified
  → stop hook accepts completion
  → PR and CI report the same verified result
  → Fleet Insights provides real-app measurement evidence
```

The demo is successful when a teammate who did not build usabl can explain:

1. What a person experiences when the interface is broken.
2. What usabl checked and what it could not check.
3. Why Claude could not call incomplete work done.
4. Why the repaired source is verified.
5. Why the Fleet Insights run is external evidence, not fixture proof.

## Product decisions

### One result across real surfaces

The dev overlay, CLI self-check, Claude stop hook, and PR comment all project the
same gate-owned `Result`. No demo surface calculates a finding, verdict, coverage
answer, or receipt.

When two surfaces disagree for the same code state, that is a product defect.

### Real application first

`usabl-app` becomes a realistic PatternFly operations workflow. It is not a page
of disconnected broken controls. Every planted barrier belongs to a task a person
can attempt.

### The overlay is a product surface

Upgrade the real usabl Vite overlay. Do not build a second demo-only inspector.
The overlay remains advisory, read-only, and excluded from the engine's browser
scan.

### Broken and repaired previews are not proof

The fixture can expose focused scenario and preview controls for teaching. Those
controls do not change the checked source state and must say so.

The proof loop uses real source changes. Query parameters are never presented as
the accessibility ratchet or as evidence that code was repaired.

### Real app evidence stays honest

Fleet Insights begins as a measurement-only run. It does not gate, mint a receipt,
or claim verified until its routes, authentication, coverage, and policy have been
reviewed.

## Demo architecture

### Fixture layout

Use an app-first layout:

- Realistic operations navigation and content occupy the main workspace.
- A docked accessibility inspector occupies the right side when open.
- A compact demo control strip selects a scenario and preview state.
- Demo controls are visually and semantically separate from the application under
  test.
- The inspector can collapse to a status button for normal development.

The demo control strip is fixture code. The inspector is usabl product code.

### Window sequence

The operator uses real windows:

1. Browser running `usabl-app`.
2. Claude Code working in the demo branch.
3. GitHub showing the pull request and checks.

Do not recreate Claude or GitHub inside `usabl-app`.

### Source state

Prepare a disposable demo worktree from current `main`. Apply a reviewed demo
feature seed that adds a realistic workflow with accessibility barriers. The
resulting files remain real Git changes.

The seed must satisfy these rules:

- It changes mapped user-interface files.
- It introduces the complete workflow and its barriers.
- The repaired workflow remains a meaningful source diff from `main`.
- It does not edit policy, the evidence floor, waivers, or gate code.
- It can be applied repeatedly in a disposable worktree.
- Its resulting source tree is recorded for rehearsal evidence.

The demo setup must not depend on an untracked dependency link or a query-only
variant.

## Fixture scenario set

Target six to ten distinct findings across five user tasks. The exact count becomes
an integration oracle only after the rendered fixture and provider output have been
reviewed. Do not add defects only to increase the number.

### Scenario 1: Cluster details dialog

User task: open cluster details, inspect the content, close the dialog, and continue
from the same control.

Broken behavior:

- Focus does not enter the dialog.
- Focus does not return to the trigger after close.

Expected rules:

- `pf-focus-into-dialog`
- `pf-modal-focus-return`

### Scenario 2: Deployment table actions

User task: inspect a deployment and open the correct row action.

Broken behavior:

- One icon action has no accessible name.
- Repeated row actions announce the same name.
- Header relationships do not preserve row and column context.

Expected rules:

- `button-name` or the deduplicated PatternFly equivalent
- `pf-row-action-name-unique`
- `pf-table-header-assoc`

### Scenario 3: Deployment action menu

User task: open the action menu and select an operation with a keyboard.

Broken behavior:

- The toggle does not expose expanded state.
- Focus remains on the toggle after the menu opens.

Expected rule:

- `pf-kebab-expanded-state`

The static and interaction evidence may deduplicate into one user-facing finding.

### Scenario 4: Deployment notification

User task: start a deployment and learn whether the action succeeded.

Broken behavior:

- A toast appears visually but is not announced.

Expected rule:

- `pf-toast-live-region`

### Scenario 5: Filters and status

User task: filter deployments and understand the resulting status.

Broken behavior:

- Repeated toolbars do not have distinct names.
- One form control lacks an accessible label.
- One important status does not meet minimum contrast.

Expected rules:

- `pf-toolbar-labeled-when-repeated`
- the applicable axe label rule
- `color-contrast`

### Required states

The fixture provides:

- One focused preview per scenario.
- One `all broken` preview.
- One repaired preview.
- One clean control route with no planted barriers.

The clean control route remains a false-positive oracle.

## Accessibility inspector

### Projection contract

Extend the current overlay projection only with fields already owned by `Result`.
The projection needs:

- `verdict`
- `summary`
- `exitCode` for display only
- `coverage.affected`
- `coverage.unresolvedFiles`
- `coverage.gaps`
- finding rule, screen, layer, severity, status, confidence, element identity,
  user experience, reason, and repair
- receipt summary when present
- dirty guarded paths

The overlay keeps `advisory: true` and `displayExitCode: 0`.

Scrub and frame untrusted page-derived text before it reaches the browser client.
Render it with text nodes. Never use page-derived HTML.

### Inspector hierarchy

Header:

- usabl
- verdict label and symbol
- finding count
- scan state
- collapse control

Coverage summary:

- affected screens
- unresolved files
- explicit gaps

Finding list:

- group by screen by default
- use the user-experience text as the primary finding label
- show rule and provider as supporting detail
- distinguish new, carried, fixed, waived, and unverified

Selected finding:

- what the person experiences
- why it matters
- suggested repair
- screen, element identity, rule, provider, and severity

Verified state:

- receipt source-tree prefix
- checked screens
- policy and runner binding
- active waiver count

### Inspector states

Implement and test:

- Loading or scanning
- Idle
- Regression
- Not covered
- Approval required
- Verified
- Error with `NOT verified`
- Empty finding list
- Long user-impact and repair text
- Many findings

### Inspector accessibility

- Logical landmarks and headings
- Full keyboard operation
- Visible focus
- Escape closes only the inspector detail or inspector, not the host workflow
- Focus returns to the collapse control after close
- Status uses text and symbols, not color alone
- Minimum WCAG 2.2 AA contrast
- Usable at 200 percent zoom
- Reduced motion support
- No focus trap when the inspector is non-modal
- No effect on the host application's heading order or accessible name computation

Test the inspector against a clean host page. Intentional fixture defects must not
hide accessibility defects in the inspector itself.

## Claude demonstration

### Mid-session check

Claude runs:

```bash
npx usabl check --self-check
```

The command remains advisory and exits zero. Its output must state that the stop
hook is the gate.

For a committed branch comparison, the rehearsed command may add:

```bash
--trusted-ref origin/main
```

Do not use `--ci` in the local Claude session.

### Stop hook

The existing Claude hook runs automatically when Claude tries to finish.

The demo must show:

1. A mid-session regression result.
2. An incomplete first repair.
3. A stop-hook block naming a remaining barrier.
4. A second repair pass.
5. A verified self-check and receipt.
6. Stop-hook acceptance for the repaired source.

The incomplete first repair must be deterministic. Do not depend on Claude
forgetting a barrier by chance. The rehearsed task should divide the work into an
initial requested repair and the remaining gate-owned accessibility work.

### Claude prompt

The prompt must be direct and reusable. It should tell Claude:

- the user task to make accessible
- to inspect the current usabl result
- to run the mid-session self-check
- to change source and tests
- not to edit policy, the evidence floor, waivers, or gate code
- to report the final receipt

Keep the prompt in the demo runbook. Do not hide instructions in application text.

### Failure recovery

Prepare for:

- Claude does not run the self-check
- the stop hook is not installed
- the check returns Idle
- the check returns `not_covered`
- a receipt is stale
- Claude edits a guarded path
- the browser or dev server is unavailable

Every recovery step must preserve the honest result. Never replace a failed live
check with a screenshot and call the current run verified.

## Pull request and CI demonstration

Use one real draft pull request with two observable states:

1. Broken source produces `regression`.
2. Repaired source produces `verified`.

The GitHub workflow must:

- use the isolated installed engine
- require `--ci --trusted-ref`
- read trusted policy from the protected base
- fail closed when no exit code is produced
- post or update one sticky usabl comment
- preserve the canonical schema version

The operator shows:

- failing check
- sticky comment with the same finding identity seen locally
- repair commit
- rerun in progress
- verified check and updated comment
- receipt bound to the repaired source

Capture backup evidence for both pull request states. Label each artifact with the
repository, commit, source-tree hash, and run time.

## Fleet Insights validation

Target:

`https://fleet-insights.apps.engineering.openshift.org/`

An unauthenticated request currently returns HTTP 403. The measurement needs a real
authenticated browser session.

### Authentication

- A human signs in through the normal Red Hat authentication flow.
- Export Playwright `storageState` once for the measurement session.
- Store it outside the repository.
- Never print, commit, upload, or include it in demo artifacts.
- Delete or rotate it after the measurement window.

### Route selection

After authentication, choose a small stable route set that covers:

- a page shell and navigation
- a data table or list
- a details workflow
- a filter or form
- an interaction that opens a dialog, menu, or notification when available

Record the final URLs only after live discovery. Do not invent them from the
unauthenticated response.

### Measurement output

Record:

- screens attempted
- screens loaded
- total drafts
- findings grouped by provider and screen
- coverage gaps with reasons
- capability denials
- scan duration
- false positives
- false negatives found by human review
- selectors or interactions that did not generalize

The measurement run does not call the gate or mint a receipt.

### Review

An accessibility reviewer inspects every reported finding on the selected routes.
Classify each as:

- confirmed barrier
- false positive
- needs human judgment
- coverage gap

Any engine change found through Fleet Insights gets its own focused `usabl` issue
and pull request. Do not patch the engine inside the measurement record.

### Team demo use

Show a short Fleet Insights segment after the controlled fixture proof:

- authenticated real page
- routes attempted
- findings and gaps
- one confirmed useful result
- one limitation or coverage gap when present

Label it `measurement-only`. Do not show a green verdict or receipt.

Keep a redacted local backup report in case authentication is unavailable during the
team session.

## Run of show

Target twelve minutes for the initial team demo.

| Time | Surface | Action | Evidence |
| --- | --- | --- | --- |
| 0:00 | Browser | Attempt the broken deployment workflow | People experience the barriers |
| 1:30 | Overlay | Open inspector and select findings | Regression, coverage, user impact, repair |
| 3:00 | Claude | Run mid-session self-check | Same regression and finding identity |
| 4:00 | Claude | Complete the first repair pass | Stop hook blocks on remaining work |
| 5:00 | Browser and Claude | Repair remaining barriers | Overlay updates from the real source |
| 7:00 | Claude | Run self-check and finish | Verified receipt and stop acceptance |
| 8:30 | GitHub | Show draft PR before and after repair | Sticky comment and CI result agree |
| 10:30 | Fleet Insights | Show real-app measurement | Confirmed evidence plus honest gaps |
| 11:30 | Browser | State the claim boundary | What is proven and what is not |

The later contest version can compress this sequence. Do not remove the user
experience, stop-hook block, trusted CI, or claim boundary.

## Delivery sequence

The package execution fix in usabl PR 54 is the prerequisite. Do not build the
demo around an installed command that can exit without running the engine.

### Remainder of Week 2: establish the three work lanes

Core lane:

- merge the package execution fix after review
- start Core slice A for the richer overlay projection
- freeze the projection shape used by the inspector

Fixture lane:

- start Fixture slice A from the current `usabl-app` main branch
- preserve the existing modal oracle while the new shell lands

Real-app lane:

- complete the Fleet Insights authenticated session setup
- discover candidate routes
- run the first measurement without changing engine code

Week 2 exit: the installed command runs the engine, Core slice A and Fixture
slice A are ready for review, and Fleet Insights has an honest first coverage
report.

### Week 3: build detection breadth and the inspector

Core lane:

- merge Core slice A
- open and merge Core slice B for the inspector

Fixture lane:

- merge Fixture slice A
- land interaction scenarios in Fixture slice B
- land semantic scenarios in Fixture slice C

Real-app lane:

- review Fleet Insights findings with an accessibility reviewer
- turn confirmed engine defects into separate issues and tests

Week 3 exit: the overlay makes several findings understandable, the repaired and
clean fixture states have reviewed oracles, and real-app gaps are known.

### Week 4: integrate and rehearse

- land Fixture slice D
- create the real draft demo pull request
- rehearse the browser, Claude, and GitHub sequence
- capture matching backup evidence
- complete the Fleet Insights measurement record
- run the demo release gate

Week 4 exit: two people can deliver and recover the complete team demo.

Work may run in parallel across repositories. Pull requests within one repository
remain unstacked and merge in dependency order.

## Pull request map

Do not stack pull requests within a repository. Each branch starts from current
`main` after the prior dependency is squash-merged.

### Core slice A: expose the complete overlay projection

Story: the overlay can show the complete gate-owned result without recalculating it.

Scope:

- richer read-only `OverlayProjection`
- scrubbing and untrusted-text framing
- contract and integration tests
- no visual redesign

Exit:

- one canned `Result` projects every required field
- projection remains advisory
- no new verdict logic outside the gate

### Core slice B: replace the badge with the accessibility inspector

Story: a developer can understand and act on several accessibility findings without
leaving the running application.

Scope:

- docked and collapsed layouts
- all inspector states
- keyboard and focus behavior
- responsive zoom behavior
- host isolation
- browser client tests

Exit:

- inspector is usable on a clean fixture with zero axe violations
- many-finding and long-text states remain usable
- webdriver scans still exclude the overlay

### Fixture slice A: build the realistic demo workflow

Story: the fixture presents a credible operations task and accessible demo controls.

Scope:

- app shell and deployment workflow
- scenario registry
- focused and all-scenario previews
- repaired preview
- clean control route
- app-specific `PRODUCT.md`

Exit:

- every scenario is reachable by keyboard
- demo controls remain accessible in every broken preview
- no planted barrier appears outside its declared scenario

### Fixture slice B: add interaction barriers and repairs

Story: usabl catches dialog and menu behavior that a static scan cannot prove.

Scope:

- dialog focus lifecycle
- menu expanded state and focus
- component tests
- broken and repaired integration evidence

Exit:

- expected interaction rules fire in broken state
- repaired state yields no interaction finding

### Fixture slice C: add semantic barriers and repairs

Story: one workflow demonstrates names, relationships, announcements, labels, and
contrast through supported providers.

Scope:

- row action names
- table headers
- toast live region
- toolbar labels
- form label
- contrast
- unit and integration tests

Exit:

- reviewed exact finding oracle for `all broken`
- repaired preview and clean control have zero active findings

### Fixture slice D: make the all-surface demo repeatable

Story: two operators can run and recover the browser, Claude, and GitHub sequence.

Scope:

- disposable-worktree setup
- reviewed broken feature seed
- Claude prompt
- mid-session and stop-hook rehearsal
- PR and CI runbook
- backup evidence manifest

Exit:

- two people complete the sequence from a clean clone
- local surfaces agree on the same source tree
- draft PR demonstrates regression and verified states
- no step relies on a query-only proof

### Fleet Insights measurement pass

Story: usabl produces useful evidence and honest coverage gaps on a real
authenticated application.

This is an operator run, not a feature pull request.

If it exposes a product defect:

1. File one focused issue.
2. Reproduce it outside the authenticated application when possible.
3. Add a failing test.
4. Open a separate `usabl` pull request.
5. Rerun the Fleet Insights measurement after merge.

Record the redacted measurement outcome in `usabl-plans` through a separate evidence
pull request.

## Quality gates

### Every usabl pull request

- failing test observed before implementation
- `npm run check`
- `pre-commit run --all-files`
- production dependency audit
- gitleaks history scan
- Git diff whitespace check
- independent engineering review
- independent security review against `main`

### Every usabl-app pull request

- failing component or integration test observed before implementation
- `npm test`
- `npm run typecheck`
- `npm run lint`
- `npm run build`
- real-browser scenario check
- clean control oracle
- overlay exclusion check
- pre-commit and secret scan when configured
- independent engineering and security review

### Demo release gate

- all supported scenario findings reviewed by an accessibility reviewer
- overlay has no accessibility violations on a clean host
- Claude self-check shown live
- stop hook blocks live
- repaired stop succeeds live
- PR comment and CI agree with local result
- Fleet Insights measurement completed or explicitly marked unavailable
- backup artifacts match the rehearsed commits
- two operators can deliver and recover the demo

## Evidence ledger

For each rehearsal, record:

- date and operator names
- `usabl` commit
- `usabl-app` commit
- demo source-tree hash before and after repair
- policy hash
- runner version
- browser and scanner versions
- expected and observed finding identities
- overlay result
- self-check result
- stop-hook result
- PR URL and CI run
- Fleet Insights measurement report
- timing and recovery notes

Do not commit credentials, session state, private page content, or unredacted
screenshots from Fleet Insights.

## Final claim

The completed team demo may claim:

> usabl detected real machine-checkable accessibility barriers in changed interface
> code, kept the same result visible across development surfaces, blocked incomplete
> Claude work, verified the repaired source, enforced trusted policy in CI, and
> produced explicit measurement evidence on Fleet Insights.

It may not claim:

- complete accessibility
- legal compliance
- verified Fleet Insights coverage before that path is promoted from measurement
- that a query preview is a source repair
- that model judgment or screenshots minted a receipt
