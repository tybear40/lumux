# Lumux Stability Investigation — TODO for the Fix Agent

Investigated 2026-11-14 (v0.6.2, branch `main`). Scope: all 25 Python files in
`src/lumux/`, plus `pyproject.toml`, Flatpak manifest, and data files.
User report: app is **unstable on a CachyOS (Arch-based) Wayland desktop**.

Legend:
- **[P0]** hangs / crashes / data loss — most likely root causes of "unstable"
- **[P1]** functional bugs that degrade reliability
- **[P2]** robustness / cleanup
- **✅** = reproduced headlessly during this investigation (Python 3.14, gi/numpy/PIL available)
- Items marked *(verify on desktop)* could not be exercised without a display,
  portal, or Hue bridge — confirm on the user's machine before/while fixing.

**Big picture:** the app mixes a background sync thread, GStreamer streaming
threads, a portal D-Bus connection, a `requests` session, an `openssl`
subprocess, and GTK's main loop. Almost every instability class below comes from
**blocking or thread-affine work being done on the wrong thread**. The single
highest-leverage fix set is items 1–4 (all "unblock the main thread + fix thread
affinity").

---

## P0 — Freezes, crashes, data loss

### 1. [P0] Blocking HTTP on the GTK main thread → 25–30 s freezes → GApplication D-Bus timeout

`BridgeClient` (bridge_client.py:44) uses a 5 s per-request timeout. The
following all run **synchronously on the GTK main thread**:

- `__main__.py:103-106` — `_auto_activate_reading_mode`: `bridge.test_connection()`
  + `bridge.connect()` (→ `refresh_devices()` = 4 HTTP calls: lights, devices,
  entertainment configs, zones). On a dead bridge: ~4 × 5 s = **20 s freeze at startup**.
  This fires exactly when the user starts the desktop (network still initializing —
  the retry logic at __main__.py:83-129 was added for this, but the calls block the loop).
- `mode_manager.py:111-118` — `switch_to_video()`: REST activate + DTLS connect
  (`entertainment.py` `_activate_streaming` sleeps 0.5 s + `_wait_for_handshake` sleeps 0.8 s,
  plus 3–4 HTTP calls) on every "Start Sync" click.
- `mode_manager.py:188-209` — `_finish_switch_to_reading()` (fires from a GLib timeout
  on the main thread): `ReadingModeController.activate()` (reading_mode.py:43-103)
  = `refresh_devices()` (4 HTTP) + **one PUT per light** (5 s each if bridge is slow).
- `gui/settings_dialog.py:441-451` — `_load_entertainment_configs()` is called from
  `__init__` (line 97): `test_connection()` + `get_entertainment_configurations()`
  → **up to ~10 s freeze every time Settings is opened**.
- `gui/bridge_wizard.py:434` — `_on_auth_complete`: `_temp_bridge.connect()`
  (= 4 HTTP calls, up to 20 s) on the main thread right after successful auth.
- `mode_manager.py:225-266` — `turn_off()`: HTTP deactivation + per-light PUTs.

Why it matters on a desktop: `LumuxApp` is a `Gio.Application` with a D-Bus name.
Blocking the main loop for ≥25 s lets the bus consider the name unresponsive —
worst case the app aborts / loses single-instance registration; at minimum GNOME
marks it unresponsive and users kill it (→ SIGKILL → item 6).

**Fix (confident):** move every bridge/DTLS operation off the main thread.
A `get_bridge_status_async` pattern already exists (app_context.py:210-238) —
generalize it: `switch_to_video`, `switch_to_reading` completion,
`turn_off`, settings-dialog zone loading, wizard `connect()` should all run in
worker threads and hop to the main thread via `GLib.idle_add` (which is already
the established pattern in this codebase). Add a spinner/subtitle state while
pending. Keep `switch_to_reading`'s 1 s delay semantics but do the REST work off-loop.

### 2. [P0] pydbus portal connection used from the wrong thread *(verify on desktop)*

- `capture.py:239-391` `_setup_portal_session()` creates `pydbus.SessionBus()`
  **on the calling thread** — that is the `SyncLoop` worker thread (called from
  `capture()` in `sync.py:260`). pydbus is explicitly **not thread-safe**; the
  connection binds to the creating thread's GLib main context.
- `sync.py:181-195` `stop()` (called from the **main** thread) →
  `capture.stop_pipeline()` → `capture.py:710-723` `_close_portal_session()` →
  `session.Close()` — a **synchronous pydbus call on the main thread** against a
  connection owned by the sync thread.

Expected symptom: on Stop Sync / window close / app quit, a D-Bus reply is
dispatched on a context that is never iterated → the call hangs ~25 s (D-Bus
default timeout) before raising, or behaves as undefined (pydbus docs warn about
cross-thread use). Either way the window freezes or the session is never actually
closed.

**Fix (confident in direction):** give the portal its own single worker thread
with a running `GLib.MainLoop`; all session operations (`CreateSession`,
`SelectSources`, `Start`, `Close`) are queued to that thread via a
`queue.Queue` and results are delivered back via callbacks /
`GLib.idle_add`. `capture()` and `stop_pipeline()` only enqueue. This also
fixes item 3's dispatch problem for portal-side events.

Secondary issues in the same function:
- `capture.py:312` — the 30 s `GLib.timeout_add_seconds(30, loop.quit)` only
  guards `CreateSession`; the `SelectSources` (340-343) and `Start` (356-359)
  `loop.run()` calls have **no timeout** → infinite block if the portal never
  responds. Add a timeout to each.
- Each re-setup (stale-token retry at capture.py:184) creates a **new**
  `SessionBus` without closing the old one — leak. Close/replace explicitly.

### 3. [P0] GStreamer bus signals are never dispatched → capture death is never detected

`capture.py:501-505`: `bus.add_signal_watch()` + `bus.connect("message::error",
self._on_pipeline_error)` are invoked from whichever thread called
`_start_pipeline()` — the **SyncLoop** thread (`sync.py:260` → `capture()`).
A bus signal watch dispatches on the *calling thread's* default main context,
and the SyncLoop thread is a plain `while … time.sleep()` loop that **never
iterates a GLib context** (sync.py:208-253).

Consequences:
- `_on_pipeline_error` (capture.py:552) never fires → the converter-fallback
  machinery (`_runtime_failed_configs`, capture.py:59) is **dead code**.
- When the portal kills the stream (user revokes screen-share permission in
  GNOME Settings, monitor sleep/wake, cable unplug, resolution change not
  caught), `pipewiresrc` emits EOS/error, no new samples arrive, and
  `capture()` (capture.py:156-202) **keeps returning the last frame forever**.
  Lights freeze at the last color; the UI shows "Syncing"; nothing recovers
  until app restart. This matches the classic "stops working after a while on
  my desktop" report.

**Fix (confident):** one of:
  (a) add the bus signal watch on the **main** thread (create the pipeline
  there, or re-attach the watch after start) — simplest;
  (b) drain the bus in the sync loop with `bus.timed_pop_filtered(0, None)`;
  (c) move pipeline start + watch into the portal worker thread (item 2) that
  runs a real loop.
Additionally add **frame-staleness detection**: record the timestamp of the
last new sample in `_on_new_sample`; if `capture()` sees no new frame for
N ms (e.g., 3 s), treat the pipeline as dead → `_restart_pipeline()` and, if
the portal node is gone, re-run `_setup_portal_session()`.

### 4. [P0] Pipeline start/stop races between main and sync threads

`apply_settings()` (app_context.py:112-158, called from the main thread on
settings close) → `capture.update_scale_factor()` (capture.py:83-93) →
`_restart_pipeline()` — **at the same time** the SyncLoop thread is inside
`capture()` → `_start_pipeline()`. Both mutate `self._pipeline`,
`self._appsink`, `self._pipeline_running`, `_active_config_key` with **no lock**,
and call `Gst.Pipeline.set_state()` from two threads concurrently. GStreamer
pipeline state changes from arbitrary threads is unsupported → intermittent
crashes/segfaults or corrupted state (and `_stop_gst_pipeline` at
capture.py:95-106 sets `self._latest_frame = None` unsynchronized with
`_on_new_sample`, which runs on the GStreamer streaming thread and takes
`_frame_lock` only around the frame — the stop path does **not** take the lock).

**Fix (confident):** serialize *all* pipeline lifecycle operations
(`_stop_gst_pipeline`, `_start_pipeline`, `_restart_pipeline`, `stop_pipeline`)
on one thread — the portal worker thread from item 2 (or a dedicated pipeline
thread) — with a lock; the sync loop only ever *requests* a restart
(`self._needs_pipeline_restart = True`, which it already does) and
`update_scale_factor` just updates the factor + flags.

### 5. [P0] ✅ ModeManager race: `turn_off()` within 1 s of `switch_to_reading()` re-lights the room

`mode_manager.py:176-179`: when entertainment was connected, `switch_to_reading`
sets `_reading_activation_pending = True` and schedules
`GLib.timeout_add(1000, self._finish_switch_to_reading, …)` then returns.
`turn_off()` (mode_manager.py:241-243) clears the *flag* but **cannot cancel the
already-scheduled timeout**. `_finish_switch_to_reading` (188-223) re-checks
`current_mode != Mode.OFF` to cancel — but `turn_off()` has *already set the mode
to OFF*, so the check passes and reading mode activates one second later.

**Reproduced headlessly** (fakes for bridge/sync/entertainment, real GLib loop):

```
mode right after turn_off: Mode.OFF
mode 1.5s later: Mode.READING        <- light came back on after explicit "off"
```

(Also hits on app shutdown: `do_shutdown` → `turn_off` while a switch is pending;
and `LumuxApp._auto_activate_reading_mode` + a fast user Stop within the same second.)

**Fix (confident):** store the GLib source id and `GLib.source_remove()` it in
`turn_off()` and `switch_to_video()` instead of (or in addition to) the bool flag:

```python
self._reading_pending_source = GLib.timeout_add(1000, …)
# in turn_off / switch_to_video:
if self._reading_pending_source:
    GLib.source_remove(self._reading_pending_source)
    self._reading_pending_source = None
    self._reading_activation_pending = False
```

### 6. [P0] ✅ Settings: non-atomic write + all-or-nothing strict load → silent loss of user config

- `settings_manager.py:197-213` `save()` writes `settings.json` **in place**.
  SIGKILL mid-write (desktop logout kill, crash) → truncated/corrupt JSON.
  Corrupt file is caught at load (line 190), but the in-memory object may then
  be a **mix of defaults and partially-loaded sections**, and the next `save()`
  — which happens on **every sync start** via the restore-token callback
  (app_context.py:160-176) — **persists the reset config**, silently deleting
  the user's bridge IP / app key / client key.
- Loading a file written by a *newer* version of the app fails entirely:
  `HueSettings(**data["hue"])` raises `TypeError` on any unknown key, and the
  `except (json.JSONDecodeError, TypeError)` at line 190 discards **everything**
  (falls back to defaults).

**Reproduced:** a file containing one extra key (`"NEWFIELD": 1`) in `hue`
caused the whole file to be dropped — `bridge_ip` went back to `""`.

**Fix (confident):**
1. Atomic save: write to `settings.json.tmp` in the same dir, `f.flush()` +
   `os.fsync`, then `os.replace(tmp, settings.json)`; keep the previous file as
   `settings.json.bak` and recover from it if the primary is unreadable.
2. Tolerant load: filter each section's keys down to the dataclass fields
   (e.g. `dataclass_fields(cls)`) instead of blindly `**`-unpacking, so schema
   additions/changes don't nuke the file.
3. (Optional) add a `schema_version` key.

---

## P1 — Functional reliability bugs

### 7. [P1] ✅ `EntertainmentStream` message buffer assumes a 36-char config ID → protocol corruption / 30 Hz error spam

`entertainment.py:205-220` `_init_message_buffer()`:
`buf[16:52] = self._encoded_config_id` **resizes** the `bytearray` to whatever
the config ID is, but `MESSAGE_HEADER_SIZE` is the constant 52 and all channel
data is packed from offset 52. With a non-UUID ID the buffer is the wrong size
and offsets are misaligned.

**Reproduced** with config id `"default"` (7 chars): buffer ends up 37 bytes,
and `_build_xy_message()` raises
`struct.error: pack_into requires a buffer of at least 59 bytes … (actual buffer size is 37)`.

Impact when triggered: `send_colors_xy` catches it, sets `_connected = False`
and **prints an error on every frame (30×/s) forever** — log spam + sync
silently dead until sync restart. (Hue v2 zone IDs are usually 36-char UUIDs,
so this is latent — but "default"-style IDs exist on some bridges, and the
failure mode is a silent total stop.)

**Fix (confident):** allocate the config-id slot fixed:
`buf[16:52] = self._encoded_config_id.ljust(36, b"\x00")[:36]` (pad/truncate to
exactly 36), and `assert len(self._message_buffer) ==
MESSAGE_HEADER_SIZE + CHANNEL_DATA_SIZE * n` after building. Re-init the buffer
whenever `entertainment_config_id` changes, not only in `connect()`.

### 8. [P1] `HueBridge.create_user()` ignores `max_retries` / `timeout` — docstring promises retries that don't exist

`hue_bridge.py:59-84`: signature is
`create_user(self, bridge_ip, application_name="lumux", max_retries=3, timeout=10.0)`,
docstring says "Retries with exponential backoff if the link button hasn't been
pressed yet" — but the body makes a **single** call to
`BridgeClient.create_user(bridge_ip, application_name)` which hardcodes
`timeout=5` and has **no retry loop** (bridge_client.py:342-383).

The wizard (bridge_wizard.py:409-411) passes `max_retries=3, timeout=10.0` and
gets neither. So if the user clicks *Authenticate* a moment before pressing the
link button, it fails once and demands a full re-click.

**Fix (confident):** implement the loop: retry while the bridge answers with
the "link button not pressed" error (Hue v1 returns 401 with error code 101 in
the `error` object) up to `max_retries` (or a wall-clock window, e.g. 30 s),
backoff 1→2→4 s, passing `timeout` through to `BridgeClient.create_user`.
Update `BridgeClient.create_user` to accept and use `timeout`.

### 9. [P1] App/tray icon paths point at files that don't exist

- `__main__.py:17-20`: `APP_ICON_PATH` = `<repo root>/io.github.enginkirmaci.lumux.svg`
  — the icon actually lives in `flatpak/io.github.enginkirmaci.lumux.svg`
  (see `pyproject.toml` data-files and the flatpak manifest install command).
  `os.path.exists()` fails → icon silently never loads (swallowed by except).
- `tray_icon.py:41-46`: `_get_icon_path()` dev path =
  `src/lumux/io.github.enginkirmaci.lumux.svg` (one `dirname` too few — it
  lands in `src/lumux/`, not the repo root, and the file is in `flatpak/`
  anyway). The SNI child then fails pixbuf load → **empty `IconPixmap`** →
  blank/broken tray icon in dev; the fallback `IconName` is not in any icon
  theme, so the panel shows nothing.

**Fix (confident):** single source of truth for the icon path, e.g.
`Path(__file__).resolve().parents[2] / "flatpak" / "io.github.enginkirmaci.lumux.svg"`
for dev, `/app/share/icons/...` for flatpak (tray already handles the flatpak
case); use it in `__main__.py`, `main_window.py:17-20`, and `tray_icon.py`.

### 10. [P1] DTLS via `openssl s_client`: undrained pipes, zombie process, no auto-reconnect

`entertainment.py:255-324` spawns `openssl s_client` with `stdout=PIPE,
stderr=PIPE` and **never drains either** (only `stdin.write` in
`_send_via_openssl`, 405-413). Issues:

- If the bridge (or a middlebox) ever writes to the client, or openssl emits
  handshake noise past `-quiet`, the 64 KB pipe fills and **openssl blocks on
  write → DTLS stalls** (lights freeze even though the process is alive).
- When the bridge closes the DTLS association, `stdin.write` raises
  `BrokenPipeError` → `_connected = False`, but the process is **not reaped or
  replaced** (zombie until `disconnect()`), and the sync loop has **no
  reconnection logic** — video sync just silently does nothing until the user
  restarts sync.
- The "handshake succeeded" check (`_wait_for_handshake`) is only "process has
  not exited after 0.8 s" — a connection that fails *slowly* (TLS records
  accepted then dropped) reads as success.
- `connect()` can be re-entered while a previous `_openssl_proc` still exists
  (no cleanup of a prior proc before spawning the new one — `_establish_dtls_connection`
  at 255-281 pops a new Popen unconditionally).

**Fix (confident):** drain `stdout`/`stderr` in a daemon thread (or at minimum
`stderr=subprocess.DEVNULL` + a reader for stdout); in the sync loop, if
`is_connected()` flips False, schedule a reconnect (with backoff) via
`entertainment_stream.reconnect(bridge)` — reuse `connect()` after a
`_cleanup_openssl()`; ensure `connect()` cleans up any existing proc first;
reap the process on death. Longer term: consider a native DTLS client
(`ssl.SSLContext(ssl.DTLS_CLIENT)` + `socket.SOCK_DGRAM` — the `_dtls_socket`
hook at line 90 already anticipates this) to remove the subprocess entirely.

### 11. [P1] `requests.Session` shared across threads

`HueBridge.client` (hue_bridge.py:40-45) lazily creates **one** `BridgeClient`
→ **one** `requests.Session` (bridge_client.py:55). It is used concurrently
from: the main thread (status checks, wizard post-auth connect, reading-mode
PUTs, `turn_off`), the `get_bridge_status_async` worker thread
(app_context.py:218-238), and wizard worker threads. `requests`/`urllib3`
sessions are **not thread-safe**; concurrent use can corrupt the connection
pool → sporadic `ConnectionError`s, "Cannot connect to bridge" flicker in the
UI, intermittent timeouts. Classic source of hard-to-reproduce instability.

**Fix (confident):** either a thread-local session (weak-keyed cache per
`threading.get_ident()` inside `BridgeClient`), or serialize all client calls
behind a single lock (simpler; bridge calls are infrequent), or one worker
thread owning the session. Also: `get_application_id` (bridge_client.py:131-155)
uses a *separate* `urllib` path with a 5 s timeout — fine, keep, but route it
through the same timeout settings.

### 12. [P1] `apply_settings()` leaves stale `EntertainmentStream` in both consumers when entertainment is disabled

`app_context.py:138-158`: the update of `sync_controller.entertainment_stream`
and `mode_manager.entertainment_stream` only happens **inside**
`if hue.client_key and hue.entertainment_config_id:`. When the user clears the
client key or config ID in Settings (to disable streaming), neither object is
reset → `ModeManager.switch_to_video()` (mode_manager.py:105-126) and the sync
loop keep using the **stale** stream object with old credentials/config.

**Fix (confident):** in the `else` branch: `stop_entertainment()` (if
connected), `self.entertainment_stream = None`,
`self.sync_controller.entertainment_stream = None`,
`self.mode_manager.entertainment_stream = None`. (Note: the current
uncommitted diff in `app_context.py` added the `mode_manager` line inside the
`if` — it needs the `else` side too, or the whole assignment should move out
of the `if`.)

### 13. [P1] `_update_status_card("ready")` is a dead state → wrong status card after Stop

`main_window.py:835` passes `"ready"`, but `_update_status_card` (525-543) only
handles `syncing` / `reading` / `connected` — everything else falls into the
`disconnected` style. After Stop (auto-activate off) the card shows the
"disconnected" look (red-ish, network-offline icon) even though the bridge is
connected. Use `"connected"`.

### 14. [P1] Reading mode activation re-runs `refresh_devices()` on every slider release

`main_window._apply_reading_settings()` (640-658) fires on every color change
and every debounced brightness release (150 ms). Each call →
`ReadingModeController.activate()` (reading_mode.py:65-69) →
`bridge.refresh_devices()` = **4 HTTP requests + `get_entertainment_light_ids`
= 2 more** (per activation, main thread — see item 1) followed by a PUT per
light. A user dragging the brightness slider triggers a full device re-fetch
per stop.

**Fix (confident):** cache device/light info with a short TTL (e.g. 30 s) in
`HueBridge.refresh_devices`, or have `activate()`/`update_color()` skip the
refresh entirely (the light ID list doesn't change while the bridge is
connected); only refresh on connection (re)establish.

### 15. [P1] `_sync_loop` error handler swallows tracebacks

`sync.py:247-251`: `except Exception as e: timed_print(f"Sync loop error: {e}")`
— no traceback. When the loop does die (e.g. item 7's `struct.error` surfacing
differently, or a numpy shape error after a monitor change), the log only says
"Sync loop error: …" with no stack — the user cannot diagnose why sync died,
and the loop **keeps running but doing nothing** (no re-arm of pipeline/DTLS).

**Fix (confident):** `import traceback; traceback.print_exc()` (or logger) in
that handler; after N consecutive errors, emit a queued status so the GUI shows
the error and (with items 3/10) triggers recovery.

---

## P2 — Robustness & cleanup

### 16. [P2] `SyncController.stop()` is not idempotent / re-entrancy

`sync.py:181-202`: two callers (main-thread button + `ModeManager.turn_off` /
`do_close_request`) can run `stop()` back-to-back or concurrently. The
`self.running` flag guards the thread, but the *body* (thread join,
`stop_pipeline()`, `_on_stop_callback`) is not lock-protected — a double stop
can double-run `_on_stop_callback` (`mode_manager.on_video_sync_stopped`) →
duplicate `switch_to_reading` / `GLib.timeout_add` scheduling.
**Fix:** wrap the body in a `threading.Lock` with an `acquired` check
(`with_contextlib.suppress`-style `if not self._stop_lock.acquire(blocking=False): return`).

### 17. [P2] Portal `SelectSources` / `Start` loops have no timeout

`capture.py:340-343, 356-359`: `loop.run()` with no watchdog (only the first
`CreateSession` loop has the 30 s timeout at line 312). If a portal backend
hangs on these calls (some portal versions hang waiting for user input with
certain option combos), the sync thread blocks **forever** — and because the
block happens *inside* `capture()` on the sync thread, the sync loop can never
even time out. Add `GLib.timeout_add(30, loop.quit)` (and `source_remove` in
the `finally`) to each of the three loops, and clean up the leaked 30 s source
from the first loop after `loop.run()` returns.

### 18. [P2] Dead code

- `config/zone_mapping.py` (`ZoneMapping`) — instantiated (app_context.py:43),
  stored on `SyncController` (sync.py:32), **never used** in the live sync path
  (DTLS channel mapping in sync.py:85-179 supersedes it; the REST-zone path in
  `HueBridge.set_zone_color` also has no callers).
- `data/default_settings.json` — installed to `share/lumux/data` by both the
  setuptools data-files (pyproject.toml:58-62) and the flatpak manifest, but
  **no code ever reads it**.
- `HueBridge.set_zone_color` (hue_bridge.py:224-246) and
  `BridgeClient.set_zone_state/set_zone_color` — no callers.
- `main_window.py:11-11` imports `rgb_to_xy` and uses it; fine, but
  `entertainment.map_zone_to_channel` / `_get_edge_position_ranges` /
  `_find_channels_for_edge` / `_channel_matches_edge` (entertainment.py:501-569)
  are superseded by `SyncController._find_best_channel_for_zone` — no callers.

**Fix:** delete (or wire in) — recommend deleting `zone_mapping` and
`default_settings.json` (remove from `pyproject.toml` data-files and the
flatpak manifest accordingly) after confirming no external consumers.

### 19. [P2] No real logging — crashes are invisible

Everything is `print`/`timed_print` to stdout; a desktop app launched from a
launcher has no persistent log, so any uncaught exception in a worker thread
(`threading.excepthook` default just prints) or a segfault leaves no trace.
**Fix:** replace `timed_print`/`print` with a `logging` setup: stderr handler
+ rotating file in `~/.local/state/lumux/lumux.log` (or `~/.config/lumux/`);
install `threading.excepthook` and a GLib exception trap so worker-thread
crashes are captured; log at startup: Python/GStreamer/portal versions,
Wayland compositor name (`XDG_CURRENT_DESKTOP`), available GStreamer elements.
Also consider `faulthandler.enable()` so SIGSEGV (GStreamer/GL) writes a Python
stack to the log — directly relevant to the item 4 race.

### 20. [P2] Dependency / packaging notes (CachyOS-specific)

- `PyGObject` and `pycairo` ship **no wheels** — `pip install -e .` on Arch
  builds them from source and needs `-devel` packages
  (`gst-libs`, `glib2-devel`, `gtk4-devel`, `libadwaita-devel`,
  `pipewire-devel`, `meson`…). The README's "dependencies install
  automatically" claim is false on distros without wheels. Document the
  Arch/CachyOS system-deps list (or ship a venv recipe). `pycairo` is
  imported directly by `zone_preview_widget.py:7` but is only a transitive
  PyGObject dep — add it to `dependencies` to be explicit.
- `zeroconf>=0.60` is unpinned: verified against current zeroconf (0.15x) that
  `ServiceInfo.parsed_addresses()` and `ServiceBrowser` are still fine, but
  the code uses the legacy sync `ServiceBrowser`/listener on a worker thread
  with `time.sleep(timeout)` (hue_bridge.py:385-434) — pin to a known-good
  major and add `browser.close()`/`zeroconf.close()` in a `finally` (currently
  leaked on the `Exception` path — `zeroconf.close()` at line 425 only runs on
  success; move to `finally`).
- `pydbus` is a hard dependency for capture on native installs; guard the
  import in `capture._setup_portal_session` so the app still runs (with a clear
  "install pydbus" message) on systems without it — currently a bare
  `import pydbus` at capture.py:241 inside the try → the whole setup fails and
  capture returns None forever with a generic "Failed to setup portal session".
- Flatpak manifest (flatpak/io.github.enginkirmaci.lumux.yml): verify the
  GNOME 50 runtime image provides `pipewiresrc` (gst-plugins-bad) and the GL
  capture elements (gst-plugins-good) — if not, add explicit modules; a missing
  `pipewiresrc` means all pipeline configs fail and capture is dead silently.
  *(verify on desktop)*

### 21. [P2] Finish the uncommitted `app_context.py` change

The working tree has an uncommitted edit (app_context.py:154-155) syncing
`mode_manager.entertainment_stream` inside `apply_settings`. It's an incomplete
version of item 12 — rework per item 12 (handle the `else` branch) rather than
committing as-is.

### 22. [P2] Misc small issues

- `__main__.py:68, 158` — `main_window._on_sync_toggle(None)` / `_quitting`:
  reaching into private members; make a public `toggle_sync()`.
- `main_window.py:35-38` — `_preview_size`/`_compact_size`/`_reading_mode_size`
  are duplicated sizes; `_apply_window_size` uses `set_default_size`, which on
  most Wayland compositers does **nothing to an already-mapped window** —
  runtime resize silently no-ops (GNOME/KDE honor only initial size). Either
  use `resize(w, h)` or accept it and document.
- `main_window.py:745-790` `_update_status` polls every 100 ms even when the
  queue is empty and nothing is running — fine, but gate it: remove the timeout
  source when not syncing to save CPU.
- `capture.py:660-665` — `buffer.map(READ)` failure is printed every frame
  ("Failed to map buffer …") → log-once pattern (like `_pipeline_error_logged`)
  to avoid 30 Hz spam when DMA-BUF-only.
- `capture.py:22-23` `Gst.init(None)` at import time — `lumux/__init__.py`
  imports `capture`, so *any* `import lumux` (including pip metadata tools or
  the tray child) initializes GStreamer. Move to first-use.
- `bridge_client.py:94` — `timeout=timeout or self.timeout`: `timeout=0`/`None`
  both work, but a caller passing `0` would *disable* the timeout semantics
  unexpectedly — normalize with `timeout if timeout else self.timeout` is the
  same; instead validate `timeout > 0`.
- `entertainment.py:405-413` — `_send_via_openssl` also swallows the case where
  the pipe *buffer* is full (write blocks the sync thread); with `drop=true`
  streaming at 30 fps a slow DTLS peer will eventually block `stdin.write` on
  the sync thread — pair with item 10's drain thread.
- `reading_mode.py:93` — `is_active` is set from `success_count > 0` (partial
  success = active); after a full network outage every light PUT fails →
  `is_active=False` but `_state` keeps stale color — fine, but then
  `is_reading_active()` (mode_manager) flips false mid-mode; the mode manager
  never notices (mode stays `READING`) → `turn_off` skips light-off because the
  controller "isn't active". Decide the semantics; recommend: activate counts
  as active if the request *reached* the bridge, track `last_put_ok` per light.
- `zone_preview_widget.py:88-96` — division by `top_count`/`left_count`:
  safe today because settings clamp to ≥1, but `set_layout()` is public; guard
  `max(1, …)`.

---

## Suggested implementation order

1. **Item 1** (async bridge I/O) — biggest UX win, independent.
2. **Item 2** (portal worker thread) — prerequisite for 3 & 4.
3. **Items 3 + 4** (bus dispatch + lifecycle serialization) together.
4. **Item 5** (mode-manager source removal) — small, do anytime.
5. **Item 6** (atomic/tolerant settings) — small, do anytime.
6. Items 7, 8, 9, 12, 13, 15 — independent small fixes.
7. **Item 10** (DTLS hardening) — medium.
8. **Item 11** (session thread-safety) — medium.
9. P2 items as they fit.

## Verification plan (on the user's CachyOS desktop)

Reproduce/confirm before and after the fix set:

1. **Startup with bridge unreachable** (airplane mode): time how long until
   the window is responsive; expect no freeze >1 s after fix; app must not
   lose its D-Bus name (check `busctl --user list | grep lumux`).
2. **Stop within 1 s of a mode switch** (item 5): Start sync → Stop quickly →
   verify lights do *not* come back on ~1 s later.
3. **Revoke the portal permission** mid-sync (GNOME Settings → Privacy →
   Screen Sharing, or kill the portal session): after fix, the app should
   detect the dead pipeline within a few seconds, show an error, and recover
   when re-allowed (items 2/3).
4. **Change resolution scale in Settings while syncing** (item 4): rapid
   open/close of the Settings dialog while sync runs — no crash, pipeline
   restarts cleanly.
5. **SIGKILL mid-save**: start a sync loop (saves the restore token each
   start), `kill -9` the app repeatedly, then check `~/.config/lumux/settings.json`
   is always valid JSON with the original values (item 6).
6. **Quit with bridge unreachable**: `do_shutdown` should complete in <2 s
   (item 1's shutdown path off the loop); no "unresponsive" window.
7. **DTLS loss**: unplug/replug or firewall the bridge's port 2100 mid-sync →
   app should log, keep preview, and reconnect without user action (item 10).
8. **Auth wizard** with link button pressed late (item 8): should succeed
   without re-clicking.
9. Check `~/.local/state/lumux/lumux.log` for the new diagnostics (item 19),
   including `faulthandler` output if a segfault recurs.
10. Regression: normal start/stop cycle, reading-mode presets, tray menu,
    black-bar toggle, window-source capture.

---

*Investigation method: full read of all 25 source files; headless import smoke
tests of `lumux` + all GUI modules; empirical reproduction of items 5, 6, 7;
cross-checked zeroconf 0.15x API compatibility (no issue found); Flatpak
manifest + data files reviewed. Items marked *(verify on desktop)* require the
user's display/portal/bridge to confirm.*
