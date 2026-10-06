---
id: adr-0009
type: adr
aico_skills_version: "0.1.2"
title: "SDD Artifact Schema, Relative Links, and Task Lifecycle Management"
status: "Completed"
derives_from: []
created_at: "2026-10-06T01:40:00Z"
---

# ADR-0009: SDD Artifact Schema, Relative Links, and Task Lifecycle Management

## Context & Problem Statement
During Epic 001 execution, several friction points and inconsistencies emerged across SDD artifacts:
1. **Inconsistent Frontmatter**: Specifications contained YAML frontmatter, but implementation plans (`PLAN-*.md`), brainstorms (`BRAIN-*.md`), and architecture decision records (`ADR-*.md`) lacked standardized frontmatter headers.
2. **Duplicated Metadata**: Attributes like `ID`, `Status`, and `Derives From` were present in both YAML frontmatter and markdown body sections, creating maintenance hazards and synchronization drift.
3. **Rigid / Static Status Values**: Artifact statuses remained statically set to `Approved` or `Ready` even after implementation and verification were complete. A lifecycle tracking `Draft` $\to$ `Pending` $\to$ `In Progress` $\to$ `Completed` was needed.
4. **Environment-Dependent Absolute Links**: Artifact cross-references used machine-specific `file:///d:/...` URI links, breaking portability across developers, git repositories, and web viewers.
5. **Lossy Task Modification**: Task progress was tracked via markdown checkboxes (`- [x]`), with no audit trail when a task was skipped, canceled, or rewritten due to architectural evolution.

## Decision Drivers
- Universal machine-parsability for all SDD artifacts by AICO parser engines and MCP gates.
- Single source of truth for metadata: eliminate duplication between frontmatter and document body.
- Portable relative linking across repositories and environments.
- Explicit non-destructive task mutation history: tasks are never silently removed or rewritten without an audit reason and rewrite target pointer.

## Considered Options
1. **Option A (Chosen)**:
   - Mandatory standardized YAML frontmatter for all SDD artifacts (`SPEC`, `PLAN`, `BRAIN`, `DIVE`, `ADR`).
   - Single source of truth: body contains only content, frontmatter contains all metadata.
   - Standard 4-state artifact lifecycle: `Draft`, `Pending`, `In Progress`, `Completed`.
   - Strict relative markdown links (`../../adr/...`).
   - Standard 6-state task tracking: `Pending`, `In Progress`, `Done`, `Blocked`, `Skipped/Canceled`, `Rewritten` with mandatory `Reason` and `Rewrite Target` fields for canceled/rewritten items.
   - Increment AICO specification version to `0.1.1`.
2. **Option B**: Keep checkboxes for tasks and add separate change-log tables at the bottom of plan documents.
3. **Option C**: Store task tracking exclusively in embedded Kùzu database without updating markdown plan files.

## Decision Outcome
Chosen option: **Option A**. Every SDD artifact is self-describing, portable, and versioned. The task graph preserves historical intent by recording reasons and rewrite pointers directly on tasks when requirements change.

### Positive Consequences
- Tools and agents can parse frontmatter deterministically across all artifact types.
- Specs and plans can be viewed portably on any workstation, operating system, or git web UI without broken `file:///` URLs.
- Clear distinction between active tasks, completed tasks, and rewritten/canceled tasks with full audit rationale.
- Clean separation of concerns between frontmatter metadata and document body.
