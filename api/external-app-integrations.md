---
title: Authenticated external application integrations
description: Source-owned CRM publications, private version-bound documents and conditional Outlook bookings.
tags: [api, integrations, authorization, documents, calendar]
---

# External application integrations

Availability: implemented on the OSS `develop` branch; requires compatible deployment and explicit configuration. Do not assume an arbitrary hosted tenant has enabled these endpoints. This is not a production policy or Microsoft consent activation.

All routes below begin `/api/external-app/workspaces/:workspaceId`. Use a **current human Brian bearer** and `X-Correlation-ID`. Workspace paths are checked selectors, never caller authority. Machine credentials cannot impersonate the initiating human.

## Source-owned records

POST `records/:sourceId/publish`, `reconcile`, `observe`, `access`, `access-reconcile`.

Publish stable external IDs and monotonically versioned company/deal snapshots, optionally linking an existing canonical entity. Identical replay is safe; different content at the same version conflicts. Native CRM fields remain native metadata, not an editable copy of the application's approved facts. Reconciliation reports canonical merges and separate incoming observations; never turn an observation into an application approval automatically.

`publish` and `access` require `X-Source-Signature` in addition to human authorization. It is HMAC-SHA256 over canonical JSON `[workspaceId, sourceId, operation, correlationId, authorizationHeader, requestBody]`, with lexicographically sorted object keys and preserved array order. Only the owning application's server holds the signing secret. An ordinary native human caller cannot forge application-approved facts. Never expose that secret or persist a bearer in jobs.

Server config `EXTERNAL_APP_RECORD_SOURCES` controls workspace/source publisher/observer IDs, allowed application roles and `signingSecret`. Application access is requested/active/revoked for a mapped current workspace user; it does not create an account or remove Brian membership. Database migration 611 is required before use.

## Private documents

GET `documents/templates/:id/:version`; POST `documents/render`, `documents/export`, `documents/upload`, `documents/reconcile`; POST `documents/files/:id/read`; GET `documents/files/:id`.

Render exact template IDs/versions/hashes with typed scalar values and immutable source bindings. The existing Office exporters and LibreOffice produce actual DOCX/XLSX/PDF bytes; CSV/JSON exports retain their data. External uploads require byte/hash/container validation and a configured trusted scanner. Missing scanner/storage/template configuration fails explicitly.

The owning application must authorize source records and freeze the approved version itself. A render binding is not an approval grant. Artifacts remain custodian-private and unindexed, with stable binding/request digests for replay and lost-receipt reconciliation. A different request under one binding conflicts.

Download locators expire within 30 seconds and still require the correct current human bearer, scope and hash. They are **not public object URLs**. For employee payslips, the application must reauthorize the employee and proxy custodian-private bytes; never hand out the custodian token or broaden sharing.

Configuration: `EXTERNAL_APP_DOCUMENT_TEMPLATES`, `EXTERNAL_APP_SCAN_URL`, optional `EXTERNAL_APP_SCAN_TOKEN`, and existing FilesApi storage. Synthetic templates/test scanners are not approved customer layouts or production malware certification.

## Outlook calendar

PUT `outlook-calendar/:connectorInstanceId/configuration` with `{enabled:true,calendarId}`. POST `.../events/upsert` or `.../events/reconcile`.

Upsert input: `{stableKey,providerId,expectedVersion,event:{subject,body,start,end}}`. Create uses null provider ID/version; update retains the returned provider ID and exact ETag. Reconcile is read-only. Keep the original target calendar through uncertain outcomes. Lost write responses are unknown, not permission to create another event.

Only an exposed, personally owned connector is supported. OAuth and calendar configuration stay server-side; delegated Calendars.ReadWrite/offline_access consent must be obtained separately. Caller-supplied credentials, broad workspace-owner impersonation, unrelated-event overwrite, attendee invitations and recurring event creation are not implied by this API.

## Recovery

Commit local intent before network I/O. Preserve source version, correlation, initiating actor and idempotency/binding keys. Reconcile unknown provider outcomes before retrying; replace obsolete pending reminders/bookings rather than duplicate them. Verify live tenant behavior and compatible releases before rollout.
