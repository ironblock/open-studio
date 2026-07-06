# Architecture Review: Foxglove Studio vs. the Open Studio Thought Experiment

**Date:** July 2026
**Inputs:** PRD v0.5, Technology Stack (2026), State Management deep dive, M0 implementation plan
**Reference point:** the final MPL-2.0 releases of `foxglove/studio` (v1.8x, late 2023 — the lineage mirrored at `mvi-llc/foxglove-studio`)

> Method note: this review is written from detailed knowledge of the Foxglove Studio open-source codebase as it existed at the end of 2023, not from a fresh line-by-line re-read of the mirror. Load-bearing factual claims about Foxglove are flagged where confidence is less than high.

> Status note: this is a point-in-time record. Its findings were folded into the split
> specification set (see [`../README.md`](../README.md)) in July 2026. One recommendation was
> subsequently revised: §4.3.1 (defer the hosted IDE) is partially superseded by the
> extension fast-track requirement — see
> [`../product/extension-fast-track.md`](../product/extension-fast-track.md), revision note.

---

## 1. What Foxglove Studio actually was

Before grading the critique, it's worth reconstructing the real artifact, because the PRD's picture of it is right in outline but wrong in several specifics — and two of the wrong specifics matter for the design.

### 1.1 Data layer

- **`Player` interface (push model).** The core abstraction was a stateful `Player` that you attach a listener to; the player pushes `PlayerState` frames (presence, capabilities, `activeData` containing messages, currentTime, topics, datatypes, topicStats) at its own cadence. Consumers can't pull; they receive whatever the player emits. This push-model statefulness — not the absence of an abstraction — was the real source of coupling and complexity.
- **`IIterableSource` (pull model, underneath).** File-based playback was unified under `IterablePlayer`, which drives any `IIterableSource`: `initialize()`, `messageIterator({topics, start, end})`, and — importantly — `getBackfillMessages()` for seek semantics. MCAP, ROS 1 bag, ROS 2 db3, PX4 ULog, and remote (HTTP range request) MCAP all implemented it. Several ran inside workers via a comlink-wrapped `WorkerIterableSource`.
- **Two subscription tiers.** Subscriptions carried a `preload` flag. Current-frame subscriptions fed playback; `preload: true` subscriptions fed the **block loader**, which progressively loaded the *entire file* for those topics into a memory-capped cache (~1 GB order of magnitude) so the Plot panel could draw full-session series without playing through the file. The timeline showed loaded ranges as a progress bar.
- **Schema decoding.** `parseChannel()` in `mcap-support` had a hardcoded switch over message/schema encodings (ros1, cdr, protobuf, flatbuffer, json, omgidl). ROS decoding used *lazy* message readers — deserialization on field access, backed by DataView — an important performance idea the PRD drops.

### 1.2 Message pipeline and state

- `MessagePipeline` was a React context, but by the final OSS releases its internals had already been **rewritten on a Zustand store**; `useMessagePipeline(selector)` was a context-selector hook with reference-equality bail-out. The lesson from Foxglove is therefore *not* "they should have used Zustand" — they did. It's that **putting frame-rate data through the same reactive graph as UI state defeats any store library**. The store choice was never the bottleneck; the data-plane/control-plane conflation was.
- Layout state lived in `CurrentLayoutContext` with action-style updates; layouts persisted locally and (optionally) to Foxglove's cloud console. Layout was `react-mosaic` plus a custom tab-panel system.

### 1.3 Panels and the extension API

This is the most important correction. Foxglove had **two generations of panel API**:

1. **Legacy internal panels** — React components wrapped in a `Panel()` HOC, consuming `useMessagePipeline`, `useMessageReducer`, `PanelContext`, etc. Plot, Raw Messages, and most older panels lived here.
2. **`PanelExtensionContext` panels** — the extension API. A panel is `initPanel(context)`; it declares `context.watch("currentFrame")` / `watch("allFrames")` / etc., calls `context.subscribe([{topic, preload}])`, and receives data via **`context.onRender = (renderState, done) => { ... }`** — a framework-agnostic render callback with an explicit `done()` for backpressure. React is not part of the contract.

Critically, by late 2023 the flagship built-ins — the **3D panel and the Image panel (which had become a mode of the same three.js renderer)** — were built *on the extension API* via `PanelExtensionAdapter`. Foxglove did eventually dogfood. The failure was **sequencing**: the extension API arrived years after the internal panels, so the most data-hungry panels (Plot especially) never migrated, and the API's gaps were discovered late.

Extension points at end of OSS: `registerPanel`, `registerMessageConverter` (convert arbitrary schemas into well-known `foxglove.*` schemas — this *was* the supported way to feed the 3D scene from custom messages), and `registerTopicAliases`. Extensions shipped as `.foxe` zip files containing a CommonJS bundle, **evaluated in-process in the app context — there was no iframe sandbox**.

### 1.4 Threading reality

"Everything on the main thread" is not accurate:

- Several `IIterableSource` implementations ran in workers (deserialized messages crossed to the main thread via structured clone — a copy per message, which is the real indictment).
- The **Plot panel was rewritten in 2023 to render via OffscreenCanvas in a worker**.
- User Scripts (ex-Node Playground) transpiled and executed user TypeScript in workers.
- Image decoding used async `createImageBitmap`.

The accurate critique is: **no coherent threading architecture** — ad-hoc worker adoption per subsystem, structured-clone copies at every boundary, and message *consumption* (panel data extraction, message-path evaluation) still on the main thread inside React's render cycle.

### 1.5 Things Foxglove got right that the PRD currently drops

These come up again in §4; listed here for the record:

- **Message path syntax** (`/topic.pose.position.x[0]{id==5}`) with autocomplete — the single most-loved UX primitive; it's what made Plot, Raw Messages, and State Transitions composable.
- **Seek backfill** — after a seek, panels receive the *latest message at or before* the seek point per subscribed topic (last transform, last camera info, last ego pose). Without this, every panel is blank until its topics next publish.
- **Preload subscriptions + block cache + loaded-ranges UI** (§1.1).
- **`renderState` + `done()` panel contract** (§1.3) — backpressure-aware, framework-agnostic.
- **Well-known schemas** (`foxglove.SceneUpdate`, `CameraCalibration`, `CompressedImage`, `FrameTransform`, …) — a large corpus of existing MCAP files in the wild speaks these.
- **Transform tree with interpolation** — general robotics TF is out of scope for Open Studio, but the *math* (calibration chains, time-interpolated ego pose) is not optional for AV.
- **Onboarding sample data** — Foxglove shipped a hosted nuScenes MCAP as the first-run experience.
- **User scripts** — in-browser TS message transforms; conceptually adjacent to the `SceneMapper`, and a cheap way to prototype mappers.

Post-OSS footnote: closed-source Foxglove shipped WebCodecs H.264/H.265/AV1 video (`foxglove.CompressedVideo`) during 2024. This validates the PRD's Problem 3 — and also means Open Studio competes against *current* Foxglove, which no longer has that gap. The wedge is white-label + OEM extensibility + zero-copy performance, not video per se.

---

## 2. Grading the PRD's critique of Foxglove (§2.2)

| # | Claim | Verdict |
|---|---|---|
| 1 | Data format coupling; "no abstract data source contract" | **Half right.** `IIterableSource` was exactly that contract, and it was decent (the PRD's `DataSource` is a close cousin — with backfill missing, it's arguably a regression; see §4.1). What was genuinely closed: codec/schema resolution (`parseChannel` hardcoded) and the topic/datatype model being ROS-shaped. Note the PRD keeps the topic/schema/channel model it criticizes — correctly, because that model is good. Sharpen the critique to "closed codec registry + push-model Player," which is what the design actually fixes. |
| 2 | React anti-patterns | **Mostly right, one stale point, one inverted point.** Context nesting, main-thread data extraction, mosaic prop-drilling: fair. Stale: `useMessagePipeline` was already Zustand-backed with selector bail-out — the store wasn't the problem. Inverted: "a Foxglove panel IS a React component" was only true of *legacy* panels; the extension API's `onRender(renderState, done)` was framework-agnostic — and the PRD's own `PanelDescriptor.render: ComponentType<PanelRenderProps>` is *more* React-coupled than Foxglove's extension contract. See §3.1. |
| 3 | No video pipeline | **Correct** for the OSS snapshot (JPEG/PNG stills only). Now table stakes vs. commercial Foxglove. |
| 4 | 3D built for robotics, not AV | **Correct and the strongest differentiator.** One stale cell: the Three.js WebGPU status in §9.1 ("experimental") contradicts the Technology Stack doc ("production WebGPU since r171, Sept 2025"). Babylon is still defensible (TypeScript-native, `NullEngine` for CI, `registerView`) — but update the table so the decision rests on the true pillars. |
| 5 | Extensions sandboxed in iframe, narrow message-passing | **Factually wrong, premise inverted.** OSS Foxglove extensions ran unsandboxed, in-process. The real limitations were narrow API *surface* (panels + message converters + topic aliases; no custom 3D renderers, no shared UI primitives) and distribution/DX. This matters: Open Studio's "trusted extensions run in the same JS context" is *the same trust model Foxglove had*, so it can't be cited as a differentiator — the differentiators are surface (scene-entity renderers, shared primitives) and DX. Rewrite this section. |
| 6 | Unnecessary Electron | **Right for this product, but state why it was there:** native ROS TCP/UDP sockets and OS file access for *live robot connections*. Open Studio can drop Electron only because it also drops live connections (§4.3 non-goal). The two decisions are one decision — the PRD should link them, and resolve the contradiction with the "WebSocket stream" must-have (see §4.4). |
| 7 | Monorepo but not modular | **Right.** `studio-base` was a ~1,000-file grab-bag; nothing was independently consumable. The enforced dependency direction + publishable leaf packages is the correct fix, and the ESLint-enforced boundaries in M0 are the right mechanism. |

---

## 3. The three instincts, evaluated

Your three retained points are the right three. Each is directionally correct; each has a spot where the PRD as written quietly re-creates the failure it names.

### 3.1 "Panel plugins didn't hit minimum viable dogfood"

**The instinct is correct, and the PRD's structure supports it** — built-in panels are separate packages consuming `panel-api`, extensions export the same `PanelDescriptor`. But two things undermine it:

**(a) The schedule repeats Foxglove's sequencing mistake.** Milestones M1–M6 build eight panels; "Extension platform / panel descriptor API" is M7, week 30. That is exactly how Foxglove ended up with two panel generations: the built-ins get written first against whatever internal APIs are convenient, and the public API arrives after the fact, missing what the hard panels need. If `panel-api` is the OEM-facing product (and per the PRD's own business model, it is), it must be the *first* consumer-tested artifact, not the last.

*Concrete fix:* make the panel descriptor API an M1 deliverable and make this the standing acceptance test from M1 onward: **every built-in panel imports only `@open-studio/panel-api` (+ published leaf packages), and at least one built-in panel builds from a separate repository as an actual extension.** If Camera View can't be written as an out-of-tree extension in M1, the API isn't viable and it's better to learn that at week 6 than week 30. "Minimum viable dogfood" is precisely: *no privileged internal API exists at all.*

**(b) The render contract re-creates the React coupling being criticized.** `PanelRenderProps` delivers `messages`, `videoFrames`, and `sceneSnapshot` *as React props*. At playback rates that means a React reconciliation per pipeline tick per visible panel — high-rate data through the reactive graph, i.e., Foxglove's actual jank mechanism, reborn with a smaller store. Ironically, Foxglove's *extension* API already had the right shape: `onRender(renderState, done)` — an imperative callback outside React, with `done()` providing per-panel backpressure (a slow panel skips frames instead of stalling the pipeline).

*Concrete fix:* split the descriptor's contract in two:
- **Control plane (React, Zustand):** settings, toolbar, chrome, layout — `render: ComponentType` is fine here.
- **Data plane (imperative):** `onFrame(frame: PanelFrame, done: () => void)` where `PanelFrame` carries messages/video/scene snapshot. Panels that render canvas/WebGL (camera, 3D, plot — i.e., all the hot ones) never touch React with frame data. A `usePanelFrame()` hook backed by `useSyncExternalStore` can exist as a convenience for cold panels (Raw Messages, Diagnostics) — opt-in, throttled.

**(c) The API is missing what real panels need.** Judged against the eight built-ins the PRD itself lists, `PanelDescriptor` lacks:
- **Playback control** — Timeline + Annotation panel must seek; Plot click-to-seek is table stakes. (`seekPlayback`, `setPlaybackSpeed`, play/pause.)
- **Cross-panel interaction bus** — click a bounding box in 3D → highlight the track in Plot → show fields in Raw Messages. A global selection/hover store (entity id, track id, topic+path, time) is what makes a *workbench* rather than a grid of unrelated widgets. Nothing in the PRD carries this.
- **Message path addressing** — see §4.1; Plot and Vehicle State are unbuildable without it, and "subscriptions from settings" implies settings can express a topic+field path with autocomplete.
- **Custom settings UI escape hatch** — auto-generated JSON Schema forms run out around the third real panel (the 3D panel's per-topic/per-class visibility tree cannot be expressed as a JSON Schema form). Foxglove's settings *tree* API existed because forms weren't enough. Keep the schema default, add an escape hatch.
- **Typed settings** — `settings: Record<string, unknown>` forces every panel to hand-cast. Make the descriptor generic (`PanelDescriptor<TSettings>`) and derive `TSettings` from the schema (or author schemas in something zod-like and emit JSON Schema).

### 3.2 "TypeScript smells"

The critique of Foxglove is fair (pervasive `as`-casts and non-null assertions, lodash idioms, EventEmitter-classes, `any` leakage at boundaries — a codebase that reads like Java/Python authors converging on TS). The PRD addresses *architecture* smells (dependency direction, pure-data packages) but not *code-level* governance — and its own interface sketches contain the seeds of the same smells:

1. **`unknown` that immediately gets dereferenced.** `SceneMapper.mapMessage(…, message: unknown, …)` — and the very next code block does `message.objects.map(...)`, which doesn't typecheck. Same in `CustomEntity.data: Record<string, unknown>` followed by `entity.data.range * 2`. As written, every mapper and custom renderer opens with `as any` on line one — which is exactly the smell being fought. Fix: make mappers generic over decoded message types with per-schema type registration, and make `CustomEntity` generic (`CustomEntity<T>`), with the registration API tying the entity type string to `T` (a `Registry<Map>` keyed by branded type-ids). This is the difference between a codebase that *bans* `any` and one that *doesn't need* it.
2. **The `Time` type is inconsistent across the docs and heavier than needed.** PRD §5.3 has `timeDiffUsec(): number`; M0 has `timeDiffUsec(): bigint` plus a mixed number/bigint API — a smell in its own right (mixed arithmetic domains force conversions everywhere). And `{sec, usec}` allocates one object per message; at 100–400 Hz across dozens of topics plus trail buffers, that's real GC pressure. The PRD's own §5.7 notes 2^53 µs ≈ 285 years: so use it — **`type Time = number & { __brand: "usec" }`, integer microseconds since epoch.** Allocation-free, directly sortable, exact integer arithmetic in doubles until year ~2255, and it *is* the WebCodecs timestamp domain (no conversion functions needed at the video boundary). Keep `{sec, usec}` only as a wire/display conversion.
3. **`SchemaCodec.decode(schema, data)` re-parses the schema per message.** Protobuf descriptor parsing per message would be catastrophic. The contract should be two-phase: `compile(schema): MessageDecoder`, amortizing schema parsing and enabling generated accessors. Consider going one step further and specifying **lazy decoding** (Foxglove's ROS readers decoded on field access) — Raw Messages showing 3 fields of a 40 KB message shouldn't pay for full deserialization. At minimum, don't preclude it: `MessageDecoder.decode(view): T` plus optional `decodePath(view, path): unknown`.
4. **`getFieldNames(schema): string[]`** is too weak to power settings autocomplete or message paths — it needs a recursive field tree with types (`introspect(schema): SchemaTree`).
5. **Codify the governance in M0, where it's nearly free:** `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `isolatedDeclarations` on publishable leaves; ESLint bans on `!` non-null assertion, `as` outside typed guard modules, and `any` (including in tests — Foxglove's tests were where `any` bred); `readonly` on all public array/object types (the PRD's interfaces are mutable today: `points: Vec3[]` etc.); prefer discriminated unions + pure functions over class hierarchies (the scene graph and mapper already model this — say it out loud as a rule).

### 3.3 "Zero-copy SharedArrayBuffer + Web Workers"

**The instinct — get demux/decode off the UI thread with a designed threading architecture rather than Foxglove's ad-hoc one — is the most valuable idea in the project.** But the PRD's §5.4 oversells "zero-copy," and the repo's own State Management doc (§"SharedArrayBuffer-backed state stores remain theoretical") lists the four blockers — serialization, notification, isolation headers, immutable-snapshot reads — that the PRD then doesn't resolve. The two documents disagree, and the honest one is the research doc.

Walk the pipeline and ask what is *actually* zero-copy:

| Hop | PRD claim | Reality |
|---|---|---|
| I/O worker → decode worker (raw bytes) | SAB ring buffers | ✅ **Genuinely zero-copy, genuinely valuable.** This is the win; keep it. |
| Decode worker → UI (decoded messages) | "writes to shared buf" | ❌ Decoded messages are JS object graphs. They cannot live in a SAB; they cross via structured clone (a copy) no matter what. |
| Decode worker → UI (scene snapshot) | "transferred to UI thread" | ❌ `SceneSnapshot` as specified (arrays of objects, `Vec3` tuples) is not `Transferable`; `postMessage` structured-clones it. That's *fine* — snapshots are small — but it isn't zero-copy, and the doc shouldn't claim it is. |
| Decode worker → UI (video) | `VideoFrame` transfer | ✅ Correct — but note this is WebCodecs' transferable mechanism, not SAB. Also: if the demuxer and `VideoDecoder` live in the *same* worker, the "NAL units in shared memory" step buys little; compressed chunks can simply stay local. |
| Numeric bulk data (point clouds, plot series) | `Float32Array` in SAB | ✅ **The other genuine win, and underexploited** — see below. |

*Concrete reframing (this is a change to §5.4, not a retreat from the architecture):*

1. **SAB where bytes stay bytes:** file I/O → demux/decode transport, and **numeric columns** — point cloud positions, plot series, bounding-box transform arrays. Everything else crosses worker boundaries by structured clone (small control/state messages) or transferable (`VideoFrame`, `ImageBitmap`, `ArrayBuffer` ownership moves).
2. **Design the scene graph for a columnar hot path.** `entities: BoundingBoxEntity[]` at 200 boxes × 30 Hz is ~6k object allocations/sec — survivable — but trails/ghosting multiply it by the window length, and thin-instance rendering wants a flat `Float32Array` of matrices anyway. Specify a **structure-of-arrays representation for hot entity types** (id table + typed-array columns for position/orientation/dimensions/class/confidence in SAB), with the object form reserved for cold/custom entities. Then the decode worker writes columns and Babylon uploads thin-instance buffers *from the same memory* — that is the real "zero-copy demux-to-GPU" story, and it's better than what the PRD promises.
3. **Replace per-topic ring buffers with epoch-based arenas.** Ring buffers assume steady forward consumption; a replay tool's dominant operation is **seek**, which invalidates everything in flight. Per-topic rings with variable-size messages, backpressure, and seek invalidation is a lot of Atomics choreography for little gain. Simpler and more robust: slab/arena allocation per load-epoch, bump-allocated by the writer, freed wholesale when the epoch (seek generation) retires. Tag every cross-thread message with the epoch and drop stale ones on arrival. (Also note: `Atomics.wait` is forbidden on the main thread — UI-side signaling must be `Atomics.waitAsync` (fine on Chrome/Edge targets) or a `postMessage` nudge; and growable SABs are now available if the arena needs headroom.)
4. **Triple-buffer the per-frame state.** For "the current frame" (scene columns, vehicle state), the game-engine pattern the State Management doc itself cites (Third Room) is the right one: three buffers + generation counter; the decode worker writes, the UI reads the latest complete generation, no locks, no torn reads. This resolves the doc's "notification problem" — the UI doesn't need notification, it samples on its own rAF.
5. **Keep Zustand strictly on the control plane.** The PRD already implies this; make it a stated invariant: *no sensor-rate data ever enters a store, a context, or React props.* The State Management doc's conclusion ("SAB-backed stores remain theoretical") stays true — you're not building a SAB-backed store, you're building a SAB data plane *beside* the store, which is the pattern that actually works.
6. **One more thread to consider (and it's a real tension):** §5.4 leaves scene-graph diffing *and* Babylon rendering on the UI thread. Babylon supports OffscreenCanvas workers, but `engine.registerView()` (the BEV design, §5.6.6) is built around DOM canvases, and picking/input get harder — so keep rendering on the UI thread for v1, but say so explicitly, and state the mitigation: the UI thread's frame job must shrink to "read columns, update instance buffers, draw," with everything else (decode, mapping, downsampling, snapshot assembly) already off-thread. Budget it: ~4 ms UI frame time at 200 boxes + 12 video tiles, measured in CI (NullEngine won't measure GPU, but it can count draw calls and per-frame allocations).

---

## 4. Add / Change / Remove / Reconsider

Ordered by how much each matters.

### 4.1 Add (gaps that will hurt)

1. **Seek backfill in the `DataSource` contract.** `getMessages({topics, start, end})` alone means a seek lands on a blank screen until each topic next publishes (calibration topics may publish *once per session*). Add `getBackfillMessages({topics, time})` (latest-at-or-before per topic) and define the pipeline's seek sequence around it. Foxglove had this; the PRD's contract is a regression without it.
2. **Preload/full-range subscription tier.** Plot-over-the-whole-session and timeline density/heatmap views cannot be fed by a playback cursor. Add a second subscription tier (full-range, downsampled) backed by a memory-budgeted block cache with a loaded-ranges UI. Which leads to:
3. **A memory budget and eviction story.** The PRD never says what happens when the session is bigger than RAM — and AV sessions are 10–200 GB, an order of magnitude past the "hundreds of MB" the PRD cites for Foxglove jank. Chunk cache with LRU + prefetch heuristics for the HTTP range source, decoded-message cache caps, video decode-ahead budget, point-cloud retention. This deserves a PRD section of its own; it is where replay tools live or die.
4. **Message path syntax as a core package.** `/perception/objects.objects[:]{classification=="pedestrian"}.score` — plot fields, filter Raw Messages, drive Vehicle State, alarm thresholds. This was Foxglove's most-loved primitive and the Plot panel is not really buildable without it. It slots cleanly into the architecture: grammar + evaluator in a leaf package, evaluated *in the decode worker*, feeding downsampled series columns into SAB (the flagship SAB consumer, per §3.3).
5. **A `foxglove-schemas` scene mapper + compat codecs.** There is a large corpus of existing MCAP files using `foxglove.*` well-known schemas (SceneUpdate, CameraCalibration, CompressedImage, FrameTransform, LocationFix). One built-in mapper makes every one of those files light up in Open Studio on day one — the cheapest adoption lever available, and it doesn't compromise the "no canonical wire format" stance (it's just another mapper).
6. **Coordinate frames / calibration, minimally.** The scene graph fixes everything in one implicit frame, but real logs arrive in sensor frames with calibration chains, and ego pose needs time interpolation (a 10 Hz pose topic under a 30 fps camera). You don't need ROS TF; you do need: a declared canonical frame (suggest: local ENU anchored at session start, ego as a transform within it), static calibration chains, and interpolated ego pose. Put it in `scene-graph` or a sibling leaf package; `SceneMapper` gets a frame-resolution helper. Without this, every mapper reimplements quaternion math badly — the #1 source of "why is my bounding box sideways" support tickets.
7. **Cross-panel selection/hover bus** (§3.1c) — small store, huge product difference.
8. **Multi-file session model.** Real drive sessions are a *directory*: MCAP (or segmented MCAPs) + sidecar MP4s + calibration JSON. The PRD hints ("standalone MP4/WebM alongside logs") but never defines how files group into one session (naming convention? manifest? user multi-select?). Define a `Session = DataSource[]` composition and a grouping heuristic; this is the first thing a pilot OEM will hit.
9. **A video decoder budget manager.** "12 streams, sub-millisecond sync" collides with hardware reality: platform hardware decode sessions are limited (commonly single digits for H.265), and falling off the hardware cliff means software decode CPU explosion; a seek across 12 streams triggers 12 × keyframe-to-target decode storms. Add an explicit budget/priority system: visible panels get hardware decoders first; hidden/backgrounded streams degrade to keyframe-only; decode-ahead only while playing. Also restate the sync claim precisely: sub-ms *timestamp alignment* when selecting frames (display sync is bounded by refresh rate).
10. **Demo dataset story.** Foxglove's first-run experience was a hosted nuScenes sample — copy that. A public-dataset converter (nuScenes → MCAP with `foxglove.*` or native schemas) provides onboarding, marketing demos, mapper integration tests, and performance fixtures in one artifact.
11. **Extension supply-chain integrity.** Registry-served ES modules + a cache-first service worker = a *persistent* compromise vector if the registry is ever tampered with. Require content hashes (SRI) in the manifest, verify before caching, and version the cache. Cheap now, embarrassing later.

### 4.2 Change

1. **Panel render contract → imperative frame callback with backpressure** (§3.1b). The single highest-leverage change in the PRD.
2. **Pull `panel-api` to M1 and adopt the dogfood acceptance test** (§3.1a): all built-ins on the public API; ≥1 built-in maintained out-of-tree.
3. **Rewrite §5.4 as "designed data plane," not "zero-copy everywhere"** (§3.3): SAB for byte transport + numeric columns; transferables for video; structured clone for small control messages; epoch arenas instead of per-topic rings; triple-buffered current-frame state; columnar hot path through the scene graph.
4. **`Time` → branded integer-microseconds `number`** (§3.2.2), reconciling the PRD/M0 contradiction and deleting the conversion layer at the WebCodecs boundary.
5. **`SchemaCodec` → two-phase `compile()` with schema introspection** (§3.2.3–4), leaving room for lazy decode.
6. **Fix the Foxglove record in §2.2** — especially Problem 5 (no iframe sandbox existed; the differentiator is API surface and DX, not sandbox looseness) and Problem 2 (Zustand was already in there; the lesson is data-plane separation). A PRD that misstates prior art invites the wrong fixes and will be cited back at you by anyone who worked on the original.
7. **Elevate annotations/bookmarks to v1.** The vision statement ("reviewing, annotating, and debugging"), two personas, and the built-in Timeline + Annotation panel all assume it; §4.2 defers it to v1.x. Resolve the contradiction in favor of v1 — an annotation you can't export is still more useful than none for the validation-engineer workflow.
8. **Reconcile the WebSocket contradiction:** "WebSocket stream" is a must-have while "real-time live vehicle telemetry" is a non-goal. Presumably the intent is *streaming access to recorded sessions from fleet storage* — say exactly that, and drop live-source implications (which would otherwise quietly re-import the hardest parts of Foxglove's Player model: presence, late-joining, unbounded buffering).

### 4.3 Remove / defer

1. **The hosted Monaco + esbuild-wasm IDE (M7) → v1.x.** This is the biggest available scope cut: ~16 MB of assets and two heavyweight integrations for a persona (OEM tooling developer) that owns a real IDE. A `create-panel` CLI + "load extension from dev-server URL" mode in the app delivers ~90 % of the DX for ~10 % of the cost, and the freed capacity funds pulling the panel API forward (§4.2.2) — which serves the *same* persona better. Keep the hosted IDE as the flashy v1.x differentiator; it will also be better then, because the panel API it targets will have stabilized through dogfooding.
2. **VP9 at v1.** Automotive cameras are H.264/H.265 with AV1 emerging; VP9 is a web codec, not a vehicle codec. Cutting it shrinks the WebCodecs test matrix by a quarter.
3. **The Apollo mapper at v1.** Apollo's momentum has faded; Autoware is alive and worth keeping. Spend the slot on the `foxglove-schemas` mapper (§4.1.5) and the public-dataset demo mapper (§4.1.10), which have strictly larger audiences.
4. **Gaussian splatting (§5.6.8) → label as a research spike, not a v1.x feature.** Real-time splat construction from RGB-D without per-scene optimization is a research problem in 2026, not an integration task. The stated architectural accommodation (clean scene-graph/renderer split, WebGPU compute in Babylon) costs nothing to keep; the roadmap promise does.

### 4.4 Reconsider (judgment calls, both sides defensible)

1. **BEV via `engine.registerView()`.** Right call versus scene duplication, but know the ceiling: Babylon renders views on one working canvas and blits per view per frame, all views share the working canvas's resolution, and mixed-DPR panel layouts need care. Acceptable for 2–4 viewports; benchmark before promising "N viewport panels." The fallback (multiple viewports composited on a single canvas overlaying the layout) is uglier but faster.
2. **zundo's 300 ms debounce** (§5.5) coalesces *distinct* user actions into one undo entry (a panel move and a settings tweak within 300 ms merge). Debounce per action-type, or use zundo's grouping, before users notice undo "eating" steps.
3. **React 19 `<Activity>` for hidden tabs** — the Technology Stack doc flags it as directly applicable; the PRD never picks it up. Cheap win for tabbed layouts (keep panel state warm, skip its rendering *and* pause its subscriptions via the layout store).
4. **`getVideoSegment(streamId, timestamp)` granularity** — returning one segment per call implies chatty seek behavior; consider an `AsyncIterable<EncodedVideoSegment>` from a start time, symmetric with `getMessages`, so decode-ahead is a stream, not a poll loop.
5. **Schedule realism.** M1 ("video first," week 6) contains a demuxer, a WebCodecs decode pipeline with reorder/queue management, keyframe-indexed seeking, *and* microsecond sync — with panel-api also landing there per this review. Either narrow M1 (H.264-only, standalone MP4 only, embedded-MCAP video moves to M3 where MCAP lands anyway) or give it two more weeks. Similarly M9's "performance profiling" as a final milestone is backwards for a performance-differentiated product: the 200-boxes × 12-cameras benchmark should be a CI fixture from M2 onward, not a beta-gate discovery.

---

## 5. Summary judgment

The thought experiment holds up. The three instincts that stuck with you are the right three, and they are the right *ranking* too — dogfooded panel API first (it's the product), threading architecture second (it's the differentiator), TypeScript hygiene third (it's the compounding interest). The AV-first reframing (video pipeline, typed scene graph, `SceneMapper` as the per-OEM integration deliverable, white-label) is genuinely better positioning than "open Foxglove clone," and the enforced package boundaries fix the real modularity failure.

The three corrections that matter most, in one paragraph each:

1. **Dogfood is a schedule property, not an API property.** The PRD has the right API shape and the same fatal sequencing as Foxglove (panels M1–M6, extension API M7). Invert it: panel-api at M1, every built-in on the public surface, one built-in out-of-tree, forever.
2. **The render contract is the one place the PRD re-creates Foxglove's core mistake** — frame-rate data as React props. Steal the best thing Foxglove's extension API did (`onRender(renderState, done)`) and keep React for chrome and settings only.
3. **"Zero-copy" is currently a slogan where it needs to be a budget.** SAB genuinely pays in two places — raw byte transport and numeric columns — and the columnar scene-graph hot path (decode worker → typed-array columns → thin-instance buffers) is a *stronger* claim than the one §5.4 makes now. The rest of the pipeline is structured clone and transferables, and the PRD is more credible saying so, with an epoch/arena + triple-buffer design replacing the ring-buffer choreography.

And one strategic note: the critique targets the 2023 open-source snapshot, but the commercial product that grew from it shipped WebCodecs video in 2024 and kept moving. The moat is not the feature list — it's the things a closed platform structurally can't offer an OEM: white-label builds, in-house mappers and panels against a stable Apache-2.0 API, deployment on vehicle compute, and a data plane fast enough to make 12-camera review pleasant. Every recommendation above bends toward making those four things true earlier.
