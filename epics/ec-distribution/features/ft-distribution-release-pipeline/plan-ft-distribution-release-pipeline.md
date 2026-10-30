---
id: plan-ft-distribution-release-pipeline
type: plan
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 2
title: "Automated GitHub Actions Release Pipeline Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-distribution-release-pipeline"]
created_at: "2026-10-30T19:19:08.895Z"
---

# Implementation Plan: Automated GitHub Actions Release Pipeline

## 1. Technical Architecture & Decisions

- **Target repository**: `aico`.
- **Cross-feature prerequisite**: `ft-distribution-portable-runtime-root` (`TASK-DISTRIBUTION-001-1`, `TASK-DISTRIBUTION-001-2`) must be Done so the packaged binary is location-independent and consumes the bundled `skills/`.
- **Scripts** (PowerShell, runnable locally and on `windows-latest`):
  - `scripts/check-release-version.ps1`: parses `[workspace.package] version` from `Cargo.toml`, strips leading `v` from the tag, compares.
  - `scripts/package-release.ps1`: stages `aico-server.exe`, `skills/` (robocopy excluding `.git`), optional `prerequisites.json`; computes `Get-FileHash -Algorithm SHA256` per file; writes `manifest.json`; zips via `Compress-Archive`; writes `.sha256`.
  - Pester tests under `scripts/tests/`.
- **Workflow** `.github/workflows/release.yml`:
  - Triggers: `push: tags: ['v*']`, `workflow_dispatch`.
  - Steps: checkout (`submodules: recursive`) → version gate (tag builds only) → `actions/setup-node` (LTS, npm cache) → `npm ci && npm run build` in `frontend/` → `dtolnay/rust-toolchain@stable` + `Swatinem/rust-cache` → install `protoc` if required by LanceDB → `cargo build --release -p aico-server` → package → (installer build hook) → `actions/upload-artifact` → on tags: `softprops/action-gh-release` with `generate_release_notes: true`, `prerelease: ${{ contains(github.ref_name, '-') }}`.
  - Permissions: `contents: write`.
- **Verification**: Pester suite locally, `workflow_dispatch` dry run, then a `v0.2.0-rc.1` prerelease tag.

---

## 2. Task Graph (DAG)

### Phase 1: Core Vertical Tracer Bullet

- **TASK-DISTRIBUTION-002-1**: Release Version Gate Script
  - **Status**: Done
  - **Addresses**: `REQ-002`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `scripts/check-release-version.ps1` with Pester tests for match, mismatch, and `-rc.N` suffix handling.
  - **Verification Seam**: `scripts/tests/check-release-version.Tests.ps1`

- **TASK-DISTRIBUTION-002-2**: Release Bundle Packaging Script, Manifest & Checksum
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-004`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - `scripts/package-release.ps1` producing zip, `manifest.json`, `.sha256`.
    - Pester tests using a fake binary and fake skills tree asserting layout, manifest schema, per-file hashes, `.git` exclusion, and checksum correctness.
  - **Verification Seam**: `scripts/tests/package-release.Tests.ps1`

- **TASK-DISTRIBUTION-002-3**: Release Workflow with Dry Run & GitHub Release Publishing
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-005`, `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-DISTRIBUTION-002-1`, `TASK-DISTRIBUTION-002-2`
  - **What it delivers**:
    - `.github/workflows/release.yml` with build, cache, package, artifact upload, and tag-gated release publishing (prerelease detection).
    - Installer build hook step (no-op until `ft-distribution-windows-installer` supplies `installer/aico.iss`).
    - Successful `workflow_dispatch` dry run and `v0.2.0-rc.1` prerelease as human-observed evidence.
  - **Verification Seam**: `.github/workflows/release.yml`
