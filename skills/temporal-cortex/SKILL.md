---
name: temporal-cortex
description: |-
  Schedule meetings, check availability, and manage calendars across Google, Outlook, and CalDAV. Routes to focused sub-skills for datetime resolution and calendar scheduling. Use this skill when the user mentions scheduling, calendars, booking meetings, checking availability, or managing time across calendar providers.
license: MIT
---

# Temporal Cortex — Calendar Scheduling Router

This is the router skill for [Temporal Cortex](https://github.com/temporal-cortex/skills) calendar operations. It routes your task to the right sub-skill based on intent.

## Sub-Skills

| Sub-Skill | When to Use | Tools |
|-----------|------------|-------|
| **temporal-cortex-datetime** | Time resolution, timezone conversion, duration math. Zero-setup — no credentials needed. | 5 tools |
| **temporal-cortex-scheduling** | List calendars, events, free slots, availability, RRULE expansion, and booking. Requires OAuth credentials. | 8 tools |

## Routing Table

| User Intent | Route To |
|------------|----------|
| "What time is it?", "Convert timezone", "How long until..." | **temporal-cortex-datetime** |
| "Show my calendar", "Find free time", "Check availability" | **temporal-cortex-scheduling** |
| "Book a meeting", "Schedule an appointment" | **temporal-cortex-scheduling** |
| "Schedule a meeting next Tuesday at 2pm" (full workflow) | **temporal-cortex-datetime** → **temporal-cortex-scheduling** |

## Core Workflow

Every calendar interaction follows this 5-step pattern:

```
1. Discover  →  list_calendars                (know which calendars are available)
2. Orient    →  get_temporal_context           (know the current time)
3. Resolve   →  resolve_datetime              (turn human language into timestamps)
4. Query     →  list_events / find_free_slots / get_availability
5. Act       →  check_availability → book_slot (verify then book)
```

**Always start with step 1** when calendars are unknown. Never assume the current time. Never skip the conflict check before booking.

## All 12 Tools (5 Layers)

| Layer | Tools | Sub-Skill |
|-------|-------|-----------|
| 0 — Discovery | `list_calendars` | scheduling |
| 1 — Temporal Context | `get_temporal_context`, `resolve_datetime`, `convert_timezone`, `compute_duration`, `adjust_timestamp` | datetime |
| 2 — Calendar Ops | `list_events`, `find_free_slots`, `expand_rrule`, `check_availability` | scheduling |
| 3 — Availability | `get_availability` | scheduling |
| 4 — Booking | `book_slot` | scheduling |

## MCP Server Connection

All sub-skills share the same MCP server: [@temporal-cortex/cortex-mcp](https://www.npmjs.com/package/@temporal-cortex/cortex-mcp).

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

Layer 1 tools work immediately with zero configuration. Calendar tools require a one-time OAuth setup — run `npx @temporal-cortex/cortex-mcp auth google` (or `outlook` / `caldav`).

For full documentation, presets, and setup scripts, see the [skills repository](https://github.com/temporal-cortex/skills).
