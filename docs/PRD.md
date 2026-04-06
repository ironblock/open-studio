# Open Studio — Product Requirements Document

**Version:** 0.5 (Final Draft)
**Date:** April 2026
**Status:** Ready for Implementation

---

## 1. Vision

Open Studio is a white-label drive session replay and sensor data visualization platform, purpose-built for autonomous driving and ADAS development teams. It is designed to be OEM-branded and deployed by major automakers as their internal tooling for reviewing, annotating, and debugging vehicle drive sessions.

It runs entirely in the browser — no Electron, no native install. It deploys to a developer's workstation, an internal web server, or directly on a vehicle's compute platform.

**One-line pitch:** "The drive session workbench your OEM customers ship under their own brand."

### 1.1 Business Model

Open Studio is developed as a source-available product. The core platform is distributed under a Business Source License (BSL 1.1) that permits inspection, modification, and self-hosting for internal use, but requires a commercial license for redistribution or SaaS deployment. Revenue comes from:

- **Commercial licenses** sold to automakers and Tier 1 suppliers for white-label deployment
- **Support contracts** (SLA-backed support, integration services, custom panel development)
- **Training and onboarding** for OEM engineering teams adopting the platform

Individual leaf packages (codecs, container parsers) are Apache-2.0 to maximize ecosystem adoption.

---

## 2. Motivation & Prior Art Critique

### 2.1 Why reimplement?

Foxglove Studio was relicensed away from open source after building significant community investment. The last open source release reveals architectural decisions that were reasonable for a startup shipping fast, but create real problems — particularly for the autonomous driving use case:

### 2.2 Critique of Foxglove's Architecture

#### Problem 1: Data format coupling

Foxglove treats MCAP as a first-class citizen and everything else as an adapter bolted on after the fact:

- **The message pipeline assumes MCAP's topic/schema model.** Internal message types (`MessageEvent`, `TopicStats`) mirror MCAP's structure directly. Any non-MCAP source must shim itself into this shape, which loses metadata and forces unnecessary serialization boundaries.
- **Schema resolution is hardcoded.** Foxglove's `MessageReader` has baked-in knowledge of ROS message definitions, Protobuf descriptors, and JSON Schema — but the resolution logic isn't pluggable. Adding a new schema format means touching core code.
- **No abstract "data source" contract.** Data sources (file, WebSocket, Foxglove WebSocket protocol, ROS bridge) each implement their own ad-hoc interface rather than conforming to a shared trait.

**Implication for AV use case:** Every major automaker has their own log format (often Protobuf or FlatBuffers over a custom container). Bolting these onto MCAP's model creates impedance mismatch and performance overhead.

**Our approach:** Define a `DataSource` interface and a `SchemaCodec` registry as the foundation. The core never imports a specific format — it only speaks in terms of typed channels, timestamps, and raw byte buffers. Format support is injected via packages. OEM customers can add proprietary codecs without forking the core.

#### Problem 2: React anti-patterns

Foxglove's React codebase accumulated several patterns that hurt performance and maintainability:

- **Deep context provider nesting.** The app wraps panels in 10+ nested context providers (`PlayerContext`, `PanelContext`, `MessagePipelineContext`, `LayoutContext`, etc.). Each provider boundary is a potential re-render trigger, and the nesting makes it nearly impossible to reason about which state change causes which re-render.
- **Non-memoized selectors on large state objects.** `useMessagePipeline(selector)` passes the entire pipeline state through a selector on every frame. If the selector isn't referentially stable (and many aren't), every subscribed component re-renders on every tick — even if the data it cares about didn't change.
- **Synchronous message processing on the main thread.** Message deserialization and panel data extraction happen synchronously in React's render cycle. For large drive sessions (hundreds of MB), this blocks the UI thread and causes visible jank.
- **Prop drilling through layout tree.** The panel layout system (based on `react-mosaic`) passes configuration and callbacks through the component tree rather than using a store. This couples the layout engine to React's reconciliation, making it fragile during drag operations.
- **Tight coupling between "panel" and "React component."** A Foxglove panel IS a React component. There's no intermediate abstraction — the panel's data subscriptions, lifecycle, settings schema, and rendering are all tangled into a single component tree.

**Our approach:** Zustand with `zundo` middleware for fine-grained subscriptions and built-in undo/redo. Message processing in a dedicated worker with SharedArrayBuffer for zero-copy data sharing. Panels defined as declarative descriptors, not React components.

#### Problem 3: No video pipeline

Foxglove's `ImagePanel` treats video as a sequence of individually-compressed JPEG/PNG frames. There is no concept of codec-compressed video streams (H.264, H.265, AV1), frame-accurate seeking, multi-camera synchronized playback, or hardware-accelerated decoding via WebCodecs.

For a camera-first autonomy world, this is a fundamental gap, not a missing feature.

**Our approach:** A dedicated `VideoDecoder` pipeline built on the WebCodecs API, with a keyframe index for random access, multi-stream synchronization tied to the global timeline, and zero-copy frame delivery to canvas/WebGL via `VideoFrame` transferable objects. Graceful degradation to per-frame JPEG/PNG stills when WebCodecs is unavailable.

#### Problem 4: 3D scene built for robotics, not AV perception

Foxglove's 3D panel is a general-purpose robotics visualizer. It renders TF trees, URDF models, occupancy grids, and ROS marker arrays — primitives designed for robot arms and mobile robots, not AV perception debugging.

What an AV validation engineer needs to see in 3D:

- **Perception outputs:** oriented 3D bounding boxes with class labels, confidence scores, and track IDs
- **Tracked object trajectories:** velocity vectors, predicted future paths, track history trails
- **Road model:** lane markings, road boundaries, crosswalks, stop lines, traffic signs as wireframe polylines
- **Planning outputs:** ego planned vs. actual path, waypoints, decision points
- **Camera-to-world projection:** video as textures in 3D, or 3D entities projected onto 2D camera views
- **Multi-frame temporal visualization:** ghosting / trail effects across a configurable time window

Foxglove's `SceneUpdate` message and Three.js pipeline don't model these as first-class concepts. Three.js itself lacks production WebGPU support, a NullEngine for headless CI, and is written in JavaScript with bolted-on TypeScript definitions.

**Our approach:** Babylon.js as the 3D engine, with a typed scene graph layer (`@open-studio/scene-graph`) that defines AV-specific primitives as a pure data structure — no rendering dependencies. See §5.6.

#### Problem 5: Extension system limitations

Foxglove's extension API ships as a sandboxed iframe with a narrow message-passing interface. Extensions can't share state, rendering primitives, or the 3D scene graph.

**Our approach:** Panels are the extension primitive. Extensions can inject custom `SceneEntity` types into the 3D scene graph. Development happens in a hosted browser-based IDE (see §6). Each extension is a single file exporting a `PanelDescriptor`.

#### Problem 6: Unnecessary Electron dependency

**Our approach:** Web-only. Files opened via `<input type="file">` + drag-and-drop. File System Access API for directory access on Chrome/Edge. No Electron.

#### Problem 7: Monorepo but not modular

**Our approach:** Strict dependency direction enforced by tooling. No cycles. Every package publishable to npm independently.

---

## 3. Target Users

| Persona | Description | Key need |
|---|---|---|
| **AV validation engineer** | Reviews drive session logs, looking for perception failures, planning errors, edge cases | Multi-camera synced replay, 3D perception overlay, timeline scrubbing, annotation |
| **ADAS calibration engineer** | Tunes sensor parameters, reviews camera alignment, compares runs | Camera-to-3D projection, synchronized multi-channel replay |
| **Perception ML engineer** | Debugs object detection/tracking models, compares inference results | 3D bounding boxes with confidence/class, track trails, side-by-side comparison |
| **AV tooling developer (OEM)** | Builds internal panels for fleet-specific data | Hosted extension IDE, custom panel SDK, scene graph extensions, white-label theming |
| **AV program manager** | Reviews flagged incidents, shares annotated clips | Shareable layouts, bookmark/annotation, export |
| **In-vehicle reviewer** | Reviews drive data on the vehicle compute platform | Lightweight browser UI, no install |

---

## 4. Requirements

### 4.1 Must Have (v1)

**Data layer:**
- [ ] `DataSource` abstraction with implementations for: local file (via File API), HTTP range request, WebSocket stream
- [ ] `SchemaCodec` registry with built-in codecs for: Protobuf, JSON Schema, and CDR (ROS 2)
- [ ] `ContainerFormat` parsers for: MCAP, ROS 2 bag (db3/sqlite)
- [ ] Video container support: standalone MP4/WebM alongside logs, AND video streams embedded in MCAP channels
- [ ] Topic discovery, schema introspection, and random-access seeking within files
- [ ] Message decoding in a Web Worker using SharedArrayBuffer + Atomics for zero-copy data sharing (see §5.4)
- [ ] High-resolution timestamps throughout the pipeline: microsecond precision using `DOMHighResTimeStamp` (`performance.now()` / `performance.timeOrigin`) for all synchronization and event ordering (see §5.7)

**Video pipeline (first-class):**
- [ ] WebCodecs-based decoder supporting H.264, H.265, VP9, and AV1
- [ ] Keyframe index construction on file open for frame-accurate random access
- [ ] Multi-camera synchronized playback (up to 12 streams) with sub-millisecond sync precision
- [ ] Zero-copy frame delivery: `VideoFrame` objects transferred to canvas/WebGL
- [ ] Thumbnail strip generation (background worker produces keyframe thumbnails for timeline scrubber)
- [ ] Graceful degradation: keyframe still images (JPEG/PNG) when WebCodecs is unavailable
- [ ] Codec licensing disclaimer: deploying organization responsible for patent compliance

**3D perception viewport (first-class):**
- [ ] Babylon.js rendering engine with WebGPU backend (WebGL2 fallback)
- [ ] Typed scene graph (`@open-studio/scene-graph`) with AV perception primitives (see §5.6)
- [ ] Built-in scene entity types: oriented 3D bounding boxes, tracked objects (velocity vector + track trail), lane markings / road boundaries (ground-plane polylines), ego vehicle, planned/actual trajectory paths, ground plane grid
- [ ] Instanced rendering for large entity counts (hundreds of bounding boxes per frame at 60fps)
- [ ] Camera-to-3D composition: project video frames as textures into the 3D scene, or project 3D entities back onto 2D camera views
- [ ] Temporal visualization: ghosting / trail mode over configurable time window
- [ ] Scene entity filtering: toggle visibility by class, confidence threshold, track ID
- [ ] OEM-extensible: custom `SceneEntity` types registered by extensions
- [ ] Orbit, pan, follow-ego, and top-down camera modes
- [ ] Bird's-eye-view (BEV): second panel instance of the same 3D Viewport type, rendering the shared Babylon.js scene via an orthographic camera (see §5.6.6)
- [ ] Point cloud rendering via Babylon.js particle system (for teams with LiDAR data)

**Scene graph mapping layer:**
- [ ] Per-customer `SceneMapper` interface: transforms OEM-specific perception messages into the typed scene graph (see §5.6.7)
- [ ] Built-in mappers for common formats: Autoware, Apollo (Protobuf), and a generic JSON mapper
- [ ] Mapper is an extension point — OEM customers implement their own `SceneMapper` as part of their integration

**Panel system:**
- [ ] Drag-and-drop multi-panel layout (nested splits, tabs)
- [ ] Layout serialization/deserialization (JSON)
- [ ] Undo/redo for all user actions (layout changes, panel settings, timeline bookmarks, scene filter state) via Zustand temporal middleware (see §5.5)
- [ ] Panel settings UI (auto-generated from JSON Schema descriptor)
- [ ] Built-in panels: Camera View (video), Multi-Camera Grid, 3D Viewport (perspective + BEV), Plot (time series), Raw Messages, Vehicle State (speed/steering/brake overlay), Timeline + Annotation, Diagnostics/Log

**White-label system:**
- [ ] Theme engine: all visual tokens configurable via JSON theme file
- [ ] Branding slot: logo, product name, favicon, window title — injected via config
- [ ] OEM build pipeline: `theme.json` + optional CSS overrides → fully branded build
- [ ] No Open Studio branding visible in a white-label build

**Extension API:**
- [ ] Panel descriptor format: single-file TypeScript module exporting a `PanelDescriptor`
- [ ] Panel registry (register built-in + third-party panels)
- [ ] Scene graph extensibility: extensions can register custom `SceneEntity` types with custom Babylon.js renderers
- [ ] Shared rendering primitives available to extensions (video canvas, Babylon.js scene, plot library, table)
- [ ] Hosted single-file extension IDE with live preview (see §6)

**Platform:**
- [ ] Web app (Chrome/Edge primary — WebCodecs + SharedArrayBuffer + WebGPU)
- [ ] File access via `<input type="file">` + drag-and-drop (universal)
- [ ] File System Access API for directory access and file watching (Chrome/Edge progressive enhancement)
- [ ] Deployable as static files behind any web server, or served from vehicle compute platforms
- [ ] Cross-Origin Isolation headers (COEP/COOP) required for SharedArrayBuffer
- [ ] Service worker for caching application assets and installed extensions for offline use

**Developer experience:**
- [ ] Monorepo with clear package boundaries (TurboRepo + pnpm)
- [ ] `dev` command that starts the web app with hot reload
- [ ] Storybook or equivalent for developing panels in isolation
- [ ] Babylon.js Inspector integration in dev mode (toggle via keyboard shortcut)
- [ ] CI that runs lint, typecheck, test, and build for affected packages only

### 4.2 Should Have (v1.x)

- [ ] Annotation / bookmark system with export (JSON, CSV)
- [ ] Shareable layout links (encode layout + data source reference in URL)
- [ ] FlatBuffers and Cap'n Proto codecs
- [ ] ROS 1 bag reader (for legacy datasets)
- [ ] CAN bus / DBC panel (common in automotive)
- [ ] HD map overlay (OpenDRIVE / Lanelet2) rendered into the 3D scene
- [ ] Occupancy grid / freespace rendering in 3D viewport
- [ ] Side-by-side model comparison mode (two scene graphs, split-view or overlay)
- [ ] PWA manifest for "Install as app" desktop shortcut
- [ ] NullEngine-based CI screenshot regression tests for 3D viewport
- [ ] 3D scene reconstruction via Gaussian splatting (using video frames + depth data, see §5.6.8)

### 4.3 Won't Have (explicit non-goals)

- Electron or any native desktop wrapper
- Cloud backend or managed storage (OEMs bring their own infra)
- User accounts or authentication (delegated to OEM's identity layer)
- Proprietary data format that replaces MCAP
- Canonical wire format for the scene graph (perception stacks are too diverse — we provide a mapping layer instead)
- Real-time live vehicle telemetry (visualization of recorded sessions only)
- Mobile support
- Generic robotics panels with no automotive relevance (ROS TF tree, URDF viewer, joystick teleop)
- Physics simulation or ego vehicle control
- Bundled video codec patent licenses (deployer's responsibility)

---

## 5. Architecture

### 5.1 Package Map

```
open-studio/
├── apps/
│   └── web/                        # Vite entrypoint — the only app target
│
├── packages/
│   ├── core/
│   │   ├── data-source/            # DataSource interface + base implementations
│   │   ├── message-pipeline/       # Worker-based message processing pipeline
│   │   ├── video-pipeline/         # WebCodecs decoder, keyframe index, multi-stream sync
│   │   ├── scene-graph/            # Typed AV scene graph (pure data, no rendering)
│   │   ├── scene-mapper/           # SceneMapper interface + built-in mappers (Autoware, Apollo, generic JSON)
│   │   ├── panel-api/              # Panel descriptor, registry, lifecycle
│   │   ├── layout-engine/          # Layout tree, serialization, drag-and-drop (framework-agnostic)
│   │   ├── app-state/              # Zustand stores + zundo undo/redo middleware
│   │   ├── timeline/               # Global clock, high-resolution sync, playback control, seek
│   │   ├── theme/                  # Theme token resolution, CSS custom property injection
│   │   └── file-access/            # Abstraction over File API + File System Access API
│   │
│   ├── codecs/
│   │   ├── protobuf/               # Protobuf descriptor → decoder
│   │   ├── json-schema/            # JSON Schema validation + typed access
│   │   ├── ros2-cdr/               # CDR (Common Data Representation) codec
│   │   └── flatbuffers/            # (v1.x) FlatBuffers codec
│   │
│   ├── containers/
│   │   ├── mcap/                   # MCAP reader (random access, indexed, streaming)
│   │   └── rosbag2/                # ROS 2 bag (SQLite db3) reader
│   │
│   ├── video/
│   │   ├── demuxer/                # MP4/WebM demuxer → extract NAL units / codec chunks
│   │   ├── webcodecs-decoder/      # WebCodecs VideoDecoder wrapper with frame queue
│   │   ├── still-fallback/         # Keyframe extractor → JPEG/PNG stills for fallback
│   │   └── frame-index/            # Keyframe index builder for random access seeking
│   │
│   ├── renderer/
│   │   ├── babylon-core/           # Babylon.js engine setup, WebGPU/WebGL2 init, camera rigs, multi-viewport
│   │   ├── scene-renderer/         # SceneGraph → Babylon.js mesh/material mapping (diff-based)
│   │   ├── entity-renderers/       # Per-entity-type renderers (bbox, polyline, trail, point cloud, etc.)
│   │   └── camera-projection/      # Camera intrinsics/extrinsics, video-to-3D and 3D-to-2D projection
│   │
│   ├── panels/
│   │   ├── camera-view/            # Single camera video panel (WebCodecs → canvas, optional 3D overlay)
│   │   ├── multi-camera-grid/      # Synchronized N-camera grid layout
│   │   ├── viewport-3d/            # 3D perception viewport (Babylon.js, supports perspective + BEV via camera mode)
│   │   ├── plot/                   # Time series plot (uPlot-based)
│   │   ├── raw-messages/           # JSON message inspector
│   │   ├── vehicle-state/          # Speed / steering / brake / gear overlay
│   │   ├── timeline-annotation/    # Timeline scrubber + annotation markers
│   │   └── diagnostics/            # Structured log viewer
│   │
│   ├── ui/
│   │   ├── design-system/          # Tokens, primitives (Button, Input, etc.), themeable
│   │   ├── layout-ui/              # React components for the layout engine
│   │   └── panel-chrome/           # Panel wrapper (toolbar, settings, drag handle)
│   │
│   └── extension-host/
│       ├── registry/               # Extension manifest format, discovery, lazy loading, caching
│       ├── sandbox/                # Optional iframe sandbox for untrusted extensions
│       └── dev-environment/        # Hosted IDE components (Monaco editor, esbuild-wasm bundler, live preview)
│
├── tools/
│   ├── create-panel/               # CLI: scaffold a new panel extension
│   ├── build-theme/                # CLI: compile theme.json → CSS + assets for white-label
│   └── eslint-config/              # Shared lint rules + dependency direction enforcement
│
├── turbo.json
├── pnpm-workspace.yaml
└── package.json
```

### 5.2 Dependency Direction (enforced)

```
apps/web → ui → panels → renderer → core → codecs / containers / video
                   │                   │
                   │                   ├── scene-graph (pure data, no Babylon dep)
                   │                   ├── scene-mapper → scene-graph
                   │                   │
                   └── extension-host → core

renderer → scene-graph (data types only)
renderer → @babylonjs/* (external)

codecs / containers / video → (no internal deps — leaf)
```

**Critical boundary:** `core/scene-graph` is a pure TypeScript data package with zero rendering dependencies. `core/scene-mapper` depends only on `scene-graph` types. The `renderer/` packages consume both and map to Babylon.js objects. This means the data pipeline, mappers, and extensions never import Babylon.js.

### 5.3 Key Interfaces

```typescript
// ---- High-Resolution Timestamp ----
// All timestamps in the system use this type.
// Internally stored as microseconds since epoch for integer arithmetic
// (avoids floating-point precision loss at sub-ms scale).
interface Time {
  readonly sec: number;         // seconds since epoch (integer)
  readonly usec: number;        // microseconds within the second (0–999999)
}

// Utility: convert to/from DOMHighResTimeStamp for WebCodecs/performance API
function toHighResTimestamp(t: Time): DOMHighResTimeStamp;
function fromHighResTimestamp(ts: DOMHighResTimeStamp): Time;

// Comparison at microsecond precision
function timeCompare(a: Time, b: Time): -1 | 0 | 1;
function timeDiffUsec(a: Time, b: Time): number;

// ---- DataSource ----
interface DataSource {
  readonly id: string;
  readonly displayName: string;

  initialize(): Promise<DataSourceInfo>;
  getInfo(): DataSourceInfo;

  getMessages(args: {
    topics: string[];
    start: Time;
    end: Time;
  }): AsyncIterable<MessageEvent>;

  getVideoStreams(): VideoStreamDescriptor[];

  getVideoSegment(args: {
    streamId: string;
    timestamp: Time;
  }): Promise<EncodedVideoSegment>;

  close(): Promise<void>;
}

interface MessageEvent {
  topic: string;
  timestamp: Time;
  schemaName: string;
  data: Uint8Array;         // view into SharedArrayBuffer — NOT a copy
  sizeInBytes: number;
}

// ---- Video ----
interface VideoStreamDescriptor {
  readonly streamId: string;
  readonly codec: "h264" | "h265" | "av1" | "vp9";
  readonly width: number;
  readonly height: number;
  readonly source: "embedded" | "standalone";
  readonly fileRef?: string;
  readonly keyframeIndex: KeyframeEntry[];
  readonly intrinsics?: CameraIntrinsics;
  readonly extrinsics?: CameraExtrinsics;
}

interface CameraIntrinsics {
  fx: number; fy: number;
  cx: number; cy: number;
  distortion?: number[];
  model: "pinhole" | "fisheye" | "equidistant";
}

interface CameraExtrinsics {
  position: [number, number, number];
  rotation: [number, number, number, number]; // quaternion w, x, y, z
}

// ---- SchemaCodec ----
interface SchemaCodec {
  readonly schemaFormat: string;
  decode(schema: Uint8Array, data: Uint8Array): unknown;
  getFieldNames(schema: Uint8Array): string[];
}

// ---- Scene Mapper (per-customer integration point) ----
interface SceneMapper {
  readonly id: string;
  readonly displayName: string;
  readonly supportedSchemas: string[];  // schema names this mapper can handle

  // Transform a decoded perception message into scene graph entities
  mapMessage(
    topic: string,
    schemaName: string,
    message: unknown,
    timestamp: Time
  ): SceneGraphUpdate;
}

interface SceneGraphUpdate {
  entities?: SceneEntity[];
  roads?: RoadElement[];
  trajectories?: Trajectory[];
  ego?: Partial<EgoState>;
}

// ---- Panel Descriptor ----
interface PanelDescriptor {
  readonly id: string;
  readonly displayName: string;
  readonly icon: ComponentType;
  readonly category: "video" | "3d" | "data" | "visualization" | "diagnostic";

  settingsSchema: JSONSchema7;
  defaultSettings: Record<string, unknown>;

  subscriptions(settings: Record<string, unknown>): Subscription[];
  videoStreams?(settings: Record<string, unknown>): string[];
  sceneSubscription?(settings: Record<string, unknown>): SceneSubscription;

  render: ComponentType<PanelRenderProps>;
}

interface PanelRenderProps {
  messages: DecodedMessageEvent[];
  videoFrames: Map<string, VideoFrame>;
  sceneSnapshot: SceneSnapshot | undefined;
  currentTime: Time | undefined;
  settings: Record<string, unknown>;
  width: number;
  height: number;
}

// ---- Extension Manifest ----
interface ExtensionManifest {
  readonly id: string;
  readonly version: string;
  readonly displayName: string;
  readonly author: string;
  readonly description: string;
  readonly panels: PanelDescriptor[];
  readonly sceneEntityTypes?: SceneEntityTypeRegistration[];
  readonly sceneMappers?: SceneMapper[];       // custom perception → scene graph mappers
  readonly entrypoint: string;
  readonly trusted: boolean;
}
```

### 5.4 Zero-Copy Message Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SharedArrayBuffer Pool                       │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │ Ring Buf  │ │ Ring Buf  │ │ Ring Buf  │ │ Ring Buf  │  ...       │
│  │ (topic A) │ │ (topic B) │ │ (video 0) │ │ (video 1) │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │
└───────────────────────┬─────────────────────────────────────────────┘
                        │ Uint8Array views (zero-copy)
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌─────────────┐ ┌─────────────┐
│ I/O Worker   │ │ Decode      │ │ UI Thread   │
│              │ │ Worker      │ │             │
│ Reads file   │ │ Decodes     │ │ Reads       │
│ (File API),  │ │ messages,   │ │ decoded     │
│ writes raw   │ │ runs scene  │ │ data,       │
│ bytes into   │ │ mapper,     │ │ diffs scene │
│ shared ring  │ │ builds      │ │ graph,      │
│ buffers.     │ │ scene graph │ │ renders     │
│              │ │ snapshot,   │ │ Babylon.js  │
│              │ │ writes to   │ │ + video.    │
│              │ │ shared buf. │ │             │
└──────────────┘ └─────────────┘ └─────────────┘
                        │
              Atomics.wait / notify
            for backpressure signaling
```

**Key design details:**

- **SharedArrayBuffer ring buffers** per topic/stream. Zero `postMessage` copies.
- **Atomics.wait / Atomics.notify** for backpressure between I/O and decode workers.
- **VideoFrame objects** are `Transferable` — transferred from decode worker to UI thread (WebCodecs requirement). Raw compressed NAL units live in shared memory.
- **Scene graph snapshots** assembled in the decode worker by the active `SceneMapper`, then transferred to the UI thread. The 3D viewport diffs against the previous snapshot.
- **Cross-Origin Isolation headers** (COEP/COOP) required for SharedArrayBuffer.
- **Fallback:** `Transferable` ArrayBuffers via `postMessage` if SharedArrayBuffer unavailable.

### 5.5 Undo/Redo Architecture

**Zustand + zundo** — automatic state snapshots with throttle/debounce.

```typescript
import { create } from "zustand";
import { temporal } from "zundo";

interface LayoutState {
  tree: LayoutNode;
  panelSettings: Record<string, unknown>;
  bookmarks: Bookmark[];
  sceneFilters: SceneFilterState;

  movePanel: (from: string, to: string) => void;
  resizeSplit: (splitId: string, ratio: number) => void;
  updatePanelSetting: (panelId: string, key: string, value: unknown) => void;
  addBookmark: (bookmark: Bookmark) => void;
  setSceneFilter: (filter: Partial<SceneFilterState>) => void;
}

const useLayoutStore = create<LayoutState>()(
  temporal(
    (set) => ({
      tree: initialLayout,
      panelSettings: {},
      bookmarks: [],
      sceneFilters: defaultSceneFilters,
      movePanel: (from, to) =>
        set((s) => ({ tree: applyMove(s.tree, from, to) })),
      resizeSplit: (id, ratio) =>
        set((s) => ({ tree: applyResize(s.tree, id, ratio) })),
      updatePanelSetting: (panelId, key, value) =>
        set((s) => ({
          panelSettings: {
            ...s.panelSettings,
            [panelId]: { ...s.panelSettings[panelId], [key]: value },
          },
        })),
      addBookmark: (bookmark) =>
        set((s) => ({ bookmarks: [...s.bookmarks, bookmark] })),
      setSceneFilter: (filter) =>
        set((s) => ({ sceneFilters: { ...s.sceneFilters, ...filter } })),
    }),
    {
      handleSet: (handleSet) => {
        let timeout: ReturnType<typeof setTimeout>;
        return (state) => {
          clearTimeout(timeout);
          timeout = setTimeout(() => handleSet(state), 300);
        };
      },
      partialize: (state) => ({
        tree: state.tree,
        panelSettings: state.panelSettings,
        bookmarks: state.bookmarks,
        sceneFilters: state.sceneFilters,
      }),
    }
  )
);
```

**Undoable:** layout, panel settings, bookmarks, scene filter/visibility state.
**Not undoable (separate store):** playback time, playback rate, 3D camera orbit position, hover/selection, data loading.

### 5.6 Scene Graph Architecture

The scene graph is the central data structure for AV perception visualization. It is a **pure typed data model** with no rendering dependencies — it lives in `core/scene-graph` and can be produced, consumed, serialized, and tested without importing Babylon.js.

#### 5.6.1 Scene Graph Data Model

```typescript
interface SceneSnapshot {
  timestamp: Time;
  ego: EgoState;
  entities: SceneEntity[];
  roads: RoadElement[];
  trajectories: Trajectory[];
}

interface EgoState {
  position: Vec3;
  orientation: Quaternion;
  velocity: Vec3;
  steering: number;
  dimensions: Vec3;
}

type SceneEntity =
  | BoundingBoxEntity
  | PointCloudEntity
  | CustomEntity;

interface BoundingBoxEntity {
  type: "bounding-box";
  id: string;
  trackId?: string;
  class: string;               // "vehicle", "pedestrian", "cyclist", "unknown"
  confidence: number;          // 0.0–1.0
  position: Vec3;
  orientation: Quaternion;
  dimensions: Vec3;            // length, width, height (meters)
  velocity?: Vec3;
  acceleration?: Vec3;
  color?: Color4;
  metadata?: Record<string, unknown>;
}

interface PointCloudEntity {
  type: "point-cloud";
  id: string;
  positions: Float32Array;     // [x0,y0,z0, x1,y1,z1, ...] in SharedArrayBuffer
  colors?: Uint8Array;         // [r0,g0,b0,a0, ...] per-point RGBA
  intensities?: Float32Array;
  pointSize: number;
}

interface CustomEntity {
  type: string;                // OEM-registered, e.g., "acme:radar-detection"
  id: string;
  data: Record<string, unknown>;
}

type RoadElement =
  | LaneMarking
  | RoadBoundary
  | Crosswalk
  | StopLine
  | TrafficSign;

interface LaneMarking {
  type: "lane-marking";
  id: string;
  points: Vec3[];
  style: "solid" | "dashed" | "double-solid" | "double-dashed";
  color: Color4;
  width: number;
}

interface RoadBoundary {
  type: "road-boundary";
  id: string;
  points: Vec3[];
  boundaryType: "curb" | "guardrail" | "wall" | "virtual";
}

interface Crosswalk {
  type: "crosswalk";
  id: string;
  polygon: Vec3[];
}

interface StopLine {
  type: "stop-line";
  id: string;
  points: [Vec3, Vec3];
}

interface TrafficSign {
  type: "traffic-sign";
  id: string;
  position: Vec3;
  signType: string;
  facing: Quaternion;
}

interface Trajectory {
  id: string;
  source: "planned" | "actual" | "predicted";
  entityId?: string;
  points: Vec3[];
  timestamps?: Time[];
  color: Color4;
  lineWidth: number;
}

type Vec3 = [number, number, number];
type Quaternion = [number, number, number, number];  // w, x, y, z
type Color4 = [number, number, number, number];      // r, g, b, a (0–1)
```

#### 5.6.2 Scene Graph → Babylon.js Rendering Pipeline

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│ scene-graph  │     │ scene-renderer   │     │ Babylon.js Engine    │
│ (pure data)  │────▶│ (diff + map)     │────▶│ (GPU rendering)     │
│              │     │                  │     │                      │
│ SceneSnapshot│     │ Diffs against    │     │ Thin instances for   │
│ per frame    │     │ previous frame.  │     │ bounding boxes.      │
│              │     │ Creates/updates/ │     │ Line meshes for      │
│              │     │ disposes Babylon │     │ polylines.           │
│              │     │ nodes.           │     │ Particle system for  │
│              │     │                  │     │ point clouds.        │
│              │     │ Entity renderers │     │ Video textures for   │
│              │     │ are pluggable.   │     │ camera projection.   │
│              │     │                  │     │                      │
│              │     │                  │     │ WebGPU primary,      │
│              │     │                  │     │ WebGL2 fallback.     │
└──────────────┘     └──────────────────┘     └──────────────────────┘
```

**Diff-based updates:** The `scene-renderer` diffs each new `SceneSnapshot` against the previous:

- **New entity:** create Babylon.js mesh + material
- **Updated entity:** update position/rotation/scale in-place — no allocation
- **Removed entity:** dispose mesh
- **Unchanged:** no-op

For bounding boxes: Babylon.js **thin instances** — one box mesh, N instances, 1 draw call for 200+ boxes.

For point clouds: Babylon.js **particle system** with `SolidParticleSystem` — positions and colors uploaded as flat typed arrays. GPU-side point rendering with configurable point size and optional intensity-based coloring.

#### 5.6.3 Camera-to-3D Composition

**Mode A: 3D overlays on camera view (2D panel)**
The camera view panel projects 3D bounding boxes onto the camera image using intrinsics/extrinsics. This is how a validation engineer checks perception accuracy.

```
┌─────────────────────────────────────┐
│  Camera View Panel                  │
│  ┌───────────────────────────────┐  │
│  │  ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄  │  │
│  │  █  video frame             █  │  │
│  │  █         ┌────┐           █  │  │
│  │  █         │ car│ ← 3D bbox █  │  │
│  │  █         │    │  projected█  │  │
│  │  █         └────┘  to 2D   █  │  │
│  │  █▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄█  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

**Mode B: Video textures in 3D scene (3D panel)**
Camera video projected onto a plane at the camera's world position — perception boxes float over the actual image.

Both modes use `CameraIntrinsics` / `CameraExtrinsics` from `VideoStreamDescriptor`.

#### 5.6.4 Temporal Visualization

**Ghost / trail mode:** Renders entities from the previous N frames (configurable, e.g., last 2 seconds) with decreasing opacity. The scene renderer maintains a ring buffer of recent `SceneSnapshot`s. Ghost entities use alpha-blended materials: `alpha = 1.0 - (age / maxAge)`.

#### 5.6.5 Custom Scene Entities (Extension Point)

```typescript
import { registerEntityRenderer } from "@open-studio/scene-renderer";

registerEntityRenderer("acme:radar-detection", {
  create(entity: CustomEntity, scene: BABYLON.Scene): BABYLON.TransformNode {
    const cone = BABYLON.MeshBuilder.CreateCylinder("radar", {
      diameterTop: 0,
      diameterBottom: entity.data.range * 2,
      height: entity.data.range,
    }, scene);
    return cone;
  },
  update(entity: CustomEntity, node: BABYLON.TransformNode): void { /* ... */ },
  dispose(node: BABYLON.TransformNode): void { node.dispose(); },
});
```

#### 5.6.6 Bird's-Eye-View (BEV) via Multi-Camera Rendering

BEV is not a separate panel type — it is a second instance of the `viewport-3d` panel configured with an orthographic top-down camera. Both the perspective 3D viewport and the BEV panel share the same underlying Babylon.js `Scene` object, rendered through different `Camera` instances.

Babylon.js natively supports this via its multi-camera system:

```typescript
// Perspective camera (main 3D viewport)
const perspCamera = new BABYLON.ArcRotateCamera("persp", ...);
perspCamera.attachControl(perspCanvas);

// Orthographic camera (BEV panel)
const bevCamera = new BABYLON.FreeCamera("bev", new BABYLON.Vector3(0, 100, 0), scene);
bevCamera.mode = BABYLON.Camera.ORTHOGRAPHIC_CAMERA;
bevCamera.rotation = new BABYLON.Vector3(Math.PI / 2, 0, 0);  // look straight down
bevCamera.orthoLeft = -50; bevCamera.orthoRight = 50;
bevCamera.orthoTop = 50; bevCamera.orthoBottom = -50;

// Each panel gets its own view via scene.activeCameras or engine.registerView()
// Babylon's engine.registerView() allows rendering the same scene into
// multiple canvases — one per panel — with independent cameras.
const bevView = engine.registerView(bevCanvas, bevCamera);
```

**Benefits of shared scene:**
- Zero duplication of meshes, materials, or textures between perspective and BEV
- Scene entity filtering, ghosting, and visibility state applies to both views automatically
- User can open 1, 2, or N viewport panels with different camera configurations — each is just a panel instance with a camera mode setting

**Panel settings for viewport-3d:**
- Camera mode: `perspective` (orbit/follow-ego) or `orthographic` (top-down BEV)
- For BEV: configurable area size (e.g., 100m × 100m centered on ego), rotation lock (north-up vs. ego-heading-up)
- Follow-ego: camera tracks ego vehicle position (both modes)

#### 5.6.7 Scene Mapper Architecture (Per-Customer Integration)

Perception stacks are too diverse to expect a canonical scene graph wire format. Every OEM has their own Protobuf schemas, topic naming conventions, and coordinate frames. The `SceneMapper` is the explicit integration point:

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────┐
│ Raw Messages     │     │ SceneMapper      │     │ SceneSnapshot │
│ (OEM Protobuf,   │────▶│ (per-customer)   │────▶│ (typed,       │
│  CDR, JSON, etc.)│     │                  │     │  universal)   │
│                  │     │ Maps topic       │     │              │
│ /perception/     │     │ /perception/objs │     │              │
│   objects        │     │ → BoundingBox[]  │     │              │
│ /planning/path   │     │                  │     │              │
│ /vehicle/state   │     │ /planning/path   │     │              │
│                  │     │ → Trajectory     │     │              │
└──────────────────┘     └──────────────────┘     └──────────────┘
```

**Built-in mappers:**
- `autoware-mapper`: Maps Autoware's `DetectedObjects`, `PredictedObjects`, `PathWithLaneId` messages
- `apollo-mapper`: Maps Apollo's Protobuf perception/planning messages
- `generic-json-mapper`: Configurable via a JSON mapping file (topic → entity type + field mapping)

**OEM custom mappers:** Delivered as part of the OEM's extension package. A mapper is registered via the extension manifest and runs in the decode worker alongside the codec.

```typescript
const acmeMapper: SceneMapper = {
  id: "acme:perception-mapper",
  displayName: "ACME Perception Pipeline v3",
  supportedSchemas: ["acme.perception.v3.ObjectList", "acme.planning.v3.Trajectory"],

  mapMessage(topic, schemaName, message, timestamp) {
    if (schemaName === "acme.perception.v3.ObjectList") {
      return {
        entities: message.objects.map((obj) => ({
          type: "bounding-box" as const,
          id: obj.trackId,
          trackId: obj.trackId,
          class: mapAcmeClass(obj.classification),
          confidence: obj.score,
          position: [obj.x, obj.y, obj.z],
          orientation: eulerToQuat(obj.yaw, 0, 0),
          dimensions: [obj.length, obj.width, obj.height],
          velocity: obj.velocity ? [obj.vx, obj.vy, 0] : undefined,
        })),
      };
    }
    // ... other schema mappings
    return {};
  },
};
```

This is the primary integration deliverable for each OEM customer. It's small (typically 100-300 lines), testable in isolation (pure function, no rendering), and doesn't require forking any core package.

#### 5.6.8 Future: 3D Scene Reconstruction via Gaussian Splatting (v1.x)

For teams with camera + depth data (stereo cameras, or LiDAR-projected depth maps), a future enhancement is real-time 3D scene reconstruction using Gaussian splatting:

- **Input:** video frames + per-pixel depth maps + camera poses
- **Processing:** WebGPU compute shader constructs a set of 3D Gaussians from the depth + color data
- **Rendering:** Gaussians rendered via a custom Babylon.js shader, composited with the existing scene graph entities (bounding boxes appear embedded in a photorealistic reconstruction of the driving environment)

This is computationally intensive and will require WebGPU (no WebGL2 fallback). It is a v1.x feature, but the architecture must not preclude it — which is why the scene graph and renderer are cleanly separated, and why Babylon.js's WebGPU compute shader support is a factor in the engine choice.

### 5.7 High-Resolution Timestamp Architecture

AV sensor data frequently arrives at 100Hz+ (cameras at 30-60fps, radar at 20Hz, LiDAR at 10-20Hz, IMU at 100-400Hz). At 100Hz, events are 10ms apart. At 400Hz (IMU), events are 2.5ms apart. The timeline system must differentiate between events at sub-millisecond precision.

**Design decisions:**

- **`Time` type uses integer microseconds** (sec + usec), not floating-point milliseconds. This avoids IEEE 754 precision loss beyond 2^53 microseconds (~285 years — effectively unlimited).
- **All internal synchronization uses `performance.now()`** (`DOMHighResTimeStamp`, microsecond resolution) rather than `Date.now()` (millisecond resolution). The `performance.timeOrigin` anchors the high-resolution clock to wall time.
- **WebCodecs `EncodedVideoChunk.timestamp` and `VideoFrame.timestamp`** use microseconds natively — our `Time` type maps directly.
- **Timeline scrubber UI** renders at millisecond granularity visually, but the seek position is stored at microsecond precision internally. Zooming into the timeline reveals sub-millisecond event spacing.
- **Message ordering** within the pipeline uses `timeCompare()` for stable sorting at microsecond precision. Messages at the same microsecond are ordered by topic name (deterministic tiebreaker).

---

## 6. Extension Development Architecture

### 6.1 Design Philosophy: Home Assistant, not npm

OEM tooling developers should not need to clone the monorepo. The extension DX is:

1. **Discover** extensions from a registry (self-hosted by OEM)
2. **Install** by adding a manifest URL to the app configuration
3. **Develop** entirely in the browser via the hosted IDE

### 6.2 Hosted Single-File Extension IDE

Each extension is a single TypeScript file exporting a `PanelDescriptor`. This maps directly to the concept that each extension IS a panel component.

```
┌─────────────────────────────────────────────────────────────┐
│  Extension IDE Panel                                         │
│ ┌───────────────────────┐ ┌───────────────────────────────┐ │
│ │ Monaco Editor         │ │ Live Preview                  │ │
│ │                       │ │                               │ │
│ │ // my-panel.tsx       │ │ ┌───────────────────────────┐ │ │
│ │ export const desc:    │ │ │ [Your panel renders here] │ │ │
│ │   PanelDescriptor = { │ │ │  with live data from the  │ │ │
│ │   id: "acme:my-panel",│ │ │  currently loaded session  │ │ │
│ │   ...                 │ │ └───────────────────────────┘ │ │
│ └───────────────────────┘ └───────────────────────────────┘ │
│ [Save to Registry]  [Export .zip]  [Publish]                │
└─────────────────────────────────────────────────────────────┘
```

**Technical implementation:**

- **Editor:** Monaco with TypeScript intellisense against `@open-studio/panel-api` and `@open-studio/scene-graph` types
- **Bundler:** esbuild-wasm (in-browser, sub-second builds, no server-side step)
- **Live preview:** bundled module loaded via `import(blobURL)`, rendered with current drive session data
- **3D extensions:** scene entity renderers are hot-reloaded — change code, see updated geometry immediately

### 6.3 Extension Registry & Caching

Extensions are described by an `ExtensionManifest` and distributed as a single ES module bundle. The registry is a simple HTTP endpoint, self-hostable by OEMs:

```
GET /extensions/ → [{ id, version, displayName, description, author, entrypoint }]
GET /extensions/acme:can-bus-viewer/bundle.js → ES module
GET /extensions/acme:can-bus-viewer/manifest.json → ExtensionManifest
```

**Caching for offline use:** The application's service worker caches extension bundles alongside the main application assets. Once the target environment has been online and loaded the app + extensions, subsequent use works offline — no pre-bundled deployment needed. The service worker uses a cache-first strategy for extension bundles, with background updates when online.

### 6.4 Extension Loading

Trusted extensions (loaded from OEM's internal registry) run in the same JS context with full access to Babylon.js scene, Zustand stores, and theme tokens. Untrusted extensions run in an iframe sandbox with a `postMessage` bridge.

---

## 7. Platform & Deployment

### 7.1 Why No Electron

| Capability | Web alternative | Trade-off |
|---|---|---|
| **Open local files** | `<input type="file">` + drag-and-drop | User must explicitly select files |
| **Directory access** | File System Access API | Chrome/Edge only; progressive enhancement |
| **Native menus** | Web app menu bar component | Themed to match white-label |
| **Desktop icon** | PWA install | Chrome/Edge "Install as app" |
| **Auto-update** | Standard web deployment | Simpler than Electron auto-updater |

**What we gain:** No electron-builder/code signing/platform CI. Deployable to vehicle compute platforms. One build target. ~60% CI/CD reduction.

### 7.2 File Access Abstraction

```typescript
interface FileAccessProvider {
  pickFile(options?: { accept?: string[] }): Promise<FileHandle>;
  pickFiles(options?: { accept?: string[] }): Promise<FileHandle[]>;
  pickDirectory?(): Promise<DirectoryHandle>;
  watchDirectory?(handle: DirectoryHandle): AsyncIterable<FileChangeEvent>;
  readonly supportsDirectoryPick: boolean;
  readonly supportsFileWatching: boolean;
}

interface FileHandle {
  readonly name: string;
  readonly size: number;
  read(offset: number, length: number): Promise<Uint8Array>;
  stream(): ReadableStream<Uint8Array>;
}
```

### 7.3 Deployment Configurations

| Target | Server | Headers | Notes |
|---|---|---|---|
| **OEM internal server** | Nginx / Apache / any static host | COEP + COOP | Standard deployment |
| **Developer workstation** | `npx serve` or Vite preview | Auto-configured | Dev and local use |
| **Vehicle compute platform** | `lighttpd`, `busybox httpd` | COEP + COOP | Chromium kiosk mode |
| **PWA install** | Same + `manifest.json` + service worker | Same | Desktop shortcut + offline |

---

## 8. Monorepo Tooling

### Recommendation: TurboRepo + pnpm

| Concern | TurboRepo + pnpm | Bun workspaces |
|---|---|---|
| **Incremental builds** | Content-hash cache + remote caching | No remote caching |
| **Team DX** | Familiar; well-documented | Bun-specific learning curve |
| **CI time** | Turbo Remote Cache cuts CI dramatically | No equivalent |

```jsonc
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"], "outputs": [] },
    "lint": { "outputs": [] },
    "typecheck": { "dependsOn": ["^build"], "outputs": [] }
  }
}
```

---

## 9. Technical Decisions Log

| Decision | Choice | Rationale |
|---|---|---|
| Runtime target | Web-only (Chrome/Edge primary) | WebCodecs + SharedArrayBuffer + WebGPU; deployable to vehicle compute |
| 3D engine | **Babylon.js** | TypeScript-native; production WebGPU; NullEngine for CI; thin instances; multi-camera rendering for BEV; built-in Inspector |
| Scene graph | Custom typed data model | Pure data, no rendering dep; serializable; diffable; extensible |
| Scene mapping | Per-customer `SceneMapper` interface | Perception formats too diverse for canonical wire format; mapper is the integration deliverable |
| Timestamps | Microsecond precision (`sec` + `usec` integers) | Sub-ms differentiation for 100Hz+ sensors; maps directly to WebCodecs microsecond timestamps |
| State management | Zustand + zundo | Fine-grained subscriptions, undo/redo, works in workers, 4KB |
| Layout engine | Custom | Drag UX control, serialization, undo integration |
| Video decoding | WebCodecs API | Hardware-accelerated H.264/H.265/AV1/VP9; zero-copy VideoFrame |
| Video fallback | Keyframe still extraction | No WASM ffmpeg; graceful degradation |
| Point cloud rendering | Babylon.js `SolidParticleSystem` | GPU-side rendering; typed array upload; configurable point size and color |
| BEV implementation | Same Babylon `Scene`, orthographic camera via `engine.registerView()` | Zero mesh duplication; shared filtering/ghosting; N viewport panels possible |
| Charting | uPlot | 1M+ points at 60fps; tiny; Canvas 2D |
| Bundler | Vite (SWC plugin) | Fast dev; library mode; Rollup production |
| Testing | Vitest + Babylon NullEngine | Unit tests + headless 3D screenshot regression |
| CSS | CSS Modules + design tokens | Scoped, zero runtime, white-label via token override |
| Worker data sharing | SharedArrayBuffer + Atomics | Zero-copy; Transferable fallback |
| Worker RPC (control) | comlink | Type-safe RPC for commands/config |
| Extension model | Single-file `PanelDescriptor` export | 1 extension = 1 panel component; simple mental model |
| Extension bundler | esbuild-wasm (in-browser) | No server-side build; sub-second |
| Extension editor | Monaco | TypeScript intellisense for panel-api + scene-graph types |
| Extension caching | Service worker, cache-first | Offline use after first load; no pre-bundling needed |
| Container demuxing | mp4box.js / custom WebM parser | Mature MP4; WebM for AV1 |
| Codec licensing | Deployer's responsibility | Same model as FFmpeg/VLC |

### 9.1 Why Babylon.js over Three.js

| Concern | Babylon.js | Three.js |
|---|---|---|
| **TypeScript** | Written in TypeScript; structural types | JavaScript + `.d.ts`; type gaps |
| **WebGPU** | Production since 6.0; compute shaders | Experimental `WebGPURenderer` |
| **Instanced rendering** | Thin instances: 1 draw call for N boxes | `InstancedMesh` works but less ergonomic |
| **Headless / CI** | `NullEngine` — no DOM, no GPU | Requires `jsdom` + `headless-gl` — fragile |
| **Multi-camera** | `engine.registerView()` for BEV + perspective sharing one scene | Possible but requires manual render target management |
| **Inspector** | Built-in; toggle in dev mode | No built-in; relies on third-party |
| **React integration** | We manage the lifecycle ourselves via panel descriptor | `@react-three/fiber` couples 3D to React reconciliation (the anti-pattern we're avoiding) |
| **Bundle size** | ~500KB min+gz (tree-shakeable) | ~150KB + add-ons |

---

## 10. Licensing

### Business Source License (BSL 1.1)

- **Source available** — anyone can read, build, and modify
- **Non-production use unrestricted** — evaluation, development, testing, research
- **Production use requires commercial license**
- **Auto-converts to open source** after change date (3 years per release)
- **Precedent:** MariaDB, CockroachDB, Sentry, HashiCorp

**Scope:** BSL for core platform. Apache-2.0 for leaf packages (codecs, containers, `panel-api`, `scene-graph` types).

**Video codec notice:** Deploying organizations are responsible for codec patent compliance. Same model as FFmpeg/VLC/GStreamer/Chromium.

---

## 11. Migration & Compatibility

Clean room reimplementation. No Foxglove code copied. Data compatibility:

- MCAP files open without modification
- ROS 2 bag files work identically
- Standalone video files (MP4, WebM) are first-class citizens
- Video streams embedded in MCAP channels are supported
- Foxglove layout JSON is NOT a compatibility target
- Foxglove extensions will NOT run unmodified; migration guide provided
- Foxglove's `SceneUpdate` message format is NOT a compatibility target

---

## 12. Milestones

| Milestone | Scope | Target |
|---|---|---|
| **M0: Skeleton** | Monorepo, CI, design system tokens, theme engine, high-res `Time` type, empty app shell with file picker | Week 2 |
| **M1: Video first** | MP4 demuxer, WebCodecs decoder, Camera View panel, timeline scrub with keyframe seeking, microsecond-precision sync | Week 6 |
| **M2: 3D viewport** | Babylon.js engine (WebGPU + WebGL2 fallback), scene graph types, bounding box + ego rendering, orbit/follow camera, BEV mode via orthographic camera | Week 10 |
| **M3: Data pipeline** | SharedArrayBuffer pipeline, MCAP reader, Raw Messages panel, Protobuf codec, `SceneMapper` interface + generic JSON mapper | Week 14 |
| **M4: Layout + undo** | Drag-and-drop panel layout, layout save/load, undo/redo, panel settings | Week 18 |
| **M5: Perception viz** | Lane markings, road boundaries, tracked object trails, temporal ghosting, camera-to-3D projection, scene filtering UI, point cloud (particle system) | Week 22 |
| **M6: Core panels** | Multi-Camera Grid, Plot, Vehicle State, Timeline Annotation, Diagnostics | Week 26 |
| **M7: Extension platform** | Panel descriptor API, scene entity extensibility, hosted single-file IDE, registry + service worker caching | Week 30 |
| **M8: White-label** | Brand build pipeline, theme compilation, deployment guide, customer pilot with one OEM | Week 34 |
| **M9: Beta** | Performance profiling (200+ bboxes × 12 cameras at 30fps), NullEngine CI tests, documentation site, Autoware + Apollo mappers, license finalization | Week 38 |

---

## 13. Resolved Decisions

All previously open questions have been resolved:

| # | Question | Decision | Rationale |
|---|---|---|---|
| 1 | **Naming** | "Open Studio" remains the working name | White-label means the upstream name matters less than OEM branding; can rename later |
| 2 | **Extension IDE scope** | Single-file extensions at v1 | 1 extension = 1 panel component; clean mental model; multi-file can be added later |
| 3 | **Timestamp precision** | Microsecond (`sec` + `usec` integers) | Must differentiate 100-400Hz sensor events; maps to WebCodecs microsecond timestamps |
| 4 | **Offline deployment** | Service worker caching; no pre-bundling | App + extensions cached after first online load; works offline thereafter |
| 5 | **Point cloud strategy** | Babylon.js particle system (v1); Gaussian splatting (v1.x roadmap) | Particle system covers LiDAR visualization; splatting requires WebGPU compute and is a future enhancement |
| 6 | **Scene graph wire format** | No canonical format; `SceneMapper` interface per customer | Perception stacks are too diverse; the mapper IS the integration deliverable |
| 7 | **BEV implementation** | Same Babylon `Scene`, orthographic camera via `engine.registerView()` | Zero mesh duplication; shared state; user opens N viewports with different camera settings |

---

## 14. Remaining Open Question

1. **Naming:** "Open Studio" is still a working name. Final branding TBD, but lower priority given white-label nature.

---

*This document is the final draft PRD for Open Studio. It defines a web-only, white-label AV/ADAS drive session replay platform with: video-first architecture (WebCodecs), Babylon.js 3D perception visualization with typed scene graph and per-customer mapping layer, zero-copy SharedArrayBuffer threading, microsecond timestamp precision, undo/redo via Zustand + zundo, and a hosted single-file extension IDE with service worker caching. Ready for implementation breakdown.*
