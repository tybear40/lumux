---
type: "Reference"
title: "Zone Color Pipeline"
description: "Frame-to-color math: ZoneProcessor ambilight edge decomposition into 2*rows + 2*cols RGB zones, ColorAnalyzer gamma + max-component brightness + EMA smoothing, and RGB-to-CIE-XY conversion with optional phosphor-triangle gamut clipping."
tags: [zones, color, xyz, gamma, brightness, smoothing, pipeline]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
  - id: openwiki-source-0d3687dfbe25495f19a32003
    resource: repo://src/lumux/colors.py
  - id: openwiki-source-aea183e9f095e7c30102169f
    resource: repo://src/lumux/gui/main_window.py
  - id: openwiki-source-f1f362d59f63cd8185be1d88
    resource: repo://src/lumux/gui/settings_dialog.py
  - id: openwiki-source-aafb67c303d27868494f68c1
    resource: repo://src/lumux/hue_bridge.py
  - id: openwiki-source-87ed945ce05051911da34129
    resource: repo://src/lumux/sync.py
  - id: openwiki-source-09014fd114b8e79fe030dfbe
    resource: repo://src/lumux/utils/rgb_xy_converter.py
  - id: openwiki-source-28b5c779161b32953ca58614
    resource: repo://src/lumux/zones.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Zone Color Pipeline

Between a captured frame and a light update, two pure-math stages run (all NumPy, no image library): **`ZoneProcessor`** (zones.py) decomposes the frame into edge zone RGB colors, and **`ColorAnalyzer`** (colors.py) converts those into Hue-ready `(xy, brightness)` values. `utils/rgb_xy_converter.py` holds the shared colorimetry. The sync loop drives them — `_process_frame` calls `capture()`, `zone_processor.process_image()`, `analyze_zones_batch()`, `apply_smoothing()`, `_update_lights()` in order (see [Sync Engine](sync-engine.md)).

## Zone decomposition

`ZoneProcessor` (default `rows=16, cols=16`, overridable via `ZoneSettings` in AppContext) divides the frame into **edge zones only** — the screen center is ignored:

- **edge width** = `min(width // cols, height // 8)`, floored at 5 px — a strip sampled along each screen edge.
- **top / bottom**: the width is split into `cols` equal vertical strips; each zone averages the `edge_width`-px band at the top (or bottom) of its strip → int RGB tuple.
- **left / right**: the height is split into `rows` equal horizontal bands; each averages the `edge_width`-px band at the left (or right) edge.

With the 16×16 default that is **64 zones** (2·cols + 2·rows), stored as plain `(r, g, b)` int tuples. The processor normalizes inputs (grayscale 2-D arrays are stacked to 3 channels, 4-channel arrays drop alpha), returns `{}` for frames smaller than 2×2 px, and swallows *any* exception with a print plus `{}` — a bad frame becomes an empty update, never an error. The channel mapping in the [Sync Engine](sync-engine.md) maps only `rows // 2` left/right zones (covering the full edge span); the lower-half left/right zones have no channel target and are dropped at update time.

## Color analysis

Per zone, `ColorAnalyzer.analyze_zone()` applies three transforms:

1. **Gamma** — each channel is normalized to 0–1, raised to the `gamma` power (values ≤ 0 fall back to 1.0 = none), and scaled back to 0–255 ints.
2. **Brightness** — `max(r, g, b) / 255 * 254 * brightness_scale`, clamped to 1–254. The max-component model (rather than a luminance formula) deliberately keeps brightness uniform across hues: a luminance formula would weight green at ~71.5% and blue at ~7.2%, making blues and purples appear dimmer on the lights.
3. **xy chromaticity** — via `rgb_to_xy`; the optional per-zone `light_info` (carrying the light's phosphor gamut, populated by `HueBridge.refresh_devices()` from the Hue API) would enable gamut clamping — but the sync loop calls `analyze_zones_batch()` **without** a light info map, so clamping is dormant in the shipped sync path; the mechanism is available and unit-tested through the public converter API.

`brightness_scale` and `gamma` are constructed from `settings.sync` in AppContext. `apply_smoothing(current, factor)` then blends each zone toward its previous value with an exponential moving average in **xy space** (brightness lerp'd in ints, then truncated): `smoothed = prev + factor * (current − prev)`; zones seen for the first time pass through unsmoothed. The factor is `settings.sync.smoothing_factor`.

## RGB ↔ CIE xy

`utils/rgb_xy_converter.py` implements the standard conversion pair:

- **`rgb_to_xy`** linearizes each sRGB channel with the piecewise transfer function (`≤ 0.04045 → v/12.92`, else `((v+0.055)/1.055)^2.4`), multiplies by the sRGB→XYZ (D65) matrix, and normalizes: `x = X/(X+Y+Z)`, `y = Y/(X+Y+Z)`. All-black input (zero total) falls back to the white point `(0.3227, 0.3290)`.
- **Gamut clipping**: when a `gamut` (or `light_info["gamut"]`) with red/green/blue phosphor `{x, y}` points is given, a point inside the phosphor triangle is returned unchanged; an out-of-gamut point is projected to the **nearest of the three triangle edges** (closest point on each segment, clamped to the segment, then minimum distance).
- **`xy_to_rgb`** inverts the pair (`Y = 1`, XYZ→linear sRGB matrix, piecewise inverse transfer, clamp to 0–1). GUI use only: reading-mode preset swatches (main_window, settings_dialog). The same file's `rgb_to_xy` is also used by the GUI color pickers to convert chosen preset colors.

## See also

[Sync Engine](sync-engine.md) (drives the pipeline) · [Screen Capture](screen-capture.md) (frame source) · [Entertainment Streaming](../bridge/entertainment-streaming.md) (consumes xy, brightness) · [REST Client](../bridge/rest-client.md) (light_info source) · [Reading Mode](../modes/video-reading-modes.md) · [Settings](../configuration/settings-and-state.md) (`sync.gamma`, `sync.brightness_scale`, `sync.smoothing_factor`, zone grid size)
