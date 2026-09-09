---
title: Association operations API
description: Workspace-scoped API for enquiries, consent, memberships, events, ticket inventory, registrations, orders, and provider reconciliation.
tags: [api, crm, memberships, events, integrations]
---

# Association operations API

Use this API when Brian is the business system of record for a membership
organization, association, chamber, club, community, or training provider.
It extends the CRM person graph with operational records; it does not create a
second contact database.

## Authentication and authority

- Base path: `/api/association`
- Bearer credential: `sk_brain_*` Brain key or Brain OAuth access token
- `read` scope: GET endpoints
- `read_write` scope: all mutations
- Workspace: always derived from the credential. There is no workspace body or
  path parameter.

Keep the credential on a trusted backend. A public form or checkout must call
its own server, which validates the public input and forwards a bounded request
to Brian. Never embed a Brain credential in browser JavaScript.

All referenced `contactId` values must be live CRM `person` entity ids in the
credential's workspace.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/external-identities` | Link a provider subject to a CRM person |
| `GET` | `/external-identities/resolve` | Resolve `provider` + `providerSubject` |
| `POST` | `/enquiries` | Idempotently accept an enquiry |
| `GET` | `/enquiries` | List by status, queue, or owner |
| `PATCH` | `/enquiries/:id` | Update operational state, queue, or owner |
| `POST` | `/enquiries/:id/notes` | Append an internal note |
| `GET` | `/enquiries/:id/notes` | Read the internal note timeline |
| `POST` | `/consents` | Append a consent grant or withdrawal |
| `GET` | `/contacts/:contactId/consents` | Return evidence and effective preferences |
| `POST` | `/plans` | Upsert a membership plan by stable key |
| `GET` | `/plans` | List plans |
| `POST` | `/memberships` | Idempotently create an entitlement |
| `GET` | `/contacts/:contactId/memberships` | List a person's memberships |
| `PATCH` | `/memberships/:id` | Adjust status, end date, or renewal mode |
| `POST` | `/events` | Upsert an event by slug |
| `GET` | `/events` | List events |
| `POST` | `/events/:eventId/tickets` | Upsert a ticket type by stable key |
| `GET` | `/events/:eventId/tickets` | List ticket types and current availability |
| `GET` | `/events/:eventId/registrations` | List/export event attendees by state |
| `POST` | `/orders` | Reserve inventory and create registrations |
| `GET` | `/orders/:id` | Fetch order, lines, and registrations |
| `POST` | `/orders/:id/provider-events` | Reconcile signed payment-provider evidence |
| `PATCH` | `/registrations/:id` | Cancel or check in one registration |
| `GET` | `/notifications` | List delivery intents |

Lists use `limit` (1-100) and an opaque `cursor`; pass `nextCursor` unchanged.

## Idempotency

- Enquiries are unique by `(source, sourceSubmissionId)`.
- Memberships and orders use `idempotencyKey`.
- Consent/provider reconciliation can use `(provider, providerEventId)`.
- Replaying the same key and request returns the original record with HTTP 200.
- Reusing an enquiry, order, or membership key with changed input returns HTTP 409.

## Checkout rules

Submit integer money values in minor currency units when configuring plans or
tickets. Order requests name ticket ids, quantity, whether member pricing is
requested, and exactly one attendee record per place. Brian locks inventory,
checks event and ticket sale windows, ignores expired reservations, enforces
capacity and per-order limits, verifies an active eligible membership, and
calculates all prices on the server.

New orders are `pending` reservations. Do not mark them paid from a browser
success redirect. After verifying a signed payment webhook, submit a provider
event targeting `paid`, `failed`, `cancelled`, or `refunded`. Provider events
are idempotent and invalid order-state transitions return HTTP 409.
A paid event received after its reservation expired is rejected for manual
reconciliation rather than silently overbooking inventory.

Use consent purpose `newsletter` for newsletter subscription. A grant means
subscribe and a withdrawal means unsubscribe; do not maintain a separate
subscriber boolean that can drift from the evidence history.

## Provider boundaries

Brian owns the association record, entitlement, inventory, and reconciled
order state. An identity provider owns authentication sessions; a payment
provider owns payment instruments and settlement; a delivery provider owns
transport and bounce telemetry. `/notifications` exposes durable delivery
intent. A `pending` intent is not evidence that a message was delivered.

## Workspace module admission

Association commerce now has workspace lifecycle state separate from navigation and assistant grants. Existing workspaces keep enabled access; new workspaces start disabled. Ticket writes and new orders return HTTP 409 with `module_disabled` or `module_draining` when admission is stopped. Generic CRM identities, enquiries, consent, entitlement plans/grants, events and unconstrained participation remain available under their existing authority.

Disabling preserves historical reads and existing-order recovery. Exact committed order retries return the existing order even after disable; changed reuse of an idempotency key still conflicts. Existing provider reconciliation and registration cancel/check-in remain available under their original authority. A disabled module does not refund or erase an order. Credential permissions do not enable the workspace module. Owner/admin lifecycle controls and scoped integration credentials use the native command adapters documented below; commerce write permission does not grant module administration.

## Native command adapters and scoped integrations

Member sessions use `/api/crm/:workspaceId/association`; CRM-only credentials
use `/api/crm/integration/association`. Both call the same typed service as the
legacy `/api/association` adapter. No member browser needs a machine key.

Order history is available at GET `/orders` with bounded `limit`, `cursor`,
`eventId`, `contactId` and `status` filters; GET `/orders/:id` returns one order.
POST `/orders/:id/cancel` cancels an eligible pending order and does not claim
a refund. POST `/orders/:id/confirm-free` requires a zero-total, unexpired
pending order and creates no payment-provider evidence. The member/scoped
adapters expose GET `/module` and `/module-blockers` for state and pending
orders. Authorized history and recovery remain available after disabling.

Member/chat callers cannot submit payment-success evidence. Backend provider
events require the explicit reconciliation authority; a CRM integration key
needs `association.provider_events.write` for both every order event and the
provider. `association.orders.write` alone cannot mark a paid checkout successful.

Module reads use GET `/api/workspaces/:workspaceId/modules`. Owner/admin
member actions use POST `/api/workspaces/:workspaceId/modules/association/actions`
with `action` and `expectedVersion`; machine grants cannot activate a module.
Legacy plan/event writes retain their established Brain-key catalog authority
and now use the generic CRM configuration commands. New scoped keys require
`crm.catalog.configure` and matching catalog resources.


### Effective membership reads

`GET /contacts/:contactId/memberships` accepts optional `activeOnly=true|false`
and ISO `effectiveAt`, retaining `memberships` and raw status while adding each
row's `isEffective`/evaluation instant. Access requires active status with start
inclusive and end exclusive (or absent). Member-price admission always rechecks
current database time; a historical read or raw active status cannot authorize
a discount outside the grant window.


### Query-bound collection cursors

Paged enquiries, plans, events, orders, event registrations and notifications
use the common CRM cursor: immutable creation timestamp/id with microsecond
precision, a first-page upper bound, and workspace/resource/filter binding.
`createdAfter` is inclusive and `createdBefore` exclusive. Read authority and
integration event selectors are rechecked before each page. Named arrays and
URLs stay unchanged. Retired unbound cursor tokens are rejected with
`invalid_input`; restart those traversals without a cursor. Do not decode or
construct tokens in adapters.

Consent preference reads order occurrence time, recording time, then stable id,
all descending, matching generic CRM sendability and segment evaluation. A
delayed old grant does not override a newer withdrawal by receipt time alone.

Consent provider replay compares the complete business request, including metadata, occurrence time or its absence, and the legacy wording version. Changed reuse returns `409 idempotency_conflict`, also on concurrent insertion. Pre-upgrade events require exact stored fields and explicit original occurrence time. See CRM Operations → "Provider evidence replay" for the shared fingerprint contract.

## Consent wording compatibility

Consent belongs to shared CRM and remains available with Association disabled.
Migration 502 catalogues immutable default and localized wording; see
`crm-operations.md` → "Immutable wording and locale resolution". The existing
`POST /consents` keeps `wordingVersion` and accepts optional `locale` from
`en`, `zh`, `zh-CN`, `ja`. For a known purpose, the requested version must exist
and the purpose must be unarchived. The server saves its exact text/hash/version
reference and resolved locale, with stored-default fallback. Legacy purposes
without a catalog remain unlinked (null wording/hash/version id) and cannot
claim localized wording. Exact provider retries return the original evidence
before checking the current catalog. No caller-supplied text/hash is authority.

## Transaction-time scoped credential admission

Scoped CRM-key commerce writes lock and recheck the active credential and
stored grants before module/inventory locks. The original request ceiling and
current stored selectors must both permit each referenced event, plan and
provider. This covers ticket saves, new orders, exact order/provider replay,
cancellation, free-order confirmation and registration updates. Revoked,
expired, absent or malformed stored authority returns HTTP 401
`credential_revoked`; insufficient live scope returns HTTP 403
`integration_scope_denied`. A command admitted first finishes before revocation
returns; a revocation that wins admission prevents the command. This does not
cancel an already admitted command or widen the disabled-module recovery path.

## Notification retirement

`GET /api/association/notifications` accepts `status=retired` and preserves the
`notifications` array plus cursor. Rows expose `retiredAt` and
`retiredFromStatus`. Never dispatch a retired notification or resolve its erased
recipient reference as a live contact. The previous `sending` state means the
external result is uncertain; `sent` means it had been recorded sent before
retirement. Retirement itself does not assert delivery, recall or refund.

Canonical contact erasure clears attributable notification payloads, recipient
and source references, provider ids and error text. Shared notifications for
another contact block erasure. Retired rows cannot be resumed. External-provider
reconciliation and erasure remain integration/operator responsibilities.

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

## Reservation expiry and requested shutdown

Pending orders with an elapsed reservation deadline are cancelled by the OSS
Association lifecycle worker; their reserved registrations are cancelled in the
same transaction. The worker rechecks the order after locking it. A concurrent
paid transition, renewed reservation, future/indefinite hold or terminal order
is preserved. Expiry creates no payment or refund evidence and records one
expiry audit. Repeated candidates are no-ops.

A previously requested draining module becomes disabled once no pending orders
remain. Restart or two workers cannot duplicate that versioned change, and a
concurrent re-enable is respected. Home placement and saved assistant grants
remain unchanged. Paid order history remains readable. An indefinite pending
order stays a visible drain blocker until resolved through ordinary commands.

`ASSOCIATION_LIFECYCLE_ENABLED=false` or `0` pauses this worker; default is enabled
with `runWorkers` in both editions. The internal due-expiry command belongs only
to its system principal. Agents use the existing authorized cancellation and
provider-reconciliation paths, with their ordinary authority requirements.


### Inventory admission and workflow boundaries

Live ticketed or capacity-limited events require an Association order, including
zero-price admission with explicit free confirmation. Event and ticket sales
windows both apply; an ended event cannot accept a checkout. Capacity and member
pricing are checked under transaction locks at database time. Confirmation of an
expired hold is rejected, including after waiting for a concurrent writer.

`association.inventory.sold_out` and `association.inventory.available` are
committed CRM workflow events. Filter by event type and enumerated event/ticket
keys; enable automated changes to receive lifecycle expiry events. Payloads carry
catalog pointers, capacity, used count and revision, with no attendee or payment
payload. One subject/revision identity prevents duplicate emission on retry.
Cancellation, expiry and capacity edits can create availability transitions.
These events are durable workflow inputs, not proof that a notification was sent.


### Membership renewal periods

Compatibility membership input accepts `providerPeriodId` and `predecessorId`.
Provider-backed writes require backend/provider authority, including direct
compatibility stores. A terminal membership cannot be revived; renew it through
a new idempotent provider period linked to its terminal predecessor. Active
membership periods extend in place. The same period cannot create multiple grants
through changed transport keys. See the CRM operations provider renewal contract
for plan/provider scopes, cancellation timing, lineage and replay errors.

## Explicit intake-backed waitlist offers

`GET /api/association/waitlist` returns `submissions` and `nextCursor`; continue
until null. Optional `eventId` and `includeClosed=true` filter the list. The
member and scoped integration Association routers expose the same paths under
their own base. Integration reads require both event-scoped `association.read`
and definition-scoped `crm.submissions.read`.

A versioned `association_waitlist` intake definition has required, single-UUID
option, submission-only `association_event_id` and `association_ticket_id` fields
and an ordinary consent mapping. Original submission versions bind the target.
`POST /waitlist/:submissionId/offer` accepts `promotionId` (stable UUID), optional
`reservationMinutes` (1–120, default 20) and `useMemberPrice` (default false).
Integration writes require both event-scoped `association.orders.write` and
matching definition-scoped `crm.submissions.write`; current revocation applies
even on replay.

The offer uses the submission's existing person and ordinary stock reservation
in one transaction. The response has `offer`, `created` and the linked order;
201 means a new reservation/link, 200 means replay. Reusing a promotion UUID with
changed input conflicts. Pending or paid offers prevent another promotion;
a cancelled/failed/refunded offer permits a new explicit promotion UUID. Never
retry an expired offer with a new identity without a deliberate staff action.
Disabled modules reject new offers but permit history and exact replay. Offered
means reserved, not notified or paid. Confirmation and sending are separate
commands. No native waitlist status, FIFO promotion, timer or automatic charge
is implied.

## Provider binding and verified money

Before external paid checkout, a trusted backend calls
`POST /orders/:id/provider-binding` with `provider`, `providerReference`,
`amountMinor` (safe nonnegative integer) and uppercase `currency`. They must
match the canonical nonzero order. Binding a new object requires an enabled
module and unexpired pending reservation. Exact replay is 200; initial binding
is 201. An object cannot belong to two orders, and a bound identity cannot change.
Use provider keys that distinguish accounts/environments when necessary.

`POST /orders/:id/provider-events` now requires the same reference, amount and
currency in addition to event id, target status and occurrence time. Older
payloads omitting these facts are rejected. The backend verifies signatures
and provider state before normalization; redirects are never proof. Only a
successful cumulative full refund maps to `refunded`. Partial/pending/failed
refunds remain provider-resolution work and cannot assert full refund.
Zero-value orders use `confirm-free` without provider evidence.

Both commands require backend payment authority. Scoped keys need
`association.provider_events.write` with provider and all order-event selectors;
revocation applies even to replay. Human/assistant payment assertions are denied.
Exact normalized event replay is safe after timeouts; changed event-id reuse
conflicts. Semantic duplicates retain evidence without repeated notifications
or state-change effects. Bound recovery remains available after module disable.
Refunds include checked-in registrations. Expired or impossible transitions
remain errors; no stock is silently revived. This contract is payment admission;
the normalized inbox contract below adds durable receipt handling, without claiming live provider acceptance.


## Durable normalized provider receipts

The order provider-event routes now commit a normalized inbox receipt before
applying domain effects. Success includes `receipt` with id, state, attempts
and safe target fields. Atomic application and receipt acknowledgement make an
exact replay safe after a lost response. Changed input under the same
workspace/provider/event identity returns HTTP 409 `idempotency_conflict`.
A concurrent exact request may return 409 with
`details.reason=provider_event_processing`, `receiptId` and `receiptState`;
retry the identical input after processing. An error carrying a receipt does
not assert that payment or membership changed. Inspect its state.

`POST /provider-entitlement-events` accepts `provider`, `eventId`,
`providerReference`, `providerPeriodId`, `occurredAt` and a canonical typed
`grant_entitlement` or `update_entitlement` in `command`. Grant provider/object/
period fields must match the envelope and specify a finite end. Update must
match the existing provider object and period. Period-end cancellation changes
renewal mode; immediate cancellation changes status. Terminal renewal creates
a new period/grant with predecessor linkage. Older events cannot overwrite
newer applied provider state. This generic membership path works when the
Association commerce module is disabled. Legacy trusted-backend entitlement
commands remain compatible but do not claim normalized webhook receipts.

Scoped backends need `association.provider_events.write` for the provider and
`crm.entitlements.write` for the plan. Orders retain the all-order-event ceiling.
Each processing transaction checks current revocation and original resource
ceilings. The worker uses saved execution authority, never an elevated system
identity. An explicit authorized identical request can replace execution
credentials after repairing a cause; original admission provenance stays frozen.

`GET /provider-receipts` accepts opaque `cursor`, `limit` (1-100), optional
`orderId`, `entitlementId`, and `state`. It returns `{receipts,nextCursor}`.
States are `pending`, `processing`, `applied`, `retry`, `needs_reconciliation`.
Member reads use workspace membership. Scoped reads require `association.read`;
order event ceilings filter before pagination, and membership rows additionally
require `crm.entitlements.read` for their plans. Payloads, hashes, execution
credentials and lease tokens are excluded from these reads.

Known transaction/connection failures retry with bounded backoff; each automatic
cycle has at most eight attempts. Invalid evidence, late success, impossible
transitions and revoked authority remain visible reconciliation cases. Resolve
the cause and explicitly replay the same input to restart a cycle. Applied means
canonical state committed; it never means a notification was sent. The trusted
backend still verifies provider signatures before forwarding normalized input.


## Provider backend reference

The OSS `scripts/crm/reference-provider-backend.mjs` and
`docs/operations/crm-provider-reference.md` supply a fictional loopback webhook
and periodic missed-event adapter. The provider port verifies exact raw bytes
before normalization and enumerates a stable, complete event cursor. Both order
and membership events use the normalized inbox endpoints. The client verifies
the configured workspace against its CRM credential before forwarding.

The private SQLite checkpoint advances only after an applied or explicit durable
processing/reconciliation receipt. Uncertain responses and unrecorded errors
retain the cursor; HTTP 429 persists Retry-After. Restart repeats stable event
identities. Separate processes lease/compare-and-set the checkpoint. Paginated
reconciliation pointers require operator inspection; caught-up polling does not
imply every receipt applied or any notification sent. No API secrets or raw
provider bodies are stored in the checkpoint. Production provider signatures,
object mapping, polling retention and account/deployment acceptance remain the
integration owner's work.
