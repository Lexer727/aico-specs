---
id: brain-ec-tooling
type: brainstorm
aico_skills_version: "0.1.2"
epic: "ec-tooling"
title: "Hybrid Deterministic SDD CLI & Stateful MCP Engine"
status: "Completed"
derives_from: ["ADR-0009", "ADR-0010"]
created_at: "2026-10-17T21:14:00Z"
---

# Brainstorm: Hybrid Deterministic SDD CLI & Stateful MCP Engine

## 1. Executive Summary & Problem Statement

During initial AICO bootstrapping, LLM coding agents handled repetitive, deterministic Spec-Driven Development (SDD) tasks manually via raw file edits—creating directories, writing YAML frontmatter headers, calculating relative markdown links, incrementing feature ordering, mutating task status fields in plans, rolling up status changes to parent artifacts, and appending amendment logs.

Meanwhile, an early prototype crate (`crates/aico-mcp`) attempted to gate artifact creation in Rust (`gate.rs`), but suffered from three structural problems:
1. **Bootstrap & Self-Hosting Friction**: Running a persistent compiled Rust MCP binary while actively developing and recompiling AICO on Windows creates binary and database file-locking hazards (`os error 32`) and couples the portable `aico-skills` harness to a platform-specific binary.
2. **Schema & Blueprint Drift**: Because `aico-mcp` lived in the Rust codebase while SDD templates evolved in `aico-skills`, `gate.rs` remained stuck on legacy v0.1.0 conventions (`epics/<epic>/specs/<spec>/SPEC-<slug>.md` with uppercase IDs), falling behind [ADR-0009](../../adr/0009-sdd-artifact-schema-and-task-lifecycle.md) and [ADR-0010](../../adr/0010-sdd-structural-grammar-standalone-features-and-naming.md).
3. **Misplaced Responsibilities**: Stateless markdown templating and frontmatter mutations belong alongside the skill definitions in `aico-skills`, whereas `aico-mcp` is uniquely suited for stateful, in-memory queries against the embedded Kùzu property graph and LanceDB vector store (`aico-storage`).

This epic establishes a **Hybrid Tooling Architecture**:
- A **Unified Deterministic Node.js CLI (`scripts/aico-cli.mjs`)** bundled inside `aico-skills` for all stateless filesystem, scaffolding, task lifecycle, DAG frontier, and blueprint verification operations.
- A **Repurposed Stateful Rust MCP Engine (`crates/aico-mcp`)** stripped of obsolete markdown file-scaffolding gates, upgraded to Blueprint v0.1.2 traceability rules, and wired to `aico-storage` for Kùzu lineage graph traversal and LanceDB hybrid semantic search.

---

## 2. Explored Approaches & Trade-Off Analysis

| Approach | Summary | Pros | Cons | Verdict |
| :--- | :--- | :--- | :--- | :--- |
| **Option A: Pure MCP (`crates/aico-mcp`)** | Expose all 12+ SDD file, task, and graph tools via a compiled Rust MCP server. | Typed JSON-RPC schemas; single Rust codebase. | Locks binaries/DBs on Windows during self-hosting (`cargo test`); breaks portability of `aico-skills`; bloats LLM context window on every turn. | **Rejected** |
| **Option B: Pure CLI Scripts (`aico-skills/scripts/`)** | Implement all SDD and graph operations as Node.js CLI scripts. | Zero-build portability; no file locking; zero idle context overhead; co-evolves with `SKILL.md`. | Ephemeral JS scripts cannot efficiently hold persistent connections to embedded Kùzu and LanceDB stores. | **Rejected** |
| **Option C: Hybrid Architecture (CLI Scripts + Stateful MCP)** | Stateless SDD file/task/verification ops in unified `aico-cli.mjs`; stateful Kùzu/LanceDB & AST validators in `aico-mcp`. | Eliminates bootstrap locking; 100% portable skills; keeps Blueprint v0.1.2 rules in `aico-skills`; leverages Rust `aico-storage` for graph/vector queries. | Requires clear boundary definition between `aico-cli.mjs` and `aico-mcp`. | **Chosen** |

---

## 3. Core Architectural Decisions

### 3.1 Unified CLI Entry Point (`codebase/aico-skills/scripts/aico-cli.mjs`)
Rather than scattering ad-hoc scripts across skill folders, `aico-skills` exposes a single zero-dependency Node.js CLI entry point (`scripts/aico-cli.mjs`) with deterministic subcommands:
- `version`: Reads canonical `aico_skills_version` from `plugin.json` (consolidating `get-version.mjs`).
- `scaffold-epic`: Creates `epics/ec-<short_id>/features/` and initializes `brain-ec-<short_id>.md` with canonical Line-1 YAML frontmatter.
- `scaffold-feature`: Creates standalone (`features/ft-<slug>/`) or epic-child (`epics/ec-<id>/features/ft-<id>-<slug>/`) containers, auto-calculates sibling `order`, validates `repositories` against `.aico-project.json`, computes relative ADR link depths, and writes skeleton `spec-ft-*.md` and `plan-ft-*.md` files.
- `scaffold-adr` & `upsert-glossary`: Auto-assigns the next sequential `00XX` ADR number or inserts/updates a domain term in `glossary.md`.
- `task-status`: Transitions a task across the 6-state lifecycle (`Pending`, `In Progress`, `Done`, `Blocked`, `Skipped/Canceled`, `Rewritten`), enforces mandatory `Reason` and `Rewrite Target` fields, gates `Done` transitions on seam/annotation checks, automatically unblocks downstream `Blocked` tasks, and rolls up `In Progress` / `Completed` status to parent `plan-ft-*.md`, `spec-ft-*.md`, and `brain-ec-*.md` frontmatter.
- `plan-amend`: Appends a new sequential `TASK-<PREFIX>-<SEQ>` block, records a timestamped entry in the single consolidated `## 3. Amendment Log`, and reopens plan/spec/epic statuses to `"In Progress"`.
- `frontier`: Parses `Blocked by` edges across plans/epics, detects circular or dangling dependencies, and outputs unblocked `Pending` tasks.
- `trace-tags`: Outputs canonical `// @verifies` and `// @implements` comment tags for a given spec or task.
- `verify`: Executes the full dual-axis Blueprint & Ground-Truth verification audit (consolidating `verify-artifacts.mjs` while keeping wrapper scripts for backward compatibility).

### 3.2 Repurposing `crates/aico-mcp`
- **Remove**: `crates/aico-mcp/src/gate.rs` (`create_sdd_artifact`), which is superseded by `aico-cli.mjs scaffold-*`.
- **Update**: Upgrade `AnnotationExtractor`, `EarsValidator`, and `TraceabilityTracker` to Blueprint v0.1.2 conventions (canonical `spec-ft-*` lowercase IDs and prefixed `REQ-*` IDs).
- **Extend**: Connect `aico-mcp` to `aico-storage` (`KuzuGraphStore` and `HybridQueryEngine`) to expose `query_lineage_graph`, `search_hybrid`, and `find_dangling_requirements`.

---

## 4. Epic Decomposition (Child Features)

1. **`ft-tooling-sdd-cli`** (`order: 1`):
   - Unified Node.js `aico-cli.mjs` entry point in `aico-skills` covering scaffolding, task lifecycle mutations, status roll-ups, plan amendments, DAG frontier evaluation, trace-tag generation, and dual-axis verification.
2. **`ft-tooling-mcp-repurpose`** (`order: 2`):
   - Removal of legacy `gate.rs`, v0.1.2 upgrade of `annotation.rs`, `ears.rs`, and `traceability.rs`, and addition of stateful `query_lineage_graph`, `search_hybrid`, and `find_dangling_requirements` MCP tools backed by `aico-storage`.
3. **`ft-tooling-skills-integration`** (`order: 3`):
   - Updating `aico-skills` `SKILL.md` workflows to mandate `aico-cli.mjs` and repurposed `aico-mcp` tools, plus verifying workspace synchronization via `SkillsSynchronizer`.
