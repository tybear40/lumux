---
type: "Reference"
title: "Video and Reading Modes"
description: "ModeManager state machine (OFF/VIDEO/READING) with mutually exclusive transitions: video drives DTLS entertainment streaming, reading sends one-time REST puts; covers the 1-second deactivation delay, the _reading_activation_pending race guard, auto-activation when video stops, and target-light resolution that never falls back to all bridge lights."
tags: [modes, mode-manager, reading-mode, video-sync, transition, race-protection]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
  - id: openwiki-source-9d1a91d40589128cb0faa21d
    resource: repo://src/lumux/mode_manager.py
  - id: openwiki-source-77a99335182aa760e79ac6da
    resource: repo://src/lumux/reading_mode.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Video and Reading Modes

Lumux operates two mutually exclusive lighting modes plus an off state:

- **Video mode** — continuous color following via the Hue Entertainment API over DTLS (see [Entertainment Streaming](../bridge/entertainment-streaming.md)).
- **Reading mode** — a static, warm static color set with one-time REST `PUT` requests; the bridge maintains the state with no further traffic.

Mutual exclusivity is a hardware/bridge constraint, not an app choice: a zone streaming via the Entertainment API must be deactivated before REST commands are honored, so every transition spends its first steps dismantling the other mode. `ModeManager` (`src/lumux/mode_manager.py`) centralizes this discipline.

## The state machine

`Mode` is a plain `Enum` — `OFF`, `VIDEO`, `READING`. `ModeManager` is constructed by [AppContext](../architecture/application-shell.md) with the `HueBridge`, `SyncController`, `EntertainmentStream`, the `ReadingModeSettings` section, and the stored `entertainment_config_id`. AppContext also wires `mode_manager.on_video_sync_stopped` as the sync controller's on-stop callback, which is what makes reading mode "just appear" when a video session ends.

## Video mode: `switch_to_reading` is the mirror image

`switch_to_video()` runs five ordered steps:

1. If in READING, deactivate the reading controller with `turn_off=False` — lights are left at their current state for a smooth handover, not snapped to black.
2. Safety stop: `sync_controller.stop()` if somehow still running.
3. Guard: no `entertainment_stream` configured → return `False`.
4. If DTLS is not connected: `bridge.activate_entertainment_streaming(config_id)` then `entertainment_stream.connect(bridge)`. A failed DTLS connect **rolls back** the activation with `deactivate_entertainment_streaming`, leaving the bridge in a clean state.
5. `sync_controller.start()`, then flip `current_mode = Mode.VIDEO` and fire the mode-changed callback.

## Reading mode: the one-second delay

`switch_to_reading(xy, brightness, _callback)` mirrors the sequence in reverse:

1. **Duplicate-call guard** — if `_reading_activation_pending` is already set, return `True` immediately. The GUI debounce and a user double-press can reach this path twice; the second call must be a no-op, not a second REST burst.
2. Defaults from `ReadingModeSettings` fill any `None` xy/brightness.
3. Stop video sync if running.
4. If the entertainment stream is connected, `disconnect(bridge)` — this deactivates the Entertainment zone on the bridge side.
5. **Only if a stream was disconnected**: set `_reading_activation_pending = True` and schedule `_finish_switch_to_reading` with `GLib.timeout_add(1000, ...)`, returning `True` to report the switch as accepted. If no stream was connected, proceed synchronously (and in non-GLIB contexts fall back to `time.sleep(0.3)`).

The 1-second delay exists because deactivation is asynchronous on the bridge: the zone must actually finish leaving Entertainment mode before the REST `PUT` color commands are accepted. The GLib timeout keeps the main loop responsive instead of sleeping a thread.

`_finish_switch_to_reading` first clears the pending flag, then performs an **interruption check**: if `current_mode != Mode.OFF` at fire time (someone called `turn_off` during the delay), it cancels the activation, calls the callback with `False`, and stops the timeout — the lights are never REST-overwritten after a user-initiated off. Otherwise it activates the reading controller and flips the mode to READING on success.

## `turn_off` and the pending-activation race

`turn_off(turn_off_lights=True)` (called by `AppContext.shutdown` and the quit path):

1. Cancels any pending reading activation (`_reading_activation_pending = False`) *before* stopping sync.
2. `sync_controller.stop()` — this synchronously fires `on_video_sync_stopped`, which may itself auto-activate reading mode (below).
3. **Auto-activation exception**: if the mode was VIDEO and is now READING (step 2 auto-switched), `turn_off` returns `True` *without* touching the lights — the user pressed stop on a video session and asked for the reading light, so the reading light stays.
4. Otherwise: disconnect the entertainment stream, deactivate the reading controller (honoring `turn_off_lights`), and notify OFF.

## Auto-activation: `on_video_sync_stopped`

Invoked by the sync controller whenever video sync stops (user Stop button, capture failure, or `turn_off`'s own step 2):

- Only acts when `current_mode == Mode.VIDEO`; otherwise returns `False`.
- Sets the mode to OFF, then checks the `auto_activate` setting in `ReadingModeSettings`. If true, it calls `switch_to_reading()` with stored defaults — the lights fade to the reading color. If false, it simply notifies the mode change to OFF.

This is the second half of the reading-mode story: [Application Shell](../architecture/application-shell.md) shows the startup side (auto-activating reading mode on launch with a retry loop), and this callback handles the session-end side.

## `ReadingModeController` (`src/lumux/reading_mode.py`)

The controller owns a `ReadingModeState` (`is_active`, `color_xy` defaulting to warm (0.5, 0.4), `brightness` defaulting to 150) and deliberately uses **one-time REST puts** — no streaming, no timers. The bridge holds the state until changed.

- **`activate(xy, brightness, transition_ms=400)`** — refreshes bridge devices, resolves target lights, then issues `bridge.set_light_color(...)` per light. Active state is set when at least one light succeeds; a partial success still returns `True` (logged as `N/M lights`), total failure returns `False`.
- **`deactivate(turn_off=False)`** — with `turn_off=False` the lights are simply left as-is (the smooth-handover case); with `True` each target light gets `{'on': {'on': False}}` via the bridge client.
- **`update_color(xy, brightness, transition_ms=200)`** — the steady-state path used while reading mode is active (GUI swatches, color picker, debounced brightness slider). If the controller is not active it only updates stored state and returns without sending.
- **`set_target_lights(ids)`** — seed explicit targets; `ModeManager.get_reading_controller()` applies `ReadingModeSettings.light_ids` on first creation.

### Target-light resolution: never all lights

`_get_target_light_ids()` is the safety-critical piece:

1. Explicit `_target_light_ids` win if non-empty.
2. Otherwise, if an `entertainment_config_id` is configured, it queries `bridge.get_entertainment_light_ids(config_id)` — the zone membership from [Entertainment Streaming](../bridge/entertainment-streaming.md).
3. Empty zone, missing config, or a failed query all yield `[]` with a logged explanation. **There is deliberately no fallback to all bridge lights** — a misconfigured reading mode dims nothing rather than re-lighting the whole room.

## Invariants and failure behavior

- The modes are a strict enumeration of bridge control channels: VIDEO ⇔ DTLS streaming, READING ⇔ one-shot REST, OFF ⇔ neither. No code path combines them.
- Every transition that moves from a streaming state to a REST state pays the deactivation delay (1 s, or 0.3 s in non-GLIB fallback); transitions without a prior stream are immediate.
- `_reading_activation_pending` is the single token that makes the async transition idempotent: it gates duplicate `switch_to_reading` calls, is cleared in `_finish_switch_to_reading`, and is cancelled by `turn_off`.
- Failures degrade to the *quieter* state: failed DTLS connect rolls back zone activation; failed reading activation leaves the mode where it was (VIDEO stays VIDEO or OFF stays OFF) and reports `False` through the callback chain.
- Mode changes are broadcast through `set_mode_changed_callback` → GUI toggle sync; the GUI never decides mode authority — it reacts.

See also: [Screen Capture](../pipeline/screen-capture.md) (feeds video mode), [Sync Engine](../pipeline/sync-engine.md) (the frame loop behind video), [Settings and Persistent State](../configuration/settings-and-state.md) (`ReadingModeSettings` schema), [GTK4 Interface](../gui/gtk4-interface.md) (mode UI).
