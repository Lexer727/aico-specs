---
id: spec-ft-core-host-runtime
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 1
title: "Host Runtime & Local HTTP/WebSocket Server"
status: "Completed"
derives_from: ["ADR-0001", "ADR-0003", "ADR-0005"]
created_at: "2026-10-05T18:15:00Z"
---

# Feature Specification: Host Runtime & Local HTTP/WebSocket Server


---

## 1. Problem Statement & User Value

Developers require a zero-daemon local engineering environment that runs embedded vector and graph databases in-process, without the operational overhead of external Docker containers or JVM background services. Furthermore, developers need to interact with the environment through their preferred desktop browsers across multiple concurrent windows and from mobile devices over the local Wi-Fi/LAN, with secure, friction-free authentication.

This specification defines the core Host Runtime: an embedded Rust Axum HTTP/WebSocket server that serves the compiled React WebUI bundle, authenticates local and LAN clients, and multiplexes real-time streaming agent sessions.

---

## In-Scope
- Axum-based HTTP and WebSocket local host server bound to `127.0.0.1` and LAN interfaces.
- Single-PID lockfile enforcement preventing concurrent server process collisions.
- Session cookie and pairing token security validation for LAN clients.
- Graceful shutdown draining active WebSocket connections and background tasks.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want to execute `aico` to boot an in-process HTTP and WebSocket server, so that I can open the WebUI in any desktop browser without configuring external server dependencies.
   - *Independent Test*: Run the binary, issue an HTTP GET to `/health`, and receive a `200 OK` response with engine status.
2. **US-02** (Priority: P1): As a desktop user, I want loopback connections (`localhost`) to authenticate automatically, so that I can access the WebUI immediately without manual password entry.
   - *Independent Test*: Request `/` from `127.0.0.1` and verify that the server issues a valid session cookie and serves the single-page application.
3. **US-03** (Priority: P1): As a mobile user, I want to pair my phone by scanning a terminal QR code or entering a short numeric token, so that I can securely control the application over local Wi-Fi.
   - *Independent Test*: Issue an HTTP request from a non-loopback IP with the pairing token query parameter and verify session authorization.
4. **US-04** (Priority: P2): As a developer, I want to open multiple browser windows on different initiatives concurrently, so that I can work on separate tasks in parallel without state collisions.
   - *Independent Test*: Open two distinct WebSocket connections with separate client instance IDs and verify independent message dispatching.
5. **US-05** (Priority: P2): As a user installing AICO, I want the WebUI assets embedded directly in the compiled binary, so that I do not need a separate Node.js or static asset web server.
   - *Independent Test*: Verify that index.html and compiled JS/CSS assets are served directly from binary memory.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The HostRuntime shall bind an HTTP listener to port `4242` on `0.0.0.0` upon process initialization.
- **REQ-002**: The StaticAssetHandler shall serve the embedded WebUI HTML, CSS, JavaScript, and font assets directly from executable memory.
- **REQ-003**: The WebSocketManager shall multiplex bidirectional JSON-RPC messages over an established WebSocket connection at path `/ws`.

### Event-Driven Requirements
- **REQ-004**: When an HTTP request originates from a loopback address (`127.0.0.1` or `::1`), the Authenticator shall issue a signed loopback session cookie without prompting for credentials.
- **REQ-005**: When the host runtime boots with LAN listening enabled, the TerminalNotifier shall print a local pairing URL and render an ASCII pairing QR code to stdout.
- **REQ-006**: When a remote client presents a valid pairing token via URL parameter `?token=<value>` or `Authorization: Bearer <value>`, the Authenticator shall upgrade the connection and issue a persistent session cookie.
- **REQ-007**: When a client establishes a WebSocket connection with an existing session cookie, the SessionManager shall synchronize the active workspace state to that client within 100 milliseconds.

### State-Driven Requirements
- **REQ-008**: While a background agent task is executing, the EventStreamer shall broadcast task progress events to all connected WebSocket clients subscribed to the active epic.
- **REQ-009**: While multiple client windows are connected simultaneously, the SessionManager shall maintain isolated tab, navigation, and view states per client connection.

### Unwanted Behavior & Error Handling
- **REQ-010**: If a remote client request lacks a valid pairing token and lacks a valid session cookie, then the Authenticator shall reject the request with HTTP status `401 Unauthorized`.
- **REQ-011**: If the configured listening port is already occupied by another process, then the HostRuntime shall emit an error message indicating the occupied port and terminate the process with exit code `1`.
- **REQ-012**: If an active WebSocket connection disconnects unexpectedly, then the SessionManager shall retain background agent task execution without aborting the running task.

### Optional Feature Requirements
- **REQ-013**: Where the `--port <number>` CLI flag is specified, the HostRuntime shall bind the HTTP listener to the specified port number.
- **REQ-014**: Where the `--no-lan` CLI flag is specified, the HostRuntime shall bind the HTTP listener strictly to loopback address `127.0.0.1`.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Server Bootstrap Seam (`aico_core::server::HostServer`)**:
   - Signature: `pub async fn start(config: ServerConfig) -> Result<ServerHandle, ServerError>`
   - Allows integration tests to spin up the Axum server on an ephemeral port (`port: 0`), test full HTTP/WebSocket round-trips, and shut down gracefully.
2. **Authentication Seam (`aico_core::auth::Authenticator`)**:
   - Signature: `pub fn validate_request(&self, peer_ip: IpAddr, headers: &HeaderMap, query_token: Option<&str>) -> AuthResult`
   - Allows unit testing of loopback detection, pairing token verification, and session cookie generation without network overhead.
3. **HTTP/WS Endpoints Contract**:
   - `GET /health`: Returns JSON `{ "status": "ok", "version": "0.1.0", "graph": "ready", "lancedb": "ready" }`.
   - `GET /api/session`: Validates current session cookie.
   - `GET /ws`: WebSocket endpoint supporting JSON-RPC 2.0 frames.

---

## Out-of-Scope

- Remote public internet hosting or multi-tenant user authentication (AICO is strictly local-first and personal-LAN).
- Automatic public NAT traversal or UPnP router reconfiguration.
- Bundling a native OS WebView window (desktop browsers are the primary presentation interface).
- SSL/TLS termination inside the binary (local LAN communication uses plain HTTP; if HTTPS is desired, users place a local reverse proxy in front).

---

## 6. Open Clarifications

None. All constraints are ratified in [ADR-0001](../../../../adr/0001-host-runtime-and-network-topology.md) and [ADR-0003](../../../../adr/0003-security-and-auto-mode-enforcement.md).
