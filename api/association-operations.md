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

Order source inheritance is being implemented for ordinary and imported ticket
orders. The canonical store records the buyer and identified attendees' current
scope/version evidence and derives the order and generated registrations from
all those sources. Held, retired, missing or incompatible private source records
cannot be combined into a new order. Callers supply contact identifiers, never
invent department labels or source snapshots. Public ticket publication does
not make operational buyer or attendee information public. Evidence persistence
alone is not completed department authorization; the rollout still requires
current-source admission, filtered reads and legacy-record recovery.

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


## Fictional workflow recipes

The OSS `scripts/crm/fixtures/association-workflows.json` provides five disabled
recipes with canonical workflow definitions and sample inputs. Submission
notification and managed outreach use frozen delivery commands; generic event
registration keeps its stable source identity and cannot bypass order inventory;
member onboarding reads effective access before creating an attributed task.
Optional weekly digest prose has only CRM segment read tools and must follow
pagination. Policy, catalog ids, mailbox/channel targets and assistant grants
must be configured before enabling. Fake sample read outputs are test data,
never a live authority source. See `docs/operations/crm-workflow-recipes.md` in
the OSS tree for binding and replay instructions. Tool wiring and complete
workflow/delivery acceptance remain separate from recipe schema validation.

## Native Association tools

The native catalog includes getAssociationModuleStatus, listAssociationTickets,
saveAssociationTicket, listAssociationOrders, getAssociationOrder,
createAssociationOrder, confirmFreeAssociationOrder, cancelAssociationOrder,
listAssociationRegistrations, updateAssociationRegistration,
listAssociationWaitlist, offerAssociationWaitlistPlace,
listAssociationModuleBlockers and listAssociationProviderReceipts.

Every tool requires the Association app and its read/write set. Order,
registration, waitlist and provider receipt tools also require CRM and the
matching set. These grants start off for all assistants and are independent of
module state and Home visibility. Read MCP credentials see read tools only;
execution rechecks current resolved grants. Unavailable permission is an error,
not an empty collection. Follow every nextCursor with unchanged filters.

createAssociationOrder takes {order} with the canonical order schema and stable
idempotencyKey. saveAssociationTicket takes {eventId,ticket}; waitlist offer
takes {offer} with submissionId and stable promotionId. get/cancel/free-confirm
take {orderId}; registration update takes {registrationId,update}. Reuse an
unchanged intended operation after uncertainty. A new id means a new intention.
Free confirmation cannot settle a priced order. Module lifecycle and payment
binding/assertion are human/backend-only operations, never native assistant
tools. Disabled history and permitted recovery remain available.

## Versioned website membership content

The membership catalogue workflow saves drafts separately from published plan
prices and content. The native tools `previewMembershipCatalogue`,
`saveMembershipCatalogueDraft` and `publishMembershipCatalogue` use the same
validated Association service as the admin UI. They require configuration
authority and the Association app grants; publishing requires human confirmation.
Save and publish pass the exact `expectedVersion`. On a conflict, reload the draft
and review the new changes before trying again.

Owner/admin HTTP clients use `/api/crm/{workspaceId}/association/membership-catalogue`:
`GET /draft`, `POST /draft` with `{ expectedVersion, document }`, and
`POST /publish` with `{ expectedVersion }`. Integration credentials cannot save or
publish drafts. Website backends read
`GET /api/crm/integration/association/membership-catalogue/{site}` with
`association.read` and `crm.entitlements.read` capabilities, subject to their
allowed plan scope. A reader reports receipt with `POST` to the same path plus
`/observed`, carrying `{ revision }`.

The document contains typed plan terms, stable plan keys, multilingual copy,
optional per-field brand overrides, visibility and display order, supported
application requirements, document links and membership-page sections.
Publication validates required translations and references, binds canonical plan
UUIDs, and atomically updates current plan prices and an immutable published
revision. It does not change existing subscriptions or historical orders.
Previously published identities must be retained; hide or close retired plans.
Legacy direct writes to catalogue-managed plan terms are rejected.

Website readers must consume the published revision without repository fallbacks.
Draft edits cannot change live content. Featured offers derive from active,
explicitly selected promotion records, including expiry, recurrence and usage
conditions; public output never includes private codes. Checkout still validates
the submitted code, current eligibility and price. Publication reports pending
synchronization until the website reader acknowledges the revision; this does
not mean already-open browser pages have refreshed.

This contract requires the membership catalogue release and migration 557 on the
server. Documentation publication does not assert that a hosted service has
already deployed that release.

New ticket-order insertion in v2 workspaces checks the current buyer and identified attendee source permissions, and the inherited operational scope, inside the transaction. Workspace ownership alone does not grant department access. Human and import actors need current source membership; an agent or credential requires a trusted execution department grant. An unbound credential is refused rather than using the workspace owner implicitly. This is an insertion guard only: idempotent replay, existing-record reads and mutations, and integration-credential binding/recovery remain incomplete in the rollout.

Order detail and ordinary/imported idempotent retries now require both the saved protection floor and current source access. Orders with missing legacy evidence or unavailable sources are denied. Order list results, pending counts and financial summaries are filtered before pagination/aggregation using the same actor scope. Cancellation and free confirmation also renew this floor after acquiring locks and before commit. Registration rosters, provider reconciliation and other record families remain under implementation; these order gates are not full Association departmental sign-off.


Registration lists and operational rosters now apply saved registration and parent-order protection plus live source authority before pagination and totals. Registration management lookup, status updates and check-in corrections also renew that authority. In department-v2 workspaces, legacy registrations lacking source evidence are withheld until a recovery path supplies it; workspace owner/admin status does not bypass the department checks. Other operational record families and legacy recovery are still being implemented.


Direct manual/imported CRM participation now stores inherited contact protection and renews source authority for creation, replay, status changes and check-in correction. Member REST, integration REST and Brian participation listing carry the authenticated actor into the same registration and parent-order filter. Department revocation removes records from both CRM and Association discovery; a workspace role cannot restore access. Legacy records without evidence still require recovery.


Manual membership grants and imported source memberships now persist the contact-derived department floor. Membership/entitlement discovery forwards the real human or agent actor and filters saved protection plus live source authority; retries and lifecycle updates recheck that authority. Legacy rows without evidence are withheld in department-v2 workspaces. Provider renewal bindings, rescue/sponsorship creation and legacy recovery remain pending and must not be assumed to bypass these gates.


Offline membership rescue cases now retain source-derived department evidence. Their finance-role requirement remains necessary, and current department access is also required for listing, creation/retry, settlement, cancellation and reversal. A generated membership inherits the original rescue floor plus current source protection. Source declassification cannot widen it. Legacy cases lacking evidence require explicit recovery. These finance review commands remain limited to the existing human reviewer contract.


### Protected operational list cache lifetime

Association's shared operational page hook uses the existing surface cache with a maximum 30-second projection lifetime measured from request start, not response arrival. Expiry removes rows and pagination even while offline or a refresh is pending. HTTP 401/403/404 removes the cached projection immediately; ordinary transient failures may retain rows only within their original lifetime. An expired response cannot reinstall its data. These browser bounds supplement, and never replace, canonical server authorization. Failed operational commands with an authorization denial evict Association and CRM cached projections for that workspace before offering recovery. This shared-list policy does not certify component-local selected contacts, editor drafts, or independent order-detail caches; those require separate lifecycle coverage.

The offline payment action editor stores only the selected record identity and derives its content from the current authorized rescue page. Expiry or denial unmounts the editor and discards its form content. Empty payment queues describe the current viewer’s access, not an assertion that the workspace has no payments.

Mounted operational lists renew every 15 seconds and on window focus, before the 30-second deadline. A failed renewal does not extend the previous deadline. This avoids periodically discarding an authorized open payment form on healthy connections.


### UI verification record: protected payment queue (2026-10-06)

Local owner queue inspected at 1440×900 and 390×844. Desktop score: task clarity 2, findability 2, staff vocabulary 2, role fit 2, honest states 2 (10/10); phone: 10/10 by the same rubric. Phone document width is exactly 390 and the primary action measures 44px. Desktop controls use the compact size required by responsive contract M3. Screenshots: `/tmp/miniapp-association-cache-desktop.png`, `/tmp/miniapp-association-cache-phone.png`, and final revoked desktop `/tmp/miniapp-association-cache-revoked.png`. The empty copy describes the viewer's available records without exposing hidden record counts.

With the page open, expiry of a fictional local department edge removed the visible rescue rows without navigation. A second check with the payment review open removed the selected person's name and payment form and returned to the scoped empty queue. No payment was submitted through the browser. Fixtures restore the prior edge expiry automatically. Member-only denial is component-tested; a live member browser walkthrough remains outstanding. Contact selection, independent order caches and other editor-local copies still require separate verification.


### Protected order projection lifecycle

The Association operational projection hook also owns order lists, financial totals, the Home pending-order summary and expanded order detail. All share the same request-start 30-second bound, 15-second renewal, focus renewal and denial eviction through the existing surface cache. Order detail is keyed by workspace/viewer/order and only mounted while its parent order remains on the current authorized page. There is no independent component-state copy of attendee names or order lines. Closing a detail or changing the list cannot let an older request paint under another order. Failed order mutations discard cached order projections before recovery; canonical server authority remains decisive.


### UI verification record: order projection revocation (2026-10-06)

Orders inspected at 1440×900 and 390×844 with expanded ticket and attendee details. Desktop and phone: task clarity 2, findability 2, staff vocabulary 2, owner role fit 2, honest states 2 (10/10). Phone document width equals viewport width (390); primary controls have the existing phone 44px floor. Screenshots: `/tmp/miniapp-order-cache-desktop.png`, `/tmp/miniapp-order-cache-phone.png`, `/tmp/miniapp-order-cache-revoked.png`. A live fictional department expiry removed the order, financial summary, ticket lines and attendee name from the open phone view without navigation. The controlled fixture restores its original edge expiry. Live member and weaker-assistant walkthroughs remain outstanding; this record is not a complete Association sign-off.


### Contact selection authority

Association contact search results use the bounded operational projection cache. Selected contacts in membership, payment, reservation, sponsorship and privacy forms retain only a contact identifier in local state; their displayed name and hint come from a renewed canonical CRM record read. The selected-contact projection rejects unavailable, archived, non-contact or mismatched records. A denial removes the projection immediately, and transient failure cannot retain it beyond the 30-second request-start deadline. A form cannot submit with an unavailable selected contact. This does not authorize copied guest fields, previously generated sponsorship tokens or privacy preview receipts: each still needs its own source-linked lifecycle.

Order buyer-filter labels also use the canonical selected-contact projection. The membership adjustment editor stores only the membership identifier and derives its current row from the protected page, so its contact name and financial fields cannot outlive that page.

Contact erasure preview and execution controls also disappear when their selected contact is no longer readable; a previously loaded preview cannot keep that action enabled. Broader privacy receipt/source-floor and linked-guest field coverage remains separate work.


### UI verification record: selected contact lifetime (2026-10-06)

At 1440×900 and 390×844, the offline payment form displays a contact only after a canonical record read. On the phone, document width is 390px and submit height is 44px. Scores on both widths: task clarity 2, findability 2, vocabulary 1 (existing finance wording remains dense), role fit 2, honest states 2: 9/10. Screenshots: `/tmp/miniapp-contact-cache-desktop.png`, `/tmp/miniapp-contact-cache-phone.png`. With a contact and plan selected, a fictional local department expiry removed the contact chip, source-derived summary and dependent form without navigation; the remaining search returned an empty authorized projection. Evidence: `/tmp/miniapp-contact-cache-revoked.png`. No payment was submitted. Member walkthroughs and linked-guest copies remain unverified.


### Sponsorship operational source protection

Allocations persist protection inherited from the sponsor contact and the sponsoring membership's saved floor. Invitations inherit that allocation floor plus the nominee contact; redemption preserves the invitation floor in the resulting membership. Saved source evidence contains canonical contact snapshots, while current parent membership/allocation authority is also required. Current actor authority gates initial creation, idempotent replay, discovery before pagination/counts, cancellation, issue, revocation and redemption. Cascading cancellation requires authority over every affected invitation and membership and rolls back atomically on denial. Legacy rows without evidence fail closed under v2 pending explicit recovery. Token possession does not replace a trusted, currently authorized execution principal; integration binding remains a separate required rollout task.

Sponsorship writes serialize per workspace before acquiring allocation/invitation locks. Source authority locks parent memberships before contact snapshots. Allocation seat totals are nullable: if any contributing invitation is unreadable, the response returns unknown rather than leaking its count or presenting a partial count as capacity. UI uses the existing localized unknown label. Admission still checks actual capacity and requires authority over contributing invitation records before exposing capacity-dependent outcomes.

A newly issued sponsorship token is displayed only after refreshed invitation/allocation projections confirm that the current nominee and pending invitation remain readable. The UI clears that token when any supporting projection disappears or the invitation expires/revokes; refreshing an idempotent issue never recovers a token from storage.


### Linked reservation guest lifetime (2026-10-06)

Reservation drafts store linked guest identities without copying lookup names. A shared bounded canonical contact projection resolves every linked guest before display or submission. Missing, archived, mismatched or denied contacts invalidate this projection; offline failures can retain it only until its original 30-second deadline. Linked guest fields and CRM links are hidden while the projection is unavailable, including any operator edits derived from those contacts. Reservation is blocked until all linked contacts are readable. Refresh retries the read; clearing a link also clears its name and email edits so protected data cannot be converted into an unlinked guest by that action. Unlinked guests retain their ordinary editable fields. Server source checks remain authoritative at reservation execution.


### UI verification record: sponsorship and linked guests (2026-10-06)

Sponsorship was inspected at 1440×900 and 390×844. Phone document width is 390px and both submit controls measure 44px. Screenshots: `/tmp/miniapp-sponsorship-desktop.png`, `/tmp/miniapp-sponsorship-phone.png`. Scores: task clarity 1 (allocation terminology is dense), findability 2, vocabulary 1, role fit 2, honest states 2: 8/10 on both widths. Issued-token revocation has component coverage; the corresponding browser issuance/revocation and non-manager walkthrough remain unverified.

The reservation guest editor was reached through Events → event → Tickets & fees → Reserve for someone, then a buyer and linked guest were selected. At 1440×900 and 390×844 it shows the canonical guest name and permits editing. Phone width is 390px and reserve height is 44px. Scores: task clarity 2, findability 2, vocabulary 2, role fit 2, honest states 1 (the shared refresh error still refers to a list): 9/10 on both widths. Screenshots: `/tmp/miniapp-linked-guest-desktop.png`, `/tmp/miniapp-linked-guest-phone.png`. After fictional local department expiry, the open form removed guest inputs and the CRM link, and disabled reservation. Clearing the link returned blank name/email fields. Evidence: `/tmp/miniapp-linked-guest-revoked.png`. No reservation was submitted in this browser check. Non-manager browser acceptance remains unverified.


### Module and management-role projection lifetime (2026-10-06)

The Association module snapshot, including its management-role flag, is an expiring authority projection. Both navigation prefetch and mounted readers retain it for at most 30 seconds from request start; mounted readers renew every 15 seconds and on focus. A denied module read evicts the snapshot immediately. An offline or delayed renewal cannot extend the previous authority. Successful module changes refresh the complete canonical snapshot rather than copying the former management flag into a new module result. Management-only forms must unmount when the role snapshot is unavailable or read-only, with explicit unavailable/retry or member guidance. Department record protection remains independent of the staff management role and is still enforced by canonical server commands.


### UI verification record: management authority lifetime (2026-10-06)

The open offline-payment settlement editor was reviewed at 1440×900 and 390×844 after the module-cache change. Phone width is 390px and the primary control is 44px. Scores on both widths: task clarity 2, findability 2, vocabulary 1 (existing settlement wording remains), role fit 2, honest states 2: 9/10. Screenshots: `/tmp/miniapp-module-editor-desktop.png`, `/tmp/miniapp-module-editor-phone.png`. No payment action was submitted. Component tests verify immediate editor removal on a member role/denial and removal during offline expiry, but the exact live role-change walkthrough is pending explicit approval: automatic approval review rejected temporarily changing the fictional local audit user's workspace role. No role mutation was executed. This is an evidence limitation, not a completed live revocation check.

### Erasure review disclosure lifetime (2026-10-06)

The staff erasure review uses the read-only review endpoint as its display authority. A projection lasts at most 30 seconds from request start, renews every 15 seconds and on focus, and is evicted on denied or stale review responses. Execution success or uncertainty refreshes the same review; recovery never creates another preview or silently repeats erasure. Consumed receipts use the saved review floor and remain recoverable without the deleted contact. Losing management authority hides the review and receipt. Selecting another contact resets the review reference. Retention and physical cleanup need their own authority lifecycle before full privacy sign-off.


UI verification record (2026-10-06): erasure review desktop 1440×900 and phone 390×844 scored 9/10 (task clarity 2, findability 1, staff vocabulary 2, role fit 2, honest states 2). Phone document width is 390px and the execute target is 44px high. Technical identifiers, raw domain details and receipt payloads are collapsed; the visible review summarizes actions and counts. A new fictional General contact's open review disappeared after a newly linked membership added inaccessible department protection; the contact remained visible. No erasure or existing-user access change occurred. Role-loss rendering and consumed-receipt lifetime are covered by component tests; the separate live workspace-role test remains pending its earlier approval.


### Explicit ticket-order destination

Ticket-order creation accepts an optional `destination`: `{kind: "department", departmentId}` or `{kind: "general"}`. Omission retains canonical context/home admission. Human HTTP and native agent commands carry the same field into canonical admission; selecting General never removes a source department, privacy or sensitivity floor. A selected department must be active and writable under the current actor and frozen execution ceilings. The destination participates in the idempotency fingerprint: replay cannot relocate an existing order, and denied creation writes no order, registration or inventory event. Department selection/preview controls and equivalent choices for other operational families remain separate required rollout work.


Order destination preview accepts the canonical buyer/linked-contact IDs and returns only choices admitted with their full current source floor and the caller's current/frozen authority. It includes the implicit home/context choice when available and explicit General/department choices that pass the same admission as creation. Each choice carries its resulting envelope and authorized department names. A failed implicit home is not silently replaced: the caller must select another admitted choice explicitly. Preview is advisory, lasts at most 30 seconds for display, creates no business rows/events, and never substitutes for admission at creation. Native tools and HTTP share the same preview command/store.


Ticket reservation management follows the staff-console role contract. Human members can inspect admitted ticket/event history but cannot create reservations; the canonical service requires owner/admin for a human create-order command. Delegated assistant/workflow/OAuth/Home-app calls naming a human renew that person's workspace management role before reservation creation, in addition to their existing capability/source/destination ceilings. Dedicated integration credentials retain their independently granted operation/resource contract. Losing management permission closes the event ticket editor/reservation UI and removes mutation controls.

Direct Association command and read adapters reload saved integration-key execution limits. Project and assistant-visibility restrictions apply to current sources, saved output floors, parent-child reads, counts and destination admission even outside the HTTP authentication wrapper. A source becoming less restricted does not lower a saved order or membership floor.

Operational source-access refusal returns `not_authorized` with non-disclosing `details.recovery` guidance. A missing record, missing historical evidence and inaccessible source do not identify different recovery categories. Preserve the original request identity and do not automatically retry a mutation or issue a fresh payment. Retry reads only after access review. Independent fresh creation requires an operator to verify that it will not duplicate a prior effect; current contact access cannot reconstruct or certify historical protection. Historical rows remain withheld without trustworthy original evidence.

The Association staff console exposes “Can't find a record?” on every section. It links to currently accessible contacts and, for managers, payment reconciliation. After the manager checks the prior outcome, independent fresh-start links open the ordinary event, free-membership or offline-payment flow. They neither submit an operation nor recover historical lineage. Assistants must likewise preserve original request identity and verify an uncertain outcome before invoking the ordinary creation commands.
