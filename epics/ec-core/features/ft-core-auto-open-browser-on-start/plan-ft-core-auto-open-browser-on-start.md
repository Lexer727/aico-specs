---
id: plan-ft-core-auto-open-browser-on-start
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 24
title: "Auto Open WebUI with Default Browser on Server Start Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-core-auto-open-browser-on-start"]
created_at: "2026-10-30T19:30:00.000Z"
---

# Implementation Plan: Auto Open WebUI with Default Browser on Server Start

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Cross-Platform Opener Helper**:
  - Implement `open_browser(url: &str) -> std::io::Result<()>` in `crates/aico-server/src/server.rs`.
  - Re-use proven non-blocking execution:
    - Windows: `rundll32 url.dll,FileProtocolHandler <url>`.
    - macOS: `open <url>`.
    - Linux: `xdg-open <url>`.
  - Re-export `open_browser` from `crates/aico-server/src/lib.rs`.
- **Config & Test Safety**:
  - Add `pub auto_open_browser: bool` to `ServerConfig` in `crates/aico-server/src/server.rs`.
  - Set `auto_open_browser: false` in `ServerConfig::default()` so all existing and future test suites run completely headlessly.
- **Opt-Out & Startup Trigger**:
  - In `crates/aico-server/src/main.rs`:
    - Check if command-line arguments contain `--no-open`.
    - Check if `AICO_NO_BROWSER=1` or `AICO_AUTO_OPEN=false` is set in the environment.
    - Set `config.auto_open_browser = true` if none of the opt-out signals are present.
    - Following `let addr = server.bind().await?;`, if `config.auto_open_browser` is true, invoke `open_browser(&format!("http://localhost:{}/", addr.port()))` in a background spawn or thread, catching any `Err` and logging via `tracing::warn!`.
- **Verification**:
  - Add regression tests in `crates/aico-server/tests/server_test.rs`.
  - Run full test suite (`cargo test`, `npm test`, `npm run build`, `verify-artifacts.mjs`).

---

## 2. Task Graph (DAG)

- **TASK-CORE-024-1**: Implement Cross-Platform Browser Opener Helper
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-005`
  - **Blocked by**: None
  - **What it delivers**:
    - Defines `open_browser(url: &str) -> std::io::Result<()>` in `crates/aico-server/src/server.rs` with platform-specific execution and exports it via `crates/aico-server/src/lib.rs`.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

- **TASK-CORE-024-2**: Add `auto_open_browser` to `ServerConfig` and Opt-Out Parsing
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`
  - **Blocked by**: None
  - **What it delivers**:
    - Adds `pub auto_open_browser: bool` to `ServerConfig` defaulting to `false`.
    - Parses `--no-open` argument and `AICO_NO_BROWSER` / `AICO_AUTO_OPEN` environment variables in `crates/aico-server/src/main.rs`.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

- **TASK-CORE-024-3**: Trigger WebUI Auto-Open on Server Port Binding
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-CORE-024-1`, `TASK-CORE-024-2`
  - **What it delivers**:
    - Calls `open_browser(&format!("http://localhost:{}/", addr.port()))` in `crates/aico-server/src/main.rs` when `auto_open_browser` is enabled, logging any failure as a warning.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`

- **TASK-CORE-024-4**: Unit Testing, Test Suite Verification & Non-Regression
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`
  - **Blocked by**: `TASK-CORE-024-3`
  - **What it delivers**:
    - Unit tests in `crates/aico-server/tests/server_test.rs` verifying default configuration and opener behavior, ensuring green test pipeline.
  - **Verification Seam**: `crates/aico-server/tests/server_test.rs`
