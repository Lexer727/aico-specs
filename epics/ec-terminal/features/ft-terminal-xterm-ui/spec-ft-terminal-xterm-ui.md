---
id: spec-ft-terminal-xterm-ui
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 4
title: "Interactive xterm.js Terminal Console UI & Shell Tab Integration"
status: "Draft"
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
- **WebSocket Protocol Integration**:
  - Piping raw output chunks from `/api/v1/terminal/ws` directly to `term.write()`.
  - Capturing raw user keystrokes with `term.onData()` and sending them to the backend PTY.
  - Automatically recalculating dimensions on panel resize with `fitAddon.fit()` and sending `resize` messages to ConPTY.
- **Tab Strip Parity & Session Management**:
  - Dedicated `[AI (agy)]` launcher button with authentic sparkle icon.
  - `[+]` button launching the default shell (PowerShell) and dropdown menu for Git Bash and Command Prompt.
  - Tab lifecycle: tab creation, switching, middle-click close, drag-and-drop reordering, and overflow scroll chevrons.
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

3. **US-03** (Priority: P1): As a developer managing multiple shells, I want to open PowerShell, Git Bash, and AI tabs concurrently, so that I can switch between them without losing running processes.
   - *Independent Test*: Create two terminal tabs; assert switching between tabs displays respective session buffers.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The TerminalConsole shall embed an authentic `@xterm/xterm` viewport instance equipped with the `@xterm/addon-fit` addon for each active terminal tab.
- **REQ-002**: The TerminalTabStrip shall render the dedicated `[AI (agy)]` button, the `[+]` new tab button, and the shell dropdown selector menu above the xterm viewport.

### Event-Driven Requirements
- **REQ-003**: When the user presses any key inside the focused xterm viewport, the TerminalWebSocketClient shall forward the raw character sequence directly to the backend PTY via `/api/v1/terminal/ws`.
- **REQ-004**: When the terminal panel, dock position, or browser window dimensions change, the ResponsiveFitManager shall invoke `fitAddon.fit()` and emit a WebSocket `resize` message containing the updated `cols` and `rows`.
- **REQ-005**: When the user middle-clicks a tab header or clicks the close icon, the TabManager shall close the tab and notify the backend to terminate the session.
- **REQ-006**: When the user drags a tab header horizontally, the TabBarManager shall reorder the tabs with live visual preview.

### State-Driven Requirements
- **REQ-007**: While the backend PTY streams output data over WebSocket, the TerminalConsole shall feed the chunks directly into `term.write()` to render VT100 escape sequences, colors, and cursor movements.
- **REQ-008**: While an AI CLI session is active, the xterm viewport shall display the CLI's authentic built-in banner, model indicators, and thinking animations without mock HTML overlays.

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
