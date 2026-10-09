---
id: spec-ft-terminal-panel-ui
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 2
title: "Interactive Terminal Panel UI with Multi-Shell Selector & AI Launcher"
status: "In Progress"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-09T19:48:00Z"
---

# Feature Specification: Interactive Terminal Panel UI with Multi-Shell Selector & AI Launcher

## 1. Problem Statement & User Value

The bottom dockable panel in the AICO WebUI shell is currently a mock simulation (`TerminalConsole.tsx`) that only responds to a fixed set of hardcoded commands. Developers cannot use real command-line tools, cannot open genuine system shells like PowerShell, Git Bash, or CMD, and cannot invoke the local Google Antigravity (`agy`) AI CLI directly from the application workspace.

This feature transforms `TerminalConsole.tsx` into a real, multi-session terminal panel housing live streaming terminals connected via WebSockets to the host runtime:
1. **Dedicated AI Button**: An AI action button (with Sparkles/Bot icon) that directly launches `agy` in the active project directory so developers can immediately use AICO skills and converse with an LLM.
2. **Plus Button & Shell Selector Dropdown**: A Plus (`+`) button to launch the default shell (PowerShell), paired with a dropdown selector allowing on-demand launching of PowerShell, Git Bash, or Command Prompt (cmd).
3. **Real Live Streaming & ANSI Rendering**: Interactive terminal sessions streaming output from real processes with ANSI color decoding and keyboard input forwarding.
4. **Tab Lifecycle Management**: Multiple concurrent terminal tabs with individual process lifecycles and clean process termination on tab close.

---

## In-Scope

- **Dedicated AI Terminal Action**:
  - Dedicated "AI" button with AI icon (`Bot` or `Sparkles`) in the terminal strip.
  - Clicking launches a new terminal tab running `agy.exe` in the active project's root.
  - Tab title styled with AI badge / purple theme (`AI (agy)`).
- **Plus Button & Shell Selector Dropdown**:
  - Plus (`+`) button launching default shell (PowerShell).
  - Dropdown button (chevron) adjacent to Plus button opening a popover menu.
  - Menu options: `PowerShell`, `Git Bash`, `Command Prompt (cmd)`.
  - Clicking an option creates a terminal tab for that shell and closes the menu.
- **WebSocket Streaming Terminal Client**:
  - Connecting to `/api/v1/terminal/ws` per terminal tab.
  - Sending `init` with selected shell type and active project path.
  - Forwarding user input keystrokes and commands via `input` messages.
  - Streaming incoming `output` messages into terminal buffer.
  - Terminating process and closing socket on tab close.
- **ANSI Color Sequence Decoding**:
  - Interpreting ANSI escape sequences for text colors (red, green, yellow, blue, magenta, cyan, bold) so CLI outputs render accurately.
- **Automated Seam Tests**:
  - Vitest component tests verifying the AI button, Plus button, shell dropdown options, tab lifecycle, and streaming display.

---

## Out-of-Scope (Future Upgrades)

- Split-pane terminal grid (tiling 4 terminals side-by-side in one tab).
- SSH remote terminal connectivity.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer wanting to chat with an AI agent using AICO skills, I want a dedicated "AI" button in the terminal panel that opens an `agy` session, so that I can immediately collaborate with the LLM in my workspace without leaving the WebUI.
   - *Independent Test*: Click the dedicated AI button; assert a new terminal tab labeled `AI (agy)` is created and connects to the backend requesting the `agy` shell in the active project directory.

2. **US-02** (Priority: P1): As a developer needing a standard command prompt, I want a Plus button that creates a PowerShell terminal, and a dropdown menu to select Git Bash or Command Prompt, so that I can choose the right shell for each task.
   - *Independent Test*: Click the Plus button; assert a PowerShell tab is opened. Click the dropdown and select `Git Bash`; assert a Git Bash tab is opened.

3. **US-03** (Priority: P1): As a developer running tools with colored output, I want ANSI color codes to render properly in the terminal panel, so that build errors and CLI prompts are clearly readable.
   - *Independent Test*: Feed ANSI-colored output strings into the terminal component; assert rendered elements possess appropriate color styling.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: In the dockable terminal panel, the application shall provide a dedicated "AI" action button with an AI/Sparkle icon that, when clicked, creates a new terminal tab running the `agy` CLI assistant in the active project directory.
- **REQ-002**: In the dockable terminal panel, the application shall provide a primary Plus (`+`) button that, when clicked, creates a new terminal tab running the default shell (`powershell`).
- **REQ-003**: Immediately adjacent to the Plus button, the application shall render a dropdown trigger button that, when clicked, displays a menu listing selectable shell types (`PowerShell`, `Git Bash`, `Command Prompt`).
- **REQ-004**: When a user selects a shell type from the dropdown menu, the application shall create a new terminal tab running the selected shell and dismiss the dropdown menu.
- **REQ-005**: While a terminal tab is active, the component shall maintain a WebSocket connection to `/api/v1/terminal/ws`, displaying received `output` data in the terminal window and forwarding user keystrokes as `input` messages.
- **REQ-006**: When a terminal tab is closed by the user, the application shall send a `close` message, close the WebSocket connection, and remove the tab from the panel.
- **REQ-007**: The terminal output viewer shall decode standard ANSI color escape sequences, rendering formatted colored text in the terminal buffer.
- **REQ-008**: The dockable terminal panel shall allow closing terminal tabs even when only one terminal is open, transitioning the view to an empty state with quick actions to launch a session.
- **REQ-009**: In the dockable terminal panel header row, the application shall render the `TERMINALS` label and terminal icon, collapsing to the icon only on constrained viewports, with action buttons positioned in the same row.
- **REQ-010**: The primary Plus button shall display an icon without text, and terminal tabs shall display responsive label abbreviations when space is limited (`PowerShell` -> `ps`).
- **REQ-011**: Terminal sessions shall default their execution directory to the specs directory of the active project when no working directory is explicitly provided.
- **REQ-012**: The terminal viewer shall reassemble incoming streaming stdout and stderr chunks continuously without breaking partial words or token fragments across separate lines.
- **REQ-013**: In interactive terminal sessions (e.g., PowerShell), user typing shall render inline directly behind the active shell prompt without injecting an artificial standalone `>` command line into the output history.
- **REQ-014**: The AI (`agy`) terminal session shall display the initialization banner upon launch, omit the `agy>` prefix from the prompt and user input, and provide an interactive autocomplete menu when the user types a leading `/` slash command.
- **REQ-015**: The terminal tabs strip shall support middle-click to close tabs, drag-and-drop tab reordering with live preview shifting, and overflow scroll navigation chevrons, matching the interaction standards of the primary workspace TabBar.
- **REQ-016**: The terminal panel shall allow user typing directly at the active terminal prompt without a duplicate bottom input bar, and clicking or releasing text selections in the terminal window shall preserve highlighted text for copying.
- **REQ-017**: On clean fresh boot, the terminal panel shall initialize with zero open terminals in the empty state; during an active session, open terminal tabs shall persist across page reloads.
- **REQ-018**: The AI (`agy`) terminal session shall render the authentic Antigravity CLI header with multi-color arch branding, user account details, model parameters, divider separators, and bottom status shortcuts without horizontal overflow scrollbars.

---

## 4. Designated Public Test Seams

- **Terminal Panel Component Seam**: `frontend/tests/TerminalConsole.test.tsx` (verifies REQ-001 through REQ-010, REQ-012 through REQ-018)
- **Terminal Bridge Seam**: `crates/aico-server/tests/terminal_bridge_test.rs` (verifies REQ-011)
