---
id: plan-ft-specs-traceability-syntax-evolution
type: plan
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 6
title: "Traceability Syntax Evolution & Codebase In-Band Annotation Standards Plan"
status: "Completed"
repositories: ["aico", "aico-specs", "aico-skills"]
derives_from: ["spec-ft-specs-traceability-syntax-evolution"]
created_at: "2026-10-17T17:01:00Z"
---

# Implementation Plan: Traceability Syntax Evolution & Codebase In-Band Annotation Standards

## 1. Technical Architecture & Decisions

- **Canonical 3-Segment Format (`REQ-001`, `REQ-002`)**:
  - Migrate in-band annotations to `// @implements ft-<feature_slug>::TASK-<task_id>::REQ-<req_id>`.
  - Migrate in-band test verifications to `// @verifies ft-<feature_slug>::TASK-<task_id>::REQ-<req_id>`.
  - Preserve comments delimiters across languages (`//`, `#`, `/* ... */`, `{/* ... */}`).

- **Documentation Placeholder Convention (`REQ-003`)**:
  - In all skill guides and markdown documentation, format example annotations with placeholders:
    `// @implements ft-<feature>::TASK-###::REQ-###`.
  - Treat `<`, `>`, and `#` as reserved non-production tokens.

- **Strict Scanner & Comment Parsing in `aico-server` (`REQ-004`, `REQ-005`)**:
  - Update `scan_codebase_files` in `crates/aico-server/src/server.rs`.
  - Use regex matching only valid comment-delimited lines with 3 segments and non-placeholder identifiers.
  - Retain `.md` in file extension checks, but only return files with `implements_count > 0`.

- **Traceability Verification Engine Upgrade (`REQ-006`)**:
  - Update `verify-artifacts.mjs` in `aico-skills/aico-verify/scripts/verify-artifacts.mjs`.
  - Parse 3-segment annotations, validating that the feature exists, the task exists in the plan, the requirement is in the spec, and the task addresses the requirement.

- **Automated Codebase Migration Utility (`REQ-007`)**:
  - Implement a migration script that reads all `plan-ft-*.md` files to construct an inverted map `(feature, req) -> task`.
  - Walk all source files in `crates/` and `frontend/`, replacing `spec-ft-<slug>::REQ-<id>` with `ft-<slug>::TASK-<task>::REQ-<id>`.

- **WebUI Shell & CodeViewer Updates (`REQ-008`)**:
  - Update `CodeViewer.tsx` line parser to recognize the 3-segment syntax and highlight annotations.
  - Update `ShellLayout.tsx` and `OmniSearchModal.tsx` search and badge logic.

---

## 2. Task Graph (DAG)

```
TASK-SPECS-006-1 (Traceability Migration Utility & Codebase Upgrade)
  ├── TASK-SPECS-006-2 (Skill Documentation Updates with Placeholders)
  │     └── TASK-SPECS-006-3 (verify-artifacts.mjs Axis 2 Traceability Upgrade)
  └── TASK-SPECS-006-4 (aico-server Strict Scanner & False-Positive Filter)
        └── TASK-SPECS-006-5 (Frontend CodeViewer & ShellLayout 3-Segment Highlighting)
              └── TASK-SPECS-006-6 (Full Suite Regression & Blueprint Verification)
```

### Phase 1: Migration Tooling & Codebase Upgrade

- **TASK-SPECS-006-1**: Implement Migration Script & Upgrade Codebase Annotations
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-007
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Creates migration script parsing all `plan-ft-*.md` in `aico-specs` to map `(feature, req) -> task`.
    - Rewrites legacy `// @implements spec-ft-<slug>::REQ-<id>` to `// @implements ft-<slug>::TASK-<id>::REQ-<id>`.
    - Rewrites legacy `// @verifies spec-ft-<slug>::REQ-<id>` to `// @verifies ft-<slug>::TASK-<id>::REQ-<id>`.
    - Applies transformation across all source and test files in `crates/` and `frontend/`.
  - **Verification Seam**: Git diff audit and compilation check across crates and frontend.

### Phase 2: Skills Documentation Standardization

- **TASK-SPECS-006-2**: Update Skill Guides with Placeholder Standard
  - **Status**: Done
  - **Addresses**: REQ-003
  - **Blocked by**: TASK-SPECS-006-1
  - **What it delivers**:
    - Updates `aico-implementation/SKILL.md`, `aico-verify/SKILL.md`, `aico-code-to-spec/SKILL.md`, `aico-feature/SKILL.md`, `aico-guide/SKILL.md`, and `VERIFY-CHECKLIST.md`.
    - Replaces legacy examples with canonical placeholder syntax (`// @implements ft-<feature>::TASK-###::REQ-###`).
  - **Verification Seam**: Text search across `aico-skills` ensuring zero un-placeholdered production-style annotations in markdown.

### Phase 3: Artifact Verification Engine Upgrade

- **TASK-SPECS-006-3**: Upgrade `verify-artifacts.mjs` Axis 2 Traceability Engine
  - **Status**: Done
  - **Addresses**: REQ-006
  - **Blocked by**: TASK-SPECS-006-2
  - **What it delivers**:
    - Updates `implementsRegex` and `verifiesRegex` in `verify-artifacts.mjs` to extract `(feature, task, req)`.
    - Enforces that the task exists in the plan and declares the requirement in its `addresses:` list.
    - Confirms 100% requirement coverage across the codebase.
  - **Verification Seam**: `node aico-verify/scripts/verify-artifacts.mjs --workspace D:\aico\aico-specs`

### Phase 4: Backend Scanner Hardening

- **TASK-SPECS-006-4**: Implement Strict Comment Regex & False-Positive Filter in `aico-server`
  - **Status**: Done
  - **Addresses**: REQ-004, REQ-005
  - **Blocked by**: TASK-SPECS-006-1
  - **What it delivers**:
    - Updates `scan_codebase_files` in `crates/aico-server/src/server.rs` with strict regex parsing.
    - Rejects `<...>` and `###` placeholder tokens.
    - Excludes files with 0 valid annotations from the returned list while preserving `.md` extension support for annotated files.
  - **Verification Seam**: `crates/aico-server/tests/code_scanner_test.rs`

### Phase 5: Frontend Visualization

- **TASK-SPECS-006-5**: Update CodeViewer & ShellLayout for 3-Segment Syntax
  - **Status**: Done
  - **Addresses**: REQ-008
  - **Blocked by**: TASK-SPECS-006-4
  - **What it delivers**:
    - Updates `CodeViewer.tsx` to detect 3-segment annotations, highlight lines, and format badges.
    - Ensures `ShellLayout.tsx` tree view and search correctly display `@impl` badges for 3-segment annotations.
  - **Verification Seam**: `frontend/tests/CodeViewer.test.tsx`, `frontend/tests/ShellLayoutSearch.test.tsx`

### Phase 6: System Regression & Blueprint Audit

- **TASK-SPECS-006-6**: Full Suite Regression Verification & Production Build
  - **Status**: Done
  - **Addresses**: REQ-001 through REQ-008
  - **Blocked by**: TASK-SPECS-006-3, TASK-SPECS-006-5
  - **What it delivers**:
    - Run Vitest suite across frontend (`npm test`).
    - Run Cargo test suite across crates (`cargo test --workspace`).
    - Run production build (`npm run build`).
    - Run `verify-artifacts.mjs` ensuring 0 errors and 0 warnings.
  - **Verification Seam**: Vitest + Cargo + `npm run build` + `verify-artifacts.mjs`
