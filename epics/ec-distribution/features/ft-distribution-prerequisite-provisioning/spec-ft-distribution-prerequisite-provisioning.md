---
id: spec-ft-distribution-prerequisite-provisioning
type: spec
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 4
title: "Prerequisite Detection & Provisioning via winget"
status: "Completed"
derives_from: ["brain-ec-distribution", "ADR-0012"]
created_at: "2026-10-30T19:19:09.252Z"
---

# Feature Specification: Prerequisite Detection & Provisioning via winget

## 1. Problem Statement & User Value

AICO relies on external command-line tools for day-to-day workflows: Git for multi-repository management, GitHub CLI (`gh`) for project creation and remote authentication, Node.js LTS for running deterministic SDD skill scripts (`aico-cli.mjs`, `verify-artifacts.mjs`), and AI coding agent harnesses (Claude Code, Antigravity, OpenAI Codex CLI, GitHub Copilot CLI). New users often lack one or more of these tools.

This feature equips the installer with a data-driven **Prerequisite Catalog** (`prerequisites.json`) and a wizard step that detects already-installed tools (displaying their versions as disabled/installed), allows users to check or uncheck missing tools, and provisions selected tools automatically via `winget` (with catalog-defined npm or URL fallbacks) without failing the core AICO installation if an optional tool install fails.

---

## In-Scope

- A JSON Prerequisite Catalog (`installer/prerequisites.json`, bundled into `.aico/bin/prerequisites.json`) declaring each tool's `id`, `name`, `category`, `required`, `default_checked`, `detect_cmd`, `winget_id`, `npm_package` fallback, and `homepage_url`.
- Catalog entries for:
  - **Git** (`Git.Git`, required, checked by default)
  - **GitHub CLI** (`GitHub.cli`, optional, checked by default)
  - **Node.js LTS** (`OpenJS.NodeJS.LTS`, optional, checked by default)
  - **Claude Code** (`Anthropic.ClaudeCode` / npm `@anthropic-ai/claude-code`, optional, unchecked by default)
  - **Google Antigravity** (detect via PATH/AppData, winget/URL fallback, optional, unchecked by default)
  - **OpenAI Codex CLI** (npm `@openai/codex`, optional, unchecked by default)
  - **GitHub Copilot CLI** (`GitHub.Copilot` / `gh extension`, optional, unchecked by default)
- PowerShell helper script `installer/scripts/prereq-manager.ps1` supporting `-Action detect` (returns JSON status/version per item) and `-Action install -Ids <csv>`.
- Inno Setup custom wizard page displaying each prerequisite, greying out already-installed entries with their detected version, and executing provisioning for selected items during installation.

---

## Out-of-Scope

- Uninstalling prerequisites when AICO is uninstalled.
- Installing developer-only build tools (`rustc`, `cargo`, `protoc`, Visual Studio Build Tools).
- Managing API keys or interactive OAuth sign-in flows for AI harnesses during setup.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a user installing AICO on a machine that already has Git and Node.js, I want the installer to detect them automatically and show their installed versions, so that I do not reinstall them unnecessarily.
   - *Independent Test*: Run `prereq-manager.ps1 -Action detect` on a machine with Git installed; verify the JSON output marks `git` as `installed: true` with a non-empty version string.
2. **US-02** (Priority: P1): As a user missing GitHub CLI (`gh`), I want to leave the GitHub CLI checkbox ticked in the installer and have it installed automatically via `winget`, so that AICO's GitHub scaffolding works out of the box.
   - *Independent Test*: Run `prereq-manager.ps1 -Action install -Ids gh -DryRun`; verify the exact `winget install --id GitHub.cli` command is constructed.
3. **US-03** (Priority: P2): As a user who selects an optional AI CLI whose installation fails due to network or package unavailability, I want the installer to report the failure clearly while still completing the AICO installation.
   - *Independent Test*: Simulate a failing prerequisite install in `prereq-manager.ps1`; verify exit summary reports the failed item while AICO binary installation succeeds.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The system shall load prerequisite definitions from `prerequisites.json`, where each entry specifies `id`, `name`, `description`, `required`, `default_checked`, `detect_cmd`, `winget_id`, `npm_package`, and `homepage_url`.
- **REQ-002**: The Prerequisite Catalog shall include entries for Git (`git`), GitHub CLI (`gh`), Node.js LTS (`node`), Claude Code (`claude-code`), Antigravity (`antigravity`), OpenAI Codex CLI (`codex`), and GitHub Copilot CLI (`copilot`).

### Event-Driven Requirements
- **REQ-003**: When the installer initializes the Prerequisites wizard page, the system shall execute the detection command for each catalog entry and display installed prerequisites as pre-checked and disabled alongside their detected version string.
- **REQ-004**: When the user proceeds with selected uninstalled prerequisites, the system shall install each selected prerequisite using `winget install --id <winget_id> --exact --silent --accept-package-agreements --accept-source-agreements` when `winget` is available and `winget_id` is defined.

### State-Driven Requirements
- **REQ-005**: While a selected prerequisite has no `winget_id` (or `winget` is unavailable) and defines an `npm_package` while `npm` is available on PATH, the system shall install the prerequisite via `npm install -g <npm_package>`.

### Unwanted Behavior Requirements
- **REQ-006**: If the required prerequisite (`git`) is neither detected as installed nor selected for installation, then the installer shall display a warning dialog explaining that AICO requires Git for repository operations before allowing the user to continue.
- **REQ-007**: If installing a prerequisite fails or neither `winget` nor `npm` can provision it, then the system shall record the failure reason, offer to open the prerequisite's `homepage_url` on the finish page, and complete the AICO installation without aborting.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Prerequisite Catalog Schema & Contents (`installer/prerequisites.json`)**:
   - Seam Test: `installer/tests/prereq-manager.Tests.ps1::Describe 'Prerequisite Catalog Schema'` (validates JSON syntax, required fields, and presence of all 7 required IDs).
2. **Detection & Provisioning Engine (`installer/scripts/prereq-manager.ps1`)**:
   - Signature: `prereq-manager.ps1 -Action <detect|install> -CatalogPath <path> [-Ids <string[]>] [-DryRun]`
   - Seam Test: `installer/tests/prereq-manager.Tests.ps1::Describe 'Detection and Install Command Resolution'` (verifies detection output structure, winget command construction, npm fallback, and non-fatal error handling on mocked failures).

---

## 5. Open Clarifications

- Architectural alignment: [ADR-0012](../../../../adr/0012-distribution-github-releases-and-hybrid-installer.md).
- When running in `/VERYSILENT` mode without `/INSTALLPREREQS`, the installer skips prerequisite installation so CI smoke tests remain fast and isolated.
