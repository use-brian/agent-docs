---
title: Workspaces & sharing
description: A workspace is the unit of brain identity, billing, and membership; sharing lets assistants query other people's assistants.
tags: [concepts, workspaces]
canonical: https://sidan.ai/docs/workspaces
---

> Human-readable version: https://sidan.ai/docs/workspaces

Workspaces turn an assistant into a shared resource. Sharing lets your assistant call other people's assistants for data only they have.

## Workspaces

A workspace is the unit of brain identity, billing, and membership: a company brain from day 1, even when you are the only member. One workspace is auto-created when you sign up (named from the business you tell us about during onboarding). Inviting teammates later does not migrate anything; the same workspace just gains new members. A workspace owns its assistants, memories, knowledge base, connector instances, and channel installs. Memory is per (user, assistant): team-scoped facts are shared across the workspace, while personal memories stay yours.

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

## Notes for agents

- Memory is per (user, assistant): your personal memories stay yours, while team-scoped facts are shared across the workspace. The KB is workspace-wide, readable by every member's assistants subject to clearance.
- Inviting teammates never changes billing and never migrates data; it only adds members to the existing workspace.
- To have one assistant query another, the target must be Public (or a Private follow must be approved), and the owner must have shared the relevant category; then `askAssistant` becomes available.
- Configuration actions (assistants, channels, KB) require Admin or Owner; a Member can chat but not configure.

## Related

- [Assistants](./assistants.md)
- [Memory & knowledge](./memory-and-knowledge.md)
- [Channels](./channels.md)
