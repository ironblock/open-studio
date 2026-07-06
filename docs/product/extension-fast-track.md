# Extension Fast Track

**Read this when:** designing anything an extension author touches — the panel API, settings,
the registry, the IDE, docs, or onboarding.

## The requirement

> Many teams should be able to develop their own panels and plugins as easily as possible,
> despite widely varying levels of technical ability.

This is a first-class product requirement (July 2026), not a DX nicety. Inside an OEM, the
people who need a custom view of drive data range from validation engineers who have never
opened a terminal, through scripting-comfortable ML engineers, to professional tooling teams.
A single extension API pitched at one of those groups fails the other two.

The design answer is **tiers**: four ways to create a "panel," each a strict superset of the
one below, each with its own authoring surface, and all distributed through the same registry.
An author should be able to start at Tier 1 and graduate one tier at a time without rewriting
from scratch.

## The tiers

| Tier | Name | Author writes | Audience | Ships |
|---|---|---|---|---|
| 0 | **Configure** | Nothing — arranges layouts, tunes panel settings | Everyone | M4 |
| 1 | **Compose** | JSON config via in-app forms (no code) | Knows their topics, not TS | M6 |
| 2 | **Script** | One TypeScript file in the hosted IDE | Comfortable scripting, no toolchain | Late v1 |
| 3 | **Build** | A real package with its own repo/CI | Professional tooling teams | M1 onward (dogfood) |

### Tier 0 — Configure

Layouts and per-panel settings, saved and shared as JSON (later: URL links). No authoring
concepts at all. This tier already covers "our team wants a triage workspace."

### Tier 1 — Compose (declarative panels) — the fast track's center of gravity

A family of built-in **generic panels** whose entire behavior is driven by configuration built
from **message paths** (`../architecture/panel-api.md#message-paths`):

- **Plot**: N series, each a message path + style
- **Table / watch**: rows of message paths with formatting
- **Gauge / stat / state grid**: message path + thresholds + colors
- **Annotation form**: configurable fields written to bookmarks
- **Generic JSON scene mapper**: topic → entity-type + field-mapping configuration — the
  no-code tier of `SceneMapper`

Configuration is authored entirely in-app via schema-generated forms with message-path
autocomplete against the loaded session — the author never sees JSON unless they want to.
The result is saved as a **panel preset**: a named, versioned JSON artifact, distributable
through the registry exactly like a code extension (it's just data), shareable by file, and
diffable in review.

Design consequences, stated as requirements:

1. Message paths are core, Apache-2.0, and evaluated in the decode worker — Tier 1's power is
   exactly the power of the path grammar.
2. Schema introspection must produce a full typed field tree (autocomplete is the fast track's
   actual UX).
3. Presets are first-class registry artifacts with the same integrity checks as bundles.
4. The expected outcome: **most internal-team needs never leave Tier 1.** Measure this; if
   pilot teams routinely need Tier 2 for basics, Tier 1's panel family is too small.

### Tier 2 — Script (single file, hosted IDE)

One TypeScript file exporting a `PanelDescriptor`. Authored in the hosted browser IDE
(Monaco + esbuild-wasm, `../architecture/extensions.md#hosted-ide`): TypeScript IntelliSense
against `panel-api` and `scene-graph` types, live preview against the currently loaded
session, save-to-registry / export. Also the tier for single-file `SceneMapper`s.

The IDE opens from a **template gallery** (a plot variant, a table, a 2D canvas panel, a
custom scene entity, a mapper) — fast-track authors modify working code; they don't start
from a blank buffer.

### Tier 3 — Build (full package)

`create-panel` CLI scaffolds a repo (Vite library build, tests, typecheck); the app has a
"load extension from dev-server URL" mode for hot-reload development; output is a bundle +
manifest published to the registry. Multi-panel extensions, custom Babylon entity renderers,
and anything with dependencies live here. **Built-in panels are Tier 3 extensions** — the
dogfood gate in `../roadmap.md` keeps the tier honest from M1.

## Cross-cutting requirements the tiers impose

- **Error containment.** Fast-track code *will* throw. A failing panel or mapper is disabled
  in place with the error and a reset affordance; the app, the pipeline, and other panels are
  unaffected. Mapper exceptions quarantine the offending topic, not the session.
- **Graduation path.** "Eject" a Tier 1 preset to a Tier 2 file (generate the equivalent
  `PanelDescriptor` source); open a Tier 2 file in a Tier 3 scaffold. Never a rewrite.
- **One registry.** Presets, single files, and bundles are all `RegistryArtifact`s with
  manifests, versions, and SRI hashes (`../architecture/extensions.md`).
- **Docs generated from types.** `panel-api` doc comments are the IDE's IntelliSense *and*
  the reference site; they are written for the Tier 2 audience (plain language, examples).
- **Stability discipline.** Tier 1 preset schemas and the Tier 2/3 `panel-api` surface are
  versioned; breaking changes require a migration note and (for presets) an automatic
  migrator. Many low-tech-ability teams means many artifacts nobody will hand-update.

## Revision note

The July 2026 architecture review (§4.3.1) recommended deferring the hosted IDE to v1.x as a
scope cut. The fast-track requirement arrived after that review and partially supersedes it:
the IDE is retained in v1 as the Tier 2 surface, but the review's *sequencing* argument
stands — the IDE ships late in v1, after the panel API has stabilized through the M1-onward
dogfood gate, and Tier 1 (which serves the least-technical authors and needs no IDE) ships
first, at M6. What was cut instead: VP9, the Apollo mapper, and the splatting roadmap item.
