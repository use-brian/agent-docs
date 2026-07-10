---
title: Assistants
description: An assistant is the unit of identity in sidanclaw; it owns memory, channels, tools, scheduled tasks, and a system prompt.
tags: [concepts, assistants]
canonical: https://sidan.ai/docs/assistants
---

> Human-readable version: https://sidan.ai/docs/assistants

An assistant is the unit of identity in sidanclaw. It owns memory, channel connections, tools, scheduled tasks, and a system prompt. A workspace can have many assistants. Each is a separate persona with its own context.

## Core identity

An assistant's core is its name, persona, and system prompt. Every capability below hangs off that identity.

## What an assistant has

| Capability | What it is |
|---|---|
| Memory | Facts the assistant has learned about you and the people, companies, and deals around you. Saved automatically as you chat, retrieved each turn, editable from Settings -> Privacy -> Memories. |
| Channels | Where the assistant can be reached. Connected once at the workspace level and assigned to assistants. Web is always on; Telegram and Slack are opt-in via BYO credentials. |
| Tools | Built-in (memory, web search, knowledge) and connector-based (Google Calendar, Gmail, Notion, etc.). |
| Scheduled tasks | Cron-like jobs the assistant runs on its own session ("every weekday at 9am, summarize my unread email"). |
| Knowledge base | Shared facts the assistant can search. Workspace-scoped, sensitivity-tiered, optionally synced from a GitHub repo. |

## Why have multiple assistants

Memory is scoped per-assistant, so a single assistant ends up confused if you mix client-facing and internal contexts. The clean pattern is one assistant for internal ops, one for sales, and one per public-facing project (for example a community Q&A bot).

## Solo vs. workspace-shared

Every assistant belongs to a workspace: your company brain from day 1, even when you are the only member. Invite teammates and the same assistant becomes shared. Everyone talks to the same memory and knowledge base. See Workspaces & sharing.

## Notes for agents

- Memory is scoped per (user, assistant). Switching a user to a different assistant changes what is remembered about them; do not assume facts carry across assistants.
- Channels are workspace-owned, not assistant-owned. To reach an assistant on Telegram or Slack, the channel must be connected at the workspace level and routed to that assistant.
- Built-in tools (memory, web search, knowledge) exist without any connector; connector tools (Calendar, Gmail, Notion) require the service to be connected first.
- "Scheduled tasks" on an assistant are cron jobs, distinct from workspace Tasks (the brain primitive). Do not conflate the two.

## Related

- [Memory & knowledge](./memory-and-knowledge.md)
- [Workspaces & sharing](./workspaces.md)
- [Channels](./channels.md)
- [Tools & connectors](./tools-and-connectors.md)
