# Add Xteink X3 support + fix image fit-box orientation

## Summary

Two related changes, both validated on real hardware (X4 + X3):

1. **X3 support** (additive) — a device toggle in the UI with an X3 profile: 528×792 screen, same 4-level SSD1677 grayscale treatment as the X4.
2. **Portrait fit-box fix** (changes existing X4 output) — images now fit the display orientation (X4: 480×800, X3: 528×792) instead of the panel scan orientation (800×480). On-device, this is the difference between a cover that fills the screen and one that renders as a small box in the corner.

## 1. X3 support

The X3's display is **528×792 portrait** (3.68", ~259 PPI — confirmed by the PPI math against Xteink's official spec; some review sites incorrectly report it as sharing the X4's 480×800). Same SSD1677 controller and ESP32-C3 as the X4, so the existing 4-level grayscale pipeline applies unchanged — the profile only differs in dimensions.

Implementation:

- `image_processor.py`: `DEVICE_PROFILES` dict, `gray_levels` on `ImageOptions`, `_quantize_to_levels()` generalizes the existing 4-level quantizer to any palette (machinery for future devices with different gray depths)
- `epub_processor.py`: `ProcessingOptions.device`; generated covers use device dimensions and are quantized to the device palette so they match the display gamut
- `app.py`: `device=x4|x3` query param on `/process/{task_id}` (validated, defaults to x4)
- UI: X4/X3 segmented toggle above the presets; device choice is independent of the Quick/Full/Custom preset

**A finding worth documenting** (now in the README): stock X3 firmware does not render images inside EPUBs *at all*. Hardware-tested with an unmodified store EPUB and a diagnostic EPUB covering 8 encoding variants (odd/even widths, optimized/standard Huffman tables, 4:2:0/4:4:4 subsampling, single-channel JPEG, small dimensions, PNG, BMP) — every image page renders blank on stock, including the publisher's original JPEG. X3 image output therefore targets CrossPoint-family firmware, which renders 4-level grayscale on the X3 panel.

## 2. Portrait fit-box fix

The existing code fit images within **800×480** (panel scan orientation). But the readers display portrait — CrossPoint's converter profiles are:

```js
const DEVICE_PROFILES = {
  X4: { width: 480, height: 800, label: 'X4' },
  X3: { width: 528, height: 792, label: 'X3' },
```

and the firmware's `XtcTypes.h` defines `DISPLAY_WIDTH = 480; DISPLAY_HEIGHT = 800` (its debug tooling rotates the raw landscape framebuffer 270° for screenshots).

With the old 800×480 box, a portrait cover was capped at 480px tall (e.g. 320×480), and the firmware renders images at **native size without upscaling** — so covers drew as a small box with the rest of the screen blank. With the portrait box the same cover comes out 480×721 and fills the display.

Photos from an X4 (same book, same cover page):

| Old 800×480 box | New 480×800 box |
|---|---|
| *(photo: cover renders ~40% of screen)* | *(photo: cover fills screen)* |

This also fixes Light Novel mode: rotated landscape pages now fill the full 480×800 instead of being squeezed to 480px tall.

## Hardware validation

| Test | Device / firmware | Result |
|---|---|---|
| Portrait vs landscape fit box A/B | X4, CrossInk (CrossPoint fork) | Portrait fills screen; landscape renders small (photos above) |
| X3-processed EPUB | X3, CrossInk | Images render with correct dimensions and tone |
| X3-processed EPUB | X3, **stock** | Image pages blank — stock does not render EPUB images (see finding above; text renders fine) |
| Original unprocessed EPUB (control) | X3, stock | Image pages equally blank — confirms stock limitation, not a pipeline regression |
| 8-variant encoding diagnostic | X3, stock | All blank — rules out odd-width/Huffman/subsampling/format/size causes |

Pipeline-level verification (2.5MB store EPUB, 13 images): both profiles process in ~0.5s; all images baseline JPEG within the fit box; mimetype first ZIP entry stored uncompressed; all 30 XHTML files well-formed with identical word counts before/after; OPF/NCX parse cleanly.

## Incidental changes

- Batch ZIP renamed `x4_optimized_*` → `epubkit_optimized_*` (no longer X4-only)
- `*.epub` added to `.gitignore` (keeps local test books out of the repo)
- Cache-busters bumped on `style.css` / `app.js`

## Compatibility notes

- `ProcessingOptions` / `ImageOptions` field defaults are unchanged for existing callers; `device='x4'` is the default everywhere.
- The fit-box change affects existing X4 users' output (intentionally — see photos). Anyone who preferred the old behavior can pass custom `max_width`/`max_height` via `ImageOptions`.
