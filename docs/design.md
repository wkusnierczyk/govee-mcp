# govee-mcp Design Document

## 1. Govee APIs

Govee exposes two independent control planes for its devices: a cloud REST API and a local LAN API
over UDP. They are not mirrors of each other — they differ in authentication model, latency,
feature surface, and reliability characteristics.

### 1.1 Cloud API (v1 / v2)

The cloud API is a conventional HTTPS REST API hosted by Govee. Authentication uses a static API
key passed as the `Govee-API-Key` header, obtained by request from the Govee Home mobile app (sent
by email, refreshed on re-request).

**Endpoints (v1):**

- `GET /v1/devices` — list all devices on the account with model, MAC, name, and controllability
  flag
- `GET /v1/devices/state` — query the current state of a single device (on/off, brightness, color,
  color temperature)
- `PUT /v1/devices/control` — send a control command to a single device

**Commands supported via `control`:**

| Command | Parameter | Range |
|---|---|---|
| `turn` | `value: "on" \| "off"` | — |
| `brightness` | `value: 0–100` | integer |
| `color` | `{ r, g, b }` | 0–255 each |
| `colorTem` | `value` | device-dependent Kelvin range |

**Rate limiting:** Two-level enforcement. Officially documented as 10,000 requests per 24 hours
(~6.9 req/min sustained), but community reports indicate a tighter per-minute cap also applies.
Hitting it returns HTTP 429.

**State freshness caveat:** The cloud state endpoint uses cached values. Govee's own documentation
notes that `online` may return incorrect results, and state changed via Bluetooth from the app is
not reflected until a Wi-Fi sync occurs.

**v2 API:** A newer API (released late 2023) adds richer capability negotiation — devices report
which commands they support, including scenes, music mode, and DIY effects. The v2 device list
returns a `capabilities` array per device. v2 is less widely documented and community support is
thinner, but adds meaningful functionality for advanced devices. H6078 is explicitly listed in v2
changelog support.

### 1.2 Local LAN API

The local API communicates directly with devices on the same LAN segment using UDP. It must be
explicitly enabled per device in the Govee Home app (Settings → LAN Control). All three of
Wacek's devices (H6076, H6078, H6079) are confirmed to support local control.

**Transport:**

- Discovery: multicast UDP to `239.255.255.250:4001`, devices respond to the client's `4002`
- Control: unicast UDP to the device's IP on port `4003`
- These port numbers are fixed by the protocol — two LAN API implementations cannot run on the
  same host IP simultaneously

**Discovery flow:**

```json
// Client broadcasts to 239.255.255.250:4001
{ "msg": { "cmd": "scan", "data": { "account_topic": "reserve" } } }

// Device unicasts back to client:4002
{ "msg": { "cmd": "scan", "data": {
  "ip": "192.168.1.42", "device": "AA:BB:CC:DD:EE:FF:00:11",
  "sku": "H6078", "bleVersionHard": "...", "wifiVersionSoft": "..."
}}}
```

**Control commands:**

```json
{ "msg": { "cmd": "turn",       "data": { "value": 1 } } }
{ "msg": { "cmd": "brightness", "data": { "value": 80 } } }
{ "msg": { "cmd": "colorwc",    "data": { "color": {"r":255,"g":100,"b":0}, "colorTemInKelvin": 0 } } }
{ "msg": { "cmd": "devStatus",  "data": {} } }
```

**State response** (reply to `devStatus`):

```json
{ "msg": { "cmd": "devStatus", "data": {
  "onOff": 1, "brightness": 100,
  "color": {"r": 255, "g": 100, "b": 0}, "colorTemInKelvin": 7200
}}}
```

**Important quirk:** Devices do not reliably return updated state immediately after a control
command. State sync typically takes a few seconds. The recommended pattern is optimistic update
(assume success) rather than read-after-write.

### 1.3 API Comparison

| Dimension | Cloud API | Local LAN API |
|---|---|---|
| **Auth** | Static API key (HTTPS header) | None |
| **Latency** | ~200–400ms (internet round-trip) | ~5–15ms (LAN UDP) |
| **Rate limit** | ~10,000 req/24h + per-minute cap | None |
| **Requires internet** | Yes | No |
| **Device coverage** | Broader (including BT-only devices) | Subset (WiFi only, LAN enabled) |
| **State accuracy** | Cached, can be stale | Fresher, but post-command lag |
| **Scenes / DIY** | v2 API, partial | Unofficial / not documented |
| **Discovery** | API returns device list | UDP multicast |
| **Concurrent implementations** | Unlimited | One per host IP (fixed ports) |
| **Reliability** | Depends on Govee cloud uptime | Local network only |
| **Setup** | API key request (email) | Per-device enable in app |

The local API is strictly better for the primitives it covers (on/off, brightness, color). The
cloud API is the fallback for devices that don't support local control, and the source of ground
truth for device names.

---

## 2. MCP Functionality

### 2.1 Philosophy: translation vs. abstraction

A pure API translation layer (one MCP tool per API endpoint) is low effort but low value — Claude
already understands concepts like "turn on" and "set brightness". The value of an MCP server is not
just connectivity; it's vocabulary that lets Claude operate at a higher level of intent.

The proposed approach is a **two-layer model**: a thin control layer that maps cleanly to API
primitives, and a scene/workflow layer that encodes common multi-device intentions.

### 2.2 Control layer (direct API mapping)

These tools are a direct, thin wrapper over API primitives:

| Tool | Description |
|---|---|
| `list_devices` | Return all known devices with name, model, current state, and which backend is active |
| `get_device_state` | Query current state for a specific device |
| `set_power` | Turn a device on or off |
| `set_brightness` | Set brightness 0–100 |
| `set_color` | Set RGB color |
| `set_color_temp` | Set color temperature in Kelvin |

### 2.3 Workflow layer (higher-level abstractions)

These tools encode intent that would otherwise require multiple steps or LLM inference:

| Tool | Description |
|---|---|
| `set_scene` | Apply a named preset (warm evening, focus, movie, etc.) across one or all devices |
| `set_all` | Apply the same state to all devices in one call |
| `set_group` | Apply state to a named group of devices (e.g., "bedroom", "office") |
| `transition` | Smoothly animate between two states over a duration (server-side polling loop) |
| `get_backend_status` | Report which backend is active per device and whether local is reachable |

Built-in scenes encode opinionated presets (warm white at 40% for evening, daylight at 80% for
focus, dim red at 10% for night) that Claude can invoke by name without needing to know Kelvin
values.

### 2.4 What is NOT exposed

- Raw scene codes / DIY effects (device-specific, undocumented format — v3 territory)
- Firmware OTA or account management
- Scheduling (better done by the Govee app or home automation layer)

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Claude / MCP Client                    │
└─────────────────────────────┬────────────────────────────────┘
                              │ MCP (stdio / JSON-RPC 2.0)
┌─────────────────────────────▼────────────────────────────────┐
│                        govee-mcp binary                       │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    Tool Handler Layer                    │ │
│  │  list_devices · set_power · set_color · set_scene · …   │ │
│  └──────────────────────────┬──────────────────────────────┘ │
│                             │                                 │
│  ┌──────────────────────────▼──────────────────────────────┐ │
│  │                  DeviceRegistry (Arc<RwLock<>>)          │ │
│  │  name → Device { id, model, backend, cached_state }     │ │
│  └──────┬────────────────────────────────┬─────────────────┘ │
│         │                                │                    │
│  ┌──────▼──────┐                ┌────────▼──────────┐        │
│  │ CloudBackend │                │   LocalBackend    │        │
│  │  reqwest     │                │  tokio::net::Udp  │        │
│  │  HTTP REST   │                │  multicast disco  │        │
│  └─────────────┘                └───────────────────┘        │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                  BackendSelector                         │ │
│  │  auto | cloud | local — per device or global flag       │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
          │ HTTPS                          │ UDP
   Govee Cloud API              Govee devices (LAN)
```

### 3.1 Backend trait

```rust
#[async_trait]
pub trait GoveeBackend: Send + Sync {
    async fn list_devices(&self) -> Result<Vec<Device>>;
    async fn get_state(&self, id: &DeviceId) -> Result<DeviceState>;
    async fn set_power(&self, id: &DeviceId, on: bool) -> Result<()>;
    async fn set_brightness(&self, id: &DeviceId, value: u8) -> Result<()>;
    async fn set_color(&self, id: &DeviceId, r: u8, g: u8, b: u8) -> Result<()>;
    async fn set_color_temp(&self, id: &DeviceId, kelvin: u32) -> Result<()>;
    fn backend_type(&self) -> BackendType;
}
```

`BackendType` is `Cloud | Local`. `BackendSelector` holds both, selects per-device based on
reachability, and exposes a `--backend cloud|local|auto` CLI flag.

### 3.2 DeviceRegistry

Maintains an in-memory map of devices, built at startup from cloud device list (canonical names
and IDs) merged with local discovery results (IPs). Updated periodically in a background task.
Allows tools to operate by human-readable name ("bedroom lamp") rather than MAC address.

### 3.3 Configuration

Via environment variables and/or a TOML config file at `~/.config/govee-mcp/config.toml`:

```toml
api_key = "..."          # Govee cloud API key
backend = "auto"         # auto | cloud | local
discovery_interval = 60  # seconds between LAN rediscovery

[groups]
bedroom = ["H6078-1", "H6079-1"]
office  = ["H6076-1"]
```

### 3.4 Transport

MCP over `stdio` — standard for local MCP servers, compatible with Claude Desktop and Claude Code
out of the box. No HTTP server, no auth, no port management.

---

## 4. Rust Fit

### 4.1 Advantages

**Type safety across protocol boundaries.** Both the Govee API and MCP protocol are JSON-heavy.
Rust's serde ecosystem makes it straightforward to define typed structs for every request/response
shape, with compile-time guarantees that the shapes are consistent. A Python or JS implementation
would silently pass wrong shapes at runtime.

**Single binary distribution.** The output is a statically linked binary (or near-static with
musl). Users install it with `cargo install govee-mcp` or download a GitHub release binary. No
runtime, no virtualenv, no `npm install`. This matters for a tool that runs as a background daemon.

**Async without cost.** Tokio handles the stdio MCP transport, the HTTP cloud calls, and the UDP
socket concurrently in a single process. The async model is zero-cost in Rust — no GIL, no green
thread overhead.

**UDP/multicast is first-class.** `tokio::net::UdpSocket` with multicast join is clean and
well-supported. Python's `asyncio` UDP story is historically messy; Go's is reasonable but
verbose.

**Memory footprint.** The process will idle at a few MB RSS. Relevant if it runs 24/7 on a home
server or NAS.

### 4.2 Disadvantages

**Slower iteration.** Changing a tool's parameter schema or adding a new command requires a
recompile. With Python/FastMCP you'd edit a decorator and restart. The compile time for this
crate will be in the 10–30s range once the dependency graph stabilises — acceptable but not
invisible.

**rmcp maturity.** The official Rust MCP SDK (`rmcp`, now at v0.16 under
`modelcontextprotocol/rust-sdk`) is relatively young. The macro API (`#[tool]`, `#[tool_router]`,
`#[tool_handler]`) is ergonomic but has rough edges. Breaking changes between minor versions are
plausible. Pinning to a specific version and being conservative about upgrades is advisable.

**Edition 2024 / nightly risk.** Some `rmcp` versions have required Rust Edition 2024, which
historically needed nightly. Current stable (1.82+) supports edition 2021 cleanly; the design
here will target edition 2021 to stay on stable.

**No REPL / scripting.** If a user wants to add a quick custom scene, they edit TOML config and
restart. A Python server could expose a plugin API or eval endpoint. Out of scope for v1.

### 4.3 Key dependencies

| Crate | Role | Notes |
|---|---|---|
| `rmcp` | MCP server protocol | `{ version = "0.16", features = ["server", "transport-io"] }` |
| `tokio` | Async runtime | `{ features = ["full"] }` |
| `reqwest` | Cloud API HTTP client | `{ features = ["json"] }` |
| `serde` / `serde_json` | JSON serialization | Standard |
| `schemars` | JSON Schema generation for tool params | Required by rmcp macros |
| `clap` | CLI argument parsing | `{ features = ["derive", "env"] }` |
| `anyhow` / `thiserror` | Error handling | `anyhow` at tool boundary, `thiserror` for domain errors |
| `tracing` / `tracing-subscriber` | Structured logging (to stderr) | |
| `toml` | Config file parsing | |
| `async-trait` | Trait with async methods | Until async-in-traits stabilises further |

No heavy dependencies. Estimated clean build time: ~60–90s. Incremental: ~10–20s.

---

## 5. Design Discussions and Alternatives

### 5.1 Backend selection strategy

**Option A: Global flag only** (`--backend cloud|local`). Simple, but forces a choice at startup
that may not work for all devices (some may not support local).

**Option B: Per-device selection with auto-fallback (recommended).** At startup, attempt local
discovery. Devices that respond get `LocalBackend`; the rest fall back to `CloudBackend`. This is
transparent to the tool caller — they address devices by name. The `get_backend_status` tool lets
Claude report which path is being used.

**Option C: Prefer local, use cloud for writes on failure.** Adds complexity without clear benefit
— local write failures are usually a network topology issue, not a transient error that cloud would
fix.

Decision: **Option B**, with a `--backend` override flag that forces all devices to one path for
debugging.

### 5.2 Device identity and naming

Govee identifies devices by MAC address in the cloud API and by MAC + IP in the local API. Neither
is human-friendly. Options:

**Option A: Expose MAC addresses as-is.** Requires users to look up their device MACs. Terrible
UX.

**Option B: Use device names from cloud API (recommended).** The cloud API returns user-assigned
names ("Bedroom Lamp"). Use these as the canonical identifier in MCP tools, with MAC as the
internal key.

**Option C: User-defined aliases in config.** Allows overriding "H6078 Living Room" with
"couch". Additive on top of Option B. Low effort to implement, good for groups.

Decision: **Option B + C** — cloud names as default, config aliases for convenience.

### 5.3 State management

The local API has a state read mechanism (`devStatus`) but state after a command write is
unreliable for a few seconds. The cloud API state is cached and may be stale. Options:

**Option A: Always query device on `get_state`.** Accurate but slow on cloud (round-trip + Govee
cache). On local, requires waiting for device response with a timeout.

**Option B: Optimistic in-memory state (recommended).** After a successful write command, update
the internal cache immediately. Reflect this in `get_state` responses with a `source: cached`
flag. Periodically reconcile with actual device state in the background.

**Option C: No state tracking.** Every `get_state` is a live query. Simplest to implement, but
exposes all the latency and staleness issues directly to Claude.

Decision: **Option B** — optimistic cache with periodic background refresh and a `stale: bool`
field in the response.

### 5.4 MCP transport

`stdio` is the right choice for a locally-installed MCP server. Alternatives:

- **SSE / HTTP**: Required for remote deployment (e.g., running govee-mcp on a home server,
  accessed from Claude on a laptop). `rmcp` supports this. Worth adding in v2 for networked home
  automation scenarios. The H607x devices' LAN API would still work fine — govee-mcp just needs
  to be colocated on the LAN.
- **WebSocket**: Overkill for this use case.

Decision: **stdio for v1**, SSE transport in v2 config.

### 5.5 Scene representation

Built-in scenes are statically defined in the binary. Alternatives:

**Option A: Hardcoded in Rust source.** Zero deps, no config file needed for basic scenes. Easy
to document. Con: adding a scene requires a release.

**Option B: User-defined scenes in TOML config (recommended).** Scenes are name → `{brightness,
color | color_temp}` maps. The binary ships with sensible defaults baked in as fallback, but the
config file can extend or override.

**Option C: Let Claude define scenes on the fly.** Claude calls `set_color` + `set_brightness` per
device. No scene infrastructure needed. Con: verbose tool calls, no reusability across
conversations.

Decision: **Option B** with a set of well-chosen defaults (warm, focus, night, movie, bright)
compiled in.

### 5.6 Language alternatives

For completeness:

**Python + FastMCP**: Fastest to prototype, largest community, easiest for third-party
contributions. Runtime dependency (Python 3.10+), venv friction, slower binary startup. Would be
the v2 port target for marketplace reach.

**Go**: Single binary, good HTTP + UDP story, reasonable async model. No strong `rmcp`-equivalent;
would need to implement JSON-RPC manually or use a thin wrapper. Not worth it over Rust here.

**TypeScript**: Good `@modelcontextprotocol/sdk` support, familiar to web developers. Node.js
runtime dependency, worse for UDP/multicast than Rust. Not chosen.

**Raku**: Could use the raku-mcp-sdk. Novel dogfood opportunity, but the UDP/multicast story in
Raku is thinner than Rust, and the user base for Raku-distributed binaries is essentially zero.
Genuinely bad marketplace strategy.

---

## Appendix: Port and network requirements

For the local LAN backend, the host running `govee-mcp` must have:

- UDP port 4002 available for incoming discovery responses
- UDP port 4003 reachable to all Govee devices (outbound)
- Multicast traffic to `239.255.255.250` allowed on the LAN interface

Conflicts: only one process per host IP may bind the LAN API ports. Running govee-mcp alongside
Home Assistant's Govee local integration, Homebridge's Govee plugin, or govee2mqtt from the same
IP will cause port conflicts. Solutions: run govee-mcp on a separate IP (secondary NIC / Docker
macvlan), or disable the competing integration.
