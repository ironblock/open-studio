# State management in 2026: Redux vs Zustand deep dive

**Zustand has overtaken Redux Toolkit in weekly npm downloads (~24M vs ~10.6M) and become the default choice for new React projects, but Redux remains firmly entrenched in complex enterprise codebases.** The two libraries have converged on complementary roles rather than direct competition: Zustand dominates for client-side UI state and smaller teams, while Redux Toolkit retains advantages in structured side-effect management, enforced architectural patterns, and power-user debugging. This report answers six specific questions about the state of both ecosystems, their tooling gaps, and an emerging pattern for SharedArrayBuffer-backed stores.

---

## Zustand's devtools remain a Redux DevTools wrapper with incremental gains

Zustand **has no official first-party browser extension**. The primary and officially documented debugging approach is the `devtools` middleware from `zustand/middleware`, which connects to the Redux DevTools Chrome extension. This means Zustand's debugging story is fundamentally dependent on Redux's tooling infrastructure.

Through this middleware, Zustand supports **action logging, time travel debugging, state diff viewing, and export/import state** — all via Redux DevTools' built-in capabilities. However, several friction points persist. Actions default to "anonymous" unless manually named via a third argument to `set()`, though **v5.0.4** (2025) added automatic action type inference from function names, significantly reducing this burden. Call stack tracing remains unsupported (acknowledged in GitHub Issue #716). Large state payloads cause slow serialization (Issue #926), and persist middleware rehydration doesn't display correctly in DevTools after page refresh (Issue #966).

Third-party Zustand-specific extensions exist but are largely dormant. **Zukeeper** (~9,000 Chrome users, last updated ~2023) offers state hierarchy visualization and diff-ing. **Zusty** (~1,000 users, last updated February 2024) adds render time metrics and component tree views. Both require separate npm middleware packages and appear unmaintained.

The most notable 2025 development is **`@csark0812/zustand-expo-devtools`** (84 GitHub stars, 14 releases through December 2025), which provides real-time state inspection and time travel debugging integrated into Expo's DevTools platform for React Native. Additionally, **`@sucoza/zustand-devtools-plugin`** integrates Zustand store inspection into TanStack DevTools. Neither is a general-purpose browser extension replacement. The pmndrs team has shown no indication of building a dedicated Zustand browser extension — the Redux DevTools integration is considered the permanent official approach. The practical assessment from multiple 2025 sources: Zustand's DevTools middleware covers roughly **80% of debugging needs** but misses power-user features like action replay and call stack tracing that Redux DevTools offers natively.

---

## There is no Zustand equivalent of Redux Saga — by design

Zustand deliberately has **no built-in structured side-effect management** comparable to Redux Saga's generator-based primitives (`takeEvery`, `takeLatest`, `race`, `fork`, `cancel`). This is an architectural philosophy choice, not an oversight. The Zustand README states plainly: "Just call `set` when you're ready, zustand doesn't care if your actions are async or not."

The dominant pattern is **plain async/await directly in store actions**. This handles the vast majority of real-world async flows — API calls, loading states, error handling — without additional abstractions. For reactive side effects triggered by state changes, Zustand ships `subscribeWithSelector` middleware, which allows subscribing to specific state slices and running callbacks when they change. This is the closest built-in analog to a saga "watcher," but it lacks concurrency primitives entirely.

The one attempt at a true saga bridge, **`zustand-saga`** (GitHub: Nowsta/zustand-saga), has only ~30 stars and 4 total commits. It wraps the actual `redux-saga` library as Zustand middleware and uses the old v3 API — effectively a proof-of-concept, not production software. Other related libraries take different approaches: **`zustand-rx`** exposes stores as RxJS Observables, **`zustand-middleware-xstate`** integrates XState state machines, and **`leo-query`** (listed in official Zustand third-party docs) provides structured query/effect primitives with caching.

The community consensus, reinforced by multiple 2025–2026 articles and the Brainhub consultancy's architecture guide, is a **separation of concerns**: Zustand for client state (UI toggles, modals, form state) paired with **TanStack Query or SWR for server state** (data fetching, caching, optimistic updates). This combination replaces most saga use cases around data fetching. For users who genuinely need saga-level orchestration — complex multi-step workflows with cancellation, racing, and forking — the frank community advice remains: use Redux with `createListenerMiddleware` (RTK's official lightweight saga replacement) or, for the most complex cases, Redux Saga itself.

---

## Zustand selectors work but memoization requires external libraries

Zustand's hook-based API supports selectors natively — `useBearStore((state) => state.bears)` — using **strict equality (`===`)** by default to determine re-renders. For primitive values and stable references, this is efficient out of the box. For selecting multiple values, Zustand provides `useShallow` (from `zustand/react/shallow`), which performs shallow comparison of the selector output. This is critical in Zustand v5, where selecting an object without `useShallow` can trigger **infinite render loops** — a migration pitfall documented in multiple case studies.

However, **Zustand has no built-in memoization equivalent to Reselect's `createSelector`**. The distinction is important: `useShallow` recomputes the selector every time and then compares the result, while Reselect skips recomputation entirely if inputs haven't changed. Zustand maintainer dai-shi explicitly addressed this in Discussion #387: "If you want to avoid extra calculation, memoization is necessary. reselect is one and proxy-memoize is another. zustand is unopinionated about choosing memoization libraries."

Reselect works seamlessly with Zustand — you pass a `createSelector` output directly as the hook's selector function. No adapter needed. Dai-shi's own **`proxy-memoize`** library offers an alternative using Proxy and WeakMap for automatic dependency tracking, avoiding manual input selector declarations. For computed/derived state injected directly into the store, several middleware options exist:

- **`zustand-computed`** (~4,716 weekly npm downloads) automatically recomputes derived values when dependencies change and maintains referential equality
- **`derive-zustand`** (v0.2.0, by dai-shi under the official zustandjs GitHub org) creates separate derived stores from other Zustand stores, useful for cross-store computations
- **`zustand-x`** provides a store factory with chainable `extendSelectors` for derived computations

The auto-generating selectors pattern documented in Zustand's official guides (`createSelectors` utility) generates per-field selector hooks (`useBearStore.use.bears()`) but does not add memoization — it's a convenience wrapper for common field access patterns. The bottom line: Zustand's selector story is **flexible but assembly-required** for complex derived data, whereas Redux Toolkit bundles Reselect's `createSelector` out of the box.

---

## Redux Toolkit thrives while Redux Saga fades into maintenance mode

**Redux Toolkit** remains actively maintained at **v2.11.2** (released December 14, 2025) by Mark Erikson and Ben Patience. The 2025 release cycle included the Infinite Query API for RTK Query, Immer performance optimizations, SSR subscription leak fixes, TypeScript 5.4+ as the minimum supported version, and initial compatibility testing with TypeScript 7.0 (`tsgo`). RTK has ~10.6M weekly npm downloads and 11.2K GitHub stars.

**Reselect** sits at **v5.1.1**, stable and mature but with no new releases in over 12 months. This reflects its completeness rather than abandonment — Reselect 5.0 shipped alongside RTK 2.0 with `weakMapMemoize` as the new default (effectively infinite cache size via WeakMap/Map tree) and improved TypeScript performance. It maintains **~18–22M weekly downloads**, inflated by being a transitive dependency of RTK. It is bundled inside Redux Toolkit and re-exported from `@reduxjs/toolkit`.

**Redux Saga** at **v1.4.2** (November 2025) is **effectively in low-maintenance mode**. Multiple GitHub issues opened throughout 2025 remain unanswered by maintainers. A March 2025 blog post described it as deprecated, and its weekly downloads have declined to **~831K–1.4M** — a fraction of its historical peak. The Redux team's official recommended replacement is `createListenerMiddleware` for reactive side-effect logic and RTK Query for data fetching.

The current recommended Redux stack is clear and well-documented:

- **Store setup**: `configureStore` (the original `createStore` is officially deprecated)
- **Reducer logic**: `createSlice` with Immer
- **Async logic**: `createAsyncThunk` for promise-based flows
- **Data fetching**: RTK Query (built into `@reduxjs/toolkit/query`)
- **Side effects**: `createListenerMiddleware` (replaces sagas/observables)
- **Selectors**: `createSelector` (Reselect, bundled)
- **React bindings**: `useSelector` and `useDispatch` hooks (not `connect()`)

**Redux-Observable** remains stuck at a release candidate (3.0.0-rc.3) with only ~180–253K weekly downloads, effectively sidelined.

---

## SharedArrayBuffer-backed state stores remain theoretical

The specific pattern of using SharedArrayBuffer as the backing data structure for a Redux or Zustand store — where a Web Worker writes data into the SAB and the UI thread reads via selectors — **does not exist as a library or widely adopted pattern**. No npm package, GitHub repo, or production case study implements this exact architecture.

The closest real-world example is **Third Room / Manifold Engine** (matrix-org/thirdroom), a multi-threaded web game engine that uses **SAB-backed TypedArrays** for its scene graph with a TripleBuffer synchronization pattern. A worker writes scene data into SAB, and the render thread reads it — exactly the requested pattern, but for game engine ECS data, not React state management.

Several libraries move Redux to a Web Worker but use **message-passing, not SharedArrayBuffer**: `redux-in-worker` (by dai-shi) sends state diffs over postMessage, `react-redux-worker` uses Comlink for RPC, and `stockroom` (by Jason Miller) offloads Unistore to a worker. Andrea Giammarchi's **`coincident`** library provides SAB + Atomics infrastructure for synchronous cross-thread communication that could theoretically underpin such a store.

Four fundamental challenges explain why this pattern hasn't materialized:

**The serialization problem** is the most critical. SAB stores raw binary data in typed arrays; Redux/Zustand stores hold arbitrary JavaScript objects (strings, nested objects, arrays). Converting between these representations requires serialization that negates SAB's zero-copy advantage. This pattern works for **numeric/structured data** (game state, sensor readings, financial ticks) but is impractical for typical application state (user profiles, todo lists, UI flags).

**The notification problem** compounds this. When a worker writes to SAB, the main thread receives no notification. Solutions require either polling (wasteful), `Atomics.waitAsync()` (limited browser support on the main thread), or a side-channel postMessage — which partly defeats SAB's purpose, requiring a hybrid approach.

**Cross-origin isolation requirements** (`Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` headers) break many third-party integrations including OAuth popups, embedded iframes, and analytics scripts. The `sabayon` polyfill offers a ServiceWorker-based workaround.

**React integration** is theoretically possible via `useSyncExternalStore`, which both Redux and Zustand already use internally. A SAB-backed store could implement `subscribe` and `getSnapshot`. But `getSnapshot` must return an immutable snapshot, requiring copying data from the mutable SAB into a JavaScript object on each read — again negating the zero-copy benefit. The SABO blog's April 2024 analysis of multi-threaded React WebGL applications explicitly evaluated SAB and chose valtio + BroadcastChannel instead for developer ergonomics.

---

## Enterprise adoption shows a clear division of labor emerging

Multiple 2025–2026 case studies and architectural guides reveal a converging consensus rather than a winner-take-all outcome. **Zustand has become the default for new React projects** — the Meerako consultancy reports choosing Zustand for "90% of SaaS platforms, MVPs, and enterprise dashboards." An e-commerce migration reported **60% boilerplate reduction and 25% decrease in rendering time**. A React Native migration case study highlighted Zustand's `getState()` and `subscribe()` for outside-React access as its most underrated advantage.

Redux retains clear advantages for specific enterprise requirements. Financial and compliance applications needing **audit trails of every state mutation**, teams of **50+ developers** needing enforced architectural patterns, applications requiring **time-travel debugging with state export/import** for bug reproduction, and complex cross-cutting state with interdependent domains are all cited as Redux strongholds across multiple sources.

The **hybrid approach** is increasingly seen as the pragmatic best practice. One e-commerce company profiled by Toxigon uses Redux for cart/checkout flow (complex interdependent state) and Zustand for UI toggles and preferences. The Brainhub consultancy demonstrated that Redux's architectural concepts — events, reducers, pure functions — can be preserved within Zustand's simpler API via patterns like `convertReducersToActions`, giving teams Redux-style discipline without Redux's boilerplate.

The undo/redo comparison illustrates the ecosystem maturity gap narrowing. Zustand's **zundo** library (<700 bytes, 162K+ weekly downloads) provides temporal middleware with undo, redo, pause, and resume. **zustand-travel** uses JSON Patch (RFC 6902) for incremental storage, dramatically reducing memory for large state histories compared to redux-undo's full-snapshot approach.

Mark Erikson (Redux maintainer) noted on Hacker News that RTK still has ~30M monthly downloads and pushed back against "nobody uses Redux" narratives. The data supports both positions: Zustand surpassed Redux Toolkit in weekly npm downloads during 2025, but a substantial portion of Redux usage reflects deep enterprise entrenchment rather than mere legacy inertia — these codebases actively depend on Redux's structured middleware, normalized entity management (`createEntityAdapter`), and integrated data fetching (RTK Query).

---

## Conclusion

The Zustand vs Redux question in 2026 has resolved into a **complementary ecosystem** rather than a binary choice. Zustand's devtools gap is real but narrowing — the v5.0.4 inferred action types and third-party TanStack DevTools integration help, though no first-party browser extension is planned. The absence of a saga equivalent is philosophical, not accidental; the community answer is async/await plus TanStack Query, which genuinely handles most real-world async patterns. Selector memoization requires assembly from external libraries (Reselect, proxy-memoize, zustand-computed), unlike RTK's bundled approach.

SharedArrayBuffer-backed stores remain a theoretical architecture with real technical barriers — the serialization and notification problems make it impractical for typical application state, though the game engine world (Third Room/Manifold, bitECS) demonstrates it works for numeric ECS data. The most promising bridge would combine `coincident`'s Atomics infrastructure with `useSyncExternalStore`, but no one has built this yet.

The sharpest insight from the research: **the decision axis has shifted from "which is better" to "which state belongs where."** The emerging best practice treats client UI state (Zustand) and server/async state (TanStack Query or RTK Query) as fundamentally different concerns, making the Redux-vs-Zustand framing increasingly less relevant for greenfield projects — while making Redux's structured patterns increasingly valuable for the specific subset of state that truly demands them.