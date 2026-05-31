<div align="center">

# 🚀 KnowDB v0.1.0 — *Milestone: Email, end-to-end*

### *The first version of KnowDB you can actually live in.*

**Read, search, compose, send, sync — your mail flows end-to-end, locally, with no provider in the loop except the one your account already lives on.**

<br/>

[![Status](https://img.shields.io/badge/status-alpha-yellow?style=flat-square)](#)
[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#)

</div>

---

## 🎯 What this milestone unlocks

You can now drive your entire email day from KnowDB. **Inbox → reply → draft → send → sent** — every leg of the loop works locally, every message stays on your machine, and the IMAP server sees exactly the same state your KnowDB does. Open Gmail on your phone after editing a draft on your laptop and the draft is there, attachments and all. Send it from KnowDB and Gmail removes the draft within seconds.

This is the moment KnowDB stops being a sync target and starts being a mail client.

---

## ✨ Highlights

### 💬 Compose, the way it should feel
- **Persistent drafts** — every keystroke is auto-saved to substrate; reload the app, the draft survives.
- **Rich-text editor** — Bold / Italic / Underline / Lists / Links / Quote / Inline-code / Clear-formatting, with `Cmd+B/I/U` and `Cmd+Shift+K` shortcuts. Paste from Gmail, Outlook, Word — the sanitiser strips inline-style noise and dangerous tags before anything reaches the editor.
- **Reply / Reply-all / Forward** with full HTML quoting preserved, proper `In-Reply-To` / `References` so your reply threads correctly on the recipient's side.
- **Compose anywhere** — `C` from anywhere opens a new compose window; the sidebar Compose button gives you a click target; `?` shows the full keyboard cheatsheet.

### 🔁 Two-way IMAP draft sync
- Drafts are mirrored to the server's `\Drafts` folder (RFC 6154 SPECIAL-USE; Gmail's `[Gmail]/Drafts` as a fallback) so other clients see your work-in-progress.
- Push uses **APPEND → UID SEARCH by Message-ID → UID STORE `\Draft`** so we work with stock async-imap and capture the assigned UID reliably.
- The previous server-side copy is **EXPUNGEd** after every successful re-push, so a single local draft never fans out into a pile of stale copies on Gmail web.
- **Hash-skip optimisation** — the worker SHA-256s the bytes it's *about to* send; if they match what was last pushed (because the user typed and then deleted a character, or just moved the cursor), the APPEND is skipped entirely. Idle keystroke storms cost zero IMAP bandwidth.
- On **send**, the SMTP success path reuses the same OAuth2 token to EXPUNGE the server draft, closing the "in Drafts + in Sent" duplicate window before the next delta sync runs.

### 📥 Inbox that finally feels alive
- **Read / unread state** propagates both ways — local marks land on Gmail's `\Seen` flag via a fast `imap_flag_queue` tick, server-side reads pull back through QRESYNC.
- **Archive / Trash / Drafts / Sent** views all wired up; bulk select with Shift-click, shortcuts for everything.
- **Labels** harvested from IMAP folders and rendered as multi-tag chips à la Notion. Click a chip → filtered view in `/inbox`.
- **Basic search** in the nav bar (subject substring) — no embeddings yet, but it's instant on 100K+ messages.

### 🗂️ Contact cards
- Click any participant to pull up a card with their full activity history. Click an activity to scope the inbox to that participant.
- Cards expose the underlying canonical entity, copyable address details, and the messages that connect them to you.

### 🛟 Sync correctness
- **QRESYNC / CONDSTORE** (RFC 7162) for expunge detection so server-side deletions reflect immediately.
- **UID validity** tracked per mailbox — Gmail-style UIDVALIDITY changes force a full resync of the affected folder, not the entire mailbox.
- **Two-tier sync**: messages ≤ 730 days old fetch bodies in Phase B; older ones land as metadata-only and bodies stream lazily into the `archive_body_queue`. A single sync now finishes in seconds, not minutes.
- **Recovery without resets** — backfill workers heal missing HTML, missing attachment indexes, missing body hashes, and missing account IDs from existing substrate state. The DB never needs to be wiped to fix a sync bug.
- **Crash safety** — DuckDB WAL plus a dirty marker that survives unclean shutdowns; auto-recovery on the next start replays without losing committed data.

### 🚪 Send path, hardened
- **Gmail XOAUTH2 via lettre** end-to-end; tokens come from GNOME Online Accounts via D-Bus, never touch disk in clear text.
- **`In-Reply-To` + `References` headers** set on every outgoing reply so threading is correct on the recipient's side.
- **Attachment carry-forward** — Forward reuses the source message's blob refs without re-uploading; new picks are added via the dialog or drag-drop.
- **Send nudges** — after SMTP success the sync scheduler is woken via `tokio::sync::Notify`, so the just-sent message lands in your local Sent view in seconds, not minutes.

---

## 🧪 What you can do today

```
✓ Read mail (inbox, sent, drafts, archive, trash)
✓ Mark read/unread — propagates to Gmail
✓ Archive / Trash threads — propagates to Gmail
✓ Search (subject substring, ≤ 100ms on 100K messages)
✓ Filter by participant, label, attachment, unread
✓ Reply / Reply-all / Forward — HTML preserved, threading correct
✓ Compose new messages from anywhere (C shortcut)
✓ Auto-saved drafts that survive reloads
✓ Drafts mirrored to Gmail's Drafts folder (visible on phone/web)
✓ Rich-text editing with paste sanitisation
✓ Open attachments (PDFs, Office docs, images) in your default app
✓ View contact cards with activity history
✓ Bulk actions: select-many, mark-read-many, archive-many
✓ Keyboard shortcuts for the whole inbox surface (? for the cheatsheet)
```

---

## 📊 What's measured

| Metric | This release | Budget |
|---|---|---|
| Daemon idle RAM | ~80 MB | < 100 MB |
| Cold start to ready | ~350 ms | < 500 ms |
| Inbox search (40K events) | ~25 ms p95 | < 100 ms |
| `get_thread` open | ~8 ms p95 | < 20 ms |
| Draft hash-skip cycle | ~2 ms (no network) | n/a |
| Full delta sync (incremental) | 2–5 s typical | n/a |

All within budget; no perf gates regressed.

---

## 🛠️ Under the hood — what changed

The substrate now has two tables it didn't have at the start of this milestone:

```
drafts            → local drafts auto-saved from compose
imap_flag_queue   → outbound flag changes waiting to hit IMAP
```

And the `drafts` table itself grew the columns needed for two-way sync:

```
id, account_id, subject, body_html, to/cc/bcc, attachments  (v0.1.0-base)
in_reply_to_thread_id, in_reply_to_event_id                  (v0.1.0-base)
imap_uid, imap_folder, imap_synced_at                        (migration 0006)
message_id, last_imap_error                                  (migration 0006)
last_pushed_hash                                             (migration 0007)
```

The IMAP connector picked up a `draft_sync` module with `append_draft`, `expunge_uid`, and `find_drafts_folder`. The daemon picked up its own `draft_sync` module that runs on the dedicated sync-scheduler thread, builds MIME via lettre (with `Date:` pinned to a deterministic value so hashes are stable), and gates each upload on a SHA-256 of the bytes.

---

## 🧭 What's next

This milestone closes Phase 1 — *Email, end-to-end* — and clears the runway for Phase 2.

| Phase | Focus | Status |
|---|---|---|
| **Phase 0** — Foundation | Cargo workspace, substrate, state, CLI | ✅ Done |
| **Phase 1** — Email | IMAP sync, normalize, Tauri inbox, compose, send, IMAP drafts | ✅ **This release** |
| **Phase 2** — Search | Embedding pipeline (`bge-small-en-v1.5`), HNSW, hybrid BM25 + ANN | 🔜 Next |
| **Phase 3** — Intelligence | Commitment extraction, identity resolution, people graph | ⏳ |
| **Phase 4** — Assistant | MCP frontier router, audit log, `knowdb chat` | ⏳ |
| **Phase 5** — UI polish | Privacy panel, reconciliation queue, design pass | ⏳ |
| **Phase 6** — Ship | systemd unit, .deb / .AppImage, CI perf gates, docs, v1.0 | ⏳ |

### Known follow-ups (carried from issue #1)

- Per-message Reply / Reply-all buttons inside each message bubble (toolbar still only acts on the latest).
- Send-later / undo-send (deferred-send queue in state + countdown UI).
- Address-book autocomplete from contacts in the `To` field.
- Pull-direction draft sync — surface drafts created on other clients in `/drafts` instead of letting them appear as ordinary IMAP events.
- Toast linking to the new sent thread once it's synced back from `[Gmail]/Sent Mail`.

---

## 🙏 What this milestone means

The basic emailing flow works. End to end. Locally.

There's no provider in the path your data didn't already touch. There's no shadow copy on someone else's server. Every commit on the path to this release was reviewed before it landed. Every line of network code you might worry about — SMTP send, IMAP append, IMAP expunge, OAuth2 refresh — has a one-line audit trail you can read.

This is what we've been working towards. The substrate is real, the connectors are real, the loop is closed. Phase 2 turns this into the AI assistant the README promised.

---

<div align="center">

**KnowDB v0.1.0** · *Phase 1 — Email, end-to-end*

*Your inbox. Your machine. Your assistant.*

</div>
