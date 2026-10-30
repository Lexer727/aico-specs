---
id: adr-0012
type: adr
aico_skills_version: "0.1.2"
title: "Distribution via GitHub Releases & Inno Setup Hybrid Installer"
status: "Proposed"
derives_from: ["ADR-0006", "ADR-0008"]
created_at: "2026-10-30T19:19:08.350Z"
---

# ADR-0012: Distribution via GitHub Releases & Inno Setup Hybrid Installer

## Context & Problem Statement

[ADR-0006](0006-packaging-aico-specs-and-version-tagging.md) mandates a single self-contained AICO binary and [ADR-0008](0008-unified-root-discovery-and-project-manifest.md) mandates a single AICO Root chosen at installation time. Neither ADR defines **how** the binary reaches end users, **where** it is installed, how it is **updated**, or how it is **removed**.

Today AICO only runs from a developer checkout (`cargo run -p aico-server`):
- The server falls back to a hard-coded `d:/aico` root and a hard-coded `d:/aico/aico-specs` specs path.
- Workspace skills, rules, and `mcp_config.json` are synchronized exclusively from the `codebase/aico/skills` git submodule, which end users do not have.
- No CI pipeline or release artifact exists.

End users additionally need the surrounding toolchain (Git, GitHub CLI, Node.js, AI agent harnesses) to use AICO productively.

## Decision Drivers

- No self-hosted download server or infrastructure cost.
- Reproducible, tag-driven builds; release version strictly equals Cargo workspace version.
- Users always obtain the newest release, while installation still works offline.
- Per-user installation without administrator rights.
- The developer's existing AICO Root (`d:/aico`) and its `.aico/storage` must be reusable by the same installer.
- Uninstallation must never destroy user-authored specs or codebases.

## Considered Options

1. **Option A (Chosen)**: GitHub Actions builds on `v*` tags and publishes a GitHub Release (`aico-windows-x64.zip`, `.sha256`, `aico-setup-<version>.exe`). An **Inno Setup 6** hybrid installer bundles the release payload offline and, at install time, queries `releases/latest` to offer a newer verified bundle. Binaries are installed into `<root>/.aico/bin`.
2. **Option B**: Self-hosted download/update server.
3. **Option C**: Web-only bootstrapper that always downloads at install time (no offline payload).
4. **Option D**: Custom Rust GUI installer crate (egui/iced).
5. **Option E**: WiX / MSI package.

## Decision Outcome

Chosen option: **Option A**.

### 1. Release Channel
- Source repository `Lexer727/aico` is public; releases are published directly on it.
- A pushed tag `vX.Y.Z` triggers `.github/workflows/release.yml`; tags containing `-` (e.g. `v0.2.0-rc.1`) are published as prereleases.
- The stable download URL `https://github.com/Lexer727/aico/releases/latest/download/aico-windows-x64.zip` always resolves to the newest non-prerelease bundle.
- Every bundle ships a `manifest.json` (version, commit, file hashes) and a detached `.sha256` checksum.

### 2. Install Layout
```
<AICO Root>/
├── specs/
├── codebase/
├── .agents/
└── .aico/              # AICO Home
    ├── storage/        # preserved across upgrades and (by default) uninstall
    ├── logs/
    └── bin/            # Install Bin Directory (aico-server.exe, skills/, uninstaller, manifest.json)
```

### 3. Runtime Root Discovery
The server derives the AICO Root without hard-coded paths: `--root` → `AICO_ROOT` → executable-relative (`<root>/.aico/bin/`) → nearest ancestor of the working directory containing `.aico/` → working directory.

### 4. Uninstall Boundary
The uninstaller removes only installer-owned artifacts (`.aico/bin`, shortcuts, PATH entry, uninstall registry key). Storage, logs, and AICO-managed `.agents` skills are removed only on explicit opt-in. `specs/`, `codebase/`, and the User Config Home (`~/.aico`) are never removed.

### Positive Consequences
- Zero hosting cost; GitHub CDN serves downloads.
- Same installer handles fresh install and in-place upgrade.
- The developer's workspace becomes an ordinary install target, eliminating dev/user divergence.

### Negative Consequences & Follow-ups
- Unsigned installers trigger Windows SmartScreen warnings; code signing (e.g. Azure Trusted Signing) is a deferred follow-up.
- Windows-only installer in v1; macOS/Linux packaging is deferred.
- Unauthenticated GitHub API calls are rate-limited (60 requests/hour/IP); the installer must degrade gracefully to the bundled payload.
