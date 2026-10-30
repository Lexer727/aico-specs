---
id: plan-ft-distribution-uninstaller
type: plan
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 5
title: "Scoped Uninstaller in Install Bin Directory Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-distribution-uninstaller"]
created_at: "2026-10-30T19:19:09.432Z"
---

# Implementation Plan: Scoped Uninstaller in Install Bin Directory

## 1. Technical Architecture & Decisions

- **Target repository**: `aico` (`installer/aico.iss`, `installer/tests/smoke-uninstall.ps1`).
- **Cross-feature prerequisite**: `ft-distribution-windows-installer` (`TASK-DISTRIBUTION-003-1`) creates `installer/aico.iss`.
- **Uninstaller placement**:
  - `UninstallFilesDir={app}\.aico\bin` in `[Setup]`.
  - Generate `aico-uninstall.cmd` inside `{app}\.aico\bin` delegating to `"%~dp0unins000.exe" %*` so users also have a predictable filename.
- **Uninstall `[Code]` hooks**:
  - `InitializeUninstall()`: detect and terminate running `{app}\.aico\bin\aico-server.exe`. Determine whether to purge runtime data: if `/PURGEDATA` is on the command line → `true`; else if `UninstallSilent()` → `false`; else show `MsgBox('Do you also want to remove AICO database storage (.aico\storage), logs, and synced AICO skills? Your specs/ and codebase/ folders will never be deleted.', mbConfirmation, MB_YESNO or MB_DEFBUTTON2) = IDYES`.
  - `CurUninstallStepChanged(usUninstall)`: remove `{app}\.aico\bin` from `HKCU\Environment` `Path` and broadcast `WM_SETTINGCHANGE`.
  - `CurUninstallStepChanged(usPostUninstall)`: remove `{app}\.aico\bin` directory if any residual files remain; if purge-data was chosen, delete `{app}\.aico\storage`, `{app}\.aico\logs`, `{app}\.agents\skills\aico-*`, and `{app}\.aico` if empty. Never touch `{app}\specs`, `{app}\codebase`, or `%USERPROFILE%\.aico`.

---

## 2. Task Graph (DAG)

### Phase 1: Core Vertical Tracer Bullet

- **TASK-DISTRIBUTION-005-1**: Scoped Uninstaller Hooks, Opt-In Data Purge & Smoke Tests
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `InitializeUninstall` and `CurUninstallStepChanged` Pascal script implementation in `installer/aico.iss` plus `aico-uninstall.cmd` wrapper.
    - `installer/tests/smoke-uninstall.ps1` testing both default uninstall and `/PURGEDATA` uninstall against pre-seeded workspace markers, wired into `.github/workflows/release.yml`.
  - **Verification Seam**: `installer/tests/smoke-uninstall.ps1`
