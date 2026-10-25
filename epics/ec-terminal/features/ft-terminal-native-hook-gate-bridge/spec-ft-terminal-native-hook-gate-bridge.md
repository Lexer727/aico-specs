---
id: spec-ft-terminal-native-hook-gate-bridge
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 8
title: "Native Antigravity Hook Gate Bridge for Blue Waiting Status"
status: "Completed"
derives_from: ["brain-ec-terminal", "ADR-0004"]
created_at: "2026-10-25T20:28:00.000Z"
---

# Feature Specification: Native Antigravity Hook Gate Bridge for Blue Waiting Status

## 1. Problem Statement & User Value

In AICO, the browser tab title (`useTerminalTitleSync`) and terminal tab strip status dots (`TerminalConsole.tsx`) display real-time status indicators for AI agent sessions: green circle (`🟢`) for finished/ready, braille spinner / amber circle (`⠋` / spinning circle) for active working, and blue bouncing circle (`🔵`) for `waiting_for_input`.

However, when Google Antigravity or an AICO skill invokes the interactive multiple-choice question tool (`ask_question`), the agent halts execution and waits for developer selection inside a modal UI. Because `ask_question` is dispatched as an internal agent tool call rather than emitted as plaintext to the terminal stdout stream:
1. Terminal stdout regex scanners in `TerminalConsole.tsx` fail to match any gate marker.
2. The 1200ms debounce timer in `scheduleWorkingToFinished` elapses and falsely transitions the session status to `finished` (`🟢`).
3. The developer multitasking in other windows or browser tabs is misled into believing the task completed, instead of knowing that the agent is blocked awaiting their input.

This feature resolves this issue by implementing an event-driven, push-based bridge using Antigravity's native `PreToolUse` lifecycle hooks (`hooks.json`). When `ask_question` is called, the hook fires immediately, notifying `aico-server` via `POST /api/v1/terminal/ai-status`, which broadcasts the `waiting_for_input` state over WebSocket to `TerminalConsole.tsx` and `useTerminalTitleSync`, locking the blue bouncing circle (`🔵`) until the user responds.

---

## In-Scope

- **Native Hook Configuration & Dispatcher Script**:
  - Defining `.agents/hooks.json` lifecycle hooks for `PreToolUse` (`ask_question`), `PostToolUse` (`ask_question`), `PreInvocation`, and `Stop`.
  - Creating a non-blocking node script `.agents/scripts/aico-hook-status.mjs` conforming to the Antigravity hook stdin/stdout contract and dispatching HTTP `POST` requests to `aico-server`.
  - Failing gracefully with timeout protection and empty JSON output `{}` when `aico-server` is offline.
- **Backend AI Status Route & WebSocket Broadcast**:
  - Exposing `POST /api/v1/terminal/ai-status` in `crates/aico-server/src/server.rs`.
  - Broadcasting `ai_status_update` messages over `/api/v1/terminal/ws` to connected WebUI frontend clients.
- **Frontend Reactive Status Synchronization**:
  - Processing `ai_status_update` WebSocket messages in `TerminalConsole.tsx`.
  - Updating `aiTabStatuses` with `waiting_for_input` (`🔵`) on `PreToolUse` and resetting to `working` on `PostToolUse`.
  - Locking `waitingGateLockRef` so trailing debounce timers cannot overwrite status to `finished`.
  - Triggering browser document title synchronization (`AICO 🔵`) and Windows desktop popup notifications.
- **Public Test Seams**:
  - Seam tests in `crates/aico-server/tests/server_test.rs`, `frontend/src/components/terminal/__tests__/TerminalGateAndQuota.test.tsx`, and `frontend/tests/TerminalTitleStatus.test.tsx`.

---

## Out-of-Scope

- Polling or watching `transcript.jsonl` files on disk.
- Third-party CLI tools (e.g. raw bash commands) that do not support Antigravity lifecycle hooks.
- Modifying Antigravity core engine or binary internals.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer multitasking while Antigravity runs an AICO skill, I want the browser tab title and terminal tab dot to turn blue (`🔵`) the instant an interactive multiple-choice question (`ask_question`) is raised, so that I know immediately that my decision is needed.
   - *Independent Test*: Dispatch simulated `PreToolUse` event for `ask_question` to `/api/v1/terminal/ai-status`; assert browser document title updates to `AICO 🔵` and terminal tab indicator renders sky bouncing dot (`🔵`).

2. **US-02** (Priority: P1): As a developer answering an interactive question, I want the status to switch back to working spinner when I submit my choice, and settle to green (`🟢`) only when the agent has completed all turns, so that the status accurately reflects execution lifecycle.
   - *Independent Test*: Dispatch `PostToolUse` event; assert status transitions to `working` with rotating braille spinner; dispatch `Stop` event; assert status settles to `finished` (`🟢`).

3. **US-03** (Priority: P2): As a developer running Antigravity outside the AICO WebUI (stand-alone CLI), I want the hook script to fail silently without throwing errors or blocking the agent loop if `aico-server` is not running.
   - *Independent Test*: Execute `.agents/scripts/aico-hook-status.mjs` with `aico-server` stopped; assert exit code 0 and valid JSON on stdout.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The system shall expose route `POST /api/v1/terminal/ai-status` on `aico-server` accepting JSON payload `{ tab_id?: string, state: "working" | "waiting_for_input" | "finished" | "error", reason?: string }`.
- **REQ-002**: The system shall provide an Antigravity lifecycle hook configuration in `.agents/hooks.json` mapping `PreToolUse` (`ask_question`), `PostToolUse` (`ask_question`), `PreInvocation`, and `Stop` to `.agents/scripts/aico-hook-status.mjs`.

### Event-Driven Requirements
- **REQ-003**: WHEN `POST /api/v1/terminal/ai-status` receives a valid status payload, the system shall broadcast a message `{ type: "ai_status_update", tab_id, state, reason }` to all connected terminal WebSocket clients.
- **REQ-004**: WHEN `TerminalConsole.tsx` receives an `ai_status_update` message with `state === "waiting_for_input"`, the system shall update `aiTabStatuses`, set `waitingGateLockRef` to `true`, update `document.title` to include `🔵`, and invoke `triggerDesktopPopupNotification`.
- **REQ-005**: WHEN `TerminalConsole.tsx` receives an `ai_status_update` message with `state === "working"`, the system shall clear `waitingGateLockRef` and advance the braille spinner frame.
- **REQ-006**: WHEN `.agents/scripts/aico-hook-status.mjs` executes while `aico-server` is unreachable or times out after 250ms, the script shall suppress network errors and output `{}` to `stdout` with exit code 0.

### Unwanted Behavior / Edge-Case Requirements
- **REQ-007**: WHILE `waitingGateLockRef` is locked via an active `waiting_for_input` hook event, trailing debounce timers in `scheduleWorkingToFinished` shall not overwrite the status to `finished` (`🟢`) until a subsequent `PostToolUse` or user input event occurs.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **AI Status Endpoint Seam (`crates/aico-server/src/server.rs`)**:
   - Signature: `POST /api/v1/terminal/ai-status`
   - Test File: `crates/aico-server/tests/server_test.rs`
   - Verifies: Valid status ingestion, schema parsing, and WebSocket broadcast.
2. **Hook Dispatcher Seam (`.agents/scripts/aico-hook-status.mjs`)**:
   - Signature: CLI invocation via `stdin` JSON payload
   - Test File: `.agents/tests/aico-hook-status.test.mjs`
   - Verifies: Non-blocking execution, offline server tolerance, timeout enforcement.
3. **Frontend Terminal Title and Status Seam (`frontend/src/components/terminal/TerminalConsole.tsx`)**:
   - Signature: `useTerminalTitleSync(tabs, aiTabStatuses)` & `handleAiOutputChunk`
   - Test Files: `frontend/src/components/terminal/__tests__/TerminalGateAndQuota.test.tsx` and `frontend/tests/TerminalTitleStatus.test.tsx`
   - Verifies: Transition to `waiting_for_input` (`🔵`), gate lock enforcement, and timer suppression.

---

## Out-of-Scope

- Modifying upstream Google Antigravity binary executables.
- In-memory inspection of unmanaged terminal child processes.
