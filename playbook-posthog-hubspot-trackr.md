# PLG Playbook — PostHog + HubSpot for Trackr
**Author:** Adrien Balcioglu · Freelance PM · Eidos Labs
**Version:** 1.0 · June 2026

> Operational guide for implementing the PostHog → HubSpot PLG stack on Trackr.
> Companion to the [Instrumentation PRD](./PRD.md).

---

## Step 1 — PostHog Setup

### 1.1 Install the JavaScript SDK

```js
// In your app's entry point (e.g. index.js or _app.tsx)
import posthog from 'posthog-js'

posthog.init('YOUR_POSTHOG_PROJECT_API_KEY', {
  api_host: 'https://app.posthog.com',
  capture_pageview: true,
  persistence: 'localStorage'
})
```

### 1.2 Identify users on login

```js
// Call after successful authentication
posthog.identify(user.id, {
  email: user.email,
  name: user.name,
  company: user.company,
  plan: user.plan,                         // 'free' | 'starter' | 'pro'
  signup_date: user.createdAt,
  usage_events_this_month: user.monthlyUsage
})
```

### 1.3 Set up group analytics (workspace-level)

```js
// Call after identify, once company context is available
posthog.group('company', user.companyId, {
  name: user.companyName,
  plan: user.companyPlan,
  seat_count: user.companySeats,
  mrr: user.companyMrr
})
```

---

## Step 2 — Event Instrumentation

### 2.1 Activation events

```js
// Project created
posthog.capture('project_created', {
  project_id: project.id,
  language: project.language,    // 'node' | 'python' | 'go' | etc.
  framework: project.framework
})

// SDK installed (first event received from user's app)
// Fire server-side when first inbound event is received
posthog.capture('sdk_installed', {
  project_id: project.id,
  sdk_version: event.sdkVersion,
  platform: event.platform
})

// Dashboard viewed with live data — the aha moment
posthog.capture('dashboard_viewed', {
  project_id: project.id,
  dashboard_id: dashboard.id,
  data_points_count: dashboard.dataPointsCount,  // must be > 0
  is_first_view: isFirstView
})
```

### 2.2 Engagement events

```js
// Custom insight / chart built
posthog.capture('insight_created', {
  insight_type: 'timeseries' | 'funnel' | 'heatmap',
  project_id: project.id
})

// Alert configured
posthog.capture('alert_configured', {
  alert_type: 'error_rate' | 'latency' | 'volume',
  project_id: project.id
})

// Teammate invited
posthog.capture('teammate_invited', {
  invitee_email: invite.email,   // hashed recommended for privacy
  role: invite.role
})
```

### 2.3 Expansion events

```js
// Upgrade page viewed
posthog.capture('upgrade_page_viewed', {
  source: 'sidebar' | 'limit_banner' | 'direct',
  current_plan: user.plan,
  usage_pct: Math.round((user.monthlyUsage / user.planLimit) * 100)
})

// Limit reached (fire server-side when threshold hit)
posthog.capture('limit_reached', {
  limit_type: 'events' | 'seats' | 'retention',
  current_usage: user.monthlyUsage,
  plan_limit: user.planLimit
})

// Plan upgraded (fire server-side on successful payment)
posthog.capture('plan_upgraded', {
  previous_plan: previousPlan,
  new_plan: newPlan,
  mrr: newMrr,
  trigger_source: 'limit_banner' | 'upgrade_page' | 'sales'
})
```

---

## Step 3 — Funnels & Cohorts in PostHog

### 3.1 Build the activation funnel

In PostHog → **Funnels**:

```
Step 1: user_signed_up
Step 2: project_created
Step 3: sdk_installed
Step 4: dashboard_viewed  (filter: data_points_count > 0)

Conversion window: 7 days
Breakdown: by plan, by referrer, by language
```

**What to look for:** The biggest drop-off step is where engineering or onboarding effort should go first.

### 3.2 Build the expansion funnel

```
Step 1: dashboard_viewed (3rd occurrence — use "performed event N times")
Step 2: teammate_invited OR insight_created (use "any of" filter)
Step 3: upgrade_page_viewed
Step 4: plan_upgraded

Conversion window: 30 days
```

### 3.3 Create cohorts

Navigate to **PostHog → Cohorts → New cohort**:

| Cohort | Filter logic |
|---|---|
| `activated_users` | Performed `dashboard_viewed` where `data_points_count > 0` within 7 days of `user_signed_up` |
| `power_users` | Performed any event ≥ 5 times in last 7 days AND performed ≥ 2 distinct event types |
| `at_risk` | In `activated_users` AND last event > 14 days ago |
| `near_limit` | Has property `usage_pct >= 80` |
| `expansion_ready` | Performed `teammate_invited` ≥ 3 times AND in `power_users` |

---

## Step 4 — PostHog → HubSpot Sync

### 4.1 Option A — Native PostHog connector (recommended)

1. Go to **PostHog → Data pipelines → Destinations**
2. Select **HubSpot**
3. Authenticate with HubSpot OAuth
4. Map PostHog person properties to HubSpot contact properties

PostHog will sync person property updates to HubSpot contacts in near real-time.

### 4.2 Option B — Webhook + n8n (more control)

```js
// PostHog webhook payload on event capture
{
  "event": "dashboard_viewed",
  "distinct_id": "user_123",
  "properties": {
    "email": "jane@acme.com",
    "plan": "free",
    "usage_pct": 67,
    "project_id": "proj_abc"
  },
  "timestamp": "2026-06-01T10:00:00Z"
}
```

n8n workflow:
```
Webhook (PostHog) → Filter by event name → HubSpot: Update contact property
                                          → HubSpot: Set lifecycle stage
                                          → HubSpot: Enroll in workflow (if PQL)
```

### 4.3 Custom HubSpot contact properties to create

Go to **HubSpot → Settings → Properties → Contact properties → Create**:

| Property name | Type | Description |
|---|---|---|
| `posthog_plan` | Single-line text | Current Trackr plan |
| `posthog_activated` | Boolean | Has reached aha moment |
| `posthog_monthly_events` | Number | Events sent this month |
| `posthog_usage_pct` | Number | % of plan limit used |
| `posthog_seat_count` | Number | Number of teammates invited |
| `posthog_last_seen` | Date | Last activity date |
| `posthog_pql` | Boolean | Meets PQL criteria |

---

## Step 5 — HubSpot Workflows

### Workflow 1 — Activation confirmation

**Trigger:** Contact property `posthog_activated` is set to `true`

**Actions:**
1. Set lifecycle stage → SQL
2. Send email: "You're live on Trackr 🎉" (template: activation_confirmation)
3. Wait 3 days
4. Send email: "Your first week on Trackr — 3 things to try next"

---

### Workflow 2 — Near-limit upgrade nudge

**Trigger:** Contact property `posthog_usage_pct` is greater than 80

**Actions:**
1. Send email: "You've used 80% of your monthly events"
   - Include dynamic usage bar (use HubSpot personalization tokens)
   - CTA: "See upgrade options"
2. If `posthog_usage_pct` > 95 after 3 days:
   - Send second email with urgency copy
   - Create HubSpot task for sales rep if `posthog_seat_count >= 2`

---

### Workflow 3 — PQL sales alert

**Trigger:** Contact property `posthog_pql` is set to `true`

**Actions:**
1. Set lifecycle stage → Opportunity
2. Create task for sales rep: "PQL alert — [contact name] at [company]"
   - Priority: High
   - Due date: Today + 1 day
3. Send internal Slack notification (via HubSpot → Slack integration):
   `"New PQL: Jane Doe (Acme Corp) — 3 seats, 67% usage, visited upgrade page"`

---

### Workflow 4 — Re-engagement

**Trigger:** Contact property `posthog_last_seen` is more than 14 days ago AND `posthog_activated = true`

**Actions:**
1. Send email: "It's been a while — here's what's new on Trackr"
2. If no activity after 7 more days:
   - Send final email: "Is Trackr still useful for you?" (with unsubscribe option)
3. If still no activity: set lifecycle stage → Other (dormant)

---

## Step 6 — Monitoring & Iteration

### Weekly PLG review checklist

- [ ] Check activation funnel drop-off in PostHog — any new bottleneck?
- [ ] Review `at_risk` cohort size — is it growing?
- [ ] Check `near_limit` cohort → upgrade conversion rate
- [ ] Review PQL tasks in HubSpot — any unactioned leads?
- [ ] Check `plan_upgraded.trigger_source` breakdown — which trigger drives most conversions?

### Key questions to answer each sprint

1. Where do users drop off in the activation funnel this week?
2. What's the average time from `sdk_installed` to `dashboard_viewed`?
3. Which feature usage correlates most with conversion? (use PostHog correlation analysis)
4. Are PQL contacts converting within the expected window?

---

## Architecture Overview

```
Trackr App (frontend + backend)
        │
        ├── posthog.capture() ──────────────────────► PostHog Cloud
        ├── posthog.identify()                              │
        └── posthog.group()                                 │
                                                    Person properties
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

*This playbook is a portfolio exercise authored by Adrien Balcioglu. Trackr is a fictitious product. All code samples are illustrative.*
