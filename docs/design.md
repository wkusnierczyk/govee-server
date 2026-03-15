# `govee-server` — Design Document

`govee-server` is a deployable REST service and web GUI built on the `govee` library. It exposes
Govee device control over HTTP, making it accessible from browsers, home automation platforms, and
any HTTP client — without requiring a locally installed binary or Rust toolchain.




## Table of Contents

- [1. Goals](#1-goals)
- [2. REST API](#2-rest-api)
- [3. WebSocket](#3-websocket)
- [4. Web GUI](#4-web-gui)
- [5. Authentication](#5-authentication)
- [6. Configuration](#6-configuration)
- [7. Deployment](#7-deployment)
- [8. Design discussions](#8-design-discussions)
  - [8.1 Device addressing: name vs ID in path](#81-device-addressing-name-vs-id-in-path)
  - [8.2 WebSocket vs SSE for push](#82-websocket-vs-sse-for-push)
  - [8.3 Frontend technology](#83-frontend-technology)
  - [8.4 Workflow execution via REST](#84-workflow-execution-via-rest)
- [9. Dependencies](#9-dependencies)
- [10. Relationship to other consumers](#10-relationship-to-other-consumers)

---

## 1. Goals
<sub>[↑ TOC](#table-of-contents) · [2. REST API →](#2-rest-api)</sub>


- REST API covering the full `govee` library surface, usable from any HTTP client
- Real-time device state via WebSocket push
- Minimal embedded web GUI for basic control without a CLI
- Deployable locally (home server, NAS, Docker) or to a cloud host
- Single statically linked binary with no external runtime dependencies
- Optional authentication for non-local deployments

---

## 2. REST API
<sub>[↑ TOC](#table-of-contents) · [← 1. Goals](#1-goals) · [3. WebSocket →](#3-websocket)</sub>


All control endpoints under `/api/v1/`. Devices are addressed by name or alias in the path (not
by MAC address — the registry handles resolution internally).

| Method | Path | Body | Description |
|---|---|---|---|
| `GET` | `/api/v1/devices` | — | List all devices with current state |
| `GET` | `/api/v1/devices/{name}/state` | — | State for one device |
| `PUT` | `/api/v1/devices/{name}/power` | `{"on": bool}` | Turn on or off |
| `PUT` | `/api/v1/devices/{name}/brightness` | `{"value": 0–100}` | Set brightness |
| `PUT` | `/api/v1/devices/{name}/color` | `{"r": 0–255, "g": 0–255, "b": 0–255}` | Set RGB colour |
| `PUT` | `/api/v1/devices/{name}/color-temp` | `{"kelvin": u32}` | Set colour temperature |
| `GET` | `/api/v1/scenes` | — | List available scenes |
| `POST` | `/api/v1/scenes/{scene}/apply` | `{"target": "all" \| name \| group}` | Apply a scene |
| `POST` | `/api/v1/workflows/run` | `{"path": string}` | Run a workflow file on the server |
| `GET` | `/api/v1/backend/status` | — | Backend health and active backend per device |

`{name}` in the path accepts device names, aliases, group names, and the literal `all` where
semantically appropriate (bulk operations).

All responses are JSON. Errors use the shape `{"error": "<message>", "code": "<kind>"}` with
appropriate HTTP status codes (404 for device not found, 429 for rate limit, 503 for backend
unavailable).

---

## 3. WebSocket
<sub>[↑ TOC](#table-of-contents) · [← 2. REST API](#2-rest-api) · [4. Web GUI →](#4-web-gui)</sub>


`GET /ws` upgrades to a WebSocket connection. The server pushes JSON events:

```json
{"event": "state_changed",      "device": "bedroom", "state": { ... }}
{"event": "backend_changed",    "device": "bedroom", "backend": "local"}
{"event": "discovery_complete", "device_count": 3}
{"event": "rate_limited",       "retry_after_secs": 42}
```

Clients may also send control commands over the WebSocket using the same JSON body shapes as the
REST endpoints, with an additional `"action"` field identifying the operation:

```json
{"action": "set_power",    "device": "bedroom", "on": true}
{"action": "apply_scene",  "scene": "warm",     "target": "all"}
```

WebSocket is primarily intended for the web GUI. REST clients should use the REST API.

---

## 4. Web GUI
<sub>[↑ TOC](#table-of-contents) · [← 3. WebSocket](#3-websocket) · [5. Authentication →](#5-authentication)</sub>


A minimal single-page application served at `/`. It communicates exclusively via WebSocket.

**Functionality:**

- Device list with live state (on/off indicator, brightness bar, colour swatch)
- Per-device controls: toggle, brightness slider, colour picker, colour temperature slider
- Scene selector applied to one device, a group, or all
- Backend status panel
- No login page in local mode; API key prompt if auth is enabled

**Technology:** vanilla TypeScript compiled to a single JS bundle, embedded in the binary via
`include_dir!`. No npm, no build step for the end user. The GUI is deliberately minimal — it is
not a full home automation dashboard. Users wanting that should integrate `govee-server`'s REST
API with Home Assistant or similar.

**Build:** the frontend is pre-built and committed to the repository as compiled assets. The
`include_dir!` macro embeds them at compile time. This keeps the Rust build self-contained.

---

## 5. Authentication
<sub>[↑ TOC](#table-of-contents) · [← 4. Web GUI](#4-web-gui) · [6. Configuration →](#6-configuration)</sub>


**Local mode (default):** no authentication. The server binds to `127.0.0.1:8080` by default.
Access control is the user's responsibility at the network level.

**Auth mode:** enabled by setting `auth_key` in config. All requests must include
`Authorization: Bearer <key>`. The same key is used for both REST and WebSocket. No OAuth, no
user accounts, no session management — a single shared key is the right scope for personal home
automation.

---

## 6. Configuration
<sub>[↑ TOC](#table-of-contents) · [← 5. Authentication](#5-authentication) · [7. Deployment →](#7-deployment)</sub>


Extends `~/.config/govee/config.toml` with a `[server]` section:

```toml
[server]
bind     = "127.0.0.1:8080"  # change to "0.0.0.0:8080" for LAN access
auth_key = ""                 # empty string = no auth
```

All `govee` library configuration (API key, backend preference, aliases, groups) is read from the
same file.

---

## 7. Deployment
<sub>[↑ TOC](#table-of-contents) · [← 6. Configuration](#6-configuration) · [8. Design discussions →](#8-design-discussions)</sub>


**Home server / NAS:**

```sh
cargo install govee-server
# then manage with systemd, launchd, or supervisord
```

Example systemd unit:

```ini
[Service]
ExecStart=/usr/local/bin/govee-server
Restart=on-failure
Environment=GOVEE_API_KEY=your-key
```

**Docker:**

```dockerfile
FROM scratch
COPY govee-server /govee-server
EXPOSE 8080
ENTRYPOINT ["/govee-server"]
```

```sh
docker run -e GOVEE_API_KEY=your-key \
           -p 8080:8080 \
           --network host \   # required for local LAN backend
           wkusnierczyk/govee-server
```

`--network host` is required for the local LAN backend because the Govee LAN protocol uses fixed
UDP ports that must be reachable from the container's IP. For cloud-only mode, bridge networking
with `-p 8080:8080` is sufficient.

---

## 8. Design discussions
<sub>[↑ TOC](#table-of-contents) · [← 7. Deployment](#7-deployment) · [8.1 Device addressing: name vs ID in path →](#81-device-addressing-name-vs-id-in-path)</sub>


### 8.1 Device addressing: name vs ID in path
<sub>[↑ TOC](#table-of-contents) · [← 8. Design discussions](#8-design-discussions) · [8.2 WebSocket vs SSE for push →](#82-websocket-vs-sse-for-push)</sub>


REST convention uses stable opaque IDs in paths (e.g., `/devices/abc123`). Using names
(`/devices/bedroom`) is more readable but breaks if a device is renamed in the Govee app.

Decision: **name/alias in path**, because this is a personal tool and readability beats pedantic
REST correctness. Renames are rare; when they happen the config alias can paper over them without
touching the URL. The MAC-derived stable ID is available in the response body for clients that
prefer it.

### 8.2 WebSocket vs SSE for push
<sub>[↑ TOC](#table-of-contents) · [← 8.1 Device addressing: name vs ID in path](#81-device-addressing-name-vs-id-in-path) · [8.3 Frontend technology →](#83-frontend-technology)</sub>


Server-Sent Events are simpler (one-directional, plain HTTP) but WebSocket is bidirectional and
allows the GUI to also send commands over the same connection, reducing complexity in the frontend.

Decision: **WebSocket**, primarily for GUI simplicity.

### 8.3 Frontend technology
<sub>[↑ TOC](#table-of-contents) · [← 8.2 WebSocket vs SSE for push](#82-websocket-vs-sse-for-push) · [8.4 Workflow execution via REST →](#84-workflow-execution-via-rest)</sub>


Options considered: React/Next.js, SvelteKit, vanilla JS, HTMX.

React and SvelteKit require a build pipeline and produce large bundles. HTMX is elegant for
server-rendered UIs but WebSocket-heavy real-time state is awkward. Vanilla TypeScript compiled
with `esbuild` produces a ~20KB bundle with no framework overhead and no npm install step for the
end user.

Decision: **vanilla TypeScript + esbuild**, pre-compiled and committed.

### 8.4 Workflow execution via REST
<sub>[↑ TOC](#table-of-contents) · [← 8.3 Frontend technology](#83-frontend-technology) · [9. Dependencies →](#9-dependencies)</sub>


`POST /api/v1/workflows/run` takes a path on the server's filesystem, not an uploaded YAML body.
This is intentional — the server is a long-running process on the same machine as the workflow
files, and file-path semantics are simpler and safer than accepting arbitrary YAML from the
network. If remote workflow execution is needed later, it can be added as a separate endpoint.

---

## 9. Dependencies
<sub>[↑ TOC](#table-of-contents) · [← 8.4 Workflow execution via REST](#84-workflow-execution-via-rest) · [10. Relationship to other consumers →](#10-relationship-to-other-consumers)</sub>


| Crate | Role |
|---|---|
| `govee` | All device control logic |
| `axum` | HTTP server, WebSocket, routing |
| `tokio` | Async runtime |
| `tower` / `tower-http` | Middleware: auth, CORS, request logging |
| `serde` / `serde_json` | JSON bodies |
| `include_dir` | Embed compiled frontend assets in binary |
| `anyhow` | Error handling at binary boundary |
| `tracing-subscriber` | Structured request logging |

---

## 10. Relationship to other consumers
<sub>[↑ TOC](#table-of-contents) · [← 9. Dependencies](#9-dependencies)</sub>


`govee-server` and `govee-mcp` may run simultaneously — they share no ports and do not conflict.
Both are separate processes with their own `DeviceRegistry`. If both use the local LAN backend
from the same host IP, they will conflict on UDP port 4002 (a hard constraint of the Govee LAN
protocol). One must be configured with `backend = "cloud"`, or they must run on separate IPs
(secondary NIC, Docker macvlan). Both binaries detect this at startup and emit a clear error
rather than silently failing.
