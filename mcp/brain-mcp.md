---
title: Brain MCP Server
description: Connect an MCP client to a sidanclaw workspace brain to read and write memory, tasks, CRM, files, and knowledge.
tags: [mcp, brain]
canonical: https://sidan.ai/docs/api/brain-mcp
---

> Human-readable version: https://sidan.ai/docs/api/brain-mcp

A sidanclaw workspace brain is exposed as one MCP server over Streamable HTTP. An MCP client (Claude Code, Claude Desktop, ChatGPT connectors, or any custom client) authenticates with a workspace-scoped credential and receives the brain as tools: read and write memory, tasks, CRM, files, and knowledge. This is the page for an agent connecting itself to a user's brain.

## Endpoint

| Field | Value |
|---|---|
| Method + URL | `POST https://api.sidan.ai/api/brain/mcp` |
| Transport | Streamable HTTP MCP |
| Protocol | `initialize`, `tools/list`, `tools/call` |
| Scope | One credential = one workspace |

Discover the tool set with a standard `tools/list` call: do not hardcode it. File tools appear only on deployments configured with file storage.

## Authentication

Send the credential as `Authorization: Bearer <token>`. Two credential formats are accepted; both resolve to the same workspace-scoped principal.

### API key

| Field | Value |
|---|---|
| Token shape | `sk_brain_<keyId>_<secret>` |
| Issued / revoked | Studio: Programmatic Access |
| Header | `Authorization: Bearer sk_brain_...` |
| Best for | Claude Code, scripts, servers |

The plaintext key is shown once at creation. sidanclaw stores a hash and a prefix only.

### OAuth 2.1

For clients with a Connect button (claude.ai, Claude Desktop, ChatGPT connectors) that do not expose a raw header field.

| Field | Value |
|---|---|
| Grant | Authorization code + PKCE, `S256` only |
| Client onboarding | Dynamic client registration |
| Consent | User signs in, picks one workspace, approves the scope |
| Access token | `oat_<...>`, valid 10 minutes |
| Refresh token | `ort_<...>`, valid 30 days, rotated on every use |

Reusing a refresh token after rotation fails. `plain` PKCE is rejected.

### Per-credential scope

Each credential carries a `scope`. A `read` credential never sees the write tools: they are omitted from `tools/list` and rejected at `tools/call`.

| `scope` | Tools exposed |
|---|---|
| `read_write` (default) | read tools + write tools |
| `read` | read tools only |

Hand a `read` credential to a low-trust integration; use `read_write` for your own agent.

### Clearance and authority

Reads are clearance-filtered to the credential's ceiling: a credential never returns rows above its tier. Writes act with the workspace's default authority and carry the same permissions and audit trail as a chat turn. There is no separate, unaudited write path.

## Connect Claude Code

One command. Replace the key with your own:

```
claude mcp add sidanclaw --transport http https://api.sidan.ai/api/brain/mcp --header "Authorization: Bearer sk_brain_..."
```

## Read tools

Available on both `read` and `read_write` credentials.

| Tool | Inputs | Returns |
|---|---|---|
| `searchBrain` | `query` (required); optional `scope` (`memory` \| `task` \| `contact` \| `company` \| `deal` \| `file` \| `kb_chunk` \| `entity` \| `file_segment`, single value or array, omit = all scopes); optional `limit` (default 20, max 100) | Unified retrieval across every primitive, clearance-filtered and scoped to the workspace |
| `searchFileContent` | `fileId` + `query`, or `fromIndex` / `toIndex` for sequential paging | Passages inside one stored document. Never returns the whole file |
| `getEntity` | entity `id` or display name; optional `walk_depth` / `walk_edge_types` | Entity rollup: existing relationship edges, recent episodes, memory, open tasks. Use it to resolve an entity's UUID and read existing edges before a linking write |
| `getMemory` / `getTask` / `getContact` / `getCompany` / `getDeal` | record `id` | Full record |
| `listTasks` | assignee / status / due range / tag / parent filters | Compact task projection |
| `listContacts` / `listCompanies` / `listDeals` | CRM filters | Compact CRM projection |
| `fileRead` | file `id` or path | Full content + metadata of one workspace file |
| `fileSearch` | title / summary / tag / name query | Compact file projection |
| `readPage` / `listPages` / `listPageTemplates` | page `id` or title; list filters; template catalog | Read doc pages. Present only on deployments with the doc surface wired |
| `searchKnowledge` | `query` | Deprecated alias for `searchBrain` with `scope: 'kb_chunk'`. Prefer `searchBrain` |

## Write tools

Available only on `read_write` credentials.

| Tool | What it does |
|---|---|
| `ingestToBrain` | Capture content into the brain. `decompose: true` (default) runs the full extraction pipeline (entities, edges, tasks, memories, deduplicated) and returns a summary. `decompose: false` files one distilled memory verbatim, no extraction. Inputs: `content`, optional `sourceLabel`, optional `decompose` |
| `saveMemory` / `deleteMemory` | Save one distilled workspace memory; soft-delete a memory by id |
| `saveTask` / `updateTask` / `closeTask` / `reopenTask` | Create, patch, and transition tasks |
| `saveContact` / `updateContact` | Upsert / patch a contact |
| `saveCompany` / `updateCompany` | Upsert / patch a company |
| `saveDeal` / `updateDeal` / `advanceDealStage` | Create / patch a deal; move a deal through `lead` -> `qualified` -> `proposal` -> `negotiation` -> `won` / `lost` |
| `fileWrite` / `fileAppend` / `fileSetMeta` / `fileDelete` | Create or overwrite a file with authored text, append text, patch metadata, delete a file |
| `saveFileToBrain` | Promote a previously uploaded cached file (a `fileId`) into the brain, preserving the original bytes |
| `saveFileBytes` | Persist a file from raw bytes supplied as base64, preserving the exact bytes. Size-capped; larger files use the HTTP upload route |
| `createPage` / `editPage` / `deletePage` / `createPageFromTemplate` | Author doc pages: create, edit, delete, or seed from a template. Present only on deployments with the doc surface wired |

`ingestToBrain` (default `decompose: true`) is the path for raw notes and documents: it derives entities and edges. `saveMemory` is for a single fact you have already distilled. Do not route task-shaped content through `saveMemory`: use `saveTask` or `ingestToBrain`.

## Notes for agents

- One credential reaches exactly one workspace. To span workspaces, obtain one credential per workspace.
- Call `tools/list` after `initialize` and select from what is returned. If a write tool is absent, the credential is `read`-scoped; if file tools are absent, the deployment has no file storage; if the page tools are absent, the deployment has no doc surface.
- An empty read result may be a clearance filter, not an empty brain: the credential only sees rows at or below its tier.
- Every write is audited exactly like a chat write. Assume no action is invisible to the workspace owner.
- Prefer `searchBrain` over the deprecated `searchKnowledge`.

## Related

- [MCP usage patterns](usage-patterns.md)
- [Pricing and credits](../operations/pricing-and-credits.md)
- [Self-hosting](../self-hosting.md)
