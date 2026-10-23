---
id: spec-ft-terminal-quotas-gates-and-notifications
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 7
title: "Argon Quota Split, AICO Gate Blue Status Detection & Windows Popup Notifications"
status: "Completed"
derives_from: ["brain-ec-terminal", "ADR-0004"]
created_at: "2026-10-23T19:05:21.510Z"
---

# Feature Specification: Argon Quota Split, AICO Gate Blue Status Detection & Windows Popup Notifications

## 1. Problem Statement & User Value

Three terminal telemetry and notification gaps reduce visibility when orchestrating AI agents in AICO:
1. **Gemini vs. Argon Quota Conflation**: Antigravity's `RetrieveUserQuotaSummary` RPC returns up to three quota groups: `"Gemini Models"` (`gemini-weekly` and `gemini-5h`), `"Claude and GPT models"` (`3p-weekly` and `3p-5h`), and `"Gemini 4 Argon"` (`gemini-argon-wtu-weekly`, which has only a weekly window and no 5-hour bucket). Because `parse_language_server_quota_summary` previously matched any group whose `displayName` contained `"gemini"` or whose `bucketId` started with `"gemini-"`, `"Gemini 4 Argon"` was accidentally bound to the `Gemini` row whenever normal Gemini Models were omitted or ordered after Argon. Developers need `Gemini` to strictly show the weekly and 5-hour limits for normal Gemini models, and `Argon` to appear as a separate row below `Gemini` showing **only** its weekly limit (`W`).
2. **AICO Gates Misclassified as Completed (`finished`)**: When an AI agent reaches an AICO human review gate (e.g., Phase 2 Spec Sign-Off or Phase 3 Post-Spec Confirmation) and waits for developer approval, trailing TUI redraw chunks containing braille spinner frames cause `scheduleWorkingToFinished` in `TerminalConsole.tsx` to transition the tab indicator to green `finished` (`🟢`) instead of blue `waiting_for_input` (`🔵`).
3. **Missing Windows Desktop Popup Notifications**: When a long-running AI terminal finishes its task or halts at an input gate while the developer is working in another window, no native Windows desktop notification is raised.

---

## In-Scope
- Separating `Gemini` (`id: "gemini"`, weekly + 5h buckets) and `Argon` (`id: "argon"`, weekly-only bucket) in `crates/aico-server/src/quotas.rs` and `frontend/src/components/layout/AiStatusMonitor.tsx`.
- Rendering only the Weekly (`W`) meter on the `Argon` row in `AiStatusMonitor.tsx` when `five_hour` is `None`/`null`.
- Enhancing `TerminalConsole.tsx` (`analyzeAiOutputChunk` and `scheduleWorkingToFinished`) to inspect the recent output buffer upon output settle and lock `waiting_for_input` (`🔵`) when an AICO gate or confirmation prompt is active until the developer sends input.
- Standardizing the `[AICO GATE: Waiting for Confirmation]` marker in `aico-start`, `aico-feature`, `aico-epic`, and `aico-spec-creation` skills.
- Adding `POST /api/v1/notifications/popup` in `crates/aico-server/src/server.rs` to trigger native Windows popup notifications, wired to `TerminalConsole.tsx` state transitions (`working -> finished` and `working -> waiting_for_input`) with a user preference toggle in `SettingsModal.tsx`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer monitoring AI model quotas in the sidebar, I want `Gemini` to strictly report the Weekly and 5h limits for normal Gemini models and `Argon` to appear as a separate entry below showing only its Weekly limit, so that Argon usage never masks my standard Gemini Flash/Pro quota.
   - *Independent Test*: Parse a `RetrieveUserQuotaSummary` payload containing both `"Gemini Models"` (or omitted `"Gemini Models"` at 100%) and `"Gemini 4 Argon"`, and verify `gemini` has both `weekly` and `five_hour` while `argon` has `weekly` and `five_hour: None`.
2. **US-02** (Priority: P1): As a developer using AICO's 4-phase review gates, I want the terminal tab status dot and sidebar monitor to turn blue (`waiting_for_input`) whenever the agent halts for specification or implementation sign-off, so that I immediately know the agent is waiting for my confirmation rather than completely done.
   - *Independent Test*: Feed simulated TUI output chunks containing `[AICO GATE: Waiting for Confirmation]` or `awaiting confirmation` followed by trailing braille spinner frames, and verify the status settles on `waiting_for_input` (`🔵`).
3. **US-03** (Priority: P2): As a developer multitasking across windows, I want a native Windows popup notification when an AI agent finishes a task or pauses for input, so that I am alerted immediately without polling the browser tab.
   - *Independent Test*: Post a notification payload to `/api/v1/notifications/popup` and verify a transition from `working` to `finished` or `waiting_for_input` dispatches the notification when enabled.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The system shall expose distinct quota provider entries in `GET /api/v1/ai/quotas` for `gemini` (`Gemini`, with Weekly and 5h limits) and `argon` (`Argon`, with Weekly limit only), and shall include `claude` (`Claude`) only when Claude CLI is genuinely authenticated (`discover_claude_live`).
- **REQ-002**: The system shall emit the `[AICO GATE: Waiting for Confirmation]` marker in AICO skill prompts whenever halting at Phase 2 (Spec Sign-Off) or Phase 3 (Post-Spec Implementation Confirmation).

### Event-Driven Requirements
- **REQ-003**: WHEN parsing `RetrieveUserQuotaSummary` in `crates/aico-server/src/quotas.rs`, the system shall bind `"Gemini 4 Argon"` (`bucketId` containing `"argon"` or `displayName` containing `"argon"`) exclusively to the `argon` provider with a `weekly` window and `five_hour: None`, bind `"Gemini Models"` (`bucketId == "gemini-weekly"` and `"gemini-5h"`, excluding `"argon"`) exclusively to the `gemini` provider, and omit Antigravity's `3p` (`"Claude and GPT models"`) bucket so unauthenticated Claude CLI installations never appear in `/api/v1/ai/quotas`.
- **REQ-004**: WHEN an AI terminal output stream settles after activity in `TerminalConsole.tsx`, the system shall inspect the accumulated recent output tail for AICO gate markers (`[AICO GATE`, `waiting for confirmation`, `awaiting confirmation`, `halt for implementation confirmation`, `proceed with implementation?`) and transition the session status to `waiting_for_input` (`🔵`).
- **REQ-005**: WHEN an AI terminal session transitions from `working` to `finished` or `waiting_for_input` while desktop notifications are enabled, the system shall invoke `POST /api/v1/notifications/popup` to display a native Windows popup notification and trigger browser notification fallback.

### Unwanted Behavior / Edge-Case Requirements
- **REQ-006**: IF a quota provider entry has `five_hour: null` or `id === "argon"`, THEN the system shall render only the Weekly (`W`) progress meter on that provider's row in `AiStatusMonitor.tsx` and never render a `5h` meter.
- **REQ-007**: WHILE a terminal session is in the `waiting_for_input` state, the system shall not overwrite the status to `finished` (`🟢`) on trailing idle timers until the user submits input in that terminal.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Quota Summary Parser Seam (`crates/aico-server/src/quotas.rs`)**:
   - Signature: `pub fn parse_language_server_quota_summary(payload: &serde_json::Value) -> Option<Vec<AiProviderQuota>>`
   - Seam Test: `crates/aico-server/src/quotas.rs::tests::test_parse_quota_summary_separates_gemini_and_argon`
2. **Terminal Gate Detection Seam (`frontend/src/components/terminal/TerminalConsole.tsx`)**:
   - Signature: `export function analyzeAiOutputChunk(rawChunk: string, recentBuffer?: string): 'working' | 'waiting_for_input' | 'finished' | null`
   - Seam Test: `frontend/src/components/terminal/__tests__/TerminalGateAndQuota.test.tsx`
3. **Windows Popup Notification Seam (`crates/aico-server/src/server.rs`)**:
   - Signature: `POST /api/v1/notifications/popup`
   - Seam Test: `crates/aico-server/tests/server_test.rs::test_windows_popup_notification_endpoint`

---

## Out-of-Scope
- Non-Windows native OS notification daemons (macOS/Linux fall back to standard Web Notification API).
- Modifying Antigravity's upstream language server binary.

---

## 5. Open Clarifications

None. Ratified in [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md).
