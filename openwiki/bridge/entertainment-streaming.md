---
type: "Reference"
title: "Entertainment Streaming (DTLS)"
description: "HueStream v2 DTLS-PSK client for entertainment zones: connect/disconnect lifecycle with REST activation, openssl s_client transport, 52-byte message framing, and XY/RGB channel packing."
tags: [entertainment, dtls, huestream, hue-bridge, streaming]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-e01b0bac62642a9db41b15fe
    resource: repo://src/lumux/app_context.py
  - id: openwiki-source-5f020fc177d3c3a9da9013f5
    resource: repo://src/lumux/bridge_client.py
  - id: openwiki-source-f69325c01fbd74355c2bd36a
    resource: repo://src/lumux/entertainment.py
  - id: openwiki-source-87ed945ce05051911da34129
    resource: repo://src/lumux/sync.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Entertainment Streaming (DTLS)

`EntertainmentStream` (`src/lumux/entertainment.py`) is the low-latency client for Hue entertainment zones. Where the REST API is request/response, entertainment streaming is a persistent DTLS-PSK channel to the bridge's UDP port 2100: the app keeps a socket open for the lifetime of video mode and pushes one binary HueStream v2 message per frame.

## Protocol constants

`HueStreamProtocol` pins the wire format:

| Element | Value |
|---|---|
| Magic header | `b"HueStream"` (9 bytes) |
| Version | 2.0 (`0x02 0x00`) |
| Port | 2100 (DTLS over UDP) |
| Colorspace byte | `0x00` RGB, `0x01` XY |
| Header size | 52 bytes = 9 magic + 2 version + 1 sequence + 2 reserved + 1 colorspace + 1 reserved + 36 config ID |
| Channel record | 7 bytes = 1 channel ID + 3 × big-endian uint16 (`>BHHH`) |
| XY scaling | x, y clamped to [0,1] × 65535 |
| Brightness scaling | 0–254 × 257 (≈65278, the protocol max) |

A full message is `52 + 7×N_channels` bytes, where N is the number of channels in the entertainment configuration.

## Connection lifecycle

`connect(bridge)` runs a strict sequence, with REST bookkeeping around the DTLS connection:

1. **Fetch config** — `GET /resource/entertainment_configuration/{id}` via the bridge; abort if missing.
2. **Parse channels** — each channel entry becomes a `ChannelInfo` (channel ID, 3D position, member list); member `service.rid` values are indexed into `light_to_channel` for the reverse light→channel lookup.
3. **Init message buffer** — one `bytearray` of the exact total size; the static header (magic, version, zeroed reserved bytes, 36-byte ASCII config UUID at offset 16) is written once. Per-message updates only touch byte 11 (sequence) and byte 14 (colorspace) plus the channel payload, so no allocation happens per frame.
4. **Fetch PSK identity** — `hue-application-id` from the bridge's `/auth/v1` endpoint (the correct DTLS identity per the official API docs), falling back to `app_key` when unavailable.
5. **Activate via REST** — `PUT` `{"action": "start"}` on the entertainment configuration, then sleep `connection_timeout` (default 0.5 s) to let the bridge prepare.
6. **Open DTLS** — if the handshake fails, the REST `{"action": "stop"}` deactivation is issued to leave the bridge in a clean state.

## DTLS transport (openssl subprocess)

The connection is an `openssl s_client` child process, not a native socket:

```
openssl s_client -dtls1_2 -psk_identity <hue-application-id|app_key> \
    -psk <client_key> -cipher PSK-AES128-GCM-SHA256:PSK-CHACHA20-POLY1305 \
    -connect <bridge_ip>:2100 -quiet
```

with piped stdin/stdout/stderr. The `client_key` is a 32-char hex string obtained at authentication time from the legacy v1 user-creation call (`generateclientkey=true`).

Because there is no readable certificate or session callback, handshake success is verified by **process polling**: sleep `connection_timeout` (0.5 s), check `proc.poll()`; sleep `handshake_delay` (0.3 s), check again. A dead process means failure — its stdout/stderr are drained and printed with the exit code, and the connection is reported as failed.

**Send path** — `send_colors_xy` (used by the sync loop) and `send_colors` (RGB variant) both: check `is_connected()`, then under `_lock` build the message into the pre-allocated buffer and write it to the subprocess stdin with a flush. A `BrokenPipeError`/`OSError` from the pipe marks `_connected = False` and re-raises; the sender catches the exception and resets the flag.

**Connection-loss detection** — `is_connected()` returns true only when the `_connected` flag is set *and* `proc.poll() is None` (process still alive), so a silently-dead openssl process is treated as disconnected before the next send is attempted. The sync loop consults this on every `_update_lights` call.

A native socket path exists as a placeholder: `_dtls_socket` is initialized to `None` with a "Future: native DTLS implementation" comment, and `_send_via_socket` is implemented but unreachable because the socket is never assigned.

## Send API

- **`send_colors_xy(channel_colors)`** — the primary path. `channel_colors` maps channel ID → `((x, y), brightness)` with x/y as CIE coordinates (0–1) and brightness 0–254. Each channel is packed as `>BHHH`: ID, `x×65535`, `y×65535`, `brightness×257`.
- **`send_colors(colors)`** — RGB variant packing `r,g,b` (0–1) × 65535; currently unused by the app (the sync loop always sends XY).
- Channels present in the configuration but absent from the input dict are zeroed (black) each message; the 8-bit sequence wraps at 256 after each send.

## Spatial mapping helpers

`get_channel_positions()` exposes the configuration's per-channel `{x, y, z}` positions (normalized bridge space), which `SyncController` uses for nearest-channel zone mapping. `map_zone_to_channel()` is a self-contained alternative: it matches edge ranges (top `z ∈ [0.5, 1]`, bottom `z ∈ [-1, -0.5]`, left `x ∈ [-1, -0.5]`, right `x ∈ [0.5, 1]`), sorts matching candidates along the edge axis, and picks by clamped index. As of this snapshot no caller invokes `map_zone_to_channel`; the sync engine implements its own nearest-position mapping instead.

## Teardown

`disconnect(bridge)` clears `_connected`, cleans up the subprocess (close stdin → terminate → `wait(timeout=2)` → kill on timeout), then issues the REST stop so the bridge releases the zone. `AppContext.stop_entertainment` only calls it when `is_connected()` is true; `AppContext.start_entertainment` requires `bridge.test_connection()` to pass before attempting `connect`.

## Cross-references

- REST endpoints used here (config fetch, activate/deactivate, `/auth/v1`): [Hue Bridge REST Client](rest-client.md)
- Mode switching that drives connect/disconnect: [Video and Reading Modes](../modes/video-reading-modes.md)
- Per-frame consumer of `send_colors_xy`: [Sync Engine](../pipeline/sync-engine.md)
