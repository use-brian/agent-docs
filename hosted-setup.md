---
title: Hosted setup
description: Sign in, create a workspace, verify a first conversation, and optionally add assistants, tools and channels.
tags: [getting-started, hosted]
canonical: https://usebrian.ai/docs/quickstart
---

> Human-readable version with screenshots: https://usebrian.ai/docs/quickstart

Hosted Brian runs the infrastructure. The user needs a browser and an email
address or Google account; no server installation or model API key is required.
For local installation, use [self-hosting.md](self-hosting.md).

## 1. Sign in

Open https://usebrian.ai/login. Choose Continue with Google, or enter an email
address and choose Send magic link. The email contains a link and a six-digit
code; either completes email sign-in. Google account sign-in does not grant
Gmail, Calendar or Drive access. Those are separate connector choices.

Success means arriving at workspace setup or an existing workspace. If email
does not arrive, check spam and the address; request a new link or code if expired.

## 2. Set up the workspace

Give the workspace a recognizable name and describe its purpose during
onboarding. Teammate invitations can be completed now or skipped until later.
An app-specific entry can add connection or plan steps. Open the workspace
avatar at the top of the left rail to verify the active workspace, switch
workspaces, invite members, choose Add workspace, or open Settings.

## Find your way around

- Home always opens Chat. Projects, Brain and Live are fixed rail destinations;
  Studio and Account remain near the bottom.
- The More apps ellipsis shows workspace-selected apps. Explore more opens
  Studio → Mini apps, where authorized owners/admins manage availability.
  Reorder menu changes only the person's order; up to two extra apps can be
  pinned. Neither preference enables an app or grants access.
- The sidebar button beside Back/Forward collapses the contextual panel; its
  right divider resizes it. Search workspace includes a Conversations filter.
- Inbox beside the app heading shows assistant replies and mentions. Theme and
  connection status are at the rail foot. Your work below a fresh composer
  opens requests and follow-ups needing attention.
- On phones, open the header drawer for the same destinations. App selection
  closes it. Audience and context controls remain available in the compact UI.

## 3. Start a conversation

Choose Home, then New chat. The Chat heading for personal conversations and
Work heading for workspace conversations open the Personal/Workspace menu.
Use the compact top-bar heading on phones or with the sidebar collapsed.
Opening a menu creates no chat. Review Department and Project before the first
message; existing conversations retain their original audience and context.

When multiple assistants are available, the assistant control beside Effort
chooses who answers. Effort directly opens tier choices. On phones the
assistant uses an avatar and the plus menu contains attachments and recording
options. These are separate controls, not a nested Chat options sheet.

Send one concrete request, such as “Help me plan the first week of our next
project.” Verify that Brian replies, then continue in the same conversation.
Ask anything opens the chat dock from other app screens.

## 4. Add an assistant when needed

The default assistant is enough to start. For a separate role, open
Studio → Assistants, create an assistant, name it, describe its responsibility,
and configure permitted tools. See [assistants](concepts/assistants.md).

## 5. Connect tools (optional)

Open Studio → Connectors. If the service is not listed, choose Add connector →
Browse directory and find it there. Select the service, choose Connect, and
complete its authorization or credential setup. Follow the Assistant Connectors tab link to review
allowed tools and approval requirements. An account connection and permission
to use its tools are separate. Verify the connection status and try a small read
request. See [tools and connectors](concepts/tools-and-connectors.md).

## 6. Add channels (optional)

Studio → Channels connects supported channels such as Telegram and Slack.
Supply the channel's credentials, assign an assistant, and send a test message
there. Web chat works without this step. See [channels](concepts/channels.md).

## Continue learning

Read [memory](concepts/memory-and-knowledge.md),
[workspaces](concepts/workspaces.md), and
[pricing and credits](operations/pricing-and-credits.md). A plan or credit
notice should be interpreted using the current pricing reference.

The human guide's screenshots show public hosted sign-in and shared controls
captured in a fictional local demo workspace. A drafted message or disconnected
provider screenshot is not evidence of a completed model call or installation.
