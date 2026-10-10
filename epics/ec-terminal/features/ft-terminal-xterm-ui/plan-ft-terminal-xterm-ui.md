---
id: plan-ft-terminal-xterm-ui
type: plan
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 4
title: "Interactive xterm.js Terminal Console UI Implementation Plan"
status: "Draft"
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
  - **Status**: Ready
  - **Addresses**: REQ-001
  - **Blocked by**: none
  - **Verification Seam**: `frontend/package.json`
  - **What it delivers**:
    - Installs `@xterm/xterm` and `@xterm/addon-fit` packages and imports the required CSS styles.

- **TASK-XTERM-002**: Refactor TerminalConsole to mount xterm viewport with FitAddon
  - **Status**: Ready
  - **Addresses**: REQ-001, REQ-002, REQ-005, REQ-006
  - **Blocked by**: TASK-XTERM-001
  - **Verification Seam**: `frontend/src/components/terminal/TerminalConsole.tsx`
  - **What it delivers**:
    - Replaces custom text rendering with an authentic xterm.js terminal instance styled with AICO dark tokens.
    - Preserves all tab-strip functionality: `[AI (agy)]`, `[+]`, `[▾ Shells]`, drag-and-drop tab ordering, and dock toggles.

- **TASK-XTERM-003**: Wire bidirectional WebSocket streaming and dynamic resize listener
  - **Status**: Ready
  - **Addresses**: REQ-003, REQ-004, REQ-007, REQ-009
  - **Blocked by**: TASK-XTERM-002
  - **Verification Seam**: `frontend/src/components/terminal/TerminalConsole.tsx`
  - **What it delivers**:
    - Forwards raw user keystrokes (`term.onData`) directly to the backend PTY.
    - Writes incoming PTY output chunks into `term.write()`.
    - Dispatches `{ "type": "resize", "cols": ..., "rows": ... }` whenever the panel or viewport resizes via `ResizeObserver` and `fitAddon.fit()`.

- **TASK-XTERM-004**: Remove hardcoded emails, static model footers, and fragile HTML slash menus
  - **Status**: Ready
  - **Addresses**: REQ-008, REQ-010
  - **Blocked by**: TASK-XTERM-003
  - **Verification Seam**: `frontend/src/components/terminal/TerminalConsole.tsx`
  - **What it delivers**:
    - Removes hardcoded `a.staub1307@gmail.com`, static shortcut footers, and the buggy `bottom-full` HTML dropdown.
    - Allows the authentic CLI (AGY, Claude, etc.) to natively render its own TUI menus, model banners, and spinners.

- **TASK-XTERM-005**: Update and verify frontend unit tests in TerminalConsole.test.tsx
  - **Status**: Ready
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-005
  - **Blocked by**: TASK-XTERM-004
  - **Verification Seam**: `frontend/tests/TerminalConsole.test.tsx`
  - **What it delivers**:
    - Comprehensive Vitest suite testing xterm initialization, FitAddon execution, tab switching, and WebSocket communication.

---

## 3. Amendment History

- **2026-10-10T03:05:00Z**: Initial creation of xterm.js frontend plan under epic `ec-terminal`.
