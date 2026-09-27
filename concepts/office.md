---
title: Office
description: Create, review, and refine documents, presentations, and spreadsheets with Brian.
canonical: https://usebrian.ai/docs/office
tags: [office, documents, presentations, spreadsheets]
---

# Office

Office is a first-party workspace surface for Documents, Presentations, and
Spreadsheets. It is available in the open-source core. Hosted services and
model access depend on the deployment configuration.

## Start from a template

Open Office from Home. Upload a supported DOCX, PPTX, or XLSX template or
generate one from guidance. Review the editable template and publish it before
creating files from it. Choose the template and describe the intended result.

## Edit and review

Brian creates an editable draft. Supported assistant revisions are scoped to
selected content and use the same canonical commands and authority as direct
editing. Do not assume an arbitrary operation is supported: use the available
tool schema and the returned capability errors. Users can also refine work in
the native editor. View, Comment, and Edit roles, comments, document
suggestions, history, and restore support team review under workspace access.

## Supported file exchange

DOCX, PPTX, and XLSX support is a defined subset. Unsupported constructs fail
admission instead of being silently discarded. Do not promise arbitrary
Office-suite fidelity, macros, external workbook links, pivot tables, or
unsupported formula functions. Review and release remain separate from
creating or editing a draft. A prepared draft is not proof that a file was
shared or an external message was sent.


## Department access

A current temporary department read grant can make an Office artifact readable.
It does not grant comment, edit, restore, sharing-management, or deletion rights,
even when a stored Office role otherwise allows the operation. Its effective role
is View until mutation scope is also available. Mutation also requires
ordinary department reach and the current execution scope. Read and mutation
checks apply to the artifact and its persisted child records. Recheck the returned
capabilities instead of treating an earlier read or retained artifact ID as
continuing authority. Organization placement alone does not grant access.

Template and resource metadata also depend on current access to their linked
drafts, bundle files and declared resource dependencies. A retained version or
resource ID does not bypass a revoked grant, private file, or held source.
Template publication records its declared resource links atomically with its
version and head; an inaccessible dependency prevents publication. Temporary
read grants do not authorize library edits or publication.

Office resource byte responses use authenticated no-store delivery and the
`X-Brian-Media-Valid-For-Ms` header. Clients must subtract request/body-transfer
time from that lifetime, discard expired bytes, and clear retained projections
when the viewer or workspace changes. An immutable resource hash never authorizes
permanent caching. Each read rechecks the live artifact reference, current resource
and durable file authority before returning bytes.

Office SQL metadata lists, details, live snapshots, comments, suggestions, template
routing and job/events reads return `X-Brian-Projection-Valid-For-Ms` with no-store
responses. Revalidate retained metadata before that lifetime expires, including
request time in the budget. A `409 office_projection_changed` means the read's
content or authority changed before publication: discard any old projection and
make a fresh read. It does not authorize retrying a mutation.
