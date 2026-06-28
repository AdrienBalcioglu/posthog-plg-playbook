# PostHog PLG Playbook — PostHog + HubSpot

> A practical PM guide to instrumenting a SaaS product with PostHog and syncing it with HubSpot to power a Product-Led Growth motion — from event taxonomy to PQL scoring.

**Fictitious product:** Trackr — a developer analytics SaaS (API performance monitoring, error tracking, usage insights). Freemium model: free up to 1M events/month, paid from $49/month.

**Stack:** PostHog (product analytics) · HubSpot (CRM) · n8n (optional middleware)

---

## What's in this repo

| File | Description |
|---|---|
| [`PRD.md`](./PRD.md) | Instrumentation PRD — event taxonomy, funnel definitions, cohorts, PostHog → HubSpot sync spec, PQL definition, success metrics |
| [`PLAYBOOK.md`](./PLAYBOOK.md) | Operational playbook — SDK setup, event capture code samples, funnel config, HubSpot workflow specs, weekly review checklist |

---

## The PLG architecture

```
Trackr App (frontend + backend)
        │
        ├── posthog.capture() ──────────────────► PostHog Cloud
        ├── posthog.identify()                         │
        └── posthog.group()                     Person properties
                                                Cohorts / Funnels
                                                Dashboards
                                                       │
                                         HubSpot Connector / n8n
                                                       │
                                               ┌───────▼────────┐
                                               │   HubSpot CRM  │
                                               │  - Contacts    │
                                               │  - Lifecycle   │
                                               │  - Workflows   │
                                               │  - PQL tasks   │
                                               └────────────────┘
```

---

## Key concepts covered

**Aha moment definition**
Trackr's aha moment = first API project connected + first dashboard with live data viewed within 7 days of signup. Every instrumentation decision flows from this definition.

**Event taxonomy**
10 core events structured around three PLG stages: activation (`project_created`, `sdk_installed`, `dashboard_viewed`), engagement (`insight_created`, `alert_configured`, `teammate_invited`), expansion (`upgrade_page_viewed`, `limit_reached`, `plan_upgraded`).

**Funnel architecture**
Three funnels: activation (7-day window, signup → aha moment), expansion (30-day window, recurring usage → paid), limit-to-upgrade (urgency-driven conversion). Each with breakdown dimensions and target conversion rates.

**Cohort strategy**
Five operational cohorts: `activated_users`, `power_users`, `at_risk`, `near_limit`, `expansion_ready` — each mapped to a specific action (re-engagement, sales outreach, upgrade nudge).

**PostHog → HubSpot sync**
Seven custom HubSpot contact properties synced from PostHog (plan, activated status, usage %, seat count, last seen). Lifecycle stage mapping from Lead → MQL → SQL → Opportunity → Customer driven entirely by product signals.

**PQL scoring**
A contact is flagged as Product-Qualified Lead when 3 of 5 conditions are met: activated, usage ≥60%, ≥2 seats, has created an insight, has visited the upgrade page. No sales call needed to qualify.

**HubSpot workflows**
Four automated workflows: activation confirmation, near-limit upgrade nudge, PQL sales alert (with Slack notification), re-engagement sequence for dormant activated users.

---

## Why this matters for PLG

Most SaaS teams treat their CRM and their analytics tool as separate systems. The PLG approach connects them so that product behavior — not form fills, not demo requests — drives the entire revenue motion:

- A user who installs the SDK becomes an MQL automatically
- A user who reaches the aha moment becomes an SQL automatically
- A user who invites teammates and hits 80% usage gets a tailored upgrade email — no SDR involved
- A sales rep only gets notified when a PQL is identified — high signal, no noise

---

## About

Built by **[Adrien Balcioglu](https://github.com/AdrienBalcioglu)** — Freelance PM with 5+ years on mobile analytics and SaaS platforms (PostHog, HubSpot, Databricks, Salesforce integrations at scale).

This repo is a portfolio project. Trackr is a fictitious product. All code samples are illustrative and do not represent a production implementation.

---

*Open to freelance PM missions and full-time roles at product-driven companies. [Let's connect →](https://www.linkedin.com/in/adrienbalcioglu)*
