# Lossy Compressors — Image & Video
### `scripts/compressors/lossy/image-jpg.js` · `scripts/compressors/lossy/video-mp4.js`

My contribution to **MACS File Compressor**, a browser-native Chrome Extension that compresses files entirely offline. This module covers two lossy compressors:

- **`image-jpg.js`** — spatial redundancy reduction for JPEG images via `jpeg-js`
- **`video-mp4.js`** — temporal redundancy reduction for MP4 video via `ffmpeg.wasm`

Both modules expose a consistent `compress() / decompress()` API and return PSNR quality metrics consumed by the shared `metrics.js` utility.

---

## Table of Contents

- [How It Works](#how-it-works)
  - [Image: Spatial Redundancy](#image-spatial-redundancy)
  - [Video: Temporal Redundancy](#video-temporal-redundancy)
- [File Structure](#file-structure)
- [Dependencies](#dependencies)
- [Setup — Manual Steps Required](#setup--manual-steps-required)
- [API Reference](#api-reference)
  - [ImageJpgCompressor](#imagejpgcompressor)
  - [VideoMp4Compressor](#videomp4compressor)
- [Quality Metric: PSNR](#quality-metric-psnr)
- [Integration with the Rest of the Extension](#integration-with-the-rest-of-the-extension)
- [Known Limitations](#known-limitations)

---

## How It Works

### Image: Spatial Redundancy

> **Spatial redundancy** — neighboring pixels in a natural image are highly correlated. A blue sky has thousands of pixels that differ by only 1–2 values. JPEG exploits this to throw away data your eye will never notice.

`image-jpg.js` drives the following pipeline end-to-end inside the browser:

```
Input File (.jpg / .png)
       │
       ▼
  Decode to raw RGBA pixels
  (jpeg-js for JPEG · Canvas API fallback for PNG)
       │
       ▼
  RGB → YCbCr color-space conversion          ← chroma separated from luma
  Chroma subsampling 4:2:0                    ← halves U/V resolution
  8×8 DCT per block                           ← spatial → frequency domain
  Quantization (quality-scaled matrix)        ◄── LOSSY STEP
  Zigzag scan + RLE + Huffman coding          ← lossless entropy coding
       │
       ▼
  JPEG-encoded bytes (output)
       │
       ▼
  Re-decode output → compare pixel buffers → PSNR
```

All of this happens inside `jpeg-js` with a single `encode(frameData, quality)` call. The quality parameter (1–100) controls the quantization matrix — lower quality = coarser coefficients = smaller file = more visible artifacts.

---

### Video: Temporal Redundancy

> **Temporal redundancy** — consecutive video frames share large regions of unchanged pixels. A talking-head video might have 95% of each frame identical to the previous one. H.264 encodes only what *changed*.

`video-mp4.js` drives FFmpeg (compiled to WebAssembly) with the following frame types:

| Frame Type | Description |
|---|---|
| **I-frame** (Intra) | Full spatial compression of a single frame — like a JPEG. No reference to other frames. Inserted at each keyframe interval (GOP). |
| **P-frame** (Predicted) | Only the *difference* from the previous reference frame. Motion vectors point to 16×16 macroblocks; only the residual error is encoded. |
| **B-frame** (Bi-directional) | References both a past and a future frame. Highest compression but requires frame reordering in the bitstream. |

**CRF (Constant Rate Factor)** controls the quality/size trade-off for H.264:

| CRF | Quality |
|---|---|
| 0 | Lossless |
| 18 | Visually lossless |
| 23 | Default — good quality |
| 28 | Acceptable, noticeably smaller |
| 51 | Worst quality |

PSNR is measured by running ffmpeg's built-in `psnr` lavfi filter in a second pass, comparing the compressed output frame-by-frame against the original decoded frames.

---

## File Structure

These are the only files in scope for this module:

```
macs-file-compressor/
├── scripts/
│   └── compressors/
│       └── lossy/
│           ├── image-jpg.js      ← spatial redundancy (this module)
│           └── video-mp4.js      ← temporal redundancy (this module)
└── lib/
    ├── jpeg-js.min.js            ← must be added manually (see Setup)
    └── ffmpeg.min.js             ← must be added manually (see Setup)
```

---

## Dependencies

| Library | Used by | Purpose |
|---|---|---|
| [`jpeg-js`](https://www.npmjs.com/package/jpeg-js) | `image-jpg.js` | Pure-JS JPEG encoder + decoder |
| [`ffmpeg.wasm`](https://github.com/ffmpegwasm/ffmpeg.wasm) | `video-mp4.js` | Full FFmpeg compiled to WebAssembly |

Both are loaded as global browser scripts via `lib/`. Neither requires Node.js or a server.

---

## Setup — Manual Steps Required

### 1. Download library files into `lib/`

**jpeg-js:**
```
https://cdn.jsdelivr.net/npm/jpeg-js/lib/index.js
→ save as lib/jpeg-js.min.js
```

**ffmpeg.wasm** — download all three files from the [ffmpegwasm releases page](https://github.com/ffmpegwasm/ffmpeg.wasm/releases) and place them in `lib/`:
```
ffmpeg.min.js
ffmpeg-core.js
ffmpeg-core.wasm
ffmpeg-core.worker.js
```

> ffmpeg.wasm will not work with only one file — all three core files must be co-located.

---

### 2. Add COOP/COEP headers to `manifest.json`

`ffmpeg.wasm` requires `SharedArrayBuffer`, which Chrome blocks unless these headers are set. Add to your manifest:

```json
"content_security_policy": {
  "extension_pages": "script-src 'self' 'wasm-unsafe-eval'; object-src 'self'"
}
```

And handle the response headers in `background/service-worker.js` via `declarativeNetRequest` or `webRequest`:
```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

---

### 3. Load scripts in `popup.html` in this order

```html
<!-- Dependencies first -->
<script src="lib/jpeg-js.min.js"></script>
<script src="lib/ffmpeg.min.js"></script>

<!-- Then this module -->
<script src="scripts/compressors/lossy/image-jpg.js"></script>
<script src="scripts/compressors/lossy/video-mp4.js"></script>
```

---

### 4. Preload ffmpeg on popup open

In `scripts/ui/dom-handler.js`, call this as early as possible:

```js
VideoMp4Compressor.preload();
```

This warms up the ~25 MB WASM binary in the background so the first compression call doesn't appear frozen.

---

## API Reference

### `ImageJpgCompressor`

Exposed as `window.ImageJpgCompressor`.

#### `compress(file, options)`

```js
const result = await ImageJpgCompressor.compress(file, { quality: 75 });
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | `File` | — | Input image file (`.jpg` or `.png`) |
| `options.quality` | `number` | `75` | JPEG quality 1–100. Lower = smaller file, more loss. |

**Returns:**

```js
{
  compressedBlob   : Blob,     // JPEG output ready for download
  originalSize     : number,   // bytes
  compressedSize   : number,   // bytes
  compressionRatio : number,   // originalSize / compressedSize
  spaceSavings     : number,   // percent saved, e.g. 42.5
  psnr             : number,   // dB — see PSNR section below
  width            : number,   // pixels
  height           : number,   // pixels
  quality          : number,   // quality value actually used
}
```

---

#### `decompress(file)`

```js
const result = await ImageJpgCompressor.decompress(file);
```

Decodes the JPEG to raw pixels and re-encodes at quality 100 as the best possible reconstruction. JPEG is inherently lossy — perfect pixel recovery is impossible; PSNR reflects accumulated quantization loss.

**Returns:**

```js
{
  decompressedBlob  : Blob,
  compressedSize    : number,
  decompressedSize  : number,
  psnr              : number | null,
  width             : number,
  height            : number,
}
```

---

### `VideoMp4Compressor`

Exposed as `window.VideoMp4Compressor`.

#### `compress(file, options)`

```js
const result = await VideoMp4Compressor.compress(file, {
  crf          : 23,
  preset       : 'medium',
  audioBitrate : 128,
  onProgress   : ({ ratio }) => console.log(ratio),
});
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `file` | `File` | — | Input video file (`.mp4`) |
| `options.crf` | `number` | `23` | H.264 CRF (0–51). Lower = better quality, larger file. |
| `options.preset` | `string` | `'medium'` | x264 preset: `ultrafast` → `veryslow`. Slower = better compression at same CRF. |
| `options.audioBitrate` | `number` | `128` | AAC audio bitrate in kbps. |
| `options.onProgress` | `function` | `null` | Progress callback `{ ratio: 0–1 }` during WASM load. |

**Returns:**

```js
{
  compressedBlob   : Blob,     // MP4 output ready for download
  originalSize     : number,
  compressedSize   : number,
  compressionRatio : number,
  spaceSavings     : number,   // percent
  psnr             : number | null,
  codec            : string,   // "H.264 (libx264)"
  crf              : number,
  preset           : string,
}
```

---

#### `decompress(file, options)`

```js
const result = await VideoMp4Compressor.decompress(file, { onProgress });
```

Re-encodes the video at CRF 0 (lossless H.264) as the highest-fidelity reconstruction of already-decoded frames. Reports PSNR of accumulated loss.

**Returns:**

```js
{
  decompressedBlob  : Blob,
  compressedSize    : number,
  decompressedSize  : number,
  psnr              : number | null,
  codec             : string,   // "H.264 CRF-0 (lossless decode)"
}
```

---

#### `preload(onProgress?)`

```js
await VideoMp4Compressor.preload();
```

Pre-initialises the ffmpeg.wasm instance. Call this once on popup load to avoid a cold-start delay on first compression.

---

## Quality Metric: PSNR

Both modules report **Peak Signal-to-Noise Ratio (PSNR)** in decibels (dB) as the quality metric.

```
MSE  = Σ (original_pixel - compressed_pixel)² / (3 × N_pixels)   [RGB channels]
PSNR = 10 × log₁₀(255² / MSE)
```

| PSNR | Perceptual Quality |
|---|---|
| > 40 dB | Excellent — visually lossless |
| 30–40 dB | Good — minor artifacts under close inspection |
| 20–30 dB | Noticeable degradation |
| < 20 dB | Significant distortion |
| ∞ | Images are pixel-identical |

For **images**, PSNR is computed in JavaScript by comparing raw RGBA pixel buffers before and after encode/decode.

For **video**, PSNR is computed by ffmpeg's `psnr` lavfi filter in a dedicated second pass, comparing the compressed output frame-by-frame against the original. Per-frame stats are written to `psnr_stats.log` in the WASM virtual filesystem and parsed from there.

---

## Integration with the Rest of the Extension

Both modules return the same consistent shape so `metrics.js` can consume them uniformly:

```js
// In dom-handler.js — image example
const result = await ImageJpgCompressor.compress(file, { quality: 75 });
MetricsUtil.display({
  originalSize     : result.originalSize,
  compressedSize   : result.compressedSize,
  compressionRatio : result.compressionRatio,
  spaceSavings     : result.spaceSavings,
  psnr             : result.psnr,
});

// Video follows the exact same pattern
const result = await VideoMp4Compressor.compress(file, { crf: 23 });
MetricsUtil.display({ ...result });
```

Both globals (`window.ImageJpgCompressor`, `window.VideoMp4Compressor`) are available as soon as their respective `<script>` tags have loaded in `popup.html`.

---

## Known Limitations

- **JPEG is inherently lossy** — `decompress()` cannot recover the exact original pixels. PSNR reflects this unavoidable loss.
- **ffmpeg.wasm is large** — the WASM binary is ~25 MB. `preload()` mitigates the UX impact but the initial load on a cold popup will take a few seconds.
- **SharedArrayBuffer requirement** — `ffmpeg.wasm` will silently fail or throw in Chrome if COOP/COEP headers are not set correctly. See [Setup](#setup--manual-steps-required).
- **Video PSNR may return `null`** — if the `psnr` lavfi filter is unavailable in the ffmpeg.wasm build variant you use, PSNR measurement is skipped gracefully and `null` is returned.
- **PNG input to `image-jpg.js`** — PNG files are decoded via the Canvas API (not jpeg-js, which is JPEG-only). The output is always JPEG regardless of input format.