---
title: Memory & knowledge
description: Memory holds per-(user, assistant) facts about you; the knowledge base holds per-workspace facts the assistant can look up.
tags: [concepts, memory]
canonical: https://sidan.ai/docs/memory
---

> Human-readable version: https://sidan.ai/docs/memory

Two systems handle long-term context. Memory holds facts about you. The knowledge base holds facts about the world that the assistant should be able to look up.

## Memory

Memory is per (user, assistant). The assistant extracts patterns from what you say ("we're based in Hong Kong," "we invoice net-30," "the Q3 launch ships Sept 18") and stores them. On every turn, a curated summary of relevant memories is included in the system prompt.

### How a memory is shaped

| Facet | What it holds |
|---|---|
| Preference | Stable facts about how you work ("prefers markdown", "invoices net-30"). Surfaced first when retrieval is relevant. |
| Context | Situational facts the assistant has observed ("Q3 launch ships Sept 18", "Acme renewal closes this Friday"). Most rows live here. |
| Provenance | Every row points back to the episode that produced it (the chat turn, voice memo, or connector event), so any belief traces to its source. |

### Your controls

Every memory is editable. Settings -> Privacy -> Memories shows all stored memories: search, edit, or delete one-by-one or wholesale. Asking the assistant "forget that" works too.

## Knowledge base

The KB is per-workspace, not per-user. Use it for facts the assistant should know (proposal voting rules, product specs, FAQ answers). The assistant has `searchKnowledge`, `browseKnowledge`, `readKnowledgeEntry`, and `addKnowledgeEntry` tools to retrieve and append. Connect a GitHub repo from Studio -> Knowledge to sync markdown docs automatically.

### Sensitivity tiers

Each KB entry is tagged `public`, `internal`, or `confidential`. The assistant's clearance gates what it can read across every channel, including the public API. To expose only public KB to API consumers, set the assistant's clearance to public; for an internal integration, use an assistant with the matching clearance.

## How knowledge enters the brain

Ingestion runs in four stages: Conversation (every channel feeds the same pipeline; voice notes are transcribed on arrival) -> Extract (a background pass distils stable facts from each turn; mentions of yourself become identity candidates, mentions of others become entity candidates) -> Consolidate (a light pass dedups near-duplicates against existing memory and KB; a deep pass synthesises narratives, prunes stale rows, and adjusts confidence) -> Land (each row lands with tags, source, and a pointer to its episode).

## How the brain answers

Every turn fans in identity, relevant memories, knowledge base, tools and connectors, and the recent session, then composes one prompt (system prompt + selected context + tool catalogue + your turn). The model runs at the Standard / Pro / Max tier set in the chat header. Background work (extraction, embedding, classifiers) always runs Standard. The reply streams back to your channel and the loop begins again at Extract.

## Under the surface

Memory and KB are the two surfaces you see in chat. Beneath them sits the brain (entities, edges, episodes, plus tasks, CRM rows, and files), which is what the assistant actually retrieves over.

## Notes for agents

- Memory is keyed to (user, assistant): the same person talking to two assistants accumulates two separate memory sets.
- The KB is workspace-scoped: any member's assistants can search the same entries, subject to the assistant's clearance.
- Clearance gates KB reads on every channel including the public API. An assistant set to `public` clearance cannot read `internal` or `confidential` entries.
- To append durable workspace facts programmatically, use `addKnowledgeEntry` (KB), not memory writes; memory is auto-extracted from conversation.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Assistants](./assistants.md)
- [Workspaces & sharing](./workspaces.md)
