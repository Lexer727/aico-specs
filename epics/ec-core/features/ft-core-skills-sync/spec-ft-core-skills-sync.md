---
id: spec-ft-core-skills-sync
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 12
title: "Startup Workspace Skills & Agent Rules Synchronization"
status: "Completed"
derives_from: ["brain-ec-core", "ADR-0006", "ADR-0008"]
created_at: "2026-10-16T20:35:00Z"
---

# Feature Specification: Startup Workspace Skills & Agent Rules Synchronization

## 1. Problem Statement & User Value

The AICO methodology skills (`aico-*`) and workspace guardrail rules (`GEMINI.md`, `AGENTS.md`) are maintained in the `aico-skills` repository and integrated as a Git submodule at `codebase/aico/skills`. However, the Antigravity CLI and workspace agents discover project-scoped skills and rules from the AICO workspace root (`<root_dir>/.agents/skills/`, `<root_dir>/.agents/rules/`, and `<root_dir>/GEMINI.md`). Furthermore, Antigravity CLI assigns workspace-level skills in `.agents/skills/` higher loading precedence than global plugins (`~/.gemini/config/plugins/`).

Currently, the AICO host runtime does not synchronize `<root_dir>/.agents/` or `<root_dir>/GEMINI.md` from `<root_dir>/codebase/aico/skills` on startup. As a result, `.agents/skills/` retains outdated skill definitions and stray files that shadow newer skills, while `GEMINI.md` and `.agents/rules/` require error-prone manual copying whenever rules evolve.

This specification defines automated, idempotent startup synchronization of skills and agent rule files from the `codebase/aico/skills` submodule into the AICO workspace root so that the Antigravity CLI always discovers up-to-date skills and rules immediately upon launching AICO.

---

## In-Scope

- Automatic resolution of the skills submodule directory at `<root_dir>/codebase/aico/skills` on host runtime startup.
- Recursive synchronization of all skill directories (directories containing `SKILL.md`) from `<root_dir>/codebase/aico/skills` into `<root_dir>/.agents/skills/`.
- Synchronization of supporting root assets (`VERSION`, `README.md`, and `scripts/`) into `<root_dir>/.agents/skills/` so relative script and version resolution works from `.agents/skills/`.
- Pruning of stale `aico-*` skill directories in `<root_dir>/.agents/skills/` that no longer exist in the submodule, as well as removal of any stray root `<root_dir>/.agents/skills/SKILL.md` file.
- Automatic synchronization of `rules/GEMINI.md` (with fallback to root `GEMINI.md`) to both `<root_dir>/GEMINI.md` and `<root_dir>/.agents/rules/GEMINI.md`.
- Automatic synchronization of `rules/AGENTS.md` to `<root_dir>/.agents/rules/AGENTS.md`.
- Content-diff check before writing destination files to avoid unnecessary disk writes and file-watcher churn.
- Non-fatal fallback warning when `<root_dir>/codebase/aico/skills` is absent or uninitialized.

---

## Out-of-Scope

- Executing network-bound `git submodule update --remote` commands automatically on startup without user action.
- Modifying global user home directory configurations outside `<root_dir>`.
- Custom UI modals for editing `SKILL.md` files in the browser.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer starting the AICO server, I want `<root_dir>/.agents/skills/` to be automatically updated from `<root_dir>/codebase/aico/skills`, so that the Antigravity CLI discovers all current skills without manual copying or stale skill shadowing.
   - *Independent Test*: Run startup synchronization against a workspace containing outdated `.agents/skills/` entries and verify all current `aico-*` skills and supporting scripts are present and identical to `codebase/aico/skills`.
2. **US-02** (Priority: P1): As a developer starting the AICO server, I want `<root_dir>/GEMINI.md` and `<root_dir>/.agents/rules/` (`GEMINI.md` and `AGENTS.md`) to be automatically updated from the submodule's rule files, so that workspace guardrails are never out of sync.
   - *Independent Test*: Run startup synchronization and verify `<root_dir>/GEMINI.md`, `<root_dir>/.agents/rules/GEMINI.md`, and `<root_dir>/.agents/rules/AGENTS.md` match the canonical files in `codebase/aico/skills/rules/`.
3. **US-03** (Priority: P2): As a developer removing or renaming a skill in the `aico-skills` submodule, I want obsolete `aico-*` directories and stray root `SKILL.md` files in `.agents/skills/` to be pruned on startup, so that deprecated skills do not linger in the workspace.
   - *Independent Test*: Place a stale `aico-legacy` folder and stray `SKILL.md` in `.agents/skills/`, run synchronization, and verify both are removed while valid skills remain.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `SkillsSynchronizer` shall resolve the canonical skills source directory at `<root_dir>/codebase/aico/skills` upon application startup.

### Event-Driven Requirements
- **REQ-002**: When the host runtime initializes on startup, the `SkillsSynchronizer` shall synchronize all skill directories (subdirectories containing `SKILL.md`) and their nested files from the skills source directory into `<root_dir>/.agents/skills/`.
- **REQ-003**: When synchronizing `<root_dir>/.agents/skills/`, the `SkillsSynchronizer` shall synchronize supporting root metadata and scripts (`VERSION`, `README.md`, and `scripts/`) into `<root_dir>/.agents/skills/`.
- **REQ-004**: When synchronizing `<root_dir>/.agents/skills/`, the `SkillsSynchronizer` shall remove any stale `aico-*` subdirectory in `<root_dir>/.agents/skills/` that does not exist as a valid skill directory in the source submodule, and shall remove any stray `<root_dir>/.agents/skills/SKILL.md` file.
- **REQ-005**: When the host runtime initializes on startup, the `SkillsSynchronizer` shall synchronize `rules/GEMINI.md` (or fallback `GEMINI.md`) from the skills source directory to both `<root_dir>/GEMINI.md` and `<root_dir>/.agents/rules/GEMINI.md`.
- **REQ-006**: When `rules/AGENTS.md` exists in the skills source directory, the `SkillsSynchronizer` shall synchronize it to `<root_dir>/.agents/rules/AGENTS.md`.

### Unwanted Behavior & Error Handling
- **REQ-007**: If the skills source directory `<root_dir>/codebase/aico/skills` does not exist or contains no skill directories, then the `SkillsSynchronizer` shall log a non-fatal warning and return gracefully without aborting server startup or deleting existing `.agents` files.

### State-Driven Requirements
- **REQ-008**: While synchronizing skill and rule files, the `SkillsSynchronizer` shall compare source and destination byte contents and write to disk only when the destination file is missing or differs from the source file.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Workspace Skills Synchronizer Seam (`aico_core::skills_sync::SkillsSynchronizer`)**:
   - Signatures:
     - `pub fn sync_workspace<P: AsRef<Path>>(root_dir: P) -> Result<SyncReport, SkillsSyncError>`
     - `pub fn sync_from_source<P: AsRef<Path>, Q: AsRef<Path>>(source_skills_dir: P, root_dir: Q) -> Result<SyncReport, SkillsSyncError>`
   - Contract Test Location: `crates/aico-core/tests/scanner_test.rs`
   - Verifies recursive skill copying, `VERSION`/`README.md`/`scripts/` copying, stale `aico-*` and stray `SKILL.md` pruning, `GEMINI.md` and `.agents/rules/` synchronization, idempotent no-op writes on identical content, and graceful fallback when the submodule directory is missing.
2. **Host Server Startup Integration (`crates/aico-server/src/main.rs`)**:
   - Invokes `SkillsSynchronizer::sync_workspace(&root_dir)` during startup initialization and logs synchronization counts.

---

## 5. Open Clarifications

None. Derived from [Brainstorm: AICO Core Architecture](../../brain-ec-core.md), [ADR-0006](../../../../adr/0006-packaging-aico-specs-and-version-tagging.md), and [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
