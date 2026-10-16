---
id: plan-ft-terminal-ai-quota-monitor
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 5
title: "Multi-Provider AI Token Quota Monitor & Terminal Session Status Implementation Plan"
status: "Completed"
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
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-009, REQ-011
  - **Blocked by**: none
  - **Verification Seam**: `crates/aico-server/src/server.rs`
  - **What it delivers**:
    - Discovers installed AI CLIs (`agy`, `claude`), reads active credentials dynamically, retrieves token quotas, and filters output strictly to connected providers.

- **TASK-QUOTA-002**: Implement AI terminal session state synchronization
  - **Status**: Done
  - **Addresses**: REQ-006, REQ-007, REQ-008
  - **Blocked by**: TASK-QUOTA-001
  - **Verification Seam**: `crates/aico-server/src/terminal.rs`
  - **What it delivers**:
    - Connects PTY runtime activity to `/api/v1/ai/quotas` to expose real-time session state (`not_open`, `waiting_input`, `working`).

- **TASK-QUOTA-003**: Implement compact 2-line AiStatusMonitor component in frontend
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004, REQ-005, REQ-010
  - **Blocked by**: TASK-QUOTA-002
  - **Verification Seam**: `frontend/src/components/layout/AiStatusMonitor.tsx`
  - **What it delivers**:
    - Renders Gemini weekly & 5-hour limits on Line 1 with circular progress rings and reset countdowns.
    - Renders Claude limits on Line 2 only if authenticated.
    - Displays live AI terminal state badge (`Inactive`, `Waiting for Input`, `Working`).

- **TASK-QUOTA-004**: Replace ActivityMonitor in ShellLayout and purge hardcoded host mocks
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-011
  - **Blocked by**: TASK-QUOTA-003
  - **Verification Seam**: `frontend/src/components/layout/ShellLayout.tsx`
  - **What it delivers**:
    - Replaces mock `ActivityMonitor` in sidebar footer with `AiStatusMonitor` and eliminates hardcoded CPU/RAM values.

- **TASK-QUOTA-005**: Implement automated test suites for quota API and monitor UI
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005, REQ-008
  - **Blocked by**: TASK-QUOTA-004
  - **Verification Seam**: `crates/aico-server/tests/ai_quota_test.rs`
  - **What it delivers**:
    - Integration test in `crates/aico-server/tests/ai_quota_test.rs` validating API response and provider filtering.
    - Unit test in `frontend/tests/AiStatusMonitor.test.tsx` verifying compact 2-line rendering and state badge transitions.

- **TASK-QUOTA-006**: Switch agy quota discovery to RetrieveUserQuotaSummary (gemini-weekly and gemini-5h) and remove 3p/Claude tracking from agy while preserving standalone multi-CLI extensibility
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-004, REQ-005
  - **Blocked by**: TASK-QUOTA-005
  - **Verification Seam**: `crates/aico-server/src/quotas.rs`
  - **What it delivers**:
    - Queries `RetrieveUserQuotaSummary` on `language_server` to extract distinct `gemini-weekly` and `gemini-5h` remaining percentages and reset countdowns.
    - Ignores `3p-weekly` / `3p-5h` (`Claude and GPT models`) buckets from `agy` while keeping standalone CLI provider aggregation (`providers[]`) for future CLIs such as Claude Code.

- **TASK-QUOTA-007**: Verify Claude CLI login status for Line 2 quota display and remove global status pill from AiStatusMonitor
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-003, REQ-005, REQ-006, REQ-009, REQ-010
  - **Blocked by**: TASK-QUOTA-006
  - **Verification Seam**: `crates/aico-server/tests/ai_quota_test.rs`, `frontend/tests/AiStatusMonitor.test.tsx`
  - **What it delivers**:
    - Verifies whether the user is logged into Claude Code via `~/.claude/.credentials.json` (`claudeAiOauth.accessToken`) and/or `claude.exe auth status --json` (`"loggedIn": true`), querying Anthropic OAuth usage when a token is available and omitting `Claude` when not logged in.
    - Removes the global session status badge from `AiStatusMonitor.tsx` (relocated to per-tab circle indicators in `TerminalConsole.tsx`).

---

## 3. Amendment History

- **2026-10-10T03:05:00Z**: Initial creation of AI quota monitor plan under epic `ec-terminal`.
- **2026-10-12T17:06:00Z**: Added `TASK-QUOTA-006` to query `RetrieveUserQuotaSummary` for distinct `gemini-weekly` and `gemini-5h` buckets and remove `3p`/Claude tracking from `agy` while preserving standalone multi-CLI tracking.
- **2026-10-12T20:15:00Z**: Added `TASK-QUOTA-007` to verify Claude login status (`claude auth status --json` / `.credentials.json`) for Line 2 display below Gemini and move the session status indicator into the AI terminal tabs.
