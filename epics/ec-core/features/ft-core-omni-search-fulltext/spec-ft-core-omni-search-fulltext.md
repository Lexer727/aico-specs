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

In the AICO WebUI, the sidebar search bar currently only performs strict exact-substring filtering against entity IDs and titles (`e.id`, `e.title`, `f.id`, `f.name`, `s.id`, `s.title`, `a.id`, `a.title`). Full-text search across associated artifact content is completely missing.

If a developer searches for terms located within an Epic's executive summary, brainstorm notes (`brain-*.md`), deep dive analyses (`deepdive-*.md`), feature purpose/scope, EARS requirement statements (`spec-*.md`), task titles or descriptions (`plan-*.md`), or ADR decisions, the search returns zero results.

Furthermore, the existing search matching is completely rigid: minor typographical errors (e.g. typing `ft-cor-host-runtime` instead of `ft-core-host-runtime`) fail to find the desired entity.

Additionally, in Google Chrome and modern browsers, pressing `Ctrl+K` intercepts the browser's default behavior and opens the browser address bar for Google search because the keydown event is not intercepted at the top level with `event.preventDefault()`, and the existing `OmniSearchModal` component is disconnected from live workspace data.

This feature resolves these issues by:
1. Providing an in-memory full-text fuzzy search engine indexing all associated document content (executive summaries, brainstorms, deep dives, requirements, task plans, ADRs, glossary) alongside identifiers.
2. Supporting fuzzy typographical tolerance so minor spelling errors match target entities.
3. Automatically expanding matching accordion tree sections in the sidebar with visual match indicators.
4. Integrating the global `OmniSearchModal` with live workspace data, intercepting `Ctrl+K` and `Ctrl+P` with `event.preventDefault()` to prevent browser hijacking, and providing a clickable search button in the top navigation.
5. Providing a backend `GET /api/v1/search` endpoint in `aico-server` for scanning disk artifacts and files.

---

## In-Scope

- **Full-Text In-Memory Search Index**:
  - Indexing Epics: `id`, `title`, `executive_summary`, `brainstorm_content`, `deepdive_content`, `ubiquitous_language`.
  - Indexing Features: `id`, `name`, `description`, `in_scope`, `out_of_scope`, `brainstorm_content`, `deepdive_content`.
  - Indexing Specifications: EARS requirements (`id`, `statement`, `pattern`).
  - Indexing Plans: Tasks (`id`, `title`, `description`).
  - Indexing ADRs: `id`, `title`, `context`, `decision`, `consequences`.
  - Indexing Glossary: `term`, `definition`.
- **Fuzzy Matching Algorithm**:
  - Typographical error tolerance with approximate string matching (Levenshtein distance $\le 2$ or sub-token similarity).
  - Prefix matching, acronym matching, and word-boundary matching.
- **Sidebar Accordion Tree Integration (`ShellLayout.tsx`)**:
  - Auto-expanding tree sections (`Epics`, `Features`, `Specs`, `Code`, `ADRs`) that contain matching results when a query is active.
  - Displaying match indicators and hit counters on nodes matching via full-text content.
  - Preserving scope checkbox toggles (`Epics`, `Features`, `Specs`, `Code`, `ADRs`).
- **Global Cross-Project OmniSearch Modal Integration (`OmniSearchModal.tsx`, `App.tsx`)**:
  - Intercepting `Ctrl+K` globally with `e.preventDefault()` to prevent Chrome address bar focus.
  - Adding a clickable Search button/icon in the top application header and sidebar header.
  - Feeding live indexed workspace results across all registered projects into `OmniSearchModal`, displaying a project badge on each result.
  - When selecting an item from a different project, switching the active project (`setActiveProject`) and opening the corresponding viewer tab.
- **Backend Search Endpoint (`aico-server`)**:
  - Exposing `GET /api/v1/search?q=<query>&project=<id>` in `aico-server` scanning filesystem artifacts across all registered projects (or filtered to a specific project if provided).
- **Automated Seam Tests**: Unit and integration test suites covering the search engine, sidebar filtering, OmniSearch shortcut handling, and server endpoint.

---

## Out-of-Scope (Future Upgrades)

- **Sidebar Tree Cross-Project Mixing**: The workspace sidebar search remains strictly project-specific, filtering the current active project's tree.
- **Dedicated Popover / Full-Page Ranked Results Panel with Syntax Snippets**: Option A from Socratic Grilling Q2 (rich flyout list with multi-line highlighted code snippets) is scheduled as a future project-wide search upgrade; the primary interaction for this feature centers on tree filtering and the `OmniSearchModal`.
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

---

## 3. Functional Requirements (EARS)

- **REQ-001**: When a user enters a search query containing minor typographical errors (e.g., character omission, transposition, or substitution within edit distance $\le 2$), the search engine shall fuzzy-match and return the target entity.
- **REQ-002**: When a user enters a search term present in any associated document content (including epic executive summaries, brainstorm documents, deep dive analyses, feature descriptions, in-scope/out-of-scope lists, EARS requirement statements, task titles/descriptions, ADR context/decisions, or glossary definitions), the search engine shall return the associated parent entity as a search match.
- **REQ-003**: While a search query is active in the workspace sidebar (`ShellLayout.tsx`), the accordion tree shall automatically expand all sections containing matching items and display matching nodes with visual match indicators.
- **REQ-004**: While a search query is active, the workspace sidebar shall honor active scope filter checkboxes (`Epics`, `Features`, `Specs`, `Code`, `ADRs`) by restricting displayed matches to enabled scopes.
- **REQ-005**: When `Ctrl+K` is pressed anywhere in the WebUI window, the application shall intercept the event, call `event.preventDefault()` to prevent browser URL bar interception, and open the `OmniSearchModal`.
- **REQ-006**: When a user clicks the search trigger button in the top navigation or sidebar header, the application shall open the `OmniSearchModal`.
- **REQ-007**: When an item is selected from the `OmniSearchModal`, the application shall close the modal, switch the active project if the target belongs to another registered project, open the appropriate tab, and navigate to the selected entity.
- **REQ-008**: When a `GET /api/v1/search?q=<query>` request is sent to the backend server, the server shall return a JSON array of matching artifacts discovered across registered projects on the filesystem.

---

## 4. Designated Public Test Seams

- **Search Engine Unit Seam**: `frontend/tests/searchEngine.test.ts` (verifies REQ-001, REQ-002)
- **Sidebar Tree Fuzzy Seam**: `frontend/tests/ShellLayoutSearch.test.tsx` (verifies REQ-003, REQ-004)
- **OmniSearch Shortcut & Interaction Seam**: `frontend/tests/OmniSearchModal.test.tsx` (verifies REQ-005, REQ-006, REQ-007)
- **Backend Search Route Seam**: `crates/aico-server/tests/search_test.rs` (verifies REQ-008)
