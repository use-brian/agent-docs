---
title: Pricing and Credits
description: Plans, the per-message tier menu, credit rates, overage, and safety caps for hosted sidanclaw.
tags: [operations, pricing]
canonical: https://sidan.ai/docs/pricing
---

> Human-readable version: https://sidan.ai/docs/pricing

Hosted sidanclaw runs on credits. Billing is monthly and scoped to a workspace, not a user. One credit is roughly one Pro chat message; the tier menu lets a message spend fewer or more credits.

## Plan ladder

Every new workspace starts with a 30-day Pro trial. After that, hosted sidanclaw needs a paid plan (or self-host the open-source version). Plans differ only in monthly credit allocation. There is no free plan and no per-seat fee.

| Plan | Price | Credits / month | For |
|---|---|---|---|
| Pro | $20 / month | 1,000 | Solo founders and small teams. The default |
| Max 5x | $100 / month | 5,000 | Heavy-use teams |
| Max 10x | $200 / month | 10,000 | Whole-business workflows and dense agentic work |
| Enterprise | Custom | Custom | SSO, audit log, security floors, data residency, self-hosted brain, SLA |

## Tier menu

Each message costs credits based on the tier picked in the chat header. Background work (extraction, embeddings, classifiers) always runs on Standard and is bundled into the message price.

| Tier | Credits / message | Current model | Use |
|---|---|---|---|
| Standard | 1 | Gemini Flash 3 | Routine queries, status checks, quick lookups. Same model as Pro on a tighter tool budget |
| Pro | 2 | Gemini Flash 3 | Default tier. Balanced reasoning for most work |
| Max | 10 | Gemini Flash 3.5 | Premium tier for hard reasoning and deep agentic work (doubled turn budget) |
| Research | 20 | Gemini Pro 3.1 | Deep web synthesis on a long multi-step budget. Opt in per turn; Max-plan only |

## Per-operation rate

MCP tool calls and brain memory operations bill at a flat `0.1` credits per operation, drawn from the workspace credit pool. This covers external-agent recall, write, and note operations without running the full chat loop.

## At-cap behavior

A workspace that exhausts its monthly credits behaves differently depending on whether it has an active plan. There is no automatic overage billing, ever.

| Situation | Behavior |
|---|---|
| Paid workspace, credits exhausted (default) | Not blocked and not `429`'d. Further messages are automatically served on the Standard tier for the rest of the billing period; feature access is unchanged |
| Paid workspace, wants full speed back | Buy an extra usage pack or upgrade the plan; either restores full-tier access for the current cycle |
| No active plan (trial ended, no plan picked) | Messages are blocked with `429` until a plan is chosen |

## Extra usage packs

The only way to add credits mid-cycle is a prepaid one-time pack: no metered or automatic overage exists.

| Field | Value |
|---|---|
| Pack | $100 for 2,500 credits |
| Effective rate | $0.040 per credit |
| Purchase | One-time, by the workspace owner |
| Scope | Applies to the current billing cycle only |

## Trial

Every new workspace gets a 30-day Pro trial: 1,000 credits per month, the full tier menu, no credit card to start. After the trial the workspace pauses until a paid plan is picked: data stays intact and exportable, but assistant compute stops. The first 25 accounts at launch receive a 90-day variant of the same trial.

## Per-workspace billing

Each workspace is its own billing entity with its own plan, credit pool, and members. One account can own multiple workspaces, each billed independently. Inviting teammates never changes the bill.

## Notes for agents

- Brain/MCP operations are cheap (`0.1` credits each) but not free: they draw from the same workspace pool as chat.
- A `429` means the workspace has no active plan (trial ended, no plan picked), not a transient error. Do not blindly retry; the workspace owner must pick a plan.
- A paid workspace never hard-blocks on running out of credits: it silently drops to the Standard tier for the rest of the billing period. Expect a cheaper model on that path, with feature access unchanged. Full speed returns only when the owner buys an extra usage pack ($100 / 2,500 credits, current cycle only) or upgrades the plan; there is no automatic overage billing.
- After a trial ends without a paid plan, assistant and MCP compute stops while stored data remains readable through export.

## Related

- [Privacy and data](privacy-and-data.md)
- [Brain MCP server](../mcp/brain-mcp.md)
- [Self-hosting](../self-hosting.md)
