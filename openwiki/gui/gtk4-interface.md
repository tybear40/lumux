---
type: "Reference"
title: "GTK4 Interface"
description: "The Adwaita GUI layer: MainWindow (status polling, mode toggle, reading controls, minimize-to-tray), the six-page SettingsDialog, the three-step BridgeWizard, the separate-process system tray, and the cairo ZonePreviewWidget."
tags: [gui, gtk4, adwaita, main-window, settings, bridge-wizard, tray, zone-preview]
openwiki_generated: true
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-500d9115afc4c6ff7d1792f2
    resource: repo://src/lumux/gui/bridge_wizard.py
  - id: openwiki-source-aea183e9f095e7c30102169f
    resource: repo://src/lumux/gui/main_window.py
  - id: openwiki-source-f1f362d59f63cd8185be1d88
    resource: repo://src/lumux/gui/settings_dialog.py
  - id: openwiki-source-72bdd5409ae1f370a8c5a17b
    resource: repo://src/lumux/gui/tray_icon.py
  - id: openwiki-source-1b0d8077c8ecf5ae51c16952
    resource: repo://src/lumux/gui/zone_preview_widget.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

<!-- openwiki: mermaid parse failed and this diagram was converted to a text fence so it does not break rendering. Fix the diagram source and restore the mermaid fence. Parser error: Heuristic: an unescaped angle bracket inside a label breaks rendering; rephrase the label. -->
```text
flowchart LR
    App[LumuxApp<br/>Adw.Application] --> AC[AppContext]
    App --> MW[MainWindow<br/>Adw.ApplicationWindow]
    MW --> ZPW[ZonePreviewWidget]
    MW --> TI[TrayIcon]
    TI -. JSON lines over stdin/stdout .-> TP[tray child process<br/>SNI / AppIndicator]
    MW --> SD[SettingsDialog<br/>Adw.PreferencesDialog]
    SD --> BW[BridgeWizard<br/>Adw.NavigationPage]
    MW -- 100 ms status poll --> SC[SyncController]
    MW -- mode switching --> MM[ModeManager]
    MW -- read / write --> SET[SettingsManager]
    BW -- discovery + auth --> HB[HueBridge]
```

# Architecture

The GUI layer is five widget classes, all in `src/lumux/gui/`, and the only layer of the app that touches GTK. Everything except the tray child process is GTK4 + Adwaita (`gi.repository: Gtk, Adw, Gdk, GdkPixbuf, Pango`); the tray runs in a **separate process** specifically to allow a GTK3-based AppIndicator fallback without forcing GTK3 into the main application runtime.

The layer is deliberately decoupled from the capture and lighting core. The widgets talk to the rest of the system only through `AppContext` (which hands them `SyncController`, `ModeManager`, `SettingsManager`, and the `HueBridge`) plus the pure `rgb_to_xy` / `xy_to_rgb` color utilities. No GUI file imports `ScreenCapture`, `EntertainmentStream`, or `SyncController`'s internals directly — the main window drives `ModeManager` and polls `SyncController` status/stats methods.

A recurring pattern: any blocking network or discovery work (wizard discovery, authentication, zone loading) runs in a daemon `threading.Thread`, and results are marshaled back to the GTK main loop with `GLib.idle_add`. Periodic UI updates use GLib sources (a 100 ms status poll, a 150 ms brightness debounce) instead of timers in worker threads.

The classes:

- **`MainWindow`** — the `Adw.ApplicationWindow` with status card, mode toggle, zone preview, sync/reading controls, and tray/minimize integration.
- **`SettingsDialog`** — `Adw.PreferencesDialog` with six pages covering every settings group; also hosts the wizard.
- **`BridgeWizard`** — three-step `Adw.NavigationPage` for first-time bridge setup.
- **`TrayIcon`** — parent-side handle for the separate-process system tray.
- **`ZonePreviewWidget`** — cairo-drawn live visualization of the zone colors.

# MainWindow

`MainWindow` (884 lines, the largest GUI file) is an `Adw.ApplicationWindow` built around an `Adw.ToolbarView` with a strict-centering `Adw.HeaderBar`, whose content is an `Adw.Clamp` (max width 800, tightening threshold 600) wrapping a vertical `Gtk.Box`.

## Window shell and sizing

The window size adapts to mode: `800x700` when the zone preview is shown or reading mode is active, and a compact `500x500` when the preview is off. The header carries two flat symbolic buttons — Settings (`emblem-system-symbolic`) and About (`help-about-symbolic`).

## Status card and polling

The status card is a `Gtk.Box` styled by one of four CSS classes applied by `_update_status_card` — `status-disconnected`, `status-connected`, `status-syncing`, `status-reading` — each with its own background gradient/border and icon (`network-offline`, `network-transmit-receive`, `emblem-synchronizing`, `weather-clear-night`).

Updates arrive through a 100 ms `GLib.timeout_add` loop (`_update_status`), which drains the `SyncController` status queue, then updates the label/subtitle, card class, sync button state, the zone preview colors, and the FPS stat from `SyncController.get_stats()`. This is the GUI's single heartbeat; there is no per-frame UI update path.

## Bridge status

On startup and after every settings-dialog close, the window calls `app_context.get_bridge_status_async(callback, attempt_connect=True)`. The result drives the card (Connected / Disconnected / Not Configured), shows or hides an "Open Bridge Settings" helper button, and enables or disables the sync button and both mode toggles.

## Mode toggle

Two grouped `Gtk.ToggleButton`s (Video Mode / Reading Mode) switch the UI: reading mode hides the video controls, preview, and stats and shows the reading controls; switching also resizes the window. The `ModeManager` callback `_on_mode_changed` keeps the toggles, status text, and card in sync when the core changes mode programmatically (e.g. reading auto-activation).

## Sync button

The pill-shaped Start/Stop sync button swaps icon/label and CSS class (`suggested-action` ↔ `destructive-action`) and mirrors state to the tray. Starting requires a connected bridge and calls `ModeManager.switch_to_video()`; stopping honors the reading-mode "auto-activate on stop" setting, otherwise it turns the lights off via `ModeManager.turn_off`. When the "minimize to tray on sync" setting is on, the window hides instead of staying visible while syncing and re-presents when sync stops.

## Reading controls

The reading-mode panel has:

- **Quick presets** — 8 round swatch buttons (warm white, cool white, daylight, candle, sunset, relax, focus, reading), each a `Gtk.Box` colored by a per-widget `Gtk.CssProvider`; the active preset gets an accent ring.
- **Custom color** — a `Gtk.ColorDialogButton` seeded from the stored CIE xy via `xy_to_rgb`.
- **Brightness** — a 0–254 `Gtk.Scale` whose value label updates instantly while the apply is debounced 150 ms.

Any change is applied immediately: `_apply_reading_settings` converts the picked RGB to CIE xy, writes and saves the settings, then either switches into reading mode or updates the already-active reading lights.

## Tray and window lifecycle

The window owns a `TrayIcon` instance and intercepts `notify::minimized`: a window-manager minimize becomes a `hide()` when a tray is available. `do_close_request` likewise hides to the tray rather than quitting; a real quit (via the tray Quit item or app termination) routes through `ModeManager.turn_off`, removes the GLib sources, and destroys the tray process.

## About

The About dialog is an `Adw.AboutDialog` with the app's icon, version 0.6.2, MIT license type, and a description of the real-time screen-to-lights pipeline.

# SettingsDialog

`SettingsDialog` is an `Adw.PreferencesDialog` with search enabled and six `Adw.PreferencesPage`s. Every value is written back to `SettingsManager` on close (`_on_closed` → `_save_settings` → `settings.save()`), and the parent main window refreshes its UI, reapplies settings, and re-checks the bridge afterwards.

## Bridge page

Connection status row (live icon/subtitle), bridge IP `EntryRow`, app/client key `PasswordEntryRow`s, a "Setup Bridge" button that presents the `BridgeWizard`, and an entertainment-zone `ComboRow` (loaded from the bridge, with a refresh button).

## General page

- **Start at Login** — toggling calls `SettingsManager.enable_autostart()` / `disable_autostart()` immediately; if enabling fails inside a sandbox it shows a dialog with the exact `flatpak override --user --filesystem=host io.github.enginkirmaci.lumux` command to run.
- **Minimize to Tray on Sync** and **Minimize at Startup** — both persisted to the UI settings.

## Capture page

Capture source combo, resolution scale spinner (0.01–1.0), and the black-bar detection group (enable switch, luminance threshold 0–50, detection rate 1–120).

## Zones page

Preview visibility switch plus edge rows/columns spinners (1–64).

## Sync page

Target FPS (1–60), transition time (0–1000 ms), and the color adjustment group: brightness scale (0–2.0), gamma (0.1–3.0), smoothing (0.1–1.0).

## Reading page

Default color picker, default brightness (0–254), and auto-activate-on-stop / auto-activate-on-startup switches.

# BridgeWizard

`BridgeWizard` is an `Adw.NavigationPage` containing a `Gtk.Stack` (slide left/right) of three `Adw.PreferencesPage` steps, with a single header Next/Finish button whose enabled state and label are recomputed per step: step 1 needs a bridge IP, step 2 needs an app key, step 3 needs a selected zone.

## Step 1 — Find Bridge

A status row with spinner, a Discover button that runs bridge discovery in a worker thread (SSDP/mDNS/cloud, per the row description), a `ComboRow` of discovered bridges, and a manual IP `EntryRow` — selecting either sets `bridge_ip`.

## Step 2 — Connect

Displays the chosen IP and a status row, plus an Authenticate button. Authentication runs in a worker thread against a temporary `HueBridge` (`create_user` with 3 retries, 10 s timeout — the user presses the physical link button on the bridge); on success the app/client keys appear in read-only `PasswordEntryRow`s and the temp bridge connects.

## Step 3 — Select Entertainment Zone

Loads entertainment configs from the bridge in a worker thread into a `ComboRow`, with a refresh button and a zone info group showing name and channel count.

Finishing emits `finished` and calls the callback with `(bridge_ip, app_key, client_key, entertainment_config_id)`; canceling emits `cancelled`. The public `get_bridge_settings()` / `set_bridge_settings()` methods round-trip the wizard state, which is how the Settings dialog pre-fills it and persists it.

# TrayIcon

GTK4 applications cannot host the legacy GTKStatusIcon, so the tray lives in a child Python process while `TrayIcon` in the main process manages it. This also lets the app pick a GTK3 AppIndicator backend without importing GTK3 into the main runtime.

## Backend detection

`_detect_tray_backend` spawns a subprocess that probes, in order: (1) a D-Bus `StatusNotifierWatcher` on the session bus (`IsStatusNotifierHostRegistered` under either the KDE or freedesktop interface name) → `sni` via `pydbus`; (2) inside Flatpak, `sni` if `pydbus` imports (the manifest ships it and grants the watcher talk-names); (3) `AyatanaAppIndicator3`; (4) `AppIndicator3`; otherwise no backend. When none is found the parent logs a hint (install a tray applet or the ayatana indicator) and runs without a tray — the UI feature is optional.

## IPC

Commands flow as JSON lines. The parent's `_send_to_tray` writes to the child's stdin; a daemon listener thread in the parent reads the child's stdout and dispatches on the main loop via `GLib.idle_add` to the window's handlers. The menu offers Show, Start/Stop Sync (label tracks sync state via `update_sync_status`), Settings, and Quit.

## SNI implementation

The generated SNI script (the primary backend) implements the full `StatusNotifierItem` and `com.canonical.dbusmenu` D-Bus interfaces with `pydbus`: it publishes bus name `io.github.enginkirmaci.lumux.Tray`, generates a 22×22 ARGB icon pixmap from the app SVG via `GdkPixbuf`, registers with whichever StatusNotifierWatcher is present, and updates the menu layout + emits `LayoutUpdated` when sync state changes. The AppIndicator backend generates an equivalent script around the GTK3 indicator library, reusing the installed SVG when present.

# ZonePreviewWidget

`ZonePreviewWidget` is a `Gtk.DrawingArea` that paints the current zone colors with cairo: a central "screen" rectangle (with a dark bezel and diagonal sheen) surrounded by the top, bottom, left, and right edge strips, each subdivided into `rows`/`cols` cells sized to the allocated area. Each cell is filled with a vertical linear gradient from 1.2× to 0.85× the zone brightness, plus a faint outline, for a soft LED look.

`set_layout` adapts the grid (used when zone rows/cols change in settings), and `update_colors(colors)` maps an `n×3` color array onto the edge zones in order — top, bottom, left, right — skipping the redraw entirely when the rounded color values are unchanged, which keeps the 100 ms status poll cheap while idle.

# Sources

- `src/lumux/gui/main_window.py` — window shell, status card, mode/reading controls, tray/minimize lifecycle.
- `src/lumux/gui/settings_dialog.py` — six-page preferences dialog, save-on-close, autostart with Flatpak fallback.
- `src/lumux/gui/bridge_wizard.py` — three-step discovery/authentication/zone wizard.
- `src/lumux/gui/tray_icon.py` — separate-process tray with SNI and AppIndicator backends.
- `src/lumux/gui/zone_preview_widget.py` — cairo zone color preview.
