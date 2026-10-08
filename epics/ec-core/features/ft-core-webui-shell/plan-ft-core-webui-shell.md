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

---

## 3. Amendment Log

- **2026-10-06T03:30:00Z**: Appended `TASK-CORE-006-11` via `/aico-plan-amendment` for tree-integrated search filtering, 5 scope checkboxes (adding Features), removing top-bar search button, and enabling middle-click tab closure.
- **2026-10-06T02:50:00Z**: Reopened plan to `In Progress` via `/aico-plan-amendment`. Added `TASK-CORE-006-8`, `TASK-CORE-006-9` (Multi-Tab Terminal System with Dynamic Terminal Creation), and `TASK-CORE-006-10` (Sidebar Settings Modal & Configuration View) to address user feedback: missing sidebar stats, inline search with persistent checkboxes, terminal renaming and multi-tab creation, and bottom-left settings.
- **2026-10-06T02:27:00Z**: Implemented Tailwind CSS integration (30.58 KB compiled stylesheet) and dynamic project discovery sync. Verified clean with 15/15 passing Vitest tests. Plan status transitioned to `Completed`.
- **2026-10-06T02:20:00Z**: Reopened plan to `In Progress` via `/aico-plan-amendment`. Added `TASK-CORE-006-6` and `TASK-CORE-006-7` after browser inspection identified that Tailwind CSS utility styles were missing from the production build, leaving the UI unstyled in default Vite placeholder CSS.
