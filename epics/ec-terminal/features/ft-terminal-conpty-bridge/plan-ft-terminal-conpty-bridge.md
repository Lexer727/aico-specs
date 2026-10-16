---
id: plan-ft-terminal-conpty-bridge
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 3
title: "Terminal ConPTY Pseudo-Terminal Backend Bridge Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-terminal-conpty-bridge"]
created_at: "2026-10-10T03:05:00Z"
---

# Implementation Plan: Terminal ConPTY Pseudo-Terminal Backend Bridge

## 1. Executive Summary & Verification Strategy

This plan migrates the terminal subsystem from piped standard streams (`Stdio::piped()`) to native ConPTY pseudo-terminals using `portable-pty`. This enables genuine interactive TUI support, raw key input, and dynamic window resizing. Verification is driven by end-to-end integration tests in `crates/aico-server/tests/conpty_bridge_test.rs`.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

- **TASK-PTY-001**: Add portable-pty dependency and implement PtyManager in aico-server
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-003
  - **Blocked by**: none
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Adds `portable-pty = "0.8"` to `Cargo.toml`.
    - Implements cross-platform PTY abstraction creating native master/slave pairs (Windows ConPTY on Windows) and spawning shell processes directly inside the slave.

- **TASK-PTY-002**: Upgrade WebSocket endpoint with raw byte streaming, dynamic resize, and process cleanup
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-004, REQ-005, REQ-006, REQ-009
  - **Blocked by**: TASK-PTY-001
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Updates `/api/v1/terminal/ws` message handler to process raw `input` bytes, dispatch `resize` events to PTY master, stream output chunks, and terminate process trees on disconnect.

- **TASK-PTY-003**: Implement AI terminal session state tracking
  - **Status**: Done
  - **Addresses**: REQ-008
  - **Blocked by**: TASK-PTY-002
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Tracks active AI terminal sessions and exposes real-time state (`idle`, `waiting_for_input`, `working`) for consumption by status monitors.

- **TASK-PTY-004**: Implement automated ConPTY integration test suite
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-006
  - **Blocked by**: TASK-PTY-003
  - **Verification Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`
  - **What it delivers**:
    - End-to-end integration test validating ConPTY allocation, command execution, raw input forwarding, dynamic resizing, and clean process termination.

- **TASK-PTY-005**: Restore default terminal working directory (cwd) to the AICO Workspace Root
  - **Status**: Done
  - **Addresses**: REQ-003
  - **Blocked by**: TASK-PTY-004
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Updates `resolve_working_dir` in `crates/aico-server/src/terminal.rs` to default to `active.specs_path.parent()` (the AICO Workspace Root where `.agents/skills`, `*-specs`, and `codebase/` reside).
    - Removes the `<project>-specs` `cwd` override in `frontend/src/App.tsx` while keeping clean project names in `ProjectSwitcher.tsx`.

- **TASK-PTY-006**: Strip answered initial ConPTY DSR `\x1b[6n` query from WebSocket output stream and resolve native Windows `claude.exe` path
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-007, REQ-010
  - **Blocked by**: TASK-PTY-005
  - **Verification Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`
  - **What it delivers**:
    - Strips the consumed initial `\x1b[6n` Device Status Report query from `data` in `crates/aico-server/src/terminal.rs` when `answered_dsr` replies with `\x1b[1;1R`, preventing `@xterm/xterm` from sending a duplicate `\x1b[1;1R` into PowerShell's `PSReadLine` stdin buffer (which caused a console bell beep and swallowed the first keystroke).
    - Resolves `%USERPROFILE%\.local\bin\claude.exe` on Windows in `resolve_shell_command(ShellType::Claude)` before falling back to `cmd.exe /c claude`.

- **TASK-PTY-007**: Implement `tab_id`-keyed PTY session persistence, re-attachment, and scrollback buffer replay across WebSocket reconnects
  - **Status**: Done
  - **Addresses**: REQ-006, REQ-011
  - **Blocked by**: TASK-PTY-006
  - **Verification Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`
  - **What it delivers**:
    - Extends `ClientTerminalMessage::Init` with optional `tab_id: Option<String>`.
    - Retains active `PtySession` handles and a rolling UTF-8 output scrollback buffer (up to 128 KB) keyed by `tab_id` in `TerminalSessionRegistry`.
    - Replays buffered scrollback output and re-binds the WebSocket sender when a client reconnects with an existing `tab_id` (such as after `F5` window reload), and terminates the PTY session only upon explicit `close`/`kill` or process exit.
    - Adds integration test coverage in `crates/aico-server/tests/conpty_bridge_test.rs`.

- **TASK-PTY-008**: Implement multi-client PTY output fan-out and concurrent input routing
  - **Status**: Done
  - **Addresses**: REQ-012
  - **Blocked by**: TASK-PTY-007
  - **Verification Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`
  - **What it delivers**:
    - Refactors `client_tx` in `PtySession` (`crates/aico-server/src/terminal.rs`) into a multi-client sender collection (`Arc<Mutex<Vec<mpsc::Sender<ServerTerminalMessage>>>>`).
    - Broadcasts PTY output chunks to all attached WebSocket channels simultaneously, appending to the rolling scrollback buffer.
    - Routes input from any attached client to the shared PTY master writer without disconnecting existing listeners.
    - Adds integration test coverage in `conpty_bridge_test.rs` verifying two clients connected to the same `tab_id`.

---

## 3. Amendment History

- **2026-10-16T22:30:00Z**: Added `TASK-PTY-008` (`REQ-012`) via `/aico-plan-amendment` for multi-client PTY fan-out supporting concurrent desktop and mobile companion connections to the same named terminal tab.
- **2026-10-16T20:58:00Z**: Added `TASK-PTY-007` (`REQ-011`) for `tab_id`-keyed PTY session persistence, re-attachment, and scrollback replay across browser reloads.
