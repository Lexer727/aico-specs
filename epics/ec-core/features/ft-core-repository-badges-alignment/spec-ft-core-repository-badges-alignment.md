---
id: spec-ft-core-repository-badges-alignment
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 8
title: "Feature and Epic Repository Header Alignment & Badge Layout"
status: "Completed"
derives_from: ["brain-ec-core"]
created_at: "2026-10-07T01:07:00Z"
---

# Feature Specification: Feature and Epic Repository Header Alignment & Badge Layout

## 1. Problem Statement & User Value

In the WebUI, the Feature Viewer currently renders quick-jump spec and plan badge links directly beneath the title heading, which clutters the visual hierarchy since the specification and plan are already primary viewer tabs. Additionally, the Epic Hub rendered repositories in the top metadata pill row while the Feature Viewer rendered them under the title, creating an inconsistent user experience.

This feature aligns both views: removing the spec and plan badge links from beneath the feature title, placing the target repository badges directly beneath the title heading on both Feature Viewer and Epic Hub, and using a clean `FolderGit2` icon without a redundant textual "Repositories:" prefix.

---

## In-Scope
- Removing spec and plan badge tags from beneath the title in `FeatureViewer.tsx`.
- Keeping target repository badge pills beneath the title in `FeatureViewer.tsx`, preceded by `FolderGit2` icon without textual "Repositories:" prefix.
- Moving repository badge pills from the metadata pill row to directly beneath the title in `EpicHubView.tsx`.
- Ensuring both views use an identical, unified layout: `<FolderGit2 ... /> [repo1] [repo2]`.
- Updating test suites in `frontend/tests/`.

---

## Out-of-Scope
- Changing tab navigation or document viewer functionality.
- Altering the dashboard repositories card layout.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer viewing a feature, I want the area below the title to cleanly show target repositories rather than duplicate spec/plan links, so that visual clutter is eliminated.
   - *Independent Test*: Inspect Feature Viewer header and confirm spec/plan badge links are removed and repository badges remain visible.
2. **US-02** (Priority: P1): As a developer viewing an epic, I want the repositories to appear directly below the title just like on the feature view, so that both views have a unified, familiar design.
   - *Independent Test*: Inspect Epic Hub header and confirm repository badges are rendered beneath the title heading with the `FolderGit2` icon.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: The Feature Viewer (`FeatureViewer.tsx`) shall omit spec and plan jump badge links directly beneath the title heading.
- **REQ-002**: The Feature Viewer shall render target repository badge pills directly beneath the title heading, preceded by the `FolderGit2` icon.
- **REQ-003**: The Epic Hub (`EpicHubView.tsx`) shall render aggregated repository badge pills directly beneath the title heading, matching the Feature Viewer layout.
- **REQ-004**: The Epic Hub and Feature Viewer shall render the `FolderGit2` icon without the textual label prefix "Repositories:".

---

## 4. Designated Public Test Seams

- **Feature Viewer Seam**: `frontend/tests/FeatureViewerRepositories.test.tsx`
- **Epic Hub Seam**: `frontend/tests/EpicHubRepositories.test.tsx`
