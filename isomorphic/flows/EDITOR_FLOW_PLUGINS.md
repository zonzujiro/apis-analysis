# EditorFlowAPI — `contributePlugin` Complete Registry

All plugins registered via `editorFlowAPI.contributePlugin()` in both repos.
Covers all three topics: **add**, **layout**, **hierarchy**.

> **Why this matters for server migration:**
> `EditorFlowAPI.run()` fires these plugins around every transaction. Replacing
> `EditorFlowAPI.run()` with `TransactionsAPI.run()` (the planned server fix) skips
> all plugins. This document verifies that skipping them is correct for server contexts
> and flags any that carry document-model side effects.

---

## Classification framework reminder

| Color | Meaning |
|-------|---------|
| ✗ RED | Client-only (DOM, stage, UI selection, UI panels, snackbars, measurements) |
| ⚠️ ORANGE | Needs splitting — has ORANGE API deps |
| ✓ GREEN | Server-safe — pure data |

---

## By-Topic Registry

### Topic: `add` (6 plugins)

| # | Package | Repo | What it does | Key APIs | Verdict |
|---|---------|------|-------------|---------|---------|
| 1 | `odeditor-package-auto-grid` | Harmony | Recalculates AutoGrid layout on parent after component add | `ComponentMeasureAPI`, `StageContextBuilderAPI`, `OdeditorBreakpointsAPI` | ✗ RED |
| 2 | `odeditor-package-auto-dom` | Harmony | `AutoDOMOrderFlowsAPI.runAutoReorderInParent()` — reorders DOM render order in parent after add | `AutoDOMOrderFlowsAPI` → `DOMSortingAPI` → `ComponentMeasureAPI` → `PreviewDisplayAPI` | ✗ RED |
| 3 | `editor-package-components` | REP | Selects newly added component if `selectOnStage` option set | `ComponentSelectFlowsAPI`, `ComponentRoutingAPI` | ✗ RED (UI selection state) |
| 4 | `editor-package-component-editors` | REP | Tracks the last added component for UI focus | `ComponentInteractionAPI`, `StageContextBuilderAPI`, `PagesDataServiceAPI.getFocusedPageId()` | ✗ RED |
| 5 | `editor-package-dom-order` | REP | `AutoDOMOrderFlowsAPI.runAutoReorderInParent()` — same as #2 | Same chain | ✗ RED |

> All 5 add-topic plugins are RED. Skipping via `TransactionsAPI.run()` is correct — these are stage/render concerns with no document-model side effects.

---

### Topic: `layout` (6 plugins)

| # | Package | Repo | What it does | Key APIs | Verdict |
|---|---------|------|-------------|---------|---------|
| 6 | `odeditor-package-auto-grid` | Harmony | Recalculates grid cell heights on layout change (overflow detection) | `ComponentMeasureAPI`, `StageContextBuilderAPI`, `SharedBlocksAPI` (RED) | ✗ RED |
| 7 | `odeditor-package-auto-dom` | Harmony | `AutoDOMOrderFlowsAPI.runAutoReorder()` — reorders DOM render order after layout change | `AutoDOMOrderFlowsAPI` → `DOMSortingAPI` → `ComponentMeasureAPI` | ✗ RED |
| 8 | `editor-package-dom-order` | REP | `AutoDOMOrderFlowsAPI.runAutoReorder()` — same as #7 | Same chain | ✗ RED |
| 9 | `editor-package-mesh-layout` (critical¹) | REP | Freezes mesh container layout + fixes margins before full track span | `MeshLayoutAPI.freeze()`, `MeshLayoutPrivateAPI.fixMarginsBeforeApplyFullTrackSpan()`, `MeshDerivativeStateAPI` | ✗ RED |
| 10 | `editor-package-page-grid` | REP | Validates page grid / section integrity after layout change | `ComponentEditorAPI.hasSectionBehaviors()`, `PagesDataServiceAPI.getPageOfComponent()`, `SectionFixerPrivateAPI` | ⚠️ ORANGE |
| 11 | `editor-package-content-max-width` | REP | Shows snackbar when components are placed outside max-width boundary | `StageContextBuilderAPI.getGlobalPointer()`, `ContentMaxWidthUIPrivateAPI.activateOutSideOfMaxWidthSnackBarIfNeeded()` | ✗ RED (UI notification) |

> ¹ Registered with `shouldRollbackOnFailure=true` (critical plugin — runs before the transaction).

---

### Topic: `hierarchy` (4 plugins)

| # | Package | Repo | What it does | Key APIs | Verdict |
|---|---------|------|-------------|---------|---------|
| 12 | `odeditor-package-auto-grid` | Harmony | Recalculates AutoGrid for affected parents on hierarchy change | `ComponentMeasureAPI`, `StageContextBuilderAPI`, `SharedBlocksAPI` | ✗ RED |
| 13 | `editor-package-components` | REP | Updates selection to the rendered instance of reparented component | `ComponentInteractionAPI.select()`, `ComponentRoutingAPI`, `PointerComparisonAPI` | ✗ RED (UI selection) |
| 14 | `editor-package-mesh-layout` | REP | Generates AutoGrid for meshed containers on hierarchy change | `MeshLayoutAPI.generateAutoGrid()`, `ComponentHierarchyAPI`, `MeshDerivativeStateAPI` | ✗ RED |
| 15 | `editor-package-interactions` | REP | Closes the interactions panel when the trigger component is reparented | `InteractionsPanelFlowsPrivateAPI.closePanel()`, `InteractionsPanelStatePrivateAPI.getTriggerComp()` | ✗ RED (UI panel state) |

---

## Classification notes

**AutoGrid `['layout', 'hierarchy']` (Harmony):** Same package as the `['add']` plugin but registered separately. Uses `SharedBlocksAPI` (RED, reclassified from GREEN — uses forbidden DS ops + PreviewAPI). This is a separate entry in the RED column.

**Section Fixer `['layout']` (REP) — ORANGE:** `ComponentEditorAPI.hasSectionBehaviors()` is ORANGE (extends UI behaviors API). `PagesDataServiceAPI.getPageOfComponent()` — this is a data-layer lookup (component → page pointer) and may be server-safe, unlike `getCurrentPage()` / `getFocusedPage()`. `SectionFixerPrivateAPI` needs its own investigation to confirm it doesn't call DOM/measurement APIs. Classified ORANGE until verified.

**Interactions Panel Close `['hierarchy']` (REP) — RED but trivially skippable:** `closePanel()` is pure UI state — closing a panel. No document-model mutation. Skipping on server has no correctness impact.

**Content Max Width `['layout']` (REP) — RED but trivially skippable:** Shows a snackbar notification. No document-model mutation. Skipping on server is correct.

**Component Selection `['hierarchy']` (REP) — RED:** The agent classified this GREEN, but `componentInteractionAPI.select()` is a UI selection state operation (visual focus). Consistent with our earlier RED classification of `ComponentInteractionAPI`.

**Set Last Added Component (REP) — RED:** The agent re-classified this GREEN but it uses `StageContextBuilderAPI.addCurrentContextToRef()` + `ComponentInteractionAPI.setLastAddedComponent()` + `PagesDataServiceAPI.getFocusedPageId()` (forbidden DS op). Our earlier classification stands: RED.

---

## Summary by verdict

| Verdict | Count | Plugins |
|---------|-------|---------|
| ✗ RED | 14 | #1–9, #11–15 |
| ⚠️ ORANGE | 1 | #10 Section Fixer |
| ✓ GREEN | 0 | — |

**All 15 plugins are RED or ORANGE.** Replacing `EditorFlowAPI.run()` with `TransactionsAPI.run()` on the server correctly skips all of them.

The Section Fixer (ORANGE) is the only candidate with any isomorphic potential, but it fires on **layout** changes — site-optimizer layout actions (SET_FLEX_*) are themselves already blocked by `StageContextBuilderAPI`. If/when those actions become server-safe, Section Fixer needs its own evaluation.

---

## Plugins by site-optimizer action

Which site-optimizer actions use `EditorFlowAPI.run()` and thus trigger plugins:

| Action | Uses EditorFlowAPI.run()? | Plugin topics triggered |
|--------|--------------------------|------------------------|
| **ADD_COMPONENT** (Harmony) | ✓ Yes | `add` → plugins #1–5 |
| **MIGRATION** (Harmony) | ✓ Yes (via EditorFlowAPI) | unclear — no add/layout/hierarchy ops at top level |
| **SET_*_ANIMATION** (Harmony) | Uses `EditorUIAPI.run()` (alias) | unclear |
| **SET_FLEX_*_LAYOUT** (REP) | Depends on flexSiteOptimizerEntryPoint impl | possibly `layout` → plugins #6–11 |
| **REORDER / UPDATE_PARENT** (REP) | Depends on componentFlowsSiteOptimizerActionsEntryPoint | possibly `hierarchy` → plugins #12–15 |

> **TODO:** Verify whether flexSiteOptimizerEntryPoint and componentFlowsSiteOptimizerActionsEntryPoint wrap their actions in `EditorFlowAPI.run()`. If they do, the layout/hierarchy plugins fire for SET_FLEX_* and REORDER/UPDATE_PARENT too.
