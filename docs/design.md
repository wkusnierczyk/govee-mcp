# `govee-mcp` — Design Document

`govee-mcp` is an MCP (Model Context Protocol) server that exposes Govee smart lighting control
to AI assistants, primarily Claude. It is a thin binary crate wrapping the `govee` library —
protocol translation only, no device logic of its own.




## Table of Contents

- [1. Goals](#1-goals)
- [2. MCP tools](#2-mcp-tools)
  - [2.1 Control tools](#21-control-tools)
  - [2.2 Scene and status tools](#22-scene-and-status-tools)
  - [2.3 What is not exposed](#23-what-is-not-exposed)
- [3. Tool design philosophy](#3-tool-design-philosophy)
- [4. Tool response format](#4-tool-response-format)
- [5. Transport](#5-transport)
- [6. Configuration](#6-configuration)
- [7. Design discussions](#7-design-discussions)
  - [7.1 Tool granularity](#71-tool-granularity)
  - [7.2 Group and `all` handling](#72-group-and-all-handling)
  - [7.3 Exposing workflow execution](#73-exposing-workflow-execution)
  - [7.4 Scene definition by Claude](#74-scene-definition-by-claude)
- [8. rmcp notes](#8-rmcp-notes)
- [9. Dependencies](#9-dependencies)
- [10. Distribution](#10-distribution)
- [11. Relationship to other consumers](#11-relationship-to-other-consumers)

---

## 1. Goals
<sub>[↑ TOC](#table-of-contents) · [2. MCP tools →](#2-mcp-tools)</sub>


- Expose the `govee` library surface as MCP tools, at the right level of abstraction for an LLM
- Operate as a locally installed stdio MCP server (Claude Desktop, Claude Code)
- Single statically linked binary, installable via `cargo install govee-mcp`
- No configuration beyond what the `govee` library already reads

---

## 2. MCP tools
<sub>[↑ TOC](#table-of-contents) · [← 1. Goals](#1-goals) · [2.1 Control tools →](#21-control-tools)</sub>


### 2.1 Control tools
<sub>[↑ TOC](#table-of-contents) · [← 2. MCP tools](#2-mcp-tools) · [2.2 Scene and status tools →](#22-scene-and-status-tools)</sub>


| Tool | Parameters | Description |
|---|---|---|
| `list_devices` | — | All devices with name, model, backend, current state |
| `get_device_state` | `name: string` | Full state for one device |
| `set_power` | `name: string, on: bool` | Turn on or off |
| `set_brightness` | `name: string, value: integer (0–100)` | Set brightness |
| `set_color` | `name: string, r: integer, g: integer, b: integer` | Set RGB colour |
| `set_color_temp` | `name: string, kelvin: integer` | Set colour temperature in Kelvin |

`name` accepts a device name, alias, group name, or the literal `"all"`.

### 2.2 Scene and status tools
<sub>[↑ TOC](#table-of-contents) · [← 2.1 Control tools](#21-control-tools) · [2.3 What is not exposed →](#23-what-is-not-exposed)</sub>


| Tool | Parameters | Description |
|---|---|---|
| `list_scenes` | — | All available scenes with colour and brightness |
| `apply_scene` | `scene: string, target?: string` | Apply a scene; default target is `"all"` |
| `get_backend_status` | — | Active backend (cloud/local) and reachability per device |

### 2.3 What is not exposed
<sub>[↑ TOC](#table-of-contents) · [← 2.2 Scene and status tools](#22-scene-and-status-tools) · [3. Tool design philosophy →](#3-tool-design-philosophy)</sub>


- MAC addresses or model strings — callers always use human names
- Raw Govee API command structures — these are library internals
- Workflow execution — deferred until the engine is implemented in `govee`
- Firmware, OTA, account management

---

## 3. Tool design philosophy
<sub>[↑ TOC](#table-of-contents) · [← 2.3 What is not exposed](#23-what-is-not-exposed) · [4. Tool response format →](#4-tool-response-format)</sub>


A one-to-one mapping of API endpoints to MCP tools is low value — Claude already understands
"turn on" and "set brightness to 80%." The tool surface is designed around **intent**:

**Names over IDs.** All tools take `name` (device name, alias, or group), not MAC addresses.
Claude can say "the bedroom lamp" and the library resolves it. No lookup step, no context leakage
of hardware identifiers into the conversation.

**Bulk operations.** `name = "all"` and group names let Claude control multiple devices in a
single tool call. Without this, a request like "set everything to movie mode" would require Claude
to call `list_devices`, then call `set_color_temp` for each result — verbose and fragile.

**Scenes as vocabulary.** `apply_scene "warm"` is a single call. The alternative — Claude
computing 2700K, 40% and calling `set_color_temp` + `set_brightness` — requires Claude to know or
guess the right values. Scenes encode opinionated, tested presets.

**LLM-appropriate responses.** Tool responses include a natural-language summary alongside the
structured data. "Turned on 3 devices (bedroom, office, cylinder)" is more useful to Claude than a
raw JSON array when deciding what to say to the user.

---

## 4. Tool response format
<sub>[↑ TOC](#table-of-contents) · [← 3. Tool design philosophy](#3-tool-design-philosophy) · [5. Transport →](#5-transport)</sub>


Each tool returns a `CallToolResult` with two parts:

1. **Structured content** (`application/json`): the raw data (device states, scene list, etc.)
2. **Text summary** (`text/plain`): a one-sentence description of what happened or was found

Example for `set_power`:

```json
{
  "content": [
    {
      "type": "text",
      "text": "Turned on 2 devices: bedroom (local), office (cloud)."
    },
    {
      "type": "resource",
      "mimeType": "application/json",
      "text": "[{\"name\": \"bedroom\", \"on\": true, ...}, ...]"
    }
  ]
}
```

Errors are returned as MCP errors (not tool result content), with a message that is useful to both
Claude and a human reading logs.

---

## 5. Transport
<sub>[↑ TOC](#table-of-contents) · [← 4. Tool response format](#4-tool-response-format) · [6. Configuration →](#6-configuration)</sub>


**v1: stdio.** The standard transport for locally installed MCP servers. No port, no auth, no
daemon. Compatible with Claude Desktop and Claude Code out of the box.

Claude Desktop configuration (macOS, `~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "govee": {
      "command": "govee-mcp",
      "env": {
        "GOVEE_API_KEY": "your-key-here"
      }
    }
  }
}
```

**v2: SSE / HTTP.** Required for running `govee-mcp` on a home server and connecting from Claude
on a different machine. `rmcp` supports SSE; this is a configuration option in v2, not a
rewrite. The local LAN backend still works — `govee-mcp` just needs to be colocated on the same
LAN as the devices.

---

## 6. Configuration
<sub>[↑ TOC](#table-of-contents) · [← 5. Transport](#5-transport) · [7. Design discussions →](#7-design-discussions)</sub>


`govee-mcp` reads `~/.config/govee/config.toml` — the same file as the `govee` library and
`govee-cli`. No separate config file. The `GOVEE_API_KEY` environment variable is the preferred
way to pass the API key when running under Claude Desktop (avoids storing the key in the shared
config file in plain text).

MCP-specific settings, if they ever arise, go in a `[mcp]` section of the shared config. None are
anticipated for v1.

---

## 7. Design discussions
<sub>[↑ TOC](#table-of-contents) · [← 6. Configuration](#6-configuration) · [7.1 Tool granularity →](#71-tool-granularity)</sub>


### 7.1 Tool granularity
<sub>[↑ TOC](#table-of-contents) · [← 7. Design discussions](#7-design-discussions) · [7.2 Group and `all` handling →](#72-group-and-all-handling)</sub>


**Option A: One tool per primitive** (`set_power`, `set_brightness`, `set_color`, `set_color_temp`
as separate tools). Clean separation, explicit parameters.

**Option B: One `set_light` tool with optional parameters.** Fewer tools in the tool list,
but Claude must know which parameters to include. More ambiguous schema.

Decision: **Option A**. Explicit tools are easier for Claude to select correctly, and MCP tool
lists are not a cost the way API endpoints are. Clear names beat conciseness.

### 7.2 Group and `all` handling
<sub>[↑ TOC](#table-of-contents) · [← 7.1 Tool granularity](#71-tool-granularity) · [7.3 Exposing workflow execution →](#73-exposing-workflow-execution)</sub>


Groups and `all` could be separate tools (`set_group_power`, `set_all_power`). Alternatively,
a single `name` parameter that the registry resolves to one device, a group, or all.

Decision: **unified `name` parameter**, resolved by the library. This matches how a human would
speak ("turn off the bedroom lights" vs "turn off all lights") and keeps the tool count low.

### 7.3 Exposing workflow execution
<sub>[↑ TOC](#table-of-contents) · [← 7.2 Group and `all` handling](#72-group-and-all-handling) · [7.4 Scene definition by Claude →](#74-scene-definition-by-claude)</sub>


`run_workflow` could be exposed as a tool so Claude can trigger complex lighting sequences by
name. Deferred until the workflow engine is implemented — exposing a stub that always errors would
confuse Claude into trying it and failing. The tool will be added in the release that ships the
engine.

### 7.4 Scene definition by Claude
<sub>[↑ TOC](#table-of-contents) · [← 7.3 Exposing workflow execution](#73-exposing-workflow-execution) · [8. rmcp notes →](#8-rmcp-notes)</sub>


Claude could construct scenes on the fly by calling `set_color_temp` + `set_brightness` per
device, without a `apply_scene` tool. This works but requires Claude to know or guess Kelvin
values and brightness levels that produce good results. `apply_scene` with tested presets is
strictly more reliable for common cases, while `set_color_temp` remains available for custom
requests.

---

## 8. rmcp notes
<sub>[↑ TOC](#table-of-contents) · [← 7.4 Scene definition by Claude](#74-scene-definition-by-claude) · [9. Dependencies →](#9-dependencies)</sub>


The official Rust MCP SDK (`rmcp`, `modelcontextprotocol/rust-sdk`) is at v0.16 as of March 2026.
The macro API (`#[tool]`, `#[tool_router]`, `#[tool_handler]`) is ergonomic and eliminates
boilerplate, but the crate is young and minor-version breaks are plausible. Pin to a specific
version in `Cargo.toml` and upgrade deliberately.

Some earlier versions required Rust Edition 2024 / nightly. This project targets edition 2021 on
stable Rust. Verify the pinned version compiles on stable before committing.

---

## 9. Dependencies
<sub>[↑ TOC](#table-of-contents) · [← 8. rmcp notes](#8-rmcp-notes) · [10. Distribution →](#10-distribution)</sub>


| Crate | Role |
|---|---|
| `govee` | All device control logic |
| `rmcp` | MCP server protocol (`features = ["server", "transport-io"]`) |
| `schemars` | JSON Schema generation for tool parameters (required by rmcp macros) |
| `serde` / `serde_json` | Parameter deserialization and response serialization |
| `tokio` | Async runtime |
| `anyhow` | Error handling at binary boundary |
| `tracing-subscriber` | Log output to stderr (not stdout — stdout is the MCP transport) |

---

## 10. Distribution
<sub>[↑ TOC](#table-of-contents) · [← 9. Dependencies](#9-dependencies) · [11. Relationship to other consumers →](#11-relationship-to-other-consumers)</sub>


Published to crates.io as `govee-mcp`. GitHub releases provide pre-built binaries for Linux
x86_64, Linux aarch64, and macOS arm64. The binary is the entire deliverable.

```sh
cargo install govee-mcp
```

No runtime, no config file required for basic use (cloud backend with API key from env var), no
install script. Claude Desktop picks it up from `PATH` after installation.

---

## 11. Relationship to other consumers
<sub>[↑ TOC](#table-of-contents) · [← 10. Distribution](#10-distribution)</sub>


`govee-mcp` and `govee-server` may run simultaneously — they share no ports and do not conflict.
Both maintain their own `DeviceRegistry` as separate processes. If both use the local LAN backend
from the same host IP, they will conflict on UDP port 4002 — a hard constraint of the Govee LAN
protocol. One must use `backend = "cloud"`, or they must run on separate IPs. Both binaries detect
this at startup and emit a clear error rather than failing silently.
