# Real-Time Financial State Machine & Budget Alerting System

Fault-tolerant budget monitoring system built on a PostgreSQL state machine. Tracks every atomic budget change per project, detects state transitions, and fires alerts only when a project's health status actually changes — eliminating alert fatigue from duplicate notifications.

---

## Demo

[▶ Loom Walkthrough](https://www.loom.com/share/679eb54f26de4ab98d895c8f9c7f493b)

---

## The Problem

Project budget overruns are invisible until it is too late. There is no real-time visibility into which projects are burning through allocated hours, and threshold-based alerting systems fire on every execution cycle — including when the status has not changed — creating noise that engineers learn to ignore.

---

## Architecture

```
Trigger
  → [GET] Fetch_Active_Tasks (ClickUp API)
      → [TRANSFORM] Normalize_Payload      [INSERT] Log_API_Failure (DLQ)
          → [GET] DB_Previous_States (PostgreSQL)
              → [CALCULATE] Delta_Engine
                  → [UPSERT] PG_Save_State
                  → [GATE] Check_Alert_Flag
                        → [POST] Alert_Manager (Webhook)
```

---

## State Machine Design

Each project exists in exactly one of three states at any point in time:

```
HEALTHY  →  WARNING (burn rate ≥ 80%)  →  CRITICAL (burn rate ≥ 100%)
```

State transitions are the only trigger for alerts. A project already in CRITICAL state does not fire an alert on the next execution cycle — only the transition from WARNING to CRITICAL fires once.

**Burn rate formula:**
```
burn_rate = (hours_tracked / budget_hours) × 100
```

---

## Node Breakdown

**[GET] Fetch_Active_Tasks**
Pulls all active tasks from a configured ClickUp list. Extracts `Budget Hours` from the nested `custom_fields` array and raw `time_spent` data.

**[TRANSFORM] Normalize_Payload**
Converts `time_spent` from milliseconds to hours. Extracts budget from ClickUp's nested custom field structure. Outputs a flat, consistent JSON object per project for downstream processing.

**[GET] DB_Previous_States**
Queries PostgreSQL for the last known health state of every active project. This is the foundation of delta detection — without the previous state, the system cannot determine what changed.

**[CALCULATE] Delta_Engine**
The core logic node. For each project:
1. Calculates current burn rate
2. Derives new state from burn rate thresholds
3. Compares new state against previous state from DB
4. Sets `requires_alert = true` only if state has changed AND new state is not HEALTHY
5. Prevents duplicate alerts for projects already in a degraded state

**[UPSERT] PG_Save_State**
Persists new state using `INSERT ... ON CONFLICT DO UPDATE`. Guarantees exactly one state record per project at all times. No duplicate rows, no orphaned records.

**[GATE] Check_Alert_Flag**
IF node that routes only projects where `requires_alert = true` to the alert pathway. All other projects exit silently.

**[POST] Alert_Manager**
Fires a structured webhook payload containing: task name, previous state, new state, burn rate percentage, hours tracked, and budget hours. Downstream alert consumers (Slack, email, PagerDuty) receive a consistent schema.

**[INSERT] Log_API_Failure (DLQ)**
Captures ClickUp API failures with full error context in PostgreSQL. Failed executions are logged with timestamp and error body for debugging and manual replay.

---

## Engineering Highlights

**Delta-gate alerting pattern**
Alerts fire on state transitions, not on every execution cycle. This eliminates the most common failure mode of threshold-based alerting — alert fatigue from repeated notifications about a known problem.

**PostgreSQL state machine**
State is owned by the database, not by the workflow. The pipeline can be restarted, redeployed, or interrupted at any point without losing state. On resume, the delta engine reads the last persisted state and continues correctly.

**Idempotent writes**
The `INSERT ... ON CONFLICT DO UPDATE` pattern guarantees that running the pipeline multiple times on the same data produces identical state — no duplicate records, no state drift.

**Dead Letter Queue**
ClickUp API failures are captured and logged before any downstream processing. The pipeline does not silently swallow errors. Failed task fetches can be replayed from the DLQ table without re-running a full cycle.

---

## Failure Handling

| Failure | Behaviour |
|---|---|
| ClickUp API failure | Error logged to DLQ; pipeline exits cleanly; state not updated |
| PostgreSQL write failure | n8n retries with backoff; alert not fired until state is confirmed persisted |
| Webhook alert failure | Alert delivery logged; state already saved; alert can be retried independently |
| Partial task fetch | Only fetched tasks are processed; missing tasks retain their last known state |

---

## Setup

**Prerequisites**
- n8n instance (self-hosted)
- ClickUp API key and list ID
- PostgreSQL database
- Webhook endpoint for alert delivery (Slack, email, or any HTTP consumer)

**Steps**
```bash
# 1. Import workflow
# Import Financial_Profitability_Guardrail.json into n8n

# 2. Run database schema
# Execute /docs/schema.sql in your PostgreSQL instance
# Creates: project_states table, dql_api_failures table

# 3. Connect credentials
# ClickUp API key, PostgreSQL connection string

# 4. Configure ClickUp
# Set your list ID and the custom field name for Budget Hours

# 5. Configure alert webhook
# Set your webhook URL in the Alert_Manager node

# 6. Set trigger schedule
# Default: every 15 minutes. Adjust in the Trigger node.

# 7. Activate
```

**.env.example**
```
CLICKUP_API_KEY=your_key_here
CLICKUP_LIST_ID=your_list_id
PG_CONNECTION_STRING=postgresql://user:pass@host:5432/db
ALERT_WEBHOOK_URL=https://your-alert-endpoint.com/webhook
```

**Database schema**
```sql
CREATE TABLE project_states (
  task_id         TEXT PRIMARY KEY,
  task_name       TEXT NOT NULL,
  health_state    TEXT NOT NULL,       -- HEALTHY | WARNING | CRITICAL
  burn_rate       NUMERIC(5,2),
  hours_tracked   NUMERIC(8,2),
  budget_hours    NUMERIC(8,2),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE dql_api_failures (
  id              SERIAL PRIMARY KEY,
  error_body      JSONB,
  failed_at       TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Stack

- **Orchestration:** n8n
- **Task Data:** ClickUp API
- **State Machine:** PostgreSQL
- **Alerting:** Webhook (Slack / email / PagerDuty compatible)

---

## License

MIT
