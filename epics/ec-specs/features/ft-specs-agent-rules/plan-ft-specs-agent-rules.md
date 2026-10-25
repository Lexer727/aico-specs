---
id: plan-ft-specs-agent-rules
type: plan
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 5
title: "Workspace Agent Rules & Mandatory Human Approval Gate Implementation Plan"
status: "Completed"
repositories: ["aico"]
derives_from: ["spec-ft-specs-agent-rules"]
created_at: "2026-10-10T01:21:00Z"
---

# Implementation Plan: Workspace Agent Rules & Mandatory Human Approval Gate

## 1. Technical Architecture & Decisions

- **Version-Controlled Repository Rules (`aico-specs/GEMINI.md`)**:
  - Author a canonical `GEMINI.md` file at the root of `d:/aico/aico-specs`.
  - Embed the mandatory 4-phase review sequence: Investigate & Diagnose -> Present Proposals & Await Spec Sign-Off -> Write Specs/Plans & Halt for Implementation Confirmation -> Implement Only After Post-Spec Confirmation.
  - Require strict adherence to Blueprint v0.1.2 standards and relative file paths.
- **Root Workspace Discovery Mirror (`d:/aico/GEMINI.md`)**:
  - Mirror the identical `GEMINI.md` at the composite workspace root (`d:/aico/GEMINI.md`) so that ongoing Antigravity IDE and CLI sessions immediately discover and enforce the rules during bootstrap operations.
- **Blueprint Verification**:
  - Ensure all artifacts strictly conform to Blueprint v0.1.2 standards with `verify-artifacts.mjs`.
- **Interactive Review Gates via ask_question (`REQ-010`)**:
  - Modernize review gate halts (Phase 2 Spec Sign-Off and Phase 3 Implementation Confirmation) and Socratic trade-off questions by calling `ask_question` with structured options (e.g. `(Recommended) Approve and proceed`, architectural options, `Cancel`) and the default write-in field, replacing unstructured passive text halts.

---

## 2. Tracer-Bullet Task Graph

### Phase 1: Epic Registration & Scaffolding

- **TASK-RULES-001**: Register ft-specs-agent-rules in epics/ec-specs/brain-ec-specs.md
  - **Status**: Done
  - **Addresses**: REQ-001
  - **Blocked by**: None
  - **What it delivers**:
    - Adds child feature 5 to the overarching epic brainstorm decomposition.

- **TASK-RULES-002**: Scaffold spec-ft-specs-agent-rules.md and plan-ft-specs-agent-rules.md
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-003, REQ-004
  - **Blocked by**: TASK-RULES-001
  - **What it delivers**:
    - Creates formal EARS requirements and vertical tracer-bullet execution plan.

### Phase 2: Rules File Deployment

- **TASK-RULES-003**: Create canonical GEMINI.md in aico-specs and mirror to root workspace
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-005, REQ-006
  - **Blocked by**: TASK-RULES-002
  - **What it delivers**:
    - Deploys version-controlled rules file to `aico-specs` and active bootstrap workspace `d:/aico`.

### Phase 3: Verification & Parity

- **TASK-RULES-004**: Verify full suite artifacts with verify-artifacts.mjs and confirm host server discovery
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-006
  - **Blocked by**: TASK-RULES-003
  - **What it delivers**:
    - Audits entire repository for 100% structural and ground-truth parity.

### Phase 4: Plugin Harness Packaging & Distribution

- **TASK-RULES-005**: Package canonical rules/GEMINI.md, rules/AGENTS.md, and root GEMINI.md in aico-skills
  - **Status**: Done
  - **Addresses**: REQ-007
  - **Blocked by**: TASK-RULES-004
  - **What it delivers**:
    - Adds `rules/GEMINI.md`, `rules/AGENTS.md`, and root `GEMINI.md` to `d:/aico/codebase/aico-skills` so the plugin distributes rules automatically.

- **TASK-RULES-006**: Synchronize rules directory to global plugin deployment
  - **Status**: Done
  - **Addresses**: REQ-008
  - **Blocked by**: TASK-RULES-005
  - **What it delivers**:
    - Copies/links `rules/` into `C:\Users\astau\.gemini\config\plugins\aico-skills\rules\` for active machine runtime discovery.

- **TASK-RULES-007**: Commit aico-skills updates and verify dual-repo blueprint parity
  - **Status**: Done
  - **Addresses**: REQ-007, REQ-008
  - **Blocked by**: TASK-RULES-006
  - **What it delivers**:
    - Verifies all artifacts pass with `verify-artifacts.mjs` and commits distribution assets.

- **TASK-RULES-008**: Update GEMINI.md and AGENTS.md across all workspace, specs, and plugin locations to enforce post-spec-write confirmation before implementation
  - **Status**: Done
  - **Addresses**: REQ-009
  - **Blocked by**: TASK-RULES-007
  - **What it delivers**:
    - Updates `GEMINI.md` and `AGENTS.md` to codify the 4-phase workflow requiring the agent to halt after writing/updating specifications and plans and await explicit confirmation before modifying codebase or test files.

### Phase 5: Interactive Multi-Choice Gate & Clarification Standardization

- **TASK-RULES-009**: Codify Interactive Multi-Choice Review Gates & Clarifications via ask_question
  - **Status**: Done
  - **Addresses**: `REQ-010`
  - **Blocked by**: `TASK-RULES-008`
  - **What it delivers**:
    - Updates canonical `GEMINI.md` and `AGENTS.md` across workspace root (`d:/aico`), `aico-specs`, `.agents/rules`, and `codebase/aico-skills/rules` to require calling `ask_question` at Phase 2 and Phase 3 gates and for Socratic trade-off inquiries.
    - Updates AICO skill definitions (`aico-start`, `aico-feature`, `aico-epic`, `aico-brainstorm`, `aico-spec-creation`) to instruct agents to trigger `ask_question` at all human gates.
    - Verifies artifact conformance with `verify-artifacts.mjs`.
  - **Verification Seam**: `verify-artifacts.mjs` and gate audit.

---

## 3. Amendment History

- **2026-10-10T01:21:00Z**: Created plan for workspace agent rules and mandatory review gate under epic `ec-specs`.
- **2026-10-10T01:52:00Z**: Appended Phase 4 (TASK-RULES-005, TASK-RULES-006, TASK-RULES-007) to package `rules/GEMINI.md` and `rules/AGENTS.md` directly into the `aico-skills` plugin repository and synchronize with global plugin deployment per REQ-007 and REQ-008.
- **2026-10-12T17:50:00Z**: Added `TASK-RULES-008` (`REQ-009`) to enforce a mandatory confirmation gate after spec/plan writes before starting implementation.
- **2026-10-25T19:50:00Z**: Added `TASK-RULES-009` (`REQ-010`) to codify interactive multiple-choice review gates and Socratic clarifications using the native `ask_question` tool across rules and skills.
