---
id: brain-ft-fast-trace
type: brain
aico_skills_version: "0.1.2"
epic: null
order: 1
title: "Fast Traceability Validator Brainstorming"
status: "Completed"
derives_from: ["../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md"]
created_at: "2026-10-06T15:40:00Z"
---

# Brainstorming: Fast Traceability Validator

## Problem Space & Ambiguity Analysis
Developers working in Spec-Driven Development (SDD) need instant feedback on whether their `@implements` and `@verifies` annotations in source code correctly reference valid requirement identifiers (e.g., `spec-core-host-runtime::req-001`). Currently, verifying annotations requires running the full build and test suite, which creates friction in tight inner-loop development.

## Trade-Off Matrix

| Dimension | In-Process Tree-Sitter Pass | Full Test Suite Execution | External Linter CLI |
| :--- | :--- | :--- | :--- |
| **Latency** | < 15ms per project scan | 5-15 seconds | 200-500ms startup |
| **Dependencies** | Zero external runtime | Rust toolchain required | Node or Python env required |
| **Precision** | Exact AST node spans | Exit-code only | Regex approximations |
| **Decision** | **Chosen**: Embedded in-process scanner for instant inner loop | Retain for CI verification gate | Reject |

## Key Decisions
1. Implement single-pass regex and Tree-sitter tag extraction within the local server process.
2. Provide deterministic warnings if any source symbol references an obsolete or missing requirement ID.
