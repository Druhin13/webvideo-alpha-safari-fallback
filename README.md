# Transparent Video (Alpha) Workflow + Web Integration

A public, reproducible workflow for creating lightweight **transparent videos** that work across modern browsers. The pipeline uses a **WebM (VP9 + alpha)** source for Chromium-based browsers and a **ProRes 4444 MOV (alpha)** fallback for Safari. This repository documents the steps, provides sample commands, and shows how to integrate on the web (including no‑framework HTML embeds).

> This README is scrubbed of private details. Replace all placeholder paths and URLs with your own assets and hosting.


## Contents
- [Overview](#overview)
- [Outputs](#outputs)
- [Requirements](#requirements)
- [Workflow](#workflow)
  - [1) Create Master ProRes 4444 MOV (via Shutter Encoder)](#1-create-master-prores-4444-mov-via-shutter-encoder)
  - [2) Optimize MOV with FFmpeg](#2-optimize-mov-with-ffmpeg)
  - [3) Integrate on the Web](#3-integrate-on-the-web)
- [Browser Support Matrix](#browser-support-matrix)
- [Folder Structure](#folder-structure)
- [Notes & Tips](#notes--tips)
- [License](#license)


## Overview
Safari (especially iOS) does not support alpha transparency in WebM or HEVC the same way Chromium browsers do. To ensure consistent rendering, we deliver:
- **WebM (VP9 + alpha)** to Chrome, Edge, and Firefox.
- **ProRes 4444 MOV (alpha)** to Safari (macOS/iOS) via simple runtime detection and source swapping.

This approach preserves real transparency while keeping file sizes manageable for the web.


## Outputs
- `all-locales-placeholder.webm` — VP9 with alpha (for Chromium browsers)
- `all-locales-placeholder.mov` — ProRes 4444 with alpha (master)
- `all-locales-placeholder_prores4444_960w_24fps_q24.mov` — optimized desktop MOV
- `all-locales-placeholder_prores4444_480w_24fps_q28.mov` — optimized mobile MOV

> Filenames are examples. Use any naming strategy that suits your project.


## Requirements
- [Shutter Encoder](https://www.shutterencoder.com/) (GUI) for converting to ProRes 4444 with alpha
- [FFmpeg](https://ffmpeg.org/) (CLI) for resizing/compressing while preserving alpha
- A place to host your assets (use your own server/CDN; do not embed private service URLs)


## Workflow

### 1) Create Master ProRes 4444 MOV (via Shutter Encoder)
1. Open **Shutter Encoder**.
2. Add your **transparent WebM**: `all-locales-placeholder.webm`.
3. In “Choose function”, select **ProRes 4444**.
4. Ensure **Keep alpha channel** is enabled.
5. Click **Start function**.

**Output (master):**
- Container: `.mov`
- Codec: Apple ProRes 4444
- Pixel format: `yuva444p10le` (10-bit color + 8-bit alpha)


### 2) Optimize MOV with FFmpeg

Create device‑targeted sizes while preserving the alpha channel using `prores_ks` with `-profile:v 4` and `-pix_fmt yuva444p10le`.

**Desktop (960 px wide):**
```bash
ffmpeg -y -i "/absolute/or/relative/path/to/all-locales-placeholder.mov" -an -vf "scale=960:-2" -r 24 -c:v prores_ks -profile:v 4 -pix_fmt yuva444p10le -qscale:v 24 -movflags +faststart "./dist/all-locales-placeholder_prores4444_960w_24fps_q24.mov"
```

**Mobile (480 px wide):**
```bash
ffmpeg -y -i "/absolute/or/relative/path/to/all-locales-placeholder.mov" -an -vf "scale=480:-2" -r 24 -c:v prores_ks -profile:v 4 -pix_fmt yuva444p10le -qscale:v 28 -movflags +faststart "./dist/all-locales-placeholder_prores4444_480w_24fps_q28.mov"
```

**Verify alpha:**
```bash
ffprobe -hide_banner -show_streams -select_streams v:0 "./dist/all-locales-placeholder_prores4444_480w_24fps_q28.mov" | grep pix_fmt
# Expect: pix_fmt=yuva444p10le
```

**Quality guidance for `-qscale:v`:**
- 22 = higher quality (bigger)
- 24 = balanced
- 26 = lightweight
- 28 = aggressive (smallest acceptable for many UI animations)


### 3) Integrate on the Web

#### HTML + CSS (WebM by default)
Use your own hosting for the source URLs.

```html
<style>
.video-host video {
  width: 100%;
  height: 100%;
  object-fit: contain;
}
</style>

<div class="video-host">
  <video id="video-orbit" muted autoplay loop playsinline loading="lazy" poster="" style="pointer-events: none;">
    <source id="source-orbit" src="https://your.cdn.example/assets/video/all-locales-placeholder.webm" type="video/webm">
  </video>
</div>
```

#### Safari detection → swap to ProRes 4444 MOV
Place this after the video element (or just before </body>).

```html
<script>
  function isSafariWithAlphaIssue() {
    const ua = navigator.userAgent;
    const isSafari = /^((?!chrome|android).)*safari/i.test(ua);
    const isIOS = /iP(hone|od|ad)/i.test(ua);
    const isMac = /Macintosh/i.test(ua);
    const hasSafariProps = !!window.safari || typeof navigator.standalone !== "undefined";
    return isSafari && (isIOS || isMac) && hasSafariProps;
  }

  if (isSafariWithAlphaIssue()) {
    const video = document.getElementById("video-orbit");
    const source = document.getElementById("source-orbit");
    source.src = "https://your.cdn.example/assets/video/all-locales-placeholder_prores4444_480w_24fps_q28.mov";
    source.type = "video/quicktime";
    video.load();
  }
</script>
```

> Use the 480w MOV for mobile. For large desktop placements, you can point Safari to the 960w MOV instead:
> `all-locales-placeholder_prores4444_960w_24fps_q24.mov`


## Browser Support Matrix

| Browser | Alpha format | Delivery |
|---|---|---|
| Chrome / Edge / Firefox | WebM (VP9 + alpha) | Default `<source>` |
| Safari (macOS + iOS) | MOV (ProRes 4444 + alpha) | JS detection fallback |


## Folder Structure

```
repo-root/
├─ src/
│  └─ video/
│     ├─ all-locales-placeholder.webm
│     └─ all-locales-placeholder.mov         # master (ProRes 4444)
├─ dist/
│  ├─ all-locales-placeholder_prores4444_960w_24fps_q24.mov
│  └─ all-locales-placeholder_prores4444_480w_24fps_q28.mov
├─ README.md
└─ .gitignore
```

A minimal `.gitignore` can include:
```
.DS_Store
*.tmp
node_modules/
dist/
```

(Commit the `dist/` MOVs only if you intend to host them from the repo. Otherwise publish them to your storage/CDN and keep `dist/` out of version control.)


## Notes & Tips
- Keep the **ProRes 4444 master** archived to regenerate sizes/qualities without re-rendering from your NLE.
- Prefer **480w** MOV for mobile Safari and **960w** MOV for desktop Safari.
- If you need even smaller MOVs, increase `-qscale:v` (e.g., 26 or 28). Inspect edges/gradients for banding before shipping.
- Host assets from infrastructure you control. Avoid hard-coding third‑party CDN examples into your production code.


## License
This documentation is provided under the MIT License. The media files you process may be subject to separate copyright or licensing terms; ensure you have the rights to distribute any example assets.
