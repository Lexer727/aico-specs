---
id: spec-ft-git-backend-status
type: spec
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 1
title: "Multi-Repo Manifest Discovery & Git Status Inspection Engine"
status: "Completed"
derives_from: ["brain-ec-git"]
created_at: "2026-10-07T00:26:00Z"
---

# Feature Specification: Multi-Repo Manifest Discovery & Git Status Inspection Engine

## 1. Problem Statement & User Value

Engineering workspaces commonly involve multiple Git repositories (e.g. implementation codebase, specifications repo, and shared skills repo). The AICO host daemon currently tracks only a single `codebase_path` and possesses no Git introspection capabilities.

This feature introduces multi-repository declarations in `.aico-project.json`, an asynchronous Rust Git status scanner that queries local Git repositories, and a REST endpoint `GET /api/v1/git/status` to serve live repository telemetry.

---

## In-Scope
- Extending `ProjectManifest` in `aico-core` to parse optional `repositories: [...]` array with fallback to specs root, eliminating mandatory `codebase_path` and `version` constraints.
- Supporting optional remote web `url` (e.g. GitHub URL) on repository declarations and propagating it to Git status responses.
- Implementing `GitStatusScanner` in `aico-server` using asynchronous Tokio subprocesses running `git status --porcelain=v2 --branch` and `git log -1`.
- Parsing branch name, upstream tracking branch, ahead/behind counts, dirty/clean status, and latest commit info.
- Exposing REST API `GET /api/v1/git/status`.
- Graceful degradation when a repository path is missing or not a git repository.
- Rendering clickable external repository links in the WebUI Dashboard.

---

## Out-of-Scope
- Executing git mutations (commit, push, pull, checkout) from the server.
- Interactive git conflict resolution or merge tooling.
- Managing remote Git credentials or SSH key generation.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer with multiple codebase repos, I want to list them in `.aico-project.json`, so that AICO discovers and monitors all relevant repositories.
   - *Independent Test*: Provide an `.aico-project.json` with 3 repositories and verify `ProjectManifest::from_json_str` registers all 3.
2. **US-02** (Priority: P1): As a developer or agent, I want to query `GET /api/v1/git/status`, so that I can see whether repositories are clean and how many commits they are ahead or behind upstream.
   - *Independent Test*: Run a curl request against `GET /api/v1/git/status` and verify response contains accurate branch, ahead/behind counts, and clean flag.
3. **US-03** (Priority: P2): As a developer opening a non-git directory, I want the scanner to return an error status rather than failing the entire server, so that other repositories continue functioning.
   - *Independent Test*: Pass an invalid path to `GitStatusScanner` and verify it returns a structured `unavailable` response without crashing.
4. **US-04** (Priority: P2): As a developer with remote GitHub repositories, I want repository cards on the dashboard to link directly to GitHub, so that I can open the remote repo in one click.
   - *Independent Test*: Declare `"url": "https://github.com/..."` in `.aico-project.json` and verify repository card renders a clickable external link.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: When parsing `.aico-project.json`, the ManifestParser shall support a project manifest containing a required project `name` identifier and an optional `repositories` array, where each repository contains `id`, `name`, `path`, `type`, and an optional remote `url` (e.g. GitHub link). Fields `version`, `created_at`, and `codebase_path` shall be completely optional and unconstrained.
- **REQ-002**: Where the `repositories` array is omitted or empty in `.aico-project.json`, the ManifestParser shall default to registering the specifications repository itself (`specs_path`), eliminating any mandatory requirement for a `codebase_path` or local AICO engine path.
- **REQ-003**: The GitStatusScanner shall execute asynchronous CLI commands `git status --porcelain=v2 --branch` within each declared repository path.
- **REQ-004**: The GitStatusScanner shall extract the current branch, upstream tracking remote, commit count ahead, commit count behind, and working tree cleanliness.
- **REQ-005**: The HostServer shall expose `GET /api/v1/git/status` returning the array of Git repository statuses for the active project.
- **REQ-006**: If a declared repository path does not contain a valid `.git` directory or git execution fails, the GitStatusScanner shall return a structured `is_available: false` state with the associated error description.
- **REQ-007**: The `GET /api/v1/git/status` endpoint shall include the declared `url` in each repository status object, and the WebUI dashboard (`GitRepositoriesCard`) shall render clickable external links to remote repositories.

---

## 4. Designated Public Test Seams

- **Manifest Seam**: `crates/aico-core/tests/manifest_test.rs::test_manifest_multi_repo_parsing`
- **Scanner Seam**: `crates/aico-server/tests/git_scanner_test.rs::test_git_status_inspection`
- **Server Route Seam**: `crates/aico-server/tests/server_test.rs::test_git_status_endpoint`
