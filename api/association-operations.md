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
