---
title: Tasks
description: The brain's universal verb; a workspace-scoped, schema-frozen primitive for every commitment the assistant tracks.
tags: [concepts, tasks]
canonical: https://sidan.ai/docs/tasks
---

> Human-readable version: https://sidan.ai/docs/tasks

Tasks are the universal verb of the brain: every commitment, follow-up, and unit of work the assistant should keep track of. They live in the same database as memories and CRM rows, so the assistant reads, writes, and reasons over them without crossing a service boundary.

## Shape of a task

A v1 task is intentionally narrow: `title`, `status`, optional `assignee`, optional due date, `tags`, an optional `parent` for sub-tasks, and a free-form `external_ref` for synced rows. There are no typed priority / description / estimate columns; sprint estimation and ordering go into a single `attributes` JSONB bag. Tasks are workspace-scoped and die with their workspace.

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

## Tasks vs scheduled tasks

Workspace Tasks are durable forward-commitments visible to every workspace member and to the assistant. Scheduled tasks (see Tools & connectors) are cron-style jobs that fire on a timer to run an assistant turn. Different primitives; the assistant can use one to remember to schedule the other.

## Notes for agents

- To remove a task, set `status='archived'` (or call `closeTask`); there is no delete. Archived tasks are hidden from `listTasks` unless you ask for them explicitly.
- Assign a task by resolving the teammate through `listWorkspaceMembers` first; `assignee_id` references a workspace membership, not a global user id, and clears if that member leaves.
- Any non-core field (priority, estimate, ordering) belongs in the `attributes` JSONB bag; do not expect dedicated columns for them.
- A "remind me / do this on a schedule" request is a scheduled task (a timer), not a workspace Task (a tracked commitment). Pick the primitive that matches.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Tools & connectors](./tools-and-connectors.md)
- [CRM](./crm.md)
