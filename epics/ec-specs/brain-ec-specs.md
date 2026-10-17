---
id: brain-ec-specs
type: brainstorm
aico_skills_version: "0.1.2"
epic: "ec-specs"
title: "AICO Specification & Plan Blueprints Evolution"
status: "Completed"
derives_from: ["ADR-0009"]
created_at: "2026-10-06T01:42:00Z"
---

# Brainstorm: AICO Specification & Plan Blueprints Evolution

## 1. Executive Summary & Problem Statement

As the AICO platform transitions from initial bootstrapping (Epic 001) to scalable multi-feature execution, the blueprint templates and conventions governing Spec-Driven Development (SDD) require standardizing and hardening.

Specific limitations identified in Epic 001:
1. **Missing Frontmatter in Plans, Brainstorms, and ADRs**: Plans lacked structured YAML frontmatter, making them difficult for automated tooling to query and catalog.
2. **Duplicated Metadata**: Attributes like `ID`, `Status`, and `Derives From` were mirrored across both the YAML frontmatter and the top section of the markdown body, inviting inconsistency.
3. **Static Status Values**: Documents remained in `Approved` or `Ready` state indefinitely, failing to reflect active execution or verified completion.
4. **Environment-Coupled Absolute Paths**: Markdown links contained absolute file system URIs (`file:///d:/...`), breaking portability across workstations and git remotes.
5. **Lossy Task Modification**: Task progress was marked by simple `- [x]` checkboxes, with no structured audit trail when tasks were canceled or rewritten.
6. **Lack of Automated Migration Tooling**: No dedicated skill existed to safely bump specifications from version `0.1.0` to `0.1.1` and transform legacy formats into compliant schemas.

This epic establishes the upgraded SDD Blueprint Standard, updates the core `aico-skills` suite, introduces a new `aico-spec-migration` skill, and migrates all existing Epic 001 artifacts to AICO specification version `0.1.1`.

---

## 2. Core Enhancements & Decisions

### 2.1 Universal YAML Frontmatter Schema
Every SDD artifact (`SPEC`, `PLAN`, `BRAIN`, `DIVE`, `ADR`) must have a canonical YAML frontmatter block at line 1:
```yaml
---
id: <ARTIFACT-ID>
type: <spec|plan|brainstorm|deep_dive|adr>
aico_version: "0.1.1"
epic: "<epic-slug>" # Optional for ADRs
title: "<Human Readable Title>"
status: <Draft|Pending|In Progress|Completed>
derives_from: ["<Pointers to upstream IDs>"]
created_at: "<ISO-8601 Timestamp>"
---
```

### 2.2 Relative File Path Standard
All markdown links pointing to other specifications, ADRs, plans, or code seams must use relative paths (e.g. `../../adr/0001-host-runtime-and-network-topology.md`), never machine-specific `file:///` URLs.

### 2.3 4-State Artifact Lifecycle
Standardize artifact-level status across all SDD documents:
- `Draft`: Initial drafting or exploratory brainstorming.
- `Pending`: Reviewed and approved, ready to begin implementation.
- `In Progress`: Actively being implemented or tested by agents.
- `Completed`: All requirements verified with passing green seam tests.

### 2.4 Single Source of Truth (No Duplicated Body Metadata)
The body of the artifact must not replicate fields already defined in the frontmatter (`ID`, `Status`, `Derives From`, `Title`). The document body begins directly with the top-level `# <Title>` heading, followed by domain-specific content sections.

### 2.5 6-State Task Tracking with Audit Trails
Implementation plan tasks replace simple checkboxes with structured status records:
- **Status Options**: `Pending`, `In Progress`, `Done`, `Blocked`, `Skipped/Canceled`, `Rewritten`.
- **Audit Requirement**: If a task transitions to `Skipped/Canceled` or `Rewritten`, it must specify:
  - `- **Reason**: <Rationale explaining why the change occurred>`
  - `- **Rewrite Target**: <Link or ID of the replacement task or document>`
- Historical tasks are never silently overwritten or discarded.

### 2.6 Automated Migration Engine & Version Tagging
A new skill, `aico-spec-migration`, systematically audits and updates existing artifacts to version `0.1.1`, converting absolute paths to relative links, injecting frontmatter where absent, removing redundant body metadata, and upgrading task tracking blocks.

---

## 3. Epic Decomposition (Composite Multi-Spec Structure)

This epic decomposes into 4 child specifications:

1. **`ft-specs-blueprint-standards`** (`spec-ft-specs-blueprint-standards`, `plan-ft-specs-blueprint-standards`):
   - Formal EARS requirements for the universal frontmatter schema, relative links, 4-state artifact lifecycle, and 6-state task tracking.
2. **`ft-specs-skills-evolution`** (`spec-ft-specs-skills-evolution`, `plan-ft-specs-skills-evolution`):
   - Updates to `aico-skills` (`aico-spec-creation`, `aico-planning`, `aico-epic`, `aico-domain-modeling`, `aico-implementation`) and `aico-mcp` to enforce the new blueprints.
3. **`ft-specs-migration-engine`** (`spec-ft-specs-migration-engine`, `plan-ft-specs-migration-engine`):
   - Specification and implementation of the `aico-spec-migration` skill, plus migration of all existing `ec-core` and ADR artifacts.
4. **`ft-specs-skills-versioning-and-naming`** (`spec-ft-specs-skills-versioning-and-naming`, `plan-ft-specs-skills-versioning-and-naming`):
   - Decoupling of AICO skills versioning (`aico-skills/VERSION`) from the platform version, standardization of artifact terminology to AICO artifacts, dynamic template placeholders (`{aico_skills_version}`), and WebUI Shell Settings alignment.
5. **`ft-specs-agent-rules`** (`spec-ft-specs-agent-rules`, `plan-ft-specs-agent-rules`):
   - Formal specification of workspace agent rules (`GEMINI.md` / `AGENTS.md`) and enforcement of the mandatory human review and approval gate before any code or specification modifications.
6. **`ft-specs-traceability-syntax-evolution`** (`spec-ft-specs-traceability-syntax-evolution`, `plan-ft-specs-traceability-syntax-evolution`):
   - Canonical 3-segment in-band annotation standard (`ft-<slug>::TASK-<task>::REQ-<req>`), skill placeholder conventions, strict comment scanning, and automated codebase migration.
