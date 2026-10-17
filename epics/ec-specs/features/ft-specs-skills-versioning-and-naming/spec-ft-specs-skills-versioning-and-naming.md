---
id: spec-ft-specs-skills-versioning-and-naming
type: spec
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 4
title: "AICO Skills Version Decoupling & Artifact Terminology Standardization"
status: "Completed"
derives_from: ["ADR-0010", "brain-ec-specs"]
created_at: "2026-10-07T00:10:00Z"
---

# Feature Specification: AICO Skills Version Decoupling & Artifact Terminology Standardization

## 1. Problem Statement & User Value

Prior iterations of the specification tooling conflated the AICO platform runtime version (governed by the Cargo workspace at `v0.1.1`) with the AICO skills version. Additionally, templates contained hardcoded version numbers rather than dynamic placeholders, and version state was duplicated across both `plugin.json` (`"version": "0.1.2"`) and a standalone `aico-skills/VERSION` file.

This specification defines the strict decoupling of skills versioning from platform versioning, standardizes artifact nomenclature to AICO artifacts, establishes `plugin.json` as the sole canonical source of truth for the skills harness version, retires the redundant `VERSION` file, and aligns the WebUI Shell Settings display and synchronization engine.

---

## In-Scope
- Standardizing nomenclature from "SDD artifacts" to "AICO artifacts" across skills documentation, templates, and verification tooling.
- Single-source version resolution: reading the skills version dynamically from `plugin.json` across tooling (`get-version.mjs`, `verify-artifacts.mjs`, `SkillsSynchronizer`, and `ArtifactToolGate`).
- Retiring redundant `VERSION` file duplication in `aico-skills`.
- Synchronizing `plugin.json` into `<root_dir>/.agents/skills/plugin.json` so workspace tools can resolve the version without submodule dependencies.
- Removing hardcoded version numbers from skill headers, descriptions, and audit checklists.
- WebUI Shell Settings alignment: renaming "SDD Blueprint Standards" to "AICO Skills" and displaying "AICO Skills Version".
- Verification seam testing across the WebUI, MCP tool gate, and verification engine.

---

## Out-of-Scope
- Modifying the underlying 6-state task lifecycle state machine.
- Changing Kùzu or LanceDB database table schemas.
- Altering the platform product version schema (`v0.1.1` in `Cargo.toml`).

---

## 2. User Stories

1. **US-01** (Priority: P1): As an AI agent or developer authoring specifications, I want templates to use a dynamic `{aico_skills_version}` placeholder, so that version bumps in `plugin.json` automatically propagate without editing every template.
   - *Independent Test*: Inspect `SPEC-TEMPLATE.md`, `PLAN-TEMPLATE.md`, and `ADR-FORMAT.md` and verify `{aico_skills_version}` is present without hardcoded version numbers.
2. **US-02** (Priority: P1): As a developer reading skill documentation, I want consistent terminology referring to "AICO artifacts", so that documentation is cohesive and branded to the platform.
   - *Independent Test*: Run text searches across the `aico-skills` repository and confirm zero instances of deprecated "sdd artifacts".
3. **US-03** (Priority: P1): As a user inspecting system settings in the WebUI, I want the settings modal to display "AICO Skills" with "AICO Skills Version" alongside "Platform Version", so that I understand both versions are separate.
   - *Independent Test*: Render `SettingsModal` and verify both "AICO Skills" and "AICO Skills Version" appear in the DOM.
4. **US-04** (Priority: P2): As a developer maintaining `aico-skills`, I want the version defined solely in `plugin.json`, so that I do not have to maintain a separate redundant `VERSION` text file.
   - *Independent Test*: Run `node scripts/get-version.mjs` and verify it parses `plugin.json` and returns the expected version string.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: The SkillsHarness shall reference all specifications, plans, brainstorms, deep dives, and architecture decision records as AICO artifacts or AICO-skill artifacts across all documentation, templates, and verification scripts.
- **REQ-002**: When instantiating any artifact template or resolving the skills harness version, the SkillsHarness and associated tooling shall dynamically read the canonical skills version from `plugin.json` (with fallback to `VERSION`) and populate `{aico_skills_version}` in line-1 YAML frontmatter.
- **REQ-003**: While defining skills and authoring templates, the SkillsHarness shall omit hardcoded version numbers from markdown descriptions, section headers, checklists, and template bodies.
- **REQ-004**: The WebUIShell SettingsModal shall present the configuration section titled "AICO Skills" with the property "AICO Skills Version".
- **REQ-005**: The System shall keep the platform version (e.g. `v0.1.1` from Cargo workspace) strictly decoupled and visually separated from the skills version (e.g. `0.1.2` from `plugin.json`).
- **REQ-006**: Where an artifact template or specification introduces hardcoded legacy version numbers or refers to deprecated SDD artifact nomenclature, the ArtifactVerifier shall flag a blueprint lint warning.
- **REQ-007**: The `SkillsSynchronizer` and MCP runtime shall synchronize `plugin.json` into workspace `.agents/skills/` and resolve the active skills version from `plugin.json` across workspace installations.

---

## 4. Designated Public Test Seams

- **UI Seam**: `frontend/tests/SettingsModal.test.tsx` (`test_settings_modal_renders_aico_skills_section`)
- **MCP Gate Seam**: `crates/aico-mcp/tests/mcp_gate_test.rs` (`test_create_sdd_artifact_gate`)
- **Verification Script Seam**: `aico-verify/scripts/verify-artifacts.mjs` (Dual-axis verification audit)
- **Version Script Seam**: `aico-skills/scripts/get-version.mjs`
