---
id: spec-core-webui-shell
type: spec
aico_version: "0.1.2"
epic: "ec-core"
order: 6
title: "WebUI Application Shell, Navigation & Dockable Layout"
status: "Completed"
derives_from: ["ADR-0004", "ADR-0005"]
created_at: "2026-10-05T19:15:00Z"
---

# Feature Specification: WebUI Application Shell, Navigation & Dockable Layout


---

## 1. Problem Statement & User Value

Modern developers work across multiple projects, multiple epics, and dense terminal outputs. Traditional developer interfaces either suffer from extreme tab clutter, unmovable panels that waste widescreen space, or rigid desktop-only layouts that cannot be monitored from a mobile phone on the local network.

This specification defines the AICO WebUI Shell: a high-performance React application featuring a top bar with project-scoped tabs and instant project switching, a full-height omni-search sidebar with checkbox scope filters, a dockable (Bottom/Right) Chat & Terminals panel, and a mobile-optimized responsive layout.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want the Top Bar project dropdown to switch projects while restoring each project's open tabs, so that my workspace stays clean and free of cross-project tab clutter.
   - *Independent Test*: Open 2 tabs in Project A, switch to Project B, open 1 tab, switch back to Project A, and verify Project A's 2 tabs are restored.
2. **US-02** (Priority: P1): As a developer, I want to search across Epics, Specs, Code, and ADRs with checkbox filters in the sidebar, so that I can pinpoint any artifact instantly.
   - *Independent Test*: Uncheck `Code` and `ADRs`, search for `auth`, and verify only Epic and Spec results are returned.
3. **US-03** (Priority: P1): As a developer, I want to dock the Chat and Terminal panel on the right or at the bottom with one click, so that I can optimize screen space for code diffs or wide logs.
   - *Independent Test*: Click the dock toggle button and verify the panel transitions between bottom and right layouts while preserving terminal state.
4. **US-04** (Priority: P1): As a developer, I want the left sidebar to span full height from top to bottom, so that switching the dock position never shifts or clips my primary navigation tree.
   - *Independent Test*: Toggle the bottom dock and verify the left sidebar height remains unbroken at 100% viewport height.
5. **US-05** (Priority: P2): As a mobile user, I want the UI to collapse into a single-column layout with a slide-out drawer, so that I can view streaming logs and approve commands from my phone.
   - *Independent Test*: Resize viewport to mobile width (<768px) and verify sidebar collapses to a touch drawer and chat fills the screen.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The ShellLayout shall render a top bar, a full-height left navigation sidebar, a central main content canvas, and a dockable auxiliary panel.
- **REQ-002**: The LeftSidebar shall extend unbroken across 100% of the viewport height adjacent to the top bar.
- **REQ-003**: The ProjectSwitcher shall display the active project name and render a dropdown listing all registered projects discovered by the backend.
- **REQ-004**: The TabBarManager shall scope all active tabs strictly to the currently selected project.

### Event-Driven Requirements
- **REQ-005**: When the user selects a different project in the ProjectSwitcher dropdown, the TabBarManager shall save the active tab state for the current project and load the tab state of the selected project.
- **REQ-006**: When the user right-clicks a project in the ProjectSwitcher dropdown, the ContextMenuManager shall display an option to `Open Project in New Window`.
- **REQ-007**: When the user types into the omni-search input, the SearchService shall dispatch a hybrid search query to LanceDB filtered by the active scope checkboxes (`Epics`, `Specs`, `Code`, `ADRs`).
- **REQ-008**: When the user clicks a chevron icon in the sidebar tree, the TreeViewManager shall toggle the expansion state of that node without opening a tab.
- **REQ-009**: When the user double-clicks an item label in the tree view or search results, the TabBarManager shall open that item in a new or focused internal tab.
- **REQ-010**: When the user clicks the dock position toggle button, the DockManager shall transition the auxiliary panel between the bottom position and the right-hand position.

### State-Driven Requirements
- **REQ-011**: While the auxiliary panel is docked at the bottom, the DockManager shall constrain the panel width strictly to the main canvas area between the sidebar edge and right screen edge.
- **REQ-012**: While a background task is running, the ActivityMonitor at the bottom of the sidebar shall display live CPU, task count, and progress indicators received via WebSocket.
- **REQ-013**: While the viewport width is less than 768 pixels (mobile mode), the ResponsiveViewManager shall render the LeftSidebar as an off-canvas slide-out drawer and maximize the active tab or chat view.

### Unwanted Behavior & Error Handling
- **REQ-014**: If the WebSocket connection disconnects, then the StatusBar shall display a reconnection banner and disable interactive terminal inputs until re-established.

### Optional Feature Requirements
- **REQ-015**: Where the `@aico/ui` component library is used, all UI elements shall inherit semantic design tokens for light and dark color schemes.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Tab Store Seam (`WebUI.stores.useTabStore`)**:
   - Signature: `openTab(project: string, item: TabItem)`, `switchProject(project: string)`
   - Allows unit testing tab isolation, project switching, and state restoration in React testing environments.
2. **Dock Layout Store Seam (`WebUI.stores.useDockStore`)**:
   - Signature: `setDockPosition(position: 'bottom' | 'right')`, `toggleDock()`
   - Verifies panel positioning and layout boundaries.
3. **Omni-Search Filter Seam (`WebUI.components.OmniSearch`)**:
   - Signature: `onSearch(query: string, scopes: SearchScopes)`
   - Verifies filter toggling, debouncing, and search dispatching.

---

## 5. Out of Scope

- Native operating system multi-window docking (browser windows provide multi-window capability).
- Custom desktop title bars (native browser tab strips and window frames are used).
- Full WYSIWYG rich text document editing (Markdown and EARS tables are edited via structured views).

---

## 6. Open Clarifications

None. Ratified in [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md) and [ADR-0005](../../../../adr/0005-unified-accordion-workspace-and-dockable-panels.md).
