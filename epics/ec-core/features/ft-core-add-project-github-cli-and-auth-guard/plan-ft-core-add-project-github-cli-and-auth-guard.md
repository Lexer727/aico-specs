---
id: plan-ft-core-add-project-github-cli-and-auth-guard
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 21
title: "GitHub CLI Login, Unauthenticated Private Repo Guard & Add Project Defaults Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-core-add-project-github-cli-and-auth-guard"]
created_at: "2026-10-30T17:55:00.000Z"
---

# Implementation Plan: GitHub CLI Login, Unauthenticated Private Repo Guard & Add Project Defaults

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Robust `gh` Executable Discovery**:
  - In `crates/aico-core/src/github.rs`, implement `find_gh_binary() -> Option<PathBuf>`:
    1. Check system PATH via `where gh` or direct executable search.
    2. Fall back to standard Windows installation paths:
       - `C:\Program Files\GitHub CLI\gh.exe`
       - `C:\Program Files (x86)\GitHub CLI\gh.exe`
       - `%LOCALAPPDATA%\Programs\GitHub CLI\gh.exe`
    3. Update `resolve_token()` in `GitHubCredentialManager` to use `find_gh_binary()` when querying `gh auth token`.
- **GitHub CLI REST Endpoints**:
  - In `crates/aico-server/src/server.rs`:
    - `GET /api/v1/github/cli/status`: Executes `gh auth status` via the discovered binary, parses output to detect logged-in account, and returns `{ "installed": bool, "binary_path": Option<String>, "logged_in": bool, "username": Option<String> }`.
    - `POST /api/v1/github/cli/sync`: Executes `gh auth token`, calls GitHub `/user` REST API to verify token and retrieve user profile, writes to `~/.aico/config.json`, and returns `{ "status": "ok", "username": username }`.
- **Unauthenticated Private Repository Guard**:
  - In `crates/aico-server/src/github_service.rs`, update `check_repository(url)`:
    - If `token` is `None` and `api.github.com/repos/{owner}/{repo}` returns `404 Not Found`:
      - Return `CheckRepoResponse { exists: false, is_specs: false, is_empty: false, unauthenticated: true, error: Some("Repository was not found (or is a private repository). Connect your GitHub account in Settings to verify private repositories or create new ones on GitHub.".to_string()), .. }`.
- **Add Project Modal Defaults**:
  - In `frontend/src/components/navigation/ProjectConfigModal.tsx`:
    - When `!githubStatus?.authenticated`:
      - Default `localOnly: true`.
      - Never pre-fill `<username>` into `specsUrl`.
      - When `localOnly` is unchecked by user while unauthenticated:
        - Render an alert callout: *"GitHub account is not connected. Private repositories cannot be checked and remote repositories cannot be created without authentication."*
        - Render button: `"Connect GitHub in Settings"`.
      - In `handleCheckRepo`:
        - If `checkResult?.unauthenticated`, display an amber authentication guidance banner explaining private repo ambiguity and prompting connection in Settings.
- **Settings Modal Dual Login Modes**:
  - In `frontend/src/components/settings/SettingsModal.tsx`:
    - In `section-github-integration`, display GitHub CLI status:
      - If installed: show detected path and login state.
      - Add `"Connect with GitHub CLI"` button which calls `POST /api/v1/github/cli/sync`.
      - Offer Personal Access Token (PAT) input as alternative.

---

## 2. Task Graph (DAG)

- **TASK-CORE-021-1**: Implement Robust `gh` Discovery & CLI Status Endpoint
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`
  - **Blocked by**: None
  - **What it delivers**:
    - Implements `find_gh_binary()` in `crates/aico-core/src/github.rs` and exposes `GET /api/v1/github/cli/status` in `crates/aico-server/src/server.rs`.
  - **Verification Seam**: `crates/aico-server/tests/github_cli_api_test.rs`

- **TASK-CORE-021-2**: Implement GitHub CLI Credential Sync Endpoint
  - **Status**: Done
  - **Addresses**: `REQ-006`
  - **Blocked by**: `TASK-CORE-021-1`
  - **What it delivers**:
    - Implements `POST /api/v1/github/cli/sync` in `crates/aico-server/src/server.rs` to pull token from `gh auth token`, validate against GitHub `/user`, and persist to config.
  - **Verification Seam**: `crates/aico-server/tests/github_cli_api_test.rs`

- **TASK-CORE-021-3**: Add Unauthenticated Private Repo Guard in Repository Inspector
  - **Status**: Done
  - **Addresses**: `REQ-005`
  - **Blocked by**: None
  - **What it delivers**:
    - Updates `CheckRepoResponse` and `check_repository` in `crates/aico-server/src/github_service.rs` to flag `unauthenticated: true` on 404 when no token is present.
  - **Verification Seam**: `crates/aico-server/tests/github_cli_api_test.rs`

- **TASK-CORE-021-4**: Add GitHub CLI Integration Option to Settings Modal
  - **Status**: Done
  - **Addresses**: `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-CORE-021-1`, `TASK-CORE-021-2`
  - **What it delivers**:
    - Enhances `SettingsModal.tsx` to detect `gh` CLI and support one-click sync alongside PAT entry.
  - **Verification Seam**: `frontend/tests/SettingsModal.test.tsx`

- **TASK-CORE-021-5**: Overhaul Add Project Modal for Unauthenticated Defaults & Alert Guard
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-004`
  - **Blocked by**: `TASK-CORE-021-3`
  - **What it delivers**:
    - Updates `ProjectConfigModal.tsx` to default to local-only when not authenticated, omit `<username>` placeholder, and show alert notice with Settings trigger.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`

- **TASK-CORE-021-6**: Comprehensive Integration and UI Verification across Seams
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-006`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-CORE-021-4`, `TASK-CORE-021-5`
  - **What it delivers**:
    - Full end-to-end test coverage verifying all backend endpoints and frontend components with zero regressions.
  - **Verification Seam**: `crates/aico-server/tests/github_cli_api_test.rs`, `frontend/tests/ProjectSwitcher.test.tsx`, `frontend/tests/SettingsModal.test.tsx`
