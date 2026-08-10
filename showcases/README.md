---
title: Use Brian showcases
description: Discover outcome-led Use Brian scenario specifications, their required context, intended result, capabilities, and evidence boundaries.
tags: [showcases, use-cases, discovery, catalog, agents]
---

# Use Brian showcases

This is the public discovery catalog for outcome-led Use Brian scenarios. It is designed for both humans and agents deciding what Brian may be suitable for.

## Evidence status

Every current record has `evidence_level: specification`:

- It describes a synthetic, reviewable scenario and its expected result.
- It is approved for public discovery.
- It is **not** evidence that the product has passed the scenario.
- It is **not** a customer example, benchmark, or universal capability claim.

The `customer-meeting-to-cited-brief` scenario is additionally marked `needs-rework` because its retained product run exposed incomplete transcription and document-generation behavior. The other 19 are `prepared` and awaiting product rehearsal.

## Machine access

- [`catalog.json`](catalog.json) — all records, tags, capability labels, and evidence status
- [`schema.json`](schema.json) — JSON Schema for validating the catalog

Agents should filter by `category`, `tags`, or `capabilities`, then read the linked Markdown record. Treat `expected_result` as a scenario contract, not observed evidence.

## Catalog

| Outcome | Category | Status |
|---|---|---|
| [Turn a customer meeting into a cited operating brief](customer-meeting-to-cited-brief.md) | Customer relationships | needs-rework |
| [Turn three weekly updates into one exception brief](weekly-inputs-to-exception-brief.md) | Operations | prepared |
| [Turn a small pipeline into a prioritized follow-up plan](pipeline-to-follow-up-plan.md) | Sales | prepared |
| [Turn a qualified opportunity into a bounded proposal draft](opportunity-to-proposal-draft.md) | Sales | prepared |
| [Turn scattered onboarding notes into one operating SOP](scattered-notes-to-operating-sop.md) | Team operations | prepared |
| [Turn weekly metrics and notes into an executive update](weekly-metrics-to-executive-update.md) | Leadership | prepared |
| [Turn support tickets into a prioritized triage brief](support-tickets-to-triage-brief.md) | Customer success | prepared |
| [Turn customer feedback into evidence-backed product themes](customer-feedback-to-product-themes.md) | Product | prepared |
| [Turn account signals into a renewal risk brief](renewal-account-to-risk-brief.md) | Customer success | prepared |
| [Turn a job posting into an outcome-led demo plan](job-posting-to-outcome-demo-plan.md) | Go-to-market | prepared |
| [Turn a prospect pack into a discovery brief](prospect-pack-to-discovery-brief.md) | Sales | prepared |
| [Turn a campaign goal into an approval-ready lifecycle email](campaign-goal-to-approval-ready-email.md) | Go-to-market | prepared |
| [Turn project updates into a delivery risk register](project-updates-to-delivery-risk-register.md) | Delivery | prepared |
| [Turn vendor quotes into a decision-ready comparison](vendor-quotes-to-comparison.md) | Operations | prepared |
| [Turn incident notes into a blameless postmortem draft](incident-notes-to-postmortem.md) | Engineering operations | prepared |
| [Apply workspace policy when personal instructions conflict](policy-and-request-to-governed-answer.md) | Governance | prepared |
| [Turn handover notes into a role continuity brief](handover-notes-to-role-brief.md) | Team operations | prepared |
| [Turn expenses into a cash watchlist](expenses-to-cash-watchlist.md) | Finance | prepared |
| [Turn open invoices into a collections review plan](invoices-to-collections-plan.md) | Finance | prepared |
| [Turn a recurring process into an approval-gated workflow spec](recurring-process-to-workflow-spec.md) | Automation | prepared |

## Shared boundaries

- All examples use synthetic data.
- Customer-specific in-app content is never used for selling or showcase production.
- External writes remain exact-action approval-gated.
- A public scenario record is not permission to send, schedule, purchase, invite, configure, or publish anything.
- Video and poster URLs remain absent until a separately approved verified run exists.
