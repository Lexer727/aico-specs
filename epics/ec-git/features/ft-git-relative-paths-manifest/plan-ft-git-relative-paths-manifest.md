---
id: plan-ft-git-relative-paths-manifest
type: plan
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 4
title: "Manifest Relative Repository Paths & Workspace Portability Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-git-relative-paths-manifest"]
created_at: "2026-10-07T01:06:00Z"
---

# Implementation Plan: Manifest Relative Repository Paths & Workspace Portability

## 1. Technical Architecture & Decisions

- **Portable Relative Paths**: Update `.aico-project.json` to declare `"../codebase/aico"`, `"."`, and `"../codebase/aico-skills"`.
- **Manifest Normalization**: Ensure `ProjectManifest::get_effective_repositories` in `crates/aico-core/src/manifest.rs` maps `.` and relative components cleanly through `resolve_path`.
- **Test Seam Verification**: Add `test_manifest_relative_repositories_resolution` in `crates/aico-core/tests/manifest_test.rs`.

---

## 2. Task Graph (DAG)

### Phase 1: Manifest Relative Paths

- **TASK-GIT-004-1**: Configure Relative Paths in `.aico-project.json`
  - **Status**: Done
  - **Addresses**: REQ-001
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Update `d:/aico/aico-specs/.aico-project.json` with `../codebase/aico`, `.`, and `../codebase/aico-skills`.
  - **Verification Seam**: `.aico-project.json`

### Phase 2: Core Resolution & Verification Seam

- **TASK-GIT-004-2**: Relative Path Resolution Seam Test in `aico-core`
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-003, REQ-004
  - **Blocked by**: TASK-GIT-004-1
  - **What it delivers**:
    - Add `test_manifest_relative_repositories_resolution` to `crates/aico-core/tests/manifest_test.rs`.
    - Verify `cargo test -p aico-core`.
  - **Verification Seam**: `crates/aico-core/tests/manifest_test.rs::test_manifest_relative_repositories_resolution`
