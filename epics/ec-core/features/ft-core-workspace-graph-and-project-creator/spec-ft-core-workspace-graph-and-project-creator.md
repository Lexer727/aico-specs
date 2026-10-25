---
id: spec-ft-core-workspace-graph-and-project-creator
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 17
title: "Workspace UI Refinement, Project Creator & Interactive Lineage Graph"
status: "Completed"
derives_from: ["brain-ec-core", "ADR-0004", "ADR-0005", "ADR-0008"]
created_at: "2026-10-23T19:05:21.133Z"
---

# Feature Specification: Workspace UI Refinement, Project Creator & Interactive Lineage Graph

## 1. Problem Statement & User Value

Several core workspace navigation and visualization workflows require refinement to eliminate non-functional visual clutter, streamline multi-project initialization, and replace static mock lineage diagrams with a live property graph:

1. **Sidebar Header Clutter**: The left sidebar header in `ShellLayout.tsx` displays a blinking blue dot next to `AICO WORKSPACE` that conveys no telemetry or state.
2. **Epic Task Plan Ergonomics**: In `EpicHubView.tsx`, each feature group in the `Task Plan` column includes a separate `View Feature` button and always renders all child tasks expanded, making large epics with completed features noisy to scan. Making the `ft-*` badge itself clickable, making each feature group collapsible, and auto-collapsing features that are 100% implemented significantly improves readability.
3. **Interactive Project Creation**: `ProjectSwitcher.tsx` only selects existing projects and cannot scaffold a new project. Developers need an integrated `Create New Project` workflow in the top project selector that prompts for a project name, creates `<project-name>-specs` under the workspace root with `.aico`, `adr`, `epics`, `features`, `.aico-project.json`, and `glossary.md`, initializes the Git repository (`git init`), registers the project, and switches to it.
4. **Live AicoGraphStore Property Graph & Lineage Canvas**: `AicoGraphStore` (ratified in [ADR-0011](../../../../adr/0011-aico-graph-store-and-lancedb-hybrid-storage.md)) stores and queries artifact and code relationships (`ADR` -> `Epic` / `Brainstorm` -> `Feature` -> `Spec` -> `Requirement` <- `Task` / `Plan` <- `Code`) and renders a true interactive node-and-edge graph with SVG connections in the Epic Hub and Home Dashboard.

---

## In-Scope

- Removing the decorative blinking dot next to `AICO WORKSPACE` in `ShellLayout.tsx`.
- Removing the `View Feature` button in `EpicHubView.tsx` `Task Plan`, making the `ft-*` feature ID badge clickable (`onOpenFeature`), making each feature group in `Task Plan` collapsible, and auto-collapsing any feature group whose tasks are 100% completed (`featPct === 100 && featTotal > 0` or `status === 'Completed'`).
- Adding a `+ New Project` action and inline/modal name prompt in `ProjectSwitcher.tsx` connected to `POST /api/v1/projects`.
- Implementing `POST /api/v1/projects` in `aico-server` and project scaffolding in `aico-core` to create `<workspace-root>/<project-name>-specs` with `.aico/`, `adr/`, `epics/`, `features/`, `.aico-project.json`, and `glossary.md`, plus running `git init`.
- Implementing `AicoGraphStore` in `crates/aico-storage/src/graph.rs` as a functional persistent property graph supporting Cypher node/edge creation, `MATCH` queries, and multi-hop neighborhood traversal for `query_lineage_graph`.
- Exposing `GET /api/v1/graph` (with optional `?epic=<epic-id>` filter) in `aico-server` that builds and returns the complete artifact and code lineage graph (`nodes` and `edges` across ADRs, Epics, Brainstorms, Features, Specs, Plans, Requirements, Tasks, and annotated Code files).
- Updating `VisualLineageCanvas.tsx` and `App.tsx` to fetch and render the live Epic lineage graph with layered node layout, SVG directed edges, interactive node selection (highlighting upstream provenance and downstream blast radius), and click-to-open navigation.

---

## Out-of-Scope

- External cloud graph database dependencies (graph storage remains 100% local and zero-daemon inside `.aico/storage/graph`).
- Destructive deletion of existing `<project-name>-specs` directories via the UI.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer viewing the workspace sidebar, I want the non-functional blue blinking dot next to `AICO WORKSPACE` removed, so that visual animations are reserved for meaningful status indicators.
   - *Independent Test*: Render `ShellLayout` and verify the `AICO WORKSPACE` header contains no `animate-pulse` dot element.
2. **US-02** (Priority: P1): As a developer inspecting an Epic's Task Plan, I want to click the `ft-*` badge directly instead of a `View Feature` button, and have 100% implemented features auto-collapse while remaining manually expandable/collapsible, so that I can focus on active work.
   - *Independent Test*: Render `EpicHubView` with a 100% completed feature and an in-progress feature; verify `View Feature` is absent, clicking `ft-*` invokes `onOpenFeature`, the completed feature is collapsed by default, and clicking its header expands its tasks.
3. **US-03** (Priority: P1): As a developer starting a new initiative, I want to create a new project from the top `ProjectSwitcher` dropdown by entering a project name, so that `<project-name>-specs` is scaffolded with all canonical folders, `.aico-project.json`, `glossary.md`, and an initialized Git repo.
   - *Independent Test*: Call `POST /api/v1/projects` with `{ "name": "demo-app" }` and verify `demo-app-specs` is created with `.aico`, `adr`, `epics`, `features`, `.aico-project.json`, `glossary.md`, and `.git`.
4. **US-04** (Priority: P1): As a developer or agent inspecting an Epic's lineage graph or calling `query_lineage_graph`, I want AicoGraphStore storage and the Epic Hub graph view to reflect real connections between ADRs, Epics, Features, Specs, Plans, Requirements, Tasks, and Code files, so that architectural provenance is accurately visualized.
   - *Independent Test*: Query `GET /api/v1/graph?epic=ec-core` and `AicoGraphStore::execute_query`; verify real nodes and directed edges are returned and rendered with SVG connections in `VisualLineageCanvas`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `ShellLayout` component shall render the `AICO WORKSPACE` sidebar header without a decorative blinking dot indicator.
- **REQ-002**: The `EpicHubView` component shall render each feature group in the `Task Plan` without a `View Feature` button and shall make the feature's `ft-*` badge clickable to open the corresponding feature tab.
- **REQ-003**: The `EpicHubView` component shall allow each feature group in the `Task Plan` to be collapsed and expanded independently.
- **REQ-004**: The `AicoGraphStore` shall persist nodes and directed relationships in `.aico/storage/graph` and evaluate lineage `CREATE`, `MERGE`, and `MATCH` queries against the stored graph.
- **REQ-005**: The HostServer shall expose `GET /api/v1/graph` returning all discovered lineage nodes and directed edges connecting ADRs, Epics, Brainstorms, Deep-Dives, Features, Specifications, Plans, Requirements, Tasks, and annotated Code files, normalizing ADR and artifact IDs case-insensitively (`ADR-0004`) and emitting direct `Spec -> Plan` and `Plan -> Code` edges alongside task-level edges.
- **REQ-006**: The `VisualLineageCanvas` component shall render a 6-column layered graph (`ADR` -> `Epic / Brainstorm / Deep-Dive` -> `Feature` -> `Spec` -> `Plan` -> `Code`) with SVG edges connecting linked artifacts, hiding granular `requirement` and `task` nodes by default behind an explicit `"Show Reqs & Tasks"` toggle so the Epic lineage view remains compact and readable.

### State-Driven Requirements
- **REQ-007**: While a feature in the `EpicHubView` `Task Plan` has 100% of its tasks completed (`featDone === featTotal && featTotal > 0` or `status === 'Completed'`) and has not been manually toggled by the user, the `EpicHubView` shall render that feature's task list in the collapsed state by default.
- **REQ-008**: While a feature in the `EpicHubView` `Task Plan` has incomplete tasks (`featPct < 100`) and has not been manually toggled by the user, the `EpicHubView` shall render that feature's task list in the expanded state by default.
- **REQ-013**: While an epic tab remains open in the workspace, the system shall persist the user's manual feature collapse toggles across tab switches and page reloads in `useAccordionStore` and `sessionStorage`, and shall only reset a feature's collapse state to default when its parent epic tab is closed.
- **REQ-014**: The `VisualLineageCanvas` shall order filter pills strictly by the graph's left-to-right architectural column tiers (`adr`, `epic`, `brainstorm`, `deepdive`, `feature`, `spec`, `plan`, `requirement`, `task`, `code`), support multi-select filtering with all filter pills highlighted when "All" is active, and dynamically render column headings only for tiers with at least one visible node.
- **REQ-015**: When a user left-clicks a node in `VisualLineageCanvas`, the component shall highlight its connected upstream provenance and downstream blast-radius without opening documents or switching tabs, and shall provide a dedicated popout icon button in the top-right corner of each node to open the target artifact in a separate tab.
- **REQ-016**: The `VisualLineageCanvas` shall support zooming via mouse wheel scrolling dynamically anchored around the cursor position (adjusting both zoom scale and canvas pan offsets so the point under the mouse cursor remains invariant), and shall provide an "Open in separate tab" action on the graph card to display the canvas in a dedicated full-window `'graph'` tab.
- **REQ-017**: The `VisualLineageCanvas` component shall render a clean 6-column architectural artifact graph by default (`ADR` -> `Epic` -> `Feature` -> `Specification` -> `Plan` -> `Code Seam`), excluding supplementary brainstorm notes from the Epic column and omitting global granular task dumping.
- **REQ-018**: The `VisualLineageCanvas` component shall support per-feature drill-down that dynamically expands only the selected feature's child `requirement` and `task` nodes into dedicated `Requirement` and `Task` columns between `Specification` and `Plan`, complete with a one-click reset to the Epic overview.
- **REQ-019**: The `VisualLineageCanvas` component shall support tri-state multi-select filter toggling: clicking an active type filter pill while "All" is selected shall unselect that individual type while keeping all other types selected; clicking "All" while "All" is active shall unselect all types to show an empty canvas view; and clicking a type filter pill from the empty view shall select only that individual type.
- **REQ-020**: The `VisualLineageCanvas` component and HostServer graph API shall collapse separate specification and plan file nodes into the parent `Feature` node, rendering an architectural DAG ordered as `ADR` -> `Epic` -> `Feature` -> `Requirement` -> `Task` -> `Code` (displaying 4 tiers in default Epic overview, and expanding dedicated `Requirement` and `Task` tiers between `Feature` and `Code` during feature drill-down).
- **REQ-021**: The `VisualLineageCanvas` component shall omit `requirement` nodes from the visual lineage graph and filter pills, establishing a canonical 5-tier architectural pipeline: `ADR` -> `Epic` -> `Feature` -> `Task` -> `Code` (displaying 4 tiers in default Epic overview, and expanding dedicated `Task` tier between `Feature` and `Code` during feature drill-down).
- **REQ-022**: The `VisualLineageCanvas` component shall render directed edges strictly between immediately adjacent visible column tiers ($j = i + 1$). When intermediate tiers (such as `Task`) are visible, multi-tier bypass edges (such as `Feature -> Code`) shall be suppressed; when intermediate tiers are not visible or unselected, edges shall bridge directly to the next visible tier.
- **REQ-023**: The HostServer graph engine shall parse task definitions in `plan-*.md` exclusively on lines starting with a canonical task declaration prefix (`- **TASK-`, `### TASK-`, or `**TASK-`), preventing dependency notes (`Blocked by`), amendment logs, or rewrite targets from declaring false task nodes or leaking cross-epic verification seams.
- **REQ-024**: The `VisualLineageCanvas` component shall compute feature drill-down node membership strictly through downstream directed traversal from the drilled feature identifier, preventing reverse traversal through shared code verification seams from pulling unrelated features into the canvas.
- **REQ-025**: When an artifact node (such as an Epic or Feature) at column tier $C$ is selected in `VisualLineageCanvas`, all nodes in upstream tiers and in the same tier (tier <= C) shall remain visible, while downstream tiers (tier > C) shall display strictly only nodes connected downstream from the selected node. Edge rendering shall be isolated to connections linked to the selected node. Clicking empty canvas background without dragging (stationary click) or clicking the active node again shall unselect and restore the full graph overview; panning or dragging the canvas viewport shall preserve the active node selection.

### Event-Driven Requirements
- **REQ-009**: When a user selects `Create New Project` in `ProjectSwitcher` and submits a valid project name, the frontend shall send `POST /api/v1/projects` and switch the active workspace project to the newly created project.
- **REQ-010**: When `POST /api/v1/projects` receives a valid project name, the HostServer shall create `<workspace-root>/<project-name>-specs` containing `.aico/`, `adr/`, `epics/`, `features/`, `.aico-project.json`, and `glossary.md`, initialize a Git repository (`git init`), register the project in `WorkspaceRegistry`, and return the created `DiscoveredProject`.
- **REQ-011**: When a user clicks a node in `VisualLineageCanvas`, the component shall highlight its connected upstream provenance and downstream blast-radius edges and provide navigation to open the selected artifact or code file.

### Unwanted Behavior & Error Handling
- **REQ-012**: If `POST /api/v1/projects` is called with an invalid project identifier or a project directory that already exists, then the HostServer shall return HTTP `400 Bad Request` or `409 Conflict` with a descriptive error message without overwriting existing files.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Sidebar & Epic Hub UI Seam (`frontend/tests/EpicHubView.test.tsx` & `frontend/tests/ShellLayout.test.tsx`)**:
   - Signature: `<ShellLayout />` and `<EpicHubView epic={...} onOpenFeature={...} />`
   - Verifies: Absence of blinking dot (`REQ-001`), clickable `ft-*` badge (`REQ-002`), collapsible feature task lists (`REQ-003`), auto-collapse of 100% completed features (`REQ-007`, `REQ-008`).
2. **Project Creator API & UI Seam (`crates/aico-server/tests/project_api_test.rs` & `frontend/tests/ProjectSwitcher.test.tsx`)**:
   - Signature: `POST /api/v1/projects` and `<ProjectSwitcher projects={...} />`
   - Verifies: New project scaffolding (`.aico`, `adr`, `epics`, `features`, `.aico-project.json`, `glossary.md`, `.git`) and error handling (`REQ-009`, `REQ-010`, `REQ-012`).
3. **AicoGraphStore Engine & Lineage Canvas Seam (`crates/aico-storage/tests/storage_hybrid_test.rs`, `crates/aico-mcp/tests/mcp_graph_hybrid_test.rs`, `crates/aico-server/tests/server_test.rs` & `frontend/tests/VisualLineageCanvas.test.tsx`)**:
   - Signature: `AicoGraphStore::execute_query`, `GET /api/v1/graph`, and `<VisualLineageCanvas />`
   - Verifies: Persistent property graph storage/queries (`REQ-004`), lineage graph API (`REQ-005`, `REQ-023`), and SVG node-edge lineage rendering (`REQ-006`, `REQ-011`, `REQ-021`, `REQ-022`, `REQ-024`).

---

## 5. Open Clarifications

None. Ratified in [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md) and [ADR-0005](../../../../adr/0005-unified-accordion-workspace-and-dockable-panels.md) and [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
