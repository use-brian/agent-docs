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

- Git
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
- The optional WeChat desktop personal-account bridge requires a container
  runtime, the chat message archive, and a host that can run the WeChat Linux
  client with `SYS_PTRACE` and `seccomp=unconfined`.

## Local setup

This is the main starting path. It uses the embedded PGLite database and does
not require Docker. Model requests still go to the selected provider; enabled
connectors and search providers can make outbound requests too. The human guide
includes real screenshots of the provider controls, Chat, and Connectors.

### 1. Download and install

Check `node --version`, `pnpm --version`, and `git --version` first. Then:

```bash
git clone https://github.com/use-brian/use-brian.git
cd use-brian
pnpm install
```

Resolve installation errors before continuing. Keep subsequent commands in the
same repository directory. LibreOffice is optional for PDF export; ffmpeg and
ffprobe are needed for recordings, not the first text conversation.

### 2. Run the launcher

```bash
pnpm dev
```

Use an interactive terminal and leave it open. A fresh setup asks for a model
provider and display name: choose 1 for ChatGPT sign-in or 2 for a Gemini API
key. Saved configuration can skip these prompts. The launcher builds packages,
prepares the embedded database, and starts the API, document sync, web app and
local bridges. Wait for the startup URLs before starting another instance.

### 3. Connect the provider

For ChatGPT, open the workspace menu, then **Settings → Models → Providers**.
Complete **Sign in with ChatGPT** or **Use device code**. This backend is Beta
and depends on the subscription's quota and availability. A Gemini key entered
in the launcher is used directly. Vertex AI and DashScope use `.env` settings;
OpenAI-compatible endpoints can be added in the app. See the repository's
[model backends](https://github.com/use-brian/use-brian#model-backends).

### 4. Verify a first conversation

The launcher normally opens the local app and signs in the local owner. If the
browser does not open, visit:

```text
http://localhost:3003/api/auth/local-session
```

The local owner flow does not need hosted Google or email sign-in. Open Chat,
send a short request, and verify that Brian replies. A loaded page or listening
port alone does not prove that the provider works.

### 5. Add tools when needed

Open **Studio → Connectors** or **Studio → Channels**. Self-hosted OAuth
connectors may need an operator-owned provider application, its credentials and
correct redirect URLs before Connect can succeed. Use the repository's
[environment reference](https://github.com/use-brian/use-brian/blob/main/.env.example).
Review the assistant's connector permissions separately from the account connection.

### 6. Stop, back up and update

Ctrl-C stops the launcher; `pnpm dev` restarts it. For the default local setup:

- `~/.usebrian/config.json`: settings and generated secrets.
- `~/.usebrian/brain`: embedded database.
- `~/.usebrian/files`: uploaded files.

Stop Brian before copying the complete `~/.usebrian` directory for a consistent
embedded-database backup. Include custom storage paths and protect the secrets.
For an unmodified checkout, update with:

```bash
git pull --ff-only
pnpm install
pnpm dev
```

Review and preserve source changes before updating a modified checkout.

### Troubleshooting

- A launcher waiting for input needs an interactive terminal; check the provider,
  key and display-name prompts.
- Default ports are app 3003, API 4000, document sync 8080, embedded database
  54329. Stop an earlier Brian instance; `USEBRIAN_API_PORT` can resolve an API
  conflict. Do not stop unrelated services blindly.
- If the UI opens without a reply, check provider connection, credentials, quota,
  and terminal errors.
- `DATABASE_URL` selects external PostgreSQL and skips embedded setup. Provision
  that database and apply OSS migrations before boot; setting the URL is not
  sufficient.

A public server additionally needs persistent storage, process supervision,
HTTPS, authentication, backups and correctly configured app/API/document-sync
origins. Do not expose the local-owner entry point as public sign-in. See the
[container reference](https://github.com/use-brian/use-brian#container-images)
and the production deployment section below.

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

The separate WeChat desktop bridge mirrors a personal account instead of
creating an iLink bot. Clone `brian-message-store` beside `use-brian`; its
`agent-wechat/` directory is the pinned standalone runtime source used by the
`wechat-desktop-bridge` Compose stack. Enable the chat message archive, create
a Custom channel in Studio, and give the bridge the one-time channel token.
The runtime port stays on the Compose network; the bridge needs only outbound
HTTPS to the Use Brian API. Pairing is completed by scanning the QR shown in
the Custom channel detail panel.

The account owner must personally scan that QR and confirm the desktop login.
One host may run multiple consenting owners only as separate one-account
stacks, each with its own runtime directory, tokens, cursor state, named
volumes, loopback ports, containers, and service units. The `deploy-brian`
helper records only generic instance slugs in `WECHAT_DESKTOP_INSTANCES` after
an instance is healthy. Do not pool credentials or message data, expose the
agent-wechat endpoint, or operate it as a multi-tenant WeChat service.

The imported `agent-wechat` revision currently has no upstream `LICENSE` or
`COPYING` file. That is not a redistribution grant: keep modified images on the
operator's own deployment until upstream licensing is resolved. See
`agent-wechat/UPSTREAM.md` in `brian-message-store` for the exact revision.

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

The default embedded PGLite database lives at `~/.usebrian/brain`. External
PostgreSQL is supported through `DATABASE_URL`; provision it and apply OSS
migrations separately. See the local backup and configuration steps above.

## Chat message archive

Self-hosted only, and off by default. `brian-message-store` keeps a searchable
archive of the messages your channels carry, including the attachment bytes, so
an assistant can answer from what was actually said months ago rather than only
from the current conversation. It adds the `searchChatHistory`,
`listChatChannels`, and `saveChatMedia` tools: search hits carry the sender's
id, display/push name, and (when the owner's WhatsApp address book has synced)
their saved contact name, plus a `media_sha256` for any attachment.
`saveChatMedia` pulls those stored bytes into the workspace file layer, where
`sendFile` can deliver them on a document-capable channel.

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

WhatsApp history can come from a chat export, an already-decrypted Android
`msgstore.db`, or an already-decrypted iOS `ChatStorage.sqlite` from an iPhone
backup. The two databases are different formats and are not interchangeable, so
each has its own source. Decrypting the backup is a separate step you run
yourself — the archive is handed plaintext files and never derives a key.

An iOS import also loads your WhatsApp address book, which is what lets search
find a conversation by the name you know someone under rather than by their
number. That matters more than it sounds: WhatsApp increasingly addresses people
by a privacy identifier that contains no phone number, so a participant you have
never saved may have no name and no number anywhere in the backup. They are
still archived and still searchable by what they said.

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

Feed image placeholders let you draft first and generate later. Open a
placeholder's options and choose Gemini 3.1 Flash Image or, in OSS, Codex
(ChatGPT subscription). Codex requires an eligible connected ChatGPT plan and
uses the pinned runtime's built-in image engine. Review the estimate and confirm
explicitly before generation. Codex consumes subscription quota, whose exact
usage cannot be quoted in advance, with no Brian image surcharge. Results stay
as candidates until accepted; a provider failure never switches providers.

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
| Web search | `BRAVE_SEARCH_API_KEY`, `SERPER_API_KEY`, `SERPAPI_API_KEY`, `TAVILY_API_KEY`, or `BAIDU_SEARCH_API_KEY` | Upgrade search past the keyless DuckDuckGo fallback; Baidu adds Chinese-language and mainland-China coverage |
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

## CRM recovery tooling

The OSS tree supplies `scripts/operations/brian-backup.mjs`,
`brian-restore-check.mjs` and `brian-wal-archive.mjs`, with configuration and
readiness gates in `docs/operations/crm-recovery.md`. They use PostgreSQL 18,
explicit private credential/key files, encrypted artifacts and new disposable
loopback restore targets. Logical full restore and physical base-backup plus
continuous-WAL PITR are separate paths. Default CLI behavior is preflight.

Restore verifies migration/schema and content hashes, replays protected erasure
effects and checks referential integrity before application verification. No
application or sender starts automatically. A storage-only proof remains blocked
for application acceptance. Off-instance custody, upload/scheduling/alerts,
measured RPO/RTO and a deployment-specific application rehearsal remain operator
work; a synthetic local pass does not approve cutover.

## CRM engineering acceptance

`scripts/crm/brian-contract-check.mjs --mode local --report-dir /new/private/report`
runs an explicit suite catalog with disposable PostgreSQL 18, a restricted app
role, loopback API routes, fake providers and complete persisted workflows.
Select an installation with pgvector/pg_trgm using `--pg-bin`. It rebuilds
shared/core first and records actual SHA, schema/migration and fixture hashes,
assertion names and logical/PITR recovery evidence. Missing, skipped or failed
evidence cannot pass. Existing report directories are refused.

Remote QA requires an explicit dedicated workspace, controlled synthetic prefix,
CRM-scoped credential and confirmation. QA and production modes expose only
catalog qualification; production cannot dispatch mutations, sends, erasure or
load tests. These modes report the engineering matrix as unexecuted, even when
qualification succeeds. See the OSS `docs/operations/crm-assurance-runbook.md`.
Local success does not approve policy, live integrations, data cutover,
off-instance recovery, staff acceptance or measured soak.

### LinkedIn through Feed Cloud Link

Unlinked OSS includes canonical LinkedIn drafting, conversion, approval and manual
newsletter preparation/completion. Managed API publishing uses an approved paid
Feed Cloud Link and the hosted service's official LinkedIn app. No BYO LinkedIn
app credentials or native newsletter API are included.

Cloud Link sends a revision-bound approved payload and transfers its authorized
image bytes through reservation-scoped slots. Target aliases are bound to the
link; do not substitute a hosted destination ID. Entitlement, link revocation and
publishing permission are checked again before provider work. An uncertain
network result leaves the local draft unresolved until an actual receipt or
explicit operator reconciliation is available. Provider activation remains gated
on hosted OAuth/product approval and live smoke evidence.
