---
title: Privacy and Data
description: What hosted sidanclaw stores, the controls a user has, retention windows, and the third parties involved.
tags: [operations, privacy]
canonical: https://sidan.ai/docs/privacy
---

> Human-readable version: https://sidan.ai/docs/privacy

sidanclaw is a shared brain for a team. Its value depends on remembering the business, so it stores data. This page states what, and what controls exist. sidanclaw does not train on customer data and does not sell data.

## What is stored

| Data | Detail |
|---|---|
| Account | Email, display name (Google OAuth), any inferred timezone |
| Conversation history | Every message in every session, per channel |
| Memories | Structured facts the assistant extracted about the user |
| Channel credentials | Encrypted at rest with AES-256-GCM. The plaintext token is never logged or stored |
| Usage | Per-turn token counts and cost, for budget tracking |

## Where it lives

| Zone | Contents | Control |
|---|---|---|
| You | Browser localStorage (UI prefs), channel app installs | 100% yours |
| sidanclaw database | Account, sessions, memory, knowledge base, encrypted channel credentials. Workspace-scoped, exportable, deletable | Settings: Privacy |
| Google Gemini | Inference only: system prompt, selected context, the current turn. No retention beyond the response | API contract |

## User controls

| Control | Path | Effect |
|---|---|---|
| Manage memories | Settings: Privacy: Memories | Search, edit, or delete any memory; "Delete all" wipes them in bulk |
| Delete account | Settings: Privacy: Delete account | Cascading delete of the user row, all sessions, memories, channel integrations, and owned assistants. Irreversible |
| Forget in chat | Any chat | Ask "forget that" or "delete the memory about X"; the assistant uses its `deleteMemory` tool |

## Retention

| Data | Window |
|---|---|
| Tier 2 anonymous shadow users (API path, unidentified Slack/Telegram) | Auto-pruned 30 days after the last session |
| Identified users | Persist until the account is deleted |
| Backend logs (analytics events, error reports) | Cloud Run defaults, typically 30 days |

## Third parties

| Provider | Role |
|---|---|
| Google Gemini | Primary inference |
| Anthropic Claude Haiku | Fallback when Gemini returns a retryable error |
| Brave Search, Serper, Tavily | Web research, chosen per query |
| xAI (Grok) | X-aware `urlReader` / `xSearch` tools |
| Jina Reader | JS-heavy page reads |
| Connector providers | Connector calls (Google Calendar, Gmail, Notion, GitHub, and others) route through the connector's own provider |

Model-training on customer data is forbidden in both the Gemini and Claude contracts. Distribution providers (Meta, X) are listed in the full Privacy Policy.

## Data export

An export tool is on the roadmap. The database schema is documented; a self-host deployment has direct access to its own data. Hosted users can contact sidanclaw for a one-off export.

## Notes for agents

- Anything an agent writes over MCP lands in the sidanclaw database zone: exportable and deletable by the user, and covered by the same "no training, no selling" guarantee.
- A memory an agent saves is user-controllable: the user can delete it from Settings or by asking the assistant to forget it.
- Inference of any given turn may route to Gemini or, on a retryable error, to Claude Haiku. Neither provider retains or trains on the content.
- Unidentified API/channel sessions are pruned after 30 days; do not assume long-lived state for an anonymous caller.

## Related

- [Pricing and credits](pricing-and-credits.md)
- [Self-hosting](../self-hosting.md)
