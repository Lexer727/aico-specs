---
id: plan-ft-terminal-ai-quota-monitor
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 5
title: "Multi-Provider AI Token Quota Monitor & Terminal Session Status Implementation Plan"
status: "Draft"
repositories: ["aico"]
derives_from: ["spec-ft-terminal-ai-quota-monitor"]
created_at: "2026-10-10T03:05:00Z"
---

# Implementation Plan: Multi-Provider AI Token Quota Monitor & Terminal Session Status

## 1. Executive Summary & Verification Strategy

This plan implements real-time AI quota tracking and session status monitoring in AICO. It introduces the backend `/api/v1/ai/quotas` discovery service and replaces the mock `ActivityMonitor.tsx` with a compact 2-line `AiStatusMonitor.tsx` widget. Verification is enforced via Rust integration tests and frontend Vitest component tests.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

- **TASK-QUOTA-001**: Implement /api/v1/ai/quotas endpoint with dynamic CLI discovery in aico-server
  - **Status**: Ready
  - **Addresses**: REQ-001, REQ-002, REQ-009, REQ-011
  - **Blocked by**: none
  - **Verification Seam**: `crates/aico-server/src/server.rs`
  - **What it delivers**:
    - Discovers installed AI CLIs (`agy`, `claude`), reads active credentials dynamically, retrieves token quotas, and filters output strictly to connected providers.

- **TASK-QUOTA-002**: Implement AI terminal session state synchronization
  - **Status**: Ready
  - **Addresses**: REQ-006, REQ-007, REQ-008
  - **Blocked by**: TASK-QUOTA-001
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Connects PTY runtime activity to `/api/v1/ai/quotas` to expose real-time session state (`not_open`, `waiting_input`, `working`).

- **TASK-QUOTA-003**: Implement compact 2-line AiStatusMonitor component in frontend
  - **Status**: Ready
  - **Addresses**: REQ-003, REQ-004, REQ-005, REQ-010
  - **Blocked by**: TASK-QUOTA-002
  - **Verification Seam**: `frontend/src/components/layout/AiStatusMonitor.tsx`
  - **What it delivers**:
    - Renders Gemini weekly & 5-hour limits on Line 1 with circular progress rings and reset countdowns.
    - Renders Claude limits on Line 2 only if authenticated.
    - Displays live AI terminal state badge (`Inactive`, `Waiting for Input`, `Working`).

- **TASK-QUOTA-004**: Replace ActivityMonitor in ShellLayout and purge hardcoded host mocks
  - **Status**: Ready
  - **Addresses**: REQ-003, REQ-011
  - **Blocked by**: TASK-QUOTA-003
  - **Verification Seam**: `frontend/src/components/layout/ShellLayout.tsx`
  - **What it delivers**:
    - Replaces mock `ActivityMonitor` in sidebar footer with `AiStatusMonitor` and eliminates hardcoded CPU/RAM values.

- **TASK-QUOTA-005**: Implement automated test suites for quota API and monitor UI
  - **Status**: Ready
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-008
  - **Blocked by**: TASK-QUOTA-004
  - **Verification Seam**: `crates/aico-server/tests/ai_quota_test.rs`
  - **What it delivers**:
    - Integration test in `crates/aico-server/tests/ai_quota_test.rs` validating API response and provider filtering.
    - Unit test in `frontend/tests/AiStatusMonitor.test.tsx` verifying compact 2-line rendering and state badge transitions.

---

## 3. Amendment History

- **2026-10-10T03:05:00Z**: Initial creation of AI quota monitor plan under epic `ec-terminal`.
