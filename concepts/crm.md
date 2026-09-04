---
title: CRM
description: First-party contacts, companies, and deals that the brain reads and writes the same way it reads memories.
tags: [concepts, crm]
canonical: https://usebrian.ai/docs/crm
---

> Human-readable version: https://usebrian.ai/docs/crm

CRM in Use Brian is people, companies, and deals: the durable graph of who the team talks to, where the relationship stands, and what it is worth. It is first-party, so the brain reads and writes contacts the same way it reads memories: same brain graph, no translation layer. Attio and HubSpot are sync targets, not the primary surface.

## Entity-backed, not a separate store

CRM is not a separate database. Every contact (person), company (organization), and deal (opportunity) is a node in the same brain graph as your memories and tasks. The universal fields remain fixed, while each workspace may add bounded typed fields: text, number, date, boolean, single-select, multi-select, or a reference to another visible CRM entity. `tags` remain lightweight labels on contacts and companies; they are not a substitute for structured deal or relationship fields. `external_ref` is a free-form JSONB passthrough for synced rows.

## Workspace fields

Call `listCrmFields` before reading or changing a workspace-specific dimension. It returns the live field keys, entity kind, type, select options, required state, and allowed target kinds for reference fields. Pass `record_id` to include the current custom values of one visible contact, company, or deal. Do not invent a field key or infer one from its label.

Use `setCrmCustomFields` to patch one visible contact, company, or deal by entity id. Values are validated against the current catalog. A reference value must be the id of a visible entity whose kind is allowed by the definition. When a call rejects an unknown key, refresh the catalog rather than retrying a guessed vocabulary.

## Deal stages

Deal stages come from the workspace's live pipeline catalog. Call
`listCrmPipelines` and use its stable pipeline and stage ids.
`setDealPipelineStage({ dealId, pipelineId, stageId })` is the canonical write
and validates that the stage belongs to the selected pipeline. Stage category
is `open`, `won`, or `lost`; names and keys are workspace-defined.

The default pipeline retains the legacy `lead`, `qualified`, `proposal`,
`negotiation`, `won`, and `lost` keys for compatibility. `advanceDealStage`
works only against that default legacy catalog and delegates to the same
canonical operation. Do not treat those six values as universal.

## Amounts and dates

`amount` is a decimal major-currency value, not minor units. Users type
`50000`, not `5000000`, for fifty thousand units. Use the deal's explicit ISO
currency code. `close_date` is a calendar date; "Q3 close" is not a wall-clock
instant. Reports group amounts by currency and never silently convert them.

## Chat tools

Every assistant with the `crm` capability gets the record CRUD surface plus the
catalog-backed operations allowed for that assistant. `updateDeal` does not
accept a stage. Use `listCrmPipelines` followed by `setDealPipelineStage`.
`advanceDealStage` is a default-pipeline compatibility tool. There are no
delete tools in v1; close a deal by selecting a stage whose category is `lost`
and clear nullable fields through the normal update tools.

`saveContact` / `getContact` / `listContacts` / `updateContact` · `saveCompany` / `getCompany` / `listCompanies` / `updateCompany` · `saveDeal` / `getDeal` / `listDeals` / `updateDeal` / `advanceDealStage` · `listCrmFields` / `setCrmCustomFields` · `listCrmPipelines` / `setDealPipelineStage`

The broader operational tool catalog covers typed submissions,
consent/suppression and sendability, shared segments, entitlements, events, and
participation. Discover it at runtime and follow [CRM Operations API](../api/crm-operations.md)
for the credential and closed-world catalog contracts.

## Canonical email drafts

When the runtime exposes `saveEmailDraft`, save the complete envelope, body,
and `attachments` list before presenting or revising an email draft. Each
attachment is a saved workspace file id or absolute path, with at most 10 per
draft. Include the complete list on every revision; `[]` removes all
attachments. Omit `draft_id` to revise the conversation's active draft, or use
`start_new: true` for a different email. `getEmailDraft` returns the exact saved
revision, including its attachment references.

Saving a CRM draft does not create a provider draft, request send approval, or
send email. When the user asks to send, pass the saved references to the
available email send tool, which resolves the files and applies its normal
access, size, and approval checks. A photo saved in the workspace is attached
to a draft only after its reference is included in that draft's saved list.

## Accrued client contacts

Identified end users arriving through the public API accrue a contact automatically: the first identified turn materializes a person entity paired to the integration's `externalUserId`, visible to the workspace team like any other contact. The client side is write-only: an end user can never read the contact that describes them, nor anything another end user's turns wrote (every client turn's writes are walled into a per-client compartment). See [Identity & memory](../api/identity.md).

An identified request carrying a verified email may also send `clientLead: { key, name? }` to atomically
ensure that exact client contact and one linked deal at the lead stage. The
server owns the stage and client compartment; retrying the same key for the
same API-key client identity does not create a duplicate.

## Relationships across the brain

Every CRM row is an entity in the underlying graph. Save a memory about a contact, link a task to a deal, or open an explicit edge via the universal `links` param: the entity rollup (`getEntity`) returns the contact, every memory anchored to it, and the deals it is attached to in one call.

## Notes for agents

- Change a deal's stage through `setDealPipelineStage` using ids returned by
  `listCrmPipelines`; `updateDeal` rejects the stage field. Use
  `advanceDealStage` only for a known default-legacy pipeline value.
- Write `amount` in major currency units with its explicit ISO currency code,
  never minor units. Write `close_date` as a calendar date, not a timestamp.
- There is no delete: close a deal with a catalog stage in the `lost` category;
  remove nullable values through ordinary field updates.
- CRM rows are brain entities, so use the `links` param and `getEntity` to connect and retrieve related memories, tasks, and deals in one place rather than treating CRM as a separate store.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Tasks](./tasks.md)
- [Memory & knowledge](./memory-and-knowledge.md)
