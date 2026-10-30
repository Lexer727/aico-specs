---
id: spec-ft-git-remote-fetch-and-sync
type: spec
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 6
title: "Git Remote Fetch & Status Synchronization"
status: "Completed"
derives_from: ["brain-ec-git"]
created_at: "2026-10-30T21:28:00Z"
---

# Feature Specification: Git Remote Fetch & Status Synchronization

## 1. Problem Statement & User Value

The AICO Git dashboard card displays repository branch status, staged/unstaged changes, and commit metrics (ahead / behind upstream). However, the underlying inspection engine (`GitStatusScanner::scan_repo`) only invokes local Git commands (`git status --porcelain=v2 --branch` and `git log -1`). It never executes a remote `git fetch`.

Consequently, when remote changes occur on upstream branches (e.g. commits pushed to GitHub by collaborators, CI actions, or web edits), local tracking references (`refs/remotes/origin/*`) remain stale. The dashboard continues to report `behind: 0` and clicking "Refresh" simply re-queries the stale local references. Changes are only reflected if an external tool (such as VS Code's background fetch) runs `git fetch`.

This feature introduces on-demand remote fetch capabilities via `GET /api/v1/git/status?fetch=true`, resilient timeout and offline fallback handling, a periodic background auto-fetch task in the AICO server daemon, and frontend Refresh integration with live loading indicators.

---

## In-Scope

- Query parameter support on `GET /api/v1/git/status?fetch=true` to fetch upstream branches prior to reading status.
- Resilient fetch execution with a 5-second per-repository timeout and graceful offline/auth error handling.
- Periodic background auto-fetch service running inside `aico-server` on a 5-minute interval.
- Frontend `GitRepositoriesCard.tsx` updating the Refresh button to trigger `fetchGitStatus({ fetch: true })` and animating a spinner while fetching.
- Integration tests simulating remote commits and validating ahead/behind synchronization.
- Component tests verifying frontend fetch parameter dispatch and spinning visual feedback.

---

## Out-of-Scope

- Automatic `git pull` or automated merge conflict resolution (mutating working trees without user confirmation).
- Credential entry dialogs or interactive SSH passphrase prompts.
- Modifying remote URL origins or branch upstream tracking configurations.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer viewing the AICO Dashboard, I want clicking the "Refresh" button to fetch recent changes from remote Git repositories, so that I immediately see if remote commits are available to pull without opening an external IDE.
   - *Independent Test*: Push a commit to a remote upstream branch; click "Refresh" on the AICO Dashboard; verify the repository badge displays `1 behind` and enables the "Pull" action.

2. **US-02** (Priority: P1): As an offline or disconnected developer, I want the Git refresh and status endpoints to degrade gracefully when remotes are unreachable, so that my dashboard loads quickly without crashing, timing out indefinitely, or failing with a 500 internal server error.
   - *Independent Test*: Query `GET /api/v1/git/status?fetch=true` while network access is disabled or remote URLs are unreachable; verify response status is 200 OK within 5 seconds with local git status data and descriptive warning telemetry.

3. **US-03** (Priority: P2): As a developer working continuously in AICO, I want the server daemon to periodically fetch remote repositories in the background, so that my git telemetry remains up to date even if I have not clicked Refresh.
   - *Independent Test*: Leave the server running with an upstream remote that receives a commit; verify that within 5 minutes the server has fetched remote refs and telemetry reports behind count > 0.

---

## 3. Behavioral Requirements (EARS)

- **REQ-001**: WHEN a client submits `GET /api/v1/git/status` with query parameter `fetch=true`, the server SHALL execute `git fetch` across all configured repositories before returning repository status.
- **REQ-002**: IF a `git fetch` operation exceeds a 5-second timeout or encounters a network or authentication error, THEN the server SHALL record the failure without terminating the request and SHALL return the local repository status with error telemetry.
- **REQ-003**: The server SHALL run a periodic background synchronization task at a 5-minute interval that performs `git fetch` across configured repositories to refresh local remote tracking branches.
- **REQ-004**: WHEN a user activates the Refresh control in the Git Repositories dashboard card, the client SHALL issue `GET /api/v1/git/status?fetch=true` and display a spinning loading indicator on the Refresh button until the response completes.
- **REQ-005**: WHILE a git fetch operation is active in the background, the server SHALL prevent concurrent git operations from corrupting repository lock files.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Git Fetch & Status Endpoint (`crates/aico-server/tests/git_fetch_test.rs`)**:
   - Integration Test: Verifies `GET /api/v1/git/status?fetch=true` fetches from upstream bare remote, updates `behind` count, and handles unreachable remotes gracefully within 5 seconds.
2. **Dashboard Git Repositories Card (`frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`)**:
   - Component Test: Verifies clicking the Refresh button dispatches `fetchGitStatus({ fetch: true })` and displays the spinning loading indicator.
