---
id: spec-ft-core-assets-manager
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 11
title: "Multi-Modal Asset Management, Previews & OS Integration"
status: "Completed"
derives_from: ["ADR-0005", "ADR-0007"]
created_at: "2026-10-16T00:05:00Z"
---

# Feature Specification: Multi-Modal Asset Management, Previews & OS Integration

## 1. Problem Statement & User Value

Working with engineering specifications and multi-agent code orchestration requires non-code contextual assets: UI mockups, brand guidelines, audio cues, test data schemas, and architecture sketches. In the current AICO WebUI shell:
1. **Drop Zone Lacks File Picker**: The `FileDropZone` handles drag-and-drop and clipboard paste, but clicking the drop zone does not open the native file browser dialog.
2. **Missing Persistence & Scoped Storage**: The Assets panel in `EpicHubView` only stored temporary file names in a local React state array without disk persistence. `FeatureViewer` lacked file upload and asset display entirely.
3. **Absence of Asset Previews & Navigation**: Developers cannot preview images or inspect text snippets directly in the assets panel, cannot double-click an asset to inspect it in a dedicated tab, and cannot open an asset in the host operating system's registered application.
4. **No Hierarchical Aggregation in Epics**: Epics must provide complete visibility into all assets across their child features with distinct origin markings, while individual features must only show assets scoped strictly to themselves.
5. **Context Menu Viewport Clipping**: When right-clicking an asset near the bottom or right boundaries of the screen, the context menu opens partially outside the visible viewport.
6. **No Asset Counter in Folded Accordions**: Accordion handles display a generic "Assets" label with no indication of whether assets are present without expanding the panel.
7. **Cross-Tab Accordion State Pollution**: Expanding or collapsing accordion panels in one feature tab leaks into other feature tabs and resets when navigating to epics.
8. **Unrestricted File Type Uploads**: The Assets panel currently allows uploading arbitrary binary files rather than restricting uploads to images and text-readable documents/code (`.txt`, `.md`, `.html`, `.py`, `.ipynb`, and source files) managed via a simple extensible configuration file.

This specification establishes end-to-end asset ingestion, scope-aware storage, responsive gallery previews with hover zoom/snippets, tab-based asset viewing, host operating system integration, smart viewport positioning, non-zero counter badges, entity-scoped accordion state persistence, and extensible config-driven file type filtering (`aico.config.json`).

---

## In-Scope

- **Native File Browser Trigger**: Clicking anywhere in the `FileDropZone` opens the native operating system file picker (`<input type="file" multiple />`).
- **Scope-Aware Filesystem Persistence**:
  - Epic assets: `<specs_root>/epics/<ec-id>/assets/<filename>`
  - Epic child feature assets: `<specs_root>/epics/<ec-id>/features/<ft-id>/assets/<filename>`
  - Standalone feature assets: `<specs_root>/features/<ft-id>/assets/<filename>`
- **Scope-Aware Asset Discovery**:
  - Feature view lists only assets belonging to that feature.
  - Epic view lists direct epic assets tagged with `[epic]` and sub-feature assets tagged with their child feature ID (e.g. `[ft-xxx]`).
- **Asset Gallery & Interactive Previews**:
  - Image thumbnail grid with lazy loading and enlarged hover popovers.
  - Document/code icons with hover text snippet previews (~200 chars).
- **Tab Navigation & Asset Viewer**:
  - Double-clicking an asset opens a dedicated tab (`type: 'asset'`).
  - Dedicated `AssetViewer` component rendering image viewports, markdown/text viewers, or audio players.
- **Host OS Launch ("Open on system")**:
  - Right-click context menu on asset items with an "Open on system" action.
  - "Open on system" toolbar button in the `AssetViewer` tab.
  - Backend execution via host shell handlers (`rundll32 url.dll,FileProtocolHandler` on Windows, `open`/`xdg-open` on Unix).
- **Safe File Deletion with Confirmation**:
  - Right-click "Delete" option in `AssetPanel` and toolbar "Delete" button in `AssetViewer`.
  - Mandatory confirmation modal before deleting from disk.
  - Automated tab closure upon asset deletion.
- **Smart Viewport Context Menu Positioning**:
  - Clamping context menu coordinates to window dimensions so the menu never renders outside the viewport.
- **Non-Zero Asset Counter Badges**:
  - Displays `(x)` count badge in `EpicHubView` and `FeatureViewer` accordion headers when assets > 0.
- **Entity-Scoped Accordion Persistence**:
  - Preserves panel expansion states individually per entity ID (`epicId`, `featureId`) across tab switches.
- **Config-Driven File Type Filtering (`aico.config.json`)**:
  - Extensible JSON configuration file (`aico.config.json`) defining `assets.allowed_image_extensions`, `assets.allowed_text_extensions`, and `assets.allow_utf8_text_fallback`.
  - Restricts asset uploads to configured image and text/code file types (including `.txt`, `.md`, `.html`, `.py`, `.ipynb`, and any UTF-8 readable text/source files) while rejecting unsupported binary files.

---

## Out-of-Scope (Future Upgrades)

- In-browser vector graphic vectorization or video re-encoding.
- Cloud cloud-storage bucket syncing (AICO operates on local-first disk files).
- Binary diffing between multiple asset revisions.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want to drag files into or click the Assets panel to upload design mockups or specifications, so that they are saved directly to the appropriate epic or feature assets folder on disk.
2. **US-02** (Priority: P1): As a developer viewing an epic, I want to see both epic-level assets and all sub-feature assets labeled with their origin, so that I have a holistic view of all design context in one place.
3. **US-03** (Priority: P1): As a developer viewing a feature, I want to see only assets specific to that feature, so that feature context remains focused and uncluttered.
4. **US-04** (Priority: P2): As a developer hovering over an asset, I want an enlarged preview for images or a snippet for text files, so that I can quickly inspect contents without opening a new view.
5. **US-05** (Priority: P1): As a developer, I want to double-click an asset to open it in an AICO tab, and right-click or use the tab toolbar button to open the file in my system's default viewer.
6. **US-06** (Priority: P1): As a developer, I want to delete unwanted assets from the right-click menu or the asset tab, but only after explicit confirmation, so that I don't accidentally remove critical specification media.
7. **US-07** (Priority: P2): As a developer right-clicking an asset near the bottom or right of the screen, I want the context menu to automatically adjust its position so that it is never clipped or hidden outside the viewport.
8. **US-08** (Priority: P2): As a developer viewing an epic or feature, I want to see an asset count badge (e.g. `(3)`) in the Assets accordion header whenever assets exist, so that I immediately know if assets are available without expanding the panel.
9. **US-09** (Priority: P2): As a developer navigating across different feature and epic tabs, I want each tab to remember which panels I expanded or collapsed, so that my view state is preserved when switching tabs.
10. **US-10** (Priority: P1): As a developer, I want asset uploads restricted to images and text/code files (`.txt`, `.md`, `.html`, `.py`, `.ipynb`, and any text-readable source files) managed via a simple extensible config file (`aico.config.json`), so that unsupported binary files cannot pollute the specs repository.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `FileDropZone` shall support both file drag-and-drop and mouse click interactions, triggering the native operating system file picker on click.
- **REQ-002**: The server shall store uploaded files under `<specs_root>/epics/<ec-id>/assets/` for epics, `<specs_root>/epics/<ec-id>/features/<ft-id>/assets/` for child features, or `<specs_root>/features/<ft-id>/assets/` for standalone features.
- **REQ-003**: The server shall reject any asset file path containing directory traversal sequences (`..`) with a `400 Bad Request` status.
- **REQ-013**: The system shall load extensible workspace configuration from `aico.config.json` (defining `assets.allowed_image_extensions`, `assets.allowed_text_extensions`, and `assets.allow_utf8_text_fallback` with built-in defaults) and expose the active configuration via `GET /api/v1/config`.

### Event-Driven Requirements
- **REQ-004**: When the client requests assets for an epic, the server shall return all assets residing in the epic's assets folder marked with `source_label: "epic"` and all assets residing in any child feature's assets folder marked with `source_label: "<ft-id>"`.
- **REQ-005**: When the client requests assets for a feature, the server shall return only assets residing in that specific feature's assets folder.
- **REQ-006**: When hovering over an image asset card, the UI shall render an enlarged image preview popover; when hovering over a text or markdown asset card, the UI shall render a text snippet preview popover.
- **REQ-007**: When the user double-clicks an asset item in the gallery, the `TabBarManager` shall open a dedicated tab (`type: 'asset'`) rendering `AssetViewer`.
- **REQ-008**: When the user clicks "Open on system" from an asset's right-click context menu or the `AssetViewer` toolbar, the server shall invoke the host operating system's registered application to open the file.
- **REQ-009**: When the user initiates asset deletion via the context menu in `AssetPanel` or the toolbar in `AssetViewer`, the WebUI shall display an explicit confirmation dialog before dispatching deletion; upon confirmation, the server shall remove the file from disk, return success, and the WebUI shall refresh the asset gallery and close any open tab displaying the deleted asset.
- **REQ-010**: When the user opens the asset context menu, the WebUI shall calculate its bounding coordinates against the window viewport dimensions; if the cursor is near the bottom or right boundaries, the menu shall reposition itself (e.g. shift upwards and/or leftwards) so that it remains fully visible within the viewport.
- **REQ-011**: When an epic or feature possesses one or more attached assets, the accordion header shall render an asset count badge `(x)` next to the Assets title; when the asset count is zero, the badge shall be omitted.
- **REQ-012**: When navigating between tabs or toggling accordion sections, the WebUI shall persist the expanded and collapsed states of each section keyed by the specific entity identifier (`epicId` or `featureId`), ensuring that panel state changes in one entity do not mutate other entities and are preserved across tab switches.
- **REQ-014**: When a file upload is initiated via `FileDropZone` or `POST /api/v1/assets/upload`, the system shall permit the upload only if the file's extension matches a configured image or text/code extension in `aico.config.json` or (when `allow_utf8_text_fallback` is enabled) the file content is valid UTF-8 text without binary null bytes; otherwise, the system shall reject the upload with a descriptive error message.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Backend Asset API Seam (`crates/aico-server/tests/assets_test.rs`)**:
   - `GET /api/v1/config`: Verifies retrieval of `aico.config.json` asset upload policies.
   - `GET /api/v1/epics/:id/assets`: Verifies aggregate listing of direct epic assets and sub-feature assets with `source_label` and `is_subfeature`.
   - `GET /api/v1/features/:id/assets`: Verifies isolated single-feature listing.
   - `POST /api/v1/assets/upload`: Verifies physical persistence for allowed image and text/code files (including `.ipynb`, `.py`, and UTF-8 readable text) and rejection (`400 Bad Request`) of disallowed binary files.
   - `POST /api/v1/assets/open-system`: Verifies host system launch handler and path traversal safety.
   - `DELETE /api/v1/assets`: Verifies physical deletion from disk and path traversal protection.
2. **DropZone Component Seam (`frontend/tests/FileDropZone.test.tsx`)**:
   - Signature: `<FileDropZone onFileUpload={...} />`
   - Verifies click-to-pick dialog trigger, drag-drop emission, and custom/configured file-type validation.
3. **Asset Panel Component Seam (`frontend/tests/AssetPanel.test.tsx`)**:
   - Signature: `<AssetPanel scopeType="epic"|"feature" scopeId="..." />`
   - Verifies thumbnail grid rendering, origin badges in epics, hover preview popovers, double-click tab dispatch, right-click context menu, asset deletion confirmation flow, viewport edge clamping, and rejection of disallowed binary uploads.
4. **Asset Viewer Component Seam (`frontend/tests/AssetViewer.test.tsx`)**:
   - Signature: `<AssetViewer assetPath="..." filename="..." onOpenSystem={...} />`
   - Verifies content rendering (including `.ipynb` and code files), "Open on system" toolbar button dispatch, and toolbar delete confirmation flow.
5. **Accordion Navigation Seam (`frontend/tests/EpicHubView.test.tsx`, `frontend/tests/FeatureViewer.test.tsx`)**:
   - Verifies non-zero asset count badge `(x)` in accordion handles and entity-scoped accordion state preservation across tab switches.

---

## Out-of-Scope

- In-browser vector graphic vectorization or video re-encoding.
- Cloud storage synchronization.
- Multi-user concurrent asset locking.

---

## 6. Open Clarifications

None. Ratified in [ADR-0005](../../../../adr/0005-unified-accordion-workspace-and-dockable-panels.md) and [ADR-0007](../../../../adr/0007-multi-modal-assets-and-human-gate-tasks.md).
