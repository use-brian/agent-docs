---
title: End-user bridge
description: Serve your own end users through one Use Brian assistant. Scope comes only from the identity headers; no tool may take a principal-selecting argument, so a cross-customer request is inexpressible rather than denied.
tags: [mcp, api, identity, security]
---

You want a Use Brian assistant in front of **your own end users** — customer support, member services, an account portal. Each user is authenticated by your login, each conversation must stay private to that user, and answers must come from that user's records.

The obvious design is broken. If you expose `getOrders(customerId)` and instruct the assistant in its prompt to only ever pass the current customer's id, the model still chooses the argument. A prompt is a preference, not a boundary. One injection, one hallucinated id, one ambiguous follow-up and it reads another customer's data, silently, looking correct in the logs.

Build it so the leak is **inexpressible** instead.

## The contract

> Scope derives only from the identity transport. No tool may accept a principal-selecting argument.

Both halves are required.

1. **Use Brian forwards the principal out of band.** Identity arrives as reserved HTTP headers, resolved by Use Brian's backend from the authenticated API key and attached *after* the model produced its tool call. The model never reads them and cannot set them.
2. **Your bridge scopes on the header and offers no way to ask about anyone else.** Self-scoped signatures: `getMyOrders()`, `getMyInvoice(invoiceId)`. Never `getOrders(customerId)`. Never an optional `customerId` that defaults to the caller.

A tool with no parameter naming a person cannot be made to name one. Validating a `customerId` is strictly weaker: it holds only while your check is present, correct, and never refactored away.

## Trust model

Use Brian never authenticates your users. Your backend does, then attests the identity on each request. Use Brian acts on behalf of that attested principal against your bridge. Same shape as OAuth token exchange: a claim is exactly as trustworthy as the key holder, which is the intended bar.

```
your user ──► your backend ──► POST /api/v1/assistants/{id}/messages ──► turn ──► your bridge MCP
              (authenticates)   externalUserId + claims                  (headers)  (scopes to that user)
                                Authorization: Bearer sk_live_…
```

Keep the key **server-side**. A browser-held key lets any visitor attest any identity and the whole model collapses.

## Headers your bridge receives

Sent on discovery and every tool call, once the workspace owner enables identity for your connector. Each is also emitted in the legacy `X-Sidanclaw-*` namespace.

| Header | Value | Use it for |
|---|---|---|
| `X-UseBrian-Actor-Channel` | `api` | Rejecting channels you do not serve |
| `X-UseBrian-Actor-Id` | the `externalUserId` your backend sent | **The lookup key** |
| `X-UseBrian-Actor-Email` | `claims.email`, when attested | Display, correlation. Not a key. |
| `X-UseBrian-Actor-Org` | `claims.orgId`, when sent | Tenant selection |
| `X-UseBrian-User-Id` | stable Use Brian user id | A key that survives the user arriving on another channel |

`claims.roles` is **not** here, deliberately. Roles are advisory and prompt-visible only. Identity is forwarded, authorization is derived: resolve what an actor may do from your own records keyed on the actor id. A role travelling in a request is a permission claim made by something that is not the enforcement point.

**Absent claims mean gates closed.** Use Brian does not carry a previous session's claims forward, so a user who signed in months ago and now browses logged out arrives anonymous. Do the same: an actor id you cannot resolve means no data, never a default user.

## Why the headers are trustworthy

- The `X-UseBrian-` namespace is reserved and stripped from user-configurable connector headers, so a workspace member cannot forge identity through settings.
- Actor headers merge at highest precedence, over the connector's auth headers and any static preflight headers.
- Values are resolved server-side from the authenticated key, never from model output.

That covers everything up to your door. The last gap is yours: see item 1 below.

## Conformance checklist

1. **Authenticate the connection.** Bearer token or custom auth header on your connector. The identity headers are meaningful because only Use Brian can reach your authenticated endpoint — like a reverse proxy adding `X-Forwarded-User`. **On an open endpoint they mean nothing**: anyone can send them. Most important item here.
2. **Authorize on the header, never on a tool argument.**
3. **No identity parameter in any tool schema.** Not `customerId`, `userId`, `email`, or an "optional" override. If it is in the schema, the model fills it.
4. **Key on a stable id** (`X-UseBrian-Actor-Id` or `X-UseBrian-User-Id`). Not email.
5. **Fail closed.** Missing header, unknown actor, unresolvable actor: error. Never a default user, an admin context, or an unscoped query.
6. **Run the cross-customer probe.** As customer A, ask for B's data — by name, by id, as "the other order", and via an instruction injected into a message body. Passing is not "the request was denied". Passing is that **no tool call expressing it can be constructed**. If you are writing a denial branch for a cross-customer argument, the signature is wrong: remove the argument.

## Write tools

Write tools work, with one wrinkle. The API channel strips confirmation-required tools after injection, because a programmatic consumer has no Approve/Deny loop — and MCP injectors tag writes as confirmation-required by default. So a freshly connected bridge's write tools silently do not appear.

The workspace owner sets that specific tool's policy to **allow** (Studio, the assistant's Connectors tab) to let it through. Per tool, deliberately.

That is safe because of the contract: a self-scoped write can only touch the acting user's own records. It is not safe for a tool taking a target id. Any confirmation UX belongs on your side, where the user is and where you have a UI to ask in.

## Setup

Owner-side, once:

- **A dedicated assistant** at `public` clearance. Not the team's primary — that one carries workspace clearance, team memory, and company connector grants.
- **Zero connector grants except your bridge.** Every other connector runs as the *workspace owner*, not the end user, so an unrelated grant is a path from an end-user turn into company data. The bridge is the only connector whose scoping follows the end user.
- **`sendActorIdentity` enabled** on the bridge connector, plus its bearer token. Without it no headers are sent and every user looks identical to your server.
- **A `chat`-scope key** in your backend's server environment.
- **A knowledge-base sensitivity pass.** Anything marked `public` is now customer-facing copy.

Per request: send `externalUserId` always, `claims` whenever the user is authenticated, and `endUserContext` when per-turn account state would save the assistant a tool call. See [Send messages](../api/messages.md).

## Anti-patterns

- A `customerId` parameter with a server-side check. The check is the weak form. Delete the parameter.
- Trusting `claims.roles` for access. They never reach your server, and would not be authority if they did.
- An unauthenticated bridge endpoint.
- Reusing the team's primary assistant.
- Holding the API key in the browser.
- Prompt-only scoping.

## What the workspace accrues

An identified end-user turn accrues into the workspace brain, behind a per-user wall:

- The user is materialized as a **CRM contact** the workspace team can see, paired to the `externalUserId` your backend attests.
- Everything the turn writes is stamped into a per-user compartment. One end user's turns can never read what another end user's turns wrote, and cannot read their own contact row either: accrual is write-only from the end-user side.

The stored pairing is identity bookkeeping, never authority. It is not read back on later turns, so absent claims still mean gates closed.

## Not available yet

- **A signed principal assertion.** Per-connector bearer auth already proves the request is Use Brian for a bridge serving one workspace; a signature is only needed once one bridge serves many.
- **Principal-scoped built-in connectors.** If your user data lives in a SaaS Use Brian already integrates and you have no backend, there is no supported shape today.
- **End-user reads of workspace CRM data.** A user cannot read what the brain has accrued about them.

## Related

- [Send messages](../api/messages.md): the `claims` and `endUserContext` fields
- [Connector identity](../api/connector-identity.md): the header rail in general, across all channels
- [Identity & memory](../api/identity.md): anonymous vs identified tiers
- [Brain MCP Server](brain-mcp.md): the other direction — an agent reading a user's own brain

## Protected delegated connector profile (local implementation checkpoint)

The Node-only `@use-brian/brian-app/connector` entry provides strict Ed25519
`ub.actor.v1` assertion verification and source-owned authorization callbacks.
This profile requires a separately authenticated transport, pinned installation
issuer/source audience/workspace/instance/public keys, actual initiator and
executing-assistant mapping, exact method/encoded request target/raw-byte digest,
and atomic replay consumption. Tokens last 60 seconds with at most five seconds
clock skew. The bounded single-process replay consumer refuses exhaustion;
clustered sources supply shared atomic storage. Token-provided key URLs are refused.

A valid assertion alone grants no source access. The source invokes its current
canonical ACL after transport and signature checks. The readable assistant ID
header accompanies the existing identity opt-in; config and dynamic hooks cannot
replace reserved identity/assertion headers. Deployment keys are independent
Ed25519 keys stored encrypted by Brian; only public JWKS is exposed at
`/.well-known/usebrian-connector-jwks.json`. Rotation retains retiring public keys
for the last assertion window, while emergency revoke withdraws them immediately.

This SDK/key-storage/wire checkpoint does not activate protected delegated trust,
source retention or broadened discovery. Department admission, current source
rights, complete evidence/retention consumers, management and runtime qualification
are still being implemented. Existing installations must not treat legacy headers
or this documentation as proof of that acceptance barrier.

The shared channel adapter now intersects enabled connectors with the actual
sender's current native Department/Project scope and its existing audience
ceiling. Legacy assistant-only channel settings and room scope cannot grant
staff authority to an unlinked/non-member sender. Explicit Department edges are
checked even under a legacy workspace read flag; owner/admin universe reads do
not replace them. External-app/public-bot resource releases remain separate.
This channel checkpoint does not certify common dispatch or source retention.

The common injector now adds a current native actor/assistant floor and keeps
actor-specific discovery caches separate. Retained gateway/direct tools refuse
observed native access loss, connector disconnection, changed exposure metadata
or custom transport credential replacement, and withhold a result after contraction.
An enabled shared assistant or credential owner does not supply the initiating
person's Department membership. This native checkpoint remains separate from
protected source ACL/field/retention and complete consumer qualification.

Remote search now advertises stable `connector:<instance UUID>` identities beside
human labels. Unique legacy labels and existing direct aliases remain accepted;
ambiguous labels require selecting the stable identity returned by search. Dispatch
keeps the selected instance's transport and credentials even when two connectors
share a URL. Hook metadata separates the actual actor, human server label and
stable source ID. This checkpoint does not activate protected source trust.

The Node connector SDK now validates `ub.source.v1` evidence, capabilities and
Brian-specific `POST /usebrian/source/{capabilities,authorize,reconcile}` contracts.
Use the source's own current authorization functions for both ordinary reads and
sidecar decisions. A valid actor assertion without a source mapping grants no
records. Evidence names exact resource/content/policy versions, returned fields,
subjects, actions, retention and export rights. Missing authorization pairs deny;
short receipts bind an exact operation and never act as standalone bearer grants.

MCP evidence travels in reserved `_meta["ub.source.v1"]` over the authenticated
source connection. HTTP adapters keep content and the validated envelope separate.
JSON embedded in tool display text cannot create trusted evidence. This local
checkpoint includes a coordinator and independent persisted HTTP/MCP fixture;
protected runtime trust, retained descendant readers, all consumers and management
remain unqualified. Production activation and provider adoption remain separate.

Brian now maps each trusted source reference to a stable local UUID and keeps
immutable field observations separately. A narrower later read cannot replace
earlier field evidence. Current source decisions can be pinned to one short local
SQL operation; transaction, actor/assistant/recipient, fields, versions and local
authority changes are checked again. Missing external evidence is held or
unverifiable, including during outage/withdrawal; it never becomes an allowed
historical copy. The initial SQL adapter covers attended native human fixtures.
Other principal/lane adapters, complete retained consumers and runtime qualification
remain required before protected source activation.

The attended native read adapter now reaches the common custom-MCP injector and
web-chat boot seam. It checks the exact current execution/accumulator, sends fresh
signed discovery rather than reusing permission-bearing catalogs, and keeps each
source's current field dictionary. Successful reads bind every immutable field
observation before display conversion. Pending/revoked profiles, missing keys or
source mappings refuse with a setup/access recovery notice; they never choose a
legacy owner dispatch. Profiles require HTTPS for remote sources.

This remains a local read-profile checkpoint. Protected commands, application or
service releases, durable delegation, other retention modes and all retained/
background/delivery consumers still require their complete adapters and readiness
proof. Product qualification stays closed until those obligations pass.
