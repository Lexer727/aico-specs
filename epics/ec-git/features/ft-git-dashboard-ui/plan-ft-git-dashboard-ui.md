---
id: plan-ft-git-dashboard-ui
type: plan
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 3
title: "Dashboard Git Repository Status & Workspace Traceability UI Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-git-dashboard-ui"]
created_at: "2026-10-07T00:36:00Z"
---

# Implementation Plan: Dashboard Git Repository Status & Workspace Traceability UI

## 1. Technical Architecture & Decisions

- **Git Status Hook & State**: Create a lightweight `useGitStatus` hook or API query fetching `GET /api/v1/git/status`.
- **Telemetry Card UI**: Render a clean "Codebase Repositories" card on the Home Dashboard featuring branch icons, ahead/behind indicators, and working directory clean/dirty indicators.
- **Dynamic Frontmatter Roll-Up**:
  - In `FeatureViewer.tsx`, read `repositories` directly from the feature's implementation plan frontmatter.
  - In `EpicHubView.tsx`, compute the unique union of all repositories declared across child feature plans and render them as high-visibility badge pills in the epic header.
- **Resilient Fallback**: Handle offline, missing, or erroring repositories gracefully with an "Unavailable" badge.

---

## 2. Task Graph (DAG)

### Phase 1: Dashboard Git Telemetry

- **TASK-GIT-003-1**: Home Dashboard Repositories Telemetry Card
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-005
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `GitRepositoriesCard` component displaying all repositories with branch, ahead/behind counts, and clean status.
    - Integration into `DashboardView.tsx`.
  - **Verification Seam**: `frontend/tests/DashboardGitStatus.test.tsx`

### Phase 2: Epic & Feature Dynamic Roll-up

- **TASK-GIT-003-2**: Dynamic Epic & Feature Workspace Repository Badges
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004
  - **Blocked by**: TASK-GIT-003-1
  - **What it delivers**:
    - Aggregated repository badges in `EpicHubView.tsx` from child feature plans.
    - Target repository tags in `FeatureViewer.tsx` header.
  - **Verification Seam**: `frontend/tests/EpicHubRepositories.test.tsx`
