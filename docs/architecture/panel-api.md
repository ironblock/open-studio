# Panel API

**Read this when:** building any panel (built-in or extension), or changing anything a panel
can see. This package (`@open-studio/panel-api`) is Apache-2.0 and is the product's most
important public surface — see `../product/extension-fast-track.md`.

## The core split: control plane vs. data plane

The single most important lesson from the predecessor: **frame-rate data must not flow
through React.** A Foxglove-style "panel = React component receiving messages" means a
reconciliation per pipeline tick per panel — that was the jank mechanism, and its extension
API had already found the fix (`onRender(renderState, done)`); we adopt that shape.

- **Control plane (React):** panel chrome, settings UI, toolbars — re-renders on *user*
  actions only.
- **Data plane (imperative):** a frame callback with explicit backpressure — no React in the
  loop.

```typescript
interface PanelDescriptor<TSettings = EmptySettings> {
  readonly id: string;                       // "acme:can-inspector"
  readonly displayName: string;
  readonly icon: IconRef;
  readonly category: "video" | "3d" | "data" | "visualization" | "diagnostic";

  readonly settings: SettingsDefinition<TSettings>;

  /** Declarative data needs, derived from settings. Re-evaluated on settings change. */
  subscriptions(settings: TSettings): readonly Subscription[];
  videoStreams?(settings: TSettings): readonly string[];
  sceneSubscription?(settings: TSettings): SceneSubscription;

  /** Data plane. Called once per panel instance. */
  createRenderer(ctx: PanelContext<TSettings>): PanelRenderer;

  /** Control plane. Optional React chrome (toolbars, overlays). Never receives frame data. */
  chrome?: ComponentType<PanelChromeProps<TSettings>>;
}

interface PanelRenderer {
  /**
   * The hot path. `frame` is valid only until `done()` is called; views into it must not
   * be retained. Calling done() signals readiness for the next frame — a slow panel skips
   * frames; it never stalls the pipeline or other panels.
   */
  onFrame(frame: PanelFrame, done: () => void): void;

  onSettingsChange?(settings: TSettings): void;
  onResize?(width: number, height: number, dpr: number): void;
  onSelectionChange?(selection: Selection | undefined): void;
  dispose(): void;
}

interface PanelFrame {
  readonly epoch: number;                    // see data-plane.md#epochs
  readonly currentTime: Time;
  readonly didSeek: boolean;                 // true on the first frame after a seek (post-backfill)
  readonly messages: readonly DecodedMessageEvent[];      // current-frame tier
  readonly fullRange?: FullRangeData;                     // full-range tier (columns; plots)
  readonly videoFrames: ReadonlyMap<string, VideoFrame>;
  readonly scene?: SceneSnapshot;
}

interface PanelContext<TSettings> {
  readonly canvas: HTMLCanvasElement | null;  // owned drawing surface (camera/3D/plot panels)
  readonly playback: PlaybackControls;        // seek(t), play(), pause(), setSpeed(r)
  readonly selection: SelectionBus;
  readonly paths: MessagePathApi;             // parse/evaluate/autocomplete
  readonly projection?: CameraProjectionApi;  // 3D↔2D, from renderer/camera-projection
  readonly settings: () => TSettings;
  updateSettings(patch: Partial<TSettings>): void;  // flows through undo/redo
}
```

Convenience for **cold** panels (Raw Messages, Diagnostics): `usePanelFrame(ctx, throttleMs)`
— a `useSyncExternalStore`-backed hook sampling the latest frame at a bounded rate. Opt-in,
throttled, and documented as the slow path. Hot panels (camera, 3D, plot) draw in `onFrame`.

## Playback control

Panels can drive playback (`ctx.playback`). This is not optional surface: Timeline +
Annotation must seek; Plot click-to-seek is table stakes. Playback state itself (cursor,
speed, playing) lives in the timeline store — panels command it, the pipeline observes it.

## Selection bus

The difference between a workbench and a grid of unrelated widgets: click a bounding box in
3D → the Plot highlights that track → Raw Messages shows the source message → the timeline
marks the instant.

```typescript
type Selection =
  | { kind: "entity"; entityId: string; trackKey?: string; time: Time }
  | { kind: "track";  trackKey: string }
  | { kind: "path";   topic: string; path: MessagePath; time?: Time }
  | { kind: "time";   time: Time };

interface SelectionBus {
  get(): Selection | undefined;
  set(sel: Selection | undefined, origin: PanelInstanceId): void;
  hover: /* same shape, higher frequency, separate channel */;
}
```

Backed by a small Zustand store (control plane — selection changes at human rate). Every
built-in panel both publishes and reacts to it; extensions get it via `ctx.selection`.

## Message paths {#message-paths}

`core/message-path` (Apache-2.0 leaf): grammar, evaluator, and autocomplete for addressing
values inside messages — the primitive that makes Plot, Vehicle State, Raw Messages filtering,
and the entire Tier 1 fast track composable:

```
/perception/objects.objects[:]{classification=="pedestrian"}.score
/vehicle/status.speed_mps
/planning/trajectory.points[0].pose.position.x
```

- Grammar: topic, nested fields, indexing/slicing, predicate filters, well-known conversions.
- **Evaluation runs in the decode worker** against decoded messages (or `decodePath` lazy
  decode when available), emitting numeric columns for full-range subscriptions —
  the flagship consumer of the SAB column path (`data-plane.md#columnar`).
- Autocomplete is powered by `CompiledSchema.tree` (`data-sources.md#codecs`) and is a
  first-class UI component reused by every settings form that accepts a path.

## Settings

- `SettingsDefinition<TSettings>` = a schema (JSON-Schema-compatible, authored via a typed
  builder so `TSettings` is *derived*, never hand-cast) + optional custom UI escape hatch:
  `settingsComponent?: ComponentType<SettingsProps<TSettings>>`.
- Auto-generated forms cover Tier 1/2 authors; the escape hatch exists because real panels
  outgrow forms (the 3D panel's per-topic visibility tree cannot be a JSON-Schema form —
  the predecessor learned this and grew a settings-tree subsystem late; we keep the hatch
  from day one).
- Settings changes flow through the undo system (`state-management.md`) and re-evaluate
  `subscriptions()`.

## Error containment (normative)

A throwing `PanelRenderer` (or chrome component) disables that panel instance in place —
error + reset affordance — and must never affect the pipeline, other panels, or the layout.
Fast-track authors will ship bugs; the platform absorbs them.

## Dogfood rule

Every built-in panel imports only this package plus published leaf packages, and at least one
built-in is maintained out-of-tree as a true extension. If a built-in needs something
`panel-api` doesn't expose, the API grows (or the need is rejected) — a privileged internal
path is never added. Enforced by lint (`overview.md`) and by the M1 gate (`../roadmap.md`).
