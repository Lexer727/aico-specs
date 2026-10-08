---
id: plan-ft-specs-skills-versioning-and-naming
type: plan
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 4
title: "AICO Skills Version Decoupling & Artifact Terminology Standardization Plan"
status: "Completed"
repositories: ["aico-skills", "aico-specs"]
derives_from: ["spec-ft-specs-skills-versioning-and-naming"]
created_at: "2026-10-07T00:10:00Z"
---

# Implementation Plan: AICO Skills Version Decoupling & Artifact Terminology Standardization

## 1. Technical Architecture & Decisions

- **Single Source of Version Truth**: The `aico-skills` version is single-sourced in `aico-skills/VERSION`. Artifact templates must never hardcode version numbers; instead, they declare `{aico_skills_version}`, which is resolved dynamically by agents and tooling at runtime.
- **Terminology Standardization**: "SDD artifacts" is replaced across all documentation, skill instructions, and reports with "AICO artifacts" or "AICO-skill artifacts" to establish consistent branding and platform identity.
- **Decoupled Platform & Skills Display**: The WebUI Shell cleanly delineates the Platform Version (`v0.1.1` from Cargo workspace) from the Skills Version (`0.1.2` from `VERSION`), updating the settings modal section to "AICO Skills".

---

## 2. Task Graph (DAG)

### Phase 1: Nomenclature & Skill Cleanup

- **TASK-SPECS-004-1**: Standardize Artifact Nomenclature to AICO Artifacts
  - **Status**: Done
  - **Addresses**: `REQ-001`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**: Replaced instances of "SDD artifacts" with "AICO artifacts" across `aico-verify/SKILL.md`, `aico-verify/VERIFY-CHECKLIST.md`, `aico-code-to-spec/SKILL.md`, `README.md`, and `verify-artifacts.mjs`.
  - **Verification Seam**: `aico-verify/scripts/verify-artifacts.mjs`

- **TASK-SPECS-004-2**: Dynamic Version Resolution & Template Placeholder Injection
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`
  - **Blocked by**: `TASK-SPECS-004-1`
  - **What it delivers**: Added `aico-skills/scripts/get-version.mjs`, replaced hardcoded `0.1.2` with `{aico_skills_version}` in `SPEC-TEMPLATE.md`, `PLAN-TEMPLATE.md`, and `ADR-FORMAT.md`, and removed hardcoded versions from skill descriptions and headers.
  - **Verification Seam**: `aico-skills/scripts/get-version.mjs`

### Phase 2: WebUI Shell & Settings Alignment

- **TASK-SPECS-004-3**: WebUI Settings Modal & Nav Alignment
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-005`
  - **Blocked by**: `TASK-SPECS-004-2`
  - **What it delivers**: Renamed settings modal section to "AICO Skills" and version label to "AICO Skills Version", added feature to `ShellLayout.tsx` nav items, and updated `SettingsModal.test.tsx` assertions.
  - **Verification Seam**: `frontend/tests/SettingsModal.test.tsx`

### Phase 3: Runtime MCP Tool Dynamic Versioning

- **TASK-SPECS-004-4**: MCP Artifact Creation Dynamic Version Resolution
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-005`, `REQ-006`
  - **Blocked by**: `TASK-SPECS-004-3`
  - **What it delivers**: Implemented `ArtifactToolGate::resolve_skills_version()` in `aico-mcp` to dynamically read from `VERSION` files or environment variables with fallback, and verified clean test passage.
  - **Verification Seam**: `crates/aico-mcp/tests/mcp_gate_test.rs`
