---
title: Assistants
description: An assistant is the unit of identity in Use Brian; it owns memory and behavior and is assigned to workspace channels and tools.
tags: [concepts, assistants]
canonical: https://usebrian.ai/docs/assistants
---

> Human-readable version: https://usebrian.ai/docs/assistants

An assistant is the unit of identity in Use Brian. It owns memory, behavior, tool access, and scheduled tasks, and it can be assigned to workspace-owned channels. A workspace can have many assistants. Each is a separate persona with its own context.

## Core identity

An assistant's core is its name, persona, and system prompt. Every capability below hangs off that identity.

## What an assistant has

| Capability | What it is |
|---|---|
| Memory | Facts the assistant has learned about you and the people, companies, and deals around you. Saved automatically as you chat, retrieved each turn, editable from Settings -> Privacy -> Memories. |
| Channels | Where the assistant can be reached. Channels are created or connected at the workspace level, then assigned to an assistant. Assistant email uses a required **Handled by default** assignment plus optional exact sender routes on the inbox itself. |
| Tools | Built-in (memory, web search, knowledge) and connector-based (Google Calendar, Gmail, Notion, etc.). |
| Scheduled tasks | Cron-like jobs the assistant runs on its own session ("every weekday at 9am, summarize my unread email"). |
| Knowledge base | Shared facts the assistant can search. Workspace-scoped, sensitivity-tiered, optionally synced from a GitHub repo. |

## Why have multiple assistants

Memory is scoped per-assistant, so a single assistant ends up confused if you mix client-facing and internal contexts. The clean pattern is one assistant for internal ops, one for sales, and one per public-facing project (for example a community Q&A bot).

## Solo vs. workspace-shared

Every assistant belongs to a workspace: your company brain from day 1, even when you are the only member. Invite teammates and the same assistant becomes shared. Everyone talks to the same memory and knowledge base. See Workspaces & sharing.

## Notes for agents

- Memory is scoped per (user, assistant). Switching a user to a different assistant changes what is remembered about them; do not assume facts carry across assistants.
- Channels are workspace-owned, not assistant-owned. Route messaging Channels to an assistant from Studio -> Channels. For assistant email, create the inbox there, set **Handled by default**, and manage access and exact sender routes there; do not configure AgentMail from the assistant's Tools page.
- Built-in tools (memory, web search, knowledge) exist without any connector; connector tools (Calendar, Gmail, Notion) require the service to be connected first.
- "Scheduled tasks" on an assistant are cron jobs, distinct from workspace Tasks (the brain primitive). Do not conflate the two.

## Related

- [Memory & knowledge](./memory-and-knowledge.md)
- [Workspaces & sharing](./workspaces.md)
- [Channels](./channels.md)
- [Tools & connectors](./tools-and-connectors.md)
