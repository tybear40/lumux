# Files

- [Screen Capture Pipeline](screen-capture.md) - How Lumux gets pixels on Wayland: the xdg-desktop-portal ScreenCast session lifecycle (CreateSession with persist_mode, SelectSources, Start, restore tokens for persistent consent), the ranked GStreamer pipeline candidates with runtime-failure memory, frame extraction with stride handling, format conversion, PIL scaling, and black-bar cropping.
- [Sync Engine](sync-engine.md) - SyncController: the single background sync thread, FPS clamping and pacing, per-stage timing stats, the bounded status queue drained by the GUI, position-based zone-to-entertainment-channel mapping with multi-zone averaging, and light update dispatch over DTLS.
- [Zone Color Pipeline](zone-color-pipeline.md) - Frame-to-color math: ZoneProcessor ambilight edge decomposition into 2*rows + 2*cols RGB zones, ColorAnalyzer gamma + max-component brightness + EMA smoothing, and RGB-to-CIE-XY conversion with optional phosphor-triangle gamut clipping.
