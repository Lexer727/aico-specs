---
id: PLAN-CORE-005
type: plan
aico_version: "0.1.1"
epic: "001-aico-core"
title: "Deterministic MCP Tool Gates & Code Annotations"
status: Completed
derives_from: ["SPEC-CORE-005"]
created_at: "2026-10-05T20:00:00Z"
---

# Implementation Plan: Deterministic MCP Tool Gates & Code Annotations


---

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-mcp` (part of the AICO Rust Cargo workspace).
- **Core Dependencies**:
  - `mcp-core` / JSON-RPC 2.0 parser: Implements the Model Context Protocol over stdio and in-process dispatch.
  - `regex`: Fast extraction of `// @verifies SPEC-ID::REQ-ID` and `// @implements SPEC-ID::REQ-ID` comment annotations.
  - `ears-parser`: Custom AST/regex parser for validating EARS syntax patterns (`While ... the system shall ...`, `When ... the system shall ...`, `Where ... the system shall ...`, `If ... then the system shall ...`, `The system shall ...`).
  - `kuzu`: Transactional writes of artifact nodes, requirement relationships, and verification edges.
- **Gate Guarantee**: Artifacts cannot be written directly by agents bypassing validation. Agents must invoke `create_sdd_artifact`, which enforces syntax, required metadata, and graph registration atomically.

---

## 2. Task Graph (DAG)

### Phase 1: Annotation Scanner & EARS Syntax Validator (Unblocked Frontier)

- **TASK-CORE-005-1**: Source Code Annotation Scanner & EARS Validator
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-004`, `REQ-005`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implements `AnnotationExtractor` scanning test files for `// @verifies <SPEC-ID>::<REQ-ID>` and source files for `// @implements <SPEC-ID>::<REQ-ID>`.
    - Implements `EarsSyntaxValidator` verifying modal keyword `shall`, precondition clauses, and requirement ID syntax (`REQ-\d{3}`).
    - Returns structured validation errors with line numbers and corrective guidance.
  - **Verification Seam**: `tests/mcp_annotation_test.rs::test_annotation_extraction_and_ears_validation`

---

### Phase 2: In-Process MCP Server & Dispatch Hub

- **TASK-CORE-005-2**: In-Process MCP Server & Protocol Dispatcher
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-008`, `REQ-009`
  - **Blocked by**: `TASK-CORE-001-1` (Axum Server / Tokio Runtime)
  - **What it delivers**:
    - Implements `McpServerManager` providing standard MCP tool registry (`tools/list`, `tools/call`).
    - Exposes core SDD tools: `create_sdd_artifact`, `verify_task`, `query_graph`, `run_test_seam`.
    - Integrates with `PolicyEngine` to enforce Tier classification and audit logging on every tool execution.
    - Formats error responses with actionable repair suggestions on validation failures.
  - **Verification Seam**: `tests/mcp_server_test.rs::test_mcp_tool_listing_and_dispatch`

---

### Phase 3: SDD Artifact Gate & File Atomicity

- **TASK-CORE-005-3**: Deterministic SDD Artifact Creation Gate
  - **Status**: Done
  - **Addresses**: `REQ-002`
  - **Blocked by**: `TASK-CORE-005-1`, `TASK-CORE-005-2`
  - **What it delivers**:
    - Implements handler for `create_sdd_artifact`.
    - Validates artifact type (`brainstorm`, `deep_dive`, `spec`, `plan`).
    - Enforces required YAML frontmatter and derives-from IDs.
    - Performs atomic file write to the dedicated specs directory (`<project>-specs/epics/<slug>/...`).
  - **Verification Seam**: `tests/mcp_gate_test.rs::test_create_sdd_artifact_gate`

---

### Phase 4: Traceability Graph & Test-Before-Code Enforcement

- **TASK-CORE-005-4**: Traceability Graph Verification & Test-Before-Code Gate
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-CORE-005-1`, `TASK-CORE-003-2` (Storage Schema)
  - **What it delivers**:
    - Writes `VERIFIES` and `IMPLEMENTS` edges between tests, code symbols, and requirements into Kùzu.
    - Implements `verify_task` gate checking that red behavioral tests were executed and passed before marking an implementation task complete.
    - Exposes graph query checking for dangling requirements (uncovered by tests or unaddressed by tasks).
  - **Verification Seam**: `tests/mcp_verification_test.rs::test_traceability_and_test_before_code_gate`

---

## 3. Initial Active Frontier

- `TASK-CORE-005-1`: Source Code Annotation Scanner & EARS Validator (Ready for `/aico-test-scaffolding`).
