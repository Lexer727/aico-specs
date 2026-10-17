---
id: plan-ft-tooling-skills-integration
type: plan
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 3
title: "AICO Skills Suite Integration with aico-cli.mjs & Repurposed MCP Tools Implementation Plan"
status: "Completed"
repositories: ["aico-skills", "aico", "aico-specs"]
derives_from: ["spec-ft-tooling-skills-integration"]
created_at: "2026-10-17T21:14:45Z"
---

# Implementation Plan: AICO Skills Suite Integration with aico-cli.mjs & Repurposed MCP Tools

## 1. Technical Architecture & Decisions

- **Skills Harness Updates**: Update `SKILL.md` files in `codebase/aico-skills/` (and sync to `codebase/aico/skills` / `.agents/skills/`) to replace manual folder/frontmatter/status instructions with `node <skills-path>/scripts/aico-cli.mjs <subcommand>`.
- **Synchronization**: Use `SkillsSynchronizer` in `aico-core` to ensure `.agents/skills/scripts/aico-cli.mjs` and all updated `SKILL.md` files stay in lockstep.

---

## 2. Task Graph (DAG)

### Phase 1: Skill Manifests Upgrade & Workspace Synchronization

- **TASK-TOOL-003-1**: Update `SKILL.md` Workflows for `aico-cli.mjs` & `aico-mcp` and Verify Workspace Sync
  - **Status**: Done
  - **Addresses**: `REQ-001`, `REQ-002`, `REQ-003`, `REQ-004`
  - **Blocked by**: `TASK-TOOL-001-3`, `TASK-TOOL-001-4`, `TASK-TOOL-002-2`
  - **What it delivers**:
    - Updates `aico-epic`, `aico-feature`, `aico-spec-creation`, `aico-planning`, `aico-plan-amendment`, `aico-implementation`, `aico-domain-modeling`, `aico-verify`, `aico-brainstorm`, `aico-deep-dive`, and `skills-specification.md` in `codebase/aico-skills/`.
    - Verifies that `SkillsSynchronizer` syncs `scripts/aico-cli.mjs` and updated skills to `.agents/skills/`.
  - **Verification Seam**: `../aico-skills/tests/aico-cli.test.mjs::test_skills_documentation_references_cli`
