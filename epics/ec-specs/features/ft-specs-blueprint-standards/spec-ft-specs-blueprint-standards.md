---
id: spec-ft-specs-blueprint-standards
type: spec
aico_version: "0.1.2"
epic: "ec-specs"
order: 1
title: "SDD Blueprint Standards, Relative Linking & Task Lifecycle"
status: Completed
derives_from: ["ADR-0009", "brain-002-aico-specs"]
created_at: "2026-10-06T01:43:00Z"
---

# Feature Specification: SDD Blueprint Standards, Relative Linking & Task Lifecycle

## 1. Problem Statement & User Value

Engineering teams building software with AI agents rely on specifications, plans, and architectural decisions as their source of truth. If metadata is duplicated, paths are machine-locked, or status transitions are ambiguous, agents misunderstand task readiness and developers suffer from documentation drift.

This specification formalizes the canonical structural grammar for all AICO Spec-Driven Development artifacts, mandating frontmatter integrity, relative linking, single-source metadata, and an auditable task lifecycle.

---

## In-Scope
- Canonical YAML frontmatter specifications with decoupled ordering for all SDD artifacts.
- Six-state task lifecycle validation (Pending, Ready, In Progress, Blocked, Done, Skipped/Canceled).
- Linting against absolute `file:///` URLs and unmeasurable requirements adjectives.
- Automatic rejection of malformed plans and specifications at MCP tool gates.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an engineer or agent, I want every SDD artifact to start with a standardized YAML frontmatter block, so that tooling and parsers can extract provenance, status, and version deterministically.
   - *Independent Test*: Parse any markdown artifact in the repository and verify the presence of all required frontmatter attributes.
2. **US-02** (Priority: P1): As a developer opening specifications in a browser or IDE on any workstation, I want all cross-document links to be relative paths, so that no broken `file:///` URLs appear.
   - *Independent Test*: Run a link linter across all markdown artifacts and confirm zero instances of absolute file system schemas (`file:///`).
3. **US-03** (Priority: P1): As an orchestrating agent, I want to distinguish between `Draft`, `Pending`, `In Progress`, and `Completed` artifacts, so that I only implement plans that are ready and only consider epics done when all child specs are completed.
   - *Independent Test*: Inspect artifact status transitions across the creation, execution, and verification phases.
4. **US-04** (Priority: P1): As an engineer, I want plan tasks to track explicit status and never delete canceled or rewritten tasks, so that our team retains a full audit trail of architectural decisions and their rationales.
   - *Independent Test*: Verify that rewritten or canceled tasks include explicit `Reason` and `Rewrite Target` fields.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The ArtifactParser shall require a YAML frontmatter block containing `id`, `type`, `aico_version`, `title`, `status`, `derives_from`, and `created_at` at the beginning of every SDD artifact.
- **REQ-002**: The ArtifactValidator shall enforce that the markdown body of an SDD artifact does not duplicate metadata fields present in the YAML frontmatter.
- **REQ-003**: The ArtifactLinker shall format all internal markdown cross-references using relative file system paths without `file:///` schemes.
- **REQ-004**: The PlanParser shall require an explicit `Status` field on every task node in a plan task graph.

### State-Driven Requirements
- **REQ-005**: While an artifact is being drafted, the DocumentManager shall maintain its status as `Draft`.
- **REQ-006**: While an artifact has been reviewed and ratified for implementation, the DocumentManager shall maintain its status as `Pending`.
- **REQ-007**: While tasks within an implementation plan are actively executing, the DocumentManager shall maintain the artifact status as `In Progress`.
- **REQ-008**: While all verification tests associated with a specification pass green, the DocumentManager shall maintain the artifact status as `Completed`.

### Event-Driven Requirements
- **REQ-009**: When a task in an implementation plan is canceled or rewritten, the PlanManager shall preserve the original task and append a mandatory `Reason` field and `Rewrite Target` reference.
- **REQ-010**: When an implementation plan task is unblocked and ready for execution, the PlanManager shall transition the task status to `Pending` or `In Progress`.

### Unwanted Behavior & Error Handling
- **REQ-011**: If an SDD artifact contains absolute `file:///` paths, then the ArtifactValidator shall emit a lint violation identifying the line and file.
- **REQ-012**: If a task has status `Skipped/Canceled` or `Rewritten` and lacks a `Reason` field, then the PlanValidator shall reject the plan artifact as malformed.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Artifact Parser & Validator (`aico_core::parser::ArtifactValidator`)**:
   - Signature: `validate_artifact_content(content: &str, path: &Path) -> Result<ArtifactMeta, Vec<ValidationError>>`
   - Validates frontmatter presence, absence of duplicate body metadata, relative links, and task status blocks.
2. **MCP Artifact Tool Gate (`aico_mcp::gate::ArtifactToolGate`)**:
   - Injects canonical frontmatter on creation and rejects custom non-relative links.

---

## Out-of-Scope

- Automated git branch creation per task (handled by developer workflows).
- Real-time WYSIWYG rich text document collaboration.
- Non-markdown artifact formats (all SDD contracts are Markdown + YAML frontmatter).

---

## 6. Open Clarifications

None. Ratified in [ADR-0009](../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md).
