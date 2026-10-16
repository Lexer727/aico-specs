---
id: spec-ft-core-webui-shell
type: spec
aico_skills_version: "0.1.2"
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

## In-Scope
- Top bar with project-scoped tabs, middle-click tab closure, instant project switching, and in-place workspace state refresh.
- Independent desktop window tab states via `sessionStorage` with multi-window isolation.
- Window-bound QR code mobile pairing modal for LAN connection.
- Mobile companion desktop window switcher dropdown and two-way tab synchronization.
- Mobile bottom navigation bar with swappable full-screen views (Tree / Explorer, Main / Home, Terminal).
- Adaptive multi-line top bar wrapping on mobile viewports.
- Dynamic viewport height (`100dvh`) and safe-area inset layout preventing bottom navigation bar clipping on physical mobile browsers.
- Sidebar with integrated real-time search, 5 scope checkboxes, and project tree view.
- Dockable panel supporting Bottom and Right docking layouts with persistent state.
- Embedded terminal consoles and live status feedback via WebSocket streaming.

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
5. **US-05** (Priority: P2): As a mobile companion user, I want a bottom navbar to swap between full-screen Tree view, Main view, and Terminal view, with an adaptive multi-line top bar that never cuts off controls, so that I can comfortably browse artifacts and run terminal commands on my phone without screen clutter.
   - *Independent Test*: Resize viewport to mobile width (<768px), verify floating drawer trigger is absent and bottom navbar is present, tap Terminal to view full-screen terminal, and tap an item in Tree to open tab and auto-switch to Main view.
6. **US-06** (Priority: P1): As a developer with multiple desktop browser tabs, I want each tab to maintain its own independent main view tabs and active tab across reloads without syncing between desktop tabs, while allowing my mobile phone to connect via QR code to a specific desktop window and switch between them.
   - *Independent Test*: Open two Chrome tabs; open a spec in Tab 1 and verify Tab 2 is unchanged; scan Tab 1 QR code on mobile and verify mobile mirrors Tab 1, then switch the mobile dropdown to Tab 2 and verify mobile mirrors Tab 2.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The ShellLayout shall render a top bar, a full-height left navigation sidebar, a central main content canvas, and a dockable auxiliary panel.
- **REQ-002**: The LeftSidebar shall extend unbroken across 100% of the viewport height adjacent to the top bar.
- **REQ-003**: The ProjectSwitcher shall display the active project name and render a dropdown listing all registered projects discovered by the backend.
- **REQ-004**: The TabBarManager shall scope all active tabs strictly to the currently selected project.
- **REQ-017**: The TabBarManager shall isolate main view tab state per browser window session using `sessionStorage`, ensuring that opening, closing, or reordering tabs in one desktop window does not mutate tab states in any other desktop window, and that tab state is preserved across page reloads within the same window.

### Event-Driven Requirements
- **REQ-005**: When the user selects a different project in the ProjectSwitcher dropdown, the TabBarManager shall save the active tab state for the current project and load the tab state of the selected project.
- **REQ-006**: When the user right-clicks a project in the ProjectSwitcher dropdown, the ContextMenuManager shall display an option to `Open Project in New Window`.
- **REQ-007**: When the user types into the omni-search input, the SearchService shall dispatch a hybrid search query to LanceDB filtered by the active scope checkboxes (`Epics`, `Specs`, `Code`, `ADRs`).
- **REQ-008**: When the user clicks a chevron icon in the sidebar tree, the TreeViewManager shall toggle the expansion state of that node without opening a tab.
- **REQ-009**: When the user double-clicks an item label in the tree view or search results, the TabBarManager shall open that item in a new or focused internal tab.
- **REQ-010**: When the user clicks the dock position toggle button, the DockManager shall transition the auxiliary panel between the bottom position and the right-hand position.
- **REQ-016**: When an AI terminal session completes a working turn, or upon browser reload (F5 / native refresh), or when programmatically requested, the MainWorkspace shall re-fetch discovered Epics, Features, ADRs, Projects, and active viewer data in-place without losing window-isolated tab state. The redundant manual top-left reload button (`refresh-workspace-btn`) is removed from the sidebar header.
- **REQ-019**: When the user clicks the QR pairing trigger (`data-testid="qr-pairing-btn"`), the TopBarManager shall display a modal with an SVG QR code and LAN pairing URL encoding the current window's `window_id` (`?token=...&win=<window_id>`); when a mobile companion connects via that pairing URL and transmits `companion_paired`, the TopBarManager shall automatically dismiss the modal.
- **REQ-021**: When the user selects an entity (epic, feature, spec, code file, ADR) from the Tree view on mobile, the ResponsiveViewManager shall open that entity in a project tab and automatically transition the active mobile view to Main / Home.

### State-Driven Requirements
- **REQ-011**: While the auxiliary panel is docked at the bottom, the DockManager shall constrain the panel width strictly to the main canvas area between the sidebar edge and right screen edge.
- **REQ-012**: While a background task is running, the ActivityMonitor at the bottom of the sidebar shall display live CPU, task count, and progress indicators received via WebSocket.
- **REQ-013**: While the viewport width is less than 768 pixels or when running in mobile mode, the ResponsiveViewManager shall render a bottom navigation bar (`data-testid="mobile-bottom-navbar"`) constrained within the dynamic viewport height (`100dvh`) with safe-area inset padding (`env(safe-area-inset-bottom)`), with three swappable full-screen view triggers: Tree / Explorer (`data-testid="mobile-tab-tree"`), Main / Home (`data-testid="mobile-tab-main"`), and Terminal Console (`data-testid="mobile-tab-terminal"`), allocating the remaining viewport height between the top bar and bottom navbar to the active view without dynamic mobile browser chrome clipping.
- **REQ-018**: While running on the desktop, each desktop window shall assign a unique `window_id` and register its presence and current tab state over the WebSocket hub (`topic: "desktop_window_register"` and `topic: "desktop_window_state"`).
- **REQ-020**: While the client is running in mobile mode, the TopBarManager shall render a custom styled desktop window switcher dropdown (`data-testid="mobile-window-switcher"`) matching the visual design, hover states, and click-outside behavior of `ProjectSwitcher`, displaying the active desktop window name/title with a dropdown chevron, and allowing the user to select and mirror any active desktop window bidirectionally without opening browser-native picker dialogs; when the user opens, closes, or switches tabs on the mobile companion, the MobileCompanionManager shall transmit `companion_tabs_update` to the targeted desktop window with mutual echo suppression to update the PC window without rollback.
- **REQ-022**: While running in mobile mode, the TopBarManager shall structure the header into a two-line layout: the top line shall position history navigation and project selection on the left, and the connection status indicator (`data-testid="connection-status-icon"`) on the far right displaying only the connection status icon without text; the second line shall position the desktop window switcher and omni-search trigger without horizontal clipping.
- **REQ-023**: While running on desktop viewports, the TopBarManager shall position the OmniSearch bar (`data-testid="header-omnisearch-btn"`) horizontally centered across the entire screen viewport (`50vw`), rendering an input-styled trigger with search icon, placeholder text, and `Ctrl+K` keyboard shortcut indicator without obstructing adjacent left navigation or right utility controls.

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

## Out-of-Scope

- Native operating system multi-window docking (browser windows provide multi-window capability).
- Custom desktop title bars (native browser tab strips and window frames are used).
- Full WYSIWYG rich text document editing (Markdown and EARS tables are edited via structured views).

---

## 6. Open Clarifications

None. Ratified in [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md) and [ADR-0005](../../../../adr/0005-unified-accordion-workspace-and-dockable-panels.md).
