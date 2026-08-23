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
