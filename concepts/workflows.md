---
title: Workflows
description: Workspace-scoped DAGs that fuse the brain with action via assistant_call, tool_call, wait, and branch steps.
tags: [concepts, workflows]
canonical: https://sidan.ai/docs/workflows
---

> Human-readable version: https://sidan.ai/docs/workflows

A workflow is a workspace-scoped DAG that fuses the brain with action. Author it in chat ("when a Threads draft is approved, post it, wait 24 hours, then ask the brand specialist to summarize engagement") or in the web builder (the Workflow tab in the app). The runtime knows about assistants, memory, and tools, so an `assistant_call` step inherits the workspace's brain at the moment it runs.

## Step types

Workflows are built from four step types. Sequential by default; `branch` steps fan out; `wait` steps pause the run and resume on a scheduler tick.

| Step | What it does |
|---|---|
| `assistant_call` | Invoke an assistant with a prompt. The callee has the workspace's tools, memory, and knowledge available. Optional fields restrict its tools, push the output to a channel, or carry a persistent session across runs. |
| `tool_call` | Invoke a first-party or MCP connector tool directly. Allow-policy tools run unattended; ask-policy tools pause the run on the unified approvals queue; block-policy errors at dispatch. |
| `wait` | Pause until an absolute timestamp or for a relative duration. The runtime stores a `scheduled_jobs` row and the poll worker resumes the run at the deadline. |
| `branch` | Evaluate a JSONLogic condition against the previous step's output and the run-scope vars bag. Routes to one of two `nextStepIds`. |

## Triggers

Four ways to fire a workflow. Every trigger feeds the same runtime.

| Trigger | How it fires |
|---|---|
| Manual | `POST /api/workflows/:id/run` from your service, or click "Run now" in the web builder. |
| Schedule | Cron expression in your workspace timezone. Runs on the scheduled-jobs poll worker. |
| Webhook | HMAC-signed `POST /api/workflow-webhooks/:slug` from any external service. Per-row slug + secret. |
| Event | A subscribed connector instance (GitHub, Fathom, Calendar) or channel integration (Slack, Telegram). Fires whenever its match filter passes. |

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

## Related

- [Tools & connectors](./tools-and-connectors.md)
- [Doc](./doc-pages.md)
- [Brain (entities & episodes)](./brain.md)
