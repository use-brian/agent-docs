---
title: Workflows
description: Workspace-scoped DAGs that fuse the brain with action via assistant_call, tool_call, wait, and branch steps.
tags: [concepts, workflows]
canonical: https://usebrian.ai/docs/workflows
---

> Human-readable version: https://usebrian.ai/docs/workflows

A workflow is a workspace-scoped DAG that fuses the brain with action. Author it in chat ("when a Threads draft is approved, post it, wait 24 hours, then ask the brand specialist to summarize engagement") or in the web builder (the Workflow tab in the app). The runtime knows about assistants, memory, and tools, so an `assistant_call` step inherits the workspace's brain at the moment it runs.

## Feature walkthroughs

These are illustrative, step-by-step examples. They are not claims that a run, send, or publication has occurred.

### Build a workflow

Turn a repeatable process into an explicit sequence of steps.

1. Open the workflow builder or describe the process to Brian. Start with a manual trigger so you can inspect it before scheduling.
2. Choose the assistant or tool for each step and connect the outputs that later steps need. Add a wait or branch only where the process needs one.
3. Review the saved workflow and run a small example. Inspect each step's status and output.

**What to check:** The run shows what actually happened at each step. A saved workflow is a definition, not evidence that it has already run.

### Choose a trigger

Start the same workflow manually, on a schedule or from an event.

1. Open the workflow's trigger settings and choose the trigger that fits the process.
2. For a schedule, check its timezone and timing. For a webhook or event, select the intended source and any matching conditions.
3. Save the trigger, then inspect a real matching run. Check the run history before assuming a source event triggered work.

**What to check:** The trigger starts the workflow under its configured conditions. A scheduled run and a manual run use the same workflow steps.

### Review approvals and runs

Understand a pause, approve the intended action and inspect the outcome.

1. Open the run and locate the step waiting for approval or reporting a failure.
2. Read the exact action and inputs. Approve only if correct, or reject and revise the workflow; grant continuing permission only when you intend future matching runs.
3. Return to the run and check the resumed step's output and any delivery receipt. Verify earlier effects before retrying a failed run.

**What to check:** Approval authorizes the named action; it does not guarantee success. Runs may consume credits and repeated runs can repeat external effects.

## Everyday workflow

1. **Choose what starts the work.** Define the workflow in chat or the builder, then choose a manual, scheduled, webhook, or event trigger.
2. **Connect the steps.** Combine assistant calls, tool calls, waits, and branches in the order the work should happen.
3. **Review a run.** Check step outputs and approval requests. Grant ongoing permission only for actions you want the workflow to repeat.

## Step types

Workflows are built from four step types. Sequential by default; `branch` steps route down one arm; `wait` steps pause the run and resume on a scheduler tick.

| Step | What it does |
|---|---|
| `assistant_call` | Invoke an assistant with a prompt. The callee has the workspace's tools, memory, and knowledge available. Optional fields restrict its tools, push the output to a channel, or carry a persistent session across runs. |
| `tool_call` | Invoke a first-party or MCP connector tool directly. Allow-policy tools run unattended; ask-policy tools pause the run on the unified approvals queue; block-policy errors at dispatch. |
| `wait` | Pause until an absolute timestamp or for a relative duration. The runtime stores a `scheduled_jobs` row and the poll worker resumes the run at the deadline. |
| `branch` | Evaluate a JSONLogic condition against the previous step's output and the run-scope vars bag. Routes to one of two `nextStepIds`. |

## Parallel fan-out

A step's `nextStepId` accepts an array of step ids (distinct, max 5). When the step completes, every listed step starts in parallel. The definition's `startStepId` accepts the same scalar-or-array shape: an array is a trigger-level fan-out, starting every listed entry step in parallel the moment the trigger fires, under the same rules below. A downstream step that several branches point at is the implicit join: it runs once, after every branch that can still reach it has settled, and can read each branch's `storeOutputAs` var via `{{vars.<name>}}` (give parallel siblings distinct names; same-name writes are last-settled-wins).

Rules an authoring agent must respect:

- The step graph must be acyclic. A cycle is rejected at authoring time; there is no loop step (iterate across runs with a recurring trigger + `{{lastRun.*}}`).
- The first branch failure fails the run; in-flight siblings settle and record their step runs honestly, and the join never executes.
- A pause must hold the only live cursor: never place a `wait` step, or an ask-policy `tool_call`, on a parallel branch that some sibling branch never rejoins. Authoring rejects the statically-detectable cases; anything that slips through fails at run time with `pause_in_parallel`. A `wait` or ask-policy step at or after the join is fine.
- At most 5 steps execute concurrently per run.

## Per-step timeout

An `assistant_call` step's wall-clock limit is `depth.timeoutMs` (milliseconds): default 90000, deep research 300000, clamped to 1000-900000. Values above 300000 must be set explicitly per step. When the limit is hit the run terminates with status `timeout` (distinct from `failed`) and the step's partial output is preserved on its step run.

## Triggers

Four ways to fire a workflow. Every trigger feeds the same runtime.

| Trigger | How it fires |
|---|---|
| Manual | `POST /api/workflows/:id/run` from your service, or click "Run now" in the web builder. |
| Schedule | Cron expression in your workspace timezone. Runs on the scheduled-jobs poll worker. |
| Webhook | HMAC-signed `POST /api/workflow-webhooks/:slug` from any external service. Per-row slug + secret. |
| Event | A subscribed connector instance (GitHub, Fathom, Calendar) or channel integration (Slack, Telegram, Feishu/Lark). Fires whenever its match filter passes. |

## Client-isolated email drafts

An IMAP event workflow may bind its whole definition to one external API-key
client principal. `resolve.kind: verified_email_pairing` lowercases the inbound
sender and resolves it only through server-owned identity pairings previously
written by identified turns on the exact configured active external chat key
and assistant. Zero matches or more than one distinct external identity fails
before any assistant registry, context, model call, CRM read, or draft. It
never searches workspace contacts, deals, memories, or mail archives, so other
CRM entries cannot enter the resolution surface.

The principal-bound assistant lane is read-only and draft-only. A workflow may
place one structurally reviewed `imapSendMessage` reply after the draft, but it
must target the triggering sender and message, freeze its envelope, and require
explicit human approval. Sender routing is not authentication, and this lane
never auto-sends.

## Cost

A workflow run is billed as the sum of its `assistant_call` steps at whatever tier each step uses. `tool_call`, `wait`, and `branch` steps cost zero credits. Before you run a workflow, the builder shows the exact message count ("Running 'Competitor Analysis': 3 Pro messages per run"); branches are shown as a range.

## REST surface

The web builder is one client of the REST API. Other clients can use the same surface. Auth is the standard workspace-member check.

```
GET  /api/workflows?workspaceId=
GET  /api/workflows/:id
POST /api/workflows
PATCH /api/workflows/:id
DELETE /api/workflows/:id
POST /api/workflows/:id/run
GET  /api/workflows/:id/runs
POST /api/workflow-webhooks/:slug
```

## Approvals

Workflows pause on the same unified approvals queue every other surface uses (chat, app-kind staged writes, distribution drafts). One `pending_approvals` table, one resolve endpoint, four canonical kinds. Resolution is cross-channel: start a run from web, approve from Telegram.

| Kind | When it fires |
|---|---|
| `tool_invocation` | An ask-policy tool reached during a chat turn pauses the loop until the user approves. Per-tool default expiry (5 min on TG / Slack / web); fires immediately on rejection. |
| `workflow_step` | An ask-policy tool reached inside a workflow run inserts a pending row and the run pauses. Resolution resumes via the same resume protocol the chat path uses. |
| `staged_write` | Operational-primitive writes proposed by a `kind='app'` assistant (CRM / Tasks / KB / Files / Entities) queue for founder review with the proposed diff embedded in the payload. |
| `distribution_draft` | An app-kind reply draft awaiting publish on Threads or X. Lives in the feed UI; same row shape, same resolve endpoint. |

## Auto-approve via permission grants

A workflow definition can carry `permission_grants: { action_kind, grant: 'allow' | 'ask' | 'block' }` entries that auto-approve listed actions inside an active run while leaving the assistant's per-tool defaults untouched everywhere else. Two ways to author them: tick "always allow within this workflow" on the first run's approval prompts (the ticks land back on the workflow definition), or edit the grants table directly. The audit trail names the `workflow_run_id` and the grant entry that authorized each auto-approved action; revoking a grant takes effect on subsequent runs, not in-flight ones.

## Notes for agents

- Only `assistant_call` steps cost credits; `tool_call`, `wait`, and `branch` are free. Estimate run cost by counting assistant_call steps and their tiers.
- An ask-policy `tool_call` inside a run pauses the whole run on the approvals queue until resolved; a block-policy tool errors at dispatch. Prefer allow-policy tools (or a `permission_grants` entry) for unattended runs.
- Approvals resolve cross-channel and against one `pending_approvals` table, so a run started from web can be approved from Telegram.
- Revoking a permission grant applies to future runs only; it does not retroactively pause an in-flight run.
- Hosted operator assistants may discover `listOperatorWorkflowEventBindings`
  plus configure/disable/test binding tools. They appear only when the bound
  assistant holds the operator-only `operator_automation` grant; mutation also
  requires the normal `configure` gate and stages human approval. Build the
  webhook workflow first, then bind by workflow id. Never ask for or copy its
  webhook slug or HMAC secret: the server provisions and resolves those
  credentials internally.

## Related

- [Tools & connectors](./tools-and-connectors.md)
- [Doc](./doc-pages.md)
- [Brain (entities & episodes)](./brain.md)


## Departmental run authority (implementation in progress)

Chat-authored workflows, scheduled work and goals retain the authoring turn's department limits, including its department context, credential binding and clearance cap. Later grants do not expand that saved consent. If the original access expires or is revoked, start a new reviewed request under current permissions; do not retry by supplying broader authority fields.

Member workflow runs capture an internal starting access ceiling before executing. It survives waits and approvals; later permission expansion cannot widen that run, and permission loss blocks resume. A resumed legacy run without a captured ceiling must be reviewed and started as a new run. Do not supply or edit this authority through workflow input or run variables. Assistant calls inherit the pinned ceiling. External-client workflows retain their API-key principal.

Workflows and confirmed goals also persist the attended author's internal ceiling before they can run in the background. Chat proposals freeze it before approval; authenticated builder, scheduling, configuration and goal-confirmation actions capture the same boundary. A schedule row, target assistant, credential owner, approver or billed account cannot replace that author. Existing workflows/goals that predate this envelope must be explicitly reviewed and re-saved or confirmed before unattended execution. Goal runs satisfy both the goal and workflow envelopes. These fields are server-owned and are never accepted from workflow input, run variables or API request bodies. History and delivery-audience checks remain unfinished, so this is not a claim of complete strict departmental isolation.

Workflow channel delivery carries the run's accumulated sensitivity, Team and
Project evidence. The target is checked before a known delivery-bound assistant
call starts and again immediately before persistence or push. An exact current
personal-channel session can prove one member recipient. Restricted output to a
group or otherwise unverifiable external conversation requires an owner/admin-
approved audience binding for that exact conversation on the selected channel
integration. A missing, expired or too-narrow binding records
`delivery_audience_unverified` and sends nothing. Bot credentials, the workflow
author and the billed account do not prove recipient access.

Deterministic steps, approved tool invocations and delivery now recheck live member authority. If access changes during an operation, its result is withheld and the operation may already have executed: inspect its outcome before starting another run. Delivery also checks current recipient authority; these checks do not make external side effects transactionally atomic.

An old approval card cannot restart a failed or completed run, even if access is restored. Approval notifications also renew current authority before sending.

## CRM wake-ups for goals

A waiting goal can resume from a CRM event only when its recorded creator can read
that event's saved and current source under the selected Team/Project. The server
re-matches the stored event and atomically claims the current park generation.
Duplicate or stale dispatch cannot consume a later wait. The event's restrictions
continue to apply to the goal and its workflow history after resumption, including
outcome-copy reads and execution lease renewal.

Source permission loss blocks work without retries, host writeback or outcome
delivery. An authorized viewer can use the goal's Review department access action
to open workspace permission settings; this action neither grants access nor
resumes work. Source receipts remain protective after source or goal deletion.
Other native event families and complete unattended execution ceilings/consent
remain under implementation; these changes do not activate strict departments.

A department- or project-scoped goal can reuse a company-wide workflow. Execution intersects both scopes, and new writes retain their required department/project labels. Changing the goal context invalidates an existing run; editing workflow input cannot remove its saved goal binding. Direct application-role source reads also enforce the executing clearance, department and project ceilings. These safeguards do not enable strict departmental activation on their own.

A workflow authority failure also blocks its owning goal. Failed carried runs are not silently replaced, and permission changes do not trigger automatic goal retries. Review the prior operation outcome before starting fresh work.

Member workflow assistant calls now carry the trusted source evidence accumulated by the caller. The receiver checks that these sources still match and fit its access before using the question, and rechecks them during execution. Derived memory creates and updates preserve those source links. Missing, changed or inaccessible source context stops the request; do not retry it automatically. This does not yet certify event payloads or other inputs whose producers have not supplied canonical evidence.

### CRM input source protection

Member workflows resolve CRM input scope from saved event, copied-run and goal
receipts. The original event audience and the current CRM record both constrain
derived notes. Editing, holding or retiring a source can block execution or
withhold an in-flight result; stale persisted evidence requires review and a new
run rather than retrying effects. Legacy or aggregate events without complete
source evidence are refused at execution. Input JSON cannot grant access or
replace these bindings. This coverage does not certify all event families or
complete departmental strict-mode activation.

For an authorized failed run, the run-detail page offers **Review department access**. This opens workspace access settings; check the completed steps before starting a new run, since an interrupted operation may already have executed.

For approvals tied to a chat session, workspace membership or assignment as approver does not grant access to that session's private or departmental content. The queue and badge omit approvals whose source session you cannot currently read. Preview, revision and response lookups return not found after access is revoked or the required source is deleted; do not retry a stale approval automatically. This protection covers session-backed approvals; other source families still require their own authority checks.

Workflow-step approvals also inherit the current read restrictions of their referenced run and step. An inaccessible or mismatched parent removes the card and badge count. Web and channel decision dispatch rechecks that visibility and the assigned approver before claiming the decision; an unavailable result leaves the row unsettled and must not be retried as a fresh operation automatically. This closes the approval bypass of existing parent policies, not a complete certification of v2 workflow history or all event-source rules.

HTTP decision endpoints return 404 if approval authority is lost during dispatch. Channel replies direct you to review current status and access on the approvals page; supplying a fuller ID cannot restore lost access.

For CRM-sourced workflow history in v2 workspaces, the saved event audience and current source both use current department membership and per-department clearance. Public General clearance does not cap a Confidential department edge, and owner/admin roles do not replace an edge. Removing or expiring an edge can hide an old run even after its live CRM source was released to General. Explicit workflow execution limits and credential restrictions still apply. This repair does not certify history for other event-source families or migrate every legacy execution envelope.

### Workflow run department context floor

All user-scoped workflow history, including non-CRM runs, steps and copy edges, must satisfy the run's captured department context and that of every recorded source-run ancestor. The current workflow definition's selected department may restrict access further. These operational rows have no independent sensitivity column: this additional gate checks department membership at the public tier; existing CRM/source sensitivity and execution ceilings remain authoritative and compose with it. General operational metadata retains workspace-member visibility, and Projects remain a retrieval lens.

Migration 679 captures run context from its canonical workflow on insert, rejects mismatched supplied bindings, and prevents changing the captured department array or moving a run to another workflow/workspace. Clearing a deleted department foreign key does not clear the saved array. Existing inconsistent group/array evidence fails closed; the migration does not fabricate historical ownership. The v2 read policy uses current human/assistant grants, credential bindings and expiry, and preserves the legacy workspace path. This is run-history protection; workflow-definition editing, non-CRM source provenance and execution-side admission remain separately required.

### Workflow outcome-copy department admission

Migration 680 shares the run-history department predicate with trusted outcome-copy admission. A copy insert checks both the consumer and source ancestry as the consumer's recorded actor, constrained by any active assistant/binding context. Missing department access refuses the edge, including a duplicate insert attempt after revocation. The actor-parameter helper is private; the public read wrapper remains bound to transaction identity. The canonical outcome reader checks the same gate before returning content or enrichment, even when the edge already exists, and returns no auxiliary outcome when department admission is denied. It renews this admission after enrichment, immediately before commit; a late denial rolls back the new lineage edge. It keeps newest-terminal ordering and does not fall back to an older accessible outcome. Legacy execution-envelope conversion and independent blueprint/source classification remain separate requirements.

### Outcome enrichment source admission

The canonical outcome reader admits the latest blueprint enrichment through its own canonical review-source envelope, current member/assistant scope, sensitivity, department bindings and held state. The record is locked while its fields are read. An inaccessible latest enrichment withholds the entire auxiliary result and rolls back the new copy edge; it never substitutes an older record or exposes an unreviewed partial handoff. Immediately before commit, renewal includes both runs' CRM/goal source authority and the selected blueprint envelope in addition to the run department contexts. This closes the existing enrichment bypass; durable blueprint provenance in downstream execution evidence and legacy execution-envelope conversion remain tracked separately.

### Durable blueprint copy evidence

A workflow copy receipt saves the exact canonical blueprint source envelope selected at copy time. The database captures it, never a caller-supplied envelope. JSON null proves no enrichment was selected; SQL null marks historical receipts with unknown enrichment and fails closed until an explicit recovery path exists. Receipts remain immutable. Reusing a receipt requires the currently selected enrichment to match the captured envelope; a newer or changed record cannot silently replace the consumed source.

Workflow input evidence traverses copy ancestry and includes each captured blueprint source. Missing, held, deleted or changed sources refuse execution. Canonical derived writes retain that source's sensitivity, departments, projects and visibility. Blueprint edits/deletion hold existing derived descendants and reject stale writes. This applies to human-triggered and agent-triggered executions through the same canonical reader and evidence resolver. It does not claim complete workflow definition mutation or legacy recovery coverage.

The executor records auxiliary outcome lineage before resolving its execution scope. This ensures the first step receives the copied blueprint evidence and does not discover a new causal source only during authority renewal. Run-history reads also enforce captured enrichment authority through copy ancestry.

A persisted `workflow_cancelled` failure terminates the run's execution authority. New operations and fresh resume attempts are refused; a result completing across cancellation is withheld because an already-started effect may have occurred. Inspect the existing outcome before retrying. Ordinary step-failure reporting keeps its existing behavior.

Workflow-origin browser tasks retain the originating run's actor, context and authority/source fingerprints. Cold continuation revalidates that exact run through canonical workflow authority; changed evidence, cancellation or lost source access requires inspection and a fresh start. A replacement invocation or broader owner grant cannot supply the old task's authority. Workflow downloads also retain the canonical workflow source and every required parent receipt, with authority renewed in the file transaction. Unattended credential issuance remains separate.

A legacy browser task with an acting-assistant ceiling but no saved source evidence cannot resume, even from a new authorized invocation. Use the task-bound discard path and start a new task with current source evidence; the old task is not automatically reclassified.

Live workflow execution renews current workflow/run department visibility as well as CRM source visibility. Explicit execution Project and assistant-visibility limits also constrain saved and current source evidence under permission v2; department access does not bypass those limits.

Copied prior outcomes retain their accumulated source protection, including private sources and explicit Project and assistant restrictions. Changed or missing retained evidence prevents reuse. A fresh run can omit an unverifiable historical prior outcome instead of treating it as unprotected context.

Prior-outcome copying validates accumulated evidence against the current member and ambient execution ceiling before returning content. A denied copy returns no auxiliary outcome and does not add a receipt. Authorized content edits remain usable under current-envelope audience checks; a reclassification outside the receiver's access is withheld.

Workflow history filters captured accumulated evidence before returning run rows, steps or counts. Copied-run history retains the saved evidence/version requirement. This includes independent private, Project and assistant limits and current source reclassification checks; authorized per-department tiers still apply. History without an explicit valid accumulated envelope is withheld, with the original run data retained. Return to the independently accessible workflow and check earlier actions before explicitly starting a fresh run; the old run is neither relabeled nor replayed.

Persisted cancellation terminates workflow execution authority even when the run's history is unavailable under current content permissions. Hidden history is never evidence that cancellation did not occur.

A manual-run HTTP request may finish execution but lose permission to return its result. The endpoint then returns HTTP 409 with `error: run_result_unavailable` and `operationMayHaveExecuted: true`, without result or step content. Check current workflow history and any completed effects before deciding whether to run again; do not automatically retry this response.

The browser source binds to the initial accumulated evidence saved before the first step executes, including causal input and context write floors. Normal initialization does not invalidate the new task. Cold continuation still requires the exact explicit saved envelope; missing or stripped evidence requires recovery rather than reconstruction from current permissions.

A browser task started in a later step binds the accumulated evidence successfully saved for that step, including protected reads from earlier steps. Previously retained task descriptors are not updated to match later evidence; their exact-evidence renewal requirement remains in effect.

Canonical workflow-derived file admission retains independent saved source restrictions and requires exact upstream dependency receipts. Changes to a causal source or cancellation of an upstream copied run invalidate dependent files. Workflow-origin browser downloads use this complete dependency admission and a deferred grant-expiry boundary; failed authority renewal prevents publication.

Workflow enrichment can participate as an exact upstream dependency of a derived file. Enrichment changes invalidate dependent files. Reading enrichment uses the admitted department's tier even when General clearance is lower, while explicit execution Project limits and current membership still apply.


Task-triggered workflows inherit the task event's saved visibility, sensitivity, departments and Projects, including the previous task version when an update contains previous values. Dispatch requires the saved workflow author's current bounded access; execution also checks the executing assistant. Later relabeling or deletion does not erase the original event restrictions. A missing historical source receipt requires a new task lifecycle event, not rewritten event labels.

Task-source access also gates automatic storm pauses. An event the saved workflow author cannot read cannot pause that workflow; authorized storm protection still applies.

Knowledge lifecycle events include the canonical source version. Saved restrictions continue to apply after the entry changes or is deleted. Missing historical source receipts require a new canonical event; names, paths and tags are not public merely because the event omits the body.

Primitive event titles, paths, tags and task lifecycle fields must match the write-time receipt. Unknown payload fields or substituted metadata are rejected before a run or storm pause is committed. Older runs lacking metadata verification stay unavailable; they are not certified by replaying source IDs.

Page-triggered runs require the canonical page-event receipt and matching title, action, actor and watched-page identity. The workflow author must pass both saved and current private/Teamspace/department restrictions before queue insertion or storm pause. Run input and history retain immutable original and observed protection after page changes or deletion. Missing historical receipts require a new canonical event; hand-authored page labels do not establish authority.


Page-trigger-derived files retain the original page and Teamspace access boundary in their provenance. Renaming or copying the file does not remove it. Extracted text follows the same boundary; losing membership hides the file and its text, while regaining authorized membership restores access. Missing dependency evidence prevents publication rather than producing an unprotected output. Other page-derived output families remain under implementation and must not be treated as generally supported yet.


The canonical file writer expands required provenance from the exact source snapshots it receives. File tools do not need to reconstruct hidden page dependencies. If a source page has become more restrictive, a new derived file inherits the admitted stricter boundary; later sharing or deletion does not remove that observed restriction.

## Department-context workflows

A workflow saved with a department context is visible and editable only to people
(and their assistants) who may read that department. Others, including an owner
or admin without that department, see it as not found in lists and reads and
cannot rename, disable or delete it.

A workflow triggered by a connector event (for example new mail) only runs for
events from connectors whose department audience fits the workflow's
department and that its author can currently use; a private connector only
triggers its owner's workflows. Other events are skipped without pausing the
workflow.
