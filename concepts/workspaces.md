---
title: Workspaces & sharing
description: A workspace is the unit of brain identity, billing, and membership; sharing lets assistants query other people's assistants.
tags: [concepts, workspaces]
canonical: https://usebrian.ai/docs/workspaces
---

> Human-readable version: https://usebrian.ai/docs/workspaces

Workspaces turn an assistant into a shared resource. Sharing lets your assistant call other people's assistants for data only they have.

## Workspaces

A workspace is the unit of brain identity, billing, and membership: a company brain from day 1, even when you are the only member. One workspace is auto-created when you sign up (named from the business you tell us about during onboarding). Inviting teammates later does not migrate anything; the same workspace just gains new members. A workspace owns its assistants, memories, knowledge base, connector instances, and channel installs. Memory is per (user, assistant): team-scoped facts are shared across the workspace, while personal memories stay yours.

## Workspace data reset

The owner-only `DELETE /api/workspaces/:workspaceId/data` route resets the
workspace's learned/produced content while preserving its identity, members,
assistants, connector configuration, settings and policies. It also works for a
Personal workspace. This is a destructive workspace reset, separate from a
contact erasure request; it clears intake replay history. Retained address
suppression survives the reset; missing required suppression policy or key
material blocks it with a structured 409. Existing user review
and confirmation still apply.

The operation is transactional in both OSS and hosted editions. Optional
hosted content tables absent from OSS contribute zero to the deleted counts;
installed tables are processed. Missing required tables or other SQL failures
roll back the reset. A successful response reports `ok`, per-table `deleted`
counts and `total`; cascade-deleted children are not separately counted.
Off-database file cleanup and external-provider erasure are separate facilities.

## Roles

| Role | What it can do |
|---|---|
| Owner | Full control. Manages billing, members, and deletion. |
| Admin | Can manage assistants, channels, and KB. Cannot delete the workspace. |
| Member | Can chat with workspace assistants. Cannot configure them. |

## Sharing & inter-assistant

Each assistant has a sharing mode:

| Mode | Behavior |
|---|---|
| Off | Invisible. |
| Private | Discoverable; follows require approval. |
| Public | Auto-accept follows. |

When you follow another public assistant, your assistant gains an `askAssistant` tool to query it for the categories the owner has shared (calendar, knowledge, tasks, memories).

## Following assistants

Manage assistant connections from an assistant's Network tab. After following another, yours gains an `askAssistant` tool that fires when a question matches their domain.

## Account, workspace, resources

One account can own one or more workspaces, each owning the assistants and resources its members see.

- Account: one person, one login. Billing for every workspace you own runs through this account.
- Company brain: auto-created on signup, named after your business. The default workspace from day 1, solo or team.
- Extra workspace: for genuinely distinct contexts (multiple companies, client isolation, opt-in personal brain). Each extra workspace is billed independently and needs its own paid plan.
- Workspace-owned resources: assistants, memory, knowledge, connectors, channels, billing.

## Private workspace links

An authorized member can share a workspace address such as
`https://brain.example/s/product`. Opening it still requires membership on that
deployment; the link grants nothing and works independently of public-page or
custom-domain settings. The alias stays stable when the workspace name changes,
and old aliases continue to resolve after an explicit rename.

Brian's `getInternalShareLink` command returns the confirmed workspace link
when `pageId` is omitted. `setWorkspaceLinkAlias` changes the readable alias
for an owner/admin. Use the command result as the link and never infer an alias
from the workspace name.

## Notes for agents

- Memory is per (user, assistant): your personal memories stay yours, while team-scoped facts are shared across the workspace. The KB is workspace-wide, readable by every member's assistants subject to clearance.
- Inviting teammates never changes billing and never migrates data; it only adds members to the existing workspace.
- To have one assistant query another, the target must be Public (or a Private follow must be approved), and the owner must have shared the relevant category; then `askAssistant` becomes available.
- Configuration actions (assistants, channels, KB) require Admin or Owner; a Member can chat but not configure.

## Related

- [Assistants](./assistants.md)
- [Memory & knowledge](./memory-and-knowledge.md)
- [Channels](./channels.md)


## Departmental access readiness (implementation in progress)

`inspectWorkspaceAccess` returns the current server readiness along with visible departments, requests and grants. This release refuses new departmental delegation while the required enforcement is incomplete. `requestWorkspaceAccess`, manager capability additions and new read-grant approvals return `departmental_enforcement_incomplete`; changing client arguments or calling the shared approval endpoint cannot bypass that decision. Inspect existing settings, cancel or reject requests, reduce manager responsibilities and revoke grants through the normal commands. The web path is Settings > Department access, also reachable from Organization. Hierarchy changes do not change data permissions. Do not interpret these controls as certification that full strict departmental isolation is ready.

## Existing data review (implementation in progress)

Administrators can use `inspectScopeReview` to list metadata-only inventory for memories, entities, relationships, tasks, files, episodes, knowledge entries and knowledge chunks. `manageScopeReview` creates an explicit preview, applies up to 25 items per call, or cancels remaining work. Always use the saved review ID, current version and payload hash. An old-version retry returns recorded progress without replaying writes. After a lost response, inspect that review before choosing the next operation. Saved reviews are paginated in pages of 20: pass `nextReviewCursor` back as `reviewAfter` to `inspectScopeReview` for older jobs. Omit `reviewAfter` for the latest jobs. The inventory `after` cursor and exact `reviewId` selection are independent; the selected job remains available across history pages. Cursors are workspace-bound and unknown or foreign anchors fail without exposing their metadata. Data/policy changes can make remaining items stale; previously committed pages remain applied.

The web path is Settings > Department access > Review existing data. General means no Team requirement; it does not lower clearance, remove personal/assistant visibility or remove Project restrictions. Assigning a Team preserves those other protections. Holding may hold known derived descendants. This path cannot release held content. The tools require a verified human with current owner/admin authority; an unattended assistant or API credential cannot substitute its owner. Coverage is incomplete, and neither a completed batch nor an empty per-kind inventory certifies strict isolation.


Departmental inspection accepts `explain` with optional `memberId`, `assistantId`,
`contextTeamId`, `contextProjectId`, `targetTeamId`, `action` (`read` or `edit`) and
`sensitivity`. Only administrators may select another member. The result explains
separate read/mutation ceilings and independent active grant paths; it does not
bypass resource visibility or authorize edits. Unknown and unavailable directory
references return the same error. The explanation includes current authorized assistant and Project choices. A null read/edit Team list means unrestricted Team reach; an empty list means General-only reach. Neither value bypasses other resource gates. `history:"events"` inspects the filtered audit
with the same `after` / `expectedPolicyRevision` continuation protocol as requests
and grants. Raw audit payloads and content are not returned. Both operations use
the current verified human and server policy, not an assistant owner's authority.
