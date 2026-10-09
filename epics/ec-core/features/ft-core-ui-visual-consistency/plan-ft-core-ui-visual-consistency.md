---
id: plan-ft-core-ui-visual-consistency
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 10
title: "WebUI Visual Token Alignment and Shell Consistency Plan"
status: "Draft"
repositories: ["aico"]
derives_from: ["spec-ft-core-ui-visual-consistency"]
created_at: "2026-10-09T19:16:00Z"
---

# Implementation Plan: WebUI Visual Token Alignment and Shell Consistency

## 1. Technical Architecture & Decisions

- **Top-Level Shortcut Trigger Placement (`App.tsx`, `ShellLayout.tsx`)**:
  - The primary global OmniSearch button is housed in the top application header (`App.tsx`) immediately adjacent to `ProjectSwitcher`.
  - The duplicate trigger button in `ShellLayout.tsx` sidebar header (positioned between the workspace brand and `v0.1.1` version badge) will be removed. The version badge will remain in the sidebar header.
- **Action Button Layout Rigidization (`DashboardView.tsx`)**:
  - In the Project Epics grid on `DashboardView.tsx`, the action button container will apply `shrink-0 whitespace-nowrap` to prevent the button from breaking across multiple lines when an epic title is long.
  - The adjacent epic metadata header container will apply `min-w-0 flex-1 mr-4` to allow titles to flow naturally without compressing the action button.
- **Canonical Feature Color Token Alignment**:
  - Standardize all Feature representation to **Purple** (`text-purple-400`, `Package` icon, `bg-purple-950/70`, `border-purple-800`).
  - Update `ShellLayout.tsx` Features accordion header icon to `<Package className="w-4 h-4 text-purple-400" />` and standalone badge to purple.
  - Update `TabBar.tsx` `getIcon('feature')` to `<Package className={clsx(iconClass, 'text-purple-400')} />`.
  - Update `EpicHubView.tsx` Child Features section header icon to `<Package className="w-4 h-4 text-purple-400 flex-shrink-0" />`.
  - OmniSearch modal already uses `text-purple-400` and `TagBadge` already uses purple for features.
- **Specs Repository Badge Parity (`GitRepositoriesCard.tsx`)**:
  - In `GitRepositoriesCard.tsx`, update `getTypeBadgeColor('specs')` from purple to canonical **Sky** styling: `bg-sky-950/80 text-sky-400 border-sky-800/60`.
  - This establishes 100% parity with the `BookOpen text-sky-400` icon used for specifications throughout the workspace tree, tab bar, and tag badges.
- **ADR Icon Alignment Across Shell Views**:
  - Update `ShellLayout.tsx` and `TabBar.tsx` to render `<Scale className="... text-amber-400" />` instead of `FileText`, matching `OmniSearchModal.tsx` and semantically representing architecture decision balancing.

---

## 2. Task Graph (DAG)

### Phase 1: Header Deduplication & Dashboard Layout Stabilization

- **TASK-CORE-010-1**: Deduplicate OmniSearch Trigger in Shell Header
  - **Status**: Pending
  - **Addresses**: REQ-001
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Remove the secondary `sidebar-omnisearch-btn` next to `v0.1.1` in `ShellLayout.tsx`.
    - Verify that the primary `header-omnisearch-btn` next to `ProjectSwitcher` in `App.tsx` remains the sole shortcut trigger.
    - Update `ShellLayoutSearch.test.tsx` to remove assertions on the deleted sidebar button.
  - **Verification Seam**: `frontend/tests/ShellLayoutSearch.test.tsx`

- **TASK-CORE-010-2**: Stabilize Dashboard Project Epics Action Button Layout
  - **Status**: Pending
  - **Addresses**: REQ-002
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Update `DashboardView.tsx` so that the "Open Epic" action button has `shrink-0 whitespace-nowrap`.
    - Provide `min-w-0 flex-1 mr-4` to the title/badge container so long epic titles do not compress the action button into two lines.
    - Write/update seam test in `frontend/tests/DashboardView.test.tsx` verifying the classes and layout resilience with long titles.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx`

### Phase 2: Design Token & Semantic Icon Alignment

- **TASK-CORE-010-3**: Unify Feature Design Tokens Across Tree View, Tab Bar, and Epic Hub
  - **Status**: Pending
  - **Addresses**: REQ-003
  - **Blocked by**: TASK-CORE-010-1
  - **What it delivers**:
    - In `ShellLayout.tsx`, change Features section header icon to `<Package className="w-4 h-4 text-purple-400" />` and standalone badge to purple.
    - In `TabBar.tsx`, update `getIcon('feature')` to return `<Package className={clsx(iconClass, 'text-purple-400')} />`.
    - In `EpicHubView.tsx`, update features header icon to `<Package className="w-4 h-4 text-purple-400 flex-shrink-0" />`.
    - Verify OmniSearch already uses `text-purple-400`.
    - Seam test in `frontend/tests/TabBar.test.tsx` and `frontend/tests/ShellLayoutSearch.test.tsx`.
  - **Verification Seam**: `frontend/tests/TabBar.test.tsx`, `frontend/tests/ShellLayoutSearch.test.tsx`

- **TASK-CORE-010-4**: Align Repositories Specs Badge Color and ADR Scale Icon
  - **Status**: Pending
  - **Addresses**: REQ-004, REQ-005
  - **Blocked by**: TASK-CORE-010-2
  - **What it delivers**:
    - In `GitRepositoriesCard.tsx`, update `specs` badge color to `bg-sky-950/80 text-sky-400 border-sky-800/60`.
    - In `ShellLayout.tsx` and `TabBar.tsx`, update ADR icon from `FileText` to `Scale` with `text-amber-400`.
    - Seam test in `frontend/tests/DashboardView.test.tsx`, `frontend/tests/TabBar.test.tsx`, and `frontend/tests/OmniSearchModal.test.tsx`.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx`, `frontend/tests/TabBar.test.tsx`, `frontend/tests/OmniSearchModal.test.tsx`

### Phase 3: Verification & Blueprint Compliance

- **TASK-CORE-010-5**: Full Suite Verification and Build
  - **Status**: Pending
  - **Addresses**: REQ-001 through REQ-005
  - **Blocked by**: TASK-CORE-010-3, TASK-CORE-010-4
  - **What it delivers**:
    - Run Vitest suite across frontend.
    - Run `npm run build` production asset generation.
    - Run `verify-artifacts.mjs` ensuring 0 errors and 0 warnings across all artifacts.
  - **Verification Seam**: Vitest + `verify-artifacts.mjs`
