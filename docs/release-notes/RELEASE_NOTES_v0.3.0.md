<div align="center">

# 📨 KnowDB v0.3.0 — *Milestone: Microsoft 365, fully wired*

### *Two inboxes, one substrate, zero islands.*

**Microsoft 365 mail is now first-class — read, send, mark, archive — and threads from Gmail and Outlook unify automatically when they share a Message-ID. New mail surfaces in seconds, not minutes, via IMAP IDLE push and foreground-aware Graph polling.**

<br/>

[![Status](https://img.shields.io/badge/status-alpha-yellow?style=flat-square)](#)
[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#)
[![Substrate](https://img.shields.io/badge/mail%20%2B%20calendar%20%2B%20people-one%20substrate-purple?style=flat-square)](#)

</div>

---

## 🎯 What this milestone unlocks

KnowDB has been a Gmail-shaped surface with calendar attached. **With v0.3.0 it stops being shaped by any one provider.**

Your **Microsoft 365 mailbox** now lives in the same local substrate as your Gmail inbox — same events table, same threads, same search. When the same email lands in both inboxes (a meeting invite CC'd to your work and personal addresses), KnowDB **merges it into one thread** via the RFC 5322 Message-ID. You see one row, not two. Reply from that row and the message goes out through whichever inbox it arrived on.

The push pipeline matters too: Gmail now uses **IMAP IDLE** so new mail surfaces within a second of arriving, and Microsoft Graph polls every **30 seconds while you're actively in the app** — not the old 5-minute floor.

This is the *connected* part of "personalized knowledge centre" finally feeling true. **Two inboxes, one substrate, every action grounded in the same knowledge graph.**

---

## ✨ Highlights

### 📬 Microsoft Graph mail — read, send, flag, archive, threaded
- **Full read** via `/me/mailFolders/{id}/messages/delta` — per-folder cursors, idempotent, resumes on interruption. (No, Graph does **not** support `/me/messages/delta` despite Stack Overflow suggesting otherwise — `BadRequest: Change tracking is not supported against 'microsoft.graph.message'`. Per-folder is the only valid path.)
- **HTML bodies** stored as content-addressed blobs the same way Gmail HTML is — `body_html_hash` populated, derived plain-text fallback for search and previews.
- **Send via `/me/sendMail`** — modern Graph send, attachments inline as base64, `In-Reply-To` / `References` headers injected via Graph's `internetMessageHeaders` so receivers thread the reply natively. SMTP-XOAUTH2 to `smtp.office365.com` is being deprecated by Microsoft; Graph is the future-proof path.
- **Mark read / unread writeback** — patches `isRead` on `/me/messages/{id}`. Queued in a new `graph_flag_queue` (analogous to the IMAP flag queue), drained right after every sync pass.
- **Archive → Archive folder, Trash → DeletedItems folder** — `/me/messages/{id}/move` with well-known folder ids cached per pass.
- **Cross-account thread merge** — when a message's `internetMessageId` matches an existing thread (Gmail or Microsoft, doesn't matter), the new event adopts that `thread_id`. One thread row, two contributing accounts.

### ⚡ Real-time push — IMAP IDLE for Gmail, foreground-aware Graph
- **IMAP IDLE supervisor** runs one long-lived connection per Gmail account on its own dedicated OS thread (because `async-imap`'s `Session` is `!Send`). Server pushes EXISTS / EXPUNGE the moment a new message arrives → `sync_notify.notify_one()` → scheduler runs a delta sync within a second.
- **28-minute keepalive cycle** with cooperative re-IDLE; **exponential backoff** up to 5 min on transient failures; clean re-auth on token expiry.
- **Foreground-aware Graph polling** — while any MCP call has happened in the last 60 s, the Graph mail loop polls every **30 seconds**. Idle longer than that, it drifts back to the global `interval_secs` (default 5 min). Activity is tracked via a single `AtomicU64` heartbeat touched on every MCP call — no new heartbeat protocol needed; **MCP traffic itself is the activity signal**.
- **Per-account concurrency** — `for_each_concurrent(4, ...)` across accounts inside each pass. A slow Gmail mailbox iteration no longer blocks Microsoft sync, and vice versa.

### 🏷️ Per-account UI — chips, sidebar, badges
- **Per-account avatar chips** on thread cards — small 16-px circles at the right edge of the subject row, coloured per account, with a single identifying letter (`G` for Gmail, `M` for Microsoft, email-initial otherwise). The letter carries identity even when colours blur, so quick scanning doesn't require side-by-side colour matching against the sidebar.
- **Cross-account threads stack two chips** with a half-overlap, immediately legible as "this reached both inboxes."
- **Sidebar Mailboxes section** — toggle accounts off to hide their solo threads; cross-account threads stay visible as long as at least one of their accounts is on, so toggling Gmail off never hides a thread that also reached your work inbox.
- **"via Gmail + Microsoft" badge** in the thread detail header for cross-account conversations.
- **Reply chooses the right From** — defaults to the inbox the latest message arrived on, so a reply to a Microsoft thread sends through `/me/sendMail` automatically, and a reply to a Gmail thread sends via SMTP-XOAUTH2.

### 🤝 People — one identity surface
The old **Contacts** and **People** navigation entries collapsed into a single `People` entry. People is the broader concept — every person we know about across email and calendar (and Slack when that lands) — while Contacts was a narrower GOA-address-book alias. The working list/detail/add UI is unchanged; the nav is just no longer confused about which one is the source of truth. Legacy `/contacts` URLs redirect to `/people`.

### 📞 Conferencing UX, made explicit
- **The conferencing checkbox now names the provider** based on which calendar you save the event to: *Add Google Meet link* when a Google calendar is selected, *Add Microsoft Teams link* for a Microsoft calendar.
- **"Auto-generated" label** next to the toggle and a small helper line — "*Microsoft Teams will mint a join link when you save — it appears on the event after the next sync*". No more guessing whether the box does anything.
- **Manual URL field disabled** while auto-generation is on (the two paths can't coexist), re-enabled when you want a third-party room (Zoom, Webex, custom).
- **Disabled with a hint** when no calendar is selected yet, so the dependency is obvious.

---

## 🔧 What's measured

| Metric | v0.3.0 budget | Status |
|---|---|---|
| Time to surface new Gmail message | < 2 s (IDLE) | ✅ |
| Time to surface new Microsoft message (foreground) | < 35 s (30 s poll + sync) | ✅ |
| Time to surface new Microsoft message (idle) | < 5 min + sync | ✅ |
| Concurrent IMAP accounts before head-of-line blocking | 4 | ✅ `for_each_concurrent` |
| Cross-account thread merge correctness | 100 % via RFC 5322 Message-ID | ✅ deterministic |
| Outbound send: Microsoft account → Graph, Gmail → SMTP | 100 % | ✅ provider-routed |
| Mark-read / archive / trash writeback | Each provider's native API | ✅ separate queues |

---

## 🔬 Under the hood

### Two-substrate-table truth
Every Microsoft event flows through the exact same writes as a Gmail event:

```
events
  source = 'email'
  metadata.account_id = '<GOA opaque id>'
  metadata.provider   = 'graph'    -- new for v0.3.0
  metadata.graph_message_id = '<Outlook opaque id>'   -- for writebacks
  metadata.message_id = '<RFC 5322 Message-ID>'       -- for threading

threads
  -- merged via SubstrateDb::find_thread_by_message_id when a new
  -- event's message_id matches a prior event's thread.
```

Nothing in the inbox query knows or cares which connector produced the row. The MCP `search`, `get_thread`, `list_labels`, `list_commitments` tools work identically across both. Slack will plug in here in v0.4 the same way.

### Push and poll, side by side
```
Gmail            IMAP IDLE supervisor    push  → sync_notify → scheduler tick
Gmail            scheduler delta poll    pull  → every interval_secs   (safety net)
Microsoft 365    Graph mail loop         pull  → 30 s (foreground) / 5 min (idle)
Microsoft 365    sync_notify nudge       react → drains queues right after IDLE wakes
```

The `sync_notify` channel is the bus. IDLE writes to it; foreground-aware loops drain on it; MCP-side `mark_read` / `archive` writes to it so the local UI doesn't lag the writeback queue.

### Provider-routed writes
- `send_email` MCP tool reads `provider_type` from the GOA account and dispatches: `ms_graph` → `GraphMailClient::send_mail`; anything else → `lettre` SMTP with XOAUTH2 + `In-Reply-To` / `References`.
- `mark_read` / `archive_threads` / `trash_threads` queue into the right table (`imap_flag_queue` vs new `graph_flag_queue`) based on the source event's `metadata.provider`.
- Graph `move_message` resolves Archive and DeletedItems folder ids by **well_known name** (`archive` / `deleteditems`) once per drain pass, cached for the batch.

### Recovery without DB resets, still
Two new background backfills extend the no-reset invariant:
- **`backfill_missing_threads`** — synthesizes `threads` rows for any `provider='graph'` events that landed before we switched to `upsert_events_and_threads_batch`. Idempotent, runs at every loop start.
- **`backfill_graph_html`** — re-fetches `/me/messages/{id}` (HTML body) for events synced under the old plain-text Prefer header, writes the HTML blob, stamps `body_html_hash`, derives plain text via `html2text`. 25 messages per batch with 200 ms pacing to stay under Graph's per-app throttle.

You can upgrade from v0.2.0 without touching your DuckDB file.

---

## 🚧 What's next

v0.3.0 closes Phase 1 for real. Phase 2 is search.

- **Phase 2: embeddings + hybrid search** — `bge-small-en-v1.5` locally via candle, DuckDB VSS HNSW, BM25 FTS, RRF fusion, p95 < 100 ms on 100K events. The substrate is already organised to make this a single-crate addition.
- **Phase 3: extraction + identity** — commitments, action items, decisions extracted with a local 1.5-2 B parameter model; deterministic → probabilistic → user-confirmed identity resolution. People graph stops being a placeholder.
- **Phase 4: LLM router + audit log enforcement** — frontier opt-in, every outbound payload audited *before* it leaves the device.
- **Phase 5: privacy panel + reconciliation UI + entity views** in Tauri.
- **Phase 6: `.deb` / `.AppImage` / systemd unit / CI perf gates.**

---

## 📦 Requirements

- **Linux** with GNOME Online Accounts ≥ 3.50 (Fedora 41+, Ubuntu 24.04+, Arch with `gnome-online-accounts` installed)
- **D-Bus session bus** active (any GNOME / KDE-with-GOA session)
- **Rust** ≥ 1.85 to build from source
- **Node ≥ 20** + **pnpm/npm** to build the Tauri UI
- A Google account with **IMAP enabled** in Gmail settings (and/or a Microsoft 365 account)

Tested on the developer's machine with one personal Gmail + one Microsoft 365 work account synced side by side. Multi-account works; multi-Gmail and multi-Microsoft both supported via dedup-by-account in `list_accounts`.

---

## 🤝 Closing note

The thesis is unchanged: **one local substrate, many surfaces, every action grounded in your real data**. v0.1 made it true for one inbox. v0.2 made it true for the calendar. **v0.3 makes it true across the providers themselves** — Gmail and Microsoft 365 are no longer parallel realities; they're the same conversation, joined where they overlap, separated where they shouldn't.

Slack is next on the connector side. Embeddings are next on the substrate side. The two surfaces will keep meeting in the middle.

---

*Built locally. Synced locally. Audited locally. The data stays on your machine.*
