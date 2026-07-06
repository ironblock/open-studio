# 3D Rendering

**Read this when:** working in `renderer/*`, adding an entity renderer, or touching the 3D
viewport panel.

## Engine: Babylon.js

WebGPU primary, WebGL2 fallback. The decisive factors (see `../decisions.md` D-04): written
in TypeScript, production WebGPU including compute, `NullEngine` for headless CI,
thin instances, `engine.registerView()` multi-view, built-in Inspector.

Honesty note carried from the review: Three.js's `WebGPURenderer` reached production quality
in 2025 — the stale "experimental" claim in the archived PRD is not part of the rationale.
Babylon wins on NullEngine CI, TS-nativeness, and multi-view ergonomics, and because we
manage the render lifecycle ourselves (no React reconciler in the 3D loop — that constraint
rules out the `@react-three/fiber` ecosystem advantage Three would bring).

## Pipeline

```
SceneSnapshot (pure data)  →  scene-renderer (diff + map)  →  Babylon nodes  →  GPU
```

- **Columnar fast path:** `BoundingBoxColumns` (`scene-graph.md#columnar`) feed thin-instance
  matrix buffers built from typed-array views over arena memory
  (`data-plane.md#columnar`) — one box mesh, N instances, 1 draw call for 200+ boxes, zero
  per-frame heap allocation. Per-instance color/alpha via instance attributes
  (class color, confidence, ghost fade).
- **Object path:** diff each snapshot against the previous — create/update-in-place/dispose.
  Entity renderers are pluggable per type (`registerEntityRenderer`, typed via
  `scene-graph.md#typing`).
- **Point clouds:** GPU point rendering fed directly from arena `Float32Array`s
  (positions/colors/intensity), configurable point size, intensity color ramps.
- **Polylines** (lanes, boundaries, trajectories): line meshes, updated in place.
- **Temporal ghosting:** ring buffer of recent column generations; ghosts render as
  additional thin instances with age-based alpha — the columnar form makes the trail window
  nearly free.

The UI-thread frame job must stay within the 4 ms budget: read columns → update instance
buffers → draw. Anything heavier belongs in the decode worker (`data-plane.md#where-rendering-runs`).

## Multi-view: perspective + BEV

BEV is not a separate panel type: it is a second `viewport-3d` panel instance over the **same
Babylon scene** with an orthographic top-down camera, via `engine.registerView(canvas, camera)`.
Zero mesh/material duplication; filtering and ghosting apply to all views automatically.

Known ceilings (accepted for v1, benchmark before promising more):

- Babylon renders views through one working canvas and **blits per view per frame**; cost
  scales with view count and resolution.
- All views share the working canvas resolution; mixed-DPR panel layouts need explicit
  handling.
- Per-view input routing (orbit vs. BEV pan) and picking are managed by `babylon-core`'s
  camera-rig layer.
- Target: 2–4 simultaneous viewports. The fallback design (multiple viewports composited on
  one canvas positioned over the layout) is uglier but cheaper — keep it viable.

Camera modes: orbit, follow-ego, top-down BEV (area size, north-up vs. heading-up lock).

## Picking & selection

Pointer picking on instances resolves to (entity/track) via the instance→column-index map and
publishes to the selection bus (`panel-api.md#selection-bus`). Hover uses the same path on a
throttle. Selected/hovered instances get highlight treatment in the instance attributes —
no scene-graph mutation.

## CI

`NullEngine` runs the full scene-renderer against fixture snapshots headlessly: draw-call
counts, per-frame allocation assertions, and (v1.x) screenshot regression via browser-mode
Vitest. The reference-scene budget fixture (`overview.md#performance-budgets`) lives here.
