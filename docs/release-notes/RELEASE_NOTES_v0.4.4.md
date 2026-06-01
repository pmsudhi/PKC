<div align="center">

# 📅 KnowDB v0.4.4 — *Calendar over MCP*

### *Your meetings are now visible to MCP clients.*

</div>

---

## Fix

The `knowdb mcp-stdio` bridge (v0.4.3) exposed an 11-tool **read-only** surface
that — by oversight — **omitted the calendar tools**. An MCP client asking
"what's on my calendar next week" got nothing from KnowDB and fell back to
whatever other connector it had, even though KnowDB holds the full schedule
across every linked account.

v0.4.4 adds five read-only calendar / meeting tools to the bridge:

- **`list_events`** — events in a date range (the "next week" query)
- **`search_events`** — calendar search by title / location / attendees
- **`get_event`** — one event in full (attendees, times, conferencing link)
- **`people_on_calendar_today`** — who you're meeting today
- **`contact_upcoming_meetings`** — upcoming meetings with a given person

The exposed surface stays **read-only and account-agnostic** — events from every
linked account are returned together; `create/update/delete_event` remain
out of scope for external clients.

The bridge now exposes 16 tools. No daemon protocol change, no schema migration.

---

## Note

`list_commitments` still returns empty — commitment **extraction** isn't built
yet (the rest of Phase 3). That's a roadmap item, not a regression.
