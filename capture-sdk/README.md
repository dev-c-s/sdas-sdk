# SDAS-SDK — Smart Document And Selfie SDK

Embeddable browser SDK for capturing a document and/or a selfie, with live
quality feedback (glare, perspective, framing), AI corner detection and an
optional manual shutter. It drops into any page with one `<script>` tag and a
single `SDAS.open()` call.

## Quickstart

```html
<script src="./capture-sdk/sdas-sdk.js"></script>
<script>
  const result = await SDAS.open({
    mode: "doc",              // "doc" | "selfie" | "doc+selfie"; omit for a selection pane
    lang: "en",               // "en" / "de"; unknown codes fall back to en
    report: false,            // no Report button unless you opt in
  });
  // result.doc.image is a JPEG Blob
</script>
```

## Options

| Key | Type | Default | Meaning |
| --- | --- | --- | --- |
| `mode` | `"doc" \| "selfie" \| "doc+selfie"` | omitted → selection pane | Capture flow. `doc+selfie` runs the document first, then the selfie automatically. |
| `lang` | `string` | `"en"` | UI language. Unknown code falls back to `en`. |
| `strings` | `Record<string, string>` | `{}` | Per-key string overrides, highest precedence over `lang`. |
| `fullscreen` | `boolean` | `true` | Fixed full-viewport overlay. `false` requires a `container`. |
| `container` | `HTMLElement \| string` | none | Mount target (element or selector) when embedding in a page region. Wins over `fullscreen` if both are given. The host fills 100% of the container but enforces a `min-height` of 480px — a container with auto height would otherwise render a zero-height capture UI. |
| `manual` | `number \| boolean` | `false` | `N` = seconds after the camera goes live until the shutter appears; `true` = shutter from the start. |
| `aiCapture` | `boolean` | `true` | `false` disables auto detection — manual shutter from the start. |
| `qualityChecks` | `boolean` | `true` | `false` skips the QC gates (a stable quad is still required for auto capture). |
| `maxEdgePx` | `number` | `4096` | Long-edge clamp at accept, aspect preserved (JPEG 0.9). The camera track requests 2560 max edge on Chromium browsers, 1920 on Firefox; iOS document streams use square-min constraints (≥1920 both axes — WebKit workaround, delivers the camera's best 4:3 mode) and selfie streams target a 1920×1920 ideal (incapable cameras silently deliver their native max — the negotiation is the capability probe; `?cap=N` overrides). The capture fill floor adapts per device: it targets ~1650px of true pixels on the axis the card aligns with, clamped to 62-85% fill and capped by a perpendicular cutOff budget, a 76% focus-zone cap and a 65% comfort cap (never pushes the user into the camera's min-focus distance). No re-encode under the limit. |
| `cameraSelection` | boolean | `true` | Show the in-capture camera dropdown so the user can switch cameras. Set `false` for a locked-down UI (the automatic camera choice is unaffected). |
| `report` | `boolean` | `false` | Shows the Report button in the capture UI. |

`SDAS.open()` resolves when the user accepts. If the capture engine fails to
initialize (or hangs), the promise REJECTS with an error carrying
`code: "model_failed"` and the overlay is torn down — handle the rejection.
There is no user-facing cancel control (no user-facing abort): to replace an
active capture, call `SDAS.open()` again — the active capture is destroyed and
its promise rejects with `{ cancelled: true }`, then the new capture opens
(last-open-wins). A newer call that starts while an earlier one is still
initializing supersedes it the same way.

`SDAS.version` is the build's semver string (e.g. `"0.5.3"`).

## What you get back

```ts
type CaptureOutput = {
  image: Blob;      // JPEG
  width: number;
  height: number;
  ok: boolean;      // QC verdict for the captured frame (auto OR manual)
  manual: boolean;
  flags: object;    // QC flags + diagnostics (e.g. perspDeg); {} if none ran
};

type CaptureResults = {
  mode: string;              // the mode that ran
  doc?: CaptureOutput;
  selfie?: CaptureOutput;
};
```

`image` is a `Blob` in memory — create a `URL.createObjectURL(result.doc.image)`
to preview it, or upload/`FormData` it directly. Nothing is uploaded by the SDK
itself. Selfie captures are mirrored to match the live framing preview (WYSIWYG
— what the user framed is what they get).

## i18n

`strings.json` ships beside `sdas-sdk.js` and is fetched relative to the script
URL. It holds one object per language (`{ "en": { ... }, "de": { ... } }`).
Lookup order: requested language section → `en` fallback → the raw key, then
per-call `strings` overrides merge last.

To add a language, add a section to `strings.json` with the same keys as `en`
(translate the values) and ship the file next to the SDK.

## Manual capture semantics

- `manual: N` — the N-second timer starts when the camera stream is live; on
  expiry a round shutter appears bottom-center. Pressing it captures the current
  frame, bypassing the capture gates. The result carries `manual: true`, and
  `ok`/`flags` report the QC verdict of the most recently analyzed frame (≤~200ms
  before the grab; no fresh analysis is run at capture time). If no analysis has
  run yet, or quality checks are off, they are `ok: false` + `flags: {}`.
- `manual: true` or `aiCapture: false` — the shutter is visible from the start.

## Camera permissions & stream reuse

`SDAS.open()` acquires the camera. Calling `open()` again to switch modes
(doc → selfie → doc) keeps the stream alive and reuses it for the **same camera
type**, so desktop browsers do not re-prompt and repeated same-type switches never
call `getUserMedia` again. Switching between front and rear cameras necessarily
re-acquires the device, and iOS (Safari and Chrome) may show one additional
permission prompt per front/rear switch — that is platform behavior, not the SDK
re-prompting on every mode switch.

## Files that must stay together

Ship this folder structure, and keep the folder name stable relative to your
page:

```
sdas-sdk.js
sdas-models.pak
strings.json
wasm/engine/engine-core.js
wasm/engine/engine-core.wasm
wasm/engine/engine-compat.js
wasm/engine/engine-compat.wasm
wasm/engine/j1.js
wasm/engine/j1.wasm
```

`sdas-sdk.js` is the library (global `SDAS`), `sdas-models.pak` holds the models,
the two worker chunks and the watermark (~1.6MB), and `strings.json` holds
the UI text. The SDK resolves all of these relative to the script URL.

`wasm/engine/` holds the three LiteRT engine builds (core = relaxed SIMD,
compat = fixed-width SIMD, `j1` = JSPI/WebGPU bridge). **Only the one variant
the device selects at runtime is downloaded** — typically a single `.js` + its
`.wasm` (the others are never fetched). Serve them precompressed: brotli or gzip
cuts the ~9.6 MB wasm to roughly 2.5-3.5 MB (wasm compresses ~3-4x). If you use
a CDN or a static host without automatic compression, precompress these files.

## Requirements

- **HTTPS is mandatory** — browsers only grant camera access (`getUserMedia`) on
  secure origins (`https://` or `localhost`).
- Modern **Chrome, Safari, Firefox and Edge** (WebGPU is optional and used when
  available; wasm is the fallback).
- **No server headers are required** — no COOP/COEP, no MIME configuration
  beyond serving `.js` as JavaScript. `sdas-models.pak` is fetched as a binary
  blob and benefits from gzip/brotli like any static asset.

## Watermark

A small SDAS-SDK wordmark is displayed at the bottom of the camera area
(`opacity: 0.5`, non-interactive). It is bundled inside the pak as
`/watermark.png`; the shipped watermark is the SDAS brand mark.
Integrator-supplied watermarks are not part of this release. The shipped
artifacts are integrity-protected — use them as distributed, without
modification.

## Privacy

The SDK is offline-capable and does not upload captures: `report` defaults to
`false`. When `report: true` is set, a Report button appears in the capture UI;
pressing it POSTs the current camera frame plus device/diagnostic metadata to
the vendor's `/idscan/report` endpoint for support/QA. Leave `report` disabled
unless you and your users intend that telemetry.
## Attribution

This distribution includes compiled runtime components derived from LiteRT
(formerly TensorFlow Lite), Apache License 2.0 — © the LiteRT Authors. The
license text is available at https://github.com/google-ai-edge/LiteRT.
