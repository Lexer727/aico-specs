---
id: spec-ft-tooling-skills-integration
type: spec
aico_skills_version: "0.1.2"
epic: "ec-tooling"
order: 3
title: "AICO Skills Suite Integration with aico-cli.mjs & Repurposed MCP Tools"
status: "Completed"
derives_from: ["brain-ec-tooling", "spec-ft-tooling-sdd-cli", "spec-ft-tooling-mcp-repurpose"]
created_at: "2026-10-17T21:14:45Z"
---

# Feature Specification: AICO Skills Suite Integration with aico-cli.mjs & Repurposed MCP Tools

## 1. Problem Statement & User Value

Even with a unified `aico-cli.mjs` and repurposed `aico-mcp`, LLM agents will continue manually creating folders and hand-editing markdown metadata unless the `SKILL.md` instructions across `aico-skills` explicitly direct the agent to run `aico-cli.mjs` subcommands and `aico-mcp` query tools.

This specification updates the `aico-skills` manifests (`aico-epic`, `aico-feature`, `aico-spec-creation`, `aico-planning`, `aico-plan-amendment`, `aico-implementation`, `aico-domain-modeling`, `aico-verify`, `aico-brainstorm`, and `aico-deep-dive`) to invoke `aico-cli.mjs` and `aico-mcp`, and verifies that `SkillsSynchronizer` propagates the unified CLI into `.agents/skills/scripts/`.

---

## In-Scope
- Updating `SKILL.md` files in `codebase/aico-skills/` to mandate `node <skills-path>/scripts/aico-cli.mjs <subcommand>` for scaffolding, task status transitions, plan amendments, frontier resolution, trace-tag generation, and verification.
- Updating `aico-brainstorm` and `aico-deep-dive` `SKILL.md` files to reference `search_hybrid` and `query_lineage_graph` MCP tools alongside removes of obsolete `create_sdd_artifact` references in `skills-specification.md`.
- Verifying `SkillsSynchronizer::sync_workspace` copies `scripts/aico-cli.mjs` into `.agents/skills/scripts/aico-cli.mjs`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an agent executing `/aico-epic`, `/aico-feature`, `/aico-implementation`, or `/aico-plan-amendment`, I want clear CLI command invocations in `SKILL.md`, so that I use deterministic tooling instead of manual directory/status editing.
   - *Independent Test*: Audit `SKILL.md` files in `codebase/aico-skills` to verify each lifecycle skill contains the corresponding `aico-cli.mjs` subcommand invocation.
2. **US-02** (Priority: P1): As a developer running `aico-server`, I want `SkillsSynchronizer` to automatically sync `scripts/aico-cli.mjs` to `.agents/skills/scripts/aico-cli.mjs`, so that workspace agents always have the latest CLI tooling.
   - *Independent Test*: Run `SkillsSynchronizer::sync_from_source` and verify `.agents/skills/scripts/aico-cli.mjs` exists and matches source bytes.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The scaffolding and modeling skills (`aico-epic`, `aico-feature`, `aico-spec-creation`, `aico-planning`, `aico-domain-modeling`) shall instruct agents to execute `aico-cli.mjs` subcommands (`scaffold-epic`, `scaffold-feature`, `scaffold-adr`, `upsert-glossary`) rather than manually creating folders or frontmatter blocks.
- **REQ-002**: The execution and maintenance skills (`aico-implementation`, `aico-plan-amendment`, `aico-verify`) shall instruct agents to execute `aico-cli.mjs` subcommands (`task-status`, `plan-amend`, `frontier`, `trace-tags`, `verify`) for task transitions, DAG queries, and dual-axis verification.
- **REQ-003**: The exploration skills (`aico-brainstorm`, `aico-deep-dive`) and `skills-specification.md` shall reference the repurposed `aico-mcp` tools (`search_hybrid`, `query_lineage_graph`, `find_dangling_requirements`) and `aico-cli.mjs` rather than the removed `create_sdd_artifact` MCP tool.

### Event-Driven Requirements
- **REQ-004**: When `SkillsSynchronizer::sync_workspace` runs, the synchronizer shall copy `scripts/aico-cli.mjs` and updated `SKILL.md` files into `.agents/skills/`.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Skill Manifest CLI Reference Audit Seam (`codebase/aico-skills/tests/aico-cli.test.mjs`)**:
   - Signature: `test_skills_documentation_references_cli`
   - Seam Test: `../aico-skills/tests/aico-cli.test.mjs::test_skills_documentation_references_cli`
2. **Workspace Skills Synchronizer Seam (`crates/aico-core/src/skills_sync.rs`)**:
   - Signature: `SkillsSynchronizer::sync_from_source(source_skills_dir, root_dir) -> Result<SyncReport, SkillsSyncError>`
   - Seam Test: `crates/aico-core/tests/scanner_test.rs`

---

## Out-of-Scope
- Adding new slash-command skill directories beyond the existing `aico-skills` catalog.
- Modifying browser WebUI React components.

---

## 5. Open Clarifications

None. Ratified in [ADR-0009](../../../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md) and [ADR-0010](../../../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md).
