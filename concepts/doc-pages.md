---
title: Pages
description: A Notion-style page surface where chat assembles renderable, brain-bound views over workspace primitives.
tags: [concepts, doc]
canonical: https://usebrian.ai/docs/doc
---

> Human-readable version: https://usebrian.ai/docs/doc

Doc is a Notion-style page surface where chat assembles renderable views over your workspace primitives. It is the inbound counterpart to the outbound Feed surface: Doc marshals tasks, CRM rows, files, deals, and research findings into pages you can drag, save, and revisit. The whole app lives at `app.usebrian.ai` under a separate deployable.

## Feature walkthroughs

These are illustrative, step-by-step examples. They are not claims that a run, send, or publication has occurred.

### Create a page

Turn a request into a readable workspace page.

1. Open Page in your workspace and start a draft. Open the page's Brian chat dock.
2. Describe the page you want, the source material to use and the audience it should serve.

   > Example request: Create a launch brief page with an audience summary, key message and checklist. Use the information in this conversation and mark any missing details.

3. Inspect the page itself. Ask Brian to correct a section or edit the blocks directly before saving.

**What to check:** The result lives on the page, with headings and useful context. Missing source material should be identified, not filled with invented facts.

### Refine and organize

Change a section without losing the structure around it.

1. Open an existing page and locate the section you want to improve.
2. Use the chat dock to name the section and requested change, or edit and reorder its blocks directly.
3. Check the result on the page. Use subpages when a topic needs its own space, then save the page you want to keep.

**What to check:** The relevant section changes and the surrounding page still makes sense. Page authoring works with the selected workspace assistant; it does not require a special Page assistant.

### Build a live view

Keep a page connected to current tasks or CRM records.

1. Open a page and identify the exact workspace data you want to display.
2. Ask for a live data view, including the filter and grouping you need.

   > Example request: Add a live view of my open tasks, grouped by status. Include a short introduction explaining what this view shows.

3. Open the page again after the underlying records change. Check that the data block reflects current accessible records.

**What to check:** Bound data refreshes when the page opens. Surrounding explanation is authored content and should be reviewed when the meaning changes.

## Everyday workflow

1. **Ask for a page.** Try “show my tasks this week” or “show the pipeline by stage” in chat.
2. **Open and refine.** Follow the page link, then ask for changes in the chat dock. Your current assistant can do the editing.
3. **Save what you need.** Save the page to keep it. Its data blocks refresh from current workspace records when you return.

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

## External agent page edits

Use the brain MCP `createPage` tool for a new page and `editPage` for an existing page. `editPage` accepts Markdown with lists, headings, tables, and prose; `append` adds to the end and `replace` replaces the body while keeping the title. Mixed list and prose content preserves its authored order. No list-free workaround is required on a deployment with the append-order fix.

On a deployment with doc-sync, edits use the same collaborative document as the editor. `readPage` confirms stored content; it does not prove that a desktop or browser has completed live synchronization. If the editor shows “Reconnecting…”, check its sync connection before replacing the content again.

## Private member links

Internal page links use a deployment-owned address such as
`https://brain.example/s/product/roadmap`. They require the recipient to sign
into that exact deployment and pass the normal workspace/page checks. They do
not publish the page or change its grants. Renaming the page, workspace, or
link alias does not break links already shared.

The first-party Brian toolset exposes `getInternalShareLink` to ensure and
return a confirmed link for the current workspace or a readable `pageId`.
`setPageLinkAlias` changes a page alias when the actor has the existing
page-share management authority. Never invent an alias or construct a pretty
URL from a title: use the returned confirmed URL. These commands are Brian
runtime tools; they are separate from the brain MCP page-authoring surface.

## Related

- [Workflows](./workflows.md)
- [Brain (entities & episodes)](./brain.md)
- [Tasks](./tasks.md)
