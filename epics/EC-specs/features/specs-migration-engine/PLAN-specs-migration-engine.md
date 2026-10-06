---
id: plan-specs-migration-engine
type: plan
aico_version: "0.1.2"
epic: "ec-specs"
order: 3
title: "AICO Specification Migration Skill & 0.1.1 Repository Upgrade"
status: Completed
derives_from: ["spec-specs-migration-engine"]
created_at: "2026-10-06T01:45:00Z"
---

# Implementation Plan: AICO Specification Migration Skill & 0.1.1 Repository Upgrade

## 1. Technical Architecture & Decisions

- Author new `aico-spec-migration` skill in `d:/aico/codebase/aico-skills/aico-spec-migration/`.
- Execute migration script/process across all legacy artifacts in `d:/aico/aico-specs/`:
  - `adr/*.md` (ADR-0001 through ADR-0008)
  - `epics/001-aico-core/brain-001-aico-core.md`
  - `epics/001-aico-core/specs/*/*.md` (SPEC-CORE-001..007, PLAN-CORE-001..007)
- Validate complete conversion: 0 absolute links, 100% frontmatter coverage, 100% task status formatting, version set to `0.1.1`.

---

## 2. Task Graph (DAG)

### Phase 1: Migration Skill Creation

- **TASK-SPECS-003-1**: Author `aico-spec-migration` Skill
  - **Status**: Done
  - **Addresses**: REQ-001
  - **Blocked by**: None
  - **What it delivers**:
    - Creates `SKILL.md` with migration procedures, heuristics, and verification checks.

---

### Phase 2: ADR Migration

- **TASK-SPECS-003-2**: Migrate `ADR-0001` through `ADR-0008` to Version 0.1.1
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-003
  - **Blocked by**: TASK-SPECS-003-1
  - **What it delivers**:
    - Adds YAML frontmatter to all 8 ADRs with `type: adr`, `aico_version: "0.1.1"`, and `status: Completed`.
    - Converts any absolute links to relative paths.

---

### Phase 3: Epic 001 Specifications & Plans Migration

- **TASK-SPECS-003-3**: Migrate `brain-001-aico-core.md` to Version 0.1.1
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-006
  - **Blocked by**: TASK-SPECS-003-1
  - **What it delivers**:
    - Injects YAML frontmatter and sets status to `Completed`.

- **TASK-SPECS-003-4**: Migrate `SPEC-CORE-001` through `SPEC-CORE-007` to Version 0.1.1
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-003, REQ-004, REQ-006
  - **Blocked by**: TASK-SPECS-003-1
  - **What it delivers**:
    - Removes duplicated `ID`, `Status`, `Derives From` from body.
    - Sets `aico_version: "0.1.1"` and `status: Completed` in frontmatter.
    - Replaces all `file:///` URLs with relative markdown links.

- **TASK-SPECS-003-5**: Migrate `PLAN-CORE-001` through `PLAN-CORE-007` to Version 0.1.1
  - **Status**: Done
  - **Addresses**: REQ-002, REQ-003, REQ-005, REQ-006
  - **Blocked by**: TASK-SPECS-003-4
  - **What it delivers**:
    - Injects canonical YAML frontmatter (`type: plan`, `aico_version: "0.1.1"`, `status: Completed`).
    - Removes duplicated body header fields.
    - Upgrades all tasks from `- [x]` to structured `- **Status**: Done` fields.
    - Replaces all `file:///` URLs with relative markdown links.

---

## 3. Initial Active Frontier

- None remaining (All tasks completed and verified).
