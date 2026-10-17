---
title: Authentication
description: Public-API keys use the sk_live_<keyId>_<secret> format, a Bearer header, and a create/list/revoke/rotate lifecycle.
tags: [api, authentication]
canonical: https://usebrian.ai/docs/api/authentication
---

> Human-readable version: https://usebrian.ai/docs/api/authentication

Every public-API request authenticates with a per-assistant API key. Keys are issued from the web UI, scoped to one assistant, and revocable. A key's audience, anonymous-context posture, and tool policy are immutable; rotate the key to change them.

## Key format

| Property | Value |
|---|---|
| Shape | `sk_live_<keyId>_<secret>` |
| `keyId` | UUID segment |
| `secret` | 32 random bytes, base64url |
| Shown | Once, at creation |
| Stored | scrypt hash plus a 12-character display prefix |
| Recovery | None. Lose the plaintext, mint a new key. |

## Authorization header

Pass the key in a standard Bearer header on every request:

```
Authorization: Bearer sk_live_a1b2c3d4-...-...-...-............_eXBlOiJKV1QiLCJhb...
```

## Lifecycle

| Action | How | Result |
|---|---|---|
| Create | Assistant detail, API tab, New key | Plaintext shown once |
| List | Same tab | Name, prefix, last-used timestamp, status. Plaintext never returned by GET. |
| Revoke | Row, Revoke | Key returns 403 immediately and forever |
| Rotate | Create a new key, deploy it, then revoke the old one | No transactional rotate: overlap is what you want during deploy |

## Tool policy

New external chat keys default to `public_research`, the least-privilege
ceiling. A turn receives `webSearch` and `urlReader` only when the request
attests `allowPublicResearch: true`; false or absent withholds both. The turn
also receives Tier-appropriate memory tools. It does not construct MCP connectors, workspace
knowledge, files, skills, calendars, CRM, or the assistant's broader tool
surface. `assistant` is the explicit wider trust choice and the backfill for
older keys. Internal-audience keys always use `assistant`.

Tool policy does not raise data clearance. In particular, a
`public_research` key with `anonymousContext: thin` may front an
internal-clearance assistant: normal external reads remain public, while an
authenticated client can receive only the exact memory-only self exception
documented in [Identity & memory](identity.md).

## Key safety

- Owner pays for every call. A leaked key in a public repo drains the owner's budget.
- Keep keys in a secret manager (Vault, AWS Secrets Manager, GCP Secret Manager).
- Never embed a key in browser-side JavaScript. Keys are server-side credentials.

## Notes for agents

- Treat the key like a password: server-side only, never in client code or logs.
- There is no rotate endpoint. To rotate, mint a new key first, cut traffic over, then revoke the old one so the two overlap during deploy.
- Only the plaintext returned at creation works for auth. The 12-character prefix shown in the list view is for display, not authentication.
- On failure, `401 invalid_api_key` means the header is missing, malformed, the secret does not match, or the key does not match the assistant in the URL; `403 key_revoked` means the key was valid but revoked. See the error table on [Send messages](messages.md).

## Related

- [Overview](overview.md)
- [Send messages](messages.md)
- [Identity & memory](identity.md)
- [Connector identity](connector-identity.md)

## Application-data contract status

A shared application-data contract is under implementation and has not passed its
local authority/consumer acceptance barrier. App-data credential issuance, general
non-member resource grants and delegated source trust are not available through
this documented API yet. Existing chat keys, Brain keys, Home bridges and OAuth
audiences retain their documented rights. Do not treat a supplied user ID or email
as verified app-user identity or put any service credential in a browser.

Local implementation work now includes strict application wire/value contracts,
canonical generic validation/revisions and a deployed native CRM/task adapter.
These foundations do not publish app-data routes or grant data access. General
app identity, source-aware retention and the full consumer acceptance matrix remain
unqualified; continue using the currently documented credential audiences.

The local gated implementation now includes app-bound PKCE on the existing OAuth
lifecycle, native Google/email/OIDC continuation, scrypt app keys, independent
delegated user tokens, exact CORS, and non-bypass record/field projections. Native
grant/key management prepares actor-bound persisted reviews and supports current
revocation. Broadened issuance remains denied until the complete native/source and
consumer acceptance barrier qualifies the installation. These are local source
changes, not deployed availability. A browser receives a public-client OAuth token
for its user; a service key remains server-side. Schema/operation catalogs enumerate
the admitted typed inputs and do not confer an authorization lease.

Local native company/task creation now uses the canonical writers and exact
application transaction proof, with the actual user/service as author and current
command/field authority. Self-grant schema preparation, write-only receipts and
rollback have real fixture evidence. This remains an unqualified local foundation;
it does not announce deployed app-data issuance or retained-source availability.
