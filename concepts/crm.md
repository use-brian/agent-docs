---
title: CRM
description: First-party contacts, companies, and deals that the brain reads and writes the same way it reads memories.
tags: [concepts, crm]
canonical: https://usebrian.ai/docs/crm
---

> Human-readable version: https://usebrian.ai/docs/crm

CRM in Use Brian is people, companies, and deals: the durable graph of who the team talks to, where the relationship stands, and what it is worth. It is first-party, so the brain reads and writes contacts the same way it reads memories: same brain graph, no translation layer. Attio and HubSpot are sync targets, not the primary surface.

## Everyday workflow

1. **Record the relationship.** Ask Brian to save a contact and their company so future conversations have the right context.
2. **Track an opportunity.** Create a deal with its amount and expected close date, then keep the stage up to date.
3. **Connect the follow-up.** Link tasks and relevant memories to the relationship so the next action stays close to the context.

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

## Native aliases

CRM records use the Brain entity's native `aliases` array, separate from the
canonical display name. To teach a nickname for an existing contact, company,
or deal, resolve its explicit record id and use `noteAlias` with `entity_id`
and `alias`. CRM ids are entity ids. Use `splitAlias` to remove an incorrect
alias. These tools are available independently of the optional reclassifier
in both hosted and OSS editions.

Do not append an alias to the name, delete/recreate the contact, or substitute
a custom field or memory. A rename requires an actual canonical-name change
request. Confirm a saved alias from the mutation result's persisted `aliases`;
CRM get/list reads expose the same array. Contact and company searches,
collection search, and relationship pickers match aliases. An ordinary CRM
update preserves the id and relationships.

Aliases are retrieval evidence, not authority to merge people or choose a
person write target. Resolve ambiguity with the user. Conflicts never merge
records automatically. Alias reads and writes respect the caller's workspace
and access scope.

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

Entity mutations keep the authenticated user as the actor. An explicit viewer
context cannot substitute another user, and an assistant execution must keep the
same workspace. A mismatch is refused before any entity write.

Current member Team reach and clearance are checked for existing entity edits,
including user-only requests and composed writes. Alias conflict responses omit
inaccessible entity IDs. Generic sensitivity edits cannot lower the existing
classification: Review returns HTTP 409 with `scope_declassification_required`.
This is an audited-release requirement, not a stale revision; do not retry the
same downgrade with a refreshed revision. No fields or verification audit are
changed by that refusal. Departmental rollout remains gated while the remaining
writers, release workflow and complete derivation coverage are unfinished.


- Change a deal's stage through `setDealPipelineStage` using ids returned by
  `listCrmPipelines`; `updateDeal` rejects the stage field. Use
  `advanceDealStage` only for a known default-legacy pipeline value.
- Write `amount` in major currency units with its explicit ISO currency code,
  never minor units. Write `close_date` as a calendar date, not a timestamp.
- There is no delete: close a deal with a catalog stage in the `lost` category;
  remove nullable values through ordinary field updates.
- CRM rows are brain entities, so use the `links` param and `getEntity` to connect and retrieve related memories, tasks, and deals in one place rather than treating CRM as a separate store.

Typed contact, company and legacy deal edits check current member Team reach,
clearance and private visibility before reading the source attributes, including
when composed into a larger transaction. A hidden or held source returns the
same unavailable result before relationship validation. The final write still
requires mutation authority. Graph projection/close paths and the complete CRM record/custom-field surface
have not yet completed departmental isolation acceptance; runtime activation remains gated.

Stable provider-identity saves now resolve, create and bind within one transaction.
A binding failure or race rolls back the operation; it does not leave an unbound
person or publish a hidden target id. Upsert no-ops require current-member source
authority, and incoming sensitivity and Team/Project floors are retained.
Typed relationship targets must be currently readable and have the right kind.
Their sensitivity and Team/Project requirements join the destination. Fresh
records also inherit private user/assistant restrictions; existing records must
already carry a compatible private binding. Clearing a relationship
does not reduce those protections. Brian retains independent mutation reach and
sensitivity floors when forwarding CRM writes. A scope refusal is not permission
to omit a relationship or change workspace to bypass it. Audited release and full
derived provenance remain incomplete; runtime activation stays gated.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Tasks](./tasks.md)
- [Memory & knowledge](./memory-and-knowledge.md)


Managed CRM email is available through native tools and scoped backend commands.
Workspace CRM permission alone does not authorize a mailbox: the current account
send grant, purpose policy and every recipient's sendability still apply. Durable
receipts distinguish provider acceptance from confirmed delivery and preserve
uncertain outcomes for review. Association enablement is independent of these
generic CRM controls. See [managed delivery](../api/crm-operations.md#native-managed-delivery-tools).

Standalone contact and deal creation holds referenced records stable until its database transaction commits. A refused or failed commit rolls back the new record and emits no graph projection. Imported creation reuses its owning transaction. This guarantee does not yet cover every existing-record edit, concurrent membership change, or later reclassification of a source.

Selecting a primary deal contact now checks current source and reference access through the canonical CRM writer before changing participant flags. Contact identity, inherited protection and primary flags commit together; a denial or failed commit leaves both representations unchanged. Clearing the primary retains the deal's existing protection. Full HTTP record-edit atomicity remains outside this guarantee.

Secondary deal participant edits now check and lock the current deal and contact, preserve private visibility, and inherit contact protection before writing. Role edits preserve an existing primary flag. Removing the canonical primary contact clears it in the same transaction without lowering protection; a refused or failed command rolls back both relationship and deal changes. These checks also respect an ambient assistant mutation ceiling even if an explicit caller context is broader.

Participant lists recheck current access to both deal and contact, excluding held, retracted, retired or otherwise hidden contacts even on legacy broader deal links. Participant write endpoints return HTTP 403 with code `scope_operation_denied` and a generic administrator-review message for scope refusals. Do not remove a relationship or change workspace to bypass a refusal.

Custom-field updates now validate and lock the source, field catalog and supplied reference records within one transaction. Reference sensitivity and department/project protection carry into the destination; incompatible private references are refused, and clearing a reference does not lower existing protection. Even an empty patch requires mutation authority. Hidden references return a content-free scope refusal. This does not yet certify legacy references omitted from the patch, full record-creation composition or activity/delivery lineage.

Archive/unarchive and the legacy stage writer now admit and lock the source inside their transaction. The selected stage and pipeline must remain live through commit; unknown or archived IDs make no change. Record, custom-field, archive and stage HTTP handlers map scope refusals to the same content-free HTTP 403 recovery contract. Route-level history/events and the operations-service stage implementation still need their own atomicity and authority verification.

### Stage source authority

Stage commands authorize the deal before catalog validation or an unchanged replay.
Human/import callers require current workspace membership. Assistant/workflow calls
require a complete bound execution context. Unbound credential-only machine calls
currently fail closed; credential ownership does not substitute for member authority.
Member and CRM-key operations REST adapters return HTTP 403 with `scope_operation_denied`
and administrator-review guidance, without hidden source details. Successful stage,
activity, audit and outbox writes share one transaction. This does not certify
complete machine scope support or protected downstream history/event delivery.
Brian stage tools preserve `scope_operation_denied` with the same recovery guidance.

### Activity source checks

Appending CRM activity requires current source mutation authority, including current membership and the caller’s independent mutation scope. Read-only access to a record does not grant activity writes. The activity REST endpoint returns an unavailable response if the record disappears before the write, or a content-free `403 scope_operation_denied` with administrator-review guidance for scope refusal. Timeline and report history queries recheck the current source; inaccessible contacts cannot contribute mailbox address matches. Brian custom-field store writes and their activity history now share one transaction. Historical snapshot protection, mailbox classification and complete downstream event authority remain unfinished.

### Saved activity audience

New CRM activity rows capture their source's sensitivity, department/project requirements and private visibility when written. Timeline and report history require access to both that saved audience and the current source. Broadening a record's audience does not broaden its earlier activity. A caller cannot override or lower the saved protection. Existing activity is marked as legacy, and strict classification withholds it until audited history review is implemented. This does not yet certify audit/outbox history, mailbox classification, history release or downstream event authority.

### Event source admission

CRM event workflow starts require the recorded actor to read the saved and current source audience before the run input is stored. Replaying an idempotency key does not bypass current access. Run and step reads retain the event floor, including through copied prior outcomes. Execution refuses a source that becomes inaccessible and withholds an in-flight result if access changes. Review Department access and start a new run after resolving an access failure; do not automatically retry an operation that may have run. Legacy, aggregate and inventory-event classification is not yet certified for strict mode.

The event-delivery listing also requires a current member envelope. An integration credential's `crm.audit.read` grant currently receives HTTP 403 `not_authorized` with Department access guidance because persisted machine ceilings are not implemented yet. It cannot borrow the credential creator's role. Current owner/admin members can inspect minimized, terminal retirement receipts; these cannot restart delivery. The operations-audit list is a separate surface whose audience protection remains pending.
