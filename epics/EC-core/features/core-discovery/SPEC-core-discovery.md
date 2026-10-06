---
id: spec-core-discovery
type: spec
aico_version: "0.1.2"
epic: "ec-core"
order: 4
title: "Project Discovery, Manifest Management & Workspace Pairing"
status: "Completed"
derives_from: ["ADR-0002", "ADR-0008"]
created_at: "2026-10-05T19:15:00Z"
---

# Feature Specification: Project Discovery & Workspace Manifests


---

## 1. Problem Statement & User Value

Managing multiple software projects across separate Git repositories requires error-free pairing between each project's specifications and its implementation code. Asking developers to configure individual paths manually creates friction, while hardcoded directory assumptions prevent onboarding external codebases.

This specification defines the Project Discovery and Manifest Engine: an automated scanner that initializes from a single AICO Root (`d:/aico/`), detects projects via lightweight `.aico-project.json` manifests, resolves relative or absolute codebase bindings, and populates the UI Project Switcher with zero configuration friction.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer, I want to configure a single root path (e.g. `d:/aico/`) on installation, so that AICO discovers all my project specs and codebases automatically.
   - *Independent Test*: Set the root directory and verify all sibling `*-specs` folders containing `.aico-project.json` are registered on boot.
2. **US-02** (Priority: P1): As a developer, I want to link an existing external codebase anywhere on my drive, so that I can use AICO with repositories that already exist outside `d:/aico/codebase/`.
   - *Independent Test*: Create a specs folder with `"codebase_path": "d:/dev/external-app"` and verify AICO binds to the external directory.
3. **US-03** (Priority: P1): As a developer, I want to create a new project with one command or click, so that AICO scaffolds the `<project>-specs` Git repository and links it to the codebase automatically.
   - *Independent Test*: Run `aico project create my-new-app` and verify `<aico-root>/my-new-app-specs/` and `<aico-root>/codebase/my-new-app/` are initialized with Git and manifest.
4. **US-04** (Priority: P2): As a developer, I want switching projects in the UI to instantly re-scope the active tree, specs, and search index, so that I can transition between tasks cleanly.
   - *Independent Test*: Dispatch an active project switch event and verify the backend switches active Kùzu and LanceDB handles within 50ms.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The DiscoveryScanner shall enumerate all directories in the configured AICO root matching the pattern `*-specs` upon application startup.
- **REQ-002**: The ManifestManager shall parse and validate the schema of `.aico-project.json` inside each discovered specs repository.
- **REQ-003**: The ProjectRegistry shall maintain an in-memory catalog of all registered projects containing project name, specs path, resolved codebase path, and Git status.

### Event-Driven Requirements
- **REQ-004**: When the ManifestManager reads a relative `codebase_path` (e.g. `"../codebase/project-a"`), the PathResolver shall resolve the path relative to the specs repository directory.
- **REQ-005**: When the ManifestManager reads an absolute `codebase_path` (e.g. `"d:/dev/aico"`), the PathResolver shall verify that the target directory exists on disk.
- **REQ-006**: When a client issues an active project switch request via WebSocket, the ProjectRegistry shall activate the selected project and trigger the TreeSitterWatcher on its codebase.
- **REQ-007**: When a user creates a new project via CLI or UI wizard, the ProjectScaffolder shall initialize `<project>-specs/.git`, generate `.aico-project.json`, and create the baseline `GLOSSARY.md`.

### State-Driven Requirements
- **REQ-008**: While AICO is running, the DiscoveryScanner shall monitor the root directory for newly created or deleted `*-specs` folders and update the ProjectRegistry dynamically.
- **REQ-009**: While a project is active, the ProjectRegistry shall enforce that all SDD artifact generation operations target that project's specs repository.

### Unwanted Behavior & Error Handling
- **REQ-010**: If a discovered specs directory contains an invalid or malformed `.aico-project.json` file, then the ManifestManager shall log a warning and mark the project as `Status::MalformedManifest` without crashing.
- **REQ-011**: If the resolved `codebase_path` points to a non-existent directory, then the ProjectRegistry shall flag the project as `Status::MissingCodebase` and disable code-editing tool operations.

### Optional Feature Requirements
- **REQ-012**: Where the `--root <path>` CLI flag is specified, the DiscoveryScanner shall override the default AICO root path with the specified directory.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Discovery Scanner Seam (`aico_core::project::DiscoveryScanner`)**:
   - Signature: `pub fn scan_root(&self, root_dir: &Path) -> Result<Vec<DiscoveredProject>, DiscoveryError>`
   - Tests file discovery across fixture directory trees without network services.
2. **Manifest Manager Seam (`aico_core::project::ManifestManager`)**:
   - Signature: `pub fn load_manifest(&self, specs_dir: &Path) -> Result<ProjectManifest, ManifestError>`
   - Verifies JSON parsing, schema validation, and relative/absolute path resolution.
3. **Project Registry Seam (`aico_core::project::ProjectRegistry`)**:
   - Signature: `pub fn switch_active_project(&mut self, name: &str) -> Result<ActiveProjectContext, RegistryError>`
   - Verifies project switching, handle swaps, and event dispatching.

---

## 5. Out of Scope

- Hosting remote multi-user project repositories on cloud servers.
- Automatic Git remote pushing or branch creation across third-party Git hosts (GitHub/GitLab APIs).
- Automatic code transpilation during project switching.

---

## 6. Open Clarifications

None. Ratified in [ADR-0002](../../../../adr/0002-decoupled-sdd-and-codebase-repositories.md) and [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
