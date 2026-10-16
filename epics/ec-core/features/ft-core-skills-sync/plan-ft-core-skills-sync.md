---
id: plan-ft-core-skills-sync
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 12
title: "Startup Workspace Skills & Agent Rules Synchronization Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-skills"]
derives_from: ["spec-ft-core-skills-sync"]
created_at: "2026-10-16T20:35:00Z"
---

# Implementation Plan: Startup Workspace Skills & Agent Rules Synchronization

## 1. Technical Architecture & Decisions

- **Submodule Alignment (`codebase/aico/skills`)**:
  - Fast-forward the `codebase/aico/skills` submodule pointer to the latest `aico-skills` commit (`d654ddb`) so its packaged `rules/GEMINI.md`, `rules/AGENTS.md`, and root `GEMINI.md` reflect the canonical 4-phase review gate.
- **Core Synchronizer Module (`crates/aico-core/src/skills_sync.rs`)**:
  - `SkillsSynchronizer::sync_workspace(root_dir)` resolves `<root_dir>/codebase/aico/skills` and delegates to `SkillsSynchronizer::sync_from_source(source_skills_dir, root_dir)`.
  - Enumerates all immediate subdirectories of `source_skills_dir` containing a `SKILL.md` file and recursively synchronizes their contents to `<root_dir>/.agents/skills/<skill_name>/`, pruning any removed files inside those skill folders.
  - Copies supporting root files (`VERSION`, `README.md`) and the `scripts/` directory into `<root_dir>/.agents/skills/` so tools like `aico-verify/scripts/verify-artifacts.mjs` resolve `../../VERSION` and helper scripts cleanly.
  - Prunes stale `aico-*` directories in `<root_dir>/.agents/skills/` that are no longer present in `source_skills_dir`, and removes any stray `<root_dir>/.agents/skills/SKILL.md` file.
  - Synchronizes `rules/GEMINI.md` (falling back to `GEMINI.md`) to `<root_dir>/GEMINI.md` and `<root_dir>/.agents/rules/GEMINI.md`, and synchronizes `rules/AGENTS.md` to `<root_dir>/.agents/rules/AGENTS.md`.
  - Performs byte-equality checks prior to writing any file so unchanged files are not rewritten on subsequent startups (`SyncReport` tracks `files_updated`, `files_unchanged`, and `entries_removed`).
- **Host Server Boot Integration (`crates/aico-server/src/main.rs`)**:
  - Invokes `SkillsSynchronizer::sync_workspace(&root_dir)` immediately after resolving `root_dir` on server startup.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

### Phase 1: Submodule Alignment & Core Skills Synchronization

- **TASK-SYNC-001**: Fast-forward `codebase/aico/skills` submodule to latest `aico-skills` commit
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-005, REQ-006
  - **Blocked by**: None
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs`
  - **What it delivers**:
    - Updates `codebase/aico/skills` submodule to the latest `aico-skills` commit (`d654ddb`) containing the 4-phase `rules/GEMINI.md` and `rules/AGENTS.md` files.

- **TASK-SYNC-002**: Implement `SkillsSynchronizer` for `.agents/skills/` synchronization and stale skill pruning
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003, REQ-004, REQ-007, REQ-008
  - **Blocked by**: TASK-SYNC-001
  - **Verification Seam**: `crates/aico-core/tests/scanner_test.rs`
  - **What it delivers**:
    - Scaffolds `@verifies` contract tests in `crates/aico-core/tests/scanner_test.rs` and implements `SkillsSynchronizer` in `crates/aico-core/src/skills_sync.rs` (exported via `crates/aico-core/src/lib.rs`).
    - Synchronizes all skill directories, `VERSION`, `README.md`, and `scripts/` into `<root_dir>/.agents/skills/`, prunes obsolete `aico-*` folders and stray `SKILL.md` files, and skips rewriting unchanged files.

### Phase 2: Rule Files Synchronization & Server Startup Integration

- **TASK-SYNC-003**: Implement rule files synchronization (`GEMINI.md` and `.agents/rules/`) and wire into `aico-server` startup
  - **Status**: Done
  - **Addresses**: REQ-005, REQ-006, REQ-007, REQ-008
  - **Blocked by**: TASK-SYNC-002
  - **Verification Seam**: `crates/aico-server/src/main.rs`
  - **What it delivers**:
    - Synchronizes `rules/GEMINI.md` to `<root_dir>/GEMINI.md` and `<root_dir>/.agents/rules/GEMINI.md`, and `rules/AGENTS.md` to `<root_dir>/.agents/rules/AGENTS.md`.
    - Integrates `SkillsSynchronizer::sync_workspace(&root_dir)` into `crates/aico-server/src/main.rs` startup sequence and executes synchronization on the active `d:/aico` workspace.

---

## 3. Amendment History

- **2026-10-16T20:35:00Z**: Initial creation of implementation plan for startup workspace skills and agent rules synchronization under epic `ec-core`.
