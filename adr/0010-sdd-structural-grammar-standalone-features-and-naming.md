---
id: adr-0010
type: adr
aico_version: "0.1.2"
title: "SDD Structural Grammar v0.1.2: Standalone Features, EC- Prefixes, Wave-Internal Tasks, and Frontmatter Ordering"
status: Completed
derives_from: ["ADR-0009"]
created_at: "2026-10-06T15:10:00Z"
---

# ADR-0010: SDD Structural Grammar v0.1.2: Standalone Features, EC- Prefixes, Wave-Internal Tasks, and Frontmatter Ordering

## Context & Problem Statement

Following the initial release of the AICO WebUI shell, several structural limitations in the SDD specification blueprints and workflow became evident:
1. **Mandatory Epic Overhead**: Every feature was forced into an epic container (`epics/<epic>/specs/<feature>`), even when a capability was small, self-contained, or an isolated utility. Developers lacked a first-class **Standalone Feature** workflow.
2. **Ambiguous Entry Point**: Starting a new idea lacked a triage front door to brainstorm and determine whether a request warranted a multi-feature Epic or a standalone Feature.
3. **Misaligned Wave Concept**: Epics were using "Waves" as separators between entire specifications, whereas an Epic contains Features, and individual vertical tasks inside a feature are what logically progress through execution waves (Scaffolding $\to$ Implementation $\to$ Seam Verification $\to$ Integration).
4. **Fragile Numeric Prefixes**: Directories (`001-aico-core`, `01-host-runtime`) and files (`spec-core-001.md`) encoded sequence numbers in filenames. Inserting or reordering features broke cross-document links and git history.
5. **Redundant Project Naming**: Repositories repeated the project name across every folder and file (e.g. `001-aico-core`, `002-aico-specs`), adding unnecessary noise.
6. **Underdeveloped Feature Overview**: While the Epic Hub provided a rich two-column view, the Feature Viewer was a skeletal placeholder lacking EARS requirements inspection, task progress, and seam traceability.

## Decision Drivers

- First-class support for both **Epic Initiatives** and **Standalone Features**.
- Clear triage entry point (`aico-start`) that conducts initial brainstorming and routes to `aico-epic` or `aico-feature`.
- Logical decomposition of tasks into internal execution waves (Scaffolding $\to$ Implementation $\to$ Verification).
- Stable, semantic naming: `EC-{short_id}` for epics, `{epic_id}-{feature_suffix}` for epic features, and `{feature_suffix}` for standalone features.
- Decouple display ordering from directory/filenames via a first-class `order: <number>` YAML frontmatter attribute.
- Full parity between Feature Hub and Epic Hub so standalone features have a complete, self-contained workspace.

## Considered Options

1. **Option A (Chosen)**:
   - Introduce **`aico-start`** as the universal front door to brainstorm and decide between Epic vs. Standalone Feature.
   - Introduce **`aico-feature`** for standalone feature creation, saving to top-level `<specs-root>/features/<feature-slug>/`.
   - Rename `epics/<epic>/specs/` to `epics/<epic>/features/` to accurately reflect domain contents.
   - Adopt `EC-{short_id}` prefix for epic directories (e.g. `epics/EC-core`, `epics/EC-specs`).
   - Remove numeric sequence prefixes from directory and file names. Use `order: <number>` in YAML frontmatter for display ordering.
   - Artifact naming standard:
     - Epic Brainstorm: `BRAIN-EC-{short_id}.md`
     - Epic Feature Spec/Plan: `SPEC-{epic_id}-{feature_suffix}.md` / `PLAN-{epic_id}-{feature_suffix}.md`
     - Standalone Feature Spec/Plan: `SPEC-{feature_suffix}.md` / `PLAN-{feature_suffix}.md`
   - Define **Execution Waves inside tasks**: Scaffolding (Red), Implementation (Green), Verification (Hardening), and Integration/Human Gate.
   - Overhaul `FeatureViewer.tsx` to display complete KPI metrics, EARS requirements breakdown, task execution waves, test seams, and full document readers.
   - Advance AICO specification version to `0.1.2`.
2. **Option B**: Keep all features strictly under epics, but create an artificial "misc" or "standalone" epic container. *(Rejected: Adds artificial nesting and clutters the epic graph).*
3. **Option C**: Keep numeric folder prefixes (`01-`, `02-`) and rename directories whenever ordering changes. *(Rejected: Fragile; causes broken markdown links and git churn).*

## Decision Outcome

Chosen option: **Option A**. The system decouples ordering from filesystem names, supports first-class standalone features, establishes `aico-start` as the triage entry point, and formalizes execution waves inside task DAGs.

### Consequences

- **Positive**:
  - Developers can build standalone features without epic bureaucracy.
  - Adding or reordering features never breaks relative markdown links or file paths.
  - Clear entry point with `aico-start` prevents architectural over-engineering or under-scaffolding.
  - Tasks declare verifiable execution waves with strict red-to-green contract enforcement.
  - The Feature Hub provides a rich, self-contained overview for every feature.
- **Negative / Migration Effort**:
  - Requires migrating existing `001-aico-core` and `002-aico-specs` directories, filenames, and links to the new `EC-` format.
  - Requires updating backend discovery parsers in `crates/aico-server` and frontend components in `frontend/src/`.
