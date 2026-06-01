<div align="center">

# 🧭 KnowDB v0.4.6 — *KnowDB tells the model what it is*

### *Server-level instructions so an AI client knows to reach for your knowledge base.*

</div>

---

## Why

Tool names alone don't tell an MCP client (Claude, Cursor, …) **what KnowDB is
or when to use it**. MCP has fields for exactly that — KnowDB was using none of
them. So a client could connect and not realise these 17 tools are the gateway
to the user's own email, calendar, and people.

## What changed (all in the `knowdb mcp-stdio` bridge)

- **`initialize.instructions`** — a server-level briefing the client feeds to the
  model at connect time. It states plainly: KnowDB is the user's local, unified
  email/calendar/contacts knowledge base across **all** accounts; reach for it
  for any question about the user's own world (messages, what someone said,
  meetings, people, "what's the latest with X"); prefer it over guessing; cite
  sources, never invent; a conversation is the unit (`search` → `get_thread`),
  `communications_with` for a person, `list_events` for the calendar; account is
  metadata, not a boundary; tools are read-only.
- **`serverInfo.title`** — "KnowDB — your personal knowledge base (email,
  calendar, people)" instead of the bare slug.
- **`readOnlyHint`** annotation on all 17 tools — signals they're safe,
  side-effect-free, so the model calls them freely to gather context.

No new tools, no protocol change, no schema migration. Reinstall and **restart
your MCP client** to pick up the new handshake.

## Next

A follow-up will add **MCP prompts** — ready-made templates ("what did I commit
to this week", "brief me before my next meeting") that surface as slash-commands
in MCP clients.
