# Data Sources & Sessions

**Read this when:** implementing a container reader, codec, data-source transport, or anything
that feeds the message pipeline.

## Contract

```typescript
interface DataSource {
  readonly id: string;
  readonly displayName: string;

  initialize(): Promise<DataSourceInfo>;   // topics, schemas (full trees), time range, stats
  getInfo(): DataSourceInfo;

  /** Forward iteration for playback. Batched to amortize boundary costs. */
  messageIterator(args: {
    topics: readonly string[];
    start: Time;
    end: Time;
  }): AsyncIterable<MessageBatch>;

  /**
   * Latest message at-or-before `time`, per topic. THE seek primitive: without it, a seek
   * lands on a blank screen until each topic next publishes — calibration topics may publish
   * once per session. Every panel's post-seek correctness depends on this.
   */
  getBackfillMessages(args: {
    topics: readonly string[];
    time: Time;
  }): Promise<readonly MessageEvent[]>;

  getVideoStreams(): readonly VideoStreamDescriptor[];

  /** Streamed decode-ahead, symmetric with messageIterator (not a per-segment poll). */
  videoSegmentIterator(args: {
    streamId: string;
    start: Time;
  }): AsyncIterable<EncodedVideoSegment>;

  close(): Promise<void>;
}

interface MessageEvent {
  readonly topic: string;
  readonly timestamp: Time;
  readonly schemaName: string;
  readonly data: Uint8Array;      // view into the pipeline arena — NOT owned by the consumer
  readonly sizeInBytes: number;
}
```

Multiple concurrent iterators must be supported (playback cursor + block loader + thumbnail
builder run simultaneously).

## Subscription tiers

Two tiers, declared per subscription (the review traced several Foxglove panel pathologies to
the absence of this distinction in v1 designs):

- **`current-frame`** — feeds playback: messages near the cursor, delivered in play order,
  plus backfill on seek.
- **`full-range`** — feeds whole-session views (Plot, timeline density/heatmap): the block
  loader progressively fetches the entire session for these topics into a memory-budgeted
  cache, evaluating message paths and downsampling *in the decode worker* into numeric
  columns (`data-plane.md`). The timeline UI shows loaded ranges as progress.

## Sessions are directories

Real drive sessions are multi-file: segmented MCAPs + sidecar MP4s + calibration JSON. A
`Session` composes `DataSource`s into one timeline:

```typescript
interface Session {
  readonly sources: readonly DataSource[];   // merged topic namespace, merged time range
  readonly timeRange: { start: Time; end: Time };
}
```

- Grouping: user multi-select or directory pick; an optional `session.json` manifest wins when
  present; otherwise heuristics (shared basename prefixes, timestamp-range overlap) propose a
  grouping the user confirms.
- Sidecar video maps to `VideoStreamDescriptor` with time offset + calibration from the
  manifest or a calibration file.
- Sessions stay first-class (plural) in the app model so v1.x run-comparison (two sessions,
  aligned timelines) is an additive feature, not a refactor.

## Codecs

```typescript
interface SchemaCodec {
  readonly schemaFormat: string;   // "protobuf" | "jsonschema" | "ros2msg" | ...
  /** Parse once, decode many. Schema parsing per message is a design defect. */
  compile(schema: Uint8Array): CompiledSchema;
}

interface CompiledSchema {
  decode(data: Uint8Array): unknown;
  /** Full recursive field tree with types — powers message-path autocomplete and settings UI. */
  readonly tree: SchemaTree;
  /** Optional: decode a single path without full deserialization (lazy decode). */
  decodePath?(data: Uint8Array, path: MessagePath): unknown;
}
```

The registry is injected; core never imports a codec. `decodePath` is optional in v1 but the
contract reserves it: Raw Messages showing three fields of a 40 KB message should not require
full deserialization forever.

## Memory budget

Sessions are 10–200 GB; RAM is not. This section is normative — replay tools die here.

- **Global budget** (default ~60 % of `navigator.deviceMemory`-derived heuristic, configurable)
  split across: block cache (full-range tier), chunk cache (HTTP range source), decoded-message
  LRU, video decode-ahead (`video-pipeline.md`), point-cloud retention window.
- **Block cache**: per-topic full-range data, LRU-evicted by (range, topic) block; evicted
  ranges reload transparently; loaded-ranges UI reflects reality.
- **Chunk cache** (remote sources): fixed-size chunks, LRU + sequential prefetch during
  playback, priority to the cursor's neighborhood on seek.
- **Never** an unbounded `Map` keyed by topic or time anywhere in the pipeline. Every cache
  declares its budget and its eviction policy in code review.

## Seek sequence (normative)

1. Timeline store sets target `Time`, increments the pipeline **epoch** (`data-plane.md#epochs`).
2. In-flight iterators for the old epoch are cancelled; arena pages from the old epoch are
   reclaimed once their readers retire.
3. `getBackfillMessages` for all current-frame subscriptions at the target time; decoded and
   mapped (scene mappers run on backfill too — the 3D scene must be correct while paused).
4. Video pipeline seeks each visible stream to the preceding keyframe and decodes forward
   (`video-pipeline.md`).
5. Forward iteration resumes from the target if playing.

Every stage tags its outputs with the epoch; consumers drop mismatched epochs on arrival.
