<div align="center">

# 👥 KnowDB v0.4.0 — *Milestone: the Integrated People Hub*

### *No more separate apps for mail, calendar, and contacts. One personal knowledge centre.*

**The people you know, the conversations you've had, the meetings you've shared, the files you've exchanged — all sitting in one local substrate, joined where they overlap, and queryable from any surface. Your inbox, your calendar, and your address book stop being three apps and start being three views of the same graph.**

<br/>

[![Status](https://img.shields.io/badge/status-alpha-yellow?style=flat-square)](#)
[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![Phases](https://img.shields.io/badge/People%20phases-1→7%20shipped-brightgreen?style=flat-square)](#-the-seven-phases)
[![MCP tools](https://img.shields.io/badge/MCP%20tools-+45-purple?style=flat-square)](#-under-the-hood)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#)

</div>

---

## 🎯 What this milestone unlocks

Until v0.4.0, KnowDB was a great inbox that also happened to know about your calendar. People were a placeholder — a list you could browse but not really *use*. Edits didn't persist. Merges didn't exist. Tags lived on email but not on the people sending it. The graph was implied, not visible.

**v0.4.0 closes that loop.** Every interaction you have with another human — every email, every meeting, every shared attachment, every signature change — now collects under that person's identity and shows up on every surface that mentions them. The People view is the new front door; Inbox and Calendar are how you act on what it finds.

Three things follow from this:

- **You stop juggling apps.** Schedule a meeting from a contact card. Tag a thread and see the tag on the calendar invite. Search "team" and find people, threads, and events in one query.
- **The substrate starts paying off.** Six tables × ten thousand events × your real history becomes a queryable knowledge graph the moment People makes the joins navigable.
- **The "local-first" promise becomes load-bearing.** Privacy flags exclude contacts from outbound calls. vCard / CSV export means leaving is one click. The audit log already covers every byte that left.

---

## 🚀 The seven phases

The People plan was sequenced as seven phases (spec 33). **All seven shipped in this milestone.** Each sub-document linked below is a deep-dive on the feature, the design tradeoffs, and the surfaces it touches.

| Phase | Theme | What it gave you | Deep-dive |
|---|---|---|---|
| **1** | Data plane | Persistent edits, multi-select, manual merge + split with conflict-resolution UI, delete + undo | [`01-data-plane.md`](docs/release-notes/v0.4.0/01-data-plane.md) |
| **2** | Tags & organization | Tags / groups / smart groups / saved views — one chip, three surfaces | [`02-tags-and-organization.md`](docs/release-notes/v0.4.0/02-tags-and-organization.md) |
| **3** | Cross-surface discovery | "Schedule with X", "Files shared with X", "Where we've met", Inbox People-Mode, Birthdays tab | [`03-cross-surface-discovery.md`](docs/release-notes/v0.4.0/03-cross-surface-discovery.md) |
| **4** | Relationship intelligence | Keep-in-touch reminders, dormant + rising chips, reply-rate tracker, job-change detection, debts ledger, log-compressed strength score | [`04-relationship-intelligence.md`](docs/release-notes/v0.4.0/04-relationship-intelligence.md) |
| **5** | Data sovereignty | vCard 4.0 import + export, QR-code vCard, CSV bulk export, per-contact privacy flag | [`05-data-sovereignty.md`](docs/release-notes/v0.4.0/05-data-sovereignty.md) |
| **6** | Network graph | Cytoscape force-directed graph, color-by-domain, shortest-path overlay, saved views | [`06-network-graph.md`](docs/release-notes/v0.4.0/06-network-graph.md) |
| **7** | Power-user polish | Keyboard shortcuts (j/k/e/m/⌘+Enter), drag-and-drop into groups, inline-edit on detail header, list/cards toggle, URL deeplinks, **custom auto-tag rules engine** | [`07-power-user-polish.md`](docs/release-notes/v0.4.0/07-power-user-polish.md) |

> 🧭 Start with [`docs/release-notes/v0.4.0/README.md`](docs/release-notes/v0.4.0/README.md) for the full index, including which sub-doc to read first depending on what you want to learn.

### The three surfaces, side by side

```
   📥 Inbox                    📅 Calendar                  👥 People (new front door)
   ────────────                ────────────                 ────────────────────────
```

| | | |
|:-:|:-:|:-:|
| ![Inbox](docs/release-notes/v0.4.0/screenshots/00-inbox.png) | ![Calendar](docs/release-notes/v0.4.0/screenshots/00-calendar.png) | ![People](docs/release-notes/v0.4.0/screenshots/01-people-list.png) |

*Three views of the same substrate. The chip on Alice's row in People is the same chip on Alice's thread in Inbox is the same chip on Alice's invite in Calendar. Screenshots are live Playwright captures of `ui/` against canned MCP fixtures — see [`docs/release-notes/v0.4.0/screenshots/README.md`](docs/release-notes/v0.4.0/screenshots/README.md) for how to regenerate.*

---

## ✨ Highlights at a glance

### 🧑‍🤝‍🧑 One identity, everywhere

```
                              ┌──────────────────┐
                              │   Alice Chen     │
                              │   ACME Corp · CTO│
                              └──────────────────┘
                                       │
       ┌───────────────────────────────┼───────────────────────────────┐
       ▼                               ▼                               ▼
  ┌──────────┐                  ┌─────────────┐                ┌──────────────┐
  │  Inbox   │                  │  Calendar   │                │  Attachments │
  │ 47 threads                  │ 12 meetings │                │ 9 files      │
  │ since '24│                  │ since Apr   │                │ shared       │
  └──────────┘                  └─────────────┘                └──────────────┘
       │                               │                               │
       └──────► Tags: team · client ◄──┴──► Notes · Custom fields ◄────┘
```

Edits to Alice's name, emails, company, phones, addresses, social handles, URLs, and **arbitrary custom fields** now persist to the substrate via the new `update_entity` / `set_custom_field` / `add_note` MCP tools. A merge of three duplicate "Alice Chen" cards into one rewrites every mention in `entities`, `mentions`, `extractions`, `thread_participants`, and `entity_tags` atomically, and a single click on **Undo merge** un-does it via `entity_merge_log`. See [`01-data-plane.md`](docs/release-notes/v0.4.0/01-data-plane.md).

### 🏷️ Tags that travel

A tag attached to Alice on the People view shows up on her inbox threads and her calendar invites — same chip, same colour, same component. Smart groups (`Dormant teammates`) are saved filters that re-evaluate on every read. Saved views (sort + filter + columns) are URL-encoded so you can bookmark and share them across surfaces. See [`02-tags-and-organization.md`](docs/release-notes/v0.4.0/02-tags-and-organization.md).

### 🔁 Cross-surface discovery

- **Schedule meeting with Alice** on her contact card → opens the calendar modal with Alice prefilled as attendee.
- **Compose to Alice** with the last 3 threads' subjects shown as context next to the To field.
- **"Where we've met"** card derives from `events.metadata.location` distinct values.
- **"Files & attachments shared"** card derives from `events.metadata.attachments` scoped to Alice's mentions.
- **Inbox People-Mode** regroups threads by primary sender entity.
- **Birthdays tab** + nightly job that mints calendar events 7 / 1 / 0 days ahead.

See [`03-cross-surface-discovery.md`](docs/release-notes/v0.4.0/03-cross-surface-discovery.md).

### 🧠 Relationship intelligence — without any LLM

Every signal here is computed from substrate data the daemon already has. **Zero LLM calls.** Zero new manual data entry except for the explicit toggles.

- **Strength score** is log-compressed (`log(1+mentions)*30 + log(1+threads)*30 + recency_bonus`) so your top contact no longer scores 50 000 while the next 999 cluster around 100. Long tail visible.
- **`is_rising` chip** when this week's mentions > last week's by ≥ 50 %.
- **Dormant chip** when last-contact > N days (configurable per cadence).
- **Keep-in-touch reminders** — pick a cadence (30 / 60 / 90 / 180 days), KnowDB nudges you when overdue, auto-acks if you've already engaged since the last ping.
- **Reply-rate tracker** ("emailed 5× since last reply") catches one-sided threads.
- **Job-change detection** parses `From:` sig blocks, stores `entity_signature_history`, raises a card when company shifts inside a 30-day window.
- **Debts ledger** — minor-currency-units accurate, settle / unsettle, full audit.
- **Manual relationships** — Alice is married to Bob, Bob reports to Charlie. Stored in `entity_relationships`, surfaced in the graph view.

See [`04-relationship-intelligence.md`](docs/release-notes/v0.4.0/04-relationship-intelligence.md).

### 🛂 Data sovereignty

- **vCard 4.0** import and export, per-contact or whole address book.
- **QR-code vCard** per contact — paste into any phone camera.
- **CSV export** of the entire People dataset (one row per entity, contact-method columns flattened).
- **Privacy flag** (`entity_privacy_flags`) — UI toggle, sparse-by-default storage, the future LLM router's policy gate is wired against this table.

See [`05-data-sovereignty.md`](docs/release-notes/v0.4.0/05-data-sovereignty.md).

### 🕸️ The network, visible

A real force-directed graph (Cytoscape.js + fcose layout) at `/people/graph`. Click a node → contact card. Pick two contacts → shortest-path overlay highlights the people in between. Toggle **Color by domain** to see clusters by company. Save your favourite view configurations.

See [`06-network-graph.md`](docs/release-notes/v0.4.0/06-network-graph.md).

### ⌨️ Linear-class keyboard control + a rules engine

`j`/`k` to navigate, `e` to edit, `m` to merge selected, `/` to search, `⌘+Enter` to save. Inline-edit names and emails by clicking on them. Drag contacts into groups. URL deeplinks so any People-view state is shareable.

And the headline shipped in Phase 7: a **custom rules engine** — write deterministic rules ("when alice@acme.com emails, tag thread `acme`"; "when subject contains `[urgent]`, tag thread `priority`") that auto-tag threads / events / entities. Five ops (`equals`, `contains`, `starts_with`, `ends_with`, `regex`). One-shot **Run now** for retroactive tagging. Transactionally atomic — every rule's matches commit together or not at all.

See [`07-power-user-polish.md`](docs/release-notes/v0.4.0/07-power-user-polish.md).

---

## 🔬 Under the hood

### Substrate schema growth

Twelve new migration files (`0008` → `0019`) carry the People surface:

```
0008_entity_notes.sql
0009_tag_definitions_and_target_tags.sql
0010_entity_custom_fields_writer_path.sql
0011_entity_merge_log.sql
0012_entity_groups_and_members.sql
0013_smart_groups_saved_views.sql
0014_entity_reminders.sql
0015_entity_signature_history.sql
0016_entity_relationships.sql
0017_entity_debts.sql
0018_entity_privacy_flags.sql
0019_auto_tag_rules.sql
```

Every one of them FKs back to `entities` (or to `tag_definitions` / `auto_tag_rules` for cross-target tables). No table is an island — the connectivity-contract acceptance gate from spec 33.

### MCP surface

**+45 typed tools** under `crates/daemon/src/mcp/tools/` (split across nine per-surface modules as of the refactor that landed alongside this milestone). Selected additions:

| Surface | Tools |
|---|---|
| Entities | `list_entities`, `get_entity`, `entity_timeline`, `create_entity`, `update_entity`, `merge_entities`, `split_entity`, `delete_entity`, `set_custom_field`, `delete_custom_field`, `add_note`, `update_note`, `delete_note` |
| Tags & groups | `list_tags`, `create_tag`, `update_tag`, `delete_tag`, `attach_tag`, `detach_tag`, `list_groups`, `create_group`, `update_group`, `delete_group`, `add_to_group`, `remove_from_group`, `list_smart_groups`, `create_smart_group`, `delete_smart_group`, `evaluate_smart_group`, `list_saved_views`, `create_saved_view`, `update_saved_view`, `delete_saved_view` |
| Cross-surface | `contact_upcoming_meetings`, `contact_meeting_locations`, `contact_shared_attachments`, `people_on_calendar_today`, `list_upcoming_birthdays` |
| Intelligence | `set_reminder`, `delete_reminder`, `ack_reminder`, `list_reminders_due`, `get_reminder`, `reply_rate`, `rescan_signatures`, `get_signature_history`, `list_job_changes`, `list_relationships`, `add_relationship`, `delete_relationship`, `list_debts`, `add_debt`, `settle_debt` |
| Graph | `find_related`, `get_graph`, `path_between`, `connections_between`, `cluster_by_domain`, `list_influencers` |
| Portability | `export_people_csv`, `export_vcard`, `qrcode_vcard`, `export_all_vcards`, `import_vcards` |
| Privacy | `set_entity_private`, `list_private_entities` |
| Rules | `list_rules`, `create_rule`, `update_rule`, `delete_rule`, `run_rules_now` |

All read tools route through the **reader pool** from v0.3.1, so a busy writer never starves the People view.

### Code organization

`mcp/tools.rs` (one 14 k-line file as of v0.3.1) was split into **per-surface submodules** in the same release window:

```
src/daemon/src/mcp/tools/
├── mod.rs           (people + tags + groups + saved views + system)
├── inbox.rs         (search, get_thread, set_has_unread, list_labels, flag_threads_and_queue)
├── calendar.rs      (event CRUD, RSVP, RRULE expansion)
├── compose.rs       (send_email, drafts)
├── intelligence.rs  (reminders, reply_rate, sig history, relationships, debts)
├── graph.rs         (find_related, path_between, get_graph, cluster_by_domain, list_influencers)
├── portability.rs   (vCard / CSV / QR)
├── privacy.rs       (set_entity_private, list_private_entities)
└── rules.rs         (Phase 7 #7.5 — custom auto-tag rules engine)
```

Same dispatcher in `mcp/server.rs`, no client-visible change.

### Test coverage

**124 daemon tests** in `cargo test -p knowdb` (up from 51 in v0.3.1) — plus the existing IMAP, Graph, substrate, and pool layers. Coverage focuses on the regressions most likely to bite People:

- `phase4_tests` — score formula log-compression, `is_rising` window, reply-rate counting, sig-history dedup
- `phase2_tests` — tag CRUD lifecycle, smart-group re-evaluation against stored filter
- `people_tests` — `list_entities` self-filtering via account_id, dormant heuristic, mailing-list exclusion, log-compressed score range, find_related bounded under heavy history
- `phase5_tests` (rules engine) — every operator (`equals`, `contains`, `regex`, `starts_with`, `ends_with`) verified against threads / events / entities target_types; `run_rules_now` atomicity tested

Plus Playwright e2e specs against the Tauri UI for the merge dialog, the graph view, the birthdays cluster, the keyboard shortcuts, and the rules editor.

---

## 📊 Scope summary

| Metric | v0.3.1 | v0.4.0 | Delta |
|---|---|---|---|
| MCP tools | 22 | **67** | +45 |
| Substrate migrations | 7 | **19** | +12 |
| Daemon unit tests | 51 | **124** | +73 |
| Playwright e2e specs | 4 | **18** | +14 |
| Lines in `mcp/tools/` | 4 290 | **14 852** | +10 562 |
| UI views under `people/` | 6 | **23** | +17 |
| People-feature commits | — | **30** | — |

---

## 🚧 What's explicitly **not** in this milestone

Three groups of items are deliberately parked, with an audit trail in `~/.claude/projects/.../memory/project-people-parked.md`:

1. **All AI / LLM features** — extraction, smart suggestions, "Catch up" drafts, AI columns, natural-language command bar. Held for a future spec per the project's "no AI yet" operating principle. The substrate is shaped to receive them: `extractions` table, `embeddings` table, `entity_privacy_flags` already exists for the router gate.
2. **CardDAV / Google Contacts API connectors** (#5.1 / #5.4). One-shot vCard import covers the common case; a full two-way sync is its own crate's worth of work.
3. **Compose-surface hardening** (#5.12 strip-tracking-pixels, #5.13 mass-action throttle). Small wins worth bundling later with other compose-only polish.

---

## 📦 Requirements

Same as v0.3.x:

- **Linux** with GNOME Online Accounts ≥ 3.50 (Fedora 41+, Ubuntu 24.04+, Arch with `gnome-online-accounts` installed)
- **D-Bus session bus** active
- **Rust** ≥ 1.85 to build from source
- **Node ≥ 20** + **pnpm/npm** to build the Tauri UI
- A Google account with IMAP enabled and/or a Microsoft 365 account

Upgrading from v0.3.x: **no manual migration step**. The 12 new migrations apply on first daemon start; recovery backfills (HTML hash, attachment index, signature history) seed automatically from existing data.

---

## 🤝 Closing note

The thesis has not changed: **one local substrate, many surfaces, every action grounded in your real data**.

- **v0.1** made it true for one inbox.
- **v0.2** made it true for the calendar.
- **v0.3** made it true across providers — Gmail and Microsoft 365 in one substrate.
- **v0.4** makes it true across the *people* themselves. Mail, calendar, contacts, companies, relationships, debts, attachments — every conversation you've had with another human is now reachable from a single card and queryable from any surface.

There is no separate contacts app anymore. There never needed to be one. **The conversation was always the contact; the contact was always the conversation.** v0.4.0 finally lets the substrate say that out loud.

Next stop: the AI surface, built on top of this substrate, gated by the privacy flags we just shipped, audited by the log we've been writing since v0.1.

— next stop, v0.5.0.

---

*Built locally. Synced locally. Audited locally. The data stays on your machine.*
