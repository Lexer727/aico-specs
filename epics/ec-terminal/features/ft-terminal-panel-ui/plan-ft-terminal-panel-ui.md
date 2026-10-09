---
id: plan-ft-terminal-panel-ui
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 2
title: "Interactive Terminal Panel UI Implementation Plan"
status: "In Progress"
repositories: ["aico"]
derives_from: ["spec-ft-terminal-panel-ui"]
created_at: "2026-10-09T19:49:00Z"
---

# Implementation Plan: Interactive Terminal Panel UI with Multi-Shell Selector & AI Launcher

## 1. Technical Architecture & Decisions

- **Terminal Panel Header & Shell Selector (`TerminalConsole.tsx`)**:
  - Add a dedicated **"AI" Button** with `Sparkles` or `Bot` icon and purple styling (`bg-purple-900/60 hover:bg-purple-800 text-purple-300 border border-purple-700/60`), which creates a tab with `shell: 'agy'`.
  - Add a primary **Plus (`+`) Button** with title "New Terminal (PowerShell)" which creates a tab with default `shell: 'powershell'`.
  - Add a **Dropdown Trigger (`ChevronDown`)** immediately adjacent to the Plus button. Clicking opens a dropdown menu with choices:
    - `PowerShell` (icon: Terminal)
    - `Git Bash` (icon: Terminal / GitBranch)
    - `Command Prompt (cmd)` (icon: Terminal)
  - Close dropdown when clicking outside or upon option selection.
- **Real WebSocket Terminal Stream Client**:
  - Connect each terminal tab to `ws://<host>:<port>/api/v1/terminal/ws`.
  - On open: Send `{"type": "init", "shell": tab.shell, "cwd": activeProject?.codebase_path || ""}`.
  - On receiving `{"type": "output", "data": "..."}`: Append to terminal buffer.
  - On typing in input bar: Send `{"type": "input", "data": cmd + "\n"}`.
  - On closing tab: Send `{"type": "close"}` and disconnect WebSocket.
- **ANSI Color Code Parsing**:
  - Implement a lightweight ANSI escape sequence decoder converting standard SGR color codes (30-37, 90-97, 1 for bold) into formatted HTML spans with Tailwind color classes.
- **Automated Seam Tests**:
  - Component tests in `frontend/tests/TerminalConsole.test.tsx` testing the AI button, Plus button, dropdown menu, tab switching, and streaming output.

---

## 2. Task Graph (DAG)

### Phase 1: Header Actions & Multi-Shell Selector

- **TASK-TERM-002-1**: Implement AI Button, Plus Button, and Shell Dropdown Selector
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Add dedicated "AI" button with purple styling launching `agy`.
    - Add primary "+" button launching default PowerShell.
    - Add dropdown button next to "+" with selectable menu: PowerShell, Git Bash, CMD.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

### Phase 2: Live WebSocket Streaming & Session Management

- **TASK-TERM-002-2**: Implement Real WebSocket Process Client in TerminalConsole
  - **Status**: Done
  - **Addresses**: REQ-005, REQ-006
  - **Blocked by**: TASK-TERM-002-1
  - **What it delivers**:
    - Connect active terminal tabs to `/api/v1/terminal/ws`.
    - Transmit `init` with selected shell type and active project path.
    - Handle live streaming `output` messages and input forwarding.
    - Gracefully close process and socket on tab removal.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

### Phase 3: ANSI Color Rendering & Test Seams

- **TASK-TERM-002-3**: Implement ANSI Color Decoding & Interactive Viewer
  - **Status**: Done
  - **Addresses**: REQ-007
  - **Blocked by**: TASK-TERM-002-2
  - **What it delivers**:
    - Decode ANSI escape codes into colored styled elements.
    - Preserve line breaks and scroll-to-bottom behavior.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

### Phase 4: Full Suite Verification & Build

- **TASK-TERM-002-4**: Full Suite Verification and Blueprint Conformance
  - **Status**: Done
  - **Addresses**: REQ-001 through REQ-007
  - **Blocked by**: TASK-TERM-002-3
  - **What it delivers**:
    - Vitest test suite execution.
    - `verify-artifacts.mjs` verification across all 58 artifacts in `aico-specs`.
  - **Verification Seam**: Vitest + `verify-artifacts.mjs`

### Phase 5: Header Alignment & Runtime UX Remediation

- **TASK-TERM-002-5**: Header Row Consolidation & Plus Button Icon-Only
  - **Status**: Done
  - **Addresses**: REQ-009, REQ-010
  - **Blocked by**: TASK-TERM-002-4
  - **What it delivers**:
    - Place action buttons in the same row as TERMINALS.
    - Render terminal icon alongside TERMINALS label, collapsing to icon only if space is limited.
    - Render Plus button with icon and no text.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-6**: Tab Bar Styling Parity & Single-Tab Closability with Empty State
  - **Status**: Done
  - **Addresses**: REQ-008, REQ-010
  - **Blocked by**: TASK-TERM-002-5
  - **What it delivers**:
    - Style tab strip to match main view `TabBar.tsx`.
    - Allow closing single terminal tab when only 1 is open.
    - Show clean empty state when no terminals are open with quick-launch triggers.
    - Provide responsive tab label abbreviation (`PowerShell` -> `ps`).
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-7**: Interactive AGY Shell & Active Project Specs CWD Resolution
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-011
  - **Blocked by**: TASK-TERM-002-6
  - **What it delivers**:
    - Launch Antigravity CLI via interactive PowerShell session with startup banner.
    - Automatically resolve active project specs directory as default shell cwd.
  - **Verification Seam**: `crates/aico-server/tests/terminal_bridge_test.rs`

### Phase 6: Terminal Fidelity, Stream Buffering & AI Slash Autocomplete Remediation

- **TASK-TERM-002-8**: Continuous Terminal Stream Buffering & Chunk Reassembly
  - **Status**: Done
  - **Addresses**: REQ-012
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Reassemble fragmented incoming WebSocket stdout/stderr chunks across line boundaries without breaking partial words across lines.
    - Preserve ANSI styling across streaming buffer tokens.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-9**: Prompt Inline Keystroke Alignment & Duplicate Prefix Removal
  - **Status**: Done
  - **Addresses**: REQ-013
  - **Blocked by**: TASK-TERM-002-8
  - **What it delivers**:
    - Cease artificial injection of `> ${cmd}` on separate lines in history.
    - Render user keystrokes inline directly after the active prompt, allowing native shell command echo and prompt continuity.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-10**: AI Session Banner, Prompt Normalization & Slash-Command Autocomplete
  - **Status**: Done
  - **Addresses**: REQ-014
  - **Blocked by**: TASK-TERM-002-8
  - **What it delivers**:
    - Render official initialization banner on AI tab startup without prepending `agy>` to user prompts.
    - Implement interactive autocomplete dropdown suggesting available slash commands (`/help`, `/skills`, `/agents`, `/model`, `/effort`, `/plan`, `/clear`, `/exit`) when typing `/`.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

### Phase 7: TabBar Parity, Inline Typing, Session Persistence & Authentic AGY Branding

- **TASK-TERM-002-11**: TabBar Parity for Terminal Strip (Middle-Click, Drag-Drop, Overflow Chevrons)
  - **Status**: Done
  - **Addresses**: REQ-015
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Support middle mouse button click to close terminal tabs.
    - Implement drag-and-drop terminal tab reordering with live preview shifting.
    - Add horizontal scroll overflow navigation with left and right chevrons matching TabBar.tsx.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-12**: Inline Terminal Input Integration & Text Selection Retention
  - **Status**: Done
  - **Addresses**: REQ-016
  - **Blocked by**: TASK-TERM-002-11
  - **What it delivers**:
    - Move input typing directly to the active prompt in the terminal window, eliminating the duplicate bottom bar.
    - Preserve mouse text highlights on click/mouseup so text can be copied without losing selection.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-13**: Session Persistence vs Clean Boot Initialization
  - **Status**: Done
  - **Addresses**: REQ-017
  - **Blocked by**: TASK-TERM-002-11
  - **What it delivers**:
    - Initialize with zero open terminals on fresh boot, presenting the clean empty state.
    - Persist open terminal tabs across browser reloads via sessionStorage.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-14**: Authentic Antigravity CLI Branding & Responsive Panel Fitting
  - **Status**: Done
  - **Addresses**: REQ-018
  - **Blocked by**: TASK-TERM-002-12
  - **What it delivers**:
    - Render authentic Antigravity CLI header with multi-color arch branding, user account details, model parameters, divider rules, and bottom status bar.
    - Ensure terminal content fits responsive panel width without horizontal overflow scrollbars.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

### Phase 8: Native Prompt Cleanup, History Retention, Wheel Scroll Fix & Forceful Kill Action

- **TASK-TERM-002-15**: Native Prompt Input Cleanup (Remove Shell Placeholder Pre-Text)
  - **Status**: Pending
  - **Addresses**: REQ-019
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Remove placeholder pre-text (`Type command...`) for PowerShell, Git Bash, and CMD sessions, leaving only native shell prompt and cursor.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-16**: Terminal Output Buffer & Session State Persistence Across Reloads
  - **Status**: Pending
  - **Addresses**: REQ-020
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Serialize output history lines (recent buffer) and pending prompt state in `sessionStorage`.
    - Restore previous terminal buffer on page refresh without clearing screen.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-17**: Non-Passive TabBar Wheel Scroll Physics & Snap-Back Elimination
  - **Status**: Pending
  - **Addresses**: REQ-021
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Bind non-passive wheel event listeners with `e.preventDefault()` on both `TabBar.tsx` and `TerminalConsole.tsx` tab strip containers.
    - Remove `scroll-smooth` class from tab containers to eliminate animation stutter and snap-back during continuous wheel scrolling.
  - **Verification Seam**: `frontend/tests/TabBar.test.tsx`, `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-18**: Forceful Kill Terminal Action & Backend Process Tree Cleanup
  - **Status**: Pending
  - **Addresses**: REQ-022
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Add `{ type: "kill" }` protocol message in backend and child process kill on disconnect/kill command.
    - Replace "Clear terminal output" button with "Kill Terminal" button in terminal header.
    - Immediately terminate process and dispose tab from UI even when unresponsive.
  - **Verification Seam**: `crates/aico-server/tests/terminal_bridge_test.rs`, `frontend/tests/TerminalConsole.test.tsx`

---

## 3. Amendment Log

- **2026-10-09T23:12:00Z**: Added TASK-TERM-002-8, TASK-TERM-002-9, and TASK-TERM-002-10 to remediate stream chunk fragmentation, prompt/input line disconnection, and AI session UX with slash command autocomplete per REQ-012, REQ-013, and REQ-014. Reopened plan to `In Progress`.
- **2026-10-09T23:48:00Z**: Added Phase 7 (TASK-TERM-002-11 through TASK-TERM-002-14) to implement TabBar parity (middle-click, drag-drop, overflow chevrons), inline prompt typing (eliminating duplicate bottom bar), text selection retention, sessionStorage persistence vs clean boot, and authentic Antigravity CLI arch branding per REQ-015 through REQ-018. Reopened plan to `In Progress`.
- **2026-10-10T00:13:00Z**: Added Phase 8 (TASK-TERM-002-15 through TASK-TERM-002-18) to remove shell placeholder pre-text, persist terminal output history buffers across reloads, fix non-passive wheel scrolling without snap-back in TabBar and terminal tab strip, and add forceful Kill Terminal action per REQ-019 through REQ-022. Reopened plan to `In Progress`.
