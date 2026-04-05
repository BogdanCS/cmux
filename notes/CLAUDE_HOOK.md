How cmux claude-hook notification is implemented

The flow spans three layers:

Layer 1 — The claude wrapper (Resources/bin/claude)

cmux ships a bash wrapper that sits on $PATH ahead of the real claude binary. When you run claude
inside a cmux terminal:

 1. It detects CMUX_SURFACE_ID is set (injected by cmux into every terminal it opens)
 2. Pings the socket to confirm the app is alive
 3. Generates a fresh SESSION_ID via uuidgen
 4. Builds a HOOKS_JSON blob and passes it as --settings to the real claude binary

The hooks JSON includes a Notification hook:

 "Notification": [{ "hooks": [{ "type": "command",
   "command": "\"${CMUX_CLAUDE_HOOK_CMUX_BIN:-cmux}\" claude-hook notification",
   "timeout": 10 }] }]

CMUX_CLAUDE_HOOK_CMUX_BIN is pinned to the bundled CLI binary at startup so hooks use the version that
shipped with the running app, not whatever cmux is on $PATH.

------------------------------------------------------------------------------------------------------

Layer 2 — CLI dispatch (CLI/cmux.swift, runClaudeHook)

When Claude Code fires the Notification hook, it execs the above command with the hook payload as JSON
on stdin. The CLI:

 1. Reads stdin → parseClaudeHookInput() — deserialises the JSON, strips it down to a compact 
representation (field lengths capped, only known keys kept)
 2. Extracts session ID from session_id / sessionId (tried at the top level, in nested notification and
 data objects, and in session.id)
 3. Looks up the session in ~/.cmuxterm/claude-hook-sessions.json (written by session-start) to recover
 the workspace/surface UUIDs even if CMUX_WORKSPACE_ID env is no longer correct
 4. Builds a notification summary via summarizeClaudeHookNotification():
  - Extracts a "signal" string from event/type/kind fields
  - Extracts a "message" from message/body/text/prompt/error/description fields  
  - Classifies into subtitle categories (Permission, Error, Completed, Waiting, Attention) using 
keyword matching on the combined signal+message string
  - Fallback enrichment: if the body is generic ("needs your attention"/"needs your input") and a 
richer body was saved during pre-tool-use (e.g. an AskUserQuestion tool name → actual question text), 
the saved body is substituted
 5. Sanitises the fields (collapses whitespace, replaces | separators with ¦ to avoid protocol 
corruption)
 6. Sends two socket commands over the Unix socket:
  - notify_target <workspaceId> <surfaceId> Claude Code|<subtitle>|<body> → V1 protocol
  - set_status to mark the workspace as "Needs input 🔔" (blue)
 7. Updates the session store with lastSubtitle/lastBody for future enrichment
 8. Prints OK or the raw socket response to stdout

------------------------------------------------------------------------------------------------------

Layer 3 — App (TerminalController → TerminalNotificationStore)

TerminalController.processCommand("notify_target …") dispatches to notifyTarget(), which calls 
TerminalNotificationStore.shared.addNotification(tabId:surfaceId:title:subtitle:body:). This triggers
everything visible: blue pane ring, sidebar highlight, system notification, dock badge.

------------------------------------------------------------------------------------------------------

Testing

There are four dedicated test files for the claude-hook integration. All are Python E2E/integration
tests — not unit tests:

test_claude_hook_session_mapping.py — full happy-path E2E

Connects to a real running cmux app over the socket. Tests the complete three-step lifecycle:

 1. session-start → asserts the session file on disk maps session_id → workspace/surface
 2. notification with type: permission → polls list_notifications until a notification with subtitle 
"Permission" appears, asserts it's routed to the correct surface
 3. stop → asserts a "Completed" notification appears containing the project directory name + "Last:" +
 the message from step 2; asserts session entry is consumed (deleted) from disk

test_claude_hook_missing_socket_error.py — error-path regression

Runs claude-hook stop against a non-existent socket path, asserts exit code is non-zero and stderr
contains a clear "Socket not found" message.

test_claude_hook_stop_ignores_teardown_unavailable.py — resilience regression

Spins up a fake Python socket server that returns ERROR: TabManager not available for every command
(simulating a workspace being torn down mid-session). Asserts that claude-hook stop still exits 0 and
prints OK — it must not surface teardown errors as failures.

test_claude_wrapper_hooks.py — wrapper injection tests

Tests Resources/bin/claude (the bash wrapper) in isolation using a fake claude binary and a fake cmux
binary. Verifies:

 - Correct hook names (SessionStart, Stop, SessionEnd, Notification, UserPromptSubmit, PreToolUse) and 
their subcommand strings are injected into --settings
 - PreToolUse has async: true (so it doesn't block tool execution)
 - SessionEnd has a short timeout (≤2s, since the session is exiting)
 - CMUX_CLAUDE_HOOK_CMUX_BIN is pinned to the bundled CLI
 - NODE_OPTIONS is set with a preload module (for heap cap restoration) and --max-old-space-size=4096
 - Behaviour with stale/live/missing sockets
 - Nested Claude Code sessions are skipped (CLAUDECODE env guard)

cmuxTests/GhosttyConfigTests.swift — settings unit tests

Only tests the ClaudeCodeIntegrationSettings.hooksEnabled UserDefaults preference, not the hook logic
itself.

------------------------------------------------------------------------------------------------------

What's not tested

 - summarizeClaudeHookNotification() / classifyClaudeNotification() — the notification body 
summarisation logic has no unit tests. Edge cases in the keyword classifier and session-store 
enrichment fallback are only exercised by the single E2E happy path.
 - parseClaudeHookInput() and compactClaudeHookObject() — the JSON parsing and field truncation logic 
is not unit-tested independently.
 - pre-tool-use hook (AskUserQuestion enrichment path) — not covered by any dedicated test.
 - session-end — not covered beyond the test_claude_hook_stop_ignores_teardown_unavailable.py side 
effects.
