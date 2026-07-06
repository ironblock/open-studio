# Extensions: Packaging, Registry, Security, Tooling

**Read this when:** working on the extension host, registry, hosted IDE, or `create-panel`
CLI. The authoring-experience tiers these serve are defined in
`../product/extension-fast-track.md`.

## Extension model

An extension contributes panels, scene mappers, and/or custom scene-entity renderers via a
manifest. All tiers produce **registry artifacts** with the same envelope:

```typescript
interface RegistryArtifact {
  readonly id: string;                  // "acme:can-bus-viewer"
  readonly version: string;             // semver
  readonly kind: "preset" | "extension";   // Tier 1 JSON vs. Tier 2/3 code bundle
  readonly displayName: string;
  readonly author: string;
  readonly description: string;
  readonly integrity: string;           // SRI hash of the payload — required
  readonly apiVersion: string;          // panel-api compat range
}

interface ExtensionManifest extends RegistryArtifact {
  readonly kind: "extension";
  readonly entrypoint: string;          // single ES module bundle
  readonly panels: readonly string[];   // contributed panel ids (for discovery UI)
  readonly sceneMappers?: readonly string[];
  readonly sceneEntityTypes?: readonly string[];
  readonly trusted: boolean;
}
```

Loading: trusted extensions (OEM internal registry) run in the app context with full access
to the panel API, scene registration, and theme tokens. Untrusted extensions run in an iframe
sandbox with a `postMessage` bridge and a reduced API. Note the honest framing (corrected
from the archived PRD): the predecessor's extensions were *also* unsandboxed in-process —
in-context trusted execution is not a differentiator; the differentiators are API surface
(scene renderers, shared primitives, selection bus) and the tiered authoring DX.

## Registry

A plain HTTP endpoint, self-hostable by OEMs (static file server suffices):

```
GET /artifacts/                         → RegistryArtifact[] (index)
GET /artifacts/<id>/<version>/manifest.json
GET /artifacts/<id>/<version>/bundle.js     (kind=extension)
GET /artifacts/<id>/<version>/preset.json   (kind=preset)
```

Presets (Tier 1) and code extensions (Tier 2/3) share discovery, versioning, and integrity —
one mental model, one deployment story.

## Supply-chain integrity (normative)

Registry-served modules + a cache-first service worker would otherwise make a registry
compromise **persistent** on every client. Therefore:

- `integrity` (SRI hash) is mandatory in every manifest; the host verifies **before** module
  evaluation and before service-worker caching. Mismatch = artifact rejected and reported.
- Caches are keyed by (id, version, hash); updates are background-fetched, verified, then
  swapped. No mutable "latest" URL is ever cached.
- Optional (v1.x): registry-level signing (public key pinned in the white-label build config)
  for OEMs whose threat model includes the registry host itself.

## Offline

The app's service worker caches application assets and installed artifacts (after integrity
verification). Once a vehicle-compute or air-gapped deployment has loaded the app and its
extensions online once, everything works offline; updates apply when connectivity returns.

## Tooling

### `create-panel` CLI + dev-URL loading (Tier 3 — ships before the IDE)

- `pnpm create @open-studio/panel` scaffolds: Vite library build, `panel-api` types, tests,
  a working example panel, publish script producing `bundle.js + manifest.json`.
- The app's **developer mode** loads an extension from a dev-server URL
  (`https://localhost:5173/my-panel.js`) with hot reload — full-fidelity development against
  real session data, no registry round-trip. This plus the CLI is the primary pre-IDE DX and
  remains the power-user path afterward.

### Hosted IDE (Tier 2) {#hosted-ide}

In-app Monaco + esbuild-wasm single-file editor with live preview:

- TypeScript IntelliSense against `panel-api` + `scene-graph` (type bundles shipped with the
  app; doc comments are the documentation — write them for this audience).
- Bundling via esbuild-wasm (~11 MB, lazily loaded, service-worker cached); output evaluated
  via `import(blobURL)` into a live preview panel fed by the currently loaded session.
- Opens from a **template gallery** (plot variant, table, canvas panel, custom scene entity,
  mapper) — authors modify working code rather than facing a blank buffer.
- Save-to-registry, export, and "eject to Tier 3 scaffold" (downloads a `create-panel`
  project containing the file).
- Sequencing (see `../roadmap.md`): the IDE ships late in v1, *after* the panel API has
  stabilized through the dogfood gate — the IDE is a thin shell over a stable API, not a
  driver of API churn.

### Versioning

`apiVersion` ranges are checked at load; incompatible artifacts are refused with a clear
message. Tier 1 preset schemas carry automatic migrators (`../product/extension-fast-track.md`
— many artifacts will never be hand-updated).
