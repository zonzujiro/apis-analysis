# Isomorphic Research

Research and analysis for the isomorphic migration effort — making core APIs runnable on the server.

---

## Start Here

### [ISOMORPHIC_CLASSIFICATION_RULES.md](./ISOMORPHIC_CLASSIFICATION_RULES.md)
Rules and heuristics for classifying APIs as isomorphic or client-only.
Use this as a reference before starting any API research.
Includes: always-red categories, always-green categories, ambiguous patterns, layer heuristics, known root causes, and a quick checklist.

---

## Analysis

### [ISOMORPHIC_CONFLICTS.md](./ISOMORPHIC_CONFLICTS.md)
Automated 1-hop analysis of 2,028 REP entry points.
Found 205 core-candidate APIs (SITE → FLOWS) that directly depend on browser-only APIs.
Key finding: `PreviewAPI` (161 refs) and `ComponentMeasureAPI` (102 refs) are the root causes of most conflicts.

---

## Case Studies

### [REMOVE_COMPONENT_FLOW.md](./REMOVE_COMPONENT_FLOW.md)
Full dependency graph of `ComponentFlowsAPI.removeComponent()` across 3 levels:
- Level 1: `removeComponent()` — 20 direct API dependencies
- Level 2: `unstable_removeComponent()` — draft guard + strategy dispatch
- Level 3: `getRemoveComponentStrategy()` — 6 branches by component type

### [REMOVE_COMPONENT_ISOMORPHIC_ANALYSIS.md](./REMOVE_COMPONENT_ISOMORPHIC_ANALYSIS.md)
Isomorphic breaking point analysis of the `removeComponent` flow.
Identifies which dependencies are red/ambiguous/green, diagnoses the structural problem,
and proposes a concrete client/server split.
Key finding: the flow is fundamentally client-only, but `DocumentServicesAPI.components.remove()` at its core is isomorphic.
