---
title: Association operations API
description: Association API retirement and the CRM operations that remain available.
tags: [api, crm, memberships, events, integrations]
---

# Association operations (retired)

Association was retired on 2026-11-15. The native app, assistant tools,
Association HTTP endpoints and commerce import adapter are unavailable.
Migration `8270_retire_association.sql` deletes Association commerce records,
website content, configuration and grants. Generic CRM contacts, consent,
submissions, plans, events and independent entitlements/participation remain.
Historical `association_*` table names are still used by shared CRM storage;
they do not indicate that the retired module is available.

Use [CRM operations](crm-operations.md) for supported CRM integrations.
