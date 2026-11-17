---
type: "Reference"
title: "Application Shell and Startup"
openwiki_generated: true
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-f4d1578b16cd7e96bab8322b
    resource: repo://src/lumux/__main__.py
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Application Shell and Startup

Lumux's application shell is `LumuxApp` in `src/lumux/__main__.py`, a subclass of `Adw.Application` (GTK4 / libadwaita) with application ID `io.github.enginkirmaci.lumux`. It is a thin orchestration layer: it owns the `AppContext` dependency container, registers the tray menu actions, handles Unix signals, and drives the startup sequence (settings → AppContext → bridge bootstrap → main window → optional reading-mode auto-activation). `main()` constructs the app and calls `app.run(None)`.

## Startup sequence

`on_activate` runs the full bootstrap, in order:

1. If a main window already exists, re-present it and return (single-instance guard).
2. Force the Adwaita dark color scheme via `Adw.StyleManager`.
3. Set up the application icon: `APP_ICON_PATH` resolves two directory levels above `src/lumux/` (i.e. the project root) to `io.github.enginkirmaci.lumux.svg` and is only registered with the icon theme if the file exists there.
4. Create the `SettingsManager` singleton, construct `AppContext(settings)`, and call `app_context.start()`, which returns a `BridgeStatus` after attempting the bridge connection. A warning is printed when the bridge is unreachable (the user is pointed at Settings).
5. Construct `MainWindow(self, self.app_context)`.
6. Present the window, or hide it immediately when `settings.ui.minimize_at_startup` is set and a tray icon is available.
7. When `settings.reading_mode.auto_activate_on_startup` is enabled, schedule `_auto_activate_reading_mode` — 500 ms after activation if the bridge is already connected, 2000 ms otherwise (the comment notes the network may still be initializing at system startup).

## Reading-mode auto-activation

`_auto_activate_reading_mode` is a GLib timeout callback that retries every 1 second, up to 20 times. Each attempt: skips if reading mode is already active; ensures the bridge is connected via `bridge.test_connection()` / `bridge.connect()`; then calls `mode_manager.switch_to_reading()`. The retry chain stops on success, after 20 attempts, or on exception. Note that the HTTP calls run on the GTK main thread — this is one of the known instability sources catalogued in `STABILITY_TODOS.md`.

## Shutdown

- `LumuxApp.__init__` installs Python `signal.signal` handlers for `SIGTERM` and `SIGINT`. The handler prints the signal number and defers `_quit_safely` to the GLib main loop via `GLib.idle_add`. `_quit_safely` sets `main_window._quitting = True` before `self.quit()` so the window's close handler does not treat a real shutdown as minimize-to-tray.
- `do_shutdown` (GTK application teardown) calls `app_context.shutdown()`, which calls `mode_manager.turn_off(turn_off_lights=True)` — turning the lights off on clean exit — or, if no `ModeManager` exists, stops the sync controller and the entertainment stream.
- The tray `quit` action does the same thing directly (`_quitting = True`, `self.quit()`).

## AppContext: the composition root

`AppContext` (`src/lumux/app_context.py`) is the hand-rolled dependency container that every subsystem is wired through. `__init__(settings)` constructs, in order:

- `HueBridge(bridge_ip, app_key)`
- `ScreenCapture(scale_factor, black_bar_settings, source_type, restore_token)`, with `set_on_restore_token_changed(self._on_restore_token_changed)` registered on it
- `ZoneProcessor(settings=settings.zones)`
- `ColorAnalyzer(brightness_scale, gamma)`
- `settings.get_zone_mapping()`
- An optional `EntertainmentStream` — created only when both `client_key` and `entertainment_config_id` are configured
- `SyncController(bridge, capture, zone_processor, color_analyzer, zone_mapping, settings.sync, entertainment_stream)`
- `ModeManager(bridge, sync_controller, entertainment_stream, reading_mode, entertainment_config_id)`

Finally it wires `sync_controller.set_on_stop_callback(mode_manager.on_video_sync_stopped)`, so that a stopped video sync lets the `ModeManager` react (e.g. auto-activate reading mode), and it creates a `threading.Lock` that guards live reconfiguration.

Key methods:

- `start()` returns `get_bridge_status(attempt_connect=True)`.
- `start_entertainment()` / `stop_entertainment()` gate DTLS streaming on the stream being configured and the bridge being reachable.
- `apply_settings()` reconfigures live components under the lock: it updates `HueBridge` credentials (and resets the cached `hue`/`bridge` API handles when the IP or key changed), updates capture scale factor, source type, and black-bar settings, updates the color analyzer's brightness scale and gamma, updates the zone grid rows/cols and calls `_rebuild_zone_ids()`, recreates the `EntertainmentStream` when the entertainment config changed (disconnecting the old one first) and points both the sync controller and the mode manager at it, and refreshes the mode manager's reading settings. `MainWindow` calls this after the settings dialog closes.
- `_on_restore_token_changed(token)` persists the portal restore token into `settings.capture.restore_token` and saves `settings.json` so screen-share consent survives app restarts (see also [Settings and Persistent State](../configuration/settings-and-state.md)).
- `get_bridge_status(attempt_connect)` builds the frozen `BridgeStatus` dataclass (`connected`, `configured`, `bridge_ip`, entertainment zone name, channel count, `entertainment_connected`), attempting a connection when requested. `get_bridge_status_async(callback, attempt_connect)` runs the same work on a daemon thread and delivers the result to the GLib main loop via `GLib.idle_add`, falling back to a direct callback if GLib is unavailable.

## Tray actions

`_setup_actions` registers three application-level `Gio.SimpleAction`s that back the tray menu: `show` (present the window), `toggle-sync` (forwards to `main_window._on_sync_toggle(None)`), and `quit`. The tray icon implementation itself (StatusNotifierWatcher / KStatusNotifier D-Bus) lives in `gui/tray_icon.py` — see [GTK4 Interface](../gui/gtk4-interface.md).

## Cross-references

- Per-frame data path (capture → zones → colors → sync): [Architecture Overview](overview.md) and the [pipeline pages](../pipeline/).
- What each mode transition does: [Video and Reading Modes](../modes/video-reading-modes.md).
- Persistent state written by the shell: [Settings and Persistent State](../configuration/settings-and-state.md).
- Known startup/shutdown instability classes: [Stability and Troubleshooting](../operations/stability-and-troubleshooting.md).
