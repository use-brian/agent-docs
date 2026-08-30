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
| Member JWT | Current user's workspace membership and role | The first-party UI or a member-authenticated integration |

Never put either key family in browser code. `sk_intake_*` is deliberately not
accepted by Brain MCP or member routes. It cannot list contacts, read
submissions, search the brain, mutate configuration, or call arbitrary tools.

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
