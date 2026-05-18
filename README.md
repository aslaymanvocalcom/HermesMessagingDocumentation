# Hermes360 Messaging — Documentation

Documentation for **Hermes360 Messaging Provider Integration Layer** — the
messaging integration platform that connects Hermes360 to channel
providers (currently Infobip) and tenant-owned WhatsApp Business
Accounts (via Meta's Embedded Signup / Tech Provider Program).

The companion source code lives in
[`HermesMessaging`](../HermesMessaging).

## Browse online (GitHub Pages)

When GitHub Pages is enabled on this repo, the docs are published at:

- **Landing page** — [`index.html`](index.html)
- **Architectural reference** — [`visual-summary.html`](visual-summary.html)
- **Tenant onboarding guide** — [`tenant-whatsapp-onboarding-guide.html`](tenant-whatsapp-onboarding-guide.html)

## Browse here on GitHub

Markdown references (intended for engineers reading the repo, not for the
Pages site):

- [`Hermes360_Infobip_Mapping.md`](Hermes360_Infobip_Mapping.md) — canonical
  mapping between Hermes360 entities and Infobip CPaaS-X primitives
- [`hermes-infobip-provisioning-plan.md`](hermes-infobip-provisioning-plan.md)
  — design plan with implementation status snapshot
- [`task-tenant-number-endpoints.md`](task-tenant-number-endpoints.md) —
  historical task brief for the numbers endpoint slice, with a §0 deviation
  log capturing how the shipped code differs from the original plan

## Enabling GitHub Pages

After pushing to GitHub:

1. **Settings → Pages**
2. **Source:** Deploy from a branch
3. **Branch:** `main` (or `master`), **folder:** `/ (root)`
4. **Save**

The `.nojekyll` file in the repo root disables Jekyll, so the HTML files
are served as-is without theme processing.

## Repo layout

```
.
├── .nojekyll                              # disable Jekyll on GitHub Pages
├── README.md                              # this file (GitHub repo landing)
├── index.html                             # GitHub Pages landing
├── visual-summary.html                    # architectural reference (HTML)
├── tenant-whatsapp-onboarding-guide.html  # tenant onboarding guide (HTML)
├── Hermes360_Infobip_Mapping.md           # canonical mapping reference (MD)
├── hermes-infobip-provisioning-plan.md    # design plan (MD)
└── task-tenant-number-endpoints.md        # task brief, historical (MD)
```

## Updating the docs

The docs in this repo are the canonical copies. Edit them directly here —
they are no longer duplicated in the `HermesMessaging` source repo.
