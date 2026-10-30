---
id: spec-ft-core-nested-specs-folder-and-relative-paths
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 22
title: "Nested Specs Directory Structure and Relative Path Normalization"
status: "Completed"
derives_from: ["spec-ft-core-workspace-path-normalization", "spec-ft-core-workspace-graph-and-project-creator", "ADR-0004", "ADR-0008"]
created_at: "2026-10-30T18:20:00.000Z"
---

# Feature Specification: Nested Specs Directory Structure and Relative Path Normalization

## 1. Problem Statement & User Value

In the initial AICO platform topology, project specification repositories were located directly under the top-level workspace root (`d:/aico/<project>-specs/`, e.g. `d:/aico/aico-specs/`). While functional, having specifications scattered at the workspace root mixed specs repositories with platform configuration folders (`.agents/`, `.aico/`, `.cargo/`) and codebase directories (`codebase/`).

To establish a clean, symmetrical, and scalable workspace organization:
1. **Clean Root Topology**: All specifications repositories must reside in a dedicated `specs/` directory (`<workspace-root>/specs/<project>-specs/`), perfectly mirroring the `codebase/` directory (`<workspace-root>/codebase/<project>/`).
2. **Relative Path Consistency**: Moving specifications one level deeper changes the relative filesystem trajectory between a specification repository and its codebase repositories from `../codebase/<id>` to `../../codebase/<id>`.
3. **Automated Scaffolding & UI Visibility**: All project creation endpoints, wizard tools, and frontend modals must scaffold new projects under `specs/` and clearly present the normalized relative paths (`specs/<project>-specs` and `../../codebase/<id>`).

---

## In-Scope

- Updating `RootScanner::scan()` in `crates/aico-core/src/scanner.rs` to scan `<workspace-root>/specs/` as the primary specifications directory, with graceful fallback to root-level folders for backwards compatibility.
- Updating `get_workspace_root()` in `crates/aico-server/src/server.rs` and `StorageFederationManager` in `crates/aico-storage/src/manager.rs` to correctly resolve the workspace root when `specs_path` is nested under `specs/`.
- Updating `ProjectManifest::resolve_path()` and path serialization in `crates/aico-core/src/manifest.rs` to support and produce `../../codebase/<id>` paths with resilient fallback.
- Updating `CodebaseWizard::scaffold_project_specs()` and `handle_create_project` to target `<workspace-root>/specs/<slug>-specs/`.
- Updating `resolve_clone_target_dir()` in `crates/aico-server/src/git.rs` to clone specs repositories into `<workspace-root>/specs/<repo_id>`.
- Updating `ProjectConfigModal.tsx` and `SettingsModal.tsx` in the WebUI to display and submit normalized `specs/` and `../../codebase/` paths.
- Updating `aico-cli.mjs` verification engine to discover manifests in `<workspace-root>/specs/<repo_id>/`.
- Relocating the active `aico-specs` repository to `specs/aico-specs` and updating its `.aico-project.json`.

---

## Out-of-Scope

- Relocating the `codebase/` directory (remains at `<workspace-root>/codebase/`).
- Arbitrary multi-nested directory structures beyond `<workspace-root>/specs/<repo_id>`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer adding a new project in AICO, I want the specifications repository to be created under `specs/<project-name>-specs`, so that my workspace root remains organized and clean.
   - *Independent Test*: Create a new project via `POST /api/v1/projects`; verify the specifications directory is scaffolded under `<workspace-root>/specs/<project-name>-specs`.
2. **US-02** (Priority: P1): As a developer configuring linked codebase repositories, I want relative paths to accurately point to `../../codebase/<id>`, so that Git and toolchain operations resolve the correct code directories from inside the specs repository.
   - *Independent Test*: Inspect `.aico-project.json` in a created project; verify repository paths use `../../codebase/<id>` and resolve to valid absolute paths.
3. **US-03** (Priority: P1): As a developer inspecting workspace topology in Settings or the Add Project modal, I want to see `specs/<project>-specs` and `../../codebase/<id>` clearly displayed, so that I understand where files reside on disk.
   - *Independent Test*: Open Settings modal and Add Project modal; verify UI labels and previews reflect the `specs/` container.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The platform workspace layout shall house all project specification repositories inside a dedicated `specs/` directory under the workspace root (`<workspace-root>/specs/<project_slug>-specs/`).
- **REQ-002**: `RootScanner::scan()` shall discover project specifications repositories located within `<workspace-root>/specs/`, while preserving fallback discovery for legacy root-level specification repositories.
- **REQ-003**: `get_workspace_root()` in `aico-server` and `StorageFederationManager` in `aico-storage` shall correctly resolve the workspace root container (`d:/aico`) when given a nested `specs_path` (e.g. `d:/aico/specs/aico-specs`).
- **REQ-004**: Project manifest path resolution (`ProjectManifest::resolve_path`) and generation shall standardize relative paths between nested specification directories and codebase repositories to `../../codebase/<id>`, supporting resilient fallback for legacy single-step paths.

### Event-Driven Requirements
- **REQ-005**: When `handle_create_project` or `CodebaseWizard::scaffold_project_specs` is called to create or clone a project, the system shall scaffold the specifications repository under `<workspace-root>/specs/<slug>-specs/` and generate relative codebase repository paths as `../../codebase/<id>`.
- **REQ-006**: In `ProjectConfigModal.tsx`, the UI shall display the folder preview as `specs/<project-name>-specs`, format linked repository path previews and submissions as `../../codebase/<id>`, and update codebase folder browser labels and hints accordingly.
- **REQ-007**: In `SettingsModal.tsx`, the Workspace Architecture & Topology section shall document the specifications directory as `<workspace-root>/specs/*-specs/` (e.g. `d:/aico/specs/aico-specs`).
- **REQ-008**: The unified verification engine (`aico-cli.mjs`) and CLI helper scripts shall resolve project manifests located in `<workspace-root>/specs/<project-specs>/` and support verification invocations targeting `specs/<project-specs>`.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Core Scanner & Manifest Seam (`crates/aico-core/tests/scanner_test.rs` & `manifest_test.rs`)**:
   - Signature: `RootScanner::scan()`, `ProjectManifest::resolve_path()`, `resolve_codebase_path()`.
   - Verifies: `REQ-001`, `REQ-002`, `REQ-004`.
2. **Project Creation & Git Seam (`crates/aico-server/tests/project_api_test.rs` & `server_test.rs`)**:
   - Signature: `POST /api/v1/projects`, `get_workspace_root()`, `resolve_clone_target_dir()`.
   - Verifies: `REQ-003`, `REQ-005`.
3. **Frontend UI Seam (`frontend/tests/ProjectSwitcher.test.tsx` & `SettingsModal.test.tsx`)**:
   - Signature: `<ProjectConfigModal />` path previews and submissions, `<SettingsModal />` topology section.
   - Verifies: `REQ-006`, `REQ-007`.
4. **Artifact Verification Seam (`verify-artifacts.mjs`)**:
   - Signature: `node .agents/skills/aico-verify/scripts/verify-artifacts.mjs specs/aico-specs`.
   - Verifies: `REQ-008`.

---

## 5. Open Clarifications

None. Approved in architectural triage.
