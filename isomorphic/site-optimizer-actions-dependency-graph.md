# Site Optimizer Contributed Actions — Dependency Graph with Client-Coupling Classification

> **Goal:** Decouple site-optimizer contributed actions from client/browser concerns so they can run on the server (isomorphic).
>
> **Color key:**
> - 🟢 **GREEN** — Server-ready (no client dependency)
> - 🔴 **RED** — Definitely client-coupled (DOM, canvas, viewport, etc.)
> - 🟠 **ORANGE** — Needs splitting (some methods are server-OK, others are client-coupled)
> - 🔵 **BLUE** — Root action (entry point)

---

## 1. Overview

**Total Actions:** 17 (across 5 entry points)

| Entry Point | Repository | Actions |
|-------------|-----------|---------|
| siteOptimizerActionsContributorEntryPoint | odeditor-packages | ADD_COMPONENT, MIGRATION, PIN_TO_PAGE, UNPIN_FROM_PAGE |
| SiteOptimizerAnimationEntryPoint | odeditor-packages | SET_ENTRANCE_ANIMATION, SET_LOOP_ANIMATION, SET_MOUSE_EFFECT_ANIMATION, SET_SCROLL_ANIMATION |
| presetsSiteOptimizerActionEntryPoint | responsive-editor-packages | SET_PRESET |
| componentFlowsSiteOptimizerActionsEntryPoint | responsive-editor-packages | REMOVE_COMPONENT, UPDATE_PARENT, SET_STYLE, SET_DATA, REORDER |
| flexSiteOptimizerEntryPoint | responsive-editor-packages | SET_FLEX_CONTAINER_LAYOUT, SET_FLEX_ITEM_LAYOUT, SET_FLEX_GAPS |

---

## 2. Action → L1 API Adjacency List

Each action and the L1 APIs it directly depends on, color-coded.

| Action | L1 APIs |
|--------|---------|
| **ADD_COMPONENT** | DocumentServicesAPI 🟢, EditorFlowAPI 🟠, EditorPointerAPI 🟢, ComponentEditorAPI 🟠, ComponentRoutingAPI 🟢, PagesDataServiceAPI 🟠, ExperimentsAPI 🟢, AiComponentsContributionAPI 🟢, OdeditorLayoutBuilderAPI 🟠, AddPanelDataAPI 🟠, TemplatesCmsDataServiceAPI 🟢, LayoutConverterAPI 🟠, PinnedToContainerFlowsAPI 🟠 |
| **MIGRATION** | ComponentEditorAPI 🟠, PagesDataServiceAPI 🟠, LayoutConverterAPI 🟠, ExperimentsAPI 🟢 |
| **PIN_TO_PAGE** | PinnedToContainerFlowsAPI 🟠 |
| **UNPIN_FROM_PAGE** | DocumentServicesAPI 🟢, PinnedToContainerFlowsAPI 🟠 |
| **SET_ENTRANCE_ANIMATION** | EditorUIAPI 🔴, ComponentPointerFactoryAPI 🟢, AnimationFlowsPrivateAPI 🟢, ComponentReactionsAPI 🟢 |
| **SET_LOOP_ANIMATION** | EditorUIAPI 🔴, ComponentPointerFactoryAPI 🟢, AnimationFlowsPrivateAPI 🟢, ComponentReactionsAPI 🟢 |
| **SET_MOUSE_EFFECT_ANIMATION** | EditorUIAPI 🔴, ComponentPointerFactoryAPI 🟢, AnimationFlowsPrivateAPI 🟢, ComponentReactionsAPI 🟢 |
| **SET_SCROLL_ANIMATION** | EditorUIAPI 🔴, ComponentPointerFactoryAPI 🟢, AnimationFlowsPrivateAPI 🟢, ComponentReactionsAPI 🟢 |
| **SET_PRESET** | EditorPlatformAPI 🟠 |
| **REMOVE_COMPONENT** | ComponentFlowsSiteOptimizerActionsPrivateAPI 🔴 → ComponentFlowsAPI 🟢 (`removeComponent` full UI flow is RED; `unstable_removeComponent` is GREEN), ComponentHierarchyFlowsAPI 🟢 |
| **UPDATE_PARENT** | ComponentFlowsSiteOptimizerActionsPrivateAPI 🟢 → ComponentHierarchyFlowsAPI 🟢 |
| **SET_STYLE** | GenericStyleAPI 🟢, EditorPlatformAPI 🟠, ExperimentsAPI 🟢 |
| **SET_DATA** | EditorPlatformAPI 🟠 |
| **REORDER** | ComponentArrangementDerivativeStateAPI 🟢, ComponentArrangementFlowsAPI 🟢 |
| **SET_FLEX_CONTAINER_LAYOUT** | ComponentLayoutFlowsAPI 🟢, ComponentLayoutDerivativeStateAPI 🟠, StageContextBuilderAPI 🔴 |
| **SET_FLEX_ITEM_LAYOUT** | ComponentLayoutFlowsAPI 🟢, ComponentLayoutDerivativeStateAPI 🟠, StageContextBuilderAPI 🔴 |
| **SET_FLEX_GAPS** | ComponentLayoutFlowsAPI 🟢, ComponentLayoutDerivativeStateAPI 🟠, LayoutBuilderAPI 🟢, StageContextBuilderAPI 🔴 |

---

## 3. Per-API Dependency Table (L1 → L2 → L3)

Only APIs with notable sub-dependencies are expanded. De-duplicated.

### PinnedToContainerFlowsAPI 🟠 (28 L2 deps)

Used by: ADD_COMPONENT, PIN_TO_PAGE, UNPIN_FROM_PAGE

| L2 Dependency | Color | Notes |
|---------------|-------|-------|
| LayoutHeursticsDerivativeStateAPI | 🔴 | Measurement-based heuristics |
| ComponentTypeAPI | 🟢 | |
| HistoryAPI | 🟠 | Server-compatible implementation exists (no-op history); batchHistory() wrapping is safe |
| ComponentLayoutAPI_deprecated | 🔴 | Deprecated, must not exist on server |
| PinnedToContainerDerivativeStateAPI | 🔴 | → L3: LayoutHeuristicsAPI 🔴, PreviewAPI 🔴, EditorCacheAPI 🔴 — all deps are RED, no isomorphic subset |
| StageContextBuilderAPI | 🔴 | Canvas/stage is UI concept |
| ComponentMeasureAPI | 🔴 | DOM measurement (getBoundingBox) |
| PositionDerivativeStateAPI | 🟠 | `getPosition()` OK, `canSetStickyPosition()` may need measurements |
| PointerComparisonAPI | 🟢 | |
| ComponentEditorBIAPI | 🟠 | BI sending is server-OK; ambiguity is its `PreviewAPI` dependency |
| ExperimentalFlowsAPI | 🟢 | |
| ComponentHierarchyAPI | 🟢 | |
| ComponentHierarchyFlowsAPI | 🟢 | |
| ComponentLayoutFlowsAPI | 🟢 | |
| ComponentsLocatorAPI | 🔴 | `getCompRefsFromPoint(Point)` — viewport pixel coords |
| ComponentsDerivativeStateAPI | 🟢 | |
| ComponentArrangementDerivativeStateAPI | 🟢 | |
| ComponentArrangementFlowsAPI | 🟢 | |
| UnitOfWorkFlowAPI | 🟢 | |
| ComponentLayoutDerivativeStateAPI | 🟠 | *(reclassified)* — entry point declares StageContextBuilderAPI; `unstable_getCurrentEffectiveLayout` uses current stage context |
| ExtendedDocumentServicesAPI | 🟢 | |
| VariantsIteratorAPI | 🟢 | |
| LayoutBuilderAPI | 🟢 | |
| TransactionsAPI | 🟢 | |
| ComponentEditorAPI | 🟠 | |
| ComponentRoutingAPI | 🟢 | |
| PagesDataServiceAPI | 🟠 | |
| InteractionContextAPI | 🟠 | General analysis reclassified to GREEN (generic scope wrapper, DATA_SERVICE layer — see ADD_COMPONENT analysis); specific usage in PinnedToContainerFlowsAPI needs verification |
| EditorPointerAPI | 🟢 | |

### LayoutConverterAPI 🟠 (21 L2 deps)

Used by: ADD_COMPONENT (via migration path), MIGRATION

| L2 Dependency | Color | Notes |
|---------------|-------|-------|
| SuperGridDerivativeStateAPI | 🟢 | |
| ComponentVisibilityDerivativeStateAPI | 🟢 | |
| ComponentHierarchyAPI | 🟢 | |
| ComponentRoutingAPI | 🟢 | |
| DocumentServicesAPI | 🟢 | |
| SystemContainerDerivativeStateAPI | 🟢 | |
| ComponentLayoutFlowsAPI | 🟢 | |
| ContentMaxWidthFlowsAPI | 🟢 | |
| GenericStyleAPI | 🟢 | |
| SystemComponentsDerivativeStateAPI | 🟢 | |
| GlobalVariablesDerivativeStateAPI | 🟢 | |
| EditorPointerAPI | 🟢 | |
| SharedBlocksAPI | 🔴 | *(reclassified)* Uses `pagesDataServiceAPI.getFocusedPage/getCurrentPage()` (forbidden DS ops) + `previewAPI.memoizeForSiteUpdates()` (PreviewAPI = RED) |
| SuperGridCellDerivativeStateAPI | 🟢 | |
| ComponentTypeAPI | 🟢 | |
| ComponentFlowsAPI | 🟢 | |
| ComponentMeasureAPI | 🔴 | DOM measurement — blocks server migration |
| ResponsiveFedopsAPI | 🟢 | |
| ResponsiveBIAPI | 🟢 | |
| AutoGridUtilsAPI | 🟢 | |
| HeaderComponentsManagementAPI | 🟢 | |

### AnimationFlowsPrivateAPI 🟢 (9 L2 deps — all green)

Used by: SET_*_ANIMATION (x4)

| L2 Dependency | Color |
|---------------|-------|
| ComponentTriggersAPI | 🟢 |
| ComponentEffectsFlowsAPI | 🟢 |
| ComponentReactionsDerivativeStateAPI | 🟠 | *(reclassified)* — `getCurrentEffective()` calls `stageContextBuilderAPI.addCurrentContextToRef()`; animation flow uses only GREEN methods so AnimationFlowsPrivateAPI remains effectively GREEN for this use case |
| ComponentEffectsDerivativeStateAPI | 🟢 |
| VariantsIteratorAPI | 🟢 |
| ComponentReactionsAPI | 🟢 |
| PointerComparisonAPI | 🟢 |
| ComponentPointerBuilderAPI | 🟢 |
| ComponentEffectsAPI | 🟢 |

### EditorPlatformAPI 🟠 (2 L2 deps)

Used by: SET_PRESET, SET_STYLE, SET_DATA

| L2 Dependency | Color | Notes |
|---------------|-------|-------|
| EditorPlatformHostIntegrationAPI | 🟢 | Platform host integration |
| WorkerManager (via getWorkerManager()) | 🔴 | Workers are client concept |

**Method-level split:** `getStyle()` and `getData()` are 🟢 GREEN — they call `ComponentStyleAPI.getInBreakpoint()` and `ComponentDataAPI.get()` directly, bypassing WorkerManager entirely. Only the SDK mutation methods (`setStyle`, `setData`, `setPreset`) use the `WorkerManager → SDKHostContext` chain.

### PinnedToContainerDerivativeStateAPI 🟠 (L3 deps)

Reached via PinnedToContainerFlowsAPI

| L3 Dependency | Color | Notes |
|---------------|-------|-------|
| LayoutHeuristicsAPI | 🔴 | Heuristics based on DOM measurements |
| PreviewAPI | 🔴 | No preview rendering on server |
| EditorCacheAPI | 🔴 | Client-side editor caching |
| ComponentLayoutAPI_deprecated | 🔴 | Deprecated |

### PinnedToContainerEntryPoint (VERTICAL layer, L2)

| Dependency | Color | Notes |
|------------|-------|-------|
| MockComponentMeasureAPI | 🔴 | Mock of DOM measurement |
| ScrolledComponentsDerivativeStateAPI | 🔴 | Scrolling is client-only |

---

## 4. Full Unique API Registry

Master table of all ~67 unique APIs across the dependency graph, classified by client-coupling.

### 🔴 RED — Definitely Client-Coupled (17 APIs)

| # | API | Reason | Reached Via |
|---|-----|--------|-------------|
| 1 | **EditorUIAPI** | Editor UI — wraps operations in UI context, handles UI refresh | Animation actions (L1) |
| 2 | **ComponentMeasureAPI** | DOM Measurements — `getBoundingBox` | PinnedToContainerFlowsAPI (L2), LayoutConverterAPI (L2) |
| 3 | **StageContextBuilderAPI** | Stage/Preview Logic — no stage on server | Flex actions (L1), PinnedToContainerFlowsAPI (L2) |
| 4 | **ComponentLayoutAPI_deprecated** | Deprecated — must not exist on server | PinnedToContainerFlowsAPI (L2), PinnedToContainerDerivativeStateAPI (L3) |
| 5 | **PreviewAPI** | Stage/Preview Logic — no preview rendering on server | PinnedToContainerDerivativeStateAPI (L3) |
| 6 | **ComponentsLocatorAPI** | DOM Measurements — `getCompRefsFromPoint(Point)` uses viewport pixel coords | PinnedToContainerFlowsAPI (L2) |
| 7 | **LayoutHeuristicsAPI** | DOM Measurements — heuristics based on DOM measurements | PinnedToContainerDerivativeStateAPI (L3) |
| 8 | **LayoutHeursticsDerivativeStateAPI** | DOM Measurements — derived from measurement-based heuristics | PinnedToContainerFlowsAPI (L2) |
| 9 | **EditorCacheAPI** | Editor State — client-side editor caching | PinnedToContainerDerivativeStateAPI (L3) |
| 10 | **MockComponentMeasureAPI** | DOM Measurements — mock of DOM measurement API | PinnedToContainerEntryPoint (VERTICAL layer) |
| 11 | **ScrolledComponentsDerivativeStateAPI** | Scroll — scrolling is client-only | PinnedToContainerEntryPoint (VERTICAL layer) |
| 12 | **EditorFlowAPI** | *(reclassified from 🟠)* — deprecated in favour of `EditorUIAPI` (RED); includes UI notification deferral | ADD_COMPONENT, MIGRATION (L1) |
| 13 | **PinnedToContainerDerivativeStateAPI** | *(reclassified from 🟠)* — depends on `PreviewAPI` + `LayoutHeuristicsAPI` + `EditorCacheAPI` (all RED root causes); no isomorphic subset | PinnedToContainerFlowsAPI (L2) |
| 14 | **SharedBlocksAPI** | *(reclassified from 🟢)* — uses `pagesDataServiceAPI.getFocusedPage/getCurrentPage()` (forbidden DS ops) + `previewAPI.memoizeForSiteUpdates()` | LayoutConverterAPI (L2) |
| 15 | **OdeditorLayoutBuilderAPI** | *(reclassified from 🟠)* — depends on `StageContextBuilderAPI` (Stage/Preview); confirmed RED in implementation analysis | ADD_COMPONENT getComponentStructure() (L1) |
| 16 | **ComponentFlowsSiteOptimizerActionsPrivateAPI** | *(reclassified from 🟢)* — calls full UI flow `ComponentFlowsAPI.removeComponent()` which includes confirmation dialogs, selection clearing, render triggers | REMOVE_COMPONENT site-opt action (L1) |

### 🟠 ORANGE — Needs Splitting / Partial Client Coupling (11 APIs)

> EditorFlowAPI, PinnedToContainerDerivativeStateAPI, SharedBlocksAPI, OdeditorLayoutBuilderAPI, and ComponentFlowsSiteOptimizerActionsPrivateAPI have been reclassified to RED (see above). HistoryAPI has been reclassified back to 🟠 ORANGE (server-compatible implementation exists). InteractionContextAPI moved from RED to ORANGE (confirmed GREEN in addComponent context; usage in PinnedToContainerFlowsAPI still needs verification). ComponentLayoutDerivativeStateAPI and ComponentReactionsDerivativeStateAPI reclassified from GREEN to ORANGE (both have StageContextBuilderAPI in entry point deps).

| # | API | Server-OK Methods | Client-Coupled Methods | Notes |
|---|-----|-------------------|----------------------|-------|
| 1 | **PagesDataServiceAPI** | `isPage()`, `getPageData()` | `doesPageExist()` and `getPageIdList()` — **forbidden DS ops** (`pages.doesPageExist`, `pages.getPageIdList`); `getFocusedPageId()`, `getCurrentPageId()`, `getCurrentPage()`, `getFocusedPage()` — forbidden + Editor State | Split to PagesClientDataServiceAPI; server-safe surface is smaller than assumed |
| 2 | **ComponentEditorAPI** | `hasSectionBehaviors()`, `getAddComponentStrategy()` | Extends `UiBehaviorsStageAPI`, `UiBehaviorsWorkspaceAPI` — **Stage/Preview + Editor UI** (both RED categories) | Large API needing significant split |
| 3 | **PositionDerivativeStateAPI** | `getPosition()`, `isStickyPosition()` — data-based | `canSetStickyPosition()` may depend on measurements | Verify implementation deps |
| 4 | **ComponentEditorBIAPI** | All BI sending (BI is server-OK per classification rules) | Depends on `PreviewAPI` (RED root cause) — the dep, not the BI itself, is the problem | Fix: remove PreviewAPI dep, BI sending is fine on server |
| 5 | **AddPanelDataAPI** | `getData()` reads in-memory state (no browser deps in getData itself); DERIVATIVE_STATE layer | Entry point declares `PanelsAPI` (client-only) — prevents loading on server; state populated from client-side CMS fetch so `getData()` returns null on server | In ADD_COMPONENT, null return falls through to OdeditorLayoutBuilderAPI (RED); if AI contribution covers all component types, this path is never reached |
| 6 | **HistoryAPI** | `add()`, `canUndo()`, `current()` methods map to `ds.history.*` — history DS ops available on server | Depends on `PreviewDisplayAPI` in the entry point; specific server path needs verification | ORANGE pending full verification; keep as-is |
| 7 | **InteractionContextAPI** | *(reclassified from 🔴)* — confirmed GREEN in addComponent flow (generic scope wrapper, DATA_SERVICE layer) | Usage in PinnedToContainerFlowsAPI may differ — needs verification | If PinnedToContainerFlowsAPI usage is the same `runInContext()` pattern, fully GREEN |
| 8 | **EditorPlatformAPI** | SDK host concept | `getWorkerManager()` — browser Web Workers are client-only | Need server-side SDK host |
| 9 | **PinnedToContainerFlowsAPI** | Pin/unpin concept is isomorphic | Depends on 10+ RED APIs (ComponentMeasureAPI, StageContextBuilderAPI, etc.) | Current impl is RED; migration target |
| 10 | **LayoutConverterAPI** | Migration logic concept is isomorphic | Depends on `ComponentMeasureAPI` + `SharedBlocksAPI` (both RED) | Current impl is RED; migration target |
| 11 | **ComponentLayoutDerivativeStateAPI** | *(reclassified from 🟢)* — `getScopedLayout()`, `getEffectiveLayout()` take context-qualified pointers (likely server-safe); `unstable_getCurrentEffectiveLayout(comp)` reads current stage context | Entry point declares `StageContextBuilderAPI`; the "current" method is the blocker | Fix: stub `addCurrentContextToRef()` as identity on server → fully GREEN |
| 12 | **ComponentReactionsDerivativeStateAPI** | *(reclassified from 🟢)* — `getCurrentEffective()` calls `stageContextBuilderAPI.addCurrentContextToRef()`; all other reaction/effect methods are GREEN | Used in AnimationFlowsPrivateAPI via GREEN methods only, so animation flow is not effectively blocked | Fix: same `StageContextBuilderAPI` identity stub → fully GREEN |

### 🟢 GREEN — Server-Ready (42 APIs)

| # | API | Notes |
|---|-----|-------|
| 1 | DocumentServicesAPI | Core document manipulation |
| 2 | EditorPointerAPI | Pointer utilities, data-based |
| 3 | SiteOptimizerContributionAPI | Registration framework |
| 4 | ExperimentsAPI | Feature flags |
| 5 | AiComponentsContributionAPI | Component structures |
| 6 | TemplatesCmsDataServiceAPI | Template resolution |
| 7 | ComponentRoutingAPI | Routing/CSS override targets |
| 8 | ComponentPointerFactoryAPI | Pointer creation |
| 9 | ComponentFlowsAPI | Component CRUD flows (`unstable_removeComponent` is GREEN; `removeComponent` full UI flow is not) |
| 10 | ComponentHierarchyFlowsAPI | Parent/child mutations |
| 11 | ComponentHierarchyAPI | Parent/child navigation |
| 12 | GenericStyleAPI | Style updates |
| 13 | ComponentArrangementDerivativeStateAPI | Z-order queries |
| 14 | ComponentArrangementFlowsAPI | Z-order mutations |
| 15 | ComponentLayoutFlowsAPI | Layout mutations |
| 16 | LayoutBuilderAPI | Layout construction |
| 17 | ComponentReactionsAPI | Reactions data |
| 18 | AnimationFlowsPrivateAPI | Animation logic (all L2 deps are green for animation use case; see ComponentReactionsDerivativeStateAPI note in ORANGE) |
| 19 | ComponentTriggersAPI | Trigger types |
| 20 | ComponentEffectsFlowsAPI | Effects mutations |
| 21 | ComponentEffectsDerivativeStateAPI | Effects state |
| 22 | ComponentPointerBuilderAPI | Pointer building |
| 23 | ComponentEffectsAPI | Effects CRUD |
| 24 | VariantsIteratorAPI | Breakpoint iteration |
| 25 | PointerComparisonAPI | Pointer equality |
| 26 | ExperimentalFlowsAPI | Experimental features |
| 27 | ComponentsDerivativeStateAPI | Component state queries |
| 28 | ComponentTypeAPI | Component type metadata |
| 29 | TransactionsAPI | Transaction management |
| 30 | ExtendedDocumentServicesAPI | Extended DS utilities |
| 31 | UnitOfWorkFlowAPI | Atomic operations |
| 32 | SystemContainerDerivativeStateAPI | System container state |
| 33 | ComponentVisibilityDerivativeStateAPI | Visibility state |
| 34 | SuperGridDerivativeStateAPI | Grid state |
| 35 | ContentMaxWidthFlowsAPI | Content width flows |
| 36 | SystemComponentsDerivativeStateAPI | System component state |
| 37 | GlobalVariablesDerivativeStateAPI | Global variables |
| 38 | SuperGridCellDerivativeStateAPI | Grid cell state |
| 39 | AutoGridUtilsAPI | Grid utilities |
| 40 | HeaderComponentsManagementAPI | Header management |
| 41 | EditorPlatformHostIntegrationAPI | Platform host integration |
| 42 | ResponsiveFedopsAPI | Fedops logging |
| 43 | ResponsiveBIAPI | BI reporting |

---

## 5. Server-Migration Effort by Entry Point

### Group 1: siteOptimizerActionsContributorEntryPoint (ADD_COMPONENT, MIGRATION, PIN_TO_PAGE, UNPIN_FROM_PAGE)

**Verdict: HARD** — Most complex group (~40+ total APIs). The key blockers are:
- **PinnedToContainerFlowsAPI** 🟠 — depends on 10+ RED APIs (ComponentMeasureAPI, StageContextBuilderAPI, ComponentsLocatorAPI, InteractionContextAPI, etc.)
- **LayoutConverterAPI** 🟠 — depends on ComponentMeasureAPI 🔴
- **PagesDataServiceAPI** 🟠 — `getFocusedPageId()` / `getCurrentPageId()` are client-only
- **EditorFlowAPI** 🟠 — deprecated, wraps EditorUIAPI 🔴
- **ComponentEditorAPI** 🟠 — extends UI behaviors

**Recommended approach:**
1. Create server-side `PinnedToContainerFlowsAPI` that takes layout data as input instead of measuring DOM
2. Create `PagesServerDataServiceAPI` with only data methods (no "focused"/"current" page)
3. Replace `EditorFlowAPI.run()` with `TransactionsAPI` directly
4. Extract `ComponentEditorAPI` data methods into a new server-safe API

### Group 2: SiteOptimizerAnimationEntryPoint (SET_*_ANIMATION x4)

**Verdict: EASY (one blocker)** — ~15 APIs, all L2 deps are green. **Only blocker:**
- **EditorUIAPI** 🔴 — wraps animation operations in UI context

**Recommended approach:**
1. Remove the `EditorUIAPI.run()` wrapper — run animation flows directly or use a server-compatible transaction wrapper

### Group 3: presetsSiteOptimizerActionEntryPoint (SET_PRESET)

**Verdict: MEDIUM** — Only 1 L1 API, but:
- **EditorPlatformAPI** 🟠 — `getWorkerManager()` uses client-side workers

**Recommended approach:**
1. Create a server-side SDK host that doesn't require a WorkerManager
2. Call `@wix/editor` element module directly with server context

### Group 4: componentFlowsSiteOptimizerActionsEntryPoint (REMOVE, UPDATE_PARENT, SET_STYLE, SET_DATA, REORDER)

**Verdict: MOSTLY EASY** — ~10 APIs, mostly green. Blockers:
- **EditorPlatformAPI** 🟠 (SET_STYLE, SET_DATA) — same worker issue as SET_PRESET
- SET_STYLE has two paths: GenericStyleAPI 🟢 (legacy) and SDK path 🟠

**Recommended approach:**
1. UPDATE_PARENT, REORDER → already server-ready (all green)
2. REMOVE_COMPONENT → **RED** — `ComponentFlowsSiteOptimizerActionsPrivateAPI.removeComponent()` delegates to the full UI flow `ComponentFlowsAPI.removeComponent()`. Fix: call `ComponentHierarchyFlowsAPI.unstable_removeComponent()` directly (one-line change in the private API)
3. SET_STYLE → use GenericStyleAPI path directly (skip SDK/experiment toggle)
4. SET_DATA → create server-side SDK host

### Group 5: flexSiteOptimizerEntryPoint (SET_FLEX_*_LAYOUT, SET_FLEX_GAPS)

**Verdict: EASY (one blocker)** — ~5 APIs. **Only blocker:**
- **StageContextBuilderAPI** 🔴 — canvas/stage concept

**Recommended approach:**
1. Remove or replace `StageContextBuilderAPI` dependency — flex layout operations shouldn't need canvas context
2. Pass layout dimensions as data inputs instead of reading from stage

---

## 6. Architecture Notes

### Repluggable Pattern
All APIs are accessed via `shell.getAPI(SlotKey)` dependency injection. This is actually helpful for server migration — we can provide server-compatible API implementations via the same DI framework.

### Private API Pattern
Complex actions use private APIs that encapsulate implementation details:
- `AnimationFlowsPrivateAPI` — all deps green, easiest to migrate
- `ComponentFlowsSiteOptimizerActionsPrivateAPI` — all deps green
- `FlexSiteOptimizerPrivateAPI` — has StageContextBuilderAPI 🔴 dep

### SDK Integration Pattern
Newer actions (SET_PRESET, SET_STYLE, SET_DATA) use the Wix SDK:
```
EditorPlatformAPI → WorkerManager → SDKHostContext → @wix/sdk client
```
For server, this needs to be replaced with a server-side SDK host context.
