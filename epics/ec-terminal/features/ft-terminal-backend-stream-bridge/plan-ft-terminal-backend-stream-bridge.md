---
id: plan-ft-terminal-backend-stream-bridge
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 1
title: "Terminal Subsystem Backend Process Bridge Implementation Plan"
status: "Draft"
repositories: ["aico"]
derives_from: ["spec-ft-terminal-backend-stream-bridge"]
created_at: "2026-10-09T19:47:00Z"
---

# Implementation Plan: Terminal Subsystem Backend Process Bridge & WebSocket Streaming

## 1. Technical Architecture & Decisions

- **Process Spawner Module (`terminal.rs` in `aico-server`)**:
  - Implement shell resolution supporting:
    - `powershell`: `powershell.exe -NoLogo`
    - `cmd`: `cmd.exe`
    - `bash`: Checks `C:\Program Files\Git\bin\bash.exe`, then `C:\Program Files (x86)\Git\bin\bash.exe`, then `bash.exe` on PATH.
    - `agy`: Checks `agy.exe` on PATH or user AppData (`C:\Users\astau\AppData\Local\agy\bin\agy.exe`).
  - Spawns child process via `tokio::process::Command` with piped `stdin`, `stdout`, and `stderr`.
- **WebSocket Streaming Protocol**:
  - WebSocket route: `/api/v1/terminal/ws`.
  - JSON messages:
    - Client to Server:
      - `{"type": "init", "shell": "powershell" | "cmd" | "bash" | "agy", "cwd": "<path>"}`
      - `{"type": "input", "data": "<text>"}`
      - `{"type": "close"}`
    - Server to Client:
      - `{"type": "output", "data": "<text>"}`
      - `{"type": "exit", "code": 0}`
      - `{"type": "error", "message": "<error>"}`
- **Concurrency & Lifecycle Management**:
  - Uses `tokio::io::AsyncReadExt` and `tokio::io::AsyncWriteExt` to stream data asynchronously between WebSocket frames and process pipes.
  - Automatically kills child process (`child.kill()`) when WebSocket disconnects or receives `close`.

---

## 2. Task Graph (DAG)

### Phase 1: Process Spawner & Resolver

- **TASK-TERM-001-1**: Implement Shell Process Spawner & Resolver
  - **Status**: Pending
  - **Addresses**: REQ-002, REQ-006, REQ-007
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implement `TerminalProcess` struct and `spawn_shell` function in `crates/aico-server/src/terminal.rs`.
    - Support shell resolution for `powershell`, `cmd`, `bash`, and `agy`.
    - Unit tests verifying process execution and executable path resolution.
  - **Verification Seam**: `crates/aico-server/tests/terminal_bridge_test.rs`

### Phase 2: WebSocket Streaming & Session Handler

- **TASK-TERM-001-2**: Implement Terminal WebSocket Endpoint & Streaming Bridge
  - **Status**: Pending
  - **Addresses**: REQ-001, REQ-003, REQ-004, REQ-005
  - **Blocked by**: TASK-TERM-001-1
  - **What it delivers**:
    - Add `/api/v1/terminal/ws` route in `crates/aico-server/src/server.rs`.
    - Forward WebSocket input messages into process stdin.
    - Read process stdout and stderr concurrently, streaming chunks as `output` WebSocket messages.
    - Handle client disconnect and process termination.
  - **Verification Seam**: `crates/aico-server/tests/terminal_bridge_test.rs`

### Phase 3: Integration Seam Tests & Verification

- **TASK-TERM-001-3**: End-to-End Terminal WebSocket Integration Tests
  - **Status**: Pending
  - **Addresses**: REQ-001 through REQ-007
  - **Blocked by**: TASK-TERM-001-2
  - **What it delivers**:
    - Integration tests in `crates/aico-server/tests/terminal_bridge_test.rs` connecting via `tokio-tungstenite`.
    - Test PowerShell execution and output loopback.
    - Test `agy` executable resolution and error handling.
  - **Verification Seam**: `crates/aico-server/tests/terminal_bridge_test.rs`
