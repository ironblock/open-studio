# Scene Graph & Mappers

**Read this when:** working on perception visualization, writing a `SceneMapper`, or adding an
entity type.

The scene graph is the central data structure for AV perception visualization: a **pure typed
data model** in `core/scene-graph` (Apache-2.0) with zero rendering dependencies. It can be
produced, serialized, diffed, and tested without Babylon. Renderers live in `renderer/*`
(`rendering-3d.md`).

## Two representations {#columnar}

High-count entity types have **two forms with one logical schema**:

1. **Columnar (SoA) — the hot path.** Typed-array columns in the data-plane arena
   (`data-plane.md#columnar`): `Float32Array` position/orientation/dimensions/velocity,
   `Float32Array` confidence, integer class/track-id tables with a string intern pool. Written
   by the decode worker, consumed as views by thin-instance rendering, trails/ghosting, and
   plot extraction. Zero per-frame heap allocation.
2. **Object graph — the cold path.** Ergonomic `readonly` objects for custom entities,
   low-count elements (road geometry, trajectories), tests, and Tier 1/2 extension authors.

Bounding boxes and point clouds are columnar in v1; everything else may start as objects and
graduate with profiler evidence.

```typescript
interface SceneSnapshot {
  readonly timestamp: Time;
  readonly ego: EgoState;
  readonly boxes: BoundingBoxColumns;          // SoA, arena-backed
  readonly pointClouds: readonly PointCloudEntity[];  // positions/colors are arena-backed arrays
  readonly entities: readonly SceneEntity[];   // cold/custom object entities
  readonly roads: readonly RoadElement[];
  readonly trajectories: readonly Trajectory[];
}

interface BoundingBoxColumns {
  readonly count: number;
  readonly position: Float32Array;      // xyz interleaved, length = 3*count
  readonly orientation: Float32Array;   // wxyz quaternions, 4*count
  readonly dimensions: Float32Array;    // lwh, 3*count
  readonly velocity: Float32Array;      // 3*count (NaN when absent)
  readonly confidence: Float32Array;    // count
  readonly classId: Uint16Array;        // index into classNames
  readonly trackId: Uint32Array;        // index into trackKeys
  readonly classNames: readonly string[];
  readonly trackKeys: readonly string[];
}
```

Road elements, trajectories, ego state, and the remaining object types match the archived PRD
§5.6.1 shapes (with `readonly` applied throughout); they are not restated here.

## Typing custom entities {#typing}

The archived PRD's `CustomEntity.data: Record<string, unknown>` forced `as any` on the first
line of every custom renderer — the exact smell this project exists to avoid. Custom entity
types are **registered with their payload type**, and the type id ties producer and renderer
together:

```typescript
// In the extension, once:
declare module "@open-studio/scene-graph" {
  interface CustomEntityTypes {
    "acme:radar-detection": { range: number; azimuth: number; snr: number };
  }
}

type CustomEntity<K extends keyof CustomEntityTypes = keyof CustomEntityTypes> = {
  readonly type: K;
  readonly id: string;
  readonly data: CustomEntityTypes[K];   // fully typed at both ends
};
```

`registerEntityRenderer("acme:radar-detection", …)` receives `CustomEntity<"acme:radar-detection">`
— no casts anywhere. The same declaration-merging pattern types mapper inputs (below).

## Coordinate frames & calibration {#frames}

Generic robotics TF is a non-goal; the *math* is not optional. Real logs arrive in sensor
frames with calibration chains, and ego pose needs time interpolation (a 10 Hz pose topic
under a 30 fps camera). Without platform support, every mapper reimplements quaternion math
badly — this is the #1 "why is my bounding box sideways" source.

- **Canonical frame:** local ENU, anchored at session start. All `SceneSnapshot` geometry is
  expressed in it. Ego is a pose within it.
- **Static calibration chains:** sensor→ego transforms loaded from calibration data
  (`foxglove.FrameTransform`, session manifest, or mapper-provided), composed once.
- **Interpolated ego pose:** `FrameProvider.egoPoseAt(t: Time)` with configurable
  interpolation (linear/slerp), fed by the ego-pose topic via backfill + window.
- Mappers receive a `FrameContext` helper: `toCanonical(point, fromFrame, t)`. Mappers that
  ignore it and hand-roll transforms fail review.

## Scene mappers

The per-customer integration point: decoded messages → scene graph. Runs **in the decode
worker**. Pure, testable, no rendering imports.

```typescript
interface SceneMapper {
  readonly id: string;
  readonly displayName: string;
  readonly supportedSchemas: readonly string[];

  mapMessage(input: MapperInput, ctx: MapperContext): SceneGraphUpdate;
}

// Typed via the same declaration-merging registry as custom entities:
// registerSchema<"acme.perception.v3.ObjectList">() ties schemaName → decoded type,
// so mapMessage narrows input.message by input.schemaName — no `unknown` dereferencing,
// no casts in mapper code.
```

`MapperContext` provides `FrameContext`, the column writers (mappers emitting boxes write
columns directly), and an entity-lifetime helper (declare TTL vs. explicit tombstones).

**Error containment (fast-track requirement):** a throwing mapper quarantines the offending
topic — banner on affected panels, session unaffected. Never a crashed pipeline.

### Built-in mappers (v1)

| Mapper | Why |
|---|---|
| `foxglove-schemas` | `SceneUpdate`, `CameraCalibration`, `CompressedImage`, `FrameTransform`, `LocationFix`, … — the existing MCAP corpus lights up on day one; the cheapest adoption lever available |
| `autoware` | `DetectedObjects`, `PredictedObjects`, `PathWithLaneId` |
| `generic-json` | JSON-config-driven field mapping — the no-code Tier 1 mapper (`../product/extension-fast-track.md`) |
| `demo-dataset` | nuScenes-derived sample session mapper; first-run experience + integration/perf fixture |

Apollo: dropped from v1 (momentum faded); v1.x if demand materializes.

## Serialization

No canonical *wire* format (perception stacks are too diverse — the mapper IS the interface),
but `SceneSnapshot` has a defined JSON encoding for golden tests, fixtures, and preset
debugging. The encoding is a test utility, not an interchange promise.
