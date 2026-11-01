---
name: activity-heartbeat-polling-pattern
description: Replace stale-timeout-based polling with activity heartbeat for long-running AI API tasks. Check if backend is still calling APIs, not if progress changed.
---

# Activity Heartbeat Polling Pattern

Replace "stale timeout" (progress-change-based) with activity-heartbeat-based monitoring for long-running background tasks.

## The Problem

Old logic:
```
Poll for progress → if no progress change in N seconds → show "stalled" error
```

This is **wrong** when the backend makes long-running API calls (DeepSeek, OpenAI, Kling, etc.).
During an API call (60-120s), progress naturally doesn't change — this is normal, not an error.

## The Fix: Activity Heartbeat

**Backend**: Write `last_activity_at` timestamp right before each external API call.
**Frontend**: Check `last_activity_at` instead of progress change gap.

```
Poll → progress didn't change for 60s → check last_activity_at
  → if now - last_activity_at < apiTimeoutMs (300s) → backend is alive, keep waiting
  → if now - last_activity_at > apiTimeoutMs → backend likely dead, show error
```

## Backend Implementation

```python
# 1. Add last_activity_at to progress file writer
def _write_step_progress(dir, status, round=0, message="", result=None):
    data = {
        "status": status,
        "message": message,
        "timestamp": datetime.now().isoformat(),
        "last_activity_at": datetime.now().isoformat(),
    }
    ...

# 2. Lightweight activity marker (call before every AI API call)
def _mark_activity(dir):
    path = os.path.join(dir, "step_progress.json")
    now = datetime.now().isoformat()
    if os.path.exists(path):
        with open(path, "r") as f: data = json.load(f)
    else:
        data = {"status": "running", "round": 0, "message": ""}
    data["last_activity_at"] = now
    with open(path, "w") as f: json.dump(data, f, ensure_ascii=False)

# 3. In background task, mark before every AI call
_mark_activity(l03_dir)
result = _call_ai(system_prompt, user_prompt)
```

## Frontend Implementation

Add activity tracking in the polling handler:

```javascript
// Track lastActivityAt from progress response
if (body.last_activity_at) {
    state.lastActivityAt = new Date(body.last_activity_at).getTime();
}

// Replace stale timeout with activity check
if (getStaleElapsed() > cfg.staleTimeoutMs) {
    if (state.lastActivityAt) {
        var activityAge = Date.now() - state.lastActivityAt;
        if (activityAge < cfg.apiTimeoutMs) {
            // Backend still calling API, keep waiting
            state.lastChangeAt = Date.now();
        } else {
            // Activity heartbeat expired, backend dead
            onStale();
        }
    } else {
        // No heartbeat data, fall back to old behavior
        onStale();
    }
}
```

## Key Parameters

| Param | Value | Why |
|---|---|---|
| `apiTimeoutMs` | 300000 (5 min) | Matches backend HTTP timeout |
| `staleTimeoutMs` | 60000 (1 min) | Triggers activity check; backward compat |
| Heartbeat freq | Before every `_call_ai()` | Ensures frontend knows backend is alive |

## Files to Modify

- `app/routes/l03.py` — `_write_step_progress()` + `_mark_activity()` + markers before all `_call_ai()` sites
- `templates/components/_loading.html` — `startPollingWithStaleness`: add `apiTimeoutMs`, replace stale check
- `static/js/step-task-manager.js` — Use `startPollingWithStaleness` with `apiTimeoutMs`

## Pitfalls

1. **Mark activity BEFORE API call**, not after — if it hangs, timestamp should be from when it started
2. **Match `apiTimeoutMs` to actual HTTP timeout** — if `requests.post(timeout=120)`, set >= 120000 + buffer
3. **Keep `staleTimeoutMs`** — it triggers the activity check and handles steps without heartbeat
4. **Backward compatible** — steps without `last_activity_at` fall back to old stale timeout behavior
