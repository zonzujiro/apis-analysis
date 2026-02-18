# Rules for Creating Core

## Who Defines Core

**1. Core is defined by API owners, not by consumption metrics**

The developers and owners of each API decide whether it belongs in core. Usage data (like this dashboard) informs the decision, but doesn't make it. An API owner understands the intent, roadmap, and architectural role of their API better than any automated analysis.

## Layer-Based Core Boundaries

**2. Layers below DATA_SERVICE are most likely core**

The foundational layers — SITE and EDITOR_INFRA — form the natural core. These provide the base infrastructure that any editor built on this stack needs.

- EDITOR_INFRA → Almost certainly core
- SITE → Almost certainly core

**3. DATA_SERVICE and above: case-by-case**

Higher layers can contain core APIs, but it's not automatic. The decision depends on whether the API represents shared infrastructure or editor-specific product logic.

- DATA_SERVICE → Sometimes core
- DERIVATIVE_STATE → Sometimes core
- FLOWS → Sometimes core
- UI → Never core
- VERTICAL → Never core

## Isomorphic Requirement & Client/Server Split

**5. Core must be usable on the server — therefore isomorphic**

A key requirement for the new core: it must run on the server (e.g. for SSR, headless rendering, server-side layout calculation). This means core code **cannot depend on browser APIs** (DOM, window, measurements, etc.).

**6. But some core APIs have client-only parts**

From a consumer's perspective, `LayoutHeuristicsAPI` is a core API. But its implementation has two sides:

- **Server part (isomorphic):** Pure calculations with layout data, bounding boxes, heuristic rules — no DOM needed
- **Client part (browser-only):** DOM measurements, viewport queries, real-time bounding box reads — requires browser

**Implication:** The core repo should have both **client** and **server** parts. The same API interface can have different implementation paths depending on the runtime environment. The server part is the true "core"; the client part extends it with browser capabilities.

```
        ┌──────────────────────────────┐
        │    Core API Interface        │
        │         (SlotKey)            │
        │  e.g. LayoutHeuristicsAPI,   │
        │  SuperGridDerivativeStateAPI │
        └──────────┬───────────────────┘
                   │
          ┌────────┴────────┐
          │                 │
  ┌───────▼───────┐  ┌─────▼─────────┐
  │  Server impl  │  │  Client impl  │
  │  Isomorphic,  │  │  + DOM        │
  │  pure logic   │  │  measurements │
  └───────────────┘  └───────────────┘
```

## What Does NOT Belong in Core

**7. TPA-specific logic stays outside**

SEO, A11Y, CMS implementations should not live in core, regardless of integration pattern. Core may provide _extension points_ (External APIs, contribution slots), but not the TPA logic itself.

**8. Component-type-specific editors stay outside**

VERTICAL layer packages (button editor, box editor, section editor) are consumers, not core. They implement the `ComponentEditorAPI` contract but don't define it.

## Open Questions

**Where exactly is the DATA_SERVICE boundary?**

Some DATA_SERVICE APIs are clearly core (e.g. `ComponentDataAPI`, `LayoutAPI`). Others may be editor-specific. API owners need to classify each one. Our analysis shows 63 DATA_SERVICE APIs used by Harmony out of 194 total — the unused 131 need owner review to determine if they're core, deprecated, or Studio-specific.

**How do core-features define their boundary?**

If SuperGrid is a core-feature, does that mean all SuperGrid APIs across all layers (DATA_SERVICE + DERIVATIVE_STATE + FLOWS) move together? Or only the isomorphic parts? The answer likely depends on the specific feature and its server-side requirements.

**Client/server split granularity**

Should the split happen at the package level (separate npm packages for client vs server), at the entry point level (different entry points for different runtimes), or at the implementation level (same package, conditional imports)? Each approach has tradeoffs for bundle size, DX, and testing.

**Core can have feature-specific parts**

Not everything in core is generic infrastructure. Some features are so fundamental to the layout engine that they belong in core even though they're feature-specific. Examples: SuperGrid, spx\* (spacing), Layout heuristics, Grid derivatives.

These are **core-features** — domain-specific logic that multiple editors need for correct layout behavior. They span multiple layers (DATA_SERVICE, DERIVATIVE_STATE, possibly FLOWS) but form a cohesive unit.
