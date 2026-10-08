---
id: spec-ft-git-skills-traceability
type: spec
aico_skills_version: "0.1.2"
epic: "ec-git"
order: 2
title: "Plan Frontmatter Multi-Repo Traceability & Blueprint Standards"
status: "Completed"
derives_from: ["brain-ec-git"]
created_at: "2026-10-07T00:32:00Z"
---

# Feature Specification: Plan Frontmatter Multi-Repo Traceability & Blueprint Standards

## 1. Problem Statement & User Value

In multi-repository development environments, agents and developers need to know precisely which repositories are impacted by a given implementation plan. Specifications define domain behavior independent of physical storage, while plans dictate concrete technical deliverables.

Currently, legacy plans across `ec-specs` and `ec-core` lack the `repositories: [...]` frontmatter attribute, causing views such as `ec-specs` to display zero repositories. Furthermore, `verify-artifacts.mjs` only validates the attribute if already present (`if (fm.repositories !== undefined)`), allowing plans with missing repository attributes to silently pass verification.

This feature establishes `plan-ft-*.md` line-1 YAML frontmatter as the single source of truth for code repository targeting (`repositories: ["repo-id", ...]`), updates planning templates and skills, equips the artifact verification suite with strict mandatory validation for repository declarations, and backfills repository attributes across all project plans.

---

## In-Scope
- Adding `repositories: ["{repo_id}"]` to `PLAN-TEMPLATE.md`.
- Documenting the repository declaration conventions and dynamic roll-up rules in `aico-planning/SKILL.md`.
- Extending `verify-artifacts.mjs` to validate that implementation plans declare a valid `repositories` array.
- Enforcing `repositories` as a required frontmatter field in `verify-artifacts.mjs` for all `type: plan` artifacts (flagging omissions).
- Validating declared repository IDs against registered project repositories in `.aico-project.json` when available.
- Backfilling missing `repositories: [...]` frontmatter into legacy plans across `ec-specs` and `ec-core`.

---

## Out-of-Scope
- Modifying specification frontmatter (`spec-ft-*.md`), keeping specs repository-agnostic.
- Automated git checkouts or branch switching during verification.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an engineer or agent creating a feature plan, I want a designated `repositories` frontmatter field in `PLAN-TEMPLATE.md`, so that I can declare target repositories in a standardized format.
   - *Independent Test*: Inspect `PLAN-TEMPLATE.md` and confirm `repositories: ["{repo_id}"]` is present in YAML frontmatter.
2. **US-02** (Priority: P1): As an engineer reviewing plans, I want `verify-artifacts.mjs` to check that plans declare valid repository IDs and flag missing attributes, so that orphaned or missing repository mappings are caught during linting.
   - *Independent Test*: Run `verify-artifacts.mjs` against a plan with an invalid or missing repository field and verify it produces an informative warning/error.
3. **US-03** (Priority: P1): As a developer viewing `ec-specs` or `ec-core`, I want all child feature plans to declare their target repositories, so that the parent epic dynamically aggregates and displays all touched repositories.
   - *Independent Test*: Inspect `ec-specs` Epic Hub and confirm relevant repositories (`aico-specs`, `aico-skills`) appear under the title.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: The Planning skill template (`PLAN-TEMPLATE.md`) shall include a frontmatter array field `repositories` containing target repository identifiers.
- **REQ-002**: The Planning skill documentation (`aico-planning/SKILL.md`) shall specify that plan frontmatter is the sole source of truth for target repositories, rolling up dynamically to features and epics.
- **REQ-003**: The Artifact Verification engine (`verify-artifacts.mjs`) shall inspect `plan-ft-*.md` frontmatter and verify that `repositories` is declared as an array of string identifiers.
- **REQ-004**: Where a project manifest (`.aico-project.json`) exists, the Artifact Verification engine shall validate that all declared repository IDs in `repositories` correspond to known repository entries in the manifest.
- **REQ-005**: Where an implementation plan (`plan-ft-*.md`) omits the `repositories` frontmatter field or provides an empty array, the Artifact Verification engine shall emit a blueprint warning or error.
- **REQ-006**: All project implementation plans across `ec-specs`, `ec-core`, and `ec-git` shall declare valid repository identifiers matching registered manifest repositories.

---

## 4. Designated Public Test Seams

- **Planning Template Seam**: `d:/aico/codebase/aico-skills/aico-planning/PLAN-TEMPLATE.md`
- **Verification Engine Seam**: `d:/aico/codebase/aico-skills/aico-verify/scripts/verify-artifacts.mjs`
