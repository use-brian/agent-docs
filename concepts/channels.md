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
| Microsoft Teams | BYO Azure Bot | @-mention | N/A | App ID + secret + tenant |
| WhatsApp | BYO number (QR pairing) or Meta Cloud API | Opt-in per group (BYON) | Yes | QR scan or Meta credentials |
| WeChat | BYO bot (QR pairing) | Not supported (DMs only) | STT only | QR scan |
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

## WhatsApp Cloud API

Studio can connect a workspace-owned Meta WhatsApp Business Cloud API number. Provide the Meta access token, app secret, verify token, Phone Number ID, and WhatsApp Business Account ID. Studio validates the number, subscribes the app to the business account, and gives you the callback URL to register in Meta's WhatsApp webhook configuration; select the `messages` field there.

Cloud API traffic is direct messages only. Configure the sender access policy before relying on the number: access fails closed until a policy is chosen. Allowlisted external senders are isolated from workspace data, and connector-tool access is available only when that allowlist policy permits it. Authorized inbound messages can also start channel-triggered workflows, whether or not the channel sends an assistant reply.

## WeChat (BYO bot, QR pairing)

WeChat rides Tencent's iLink Bot API, the sanctioned personal-WeChat bot surface. Studio -> Channels -> WeChat shows a QR code; scanning it with a WeChat account binds a new bot identity (iLink may additionally ask for a pairing code shown on the phone).

Hard limits inherent to iLink -- set expectations before recommending this channel:

- The bot is its own WeChat contact. It does not act as the user's personal account; contacts must message the bot.
- Direct messages only. Group chats are not delivered to bots.
- No chat history: the bot only sees messages sent to it after connecting.
- One connection per bot account. Connecting the same bot elsewhere steals the session.

Inbound text, images, and files work; voice notes are understood when WeChat attaches its own speech-to-text. Outbound is text (a markdown subset renders in the WeChat bot chat).

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

In Telegram and Slack groups, the bot only responds when @mentioned. Anonymous group members get session-only context. Chat works, but no personal memories are written about them. WeChat has no group support at all (DMs only).

## Notes for agents

- Channels are workspace-owned. Connecting a bot does not attach it to an assistant until you route the channel to one.
- Assistant email is also workspace-owned, but its **Handled by default** assistant is required at creation. Manage default handling, access mode, sender routing, ingestion, and outbound actions on the email Channel itself.
- Messaging platforms are bring-your-own credentials: the user owns the bot, Use Brian is the brain. Web is the only zero-setup channel.
- In any group chat, expect a reply only when the bot is @mentioned, and expect no personal memory to be written for anonymous group members.
- The official @use_brian_bot covers only the default assistant; routing a channel to any other assistant, or custom branding, requires a BYO bot.

## Related

- [Assistants](./assistants.md)
- [Workspaces & sharing](./workspaces.md)
- [Tools & connectors](./tools-and-connectors.md)
