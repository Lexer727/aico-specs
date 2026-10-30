---
id: spec-ft-distribution-windows-installer
type: spec
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 3
title: "Windows Hybrid Installer (Inno Setup)"
status: "Completed"
derives_from: ["brain-ec-distribution", "ADR-0012"]
created_at: "2026-10-30T19:19:09.071Z"
---

# Feature Specification: Windows Hybrid Installer (Inno Setup)

## 1. Problem Statement & User Value

End users need a single, familiar Windows installer that asks **one question** — where the AICO Root should live — and produces a fully working AICO workspace: the canonical folder layout (`specs/`, `codebase/`, `.agents/`, `.aico/`), the runtime in the Install Bin Directory (`<root>/.aico/bin`), and optional conveniences (desktop shortcut, Start menu entry, PATH). Because releases move quickly, the installer must also pick up a newer release than the one it was built with, while still working offline. The developer must be able to point the same installer at an existing root (e.g. `d:/aico`) without losing `.aico/storage` or any repositories.

---

## In-Scope

- Inno Setup 6 script `installer/aico.iss` compiled in CI from the Release Bundle contents.
- Per-user installation (`PrivilegesRequired=lowest`), default AICO Root `%USERPROFILE%\aico`.
- Folder layout creation and existing-workspace detection.
- Bundled offline payload plus `releases/latest` check, download, SHA-256 verification, and fallback.
- Tasks: desktop shortcut, Start menu entry, user PATH entry, launch after install.
- Stopping a running `aico-server.exe` before file replacement; in-place upgrade.
- Silent install support for CI smoke testing.
- Installer build step integrated into `.github/workflows/release.yml`.

---

## Out-of-Scope

- Prerequisite detection/installation pages (covered by `ft-distribution-prerequisite-provisioning`).
- Uninstall behaviour beyond Inno Setup defaults (covered by `ft-distribution-uninstaller`).
- Per-machine (admin) installation, multiple concurrent registered installations per Windows user.
- Code signing; macOS/Linux installers.
- Background service, tray icon, or auto-start on login.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a new user, I want to choose an AICO Root and get a working AICO install, so that I can start using AICO immediately.
   - *Independent Test*: Silent install into `<tmp>`; assert layout and that `<tmp>/.aico/bin/aico-server.exe --version` exits 0.
2. **US-02** (Priority: P1): As the AICO developer, I want to install into `d:/aico` without losing `.aico/storage`, `specs/`, `codebase/`, or `.agents/`, so that I can dogfood the installed build.
   - *Independent Test*: Pre-populate `<tmp>/.aico/storage/marker`, `<tmp>/specs/marker`; silent install; assert markers unchanged.
3. **US-03** (Priority: P1): As a user with an older installer file, I want the installer to offer the newest release, so that I do not install an outdated version.
   - *Independent Test*: Run with a mocked latest-release endpoint reporting a higher version; assert the downloaded bundle is installed after checksum verification.
4. **US-04** (Priority: P2): As a user, I want a desktop shortcut and optional PATH entry, so that I can launch AICO conveniently.
   - *Independent Test*: Silent install with `/TASKS="desktopicon,addtopath"`; assert the `.lnk` exists and the user `Path` contains `<root>\.aico\bin`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The installer shall prompt for the AICO Root with a default of `%USERPROFILE%\aico` and shall create `specs\`, `codebase\`, `.agents\`, `.aico\storage\`, `.aico\logs\`, and `.aico\bin\` beneath it when they do not exist.
- **REQ-003**: The installer shall install per-user without requesting administrator elevation and shall place `aico-server.exe`, `skills\`, `prerequisites.json`, and `manifest.json` into `<root>\.aico\bin\`.
- **REQ-005**: The installer shall offer the tasks "Create a desktop shortcut" (checked by default), "Create a Start menu entry" (checked by default), "Add AICO to the user PATH" (unchecked by default), and "Launch AICO after installation" (checked by default).
- **REQ-009**: The installer shall support unattended installation via `/VERYSILENT /SUPPRESSMSGBOXES /DIR=<root>` with task selection via `/TASKS=` and a `/NOUPDATECHECK` switch that skips the latest-release check.

### State-Driven Requirements
- **REQ-002**: While the selected AICO Root already contains `.aico\storage\`, the installer shall display an "Existing AICO workspace detected — storage and repositories will be kept" notice and shall not modify, replace, or delete any content in `.aico\storage\`, `specs\`, `codebase\`, or `.agents\`.
- **REQ-007**: While a previous AICO installation is registered for the current user, the installer shall default the AICO Root to the previously used root and shall perform an in-place upgrade of `<root>\.aico\bin\`.

### Event-Driven Requirements
- **REQ-004**: When the GitHub `releases/latest` endpoint for `Lexer727/aico` reports a version greater than the bundled payload version, the installer shall offer to download `aico-windows-x64.zip`, shall verify its SHA-256 against `aico-windows-x64.zip.sha256`, and shall install the downloaded bundle instead of the bundled payload.
- **REQ-006**: When an `aico-server.exe` process running from `<root>\.aico\bin\` is detected before file installation, the installer shall prompt the user and terminate that process before replacing files.
- **REQ-008**: When the "Launch AICO after installation" task is selected and installation completes, the installer shall start `<root>\.aico\bin\aico-server.exe` with the AICO Root as its working directory.

### Unwanted Behavior Requirements
- **REQ-010**: If the latest-release check fails, the download fails, or the checksum does not match, then the installer shall log the reason, inform the user, and install the bundled payload.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Silent Install Smoke Test (`installer/tests/smoke-install.ps1`)**:
   - Signature: `smoke-install.ps1 -SetupExe <path> -Root <tmpdir> [-Tasks <list>] [-Preseed]`
   - Seam Test: asserts layout (REQ-001, REQ-003), preserved markers (REQ-002), shortcut/PATH effects (REQ-005), and `aico-server.exe --version` exit code; runs in the release workflow after `iscc`.
2. **Latest-Release Resolution Logic (`installer/scripts/release-check.ps1`)**:
   - Signature: `release-check.ps1 -BundledVersion <semver> -ApiUrl <url> -OutDir <path>` returning JSON `{ "action": "bundled" | "downloaded", "version": "...", "reason": "..." }`
   - Seam Test: `installer/tests/release-check.Tests.ps1` (Pester, local HTTP fixture: newer, equal, older, API failure, checksum mismatch).

### Secondary Seams:
3. **Script Compilation**:
   - Seam Test: `iscc installer/aico.iss` succeeds in CI with `/DAppVersion=<version>`.

---

## 5. Open Clarifications

- Architectural alignment: [ADR-0012](../../../../adr/0012-distribution-github-releases-and-hybrid-installer.md).
- Version comparison follows SemVer 2.0 precedence; prereleases are never offered by the latest-release check.
- An application icon (`.ico`) is required for shortcuts and the installer; if none exists in `frontend/public`, one shall be generated from the WebUI favicon.
