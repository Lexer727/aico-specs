---
id: plan-ft-git-manifest-and-workflow-skill
type: plan
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 5
title: "Manifest Spec Repo Elevation, Dashboard Repo Split & Multi-Repo Git Workflow Implementation Plan"
status: "Completed"
repositories: ["aico", "aico-skills", "aico-specs"]
derives_from: ["spec-ft-git-manifest-and-workflow-skill"]
created_at: "2026-10-23T19:05:21.326Z"
---

# Implementation Plan: Manifest Spec Repo Elevation, Dashboard Repo Split & Multi-Repo Git Workflow

## 1. Technical Architecture & Decisions

- **Target repositories**: `aico`, `aico-skills`, `aico-specs`.
- **Manifest Schema Upgrade (`.aico-project.json`)**:
  - Update `ProjectManifest` in `crates/aico-core/src/manifest.rs` to support `project_name: Option<String>` (with fallback to `name` for backward compatibility), `id: Option<String>` (defaulting to `{project_name}-specs`), `name: String` (serving as display name `"AICO Specs"` when `project_name` is present), and `url: Option<String>`.
  - Make `path` on `ProjectRepository` default to `../codebase/{id}` when omitted or empty, while preserving explicit `path` values (`../codebase/aico`).
  - Update `d:/aico/aico-specs/.aico-project.json` to the elevated format (`project_name: "aico"`, `id: "aico-specs"`, `name: "AICO Specs"`, `url: "https://github.com/Lexer727/aico-specs"`, and `repositories` with `aico` and `aico-skills`).
  - Update `getKnownProjectRepositories` in `aico-cli.mjs` so both top-level `manifest.id` (`aico-specs`) and all entries in `manifest.repositories` are recognized as valid repository IDs.
- **Backend Git Operations & Clone Placement**:
  - In `crates/aico-server/src/git.rs`, add `GitStatusScanner::pull_repo`, `push_repo`, `commit_repo`, and `clone_repo`, plus `resolve_clone_target_path(workspace_root, repo_id, repo_type)` which routes `specs` repos to `<workspace_root>/<repo_id>` and `codebase`/`skills` repos to `<workspace_root>/codebase/<repo_id>`.
  - Register `POST /api/v1/git/pull`, `POST /api/v1/git/push`, `POST /api/v1/git/commit`, and `POST /api/v1/git/clone` in `crates/aico-server/src/server.rs`.
- **Dashboard UI Split & Action Controls**:
  - In `frontend/src/features/dashboard/GitRepositoriesCard.tsx`, split `repoStatuses` into **Specs Repository** (`repo.type === 'specs'`) and **Codebase Repositories** (`repo.type !== 'specs'`), and add `Pull`, `Push`, `Commit` (with inline commit message input), and `Clone` (when `!repo.is_available` or via Clone modal/inline form) buttons.
- **AICO Git Skill (`/aico-git`)**:
  - Create `codebase/aico-skills/aico-git/SKILL.md` (and sync to `.agents/skills/aico-git/SKILL.md`) and add `git-pull`, `git-push`, `git-commit`, and `git-clone` subcommands to `codebase/aico-skills/scripts/aico-cli.mjs`.

---

## 2. Task Graph (DAG)

### Phase 1: Manifest Schema Upgrade & `.aico-project.json` Migration

- **TASK-GIT-005-1**: Elevate Spec Repo in `ProjectManifest`, `.aico-project.json` & `aico-cli.mjs`
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updates `ProjectManifest` in `crates/aico-core/src/manifest.rs` to parse top-level `project_name`, `id`, `name`, `url` for the specs repository and optional `path` on `repositories` entries.
    - Updates `getKnownProjectRepositories` in `codebase/aico-skills/scripts/aico-cli.mjs` to include top-level `manifest.id`.
    - Migrates `d:/aico/aico-specs/.aico-project.json` to the elevated structure.
  - **Verification Seam**: `crates/aico-core/tests/manifest_test.rs`

### Phase 2: Backend Git Operations (`commit`, `pull`, `push`, `clone`) & `/aico-git` Skill

- **TASK-GIT-005-2**: Implement Backend `/api/v1/git/{pull,push,commit,clone}` Endpoints & Clone Path Routing
  - **Status**: Done
  - **Addresses**: `REQ-004`, `REQ-006`, `REQ-007`, `REQ-009`
  - **Blocked by**: `TASK-GIT-005-1`
  - **What it delivers**:
    - Implements `pull_repo`, `push_repo`, `commit_repo`, `clone_repo`, and `resolve_clone_target_path` in `crates/aico-server/src/git.rs`.
    - Ensures `clone` places codebase/skills repos in `<workspace-root>/codebase/<id>` and specs repos in `<workspace-root>/<project>-specs`.
    - Wires `POST /api/v1/git/{pull,push,commit,clone}` in `crates/aico-server/src/server.rs`.
  - **Verification Seam**: `crates/aico-server/tests/git_scanner_test.rs`

- **TASK-GIT-005-3**: Create `/aico-git` Skill & `aico-cli.mjs` Multi-Repo Git Subcommands
  - **Status**: Done
  - **Addresses**: `REQ-005`, `REQ-006`, `REQ-007`
  - **Blocked by**: `TASK-GIT-005-1`
  - **What it delivers**:
    - Adds `git-pull`, `git-push`, `git-commit`, and `git-clone` subcommands to `codebase/aico-skills/scripts/aico-cli.mjs` with canonical `<root>/codebase/<id>` vs `<root>/<id>-specs` target resolution.
    - Authors `codebase/aico-skills/aico-git/SKILL.md` and syncs it to `.agents/skills/aico-git/SKILL.md`.
  - **Verification Seam**: `codebase/aico-skills/tests/aico-cli.test.mjs`

### Phase 3: Home Dashboard Specs vs Codebase Separation & Interactive Git Controls

- **TASK-GIT-005-4**: Separate Specs & Codebase Repositories and Add Git Controls in `GitRepositoriesCard.tsx`
  - **Status**: Done
  - **Addresses**: `REQ-003`, `REQ-008`
  - **Blocked by**: `TASK-GIT-005-2`
  - **What it delivers**:
    - Splits `GitRepositoriesCard.tsx` into distinct **Specs Repository** and **Codebase Repositories** sections.
    - Adds interactive `Pull`, `Push`, `Commit`, and `Clone` controls on repository cards wired to `/api/v1/git/*`.
  - **Verification Seam**: `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`
