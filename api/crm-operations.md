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

Shared boot binds durable sending for member/scoped-integration requests and
native tools. An explicitly unbound custom deployment returns
`delivery_unavailable`. Live mailbox configuration and transport qualification
remain operator responsibilities.

`POST .../operations/deliveries` accepts a stable UUID
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

## Streamed CRM privacy exports

The member workspace route `GET /api/crm/:workspaceId/operations/privacy-export`
keeps the existing JSON response when `format` is absent or
`crm-operations-privacy-v1`. Use `?format=crm-privacy-v2` for the NDJSON export.
`GET /api/crm/:workspaceId/operations/contacts/:contactId/privacy-export`
defaults to v2. Both require a current owner/admin session. An existing person
id is required for the subject route.

Scoped integration equivalents are
`GET /api/crm/integration/operations/privacy-export` and
`GET /api/crm/integration/operations/contacts/:contactId/privacy-export`.
These accept v2 only, derive workspace from the credential, and require the
current `crm.privacy.export` grant. This grant is independently required even
when the key can read ordinary CRM records. Revoked credentials fail admission;
Association being disabled does not prevent privacy exports.

The response is `application/x-ndjson` with `Cache-Control: no-store`. Consume it
as UTF-8 lines, preserving each record line's exact bytes and trailing newline:

1. A `type:header` line identifies `schema:crm-privacy-v2`, `exportId`,
   `workspaceId`, `scope`, optional subject `contactId`, and `snapshotAt`.
2. Every `type:record` line carries `domain` and `record`. JSON integers may
   exceed JavaScript's safe integer range; use a lossless reader for amounts.
3. The final `type:manifest` line must match the export id, carry `complete:true`,
   and verify aggregate `totalRecords`/`sha256` plus every domain's
   `count`/`sha256`. Hash only complete record lines, including their newlines,
   in emitted order. An empty domain hashes the empty byte string.

Verify the final manifest before accepting the download as evidence. An HTTP
200 without that manifest is incomplete. An interrupted export can be restarted
safely as a new snapshot; do not splice lines from two exports together.
Database failures after headers close the stream without a success manifest.
An oversized projected row fails with `privacy_export_row_too_large` rather
than being truncated. Each export uses a repeatable-read database snapshot and
bounded cursor fetches, with no record-count cap.

The manifest names included, redacted and excluded domains/columns. The CRM
slice covers canonical CRM/custom fields, attributed links/tasks/activities,
identity and correction history, intake/consent/suppression, entitlement and
commerce records, drafts/delivery evidence, import lineage, and relevant
configuration metadata. Shared financial context remains while other
attendees' identities are redacted. Shared messages redact content,
attachments and other recipients; single-subject message content remains.
Raw shared import sources and unpartitioned file content require separate
review. Failed/legacy import rows without explicit subject links and incidental
free-text mentions are declared coverage limits, not inferred matches.
Retained suppression appears without HMACs or secret key checks; missing
required retained key material fails the subject export rather than implying
no suppression exists.

This is a CRM slice, not a database backup or whole-brain export. Manifest links
to other export facilities require their own authority; the workspace reset
link is explicitly marked destructive and is not an export. Export completion
does not imply erasure, retention policy, restore-journal or operational
assurance completion.

## Privacy operations and concurrent writes

Canonical entity purge and CRM retention now acquire brief exclusive write
admission for the workspace. An active writer makes the privacy operation
refuse with `conflict`, reason `privacy_operation_busy`, before it mutates
records. Retry the operation after that writer finishes. Conversely, a
concurrent write to a guarded CRM table can be refused while privacy holds
admission; the database reports SQLSTATE `55P03` with the fixed message
`crm_privacy_operation_busy`. Transaction rollback leaves that write unapplied.
Ordinary reads and writes in other workspaces remain independent.

The mailbox admission transaction holds shared privacy admission through the
provider call and receipt commit. A competing privacy operation refuses the
send before provider invocation, including raw managed mailbox sends.

This guard closes the write race needed by erasure/retention previews. It
does not approve erasure, select retention periods, or prove complete copied
data and recovery-journal coverage. Those remain separate requirements.

## Owner-reviewed CRM contact erasure

Use `POST /api/crm/:workspaceId/operations/privacy/erasure-preview` with
`{contactId}` to create a review. The response includes `id`, `previewHash`,
`expiresAt`, `policyVersion`, domain actions/counts, `blockers`, `scopeLimits`
and `status` (`ready` or `blocked`). A preview lasts 15 minutes; this is an
approval window, not a retention period. It stores no copied content.

A current owner/admin human member must review the result, then call
`POST /api/crm/:workspaceId/operations/privacy/erase` with `contactId`,
`previewId`, `previewHash` and `confirmed:true`. The same creating member
must execute it. Integration keys, assistants and system jobs cannot
approve this operation. Never treat the hash as bearer authority.

Execution revalidates the affected row versions and privacy policy under
exclusive workspace write admission, then invokes the existing canonical
purge. A stale, expired, mismatched or blocked preview returns a conflict
(`privacy_preview_stale`, `privacy_preview_expired`,
`privacy_preview_mismatch` or `privacy_preview_blocked`). Obtain and review
a new preview when its inputs have changed. Consumption and purge commit
together; an exact replay by the same still-authorized member returns the
content-free receipt with `duplicate:true`, without another deletion.

Known copied-data domains that the current purge cannot clear are explicit
blockers, including shared drafts/tasks, exact subject references in saved
segments, source files and workflow/decision payloads. Shared notifications
remain dependencies; eligible notification copies retire as described below. Financial
references can also block deletion. Do not bypass these blockers or call
a blocked preview successful erasure. They remain work in the full CRM
assurance programme. The response status `crm_contact_purged` and its
`scopeLimits` describe a CRM contact purge, not whole-brain deletion, a
legal certificate, or proof that backup copies and restore journals have
been handled. Legacy null-workspace CRM history is attributed through its
current canonical parent; new non-null snapshots cannot recreate a purged
CRM parent.

CRM history before-images and free-text mutation reasons are now redacted
inside the canonical contact purge. The preview reports `brain_row_versions`
as `redact`; minimized version receipts retain their workspace and erasure
stamp, including legacy rows whose workspace was previously null. Other
subjects and non-CRM primitives are unchanged. Indirect Association audit
metadata is also redacted before submission, membership or attendee references
are removed. Editing an attributable audit row invalidates an earlier review.
These redactions roll back with a failed purge and do not complete the other
copied-data or recovery-journal requirements above.

## Draft and task copies in privacy review

Contact previews and exports include whole attributable draft families: a
matching current or historical To/Cc/Bcc address selects the parent, every
revision and session anchors. Erasure deletes an eligible unshared family.
A different/null recipient or an address also bound to another contact returns
`shared_or_ambiguous_draft`. Each exported projection/revision filters recipient
arrays and redacts content if it lacks a subject recipient or has shared or
ambiguous ownership; an independent earlier subject-only revision keeps its
own content. Historical attachment ids/paths remain file dependencies.

Task copies include CRM-attributed roots, descendants and connected
supersession chains. Unshared sets and incident links are deleted atomically;
shared task components return `shared_or_unresolved_task`, with their content
and correction/sidecar payloads redacted in subject exports. A removed intake
submission can still be owned through the generated task's exact retained
contact attribute; unresolved or conflicting ownership blocks. A malformed
cross-workspace cascade returns `cross_workspace_task_dependency`. Workspace
CRM exports traverse the same complete task closure, excluding unrelated tasks.

Task snapshots and all four entity-alias histories are minimized before
canonical deletion. New task before-images require a live same-workspace
parent. Shared segment predicates are dependencies identified through exact
typed relationship UUID or complete base email values, including nested
arrays/groups. They return `crm_copy_resolution_required`; configuration is
preserved for owner resolution, with names/keys/predicate content redacted in
subject exports. Incidental id text does not match. Decision source/artifact
references use exact typed ids too. New copies invalidate a reviewed hash.

The same checks protect legacy canonical purge. No copy deletion commits if a
later dependency refuses erasure. Resolve retained dependencies and request a
fresh preview; do not bypass a blocked result. This closes draft/task copy
handling, not the remaining source, workflow/decision, financial, retention or
restore-journal requirements of the CRM assurance programme.

## Retired notifications and CRM workflow sources

CRM event delivery and Association notification rows can have terminal status
`retired`, with `retiredAt` and `retiredFromStatus`. They remain evidence of the
previous state, not deliverable work. Contact erasure clears attributed payloads,
recipient/source references, provider identifiers and free-text errors before
removing the source records. A prior `sending` notification has an uncertain
external outcome; retirement does not prove recall or provider erasure. Shared
notifications addressed to another contact return
`shared_notification_dependency` and require resolution before erasure.

The CRM worker leases ids and attempt numbers, then reloads a current unexpired
lease under privacy admission immediately before dispatch. Obsolete attempts
make zero dispatch calls; failure uses a fixed content-free message and bounded
retry. Admission lasts through strict workflow enqueue and the delivery receipt.
Retired rows cannot be rewritten or requeued. Delivered-only retention leaves
retired evidence intact until its explicit retention policy is implemented.

CRM-triggered workflow runs retain a same-workspace source binding even if their
mutable input changes. A manual input cannot assert CRM provenance. New runs
cannot use missing/retired events, including from stale transaction snapshots.
Migration fails legacy non-terminal CRM runs with unavailable sources using
`crm_privacy_source_unavailable`; it does not fabricate completed work. Bound
sources cannot be pruned while run dependencies exist. Run/step lineage is now
included in CRM privacy exports, with content redacted in both scopes. Eligible
local copies can retire as described below; active and artifact-bearing runs
remain explicit dependencies. The full privacy programme is not complete.

Legacy delivered-event retention skips workflow-referenced events and preserves
their replay keys while pruning other eligible events in the same transaction.

A legacy CRM run with an unavailable source cannot remove its remaining
attribution by changing its trigger kind or typed input. Its retained copies
continue to appear as review dependencies.

## Workflow outcome copies in CRM erasure

Privacy export and previews follow durable outcome-copy edges, including
resumed runs that consumed multiple previous outcomes. Run/step content remains
redacted in exports. A ready erasure preview can retire never-started runs and
completed local-branch runs, clear their stored content, and delete step copies
in the canonical contact-purge transaction. The retained receipt cannot resume
or supply an outcome to a later run. Replay identity remains reserved.

Blockers now name active/claimed execution, artifact-producing steps and
attached approvals/wake-ups/blueprints, pre-migration lineage uncertainty, and
shared input/source dependencies. A copied outcome does not authorize deleting
another person's input. Creating another consumer invalidates an older preview.
Workflow writes may briefly return a privacy-busy conflict while an erasure
validates and commits. Missing or cross-workspace typed copy sources are refused.
Late workflow audit writes are minimized after retirement; they cannot restore
the prior details. Broader artifact and legacy resolution remain required work.

Client-authored workflow replay keys produce `workflow_replay_dependency`.
Automatic retirement preserves only absent keys or the server-generated CRM
source-event key; a client key may itself contain personal content.

## CRM import source retirement

Owner/admin policy approval accepts optional
`importSourceErasure: { receiptRetentionSeconds, heldSourceIds } | null` through
the existing member privacy-policy command. Omission preserves the prior setting;
null leaves source retirement unconfigured. Durations have no default. Holds are
at most 250 distinct current-workspace source UUIDs and protect source bytes and
job history. Machine credentials cannot approve this policy.

A contact erasure can retire CRM-owned CSV bytes only when complete source
lineage proves that every consumer job and every successful row belongs to the
subject. Byte-identical staged copies are included even without their own row
receipt; an unprocessed matching copy is a named dependency. Legacy or pruned
consumer history, shared/unknown rows, incomplete jobs, failed rows and holds
return named preview blockers. Adding a consumer or changing
policy invalidates an earlier approval. General workspace Files still require
separate copy resolution.

Retirement clears CSV bytes, fingerprints, captured grants, job mappings and
row/chunk/error copies. Source-key receipts retain their explicit policy expiry.
While a receipt exists, source reads and same-key staging return `409` with
`reason: import_source_retired`; never retry by changing the key to recreate an
erased source. Completed import receipts expose `privacyErased` and
`privacyErasedAt`, and resume returns that terminal result without executing rows.
Members retain receipt visibility; a machine can read an erased job receipt only
with the original still-authorized credential and the required import operation.

A parser holding old bytes cannot commit work after retirement. Database parent
admission also rejects late consumers and stale snapshots. Unknown live-source
history blocks contact erasure throughout the workspace until resolved; absent
attribution cannot exclude a subject. Housekeeping preserves all jobs backed by
live sources and held source jobs, and does not remove source replay identity
before its approved expiry. This contract does not prove deletion of external Files, backups or
unattributed free text.

## Reviewed CRM retention

Owner/admin member policy approval supports nullable `retention` with explicit
positive age values (seconds), or null to leave a domain unconfigured:
`resolvedSubmissionsSeconds`, `importReceiptsSeconds`, `deliveryReceiptsSeconds`,
`auditSeconds`, and `financialRecordsSeconds`. `openSubmissions` is null or
`{ afterSeconds, fields }`, with distinct fields from `subject`, `message`,
`metadata`, and `notes`. Open-field redaction preserves submission status and
contact identity. `scheduled` must be explicit; `intervalSeconds` is 60–86400.
No age or legal basis is selected automatically.

`holds` accepts at most 500 distinct `{ domain, id }` pairs for `contact`,
`submission`, `order`, or `file`, validated in the current workspace. Holds
also apply to canonical contact erasure and the legacy retention endpoint.
Unchanged policy approval is a no-op; omission preserves the current retention
policy and explicit null disables it. Import source holds remain in
`importSourceErasure`.

- `POST /api/crm/:workspaceId/operations/retention/dry-run`: `{ before }`.
  Returns owner-bound id/hash, policy version, frozen cutoffs, actions/counts,
  retained dependencies, a 15-minute expiry and `hasMore` for additional eligible
  records beyond the bounded 500-record mutation batch per domain.
- `POST /api/crm/:workspaceId/operations/retention/execute`:
  `{ previewId, previewHash, confirmed: true }`. Rechecks current owner/admin
  membership, policy and affected row versions. Stale/expired/blocked review
  returns a conflict without mutation; request another review. Repeating a
  completed execution returns the original receipt with `duplicate:true`.
- `GET /api/crm/:workspaceId/operations/retention/runs`: common CRM pagination
  with `runs` and `nextCursor`, showing content-minimized results and failures.

These are human administration routes; integration credentials and assistants
cannot approve or execute retention. The same selector/mutator serves the
opt-in worker, governed by `runWorkers` and `CRM_RETENTION_ENABLED` (false/0
stops scheduling). It rechecks the latest owner-approved policy and due time
under workspace privacy admission. Two workers cannot apply the same due run.

Reports distinguish erased fields from retained task/import/delivery copies,
live source history, ambiguous sends, failed/undelivered events and financial
or audit dependencies. Eligible terminal submissions retire replay receipts
and minimize their audit details before deletion. Preview time also bounds
receipt expiry; elapsed review time cannot silently widen deletion. CRM export
includes safe run metadata and excludes approval hashes. Workspace reset clears
runs but preserves policy. A retention receipt is neither whole-contact erasure
nor an off-instance recovery journal.

## Reviewed staged-file cleanup

Owner/admin member sessions can review and retire eligible workspace Files used
by completed CRM imports. These are human privacy administration commands;
integration and Brain keys cannot approve them.

- `POST /api/crm/:workspaceId/operations/privacy/file-cleanup-preview`:
  `{ fileId, before }`. Returns `id`, `previewHash`, `expiresAt`, `policyVersion`,
  domain counts and named blockers. The cutoff cannot be in the future.
- `POST /api/crm/:workspaceId/operations/privacy/file-cleanup-execute`:
  `{ previewId, previewHash, confirmed:true }`. Rejects changed data, policy,
  owner membership, expired or blocked reviews without committing partial work.
- `GET /api/crm/:workspaceId/operations/privacy/file-cleanups/:id`:
  current cleanup receipt. Private storage locators and approval/lease tokens
  never enter the returned receipt or CRM export.

The current privacy policy must configure
`importSourceErasure.receiptRetentionSeconds`. Every file consumer must be a
terminal import older than the cutoff. File/contact holds, independent or
foreign-workspace references, ingested files, read-only local-directory files
and missing lineage block cleanup. Shared files are retained dependencies.

A successful execution returns `queued`, not completed byte erasure. It retires
the proven database copies and queues the exact object atomically. Workers use
the Files resolver; states are `queued`, `leased`, `failed`, `completed`.
Failed calls expose only `file_cleanup_failed`, retry with backoff and recover
expired leases after restart. Duplicate execution reuses the same receipt;
read its current state instead of authorizing another cleanup. Pending/failed
cleanup blocks contact erasure throughout the workspace even after its index
row has gone. Workspace reset preserves this work. Completion clears the
locator; approved retention expires content-free receipts after their horizon.

Completion proves live-object removal acknowledged by storage. Provider
soft-delete, versions, backups and account/workspace teardown remain separate
storage recovery and erasure operations, not implied by this receipt.

## Automatic manual entitlement expiry

Due manual grants now transition from active to expired through the generic CRM
worker even when the Association module is disabled. The worker scans all due
rows in bounded pages, locks/rechecks each grant at database time and emits one
committed lifecycle event. A concurrent extension that wins the row lock is
respected. Repeated scans, restart and two workers do not duplicate the change.

Provider-managed, indefinite, future and terminal grants are untouched. Effective
access still uses inclusive start/exclusive end and no grace period, so an
outage cannot extend a provider-managed grant's access. Provider reconciliation
remains responsible for its raw lifecycle state. Deployments may pause manual
expiry with `CRM_ENTITLEMENT_EXPIRY_ENABLED=false` or `0`; this does not change
the effective-access predicate.

The internal due-expiry command is reserved for its system principal. External
agents use the existing authorized grant/adjustment commands; they cannot invoke
an expiry worker identity or mutate terminal grants back to active.


### Historical participation imports

`record_participation` retains unconstrained manual/form/workflow participation.
For a ticketed or capacity-limited event, new live participation returns a conflict
with reason `association_order_required`; use Association checkout. A historical
import requires `sourceKind: "import"`, `historicalImport: true`, a current human
workspace owner/admin, and an event that has already ended. CSV mapping supports
`participationHistoricalImport` with literal `true` or `false`. An integration key
or workflow cannot claim human historical-import authority. Future/current event
imports cannot use this exception.

Historical records return `historicalImport: true`, preserve immutable source
identity and never consume present inventory. Replaying a historical import still
requires current admin authority. Generic participation updates cannot turn such
a record into a commerce reservation. Shared CRM remains available with the
Association module disabled.


### Provider entitlement renewal

Provider-backed grants and updates require a backend credential, matching trusted
provider actor, or the entitlement reconciliation job. Humans and assistants can
manage manual grants; they cannot manufacture provider-backed access. Scoped
integrations need both `crm.entitlements.write` for the plan and
`association.provider_events.write` for the provider. Revocation is rechecked on
replay as well as new writes.

Explicit provider periods use `providerPeriodId` and an optional `predecessorId`,
with a finite `endsAt`. One provider object/period identifies one immutable grant
request, including across different transport idempotency keys. Changed period
payload returns `idempotency_conflict`. Extend active/pending grants through the
existing update command. After a terminal grant, create a new period naming its
same-workspace/contact/plan/provider predecessor, with a later start. A stale,
foreign or active predecessor returns `provider_period_predecessor_invalid`.
Terminal grants remain terminal and a predecessor can have only one successor.
Legacy grants without period identities retain their original replay behavior.

`renewalMode: "none"` with an active status and future end represents cancellation
at period end. `status: "cancelled"` removes effective access immediately. Reads
return period/predecessor lineage separately from raw status and effective access.
Lineage is not proof of webhook verification or notification delivery.

## Generic catalog tools

saveCrmEntitlementPlan takes {plan}; saveCrmEvent takes {event}. Read the existing
plan/event catalog first, then use the declared canonical fields and stable
key/slug. Both tools require the owner/admin-granted configure capability plus
CRM app/write permissions, including direct MCP calls. They retain assistant
or credential identity and remain usable when Association is disabled. They
cannot approve privacy/identity policy, create credentials or enable modules.


## Native managed delivery tools

`sendCrmMessage` accepts the same delivery fields as the POST body above, without
`kind`, workspace or authority fields. `getCrmDelivery` accepts `{deliveryId}`
and returns `{receipt}`. The native send returns the canonical command result
(`record`, `created`, `duplicate`), preserving the stable delivery UUID.

Discovery and direct invocation require `crm` plus `home_app:crm:write` for
send, or `home_app:crm:read` for receipt inspection. Brain MCP read credentials
never expose dispatch. The credential's bound primary assistant must retain
its grants. The current credential, workspace membership when applicable,
connector exposure and turn context, per-account send action and blocked policy
are checked again at the provider boundary. IMAP and AgentMail require exact
instance grants; AgentMail also requires its active assigned email channel.
A legacy Gmail provider grant covers only the current primary mailbox.

This command does not grant mailbox access or enable an Association module.
Native chat/workflow sending requests approval; an explicit programmatic
read/write request is its own write authorization. All recipients must pass the
managed purpose policy at dispatch. `sent` proves provider acceptance only.
Inspect `needs_reconciliation` under the original UUID; a fresh UUID may send
a duplicate. No message is resent by receipt reads or exact replay.

## Protected recovery evidence

CRM privacy-v2 declares `crm_erasure_journal` as protected recovery evidence.
Workspace exports include existence metadata only; primary keys and recorded
mutation values are excluded. The schema registry has no workspace content and
is excluded. Integration credentials cannot edit recovery evidence.

Completed hard purges, retention, staged-file cleanup and workspace reset record
transactional recovery effects from migration 522 onward. Restoring an older
backup requires a current encrypted journal checkpoint before sending resumes.
This is an operator recovery procedure, not a CRM command or a replacement for
provider/storage erasure. Earlier coverage, journal custody and retention need
explicit privacy/deployment qualification.

## Department scope for operation history

Audit and event-delivery history require verified current member authority.
Each returned record must satisfy both its saved source audience and the current
source audience. Assistant/workflow contexts additionally need a complete bound
execution scope. `crm.audit.read` alone is insufficient: credential-only callers
receive `not_authorized` with Department access guidance until persisted machine
ceilings are supported. This applies to both member and integration endpoints.

Audit subject capture covers canonical brain records and persisted CRM contact
relationships, including typed source references in audit details. New envelopes
cannot be supplied or widened by callers. Legacy and unresolved audit/event
families remain excluded from strict mode pending classification and review.
Canonical privacy erasure leaves only a terminal administrative receipt; it does
not release protected source content. Security envelopes are not exported as CRM
content. These changes do not activate strict classification for a workspace.


### Erasure subject departmental authority (2026-10-06)

Owner/admin status does not grant access to a protected contact. Contact erasure preview creation must prove the caller's current canonical department grant before inspecting counts and again before returning the preview. Migration 676 saves the subject's inherited protection floor on the preview, without retaining its name, address or a new source-ID list. Execution checks both that saved floor and the live subject before destructive review. A consumed receipt checks the saved floor against current authority even though the contact has been erased; losing the department edge denies replay. A legacy preview without a saved floor fails closed under department v2. This implements the subject boundary only: floors of additional linked records, workspace-wide retention/export and file-cleanup authority still require their own complete coverage.

### Erasure linked Association protection (2026-10-06)

Contact erasure reviews also check every attributed Association order, membership, sponsorship allocation/invitation, offline rescue and registration selected by the canonical privacy coverage predicates. Each record must satisfy its saved scope, live source and parent authority checks before any domain counts are returned. The preview scope combines the subject and these record/source floors, so declassifying the contact does not weaken historical operational protection. Execution rechecks the linked records, and consumed receipt replay retains their combined floor after those rows are erased. Unclassified legacy operational rows fail closed under v2. A saved preview whose floor omits any currently required linked-record scope is stale and cannot be consumed; an older subject-only preview cannot produce a less-protected receipt. This does not yet certify the other CRM, import, workflow, file, audit or outbound record families.

### Read-only erasure review renewal (2026-10-06)

The member endpoint `GET /api/crm/:workspaceId/operations/privacy/erasure-previews/:previewId` reads an existing erasure review without creating a new preview or executing a purge. Only the original reviewing owner/admin can read it. Current owner/admin membership and the saved combined department floor are mandatory on every read. An unconsumed review additionally checks current subject and linked-record authority; a changed floor makes the review stale. A consumed review returns only its minimized receipt after checking the saved floor, without requiring the deleted subject or exposing historical domain counts. Responses are non-cacheable. This is the canonical renewal/recovery path for the staff UI; mutation confirmations and exact-preview execution remain unchanged. The erasure staff UI now renews this read through the shared bounded protected projection; denied or stale reads evict its review/receipt display. Retention and file-cleanup reviews require separate coverage.

### Contact privacy export department admission (2026-10-06)

Contact-scoped privacy-v2 exports apply the same subject and attributed Association operational authority as erasure reviews before emitting a header. The shared `privacy-subject-authority.ts` helper preserves saved order, membership, sponsorship, rescue and registration floors plus live source protection. Source read locks last until the snapshot finishes, so reclassification cannot invalidate that source snapshot mid-export. Department grants are renewed outside the repeatable-read snapshot before each yielded record and before the success manifest; revoked access aborts without a complete manifest. A credential's export operation alone does not establish a department identity: unbound integration callers fail closed for contact exports under department v2, while explicitly legacy workspaces retain their old grant contract. Existing domain projections, redactions and checksums remain unchanged. This covers contact/Association authority only; workspace-wide exports, legacy JSON export and the independent floors of other CRM/import/workflow/file/outbound families remain outstanding.


Contact export coverage inventory explicitly excludes source-scope/authority snapshots and scheduler execution bindings; these are authorization evidence rather than CRM content. Website catalogue and site-content draft/revision tables are explicitly excluded from this CRM slice: their typed website documents have no canonical CRM subject attribution and require the website content facility. Each excluded table and column remains declared in the coverage manifest, so schema changes cannot silently escape review.

### Workspace privacy export root protection (2026-10-06)

Workspace privacy-v2 export is a complete declared slice, not a silently filtered list. Before its header, a paged preflight checks every included CRM person/company/deal scope, every Association order/registration/membership/rescue/sponsorship allocation/invitation, and every saved erasure-review scope. Current canonical department grants apply even to workspace owners. An inaccessible record refuses the whole bundle; erased subjects cannot weaken surviving receipt protection. Historical CRM root rows retain their stored protection, while unresolved held scopes fail closed. The combined floor is renewed outside the export snapshot before records and the final manifest, as for contact exports. Unbound integration callers fail closed in department-v2 workspaces; legacy workspaces keep their existing grant contract. This extends root and Association coverage; independent activity/import/workflow/file/outbound floors and legacy JSON exports remain required before complete export sign-off.


### Legacy JSON export root protection (2026-10-06)

The legacy operations JSON export accepts a validated actor context, checks current owner/admin membership, and reads its tables in one repeatable-read transaction. It uses the same full-workspace CRM/Association/saved-review preflight as privacy-v2 and renews the combined department floor outside the snapshot before returning its buffered response. Denial returns no partial bundle. Its existing JSON schema and projections remain compatible; responses are non-cacheable. This closes the legacy bypass for those roots, not the independent activity/import/workflow/file/outbound scope gaps tracked above.


### Privacy activity protection (2026-10-06)

Department-v2 privacy exports and erasure reviews preserve each included CRM activity's saved audience independently of its current contact/company/deal. The canonical coverage predicates select activity pages; the gate locks each selected row, requires captured and non-held evidence, and checks both the immutable activity floor and current source. A source reclassification cannot lower historical activity protection. Contact reviews include these floors in the immutable preview and consumed receipt; a previously created review with a weaker floor becomes stale. Workspace and contact exports renew the combined floor before delivery. Legacy JSON does not include activity records, but conservatively shares the complete CRM preflight. Legacy/unresolved activity evidence is denied under v2 until explicit recovery exists. Workflow/import/file/outbound families still need their own coverage.


### Department-v2 activity and current-member parity

Activity RLS applies the canonical current department grant to both saved history and live CRM source in v2 workspaces. Department-specific clearance can exceed the member's base clearance; base clearance never substitutes for a missing or weaker department edge. Current assistant limits, context department, credential bindings, private visibility and mutation restrictions remain effective. Projects are organizational context, not a v2 read boundary. Legacy workspaces retain their prior activity predicate. The shared current-member source SQL helper follows the same v2 member floor for its CRM and other canonical source consumers, while retaining the legacy branch only for explicitly non-v2 workspaces. Missing current membership always denies. The app-role SQL path uses the boolean `department_member_source_allows` adapter; the underlying per-actor grant-map function remains private.


### Event privacy scope survives retirement

CRM event receipts retain a minimal immutable `privacy_scope` containing only the resource audience, with no source identity, name, payload or version. Migration 678 derives it from captured source evidence for existing non-retired events and new events. Canonical retirement clears the source pointer as before but preserves this audience. Under v2, exports and erasure review authority check the saved event audience plus its current source until retirement; retired receipts require the saved audience without the deleted source. Missing legacy evidence fails closed rather than becoming General. The ordinary retired-event read path applies the same saved floor. Explicitly legacy workspaces keep their existing minimized-receipt rule. Source-less historical recovery, workflow lineage beyond these events and other privacy families remain separate requirements.


### Retired CRM event receipt audience

Canonical event retirement retains a minimal immutable `privacy_scope` audience and clears source identity as before. In department-v2 workspaces the retired receipt read path checks this saved floor, including current assistant and binding limits; workspace ownership is not a department override. Old retired events without evidence fail closed under v2. Non-retired events still require saved and current source authority. CRM exports and erasure reviews share this floor; see `features/crm-operations.md`, "Event privacy scope survives retirement". This does not certify every workflow's independent definition, execution or copy-lineage authority.

### CRM event protection through workflow copy ancestry

Privacy authority uses the canonical workflow copy set, then walks its immutable source-run ancestry with cycle-safe traversal. Every directly attached CRM event and every event attached through a source goal contributes its saved audience and current-source restrictions to export and erasure-review authority. This includes another contact's event copied into a selected consumer. The combined review floor and stream renewal preserve those restrictions after the exported contact's direct sources change. Retired events use their minimal immutable audience. This closes CRM-event ancestry coverage; independent workflow authoring/context, non-CRM inputs and legacy unrecorded lineage remain separate audit requirements.

### Workflow context and blueprint privacy floors

Privacy export and erasure-review authority covers every run in the canonical selected copy set and its upstream ancestry. It includes the immutable run department/project context and current workflow definition context, all attributed workflow/research blueprint records, and the saved blueprint envelope on each copy receipt. Blueprint records are locked and checked for held or unavailable scope. A copied envelope keeps its original audience after source declassification; the live source may add protection. Unknown historical capture or a missing live blueprint fails closed rather than omitting a protected contribution. Scans use bounded pages.

These floors participate in contact, workspace and legacy export preflight, stream authority renewal, stale-review detection and immutable erasure receipt scope. They add to CRM event ancestry protection. Independent import/file/audit/outbound protection and trusted integration binding remain separate requirements.

### Canonical copy inventory privacy authority

The privacy gate checks each selected task, entity link and workspace file against its own canonical scope, independently of the CRM contact or company. Selection uses the same coverage predicates as export, after canonical copy attribution is prepared. Reads use bounded pages and hold source locks for the enclosing transaction. Held or missing source metadata refuses the operation; redacting file names or payloads does not waive authority for the remaining inventory. The combined floor applies to export admission and renewal and to saved erasure reviews/receipts. This includes import staging files and draft attachments selected by the canonical coverage inventory. It does not yet establish independent raw import-source, audit or outbound-envelope classification.

### File cleanup department admission and saved receipts

Workspace owner/admin status is necessary but does not override the source file's department. Preview checks the canonical file and each attributed import consumer entity before inspecting counts. Migration 682 saves their combined minimal audience on the immutable cleanup review, without retaining a new source-ID list. Execution and unqueued review reads check the saved floor and current file/consumer protection; a changed floor requires a new review. Authority renews immediately before preview return and index deletion. Missing or held sources fail closed; unknown historical review scope is unavailable under department v2.

After queue commit the file index is absent. Receipt reads and duplicate execute recovery check the saved audience against current authority, so revoked readers cannot recover cleanup details. The storage worker completes the already committed deletion; queue commit is the authorization boundary, and later access loss does not resurrect deleted index data or cancel physical cleanup. Workspace privacy exports include saved cleanup receipt floors. Explicit legacy workspaces retain their prior access behavior. Independent audit-history and raw import-source classification, review UI renewal and retention command authority remain separate requirements.

### Scheduled retention approver revocation

Saved policy approval does not grant permanent execution authority. Each scheduled run must resolve the approving user against current owner/admin membership before selecting affected records or mutating them. The membership row is locked for the transaction, serializing a concurrent role downgrade or removal with the admitted run. A revoked approver causes a failed run with the existing fixed failure code and no affected-record summary; no retention mutation commits. An authorized owner/admin can recover by explicitly approving the current policy. An explicit save of an unchanged scheduled retention policy creates a new policy version under the current approver only when the prior approver no longer has owner/admin membership. Unchanged approvals otherwise remain no-ops. This membership check is necessary in both legacy and department-v2 workspaces; it does not replace the still-required per-record department authority and saved receipt floors.

Canonical contact source loading normalizes UUID letter case before deduplication and matching database snapshots. Uppercase and lowercase spellings of the same contact do not change department authority or invalidate a legitimate held-contact review.

### Retention canonical source floors

Manual and scheduled retention use the same candidate selector and authority collector. For submission candidates, their contact's canonical scope is required; missing contacts fail closed. Financial order and offline-rescue candidates preserve their saved and live Association source floors. Domain-event candidates preserve their saved privacy audience and current source, and file-cleanup candidates preserve their immutable receipt audience. The collector visits every candidate contributing to reported counts, including retained rows and rows beyond the 500-row mutation limit, using bounded pages. It does not infer authority from owner/admin status or silently omit protected candidates.

Migration 683 records the combined minimal canonical audience on each retention review and receipt. This floor is immutable, participates in the review fingerprint, and is rechecked before execution or duplicate receipt recovery. Selection is reevaluated for fresh source restrictions, and current grants renew before disclosure or mutation. Scheduled runs act as the current saved approver. Run history uses the app-role RLS gate on the saved floor; workspace privacy export includes the same floor. Explicit legacy workspaces preserve existing behavior; old reviews without provable scope are unavailable under department v2. Failed scheduled runs retain only the existing fixed failure code with an empty affected-record summary and a General public floor.

This closes the listed canonical source families, not independent raw import, delivery-envelope, audit-history or expired intake/suppression classification. Those remaining families and the legacy retention endpoint require their own authority work; a saved canonical floor must not be described as complete retention coverage. UI authority renewal remains required as well.

### Legacy retention actor and canonical source admission

The confirmed legacy retention endpoint passes its authenticated operations context into the canonical pruning function; a workspace ID alone is not execution authority. The function parses the context, verifies current owner/admin membership under a transaction lock, and checks selected submission-contact and domain-event source audiences with the same collector as reviewed retention. Event identities are selected and authorized before any source mutation, then deletion uses exactly those identities. Authority renews before mutation and before commit, so denial rolls back the complete operation. Existing explicit cutoff, hold rules and result shape are preserved. Internal callers and tests must also provide actor context; there is no ambient-owner fallback. The remaining independent import/intake scope work applies to this legacy path as well.

Privacy admission coverage also includes Association site content, membership catalogues and programme catalogues, together with each revision table. Migration 684 adds the shared workspace write guard to these six exported physical domains. They follow the same insert/update/delete and workspace-move exclusion contract as other privacy-covered records; being presentation content does not exempt their writes from an active workspace privacy operation.


### Departmental integration authority

Operation grants alone do not authorize protected records. In v2, integration keys need an admitted issuer, department binding and clearance cap; current issuer and any acting assistant authority must still permit the operation. A missing historical binding requires reviewed credential rotation, not an automatic workspace-wide grant. An empty binding means General only. Keep source protection on derived records and treat revoked/expired authority as a refusal; do not retry a provider side effect blindly. Implementation and acceptance progress is tracked in the canonical departmental audit.


A bound integration privacy export must be authorized for the entire requested slice, including saved operational and derived-source protection. Verify the final NDJSON manifest: an interrupted stream after credential expiry, revocation or department loss is incomplete. Credential rotation does not authorize replaying a remote side effect.


Integration read requests renew stored operation/resource and department authority before returning. Reusing an earlier authenticated request does not preserve revoked access. New grants cannot expand that request's original scope; retry with a fresh authenticated request after a deliberate grant change.


`create_intake_credential` accepts an optional `departmentBinding` with `departmentIds`, optional `assistantId`, and `cap` (`public`, `internal`, or `confidential`; default `internal`). Omit department IDs to use the issuer's current context/home, or pass an empty list for General. New v2 keys retain this ceiling and renew the issuer/assistant on each submission; missing historical evidence requires reviewed rotation. Rotation does not automatically grant access to the old key's records.

Submission discovery, detail and attachment reads require current access to both the saved submission audience and live source. Reclassifying a contact does not broaden an earlier submission or its replay.

Consent history and sendability renew the current actor/credential and admit saved consent and suppression evidence before returning events or a verdict. Hidden evidence is a denial, never an omitted withdrawal that turns into permission to send. Purpose selectors still constrain integration reads. Contact declassification does not release saved evidence; provider-event replay also requires its retained scope. Historical unclassified records require explicit recovery, not guessed labels.

Managed delivery admission checks saved consent/suppression scope before preparation and final provider handoff. Native adapters capture the trusted turn's department, Project and assistant visibility ceiling in server-side context. Missing or revoked source authority prevents dispatch; an already-attempted remote operation retains its uncertain-outcome receipt and must be reconciled before retrying.

Durable delivery receipts keep their creation-time recipient and consent/suppression protection. Read and exact replay require saved/current source access; denial never means the delivery ID is unused. Redaction removes source identifiers and payloads while preserving a minimal authorization floor and the no-resend receipt. Missing historical evidence is withheld rather than silently assigned General scope.


Provider callbacks require source authority at inbox admission, before a receipt is created. Order callbacks check the order's retained scope and current sources; entitlement callbacks check the contact and any existing entitlement. A source-authority rejection creates no new receipt or business effect. If an existing backend credential expires, an operator retry cannot revive it: resubmit the exact event using a currently authorized credential. Original admitted-actor provenance and event identity remain unchanged.


New provider receipts preserve their admitted source scope throughout pending recovery and exact replay. A replacement credential must cover that original scope as well as current sources. Entitlements created from pending receipts inherit the saved floor even if the contact was declassified in the meantime. Receipt history applies actor scope before pagination, and operator retry requires access to the receipt. Legacy unclassified receipts are withheld under v2; absence from a filtered list does not authorize repeating an uncertain remote effect.


Association waitlist discovery filters by current actor access to the saved submission and linked orders before pagination. An inaccessible offer is not reported as a waiting person. Promotion and replay require submission and order access; new orders inherit the submission's saved floor even after contact declassification. Restore source authority before retrying the same promotion identity.


Notification history requires current actor access to the retained source envelope and active source. Retirement removes the source reference and payload but preserves the original authorization floor and prior delivery state. A retired notification remains a minimized outcome, not a new delivery or workspace-wide record. Notification list filtering occurs before pagination; missing historical scope requires explicit recovery.


Promotion configuration responses use nullable `reservedUses`, `redeemedUses`, and `sourceRedeemedUses`. Null means the complete usage evidence is unavailable to the actor; it is not zero and must not be used to estimate remaining capacity. Promotion settings remain available, but changing global/per-contact limits requires complete usage authority. Reservation cap enforcement always uses full canonical usage. Historical imported totals without captured source evidence remain unavailable pending review.


New promotion imports with complete per-contact usage attribution capture the admitted contacts' immutable scope. Authorized list and exact replay can return those imported totals; contact declassification cannot release the saved evidence. Partial attribution and historical rows without captured evidence still return null totals. Import actors must cover every supplied contact before any promotion or attribution rows are created. Subject/workspace privacy admission checks imported usage independently of the shared promotion settings.


Canonical contact erasure removes the contact identifier from captured promotion-usage evidence after clearing its attribution. The immutable scope and usage total remain. Exact replay returns the protected aggregate without restoring erased contact links; remaining contacts still impose live source restrictions. Access to minimized usage requires the original scope.

### Credential binding preview

Owner/admin member clients can read `GET /api/crm/:workspaceId/operations/integration-credentials/binding-options` with optional `assistantId` and `cap` (`public`, `internal`, or `confidential`; default `internal`). The response identifies legacy or department-v2 mode. In department-v2 mode, `choices` contain admitted selections (`departmentIds` omitted for current context, empty for General, or explicit IDs), their department names and resolved binding. `assistants` contains visible choices. These advisory choices expire after `validForMs` (30000); credential creation repeats current issuer/assistant authority checks. A preview does not create a secret or authorize a later write.

Credential creation accepts a UUID `requestId` for retry safety. Repeating the same identity, issuer and parsed payload returns HTTP409 `conflict` with `reason: credential_already_issued` and `credentialId`; no new key is issued and the plaintext secret is never returned again. Changed input conflicts without revealing the existing ID. Inspect the existing key before explicitly rotating a lost secret under a new request identity.

Integration record reads and member-profile edits retain any trusted caller Project, assistant-visibility and shared-audience limits in addition to key department authority. Filtering occurs before pagination; an unavailable profile cannot be edited through the integration adapter.

Execution limits captured by the trusted host at credential issuance are immutable authorization metadata, not request fields. Authentication retains saved Project, assistant-visibility and mutation limits; record reads and profile edits intersect them with a narrower current caller. A direct consumer cannot discard saved limits by omitting them from its principal. Attended native and authenticated OAuth Brain-MCP issuance retain these limits. OAuth child keys also retain the exact parent lifetime; other programmatic parent kinds remain unavailable.

New integration-key lifecycle audits retain the key's immutable department binding as their source floor. Losing that department hides the audit without preventing an authorized administrator from revoking the key; revocation appends a receipt with the same protection. Historical audits are not assigned invented lineage.

A selected acting assistant also contributes its complete current execution scope at issuance. Authentication and canonical transaction admission resolve that assistant again and intersect current Project, visibility and mutation limits with the saved ceiling. Later restrictions take effect for already-authenticated direct callers; later expansion cannot exceed the key’s saved limits. Missing assistant or issuer authority denies access.

Retained CRM read adapters preserve a narrower caller execution ceiling across invocations and intersect it with renewed credentials. Intake authentication and submission transactions carry the same saved/current assistant execution limits. After Project access loss, matching a hidden contact must fail before contact, submission or follow-up effects; authorized follow-up tasks inherit the source Project.

Direct machine import chunk execution renews the credential’s bound issuer/assistant and complete execution limits before row work and again before commit. A missing bound assistant denies a contact-only import without imported entity or row/chunk effects; restoring valid authority permits resumption. This applies even without an HTTP execution wrapper.

The trusted server-side native lifecycle admission path requires pinned authoring authority independently of request JSON. It renews the executing assistant and current owner/admin issuer for preview, issuance, list and revoke. Delegated issuance binds that assistant, retains frozen execution limits and requires a stable request UUID. The attended tools listed below use this store path; full device and programmatic-parent acceptance remain pending.

Native attended credential tools are `previewCrmCredentialBindings`, `listCrmCredentials`, `createCrmCredential` and `revokeCrmCredential`. They require configure and CRM write capabilities plus current owner/admin membership. Preview supplies the operation/selector catalog and expiring binding choices. Issuance requires `requestId`, `label`, `expiresAt`, `grants` and `departmentBinding`; optional `revokeCredentialId` rotates an existing key. The executing assistant and authority are server-pinned, never request inputs. Preserve the request UUID across uncertain retries; only first success returns `oneTimeSecret`. A conflict with `credential_already_issued` names the existing key without revealing its secret. OAuth Brain-MCP invocation is available only with an authenticated read-write parent and live execution lease. Workflow invocation requires the exact retained server-side execution source and a current execution lease.

Native credential lifecycle operations renew active configure, CRM and CRM-write grants inside the canonical transaction. Previously discovered tools or a retained tool context do not authorize issuance after a grant is revoked. Human credential administration does not depend on assistant grants.

The canonical store supports a trusted OAuth-parent binding for delegated keys: child expiry is capped by the exact authenticated access token, and current admission rejects token rotation/expiry, loss of write scope, or authorization/client revocation. The stored binding contains a one-way verifier fingerprint, never the bearer token or raw verifier. OAuth Brain-MCP authentication supplies this descriptor through trusted context to native tool execution; callers cannot supply it as tool input.

Read-write Brain keys also support the four native credential lifecycle tools under current owner/primary-assistant authority and configure/CRM write grants. The child retains the exact parent key, owner, verifier fingerprint, cap and admitted context. Parent revocation, rotation or changed authority invalidates child admission. Model input cannot choose a parent or acting owner; Workflow delegation uses its own exact execution source.

Read-write Home-app bridge tokens support the native credential lifecycle as the authenticated viewer, subject to current owner/admin membership and primary-assistant configure/CRM write grants. Child expiry cannot exceed the exact bridge token expiry. App grant/cap changes, viewer role loss and server signing-key rotation invalidate child admission. Parent evidence is captured by authentication; callers cannot choose it, and no bridge bearer or signing secret is persisted.

Workflow-native credential lifecycle retains the original run, actor, executing assistant, context and authority/evidence fingerprints. The tool never accepts these as input. Child authentication renews that saved source and denies cancelled runs, changed evidence or lost authority. An unavailable old child can be inspected, revoked or rotated from a currently authorized attended session; reuse issuance request IDs after uncertain outcomes and never assume a missing response means no credential was created. Successful-use metadata is committed only after authority admission; refused authentication does not update it.
