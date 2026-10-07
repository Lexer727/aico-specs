---
id: spec-ft-git-relative-paths-manifest
type: spec
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 4
title: "Manifest Relative Repository Paths & Workspace Portability"
status: "Completed"
derives_from: ["brain-ec-git"]
created_at: "2026-10-07T01:05:00Z"
---

# Feature Specification: Manifest Relative Repository Paths & Workspace Portability

## 1. Problem Statement & User Value

Absolute paths in `.aico-project.json` (such as `d:/dev/aico`) prevent projects and workspace repositories from being portable across machines, operating systems, and cloned checkout paths. Additionally, declaring repositories with relative paths (e.g. `../codebase/aico` and `.`) allows teams to share manifest definitions without machine-specific hardcoding.

This feature updates `.aico-project.json` to canonical relative paths and asserts that relative path resolution in `aico-core` correctly resolves `.` to the active specs root and `../` paths to external sibling codebases.

---

## In-Scope
- Updating `.aico-project.json` to declare relative paths: `../codebase/aico`, `.`, and `../codebase/aico-skills`.
- Verifying `ProjectManifest::get_effective_repositories` resolves `.` to specs root and `../` relative paths against the specs directory.
- Ensuring `GitStatusScanner` operates seamlessly against resolved relative paths.
- Unit and contract test coverage in `crates/aico-core/tests/manifest_test.rs`.

---

## Out-of-Scope
- Rewriting remote Git URLs or credentials.
- Automatic creation of missing directories.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer checking in `.aico-project.json`, I want all repository paths to be relative, so that another developer cloning the repo can use AICO without modifying absolute drive letters.
   - *Independent Test*: Inspect `.aico-project.json` and confirm all `path` entries are relative (`.` or `../`).
2. **US-02** (Priority: P1): As the AICO server daemon, I want to resolve relative repository paths against the specs directory, so that `GitStatusScanner` locates the correct directories.
   - *Independent Test*: Parse a manifest with relative paths and verify `get_effective_repositories` returns normalized paths matching target directories.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: The Project Manifest (`.aico-project.json`) shall declare repository paths as portable relative paths.
- **REQ-002**: When parsing declared repositories, the ManifestParser shall resolve relative paths against the active specs root directory.
- **REQ-003**: Where a repository path is declared as `.` or `./`, the ManifestParser shall resolve the path to the specs root directory itself.
- **REQ-004**: The GitStatusScanner shall execute git status inspections against the resolved relative repository paths.

---

## 4. Designated Public Test Seams

- **Manifest Seam**: `crates/aico-core/tests/manifest_test.rs::test_manifest_relative_repositories_resolution`
