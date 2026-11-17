---
type: "Reference"
title: "Hue Bridge REST Client"
description: "Two-layer Hue v2 REST stack: BridgeClient (requests session, self-signed TLS, /clip/v2 endpoints, error taxonomy, legacy user creation) and the HueBridge facade (device/light-info caching, spatial positions, SSDP/mDNS/N-UPnP discovery)."
tags: [hue-bridge, rest-client, discovery, authentication, zones]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-15T02:33:14.256Z
sources:
  - id: openwiki-source-5f020fc177d3c3a9da9013f5
    resource: repo://src/lumux/bridge_client.py
  - id: openwiki-source-aafb67c303d27868494f68c1
    resource: repo://src/lumux/hue_bridge.py
generated: { by: "pi", at: "2026-11-15T02:33:14.256Z" }
---

# Hue Bridge REST Client

The Hue integration is split into two layers: `BridgeClient` (`src/lumux/bridge_client.py`) — a thin, consistent client for the Hue v2 REST API that replaced the `python-hue-v2` dependency — and `HueBridge` (`src/lumux/hue_bridge.py`), the high-level facade the app actually uses. It manages connection state, caches device data, resolves spatial positions, and exposes discovery.

## BridgeClient (low-level)

### Transport and authentication

- One `requests.Session` with `verify = False` (the Hue bridge serves a self-signed certificate); `urllib3` insecure-request warnings are disabled at module import.
- Every request goes to `https://{bridge_ip}/clip/v2{path}` with the app key sent as the `hue-application-key` header; default timeout is 5 s.
- `hue-application-id` lookup is the one call that bypasses the session: `get_application_id()` uses `urllib.request` with an unverified TLS context to `GET /auth/v1` and returns the `hue-application-id` **response header** (requests drops response headers it doesn't model, and this is the DTLS PSK identity needed for [entertainment streaming](entertainment-streaming.md)). Any failure returns `None` rather than raising.

### Error taxonomy

`_request` maps failures onto a three-class hierarchy rooted at `BridgeError`:

| Condition | Exception |
|---|---|
| HTTP 401, 403 | `BridgeAuthError` |
| HTTP 404 | `BridgeError` ("Resource not found") |
| HTTP ≥ 500 | `BridgeConnectionError` |
| requests timeout / connection refused | `BridgeConnectionError` |
| any other `RequestException` | `BridgeError` |

`create_user` and `test_connection` (a probe `GET /resource/light`) complete the surface.

### Endpoint groups

- **Lights** — `get_lights`, `get_light`, `set_light_state` (raw payload), `set_light_color` (xy + brightness clamped 0–254, converted to the v2 0–100% dimming scale, `on: true`, optional `dynamics.duration` transition), `set_light_gradient` (validates each point to `{"color": {"xy": {x, y}}}`, rejects fewer than 2 points).
- **Zones** — `get_zones`, `set_zone_state`, `set_zone_color` (same xy/brightness payload shape as lights).
- **Entertainment** — list/get `entertainment_configuration`, and `activate_entertainment_streaming` / `deactivate_entertainment_streaming`, which `PUT` `{"action": "start"}` / `{"action": "stop"}` on the configuration resource.
- **Devices** — `get_devices` (`/resource/device`), used for the service-RID resolutions below.
- **User creation** — `create_user` is a `@staticmethod` speaking the **legacy v1 API**: `POST https://{bridge_ip}/api` with `{"devicetype": "<name>#user", "generateclientkey": true}` (`verify=False`). On success it returns `{"app_key": success.username, "client_key": success.clientkey}`; a bridge-level error raises `BridgeAuthError`, a transport error raises `BridgeConnectionError`. The link button must already have been pressed.

## HueBridge (facade)

- **Lazy client** — the `client` property constructs a `BridgeClient` only when both `bridge_ip` and `app_key` are set; `create_user` stores the returned credentials and resets the client so the next call rebuilds with them.
- **`connect()`** — succeeds if `refresh_devices()` runs without `BridgeError`.
- **`refresh_devices()`** — rebuilds three caches: `lights` (id → raw resource), `light_info` (per light: name, archetype, modelid, type, on-state, `is_gradient` derived from `gradient.points`/`points_capable`, `gradient_points`, gamut fields, and `position` initialized to `None`), then `zones` (id → raw resource).
- **Spatial data** — `_refresh_spatial_data()` first maps each device's light service RID to its entertainment service RID (from the device's `services` list), then walks every entertainment configuration's `locations.service_locations` and fills `light_info[light_rid]["position"]` wherever the RIDs line up. This is how the app learns where physical lights sit.
- **Zone membership** — `get_entertainment_light_ids(config_id)` collects every service RID referenced by the configuration's channel members and service locations, then returns the light RIDs of any device whose services intersect that set. This is the definition of "which lights are in the entertainment zone."
- **Swallowed errors** — every facade control method (`set_light_color`, `set_light_gradient`, `set_zone_color`, entertainment activate/deactivate, configuration getters) catches `BridgeError`, prints it, and returns a safe default (`False`, `[]`, `None`). The GUI and sync loops therefore never see a raise from the bridge layer.
- **`create_user` note** — its signature accepts `max_retries` and `timeout` and the docstring describes exponential-backoff retry while waiting for the link button, but the implementation makes exactly one attempt and ignores both parameters. The promised retry behavior is not implemented in this snapshot.

## Discovery

`HueBridge.discover_bridges()` is a classmethod that runs three independent, exception-tolerant probes in order, then deduplicates:

1. **SSDP** (`_discover_ssdp`) — sends an `M-SEARCH *` (ST: `ssdp:all`) UDP datagram to the `239.255.255.250:1900` multicast group on a timed socket, collects source IPs of responses containing `hue-bridgeid` or `phillips-hue` for up to `timeout` seconds.
2. **mDNS** (`_discover_mdns`) — optionally imports `zeroconf` and browses `_hue._tcp.local.` for `timeout` seconds; a missing `zeroconf` package degrades to a warning, not an error.
3. **N-UPnP** (`_discover_nupnp`) — queries the cloud endpoint `https://discovery.meethue.com/` for bridges associated with the machine's public IP.

If a full pass finds nothing, the whole sequence is retried up to `max_retries` times with exponential backoff capped at 10 s. Each probe individually swallows its own exceptions, so any single failing method never aborts discovery.

## Failure behavior and invariants

- Bridge errors are never fatal at app level: transport/HTTP problems surface as typed exceptions in `BridgeClient` and are converted to safe defaults by `HueBridge`.
- Credential bootstrap (`create_user`) is the only flow that uses the v1 `/api` surface; everything else is v2 `/clip/v2`.
- The `client_key` returned by user creation feeds the DTLS PSK of [Entertainment Streaming](entertainment-streaming.md); the `hue-application-id` header feeds its PSK identity.

## Cross-references

- DTLS consumer of the PSK identity and client key: [Entertainment Streaming](entertainment-streaming.md)
- Startup flow that connects the bridge and refreshes devices: [Application Shell and Startup](../architecture/application-shell.md)
- Whole-system context: [Architecture Overview](../architecture/overview.md)
- Mode switching that drives zone control: [Video and Reading Modes](../modes/video-reading-modes.md)
