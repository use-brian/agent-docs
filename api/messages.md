---
title: Send messages
description: POST /api/v1/assistants/{assistantId}/messages runs one assistant turn with JSON or live SSE delivery; request fields, event contract, errors, and examples.
tags: [api, messages]
canonical: https://usebrian.ai/docs/api/messages
---

> Human-readable version: https://usebrian.ai/docs/api/messages

`POST /api/v1/assistants/{assistantId}/messages` runs one assistant turn. JSON is the default response; an explicit `Accept: text/event-stream` streams the live reply.

## Endpoint

```
POST https://api.usebrian.ai/api/v1/assistants/{assistantId}/messages
Authorization: Bearer sk_live_...
Content-Type: application/json
Accept: text/event-stream  # optional
```

## Request

```json
{
  "externalUserId": "user:42",
  "externalUserName": "Alice",
  "externalUserEmail": "alice@example.com",
  "identified": true,
  "claims": {
    "email": "alice@example.com",
    "orgId": "acme-corp",
    "roles": ["admin"]
  },
  "endUserContext": "plan: pro\nopen orders: #4471",
  "clientMemory": {
    "key": "consultation-profile-v1",
    "summary": "Alice leads operations at Acme",
    "detail": "Visitor-supplied consultation context.",
    "tags": ["consultation", "prospect"]
  },
  "clientLead": {
    "key": "consultation-handoff-v1",
    "name": "Alice - consultation"
  },
  "allowPublicResearch": true,
  "sessionId": "thread-789",
  "message": "How does the proposal vote work?"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `externalUserId` | string | Yes | Stable id for the end user in your system. Opaque to Use Brian, so namespace it however you want (e.g. `user:42`, `wallet:addr1q...`). |
| `externalUserName` | string | No | Display name for the visitor. Shown in the assistant's member list. |
| `externalUserEmail` | string | No | Email for the visitor. Alias of `claims.email`. Implies `identified=true` and enables auto-merge if the same human later signs up via OAuth. |
| `identified` | boolean | No | Opt into Tier 1 (memory on) without an email. See [Identity & memory](identity.md). |
| `claims` | object | No | Turn-scoped identity your backend attests for this request: `email`, `orgId`, `roles`. See [End-user claims](#end-user-claims). |
| `endUserContext` | string | No | Free-form account context for this turn (plan tier, open orders, ticket state). Max 4,000 characters. Passed to the model verbatim, labelled as attested by you and unverified. Never stored. |
| `clientMemory` | object | No | Identified external turns only. Deterministically upserts one internal, exact-self memory using `{ key, summary, detail?, tags? }`. See [Deterministic client memory](#deterministic-client-memory). |
| `clientLead` | object | No | Identified external turns carrying a verified email only. Atomically ensures the exact client contact and one idempotent linked CRM lead using `{ key, name? }`. See [Deterministic client lead](#deterministic-client-lead). |
| `allowPublicResearch` | boolean | No | `public_research` keys only. Set true only after explicit user consent; false or absent structurally withholds `webSearch` and `urlReader` for the turn. |
| `actorEmail` | string | No | Internal-audience keys only: the workspace member this turn acts as. Omitted, the turn acts as the key's creator. Returns `400` on an external-audience key and `403 actor_not_member` if the email does not resolve to a workspace member. See [Key audience](identity.md#key-audience). |
| `sessionId` | string | No | Conversation key. Reuse to continue a thread; pass a new value to start fresh. Defaults to `externalUserId` (one thread per user). |
| `message` | string | Yes | The user's text. Max 16,000 characters. |

The body is strict: an unknown top-level field, or an unknown field inside `claims`, `clientMemory`, or `clientLead`, returns `400 invalid_input`.

## Deterministic client memory

Use `clientMemory` when your backend has authenticated the end user and has
permission to retain a bounded profile, handoff, or consultation snapshot.
The request must be an identified external turn, and the keyed assistant must
have at least internal clearance. The server always stores the memory at
`internal`, exact to the resolved user and assistant, with exactly the
machine-minted `client:<externalUserId>` compartment. The caller cannot supply
sensitivity or compartments. Reusing `key` supersedes the prior value.

The self-memory exception applies only to memory. It does not expose the
visitor's team-only contact, workspace memory, knowledge, files, CRM,
connectors, confidential data, another assistant's memory, or another
visitor's rows. Public-web provenance never lowers retained client context to
public.

## Deterministic client lead

Use `clientLead` when a verified application handoff should create a CRM lead
without relying on the model to call a CRM tool. The request must be an
identified external turn carrying `claims.email` or `externalUserEmail`. Use Brian atomically ensures the team-visible client
contact and one linked deal at the `lead` stage, stamped with the exact client
compartment. The request cannot choose the workspace, contact id, stage,
sensitivity, or compartments. Reusing `key` for the same API-key client
identity reuses the deal instead of creating a duplicate.

## End-user claims

Use these when you are exposing one assistant to **your own end users** and each conversation must stay private to that person.

Use Brian never authenticates your users. Your backend does that and attests the result on each request, so a claim is exactly as trustworthy as the API key. Keep the key server-side: a browser-held key would let any visitor attest any identity.

| Field | Type | What it does |
|---|---|---|
| `claims.email` | string | The same field as `externalUserEmail`, with the same effect (implies Tier 1). Send one or the other; sending both with **different** values returns `400 invalid_input`. |
| `claims.orgId` | string | Your tenant id for this user. Forwarded to your MCP connector as `X-UseBrian-Actor-Org`. No other effect today. |
| `claims.roles` | string[] | Up to 16 role labels. Visible to the model for tone and routing, and **never** sent to your connector as a header. |

Two rules matter when you integrate:

- **`externalUserId` identifies; `claims` authorize.** `externalUserId` is the permanent index key and becomes the user's durable identity in Use Brian. Claims are scoped to the single request and are never stored as authority. Send claims on **every** authenticated request, and omit them when the visitor is logged out: Use Brian does not carry a previous session's claims forward, so a returning logged-out visitor is treated as anonymous and the assistant is told to withhold account specifics.
- **Roles are advisory, not permission.** Do not implement access control by sending a role and expecting Use Brian or your connector to honour it. Authorize inside your own MCP server, keyed on `X-UseBrian-Actor-Id`. See [Connector identity](connector-identity.md).

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
| `model` | string | Which model produced the reply (e.g. `gemini-3-flash-standard` for Standard, `gemini-3-flash-preview` for Pro, `gemini-3.5-flash` for Max, `gemini-3-pro-research` for Research). Since the model registry, deployments may also serve metered pay-per-use models (e.g. `qwen3.7-plus`, `deepseek-v4-pro` via DashScope); those ids appear here verbatim, and the billing tier is recorded separately from the model id, so do not infer pricing from the model string. |

## Streaming with SSE

Send `Accept: text/event-stream` to the same POST endpoint. The request body, authentication, identity, session, retry/edit, tool, persistence, and billing semantics are unchanged. The response emits real query-loop deltas as the model produces them:

```text
event: session
data: {"sessionId":"thread-789"}

event: text_delta
data: {"text":"Proposals reach "}

event: text_delta
data: {"text":"a vote after..."}

event: turn_complete
data: {"sessionId":"thread-789","messageId":"9f1e7c2a-...","model":"gemini-3-flash-preview"}

event: done
data: {}
```

Authentication, validation, assistant, and budget failures found before streaming starts retain their normal non-2xx JSON response. If the provider fails after the SSE response opens, the stream emits `error` with the ordinary error slug and optional detail, then `done`, then closes. A client must not treat EOF without both `turn_complete` and `done` as a successful turn.

Use `fetch()` and a `ReadableStream` reader for POST-returning SSE. Browser `EventSource` cannot send this POST body or the bearer header. Preserve the response's `Cache-Control: no-cache, no-transform`; proxies that transform or compress SSE may buffer it into one final chunk.

## Which model answers

You do not choose the model per request. Every turn on this endpoint runs on the
assistant's **API model tier**, which the workspace owner sets in the app at
Assistant, API, Model tier. The same tier governs the assistant's public chat
link (`/c/<token>`), because both share one pipeline and bill the same
workspace.

That tier is independent of the tier the assistant uses on its own messaging
channels, so an owner can serve API traffic on Standard while their Telegram
bot runs on Max, or the reverse. If replies change quality without a change on
your side, that setting is the first thing to check.

The workspace plan still clamps the tier: a plan that does not include Pro or
Max resolves down before the call. Always read the concrete model from the
response's `model` field rather than assuming a tier.

## Errors

Error responses use the shape `{ "error": "<slug>", "detail": "..." }`.

| HTTP | `error` | When it happens |
|---|---|---|
| 400 | `invalid_input` | Body missing required field, malformed email, invalid `clientMemory` or `clientLead`, contradictory audience fields, or message too long. |
| 401 | `invalid_api_key` | Authorization header missing, malformed, hash mismatch, or the key does not match the assistant in the URL. |
| 403 | `key_revoked` | Key exists but has been revoked. |
| 403 | `actor_not_member` | Internal-audience keys only: the attributed actor (or the key's creator) is not a member of the assistant's workspace. |
| 404 | `assistant_not_found` | Key valid but the assistant no longer exists. |
| 429 | `budget_exhausted` | The workspace has no active plan (trial ended or plan lapsed). The owner must pick a plan, or self-host. |
| 502 | `upstream_failed` | LLM provider error after retries. Treat as transient. |
| 500 | `internal` | Anything else. Open a bug report. |

## Example: curl

```bash
curl -X POST "https://api.usebrian.ai/api/v1/assistants/<ASSISTANT_ID>/messages" \
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
  `https://api.usebrian.ai/api/v1/assistants/${assistantId}/messages`,
  {
    method: "POST",
    headers: {
      Authorization: `Bearer ${process.env.USEBRIAN_API_KEY}`,
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
    f"https://api.usebrian.ai/api/v1/assistants/{assistant_id}/messages",
    headers={
        "Authorization": f"Bearer {os.environ['USEBRIAN_API_KEY']}",
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

JSON mode returns when the turn completes, typically in 3-15 seconds. SSE mode delivers `text_delta` events as soon as the model produces visible text. Allow at least 30 seconds for either connection and longer for tool-using turns.

## Follow-up suggestions: the `<followup>` tag

When enabled for a client, the assistant may append a machine-readable block of 2-4 short follow-up questions at the very end of `reply`. Today the public API does NOT inject the instruction that produces this tag, so v1 API consumers will not normally see it. Clients that build on top of Use Brian (the web chat at usebrian.ai, embeds, custom UIs) do. The format is documented so any client can render the suggestions as chips or strip the tag from the displayed body, deterministically.

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

Match `/<followup>\s*(\[[\s\S]*?\])\s*<\/followup>/` against `reply`, then `JSON.parse` the captured group and filter to non-empty strings (max 4). The display text is `reply.slice(0, indexOf('<followup')).trimEnd()`. The same logic ships in `@use-brian/shared` as `parseFollowUps(text)`.

```javascript
// Reference parser. Matches the @use-brian/shared parseFollowUps() helper.
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
