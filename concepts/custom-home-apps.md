---
title: Custom Home Apps
description: Build a static web app that renders full-page inside a sidanclaw workspace and reads the brain through a scoped bridge token. Manifest schema, bundle format, sandbox limits, the postMessage bridge, and the consent model.
tags: [apps, mcp, brain]
canonical: https://sidan.ai/docs/apps/custom-home-apps
---

> Human-readable version: https://sidan.ai/docs/apps/custom-home-apps

A **custom Home app** is a static web app a workspace installs. It renders
full-page in a slot on the workspace's Home strip, runs sandboxed in the
viewer's browser, and reads the workspace brain through a scoped bridge token
that resolves to the same MCP tool surface an API key gets.

This is the page for an agent **building** one. If you are connecting yourself
to a brain over MCP instead, read [mcp/brain-mcp.md](../mcp/brain-mcp.md).

Two authoring paths, one artifact format:

| Path | How | Best when |
|---|---|---|
| GitHub repo | Template repo, imported and re-synced every 15 min | The app is versioned, reviewed, or shared between workspaces |
| Assistant-written | The workspace assistant writes it via `writeHomeApp` | Iterating conversationally; no repo wanted |

Template: `https://github.com/use-brian/brian-app-template`.
Validator: `npx @use-brian/brian-app lint` — the same code the importer runs.

## Bundle format

```
brian-app.json     manifest (required)
index.html         entry (required; any name the manifest points at)
assets/**          css, js, images, fonts (optional)
```

No build step, no server, no install. Files that cannot be served (`README.md`,
`LICENSE`, CI config, lockfiles) are ignored, so an ordinary repo works.

| Limit | Value |
|---|---|
| Files | 100 |
| Total size | 5 MB |
| Per file | 2 MB |

Served content types are pinned from an allowlist: `html`, `css`, `js`, `mjs`,
`json`, `svg`, `png`, `jpg`, `jpeg`, `gif`, `webp`, `ico`, `woff`, `woff2`,
`ttf`, `txt`, `map`. Anything else is not part of the bundle.

## Manifest

```json
{
  "manifestVersion": 1,
  "name": "Pipeline board",
  "description": "This quarter's deals, by stage",
  "icon": "Users",
  "entry": "index.html",
  "scopes": {
    "data": "read",
    "identity": false,
    "net": []
  }
}
```

| Field | Required | Notes |
|---|---|---|
| `manifestVersion` | yes | Must be `1`. An unknown version is rejected, not tolerated. |
| `name` | yes | ≤ 60 chars. Shown on the Home strip and the consent screen. |
| `description` | no | ≤ 280 chars. Shown on the consent screen. |
| `icon` | no | A lucide icon name. Unknown names fall back to a puzzle glyph. |
| `entry` | no | Defaults to `index.html`. Must be a relative `.html` path inside the bundle. |
| `scopes.data` | yes | `read` or `read_write`. Gates the bridge tool list. |
| `scopes.identity` | no | Release the viewer's display **name**. A stable `userId` is always present without it. |
| `scopes.net` | no | Extra origins the app may fetch. Bare `https://host` only — no path, no wildcard. |

Unknown **top-level** fields are preserved as metadata. Unknown keys inside
`scopes` are a hard error.

## Consent model

An app does not render until a workspace owner or admin **grants the scopes its
manifest requests**. Registering or importing an app is not approval.

**Widening scopes voids the grant.** If a sync brings a manifest asking for
more than was granted, the app drops to `needs_consent` and leaves the Home
strip until an admin re-approves. Plan for this: ship features freely, batch
scope changes.

Admins may also cap an app's clearance ceiling and its daily bridge-call
budget. A call past the budget returns `429`.

## Sandbox

The app runs in an iframe with `sandbox="allow-scripts allow-forms"` and
**without `allow-same-origin`**, so it executes at an opaque origin:

- no cookies
- no `localStorage` / `sessionStorage` / `IndexedDB`
- no access to the surrounding page

Bundle responses carry a CSP: `default-src 'none'`, with `script-src` /
`style-src` allowing `'self'` and inline, `img-src` allowing `data:` and
`blob:`, and `connect-src` limited to the API origin plus any granted
`scopes.net` origins.

## Bridge

### Handshake

```js
const ctx = await new Promise((resolve) => {
  window.addEventListener("message", (e) => {
    if (e.data?.type === "ub:token" && e.data.token) resolve(e.data);
  });
  parent.postMessage({ type: "ub:ready" }, "*");
});
// ctx = { token, apiOrigin, appId, workspaceId }
```

The token is short-lived (~10 min) and refreshed by the host. Post
`{ type: "ub:token" }` to request one on demand.

### Host verbs

| Message | Direction | Effect |
|---|---|---|
| `{ type: "ub:ready" }` | app → host | Request the bridge token |
| `{ type: "ub:token" }` | app → host | Refresh the bridge token |
| `{ type: "ub:token", token, apiOrigin, appId, workspaceId }` | host → app | The token payload |
| `{ type: "ub:navigate", path }` | app → host | Navigate the host. In-app paths inside the app's own workspace only; anything else is ignored. |

There is no data verb — data goes through MCP.

### Reading the brain

The bridge token authenticates against the brain MCP server, gated to the
granted scope and the viewer's clearance:

```http
POST {apiOrigin}/api/brain/mcp
Authorization: Bearer {ctx.token}
Content-Type: application/json

{"jsonrpc":"2.0","id":1,"method":"tools/call",
 "params":{"name":"searchBrain","arguments":{"query":"open deals"}}}
```

Tools available at `read`: `searchBrain`, `searchRecording`,
`searchFileContent`, `getEntity`, `getMemory`, `getTask`, `listTasks`,
`getContact`, `listContacts`, `getCompany`, `listCompanies`, `getDeal`,
`listDeals`, `fileRead`, `fileSearch`, `readPage`, `listPages`,
`listPageTemplates`. `read_write` adds the matching write tools. Full table:
[mcp/brain-mcp.md](../mcp/brain-mcp.md).

**Results are clearance-filtered per viewer.** Two people can open the same app
and see different rows. Do not cache one viewer's results into
workspace-scoped state.

### State

The sandbox has no browser storage, so apps persist through bridge KV:

```http
GET {apiOrigin}/api/home-apps/{appId}/state?scope=user
PUT {apiOrigin}/api/home-apps/{appId}/state?scope=user
Authorization: Bearer {ctx.token}
Content-Type: application/json

{"data": {"lastTab": "deals"}}
```

`scope=user` is per viewer; `scope=workspace` is shared. 256 KB each. A write
over the cap returns `413`.

## Errors

| Status | Meaning |
|---|---|
| `401` on a bundle or bridge call | Missing, expired, or wrong-audience token |
| `403` on a bridge call | The app is not active, or its grant was revoked |
| `429` on a bridge call | The app's daily bridge-call budget is spent |
| `413` on a state write | Over the 256 KB per-scope cap |

## Sync

A GitHub-kind app re-syncs every 15 minutes, or on demand from Studio. There
are no webhooks: PAT-only connectors cannot register them. Credentials resolve
through the workspace's GitHub connector, so revoking the connector revokes the
sync.

An unchanged branch HEAD is a no-op. A sync that fails records its reason
against the app and does not disturb the previous working bundle.
