---
type: "Reference"
title: "Sync Engine"
description: "SyncController: the single background sync thread, FPS clamping and pacing, per-stage timing stats, the bounded status queue drained by the GUI, position-based zone-to-entertainment-channel mapping with multi-zone averaging, and light update dispatch over DTLS."
tags: [sync, threading, fps, statistics, entertainment, architecture]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-aea183e9f095e7c30102169f
    resource: repo://src/lumux/gui/main_window.py
  - id: openwiki-source-aafb67c303d27868494f68c1
    resource: repo://src/lumux/hue_bridge.py
  - id: openwiki-source-77a99335182aa760e79ac6da
    resource: repo://src/lumux/reading_mode.py
  - id: openwiki-source-87ed945ce05051911da34129
    resource: repo://src/lumux/sync.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Sync Engine

`SyncController` (sync.py, 369 lines) is the orchestrator that ties capture → zones → colors → lights into one loop running on a **single daemon thread** named `SyncLoop`. It owns: the thread lifecycle, FPS pacing, per-stage timing statistics, the cross-thread status channel to the GUI, the zone→entertainment-channel mapping, and the final DTLS dispatch. It holds references to `HueBridge`, `ScreenCapture`, `ZoneProcessor`, `ColorAnalyzer`, `ZoneMapping`, settings, and an optional `EntertainmentStream` (all injected — see [Overview](../architecture/overview.md)).

## Thread lifecycle

- **`start()`** (L59) is a no-op if already running. If an entertainment stream object exists it checks `is_connected()`; only when connected does it call `_build_zone_channel_mapping()` — an unconnected stream at start time just leaves an empty map (with a warning), since zone colors have no channel to go to. Then it sets `running = True` and spawns the daemon thread.
- **`_sync_loop`** (L208) runs `while self.running`: process one frame, then sleep to hit the FPS target. `KeyboardInterrupt` breaks the loop; any other exception increments `errors`, queues an `"error"` status, and sleeps 1 s before continuing — the loop never dies on a transient failure. On exit it queues `("status", "stopped")`.
- **`stop()`** (L181) is a clean shutdown sequence: `running = False` → `thread.join(timeout=3)` (a thread still alive after 3 s logs a warning — the loop sleeps at most one frame interval, so this is only for wedged pipelines) → `capture.stop_pipeline()` (releases the portal session) → the optional `_on_stop_callback()` in a try/except (AppContext wires this for auto-switching into reading mode after a manual stop).

## FPS pacing and stats

The loop clamps the configured FPS into a safe range and paces by sleeping the remainder of the frame budget:

```
fps_target = int(settings.fps, fallback 30)
fps_target = max(1, min(60, fps_target))     # safe range 1–60
delay = max(0, 1.0/fps_target - elapsed)      # elapsed = capture+analyze+update
time.sleep(delay)
```

The reported **real FPS** is measured over a rolling window of the last 30 *total* loop times (including sleep), so it reflects achieved throughput, not the target. The stats dict is:

| key | meaning |
|-----|---------|
| `fps` | 1 / mean total loop time over the last 30 frames |
| `frame_count` | frames successfully processed since start |
| `errors` | exceptions caught in the loop |
| `last_update` | wall time of stats creation/reset |
| `last_stage_times` | dict of `capture` / `zones` / `analyze` / `smooth` / `update` / `total`, seconds, recorded under `self.lock` each frame |

`_process_frame` (L255) times every stage and bails early on `None`/empty results (a `None` frame from a capture timeout is simply skipped without an error). `get_stats()` returns a copy under the lock; `reset_stats()` restores the four base keys. The GUI reads them from the GTK thread — the lock makes that safe.

## Status queue to the GUI

Cross-thread UI updates go through `queue.Queue(maxsize=100)`:

- The sync thread calls `_queue_status(status_type, message, data)` with `put_nowait` and **drops on `queue.Full`** — the pipeline never blocks on a slow UI.
- The main window drains it in `_update_status()` (main_window.py L59, L745) on a **100 ms GLib timeout**: it loops `get_status()` (`get_nowait`) until the queue is empty and applies only the *latest* item. `"status"/"syncing"` items carry RGB `zone_colors` as `data`, which feed the zone preview widget; `"stopped"` re-enables the sync button (and re-presents the window if it had minimized to tray); `"error"` items set the status card to disconnected. FPS is rendered as `f"{stats['fps']:.1f}"`.

## Zone → entertainment channel mapping

Entertainment streaming cannot address arbitrary zones, so sync maps the 8-zone ambilight layout onto the physical light *channels* the bridge exposes. `_build_zone_channel_mapping()` (L85) generates zone ids — `top_0..top_{cols-1}`, `bottom_0..bottom_{cols-1}`, `left_0..left_{rows/2-1}`, `right_0..right_{rows/2-1}` — and for each calls `_find_best_channel_for_zone()` (L118):

1. Parse the zone id into (edge, index).
2. Convert to the entertainment position space, where **x is left (−1) → right (+1)** and **z is bottom (−1) → top (+1)** (a left-edge zone, for example, targets x=−1 with z interpolated top→bottom across its index range).
3. Pick the channel whose reported `(x, z)` — from `EntertainmentStream.get_channel_positions()`, populated when the DTLS connection is accepted — has the smallest squared distance.

The result is a `zone_id → channel_id` dict; several zones typically land on the same physical light. `_update_lights()` (L303) therefore **merges multiple zones per channel**: xy coordinates are averaged per-component and brightness integer-averaged (`// 2`) on each merge. If the stream dropped (checked every frame, warning logged once per 300 frames) the update is skipped. Otherwise the merged channel dict is sent in one call: `entertainment_stream.send_colors_xy(channel_colors)` — the DTLS path documented in [Entertainment Streaming](../bridge/entertainment-streaming.md). The REST client's per-light setters (`HueBridge.set_light_color` / `set_light_gradient` / `set_zone_color`) exist but are used by **Reading Mode** for one-shot scene settings, not by the sync loop — so a dropped DTLS connection (rather than a REST fallback) is what halts live updates.

## See also

[Overview](../architecture/overview.md) · [Entertainment Streaming](../bridge/entertainment-streaming.md) · [Zone Color Pipeline](zone-color-pipeline.md) · [Screen Capture](screen-capture.md) · [GTK 4 Interface](../gui/gtk4-interface.md) (the queue consumer) · [Stability and Troubleshooting](../operations/stability-and-troubleshooting.md)
