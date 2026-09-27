---
title: Tools & connectors
description: Connectors expose third-party APIs as per-tool-governed capabilities; scheduled tasks let the assistant run jobs on its own.
tags: [concepts, tools]
canonical: https://usebrian.ai/docs/tools
---

> Human-readable version: https://usebrian.ai/docs/tools

Tools give your assistant the ability to do things, not just talk. Connectors expose third-party APIs (Google Calendar, Gmail, Notion, GitHub, and more). Scheduled tasks let the assistant run jobs on its own.

## Connectors (MCP)

Connect a service from Studio -> Connectors. Each connector exposes a set of tools (for example Google Calendar exposes `googleCalendarCreateEvent`, `googleCalendarListEvents`, etc.). After connecting, you decide which tools each assistant can use from the assistant's Tools tab. Workspace Files, Office, and Computer Use are first-party built-in primitives (no external account); most of the rest authenticate via OAuth or a personal access token.

### Official built-in connectors

| Connector | Notes |
|---|---|
| Google Calendar | Calendar only. Use `googleCalendarListEventColors` before marking an event, then pass an exact returned `eventLabelId` (preferred named label) or legacy `colorId` to create/update. The connector can apply or clear existing labels and colours, but it does not create or edit calendar label definitions. |
| Gmail | **Send only** - one tool, `gmailSendMessage` (approval-gated, sends as the connected Google account or a verified "Send mail as" alias, and can attach workspace files as real MIME parts). It **cannot read, list, or search mail**: the OAuth grant requests `gmail.send` and nothing else, so no per-assistant tool grant unlocks reading. To read a mailbox use Company Email (IMAP) or Assistant Email below. |
| Notion | |
| Google Drive, Docs, Sheets & Slides | Connect with Brian for Google-enforced per-file access, or bring an Internal Google Workspace OAuth app for `drive.readonly`. BYO connections choose Entire Drive or up to 50 recursive root folders per Brian workspace. Folder scoping is enforced by Brian, not by Google OAuth. Brian builds a metadata-only search catalog in the background and deep-enriches a file version only after a useful content read. |
| GitHub | |
| Fathom | |
| Shopify | Full store operator surface: 22 reads, 17 writes behind approval, and 4 destructive verbs behind approval cards. This includes typed, privacy-preserving customer-segment preview and creation for campaigns. Optional ambient ingest: store events flow into the brain with a daily digest and can trigger workflows (OAuth-connected stores only). Connect per store via OAuth or a pasted Admin API access token (`shpat_...`); each store is its own connector instance. Order history is limited to roughly the last 60 days until Shopify grants the app extended access; customer PII fields may be null until the protected-data review clears. |
| WordPress | Managed-content surface: `wordpressGetManagedPage` reads named text/image slots; `wordpressUpdatePageText` and `wordpressReplacePageImage` are approval-gated writes. Requires the OSS Use Brian Bridge plugin plus a WordPress Application Password. Site/theme code explicitly registers every writable slot; the connector cannot edit arbitrary posts, HTML, metadata, options, themes, plugins, or selectors. Image replacement uploads a new Media Library attachment, checks the current page revision and attachment ID, and retains the previous attachment for rollback. Each site is its own connector instance. |
| Google Search Console | Read-only over a bring-your-own Google service-account key that the workspace pastes (no Google OAuth): `searchConsoleListSites` (properties + permission level), `searchConsoleQuery` (clicks / impressions / CTR / position by `query`, `page`, `country`, `device`, `date`, `searchAppearance` over a date range, optional AND filters, `searchType`, paging via `startRow` / `nextStartRow`), `searchConsoleInspectUrl` (index verdict, coverage / crawl state, canonicals), `searchConsoleListSitemaps`. Every tool takes an optional `siteUrl`; omit it to use the property chosen at connect time, or pass one of the `siteUrl` values `searchConsoleListSites` returns verbatim (`sc-domain:example.com` and `https://example.com/` are different properties). The service-account email must be added as a Restricted user of each property in Search Console. No writes; unmetered. One instance per service account. |
| Company Email (IMAP) | The user's own corporate mailbox over IMAP/SMTP - any provider, with Alibaba enterprise mail auto-detected from the address. Connect with the work email plus an app password (client security password); the credential is verified live before it is stored. Tools: `imapSearchMessages` (INBOX + Sent, threaded results), `imapGetMessage`, `imapSendMessage` (approval-gated, sends as the user), `imapSaveAttachment` (save one email attachment into the workspace as a file, then deliver it with `sendFile`; on request only, 45 MB max, which is exactly the messaging-channel document limit), `syncMailboxNow` (pull new mail into the searchable archive on demand), and `searchEmailArchive` (semantic recall over the opt-in full-mailbox archive). Multiple mailboxes can be connected; every tool takes an optional `account` (the mailbox email) and defaults to the primary (first-connected) one. Each archive is private to its owner. Distinct from Gmail (the user's Google account) and Assistant Email (the assistant's own address) - no lane substitutes for another. |
| Workspace Files | Built-in primitive; no external account. |
| Office | Built-in primitive; create, read, and revise Brian-native Documents, Presentations, and Spreadsheets in the workspace. |
| Computer Use | Built-in primitive; a controlled browser (the user's own Chrome via the Use Brian extension for account-sensitive sites, or a cloud browser for public ones). Sends require approval. |
| Google Cloud Storage | Bring-your-own storage via a service-account key; exposes no assistant tools. |

### Google Maps location intelligence

Google Maps is a deployment-keyed first-party read capability, not a user connector and not an OAuth account. When the deployment sets `GOOGLE_MAPS_SERVER_API_KEY`, Brian can discover three canonical tools on demand:

| Tool | Use |
|---|---|
| `googleMapsSearchPlaces` | Find current places, businesses, addresses, and points of interest. |
| `googleMapsLookupWeather` | Read current, hourly, or daily weather for an address, Place ID, or coordinates. |
| `googleMapsComputeRoute` | Compute current walking or driving distance and duration between two locations. |

Maps results are current evidence, not durable workspace facts. Cite the Google source links returned with the result. Refresh hours, ratings, weather, and route duration rather than saving them to memory. A Google Place ID may be persisted alongside user-owned aliases or notes. The Maps tools never perform a Calendar, CRM, memory, or workflow write; use the separately governed write tool after the user chooses an option.

### Shopify campaign audiences

The Shopify connector exposes two purpose-built audience tools for campaign preparation:

| Tool | Class | Behavior |
|---|---|---|
| `shopifyPreviewCustomerSegment` | Read | Accepts `all_subscribers` or `product_buyers` plus up to 500 product IDs. Returns the generated Shopify segment query and aggregate `total_count`, never customer records or email addresses. |
| `shopifyCreateCustomerSegment` | Write | Creates or reuses an exact matching saved segment and returns its ID, name, canonical query, reuse status, and Shopify Admin URL. Requires the connector action grant and approval according to the assistant's tool policy. |

Both tools always add `email_subscription_status = 'SUBSCRIBED'`. Product-buyer audiences use Shopify's lifetime `products_purchased` predicate without a date constraint, so the tools do not need `read_all_orders`. They accept only the typed audience definition and never accept raw ShopifyQL.

The Shopify mini app's Campaign tab uses these tools to prepare a restock campaign package. It also creates the time-limited discount code, drafts editable copy, and lets the merchant choose a featured product photo while reviewing a live message preview. `shopifyListProducts` and `shopifyGetProduct` return `featured_image_url` and `featured_image_alt` when Shopify has a featured image. The prepared package carries the image URL for the merchant to add manually in Shopify Messaging. Shopify Messaging remains the final testing, scheduling, compliance, and send surface; the connector does not attach the image or send a Shopify Messaging campaign through the Admin API.

### Built-in primitives and their off switch

Workspace Files, Office, and Computer Use are built-in primitives: first-party connectors with no external account, no OAuth, and no credential. They are on by default for every assistant, and each can be switched off per assistant from the assistant's Tools tab. Studio -> Connectors shows them under a neutral "Built-in" pill with no on/off control there, because the switch is per-assistant.

Switching a primitive off removes its tools from that assistant entirely: the tools are absent from the toolset, not present-but-blocked, and the prompt text advertising the capability goes with them. The switch holds on every path - chat, messaging channels, the public API, scheduled work, and assistant-to-assistant calls. Per-tool allow/ask/block policy still governs whatever tools remain when the primitive is on.

## Per-tool policy

Three modes per tool:

| Policy | Behavior |
|---|---|
| Allow | Runs without asking. |
| Ask | Confirms in chat before running each time. |
| Block | Never runs. |

Read tools default to Allow; write and destructive tools default to Ask. You can change the defaults per assistant.

For Gmail, Company Email (IMAP), and Assistant Email send/draft tools, this configured policy is authoritative once the separate action grant also admits the tool. Ask freezes the exact recipients, subject, body, and attachments for confirmation; Allow executes without that pause; Block refuses. A sensitivity label or audit classifier may annotate the approval/audit record but does not add a hidden veto after admission. The action can still fail for an objective reason such as an unreadable attachment, size limit, invalid sender, expired credential, or provider rejection.

### Tool policy matrix

Defaults by tool class, all overridable per assistant from the Tools tab:

| Tool class | Example | Default |
|---|---|---|
| Read | List events, search Notion, fetch a URL | Allow |
| Write | Send email, create event, write a page | Ask |
| Destructive | Delete event, archive thread, drop a row | Ask |

## Write grants

Separate from per-tool policy, each assistant carries a write-grant list per connector. A write or destructive connector tool runs only if the assistant's owner granted that specific action (for example `githubCreateIssue`) in Studio -> Assistants -> Tools. Grants bind every caller of the assistant the same way: team members, scheduled tasks, assistant-to-assistant calls, and API calls all get the same decision. A fresh assistant has no grants, so its connector write actions are refused until granted. Read tools are unaffected.

A refused write returns an "action not granted" tool error naming the connector and action. Surface it to the user; only the assistant's owner can grant the action in Studio.

## Scheduled tasks

Tell your assistant when to run something: "every weekday at 9am, summarize the team's Slack" or "follow up with the Acme lead in two hours." Use Brian schedules a cron job that runs on its own session, executes tools, and delivers the result via your preferred channel.

Where to see them: scheduled work lives on the Workflow surface. Each scheduled workflow shows its cadence and last run; pause, edit, or delete it there. Jobs survive restarts.

## Not to be confused with workspace Tasks

Scheduled tasks are timed jobs. They fire on a cron and run an assistant turn. Workspace Tasks (see the Tasks page) are the brain primitive: durable forward-commitments the assistant tracks for you. Different things; an assistant can use one to remember to schedule the other.

## Notes for agents

- A tool that is `Block` never runs, and write/destructive tools default to Ask, so a write action may pause for user confirmation before it executes. Do not assume a write succeeded until the confirmation resolves.
- A connector write can also be refused with an "action not granted" error when the assistant lacks the write grant for that action. This is not a transient failure; retrying will not help. Tell the user which action needs granting in Studio -> Assistants -> Tools.
- For an admitted email send/draft, do not invent a second confidentiality-policy refusal. Follow the configured grant and Allow/Ask/Block decision; report only objective execution failures returned by the tool.
- Connector tools only exist after the service is connected in Studio -> Connectors and enabled for the assistant in its Tools tab. Never reference a connector tool that has not been connected.
- On a selected-folder Google Drive connection, `googleDriveListFiles` searches Brian's active metadata catalog by file name and folder path. It does not run an account-wide Google full-text query, and reads outside the active folder snapshot are refused. Do not retry an out-of-scope file id; ask the workspace owner to update the Drive scope.
- Workspace Files, Office, and Computer Use are built-in primitives and need no external account. Google Maps is a separate deployment-keyed read capability. Other tool-exposing connectors require OAuth or a personal access token.
- `createOfficeArtifact` requires an admitted template plus the outcome and audience. Its optional `additionalContext` accepts facts, constraints, examples, or reference URLs for that one artifact. Company website setup belongs to guided template creation and is not repeated in the artifact tool.
- `getOfficeArtifact` returns one bounded permission-filtered page of stable section, object, slide, theme, master, layout, worksheet, cell, and image target IDs. When it returns `nextTargetOffset`, read the next page with that offset before revising a later target. `reviseOfficeArtifact` requires at least one returned ID (or a target selected by the user in the editor) and applies only supported canonical commands inside that boundary. It can edit Document structure/formatting/page setup, Presentation objects/layout/ordering, and Spreadsheet values/formulas/formatting/sheet structure. If the artifact advances before the job runs, the validated command batch becomes a proposal instead of overwriting intervening edits. It does not control editor chrome or authorize export, sharing, sending, or publishing.
- If a file, office, or browser tool you expect is missing, the primitive may be switched off for that assistant. That is a configuration state, not a product limitation: the owner re-enables it on the assistant's Tools tab.
- "Every weekday at 9am..." style requests create a scheduled task (a cron job on the assistant), which is distinct from a workspace Task.

## Related

- [Tasks](./tasks.md)
- [Workflows](./workflows.md)
- [Channels](./channels.md)

## Per-assistant mini-app tool access

Users configure Page, Office, Browse, Tasks, CRM, and Feed under Assistant >
Tools > Mini apps. Each app has a master switch and separate Read and Write
tool-set switches. A disabled app or set removes its tools from the assistant's
available tool list and refuses stale calls. Turning the app back on preserves
its previous set selections. These controls are independent of workspace Home
navigation and do not grant connector credentials or bypass tool confirmations.
If an expected tool is unavailable, ask the user to enable the relevant app and
tool set for that assistant instead of retrying an unavailable tool name.

## Organization directory operations

Authenticated attended conversations can discover `getOrganizationChart` and
`updateOrganizationChart`. The read returns only the human actor's permitted directory.
The write uses the same typed commands and optimistic versions as Workspace >
Organization, requires current workspace owner/admin authority and per-call human
confirmation, and changes structure or directory visibility only. Department membership,
clearance and content access are separate operations. Billing-owner identity does not
authorize either operation. These administration tools do not currently accept unbound
programmatic credentials or autonomous/callee identities.

Workspace access operations: `inspectWorkspaceAccess`, `requestWorkspaceAccess`, and `manageWorkspaceAccess` use the same authority checks as Settings > Department access. They require a verified human in the current attended conversation. Read grants never confer mutation authority; decisions require the current reviewed policy revision and immutable request hash/version. Unbound programmatic credentials cannot invoke these administrative operations.

Administrators can refresh the independent reviewer of a pending access request using `manageWorkspaceAccess` with `access.request.assign`. Manager changes reevaluate pending reviewers. A stale review must be inspected again before deciding; an unavailable independent reviewer leaves the request pending and unassigned.


### Reviewed organization initialization

Owners/admins can inspect the `initialization` metadata returned by
`getOrganizationChart` and explicitly choose a person or assistant and one of their
direct Team assignments. `updateOrganizationChart` accepts `org.initialize.subject`
with `subjectId`, `kind`, `teamId`, `expectedRevision` and `expectedPolicyRevision`.
Confirm the selected subject, primary placement and whether a members-only root
unit must be created. Existing linked units are reused. Missing or stale assignments
return `organization_conflict`; refresh and review again. No primary Team is guessed,
no reporting/accountable relationship is inferred, and no data permission changes.
Setup metadata is absent for non-admin callers. The web path is Organization >
Use department assignments. This directory setup does not activate departmental
read grants or strict isolation.

### Departmental connector exposure

A departmental read grant does not by itself make that department's live
connector tools available. The server also requires mutation authority for the
connector's entire scope. A tool's read-only description is not an exception.
Do not retry through another connector or assistant to bypass unavailable scope.
Native OCR jobs retain their initiating mutation ceiling when resumed; older
jobs without that ceiling require a new preflight. Native CRM delivery checks
its independent mutation ceiling before preparing the provider request.
These checks are part of the isolation implementation; they do not indicate
that strict departmental rollout is available.
### Workspace file publication

`fileAppend` publishes a new version and returns its new file ID. The old object
remains unchanged for history. Use the returned ID or durable path for subsequent
operations. Concurrent edits or a changed source scope return a conflict: read the
current file before making a new edit; deleting the file is not conflict recovery.
An unconfirmed publication returns `file_publication_uncertain` with
`retrySafe: false`. Inspect the current file before retrying because the publication
may already have succeeded. Read-only departmental reach does not authorize an
append, metadata edit or deletion. Derived writes retain their accumulated scope
and sensitivity. These protections do not activate the full departmental rollout.

### Memory version IDs

A successful `saveMemory` update returns the successor ID. Use that ID for the
next edit; the previous ID identifies a retired version. Memory reads and save
results carry server-owned source evidence outside the model-facing data. Do not
supply source IDs, classification metadata or derivation fields in tool arguments
to claim authority. This source tracking does not indicate that strict departmental
activation or complete ordinary-save provenance is available.

### Classification review impact

Administrator classification previews include an immutable, content-free list of
known descendant memories. Inspect the saved review before applying it; Brian's
confirmation shows unique affected IDs and how many were already held. A changed
dependency set requires a fresh preview. Earlier previews without impact evidence
remain inspectable and cancellable but cannot apply. A selection may contain up
to 100 sources and affect at most 500 known memories; use smaller batches when
the limit is exceeded. Unknown lineage remains unverified, and these previews do
not enable strict departmental activation.

### Task mutation boundaries

Task creation and updates require mutation authority; a departmental read grant
alone cannot authorize them. Updates retain existing and known input sensitivity,
Team/Project restrictions and private visibility. Incompatible private sources
are refused. A duplicate create only reuses a task with matching content, scope,
author and recorded provenance; active Project admission still applies.
The server rechecks current member Teams for creation, deduplication, edits and
child moves, even without an assistant context. An older assistant envelope cannot
restore removed Team reach or clearance on a source edit. Inherited destination
Teams also require current membership.
Task updates return a successor ID. Use it for subsequent edits. If related
records cannot be updated under current authority, the transaction rolls back
and Brian asks you to refresh and review access before trying a new edit.
History checks each version independently. Server-owned task source evidence
tracks known inputs; it does not certify complete turn provenance or activate
strict departmental rollout.

## LinkedIn Feed publishing

LinkedIn publishing is a Feed capability, not a generic MCP connector. Choose an
explicit personal or company Page destination on the canonical draft. A personal
identity and several Pages can coexist; reconnecting does not replace the author
of an approved draft. Personal workspace publishing requires the connector
owner's explicit grant.

Use the exact converted preview, then confirm the whole revision and authorize
its public release before publishing. Publication tools accept the saved revision
and its preview hash, never arbitrary replacement text. Source, target or image
changes require new approval. Preserve all ordered images and alt text; link
posts use their explicit link metadata and optional thumbnail.

After an ambiguous send, inspect delivery status and check LinkedIn. Do not retry
with a new key. The observed post URL can be recorded as operator-confirmed
evidence through the same reconciliation command used by the UI.

Native newsletter editions remain manual: prepare rich/plain text and the ZIP
with cover/body images, publish in LinkedIn's editor, then explicitly record the
actual edition URL. Copy/export/opening the editor never marks an edition posted.
An optional promotional link post is a separate draft with its own approval.
Managed provider activation stays disabled until app approvals and credentialed
smoke tests are complete; capability responses are authoritative.

### Brain verification and deletion boundaries

Generic Brain verification/deletion and assistant-memory confirmation require current
source access and independent mutation authority. Read-only reach cannot confirm, delete or reject an entry.
An inaccessible, absent or foreign entry returns the same not-found response;
bulk deletion reports only entries actually deleted. A denied action creates no
verification receipt, rejection tombstone or goal change.

New verification receipts retain server-captured source restrictions. App-role
receipt and corresponding journal reads require both current source access and
that retained scope, so later declassification does not widen historical evidence.
Legacy unclassified receipts are not exposed by these policies. This is a bounded
implementation: full correction/rejection derivation, privileged audit consumers
and complete model-input provenance remain unfinished. Departmental read-grant
expansion and strict activation remain unavailable.


Departmental inspection accepts `explain` with optional `memberId`, `assistantId`,
`contextTeamId`, `contextProjectId`, `targetTeamId`, `action` (`read` or `edit`) and
`sensitivity`. Only administrators may select another member. The result explains
separate read/mutation ceilings and independent active grant paths; it does not
bypass resource visibility or authorize edits. Unknown and unavailable directory
references return the same error. The explanation includes current authorized assistant and Project choices. A null read/edit Team list means unrestricted Team reach; an empty list means General-only reach. Neither value bypasses other resource gates. `history:"events"` inspects the filtered audit
with the same `after` / `expectedPolicyRevision` continuation protocol as requests
and grants. Raw audit payloads and content are not returned. Both operations use
the current verified human and server policy, not an assistant owner's authority.


Workspace assistant clearance changes use the `assistant.clearance.set` command
(`assistantId`, `clearance`) through the existing saved command review and apply
protocol. Current workspace owner/admin or direct assistant-owner membership is
required again at application. The legacy assistant PATCH adapter accepts only a
clearance-only body with matching `X-Brian-Access-Review-Id` and
`X-Brian-Access-Review-Hash` headers; raw workspace clearance PATCH is refused.
A cancelled review makes no change. Saved-review replay is idempotent.


Access explanations in a running conversation are also bounded by that
conversation's server-held execution ceiling. Current human grants and a broader
selected assistant cannot widen its read, edit, sensitivity or Project limits.
The web preview describes current permissions; existing conversations can be
narrower. Independent human grant paths do not override the final intersection.


Use `inspectWorkspaceAccess` with `registry:true` for the department editor's
current registry snapshot (HTTP: `GET /api/workspaces/:workspaceId/access/registry`).
It contains only authorized department metadata, assignment references, visible
people/assistants and linked org units, plus current admin capability, revision,
expiry and the existing request-duration policy. It returns no emails, content or
hidden counts. `registry` cannot be combined with `explain` or history selectors.


Durable workspace-file text and byte reads recheck current authority and source
revision after fetching storage, before returning content. A revoked, changed or
unavailable source returns not-found. Doc-media GETs return authenticated no-store
bytes for every backend, including `?redirect=0`; clients must not expect a signed
storage URL. Images/downloads use a local object URL after the authorized read.

Cached durable doc-media displays must honor `X-Brian-Media-Valid-For-Ms`
(after subtracting the full request time) and purge on identity/authority changes.
This exposed response header bounds local display; it is not a storage capability
and cannot authorize a later read. The server rechecks the source revision and
current canonical read predicate before issuing it.

This lifetime also applies to Feed and campaign previews and to byte-only reads.
A durable media download starts on user action and verifies the same mounted
viewer/workspace, current cache ownership and unexpired admission after reading
the body. An invalidated or detached late response must not trigger a browser
download. Object URLs remain cache-owned so downloading does not revoke an
active preview.

Temporary file-cache previews use authenticated no-store byte reads with the same
current-source and display-lifetime checks. Both original and converted PDF reads
require `workspaceId` and authentication; retained IDs or old `sig` values do not
authorize them. The compatibility `/api/files/:id/preview-url` endpoint returns
`{url, requiresAuth:true}`, an authenticated locator, never a signed capability.
The browser uses separate original/PDF cache keys, purges on authority or identity
changes, and discards expired or detached results. Conversion rechecks source
revision and current permission before delivering bytes.
