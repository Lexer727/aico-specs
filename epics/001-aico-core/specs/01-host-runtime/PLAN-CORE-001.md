# Implementation Plan: Host Runtime & Local Axum Server

- **ID**: PLAN-CORE-001
- **Spec Reference**: [SPEC-CORE-001.md](file:///d:/aico/aico-specs/epics/001-aico-core/specs/01-host-runtime/SPEC-CORE-001.md)
- **Status**: Ready

---

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-server` (part of the AICO Rust Cargo workspace).
- **Core Dependencies**:
  - `axum`: High-performance asynchronous HTTP and WebSocket server framework on top of `tokio` and `tower`.
  - `tokio`: Async runtime with signal handling for graceful shutdown (`tokio::signal::ctrl_c`).
  - `tower-http`: CORS, tracing, compression, and request routing middleware.
  - `rust-embed`: Zero-external-file static asset bundling for the React WebUI build artifacts.
  - `qrcode`: Generator for pairing QR codes in terminal stdout and `/api/v1/auth/qr` endpoint.
  - `tracing` / `tracing-subscriber`: Structured asynchronous diagnostic logging with JSON/compact formatting.
- **Port Strategy**: Default `4242`. If occupied, sequentially probe `4243..=4252` before failing.

---

## 2. Task Graph (DAG)

### Phase 1: Prefactoring & Scaffolding (Unblocked Frontier)

- [x] **TASK-CORE-001-1**: Axum Server Scaffolding & Dynamic Port Binding
  - **Addresses**: `REQ-001`, `REQ-006`, `REQ-010`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Creates `crates/aico-server` with `HostServer` struct.
    - Implements port resolution starting at `4242` with fallback probing.
    - Adds `GET /api/v1/health` returning JSON uptime and status.
    - Integrates `tracing` subscriber for structured operational logs.
  - **Verification Seam**: `tests/host_runtime_test.rs::test_server_binds_and_health_check`

---

### Phase 2: Static Asset Serving & SPA Routing

- [ ] **TASK-CORE-001-2**: Embedded Static Asset Serving with SPA Fallback
  - **Addresses**: `REQ-002`
  - **Blocked by**: `TASK-CORE-001-1`
  - **What it delivers**:
    - Configures `rust-embed` macro to bundle `frontend/dist`.
    - Implements custom Axum handler serving matching MIME types (`text/html`, `application/javascript`, `image/svg+xml`).
    - Implements SPA wildcard fallback returning `index.html` for client-side routing.
  - **Verification Seam**: `tests/host_runtime_test.rs::test_embedded_spa_fallback`

---

### Phase 3: Real-Time WebSocket Infrastructure

- [x] **TASK-CORE-001-3**: WebSocket Hub, Client State & Heartbeat Loop
  - **Addresses**: `REQ-003`, `REQ-007`, `REQ-009`
  - **Blocked by**: `TASK-CORE-001-1`
  - **What it delivers**:
    - Implements `WebSocketHub` managing client sessions via Tokio broadcast channels (`tokio::sync::broadcast`).
    - Mounts `GET /ws` with protocol upgrade to WebSocket.
    - Runs 30-second ping/pong heartbeat timer, evicting unresponsive clients after 60s.
    - Exposes `broadcast_event(topic, payload)` to notify all active subscribers.
  - **Verification Seam**: `tests/host_runtime_test.rs::test_websocket_broadcast_and_heartbeat`

---

### Phase 4: LAN Authentication & Pairing

- [x] **TASK-CORE-001-4**: LAN Token Authentication & Terminal QR Code Generator
  - **Addresses**: `REQ-004`, `REQ-008`
  - **Blocked by**: `TASK-CORE-001-3`
  - **What it delivers**:
    - Implements `LanAuthenticator` generating cryptographically random 256-bit pairing tokens on startup.
    - Binds server to `0.0.0.0:4242` and prints an ANSI terminal QR code with mobile URL (`http://<LAN-IP>:4242?token=...`).
    - Enforces Bearer token header or query parameter validation for all non-health HTTP endpoints and WS upgrades.
  - **Verification Seam**: `tests/host_runtime_test.rs::test_lan_token_authentication`

---

### Phase 5: Graceful Shutdown & Drain

- [x] **TASK-CORE-001-5**: Graceful Shutdown Handler & Drain Loop
  - **Addresses**: `REQ-005`
  - **Blocked by**: `TASK-CORE-001-3`
  - **What it delivers**:
    - Intercepts `SIGINT` (Ctrl+C) and `SIGTERM` signals.
    - Broadcasts `server_shutdown` message to all connected WebSocket clients.
    - Grants a 5-second graceful connection drain timeout before terminating runtime threads.
  - **Verification Seam**: `tests/host_runtime_test.rs::test_graceful_shutdown_drain`

---

## 3. Initial Active Frontier

- `TASK-CORE-001-1`: Axum Server Scaffolding & Dynamic Port Binding (Ready for `/aico-test-scaffolding`).
