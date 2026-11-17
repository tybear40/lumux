---
type: "Reference"
title: "Architecture Overview"
openwiki_generated: true
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-cb3129d36b1c78edd3deb82a
    resource: repo://src/lumux/__init__.py
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
  - id: openwiki-source-5f020fc177d3c3a9da9013f5
    resource: repo://src/lumux/bridge_client.py
  - id: openwiki-source-cb6086c48ed7f1859e73fe14
    resource: repo://src/lumux/capture.py
  - id: openwiki-source-f69325c01fbd74355c2bd36a
    resource: repo://src/lumux/entertainment.py
  - id: openwiki-source-aea183e9f095e7c30102169f
    resource: repo://src/lumux/gui/main_window.py
  - id: openwiki-source-87ed945ce05051911da34129
    resource: repo://src/lumux/sync.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Architecture Overview

Lumux is a Python GTK4/libadwaita application that captures the screen in real time and drives Philips Hue lights. The `lumux` package (`src/lumux/__init__.py`, version 0.6.2) re-exports its core components — `AppContext`, `HueBridge`, `BridgeClient`, `ScreenCapture`, `ZoneProcessor`, `SyncController`, `ModeManager`, `EntertainmentStream`, `ReadingModeController`, `BlackBarDetector`, `ColorAnalyzer` — which together form a capture → zone → color → sync → bridge pipeline.

The core value flow per video frame:

1. **Screen capture** (`capture.py`): a GStreamer `pipewiresrc` pipeline, fed by an xdg-desktop-portal ScreenCast session, produces RGB frames on GStreamer's streaming thread.
2. **Zone decomposition** (`zones.py`): each frame is split into ambilight edge zones (top/bottom/left/right strips) and each zone is reduced to an average RGB color.
3. **Color analysis** (`colors.py`): zone RGB is converted to CIE xy + brightness with gamma and brightness-scale correction, then temporally smoothed.
4. **Sync orchestration** (`sync.py`): a background `SyncLoop` thread paces the pipeline to the configured FPS, maps zones to entertainment channels, and ships colors to the lights.
5. **Bridge transport** (`bridge_client.py`, `entertainment.py`): colors leave over one of two transports — the low-latency DTLS entertainment stream (video mode) or the REST v2 API (discovery, configuration, reading mode).

## Component ownership

`AppContext` (`app_context.py`) is the composition root. It is built from the `SettingsManager` singleton and owns every long-lived component:

| Component | File | Role |
|---|---|---|
| `HueBridge` / `BridgeClient` | `hue_bridge.py` / `bridge_client.py` | Bridge discovery, auth, REST v2 API, device/zone caches |
| `ScreenCapture` | `capture.py` | Portal session + GStreamer pipeline, latest-frame buffer |
| `ZoneProcessor` | `zones.py` | Ambilight zone grid → average RGB per zone |
| `ColorAnalyzer` | `colors.py` | RGB → CIE xy + brightness, gamma, EMA smoothing |
| `EntertainmentStream` | `entertainment.py` | DTLS-PSK HueStream v2 client (openssl subprocess) |
| `SyncController` | `sync.py` | Background sync loop, FPS pacing, GUI status queue |
| `ModeManager` | `mode_manager.py` | OFF/VIDEO/READING state machine |

AppContext also wires the cross-component edge: `sync_controller.set_on_stop_callback(mode_manager.on_video_sync_stopped)`, so a stopped video sync lets the mode manager react (e.g. auto-activate reading mode).

## Threading model

| Thread | Owner | Work |
|---|---|---|
| GTK main loop | `MainWindow`, `ModeManager`, tray | UI updates, mode switches, portal D-Bus session setup |
| `SyncLoop` (daemon) | `SyncController._sync_loop` | Per-frame: `capture()` → `process_image` → `analyze_zones_batch` → `apply_smoothing` → `_update_lights` |
| GStreamer streaming thread | `ScreenCapture._on_new_sample` | `pull-sample`, pixel extraction, frame stored in `_latest_frame` under `_frame_lock` |
| GStreamer bus watch thread | `ScreenCapture` bus handlers | Pipeline errors → converter fallback / restart request |
| Daemon status thread | `AppContext.get_bridge_status_async` | Bridge status polling off the main loop |
| openssl `s_client` child process | `EntertainmentStream` | DTLS-PSK socket to bridge port 2100 |

Key shared-state boundaries:

- **Frame handoff** — `_latest_frame` is written by the GStreamer thread and read by the sync thread under `_frame_lock`; `capture()` only ever returns the newest frame (latest-wins), never blocking on a slow consumer.
- **GUI handoff** — `SyncController` exposes a bounded `queue.Queue(maxsize=100)`; `_queue_status` drops updates when full (`put_nowait` + `queue.Full` → pass). `MainWindow` drains it via `GLib.timeout_add(100, self._update_status)` — a 100 ms poll on the main loop.
- **Stats** — the `_stats` dict (fps, frame_count, errors, per-stage timings) is guarded by `SyncController.lock`; the GUI polls `get_stats()`.
- **Pipeline health** — `_needs_pipeline_restart` / `_runtime_failed_configs` let the sync thread's `capture()` call trigger converter fallback without a dedicated pipeline thread.

## The two bridge transports

- **REST v2 API** (`BridgeClient`): a `requests.Session` with a 5 s default timeout and `verify=False` (the Hue bridge serves a self-signed certificate). Used for discovery, user creation, device/zone/entertainment-config queries, and one-shot light state writes (reading mode).
- **DTLS entertainment streaming** (`EntertainmentStream`): an `openssl s_client -dtls1_2` subprocess connected to the bridge's port 2100, speaking the HueStream v2 binary protocol. Used for per-frame color updates in video mode, because DTLS avoids per-request HTTP round trips.

`SyncController._update_lights` only sends when the entertainment stream reports connected; otherwise frames are computed but dropped (with a periodic warning). Video mode = DTLS + sync loop; reading mode = REST puts only; OFF = neither.

## Data-flow notes

- `SyncController.start()` builds the zone→channel mapping first (position-based nearest-channel matching, since entertainment channel x/z coordinates are in normalized -1..1 space) and then starts the `SyncLoop` daemon thread.
- FPS is clamped to 1–60; pacing sleeps for the remainder of the target frame time, so heavy frames degrade gracefully rather than overrunning the lights.
- Errors in the loop increment the error counter, queue an `"error"` status for the GUI, and back off 1 s before continuing — the loop never dies on a transient bridge failure.

## Cross-references

- Capture detail (portal, pipeline fallback, black bars): [Screen Capture Pipeline](../pipeline/screen-capture.md)
- Zone/color math: [Zone and Color Pipeline](../pipeline/zone-color-pipeline.md)
- Sync loop internals: [Sync Engine](../pipeline/sync-engine.md)
- REST client + discovery: [Hue Bridge REST Client](../bridge/rest-client.md)
- DTLS protocol: [Entertainment Streaming (DTLS)](../bridge/entertainment-streaming.md)
- Mode state machine: [Video and Reading Modes](../modes/video-reading-modes.md)
