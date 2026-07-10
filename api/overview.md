---
title: Public API Overview
description: Server-to-server JSON API for embedding a sidanclaw assistant; API-key auth, consumer-supplied identity, one assistant turn per request.
tags: [api, overview]
canonical: https://sidan.ai/docs/api
---

> Human-readable version: https://sidan.ai/docs/api

The public API lets a third-party service embed a sidanclaw assistant. Authentication is by API key, calls are server-to-server JSON, and identity is consumer-supplied. One request runs one assistant turn: the model thinks, optionally searches the knowledge base, and returns a reply.

## Endpoint

```
POST https://api.sidan.ai/api/v1/assistants/{assistantId}/messages
Authorization: Bearer sk_live_...
Content-Type: application/json
```

One endpoint, one auth header, JSON in / JSON out. Each request runs exactly one assistant turn.

## When to use

- You are building a website or app and want "ask the assistant" without making every visitor sign up for sidanclaw.
- You have your own user accounts and want each user's conversation memory to follow them.
- You want full control over the chat UI (colors, branding, history, abuse mitigation).

## When not to use

- Your users are signing up for sidanclaw anyway. The web chat is more featureful.
- You need streaming output today. v1 is synchronous JSON only; SSE is on the roadmap.
- You need the assistant to call write-tools (Calendar, Gmail). The API path deliberately does not expose write-tools, because there is no human in the loop to approve confirmations.

## Request flow

1. Your backend receives a user message.
2. `POST /api/v1/assistants/{id}/messages` with the message plus your stable user id.
3. sidanclaw resolves the user, runs the assistant turn, and returns the reply (3-15 s typical).
4. Your frontend shows the reply.

## Setup

1. Sign in to sidanclaw, create the assistant you want to expose, and fill its knowledge base with anything the assistant should know.
2. Open the assistant detail, API tab. Click "New key", name it (e.g. "production"), and copy the plaintext shown. You will not see the secret again, so store it in your secret manager.
3. Call `POST /api/v1/assistants/{id}/messages` from your backend. See [Send messages](messages.md) for the wire format.

## Notes for agents

- Base URL is `https://api.sidan.ai`. The `{assistantId}` path segment is a UUID the owner copies from the assistant detail, API tab.
- Read-only tool surface: every tool that requires a confirmation (send, create, delete) is stripped, because there is no human in the loop to approve it. Read-only tools stay available (web search, web fetch, knowledge-base search, read-only connector reads). Do not expect the assistant to send email, create calendar events, or take other confirmable actions here.
- No streaming in v1: the call blocks until the turn finishes, then returns one JSON body. Budget for it (see the latency note on the messages page).
- One turn per request. Multi-turn context is carried by reusing `sessionId`, not by the endpoint holding state.

## Related

- [Authentication](authentication.md)
- [Send messages](messages.md)
- [Identity & memory](identity.md)
- [Connector identity](connector-identity.md)
