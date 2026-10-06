---
id: SPEC-fast-trace
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

## Purpose
Provides instantaneous CLI and in-editor validation of `@implements` and `@verifies` code comment annotations against formal EARS requirement identifiers across the repository without requiring full test suite execution.

## In-Scope
- Single-pass regex and Tree-sitter parsing of source files for SDD tags.
- Verification that all referenced `SPEC-*::REQ-*` tags exist in approved specs.
- Reporting unresolved or orphan contract references with line numbers.

## Out-of-Scope
- Automatic code repair or AI code generation.
- Dynamic runtime AST instrumentation.

## Behavioral Requirements

### REQ-001: Scan Workspace Source Files (Ubiquitous)
The system SHALL scan all workspace source files for `@implements <spec_id>::<req_id>` annotations upon request.

### REQ-002: Verify Against Specification Catalog (Event-Driven)
WHEN a source annotation is encountered, the system SHALL cross-reference the requirement ID against parsed specifications in the active project.

### REQ-003: Flag Unresolved References (Unwanted Behavior)
IF an annotation references a non-existent specification or requirement ID, THEN the system SHALL report an unresolved diagnostic warning with the exact source path and line number.
