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
  - **Status**: Pending
  - **Addresses**: REQ-012
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Reassemble fragmented incoming WebSocket stdout/stderr chunks across line boundaries without breaking partial words across lines.
    - Preserve ANSI styling across streaming buffer tokens.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-9**: Prompt Inline Keystroke Alignment & Duplicate Prefix Removal
  - **Status**: Pending
  - **Addresses**: REQ-013
  - **Blocked by**: TASK-TERM-002-8
  - **What it delivers**:
    - Cease artificial injection of `> ${cmd}` on separate lines in history.
    - Render user keystrokes inline directly after the active prompt, allowing native shell command echo and prompt continuity.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

- **TASK-TERM-002-10**: AI Session Banner, Prompt Normalization & Slash-Command Autocomplete
  - **Status**: Pending
  - **Addresses**: REQ-014
  - **Blocked by**: TASK-TERM-002-8
  - **What it delivers**:
    - Render official initialization banner on AI tab startup without prepending `agy>` to user prompts.
    - Implement interactive autocomplete dropdown suggesting available slash commands (`/help`, `/skills`, `/agents`, `/model`, `/effort`, `/plan`, `/clear`, `/exit`) when typing `/`.
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`

---

## 3. Amendment Log

- **2026-10-09T23:12:00Z**: Added TASK-TERM-002-8, TASK-TERM-002-9, and TASK-TERM-002-10 to remediate stream chunk fragmentation, prompt/input line disconnection, and AI session UX with slash command autocomplete per REQ-012, REQ-013, and REQ-014. Reopened plan to `In Progress`.
