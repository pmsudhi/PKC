<div align="center">

# 🗓️ KnowDB v0.2.0 — *Milestone: Calendar, unified*

### *No more independent information islands.*

**Your meetings, your messages, and the conversations leading into them all live in one substrate. Calendar is a first-class citizen — synced two-way with Gmail and Microsoft 365, woven into the same knowledge graph your inbox already powers.**

<br/>

[![Status](https://img.shields.io/badge/status-alpha-yellow?style=flat-square)](#)
[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#)
[![Substrate](https://img.shields.io/badge/email%20%2B%20calendar-one%20substrate-purple?style=flat-square)](#)

</div>

---

## 🎯 What this milestone unlocks

KnowDB has been waiting to stop being a mailbox. With v0.2.0 it isn't one anymore.

Your **calendar is now in the same local substrate as your inbox** — synced bidirectionally with Google (via CalDAV) and Microsoft 365 (via Graph), grouped under the same accounts, with the same privacy guarantees. But more importantly, the two surfaces *know about each other*:

- Open any upcoming meeting and the prep panel shows **the recent email threads with the people you're meeting** — pulled live from the same substrate, scoped per attendee.
- Click any email thread and one button turns it into a calendar invite with everyone on the To-line already in attendance.
- Block out time, the calendar suggests it from your last commitments — coming soon.

This is what the README has been promising. **One knowledge centre, three surfaces, every action grounded in your real data.**

---

## ✨ Highlights

### 🌐 First-class calendar — two providers, one experience
- **CalDAV** for Google Calendar, with full discovery (`current-user-principal` → `calendar-home-set` → multistatus walk).
- **Microsoft Graph** for Outlook / Microsoft 365 — including events that GOA's CalDAV bridge doesn't expose.
- **All your calendars in one place** — primary, family, holidays, secondary calendars, all under per-account toggles in the sidebar.
- **Per-calendar colour-coding** with a source chip on every event surface so you always know whose calendar it lives on.

### 📅 Recurring events done right
The hardest piece of any calendar app, fully covered:
- **Master + exception storage** — your modified Tuesday only-this-week meeting and the master series live as separate rows, no clobber.
- **`EXDATE` honoured** at expansion — cancelled occurrences actually disappear.
- **This / this-and-future / all** scope prompt on every Delete · Edit · RSVP. CalDAV truncates the master with `UNTIL`; Graph patches the master recurrence range. Edit-this-and-future even *splits the series* and PUTs a fresh VEVENT continuing forward.
- **RRULE picker** on create (`Daily / Weekly / Every 2 weeks / Monthly / Yearly / Custom…`) translates to both RFC 5545 (CalDAV) and Microsoft's typed recurrence shape (Graph).
- **Background backfill** worker fills in `seriesMasterId` for any Graph events synced before the capture work — no full resync required.

### 🚀 The "Up Next" view — KnowDB's signature surface
The new default landing for `/calendar`. Instead of a grid, you get the next 7 days as rich **meeting-prep cards**:
- Date, time, "starts in N minutes" badge, source-calendar chip.
- Attendee avatars, with a one-click **Join** button when there's a Meet / Teams / Zoom link.
- **"Recent with attendees"** — for each non-self attendee, the most recent email threads in your substrate involving *both* of you, last 21 days. Walk into the meeting knowing what they last asked, what you last committed to.
- Click any thread → jumps straight to your inbox at that thread.

### ✍️ Compose-style event windows
Same window-chrome muscle memory as the compose flow:
- **Floating, docked bottom-right** — `560 × 640`, with **Minimize / Expand / Close** controls.
- **Notion-style date+time picker** — separate date and time pills, each with its own popover (calendar grid · 15-min time list).
- **Quick add** input at the top — *"Coffee with priya@example.com Monday 10am for 30 min at Café"* parses into title / start / end / attendees / location.
- **Add video conferencing** toggle — Google Meet (CalDAV `X-GOOGLE-CONFERENCE`) or Microsoft Teams (`isOnlineMeeting=true`) auto-minted by the provider on save.
- **Role-aware actions** in the detail drawer — organisers see Edit + Delete, attendees see RSVP + Propose new time.
- **Propose new time** sends a proper iTIP `METHOD:COUNTER` over SMTP (Gmail) or hits `/me/events/{id}/tentativelyAccept` with `proposedNewTime` (Graph). The organiser sees a real "X proposed a new time" card on their end.

### 🖱️ Modern direct-manipulation grid
- **Drag-to-move** event blocks along the time axis, snapping to 15 min.
- **Drag-to-resize** via a bottom handle that surfaces on hover.
- **Click-and-drag** in empty space to lasso a time range — pops the New Event window pre-filled.
- **Conflict highlighting** — overlapping events get a subtle diagonal stripe and a tooltip explaining the overlap.
- **Working hours** off-time shading (default 9–18, user-configurable).
- **Multi-timezone strip** — optional second hour column for the distributed team case.

### ⌨️ Keyboard-first
Notion / Vimcal level shortcut coverage:
- `1`–`5` switch between Up-Next / Agenda / Month / Week / Day.
- `T` jumps to today, `J` / `K` (or arrow keys) step a period.
- `N` opens New Event, `/` focuses search.
- All ignored inside inputs / textareas / contenteditables so typing never accidentally triggers a shortcut.

### 🔍 Event search across all calendars
A substrate-backed search (substring on subject · body · location · attendees) accessible from a small icon in the toolbar — or `/`. Click any result to jump the focused date and open the detail drawer.

### 🪄 Cross-substrate moves
This is the part you can't get from a calendar-only app:
- **Schedule meeting from email thread** — button in the inbox toolbar. The New Event window pops globally (no route change) with subject and attendees from the thread.
- **Recent threads with attendees** in the event drawer and Up-Next card — grouped per-attendee, excludes self-from-self traffic across your own connected accounts.
- **Source-calendar chip + account email** on every event surface so the multi-account merge stays legible.

---

## 🧪 What you can do today

```
✓ Sync calendars from Gmail (CalDAV) and Microsoft 365 (Graph)
✓ Multi-account merged view with per-calendar visibility toggles
✓ Display every recurring-event variant: master, override, EXDATE, cancelled
✓ Create / edit / delete events end-to-end on both backends
✓ RRULE picker with provider-correct translation
✓ This / this-and-future / all scope on Delete, Edit, RSVP for recurring
✓ RSVP (Accept / Decline / Tentative) on both backends
✓ Propose new time — iTIP COUNTER over SMTP on Gmail, Graph tentativelyAccept on MS
✓ Drag-to-move, drag-to-resize, click-and-drag-to-create on the time grid
✓ Conflict highlighting + working-hours shading + secondary timezone strip
✓ Natural-language quick-add ("Coffee with priya@... Monday 10am at Café")
✓ Schedule meeting from any email thread (inbox toolbar)
✓ "Add video conferencing" auto-mints Meet / Teams links on save
✓ Up-Next view with per-attendee email-thread roll-ups
✓ Search events by title / attendee / location across every connected calendar
✓ Keyboard shortcuts on every action surface
```

---

## 📊 What's measured

| Metric | This release | Budget |
|---|---|---|
| Daemon idle RAM | ~85 MB | < 100 MB |
| Cold start to ready | ~370 ms | < 500 ms |
| Calendar sync (incremental, both accounts) | 3–8 s typical | n/a |
| `list_events` over a 7-day window | ~15 ms p95 | < 100 ms |
| RRULE expansion (`FREQ=WEEKLY`, 1-year window) | ~2 ms | n/a |
| `event_context` (per-attendee thread roll-up) | ~25 ms p95 | n/a |
| `search_events` LIKE pass (10K events) | ~12 ms p95 | < 100 ms |
| seriesMasterId backfill batch (50 events) | ~600 ms | n/a |

All within budget; perf gates green.

---

## 🛠️ Under the hood — what changed

### Substrate schema
Three new tables / column groups joined the substrate this milestone:

```
calendars                  → discovered calendars, one row per source
events (extended)          → calendar columns: calendar_id, calendar_end_at,
                             all_day, rrule, recurrence_id, calendar_etag,
                             is_cancelled
events.metadata            → exdates, series_master_id, _series_master_checked,
                             organizer, conferencing_url
```

Migrations 0008 (calendar foundation), 0009-0010 (small fixes for index/partial constraints), all forward-only and idempotent.

### New crates
- **`crates/connectors/calendar/`** — provider-agnostic types, the iCalendar parser, plus `caldav.rs` and `graph.rs` modules.

### Substrate dedupe magic
Master + exception VEVENTs share a UID but used to clobber each other. Now `source_event_id` encodes `recurrence_id` as `uid#YYYYMMDDTHHMMSSZ`, so masters and overrides are distinct rows that still match via `ON CONFLICT (source, source_id)`. The expander at query time joins them back together — overrides substitute their corresponding occurrence; cancelled overrides act as implicit `EXDATE`s.

### Two-way scope-aware writes
- **Delete** — CalDAV `EXDATE`-the-master (this) / DELETE the .ics (all) / `UNTIL`-truncate (this-and-future). Graph DELETEs the occurrence (this) / DELETEs the seriesMasterId (all) / PATCHes recurrence.range.endDate (this-and-future).
- **Edit** — CalDAV writes an override VEVENT into the same .ics (this) or PUTs the master (all) or splits the series with a UNTIL'd master + fresh VEVENT (this-and-future). Graph PATCHes the occurrence / master / splits via PATCH master + POST new event.
- **RSVP** — CalDAV mutates `PARTSTAT` in the master (all) or writes an override (this). Graph hits `/accept|decline|tentativelyAccept` on the occurrence or master id.

### The cross-substrate query
`event_context(event_id)` is the new MCP tool that makes Up-Next + the detail drawer interesting:

```sql
For each attendee on the event (minus the user's own accounts):
  SELECT threads where some event in the last 21 days
    has BOTH a user-email AND that attendee's email
    in raw_participants.
  GROUP BY thread, ORDER BY recency, LIMIT 5.
```

The user's own emails are looked up dynamically from `calendars.metadata.email` — nothing hardcoded, so adding a third account automatically extends the dedupe.

### Background workers
Both new workers live on the dedicated sync-scheduler thread (same `current_thread` runtime as IMAP draft sync, because `async-imap` / reqwest's connection sharing is happier without `Send`):
- **`calendar_sync`** — pulls events from each enabled GOA calendar account every ~5 min.
- **`backfill_graph_series_master`** — for any pre-existing Graph events lacking `series_master_id`, fetches the projection (`id, seriesMasterId, type`) and stamps the answer. 50 per cycle, bounded.

---

## 🧭 What's next

This milestone closes Phase 1.5 — *Email + Calendar, unified* — and clears the runway for Phase 2's embedding-powered semantic search and Phase 3's commitment extraction.

| Phase | Focus | Status |
|---|---|---|
| **Phase 0** — Foundation | Cargo workspace, substrate, state, CLI | ✅ Done |
| **Phase 1** — Email | IMAP sync, normalize, compose, send, IMAP drafts | ✅ Done (`v0.1.0`) |
| **Phase 1.5** — Calendar | CalDAV + Graph, recurrences, cross-substrate prep cards | ✅ **This release** |
| **Phase 2** — Search | bge-small embedding pipeline, HNSW, hybrid BM25 + ANN | 🔜 Next |
| **Phase 3** — Intelligence | Commitment extraction, identity resolution, people graph | ⏳ |
| **Phase 4** — Assistant | MCP frontier router, audit log, `knowdb chat` | ⏳ |
| **Phase 5** — UI polish | Privacy panel, reconciliation queue, design pass | ⏳ |
| **Phase 6** — Ship | systemd unit, .deb / .AppImage, CI perf gates, docs, v1.0 | ⏳ |

### Known follow-ups
Listed in full in [`CALENDAR_ROADMAP.md`](CALENDAR_ROADMAP.md), in priority order:
- Cross-day drag in Week view (currently same-column only).
- Find-a-time across attendees (free/busy overlay).
- Pre-meeting LLM briefing (3-bullet summary of the related threads — Phase 4 hook).
- AI auto-scheduling for habits / focus time (Motion / Reclaim pattern).
- Public booking pages (Cal.com equivalent).
- Travel time, hourly heatmap, snooze, bulk operations, iTIP REPLY emails.

---

## 🙏 What this milestone means

For 15 years calendar and email have lived as separate apps with separate data, talking past each other through awkward "Add to calendar" buttons and notification badges that don't know what they reference. The conventional answer has been to put both into someone else's cloud and hope the cross-references happen on their servers.

KnowDB makes them *the same thing* — locally. Two surfaces, one substrate, every join queryable. When you open Up Next, the meeting carries the conversational context with it because the database knows both pieces of your day are about the same people.

There are no information islands. There is no provider in the middle deciding which thread "relates to" which meeting. There is no third party with a copy of either your inbox or your calendar. The graph is yours, the queries are yours, and the answers are grounded in the actual messages and the actual events.

Phase 2 will add semantic search across the whole substrate. Phase 3 will start extracting commitments and surfacing them as soft blocks on the calendar. Phase 4 will let you chat with this graph in natural language. Every step earns the next.

But even now, today, the loop is closed. **You have one place for your working day.**

---

<div align="center">

**KnowDB v0.2.0** · *Calendar, unified*

*Your meetings. Your messages. Your machine. Your assistant.*

</div>
