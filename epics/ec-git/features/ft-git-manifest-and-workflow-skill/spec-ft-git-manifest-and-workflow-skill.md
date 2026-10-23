---
id: spec-ft-git-manifest-and-workflow-skill
type: spec
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 5
title: "Manifest Spec Repo Elevation, Dashboard Repo Split & Multi-Repo Git Workflow"
status: "Completed"
derives_from: ["brain-ec-git", "ADR-0002", "ADR-0008"]
created_at: "2026-10-23T19:05:21.326Z"
---

# Feature Specification: Manifest Spec Repo Elevation, Dashboard Repo Split & Multi-Repo Git Workflow

## 1. Problem Statement & User Value

Currently, `.aico-project.json` embeds the specs repository inside the `repositories` array alongside codebase repositories (`path: "."`, `type: "specs"`), and the Home Dashboard (`GitRepositoriesCard.tsx`) renders all repositories together under a single "Codebase Repositories" heading. In addition, developers and agents lack built-in Git workflow operations (`commit`, `pull`, `push`, `clone`) in both the WebUI Dashboard and the AICO Skills suite.

This feature:
1. Elevates the specs repository metadata to the top level of `.aico-project.json` (`project_name`, `id`, `name`, `url`, `created_at`), reserving `repositories` for codebase and skills repositories (with `path` defaulting to `../codebase/<id>` when omitted, while preserving explicit `path` values when present).
2. Separates the **Specs Repository** and **Codebase Repositories** in the Home Dashboard (`GitRepositoriesCard.tsx`).
3. Provides full `commit`, `pull`, `push`, and `clone` support via both **WebUI repository card buttons** (`POST /api/v1/git/{pull,push,commit,clone}`) and a dedicated **`/aico-git` agent skill** (`aico-skills/aico-git/SKILL.md` + `aico-cli.mjs`), enforcing that cloned codebase repositories land in `<workspace-root>/codebase/<repo-id>` and cloned specs repositories land in `<workspace-root>/<project>-specs`.

---

## In-Scope

- Upgrading `ProjectManifest` (`crates/aico-core/src/manifest.rs`), `aico-cli.mjs`, and `d:/aico/aico-specs/.aico-project.json` to support top-level `project_name`, `id` (e.g. `aico-specs`), `name` (e.g. `AICO Specs`), and `url`, with `repositories` containing codebase/skills repositories and optional `path` resolution (defaulting to `../codebase/<id>`).
- Separating the Specs Repository (`type: "specs"`) from Codebase Repositories (`type: "codebase" | "skills"`) in `GitRepositoriesCard.tsx` on the Home Dashboard.
- Adding backend Git action endpoints (`POST /api/v1/git/pull`, `POST /api/v1/git/push`, `POST /api/v1/git/commit`, `POST /api/v1/git/clone`) in `crates/aico-server/src/git.rs` and `server.rs`.
- Enforcing canonical clone target placement: codebase/skills repositories clone into `<workspace-root>/codebase/<repo-id>`, and specs repositories clone into `<workspace-root>/<project>-specs`.
- Adding `Pull`, `Push`, `Commit`, and `Clone` action controls to `GitRepositoriesCard.tsx`.
- Creating the `/aico-git` skill (`aico-skills/aico-git/SKILL.md`) and `aico-cli.mjs` Git subcommands (`git-status`, `git-pull`, `git-push`, `git-commit`, `git-clone`).

---

## Out-of-Scope

- Interactive merge-conflict resolution editors inside the WebUI.
- Force-pushing (`git push --force`) or history rewriting (`git rebase -i`).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer configuring `.aico-project.json`, I want `project_name` and the specs repository metadata (`id`, `name`, `url`) at the top level and codebase repositories inside `repositories`, so that the project manifest cleanly reflects the decoupled SDD topology.
   - *Independent Test*: Parse the updated `.aico-project.json` in `ProjectManifest::from_json_str` and `aico-cli.mjs`; verify both `aico-specs` and codebase repos resolve with accurate paths and URLs.
2. **US-02** (Priority: P1): As a developer viewing the Home Dashboard, I want the Specs Repository and Codebase Repositories displayed in separate sections with `Pull`, `Push`, `Commit`, and `Clone` actions, so that I can manage SDD artifacts and implementation codebases independently.
   - *Independent Test*: Render `GitRepositoriesCard` with `aico-specs`, `aico`, and `aico-skills`; verify `Specs Repository` and `Codebase Repositories` are visually separated and action buttons invoke their respective API endpoints.
3. **US-03** (Priority: P1): As a developer or agent cloning a repository via the WebUI or `/aico-git` skill, I want codebase repos to be cloned into `<workspace-root>/codebase/<repo-id>` and specs repos into `<workspace-root>/<project>-specs`, so that workspace directory conventions are always preserved.
   - *Independent Test*: Invoke `resolve_clone_target_path` / `POST /api/v1/git/clone` and `aico-cli.mjs git-clone` for a `codebase` repo and a `specs` repo; verify target paths resolve to `<root>/codebase/<id>` and `<root>/<id>` respectively.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `ProjectManifest` parser and `aico-cli.mjs` shall support top-level `project_name`, `id`, `name`, and `url` fields describing the project's Specs Repository alongside the `repositories` array for codebase and skills repositories.
- **REQ-002**: Where a codebase repository entry in `repositories` omits the `path` field, the `ProjectManifest` shall default its path to `../codebase/<id>` relative to the specs directory, and where `path` is provided, the `ProjectManifest` shall resolve the declared relative path.
- **REQ-003**: The `GitRepositoriesCard` component on the Home Dashboard shall render the Specs Repository (`type: "specs"`) in a dedicated section separate from the Codebase Repositories (`type: "codebase" | "skills"`).
- **REQ-004**: The HostServer shall expose `POST /api/v1/git/pull`, `POST /api/v1/git/push`, `POST /api/v1/git/commit`, and `POST /api/v1/git/clone` to execute Git operations across registered project repositories.
- **REQ-005**: The `aico-skills` suite shall provide an `aico-git` skill (`aico-git/SKILL.md`) and `aico-cli.mjs` Git subcommands supporting `commit`, `pull`, `push`, and `clone` across project repositories.

### Event-Driven Requirements
- **REQ-006**: When a `clone` operation is requested for a repository with `type: "codebase"` or `type: "skills"`, the system shall clone the repository into `<workspace-root>/codebase/<repo-id>`.
- **REQ-007**: When a `clone` operation is requested for a repository with `type: "specs"`, the system shall clone the repository into `<workspace-root>/<project-name>-specs` at the workspace root.
- **REQ-008**: When a user clicks `Pull`, `Push`, `Commit`, or `Clone` on a repository card in `GitRepositoriesCard`, the frontend shall invoke the corresponding `/api/v1/git/*` endpoint and refresh the repository telemetry state.

### Unwanted Behavior & Error Handling
- **REQ-009**: If a Git operation (`pull`, `push`, `commit`, or `clone`) fails due to a non-zero Git exit code or missing commit message, then the system shall return a structured error message without corrupting the repository state.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Manifest Top-Level Spec & Default Path Seam (`crates/aico-core/tests/manifest_test.rs`)**:
   - Signature: `ProjectManifest::from_json_str` & `ProjectManifest::get_effective_repositories`
   - Verifies: Top-level `project_name` / `id` / `name` / `url` parsing (`REQ-001`) and optional `path` fallback to `../codebase/<id>` (`REQ-002`).
2. **Backend Git Operations & Clone Placement Seam (`crates/aico-server/tests/git_scanner_test.rs`)**:
   - Signature: `GitStatusScanner::resolve_clone_target` & `POST /api/v1/git/{pull,push,commit,clone}`
   - Verifies: Codebase clone placement in `<root>/codebase/<id>` (`REQ-006`), specs clone placement in `<root>/<project>-specs` (`REQ-007`), and `commit`/`pull`/`push` execution (`REQ-004`, `REQ-009`).
3. **Dashboard Split & Git Actions UI Seam (`frontend/tests/GitRepositoriesCard.test.tsx`)**:
   - Signature: `<GitRepositoriesCard statuses={...} />`
   - Verifies: Visual separation of `Specs Repository` vs `Codebase Repositories` (`REQ-003`) and interactive `Pull`, `Push`, `Commit`, and `Clone` triggers (`REQ-008`).
4. **AICO Git Skill & CLI Seam (`codebase/aico-skills/tests/aico-cli.test.mjs`)**:
   - Signature: `aico-cli.mjs git-clone / git-commit / git-pull / git-push` and `aico-git/SKILL.md`
   - Verifies: Multi-repo Git CLI operations and manifest compatibility (`REQ-001`, `REQ-005`, `REQ-006`, `REQ-007`).

---

## 5. Open Clarifications

None. Ratified in [ADR-0002](../../../../adr/0002-decoupled-sdd-and-codebase-repositories.md) and [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
