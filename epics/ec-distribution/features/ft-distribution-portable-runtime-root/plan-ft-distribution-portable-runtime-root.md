---
id: plan-ft-distribution-portable-runtime-root
type: plan
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 1
title: "Portable Runtime Root Resolution & Bundled Skills Source Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-distribution-portable-runtime-root"]
created_at: "2026-10-30T19:19:08.717Z"
---

# Implementation Plan: Portable Runtime Root Resolution & Bundled Skills Source

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Root resolver**: New module `crates/aico-core/src/root.rs` exporting `resolve_aico_root(...)` and `RootResolutionError`. Pure function (all inputs injected) so the precedence matrix is testable without touching process state. Re-exported from `crates/aico-core/src/lib.rs`.
- **Process-wide root**: `main.rs` parses `--root`/`--version` from `std::env::args()`, calls the resolver with `std::env::current_exe()` and `std::env::current_dir()`, and stores the result in a `std::sync::OnceLock<PathBuf>` exposed as `aico_core::root::aico_root()`. `resolve_workspace_root_from_specs_path` (server.rs) delegates to it, removing its `d:/aico` fallback.
- **Specs fallback**: `get_effective_specs_path` (assets.rs) returns `<aico_root>/specs` instead of `d:/aico/aico-specs` when no project is active.
- **Skills fallback**: `SkillsSynchronizer::resolve_skills_source(root, exe_dir)` returns the dev submodule if it contains at least one `*/SKILL.md`, else `<exe_dir>/skills` if it does, else `None`. `sync_workspace` gains an `exe_dir` parameter (call sites updated).
- **Asset policy**: `disk_assets_enabled()` returns `cfg!(debug_assertions) || AICO_DEV_ASSETS == "1"`; `static_asset_handler` skips the disk loops otherwise.
- **Logging**: Add `tracing-appender` (non-rolling file writer) layered with stdout in `main.rs`; logs at `<root>/.aico/logs/aico-server.log`.
- **Verification**: `cargo test --workspace`, `cargo clippy --workspace`, `verify-artifacts.mjs`.

---

## 2. Task Graph (DAG)

### Phase 1: Core Vertical Tracer Bullet

- **TASK-DISTRIBUTION-001-1**: AICO Root Resolver, `--root`/`--version` CLI & Hard-Coded Path Removal
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-005`, `REQ-006`, `REQ-008`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `crates/aico-core/src/root.rs` with `resolve_aico_root` and `aico_root()` `OnceLock` accessor.
    - `main.rs` argument parsing for `--root` and `--version`; banner uses `env!("CARGO_PKG_VERSION")`.
    - Removal of `d:/aico` / `d:/aico/aico-specs` literals in `main.rs`, `server.rs`, `assets.rs`.
    - Red-to-green tests: root precedence matrix, non-existent root error, `--version` binary test, hard-coded path guard.
  - **Verification Seam**: `crates/aico-core/tests/root_resolution_test.rs`

- **TASK-DISTRIBUTION-001-2**: Bundled Skills Source Fallback
  - **Status**: Done
  - **Addresses**: `REQ-003`
  - **Blocked by**: `TASK-DISTRIBUTION-001-1`
  - **What it delivers**:
    - `SkillsSynchronizer::resolve_skills_source(root, exe_dir)` and updated `sync_workspace` signature.
    - `main.rs` passes `current_exe().parent()` as `exe_dir`.
    - Tests for dev-submodule precedence, bundled fallback, and graceful no-source behaviour.
  - **Verification Seam**: `crates/aico-core/tests/skills_sync_test.rs`

### Phase 2: Hardening

- **TASK-DISTRIBUTION-001-3**: Embedded-Only Asset Policy for Release Builds
  - **Status**: Done
  - **Addresses**: `REQ-004`
  - **Blocked by**: `TASK-DISTRIBUTION-001-1`
  - **What it delivers**:
    - `disk_assets_enabled()` helper and guarded disk lookup loops in `static_asset_handler`.
    - Test asserting the policy truth table.
  - **Verification Seam**: `crates/aico-server/tests/assets_test.rs`

- **TASK-DISTRIBUTION-001-4**: File Logging into AICO Home
  - **Status**: Done
  - **Addresses**: `REQ-007`
  - **Blocked by**: `TASK-DISTRIBUTION-001-1`
  - **What it delivers**:
    - `tracing-appender` workspace dependency and dual stdout/file subscriber in `main.rs`.
    - Helper `aico_core::root::logs_dir(root)` creating `<root>/.aico/logs`.
    - Test asserting directory creation and log file path.
  - **Verification Seam**: `crates/aico-core/tests/root_resolution_test.rs`
