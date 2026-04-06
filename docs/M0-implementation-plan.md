# M0: Skeleton — Implementation Plan

**Target:** Week 2
**Goal:** A running app shell with monorepo infrastructure, CI, theming, and file picker. No data loading yet — just the structural foundation every subsequent milestone builds on.

---

## Phase 1: Monorepo Bootstrap (Day 1–2)

### 1.1 Initialize repo and tooling

```bash
mkdir open-studio && cd open-studio
pnpm init
pnpm add -Dw turbo typescript @types/node vitest prettier eslint
```

- [ ] Create `pnpm-workspace.yaml` with package globs
- [ ] Create `turbo.json` with build/dev/test/lint/typecheck tasks
- [ ] Create root `tsconfig.base.json` (strict, ES2022 target, paths aliases)
- [ ] Create `.prettierrc` and `.eslintrc` (flat config)
- [ ] Create `.gitignore`, `.nvmrc` (Node 22 LTS)
- [ ] Verify `pnpm turbo build` runs (empty, no errors)

### 1.2 Create package scaffolds (empty, buildable)

Each package gets: `package.json`, `tsconfig.json` (extends base), `src/index.ts`, `vitest.config.ts`

**Core packages:**
- [ ] `packages/core/data-source/` — exports `DataSource`, `MessageEvent`, `Time` interfaces
- [ ] `packages/core/scene-graph/` — exports scene graph type definitions
- [ ] `packages/core/scene-mapper/` — exports `SceneMapper` interface
- [ ] `packages/core/panel-api/` — exports `PanelDescriptor`, `PanelRenderProps`
- [ ] `packages/core/layout-engine/` — empty scaffold
- [ ] `packages/core/app-state/` — empty scaffold (Zustand stores later)
- [ ] `packages/core/timeline/` — empty scaffold
- [ ] `packages/core/theme/` — theme token types + CSS injection
- [ ] `packages/core/file-access/` — `FileAccessProvider` interface + implementations
- [ ] `packages/core/message-pipeline/` — empty scaffold
- [ ] `packages/core/video-pipeline/` — empty scaffold

**Leaf packages (empty scaffolds):**
- [ ] `packages/codecs/protobuf/`
- [ ] `packages/codecs/json-schema/`
- [ ] `packages/codecs/ros2-cdr/`
- [ ] `packages/containers/mcap/`
- [ ] `packages/containers/rosbag2/`
- [ ] `packages/video/demuxer/`
- [ ] `packages/video/webcodecs-decoder/`
- [ ] `packages/video/still-fallback/`
- [ ] `packages/video/frame-index/`

**Renderer packages (empty scaffolds):**
- [ ] `packages/renderer/babylon-core/`
- [ ] `packages/renderer/scene-renderer/`
- [ ] `packages/renderer/entity-renderers/`
- [ ] `packages/renderer/camera-projection/`

**Panel packages (empty scaffolds):**
- [ ] `packages/panels/camera-view/`
- [ ] `packages/panels/multi-camera-grid/`
- [ ] `packages/panels/viewport-3d/`
- [ ] `packages/panels/plot/`
- [ ] `packages/panels/raw-messages/`
- [ ] `packages/panels/vehicle-state/`
- [ ] `packages/panels/timeline-annotation/`
- [ ] `packages/panels/diagnostics/`

**UI packages:**
- [ ] `packages/ui/design-system/`
- [ ] `packages/ui/layout-ui/`
- [ ] `packages/ui/panel-chrome/`

**Extension host (empty scaffolds):**
- [ ] `packages/extension-host/registry/`
- [ ] `packages/extension-host/sandbox/`
- [ ] `packages/extension-host/dev-environment/`

**Tools:**
- [ ] `tools/create-panel/`
- [ ] `tools/build-theme/`
- [ ] `tools/eslint-config/` — custom ESLint plugin enforcing dependency direction

**App:**
- [ ] `apps/web/` — Vite 8 + React 19 entrypoint

### 1.3 Dependency direction enforcement

- [ ] Create `tools/eslint-config/rules/no-restricted-imports.ts`
  - Codecs/containers/video cannot import from `core/`, `ui/`, `panels/`, `renderer/`
  - `core/scene-graph` cannot import from `renderer/` or `@babylonjs/*`
  - `core/scene-mapper` can only import from `core/scene-graph`
  - `renderer/` can import from `core/scene-graph` and `@babylonjs/*`
  - `panels/` can import from `renderer/`, `core/`, `ui/`
  - `apps/` can import from anything
- [ ] Verify with `pnpm turbo lint` — violation = CI failure

---

## Phase 2: Core Type Foundations (Day 2–3)

### 2.1 `Time` type and utilities (`packages/core/data-source/`)

```typescript
// src/time.ts
export interface Time {
  readonly sec: number;   // seconds since epoch (integer)
  readonly usec: number;  // microseconds within second (0–999999)
}

export function timeCreate(sec: number, usec: number): Time;
export function timeFromMicroseconds(totalUsec: bigint): Time;
export function timeToMicroseconds(t: Time): bigint;
export function timeFromHighRes(ts: DOMHighResTimeStamp): Time;
export function timeToHighRes(t: Time): DOMHighResTimeStamp;
export function timeCompare(a: Time, b: Time): -1 | 0 | 1;
export function timeDiffUsec(a: Time, b: Time): bigint;
export function timeAdd(t: Time, usec: bigint): Time;
export function timeLerp(a: Time, b: Time, fraction: number): Time;
```

- [ ] Implement all functions with tests
- [ ] Verify microsecond precision with edge cases (epoch boundaries, negative diffs)

### 2.2 Core interfaces (`packages/core/data-source/`)

- [ ] `DataSource` interface
- [ ] `MessageEvent` interface
- [ ] `DataSourceInfo` type (topics, schemas, time range)
- [ ] `VideoStreamDescriptor`, `CameraIntrinsics`, `CameraExtrinsics`
- [ ] `SchemaCodec` interface

### 2.3 Scene graph types (`packages/core/scene-graph/`)

- [ ] Full type definitions from PRD §5.6.1 (SceneSnapshot, EgoState, BoundingBoxEntity, PointCloudEntity, CustomEntity, all RoadElement types, Trajectory, Vec3, Quaternion, Color4)
- [ ] `SceneGraphUpdate` type
- [ ] No logic — pure type exports

### 2.4 Scene mapper interface (`packages/core/scene-mapper/`)

- [ ] `SceneMapper` interface
- [ ] Generic JSON mapper (configurable via mapping definition)

### 2.5 Panel API types (`packages/core/panel-api/`)

- [ ] `PanelDescriptor` interface
- [ ] `PanelRenderProps` interface
- [ ] `Subscription` type
- [ ] `SceneSubscription` type
- [ ] `ExtensionManifest` interface

### 2.6 File access (`packages/core/file-access/`)

- [ ] `FileAccessProvider` interface
- [ ] `StandardFileAccess` — implementation using `<input type="file">` + `File.slice()` for random access
- [ ] `FSAccessProvider` — implementation using File System Access API with feature detection
- [ ] `createFileAccessProvider()` factory with capability detection

---

## Phase 3: Theme Engine (Day 3–4)

### 3.1 Token definitions (`packages/core/theme/`)

```typescript
// src/tokens.ts
export interface ThemeTokens {
  color: {
    surface: { primary: string; secondary: string; tertiary: string; };
    text: { primary: string; secondary: string; disabled: string; };
    border: { default: string; strong: string; };
    accent: { primary: string; secondary: string; };
    semantic: { error: string; warning: string; success: string; info: string; };
  };
  typography: {
    fontFamily: { sans: string; mono: string; };
    fontSize: Record<'xs' | 'sm' | 'md' | 'lg' | 'xl' | '2xl', string>;
    fontWeight: Record<'regular' | 'medium' | 'semibold' | 'bold', string>;
    lineHeight: Record<'tight' | 'normal' | 'relaxed', string>;
  };
  spacing: Record<'0' | '1' | '2' | '3' | '4' | '6' | '8' | '12' | '16', string>;
  radius: Record<'none' | 'sm' | 'md' | 'lg' | 'full', string>;
  shadow: Record<'none' | 'sm' | 'md' | 'lg', string>;
}
```

- [ ] Default light theme
- [ ] Default dark theme
- [ ] `applyTheme(tokens: ThemeTokens)` — injects CSS custom properties at `:root`
- [ ] `loadThemeFromJSON(url: string)` — fetch + validate + apply

### 3.2 Brand configuration (`packages/core/theme/`)

```typescript
export interface BrandConfig {
  productName: string;
  logo: { light: string; dark: string; };
  favicon: string;
  windowTitle: string;
  theme: ThemeTokens;
}
```

- [ ] `applyBrand(config: BrandConfig)` — sets document title, favicon, injects logo

### 3.3 Tailwind CSS v4 integration (`apps/web/`)

- [ ] Install `tailwindcss` v4 + `@tailwindcss/vite`
- [ ] Configure `@theme` to reference CSS custom properties from the theme engine
- [ ] Verify that changing theme tokens at runtime updates all Tailwind-styled elements

---

## Phase 4: App Shell (Day 4–5)

### 4.1 Vite 8 + React 19 setup (`apps/web/`)

- [ ] `pnpm create vite` with React + TypeScript + SWC
- [ ] Configure Vite dev server with COEP/COOP headers (required for SharedArrayBuffer)
- [ ] Configure path aliases matching monorepo packages
- [ ] Verify hot reload works across package boundaries

### 4.2 Empty shell UI

- [ ] App chrome: top bar (logo, product name, menu), main content area, status bar
- [ ] File open button → triggers `FileAccessProvider.pickFile()`
- [ ] Drag-and-drop zone (entire window) → triggers file open
- [ ] Empty state: "Open a drive session to begin" with file format hints
- [ ] Dark/light theme toggle (demonstrates theme engine)
- [ ] All branding from `BrandConfig` (no hardcoded "Open Studio" in the UI)

### 4.3 Zustand store scaffold (`packages/core/app-state/`)

- [ ] `useSessionStore` — currently just `{ file: FileHandle | null }`
- [ ] `useLayoutStore` — empty layout tree, panel settings, bookmarks, scene filters (with zundo temporal middleware wired up but no panels to undo yet)
- [ ] `useThemeStore` — current theme + brand config
- [ ] `devtools` middleware enabled in dev mode
- [ ] Verify Redux DevTools shows store state

---

## Phase 5: CI (Day 5)

### 5.1 GitHub Actions

- [ ] `.github/workflows/ci.yml`:
  - Trigger on push to `main` + PRs
  - `pnpm install --frozen-lockfile`
  - `pnpm turbo lint`
  - `pnpm turbo typecheck`
  - `pnpm turbo test`
  - `pnpm turbo build`
  - `--filter` for affected packages only (Turbo's built-in)
- [ ] Turbo Remote Cache (Vercel or self-hosted) — optional, can add later

### 5.2 Pre-commit hooks

- [ ] `husky` + `lint-staged` for formatting (Prettier) and linting on commit

---

## Deliverables Checklist

At the end of M0, the following must be true:

- [ ] `pnpm turbo build` succeeds for all packages (most are empty scaffolds, but they compile)
- [ ] `pnpm turbo test` runs Time utility tests and passes
- [ ] `pnpm turbo lint` enforces dependency direction (a codec importing from `ui/` fails)
- [ ] `pnpm turbo dev` starts the web app with hot reload
- [ ] The app shell renders with branding from `BrandConfig`
- [ ] The file picker opens and returns a `FileHandle` (but doesn't process the file yet)
- [ ] Theme toggle switches between light and dark
- [ ] Redux DevTools shows Zustand store state
- [ ] CI passes on GitHub Actions
- [ ] All 40+ packages exist as scaffolds with correct dependency declarations
- [ ] No package has a dependency cycle (enforced by ESLint rule)

---

## What M0 Does NOT Include

- No data loading or parsing (M2)
- No video decoding (M1)
- No 3D rendering (M2)
- No panel layout system (M4)
- No extension system (M7)
- No Babylon.js dependency yet (M2)
- No Monaco editor (M7)

---

## Tech Stack Confirmed for M0

| Concern | Choice |
|---|---|
| Package manager | pnpm 10 |
| Monorepo orchestrator | TurboRepo 2.9 |
| Bundler | Vite 8 (Rolldown) |
| Framework | React 19.2 |
| State management | Zustand 5 + zundo |
| Styling | Tailwind CSS v4 + CSS custom properties |
| Testing | Vitest 4 |
| CI | GitHub Actions |
| Linting | ESLint (flat config) + Prettier |

---

*M0 establishes the invariants that every subsequent milestone depends on: dependency direction, theme token contract, Time type, file access abstraction, and CI. Getting this right means M1–M9 can move fast without structural rework.*
