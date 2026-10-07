---
id: plan-ft-core-repository-badges-alignment
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 8
title: "Feature and Epic Repository Header Alignment & Badge Layout Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-repository-badges-alignment"]
created_at: "2026-10-07T01:08:00Z"
---

# Implementation Plan: Feature and Epic Repository Header Alignment & Badge Layout

## 1. Technical Architecture & Decisions

- **Consistent Header Hierarchy**:
  - In `FeatureViewer.tsx`, remove `spec-badge-link` and `plan-badge-link`. Under `<h1 className="text-2xl font-bold ...">{feature.name}</h1>`, render `<FolderGit2 ... /> [repo1] [repo2]`.
  - In `EpicHubView.tsx`, remove the repositories list from the metadata pill row. Under `<h1 className="text-2xl font-bold ...">{epic.title}</h1>`, render `<FolderGit2 ... /> [repo1] [repo2]`.
- **Text Label Elimination**: Neither view will render textual "Repositories:". Only the `FolderGit2` icon (with `title="Repositories"`) followed by the repository badge pills.
- **Frontend Test Suite Updates**: Update `FeatureViewer.test.tsx`, `FeatureViewerRepositories.test.tsx`, and `EpicHubRepositories.test.tsx`.

---

## 2. Task Graph (DAG)

### Phase 1: Feature Viewer Header Alignment

- **TASK-CORE-008-1**: Align Feature Viewer Header & Repository Badges Layout
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-004
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Remove spec and plan tag links under feature title in `FeatureViewer.tsx`.
    - Render `FolderGit2` icon and repository badge pills under title without textual prefix.
  - **Verification Seam**: `frontend/tests/FeatureViewerRepositories.test.tsx`

### Phase 2: Epic Hub Header Alignment

- **TASK-CORE-008-2**: Align Epic Hub Header & Move Repository Badges Under Title
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004
  - **Blocked by**: TASK-CORE-008-1
  - **What it delivers**:
    - Move repository badges in `EpicHubView.tsx` from top pill row to directly under the title.
    - Omit "Repositories:" text and render `FolderGit2` icon matching Feature Viewer.
  - **Verification Seam**: `frontend/tests/EpicHubRepositories.test.tsx`
