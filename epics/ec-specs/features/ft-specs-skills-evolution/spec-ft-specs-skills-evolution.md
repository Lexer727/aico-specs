---
id: spec-ft-specs-skills-evolution
type: spec
aico_version: "0.1.2"
epic: "ec-specs"
order: 2
title: "AICO Skills Suite Evolution & MCP Gate Hardening"
status: "Completed"
derives_from: ["ADR-0009", "spec-specs-001"]
created_at: "2026-10-06T01:44:00Z"
---

# Feature Specification: AICO Skills Suite Evolution & MCP Gate Hardening

## 1. Problem Statement & User Value

Agents executing workflows under `aico-skills` need precise instructions, templates, and guardrails to generate compliant SDD artifacts. If skills provide obsolete templates lacking frontmatter, using checkboxes instead of structured status fields, or allowing duplicate metadata, agents will continue generating legacy artifacts.

This specification details the updates required across the `aico-skills` plugin suite and the `aico-mcp` artifact tool gate to enforce the 0.1.1 blueprint standard.

---

## In-Scope
- Synchronization of skill templates across workspace and plugin locations.
- Enforcement of EARS syntax and canonical artifact naming conventions.
- Automation scripts for validating and testing skills suite compatibility.
- Decoupled ordering support and vertical tracer-bullet DAG task execution.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an agent executing `/aico-spec-creation`, I want the spec template to include canonical frontmatter and omit duplicate body metadata, so that my generated specifications comply with version 0.1.1 standards.
   - *Independent Test*: Run spec generation template and verify output against 0.1.1 frontmatter and body schema.
2. **US-02** (Priority: P1): As an agent executing `/aico-planning`, I want the plan template to include frontmatter and 6-state task tracking blocks, so that task progress and mutation history are strictly recorded.
   - *Independent Test*: Generate an implementation plan and verify that all tasks contain structured `Status` fields.
3. **US-03** (Priority: P1): As a tool gate in `aico-mcp`, I want `ArtifactToolGate::create_artifact` to generate frontmatter with version `0.1.1` and new status conventions, so that artifacts created via MCP meet system standards.
   - *Independent Test*: Call `create_artifact` via MCP tests and verify resulting file header.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The SpecTemplate in `aico-spec-creation` shall include full YAML frontmatter and exclude redundant `ID`, `Status`, and `Derives From` fields from the markdown body.
- **REQ-002**: The PlanTemplate in `aico-planning` shall include full YAML frontmatter with `type: plan` and format tasks with explicit `Status` fields.
- **REQ-003**: The ArtifactToolGate in `aico-mcp` shall inject YAML frontmatter with `aico_version: "0.1.1"` and `status: "Draft"` upon new artifact creation.

### Event-Driven Requirements
- **REQ-004**: When an agent executing `/aico-implementation` marks a task as finished, the SkillInstruction shall require updating the task's `Status` to `Done`.
- **REQ-005**: When an agent supersedes or cancels a task, the SkillInstruction shall require updating the task's `Status` to `Rewritten` or `Skipped/Canceled` with an accompanying `Reason` and `Rewrite Target`.
- **REQ-006**: When `/aico-review-loop` passes all verification checks, the SkillInstruction shall update the specification and plan frontmatter status to `Completed`.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **MCP Gate Test (`crates/aico-mcp/tests/mcp_gate_test.rs`)**:
   - Verifies `ArtifactToolGate` writes `0.1.1` frontmatter.
2. **Skill Template Integrity**:
   - Inspection of `spec-template.md` and `plan-template.md`.

---

## Out-of-Scope

- Modifying non-AICO third-party agent frameworks.
- Altering external language servers.

---

## 6. Open Clarifications

None. Ratified in [ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md).
