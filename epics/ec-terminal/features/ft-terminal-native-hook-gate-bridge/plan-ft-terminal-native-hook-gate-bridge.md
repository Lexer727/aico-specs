---
id: plan-ft-terminal-native-hook-gate-bridge
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 8
title: "Native Antigravity Hook Gate Bridge for Blue Waiting Status Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-skills"]
derives_from: ["spec-ft-terminal-native-hook-gate-bridge"]
created_at: "2026-10-25T20:28:00.000Z"
---

# Implementation Plan: Native Antigravity Hook Gate Bridge for Blue Waiting Status

## 1. Technical Architecture & Decisions

- **Target Repositories**: `aico` (`crates/aico-server`, `frontend`), `aico-skills` (`.agents/hooks.json`).
- **Hook Dispatcher Architecture (`.agents/scripts/aico-hook-status.mjs`)**:
  - The script runs synchronously under Node.js via Antigravity's `hooks.json` engine.
  - Reads `stdin` JSON payload conforming to Antigravity's `PreToolUse`, `PostToolUse`, `PreInvocation`, and `Stop` specs.
  - Dispatches an asynchronous HTTP `POST` to `http://127.0.0.1:4242/api/v1/terminal/ai-status` with an abort controller timeout of 250ms.
  - Wraps all execution in `try / catch` to ensure network errors (e.g. `ECONNREFUSED` when `aico-server` is not running) never throw and never fail Antigravity tool execution, always outputting `{}` to `stdout` with exit code 0.
- **Backend WebSocket Broadcast (`crates/aico-server/src/server.rs`)**:
  - Adds route `POST /api/v1/terminal/ai-status` taking `{ tab_id: Option<String>, state: String, reason: Option<String> }`.
  - Broadcasts `{"type": "ai_status_update", "tab_id": tab_id, "state": state, "reason": reason}` across the WebSocket terminal bridge to all connected frontend clients.
- **Frontend Status Synchronization (`TerminalConsole.tsx` & `useTerminalTitleSync.ts`)**:
  - In `TerminalConsole.tsx`, WebSocket message listener catches `ai_status_update`.
  - On `state === "waiting_for_input"`:
    - Sets `aiTabStatuses[tabId]` to `{ state: "waiting_for_input", reason: "Waiting for user input (ask_question)" }`.
    - Sets `waitingGateLockRef.current[tabId] = true` to protect against trailing debounce timers in `scheduleWorkingToFinished`.
    - Invokes `triggerDesktopPopupNotification(tabName, 'waiting_for_input')`.
    - Triggers `useTerminalTitleSync` to render `AICO 🔵`.
  - On `state === "working"`:
    - Clears `waitingGateLockRef.current[tabId] = false`.
    - Resumes spinner animation.

---

## 2. Task Graph (DAG)

### Phase 1: Backend Route & WebSocket Broadcast

- **TASK-HOOK-001-1**: Implement `POST /api/v1/terminal/ai-status` and WebSocket Broadcast
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-003`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - In `crates/aico-server/src/server.rs`, implements `POST /api/v1/terminal/ai-status` handler and wires broadcast to terminal WebSocket clients.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs::test_ai_status_endpoint`

### Phase 2: Antigravity Lifecycle Hook Script & Configuration

- **TASK-HOOK-001-2**: Implement Hook Dispatcher Script and Register in `.agents/hooks.json`
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-006`
  - **Blocked by**: `TASK-HOOK-001-1`
  - **What it delivers**:
    - Implements `.agents/scripts/aico-hook-status.mjs` handling `PreToolUse` on `ask_question`, `PostToolUse` on `ask_question`, `PreInvocation`, and `Stop`.
    - Configures `.agents/hooks.json` lifecycle hooks.
  - **Verification Seam**: `.agents/tests/aico-hook-status.test.mjs`

### Phase 3: Frontend Status Synchronization & Gate Lock

- **TASK-HOOK-001-3**: Wire TerminalConsole.tsx and useTerminalTitleSync to ai_status_update Events
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-005`, `REQ-007`
  - **Blocked by**: `TASK-HOOK-001-1`
  - **What it delivers**:
    - Updates `TerminalConsole.tsx` to handle `ai_status_update` WebSocket messages, lock `waiting_for_input` (`🔵`), update document title, and fire desktop popup notification.
  - **Verification Seam**: `frontend/src/components/terminal/__tests__/TerminalGateAndQuota.test.tsx` and `frontend/tests/TerminalTitleStatus.test.tsx`

### Phase 4: End-to-End Integration Verification

- **TASK-HOOK-001-4**: End-to-End Integration Test for Interactive Gate Blue Dot Transition
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-007`
  - **Blocked by**: `TASK-HOOK-001-2`, `TASK-HOOK-001-3`
  - **What it delivers**:
    - Comprehensive integration test verifying simulated `ask_question` hook event updates tab dot and browser document title to blue circle.
  - **Verification Seam**: `frontend/tests/TerminalTitleStatus.test.tsx`
