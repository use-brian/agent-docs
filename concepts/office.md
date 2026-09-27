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

Read lifetime annotations in the web client belong to the requesting viewer and
are discarded on expiry or authority refresh. They are not stored in template
routing or snapshot JSON. Refetch protected metadata after a viewer change; copying
a prior read into another cache or command does not grant fresh authority.


The template picker and editable routing inspectors apply these same bounded
reads. Creation forms disappear when their template list expires; routing drafts
are discarded on expired or changed authorization and on changed server routing.
Identical authorized routing renewals preserve pending edits. A routing PUT
acknowledgement does not extend read access: the browser performs a fresh bounded
GET before displaying the saved routing. Late completions from an expired or
replaced viewer/workspace cannot restore drafts or navigate from creation.


The web activity panel renews both job details and events, including terminal
jobs, and clears expired activity. Version-history previews, rename/copy prompts
and pending actions belong to a current authorized version list. Expiry or viewer
replacement clears that state; a late preview or copy response cannot restore it
or navigate. Server preview/mutation authorization remains independently required.


Offline packages and queued edits belong to the viewer and workspace that created
them on the device. Account changes do not adopt another viewer's cached content
or replay their edits. Separate non-extractable encryption roots protect each
viewer/workspace partition, and pending storage work refuses changed identities.
Unowned legacy ciphertext and its keys remain quarantined, without automatic
replay or disclosure. Re-pin old cached packages through a current authorized
request. Legacy journal recovery and current offline read/revocation authorization
remain required follow-up work; storage partitioning alone does not complete the
isolation rollout. Artifact and snapshot browser caches also include the viewer
and workspace, with local editor/panel state reset on identity replacement.


Online comment and suggestion panels use bounded metadata reads. Expiry or
identity/authority invalidation removes their content, drafts and editor highlight
projections. A mutation acknowledgement does not extend that read lifetime; the
panel refetches an authorized collection. Pending bulk decisions, queued-comment
replay and comment-triggered job polling stop when their owning read expires.
Write-capability loss also stops subsequent bulk actions. Offline read/recovery
and shared mention-directory authorization remain separate required boundaries.
Bounded discussion collections alone do not certify all editor retention. Cold
pinned-package authority remains a separate required closure.


The document editor's comment and suggestion decorations now read the same bounded
collections as their panels, including while the panels are closed. Detachment
acknowledgements perform authorized readback and cannot refill an old local array.
Delayed detachment stops when its read, viewer or editable artifact is no longer
current. Local queued comments remain separate from fetched server comments, so
closing/reopening an offline panel preserves local additions without extending
server comment retention. Pinned-package authority remains independently required.


Online artifact and snapshot reads also expire. List-page rows used for immediate
editor chrome keep the original collection deadline. Live working snapshots do
not renew that deadline, and the editor removes its content, selection and
presentation state when either read loses authority. Successful command and
initialization responses require a fresh bounded GET before their content is
shown; a delayed acknowledgement cannot restore access. Once online ownership
has been established, a previous device package cannot bypass expiry or denial.
Cold offline authorization/recovery and canonical collaboration delivery remain
required independent boundaries; this adapter does not complete the rollout.
