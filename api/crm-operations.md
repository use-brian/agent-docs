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

## Managed email account policy

Owner/admin member sessions can read or approve a policy at
`GET`/`POST /api/crm/:workspaceId/operations/mailbox-policies/:connectorInstanceId`.
POST requires `providerKey`, `expectedVersion` (0 for a new binding),
`confirmed:true`, `managed`, `purposeKeys`, and optional `templatePurposes`.
The canonical command is `save_managed_mailbox_policy`. CRM machine grants
cannot approve it. Stale versions conflict; no-op updates create no new audit.

Managed Gmail, IMAP/SMTP and AgentMail sends require `crmPurposeKey` and an
optional approved `crmTemplateKey`. Current consent/suppression checks cover
all To/Cc/Bcc recipients at dispatch; missing or ambiguous people block the
entire send. Omitting these fields does not bypass a managed account. Ordinary
unmanaged mail keeps its existing behavior. A provider timeout may mean it
accepted the email: verify delivery before retrying.

Managed provider-scheduled drafts are unavailable. Mutable AgentMail drafts
and implicit recipient replies return `managed_recipient_snapshot_required`;
use explicit recipients. Interactive incoming-email replies retain their
separate server-bound channel path. SMTP partial recipient acceptance requires
reconciliation; do not resend the whole envelope automatically.

Durable command and member/scoped-integration adapters are implemented behind
an explicit server delivery port, which normal boot has not enabled at this
checkpoint. Do not infer availability from the route catalog or a CRM write
grant: an unbound deployment returns `delivery_unavailable`. Assistant/MCP
exposure remains pending the admission barrier.

When enabled, `POST .../operations/deliveries` accepts a stable UUID
`deliveryId`, exact `connectorInstanceId`, `purposeKey`, optional `templateKey`,
`to`/`cc`/`bcc` arrays, `subject`, Markdown `body` and inline-base64
`attachments` (`filename`, `mime`, `contentBase64`). The prefixes are
`/api/crm/:workspaceId` for members and `/api/crm/integration` for scoped keys.
Do not supply workspace, actor or authority fields in the body. Bound the
entire envelope to 8 MiB, 1,000 total recipients, 20 attachments and 200,000
body characters. POST returns the ordinary command result with `record`,
`created` and `duplicate`; `GET .../operations/deliveries/:deliveryId` returns
`{receipt}`, without message content or private claim material.

A committed `dispatching` receipt precedes external sending. Exact replay
by the same principal returns the same receipt without a new attempt; changed
reuse or another actor reusing that id conflicts. `sent` means provider
acceptance only, with `confirmedAt:null` until separately verified. Refusals
after claiming are `blocked`, definite rejection is `failed`, and ambiguous
outcomes are `needs_reconciliation`. An expired abandoned claim is also
uncertain. Never automatically retry it under a fresh id. Erasure removes
content while retaining the original replay identity.

Scoped sending requires both `crm.delivery.dispatch` purpose/provider
selectors and an owner-approved exact mailbox binding. Owners/admins manage
the binding with `GET`/`POST` at
`.../operations/mailbox-policies/:connectorInstanceId/integration-grants/:credentialId`.
POST takes `expectedVersion` (0 initially), `confirmed:true` and `enabled`.
Configuration is human-only and checked against current membership. Revoking
an existing binding remains possible after credential expiry or disconnection.
Receipt GET requires independent `crm.delivery.read` selectors; dispatch alone
does not authorize unrelated receipt reads.

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
and selector vocabulary, current grants, and credential-derived `workspaceId`
and `credentialId` with `Cache-Control: no-store`. Verify `workspaceId` against
your intended destination before a configuration apply. Unknown routes and missing
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
  `duplicate: true` while its parents remain live; after retirement within the
  approved replay horizon it returns only `duplicate:true` and
  `outcome:"submission_retired"`. It does not repeat consent, tasks, audit, or
  events. Do not change the key to recreate a retired submission.
- Same key and changed request returns HTTP 409 with
  `error: "idempotency_conflict"`. Generate a new key only for a genuinely new
  submission.
- Racing identical retries converge on one logical submission.
- A non-2xx transactional failure leaves no partial contact, submission,
  consent, task, audit, or workflow event.

## Example backend adapter

This unverified-form example requires a `new_or_review` definition. It uses
reserved `example.com` data and keeps the intake credential on the server.
Trusted matching instead requires the backend proof contract below:

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

### Intake credential rotation and replay namespaces

Migration 503 adds immutable `replay_scope_id` and nullable
`rotated_from_credential_id` to intake credentials. Existing keys are backfilled
with their own id, preserving every existing `intake_key:<id>` receipt key.
Ordinary key creation starts a separate namespace. Owner/admin creation may
supply `rotateFromCredentialId` naming a credential in the same workspace,
including a revoked credential during recovery. The database derives the new
key's namespace from that parent; requests cannot supply a namespace. A parent
link is workspace-safe and becomes null if its credential is later deleted;
the inherited namespace remains immutable. Credential secret/grant checks and
expiry of replay receipts are independent of that namespace.

Rotation does not copy or widen grants: creation still carries an explicit
validated definition list. Every submission, including an exact replay, checks
the current key's active state and definition binding inside its transaction,
holding shared locks on both through commit so revocation serializes with an
in-flight submission. Only then does it claim `(workspace, inherited scope, definition, idempotency
key)` with the same canonical request hash. An exact committed replay returns
before applying the latest field schema or definition payload limit, so a schema
revision cannot invalidate already accepted bytes. Current credential/definition
authority and the route's hard payload bound still apply. New submissions must
pass the current schema and configured size limit before any CRM effect.
Identical retries across a chain of
replacements return the original ids without new contact, consent, task, audit
or outbox effects; changed input conflicts. An unrelated key cannot select a
prior namespace by label or body fields. New canonical enquiries use
`source_submission_id = crm:<receipt id>`, so their older workspace/source
uniqueness constraint cannot collapse independent namespaces. The backend key
remains in `crm_intake_idempotency.idempotency_key`; existing submission ids and
source ids are unchanged. A committed replay returns before creating an enquiry.
The winning submission retains its
original actor/credential attribution. Rotation does not extend retention or
restore erased content; the retired-receipt contract below governs deletion.

`POST .../operations/intake-credentials` retains its existing fields and accepts
optional `rotateFromCredentialId`. Creation returns the one-time new secret and
safe parent metadata. New display prefixes contain the full non-secret key id
to avoid the old four-hex-character prefix uniqueness collisions; existing
prefixes remain unchanged. Lists and privacy export include safe lineage metadata,
never secrets/hashes. The native credential control creates a replacement with
the original definition list and explains that the old key remains active until
explicitly revoked after backend cutover. It also allows recovery from a revoked
key. No existing credential is silently revoked by rotation.

### Trusted intake verification and occurrence time

Trusted identity policies require `definition.identityVerification` with
`{keyId, publicKey, maxAgeSeconds, acknowledged:true}`. `publicKey` is the
43-character canonical base64url Ed25519 public-key x coordinate; no private key
enters Brian. `maxAgeSeconds` is explicitly chosen between 1 and 86400 seconds,
with no default. Only a member-authenticated owner/admin may save a trusted
version and acknowledge that the backend verifies address control (email policy)
or authenticates the configured provider subject (external policy) before
signing. Machine, assistant, workflow and Home app actors cannot give that
acknowledgement. Ordinary `new_or_review` versions contain no verification key.

Configuration lives in the existing immutable-by-service version's
`schema_snapshot.identityVerification`; `created_by_user_id` and `created_at`
record its acknowledging member and time. The save transaction rechecks and
locks current owner/admin membership; admission holds the definition lock
through commit so version/deactivation changes serialize with submissions.
Catalog reads expose the safe acknowledgement fields.
Legacy trusted versions lack the configuration and refuse new submissions with
`409 conflict`, reason `identity_verification_unconfigured`, until an owner saves
a configured version. The same blocker applies if the recorded acknowledging
member is unavailable; a current owner/admin can save a newly acknowledged
version. Records and committed replay receipts remain available.
The native editor defaults to `new_or_review`, exposes trusted configuration and
explicit acknowledgement, and edits existing versions without dropping routing,
consent or follow-up settings. Key rotation creates a new definition version.

New trusted submissions require `identityProof`:
`{keyId, definitionVersion, verifiedAt, signature}`. The signature is a canonical
86-character base64url Ed25519 signature over UTF-8 `canonicalCrmRequest` of:

```json
{"protocol":"crm-intake-identity-v1","workspaceId":"uuid","definitionKey":"fixture","definitionVersion":1,"idempotencyKey":"opaque-key","requestHash":"64-hex","keyId":"backend_key","verifiedAt":"ISO-instant"}
```

`requestHash` is the existing canonical hash of
`{definitionKey,fields,externalIdentity: claim-or-null,submittedAt: supplied-or-null}`.
Thus proof binds the workspace, current definition version, backend submission
key, all field values, provider/subject and explicit occurrence time. The signer
must actually verify the mapped address or provider subject; a signature proves
that the configured backend attested this, not that Brian independently ran the
backend's challenge. Provider verification wiring remains a live operator gate.
The test fixture signs synthetic assertions only. The portable reference
adapter remains part of the pending assurance tooling.

The service verifies current key id/version, canonical encodings, signature and
`now-maxAgeSeconds <= verifiedAt <= now` before identity lookup or CRM effects.
`now` is server receipt time; no future clock allowance. Invalid/missing proof
returns `401 not_authorized` with reason `identity_verification_required` or
`identity_verification_invalid`. Legacy unconfigured status takes precedence.
Malformed wire shapes remain 400. A caller-supplied `verified` or private key is
rejected, including nested external-identity authority fields.

New caller-supplied `submittedAt` requires valid trusted proof and the same time
window; otherwise return `400 invalid_input`, reason
`occurrence_time_requires_verification` or `occurrence_time_out_of_window`.
Omitted time is server receipt time. Direct evidence commands retain their
explicit `occurredAt` contract, including historical CSV mappings described
below. Proof failure never changes existing
contact data, consent, tasks, audit or outbox and rolls back the pending receipt.
Migration 504 adds nullable bounded `identity_verification_evidence` to enquiries;
only successfully verified proof plus its business request hash is stored and
returned by authorized submission detail reads. No
raw identity or secret is added to audit/outbox. The existing enquiry export,
retention, erasure and workspace-flush coverage owns this column too.

Proof is admission metadata, excluded from the business request hash. Exact
committed replay still requires current credential/definition authority, then
returns before proof/key-version/age validation. It may carry a renewed proof or
no proof and cannot create effects. Changed business bytes still conflict; a
proof cannot be transplanted to a new idempotency key, workspace or definition.
This does not promise replay beyond the separately configured receipt horizon.

### Historical CSV evidence times

Optional `consentOccurredAt` and `suppressionOccurredAt` CSV targets carry each
source event's historical instant to the canonical command unchanged, preserving
offsets and up to six fractional digits. They use the same persisted ISO instant
validator as direct evidence commands; excess precision, a date-only,
timezone-free, invalid calendar or malformed value
is a dry-run row error. A timestamp without its complete consent/suppression
field group is also an error. Blank or unmapped times retain the existing
receipt-time behavior and request fingerprints. That compatibility fallback is
not evidence of a historical event time; a migration claiming historical
ordering must map and reconcile the actual source timestamps. Event ids remain
job/row-scoped, and delayed imported grants or releases cannot override later
withdrawals or suppressions. Import authority and confirmation remain required;
this mapping cannot confer trusted public-intake identity authority.

### Atomic execution contract

The server owns one transaction per bounded 50-row chunk. It locks the job
before admission, rechecks current member or integration authority, and keeps
that lock through chunk/job checkpoints. A concurrent resume fails with a
processing conflict; cancellation serializes on the same row and cannot return
success while a later chunk is admitted under the previous state. A disconnected
process releases its transaction, so recovery needs no five-minute stale lease.
File reads finish before borrowing the transaction connection; locked job
immutability and source/mapping hashes validate the prefetched bytes. Reads
performed inside the transaction (custom fields, attribution and identities)
reuse that connection, avoiding nested pool acquisition under a small pool.

Each row uses a savepoint. Canonical entity creation/updates, custom fields,
stable identity bindings, operations commands, audit/outbox and its completed
receipt share the caller-owned client. A row rejection rolls these effects back
before writing its failed receipt/error. Connection loss, serialization failure
or deadlock aborts the whole chunk instead of recording a business rejection.
Chunk counts and the job checkpoint commit together; committed legacy chunks
reconcile absolute progress when their former job checkpoint is missing.

The import composition constructs `CrmOperationsService` over the same
transaction client; the command store neither begins nor commits an outer
transaction it does not own. Existing standalone callers retain owned
transactions. `crm.ts` and `entities-store.ts` accept explicit transaction
clients while preserving caller access predicates and workspace checks.
Custom-field definitions/references use that client too. Graph relationships
remain best-effort projections of canonical CRM attributes: enqueue their
effects while writing, discard them on row/chunk rollback and invoke only after
commit. No ambient transaction interception or alternate SQL mutation engine.

Trusted import email matching uses the same normalized-email namespace lock as
intake, excludes inactive/archived records, and returns review conflict for
multiple matches. A stable-provider import holds the existing identity lock
before resolution; unavailable identity persistence fails the row instead of
silently creating an unbound record. None of these locks grants public-intake
verification or widens the captured source/job grant ceiling.

### Retired intake receipts and explicit replay policy

Migration `505_crm_intake_retired_receipts.sql` introduces versioned
`crm_privacy_policies` and extends intake receipts. The first policy domain is
`intakeReplay: { retentionSeconds: positive integer } | null`; absence/null is
unconfigured. The upper bound 2147483647 is a storage bound, not a recommended
period. No legal duration is selected by the product. Other privacy domains,
previews, suppression tombstones and restore journals remain subsequent work.

`GET /api/crm/:workspaceId/operations/privacy-policy` returns the current
version (0 and null when absent). Owner/admin
`POST .../operations/privacy-policy` submits `expectedVersion`, `confirmed:true`
and `intakeReplay` to `save_privacy_policy`. Only a current human member with
owner/admin role may approve it; neither assistants nor integration credentials
may self-approve. The canonical command locks/rechecks membership, serializes
policy versions, rejects stale versions, and treats identical saves as no-ops.
Policy versions are immutable during normal use; audit records contain version
and configuration state, not submissions. The native CRM settings surface uses
the same command and explicit confirmation. Policies are exported as settings
and preserved by workspace data flush; workspace deletion removes them.

A new receipt captures the then-current policy version and expiry measured
from its database `created_at`. Changing/removing a policy never shortens or
extends an already captured horizon. Legacy/unconfigured receipts acquire the
current explicitly approved period, also measured from their original creation,
when their parent is retired. If such a receipt has no approved period, erasure
or retention refuses with `conflict`, reason `intake_replay_policy_unconfigured`,
before deleting its evidence. Workspaces with no affected receipts need no
replay policy to erase unrelated records. This is a replay-specific prerequisite,
not completion of the programme's wider privacy/retention policy gates.

The existing contact purge hook and retention transaction retire receipts
*before* deleting enquiries/people. A retired receipt keeps its scoped opaque
key, request fingerprint, timestamps and policy version/expiry, and clears all
person/submission/task references. These are minimized pseudonymous processing
records, not claimed anonymous data; backends must use opaque source ids, not
addresses, for keys. Parent FKs refuse accidental deletion while live receipts
still refer to them. Full workspace flush deletes receipts before entities.
Both paths lock affected enquiries in id order before receipt locks, so a
concurrent contact purge cannot hold a receipt while retention holds its
enquiry. Retention locks the exact resolved/spam enquiries it will delete; it never
prunes a live committed receipt or failed/undelivered outbox event as housekeeping.

Current credential/definition authorization still precedes replay. An exact
non-expired retired replay is HTTP 200 with only
`{ duplicate:true, outcome:"submission_retired" }`; the canonical result has
that outcome, `created:false`, and no emitted events. No old ids or payload are
returned and no contact, consent, enquiry or task is recreated. Changed-body
reuse remains 409. Expired retired receipts can be forgotten by retention or
the next scoped claim, after which the key is a new submission and all current
validation applies. Live parents retain ordinary result replay even after that
minimum horizon. There is no perpetual post-erasure idempotency promise.

Actual PostgreSQL tests exercise policy authority/version races, missing-policy
rollback, parent FK refusal, contact and retention retirement, key rotation and
revocation, changed-body conflicts, expiry and concurrent retries, export/RLS
and flush classification in both schema compositions.

### Durable backend reference

The OSS `scripts/crm/durable-intake-queue.mjs` and
`scripts/crm/reference-intake-backend.mjs` provide an owner-private SQLite
queue and loopback fixture. They commit a receipt before queued acknowledgement,
share pacing and expiring leases across workers, preserve key/body across
restart or lost response, and honor Retry-After with bounded backoff. Permanent
4xx responses are not retried automatically. A configured replay deadline
pauses uncertain work before upstream receipts may be forgotten; qualify it
against the approved workspace policy. A `submission_retired` result is final.
No API secret is persisted, and successful/retired payloads are logically
cleared. Failed/pending payloads and local WAL/backups need the operator's
privacy and recovery policy. This reference does not prove identity ownership
or supply a production website. See the engineering CRM assurance specification
for its prerequisite and acceptance boundary.

The operator guide at `use-brian/docs/operations/intake-reference.md` describes the loopback CLI, private receipt inspection, explicit retry/cancel actions and deployment gates. Queue payload deletion is logical; the backend queue, WAL and backups need their own approved privacy/recovery policy. Fatal fixture worker failure stops accepting submissions and exits nonzero.

### Transaction-time integration admission

Generic CRM commands and commerce writes recheck a scoped key inside their
transaction, before module/domain locks. The workspace-bound credential and
grant rows remain locked through commit; current stored permission and the
original request/source ceiling must both allow the operation and resources.
Missing, revoked, expired or malformed authority returns HTTP 401
`credential_revoked`; insufficient live grants return HTTP 403
`integration_scope_denied`. Expiry is evaluated with database time after the
credential lock is acquired. A revocation that wins admission prevents the
write; an already admitted write can finish before revocation returns. Exact
commerce replay requires the same current authority. This does not recall
in-flight operations or replace import source ceilings.

## Read-only configuration discovery

Use `GET /api/crm/integration/operations/record-fields` and `/operations/pipelines`
with `crm.records.read`. Member JWT clients use the corresponding
`/api/crm/:workspaceId/operations/` paths. These return `fields` or `pipelines`
and `nextCursor`, accept the common page/time filters and `includeArchived=true`,
and otherwise select live configuration. Fields accept `entityKind` of person,
company or deal; pipelines accept deal and include their selected stages.
Traverse every page and treat a denied, malformed or incomplete catalog as a
failed preview. Neither endpoint seeds configuration or appends audit. Avoid
`/api/crm/:workspaceId/config` for manifest preview: that settings getter can
create the default pipeline. Discovery alone does not apply a manifest.

## Configuration commands

Member `POST /api/crm/:workspaceId/operations/commands` accepts
`create_record_field`, `update_record_field`, `set_record_field_archived`,
`create_pipeline`, `update_pipeline`, `create_pipeline_stage`, and
`update_pipeline_stage`. CRM keys use the existing scoped commands endpoint.
They need `crm.catalog.configure` with explicit `all` for all four catalog
dimensions: definition ids, purpose keys, plan ids and event ids. Reads still
need their own grants. Current member role or credential admission is checked
inside the transaction; a request-time permission snapshot is insufficient.

Create requires the business identity and configuration; updates require an
existing id and change only supplied fields. Field key/type/entity kind cannot
be changed. Duplicate creation returns 409 `conflict` with
`details.reason=configuration_exists`; read its current values before choosing
an update. Unique-name update collisions are 409 `configuration_conflict` in
details. Unchanged updates/archive states return `duplicate: true` and write
no configuration timestamps or audit. This is not a general request replay key.

Existing member settings and preset routes delegate to these commands and
retain their resource/ok responses. Field limit, used-option, live-deal archive
and default-pipeline barriers apply. Select another default before archiving
the current one; setting its `isDefault` to false alone fails. Restore an
archived field before editing it; stage restoration and edits can share one
command. No Association activation is needed for generic CRM configuration.
Each resource command is atomic. The manifest CLI applies resources independently,
with fresh discovery, partial recovery and final residual-diff verification.

## Scoped segment discovery

Member and scoped integration `GET .../operations/segments` share the existing
segment read store and query schema. Responses carry `segments`, `nextCursor`
and the complete predicate `catalog`; select each entity kind explicitly when
discovering all segments. Archived rows require `includeArchived=true`. The
integration adapter retains the store's workspace-wide segment authority:
`crm.records.read`, global `crm.catalog.read` (all four catalog dimensions),
global consent, entitlement and participation read grants. A partial grant
cannot expose a wider derived catalog. Discovery performs no configuration,
audience materialization or command execution.


## Manifest input and discovery foundation

The OSS version-1 manifest schema is `packages/core/src/crm/manifest.ts`.
It derives business validation from canonical commands, with local resource
references and a pipeline reference on each stage. The fictional full input
is `scripts/crm/fixtures/community-manifest.v1.json`. Sensitive intake fields
are submission-only; trusted identity setup, archive operations, authority,
credentials and module activation are outside version-1 manifest input.

`scripts/crm/manifest-client.mjs` supplies private token loading and pure
all-page discovery for member and integration modes. Build `@use-brian/core`
before importing it. Discovery checks the explicit workspace against the key's
catalog, refuses redirects and incomplete catalogs, and requests only required
resource/dependency catalogs. Segment discovery requires the global grants
listed above. The operator CLI is `scripts/crm/apply-manifest.mjs`; its default is a pure
JSON preview. Add `--apply` for canonical writes and review the report of
completed and failed references. An identical reapply issues zero commands.

## Manifest execution

The operator guide is `docs/operations/crm-manifest.md` in the OSS repository.
Run the CLI with explicit `--manifest`, `--api-url`, `--workspace`, `--mode`
and either `--token-env NAME` or `--token-file PATH`. Values of bearer tokens
are never command arguments. Preview is GET-only; `--apply` requires the
appropriate canonical write grants. Segment writes require `crm.records.write`
in addition to their derived-catalog read grants.

The planner shares the live segment vocabulary and validates projected fields,
purposes, plans, events, mappings and wording locales before mutation. Explicit
ids bind renames; natural keys prevent a partial rerun from creating duplicates.
Each command has its own transaction; final success requires zero residual drift.
A create conflict or uncertain response is accepted only when a re-read proves
equivalent current state. No blind mutation retry is issued. Other errors stop
and retain completed references in the JSON report. SIGINT/SIGTERM abort requests
and return a nonzero interrupted report when possible; re-read after any lost
response before assuming a change did or did not commit.


## Address suppression after erasure

The member privacy-policy command accepts optional `addressSuppression:
{ retentionSeconds } | null`. Omission preserves the current setting. Only a
current owner/admin human may approve policy or release retained suppression.
Erasure and workspace reset capture effective restrictions transactionally;
missing policy, key material or usable normalization blocks the transaction.
A reset preserves these restrictions even though it clears ordinary CRM rows.

Sendability may return `blocked` with `address_suppression` when a recreated
contact's address matches retained evidence. Imports and ordinary intake do not
release it. Missing or changed retained key material returns a fixed conflict category
instead of an allowed verdict. Key rotation cannot extend an already captured
retention horizon. Existing consent/channel checks still apply after release.

Owner/admin `GET /api/crm/:workspaceId/operations/address-suppression` returns
`tombstones` and `nextCursor`, without addresses or digests. Follow the cursor
for complete review. `POST .../address-suppression/:tombstoneId/release` accepts
`confirmed:true`, `evidenceKind` and `evidenceId`. A withdrawal needs
`evidenceKind:consent_event` naming a later human-recorded grant for the same
purpose/address, recorded after capture. Older, future-dated, imported,
withdrawn or wrong-address evidence fails.
Other reasons need `evidenceKind:workspace_file` naming a current file in this
workspace. The owner must review that evidence before confirming. Identical
release replay returns `duplicate:true` without another audit; changed evidence
for an already released row conflicts. These owner controls are unavailable to
integration/intake keys and do not grant permission to send.

Dedicated server HMAC keys and their retained versions are an operator custody
requirement. Retained rows are pseudonymous sensitive data. No production
policy period, key provisioning or operational privacy readiness is implied.

## Correction history after person erasure

Person hard purge minimizes the matching correction history and the new purge
receipt in its existing transaction. Free-text reasons, ticket references,
details and snapshots are not retained there as personal-data copies. The audit
keeps its action, subject/actor identifiers and time. A missing target fails
under a transaction lock; stale soft deletion also cannot append a new personal
audit copy after erasure. A failed deletion rolls the audit changes back. This
contract covers correction history, not a claim of complete workspace or
external-backup erasure.
