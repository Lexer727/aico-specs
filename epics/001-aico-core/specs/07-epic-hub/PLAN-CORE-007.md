# Implementation Plan: Epic Hub, Visual Lineage Canvas & Asset Drop Zones

- **ID**: PLAN-CORE-007
- **Spec Reference**: [SPEC-CORE-007.md](file:///d:/aico/aico-specs/epics/001-aico-core/specs/07-epic-hub/SPEC-CORE-007.md)
- **Status**: Ready

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

- [x] **TASK-CORE-007-1**: Local Asset Ingestion Service & Deduplication
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

- [x] **TASK-CORE-007-2**: UI Drag-and-Drop Zones (Chat, Sidebar, and Task Card)
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

- [x] **TASK-CORE-007-3**: Human Gate Task Engine & Blocker Resolution
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

- [x] **TASK-CORE-007-4**: Epic Hub Dashboard & Progress Reporting
  - **Addresses**: `REQ-001`, `REQ-010`
  - **Blocked by**: `TASK-CORE-006-1` (UI Shell), `TASK-CORE-004-3` (Project API)
  - **What it delivers**:
    - Implements `EpicHubView` listing all project epics with status pills (`Drafting`, `Active`, `Verifying`, `Done`).
    - Displays progress indicators (percentage of passing red-to-green seam tests and completed tasks).
    - Generates one-click exportable Markdown/JSON milestone reports.
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx::renders_epic_cards_and_progress_bars`

---

### Phase 5: Visual Lineage Canvas & Task DAG Explorer

- [x] **TASK-CORE-007-5**: Visual Lineage Graph Canvas & Task DAG Explorer
  - **Addresses**: `REQ-002`, `REQ-008`
  - **Blocked by**: `TASK-CORE-007-4`, `TASK-CORE-003-6` (Hybrid Query Engine)
  - **What it delivers**:
    - Implements interactive graph canvas rendering end-to-end traceability: `Epic -> Spec -> Plan Task -> Test -> Code Symbol`.
    - Highlights active frontier tasks with visual glowing outline.
    - Allows clicking any node to inspect detailed markdown view or test run output in the dockable panel.
  - **Verification Seam**: `frontend/tests/VisualLineageCanvas.test.tsx::renders_dag_nodes_and_interactivity`

---

## 3. Initial Active Frontier

- `TASK-CORE-007-1`: Local Asset Ingestion Service & Deduplication (Blocked by `TASK-CORE-003-1`).
