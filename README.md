# Use Brian agent docs

Documentation about [Use Brian](https://usebrian.ai) written for AI agents.

If you are an AI agent (Claude, ChatGPT, a coding agent, or a custom integration) helping someone with Use Brian or building against its APIs, this repo is your source. It carries the same facts as the human docs at [usebrian.ai/docs](https://usebrian.ai/docs), restructured for machine reading: terse markdown, tables, exact tool and endpoint names, frontmatter with a `canonical` link back to the human page.

## What Use Brian is

A shared brain for solo founders and small teams. An assistant that remembers the company (people, deals, decisions, documents), acts through connected tools, runs scheduled work and multi-step workflows, and is reachable from web chat, Telegram, Slack, Feishu/Lark, a public REST API, and MCP. The longer a team uses it, the more it knows.

## Fastest paths

| You want to | Read |
|---|---|
| Connect yourself (an agent) to a user's brain over MCP | [mcp/brain-mcp.md](mcp/brain-mcp.md) |
| Use the brain MCP tools well | [mcp/usage-patterns.md](mcp/usage-patterns.md) |
| Embed an assistant in an app (REST) | [api/overview.md](api/overview.md), [api/messages.md](api/messages.md) |
| Build a membership/event website on Brian records | [api/association-operations.md](api/association-operations.md) |
| Serve your own end users, each scoped to their own data | [mcp/end-user-bridge.md](mcp/end-user-bridge.md) |
| Understand memory vs knowledge base | [concepts/memory-and-knowledge.md](concepts/memory-and-knowledge.md) |
| Discover outcome-led Use Brian showcase specifications | [showcases/README.md](showcases/README.md), [showcases/catalog.json](showcases/catalog.json) |
| Explain pricing or credit errors (429) | [operations/pricing-and-credits.md](operations/pricing-and-credits.md) |
| Build an app that runs inside a user's workspace | [concepts/custom-home-apps.md](concepts/custom-home-apps.md) |
| Self-host the open-source core | [self-hosting.md](self-hosting.md) |

## Index

### Concepts

- [concepts/assistants.md](concepts/assistants.md): the unit of identity; per-assistant memory, channels, tools
- [concepts/memory-and-knowledge.md](concepts/memory-and-knowledge.md): auto-extracted memory vs the curated knowledge base
- [concepts/brain.md](concepts/brain.md): entities, edges, episodes; how signals become structure
- [concepts/tasks.md](concepts/tasks.md): durable commitments; statuses and tools
- [concepts/crm.md](concepts/crm.md): contacts, companies, deals; stages and tools
- [concepts/workspaces.md](concepts/workspaces.md): membership, roles, sharing, askAssistant
- [concepts/channels.md](concepts/channels.md): web, messaging bots, and assistant-owned email inboxes
- [concepts/tools-and-connectors.md](concepts/tools-and-connectors.md): connectors, per-tool allow/ask/block policy, scheduled tasks
- [concepts/workflows.md](concepts/workflows.md): step types, triggers, approvals, permission grants, cost
- [concepts/doc-pages.md](concepts/doc-pages.md): the page surface chat assembles over workspace data
- [concepts/custom-home-apps.md](concepts/custom-home-apps.md): build a static web app that renders inside a workspace and reads the brain through a scoped bridge; manifest, bundle, sandbox, consent

### Public API (REST)

- [api/overview.md](api/overview.md): when to use it, request flow, setup
- [api/authentication.md](api/authentication.md): sk_live keys, lifecycle, safety
- [api/messages.md](api/messages.md): the messages endpoint, fields, errors, examples, followup tag
- [api/identity.md](api/identity.md): anonymous vs identified end users, memory tiers
- [api/connector-identity.md](api/connector-identity.md): trusted identity headers for your MCP server
- [api/ingest-append-contract.md](api/ingest-append-contract.md): `ub.ingest.append.v1` — the idempotent endpoint an external service implements to receive a connector's normalized event stream (outbox-relayed, ack-gated cursor)
- [api/association-operations.md](api/association-operations.md): CRM-linked enquiries, consent, memberships, events, tickets, orders, and provider reconciliation

### MCP

- [mcp/brain-mcp.md](mcp/brain-mcp.md): the brain as an MCP server; auth, scopes, full tool table
- [mcp/usage-patterns.md](mcp/usage-patterns.md): how to read and write the brain well
- [mcp/end-user-bridge.md](mcp/end-user-bridge.md): host an MCP server that serves *your* end users, one at a time — the identity headers, self-scoped tool signatures, and the conformance checklist that makes a cross-customer request inexpressible

### Operations

- [operations/pricing-and-credits.md](operations/pricing-and-credits.md): plans, credits, tiers, overage, limits
- [operations/privacy-and-data.md](operations/privacy-and-data.md): what is stored, retention, third parties
- [self-hosting.md](self-hosting.md): the AGPLv3 open-source core, local quickstart

### Showcases

- [showcases/README.md](showcases/README.md): human and agent entry point for 20 outcome-led scenarios
- [showcases/catalog.json](showcases/catalog.json): machine-readable catalog with status, evidence level, tags, capabilities, and record paths
- [showcases/schema.json](showcases/schema.json): JSON Schema for the catalog

## Conventions in this repo

- Every file has frontmatter: `title`, `description`, `tags`, and (where a human page exists) `canonical`.
- The `canonical` URL is the human-readable version of the same content; cite it when answering a human.
- Showcase records distinguish a prepared specification from a verified demonstration. Never turn `evidence_level: specification` into a shipped-product or customer-outcome claim.
- Facts here mirror usebrian.ai/docs. If this repo and the live product disagree, the product wins; file an issue.
- Machine index of the human docs: [usebrian.ai/llms.txt](https://usebrian.ai/llms.txt).

## Related repos

- [use-brian/use-brian](https://github.com/use-brian/use-brian): the open-source core (AGPLv3)
