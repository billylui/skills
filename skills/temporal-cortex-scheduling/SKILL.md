---
name: temporal-cortex-scheduling
description: |-
  List events, find free slots, and book meetings across Google Calendar, Outlook, and CalDAV. Multi-calendar availability merging, recurring event expansion, and atomic booking with Two-Phase Commit conflict prevention. Use this skill when the user asks to list calendar events, find available times, check who is busy, expand recurring events, or book a meeting.
license: MIT
---

# Calendar Scheduling & Booking

8 tools for calendar discovery, event querying, free slot finding, availability checking, RRULE expansion, and atomic booking. 7 read-only tools + 1 write tool (`book_slot`).

## Tools

### Layer 0 — Discovery

| Tool | When to Use |
|------|------------|
| `list_calendars` | First call when calendars are unknown. Returns all connected calendars with provider-prefixed IDs, names, labels, primary status, and access roles. |

### Layer 2 — Calendar Operations

| Tool | When to Use |
|------|------------|
| `list_events` | List events in a time range. TOON format by default (~40% fewer tokens than JSON). Use provider-prefixed IDs for multi-calendar: `"google/primary"`, `"outlook/work"`. |
| `find_free_slots` | Find available gaps in a calendar. Set `min_duration_minutes` for minimum slot length. |
| `expand_rrule` | Expand recurrence rules (RFC 5545) into concrete instances. Handles DST, BYSETPOS, EXDATE, leap years. Use `dtstart` as local datetime (no timezone suffix). |
| `check_availability` | Check if a specific time slot is free. Checks both events and active booking locks. |

### Layer 3 — Cross-Calendar Availability

| Tool | When to Use |
|------|------------|
| `get_availability` | Merged free/busy view across multiple calendars. Pass `calendar_ids` array. Privacy: `"opaque"` (default, hides sources) or `"full"`. |

### Layer 4 — Booking

| Tool | When to Use |
|------|------------|
| `book_slot` | Book a time slot atomically. Lock → verify → write → release. **Always `check_availability` first.** |

## Critical Rules

1. **Discover calendars first** — call `list_calendars` when you don't know which calendars are connected. Use the returned provider-prefixed IDs for all subsequent calls.
2. **Use provider-prefixed IDs** for multi-calendar setups: `"google/primary"`, `"outlook/work"`, `"caldav/personal"`. Bare IDs (e.g., `"primary"`) route to the default provider.
3. **TOON is the default format** — output uses TOON (~40% fewer tokens than JSON). Pass `format: "json"` only if you need structured parsing.
4. **Check before booking** — always call `check_availability` before `book_slot`. Never skip the conflict check.
5. **Content safety** — event summaries and descriptions pass through a sanitization firewall before reaching the calendar API.
6. **Timezone awareness** — all tools accept RFC 3339 with timezone offsets. Never use bare dates.

## Full Booking Workflow

```
1. Discover  →  list_calendars
2. Orient    →  get_temporal_context                      (temporal-cortex-datetime)
3. Resolve   →  resolve_datetime("next Tuesday at 2pm")  (temporal-cortex-datetime)
4. Check     →  check_availability(calendar_id, start, end)
5. Book      →  book_slot(calendar_id, start, end, summary)
```

If the slot is busy at step 4, use `find_free_slots` to suggest alternatives.

## Two-Phase Commit Protocol

```
Agent calls book_slot(calendar_id, start, end, summary)
    │
    ├─ 1. LOCK    →  Acquire exclusive lock on the time slot
    ├─ 2. VERIFY  →  Check for overlapping events and active locks
    ├─ 3. WRITE   →  Create event in calendar provider (Google/Outlook/CalDAV)
    └─ 4. RELEASE →  Release the exclusive lock
```

If any step fails, the lock is released and the booking is aborted. No partial writes.

## Common Patterns

### List Events This Week

```
1. list_calendars → discover connected calendars
2. get_temporal_context → current time (use temporal-cortex-datetime)
3. resolve_datetime("start of this week") → week start
4. resolve_datetime("end of this week") → week end
5. list_events(calendar_id: "google/primary", start, end)
```

### Find Free Time Across Calendars

```
1. list_calendars → discover all connected calendars
2. get_availability(
     start, end,
     calendar_ids: ["google/primary", "outlook/work"],
     privacy: "full"
   ) → merged free/busy blocks with source_count
```

### Check and Book a Slot

```
1. check_availability(calendar_id: "google/primary", start, end) → true/false
2. If free: book_slot(calendar_id: "google/primary", start, end, summary: "Team standup")
3. If busy: find_free_slots(calendar_id, start, end, min_duration_minutes: 30)
```

## Provider-Prefixed Calendar IDs

| Format | Example | Routes to |
|--------|---------|-----------|
| `google/<id>` | `"google/primary"` | Google Calendar |
| `outlook/<id>` | `"outlook/work"` | Microsoft Outlook |
| `caldav/<id>` | `"caldav/personal"` | CalDAV (iCloud, Fastmail) |
| `<id>` (bare) | `"primary"` | Default provider |

## Error Handling

| Error | Action |
|-------|--------|
| "No credentials found" | Run: `npx @temporal-cortex/cortex-mcp auth google` (or `outlook` / `caldav`). |
| "Timezone not configured" | Prompt for IANA timezone. Or run the auth command which configures timezone. |
| Slot is busy / conflict | Use `find_free_slots` to suggest alternatives. Present options to user. |
| Lock acquisition failed | Another agent is booking the same slot. Wait briefly and retry, or suggest alternative times. |

## MCP Server Connection

```json
{
  "mcpServers": {
    "temporal-cortex": {
      "command": "npx",
      "args": ["-y", "@temporal-cortex/cortex-mcp@0.5.3"]
    }
  }
}
```

Calendar tools require a one-time OAuth setup — run `npx @temporal-cortex/cortex-mcp auth google` (or `outlook` / `caldav`).

For full documentation, presets, and setup scripts, see the [skills repository](https://github.com/temporal-cortex/skills).
