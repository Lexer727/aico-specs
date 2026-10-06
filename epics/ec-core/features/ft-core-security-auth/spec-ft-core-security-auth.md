---
id: spec-ft-core-security-auth
type: spec
aico_version: "0.1.2"
epic: "ec-core"
order: 2
title: "Security, Auto-Mode & Command Execution Engine"
status: "Completed"
derives_from: ["ADR-0003", "ADR-0008"]
created_at: "2026-10-05T19:15:00Z"
---

# Feature Specification: Security, Auto-Mode & Command Execution Engine


---

## 1. Problem Statement & User Value

Autonomous coding agents need to read documentation, modify source code, run tests, and manage dependencies without forcing the developer to approve every trivial step. However, granting unconstrained shell and filesystem execution risks accidental file corruption, system mutation, or network egress attacks.

This specification defines the 3-Tier Security Policy Engine running in the Rust backend that enforces strict filesystem boundaries via path canonicalization, categorizes shell commands through AST parsing into safe auto-executable tiers, and gates destructive or network-egress commands behind interactive human approval prompts.

---

## In-Scope
- Multi-tier command permission classification (Safe, Supervised, Blocked).
- Path boundary containment and canonicalization preventing directory traversal.
- Audit logging for all destructive CLI operations and file modifications.
- Local loopback pairing token validation with session expiration.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want the agent to auto-execute read-only and testing commands in Auto-Mode without prompting me, so that the development loop runs uninterrupted.
   - *Independent Test*: Issue a command `cargo test` under Auto-Mode and verify execution proceeds immediately without prompt.
2. **US-02** (Priority: P1): As a developer, I want file writes strictly confined to the active project codebase and specs directory, so that the agent cannot overwrite external files or system binaries.
   - *Independent Test*: Attempt to write to `C:/Windows/temp.txt` or `d:/other-project/` and verify the engine rejects the write with a boundary violation error.
3. **US-03** (Priority: P1): As a developer, I want dangerous or network-egress commands (`git push`, `rm -rf`, `npm publish`) to pause and prompt me for 1-click confirmation, so that I maintain final control over external side effects.
   - *Independent Test*: Dispatch `git push origin main` and verify execution halts and emits a `PermissionRequest` event over WebSocket.
4. **US-04** (Priority: P2): As a developer, I want every auto-executed mutating command logged to an append-only audit trail, so that I can inspect and revert changes made during autonomous sessions.
   - *Independent Test*: Execute `npm install lodash` under Auto-Mode and verify the command, timestamp, and exit status are recorded in the audit log table.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The PathCanonicalizer shall resolve all target file paths to their absolute canonical form via `std::fs::canonicalize` before any read or write operation.
- **REQ-002**: The CommandAstParser shall parse raw shell command strings into distinct executable binary names and argument tokens using shell lexing grammar.
- **REQ-003**: The PolicyEngine shall assign every incoming command to Tier 1 (Auto-Allowed), Tier 2 (Workspace Mutate), or Tier 3 (Interactive Gate) prior to execution.
- **REQ-004**: The AuditLogger shall append a structured record containing timestamp, command string, working directory, and exit code to SQLite storage upon completion of any Tier 2 command.

### Event-Driven Requirements
- **REQ-005**: When an agent requests a file write, the PathCanonicalizer shall verify that the canonical path is a descendant of the active project codebase path or active specs repository path.
- **REQ-006**: When an incoming command matches the Tier 1 whitelist (`git status`, `git diff`, `git log`, `cargo test`, `npm test`, `pytest`, `tsc --noEmit`), the ExecutionSandbox shall execute the process immediately in the project working directory without human prompt.
- **REQ-007**: While Auto-Mode is active, when a command matches Tier 2 workspace mutations (`npm install`, `cargo add`, `git add`, `git commit`), the ExecutionSandbox shall execute the process and stream standard output to the AuditLogger.
- **REQ-008**: When a command matches Tier 3 criteria (containing network egress like `git push` or `npm publish`, destructive commands like `rm -rf`, or arbitrary shell pipes), the InteractiveGate shall pause agent execution and emit a `PermissionRequest` WebSocket event.
- **REQ-009**: When the user clicks `[Approve]` on desktop or mobile for a pending `PermissionRequest`, the InteractiveGate shall resume execution of the paused command.
- **REQ-010**: When the user clicks `[Deny]` for a pending `PermissionRequest`, the InteractiveGate shall cancel the command and return a `PermissionDenied` error to the calling agent.

### State-Driven Requirements
- **REQ-011**: While Auto-Mode is disabled, the PolicyEngine shall route all Tier 2 commands to the InteractiveGate for manual confirmation.
- **REQ-012**: While a Tier 3 command is awaiting user confirmation, the InteractiveGate shall enforce a 5-minute timeout, rejecting the command automatically if no response is received.

### Unwanted Behavior & Error Handling
- **REQ-013**: If a requested file write targets a path outside the active project codebase and outside the active specs repository, then the PathCanonicalizer shall abort the write operation with error `SecurityError::PathBoundaryViolation`.
- **REQ-014**: If the CommandAstParser encounters unparseable shell syntax or dangerous shell injection tokens, then the PolicyEngine shall escalate the command to Tier 3 requiring interactive approval.

### Optional Feature Requirements
- **REQ-015**: Where a project-level `.aico/whitelist.json` file is present, the PolicyEngine shall append custom user-defined command patterns to the Tier 1 auto-allowed list.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Policy Evaluation Seam (`aico_core::security::PolicyEngine`)**:
   - Signature: `pub fn evaluate_command(&self, cmd: &str, cwd: &Path, mode: AutoModeState) -> PolicyDecision`
   - Allows unit testing command classification across Tier 1, Tier 2, and Tier 3 without spawning OS processes.
2. **Path Boundary Guard Seam (`aico_core::security::PathGuard`)**:
   - Signature: `pub fn validate_write_path(&self, target: &Path, allowed_roots: &[PathBuf]) -> Result<PathBuf, SecurityError>`
   - Verifies path canonicalization, symlink resolution, and directory containment.
3. **Execution Sandbox Seam (`aico_core::security::ExecutionSandbox`)**:
   - Signature: `pub async fn run_command(&self, cmd: ParsedCommand, cwd: &Path) -> Result<ProcessOutput, ExecutionError>`
   - Captures stdout/stderr, monitors exit codes, and pipes logs to the audit trail.

---

## Out-of-Scope

- OS-level kernel sandboxing (e.g. Linux cgroups, Windows AppContainer, or Docker isolation are not required for local developer use).
- Modifying OS-level file permissions (chmod/ACLs).
- Monitoring outbound network traffic outside the command execution gateway.

---

## 6. Open Clarifications

None. Ratified in [ADR-0003](../../../../adr/0003-security-and-auto-mode-enforcement.md).
