---
title: Memory & knowledge
description: Memory holds per-(user, assistant) facts about you; the knowledge base holds per-workspace facts the assistant can look up.
tags: [concepts, memory]
canonical: https://usebrian.ai/docs/memory
---

> Human-readable version: https://usebrian.ai/docs/memory

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

Each KB entry is tagged `public`, `internal`, or `confidential`. A synced entry takes its tier from an explicit `sensitivity:` frontmatter key when one is present; otherwise it gets the source's configured default (Studio -> Knowledge -> Clearance; `internal` unless changed). Explicit frontmatter always wins, so if you author markdown in a synced KB repo, stamp `sensitivity:` yourself whenever the source default is not what you mean. Changing a source's default re-syncs the whole source and re-stamps only the entries without an explicit tier.

The assistant's clearance gates what it can read across every channel, including the public API. To expose only public KB to API consumers, set the assistant's clearance to public; for an internal integration, use an assistant with the matching clearance.

### Self-maintaining sources

A synced source can run a maintenance agent (Studio -> Knowledge -> Self-maintain) that watches for drift and proposes KB updates. It is suggestion-first: every proposed write parks in the Approvals inbox, and nothing lands in the KB without a human approving it.

## How knowledge enters the brain

Ingestion runs in four stages: Conversation (every channel feeds the same pipeline; voice notes are transcribed on arrival) -> Extract (a background pass distils stable facts from each turn; mentions of yourself become identity candidates, mentions of others become entity candidates) -> Consolidate (a light pass dedups near-duplicates against existing memory and KB; a deep pass synthesises narratives, prunes stale rows, and adjusts confidence) -> Land (each row lands with tags, source, and a pointer to its episode).

### Durable application and recovery

Extraction success and application success are separate. A successful extraction freezes a normalized plan before derived entities, edges, memories, tasks, and finalization receipts are applied. Each item records a durable outcome, so a partial database or policy failure is visible instead of being mislabeled as a complete learning pass.

Studio -> Events shows incomplete application work. An authorized owner or administrator can confirm a retry there. Retry resumes only pending or retryable failed items from the same frozen plan; it does not rerun extraction, classifiers, embeddings, provider probes, or paid model work, and already-applied items are not written again. Held and rejected items remain intentional outcomes. Archived source episodes stay archived. Older episodes without a ledger are reported as `legacy_untracked`, never assumed complete.

## How the brain answers

Every turn fans in identity, relevant memories, knowledge base, tools and connectors, and the recent session, then composes one prompt (system prompt + selected context + tool catalogue + your turn). The model runs at the Standard / Pro / Max tier set in the chat header, or at a metered pay-per-use model profile when the workspace picked one (metered picks always pass an estimate-and-confirm step and bill 5 credits plus actual model cost). Background work (extraction, embedding, classifiers) always runs Standard. The reply streams back to your channel and the loop begins again at Extract.

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

When a member assistant derives a memory from canonical sources supplied by its caller, saveMemory preserves their restrictions and source links. A requested personal note stays personal even when the source is shared knowledge. A source change can withhold its dependent memories until review; stale source evidence is not permission to retry a write automatically. This applies to sources the current producer can identify and does not certify complete provenance across every input type.

### Learning from scoped corrections

Correction reflection uses versioned memory and brain verification receipts. It
keeps personal, assistant, clearance, department and project restrictions, and
separates incompatible audiences into different model batches. Editing a receipt
or its target can hold previously learned patterns until reviewed or regenerated
from current evidence. Whole-turn negative feedback keeps the exact assistant turn,
its initiating message, tool/read inputs and recalled memories in the same versioned
lineage. Procedural skill proposals likewise require a persisted canonical transcript
window, partition model inputs by the exact audience and preserve that evidence for
approval-time revalidation. Legacy receipts and unclassified skill inputs are withheld.
Workflow-origin procedural review remains unavailable until workflow definitions and
step outputs expose canonical scope receipts. This does not enable strict departmental
activation.

Retraction and soft-deletion reasons now use versioned correction receipts for
reflection. Learned patterns retain the saved and current target audience;
receipt/source changes or erasure invalidate their descendants. Older correction
receipts without saved scope are excluded, and snapshot/detail blobs are not
model inputs. Strict departmental isolation remains disabled until the complete
rollout barrier is met.

Department grant execution now carries separate read and mutation ceilings. A
read-only grant can support an audience-protected derived memory without enabling
source edits or live connector mutation. Canonical source RLS rechecks current
member permissions, including grant expiry/revocation, even for human reads. The
context picker can expose granted Teams without importing their foreign bundles.
This is still behind the incomplete departmental rollout gate; do not assume all
legacy endpoints or delivery/replay paths have completed acceptance.

The assistant Memory API carries the viewer's read and mutation ceilings through
edits, scope changes, verification and deletion. Read-only department access does
not authorize source changes. Pending-review lists and counts use the viewer's
scope before pagination. Personal scope changes retain the workspace partition;
workspace IDs are not a privacy toggle. Complete departmental activation remains
subject to the documented readiness barrier.

Department management commands now cover Team creation, metadata/archive,
read-package edits, membership and assistant audiences. Legacy Team and
page-sharing membership endpoints use the same service. Ordinary membership
changes preserve the current access mode; explicitly activating assigned mode
requires an administrator and complete departmental readiness. Read-package edits
change the ordinary membership package, including its existing mutation authority;
they are distinct from approved temporary read-only requests. The Settings
readiness display cannot treat v1 context checks alone as departmental readiness.

Person permission review is available in Settings > Department access > People.
`inspectWorkspaceAccess` returns a person's clearance, stored Team mode and separate
current read/membership reach only to that person or an administrator. Broad legacy
reach and owner/admin authority are displayed explicitly. `member.access.set` uses
the same canonical service for the UI and native tool, requires the reviewed policy
revision, and changes only an ordinary member's clearance and Team mode. It rejects
stale reviews, privileged targets and returning to legacy mode. Transitioning to
assigned mode requires complete departmental readiness. An unavailable release
keeps that transition blocked; the form is not evidence of strict isolation.

Department access tools now take an intent containing `command`, the inspected
`expectedPolicyRevision`, and a UUID `idempotencyKey`. Preserve the exact intent
when retrying. Confirmation prepares an immutable server review; execution consumes
that review and cannot create one silently. Web clients prepare at
`POST /api/workspaces/:workspaceId/access/command-review`, then apply
`{type: "access.command.apply", reviewId, payloadHash}` through `/access/commands`.
Reviews expire after fifteen minutes. Policy changes require a new review. A retry
of an applied review returns current filtered access without repeating its changes;
it does not replay an old privileged response. The legacy Team/assistant configuration and Team-kind page-sharing membership
adapters require the same saved review: attach `X-Brian-Access-Review-Id` and
`X-Brian-Access-Review-Hash` from the confirmed review. The shared CORS allowlist
accepts both headers. The route target, operation and body must match the saved
command; mismatch returns `access_review_changed` (409), including on replay.
Missing/malformed proof returns `access_review_required` (409). Omitted membership
`activateAssigned` is equivalent to false. Ordinary sharing-group membership
retains its existing contract. These adapters never prepare or confirm implicitly.

Retrying confirmation for an already-applied intent returns a receipt-only notice,
without replaying old permission details. Confirmation and execution both retain
the original intent identity, so a lost response does not require a new mutation.


Organization mutations now use the same saved-review lifecycle with a separate
intent: `{command, expectedRevision, expectedPolicyRevision, idempotencyKey}`.
Read `getOrganizationChart` for the chart revision and administrator-only
`initialization.policyRevision`. `updateOrganizationChart` prepares that intent
for per-call confirmation and execution consumes its existing review. For HTTP,
prepare at `POST /api/workspaces/:workspaceId/org-chart/command-review`, inspect
the exact unit/placement effects, and apply `{type:"org.command.apply",reviewId,
payloadHash}` at `/access/commands`. Raw `org.*` commands at the application
endpoint return `access_review_required` (409). Both revisions, current administrator
authority and the structural effects are rechecked. Stale revisions or changed
effects require a new review. Keep the original intent/receipt for uncertain-result
retries: a committed retry returns only the current filtered chart, and repeated
confirmation returns a receipt notice without old privileged effects. Organization
placement, reporting and accountability do not grant departmental data access.

Department access overview now returns the newest 50 authorized requests/grants
and independent `nextRequestCursor` / `nextGrantCursor` values. Continue with
`GET /api/workspaces/:workspaceId/access/requests` or `/access/grants`, passing
`after` and `expectedPolicyRevision` together; omit both to restart. Native
`inspectWorkspaceAccess` uses `{history:"requests"|"grants",after,expectedPolicyRevision}`
for the same service. Each page includes its policy revision, validity duration
and next cursor, without a total. Authorization precedes cursor lookup and limit.
A stale revision or unknown/inaccessible cursor returns `access_history_changed`
(409): discard the page and inspect the newest records. A grant history does not
become visible merely because the actor requested access for another beneficiary.
Old-request confirmation uses a direct authorized lookup rather than the overview
window. Mutation confirmation and application retain their saved-review contract.
