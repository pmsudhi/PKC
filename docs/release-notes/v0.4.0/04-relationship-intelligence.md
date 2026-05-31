# Phase 4 — Relationship Intelligence (without any LLM)

> *Every signal here is computed from substrate data the daemon already has. No new manual data entry except for the explicit "I want to track this" toggles. No LLM calls. Yet.*

---

## Why no LLM yet

The project's operating principle (see `~/.claude/projects/.../memory/project-people-principles.md`) is **AI is deferred until the deterministic surface is solid.** The reasons:

1. **The substrate has more signal than people realise.** Most "smart contact" features in commercial CRMs are dressed-up SQL. We wanted to prove how far that goes before reaching for a model.
2. **AI features that hide their source are worse than no feature.** The local-first thesis demands we be able to cite every claim. Phase 4 surfaces only computations that can show their work directly in the UI.
3. **Privacy gates first, AI second.** [`05-data-sovereignty.md`](05-data-sovereignty.md) lands the `entity_privacy_flags` table the future LLM router will gate on. Inverting that order would mean some sensitive context could leak before the gate existed.

So Phase 4 is the deterministic upper bound. Everything below comes from `events`, `mentions`, `thread_participants`, and a handful of new tables that the user toggles into existence intentionally.

---

## What shipped — at a glance

```
              ┌───────────────────────────────────────────────┐
              │   Strength score      log-compressed range    │
              │   ─────────────       ─────────────────────   │
              │   is_rising chip      this week ▲ vs last     │
              │   is_dormant chip     last contact > cadence  │
              │   Reply-rate          emailed N× / N replies  │
              │   Job-change cards    sig-block delta detect  │
              ├───────────────────────────────────────────────┤
              │   Keep-in-touch       cadence + auto-ack      │
              │   reminders           on engagement           │
              │   Debts ledger        minor-currency-units    │
              │   Relationships       manual entry; AI later  │
              ├───────────────────────────────────────────────┤
              │   Graph algos         path_between, clusters, │
              │                       influencers             │
              └───────────────────────────────────────────────┘
```

---

## Strength score, fixed

Spec 32 #18 flagged the old linear scorer: the user's top contact scored ~50 000 and the next 999 clustered around 100. Long tail invisible. Phase 4 replaces the linear sum with a **log-compressed** formula:

```rust
score = log(1 + mentions_30d) * 30
      + log(1 + threads_30d)  * 30
      + recency_bonus            // 50 if contact ≤ 7d, 25 if ≤ 30d, 0 otherwise
```

The result: top contacts cluster around 250, dormant contacts cluster around 30, the long tail is **uniformly distributed** between 30 and 200 instead of squashed against zero. Tests in `mcp::tools::people_tests::list_entities_score_is_log_compressed` guard the dynamic range.

---

## `is_rising` — derived in one CTE

`is_rising` is true when this week's mention count exceeds last week's by ≥ 50 %. No worker, no schema — a CTE on `mentions` joined to the calendar week boundary inside `list_entities`. The chip appears in the contact row and on ContactDetail.

---

## Dormant chip — the engagement decay signal

`is_dormant` flips true when last-contact > `cadence_days` (default 90, configurable globally and per-contact via the keep-in-touch reminder). A red dormant chip on the row, visible alongside the strength meter so users see who's slipping before they're surprised by it.

```
┌──────────────────────────────────────────────────────────────────┐
│ ◯ DK  Devon Liu                                                  │
│       devon@gnostic.io · Gnostic Labs                            │
│       Strength ▰▰▱▱▱▱▱   ● dormant 67d                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Keep-in-touch reminders

The first opt-in feature in this phase. Pick Alice → "Keep in touch every 30 / 60 / 90 / 180 days." The daemon writes a row to `entity_reminders`:

```sql
CREATE TABLE entity_reminders (
  entity_id     UUID PRIMARY KEY REFERENCES entities(id),
  cadence_days  INTEGER NOT NULL,
  last_acked_at TIMESTAMP,
  next_due_at   TIMESTAMP NOT NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT NOW()
);
```

A nightly job bumps `next_due_at` for any reminder that's been auto-ack'd by engagement (you sent / received an email or attended a meeting since `last_acked_at`). The "Due to ping" widget on the People sidebar lists overdue reminders.

```
┌─────────────────────────────────────────┐
│  Due to ping                            │
├─────────────────────────────────────────┤
│  ● Eva Park        12 days overdue      │
│  ● Mira Khan        7 days overdue      │
│  ● Sam Albright     3 days overdue      │
│  ▸ Show all (5)                         │
└─────────────────────────────────────────┘
```

🖼 Live screenshot — People surface with Due-to-ping, Job-changes, dormant chip on Devon Liu, and rising chip on Alice (the same Phase-4 fixture used by `phase4-intelligence.spec.ts`):
![Phase 4 — relationship intelligence on the People surface](screenshots/04-relationship-intelligence.png)

**Auto-ack is the differentiator.** Most contact CRMs nag you to confirm "yes, I talked to them" — KnowDB sees the inbox, sees the calendar attendance, and just *knows*. Cadence resets without input.

---

## Reply-rate tracker

A simple ratio computed on the fly: outbound vs inbound counts in the last `lookback_days` (default 30). When you're emailing Alice 5× per reply, that's worth knowing before you send the sixth follow-up.

```
Alice Chen
  ─────────────
  Reply rate (30d)  3 / 5   ↘ slow            // 5 outbound, 3 inbound
  Reply rate (30d)  4 / 4   ◯ balanced
  Reply rate (30d)  5 / 1   ↘ one-sided        // chip turns red
```

The MCP tool: `reply_rate(entity_id, lookback_days?)`.

---

## Job-change detection

The signature-block worker (daemon-side, scheduled hourly) parses the `From:` block of incoming email signatures, extracts company + job title, and stores a row in `entity_signature_history`:

```sql
CREATE TABLE entity_signature_history (
  id          UUID PRIMARY KEY DEFAULT uuid(),
  entity_id   UUID NOT NULL REFERENCES entities(id),
  company     VARCHAR,
  job_title   VARCHAR,
  first_seen  TIMESTAMP NOT NULL,
  last_seen   TIMESTAMP NOT NULL,
  source_event_id UUID
);
```

When company shifts across a 30-day window for the same entity, the daemon raises a **job-change** card on the contact:

```
┌─────────────────────────────────────────────────────────────────┐
│  ✦ Job change detected                                          │
│    Alice moved from ACME Corp (PM) → Stripe (Senior PM)         │
│    First seen at new role: 12 days ago                          │
│    [ Acknowledge ]   [ Update company on profile ]              │
└─────────────────────────────────────────────────────────────────┘
```

🖼 Live screenshot — Alice's ContactDetail with job-change banner, reply-rate, debts, and relationships:
![Phase 4 — contact detail with intelligence cards](screenshots/04-contact-detail-intelligence.png)

Rule-based regex extraction, no LLM. The accuracy is bound by signature-block consistency — false positives happen when a contact emails from a personal address and the regex misreads their hobby project as a new company. The "Acknowledge" button writes a `dismissed_at` so the same change doesn't keep raising.

---

## Debts ledger

The grown-up version of "I'll buy next time."

```sql
CREATE TABLE entity_debts (
  id            UUID PRIMARY KEY DEFAULT uuid(),
  debtor_id     UUID NOT NULL,      -- who owes
  creditor_id   UUID NOT NULL,      -- who is owed
  amount_minor  BIGINT NOT NULL,    -- 1234 cents = $12.34
  currency      VARCHAR(3) NOT NULL,
  note          VARCHAR,
  settled_at    TIMESTAMP,
  created_at    TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Either side of the dyad can be `entity_id = 'self'` (a sentinel for the user). The ledger card on ContactDetail shows owed vs owing, settle / unsettle buttons:

```
┌────────────────────────────────────────────────────┐
│  Debts                                             │
├────────────────────────────────────────────────────┤
│  Alice owes you                                    │
│    $40.00   Lunch — Tartine (Mar 12)   [ Settle ]  │
│                                                    │
│  You owe Alice                                     │
│    $80.00   Concert tickets (Apr 28)   [ Settle ]  │
└────────────────────────────────────────────────────┘
```

<!-- Debts ledger close-up pending — drop `04-debts-ledger.png` into screenshots/. -->


Stored in *minor units* (cents / pence / paise) — no floating point. The settle action stamps `settled_at`; the row stays for audit.

---

## Manual relationships

Family / team / vendor / report-to — captured manually for now. AI-extracted relationships are deferred to a later spec.

```sql
CREATE TABLE entity_relationships (
  id            UUID PRIMARY KEY DEFAULT uuid(),
  entity_a_id   UUID NOT NULL,
  entity_b_id   UUID NOT NULL,
  relation_type VARCHAR NOT NULL,   -- 'family' | 'team' | 'vendor' | 'reports_to' | …
  source        VARCHAR NOT NULL DEFAULT 'user_manual',
  created_at    TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Relationships render on ContactDetail as a short list, and feed into the graph view (Phase 6) where they appear as labeled edges separate from the email-derived edges.

---

## Graph algorithms (preview — surfaced in Phase 6)

Three MCP tools added in this phase, consumed by the graph view in [`06-network-graph.md`](06-network-graph.md):

- `path_between(a_id, b_id, max_depth=4)` — BFS over the mentions graph, returns the chain of intermediaries.
- `cluster_by_domain()` — groups contacts by email domain, returns cluster sizes + representatives.
- `list_influencers(limit=20)` — ranks by `find_related` output size; surfaces the connectors in your network.

---

## What it feels like on Monday morning

Open KnowDB. The sidebar shows three overdue reminders. The People list shows one contact with a 📈 *rising* chip — someone you've started emailing more often without noticing. ContactDetail on a teammate shows a 🟢 *job change* banner — they moved companies last week. The debts card on a friend's profile shows $40 you owe them. Five seconds of glanceable signal. No model called.

---

## MCP surface (full)

```
set_reminder(entity_id, cadence_days)
delete_reminder(entity_id)
ack_reminder(entity_id)
list_reminders_due(within_days?)
get_reminder(entity_id)

reply_rate(entity_id, lookback_days?)

rescan_signatures(force?)           # backfill / rerun the sig-block worker
get_signature_history(entity_id)
list_job_changes(window_days?)

list_relationships(entity_id?)
add_relationship(a, b, type)
delete_relationship(id)

list_debts(entity_id?)
add_debt(debtor, creditor, amount_minor, currency, note?)
settle_debt(id)                     # idempotent — stamps settled_at

path_between(a, b, max_depth?)
cluster_by_domain()
list_influencers(limit?)
```

---

## Developer notes

- The keep-in-touch auto-ack heuristic runs **after** the nightly delta-sync so engagement signals from the last 24 h are visible before the bumper fires.
- `entity_signature_history` dedupes on `(entity_id, company, job_title)` so a stable signature doesn't bloat the table.
- `list_influencers` runs in O(n + e) over the mentions graph but is cached for 5 min per request (in-memory) — it's the slowest read tool. If you call it inside a hot loop, batch.
- The reply-rate counts outbound vs inbound from the user's perspective using `source_account` on events, so multi-account users get aggregate counts across all their addresses.

---

## Acceptance from spec 33 — checked

✅ Strength score log-compressed; long tail visible
✅ `is_rising` chip on rows
✅ `is_dormant` chip on rows
✅ Keep-in-touch reminders with cadence + auto-ack
✅ Reply-rate tracker on ContactDetail
✅ Job-change detection + banner card + acknowledge
✅ Debts ledger with settle/unsettle, minor-unit math
✅ Manual relationships, queryable in graph
✅ Graph algorithms (`path_between`, `cluster_by_domain`, `list_influencers`)
✅ Cross-account dedup chip ("This person is on Gmail + Outlook")

---

## Cross-references

- See [`06-network-graph.md`](06-network-graph.md) for how the three graph algorithms surface visually.
- See [`02-tags-and-organization.md`](02-tags-and-organization.md) for smart groups that filter on `is_dormant` / `is_rising` / strength range.
- See [`05-data-sovereignty.md`](05-data-sovereignty.md) for the privacy flag that the future LLM router will gate on, before any of this signal is shared externally.
