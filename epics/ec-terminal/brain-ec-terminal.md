---
id: brain-ec-terminal
type: brainstorm
aico_skills_version: "0.1.2"
epic: "ec-terminal"
title: "Interactive Terminal Subsystem & Multi-Shell Runtime"
status: "Completed"
derives_from: []
created_at: "2026-10-09T19:45:00Z"
---

# Brainstorm: Interactive Terminal Subsystem & Multi-Shell Runtime

## 1. Executive Summary & Problem Statement

The AICO WebUI shell currently features a bottom dockable panel labeled "Terminals", but its contents consist of a simulated mock console (`TerminalConsole.tsx`). This mock console intercepts commands with a hardcoded `switch` statement (`help`, `status`, `epics`, `test`) and prints pre-canned text rather than executing commands in a real shell environment.

This presents several critical limitations for developer workflows:
1. **Mock Shell Limitation**: Developers cannot run real commands, compilers (`cargo`, `npm`), Git operations, or script tools directly from the AICO environment.
2. **Missing Local AI Integration**: To run Google Antigravity (`agy`) with AICO skills, developers must leave the application and open an external terminal window manually.
3. **No Multi-Shell Flexibility**: Developers have no way to choose between different shell environments on their workstation (PowerShell, Git Bash, Command Prompt) within the WebUI.

**Epic Vision (`ec-terminal`)**:
This epic transforms the dockable terminal panel into an interactive terminal environment:
- **Real Backend Process Bridge**: An Axum WebSocket endpoint (`/api/v1/terminal/ws`) spawning real operating system processes (`powershell.exe`, `cmd.exe`, Git `bash.exe`, and `agy.exe`) with bidirectional stream I/O.
- **Dedicated AI Terminal Action**: A dedicated "AI" trigger button (with AI icon) that instantly launches an interactive `agy` CLI session running in the active project directory, pre-loaded with AICO skills to chat with an LLM and execute actions.
- **Multi-Shell Launcher & Selector**: A primary Plus (`+`) button to launch the default shell (PowerShell), alongside a dropdown selector allowing on-demand launching of PowerShell, Git Bash, or Command Prompt (cmd).
- **Embedded Web Terminal**: Interactive terminal emulation inside the dockable panel with ANSI escape sequence styling, scrollback buffer, tab lifecycle management, and external window popout action.

---

## 2. Architectural Architecture & Component Seams

```
┌─────────────────────────────────────────────────────────────┐
│                       AICO WebUI Shell                      │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ DockablePanel: Terminals                              │  │
│  │ ┌───────────────────────────────────────────────────┐ │  │
│  │ │ [AI (agy)] [+] [▾ Shells] | [Term 1] [Term 2] [X] │ │  │
│  │ ├───────────────────────────────────────────────────┤ │  │
│  │ │ Real Interactive Terminal Stream                  │ │  │
│  │ │ (ANSI Colors, Keystroke forwarding, Cursor)       │ │  │
│  │ └─────────────────────────┬─────────────────────────┘ │  │
│  └───────────────────────────┼───────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────┘
                               │ WebSocket (/api/v1/terminal/ws)
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    aico-server (Rust / Axum)                │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ TerminalManager (Session Registry)                    │  │
│  │  - Spawns child process with piped stdin/stdout/stderr│  │
│  │  - Streams stdout/stderr chunks -> WS client          │  │
│  │  - Forwards WS client input chunks -> process stdin   │  │
│  │  - Manages graceful shutdown on session drop/close    │  │
│  └──────┬──────────────┬──────────────┬─────────────┬────┘  │
└─────────┼──────────────┼──────────────┼─────────────┼───────┘
          ▼              ▼              ▼             ▼
      agy.exe      powershell.exe   bash.exe       cmd.exe
   (AI Assistant)   (Default PS)   (Git Bash)   (Command Prompt)
```

---

## 3. Supported Shell Environments & Paths

On Windows workstations:
1. **AI Terminal (`agy`)**:
   - Command: `agy.exe`
   - Working Directory: Active project codebase or specs root.
   - Purpose: Direct interactive agent session utilizing AICO skills.
2. **PowerShell (Default Shell)**:
   - Command: `powershell.exe -NoLogo`
   - Working Directory: Active project root.
3. **Git Bash**:
   - Command: `C:\Program Files\Git\bin\bash.exe` (or `bash.exe` on PATH) with `--login -i`.
   - Working Directory: Active project root.
4. **Command Prompt (CMD)**:
   - Command: `cmd.exe`
   - Working Directory: Active project root.

---

## 4. Child Feature Decomposition

This epic decomposes into five vertical child features across two architectural generations:

### Generation 1: Initial Process Streaming & Panel Scaffold (Foundational)
1. **`ft-terminal-backend-stream-bridge`** (`order: 1`):
   - **Scope**: Backend Rust Axum WebSocket endpoint (`/api/v1/terminal/ws`), process execution and streaming bridge, session registry, stdin writing, stdout/stderr chunk streaming, and process termination on tab close or client disconnect.
   - **Contract**: WebSocket JSON protocol (`{"type": "init", "shell": "powershell" | "agy" | "bash" | "cmd", "cwd": "..."}`, `{"type": "input", "data": "..."}`, `{"type": "output", "data": "..."}`).

2. **`ft-terminal-panel-ui`** (`order: 2`):
   - **Scope**: Frontend `TerminalConsole.tsx` replacement housing real streaming terminal sessions inside `DockablePanel`.
   - Dedicated "AI" button launching `agy` CLI in active project directory.
   - Plus (`+`) button launching default shell (PowerShell).
   - Shell dropdown menu next to Plus button for selecting PowerShell, Git Bash, or Command Prompt.
   - ANSI color rendering, scrollback buffer, tab creation/closing, and native window popout trigger.

### Generation 2: Native ConPTY Runtime, xterm.js Engine & Multi-Provider AI Quota Monitor (Active Evolution)
3. **`ft-terminal-conpty-bridge`** (`order: 3`):
   - **Scope**: Backend native Pseudo-Terminal (PTY) engine powered by `portable-pty` (Windows ConPTY and Unix PTY). Full TUI support, raw keystrokes without line-buffering delay, dynamic resize (`PtySize`), and clean process tree lifecycle management.

4. **`ft-terminal-xterm-ui`** (`order: 4`):
   - **Scope**: Industry-standard `@xterm/xterm` engine with `@xterm/addon-fit` embedded in `TerminalConsole.tsx`. Fixes the slash menu positioning bug, removes hardcoded emails and shortcuts, and provides 100% VT100/ANSI terminal emulation with interactive prompt selection.

5. **`ft-terminal-ai-quota-monitor`** (`order: 5`):
   - **Scope**: Backend dynamic quota discovery service (`/api/v1/ai/quotas`) and compact 2-line `AiStatusMonitor.tsx` in the sidebar. Displays Gemini Weekly and 5-Hour remaining limits on line 1, Claude/secondary providers on line 2 (only if authenticated), and real-time AI terminal state (`Inactive`, `Waiting for Input`, `Working`).

6. **`ft-terminal-tab-title-status`** (`order: 6`):
   - **Scope**: Reactive synchronization between active AI terminal sessions and the browser window title (`document.title`). Formats ambient status indicators with animated braille spinners during active work (`working`), green circles (`🟢`) when finished, blue circles (`🔵`) when awaiting user input, and red circles (`🔴`) on error—ordered with Antigravity first and Claude second, collapsing cleanly to `AICO` when no AI CLI terminals are open.

---

## 5. Milestone Convergence & Verification Seams

1. **Backend ConPTY Bridge Seam**: `crates/aico-server/tests/conpty_bridge_test.rs`
   - Validates ConPTY process spawning, raw byte streaming, dynamic resizing, and process termination on Windows.
2. **xterm Console Component Seam**: `frontend/tests/TerminalConsole.test.tsx`
   - Validates xterm.js instance initialization, FitAddon resizing, tab switching, and WebSocket communication.
3. **AI Quota API Seam**: `crates/aico-server/tests/ai_quota_test.rs`
   - Validates `/api/v1/ai/quotas` endpoint, dynamic CLI credential discovery, and omission of unauthenticated providers.
4. **AI Status Monitor UI Seam**: `frontend/tests/AiStatusMonitor.test.tsx`
   - Validates compact 2-line layout, circular progress meters, and AI activity badge transitions.
5. **Terminal Title Status Seam**: `frontend/tests/TerminalTitleStatus.test.tsx`
   - Validates dynamic `document.title` synchronization with terminal tabs and AI status states, braille spinner animation loop, and cleanup on unmount.

