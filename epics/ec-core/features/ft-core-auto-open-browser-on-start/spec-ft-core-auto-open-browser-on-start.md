---
id: spec-ft-core-auto-open-browser-on-start
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 24
title: "Auto Open WebUI with Default Browser on Server Start"
status: "Completed"
derives_from: ["spec-ft-core-host-runtime", "ADR-0001", "ADR-0004"]
created_at: "2026-10-30T19:30:00.000Z"
---

# Feature Specification: Auto Open WebUI with Default Browser on Server Start

## 1. Problem Statement & User Value

When launching `aico-server` via the command line or terminal, developers currently must manually copy the `http://localhost:<port>/` URL or switch to their web browser and type the address. For a local desktop companion and spec-driven IDE, opening the primary workspace WebUI automatically upon server readiness provides an immediate, streamlined user experience.

However, automated testing (`cargo test`), headless environments, and CI runners must never trigger browser launches. Therefore, the capability must be strictly opt-out capable, defaulted to `false` during automated tests, and non-fatal if no desktop environment is present.

---

## In-Scope

- Implementing a cross-platform detached process launcher `open_browser(url: &str) -> std::io::Result<()>` in `crates/aico-server/src/server.rs` using `rundll32 url.dll,FileProtocolHandler` (Windows), `open` (macOS), and `xdg-open` (Linux).
- Adding `pub auto_open_browser: bool` to `ServerConfig` defaulting to `false` in `ServerConfig::default()` to guarantee test isolation.
- Supporting `--no-open` command line argument and `AICO_NO_BROWSER=1` / `AICO_AUTO_OPEN=false` environment variables in `crates/aico-server/src/main.rs`.
- Triggering `open_browser` targeting `http://localhost:<port>/` upon successful server port binding in `crates/aico-server/src/main.rs`.
- Gracefully logging warnings without aborting or crashing if the browser opener process fails.
- Adding regression tests to ensure automated tests do not open browsers and config defaults are preserved.

---

## Out-of-Scope

- Modifying the frontend application code or React state.
- Remote pairing auto-launching (pairing QR codes / LAN URLs remain opt-in and accessible in terminal/dashboard).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer running `cargo run -p aico-server`, I want my default web browser to automatically open to `http://localhost:4242/` as soon as the server is ready, so that I can immediately start interacting with the WebUI.
   - *Independent Test*: Start `aico-server` locally; verify that the operating system default browser opens with the local WebUI URL.
2. **US-02** (Priority: P1): As an automated test runner executing `cargo test`, I want the server test suite to execute headlessly without spawning browser windows, so that tests run quietly and reliably in CI.
   - *Independent Test*: Run `cargo test --manifest-path codebase/aico/Cargo.toml`; verify all tests pass without opening browser windows.
3. **US-03** (Priority: P2): As a developer running the server in an SSH or headless session, I want to pass `--no-open` or set `AICO_NO_BROWSER=1` so that no browser launch is attempted.
   - *Independent Test*: Run the server with `--no-open`; verify that no browser process is launched.

---

## 3. Requirements (EARS Syntax)

- **REQ-001**: The system shall provide a cross-platform helper function `open_browser(url: &str) -> std::io::Result<()>` that launches the system default web browser with the specified URL using `rundll32 url.dll,FileProtocolHandler` on Windows, `open` on macOS, and `xdg-open` on Linux.
- **REQ-002**: When `ServerConfig` is instantiated via `ServerConfig::default()`, the system shall set `auto_open_browser` to `false` so that test fixtures do not spawn browser windows.
- **REQ-003**: When `aico-server` initializes in `crates/aico-server/src/main.rs`, the server shall enable `auto_open_browser` by default unless the command line contains `--no-open` or the environment variables `AICO_NO_BROWSER=1` or `AICO_AUTO_OPEN=false` are detected.
- **REQ-004**: When `auto_open_browser` is true and the server successfully binds to its port, the server shall invoke `open_browser()` with the resolved `http://localhost:<port>/` URL in a detached background thread.
- **REQ-005**: Where invoking `open_browser()` results in an operating system error (such as headless environments without a display server), the system shall record a warning via `tracing::warn!` and continue server execution without failing.

---

## 4. Designated Public Test Seams

1. **`server_test::test_server_config_auto_open_defaults_to_false`**:
   - Seam: `crates/aico-server/tests/server_test.rs`
   - Verifies: `ServerConfig::default().auto_open_browser == false`.
   - Tags: `@verifies ft-core-auto-open-browser-on-start::TASK-CORE-024-2::REQ-002`

2. **`server_test::test_browser_opener_helper_handles_valid_url`**:
   - Seam: `crates/aico-server/tests/server_test.rs`
   - Verifies: `open_browser` command invocation structure without error.
   - Tags: `@verifies ft-core-auto-open-browser-on-start::TASK-CORE-024-1::REQ-001`, `@verifies ft-core-auto-open-browser-on-start::TASK-CORE-024-1::REQ-005`
