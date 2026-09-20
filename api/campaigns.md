---
title: Native campaigns, attribution, and approved email
description: Install first-party campaign tracking, record trusted conversions, and operate approved SMTP broadcasts without an analytics vendor.
tags: [api, campaigns, attribution, crm, email, smtp]
canonical: https://usebrian.ai/docs/api/campaigns
---

# Native campaigns, attribution, and approved email

Use Brian provides Feed campaigns, stable tracked links, first-party browser
observations, trusted business conversions, CRM attribution, and approved SMTP
broadcasts in the open-source edition. No analytics vendor is required.

## Browser installation

Register each exact allowed origin on a campaign site, then load
`GET /api/campaign-tracking/tracker.js` from the same Use Brian API that accepts
events. Initialize `window.BrianCampaign` with:

- `collectUrl`: `/api/campaign-tracking/collect` on that API.
- `siteId`: the public site ID. It is not a credential.
- `enabled`: explicit collection state.
- `storageMode`: `none` by default or `first_party` after consent.
- `storageAllowed`: true only after the host preference flow grants it.
- `cookieDomain`: optional registered sibling-domain scope.
- `test`: true for synthetic traffic excluded from production totals.

The tracker emits `page_view` for initial and SPA navigation. Call
`track("cta_clicked", metadata)` or `track("form_started", metadata)` for
bounded interaction markers. Never send form values, full URLs, identities, or
credentials. `disable()` stops events and clears Brian-owned continuity.

## Trusted conversions

Send `POST /api/campaign-tracking/conversions` from a backend with a scoped site
credential after the business outcome commits. Required fields are `version: 1`,
the public `siteId`, `conversionKind`, stable `externalOutcomeId`, stable
`occurredAt`, and bounded `metadata`. Forward only the tracker's bounded
attribution context when present.

Use the exact same payload on retry. Exact replay is a duplicate and changed
reuse conflicts. Browser events and public clicks cannot assert a verified
conversion or CRM contact identity. CRM-linked outcomes come from committed CRM
intake and its transactional outbox.

## Links, reporting, and limits

Tracked links preserve unrelated query fields and fragments. They add standard
UTMs plus opaque `brian_link`; destination is immutable after creation. Two
placements report independently while one trusted outcome deduplicates. Manual
social publication does not require provider OAuth.

Reports separate redirect requests, page views, continuity-backed sessions,
verified conversions, CRM leads/deals, and SMTP acceptance. Missing evidence is
Unattributed. Provider impressions, inbox delivery, opens, replies, bounces,
and complaints stay unavailable without provider evidence.

Defaults: 30-day attribution lookback, 30-day first-party visitor continuity,
90-day raw observation retention, 13 calendar months of aggregates, 500
recipients per broadcast, and 10 SMTP handoffs per minute per sender.

## Approved SMTP email

Subject, preheader, body, sender, purpose, personalization, links, and recipient
snapshot are one immutable approved Feed revision. Current CRM consent,
suppression, member authority, and sender authority are rechecked immediately
before each handoff. Later segment matches are never added.

Every recipient receives a separate envelope with a visible native unsubscribe
link. GET shows a non-mutating preview; POST records withdrawal. RFC 8058
one-click is advertised only when DKIM covers the required headers. An
ambiguous post-DATA result becomes `needs_reconciliation` and is not retried
automatically. Pause or cancel affects only recipients not yet admitted.

## Failure and privacy contract

Collector failure never breaks a valid redirect, signup, or enquiry. An
unconfigured client emits nothing and reports the integration as unavailable.
The tracker stores no raw IP or user-agent string, full URL, form value,
fingerprint, email hash, or open pixel. Subject/workspace export and erasure
cover linked events, credentials, tokens, queued work, and cached projections.
