---
id: plan-ft-distribution-update-check
type: plan
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 6
title: "Release Update Check & WebUI Banner Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-distribution-update-check"]
created_at: "2026-10-30T19:19:09.613Z"
---

# Implementation Plan: Release Update Check & WebUI Banner

## 1. Technical Architecture & Decisions

- **Target repository**: `aico` (`crates/aico-server`, `frontend/`).
- **Backend endpoint** `GET /api/v1/system/update`:
  - Implemented in `crates/aico-server/src/server.rs` (or `crates/aico-server/src/update.rs`) with a `tokio::sync::RwLock<Option<CachedUpdateStatus>>` on `AppState` (TTL: 6 hours).
  - Queries `AICO_UPDATE_API_URL` (default `https://api.github.com/repos/Lexer727/aico/releases/latest`) with a 5-second timeout and `User-Agent: aico-server/<version>`.
  - Compares SemVer tuple `(major, minor, patch)` stripped of leading `v`; ignores prereleases.
  - Degrades to `update_available: false` on error or when `AICO_NO_UPDATE_CHECK=1`.
- **Frontend banner**:
  - `UpdateBanner` component in `frontend/src/components/layout/UpdateBanner.tsx` mounted in the top shell header.
  - Fetches `/api/v1/system/update` once on mount; stores dismissed `latest_version` in `localStorage.getItem('aico_dismissed_update_version')`.

---

## 2. Task Graph (DAG)

### Phase 1: Core Vertical Tracer Bullet

- **TASK-DISTRIBUTION-006-1**: Backend `GET /api/v1/system/update` Endpoint with Caching & Graceful Fallback
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-003`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - SemVer comparison, 6-hour cached GitHub release check, `AICO_UPDATE_API_URL` and `AICO_NO_UPDATE_CHECK` handling, and route registration at `/api/v1/system/update`.
    - Integration tests in `crates/aico-server/tests/update_check_test.rs`.
  - **Verification Seam**: `crates/aico-server/tests/update_check_test.rs`

- **TASK-DISTRIBUTION-006-2**: WebUI Update Notification Banner & Dismissal Persistence
  - **Status**: Done
  - **Addresses**: `REQ-002`
  - **Blocked by**: `TASK-DISTRIBUTION-006-1`
  - **What it delivers**:
    - `UpdateBanner` React component integrated into the WebUI header with `localStorage` per-version dismissal.
    - Vitest component tests in `frontend/tests/UpdateBanner.test.tsx`.
  - **Verification Seam**: `frontend/tests/UpdateBanner.test.tsx`
