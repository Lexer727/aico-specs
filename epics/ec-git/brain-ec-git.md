---
id: brain-ec-git
type: brainstorm
aico_skills_version: "0.1.2"
epic: "ec-git"
title: "AICO Multi-Repo Git Integration & Workspace Traceability"
status: "In Progress"
derives_from: ["ADR-0002", "ADR-0008"]
created_at: "2026-10-07T00:25:00Z"
---

# Brainstorm: AICO Multi-Repo Git Integration & Workspace Traceability

## 1. Executive Summary & Problem Statement

Modern software projects developed with AI agents frequently span multiple git repositories—for example, an implementation engine repository (`d:/dev/aico`), a dedicated specifications repository (`d:/aico/aico-specs`), and a shared skills suite repository (`d:/aico/codebase/aico-skills`). 

Currently, AICO exhibits several limitations in multi-repository management:
1. **Zero Git Visibility on the Dashboard**: The home dashboard displays project metadata and stats, but provides no visibility into the Git status of underlying repositories (current branch, uncommitted/dirty changes, commits ahead/behind upstream remote HEAD).
2. **Single-Path Manifest Constraint**: `.aico-project.json` exposes only a single `codebase_path`, lacking a clean convention to declare auxiliary repositories (e.g. skills repo, client frontend, microservices).
3. **Missing Spec-to-Codebase Traceability**: Developers and agents have no standardized, single-source mechanism to declare which specific code repositories are targeted or touched by a given feature or implementation plan.
4. **Fragmented UI Context**: Epics and features do not communicate which repositories are relevant to them, forcing developers to manually inspect git remotes across separate terminal shells.

This epic establishes native Git integration across the AICO stack: a fast background Git inspection engine in the Rust Axum daemon, single-source repository declarations in `plan-ft-*.md` frontmatter that dynamically roll up to Features and Epics, and real-time Git status telemetry cards on the WebUI Home Dashboard.

---

## 2. Core Architectural Decisions

### 2.1 Single Source of Truth: Plan Frontmatter with Dynamic Roll-Up
To prevent configuration drift and duplicate maintenance:
- **Canonical Declaration Point**: Line-1 YAML Frontmatter of `plan-ft-*.md`:
  ```yaml
  repositories: ["aico", "aico-specs"]
  ```
  *Rationale*: Specifications define behavioral requirements (*what* the system does) independent of concrete repositories. Implementation plans define *how and where* code is delivered, making the plan frontmatter the sole authority.
- **Dynamic Propagation**:
  - **Feature Viewer**: Automatically derives target repositories from its plan.
  - **Epic Hub**: Automatically rolls up the union of all repositories declared across child feature plans.
  - **Home Dashboard**: Displays all project repositories with live badges indicating which active features/plans currently touch each repository.

### 2.2 Manifest Evolution: `.aico-project.json` Multi-Repo Support
Extend the project manifest while preserving 100% backward compatibility:
```json
{
  "name": "aico",
  "version": "0.1.1",
  "codebase_path": "../codebase/aico",
  "repositories": [
    {
      "id": "aico",
      "name": "AICO Engine & WebUI",
      "path": "d:/dev/aico",
      "type": "codebase"
    },
    {
      "id": "aico-specs",
      "name": "AICO Specs",
      "path": "d:/aico/aico-specs",
      "type": "specs"
    },
    {
      "id": "aico-skills",
      "name": "AICO Skills Suite",
      "path": "d:/aico/codebase/aico-skills",
      "type": "skills"
    }
  ]
}
```
*Fallback*: If `"repositories"` is absent, the server automatically defaults to two repositories: `codebase` (from `codebase_path`) and `specs` (from the active specs root).

### 2.3 Lightweight CLI Git Scanner in Rust
- Executes asynchronous `git status --porcelain=v2 --branch` via `tokio::process::Command`.
- Zero extra C/C++ dependencies (no `libgit2` build overhead).
- Natively inherits developer SSH keys, Git credentials, and global configurations.
- Sub-15ms parsing latency providing:
  - Current branch name and upstream tracking reference.
  - Number of commits ahead / behind upstream HEAD.
  - Working tree cleanliness (staged, unstaged, untracked counts).
  - Latest commit hash, message, author, and timestamp.

---

## 3. Epic Decomposition & Child Features

This epic decomposes into 3 cohesive child features:

1. **`ft-git-backend-status`** (`spec-ft-git-backend-status`, `plan-ft-git-backend-status`, `order: 1`):
   - Manifest parsing extension for `repositories` in `aico-core`.
   - Rust asynchronous Git status scanner in `aico-server`.
   - REST API endpoint `GET /api/v1/git/status` and WebSocket status broadcaster.

2. **`ft-git-skills-traceability`** (`spec-ft-git-skills-traceability`, `plan-ft-git-skills-traceability`, `order: 2`):
   - Skills harness update: add `repositories: [...]` to `PLAN-TEMPLATE.md` and `aico-planning/SKILL.md`.
   - Dynamic roll-up logic in verification scripts and documentation linters.

3. **`ft-git-dashboard-ui`** (`spec-ft-git-dashboard-ui`, `plan-ft-git-dashboard-ui`, `order: 3`):
   - Home Dashboard "Codebase Repositories" telemetry card with branch, ahead/behind pills, and dirty status.
   - Epic Hub "Relevant Repositories" badge section.
   - Feature Viewer repository target tags.

---

## 4. Milestone Convergence Criteria

- [ ] All 3 child features specified and planned with vertical tracer-bullet DAGs.
- [ ] Passing unit and integration tests across Rust backend and React frontend.
- [ ] Home Dashboard dynamically displays real-time Git status for `aico`, `aico-specs`, and `aico-skills`.
- [ ] Child plans declare `repositories: [...]` matching project manifest IDs.
- [ ] Epic status transitions to `Completed` when all child feature DAGs are green.
