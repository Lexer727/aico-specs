---
id: spec-specs-migration-engine
type: spec
aico_version: "0.1.2"
epic: "ec-specs"
order: 3
title: "AICO Specification Migration Skill & 0.1.1 Repository Upgrade"
status: Completed
derives_from: ["ADR-0009", "spec-specs-001", "spec-specs-002"]
created_at: "2026-10-06T01:45:00Z"
---

# Feature Specification: AICO Specification Migration Skill & 0.1.1 Repository Upgrade

## 1. Problem Statement & User Value

As the AICO specification format evolves, repositories containing existing specs and plans from earlier versions (such as `0.1.0`) must be updated without data loss, broken references, or manual editing errors.

This specification defines the new `aico-spec-migration` skill, which provides structured instructions and verification procedures to inspect any specifications repository, identify legacy syntax, inject canonical frontmatter, normalize relative links, eliminate duplicated metadata, and advance the version tag to `0.1.1`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an engineer or agent, I want to invoke `/aico-spec-migration`, so that I can automatically audit and migrate all existing artifacts in `aico-specs` to compliant 0.1.1 schemas.
   - *Independent Test*: Run the migration skill on `epics/001-aico-core` and verify that all specifications and plans conform to 0.1.1 standards with zero manual errors.
2. **US-02** (Priority: P1): As an engineer inspecting migrated plans, I want existing checked tasks (`- [x]`) converted to explicit `Status: Done` records, so that my historical execution record is preserved in the new task graph schema.
   - *Independent Test*: Inspect migrated `PLAN-CORE-*.md` files and confirm all tasks contain structured `Status: Done` fields.
3. **US-03** (Priority: P1): As a developer browsing documentation, I want all legacy `file:///` URLs converted to clean relative markdown links, so that documentation renders properly on all machines.
   - *Independent Test*: Verify that zero `file:///` URLs remain in any migrated artifact.

---

## 4. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The MigrationSkill shall provide step-by-step instructions and heuristics for migrating any legacy AICO specification repository to the active version.
- **REQ-002**: The MigrationService shall scan all `.md` files in `adr/` and `epics/` and verify the presence of YAML frontmatter blocks.
- **REQ-003**: The MigrationService shall replace machine-specific `file:///` links with clean relative paths.

### Event-Driven Requirements
- **REQ-004**: When migrating a legacy `SPEC-*.md` file, the MigrationService shall remove duplicate `ID`, `Status`, and `Derives From` fields located in the document body.
- **REQ-005**: When migrating a legacy `PLAN-*.md` file, the MigrationService shall inject YAML frontmatter with `type: plan` and update all tasks to include an explicit `Status` field.
- **REQ-006**: When migrating an existing completed epic (such as `001-aico-core`), the MigrationService shall update the frontmatter status of the epic brainstorm, child specifications, and plans to `Completed`.

---

## 5. Designated Public Test Seams

### Primary Seams:
1. **Migration Verification**:
   - Comprehensive audit verifying all files under `d:/aico/aico-specs` have YAML frontmatter, relative links, no duplicate metadata, and `aico_version: "0.1.1"`.
2. **Skill Registration**:
   - `C:\Users\astau\.gemini\config\plugins\aico-skills\skills\aico-spec-migration\SKILL.md` (via `d:\aico\codebase\aico-skills\aico-spec-migration\SKILL.md`).

---

## 6. Out of Scope

- Migrating code files (only SDD markdown artifacts are updated).
- Breaking change migrations across non-AICO markdown specifications.

---

## 7. Open Clarifications

None. Ratified in [ADR-0009](../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md).
