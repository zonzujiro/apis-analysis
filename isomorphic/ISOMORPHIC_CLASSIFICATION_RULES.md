# Isomorphic Classification Rules

Reference for determining whether an API can be isomorphic (runnable on the server)
or is client-only. Used during research and migration planning.

An API is **isomorphic** if it can run on the server without DOM, browser APIs, or UI.
An API is **client-only** if it depends on any of those, directly or transitively.

---

## Always Client-Only (Red)

These categories are always client-only, regardless of layer or usage.

### DOM Measurements
APIs that read or compute layout/size/position from the browser DOM.

Examples: `ComponentMeasureAPI`, `MockComponentMeasureAPI`, `ComponentMeasureDerivativeStateAPI`,
`ComponentBoundingBoxAPI`, `ScrollBarMeasureAPI`, `ClientDimensionsAPI`

### Editor UI
APIs that render or manage visible editor chrome — panels, dialogs, selection handles,
hover state, tooltips, etc.

Examples: `PanelsAPI`, `ComponentInteractionAPI`, `ComponentSelectFlowsAPI`,
`LightboxRemoveConfirmationAPI`

### Editor State
APIs that track the editor's runtime UI state — what is selected, what is open,
what mode the editor is in from the user's perspective.

Examples: `ComponentEditorStateAPI`

### Stage / Preview Logic
APIs tied to the site preview iframe or stage rendering context.

Examples: `StageContextBuilderAPI`, `PreviewAPI`, `PreviewDisplayAPI`, `PreviewZoomAPI`,
`PreviewMouseActionContributionAPI`, `PreviewEventsPluginAPI`, `PreviewModeFlowsAPI`,
`PreviewContextMenuAPI`, `ComponentPreviewAPI`

### Scroll
APIs that track or manipulate DOM scroll position.

Examples: `ScrolledComponentsDerivativeStateAPI`, `ScrolledComponentsFlowsAPI`,
`ScrolledComponentsStoreAPI`

### User Input (Mouse / Keyboard / Drag)
APIs that handle browser input events — clicks, drags, keypresses.

Examples: `SelectionModelAPI`, `SelectionModelConfigurationAPI`,
`ContinuousKeyboardActionAPI`, `ContinuesActionAcceptorWrapperAPI`,
`ComponentDragFlowsAPI`, `DragMouseActionAcceptorAPI`

### DOM Order
APIs that depend on or manipulate the actual DOM node ordering.

Examples: `DOMOrderFlowsAPI`, `DOMSortingAPI`, `AutoDOMOrderFlowsAPI`

### Animations
APIs that drive CSS or DOM animations.

Examples: `StageAnimationsFlowsAPI`, `ResizeTransitionAPI`

### Stage Indicators
All visual indicator APIs — they render overlays in the browser DOM.

Examples: `ViewportBoxIndicatorAPI`, `FocusBoxIndicatorAPI`, `ResizeIndicatorAPI`,
`EditBoxIndicatorAPI`, `DragBoxIndicatorAPI`, `HaloIndicatorAPI`,
`HighlightBoxIndicatorAPI`, `HoverBoxIndicatorAPI`, `SelectionBoxIndicatorAPI`,
`RotationIndicatorAPI`, `SpacingIndicatorAPI`, `SnapIndicatorAPI`,
`IndicatorRegistryAPI`, and all other `*IndicatorAPI`

### Browser / Viewport Globals
APIs wrapping `window`, `document`, or viewport state. Also any entry point that
directly references `window.*` or `document.*`.

Examples: `BrowserAPI`, `BlockingLayerAPI` — and any entrypoint using these globals directly

### Render Triggers
Any API whose primary purpose is to trigger a browser repaint or re-render.

Examples: `TransactionsAPI.render()`

---

## Not Client-Only (can be used on server)

### BI / Analytics APIs
BI events can be sent from the server. Do not assume BI APIs are client-only.

Examples: `ComponentEditorBIAPI`, and BI-related APIs in general.

---

## Settled Experiments

Some experiments are permanently open or permanently closed. When tracing code paths,
**skip branches gated behind these experiments** — they are effectively dead or guaranteed code.

### `specs.responsive-editor.addFlowNoStructureManipulation` — always TRUE
100% open since 2024. Treat the `true` branch as the only live path.

**Implication:** `componentAddFlowsAPI.prepareStructureToAdd` is dead code in this path.
It uses measurements and other browser operations — but since the experiment is always true,
those calls are never reached. This experiment can be merged to permanently remove that method.

### `specs.responsive-editor.validateIsContainableBeforeAdd` — always FALSE
100% false. Treat the `false` branch as the only live path.

**Implication:** `isContainable` is never called in the add flow. It depends on
`traverseDataServiceAPI.OldBadPerformance.getParent`, which can be ignored when
analyzing the add component flow. Merging this experiment removes the `isContainable` call.

### Experimental flow — always ON
When code branches on `experimentalFlowsAPI.isInExperimentalFlow()`, treat the experimental
(true) branch as the only live path. The legacy path is dead.

### Add component flow — parameter assumptions
In the add component flow, the following parameters should always be defined:
`compMeasure`, `containerRef`, `containerMeasure`.

**Implication:** Code paths that fall back to measuring these values (because they
are undefined) are dead paths. When analyzing for isomorphic compatibility, skip
those fallback branches — measurements are not needed if the caller always provides them.

### `shouldConfirmAdd` — always FALSE (except site creation)
The add confirmation dialog is skipped in almost all cases.

**Implication:** The dialog path is effectively dead for normal flows.
Note: the site creation flow is the exception — it uses this flag to avoid notifications.
That case needs to be extracted to a contribution rather than hardcoded.

---

## Ambiguous — Needs Case-by-Case Analysis

These are not automatically red or green. Check the specific implementation and
any contributed hooks/slots.

### Hook / Slot Systems
Hooks and contribution slots are only as isomorphic as their contributors.
A slot itself may be neutral, but if any contributed implementation touches
a client-only API, the slot becomes client-only for that package.

Pattern to look for: `someSlot.getItems()`, `contributeXxx()`, `hooks.structure.*`

### History / Transaction Orchestration
Flow orchestration APIs (e.g. `EditorFlowAPI.run()`) may wrap isomorphic operations
but also manage undo stacks and state that could depend on client context.

`HistoryAPI` has a server-compatible implementation — history recording is a no-op on the
server but the API is injectable and calls like `batchHistory()` do not block server execution.
Treat HistoryAPI as **ORANGE** (ambiguous), not RED. Verify the specific usage — e.g.
`batchHistory()` wrapping a data mutation is fine; anything that depends on undo stack state
being present needs case-by-case review.

### Page / Navigation Flows
Page removal, navigation, and routing flows often involve UI (confirmation dialogs,
URL changes) but may have isomorphic subsets.

---

## Always Isomorphic (Green)

### Document Model Access
APIs that read or write the document data structure without touching the DOM.

Examples: `DocumentServicesAPI.components.*`, `ComponentDataAPI`, `LayoutAPI`,
`DocumentStructureAPI`, `ComponentHierarchyAPI`

### Pointer / Ref Utilities
Pure pointer resolution and comparison — no side effects, no DOM.

Examples: `ComponentRoutingAPI`, `PointerComparisonAPI`, `EditorPointerAPI`

### Draft / Mode Flags
Simple state flags — no DOM, no UI.

Examples: `ComponentDraftDataModeAPI.ifOnDraftMode()`

### Experiment / Feature Flags
Boolean flag reads — always isomorphic.

Examples: `ExperimentsAPI`, `ExperimentalFlowsAPI`

---

## Layer-Based Heuristics

Use these as starting points, not final answers. Owner decision always takes precedence
(see Core Rules).

| Layer | Heuristic |
|---|---|
| SITE | Usually isomorphic — foundational document model |
| EDITOR_INFRA | Usually isomorphic — but watch for preview/iframe deps |
| DATA_SERVICE | Often isomorphic — verify measurement and preview deps |
| DERIVATIVE_STATE | Mixed — measurement-heavy APIs are client-only |
| FLOWS | Mixed — orchestration may include UI steps |
| UI | Always client-only |
| VERTICAL | Always client-only |

---

## Known Root Causes (from conflict analysis)

These APIs appear as dependencies in the majority of isomorphic conflicts.
If the API you are researching depends on any of these, it is likely client-only.

| API | Category | Conflict count |
|---|---|---|
| `PreviewAPI` | Stage/Preview | 161 refs (largest blocker) |
| `PreviewDisplayAPI` | Stage/Preview | part of 161 preview refs |
| `ComponentMeasureAPI` | DOM Measurements | 102 refs |
| `ClientDimensionsAPI` | Browser/Viewport | part of browser refs |
| `BrowserAPI` | Browser/Viewport | 13 refs |
| `ScrolledComponentsStoreAPI` | Scroll | part of 17 scroll refs |

Most conflicts cascade from just these root APIs. Resolving their abstractions
(isomorphic interface + client/server split) would eliminate the majority of
downstream conflicts.

---

## Forbidden DocumentServices Operations

If an API's implementation calls any of the following `documentServicesAPI` (or
`extendedDocumentServicesAPI`) methods — directly or transitively — it is **RED**,
regardless of its own dep chain. These operations are not available on the server.

### Runtime / DOM measurements
- `responsiveLayout.runtime.measure.getBoundingBox`
- `responsiveLayout.runtime.measure.getGridMeasures`
- `responsiveLayout.runtime.measure.getPadding`
- `components.responsiveLayout.getRuntimeProperty`

### Runtime component state (viewer instance)
- `components.runtime.getProps`
- `components.runtime.updateProps`
- `components.runtime.removeProps`
- `components.runtime.resolveManifestOverrides`
- `platform.controllers.getStageData`

### Scroll / animation
- `site.animateScroll`
- `site.setScroll`
- `site.getScroll`
- `components.scroll.isScrollable`
- `site.isWithStaticSharedPartsInflation`
- `components.isRenderedOnSite`

### Editor UI state (focused/current page, view mode)
- `pages.getFocusedPage`
- `pages.getFocusedPageId`
- `pages.getCurrentPage`
- `pages.getCurrentPageId`
- `pages.popupPages.isPopupOpened`
- `viewMode.get`
- `documentMode.getComponentViewMode`
- `documentMode.setExtraSiteHeight`
- `documentMode.getExtraSiteHeight`

### History / save (require full editor context)
- `history.add`
- `history.canUndo`
- `history.canRedo`
- `history.getUndoLastSnapshotId`
- `history.getUndoLastSnapshotParams`
- `initAutosave`
- `save`
- `canAutosave`
- `canUserPublish`

### Page data not available on server
- `pages.doesPageExist`
- `pages.getPageIdList`
- `pages.getRootNavigationInfo`
- `site.getWidth`
- `site.getHeight`

### Site / publishing state not available on server
- `siteDisplayName.get`
- `generalInfo.getPublicUrl`
- `generalInfo.isSitePublished`
- `generalInfo.isDraft`
- `generalInfo.getRevision`
- `generalInfo.isFirstSave`
- `premiumFeatures.get`
- `platform.getAppDataByApplicationId`
- `platform.getAppDataByAppDefId`
- `platform.notifyAppsOnCustomEvent`
- `builder.components.getCompAppData`

### Deprecated traversal (never server-safe)
- `deprecatedOldBadPerformanceApis.components.getContainer`
- `deprecatedOldBadPerformanceApis.components.getAncestors`
- `deprecatedOldBadPerformanceApis.components.getChildren`
- `deprecatedOldBadPerformanceApis.components.get.byType`

### Component metadata (render-aware or deprecated)
- `components.is.resizeOnlyProportionally`
- `components.is.resizable`
- `components.is.rotatable`
- `components.serialize`
- `components.skin.get`
- `components.scopes.extractScopeFromPointer`
- `components.scopes.getRootComponent`
- `components.scopes.getDefinedScopes`

**Note:** `HistoryAPI` (the Repluggable API) has a server-compatible no-op implementation and
is **not** blocked by this rule. Only the underlying `documentServicesAPI.history.*` calls
listed above are forbidden.

---

## Quick Checklist for Researching an API

1. **Check the layer** — UI/VERTICAL = stop, always red.
2. **Check the name** — does it mention measure, preview, stage, panel, interaction,
   scroll, render, animation, drag, indicator, DOM? Likely red.
3. **Check for `window` / `document` usage** — any direct use of browser globals = red.
4. **Check `getDependencyAPIs()`** — does it depend on any known red API or root cause above?
   If yes, flag as red unless there is an explicit client/server split.
5. **Check DS operations** — does the implementation call any forbidden DocumentServices
   operation from the list above? If yes, red regardless of API deps.
6. **Check hook/slot contributions** — if the API exposes hooks, each contributor
   must be checked separately.
7. **Check for BI** — BI APIs are not automatically red; they can run server-side.
8. **Check for dialogs / confirmation flows** — any `confirm*`, `removeWith*`,
   `*Dialog*` method name is a strong signal of client-only.
