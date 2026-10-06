---
id: spec-ft-core-mcp-tools
type: spec
aico_version: "0.1.2"
epic: "ec-core"
order: 5
title: "Deterministic MCP Tool Gates & In-Band Code Annotations"
status: "Completed"
derives_from: ["ADR-0003", "ADR-0006"]
created_at: "2026-10-05T19:15:00Z"
---

# Feature Specification: Deterministic MCP Tool Gates & Code Annotations


---

## 1. Problem Statement & User Value

When LLM coding agents write documentation and modify source code via plain text commands, they frequently hallucinate arbitrary file paths, fabricate non-existent requirement IDs, or miss critical lineage relationships. Without rigid tool interfaces, graph relationships and specification structures rot rapidly.

This specification defines AICO's Model Context Protocol (MCP) Tool Gates and in-band annotation engine. Agents interact with specifications, tasks, and graphs strictly through typed, deterministic MCP tool calls that calculate file paths automatically, inject immutable YAML frontmatter, and sync in-band code annotations (`// @implements` and `// @verifies`) directly into the Kùzu property graph.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want agents to create specifications strictly through typed MCP tools, so that file naming, frontmatter, and directory placement are 100% consistent.
   - *Independent Test*: Call `create_sdd_artifact` with `artifact_type: "spec"`, `epic: "001-core"`, and `slug: "auth"` and verify the file is created at `<specs>/epics/001-core/specs/auth/spec-auth.md` with complete YAML frontmatter.
2. **US-02** (Priority: P1): As a developer, I want functions tagged with `// @implements SPEC-ID::REQ-ID` linked to their requirements in the graph on save, so that requirement coverage is tracked automatically.
   - *Independent Test*: Save a file containing `// @implements SPEC-AUTH-001::REQ-042` and verify Kùzu creates an `(CodeSymbol)-[:IMPLEMENTS]->(Requirement)` edge.
3. **US-03** (Priority: P1): As a developer, I want tests tagged with `// @verifies SPEC-ID::REQ-ID` linked to their requirements in the graph, so that test pass/fail status updates the specification table in real time.
   - *Independent Test*: Save a test containing `// @verifies SPEC-AUTH-001::REQ-042` and verify Kùzu creates a `(Test)-[:VERIFIES]->(Requirement)` edge.
4. **US-04** (Priority: P2): As an agent implementing code, I want to query a 3-hop structural subgraph around my target function via MCP, so that I receive exact caller and contract context without reading unrelated files.
   - *Independent Test*: Call MCP tool `query_lineage_graph` for symbol `validate_token` and verify the returned JSON contains callers, implementing requirements, and parent brainstorms.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The McpServer shall expose typed Model Context Protocol JSON-RPC endpoints over local stdio and authenticated WebSocket transports.
- **REQ-002**: The PathDeterminizer shall calculate target destination file paths using the canonical pattern `<specs>/epics/<epic-slug>/specs/<spec-slug>/<ARTIFACT>-<slug>.md`.
- **REQ-003**: The FrontmatterInjector shall inject immutable YAML frontmatter containing `id`, `type`, `aico_version`, `epic`, `created_at`, and `derives_from` into every created SDD document.
- **REQ-004**: The AnnotationParser shall extract in-band annotations (`@implements <SPEC-ID>::<REQ-ID>` and `@verifies <SPEC-ID>::<REQ-ID>`) from comment tokens in source and test files.

### Event-Driven Requirements
- **REQ-005**: When an agent invokes tool `create_sdd_artifact`, the ArtifactToolGate shall validate parameters, write the file to the deterministically calculated path, and merge document nodes into Kùzu.
- **REQ-006**: When an agent invokes tool `query_lineage_graph`, the GraphToolGate shall execute a 2-to-4 hop Cypher query in Kùzu and return the structural subgraph as a formatted JSON context payload.
- **REQ-007**: When source code containing an `@implements` annotation is saved, the AnnotationParser shall create an `[:IMPLEMENTS]` edge in Kùzu connecting the enclosing `CodeSymbol` to the target `Requirement`.
- **REQ-008**: When test code containing an `@verifies` annotation is saved, the AnnotationParser shall create a `[:VERIFIES]` edge in Kùzu connecting the enclosing `Test` to the target `Requirement`.
- **REQ-009**: When a test runner execution completes, the TestResultSynchronizer shall update the verification status property (`status: "Completed"` or `status: "Completed"`) on all `[:VERIFIES]` edges.

### State-Driven Requirements
- **REQ-010**: While an agent is generating artifacts, the ArtifactToolGate shall reject any attempt to pass custom relative or absolute target file paths, enforcing deterministic path calculation.
- **REQ-011**: While the project watcher is active, the AnnotationParser shall remove stale `[:IMPLEMENTS]` and `[:VERIFIES]` edges when annotations are deleted from source files.

### Unwanted Behavior & Error Handling
- **REQ-012**: If an agent attempts to create a document with an ID that already exists in Kùzu, then the ArtifactToolGate shall reject the request with `ToolError::DuplicateArtifactId`.
- **REQ-013**: If an `@implements` annotation references a requirement ID that does not exist in any specification, then the AnnotationParser shall log a lint warning and create a dangling requirement reference node marked `status: "Completed"`.

### Optional Feature Requirements
- **REQ-014**: Where the language syntax uses `#` for comments (e.g. Python), the AnnotationParser shall match `# @implements` and `# @verifies` comment patterns.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **MCP Service Dispatcher Seam (`aico_core::mcp::McpService`)**:
   - Signature: `pub async fn handle_tool_call(&self, name: &str, params: Value) -> Result<ToolResult, McpError>`
   - Tests tool execution, parameter parsing, and side effects across all MCP gates.
2. **Annotation Parser Seam (`aico_core::annotations::AnnotationParser`)**:
   - Signature: `pub fn parse_file_annotations(&self, content: &str, file_type: Language) -> Vec<ParsedAnnotation>`
   - Verifies comment parsing across Rust, TypeScript, and Python without database dependencies.
3. **Graph Tool Gate Seam (`aico_core::mcp::GraphToolGate`)**:
   - Signature: `pub fn get_symbol_subgraph(&self, symbol_name: &str, hops: usize) -> Result<SubgraphPayload, GraphError>`
   - Validates context bundling for agent prompt injection.

---

## 5. Out of Scope

- Arbitrary file creation outside specifications and codebase boundaries.
- Fabricating Git commits inside MCP tool handlers (commits are handled through the versioning engine).
- Automatic refactoring of code symbols outside user-approved sessions.

---

## 6. Open Clarifications

None. Ratified in [ADR-0003](../../../../adr/0003-security-and-auto-mode-enforcement.md) and [ADR-0006](../../../../adr/0006-packaging-aico-specs-and-version-tagging.md).
