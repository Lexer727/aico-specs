---
id: plan-ft-specs-blueprint-standards
type: plan
aico_version: "0.1.2"
epic: "ec-specs"
order: 1
title: "SDD Blueprint Standards, Relative Linking & Task Lifecycle"
status: Completed
derives_from: ["spec-ft-specs-blueprint-standards"]
created_at: "2026-10-06T01:43:00Z"
---

# Implementation Plan: SDD Blueprint Standards, Relative Linking & Task Lifecycle

## 1. Technical Architecture & Decisions

- **Scope**: Canonical specification templates and validator schemas.
- **Dependencies**: Rust standard regex and frontmatter parsers in `aico-core` and `aico-mcp`.
- **Integrations**: EARS linting rules and MCP artifact generation gates.

---

## 2. Task Graph (DAG)

### Phase 1: Blueprint Schema Definitions

- **TASK-SPECS-001-1**: Formalize Universal YAML Frontmatter Schema
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-005, REQ-006, REQ-007, REQ-008
  - **Blocked by**: None
  - **What it delivers**:
    - Defines mandatory frontmatter fields: `id`, `type`, `aico_version`, `title`, `status`, `derives_from`, `created_at`.
    - Enforces 4-state lifecycle: `Draft`, `Pending`, `In Progress`, `Completed`.

- **TASK-SPECS-001-2**: Formalize Single Source of Truth & Relative Link Rule
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-003, REQ-011
  - **Blocked by**: TASK-SPECS-001-1
  - **What it delivers**:
    - Establishes rule that markdown body must not duplicate frontmatter attributes.
    - Requires relative markdown paths (`../../adr/...`) across all documents.

- **TASK-SPECS-001-3**: Formalize 6-State Task Tracking Schema with Audit Trail
  - **Status**: Done
  - **Addresses**: REQ-004, REQ-009, REQ-010, REQ-012
  - **Blocked by**: TASK-SPECS-001-1
  - **What it delivers**:
    - Defines task statuses: `Pending`, `In Progress`, `Done`, `Blocked`, `Skipped/Canceled`, `Rewritten`.
    - Enforces mandatory `Reason` and `Rewrite Target` fields for canceled or rewritten tasks.

---

## 3. Initial Active Frontier

- None remaining (All tasks completed and verified).
