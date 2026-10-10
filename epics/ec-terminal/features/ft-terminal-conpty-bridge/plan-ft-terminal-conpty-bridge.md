---
id: plan-ft-terminal-conpty-bridge
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 3
title: "Terminal ConPTY Pseudo-Terminal Backend Bridge Implementation Plan"
status: "Draft"
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
  - **Status**: Ready
  - **Addresses**: REQ-001, REQ-003
  - **Blocked by**: none
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Adds `portable-pty = "0.8"` to `Cargo.toml`.
    - Implements cross-platform PTY abstraction creating native master/slave pairs (Windows ConPTY on Windows) and spawning shell processes directly inside the slave.

- **TASK-PTY-002**: Upgrade WebSocket endpoint with raw byte streaming, dynamic resize, and process cleanup
  - **Status**: Ready
  - **Addresses**: REQ-002, REQ-004, REQ-005, REQ-006, REQ-009
  - **Blocked by**: TASK-PTY-001
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Updates `/api/v1/terminal/ws` message handler to process raw `input` bytes, dispatch `resize` events to PTY master, stream output chunks, and terminate process trees on disconnect.

- **TASK-PTY-003**: Implement AI terminal session state tracking
  - **Status**: Ready
  - **Addresses**: REQ-008
  - **Blocked by**: TASK-PTY-002
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Tracks active AI terminal sessions and exposes real-time state (`idle`, `waiting_for_input`, `working`) for consumption by status monitors.

- **TASK-PTY-004**: Implement automated ConPTY integration test suite
  - **Status**: Ready
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-006
  - **Blocked by**: TASK-PTY-003
  - **Verification Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`
  - **What it delivers**:
    - End-to-end integration test validating ConPTY allocation, command execution, raw input forwarding, dynamic resizing, and clean process termination.

---

## 3. Amendment History

- **2026-10-10T03:05:00Z**: Initial creation of ConPTY backend implementation plan under epic `ec-terminal`.
