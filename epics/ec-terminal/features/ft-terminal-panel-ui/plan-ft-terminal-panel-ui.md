---
id: plan-ft-terminal-panel-ui
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 2
title: "Interactive Terminal Panel UI Implementation Plan"
status: "Completed"
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
    - `verify-artifacts.mjs` verification across all 57 artifacts in `aico-specs`.
  - **Verification Seam**: Vitest + `verify-artifacts.mjs`
