---
title: Tools & connectors
description: Connectors expose third-party APIs as per-tool-governed capabilities; scheduled tasks let the assistant run jobs on its own.
tags: [concepts, tools]
canonical: https://sidan.ai/docs/tools
---

> Human-readable version: https://sidan.ai/docs/tools

Tools give your assistant the ability to do things, not just talk. Connectors expose third-party APIs (Google Calendar, Gmail, Notion, GitHub, and more). Scheduled tasks let the assistant run jobs on its own.

## Connectors (MCP)

Connect a service from Studio -> Connectors. Each connector exposes a set of tools (for example Google Calendar exposes `googleCalendarCreateEvent`, `googleCalendarListEvents`, etc.). After connecting, you decide which tools each assistant can use from the assistant's Tools tab. Workspace Files is first-party (no external account); most of the rest authenticate via OAuth or a personal access token.

### Official built-in connectors

| Connector | Notes |
|---|---|
| Google Calendar | Also exposes Google Tasks tools. |
| Gmail | Send and read email. |
| Notion | |
| Google Docs, Sheets & Slides | |
| GitHub | |
| Fathom | |
| Workspace Files | First-party; no external account. |
| Google Cloud Storage | Bring-your-own storage via a service-account key; exposes no assistant tools. |

## Per-tool policy

Three modes per tool:

| Policy | Behavior |
|---|---|
| Allow | Runs without asking. |
| Ask | Confirms in chat before running each time. |
| Block | Never runs. |

Read tools default to Allow; write and destructive tools default to Ask. You can change the defaults per assistant.

### Tool policy matrix

Defaults by tool class, all overridable per assistant from the Tools tab:

| Tool class | Example | Default |
|---|---|---|
| Read | List events, search Notion, fetch a URL | Allow |
| Write | Send email, create event, write a page | Ask |
| Destructive | Delete event, archive thread, drop a row | Ask |

## Write grants

Separate from per-tool policy, each assistant carries a write-grant list per connector. A write or destructive connector tool runs only if the assistant's owner granted that specific action (for example `githubCreateIssue`) in Studio -> Assistants -> Tools. Grants bind every caller of the assistant the same way: team members, scheduled tasks, assistant-to-assistant calls, and API calls all get the same decision. A fresh assistant has no grants, so its connector write actions are refused until granted. Read tools are unaffected.

A refused write returns an "action not granted" tool error naming the connector and action. Surface it to the user; only the assistant's owner can grant the action in Studio.

## Scheduled tasks

Tell your assistant when to run something: "every weekday at 9am, summarize the team's Slack" or "follow up with the Acme lead in two hours." sidanclaw schedules a cron job that runs on its own session, executes tools, and delivers the result via your preferred channel.

Where to see them: scheduled work lives on the Workflow surface. Each scheduled workflow shows its cadence and last run; pause, edit, or delete it there. Jobs survive restarts.

## Not to be confused with workspace Tasks

Scheduled tasks are timed jobs. They fire on a cron and run an assistant turn. Workspace Tasks (see the Tasks page) are the brain primitive: durable forward-commitments the assistant tracks for you. Different things; an assistant can use one to remember to schedule the other.

## Notes for agents

- A tool that is `Block` never runs, and write/destructive tools default to Ask, so a write action may pause for user confirmation before it executes. Do not assume a write succeeded until the confirmation resolves.
- A connector write can also be refused with an "action not granted" error when the assistant lacks the write grant for that action. This is not a transient failure; retrying will not help. Tell the user which action needs granting in Studio -> Assistants -> Tools.
- Connector tools only exist after the service is connected in Studio -> Connectors and enabled for the assistant in its Tools tab. Never reference a connector tool that has not been connected.
- Workspace Files is first-party and needs no external account; every other connector requires OAuth or a personal access token.
- "Every weekday at 9am..." style requests create a scheduled task (a cron job on the assistant), which is distinct from a workspace Task.

## Related

- [Tasks](./tasks.md)
- [Workflows](./workflows.md)
- [Channels](./channels.md)
