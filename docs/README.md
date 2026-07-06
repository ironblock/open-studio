# Open Studio Documentation

Open Studio is a web-only, white-label drive-session replay and sensor-data visualization
platform for AD/ADAS teams. These docs are the product and architecture specification —
there is no code yet.

The docs are deliberately split into small, self-contained files (progressive disclosure).
Each file opens with a **"Read this when"** line. Read the index below, then only what your
task needs. When two documents disagree, [`decisions.md`](./decisions.md) is authoritative.

## Map

| Path | Contents |
|---|---|
| [`product/vision.md`](./product/vision.md) | What this is, who it's for, business model, non-goals |
| [`product/requirements.md`](./product/requirements.md) | Must / should / won't-have checklists |
| [`product/extension-fast-track.md`](./product/extension-fast-track.md) | Tiered panel/plugin authoring for teams of varying technical ability |
| [`architecture/overview.md`](./architecture/overview.md) | Package map, dependency direction, system invariants |
| [`architecture/time.md`](./architecture/time.md) | The `Time` type (branded integer microseconds) |
| [`architecture/data-sources.md`](./architecture/data-sources.md) | `DataSource` contract, sessions, backfill, preload, memory budget |
| [`architecture/data-plane.md`](./architecture/data-plane.md) | Worker topology, SharedArrayBuffer usage, what is and isn't zero-copy |
| [`architecture/scene-graph.md`](./architecture/scene-graph.md) | Typed AV scene graph, columnar hot path, frames/calibration, mappers |
| [`architecture/video-pipeline.md`](./architecture/video-pipeline.md) | WebCodecs decoding, keyframe index, decoder budget, multi-stream sync |
| [`architecture/panel-api.md`](./architecture/panel-api.md) | Panel descriptor, render contract, message paths, selection bus |
| [`architecture/extensions.md`](./architecture/extensions.md) | Extension packaging, registry, security, hosted IDE, CLI |
| [`architecture/rendering-3d.md`](./architecture/rendering-3d.md) | Babylon.js engine layer, entity renderers, BEV / multi-view |
| [`architecture/state-management.md`](./architecture/state-management.md) | Zustand stores, undo/redo, the control-plane invariant |
| [`roadmap.md`](./roadmap.md) | Milestones, dogfood gates, changes vs. the archived plan |
| [`decisions.md`](./decisions.md) | Decision log (authoritative; supersessions marked) |
| [`research/`](./research/) | Point-in-time technology research (inputs, not specs) |
| [`reviews/`](./reviews/) | Architecture reviews (point-in-time records) |
| [`archive/`](./archive/) | Superseded documents, retained verbatim for history |

## Reading paths

- **Implementing a panel or extension** → `product/extension-fast-track.md` → `architecture/panel-api.md` → `architecture/extensions.md`
- **Data layer / pipeline work** → `architecture/time.md` → `architecture/data-sources.md` → `architecture/data-plane.md`
- **3D / perception visualization** → `architecture/scene-graph.md` → `architecture/rendering-3d.md`
- **Video work** → `architecture/video-pipeline.md` (plus `data-plane.md` for threading)
- **Planning / prioritization** → `product/requirements.md` → `roadmap.md`
- **"Why is it this way?"** → `decisions.md`, then `reviews/2026-07-architecture-review.md`

## Provenance

The original monolithic PRD (v0.5, April 2026) and M0 plan live in `archive/`. In July 2026
they were superseded by this structure, which folds in:

1. The findings of [`reviews/2026-07-architecture-review.md`](./reviews/2026-07-architecture-review.md)
   (a retrospective on Foxglove Studio's actual architecture and a critique of the PRD).
2. A new product requirement: a **fast track** enabling many teams — of widely varying
   technical ability — to build their own panels and plugins as easily as possible. See
   `product/extension-fast-track.md`. This requirement postdates (and revises one
   recommendation of) the July 2026 review.

## Conventions

- Docs are specifications: normative language ("must", "never") is intentional.
- Interface sketches are illustrative TypeScript; exact signatures are settled in code review,
  but the *shape* (sync/async, batching, ownership, generics) is normative.
- `decisions.md` wins conflicts; update it in the same change that alters any spec.
