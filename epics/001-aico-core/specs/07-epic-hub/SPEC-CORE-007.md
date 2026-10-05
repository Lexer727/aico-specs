---
id: SPEC-CORE-007
type: spec
aico_version: "0.1.1"
epic: "001-aico-core"
title: "Epic Hub, Visual Lineage Canvas & Multi-Modal Drop Zones"
status: "Completed"
derives_from: ["ADR-0005", "ADR-0007"]
created_at: "2026-10-05T19:15:00Z"
---

# Feature Specification: Epic Hub, Visual Lineage Canvas & Asset Drop Zones


---

## 1. Problem Statement & User Value

Working on an engineering capability requires inspecting multiple interdependent artifacts: the initial brainstorm, system blast radius, EARS specifications, task DAGs, reference design assets, and live test execution status. Jumping between disconnected browser tabs causes disorientation and cognitive fatigue. Furthermore, developers need a seamless way to provide non-code context (mockups, audio, guides) that directly unblocks automated tasks.

This specification defines the Epic Hub: a unified single-view workspace presenting all SDD artifacts in clean collapsible accordion sections with pop-out tab capabilities, an interactive Kùzu visual lineage graph canvas, and multi-modal drag-and-drop zones that satisfy Human Gate Tasks in the Plan DAG.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want all artifacts for an epic (brainstorm, deep dive, spec, plan) viewable in one single page with collapsible accordions, so that I never have to context-switch across multiple tabs.
   - *Independent Test*: Open an Epic Hub tab and verify all 5 artifact sections expand and collapse cleanly without reloading page state.
2. **US-02** (Priority: P1): As a developer, I want an "Open in separate tab" icon (`↗`) on every artifact accordion, so that I can detach a spec or task DAG into its own full-screen tab when focusing on it.
   - *Independent Test*: Click the pop-out button on the Spec accordion and verify the specification opens in an independent tab while remaining synchronized.
3. **US-03** (Priority: P1): As a developer, I want to drag and drop reference mockups or sound effects into an epic's asset zone, so that documents are indexed into LanceDB and media is available to multi-modal agents.
   - *Independent Test*: Drop a PDF document into the assets accordion and verify it is chunked and searchable via omni-search within 2 seconds.
4. **US-04** (Priority: P1): As a developer, I want human deliverable tasks in the Plan DAG to render with a drop zone that unblocks dependent agent tasks when satisfied, so that I can easily provide needed assets.
   - *Independent Test*: Drop an image onto a pending `Human Gate Task` card and verify the task completes and the dependent agent task unlocks on the frontier.
5. **US-05** (Priority: P2): As a developer, I want to explore caller chains and blast-radius dependencies on an interactive property graph canvas, so that I can visually verify which modules will be affected before implementation.
   - *Independent Test*: Open the Deep Dive visual canvas, click a function node, and verify its 2-hop caller and callee nodes highlight with smooth zoom and pan controls.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The EpicHubView shall render a consolidated single-page view for the active epic displaying Overview, Brainstorm, ADRs, Child Specifications, Deep Dive, Task Plan, and Assets.
- **REQ-002**: The AccordionManager shall render each artifact section as an independently expandable and collapsible card.
- **REQ-003**: The LineageGraphCanvas shall render Kùzu property graph nodes (Epics, Specs, Requirements, Modules, CodeSymbols, Tests) with interactive pan, zoom, and node selection controls.
- **REQ-004**: The AssetDropManager shall accept file drag-and-drop events and clipboard paste (`Ctrl+V`) actions across the Chat, Epic Hub assets pane, and Plan task cards.

### Event-Driven Requirements
- **REQ-005**: When the user clicks the pop-out icon (`↗`) on any accordion header, the TabBarManager shall open that specific artifact in a new dedicated tab.
- **REQ-006**: When a search hit matches an artifact within an epic, the EpicHubView shall open the parent Epic Hub, expand the matching accordion section, and scroll the target element into view.
- **REQ-007**: When a file is dropped into an asset drop zone, the AssetDropManager shall persist the file to disk in `<project>-specs/epics/<epic-slug>/assets/` and trigger LanceDB text chunking for documents.
- **REQ-008**: When a file matching the expected MIME type is dropped onto a `Human Gate Task` card in the Plan DAG, the HumanGateManager shall mark the task as `status: "Completed"` and transition dependent tasks from `status: "Completed"` to `status: "Completed"`.
- **REQ-009**: When test execution events arrive via WebSocket, the EARSRequirementTable shall update the verification badge (`Green [Passed]` or `Red [Failed]`) for each corresponding `REQ-###` row in real time.

### State-Driven Requirements
- **REQ-010**: While viewing the LineageGraphCanvas, selecting any node shall highlight upstream provenance edges (towards specs and brainstorms) in blue and downstream blast-radius edges (towards callers and tests) in amber.
- **REQ-011**: While a child specification contains unverified requirements, the EpicHubView shall display a summary status badge indicating verified requirement count (e.g. `[4/7 Verified]`).

### Unwanted Behavior & Error Handling
- **REQ-012**: If a file dropped onto a `Human Gate Task` card violates the specified MIME type constraints, then the AssetDropManager shall reject the file and display an actionable format error message.
- **REQ-013**: If a graph rendering traversal exceeds 500 nodes, then the LineageGraphCanvas shall cluster distant module nodes to maintain smooth 60 FPS viewport rendering.

### Optional Feature Requirements
- **REQ-014**: Where an image asset is dropped into the Chat input, the ChatManager shall attach the image thumbnail to the prompt payload and transmit it to multi-modal vision LLMs.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Epic Hub Component Seam (`WebUI.components.EpicHub`)**:
   - Signature: `<EpicHub epicId="001-aico-core" />`
   - Verifies accordion state toggling, artifact loading, and EARS badge rendering.
2. **Lineage Graph Canvas Seam (`WebUI.components.LineageCanvas`)**:
   - Signature: `renderGraph(elements: GraphElements, options: RenderOptions)`
   - Tests Cytoscape/WebGL graph rendering, node click handlers, and blast-radius edge highlighting.
3. **Human Gate Task Manager Seam (`WebUI.components.HumanGateTask`)**:
   - Signature: `onAssetDelivered(taskId: string, file: File) -> Promise<TaskResolution>`
   - Verifies MIME validation, asset upload, and DAG frontier unblocking.

---

## 5. Out of Scope

- Direct 3D CAD modeling or vector graphic manipulation inside the UI.
- Video transcoding or audio manipulation (AICO displays and plays media natively via standard browser HTML5 `<video>` and `<audio>` elements).
- Real-time multiplayer collaborative cursor editing (single-developer local-first model).

---

## 6. Open Clarifications

None. Ratified in [ADR-0005](../../../../adr/0005-unified-accordion-workspace-and-dockable-panels.md) and [ADR-0007](../../../../adr/0007-multi-modal-assets-and-human-gate-tasks.md).
