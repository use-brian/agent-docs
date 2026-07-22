---
title: Channels
description: Where an assistant can be reached; workspace-owned surfaces (web always on, Telegram and Slack bring-your-own bots).
tags: [concepts, channels]
canonical: https://sidan.ai/docs/channels
---

> Human-readable version: https://sidan.ai/docs/channels

Channels are where the assistant can be reached. They are owned by the workspace, not by individual assistants. You connect once at the workspace level (Studio -> Channels) and route each channel to one of the workspace's assistants. Web is always available; messaging platforms are opt-in and bring-your-own credentials.

## Channels at a glance

| Channel | Setup | Groups | Voice | Auth |
|---|---|---|---|---|
| Web | Always on | N/A | Yes | Google |
| Telegram | BYO bot (or @sidanclaw_bot) | @-mention | Yes | Bot token |
| Slack | BYO bot | @-mention | N/A | OAuth + signing secret |
| Discord | BYO bot | @-mention or reply | N/A | Bot token |
| Microsoft Teams | BYO Azure Bot | @-mention | N/A | App ID + secret + tenant |
| WhatsApp | BYO number (QR pairing) | Opt-in per group | Yes | QR scan |
| WeChat | BYO bot (QR pairing) | Not supported (DMs only) | STT only | QR scan |

## Web

Chat in the app at `app.sidan.ai`. Always on, no setup. Streaming responses, file uploads, voice input.

## Telegram (BYO bot)

1. Open @BotFather on Telegram. Send `/newbot` and follow the prompts to name your bot.
2. Copy the bot token BotFather gives you.
3. Open Studio -> Channels -> Telegram and paste the token. The channel lives at the workspace level; you then route each Telegram message to one of the workspace's assistants.
4. Send `/start` to your bot. The first time you use it, send the 6-character link code from the wizard to bind your Telegram identity to your sidanclaw account.

If you do not want to manage your own bot, the official @sidanclaw_bot works for the default assistant. It does Mini App OAuth and links your account automatically. BYO is required for custom branding or non-default assistants.

## Slack (BYO bot)

1. Go to api.slack.com/apps -> Create New App -> From an app manifest. Copy the manifest from Studio -> Channels -> Slack.
2. Install the app to your workspace and copy the Bot User OAuth Token (`xoxb-...`) and Signing Secret.
3. Paste both into Studio -> Channels -> Slack. Validation runs `auth.test` against Slack; you get a clear error if either is wrong.
4. DM the bot in Slack to verify. In channels, the bot replies only when @mentioned (configurable).

## WeChat (BYO bot, QR pairing)

WeChat rides Tencent's iLink Bot API, the sanctioned personal-WeChat bot surface. Studio -> Channels -> WeChat shows a QR code; scanning it with a WeChat account binds a new bot identity (iLink may additionally ask for a pairing code shown on the phone).

Hard limits inherent to iLink -- set expectations before recommending this channel:

- The bot is its own WeChat contact. It does not act as the user's personal account; contacts must message the bot.
- Direct messages only. Group chats are not delivered to bots.
- No chat history: the bot only sees messages sent to it after connecting.
- One connection per bot account. Connecting the same bot elsewhere steals the session.

Inbound text, images, and files work; voice notes are understood when WeChat attaches its own speech-to-text. Outbound is text (a markdown subset renders in the WeChat bot chat).

## Group chats

In Telegram and Slack groups, the bot only responds when @mentioned. Anonymous group members get session-only context. Chat works, but no personal memories are written about them. WeChat has no group support at all (DMs only).

## Notes for agents

- Channels are workspace-owned. Connecting a bot does not attach it to an assistant until you route the channel to one.
- Messaging platforms are bring-your-own credentials: the user owns the bot, sidanclaw is the brain. Web is the only zero-setup channel.
- In any group chat, expect a reply only when the bot is @mentioned, and expect no personal memory to be written for anonymous group members.
- The official @sidanclaw_bot covers only the default assistant; routing a channel to any other assistant, or custom branding, requires a BYO bot.

## Related

- [Assistants](./assistants.md)
- [Workspaces & sharing](./workspaces.md)
- [Tools & connectors](./tools-and-connectors.md)
