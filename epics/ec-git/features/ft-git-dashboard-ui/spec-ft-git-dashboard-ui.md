---
id: spec-ft-git-dashboard-ui
type: spec
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 3
title: "Dashboard Git Repository Status & Workspace Traceability UI"
status: "Completed"
derives_from: ["brain-ec-git"]
created_at: "2026-10-07T00:35:00Z"
---

# Feature Specification: Dashboard Git Repository Status & Workspace Traceability UI

## 1. Problem Statement & User Value

Developers working across multiple repositories need instant feedback on working copy state without constantly switching to terminal tabs to run `git status`. Additionally, when viewing an Epic or Feature in the WebUI, developers should immediately see which codebases are impacted.

This feature introduces a dedicated "Codebase Repositories" widget on the Home Dashboard showing real-time Git telemetry (branch, ahead/behind status, and dirty state), along with dynamic repository roll-up displays in Epic Hub and Feature Viewer.

---

## In-Scope
- "Codebase Repositories" card on the Home Dashboard (`DashboardView.tsx`).
- Visual status pills for current branch, commits ahead/behind upstream, and clean/modified status.
- "Relevant Repositories" section in the Epic Hub aggregating repositories declared in child plans.
- Repository badge display in the Feature Viewer header.
- Graceful rendering for repositories with error or unavailable states.

---

## Out-of-Scope
- In-browser Git commit, push, pull, or checkout interactions.
- Git graph / history commit DAG visualizations.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer visiting the Home Dashboard, I want to see all project repositories with their current branch and ahead/behind counts, so that I immediately know if my code is synced with upstream.
   - *Independent Test*: Mount `DashboardView` with mock Git status data and verify repository cards render branch and ahead/behind badges.
2. **US-02** (Priority: P2): As a developer viewing an Epic, I want to see which repositories are touched by that epic, so that I know the architectural blast radius.
   - *Independent Test*: Mount `EpicHubView` and verify repository badges display the union of repositories from child plans.
3. **US-03** (Priority: P2): As a developer viewing a Feature, I want to see its target repositories, so that I know which codebase it modifies.
   - *Independent Test*: Mount `FeatureViewer` and verify target repository badges render from plan frontmatter.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: The Home Dashboard shall render a "Codebase Repositories" card displaying all repositories declared in the active project manifest.
- **REQ-002**: For each discovered repository, the Home Dashboard card shall display the repository name, type badge (`codebase`, `specs`, `skills`), current branch, commits ahead/behind upstream remote, and working tree clean/dirty status.
- **REQ-003**: The Epic Hub shall dynamically aggregate and display the union of all `repositories` declared in child feature plans.
- **REQ-004**: The Feature Viewer shall display repository badge pills corresponding to the `repositories` declared in the feature's implementation plan.
- **REQ-005**: When the Git status API reports an unavailable status or communication error, the UI shall render a graceful warning badge without disrupting the remaining dashboard components.

---

## 4. Designated Public Test Seams

- **Dashboard Seam**: `frontend/tests/DashboardGitStatus.test.tsx`
- **Epic Hub Roll-up Seam**: `frontend/tests/EpicHubRepositories.test.tsx`
- **Feature Viewer Seam**: `frontend/tests/FeatureViewerRepositories.test.tsx`
