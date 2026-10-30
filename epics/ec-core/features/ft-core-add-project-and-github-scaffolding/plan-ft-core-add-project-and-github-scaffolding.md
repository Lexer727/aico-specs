---
id: plan-ft-core-add-project-and-github-scaffolding
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 20
title: "Add Project Flow, GitHub Remote Scaffolding & Codebase Browser Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-core-add-project-and-github-scaffolding"]
created_at: "2026-10-30T17:10:00.000Z"
---

# Implementation Plan: Add Project Flow, GitHub Remote Scaffolding & Codebase Browser

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Hybrid GitHub Credential Store**:
  - Implement `GitHubCredentialManager` in `crates/aico-core/src/github.rs`:
    1. Check `GITHUB_TOKEN` and `GH_TOKEN` environment variables.
    2. Check `gh auth token` via command execution if `gh` CLI exists.
    3. Check persistent user config file at `~/.aico/config.json`.
  - Expose management endpoints in `crates/aico-server/src/server.rs`:
    - `GET /api/v1/github/status`: Returns `{ "authenticated": bool, "username": Option<String>, "scopes": Vec<String> }`.
    - `POST /api/v1/github/token`: Validates token against `https://api.github.com/user` using `reqwest` with `User-Agent: aico`, saves to `~/.aico/config.json`, and returns verified username.
    - `DELETE /api/v1/github/token`: Removes stored token from `~/.aico/config.json`.
- **GitHub Repository Inspector & Creator Service**:
  - In `crates/aico-server/src/github_service.rs`:
    - `check_repository(token, url)`:
      - Extracts owner and repo from URL (e.g. `https://github.com/Lexer727/demo-specs.git` -> `Lexer727`, `demo-specs`).
      - Calls `GET https://api.github.com/repos/{owner}/{repo}`:
        - If `404 Not Found`: Returns `{ "exists": false }`.
        - If `200 OK`: Inspects repository size and branches. If size is 0, returns `{ "exists": true, "is_specs": false, "is_empty": true }`.
        - If not empty, calls `GET https://api.github.com/repos/{owner}/{repo}/contents/.aico-project.json`:
          - If found, parses JSON content and returns `{ "exists": true, "is_specs": true, "is_empty": false, "manifest": <parsed_json> }`.
          - If 404, returns `{ "exists": true, "is_specs": false, "is_empty": false }`.
    - `create_repository(token, name, private)`:
      - Calls `POST https://api.github.com/user/repos` with payload `{ "name": name, "private": private, "auto_init": false }`.
      - Returns clone URL.
- **Local Codebase Folder Browser**:
  - In `crates/aico-server/src/server.rs`, add `GET /api/v1/workspace/codebase-folders`:
    - Reads `<workspace_root>/codebase/`.
    - Returns array of `{ "id": String, "name": String, "path": String, "is_git": bool }`.
- **Project Scaffolding Engine Enhancements**:
  - Update `handle_create_project` in `crates/aico-server/src/server.rs`:
    - If `local_only == true`: creates local `<project_name>-specs` directory, calls `git init`, creates standard scaffolding files, and registers project.
    - If GitHub repo does not exist: calls `create_repository`, scaffolds directory, runs `git init`, `git remote add origin <url>`, commits scaffolding files, runs `git push -u origin main` (or `master`), and registers project.
    - If GitHub repo exists and is a specs repo: clones specs repo via `git clone <url>`, registers project, and iterates `repositories` in `.aico-project.json` to clone any missing codebase repositories into `<workspace_root>/codebase/<id>`.
    - If GitHub repo exists and non-empty/non-specs: clones or fetches existing files, scaffolds AICO structure, commits, pushes, and registers project.
- **MCP `create_project` Tool**:
  - In `crates/aico-mcp/src/server.rs`, register tool `create_project` with JSON schema matching `CreateProjectRequest`.
  - Dispatch calls through unified project creation service so agents can initialize projects autonomously.
- **Frontend UI Enhancements**:
  - Update `SettingsModal.tsx` to add a GitHub Integration card with token entry input, verify button, and connection status.
  - Update `ProjectSwitcher.tsx`:
    - Rename trigger button label from `+ New Project` to `+ Add Project`.
    - Modal title to `Add Project (.aico-project.json)`.
  - Update `ProjectConfigModal.tsx`:
    - Auto-prefill Specs Remote URL: `https://github.com/<username>/<project_name>-specs.git` as soon as `project_name` is typed.
    - Validate `-specs` suffix: if missing, render a yellow warning pill with explanatory text without blocking submission.
    - Add "Local repository only" checkbox: hides/clears GitHub URL input and disables remote checks.
    - Add "Private repository" toggle, defaulting to true.
    - Add "Check Repo" button next to GitHub URL: displays state-aware feedback banners for all 4 check cases (Auto-import specs, Empty repo, Non-specs warning with confirmation check, Repo to be created).
    - Remove `"Default codebase repository (<id>)..."` text when repositories list is empty.
    - Remove `Type` selector from linked repository rows (fixed to `codebase`).
    - Add "Browse ../codebase" picker button to easily select discovered codebase folders.

---

## 2. Task Graph (DAG)

- **TASK-CORE-020-1**: Implement GitHub Credential Store & Hybrid Auth Endpoints
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: None
  - **What it delivers**:
    - Creates `crates/aico-core/src/github.rs` providing `GitHubCredentialManager` to resolve credentials across `GITHUB_TOKEN` / `GH_TOKEN` env, `gh auth token` CLI, and `~/.aico/config.json`.
    - Adds Axum endpoints `GET /api/v1/github/status`, `POST /api/v1/github/token`, and `DELETE /api/v1/github/token` in `crates/aico-server/src/server.rs`.
  - **Verification Seam**: `crates/aico-server/tests/github_api_test.rs`

- **TASK-CORE-020-2**: Implement GitHub Repository Inspector & Creator Service
  - **Status**: Done
  - **Addresses**: `REQ-008`, `REQ-009`, `REQ-011`
  - **Blocked by**: `TASK-CORE-020-1`
  - **What it delivers**:
    - Implements GitHub REST client in `crates/aico-server/src/github_service.rs` to inspect remote repositories (`POST /api/v1/github/check-repo`), detect empty vs. non-empty vs. specs repos (parsing `.aico-project.json`), and create repositories on GitHub (`POST /user/repos`).
  - **Verification Seam**: `crates/aico-server/tests/github_api_test.rs`

- **TASK-CORE-020-3**: Implement Local Codebase Folder Browser Endpoint
  - **Status**: Done
  - **Addresses**: `REQ-003`
  - **Blocked by**: None
  - **What it delivers**:
    - Implements `GET /api/v1/workspace/codebase-folders` in `crates/aico-server/src/server.rs` to scan `<workspace_root>/codebase/` and return discovered folder items (`id`, `name`, `path`, `is_git`).
  - **Verification Seam**: `crates/aico-server/tests/github_api_test.rs`

- **TASK-CORE-020-4**: Extend Project Scaffolding Engine for GitHub Creation, Push & Remote Specs Import
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-007`, `REQ-009`, `REQ-011`
  - **Blocked by**: `TASK-CORE-020-1`, `TASK-CORE-020-2`
  - **What it delivers**:
    - Updates `handle_create_project` in `crates/aico-server/src/server.rs` to support `local_only`, GitHub remote creation with public/private toggle, initial Git push, and remote specs repository cloning with automatic linked codebase repositories cloning into `<workspace_root>/codebase/<id>`.
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs`

- **TASK-CORE-020-5**: Add `create_project` Tool to MCP Server for Agent Automation
  - **Status**: Done
  - **Addresses**: `REQ-010`
  - **Blocked by**: `TASK-CORE-020-4`
  - **What it delivers**:
    - Adds `create_project` tool definition to `crates/aico-mcp/src/server.rs` accepting `project_name`, `display_name`, `url`, `repositories`, `local_only`, and `private`, delegating to the unified project creation service.
  - **Verification Seam**: `crates/aico-mcp/tests/mcp_server_test.rs`

- **TASK-CORE-020-6**: Add GitHub Integration Section to Settings Modal
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-CORE-020-1`
  - **What it delivers**:
    - Updates `frontend/src/components/settings/SettingsModal.tsx` to include a GitHub Connection card with token entry, account verification against `/api/v1/github/token`, and connected user profile badge.
  - **Verification Seam**: `frontend/tests/SettingsModal.test.tsx`

- **TASK-CORE-020-7**: Overhaul Project Config Modal to Add Project Flow & Codebase Browser
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-005`, `REQ-006`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-CORE-020-2`, `TASK-CORE-020-3`, `TASK-CORE-020-4`, `TASK-CORE-020-6`
  - **What it delivers**:
    - Updates `ProjectSwitcher.tsx` with `+ Add Project` label.
    - Updates `ProjectConfigModal.tsx` to "Add Project", dynamic `-specs` URL pre-fill, permissive `-specs` warning, "Local repository only" checkbox, private/public toggle, "Check Repo" inspection feedback, codebase folder browse picker, and removes legacy default text and `Type` dropdown.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`

- **TASK-CORE-020-8**: End-to-End Test Suite Verification across Backend Seams and Frontend Tests
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-006`, `REQ-007`, `REQ-008`, `REQ-009`, `REQ-010`, `REQ-011`
  - **Blocked by**: `TASK-CORE-020-5`, `TASK-CORE-020-7`
  - **What it delivers**:
    - Comprehensive unit and integration test coverage across all new endpoints and UI components, verifying zero regressions across backend and frontend test suites.
  - **Verification Seam**: `crates/aico-server/tests/github_api_test.rs`, `crates/aico-server/tests/project_api_test.rs`, `frontend/tests/ProjectSwitcher.test.tsx`, and `frontend/tests/SettingsModal.test.tsx`
