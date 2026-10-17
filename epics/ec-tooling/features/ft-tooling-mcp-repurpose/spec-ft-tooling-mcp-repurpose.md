---
id: spec-ft-tooling-mcp-repurpose
type: spec
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 2
title: "Repurpose aico-mcp for Stateful Graph, Hybrid Search & v0.1.2 Contract Validation"
status: "Completed"
derives_from: ["brain-ec-tooling", "ADR-0009", "ADR-0010"]
created_at: "2026-10-17T21:14:30Z"
---

# Feature Specification: Repurpose aico-mcp for Stateful Graph, Hybrid Search & v0.1.2 Contract Validation

## 1. Problem Statement & User Value

The `crates/aico-mcp` crate originally bundled a legacy v0.1.0 `create_sdd_artifact` file-scaffolding gate (`gate.rs`) that generated uppercase filenames (`SPEC-*.md`) inside obsolete `epics/<epic>/specs/` directories, while lacking integration with `aico-storage` for Kùzu graph traversal and LanceDB hybrid search. Now that stateless SDD file and task mutations live in `aico-cli.mjs` (`ft-tooling-sdd-cli`), `aico-mcp` must be repurposed to focus on stateful graph/vector queries and Blueprint v0.1.2 contract validation.

---

## In-Scope
- Removal of obsolete `gate.rs` and the legacy `create_sdd_artifact` tool from `crates/aico-mcp`.
- Upgrading `AnnotationExtractor`, `EarsValidator`, and `TraceabilityTracker` to Blueprint v0.1.2 (`spec-ft-*` normalization, extended `REQ-*` identifiers, and dangling requirement queries).
- Integrating `aico-mcp` with `aico-storage` (`KuzuGraphStore` and `HybridQueryEngine`) to expose `query_lineage_graph`, `search_hybrid`, and `find_dangling_requirements`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an agent performing a deep-dive or brainstorm, I want MCP tools `query_lineage_graph` and `search_hybrid` backed by `aico-storage`, so that I can retrieve multi-hop Kùzu subgraphs and LanceDB vector matches without loading unrelated files.
   - *Independent Test*: Dispatch `query_lineage_graph` and `search_hybrid` via `McpServerManager::call_tool` and verify structured graph and vector payloads are returned.
2. **US-02** (Priority: P1): As a developer maintaining contract traceability, I want `AnnotationExtractor`, `EarsValidator`, and `TraceabilityTracker` in `aico-mcp` to normalize `spec-ft-*` IDs and support prefixed requirement IDs (`REQ-CORE-001`), so that Rust in-process validation matches Blueprint v0.1.2.
   - *Independent Test*: Extract annotations from source comments with both `ft-core-discovery` and `spec-ft-core-discovery` prefixes and verify normalized `spec_id` output and dangling requirement reports.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `McpServerManager` shall expose tools `query_lineage_graph`, `search_hybrid`, `find_dangling_requirements`, `verify_task`, `extract_annotations`, and `validate_ears`, and shall omit the deprecated `create_sdd_artifact` tool.

### Event-Driven Requirements
- **REQ-002**: When `extract_annotations` is invoked, the `AnnotationExtractor` shall extract 2-part and 3-part `@verifies` and `@implements` tags and normalize `spec_id` to the canonical lowercase `spec-ft-*` prefix when a `ft-*` slug is supplied.
- **REQ-003**: When `validate_ears` is invoked, the `EarsValidator` shall accept both standard (`REQ-001`) and domain-prefixed (`REQ-CORE-001`) requirement identifiers and classify the statement into one of the 5 canonical EARS patterns requiring modal `shall`.
- **REQ-004**: When `query_lineage_graph` is invoked with a `symbol` or `requirement_id` and `hops` between 1 and 4, the `GraphToolHandler` shall execute a lineage query against `KuzuGraphStore` and return a structured JSON subgraph payload.
- **REQ-005**: When `search_hybrid` is invoked with a `query` string and `top_k`, the `HybridToolHandler` shall execute `HybridQueryEngine` search across `LanceVectorStore` and `KuzuGraphStore` and return ranked vector matches with graph context.
- **REQ-006**: When `find_dangling_requirements` is invoked with declared requirements and source/test annotations, the `TraceabilityTracker` shall return all requirements missing either verifying tests or implementing symbols.

### Unwanted Behavior & Error Handling
- **REQ-007**: If `query_lineage_graph` is invoked with `hops` outside `1..=4` or an empty `symbol`, then the `McpServerManager` shall reject the call with `ToolError::InvalidParameters`.
- **REQ-008**: If `search_hybrid` is invoked with an empty `query` string or `top_k` of `0`, then the `McpServerManager` shall reject the call with `ToolError::InvalidParameters`.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Annotation & EARS Validator Seam (`crates/aico-mcp/src/annotation.rs`, `crates/aico-mcp/src/ears.rs`)**:
   - Signature: `AnnotationExtractor::extract(source: &str) -> Vec<Annotation>`, `EarsValidator::validate(req_id: &str, statement: &str) -> Result<EarsPattern, EarsValidationError>`
   - Seam Test: `crates/aico-mcp/tests/annotation_test.rs`
2. **MCP Server Registry & Dispatcher Seam (`crates/aico-mcp/src/server.rs`)**:
   - Signature: `McpServerManager::list_tools() -> Vec<ToolDefinition>`, `McpServerManager::call_tool(name: &str, params: Value) -> Result<Value, ToolError>`
   - Seam Test: `crates/aico-mcp/tests/mcp_server_test.rs`
3. **Traceability & Dangling Requirement Seam (`crates/aico-mcp/src/traceability.rs`)**:
   - Signature: `TraceabilityTracker::verify_task(...)`, `TraceabilityTracker::find_dangling_requirements(&self) -> Vec<DanglingRequirement>`
   - Seam Test: `crates/aico-mcp/tests/mcp_verification_test.rs`
4. **Stateful Graph & Hybrid Search Seam (`crates/aico-mcp/src/graph_tools.rs`)**:
   - Signature: `GraphToolHandler::query_lineage_graph(...)`, `HybridToolHandler::search_hybrid(...)`
   - Seam Test: `crates/aico-mcp/tests/mcp_graph_hybrid_test.rs`

---

## Out-of-Scope
- Creating or mutating markdown files on disk from `aico-mcp` (delegated to `aico-cli.mjs` in `ft-tooling-sdd-cli`).
- External network LLM API calls inside `aico-mcp`.

---

## 5. Open Clarifications

None. Ratified in [ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md) and [ADR-0010](../../../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md).
