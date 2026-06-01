<div align="center">

# 🩹 KnowDB v0.4.7 — *Senders restored*

### *Inbox items that showed "0 participants / no sender" now show who it's from.*

</div>

---

## The bug

Microsoft Graph's `/messages/delta` occasionally delivers a **real** message —
HTML body and attachments included — with `from` / `sender` / `recipients`
(and sometimes `subject`) **null**. A known delta inconsistency for mail caught
mid-arrival. KnowDB stored those events as-is, so they appeared in the inbox with
**no sender** ("0 participants") and no subject — and the body/attachments only
showed up later once the existing backfills re-fetched them.

## The fix

A new background heal, **`backfill_graph_participants`**: it finds Graph email
events with **empty participants** + a `graph_message_id`, re-fetches the full
message via `get_message` (the default representation, which carries the real
`from`/`to`/`subject`), and stamps the real **participants** (and subject when
present).

- **Re-fetch, not delete** — these are real messages with bodies and
  attachments. An earlier draft of this fix deleted them; testing on a copy of
  the live DB caught that they had attachments, so the approach was changed.
- **Safe** — per-row `UPDATE`s of a handful of rows (like the HTML backfill),
  not a mass update.
- **Automatic + idempotent** — runs at daemon startup; a row drops out of the
  set once it has participants. Existing affected messages heal on first launch.

Validated on a copy of the author's live DB: `healed=3`, senders restored, zero
"no sender" inbox items remaining.

No schema migration. Reinstall and restart the daemon — the heal runs on its own.
