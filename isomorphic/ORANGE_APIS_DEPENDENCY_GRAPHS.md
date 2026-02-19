# Orange APIs — Dependency Graphs

Dependency graphs for all 9 ORANGE APIs from the site-optimizer actions analysis.
Each dep is classified using `ISOMORPHIC_CLASSIFICATION_RULES.md`.

**Legend:**
- 🔴 Red node — client-only (DOM, browser, Editor UI, Stage/Preview, HistoryAPI, etc.)
- 🟠 Orange node — ambiguous, needs case-by-case or per-method analysis
- 🟢 Green node — isomorphic, server-safe

---

## Verdict Summary

| API | Layer | RED deps | ORANGE deps | Verdict after analysis |
|---|---|---|---|---|
| PagesDataServiceAPI | DATA_SERVICE | PreviewAPI, EditorCacheAPI | PagesDataServicePrivateAPI | ORANGE — 2 RED deps, isomorphic methods exist |
| ComponentEditorAPI | EDITOR_INFRA | MouseAPI, PreviewAPI, EditorCacheAPI, ComponentEditorCachePrivateAPI + extends 4 Stage/UI interfaces | TraverseDataServiceAPI, SequenceLoaderAPI | ORANGE → heavily RED-leaning, large split needed |
| PositionDerivativeStateAPI | DERIVATIVE_STATE | ComponentLayoutAPI_deprecated, PinnedToContainerDerivativeStateAPI | ComponentEditorAPI | **Reclassify → RED** — 2 RED deps including reclassified PinnedToContainerDerivativeStateAPI |
| ComponentEditorBIAPI | EDITOR_INFRA | Same RED deps as ComponentEditorAPI (shared entry point) | TraverseDataServiceAPI, SequenceLoaderAPI | ORANGE — BI itself is server-OK, needs its own entry point to shed ComponentEditorAPI's dep chain |
| AddPanelDataAPI | DERIVATIVE_STATE | PanelsAPI, AddPanelStateAPI | EditorParamsAPI | ORANGE — data fetch is isomorphic, panel state is RED |
| OdeditorLayoutBuilderAPI | DERIVATIVE_STATE | ComponentMeasureAPI | LayoutGeneratorAPI | **Reclassify → RED** — direct ComponentMeasureAPI dep (root cause, 102 refs) |
| EditorPlatformAPI | DATA_SERVICE | WorkerManager (runtime call) | — | ORANGE → close to GREEN, only runtime WorkerManager is the blocker |
| PinnedToContainerFlowsAPI | FLOWS | 10+ RED deps | PositionDerivativeStateAPI, ComponentEditorBIAPI, ComponentEditorAPI, PagesDataServiceAPI | ORANGE — concept is isomorphic, impl is not |
| LayoutConverterAPI | FLOWS | ComponentMeasureAPI | — | ORANGE — concept is isomorphic, single RED root cause |

---

## 1. PagesDataServiceAPI

**Layer:** DATA_SERVICE (REP)

```mermaid
flowchart LR
    API["PagesDataServiceAPI\nDATA_SERVICE"]:::orange

    API --> PRIV["PagesDataServicePrivateAPI"]:::orange
    API --> EDS["ExtendedDocumentServicesAPI"]:::green
    API --> EP["EditorPointerAPI"]:::green
    API --> EC["EditorCacheAPI"]:::red
    API --> EX["ExperimentsAPI"]:::green
    API --> MEM["MemoizationAPI"]:::green
    API --> PREV["PreviewAPI"]:::red
    API --> DS["DocumentStructureAPI"]:::green
    API --> RBI["ResponsiveBIContributionAPI"]:::green

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps:**
- `PreviewAPI` — Stage/Preview (root cause, 161 refs)
- `EditorCacheAPI` — Editor State (client-side caching)

**Fix:** Remove `PreviewAPI` and `EditorCacheAPI` from the server path. Both are used by `getFocusedPageId()` / `getCurrentPageId()` style methods. The data-only methods (`isPage()`, `getPageData()`, etc.) don't need them.

---

## 2. ComponentEditorAPI

**Layer:** EDITOR_INFRA (REP)

```mermaid
flowchart LR
    API["ComponentEditorAPI\nEDITOR_INFRA"]:::orange

    API -->|extends| UBS["UiBehaviorsStageAPI"]:::red
    API -->|extends| UBW["UiBehaviorsWorkspaceAPI"]:::red
    API -->|extends| SIA["StageIndicatorsAPI"]:::red
    API -->|extends| SDA["StageDataAPI"]:::red

    API --> EP["EditorPointerAPI"]:::green
    API --> DS["DocumentServicesAPI"]:::green
    API --> MA["MouseAPI"]:::red
    API --> RBIA["ResponsiveBIAPI"]:::green
    API --> PREV["PreviewAPI"]:::red
    API --> EO["ExtensibleObjectAPI"]:::green
    API --> EX["ExperimentsAPI"]:::green
    API --> HS["HybridStoreAPI"]:::green
    API --> CT["ComponentTypeAPI"]:::green
    API --> TR["TraverseDataServiceAPI"]:::orange
    API --> EC["EditorCacheAPI"]:::red
    API --> SL["SequenceLoaderAPI"]:::orange
    API --> PC["PointerComparisonAPI"]:::green
    API --> MEMO["MemoizationAPI"]:::green
    API --> CEP["ComponentEditorCachePrivateAPI"]:::red
    API --> TA["TransactionsAPI"]:::green

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps:**
- `MouseAPI` — User Input (browser events)
- `PreviewAPI` — Stage/Preview (root cause)
- `EditorCacheAPI` — Editor State
- `ComponentEditorCachePrivateAPI` — Editor State (also depends on PreviewAPI)
- `UiBehaviorsStageAPI` / `UiBehaviorsWorkspaceAPI` / `StageIndicatorsAPI` / `StageDataAPI` — interface extensions, all Stage/Editor UI

**Note:** This API is the single most coupled API in the graph. It is a large API covering many concerns — any server migration requires a significant split, extracting only the data/structure methods.

---

## 3. PositionDerivativeStateAPI

**Layer:** DERIVATIVE_STATE (REP)

```mermaid
flowchart LR
    API["PositionDerivativeStateAPI\nDERIVATIVE_STATE"]:::orange

    API --> CLD["ComponentLayoutAPI_deprecated"]:::red
    API --> CEA["ComponentEditorAPI"]:::orange
    API --> PCD["PinnedToContainerDerivativeStateAPI"]:::red
    API --> EF["ExperimentalFlowsAPI"]:::green
    API --> CLDA["ComponentLayoutDerivativeStateAPI"]:::green
    API --> CPF["ComponentPointerFactoryAPI"]:::green
    API --> CR["ComponentRoutingAPI"]:::green

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps:**
- `ComponentLayoutAPI_deprecated` — deprecated, must not exist on server
- `PinnedToContainerDerivativeStateAPI` — recently reclassified to RED (PreviewAPI + LayoutHeuristicsAPI + EditorCacheAPI all RED)

**⚠️ Reclassify to RED.** Both RED deps are direct, not transitive. The only non-green non-red dep (`ComponentEditorAPI`) is itself heavily RED-leaning. The `getPosition()` method may be salvageable but requires reimplementation without these deps.

---

## 4. ComponentEditorBIAPI

**Layer:** EDITOR_INFRA (REP)

```mermaid
flowchart LR
    API["ComponentEditorBIAPI\nEDITOR_INFRA"]:::orange

    API -->|shared entry point| NOTE["Shares componentEditorAPIEntryPoint\nwith ComponentEditorAPI"]:::orange

    NOTE --> EP["EditorPointerAPI"]:::green
    NOTE --> DS["DocumentServicesAPI"]:::green
    NOTE --> MA["MouseAPI"]:::red
    NOTE --> RBIA["ResponsiveBIAPI"]:::green
    NOTE --> PREV["PreviewAPI"]:::red
    NOTE --> EO["ExtensibleObjectAPI"]:::green
    NOTE --> EX["ExperimentsAPI"]:::green
    NOTE --> HS["HybridStoreAPI"]:::green
    NOTE --> CT["ComponentTypeAPI"]:::green
    NOTE --> TR["TraverseDataServiceAPI"]:::orange
    NOTE --> EC["EditorCacheAPI"]:::red
    NOTE --> SL["SequenceLoaderAPI"]:::orange
    NOTE --> PC["PointerComparisonAPI"]:::green
    NOTE --> MEMO["MemoizationAPI"]:::green
    NOTE --> CEP["ComponentEditorCachePrivateAPI"]:::red
    NOTE --> TA["TransactionsAPI"]:::green

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**Key note:** `ComponentEditorBIAPI` is declared in the same entry point as `ComponentEditorAPI` — it inherits the entire dep chain even though BI itself is server-OK. The RED deps here are not intrinsic to BI; they belong to other APIs in the same entry point.

**Fix:** Extract `ComponentEditorBIAPI` into its own entry point with only the deps it actually needs. BI sending is server-safe — the RED chain is inherited baggage, not inherent.

---

## 5. AddPanelDataAPI

**Layer:** DERIVATIVE_STATE (Harmony)

```mermaid
flowchart LR
    API["AddPanelDataAPI\nDERIVATIVE_STATE\n(Harmony)"]:::orange

    API --> HS["HybridStoreAPI"]:::green
    API --> HC["HttpClientAPI"]:::green
    API --> TC["TemplatesCmsDataServiceAPI"]:::green
    API --> TCC["TemplatesCmsConditionsServiceAPI"]:::green
    API --> DS["DocumentServicesAPI"]:::green
    API --> EPA["EditorParamsAPI"]:::orange
    API --> EX["ExperimentsAPI"]:::green
    API --> RF["ResponsiveFedopsAPI"]:::green
    API --> PAN["PanelsAPI"]:::red
    API --> APS["AddPanelStateAPI"]:::red

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps:**
- `PanelsAPI` — Editor UI (panel = always client-only)
- `AddPanelStateAPI` — Editor State (tracks panel open/closed state, UI concern)

**Fix:** The data fetching methods (`getData()`, template resolution) are fully isomorphic. `PanelsAPI` and `AddPanelStateAPI` are only used for panel open/close state, not for the data itself. Split into data-only server interface.

---

## 6. OdeditorLayoutBuilderAPI

**Layer:** DERIVATIVE_STATE (Harmony)

```mermaid
flowchart LR
    API["OdeditorLayoutBuilderAPI\nDERIVATIVE_STATE\n(Harmony)"]:::orange

    API --> LG["LayoutGeneratorAPI"]:::orange
    API --> EPHI["EditorPlatformHostIntegrationAPI"]:::green
    API --> CMA["ComponentMeasureAPI"]:::red

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps:**
- `ComponentMeasureAPI` — DOM Measurements (root cause, 102 refs)

**⚠️ Reclassify to RED.** `ComponentMeasureAPI` is a direct dependency and is a known root cause. This confirms the earlier suspicion ("may depend on measurement hints"). The API cannot be isomorphic in its current form.

**Fix:** Pass layout dimensions as data inputs instead of measuring via `ComponentMeasureAPI`. This is the same pattern proposed for the flex actions (`StageContextBuilderAPI` replacement).

---

## 7. EditorPlatformAPI

**Layer:** DATA_SERVICE (REP)

```mermaid
flowchart LR
    API["EditorPlatformAPI\nDATA_SERVICE"]:::orange

    API --> EPHI["EditorPlatformHostIntegrationAPI"]:::green
    API -.->|runtime call| WM["WorkerManager\n(browser Web Workers)"]:::red

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps:**
- `WorkerManager` — not a declared dep, accessed via `getWorkerManager()` at runtime. Browser Web Workers are client-only.

**Note:** This is the closest ORANGE API to GREEN. Its only declared dep is `EditorPlatformHostIntegrationAPI` (🟢). The blocker is a single runtime call to `getWorkerManager()`. A server-side SDK host that doesn't use a WorkerManager would make this fully isomorphic.

---

## 8. PinnedToContainerFlowsAPI

**Layer:** FLOWS (REP)

```mermaid
flowchart LR
    API["PinnedToContainerFlowsAPI\nFLOWS"]:::orange

    API --> LHD["LayoutHeursticsDerivativeStateAPI"]:::red
    API --> CTA["ComponentTypeAPI"]:::green
    API --> HA["HistoryAPI"]:::orange
    API --> CLD["ComponentLayoutAPI_deprecated"]:::red
    API --> PCD["PinnedToContainerDerivativeStateAPI"]:::red
    API --> SCB["StageContextBuilderAPI"]:::red
    API --> CMA["ComponentMeasureAPI"]:::red
    API --> PDS["PositionDerivativeStateAPI"]:::red
    API --> PCO["PointerComparisonAPI"]:::green
    API --> CEBI["ComponentEditorBIAPI"]:::orange
    API --> EF["ExperimentalFlowsAPI"]:::green
    API --> CHA["ComponentHierarchyAPI"]:::green
    API --> CHFA["ComponentHierarchyFlowsAPI"]:::green
    API --> CLFA["ComponentLayoutFlowsAPI"]:::green
    API --> CLC["ComponentsLocatorAPI"]:::red
    API --> CDSA["ComponentsDerivativeStateAPI"]:::green
    API --> CADA["ComponentArrangementDerivativeStateAPI"]:::green
    API --> CAFA["ComponentArrangementFlowsAPI"]:::green
    API --> UOW["UnitOfWorkFlowAPI"]:::green
    API --> CLDA["ComponentLayoutDerivativeStateAPI"]:::green
    API --> EDSA["ExtendedDocumentServicesAPI"]:::green
    API --> VIA["VariantsIteratorAPI"]:::green
    API --> LBA["LayoutBuilderAPI"]:::green
    API --> TA["TransactionsAPI"]:::green
    API --> CEA["ComponentEditorAPI"]:::orange
    API --> CRA["ComponentRoutingAPI"]:::green
    API --> PDSA["PagesDataServiceAPI"]:::orange
    API --> ICA["InteractionContextAPI"]:::red
    API --> EPA["EditorPointerAPI"]:::green

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps (8):** `LayoutHeursticsDerivativeStateAPI`, `ComponentLayoutAPI_deprecated`, `PinnedToContainerDerivativeStateAPI`, `StageContextBuilderAPI`, `ComponentMeasureAPI`, `PositionDerivativeStateAPI` *(reclassified RED)*, `ComponentsLocatorAPI`, `InteractionContextAPI`

**ORANGE deps (updated):** `HistoryAPI` — has server-compatible implementation (no-op history recording)

---

## 9. LayoutConverterAPI

**Layer:** FLOWS (REP)

```mermaid
flowchart LR
    API["LayoutConverterAPI\nFLOWS"]:::orange

    API --> SGDA["SuperGridDerivativeStateAPI"]:::green
    API --> CVDA["ComponentVisibilityDerivativeStateAPI"]:::green
    API --> CHA["ComponentHierarchyAPI"]:::green
    API --> CRA["ComponentRoutingAPI"]:::green
    API --> DSA["DocumentServicesAPI"]:::green
    API --> SCDA["SystemContainerDerivativeStateAPI"]:::green
    API --> CLFA["ComponentLayoutFlowsAPI"]:::green
    API --> CMW["ContentMaxWidthFlowsAPI"]:::green
    API --> GSA["GenericStyleAPI"]:::green
    API --> SCMDA["SystemComponentsDerivativeStateAPI"]:::green
    API --> GVDA["GlobalVariablesDerivativeStateAPI"]:::green
    API --> EPA["EditorPointerAPI"]:::green
    API --> SBA["SharedBlocksAPI"]:::green
    API --> SGCDA["SuperGridCellDerivativeStateAPI"]:::green
    API --> CTA["ComponentTypeAPI"]:::green
    API --> CFA["ComponentFlowsAPI"]:::green
    API --> CMA["ComponentMeasureAPI"]:::red
    API --> RFPA["ResponsiveFedopsAPI"]:::green
    API --> RBIA["ResponsiveBIAPI"]:::green
    API --> AGUA["AutoGridUtilsAPI"]:::green
    API --> HCMA["HeaderComponentsManagementAPI"]:::green

    classDef red fill:#4a1a1a,stroke:#f44,color:#eee
    classDef orange fill:#3a2a0a,stroke:#f90,color:#eee
    classDef green fill:#1a3a1a,stroke:#4c4,color:#eee
```

**RED deps (1):** `ComponentMeasureAPI` — the single blocker. 20 out of 21 deps are green.

**Fix:** Replace `ComponentMeasureAPI` usage with data-input parameters. This is the cleanest migration candidate among the heavy ORANGE APIs.

---

## Reclassifications from This Analysis

| API | Was | Now | Reason |
|---|---|---|---|
| `PositionDerivativeStateAPI` | 🟠 | 🔴 | `PinnedToContainerDerivativeStateAPI` (RED) + `ComponentLayoutAPI_deprecated` (RED) are direct deps |
| `OdeditorLayoutBuilderAPI` | 🟠 | 🔴 | Direct `ComponentMeasureAPI` dep confirmed — measurement root cause |
