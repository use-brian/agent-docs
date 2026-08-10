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
| Google Calendar | Also exposes Google Tasks tools. |
| Gmail | **Send only** - one tool, `gmailSendMessage` (approval-gated, sends as the connected Google account or a verified "Send mail as" alias, and can attach workspace files as real MIME parts). It **cannot read, list, or search mail**: the OAuth grant requests `gmail.send` and nothing else, so no per-assistant tool grant unlocks reading. To read a mailbox use Company Email (IMAP) or Assistant Email below. |
| Notion | |
| Google Drive, Docs, Sheets & Slides | Connect with Brian for Google-enforced per-file access, or bring an Internal Google Workspace OAuth app for `drive.readonly`. BYO connections choose Entire Drive or up to 50 recursive root folders per Brian workspace. Folder scoping is enforced by Brian, not by Google OAuth. Brian builds a metadata-only search catalog in the background and deep-enriches a file version only after a useful content read. |
| GitHub | |
| Fathom | |
| Shopify | Full store operator surface: 22 reads, 17 writes behind approval, and 4 destructive verbs behind approval cards. This includes typed, privacy-preserving customer-segment preview and creation for campaigns. Optional ambient ingest: store events flow into the brain with a daily digest and can trigger workflows (OAuth-connected stores only). Connect per store via OAuth or a pasted Admin API access token (`shpat_...`); each store is its own connector instance. Order history is limited to roughly the last 60 days until Shopify grants the app extended access; customer PII fields may be null until the protected-data review clears. |
| Company Email (IMAP) | The user's own corporate mailbox over IMAP/SMTP - any provider, with Alibaba enterprise mail auto-detected from the address. Connect with the work email plus an app password (client security password); the credential is verified live before it is stored. Tools: `imapSearchMessages` (INBOX + Sent, threaded results), `imapGetMessage`, `imapSendMessage` (approval-gated, sends as the user), `imapSaveAttachment` (save one email attachment into the workspace as a file, then deliver it with `sendFile`; on request only, 45 MB max, which is exactly the messaging-channel document limit), `syncMailboxNow` (pull new mail into the searchable archive on demand), and `searchEmailArchive` (semantic recall over the opt-in full-mailbox archive). Multiple mailboxes can be connected; every tool takes an optional `account` (the mailbox email) and defaults to the primary (first-connected) one. Each archive is private to its owner. Distinct from Gmail (the user's Google account) and Assistant Email (the assistant's own address) - no lane substitutes for another. |
| Workspace Files | Built-in primitive; no external account. |
| Office | Built-in primitive; create, read, and revise Brian-native Documents and Presentations in the workspace. |
| Computer Use | Built-in primitive; a controlled browser (the user's own Chrome via the Use Brian extension for account-sensitive sites, or a cloud browser for public ones). Sends require approval. |
| Google Cloud Storage | Bring-your-own storage via a service-account key; exposes no assistant tools. |

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
- Connector tools only exist after the service is connected in Studio -> Connectors and enabled for the assistant in its Tools tab. Never reference a connector tool that has not been connected.
- On a selected-folder Google Drive connection, `googleDriveListFiles` searches Brian's active metadata catalog by file name and folder path. It does not run an account-wide Google full-text query, and reads outside the active folder snapshot are refused. Do not retry an out-of-scope file id; ask the workspace owner to update the Drive scope.
- Workspace Files, Office, and Computer Use are built-in primitives and need no external account; every other tool-exposing connector requires OAuth or a personal access token.
- If a file, office, or browser tool you expect is missing, the primitive may be switched off for that assistant. That is a configuration state, not a product limitation: the owner re-enables it on the assistant's Tools tab.
- "Every weekday at 9am..." style requests create a scheduled task (a cron job on the assistant), which is distinct from a workspace Task.

## Related

- [Tasks](./tasks.md)
- [Workflows](./workflows.md)
- [Channels](./channels.md)
