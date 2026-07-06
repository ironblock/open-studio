# Decision Log

**Read this when:** two docs disagree (this file wins), or before reopening a settled question.

Entries marked **⟲ revised** changed after the archived PRD v0.5; the "Where" column points to
the governing spec.

| ID | Decision | Choice | Rationale (one line) | Where |
|---|---|---|---|---|
| D-01 | Runtime target | Web-only, Chrome/Edge primary; **no Electron** — coupled to the "no live vehicle connections" non-goal (Electron existed upstream for native sockets) | WebCodecs + SAB + WebGPU; vehicle-compute deploys | `product/vision.md` |
| D-02 | Time representation ⟲ | **Branded integer-µs `number`** (was `{sec,usec}` struct + bigint mix) | Allocation-free, exact to ~2255, natively the WebCodecs domain | `architecture/time.md` |
| D-03 | Worker data sharing ⟲ | SAB **only where bytes stay bytes** (raw transport + numeric columns); transferables for video; structured clone for small objects; **epoch arenas, not per-topic rings**; triple-buffered frame handoff | Seek-dominant workload; honesty table | `architecture/data-plane.md` |
| D-04 | 3D engine | Babylon.js | TS-native, NullEngine CI, thin instances, `registerView`; (Three's WebGPU is production-grade too — not the discriminator) | `architecture/rendering-3d.md` |
| D-05 | Scene graph ⟲ | Pure typed data model, **dual representation: columnar SoA hot path + object cold path**; typed custom entities via declaration merging (no `unknown` payloads) | Zero-alloc decode→GPU; no `as any` in extension code | `architecture/scene-graph.md` |
| D-06 | Scene mapping | Per-customer `SceneMapper` in decode worker; no canonical wire format; built-ins: **foxglove-schemas** ⟲ (new), Autoware, generic-JSON, demo-dataset; Apollo dropped ⟲ | Mapper is the integration deliverable; foxglove-schemas = day-one corpus interop | `architecture/scene-graph.md` |
| D-07 | Coordinate frames ⟲ (new) | Canonical local-ENU frame + static calibration chains + interpolated ego pose; provided to mappers | Every mapper reimplementing quaternion math badly is the alternative | `architecture/scene-graph.md#frames` |
| D-08 | Panel render contract ⟲ | **Imperative `onFrame(frame, done)` with backpressure** (was React props) | Frame-rate data through React was the predecessor's jank mechanism; its own extension API proved the fix | `architecture/panel-api.md` |
| D-09 | Panel/extension model ⟲ | Four authoring tiers (configure / compose / script / build); presets and bundles share one registry | Fast-track requirement: many teams, varying ability | `product/extension-fast-track.md` |
| D-10 | Dogfood gate ⟲ (new) | `panel-api` at M1; built-ins use only the public API; ≥1 built-in out-of-tree | Sequencing was the predecessor's dogfood failure, not API shape | `roadmap.md` |
| D-11 | Message paths ⟲ (new) | Core Apache-2.0 grammar/evaluator/autocomplete; evaluated in decode worker → columns | Powers Plot, Vehicle State, and the whole Tier 1 fast track | `architecture/panel-api.md#message-paths` |
| D-12 | DataSource contract ⟲ | Adds `getBackfillMessages`, batched iterators, subscription tiers, `videoSegmentIterator`; sessions are multi-file | Seek correctness + whole-session views; regression vs. upstream otherwise | `architecture/data-sources.md` |
| D-13 | Codec contract ⟲ | Two-phase `compile()` → `CompiledSchema` with full schema tree; lazy `decodePath` reserved | Schema parse per message is a defect; tree powers autocomplete | `architecture/data-sources.md#codecs` |
| D-14 | Memory management ⟲ (new) | Global budget with per-cache allocations + eviction policies; loaded-ranges UI | Sessions are 10–200 GB; this is where replay tools die | `architecture/data-sources.md#memory-budget` |
| D-15 | Video codecs ⟲ | H.264 + H.265 + AV1; **VP9 dropped to v1.x** | Vehicle codecs only; smaller test matrix | `architecture/video-pipeline.md` |
| D-16 | Decoder budget ⟲ (new) | Priority-managed hardware decode sessions; explicit degradation | 12 streams exceed platform HW sessions; silent software fallback is a CPU cliff | `architecture/video-pipeline.md` |
| D-17 | State management | Zustand + zundo, **control plane only** (invariant); per-gesture undo grouping (not global debounce ⟲) | Store library was never the predecessor's problem; data-plane separation is | `architecture/state-management.md` |
| D-18 | BEV | Same Babylon scene, orthographic camera via `registerView`; 2–4 view ceiling acknowledged | Zero duplication; blit cost known and budgeted | `architecture/rendering-3d.md` |
| D-19 | Charting | uPlot + worker-side min/max downsampling into SAB columns | 1M+ points; columns are the flagship SAB consumer | `research/technology-stack-2026.md` |
| D-20 | Extension security ⟲ (new) | Mandatory SRI in manifests, verify-before-cache, versioned caches | Cache-first SW + registry = persistent compromise vector otherwise | `architecture/extensions.md` |
| D-21 | Hosted IDE ⟲ | Retained in v1 (fast-track Tier 2) but ships M8, after API stabilization; CLI + dev-URL ships M1 | Review said defer; fast-track requirement supersedes with sequencing kept | `product/extension-fast-track.md` |
| D-22 | Annotations ⟲ | v1 (M6), with export | Vision, personas, and a built-in panel already assumed them | `product/requirements.md` |
| D-23 | Error containment ⟲ (new) | Throwing panels/mappers disabled in place; pipeline and session unaffected | Fast-track authors will ship bugs; the platform absorbs them | `architecture/panel-api.md` |
| D-24 | Splatting ⟲ | Research track, no roadmap promise (was v1.x feature) | Real-time RGB-D splat construction is research in 2026 | `product/requirements.md` |
| D-25 | Monorepo/tooling | pnpm + TurboRepo; Vite 8 (Rolldown); Vitest 4; Tailwind v4 tokens; Serwist | Per research docs | `research/technology-stack-2026.md` |
| D-26 | Licensing | BSL 1.1 core; **Apache-2.0 for every extension-reachable package** | Extension authors must never touch BSL code | `product/vision.md` |

Superseded documents: `archive/PRD-v0.5.md`, `archive/M0-implementation-plan-v1.md` —
retained verbatim; do not implement from them.
