---
id: spec-ft-core-add-project-and-github-scaffolding
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 20
title: "Add Project Flow, GitHub Remote Scaffolding & Codebase Browser"
status: "Completed"
derives_from: ["brain-ec-core", "ADR-0004", "ADR-0008"]
created_at: "2026-10-30T17:10:00.000Z"
---

# Feature Specification: Add Project Flow, GitHub Remote Scaffolding & Codebase Browser

## 1. Problem Statement & User Value

The existing project creation modal and onboarding workflow has several usability and automation limitations:

1. **Unclear Terminology & Misleading Default Codebase Text**: The current modal is labeled "Create New Project", which does not fit when onboarding an existing project or importing a remote specs repository. When no linked repositories are added, the UI displays a misleading notice (`"Default codebase repository (<id>) will be created automatically, or click 'Add Repository' to customize."`), but no default codebase repository is actually created on disk—it only appears virtually in the dashboard view.
2. **Redundant Repository Type Selection**: Linked repositories include a `Type` dropdown with `codebase` and `skills` choices, but all project repositories are codebases.
3. **Manual Codebase Path Entry**: Linking local codebase repositories requires manually typing the repository ID and path, with no way to discover or select existing projects located inside `../codebase/`.
4. **Manual GitHub Setup & Scaffolding**: Creating a new specs repository on GitHub currently requires the developer to open a browser, create a repository on GitHub, copy its clone URL, paste it into AICO, initialize local Git, and push. AICO lacks the ability to automatically check if a repository exists on GitHub, verify if it is already a specs repository, pre-fill its configuration, or create and push the repository automatically.
5. **No Agent-Driven Project Scaffolding Tool**: Autonomous agents running in background sessions or terminal workflows cannot programmatically scaffold or import AICO projects via MCP.

Reworking this workflow into an automated **Add Project** flow with GitHub credential management, remote repository status verification, dynamic URL pre-filling, local-only mode, codebase folder discovery, and an MCP agent tool provides a seamless onboarding experience for both human developers and AI coding agents.

---

## In-Scope

- Rebranding all "Create New Project" UI elements to **"Add Project"** (trigger button `+ Add Project`, modal title `Add Project (.aico-project.json)`, action button `Add Project`).
- Removing the misleading `"Default codebase repository (<id>)..."` placeholder text from `ProjectConfigModal.tsx`.
- Removing the `Type` select dropdown (`codebase` / `skills`) from linked repository configuration—all linked repositories default to and persist as `type: "codebase"`.
- Adding local codebase folder discovery: endpoint `GET /api/v1/workspace/codebase-folders` scanning `<workspace_root>/codebase/` and a "Browse ../codebase" picker button in `ProjectConfigModal` (in both Add and Edit modes).
- Multi-tier GitHub credential management (Hybrid Resolver: `GITHUB_TOKEN` / `GH_TOKEN` env, `gh auth token` CLI fallback, and persistent storage `~/.aico/config.json`) exposed via `GET /api/v1/github/status`, `POST /api/v1/github/token`, and `DELETE /api/v1/github/token`.
- Adding a GitHub Connection card to `SettingsModal.tsx` displaying connection status, account username, and token management.
- Dynamic GitHub Specs Remote URL pre-filling (`https://github.com/<username>/<project_name>-specs.git`) upon entering `project_name`, with permissive validation warning if `-specs` postfix is missing.
- "Local repository only" checkbox to disable remote URL pre-filling and skip GitHub remote creation/pushing.
- Public vs. Private repository toggle for GitHub repository creation, defaulting to Private.
- Remote specs repository inspector button ("Check Repo") calling `POST /api/v1/github/check-repo` handling 4 distinct cases:
  1. *Exists & is AICO specs repo*: Reads `.aico-project.json`, pre-fills Display Name & linked repositories, and clones/pulls specs and linked repos on Add Project.
  2. *Exists & is empty*: Confirms repository is ready for initial scaffolding push.
  3. *Exists & non-empty / non-specs*: Displays a warning notice with a confirmation checkbox before initializing specs.
  4. *Does not exist*: Informs user that the repository will be created automatically on GitHub upon clicking Add Project.
- Extending `POST /api/v1/projects` to orchestrate remote GitHub repository creation via GitHub API, scaffolding push, remote cloning, and local-only mode.
- Adding a `create_project` tool to `crates/aico-mcp` enabling autonomous agents to scaffold and onboard projects.

---

## Out-of-Scope

- OAuth web server callback authentication (uses Personal Access Tokens and local CLI/environment credentials).
- Deleting or destroying remote GitHub repositories from within the AICO WebUI.
- Automatic Git hosting providers other than GitHub (GitLab, Bitbucket, self-hosted Gitea).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer adding a new project to AICO, I want the project creator to be named "Add Project" and automatically pre-fill a GitHub specs URL ending in `-specs`, so that I don't have to manually create the repo on GitHub first.
   - *Independent Test*: Open Add Project modal, enter `alpha-service`; verify Display Name is `Alpha Service Specs` and Specs URL is `https://github.com/<user>/alpha-service-specs.git`.
2. **US-02** (Priority: P1): As a developer with existing codebase repositories under `../codebase/`, I want to click "Browse" in the linked repositories section and select a folder, so that I don't have to type repository IDs and paths manually.
   - *Independent Test*: Query `GET /api/v1/workspace/codebase-folders` and verify folders under `codebase/` are returned; render modal, click "Browse", select a folder, and verify repo row is populated.
3. **US-03** (Priority: P1): As a developer connecting an existing remote specs repository from GitHub, I want the "Check Repo" button to detect that it's a specs repo and load its `.aico-project.json`, so that all linked repositories and project settings are imported automatically.
   - *Independent Test*: Dispatch `POST /api/v1/github/check-repo` with an existing specs repo URL; verify response contains `is_specs: true` and parsed manifest data.
4. **US-04** (Priority: P2): As a developer working purely offline or locally, I want to check "Local repository only", so that AICO skips all GitHub interactions and keeps the project entirely on my local disk.
   - *Independent Test*: Check "Local repository only"; verify GitHub URL input is hidden/disabled and project creates cleanly with local `git init` only.
5. **US-05** (Priority: P2): As an autonomous AI agent, I want an MCP tool `create_project`, so that I can scaffold a new project workspace or onboard existing codebases directly from terminal or chat workflows.
   - *Independent Test*: Invoke MCP tool `create_project` via JSON-RPC; verify new project directory is scaffolded and registered in `WorkspaceRegistry`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The WebUI Shell and `ProjectSwitcher` shall rebrand the project creation workflow from "Create New Project" to **"Add Project"**, updating trigger buttons (`+ Add Project`), modal titles (`Add Project (.aico-project.json)`), and submit actions.
- **REQ-002**: The `ProjectConfigModal` component shall omit the placeholder notice (`"Default codebase repository (<id>) will be created automatically..."`) and shall remove the `Type` select dropdown (`codebase` / `skills`) from linked repository rows; all linked repositories shall strictly default to and be stored as `type: "codebase"`.
- **REQ-003**: The HostServer shall expose `GET /api/v1/workspace/codebase-folders` scanning `<workspace_root>/codebase/` and returning folder items (`id`, `name`, `path`, `is_git`), and `ProjectConfigModal` (in both Add and Edit modes) shall provide a "Browse ../codebase" button/picker to populate repository fields with a single click.
- **REQ-004**: The system shall provide a multi-tier Hybrid Credential Resolver checking `GITHUB_TOKEN` / `GH_TOKEN` environment variables, `gh auth token` CLI output, and local persistent storage (`~/.aico/config.json`), exposed via `GET /api/v1/github/status`, `POST /api/v1/github/token`, and `DELETE /api/v1/github/token`.
- **REQ-005**: While configuring an Add Project modal, entering `project_name` shall automatically pre-fill the Specs Repository Remote URL with `https://github.com/<username>/<project_name>-specs.git`. If the user manually edits the URL and omits the `-specs` postfix, the UI shall render a permissive warning indicator while permitting submission upon user confirmation.
- **REQ-006**: While the "Local repository only" checkbox is checked, the Specs Remote URL input and GitHub verification controls shall be hidden or disabled, and project creation shall skip all GitHub API operations.
- **REQ-007**: When creating a remote repository on GitHub, the `ProjectConfigModal` shall provide a toggle for Public vs. Private repository creation, defaulting to Private.

### Event-Driven Requirements
- **REQ-008**: When a user clicks the "Check Repo" button next to the remote specs URL in `ProjectConfigModal`, the frontend shall send `POST /api/v1/github/check-repo`:
  - **Case A (Exists & is AICO specs repo)**: The backend shall return `{ exists: true, is_specs: true, manifest: ... }`, and the modal shall automatically populate Display Name and linked repositories from remote `.aico-project.json`. On Add Project, the system shall clone the specs repository and pull/clone any declared linked repositories into `../codebase/<id>`.
  - **Case B (Exists & is empty)**: The backend shall return `{ exists: true, is_specs: false, is_empty: true }`, and the modal shall display confirmation that the repository is ready for initial scaffolding push.
  - **Case C (Exists & non-empty / non-specs)**: The backend shall return `{ exists: true, is_specs: false, is_empty: false }`, and the modal shall render a warning notice requiring a confirmation checkbox ("Initialize AICO specs in this existing repository") before allowing creation.
  - **Case D (Does not exist)**: The backend shall return `{ exists: false }`, and the modal shall inform the user that the repository will be created automatically on GitHub upon clicking Add Project.
- **REQ-009**: When `POST /api/v1/projects` receives an Add Project request, the HostServer shall orchestrate repository creation and synchronization based on target state:
  - If the target repository does not exist on GitHub and is not local-only, the HostServer shall call the GitHub REST API (`POST /user/repos`) to create the repository (with specified privacy), scaffold local directories (`adr/`, `epics/`, `features/`, `.aico-project.json`, `glossary.md`), initialize Git (`git init`), add remote `origin`, create an initial commit, push to GitHub, and register the project in `WorkspaceRegistry`.
  - If the repository exists and is a specs repo, the HostServer shall clone the specs repository locally, register the project, and clone/pull any declared linked repositories into `<workspace_root>/codebase/<id>`.
  - If local-only is set, the HostServer shall scaffold local directories, initialize Git (`git init`), and register the project without configuring a remote or contacting GitHub.
- **REQ-010**: When an AI agent calls the `create_project` MCP tool on `crates/aico-mcp` with arguments `project_name`, optional `display_name`, optional `url`, optional `repositories`, optional `local_only`, and optional `private`, the MCP server shall invoke the unified project scaffolding and GitHub synchronization engine, returning structured project metadata.

### Unwanted Behavior & Error Handling
- **REQ-011**: If GitHub repository creation fails (e.g. invalid token, insufficient permissions, name conflict), then the HostServer shall return HTTP `400 Bad Request` or `502 Bad Gateway` with a descriptive error message without leaving broken directory state on disk.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **GitHub Auth, Inspector & Codebase Discovery Seam (`crates/aico-server/tests/github_api_test.rs`)**:
   - Signature: `GET /api/v1/github/status`, `POST /api/v1/github/token`, `POST /api/v1/github/check-repo`, and `GET /api/v1/workspace/codebase-folders`
   - Verifies: Hybrid credential resolution, token validation (`REQ-004`), repository existence and specs manifest extraction (`REQ-008`), and codebase directory enumeration (`REQ-003`).
2. **Project Scaffolding & MCP Creation Seam (`crates/aico-server/tests/project_api_test.rs` & `crates/aico-mcp/tests/mcp_server_test.rs`)**:
   - Signature: `POST /api/v1/projects` and MCP tool `create_project`
   - Verifies: Local-only scaffolding (`REQ-006`), remote GitHub repo creation & push (`REQ-007`, `REQ-009`), remote specs clone with linked repos (`REQ-008`), error recovery (`REQ-011`), and agent MCP tool execution (`REQ-010`).
3. **Frontend UI Seam (`frontend/tests/ProjectSwitcher.test.tsx` & `frontend/tests/SettingsModal.test.tsx`)**:
   - Signature: `<ProjectSwitcher />`, `<ProjectConfigModal />`, and `<SettingsModal />`
   - Verifies: Rebranded "Add Project" UI (`REQ-001`), removal of default text/type dropdown (`REQ-002`), codebase folder browse picker (`REQ-003`), GitHub settings card (`REQ-004`), dynamic URL pre-fill and permissive warning (`REQ-005`), local-only toggle (`REQ-006`), private toggle (`REQ-007`), and "Check Repo" inspection states (`REQ-008`).

---

## 5. Open Clarifications

None. Ratified in [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md) and [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
