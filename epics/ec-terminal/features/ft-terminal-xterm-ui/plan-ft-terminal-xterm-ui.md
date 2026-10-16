---
id: plan-ft-terminal-xterm-ui
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 4
title: "Interactive xterm.js Terminal Console UI Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-terminal-xterm-ui"]
created_at: "2026-10-10T03:05:00Z"
---

# Implementation Plan: Interactive xterm.js Terminal Console UI

## 1. Executive Summary & Verification Strategy

This plan replaces the custom HTML input and ANSI regex parser in `TerminalConsole.tsx` with a high-performance `@xterm/xterm` emulator equipped with `@xterm/addon-fit`. It preserves the full multi-tab interface, shell selector, drag-drop reordering, and dock controls, while eliminating the slash menu positioning bug and all hardcoded user strings. Verification is ensured via Vitest in `frontend/tests/TerminalConsole.test.tsx`.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

- **TASK-XTERM-001**: Add @xterm/xterm and @xterm/addon-fit dependencies to frontend
  - **Status**: Done
  - **Addresses**: REQ-001
  - **Blocked by**: none
  - **Verification Seam**: `frontend/package.json`
  - **What it delivers**:
    - Installs `@xterm/xterm` and `@xterm/addon-fit` packages and imports the required CSS styles.

- **TASK-XTERM-002**: Refactor TerminalConsole to mount xterm viewport with FitAddon
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-005, REQ-006
  - **Blocked by**: TASK-XTERM-001
  - **Verification Seam**: `frontend/src/components/terminal/TerminalConsole.tsx`
  - **What it delivers**:
    - Replaces custom text rendering with an authentic xterm.js terminal instance styled with AICO dark tokens.
    - Preserves all tab-strip functionality: `[AI (agy)]`, `[+]`, `[▾ Shells]`, drag-and-drop tab ordering, and dock toggles.

- **TASK-XTERM-003**: Wire bidirectional WebSocket streaming and dynamic resize listener
  - **Status**: Done
  - **Addresses**: REQ-003, REQ-004, REQ-007, REQ-009
  - **Blocked by**: TASK-XTERM-002
  - **Verification Seam**: `frontend/src/components/terminal/TerminalConsole.tsx`
  - **What it delivers**:
    - Forwards raw user keystrokes (`term.onData`) directly to the backend PTY.
    - Writes incoming PTY output chunks into `term.write()`.
    - Dispatches `{ "type": "resize", "cols": ..., "rows": ... }` whenever the panel or viewport resizes via `ResizeObserver` and `fitAddon.fit()`.

- **TASK-XTERM-004**: Remove hardcoded emails, static model footers, and fragile HTML slash menus
  - **Status**: Done
  - **Addresses**: REQ-008, REQ-010
  - **Blocked by**: TASK-XTERM-003
  - **Verification Seam**: `frontend/src/components/terminal/TerminalConsole.tsx`
  - **What it delivers**:
    - Removes hardcoded `a.staub1307@gmail.com`, static shortcut footers, and the buggy `bottom-full` HTML dropdown.
    - Allows the authentic CLI (AGY, Claude, etc.) to natively render its own TUI menus, model banners, and spinners.

- **TASK-XTERM-005**: Update and verify frontend unit tests in TerminalConsole.test.tsx
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005
  - **Blocked by**: TASK-XTERM-004
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Comprehensive Vitest suite testing xterm initialization, FitAddon execution, tab switching, and WebSocket communication.

- **TASK-XTERM-006**: Replace text AI button with icon-only Antigravity (`agy-terminal-btn`) and Claude (`claude-terminal-btn`) launcher buttons and matching tab branding
  - **Status**: Done
  - **Addresses**: REQ-002
  - **Blocked by**: TASK-XTERM-005
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Replaces the text `AI` button with an icon-only Antigravity button (`data-testid="agy-terminal-btn"`, Antigravity multi-color arch SVG icon and colorway) that opens a tab labeled `Antigravity` (`AGY`).
    - Adds an icon-only Claude button (`data-testid="claude-terminal-btn"`, Claude sunburst SVG icon and terracotta colorway) that opens a tab labeled `Claude` (`claude`).
    - Updates `TerminalConsole.test.tsx` to verify both icon-only buttons and tab creation.

- **TASK-XTERM-007**: Implement per-tab 4-state circular AI status indicator and hover tooltip in TerminalConsole
  - **Status**: Done
  - **Addresses**: REQ-011
  - **Blocked by**: TASK-XTERM-006
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Renders a per-tab circular status indicator (`data-testid="ai-tab-status-<tabId>"`) inside AI tabs (`agy` and `claude`): orange spinning circle (`working`), blue bouncing circle (`waiting_for_input`), green static circle (`finished`), and red static circle (`error` / not logged in / out of tokens).
    - Ignores local user keystroke echoes before `Enter` (`\r`) submission so typing a prompt never triggers `working`.
    - Sets descriptive short hover text (`title`) on the AI tab header and status circle.

- **TASK-XTERM-008**: Enforce singleton tab limit per AI CLI (`agy` and `claude`) while keeping standard shell tabs unlimited
  - **Status**: Done
  - **Addresses**: REQ-012
  - **Blocked by**: TASK-XTERM-007
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Updates `handleAddTab` (and `getInitialTabs`) in `TerminalConsole.tsx` so at most one `Antigravity` (`agy`) tab and one `Claude` (`claude`) tab can be open concurrently, focusing the existing AI tab if clicked again while leaving standard shells (`powershell`, `bash`, `cmd`) unlimited.
    - Adds unit test coverage in `frontend/tests/TerminalConsole.test.tsx`.

- **TASK-XTERM-009**: Implement selection-aware `Ctrl+C` copy, `Ctrl+V` clipboard paste, and `tab_id` session re-attachment in TerminalConsole
  - **Status**: Done
  - **Addresses**: REQ-013, REQ-014
  - **Blocked by**: TASK-XTERM-008
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Attaches `term.attachCustomKeyEventHandler` in `TerminalConsole.tsx` so `Ctrl+C` / `Ctrl+Shift+C` copies selected text (`term.getSelection()`) to `navigator.clipboard.writeText`, clears selection, and blocks `\x03` when text is selected, while forwarding `\x03` (`SIGINT`) when no text is selected; and `Ctrl+V` / `Ctrl+Shift+V` reads `navigator.clipboard.readText()` and calls `term.paste(text)` while suppressing `\x16`.
    - Passes `tab_id` in the WebSocket `init` message and preserves backend PTY sessions on unmount/reload (`F5`), sending `close`/`kill` only on explicit tab closure or termination.
    - Invokes `onWorkspaceRefresh?.()` when an AI terminal transitions from `working` to `finished`.
    - Adds unit tests in `frontend/tests/TerminalConsole.test.tsx`.

- **TASK-XTERM-010**: Implement companion terminal tabs synchronization for mobile clients
  - **Status**: Done
  - **Addresses**: REQ-015
  - **Blocked by**: TASK-XTERM-009
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - On desktop windows, broadcasts terminal tabs and active terminal tab over `/ws` (`topic: "companion_terminals_sync"`, payload: `{ window_id, tabs, activeTabId }`).
    - On mobile companion clients, listens for `companion_terminals_sync` for the currently selected `window_id`, mounting the same terminal tabs and active console.
    - Automatically routes mobile terminal tabs to the shared `tab_id` PTY sessions so mobile and desktop interact with the same live console.
    - Adds unit test coverage in `frontend/tests/TerminalConsole.test.tsx`.

- **TASK-XTERM-011**: Terminal Tab Periodic Heartbeat & Connection Sync
  - **Status**: Done
  - **Addresses**: REQ-015
  - **Blocked by**: TASK-XTERM-010
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Adds a periodic 3-second heartbeat broadcast for `companion_terminals_sync` on desktop windows, ensuring companion connections arriving after initial boot reliably receive the terminal tab state.
    - Triggers `request_companion_terminals` from mobile companion when WebSocket status transitions to `connected` or when `targetWindowId` switches.
    - Synchronizes mobile tab creations and active tab selections back to the desktop window via `companion_terminals_update`.

- **TASK-XTERM-012**: Mobile Terminal Kill Synchronization & Desktop ConPTY Disposal
  - **Status**: Done
  - **Addresses**: REQ-015
  - **Blocked by**: TASK-XTERM-011
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Dispatches `companion_terminals_update` with `killTabId` from mobile when the user clicks the Kill Terminal button.
    - Handles `killTabId` in desktop `companion_terminals_update` listener to call `disposeTabResources(payload.killTabId, 'kill')`, terminating the ConPTY process and removing the tab on both devices.
    - Ensures touch-friendly sizing (`min-w-[28px] min-h-[28px]`, `type="button"`) on `kill-terminal-btn`.

---

## 3. Amendment History

- **2026-10-17T00:15:00Z**: Added `TASK-XTERM-012` (`REQ-015`) via `/aico-plan-amendment` for mobile kill terminal propagation to desktop ConPTY process.
- **2026-10-16T23:20:00Z**: Added `TASK-XTERM-011` (`REQ-015`) via `/aico-plan-amendment` for periodic 3-second desktop heartbeat broadcast, connection-triggered companion terminal requests, and mobile-to-desktop active tab routing.
- **2026-10-16T22:30:00Z**: Added `TASK-XTERM-010` (`REQ-015`) via `/aico-plan-amendment` for mobile companion terminal tabs synchronization with the targeted desktop window.
- **2026-10-10T03:05:00Z**: Initial creation of xterm.js frontend plan under epic `ec-terminal`.
- **2026-10-12T19:56:00Z**: Added `TASK-XTERM-006` (`REQ-002`) for icon-only Antigravity (`agy-terminal-btn`) and Claude (`claude-terminal-btn`) buttons and tab branding.
- **2026-10-12T20:15:00Z**: Added `TASK-XTERM-007` (`REQ-011`) for per-tab 4-state circular AI status indicators with hover tooltips and user-typing echo exclusion.
- **2026-10-12T21:18:00Z**: Added `TASK-XTERM-008` (`REQ-012`) to enforce at most one tab per AI CLI (`agy` and `claude`) while keeping standard shell tabs unlimited.
- **2026-10-16T20:58:00Z**: Added `TASK-XTERM-009` (`REQ-013`, `REQ-014`) for selection-aware `Ctrl+C` copy, `Ctrl+V` clipboard paste, and `tab_id` session re-attachment across browser reloads.
