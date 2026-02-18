# TPA Integration: Studio Pattern vs Harmony Pattern

SEO, A11Y, and CMS are currently integrated differently in Studio (REP) vs Harmony. As part of migrating core from Studio to a dedicated repo, we need to decide which pattern TPAs should follow.

## Current State: Studio (REP)

**TPAs are first-class packages inside the monorepo:**

- `editor-package-tpa` — 24+ entry points spanning DATA_SERVICE, SITE, FLOWS, UI layers
- `editor-package-a11y-wizard-adapter` — 9+ entry points across 4 layers, consumes 20+ core APIs directly
- SEO logic embedded in `responsive-pages` package (`SeoAutoRedirectPrivateAPI` at FLOWS layer)
- `editor-package-tpa-events` — hooks into publish, edit-mode, breakpoint changes
- All bundled at build time with the editor, loaded synchronously

## Current State: Harmony

**TPAs live in separate repos, integrated via External APIs:**

- TPAs in own repos: `@wix/seo-repluggable`, `@wix/a11y-repluggable`, CMS packages
- Loaded dynamically via `odeditor-package-external-packages-loaders` during `site-load` sequence
- **Two-package pattern**: `odeditor-package-pages-external` (public types) + `odeditor-package-pages` (internal implementation)
- TPAs contribute via `api.contribute*()` methods, not direct API access
- 4 existing external packages: pages, site-profile, stage-indicators, add-panel

## Comparison

| Aspect          | Keep TPAs inside core repo (Studio)                                                                                 | External APIs (Harmony)                                                                           |
| --------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| **Bundle size** | All TPAs bundled = larger initial load, regardless of whether user needs them                                       | Async loading during `site-load` — smaller initial bundle, better TTI                             |
| **API surface** | TPAs access entire core API surface — hard to distinguish public vs private, easy to create accidental dependencies | Controlled surface via External API contracts — TPAs only access what's explicitly exposed        |
| **Ownership**   | Core team responsible for TPA code on-call; TPA teams need core repo access                                         | TPA teams fully own their code; core team only maintains the External API contract                |
| **Flexibility** | TPAs can use any core API method directly                                                                           | TPAs limited to what's designed into the External API; new TPA needs require External API changes |

## Key Consideration

The External API pattern fundamentally **limits what TPAs can consume**.

**In Studio, the A11Y adapter directly uses 20+ core APIs (`BreakpointsAPI`, `ColorPickerAPI`, `DialogsFlowsAPI`, `DocumentServicesAPI`, etc.).** With External APIs, it would only access what's explicitly designed into the external contract.

## Harmony's External API Architecture

```
  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────┐
  │ @wix/seo-       │  │ @wix/a11y-       │  │ CMS         │
  │ repluggable     │  │ repluggable      │  │ packages    │
  │ (separate repo) │  │ (separate repo)  │  │ (sep. repos)│
  └────────┬────────┘  └────────┬─────────┘  └──────┬──────┘
           │                    │                    │
           │     dynamic import via SequenceLoaderAPI
           │                    │                    │
  ┌────────▼────────────────────▼────────────────────▼──────┐
  │            external-packages-loaders                     │
  │        Loads bundle descriptors at site-load             │
  └────────────────────────┬─────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │                               │
  ┌────────▼────────┐            ┌─────────▼────────┐
  │ *-external pkgs │            │ internal packages │
  │ Public types +  │            │ Implementation +  │
  │ SlotKeys        │            │ contributeAPI()   │
  └─────────────────┘            └──────────────────┘
           │
           ├── pages-external
           ├── site-profile-external
           ├── stage-indicators-external
           └── add-panel-external
```
