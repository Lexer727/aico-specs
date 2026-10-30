---
id: plan-ft-distribution-prerequisite-provisioning
type: plan
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 4
title: "Prerequisite Detection & Provisioning via winget Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-distribution-prerequisite-provisioning"]
created_at: "2026-10-30T19:19:09.252Z"
---

# Implementation Plan: Prerequisite Detection & Provisioning via winget

## 1. Technical Architecture & Decisions

- **Target repository**: `aico` (`installer/prerequisites.json`, `installer/scripts/prereq-manager.ps1`, `installer/aico.iss`).
- **Cross-feature prerequisite**: `ft-distribution-windows-installer` (`TASK-DISTRIBUTION-003-1`) establishes `installer/aico.iss`.
- **Data-driven catalog** `installer/prerequisites.json`:
  - Declares the 7 tools (`git`, `gh`, `node`, `claude-code`, `antigravity`, `codex`, `copilot`) with detection commands, winget IDs, npm fallbacks, and homepage URLs.
  - Bundled into `.aico/bin/prerequisites.json` by `scripts/package-release.ps1`.
- **PowerShell provisioning script** `installer/scripts/prereq-manager.ps1`:
  - `-Action detect`: executes each entry's `detect_cmd` with a 5-second timeout, parses the first version token, and emits a JSON array of `{ id, name, installed, version, required, default_checked }`.
  - `-Action install -Ids <list> [-DryRun]`: for each requested ID, chooses `winget` if `winget_id` is non-empty and `winget` is on PATH, else `npm install -g` if `npm_package` is non-empty and `npm` is on PATH, else returns `status: "manual"` with `homepage_url`. Never throws an unhandled terminating error on individual tool failure.
- **Inno Setup wizard integration**:
  - Custom `TInputOptionWizardPage` created after directory selection.
  - Calls `prereq-manager.ps1 -Action detect` during wizard startup; populates checkboxes (installed items disabled with `[Installed: vX.Y.Z]` suffix).
  - Warns on `NextButtonClick` if `git` is missing and unchecked.
  - Runs `prereq-manager.ps1 -Action install` during `ssInstall` when not in `/VERYSILENT` mode (unless `/INSTALLPREREQS` is passed).

---

## 2. Task Graph (DAG)

### Phase 1: Core Vertical Tracer Bullet

- **TASK-DISTRIBUTION-004-1**: Prerequisite Catalog & Detection/Install Script with Pester Suite
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-007`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `installer/prerequisites.json` containing the 7 tool definitions.
    - `installer/scripts/prereq-manager.ps1` supporting `-Action detect`, `-Action install`, and `-DryRun`.
    - Pester test suite `installer/tests/prereq-manager.Tests.ps1` verifying catalog schema, live detection, winget/npm fallback selection, and graceful failure reporting.
  - **Verification Seam**: `installer/tests/prereq-manager.Tests.ps1`

### Phase 2: Wizard Integration

- **TASK-DISTRIBUTION-004-2**: Inno Setup Prerequisite Page & Required-Git Guard
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-004`, `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-DISTRIBUTION-004-1`
  - **What it delivers**:
    - Custom Prerequisites wizard page in `installer/aico.iss` wired to `prereq-manager.ps1`.
    - Missing-Git warning dialog when `git` is not installed and unchecked.
    - Post-install failure summary offering to open `homepage_url` for any failed optional tool.
  - **Verification Seam**: `installer/tests/smoke-install.ps1`
