---
title: CRM Operations API
description: Submit atomic external intake with a least-privilege credential and use the shared CRM operations available through Brain MCP.
tags: [api, crm, intake, mcp]
canonical: https://usebrian.ai/docs/api/crm-operations
---

> Human-readable version: https://usebrian.ai/docs/api/crm-operations

Use Brian CRM operations provides one workspace-scoped command plane for typed
submissions, compliance evidence and sendability, shared segments,
entitlements, events and participation, deal pipelines, committed workflow
events, and resumable imports. The same operations back Brian chat, Brain MCP,
member REST, the first-party CRM UI, workflows, imports, and the legacy
Association API adapter.

## Choose the right credential

| Credential | Authority | Use it for |
|---|---|---|
| `sk_intake_*` | Write-only, bound to one workspace and one or more intake definitions | A public website's trusted backend submitting a form or application |
| `sk_brain_*` with `read` | Workspace-scoped Brain MCP reads only | An external agent reading CRM operational state |
| `sk_brain_*` with `read_write` | Workspace-scoped Brain MCP reads and permitted writes | An external agent operating CRM records |
| `sk_crm_*` | CRM-only operation grants with explicit resource selectors, expiry and revocation | A backend integrating CRM catalogs, submissions, entitlements, participation or Association commerce |
| Member JWT | Current user's workspace membership and role | The first-party UI or a member-authenticated integration |

Never put machine keys in browser code. `sk_intake_*` is deliberately not
accepted by Brain MCP or member routes. It cannot list contacts, read
submissions, search the brain, mutate configuration, or call arbitrary tools.

## Scoped CRM integration requests

Authenticate `/api/crm/integration/*` with `Authorization: Bearer sk_crm_<uuid>_<secret>`.
Workspace and actor come from that credential. Do not send `workspaceId`,
`actor`, or `authority` in command bodies. `GET /catalog` returns the operation
and selector vocabulary and the current grants. Unknown routes and missing
operations fail with 403 `integration_scope_denied`; invalid, expired or
revoked credentials return 401. CRM keys do not authenticate Brain MCP, intake,
member/admin routes, or public chat.

`POST /operations/commands` uses the canonical typed command union. Resource
aliases accept business fields at `/operations/intake-definitions`,
`/consent-purposes`, `/entitlement-plans`, `/events`, `/submissions`,
`/entitlements`, and `/participation`. Catalog saves require
`crm.catalog.configure`; reading a catalog uses `crm.catalog.read`, not an
implied write grant. Definition, plan and event selectors are UUIDs; purpose
and provider selectors are stable keys. An omitted selector permits none.
Creating a catalog resource requires explicit `all` for its dimension.
Integration configuration cannot acknowledge trusted identity sources, issue
credentials, enable modules or erase subjects.

Corresponding read resources return the existing named arrays. Event, plan,
submission, entitlement and participation reads filter by granted resources
before applying their page limit. `/association/*` provides the shared
Association command adapter and checks every event referenced by an order.
A mixed-event order is available only if every event is permitted.

An owner/admin member issues keys through
`/api/crm/:workspaceId/operations/integration-credentials`. POST accepts label,
future expiry, grants, and optional `revokeCredentialId` for atomic rotation.
The plaintext `oneTimeSecret` appears only in that creation response. GET
returns safe metadata; POST `/:id/revoke` revokes a key for the next request.


## Traverse complete collections

CRM operations collections keep their named arrays and add `nextCursor`.
Use `limit` (1-100, default 50), optional `cursor`, `createdAfter` (inclusive)
and `createdBefore` (exclusive). Follow each returned cursor with unchanged
workspace, resource and filters until it is null; changing page size is allowed.
An old cursor used for another query returns `invalid_input`. Ordering is by
immutable creation time/id, with PostgreSQL microseconds retained and an upper
bound from the first page. This is a traversal contract, not a snapshot across
record edits/deletes or backdated inserts. Current grants are rechecked.

This applies to intake definitions/credentials, purposes, submissions, plans,
entitlements, events, participation, pipelines, CRM records/custom-field
definitions, import jobs, integration credentials, and audit/event-delivery
history. Scoped `/operations/audit` and `/operations/event-delivery` require
`crm.audit.read`; outbound message receipt reads use `crm.delivery.read`.
Never infer a complete count from one page.


## Scoped record reads and CSV imports

`GET /api/crm/integration/operations/records` requires `crm.records.read`.
Use `kind=person|company|deal`, optional `query`, `includeArchived=true`,
`limit` (1-100), and the returned `nextCursor`. `/:id` reads one CRM record;
`/record-fields` enumerates custom-field definitions. These grant no general
Brain or workspace Files access.

Upload immutable CSV bytes to `POST /api/crm/integration/operations/import-sources`
with `Content-Type: text/csv` and a stable UUID `Idempotency-Key` (maximum 30 MiB).
Keep the returned `sourceId`; equal upload replay returns it, changed bytes
conflict. `POST /operations/imports/dry-run` takes `sourceId`,
`entityKind` (`contact|company|deal|operations`) and `mapping.columns` from
column indexes to enumerated import targets. `POST /operations/imports` adds
`confirmed: true` and the exact `dryRunHash`. Confirm creates a job; POST
`/operations/imports/:id/resume` processes one bounded chunk. Read progress with
GET `/operations/imports/:id`, list jobs with GET `/operations/imports`, cancel
with POST `/operations/imports/:id/cancel`, and download failures at GET
`/operations/imports/:id/errors.csv`.

Imports require `crm.imports.write` plus each mapped domain write operation.
The import selectors and domain selectors both constrain purpose/plan/event
references. Source and job authority is immutable: a replacement key must cover
the original grants, and a wider key cannot widen an old source or job. Read
inspection requires corresponding read grants. Use equal-grant key rotation to
resume existing jobs. `stagedFileId` is for member sessions only; a CRM key
cannot read arbitrary workspace files or legacy file jobs. Machine imports
cannot nominate a trusted identity source. The dry run rejects scope violations
before creating contacts or compliance evidence.


## Atomic intake endpoint

```text
POST https://api.usebrian.ai/api/crm/intake/:definitionKey/submissions
Authorization: Bearer sk_intake_<credential-id>_<secret>
Idempotency-Key: <opaque retry key>
Content-Type: application/json
```

The workspace, allowed definition, identity policy, CRM/custom-field mappings,
consent wording, queue, owner, and follow-up task template all come from the
credential and versioned definition. Do not send them in the body. The body
contains only the definition's declared field keys and end-user data.

Successful response:

```json
{
  "submissionId": "9d55ba47-1c6f-4f71-949a-75133aedfe98",
  "contactId": "44b4f2b7-fdd8-4fce-8d80-50e283ba6489",
  "followUpTaskId": "4f111555-ca74-4d82-aaac-fcb551a32eb0",
  "duplicate": false
}
```

The response does not say whether a contact already existed and contains no
CRM fields, consent history, or segment membership.

## Idempotency

The `Idempotency-Key` header is required and scoped to the credential and
definition. Retry an uncertain request with the same key and byte-equivalent
JSON body.

- Same key and same canonical request returns the original committed ids with
  `duplicate: true`; it does not repeat consent, tasks, audit, or events.
- Same key and changed request returns HTTP 409 with
  `error: "idempotency_conflict"`. Generate a new key only for a genuinely new
  submission.
- Racing identical retries converge on one logical submission.
- A non-2xx transactional failure leaves no partial contact, submission,
  consent, task, audit, or workflow event.

## Example backend adapter

This example uses reserved `example.com` data and keeps the intake credential on
the server:

```ts
export async function submitApplication(form: {
  name: string;
  email: string;
  acceptedUpdates: boolean;
  idempotencyKey: string;
}) {
  const response = await fetch(
    "https://api.usebrian.ai/api/crm/intake/community-application/submissions",
    {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.USE_BRIAN_INTAKE_KEY}`,
        "content-type": "application/json",
        "idempotency-key": form.idempotencyKey,
      },
      body: JSON.stringify({
        fields: {
          full_name: form.name,
          email: form.email,
          updates_opt_in: form.acceptedUpdates,
        },
      }),
    },
  );

  if (!response.ok) {
    throw new Error(`CRM intake failed with HTTP ${response.status}`);
  }
  return response.json();
}
```

Generate (for example with `randomUUID`) and persist the idempotency key with
the local form attempt before the first call. Reuse it for network/time-out
retries. Do not generate a fresh key inside each retry loop.

## Brain MCP tools

Discover with `tools/list`; do not hardcode availability. Both key scopes can
receive these reads:

- `listCrmIntakeDefinitions`, `listCrmSubmissions`, `getCrmSubmission`
- `listCrmConsentPurposes`, `getCrmConsent`, `checkCrmSendability`
- `listCrmSegments`, `previewCrmSegment`
- `listCrmEntitlementPlans`, `listCrmEntitlements`
- `listCrmEvents`, `listCrmParticipation`, `listCrmPipelines`

`read_write` may additionally receive:

- `recordCrmSubmission`, `updateCrmSubmission`
- `recordCrmConsent`, `recordCrmSuppression`
- `saveCrmSegment`, `archiveCrmSegment`
- `grantCrmEntitlement`, `updateCrmEntitlement`
- `recordCrmParticipation`, `updateCrmParticipation`
- `setDealPipelineStage`

Call the catalog/list tool first and pass stable ids or keys from its result.
Unknown fields, purposes, definitions, plans, events, pipelines, and stages fail
closed and return bounded valid choices. Do not guess labels from prose.

`checkCrmSendability` returns `allowed`, `blocked`, or `unknown` with reasons and
effective evidence ids. Treat only `allowed` as permission. This API does not
send campaigns or override provider restrictions.

## CRM workflow event source

Committed CRM operations can start workflows through the id-less `crm` event
source. Event type uses `match.inChannels`; stable definition, purpose, plan,
event, pipeline, or stage keys use `match.tags`:

```json
{
  "kind": "event",
  "event": {
    "sources": [
      {
        "source": { "type": "crm" },
        "match": {
          "inChannels": ["crm.submission.received"],
          "tags": ["website_contact"]
        }
      }
    ]
  }
}
```

The closed event types are `crm.submission.received`,
`crm.submission.updated`, `crm.consent.changed`,
`crm.suppression.changed`, `crm.entitlement.changed`,
`crm.participation.changed`, and `crm.deal.stage_changed`. Enumerate the
workspace filter catalog before selecting a stable key; never guess one from a
display label. Events carry stable record pointers and classifications, not
personal field values. A workflow step that needs detail reads it with its own
authorized CRM tool. Assistant/system-authored operations require
`match.fromBots: true`.

## Credential lifecycle

Workspace owners/admins create, list, rotate, and revoke intake credentials in
CRM settings or member REST. A secret is shown once and only its hash is stored.
Rotation creates a new secret and explicitly revokes the old credential. No API
reveals an existing secret.

## Member import and operational audit

A member-authenticated integration can stage a workspace file and use the
server import job. Production import has a mandatory write-free dry run and a
separate confirmed commit:

```text
POST /api/crm/:workspaceId/operations/imports/dry-run
POST /api/crm/:workspaceId/operations/imports
GET  /api/crm/:workspaceId/operations/imports
GET  /api/crm/:workspaceId/operations/imports/:jobId
POST /api/crm/:workspaceId/operations/imports/:jobId/resume
POST /api/crm/:workspaceId/operations/imports/:jobId/cancel
GET  /api/crm/:workspaceId/operations/imports/:jobId/errors.csv
```

The server reads the complete staged file, limited to 30 MB and 100,000 data
rows. Confirmation must provide the exact dry-run hash and immutable mapping.
Jobs advance in replay-safe 50-row chunks and may map contacts, companies,
deals, external identities, consent, suppression, entitlements, and
participation. Verified-email matching requires an explicitly selected trusted
source and admin authority. Always inspect the dry run and obtain approval
before committing real data. Custom values are parsed against their live field
types during dry run. Replays use deterministic evidence identities and exact
repeated stage writes are no-ops, so a crash before the row receipt does not
duplicate consent, suppression, audit, or workflow events.

Members can inspect bounded execution evidence with:

```text
GET /api/crm/:workspaceId/operations/audit
GET /api/crm/:workspaceId/operations/event-delivery
```

Owners/admins may request a complete operations privacy export at
`GET /api/crm/:workspaceId/operations/privacy-export`. Intake credential secret
hashes are excluded. The confirmed retention endpoint requires a caller-chosen
cutoff; Use Brian does not invent a legal retention period for the workspace.

## Association compatibility

`/api/association` remains available for existing integrations. Generic
enquiries, consent, membership/entitlement, events, and participation map to the
same canonical records and service. Association-specific ticket inventory,
orders, and provider reconciliation retain their commerce rules. New generic
integrations should use CRM operations contracts and CRM contact ids.

## Related

- [Brain MCP](../mcp/brain-mcp.md)
- [Association operations](association-operations.md)
- [CRM concepts](../concepts/crm.md)

### Segment and compliance completeness

Segment lists keep `segments` and `catalog`, and return `nextCursor`. Catalog
choices are complete, including fields/options and workflow event stable keys.
`GET .../operations/segments/:segmentId/preview` accepts the common page inputs
plus `snapshotCursor` and `snapshotLimit` (1-10,000; default 1,000). Its existing
`rows`, `count`, and `snapshotIds` gain `nextCursor` for rows and
`snapshotNextCursor` for IDs. Follow each stream to null with the same segment
and filters. An edited segment invalidates either cursor; begin a new traversal.
The count is current dynamic membership, not a frozen database snapshot or
continuing permission to send. The native SDK completes the ID stream before
presenting a snapshot. Contact compliance reads return all authorized purposes,
consent and suppression evidence, including histories beyond 500 events.


### Effective entitlement filters

`GET .../operations/entitlements` accepts `activeOnly=true|false` and optional
ISO `effectiveAt`. Rows keep raw `status` and add `isEffective` plus the resolved
evaluation time. Active status alone is insufficient: access starts inclusively
and ends exclusively; no end means no expiry. Default evaluation uses database
time retained across cursor pages. Keep filters unchanged while continuing.
A historical read grants no present-day commerce authority. Segment plan
status `active` means effective access; raw-active periods outside the window
have derived value `inactive`, with raw stored status preserved.


### Identity review conflicts

Trusted email resolution normalizes whitespace/case without provider-specific
dot or plus rules. Multiple live matches return HTTP 409 `conflict`, with
`reason: identity_review_required`; no third contact, submission or replay
receipt is committed. Resolve the ambiguous CRM records before retrying.
An external-subject binding to an archived/retracted/superseded person also
requires review. Concurrent lookup/create/bind for one workspace/identity is
serialized. This is independent of verification authority and does not make a
claimed address in a `new_or_review` submission a merge instruction.

## Effective evidence ordering

Consent and suppression use occurrence time, recording time, then stable id,
all descending. A delayed old grant cannot override a newer withdrawal merely
because Brian received it later. The shared evaluator preserves PostgreSQL
microseconds and equivalent timezone offsets; invalid timestamps fail closed.
Read queries select the latest consent and latest suppression per channel by
the same ordering, keeping evaluation bounded. A purpose's nonempty channel
list restricts it to those channels; empty or legacy absent lists mean all.
A mismatch returns blocked with `purpose_channel_inapplicable`, including for
purposes configured without required consent. Releasing suppression does not
undo withdrawal or suppression in another channel scope.

## Intake rate-limit retry contract

The initial limit remains 60 attempts per 60-second sliding window, keyed by
credential candidate and resolved source address within each API process.
Duplicates/rejected requests consume the window. HTTP 429 includes
`Retry-After: 60` and `{error:"rate_limited",retryable:true,retryAfterSeconds:60}`.
This is a conservative delay, not a remaining-quota or durable-queue claim.
Retain the same body/idempotency key and use aggregate pacing plus bounded
backoff/jitter; do not automatically retry changed-body 409s or permanent 4xxs.
Source IP uses Express's configured proxy-trust resolution, then the socket,
never a raw forwarded-header fallback. Shared boot leaves proxy trust disabled.
Operators must verify their actual trusted proxy boundary; arbitrary forwarded
prefixes cannot choose a bucket outside that boundary. Durable backend receipts
and multi-instance aggregate pacing remain separate integration obligations.

### Provider evidence replay

Consent and suppression provider ids each identify one workspace-scoped event.
Migration 501 adds a nullable SHA-256 request fingerprint to both existing
streams. New writes bind the normalized business request: contact, purpose or
channel/reason, action, source, metadata, and explicit occurrence time (UTC,
six-digit precision) or an omitted-time marker. Actor credentials and the
server's receipt time are not request identity. Canonical consent also excludes
the current purpose wording, so an identical retry returns its original snapshot
after a wording edit or archive. It still needs current write/resource authority.
The legacy consent API's explicit wording version is part of its request; moving
the same provider id between unequal legacy/canonical envelopes conflicts.

The database's provider unique index serializes concurrent inserts. Both the
early replay lookup and a losing insert compare the fingerprint before returning
the original event. Different payload reuse returns `409 idempotency_conflict`
with no new audit/outbox effects. Exact retries add no effects. Streams remain
separate; a consent id is not a suppression id.

Pre-upgrade events retain a null fingerprint because original request bytes
cannot be reconstructed honestly. To replay one, callers must supply its exact
stored occurrence time and matching persisted business fields (including legacy
wording version where applicable). Missing time requires review and returns
`idempotency_conflict` with `legacy_evidence_requires_occurred_at`; changed
evidence conflicts. Never invent a new event id merely to bypass a conflict.
The shared persistence helper is
`use-brian/packages/api/src/crm-operations/evidence-replay.ts`.
Fingerprints follow their event's existing RLS/export/purge/flush classification;
they are personal-data-derived integrity metadata, not anonymized data.

### Immutable wording and locale resolution

Migration 502 adds `crm_consent_purpose_versions`, unique by workspace, purpose
and bounded string version. A version freezes default wording/hash, an optional
default locale, and optional locale wording/hash maps. Supported locale keys are
the shared app catalog (`en`, `zh`, `zh-CN`, `ja`), also consumed by OSS app i18n.
No locale is guessed for existing combined-text wording: its default locale is
null. Missing requested translations fall back to that stored default and the
event records the resolved locale (possibly null), not a falsely claimed
translation. If a locale map includes the declared default locale, that text
must equal the default wording.

The purpose keeps its existing active-version/default-wording fields and gains
`defaultLocale`, `localeWordings`, and server-computed `localeWordingHashes`.
Its database trigger inserts or selects the immutable version on every write;
different wording/default locale/locale maps under an existing version fails
with a conflict. Label, description, channel policy and archive edits may reuse
unchanged wording. A deferred composite FK binds the active purpose to its own
version, and version rows cannot be updated. Purpose deletion still cascades
through version rows for approved privacy/flush operations. Configuration uses
the existing owner/admin command authority; version rows have member-read and
owner/admin-write RLS with the existing system bypass convention.

Backfill preserves the current purpose snapshot. Historical event versions are
added only when their stored text/hash pair is unambiguous. An old event whose
same version label carries conflicting or missing wording stays unlinked;
its original snapshot is retained and never relabelled as verified history.
Consent events gain `wordingVersionId` and `wordingLocale` with workspace/purpose
FKs; exact stored snapshots remain available in compliance and privacy exports.
Versions are included in workspace privacy export and cleared before purposes
in workspace flush. Subject export later selects only versions referenced by
that subject's evidence (§ privacy assurance).

`save_consent_purpose` accepts optional `defaultLocale` and `localeWordings`
(locale to text); hashes and immutable ids are always server-owned. Omitted
locale settings preserve the current settings for an update, while explicit
null/default and an empty map clear them only under a new wording version.
`record_consent` accepts an optional enumerated `locale`, which is part of
provider replay identity when supplied; omitting it preserves migration 501's
request fingerprint. Consent-answer mappings may choose a fixed `locale` or
`localeFieldKey` (mutually exclusive); a locale field must be a required text
field with nonempty options drawn entirely from the shared locale catalog.
Neither route bodies nor mapped fields may supply authoritative text/hashes.
The member and canonical consent schemas reject those unknown authority fields.

Compatibility `/api/association/consents` retains its explicit `wordingVersion`.
For a catalogued purpose, it resolves that immutable version (including an
optional locale), saves server-owned text/hash/id, and rejects unknown versions
or archived purposes. Uncatalogued legacy purposes remain accepted without a
locale and retain null text/hash/version references, never fabricated evidence.
Exact provider replays still precede catalog validation and return the original
snapshot, including after archival. A legacy null-fingerprint replay cannot
assert a locale that its stored evidence does not establish.

The native purpose editor supports creating/selecting a purpose, inspecting its
current wording, and saving a new version with optional translations and an
explicit default. Consent actions expose locale selection with the stored
default option. Intake configuration exposes consent mappings and their locale
binding alongside the existing field schema. All use the canonical commands
and the four locale dictionaries. No Association enablement/grant changes are
part of wording configuration.
