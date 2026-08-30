---
title: Brain MCP Server
description: Connect an MCP client to a Use Brian workspace brain to read and write memory, tasks, CRM, files, and knowledge.
tags: [mcp, brain]
canonical: https://usebrian.ai/docs/api/brain-mcp
---

> Human-readable version: https://usebrian.ai/docs/api/brain-mcp

A Use Brian workspace brain is exposed as one MCP server over Streamable HTTP. An MCP client (Claude Code, Claude Desktop, ChatGPT connectors, or any custom client) authenticates with a workspace-scoped credential and receives the brain as tools: read and write memory, tasks, CRM, files, and knowledge. This is the page for an agent connecting itself to a user's brain.

## Endpoint

| Field | Value |
|---|---|
| Method + URL | `POST https://api.usebrian.ai/api/brain/mcp` |
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

The plaintext key is shown once at creation. Use Brian stores a hash and a prefix only.

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

The doc-page tools follow the same rule inside teamspaces: a page that lives in a teamspace resolves by the credential's clearance against the teamspace's sensitivity (and the page's own clearance), never by any human account's teamspace membership. A page you expect that does not resolve usually means the clearance ceiling, not a missing page.

### Team and Project binding

An `sk_brain_*` key may be issued with one Team and/or one Project binding in
Studio: Programmatic Access. The binding is immutable for that key and is
intersected with the workspace primary assistant's grants on every request.
General rows remain available; other Team/Project rows do not. Project
participation is not an ACL.

There is no MCP argument or request header that selects or widens this context.
To switch or widen scope, a workspace admin must issue a new key. If the bound
Team or Project is archived or no longer available, the request fails closed
rather than falling back to company-wide access. Stable Team/Project ids may be
shown in management UI; raw compartment keys are never part of the MCP
contract. OAuth-issued Brain credentials are currently workspace-wide subject
to their normal clearance ceiling.

## Connect Claude Code

One command. Replace the key with your own:

```
claude mcp add Use Brian --transport http https://api.usebrian.ai/api/brain/mcp --header "Authorization: Bearer sk_brain_..."
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
| `listCrmFields` | optional CRM entity kind and record id | Live custom-field keys, types, options, and reference targets; a visible record id also returns its current values |
| `fileRead` | file `id` or path | Text content + metadata of one workspace file. **Text only** — on an image, PDF, or other binary it returns an error rather than a lossy decode, so do not use it to inspect a stored image. Nothing returns raw bytes: bytes leave the brain by being delivered (attached to an email, sent to a chat channel), never by entering a tool result |
| `fileSearch` | title / summary / tag / name query | Compact file projection |
| `getBrand` | optional `slug` (omit = the workspace default brand); optional `include_draft` | The workspace's brand record: naming and legal usage, strategy, messaging and voice, color tokens, typography, logo variants bound by workspace file id, applications, claims, rights, governance, sources. Returns the APPROVED record by default; `include_draft: true` returns unapproved changes, which are a proposal. Present only on deployments with the brand surface wired |
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
| `setCrmCustomFields` | Patch typed workspace fields on one visible CRM entity. Call `listCrmFields` first and use its stable keys |
| `fileWrite` / `fileAppend` / `fileSetMeta` / `fileDelete` | Create or overwrite a file with authored text, append text, patch metadata, delete a file |
| `saveFileToBrain` | Promote a previously uploaded cached file (a `fileId`) into the brain, preserving the original bytes |
| `saveFileBytes` | Persist a file from raw bytes supplied as base64, preserving the exact bytes. Size-capped; larger files use the HTTP upload route |
| `saveBrandDraft` | Propose a change to the workspace's brand record. Writes the DRAFT only. It does not take effect until a workspace owner or admin approves it in Studio, which creates a new approved version. Each field group you pass replaces that group whole, so call `getBrand` first and send the complete group back when adding to a list. Bind assets by workspace file id (upload with `saveFileBytes` first), never by path |
| `createPage` / `editPage` / `deletePage` / `createPageFromTemplate` | Author doc pages: create, edit, delete, or seed from a template. `editPage` accepts Markdown and applies it to the live collaborative document when doc-sync is configured, so an edit is immediately shared with open page editors; deployments without doc-sync use the legacy version-checked page store. Present only on deployments with the doc surface wired |

`ingestToBrain` (default `decompose: true`) is the path for raw notes and documents: it derives entities and edges. `saveMemory` is for a single fact you have already distilled. Do not route task-shaped content through `saveMemory`: use `saveTask` or `ingestToBrain`.

## Errors

Every HTTP-level refusal carries a `message` beside the `error` slug, so an agent can act on it without a docs lookup:

| HTTP | `error` | What the `message` tells you |
|---|---|---|
| 401 | `invalid_brain_key` | One uniform body for every authentication failure (missing header, malformed token, unknown key, revoked key, expired or revoked access token — deliberately not distinguished): send `Authorization: Bearer sk_brain_<keyId>_<secret>` or `Bearer oat_<authorizationId>_<secret>`; an `oat_` access token expires after 10 minutes and is renewed with `grant_type=refresh_token`; API keys are issued in Studio → Programmatic access. Retrying the same credential will fail the same way. |
| 500 | `brain_mcp_error` | Authentication already succeeded — do not go looking at your credential. Transient; retry once, then stop. |

Inside a tool call, a failure is a normal MCP tool result with `isError` and a text body that states what failed on which target, why, the next step (which sibling tool discovers a valid id, or "ask the user"), and whether the same input can ever succeed. Argument validation failures arrive as compact `path: message` lines, not a JSON blob. A workspace-less credential is refused with the remedy (a workspace admin re-issues or re-scopes the key) — no argument change helps.

## Notes for agents

- One credential reaches exactly one workspace. To span workspaces, obtain one credential per workspace.
- **There is no tool that approves a brand, and there will not be one.** `saveBrandDraft` writes a draft; a human with an owner or admin role approves it in the app. If you are delivering a brand system into a client's workspace, push the draft, upload the master assets with `saveFileBytes`, bind them by file id, and then tell the client to approve. Do not report the brand as changed before they do.
- Call `tools/list` after `initialize` and select from what is returned. If a write tool is absent, the credential is `read`-scoped; if file tools are absent, the deployment has no file storage; if the page tools are absent, the deployment has no doc surface.
- An empty read result may be a clearance filter, not an empty brain: the credential only sees rows at or below its tier.
- An empty read may also be the key's Team/Project envelope. Never infer that a hidden Project is absent or ask the caller for a raw compartment key.
- Every write is audited exactly like a chat write. Assume no action is invisible to the workspace owner.
- Prefer `searchBrain` over the deprecated `searchKnowledge`.

## Related

- [MCP usage patterns](usage-patterns.md)
- [Pricing and credits](../operations/pricing-and-credits.md)
- [Self-hosting](../self-hosting.md)
