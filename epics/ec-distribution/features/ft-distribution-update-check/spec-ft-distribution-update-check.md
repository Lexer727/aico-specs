---
id: spec-ft-distribution-update-check
type: spec
aico_skills_version: "0.1.2"
epic: "ec-distribution"
order: 6
title: "Release Update Check & WebUI Banner"
status: "Completed"
derives_from: ["brain-ec-distribution", "ADR-0012"]
created_at: "2026-10-30T19:19:09.613Z"
---

# Feature Specification: Release Update Check & WebUI Banner

## 1. Problem Statement & User Value

Once users have installed AICO, they need an unobtrusive way to learn when a newer release has been published on GitHub without manually polling the repository. This feature adds a cached backend endpoint `GET /api/v1/system/update` that compares the running `CARGO_PKG_VERSION` against GitHub's `releases/latest` endpoint and surfaces a dismissible notification banner in the WebUI top bar with a direct link to the release notes and installer download.

---

## In-Scope

- Backend endpoint `GET /api/v1/system/update` in `crates/aico-server` returning `{ current_version, latest_version, update_available, release_url, checked_at }`.
- In-memory 6-hour caching of the GitHub `releases/latest` response so the server does not exhaust unauthenticated rate limits.
- Environment override `AICO_UPDATE_API_URL` (for testing) and opt-out `AICO_NO_UPDATE_CHECK=1`.
- WebUI top-bar notification badge/banner when `update_available == true`, linking to `release_url` and dismissible per version in `localStorage`.

---

## Out-of-Scope

- Automatic background self-replacement of the running `aico-server.exe` binary without user interaction.

---

## 2. User Stories

1. **US-01** (Priority: P2): As an AICO user, I want the WebUI to notify me when a newer version of AICO is available on GitHub, so that I can download and run the updated installer at my convenience.
   - *Independent Test*: Mock the GitHub latest-release endpoint to return `tag_name: "v99.0.0"`; query `GET /api/v1/system/update` and verify `update_available: true` and that the WebUI renders the update banner.
2. **US-02** (Priority: P2): As an offline user, I want failed update checks to degrade silently without showing error toasts or blocking the WebUI.
   - *Independent Test*: Point `AICO_UPDATE_API_URL` to an unreachable local port; query `GET /api/v1/system/update` and verify HTTP 200 with `update_available: false`.

---

## 3. Functional Requirements (EARS)

### Event-Driven Requirements
- **REQ-001**: When a client sends `GET /api/v1/system/update`, the server shall return a JSON payload containing `current_version` (from `CARGO_PKG_VERSION`), `latest_version`, `update_available` (boolean SemVer comparison `latest > current`), `release_url`, and `checked_at`, caching upstream GitHub responses for 6 hours.
- **REQ-002**: When the WebUI loads and `GET /api/v1/system/update` returns `update_available: true` for a version not yet dismissed by the user, the WebUI shall render a non-blocking update banner with a link to `release_url` and a dismiss button.

### Unwanted Behavior Requirements
- **REQ-003**: If the upstream GitHub request fails, times out, or `AICO_NO_UPDATE_CHECK=1` is set, then `GET /api/v1/system/update` shall return HTTP 200 with `update_available: false` and `latest_version: null` without failing the request.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Update Check Endpoint (`crates/aico-server/tests/update_check_test.rs`)**:
   - Seam Test: Tests `GET /api/v1/system/update` against a local mock HTTP server returning newer, equal, and error responses, plus cache hit behavior.
2. **WebUI Update Banner (`frontend/tests/UpdateBanner.test.tsx`)**:
   - Seam Test: Vitest test verifying banner rendering when `update_available: true`, hiding when `false`, and `localStorage` dismissal persistence.

---

## 5. Open Clarifications

- Architectural alignment: [ADR-0012](../../../../adr/0012-distribution-github-releases-and-hybrid-installer.md).
