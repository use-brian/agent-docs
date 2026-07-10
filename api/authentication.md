---
title: Authentication
description: Public-API keys use the sk_live_<keyId>_<secret> format, a Bearer header, and a create/list/revoke/rotate lifecycle.
tags: [api, authentication]
canonical: https://sidan.ai/docs/api/authentication
---

> Human-readable version: https://sidan.ai/docs/api/authentication

Every public-API request authenticates with a per-assistant API key. Keys are issued from the web UI, scoped to one assistant, and revocable.

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
