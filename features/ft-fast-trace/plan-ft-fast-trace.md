---
id: plan-ft-fast-trace
type: plan
aico_version: "0.1.2"
epic: null
order: 1
title: "Fast Traceability Validator Implementation Plan"
status: "Completed"
derives_from: ["spec-ft-fast-trace"]
created_at: "2026-10-06T15:30:00Z"
---

# Implementation Plan: Fast Traceability Validator

---

## 1. Technical Architecture & Decisions

- **Crate**: `crates/aico-core` (part of the AICO Rust Cargo workspace).
- **Core Dependencies**:
  - `tree-sitter`: In-process AST parsing of Rust, TypeScript, and Python comments for `@implements` and `@verifies` tags.
  - `regex`: Fast initial line scanner for instant sub-15ms project scans.
  - `walkdir`: Fast recursive traversal respecting `.gitignore` boundaries.
- **Scanning Strategy**:
  - In-process execution within `aico-server` daemon, eliminating external linter startup overhead.
  - Dual indexing: builds an in-memory inverted index of declared requirement IDs from `spec-*.md` frontmatter and body headings.
  - Real-time diagnostics emitted via server WebSocket topics (`trace_diagnostics`).

---

## 2. Task Graph (DAG)

```
TASK-FAST-TRACE-001 (Scanner Engine & Seams)
  └── TASK-FAST-TRACE-002 (Catalog Cross-Referencing)
        └── TASK-FAST-TRACE-003 (Orphan Diagnostic Warning Suite)
```

### Phase 1: Source Scanner & Seam Scaffolding

  - **TASK-FAST-TRACE-001**: Single-Pass Source Comment Annotation Scanner
  - **Status**: Done
  - **Addresses**: `REQ-001`
  - **Blocked by**: None (Can start immediately)
  - **What it delivers**:
    - Creates `crates/aico-core/src/trace.rs` with `TraceabilityScanner` struct.
    - Implements regex and Tree-sitter parsers extracting `@implements` and `@verifies` annotations with line numbers.
    - Exposes public trait seam `scan_workspace_annotations(root_path: &Path) -> Vec<SourceAnnotation>`.
  - **Verification Seam**: `crates/aico-core/tests/trace_test.rs::test_scan_workspace_annotations`

---

### Phase 2: Specification Catalog Cross-Referencing

- **TASK-FAST-TRACE-002**: Specification Catalog Validator & Reference Matching
  - **Status**: Done
  - **Addresses**: `REQ-002`
  - **Blocked by**: `TASK-FAST-TRACE-001`
  - **What it delivers**:
    - Implements specification parser indexing all declared requirement IDs (`spec-core-*::req-*`).
    - Compares scanned source annotations against indexed specification requirements.
    - Detects valid, active, and satisfied requirement bindings.
  - **Verification Seam**: `crates/aico-core/tests/trace_test.rs::test_validate_requirement_references`

---

### Phase 3: Diagnostic Diagnostics & Hardening

- **TASK-FAST-TRACE-003**: Orphan Annotation Diagnostics & Broken Reference Reporter
  - **Status**: Done
  - **Addresses**: `REQ-003`
  - **Blocked by**: `TASK-FAST-TRACE-002`
  - **What it delivers**:
    - Generates structured compiler-style diagnostic warnings for missing, typoed, or orphan requirement IDs.
    - Formats output with exact file path, line number, column, and recommended suggestion.
    - Integrates diagnostic reporting with CLI and server WebSocket event streams.
  - **Verification Seam**: `crates/aico-core/tests/trace_test.rs::test_broken_annotation_diagnostics`
