---
id: plan-ft-tooling-mcp-streamable-http
type: plan
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 4
title: "Streamable HTTP MCP Transport on aico-server Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-tooling-mcp-streamable-http"]
created_at: "2026-10-22T20:24:06.575Z"
---

# Implementation Plan: Streamable HTTP MCP Transport on aico-server

## 1. Technical Architecture & Decisions

- **Transport**: Standard Model Context Protocol (MCP) Streamable HTTP (`POST /mcp` and `GET /mcp`) served by `aico-server` Axum router.
- **Protocol Dispatcher**: Zero-overhead JSON-RPC 2.0 parser and tool dispatcher in `crates/aico-mcp/src/protocol.rs` implementing `initialize`, `ping`, `tools/list`, and `tools/call`.
- **Storage Context**: Thread-safe injection of active project `StorageManager` into `AppState` and MCP handlers, enabling queries directly against Kùzu property graphs and LanceDB embeddings without file locks.
- **Client Configuration**: Global Antigravity registration via `~/.gemini/config/mcp_config.json`.

---

## 2. Task Graph (DAG)

### Phase 1: Protocol Envelope & JSON-RPC Dispatcher

- **TASK-TOOL-004-1**: Implement MCP JSON-RPC 2.0 Protocol Envelope & Tool Dispatcher
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-003`, `REQ-004`, `REQ-005`, `REQ-009`, `REQ-010`, `REQ-011`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Adds `crates/aico-mcp/src/protocol.rs` handling JSON-RPC 2.0 requests (`initialize`, `notifications/initialized`, `ping`, `tools/list`, and `tools/call`).
    - Maps `tools/list` to `McpServerManager::list_tools()` and `tools/call` to `McpServerManager::call_tool()`.
    - Handles standard error codes (`-32700`, `-32601`, `-32602`) and tool errors with `isError: true`.
  - **Verification Seam**: `crates/aico-mcp/tests/mcp_protocol_test.rs::test_handle_json_rpc`

---

### Phase 2: Context-Aware StorageManager Binding

- **TASK-TOOL-004-2**: Connect Graph & Hybrid Search Tools to Active StorageManager
  - **Status**: Done
  - **Addresses**: `REQ-005`, `REQ-006`
  - **Blocked by**: `TASK-TOOL-004-1`
  - **What it delivers**:
    - Updates `GraphToolHandler` and `HybridToolHandler` in `crates/aico-mcp/src/graph_tools.rs` to accept an optional `&StorageManager`.
    - When `StorageManager` is provided, queries run directly against the active Kùzu and LanceDB instances.
    - When storage is not available or not yet initialized, returns an informative diagnostic message rather than failing silently.
  - **Verification Seam**: `crates/aico-mcp/tests/mcp_storage_binding_test.rs::test_storage_binding`

---

### Phase 3: Axum Streamable HTTP Endpoint in aico-server

- **TASK-TOOL-004-3**: Mount Streamable HTTP `/mcp` Route and AppState Storage Bridge
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-006`, `REQ-007`, `REQ-008`
  - **Blocked by**: `TASK-TOOL-004-2`
  - **What it delivers**:
    - Adds `aico-mcp` and `aico-storage` dependencies to `crates/aico-server/Cargo.toml`.
    - Integrates `Option<Arc<StorageManager>>` into `AppState` initialized for the active project.
    - Mounts `.route("/mcp", post(mcp_post_handler).get(mcp_get_handler))` in `HostServer::build_router`.
    - Implements loopback authorization for local agent requests without token prompts.
  - **Verification Seam**: `crates/aico-server/tests/mcp_http_test.rs::test_mcp_http_endpoint`

---

### Phase 4: Global Registration & End-to-End Verification

- **TASK-TOOL-004-4**: Register Local MCP Server in mcp_config.json & Validate Discovery
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-004`
  - **Blocked by**: `TASK-TOOL-004-3`
  - **What it delivers**:
    - Updates `~/.gemini/config/mcp_config.json` with the Streamable HTTP `aico` server configuration.
    - Adds an end-to-end integration test validating server startup, client discovery, and tool call round-trip.
  - **Verification Seam**: `crates/aico-server/tests/mcp_http_test.rs::test_mcp_discovery_roundtrip`

- **TASK-TOOL-004-5**: Add CORS OPTIONS Preflight & Startup Console Banner
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-012`, `REQ-013`
  - **Blocked by**: `TASK-TOOL-004-4`
  - **What it delivers**:
    - OPTIONS /mcp handler returning standard CORS headers and MCP Endpoint line in server startup banner
  - **Verification Seam**: `crates/aico-server/tests/mcp_http_test.rs::test_mcp_options_cors`

---

- **TASK-TOOL-004-6**: Auto-Distribute mcp_config.json via SkillsSynchronizer
  - **Status**: Done
  - **Addresses**: `REQ-014`
  - **Blocked by**: `TASK-TOOL-004-5`
  - **What it delivers**:
    - Canonical mcp_config.json in aico-skills and automatic sync to workspace .agents/mcp_config.json
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs::test_skills_synchronizer_syncs_mcp_config`

- **TASK-TOOL-004-7**: End-to-End Verification of Workspace MCP Discovery & Antigravity Health
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-012`, `REQ-014`
  - **Blocked by**: `TASK-TOOL-004-6`
  - **What it delivers**:
    - End-to-end integration test validating workspace .agents/mcp_config.json sync, OPTIONS preflight, and clean Antigravity discovery
  - **Verification Seam**: `crates/aico-server/tests/mcp_http_test.rs::test_mcp_options_cors`


## 3. Amendment Log

- **2026-10-22T21:24:44.293Z**: Fixes RED status in Antigravity by supporting preflight checks, and displays MCP URL on server launch (Added `TASK-TOOL-004-5`).
- **2026-10-22T21:24:50.334Z**: Eliminates global machine configuration requirement and ensures every workspace has local MCP discovery on startup (Added `TASK-TOOL-004-6`).
- **2026-10-22T21:24:56.287Z**: Confirms that local workspace MCP discovery and CORS preflight turn green without manual setup (Added `TASK-TOOL-004-7`).
