---
id: spec-ft-terminal-ai-quota-monitor
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 5
title: "Multi-Provider AI Token Quota Monitor & Terminal Session Status Widget"
status: "Completed"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-10T03:05:00Z"
---

# Feature Specification: Multi-Provider AI Token Quota Monitor & Terminal Session Status Widget

## 1. Problem Statement & User Value

The bottom-left status bar in the AICO sidebar (`ActivityMonitor.tsx`) currently renders hardcoded mock values (`HOST MONITOR 4242 OK`, `CPU: 1.8%`, `RAM: 184MB`). These static numbers provide zero utility for developers managing AI coding workflows.

Developers need real-time, actionable visibility into their active AI runtime:
1. **Token Limits & Quotas**: Real-time awareness of remaining weekly and 5-hour quota pools for Gemini and Claude models, avoiding unexpected rate limits mid-task.
2. **Compact 2-Line Layout**: Consolidating Gemini Weekly and 5-Hour limits onto a single line, with secondary providers (Claude) on the line below.
3. **Connected-Only Providers**: Dynamically omitting unauthenticated or uninstalled CLIs, showing only the tools the developer is actively logged into.

This feature implements the backend quota discovery service and replaces `ActivityMonitor.tsx` with a compact `AiStatusMonitor.tsx` component focused on connected provider quotas, while live per-tab AI session status is rendered directly inside the AI terminal tab bar (`ft-terminal-xterm-ui`).

---

## In-Scope

- **Backend Quota Discovery API (`/api/v1/ai/quotas`)**:
  - Scanning host environment for installed and authenticated AI CLIs (`agy`, `claude`).
  - Verifying Claude login status via `~/.claude/.credentials.json` (`claudeAiOauth.accessToken`) and/or `claude.exe auth status --json` (`"loggedIn": true`).
  - Querying quota metrics: weekly percentage, weekly reset time, 5-hour percentage, 5-hour reset time.
  - Dynamically retrieving authenticated account names without hardcoded literals.
- **Frontend `AiStatusMonitor.tsx`**:
  - Compact 2-line layout at the bottom of the sidebar.
  - Line 1: Gemini weekly & 5-hour limits on one line with circular meters and countdown timers.
  - Line 2: Claude CLI limits on the line below (rendered only if the user is logged into Claude).
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
   - *Independent Test*: Query `/api/v1/ai/quotas`; assert returned JSON contains `weekly_remaining_pct`, `five_hour_remaining_pct`, and respective reset timers; assert `AiStatusMonitor` renders them on line 1.

2. **US-02** (Priority: P1): As a developer who is not logged into Claude CLI, I do not want Claude to appear in my status bar until I log in, so that only authenticated providers are visible.
   - *Independent Test*: Verify that when Claude is not logged in (`loggedIn: false` / no OAuth token), the Claude row is omitted from the widget, and appears below Gemini once logged in.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The HostServer shall expose `GET /api/v1/ai/quotas` returning the authentication status, active account identity, and token quota limits for connected AI CLIs.
- **REQ-002**: The QuotaDiscoveryService shall restrict the returned quota provider list strictly to AI CLIs that are installed and authenticated (logged in) on the host.
- **REQ-003**: The AiStatusMonitor shall render a compact multi-line quota status component at the bottom of the left navigation sidebar without a redundant global session state badge (as per-tab AI status is displayed directly in the terminal tab bar per `spec-ft-terminal-xterm-ui::REQ-011`).

### State-Driven Requirements
- **REQ-004**: While Google Antigravity (`agy`) is authenticated, the QuotaDiscoveryService shall query `RetrieveUserQuotaSummary` to extract the distinct `gemini-weekly` (`window: "weekly"`) and `gemini-5h` (`window: "5h"`) buckets for the Gemini primary row, and shall ignore Antigravity's `3p` (`Claude and GPT models`) buckets.
- **REQ-005**: While the user is logged into the standalone Claude Code CLI (verified via a valid `claudeAiOauth.accessToken` in `~/.claude/.credentials.json` or `claude auth status --json` returning `"loggedIn": true`), the QuotaDiscoveryService shall include `Claude` (`id: "claude"`) in `providers[]` and the AiStatusMonitor shall display Claude's weekly and 5-hour token limits on the line below Gemini.
- **REQ-006**: While the user is not logged into Claude Code (`"loggedIn": false` and no valid OAuth token in `~/.claude/.credentials.json`), the QuotaDiscoveryService shall omit `Claude` from connected providers so it does not render in `AiStatusMonitor`.
- **REQ-007**: While an AI CLI session is active, the `TerminalSessionRegistry` shall continue to expose session metadata on `/api/v1/terminal/sessions` and `/api/v1/ai/quotas` for consumption by terminal tabs.
- **REQ-008**: While any connected provider has 0% remaining quota (`weekly_remaining_pct == 0` or `five_hour_remaining_pct == 0`), the quota payload shall expose `0%` so the corresponding AI terminal tab can surface an out-of-tokens error state.

### Event-Driven Requirements
- **REQ-009**: When a user completes login via `agy login` or `claude login` inside the terminal, the QuotaDiscoveryService shall detect the updated credentials and include the newly connected provider on the next poll.

### Unwanted Behavior & Error Handling
- **REQ-010**: If an AI CLI is not installed or not authenticated, then the AiStatusMonitor shall omit that provider row from the widget without displaying error alerts.
- **REQ-011**: The HostServer and AiStatusMonitor shall not expose or display hardcoded user email literals.
- **REQ-012**: While no running language server process is detected on the host during live host discovery, the QuotaDiscoveryService shall launch a transient headless probe using the installed Antigravity language server binary (`language_server.exe` with `--standalone`, `--subclient_type hub`, `--override_ide_name antigravity`, and stdin closed), extract the assigned HTTP port, query `RetrieveUserQuotaSummary` and `GetUserStatus` with CSRF authentication, parse authentic account and Gemini quota metrics, terminate the transient probe process, and cache the result.

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
