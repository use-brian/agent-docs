---
title: Channels
description: Where an assistant can be reached; workspace-owned web, messaging, and assistant-email surfaces.
tags: [concepts, channels]
canonical: https://usebrian.ai/docs/channels
---

> Human-readable version: https://usebrian.ai/docs/channels

Channels are where the assistant can be reached. They are owned by the workspace, not by individual assistants. You create or connect them in Studio -> Channels and assign each one to a workspace assistant. Web is always available; messaging platforms are opt-in and assistant email is provisioned as an inbox.

## Channels at a glance

| Channel | Setup | Groups | Voice | Auth |
|---|---|---|---|---|
| Web | Always on | N/A | Yes | Google |
| Telegram | BYO bot (or @use_brian_bot) | @-mention | Yes | Bot token |
| Slack | BYO bot | @-mention | N/A | OAuth + signing secret |
| Discord | BYO bot | @-mention or reply | N/A | Bot token |
| Feishu / Lark | BYO enterprise app | @-mention | Short audio | App ID + App Secret |
| Microsoft Teams | BYO Azure Bot | @-mention | N/A | App ID + secret + tenant |
| WhatsApp | BYO number (QR pairing) or Meta Cloud API | Opt-in per group (BYON) | Yes | QR scan or Meta credentials |
| WeChat | One BYO bot per person (QR pairing) | Not supported (DMs only) | STT only | QR scan, then link to a Brian account |
| WeChat desktop (self-host) | Personal-account bridge | @-mention in groups | Yes | Custom-channel token + QR scan |
| Assistant email | Provisioned inbox | Email threads | N/A | Managed AgentMail or BYO key |

## Web

Chat in the app at `app.usebrian.ai`. Always on, no setup. Streaming responses, file uploads, voice input.

## Telegram (BYO bot)

1. Open @BotFather on Telegram. Send `/newbot` and follow the prompts to name your bot.
2. Copy the bot token BotFather gives you.
3. Open Studio -> Channels -> Telegram and paste the token. The channel lives at the workspace level; you then route each Telegram message to one of the workspace's assistants.
4. Send `/start` to your bot. The first time you use it, send the 6-character link code from the wizard to bind your Telegram identity to your Use Brian account.

If you do not want to manage your own bot, the official @use_brian_bot works for the default assistant. It does Mini App OAuth and links your account automatically. BYO is required for custom branding or non-default assistants.

## Slack (BYO bot)

1. Go to api.slack.com/apps -> Create New App -> From an app manifest. Copy the manifest from Studio -> Channels -> Slack.
2. Install the app to your workspace and copy the Bot User OAuth Token (`xoxb-...`) and Signing Secret.
3. Paste both into Studio -> Channels -> Slack. Validation runs `auth.test` against Slack; you get a clear error if either is wrong.
4. DM the bot in Slack to verify. In channels, the bot replies only when @mentioned (configurable).

## Feishu / Lark (BYO enterprise app)

1. In the Feishu or Lark developer console, create a tenant-owned enterprise app and enable its bot.
2. Configure long-connection delivery for message, card-action, and reaction events, then publish or approve the app for the tenant.
3. Open Studio -> Channels -> Feishu / Lark, choose the China or international platform, and paste the App ID and App Secret.
4. DM the bot to verify. In groups, the bot replies only when @mentioned by default; mention gating, threaded replies, processing reactions, and sender access policy are configurable.

The channel sends proactive and scheduled messages, interactive confirmation cards, reactions, images, and files. Outbound files are capped at 20 MiB. Short audio is transcribed before the reply; long or costly media processing still requires the web app's estimate-and-confirm flow. Feishu China and international Lark use different API domains, so choose the platform that issued the credentials.

Interactive Feishu/Lark replies support streaming cards with a live tool-progress
timeline. Grant `cardkit:card:write` and publish/approve the app version to enable
CardKit. Without that permission, Brian falls back to editable rich-text posts.
The final approved answer replaces the progress card; private reasoning and
speculative text preceding tool calls are never streamed. Long answers,
attachments, and confirmation actions retain their normal follow-up delivery.

Passive group ingestion is separate from chat replies. A tenant administrator must grant `im:message.group_msg`, subscribe and publish the app for all group messages, and approve the change. Then a workspace owner or admin enables individual observed groups in Studio -> Events. Nothing is ingested from a group until that per-group opt-in is present. Feishu uses the same routing, scheduled digest, Episode, and brain-extraction pipeline as Slack; addressed messages stay on the interactive chat path so they are not distilled twice.

Brian matches a Feishu/Lark sender to their Use Brian account automatically when the Feishu email equals the sign-in email. This needs the app to hold a contact-read scope (`contact:contact.base:readonly` or `contact:contact:readonly_as_app`) and `contact:user.email:readonly`, published in a new app version; Studio -> Channels -> the Feishu channel shows workspace admins whether email matching is on and which scopes are missing. When the emails differ, open Settings -> Account -> Connected accounts -> Feishu / Lark (or the "You on this channel" section of the channel), generate a six-character code, and send it to Brian in Feishu/Lark. Until a sender is matched or connected, they are a guest: session history only, with no workspace access, tasks, or personal memory.

## WhatsApp Cloud API

Studio can connect a workspace-owned Meta WhatsApp Business Cloud API number. Provide the Meta access token, app secret, verify token, Phone Number ID, and WhatsApp Business Account ID. Studio validates the number, subscribes the app to the business account, and gives you the callback URL to register in Meta's WhatsApp webhook configuration; select the `messages` field there.

Cloud API traffic is direct messages only. Configure the sender access policy before relying on the number: access fails closed until a policy is chosen. Allowlisted external senders are isolated from workspace data, and connector-tool access is available only when that allowlist policy permits it. Authorized inbound messages can also start channel-triggered workflows, whether or not the channel sends an assistant reply.

## WeChat (BYO bot, QR pairing)

WeChat rides Tencent's iLink Bot API, the sanctioned personal-WeChat bot surface. Studio -> Channels -> WeChat shows a QR code; scanning it with a WeChat account issues a bot for that account (iLink may additionally ask for a pairing code shown on the phone).

**One bot per person, acting as a verified Brian account.** Each bot answers only the WeChat account that scanned its QR, and only after that person is linked to a Use Brian account in the workspace. Contacts do not share one bot. A team of twelve pairs twelve bots through one connector service.

- **QR transport vs verified identity.** The QR scan proves which WeChat account controls the bot; it does not say which Brian account acts. A turn runs only as a *bound* account, with that account's current membership, departments, clearance and connector grants. Billing stays with the workspace. There is no fallback to the workspace owner, the person who paired the bot, or a guest identity: an unlinked bot or an unknown sender gets a fixed notice and no model or tool call.
- **Two ways to link.** "Connect my WeChat" (any member) links the scanner to the signed-in account when they confirm. "Prepare for a staff member" (owners/admins) saves an unlinked connection; the staff member signs in, opens the shared link (`/w/<workspace>/studio/channels?channel=<channel>`), creates a code and sends `LINK <code>` to the bot from the WeChat account that paired it. The code is consumed before anything reaches a model or an archive.
- **One account per bot.** A different account cannot take over a linked bot: the current account disconnects it (Settings -> Account -> Connected accounts) or an owner/admin revokes the binding, and the new account links again. History never moves to the new account.
- **Revocation is live.** Disconnecting, revoking, re-pairing, or removing the person's workspace access stops an in-flight turn before its next private tool call or reply.
- **Existing bots must be linked.** Bots paired before this model have no linked account and answer nothing until their user links them.

Hard limits inherent to iLink -- set expectations before recommending this channel:

- The bot is its own WeChat contact. It does not act as the user's personal WeChat account.
- Direct messages only. Group chats are not delivered to bots.
- No chat history over the wire: the bot only receives messages sent to it
  after connecting. A self-hosted deployment can still import prior
  conversations from an authorized export into the
  [chat message archive](../self-hosting.md#chat-message-archive), which is
  what `searchChatHistory` reads — but iLink itself delivers nothing
  retroactively.
- One connection per bot account. Connecting the same bot elsewhere steals the session, and a bot connected twice is paused as a conflict until an admin disconnects the duplicate.
- Scheduled and workflow messages cannot be sent to WeChat; a bot answers only its linked person, live.

Inbound text, images, and files work; voice notes are understood when WeChat attaches its own speech-to-text. Outbound is text (a markdown subset renders in the WeChat bot chat).

The assistant cannot see or hear an attachment during the turn it arrives in — WeChat delivers images inline, but voice and video reach the model only as a note that something was sent. With the [chat message archive](../self-hosting.md#chat-message-archive) enabled, their contents are transcribed and described shortly afterwards and become searchable, so a follow-up question can be answered even though the original turn could not be.

## WeChat desktop bridge (self-hosted personal account)

This is separate from the sanctioned iLink bot above. A self-hosted operator
runs the WeChat Linux client and the open WeChat desktop bridge in containers,
then creates a Custom channel in Studio and pairs the user's personal account
by QR. The bridge mirrors direct messages and groups; by default, groups answer
only when the personal account is @-mentioned.

The account owner must personally scan the QR and confirm the login. Multiple
accounts may share an operator host only as isolated one-account stacks with
separate tokens, cursor state, volumes, ports, containers, and service units;
credentials and message data must never be pooled, and agent-wechat stays
loopback-only. This is not a multi-tenant WeChat service.

Inbound images and readable documents are available to the same turn. Short
voice notes are synchronously transcribed before the assistant answers. Video,
long media, and other files are archived first, described or transcribed in the
background, and become available through chat-history search. Attachment
recovery is durable: a not-yet-downloaded WeChat asset holds that chat's cursor
and retries until the provider returns bytes or an explicit terminal reason.
Once bytes are available, the bridge streams them into the message store and
persists that stored reference before posting the inbound message, so a bridge
restart cannot turn a recovered file back into a missing attachment. Outbound
documents work when the bridge reports its file-send capability.

Tool confirmation uses `yes`, `no`, `always`, and `never`, including prompts
from scheduled or workflow runs. For a long-running turn, the bridge may send
one fixed `Working...` status followed by the final answer. It does not expose
the internal SSE stream, model deltas, tool inputs, hidden reasoning, or
chain-of-thought.

## Assistant email

Studio -> Channels -> Email creates an address owned by the assistant, such as
`brian@agentmail.to` or an address on a verified workspace domain. This is an
email Channel, not an assistant-level connector. The inbox detail is its single
configuration home:

- **Handled by default** is required. That assistant answers mail without a
  sender-specific rule and can search the inbox. Reassigning the handler moves
  mailbox access immediately.
- **Who can receive replies?** can be Approved senders only or Anyone.
  Approved-only accepts workspace members and the listed exact addresses;
  other mail is filed into the brain and waits for review. Anyone accepts any
  human sender, subject to machine-sender, block, rate, and credit safeguards.
- Sender routing can map an exact email address to another workspace assistant.
  A new thread uses that rule and stays pinned to the selected assistant for
  the rest of the conversation. Everyone else uses the default handler.
- Every non-member is an isolated external guest, including approved contacts.
  They can converse in their current thread but cannot access workspace memory,
  files, skills, connected tools, mailbox search, or unrelated outbound email.
- Brain ingestion can be enabled or disabled for the inbox.
- Send email and create/schedule draft permissions are granted separately for
  the handling assistant. Both actions still require confirmation.

Do not tell users to configure AgentMail from Studio -> Assistants -> Tools.
The assistant page is not an authority for inbox routing or permissions.
Gmail and assistant email are separate identities: Gmail sends as the connected
human, while assistant email sends from the assistant's own address.


## Group chats

A group connected to a workspace through its channel (Slack, Telegram, Discord, Feishu/Lark, Microsoft Teams, or a BYO-number WhatsApp group) is **one shared workspace conversation**, a room. Everyone in the group shares it:

- The bot replies only when @mentioned (or replied to). Messages that do not mention it are still kept in the room as context, and may be saved to the team brain. A workspace owner or admin can switch saving off per group; the messages then stay context only.
- When the room starts, the bot posts a one-time notice in the group saying so. A Telegram bot in privacy mode only sees messages that mention it, and its notice says that.
- Group members who are not in the workspace are guests: they can ask the bot and get replies, labeled by their platform handle, and nothing is ever saved to a guest's memory.
- No one's personal memory is used in a group, including the person who asked.
- Usage is billed to the workspace, with the person who asked recorded.
- The room appears in the Chat app's Workspace list with the platform's badge. It is read-only on the web: reply in the group.

A group that is not connected to a workspace (for example the official @use_brian_bot added without a channel binding) keeps the older behaviour: each person gets their own session, and anonymous members get session-only context. The WeChat iLink bot has no group support; the self-hosted WeChat desktop personal-account bridge supports groups and defaults to @-mention gating.

Bot credentials and inbound sender policy do not prove who can read an outbound
group conversation. Public workflow/relay output can use the normal destination.
Before restricted output can be sent to a group or other unverifiable external
conversation, a workspace owner/admin must approve an exact audience binding for
that conversation, including its maximum clearance and allowed Teams/Projects.
The server records the approver/time and rechecks the binding, current admin role,
expiry and assistant routing at send time. Without that proof the result is
`delivery_audience_unverified` and no message is persisted or sent.

The same rule applies to interactive replies. The active channel connection,
current member/session authority and final evidence are rechecked before model
generation, streamed edits, persistence and the adapter send. Revocation stops
later output; an unverified group receives zero restricted tokens and no outbound
call.

### Pending: group clearance and shared upload receipts

Status: implemented in [OSS PR #433](https://github.com/use-brian/use-brian/pull/433),
merged to develop; awaiting deployment. Do not assume existing hosts already have this fix.

The approved group audience, shared across active bot installations, limits what
each bot can read and reply with. A Confidential bot can help an Internal group,
but its entire turn uses only data permitted by the group's limits and its own
Department/Project grants. Personal memory stays excluded. Stored history,
quotes, summaries and tools use the same limits; older restricted context does
not become readable after a policy downgrade. Telegram topic ownership remains
in force for passive participation.

New Telegram document uploads use one shared operation across bots and webhook
retries: one file write and one receipt attempt. Albums use their shared provider
album id and one aggregate receipt. A failed or interrupted claim is not replayed
automatically after an uncertain write or send; resending creates a new operation.

### Pending: room text and memory capture repairs

These follow-up repairs are implemented in [OSS PR #435](https://github.com/use-brian/use-brian/pull/435) and [hosted PR #453](https://github.com/use-brian/brian-platform/pull/453), merged to develop and awaiting deployment:

- Passive and addressed copies share one provider message row. Repeated webhook
  deliveries can start each addressed bot's turn only once; a passive copy does
  not consume its turn. Uncertain attempts are not automatically replayed.
- Group memory saves use shared workspace ownership at the room's limits.
  Explicit personal saves require a private chat. Actual private reads remain
  blocked; the delivery privacy guard is unchanged.
- Provider rooms use their consented capture pipeline for shared learning,
  instead of legacy compaction that reads personal memory. Room capture batches
  have a valid session parent, shared ownership and an attributed assistant.
- The original shared room and Telegram topic ownership remain. Existing
  history is preserved; these changes do not clean up old duplicate rows or
  rename the room's original assistant heading.

### Pending: cross-workspace Telegram admission

Follow-up [OSS PR #436](https://github.com/use-brian/use-brian/pull/436) and
[hosted PR #454](https://github.com/use-brian/brian-platform/pull/454) repair the
remaining gaps after #435/#453. They require conversation ownership
before ambient Telegram content can reach workflows, discussion snapshots,
shared rooms or file/recording intake. They preserve #432's topic exclusions,
same-workspace shared context, group clearance limits and deduplicated receipts.

Only active installations in the receiving workspace can establish ownership
through configured chat/topic scopes, exact surface routing or current
owner/admin group approval. Membership in the Telegram group, equal clearance,
a sender's linked account, seen-chat inventory and existing rooms are insufficient.
Self-mentions, self-qualified commands, own replies and valid scoped callbacks
admit only that interaction; bare commands grant no foreign-workspace access.
Quoted discussion roots require conversation ownership. The official shared bot
drops unaddressed group media before resolving the sender's workspace.

Status: implemented and verified locally; deployment pending. Existing copied
records are preserved and require separate supported recovery.

## Notes for agents

### Pending: Personal Telegram recordings and Project chat groups

Implemented locally on 2026-10-27; release, production migration and live
provider acceptance remain pending. Existing hosts may still use the shared-room
behavior above. Do not assume this recording policy is already active.

Telegram BYO group recording defaults to Personal for the authenticated person
who connected the bot. All admitted speakers are recorded with their actual
sender attribution. Historical connections require explicit owner confirmation;
missing or departed owners stop recording. Studio -> Channels lets that owner
stop recording or select Work, optionally in an authorized Project. A destination
change starts a new future-only session; earlier history keeps its audience.
Provider replay claims survive transcript deletion.

The initial path saves text and attachment-type placeholders. It bypasses legacy
group archives, workflow events, discussion capture and automatic brain/file
intake, including for Work destinations. Addressed attachments can be interpreted
for the immediate reply without a durable file-cache copy. Personal group replies
use current provider input and general knowledge, with no connected tools, private
memory or saved-history reuse. Work replies retain existing group audience checks.
Recording ownership never grants the group the owner's credentials or memories.
Stopping recording does not mute an explicitly addressed reply. Deleting a chat
removes its transcript, not separate archives, files or learned facts.

Work chat navigation groups authorized conversations under expandable Projects,
with General chats above them. New chat within a Project uses its canonical
scope; a Project name or sidebar heading grants no additional access.

### Current integration guidance

- Channels are workspace-owned. Connecting a bot does not attach it to an assistant until you route the channel to one.
- Assistant email is also workspace-owned, but its **Handled by default** assistant is required at creation. Manage default handling, access mode, sender routing, ingestion, and outbound actions on the email Channel itself.
- Messaging platforms are bring-your-own credentials: the user owns the bot, Use Brian is the brain. Web is the only zero-setup channel.
- In any group chat, expect a reply only when the bot is @mentioned. In a group connected to a workspace, expect one shared conversation for everyone in it: un-mentioned messages are kept as context, members outside the workspace are guests, and no personal memory is used or written.
- The official @use_brian_bot covers only the default assistant; routing a channel to any other assistant, or custom branding, requires a BYO bot.

## Related

- [Assistants](./assistants.md)
- [Workspaces & sharing](./workspaces.md)
- [Tools & connectors](./tools-and-connectors.md)

## Desktop bridge acknowledgement recovery

A running WeChat desktop bridge retries an unavailable outbox acknowledgement
before polling for more work. It keeps the completed send result and does not
resend text or files just to recreate that acknowledgement. A failed send also
keeps its original failed result while acknowledgement is retried.

This protection lasts for that worker process. Restart or multiple-worker
reconciliation is still pending. An expired outbox lease or a missing
acknowledgement does not prove that nothing was delivered; check the destination
before initiating another send.
