# Technology stack for an AV/ADAS replay platform in 2026

**React 19.2, Vite 8, Zustand, and pnpm form the strongest foundation for this platform.** The 2025–2026 period brought transformative changes — Vite 8 shipped Rolldown (10–30× faster builds), React Compiler v1.0 eliminated memoization overhead, and Vitest 4 became Angular's default test runner. This report evaluates all 13 technology areas with specific version numbers, metrics, and recommendations tailored to the platform's requirements: Babylon.js 3D, WebCodecs video, SharedArrayBuffer threading, panel-based layout, undo/redo, hosted extension IDE, and white-label theming on Chrome/Edge.

---

## 1. Zustand + zundo wins state management decisively

For a platform needing undo/redo, Web Worker compatibility, and fine-grained subscriptions at minimal bundle cost, **Zustand 5.0.12 + zundo 2.3.0 totals under 2 KB gzipped** — the smallest option by far.

| Library | Version | Stars | npm/week | Size (gzip) | Undo/Redo | Worker Ready |
|---------|---------|-------|----------|-------------|-----------|-------------|
| **Zustand** | 5.0.12 | ~57K | ~21M | ~1 KB | zundo (<700 B) | ✅ vanilla store |
| **Redux Toolkit** | 2.11.2 | ~61K/11K | ~5.5M | ~11–13 KB | redux-undo (2.5 KB) | ✅ |
| **Jotai** | 2.19.0 | ~21K | ~3M | ~3.3 KB | Manual only | ⚠️ needs bridge |
| **Valtio** | 2.3.1 | ~10K | ~1.1M | ~3–4 KB | valtio-history | ⚠️ no proxy transfer |
| **Legend State** | 3.0.0-beta | ~4K | ~26K | ~4 KB | Manual only | ⚠️ vanilla core |
| **Preact Signals** | core 1.14.0 | ~3.9K | ~172K | ~1.3 KB | Manual only | ✅ core |

**Zustand's `zustand/vanilla` store** runs in any JS environment including Web Workers and SharedWorkers — critical for this platform's zero-copy threading architecture. The zundo middleware tracks `pastStates[]` and `futureStates[]` with `partialize` (track only specific fields), configurable `limit` (history depth), and `handleSet` (throttle/debounce). It powers undo/redo at Stability AI, Yext, and NutSH.ai. For very large state snapshots common in ADAS replay (>10 KB per snapshot with sensor data), consider JSON Patch–based approaches instead of full snapshot history.

**Redux Toolkit 2.11.2** remains solid but carries ~11–13 KB overhead (Immer, Reselect, Redux Thunk). The redux-undo package (1.1.0) is stable but in maintenance mode and looking for maintainers. **Jotai 2.19.0** offers atom-level fine-grained subscriptions ideal for concurrent React, but its React-context-based architecture makes Worker interop awkward, and it lacks a mature undo middleware. **Valtio 2.3.1** provides proxy-based auto-tracking (no manual selectors), but Proxies cannot be transferred across worker boundaries via structured cloning. **Legend State** is innovating with its v3 beta's local-first sync engine, but the beta status and ~26K weekly downloads make it risky for enterprise production. The **TC39 Signals proposal** remains at Stage 1 — champions are "especially conservative," with native browser availability estimated at 2028+ at earliest. Preact Signals (core 1.14.0) provides the best current signals experience but requires Babel transforms or explicit hook calls for React.

**Recommendation:** Zustand + zundo. Create separate stores for theme, panel layout, and session data. Use `zustand/vanilla` for any state that needs Worker access.

---

## 2. Vite 8 with Rolldown is the era-defining bundler

**Vite 8 completed the Rolldown integration**, replacing both esbuild (dev) and Rollup (production) with a single Rust-based bundler. Linear reported builds dropping from **46 seconds to 6 seconds**. Rolldown (1.0.0-rc.4) uses Oxc for parsing, transforming, resolving, and minifying — the esbuild option is deprecated and will be removed.

| Bundler | Version | npm/week | Library Mode | Status |
|---------|---------|----------|-------------|--------|
| **Vite** | 8.x | ~65M | ✅ Excellent `build.lib` | Production, Rolldown-powered |
| **Rspack** | 1.7 / 2.0-beta.9 | ~800K | ✅ Via Rslib | Best webpack migration path |
| **Farm** | 1.7.11 | Low | ⚠️ Limited | ⚠️ Last publish 8 months ago |
| **Turbopack** | Next.js 16.2 | N/A | ❌ | Next.js only, not standalone |
| **Rsbuild** | 2.0.0-rc.0 | Growing | ✅ source-build plugin | Rspack with sensible defaults |

Vite 8's additional features include built-in Devtools, tsconfig paths support, Lightning CSS for minification, and an experimental Full Bundle Mode (3× faster dev startup, 40% faster reloads). For transpilation, **Oxc** (v0.121.0) is now Vite's default — **3–5× faster than SWC and 40× faster than Babel** at just 2 MB versus SWC's 37 MB. SWC (v1.15.21) remains the standard for Rspack and Next.js/Turbopack projects.

**Turbopack is production-ready and the default in Next.js 16** but remains tightly coupled to Next.js with no standalone availability. **Farm's development has stalled** (last npm publish July 2025). **Rspack 2.0** (beta.9) is the best choice for teams migrating from webpack, offering 5–10× faster builds with ~85% webpack plugin compatibility.

**Recommendation:** Vite 8 for the monorepo. Use `build.lib` for shared library packages. The Rolldown integration makes it the clear leader for new projects.

---

## 3. pnpm 10 anchors the monorepo with TurboRepo orchestration

**pnpm 10.33.0** is the gold standard package manager for monorepos in 2026 — content-addressable storage, catalogs for centralized version management, and the most ergonomic `--filter` system. Pair it with **TurboRepo 2.9.3** for task caching and orchestration.

| Package Manager | Version | Workspace Quality | Speed |
|-----------------|---------|------------------|-------|
| **pnpm** | 10.33.0 (v11 beta) | ★★★★★ Catalogs, filters | Very fast |
| **Bun** | 1.3.11 | ★★★★ Functional | Fastest raw installs |
| **Yarn Berry** | 4.9.4 | ★★★★ PnP, catalogs | Competitive |
| **npm** | 11.12.1 | ★★★ Basic | Slowest |

| Monorepo Tool | Version | Stars | Remote Cache | Best For |
|---------------|---------|-------|-------------|----------|
| **TurboRepo** | 2.9.3 | ~26K | Vercel (free) or self-hosted | Simple caching + orchestration |
| **Nx** | 22.6.4 | ~25K | Nx Cloud, distributed execution | Enterprise, AI-assisted CI |
| **Moon** | 2.1.3 | ~3K | Moonbase or self-hosted | Polyglot monorepos |

TurboRepo 2.9.3 added composable configuration (v2.7), visual Devtools, and stable `turbo query` — reporting **96% faster task execution** in March 2026. It requires just ~20 lines of `turbo.json`. Nx 22.6.4 offers more power (AI-native MCP server, distributed task execution, TypeScript project references) but adds complexity. **Bun 1.3.11** is viable for greenfield projects and has the fastest raw install times (6–35× faster than npm), but workspace features remain less mature than pnpm's — no catalogs equivalent, and the lockfile format is still evolving.

**Recommendation:** pnpm + TurboRepo. pnpm handles dependency management and workspace linking; TurboRepo handles task orchestration and caching. This combination offers the best balance of maturity, performance, and simplicity.

---

## 4. Tailwind CSS v4 leads styling for white-label theming

For a white-label platform requiring runtime theme switching, **Tailwind CSS v4.2.2** and **vanilla-extract 1.20.1** are the top two choices, with fundamentally different philosophies.

| Approach | Version | npm/week | Runtime Cost | Theme Type Safety |
|----------|---------|----------|-------------|-------------------|
| **Tailwind CSS** | 4.2.2 | ~12M | Zero (purged CSS) | ❌ Manual |
| **vanilla-extract** | 1.20.1 | ~450K | Zero (build-time) | ✅ Full TypeScript |
| **Panda CSS** | 1.0 (stable) | ~200K | Zero (build-time) | ✅ Semantic tokens |
| **StyleX** | 0.17.1 | ~237K | Near-zero (atomic) | ✅ defineVars |
| **CSS Modules** | Built-in | N/A | Zero | ❌ None |
| **Pigment CSS** | 0.0.30 | Very low | N/A | N/A |

**Pigment CSS development is paused** — MUI's "2026 and Beyond" blog explicitly states it is deprioritized. Do not build on it.

Tailwind v4 (released January 2025) introduced CSS-first configuration via `@theme` directives that **natively produce CSS custom properties**. Every design token becomes a `var(--token-name)`, making runtime theme switching natural: define a base theme, then override variables per tenant via `[data-theme="tenant-a"]`. The Oxide engine delivers 3.5–5× faster full rebuilds and **100×+ faster incremental builds** (microseconds). The first-party `@tailwindcss/vite` plugin integrates cleanly with Vite 8.

**vanilla-extract** offers the strongest type safety via `createThemeContract` + multiple `createTheme` implementations — TypeScript enforces that every theme implements the full token contract at compile time. This is purpose-built for multi-tenant theming. The tradeoff is a smaller ecosystem and the `.css.ts` file convention.

**Recommendation:** Tailwind CSS v4 with a thin design token abstraction layer for theme contracts. Use `@theme` for CSS custom property generation and scope overrides per tenant. If compile-time theme enforcement is more valuable than ecosystem breadth, choose vanilla-extract instead.

---

## 5. Vitest 4 is the only modern testing choice

**Vitest 4.1.2** has become the standard testing framework, reaching **~17M weekly downloads** — up from ~7M a year prior. Angular 21 adopted it as its default test runner, replacing Karma/Jest.

| Framework | Version | npm/week | ESM | Browser Mode | Visual Regression |
|-----------|---------|----------|-----|-------------|-------------------|
| **Vitest** | 4.1.2 | ~17M | ✅ Native | ✅ Stable (v4) | ✅ Built-in |
| **Jest** | 30.x | Millions | ⚠️ Experimental | ❌ | ❌ |
| **Bun test** | Built into 1.3+ | N/A | ✅ Native | ❌ | ❌ |

Vitest 4.0 graduated **Browser Mode to stable** — tests run in real Chromium, Firefox, and WebKit via Playwright. It includes built-in visual regression testing (`toMatchScreenshot`), Playwright Traces integration for debugging, and schema matching (Zod, Valibot, Arktype). **Jest 30** (June 2025) brought improvements but ESM support remains experimental after years of development, requiring `--experimental-vm-modules`. Bun test offers ~20× faster raw execution than Jest but lacks browser mode, Vite integration, and visual regression testing.

**Recommendation:** Vitest 4. Native ESM/TypeScript, shared Vite config, stable browser mode, and Jest-compatible API for easy migration.

---

## 6. React 19.2 is the only viable framework for this stack

The Babylon.js and Monaco editor integration requirements make **React 19.2.4** the clear choice — no other framework has comparable ecosystem depth for these critical dependencies.

| Framework | Version | Stars | npm/week | Bundle | Babylon.js | Monaco |
|-----------|---------|-------|----------|--------|-----------|--------|
| **React** | 19.2.4 | ~243K | ~87M | ~40 KB | ★★★★★ react-babylonjs | ★★★★★ @monaco-editor/react |
| **Preact** | 10.29.0 | ~38.5K | ~13.5M | ~3 KB | ★★★★ via compat | ★★★★ via compat |
| **SolidJS** | 1.9.11 (2.0 beta) | ~35.2K | ~1.65M | ~7 KB | ★★ imperative only | ★★ no wrapper |
| **Svelte** | 5.55.1 | ~86.2K | ~3M | ~2–4 KB | ★★★ community | ★★★ community |
| **Lit** | 3.3.2 | ~21.3K | ~4.5M | ~5 KB | ★★ basic | ★★ basic |

**react-babylonjs** provides a mature custom React reconciler with fully declarative JSX for Babylon.js scenes, full TypeScript support, and React 19 compatibility. **Reactylon** (2025–2026) adds cross-platform capabilities. **@monaco-editor/react** (v4+) is the most popular Monaco wrapper for any framework, providing zero-config integration with `<Editor>`, `<DiffEditor>`, and `useMonaco` hook. For panel-based architecture, React offers **react-grid-layout** (drag-and-drop grid dashboards), **GoldenLayout**, **FlexLayout**, **Allotment**, **dnd-kit**, and **pragmatic-drag-and-drop** (Atlassian, framework-agnostic).

**React Compiler v1.0** (October 7, 2025) eliminates the historical performance overhead of `useMemo`/`useCallback` through automatic memoization. React 19.2 added the `<Activity>` component for pre-rendering hidden panels — directly useful for panel-based architectures where panels may be hidden but should remain in memory. The **40 KB framework overhead is negligible** compared to Babylon.js (~1–3 MB) and Monaco (~2–5 MB). The React Foundation launched under the Linux Foundation in February 2026, ensuring long-term governance stability.

**SolidJS 2.0** entered beta with best-in-class fine-grained reactivity, but ecosystem gaps are critical — no Babylon.js bindings, no Monaco wrapper, no panel layout libraries. **Svelte 5.55.1** has excellent runes-based reactivity but Babylon.js and Monaco integrations are immature community projects. **Preact 10.29.0** is a viable alternative if bundle size is a hard constraint, offering React ecosystem access via `preact/compat` at ~3 KB, but it does not support React 19 APIs (Compiler, `<Activity>`, `use()`).

**Recommendation:** React 19.2. The ecosystem advantages for Babylon.js, Monaco, and panel-based layouts are decisive. Server Components can be entirely ignored for this client-only SPA.

---

## 7. uPlot anchors charting while ChartGPU shows WebGPU promise

For **1M+ data points at 100Hz+ sensor rates**, the charting landscape divides sharply between proven Canvas 2D solutions and emerging WebGPU options.

| Library | Version | Stars | Bundle Size | Max Points (60fps) | Rendering |
|---------|---------|-------|-------------|-------------------|-----------|
| **uPlot** | 1.6.24 | ~9.9K | ~50 KB | ~100K visible | Canvas 2D |
| **ChartGPU** | 0.1.9 | New | Early stage | Claims 35M @ 72fps | **WebGPU** |
| **ECharts** | 6.0.0 | ~61K | ~1 MB | Millions (not 60fps) | Canvas 2D/SVG |
| **Plotly.js** | 3.5.0 | ~17K | ~3.6 MB | ~1M static (WebGL) | SVG + WebGL |
| **Lightweight Charts** | 5.1.0 | ~14.2K | ~35 KB | Not designed for scale | Canvas 2D |

**uPlot 1.6.24** renders 166,650 points in 25ms on cold start (~100K pts/ms) and has a working 10M-point demo. At 60fps streaming with 3,600 points it uses just **10% CPU and 12.3 MB RAM** — competitors use 40–70% CPU. The key limitation: it begins to struggle beyond ~100K in-view points at 60fps. The author explicitly recommends WebGL alternatives for massive streaming datasets. For ADAS replay, the solution is **min/max-per-bucket downsampling** from the full 1M+ dataset to a ~50K visible window — this is the same approach used in audio waveform rendering and maintains visual fidelity for peak detection.

**ChartGPU 0.1.9** (January 2026, MIT) is the first dedicated open-source WebGPU charting library. It claims **35M points at ~72fps** with LTTB downsampling as a GPU compute shader, GPU-accelerated hit-testing, and instanced draws. It supports React bindings (`chartgpu-react`), streaming via `appendData()` with TypedArray support, and dark/light themes. However, it is v0.1.x — immature, with concerns about idle CPU usage (render loop ticking when static), suboptimal data format (array-of-arrays versus columnar layout), and the WebGPU context limit (Chrome caps at 16 GL contexts per page). Firefox does not support WebGPU. For a Chrome/Edge-only platform, this is viable to evaluate.

**Lightweight Charts 5.1.0** is designed for financial OHLC data — its data model expects `{time, value}` pairs with unique timestamps and is fundamentally unsuitable for 100Hz sensor data. **Plotly.js** has a 3.6 MB bundle, 310ms load time in benchmarks, and a maximum 8 WebGL contexts per page. **ECharts 6.0.0** handles millions via incremental rendering and TypedArray support but benchmarks show **70% CPU / 85 MB** versus uPlot's 10% / 12.3 MB.

**Recommendation:** uPlot with intelligent downsampling for production today. Evaluate ChartGPU for GPU-accelerated rendering in a future iteration. For the most demanding visualizations, consider a custom WebGPU renderer using Babylon.js's existing WebGPU capabilities (Three.js r171+ has production WebGPU since September 2025).

---

## 8. Raw postMessage with TypeScript types for SharedArrayBuffer workers

**No existing library manages SharedArrayBuffer coordination directly.** For zero-copy SAB threading, **typed postMessage is the recommended approach**.

| Library | Version | Stars | Downloads/wk | SAB Support | Active? |
|---------|---------|-------|-------------|-------------|---------|
| **Comlink** | 4.4.2 | 12.6K | ~900K | Indirect (passthrough) | ⚠️ Stagnant |
| **threads.js** | 1.7.0 | ~3K | ~575K | ❌ | ❌ Dead (4+ years) |
| **Penpal** | 7.x | ~514 | ~113K | ❌ | ✅ Active (v7 added workers) |
| **Raw postMessage** | Native | N/A | N/A | ✅ Full | N/A |

**Comlink 4.4.2** (Google Chrome Labs) uses ES6 Proxies to make cross-worker calls feel like local async functions. At ~1.1 KB gzipped, it's excellent for RPC-style communication. However, it is **maintenance-stagnant** — 76 open issues, 41 open PRs, no releases in over a year, and original creator Surma has left Google. It does not provide SharedArrayBuffer abstractions or `Atomics` synchronization. **No new worker communication library has emerged in 2025–2026** that targets SAB + typed communication for high-performance use cases.

For the ADAS platform's architecture, use TypeScript discriminated unions for compile-time message type safety, `SharedArrayBuffer` for shared memory (zero-copy, requires cross-origin isolation headers), `Atomics.wait()`/`Atomics.notify()` for synchronization, and Transferable objects for one-way ArrayBuffer ownership transfer. Comlink can complement this for higher-level RPC (e.g., calling utility functions across workers) while the core SAB communication layer uses typed postMessage directly.

**Recommendation:** Raw postMessage with TypeScript discriminated union types for all SAB communication. Optionally layer Comlink on top for convenience RPC where SAB isn't needed. Budget for Comlink potentially becoming unmaintained and plan for replacement if needed.

---

## 9. Monaco Editor dominates the extension IDE, with esbuild-wasm for bundling

### Code editor

**Monaco Editor 0.55.1** provides VS Code–level TypeScript IntelliSense out of the box — diagnostics, auto-imports, parameter hints, go-to-definition — with language services running automatically in **dedicated Web Workers**. No other editor matches this for a hosted TypeScript IDE.

| Editor | Version | Stars | npm/week | TS IDE | Bundle Size |
|--------|---------|-------|----------|--------|-------------|
| **Monaco** | 0.55.1 | ~45.7K | ~4M | ✅ Built-in (full) | ~5 MB (optimizable to ~2.4 MB) |
| **CodeMirror 6** | 6.0.2+ | ~10K | Higher combined | ❌ Requires integration | ~300 KB–1 MB |
| **Ace** | 1.43.5 | ~27.1K | ~1.2M | ❌ Syntax only | ~1–2 MB |

CodeMirror 6 is modular and lightweight (~300 KB basic) with excellent mobile support, but TypeScript language services require significant custom integration — the primary community solution (`codemirror-ts` by Val Town) was **archived in September 2025**. Ace Editor (1.43.5) provides only syntax highlighting for TypeScript with no language service — not suitable for an IDE. Any editor integrating the full TypeScript compiler will need ~5–8 MB for the TS compiler loaded in a Web Worker; Monaco handles this transparently. **modern-monaco** (esm-dev) is a new wrapper that loads modules on-demand from CDN with Shiki pre-highlighting to reduce perceived load time.

### In-browser bundler

**esbuild-wasm 0.27.4** is the only production-ready option for in-browser TypeScript bundling.

| Tool | Can Bundle? | WASM Size | Browser Ready? |
|------|------------|-----------|---------------|
| **esbuild-wasm** | ✅ | ~11 MB | ✅ Production (powers bundlejs.com) |
| **@swc/wasm-web** | ❌ Transform only | ~2–3 MB | ✅ (transform only) |
| **Rolldown WASM** | ✅ (needs WASI) | ~3–5 MB | ❌ Requires WASI runtime |
| **@rollup/browser** | ✅ | ~2–3 MB | ✅ Slower than esbuild |

esbuild-wasm is ~10× slower than native esbuild (due to Go's single-threaded WASM compilation), but native esbuild is so fast that this is still adequate for interactive extension bundling. The full plugin API works in-browser — critical for implementing virtual file systems, HTTP module resolution, and custom transforms. **Rolldown's WASM build** targets `wasm32-wasip1-threads` (WASI) and is not yet directly usable in standard browsers without a WASI polyfill; the team confirms it only works in environments like StackBlitz WebContainers. **@swc/wasm-web** (1.15.4, ~2–3 MB) is excellent for single-file transpilation but cannot bundle — SWC plugins are also not supported in WASM.

**Recommendation:** Monaco Editor + esbuild-wasm. Total additional download: ~5 MB (Monaco) + ~11 MB (esbuild.wasm) = ~16 MB, fully cacheable. Watch Rolldown for future browser-native WASM support.

---

## 10. mp4box.js is the WebCodecs demuxing standard

**There is no native browser demux API in WebCodecs** — the W3C spec explicitly covers only encoding/decoding. Third-party libraries are the only path.

| Library | Version | Stars | Format Support | WebCodecs Native |
|---------|---------|-------|---------------|-----------------|
| **mp4box.js** | 2.3.0 | ~2.3K | MP4/ISOBMFF only | ✅ W3C recommended |
| **web-demuxer** | 4.0.0 | ~180 | MP4/WebM/MKV/FLV/AVI/TS | ✅ First-class API |
| **Mediabunny** | New | New | MP4/WebM/MKV | ✅ Pure TS, tree-shakable |
| **jMuxer** | 2.0.x | ~1.2K | N/A | N/A |

**mp4box.js 2.3.0** (November 2025) is the **de facto standard** — W3C WebCodecs samples and MDN documentation officially reference it. It provides progressive parsing (streaming-friendly), sample extraction per track, and produces data compatible with `EncodedVideoChunk` / `EncodedAudioChunk`. The v2.x series includes modern ESM/CJS/IIFE builds via tsup and TypeScript definitions. The main limitation is MP4-only support.

**web-demuxer 4.0.0** (Bilibili, December 2025) is WASM-based with a **WebCodecs-first API**: `getDecoderConfig()` returns `VideoDecoderConfig` directly, `seek()` returns `EncodedVideoChunk`. It supports MP4/WebM/MKV/FLV and more via customizable FFmpeg-derived builds. The tradeoff is WASM overhead and a smaller community (180 stars). **Mediabunny** is an emerging pure-TypeScript toolkit (~11 KB for WebM muxer only) that deprecated both `webm-muxer` and `mp4-muxer` in its favor. **jMuxer is a muxer, not a demuxer** — it creates MP4 segments from raw H264/AAC and is not applicable for demuxing existing files.

**Recommendation:** mp4box.js for MP4 demuxing. If WebM support is also needed, evaluate web-demuxer. Build a thin wrapper to extract WebCodecs-compatible config (`VideoDecoderConfig`) from mp4box.js samples.

---

## 11. Serwist modernizes service worker tooling

| Tool | Version | Stars | npm/week | Vite Plugin | TypeScript-First |
|------|---------|-------|----------|------------|-----------------|
| **Workbox** | 7.4.0 | ~12.9K | ~6.4M (core) | ❌ Via vite-plugin-pwa | ❌ JS + types |
| **Serwist** | 9.5.7 | ~1.4K | Growing | ✅ @serwist/vite | ✅ Native |

**Serwist 9.5.7** is the modern, TypeScript-first successor forked from Workbox when its development stagnated in 2023. It is **ESM-only**, merges all SW packages into a single `serwist` import, removes GenerateSW in favor of InjectManifest-only (more control), and provides native plugins for **Vite, Next.js, Nuxt, Webpack, and Turbopack**. The v10 preview adds React Router, Astro, and SvelteKit support. Serwist's creation actually helped resurrect Workbox development — both projects now coexist healthily.

**Workbox 7.4.0** remains safe and proven with ~6.4M weekly downloads, but lacks native Vite integration (requires the third-party `vite-plugin-pwa` wrapper) and uses CJS with bolted-on TypeScript types. For caching extension bundles specifically, both support custom runtime caching rules — Serwist's InjectManifest-only approach gives more control over complex scenarios like extension bundle versioning.

**Recommendation:** Serwist for new Vite-based projects. Native Vite plugin, TypeScript-first, ESM-only, actively maintained with frequent releases.

---

## Conclusion: a cohesive, modern stack

The recommended stack forms a coherent, well-integrated foundation:

- **React 19.2 + Zustand 5 + zundo** — declarative UI with lightweight undo/redo and Worker-compatible state
- **Vite 8 (Rolldown) + pnpm 10 + TurboRepo 2.9** — fast builds, efficient dependency management, intelligent task caching
- **Tailwind CSS v4** — CSS-variable-powered theming with zero runtime cost
- **Vitest 4** — native ESM testing with stable browser mode
- **uPlot** — proven 100Hz+ sensor data visualization with downsampling
- **Monaco + esbuild-wasm** — full TypeScript IDE experience with in-browser bundling
- **mp4box.js 2.3** — W3C-standard MP4 demuxing for WebCodecs
- **Raw postMessage + TypeScript** — typed SAB communication where no library suffices
- **Serwist 9.5** — modern service worker tooling with native Vite integration

Two strategic technology watches deserve attention: **ChartGPU** for WebGPU-accelerated charting (evaluate once it reaches v0.5+), and **Rolldown browser WASM** as a potential esbuild-wasm replacement once WASI browser support matures. The TC39 Signals proposal is too early (Stage 1, 2028+ for browser shipping) to influence architecture decisions today, but Zustand's vanilla store pattern aligns well with a future signals-based migration if needed.