---
title: Brain (entities & episodes)
description: The structured graph of entities, edges, and immutable episodes beneath memory and the knowledge base.
tags: [concepts, brain]
canonical: https://sidan.ai/docs/brain
---

> Human-readable version: https://sidan.ai/docs/brain

Memory and the knowledge base are the two surfaces you see in chat. Underneath sits the brain: a structured graph of entities, edges, and immutable episodes that grows every time you or a connector feeds it a new signal.

## From signal to structure

Everything the workspace hears (chat, email, meetings, GitHub, files and voice) becomes an immutable episode in an append-only log. One extractor turns episodes into structure, trying the strongest shape first along a precedence ladder: Task -> Entity + edge -> CRM -> Memory -> Ephemeral. Chat then retrieves over the result.

## The primitives

Five operational primitives, three cognitive ones. The operational primitives are what you query directly in chat: CRM (people + companies + deals), Tasks, KB, Memory, and Files. The cognitive substrate threads them together.

| Cognitive primitive | What it is |
|---|---|
| Entities | Canonical nouns: people, companies, projects, deals, products. Every brain primitive resolves to or links from an entity. |
| Edges | Typed graph relationships between entities (`works_at`, `engagement_of`, `mentioned_in`). Bi-temporal: a closed relationship keeps its history. |
| Episodes | The append-only observation log. Every memory, entity, and edge points back to the episode that observed it. Episodes are immutable; consolidation reads, never writes. |

## How signals become structure

Pipeline B is the unified extraction surface. Given any episode, it runs a single LLM call that emits entities, edges, tasks, memories, and ephemeral items, with every observation evaluated against the precedence ladder (Task -> Entity/Edge -> CRM -> Memory -> Ephemeral). Memory is the last resort; emissions carry `why_not_entity` / `why_not_task` justifications so the model confronts the alternative.

### Three pipelines, one extractor

| Pipeline | Role |
|---|---|
| Pipeline A: chat | Live chat turns hit a compaction checkpoint; the compacted window becomes a `web_chat` episode and runs Pipeline B fire-and-forget. |
| Pipeline B: episode to derived rows | The shared extractor. Called by A, C, and active capture (file upload, voice memo, manual save). |
| Pipeline C: external event ingest | Connector events (GitHub, Calendar, Fathom, Slack) flow through the rules engine; matched events become episodes and run Pipeline B. |

## Rules engine

Per-connector-instance rules decide how each event is routed: `realtime` (straight to Pipeline B), `scheduled` (queued in `pending_ingest_batches`, drained on a cron), or `drop` (discarded). Filters are first-match-wins, with default templates per source (":crm_contacts -> realtime" for Calendar, "is_dm -> realtime" for Slack, "pull_request.merged -> realtime+alert" for GitHub). Edit them under Studio -> Ingestion.

## Self-healing classifier

The v2 classifier does not decide once. Pipeline B's emissions are scored against the original episode; low-confidence rows go back through a reclassifier pass that can re-tier, merge near-duplicates, or downgrade a Memory to Ephemeral. Existing memories that drift from their evidence are surfaced for retraction in the corrections queue.

## How chat reads the brain

Chat sees the brain through a 7-tool surface: `getEntity` (entity rollup with summary + edges + recent episodes), `search` (hybrid FTS + vector + graph + recency, RRF-fused), `recentEpisodes`, `provenance` (one-level walk for any row), `markUseful` (boosts a row's retrieval rank), `aggregate` (typed measures over typed columns or JSONB attributes), and `getRowHistory` (the supersession-audit chain).

## Brain vs memory

Memory is one primitive in the brain: the layer that holds inferred behavioral facts about people. Entities, edges, tasks, deals, and files are the rest. The acid test for the boundary: "if the source disappeared, does this row stay?" Yes for memory, typically no for KB chunks (which are re-importable from the source repo or docs).

## Other surfaces worth knowing

- Explicit links: every brain-write tool (`saveMemory`, `saveContact`, `saveDeal`, `createTask`, `fileWrite`, ...) accepts a universal `links` parameter to connect new rows to existing entities in the same call. Capped at 20 links per call.
- Graph view: an Obsidian-style force-directed visualization via the Graph toggle on the Brain page. Kind-coloured nodes, click-through to the detail drawer. Capped at 500 nodes.
- Trust signals: every row carries a `source` (user, model, connector, episode-graduation) and an optional `verified_by_user_id`. Retrieval ranks rows via `SOURCE_WEIGHTS` x `VERIFIED_BOOST`: provenance, not a numeric confidence score.

## Notes for agents

- Episodes are immutable and append-only; there is no path to edit or delete an observation, only to supersede or retract the rows derived from it.
- Extraction always prefers the strongest shape: a stated commitment becomes a Task, a named person an Entity/CRM row, and only leftover behavioral facts become Memory. Expect a "remember X" to land as the most structured primitive that fits, not always as a memory.
- Every derived row is traceable: use `provenance` to walk one level back to the episode, and `getRowHistory` for the supersession chain.
- Ingestion routing is per-connector-instance and first-match-wins; a connector set to `drop` or `scheduled` will not produce realtime brain rows.

## Related

- [Memory & knowledge](./memory-and-knowledge.md)
- [Tasks](./tasks.md)
- [CRM](./crm.md)
