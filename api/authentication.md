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


### Invocation construction and field/configuration changes (local R1)

Canonical native and application invocations pin current field/configuration
policy metadata before the first read or model session. A role gain, role loss,
timed assignment transition, schema/owner/contract or configuration revision change
invalidates that invocation, including its session lease. Restoring policy does
not revive a rejected invocation. Start a fresh request for current permissions;
never automatically retry an effect whose result may be uncertain. Business-value
writes still use separate source/version checks. Independent authorized readers
retain their own current access to canonical observations. Deployed external module
configuration requires a private metadata revision port; unverifiable required
revisions hold access. This bounded local checkpoint does not qualify application
data issuance, every consumer, production activation or the remaining R2-R6 work.


The complete local R1 SQL matrix now passes 316 cases across 14 files, including
37 invocation-epoch cases, with zero feature skips. Policy epochs distinguish
actual configuration from ordinary memory/source bookkeeping; a business edit
uses source/version admission, not a workspace-wide invocation reset. A current
result read before that edit is refused by the canonical result version check.
Retained historical observations keep their existing separate admission contract.
Local proof does not enable application-data issuance or satisfy R2-R6/rollout.


Local R2 query work additionally preserves finite ID/self/registered-relationship
intersections as canonical query points. An independently authorized later reader
uses its own current point, field and relationship-anchor rights; another reader's
revocation does not grant or remove those independent rights. Absent or excluded
IDs retain their original domain obligations. This remains unpublished source
work behind the existing qualification barrier, not available app-data issuance.


The local legacy generic getter/query implementation now shares canonical field
capture and the admitted candidate query engine with application operations.
Predicates run only after whole-candidate field admission, and empty/nonreturned
predicate evidence reaches the actual Brian query tool. Native and application
readers keep their existing independent ceilings. Canonical writer, governance,
all-consumer and final R2 acceptance work remains in progress; this does not change
credential availability or certify deployed behavior.


The local canonical generic writer now restricts actual inserted/changed native
cells at the database boundary, including old SQL writers. Legacy partial patches
preserve omitted hidden values; create/update responses are projected before the
transaction commits. Native field roles and each application/issuer/parent ceiling
remain mandatory. Whole-record application deletion requires its distinct exact
row-delete and named-command actions. A field write grant cannot confer deletion.
Reference captures use canonical file-owner metadata for independent native/app
readers. These are local implementation checkpoints; qualification and credential
issuance remain unavailable pending all-consumer acceptance.


Canonical task updates now have a local caller-owned transaction contract and
optional expected source version. Registered native commands can commit task
supersession, canonical links, receipt, audit and event atomically and reauthorize
replay after current audience changes. This is a reusable writer checkpoint;
complete native application operation catalogs, governance, downstream adoption
and final acceptance remain pending. Application-data issuance stays disabled.


Legacy generic records now return nullable managed timestamps/authors: a metadata
property carries a value only when a corresponding declared system cell is currently
readable; otherwise its value is null. The application record projection wire is
unchanged. Custom tables and rendered pages preserve private query/field evidence,
including empty queries, for subsequent current-access checks. This local reader
checkpoint does not qualify runtime or certify the later model/cache consumers.


Legacy generic schema discovery now advertises only field declarations currently
readable or writable through the canonical catalog. Native generic history returns
only currently readable fields and retains private evidence on its result wrapper.
Missing required private policy configuration, invalid revisions or public reader
ACLs hold direct legacy reads and writes as well as application invocations. These
floors do not grant row access, register handlers or enable runtime qualification.


Native record patches check every supplied field, including equal-value assignments.
A title-only patch preserves untouched fields without requesting write permission
for them. Native generic command replay rechecks its original write fields under
current native roles, row/scope and review state. Refused replay repeats no effects;
receipts contain references only. Deployed semantic commands can compose canonical
multi-record writers under one transaction with state/CAS and result checks.

Sandboxed Home frames may send Origin `null` only to active Home application
clients with the installed Home authentication adapter. Each actual request needs
the existing exact signed Home binding and current viewer/manifest authority. This
does not accept application OAuth/service keys or native session credentials for
opaque origins, and authentication/budget consumption occur once per request.
Preflight returns only client-mode metadata. CORS reflects the exact opaque origin
without wildcard or cookie credentials. `/api/applications/:appId/schemas/:type`
returns the current projected catalog resource. Schema mutation still requires
native governance. Qualification, binding capture and real issuance remain gated.


Protected generic schema preview and mutation require native governance and the
complete current catalog/record read/write and scope floors before inspecting
business values. Owner/admin status alone grants no field authority. Direct legacy
schema edits/deletion use the same canonical guard; the exact owner declaration
retires with an authorized parent schema deletion. Schema reviews bind the actor,
assistant, monotonic policy revision, schema and affected-record versions. A policy
change and restore requires fresh preview; applied recovery rechecks current
rights against its saved post-apply revision. Legacy reviews without this evidence
require fresh preview. Runtime JSON cannot register readers, handlers or owners.


Native-session schema management uses:
- POST `/api/applications/workspaces/:workspaceId/schemas/:schemaId/preview` with
  `{ expectedVersion, properties, assistantId? }`.
- POST `.../schema-reviews/:reviewId/apply` with `{ assistantId? }`.
- POST `.../schema-mcp` for native `applicationSchemaPreview` and
  `applicationSchemaApply`; optional assistantId query selection.

The existing native-session/origin guards and canonical native member/assistant
context factory apply. JSON supplies no actor or roles. Application OAuth/service/
Home credentials acquire no management audience. Preview counts require complete
current governance and independent field references, including person/files/
relation/managed-author targets. Counts are withheld after an awaited contraction.
Both data and management services reject direct/inherited owner connections.

Preview returns an actor/assistant/policy-bound review; apply returns
`{ entityTypeId, schemaVersion, applied }`. The review ID is the once-applied identity.
Schema apply changes its pinned epoch: MCP may report `grant_revoked` with
`effect: committed` and `retrySafe: false`. Reconcile the same review in a fresh
invocation; never automatically retry a write. SDK schema helpers consume explicit
deployment management ports without receiving native-session credentials in the
browser application transport. Qualification and real app issuance remain disabled.


Native module role/owner governance is a native-session management profile:
- GET `/api/applications/workspaces/:workspaceId/governance/catalog` with
  `moduleKey`, optional `afterId` and optional `assistantId` query values.
- POST `.../governance/preview` with `{ moduleKey, expectedRevision, operation,
  assistantId? }`.
- POST `.../governance/reviews/:reviewId/apply` with `{ assistantId? }`.
- POST `.../governance/mcp?moduleKey=...` with optional assistant selection.

Catalog returns a current revision, deployed role keys and at most 100 assignment
metadata rows with IDs/versions and `nextCursor`. Operations are strict:
`assign_role({userId,roleKey,startsAt,expiresAt})`,
`revoke_role({assignmentId,expectedVersion})`, and
`bind_owner({entityTypeId})`. Each operation object includes its `kind` discriminator.
The binding can attach an unowned generic schema to a deployed protected module;
it cannot remove/reassign an existing protected owner or upload code/readers.

Owner/admin membership alone gives no missing field role. Complete current native
schema/row/field/reference governance applies, and delegation needs the actor's
own live module role without exceeding its expiry. Preview is actor/assistant/
revision bound. Apply commits its mutation/review/content-free event once. Changed
or restored authority requires fresh review; replay still needs current authority.
An explicit self-revocation can return its committed status while further reads
and replay use the reduced authority. No business payload or old allowance is
cached in the outcome. Required private configuration ports remain mandatory.

Native MCP exposes `applicationGovernanceCatalog`, `applicationGovernancePreview`
and `applicationGovernanceApply`, with the selected module's actual role enum.
A committed apply changes its invocation epoch; reconcile the same review in a
fresh invocation after `grant_revoked` with `effect: committed`. App OAuth/service/
Home credentials acquire no native management audience. Qualification and app
key/grant issuance remain disabled; native module bindings and compiled external
configuration update handlers are still under implementation.


Protected generic record deletion retains every current native member/issuer and
module field-release floor for the actual deleted fields, in addition to the exact
application delete and named-command actions. A member using an external profile
keeps the member floor; a child cannot borrow its issuer's rights for a parent.
A field mask alone still grants no row-delete action. Ordinary unowned exact row-
delete capability retains its existing contract.

Deletion receipts contain refs only. A deployed protected module may retain its
current per-record policy after deletion and use the private canonical tombstone
metadata port for scope/version; no field values are retained there. Native recovery
checks current write rights to the recorded deleted keys. App recovery repeats the
current principal, issuer, parent and release checks. Missing retained policy or
unknown legacy deleted-key metadata holds protected recovery with access_unavailable.
Role restoration permits a fresh authorized recovery of the same receipt without
repeating deletion, audit or event. A tombstone never restores old field payloads
or authorizes reading them through historical observations. Qualification and app
issuance remain disabled.


Native entity-backed CRM updates keep the current deployed module field-write
mask alongside ordinary member/assistant/row bounds. Typed native company,
contact, deal, stage and custom-field writers authorize their original supplied
keys before reading and merging stored values. Equal hidden-value assignments
remain unavailable. The canonical SQL guard repeats current admission for actual
business changes; raw attribute replacement requires its complete supplied field
set, including equal values. Exact-record partial intent only restricts which
unchanged fields a semantic writer preserves; it never permits a hidden change.
Allowed partial edits preserve hidden untouched fields and canonical IDs/links.
Missing current protected metadata fails closed. This is field-write admission,
not full native operation/history qualification; issuance remains disabled.


Canonical native deal amounts accept nonnegative decimal strings with at most 18
fractional digits under the bounded application decimal grammar. Use strings for
exact values; unsafe integer numbers, exponent strings and invalid/negative values
refuse. Safe finite legacy numeric inputs remain supported. Strings survive typed
create/patch/get/list, REST/views/chat and CSV import without conversion to Number.
Application deal catalogs declare amount as decimal (scale 18), and projections
return decimal text even for JSON numeric storage. Native field observations use
the same text codec; source versions and policy epochs are unchanged by projection.
Existing browser amount inputs, ordering and totals preserve exact strings.
The additive shared applicationDecimalText/Add/Compare helpers use BigInt scaled
arithmetic and do not imply currency rounding. Other native aggregate/read/history
closure still belongs to R2; this checkpoint does not enable runtime qualification.


Typed native CRM partial edits merge their original semantic keys into canonical
JSONB in PostgreSQL. Untouched values stay intact even when a hidden stored JSON
number has more precision than JavaScript can represent. Current row/source/field
checks still govern the operation and every actual changed field. The pure merge
codec reads no database source and grants no permission. Raw full-attribute
replacement remains a full-field operation. Native deal decoders retain original
SQL decimal text when JSON numeric parsing would otherwise round an amount,
without changing untouched stored values into strings as a side effect of a patch.
Explicit nullable clears affect only the named key; canonical IDs/links and one
version increment remain intact. Full native consumer/history/operation governance
closure is still pending, and qualification/issuance remain disabled.


Finite/self/registered-relationship selector intersections apply to actual member
OAuth, members using external profiles, nonmember external OAuth and named service
credentials. Each current parent and the credential's immutable issuance maximum
remain restrictions. Fields may union only on the same candidate within one
layer; a broader live app grant cannot fill a field absent from its captured
credential/OAuth ceiling. The server derives issuance snapshots, and later writes
to those snapshots are refused. A still-valid credential may retain an unrelated
resource while a revoked parent makes the requested resource unavailable.
Current genuine-member, issuer and executing-assistant bounds remain enforced;
owner/client registration does not promote an external principal into a member.
Qualification and issuance remain disabled while native/persistent consumer
closure and final acceptance are incomplete.


Canonical entity create, patch and supersession reject undeclared root keys,
forged creation metadata, malformed business scalars/arrays/dates/source enums
and non-JSON or nonfinite attribute values as invalid_value before writes.
Validation preserves the original trusted derivation object; it never creates
source authority. Explicit null Department/Project destinations retain canonical
destination-admission errors rather than being treated as omitted selections.
These checks compose with current row/field/source/revision admission and do not
turn on application issuance or qualification.


Canonical archive/restore, catalog pipeline-stage changes, participant removal,
registered CRM stages and import enrichment carry exact original business-field
sets into the canonical entity writer. Partial SQL merges preserve untouched
hidden and precise values; clears and equal attempted fields still require
current write admission. Archive/pipeline field names match the public semantic
vocabulary. Registered nonowner stage commands resolve outbox IDs under the
existing read policy after the insert, using a workspace/event-key transaction
lock and current-policy existing-receipt read for idempotency. No owner data
read or relaxed RLS is used by the composed application path. Outbox failure
rolls back business rows and audits. Remaining native/HTTP/query/history closure
is still required, and qualification and real issuance remain disabled.


Canonical native CRM business values are constrained on the actual entity row,
below typed and raw writers: known text/tag/reference-ID shapes, deal stage and
bounded nonnegative amount/date values, current custom-field types/options and
required values. Unknown custom keys refuse; archived values can be retained
unchanged but not edited or removed. Existing curated person-profile text slots
organisation_name/position (200) and mailing_address (600) remain a finite native
vocabulary and obey those bounds. Configured definitions add restrictions.
Raw SQL reports 22023; application dispatch maps constraint refusal to invalid_value.
Unsafe JavaScript integer amount inputs refuse before serialization at raw native
create/patch/supersession; exact decimal strings and original SQL numeric text
retain precision. A type-specific patch check admits and locks its current source,
so a hidden source keeps its normal miss rather than revealing its kind.
These constraints do not grant row/field access or certify complete native relation,
read/query/history, creation-input and governance closure. Issuance/qualification
remain disabled pending the remaining R2 and downstream work.
