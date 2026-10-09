---
id: spec-ft-terminal-backend-stream-bridge
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 1
title: "Terminal Subsystem Backend Process Bridge & WebSocket Streaming"
status: "Draft"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-09T19:46:00Z"
---

# Feature Specification: Terminal Subsystem Backend Process Bridge & WebSocket Streaming

## 1. Problem Statement & User Value

The AICO WebUI shell currently lacks a real backend process execution and streaming bridge for terminals. To provide real, interactive terminal sessions inside the WebUI dockable panel, the backend server must be capable of spawning real host shell processes (PowerShell, Git Bash, Command Prompt, and Google Antigravity `agy.exe`), streaming their output asynchronously to the browser, and piping user input keystrokes to the running process over WebSockets.

This feature provides the core backend process execution, stream multiplexing, and session management engine in `aico-server`.

---

## In-Scope

- **WebSocket Terminal Endpoint**:
  - Exposing `/api/v1/terminal/ws` in `aico-server`.
  - Managing terminal session lifecycle (init, stream, close, process exit).
- **Multi-Shell Process Spawner**:
  - Spawning `powershell.exe`, `cmd.exe`, Git `bash.exe` (`C:\Program Files\Git\bin\bash.exe` or PATH), and `agy.exe`.
  - Setting working directory to the active project root or provided path.
- **Bi-directional Streaming Protocol**:
  - Inbound messages: `init` (shell, cwd), `input` (data string), `resize` (cols, rows), `close`.
  - Outbound messages: `output` (text data chunk), `exit` (exit code), `error` (error message).
- **Process Lifecycle & Cleanup**:
  - Automatically terminating spawned child processes when the WebSocket connection disconnects or session is closed.
- **Automated Seam Tests**:
  - Integration tests verifying process spawning, input writing, output streaming, and cleanup in `aico-server`.

---

## Out-of-Scope (Future Upgrades)

- Full pseudo-terminal (ConPTY) hardware emulation bindings (plain async piped streams satisfy interactive REPL and command needs without requiring external native C DLL dependencies).
- Multi-user terminal collaboration or remote multiplexing over WAN.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer interacting with the WebUI terminal, I want the backend to spawn a real PowerShell process, so that I can run genuine commands and scripts.
   - *Independent Test*: Connect to `/api/v1/terminal/ws`, send init for `powershell`, send input `Write-Host "hello from aico"` followed by newline; assert output contains `hello from aico`.

2. **US-02** (Priority: P1): As a developer requesting an AI terminal, I want the backend to spawn `agy.exe` in my active project directory, so that I can converse with the AI agent and execute AICO skills.
   - *Independent Test*: Connect to `/api/v1/terminal/ws`, send init for `agy`; assert child process is spawned using `agy.exe`.

3. **US-03** (Priority: P1): As a developer closing a terminal tab, I want the backend to immediately terminate the child process, so that orphaned processes do not leak system memory.
   - *Independent Test*: Spawn a long-running process over `/api/v1/terminal/ws` and disconnect WebSocket; assert child process terminates cleanly.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: When a WebSocket connection request is received at `/api/v1/terminal/ws`, the server shall accept and upgrade the connection to an active terminal communication channel.
- **REQ-002**: When an `init` message is received specifying a shell type (`powershell`, `cmd`, `bash`, or `agy`) and optional working directory, the server shall spawn the requested executable with piped standard I/O in the target directory (defaulting to active project root).
- **REQ-003**: While a spawned shell process produces data on its standard output or standard error streams, the server shall forward the data as `output` JSON messages over the WebSocket to the client.
- **REQ-004**: When an `input` message containing text or keystrokes is received from the client, the server shall write the data to the standard input stream of the active child process.
- **REQ-005**: When the WebSocket connection is closed or a `close` message is received, the server shall terminate the child process and reclaim process resources.
- **REQ-006**: When an `init` message requests the `agy` shell, the server shall resolve `agy.exe` and spawn the interactive assistant session in the active project directory.
- **REQ-007**: If a requested shell executable fails to spawn or is not found on the host machine, the server shall send an `error` message to the client without terminating the server.

---

## 4. Designated Public Test Seams

- **Terminal Bridge Integration Seam**: `crates/aico-server/tests/terminal_bridge_test.rs` (verifies REQ-001 through REQ-007)
