<div align="center">

# 🛡️ KnowDB v0.4.1 — *Hardening the People Hub*

### *No new features. Sixty-one commits of rigor underneath the ones you already have.*

**A panic firewall so one bad tool call can't take down the daemon. A dependency tree with zero known vulnerabilities and a CI gate that keeps it that way. UUID lookups 16× faster, a binary 46% smaller, a first paint 43% lighter. And the full React-side review — ~50 Critical findings closed across six categories — folded back into the surfaces you use every day.**

<br/>

[![Status](https://img.shields.io/badge/status-alpha-yellow?style=flat-square)](#)
[![Platform](https://img.shields.io/badge/platform-Linux-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![Tests](https://img.shields.io/badge/tests-409%20passing-brightgreen?style=flat-square)](#-the-numbers)
[![Vulns](https://img.shields.io/badge/known%20vulns-0-brightgreen?style=flat-square)](#-security--vulnerabilities)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#)

</div>

---

## 🎯 What this release is

v0.4.0 shipped seven People phases back-to-back. That pace bought a great product surface and a stack of non-functional debt: a daemon that aborted the whole process on a single bad input, a dependency tree no one had scanned, hot-path queries no one had measured at scale, and a 216-file React codebase that had never had a principal-level review.

**v0.4.1 is the rigor pass.** Same features, audited along six axes — reliability, security, vulnerabilities, best-practices, scalability, resource use — one slice at a time, each finished and benchmarked before the next began. The operating contract for the branch was explicit: **no new features, no new views, no new tables** unless a stabilization task demanded one.

If v0.4.0 was the demo that proved the thesis, v0.4.1 is the engineering that makes it hold up under daily use.

---

## 🧯 Reliability — the daemon stops crashing

The biggest behavioral change: **a single malformed input can no longer abort the daemon.**

Before v0.4.1 the process was built with `panic = "abort"`. Any panic anywhere — a non-char-boundary string slice in snippet truncation, a duplicate-id batch upsert, an unexpected shape from a connector — killed every connection, every in-flight sync, the whole process. Several real abort vectors were live in production code.

This release closes them at the architecture layer:

- **Panic firewall.** The build switched to `panic = "unwind"`, and every MCP tool dispatch is wrapped in `catch_unwind`. One bad tool call now returns an error to that one caller — the daemon, every other connection, and every background sync keep running.
- **Supervised workers.** Long-running workers (IMAP IDLE, Graph poll, normalize, calendar, archive seeder) spawn through `supervised_spawn`, which restarts them on panic instead of leaving a dead loop and a silently stalled pipeline.
- **Specific abort vectors fixed.** Char-safe (not byte-offset) snippet truncation in `event_context`; dedup-by-id in the substrate batch upsert; the remaining audited daemon-abort paths closed one by one.
- **Connector resilience.** Graph and CalDAV HTTP calls now retry with backoff and classify errors (auth vs transient vs fatal) instead of treating every failure the same. Token headers are attached once, Graph cursors are isolated per account, and calendar pagination no longer drops the tail page.
- **Panic hook → tracing.** Panics that do happen are captured to the structured log with full context rather than vanishing to stderr.

---

## 🔒 Security & vulnerabilities

A from-scratch security and dependency review, plus the CI gates to keep the posture from regressing:

- **Zero known vulnerabilities.** `cargo-audit` over 485 crate dependencies and `npm audit` over the UI tree both come back clean. Two live advisories were fixed to get there: **RUSTSEC-2026-0141** (Rust dependency) and **GHSA-q8mj-m7cp-5q26** (the `qs` prototype-pollution / DoS in the UI toolchain).
- **MCP transport hardened.** The MCP server moved fully onto a Unix domain socket (mode `0600`), closing the network-exposure and auth gaps flagged as S2 + R1.
- **Audit-before-send, enforced.** The outbound-mail path writes its `audit_log` row *before* the bytes leave, per the non-negotiable invariant — now covered by audit-log smoke tests so it can't quietly regress.
- **XSS surfaces closed.** The React review flagged seven injection / privacy-leak surfaces (S1–S7) across rich-body rendering, query-client retry policy, and the frontier-call confirmation flow. All closed.
- **Input validation at the boundary.** UTF-8-safe vCard import, safe blob-path construction, a PowerShell-escape fix on the one shell-touching path, and config that fails loud instead of silently falling back.
- **Supply-chain gates.** New `cargo-deny` config (advisories + bans + licenses + sources) and a CI `audit` workflow running `cargo-audit` + `cargo-deny` + `npm audit` on every push.

---

## ⚡ Performance & scalability

Every claim here was measured before and after.

| Change | Before | After |
|---|---|---|
| Stripped daemon binary | 63 MB | **34 MB** (size-tuned release profile) |
| UUID primary-key lookup | `::VARCHAR = ?` cast antipattern | **16× faster** (cast eliminated) |
| UI first paint | monolithic bundle | **43% smaller**, cytoscape lazy-loaded |

- **Killed the `::VARCHAR` cast antipattern.** Comparing a `UUID` column against a string parameter forced DuckDB to cast every row before matching — a full-scan-shaped cost on what should be a point lookup. Binding the parameter as a native UUID makes the index do its job: **16× faster** PK lookups, holding the `< 5 ms` budget.
- **N+1 collapses across the MCP surface.** `path_between`'s BFS and `cluster_by_domain` were issuing a query per hop / per node; `has_unread` was computed in application code per thread. All three were pushed into single set-based SQL statements. Bounded the remaining per-row scans so they can't blow up on large entity histories.
- **Writer-starvation fix.** The graph batch write now runs under `block_in_place`, so a big graph commit on the writer runtime no longer starves the cross-runtime readers (the same reader-pool architecture from v0.3.1).
- **Allocation diet.** Dropped a double-serialize on the hot draft path, collapsed `get_draft` to a single row fetch, and added a CID early-return so inline-image resolution skips work when there's nothing to resolve.
- **Smaller, split UI bundle.** Code-splitting drops the main chunk to **425 kB (122 kB gzipped)** and moves the 566 kB Cytoscape graph engine into its own lazy chunk that only loads when you open the network graph.

**Latency budgets, all green** (release build, gated in CI):

```
count_query_under_budget          < 50 ms   on 10 000 events   ✓
inbox_top_50_under_budget         < 100 ms  on 50 000 events   ✓
lookup_by_id_under_budget         < 5 ms    on 10 000 events   ✓
try_clone_cold_query_under_budget < 20 ms   on 5 000 events    ✓
pk_lookup_uses_uuid_cast_pattern  (guards the antipattern)     ✓
```

---

## ⚛️ The React review campaign

All 216 React/TypeScript source files went through a principal-engineer review (per-file subagents, the canonical prompt, six waves). Findings roughly doubled from the first pass to **~50 Critical**, then landed as one focused commit per category — the same one-category-per-commit cadence the rest of the branch used:

| Bucket | Findings | What it covered |
|---|---|---|
| **Security** | S1–S7 | XSS surfaces, query-client retry policy, frontier-call confirmation flow |
| **Data-loss** | D1–D12 | Silent-write hazards, derived-state desync, persistence + hydration, frozen-"today" key |
| **Calendar / TZ** | T1–T11 | Timezone + DST hazards, picker→UTC at the wire boundary, cache stability, modal remount |
| **Render-perf** | P1–P8 | Render-phase side effects, memo-defeat, per-field selectors |
| **IME / a11y** | I1–I13 | Keyboard-guard sweep, IME composition, nested-button + role/label fixes |
| **One-shot** | O1–O7 | Daemon-guard cascade, unified avatar palette, route-stub URL↔store sync, boot/ErrorBoundary |

Alongside the fixes, three structural sweeps tightened the codebase without changing behavior:

- **Sweep A** — per-field Zustand selectors, killing whole-store subscriptions that re-rendered the world on any change.
- **Sweep B** — inbox / contact / calendar writes moved onto `useMutation`, so optimistic updates and error surfacing go through one path.
- **Sweeps F + G** — a typed `McpTools` registry + `mcpCall` wrapper (compile-time arg/result inference for 25+ tools) and the split of the monolithic `lib/contacts.ts` / `lib/tags.ts` into feature submodules.

The review pattern itself is now a reusable `/react-review` skill, shipped both user-global and in-repo so a fresh clone discovers it.

---

## 🚦 The gates that keep it green

Stabilization only sticks if regressions get caught automatically. This branch built `scripts/stabilize.sh` — one command that runs every gate the audit produced:

```bash
scripts/stabilize.sh          # all gates, ~3 min
scripts/stabilize.sh --fast   # skip release + UI build, ~1 min
scripts/stabilize.sh --heavy  # add the 100 k-event EXPLAIN probe, ~5 min
```

It runs `cargo-audit` + `cargo-deny` + `npm audit`, clippy at `-D warnings`, eslint (with `eslint-plugin-react` + `jsx-a11y`), the daemon + UI test suites, the latency budgets, the supervisor + Unix-socket tests, and the release build + 50 MB binary-size budget. The **same gates run in CI** (`audit.yml` + `lint.yml`) so local and CI never drift.

---

## 📊 The numbers

| | v0.4.1 |
|---|---|
| Daemon test suite | **137 passing** |
| UI test suite (vitest) | **272 passing** (35 files) |
| Latency budgets | **5/5 green** |
| Known vulnerabilities | **0** (485 Rust deps + UI tree scanned) |
| clippy | **clean at `-D warnings`** |
| eslint | **0 errors** (54 warnings, under ceiling) |
| Daemon binary | **34 MB** stripped (was 63 MB) |
| Critical React findings closed | **~50** across 6 categories |

---

## 📦 Requirements

Unchanged from v0.4.0:

- Linux with GNOME Online Accounts ≥ 3.50
- D-Bus session bus active
- Rust ≥ 1.85 to build
- Node ≥ 20 + npm for the Tauri UI
- A Gmail account (IMAP enabled) and/or a Microsoft 365 account

**No database migration required.** v0.4.1 changes no schema — upgrading from v0.4.0 is a drop-in replacement of the daemon + UI.

---

## 🚧 What's next

v0.4.1 closes the stabilization sprint. With the foundation hardened, the roadmap returns to the spec backlog:

- **Phase 2 proper** — the embedding pipeline (`bge-small-en-v1.5`) and DuckDB VSS HNSW index, then hybrid BM25 + ANN search behind the `< 100 ms` p95 budget on a 100 k-event corpus.
- **The parked People-plan slices** — see the parked-items ledger for what was deferred and why.
- **Mock IMAP server tests** — the Tier-2 protocol-level coverage still outstanding.

---

## 🤝 Closing note

Nothing in v0.4.1 shows up as a new button. What it changes is everything underneath: a daemon that survives bad input, a dependency tree that's been scanned and gated, queries that hold their budgets at scale, and a UI that's been read line-by-line by a reviewer that doesn't get tired.

The data stays on your machine. The architecture, now, stays out of your way under load too.

— on to Phase 2.
