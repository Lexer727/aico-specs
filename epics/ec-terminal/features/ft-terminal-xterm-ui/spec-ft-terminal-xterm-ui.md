---
id: spec-ft-terminal-xterm-ui
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 4
title: "Interactive xterm.js Terminal Console UI & Shell Tab Integration"
status: "Completed"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-10T03:05:00Z"
---

# Feature Specification: Interactive xterm.js Terminal Console UI & Shell Tab Integration

## 1. Problem Statement & User Value

The existing terminal panel UI in AICO (`TerminalConsole.tsx`) simulated a terminal by parsing ANSI escape codes with regular expressions and capturing input via a standard HTML `<input>` tag at the bottom. This approach has critical UX and functional flaws:
1. **Interactive Prompt Breakdown**: Arrow keys, cursor positioning, and single-keystroke listeners cannot be captured by a text input, making interactive CLI selections fail.
2. **Slash Menu Positioning Bug**: When the session has few output lines, the input prompt is at the top of the container. The slash command menu styled with `bottom-full` was pushed above the top of the container, rendering it completely unreadable.
3. **Hardcoded Metadata**: Shortcuts (`? for shortcuts`), models (`Gemini 3.8 Flash · high`), and user accounts (`a.staub1307@gmail.com`) were hardcoded in JSX rather than delivered by the CLI runtime.

This feature embeds the industry-standard [`@xterm/xterm`](https://github.com/xtermjs/xterm.js) engine with `@xterm/addon-fit` into the AICO dockable panel. This delivers full terminal emulation, native interactive menu rendering, bidirectional raw keystroke transmission, responsive panel resizing, and removes all hardcoded text.

---

## In-Scope

- **Native Terminal Emulation (`@xterm/xterm`)**:
  - Embedding `@xterm/xterm` with the `@xterm/addon-fit` addon into `TerminalConsole.tsx`.
  - Dark theme styling matching the AICO design system with monospace font rendering and cursor blink.
  - Full ANSI/VT100 support: colors, cursor addressing, line clear, and mouse selection.
  - Selection-aware `Ctrl+C` (`Ctrl+Shift+C`) clipboard copy vs. `\x03` (`SIGINT`) interrupt and `Ctrl+V` (`Ctrl+Shift+V`) clipboard paste across all shells.
- **WebSocket Protocol Integration**:
  - Piping raw output chunks from `/api/v1/terminal/ws` directly to `term.write()`.
  - Capturing raw user keystrokes with `term.onData()` and sending them to the backend PTY.
  - Automatically recalculating dimensions on panel resize with `fitAddon.fit()` and sending `resize` messages to ConPTY.
  - Passing `tab_id` on `init` to re-attach live PTY sessions and replay scrollback across page reloads (`F5`).
- **Tab Strip Parity & Session Management**:
  - Dedicated icon-only **Antigravity** (`data-testid="agy-terminal-btn"`) and **Claude** (`data-testid="claude-terminal-btn"`) launcher buttons with authentic brand icons and colorways, launching singleton tabs labeled `Antigravity` (`AGY`) and `Claude`.
  - `[+]` button launching the default shell (PowerShell) and dropdown menu for Git Bash and Command Prompt (supporting unlimited concurrent shell tabs).
  - Tab lifecycle: tab creation, switching, middle-click close, drag-and-drop reordering, and overflow scroll chevrons.
  - Mobile companion terminal tabs synchronization with the targeted desktop window.
- **Removal of Mock Overlays**:
  - Removing hardcoded user email, static model footers, and fragile HTML slash menus (all rendered natively by the CLI inside the terminal).

---

## Out-of-Scope

- Standalone desktop multi-windowing (browser popup and dockable layout satisfy all needs).
- Native OS terminal tabs outside the browser shell.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer interacting with `agy` or `claude`, I want the terminal to render interactive multiple-choice lists with arrow navigation, so that I can select options without prompts failing or auto-skipping.
   - *Independent Test*: Open an AI terminal session; assert xterm.js instance captures arrow keystrokes and renders TUI changes accurately.

2. **US-02** (Priority: P1): As a developer resizing the terminal drawer or docking it to the right, I want the terminal to automatically resize its buffer columns and rows, so that the display fits the viewport without distortion.
   - *Independent Test*: Trigger dock position change; verify `fitAddon.fit()` is invoked and a WebSocket `resize` payload is transmitted.

3. **US-03** (Priority: P1): As a developer managing multiple shells, I want to open multiple PowerShell or Git Bash tabs concurrently while keeping a single Antigravity and a single Claude tab, so that I can switch between them without opening duplicate AI sessions.
   - *Independent Test*: Click Antigravity and Claude buttons multiple times; assert only one tab per AI CLI is created, while clicking `+` opens multiple PowerShell tabs.

4. **US-04** (Priority: P1): As a developer using a mobile companion, I want the terminal tabs and active console on my mobile device to mirror the desktop window I selected, so that I can see identical shells and outputs on both screens.
   - *Independent Test*: Open two terminal tabs on Desktop Window A; connect Mobile Companion to Window A; assert Mobile Companion displays the same two terminal tabs and active console.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The TerminalConsole shall embed an authentic `@xterm/xterm` viewport instance equipped with the `@xterm/addon-fit` addon for each active terminal tab.
- **REQ-002**: The TerminalTabStrip shall render dedicated icon-only Antigravity (`data-testid="agy-terminal-btn"`, with Antigravity multi-color arch icon and colorway opening a tab labeled `Antigravity` / `AGY`) and Claude (`data-testid="claude-terminal-btn"`, with Claude sunburst icon and terracotta colorway opening a tab labeled `Claude`) launcher buttons alongside the `[+]` new tab button and shell dropdown selector menu above the xterm viewport.

### Event-Driven Requirements
- **REQ-003**: When the user presses any key inside the focused xterm viewport, the TerminalWebSocketClient shall forward the raw character sequence directly to the backend PTY via `/api/v1/terminal/ws`.
- **REQ-004**: When the terminal panel, dock position, or browser window dimensions change, the ResponsiveFitManager shall invoke `fitAddon.fit()` and emit a WebSocket `resize` message containing the updated `cols` and `rows`.
- **REQ-005**: When the user middle-clicks a tab header or clicks the close icon, the TabManager shall close the tab and notify the backend to terminate the session.
- **REQ-006**: When the user drags a tab header horizontally, the TabBarManager shall reorder the tabs with live visual preview.
- **REQ-012**: When the user clicks the Antigravity (`agy-terminal-btn`) or Claude (`claude-terminal-btn`) launcher button while a terminal tab for that AI CLI (`agy` or `claude`) is already open, the TabManager shall activate and focus the existing tab for that AI CLI without opening a duplicate tab, whereas standard shell types (`powershell`, `bash`, `cmd`) shall allow unlimited concurrent tabs.
- **REQ-013**: When the user presses `Ctrl+C` (or `Ctrl+Shift+C`) inside a focused xterm viewport, the TerminalConsole shall copy the active selection to the clipboard (`navigator.clipboard.writeText`), clear the selection, and suppress `\x03` transmission if `term.hasSelection()` is true, or allow `\x03` (`SIGINT`) transmission if no text is selected; when the user presses `Ctrl+V` (or `Ctrl+Shift+V`), the TerminalConsole shall read text from `navigator.clipboard.readText()` and paste it into the terminal via `term.paste()` while suppressing raw `\x16` transmission.
- **REQ-014**: When initializing a terminal WebSocket connection, the TerminalConsole shall include `tab_id` in the `init` message and preserve the backend PTY session on component unmount or page reload (`F5`), sending `close` or `kill` only on explicit user tab close or kill actions.
- **REQ-015**: When connected to a targeted desktop window, the TerminalConsole on mobile companion clients shall synchronize terminal tab definitions and the active terminal tab with that window via the WebSocket hub (`topic: "companion_terminals_sync"`), supported by a periodic 3-second heartbeat broadcast on desktop and an on-connect request from mobile, allowing the mobile companion to view, add, and interact with the selected desktop window's terminal sessions. When the user clicks the Kill Terminal button (`data-testid="kill-terminal-btn"`), the TerminalConsole shall transmit `companion_terminals_update` with `killTabId` to the targeted desktop window, and the targeted desktop window shall invoke `disposeTabResources(killTabId, 'kill')` to terminate the ConPTY process and remove the tab without heartbeat resurrection.

### State-Driven Requirements
- **REQ-007**: While the backend PTY streams output data over WebSocket, the TerminalConsole shall feed the chunks directly into `term.write()` to render VT100 escape sequences, colors, and cursor movements.
- **REQ-008**: While an AI CLI session is active, the xterm viewport shall display the CLI's authentic built-in banner, model indicators, and thinking animations without mock HTML overlays.
- **REQ-011**: While an AI terminal tab (`agy` or `claude`) is open in the TerminalTabStrip, the tab header shall render a per-tab circular status indicator (`data-testid="ai-tab-status-<tabId>"`) with a short status description on tab hover (`title`) and without treating local user keystroke echoes before `Enter` submission as AI activity:
  - **`working`** (Orange spinning circle): When the LLM is actively working, thinking, or streaming output after prompt submission.
  - **`waiting_for_input`** (Blue bouncing circle): When the AI agent prompts the user for permission, confirmation, or a multiple-choice selection.
  - **`finished`** (Green static circle): When the AI session has completed its output and is ready at the prompt for the next instruction.
  - **`error`** (Red static circle): When the AI CLI is not logged in, out of tokens (`remaining_pct == 0` / rate limit), or encounters a session/connection error.

### Unwanted Behavior & Error Handling
- **REQ-009**: If the WebSocket connection disconnects unexpectedly, then the TerminalConsole shall render a discreet disconnected banner and disable keypress transmission until re-established.
- **REQ-010**: The TerminalConsole shall not render hardcoded user emails, static shortcut hints, or mock model footers.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Terminal Console Component Seam (`frontend/tests/TerminalConsole.test.tsx`)**:
   - Signature: Validates xterm instantiation, FitAddon integration, tab creation, tab switching, and WebSocket message dispatch.
   - Verifies 100% component fidelity, tab strip actions, and cleanup on unmount.

---

## Out-of-Scope

- Custom font file downloading (standard monospace font stacks are used).
- Terminal session recording to video.
