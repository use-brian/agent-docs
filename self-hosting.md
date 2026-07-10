---
title: Self-Hosting
description: Run the open-source sidanclaw core locally with one model key, plus what the hosted product adds.
tags: [operations, open-source]
canonical: https://sidan.ai/docs/open-source
---

> Human-readable version: https://sidan.ai/docs/open-source

The sidanclaw core is open source: the brain, the agent, workflows, channels, and the doc surface. Run it on your own machine with one model key, or let sidan.ai run it hosted.

## License

AGPLv3, OSI- and FSF-approved open source with a network-copyleft clause: run a modified sidanclaw as a hosted service and you publish your changes. A commercial license is available for orgs that cannot accept AGPL. Every contributor signs a CLA.

## Prerequisites

- Node 22+
- pnpm 10+
- A free Gemini API key (`aistudio.google.com/apikey`)

## Quickstart

```bash
git clone https://github.com/sidanclaw/sidanclaw.git
cd sidanclaw
export GEMINI_API_KEY=...   # or let the launcher prompt you; persisted under ~/.sidanclaw/
pnpm install
pnpm dev                    # api + canvas sidecar + web app, opens your browser
```

There is no step three.

## Storage

The store defaults to an embedded PGLite database under `~/.sidanclaw/`: nothing to install or run besides Node. Point `DATABASE_URL` at a local Postgres if you prefer a container.

## Local-first guarantee

Zero external services, one model key. The brain, the store, and the canvas all run on your machine. The only outbound call sidanclaw makes is to the Gemini API with your own key. Nothing else about your work leaves the machine.

## Tool governance defaults

Tools are governed by what they do, fail-closed:

| Action | Default |
|---|---|
| Reads (search, list, fetch) | Allowed |
| Writes (send, create, update) | Ask first, until you tell it "always" for one |
| Destructive (delete, revoke, cancel) | Blocked until enabled per tool |

A fresh install reads and drafts freely but cannot send an email or delete an event without you. Policy is set per tool in the app.

## Optional connector keys

The one Gemini key is the floor. Each key below is optional; nothing turns on by itself. Set them in `.env` or under `~/.sidanclaw/`.

| Capability | Key(s) | What you get |
|---|---|---|
| Web search | `BRAVE_SEARCH_API_KEY`, `TAVILY_API_KEY`, or `SERPER_API_KEY` | Upgrade search past the keyless DuckDuckGo fallback |
| Page fetches | `JINA_API_KEY` | Cleaner reads via Jina Reader (works keyless at lower limits) |
| Read X / Twitter | `TWITTER_BEARER_TOKEN` | Read x.com permalinks through the official X API v2 |
| X search | `XAI_API_KEY` | xAI Grok fallback plus the `xSearch` tool |
| Model fallback | `FALLBACK_PROVIDER_ENABLED=true` + `ANTHROPIC_API_KEY` | Keep running if Gemini is unavailable |
| Google connector | `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Calendar, Gmail, Drive via your own OAuth app |
| Notion connector | `NOTION_CLIENT_ID` / `NOTION_CLIENT_SECRET` | Notion via your own OAuth app |
| Fathom connector | `FATHOM_CLIENT_ID` / `FATHOM_CLIENT_SECRET` | Fathom via your own OAuth app |
| GitHub connector | Personal Access Token (entered in the UI) | GitHub, no env key needed |

Connector client id / secret can also live in `~/.sidanclaw/connectors.config.json`. Every key is documented in `.env.example`.

## What the hosted product adds

| Open core | Hosted platform adds |
|---|---|
| Agent engine, brain, memory, knowledge | Managed database, upgrades, backups |
| Channels, workflows, doc surface | Plans, credits, team billing |
| MCP server and public API | Monitoring, abuse protection, support |

The hosted product also gives every new workspace a 30-day Pro trial. See [Pricing and credits](operations/pricing-and-credits.md).

## Notes for agents

- A self-hosted instance exposes the same [Brain MCP server](mcp/brain-mcp.md) and public API as hosted; the difference is who operates the database and billing.
- On a local install, writes and destructive actions are gated by default. An agent may hit an ask-first or blocked policy until the user enables the tool.
- The only guaranteed outbound dependency is Gemini. Connector-backed tools are absent until the user adds that connector's key.

## Related

- [Pricing and credits](operations/pricing-and-credits.md)
- [Brain MCP server](mcp/brain-mcp.md)
- [Privacy and data](operations/privacy-and-data.md)
