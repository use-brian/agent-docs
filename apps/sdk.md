---
title: Application SDK profiles
description: Locally tested application, deployed module and independent source kit contract.
tags: [applications, sdk, connectors]
canonical: https://usebrian.ai/docs/apps
---

# Application SDK and scaffolds

OSS bundle validation, browser application client and deployed module/source kits.
R5a conformance is local package evidence. Runtime qualification remains false;
this package does not activate protected operations or certify consumers. The
profiles and private canonical type packages are not claimed registry-installable
before R6/publication.

| Export | Environment | Purpose |
|---|---|---|
| `@use-brian/brian-app` | Node or browser | Existing manifest/bundle validation |
| `@use-brian/brian-app/application` | Browser | Typed application client, safe errors, app-bound PKCE and Home adapters |
| `@use-brian/brian-app/module` | Node only | Canonical module/registration types; service/delegated credential transport |
| `@use-brian/brian-app/connector` | Node only | Ed25519 actor verifier and independent source protocol helpers |

Canonical types come from shared/core through erased type imports. Deployments
supply matching canonical packages; the browser runtime graph contains neither
root. Deployed registration uses the host's `defineApplicationModules` and the
canonical service. No uploaded code registration or central ACL edit is required.

From the OSS root, with installed local tooling:

```sh
pnpm --filter @use-brian/brian-app build
node packages/brian-app/bin/brian-app.js scaffold --profile native output/sdk-native
node packages/brian-app/bin/brian-app.js scaffold --profile connector output/sdk-source
node packages/brian-app/bin/brian-app.js output/sdk-native
node packages/brian-app/bin/brian-app.js lint output/sdk-native
pnpm --filter @use-brian/brian-app test
```

Scaffolds require an empty destination and reject traversal, symlinks and unknown
profiles. With the matching local packed packages/toolchain provided, run
`npm run build` and `npm run contract` inside either generated directory. Tests
perform this outside the workspace without registry or global installations.
The native HTML is a placeholder for bundling `src/browser.ts`; configure the
server module and build the browser assets with your normal browser bundler.

```ts
import { createApplicationClient, createFetchTransport } from '@use-brian/brian-app/application'
const client = createApplicationClient({ appId: registeredAppId,
  transport: createFetchTransport({ apiOrigin: 'https://brain.example',
    credential: async () => inMemoryAppOAuthToken }) })
const record = await client.get('request', 'synthetic-id', { signal })
// record.ref.version is opaque; preserve fields, source evidence and receipts.
```

Helpers map to `/api/applications/:appId`: `catalog`, `schemas/:type`,
`records/:type/:id`, `records/:type/query`, `records/:type/:id/history`,
`queries/:queryKey` (registered input), and `commands/:commandKey` (shared
idempotency/version/input envelope). Query defaults/types are the shared wire
contract. Query results retain the page envelope (`records`, optional `nextCursor`
and aggregate entries with lossless string or null values), including all evidence.
Schema governance uses explicit native-management `schemas` ports;
absent ports fail `operation_not_available`. Application credentials never become
native schema authority. No helper computes permission from a grant.

In a trusted Node deployment, supply an authenticated native-management fetch:

```ts
import { createNativeSchemaPorts } from '@use-brian/brian-app/module'
const schemas = createNativeSchemaPorts({ apiOrigin: 'https://brain.example',
  workspaceId, fetch: authenticatedNativeManagementFetch })
const review = await schemas.preview({ typeId, expectedVersion: 1, properties, assistantId }, { signal })
if (review.canApply) await schemas.apply({ previewId: review.id, assistantId }, { signal })
```

These ports also implement `createApplicationClient({ appId, transport, schemas })`.
Preview calls `POST /api/applications/workspaces/:workspaceId/schemas/:typeId/preview`
with `{ expectedVersion, properties, assistantId? }`; apply calls
`POST .../schema-reviews/:previewId/apply` with `{ assistantId? }`. Properties use
the canonical core `PropertyDef` type; schema versions are numeric, while record
versions remain opaque strings. Preserve review metadata and validity counts.
The server authenticates the native actor and selects a visible assistant; the SDK
does not send user IDs or roles. Keep the authenticated fetch in the trusted
deployment, outside static application bundles. Review creation and apply both
persist state; interruption is uncertain and does not trigger a retry. Reconcile
the same saved review under fresh current authority before deciding what to do.

`ApplicationSdkError` exposes `code`, `effect`, `retrySafe`, `status` and
`cancelled`. Returned objects are preserved intact; there is no persistent cache,
optimistic permission widening or automatic retry. After dispatch, a disconnected
or cancelled write is uncertain. Reconcile using its idempotency key/receipt
before deciding to retry. HTTP error bodies and transport exception details are
never included in SDK errors.
Credential suppliers receive optional call options containing the operation's
AbortSignal. Cancellation while waiting for browser, service or delegated
credentials rejects promptly before dispatch with effect `none`. Late credential
results are ignored; no authentication or operation is retried automatically.
Existing suppliers may continue to ignore the optional call options.

Standalone login uses `createApplicationSignIn`: keep the originating page alive,
open a popup, call `begin({ popup })`, and navigate that popup to `authorizationUrl`.
The registered callback page posts `{ type: 'ub:application:callback', appId, url:
location.href }` to its opener using the exact app origin. Pass that MessageEvent
to `completeFromMessage`; it checks the bound window/origin/app, then consumes the
exact redirect/state/PKCE proof through the existing app-bound token route.
`complete(callbackUrl)` is for a trusted callback transport; `cancel()` discards
its one-use proof. Full-page navigation discards this memory-only proof, so use a
popup or a trusted server-side session adapter. Native sign-in/consent handles
same-origin sessions. Login alone supplies no records.
Starting another login or calling `cancel()` invalidates an earlier pending PKCE
generation. Late digests cannot replace the newest proof, and a cancelled or
superseded exchange cannot return a token after response-body decoding.

Home uses `createHomeApplicationTransport({ appId, homeAppId, workspaceId,
apiOrigin, hostOrigin, window })`; its `transport` requests the existing signed
Home bridge token via `ub:ready`/`ub:token` and checks parent-window identity,
exact host/API origin and Home/workspace binding. Call `dispose()` on unmount;
pending requests fail explicitly. Backend qualification still holds binding
issuance unavailable. Never substitute a native JWT or service key.
Opaque Home Origin `null` is admitted by the backend only for an active Home
client using the signed Home profile. The SDK never overrides Origin. Budget
denial preserves status 429 and safe effect/retry metadata without retrying.

Service/delegated credentials belong only in Node `./module`. A delegated
credential requires app-bound user OAuth in `X-UseBrian-User-Authorization`; it
accepts no free-form user identity. Never log tokens, embed secrets/cookies/native
sessions in static assets or write them to localStorage.

A connector deployment must supply authenticated transport, exact trust binding,
public keys, durable atomic replay storage, source account/assistant mapping,
canonical per-record/field/recipient ACL callbacks and saved idempotency/effect
receipts. Unconfigured callbacks fail unavailable; unknown source mappings deny.
Capabilities must truthfully advertise retention modes and reconciliation.
Governed observations require current authorization, released snapshots require
independent release receipts and expiry, and ephemeral output requires a lane
that does not persist it. A valid Brian assertion proves identity, never business
permission. Reconcile uncertain effects; never retry automatically.

The dedicated spec is `docs/architecture/features/application-platform-sdk.md` in
the platform repository; OSS guide: `docs/applications/sdk.md`. R2 runtime QA,
R3 retained evidence/cache integration, R5b management, R5c real reference journeys,
R6 qualification and all publication, production setup, trust provisioning, live
provider QA and independent-source adoption remain owed.

Cancellation while consuming an effect response remains uncertain. Already-aborted
schema operations never invoke their injected deployment port.


## Runnable reference applications

The OSS `examples/application-platform/` reference group supplies separate static
operations and customer portal frontends, a server-only service/delegated client,
and an independent loopback source. Build browser assets with the existing SDK
build followed by `node examples/application-platform/build.mjs <output-directory>`;
serve that output with an ordinary static server. Both frontends use Brian's
registered application OAuth/PKCE routes and the same configured API backend.
A configured API origin, application ID, OAuth client ID, redirect URI and workspace
are public deployment metadata. Tokens remain in memory. No frontend accepts a
service secret, native bearer, claimed user ID or local permission override.
A portal requires a verified Brian nonmember identity and explicit current grant;
a static frontend alone does not establish that deployment prerequisite.

The browser session binds app/workspace/API deployment and one in-memory login
generation. It aborts outstanding calls and clears rendered protected values on
logout, expiration, offline, hidden-page and authority invalidation. Superseded
responses cannot repaint a new session. Reconnect starts with empty state and a
fresh explicit read. It stores no response, token, queued write or protected value
in browser persistence. Interrupted writes preserve their idempotency key and
uncertain effect in memory and require explicit reconciliation before another
write; reconnect never resends a mutation.

The deployed purchase module uses canonical generic schemas for requests, line
items and orders, including canonical CRM company relations. Required deployment
ports authorize the existing requester/manager roles; unconfigured role authority
refuses. Selected canonical reads and mutations retain existing field/source
admission. Exact decimal cost strings and integer quantities are calculated with
BigInt, without a floating point conversion. Approval requires the submitted
state and an admitted budget, then changes the request and creates an order in
one ApplicationPlatformService transaction. The existing service owns CAS,
replay, revisions, command audit and event publication; no example-local business
store or transaction commits independently.

The independent source owns persisted identity mappings, field/record/action ACL,
request replay consumption and effect receipts. SDK assertion verification binds
issuer, audience, workspace, connector instance, actor and executing assistant;
identity alone grants no source access. Source UI/API and sidecar paths use the
same current source authorization function. HTTP reads return the canonical `{data,envelope}` shape. Model use is disabled by default; it requires explicit `model: true` on current grants covering every selected field (and current context rights), or on the exact released snapshot. Combining field grants never lends model permission to an uncovered field. Released snapshots require a persisted
explicit audience and expiry. Ephemeral capability at this source does not qualify
a Brian retention lane. Source command uncertainty reconciles against its saved
idempotency receipt without automatic retries. Second-module/source conformance
uses the public SDK contract and does not import the purchase module.

These examples do not widen capability activation. Local module/client/source
proof, actual authenticated browser journeys, runtime admission and whole-platform
acceptance are recorded separately; missing runtime or browser proof remains
incomplete implementation acceptance.


### Local acceptance verifier

`node scripts/application-platform/verify-local.mjs --run <safe-id> --pg-bin <directory>`
uses the existing disposable PostgreSQL fixture and fixed API/SDK/reference test
commands. It never executes a command supplied by JSON. `--collect-only` validates
already recorded receipts without starting a suite. The input is
`output/application-platform/<safe-id>/input.json`; the deterministic output is
`manifest.json` in that directory. `--help --json` describes options and exit codes.
Unsafe IDs, ambient database targets and a nonlocal/symlinked output path fail before
suite execution. A test process has a recorded PID/group and actual watchdog;
SQL work uses the existing validation advisory lock and maintained fixture cleanup.
The caller supplies the fixed campaign deadline when applicable; no suite starts
without its full allowance. Required root gates are collected as source-bound
external receipts, never silently waived or restarted.

Input version 1 includes exact platform/OSS heads, source/dependency hashes,
original coverage-ledger checksum and all 503 consumer IDs, ten original family
closures, L01-L12/A01-A22/F01-F07 assertions, command receipts, browser evidence,
fresh/overlay/legacy migration evidence and owed operator work. A proof names an
actual receipt and exact passing test case. Every receipt binds a real command,
log checksum, source hashes and machine test results; failed, skipped, timed-out,
missing or stale cases fail. Delegated consumer closure names the actual caller,
canonical callee and enforcing case. Nonconsumer closure includes a concrete
reason and its validating case. Browser evidence names actual maintained-rig
journeys, artifacts, four locales and phone/desktop viewports; static HTML or test
HTTP endpoints do not replace that evidence. Rehearsals include their migration
set checksums and non-superuser runtime proof. A green local report is evidence,
never a runtime qualification credential or an activation control.

### Guarded campaign receipt ingestion

Campaign commands may run through the maintained budget/deadline guard. The
verifier imports that guard's immutable JSON receipt and log without rewriting
its recorded argv, working directory, start/end, exit status or timeout result.
Before launching a command, capture a source snapshot containing its relevant
public code/spec/dependency inputs and both repository heads. The snapshot must
predate that command, and all captured inputs must still match when collected.
A post-command snapshot cannot establish the tested source. Exact gate commands
and supported maintained wrappers are supplied by the reviewed verifier catalog,
never by a report's claimed success. Machine results and browser artifacts remain
independently checksum-bound; a successful process alone proves no test case.
Raw guard receipts, source snapshots and parsed output stay separate so failed,
missing, stale or unstarted commands remain inspectable evidence of incompletion.

### Explicit command reconciliation

`client.reconcile(commandKey, originalCommand, options?)` posts the exact original
command input, idempotency key and expected version to
`POST /api/applications/:appId/commands/:commandKey/reconcile`. This is a read-only
lookup under the current actor/app/module/input/result authority; it never executes
the command handler. `{ state: 'applied', receipt }` identifies the admitted saved
receipt; `{ state: 'uncertain' }` proves no safe retry. Missing receipts never imply
`not_applied`. The SDK performs no automatic retry. Keep the original envelope for a later
recovery check; an uncertain result does not authorize a new write.

App/service/Home MCP offers the same lookup as `applicationReconcile`. Native
members use `POST /api/applications/workspaces/:workspaceId/modules/reconcile`
with `{ moduleKey, commandKey, command, assistantId? }`, or `nativeModuleReconcile`
on the native module MCP profile. Both retain the original command envelope and
current authority, never execute the handler, and keep the existing read-only
surface refusal. The app SDK helper does not authenticate the native audience.
Lifecycle refusals preserve `module_draining` or `module_disabled` over HTTP409,
MCP and `ApplicationSdkError`, with effect `none` and `retrySafe: false`; refresh
state before deciding what to do. An interrupted effectful COMMIT remains
`outcome_uncertain`, regardless of any later lifecycle-looking error.

Source release decisions additionally carry `release: { id, expiresAt }` exactly
when `retentionMode` is `released_snapshot`. The release identity/expiry are bound
into the source decision digest. Governed decisions keep their existing shape.
The source must match the requested ID and exact expiry for the initiator and
every recipient; another valid release cannot substitute. Initial unbound source
reads may select an eligible release and emit its exact tuple for later checks.
Client-supplied release identity alone never grants retention or qualification.

### Self-hosted application sign-in portal

A self-hosted installation offering verified nonmember applications deploys the
existing OSS `apps/auth-web` image beside its API and app-web. Configure API
`AUTH_PORTAL_URL` and `AUTHED_APP_URL`; configure app-web
`PUBLIC_PRIMARY_AUTH_URL` to that same auth portal; configure auth-web
`INTERNAL_API_URL`, `AUTH_PORTAL_URL`, `AUTHED_APP_URL` and the existing production
`COOKIE_DOMAIN` shared by its configured HTTPS origins. Use the ordinary SMTP
settings on the API for genuine email verification. The existing auth-web
Dockerfile/entrypoint is the deployable UI; no hosted account service is required.
Missing portal/email configuration leaves nonmember sign-in unavailable.

The trusted application consent page uses its native session without automatic
OSS owner recovery. Missing/expired identity returns to configured auth-web with
only the exact same-origin application consent path and signed request. The
backend validates that request before selecting application-only onboarding.
The ordinary single-owner OSS entry elsewhere keeps its existing behavior.
The administrative rig composes this same auth-web app on loopback3005 and its
local SMTP capture solely for synthetic acceptance; real deployment and SMTP
configuration remain operator work.

### Optional source command binding

The Node connector/source protocol may advertise `capabilities.effectBinding: 'command_v1'`. A mutate authorization then carries `effect: { tool, input, idempotencyKey }`; the canonical decision digest binds the exact outbound command. Exact finite JSON input rejects sparse arrays, including nested missing own indices, while dense values keep their existing digest bytes. The source independently checks the real record, fields, expected version, idempotency key, and current mutation rights before dispatch. Reconciliation with an `effectDigest` includes the original complete authorization envelope and checks the actor-bound source receipt without dispatching again. Missing, mismatched, or unknown evidence remains uncertain. Existing read and governed authorization digests retain their prior shape. These locally proved protocol additions do not qualify a deployed source mutation profile or claim it is available to users.

### Public SDK guide

The existing `/docs/apps` page includes dedicated localized SDK sections beside
the Home bundle guide. The guide distinguishes the browser `./application`,
deployed `./module`, and independent `./connector` profiles, shows an app-bound
OAuth read with the canonical query envelope, and explains explicit reconciliation
using the unchanged original command envelope. Its self-host section names the
existing auth-web/API/app-web configuration and SMTP prerequisites. Sign-in alone
grants no records; no example stores credentials or claims whole-runtime qualification.

The verifier exports its reviewed command catalog for preparation of exact final-gate
recipes without launching them. It includes actual management locale/viewport
observations and the four-locale public SDK render guide. The app-web build uses
the same maintained administrative environment and loopback-only runtime adapter
as local acceptance, so ambient provider configuration cannot enter that build.
The SQL matrix excludes the orchestrator-only upgrade suite: fresh and concurrent
upgrade observations for OSS and overlay remain four independent required receipts.
The matrix also includes the affected Goal source-admission compatibility suite;
existing red compatibility cases remain required rather than being excluded.
Its maintained fixture uses `--runtime-role app_user`, matching actual production
non-owner deployment ACLs without adding grants. The default fixture role and
its original failed module receipt remain preserved.

### Independent source recovery after host retirement

Normal host profile review rejects every `effectBinding` key. Pending command
setups cannot apply; consumed setup replay retains the original qualified profile
ID under current checks. New command grants/keys, host `sourceEffect` and
`sourceReconcile` tools, and the private ephemeral source route are unavailable.
Governed/released reads and ordinary ephemeral refusal remain.

The connector SDK retains independent command-bound codecs and signed read-only
receipt reconciliation. Supply the original complete authorization, key and exact
effect digest when present. Current source/native authority remains required;
unavailable, malformed or mismatched evidence stays uncertain; authoritative
source-owned absence retains `not_applied`. Neither permits host dispatch or retry. Base
receipts without effect fields retain their original recovery shape. Passive
profile identity and existing credential generations remain unchanged.

### One-time native credential display

A successful human credential delivery keeps its one-time material only in the
current panel's memory. It defers the delivery's immediate overview refresh until
the human closes that display, preventing the newly created credential metadata
from clearing its own only presentation. Closing clears the material before
refreshing. Existing current-authority events, projection expiry/renewal,
workspace/app selection changes and unmount continue to purge it normally.
This sequencing does not cache the credential or relax any server or projection
authorization. Ordinary operation-receipt presentation remains separately tested.

## Retained application review recovery

Saved application review recovery uses native authenticated
`POST /api/applications/workspaces/:workspaceId/management/commands` and the
`applicationManage` tool at the matching `/management/mcp` endpoint: `legacy.view` takes the original
`reviewId` (and optional visible `assistantId`); `legacy.apply` and `legacy.cancel`
also require the exact returned decimal-string `expectedVersion` and `payloadHash`.
New `legacy.inventory`/`legacy.preview` authoring is retired. These metadata-only
recovery views require the current original creator and assistant; application
credentials cannot substitute. View has no mutation effect. An uncertain apply or
cancel must be reloaded by ID under current authority; never regenerate its hash,
substitute a native scope review, or automatically retry.

## Successful management receipt acknowledgement

A safe successful operation receipt stays in current panel memory until the human
closes it, changes application/workspace, or the existing authority/expiry purge
clears the projection. New panel writes are held while that receipt awaits
acknowledgement. Close clears the receipt, message and any one-time material
before refreshing the overview; the copy states that current state is read on
Close. The operation's own immediate metadata refresh must not erase its receipt.
Shared caches, periodic checks, projection comparison, denial handling and native
command authorization are unchanged.


### Fresh management browser fixture

Current management receipt acceptance uses a newly owned maintained administrative
fixture port and volume with the current migration set. Create the owner through
the maintained native local-session flow, then create the workspace, schema and
record through supported native HTTP routes; register/activate the app and prepare
its service and exact grant through saved management review/apply commands. Old
workspace or schema IDs are not fresh-fixture inputs. The credential browser input
contains explicit `fixturePgPort` and `nativeSessionPath` metadata, with the latter
equal to `.rig/admin-<port>/session.json`; the secret session file is read only for
supported native cleanup and is never copied into evidence. Receipt observations
remain tied to their actual prelaunch source/build inputs; fresh migrations alone
do not establish final frozen-source acceptance.


The reference source supports an explicit `dispatchCommands: true` deployment
option. Its signed MCP `dispatchRecord` takes `{ type, id, fields,
expectedVersion, expectedPolicyVersion, destination: "source_outbox",
idempotencyKey }`. It checks current source read/mutate rights and exact signed
command intent, then atomically stores selected values in its own durable
`dispatches` outbox plus a receipt in `effects`. It leaves parent record versions
unchanged because dispatch is a separate child operation. Brian receives only
state/receipt/digest and keeps its full original ancestry. Replay and
reconciliation do not enqueue again; current source revocation still applies.
The dispatch profile advertises `command_v1`, serves native setup reads at
`/http/read`, and refuses the legacy `/records/command` endpoint. Record-changing receipts additionally require the exact source-owned historical authorization described below. Unsupported sources retain the held uncertainty result.


Normal host workflows do not dispatch source mutations or retain effect receipts.
Independent reconciliation requires current authority and never redispatches.
Transport failure, unusable evidence or digest mismatch remains uncertain with
no automatic retry.


### Source-owned governed history after record changes

The independent reference source stores an immutable exact before-image in the
same transaction as a record-changing command and its idempotency receipt. Its
explicit governed-history policy permits historical read, context, derive and
retain decisions only for that saved resource/content/policy version, while the
current resource remains present, unwithdrawn and at the same policy version.
Both the saved fields and current fields must pass the same current source
account/assistant and field/action ACL. Historical mutation is forbidden. A
missing/purged before-image, policy change, withdrawal or current field/action
contraction refuses; no old observation is resolved to the latest content.

The independent source example retains this source-owned historical contract;
normal host workflows do not execute source commands. Signed reconciliation reads
the same actor/assistant-bound durable receipt and never dispatches again. Missing
current historical authorization remains uncertain with no retry. The independent
unchanged-parent dispatch operation remains separately supported.


The independent source example explicitly supports governed historical
read/context/derive/retain for exact saved before-images. Record-changing commands
store that history atomically with the source receipt. Current account/assistant,
field/action rights and unchanged current policy remain required; missing history,
withdrawal or contraction refuses. Historical mutation is forbidden. Brian never
aliases an old observation to latest content or infers a source retention right.
With `dispatchCommands: true`, signed MCP `updateRecord` takes
`{ type, id, expectedVersion, expectedPolicyVersion, values, idempotencyKey }` and
shares the source's canonical atomic mutation. Its command-bound reconciliation
checks the original actor/assistant/digest and current ACL without redispatch.
An unsupported source keeps its uncertain, no-retry workflow receipt.


Named person self-profile fields (`profile.name`, `profile.role`, `profile.company`,
`profile.location`, `profile.birthday`, `profile.pronouns`, legacy
`profile.role_or_company`, and `profile.sources`) are read-only canonical JSON
objects: `{value: originalJsonValue}` when present, or null when absent. The
read-only `profile.extra` object contains only additional attributes; canonical
person/common and custom fields retain their separate field permissions and
cannot be reintroduced through that object. Select each required field under
current grants. Person `tags` and `externalRef` are null when absent; explicit
`[]` and `{}` remain unchanged. These local read codecs preserve the named-profile and disjoint extra forms
accepted by the existing producer. Exact raw presence of an empty `custom_fields`
container or null-valued custom keys remains unqualified; existing custom fields
keep their own permissions. The codecs do not enable protected source-aware
self-profile mutation or certify other callers.

## Registered operation names

Only explicitly registered own operation keys are callable. An unknown query,
command or reconciliation key returns `operation_not_available` when the caller
passes the existing outer authentication, module and action-admission checks; the
application HTTP transport maps that code to 404. Otherwise the existing access
refusal takes precedence. Prototype-property names do not become operations through JavaScript
object inheritance. This refusal has no business effect; choose a key from the
current authorized catalog instead of retrying an unknown name.

### Command retries and normalized input

Reuse the same idempotency key with the same JSON input and expected version.
Object key ordering does not matter; array ordering and different supplied values
do matter, even if validation normalizes them to the same value. Correlation keys
can change without creating a new business command. A changed request returns
`record_version_conflict` after current admission checks.

Receipts created before wire-input fingerprinting cannot always prove which input
ran. Their command retries return `outcome_uncertain`; reconciliation returns
`uncertain`. They do not run the command again. Keep the original key and use the
application's existing recovery path; a new key is not proof that retrying is safe.

### Existing lifecycle identity

A deployed module can add its application operations to a lifecycle-only entry in
the supplied registry. Keep the same key and default state; omit the existing
blocker compatibility mapping to inherit it, or supply the exact same mapping.
Conflicting metadata and a second application registration are rejected. Existing
persisted state and the lifecycle blocker adapter remain authoritative.

### Scoped native module and recovery limits

A deployed custom native module may expose exact existing record IDs through the
reviewed `canonical_module_service_v1` profile. Keep its declared origin/type and
opaque ID unchanged, including decimal-text IDs. Request only released fields and
declared semantic commands. Registration or an ID grant does not replace current
app-owned record, action, field or source checks. Custom native self/relationship
selectors, frontend admission and role governance are not enabled by this profile.
Global qualification remains closed.

A lifecycle-only transition does not replace an existing activation or credential
generation. Use a fresh current invocation for a trusted recovery command after
draining or disable; new work remains refused. New custom-module activation, grants, credentials,
OAuth bindings, source-service bindings and service-role assignments still require
enabled admission. Current revocation and deployment/configuration changes refuse
recovery too. Reconcile uncertain effects before considering another operation.

The optional reviewed `canonical_app_delegation_v1` profile requires a named service
credential plus a separately authenticated current app-user OAuth binding for the
same app and workspace. Authority intersects both complete current grant chains
and issuance maxima. Caller-supplied user or actor fields grant no authority; loss
of either credential, OAuth binding, parent, issuer or activation refuses access.
This profile does not activate source mutations or certify application rollout.

### Generic record relationship selectors

The reviewed canonical records service may select a declared generic relation
using its registered `relation.<SHA-256-of-canonical-property-name>` resolver and
an independently readable anchor. Include the relation property in the grant's
read fields. Current and issued predicates must each retain that field after all
field masks, including title-only, empty and aggregate queries. Moving a record
out of the relationship or revoking anchor access removes access; a relationship
key does not gain an alternative child ID grant. Custom native relationship
selectors and global qualification remain closed.

## Attended source actions

`ub.source.v1` capabilities may declare `attendedActions: { idempotencyMetadataKey }`
alongside `reconciliation: true`. Upgrade Brian and the source SDK to versions
that accept this optional capability before advertising it; older strict codecs
reject unknown capability fields. The metadata key is a namespaced string such as
`source/operation-key`. This promises durable source-owned replay and business
authorization for signed attended human commands. The host places its stable
operation key in MCP tool-call `_meta`, covered by the request-bound actor
assertion. This is independent of retired service `command_v1` authoring.

The source must reserve each key under the verified human, assistant and
connection before applying the business effect, reject conflicting reuse, and
reconcile that key under current permissions. Protected mutation results carry
the same reserved source evidence as reads. Missing evidence releases no business
payload. See [native management](management.md#attended-company-connectors) for
reviewed setup and the attended operation envelope.
