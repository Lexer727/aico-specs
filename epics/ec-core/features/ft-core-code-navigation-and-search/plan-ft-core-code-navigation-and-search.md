---
id: plan-ft-core-code-navigation-and-search
type: plan
aico_skills_version: "0.1.2"
epic: "ec-core"
order: 13
title: "Code Navigation, Inline Annotations & Search Integration Plan"
status: "Completed"
repositories: ["aico", "aico-specs"]
derives_from: ["spec-ft-core-code-navigation-and-search"]
created_at: "2026-10-17T17:40:00Z"
---

# Implementation Plan: Code Navigation, Inline Annotations & Search Integration

## 1. Technical Architecture & Decisions

- **Single Clickable Feature Pill in `CodeViewer.tsx` (`REQ-001`, `REQ-002`, `REQ-003`)**:
  - In `CodeViewer.tsx`, parse in-band 3-segment annotations:
    ```regex
    ^\s*(?:\/\/|#|\/\*|\{\/\*)\s*@(implements|verifies)\s+(ft-[a-z0-9\-]+)::(TASK-[A-Za-z0-9\-]+)::(REQ-[0-9]{3,4})
    ```
  - Consolidate the 3 inline badges (`[feature]`, `[task]`, `[req]`) into a single clickable feature pill:
    ```tsx
    <button
      type="button"
      onClick={() => onSelectFeature?.(meta.feature)}
      className="inline-flex items-center gap-1 px-1.5 py-0.5 rounded text-[11px] font-mono bg-violet-500/20 text-violet-300 border border-violet-500/30 hover:bg-violet-500/30 transition-colors cursor-pointer"
      title={`Feature: ${meta.feature} | Task: ${meta.task} | Req: ${meta.req}`}
    >
      <Tag className="w-3 h-3 text-violet-400" />
      <span>{meta.feature}</span>
    </button>
    ```
  - Extend `CodeViewerProps` with `onSelectFeature?: (featureId: string) => void`.

- **Sidebar Full-Text Code Search in `ShellLayout.tsx` & `searchEngine.ts` (`REQ-004`, `REQ-005`)**:
  - In `ShellLayout.tsx`, when the search query is non-empty, issue a debounced query (150ms) to `/api/v1/search?q=${encodeURIComponent(query)}`.
  - Filter backend search hits where `hit.scope === 'code'` and merge their relative file paths into `filteredCode`.
  - In `searchEngine.ts`, update `createSearchIndex` to index code file annotation targets, task tokens, and path fragments for instant client-side matching.

- **FeatureViewer Code Implementations Section & Task-Level Code Links (`REQ-006`, `REQ-007`)**:
  - Extend `FeatureViewerProps` with `codeFiles?: CodeFileInfo[]` and `onSelectCode?: (path: string) => void`.
  - Add a collapsible "Code Implementations" card positioned immediately above the Assets section in `FeatureViewer.tsx`, displaying all files where `file.annotations.some(a => a.target.startsWith(feature.id) || a.target.startsWith('ft-' + feature.id))`.
  - In the Task View of `FeatureViewer.tsx`, match each task card against `codeFiles` matching `::${task.id}::` and render clickable code badge links calling `onSelectCode(file.path)`.

- **EpicHubView Aggregated Code Files Roll-Up (`REQ-008`)**:
  - Extend `EpicHubViewProps` with `codeFiles?: CodeFileInfo[]` and `onSelectCode?: (path: string) => void`.
  - Aggregate child feature IDs for the active epic (`epic.features.map(f => f.id)`), filter `codeFiles` with matching annotations, and render a dedicated "Code Files" section with clickable navigation links.

- **App-Level Integration & Regression (`REQ-001`, `REQ-003`, `REQ-006`, `REQ-007`, `REQ-008`)**:
  - In `App.tsx`, wire `onSelectFeature={handleOpenFeature}` to `<CodeViewer />`.
  - In `App.tsx`, wire `codeFiles={codeFiles}` and `onSelectCode={handleOpenCode}` to `<FeatureViewer />` and `<EpicHubView />`.
  - Execute full Vitest frontend suite, Rust tests, and blueprint verification.

---

## 2. Vertical Task Directed Acyclic Graph (DAG)

```
TASK-CORE-013-1 (CodeViewer Single Feature Pill & Navigation)
  ├── TASK-CORE-013-3 (FeatureViewer Code Implementations & Task Links)
  └── TASK-CORE-013-4 (EpicHubView Aggregated Code Files Section)
TASK-CORE-013-2 (Sidebar Full-Text Code Search in ShellLayout & searchEngine)
  └── TASK-CORE-013-5 (App Wiring & Full Suite Regression)
```

### Phase 1: CodeViewer Single Feature Pill & Navigation

- **TASK-CORE-013-1**: Consolidate `CodeViewer` annotation badges into a single clickable feature pill with tooltip metadata and `onSelectFeature` callback
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-002, REQ-003
  - **Blocked by**: None
  - **Verification Seam**: `frontend/tests/CodeViewer.test.tsx`
  - **What it delivers**:
    - Updates `CodeViewer.tsx` to replace multiple inline badges with a single `[ft-<feature>]` button badge.
    - Adds `title` tooltip displaying full feature, task, and requirement metadata.
    - Adds `onSelectFeature?: (featureId: string) => void` prop to `CodeViewerProps` and invokes it upon clicking the pill.
    - Updates `frontend/tests/CodeViewer.test.tsx` to verify single pill rendering, tooltip contents, and callback invocation.

### Phase 2: Sidebar Tree Full-Text Code Search

- **TASK-CORE-013-2**: Integrate full-text content search and annotation indexing into sidebar Code tree view
  - **Status**: Done
  - **Addresses**: REQ-004, REQ-005
  - **Blocked by**: None
  - **Verification Seam**: `frontend/tests/ShellLayoutSearch.test.tsx`
  - **What it delivers**:
    - Enhances `searchEngine.ts` to index code file annotations, target identifiers, and path components.
    - Updates `ShellLayout.tsx` to query `/api/v1/search?q=...` with debouncing and merge full-text code hits into `filteredCode`.
    - Updates `frontend/tests/ShellLayoutSearch.test.tsx` to verify sidebar Code tree filtering when searching code content.

### Phase 3: FeatureViewer & EpicHubView Code Links

- **TASK-CORE-013-3**: Implement "Code Implementations" section above Assets and task-level code links in `FeatureViewer`
  - **Status**: Done
  - **Addresses**: REQ-006, REQ-007
  - **Blocked by**: TASK-CORE-013-1
  - **Verification Seam**: `frontend/tests/FeatureViewer.test.tsx`
  - **What it delivers**:
    - Adds `codeFiles` and `onSelectCode` props to `FeatureViewerProps`.
    - Renders a collapsible "Code Implementations" section above Assets in `FeatureViewer.tsx`.
    - Renders clickable in-band code links inside individual task cards in the Task View.
    - Updates `frontend/tests/FeatureViewer.test.tsx` to verify code section and task card links.

- **TASK-CORE-013-4**: Implement aggregated Code Files section in `EpicHubView`
  - **Status**: Done
  - **Addresses**: REQ-008
  - **Blocked by**: TASK-CORE-013-1
  - **Verification Seam**: `frontend/tests/EpicHubView.test.tsx`
  - **What it delivers**:
    - Adds `codeFiles` and `onSelectCode` props to `EpicHubViewProps`.
    - Aggregates codebase files implementing any child feature of the active epic and renders a "Code Files" section in `EpicHubView.tsx`.
    - Updates `frontend/tests/EpicHubView.test.tsx` to verify epic code files roll-up and navigation callback.

### Phase 4: App Wiring & Regression Suite

- **TASK-CORE-013-5**: Wire navigation callbacks across `App.tsx` and run regression suite
  - **Status**: Done
  - **Addresses**: REQ-001, REQ-003, REQ-006, REQ-007, REQ-008
  - **Blocked by**: TASK-CORE-013-2, TASK-CORE-013-3, TASK-CORE-013-4
  - **Verification Seam**: `frontend/tests/ShellLayoutSearch.test.tsx`
  - **What it delivers**:
    - Connects `onSelectFeature` in `CodeViewer` and `codeFiles` / `onSelectCode` in `FeatureViewer` and `EpicHubView` inside `App.tsx`.
    - Executes full Vitest frontend suite, Rust tests, and `verify-artifacts.mjs`.
