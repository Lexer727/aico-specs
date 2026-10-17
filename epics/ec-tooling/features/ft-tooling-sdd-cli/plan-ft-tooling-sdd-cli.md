---
id: plan-ft-tooling-sdd-cli
type: plan
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 1
title: "Unified Deterministic SDD CLI (aico-cli.mjs) Implementation Plan"
status: "Completed"
repositories: ["aico-skills", "aico-specs"]
derives_from: ["spec-ft-tooling-sdd-cli"]
created_at: "2026-10-17T21:14:00Z"
---

# Implementation Plan: Unified Deterministic SDD CLI (aico-cli.mjs)

## 1. Technical Architecture & Decisions

- **Runtime**: Zero-dependency Node.js ESM (`node:fs`, `node:path`, `node:url`, `node:child_process`) at `codebase/aico-skills/scripts/aico-cli.mjs`.
- **Test Runner**: Built-in Node.js test runner (`node --test codebase/aico-skills/tests/aico-cli.test.mjs`) requiring zero external npm packages.
- **Backward Compatibility**: `codebase/aico-skills/scripts/get-version.mjs` and `codebase/aico-skills/aico-verify/scripts/verify-artifacts.mjs` remain callable at their existing paths while sharing/delegating core logic with `scripts/aico-cli.mjs`.

---

## 2. Task Graph (DAG)

### Phase 1: CLI Core & Scaffolding Engine

- **TASK-TOOL-001-1**: CLI Entry Point, Version & SDD Scaffolding Engine
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`, `REQ-005`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Creates `codebase/aico-skills/scripts/aico-cli.mjs` with argument parser and subcommand router.
    - Implements `version`, `scaffold-epic`, `scaffold-feature`, `scaffold-adr`, and `upsert-glossary`.
    - Enforces lowercase `ec-`/`ft-` naming, dynamic `aico_skills_version` resolution, auto-incremented `order`, `.aico-project.json` repository validation, and depth-aware relative ADR links.
  - **Verification Seam**: `../aico-skills/tests/aico-cli.test.mjs::test_scaffold_commands`

---

### Phase 2: Task Lifecycle Mutator & Automatic Status Roll-Up

- **TASK-TOOL-001-2**: Task Status Mutator, Completion Gate & Frontmatter Roll-Up
  - **Status**: Done
  - **Addresses**: `REQ-006`, `REQ-010`, `REQ-011`
  - **Blocked by**: `TASK-TOOL-001-1`
  - **What it delivers**:
    - Implements `task-status` subcommand updating task blocks in `plan-ft-*.md`.
    - Enforces mandatory `--reason` (for `Skipped/Canceled`) and `--reason` + `--rewrite-target` (for `Rewritten`).
    - Implements `TaskCompletionGate` verifying seam existence on disk and `@verifies`/`@implements` coverage before allowing `Done` when `--code-dir` is passed.
    - Automatically unblocks downstream `Blocked` tasks and rolls up `In Progress` / `Completed` status to `plan-ft-*.md`, `spec-ft-*.md`, and parent `brain-ec-*.md`.
  - **Verification Seam**: `../aico-skills/tests/aico-cli.test.mjs::test_task_status_and_rollup`

---

### Phase 3: Plan Amendment & Dynamic Frontier Engine

- **TASK-TOOL-001-3**: Non-Destructive Plan Amendment & DAG Frontier Resolver
  - **Status**: Done
  - **Addresses**: `REQ-007`, `REQ-008`, `REQ-012`
  - **Blocked by**: `TASK-TOOL-001-2`
  - **What it delivers**:
    - Implements `plan-amend` subcommand auto-computing sequential `TASK-<PREFIX>-<SEQ>` IDs, appending task blocks, maintaining a single `## 3. Amendment Log` section, and reopening plan/spec/epic statuses to `"In Progress"`.
    - Implements `frontier` subcommand parsing `Blocked by` edges across plans or epics, detecting circular or missing dependencies, and outputting unblocked `Pending` tasks.
  - **Verification Seam**: `../aico-skills/tests/aico-cli.test.mjs::test_plan_amend_and_frontier`

---

### Phase 4: Unified Verification, Trace Tags & Legacy Script Delegation

- **TASK-TOOL-001-4**: Unified `verify` & `trace-tags` Subcommands and Wrapper Parity
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-009`, `REQ-012`
  - **Blocked by**: `TASK-TOOL-001-1`
  - **What it delivers**:
    - Implements `trace-tags` subcommand emitting canonical `@verifies` and `@implements` comments for any spec or task.
    - Integrates `verify-artifacts.mjs` dual-axis verification (enhanced with multi-repo path resolution from `.aico-project.json` and DAG cycle checks) into `aico-cli.mjs verify`.
    - Ensures `scripts/get-version.mjs` and `aico-verify/scripts/verify-artifacts.mjs` delegate cleanly without breaking existing callers.
  - **Verification Seam**: `../aico-skills/tests/aico-cli.test.mjs::test_verify_and_trace_tags`
