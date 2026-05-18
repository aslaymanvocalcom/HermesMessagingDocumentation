# Hermes360 ↔ Infobip — Mapping Reference

> **Purpose.** Canonical mapping between Hermes360 entities/use cases and
> Infobip CPaaS-X primitives, with concrete API endpoints. Use this as the
> source of truth when generating mappers, adapters, or routing logic in
> `Hermes360.Messaging.*` and the `Hermes360.Messaging.Infobip` adapter.
>
> **Status.** Draft v2 — last refreshed 2026-05-12. Concept-level mapping
> is locked. Outbound WhatsApp send + DLR roundtrip verified end-to-end
> against `xr5e2e.api.infobip.com` on 2026-05-11. Numbers passthrough
> shipped (enrollment-only). Hermes C# class names and a few endpoint
> specifics still TODO — see §10.
>
> **What's wired up in `HermesMessaging.ApiService` today:**
> - `ApplicationsController` → `/provisioning/1/applications` (Speed2Lead bootstrap is idempotent).
> - `EntitiesController` → `/provisioning/1/entities` (Hermes Customer ↔ Infobip Entity, 1:1).
> - `MessagesController` → `/messages-api/1/messages` (unified send + WhatsApp text/template convenience).
> - `SubscriptionsController` → `/subscriptions/1/subscription/{channel}` (per-channel webhook routing).
> - `WhatsappTemplatesController` → `/whatsapp/2/senders/{sender}/templates` (CRUD; status is Meta-controlled).
> - `NumbersController` → tenant-facing reads (`GET /api/Numbers`,
>   `GET /api/Numbers?entityId=...`) which serve from Hermes' local
>   DB once the mapping table lands (today: stubs returning 501).
>   WhatsApp onboarding is handled separately via Meta Embedded Signup
>   (`POST /whatsapp/1/embedded-signup/registrations/share-waba`). The
>   `AdminNumbersController` exposes `GET /api/admin/numbers/inventory`
>   as the single remaining call to Infobip Resource Management,
>   admin-only, for reconciliation / audit. Resource Request and
>   Resource Association are no longer part of the Hermes design.
>
> **New persistence slice (planned — not in code yet):** every send
> response will land one row per recipient in `InfobipMessageDispatch`
> (BulkId, MessageId, destination, latest status — never the body), so
> that DLRs, status refresh, error recovery, and retries have a durable
> handle. See §7.1 and Visual Summary §10.
>
> **Real-world wire-name gotchas captured during the outbound roundtrip
> (2026-05-11)** — assume they apply across the rest of the API surface:
> - Provisioning create bodies use `entityName` / `applicationName`, not
>   `name`. A wrong name field is silently accepted and the alias is
>   stored as `null` — read-back verification is mandatory.
> - List responses wrap items in `{ results, paging }` (not a top-level
>   array). DTOs must match.
> - The legacy `/numbers/1/numbers` endpoint ignores `entityId` filters —
>   it returns account inventory only. Tenant-scoped views must use the
>   Resource Association endpoint.
> - Auth is `Authorization: App {ApiKey}` (not `Bearer`). The API key is
>   account-scoped to the host (`xr5e2e.api.infobip.com` here).
> - Reports (`/messages-api/1/reports`) are a *drain queue* — each DLR is
>   returned once then deleted. Do not poll; fire one fetch per send.
>
> **Sources used to produce this document:**
> - `Hermes360 — Campaigns & SpeedToLead Context (Infobip Integration Scope)` (project knowledge)
> - `messages-api.json` — Infobip OpenAPI spec v3.168.0 (project knowledge)
> - `subscriptions-api.json` — Infobip OpenAPI spec v3.168.0 (project knowledge)
> - `HermesMessaging.ApiService.http` — live probes against `xr5e2e.api.infobip.com` (source of truth for paths actually used today).
> - Infobip CPaaS-X documentation:
>   - <https://www.infobip.com/docs/cpaas-x>
>   - <https://www.infobip.com/docs/cpaas-x/resources>
>   - <https://www.infobip.com/docs/cpaas-x/application-and-entity-management>
>   - <https://www.infobip.com/docs/cpaas-x/sending-strategy-management>
>
> **Terminology.** "Hermes user" and "Hermes client" are used as synonyms
> here — both refer to the billable tenant account in Hermes360 that
> owns campaigns and pays for traffic. The original context doc uses
> "client"; everyday team conversations sometimes say "user". They are
> the same thing. The provisioning plan calls this the **Customer**
> (Hermes table) ↔ **Entity** (Infobip primitive) pair. Where the
> distinction matters (e.g., individual logins inside a tenant), this
> document calls those out explicitly.
>
> **Deployment context.** Hermes360 ships on-premise. Each deployer
> brings their own Infobip subscription and supplies Hermes with a base
> URL + API key at install time. Hermes is the only consumer of Infobip
> within a deployment; customers (Hermes tenants) never see Infobip
> identifiers. See `hermes-infobip-provisioning-plan.md` for the full
> deployer/customer model.

---

## 1. Anchoring concepts — Infobip CPaaS-X primitives

Infobip's CPaaS-X model has four primitives that scope every outbound
message, every webhook, every report, and every credential:

| Primitive | Definition (verbatim from `subscriptions-api.json`, `ApplicationEntityPair` schema) |
|---|---|
| **`applicationId`** | "CPaaS X property identifying an **application, a use case or an environment** on your system." |
| **`entityId`** | "CPaaS X property identifying a **unique actor** on your system." |
| **`callsConfigurationId`** | Voice equivalent of `applicationId`. Out of scope this phase. |
| **Resource** | Sender assets (numbers, alpha senders, email domains, ChatApp IDs) associated to an Application, an Entity, or an `(App, Entity)` pair. |

Operational characteristics that drive the mapping decision:

- Application and Entity IDs are **stable** — once created, ID cannot
  be changed. Only the alias is editable.
- Deleting an Application/Entity **cascades**: linked sending
  strategies, resource associations, and subscriptions are deleted with it.
- Sending strategies, API keys, resource associations, and subscriptions
  all attach to Application, Entity, or `(App, Entity)` pair.
- Infobip's own platform-pattern guidance: *"Add new clients as
  individual entities with a single API call."*
- Canonical Infobip example: *"a client with multiple customers, each
  using different resources"* — customers are entities.

---

## 2. The mapping decision

### 2.1 Recommended mapping

| Hermes360 axis | Infobip primitive | Cardinality | Rationale |
|---|---|---|---|
| **Hermes user (tenant)** | `entityId` | 1 entity per tenant | Matches Infobip's "unique actor" semantics. Per-tenant consumption, per-tenant resource pools, per-tenant API-key isolation all follow naturally. |
| **Hermes use case** (SpeedToLead, OutboundCampaigns, Inbound, Email, Chat, …) | `applicationId` | ~5–10 apps total (one per use case) | Matches Infobip's "use case" semantics. Allows per-use-case sending strategies, per-use-case webhook subscriptions, per-use-case reporting across all tenants. |
| **(use case, tenant) at runtime** | `(applicationId, entityId)` pair | many-to-many | Every outbound message carries both. Subscriptions, sending strategies, and resource associations can be scoped to the pair, only the App, or only the Entity. |
| **Hermes Campaign** | `options.campaignReferenceId` (Messages API) | 1 reference per campaign | Purpose-built field. The OpenAPI spec describes it as: *"ID that allows you to track, analyze, and show an aggregated overview and the performance of individual campaigns per sending channel."* |
| **Hermes "send batch"** within a campaign | `bulkId` | 1 per batch | Auto-generated if not supplied; returned in the response. Use a Hermes-deterministic id when batch-level idempotency is needed. |
| **Hermes ContactAttempt** (single send) | `messageId` | 1 per attempt | Client-supplied → free idempotency. |
| **Hermes Contact / Lead** | `destinations[].to` | 1 per recipient | MSISDN / email / WA business id, depending on channel. |

### 2.2 Why not the alternatives

- **Tenant-as-Application, Campaign-as-Entity.** Rejected. Entities are
  heavyweight configuration identities (resources, strategies, keys,
  subscriptions attach to them; deletion cascades; IDs are immutable).
  Campaigns churn — created, paused, archived, cloned. Mapping
  campaigns to entities means thousands of entities accumulate in the
  Infobip account, leak or cascading-delete on archive, and `campaignReferenceId`
  — the field literally designed for campaign tracking — sits unused.
  The use case dimension also disappears entirely under this model.
- **Tenant-as-Application, Use-Case-as-Entity.** Mechanically works but
  inverts Infobip's stated semantics on both axes. Loses the natural
  "one app = one use case" reporting.
- **Single global Application, Tenant-as-Entity.** Loses the
  per-use-case dimension; all use case webhooks fan into one subscription
  and have to be demuxed in the receiver.

The recommended model is the only one that aligns with Infobip's stated
semantics on both axes simultaneously.

---

## 3. Master mapping table

Every entry below is concept-level. C# class names from the Hermes360
codebase still need to be filled in (see §10).

### 3.1 Identity & tenant model

| Hermes360 | Infobip concept | API endpoint | Notes |
|---|---|---|---|
| Customer onboarding (new Hermes tenant) | Create Entity | `POST /provisioning/1/entities` (body: `{ entityId, entityName }`) | `entityId` = stable Hermes Customer.CustomerId stringified. Done **once** per customer — eagerly at customer-creation time for new customers, and reconciled automatically on every Hermes application startup for any customer that still has `infobip_entity_id IS NULL` (zero-traffic customers cost nothing). Wire field is **`entityName`**, not `name`. |
| Use Case bootstrap (every app start) | Create Application | `POST /provisioning/1/applications` (body: `{ applicationId, applicationName }`) | One per Hermes use case. Run automatically by the Hermes startup bootstrap (not a one-off operator action). Today only `speed2lead` is wired up; `BootstrapSpeed2Lead` endpoint is idempotent (GET-or-create). Wire field is **`applicationName`**, not `name`. |
| WhatsApp tenant onboarding | Embedded Signup · share WABA | `POST /whatsapp/1/embedded-signup/registrations/share-waba` | Tenant launches Hermes-hosted Meta Embedded Signup under Infobip's Tech Provider Program; Hermes captures `wabaId`/`phoneNumberId` from the ES callback and hands them to Infobip via share-waba. Replaces the previous Resource Request flow. **Not driven by `NumbersController`** — lives on a separate ES controller. Infobip's Automatic Resource Association binds the resource to `entityId` during ES, so Hermes never POSTs a Resource Association either. |
| Admin reconciliation / audit | Resource Management | `GET /resource/1/resources` | Single remaining call to Resource Management. Wired to `GET /api/admin/numbers/inventory` (admin-only). Not on the tenant onboarding path. The legacy `/numbers/1/numbers` endpoint returns the same shape but ignores `entityId`. |
| Per-tenant sending behaviour | Sending Strategy | Sending Strategy Management API | Recommended: define at Application level; override per `(App, Entity)` only when a tenant needs different behaviour. |
| API key linked to a tenant | CPaaS X-linked API key | `POST` Create CPaaS X API key | Recommended: one key per `(App, Entity)` pair → leak blast radius is one tenant × one use case. Note: today Hermes uses a single account-level key supplied by the deployer; per-`(App, Entity)` keys are a future hardening step. |

**Numbers slice — what's NOT in scope right now.** The `phone_number`
and `campaign_phone_number` tables described in the provisioning plan
§3 have not been built. The tenant-facing read endpoints
(`GET /api/Numbers[?entityId=...]`) currently return 501 — they will
serve from Hermes' local DB once the mapping table lands. The ES
registration webhook receiver, which populates that table, is the
other half of the dependency. The admin reconciliation endpoint
(`GET /api/admin/numbers/inventory`) is the only Hermes route that
calls Infobip Resource Management today.

### 3.2 Outbound message identification (every channel)

| Hermes360 | Infobip Messages API field | Notes |
|---|---|---|
| Hermes use case | `messages[].options.platform.applicationId` | Required for CPaaS-X tracking. |
| Hermes user (tenant) | `messages[].options.platform.entityId` | Required for CPaaS-X tracking. |
| Hermes Campaign id | `messages[].options.campaignReferenceId` | Returned on inbound MOs and DLRs — use for cross-channel campaign attribution. |
| Hermes ContactAttempt id | `messages[].destinations[].messageId` | Client-supplied → idempotency. |
| Hermes Contact (recipient) | `messages[].destinations[].to` | |
| Hermes sender | `messages[].sender` | Resolved per `(App, Entity)` via Sending Strategy. |
| Hermes correlation token (propagates to MO replies) | `messages[].options.correlationData` | Auto-generated if not set; **returned in inbound message** — use to match a reply to the original ContactAttempt. |
| Hermes DLR callback payload | `messages[].webhooks.callbackData` | Returned in delivery report — use to match DLRs back to ContactAttempt. |

---

## 4. Outbound — recommended unified path (Messages API)

Validated against `messages-api.json` v3.168.0 in project knowledge.

| Capability | Endpoint / field |
|---|---|
| Send any channel | `POST /messages-api/1/messages` |
| Validate before send | `POST /messages-api/1/messages/validate` |
| Channel selector | `messages[].channel` enum: `SMS`, `MMS`, `WHATSAPP`, `RCS`, `VIBER_BM`, `VIBER_BOT`, `MESSENGER`, `LINE_ON`, `INSTAGRAM_DM`, `APPLE_MB` |
| WhatsApp / RCS / Viber / Apple template message | Use `MessagesApiTemplateMessage` variant with `messages[].template` |
| Channel failover | `messages[].failover[]` + `MessagesApiChannelsDestination` per channel |
| Scheduled send | `options.schedule` (`RequestSchedulingSettings`) |
| Link tracking | `options.tracking` (`UrlOptions`) |
| Message ordering / pacing | `options.messageOrdering` |
| Per-message DLR webhook URL override | `messages[].webhooks.delivery.url` |
| Per-message Seen webhook (WA, Viber) | `messages[].webhooks.seen` |
| Bulk response | `bulkId` + per-message `messageId` echoed on every DLR/MO |

Recommendation: target the unified Messages API for all new code
(SpeedToLead first). It's the explicit CPaaS-X direction, supports
every channel Hermes needs, and gives native channel failover.

---

## 5. Outbound — legacy per-channel paths

Still supported. Useful when migrating use cases already integrated
against the older per-channel APIs.

| Channel | Legacy endpoint | Current endpoint | Notes |
|---|---|---|---|
| SMS | `POST /sms/2/text/advanced` (V2) | `POST /sms/3/messages` (V3) | V3 is the current recommended per-channel SMS endpoint. App/Entity passed as direct fields on the message. |
| MMS | `POST /mms/1/advanced` | (current) | App/Entity as `applicationId`, `entityId` on each message. |
| Email | `POST /email/3/send` (V3) | `POST /email/4/messages` (V4) | V3 → V4 migration occurred; V4 uses JSON-based requests with templates. |
| WhatsApp template | `POST /whatsapp/1/message/template` | (current per-channel) | Or use the unified Messages API with `MessagesApiTemplateMessage`. |
| WhatsApp text (session) | `POST /whatsapp/1/message/text` | (current per-channel) | |
| Viber | `POST /viber/1/message` | (current per-channel) | |
| RCS / Apple MB / Messenger / LINE / Instagram DM | n/a (no legacy per-channel) | unified Messages API only | These channels are reachable only via `/messages-api/1/messages`. |

> Section 4 of the original Hermes context doc references the **legacy**
> versions (`/sms/2/text/advanced`, `/email/3/send`, `/whatsapp/1/message/template`).
> Those rows should be updated when the team commits to a target
> generation. Current recommendation: target `/messages-api/1/messages`.

---

## 6. Inbound messaging & delivery reports

| Hermes360 event | Infobip primitive | Endpoint / webhook |
|---|---|---|
| `MessageDeliveredEvent` / `MessageFailedEvent` (push) | DLR webhook | URL configured via Subscriptions API or `messages[].webhooks.delivery.url`. Schema: `MessagesApiDeliveryResult`. |
| `MessageDeliveredEvent` / `MessageFailedEvent` (pull) | Fetch DLRs | `GET /messages-api/1/reports?channel=…&applicationId=…&entityId=…&campaignReferenceId=…` |
| `InboundMessageReceivedEvent` (MO push) | Inbound webhook (per channel) | URL configured via Subscriptions API. |
| `InboundMessageReceivedEvent` (MO pull) | Fetch inbound batch | `GET /messages-api/1/inbound?channel=…&applicationId=…&entityId=…&campaignReferenceId=…` |
| Read receipts (WA, Viber) | Seen webhook | `webhooks.seen` per-message, or Subscriptions. |
| Manage subscriptions | Subscriptions API | `POST /subscriptions/1/subscription/{channel}`. Filter via `criteria[]` = list of `(applicationId, entityId)` pairs. Validated against `subscriptions-api.json`. |

Reverse correlation (Infobip → Hermes):

| Need | Field on the inbound payload |
|---|---|
| Which tenant? | `platform.entityId` |
| Which use case? | `platform.applicationId` |
| Which campaign? | `campaignReferenceId` |
| Which ContactAttempt did this MO reply to? | `correlationData` (echoed from outbound) |
| Which ContactAttempt does this DLR concern? | `messageId` + `callbackData` |

---

## 7. Status & error normalization

| Hermes360 | Infobip field | Notes |
|---|---|---|
| `MessageStatus` enum | `status.groupName` (`PENDING`, `UNDELIVERABLE`, `DELIVERED`, `EXPIRED`, `REJECTED`) + `status.name` | Group is stable; `name` is finer. Map both via `InfobipStatusMapper`. |
| `ProviderError` | `error.groupName` + `error.name` + `error.permanent` | `permanent` flag drives Hermes retry logic. |
| Bulk correlation | response `bulkId` + per-message `messageId` | Echoed on every DLR/MO. |
| `IdempotencyKey` (Hermes) | `messages[].destinations[].messageId` | Use a deterministic id from `ContactAttempt`. |
| `CorrelationId` (Hermes) | `bulkId` + `messageId` (system) and `correlationData` + `callbackData` (user-defined) | Propagate both ways for traceability. |

### 7.1 Message-dispatch tracking table (`InfobipMessageDispatch`)

The dispatch slice **does** persist — but only the response metadata, not
the message body. The body lives in Hermes' own outbound store (or is
ephemeral); this table is the durable handle for refresh / retry / error
recovery.

**Granularity.** One row per Infobip `messageId` (per recipient). A bulk
send that fans out to 50 destinations produces one row per destination,
all sharing the same `BulkId`.

**Write path.**

1. **On send response** — for every `MessageAck` in `SendMessagesResponse`,
   `INSERT` a row with `BulkId`, `MessageId`, `Destination` (= `to`),
   initial `LatestStatusGroupName` (usually `PENDING`), and the
   `(applicationId, entityId, campaignReferenceId, channel)` context.
2. **On DLR push** (Subscriptions webhook) — match by `MessageId`,
   `UPDATE` the `LatestStatus*` / `LatestError*` columns, bump
   `LastStatusRefreshAt`, and set `DeliveredAt` / `FinalizedAt` when the
   status reaches a terminal `groupName` (`DELIVERED`, `UNDELIVERABLE`,
   `EXPIRED`, `REJECTED`).
3. **On DLR pull** (`GET /messages-api/1/reports`) — same as push; used
   as the recovery path when a webhook is missed.
4. **Background refresh worker** — for rows still in `PENDING` older
   than the refresh window, fire `GET /reports?messageId=…` to drain
   the report queue and update state.
5. **Retry path** — for rows with a non-permanent error
   (`LatestErrorIsPermanent = false`), Hermes may re-send. The retry
   produces a **new** `messageId` (and therefore a new row); the
   previous row keeps its terminal status as the audit trail, with
   `RetryOfMessageId` pointing back to the predecessor.

**Why per-`messageId` and not per-`bulkId`.** A bulk's children can
diverge: one recipient `DELIVERED`, another `REJECTED` for invalid
MSISDN, another `EXPIRED` after 24 h. Status refresh and retry are
per-recipient operations, so the primary record is per-recipient. The
`BulkId` index lets us still answer "how did batch X do overall?" in one
query.

**Columns** (full schema lives in `visual-summary.html` §10):

| Column | Source |
|---|---|
| `MessageId` (unique) | `MessageAck.messageId` |
| `BulkId` (indexed) | `SendMessagesResponse.bulkId` |
| `Destination` | `MessageAck.to` |
| `InfobipApplicationId`, `InfobipEntityId` | `options.platform.*` echoed back |
| `CampaignReferenceId` | `options.campaignReferenceId` |
| `Channel` | top-level `channel` on the request |
| `Sender` | `messages[].sender` |
| `LatestStatusGroupId`, `LatestStatusGroupName`, `LatestStatusId`, `LatestStatusName`, `LatestStatusDescription` | initially from `MessageAck.status`, then refreshed from DLR |
| `LatestErrorGroupName`, `LatestErrorName`, `LatestErrorIsPermanent` | DLR `error.*` |
| `CorrelationData`, `CallbackData` | echoed from outbound; used to re-link a DLR/MO to a `ContactAttempt` if the row is missing |
| `RetryCount`, `RetryOfMessageId` (nullable, FK self) | Hermes retry chain |
| `SentAt`, `LastStatusRefreshAt`, `DeliveredAt`, `FinalizedAt` | timestamps |
| `CreatedAt`, `UpdatedAt` | row audit |

**Body is explicitly NOT stored.** No `Text`, `Body`, `Template`,
`Variables`, or `MediaUrl` columns. Two reasons:
- GDPR / data-minimisation: outbound bodies are often customer PII
  (names, OTPs, transactional details).
- Storage growth: at Hermes' send volume the body is the row size
  driver; metadata is cheap.

If a future feature needs the body for replay, it should live in a
separate Hermes-side store under a stricter retention policy and join to
this table by `MessageId`.

---

## 8. Multi-instance topology

Hermes360 currently runs as 3 cloud instances; each instance hosts a
disjoint set of users; each user owns its own campaigns. The "instance"
dimension is **not** an Infobip concept and should not consume a
structured slot.

### 8.1 Outbound

No special handling. Each instance issues `POST /messages-api/1/messages`
with the `(applicationId, entityId, campaignReferenceId)` triple of the
calling user. Infobip is instance-blind.

### 8.2 Inbound routing (DLR + MO)

Solved at the **Subscription** layer, not by burning a structured slot:

- Each instance creates its own subscriptions
  (`POST /subscriptions/1/subscription/{channel}`), with `criteria[]`
  listing the `entityId`s of users it hosts and the webhook URL pointing
  at that instance.
- When a user is migrated between instances → update one subscription.
- Alternative for tenants with dedicated numbers: set the webhook URL
  at the **inbound configuration of the number** itself. Inbound config
  takes precedence over number-level config.

### 8.3 Per-instance billing/reporting

Three options, ranked:

1. **Derive instance from user.** Each user lives on exactly one
   instance, so `instance = lookup(user)`. Group consumption by instance
   in Hermes' own reporting layer; no Infobip-side dimension needed.
   **Recommended.**
2. Composite `applicationId` (`speedtolead-inst1`, `speedtolead-inst2`,
   …). Cardinality stays small (~3 × ~5 use cases = ~15 apps). Reporting
   becomes prefix-queries. Works but ugly.
3. Sacrifice the use case dimension — let `applicationId = instance-N`
   and put use case identification in `callbackData`. Loses per-use-case
   sending strategies. **Not recommended.**

---

## 9. Tenant provisioning sequence

When a new Hermes customer is onboarded, the following Infobip-side
operations must occur **once**:

1. `POST /provisioning/1/entities` — create the Entity.
   - Body: `{ "entityId": "<Hermes Customer.CustomerId>", "entityName": "<human-readable name>" }`.
   - **`entityName`**, not `name` — a wrong key is silently accepted and
     the alias persists as `null`. Always read back via `GET
     /provisioning/1/entities/{entityId}` to confirm the alias landed.
2. (When a tenant brings a WhatsApp number) Tenant completes Meta
   Embedded Signup in Hermes' hosted ES widget (Infobip's Tech
   Provider Program). On callback, Hermes calls
   `POST /whatsapp/1/embedded-signup/registrations/share-waba` with
   the captured `wabaId` and `phoneNumberId`. Infobip's Automatic
   Resource Association binds the resulting resource to the tenant's
   `entityId` during ES — Hermes does not POST a Resource
   Association. Registration completion (`IN_PROGRESS` /
   `FINISHED` / `FAILED`) is reported asynchronously via the
   registration webhook, which populates Hermes' local mapping table.
3. (Optional, per-tenant override) Create a Sending Strategy at
   `(App, Entity)` level if the tenant needs different sender selection
   from the use case default.
4. (Recommended, not yet implemented) Create a CPaaS-X-linked API key
   scoped to the Entity (or to specific `(App, Entity)` pairs). Today
   Hermes uses a single account-level key from `appsettings.json`.
5. (Per channel) Update the Subscription that delivers webhooks to the
   instance hosting this user — add the new `entityId` to its
   `criteria[]` of `POST /subscriptions/1/subscription/{channel}`.

Use-Case-level Applications are created **automatically on every Hermes
application startup** via `POST /api/Applications/bootstrap-speed2lead`
(idempotent GET-or-create), not per tenant and not as a manual operator
step.

This entire sequence is owned by the **Hermes Customer lifecycle**, not
by the messaging abstraction. `Hermes360.Messaging.*` should not own
provisioning calls; it consumes already-provisioned IDs. In the current
implementation, `HermesMessaging.ApiService` exposes the provisioning
operations as REST endpoints (`/api/Entities`,
`/api/Subscriptions/{channel}`, and the Embedded Signup controller's
share-waba route), which the Hermes tenant-lifecycle code calls from
the deployer's Hermes360 process. The tenant-facing number reads
(`GET /api/Numbers[?entityId=...]`) serve from Hermes' local mapping
table once it lands; `GET /api/admin/numbers/inventory` is an
admin-only reconciliation passthrough to Infobip Resource Management.

---

## 10. Gaps and open questions

> **Resolved as of 2026-05-12** (kept for history; details now live in
> the front-matter status banner and §3.1):
>
> - **Q2 — Outbound endpoint generation.** Decided: target the unified
>   Messages API (`POST /messages-api/1/messages`). Verified end-to-end
>   on 2026-05-11 against `xr5e2e.api.infobip.com` (WhatsApp text +
>   template + DLR roundtrip).
> - **Q3 — Tenant provisioning runbook.** Decided: Hermes is the only
>   consumer of Infobip; the Hermes Customer-lifecycle code calls the
>   provisioning endpoints exposed by `HermesMessaging.ApiService`.
>   Customer↔Entity is eager and 1:1 at customer-creation time.
> - **New finding (numbers, refactored 2026-05-18).** Hermes never
>   procures from Infobip's inventory. WhatsApp onboarding flows
>   through Meta Embedded Signup (Infobip Tech Provider Program) and
>   `POST /whatsapp/1/embedded-signup/registrations/share-waba`, on a
>   separate controller. `NumbersController` now exposes only tenant
>   read endpoints, which serve from Hermes' local mapping table once
>   it lands (today: stubs returning 501). The single remaining call
>   to Infobip Resource Management lives on
>   `AdminNumbersController.Inventory` for reconciliation / audit.
>   Resource Request and Resource Association are out of scope. The
>   legacy `/numbers/1/numbers` endpoint silently ignores `entityId`;
>   do not use it for tenant filtering.

1. **Hermes C# class names are still TODO.** Section 2 of the original
   context doc lists placeholders (`Campaign`, `ContactAttempt`,
   `MessageTemplate`, …). Once class names, namespaces, and table
   names are filled in, every row in §3 here needs the concrete Hermes
   type added.

2. ~~**Endpoint version drift in the existing context doc.**~~ **Resolved
   2026-05-11** — target the unified Messages API. See banner above.

3. ~~**Tenant provisioning runbook missing.**~~ **Resolved 2026-05-12,
   refined 2026-05-18** — provisioning is owned by the Hermes
   Customer-lifecycle code, calling `/api/Entities` and
   `/api/Subscriptions/{channel}` on `HermesMessaging.ApiService`.
   WhatsApp tenant onboarding goes via Meta Embedded Signup → share-waba
   on a separate controller. Tenant-facing number reads serve from
   Hermes' local mapping table once it lands; admin reconciliation
   uses `/api/admin/numbers/inventory`.

4. **API key strategy unspecified.** Three options on the table — one
   global key, one key per Application/use case, one key per
   `(App, Entity)` pair. Today's implementation uses **one global key**
   per deployment (supplied by the deployer at install time, stored in
   `appsettings.json` under `Infobip.ApiKey`). Per-`(App, Entity)` keys
   remain the recommended hardening target. Affects
   `IMessagingProviderFactory` design.

5. **Sending Strategy ownership.** Use-Case-wide or per-tenant? Default
   recommendation: define at Application level, allow per-tenant
   override via `(App, Entity)` strategy.

6. **WhatsApp template registry.** Out of scope per phase-1, but
   flagged: WA templates are approved at the WhatsApp Business Account
   level. If multiple Hermes tenants share one WABA, the template
   namespace is shared. If each tenant has its own WABA, the
   abstraction must resolve templates by `(tenant, template-name)`.
   Need to know which model.

7. **`campaignReferenceId` length/format constraints not validated.**
   Hermes campaign IDs may be GUIDs or composite strings; need to
   confirm Infobip accepts the exact format. Same question for
   client-supplied `messageId`.

8. **Voice / `callsConfigurationId`.** Out of scope per phase-1 but
   symmetric to Application/Entity. If Hermes plans to route Voice
   through Infobip later, the same use-case-as-App, tenant-as-Entity
   model extends cleanly. Flagged so the abstraction does not paint
   itself into a corner.

9. **Subscription cardinality not decided.** One subscription per
   `(App, Entity)` pair = many subscriptions, surgical webhook
   routing. One subscription per Application with a list of Entity
   criteria = fewer subscriptions, fan-in at the receiver. Both are
   supported. Affects how `Hermes360.Messaging.*` parses inbound
   webhooks. Recommendation: one subscription per Application per
   instance, with all hosted `entityId`s in `criteria[]`.

10. **Inbound number ↔ tenant routing.** When an MO arrives on a
    number, Infobip routes to `(App, Entity)` based on the number's
    general settings or its inbound configuration. Each tenant's
    inbound numbers must carry the tenant's `entityId` at the
    number-config level, otherwise inbound messages cannot be
    attributed to the tenant.

11. **Per-instance billing dimension.** Recommendation in §8.3 is to
    derive instance from user. If the team wants instance to be a
    first-class Infobip-side dimension, switch to composite
    `applicationId` strategy and update §3.1.

---

## 11. Channel as a billing/reporting dimension

**Channel is orthogonal to the `(App, Entity, Campaign)` model.** It
does not consume a structured slot. Every outbound message carries
`channel` as a top-level field, and every Infobip reporting/inbound
endpoint accepts it as a query parameter.

### 11.1 The four-dimensional billing slice

`(use case, client, campaign, channel)` is queryable in a single call:

```
GET /messages-api/1/reports
    ?applicationId=<use case>          # use case
    &entityId=<tenant>               # client
    &campaignReferenceId=<id>        # campaign
    &channel=WHATSAPP                # channel
```

Same query shape applies to:
- `GET /messages-api/1/inbound` — inbound MO volume per dimension.
- DLR webhook payloads — every report carries `channel`,
  `platform.applicationId`, `platform.entityId`, and
  `campaignReferenceId`, so Hermes' own consumption rollup can compute
  the same slice from received DLRs without polling.

### 11.2 Today: WhatsApp only

SpeedToLead sends with:

```
channel:             WHATSAPP
applicationId:       speedtolead
entityId:            <hermes-tenant>
campaignReferenceId: <hermes-campaign>
```

Hermes' billing job filters consumption on those four. No special case.

### 11.3 Tomorrow: WhatsApp + SMS (or any second channel) on the same use case

The mapping does not change. The same campaign on the same tenant emits:

```
# WhatsApp attempt
channel: WHATSAPP, applicationId: speedtolead, entityId: u42, campaignReferenceId: c99

# SMS attempt (failover or parallel) for the same campaign
channel: SMS,      applicationId: speedtolead, entityId: u42, campaignReferenceId: c99
```

Billing question *"what did campaign c99 cost tenant u42 in October,
broken down by channel?"* → either run the report once per channel, or
omit the `channel` filter and group by it client-side.

### 11.4 What the unified Messages API gives you here

Targeting `POST /messages-api/1/messages` (per §4 recommendation)
specifically pays off when channels are added:

- **One endpoint, all channels.** Adding SMS to SpeedToLead = setting
  `"channel": "SMS"` on the request, not a new integration. No new
  HTTP client, no new auth wiring, no new DTOs.
- **Native channel failover.** `messages[].failover[]` lets a single
  campaign attempt WhatsApp, fall back to SMS. Each attempt emits its
  own DLR with its own `channel`, so billing attributes correctly to
  whichever channel actually delivered.
- **Same status/error mappers across channels.** `InfobipStatusMapper`
  and `InfobipErrorMapper` do not grow when channels are added —
  status/error fields are identical in shape across channels.

By contrast, sticking with legacy per-channel endpoints
(`/sms/3/messages`, `/whatsapp/1/message/template`,
`/email/4/messages`, `/viber/1/message`, …) means each new channel is
a separate integration inside `Hermes360.Messaging.Infobip`.

### 11.5 What changes operationally per channel

Channel is free in the *mapping*, but three operational concerns are
genuinely per-channel:

1. **Resource provisioning is per channel.** A WhatsApp sender (WABA
   number), an SMS sender (long number / alpha ID), and an email
   sender (domain) are entirely different resource types. Each needs
   its own `POST /provisioning/1/associations` call against the same
   `(App, Entity)` pair. The tenant provisioning sequence in §9
   becomes a per-channel loop:

   ```
   For each channel the tenant uses:
       associate the channel-specific sender resource to (App, Entity).
   ```

2. **Sending Strategy can be channel-scoped.** Per Infobip's docs,
   sending strategies can be limited to a specific channel. A tenant
   running WhatsApp + SMS may need two strategy entries, both keyed
   off the same `(App, Entity)`. Confirm exact channel-scoping
   behaviour with Infobip support before the second-channel
   integration starts (open question — see §12 #12).

3. **Tenant × channel capability gate in Hermes.** Not every tenant
   uses every channel. `IMessagingProviderFactory.GetProvider(tenant,
   channel)` must validate the pair *before* attempting a send,
   otherwise the call fails late at Infobip with "no sender
   available". Two implementation options:

   - Query `GET /provisioning/1/associations?entityId=…` at runtime,
     cached. Source of truth = Infobip. Slower; Infobip is canonical.
   - Maintain a tenant-channel capability table in Hermes' own
     tenant config, kept in sync at provisioning time. Faster; source
     of truth = Hermes.

   **R              