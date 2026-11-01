---
name: backend-time-format-avoid-js-timezone
description: Send pre-formatted Beijing time strings from backend instead of ISO format to avoid JavaScript Date() timezone parsing issues.
---

# Backend Time Format to Avoid JS Timezone Bugs

## Problem
When backend APIs return timestamps using `.isoformat()` (Python), they include timezone info like:
```
"2026-06-29T16:10:38.373998+08:00"
```

JavaScript `new Date(isoStr)` correctly parses this, but then `.getHours()` returns the **browser's local time**, not the original time. This means:
- A user in UTC+8 sees the correct time
- A user in UTC sees `08:10` instead of `16:10`
- A user in UTC+2 sees `10:10` instead of `16:10`

Even if all users are in the same timezone, this is fragile — any admin accessing from a different timezone or via a VPN sees wrong times.

## Fix
Format timestamps **on the backend** (which runs in the target timezone) before sending to the frontend:

### Python (server in Asia/Shanghai):
```python
# Before (problematic):
"created_at": l.created_at.isoformat() if l.created_at else None,
# → "2026-06-29T16:10:38.373998+08:00"

# After (safe):
"created_at": l.created_at.strftime("%Y-%m-%d %H:%M") if l.created_at else None,
# → "2026-06-29 16:10"
```

### Frontend JS:
```javascript
// No need for new Date() parsing anymore
function formatTime(str) { return str || ''; }
// Just display the string directly
```

## Prerequisites
- Server timezone is set to the target timezone (e.g., `Asia/Shanghai`)
- PostgreSQL timezone matches (check with `SHOW timezone;`)
- All datetime objects in the database are stored in the same timezone

## Pitfalls
- Only do this for **display purposes** (not for calculations or sorting — keep ISO format for those)
- If you need the time for sorting: ISO format is better because it's lexicographically sortable
- `strftime("%Y-%m-%d %H:%M")` drops seconds and timezone — intentional for clean display
- This assumes the server timezone is correct and consistent
