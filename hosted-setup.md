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
An app-specific entry can add connection or plan steps. Verify the correct
workspace name appears in the upper-left corner. Existing users can open that
workspace menu to switch workspace or choose Add workspace.

## 3. Start a conversation

Open Chat in the sidebar. Start a new chat, check Personal or Workspace scope,
and send one concrete request, such as “Help me plan the first week of our next
project.” Verify that Brian replies, then continue in the same conversation.
Ask anything opens the chat dock from other screens.

## 4. Add an assistant when needed

The default assistant is enough to start. For a separate role, open
Studio → Assistants, create an assistant, name it, describe its responsibility,
and configure permitted tools. See [assistants](concepts/assistants.md).

## 5. Connect tools (optional)

Open Studio → Connectors, choose a service, select Connect, and complete its
authorization flow. Then follow the Assistant Connectors tab link to review
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
