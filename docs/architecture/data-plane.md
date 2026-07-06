# Data Plane: Workers, Shared Memory, and What "Zero-Copy" Means Here

**Read this when:** working on the message pipeline, worker topology, or anything
performance-critical between file bytes and pixels.

## Design stance

The predecessor's real defect was not "everything on the main thread" (parts of Foxglove ran
in workers) but the **absence of a designed threading architecture**: ad-hoc worker adoption
per subsystem and a structured-clone copy at every boundary. Open Studio designs the topology
once, and is honest about where copies happen. "Zero-copy" is a budget, not a slogan: SAB is
used exactly where bytes stay bytes, and nowhere else.

## Topology

```
┌────────────────────────── SharedArrayBuffer arenas (per epoch) ─────────────────────────┐
│   raw message bytes ── numeric columns (scene SoA, plot series, point clouds) ──────────│
└───────────────┬───────────────────────────┬─────────────────────────────┬───────────────┘
                │ write                     │ write                       │ read (views)
        ┌───────┴──────┐            ┌───────┴───────┐              ┌──────┴──────┐
        │  I/O worker  │  bytes →   │ decode worker │   columns →  │  UI thread  │
        │  file/HTTP/WS│            │ codecs, paths,│              │ Babylon, uPlot,
        │  demux MCAP  │            │ scene mappers,│              │ panel onFrame,
        │              │            │ downsampling  │              │ React chrome │
        └──────────────┘            └───────┬───────┘              └─────────────┘
                                            │ VideoFrame / ImageBitmap (Transferable)
                                            │ small control + snapshot deltas (structured clone)
```

Video decode workers (one per active stream group) sit beside the decode worker; see
`video-pipeline.md`.

## The honesty table

| Hop | Mechanism | Copies |
|---|---|---|
| I/O → decode: raw message/chunk bytes | SAB arena, `Uint8Array` views | **0** — the genuine SAB win |
| Decode → UI: numeric columns (scene SoA, plot series, point clouds) | SAB arena, typed-array views | **0** — the other genuine win, and the one that feeds the GPU |
| Decode → UI: decoded message *objects* (Raw Messages, small state topics) | structured clone | 1 copy — unavoidable for JS object graphs; kept small by sending only what cold panels subscribed to |
| Decode → UI: scene snapshot metadata (ids, classes, custom entities) | structured clone | 1 copy — snapshots are small once hot data is columnar |
| Video: compressed chunks | stay local to their decode worker (demux and `VideoDecoder` are co-located on purpose) | 0 |
| Video: decoded frames | `VideoFrame` transfer (WebCodecs mechanism, not SAB) | 0 |

Anything claiming to add SAB usage must say which row it changes and why the copy it removes
was measured to matter.

## Epoch arenas (not per-topic ring buffers)

The archived PRD specified per-topic SAB ring buffers. Rejected: ring buffers assume steady
forward consumption, while a replay tool's dominant operation is **seek**, which invalidates
everything in flight — plus variable-size messages and cross-thread backpressure make rings a
lot of Atomics choreography for little gain.

Instead: **bump-allocated arena pages, tagged by epoch.**

- The writer bump-allocates from the current page; full pages chain to fresh ones (growable
  SAB or a page pool).
- A **seek increments the global epoch** (`data-sources.md#seek-sequence`). Old-epoch pages
  are reclaimed wholesale once their readers retire — no per-message free, no compaction.
- Every cross-thread notification carries its epoch; consumers drop stale epochs on arrival.
  This one rule eliminates the entire class of "stale frame after seek" bugs.

## Frame handoff: triple buffer, no locks

For "the current frame" state read by the UI each rAF (scene columns, ego/vehicle state), the
decode worker maintains **three buffer slots + an atomic generation counter**: writer fills
the next slot, publishes by storing the generation; the UI reads the latest complete
generation. No locks, no torn reads, no notification needed — the UI *samples* on its own
rAF. This resolves the "notification problem" from the state-management research: the main
thread never waits on the data plane.

Signaling rules:
- Workers may block: `Atomics.wait` for I/O↔decode backpressure is fine.
- The main thread must never block: `Atomics.wait` is illegal there; use rAF sampling
  (preferred) or `Atomics.waitAsync` (Chrome/Edge — our targets) for rare cases.

## Columnar hot path {#columnar}

The strongest zero-copy claim in the system, and the reason the scene graph has a SoA form
(`scene-graph.md#columnar`): the decode worker writes entity columns
(`Float32Array` position/orientation/dimensions, class/track id tables) straight into the SAB
arena; the UI thread builds Babylon **thin-instance matrix buffers from views over the same
memory**. Demux → decode → GPU upload with zero intermediate object graphs and zero per-frame
heap allocation. Plot series and point clouds use the identical pattern (columns → uPlot
arrays / particle system buffers).

Object-graph forms exist for cold/custom entities; they are the slow path and are budgeted
accordingly (`overview.md#performance-budgets`).

## Where rendering runs (v1 decision)

Babylon rendering and scene-column consumption stay on the **UI thread** in v1. OffscreenCanvas
workers were considered and deferred: `engine.registerView` (the BEV design,
`rendering-3d.md`) is built around DOM canvases, and picking/input routing across a worker
boundary is real complexity. The mitigation is that the UI thread's frame job is reduced to
"read columns → update instance buffers → draw" (≤ 4 ms budget); everything else already
happens off-thread. Revisit only with profiler evidence that this budget cannot hold.

## Fallback

Without cross-origin isolation (no SAB): identical topology, arenas become per-epoch
`ArrayBuffer`s posted with **transfer** (ownership move, still no byte copy for the numeric
columns; small increase in message counts). The code paths differ only in allocator choice —
keep it that way.

## Control-plane boundary

Zustand stores never contain sensor-rate data (invariant 1, `overview.md`). Worker RPC for
commands/config (subscribe, seek, budgets) is typed `postMessage` with discriminated unions;
comlink may wrap *control* calls only, never the data path.
