# Phase 1 — The Data Plane

> *Edits persist. Merges are real. Splits are real. Deletes can be undone. The People page stops being read-only.*

---

## The problem we ended

Through v0.3.x, the People view was a *projection* over the substrate. You could browse it, search it, hover over it — but you couldn't edit it in any way that survived a daemon restart. Two well-meaning systems were both wrong about who owned the data:

```
            ┌─────────────────────────┐
            │   Tauri UI (zustand)    │
            │   ─ local-only edits    │   ── reload → gone
            │   ─ applyLocalOverride  │
            └─────────────────────────┘
                       ▲
                       │  read only
            ┌─────────────────────────┐
            │   DuckDB substrate      │
            │   ─ entities table      │   ── unmodifiable from UI
            │   ─ mentions table      │
            └─────────────────────────┘
```

You could "rename" Alice in the UI; the name reverted next time the daemon resync'd from email signatures. You could "merge" two duplicates; they un-merged on next launch. **Manual control was a fiction.**

The Phase 1 acceptance test (from spec 33) made the goal concrete:

> *A user can manually merge 3 duplicates of Alice into one card, choose which name/email wins, see the merge in the audit log, and then split it back if they change their mind. Edits persist across daemon restart.*

---

## What shipped

### 1.1 — Schema additions

```sql
-- 0008_entity_notes.sql
CREATE TABLE entity_notes (
  id          UUID PRIMARY KEY DEFAULT uuid(),
  entity_id   UUID NOT NULL REFERENCES entities(id),
  body        TEXT NOT NULL,
  source      VARCHAR NOT NULL DEFAULT 'user',  -- 'user' | 'ai' (future)
  created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- 0011_entity_merge_log.sql  — the undo table
CREATE TABLE entity_merge_log (
  id              UUID PRIMARY KEY DEFAULT uuid(),
  primary_id      UUID NOT NULL,
  merged_ids      UUID[] NOT NULL,
  merge_choices   JSON NOT NULL,    -- per-field winner snapshot
  merged_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  undone_at       TIMESTAMP,
  undone_choices  JSON              -- per-field state at undo time
);
```

`entity_custom_fields` already existed from earlier work; the missing piece was the **writer path** — Phase 1 added the MCP tools and UI wiring that actually put data in it.

### 1.2 — Write-path MCP tools

| Tool | Effect |
|---|---|
| `update_entity` | name, emails[], phones[], company, jobTitle, urls[], addresses[], socialHandles{} — atomic upsert with `replace_contact_methods` for the many-to-one tables |
| `set_custom_field` / `delete_custom_field` | freeform key/value rows in `entity_custom_fields` |
| `add_note` / `update_note` / `delete_note` | persisted notes (replaces the zustand-only journal) |
| `merge_entities` | multi-id → primary id; cascading rewrite across `mentions`, `extractions`, `thread_participants`, `entity_tags`, `entity_group_members`, `entity_relationships`, `entity_debts`, `entity_notes`, `entity_signature_history`; writes one `entity_merge_log` row with the conflict-resolution choices |
| `split_entity` | reads `entity_merge_log` → restores the merged_ids → rewrites mentions back |
| `delete_entity` | purges mentions, embeddings, custom_fields, notes, debts, relationships, tags, group memberships; writes an `audit_log` row (right-to-be-forgotten foundation) |
| `undelete_entity` | within a soft-delete window — restores from the audit_log payload |
| `list_pending_deletion` | the soft-delete queue (5-second undo toast on the UI) |

### 1.3 — UI: ContactEditDrawer, AddContactModal, multi-select bar, merge conflict-resolution dialog

The drawer used to write to a zustand `applyLocalOverride` slice. v0.4.0 removed that slice entirely and routed every save through `update_entity`. AddContactModal calls the new `create_entity` tool. The drawer's notes section calls `add_note` / `update_note` / `delete_note`.

ContactList grew a multi-select column (checkboxes, shift-click range, ⌘+A select-all). The bulk-action bar slides up from the bottom with **Merge / Delete / Export / Tag** when ≥ 1 row is selected.

The **merge dialog** is where the user actually decides which fields win:

```
┌────────────────────────────────────────────────────────────┐
│  Merge 3 contacts                                        × │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Name              ● Alice Chen   ○ A. Chen   ○ Alice C.   │
│  Primary email     ● alice@acme   ○ a.chen@gmail           │
│  Company           ● ACME Corp    ○ ACME Co.   ○ —         │
│  Job title         ○ —            ● Senior PM  ○ PM        │
│  Phones            ☑ +1 415…      ☑ +1 650… (de-dup)       │
│  Photo             ● Alice's      ○ generated initials     │
│                                                            │
│  All other contact methods, notes, custom fields, tags     │
│  will be unioned across the 3 records.                     │
│                                                            │
│                       [ Cancel ]  [ Merge into Alice Chen ]│
└────────────────────────────────────────────────────────────┘
```

🖼 Live screenshot — the People surface with the substrate-backed write path:
![People list — Phase 1 data plane](screenshots/01-people-list.png)
![Contact detail — substrate-backed reads](screenshots/01-contact-detail.png)

The merge is *transactional* in the daemon — every mention, tag, debt, and relationship rewrites under one `BEGIN/COMMIT`, or all of them roll back. If the merge fails halfway, you have three contacts again, not 1.7.

---

## Why it took a whole phase

Manual merge isn't difficult to *implement*; it's difficult to make *trustworthy*. The two things that bit us during build-out:

1. **Cascading rewrites are easy to miss a table on.** The first cut missed `thread_participants` and the merged contact's threads dropped them as a participant. The fix was making the cascade table-list a single source of truth in `merge_entities` so every new entity-FK'd table added to migrations gets considered.
2. **`entity_merge_log.merge_choices` has to be enough to fully reverse the merge.** Spec 33 #1.5 requires `split_entity` to reconstruct the pre-merge cards. The choices JSON now snapshots every field's pre-merge value per merged_id, not just the winner — so split puts each card back the way it was.

---

## Developer notes

- The dispatcher's classifier (`READ_ONLY_METHODS` in `substrate_pool.rs`) flags every Phase 1 tool as a **write**, so they go through the writer connection. The People view's `list_entities` / `get_entity` / `entity_timeline` reads go through the reader pool — a 2 s `merge_entities` write will not stall the list refresh that follows.
- `entity_merge_log` is **append-only**. Undoing a merge writes `undone_at` + `undone_choices` and creates a new "split" event in the daemon log; the original row stays.
- The soft-delete window for `delete_entity` is configurable in `~/.config/knowdb/config.toml` under `[people] delete_grace_seconds` (default 5 s, matching the toast).

---

## Acceptance from spec 33 — checked

✅ Merge three duplicate Alice cards into one
✅ Choose which name / email / photo wins
✅ Audit log shows the merge
✅ Undo merge restores all three
✅ Edits persist across daemon restart
✅ Soft-delete with 5 s undo toast
✅ Custom fields write to `entity_custom_fields` and read back on reload
✅ Notes persist via `entity_notes` (and Phase 2 makes them searchable)

---

## What's next in the Phase 1 cluster

The connectivity-contract follow-up — making notes and contact_methods join the search surface — landed in Phase 2 (search refactor). See [`02-tags-and-organization.md`](02-tags-and-organization.md).

The right-to-be-forgotten audit path is wired but the per-contact privacy panel UI (#5.11) is parked until audit-log coverage is baselined across LLM / Graph / SMTP exits. See [`05-data-sovereignty.md`](05-data-sovereignty.md) for the privacy flag that's already in place.
