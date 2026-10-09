---
id: spec-ft-specs-agent-rules
type: spec
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 5
title: "Workspace Agent Rules & Mandatory Human Approval Gate"
status: "Completed"
derives_from: ["brain-ec-specs", "ADR-0009"]
created_at: "2026-10-10T01:20:00Z"
---

# Feature Specification: Workspace Agent Rules & Mandatory Human Approval Gate

## 1. Problem Statement & User Value

AI coding assistants running within complex, multi-repository projects risk making premature changes without adequate developer alignment. When agents bypass intermediate review gates—jumping straight from diagnostic discovery into modifying code, tests, or specifications—developers lose oversight over the evolution of their codebase.

This feature establishes the canonical workspace rule standard (`GEMINI.md` / `AGENTS.md`) across AICO specification repositories, codifying a strict, non-negotiable **Mandatory Human Review & Approval Gate**. The rule instructs agent runtimes to always halt after diagnosis, present proposed changes and vertical tracer-bullet DAGs, and await explicit developer authorization before modifying any files.

---

## In-Scope

- Canonical `GEMINI.md` rule file placed at the root of `aico-specs` and mirrored at the workspace root (`d:/aico`).
- Strict 3-phase workflow enforcement:
  1. Investigate & Diagnose.
  2. Present Proposals & Plan.
  3. Halt & Await Developer Sign-Off.
- Guaranteed automatic discovery across Antigravity, Google Gemini CLI (`agy`), and future agent environments.
- Zero premature file modification or spec manipulation prior to explicit user consent.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer collaborating with an AI coding agent, I want the agent to always pause after diagnosing an issue or drafting a plan, so that I can review and approve proposed changes before any files are altered.
   - *Independent Test*: Present an issue or triage prompt to the agent; verify that the agent halts after diagnosis with a proposal and waits for user confirmation before calling write/edit tools.
2. **US-02** (Priority: P1): As an engineer cloning the `aico-specs` repository, I want a version-controlled `GEMINI.md` at the repository root, so that any AI tool automatically discovers and enforces the project's behavioral rules.
   - *Independent Test*: Inspect `aico-specs` root and confirm presence of valid, well-formed `GEMINI.md`.
3. **US-03** (Priority: P2): As a developer working in a multi-repo workspace (`d:/aico`), I want root-level workspace discovery to enforce identical approval gates during bootstrap chats.
   - *Independent Test*: Inspect `d:/aico/GEMINI.md` and verify rules match repository standards.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The WorkspaceAgent shall discover and load `GEMINI.md` from the root of the active specification repository to enforce persistent behavioral guardrails across all turns.
- **REQ-002**: The WorkspaceAgent shall recognize identical rule constraints in multi-repository workspace roots.

### State-Driven Requirements
- **REQ-003**: While conducting initial problem exploration, bug diagnosis, or architecture trade-off analysis, the WorkspaceAgent shall restrict tool invocations to read-only inspection operations.
- **REQ-004**: While an implementation plan or specification amendment has not received explicit developer sign-off, the WorkspaceAgent shall halt tool execution and await approval.

### Event-Driven Requirements
- **REQ-005**: When the developer explicitly confirms approval of a proposed specification, plan, or bug fix, the WorkspaceAgent shall transition to implementation and execute the authorized changes.

### Unwanted Behavior & Guardrails
- **REQ-006**: If the developer has not provided explicit authorization to proceed, then the WorkspaceAgent shall not execute write, replace, or delete operations on source code, tests, or specification artifacts.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Rule File Integrity (`aico_specs::GEMINI.md`)**:
   - Location: `GEMINI.md` at specs root.
   - Verifies presence of Mandatory Review & Approval Gate and AICO Blueprint Conformance sections.
2. **Artifact Verification (`aico-verify`)**:
   - Automated conformance check via `scripts/verify-artifacts.mjs` verifying 100% structural compliance.

---

## Out-of-Scope

- Custom interactive GUI modal dialogs outside of the conversational approval flow.
- Automated token charging or billing limits.
- Non-markdown agent rule formats.

---

## 6. Open Clarifications

None. Ratified in [ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md).
