<div align="center">

# 🔌 KnowDB v0.4.3 — *MCP over stdio*

### *Point Claude Code, Claude Desktop, or any MCP client straight at your inbox.*

</div>

---

## What's new

`knowdb mcp-stdio` — KnowDB now speaks the **Model Context Protocol over
stdin/stdout**, so any MCP client can use it as a tool server:

```json
{
  "mcpServers": {
    "knowdb": { "command": "knowdb", "args": ["mcp-stdio"] }
  }
}
```

(or `claude mcp add knowdb -- knowdb mcp-stdio`)

Ask Claude *"what did Priya commit to last week?"* and it queries your local
substrate and answers with citations — your mail never leaves the machine.

### How it works

`mcp-stdio` is a **bridge**, not a second daemon. It implements the MCP
handshake (`initialize` / `tools/list` / `tools/call`) on stdio and forwards
each call to the **already-running daemon's** Unix socket — so there's exactly
one process holding the substrate (the single-lock invariant stays intact). If
the daemon isn't running it makes a best-effort `systemctl --user start knowdb`
and waits for the socket.

### Exposed tools (read-only)

An external assistant gets **query and discovery, never mutation** — no sending,
merging, or deleting through the bridge:

`search` · `get_thread` · `get_entity` · `list_entities` · `entity_timeline` ·
`find_related` · `list_commitments` · `list_accounts` ·
`list_upcoming_birthdays` · `audit_recent` · `get_storage_stats`

---

## Notes

- Requires the binary on `PATH` — shipped in v0.4.2 (`/usr/bin/knowdb`).
- The daemon's own Unix-socket protocol (used by the KnowDB app) is unchanged.
- **No schema migration.** Drop-in upgrade from v0.4.x.
