---
id: plan-ft-core-dashboard-and-repo-card-refinement
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 18
title: "Dashboard Workspace Refinement, Re-unified Repositories Grid & Workflow Automation Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-dashboard-and-repo-card-refinement"]
created_at: "2026-10-24T01:20:00.000Z"
---

# Implementation Plan: Dashboard Workspace Refinement, Re-unified Repositories Grid & Workflow Automation

## 1. Technical Architecture & Decisions

- **Zero-Mock Initialization**: Remove static `mockProjects` in `App.tsx` and `defaultEpics` in `DashboardView.tsx`. Replace fallback mock counts with clean `0` and empty indicators to prevent misleading data flashes during page reload.
- **Re-unified Repositories Layout**: Remove the isolated "Specs Repository" section from `GitRepositoriesCard.tsx` that left 2 empty grid columns. Merge all repositories into a single responsive 3-column grid (`grid-cols-1 md:grid-cols-3 gap-3`), rendering the Specs repo (`type: 'specs'`) first, followed by codebase and skills repos.
- **State-Aware Git Buttons**:
  - Pull: Disabled when `behind === 0`; enabled with `Pull (${repo.behind})` badge when `behind > 0`.
  - Push: Disabled when `ahead === 0`; enabled with `Push (${repo.ahead})` badge when `ahead > 0`.
  - Commit: Disabled when `repo.is_clean` is true; enabled with `Commit (${modifiedCount})` when modified/untracked files exist.
- **One-Click AI Commit Action**:
  - Expose `POST /api/v1/git/ai-commit` in `crates/aico-server` which examines git diff / status, formulates a conventional commit message (e.g. `feat(<scope>): ...`), and executes the commit.
  - In `GitRepositoriesCard.tsx`, clicking `Commit` directly triggers this AI commit endpoint with spinner feedback without manual text input.
- **Tag Standardization & Spec Pill Removal**: Replace the bulky `Open Epic` button with the interactive `<TagBadge id={epic.id} onClick={() => onOpenEpic(epic.id)} />` in `DashboardView.tsx`, and remove the spec `<TagBadge>` pills from the bottom of Project Epics cards to eliminate visual bloat.
- **Workflow Automation in Skills**: Update `aico-implementation`, `aico-review-loop`, and `aico-git` skill definitions to codify automatic staging, commit, and push across decoupled repositories upon successful task verification.
- **Codebase Repository Resolution Precedence**: In `crates/aico-server/src/server.rs` (`resolve_repo_dir`), inspect `proj.manifest.get_effective_repositories(&proj.specs_path)` first so that repositories sharing the project slug (e.g. codebase repo `aico` inside project `aico`) resolve to their actual codebase path (`codebase/aico`) rather than defaulting to `proj.specs_path` (`aico-specs`). In `GitRepositoriesCard.tsx`, include `path: repo.path` in action request payloads.
- **Workspace-Wide Lineage Graph on Home Dashboard**: Render `VisualLineageCanvas` (`epicId={undefined}`) on `DashboardView.tsx` displaying all project Epics and their associated nodes (`ADR -> Epic -> Feature -> Task -> Verification Seam / Code`), with a full-window pop-out button wired in `App.tsx`.
- **Universal Inline & Block Markdown Formatting**: Introduce `<InlineMarkdown content={...} />` in `MarkdownView.tsx` (using `marked.parseInline`) and format all markdown-bearing fields across `DashboardView.tsx`, `EpicHubView.tsx`, `FeatureViewer.tsx`, `SpecViewer.tsx`, `PlanViewer.tsx`, `GlossaryViewer.tsx`, and `AdrViewer.tsx`.

---

## 2. Task Graph (DAG)

```mermaid
flowchart TD
    T1["TASK-CORE-018-1: Eliminate Mock Defaults in App & DashboardView (REQ-001, REQ-007)"]
    T2["TASK-CORE-018-2: Standardize Project Epics & Dashboard Tags with Clickable TagBadge (REQ-006)"]
    T3["TASK-CORE-018-3: Re-unify Git Repositories Grid & State-Aware Action Buttons (REQ-002, REQ-003)"]
    T4["TASK-CORE-018-4: Implement One-Click AI Commit in UI & Backend Endpoint (REQ-004)"]
    T5["TASK-CORE-018-5: Codify Auto Commit & Push Workflow in AICO Skills (REQ-005)"]
    T6["TASK-CORE-018-6: Fix Codebase Repo Resolution Collision & Pass Path Telemetry (REQ-008)"]
    T7["TASK-CORE-018-7: Remove Spec Tags from Dashboard Epics & Add All-Epics Workspace Lineage Graph (REQ-009, REQ-010)"]
    T8["TASK-CORE-018-8: Implement InlineMarkdown & Fix Markdown Formatting Across All Views (REQ-011)"]

    T1 --> T2
    T2 --> T3
    T3 --> T4
    T4 --> T5
    T5 --> T6
    T6 --> T7
    T6 --> T8
```

### Phase 1: Zero-Mock Initialization & Header Cleanup

- **TASK-CORE-018-1**: Eliminate Hardcoded Mock Defaults in App & DashboardView
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-007`
  - **Blocked by**: None
  - **What it delivers**:
    - Removes hardcoded `Root: d:/aico/aico-specs` from `Workspace Overview` header in `DashboardView.tsx`.
    - Removes `mockProjects` in `App.tsx`, initializing `projects` with `[]`.
    - Removes `defaultEpics` from `DashboardView.tsx`, binding `activeEpics` directly to `epics`.
    - Displays clean `0` and empty state indicators instead of flashing fabricated numbers.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx`

---

### Phase 2: Tag Navigation Standardization

- **TASK-CORE-018-2**: Standardize Project Epics & Dashboard Tags with Clickable TagBadge
  - **Status**: Done
  - **Addresses**: `REQ-006`
  - **Blocked by**: `TASK-CORE-018-1`
  - **What it delivers**:
    - Replaces the `Open Epic` button and static epic ID tag in `DashboardView.tsx` with `<TagBadge id={epic.id} onClick={() => onOpenEpic(epic.id)} />`.
    - Ensures tags have `cursor-pointer`, hover styling, and tab-opening navigation.
    - Updates test suites to reflect `TagBadge` epic interactions.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx`

---

### Phase 3: Repositories Grid Re-unification & State Telemetry

- **TASK-CORE-018-3**: Re-unify Git Repositories Grid & State-Aware Action Buttons
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`
  - **Blocked by**: `TASK-CORE-018-2`
  - **What it delivers**:
    - Re-unifies `specs` and `codebase` repositories in `GitRepositoriesCard.tsx` into a single 3-column responsive grid with Specs repo first.
    - Eliminates empty column gaps.
    - Implements smart disabled states and count badges:
      - Pull disabled on `behind === 0` with tooltip; enabled on `behind > 0` with `Pull (${behind})`.
      - Push disabled on `ahead === 0` with tooltip; enabled on `ahead > 0` with `Push (${ahead})`.
      - Commit disabled on clean working tree with tooltip; enabled on modified tree with `Commit (${count})`.
  - **Verification Seam**: `frontend/tests/DashboardGitStatus.test.tsx` and `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`

---

### Phase 4: One-Click AI Commit

- **TASK-CORE-018-4**: Implement One-Click AI Commit in UI & Backend Endpoint
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-CORE-018-3`
  - **What it delivers**:
    - Implements `POST /api/v1/git/ai-commit` in `crates/aico-server` generating conventional commit message from git status/diff and committing.
    - Connects `Commit` button in `GitRepositoriesCard.tsx` to directly trigger AI commit with spinner feedback without manual text input dialog.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs` and `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`

---

### Phase 5: Workflow Automation in AICO Skills

- **TASK-CORE-018-5**: Codify Auto Commit & Push Workflow in AICO Skills
  - **Status**: Done
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-CORE-018-4`
  - **What it delivers**:
    - Updates `aico-implementation/SKILL.md`, `aico-review-loop/SKILL.md`, and `aico-git/SKILL.md` (and mirrored in `.agents/skills`).
    - Mandates that upon turning red tests green and verifying artifacts, the autonomous agent automatically stages, commits, and pushes across affected repositories (`aico` and `aico-specs`).
  - **Verification Seam**: `verify-artifacts.mjs`

---

### Phase 6: Codebase Repository Resolution & Submodule Commit Fix

- **TASK-CORE-018-6**: Fix Codebase Repo Resolution Collision & Pass Path Telemetry
  - **Status**: Done
  - **Addresses**: `REQ-008`
  - **Blocked by**: `TASK-CORE-018-5`
  - **What it delivers**:
    - Updates `resolve_repo_dir` in `crates/aico-server/src/server.rs` to query effective repositories first, preventing codebase repositories whose id equals `proj.manifest.name` from colliding with specs directory.
    - Updates `GitRepositoriesCard.tsx` to include `path: repo.path` in `handlePull`, `handlePush`, and `handleAiCommit` requests.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs` and `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`

---

### Phase 7: Dashboard Spec Tag Removal & Workspace-Wide Lineage Graph

- **TASK-CORE-018-7**: Remove Spec Tags from Dashboard Epics & Add All-Epics Workspace Lineage Graph
  - **Status**: Done
  - **Addresses**: `REQ-009`, `REQ-010`
  - **Blocked by**: `TASK-CORE-018-6`
  - **What it delivers**:
    - Removes the spec `<TagBadge>` pills at the bottom of Project Epics cards in `DashboardView.tsx`.
    - Renders the interactive `VisualLineageCanvas` (`epicId={undefined}`) on `DashboardView.tsx` showing all project Epics and their associated nodes (`ADR -> Epic -> Feature -> Task -> Verification Seam / Code`).
    - Wires `onOpenNode` and `onPopoutGraph` (`handleOpenGraph('workspace')`) in `App.tsx` and `DashboardView.tsx` so users can open any node or pop out the workspace graph to a full-window tab.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx`

---

### Phase 8: Universal Markdown Formatting Across Views

- **TASK-CORE-018-8**: Implement InlineMarkdown & Fix Markdown Formatting Across All Views
  - **Status**: Done
  - **Addresses**: `REQ-011`
  - **Blocked by**: `TASK-CORE-018-6`
  - **What it delivers**:
    - Adds `<InlineMarkdown content={...} />` in `frontend/src/components/common/MarkdownView.tsx` using `marked.parseInline`.
    - Replaces raw string rendering with `<InlineMarkdown>` / `<MarkdownView>` across `DashboardView.tsx`, `EpicHubView.tsx`, `FeatureViewer.tsx`, `SpecViewer.tsx`, `PlanViewer.tsx`, `GlossaryViewer.tsx`, and `AdrViewer.tsx`.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx` and `frontend/tests/EpicHubView.test.tsx`

---

## 3. Amendment Log

- **2026-10-24T01:20:00Z**: Initialized plan for `ft-core-dashboard-and-repo-card-refinement` covering zero-mock defaults, re-unified repositories grid, state-aware git telemetry, one-click AI commit, and skills workflow auto commit/push.
- **2026-10-24T01:45:00Z**: Appended TASK-CORE-018-6 to resolve repo directory collision when codebase repo ID matches project slug, fixing submodule commit dispatch.
- **2026-10-24T18:45:00Z**: Appended TASK-CORE-018-7 (`REQ-009`, `REQ-010`) and TASK-CORE-018-8 (`REQ-011`) to remove spec tags on Dashboard Epic cards, embed the all-epics workspace lineage graph on the Dashboard, and format inline/block Markdown across all views.
