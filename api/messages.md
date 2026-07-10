---
title: Send messages
description: POST /api/v1/assistants/{assistantId}/messages runs one assistant turn; request/response fields, error table, code samples, and the followup tag contract.
tags: [api, messages]
canonical: https://sidan.ai/docs/api/messages
---

> Human-readable version: https://sidan.ai/docs/api/messages

`POST /api/v1/assistants/{assistantId}/messages` runs one assistant turn and returns the reply.

## Endpoint

```
POST https://api.sidan.ai/api/v1/assistants/{assistantId}/messages
Authorization: Bearer sk_live_...
Content-Type: application/json
```

## Request

```json
{
  "externalUserId": "user:42",
  "externalUserName": "Alice",
  "externalUserEmail": "alice@example.com",
  "identified": true,
  "sessionId": "thread-789",
  "message": "How does the proposal vote work?"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `externalUserId` | string | Yes | Stable id for the end user in your system. Opaque to sidanclaw, so namespace it however you want (e.g. `user:42`, `wallet:addr1q...`). |
| `externalUserName` | string | No | Display name for the visitor. Shown in the assistant's member list. |
| `externalUserEmail` | string | No | Email for the visitor. Implies `identified=true` and enables auto-merge if the same human later signs up via OAuth. |
| `identified` | boolean | No | Opt into Tier 1 (memory on) without an email. See [Identity & memory](identity.md). |
| `sessionId` | string | No | Conversation key. Reuse to continue a thread; pass a new value to start fresh. Defaults to `externalUserId` (one thread per user). |
| `message` | string | Yes | The user's text. Max 16,000 characters. |

## Response (200)

```json
{
  "sessionId": "thread-789",
  "messageId": "9f1e7c2a-...",
  "reply": "Proposals reach a vote 7 days after submission. Quorum is 30% of staked supply...",
  "model": "gemini-3-flash-preview"
}
```

| Field | Type | Description |
|---|---|---|
| `sessionId` | string | Echo of the session id (or the default if you did not pass one). |
| `messageId` | string | Stable id of the stored assistant turn. |
| `reply` | string | The assistant's reply text. |
| `model` | string | Which model produced the reply (e.g. `gemini-3-flash-standard` for Standard, `gemini-3-flash-preview` for Pro, `gemini-3.5-flash` for Max, `gemini-3-pro-research` for Research). |

## Errors

Error responses use the shape `{ "error": "<slug>", "detail": "..." }`.

| HTTP | `error` | When it happens |
|---|---|---|
| 400 | `invalid_input` | Body missing required field, malformed email, or message too long. |
| 401 | `invalid_api_key` | Authorization header missing, malformed, hash mismatch, or the key does not match the assistant in the URL. |
| 403 | `key_revoked` | Key exists but has been revoked. |
| 404 | `assistant_not_found` | Key valid but the assistant no longer exists. |
| 429 | `budget_exhausted` | The workspace has no active plan (trial ended or plan lapsed). The owner must pick a plan, or self-host. |
| 502 | `upstream_failed` | LLM provider error after retries. Treat as transient. |
| 500 | `internal` | Anything else. Open a bug report. |

## Example: curl

```bash
curl -X POST "https://api.sidan.ai/api/v1/assistants/<ASSISTANT_ID>/messages" \
  -H "Authorization: Bearer sk_live_..." \
  -H "Content-Type: application/json" \
  -d '{
    "externalUserId": "user:42",
    "externalUserName": "Alice",
    "externalUserEmail": "alice@example.com",
    "sessionId": "thread-789",
    "message": "How does the proposal vote work?"
  }'
```

## Example: fetch (Node.js / Deno)

```javascript
const reply = await fetch(
  `https://api.sidan.ai/api/v1/assistants/${assistantId}/messages`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.SIDANCLAW_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      externalUserId: `user:${userId}`,
      externalUserEmail: userEmail,           // optional, Tier 1 + auto-merge
      sessionId: threadId,                    // optional, defaults to externalUserId
      message: userText,
    }),
  }
).then((r) => r.json());

console.log(reply.reply);
```

## Example: Python

```python
import os
import requests

resp = requests.post(
    f"https://api.sidan.ai/api/v1/assistants/{assistant_id}/messages",
    headers={
        "Authorization": f"Bearer {os.environ['SIDANCLAW_API_KEY']}",
        "Content-Type": "application/json",
    },
    json={
        "externalUserId": f"user:{user_id}",
        "externalUserEmail": user_email,   # optional, Tier 1 + auto-merge
        "sessionId": thread_id,            # optional
        "message": user_text,
    },
    timeout=30,
)
resp.raise_for_status()
print(resp.json()["reply"])
```

## Idempotency

Not implemented in v1. Retrying a failed request may produce a duplicate assistant turn in the same session. Plan for this by storing your own request id and checking your database before retrying. `Idempotency-Key` header support is on the roadmap.

## Latency

Synchronous endpoint. Typical response time is 3-15 s; allow at least 30 s of timeout in your backend. Streaming (SSE) is on the roadmap if your UX needs sub-second token latency.

## Follow-up suggestions: the `<followup>` tag

When enabled for a client, the assistant may append a machine-readable block of 2-4 short follow-up questions at the very end of `reply`. Today the public API does NOT inject the instruction that produces this tag, so v1 API consumers will not normally see it. Clients that build on top of sidanclaw (the web chat at sidan.ai, embeds, custom UIs) do. The format is documented so any client can render the suggestions as chips or strip the tag from the displayed body, deterministically.

### Tag shape

When present, the tag is the very last thing in `reply` (no trailing text). The contents are a JSON array of 2-4 strings, each a short stand-alone question. Anything before the opening `<followup` is the user-visible body.

Example reply with a tag:

```json
{
  "sessionId": "thread-789",
  "messageId": "9f1e7c2a-...",
  "reply": "CIP-1694 governance relies on three groups: Delegate Representatives (DReps), Stake Pool Operators (SPOs), and the Constitutional Committee. You participate by delegating your ADA's voting power to a DRep, who votes on your behalf regarding treasury withdrawals, parameter changes, and more. For an action to pass, it must meet specific consensus thresholds from these governing bodies.\n\n<followup>[\"What is a DRep?\", \"How do I delegate my voting power?\", \"What are governance thresholds?\"]</followup>",
  "model": "gemini-3-flash-preview"
}
```

### Handling in your client

Two valid strategies:

1. Parse and render. Split on the tag, show the prose part, and render each question as a clickable chip that re-sends the question text as the next user message.
2. Strip and ignore. If your surface has no chip affordance (plain SMS, voice, a logging pipeline), drop everything from `<followup` onward before showing the reply.

Always handle the partial-tag case during streaming or truncated text: if you find an opening `<followup` with no closing `</followup>`, hide everything from the opening marker to avoid flashing raw markup.

### Reference parser

Match `/<followup>\s*(\[[\s\S]*?\])\s*<\/followup>/` against `reply`, then `JSON.parse` the captured group and filter to non-empty strings (max 4). The display text is `reply.slice(0, indexOf('<followup')).trimEnd()`. The same logic ships in `@sidanclaw/shared` as `parseFollowUps(text)`.

```javascript
// Reference parser. Matches the @sidanclaw/shared parseFollowUps() helper.
const TAG = /<followup>\s*(\[[\s\S]*?\])\s*<\/followup>/;

function parseFollowUps(reply) {
  const open = reply.indexOf("<followup");
  if (open === -1) return { display: reply, questions: [] };

  // Hide the partial tag during streaming so raw markup never flashes.
  const display = reply.slice(0, open).trimEnd();

  const match = reply.match(TAG);
  if (!match) return { display, questions: [] };

  try {
    const parsed = JSON.parse(match[1]);
    const questions = Array.isArray(parsed)
      ? parsed.filter((q) => typeof q === "string" && q.trim().length > 0).slice(0, 4)
      : [];
    return { display, questions };
  } catch {
    return { display, questions: [] };
  }
}

// Render: show `display` as the assistant's body, render each `questions[i]`
// as a clickable chip that re-sends the question text. If your surface has
// no chip affordance, ignore `questions` and only show `display`.
```

## Notes for agents

- `sessionId` is the only multi-turn mechanism: reuse it to continue a thread, pass a new value to start fresh. If you omit it, it defaults to `externalUserId`, so all of a user's turns collapse into one thread.
- Enforce the 16,000-character `message` limit on your side before sending; over-length bodies return `400 invalid_input`.
- Treat `502 upstream_failed` as transient and retry with backoff. Because there is no idempotency in v1, guard retries with your own request id so a duplicate turn is not appended.
- Set your client timeout to at least 30 s. The call is synchronous and typically takes 3-15 s.
- Do not wait for a `<followup>` tag on the raw API path; v1 does not inject the instruction that produces it. If one ever appears, parse or strip it with the reference parser above.

## Related

- [Overview](overview.md)
- [Authentication](authentication.md)
- [Identity & memory](identity.md)
- [Connector identity](connector-identity.md)
