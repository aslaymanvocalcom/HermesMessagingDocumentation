# Task: Hermes API endpoints for tenant phone numbers

**Hand-off to:** Claude Code, new session
**Estimated scope:** 1–2 days of focused work
**Created:** 2026-05-11
**Status:** **Shipped 2026-05-12 with scope deviations — see §0 below before reading the original brief.**

---

## 0. What actually shipped (read this first)

The brief below is preserved for history. Three deviations were made
during implementation, all deliberate, all aligned with the project
direction documented in `hermes-infobip-provisioning-plan.md` §0 and
§9:

1. **No DB persistence in this slice.** The `phone_number` table
   described in §3 of the original brief was not created. The endpoints
   are stateless passthroughs over the three Infobip Resource APIs
   (Resource Management, Resource Request, Resource Association). The
   table — plus the state machine, `last_synced_at`, and metadata —
   remains a follow-up task once tenant-context middleware lands.

2. **`POST /api/numbers/request` was renamed to `POST /api/Numbers/register`
   and it DOES call Infobip.** The brief described it as a "request
   only — no Infobip call" stub. Project direction changed: tenants
   bring their own MSISDN, and the endpoint enrolls it on Infobip via
   `POST /resource/1/resource-requests` immediately. For WhatsApp the
   `business` bag carries Meta business verification fields forwarded
   verbatim. Infobip 200 is rewritten to 201 to match REST creation
   semantics; everything else passes through.

3. **A third endpoint was added: `POST /api/Numbers/{numberKey}/associate`.**
   This binds an already-provisioned `numberKey` to a tenant via
   `POST /resource/1/applications/{appId}/entities/{entityId}/resources`
   (Resource Association). It's the natural follow-up to `/register`
   once Infobip approves the Resource Request (immediately for
   SMS/Voice, after Meta verification for WhatsApp).

**Endpoint surface as built:**

| Verb | Path | Backed by |
|---|---|---|
| GET | `/api/Numbers` (no filter) | **TODO** — local DB (populated by the Embedded Signup registration webhook). Stub returning 501 until the mapping table lands. |
| GET | `/api/Numbers?entityId=<id>` | **TODO** — local DB filtered by `entityId`. Same stub / 501. |
| GET | `/api/admin/numbers/inventory` | Resource Management — `GET /resource/1/resources`. **Admin-only**, reconciliation / audit only. |

> **Refactor 2026-05-18.** The previous four-row passthrough surface
> (Resource Management + Resource Association list + Resource Request +
> Resource Association create) collapsed to this three-row shape. The
> drivers: (a) WhatsApp onboarding flows through Meta Embedded Signup
> (TPP) → `POST /whatsapp/1/embedded-signup/registrations/share-waba`
> on a separate controller, so `/api/Numbers/register` (Resource
> Request) is gone; (b) Infobip's Automatic Resource Association
> auto-binds during ES, so `/api/Numbers/{numberKey}/associate`
> (Resource Association) is gone; (c) tenant-facing reads should serve
> from Hermes' local DB rather than proxying Infobip JSON, so the two
> read endpoints become stubs until the mapping table lands; (d)
> Resource Management remains in the design only for admin
> reconciliation, behind `/api/admin/numbers/inventory`.

**Validation actually enforced** (`NumbersController.Validate`):
required `channel` (must be `sms`/`whatsapp`/`voice`), required `country`
(ISO 3166-1 alpha-2 round-tripped through `RegionInfo`), required
`number` (E.164 — `^\+[1-9]\d{7,14}$`), optional `notes` ≤ 500 chars.
Auth/tenant-context wiring deferred — `entityId` is supplied in the
request body until middleware lands.

**Path verification status.** Only **one** `/resource/1/...` path
remains in scope after the 2026-05-18 refactor: Resource Management
(`GET /resource/1/resources`), used by the new
`AdminNumbersController` at `GET /api/admin/numbers/inventory`. The
Resource Request and Resource Association paths previously used by
`NumbersController` are removed from Hermes' design — Resource Request
because WhatsApp onboarding flows through Embedded Signup instead,
Resource Association because Infobip's Automatic Resource Association
handles tenant↔number binding during ES. The Resource Management path
is still INFERRED from Infobip documentation URL structure; the direct
probe in `HermesMessaging.ApiService.http` is the source of truth for
the live account's behaviour.

**Follow-ups explicitly carried forward:**

- `phone_number` and `campaign_phone_number` tables + state machine.
- Embedded Signup integration: Hermes-hosted ES widget (Meta JS SDK),
  Tech Provider Program enrolment with Meta, and the share-waba
  handoff (`POST /whatsapp/1/embedded-signup/registrations/share-waba`).
- ES registration webhook receiver — populates the local mapping
  table from `IN_PROGRESS` / `FINISHED` / `FAILED` payloads
  (`businessAccountId`, per-sender `phoneNumberId`,
  `displayPhoneNumber`, `status`, `businessPortfolioId`).
- Tenant-context middleware so `entityId` is derived from auth instead
  of accepted in the body.
- Tenant-facing `GET /api/Numbers[?entityId=...]` no longer
  passthrough — both return 501 today and will read from the local
  mapping table once it lands. The only remaining Infobip passthrough
  is `GET /api/admin/numbers/inventory` → Resource Management, and
  that endpoint is admin-auth gated by design (admin route prefix +
  `// TODO: enforce admin authorization once auth middleware lands`
  on the controller, with ingress-layer gating as the interim
  control).

---

## 1. Context (read this first)

Hermes360 is an internal Vocalcom platform that aggregates messaging providers behind a single API. We are building the Infobip integration layer. The architectural rule is: **Hermes is the only consumer of Infobip APIs. Tenants never see Infobip identifiers or APIs.**

The full design plan for the Hermes-Infobip integration is here:

```
C:\Users\AliSLAYMAN\source\repos\HermesMessaging\Hermes360 — Messaging Provider Integration Layer (Infobip)\hermes-infobip-provisioning-plan.md
```

**Read sections 3 (Data model), 5 (API surface), and 9 (Phone numbers) before you start.** They define the data shape and the contract.

The existing Hermes web service objects live here — read several files in this directory to learn the project's patterns before writing any code:

```
C:\Users\AliSLAYMAN\source\repos\6.4.0\hermes360\Admin\Web_Service\Objects
```

You will follow those existing patterns for routing, authentication, tenant context, persistence, error handling, and tests. **Do not introduce new frameworks, ORMs, or patterns.** If something is genuinely missing, flag it in your summary — don't invent.

## 2. What to build

Two REST endpoints on the tenant-facing Hermes API.

### `GET /api/numbers`
Returns the list of phone numbers belonging to the calling tenant.

Response item shape:
- `id`
- `number` (E.164 string, nullable if not yet assigned)
- `channel` (`sms` | `whatsapp` | `voice`)
- `country` (ISO 3166-1 alpha-2)
- `registration_state` (see state machine below)
- `registered_at` (nullable)
- `last_synced_at` (nullable)
- `created_at`

**Never return `infobip_sender_id` or any Infobip-internal field in a tenant-facing response.**

### `POST /api/numbers/request`
Creates a new phone number request for the calling tenant. This is a request only — actual procurement is manual/out-of-band in phase 1, so no Infobip call is made here.

Request body:
- `channel` (required, `sms` | `whatsapp` | `voice`)
- `country` (required, ISO 3166-1 alpha-2)
- `preferred_number` (optional, E.164 string)
- `notes` (optional, free text, max 500 chars)

Response: the newly created `phone_number` resource, in `registration_state = requested`, HTTP 201.

## 3. Data model — new table `phone_number`

Fields:

| Field                    | Type        | Notes                                                                                              |
|--------------------------|-------------|----------------------------------------------------------------------------------------------------|
| `id`                     | PK          | Whatever Hermes uses for primary keys                                                              |
| `tenant_id`              | FK, indexed | Foreign key to the existing tenants/customers table — match the existing FK style                  |
| `number`                 | string      | E.164, nullable until assigned                                                                     |
| `channel`                | enum/string | `sms` / `whatsapp` / `voice`                                                                       |
| `country`                | string(2)   | ISO 3166-1 alpha-2                                                                                 |
| `infobip_sender_id`      | string      | Nullable. Internal only — never serialized to tenant responses                                     |
| `registration_state`     | enum/string | See state machine. Default `requested`. NOT NULL                                                   |
| `registered_at`          | timestamp   | Nullable                                                                                           |
| `last_synced_at`         | timestamp   | Nullable                                                                                           |
| `metadata`               | JSON / text | Nullable. Reserved for Meta/WhatsApp business verification and similar future data                 |
| `created_at`             | timestamp   | NOT NULL                                                                                           |
| `updated_at`             | timestamp   | NOT NULL                                                                                           |

State machine values (string enum):
```
requested -> procuring -> pending_registration -> pending_approval -> active
                                                                   -> suspended
                                                                   -> decommissioned
```
You only need to write the row in `requested` for this task. State transitions belong to workers that are out of scope here.

If the Hermes codebase uses a schema migration tool, generate a migration through that tool. If it uses code-first / hand-rolled SQL, follow that.

## 4. Authentication & authorization

Use whatever auth mechanism the existing tenant-facing endpoints use. Required behavior:

- Unauthenticated request → 401.
- Authenticated tenant A → can only see and create numbers for tenant A.
- Tenant A must never be able to see tenant B's numbers (write a test that proves this).
- Admin/ops users are out of scope for this task — separate endpoints will exist later under `/admin/...`.

## 5. Validation

- `channel` must be one of the three allowed values → 400 if not.
- `country` must be a valid ISO 3166-1 alpha-2 code → 400 if not. A lookup table or library is fine; do not invent a custom country list.
- `preferred_number` if provided must be valid E.164 (`+` followed by 8–15 digits) → 400 if not.
- `notes` max 500 characters → 400 if longer.
- Unknown / extra fields in the request body: follow the existing Hermes convention (reject or ignore — look at other endpoints).

All error responses must use the same shape as existing Hermes endpoints.

## 6. Tests

- Unit tests for the controller/service: happy path, each validation failure, authorization scenarios (own-tenant, cross-tenant, unauthenticated).
- Integration tests for the endpoints using whatever test harness Hermes uses.
- Confirm `infobip_sender_id` is NOT present in any response payload (assert on the serialized output, not the model