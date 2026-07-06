# Roadmap

**Read this when:** planning a milestone or sequencing work.

Supersedes the milestone table in `archive/PRD-v0.5.md` and the archived M0 plan. The two
structural changes, both traceable to the July 2026 review and the fast-track requirement:

1. **The panel API moves from M7 to M1.** The predecessor built its panels first and its
   extension API years later; the API never fully covered what the hard panels needed. The
   panel API is the OEM-facing product — it is the *first* tested artifact, not the last.
2. **Performance budgets become CI fixtures at M2**, not a beta-phase discovery (M9 in the
   old plan). A performance-differentiated product measures continuously.

## The dogfood gate (standing, from M1)

- Every built-in panel imports only `@open-studio/panel-api` + published leaf packages
  (lint-enforced; see `architecture/overview.md`).
- At least one built-in panel (Camera View) is developed **out-of-tree** as a true extension,
  loaded via the dev-URL mechanism, from M1 forever.
- If a built-in needs an API that doesn't exist, the public API grows or the need is
  rejected. A privileged internal path is never added.

## Milestones

| # | Scope | Gate |
|---|---|---|
| **M0** — Skeleton (wk 2) | Monorepo, CI, dependency-direction lint, **code-quality governance flags**, theme engine + brand config, branded-µs `Time` (`architecture/time.md` — supersedes the archived plan's `{sec,usec}`/bigint mix), file-access abstraction, empty shell | Build/lint/test green; a codec importing from `ui/` fails CI |
| **M1** — Video + panel API v0 (wk 7) | **Narrowed to de-risk:** H.264, standalone MP4 only (embedded-MCAP video lands with MCAP at M3). MP4 demux, WebCodecs decode worker, keyframe index/seek. **`panel-api` v0:** descriptor, `onFrame`/`done`, playback controls, error containment. Camera View panel. `create-panel` CLI + dev-URL loading (minimal) | **Camera View builds out-of-tree as an extension.** If it can't, the API isn't viable — learn it in week 7, not week 30 |
| **M2** — 3D viewport (wk 11) | Babylon engine layer (WebGPU/WebGL2), scene-graph types incl. columnar form, thin-instance boxes + ego, orbit/follow/BEV cameras, frame/calibration math | **Reference-scene perf fixture in CI** (draw calls, allocation, 4 ms budget) |
| **M3** — Data pipeline (wk 15) | Epoch arenas, I/O + decode workers, MCAP reader (incl. embedded video), Protobuf codec (compile-once + schema trees), **backfill + seek sequence**, subscription tiers + block loader + memory budget, `SceneMapper` + generic-JSON + **foxglove-schemas mappers**, Raw Messages panel, multi-file sessions | Seek a 50 GB session: correct scene while paused, memory within budget |
| **M4** — Layout + undo (wk 19) | Drag-and-drop layout, save/load, undo grouping, settings forms from schemas, selection bus, per-panel error containment UX | Tier 0 complete |
| **M5** — Perception viz (wk 23) | Lanes/boundaries, trails + ghosting, camera↔3D projection both modes, scene filtering, point clouds, H.265 + AV1, decoder budget manager, multi-camera sync | 12-stream reference session passes budget fixture |
| **M6** — Core panels + fast-track Tier 1 (wk 27) | **Message paths** (grammar, worker evaluation, autocomplete), Plot (full-range columns + downsampling), Multi-Camera Grid, Vehicle State, Timeline + **Annotations (v1, with export)**, Diagnostics, **declarative panel presets + preset registry artifacts** | Tier 1 complete: a no-code user builds a team dashboard |
| **M7** — Extension platform (wk 31) | Registry + SRI integrity + service-worker caching, manifest/versioning, untrusted iframe sandbox, scene-entity extension registration, `create-panel` CLI hardening, ROS 2 bag + CDR codec | Tier 3 complete end-to-end (scaffold → publish → install) |
| **M8** — White-label + hosted IDE (wk 35) | Brand build pipeline, theme compilation, deployment guide (COOP/COEP configs), **hosted IDE + template gallery (Tier 2)** — deliberately last of the tiers, over a now-stable API | OEM pilot deploy; Tier 2 complete |
| **M9** — Beta (wk 39) | nuScenes demo session + mapper as first-run experience, Autoware mapper, NullEngine screenshot regression, docs site generated from `panel-api` types, license finalization | Beta exit: external team ships a panel at each tier without support escalation |

## Explicitly moved or cut (vs. archived PRD)

- VP9, Apollo mapper → v1.x (`product/requirements.md`)
- Gaussian splatting → research track, no roadmap promise
- Hosted IDE → retained in v1 (fast-track Tier 2) but sequenced after API stabilization — see
  the revision note in `product/extension-fast-track.md`
- Annotations → **into** v1 (M6); they were v1.x in the archived PRD while three other
  sections assumed them
- Schedule: +1 week absorbed into M1 (video pipelines are never early), net M9 at week 39
  vs. 38 — honest rather than optimistic
