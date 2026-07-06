# Architecture Overview

**Read this when:** starting any implementation work, or placing new code in the monorepo.

## System invariants

These hold everywhere; individual docs elaborate:

1. **Control plane vs. data plane.** UI state (layout, settings, selection, playback controls)
   lives in Zustand stores and React. Sensor-rate data (messages, video frames, scene columns)
   **never** enters a store, a context, or React props — it flows through the worker data plane
   and imperative panel callbacks. See `data-plane.md`, `panel-api.md`, `state-management.md`.
2. **The core never imports a format.** Core speaks typed channels, `Time`, and byte buffers;
   codecs/containers are injected leaf packages. See `data-sources.md`.
3. **`scene-graph` is pure data.** No Babylon imports outside `renderer/*`. Mappers and
   extensions compile without a rendering dependency. See `scene-graph.md`.
4. **Built-ins are extensions.** Every built-in panel consumes only the public
   `@open-studio/panel-api` (+ published leaf packages). No privileged internal panel API
   exists. Enforced by the dogfood gate in `../roadmap.md`.
5. **Extension-reachable packages are Apache-2.0.** `panel-api`, `scene-graph`,
   `message-path`, codecs, containers. BSL applies to the platform, never to what OEM code
   imports. See `../product/vision.md`.
6. **Seek is the dominant operation.** Every pipeline stage must define its behavior across a
   seek (epoch invalidation — see `data-plane.md`), and every panel must render correctly
   immediately after one (backfill — see `data-sources.md`).

## Package map

```
open-studio/
├── apps/web/                       # Vite entrypoint — the only app target
├── packages/
│   ├── core/
│   │   ├── time/                   # Time type + utilities (leaf, Apache-2.0)
│   │   ├── message-path/           # Path grammar, evaluator, autocomplete (leaf, Apache-2.0)
│   │   ├── data-source/            # DataSource/Session contracts + base impls
│   │   ├── message-pipeline/       # Worker topology, epoch arenas, subscriptions
│   │   ├── video-pipeline/         # WebCodecs decode, keyframe index, decoder budget
│   │   ├── scene-graph/            # Typed AV scene graph + frames (pure data, Apache-2.0)
│   │   ├── scene-mapper/           # SceneMapper contract + built-in mappers
│   │   ├── panel-api/              # Panel descriptor, render contract (Apache-2.0)
│   │   ├── layout-engine/          # Layout tree, serialization, DnD (framework-agnostic)
│   │   ├── app-state/              # Zustand stores + undo/redo
│   │   ├── timeline/               # Global clock, playback, seek orchestration
│   │   ├── selection/              # Cross-panel selection/hover bus
│   │   ├── theme/                  # Token resolution, CSS custom property injection
│   │   └── file-access/            # File API + File System Access abstraction
│   ├── codecs/                     # protobuf | json-schema | ros2-cdr   (leaves)
│   ├── containers/                 # mcap | rosbag2                      (leaves)
│   ├── video/                      # demuxer | webcodecs-decoder | still-fallback | frame-index
│   ├── renderer/                   # babylon-core | scene-renderer | entity-renderers | camera-projection
│   ├── panels/                     # camera-view | multi-camera-grid | viewport-3d | plot |
│   │                               # raw-messages | vehicle-state | timeline-annotation | diagnostics
│   ├── ui/                         # design-system | layout-ui | panel-chrome
│   └── extension-host/             # registry | sandbox | dev-environment (hosted IDE)
└── tools/                          # create-panel | build-theme | eslint-config
```

## Dependency direction (lint-enforced)

```
apps/web → ui → panels → renderer → core → codecs / containers / video
                  │                   │
                  │                   ├── scene-graph  (pure data — no Babylon)
                  │                   ├── scene-mapper → scene-graph
                  └── extension-host → core

renderer → scene-graph (types only) + @babylonjs/* (external)
codecs / containers / video / time / message-path → no internal deps (leaves)
panels → panel-api ONLY among core packages (invariant 4)
```

Violations fail CI via a custom ESLint rule in `tools/eslint-config`.

## Governance

Code-level quality rules, active from M0 (rationale: the review found Foxglove's TypeScript
decay was cumulative and unenforced):

- `tsconfig`: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`;
  `isolatedDeclarations` on publishable leaves.
- ESLint (tests included — test suites are where `any` breeds): no `any`, no non-null
  assertion, `as` only inside designated typed-guard modules, `prefer-readonly` public types.
- Prefer discriminated unions + pure functions over class hierarchies; data types are
  `readonly`; APIs that would force callers to cast (e.g., `unknown` payloads without a typed
  registration path) are rejected at design time — see `scene-graph.md#typing` for the pattern.
- Per-frame allocation is a review concern on the data plane: hot paths use preallocated
  typed arrays and scratch objects (`data-plane.md#columnar`).

## Performance budgets (CI fixtures from M2)

- Reference scene: 200 oriented boxes + trails + 2 lane sets + 1 point cloud (100k pts),
  12 camera tiles, 30 fps playback.
- UI-thread frame job ≤ 4 ms on reference hardware; zero per-frame heap allocation in the
  scene-column → thin-instance path; draw calls counted via NullEngine in CI.
- Numbers are budgets, not aspirations: the fixture fails the build when exceeded.
