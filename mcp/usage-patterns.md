---
title: Brain MCP Usage Patterns
description: Practical patterns for an agent reading and writing a Use Brian brain over MCP.
tags: [mcp, patterns]
---

Practical guidance for using the [Brain MCP server](brain-mcp.md) well. Every rule below follows from that page's tool set and semantics.

## Search before you write

The brain deduplicates, but a blind write still creates noise. Before saving a memory, task, or CRM row, run `searchBrain` for the same fact. If a matching row exists, patch it (`updateTask`, `updateContact`, `updateDeal`) instead of creating a duplicate.

## Resolve entities before linking

To connect a new row to an existing entity, you need that entity's UUID. Call `getEntity` by id or display name first: it returns the entity's UUID and its existing edges. Reading the current edges lets you skip links that already exist. Then pass the resolved UUID in the write tool's `links` field.

## Pick the right capture tool

| You have | Use |
|---|---|
| Raw notes, a document, a mixed dump | `ingestToBrain` with `decompose: true` (default) |
| A stream of message-shaped work activity | `ingestToBrain` with `captureMode: "routed"`, one stable `eventId` per message |
| One already-distilled atomic fact | `saveMemory` |
| Task-shaped content (a commitment, follow-up) | `saveTask` to create the task now; `ingestToBrain` only files it as a suggestion |

The two task paths are not interchangeable. `saveTask` creates the row. Tasks that `ingestToBrain` extracts from your content are held for human review as suggestions and do not exist as tasks until someone accepts them, so do not treat a successful `ingestToBrain` call as proof a task was created. See [Tasks](../concepts/tasks.md).

`ingestToBrain` with `decompose: true` runs the full extraction pipeline and builds an entity + edge graph. `saveMemory` and `ingestToBrain` with `decompose: false` skip extraction: they store text flat. Never hand task-shaped content to `saveMemory` (unstructured, cannot be filtered, assigned, or closed).

For immediate mode, call `ingestToBrain` once per coherent unit (one project, document, or topic) with a `sourceLabel`. Extraction quality drops on one mixed blob.

For routed mode, call it once per source message. The selected assistant capture profile applies ordered rules and partitions scheduled messages into durable pools. A queued result costs no per-message extraction call; Brian runs one extraction when the time or size window flushes. Keep `eventId` stable across retries, and provide the partition field the profile requires (`sessionId` or `subjectId`). The MCP connection does not automatically observe the host conversation.

## Narrow searches with scope

`searchBrain` accepts a `scope` as a single value or an array. Passing the scopes you care about (for example `['task', 'deal']`) narrows results and avoids spending the shared result budget on primitives you do not need. Omitting `scope` fans out across everything.

## Respect read-only credentials

A `read`-scoped credential does not expose the write tools. Calling one fails. Detect this at `tools/list`: if the write tools are absent, do not attempt a write. Surface the limitation to the user rather than retrying.

## Cost

Every brain operation over MCP bills at the memory-op rate: `0.1` credits per operation, drawn from the workspace credit pool. No full chat loop runs unless you ask an assistant a question. See [Pricing and credits](../operations/pricing-and-credits.md).

## Create or edit a workflow in two calls

Workflows are written through a proposal, never directly. An agent-scoped credential with the configure grant sees `proposeWorkflow`, `createWorkflow`, and `updateWorkflow`.

1. Call `proposeWorkflow` with the name, definition, and trigger (plus `workflowId` for an edit). It validates everything and returns a `proposalReceipt` string. Nothing is written yet.
2. Show the proposal to your user and get an explicit yes.
3. Call `createWorkflow` (or `updateWorkflow` for an edit) with `{ "proposalReceipt": "<the exact string>" }`.

An MCP call carries no session history, so the receipt must be passed back: calling with `{}` fails with "A validated proposalReceipt is required". The receipt is signed by the server. Pass it byte for byte; an edited, truncated, or re-encoded receipt is rejected and you must propose again. Never resend workflow fields to the write call.

## Notes for agents

- A duplicate write is cheap to make and expensive to clean up: the search-first pass costs `0.1` credits and prevents graph rot.
- `getEntity` is both a read and a dedupe guard: use it before edge writes, not only when a user asks about an entity.
- Batching related facts into one `ingestToBrain` call yields a cleaner graph than many `saveMemory` calls.

## Related

- [Brain MCP server](brain-mcp.md)
- [Pricing and credits](../operations/pricing-and-credits.md)

## Browser profile department changes

In an attended assistant chat, `classifyBrowserProfileDepartment` accepts `profileId`, `expectedDepartmentId` (explicit null when unassigned), `departmentId`, and `reason`. It requires a fresh one-time human approval; unattended and direct programmatic invocation cannot substitute a model-supplied confirmation. Only the identity owner may act, and transferring/removing an existing department additionally requires workspace owner/admin authority. Both human and assistant need current access within the turn's department, clearance and credential limits. The profile must be enabled for the assistant. Successful changes preserve profile sharing and clearance, revoke standing automatic approvals and write an audit record. A stale reviewed department fails without changing the profile. Read admitted profile/organization metadata again before requesting another approval; do not guess hidden IDs or retry a denied change blindly. The equivalent human control is Browsers > Browser profiles > Change department.


Browser operations retain the originating turn's live authority and withhold results if it changes during an operation. New cloud tasks from validated agent executions also persist the original assistant's department ceiling and renew it after a process restart; a human owner's broader access does not replace that ceiling. Internal cleanup kills a revoked task without saving its browser session or publishing downloads; owner-scoped human and attended agent discard are available as described below. An already dispatched remote action may have executed, so check its outcome before retrying. Tasks from eligible locked, owner-only web sessions now retain the original session identity, context, audience and read classification; deletion, rebinding, a changed lock or read classification, a held binding, or denied current source read stops access. Sources that still depend on live credential, audience or workflow callbacks require the same original invocation and cannot be resumed independently after restart. Durable support and recovery for those sources, legacy task recovery and transactional child effects remain under implementation; these checks are not a claim of complete browser departmental isolation.

Local browser tasks now keep the original live authority for the lifetime of their process-local tab binding. Human takeover and protected-fill admission renew it too; a stronger current human grant or a new assistant turn cannot replace the original source. Activity updates preserve the first execution/source evidence, and source revocation withholds successful relay results. This supports live local handoffs without treating callback-only sources as independently resumable after restart.

Removing the original assistant from a browser profile now also denies its retained validated cloud and local tasks, including later human takeover. Current profile eligibility and the original classification are checked before effects and after successful provider responses; results are withheld if access changes in flight. Re-enabling another assistant or relying on the human owner's broader access does not transfer the retained task.

Local relay commands now retain a host-owned task binding. Once another task replaces it, the old task cannot read, control, stop, or navigate to reclaim that browser connection. The relay serializes bound commands and closes a connection after an uncertain timeout. A task-scoped Stop is not deferred onto a future pairing. Human and agent discard use these checks; installed-browser acceptance remains unfinished.

Bound local commands require a current relay supporting the task-command endpoint. An older relay refuses before dispatch; update the relay rather than retrying through an unbound endpoint.

Safety Stop returns only a fixed stopped acknowledgement, including after source access is revoked. It cannot disclose page data or bypass ordinary read controls. Failed or timed-out Stop is unconfirmed. The acknowledgement does not confer browser read access.

Use `browserDiscardTask` in an attended chat to stop and discard an owned task without saving browser sessions or downloads. Optional `sessionId` defaults to the current chat. The command always requires confirmation and current workspace membership, even when the original profile/source is no longer readable. It cannot stop another user's task or a replacement task, undo website actions already sent, or bypass tool policy. The equivalent human Stop and discard control is available in the browser live view, including its unavailable-task state. An unconfirmed result means inspect the browser before retrying; do not claim that teardown succeeded.

Browser and compute calls now retain cumulative task input protection. A narrower later turn cannot erase earlier source labels; conflicting versions of an input require a new task instead of silently substituting the latest source. This internal evidence retention does not yet complete source-aware download publication; do not treat workspace-file downloads as certified department-preserving until that adapter and its database acceptance are complete.
