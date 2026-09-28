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
