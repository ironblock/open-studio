# State Management & Undo/Redo

**Read this when:** adding app state, wiring undo, or tempted to put data in a store.

## The invariant

**Zustand holds the control plane only.** Layout, panel settings, bookmarks, scene filters,
selection, playback *controls*, theme — state that changes at human rate. Sensor-rate data
(messages, frames, scene columns) never enters a store, a React context, or React props; it
lives on the data plane (`data-plane.md`) and reaches panels via `onFrame`
(`panel-api.md`).

This is the durable lesson from the predecessor: it *had* migrated its pipeline state to
Zustand internally and still janked, because the problem was frame-rate data in the reactive
graph, not the store library. The research doc's conclusion
(`../research/state-management-2026.md` — "SAB-backed stores remain theoretical") stands: we
do not build a SAB-backed store; we build a SAB data plane *beside* the stores.

## Stores

| Store | Contents | Undoable |
|---|---|---|
| `layout` | layout tree, panel settings, bookmarks/annotations, scene filters | ✅ |
| `session` | loaded session descriptor, source status, loaded ranges | ❌ |
| `timeline` | cursor `Time`, playing, speed, epoch counter | ❌ |
| `selection` | selection + hover (`panel-api.md#selection-bus`) | ❌ |
| `theme` | tokens, brand config | ❌ |

Vanilla-store creation (`zustand/vanilla`) with React bindings layered on, so workers and
non-React code (layout engine, extension host) can read/subscribe without React.

## Undo/redo

Zustand + `zundo` temporal middleware on the `layout` store, `partialize`d to the undoable
fields above.

**Grouping, not a global debounce.** The archived PRD sketched a single 300 ms debounce in
`handleSet` — that coalesces *distinct* user actions (a panel move and a settings tweak
within 300 ms merge into one undo step, and undo visibly "eats" actions). Instead:

- Continuous gestures (split-drag, slider scrub) group into one entry per gesture
  (group on pointer-down → commit on pointer-up).
- Discrete actions (move panel, add bookmark, toggle filter) are one entry each, no debounce.
- Text inputs debounce per field.

History depth is bounded (`limit`); large-state growth is not a concern because sensor data
never enters the store (the invariant does double duty).

## React usage rules

- Selectors select primitives or use `useShallow`; object-returning bare selectors are a
  lint error (Zustand v5 infinite-loop footgun).
- Derived data: memoized selector utilities in the store module — components never compute
  derived state inline from whole-store subscriptions.
- Panel chrome subscribes to *its own* panel's settings slice, nothing wider.
- Tabbed/hidden panels: React 19 `<Activity>` keeps chrome state warm while skipping render
  work, and the layout store pauses their data-plane subscriptions
  (visibility feeds the video decoder budget too — `video-pipeline.md`).
- Redux DevTools middleware enabled in dev builds; actions named.
