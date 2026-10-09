---
id: spec-ft-core-ui-visual-consistency
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 10
title: "WebUI Visual Token Alignment and Shell Consistency"
status: "Draft"
derives_from: ["brain-ec-core"]
created_at: "2026-10-09T19:15:00Z"
---

# Feature Specification: WebUI Visual Token Alignment and Shell Consistency

## 1. Problem Statement & User Value

While the AICO WebUI shell provides rich navigation across projects, epics, features, and specifications, minor inconsistencies in visual tokens, keyboard shortcut button redundancy, and component layout stability detract from the developer experience:

1. **Redundant Search Shortcut Triggers**: Currently, two separate `Ctrl+K` search shortcut buttons are rendered in close proximity: one in the top application header next to project selection (`ProjectSwitcher`), and a duplicate one in the sidebar header next to the platform version badge (`v0.1.1`). Only the top header trigger adjacent to project selection should be maintained.
2. **Unstable Epic Action Button Layout**: In the Home Dashboard (`DashboardView`), the "Open Epic" button inside the Project Epics grid wraps into two lines whenever an Epic title exceeds one line. The button layout must remain rigidly uniform and single-line regardless of title length.
3. **Disparate Visual Tokens & Semantic Colors**:
   - In OmniSearch, Features were styled with `text-purple-400` while in the Sidebar Tree View (`ShellLayout`), Tab Bar (`TabBar`), and Epic Hub (`EpicHubView`), Features were styled with `text-indigo-400`, creating visual collision with Epics (which are also Indigo). Features must canonically use **Purple** (`Package` icon with `text-purple-400`).
   - In the Home Dashboard Codebase Repositories card (`GitRepositoriesCard`), the `specs` repository tag rendered in purple (`text-purple-400`), conflicting with the canonical **Sky** blue (`text-sky-400`, `BookOpen`) used for specifications across tree view, tab bar, and tag badges.
   - Architecture Decision Records (ADRs) used different icons across views (`FileText` in Tree View / Tab Bar vs. `Scale` in OmniSearch). ADRs must consistently use the **Scale** icon (`text-amber-400`) to symbolize architectural balance and trade-off decisions.

Resolving these issues ensures a cohesive, predictable, and polished design language across all AICO WebUI views.

---

## In-Scope

- **Search Trigger Deduplication**:
  - Retaining the primary `Ctrl+K` button in the top navigation header adjacent to `ProjectSwitcher`.
  - Removing the duplicate `Ctrl+K` button from the sidebar workspace header next to `v0.1.1` in `ShellLayout.tsx`.
- **Dashboard Epic Card Layout Stabilization**:
  - Enforcing `shrink-0 whitespace-nowrap` on the "Open Epic" action button in `DashboardView.tsx`.
  - Providing `min-w-0 flex-1 mr-4` on the epic title/badge container to allow flexible truncation or multi-line text without impacting action button geometry.
- **Cross-View Feature Color Alignment**:
  - Aligning Feature icons to `Package` with canonical **Purple** (`text-purple-400`) in `ShellLayout.tsx`, `TabBar.tsx`, and `EpicHubView.tsx`.
  - Aligning standalone feature badges across `ShellLayout.tsx` and `FeatureViewer.tsx` to purple styling.
- **Dashboard Repositories Specs Tag Parity**:
  - Updating `getTypeBadgeColor('specs')` in `GitRepositoriesCard.tsx` to use canonical **Sky** styling (`bg-sky-950/80 text-sky-400 border-sky-800/60`), matching `BookOpen text-sky-400`.
- **Cross-View ADR Icon and Color Parity**:
  - Updating Architecture Decision Record (ADR) icons across `ShellLayout.tsx`, `TabBar.tsx`, and `OmniSearchModal.tsx` to uniformly use `<Scale className="... text-amber-400" />`.
- **Seam Tests**:
  - Updating and extending automated component tests in Vitest verifying the unified tokens, button layout properties, and shortcut trigger placement.

---

## Out-of-Scope (Future Upgrades)

- Redesigning the entire project dashboard card grid layout or adding customizable widget layouts.
- Altering the git polling and background status fetch engine in `GitRepositoriesCard`.
- Changing the underlying fuzzy scoring or search index logic in `searchEngine.ts`.

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer viewing the WebUI shell, I want a single clear `Ctrl+K` search button in the top navigation bar next to project selection, so that the sidebar header remains clean and uncrowded.
   - *Independent Test*: Inspect rendered DOM; assert the top header contains the `Ctrl+K` trigger button, and assert the sidebar header next to `v0.1.1` does not render a duplicate search trigger button.

2. **US-02** (Priority: P1): As a developer browsing the Home Dashboard with long epic titles, I want the "Open Epic" button to remain a stable single-line button with consistent dimensions, so that cards render with a uniform layout.
   - *Independent Test*: Render an epic with a multi-line title; assert the "Open Epic" button possesses `shrink-0` and `whitespace-nowrap` classes and does not break into two lines.

3. **US-03** (Priority: P1): As a developer navigating features across the Tree View, Tab Bar, Epic Hub, and OmniSearch, I want Features to consistently render in purple (`text-purple-400`, `Package`), so that Features are immediately distinguishable from Epics (indigo).
   - *Independent Test*: Inspect Feature items in `ShellLayout`, `TabBar`, and `EpicHubView`; assert each renders `<Package ...>` with `text-purple-400`.

4. **US-04** (Priority: P1): As a developer inspecting the Codebase Repositories card on the dashboard, I want the `specs` repository type badge to match the sky-blue color of the specification icons, so that domain artifacts share an identical color language.
   - *Independent Test*: Inspect `specs` repository badge in `GitRepositoriesCard`; assert it has sky-blue styling (`text-sky-400 bg-sky-950/80 border-sky-800/60`).

5. **US-05** (Priority: P2): As a developer navigating Architecture Decision Records, I want ADRs to use the `Scale` icon in amber across all views, so that architectural balancing decisions are semantically represented.
   - *Independent Test*: Inspect ADR items in `ShellLayout`, `TabBar`, and `OmniSearchModal`; assert each renders `Scale` with `text-amber-400`.

---

## 3. Functional Requirements (EARS)

- **REQ-001**: The WebUI shell shall render a single global OmniSearch trigger button located in the top navigation bar adjacent to the project selection component, and shall not render a duplicate search button in the sidebar workspace header.
- **REQ-002**: On the Home Dashboard, each Project Epic card action button shall display the text "Open Epic" on a single unbroken line (`shrink-0 whitespace-nowrap`), preventing line wrapping regardless of the title text length.
- **REQ-003**: The WebUI components (including `ShellLayout`, `TabBar`, `EpicHubView`, `OmniSearchModal`, and `TagBadge`) shall uniformly render Feature items using the canonical purple design token (`text-purple-400`) and the `Package` icon.
- **REQ-004**: On the Home Dashboard Codebase Repositories card, the repository type badge for `specs` repositories shall render with the canonical sky-blue token (`text-sky-400 bg-sky-950/80 border-sky-800/60`), matching the `BookOpen` specifications icon across the workspace.
- **REQ-005**: The WebUI components (including `ShellLayout`, `TabBar`, and `OmniSearchModal`) shall uniformly render Architecture Decision Record (ADR) items using the `Scale` icon and amber design token (`text-amber-400`).

---

## 4. Designated Public Test Seams

- **Sidebar Header & Tree Seam**: `frontend/tests/ShellLayoutSearch.test.tsx` (verifies REQ-001, REQ-003, REQ-005)
- **Dashboard Action Button & Repositories Seam**: `frontend/tests/DashboardView.test.tsx` (verifies REQ-002, REQ-004)
- **Tab Bar Visual Tokens Seam**: `frontend/tests/TabBar.test.tsx` (verifies REQ-003, REQ-005)
- **OmniSearch Visual Tokens Seam**: `frontend/tests/OmniSearchModal.test.tsx` (verifies REQ-003, REQ-005)
