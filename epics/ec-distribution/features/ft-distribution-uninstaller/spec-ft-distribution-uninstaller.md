---
id: spec-ft-distribution-uninstaller
type: spec
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 5
title: "Scoped Uninstaller in Install Bin Directory"
status: "Completed"
derives_from: ["brain-ec-distribution", "ADR-0012"]
created_at: "2026-10-30T19:19:09.432Z"
---

# Feature Specification: Scoped Uninstaller in Install Bin Directory

## 1. Problem Statement & User Value

Because AICO installs into `<root>/.aico/bin` inside a workspace that also holds user-authored specifications (`specs/`), source code (`codebase/`), and embedded graph/vector databases (`.aico/storage`), a naive uninstaller that deletes the installation directory would cause catastrophic data loss. Conversely, leaving orphaned binaries, shortcuts, or PATH entries pollutes the user's machine.

This feature places a scoped uninstaller inside `<root>/.aico/bin` (and registers it in Windows Apps & Features) that removes `.aico/bin`, shortcuts, user PATH entries, and registry keys by default, offers an explicit opt-in prompt for deleting `.aico/storage`, `.aico/logs`, and AICO-managed `.agents/skills/aico-*` entries, and strictly preserves `specs/`, `codebase/`, `~/.aico`, and third-party tools.

---

## In-Scope

- Placing Inno Setup's uninstaller (`unins000.exe` and `unins000.dat`) and a convenience wrapper `aico-uninstall.cmd` inside `<root>/.aico/bin`.
- Per-user Windows Apps & Features registration (`HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall\...`).
- Stopping any running `aico-server.exe` from `<root>/.aico/bin` prior to file removal.
- Default removal of `<root>/.aico/bin`, Desktop and Start menu shortcuts, the `<root>/.aico/bin` entry in user `Path`, and the uninstall registry key.
- Interactive opt-in prompt (or `/PURGEDATA` CLI flag in silent mode) to additionally delete `<root>/.aico/storage`, `<root>/.aico/logs`, and `<root>/.agents/skills/aico-*`.
- Strict preservation guarantees for `<root>/specs`, `<root>/codebase`, User Config Home (`~/.aico`), and winget-installed prerequisites.

---

## Out-of-Scope

- Removing repositories inside `specs/` or `codebase/`.
- Uninstalling Git, GitHub CLI, Node.js, or AI harnesses installed during setup.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer or user uninstalling AICO, I want `.aico/bin` and shortcuts removed while my `specs/`, `codebase/`, and `.aico/storage` remain untouched by default, so that I can reinstall later or keep my work without losing data.
   - *Independent Test*: Run the uninstaller silently (`/VERYSILENT`) on a root containing `specs/marker.txt`, `codebase/marker.txt`, and `.aico/storage/marker.txt`; verify `.aico/bin` is removed and all three markers remain intact.
2. **US-02** (Priority: P2): As a user who wants a complete cleanup of AICO runtime state, I want an opt-in option during uninstall to also delete `.aico/storage`, `.aico/logs`, and `.agents/skills/aico-*`, while still keeping my `specs/` and `codebase/`.
   - *Independent Test*: Run the uninstaller with `/VERYSILENT /PURGEDATA`; verify `.aico/bin`, `.aico/storage`, `.aico/logs`, and `.agents/skills/aico-*` are removed while `specs/marker.txt` and `codebase/marker.txt` survive.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The installer shall place the uninstaller executable and metadata inside `<root>\.aico\bin\` and register the uninstaller under the current user's Windows Uninstall registry key.
- **REQ-004**: The uninstaller shall never delete or modify `<root>\specs\`, `<root>\codebase\`, the User Config Home (`%USERPROFILE%\.aico`), or non-`aico-*` subdirectories inside `<root>\.agents\`.

### Event-Driven Requirements
- **REQ-002**: When uninstallation begins, the uninstaller shall terminate any running `aico-server.exe` process whose executable resides in `<root>\.aico\bin\` and shall remove `<root>\.aico\bin\`, Desktop and Start menu shortcuts, the `<root>\.aico\bin` user `Path` entry, and the Uninstall registry entry.

### Optional Feature Requirements
- **REQ-003**: Where the user confirms the optional "Also remove AICO database storage, logs, and synced agent skills" prompt (or passes `/PURGEDATA` in silent mode), the uninstaller shall delete `<root>\.aico\storage\`, `<root>\.aico\logs\`, `<root>\.agents\skills\aico-*\`, and `<root>\.aico\` if empty.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Default Scoped Uninstall Smoke Test (`installer/tests/smoke-uninstall.ps1`)**:
   - Signature: `smoke-uninstall.ps1 -Root <tmpdir> [-PurgeData]`
   - Seam Test: Pre-populates `specs/marker`, `codebase/marker`, `.aico/storage/marker`, `.agents/skills/aico-start/SKILL.md`, and `.agents/skills/custom-skill/SKILL.md`; runs uninstaller in default mode and asserts `.aico/bin` is gone while all user/storage markers survive.
2. **Purge-Data Uninstall Smoke Test (`installer/tests/smoke-uninstall.ps1 -PurgeData`)**:
   - Seam Test: Runs uninstaller with `/VERYSILENT /PURGEDATA` and asserts `.aico/storage`, `.aico/logs`, and `.agents/skills/aico-*` are removed while `specs/marker`, `codebase/marker`, and `.agents/skills/custom-skill/SKILL.md` survive.

---

## 5. Open Clarifications

- Architectural alignment: [ADR-0012](../../../../adr/0012-distribution-github-releases-and-hybrid-installer.md).
- In silent mode (`/VERYSILENT`) without `/PURGEDATA`, the opt-in prompt defaults to **No** (preserving `.aico/storage`, `.aico/logs`, and `.agents/`).
