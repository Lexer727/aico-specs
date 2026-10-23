---
id: plan-ft-specs-parallel-subagent-execution
type: plan
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 7
title: "Parallel Subagent Execution & Atomic CLI Plan Locking Implementation Plan"
status: "Completed"
repositories: ["aico-skills"]
derives_from: ["spec-ft-specs-parallel-subagent-execution"]
created_at: "2026-10-23T19:05:21.686Z"
---

# Implementation Plan: Parallel Subagent Execution & Atomic CLI Plan Locking

## 1. Technical Architecture & Decisions

- **Target Repository**: `aico-skills` (mirrored to `.agents/skills`).
- **Atomic Plan Locking in `aico-cli.mjs`**:
  - Implement `withFileLock(targetPath, fn)` in `codebase/aico-skills/scripts/aico-cli.mjs` using `fs.openSync(lockPath, 'wx')`, a spin/sleep retry up to 5,000ms, stale lock eviction (>10,000ms), and `finally { fs.unlinkSync(lockPath) }`.
  - Wrap `cmdTaskStatus` inside `withFileLock` so concurrent subagents updating different `TASK-*` entries in the same `plan-*.md` never race or overwrite each other's changes.
- **Parallel Subagent Orchestration & Model Selection in Skills**:
  - Update `aico-planning/SKILL.md`, `aico-implementation/SKILL.md`, `aico-feature/SKILL.md`, and `aico-epic/SKILL.md` to specify:
    1. Identifying parallelizable items on the Ready Frontier (`ready_frontier.length > 1` or multiple unblocked features in an epic) with disjoint file/test seams.
    2. Offering the user the option to run unblocked features/tasks in parallel using `invoke_subagent`, and asking which tasks should use which model tier — defaulting to `Model: "flash"` (the top normal Gemini model, currently **Gemini 3.8 Flash**) while allowing per-task overrides to `Model: "inherit"` (inherits the parent agent's active model, e.g., **Gemini 4 Argon High**) or `Model: "pro"`.
    3. Scoping each subagent prompt strictly to its assigned `TASK-*` or `ft-*`, prohibiting subagents from editing files outside their seam, and collecting completion reports before running the parent `aico-review-loop` / `aico-verify` check.

---

## 2. Task Graph (DAG)

### Phase 1: Atomic Plan Locking in `aico-cli.mjs`

- **TASK-SPECS-007-1**: Add atomic lockfile synchronization to `aico-cli.mjs task-status`
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-005`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Implements `withFileLock` in `codebase/aico-skills/scripts/aico-cli.mjs` and syncs to `.agents/skills/scripts/aico-cli.mjs`.
    - Verifies concurrent `task-status` invocations against a shared `plan-*.md` preserve all updates.
  - **Verification Seam**: `codebase/aico-skills/tests/aico-cli.test.mjs`

### Phase 2: Parallel Subagent Skill Protocol & Model Selection

- **TASK-SPECS-007-2**: Update `aico-epic`, `aico-feature`, `aico-implementation`, and `aico-planning` skills with Parallel Subagent Execution and explicit `<Version> <Model> <Effort>` Role tags (`(3.8 Flash High)` vs `(4 Argon High)`)
  - **Status**: Done
  - **Addresses**: `REQ-002`, `REQ-003`, `REQ-004`, `REQ-006`
  - **Blocked by**: `TASK-SPECS-007-1`
  - **What it delivers**:
    - Updates `subagent-model-router.mjs` and `aico-epic/SKILL.md`, `aico-feature/SKILL.md`, `aico-implementation/SKILL.md`, and `aico-planning/SKILL.md` in both `codebase/aico-skills` and `.agents/skills` so every subagent `Role` parameter includes the explicit version, model name, and thinking level: e.g., `<Role Title> (3.8 Flash High)` by default and `<Role Title> (4 Argon High)` for complex overrides.
  - **Verification Seam**: `codebase/aico-skills/tests/parallel-subagents-skill.test.mjs`
