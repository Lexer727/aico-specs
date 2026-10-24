---
id: spec-ft-core-dashboard-and-repo-card-refinement
type: spec
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 18
title: "Dashboard Workspace Refinement, Re-unified Repositories Grid & Workflow Automation"
status: "Completed"
derives_from: ["brain-ec-core", "ADR-0004", "ADR-0005", "ADR-0008"]
created_at: "2026-10-24T01:20:00.000Z"
---

# Feature Specification: Dashboard Workspace Refinement, Re-unified Repositories Grid & Workflow Automation

## 1. Problem Statement & User Value

Several usability, visual layout, and automation inconsistencies exist on the Home Dashboard and within multi-repository Git operations:

1. **Misleading Workspace Header Root Path**: The Workspace Overview header displays a hardcoded filesystem path `d:/aico/aico-specs` as the workspace root. `aico-specs` is the specs repository of the active project, while `d:/aico` is the workspace root. Redundant or hardcoded path text creates confusion.
2. **Asymmetrical Repositories Grid**: The separate "Specs Repository" section in `GitRepositoriesCard` renders a single specs repository card inside a 3-column grid, leaving two grid columns completely empty and pushing codebase repositories down unnecessarily.
3. **Uninformative Git Action Buttons**: Cloned repositories display Pull, Push, and Commit buttons without status-aware disablement or count indicators, making it impossible to know whether there are uncommitted changes, commits to push, or upstream commits to pull without clicking or running CLI commands.
4. **Manual Commit Friction & Lack of Agent Workflow Integration**: Clicking Commit currently opens a manual text box. Furthermore, agents executing tasks do not automatically stage, commit, and push across affected decoupled repositories upon task verification.
5. **Redundant Open Epic Buttons**: Project Epics cards render an unclickable text tag alongside a bulky `Open Epic` button, rather than using the interactive `TagBadge` component established across the design system.
6. **Flashing Mock Defaults on Reload**: Initial state in `App.tsx` and `DashboardView.tsx` falls back to hardcoded mock projects (`AICO Core SDD`) and default epics (`ec-core`, `ec-specs`, `10 Formal Specs`), causing false numbers and labels to flash during page reload.
7. **Repository Resolution Collision When Codebase Shares Project Name**: When a codebase repository has an identifier matching the project name (e.g. `aico`), server-side repository resolution erroneously defaults to the specs directory, causing Git commit/pull/push operations on the codebase (such as updating submodules) to target the specs repository and fail with "Working tree clean".
8. **Spec Tag Bloat on Dashboard Epic Cards**: Rendering individual `spec-ft-*` pills at the bottom of each Project Epic card on the Home Dashboard clutters the cards.
9. **Missing Workspace-Wide Lineage Graph on Home Dashboard**: The interactive lineage graph is only available inside individual Epic views (`?epic=ec-*`), lacking a project-wide view on the Home Dashboard that displays all Epics and their associated nodes together.
10. **Raw Unformatted Markdown Syntax Across UI Views**: Extracted markdown strings (such as `**bold**`, `*italic*`, and `` `inline code` `` in epic executive summaries, ADR decisions/contexts/consequences, glossary definitions, scope boundaries, EARS requirement statements, seam descriptions, and task deliverables) are rendered as raw text instead of formatted HTML.

---

## In-Scope

- Removing the hardcoded root path from `DashboardView.tsx` Workspace Overview header.
- Re-unifying the separate Specs Repository and Codebase Repositories sections in `GitRepositoriesCard.tsx` into a single, cohesive 3-column grid with Specs displayed first.
- Implementing state-aware disablement, badges, and tooltips for Git action buttons:
  - Pull disabled with tooltip when `behind === 0`; enabled with `Pull (${behind})` when `behind > 0`.
  - Push disabled with tooltip when `ahead === 0`; enabled with `Push (${ahead})` when `ahead > 0`.
  - Commit disabled with tooltip when working tree is clean; enabled with `Commit (${count})` when modified/untracked files exist.
- Implementing one-click AI commit in the UI (auto-generating conventional commit message from git status/diff and committing) without requiring manual text input.
- Updating AICO skills (`aico-implementation`, `aico-review-loop`, `aico-git`) to automatically stage, commit, and push changes upon task verification.
- Standardizing Project Epics cards in `DashboardView.tsx` to use clickable `TagBadge` navigation, eliminating the separate `Open Epic` button.
- Eliminating hardcoded `mockProjects` in `App.tsx` and `defaultEpics` in `DashboardView.tsx`, displaying clean empty/loading states (`0`, `-`, `"No epics found"`).
- Resolving repository directory lookup order in `server.rs` (`resolve_repo_dir`) to check manifest effective repositories before specs fallbacks, and passing explicit `path` telemetry from `GitRepositoriesCard.tsx`.
- Removing spec tag pills from Project Epics cards in `DashboardView.tsx`.
- Embedding the interactive `VisualLineageCanvas` on `DashboardView.tsx` to display all project Epics and their associated nodes (`ADR -> Epic -> Feature -> Task -> Verification Seam / Code`), with full-window tab pop-out support in `App.tsx`.
- Adding `<InlineMarkdown>` in `MarkdownView.tsx` and rendering inline/block markdown properly across `DashboardView.tsx`, `EpicHubView.tsx`, `FeatureViewer.tsx`, `SpecViewer.tsx`, `PlanViewer.tsx`, `GlossaryViewer.tsx`, and `AdrViewer.tsx`.

---

## Out-of-Scope

- Direct Git merge conflict resolution in the browser.
- Multi-branch branching and interactive rebase in WebUI.
- Native C++ Kùzu / LanceDB storage engine migration and mtime graph caching (deferred to dedicated storage epic).

---

## 2. User Stories

1. **US-01** (Priority: P1): As a developer visiting the Home Dashboard, I want clean workspace headers without misleading hardcoded paths, and clean zero/empty states on reload instead of flashing mock numbers, so that telemetry reflects actual workspace state.
   - *Independent Test*: Mount `DashboardView` with empty epics; verify hardcoded path is absent and counts display `0` without flashing mock epics.
2. **US-02** (Priority: P1): As a developer inspecting project repositories, I want all repositories rendered in a balanced 3-column grid with the Specs repo first, so that screen space is utilized without blank column gaps.
   - *Independent Test*: Render `GitRepositoriesCard` with 1 specs repo and 2 codebase repos; verify all 3 render in a single unified grid row.
3. **US-03** (Priority: P1): As a developer checking Git status, I want Pull, Push, and Commit buttons to visually indicate whether action is required (with counts and disabled states when up to date or clean), so that I know my repository sync state at a glance.
   - *Independent Test*: Mount `GitRepositoriesCard` with a synced clean repo and verify Pull, Push, and Commit are disabled with descriptive tooltips.
4. **US-04** (Priority: P1): As a developer viewing Project Epics, I want to click the epic's `ec-*` tag badge directly to open the epic, eliminating redundant `Open Epic` buttons and cluttered spec tag lists.
   - *Independent Test*: Mount `DashboardView` and verify clicking `TagBadge` invokes `onOpenEpic`, while `Open Epic` button and spec tags are absent.
5. **US-05** (Priority: P2): As a developer working with autonomous coding agents, I want task implementation workflows in AICO skills to automatically commit and push verified changes across decoupled repositories, so that manual button clicking is not required.
   - *Independent Test*: Inspect `aico-implementation` and `aico-review-loop` skill definitions and verify automatic stage, commit, and push steps are mandated post-verification.
6. **US-06** (Priority: P1): As a developer committing updates to a codebase repository with submodules whose name matches the project, I want clicking Commit to target the actual codebase directory instead of the specs repository, so that modified submodules commit cleanly.
   - *Independent Test*: Issue a commit request for codebase repo matching project name; verify it commits in codebase directory.
7. **US-07** (Priority: P1): As a developer on the Home Dashboard, I want an interactive lineage graph showing all Epics and their associated nodes across the workspace, and I want all markdown formatting (`**bold**`, `` `code` ``, etc.) rendered cleanly across every view.
   - *Independent Test*: Mount `DashboardView`, `EpicHubView`, `FeatureViewer`, `SpecViewer`, `PlanViewer`, `GlossaryViewer`, and `AdrViewer` with markdown strings (`**bold**`, `` `code` ``); verify formatted elements (`<strong>`, `<code>`) render and `VisualLineageCanvas` is mounted on the Dashboard.

---

## 3. Functional Requirements (EARS)

### Ubiquitous Requirements
- **REQ-001**: The `DashboardView` component shall render the `Workspace Overview` header without a hardcoded filesystem root path (`d:/aico/aico-specs`).
- **REQ-002**: The `GitRepositoriesCard` component shall render all project repositories in a single unified 3-column responsive grid, placing the Specs repository (`type: 'specs'`) first with its distinct badge, followed by codebase and skills repositories without empty column gaps.
- **REQ-006**: The `DashboardView` component shall render each project epic using the reusable `<TagBadge id={epic.id} onClick={() => onOpenEpic(epic.id)} />` with `cursor-pointer` and hover feedback, omitting the redundant `Open Epic` button.
- **REQ-007**: The `App` component and `DashboardView` shall initialize project and epic states with empty arrays (`[]`), rendering clean empty/loading indicators (`0`, `-`, or loading placeholders) rather than flashing hardcoded mock projects (`AICO Core SDD`) or fallback epics (`ec-core`, `ec-specs`, `10 Formal Specs`).
- **REQ-009**: The `DashboardView` component shall omit spec `<TagBadge>` pills from the bottom of each Epic card in the `Project Epics` panel to reduce visual bloat.
- **REQ-010**: The `DashboardView` and `App` components shall render the interactive `VisualLineageCanvas` on the Home Dashboard displaying all project Epics and their associated nodes (`ADR -> Epic -> Feature -> Task -> Verification Seam / Code`), including an `Open Full Window` pop-out button.
- **REQ-011**: The frontend views (`DashboardView`, `EpicHubView`, `FeatureViewer`, `SpecViewer`, `PlanViewer`, `GlossaryViewer`, and `AdrViewer`) shall render markdown-formatted fields (epic executive summaries, ADR decisions/contexts/consequences, glossary definitions, scope boundaries, EARS requirement statements, seam descriptions, and task deliverables) using `<InlineMarkdown>` or `<MarkdownView>` so that inline syntax (`**bold**`, `*italic*`, `` `code` ``, links) is formatted as HTML rather than raw text.

### State-Driven Requirements
- **REQ-003**: The `GitRepositoriesCard` shall render state-aware Git action buttons for cloned repositories:
  - While `behind === 0`, the Pull button shall be disabled with tooltip `"Up to date (0 commits behind)"`.
  - While `behind > 0`, the Pull button shall be enabled with badge `Pull (${repo.behind})` and descriptive tooltip.
  - While `ahead === 0`, the Push button shall be disabled with tooltip `"Nothing to push (0 commits ahead)"`.
  - While `ahead > 0`, the Push button shall be enabled with badge `Push (${repo.ahead})` and descriptive tooltip.
  - While `repo.is_clean` is true, the Commit button shall be disabled with tooltip `"Working tree clean (nothing to commit)"`.
  - While `repo.is_clean` is false, the Commit button shall be enabled with badge `Commit (${modifiedCount})` and descriptive tooltip.
- **REQ-005**: While an autonomous agent executes implementation workflows in `aico-implementation`, `aico-review-loop`, or `aico-git`, the system shall automatically stage modified files, generate a conventional commit message, commit, and push across affected decoupled repositories upon verifying red tests turn green and artifacts conform.
- **REQ-008**: While resolving target repository directories for Git operations (`/pull`, `/push`, `/commit`, `/ai-commit`), the backend server shall search the active project's effective repositories list for matching `repo_id` before checking specs repository fallbacks, and shall accept optional explicit `path` parameters from the client.

### Event-Driven Requirements
- **REQ-004**: When a user clicks the enabled `Commit` button on a modified repository, the UI shall trigger an automated AI commit routine (auto-generating a conventional commit message from git status/diff and committing) with progress indicator, without opening an inline manual text input box.

---

## 4. Designated Public Test Seams

### Primary Seams:
1. **Dashboard Overview, Epics, Graph & Markdown Seam (`frontend/tests/DashboardView.test.tsx` & `frontend/tests/EpicHubView.test.tsx`)**:
   - Signature: `<DashboardView onOpenEpic={...} onOpenSpec={...} epics={...} />`
   - Verifies: Absence of hardcoded root path (`REQ-001`), interactive `TagBadge` epic navigation and omission of `Open Epic` button (`REQ-006`), zero-mock empty state (`REQ-007`), omission of spec tag pills on Epic cards (`REQ-009`), workspace-wide `VisualLineageCanvas` rendering all epics (`REQ-010`), and formatted markdown rendering (`REQ-011`).
2. **Repositories Grid & Git Telemetry Seam (`frontend/tests/DashboardGitStatus.test.tsx` & `frontend/src/features/dashboard/__tests__/GitRepositoriesCard.test.tsx`)**:
   - Signature: `<GitRepositoriesCard statuses={...} />`
   - Verifies: Re-unified single 3-column grid with Specs first (`REQ-002`), state-aware disabled and count states for Pull, Push, and Commit (`REQ-003`), one-click AI commit action (`REQ-004`), transmission of `path` telemetry (`REQ-008`).
3. **AI Commit Backend Endpoint Seam (`crates/aico-server/tests/server_test.rs`)**:
   - Signature: `POST /api/v1/git/ai-commit`
   - Verifies: Automated conventional commit message generation from git diff/status and execution (`REQ-004`).
4. **Agent Workflow Automation Seam (`codebase/aico-skills/aico-implementation/SKILL.md` & `codebase/aico-skills/aico-git/SKILL.md`)**:
   - Signature: Autonomous workflow verification and automated Git stage/commit/push gate (`REQ-005`).
5. **Codebase Repository Resolution Seam (`crates/aico-server/tests/server_test.rs`)**:
   - Signature: `resolve_repo_dir` when codebase repo ID matches project slug (`REQ-008`).

---

## 5. Open Clarifications

None. Ratified in [ADR-0004](../../../../adr/0004-react-frontend-and-reusable-design-system.md), [ADR-0005](../../../../adr/0005-unified-accordion-workspace-and-dockable-panels.md), and [ADR-0008](../../../../adr/0008-unified-root-discovery-and-project-manifest.md).
