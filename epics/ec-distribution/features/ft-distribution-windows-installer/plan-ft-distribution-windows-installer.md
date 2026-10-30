---
id: plan-ft-distribution-windows-installer
type: plan
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 3
title: "Windows Hybrid Installer (Inno Setup) Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-distribution-windows-installer"]
created_at: "2026-10-30T19:19:09.071Z"
---

# Implementation Plan: Windows Hybrid Installer (Inno Setup)

## 1. Technical Architecture & Decisions

- **Target repository**: `aico` (new `installer/` directory).
- **Cross-feature prerequisite**: `ft-distribution-release-pipeline` (`TASK-DISTRIBUTION-002-2`, `TASK-DISTRIBUTION-002-3`) supplies the staged Release Bundle and the workflow hook in which `iscc` runs.
- **Inno Setup script** `installer/aico.iss`:
  - `AppId` fixed GUID; `PrivilegesRequired=lowest`; `DefaultDirName={%USERPROFILE}\aico`; `DirExistsWarning=no`; `UsePreviousAppDir=yes`; `UninstallFilesDir={app}\.aico\bin`; `OutputBaseFilename=aico-setup-{#AppVersion}`.
  - `[Dirs]` for the canonical layout; `[Files]` source = staged bundle directory → `{app}\.aico\bin` with `Flags: ignoreversion recursesubdirs`.
  - No `[Files]` or `[UninstallDelete]` entries target `storage`, `specs`, `codebase`, or `.agents`.
  - `[Tasks]`: `desktopicon`, `startmenu`, `addtopath` (unchecked), `launch`.
  - `[Icons]` for desktop/Start menu → `{app}\.aico\bin\aico-server.exe`, `WorkingDir: {app}`.
  - `[Code]`: existing-workspace notice on `NextButtonClick(wpSelectDir)`; process detection/termination via `taskkill /F /IM aico-server.exe` scoped by path check (PowerShell `Get-Process | Where Path -like`); user PATH edit in `HKCU\Environment` with `WM_SETTINGCHANGE` broadcast; `/NOUPDATECHECK` param parsing.
- **Hybrid update logic** `installer/scripts/release-check.ps1` (extracted for testability, embedded via `ExtractTemporaryFile` and executed from `[Code]`):
  - Queries `https://api.github.com/repos/Lexer727/aico/releases/latest`, compares SemVer against `{#AppVersion}`, downloads zip + `.sha256` to `{tmp}`, verifies with `Get-FileHash`, expands with `Expand-Archive`, returns JSON. On any failure returns `action: bundled` with reason.
  - When `action: downloaded`, `[Code]` copies the extracted tree over `{app}\.aico\bin` in `CurStepChanged(ssPostInstall)`.
- **CI integration**: release workflow installs Inno Setup (`choco install innosetup -y`), runs `iscc /DAppVersion=$version /DPayloadDir=<staged> installer/aico.iss`, then runs `installer/tests/smoke-install.ps1` before upload.
- **Verification**: Pester (`release-check.Tests.ps1`), smoke install in CI, manual GUI walkthrough on a clean Windows VM.

---

## 2. Task Graph (DAG)

### Phase 1: Core Vertical Tracer Bullet

- **TASK-DISTRIBUTION-003-1**: Minimal Installer — Root Picker, Layout, Bundled Payload & Silent Smoke Test
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-009`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `installer/aico.iss` with per-user install, canonical `[Dirs]`, payload into `.aico\bin`, existing-workspace notice, silent switches.
    - `installer/tests/smoke-install.ps1` asserting layout, preserved preseeded markers, and `aico-server.exe --version`.
    - `iscc` + smoke test steps wired into `.github/workflows/release.yml`.
  - **Verification Seam**: `installer/tests/smoke-install.ps1`

### Phase 2: Conveniences & Upgrade Safety

- **TASK-DISTRIBUTION-003-2**: Shortcuts, PATH, Launch, Running-Process Handling & In-Place Upgrade
  - **Status**: Done
  - **Addresses**: `REQ-005`, `REQ-006`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-DISTRIBUTION-003-1`
  - **What it delivers**:
    - `[Tasks]`/`[Icons]`/`[Run]` sections, HKCU PATH manipulation, process termination prompt, previous-root defaulting.
    - Application icon asset `installer/assets/aico.ico`.
    - Smoke test extensions: `/TASKS="desktopicon,addtopath"` assertions and a second install over an existing install (upgrade) while a dummy server process runs.
  - **Verification Seam**: `installer/tests/smoke-install.ps1`

### Phase 3: Hybrid Latest-Release Delivery

- **TASK-DISTRIBUTION-003-3**: Latest-Release Check, Verified Download & Bundled Fallback
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-010`
  - **Blocked by**: `TASK-DISTRIBUTION-003-2`
  - **What it delivers**:
    - `installer/scripts/release-check.ps1` with SemVer comparison, download, SHA-256 verification, extraction, and JSON result.
    - Pester tests with a local HTTP fixture covering newer/equal/older/API failure/checksum mismatch.
    - `[Code]` wizard page offering the newer version and applying the downloaded tree post-install.
  - **Verification Seam**: `installer/tests/release-check.Tests.ps1`
