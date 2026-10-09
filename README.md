# Transparent Video Keying — make a plain MP4 float on your desktop

Turn an ordinary **MP4 with no alpha channel** into a background-transparent virtual avatar
for an Electron / browser desktop pet or floating assistant — by keying out the background
per frame on a `<canvas>`.

No After Effects, no ProRes, no alpha-channel export headaches.

> 🇨🇳 中文完整指南见 [`SKILL.md`](./SKILL.md)（面向 AI Agent 的技能文件，同样适合人读）。

---

## Try it in 10 seconds

Open [`references/keying-demo.html`](./references/keying-demo.html) in any Chromium-based browser —
single file, zero dependencies, works offline:

1. Click **选择视频** and pick a black-background or green-screen MP4 from your machine.
2. Pick the matching mode (green screen / black background).
3. Drag the threshold sliders. The checkerboard behind the canvas shows exactly what is truly transparent.

The demo runs the **same algorithms** as production code, and exposes them for unit testing:

```js
window.__dbg.keyTest([30,28,25,255], "chroma");  // → alpha 255 (dark hair kept)
window.__dbg.keyTest([30,28,25,255], "luma");    // → alpha 150 (dark hair partly eaten)
```

---

## The core idea

```
offscreen <video>  →  drawImage to canvas  →  getImageData  →  rewrite alpha  →  putImageData
```

The window itself is transparent (`transparent: true` in Electron), so pixels you set to
`alpha = 0` simply disappear. You get a real person standing on your desktop.

**You do not need an alpha channel.** Keeping a plain black or green background and keying at
runtime is faster to iterate (thresholds are live-tunable) and avoids a whole export step.

### Why not alpha video?

| Format | Has alpha | Plays in Chromium / Electron |
|---|---|---|
| MP4 (H.264) | ✗ | ✓ |
| MP4 (HEVC with alpha) | ✓ | ✗ |
| MOV (ProRes 4444) | ✓ | ✗ |
| **WebM (VP9 + alpha)** | ✓ | ✓ **the only usable one** |

And note: **CapCut / Premiere / Media Encoder cannot export alpha video at all.** That is the
single most common misconception in this space.

---

## Two keying modes — different units, do NOT merge their thresholds

| | **Black background (luma)** | **Green screen (chroma)** |
|---|---|---|
| Criterion | brightness `lum = max(R,G,B)` | greenness `G − max(R,B)` (clamped at 0) |
| Alpha direction | brighter ⇒ more opaque | greener ⇒ more transparent |
| Thresholds | `keyLow` = 10, `keyHigh` = 44 | `ckLow` = 8, `ckHigh` = 42 |
| Edge fix | **unmultiply** (edge was darkened by the black bg) | **despill** (edge was tinted green), default 0.7 |
| Dark hair / dark clothing | can be keyed *through* | **robust** — only green-dominant pixels are removed |

Two things that trip everyone up:

1. **The "make it more aggressive" direction is inverted between modes.** For black background,
   *raise* the transparent threshold; for green screen, *lower* it.
2. **Don't use the theoretical threshold for "standard green".** Measured: `#00B140` — the
   canonical green screen colour — comes back as `(53,140,84)` after an H.264 / yuv420p round trip.
   Its greenness drops to **55–58** (not 176+). Using the theoretical value leaves you with a
   semi-transparent green haze (measured: 74% of pixels at alpha ≈ 85).

---

## Pitfalls worth knowing before you start

| # | Pitfall | Symptom | Fix |
|---|---|---|---|
| 1 | Alpha is **straight**, not premultiplied | Soft edges look right on black but **glow white** on light UI (`255×(1−a)`; measured avg +133.9 levels) | Mathematical, not tunable — **keep UI from overlapping the person** |
| 2 | `willReadFrequently: true` missing | Per-frame `getImageData` round-trips through the GPU → much slower | Always set it on the 2D context |
| 3 | Canvas tainted by cross-origin video | `getImageData` throws `SecurityError`, keying **silently fails** | Same-origin / local files; assert readability in your self-test |
| 4 | Pixels with `alpha = 0` read back as `(0,0,0)` | Any "find dark pixels" statistic counts the whole transparent background as dark (measured 88839 phantom pixels) | `if (d[i+3] < 8) continue;` |
| 5 | Measuring hair strands by frame statistics | Canvas downscaling averages away 1px-wide detail (dark sample: theoretical 1287 → **1 pixel**) | Test **at the predicate level** — feed RGBA into the keying function and assert on output |
| 6 | Default mode doesn't match the shipped asset | Lost config ⇒ falls back to `luma` while the asset is green screen ⇒ **one solid green rectangle** | Keep the fallback in sync with your default asset (and in **both** places) |
| 7 | Material replaced in place, picture unchanged | Chromium caches `file://`; same URL string returns the old file | Bust the cache with a timestamp query |
| 8 | Rectangular hit-test blocks the app underneath | The transparent area beside the person is still "window" — a half-screen invisible box blocks your slides | Hit-test **per pixel** from canvas alpha: `getImageData(px,py,1,1).data[3] > 24` |
| 9 | **"Pure black" in video is Y ≈ 16, not 0** | With `keyLow = 10` the background pixel `(16,16,16)` resolves to **alpha ≈ 45** — a faint grey veil that reads as "not cleanly keyed" | Set `keyLow` **2–4 levels above the background's measured brightness** (~20–24). Green screen is immune: `R = G = B` ⇒ greenness is always 0 |

Full list with measured numbers: [`SKILL.md` § 8](./SKILL.md).

---

## Material specs

| Item | Requirement | Why |
|---|---|---|
| Codec | H.264 **Baseline / Main** | High profile renders **solid black on some Android WebViews, with no error** |
| Resolution | 720×1280 or 1080×1920, 24–30 fps | Portrait framing; keying is throttled to 30fps anyway |
| Background | Pure black (Y ≈ 16–19) or pure green | Dark grey gets keyed as semi-transparent → grey haze |
| Audio track | Remove it | Players are `muted`; it only wastes bytes |
| Subject size | ≥ 85% of frame height | Otherwise the subject looks tiny once keyed |
| Loop | Ping-pong (forward + reversed) or a first/last-frame generator | Measured seam PSNR 22 dB → **38 dB** after ping-pong |
| Idle vs. talking | **Identical camera & subject size** | Mismatch makes the character *pop* in size when switching — more jarring than a loop seam |

ffmpeg recipes (de-audio, compress, ping-pong, crop, alpha WebM, plus the `-auto-alt-ref 0`
gotcha): [`references/素材与编码速查.md`](./references/素材与编码速查.md).

---

## Repo layout

```
transparent-video-keying/
├── SKILL.md                          # full guide (Chinese) — algorithm, tuning, pitfalls, verification
├── README.md                         # this file
├── LICENSE                           # MIT
└── references/
    ├── keying-demo.html              # single-file, zero-dep, open-and-try demo
    └── 素材与编码速查.md              # ffmpeg / encoding cheat sheet
```

---

## Verification, not vibes

Don't eyeball it. Every failure mode here (tainted canvas, inverted threshold units, dark areas
keyed through) looks like "just needs tuning".

Export your keying functions behind a debug hook and assert on pixel values — testing the real
functions, not a re-implementation. `transparentRatio` (transparent-pixel share) should land
around **0.65–0.8**; near 0 means keying did nothing, near 1 means the subject got erased.

---

## License

MIT — use it, ship it, no attribution required.
