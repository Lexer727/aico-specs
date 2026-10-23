---
id: plan-ft-core-webui-shell
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 6
title: "WebUI Application Shell, Navigation & Dockable Layout"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-webui-shell"]
created_at: "2026-10-05T20:00:00Z"
---

# Implementation Plan: WebUI Application Shell, Navigation & Dockable Layout


---

## 1. Technical Architecture & Decisions

- **Package**: `frontend/` (React + Vite + Tailwind CSS + Lucide Icons).
- **Core Dependencies**:
  - `react` / `react-dom` (v18+): UI component framework.
  - `zustand`: Lightweight client state management for tab state, docking configuration, and project context.
  - `tailwind-merge` & `clsx`: Dynamic Tailwind utility styling.
  - `@radix-ui/react-accordion`: Accessible, keyboard-navigable collapsible workspace tree.
  - `cmdk`: Fast command palette engine for Omni-Search (`Ctrl+K`).
- **Layout Architecture**: Strict CSS Grid / Flexbox layout ensuring the left sidebar occupies 100vh full height, and the main canvas area encapsulates the main content and dockable bottom/right panel without overflow or occlusion.

---

## 2. Task Graph (DAG)

### Phase 1: WebUI Shell & Accordion Sidebar (Unblocked Frontier)

- **TASK-CORE-006-1**: App Shell Container & Full-Height Accordion Sidebar
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-008`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Scaffolds `frontend/` using Vite + React + Tailwind.
    - Implements `ShellLayout` with full-height 100vh left sidebar.
    - Implements `AccordionWorkspace` displaying active project, Epics, Specs, Plans, and Code files.
    - Adds responsive mobile breakpoint collapsing sidebar into a slide-over drawer with hamburger trigger.
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx::renders_full_height_sidebar_and_responsive_drawer`

---

### Phase 2: Dockable Panel System

- **TASK-CORE-006-2**: Dockable Bottom/Right Panel with Canvas-Bounded Sizing
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-004`
  - **Blocked by**: `TASK-CORE-006-1`
  - **What it delivers**:
    - Implements `DockablePanel` supporting `dockPosition: 'bottom' | 'right'`.
    - Guarantees `bottom` dock is constrained strictly within the main canvas area (right of the 100% height sidebar).
    - Adds one-click dock toggle button in panel header with smooth CSS transitions.
    - Persists user dock position and height/width in `localStorage`.
  - **Verification Seam**: `frontend/tests/DockablePanel.test.tsx::toggles_dock_position_and_respects_sidebar_boundary`

---

### Phase 3: Project-Scoped Tabs & Multi-Window Sync

- **TASK-CORE-006-3**: Project-Scoped Tab Bar & Multi-Window Sync
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-CORE-006-1`
  - **What it delivers**:
    - Implements `TabBar` tracking open tabs per `projectId` via Zustand.
    - Preserves and restores open tabs when switching projects in the top selector.
    - Implements "Open in New Window" action launching `?project=<id>&window=detached`.
    - Synchronizes tab updates and active state across windows using `BroadcastChannel`.
  - **Verification Seam**: `frontend/tests/TabBar.test.tsx::manages_project_scoped_tabs_and_multi_window_broadcast`

---

### Phase 4: Omni-Search Modal (`Ctrl+K`)

- **TASK-CORE-006-4**: Omni-Search Modal & Keyboard Navigation
  - **Status**: Done
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-CORE-006-1`
  - **What it delivers**:
    - Implements `OmniSearchModal` triggered globally by `Ctrl+K` (or `Cmd+K`).
    - Provides grouped instant search results: Projects, Epics, Specifications, Plans, and Source Files.
    - Supports keyboard up/down arrow navigation, Enter to open, and Esc to dismiss.
  - **Verification Seam**: `frontend/tests/OmniSearchModal.test.tsx::opens_on_shortcut_and_filters_results`

---

### Phase 5: WebSocket Client & Live Connection Status

- **TASK-CORE-006-5**: WebSocket Client Provider & Reconnect Banner
  - **Status**: Done
  - **Addresses**: `REQ-009`
  - **Blocked by**: `TASK-CORE-006-1`, `TASK-CORE-001-3` (WebSocket Hub)
  - **What it delivers**:
    - Implements `WebSocketProvider` handling `/ws` connection with exponential backoff retry.
    - Renders status pill (Connected / Reconnecting / Offline) in the top navigation bar.
    - Displays non-intrusive notification banner during connection loss with manual reconnect button.
  - **Verification Seam**: `frontend/tests/WebSocketProvider.test.tsx::handles_connection_loss_and_reconnection`

---

### Phase 6: UI Styling Restoration & Live Workspace Binding (Amended Frontier)

- **TASK-CORE-006-6**: Tailwind CSS Integration & Layout Theme Restoration
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-015`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Installs `@tailwindcss/vite` and `tailwindcss` in `frontend/package.json`.
    - Configures Tailwind Vite plugin in `frontend/vite.config.ts`.
    - Replaces Vite template styling in `frontend/src/index.css` with `@import "tailwindcss";` and defines dark theme root styling.
    - Rebuilds frontend bundle (`npm run build`) and verifies compiled CSS bundles all utility classes (`bg-zinc-900`, `flex`, `border-zinc-800`, etc.).
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`

- **TASK-CORE-006-7**: Dynamic Project Discovery & Active Workspace Binding
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-005`
  - **Blocked by**: `TASK-CORE-006-6`
  - **What it delivers**:
    - Replaces static `mockProjects` in `App.tsx` and `ProjectSwitcher.tsx` with dynamic fetching from `GET /api/v1/projects`.
    - Hooks project dropdown selection to `POST /api/v1/projects/active`.
    - Updates active workspace tabs and state upon receiving `active_project_changed` WebSocket events.
  - **Verification Seam**: `frontend/tests/ProjectSwitcher.test.tsx`

---

### Phase 7: Sidebar UX, Activity Metrics & Multi-Terminal Workspaces (Amended Frontier)

- **TASK-CORE-006-8**: Sidebar Integrated Search, Persistent Checkbox Scopes & Activity Monitor
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-007`, `REQ-012`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Mounts persistent search input directly inside the left navigation sidebar.
    - Exposes 4 persistent scope filter checkboxes (`Epics`, `Specs`, `Code`, `ADRs`) that immediately filter the visible items upon toggle or input.
    - Implements `ActivityMonitor` pinned to the bottom of the sidebar displaying live CPU usage %, active tasks count, and WebSocket sync indicators.
  - **Verification Seam**: `frontend/tests/SidebarSearch.test.tsx`

- **TASK-CORE-006-9**: Multi-Tab Terminal System with Dynamic Terminal Creation
  - **Status**: Done
  - **Addresses**: `REQ-010`, `REQ-011`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Renames dockable auxiliary panel to "Terminals".
    - Adds tabbed terminal bar with multiple terminal sessions (`Terminal 1`, `Terminal 2`, `+ New Terminal`) with tab switching and closure.
    - Provides interactive command input with execution history, clearing, and live status.
  - **Verification Seam**: `frontend/tests/TerminalPanel.test.tsx`

- **TASK-CORE-006-10**: Sidebar Settings Modal & Configuration View
  - **Status**: Done
  - **Addresses**: `REQ-015`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Adds a Settings button in the bottom-left footer of the sidebar.
    - Displays modal showing active host server configuration, pairing token QR, and environment version details.
  - **Verification Seam**: `frontend/tests/SettingsModal.test.tsx`

- **TASK-CORE-006-11**: Tree-Integrated Search, 5-Scope Filter Checkboxes & Middle-Click Tab Close
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-007`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Replaces flat search results with in-tree filtering: keeps the accordion tree structure and filters child nodes in-place under visible sections.
    - Hides/shows entire accordion tree sections based on 5 filter checkboxes (`Epics`, `Features`, `Specs`, `Code`, `ADRs`).
    - Removes redundant top bar search button.
    - Implements middle mouse click on open tabs to close them (protecting Dashboard tab from closure).
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`, `frontend/tests/TabBar.test.tsx`

- **TASK-CORE-006-12**: Add Workspace State Refresh Button & In-Place Data Reload
  - **Status**: Done
  - **Addresses**: `REQ-016`
  - **Blocked by**: `TASK-CORE-006-11`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`
  - **What it delivers**:
    - Renders a Workspace Refresh button (`data-testid="refresh-workspace-btn"`) in the WebUI header and sidebar header.
    - Re-fetches discovered Projects, Epics, Features, ADRs, and active viewer data (`SpecViewer`, `PlanViewer`, `DocumentViewer`, `AdrViewer`) in-place without reloading the browser page.
    - Automatically triggers workspace refresh when an AI terminal session finishes a working turn.

---

### Phase 8: Window Session Isolation, QR Code Pairing & Mobile Companion Sync (Amended Frontier)

- **TASK-CORE-006-13**: Window-Isolated sessionStorage & Multi-Window Desktop Decoupling
  - **Status**: Done
  - **Addresses**: `REQ-017`
  - **Blocked by**: `TASK-CORE-006-12`
  - **Verification Seam**: `frontend/tests/TabBar.test.tsx`
  - **What it delivers**:
    - Migrates `useTabStore.ts` persistence from `localStorage` to `sessionStorage` (`aico_tab_store_session_v1`).
    - Eliminates the desktop-to-desktop `BroadcastChannel('aico_tabs_sync')` so that each desktop Chrome window maintains independent open tabs and active tab, surviving `F5` reload.
    - Assigns and preserves a unique `window_id` per window session in `sessionStorage`.

- **TASK-CORE-006-14**: Window-Bound QR Code Pairing Modal & Auth Integration
  - **Status**: Done
  - **Addresses**: `REQ-019`
  - **Blocked by**: `TASK-CORE-006-13`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`
  - **What it delivers**:
    - Exposes `GET /api/v1/auth/pairing` in `aico-server` returning the host's LAN IP, port, token, and pairing URL.
    - Adds a QR pairing button (`data-testid="qr-pairing-btn"`) to the desktop WebUI top header.
    - Renders a Mobile Pairing modal displaying an SVG QR code encoding `http://<LAN-IP>:4242/?token=<token>&win=<window_id>`, direct pairing URL, and copy button.

- **TASK-CORE-006-15**: Mobile Companion Window Switcher & Two-Way Main View Synchronization
  - **Status**: Done
  - **Addresses**: `REQ-018`, `REQ-020`
  - **Blocked by**: `TASK-CORE-006-14`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`
  - **What it delivers**:
    - Desktop windows register presence and broadcast main view tabs over `/ws` (`desktop_window_register`, `desktop_window_state`).
    - When running in mobile mode, renders a desktop window switcher dropdown (`data-testid="mobile-window-switcher"`) next to the project switcher.
    - Initial connection binds to the `win` query parameter from the scanned QR code (or defaults to the first active window).
    - Selecting another window switches the mirrored main view tabs, and mobile navigation/tab mutations synchronize bidirectionally with the selected desktop window.

- **TASK-CORE-006-16**: Mobile-to-Desktop Two-Way Tab Sync & Cyclical Echo Suppression
  - **Status**: Done
  - **Addresses**: `REQ-020`
  - **Blocked by**: `TASK-CORE-006-15`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Dispatches `companion_tabs_update` from mobile companion to `targetWindowId` whenever tabs or active tab change on mobile.
    - Implements mutual `isRemoteSyncRef` guards on both mobile and desktop to prevent cyclical feedback loops and ensure tabs opened on mobile remain open on both mobile and desktop.

- **TASK-CORE-006-17**: QR Code Pairing Modal Auto-Dismiss on Companion Connection
  - **Status**: Done
  - **Addresses**: `REQ-019`
  - **Blocked by**: `TASK-CORE-006-16`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Emits `companion_paired` message over WebSocket when mobile companion connects with a `win` query parameter.
    - Listens for `companion_paired` in the desktop window and automatically closes `<MobilePairingModal>`.

---

### Phase 9: Mobile Polish — Bottom Navigation Bar & Adaptive Header (Amended Frontier)

- **TASK-CORE-006-18**: Mobile Bottom Navigation Bar & Swappable Full-Screen Views
  - **Status**: Done
  - **Addresses**: `REQ-013`, `REQ-021`
  - **Blocked by**: `TASK-CORE-006-16`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`
  - **What it delivers**:
    - Replaces the top-left floating drawer toggle button with a fixed bottom navigation bar (`data-testid="mobile-bottom-navbar"`) offering three swappable full-screen views: `Tree / Explorer` (`data-testid="mobile-tab-tree"`), `Main / Home` (`data-testid="mobile-tab-main"`), and `Terminal` (`data-testid="mobile-tab-terminal"`).
    - Swaps 100% of the viewport height between the top bar and bottom navbar to the active view.
    - When an entity (epic, feature, spec, code, ADR) is selected from the Tree view on mobile, opens the tab and automatically transitions the active view to `Main / Home`.

- **TASK-CORE-006-19**: Adaptive Multi-Line Mobile Top Bar & Header Refactor
  - **Status**: Done
  - **Addresses**: `REQ-022`
  - **Blocked by**: `TASK-CORE-006-18`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Converts the top `<header>` in `App.tsx` from fixed `h-12` to adaptive `min-h-12 h-auto py-1.5 flex flex-wrap items-center justify-between gap-y-1.5 gap-x-2 px-3`.
    - Allows `HistoryNavButtons`, `ProjectSwitcher`, `mobile-window-switcher`, `OmniSearch`, and `ConnectionStatusBanner` to wrap seamlessly without clipping or horizontal overflow on narrow mobile screens.

- **TASK-CORE-006-20**: Test Seam Coverage for Mobile Bottom Navbar & Adaptive Header
  - **Status**: Done
  - **Addresses**: `REQ-013`, `REQ-021`, `REQ-022`
  - **Blocked by**: `TASK-CORE-006-19`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`, `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Verifies presence of bottom navbar and absence of floating drawer trigger on mobile viewports.
    - Tests view switching between Tree, Main, and Terminal.
    - Tests auto-transition from Tree to Main upon entity selection.
    - Tests top header flex-wrapping on mobile.

---

### Phase 10: Mobile Dynamic Viewport & Safe-Area Inset Polish (Amended Frontier)

- **TASK-CORE-006-21**: Dynamic Viewport (`100dvh`) & Safe-Area Inset Mobile Shell Refactor
  - **Status**: Done
  - **Addresses**: `REQ-013`
  - **Blocked by**: `TASK-CORE-006-20`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`
  - **What it delivers**:
    - Updates `frontend/index.html` to add `viewport-fit=cover` to the viewport meta tag.
    - Updates `frontend/src/index.css` to declare `height: 100dvh;` on `html, body, #root`.
    - Updates `frontend/src/components/layout/ShellLayout.tsx` to replace `h-screen w-screen` with `h-[100dvh] w-full max-w-full`.
    - Replaces `h-full` on `<main>` with `min-h-0 flex-1` inside the vertical flex column to eliminate flex basis overflow.
    - Updates `<nav data-testid="mobile-bottom-navbar">` to use `h-[calc(3.5rem+env(safe-area-inset-bottom,0px))] pb-[env(safe-area-inset-bottom,0px)] shrink-0` to protect against physical mobile browser toolbar and gesture pill clipping.

- **TASK-CORE-006-22**: Vitest Seam Test Suite & End-to-End Build Verification
  - **Status**: Done
  - **Addresses**: `REQ-013`
  - **Blocked by**: `TASK-CORE-006-21`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`, `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Updates `frontend/tests/ShellLayout.test.tsx` to verify dynamic viewport height (`h-[100dvh]`) and safe-area inset classes on mobile bottom navbar.
    - Runs full Vitest suite (`npm test`) and Rust backend suite (`cargo test -p aico-server`).
    - Compiles production frontend bundle (`npm run build`) and restarts `aico-server` background daemon.

---

### Phase 11: Mobile Companion Top Bar Refactor & Terminal Kill Synchronization (Amended Frontier)

- **TASK-CORE-006-23**: Custom Styled Mobile Desktop Window Switcher
  - **Status**: Done
  - **Addresses**: `REQ-020`
  - **Blocked by**: `TASK-CORE-006-22`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Creates `DesktopWindowSwitcher.tsx` matching `ProjectSwitcher.tsx` with a styled trigger button, `Smartphone` icon, active tab title, `ChevronDown`, and floating `z-50` dropdown menu with outside-click dismissal.
    - Replaces the raw `<select>` in `App.tsx` with `<DesktopWindowSwitcher>`, eliminating mobile-native picker wheels.

- **TASK-CORE-006-24**: Mobile Two-Line Header with Far-Right Icon-Only Status
  - **Status**: Done
  - **Addresses**: `REQ-022`
  - **Blocked by**: `TASK-CORE-006-23`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Adds `iconOnly?: boolean` prop to `ConnectionStatusBanner.tsx` displaying only the status icon/badge without text.
    - Structures the mobile header in `App.tsx` into two distinct rows:
      - Line 1: Left = `<HistoryNavButtons>` + `<ProjectSwitcher>`; Far Right = `<ConnectionStatusBanner iconOnly={true} />`.
      - Line 2: Left = `<DesktopWindowSwitcher>`; Right = OmniSearch button.

- **TASK-CORE-006-25**: Mobile Terminal Kill Propagation & Desktop ConPTY Disposal
  - **Status**: Done
  - **Addresses**: `REQ-015` (in `spec-ft-terminal-xterm-ui.md`)
  - **Blocked by**: `TASK-CORE-006-24`
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - In `TerminalConsole.tsx`, updates `handleKillTerminal` to dispatch `companion_terminals_update` with `killTabId: targetId` when on mobile.
    - Updates desktop message handler to invoke `disposeTabResources(payload.killTabId, 'kill')` and update `tabs` state upon receiving `killTabId`.
    - Expands touch target on `<button data-testid="kill-terminal-btn">` to minimum 28x28px (`p-1.5`) with explicit `type="button"`.

- **TASK-CORE-006-26**: Vitest Seam Test Suite & End-to-End Build Verification
  - **Status**: Done
  - **Addresses**: `REQ-020`, `REQ-022`, `REQ-015`
  - **Blocked by**: `TASK-CORE-006-25`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`, `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Updates test suites verifying custom window switcher dropdown interactions, mobile two-line header alignment, and mobile terminal kill propagation.
    - Runs full Vitest suite (`npm test`) and Rust backend suite (`cargo test -p aico-server`).
    - Compiles production bundle (`npm run build`) and restarts `aico-server` daemon.

---

### Phase 12: Desktop Top Bar Center OmniSearch & Sidebar Header Polish (Amended Frontier)

- **TASK-CORE-006-27**: Desktop Top Bar 3-Column Layout with Centered OmniSearch
  - **Status**: Done
  - **Addresses**: `REQ-023`
  - **Blocked by**: `TASK-CORE-006-26`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Restructures desktop top header in `frontend/src/App.tsx` into a 3-zone layout (left navigation group, center OmniSearch bar, right pairing and connection status).
    - Centers the OmniSearch trigger (`data-testid="header-omnisearch-btn"`) with expanded width (`max-w-md`), placeholder text ("Search workspace, epics, specs..."), search icon, and `Ctrl+K` shortcut pill.

- **TASK-CORE-006-28**: Remove Redundant Sidebar Header Reload Button
  - **Status**: Done
  - **Addresses**: `REQ-016`
  - **Blocked by**: `TASK-CORE-006-27`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`
  - **What it delivers**:
    - Removes `<button data-testid="refresh-workspace-btn">` from the sidebar header in `frontend/src/components/layout/ShellLayout.tsx`.
    - Retains `AICO WORKSPACE` pulse indicator and `v0.1.1` version badge.
    - Updates `frontend/tests/ShellLayout.test.tsx` test seam to verify the button is absent.

- **TASK-CORE-006-29**: Vitest Seam Test Suite & End-to-End Build Verification
  - **Status**: Done
  - **Addresses**: `REQ-016`, `REQ-023`
  - **Blocked by**: `TASK-CORE-006-28`
  - **Verification Seam**: `frontend/tests/ShellLayout.test.tsx`, `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Verifies centered OmniSearch bar in desktop header and removal of redundant reload button in Vitest test suites.
    - Runs full Vitest suite (`npm test`) and Rust backend suite (`cargo test -p aico-server`).
    - Compiles production bundle (`npm run build`) and restarts `aico-server` daemon.

---

### Phase 13: Desktop OmniSearch Global Viewport Centering (Amended Frontier)

- **TASK-CORE-006-30**: Global Screen-Centered Desktop OmniSearch Bar Layout
  - **Status**: Done
  - **Addresses**: `REQ-023`
  - **Blocked by**: `TASK-CORE-006-29`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Restructures desktop `<header>` in `frontend/src/App.tsx` by wrapping the desktop OmniSearch bar trigger (`data-testid="header-omnisearch-btn"`) in a full-viewport centered overlay container (`hidden md:flex fixed top-0 inset-x-0 h-12 items-center justify-center pointer-events-none z-20 px-4`).
    - Gives the search button `pointer-events-auto`, centering the trigger at `50vw` across the physical screen rather than merely inside the main view canvas.
    - Preserves `relative z-20` on left and right header control groups separated by `<div className="flex-1" />`.

- **TASK-CORE-006-31**: Vitest Seam Test Suite & End-to-End Build Verification
  - **Status**: Done
  - **Addresses**: `REQ-023`
  - **Blocked by**: `TASK-CORE-006-30`
  - **Verification Seam**: `frontend/tests/MobilePairingAndSync.test.tsx`
  - **What it delivers**:
    - Updates Vitest tests in `frontend/tests/MobilePairingAndSync.test.tsx` to verify global viewport centering styling and interaction.
    - Runs full Vitest suite (`npm test`) and Rust backend suite (`cargo test -p aico-server`).
    - Compiles production bundle (`npm run build`) and restarts `aico-server` daemon.

- **TASK-CORE-006-32**: Implement unified isTaskDone helper and integrate across frontend views
  - **Status**: Done
  - **Addresses**: `REQ-024`
  - **Blocked by**: none
  - **Verification Seam**: `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Exposes `isTaskDone(status: string)` in `frontend/src/features/feature-task-data.ts` treating `Done`, `Completed`, `Rewritten`, and `Skipped/Canceled` as terminal completed states.
    - Updates `EpicHubView.tsx`, `FeatureViewer.tsx`, `PlanViewer.tsx`, `DashboardView.tsx`, and `computeActiveFrontier` to use `isTaskDone`.

- **TASK-CORE-006-33**: Add Vitest unit test coverage for Rewritten task status progress calculation
  - **Status**: Done
  - **Addresses**: `REQ-024`
  - **Blocked by**: `TASK-CORE-006-32`
  - **Verification Seam**: `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Adds automated unit tests in `frontend/tests/FeatureViewer.test.tsx` asserting that tasks with status `Rewritten` or `Skipped/Canceled` count toward done progress tallies and unblock dependent tasks in the DAG.

- **TASK-CORE-006-34**: Harmonize EpicHubView child feature task progress, status badges, and blocker resolution with isTaskDone
  - **Status**: Done
  - **Addresses**: `REQ-024`
  - **Blocked by**: `TASK-CORE-006-32`
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`
  - **What it delivers**:
    - Updates `EpicHubView.tsx` child feature task group calculations to use `isTaskDone(t)` for `featDone` count.
    - Updates task badge color styling in child feature cards to use `isTaskDone(task.status)`.
    - Updates blocker resolution checks in `blockedList.every` and dependency badge formatting to use `isTaskDone(blocker)`.

- **TASK-CORE-006-35**: Add Vitest unit test coverage for EpicHubView child feature Rewritten progress and blocker resolution
  - **Status**: Done
  - **Addresses**: `REQ-024`
  - **Blocked by**: `TASK-CORE-006-34`
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`
  - **What it delivers**:
    - Adds automated unit tests in `frontend/tests/EpicHubView.test.tsx` asserting that child features containing `Rewritten` tasks (such as `ft-core-mcp-tools`) display full completion progress (e.g. `4/4 Done (100%)`), display emerald completed badge styling, and resolve dependency blockers.

---

## 3. Amendment Log

- **2026-10-23T00:36:00Z**: Appended Phase 15 tasks (`TASK-CORE-006-34`, `TASK-CORE-006-35` for `REQ-024`) via `/aico-plan-amendment` to harmonize child feature task counts, status badge styling, and blocker resolution with `isTaskDone` in `EpicHubView`.

- **2026-10-23T00:12:00Z**: Appended Phase 14 tasks (`TASK-CORE-006-32`, `TASK-CORE-006-33` for `REQ-024`) via `/aico-plan-amendment` to treat `Rewritten` and `Skipped/Canceled` tasks as terminal completed states across all task plan progress roll-ups and active frontier DAG calculations.

- **2026-10-17T00:48:00Z**: Appended Phase 13 tasks (`TASK-CORE-006-30`, `TASK-CORE-006-31` for `REQ-023`) via `/aico-plan-amendment` to position the desktop OmniSearch bar horizontally centered across the entire physical display viewport (`50vw`) using a full-viewport overlay container.
- **2026-10-17T00:27:00Z**: Appended Phase 12 tasks (`TASK-CORE-006-27`, `TASK-CORE-006-28`, `TASK-CORE-006-29` for `REQ-023`, `REQ-016`) via `/aico-plan-amendment` for desktop top bar 3-column layout with centered OmniSearch and removal of redundant sidebar header reload button.
- **2026-10-17T00:15:00Z**: Appended Phase 11 tasks (`TASK-CORE-006-23`, `TASK-CORE-006-24`, `TASK-CORE-006-25`, `TASK-CORE-006-26` for `REQ-020`, `REQ-022`, and `REQ-015`) via `/aico-plan-amendment` for custom styled mobile desktop window switcher, two-line mobile header with far-right icon-only connection status, and mobile terminal kill propagation to desktop ConPTY process.
- **2026-10-17T00:03:00Z**: Appended Phase 10 tasks (`TASK-CORE-006-21`, `TASK-CORE-006-22` for `REQ-013`) via `/aico-plan-amendment` to refactor mobile layout to dynamic viewport height (`100dvh`) and safe-area insets (`env(safe-area-inset-bottom)`), resolving physical mobile Chrome toolbar clipping.
- **2026-10-16T23:45:00Z**: Appended Phase 9 tasks (`TASK-CORE-006-18`, `TASK-CORE-006-19`, `TASK-CORE-006-20` for `REQ-013`, `REQ-021`, `REQ-022`) via `/aico-plan-amendment` for mobile bottom navigation bar, swappable full-screen views (Tree, Main, Terminal), and adaptive multi-line top bar header.
- **2026-10-16T23:20:00Z**: Added `TASK-CORE-006-16` (`REQ-020`) and `TASK-CORE-006-17` (`REQ-019`) via `/aico-plan-amendment` for mobile-to-desktop two-way tab synchronization with echo suppression and QR pairing modal auto-dismiss upon companion connection.
- **2026-10-16T22:30:00Z**: Appended Phase 8 tasks (`TASK-CORE-006-13`, `TASK-CORE-006-14`, `TASK-CORE-006-15` for `REQ-017` through `REQ-020`) via `/aico-plan-amendment` for window-isolated `sessionStorage`, window-bound QR pairing, and mobile companion window switcher with two-way sync.
- **2026-10-16T20:58:00Z**: Appended `TASK-CORE-006-12` (`REQ-016`) via `/aico-plan-amendment` for in-place Workspace State Refresh button and automatic refresh on AI terminal completion.
- **2026-10-06T03:30:00Z**: Appended `TASK-CORE-006-11` via `/aico-plan-amendment` for tree-integrated search filtering, 5 scope checkboxes (adding Features), removing top-bar search button, and enabling middle-click tab closure.
- **2026-10-06T02:50:00Z**: Reopened plan to `In Progress` via `/aico-plan-amendment`. Added `TASK-CORE-006-8`, `TASK-CORE-006-9` (Multi-Tab Terminal System with Dynamic Terminal Creation), and `TASK-CORE-006-10` (Sidebar Settings Modal & Configuration View) to address user feedback: missing sidebar stats, inline search with persistent checkboxes, terminal renaming and multi-tab creation, and bottom-left settings.
- **2026-10-06T02:27:00Z**: Implemented Tailwind CSS integration (30.58 KB compiled stylesheet) and dynamic project discovery sync. Verified clean with 15/15 passing Vitest tests. Plan status transitioned to `Completed`.
- **2026-10-06T02:20:00Z**: Reopened plan to `In Progress` via `/aico-plan-amendment`. Added `TASK-CORE-006-6` and `TASK-CORE-006-7` after browser inspection identified that Tailwind CSS utility styles were missing from the production build, leaving the UI unstyled in default Vite placeholder CSS.
