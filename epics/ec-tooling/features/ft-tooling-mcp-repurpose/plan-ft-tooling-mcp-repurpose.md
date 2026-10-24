---
id: plan-ft-tooling-mcp-repurpose
type: plan
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 2
title: "Repurpose aico-mcp for Stateful Graph, Hybrid Search & v0.1.2 Contract Validation Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-tooling-mcp-repurpose"]
created_at: "2026-10-17T21:14:30Z"
---

# Implementation Plan: Repurpose aico-mcp for Stateful Graph, Hybrid Search & v0.1.2 Contract Validation

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-mcp` linked with `aico-storage` and `tokio`.
- **Decommissioning `gate.rs`**: Remove `crates/aico-mcp/src/gate.rs` and `crates/aico-mcp/tests/mcp_gate_test.rs` (replacing with `crates/aico-mcp/src/graph_tools.rs` and `crates/aico-mcp/tests/mcp_graph_hybrid_test.rs`).
- **Upgrading Reusable Validators**: Enhance `annotation.rs`, `ears.rs`, and `traceability.rs` to align with Blueprint v0.1.2 (`spec-ft-*` normalization, domain-prefixed `REQ-*` IDs, and `find_dangling_requirements` tool dispatch).

---

## 2. Task Graph (DAG)

### Phase 1: Remove Legacy Gate & Upgrade Validators to v0.1.2

- **TASK-TOOL-002-1**: Remove `gate.rs` & Upgrade Annotation, EARS, and Traceability Modules to v0.1.2
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-006`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Removes obsolete `crates/aico-mcp/src/gate.rs` and `create_sdd_artifact` registration.
    - Updates `AnnotationExtractor` to support optional `spec-` normalization and `EarsValidator` to support domain-prefixed `REQ-[A-Z0-9]+-\d{3,4}` IDs.
    - Exposes `find_dangling_requirements` as a first-class tool in `McpServerManager`.
  - **Verification Seam**: `crates/aico-mcp/tests/annotation_test.rs`

---

### Phase 2: Stateful AicoGraphStore Lineage & LanceDB Hybrid Search MCP Tools

- **TASK-TOOL-002-2**: Implement `query_lineage_graph` & `search_hybrid` Backed by `aico-storage`
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-004`, `REQ-005`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-TOOL-002-1`
  - **What it delivers**:
    - Adds `aico-storage` dependency to `crates/aico-mcp/Cargo.toml`.
    - Creates `crates/aico-mcp/src/graph_tools.rs` implementing `GraphToolHandler` (`query_lineage_graph`) and `HybridToolHandler` (`search_hybrid`).
    - Enforces hop bounds (`1..=4`), non-empty query validation, and registers both tools in `McpServerManager`.
  - **Verification Seam**: `crates/aico-mcp/tests/mcp_graph_hybrid_test.rs`
