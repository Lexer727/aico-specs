---
id: plan-ft-git-skills-traceability
type: plan
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 2
title: "Plan Frontmatter Multi-Repo Traceability & Blueprint Standards Plan"
status: "Pending"
repositories: ["aico-skills", "aico-specs"]
derives_from: ["spec-ft-git-skills-traceability"]
created_at: "2026-10-07T00:33:00Z"
---

# Implementation Plan: Plan Frontmatter Multi-Repo Traceability & Blueprint Standards

## 1. Technical Architecture & Decisions

- **Single Source of Truth**: All codebase repository traceability is anchored in `plan-ft-*.md` frontmatter via `repositories: ["<repo_id>", ...]`. Specification files (`spec-ft-*.md`) remain purely behavioral and repository-neutral.
- **Skills & Template Updates**: `PLAN-TEMPLATE.md` will explicitly declare `repositories: ["{repo_id}"]` so that newly scaffolded plans include repository bindings by default. `aico-planning/SKILL.md` will document the dynamic roll-up mechanics.
- **Linter Validation in `verify-artifacts.mjs`**: The verification engine will inspect plan frontmatter and warn if `repositories` is missing, malformed, or references repository IDs not present in `.aico-project.json`.

---

## 2. Task Graph (DAG)

### Phase 1: Planning Skill & Template Updates

- **TASK-GIT-002-1**: Planning Skill Template & Documentation Alignment
  - **Status**: Pending
  - **Addresses**: REQ-001, REQ-002
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Updated `PLAN-TEMPLATE.md` with `repositories: ["{repo_id}"]` and quoted status.
    - Updated `aico-planning/SKILL.md` detailing frontmatter repository declarations and dynamic roll-up rules.
  - **Verification Seam**: `d:/aico/codebase/aico-skills/aico-planning/PLAN-TEMPLATE.md`

### Phase 2: Verification Engine Linting

- **TASK-GIT-002-2**: Artifact Verification Linter Extension for Repository Declarations
  - **Status**: Blocked
  - **Addresses**: REQ-003, REQ-004
  - **Blocked by**: TASK-GIT-002-1
  - **What it delivers**:
    - Extended `verify-artifacts.mjs` with checks for plan `repositories` arrays.
    - Manifest cross-referencing validating declared IDs against `.aico-project.json`.
  - **Verification Seam**: `d:/aico/codebase/aico-skills/aico-verify/scripts/verify-artifacts.mjs`
