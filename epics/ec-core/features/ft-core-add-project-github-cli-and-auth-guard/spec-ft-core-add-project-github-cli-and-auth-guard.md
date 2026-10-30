---
id: spec-ft-core-add-project-github-cli-and-auth-guard
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 21
title: "GitHub CLI Login, Unauthenticated Private Repo Guard & Add Project Defaults"
status: "Completed"
derives_from: ["spec-ft-core-add-project-and-github-scaffolding", "ADR-0004", "ADR-0008"]
created_at: "2026-10-30T17:55:00.000Z"
---

# Feature Specification: GitHub CLI Login, Unauthenticated Private Repo Guard & Add Project Defaults

## 1. Problem Statement & User Value

The initial GitHub automation flow delivered in Order 20 established foundational repository inspection, scaffolding, and PAT management. However, testing and developer evaluation highlighted three usability and security ambiguities:

1. **Unauthenticated Private Repository Ambiguity**: The GitHub REST API intentionally returns HTTP `404 Not Found` for private repositories when accessed without authentication (or with insufficient permissions) to prevent username and repository enumeration. When an unauthenticated user checks a URL in AICO, treating `404` as a non-existent repository causes AICO to claim: `"Repository does not exist on GitHub yet. It will be created automatically on Add Project."` This is factually misleading for existing private repositories and creates false expectations, since unauthenticated users cannot create remote repositories on GitHub anyway.
2. **Awkward `<username>` URL Placeholder**: When unauthenticated, typing a project name auto-prefilled `https://github.com/<username>/<slug>-specs.git`. Inserting a literal `<username>` placeholder into a functional input field is awkward and produces broken Git commands if submitted. Instead, unauthenticated users should default to **Local repository only** mode, without generating dummy URL prefixes, and should be provided an actionable link to connect their GitHub account in Settings.
3. **Lack of GitHub CLI (`gh`) Integration in Settings UI**: While GitHub CLI (`gh.exe`) is installed on the system, the Settings modal only offered manual Personal Access Token (PAT) entry. Developers who have already authenticated via `gh` CLI or prefer web-based browser login should be able to detect `gh`, log in via `gh auth login`, and sync their credentials into AICO with a single click in the WebUI.

Addressing these issues makes project onboarding predictable, eliminates unauthenticated private repo confusion, and provides seamless one-click authentication through the GitHub CLI.

---

## In-Scope

- Robust GitHub CLI (`gh`) discovery in `crates/aico-core/src/github.rs` supporting system PATH and standard Windows installation directories (`C:\Program Files\GitHub CLI\gh.exe`, `C:\Program Files (x86)\GitHub CLI\gh.exe`, and user AppData).
- HostServer endpoints for GitHub CLI integration:
  - `GET /api/v1/github/cli/status`: Reports binary presence, binary path, CLI authentication state (`gh auth status`), and logged-in username.
  - `POST /api/v1/github/cli/sync`: Extracts token from `gh auth token`, validates profile against GitHub REST API, persists token to `~/.aico/config.json`, and returns verified profile.
- Updating `POST /api/v1/github/check-repo` to explicitly distinguish unauthenticated 404 responses from verified non-existence when no token is configured: returning `{ exists: false, unauthenticated: true, error: "..." }`.
- Overhauling `SettingsModal.tsx` GitHub card to provide dual login modes:
  - **GitHub CLI (`gh`)**: Displays detected CLI path, login status, and a "Connect with GitHub CLI" button.
  - **Personal Access Token (PAT)**: Manual token input with show/hide toggle.
- Overhauling `ProjectConfigModal.tsx` unauthenticated behavior:
  - Defaulting to **Local repository only** (`localOnly = true`) when no GitHub account is connected.
  - Omitting the awkward `<username>` URL placeholder entirely.
  - Rendering an informational alert notice when unchecking local-only while unauthenticated, with a direct button opening the Settings modal.
  - Rendering an explicit authentication notice in the "Check Repo" inspection feedback banner when checking repositories while unauthenticated.

---

## Out-of-Scope

- Custom OAuth server callback redirection outside the official GitHub CLI web flow (`gh auth login --web`).
- Support for alternative VCS host CLIs (e.g. `glab` for GitLab).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer who is not logged in to GitHub, I want the Add Project modal to default to local-only mode without awkward `<username>` placeholders, so that I can immediately create local projects without distraction.
   - *Independent Test*: Open Add Project modal without GitHub credentials configured; verify "Local repository only" is checked by default and no `<username>` URL is present.
2. **US-02** (Priority: P1): As a developer checking a private repository URL while unauthenticated, I want AICO to explain that authentication is required to access private repos, so that I don't mistakenly think the repository does not exist.
   - *Independent Test*: Check a repository URL while unauthenticated; verify check response indicates unauthenticated status and UI banner prompts to connect in Settings.
3. **US-03** (Priority: P1): As a developer with GitHub CLI installed, I want to click "Connect with GitHub CLI" in Settings, so that AICO automatically imports my authenticated GitHub credentials without requiring manual PAT copy-pasting.
   - *Independent Test*: In Settings modal, click "Connect with GitHub CLI"; verify `POST /api/v1/github/cli/sync` succeeds and UI displays `✓ Connected as @<username> (ghcli)`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: While a user is not authenticated with GitHub, opening the Add Project modal shall default the "Local repository only" checkbox to checked (`true`), shall leave the specs repository URL blank, and shall not prefill any dummy `<username>` placeholder.
- **REQ-002**: The system shall implement `find_gh_binary()` in `aico-core` to resolve the GitHub CLI executable across the system PATH and standard Windows installation paths (`C:\Program Files\GitHub CLI\gh.exe`, `C:\Program Files (x86)\GitHub CLI\gh.exe`, and local AppData).
- **REQ-003**: The HostServer shall expose `GET /api/v1/github/cli/status` returning JSON with `installed: bool`, `binary_path: Option<String>`, `logged_in: bool`, and `username: Option<String>`.

### Event-Driven Requirements
- **REQ-004**: When a user unchecks "Local repository only" in `ProjectConfigModal` while unauthenticated, the modal shall render an informational notice stating that GitHub authentication is required to check private repositories or create remote repositories, including an actionable button to open the Settings modal.
- **REQ-005**: When `POST /api/v1/github/check-repo` checks a repository while no GitHub credentials are resolved, if the GitHub REST API returns HTTP `404 Not Found`, the HostServer shall return `{ exists: false, unauthenticated: true, error: "Repository was not found (or is private). Sign in to GitHub in Settings to verify private repositories or scaffold remote repositories." }`, and the WebUI shall render an explanatory authentication prompt banner.
- **REQ-006**: When `POST /api/v1/github/cli/sync` is called, the HostServer shall execute `gh auth token` using the resolved `gh` binary, validate the token against `https://api.github.com/user`, save the token and username into `~/.aico/config.json`, and return `{ status: "ok", username: String }`.
- **REQ-007**: When a user clicks "Connect with GitHub CLI" in `SettingsModal`, the WebUI shall invoke `POST /api/v1/github/cli/sync` and refresh the connection badge upon completion.
- **REQ-008**: The `SettingsModal` GitHub Integration card shall offer selectable options for GitHub CLI (`gh`) and Personal Access Token (PAT), displaying the resolved CLI status and enabling instant CLI synchronization.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **GitHub CLI & Unauthenticated Check Seam (`crates/aico-server/tests/github_cli_api_test.rs`)**:
   - Signature: `GET /api/v1/github/cli/status`, `POST /api/v1/github/cli/sync`, and `POST /api/v1/github/check-repo` unauthenticated 404 handling.
   - Verifies: `REQ-002`, `REQ-003`, `REQ-005`, `REQ-006`.
2. **Frontend UI Seam (`frontend/tests/ProjectSwitcher.test.tsx` & `frontend/tests/SettingsModal.test.tsx`)**:
   - Signature: `<ProjectConfigModal />` unauthenticated defaults (`REQ-001`), callout notice with Settings button (`REQ-004`), and `<SettingsModal />` GitHub CLI connect flow (`REQ-007`, `REQ-008`).

---

## 5. Open Clarifications

None. Approved in architectural triage.
