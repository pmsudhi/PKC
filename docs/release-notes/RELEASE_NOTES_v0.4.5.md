<div align="center">

# 🧵 KnowDB v0.4.5 — *Whole conversations, knowledge-first retrieval*

### *Sent and received, reunited — and retrieval that asks for knowledge, not folders.*

</div>

---

## The bug: shattered Microsoft conversations

Microsoft Graph **rejects** the RFC `References` / `In-Reply-To` headers, and the
threading logic only linked replies via those headers (or Gmail's `X-GM-THRID`).
So every Graph message — including every reply you **sent** — became its **own
single-message thread**. A conversation of 39 messages was stored as 39
disconnected threads. Asking "what's the latest with X" returned one message with
no context: the back-and-forth was gone.

Graph's own `conversationId` (the real thread key) was captured but never used.

### The fix (A)

- **Live:** thread Graph mail by `conversationId` — every message sharing a
  conversation (inbox + sent + every reply) now lands on one thread.
- **Historical heal:** a new offline command, **`knowdb rethread`**, merges
  existing Graph mail into real conversations by `conversationId`, then rebuilds
  the substrate (EXPORT→IMPORT) so the index is rebuilt clean. Run with the
  daemon stopped (it holds the pid lock, like `knowdb compact`). On the author's
  mailbox it merged ~650 shattered conversations (one 6-singleton "Pricing
  Conformation" became a single 39-message thread), all rows preserved.

  > **Why offline + rebuild:** a bulk in-place `UPDATE events SET thread_id`
  > corrupts the DuckDB PRIMARY index (→ dup-key abort on the next append). The
  > EXPORT→IMPORT rebuild that follows the re-thread is mandatory — it reads the
  > corrected data and builds fresh, clean indexes.

## The shift: retrieval is knowledge-centric, not mailbox-centric (B)

The storage was always source-agnostic; the *retrieval* still spoke
inbox/sent/folder. An assistant almost never wants "the inbox" — it wants the
**conversation** and the **person**. So:

- **`search` spans everything by default.** New `folder: "all"` (both directions,
  every account) is the default for MCP clients. Folder is now an optional
  *filter*, not the entry point. Sent-only conversations are no longer invisible.
- **`get_thread` labels each message `direction`** (sent / received / draft) so a
  thread reads as a real exchange.
- **`communications_with(email)`** — a new knowledge-first tool returning a
  person's whole conversation history, both directions, *participant-exact* (it
  does not depend on keyword search, which is still lexical until Phase 2).

The MCP bridge now exposes 17 read-only tools.

---

## Honest limits

- **Search is still lexical** (keyword/BM25). Multi-word queries can miss; use
  `communications_with` or the `participant_email` filter for reliable
  person-centric retrieval. Semantic/hybrid search is Phase 2.
- **A fused mail+calendar timeline** isn't a single tool yet — use recent-
  conversation `search` + `list_events`. Fast-follow.
- **Commitments** remain empty until extraction (Phase 3) lands.

No schema migration. The re-thread runs automatically on first start and is a
no-op once applied.
