# Additional API Research Notes

Research findings for APIs outside the original site-optimizer action scope.
Each finding has been applied to the main dependency graph.

---

## 1. EditorPlatformAPI — `getStyle()` and `getData()`

**Verdict: 🟢 GREEN**

These two *read* methods bypass the `WorkerManager → SDKHostContext` chain entirely.
They call into existing REP data APIs directly:

- `getStyle(pointer)` → `componentStyleAPI.getInBreakpoint()` via `breakpointsDerivativeStateAPI` — pure data read, no browser deps.
- `getData(pointer)` → `componentDataAPI.get()` → `ds.components.data.get()` — pure DS read. Uses `previewAPI.memoizeForSiteUpdates()` as a memoization wrapper, but the underlying operation is a data read, not a preview/render operation.

**Contrast with RED methods:** `setStyle`, `setData`, `setPreset` call `getWorkerManager()` and dispatch to a browser Web Worker SDK host → RED.

**Impact on dep graph:** `EditorPlatformAPI` entry in the ORANGE table updated to note the method-level split. SET_STYLE's experiment-ON path (which uses `EditorPlatformAPI` for reads, not writes) is potentially GREEN for the *read* portion.

---

## 2. TransformationsDerivativeStateAPI

**Verdict: 🟢 GREEN**

No `getTransformation()` method exists. The public API surface is:

- `getFullDataForCurrentContext(pointer)` — returns full transformation data for current context
- `getPointerFullData(pointer)` — returns raw pointer data
- `unstable_getEffectiveTransformations(pointer)` — effective transformations

All declared deps are `DATA_SERVICE` / `EDITOR_INFRA` layer — no browser deps, no stage context. Fully server-safe.

**Not in current dep graph** — not a direct dependency of any site-optimizer action analyzed. Included here for completeness.

---

## 3. ComponentLayoutDerivativeStateAPI — `getLayout()`

**Verdict: 🟠 ORANGE** *(reclassified from GREEN)*

No `getLayout()` method exists. The actual public surface:

| Method | Notes |
|--------|-------|
| `getScopedLayout(pointer)` | Takes context-qualified pointer; likely server-safe |
| `getEffectiveLayout(pointer)` | Computes effective layout from scoped + defaults; takes pointer |
| `unstable_getCurrentEffectiveLayout(comp)` | Uses `stageContextBuilderAPI.addCurrentContextToRef(comp)` — ORANGE |
| `visibility.isHidden(pointer)` | |
| `pin.isPinnedToScreen(pointer)` | deprecated |
| `getSubLayoutKeysToConsiderAsOverrides(pointer)` | |

**Root cause:** Entry point declares `StageContextBuilderAPI` (RED). The "current" variant resolves via stage context.

**Fix:** Stub `StageContextBuilderAPI.addCurrentContextToRef()` as identity on server. Same stub that unblocks the flex actions.

**Dep graph update:** Moved from GREEN (#16) to ORANGE (#11). All SET_FLEX_* action entries updated.

---

## 4. LayoutConverterAPI

**Verdict: ✗ RED** — already fully documented.

`LayoutConverterAPI` (Harmony, `editor-package-layout-converter`, FLOWS layer) is the only API that uses DOM measurement (`ComponentMeasureAPI`). It is reached via the MIGRATION site-optimizer action.

See: `SITE_OPT_MIGRATION.md` and the dep graph (ORANGE section #10 / LayoutConverterAPI entry).

---

## 5. ComponentReactionsDerivativeStateAPI — `getCurrentEffective()` / `getCurrentEffect()`

**Verdict: 🟠 ORANGE** *(reclassified from GREEN)*

`getCurrentEffect()` does **not exist**. The actual method is `getCurrentEffective(pointer)`.

`getCurrentEffective(pointer)` calls `stageContextBuilderAPI.addCurrentContextToRef(pointer)` to inject current variant/breakpoint context → ORANGE.

All other methods (reaction queries, effect lookups) do not use `StageContextBuilderAPI` → GREEN.

**Impact on AnimationFlowsPrivateAPI:** The animation flow uses `ComponentReactionsDerivativeStateAPI` but only calls GREEN methods (not `getCurrentEffective`). `AnimationFlowsPrivateAPI` remains effectively GREEN for the animation use case.

**Fix:** Same `StageContextBuilderAPI` identity stub → fully GREEN.

**Dep graph update:** Moved from GREEN (#23) to ORANGE (#12). AnimationFlowsPrivateAPI note updated.
