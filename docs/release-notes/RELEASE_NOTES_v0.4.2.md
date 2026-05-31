<div align="center">

# 🔧 KnowDB v0.4.2 — *Install & read-pane fixes*

### *The CLI is on your PATH, and emails render in full.*

</div>

---

## 🐚 The `knowdb` CLI is now on `PATH`

The `.deb` / `.rpm` previously installed the daemon to `/usr/lib/knowdb/knowdb`,
which isn't on `PATH` — so `knowdb` (the CLI **and** the MCP server binary) wasn't
resolvable. An MCP client configured with `command: knowdb` would fail to connect
with `which knowdb` coming back empty.

- **`.deb` / `.rpm`** now install the binary to **`/usr/bin/knowdb`**, and the
  systemd unit's `ExecStart` points there.
- **AppImage** symlinks **`~/.local/bin/knowdb`** → the copied daemon on first
  run (modern distros put `~/.local/bin` on `PATH`).
- The CI install smoke-test now asserts `command -v knowdb` resolves.

After installing, `knowdb --help`, `knowdb query …`, and `knowdb serve` all work
from any shell. (Note: a stdio MCP transport — `knowdb --mcp-stdio` — is **not**
implemented yet; the server currently speaks JSON-RPC over a Unix socket + TCP
`127.0.0.1:7432`. Stdio support is tracked for the Phase 4 MCP work.)

## 📧 Inbox read pane renders the whole email

Clicking a message rendered only the top sliver of the body (~25%), wasting the
available space. The email is shown in a sandboxed `<iframe>` that auto-sizes to
its content — but on current WebKitGTK a `sandbox=""` frame is opaque-origin, so
the parent couldn't read `contentDocument` to measure height, and the frame
stayed clamped at its 200 px initial size.

- The frame now uses **`sandbox="allow-same-origin"`** (height measurement +
  link-click interception work again). **Scripts stay disabled** — `allow-scripts`
  is absent, and DOMPurify + the `script-src 'none'` CSP remain in place, so the
  security posture is unchanged.

---

## 📦 Requirements & upgrade

Unchanged from v0.4.1. **No schema migration** — upgrading is a drop-in
replacement of the daemon + UI.
