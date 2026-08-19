---
title: Self-Hosting
description: Run the open-source Use Brian core locally with ChatGPT sign-in or your own model credential, plus what the hosted product adds.
tags: [operations, open-source]
canonical: https://usebrian.ai/docs/open-source
---

> Human-readable version: https://usebrian.ai/docs/open-source

The Use Brian core is open source: the brain, the agent, workflows, channels,
content planning, and the doc surface. Run it on your own machine with an
eligible ChatGPT subscription or your own model credential, or let the hosted
service operate it for you.

## License

AGPLv3, OSI- and FSF-approved open source with a network-copyleft clause: run a modified Use Brian core as a hosted service and you publish your changes. A commercial license is available for orgs that cannot accept AGPL. Every contributor signs a CLA.

## Prerequisites

- Node 22+
- pnpm 10+
- Either an eligible ChatGPT subscription or one supported model credential.
  The launcher offers ChatGPT sign-in first; a free Gemini API key
  (`aistudio.google.com/apikey`), Vertex AI, and DashScope are also supported.
- `ffmpeg` (which also supplies `ffprobe`). Every recording path shells out to
  it before transcription runs, so without it uploaded recordings and voice
  notes sent through Slack, Discord, or Telegram fail at runtime rather than at
  install time. The `deploy-brian` kit installs it for you.
- The optional [chat message archive](#chat-message-archive) has its own
  prerequisites — a second PostgreSQL database and two extensions. It is off
  unless you configure it.

## Quickstart

```bash
git clone https://github.com/use-brian/use-brian.git
cd use-brian
pnpm install
pnpm dev                    # choose ChatGPT or an API-key backend; opens your browser
```

There is no step three.

## Production single-machine deployment

For a systemd-managed installation with PostgreSQL, encrypted secrets, and a
locally managed Cloudflare Tunnel, use the
[`deploy-brian`](https://github.com/use-brian/deploy-brian) deployment kit.
It provides one-shot targets for Debian 12/13 and Ubuntu 25.04/25.10. Configure
the tunnel and DNS first, then clone the kit on the server, fill the target's
`client.conf` and owner-only `private/` inputs, and run its `install.sh`.

The WeChat channel (Tencent iLink bot, QR pairing in Studio) is opt-in on
self-host: set `WECHAT_CONNECTOR=true` in `client.conf` and add
`WECHAT_CONNECTOR_SECRET` to the secrets file. The kit then runs the connector
as a loopback service; it needs no public hostname. Without the toggle, the
WeChat tab in Studio reports pairing as unavailable.

Production uses three explicit public origins on that tunnel: `APP_HOST` sends
all application traffic directly to Next.js, `API_HOST` sends API and provider
webhook traffic directly to the backend, and `DOCSYNC_HOST` sends collaboration
WebSockets directly to document sync. Protect only `APP_HOST` with interactive
Cloudflare Access; `API_HOST` and `DOCSYNC_HOST` use Brian/provider
authentication and must remain reachable by non-browser clients. Public sharing
under `APP_HOST/share/*` and `APP_HOST/api/public/*` uses specific Access Bypass
applications. The deployment kit's `shared/cloudflare.md` is the complete DNS,
ingress, Access, and verification contract.

Ubuntu 25.04 is supported as a transition path only because it is end-of-life;
use a maintained Debian release or Ubuntu LTS for a durable production host.

### Optional My Browser relay

The Debian and Ubuntu targets can run the single-instance browser relay needed
for **My Browser**, which lets an assistant use a paired Chrome extension on the
operator's workstation. Set `BROWSER_RELAY_HOST` in `client.conf`, add
`BROWSER_RELAY_SECRET` to the encrypted SOPS secrets, and create a DNS route for
that hostname through the deployment's Cloudflare Tunnel. The relay listens on
loopback `:8092`; only the tunnel publishes it. Do not put the relay hostname
behind interactive Cloudflare Access because the extension connects directly
to `/ext`. If `BROWSER_RELAY_HOST` is omitted, the relay remains disabled.

## Storage

The store defaults to an embedded PGLite database under `~/.usebrian/`: nothing
to install or run besides Node. Point `DATABASE_URL` at a local Postgres if you
prefer a container.

## Chat message archive

Self-hosted only, and off by default. `brian-message-store` keeps a searchable
archive of the messages your channels carry, including the attachment bytes, so
an assistant can answer from what was actually said months ago rather than only
from the current conversation. It adds the `searchChatHistory` and
`listChatChannels` tools.

The hosted service does not run one. Chat history is a self-hosting capability,
and the deploy scripts deliberately leave it unset.

### What it needs

- **Its own PostgreSQL database.** Not the brain's. The archive applies its own
  migrations, and pointing it at `DATABASE_URL` would put its schema inside the
  platform's database — which is the coupling this service exists to avoid.
- **A dedicated role that is not a superuser.** Startup refuses a superuser
  connection and exits. This is not a precaution to work around: superusers
  bypass row-level security *even when it is FORCED*, so every owner's messages
  would be readable by every query.
- **`pgvector` and `pg_trgm`, installed by a DBA.** The service role
  deliberately lacks `CREATE EXTENSION`. The migration checks for both up front
  and stops with an actionable message rather than failing halfway through on a
  bare permission error.
- **A disk volume for attachments.** Bytes are content-addressed on disk, never
  in the database. This grows with media, not with message count, so size it
  against the photos and voice notes your channels carry.
- **`ffmpeg`, `ffprobe`, and a SILK decoder** on the service's `PATH` if you
  want voice notes and video to be searchable by their content. WeChat voice
  notes are SILK-encoded and need `silk_v3_decoder`; video needs `ffmpeg` for
  frame sampling and audio extraction. A missing binary fails at exec time, per
  attachment, which reads as "extraction is stuck" rather than "a dependency is
  absent" — check these before debugging anything else.

```bash
BRIAN_MESSAGE_STORE_DATABASE_URL=postgres://archive_user:...@localhost:5432/archive
BRIAN_MESSAGE_STORE_HMAC_SECRET=...      # shared with the platform
BRIAN_MESSAGE_STORE_MEDIA_ROOT=/var/lib/brian/media
```

Unset `BRIAN_MESSAGE_STORE_DATABASE_URL` and the launcher skips the archive
rather than starting it against the wrong database.

### What it gives you

Messages are searchable the instant they commit, by keyword. Semantic search
follows within about a tick as embeddings are computed in the background —
appending a message never waits on a model call, because the raw message is the
one copy that cannot be fetched again from the provider.

Attachments are searchable by what they *contain*: text in an image, speech in
a voice note, what is visible in sampled video frames, and the text of a
document — Word, Excel and PowerPoint (`.docx`/`.xlsx`/`.pptx`), OpenDocument,
RTF, PDF, EPUB, CSV and plain text are all read. A file is identified by its
contents rather than by the type its provider claims, because some channels
label every attachment generically; sending a spreadsheet still finds a
spreadsheet.

Some files cannot be read: Apple Pages, Numbers and Keynote, the pre-2007 binary
Office formats (`.doc`/`.xls`/`.ppt`), and anything corrupt or password
protected. These are still archived and still findable by filename — only their
text is missing, and the assistant says so and suggests exporting to a readable
format rather than pretending the file is empty. Search results likewise say plainly when part
of the corpus is not yet embedded rather than reporting a partial answer as a
complete one.

Channel history can be imported from an authorized export, so the archive can
cover conversations that predate the connection. Import runs offline against
files you already have; it never attaches to a live account or bypasses a
provider's encryption.

### Limits worth knowing

Attachment bytes are served over an authenticated loopback endpoint. The
service binds to loopback by default, and splitting it from the platform across
hosts requires an explicit opt-in — the port serves raw personal messages and
files.

Deleting a workspace on the platform does not cascade into the archive through
a foreign key, because the two databases cannot reference each other. Deletion
is an explicit signal plus a reconciliation sweep. Budget for the archive
outliving anything you delete until that sweep runs.

## Local-first guarantee

The brain, the store, and the canvas all run on your machine. Model requests go
only to the backend you configure. Connectors and upgraded search providers make
outbound calls only when you opt into them; your local database and files are
not moved to a Brian-hosted service.

## Model backends

| Backend | Status | Authentication |
|---|---|---|
| Gemini via Google AI Studio | Supported | `GEMINI_API_KEY` |
| Gemini via Vertex AI | Supported | GCP workload or service-account credentials |
| Qwen / DeepSeek via DashScope | Supported | `DASHSCOPE_API_KEY` |
| Custom OpenAI-compatible endpoint | Supported | Optional bearer key entered in Settings |
| Claude Haiku outage fallback | Optional fallback | `ANTHROPIC_API_KEY` |
| ChatGPT / Codex subscription | Beta, OSS only | Sign in with ChatGPT; no API key |

ChatGPT-plan access uses Codex-managed OAuth and the live model catalog for the
authenticated account. It does not treat a ChatGPT token as an OpenAI API key.
Brian remains the agent harness and owns memory, context, tool policy,
confirmations, execution, and persistence. ChatGPT/Codex quota and plan limits
remain OpenAI's authority.

For a custom OpenAI-compatible backend, add the endpoint connection once in
**Settings -> Models**. Then create verified model profiles on that connection
and optionally assign separate profiles to Brian's Standard, Pro, Max, and
Research tiers. Each profile has its own wire model id and explicit context and
output limits. An unassigned tier keeps the deployment's normal model routing;
Brian never silently falls back from an explicitly selected or tier-assigned
custom profile after an upstream failure.

## Local Support Mode

The OSS edition includes an opt-in Support Mode under **Settings → Privacy**.
An owner can capture one hour, 24 hours, or seven days of bounded, sanitized
local diagnostics, preview the categories, and download a JSON support capsule.
Nothing uploads automatically. Conversation and tool content are excluded unless
the user explicitly enables them for the selected readable session. Stopping,
expiry, or a successful download hard-deletes the capture rows.

## Tool governance defaults

Tools are governed by what they do, fail-closed:

| Action | Default |
|---|---|
| Reads (search, list, fetch) | Allowed |
| Writes (send, create, update) | Ask first, until you tell it "always" for one |
| Destructive (delete, revoke, cancel) | Blocked until enabled per tool |

A fresh install reads and drafts freely but cannot send an email or delete an event without you. Policy is set per tool in the app.

## Optional connector keys

One configured model backend is the floor. Each key below is optional; nothing
turns on by itself. Set them in `.env` or under `~/.usebrian/`.

| Capability | Key(s) | What you get |
|---|---|---|
| Web search | `BRAVE_SEARCH_API_KEY`, `TAVILY_API_KEY`, or `SERPER_API_KEY` | Upgrade search past the keyless DuckDuckGo fallback |
| Page fetches | `JINA_API_KEY` | Cleaner reads via Jina Reader (works keyless at lower limits) |
| Read X / Twitter | `TWITTER_BEARER_TOKEN` | Read x.com permalinks through the official X API v2 |
| X search | `XAI_API_KEY` | xAI Grok fallback plus the `xSearch` tool |
| Google Maps | `GOOGLE_MAPS_SERVER_API_KEY` | Place search, weather, and walking/driving routes through Maps Grounding Lite; use a dedicated server-only key restricted to that API |
| Model fallback | `FALLBACK_PROVIDER_ENABLED=true` + `ANTHROPIC_API_KEY` | Keep running if Gemini is unavailable |
| Google connector | `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Calendar, Gmail, Drive via your own OAuth app |
| Notion connector | `NOTION_CLIENT_ID` / `NOTION_CLIENT_SECRET` | Notion via your own OAuth app |
| Fathom connector | `FATHOM_CLIENT_ID` / `FATHOM_CLIENT_SECRET` | Fathom via your own OAuth app |
| GitHub connector | Personal Access Token (entered in the UI) | GitHub, no env key needed |

Connector client id / secret can also live in
`~/.usebrian/connectors.config.json`. Every key is documented in `.env.example`.

## What the hosted product adds

| Open core | Hosted platform adds |
|---|---|
| Agent engine, brain, memory, knowledge | Managed database, upgrades, backups |
| Channels, workflows, doc surface | Plans, credits, team billing |
| Content planning, approval inbox, manual ready-to-post queue | Provider OAuth, automatic publishing/deletion, inbound ingest, platform insights |
| MCP server and public API | Monitoring, abuse protection, support |

The hosted product also gives every new workspace a 30-day Pro trial. See [Pricing and credits](operations/pricing-and-credits.md).

## Notes for agents

- A self-hosted instance exposes the same [Brain MCP server](mcp/brain-mcp.md) and public API as hosted; the difference is who operates the database and billing.
- On a local install, writes and destructive actions are gated by default. An agent may hit an ask-first or blocked policy until the user enables the tool.
- The configured model backend is the only required outbound dependency.
  Connector-backed tools are absent until the user adds that connector's key.
- ChatGPT sign-in is an OSS Beta. If authorization expires or the account no
  longer exposes a selected model, Brian removes unavailable models from menus
  and asks the user to reconnect or select another backend.
- Support Mode never uploads automatically; sharing its downloaded capsule is a
  separate user action.

## Related

- [Pricing and credits](operations/pricing-and-credits.md)
- [Brain MCP server](mcp/brain-mcp.md)
- [Privacy and data](operations/privacy-and-data.md)
