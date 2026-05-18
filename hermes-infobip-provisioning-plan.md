# Hermes ↔ Infobip Provisioning — Design Plan

---

## 0. Implementation status snapshot (2026-05-12)

| Area | Plan section | Status |
|---|---|---|
| Outbound send (WhatsApp text + template + DLR) | §4 client, §5 messaging surface | **Verified end-to-end 2026-05-11.** `POST /api/Messages/whatsapp/{text,template}` → Infobip Messages API → DLR roundtrip. |
| Customer ↔ Entity provisioning | §4 `InfobipProvisioningService`, §5 admin APIs | **Endpoint surface in place** (`/api/Entities` CRUD, `/api/Applications` CRUD + bootstrap). Async worker + state machine NOT built yet. |
| Application bootstrap (Speed2Lead) | §7 step 2 | **Done.** `POST /api/Applications/bootstrap-speed2lead` is idempotent (GET-or-create). |
| Numbers data model (`phone_number`, `campaign_phone_number`) | §3 new tables | **NOT built.** Numbers slice is passthrough-only — no DB persistence in the current cut (see §9 update). |
| Numbers endpoints | §5 customer-facing APIs | **Refactored 2026-05-18.** Tenant-facing `GET /api/Numbers[?entityId=...]` are stubs returning 501 — they will read from the local mapping table once it lands (populated by the Embedded Signup registration webhook). The new admin-only `GET /api/admin/numbers/inventory` is the single remaining caller of Infobip Resource Management (`GET /resource/1/resources`); it exists for reconciliation / audit. `POST /api/Numbers/register` and `POST /api/Numbers/{numberKey}/associate` were deleted — WhatsApp onboarding flows through Meta Embedded Signup (`POST /whatsapp/1/embedded-signup/registrations/share-waba`), and Infobip's Automatic Resource Association binds the number to the tenant during ES. |
| Webhook subscriptions | §6 webhook handlers | **CRUD surface in place** (`/api/Subscriptions/{channel}`). Inbound webhook receiver + resource-lifecycle handlers NOT built yet. |
| WhatsApp templates | (added scope) | **Shipped.** `/api/WhatsappTemplates/{sender}` CRUD passthrough. Status is Meta-controlled (read-only from our side). |
| Deployer config (`InfobipOptions`) | §4 `InfobipClient`, §7 step 1 | **Today: single account-level API key + base URL in `appsettings.json`.** No encryption-at-rest, no validation endpoint, no admin UI — these remain follow-ups. |
| Provisioning job audit (`infobip_provisioning_job`) | §3 new tables | **NOT built.** |
| Message-dispatch tracking (`InfobipMessageDispatch`) | §3 new tables | **NOT built.** Body is intentionally out of scope — only response metadata (BulkId, MessageId, destination, latest status, error code) so DLR refresh / status recovery / retry have a durable handle. |
| WhatsApp/Meta verification webhooks | §6, §11 phase 3 | **NOT built.** Verification status must be polled or surfaced manually until then. |

### Wire-name gotchas captured during the outbound roundtrip

Apply read-back verification across every Infobip create/update call —
each of these was hit on the way to a green end-to-end test:

1. Entity create wants `entityName`, **not** `name`. A wrong key is
   silently accepted; the alias persists as `null` and reads back blank.
2. Application create wants `applicationName`, same gotcha.
3. List endpoints return `{ results: [...], paging: {...} }`, not a
   top-level array. Match the DTO shape exactly.
4. Auth header is `Authorization: App {ApiKey}` — not `Bearer`.
5. `/messages-api/1/reports` is a *drain queue*: each DLR is returned
   once then removed. Do not poll; fire one fetch per send (`messageId`
   or `bulkId`), 30–120 s after the send.

---

## 1. Context

Hermes360 is shipped as an on-premise product. Each **deployer** (a client organization that installs and runs Hermes) holds their own commercial relationship with Infobip, subscribes directly, and provisions their own Infobip account. At install or setup time the deployer supplies Hermes with their Infobip base URL and API key; from that point on Hermes acts independently against the deployer's Infobip account.

Inside a single Hermes installation, the deployer serves many **customers**. Each customer is one company/organization that uses the deployer's services and runs campaigns through Hermes. Customers do not know Infobip exists — they interact only with Hermes. This document describes what to add or modify in Hermes so that one Hermes installation can manage all the Infobip resources for its customers automatically.

## 2. Architectural decisions

**The deployer brings the Infobip subscription.** Hermes is not bundled with an Infobip account. The deployer registers with Infobip themselves, then configures Hermes with the base URL and API key for their account. Hermes never assumes a shared or central Infobip backend.

**One Infobip application per Hermes product, per deployment.** Speed2Lead is the first application. Created automatically on Hermes application startup (idempotent GET-or-create), inside the configured Infobip account. Shared across all customers of that deployment. Future Hermes products get their own application within the same Infobip account — entities are reusable across applications.

**Customer to Infobip entity is one-to-one and eager.** Entity reconciliation runs on Hermes application startup: every existing customer without an `infobip_entity_id` is queued for entity creation, and from that point on a new entity is created in the configured Infobip account whenever a customer is created in Hermes — regardless of whether they will use Speed2Lead. This decouples identity (entity = customer) from product (application), avoids toggle-driven provisioning code paths, and stays future-proof. Customers that never send anything will show zero billing — easily filtered at reporting time.

**Activation/deactivation of Speed2Lead is a Hermes-only flag.** It does not touch Infobip. Deactivated customers keep their entity (dormant, zero traffic, zero billing).

**Hermes is the only consumer of Infobip APIs within a deployment.** Customers have no Infobip credentials, portal access, or knowledge of Infobip identifiers. Number registration, sender configuration, and reporting all flow through Hermes APIs.

**Provisioning is asynchronous, state-machine-driven.** Customer creation in Hermes must never block on Infobip availability. A background worker performs the actual Infobip calls and updates state.

## 3. Data model changes

### New tables

**`infobip_deployment_config`** — singleton (or one row per environment) holding the deployer's Infobip configuration: base URL, encrypted API key, last successful connectivity check, configured-by user, configured-at timestamp. This is the runtime configuration boundary between Hermes and the deployer's Infobip account.

**`infobip_application`** — typically one row per product. Holds the Infobip-side `applicationId`, the product it belongs to (e.g., `speedtolead`), creation timestamp, and current status. Populated automatically on Hermes application startup by the bootstrap routine (GET-or-create against the configured Infobip account).

**`phone_number`** *(deferred — not built in the current cut)* — first-class, customer-scoped resource. Phone numbers belong to customers, not to campaigns. Fields: `id`, `customer_id`, `number` (E.164), `channel` (sms/whatsapp/voice), `country`, `infobip_number_key`, `infobip_resource_request_id`, `registration_state`, `registered_at`, `last_synced_at`, `metadata` (JSON — Meta business verification details, sender type, etc.). The current `NumbersController` is a passthrough — when this table lands it will read the Resource Request id off the Infobip response and persist state per request.

Registration states: `requested` → `procuring` → `pending_registration` → `pending_approval` (WhatsApp/Meta) → `active` → `suspended` / `decommissioned`.

**`campaign_phone_number`** *(deferred — not built in the current cut)* — junction table linking campaigns to phone numbers. Many-to-many within a customer. Fields: `campaign_id`, `phone_number_id`, `role` (e.g., primary/secondary/inbound), `assigned_at`. Allows the same number to be used by multiple campaigns of the same customer.

**`infobip_provisioning_job`** *(deferred — not built in the current cut)* — audit and retry log. Captures every entity, number, and configuration call Hermes makes against Infobip. Fields: `resource_type`, `resource_id`, `action`, `state`, `attempts`, `last_error`, `last_attempt_at`. Powers the admin retry UI and post-mortem investigation.

**`InfobipMessageDispatch`** *(planned — not built in the current cut)* —
**one row per Infobip `messageId` (per recipient)**. Persists only the
response metadata returned by `POST /messages-api/1/messages`, never the
message body. A bulk that fans out to N destinations produces N rows
sharing the same `BulkId`.

Purpose: give Hermes a durable handle to every outbound attempt so that
DLRs (push or pull), status refresh, error recovery, and retries have
somewhere to land. Without this table the only record of an in-flight
send is the original API response in process memory; if the process dies
or the DLR webhook is missed, the send is unrecoverable.

Fields: `MessageId` (unique), `BulkId` (indexed), `Destination`,
`InfobipApplicationId`, `InfobipEntityId` (FK → `InfobipCustomerMapping`),
`CampaignReferenceId`, `Channel`, `Sender`, `LatestStatusGroupId`,
`LatestStatusGroupName`, `LatestStatusId`, `LatestStatusName`,
`LatestStatusDescription`, `LatestErrorGroupName`, `LatestErrorName`,
`LatestErrorIsPermanent`, `CorrelationData`, `CallbackData`,
`RetryCount`, `RetryOfMessageId` (nullable self-FK), `SentAt`,
`LastStatusRefreshAt`, `DeliveredAt`, `FinalizedAt`, `CreatedAt`,
`UpdatedAt`.

Status lifecycle: `PENDING` (write on send response) → DLR / refresh
updates → terminal (`DELIVERED` / `UNDELIVERABLE` / `EXPIRED` /
`REJECTED`). On a non-permanent error a Hermes retry produces a new row
with a new `MessageId` and `RetryOfMessageId` pointing back to the
original.

Explicitly NOT stored: message text / template body / template
variables / media URLs. Two reasons: GDPR data-minimisation (bodies are
typically customer PII) and storage growth at send volume. A separate
Hermes-side store can hold bodies under stricter retention if a replay
feature ever needs them; it would join on `MessageId`.

### Modifications to existing tables

**`customer`** — add `infobip_entity_id` (nullable), `infobip_provisioning_state` (pending/provisioning/provisioned/failed), `infobip_last_sync_at`, `infobip_last_error`.

**`campaign`** — no schema change required; association with numbers is via the junction table. Optionally add `default_inbound_number_id` for explicit inbound routing per campaign when numbers are shared (see open question 1).

## 4. Services and business logic

**`InfobipClient`** — new internal library. Wraps every Infobip HTTP call. Reads the base URL and API key from `infobip_deployment_config` (cached in memory with a reload on rotation). Owns retry policy, idempotency keys, timeout handling, and the per-endpoint wire-name conventions we've already documented (`entityName`, `applicationName`, `results/paging`, etc.). All Infobip calls in Hermes must go through this client. Must redact the API key in all logs and error reports.

**`InfobipProvisioningService`** — service layer with the high-level operations: `createEntity(customer)`, `readEntity(customer)` (with wire-name verification), `createApplication()` (invoked by the startup bootstrap routine), `registerNumber(phoneNumber)`, `syncNumberStatus(phoneNumber)`, `bindSenderToEntity(...)`. Each operation is idempotent and writes a row to `infobip_provisioning_job`.

**`InfobipConfigService`** — manages the deployer's Infobip credentials: store, encrypt at rest, validate (test connectivity against Infobip), rotate, surface health to the admin UI.

**Customer lifecycle hooks** — on customer creation, enqueue an entity provisioning job. On customer name change, enqueue an entity update job (assuming Infobip supports entity updates; to verify).

**Phone number lifecycle hooks** — on number request, kick off the procurement workflow (ops-driven in phase 1). On registration callback from Infobip webhooks, transition state and notify the customer.

## 5. API surface

### Customer-facing APIs (no Infobip identifiers exposed)

**Implemented today (post-refactor 2026-05-18):**

- `GET /api/Numbers` — **stub returning 501**. Will read from the local
  phone-number mapping table once that table lands. The table is
  populated by the Embedded Signup registration webhook
  (`businessAccountId`, per-sender `phoneNumberId` /
  `displayPhoneNumber` / `status`).
- `GET /api/Numbers?entityId=<id>` — same stub, filtered by `entityId`
  when the local table is in place.
- `GET /api/admin/numbers/inventory` — **admin-only** passthrough over
  Infobip Resource Management (`GET /resource/1/resources`). Used for
  reconciliation / audit only. Authorization is left as a TODO until
  Hermes-wide auth middleware lands; ingress-layer gating on
  `/api/admin/*` is the interim control.

**Removed (no longer in Hermes' design):**

- `POST /api/Numbers/register` — Resource Request is not used by
  Hermes. WhatsApp onboarding flows through Meta Embedded Signup
  (TPP) → `POST /whatsapp/1/embedded-signup/registrations/share-waba`,
  hosted on a separate controller.
- `POST /api/Numbers/{numberKey}/associate` — Resource Association is
  not driven explicitly by Hermes. Infobip's Automatic Resource
  Association binds the resource to the tenant's `entityId` during ES.

**Planned (not yet built):**

- `GET /api/campaigns/{id}/numbers` — numbers attached to a campaign.
- `POST /api/campaigns/{id}/numbers` — attach a number to a campaign.
- `DELETE /api/campaigns/{id}/numbers/{number_id}` — detach.

> **Naming delta from the original plan.** The original brief described
> `POST /api/numbers/request` as a no-Infobip-call placeholder writing
> `registration_state = requested` to the (then-planned) `phone_number`
> table. Scope shifted: there is no DB persistence in this slice and
> the endpoint actively hits Infobip. The implemented name is
> `POST /api/Numbers/register` to reflect "we are enrolling an existing
> tenant number, not procuring a new one."

### Admin/operator APIs (for the deployer's ops team)
`GET /admin/infobip/config` — show base URL, masked API key, last health check.
`PUT /admin/infobip/config` — set or rotate Infobip credentials, validates connectivity before saving.
`POST /admin/infobip/health-check` — force a connectivity probe.
`GET /admin/infobip/application` — show the speedtolead application status.
`POST /admin/infobip/application/bootstrap` — create the application if it doesn't already exist (idempotent).
`GET /admin/customers/{id}/infobip` — provisioning state, entity ID, last error.
`POST /admin/customers/{id}/infobip/retry` — manual retry of failed provisioning.
`GET /admin/numbers?state=...` — number inventory with state filtering.
`POST /admin/numbers/{id}/sync` — force a status sync from Infobip.

## 6. Background workers

**EntityProvisioningWorker** — picks up customers with `provisioning_state = pending`, calls `InfobipProvisioningService.createEntity()`, verifies the entityName round-tripped on read-back, updates the customer. Retries with exponential backoff; marks `failed` after N attempts and surfaces to admin UI.

**NumberSyncWorker** — periodic reconciliation of phone number state against Infobip. Catches missed webhooks and corrects drift.

**Webhook handlers** — extend the existing DLR endpoint to handle number registration status updates (especially WhatsApp/Meta approval events) and any entity-level events Infobip emits. Webhook URLs must be configurable per-deployment (the deployer registers their own callback URL with Infobip).

## 7. Deployer setup and startup bootstrap

Provisioning is owned by Hermes itself at application startup — it is **not** an explicit operator workflow. The deployer's only responsibility is supplying valid Infobip credentials in configuration; everything else is the responsibility of the startup bootstrap routine.

1. **Pre-requisite (out of Hermes, one-off)** — the deployer creates an Infobip account, completes Infobip onboarding, generates an API key with the necessary scopes, and places the base URL and API key into Hermes configuration (today: `appsettings.json`; future: encrypted admin-managed store).
2. **On every Hermes application startup** — the bootstrap routine:
   - Validates the configured Infobip credentials by calling a lightweight Infobip endpoint (e.g., list applications). If the credentials are missing or invalid, Hermes still starts, but messaging features degrade gracefully (admin banner, sends blocked with a clear error) until configuration is fixed and the app restarts.
   - Ensures the Speed2Lead Infobip application exists (GET-or-create, idempotent). The resulting `applicationId` is written to `infobip_application`.
   - Enqueues an entity-provisioning job for every customer where `infobip_entity_id IS NULL`. The EntityProvisioningWorker then processes those customers through the same code path used for new customers — without any operator action.
3. **Ongoing (out of Hermes)** — the deployer configures the Infobip portal to send webhooks (DLR, number status, etc.) to Hermes' public endpoint. This remains a one-off step in the Infobip portal.

The "Initialize Speed2Lead application" and "Trigger customer entity backfill" admin buttons described in earlier drafts are removed; both operations run automatically on every boot and are idempotent, so a restart is the standard remediation when something is out of sync.

## 8. Migration and backfill (per deployment, on upgrade)

When a deployer upgrades to the Infobip-enabled version of Hermes, the first application startup after the upgrade does the work — no operator command, no one-off migration script:

- The startup bootstrap routine (see §7) finds every customer with `infobip_entity_id IS NULL` and marks them `provisioning_state = pending`. The EntityProvisioningWorker then processes them through the same code path as new customers.
- Read-back verification on every entity ensures we don't end up with anonymous entities from prior wire-name bugs.
- Failures land in the admin UI for the deployer's ops team to handle. Restarting the application is a safe way to retry, because every step is idempotent.

Existing phone numbers (already in use somewhere in Hermes today) need a similar backfill: each must be matched to a sender that exists in the deployer's Infobip account, and its `infobip_sender_id` recorded. The exact procedure depends on what's in the deployer's Infobip account at the time of the upgrade — to be assessed per deployment.

## 9. Where phone numbers live, and the multi-campaign question

**Hermes does not procure numbers from Infobip's inventory.** Tenants
always bring their own MSISDN — typically issued by their telco.
WhatsApp tenant onboarding flows through Meta Embedded Signup
(Infobip's Tech Provider Program): the tenant completes ES in a
Hermes-hosted widget, Hermes hands the captured `wabaId` /
`phoneNumberId` to Infobip via
`POST /whatsapp/1/embedded-signup/registrations/share-waba`, and
Infobip's Automatic Resource Association binds the resource to the
tenant's `entityId` during ES. The procurement path described in
earlier drafts is dropped from scope.

**Two Infobip surfaces** are involved (post-refactor 2026-05-18):

| Step | Infobip surface | Endpoint | Hermes endpoint |
|---|---|---|---|
| Tenant onboarding (WhatsApp) | Embedded Signup · share WABA | `POST /whatsapp/1/embedded-signup/registrations/share-waba` | (separate Embedded Signup controller, not `NumbersController`) — invoked after the tenant completes Meta ES under Infobip's Tech Provider Program. |
| Admin reconciliation / audit | Resource Management | `GET /resource/1/resources` | `GET /api/admin/numbers/inventory` (admin-only — not on the tenant onboarding path). |

The two paths previously listed here (Resource Request and Resource
Association) are out of scope. Resource Request: WhatsApp onboarding
flows through ES, not Resource Request. Resource Association:
Infobip's Automatic Resource Association handles the
tenant↔resource binding during ES, so Hermes never POSTs an
association.

**Phone numbers are customer-scoped, not campaign-scoped.** A number is owned by a customer from the moment it is registered with Infobip. The customer↔number binding lives on Infobip's side via Automatic Resource Association during ES (and in Hermes' local mapping table once it lands).

**A single number can be assigned to multiple campaigns of the same customer** via the (planned) `campaign_phone_number` junction table. This is supported intentionally — customers will commonly reuse a number across a "Welcome" campaign, a "Follow-up" campaign, and a "Promotion" campaign. Until the table lands, campaign-level routing is handled outside this slice.

**A number cannot be assigned to campaigns of different customers.** Cross-customer sharing is explicitly prevented by Hermes' authorization layer and validated at every assignment. Infobip's entity tagging supports billing attribution, but the isolation is enforced by Hermes.

**Status of the three /resource/1/... paths (post-refactor 2026-05-18):**

- **Resource Management** (`GET /resource/1/resources`) — **in scope.**
  Used by `AdminNumbersController.Inventory` only. Path is still
  inferred from Infobip documentation; the direct probe in
  `HermesMessaging.ApiService.http` is the source of truth for the
  live account.
- **Resource Request** (`POST /resource/1/resource-requests`) — **out
  of scope.** WhatsApp onboarding flows through Meta Embedded Signup;
  Hermes never submits a Resource Request. The constant and DTO have
  been removed from the codebase.
- **Resource Association**
  (`POST /resource/1/applications/{app}/entities/{ent}/resources`) —
  **out of scope.** Infobip's Automatic Resource Association binds
  the resource to the tenant `entityId` during ES, so Hermes does not
  drive this call. The path builder and DTO have been removed from
  the codebase.

This raises one routing question that needs a product decision: when a number is shared across campaigns and an inbound message arrives, which campaign owns the inbound? See open question 1.

## 10. Open questions for manager review

**1. Inbound routing for shared numbers.** When the same number is attached to multiple campaigns of a customer, where does an incoming reply land? Options: (a) one campaign is marked "default inbound" per number, (b) match against an active conversation/session, (c) the first campaign attached to that number wins. Need a decision before implementing inbound.

**2. Customer deactivation lifecycle.** When a customer is deactivated or deleted in Hermes, what happens to their Infobip entity and registered numbers? Proposed: soft-delete in Hermes, keep entity dormant for billing history, surface in a separate "archived customers" view.

**3. WhatsApp business verification ownership.** WhatsApp number registration requires business documentation submitted to Meta. In an on-premise model: is the documentation provided by the customer (through Hermes) or by the deployer on the customer's behalf? Affects the customer-facing API surface significantly.

**4. Number procurement automation phase.** Phase 1 can ship with manual procurement — a customer's number request lands in an ops queue inside Hermes, the deployer's ops team procures the number in their Infobip account, then marks it available in Hermes. Full API-driven procurement is later. Is the manual flow acceptable for launch?

**5. Multi-application data model now or later.** If we know a second Hermes product is coming, model the entity-to-application relationship now (many-to-many between entity and application config). If Speed2Lead is the only product for 12+ months, defer.

**6. Activation flag semantics.** When a customer deactivates Speed2Lead, do we (a) block all outbound sends, (b) keep numbers assigned but inactive, (c) detach numbers from campaigns automatically? Each has different UX on reactivation.

**7. Admin UI scope.** A dedicated "Infobip Operations" admin section, or integrate provisioning state and retry controls into the existing customer detail page? Affects design effort.

**8. Credential storage and rotation.** Where is the Infobip API key stored, what encryption-at-rest mechanism, what rotation procedure? Hermes likely already has a secret store pattern — does it suffice, or do we need to extend it?

**9. Per-environment configuration.** Does a single Hermes deployment need to support multiple Infobip environments (e.g., dev/staging/prod for the same deployer), or is one Infobip config per deployment sufficient?

**10. Test/sandbox Infobip account.** Does Vocalcom provide a reference Infobip account that deployers can use for their initial smoke tests, or is the deployer expected to bring a production Infobip account from day one? Affects the install-time UX.

## 11. Suggested phasing

**Phase 0 — Startup bootstrap.** Application-startup routine that validates Infobip credentials, ensures the Speed2Lead application exists (GET-or-create), and enqueues entity provisioning for every existing customer. Adds credential storage + encryption and a connectivity health check. No operator "Initialize" button — the work is automated and idempotent. Without this, no later phase functions.

**Phase 1 — Customer provisioning foundations.** Customer-to-entity eager provisioning, async worker with state machine, backfill of existing customers, admin visibility and retry controls.

**Phase 2 — Numbers (manual procurement).** Phone number data model, manual ops-driven procurement flow inside the deployer's Infobip account, campaign assignment via junction table, customer-facing read APIs.

**Phase 3 — Number lifecycle automation.** API-driven number registration 