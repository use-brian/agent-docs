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


The local gated implementation now captures canonical generic/native read fields
and original relationship targets as metadata, then checks the current reader's
own app/member rights before model input or exact-version derivation. Catalog
aliases and another app's original grant do not transfer authority. Missing field
observations are withheld. Query/aggregate evidence, global native field policy,
all legacy consumers and broader retention profiles remain unfinished; no deployed
availability or qualified application-data issuance is announced by this checkpoint.


The gated native field-policy work also includes schema-bound policy algebra and a
current native-member metadata mask reader. Module-scoped roles are resolved from
the actual current member, with lifetime/revocation and record/execution floors.
These masks do not authorize source access, writes or application credential use.
App/issuer/configuration, query and legacy consumer acceptance still block global
activation; no additional production authentication audience is available.

The local gated metadata boundary also resolves nested source/scope helpers from
canonical schemas. Caller-created temporary rows cannot replace source floors or
mint field evidence. This does not activate application credentials or complete
the remaining field/query/legacy acceptance.


The gated canonical generic-record getter now applies current module field rules
for native members, member applications, external users and named services. App
and parent-grant issuers retain their own current field floor; an explicit release
profile is required for a non-member. Catalog aliases and calling modules do not
change canonical ownership. A role loss invalidates previously captured field
context and a result still awaiting admission. This does not qualify all queries,
writers, indexes, background or delivery consumers, and issuance remains disabled.

A generic projection's `fields` lists returned value keys. Optional `absentFields`
lists declared, currently readable cells whose absence was observed under the same
field evidence. Withheld cells appear in neither list. Absence is source-bound;
these names are metadata, never credentials or proof of permission.


The gated canonical query now checks predicate fields before filtering and sort
or aggregate fields across every matching contributor before cursor/limit.
Result evidence retains predicate-only nonmatches, contributors beyond the page,
returned values and readable absences. Actual query-only member, external-user
and service grants use the query action without borrowing read permission.
Managed system cells come from canonical columns after field admission; hidden
creation timestamps are not exposed by page cursors. Every retained observation
is rechecked under the current reader, with exact record versions checked before
return. Registered catalog and record metadata functions must remain private to
their canonical owner, including direct runtime-login grants.

Empty-domain/selector proof is still being implemented and currently refuses
when the original query domain cannot be established. This temporary hold is not
the final empty-query contract. The local query checkpoint does not certify the
remaining query, writer, legacy, background or retention consumers and does not
enable application-data issuance or announce deployed API availability.


Registered legacy native types now retain immutable query-domain evidence for
empty results and counts. Later readers must cover the original tier, compartment
and Project ceilings under their own current access. Readable General catalog
metadata does not cover a wider original domain. Empty aggregate field-role loss
withholds both model context and a retained memory for the affected reader; an
independently entitled reader remains usable. Native memory viewer searches,
indexes and counts apply that field ancestry in the non-owner transaction before
returning payloads. An unsupported system/share projection excludes field-backed
memories rather than treating its owner connection as a field grant.

Department-v2 and application-selector domain proofs, historical/system readers,
all remaining consumers and full qualification are still required. These local
checks do not enable issuance, protected retention profiles or deployed APIs.


The gated native query-domain contract also includes Department-v2 General and
Department tiers, required Department context, binding labels, shared Project
read membership or oversight, and the independent Project-ID ceiling. A later
reader needs its own current authority covering all original combinations.
Finite current result IDs do not prove that containment, and an owner has no
automatic Department access. This additive extension remains under local
acceptance; it does not enable application-data issuance or retention.


The gated query-only catalog path can enumerate admitted schema metadata even
when no business row exists. It intersects current and issuance-bound fields
through parent issuers and credential layers, including module roles and explicit
external/service releases. Catalog discovery grants no record or retained-result
rights. Original app query selectors and independent reader revalidation remain
under implementation; application-data activation is still unavailable.

An external app profile also retains the recipient's native limits when that
account is a workspace member. Its current clearance, Departments, Projects and
module field roles intersect with the explicit release. Joining the workspace
invalidates an existing invocation; a fresh request resolves the combined limits.
The app's profile label cannot bypass actual membership or confer membership on
a non-member.

The local selector-observation path now binds empty app queries and counts to
immutable original ID/self/relationship scope and predicate-field obligations.
A later app or native reader uses its own current authority. Relationship domains
also retain an independently readable canonical anchor, including an empty result;
revoking that anchor withholds the observation before model or memory use. Query
aliases do not change canonical resource ownership. This is still under acceptance:
finite admitted-point refinement, invocation configuration freezing and complete
consumer qualification remain required, and application-data activation stays off.

Local finite-query observations now separate admitted pure-ID points from absent
IDs. Later readers need their own query authority for the exact original points;
an unused wider grant ceiling is not a point permission. Absent IDs keep their
original domain obligations, and candidate/version changes during the read hold
the request. Ordinary native generic queries also retain scope evidence without
requiring a native module owner. Compound selector refinement and full consumer
qualification remain pending; these local changes do not enable application data.
