---
id: spec-ft-specs-parallel-subagent-execution
type: spec
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 7
title: "Parallel Subagent Execution & Atomic CLI Plan Locking"
status: "Completed"
derives_from: ["brain-ec-specs", "ADR-0009", "ADR-0010"]
created_at: "2026-10-23T19:05:21.686Z"
---

# Feature Specification: Parallel Subagent Execution & Atomic CLI Plan Locking

## 1. Problem Statement & User Value

Antigravity provides native `invoke_subagent` support (`TypeName: "self"`, `Workspace: "inherit"` or `"branch"`) for parallel agent execution, and AICO already models features and tasks as a Directed Acyclic Graph (DAG) with explicit `Blocked by` dependencies and `frontier-status` calculation. However:
1. AICO's orchestration skills (`aico-epic`, `aico-feature`, `aico-implementation`, `aico-planning`) currently prescribe strictly sequential single-agent execution (`Pick one task from the Ready Frontier -> Implement -> Verify -> Advance`), preventing developers from launching parallel subagents when multiple features or tasks are simultaneously unblocked.
2. `aico-cli.mjs task-status` performs an unlocked read-modify-write on `plan-*.md` files; if two parallel subagents finish tasks in the same plan simultaneously, one subagent's write can overwrite the other's status update.

---

## In-Scope
- Updating `aico-cli.mjs` (`task-status` and `frontier-status`) with atomic file locking (`.aico.lock` retry loop) and parallel-safe frontier metadata.
- Updating `aico-epic`, `aico-feature`, `aico-implementation`, and `aico-planning` skills in `codebase/aico-skills` (and synced `.agents/skills`) to detect when 2 or more Ready Frontier features/tasks with disjoint file seams are available, offer the developer a **Sequential vs. Parallel Subagents** execution choice with **per-task subagent model selection** (defaulting to the top normal Gemini model, `flash` / Gemini 3.8 Flash, with optional override to `inherit` / `pro` such as Gemini 4 Argon High for complex tasks), and dispatch parallel `invoke_subagent` workers after Phase 3 approval.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer executing an epic or feature with multiple unblocked tasks or child features, I want the agent to offer parallel subagent execution via `invoke_subagent` when file seams are disjoint and let me choose which model each task uses (or default to the top normal Gemini model, Gemini 3.8 Flash), so that complex architectural tasks can use frontier reasoning models (such as Gemini 4 Argon High) while standard tasks run in parallel on fast models.
   - *Independent Test*: Verify `aico-epic`, `aico-feature`, `aico-implementation`, and `aico-planning` skills define the parallel subagent dispatch protocol, seam-disjointness check, and per-task `Model` routing (`flash` default vs `inherit`/`pro` override).
2. **US-02** (Priority: P1): As a developer running concurrent subagents that update `plan-*.md` task statuses via `aico-cli.mjs task-status`, I want atomic lockfile synchronization around plan file mutations, so that concurrent updates never clobber each other.
   - *Independent Test*: Spawn concurrent `task-status` updates against the same `plan-*.md` file and verify all task status mutations are preserved without data loss.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The system shall acquire an exclusive per-plan lockfile (`<plan-path>.lock`) using atomic `wx` file creation with backoff retry before reading and writing `plan-*.md` in `aico-cli.mjs task-status`, and release the lockfile upon completion.
- **REQ-002**: The system shall document the Parallel Subagent Execution protocol (`invoke_subagent`) and per-task model routing across `aico-epic/SKILL.md`, `aico-feature/SKILL.md`, `aico-implementation/SKILL.md`, and `aico-planning/SKILL.md`.

### Event-Driven Requirements
- **REQ-003**: WHEN `aico-cli.mjs frontier-status` reports two or more `ready_frontier` tasks or unblocked child features with disjoint verification seams after Phase 3 confirmation, the agent workflow shall prompt the user for any per-task model overrides (defaulting to `Model: "flash"` for the top normal Gemini 3.8 Flash model, and supporting `Model: "inherit"` or `Model: "pro"` for high-complexity tasks such as Gemini 4 Argon High) and dispatch parallel subagents (`invoke_subagent` with `TypeName: "self"`).

### Unwanted Behavior / Edge-Case Requirements
- **REQ-004**: IF two unblocked tasks or features modify overlapping source files or verification seams, THEN the orchestration workflow shall serialize their execution rather than running them concurrently in the same workspace.
- **REQ-005**: IF a stale `<plan-path>.lock` file is older than 10 seconds during `aico-cli.mjs task-status`, THEN the system shall evict the stale lockfile and proceed with acquiring the lock.
- **REQ-006**: IF the user does not specify a custom model assignment for a subagent task, THEN the orchestration workflow shall default that subagent's `invoke_subagent` call to `Model: "flash"` (the top normal Gemini model, currently Gemini 3.8 Flash High) and append the explicit version, model family, and thinking effort tag (`(3.8 Flash High)` or `(4 Argon High)`) to the subagent's `Role` parameter.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Atomic CLI Plan Mutation Seam (`codebase/aico-skills/scripts/aico-cli.mjs`)**:
   - Signature: `node aico-cli.mjs task-status <plan-file> <task-id> <new-status>`
   - Seam Test: `codebase/aico-skills/tests/aico-cli.test.mjs`
2. **Skill Conformance Verification Seam (`codebase/aico-skills/aico-verify/scripts/verify-artifacts.mjs`)**:
   - Signature: `node verify-artifacts.mjs <specs-dir>`
   - Seam Test: `codebase/aico-skills/tests/parallel-subagents-skill.test.mjs`

---

## Out-of-Scope
- Modifying Antigravity's built-in `invoke_subagent` runtime implementation.

---

## 5. Open Clarifications

None. Ratified in [ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md) and [ADR-0010](../../../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md).
