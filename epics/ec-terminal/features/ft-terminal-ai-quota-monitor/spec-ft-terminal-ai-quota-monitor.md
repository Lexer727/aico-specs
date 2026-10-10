---
id: spec-ft-terminal-ai-quota-monitor
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 5
title: "Multi-Provider AI Token Quota Monitor & Terminal Session Status Widget"
status: "Draft"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-10T03:05:00Z"
---

# Feature Specification: Multi-Provider AI Token Quota Monitor & Terminal Session Status Widget

## 1. Problem Statement & User Value

The bottom-left status bar in the AICO sidebar (`ActivityMonitor.tsx`) currently renders hardcoded mock values (`HOST MONITOR 4242 OK`, `CPU: 1.8%`, `RAM: 184MB`). These static numbers provide zero utility for developers managing AI coding workflows.

Developers need real-time, actionable visibility into their active AI runtime:
1. **Token Limits & Quotas**: Real-time awareness of remaining weekly and 5-hour quota pools for Gemini and Claude models (as shown in Antigravity), avoiding unexpected rate limits mid-task.
2. **Compact 2-Line Layout**: Consolidating Gemini Weekly and 5-Hour limits onto a single line, with secondary providers (Claude / GPT) on the line below.
3. **Connected-Only Providers**: Dynamically omitting unauthenticated or uninstalled CLIs, showing only the tools the developer actually uses.
4. **Real-Time AI State**: Clear visual indication whether the AI terminal is `Inactive / Not Open`, `Waiting for Input`, or `Working / Thinking`.

This feature implements the backend quota discovery service and replaces `ActivityMonitor.tsx` with a compact `AiStatusMonitor.tsx` component.

---

## In-Scope

- **Backend Quota Discovery API (`/api/v1/ai/quotas`)**:
  - Scanning host environment for installed and authenticated AI CLIs (`agy`, `claude`).
  - Querying quota metrics: weekly percentage, weekly reset time, 5-hour percentage, 5-hour reset time.
  - Dynamically retrieving authenticated account names without hardcoded literals.
  - Tracking live AI terminal activity status (`not_open`, `waiting_input`, `working`).
- **Frontend `AiStatusMonitor.tsx`**:
  - Compact 2-line layout at the bottom of the sidebar.
  - Line 1: Gemini weekly & 5-hour limits on one line with circular meters and countdown timers.
  - Line 2: Claude / secondary CLI limits on the line below (rendered only if connected).
  - Status pill / badge displaying live AI terminal state.
- **Dynamic Connection & Native Login**:
  - Automatically refreshing provider display when native terminal login (`agy login`, `claude login`) completes.
- **Automated Tests**:
  - Backend API test suite in `crates/aico-server/tests/ai_quota_test.rs`.
  - Frontend component test suite in `frontend/tests/AiStatusMonitor.test.tsx`.

---

## Out-of-Scope

- Direct credit card billing management or enterprise invoice downloads.
- Custom local LLM GPU VRAM telemetry (standard cloud CLI token limits are the primary focus).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer monitoring model usage, I want to see my Gemini weekly and 5-hour remaining limits on a single line in the status bar, so that I can track quota without wasting screen space.
   - *Independent Test*: Query `/api/v1/ai/quotas`; assert returned JSON contains `weekly_remaining`, `five_hour_remaining`, and respective reset timers; assert `AiStatusMonitor` renders them on line 1.

2. **US-02** (Priority: P1): As a developer using only Antigravity CLI, I do not want unconfigured CLIs like Claude to clutter my status bar, so that only active providers are visible.
   - *Independent Test*: Verify that when Claude is not logged in, the secondary line is omitted from the widget.

3. **US-03** (Priority: P1): As a developer with the AI terminal minimized or closed, I want an indicator showing whether the agent is waiting for input or actively working, so that I know when to open the terminal.
   - *Independent Test*: Transition AI session state between `waiting_input` and `working`; assert status badge updates with appropriate color and label.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The HostServer shall expose `GET /api/v1/ai/quotas` returning the authentication status, active account identity, token quota limits, and terminal activity state.
- **REQ-002**: The QuotaDiscoveryService shall restrict the returned quota provider list strictly to AI CLIs that are installed and authenticated on the host.
- **REQ-003**: The AiStatusMonitor shall render a compact 2-line status component at the bottom of the left navigation sidebar.

### State-Driven Requirements
- **REQ-004**: While Google Antigravity (`agy`) is authenticated, the AiStatusMonitor shall display Gemini Weekly Limit Remaining and Five-Hour Limit Remaining with reset countdowns on the primary line.
- **REQ-005**: While Claude Code is authenticated on the host, the AiStatusMonitor shall display Claude token quotas on the secondary line below Gemini.
- **REQ-006**: While no AI terminal tab is active, the AiStatusMonitor shall display the AI session state as `Inactive`.
- **REQ-007**: While an active AI terminal session is waiting for user prompt input, the AiStatusMonitor shall display the AI session state as `Waiting for Input`.
- **REQ-008**: While an AI agent turn is running, executing tools, or streaming output, the AiStatusMonitor shall display the AI session state as `Working` with an animated pulse indicator.

### Event-Driven Requirements
- **REQ-009**: When a user completes login via `agy login` or `claude login` inside the terminal, the QuotaDiscoveryService shall detect the updated credentials and include the newly connected provider on the next poll.

### Unwanted Behavior & Error Handling
- **REQ-010**: If an AI CLI is not installed or not authenticated, then the AiStatusMonitor shall omit that provider row from the widget without displaying error alerts.
- **REQ-011**: The HostServer and AiStatusMonitor shall not expose or display hardcoded user email literals.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **AI Quota API Seam (`crates/aico-server/tests/ai_quota_test.rs`)**:
   - Signature: Queries `GET /api/v1/ai/quotas` against mock and live CLI environments, verifying schema validity, provider filtering, and session state transitions.
2. **AI Status Monitor UI Seam (`frontend/tests/AiStatusMonitor.test.tsx`)**:
   - Signature: Tests React rendering of `AiStatusMonitor` under various provider configurations (Gemini only, Gemini + Claude, offline fallback).

---

## Out-of-Scope

- Custom token cost calculation in fiat currencies ($/€).
- Alert sound notifications when quotas drop below threshold.
