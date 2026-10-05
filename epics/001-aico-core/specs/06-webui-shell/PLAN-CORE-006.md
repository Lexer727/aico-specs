# Implementation Plan: WebUI Application Shell, Navigation & Dockable Layout

- **ID**: PLAN-CORE-006
- **Spec Reference**: [SPEC-CORE-006.md](file:///d:/aico/aico-specs/epics/001-aico-core/specs/06-webui-shell/SPEC-CORE-006.md)
- **Status**: Ready

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

- [ ] **TASK-CORE-006-1**: App Shell Container & Full-Height Accordion Sidebar
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

- [ ] **TASK-CORE-006-2**: Dockable Bottom/Right Panel with Canvas-Bounded Sizing
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

- [ ] **TASK-CORE-006-3**: Project-Scoped Tab Bar & Multi-Window Sync
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

- [ ] **TASK-CORE-006-4**: Omni-Search Modal & Keyboard Navigation
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-CORE-006-1`
  - **What it delivers**:
    - Implements `OmniSearchModal` triggered globally by `Ctrl+K` (or `Cmd+K`).
    - Provides grouped instant search results: Projects, Epics, Specifications, Plans, and Source Files.
    - Supports keyboard up/down arrow navigation, Enter to open, and Esc to dismiss.
  - **Verification Seam**: `frontend/tests/OmniSearchModal.test.tsx::opens_on_shortcut_and_filters_results`

---

### Phase 5: WebSocket Client & Live Connection Status

- [ ] **TASK-CORE-006-5**: WebSocket Client Provider & Reconnect Banner
  - **Addresses**: `REQ-009`
  - **Blocked by**: `TASK-CORE-006-1`, `TASK-CORE-001-3` (WebSocket Hub)
  - **What it delivers**:
    - Implements `WebSocketProvider` handling `/ws` connection with exponential backoff retry.
    - Renders status pill (Connected / Reconnecting / Offline) in the top navigation bar.
    - Displays non-intrusive notification banner during connection loss with manual reconnect button.
  - **Verification Seam**: `frontend/tests/WebSocketProvider.test.tsx::handles_connection_loss_and_reconnection`

---

## 3. Initial Active Frontier

- `TASK-CORE-006-1`: App Shell Container & Full-Height Accordion Sidebar (Ready for `/aico-test-scaffolding`).
