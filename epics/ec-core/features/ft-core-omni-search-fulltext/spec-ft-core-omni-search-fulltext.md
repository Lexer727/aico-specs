---
id: spec-ft-core-omni-search-fulltext
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 9
title: "Workspace Tree Full-Text Fuzzy Search and OmniSearch Integration"
status: "Completed"
derives_from: ["brain-ec-core", "adr-0005"]
created_at: "2026-10-09T00:15:00Z"
---

# Feature Specification: Workspace Tree Full-Text Fuzzy Search and OmniSearch Integration

## 1. Problem Statement & User Value

In the AICO WebUI, the sidebar search bar initially only performed strict exact-substring filtering against entity IDs and titles (`e.id`, `e.title`, `f.id`, `f.name`, `s.id`, `s.title`, `a.id`, `a.title`). Full-text search across associated artifact content was missing.

While in-memory search was implemented for Epics, Features, and ADRs, full-text search inside specification documents and codebase files exhibits critical gaps:
1. **Spec Full-Text Omission**: When terms inside a specification (e.g. `"Remote public internet hosting"`) are queried, the search engine indexes `raw_spec_content` only onto the parent feature (`ft-...`), but omits full-text content from the specification entity (`spec-...`). Consequently, the spec does not appear under the `specs` scope in OmniSearch or the `Specifications` tree accordion.
2. **Hardcoded Code Tree & Truncated CodeViewer**: The sidebar Code section displays a hardcoded list of 12 static files rather than discovering live codebase files that contain `@implements` annotations. Furthermore, the `CodeViewer` renders static mock snippets of ~20 lines instead of fetching and rendering complete source files.
3. **Missing Codebase Full-Text Search**: The search engine only indexes the file path string of code files, and `/api/v1/search` in `aico-server` does not search inside codebase files on disk.

This feature resolves these issues by:
1. Indexing full specification content on both the child specification (`spec-...`) under `specs` scope and the parent feature (`ft-...`) under `features` scope.
2. Providing a backend codebase scanner endpoint (`GET /api/v1/code/files`) that scans the active project's codebase, identifies files containing `@implements` annotations, and displays them in the sidebar tree.
3. Enabling full file streaming via `GET /api/v1/raw?path=...` (resolving against `codebase_path` when not found in `specs_path`) and rendering complete source files in `CodeViewer`.
4. Implementing full-text codebase search in `GET /api/v1/search` and integrating code hits into `OmniSearchModal`.

---

## In-Scope

- **Full-Text In-Memory Search Index**:
  - Indexing Epics: `id`, `title`, `executive_summary`, `brainstorm_content`, `deepdive_content`, `ubiquitous_language`.
  - Indexing Features: `id`, `name`, `description`, `in_scope`, `out_of_scope`, `brainstorm_content`, `deepdive_content`.
  - Indexing Specifications: full markdown content (`raw_spec_content`, requirement statements, and in-scope text) across all epics and standalone features.
  - Indexing Plans: Tasks (`id`, `title`, `description`).
  - Indexing ADRs: `id`, `title`, `context`, `decision`, `consequences`.
  - Indexing Glossary: `term`, `definition`.
- **Dual-Match for Specifications**:
  - Querying terms inside a specification document returns both the Specification entity under `specs` scope and the parent Feature under `features` scope with match snippets.
- **Codebase File Discovery & Annotation Scanner**:
  - Exposing `GET /api/v1/code/files` in `aico-server` scanning `codebase_path` (ignoring `.git`, `target`, `node_modules`, binary files) to identify source files with `@implements` annotations and symbol locations.
  - Displaying live discovered files with `@implements` indicator badges in the tree view Code section.
- **Complete File Streaming & Syntax Display**:
  - Updating `GET /api/v1/raw` to resolve relative paths against `codebase_path` when not found in `specs_path`.
  - Updating `CodeViewer.tsx` to asynchronously fetch and render full file contents.
- **Backend Full-Text Codebase Search**:
  - Supporting codebase full-text search in `GET /api/v1/search` scanning files under `codebase_path`.
  - Feeding backend code hits with line numbers and snippets into `OmniSearchModal` under `scope: 'code'`.
- **Fuzzy Matching Algorithm**:
  - Typographical error tolerance with approximate string matching (Levenshtein distance $\le 2$ or sub-token similarity).
  - Prefix matching, acronym matching, and word-boundary matching.
- **Sidebar Accordion Tree Integration (`ShellLayout.tsx`)**:
  - Auto-expanding tree sections (`Epics`, `Features`, `Specs`, `Code`, `ADRs`) containing matching results.
  - Preserving scope checkbox toggles (`Epics`, `Features`, `Specs`, `Code`, `ADRs`).
  - Rendering an inline clear (`X`) button inside the `"Search workspace..."` input bar when `searchQuery` is non-empty, clearing the filter on click or `Escape` keydown.
- **Global Cross-Project OmniSearch Modal Integration (`OmniSearchModal.tsx`, `App.tsx`)**:
  - Intercepting `Ctrl+K` globally with `e.preventDefault()`.
  - Clicking search trigger button in header opens `OmniSearchModal`.
  - Selecting items switches active project and opens target tab.

---

## Out-of-Scope (Future Upgrades)

- **Annotation Grammar Rework**: Modifying the `@implements` syntax across all repositories (e.g. `@implements ft-...::task-...::REQ-...`) is reserved for a future architectural wave; current `@implements spec-...::REQ-...` syntax is strictly preserved.
- **Sidebar Tree Cross-Project Mixing**: The workspace sidebar search remains strictly project-specific, filtering the current active project's tree.
- **Semantic Vector Embedding Search**: Vector embeddings via LanceDB / embedding models are reserved for a future AI semantic retrieval upgrade.
- **Editing or mutating artifacts directly from the search interface**.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer searching for a capability described in an executive summary or task description, I want the sidebar search bar to find and display the matching Epic or Feature even if the query does not appear in its title, so that I can discover work without knowing exact IDs.
   - *Independent Test*: Enter a search query matching a task description or executive summary; assert the containing feature or epic appears in the filtered sidebar tree.

2. **US-02** (Priority: P1): As a developer with a minor typo (e.g. typing `ft-cor-host-runtime`), I want the search engine to fuzzy-match `ft-core-host-runtime`, so that slight spelling mistakes do not result in empty search results.
   - *Independent Test*: Enter `ft-cor-host-runtime` into the search input; assert `ft-core-host-runtime` is returned.

3. **US-03** (Priority: P1): As a developer pressing `Ctrl+K` in the WebUI in Google Chrome, I want the AICO OmniSearch modal to open rather than Chrome's Google search URL bar, so that keyboard navigation works reliably.
   - *Independent Test*: Simulate `Ctrl+K` keydown event; assert `event.preventDefault()` is invoked and the OmniSearch modal opens.

4. **US-04** (Priority: P2): As a developer who prefers mouse interaction, I want a clickable search button in the top navigation or sidebar header, so that I can open the OmniSearch modal without remembering keyboard shortcuts.
   - *Independent Test*: Click the search button in the top navigation; assert the OmniSearch modal opens.

5. **US-05** (Priority: P2): As a client or CLI tool querying the backend, I want a `GET /api/v1/search?q=...` endpoint to retrieve matching artifacts across the filesystem, so that headless search queries are supported.
   - *Independent Test*: Send HTTP GET request to `/api/v1/search?q=runtime`; assert matching feature and spec artifacts are returned in JSON.

6. **US-06** (Priority: P1): As a developer searching for text written inside a specification (e.g. `"Remote public internet hosting"`), I want the specification itself (`spec-...`) to appear under the `specs` scope in OmniSearch and in the `Specifications` tree section, alongside the parent feature.
   - *Independent Test*: Query a term present inside `spec-ft-core-host-runtime.md`; assert `spec-ft-core-host-runtime` is returned with `scope: 'specs'` and `ft-core-host-runtime` is returned with `scope: 'features'`.

7. **US-07** (Priority: P1): As a developer inspecting code in the sidebar tree, I want to see the live codebase files containing `@implements` annotations and click to view their full contents in `CodeViewer`.
   - *Independent Test*: Inspect the Code section in the sidebar; assert live files appear with `@implements` count badges, and clicking opens the full source file.

8. **US-08** (Priority: P2): As a developer searching for code via OmniSearch, I want full-text search across codebase files to find matching lines and snippets.
   - *Independent Test*: Query a source code function or string with `code` scope enabled; assert matching code file, line number, and snippet appear in OmniSearch.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: When a user enters a search query containing minor typographical errors (e.g., character omission, transposition, or substitution within edit distance $\le 2$), the search engine shall fuzzy-match and return the target entity.
- **REQ-002**: When a user enters a search term present in any associated document content (including epic executive summaries, brainstorm documents, deep dive analyses, feature descriptions, in-scope/out-of-scope lists, EARS requirement statements, task titles/descriptions, ADR context/decisions, or glossary definitions), the search engine shall return the associated parent entity as a search match; specifically for terms within a specification document, the search engine shall return both the child Specification (under `specs` scope) and the parent Feature (under `features` scope).
- **REQ-003**: While a search query is active in the workspace sidebar (`ShellLayout.tsx`), the accordion tree shall automatically expand all sections containing matching items and display matching nodes with visual match indicators.
- **REQ-004**: While a search query is active, the workspace sidebar shall honor active scope filter checkboxes (`Epics`, `Features`, `Specs`, `Code`, `ADRs`) by restricting displayed matches to enabled scopes.
- **REQ-005**: When `Ctrl+K` is pressed anywhere in the WebUI window, the application shall intercept the event, call `event.preventDefault()` to prevent browser URL bar interception, and open the `OmniSearchModal`.
- **REQ-006**: When a user clicks the search trigger button in the top navigation or sidebar header, the application shall open the `OmniSearchModal`.
- **REQ-007**: When an item is selected from the `OmniSearchModal`, the application shall close the modal, switch the active project if the target belongs to another registered project, open the appropriate tab, and navigate to the selected entity.
- **REQ-008**: When a `GET /api/v1/search?q=<query>` request is sent to the backend server, the server shall return a JSON array of matching artifacts discovered across registered projects on the filesystem.
- **REQ-009**: The client search engine and server search endpoint shall index full specification markdown content (`raw_spec_content`, requirement statements, and in-scope text) across all epics and standalone features under the `specs` scope.
- **REQ-010**: The backend server shall expose `GET /api/v1/code/files` scanning `codebase_path` (respecting ignore filters: `.git`, `target`, `node_modules`, binary files) to discover source files with `@implements` annotations and symbol locations, and the tree view Code section shall display these live discovered files with annotation badges.
- **REQ-011**: When a file path is requested via `GET /api/v1/raw?path=...`, the server shall resolve the path against `codebase_path` when not found in `specs_path`, and the `CodeViewer` component shall asynchronously fetch and render the complete file content.
- **REQ-012**: When a user searches with the `code` scope enabled, `GET /api/v1/search` shall search source files across `codebase_path` and return matching code files with line numbers and matched snippets to OmniSearch under `scope: 'code'`.
- **REQ-013**: While `searchQuery` in the workspace sidebar search input (`data-testid="sidebar-search-input"`) is non-empty, `ShellLayout` shall render an inline clear button (`X`, `data-testid="sidebar-search-clear-btn"`, `aria-label="Clear search"`), and when clicked or when `Escape` is pressed inside the input, the component shall reset `searchQuery` to `""` and restore the unfiltered accordion tree.

---

## 4. Designated Public Test Seams

- **Search Engine Unit Seam**: `frontend/tests/searchEngine.test.ts` (verifies REQ-001, REQ-002, REQ-009)
- **Sidebar Tree Fuzzy Seam**: `frontend/tests/ShellLayoutSearch.test.tsx` (verifies REQ-003, REQ-004, REQ-010, REQ-013)
- **OmniSearch Shortcut & Interaction Seam**: `frontend/tests/OmniSearchModal.test.tsx` (verifies REQ-005, REQ-006, REQ-007, REQ-012)
- **CodeViewer Live Seam**: `frontend/tests/CodeViewer.test.tsx` (verifies REQ-011)
- **Backend Search Route Seam**: `crates/aico-server/tests/search_test.rs` (verifies REQ-008, REQ-012)
- **Backend Codebase Scanner Seam**: `crates/aico-server/tests/code_scanner_test.rs` (verifies REQ-010, REQ-011)
