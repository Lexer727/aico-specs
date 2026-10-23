---
id: spec-ft-terminal-tab-title-status
type: spec
aico_skills_version: "0.1.2"
epic: "ec-terminal"
order: 6
title: "Browser Tab Title Terminal AI Status Indicator"
status: "Completed"
derives_from: ["brain-ec-terminal"]
created_at: "2026-10-22T21:53:38.036Z"
---

# Feature Specification: Browser Tab Title Terminal AI Status Indicator

## 1. Problem Statement & User Value

Currently, the browser tab title in AICO is statically defined as `<title>frontend</title>` in `frontend/index.html`. When developers work across multiple browser tabs, windows, or desktop applications while an AI agent (Google Antigravity `agy` or Claude Code `claude`) executes tasks in the terminal panel, there is no ambient visibility into execution progress. Developers are forced to constantly switch back to the AICO window to see if the AI agent is still working, has finished, is blocked waiting for input, or encountered an error.

This feature introduces reactive synchronization between active AI terminal sessions and the browser window title (`document.title`). Whenever an AI CLI terminal is open, the title dynamically displays ambient status circle indicators with rotating braille spinners during active work, green circles when finished, blue bouncing/waiting circles when awaiting user input, and red circles on failure—ordered with Antigravity first and Claude second—and cleanly collapsing to `AICO` when no AI CLI terminals are open.

---

## In-Scope

- **Static Baseline Branding**:
  - Updating the static title in `frontend/index.html` from `frontend` to `AICO`.
- **Reactive Title Synchronization**:
  - Ambient title synchronization hook (`useTerminalTitleSync`) observing terminal session tabs and `aiTabStatuses` in `TerminalConsole.tsx`.
  - Defaulting to `AICO` whenever no AI CLI terminal is present.
- **Dynamic Glyph Mapping**:
  - `working`: Rotating Unicode braille spinner sequence (`⠋`, `⠙`, `⠹`, `⠸`, `⠼`, `⠴`, `⠦`, `⠧`, `⠇`, `⠏`) advancing at an ~80ms interval.
  - `finished`: Static green circle (`🟢`).
  - `waiting_for_input`: Static blue circle (`🔵`).
  - `error`: Static red circle (`🔴`).
- **Multi-Terminal Sequencing**:
  - Single AI terminal: `AICO <glyph>` (e.g. `AICO 🟢`).
  - Dual AI terminals: `AICO <agy_glyph> <claude_glyph>` (e.g. `AICO ⠋ 🟢`).
  - Omission of standard non-AI shells (PowerShell, Git Bash, Command Prompt).
- **Lifecycle & Timer Cleanup**:
  - Automatic timer creation when one or more AI sessions enter `working`.
  - Immediate timer disposal and static title restoration when sessions stop working, tabs close, or the component unmounts.
- **Public Test Seam**:
  - Comprehensive unit and integration test suite in `frontend/tests/TerminalTitleStatus.test.tsx`.

---

## Out-of-Scope

- Ambient indicator rendering for non-AI shells (`powershell`, `bash`, `cmd`).
- Audio alerts, chime synthesis, or desktop push notifications.
- Animated favicon replacement (scoped strictly to `document.title`).
- Custom user-configurable emoji strings.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer multitasking across different desktop apps while Antigravity generates code, I want the browser tab title to show a rotating spinner during execution and a green circle when finished, so that I know exactly when my prompt has completed without tab-switching.
   - *Independent Test*: Open an Antigravity terminal session; trigger a prompt; assert `document.title` displays `AICO <spinner_char>`; advance timers past completion; assert title updates to `AICO 🟢`.

2. **US-02** (Priority: P1): As a developer running Antigravity and Claude Code concurrently, I want both status circles visible in the browser tab title in fixed priority order (`AICO <agy> <claude>`), so that I can monitor both AI agents in parallel.
   - *Independent Test*: Open both `agy` and `claude` sessions; set `agy` to `working` and `claude` to `finished`; assert title formats as `AICO <spinner_char> 🟢`.

3. **US-03** (Priority: P2): As a developer using only standard developer shells (PowerShell or Bash), I want the browser tab title to display cleanly as `AICO`, so that no irrelevant indicators clutter my browser chrome.
   - *Independent Test*: Open multiple PowerShell tabs with no AI tabs; assert `document.title` remains exactly `AICO`.

---

## 3. Requirements (EARS Syntax)

### Ubiquitous Requirements
- **REQ-001**: The frontend shell shall define the static document title in `frontend/index.html` as `AICO`.
- **REQ-002**: When no AI CLI terminal tab (`shell: 'agy' | 'claude'`) is present in the terminal tab registry, the TerminalTitleManager shall maintain `document.title` as exactly `AICO`.

### State-Driven Requirements
- **REQ-003**: While an AI CLI terminal tab is open, the TerminalTitleManager shall map its active status (`AiTabStatusState`) to the following Unicode glyphs:
  - `working`: Active rotating Unicode braille spinner frame from the sequence `['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏']`.
  - `finished`: Static green circle `🟢`.
  - `waiting_for_input`: Static blue circle `🔵`.
  - `error`: Static red circle `🔴`.
- **REQ-004**: While one or more AI CLI terminal tabs are open in the terminal tab registry, the TerminalTitleManager shall format `document.title` with fixed priority:
  - If only Antigravity (`shell: 'agy'`) is open: `AICO <agy_glyph>`.
  - If only Claude Code (`shell: 'claude'`) is open: `AICO <claude_glyph>`.
  - If both Antigravity and Claude Code are open: `AICO <agy_glyph> <claude_glyph>`.
- **REQ-005**: While at least one open AI CLI terminal tab has status `working`, the TerminalTitleManager shall advance the braille spinner frame at an 80ms interval, and shall immediately clear the animation timer when no active AI CLI tab remains in the `working` state or upon unmount.

### Event-Driven Requirements
- **REQ-006**: When a terminal tab is opened, closed, or when `aiTabStatuses` updates in `TerminalConsole`, the TerminalTitleManager shall recompute and set `document.title` within the current render cycle.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Terminal Title Status Seam (`frontend/tests/TerminalTitleStatus.test.tsx`)**:
   - Signature: `useTerminalTitleSync(tabs: TerminalTab[], aiTabStatuses: Record<string, AiTabStatus>): void`
   - Test File: `frontend/tests/TerminalTitleStatus.test.tsx`
   - Verifies:
     - Default title fallback to `AICO` with no tabs or standard shells.
     - Single Antigravity tab status mapping across all 4 states (`working`, `waiting_for_input`, `finished`, `error`).
     - Single Claude Code tab status mapping.
     - Dual-tab fixed priority ordering (`AICO <agy> <claude>`).
     - Spinner animation interval advancing frames while `working`.
     - Timer teardown and title reset on tab closure and component unmount.

---

## Out-of-Scope

- Custom font loading or graphical canvas favicon rendering.
- Shell titles for standard terminal utilities.