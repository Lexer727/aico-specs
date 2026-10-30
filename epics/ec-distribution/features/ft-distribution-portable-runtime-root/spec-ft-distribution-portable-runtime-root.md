---
id: spec-ft-distribution-portable-runtime-root
type: spec
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 1
title: "Portable Runtime Root Resolution & Bundled Skills Source"
status: "Completed"
derives_from: ["brain-ec-distribution", "ADR-0012"]
created_at: "2026-10-30T19:19:08.717Z"
---

# Feature Specification: Portable Runtime Root Resolution & Bundled Skills Source

## 1. Problem Statement & User Value

`aico-server` currently assumes it runs from a developer checkout. It falls back to a hard-coded `d:/aico` AICO Root (in `crates/aico-server/src/main.rs` and `resolve_workspace_root_from_specs_path` in `crates/aico-server/src/server.rs`), uses a hard-coded `d:/aico/aico-specs` specs fallback (`crates/aico-server/src/assets.rs`), synchronizes skills only from the `codebase/aico/skills` submodule, prefers cwd-relative on-disk frontend assets over embedded assets, and prints a hard-coded version.

When installed at `<root>/.aico/bin/aico-server.exe` on an end-user machine, none of these assumptions hold. This feature makes the binary **location-independent**: it derives the AICO Root from its own executable path, ships and uses its own bundled skills, serves only embedded WebUI assets in release builds, reports its real version, and writes logs into AICO Home. The developer workflow (`cargo run` from `d:/aico/codebase/aico`) continues to work through ancestor discovery.

---

## In-Scope

- A pure, unit-testable AICO Root resolver in `aico-core`.
- Removal of all hard-coded absolute paths (`d:/aico`, `d:/aico/aico-specs`) from runtime crates.
- Bundled skills fallback source `<exe_dir>/skills` for `SkillsSynchronizer`.
- Restricting on-disk frontend asset lookup to debug builds or an explicit opt-in environment variable.
- `--version` and `--root <path>` command line arguments.
- Banner version derived from `CARGO_PKG_VERSION`.
- File logging to `<root>/.aico/logs/aico-server.log`.

---

## Out-of-Scope

- Packaging, CI, or installer logic (covered by sibling features).
- macOS/Linux install layouts.
- Changing the storage layout under `<root>/.aico/storage`.
- System tray or background service mode.

---

## 2. User Stories

1. **US-01** (Priority: P1): As an end user who installed AICO into `C:\Users\me\aico`, I want `aico-server.exe` to discover `C:\Users\me\aico` as its AICO Root automatically, so that my projects are found without configuration.
   - *Independent Test*: Resolve the root for an executable path `<tmp>/.aico/bin/aico-server.exe` with no flags or environment variables; expect `<tmp>`.
2. **US-02** (Priority: P1): As the AICO developer, I want `cargo run -p aico-server` from `d:/aico/codebase/aico` to keep using `d:/aico`, so that my existing `.aico/storage` is reused.
   - *Independent Test*: Resolve the root for cwd `<tmp>/codebase/aico` where `<tmp>/.aico/` exists and the executable lives under `target/debug`; expect `<tmp>`.
3. **US-03** (Priority: P1): As an end user without the AICO codebase, I want workspace skills and rules to be synchronized from the skills bundled with my installation, so that `.agents/` is populated.
   - *Independent Test*: Sync with an empty `<root>/codebase` and a populated `<exe_dir>/skills`; expect `.agents/skills/aico-*` created.
4. **US-04** (Priority: P2): As a support engineer, I want `aico-server.exe --version` to print the exact version, so that installs can be verified in CI and bug reports.
   - *Independent Test*: Run the binary with `--version`; expect `CARGO_PKG_VERSION` on stdout and exit code 0.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The system shall resolve the AICO Root by evaluating, in order: the `--root <path>` argument, the `AICO_ROOT` environment variable, the executable-relative root (the grandparent of the executable directory when the executable directory is named `bin` and its parent is named `.aico`), the nearest ancestor of the current working directory that contains a `.aico` directory, and finally the current working directory.
- **REQ-002**: The system shall not reference the hard-coded absolute paths `d:/aico` or `d:/aico/aico-specs` in any runtime crate source under `crates/*/src`.
- **REQ-005**: The system shall print `AICO v<CARGO_PKG_VERSION>` in its startup banner, sourcing the version from `env!("CARGO_PKG_VERSION")`.

### Event-Driven Requirements
- **REQ-006**: When `aico-server` is invoked with the `--version` argument, the system shall print `aico-server <CARGO_PKG_VERSION>` to standard output and exit with code 0 without binding any port or touching storage.
- **REQ-007**: When the server starts, the system shall create `<root>/.aico/logs/` if absent and write its tracing log output to `<root>/.aico/logs/aico-server.log` in addition to standard output.

### State-Driven Requirements
- **REQ-003**: While `<root>/codebase/aico/skills` contains no skill directory with a `SKILL.md`, the `SkillsSynchronizer` shall use `<exe_dir>/skills` as its synchronization source.
- **REQ-004**: While the server binary is built without `debug_assertions` and the environment variable `AICO_DEV_ASSETS` is not set to `1`, the system shall serve WebUI assets exclusively from the embedded `FrontendAssets` bundle.

### Unwanted Behavior Requirements
- **REQ-008**: If a path supplied via `--root` or `AICO_ROOT` does not exist, then the system shall print an error naming the path and exit with a non-zero code.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **AICO Root Resolver (`aico_core::root`)**:
   - Signature: `pub fn resolve_aico_root(cli_root: Option<&Path>, env_root: Option<&Path>, exe_path: &Path, cwd: &Path) -> Result<PathBuf, RootResolutionError>`
   - Seam Test: `crates/aico-core/tests/root_resolution_test.rs` (temp-dir matrix covering each precedence tier and REQ-008).
2. **Skills Source Resolver (`SkillsSynchronizer`)**:
   - Signature: `pub fn resolve_skills_source(root: &Path, exe_dir: &Path) -> Option<PathBuf>`
   - Seam Test: `crates/aico-core/tests/skills_sync_test.rs::test_bundled_skills_fallback`.

### Secondary Seams:
3. **Binary CLI Contract**:
   - Seam Test: `crates/aico-server/tests/cli_version_test.rs` (spawns `env!("CARGO_BIN_EXE_aico-server")` with `--version`).
4. **Asset Source Policy**:
   - Signature: `pub fn disk_assets_enabled() -> bool`
   - Seam Test: `crates/aico-server/tests/assets_test.rs::test_disk_assets_policy`.
5. **Hard-Coded Path Guard**:
   - Seam Test: `crates/aico-core/tests/no_hardcoded_paths_test.rs` (scans `crates/*/src/**/*.rs` for forbidden literals).

---

## 5. Open Clarifications

- Architectural alignment: [ADR-0012](../../../../adr/0012-distribution-github-releases-and-hybrid-installer.md), [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
- `resolve_workspace_root_from_specs_path` shall delegate to the process-wide resolved AICO Root (stored once at startup) instead of re-reading `AICO_ROOT` and falling back to `d:/aico`.
