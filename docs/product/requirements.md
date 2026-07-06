# Requirements

**Read this when:** scoping a milestone or deciding whether something is in v1.

Changes vs. the archived PRD v0.5 are marked **(new)**, **(changed)**, or noted inline; the
rationale for each lives in `../reviews/2026-07-architecture-review.md` and `../decisions.md`.

## Must have (v1)

### Data layer
- [ ] `DataSource` abstraction: local file (File API), HTTP range request, WebSocket stream of
      recorded data (see non-goals in `vision.md` for what "stream" means)
- [ ] **(new)** Seek backfill: `getBackfillMessages` — latest message at-or-before a time per
      topic; the seek sequence is defined around it (`../architecture/data-sources.md`)
- [ ] **(new)** Two subscription tiers: current-frame (playback) and full-range/preload
      (plots, timeline density), backed by a memory-budgeted block cache with loaded-ranges UI
- [ ] **(new)** Explicit memory budget & eviction policy — sessions are 10–200 GB; see
      `../architecture/data-sources.md#memory-budget`
- [ ] **(new)** Multi-file session model: a session is a *directory* (MCAP segments + sidecar
      MP4s + calibration JSON) composed into one timeline
- [ ] `SchemaCodec` registry (compile-once contract) with Protobuf, JSON Schema, CDR (ROS 2)
- [ ] `ContainerFormat` parsers: MCAP, ROS 2 bag (db3)
- [ ] Topic discovery, schema introspection (full field tree, not just names), random-access seek
- [ ] Message decoding off the UI thread (`../architecture/data-plane.md`)
- [ ] Microsecond-precision `Time` throughout (`../architecture/time.md`)
- [ ] **(new)** Message path grammar + evaluator + autocomplete as a core Apache-2.0 package
      (`../architecture/panel-api.md#message-paths`)

### Video pipeline (first-class)
- [ ] WebCodecs decoder: H.264, H.265, AV1 — **(changed)** VP9 dropped from v1 (not a vehicle codec)
- [ ] Keyframe index on file open for frame-accurate random access
- [ ] Multi-camera synchronized playback (up to 12 streams), sub-millisecond **timestamp
      alignment** (display sync is bounded by refresh rate — phrased precisely on purpose)
- [ ] **(new)** Decoder budget manager: hardware decode sessions are a scarce platform resource;
      visible panels win, background streams degrade (`../architecture/video-pipeline.md`)
- [ ] Zero-copy frame delivery (`VideoFrame` transferables)
- [ ] Thumbnail strip generation in a background worker
- [ ] Graceful degradation to keyframe stills when WebCodecs is unavailable
- [ ] Standalone MP4/WebM alongside logs AND video embedded in MCAP channels
      (embedded convention: `foxglove.CompressedVideo`-compatible schema)

### 3D perception viewport (first-class)
- [ ] Babylon.js, WebGPU primary / WebGL2 fallback
- [ ] Typed scene graph, pure data, no rendering deps (`../architecture/scene-graph.md`)
- [ ] **(new)** Columnar (structure-of-arrays) hot path for high-count entity types
- [ ] **(new)** Minimal coordinate-frame/calibration support: canonical frame, static
      calibration chains, interpolated ego pose (`../architecture/scene-graph.md#frames`)
- [ ] Built-in entities: oriented boxes, tracked objects (velocity + trails), lane markings /
      road boundaries, ego vehicle, planned/actual trajectories, ground grid, point clouds
- [ ] Instanced rendering (hundreds of boxes at 60 fps); temporal ghosting; entity filtering
- [ ] Camera-to-3D composition both directions (video into scene; entities onto camera views)
- [ ] BEV as a second viewport panel over the same scene, orthographic camera
- [ ] OEM-extensible custom `SceneEntity` types + renderers

### Scene mappers
- [ ] `SceneMapper` interface, runs in the decode worker
- [ ] **(new)** Built-in `foxglove-schemas` mapper (SceneUpdate, CameraCalibration,
      CompressedImage, FrameTransform, …) — makes the existing MCAP corpus light up on day one
- [ ] Built-in Autoware mapper; generic JSON-configured mapper (a fast-track tier — see
      `extension-fast-track.md`) — **(changed)** Apollo mapper dropped from v1
- [ ] **(new)** Public-dataset demo mapper + hosted sample session (nuScenes) as the first-run
      experience, doubling as integration/perf fixtures

### Panel system
- [ ] Drag-and-drop multi-panel layout (nested splits, tabs); JSON serialization
- [ ] Undo/redo for layout, panel settings, bookmarks, scene filters
      (`../architecture/state-management.md`)
- [ ] **(changed)** Panel render contract is an imperative frame callback with backpressure —
      *not* React props (`../architecture/panel-api.md`). React is for chrome/settings only.
- [ ] **(new)** Cross-panel selection/hover bus (entity, track, topic+path, time)
- [ ] **(new)** Playback control API available to panels (seek, play/pause, speed)
- [ ] Settings UI auto-generated from schema, **with a custom-UI escape hatch**
- [ ] Built-in panels: Camera View, Multi-Camera Grid, 3D Viewport (persp + BEV), Plot,
      Raw Messages, Vehicle State, Timeline + Annotation, Diagnostics
- [ ] **(changed)** Annotations/bookmarks with export (JSON/CSV) are **v1**, not v1.x — the
      vision statement, two personas, and a built-in panel all assume them

### Extension platform & fast track
- [ ] **(new)** Tiered authoring model per `extension-fast-track.md`:
      Tier 0 configure, Tier 1 compose (no-code declarative panels), Tier 2 script
      (single-file, hosted IDE), Tier 3 build (full package)
- [ ] Panel registry; built-ins and extensions use the identical public API
      (**dogfood gate** — see `../roadmap.md`)
- [ ] `create-panel` CLI + load-extension-from-dev-URL mode (Tier 3 DX, ships *before* the IDE)
- [ ] Hosted single-file IDE with live preview (Tier 2; late v1)
- [ ] **(new)** Extension supply-chain integrity: SRI hashes in manifests, verify before
      service-worker caching, versioned cache
- [ ] Scene-graph extensibility (custom entity types + Babylon renderers)

### White-label & platform
- [ ] Theme engine (JSON tokens → CSS custom properties); brand slot (logo, name, favicon,
      title); OEM build pipeline; zero upstream branding in white-label builds
- [ ] Chrome/Edge primary (WebCodecs + SAB + WebGPU); COOP/COEP headers; static-file deploys;
      File System Access API progressive enhancement; service worker offline caching
- [ ] **(new)** Per-panel error containment: a throwing panel is disabled in place with a
      reset affordance; it must never take down the app (non-negotiable for fast-track authors)

### Developer experience
- [ ] TurboRepo + pnpm monorepo; enforced dependency direction (lint-failure on violation)
- [ ] **(new)** Code-quality governance from M0: `noUncheckedIndexedAccess`,
      `exactOptionalPropertyTypes`, bans on `any` / non-null assertions / unguarded `as`
      (tests included), `readonly` public types — see `../architecture/overview.md#governance`
- [ ] **(new)** Performance budgets as CI fixtures from M2 (200 boxes × 12 cameras), not a
      beta-gate discovery
- [ ] Storybook (or equivalent) panel isolation; Babylon Inspector in dev mode

## Should have (v1.x)
- [ ] Shareable layout links (layout + data-source ref in URL)
- [ ] FlatBuffers, Cap'n Proto codecs; ROS 1 bag reader
- [ ] CAN bus / DBC panel
- [ ] HD map overlay (OpenDRIVE / Lanelet2)
- [ ] Occupancy grid / freespace rendering
- [ ] Side-by-side run comparison (two sessions, aligned timelines) — architecture must keep
      sessions first-class *now* so this stays cheap
- [ ] VP9 decode; Apollo mapper (if demand materializes)
- [ ] PWA manifest; NullEngine screenshot-regression CI
- [ ] Hosted IDE enhancements (multi-file, templates gallery growth)

## Research track (explicitly not roadmap)
- Gaussian-splat scene reconstruction from RGB-D — **(changed)** relabeled from "v1.x feature"
  to research spike: real-time splat construction without per-scene optimization is a research
  problem in 2026. The architecture keeps the door open (scene-graph/renderer split, WebGPU
  compute); the roadmap makes no promise.

## Won't have

See `vision.md` non-goals. Additionally: Foxglove layout JSON, `.foxe` extensions, and
`SceneUpdate` are not compatibility targets (but see the `foxglove-schemas` *mapper* above —
reading well-known schemas is interop, not compatibility).
