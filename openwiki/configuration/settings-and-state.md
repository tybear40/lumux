---
type: "Reference"
title: "Settings and Persistent State"
description: "SettingsManager singleton: dataclass settings model with XDG/Flatpak config resolution, load/save with tuple serialization, field clamping, autostart .desktop management, zone mapping defaults, and the settings.json persistent state including the screen-cast restore token."
tags: [settings, persistence, flatpak, autostart, zone-mapping]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-ac4c6ac45ee00b030d8dfc41
    resource: repo://data/default_settings.json
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
  - id: openwiki-source-91b7b97766cd4b166d6345a8
    resource: repo://src/lumux/config/settings_manager.py
  - id: openwiki-source-17afc989609aefb00c1db725
    resource: repo://src/lumux/config/zone_mapping.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Settings and Persistent State

`SettingsManager` (`src/lumux/config/settings_manager.py`) is the process-wide configuration store: a singleton holding a `Settings` aggregate of seven typed sections, persisted as `settings.json`. Zone mappings are the one piece of "configuration" that is deliberately **not** persisted.

## Singleton and config location

- `SettingsManager` is a hand-rolled singleton: `__new__` creates `_instance` exactly once and `_init()` runs there (fresh `Settings()`, config dir resolution, initial load); `get_instance()` is the public accessor and section dataclasses are exposed as properties (`hue`, `capture`, `zones`, `sync`, `ui`, `black_bar`, `reading_mode`).
- **Config dir** — `is_running_in_flatpak()` detects the sandbox via `/.flatpak-info` or `FLATPAK_ID`. Inside Flatpak the config dir is `$XDG_CONFIG_HOME/lumux` (portable to `~/.var/app/io.github.enginkirmaci.lumux/config/lumux`); otherwise `~/.config/lumux`. The settings file is `settings.json` inside that directory.

## Settings schema

Defaults live in the dataclasses; `data/default_settings.json` (installed to `share/lumux/data` per pyproject) mirrors the same values as a shipped reference artifact. Current values:

| Section | Fields (defaults) |
|---|---|
| `hue` | `bridge_ip` "", `app_key` "", `client_key` "" (DTLS PSK), `entertainment_config_id` "", `auto_discover` true |
| `capture` | `scale_factor` 0.125, `source_type` "screen" (or "window"), `restore_token` "" |
| `zones` | `show_preview` true, `rows` 16, `cols` 16 |
| `sync` | `fps` 30, `transition_time_ms` 50, `brightness_scale` 1.0, `gamma` 1.0, `smoothing_factor` 0.3 |
| `ui` | `start_at_startup` false, `minimize_to_tray_on_sync` false, `minimize_at_startup` false |
| `black_bar` | `enabled` false, `threshold` 10, `detection_rate` 30, `smooth_factor` 0.3 |
| `reading_mode` | `color_xy` (0.5, 0.4) warm white, `brightness` 150, `auto_activate` true, `auto_activate_on_startup` false, `light_ids` [] (empty = auto) |

## Load / save semantics

- **Load** — `_load_settings()` parses the JSON file (when present), reconstructs each section with its dataclass, and handles JSON's flat types: a list-typed `color_xy` becomes a tuple, and a non-list `light_ids` is reset to `[]`. A decode/type error is logged and the section keeps defaults. Either way `_validate_settings()` runs before the manager becomes usable.
- **Save** — `save()` validates first, then writes all seven sections via `asdict` (tuples serialize back to JSON arrays) with 2-space indent.
- **Validation and clamping** — every load and save passes numeric fields through range clamps: `scale_factor` [0.01, 1], `source_type` must be "screen"/"window" (else "screen"), `fps` [1, 60], `transition_time_ms` [0, 1000], `brightness_scale` [0, 2], `gamma` [0.1, 3], `smoothing_factor` [0.1, 1], `rows`/`cols` [1, 64], black-bar `threshold` [0, 50], `detection_rate` [1, 120], `smooth_factor` [0.1, 1], reading `color_xy` clamped to [0, 1] per coordinate (or reset to (0.5, 0.4) if malformed), reading `brightness` [0, 254]. Booleans are coerced with `bool()`.
- **Restore token round-trip** — `capture.restore_token` is the persistent half of the screen-cast consent flow: `ScreenCapture` fires its `on_restore_token_changed` callback whenever the GNOME portal issues or refreshes a token, `AppContext._on_restore_token_changed` stores it in settings and calls `save()`, so consent survives restarts and the portal prompt is skipped on next launch (see [Screen Capture](../pipeline/screen-capture.md)).

## Autostart

Autostart is managed by writing/removing `io.github.enginkirmaci.lumux.desktop` in the session autostart directory (`$XDG_CONFIG_HOME/autostart` under Flatpak, `~/.config/autostart` otherwise) with `X-GNOME-Autostart-enabled=true`:

- **Native** — `Exec` is the current Python interpreter plus the absolute script path, shell-quoted.
- **Flatpak** — `Exec` is `flatpak run io.github.enginkirmaci.lumux`. The write is followed by an existence check; if the file is missing afterwards a `PermissionError` is raised advising that the app needs `filesystem=host` (a sandboxed Flatpak cannot reach the host autostart directory). Other write failures log and return `False`; disable removes the file; `get_autostart_status()` returns an enabled flag plus a human-readable message.

## Zone mapping (ephemeral)

`get_zone_mapping()` returns a **fresh, empty `ZoneMapping`** on every call — mappings are regenerated on each sync start, never written to disk. `ZoneMapping` (`src/lumux/config/zone_mapping.py`) defaults to a ten-zone ambilight grid (3 top, 3 bottom, 2 left, 2 right) with empty light lists.

`generate_ambilight_mapping(available_lights, light_info, ...)` fills it:

1. **Binning by position** — each light with spatial coordinates (from entertainment config positions, see [REST Client](../bridge/rest-client.md)) is binned by thresholds: `z > 0.4` → top, `z < -0.3` → bottom, `x < -0.4` → left, `x > 0.4` → right, otherwise → top. Lights without positions are skipped.
2. **Gradient strips** — a gradient-capable light whose name/archetype contains "strip" and sits in the upper area (`z > -0.2`) is added to the left, top, and right bins (wrapping fixtures typically span corners).
3. **Zone index** — non-gradient lights map to a zone index by normalizing the along-edge coordinate (`x` for top/bottom, `z` for left/right) into `[0, count-1]` with clamping; gradient lights are added to *all* zones on their edge. Without hardware info at all, lights fall back to a round-robin over the four edges.
4. **Validation** — `validate_mapping(available_lights)` returns mapped lights that no longer exist on the bridge, so stale mappings are pruned at sync start.

## Invariants

- `settings.json` is the only file-backed persistent state; there is no database.
- The manager is safe on first run: a missing file leaves defaults, and `_ensure_config_dir()` creates the directory before any write.
- Autostart state is read from the `.desktop` file itself, not from settings.json.

## Cross-references

- Startup consumer and restore-token persistence callback: [Application Shell and Startup](../architecture/application-shell.md)
- Portal capture side of the restore token: [Screen Capture](../pipeline/screen-capture.md)
- Position data that feeds `light_info`: [Hue Bridge REST Client](../bridge/rest-client.md)
- Mode settings and auto-activation: [Video and Reading Modes](../modes/video-reading-modes.md)
