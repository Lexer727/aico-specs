---
id: spec-ft-terminal-conpty-bridge
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 3
title: "Terminal ConPTY Pseudo-Terminal Backend Bridge & Interactive Runtime"
status: "Draft"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-10T03:05:00Z"
---

# Feature Specification: Terminal ConPTY Pseudo-Terminal Backend Bridge & Interactive Runtime

## 1. Problem Statement & User Value

The initial terminal backend used standard operating system pipes (`Stdio::piped()`) to spawn shell processes and execute one-shot commands (`-p`). Because piped standard streams are not interactive pseudo-terminals (`isatty() == false`), interactive CLI programs such as Google Antigravity (`agy`), Claude Code (`claude`), `inquirer`, and ink-based terminal user interfaces (TUIs) cannot engage raw terminal mode. When prompts (such as multiple-choice questions or arrow-key navigation) are presented, the processes fail to capture keystrokes and automatically abort or trigger "User Skipped".

This feature replaces standard piped process execution in `aico-server` with a native Pseudo-Terminal (PTY) engine powered by `portable-pty` (Windows ConPTY on Windows and Unix PTY on POSIX). This provides 100% terminal fidelity, enabling interactive prompts, cursor-addressed TUIs, arrow navigation, and real-time terminal window resizing.

---

## In-Scope

- **Native PTY Engine (`portable-pty`)**:
  - Allocating native PTY pairs (master/slave) via `portable-pty` for every terminal session.
  - Spawning shells inside the PTY slave: `powershell.exe`, `cmd.exe`, Git `bash.exe`, `agy.exe`, and future CLIs (`claude.exe`).
- **Full-Duplex WebSocket PTY Protocol**:
  - Processing `init` messages with requested shell, working directory, and initial dimensions (`cols`, `rows`).
  - Streaming raw PTY output chunks to the WebSocket client.
  - Forwarding raw keystrokes from `input` messages to the PTY master writer without line buffering.
  - Handling dynamic `resize` messages to update ConPTY window dimensions on the fly.
- **Process Lifecycle & Session Management**:
  - Gracefully terminating child process trees and releasing PTY resources when tabs close or sockets disconnect.
  - Tracking AI session activity state (`waiting_for_input`, `working`, `idle`).
- **Automated Integration Tests**:
  - Comprehensive PTY integration test suite in `crates/aico-server/tests/conpty_bridge_test.rs`.

---

## Out-of-Scope

- WebUI terminal rendering components (specified in `ft-terminal-xterm-ui`).
- Multi-host terminal clustering or remote SSH proxying.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer using the AI terminal, I want interactive prompts and multiple-choice questions to accept arrow-key and enter inputs, so that I can make selections without the CLI auto-skipping or failing.
   - *Independent Test*: Spawn an interactive CLI session inside ConPTY, emit raw ANSI cursor requests, assert PTY returns appropriate terminal responses and captures raw keystrokes.

2. **US-02** (Priority: P1): As a developer resizing the terminal panel in my browser, I want the backend PTY to adjust its column and row dimensions, so that output text and TUIs re-flow cleanly without clipping.
   - *Independent Test*: Send a `{ "type": "resize", "cols": 100, "rows": 40 }` message over `/api/v1/terminal/ws`; assert PTY master updates its geometry without error.

3. **US-03** (Priority: P1): As a developer closing a terminal tab, I want the backend to terminate the process tree immediately, so that orphaned PTY sessions do not persist on the host.
   - *Independent Test*: Open a PTY session over WebSocket and close the connection; assert child process terminates.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The PtyManager shall allocate a native pseudo-terminal (Windows ConPTY on Windows, Unix PTY on POSIX) using `portable-pty` for every spawned terminal session.
- **REQ-002**: The TerminalWebSocketHandler shall support bi-directional transmission of raw terminal byte streams and structured control messages over `/api/v1/terminal/ws`.

### Event-Driven Requirements
- **REQ-003**: When a client sends a WebSocket `init` message, the TerminalWebSocketHandler shall spawn the requested shell executable (`powershell.exe`, `cmd.exe`, `bash.exe`, `agy.exe`, or `claude.exe`) attached to the allocated PTY slave.
- **REQ-004**: When a client sends a WebSocket `input` message, the PtyManager shall write the raw input bytes directly to the PTY master writer without line-buffering delay.
- **REQ-005**: When a client sends a WebSocket `resize` message containing column and row dimensions, the PtyManager shall update the PTY master geometry via `PtySize`.
- **REQ-006**: When a WebSocket connection disconnects or receives a `close`/`kill` message, the PtyManager shall terminate the spawned child process tree and release all associated PTY handles.

### State-Driven Requirements
- **REQ-007**: While a child process generates stdout or stderr output inside the PTY, the PtyManager shall stream raw chunks from the PTY master reader to the WebSocket client as `output` messages.
- **REQ-008**: While an AI terminal session is running, the SessionStateTracker shall monitor output patterns to broadcast the AI activity status (`waiting_for_input` vs `working`).

### Unwanted Behavior & Error Handling
- **REQ-009**: If the requested shell binary is not found on the host system, then the TerminalWebSocketHandler shall emit a structured `error` message and gracefully close the socket.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **ConPTY Session Bridge Seam (`crates/aico-server/tests/conpty_bridge_test.rs`)**:
   - Signature: Spawns a PowerShell / command session via `PtyManager`, sends input, reads streamed output, issues `resize`, and validates termination.
   - Verifies 100% native ConPTY process execution and WebSocket protocol conformity on Windows.

---

## Out-of-Scope

- Custom browser keyboard shortcut interception outside standard terminal keycodes.
- Terminal recording / asciinema playback features.
