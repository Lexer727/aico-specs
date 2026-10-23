---
id: plan-ft-terminal-quotas-gates-and-notifications
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 7
title: "Argon Quota Split, AICO Gate Blue Status Detection & Windows Popup Notifications Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-skills"]
derives_from: ["spec-ft-terminal-quotas-gates-and-notifications"]
created_at: "2026-10-23T19:05:21.510Z"
---

# Implementation Plan: Argon Quota Split, AICO Gate Blue Status Detection & Windows Popup Notifications

## 1. Technical Architecture & Decisions

- **Target Repositories**: `aico` (`crates/aico-server`, `frontend`), `aico-skills`.
- **Quota Separation (`Gemini` vs `Argon`)**:
  - Make `AiProviderQuota.five_hour` optional (`Option<QuotaWindow>`) in `crates/aico-server/src/quotas.rs` (using `#[serde(skip_serializing_if = "Option::is_none")]` or serializing `null`) so providers without a 5-hour bucket (`Argon`) do not emit synthetic 5h data.
  - Ensure `parse_language_server_quota_summary` matches `"argon"` before general `"gemini"` checks, defaults normal `Gemini` to 100% Weekly / 100% 5h when `"Gemini Models"` is omitted by the language server (0% used), and outputs `[Claude, Gemini, Argon]`.
  - In `frontend/src/components/layout/AiStatusMonitor.tsx`, render `p.five_hour` conditionally so `Argon` displays only its `W` row.
- **AICO Gate `waiting_for_input` Lock**:
  - In `frontend/src/components/terminal/TerminalConsole.tsx`, strip ANSI escape sequences and keep a rolling 2,000-character plain-text tail buffer per tab. When output settles in `scheduleWorkingToFinished`, check the tail buffer for AICO gate phrases (`[AICO GATE`, `waiting for confirmation`, `awaiting confirmation`, `spec sign-off`, `halt for implementation confirmation`, etc.) or interactive prompt patterns, and transition to `waiting_for_input` (`🔵`) instead of `finished` (`🟢`).
  - Update `aico-start`, `aico-feature`, `aico-epic`, and `aico-spec-creation` skills to output `[AICO GATE: Waiting for Confirmation]` at mandatory developer review stops.
- **Windows Popup Notifications**:
  - Add `POST /api/v1/notifications/popup` in `crates/aico-server/src/server.rs` that invokes a non-blocking Windows Toast / `System.Windows.Forms.NotifyIcon` balloon notification on Windows (`#[cfg(target_os = "windows")]`) and returns `{ status: "sent" }`.
  - Trigger popup notifications in `TerminalConsole.tsx` when transitioning from `working` to `finished` or `waiting_for_input`, controlled by a toggle in `SettingsModal.tsx` (`aico-desktop-notifications-enabled`, default `true`).

---

## 2. Task Graph (DAG)

### Phase 1: Quota Split (`Gemini` vs `Argon`)

- **TASK-TERMINAL-007-1**: Separate Gemini and Argon quotas in backend parser and sidebar UI, and omit Claude when not logged in
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-003`, `REQ-006`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updates `crates/aico-server/src/quotas.rs` so `Gemini` tracks `gemini-weekly` + `gemini-5h`, `Argon` tracks `gemini-argon-wtu-weekly` with `five_hour: None`, and `Claude` is only included when `discover_claude_live()` confirms an authenticated Claude CLI session (removing `claude` from `parse_language_server_quota_summary` and `fallback_quotas`).
    - Updates `frontend/src/components/navigation/AiStatusMonitor.tsx` to render `Argon` below `Gemini` with strictly the Weekly (`W`) meter (`Boolean(provider.five_hour) && provider.id !== 'argon'`).
  - **Verification Seam**: `crates/aico-server/src/quotas.rs::tests::test_parse_quota_summary_separates_gemini_and_argon`

### Phase 2: AICO Gate Blue Status Detection

- **TASK-TERMINAL-007-2**: Detect AICO review gates as `waiting_for_input` (`🔵`) on terminal output settle and standardize skill gate markers
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-004`, `REQ-007`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updates `analyzeAiOutputChunk` and `scheduleWorkingToFinished` in `frontend/src/components/terminal/TerminalConsole.tsx` to inspect recent output upon settle and preserve `waiting_for_input` until user keystrokes.
    - Adds `[AICO GATE: Waiting for Confirmation]` to `aico-start/SKILL.md`, `aico-feature/SKILL.md`, `aico-epic/SKILL.md`, and `aico-spec-creation/SKILL.md`.
  - **Verification Seam**: `frontend/src/components/terminal/__tests__/TerminalGateAndQuota.test.tsx`

### Phase 3: Windows Desktop Popup Notifications

- **TASK-TERMINAL-007-3**: Add Windows popup notification endpoint and trigger on task completion or input gate
  - **Status**: Done
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-TERMINAL-007-2`
  - **What it delivers**:
    - Implements `POST /api/v1/notifications/popup` in `crates/aico-server/src/server.rs`.
    - Wires `TerminalConsole.tsx` state transitions (`working -> finished` and `working -> waiting_for_input`) and `SettingsModal.tsx` notification preference toggle.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs::test_windows_popup_notification_endpoint`
