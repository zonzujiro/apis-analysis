# Isomorphic Conflict Analysis

Core-candidate APIs that directly depend on browser-only APIs (1-hop check).

## Method

1. Seeded 205 "definitely not isomorphic" APIs — all UI/VERTICAL layer APIs plus explicitly browser-dependent ones (measurement, preview/iframe, scroll, DOM, browser events, indicators)
2. Parsed `getDependencyAPIs()` from 2,028 entry points across REP
3. Cross-referenced: found core-candidate APIs (SITE, EDITOR_INFRA, DATA_SERVICE, DERIVATIVE_STATE, FLOWS) that directly depend on any seed

## Summary

| Layer | Conflicts | Note |
|---|---|---|
| SITE | 6 | Mostly preview-related |
| EDITOR_INFRA | 6 | All depend on PreviewAPI |
| DATA_SERVICE | 33 | PreviewAPI, PreviewDisplayAPI, BrowserAPI, ScrollBarMeasureAPI |
| DERIVATIVE_STATE | 56 | ComponentMeasureAPI and PreviewDisplayAPI dominate |
| FLOWS | 104 | Broad mix — measurement, preview, scroll, browser |
| **Total** | **205** | |

## Top browser-dependency categories

| Category | References | Key APIs causing it |
|---|---|---|
| preview/iframe | 161 | PreviewAPI, PreviewDisplayAPI, PreviewModeFlowsAPI |
| measurement | 102 | ComponentMeasureAPI, ComponentBoundingBoxAPI, ScrollBarMeasureAPI |
| ui-layer | 17 | Various UI layer APIs |
| scroll | 17 | ScrolledComponentsDerivativeStateAPI, ScrolledComponentsStoreAPI |
| browser | 13 | BrowserAPI, ClientDimensionsAPI, BlockingLayerAPI |
| dom-order | 5 | DOMOrderFlowsAPI, DOMSortingAPI |
| user-input | 4 | SelectionModelAPI, DragMouseActionAcceptorAPI |
| animation | 2 | StageAnimationsFlowsAPI, ResizeTransitionAPI |

## Key insight: PreviewAPI is the biggest blocker

PreviewAPI (SITE layer) and PreviewDisplayAPI (DATA_SERVICE) are the two most common browser dependencies. Together they account for 161 out of 321 total dependency references. These are iframe/DOM-based APIs that manage the site preview rendering.

**This means most "isomorphic conflicts" aren't about the API's own logic being browser-dependent — they're about needing the preview iframe for context.** On the server, this context would come from a different source (e.g. a virtual document model instead of an iframe).

## SITE layer (6 conflicts)

| API | Browser deps | Reason |
|---|---|---|
| PreviewComponentAPI | PreviewAPI, ScrolledComponentsStoreAPI | preview, scroll |
| PreviewAPI | PreviewEventsPluginAPI | preview |
| PreviewPrivateAPI | ClientDimensionsAPI, ScrolledComponentsStoreAPI | browser, scroll |
| DocumentServicesAPI | ClientDimensionsAPI, ScrolledComponentsStoreAPI | browser, scroll |
| MiniSiteManagerAPI | ClientDimensionsAPI, ScrolledComponentsStoreAPI | browser, scroll |
| TpaDataServiceAPI | PreviewAPI | preview |

## EDITOR_INFRA layer (6 conflicts)

| API | Browser deps | Reason |
|---|---|---|
| ComponentEditorCachePrivateAPI | PreviewAPI | preview |
| ComponentEditorBuilderAPI | PreviewAPI | preview |
| ComponentEditorAPI | PreviewAPI | preview |
| ComponentEditorBIAPI | PreviewAPI | preview |
| ExternalStateSubscriberAPI | PreviewAPI | preview |
| SingleComponentAbstractionPrivateAPI | PreviewAPI | preview |

All EDITOR_INFRA conflicts are caused by a single dependency: **PreviewAPI**.

## DATA_SERVICE layer (33 conflicts)

Notable core APIs affected:

| API | Browser deps |
|---|---|
| ComponentDataAPI | PreviewAPI |
| ComponentLayoutAPI | PreviewDisplayAPI, PreviewAPI |
| HistoryAPI | PreviewDisplayAPI |
| PagesDataServiceAPI | PreviewAPI |
| PagesAPI | PreviewAPI |
| EditorFlowAPI | PreviewAPI |
| BreakpointsDataServiceAPI | PreviewDisplayAPI, ClientDimensionsAPI, PreviewAPI |
| SaveAPI | BrowserAPI |
| SaveAndPublishAPI | BrowserAPI |
| SpxAPI | ScrollBarMeasureAPI |
| ThemePrivateAPI | ScrollBarMeasureAPI |
| ScaleProportionallyAPI | PreviewAPI, ScrollBarMeasureAPI |

## DERIVATIVE_STATE layer (56 conflicts)

Notable core APIs affected:

| API | Browser deps |
|---|---|
| ComponentMeasureAPI | PreviewDisplayAPI |
| ComponentMeasureDerivativeStateAPI | ComponentMeasureAPI, ScrolledComponents*, PreviewDisplayAPI |
| ComponentLayoutDerivativeStateAPI | PreviewDisplayAPI, PreviewAPI, ComponentMeasureAPI |
| ComponentBoundingBoxAPI | ComponentMeasureAPI |
| ComponentsDerivativeStateAPI | ScrolledComponents*, PreviewDisplayAPI, PreviewAPI, ComponentMeasureAPI |
| BreakpointsDerivativeStateAPI | PreviewDisplayAPI, ScrolledComponents*, ComponentMeasureAPI, ClientDimensionsAPI |
| GridDerivativeStateAPI | PreviewAPI |
| GridSliceAPI | PreviewDisplayAPI, ComponentMeasureAPI |
| GridLayoutDerivativeStatePrivateAPI | ComponentMeasureAPI, PreviewAPI |
| GridChildLayoutHeuristicsAPI | ComponentMeasureAPI |
| SuperGridDerivativeStateAPI | ComponentMeasureAPI |
| UnitsConversionsAPI | PreviewDisplayAPI, ComponentMeasureAPI |
| SnapAPI | PreviewDisplayAPI, ComponentMeasureAPI |

## FLOWS layer (104 conflicts)

Too many to list individually. The dominant patterns:
- Almost every FLOWS API depends on PreviewAPI or PreviewDisplayAPI
- Many layout/grid flows depend on ComponentMeasureAPI
- Some depend on BrowserAPI (save, publish, navigation flows)

## Conclusions

1. **PreviewAPI is the single biggest isomorphic blocker.** Abstracting the preview iframe behind an isomorphic interface (virtual document on server, iframe on client) would resolve the majority of SITE and EDITOR_INFRA conflicts.

2. **ComponentMeasureAPI is the second biggest blocker**, especially in DERIVATIVE_STATE. Layout calculations need measurements, but measurements need the DOM. The client/server split discussed in Core Rules directly addresses this.

3. **The conflict count (205) overstates the real problem.** Most conflicts are caused by just 3-4 root APIs (PreviewAPI, PreviewDisplayAPI, ComponentMeasureAPI, ClientDimensionsAPI). Fix the abstraction at these roots and most downstream conflicts disappear.

4. **Some conflicts are legitimate architecture issues** (e.g. SaveAPI → BrowserAPI, BreakpointsDataServiceAPI → ClientDimensionsAPI). These need actual refactoring to separate browser-specific logic.
