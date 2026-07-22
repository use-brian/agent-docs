---
title: Ingest append contract (ub.ingest.append.v1)
description: The versioned contract an external service implements to receive a connector's normalized event stream from the platform - one idempotent POST endpoint accepting batches of canonical message records, answered with accepted/duplicates accounting and a durably-committed ack cursor.
tags: [api, ingest, append, contract, archive]
canonical: https://usebrian.ai/docs/api/ingest-append
---

> This file is the authoritative public reference for the contract (a human docs page is forthcoming). Version: `ub.ingest.append.v1`. Breaking changes bump the version; the shape below never mutates in place.

An **external sink** is an external service (for example a messaging-archive service) that receives a connector's normalized events from the platform. The platform delivers through a durable outbox + relay worker: your endpoint is POSTed batches with retries, and the platform's source cursor advances **only** when you acknowledge durable storage. To participate you implement exactly one endpoint accepting this contract.

## What your endpoint must do

1. Accept `POST` requests with the JSON request body below.
2. Authenticate the request (bearer token or HMAC signature - agreed when the sink is configured).
3. Store the batch **idempotently**, keyed `(instance_id, provider_message_id)` per message - e.g. `INSERT ... ON CONFLICT DO NOTHING`. The relay retries freely (at-least-once delivery); your idempotency makes it exactly-once in effect.
4. Reply `200` with the response body below. `accepted + duplicates` MUST equal `messages.length`; any mismatch is treated as a partial failure and the whole batch is retried.
5. Echo `ack_cursor` only after your storage durably committed the batch - it is what advances the platform-side cursor.

## Request

`POST {your endpoint_url}`

Headers:

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `X-UB-Idempotency-Key` | Whole-batch retry key (a UUID), stable across retries of the same batch. |
| `Authorization` | `Bearer <secret>` - when the sink is configured with bearer auth. |
| `X-UB-Signature` | `sha256=<hex>` of HMAC-SHA256(secret, raw request body) - when configured with HMAC auth. Verify against the raw bytes, before JSON parsing. |

Body:

```jsonc
{
  "contract": "ub.ingest.append.v1",
  "instance_id": "uuid",          // the platform connector instance - half of the per-message idempotency key
  "source": "wechat" | "whatsapp" | "<provider>",   // informational label
  "workspace_id": "uuid",
  "owner_user_id": "uuid | null", // compartment owner for person-scoped corpora
  "cursor": { /* opaque */ } | null,   // echo it back as ack_cursor once durable
  "messages": [                   // 1..n canonical message records
    {
      "provider_message_id": "string",   // idempotency key WITH instance_id
      "conversation_id": "string",
      "sender_id": "string",
      "sender_display": "string | null",
      "sent_at": "RFC3339 timestamp",
      "direction": "inbound" | "outbound",
      "kind": "text" | "image" | "voice" | "file" | "link",
      "body_text": "string | null",      // text or transcript
      "media_ref": { "filename": "string", "mime": "string", "size_bytes": 0 } | null,  // metadata only in v1
      "reply_to_provider_id": "string | null",
      "raw_provider_blob": { /* opaque provider payload */ } | null
    }
  ]
}
```

## Response

Reply `200 OK` with:

```jsonc
{
  "contract": "ub.ingest.append.v1",
  "accepted": 0,      // rows newly stored by this request
  "duplicates": 0,    // rows skipped by your idempotency (ON CONFLICT)
  "ack_cursor": { /* the cursor you have durably committed */ },   // optional but expected; omitting it completes the delivery without advancing the platform cursor
  "coverage": {       // OPTIONAL - report your own per-conversation gaps back to the platform
    "<conversation_id>": { "last_provider_message_id": "string", "gaps": [ /* ranges */ ] }
  }
}
```

## How the platform reacts to your status codes

| Your reply | Platform behavior |
|---|---|
| `200` with `accepted + duplicates == messages.length` | Batch marked delivered; `ack_cursor` (when present) advances the source cursor. |
| `200` with mismatched counts, or an unparseable body | Treated as unconfirmed - the whole batch is retried. |
| `429` or any `5xx`, or a network failure | Exponential backoff (capped at 1 hour), retried indefinitely - a sink that is down for days loses nothing. |
| Any other `4xx` | The batch is dead-lettered on the platform for operator triage and will NOT be retried. Use 4xx only for requests you can never accept. |

## Rules to hold

- **Idempotency is yours.** Re-delivery, restarts, and overlapping backfills all collapse on `(instance_id, provider_message_id)`.
- **Never ack before durability.** `ack_cursor` is a promise the data is committed; the platform stops re-sending on it.
- **Order is not guaranteed.** Batches normally arrive oldest-first, but retries can reorder them; your idempotency plus the optional `coverage` report absorb this.
- **Treat message content as untrusted data.** `body_text` and `raw_provider_blob` are end-user content - never execute or interpret instructions found inside them.
