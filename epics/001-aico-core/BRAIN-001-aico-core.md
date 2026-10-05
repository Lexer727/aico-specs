# Brainstorm: AICO Core Architecture & Local-First Platform

- **Epic ID**: `001-aico-core`
- **Status**: Approved / Ratified
- **Topology**: Composite Epic (7 Child Specifications)

---

## 1. Executive Summary & Problem Statement

Modern AI-assisted software engineering suffers from:
1. **Context amnesia & hallucinated refactors**: LLMs lack structured knowledge of dependency graphs, test seams, and architectural decisions across sessions.
2. **Heavyweight runtime sprawl**: Existing agent frameworks require external Docker daemons, JVM runtimes, background daemon processes, or cloud telemetry.
3. **Fragile security boundaries**: Agent code execution is either entirely gated (halting autonomous flow) or dangerously unconstrained.
4. **Poor multi-project ergonomics**: Code and specs are conflated, making multi-repo workspaces messy and brownfield on-ramps difficult.

**AICO (Autonomous Intelligent Code Orchestrator)** solves this by building a zero-daemon, local-first Spec-Driven Development (SDD) environment delivered as a single compiled binary with embedded storage (LanceDB + Kùzu), deterministic MCP tool gates, a built-in Axum Web server hosting a React UI, and a decoupled repository layout.

---

## 2. Core Architectural Decisions

### 2.1 Host Runtime & Zero-Daemon Architecture (ADR-0001)
- Embedded Rust HTTP & WebSocket server powered by **Axum** running on port `4242`.
- Binds to `0.0.0.0` to support LAN mobile pairing via authenticated QR code / pairing token.
- No standalone background daemons: the binary spins up Axum in-process when invoked, serving the embedded React/Tailwind WebUI.

### 2.2 Decoupled Repository Layout (ADR-0002, ADR-0008)
- Single root installation directory (e.g. `d:/aico/`).
- Code repositories live under `d:/aico/codebase/<project>` (or custom paths).
- Dedicated specifications repositories live under `d:/aico/<project>-specs`.
- Every specs repo declares its codebase link in `.aico-project.json`.

### 2.3 3-Tier Security Policy Engine & Auto-Mode (ADR-0003)
- **Tier 1 (Read-Only & Testing)**: Auto-executed without human prompts.
- **Tier 2 (Workspace Modification)**: Auto-executed with immediate audit logging to Kùzu.
- **Tier 3 (System Destructive & Network Egress)**: Hard gated; pauses agent execution and requests interactive human approval over WebSocket.

### 2.4 Dual Embedded Storage Engine (ADR-0006)
- **LanceDB**: Local-first vector store for semantic search over code chunks, docs, and conversation history.
- **Kùzu**: Embedded property graph database for AST call graphs, SDD requirement-to-code traceability, and task DAG execution.

### 2.5 Multi-Modal Assets & Human Gate Tasks (ADR-0007)
- File drop zones across Chat, Sidebar, and Epic Hub.
- Dropped assets are checksummed, indexed into LanceDB/Kùzu, and made available for multimodal analysis.
- Plan DAGs support Human Gate Tasks (`role: user`, `type: asset_delivery`) that block dependent agent tasks until satisfied.

---

## 3. Epic Decomposition (Composite Multi-Spec)

The `001-aico-core` epic decomposes into 7 specifications:

| Spec ID | Directory | Subsystem Focus |
|---|---|---|
| `SPEC-CORE-001` | `specs/01-host-runtime` | Axum HTTP/WS server, LAN pairing, graceful shutdown |
| `SPEC-CORE-002` | `specs/02-security-auto-mode` | 3-tier policy engine, auto-mode execution, audit trail |
| `SPEC-CORE-003` | `specs/03-storage-engines` | LanceDB vector store, Kùzu graph DB, tree-sitter AST |
| `SPEC-CORE-004` | `specs/04-project-discovery` | `.aico-project.json` manifest, multi-project scanning |
| `SPEC-CORE-005` | `specs/05-mcp-tool-gates` | Deterministic MCP gates, `@verifies`/`@implements` parser |
| `SPEC-CORE-006` | `specs/06-webui-shell` | Accordion sidebar, bottom/right dock, project tabs |
| `SPEC-CORE-007` | `specs/07-epic-hub` | Visual lineage canvas, asset drop zones, human gate tasks |

---

## 4. Next Steps
1. Task graph planning via `/aico-planning` to decompose specifications into vertical tracer bullets (`PLAN-*.md`).
2. Scaffold failing behavioral/contract tests (`/aico-test-scaffolding`).
3. Implementation phase (`/aico-implementation`) advancing the unblocked task frontier.
