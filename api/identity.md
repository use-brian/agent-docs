---
title: Identity & memory
description: How the public API decides whether a visitor is anonymous (Tier 2) or remembered (Tier 1), and how email bridges a shadow user to a future OAuth signup.
tags: [api, identity]
canonical: https://usebrian.ai/docs/api/identity
---

> Human-readable version: https://usebrian.ai/docs/api/identity

How Use Brian decides whether a visitor is anonymous or remembered. The choice is yours: opt in explicitly when you have a stable user identity, opt out when visitors are ephemeral.

## Two tiers

### Tier 2: anonymous (default)

If you do not pass `identified` or `externalUserEmail`, the visitor is treated as ephemeral. The assistant has session conversation history but writes no memories about them. The shadow user is auto-pruned after 30 days of inactivity. This is the safe default for unauthenticated browser visitors.

### Tier 1: identified

Triggered when you pass `identified: true` OR `externalUserEmail`. The assistant gains the `saveMemory` and `getMemory` tools, and the per-turn retrieval layer surfaces this visitor's accumulated memories in the prompt. Memory is keyed to `(visitor, assistant)` and survives across sessions. Use this when you have a stable user identity in your system (logged-in users, wallet addresses, internal uuids).

## What email adds on top

Email is the only cross-provider identity bridge. If you pass `externalUserEmail` and the same human later signs up to Use Brian via Google OAuth with that email, their shadow user automatically promotes. They keep their memory across the API and direct Use Brian use. Without email, memory is durable but does not follow the human across services.

## Resolution table

| Request signals | Tier | Memory tools | OAuth auto-merge |
|---|---|---|---|
| Neither `identified` nor `externalUserEmail` | Tier 2 (anonymous) | No | No |
| `identified: true` (no email) | Tier 1 | Yes | No |
| `externalUserEmail` (with or without `identified`) | Tier 1 | Yes | Yes |

## Key audience

Beyond the per-request tier, every API key declares an **audience** at creation, immutably. One key is never both.

### External keys (default)

Serve your own end-customers. The tier table above applies, plus one key-level choice for the anonymous lane: **anonymous visitor context**.

- **Limited** (default): anonymous turns read only public-sensitivity knowledge - the behavior external keys always had.
- **Full assistant context**: anonymous turns get everything the assistant itself can read - team memory, files, brand, skills - read-only, exactly like a public chat link. The workspace owner opts into this at key creation; the assistant's clearance becomes the exposure ceiling for anyone holding the key. Identified (Tier 1) turns are unaffected: per-customer conversations stay on the limited context plus that customer's own memory.

### Internal keys

Act as a real workspace member - for internal tools, scripts, and automations, not customer traffic. Each request may name the member it acts as with `actorEmail`; omitted, the turn acts as the key's creator. The named actor must be a member of the assistant's workspace, or the request fails with `403 actor_not_member`. Internal turns get the member's full workspace context (their memory, team memory, files, brand) with reads capped at the lower of the member's and the assistant's clearance, and they write memory as that member. The external identity fields (`identified`, `claims`, `externalUserEmail`) return `400` on an internal key, and `actorEmail` returns `400` on an external key: attribution is a property of the key, never a per-request escalation.

## Knowledge base access

API requests honour the assistant's clearance, the same setting that gates knowledge-base reads on every other channel. To expose only public KB to third-party consumers, point the API key at an assistant whose clearance is set to public; for an internal-only integration, use an assistant with internal clearance. The visible setting on the assistant detail page is the single source of truth, the same for Tier 1 and Tier 2 visitors. External keys created with full anonymous context read at the assistant's clearance on anonymous turns, as described under [Key audience](#key-audience).

## Client accrual: what the workspace keeps

An identified (Tier 1) turn leaves two things in the workspace brain:

- **A contact record.** The visitor is materialized as a CRM contact (a person entity) the workspace team can see, paired to your `externalUserId` (and `externalUserEmail` when sent). This is how what each customer taught the brain becomes visible to the owning team.
- **Client-walled writes.** Everything the turn writes (memories, CRM rows, tasks, knowledge) is stamped into a per-client compartment derived from `externalUserId`. No client can read another client's rows, and a client cannot read even its own contact record: accrual is write-only from the client's side. Workspace members read it all normally.

Anonymous (Tier 2) turns accrue no contact, but anything they do cause to be written still lands behind the same per-client wall.

Two rules follow:

- **Pairing is identity, never authority.** The stored id/email pairing is bookkeeping. It is never read back to grant a later turn anything: claims are per-request, and a request without claims is anonymous regardless of history.
- **Drift is recorded, not fatal.** If the same `externalUserId` later arrives with a different attested email than the stored pairing, the turn still runs and the disagreement is logged for the owner. A mismatch *within* one request (between `claims.email` and `externalUserEmail`) is still rejected at the wire.

One carve-out: a turn whose attested email matches an existing Use Brian account resolves to that real member. A teammate arriving through your integration is treated as a teammate, not accrued as a client.

## Do not blanket Tier 1

Setting `identified: true` on every request is a budget footgun: random per-pageview ids would each become a Tier 1 user with consolidation cost. Pass `identified: true` only when you have a real, stable user identity. Anonymous browser sessions should default to Tier 2.

## Notes for agents

- Default is Tier 2. You must opt a visitor into memory explicitly; there is no implicit promotion from traffic alone.
- Pick your `externalUserId` to be durable per human (logged-in user id, wallet address, internal uuid). Memory is keyed to `(visitor, assistant)`, so a per-pageview id fragments memory and costs the owner.
- Pass `externalUserEmail` only when you actually have the person's email and want cross-service continuity. It is the one signal that survives a future OAuth signup and merges identities.
- Knowledge-base visibility is a property of the assistant you key against, not the request. Choose a public-clearance assistant for external embeds and an internal-clearance assistant for private integrations.
- Identified turns accrue a team-visible contact and compartment every write per client. Do not expect to read the accrued contact back on the end user's behalf: accrual is write-only from the client side, by design.

## Related

- [Overview](overview.md)
- [Authentication](authentication.md)
- [Send messages](messages.md)
- [Connector identity](connector-identity.md)
