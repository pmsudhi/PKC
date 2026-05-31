<div align="center">

# 🧠 KnowDB

### *Your inbox, your calendar, your contacts — unified, searchable, and yours alone.*

**A local-first AI assistant that knows your information as well as you do, and never sends it anywhere you didn't ask.**

<br/>

[![Status](https://img.shields.io/badge/status-pre--build-orange?style=flat-square)](#-roadmap)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20soon-blue?style=flat-square)](#-requirements)
[![Language](https://img.shields.io/badge/built%20with-Rust-brown?style=flat-square&logo=rust)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/license-TBD-lightgrey?style=flat-square)](#-license)
[![Data stays](https://img.shields.io/badge/your%20data-stays%20on%20your%20machine-green?style=flat-square)](#-privacy-first)

<br/>

> *"What did I commit to this week, and to whom?"*
> *"What's the latest on the Acme renewal? Pull the thread, the meeting notes, the pricing we discussed."*
> *"Who haven't I responded to in five days that I should have?"*
>
> **KnowDB answers these. Without sending your inbox to anyone.**

</div>

---

## 🤔 The Problem Nobody Has Actually Solved

You live across **5–15 information channels** simultaneously. Email threads. Calendar invites. Slack conversations. Shared docs. Every project you're working on is scattered across all of them, and reconstructing context means opening five apps and doing the cross-referencing yourself.

The AI assistants that promise to fix this — Microsoft Copilot, Google Gemini, Shortwave, Notion AI — all share one uncomfortable requirement: **your mailbox has to live on their servers.**

For a lawyer, a consultant, a founder, a researcher, an engineer who's spent years building their network — that's often a non-starter. Either professionally, legally, or just as a matter of principle.

And so the choice has been: powerful AI assistance, or privacy. Not both.

**KnowDB is the "both."**

---

## 💡 What KnowDB Actually Is

KnowDB is a **local-first AI assistant** that runs entirely on your machine.

It ingests your email, calendar, and contacts into a single searchable substrate — a unified DuckDB database that lives in `~/.local/share/knowdb/` and never goes anywhere else. A conversational AI layer sits on top of it, powered by local embedding models and (optionally, explicitly, auditably) frontier models like Claude or GPT.

You get answers like this, all grounded in your actual data with source citations:

| Question | What KnowDB does |
|---|---|
| *"What did Priya say about the renewal terms last quarter?"* | Hybrid semantic + keyword search across 500K+ messages in under 100ms |
| *"Summarize this 40-message thread before I join the meeting"* | Local LLM summarization with every claim linked to a source message |
| *"What do I owe people, and what do they owe me?"* | Commitment extraction from email threads, tracked across time |
| *"Draft a reply that addresses her three concerns"* | AI draft grounded in thread context — you review, you send |
| *"Show me everything about the Acme account"* | Entity view: all emails, meetings, contacts, commitments, in one place |

The AI never hallucinates mail content. Every answer cites the underlying messages. If there's no evidence, there's no answer.

---

## 🔒 Privacy First — For Real This Time

Three commitments that are architectural, not marketing:

**① Your data never leaves your machine.**
There is no cloud backend. There is no multi-tenant infrastructure. There is no vendor server that could be breached, subpoenaed, or trained on. The database is a single file you control.

**② AI works offline by default.**
Embedding, search, extraction, and local chat all run on your laptop — no internet required. Frontier model calls (Claude, GPT) are opt-in, scoped to retrieved snippets only, and every single outbound call is recorded in a local audit log with the exact bytes that were sent.

**③ You can query your data directly.**
The substrate is a plain DuckDB file with a documented schema. The AI surface is exposed via [MCP](https://modelcontextprotocol.io) — the same protocol Claude Desktop uses. External tools can read your data with the same tools the UI uses. No lock-in.

```
Your data  →  ~/.local/share/knowdb/substrate.duckdb  →  stays there
                                                      ↓
                                             MCP tools (search, get, draft)
                                                      ↓
                                       Your UI  |  Claude Desktop  |  Cursor
                                                      ↓  (only when YOU ask)
                                          Anthropic / OpenAI / Google
```

---

## ⚙️ Architecture — How It's Built

KnowDB is a **single Rust daemon** running as a systemd user service. No database server to manage. No Python runtime. No Electron bloat. Just a binary that starts in under 500ms and idles at under 100MB of RAM.

```
┌─────────────────────────────────────────────────────────────┐
│                       Your Machine                          │
│                                                             │
│   ┌──────────────┐      Unix socket (MCP)                  │
│   │  Tauri UI    │◀──────────────────────┐                 │
│   └──────────────┘                       │                 │
│                              ┌───────────┴──────────────┐  │
│   ┌──────────────┐           │    Rust Daemon            │  │
│   │ Claude Desktop│◀────────▶│  ┌────────────────────┐  │  │
│   │ Cursor, etc. │  stdio    │  │ Connectors         │  │  │
│   └──────────────┘           │  │ EDS · IMAP · Graph │  │  │
│                              │  ├────────────────────┤  │  │
│                              │  │ Pipeline Workers   │  │  │
│                              │  │ embed · extract    │  │  │
│                              │  │ normalize · resolve│  │  │
│                              │  ├────────────────────┤  │  │
│                              │  │ LLM Router         │  │  │
│                              │  │ local / frontier   │  │  │
│                              │  ├────────────────────┤  │  │
│                              │  │ MCP Server (rmcp)  │  │  │
│                              │  └────────────────────┘  │  │
│                              │         │                 │  │
│                              │  ┌──────┴───────┐        │  │
│                              │  │  substrate   │        │  │
│                              │  │  DuckDB      │        │  │
│                              │  ├──────────────┤        │  │
│                              │  │  state       │        │  │
│                              │  │  SQLite      │        │  │
│                              │  ├──────────────┤        │  │
│                              │  │  blobs/      │        │  │
│                              │  │  sha256 store│        │  │
│                              └──┴──────────────┴────────┘  │
└─────────────────────────────────────────────────────────────┘
                                    │ (only when you explicitly ask)
                             Frontier provider
                          (Anthropic · OpenAI · Google)
```

---

## 🛠️ The Stack

Chosen for reliability, performance, and the ability to ship a single binary you can actually give to someone.

### Backend — Rust all the way down

| What | Choice | Why it matters |
|---|---|---|
| 🦀 Language | **Rust** | Long-running daemon with sensitive data — memory safety, no GC pauses, no memory leaks |
| ⚡ Async | **Tokio** | Multi-threaded async runtime; handles concurrent connectors, workers, and MCP clients |
| 🗄️ Substrate | **DuckDB** | Embeddable analytical DB — hybrid vector + full-text search in one file, no server needed |
| 📋 State | **SQLite** | Work queues, sync cursors, settings — high-frequency small writes separated from substrate |
| 🔌 D-Bus | **zbus** | Pure-Rust, async D-Bus — talks to GNOME's Evolution Data Server and Online Accounts |
| 📬 IMAP | **async-imap** | Pure-Rust IMAP with IDLE; fetches mail directly from your server |
| 🧮 Embeddings | **candle** | HuggingFace's Rust ML framework — `bge-small-en-v1.5` runs in-process, no Python sidecar |
| 💬 Local LLM | **llama-cpp-2** | GGUF quantized models (Qwen 2.5 7B) — full offline chat when you want it |
| 🤖 MCP | **rmcp** | Official Rust MCP SDK — exposes the tool surface to UI and external clients |
| 🔑 Secrets | **keyring** | Delegates to GNOME Keyring / macOS Keychain / Windows Credential Manager |

### UI — Tauri

Web-tech velocity, native performance. The Tauri Rust backend is a thin MCP client — it talks to the daemon over a Unix socket rather than bundling daemon logic in-process. This keeps the daemon alive and indexing when you close the UI.

### Models — Sensible defaults, fully configurable

| Role | Default | What you can swap it for |
|---|---|---|
| 🔍 Embedding | `bge-small-en-v1.5` (384-dim, 130MB) | `nomic-embed-text-v1.5` (768-dim), `bge-m3` (multilingual) |
| 🧩 Extraction | `Qwen2.5-1.5B-Instruct` GGUF Q4 | `Llama-3.2-3B`, `Phi-3.5-mini` |
| 💬 Local chat | `Qwen2.5-7B-Instruct` GGUF Q4 | `Llama-3.1-8B`, `Mistral-7B` |
| 🌐 Frontier | Anthropic Claude | OpenAI GPT-class, Google Gemini |

---

## 🗂️ What's in the Substrate

Everything lives in one place: `~/.local/share/knowdb/substrate.duckdb`

The schema was designed from day one to be **source-agnostic** — adding Slack or Teams in v2 requires zero schema changes. The `source` column is a discriminator; source-specific data lives in a `metadata JSON` field.

```
events          → every atomic thing across all sources (emails, meetings, contacts)
threads         → conversation grouping, works for email and Slack equally
entities        → resolved people, companies, projects, deals
mentions        → entity ↔ event links with confidence scores
extractions     → commitments, action items, decisions (versioned — reprocessing safe)
embeddings      → 384-dim vector representations with model version
audit_log       → immutable record of every outbound AI call, with payload hash
```

Every extraction row carries `model_version` and `prompt_version`. When a better model ships, reprocessing writes new rows — old rows are marked superseded, never deleted. Rollback is always possible.

---

## 🚀 Roadmap

| Phase | What ships | Status |
|---|---|---|
| **Phase 0** — Foundation | Cargo workspace · DuckDB substrate · SQLite state · `knowdb init` · `knowdb doctor` | 🔜 Next |
| **Phase 1** — Email | EDS + IMAP connector · MIME parsing · Calendar + Contacts sync · `knowdb query` CLI | ⏳ Planned |
| **Phase 2** — Search | `bge-small` embedding pipeline · HNSW index · Hybrid vector + FTS search | ⏳ Planned |
| **Phase 3** — Intelligence | Commitment extraction · Entity resolution · `knowdb reprocess` | ⏳ Planned |
| **Phase 4** — Assistant | MCP server · LLM router · Anthropic + local model · Audit log · `knowdb chat` | ⏳ Planned |
| **Phase 5** — UI | Tauri app · Inbox · Chat pane · Privacy panel · Reconciliation queue | ⏳ Planned |
| **Phase 6** — Ship | systemd service · `.deb` + `.AppImage` · Performance CI gates · Docs · v1.0 | ⏳ Planned |
| **Phase 7+** | Microsoft 365 Graph connector · Slack · macOS · Cross-device sync | 🔭 Future |

---

## 📐 Principles — What This Product Will Never Do

These aren't preferences. They're constraints that shape every technical decision and every pull request.

🔒 **User content never leaves the machine** — except for (a) frontier LLM calls the user explicitly invokes (snippets only, never bulk), (b) mail/calendar actions the user explicitly takes, and (c) inbound source sync. No other egress path exists.

📝 **Every frontier LLM call is audited** — the exact bytes sent are written to a local file before the request is sent. Status is updated after. The user can review, filter by date, and purge old entries.

📎 **Every AI response cites its sources** — no ungrounded generation. Hallucinated mail content is treated as a critical bug, not an edge case.

↩️ **Actions are reversible or gated** — archiving, moving, drafting: either undoable or requiring explicit confirmation. The AI never sends email autonomously.

🔓 **Open data, open contract** — the substrate is a queryable DuckDB file with a documented schema. Nothing is locked in.

⚡ **Performance is a feature** — hard limits enforced in CI: search over 500K messages in under 100ms (p95), daemon idle RAM under 100MB, cold start under 500ms. No "it's slow because it's doing important work."

---

## 🔧 Requirements

**Target platform (v1):** Linux with GNOME, kernel ≥ 5.15, glibc ≥ 2.35

**Runtime:**
- GNOME Online Accounts (for account discovery; optional for manual IMAP config)
- WebKitGTK 4.1+ (for Tauri UI)
- ~5GB free disk space recommended for a 100K-message mailbox

**Build toolchain:**
- Rust stable (MSRV TBD during Phase 0)
- `cargo`, `cmake`, `pkg-config`, `clang`

**Hardware:**
- 8GB RAM minimum (16GB recommended for 7B local chat model)
- Any modern x86-64 or ARM64 Linux machine

macOS support is planned for Phase 7+. Windows after that.

---

## 📦 Installing

KnowDB ships as a single GUI app that manages its background daemon as a
**systemd user service** (it keeps syncing after you close the window).

Download a package for your distro from the
[**latest release**](https://github.com/pmsudhi/PKC/releases/latest):

```bash
# Debian / Ubuntu / Mint / Pop!_OS — .deb
sudo apt install ./KnowDB_*_amd64.deb

# Fedora / RHEL / openSUSE — .rpm
sudo dnf install ./KnowDB-*.x86_64.rpm

# Anything else — the portable AppImage (self-updates from this repo)
chmod +x KnowDB_*_amd64.AppImage && ./KnowDB_*_amd64.AppImage
```

On first launch the app installs + starts the service for you, and the AppImage
keeps itself current by checking this repo's releases. Manage the service any
time under **Settings → Daemon**.

---

## 📥 About this repository

**PKC is KnowDB's public distribution channel** — it carries signed release
binaries (`.deb` / `.rpm` / AppImage), the updater manifest (`latest.json`), and
per-release notes. **No source code lives here**; development happens in a
separate repository. Each KnowDB release is published in two places — the source
repo and this one — and the in-app updater checks **this** repo for new
versions.

Release notes for every version live under
[`docs/release-notes/`](docs/release-notes).

---

## 🧩 Using KnowDB as an MCP Server

Once the daemon is running, any MCP-aware client can connect to it.

**With Claude Desktop** — add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "knowdb": {
      "command": "knowdb",
      "args": ["--mcp-stdio"]
    }
  }
}
```

Then ask Claude: *"What commitments do I have outstanding this week?"* — Claude will call KnowDB's tools and answer from your actual email.

**Available MCP tools:**

| Tool | What it does |
|---|---|
| `search` | Hybrid semantic + keyword search across all sources |
| `get_thread` | Full email thread with all messages |
| `get_entity` | Everything about a person, company, or project |
| `list_commitments` | Commitments owed to/from you with citations |
| `find_related` | Graph traversal from any event or entity |
| `summarize` | Thread or query summarization (local or frontier) |
| `draft_reply` | Compose a reply — *does not send; sending is your job* |
| `audit_recent` | Recent outbound AI calls for the privacy panel |

---

## 🔭 The Bigger Picture

KnowDB is structurally similar to a **personal Fabric** — the same connective-intelligence pattern that enterprises use to unify their data and ground their AI, applied to an individual's information surface.

Email is the entry vector for the same reason F&B point-of-sale data is the entry vector for enterprise intelligence: it's the richest single source, it's where commitments are made and tracked, and if you can answer hard questions over it, you've proven the substrate works.

The substrate is designed to scale from email → calendar → contacts → Slack → Teams → Drive → Notion. Adding a new source in v2 requires zero schema changes. The same tools, the same entity graph, the same search — just more channels feeding into it.

---

## 🤝 Contributing

The project is in pre-build. The `product-document.md` is the foundational spec — read it before touching architecture. Every architectural decision has a decision log entry in Section 8 with the alternatives that were considered and rejected. If you think a decision is wrong, read that entry first.

**Code conventions:**
- No comments unless the WHY is non-obvious. Well-named code documents itself.
- No half-finished features committed. Complete the thing or don't commit it.
- No error handling for scenarios that can't happen. Trust the type system.
- Performance budgets enforced by criterion benchmarks in CI. No waivers without a decision.

**Claude Code users:** This repo ships `.claude/commands/` with project-specific slash commands for every major development task — `/implement-connector`, `/implement-mcp-tool`, `/create-migration`, `/candle-embeddings`, `/duckdb-patterns`, `/rmcp-server`, `/eds-dbus`, `/tauri-daemon-arch`, and more. Run `/phase-status` to see where things stand.

---

## 📄 License

License TBD — see [Open Question Q2](product-document.md#171-pre-phase-0). MIT/Apache for permissive, AGPL to keep forks open-source. Decision before Phase 0 ships.

---

<div align="center">

**Built by [DataHQ](https://datahq.ai) · Designed to be yours, not ours**

*Your data. Your machine. Your assistant.*

</div>
