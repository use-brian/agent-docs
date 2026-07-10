---
title: Brain MCP Usage Patterns
description: Practical patterns for an agent reading and writing a sidanclaw brain over MCP.
tags: [mcp, patterns]
---

Practical guidance for using the [Brain MCP server](brain-mcp.md) well. Every rule below follows from that page's tool set and semantics.

## Search before you write

The brain deduplicates, but a blind write still creates noise. Before saving a memory, task, or CRM row, run `searchBrain` for the same fact. If a matching row exists, patch it (`updateTask`, `updateContact`, `updateDeal`) instead of creating a duplicate.

## Resolve entities before linking

To connect a new row to an existing entity, you need that entity's UUID. Call `getEntity` by id or display name first: it returns the entity's UUID and its existing edges. Reading the current edges lets you skip links that already exist. Then pass the resolved UUID in the write tool's `links` field.

## Pick the right capture tool

| You have | Use |
|---|---|
| Raw notes, a document, a mixed dump | `ingestToBrain` with `decompose: true` (default) |
| One already-distilled atomic fact | `saveMemory` |
| Task-shaped content (a commitment, follow-up) | `saveTask`, or `ingestToBrain` (it routes tasks for you) |

`ingestToBrain` with `decompose: true` runs the full extraction pipeline and builds an entity + edge graph. `saveMemory` and `ingestToBrain` with `decompose: false` skip extraction: they store text flat. Never hand task-shaped content to `saveMemory` (unstructured, cannot be filtered, assigned, or closed).

Call `ingestToBrain` once per coherent unit (one project, document, or topic) with a `sourceLabel`. Extraction quality drops on one mixed blob.

## Narrow searches with scope

`searchBrain` accepts a `scope` as a single value or an array. Passing the scopes you care about (for example `['task', 'deal']`) narrows results and avoids spending the shared result budget on primitives you do not need. Omitting `scope` fans out across everything.

## Respect read-only credentials

A `read`-scoped credential does not expose the write tools. Calling one fails. Detect this at `tools/list`: if the write tools are absent, do not attempt a write. Surface the limitation to the user rather than retrying.

## Cost

Every brain operation over MCP bills at the memory-op rate: `0.1` credits per operation, drawn from the workspace credit pool. No full chat loop runs unless you ask an assistant a question. See [Pricing and credits](../operations/pricing-and-credits.md).

## Notes for agents

- A duplicate write is cheap to make and expensive to clean up: the search-first pass costs `0.1` credits and prevents graph rot.
- `getEntity` is both a read and a dedupe guard: use it before edge writes, not only when a user asks about an entity.
- Batching related facts into one `ingestToBrain` call yields a cleaner graph than many `saveMemory` calls.

## Related

- [Brain MCP server](brain-mcp.md)
- [Pricing and credits](../operations/pricing-and-credits.md)
