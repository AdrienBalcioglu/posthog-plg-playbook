# PRD — PostHog PLG Instrumentation for Trackr
**Author:** Adrien Balcioglu · Freelance PM · Eidos Labs
**Version:** 1.0 · June 2026 · Status: Draft

---

## 1. Context

**Trackr** is a fictitious B2B SaaS product — an analytics platform built for developers that lets them monitor API performance, error rates, and usage patterns across their services. Trackr offers a freemium model: free up to 1M events/month, paid plans from $49/month.

This PRD specifies how to instrument Trackr with PostHog to power a Product-Led Growth (PLG) motion, synced with HubSpot as the CRM layer.

**Goal:** Use product usage data to automatically identify, score, and convert high-intent free users — without a dedicated sales team.

---

## 2. PLG Framework

The instrumentation is structured around three PLG stages:

| Stage | Question | PostHog role | HubSpot role |
|---|---|---|---|
| **Activation** | Did the user reach the "aha moment"? | Track milestone events | Create contact, set lifecycle stage |
| **Engagement** | Is the user getting recurring value? | Measure feature depth & frequency | Score the lead, trigger nurture |
| **Expansion** | Is the user ready to upgrade? | Detect limit approach & team signals | Trigger sales alert or upgrade email |

---

## 3. Aha Moment Definition

Trackr's aha moment = **"First API project connected + first dashboard with live data viewed within 7 days of signup."**

This is the activation gate. All instrumentation flows from it.

---

## 4. Event Taxonomy

### Naming convention
`[object]_[verb]` — snake_case, past tense. Examples: `project_created`, `dashboard_viewed`.

### Core events to instrument

| Event name | Trigger | Key properties |
|---|---|---|
| `user_signed_up` | Account creation | `plan`, `source`, `referrer` |
| `project_created` | User creates first API project | `project_id`, `language`, `framework` |
| `sdk_installed` | First event received from user's app | `project_id`, `sdk_version`, `platform` |
| `dashboard_viewed` | User opens a dashboard with live data | `project_id`, `dashboard_id`, `data_points_count` |
| `insight_created` | User builds a custom chart or query | `insight_type`, `project_id` |
| `alert_configured` | User sets up an error/threshold alert | `alert_type`, `project_id` |
| `teammate_invited` | User sends a team invite | `invitee_email`, `role` |
| `upgrade_page_viewed` | User visits pricing/upgrade page | `source`, `current_plan`, `usage_pct` |
| `plan_upgraded` | User converts to paid | `new_plan`, `mrr`, `trigger_source` |
| `limit_reached` | User hits free plan cap | `limit_type`, `current_usage`, `plan` |

### User identity
Call `posthog.identify()` on login with:
```json
{
  "email": "user@company.com",
  "name": "Jane Doe",
  "company": "Acme Corp",
  "plan": "free",
  "signup_date": "2026-06-01",
  "usage_events_this_month": 450000
}
```

### Group analytics (company-level)
Call `posthog.group('company', company_id, { name, plan, seat_count, mrr })` to track workspace-level behavior, not just individual users.

---

## 5. Funnel Definitions

### Funnel 1 — Activation funnel
```
user_signed_up
  → project_created
    → sdk_installed
      → dashboard_viewed  ← Aha moment
```
Target: >50% of signups reach `dashboard_viewed` within 7 days.

### Funnel 2 — Expansion funnel
```
dashboard_viewed (recurring, ≥3 sessions)
  → insight_created
    → teammate_invited OR alert_configured
      → upgrade_page_viewed
        → plan_upgraded
```

### Funnel 3 — Limit-to-upgrade funnel
```
limit_reached
  → upgrade_page_viewed
    → plan_upgraded
```
This funnel measures urgency-driven conversion. Target CTR from `limit_reached` to `upgrade_page_viewed`: >40%.

---

## 6. Key Dashboards in PostHog

| Dashboard | Metrics | Audience |
|---|---|---|
| **PLG Health** | Activation rate (7d), DAU/MAU, aha moment time | PM, Growth |
| **Funnel Monitor** | Drop-off at each activation step, by cohort | PM, Engineering |
| **Expansion Signals** | Workspaces with ≥3 users, usage >80%, alerts configured | PM, Sales |
| **Upgrade Pipeline** | upgrade_page_viewed → plan_upgraded conversion | PM, Finance |

---

## 7. Cohorts & Segments

Define the following cohorts in PostHog for targeting and analysis:

| Cohort | Definition | Use |
|---|---|---|
| `activated_users` | Completed aha moment within 7d of signup | Baseline for all retention metrics |
| `power_users` | ≥5 sessions/week + ≥2 features used | Expansion targets |
| `at_risk` | Activated but no activity in 14d | Re-engagement |
| `near_limit` | Usage >80% of free plan cap | Upgrade trigger |
| `expansion_ready` | ≥3 teammates invited + power user | Sales-assist trigger |

---

## 8. PostHog → HubSpot Sync

### Integration method
Use PostHog's **HubSpot connector** (native integration) or a middleware layer (n8n / Zapier / custom webhook) to sync events and properties to HubSpot contacts and companies.

### Contact properties to sync

| PostHog property | HubSpot property | Update trigger |
|---|---|---|
| `plan` | `posthog_plan` | On any plan change |
| `usage_events_this_month` | `posthog_monthly_events` | Daily sync |
| `activated` (bool) | `posthog_activated` | On `dashboard_viewed` |
| `last_active_date` | `posthog_last_seen` | On any event |
| `teammate_count` | `posthog_seat_count` | On `teammate_invited` |
| `usage_pct` | `posthog_usage_pct` | Daily sync |

### HubSpot lifecycle stage mapping

| PostHog signal | HubSpot lifecycle stage |
|---|---|
| `user_signed_up` | Lead |
| `sdk_installed` | Marketing Qualified Lead (MQL) |
| `dashboard_viewed` (aha moment) | Sales Qualified Lead (SQL) |
| `teammate_invited` OR `alert_configured` | Opportunity |
| `plan_upgraded` | Customer |

### HubSpot workflows triggered by PostHog data

| Trigger | Workflow | Action |
|---|---|---|
| `posthog_activated = true` | Activation confirmation | Send "you're set up" email + feature tip |
| `posthog_usage_pct > 80` | Near-limit nurture | Send upgrade email with usage breakdown |
| `posthog_seat_count >= 3` | Team expansion | Notify sales rep (HubSpot task) |
| `last_seen > 14 days` | Re-engagement | Send "what you missed" email |
| `limit_reached` event | Urgent upgrade | Immediate email + in-app banner |

---

## 9. PQL Definition (Product-Qualified Lead)

A contact is flagged as PQL in HubSpot when **3 of the 5 following conditions are true:**

1. `posthog_activated = true`
2. `posthog_usage_pct >= 60`
3. `posthog_seat_count >= 2`
4. `insight_created` fired at least once
5. `upgrade_page_viewed` fired at least once

PQL contacts trigger a HubSpot task assigned to the sales rep with priority = High.

---

## 10. Success Metrics

| Metric | Baseline | Target |
|---|---|---|
| Activation rate (7d) | — | >50% |
| Time to aha moment | — | <48h |
| Free-to-paid conversion | — | >8% |
| PQL-to-close rate | — | >25% |
| Expansion MRR (seat upgrades) | — | +15% MoM |
| Re-engagement rate (at_risk cohort) | — | >20% |

---

## 11. Out of Scope

- A/B testing of upgrade prompts (Phase 2)
- Salesforce integration (HubSpot first)
- Self-hosted PostHog deployment
- Mobile SDK instrumentation (web only for V1)
