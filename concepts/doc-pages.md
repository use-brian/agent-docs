---
title: Doc
description: A Notion-style page surface where chat assembles renderable, brain-bound views over workspace primitives.
tags: [concepts, doc]
canonical: https://sidan.ai/docs/doc
---

> Human-readable version: https://sidan.ai/docs/doc

Doc is a Notion-style page surface where chat assembles renderable views over your workspace primitives. It is the inbound counterpart to the outbound Feed surface: Doc marshals tasks, CRM rows, files, deals, and research findings into pages you can drag, save, and revisit. The whole app lives at `app.sidan.ai` under a separate deployable.

## Chat is the page author

You do not open Doc to write a page; you tell the doc assistant what you want to see ("my tasks this week", "the Q3 pipeline by stage", "everything in the brain about Acme"). The assistant emits a `renderView` tool call, which creates a draft page server-side and streams a deep-link pill back into the chat. Click the pill to open the full page in Doc; the chat dock stays open in the bottom-right so you can refine without leaving.

## Brain-first authoring

Every data block on a Doc page is bound to a workspace primitive (tasks, CRM contacts, files, deals, entities, memories), not a snapshot. Open a saved page tomorrow and the data block re-resolves against current state. Editing the underlying primitive in chat changes the page; editing the page nudges the primitive. There is no "sync". The page is a view, not a copy.

## Drafts, saved, prune

Every `renderView` creates a draft row in `saved_views` with `state='draft'`. Drafts auto-prune 30 days after their last touch (any read or edit bumps the deadline). Click "Save" on a page to flip `state='saved'` and clear the prune timestamp. Saved pages live forever. Click "Unsave" to drop back to draft. The sidebar lists both groups side by side.

## A2UI views

Pages are composed of block-shells (text, heading, divider, data, chart). Data blocks render through the A2UI v0.8 renderer with the same property registry as chat-inline views: Table, Board, KPI, BarChart, LineChart, PieChart. The renderer is share-safe (no eval, no model output reaches the DOM directly) and dark-mode aware via Tailwind theme tokens.

## Any assistant authors pages

Doc editing is a context-injected skill, not a dedicated assistant. The default interlocutor is your workspace primary, and the chat dock lets you switch to any other accessible assistant; whichever one runs on the Doc surface gets the page tools (tasks, CRM, views, files) injected automatically. The skill steers it to "render, don't narrate": visibility requests ("show me", "list", "everything about") begin with a `renderView` call, while analytical questions still answer in prose.

## Notes for agents

- To surface data as a page, emit `renderView`; it creates a draft and returns a deep-link pill. Visibility requests ("show me", "list", "everything about") should render; analytical questions answer in prose.
- Data blocks are live views bound to primitives, not snapshots. A saved page reflects current state on every open, so do not re-render just to refresh data.
- Draft pages auto-prune 30 days after their last touch; call "Save" to make a page permanent. Reads and edits bump the deadline.
- Any accessible assistant can author on the Doc surface because the page tools are injected by the skill; you do not need a dedicated doc assistant.

## Related

- [Workflows](./workflows.md)
- [Brain (entities & episodes)](./brain.md)
- [Tasks](./tasks.md)
