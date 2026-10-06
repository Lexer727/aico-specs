---
id: plan-ft-core-epic-hub
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 7
title: "Epic Hub, Visual Lineage Canvas & Asset Drop Zones"
status: "Completed"
derives_from: ["spec-ft-core-epic-hub"]
created_at: "2026-10-05T20:00:00Z"
---

# Implementation Plan: Epic Hub, Visual Lineage Canvas & Asset Drop Zones


---

## 1. Technical Architecture & Decisions

- **Components**: Backend Asset Service (`crates/aico-storage` or `crates/aico-server`) & Frontend Views (`frontend/src/features/epic-hub`).
- **Core Dependencies**:
  - `sha2`: SHA-256 content hashing for asset deduplication.
  - `@xyflow/react` (React Flow) / SVG graph renderer: Interactive, pannable/zoomable visual lineage DAG canvas.
  - `react-dropzone`: Drag-and-drop file upload handling with MIME type filtering and preview generation.
  - `kuzu`: Tracking `Asset` nodes, `REFERENCES` edges, and `SATISFIES` human gate edges.
- **Asset Storage**: Stored at `<specs-root>/.aico/assets/<sha256>.<ext>` with metadata recorded in Kùzu and indexed into LanceDB.

---

## 2. Task Graph (DAG)

### Phase 1: Local Asset Ingestion & Storage Service (Unblocked Frontier)

- **TASK-CORE-007-1**: Local Asset Ingestion Service & Deduplication
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-CORE-003-1` (Storage In-Process Engine)
  - **What it delivers**:
    - Implements `AssetIngestionService::store_asset(file_bytes, filename, mime_type)`.
    - Computes SHA-256 hash to deduplicate files under `.aico/assets/`.
    - Generates thumbnail previews for images/videos.
    - Creates `Asset` node in Kùzu and indexes text/metadata into LanceDB vector store.
  - **Verification Seam**: `tests/asset_ingestion_test.rs::test_asset_storage_deduplication_and_indexing`

---

### Phase 2: Multi-Modal Drag-and-Drop Zones

- **TASK-CORE-007-2**: UI Drag-and-Drop Zones (Chat, Sidebar, and Task Card)
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-004`, `REQ-005`
  - **Blocked by**: `TASK-CORE-006-1` (UI Shell), `TASK-CORE-007-1`
  - **What it delivers**:
    - Implements reusable `FileDropZone` supporting file drag-and-drop and clipboard paste (`Ctrl+V`).
    - Integrates drop zone into Chat input box for quick multimodal agent prompting.
    - Integrates drop zone into Project/Epic sidebar for reference asset management.
    - Integrates drop target directly onto Human Gate Task cards.
  - **Verification Seam**: `frontend/tests/FileDropZone.test.tsx::accepts_files_and_emits_upload_event`

---

### Phase 3: Human Gate Task Resolution Engine

- **TASK-CORE-007-3**: Human Gate Task Engine & Blocker Resolution
  - **Status**: Done
  - **Addresses**: `REQ-009`
  - **Blocked by**: `TASK-CORE-007-2`, `TASK-CORE-005-4` (Traceability Graph)
  - **What it delivers**:
    - Implements `HumanGateManager` recognizing tasks with `role: "user"` and `type: "asset_delivery"`.
    - Keeps dependent agent tasks in `blocked` state in the Plan DAG until the required asset is attached.
    - Upon asset drop, transitions the Human Gate Task to `completed` and computes the newly unblocked active frontier.
    - Broadcasts unblock event over WebSocket to resume automated agent execution.
  - **Verification Seam**: `tests/human_gate_test.rs::test_human_gate_task_blocks_and_unblocks_agent_frontier`

---

### Phase 4: Epic Hub Dashboard & Export

- **TASK-CORE-007-4**: Epic Hub Dashboard & Progress Reporting
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-010`
  - **Blocked by**: `TASK-CORE-006-1` (UI Shell), `TASK-CORE-004-3` (Project API)
  - **What it delivers**:
    - Implements `EpicHubView` listing all project epics with status pills (`Drafting`, `Active`, `Verifying`, `Done`).
    - Displays progress indicators (percentage of passing red-to-green seam tests and completed tasks).
    - Generates one-click exportable Markdown/JSON milestone reports.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx::renders_epic_cards_and_progress_bars`

---

### Phase 5: Visual Lineage Canvas & Task DAG Explorer

- **TASK-CORE-007-5**: Visual Lineage Graph Canvas & Task DAG Explorer
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-008`
  - **Blocked by**: `TASK-CORE-007-4`, `TASK-CORE-003-6` (Hybrid Query Engine)
  - **What it delivers**:
    - Implements interactive graph canvas rendering end-to-end traceability: `Epic -> Spec -> Plan Task -> Test -> Code Symbol`.
    - Highlights active frontier tasks with visual glowing outline.
    - Allows clicking any node to inspect detailed markdown view or test run output in the dockable panel.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx::renders_dag_nodes_and_interactivity`

---

### Phase 6: Multi-Epic Discovery, Home Dashboard & Tab Content Rendering (Amended Frontier)

- **TASK-CORE-007-6**: Dynamic Multi-Epic Discovery (`001-aico-core` & `002-aico-specs`) via Server API
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Adds backend endpoint `GET /api/v1/epics` in `crates/aico-server` scanning the active project's `epics/` directory.
    - Dynamically lists all epics (`001-aico-core`, `002-aico-specs`) in the UI sidebar, search index, and dashboard.
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs::test_list_epics_endpoint`

- **TASK-CORE-007-7**: Workspace Home / Dashboard View with Epic Progress & Active Frontier
  - **Status**: Done
  - **Addresses**: `REQ-001`
  - **Blocked by**: `TASK-CORE-007-6`
  - **What it delivers**:
    - Implements dedicated Home / Dashboard view component showing all active epics, progress completion rings, verified test counts, and unblocked active task frontiers.
    - Provides instant navigation cards to open epics, specs, and terminals.
  - **Verification Seam**: `frontend/tests/DashboardView.test.tsx`

- **TASK-CORE-007-8**: Active Tab Content Renderer (Spec Viewer, Code Viewer, ADR Viewer, Epic Details)
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-004`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Mounts dynamic tab view switcher in main canvas area: when a tab is active, renders its specific content (Spec Viewer, ADR Viewer, Code Viewer, or Epic Hub).
    - Ensures clicking items in sidebar tree, search modal, or dashboard opens and switches to the active tab cleanly.
  - **Verification Seam**: `frontend/tests/TabContentRenderer.test.tsx`

- **TASK-CORE-007-9**: Epic Hub Accordion Content Expansion (Collapsed 1-line vs Expanded Rich Content)
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-003`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updates `EpicHubView` accordions: when collapsed, displays concise 1-line description; when expanded, renders the actual rich content (full narrative overview, architectural decisions table, clickable specifications matrix, test seam list, task DAG, asset drop zone).
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`

- **TASK-CORE-007-10**: 2-Column Epic Hub Layout & Spatial Optimization
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Restructures `EpicHubView` into an optimized layout:
      - Top: Overview section, open by default, displaying summary, domain terms, and openable Features bundle list.
      - 2-Column Body:
        - Left column: Collapsible accordions for Brainstorm, Specs, Deep Dive, and Assets (all closed by default). Assets includes the file drop zone inside when expanded.
        - Right column: Task Plan alone on the right, open by default.
      - Bottom: Interactive Lineage Graph canvas.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`

- **TASK-CORE-007-11**: Visual Wave Task Plan with Overall & Per-Wave Progress Tracking
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-005`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Renders all tasks in exact chronological order grouped into visually distinct waves (e.g. Wave 1 Host Runtime, Wave 2 LAN Security, etc.).
    - Adds an overall progress track bar for the whole Task Plan and dedicated progress track bars for each individual wave.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`

- **TASK-CORE-007-12**: Feature Bundle View & SDD Frontmatter Parser Fix
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-004`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Adds `FeatureViewer` component for feature bundles (e.g. `01-host-runtime`) holding both its `SPEC` and `PLAN`.
    - Enables opening features from sidebar tree, search results, and epic overview.
    - Fixes YAML frontmatter parser in `server.rs` to stop at the terminating `---`, preventing body markdown examples from overwriting status (resolving the `<Draft|Pending|In Progress|Completed>` issue for `002-aico-specs`).
  - **Verification Seam**: `crates/aico-server/tests/project_api_test.rs`, `frontend/tests/FeatureViewer.test.tsx`

## 3. Amendment Log

- **2026-10-06T03:35:00Z**: Appended `TASK-CORE-007-10`, `TASK-CORE-007-11`, and `TASK-CORE-007-12` via `/aico-plan-amendment` for spatial 2-column layout in EpicHubView, visual wave task plan with overall and per-wave progress bars, feature bundle navigation, and frontmatter parsing lockdown.
