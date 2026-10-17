---
id: spec-ft-tooling-sdd-cli
type: spec
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 1
title: "Unified Deterministic SDD CLI (aico-cli.mjs)"
status: "Completed"
derives_from: ["brain-ec-tooling", "ADR-0009", "ADR-0010"]
created_at: "2026-10-17T21:14:00Z"
---

# Feature Specification: Unified Deterministic SDD CLI (aico-cli.mjs)

## 1. Problem Statement & User Value

When LLM agents manually create folders, construct YAML frontmatter, compute relative link depths, transition task statuses, and append plan amendments via text editing, they waste tokens and frequently introduce subtle schema violations (wrong relative ADR depth, missing frontmatter fields, unrolled plan/spec statuses, or duplicate amendment headers).

This specification defines `aico-cli.mjs`, a unified, zero-dependency Node.js CLI entry point bundled inside `aico-skills/scripts/` that deterministically executes all stateless SDD filesystem, scaffolding, task lifecycle, DAG frontier, and verification operations while preserving backward compatibility with existing script entry points (`get-version.mjs` and `verify-artifacts.mjs`).

---

## In-Scope
- Unified CLI entry point (`scripts/aico-cli.mjs`) with subcommands: `version`, `scaffold-epic`, `scaffold-feature`, `scaffold-adr`, `upsert-glossary`, `task-status`, `plan-amend`, `frontier`, `trace-tags`, and `verify`.
- Deterministic directory and skeleton artifact scaffolding conforming to Blueprint v0.1.2 ([ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md) and [ADR-0010](../../../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md)).
- In-place task status mutations with 6-state lifecycle enforcement, downstream blocker unblocking, and automatic status roll-up to parent plan, spec, and epic frontmatter.
- Non-destructive plan task amendment and consolidated single-header `## 3. Amendment Log` maintenance.
- Dynamic Active Frontier evaluation and DAG cycle detection.
- Backward-compatible wrappers for `scripts/get-version.mjs` and `aico-verify/scripts/verify-artifacts.mjs`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an agent scaffolding an epic or feature, I want a single CLI command to generate canonical folders, frontmatter, sibling `order`, `repositories`, and relative ADR links, so that new artifacts never fail blueprint validation.
   - *Independent Test*: Run `node scripts/aico-cli.mjs scaffold-feature --specs-dir <tmp> --epic ec-core --slug auth --title "Auth" --repos aico` and verify `spec-ft-core-auth.md` and `plan-ft-core-auth.md` are created with valid v0.1.2 frontmatter.
2. **US-02** (Priority: P1): As an agent completing a task, I want a CLI command to update the task status, verify its seam/annotations, unblock dependent tasks, and roll up frontmatter statuses automatically, so that plan and spec metadata never drift.
   - *Independent Test*: Transition all tasks in a sample plan to `Done` via `task-status` and verify the plan and spec frontmatter transition to `status: "Completed"`.
3. **US-03** (Priority: P1): As an agent amending a plan or picking the next task, I want CLI subcommands to append amended tasks and query the active unblocked DAG frontier, so that historical audit trails and dependency ordering are strictly preserved.
   - *Independent Test*: Run `plan-amend` on a `Completed` plan and verify a new sequential task is added, `## 3. Amendment Log` is updated without duplicate headers, and the plan reopens to `status: "In Progress"`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `AicoCli` shall expose a unified zero-dependency Node.js CLI entry point at `scripts/aico-cli.mjs` supporting subcommands `version`, `scaffold-epic`, `scaffold-feature`, `scaffold-adr`, `upsert-glossary`, `task-status`, `plan-amend`, `frontier`, `trace-tags`, and `verify`.
- **REQ-002**: The `AicoCli` shall maintain backward compatibility for `scripts/get-version.mjs` and `aico-verify/scripts/verify-artifacts.mjs` by delegating or supporting existing CLI flags (`--fix`, `--json`, `--verbose`).

### Event-Driven Requirements
- **REQ-003**: When `scaffold-epic` is invoked with `--specs-dir`, `--id`, and `--title`, the `ScaffoldingEngine` shall create `epics/ec-<id>/features/` and write `brain-ec-<id>.md` with Line-1 YAML frontmatter (`status: "Draft"` or `"In Progress"`), dynamically resolved `aico_skills_version`, and relative ADR links.
- **REQ-004**: When `scaffold-feature` is invoked for a standalone or epic-child feature, the `ScaffoldingEngine` shall enforce the `ft-` prefix, calculate the next sequential `order` integer in the container, validate declared `--repos` against `.aico-project.json`, and generate `spec-ft-<slug>.md` and `plan-ft-<slug>.md` (plus optional `dive-ft-<slug>.md` and `brain-ft-<slug>.md`) with canonical section headings and `status: "Draft"`.
- **REQ-005**: When `scaffold-adr` or `upsert-glossary` is invoked, the `ScaffoldingEngine` shall create the next sequential `adr/00XX-<slug>.md` document or insert/update the domain term entry in `glossary.md`.
- **REQ-006**: When `task-status` updates a task in `plan-ft-*.md`, the `TaskLifecycleMutator` shall update the task's `Status` field, transition any downstream `Blocked` tasks whose `Blocked by` prerequisites are all `Done` to `Pending`, and roll up `status: "In Progress"` or `status: "Completed"` to the enclosing `plan-ft-*.md`, sibling `spec-ft-*.md`, and parent `brain-ec-*.md` frontmatter.
- **REQ-007**: When `plan-amend` is invoked on a plan, the `PlanAmender` shall assign the next sequential `TASK-<PREFIX>-<SEQ>` identifier, append the formatted task block, append a timestamped entry to the single consolidated `## 3. Amendment Log` section, and reopen the plan, spec, and parent epic frontmatter status to `"In Progress"`.
- **REQ-008**: When `frontier` is invoked, the `DagFrontierResolver` shall evaluate `Status` and `Blocked by` edges across the target plan or epic and return all unblocked `Pending` tasks in human-readable or `--json` format.
- **REQ-009**: When `trace-tags` is invoked for a specification or task, the `TraceTagGenerator` shall output canonical `// @verifies spec-ft-<slug>::<TASK-ID>::<REQ-ID>` and `// @implements spec-ft-<slug>::<TASK-ID>::<REQ-ID>` annotation lines.

### Unwanted Behavior & Error Handling
- **REQ-010**: If `task-status` is invoked with `Skipped/Canceled` without `--reason`, or `Rewritten` without both `--reason` and `--rewrite-target`, then the `TaskLifecycleMutator` shall reject the mutation and exit with a non-zero status code.
- **REQ-011**: If `task-status` transitions a task to `Done` with `--code-dir` provided (and without `--skip-gate`), then the `TaskCompletionGate` shall reject the transition if the task's `Verification Seam` file does not exist on disk or any `Addresses` requirement lacks `@verifies` or `@implements` annotations.
- **REQ-012**: If `frontier` or `verify` detects a circular dependency or reference to a non-existent blocker task ID in `Blocked by`, then the `DagFrontierResolver` shall report a DAG integrity error and exit with a non-zero status code.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Scaffolding & Template Seam (`codebase/aico-skills/scripts/aico-cli.mjs`)**:
   - Signature: `scaffoldEpic(opts)`, `scaffoldFeature(opts)`, `scaffoldAdr(opts)`, `upsertGlossary(opts)`
   - Seam Test: `../aico-skills/tests/aico-cli.test.mjs::test_scaffold_commands`
2. **Task Lifecycle & Roll-Up Seam (`codebase/aico-skills/scripts/aico-cli.mjs`)**:
   - Signature: `setTaskStatus(opts)`
   - Seam Test: `../aico-skills/tests/aico-cli.test.mjs::test_task_status_and_rollup`
3. **Plan Amendment & Frontier DAG Seam (`codebase/aico-skills/scripts/aico-cli.mjs`)**:
   - Signature: `amendPlan(opts)`, `computeFrontier(opts)`
   - Seam Test: `../aico-skills/tests/aico-cli.test.mjs::test_plan_amend_and_frontier`
4. **Verification & Trace Tags Seam (`codebase/aico-skills/scripts/aico-cli.mjs`)**:
   - Signature: `generateTraceTags(opts)`, `runVerify(opts)`
   - Seam Test: `../aico-skills/tests/aico-cli.test.mjs::test_verify_and_trace_tags`

---

## Out-of-Scope
- Direct mutations to embedded Kùzu or LanceDB binary database files from Node.js scripts (handled by `aico-mcp` and `aico-storage`).
- Automated LLM prose generation inside CLI scripts (scripts generate deterministic structure, metadata, and skeletons).
- Git commit or branch creation inside scaffolding commands.

---

## 5. Open Clarifications

None. Ratified in [ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md) and [ADR-0010](../../../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md).
