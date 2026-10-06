---
id: spec-ft-fast-trace
type: spec
aico_version: "0.1.2"
epic: null
order: 1
title: "Fast Traceability Validator"
status: "Completed"
derives_from: ["../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md"]
created_at: "2026-10-06T15:30:00Z"
---

# Feature Specification: Fast Traceability Validator

## 1. Problem Statement & User Value

Developers working in Spec-Driven Development (SDD) require instantaneous feedback on whether `@implements` and `@verifies` code comment annotations in source files correctly reference valid requirement identifiers across specifications. Running the entire compilation and test suite introduces prohibitive latency in tight development inner loops.

This specification defines the Fast Traceability Validator: an in-process, single-pass scanner that extracts annotations, verifies them against active project specifications, and emits compiler-style diagnostic warnings for broken or orphan references in under 15 milliseconds.

---

## In-Scope
- Single-pass regex and Tree-sitter parsing of source files for `@implements` and `@verifies` tags.
- Cross-referencing against declared requirement identifiers in active project specifications.
- Reporting unresolved, typoed, or orphan contract references with exact file paths and line numbers.
- In-process execution within the local runtime without requiring full test suite invocation.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer modifying code, I want instantaneous feedback on whether my `@implements` and `@verifies` tags match valid requirement IDs, so that I can maintain contract traceability without running heavy test suites.
   - *Independent Test*: Run `scan_workspace_annotations` against a project directory and receive parsed annotations in under 15ms.
2. **US-02** (Priority: P1): As an engineer reviewing changes, I want the system to cross-reference code annotations against parsed specification requirements, so that satisfied and pending requirements are accurately attributed.
   - *Independent Test*: Verify that annotated symbols match active requirement entries in the specification catalog.
3. **US-03** (Priority: P2): As an agent executing implementation tasks, I want clear diagnostic warnings for missing, typoed, or orphan requirement IDs, so that I immediately know which references must be corrected.
   - *Independent Test*: Introduce a typo in a requirement annotation and verify that a structured diagnostic warning is emitted with exact file path and line number.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The TraceabilityScanner shall scan all workspace source files for `@implements` and `@verifies` annotations upon request in under 15 milliseconds.

### Event-Driven Requirements
- **REQ-002**: When a source annotation is encountered, the SpecificationCatalog shall cross-reference the requirement ID against parsed specifications in the active project.

### Unwanted Behavior & Error Handling
- **REQ-003**: If an annotation references a non-existent specification or requirement ID, then the DiagnosticReporter shall emit an unresolved diagnostic warning with the exact source path and line number.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Workspace Annotation Scanner (`crates/aico-core/src/trace.rs`)**:
   - Signature: `pub fn scan_workspace_annotations(root_path: &Path) -> Vec<SourceAnnotation>`
   - Seam Test: `crates/aico-core/tests/trace_test.rs::test_scan_workspace_annotations`
2. **Specification Catalog Reference Validator (`crates/aico-core/src/trace.rs`)**:
   - Signature: `pub fn validate_requirement_references(annotations: &[SourceAnnotation], specs: &[Specification]) -> TraceReport`
   - Seam Test: `crates/aico-core/tests/trace_test.rs::test_validate_requirement_references`
3. **Orphan Annotation Diagnostic Reporter (`crates/aico-core/src/trace.rs`)**:
   - Signature: `pub fn report_broken_annotations(report: &TraceReport) -> Vec<DiagnosticWarning>`
   - Seam Test: `crates/aico-core/tests/trace_test.rs::test_broken_annotation_diagnostics`

---

## Out-of-Scope
- Automatic code repair or generative source refactoring.
- Dynamic runtime AST bytecode instrumentation.
- Modifying compiler build flags or external cargo configurations.

---

## 5. Open Clarifications

None. Ratified in [ADR-0010](../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md).
