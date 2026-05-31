<div align="center">

# 🛠️ KnowDB v0.3.1 — *Stability after Microsoft 365 launch*

### *The boring release that should have come with v0.3.0.*

**The reader pool. The compose state machine. The 51-test safety net. Microsoft send through `/reply`. Every regression of the past week, fixed at the architectural layer where it actually belongs.**

<br/>

[![Status](https://img.shields.io/badge/status-alpha-yellow?style=flat-square)](#)
[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![Tests](https://img.shields.io/badge/tests-51%20passing-brightgreen?style=flat-square)](#-the-safety-net-we-built)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#)

</div>

---

## 🎯 What this milestone fixes

v0.3.0 shipped Microsoft 365 as a first-class citizen — and exposed three architectural cracks the original Gmail-only design hadn't been stressed against. Each one produced a different surface symptom:

- "Clicking an email takes 25-30 seconds to open."
- "Inbox shows mails in random order."
- "Send fails when I reply to a Microsoft thread."
- "The compose window won't close (fixed three times, came back twice)."
- "The send-error toast just sits there with no way to dismiss it."

The same root causes drive most of these: **a single `Arc<Mutex<SubstrateDb>>` serializing every read behind every write**, **a compose lifecycle with eight implicit states** (only four valid), and **Microsoft Graph's `sendMail` rejecting standard RFC 5322 headers** through `internetMessageHeaders`. v0.3.1 fixes each at the layer where it lives, and lands a **51-test safety net** so they can't regress quietly again.

---

## 🏗️ Architecture changes

### Per-connection substrate pool

Before: every worker (IMAP normalize, Graph mail sync, html-backfill, archive seeder, calendar sync, …) plus every MCP request grabbed the **same** `Arc<Mutex<SubstrateDb>>`. A 10-second writer batch meant 10 seconds of blocked reads — exactly the "click → body opens half a minute later" feel.

After: one writer + a **pool of four cloned reader connections**. DuckDB's MVCC engine lets readers see committed snapshots without locking the writer. The MCP server classifies each tool as read-or-write at dispatch and routes to the right handle.

Measured on a 75 K-event substrate:

| | Before v0.3.1 | After v0.3.1 |
|---|---|---|
| `get_thread` while writer busy | several seconds | **10 ms p95** |
| 20 concurrent reads | ~9 s queued | **2.1 s wall** |
| Read + write concurrent | reads block | **independent, both finish in normal time** |
| `search(limit=50)` while syncing | seconds | **~500 ms** (FTS cost, no contention) |

### CHECKPOINT after every migration

The DuckDB WAL replay path doesn't safely reconstruct certain DDL operations (`DROP TABLE` + `CREATE TABLE` + `RENAME` — what migration 0010 does to remove the mentions FK). If anything crashed before the next periodic checkpoint folded those into the main file, startup would explode with:

> `INTERNAL Error: Failure while replaying WAL file: Calling DatabaseManager::GetDefaultDatabase with no default database set`

The first integration test we wrote uncovered this immediately. Fix: force a `CHECKPOINT` immediately after every applied migration. One fsync on first run, no impact thereafter; the WAL never has to redo migration DDL.

### Compose lifecycle as an explicit state machine

The window had three independent boolean flags (`isOpen`, `isMinimized`, plus a local `sending` React state) — eight implicit states, only four valid. Every regression here was a transition into one of the invalid combos.

Replaced with a five-state machine: `closed → open → sending → closed`, plus `open ↔ minimized` and `sending → error → {open, closed}`. Single `close()` action is the only path back to closed; no more `closeCompose` vs `closeWindow` ambiguity (both methods existed, with subtly different draft-reset semantics).

Bonus removal: the **`composeWindows.store.ts` multi-window store was never referenced** but persisted to localStorage on every change. Pure dead weight, deleted.

### Send-error inline banner

Replaces the auto-clearing `toast.error(..., { duration: 12_000 })` with a persistent inline red banner above the action row, with an explicit **Dismiss** button. Toasts auto-clearing on send failure was easy to miss and gave no recovery path — the user kept staring at a frozen Send button without knowing why. The banner stays until the user dismisses, retries (which clears it via `sendStart`), or closes the window.

---

## 📨 Microsoft Graph send: the right endpoint

Outbound send to a Microsoft account was 100 % broken whenever the user replied to a Microsoft thread. Graph's `sendMail` endpoint refuses standard RFC 5322 header names in its `internetMessageHeaders` extension:

> `400 InvalidInternetMessageHeader: The internet message header name 'In-Reply-To' should start with 'x-' or 'X-'.`

The fix routes Graph replies through Outlook's native endpoint: `POST /me/messages/{graph_message_id}/reply`. Graph builds the `In-Reply-To` / `References` / `conversationId` chain server-side from the original message — we just supply the new body content + recipients. Fresh sends and replies-to-Gmail-from-Microsoft continue to use plain `/me/sendMail` with no threading headers (a known edge for cross-provider replies that doesn't carry RFC 5322 threading).

---

## 👥 People page restored

The People extractor + identity tooling from the Phase 1 v0.4.0 prototype caused enough lock contention that v0.3.0 had to roll it back. It still wrote ~22 500 events of entity data into the substrate before being paused — that data is intact. v0.3.1 brings back **just the four read tools** so the page lights up against existing data:

- `list_entities` — sorted by strength (or alphabetically, until v0.4.0 lights up the scorer)
- `get_entity` — full payload with aliases, all emails, custom-field slots
- `entity_timeline` — events × mentions for one entity, newest first
- `find_related` — co-participants ranked by shared-event count

All four are read-only, route through the new reader pool, and don't compete with writers. The actual extractor — the lock-contention source — stays parked on the `wip/v04-people-paused` branch until v0.4.0 brings it back on top of the new architecture.

---

## ✅ The safety net we built

51 tests across the workspace, ~1.5 s to run, gating every PR. Specifically guarding against the regressions that hit us most:

| Layer | Tests | What's covered |
|---|---|---|
| Substrate integration (`tests/`) | 7 | Per-connection pool concurrency, migration idempotency, latency budgets (count, top-50 inbox, PK lookup, try_clone cold path) |
| Graph mail (`tests/`) | 4 | `sendMail` body shape (no `internetMessageHeaders`!), `/reply` URL encoding, file-attachment wire format, 4xx error surfacing |
| IMAP (unit `#[cfg(test)]`) | 20 | Cursor serde + upsert + resume_from + legacy compat, `folder_kind` classifier (Gmail / Outlook / nested), `normalize_subject` across `Re:` / `Fwd:` / `[list]` / stacked, XOAUTH2 wire bytes + auth-handshake state machine |
| Substrate pool classifier | 3 | Inbox-path tools are read-only, write tools are not, unknown method defaults to write |
| Daemon — calendar / normalize | 17 | Pre-existing |

Adding any new MCP tool? **Add it to `READ_ONLY_METHODS` in `substrate_pool.rs`** — the classifier test will fail if you forget.

Adding any new migration that includes DDL? **It's already safe** — `run_migrations` now checkpoints after each one.

### Latency budgets

Four `cargo test --release` gates that fail if a hot-path query gets quietly slower:

```
count_query_under_budget          < 50 ms   on 10 000 events
inbox_top_50_under_budget         < 100 ms  on 50 000 events
lookup_by_id_under_budget         < 5 ms    on 10 000 events
try_clone_cold_query_under_budget < 20 ms   on 5 000 events
```

Each failure reports the measured p95 so the author sees what they broke, not just "too slow."

---

## 🔬 Under the hood

```
Tauri UI  ──MCP──►  daemon
                     ├── SubstratePool
                     │     ├── writer (single Arc<Mutex<SubstrateDb>>)
                     │     │     ↑ background workers commit here
                     │     │     ↑ MCP write tools (send_email, mark_read, …)
                     │     └── readers[4] (Arc<Mutex<SubstrateDb>>, round-robin)
                     │           ↑ MCP read tools (search, get_thread, list_*)
                     │           Independent DuckDB Connections; MVCC snapshots
                     └── activity.touch() on every MCP call
                           → drives foreground-aware Graph polling cadence
```

Adding a new MCP tool: classify it in `src/daemon/src/substrate_pool.rs::READ_ONLY_METHODS`. Default for unlisted methods is **write** — the safe direction (no chance of writing through a reader and missing the snapshot).

---

## 🚧 What's next

v0.3.1 closes the stability sprint. The next milestone is the real v0.4.0:

- **People** — the extractor + strength scorer + identity resolution work that was parked on `wip/v04-people-paused`, rebuilt on top of the new per-connection model so the contention regression cannot return.
- **Graph drafts** — push local drafts to outlook.com via Graph (currently skipped per v0.3.1's `draft_sync` ms_graph guard; local drafts still work, just don't appear in outlook.com web).
- **Mock IMAP server tests** — the Tier 2 protocol-level coverage deferred from v0.3.1. Full AUTH + SELECT + UID FETCH + IDLE round-trips against a fixture server.

---

## 📦 Requirements

Same as v0.3.0:
- Linux with GNOME Online Accounts ≥ 3.50
- D-Bus session bus active
- Rust ≥ 1.85 to build
- Node ≥ 20 + npm/pnpm for the Tauri UI
- Gmail account with IMAP enabled and/or a Microsoft 365 account

No DB migration required if you upgrade from v0.3.0 (migration 0009/0010 are no-ops on substrates that already have them recorded in `_meta`).

---

## 🤝 Closing note

v0.3.0 was a thesis demonstration: two inboxes, one substrate, real cross-account threading. v0.3.1 is the engineering scaffolding that makes that thesis hold up under daily use — a reader pool that actually scales, a UI state machine that actually closes, and a test suite that actually catches the regressions before they ship.

The data stays on your machine. The architecture stays out of your way.

— next stop, v0.4.0.
