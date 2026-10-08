---
id: plan-ft-core-security-auth
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 2
title: "Security, Auto-Mode & Execution Engine"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-security-auth"]
created_at: "2026-10-05T20:00:00Z"
---

# Implementation Plan: Security, Auto-Mode & Execution Engine


---

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-security` (part of the AICO Rust Cargo workspace).
- **Core Dependencies**:
  - `tokio::process`: Asynchronous command spawning with configurable timeouts and process tree termination.
  - `regex`: Fast pattern matching for blacklisted command strings, path boundary detection, and syntax parsing.
  - `serde` / `serde_json`: Serialization of execution policies, audit events, and gate requests.
  - `chrono`: High-precision microsecond UTC timestamps for immutable audit records.
- **Safety Philosophy**: Deny by default. Any command or tool invocation not matching an explicit Tier 1 or Tier 2 rule falls automatically into Tier 3 (interactive human gate).

---

## 2. Task Graph (DAG)

### Phase 1: Policy Classification & Path Sandboxing (Unblocked Frontier)

- **TASK-CORE-002-1**: 3-Tier Command Policy Classifier & Path Sandboxing
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-008`, `REQ-009`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implements `PolicyEngine` and `CommandTier` enum (`Tier1Read`, `Tier2WorkspaceWrite`, `Tier3DangerousOrEgress`).
    - Implements `classify_command(cmd, cwd, env)` and `classify_tool_call(tool_name, params)`.
    - Enforces canonical path resolution (`std::fs::canonicalize`) to detect and reject directory traversal attacks escaping project roots.
    - Matches blacklisted destructive patterns (e.g., recursive deletes of root, disk formatting, `--force` git pushes).
  - **Verification Seam**: `tests/security_policy_test.rs::test_policy_classification_and_sandboxing`

---

### Phase 2: Safe Subprocess Execution Engine

- **TASK-CORE-002-2**: Command Executor with Timeouts & Tree Kill
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-007`
  - **Blocked by**: `TASK-CORE-002-1`
  - **What it delivers**:
    - Implements `CommandExecutor::run_command(cmd, timeout, cwd)`.
    - Wraps `tokio::process::Command` with `kill_on_drop(true)`.
    - Implements configurable timeout handling (default 120s) with clean process group termination on Windows (`taskkill /F /T /PID`) and Unix (`kill -9 -PGID`).
    - Captures stdout/stderr streams without buffer starvation.
  - **Verification Seam**: `tests/security_executor_test.rs::test_command_timeout_and_process_termination`

---

### Phase 3: Audit Trail & Persistence

- **TASK-CORE-002-3**: Synchronous Audit Logger & Event Recording
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-005`
  - **Blocked by**: `TASK-CORE-002-2`
  - **What it delivers**:
    - Implements `AuditLogger` with structured event schema (`timestamp`, `agent_id`, `command`, `tier`, `status`, `duration_ms`, `diff_hash`).
    - Emits synchronous audit log events to the event bus and persists records in Kùzu `AuditEvent` nodes.
    - Guarantees Tier 2 workspace mutations cannot proceed without successfully writing the audit record.
  - **Verification Seam**: `tests/security_audit_test.rs::test_audit_trail_logging_and_persistence`

---

### Phase 4: Interactive WebSocket Gating & Auto-Mode Control

- **TASK-CORE-002-4**: Interactive WebSocket Gating & Auto-Mode Toggle
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-006`
  - **Blocked by**: `TASK-CORE-002-2`, `TASK-CORE-001-3` (WebSocket Hub)
  - **What it delivers**:
    - Implements `ApprovalGateManager` managing pending authorization requests via oneshot channels (`tokio::sync::oneshot`).
    - Dispatches `tier3_approval_required` WebSocket message to active UI clients and pauses caller execution.
    - Exposes `respond_approval(gate_id, approved: bool)` endpoint and WebSocket handler.
    - Adds global auto-mode level configuration (`Off`, `ReadOnly`, `Assisted`, `Autonomous`).
  - **Verification Seam**: `tests/security_gate_test.rs::test_interactive_websocket_approval_gate`

---

### Phase 5: Emergency Kill Switch

- **TASK-CORE-002-5**: Emergency Kill Switch & Global Subprocess Abort
  - **Status**: Done
  - **Addresses**: `REQ-010`
  - **Blocked by**: `TASK-CORE-002-2`
  - **What it delivers**:
    - Implements `EmergencyKillSwitch` maintaining a concurrent registry of all active child process IDs.
    - Exposes `POST /api/v1/emergency-kill` and `/ws` emergency interrupt command.
    - Terminates all active child processes within 200ms, transitions running agent tasks to `cancelled`, and logs an incident event.
  - **Verification Seam**: `tests/security_killswitch_test.rs::test_emergency_kill_switch`
