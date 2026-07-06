# Vision

**Read this when:** you need product context — who this is for, what it is not, and how it makes money.

Open Studio is a white-label drive-session replay and sensor-data visualization platform,
purpose-built for autonomous driving and ADAS development teams. It is designed to be
OEM-branded and deployed by automakers as their internal tooling for reviewing, annotating,
and debugging vehicle drive sessions.

It runs entirely in the browser — no Electron, no native install. It deploys to a developer's
workstation, an internal web server, or directly on a vehicle's compute platform.

**One-line pitch:** "The drive session workbench your OEM customers ship under their own brand."

## Strategic position

Open Studio is a clean-room reimplementation of the *idea* of Foxglove Studio (whose last
open-source release was late 2023), refocused on AV/ADAS. Two things follow:

1. **The competitor is current, commercial Foxglove — not the 2023 snapshot.** Closed-source
   Foxglove shipped WebCodecs video during 2024 and kept moving. The moat is therefore not a
   feature list; it is the things a closed platform structurally cannot offer an OEM:
   - white-label builds with no upstream branding,
   - in-house panels and mappers written against a stable Apache-2.0 API (no forking, no vendor
     review),
   - deployment on vehicle compute and air-gapped infrastructure,
   - a data plane fast enough to make 12-camera session review pleasant.
2. **Extensibility is the product.** Per-OEM `SceneMapper`s and panels are the integration
   deliverable, and many internal teams — of widely varying technical ability — must be able to
   build their own panels and plugins with minimal friction. See
   [`extension-fast-track.md`](./extension-fast-track.md); this requirement shapes the panel
   API, the settings system, and the roadmap.

## Business model

Source-available. Core platform under Business Source License (BSL 1.1): inspection,
modification, and self-hosted internal use permitted; redistribution or SaaS requires a
commercial license (auto-converts to open source 3 years per release — MariaDB/Sentry
precedent). Revenue: commercial white-label licenses, SLA support and integration services,
training.

**Leaf packages are Apache-2.0** — codecs, container parsers, `panel-api`, `scene-graph`
types, message-path grammar. This is load-bearing, not generosity: extension authors and OEM
integration code must never touch BSL-licensed code, so everything an extension imports must
be Apache-2.0.

Video codec patent compliance is the deploying organization's responsibility (FFmpeg/VLC model).

## Target users

| Persona | Key need |
|---|---|
| **AV validation engineer** | Multi-camera synced replay, 3D perception overlay, timeline scrubbing, annotation |
| **ADAS calibration engineer** | Camera-to-3D projection, synchronized multi-channel replay |
| **Perception ML engineer** | 3D boxes with confidence/class, track trails, run comparison |
| **AV tooling developer (OEM)** | Panel SDK, scene-graph extensions, registry, white-label theming |
| **Internal team power user** (new) | Builds team-specific panels *without* owning a toolchain — the fast-track persona |
| **AV program manager** | Shareable layouts, bookmarks/annotations, export |
| **In-vehicle reviewer** | Lightweight browser UI, no install |

## Explicit non-goals

- Electron or any native desktop wrapper (see `decisions.md` D-01 — this is one decision with
  the next line, not two)
- Live vehicle telemetry. Open Studio visualizes **recorded** sessions only. "Streaming" always
  means streaming access to recorded data (e.g., HTTP range requests or a WebSocket proxy in
  front of fleet storage), never a live robot connection. Dropping live connections is *why*
  Electron (native sockets) is unnecessary.
- Cloud backend, managed storage, user accounts (OEM brings infra and identity)
- A proprietary data format replacing MCAP; a canonical scene-graph wire format
- Generic robotics panels with no automotive relevance (TF-tree viewer, URDF, teleop)
- Physics simulation or vehicle control
- Mobile support
- Bundled codec patent licenses
