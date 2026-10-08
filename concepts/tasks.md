---
title: Tasks
description: The brain's universal verb; a workspace-scoped, schema-frozen primitive for every commitment the assistant tracks.
tags: [concepts, tasks]
canonical: https://usebrian.ai/docs/tasks
---

> Human-readable version: https://usebrian.ai/docs/tasks

Tasks are the universal verb of the brain: every commitment, follow-up, and unit of work the assistant should keep track of. They live in the same database as memories and CRM rows, so the assistant reads, writes, and reasons over them without crossing a service boundary.

## Feature walkthroughs

These are illustrative, step-by-step examples. They are not claims that a run, send, or publication has occurred.

### Create and assign

Give a piece of work a clear outcome, owner and due date.

1. Open Tasks in your workspace, then open Brian's chat dock. Use an assistant with the Tasks capability.
2. Ask Brian to create the example below. Replace Friday with your intended date and check the resolved due date.

   > Example request: Create a task called Prepare launch brief, assigned to me, due Friday. The description should say: draft the audience, launch message and checklist. Done means the brief is ready for review.

3. Open the task to review its title, description, assignee and due date. Adjust any field before moving on.

**What to check:** One task describes what needs doing and how you will know it is finished. Assignment tracks ownership; it does not start autonomous work.

### Organize your task list

Use filters, views and Projects to find the work that matters now.

1. Open Table, then Filter. Choose an assignee and one or more statuses.
2. Open View to choose grouping and sorting. Switch to Board for a visual view of the same work.
3. Open a task and select an existing Project if it belongs to one. Clear a filter pill to widen the list again.

**What to check:** Table and Board show the same matching tasks. Values within one filter use OR; different filters use AND. A Project organizes work without granting access.

### Track progress

Move work through its status and bring completed work back when needed.

1. Open a task's status field and choose In progress when work starts.
2. Use Blocked for an obstacle or In review when the result needs checking. Add the relevant context to the description.
3. Choose Done when the outcome is complete. To revisit it, enable Show completed in View and change it back to Todo.

**What to check:** The same status appears on the row and board card. Marking a task Done records completion; it does not prove that an external action happened.

### Break work into subtasks

Keep smaller deliverables attached to the task they support.

1. Create or open the parent task. Give Brian its exact title, and clarify which record you mean if several match.
2. Ask for concrete subtasks with the parent relationship made explicit.

   > Example request: Under Prepare launch brief, create two subtasks: draft the audience summary and review the launch checklist. Link both to that parent task.

3. Ask Brian to list the parent's subtasks and check each title, owner and due date.

**What to check:** The smaller tasks link to the intended parent in the same workspace. Update their statuses as each deliverable is finished.

### Review suggestions and rules

Decide which commitments extracted from conversations should become tasks.

1. Open Suggestions in Tasks. Expand a candidate to read its source, evidence and reason for being held.
2. Choose Add it to accept. Use Add and edit for corrections, or Not a task with a reason to dismiss.
3. Use Always add similar only when that source should create matching tasks automatically. Review these rules from the Tasks settings action.

**What to check:** Accepted suggestions become tasks. A suggestion alone is not a task; Always add similar changes how future matching candidates are handled.

### Clean up the backlog

Archive finished work or reject unwanted tasks deliberately.

1. Apply a cleanup filter such as Done not archived, Unassigned or Stale over 30d.
2. Select the intended rows. Use Select all matching only after checking the filtered scope, then choose Archive.
3. For work that should never have been a task, use Delete and provide a reason if you want to teach a rule. Review the confirmation before proceeding.

**What to check:** Archive hides work from normal lists without teaching a rejection rule. Delete with a reason can affect future task creation; these actions are different.

## Everyday workflow

1. **Capture the work.** Ask Brian to create a task with a clear title, a due date, and an assignee when needed.
2. **Keep its status current.** Move work from todo to in_progress to done. Use blocked when something prevents progress.
3. **Review and follow up.** Ask for outstanding tasks or reopen unfinished work. Archive tasks you no longer need.

## Shape of a task

A v1 task is intentionally narrow: `title`, `status`, optional `assignee`, optional due date, `tags`, an optional stable Project, an optional `parent` for sub-tasks, and a free-form `external_ref` for synced rows. There are no typed priority / description / estimate columns; sprint estimation and ordering go into a single `attributes` JSONB bag. Tasks belong to one workspace, can carry Team audience requirements, and carry zero or one Project in v1.

Longer prose belongs in the dedicated `description` field on `saveTask` and `updateTask`. It is stored inside the `attributes` bag but the tools merge it for you, so pass `description` rather than writing a `description` key into `attributes` by hand: `updateTask` overwrites the whole `attributes` object, so a hand-written key is easy to clobber on the next patch.

## Status

Six states:

| Status | Meaning |
|---|---|
| `todo` | Open, not started. |
| `in_progress` | Being worked on. |
| `in_review` | Ready for review. |
| `blocked` | Waiting on something. |
| `done` | Completed. |
| `archived` | Soft-deleted; excluded from `listTasks` by default. |

Archive keeps a task out of normal lists. Reasoned rejection is separate: `rejectTask` can teach task-creation rules. Bulk updates and archiving require confirmation in chat.

## Assignees

`assignee_id` is an FK to `workspace_members`, not `users`. When a teammate leaves the workspace, the assignee clears (`SET NULL`) but the work survives. The assistant resolves a named teammate via the `listWorkspaceMembers` tool.

## Team and Project scope

Team and Project are orthogonal. Team scope controls who may discover the task;
Project scope organizes authorized work. Project participants do not gain read
access. Workspace General is an empty Project binding; a Project-scoped turn
sees General plus its exact Project.

`saveTask` and `updateTask` accept a stable `projectId`; `listTasks` can filter
by it. The association does not change the current conversation's scope. A
write also retains the active context and all scope evidence from rows it read,
so omitting `projectId` or Team metadata cannot launder restricted source
content into General.

## Chat tools

Task tools are enabled for primary and standard assistants by default, and disabled for specialist app assistants unless granted. Manage the capability from Assistant Settings → Capabilities → Tasks. Current caller and workspace restrictions still apply.

`saveTask` · `getTask` · `listTasks` · `updateTask` · `closeTask` · `reopenTask` · `bulkUpdateTasks` · `archiveTasks` · `rejectTask` · `saveTaskRule` · `listTaskRules` · `deleteTaskRule`

## How a task gets created

Two lanes, and they behave differently on purpose.

| Lane | Trigger | Result |
|---|---|---|
| Assistant | `saveTask`, a chat request, a workflow step | The task is created. A borderline candidate is created with a warning rather than withheld, because a human asked for it. |
| Extracted | `ingestToBrain` and every other ingest source (Slack, email, GitHub) | The candidate is held as a **suggestion** for a human to accept or dismiss. No task exists yet. |

The extracted lane is suggestion-first: content the workspace ingests does not silently become work. A workspace earns automatic creation per class by adding an `allow` rule (often through the "Always create tasks like this" action on a suggestion), after which matching candidates are created immediately and the auto-approval is kept as an audit record. Suggestions expire if nobody reviews them.

For agents the practical rule is: call `saveTask` when you intend a task to exist. `ingestToBrain` is for capturing content, and any task it derives is a proposal.

## Tasks vs scheduled tasks

Workspace Tasks are durable forward-commitments, visible only when the current Team, Project, sensitivity, and ordinary visibility gates all allow them. Scheduled tasks (see Tools & connectors) are cron-style jobs that fire on a timer to run an assistant turn. Different primitives; the assistant can use one to remember to schedule the other.

## Notes for agents

- `closeTask` marks work done; `archiveTasks` hides it from default lists. Use reasoned rejection only when the user intends to reject the work and teach future creation rules. These are different operations.
- Assign a task by resolving the teammate through `listWorkspaceMembers` first; `assignee_id` references a workspace membership, not a global user id, and clears if that member leaves.
- Use stable Project ids rather than `project:<name>` tags. A task has at most one Project, and assigning it never grants access.
- Any non-core field (priority, estimate, ordering) belongs in the `attributes` JSONB bag; do not expect dedicated columns for them. `description` is the exception with a dedicated tool field: pass it directly instead of writing the key yourself.
- A successful `ingestToBrain` call does not mean a task was created. Extracted tasks wait as suggestions for a human. Use `saveTask` when the task must exist, and `listTasks` if you need to confirm.
- A "remind me / do this on a schedule" request is a scheduled task (a timer), not a workspace Task (a tracked commitment). Pick the primitive that matches.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Tools & connectors](./tools-and-connectors.md)
- [CRM](./crm.md)


## Task lifecycle scope and bulk failures

Status, assignee and due-date-only task edits preserve the target task's existing
classification, visibility, Teams and Project. The bulk tools also preserve these
for their structured priority change, which merges only that task's own attributes.
Reading several Projects in one conversation does not move every edited task into
all of them. Canonical mutation-access checks still apply. Content, parent and
dependency edits retain their full evidence scope; the one-Project task limit is
not relaxed.

A bulk result with any failed row is an error, even when other rows committed.
Successes and failures are listed separately, with failed task IDs and causes.
Verify current state, address the cause, and retry only unresolved tasks when
appropriate. Do not repeat the whole batch or switch tools to bypass the failure.

Omitting a bulk status filter includes completed tasks. Keep the user's requested
status filters or resolved IDs when recovering; do not broaden the selection.


## Inspect task protection

Task visibility depends on current department access, sensitivity and any private partition. Workspace membership alone does not grant access. Projects organize tasks; no Project is not the same as General department scope.

Use `getTask` after creation to inspect the saved `sensitivity`, `compartments` and `project_ids`. An empty `compartments` array means General department scope, still subject to sensitivity and private visibility. A null protection field means the store did not report it; do not infer a classification. Creation retains inherited source protection. Use the canonical context/classification workflow for changes rather than treating a Project edit as an access change.
