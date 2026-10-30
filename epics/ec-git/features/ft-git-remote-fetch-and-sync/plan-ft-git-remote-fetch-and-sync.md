---
id: plan-ft-git-remote-fetch-and-sync
type: plan
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 6
title: "Git Remote Fetch & Status Synchronization Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-git-remote-fetch-and-sync"]
created_at: "2026-10-30T21:28:00Z"
---

# Implementation Plan: Git Remote Fetch & Status Synchronization

## 1. Technical Architecture & Decisions

- **Target repositories**: `aico` (`crates/aico-server`, `frontend/`), `aico-specs`.
- **Backend On-Demand Fetch (`GET /api/v1/git/status?fetch=true`)**:
  - In `crates/aico-server/src/git.rs`:
    - Add `git_fetch_repo(repo_dir: &Path, timeout_secs: u64) -> Result<(), String>`.
    - Timeout enforced using `tokio::time::timeout(Duration::from_secs(5), ...)`.
    - Resilient error handling: if `git fetch` fails (e.g. offline, remote down, unauthenticated), log the error and record it in `GitRepoStatus.error` or warning telemetry, but proceed to scan and return local repository status.
    - Concurrency protection: maintain a repository lock map (or avoid overlapping git mutations) to prevent `.git/index.lock` collisions.
    - Update `collect_project_git_status(specs_root: &Path, do_fetch: bool)` to execute `git_fetch_repo` when `do_fetch` is `true`.
  - In `crates/aico-server/src/server.rs`:
    - Update `ProjectQuery` to include `fetch: Option<bool>`.
    - Pass `query.fetch.unwrap_or(false)` to `collect_project_git_status`.
- **Periodic Background Auto-Fetch Service**:
  - In `crates/aico-server/src/server.rs`, spawn a background tokio task during server initialization:
    - Runs every 5 minutes (300 seconds).
    - Iterates over configured repositories in active project contexts and invokes `git_fetch_repo`.
    - Fails open silently on network/auth errors without disrupting server operation.
- **Frontend Dashboard Refresh Integration**:
  - In `frontend/src/features/feature-task-data.ts`:
    - Update `fetchGitStatus(options?: { fetch?: boolean })` to append `?fetch=true` when requested.
  - In `frontend/src/features/dashboard/GitRepositoriesCard.tsx`:
    - Update `fetchRepos` to pass `{ fetch: true }` when initiated via the Refresh button.
    - Ensure the `RefreshCw` icon animates with `animate-spin` while the fetch request is in flight.

---

## 2. Task Graph (DAG)

### Phase 1: Test Scaffolding & Design Verification

- **TASK-GIT-006-1**: Scaffold Integration Tests for Remote Fetch & Offline Resilience
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Creates `crates/aico-server/tests/git_fetch_test.rs` creating temporary local repositories with bare remotes, asserting that `?fetch=true` detects new remote commits, and verifying offline/unreachable remotes degrade gracefully.
  - **Verification Seam**: `crates/aico-server/tests/git_fetch_test.rs`

- **TASK-GIT-006-4**: Scaffold Frontend Component Tests for Refresh Fetch Parameter
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Adds tests in `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx` verifying that clicking Refresh triggers `fetchGitStatus` with `{ fetch: true }` and shows the loading spinner.
  - **Verification Seam**: `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`

### Phase 2: Backend Implementation & Background Task

- **TASK-GIT-006-2**: Implement Backend Remote Fetch Execution with 5-Second Timeout & Graceful Error Fallback
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-005`
  - **Blocked by**: `TASK-GIT-006-1`
  - **What it delivers**:
    - Implements `git_fetch_repo` with 5s timeout and concurrency-safe execution in `crates/aico-server/src/git.rs`.
    - Updates `collect_project_git_status` and `ProjectQuery` in `crates/aico-server/src/server.rs`.
  - **Verification Seam**: `crates/aico-server/tests/git_fetch_test.rs`

- **TASK-GIT-006-3**: Implement Periodic Background Auto-Fetch Synchronization Loop
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-005`
  - **Blocked by**: `TASK-GIT-006-2`
  - **What it delivers**:
    - Adds the 5-minute background auto-fetch loop to `crates/aico-server/src/server.rs`.
  - **Verification Seam**: `crates/aico-server/tests/git_fetch_test.rs`

### Phase 3: Frontend Integration & End-to-End Verification

- **TASK-GIT-006-5**: Wire Frontend Refresh to Pass `fetch=true` & Display Feedback
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-GIT-006-4`, `TASK-GIT-006-2`
  - **What it delivers**:
    - Updates `fetchGitStatus` in `feature-task-data.ts` and `GitRepositoriesCard.tsx`.
  - **Verification Seam**: `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`

- **TASK-GIT-006-6**: End-to-End Verification & Gate
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`
  - **Blocked by**: `TASK-GIT-006-3`, `TASK-GIT-006-5`
  - **What it delivers**:
    - Full test suite execution (`cargo test`, Vitest) and blueprint validation with `verify-artifacts.mjs`.
  - **Verification Seam**: `crates/aico-server/tests/git_fetch_test.rs`
