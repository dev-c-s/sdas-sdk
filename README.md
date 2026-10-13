<div align="center">

<img src="demo-app/logo.png" alt="SDAS-SDK — Smart Document And Selfie SDK" width="240">

### Embeddable, fully on-device document + selfie capture for the web

One `<script>` tag and one `SDAS.open()` call. A neural network finds the
document corners, live quality gates coach the user, and you receive JPEG
blobs plus QC verdicts — all computed client-side, nothing uploaded.

[Live demo](https://id.cansahin.net) &nbsp;·&nbsp; [Quickstart](#integration-in-60-seconds) &nbsp;·&nbsp; [Quality gates](#document-capture--live-quality-gates) &nbsp;·&nbsp; [License](#license)

![version](https://img.shields.io/badge/version-0.5.5-2eff7b?style=flat-square&labelColor=101922)
![license](https://img.shields.io/badge/license-PolyForm%20NC%201.0.0-2a3a46?style=flat-square&labelColor=101922)
![inference](https://img.shields.io/badge/inference-100%25_on--device-2eff7b?style=flat-square&labelColor=101922)
![browsers](https://img.shields.io/badge/browsers-Chrome_·_Edge_·_Firefox_·_Safari-2a3a46?style=flat-square&labelColor=101922)

<p>
  <img src="docs/screenshots/doc-capture.jpg" alt="Document capture with live corner detection and quality overlay" width="45%">
  &nbsp;
  <img src="docs/screenshots/selfie-capture.jpg" alt="Selfie capture with head-mask guide and keypoint overlay" width="45%">
</p>
<p>
  <img src="docs/screenshots/results-view.jpg" alt="Demo results view with QC verdict chips" width="60%">
</p>

</div>

> **Try it now:** <https://id.cansahin.net> — log in with `demo` / `demo` and
> point your camera at any ID card, or switch to Selfie mode.

SDAS-SDK is an embeddable browser library for identity capture. Point a camera
at an ID document and/or take a selfie: a small on-device neural network finds
the document corners, live quality gates (glare, blur, tilt, perspective,
distance, lighting, stability) coach the user in real time and gate the
auto-capture, and the result — JPEG blobs plus a QC verdict and per-capture
diagnostics — is handed back to your code.

Both capture flows (document **and** selfie) ship inside this one library; the
models ride in `sdas-models.pak` and load on demand. The repo is source-free
by design — these are the shipped, integrity-protected artifacts.

---

## What's in this repo

| Path | What it is |
| --- | --- |
| `capture-sdk/` | The **prebuilt SDK**, ready to drop into any web app: `sdas-sdk.js` (the library, global `SDAS`), `sdas-models.pak` (both neural models, the two worker chunks and the watermark, ~1.6 MB), `wasm/engine/` (the three inference runtime variants), `strings.json` (en/de UI text) and the full integrator manual (`capture-sdk/README.md`). |
| `demo-app/` | The reference integration — the same page that runs at the live demo above. |

## Run the demo locally

```bash
git clone https://github.com/dev-c-s/sdas-sdk.git
cd sdas-sdk
python3 -m http.server 8000
# open http://localhost:8000/demo-app/
```

Browsers only grant camera access on secure origins (`https://` or
`localhost`), so the page must be served — opening `index.html` from `file://`
will not work.

---

## Integration in 60 seconds

```html
<script src="./capture-sdk/sdas-sdk.js"></script>
<script>
  const result = await SDAS.open({
    mode: "doc",              // "doc" | "selfie" | "doc+selfie"; omit for a selection pane
    lang: "en",               // "en" / "de"; unknown codes fall back to en
  });
  // result.doc.image is a JPEG Blob — preview it or upload it yourself
</script>
```

That's the whole capture flow: SDAS renders its own full-screen (or embedded)
UI inside a shadow root, acquires the camera, runs the live quality loop, and
resolves the promise when the user accepts a capture.

### What you get back

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

Which keys are populated depends on `mode`:

| `mode` | The promise resolves with |
| --- | --- |
| `"doc"` | `{ mode: "doc", doc: CaptureOutput }` |
| `"selfie"` | `{ mode: "selfie", selfie: CaptureOutput }` |
| `"doc+selfie"` | `{ mode: "doc+selfie", doc: CaptureOutput, selfie: CaptureOutput }` — **both captures in one object** |
| omitted | Landing mode-selection pane; whichever flow the user picks populates its own key |

**How `doc+selfie` runs.** The document leg runs first (rear camera). After
the user reviews and accepts the document capture, the UI switches to the
front camera and the selfie leg starts automatically — no second
`SDAS.open()` call. The promise resolves **once**, after the selfie leg, with
**both** `doc` and `selfie` populated:

```js
const res = await SDAS.open({ mode: "doc+selfie" });
await upload("https://your.api/doc",    res.doc.image);     // rear camera
await upload("https://your.api/selfie", res.selfie.image);  // front camera, mirrored
```

- Each leg carries its **own independent QC verdict** — `ok`, `manual` and
  `flags` describe that capture only. `res.doc.ok` tells you nothing about
  `res.selfie.ok`.
- Selfie captures are additionally clamped to a **1920 px long edge**, even
  when `maxEdgePx` is set higher.
- Each leg has its own retry loop (`Retry` re-captures the current leg);
  the flow only advances on accept.
- There is no user-facing cancel: a new `SDAS.open()` destroys the active
  flow, and its promise rejects with `{ cancelled: true }`.

`image` is a `Blob` in memory — create `URL.createObjectURL(result.doc.image)`
to preview it, or upload/`FormData` it directly. Nothing is uploaded by the
SDK itself. Selfie captures are mirrored to match the live framing preview
(WYSIWYG — what the user framed is what they get).

If the capture engine fails to initialize (or hangs), the promise **rejects**
with an error carrying `code: "model_failed"` and the overlay is torn down —
handle the rejection.

### Options

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
| `maxEdgePx` | `number` | `4096` | Long-edge clamp at accept, aspect preserved (JPEG 0.9). No re-encode under the limit. |
| `cameraSelection` | `boolean` | `true` | Show the in-capture camera dropdown so the user can switch cameras. Set `false` for a locked-down UI (the automatic camera choice is unaffected). |
| `report` | `boolean` | `false` | Shows the Report button in the capture UI. Leave off unless you intend the telemetry (see [Privacy](#privacy)). |

> `capture-sdk/README.md` is the complete integrator manual — manual-capture
> semantics, camera-permission behavior, i18n authoring, file layout and
> serving advice.

---

## Advanced features

### Document capture — live quality gates

Auto-capture fires only when **every** gate holds. While a gate fails, the
user sees a single, prioritized on-screen instruction; failing
*environmental* checks (blur, glare, too close/far, too dark/bright) also
appear as compact badges in the viewfinder, so co-failures stay visible
instead of hiding behind the first-true instruction. All gates use
enter/exit **hysteresis** so prompts never flicker at the threshold.

| Gate | What it verifies | Live coaching (verbatim UI text) |
| --- | --- | --- |
| Distance / minimum size | The document must fill an **adaptively derived** share of the frame: the capture floor targets ~1650 true camera pixels on the axis the card's long side aligns with, clamped between 62% and 85% fill, with a focus-zone cap (never pushes into the camera's minimum-focus distance), a comfort cap (never demands more than a natural handheld hold delivers) and a perpendicular budget (never leaves the card's short edge cut off). Proximity is measured orientation-aware — long-edge fill against the frame dimension the card aligns with, not raw frame area (raw area false-fires on portrait phones). | `MOVE CLOSER` / `MOVE BACK` |
| Too close (defocus) | Browser cameras cannot focus at very close range (no manual-focus API), so an oversized, unsharp card is rejected rather than captured — judged from long-edge proximity plus sharp-edge width and micro-texture on the rectified card, with hysteresis. A **second, one-shot defocus check at higher resolution** runs at capture-confirm time, catching close-range blur the downscaled live loop physically cannot see. | `TOO CLOSE — MOVE FARTHER` |
| Blur | Sharp-edge width and micro-texture analysis on the rectified card; sparse-text documents (thin rules, large blank bands) are calibrated separately from dense text. | `BLURRY — HOLD STILL` |
| Glare | The rectified card is analyzed in three tiers: saturated white **clip cores**, near-neutral bright **sheen fringe**, and dense fully-clipped **hotspot** blobs (flagged far below the sheen limits, since a small dense white blob over the photo area is already damaging). Pale overexposed card bases do not count as glare. | `TILT AWAY FROM GLARE` |
| Perspective / skew | Corner-angle deviation from 90° must stay within a hysteresis band tuned to natural in-hand capture (8–16° measured live; genuinely skewed locks ~27° are rejected). A second **opposite-edge foreshortening gate** catches single-axis pitch/yaw the corner-angle metric is near-blind to. | `FACE CARD HEAD-ON` |
| Tilt (roll) | Card held level within 22°. | `HOLD IT LEVEL` |
| Centering | Document position within the frame. | `CENTER THE CARD` |
| Cut-off | No card corners clipped by the frame edge. | `MOVE CARD INTO VIEW` |
| Aspect plausibility | Landscape cards must measure plausibly landscape, portrait plausibly portrait — so the corner detector locking onto a wrong surface fails instead of capturing. | (verdict gate) |
| Lighting & plausibility | Dark / too-bright mean-luma gates, plus saturation/luma plausibility on gray-world-normalized channels (illumination-invariant) so dim or washed-out scenes never capture. | `TOO DARK — ADD LIGHT` / `TOO BRIGHT` |
| Document presence | Model presence/peak plausibility — something card-like must actually be detected before any of the above are even evaluated. | `NO CARD DETECTED` |

**Corner stability.** A dedicated anti-flicker stack sits under the gates:

- per-corner **One-Euro filtering** (speed-adaptive smoothing — kills jitter
  at rest, keeps up in motion);
- **plateau tie-hold**: when the detection heatmap has two nearly tied peaks
  far apart, the corner position would yank between frames — the tracker
  coasts on the last good reference instead of trusting the ambiguous frame;
- **k-confirm re-acquisition** after any detection gap (consecutive
  consistent quads required before re-locking);
- **glitch bridging**: momentary quad-stage rejects don't restart the hold.

**Capture hold.** Instead of a numeric countdown, the corner wireframe fills
bright green bottom-up while all gates hold (`HOLD STILL — CAPTURING…`), with
graded drain-back on failure and a brief flap-freeze so a single tremor
doesn't zero the progress — an interruptible hold consistent with commercial
scan SDKs. A moved document is caught at review (`DOCUMENT MOVED — HOLD
STEADY AND TRY AGAIN`).

**Thermal pacing.** On devices whose sustained inference time sags (phone SoC
throttling), frames are paced so clocks recover between analysis rounds — the
measured *sustained* analysis rate ends up higher than ungated.

### Selfie capture

Face detection with 6-point landmarks runs on the same on-device runtime as
the document model — no second engine, no extra download.

| Gate | What it verifies | Live coaching (verbatim UI text) |
| --- | --- | --- |
| Face presence + centering | A face must be detected and centered; an on-screen **head-mask guide** frames the target, sized per viewfinder aspect so it reads correctly on phones and webcams alike. | `CENTER YOUR FACE` / `NO FACE DETECTED` |
| Distance band | Face height must land between 0.63 and 0.85 of the mask guide. | `MOVE CLOSER` / `MOVE BACK` |
| Pose | Yaw ≤10°, pitch ≤15°, roll ≤10° (linear proxies over the 6 landmarks, calibrated live). | `TURN HEAD STRAIGHT` / `LOOK STRAIGHT AT CAMERA` / `HOLD HEAD LEVEL` |
| Blur | Scale-invariant edge-width on a fixed-size face crop. | `BLURRY — HOLD STILL` |
| Lighting | Dark / too-bright mean-luma gates. | `TOO DARK — ADD LIGHT` / `TOO BRIGHT` |
| Stability | Head must hold still before the capture fires. | `HOLD STILL` |

Output is **mirrored** to match the live preview (WYSIWYG).

### Runtime & integration features

- **Fully on-device inference** — the models run in a worker; no server, no
  API keys, no data leaving the page (see [Privacy](#privacy)).
- **WebGPU acceleration with automatic fallback** — a WebGPU attempt is
  verified by a warmup timing probe (3 timed warmup inferences); if the GPU
  bridge is slow or absent, the runtime falls back through JSPI-enabled wasm
  → relaxed-SIMD wasm → fixed-width-SIMD wasm. Only the variant the device
  selects is ever downloaded (~9.6 MB wasm, ~2.5–3.5 MB brotli/gzip —
  precompress if your host doesn't compress).
- **Shadow-DOM isolation** — the capture UI mounts in its own shadow root;
  host styles don't leak in and capture styles don't leak out. Fullscreen
  overlay mode or embedded container mode.
- **Warm camera stream** — switching between doc and selfie reuses the live
  stream: same-camera switches never re-call `getUserMedia` and never
  re-prompt. Front/rear switches may prompt once per platform behavior.
- **Camera selection** — optional in-capture dropdown; virtual camera devices
  are filtered out automatically at every device enumeration.
- **Manual capture** — timed or always-on shutter that bypasses the gates,
  returning the QC verdict of the grabbed frame with the image.
- **`maxEdgePx` output clamp** — long-edge downscale at accept, aspect
  preserved, no double compression when the frame is already under the limit.
- **i18n** — English and German built in, per-key string overrides, and
  adding a language is one JSON section (no rebuild).
- **No server requirements** — no COOP/COEP headers, no special MIME config;
  serve the folder like any static site.
- **Integrity-pinned artifacts** — the shipped binaries are
  integrity-protected; use them as distributed, without modification.
- **Diagnostics for free** — every capture returns its QC `flags` object
  (boolean gate verdicts + numeric diagnostics such as perspective/yaw/pitch/
  tilt degrees), so you can build your own funnel analytics or support flow.

---

## Requirements

- **HTTPS is mandatory** — browsers only grant camera access (`getUserMedia`)
  on secure origins (`https://` or `localhost`).
- Modern **Chrome, Edge, Firefox and Safari** (including iOS). WebGPU is
  optional and used when available; the wasm fallback covers everything else.
- **No special server headers** — no COOP/COEP, nothing beyond serving `.js`
  as JavaScript. `sdas-models.pak` is fetched as a binary blob and benefits
  from gzip/brotli like any static asset.
- Keep the `capture-sdk/` folder contents together and the folder name stable
  relative to your page — the SDK resolves every asset relative to the script
  URL. Full layout: `capture-sdk/README.md`.

## Privacy

The SDK is offline-capable and does not upload captures: `report` defaults to
`false`, and the demo app in this repo ships with reporting disabled. When
`report: true` is set, a Report button appears in the capture UI; pressing it
POSTs the current camera frame plus device/diagnostic metadata to the
`/idscan/report` support endpoint. Leave `report` disabled unless you and your
users intend that telemetry.

## License

The repo contents are distributed under the **PolyForm Noncommercial License
1.0.0** — free to use, study, modify and share for any **noncommercial**
purpose; **commercial use requires a separate license from the author**
(contact: <https://cansahin.net>).

```
Required Notice: Copyright 2026 cansahin.net (https://cansahin.net)
```

The full license text is in [LICENSE](LICENSE).

**Third-party components:** the compiled runtime in `capture-sdk/wasm/engine/`
is derived from [LiteRT](https://github.com/google-ai-edge/LiteRT) (formerly
TensorFlow Lite), © the LiteRT Authors, licensed under the Apache License 2.0
— see the attribution section of `capture-sdk/README.md`.
