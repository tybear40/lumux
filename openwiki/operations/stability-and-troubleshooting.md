---
type: "Reference"
title: "Stability and Troubleshooting"
description: "Field stability picture for v0.6.2: the STABILITY_TODOS.md triage (P0 main-thread blocking, portal/GStreamer thread affinity, mode-manager and settings races, P1 protocol and concurrency bugs, P2 cleanup), where the performance knobs (fps, scale, smoothing) take effect, minimal logging, and the missing test suite."
tags: [stability, troubleshooting, threading, race-condition, diagnostics, logging]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
  - id: openwiki-source-cb6086c48ed7f1859e73fe14
    resource: repo://src/lumux/capture.py
  - id: openwiki-source-91b7b97766cd4b166d6345a8
    resource: repo://src/lumux/config/settings_manager.py
  - id: openwiki-source-87ed945ce05051911da34129
    resource: repo://src/lumux/sync.py
  - id: openwiki-source-90ab6c21f07214788bca06ec
    resource: repo://src/lumux/utils/logging.py
  - id: openwiki-source-ea1184835bd4d7b3616bae66
    resource: repo://STABILITY_TODOS.md
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Stability and Troubleshooting

The repository ships a first-class stability artifact: `STABILITY_TODOS.md`, a 550-line investigation dated 2026-11-14 covering all 25 Python files in `src/lumux/` at v0.6.2, written against a user report of an **unstable CachyOS (Arch-based) Wayland desktop**. Its stated big picture: the app mixes a background sync thread, GStreamer streaming threads, a portal D-Bus connection, a `requests` session, an `openssl` subprocess, and GTK's main loop — and *almost every instability class comes from blocking or thread-affine work being done on the wrong thread*.

This page summarizes that triage, the performance knobs, the (minimal) runtime diagnostics, and the verification story. The TODO itself is the authoritative, per-item record with code-line citations and proposed fixes; treat its fix descriptions as plans, not implemented code.

## P0 — freezes, crashes, data loss

Six items, each attributed to a component:

1. **Blocking HTTP on the GTK main thread** (`BridgeClient`, `mode_manager.py`, `settings_dialog.py`, `bridge_wizard.py`, `__main__.py`). `BridgeClient` uses a 5 s per-request timeout; `refresh_devices()` is 4 HTTP calls. Synchronous call sites on the main loop: startup auto-activation of reading mode, every "Start Sync" click (REST activate + DTLS connect), the 1 s-delayed `_finish_switch_to_reading` (a PUT per light), opening Settings (~10 s), and wizard post-auth connect. On a dead bridge these become 20–30 s main-loop freezes — long enough for the `Gio.Application` D-Bus name to be marked unresponsive or lost. Proposed fix: move bridge/DTLS work to worker threads hopping back via `GLib.idle_add` (the `get_bridge_status_async` pattern already in `app_context.py`).
2. **pydbus portal connection used from the wrong thread** (`capture.py`). pydbus is explicitly not thread-safe; `SessionBus` is created on the sync worker thread but `session.Close()` runs on the main thread at Stop/close/quit → ~25 s D-Bus hang or undefined behavior, and the session may never actually close. Proposed fix: a dedicated portal worker thread with a running `GLib.MainLoop`; all session operations enqueued via `queue.Queue`.
3. **GStreamer bus signals never dispatched** (`capture.py`). `bus.add_signal_watch()` is invoked from the sync thread, whose `while … time.sleep()` loop never iterates a GLib context — so `_on_pipeline_error` never fires, the converter-fallback machinery is dead code, and when the portal kills the stream (permission revocation, monitor sleep/wake, resolution change) `capture()` keeps returning the *last frame forever*: lights freeze, UI says "Syncing", nothing recovers until restart. This matches the classic "stops working after a while" report.
4. **Pipeline start/stop races** (`capture.py` vs `app_context.apply_settings`). Main thread and sync thread mutate `_pipeline`/`_appsink`/state concurrently with no lock — `Gst.Pipeline.set_state()` from two threads is unsupported → intermittent crashes/segfaults. The stop path also drops `_latest_frame` without taking the frame lock. Proposed fix: serialize *all* lifecycle operations on one thread; the sync loop only sets a flag.
5. **ModeManager race: room re-lights after "off"** (`mode_manager.py`) — ✅ *reproduced headlessly*. `turn_off()` within the 1 s of a reading switch clears the pending *flag* but cannot cancel the already-scheduled GLib timeout; the interruption check then passes (mode is OFF) and reading mode activates a second later. Repro transcript: mode `OFF` right after `turn_off`, `READING` 1.5 s later. Proposed fix: store the source id and `GLib.source_remove()` it.
6. **Non-atomic settings write + all-or-nothing strict load** (`settings_manager.py`) — ✅ *reproduced*. `save()` writes `settings.json` in place; a SIGKILL mid-write (it saves on *every sync start* via the restore-token callback) can persist a config that is partly defaults — silently deleting the bridge IP/app key. A file written by a newer version with one unknown key fails strict dataclass unpacking and discards *everything*. Proposed fix: `os.replace` atomic write + `.bak` recovery, tolerant per-field filtering, optional `schema_version`.

## P1 — functional reliability

Nine items, notable ones:

- **`EntertainmentStream` message buffer assumes a 36-char config ID** ✅ *reproduced* — a non-UUID zone ID resizes the buffer so channel packing is misaligned; `struct.error` is caught and printed **30×/s forever** with sync silently dead.
- **`HueBridge.create_user()` ignores `max_retries`/`timeout`** — the docstring promises exponential backoff while the body makes one 5 s call; the wizard's params go unused, so a late link-button press fails once.
- **Icon paths point at nonexistent files** (`__main__.py`, `tray_icon.py`) — blank tray icon in dev.
- **DTLS via `openssl s_client`**: undrained stdout/stderr pipes fill after 64 KB and stall DTLS; dead processes become zombies; no reconnection — video sync silently does nothing until manual restart.
- **One `requests.Session` shared across threads** — `requests`/`urllib3` sessions are not thread-safe; sporadic `ConnectionError`s and "Cannot connect" flicker.
- **`apply_settings()` leaves a stale `EntertainmentStream`** in both consumers when streaming is disabled; **dead `"ready"` status-card state** after Stop; **`refresh_devices()` re-run on every brightness slider release** (6 HTTP calls per stop); **sync loop swallows tracebacks**.

## P2 — robustness and cleanup

`SyncController.stop()` re-entrancy; portal `SelectSources`/`Start` loops with no timeout (infinite sync-thread block); dead code (`ZoneMapping`, `data/default_settings.json`, `HueBridge.set_zone_color`, superseded entertainment channel-mapping helpers); **no real logging** (item 19); dependency/packaging notes (no wheels for PyGObject/pycairo, unpinned `zeroconf`, unguarded `pydbus` import, verify Flatpak runtime ships `pipewiresrc`); an unfinished uncommitted `app_context.py` edit; and a grab-bag of small issues.

The file closes with a suggested implementation order (item 1 first for UX; the portal worker thread is a prerequisite for items 3–4; items 5–6 are small and independent) and a 10-point **on-desktop verification plan** (startup with bridge unreachable, Stop-within-1 s, portal permission revocation, scale change mid-sync, SIGKILL mid-save, quit with bridge unreachable, DTLS loss, late link-button, new log file, normal regression pass).

## Performance knobs and where they take effect

Three settings, all clamped in [SettingsManager](../configuration/settings-and-state.md) and documented in the README's Performance Tips (lower scale, 15–30 fps, more smoothing/transition):

| Knob | Setting | Where it takes effect |
|---|---|---|
| Resolution scale | `CaptureSettings.scale_factor` (0.125 default, clamped 0.01–1.0) | `capture.py` — scales the pipeline `videoscale` target dimensions; a change triggers a pipeline restart |
| Frame rate | `SyncSettings.fps` (30 default, clamped 1–60) | `sync.py` — the loop sleeps `1/fps − elapsed` each frame; live read per frame, so settings changes apply without restart |
| Smoothing | `SyncSettings.smoothing_factor` (0.3 default, clamped 0.1–1.0) | `sync.py` — lerps this frame's colors toward previous via `ColorAnalyzer.apply_smoothing` before light update |

(Black-bar detection has its own `smooth_factor` for crop-region transitions. The README's Troubleshooting section maps the three common complaints — Wayland capture, bridge connectivity, high CPU — to portal configuration, network, and these knobs respectively.)

## Runtime diagnostics

Diagnostics are deliberately thin: `utils/logging.py` is a 14-line module whose only export is `timed_print` — a timestamped `print` to stdout. There is no file logging, no `threading.excepthook`, no `faulthandler`, so worker-thread exceptions and segfaults leave no persistent trace (STABILITY_TODOS item 19 proposes a rotating log, an excepthook, `faulthandler.enable()`, and startup environment logging). The sync loop does maintain a `_stats` dict (fps, per-stage timings incl. smooth time) surfaced for the GUI, and the 100 ms status poll is the main visibility into live state.

## The missing test suite

`pyproject.toml` declares `testpaths = ["tests"]` and `pythonpath = ["src"]` under `[tool.pytest.ini_options]` and lists `pytest`/`pytest-asyncio` in the dev extras — but the repository **ships no `tests/` directory**. Verification is therefore manual plus what `STABILITY_TODOS.md` reproduced headlessly (items 5, 6, 7) and its desktop verification plan. No automated regression coverage exists for the sync pipeline, mode transitions, or settings I/O.

See also: [Application Shell](../architecture/application-shell.md), [Entertainment Streaming](../bridge/entertainment-streaming.md), [REST Client](../bridge/rest-client.md), [Screen Capture](../pipeline/screen-capture.md), [Sync Engine](../pipeline/sync-engine.md), [Settings and Persistent State](../configuration/settings-and-state.md), [Zone Color Pipeline](../pipeline/zone-color-pipeline.md), [GTK4 Interface](../gui/gtk4-interface.md), [Distribution](../packaging/distribution.md).
