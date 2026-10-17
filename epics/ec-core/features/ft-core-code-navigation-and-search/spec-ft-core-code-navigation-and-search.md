---
id: spec-ft-core-code-navigation-and-search
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 13
title: "Code Navigation, Inline Annotations & Search Integration"
status: "Completed"
derives_from: ["brain-ec-core", "spec-ft-core-omni-search-fulltext", "spec-ft-specs-traceability-syntax-evolution"]
created_at: "2026-10-17T17:40:00Z"
---

# Feature Specification: Code Navigation, Inline Annotations & Search Integration

## 1. Problem Statement & User Value

The AICO WebUI allows developers to inspect specifications, plans, epics, and codebase files directly in the browser. While recent evolutions introduced strict in-band annotations (`@implements` and `@verifies`) and full-text workspace search, several critical gaps hinder bidirectional code navigation and discovery:

1. **Cluttered Inline Annotation Display**: The `CodeViewer` currently renders 3 separate inline badges (`[feature]`, `[task]`, and `[req]`) alongside annotated code lines. This creates visual noise in dense code and lacks direct interactivity to jump to the referenced feature.
2. **Disconnected Sidebar Code Search**: While the global `OmniSearchModal` queries backend full-text search over file contents, the sidebar Code tree view search input only matches file path strings and annotation tag strings. Typing code content or function signatures (e.g. `pub async fn bind(&mut self)`) produces empty results in the sidebar tree.
3. **Missing Code Links in Features and Epics**: When viewing a feature or an epic, there is no direct link to the implementation source files. In `FeatureViewer`, developers must manually navigate to the Code section to find relevant files. In the Task View, task cards do not indicate which source files implement them. In `EpicHubView`, there is no aggregated summary of codebase touchpoints.

This specification closes these gaps by:
- Consolidating `CodeViewer` annotations into a single, compact, clickable feature pill (`[ft-<slug>]`) that jumps directly to the Feature View, preserving task and requirement details in an accessible hover tooltip.
- Integrating backend full-text code search and client-side annotation indexing into the sidebar Code tree view in `ShellLayout`.
- Adding a dedicated "Code Implementations" section above Assets in `FeatureViewer`, as well as direct in-band code links inside individual task cards in the Task View.
- Providing an aggregated "Code Files" roll-up section in `EpicHubView` listing all codebase files implementing any child feature within the epic.

---

## In-Scope

- Single interactive feature badge (`[ft-<feature>]`) in `CodeViewer` with `onSelectFeature` callback navigation.
- Accessible hover tooltip on the feature badge displaying full traceability metadata (`Feature: ft-... | Task: TASK-... | Req: REQ-...`).
- Sidebar Code tree view integration with `/api/v1/search?q=...` full-text search results for code content matching.
- Client-side search index enhancement in `searchEngine.ts` to index code annotation metadata and path tokens.
- Collapsible "Code Implementations" card in `FeatureViewer` above the Assets section showing matching code files and lines.
- Direct code file links on task cards in `FeatureViewer` Task View matching `::TASK-...::` annotations.
- Aggregated "Code Files" section in `EpicHubView` rolling up all source files implementing child features of the active epic.
- Navigation callbacks wiring in `App.tsx` (`onSelectFeature` and `onSelectCode`).

---

## Out-of-Scope

- In-browser code editing or file writing tools.
- Replacing the global OmniSearch modal (`OmniSearchModal`).
- Modifying backend tree-sitter or vector embeddings logic.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer inspecting code in `CodeViewer`, I want in-band `@implements` and `@verifies` annotations to display as a single clickable feature pill (`[ft-<feature>]`) that navigates directly to that feature, so that code remains readable and I can jump to specifications in one click.
   - *Independent Test*: View an annotated code line in `CodeViewer`; verify only one badge (`[ft-<feature>]`) is rendered; click it and confirm `onSelectFeature` is invoked.
2. **US-02** (Priority: P1): As a developer hovering over an annotation badge in `CodeViewer`, I want to see the full traceability metadata (feature, task ID, requirement ID) in a tooltip, so that I don't lose context on specific task and requirement links.
   - *Independent Test*: Hover over the feature pill and verify the `title` attribute contains `Feature: ft-<feature> | Task: TASK-<task> | Req: REQ-<req>`.
3. **US-03** (Priority: P1): As a developer searching code in the sidebar Code tree view, I want search queries to match file content (e.g. `pub async fn bind(&mut self)`), so that I can discover code files by function name or code snippet without opening OmniSearch.
   - *Independent Test*: Enter a function signature present in code into the sidebar search input and verify the containing file appears in the filtered Code section.
4. **US-04** (Priority: P1): As an engineer reviewing a feature in `FeatureViewer`, I want a "Code Implementations" card above Assets listing all code files and line numbers implementing this feature, so that I can directly open the relevant code.
   - *Independent Test*: Open a feature with implemented code and verify the "Code Implementations" card lists matching files with working `onSelectCode` links.
5. **US-05** (Priority: P1): As an engineer tracking task progress in the Task View, I want each task card to display links to its implementing code files, so that I can immediately inspect the code delivering each task.
   - *Independent Test*: In the Task View, inspect a completed task and verify clickable file badges are rendered for its implementing files.
6. **US-06** (Priority: P2): As an architect browsing an epic in `EpicHubView`, I want an aggregated "Code Files" section listing all source files across all child features, so that I can see the entire codebase footprint of the epic.
   - *Independent Test*: Open `EpicHubView` and verify a "Code Files" section lists all code files linked across all child features of the epic.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements

- **REQ-001**: The `CodeViewer` shall render in-band implementation and verification annotations with a single interactive feature pill displaying the feature identifier (`ft-<feature>`), omitting inline task and requirement badges.
- **REQ-002**: When a user hovers over the feature badge in `CodeViewer`, the `CodeViewer` shall display full traceability metadata including Feature, Task ID, and Requirement ID in an accessible tooltip (`title` attribute formatted as `Feature: ft-<feature> | Task: TASK-<task> | Req: REQ-<req>`).
- **REQ-003**: When a user clicks the feature badge in `CodeViewer`, the `CodeViewer` shall invoke `onSelectFeature` passing the target feature identifier to switch the active view to that feature.
- **REQ-004**: When the sidebar search input is non-empty, the `ShellLayout` shall query the backend full-text search endpoint (`/api/v1/search?q=...`) with debouncing and include matched code files (`scope === "code"`) in the sidebar Code tree view.
- **REQ-005**: The `searchEngine` shall index code file annotations, target identifiers, and path components into the client-side search index so that typing feature tags, task IDs, or requirement tokens immediately filters the sidebar Code tree.
- **REQ-006**: The `FeatureViewer` shall render a collapsible "Code Implementations" section positioned immediately above the Assets section, displaying all codebase files containing annotations that target the active feature, with clickable links that invoke `onSelectCode`.
- **REQ-007**: When rendering task cards in `FeatureViewer` Task View, the `FeatureViewer` shall match and render clickable code file links (`onSelectCode`) for each source file whose annotations implement that task's identifier.
- **REQ-008**: The `EpicHubView` shall render an aggregated "Code Files" section listing all unique codebase files implementing any child feature belonging to the active epic, providing clickable links to inspect each file in the `CodeViewer`.

---

## 4. Designated Public Test Seams

1. **SEAM-001**: `frontend/tests/CodeViewer.test.tsx`
   - Verifies single feature pill rendering, tooltip metadata, and `onSelectFeature` navigation click handler.
2. **SEAM-002**: `frontend/tests/ShellLayoutSearch.test.tsx`
   - Verifies sidebar search input filtering with debounced backend full-text search integration.
3. **SEAM-003**: `frontend/tests/FeatureViewer.test.tsx`
   - Verifies "Code Implementations" section above Assets and task-level code links in Task View.
4. **SEAM-004**: `frontend/tests/EpicHubView.test.tsx`
   - Verifies epic-level aggregated "Code Files" section rendering and navigation callbacks.

---

## 5. Open Clarifications

None. Trade-offs and UX decisions ratified in user consultation and ADR-0006.
