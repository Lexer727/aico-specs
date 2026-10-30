---
id: spec-ft-distribution-release-pipeline
type: spec
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 2
title: "Automated GitHub Actions Release Pipeline"
status: "Completed"
derives_from: ["brain-ec-distribution", "ADR-0012"]
created_at: "2026-10-30T19:19:08.895Z"
---

# Feature Specification: Automated GitHub Actions Release Pipeline

## 1. Problem Statement & User Value

There is no CI pipeline and no release artifact for AICO. Producing a distributable build today requires a developer to manually build the frontend, compile a release binary (with the `skills` submodule initialized), and copy files around. This feature introduces a **tag-driven GitHub Actions pipeline** that builds, packages, checksums, and publishes a versioned **Release Bundle** and installer to GitHub Releases, so that the installer and update check can always fetch `releases/latest` without any self-hosted server.

---

## In-Scope

- A local/CI-shared packaging script `scripts/package-release.ps1` producing the Release Bundle, Release Manifest, and checksum.
- A GitHub Actions workflow `.github/workflows/release.yml` triggered by `v*` tags and `workflow_dispatch`.
- Tag-to-Cargo version consistency gate.
- Publishing the Release Bundle, checksum, and installer (built by `ft-distribution-windows-installer`) as GitHub Release assets.
- Rust and npm build caching.

---

## Out-of-Scope

- Code signing of binaries and installer.
- macOS/Linux build targets.
- Automatic changelog generation beyond GitHub's generated release notes.
- Pull-request CI (test-only workflows) — may be added separately.

---

## 2. User Stories

1. **US-01** (Priority: P1): As the AICO maintainer, I want pushing `v0.2.0` to produce a GitHub Release with all installable assets, so that I never build releases by hand.
   - *Independent Test*: Push a prerelease tag on a fork; observe a Release with zip, checksum, and installer assets.
2. **US-02** (Priority: P1): As the installer, I want a stable URL and a checksum for the newest bundle, so that I can download and verify it.
   - *Independent Test*: Download `releases/latest/download/aico-windows-x64.zip` and its `.sha256`; verify the hash matches.
3. **US-03** (Priority: P2): As the maintainer, I want to dry-run the pipeline without publishing, so that I can validate packaging changes safely.
   - *Independent Test*: Trigger `workflow_dispatch`; observe uploaded workflow artifacts and no Release.

---

## 3. Functional Requirements (EARS)

### Event-Driven Requirements
- **REQ-001**: When a Git tag matching `v*` is pushed, the release workflow shall check out the repository with submodules, build the frontend via `npm ci` and `npm run build` in `frontend/`, and build `aico-server` with `cargo build --release -p aico-server` on a `windows-latest` runner.
- **REQ-005**: When the packaging step completes for a tag build, the release workflow shall create a GitHub Release named after the tag and upload `aico-windows-x64.zip`, `aico-windows-x64.zip.sha256`, and `aico-setup-<version>.exe` as assets.
- **REQ-006**: When the workflow is triggered via `workflow_dispatch`, the release workflow shall run all build and packaging steps and upload the outputs as workflow artifacts without creating a GitHub Release.

### Ubiquitous Requirements
- **REQ-003**: The packaging script shall produce `aico-windows-x64.zip` containing `aico-server.exe`, a `skills/` directory copied from the `skills` submodule excluding `.git`, `prerequisites.json`, and `manifest.json`, plus a sibling file `aico-windows-x64.zip.sha256` containing the lowercase hex SHA-256 of the zip.
- **REQ-004**: The packaging script shall write `manifest.json` containing `version`, `commit`, `built_at` (ISO 8601 UTC), and `files` (an array of objects with relative `path` and lowercase hex `sha256` for every file in the bundle except `manifest.json`).

### Unwanted Behavior Requirements
- **REQ-002**: If the pushed tag version (without the leading `v`) differs from `workspace.package.version` in `Cargo.toml`, then the release workflow shall fail before any build step with a message naming both versions.

### Optional Feature Requirements
- **REQ-007**: Where the tag name contains a `-` pre-release suffix, the release workflow shall mark the GitHub Release as a prerelease.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Packaging Script Contract (`scripts/package-release.ps1`)**:
   - Signature: `package-release.ps1 -Version <semver> -BinaryPath <path> -SkillsPath <path> -CatalogPath <path> -OutDir <path>`
   - Seam Test: `scripts/tests/package-release.Tests.ps1` (Pester): asserts zip layout, manifest schema and hashes, checksum file content.
2. **Version Gate (`scripts/check-release-version.ps1`)**:
   - Signature: `check-release-version.ps1 -Tag <tag> -CargoToml <path>`
   - Seam Test: `scripts/tests/check-release-version.Tests.ps1` (match, mismatch, prerelease suffix).

### Secondary Seams:
3. **Workflow Dry Run**:
   - Seam Test: `.github/workflows/release.yml` via `workflow_dispatch` (human-observed CI run).

---

## 5. Open Clarifications

- Architectural alignment: [ADR-0012](../../../../adr/0012-distribution-github-releases-and-hybrid-installer.md).
- The installer build step (`iscc`) is contributed by `ft-distribution-windows-installer`; until it exists, the workflow publishes only the zip and checksum.
- `prerequisites.json` is contributed by `ft-distribution-prerequisite-provisioning`; until it exists, the packaging script accepts an absent catalog and omits it.
