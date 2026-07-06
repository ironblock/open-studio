# Video Pipeline

**Read this when:** working on video decode, camera panels, thumbnails, or seek behavior.

Video is first-class: a camera-first autonomy world means codec-compressed streams (H.264,
H.265, AV1) with frame-accurate seeking and multi-camera sync — not sequences of JPEGs.

## Scope

- **Codecs (v1):** H.264, H.265, AV1 via WebCodecs `VideoDecoder`. VP9 deferred to v1.x — it
  is a web codec, not a vehicle codec, and cutting it shrinks the test matrix by a quarter.
- **Sources:** standalone MP4/WebM sidecar files (demuxed via mp4box.js / WebM parser), and
  video embedded in MCAP channels using a `foxglove.CompressedVideo`-compatible schema (the
  convention the existing corpus uses).
- **Fallback:** keyframe JPEG/PNG stills when WebCodecs is unavailable. No WASM ffmpeg.
- Patent compliance is the deployer's responsibility.

## Architecture

- **Demux and decode are co-located** in per-stream-group workers: compressed chunks never
  cross a thread boundary (`data-plane.md` honesty table). Decoded `VideoFrame`s transfer to
  the UI thread (WebCodecs transferable — zero copy).
- **Keyframe index** built at session open (from container tables when present; single
  background scan otherwise): `KeyframeEntry[] = {timestamp, byteOffset}` per stream. Powers
  frame-accurate seek and the thumbnail strip.
- **Seek:** jump to preceding keyframe, decode forward to target, present exactly one frame
  per stream at-or-before the seek time (video's backfill analogue), tagged with the pipeline
  epoch (`data-plane.md#epochs`) so late frames from a superseded seek are dropped.
- **Sync:** frame *selection* is aligned to the global `Time` cursor with sub-millisecond
  timestamp precision (`time.md` — `VideoFrame.timestamp` is already integer µs). Display
  sync is bounded by the display's refresh; the spec claims timestamp alignment, not photon
  alignment.
- **Thumbnail strip:** background worker decodes keyframes at reduced resolution into
  `ImageBitmap`s for the timeline scrubber; lowest decoder priority (below).

## Decoder budget manager

The archived PRD promised "12 synchronized streams" without confronting platform reality:
**hardware decode sessions are scarce** (commonly single digits, codec- and OS-dependent —
often fewer for H.265), and falling off the hardware cliff means software decode and a CPU
explosion. A seek across 12 streams naively triggers 12 keyframe-to-target decode storms.

The budget manager is therefore a required component, not an optimization:

- Maintains a probed estimate of available hardware sessions per codec
  (`VideoDecoder.isConfigSupported` + empirical fallback detection).
- **Priority order:** visible playing panels → visible paused panels → decode-ahead for
  playing streams → offscreen/backgrounded streams → thumbnails.
- Over-budget streams degrade explicitly: keyframe-only decoding (reduced effective frame
  rate), then stills — surfaced in the panel UI ("reduced rate: decoder budget"), never
  silent jank.
- Decode-ahead runs only while playing, within a memory budget from the global allocation
  (`data-sources.md#memory-budget`); `VideoFrame`s are closed aggressively (they pin GPU
  memory).
- Seeks are staged: visible streams first, then the rest as sessions free up.

## Camera↔3D composition

Both directions use `CameraIntrinsics`/`CameraExtrinsics` from `VideoStreamDescriptor`
(pinhole, fisheye, equidistant models; distortion coefficients):

- **Mode A — 3D onto video:** camera panels project scene entities onto the image
  (perception-accuracy checking). Projection math lives in `renderer/camera-projection`,
  consumed via the panel API — extensions get it for free.
- **Mode B — video into 3D:** frames as textures at the camera's world pose in the viewport.

Calibration flows through the frame system (`scene-graph.md#frames`), so extrinsics
interpolate consistently with ego pose.
