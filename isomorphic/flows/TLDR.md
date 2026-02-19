# Site Optimizer Actions — Isomorphic Research TLDR

Summary of all site-optimizer action isomorphic analyses.
Each action links to its dedicated analysis file.

---

## Status at a Glance

| Action | Source | Verdict | Key Blocker | Effort |
|---|---|---|---|---|
| **UPDATE_PARENT** | [SITE_OPT_COMPONENT_FLOWS](SITE_OPT_COMPONENT_FLOWS.md) | ✓ GREEN | None | — |
| **REORDER** | [SITE_OPT_COMPONENT_FLOWS](SITE_OPT_COMPONENT_FLOWS.md) | ✓ GREEN | None | — |
| **SET_ENTRANCE_ANIMATION** | [SITE_OPT_ANIMATIONS](SITE_OPT_ANIMATIONS.md) | ⚠️ ORANGE | `EditorUIAPI.run()` wrapper | Minimal |
| **SET_LOOP_ANIMATION** | [SITE_OPT_ANIMATIONS](SITE_OPT_ANIMATIONS.md) | ⚠️ ORANGE | `EditorUIAPI.run()` wrapper | Minimal |
| **SET_MOUSE_EFFECT_ANIMATION** | [SITE_OPT_ANIMATIONS](SITE_OPT_ANIMATIONS.md) | ⚠️ ORANGE | `EditorUIAPI.run()` wrapper | Minimal |
| **SET_SCROLL_ANIMATION** | [SITE_OPT_ANIMATIONS](SITE_OPT_ANIMATIONS.md) | ⚠️ ORANGE | `EditorUIAPI.run()` wrapper | Minimal |
| **SET_FLEX_CONTAINER_LAYOUT** | [SITE_OPT_FLEX_LAYOUT](SITE_OPT_FLEX_LAYOUT.md) | ⚠️ ORANGE | `StageContextBuilderAPI.addCurrentContextToRef()` | Minimal |
| **SET_FLEX_ITEM_LAYOUT** | [SITE_OPT_FLEX_LAYOUT](SITE_OPT_FLEX_LAYOUT.md) | ⚠️ ORANGE | `StageContextBuilderAPI.addCurrentContextToRef()` | Minimal |
| **SET_FLEX_GAPS** | [SITE_OPT_FLEX_LAYOUT](SITE_OPT_FLEX_LAYOUT.md) | ⚠️ ORANGE | `StageContextBuilderAPI.addCurrentContextToRef()` | Minimal |
| **SET_STYLE** | [SITE_OPT_COMPONENT_FLOWS](SITE_OPT_COMPONENT_FLOWS.md) | ⚠️ ORANGE | SDK path via `EditorPlatformAPI` (experiment ON) | Minimal |
| **ADD_COMPONENT** | [SITE_OPT_HARMONY_ADD_COMPONENT](SITE_OPT_HARMONY_ADD_COMPONENT.md) | ⚠️ ORANGE | `EditorFlowAPI.run()` + `AddPanelDataAPI` | Low–Medium |
| **REMOVE_COMPONENT** | [SITE_OPT_COMPONENT_FLOWS](SITE_OPT_COMPONENT_FLOWS.md) | ✗ RED | Full `ComponentFlowsAPI.removeComponent()` UI flow | Low (one-line fix) |
| **SET_DATA** | [SITE_OPT_COMPONENT_FLOWS](SITE_OPT_COMPONENT_FLOWS.md) | ✗ RED | SDK via `WorkerManager` (always) | High |
| **SET_PRESET** | [SITE_OPT_SET_PRESET](SITE_OPT_SET_PRESET.md) | ✗ RED | SDK via `WorkerManager` (always) | High |
| **PIN_TO_PAGE** | [SITE_OPT_PIN_UNPIN](SITE_OPT_PIN_UNPIN.md) | ✗ RED | `PinnedToContainerFlowsAPI` (10+ RED deps) | High |
| **UNPIN_FROM_PAGE** | [SITE_OPT_PIN_UNPIN](SITE_OPT_PIN_UNPIN.md) | ✗ RED | `PinnedToContainerFlowsAPI` (10+ RED deps) | High |
| **MIGRATION** | [SITE_OPT_MIGRATION](SITE_OPT_MIGRATION.md) | ✗ RED | `LayoutConverterAPI` (DOM measurement + SharedBlocksAPI RED) | High |

**Totals: 2 GREEN · 9 ORANGE · 6 RED**

---

## add/removeComponent underlying flow analyses

The table above covers only the site-optimizer *action entry points*.
The deep isomorphic analyses of the underlying flows are in separate files:

| Flow | File | Verdict |
|---|---|---|
| `AddComponentPrivateAPI.unstable_addComponent()` | [ADD_COMPONENT_ISOMORPHIC_ANALYSIS](ADD_COMPONENT_ISOMORPHIC_ANALYSIS.md) | ✗ RED (multiple blockers; GREEN core isolated) |
| `ComponentFlowsAPI.removeComponent()` | [REMOVE_COMPONENT_ISOMORPHIC_ANALYSIS](REMOVE_COMPONENT_ISOMORPHIC_ANALYSIS.md) | ✗ RED (full UI flow; `unstable_removeComponent` is GREEN) |

---

## Shared Fixes — High Leverage

A single fix can unblock multiple actions at once:

### Fix 1 — `StageContextBuilderAPI` server-safe stub

**What:** Implement `addCurrentContextToRef(ref)` as identity function on server
(bare refs, no variant context injection).

**Unblocks:** SET_FLEX_CONTAINER_LAYOUT, SET_FLEX_ITEM_LAYOUT, SET_FLEX_GAPS
(3 actions immediately ready). Also benefits the addComponent underlying flow.

**Effort:** Minimal — one stub, no callers change.

---

### Fix 2 — Remove `EditorUIAPI.run()` / `EditorFlowAPI.run()` wrapper

**What:** Replace the UI context wrapper with `TransactionsAPI.run()` directly.
`TransactionsAPI` is isomorphic and provides transactional semantics without UI overhead.

**Unblocks:**
- All 4 animation actions (SET_*_ANIMATION) — sole blocker removed
- ADD_COMPONENT experiment-OFF path — reduces to ORANGE (AddPanelDataAPI remaining)

**Effort:** Minimal per action — replace one `.run()` call with `TransactionsAPI`.

---

### Fix 3 — Server-side SDK host context

**What:** Implement a server-side SDK host that accepts `editorPointers` directly
and executes element operations against `DocumentServicesAPI` — no `WorkerManager`,
no browser Web Worker.

**Unblocks:** SET_DATA + SET_PRESET (2 actions). Likely also future SDK-based actions.

**Effort:** High — new infrastructure. But built once, shared by both actions.

---

### Fix 4 — Force `GenericStyleAPI` path for SET_STYLE on server

**What:** Gate on server context instead of the experiment toggle. The
`GenericStyleAPI.update()` path already exists and is GREEN.

**Unblocks:** SET_STYLE (1 action immediately ready).

**Effort:** Minimal — one conditional check at the entry point.

---

### Fix 5 — `REMOVE_COMPONENT`: call `unstable_removeComponent` directly

**What:** Replace `ComponentFlowsSiteOptimizerActionsPrivateAPI.removeComponent()`
to call `ComponentHierarchyFlowsAPI.unstable_removeComponent()` instead of the full
`ComponentFlowsAPI.removeComponent()` UI flow.

**Unblocks:** REMOVE_COMPONENT (1 action).

**Effort:** Low — one-line change in the private API implementation.

---

## Reclassifications Discovered During Research

| API | Was | Now | Reason | Affects |
|---|---|---|---|---|
| `SharedBlocksAPI` | ✓ GREEN | ✗ RED | Uses `pagesDataServiceAPI.getFocusedPage/getCurrentPage()` (forbidden DS ops) + `PreviewAPI` | MIGRATION (LayoutConverterAPI L2) |
| `REMOVE_COMPONENT` (site-opt action) | implied GREEN | ✗ RED | Private API delegates to full UI flow `ComponentFlowsAPI.removeComponent()` | Site-optimizer REMOVE_COMPONENT action |

---

## Actions Grouped by Solution

### Ready today (no changes needed)
- UPDATE_PARENT
- REORDER

### One shared stub (Fix 1 — `StageContextBuilderAPI`)
- SET_FLEX_CONTAINER_LAYOUT
- SET_FLEX_ITEM_LAYOUT
- SET_FLEX_GAPS

### Remove UI wrapper (Fix 2 — `EditorUIAPI`/`EditorFlowAPI`)
- SET_ENTRANCE_ANIMATION
- SET_LOOP_ANIMATION
- SET_MOUSE_EFFECT_ANIMATION
- SET_SCROLL_ANIMATION
- ADD_COMPONENT (experiment-OFF path; AddPanelDataAPI still needs resolution)

### Single-line fixes
- SET_STYLE → force GenericStyleAPI path (Fix 4)
- REMOVE_COMPONENT → use `unstable_removeComponent` (Fix 5)

### High-effort infrastructure
- SET_DATA → server-side SDK host (Fix 3, shared with SET_PRESET)
- SET_PRESET → server-side SDK host (Fix 3, shared with SET_DATA)
- PIN_TO_PAGE → split `PinnedToContainerFlowsAPI` client/server
- UNPIN_FROM_PAGE → split `PinnedToContainerFlowsAPI` or bypass via DS directly
- MIGRATION → refactor `LayoutConverterAPI` (fix SharedBlocksAPI + remove DOM measurement)

---

## Entry Points

| Package | Repo | Entry Point | Actions |
|---|---|---|---|
| `odeditor-package-site-optimizer-actions` | Harmony | `siteOptimizerActionsContributorEntryPoint` | ADD_COMPONENT, MIGRATION, PIN_TO_PAGE, UNPIN_FROM_PAGE |
| `editor-package-native-panels` | Harmony | `SiteOptimizerAnimationEntryPoint` | SET_*_ANIMATION (×4) |
| `editor-package-flows` | REP | `componentFlowsSiteOptimizerActionsEntryPoint` | REMOVE_COMPONENT, UPDATE_PARENT, SET_STYLE, SET_DATA, REORDER |
| `editor-package-flex` | REP | `flexSiteOptimizerEntryPoint` | SET_FLEX_*_LAYOUT, SET_FLEX_GAPS |
| `responsive-presets` | REP | `presetsSiteOptimizerActionEntryPoint` | SET_PRESET |
