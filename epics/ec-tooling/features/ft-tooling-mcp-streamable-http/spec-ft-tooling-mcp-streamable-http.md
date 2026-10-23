---
id: spec-ft-tooling-mcp-streamable-http
type: spec
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 4
title: "Streamable HTTP MCP Transport on aico-server"
status: "Completed"
derives_from: ["brain-ec-tooling", "ADR-0001", "ADR-0003"]
created_at: "2026-10-22T20:24:06.575Z"
---

# Feature Specification: Streamable HTTP MCP Transport on aico-server

## 1. Problem Statement & User Value

While `crates/aico-mcp` exposes 6 domain tools via `McpServerManager`, it currently functions only as an in-process Rust library without a network or stdio transport layer. In addition, its graph tools open isolated temporary directories rather than querying the active workspace's embedded storage. Consequently, AI coding agents and CLI environments cannot discover or invoke AICO tools via the Model Context Protocol (`/mcp`), and running a persistent standalone executable on Windows introduces file-locking hazards during self-hosting compilation (`cargo test`).

This specification defines a Streamable HTTP MCP transport mounted directly on `aico-server` at `POST /mcp` and `GET /mcp` with CORS preflight support. When the server is running, Antigravity and other MCP-compliant agents connect via standard HTTP JSON-RPC 2.0 to discover and execute AICO tools against the active project's indexed Kùzu property graph and LanceDB vector store with zero binary locking, and `SkillsSynchronizer` automatically provisions `mcp_config.json` into workspace `.agents/` directories.

---

## In-Scope

- JSON-RPC 2.0 protocol envelope handling in `aico-mcp` (`initialize`, `notifications/initialized`, `ping`, `tools/list`, and `tools/call`).
- Dispatching tool invocations to `McpServerManager` with context-aware `StorageManager` binding for property graph and hybrid search queries.
- Mounting Streamable HTTP endpoints (`POST /mcp`, `GET /mcp`, and `OPTIONS /mcp`) on `aico-server` with loopback authentication and CORS preflight headers.
- Displaying the active Streamable MCP endpoint in the `aico-server` startup console banner.
- Auto-distributing canonical `mcp_config.json` to `.agents/mcp_config.json` and `.agents/skills/mcp_config.json` via `SkillsSynchronizer`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an AI agent using Antigravity, I want `aico-server` to respond to MCP discovery and tool calls over Streamable HTTP, so that `/mcp` shows the `aico` server and its registered tools.
   - *Independent Test*: Send an MCP `initialize` request and `tools/list` request to `http://localhost:4242/mcp` and verify standard JSON-RPC responses containing all 6 tools.
2. **US-02** (Priority: P1): As an agent executing `query_lineage_graph` or `search_hybrid` over MCP, I want queries executed against the active project's storage, so that I receive real codebase graph context and vector embeddings.
   - *Independent Test*: Invoke `tools/call` for `query_lineage_graph` with an active `StorageManager` in `AppState` and verify the query executes against the workspace storage.
3. **US-03** (Priority: P2): As a developer editing code in the workspace, I want the MCP server to run inside the existing `aico-server` process over HTTP, so that `cargo test` and `cargo build` never encounter Windows binary file-locking errors (`os error 32`).
   - *Independent Test*: Run `cargo test` in `codebase/aico` while `aico-server` is actively handling MCP HTTP traffic and verify no file-locking errors occur.
4. **US-04** (Priority: P1): As a developer starting `aico-server` in a fresh workspace, I want `mcp_config.json` automatically installed into `.agents/` and the MCP URL printed in the console banner, so that client agents can connect immediately without manual configuration.
   - *Independent Test*: Run `SkillsSynchronizer::sync_workspace` and verify `.agents/mcp_config.json` is created, and inspect server startup output for `* MCP Endpoint: http://localhost:{port}/mcp`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `aico-mcp` crate shall provide a JSON-RPC 2.0 dispatcher that parses request envelopes and formats compliant JSON-RPC 2.0 responses and error structures.
- **REQ-002**: The `aico-server` crate shall expose an HTTP endpoint at `/mcp` accepting `POST`, `GET`, and `OPTIONS` methods.
- **REQ-012**: The `aico-server` crate shall expose an HTTP `OPTIONS` handler on `/mcp` returning CORS headers (`access-control-allow-origin: *`, `access-control-allow-methods: GET, POST, OPTIONS`, `access-control-allow-headers: *`).

### Event-Driven Requirements
- **REQ-003**: When a JSON-RPC request with method `initialize` is received at `/mcp`, the server shall return an `InitializeResult` containing protocol version `"2024-11-05"`, server capabilities declaring tools support, and server metadata (`name: "aico"`, `version: "0.1.1"`).
- **REQ-004**: When a JSON-RPC request with method `tools/list` is received at `/mcp`, the server shall return the tool definitions for `query_lineage_graph`, `search_hybrid`, `find_dangling_requirements`, `verify_task`, `extract_annotations`, and `validate_ears`.
- **REQ-005**: When a JSON-RPC request with method `tools/call` is received at `/mcp`, the server shall dispatch the tool call with arguments to `McpServerManager` and return the execution results in a JSON-RPC `content` payload.
- **REQ-006**: When `tools/call` executes `query_lineage_graph` or `search_hybrid`, the dispatcher shall route the query to the active project's `StorageManager` if available.
- **REQ-007**: When an HTTP `GET` request is received at `/mcp`, the server shall return HTTP 200 with server status, active project ID, and available MCP tools.
- **REQ-013**: When the host server starts, the console banner shall display the active Streamable MCP endpoint URL (`* MCP Endpoint: http://localhost:{port}/mcp`).
- **REQ-014**: When `SkillsSynchronizer::sync_workspace` executes, the synchronizer shall copy `mcp_config.json` from the skills repository to `.agents/mcp_config.json` and `.agents/skills/mcp_config.json`.

### State-Driven Requirements
- **REQ-008**: While `aico-server` is running, loopback HTTP requests to `/mcp` from `127.0.0.1` or `::1` shall be accepted without requiring external pairing tokens.

### Unwanted Behavior & Error Handling
- **REQ-009**: If a JSON-RPC request specifies an unknown method, then the server shall return JSON-RPC error code `-32601` ("Method not found").
- **REQ-010**: If a JSON-RPC request contains malformed JSON or missing required parameters, then the server shall return JSON-RPC error code `-32700` ("Parse error") or `-32602` ("Invalid params").
- **REQ-011**: If `tools/call` fails during tool execution, then the server shall return a JSON-RPC response with `isError: true` and the diagnostic error message.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **MCP JSON-RPC Protocol & Dispatcher Seam (`crates/aico-mcp/src/protocol.rs`)**:
   - Signature: `pub fn handle_json_rpc(request: &serde_json::Value, storage: Option<&StorageManager>) -> serde_json::Value`
   - Seam Test: `crates/aico-mcp/tests/mcp_protocol_test.rs`
2. **Context-Aware Storage Binding Seam (`crates/aico-mcp/src/graph_tools.rs`)**:
   - Signature: `GraphToolHandler::query_lineage_with_storage(storage: Option<&StorageManager>, symbol: &str, hops: usize)`
   - Seam Test: `crates/aico-mcp/tests/mcp_storage_binding_test.rs`
3. **Axum Streamable HTTP Handler Seam (`crates/aico-server/src/server.rs`)**:
   - Signature: `pub async fn mcp_post_handler(State(state): State<AppState>, Json(payload): Json<Value>) -> impl IntoResponse`
   - Seam Test: `crates/aico-server/tests/mcp_http_test.rs`
4. **CORS Preflight & Banner Seam (`crates/aico-server/tests/mcp_http_test.rs`)**:
   - Signature: `test_mcp_options_cors`
   - Seam Test: `crates/aico-server/tests/mcp_http_test.rs::test_mcp_options_cors`
5. **SkillsSynchronizer MCP Config Seam (`crates/aico-core/src/skills_sync.rs`)**:
   - Signature: `SkillsSynchronizer::sync_from_source(source_skills_dir, root_dir) -> Result<SyncReport, SkillsSyncError>`
   - Seam Test: `crates/aico-core/tests/scanner_test.rs::test_skills_synchronizer_syncs_mcp_config`

---

## Out-of-Scope

- Legacy MCP SSE transport (`/sse`), which is deprecated in favor of Streamable HTTP.
- Modifying client-side WebUI React components.
- Stdio transport process spawning in `aico-server`.

---

## 5. Open Clarifications

None. Ratified in [ADR-0001](../../../../adr/0001-host-runtime-and-network-topology.md) and [ADR-0003](../../../../adr/0003-security-and-auto-mode-enforcement.md).
