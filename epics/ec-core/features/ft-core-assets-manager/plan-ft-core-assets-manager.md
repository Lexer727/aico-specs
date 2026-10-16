---
id: plan-ft-core-assets-manager
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 11
title: "Multi-Modal Asset Management, Previews & OS Integration Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-assets-manager"]
created_at: "2026-10-16T00:05:00Z"
---

# Implementation Plan: Multi-Modal Asset Management, Previews & OS Integration

## 1. Technical Architecture & Decisions

- **Storage Structure**:
  - Epic assets: `<specs_root>/epics/<ec-id>/assets/<filename>`
  - Epic child feature assets: `<specs_root>/epics/<ec-id>/features/<ft-id>/assets/<filename>`
  - Standalone feature assets: `<specs_root>/features/<ft-id>/assets/<filename>`
- **Backend Architecture (`crates/aico-server`)**:
  - Axum handlers for `GET /api/v1/epics/:id/assets` and `GET /api/v1/features/:id/assets` returning list of `AssetInfo`.
  - Multipart upload handler `POST /api/v1/assets/upload` writing directly to target assets folder on disk.
  - Raw stream handler `GET /api/v1/assets/raw?path=...` serving binary content with MIME types.
  - System launcher `POST /api/v1/assets/open-system` with strict path validation and Windows `rundll32 url.dll,FileProtocolHandler` execution.
- **Frontend Architecture (`frontend/src`)**:
  - `FileDropZone.tsx`: Adds hidden `<input type="file" multiple />` and click-to-pick trigger.
  - `AssetPanel.tsx`: Reusable gallery rendering thumbnail cards, origin badges (`[epic]` vs `[ft-id]`), and hover popovers (zoom for images, text snippet for documents).
  - `AssetViewer.tsx`: Full-tab component rendering image viewport, markdown/text viewer, or audio player, with "Open on system" toolbar button.
  - `useTabStore.ts`: Extends `TabItem.type` with `'asset'`.
  - Context menu: Right-click on asset card to "Open on system" or "Open in tab".
- **Deletion Architecture**:
  - Backend handler `DELETE /api/v1/assets` accepting `{ "path": string }`, validating path within specs root without traversal, and removing file with `std::fs::remove_file`.
  - Frontend confirmation modal/dialog in `AssetPanel` (for context menu delete) and `AssetViewer` (for toolbar button delete).
  - Tab closure via `useTabStore` when an active tab's asset is deleted.
- **Context Menu Viewport Clamping**:
  - `AssetPanel.tsx` calculates context menu coordinates against `window.innerWidth` and `window.innerHeight`, shifting left or up when clicking within boundary thresholds.
- **Non-Zero Asset Count Badges**:
  - `EpicHubView.tsx` and `FeatureViewer.tsx` display an inline count badge `(x)` next to "Assets" when assets > 0, omitting the badge when count is 0.
- **Entity-Scoped Accordion Persistence**:
  - Zustand store `useAccordionStore.ts` tracking collapsed/expanded panel states keyed by entity ID (`[entityId][section]`), preserving view state across tab navigation and entity switches.
- **Config-Driven File Type Filtering (`aico.config.json`)**:
  - Extensible workspace config file `aico.config.json` defining `assets.allowed_image_extensions`, `assets.allowed_text_extensions`, and `assets.allow_utf8_text_fallback`.
  - Backend `GET /api/v1/config` serves the active configuration; `POST /api/v1/assets/upload` and `FileDropZone.tsx` validate file extensions and UTF-8 text readability before accepting uploads.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

- **TASK-ASSET-001**: Implement backend asset discovery, raw content streaming, and multipart upload endpoints
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-004, REQ-005
  - **Blocked by**: none
  - **Verification Seam**: `crates/aico-server/tests/assets_test.rs`
  - **What it delivers**:
    - Adds `GET /api/v1/epics/:id/assets` returning direct epic assets and sub-feature assets tagged with origin.
    - Adds `GET /api/v1/features/:id/assets` returning isolated feature assets.
    - Adds `POST /api/v1/assets/upload` and `GET /api/v1/assets/raw`.

- **TASK-ASSET-002**: Implement host system launch endpoint with path traversal protection
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-008
  - **Blocked by**: TASK-ASSET-001
  - **Verification Seam**: `crates/aico-server/tests/assets_test.rs`
  - **What it delivers**:
    - Adds `POST /api/v1/assets/open-system` validating that paths reside safely within the active project directory.
    - Dispatches default registered application on host OS (`rundll32` on Windows, `open`/`xdg-open` on Unix).

- **TASK-ASSET-003**: Enhance FileDropZone with native file browser click trigger
  - **Status**: Done
  - **Addresses**: REQ-001
  - **Blocked by**: none
  - **Verification Seam**: `frontend/tests/FileDropZone.test.tsx`
  - **What it delivers**:
    - Embeds a hidden file input element in `FileDropZone.tsx`.
    - Opens system file picker dialog when user clicks anywhere in the drop zone.
    - Emits selected files via `onFileUpload`.

- **TASK-ASSET-004**: Build AssetPanel gallery component with scope badges and hover previews
  - **Status**: Done
  - **Addresses**: REQ-004, REQ-005, REQ-006
  - **Blocked by**: TASK-ASSET-001, TASK-ASSET-003
  - **Verification Seam**: `frontend/tests/AssetPanel.test.tsx`
  - **What it delivers**:
    - Creates `frontend/src/features/assets/AssetPanel.tsx` integrated into `EpicHubView` and `FeatureViewer`.
    - Renders image cards with hover zoom popovers and text/doc cards with hover snippet popovers.
    - Displays origin badges (`[epic]` vs `[ft-id]`) in epic views.

- **TASK-ASSET-005**: Add asset tab type and build AssetViewer component with system launcher button
  - **Status**: Done
  - **Addresses**: REQ-007, REQ-008
  - **Blocked by**: TASK-ASSET-002, TASK-ASSET-004
  - **Verification Seam**: `frontend/tests/AssetViewer.test.tsx`
  - **What it delivers**:
    - Adds `'asset'` type to `TabItem` in `useTabStore.ts`.
    - Implements `frontend/src/features/assets/AssetViewer.tsx` in `App.tsx`.
    - Supports double-clicking asset items to open tabs and provides "Open on system" button.

- **TASK-ASSET-006**: Add right-click context menu and run end-to-end unit tests
  - **Status**: Done
  - **Addresses**: REQ-006, REQ-008
  - **Blocked by**: TASK-ASSET-005
  - **Verification Seam**: `frontend/tests/AssetPanel.test.tsx`
  - **What it delivers**:
    - Adds right-click context menu to asset items with "Open on system" and "Open in tab" actions.
    - Runs frontend Vitest and backend Rust integration test suites.

- **TASK-ASSET-007**: Implement asset deletion backend endpoint and WebUI confirmation workflows
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-009
  - **Blocked by**: TASK-ASSET-006
  - **Verification Seam**: `crates/aico-server/tests/assets_test.rs`, `frontend/tests/AssetPanel.test.tsx`, `frontend/tests/AssetViewer.test.tsx`
  - **What it delivers**:
    - Adds `DELETE /api/v1/assets` route and handler in `crates/aico-server`.
    - Adds integration test in `crates/aico-server/tests/assets_test.rs`.
    - Adds deletion confirmation dialog and context menu trigger in `AssetPanel.tsx`.
    - Adds deletion confirmation dialog and toolbar button in `AssetViewer.tsx`.
    - Integrates automatic tab closing on asset deletion.

- **TASK-ASSET-008**: Implement context menu boundary clamping, asset count badges, and entity-scoped accordion state persistence
  - **Status**: Done
  - **Addresses**: REQ-010, REQ-011, REQ-012
  - **Blocked by**: TASK-ASSET-007
  - **Verification Seam**: `frontend/tests/AssetPanel.test.tsx`, `frontend/tests/EpicHubView.test.tsx`, `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Adds viewport boundary calculation to context menu opening in `AssetPanel.tsx`.
    - Adds non-zero asset count badge to accordion headers in `EpicHubView.tsx` and `FeatureViewer.tsx`.
    - Creates `useAccordionStore.ts` and connects accordion toggling to entity IDs.
    - Adds test coverage for context menu viewport clamping and tab-scoped accordion memory.

- **TASK-ASSET-009**: Implement extensible `aico.config.json`, `GET /api/v1/config`, and image/text/code asset upload validation
  - **Status**: Done
  - **Addresses**: REQ-013, REQ-014
  - **Blocked by**: TASK-ASSET-008
  - **Verification Seam**: `crates/aico-server/tests/assets_test.rs`, `frontend/tests/FileDropZone.test.tsx`, `frontend/tests/AssetPanel.test.tsx`
  - **What it delivers**:
    - Creates `aico.config.json` with default allowed image and text/code extensions and UTF-8 readable text fallback.
    - Adds `GET /api/v1/config` route and enforces file-type / UTF-8 text validation in `POST /api/v1/assets/upload`.
    - Enhances `FileDropZone.tsx`, `AssetPanel.tsx`, and `AssetViewer.tsx` to validate uploads against `aico.config.json` and render text previews for `.ipynb`, `.py`, and all text/code files.
    - Adds backend and frontend test coverage verifying allowed text/code/image uploads and rejection of disallowed binary uploads.

---

## 3. Amendment History

- **2026-10-16T00:05:00Z**: Initial creation of asset management specification and vertical task plan under epic `ec-core`.
- **2026-10-16T00:27:00Z**: Added TASK-ASSET-007 and REQ-009 for asset deletion with mandatory confirmation dialogs.
- **2026-10-16T00:51:00Z**: Added TASK-ASSET-008 and REQ-010..012 for context menu viewport clamping, asset count badges, and entity-scoped accordion persistence.
- **2026-10-16T01:12:00Z**: Added TASK-ASSET-009 and REQ-013..014 for extensible `aico.config.json` and image/text/code asset upload filtering.
