---
title: Brain MCP Server
description: Connect an MCP client to a Use Brian workspace brain to read and write memory, tasks, CRM, files, and knowledge.
tags: [mcp, brain]
canonical: https://usebrian.ai/docs/api/brain-mcp
---

> Human-readable version: https://usebrian.ai/docs/api/brain-mcp

A Use Brian workspace brain is exposed as one MCP server over Streamable HTTP. An MCP client (Claude Code, Claude Desktop, ChatGPT connectors, or any custom client) authenticates with a workspace-scoped credential and receives the brain as tools: read and write memory, tasks, CRM, files, and knowledge. This is the page for an agent connecting itself to a user's brain.

## Endpoint

| Field | Value |
|---|---|
| Method + URL | `POST https://api.usebrian.ai/api/brain/mcp` |
| Transport | Streamable HTTP MCP |
| Protocol | `initialize`, `tools/list`, `tools/call` |
| Scope | One credential = one workspace |

Discover the tool set with a standard `tools/list` call: do not hardcode it. File tools appear only on deployments configured with file storage.

## Authentication

Send the credential as `Authorization: Bearer <token>`. Two credential formats are accepted; both resolve to the same workspace-scoped principal.

### API key

| Field | Value |
|---|---|
| Token shape | `sk_brain_<keyId>_<secret>` |
| Issued / revoked | Studio: Programmatic Access |
| Header | `Authorization: Bearer sk_brain_...` |
| Best for | Claude Code, scripts, servers |

The plaintext key is shown once at creation. Use Brian stores a hash and a prefix only.

### OAuth 2.1

For clients with a Connect button (claude.ai, Claude Desktop, ChatGPT connectors) that do not expose a raw header field.

| Field | Value |
|---|---|
| Grant | Authorization code + PKCE, `S256` only |
| Client onboarding | Dynamic client registration |
| Consent | User signs in, picks one workspace, approves the scope |
| Access token | `oat_<...>`, valid 10 minutes |
| Refresh token | `ort_<...>`, valid 30 days, rotated on every use |

Reusing a refresh token after rotation fails. `plain` PKCE is rejected.

### Per-credential scope

Each credential carries a `scope`. A `read` credential never sees the write tools: they are omitted from `tools/list` and rejected at `tools/call`.

| `scope` | Tools exposed |
|---|---|
| `read_write` (default) | read tools + write tools |
| `read` | read tools only |

Hand a `read` credential to a low-trust integration; use `read_write` for your own agent.

### Clearance and authority

Reads are clearance-filtered to the credential's ceiling: a credential never returns rows above its tier. Writes act with the workspace's default authority and carry the same permissions and audit trail as a chat turn. There is no separate, unaudited write path.

The doc-page tools follow the same rule inside teamspaces: a page that lives in a teamspace resolves by the credential's clearance against the teamspace's sensitivity (and the page's own clearance), never by any human account's teamspace membership. A page you expect that does not resolve usually means the clearance ceiling, not a missing page.

### Team and Project binding

An `sk_brain_*` key may be issued with one Team and/or one Project binding in
Studio: Programmatic Access. The binding is immutable for that key and is
intersected with the workspace primary assistant's grants on every request.
General rows remain available; other Team/Project rows do not. Ordinary
Project participation is not an ACL. A **shared Project** is: its material is
readable through a key only when the key's issuer and the workspace primary
assistant are both participants of that Project (or the issuer is the
workspace owner, which reads but never writes it), and a Team-bound key reads
no shared Project material. An empty read can be a shared Project's audience;
never infer that the material is absent.

There is no MCP argument or request header that selects or widens this context.
To switch or widen scope, a workspace admin must issue a new key. If the bound
Team or Project is archived or no longer available, the request fails closed
rather than falling back to company-wide access. Stable Team/Project ids may be
shown in management UI; raw compartment keys are never part of the MCP
contract. OAuth-issued Brain credentials are currently workspace-wide subject
to their normal clearance ceiling.

## Agent configuration

The Brain MCP control plane is bound to the workspace primary assistant. Its
read tools are discoverable normally; its configuration writes appear only
when a workspace owner or admin has enabled **Agent configuration** on that
assistant and the credential has `read_write` scope. Revocation takes effect on
the next request. The agent cannot grant this capability to itself, raise its
clearance, change billing, or approve a staged write.

Call `listConnectors` before configuring an integration. It returns both
visible configured instances and registry-backed catalog rows for official
connectors that are not connected yet. Catalog rows have
`availability: "available"`, `connected: false`, and `instanceId: null`; never
treat one as an active credential. `authorizationHandoff` says whether Agent
configuration can prepare the connector's browser-consent flow, and
`connectPath` identifies its workspace Studio destination.

For a supported OAuth connector such as Google Calendar, call
`requestConnectorAuthorization({ provider: "gcal" })`. If the provider is
already connected, the result identifies the visible instance and work can
continue. Otherwise it returns `human_authorization_required`, an absolute
`connectUrl`, and a canonical action id. Present the URL to the human. The human
still performs provider login, account selection, scope review, and consent;
OAuth codes and tokens never enter the tool result. Brain MCP is stateless, so
after the user reports completion, call `listConnectors` again and continue
only when a matching row is actually connected. Do not retry the authorization
preparation tool as a substitute for that verification.

Interactive Use Brian web chat uses the same preparation tool but can preserve
the live task through the consent detour: it renders a Connect action, verifies
the returned instance and workspace exposure server-side, and resumes the
original session. That automatic session resume is not an MCP push event.

## Connect Claude Code

One command. Replace the key with your own:

```
claude mcp add Use Brian --transport http https://api.usebrian.ai/api/brain/mcp --header "Authorization: Bearer sk_brain_..."
```

## Read tools

Available on both `read` and `read_write` credentials.

| Tool | Inputs | Returns |
|---|---|---|
| `searchBrain` | `query` (required); optional `scope` (`memory` \| `task` \| `contact` \| `company` \| `deal` \| `file` \| `kb_chunk` \| `entity` \| `file_segment`, single value or array, omit = all scopes); optional `limit` (default 20, max 100) | Unified retrieval across every primitive, clearance-filtered and scoped to the workspace |
| `listRecordings` | Optional `query` (title/file-name substring), `kind` (`memo` or `meeting`), `since` (inclusive ISO date/time), `until` (exclusive), `limit` (default 20, max 100) | Newest-first recording metadata: `recordingId`, `title`, `kind`, `status`, `occurredAt`, `durationMs`, `truncated`, `hasTranscript`. Includes queued and processing recordings before transcription finishes |
| `searchRecording` | `recordingId` + `query`, optional `topK` (max 20), or `fromIndex` / `toIndex` for sequential paging | Timestamped transcript passages inside one known recording. Resolve its id and status with `listRecordings` first |
| `searchFileContent` | `fileId` + `query`, or `fromIndex` / `toIndex` for sequential paging | Passages inside one stored document. Never returns the whole file |
| `getEntity` | entity `id` or display name; optional `walk_depth` / `walk_edge_types` | Entity rollup: existing relationship edges, recent episodes, memory, open tasks. Use it to resolve an entity's UUID and read existing edges before a linking write |
| `getMemory` / `getTask` / `getContact` / `getCompany` / `getDeal` | record `id` | Full record |
| `listTasks` | assignee / status / due range / tag / parent filters | Compact task projection |
| `getTaskExecution` | `task_id` or `execution_id` | The live execution of an assistant-executed task: phase, readiness and the smallest missing thing, next step, branches, remaining allowance and lifetime use, control revision, and external sessions reporting on it with their freshness (`live`, `stale`, `disconnected`) |
| `listResearchActivities` | optional `session_id`, `project_id`, `limit` | Saved research results the credential's issuer can see: status (`saved`, `partial`, `failed`), saved artifact, Project and date |
| `getResearchActivity` | `activity_id` | One saved research result: saved artifacts, sources marked read or only found, open gaps, follow-ups with their current label and version, and the lessons they support |
| `getResearchImprovement` | optional `improvement_id` | Research improvement candidates; with an id: pattern, desired behavior, linked task, the evidence you can see, and comparisons marked `local_verified` or `externally_reported` |
| `inspectApprovalReview` | `approval_id` | One approval request the issuer can see: status, who decided it (`human`, `delegated_assistant`, `standing_grant`), the assigned reviewer, the last review attempt, and a dry-run eligibility with reasons when it is still pending |
| `inspectApprovalPattern` | optional `pattern_id`, `procedure_id` | Learned approval patterns: lifecycle (`proposed`, `active`, `suspended`, `revoked`), version, distinct human approvals out of the 3 required, unresolved denial; with an id, the supporting requests the issuer can still read |
| `listContacts` / `listCompanies` / `listDeals` | CRM filters | Compact CRM projection |
| `listCrmFields` | optional CRM entity kind and record id | Live custom-field keys, types, options, and reference targets; a visible record id also returns its current values |
| `fileRead` | file `id` or path | Text content + metadata of one workspace file. **Text only** — on an image, PDF, or other binary it returns an error rather than a lossy decode, so do not use it to inspect a stored image. Nothing returns raw bytes: bytes leave the brain by being delivered (attached to an email, sent to a chat channel), never by entering a tool result |
| `fileSearch` | title / summary / tag / name query | Compact file projection |
| `getBrand` | optional `slug` (omit = the workspace default brand); optional `include_draft` | The workspace's brand record: naming and legal usage, strategy, messaging and voice, color tokens, typography, logo variants bound by workspace file id, applications, claims, rights, governance, sources. Returns the APPROVED record by default; `include_draft: true` returns unapproved changes, which are a proposal. Present only on deployments with the brand surface wired |
| `readPage` / `listPages` / `listPageTemplates` | page `id` or title; list filters; template catalog | Read doc pages. Present only on deployments with the doc surface wired |
| `searchKnowledge` | `query` | Deprecated alias for `searchBrain` with `scope: 'kb_chunk'`. Prefer `searchBrain` |

For a just-uploaded recording, use `listRecordings` even if `fileSearch` or transcript search has no hit. The catalog exists before transcription; `status` distinguishes `awaiting_upload`, `queued`, `processing`, `processed`, and `failed`. `awaiting_upload` reserves an identity but does not prove that uploaded media exists. Report queued/processing status while waiting for transcript content. The `/recordings` transcript file and transcript segments are created later, so their absence is not evidence that the recording is missing. Recording discovery observes the credential's workspace, visibility, clearance, Team and Project access.

## Write tools

Available only on `read_write` credentials.

| Tool | What it does |
|---|---|
| `ingestToBrain` | Capture content into the brain. `captureMode: "immediate"` (default) keeps the existing behavior: `decompose: true` runs full extraction and `decompose: false` files one distilled memory. `captureMode: "routed"` submits one message to the connection's configured assistant capture profile. Routed inputs require stable `eventId` and may include `occurredAt`, `sessionId`, `subjectId`, `role`, and bounded scalar `metadata`. A scheduled match is durably queued without an extraction-model call. Extraction profiles later process one pooled window; archive-only profiles preserve individual originals without extraction. |
| `saveMemory` / `deleteMemory` | Save one distilled workspace memory; soft-delete a memory by id |
| `deleteBrainRow` | Soft-delete one entity, task, KB chunk, contact, company or deal by id (history kept for audit). It deletes only a row the calling user and credential may currently change, including department access; any other id is reported as not found and nothing is changed |
| `saveTask` / `updateTask` / `closeTask` / `reopenTask` | Create, patch, and transition tasks. `executor_assistant_id`, `follow_up_member_id` and `home` set who executes, who follows up and the task's Department or Project home; `updateTask` `if_unassigned: true` claims only an unassigned task. See [Tasks](../concepts/tasks.md#executor-follow-up-and-home) |
| `controlTaskExecution` | `pause` an assistant-executed task (`execution_id`, `action`, `expected_revision` from `getTaskExecution`). `steer` and `continue` renew the shared allowance and require a person: from a credential they are refused with `human_intent_required` and change nothing |
| `correctResearchFeedback` | Relabel a research follow-up (`feedback_id`, `kind`, `expected_version`). The issuer's own follow-up gets the new label; anyone else's records a disagreement |
| `decideResearchLesson` | Approve, reject or retire a learned research lesson (`lesson_id`, `decision`, `expected_version`). A key never counts as a person, so company-wide methods cannot be approved from a credential: that refusal is `You are not allowed to do this here` and changes nothing |
| `nominateResearchImprovement` | Nominate a generic improvement candidate (`title`, `pattern`, `desired_behavior`, optional `feedback_ids`, `activity_ids`, `task_id`). Every cited id must be visible to the issuer. Describe the pattern generically; it records evidence and changes no engine |
| `recordResearchComparison` | Record an engine comparison on a candidate (`improvement_id`, `bundle`). A `traces` bundle (case manifest plus baseline and candidate traces) is recomputed and stored `local_verified`; a `reported` bundle is stored `externally_reported`. An arbitrary `passed: true` is rejected |
| `reviewApprovalRequest` | Ask the server to review one pending request as its assigned reviewer (`approval_id`, optional `explanation`). Only when the key's bound assistant is the reviewer a person explicitly assigned. The server decides: it resolves only an exact repeat of a pattern the approver approved on three distinct executions, never payments, transfers, account deletion, security changes or bulk deletes; otherwise it returns `needs_human` with reasons and changes nothing. Never report a request as approved unless this returns `resolved` |
| `assignApprovalReviewer` / `revokeApprovalPattern` | Always refused `human_intent_required` on a credential: delegating a person's approval authority, or revoking it, needs that person in the product |
| `bindExternalTaskExecution` | Bind your coding session to a task: `task_id`, a stable `external_execution_id`, `harness` (for example `codex` or `claude-code`). Idempotent. It does not make you the task's executor and gives Brian no control over your session |
| `reportTaskProgress` | Append progress for a bound session: unique `event_id`, increasing `sequence`, `milestone`, `state` (`running`, `waiting`, `blocked`, `claimed_done`), optional `summary`, `artifacts` (`kind` + `ref`, for example a branch or PR URL), `blocker`, `checks`. See "Reporting progress from a coding session" |
| `saveContact` / `updateContact` | Upsert / patch a contact |
| `saveCompany` / `updateCompany` | Upsert / patch a company |
| `saveDeal` / `updateDeal` / `advanceDealStage` | Create / patch a deal; move a deal through `lead` -> `qualified` -> `proposal` -> `negotiation` -> `won` / `lost` |
| `setCrmCustomFields` | Patch typed workspace fields on one visible CRM entity. Call `listCrmFields` first and use its stable keys |
| `fileWrite` / `fileAppend` / `fileSetMeta` / `fileDelete` | Create or overwrite a file with authored text, append text, patch metadata, delete a file |
| `saveFileToBrain` | Promote a previously uploaded cached file (a `fileId`) into the brain, preserving the original bytes |
| `saveFileBytes` | Persist a file from raw bytes supplied as base64, preserving the exact bytes. Size-capped; larger files use the HTTP upload route |
| `saveBrandDraft` | Propose a change to the workspace's brand record. Writes the DRAFT only. It does not take effect until a workspace owner or admin approves it in Studio, which creates a new approved version. Each field group you pass replaces that group whole, so call `getBrand` first and send the complete group back when adding to a list. Bind assets by workspace file id (upload with `saveFileBytes` first), never by path |
| `createPage` / `editPage` / `deletePage` / `createPageFromTemplate` | Author doc pages: create, edit, delete, or seed from a template. `editPage` accepts Markdown and applies it to the live collaborative document when doc-sync is configured, so an edit is immediately shared with open page editors; deployments without doc-sync use the legacy version-checked page store. Present only on deployments with the doc surface wired |

## Reporting progress from a coding session

When you work on a Use Brian task from an external coding harness, report progress so the team sees it next to the task:

1. Call `bindExternalTaskExecution` once per session with a stable `external_execution_id`.
2. Call `reportTaskProgress` at meaningful milestones (plan made, tests written, PR opened, blocked, done). Give each event a unique `event_id` and an increasing `sequence`; a repeated `event_id` is ignored and an older `sequence` is kept as history without changing the current state.
3. Put real references in `artifacts` and the smallest unblocker in `blocker`.

What reporting does not do: it does not make your session the task's executor, start or duplicate Brian's own work on the task, renew any allowance, change the Project's Brief, or complete the task. `checks` are stored as claims, and `claimed_done` is a claim, not completion. Brian cannot launch, stop or steer your session: the link only carries what you report. Your identity is the authenticated credential; names inside the payload are text.

`ingestToBrain` (default `decompose: true`) is the path for raw notes and documents: it derives entities and edges. `saveMemory` is for a single fact you have already distilled. Do not route task-shaped content through `saveMemory`: use `saveTask` or `ingestToBrain`.

### Project and Department Briefs

A shared Project or a Department can have a governed **Brief**: one Page whose
Direction (outcome, working rules, accepted decisions) changes only when an
owner explicitly applies it, plus Current context (findings, open questions,
references and progress) that participants and agents maintain.

| Tool | Use |
|---|---|
| `inspectBrief` | Read the accepted Direction, Current context, pending proposals (always pending, never accepted) and revisions. Pass `projectId` or `departmentId`; `create: true` creates an empty Brief. Empty Direction means not established: do not invent a goal from a name. |
| `maintainBriefContext` | Add attributable findings, questions or references with the source you inspected (`kind`, `id`, `version`, original `passage`). A repeat is a no-op; task and goal status comes from the record; a source the Brief audience cannot read is not written. Never changes Direction. |
| `manageBrief` | `propose` a Direction change with source evidence, `save_draft`, `dismiss`; owners `apply` (with the `expectedDirectionRevision` / `expectedSelectionRevision` from `inspectBrief`), `restore`, `select_departments`, `repair`; anyone who can write proposes an existing Page as direction with `adopt_page` (pending until applied). Governing actions are approve-banded: the human approves in the Approvals inbox and the command rechecks ownership. |

Rules for agents: propose, do not apply, unless the owner explicitly asked you
to apply that exact change. `editPage` on a Brief Page's Direction is refused
(`brief_direction_protected`); propose instead. Selecting a Department Brief
grants no access to Department sources. Errors are typed (`brief_owner_required`,
`brief_revision_stale`, `brief_not_found`, `brief_unavailable`, ...).

### Routed message capture

Use `captureMode: "routed"` for configured message capture. The profile chooses extraction or archive-only purpose; archive-only engineering capture never enriches operational context. The connection must first be assigned a capture assistant in Studio: Programmatic Access. The connection may inherit that assistant's default capture profile or override it. If there is no selected assistant or effective profile, the call fails closed; it never falls back to another assistant.

Profiles are workspace-owned and reusable. Each profile has one partition policy (`connection`, `user`, `session`, or `subject`) and an ordered first-match-wins rule list. Rules may match every event, keywords, actor, role, or scalar metadata and route to `drop`, `scheduled`, or `realtime`. Connection rules and assistant defaults are selected, never merged.

Each routed rule also chooses how its audience is bound. `inherit` uses the selected assistant's explicit or administrator-reviewed default. `explicit` uses the rule's exact Team and Project selection; an empty explicit selection intentionally means Workspace General for extraction. Engineering archive capture requires an explicit nonempty audience. In a strict workspace, a legacy or unavailable binding is preserved as a held queued receipt and is not extracted. A later configuration edit does not silently release that held payload.

For every routed call:

- Generate a stable `eventId` from the source message. Reusing it on a retry returns the existing receipt instead of appending twice.
- Send `occurredAt` as ISO 8601 with an explicit offset when the source timestamp is known.
- Send `sessionId` when the selected profile partitions by session, or `subjectId` when it partitions by subject.
- Treat `subjectId` and metadata as opaque routing/provenance labels. They do not select a workspace, assistant, or security principal.
- Do not set `decompose: false`; flat-memory mode is only available with immediate capture.

A queued result means the event was durably appended. It does not mean extraction ran or that the item is currently eligible to run. For extraction profiles, the cron window or the 32k-token size bound flushes an eligible batch keyed by workspace, assistant, rule, partition, and firing time. Brian orders that extraction window and makes one extraction call for the whole batch. Archive-only profiles use their dedicated per-event publisher instead; see the engineering capture contract below. MCP does not passively observe another product's transcript: the connected client must call `ingestToBrain` for each message it chooses to submit.

### CRM operations tools

When the deployment has CRM operations enabled, both scopes can receive:
`listCrmIntakeDefinitions`, `listCrmSubmissions`, `getCrmSubmission`,
`listCrmConsentPurposes`, `getCrmConsent`, `checkCrmSendability`,
`listCrmSegments`, `previewCrmSegment`, `listCrmEntitlementPlans`,
`listCrmEntitlements`, `listCrmEvents`, `listCrmParticipation`,
`listCrmPipelines`, and the business reference readers
`listCrmReferenceDefinitions`, `previewCrmReference`, `findCrmReference` and
`getCrmReferenceHistory`, and the commercial context readers
`listCrmCustomerAccounts`, `listCrmCommercialProfiles`,
`resolveCrmCommercialFields` and `listCrmOperationDefinitions`, and the
invoice readers `getCrmPreparation` and `listCrmInvoices`, and the governed
record readers `listCrmChangeRequests`, `getCrmChangeRequest`,
`listCrmEngagements`, `listCrmBillingRequests`, `getCrmBillingBalance` and
`previewCrmRecurringSuccessors`.

These catalog/program collection tools return a named array plus `nextCursor`.
Pass it as `cursor` with the same filters until it is null. `limit` is 1-100
(default 50); `created_after` and `created_before` provide inclusive/exclusive
creation-time bounds. Do not treat the first page as the complete catalog.

`read_write` may additionally receive `recordCrmSubmission`,
`updateCrmSubmission`, `recordCrmConsent`, `recordCrmSuppression`,
`saveCrmSegment`, `archiveCrmSegment`, `grantCrmEntitlement`,
`updateCrmEntitlement`, `recordCrmParticipation`, `updateCrmParticipation`,
`setDealPipelineStage`, `assignCrmReference`, `saveCrmCustomerAccount`,
`prepareCrmInvoice`, `reviseCrmPreparation`, `markCrmPreparationReady`,
`saveCrmChangeRequest`, `submitCrmChangeRequest`, `withdrawCrmChangeRequest`,
`updateCrmEngagementTracking`, `saveCrmBillingRequest`,
`cancelCrmBillingRequest`, and (each needing the user's confirmation)
`setCrmCommercialValues`, `commitCrmRecurringSuccessors`,
`recordCrmExternalInvoice` and `correctCrmExternalInvoiceReference`;
with the `configure` capability also `saveCrmReferenceDefinition`,
`replaceCrmReference`, `retireCrmReference`, `importCrmReferences`,
`saveCrmSellerProfile`, `acceptCrmCommercialProfile`,
`proposeCrmOperationVersion`, `issueCrmInvoice`, `retryCrmInvoiceIssue` and
`voidCrmInvoice`. A proposed operation version is not in force
until a workspace owner or admin accepts it in CRM. Reference writes carry an
`operation_id`: reuse the same id to retry an uncertain result. A reference
preview never reserves a number. Call catalog readers first and pass stable ids/keys;
unknown catalog values fail closed with valid choices. Treat a sendability
verdict as permission only when it is exactly `allowed`.

`sk_intake_*` is a separate write-only credential for one atomic external
submission route. Brain MCP rejects it. See
[CRM operations API](../api/crm-operations.md).

## Errors

Every HTTP-level refusal carries a `message` beside the `error` slug, so an agent can act on it without a docs lookup:

| HTTP | `error` | What the `message` tells you |
|---|---|---|
| 401 | `invalid_brain_key` | One uniform body for every authentication failure (missing header, malformed token, unknown key, revoked key, expired or revoked access token — deliberately not distinguished): send `Authorization: Bearer sk_brain_<keyId>_<secret>` or `Bearer oat_<authorizationId>_<secret>`; an `oat_` access token expires after 10 minutes and is renewed with `grant_type=refresh_token`; API keys are issued in Studio → Programmatic access. Retrying the same credential will fail the same way. |
| 500 | `brain_mcp_error` | Authentication already succeeded — do not go looking at your credential. Transient; retry once, then stop. |

Inside a tool call, a failure is a normal MCP tool result with `isError` and a text body that states what failed on which target, why, the next step (which sibling tool discovers a valid id, or "ask the user"), and whether the same input can ever succeed. Argument validation failures arrive as compact `path: message` lines, not a JSON blob. A workspace-less credential is refused with the remedy (a workspace admin re-issues or re-scopes the key) — no argument change helps.

## Notes for agents

- One credential reaches exactly one workspace. To span workspaces, obtain one credential per workspace.
- Routed capture requires a Studio-configured capture assistant and effective profile. Do not infer that ordinary Brain-MCP access passively captures the host agent's conversation.
- **There is no tool that approves a brand, and there will not be one.** `saveBrandDraft` writes a draft; a human with an owner or admin role approves it in the app. If you are delivering a brand system into a client's workspace, push the draft, upload the master assets with `saveFileBytes`, bind them by file id, and then tell the client to approve. Do not report the brand as changed before they do.
- Call `tools/list` after `initialize` and select from what is returned. If a write tool is absent, the credential is `read`-scoped; if file tools are absent, the deployment has no file storage; if the page tools are absent, the deployment has no doc surface.
- An empty read result may be a clearance filter, not an empty brain: the credential only sees rows at or below its tier.
- An empty read may also be the key's Team/Project envelope. Never infer that a hidden Project is absent or ask the caller for a raw compartment key.
- A `deleteBrainRow` not-found can likewise be an access boundary rather than a missing row. Re-resolve the id with the listed discovery tool; do not retry the same id or tell the user the record does not exist.
- Every write is audited exactly like a chat write. Assume no action is invisible to the workspace owner.
- Prefer `searchBrain` over the deprecated `searchKnowledge`.

## Related

- [MCP usage patterns](usage-patterns.md)
- [Pricing and credits](../operations/pricing-and-credits.md)
- [Self-hosting](../self-hosting.md)

## CRM-only integration credential boundary

`sk_crm_*` is accepted only at `/api/crm/integration/*`. It cannot authenticate
Brain MCP or acquire general tool access. Use a Brain credential for this
endpoint, or call the CRM integration resources with explicit operation and
resource grants. CRM integrations cannot enable modules or administer keys.

`listCrmSegments` also takes `limit`, `cursor`, `created_after` and
`created_before`; retain the same filters while following `nextCursor`.
`previewCrmSegment` exposes separate row and ID continuations: `nextCursor`
feeds `cursor`, and `snapshotNextCursor` feeds `snapshot_cursor`. Keep the
segment/filter selection unchanged; a segment edit rejects the old cursor.
`snapshot_limit` remains bounded at 1-10,000 per call. Follow the ID stream
to null for a complete audience, then recheck sendability at dispatch.

`listCrmEntitlements` accepts `active_only` and optional ISO `effective_at`.
Read raw `status` separately from `isEffective`; future-start and ended periods
provide no current access. Continue with the same filters and `nextCursor`.
Historical evaluation never authorizes present-day member pricing.


`grantCrmEntitlement` accepts `provider_period_id` and `predecessor_id` for explicit
provider renewals. Backend evidence authority is required for provider-backed
writes; ordinary human/assistant actions continue to manage manual grants.
`updateCrmEntitlement` extends an active grant in place. A terminal grant requires
a new provider period, not a status reset. The same provider period is idempotent
across transport keys, with changed business fields rejected. These tools do not
verify an external provider webhook or prove a notification was delivered.

## Native Association and catalog permissions

Association is an opt-in native app. The primary assistant needs association
and home_app:association:read or home_app:association:write. Contact-bearing
tools also need crm and its matching home_app:crm set. Module enablement and
Home selection never grant these capabilities. Read credentials cannot call
write tools even when the assistant has a write grant. Discovery and direct
invocation share the same gate.

The native catalog and exact object input envelopes are described in
[Association operations](../api/association-operations.md#native-association-tools).
Generic saveCrmEntitlementPlan/saveCrmEvent use {plan}/{event}, need configure
plus CRM write, and remain usable with Association disabled. No MCP tool can
enable the module or assert a provider payment.


Managed email uses native `sendCrmMessage` (CRM write set plus current mailbox
send authority) and `getCrmDelivery` (CRM read set). Both retain the authenticated
credential actor and bind the workspace's primary assistant. Read credentials
cannot send. Exact replay returns a content-minimized receipt; an uncertain
provider outcome must be reconciled under its original delivery id. See the
[CRM delivery contract](../api/crm-operations.md#native-managed-delivery-tools).

OAuth calls execute as the user who authorized the credential; Home-app calls execute as the authenticated viewer. Neither inherits the workspace owner’s private records or department access. A missing user identity is rejected. The workspace primary assistant and credential caps still constrain execution, while ownership remains billing/credential attribution. Brain keys retain their documented workspace-primary/owner binding.

Retained OAuth and Home-app calls revalidate their exact authentication before continuing canonical execution. OAuth token rotation, expiry, grant revocation or client revocation invalidates the retained call. Home-app token expiry or changed app grants/clearance does likewise. A Home-app token’s signed clearance cap also limits a later expanded app grant; current reductions narrow it immediately. Durable child credentials additionally require the parent binding described below.

Read-write OAuth clients can use `previewCrmCredentialBindings`, `listCrmCredentials`, `createCrmCredential` and `revokeCrmCredential` when the current user is an owner/admin and the primary assistant has configure and CRM write grants. Authentication supplies the exact parent descriptor; no tool input can provide or override it. Child expiry is capped to the authenticated access token expiry, and rotation, expiry or revocation of that token or its client invalidates child use. Preserve the issuance request UUID across uncertain retries; the secret is returned only on first success. Unattended issuance remains unavailable. See the [credential lifecycle contract](../api/crm-operations.md).

A retained Brain-key invocation also checks its original context and admission envelope. Replacement context, configuration-session or admitted department/Project values invalidate that invocation; a previously authenticated call cannot adopt the replacement authority.

Read-write Brain keys also support the four native credential lifecycle tools under current owner/primary-assistant authority and configure/CRM write grants. The child retains the exact parent key, owner, verifier fingerprint, cap and admitted context. Parent revocation, rotation or changed authority invalidates child admission. Model input cannot choose a parent or acting owner; Unattended delegation remains unavailable.

Read-write Home-app bridge tokens support the native credential lifecycle as the authenticated viewer, subject to current owner/admin membership and primary-assistant configure/CRM write grants. Child expiry cannot exceed the exact bridge token expiry. App grant/cap changes, viewer role loss and server signing-key rotation invalidate child admission. Parent evidence is captured by authentication; callers cannot choose it, and no bridge bearer or signing secret is persisted.

Home-app tokens expire at `exp <= now`. Invalid or out-of-range signed expiry timestamps are rejected before delegated credential evidence is constructed.

## Opted-in engineering capture (archive-only)

First discover `getCaptureConfiguration` and `engineeringHistory` on the target endpoint. Runtime deployment and collector package release/installation are separate from source implementation and fixture verification. No client should infer availability from this documentation alone.

Configure the existing capture profile in Studio: Programmatic Access with `processingMode: archive_only`, an explicit Department and optional Project audience, and the submitted-prompt/final-reply role filter. Select a capture assistant on the credential; its profile override wins over the assistant default. Missing/disabled/unavailable configuration fails without falling back to extraction or General. Existing profiles default to `extract` and new keys do not opt in automatically.

`getCaptureConfiguration` returns only this authenticated connection's `connectionId`, effective `processingPurpose`, `ready`, profile/revision, permitted event kinds, audience labels/references, limits, `contractVersion: 1`, `receiptContractVersion: 1` and `localEmitterObserved: false`. It contains no secret or workspace-wide profile registry. Read-only credentials cannot capture.

A deterministic collector calls `ingestToBrain` using `captureMode: routed`, a stable `eventId`, exact nonempty `content`, source `sessionId`, `role: user | assistant`, optional source time and allowlisted observed metadata. The server chooses purpose and authority. One selected event becomes one original native File and one identity/lexical projection. Archive-only processing calls no extraction/embedding model and creates no operational memories, tasks, Goals, Briefs, progress or learning proposals. Provider authorship/result claims are unverified source evidence, never approval, assigned work or completion proof.

Interpret MCP `structuredContent`, not prose, using the version-1 archive receipt. `processingPurpose` must be `archive_only`. A queued/duplicate-pending receipt has pending delivery/index state and is not searchable success. An archived receipt has `receiptStatus: completed`, `deliveryStatus: archived`, `indexStatus: ready`, `archiveId` and `fileId`. Reconcile retries with the same event identity; changed content under an existing identity fails `capture_event_conflict`. A held receipt requires configuration review. Excluded/deleted tombstones prevent replay resurrection. Schema/state mismatches are failures, not acknowledgements.

The `engineeringHistory` tool and authenticated `/api/workspaces/:workspaceId/engineering-history` HTTP service share the same native File/source authority:

| Action | Arguments / result |
|---|---|
| `list`, `search` | Optional `query` object: lexical `query`, `provider`, `sessionId`, `profileId`, `keyId`, `since`, `until`, opaque `cursor`, `limit` (1-100; default 50). Returns admitted ordered `events`, `nextCursor` and server-index health. Filters never grant access. |
| `read` | Server `eventId` (archive UUID, distinct from source event ID). Returns exact original text, provenance, annotations, event `revision` and `fileVersion`. |
| `export` | Same admitted query, bounded JSONL `jsonl`, `count`, truthful `status`, `nextCursor` and optional `failureCode`. `partial` is continuation or a partial failure, not complete delivery. |
| `annotate` | `eventId`, `text` (1-4000), `expectedRevision`, `expectedFileVersion`, UUID `idempotencyKey`. Appends attributed annotation without rewriting the original. |
| `exclude`, `delete` | Same version/idempotency arguments, without text. Native mutation/erasure authority applies. Exclusion retracts history; deletion erases the canonical File and projection. |

Read-only keys discover only read actions on this tool. HTTP uses actual web-user authentication; admitted nonadmin collaborators can read history independently of the admin configuration gate. List/search/excerpts/counts/cursors/read/export honor current workspace, Department, shared Project, credential/source and private restrictions. Owner oversight does not release restricted originals. Cursors are bound to viewer, workspace and query. Annotation/deletion retries must reuse the same command identity and expected versions; changed commands conflict. Errors are nonrevealing (`history_not_available`, `history_write_denied`, `history_cursor_invalid`, `history_version_conflict`, `history_command_conflict`, `history_storage_unavailable`).

HTTP routes are `GET /`, static `GET /export`, `GET /:eventId`, `POST /:eventId/annotations`, `POST /:eventId/exclude`, `DELETE /:eventId`. JSONL HTTP responses expose `X-Brian-Export-Status`, optional `X-Brian-Next-Cursor` and `X-Brian-Export-Error`. Each selected event is bounded at 256 KiB. Exact export pages have at most 100 events. Search uses a bounded lexical projection; exact reads/export resolve native File bytes. Missing or revoked rows are withheld before projection. Ordinary Files discovery/general retrieval excludes archive purpose.

The OSS `@use-brian/engineering-capture` source package provides `brian-capture setup|hook|worker|flush|status`. Setup emits inspectable plans and preserves unrelated hooks/trust; it installs nothing. Hook adapters select only documented `UserPromptSubmit.prompt` / `Stop.last_assistant_message` fields for Codex and Claude Code, return neutral output, queue owner-only state outside repositories and wake a leased MCP worker. System/tool/reasoning/subagent/transcript/attachment content is excluded. Missing final output, interruptions and unsupported schema are coverage gaps. Codex turn identity is used when supplied; without a documented immutable provider identity, distinct submissions receive distinct durable local IDs. Text alone never deduplicates them. Secrets are resolved only from an environment-variable reference.

Local defaults are 1000 pending events, 64 MiB pending bytes, 256 KiB selected text, 12 failed delivery attempts, backoff up to 5 minutes, a 30-second lease and a 10-second tool-call timeout. Offline/transient/uncertain failures retain WIP and stable identity; permanent authentication/configuration/source denial suspends. Explicit `flush --resume --accept-configuration` follows inspected configuration changes and does not substitute credentials/destination. Local status reports historical observed receipts and emitter diagnostics separately from server index availability; no capture is not proof of no engineering work.

Package publication, actual opted-in hook installation/trust, production migrations/deployment and installed Codex desktop/CLI/Claude Code interruption/offline/privacy acceptance remain live operations. Offline goal setting, benchmark construction/runs and coding control are outside this feature.

Queued originals remain pending until canonical publication. Removing a profile/rule with undelivered originals returns `capture_pending_archive_review_required`. A current member can explicitly discard only their own pending server archives through the Studio recovery control or authenticated `DELETE /api/workspaces/:workspaceId/programmatic-capture-profiles/pending-archives` with `{ "idempotencyKey": "<UUID>" }`. A repeated command affects the same selection; it returns only a content-free acknowledgement. Local collector WIP is managed separately.

## Routine configuration review

`getRoutineIntake` reads source-filtered configuration and proposals. Configure-gated `proposeRoutineIntake` saves inactive drafts. `applyRoutineIntake`, `rejectRoutineIntake` and `setRoutineIntakeEnabled` stage exact revisions for actual human review on agent surfaces; credentials and delegated/learned approvals cannot activate routines. See [the Workflow command contract](../concepts/workflows.md#configured-routine-task-intake).
