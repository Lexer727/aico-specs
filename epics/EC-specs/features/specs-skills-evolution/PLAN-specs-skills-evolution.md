---
id: PLAN-specs-skills-evolution
type: plan
aico_version: "0.1.2"
epic: "EC-specs"
order: 2
title: "AICO Skills Suite Evolution & MCP Gate Hardening"
status: Completed
derives_from: ["SPEC-specs-skills-evolution"]
created_at: "2026-10-06T01:44:00Z"
---

# Implementation Plan: AICO Skills Suite Evolution & MCP Gate Hardening

## 1. Technical Architecture & Decisions

- Update templates and instructions in `d:/aico/codebase/aico-skills`.
- Update Rust MCP server artifact generator in `crates/aico-mcp/src/gate.rs`.
- Update tests in `crates/aico-mcp/tests/mcp_gate_test.rs`.

---

## 2. Task Graph (DAG)

### Phase 1: Skill Template & Instruction Updates

- **TASK-SPECS-002-1**: Update `SPEC-TEMPLATE.md` and `aico-spec-creation/SKILL.md`
  - **Status**: Done
  - **Addresses**: REQ-001
  - **Blocked by**: None
  - **What it delivers**:
    - Prepend canonical YAML frontmatter.
    - Remove duplicate `ID`, `Status`, `Derives From` from body.
    - Require relative markdown links.

- **TASK-SPECS-002-2**: Update `PLAN-TEMPLATE.md` and `aico-planning/SKILL.md`
  - **Status**: Done
  - **Addresses**: REQ-002
  - **Blocked by**: None
  - **What it delivers**:
    - Prepend canonical YAML frontmatter (`type: plan`).
    - Format task nodes with explicit `Status` fields and audit rationale fields.

- **TASK-SPECS-002-3**: Update `aico-implementation`, `aico-epic`, and `aico-review-loop` Skills
  - **Status**: Done
  - **Addresses**: REQ-004, REQ-005, REQ-006
  - **Blocked by**: TASK-SPECS-002-1, TASK-SPECS-002-2
  - **What it delivers**:
    - Instruct `/aico-implementation` to update task `Status` field to `Done` upon test passage.
    - Instruct `/aico-review-loop` to transition frontmatter status to `Completed`.

- **TASK-SPECS-002-4**: Upgrade `crates/aico-mcp/src/gate.rs` to 0.1.1
  - **Status**: Done
  - **Addresses**: REQ-003
  - **Blocked by**: None
  - **What it delivers**:
    - Set `aico_version: "0.1.1"` in `ArtifactToolGate::create_artifact`.
    - Verify with `cargo test -p aico-mcp`.

---

## 3. Initial Active Frontier

- None remaining (All tasks completed and verified).
