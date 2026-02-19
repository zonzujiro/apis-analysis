# removeComponent — Dependency Research

Entry point: `ComponentFlowsAPI.removeComponent()` in `editor-package-flows`.

---

## Mermaid Diagram

```mermaid
flowchart LR
    RC["ComponentFlowsAPI\n.removeComponent()"]

    RC --> BRS["BeforeRemoveComponentCheckSlot\n.getItems() → run hooks"]
    BRS -->|"any returns 'cancelRemove'"| ABORT["↩ return (cancelled)"]
    BRS -->|all pass| BI

    RC --> BI["ComponentEditorBIAPI\nreportComponentDeleted()\n[if options.biParams]"]
    RC --> ROUTING1["ComponentRoutingAPI\n.getCompRefForRemove()"]
    RC --> CONFIRM["ComponentEditorAPI\n.flowBranching.confirmDeleteComponent()\n[unless options.silent]"]
    CONFIRM -->|"not confirmed"| ABORT2["↩ return"]
    CONFIRM -->|confirmed| CI

    RC --> CI["ComponentInteractionAPI\n.getSelected()\n.deselect()\n.clearAllHoverInteractions()"]

    RC --> EF["EditorFlowAPI.run()\n{ history, transaction }"]

    EF --> UNST["ComponentHierarchyFlowsAPI\n.unstable_removeComponent()"]

    UNST --> DRAFT["ComponentDraftDataModeAPI\n.ifOnDraftMode()"]
    DRAFT -->|draft mode| ERR["✗ throw EnhancedError"]
    DRAFT -->|normal| ROUTING2["ComponentRoutingAPI\n.getCompRefForDeleteComponent()"]

    ROUTING2 --> STRAT["ComponentEditorAPI\n.getRemoveComponentStrategy()\n[runComponentEditorFuncWithEnhancers]"]

    STRAT -->|"isRepeaterItem()"| REP["RepeaterComponentFlowsPrivateAPI\n.removeItem()"]
    STRAT -->|"isTpaPageByPointer()"| TPAPAGE["removeTpaPage()\n+ clearTpaSectionsAndWidgets()"]
    STRAT -->|TPA Widget| TPAWID["TpaAPI\n.deleteWidget()"]
    STRAT -->|Page| PAGE["PageRemoverAPI\n.removePage()"]
    STRAT -->|Lightbox| LB["PagesDataServiceAPI.getPageOfComponent()\n+ LightboxRemoveConfirmationAPI\n.removeWithConfirmationByPointer()"]
    STRAT -->|Default| DEF["EditorPointerAPI.executeDocumentPointerAPI()\n+ StageContextBuilderAPI.getGlobalPointer()"]

    DEF --> DS["DocumentServicesAPI\n.components.remove()\n✦ actual DS mutation"]

    EF --> OT["OperationsTrackerAPI\n.registerOperation()"]
    OT --> CR["ConcurrentRejectionsAPI\n.componentRemoved()"]
    OT --> HIER["ComponentHierarchyAPI\nparent / ancestors / descendants"]

    EF --> AFTER["ComponentEditorAPI\n.hooks.structure.afterRemoveFromContainer()"]
    EF --> SEL["ComponentSelectFlowsAPI\n.onDeselect()"]
    EF --> RENDER["TransactionsAPI\n.render()"]
    EF --> NOTIFY["notifyRemoveComponentListeners()"]

    style DS fill:#2a4a2a,stroke:#4caf50,color:#ccc
    style ABORT fill:#4a2a2a,stroke:#f44,color:#ccc
    style ABORT2 fill:#4a2a2a,stroke:#f44,color:#ccc
    style ERR fill:#4a2a2a,stroke:#f44,color:#ccc
    style STRAT fill:#2a3a4a,stroke:#5a9fd4,color:#ccc
```

---

## Level 1 — ComponentFlowsAPI.removeComponent()

**File**: `editor-package-flows/src/apis/createComponentRemovingFlowsAPI.ts`

Two code paths controlled by `ExperimentalFlowsAPI.isInExperimentalFlow()`. The old path delegates entirely to `ComponentFlowsHooksAPI.getDeprecatedSlotContribution().removeComponentEditorFlowOldVariant()`. All detail below is the new (experimental) path.

### API Dependencies

| API | Methods | Purpose |
|---|---|---|
| `ExperimentalFlowsAPI` | `isInExperimentalFlow()` | Branch between new/old variant |
| `ComponentRoutingAPI` | `getCompRefForRemove()` | Resolve canonical comp refs |
| `BeforeRemoveComponentCheckSlot` | `getItems()` | Cancellation hooks — any returning `'cancelRemove'` aborts |
| `ComponentEditorBIAPI` | `sendComponentEvents()` | BI event via `reportComponentDeleted()` helper |
| `ComponentRemovingFlowsUtilsAPI` | `getDeleteComponentBIParamsEnrichers()` | BI enrichers |
| `PointerComparisonAPI` | `component.isSame()` | Comp ref equality |
| `ComponentEditorAPI` | `flowBranching.confirmDeleteComponent()` | Confirmation dialog (skipped if `options.silent`) |
| `ComponentEditorAPI` | `hooks.structure.beforeRemove()` | Lifecycle hook per component and per child |
| `ComponentEditorAPI` | `hooks.structure.removeFromContainer()` | Notify container of removal |
| `ComponentEditorAPI` | `hooks.structure.afterRemoveFromContainer()` | Post-removal container hook |
| `ComponentInteractionAPI` | `getSelected()`, `deselect()`, `clearAllHoverInteractions()` | Clear selection/hover |
| `EditorFlowAPI` | `run({ history, transaction })` | Wrap in history entry + transaction |
| `ComponentHierarchyAPI` | `OldBadPerformance.parent.unstable_get()` | Get parent |
| `ComponentHierarchyAPI` | `OldBadPerformance.ancestors.unstable_get()` | Get ancestors (for re-selection) |
| `ComponentHierarchyAPI` | `OldBadPerformance.children.unstable_getDescendants()` | Get descendants (to run beforeRemove on each) |
| `ComponentHierarchyFlowsAPI` | `unstable_removeComponent()` | **Actual structural removal** ↓ |
| `DocumentStructureAPI` | `doesComponentExist()` | Check if comp still exists post-removal |
| `OperationsTrackerAPI` | `registerOperation()` | Register as hierarchy operation |
| `ConcurrentRejectionsAPI` | `componentRemoved()` | Notify concurrent ops system |
| `TransactionsAPI` | `render()` | Trigger render |
| `ComponentSelectFlowsAPI` | `onDeselect()` | Re-select nearest surviving ancestor |
| `ComponentFlowsHooksAPI` | `getDeprecatedSlotContribution()` | Old variant entry point |

---

## Level 2 — ComponentHierarchyFlowsAPI.unstable_removeComponent()

**File**: `editor-package-hierarchy/src/apis/hierarchyAPI/componentHierarchyFlowsPrivateAPI/createComponentHierarchyFlowsPrivateAPI.ts`

**SlotKey**: `FLOWS` layer, public

A thin wrapper — resolves the right comp ref, then delegates entirely to a strategy function.

```typescript
const unstable_removeComponent = componentDraftDataModeAPI.ifOnDraftMode(
  (compRef) => { throw new EnhancedError('Can not remove component while in draft mode', ...) },
  async (compRef, options) => {
    const compToRemove = componentRoutingAPI.getCompRefForDeleteComponent(compRef)
    const removeFunction = componentEditorAPI.getRemoveComponentStrategy(compToRemove)
    await removeFunction(options)
  }
)
```

### API Dependencies

| API | Method | Purpose |
|---|---|---|
| `ComponentDraftDataModeAPI` | `ifOnDraftMode()` | Guard — throws if in draft mode |
| `ComponentRoutingAPI` | `getCompRefForDeleteComponent()` | Resolve ref for deletion |
| `ComponentEditorAPI` | `getRemoveComponentStrategy()` | **Returns the type-specific removal strategy** ↓ |

---

## Level 3 — ComponentEditorAPI.getRemoveComponentStrategy()

**Wrapper**: `editor-package-component-editor-api/src/createComponentEditorAPI.ts`

```typescript
getRemoveComponentStrategy(compRef) {
  return runComponentEditorFuncWithEnhancers(compRef, 'getRemoveComponentStrategy', () => [compRef])
}
```

Routes through the enhancer chain. Each enhancer gets first pick; the first matching branch wins.

### Strategy Branches

#### Branch 1 — Repeater Item
**File**: `editor-components-repeater/src/componentEditors/createRepeaterItemEditorEnhancer.tsx`

**Condition**: `isRepeaterItem(compRef)`

| API | Method |
|---|---|
| `RepeaterComponentFlowsPrivateAPI` | `removeItem(compRef, options)` |

#### Branch 2 — TPA Page
**File**: `editor-package-tpa/src/pages/enhancer/tpaPageComponentEditorEnhancer.ts`

**Condition**: `isPageComponent(compRef) && pagesDataServiceAPI.isTpaPageByPointer(compRef)`

| API | Method |
|---|---|
| `PagesDataServiceAPI` | `isTpaPageByPointer(compRef)` |
| *(local)* | `removeTpaPage(compRef, options)` |
| *(local)* | `clearTpaSectionsAndWidgets(compRef)` |

Falls back to default strategy if page is not a TPA page.

#### Branch 3 — TPA Widget
**File**: `editor-package-tpa/src/widget/tpaWidgetComponentEditor.ts`

**Condition**: Component type is TPA Widget

| API | Method |
|---|---|
| `TpaAPI` | `deleteWidget(compRef, options)` |

#### Branch 4 — Page
**File**: `editor-components-page/src/componentEditors/createPageComponentEditor.tsx`

**Condition**: Component type is Page

| API | Method |
|---|---|
| `PageRemoverAPI` | `removePage(compRef, options)` |

#### Branch 5 — Lightbox Container
**File**: `editor-package-lightbox/src/componentEditors/lightboxContainerEditor.tsx`

**Condition**: Component type is Lightbox Container

| API | Method |
|---|---|
| `PagesDataServiceAPI` | `getPageOfComponent(compRef)` |
| `LightboxRemoveConfirmationAPI` | `removeWithConfirmationByPointer(lightboxPage, options.origin)` |

#### Branch 6 — Default (all other components)
**File**: `editor-package-component-editors/src/createDefaultComponentEditor.tsx`

**Condition**: Fallback

| API | Method |
|---|---|
| `StageContextBuilderAPI` | `getGlobalPointer(compRef)` |
| `EditorPointerAPI` | `executeDocumentPointerAPI(pointer, { executeForDocumentPointer: ... }, 'identity')` |
| `DocumentServicesAPI` | `components.remove()` — **actual document model mutation** |

---

## Key Design Observations

1. **Cancellation slot** — `BeforeRemoveComponentCheckSlot` lets any registered hook veto removal before it starts.
2. **Silent mode** — `options.silent` skips the confirmation dialog entirely.
3. **Draft guard** — `unstable_removeComponent` throws immediately if called during draft mode.
4. **Strategy pattern via enhancer chain** — `getRemoveComponentStrategy` dispatches to 6 different implementations based on component type. All enhancers fall back down the chain for unrecognized types.
5. **`DocumentServicesAPI.components.remove`** is the only true document mutation — everything else is orchestration, hooks, and BI.
6. **Old variant** — the entire new flow is gated behind `ExperimentalFlowsAPI.isInExperimentalFlow()`. The old path goes through `ComponentFlowsHooksAPI` instead.
