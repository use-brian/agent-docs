---
title: CRM
description: First-party contacts, companies, and deals that the brain reads and writes the same way it reads memories.
tags: [concepts, crm]
canonical: https://sidan.ai/docs/crm
---

> Human-readable version: https://sidan.ai/docs/crm

CRM in sidanclaw is people, companies, and deals: the durable graph of who the team talks to, where the relationship stands, and what it is worth. It is first-party, so the brain reads and writes contacts the same way it reads memories: same brain graph, no translation layer. Attio and HubSpot are sync targets, not the primary surface.

## Entity-backed, not a separate store

CRM is not a separate database. Every contact (person), company (organization), and deal (opportunity) is a node in the same brain graph as your memories and tasks. Frozen v1 shape: no custom fields, no priority, no description. `tags` are a string array; `external_ref` is a free-form JSONB passthrough for synced rows.

## Deal stages

Six values, locked:

| Stage | |
|---|---|
| `lead` | Unqualified opportunity. |
| `qualified` | Vetted as a real opportunity. |
| `proposal` | Proposal or quote out. |
| `negotiation` | Terms in discussion. |
| `won` | Closed successfully. |
| `lost` | Closed without a deal. |

Adding a stage requires a migration plus a tool-description update plus an analytics taxonomy update, by design. No custom pipelines in v1.

## Amounts and dates

`amount` is decimal dollars (USD-equivalent), not cents. Users type "50000", not "5000000". `close_date` is a calendar date; "Q3 close" is not a wall-clock instant. Currency is implicit USD; a `currency_code` column ships when multi-currency does.

## Chat tools

Every assistant with the `crm` capability gets the full CRUD surface plus `advanceDealStage` as the canonical stage-transition verb (separate from `updateDeal`, which will not accept a stage). There are no delete tools in v1: soft-delete is `advanceDealStage(id, 'lost')` and field-nulling.

`saveContact` / `getContact` / `listContacts` / `updateContact` · `saveCompany` / `getCompany` / `listCompanies` / `updateCompany` · `saveDeal` / `getDeal` / `listDeals` / `updateDeal` / `advanceDealStage`

## Relationships across the brain

Every CRM row is an entity in the underlying graph. Save a memory about a contact, link a task to a deal, or open an explicit edge via the universal `links` param: the entity rollup (`getEntity`) returns the contact, every memory anchored to it, and the deals it is attached to in one call.

## Notes for agents

- Change a deal's stage only through `advanceDealStage`; `updateDeal` rejects the stage field. The six stages are fixed, so do not attempt custom pipeline values.
- Write `amount` as decimal dollars ("50000" = $50,000), never cents. Write `close_date` as a calendar date, not a timestamp.
- There is no delete: closing a deal is `advanceDealStage(id, 'lost')`; removing other rows is field-nulling.
- CRM rows are brain entities, so use the `links` param and `getEntity` to connect and retrieve related memories, tasks, and deals in one place rather than treating CRM as a separate store.

## Related

- [Brain (entities & episodes)](./brain.md)
- [Tasks](./tasks.md)
- [Memory & knowledge](./memory-and-knowledge.md)
