# Time

**Read this when:** touching timestamps anywhere — pipeline, video, scene graph, UI.

## The type

```typescript
/** Integer microseconds since the Unix epoch. Branded — not interchangeable with number. */
export type Time = number & { readonly [TimeBrand]: true };
declare const TimeBrand: unique symbol;

export function timeFromUsec(usec: number): Time;            // asserts integer
export function timeFromSecUsec(sec: number, usec: number): Time;
export function timeFromNanos(nanos: bigint): Time;          // truncates; see MCAP note
export function timeAddUsec(t: Time, deltaUsec: number): Time;
export function timeDiffUsec(a: Time, b: Time): number;
export function timeToDate(t: Time): Date;
export function timeToSecUsec(t: Time): { sec: number; usec: number }; // wire/display only
```

A `Time` is a plain `number` at runtime: compare with `<`, sort numerically, subtract for
deltas, store in `Float64Array` columns. There are no `Time` objects and no bigint arithmetic
anywhere in the pipeline.

## Why integer microseconds in a `number`

- **Precision is sufficient by a wide margin.** 2^53 µs ≈ 285 years; integer microsecond
  arithmetic in an IEEE double is exact until the year ~2255. AV sensor cadences (IMU at
  400 Hz = 2500 µs spacing) are differentiated with three orders of magnitude to spare.
- **It is natively the WebCodecs domain.** `EncodedVideoChunk.timestamp` and
  `VideoFrame.timestamp` are integer microseconds — the video pipeline needs no conversion
  layer at its hottest boundary.
- **Allocation-free.** The archived PRD's `{sec, usec}` struct allocated one object per
  message; at 100–400 Hz across dozens of topics, plus ghost/trail buffers, that is real GC
  pressure on the data plane. A branded number costs nothing and packs into typed-array
  columns (`data-plane.md#columnar`).
- **One arithmetic domain.** The archived PRD/M0 plan mixed `number` and `bigint` APIs;
  mixed domains force conversions at every call site and were themselves a smell.

## Nanosecond sources (MCAP)

MCAP log/publish times are uint64 **nanoseconds**. Nanoseconds-since-epoch (~1.8 × 10^18)
exceed 2^53, which is precisely why nanoseconds-as-`number` is impossible and microseconds
was chosen. Policy:

- `timeFromNanos` truncates to microseconds. Sub-microsecond ordering information is
  discarded — acceptable: no supported sensor emits meaningfully sub-µs-spaced data.
- Messages landing on the same microsecond order deterministically by (channel id, log index).
- Round-trip fidelity (re-export at original ns precision) is out of scope for v1; if an
  exporter later needs it, original ns values can be carried as opaque metadata without
  touching pipeline arithmetic.

## Rules

1. All internal synchronization uses `performance.now()` / `performance.timeOrigin`
   (µs-resolution monotonic), never `Date.now()`.
2. The timeline UI *displays* at ms granularity but *stores* seek positions as `Time`;
   zooming reveals sub-ms spacing.
3. Never destructure a `Time` into sec/usec except at wire and display boundaries
   (`timeToSecUsec` exists so the temptation has a supervised exit).
4. Durations are plain `number` microseconds (unbranded); only absolute instants are `Time`.
