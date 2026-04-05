# cmux — Codebase Analysis

## What Is cmux?

cmux is a **native macOS terminal application** built in Swift/AppKit, designed for developers running multiple AI coding agent sessions in parallel (Claude Code, Codex, OpenCode, etc.). It wraps the [Ghostty](https://github.com/ghostty-org/ghostty) terminal engine (via `libghostty`) for GPU-accelerated rendering, adds a sidebar with vertical workspace tabs, a rich notification system, an in-app browser, and a comprehensive CLI/socket API for programmatic control.

The core philosophy ("the Zen of cmux") is to provide composable primitives — terminal, browser, notifications, workspaces, splits — rather than a prescriptive orchestration workflow.

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  cmux macOS App (Swift / AppKit + SwiftUI)                   │
│                                                              │
│  ┌────────────────────┐  ┌──────────────────────────────┐   │
│  │   Sidebar (SwiftUI)│  │  Workspace / Split Panes      │   │
│  │  ContentView.swift │  │  Bonsplit (vendor/bonsplit)   │   │
│  │  - Tab list        │  │  GhosttyTerminalView.swift    │   │
│  │  - Notifications   │  │  BrowserPanelView.swift       │   │
│  └────────┬───────────┘  └──────────────┬───────────────┘   │
│           │                             │                    │
│  ┌────────▼─────────────────────────────▼───────────────┐   │
│  │  TabManager.swift — workspace/tab state              │   │
│  │  TerminalNotificationStore.swift — notification state │   │
│  └──────────────────────────┬───────────────────────────┘   │
│                             │                                │
│  ┌──────────────────────────▼───────────────────────────┐   │
│  │  TerminalController.swift — Unix domain socket server │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
         ▲                         ▲
         │ CMUX_WORKSPACE_ID /     │ OSC 9 / 99 / 777
         │ CMUX_SURFACE_ID env     │ escape sequences
         │                         │
  ┌──────┴────────┐      ┌─────────┴──────┐
  │  cmux CLI     │      │ libghostty     │
  │  (CLI/cmux.  │      │ (ghostty/)     │
  │   swift)     │      └────────────────┘
  └──────────────┘
```

### Key Source Directories

| Path | Description |
|------|-------------|
| `Sources/` | Main Swift app (~20 source files, some very large) |
| `CLI/cmux.swift` | Single-file Swift CLI (~14k lines) |
| `ghostty/` | Git submodule — Ghostty terminal engine (upstream fork) |
| `vendor/bonsplit/` | Git submodule — split-pane layout library |
| `daemon/` | Remote relay daemon for SSH workspaces |
| `web/` | Next.js marketing/docs site |
| `tests/`, `tests_v2/` | Python socket integration tests |

---

## Core Components

### `cmuxApp.swift` + `AppDelegate.swift`
Entry point. `cmuxApp` is the SwiftUI `@main` struct; `AppDelegate` is the NSApplicationDelegate that wires everything together — it creates the `TabManager`, starts the `TerminalController` socket listener, handles macOS notification responses (`UNUserNotificationCenterDelegate`), and manages focus.

### `TabManager.swift`
`@MainActor` `ObservableObject` that owns the array of `Workspace` (tab) objects. Tracks which workspace is selected, handles insertion/deletion/reordering, and holds per-tab state like `agentPIDs` (used to detect active Claude sessions). It also sweeps stale agent PIDs on a timer.

### `Workspace.swift`
Represents a single workspace (tab). Contains:
- A map of panels (terminal or browser panes) keyed by UUID
- Sidebar metadata: git branch, PR number/status, listening ports, custom status entries, progress bar, log
- SSH/remote connection state
- Shell activity state per panel (`PanelShellActivityState`)

### `GhosttyTerminalView.swift`
Bridges `libghostty` to AppKit. Manages `GhosttyApp` and `GhosttyTerminalSurface` objects (the C API types from `ghostty.h`). The critical notification path lives here — it handles the `GHOSTTY_ACTION_DESKTOP_NOTIFICATION` action fired by Ghostty when it receives an OSC notification escape sequence.

### `TerminalController.swift`
Unix domain socket server (~13k lines). Implements two command protocols (see [Socket API](#socket-api)) and enforces an access-mode security policy. Every command is dispatched here; this is where all programmatic control of the app flows through.

### `TerminalNotificationStore.swift`
`@MainActor` `ObservableObject` singleton that holds the list of `TerminalNotification` values. SwiftUI views subscribe to it via `@EnvironmentObject`. It also:
- Fires `UNUserNotificationCenter` system notifications (with customisable sound including custom audio files)
- Updates the dock badge (unread count or tagged-run label)
- Supports cooldown keys to debounce rapid notifications
- Runs a custom shell command hook when a notification fires (`CMUX_NOTIFICATION_TITLE`, `_SUBTITLE`, `_BODY` env vars)

### `ContentView.swift` / `NotificationsPage.swift`
SwiftUI views for the sidebar tab list and the notification panel. Each tab row shows git branch, PR badge, working directory, ports, and the latest notification text. Notification rows link back to the originating workspace/surface.

---

## How Notifications Work

There are two independent paths through which "Claude needs your input" becomes a visible in-app notification ring and system notification.

### Path 1 — OSC Escape Sequences (generic terminal path)

```
Claude Code / any program
  → writes OSC 9/99/777 escape sequence to TTY
  → libghostty parses it, fires GHOSTTY_ACTION_DESKTOP_NOTIFICATION
  → GhosttyTerminalView catches the action
  → suppresses it if workspace.agentPIDs["claude_code"] is set
    (avoids duplicate notifications when the hook path is active)
  → calls TerminalNotificationStore.shared.addNotification(...)
```

The title/body come from the escape sequence payload. This path works for any application (terminal-notifier, Claude Code's built-in notification, etc.) without any configuration.

### Path 2 — Claude Code Hooks (structured, rich path)

Claude Code supports lifecycle hooks — shell scripts or commands invoked at specific points in a session. Users configure cmux as their hook handler:

```json
// ~/.claude/settings.json (example)
{
  "hooks": {
    "Notification": [{ "hooks": [{ "type": "command", "command": "cmux claude-hook notification" }] }],
    "Stop": [{ "hooks": [{ "type": "command", "command": "cmux claude-hook stop" }] }],
    "PreToolUse": [{ "hooks": [{ "type": "command", "command": "cmux claude-hook prompt-submit" }] }]
  }
}
```

Each hook invocation calls the `cmux` CLI binary, which:
1. Reads JSON from stdin (Claude Code writes the hook payload there)
2. Locates the app's Unix socket
3. Sends commands to `TerminalController` over the socket

The hook subcommands map to these behaviors:

| Hook / subcommand | What cmux does |
|---|---|
| `session-start` / `active` | Registers the Claude PID via `set_agent_pid claude_code <pid>` (enables OSC suppression); records session ID→workspace/surface mapping in `~/.cmuxterm/claude-hook-sessions.json` |
| `prompt-submit` | Clears existing notifications for the workspace; sets sidebar status to "Running ⚡" (blue) |
| `stop` / `idle` | Sends a structured completion notification (title: "Claude Code", body: transcript summary); sets status to "Idle ⏸" (grey) |
| `notification` / `notify` | **The main "needs your input" path.** Builds a notification payload from the hook's JSON (falls back to saved transcript summary for more context), calls `notify_target <workspace> <surface> <title>|<subtitle>|<body>` on the socket, sets status to "Needs input 🔔" |
| `session-end` | Clears agent PID, clears status |

### What Happens Inside `addNotification`

```
TerminalNotificationStore.addNotification(tabId:surfaceId:title:subtitle:body:)
  → checks cooldown (debounce)
  → removes previous notification for same tab+surface
  → if app is focused AND this surface is focused: shows a brief "focused read indicator" ring
    (suppresses system notification since user can already see it)
  → if WorkspaceAutoReorderSettings enabled: moves tab to top of sidebar
  → appends TerminalNotification to @Published notifications array
  → fires UNUserNotificationCenter notification (if app not in focus)
  → plays sound (system sound, custom file, or runs custom command)
  → updates dock badge
  → SwiftUI re-renders sidebar tab row (pane ring, tab highlight)
```

### Visual Indicators

- **Pane ring** — Blue border around the specific split pane that triggered the notification (`NotificationPaneRingSettings`)
- **Pane flash** — Brief flash animation on notification arrival (`NotificationPaneFlashSettings`)
- **Sidebar tab** — Tab row in the sidebar shows a blue dot and the notification body text
- **Notification panel** — Cmd+I opens a list of all pending notifications; each can be clicked to jump to the workspace
- **Jump shortcut** — Cmd+Shift+U focuses the most recent unread notification's workspace
- **Dock badge** — Unread count; in tagged debug builds also shows the build tag
- **macOS system notification** — Delivered via `UNUserNotificationCenter`; clicking it activates the relevant workspace

---

## Socket API

`TerminalController` binds a Unix domain socket (default: `~/.config/cmux/cmux.sock` or a path derived from the bundle ID). All commands are newline-terminated.

**Security modes** (configurable in Settings → Socket):
- `cmuxOnly` (default) — only processes that are descendants of the cmux process may connect
- `automation` — any local process belonging to the same macOS user
- `password` — requires a shared secret stored in `~/.config/cmux/socket-control-password`
- `allowAll` — fully open (unsafe)

**Two protocols:**

**V1** — space-delimited text
```
notify <title>|<subtitle>|<body>        → fires notification on current workspace
notify_target <wsId> <sfId> <payload>  → fires notification on specific surface
set_status <key> <value> --tab=<id>    → update sidebar metadata
report_git_branch <branch>             → show branch in sidebar
report_pr <number> <label> <url>       → show linked PR
report_ports <port1,port2,...>         → show listening ports
send <text>                            → send keystroke input
new_workspace <name>                   → create workspace
```

**V2** — newline-delimited JSON (`{"method": "...", "params": {...}, "id": "..."}`)
```json
{"method": "notification.create", "params": {"title": "Done", "body": "Tests passed"}}
{"method": "workspace.create", "params": {"name": "my-project"}}
{"method": "surface.send_text", "params": {"text": "ls\n"}}
{"method": "browser.navigate", "params": {"url": "http://localhost:3000"}}
{"method": "browser.snapshot", "params": {}}  ← accessibility tree snapshot
{"method": "browser.click", "params": {"ref": "e1"}}
```

The V2 browser API is a port of Vercel's [agent-browser](https://github.com/vercel-labs/agent-browser) — Playwright-style automation over a WKWebView (click, fill, eval JS, screenshots, cookies, storage, dialogs, downloads, multi-tab).

### Environment Variable Injection

When cmux spawns a terminal, it sets environment variables in the child process:
- `CMUX_WORKSPACE_ID` — UUID of the hosting workspace
- `CMUX_SURFACE_ID` — UUID of the terminal surface/panel
- `CMUX_SOCKET_PATH` — path to the Unix socket

These allow the `cmux` CLI and hooks to automatically target the correct workspace/surface without explicit flags.

---

## Ghostty Integration

`ghostty/` is a fork of the [Ghostty](https://github.com/ghostty-org/ghostty) terminal emulator, compiled as `GhosttyKit.xcframework` (a universal XCFramework built with Zig). cmux uses the public C API declared in `ghostty/include/ghostty.h` (bridged via `ghostty.h` and `cmux-Bridging-Header.h`).

Ghostty provides:
- GPU-accelerated terminal rendering
- Full terminal emulation (VT100/xterm/etc.)
- Configuration compatibility (`~/.config/ghostty/config` for fonts, colors, themes)
- Action callbacks (title changes, clipboard, bell, **desktop notifications** via `GHOSTTY_ACTION_DESKTOP_NOTIFICATION`)

The `GHOSTTY_ACTION_DESKTOP_NOTIFICATION` action is how Ghostty exposes OSC 9, OSC 99, and OSC 777 escape sequences to the host application. cmux intercepts this action in `GhosttyTerminalView.swift` and routes it to `TerminalNotificationStore`.

---

## Split-Pane Layout — Bonsplit

`vendor/bonsplit` is a custom Swift library (git submodule) implementing the split-pane tree layout. It manages:
- Binary tree splits (horizontal / vertical)
- Drag-and-drop tab reordering and pane movement
- Focus tracking between panes
- `DebugEventLog` — unified debug event log (ring buffer, file output, only in DEBUG builds)

---

## Sidebar Metadata

Each workspace tab in the sidebar can display a rich set of live metadata, all writable through the CLI/socket:

- **Git branch** — `cmux report-git-branch <branch>` (dirty flag supported)
- **Pull request** — `cmux report-pr` with number, label, URL, CI checks status
- **Working directory** — auto-detected from shell, or set via `report_pwd`
- **Listening ports** — auto-scanned by `PortScanner.swift` or set via `report_ports`
- **Status entries** — arbitrary key/value with optional icon, color, URL, priority
- **Progress bar** — `set_progress <value>` (0.0–1.0)
- **Notification text** — last unread notification body shown inline

---

## SSH Workspaces

`cmux ssh user@host` is a CLI subcommand that:
1. Creates a new cmux workspace via socket
2. Opens an SSH connection with a bootstrap script that installs a relay daemon (`daemon/`) on the remote host
3. Configures the workspace as "remote" — browser panes use the remote machine's network context (so `localhost:3000` in the browser points to the remote host's port 3000)
4. Supports image upload by dragging files into the terminal (via `TerminalImageTransfer.swift`)

---

## Claude Code Teams

`cmux claude-teams` invokes Claude Code's multi-agent "teammate" mode, spawning teammate instances as native cmux split panes. It includes a tmux-compatibility shim (the tmux subcommands `capture-pane`, `resize-pane`, etc.) so Claude Code's tmux-based teammate orchestration works transparently through cmux.

---

## Session Persistence

`SessionPersistence.swift` saves and restores:
- Window/workspace/panel layout
- Working directories
- Terminal scrollback (up to 4,000 lines / 400,000 characters, with ANSI-safe truncation)
- Browser URL and navigation history

Live process state (e.g., a running Claude Code session) is **not** restored on relaunch.

---

## Observability & Analytics

- **Sentry** — error reporting in the CLI (`CLISocketSentryTelemetry`); controlled by `CMUX_CLI_SENTRY_DISABLED=1`
- **PostHog** — anonymous product analytics in the app (`PostHogAnalytics.swift`)
- **Debug event log** — in DEBUG builds, all key/mouse/focus/split events are logged to `/tmp/cmux-debug[-<tag>].log` via `dlog()` from Bonsplit's `DebugEventLog`

---

## Build System

- **Xcode project** — `GhosttyTabs.xcodeproj` (the app was originally named GhosttyTabs)
- **Swift Package Manager** — `Package.swift` for the CLI and supporting targets
- **Zig** — used to build `libghostty` / `GhosttyKit.xcframework` (Ghostty is a Zig project)
- **Sparkle** — automatic in-app updates; nightly builds have a separate bundle ID and Sparkle feed
- **Homebrew** — `homebrew-cmux/` contains the cask tap

---

## Summary Data Flow: "Claude needs input" → Pane Ring

```
1. Claude Code process writes "needs your input" event to hook stdout or OSC to TTY

   OSC path:
   2a. libghostty parses OSC 9/99/777
   3a. Fires GHOSTTY_ACTION_DESKTOP_NOTIFICATION callback
   4a. GhosttyTerminalView.handleAction() called
   5a. Checks if workspace.agentPIDs["claude_code"] is set → if yes, suppresses (hook is handling it)

   Hook path:
   2b. Claude Code invokes: cmux claude-hook notification
   3b. CLI reads JSON payload from stdin
   4b. CLI opens Unix socket, sends: notify_target <wsId> <sfId> "Claude Code|<subtitle>|<body>"
   5b. TerminalController.processCommand() dispatches to notifyTarget()
   
6. TerminalNotificationStore.addNotification(tabId:surfaceId:title:subtitle:body:)
7. @Published notifications array updated → SwiftUI re-renders
8. Pane: blue ring drawn around the terminal surface
9. Sidebar tab: notification dot + body text shown inline
10. If app not focused: UNUserNotificationCenter fires system notification + sound
11. Dock badge unread count incremented
```
