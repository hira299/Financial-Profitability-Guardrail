# 1. 💰 Financial Profitability Guardrail

> **File:** `Financial_Profitability_Guardrail.json`  
> **Stack:** n8n · ClickUp API · PostgreSQL · Webhook Alerts  
> **Loom:** https://www.loom.com/share/679eb54f26de4ab98d895c8f9c7f493b

## The Problem
Project budget overruns were invisible until it was too late. There was no real-time visibility into which projects were burning through hours faster than allocated, and no automated alerting when thresholds were crossed.

## The Solution
A fault-tolerant budget monitoring system using a **PostgreSQL State Machine** that tracks every atomic budget change, detects state transitions, and fires alerts only when a project's health status actually changes — eliminating alert fatigue from false positives.

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

## Node Breakdown

**`[GET] Fetch_Active_Tasks`**  
Pulls all active tasks from a specific ClickUp list including custom fields (Budget Hours) and time tracking data.

**`[TRANSFORM] Normalize_Payload`**  
Extracts Budget Hours from ClickUp's nested custom_fields array. Converts time_spent from milliseconds to hours. Outputs a clean, consistent JSON structure for downstream processing.

**`[GET] DB_Previous_States`**  
Queries PostgreSQL for the last known health state of every project. This is what enables delta detection — the system knows what changed, not just what is.

**`[CALCULATE] Delta_Engine`**  
The core logic engine:
- Calculates burn rate percentage (tracked / budget × 100)
- Applies state machine logic: `HEALTHY` → `WARNING` (≥80%) → `CRITICAL` (≥100%)
- Compares new state against previous state from DB
- Sets `requires_alert = true` ONLY if state has changed AND is not HEALTHY
- Eliminates duplicate alerts for already-known bad states

**`[UPSERT] PG_Save_State`**  
Persists the new state using PostgreSQL `INSERT ... ON CONFLICT DO UPDATE` — ensuring every project always has exactly one current state record with updated timestamp.

**`[GATE] Check_Alert_Flag`**  
IF node that routes only state-changed items to the alert pathway.

**`[POST] Alert_Manager`**  
Fires a structured webhook alert with task name, new state, burn percentage, and hours tracked vs budgeted.

**`[INSERT] Log_API_Failure`**  
Dead Letter Queue — catches ClickUp API failures and logs them to PostgreSQL for debugging and replay. Prevents silent pipeline failures.

## Key Engineering Decisions
- **Delta Gate pattern** — alerts only on state transitions, not on every execution cycle
- **DLQ on API failure** — ClickUp errors are logged, not silently dropped
- **PostgreSQL UPSERT** — guarantees exactly one state record per project, no duplicates
- **Millisecond conversion** — ClickUp returns time in ms, normalized to hours for human-readable output
