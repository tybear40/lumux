---
type: "Reference"
title: "Screen Capture Pipeline"
description: "How Lumux gets pixels on Wayland: the xdg-desktop-portal ScreenCast session lifecycle (CreateSession with persist_mode, SelectSources, Start, restore tokens for persistent consent), the ranked GStreamer pipeline candidates with runtime-failure memory, frame extraction with stride handling, format conversion, PIL scaling, and black-bar cropping."
tags: [capture, pipewire, portal, gstreamer, wayland, black-bar, performance]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
  - id: openwiki-source-ce2567cb3bf56afd333d2acf
    resource: repo://src/lumux/black_bar_detector.py
  - id: openwiki-source-cb6086c48ed7f1859e73fe14
    resource: repo://src/lumux/capture.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Screen Capture Pipeline

`ScreenCapture` (capture.py, 736 lines) is the only source of pixels: on Wayland it uses the **xdg-desktop-portal ScreenCast API** (via pydbus) to obtain a PipeWire node, then runs a **GStreamer pipeline** (`pipewiresrc → converter → appsink`) that hands frames to a callback, where numpy does stride handling, format conversion, scaling, and optional letterbox cropping. There is no X11/XSHM or host-monitor path — the portal is mandatory.

## Portal session lifecycle

`_setup_portal_session()` (capture.py L239-L392) talks to `org.freedesktop.portal.Desktop` / `org.freedesktop.portal.ScreenCast` over a pydbus session bus, with each round-trip blocking a `GLib.MainLoop` that quits on the `Response` signal (a 30 s timeout guards the loop):

1. **`CreateSession`** — options include `session_handle_token` (a timestamp-suffixed unique token), **`persist_mode = 2`** (persist consent until the user explicitly revokes it), and, when available, the previously saved **`restore_token`**. An empty restore token is deliberately *not* sent — "sending an empty string can confuse some portal backends."
2. **`SelectSources(session_handle, {types, multiple: false})`** — `types` is 1 for whole screen, 2 for a specific window (follows `settings.capture.source_type`).
3. **`Start(session_handle, "", {})`** — returns the PipeWire node id (and possibly a fresh restore token).

**Persistent consent via restore tokens.** When the portal was granted `persist_mode = 2` consent, `Start` can return a `restore_token`. The class exposes it through `set_on_restore_token_changed(callback)`; AppContext wires that to `settings.capture.restore_token` + `save()` (app_context.py L33-L38, L160-L165), so the next run passes it back in `CreateSession` and the permission dialog is skipped. If the host portal is older and issues no token, the log explicitly says the prompt will recur each sync ("needs GNOME 47+ / Plasma 6.x").

**Stale-token recovery.** Inside `capture()`, if session setup fails *and* a cached restore token was in use, the token is cleared (callback fired with `""`, which AppContext persists) and `_setup_portal_session()` retries once without it, forcing a fresh permission dialog. This handles the case where a monitor or source change invalidated the token silently.

`stop_pipeline()` tears everything down in order: pipeline → reset dimensions/scale state → clear `_runtime_failed_configs` → close the portal session (D-Bus `session.Close()`), and `__del__` calls it as a safety net.

## Pipeline construction and fallback

`_get_pipeline_configs(node_id)` builds a **ranked candidate list** of `pipewiresrc path=<node> do-timestamp=true ! ... ! appsink` strings, gated by `Gst.ElementFactory.find()` probes for each converter plugin:

1. `glupload ! glcolorconvert ! glcolorscale ! gldownload ! videoscale` — GPU scale to the target size (best when `gstreamer1.0-plugins-gl` is present)
2. `glupload ! glcolorconvert ! gldownload ! videoscale` — GPU convert, CPU scale
3. `v4l2convert ! videoscale` — V4L2 hardware path
4. `videoconvert ! videoscale` — pure CPU
5. then unscaled variants (gl / v4l2 / videoconvert → `video/x-raw,format=RGB`)

When a scale factor < 1.0 is set, the early candidates carry **scaled caps** (`video/x-raw,format=RGB,width=W,height=H`) so the pipeline emits pre-scaled frames; the dimensions are **rounded down to a multiple of 4** ("so RGB row strides stay 4-byte aligned … and odd-width caps that some converters refuse to link"). The appsink is `drop=true max-buffers=1 sync=false` — it always serves the freshest frame and never queues.

`_start_pipeline()` (L476) iterates the candidates: a `set_state(PLAYING)` returning `FAILURE` marks the config and moves to the next; the first that reaches PLAYING wins and logs which converter (`desc`) it used.

**Runtime-failure memory.** Config strings embed a node id and dimensions, so `_config_key()` strips those (`path=\d+`/`width=\d+`/`height=\d+` → `*`) into a stable key. When the *running* pipeline later emits a bus error (`_on_pipeline_error`, L552 — e.g. a DMA-BUF buffer it can't map, a mode change), the active key is added to `_runtime_failed_configs` and the pipeline stops. Future starts skip remembered configs, so a converter that fails at runtime is not retried within the session; `stop_pipeline()` clears the set so a new capture session starts optimistic again. If a running pipeline's frame size changes while *not* pipeline-scaled, `_on_new_sample` sets `_needs_pipeline_restart`, which `capture()` acts on by restarting the pipeline (and rebuilding with the new target dims) before returning a frame.

When no candidate starts, `_log_pipeline_details()` dumps pipeline state, the node id, and the presence of every probed plugin (pipewiresrc, videoconvert, videoscale, v4l2convert, glupload, glcolorconvert, gldownload, glcolorscale) — this is the diagnostic the Stability page tells users to read.

## Frame processing

`_on_new_sample` (L630, the appsink `new-sample` callback) per frame:

1. **Pull** the sample, read width/height/format from its caps.
2. **Stride**: `GstVideo.buffer_get_video_meta(buffer).stride[0]` when available, else inferred from buffer size — GStreamer pads rows to alignment boundaries, so buffer size can exceed `height * width * bpp`.
3. **Map READ** and copy to bytes. A failed map is logged as the DMA-BUF signature: "This likely means DMA-BUF buffers are being received. Ensure GStreamer GL plugins (gst-plugins-gl) are installed."
4. **`_extract_pixel_rows`** (L597) — takes the raw bytes plus stride, and if padded, builds a strided numpy view via `np.lib.stride_tricks.as_strided` and `.copy()` to detach from the mapped buffer; validates buffer size and raises on a too-small buffer.
5. **Format conversion** to an H×W×3 RGB uint8 array: `BGR` is flipped and copied; `RGBA`/`RGBx` are kept as 4 channels (zone processing handles alpha); `BGRA`/`BGRx` have channel order rearranged; 15-bit `RGB15`/`BGR15` are unpacked from a uint16 view with 5→8 bit scaling; everything else is assumed RGB.
6. The frame + raw bytes are stored under `self._frame_lock`; with `drop=true max-buffers=1` the sink already discarded stale frames, so this is always the latest.

`capture()` then returns the frame through `_process_image()` in two stages:

- **Black-bar crop** (when a detector is attached — see below).
- **PIL scale** — but *only* when the pipeline itself is not already emitting scaled frames (`_pipeline_scaled` is false and scale < 1.0): the array is made C-contiguous, wrapped in `PIL.Image`, resized **bilinear**, and re-converted. Pipeline-side scaling is preferred because it happens in the GStreamer thread and avoids a second full-frame copy.

`capture()` blocks up to 2 s waiting for the first frame after (re)starting the pipeline, and returns `None` on timeout — the Sync loop tolerates that and simply renders nothing this frame.

## BlackBarDetector

`black_bar_detector.py` (292 lines) implements letterbox/pillarbox cropping (see [Zone Color Pipeline](zone-color-pipeline.md) for how the cropped frame feeds zone extraction):

- **`_detect_bars`** computes BT.601 luma `0.299R + 0.587G + 0.114B` directly on the numpy array (no PIL round-trip), reduces to row/column mean luminance, and walks in from each edge counting **contiguous** lines whose mean ≤ `threshold` (default 10, clamped 0-50).
- **Validity guards**: if cropping would remove more than half of a dimension (content < 50%), that axis is reset; each edge is additionally capped at `dim // 2 - 1`; any exception resets to "no crop" rather than corrupting the frame.
- **Rate limiting**: detection runs every `detection_rate` frames (default 30, clamped 1-120), not every frame.
- **Smoothing**: the returned crop lerps toward the target each frame at `smooth_factor` (default 0.3, clamped 0.1-1.0), with degenerate-region fallbacks (a crop ≤ 1 px wide/tall re-expands to full frame), so bar transitions are gradual.
- **`_should_crop`** suppresses sub-pixel jitter: no crop is applied unless the crop differs from the full frame by **more than 2%** in either dimension.

The detector instance is created in `ScreenCapture.__init__` from the `BlackBarSettings` dataclass; `update_black_bar_settings()` lets the Settings UI change enabled/threshold/rate/smoothing live without a capture restart. `ScreenCapture.get_black_bar_crop_region()` exposes the current crop so the main window can overlay the "cropped content" outline on the zone preview.

## See also

[Overview](../architecture/overview.md) · [Zone Color Pipeline](zone-color-pipeline.md) (what happens to each frame) · [Sync Engine](sync-engine.md) (the loop that calls `capture()`) · [Stability and Troubleshooting](../operations/stability-and-troubleshooting.md) (portal/GStreamer/DMA-BUF failure modes) · [Settings and Persistent State](../configuration/settings-and-state.md) (capture settings, restore token)
