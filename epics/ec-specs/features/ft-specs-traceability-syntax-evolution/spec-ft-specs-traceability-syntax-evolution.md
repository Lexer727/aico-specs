---
id: spec-ft-specs-traceability-syntax-evolution
type: spec
aico_skills_version: "0.1.2"
epic: "ec-specs"
order: 6
title: "Traceability Syntax Evolution & Codebase In-Band Annotation Standards"
status: "Completed"
derives_from: ["brain-ec-specs", "spec-ft-specs-blueprint-standards", "spec-ft-specs-skills-evolution"]
created_at: "2026-10-17T17:00:00Z"
---

# Feature Specification: Traceability Syntax Evolution & Codebase In-Band Annotation Standards

## 1. Problem Statement & User Value

In Spec-Driven Development (SDD), codebase source files and test suites are linked to specifications and task plans via in-band comment annotations (`@implements` and `@verifies`). 

Two critical limitations currently impact traceability across the AICO platform:
1. **Ambiguous Legacy Syntax**: Annotations currently follow the legacy format `// @implements spec-ft-<slug>::REQ-<id>`. This syntax omits the execution plan task (`TASK-...`), decoupling source symbols from the tracer-bullet task Directed Acyclic Graph (DAG) and preventing granular progress tracking and task-to-code correlation.
2. **False Positives in Codebase Scanner & Navigation**: The backend codebase scanner (`scan_codebase_files`) uses an over-permissive substring search (`line.find("@implements")`), capturing documentation files (such as `skills/aico-feature/SKILL.md` or rule guidelines) that merely describe annotation grammar in prose or code blocks. In addition, the sidebar Code section and full-text code search show non-code artifacts and lack strict syntax validation.

This specification resolves these limitations by:
1. Standardizing a canonical 3-segment in-band annotation format: `// @implements ft-<slug>::TASK-<task_id>::REQ-<req_id>` and `// @verifies ft-<slug>::TASK-<task_id>::REQ-<req_id>`.
2. Establishing a reserved placeholder standard (`ft-<slug>::TASK-###::REQ-###`) for documentation, guides, and skills, ensuring documentation examples are structurally excluded from production annotation scans.
3. Upgrading `scan_codebase_files` in `aico-server` with strict comment-delimiter matching and regex filtering to include only real code files with non-zero valid annotations in the tree view (while preserving `.md` extension support for literate files).
4. Upgrading `verify-artifacts.mjs` to enforce Axis 2 requirement-to-code traceability against the new 3-segment syntax.
5. Providing an automated migration utility to cleanly upgrade existing legacy annotations across the codebase.

---

## In-Scope

- Defining the formal 3-segment in-band annotation syntax across programming languages (Rust, TypeScript/JavaScript, Python, CSS, JSX/TSX, Shell).
- Establishing reserved placeholder patterns (`###`, `<...>`) for skill documentation and examples.
- Upgrading `scan_codebase_files` in `aico-server` to require code comment prefixes and reject placeholders.
- Filtering the sidebar Code tree view so that only files with valid, non-zero `@implements` count are displayed.
- Upgrading `verify-artifacts.mjs` Axis 2 traceability check to validate `ft-<slug>::TASK-<task>::REQ-<req>` consistency.
- Automating the migration of all legacy annotations in `crates/` and `frontend/`.
- Updating skill files (`aico-implementation`, `aico-verify`, `aico-code-to-spec`, `aico-feature`, `aico-guide`) to teach the new syntax with placeholder examples.

---

## Out-of-Scope

- Modifying the underlying EARS requirement syntax within `spec-*.md` files.
- Altering the DAG dependency model or 6-state task lifecycle in `plan-*.md`.
- Introducing external AST compiler plugins or external linters beyond in-process scanning.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an engineer or agent reading source code, I want `@implements` annotations to identify the feature, the plan task, and the specific requirement, so that I can immediately trace why a function was written and which task delivered it.
   - *Independent Test*: Inspect source code symbols across `crates/` and `frontend/` and verify that each annotation contains `ft-...::TASK-...::REQ-...`.
2. **US-02** (Priority: P1): As a developer browsing the WebUI sidebar, I want the Code section to display only files with real implementation annotations, so that documentation and skill markdown files do not clutter the code list.
   - *Independent Test*: Verify that `GET /api/v1/code/files` and the sidebar Code tree omit markdown documentation containing placeholder `@implements` examples while retaining files with valid annotations.
3. **US-03** (Priority: P1): As a CI/CD pipeline or developer running `verify-artifacts.mjs`, I want automated validation that every active requirement is covered by at least one matching 3-segment `@implements` symbol in code, so that dark code and unimplemented requirements are flagged.
   - *Independent Test*: Run `verify-artifacts.mjs` and confirm 100% Axis 2 traceability verification.
4. **US-04** (Priority: P2): As an agent implementing features via `/aico-implementation`, I want skill guides to provide clear examples using `###` placeholders, so that I know the exact annotation structure without triggering false positives during project scans.
   - *Independent Test*: Inspect `SKILL.md` files in `aico-skills` and confirm all example annotations use `ft-<slug>::TASK-###::REQ-###`.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements

- **REQ-001**: The CodeAnnotationEngine shall define the canonical in-band implementation annotation syntax as:
  ```
  // @implements ft-<feature_slug>::TASK-<task_id>::REQ-<req_id>
  ```
  supporting comment syntax for Rust (`//`), TypeScript/JavaScript (`//`, `/* ... */`), JSX/TSX (`{/* ... */}`), Python/Shell (`#`), and CSS (`/* ... */`).
- **REQ-002**: The CodeAnnotationEngine shall define the canonical in-band verification annotation syntax as:
  ```
  // @verifies ft-<feature_slug>::TASK-<task_id>::REQ-<req_id>
  ```
- **REQ-003**: The DocumentationStandard shall require all documentation, skill markdown files, and template examples to format annotation examples using reserved placeholders:
  ```
  // @implements ft-<feature>::TASK-###::REQ-###
  ```
  prohibiting production annotations from containing `<`, `>`, or `#`.
- **REQ-004**: The CodebaseScanner shall scan source and markdown files across `codebase_path` and match annotations strictly using the comment-delimited regular expression:
  ```regex
  ^\s*(?:\/\/|#|\/\*|\{\/\*)\s*@(implements|verifies)\s+(ft-[a-z0-9\-]+)::(TASK-[A-Za-z0-9\-]+)::(REQ-[0-9]{3,4})
  ```
- **REQ-005**: The CodebaseScanner shall ignore any line containing literal `<...>` or `###` placeholders and shall exclude files with zero valid annotations from the sidebar tree Code section.
- **REQ-006**: The VerificationEngine (`verify-artifacts.mjs`) shall parse 3-segment annotations and cross-reference that:
  1. `ft-<feature_slug>` corresponds to a valid feature directory.
  2. `TASK-<task_id>` is declared in the feature's `plan-ft-*.md`.
  3. `REQ-<req_id>` is declared in the feature's `spec-ft-*.md` and listed under the task's `addresses:` field.
  4. Every active requirement in `spec-ft-*.md` has at least one matching `@implements` symbol in code.
- **REQ-007**: The MigrationEngine shall provide an automated utility that parses all feature plans (`plan-ft-*.md`), resolves `(feature_id, req_id) -> task_id`, and migrates legacy 2-segment annotations (`spec-ft-<slug>::REQ-<id>`) into canonical 3-segment annotations.
- **REQ-008**: The CodeViewerComponent shall parse 3-segment annotations in streamed source files, highlighting the annotation line and rendering badge metadata for the associated task and requirement.

---

## 4. Designated Public Test Seams

1. **SEAM-001**: `crates/aico-server/tests/code_scanner_test.rs`
   - Verifies scanner parsing of 3-segment annotations, omission of placeholder documentation, and inclusion of valid markdown files.
2. **SEAM-002**: `aico-verify/scripts/verify-artifacts.mjs`
   - Verifies Axis 2 requirement-to-code traceability cross-referencing against specifications and plans.
3. **SEAM-003**: `frontend/tests/CodeViewer.test.tsx`
   - Verifies 3-segment annotation syntax highlighting in the full-file viewer.
4. **SEAM-004**: `frontend/tests/ShellLayoutSearch.test.tsx`
   - Verifies tree view rendering of code files with valid 3-segment annotation count badges.
