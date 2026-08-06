---
title: Tasks
description: The brain's universal verb; a workspace-scoped, schema-frozen primitive for every commitment the assistant tracks.
tags: [concepts, tasks]
canonical: https://usebrian.ai/docs/tasks
---

> Human-readable version: https://usebrian.ai/docs/tasks

Tasks are the universal verb of the brain: every commitment, follow-up, and unit of work the assistant should keep track of. They live in the same database as memories and CRM rows, so the assistant reads, writes, and reasons over them without crossing a service boundary.

## Shape of a task

A v1 task is intentionally narrow: `title`, `status`, optional `assignee`, optional due date, `tags`, an optional `parent` for sub-tasks, and a free-form `external_ref` for synced rows. There are no typed priority / description / estimate columns; sprint estimation and ordering go into a single `attributes` JSONB bag. Tasks are workspace-scoped and die with their workspace.

Longer prose belongs in the dedicated `description` field on `saveTask` and `updateTask`. It is stored inside the `attributes` bag but the tools merge it for you, so pass `description` rather than writing a `description` key into `attributes` by hand: `updateTask` overwrites the whole `attributes` object, so a hand-written key is easy to clobber on the next patch.

## Status

Five states:

| Status | Meaning |
|---|---|
| `todo` | Open, not started. |
| `in_progress` | Being worked on. |
| `blocked` | Waiting on something. |
| `done` | Completed. |
| `archived` | Soft-deleted; excluded from `listTasks` by default. |

There is no `deleteTask` in v1: soft-delete via `status='archived'` covers it without confirmation prompts.

## Assignees

`assignee_id` is an FK to `workspace_members`, not `users`. When a teammate leaves the workspace, the assignee clears (`SET NULL`) but the work survives. The assistant resolves a named teammate via the `listWorkspaceMembers` tool.

## Chat tools

Six tools, on for every primary and standard assistant by default, off for `kind='app'` assistants (like the Threads distribution app). Toggle the whole group from Assistant Settings -> Capabilities -> Tasks. The same allow/ask/block enum lives on the underlying capability, so future promotion to ask-mode needs no migration.

`saveTask` · `getTask` · `listTasks` · `updateTask` · `closeTask` · `reopenTask`

## How a task gets created

Two lanes, and they behave differently on purpose.

| Lane | Trigger | Result |
|---|---|---|
| Assistant | `saveTask`, a chat request, a workflow step | The task is created. A borderline candidate is created with a warning rather than withheld, because a human asked for it. |
| Extracted | `ingestToBrain` and every other ingest source (Slack, email, GitHub) | The candidate is held as a **suggestion** for a human to accept or dismiss. No task exists yet. |

The extracted lane is suggestion-first: content the workspace ingests does not silently become work. A workspace earns automatic creation per class by adding an `allow` rule (often through the "Always create tasks like this" action on a suggestion), after which matching candidates are created immediately and the auto-approval is kept as an audit record. Suggestions expire if nobody reviews them.

For agents the practical rule is: call `saveTask` when you intend a task to exist. `ingestToBrain` is for capturing content, and any task it derives is a proposal.

## Tasks vs scheduled tasks

Workspace Tasks are durable forward-commitments visible to every workspace member and to the assistant. Scheduled tasks (see Tools & connectors) are cron-style jobs that fire on a timer to run an assistant turn. Different primitives; the assistant can use one to remember to schedule the other.

## Notes for agents

- To remove a task, set `status='archived'` (or call `closeTask`); there is no delete. Archived tasks are hidden from `listTasks` unless you ask for them explicitly.
- Assign a task by resolving the teammate through `listWorkspaceMembers` first; `assignee_id` references a workspace membership, not a global user id, and clears if that member leaves.
- Any non-core field (priority, estimate, ordering) belongs in the `attributes` JSONB bag; do not expect dedicated columns for them. `description` is the exception with a dedicated tool field: pass it directly instead of writing the key yourself.
- A successful `ingestToBrain` call does not mean a task was created. Extracted tasks wait as suggestions for a human. Use `saveTask` when the task must exist, and `listTasks` if you need to confirm.
- A "remind me / do this on a schedule" request is a scheduled task (a timer), not a workspace Task (a tracked commitment). Pick the primitive that matches.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Tools & connectors](./tools-and-connectors.md)
- [CRM](./crm.md)
