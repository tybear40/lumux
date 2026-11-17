---
type: "Reference"
title: "Lumux Quickstart"
description: "Navigation hub: which wiki page answers which kind of question, how to run the app, the source entrypoints, and the known-issues and no-test-suite gotchas."
tags: [navigation, quickstart, entrypoints, testing, stability]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
  - id: openwiki-source-cb3129d36b1c78edd3deb82a
    resource: repo://src/lumux/__init__.py
  - id: openwiki-source-f4d1578b16cd7e96bab8322b
    resource: repo://src/lumux/__main__.py
  - id: openwiki-source-ea1184835bd4d7b3616bae66
    resource: repo://STABILITY_TODOS.md
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Lumux Quickstart

Routing map for the Lumux repository. Pick your task, read the page.

## Running the app

- **Flathub**: `flatpak install flathub io.github.enginkirmaci.lumux` (recommended; portals, sandbox, autostart all pre-wired — see [Distribution](packaging/distribution.md)).
- **From source**: `pip install -e .` then `python -m lumux` or the `lumux` entrypoint.
- **Code entrypoints**: `src/lumux/__main__.py` defines `LumuxApp` (the `Adw.Application`) and `main()` — this is where `AppContext`, the signal handlers, and the main window are created (see [Application Shell](architecture/application-shell.md)). `lumux/__init__.py` re-exports the 12 core classes.
- **Config**: everything persists to `~/.config/lumux/settings.json` (`/var/config/lumux/` under Flatpak) — see [Settings and State](configuration/settings-and-state.md).

## Task routing

| You want to… | Read |
|---|---|
| Understand the whole system, thread model, data flow | [Architecture Overview](architecture/overview.md) |
| Add/change a screen UI element, dialog, tray behavior | [GTK 4 Interface](gui/gtk4-interface.md) |
| Change window startup, app lifecycle, autostart, reading-mode auto-activation | [Application Shell](architecture/application-shell.md) |
| Touch capture, pipeline, black bars, sampling | [Screen Capture](pipeline/screen-capture.md) |
| Touch zone grid, gamma/brightness, xy conversion, smoothing | [Zone Color Pipeline](pipeline/zone-color-pipeline.md) |
| Touch the sync loop, per-frame chain (zones → colors → smoothing → channels), mode toggling | [Sync Engine](pipeline/sync-engine.md) |
| Touch Hue REST, mDNS discovery, light/zone/gradient handling | [Bridge REST Client](bridge/rest-client.md) |
| Touch DTLS streaming, the message buffer, channel mapping | [Entertainment Streaming](bridge/entertainment-streaming.md) |
| Touch video/reading mode behavior, presets, sleep behavior | [Video & Reading Modes](modes/video-reading-modes.md) |
| Touch Flatpak, packaging, permissions, autostart | [Distribution](packaging/distribution.md) |
| Investigate freezes, crashes, instability | [Stability and Troubleshooting](operations/stability-and-troubleshooting.md) — and the **`STABILITY_TODOS.md`** file at the repo root (see gotchas) |

## Gotchas

- **No automated tests.** `pyproject.toml` declares `testpaths = ["tests"]` and pytest dev extras, but no `tests/` directory exists in the repository — the test scaffolding is aspirational. Verify changes by running the app.
- **`STABILITY_TODOS.md` is the authoritative known-issues list.** A full stability investigation (v0.6.2, 2026-11-14) is recorded there: 22 items triaged P0–P2 with confident fixes — main-thread blocking I/O (20–30 s freezes), pydbus portal thread-affinity bugs, GStreamer signal dispatch on a non-looping thread, pipeline start/stop races, a reproduced mode-switch race, non-atomic settings writes, a fixed-size DTLS buffer assumption, and a shared `requests.Session` across threads. **Read it before making any stability-adjacent change** and check whether your change overlaps an already-diagnosed item.
- **Icon paths are broken in dev installs** (item 9 of the investigation): the app and tray icons point at repo locations where the SVG actually lives in `flatpak/`; the tray icon may render blank outside Flatpak.
- **`print` is the only logging** — there is no persistent log file; runtime behavior shows up on stdout only.
- **Uncommitted work may exist** in the tree (the investigation flagged a partial `app_context.py` edit); check `git status` before assuming the committed state matches what runs.

## See also

[Architecture Overview](architecture/overview.md) · [Settings and State](configuration/settings-and-state.md) · [Stability and Troubleshooting](operations/stability-and-troubleshooting.md) · [Distribution](packaging/distribution.md)
