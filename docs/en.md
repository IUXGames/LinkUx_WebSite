<!-- doc-shell:page slug="introduccion" -->

# Introduction

[![Godot 4](https://img.shields.io/badge/Godot-4.4+-478cbf?logo=godotengine&logoColor=white)](https://godotengine.org/)
[![Version](https://img.shields.io/badge/version-2.1.0-8435c4)](./plugin.cfg)

**LinkUx** is a multiplayer addon for **Godot 4** that unifies LAN and Online networking under a single high-level API. Instead of coding directly against ENet, Steam, or any external service, your game code only talks to LinkUx — and LinkUx handles the rest.

## Versions & compatibility

| Item | Value |
|------|--------|
| **Addon version** | **2.1.0** (`version` field in `addons/linkux/plugin.cfg`) |
| **Target Godot version** | **Godot 4.4+** |
| **Internal network protocol** | Integer from `LinkUx.get_protocol_version()` (see `addons/linkux/core/protocol_version.gd`) |
| **Addon version string** | `LinkUx.get_version()` — reads from `plugin.cfg` dynamically |

**Important:** every player must ship the **same addon build**. If protocol versions differ, you may see `PROTOCOL_VERSION_MISMATCH` (`NetworkEnums.ErrorCode`).

## Philosophy

> One public API. Multiple swappable backends.

The core principle of LinkUx is **total transport abstraction**. You can switch from LAN to Steam Online without changing a single line of gameplay code — you only change the backend.

## Layered Architecture

```
┌──────────────────────────────────────┐
│           Your game code             │  ← Only interact here
├──────────────────────────────────────┤
│         LinkUx — Public API          │  linkux.gd (Autoload)
├──────────────────────────────────────┤
│         Internal subsystems          │  SessionManager, StateReplicator,
│                                      │  RpcRelay, SceneSync, etc.
├──────────────────────────────────────┤
│          Transport layer             │  TransportLayer
├──────────────────────────────────────┤
│          Active backend              │  LAN (ENet) / Steam Online
└──────────────────────────────────────┘
```

## Available Backends

| Backend | Status | Description |
|---------|--------|-------------|
| **LAN** | ✅ Available | Local network using ENet. Room code = 8-char hex (encodes IP:port). |
| **Steam** | ✅ Available | Online multiplayer via Steam Lobbies + SteamMultiplayerPeer. Room code = 6-char alphanumeric. Requires GodotSteam GDExtension 4.4+. |

## Key Features

- **Unified API** — `create_session`, `join_session`, `close_session`, signals, RPCs: everything in the same Autoload.
- **LAN backend** — Direct local-network play with zero configuration.
- **Steam Online backend** — Internet play via Steam Lobbies; relay, NAT traversal and encryption handled by Steam.
- **Editor nodes** — `LinkUxEntity`, `LinkUxSynchronizer` and `LinkUxSpawner` configured visually in the inspector.
- **Interpolated sync** — Automatic smoothing of position/rotation for remote objects.
- **Replicated spawning** — The Spawner creates and destroys entities on all peers automatically.
- **Late join** — Players who join late automatically receive the current world state.
- **Entity authority** — Flexible model: HOST, OWNER or TRANSFERABLE.
- **Typed RPCs** — Registered message system with validation and automatic routing.
- **Built-in debug** — Log feed, network metrics and runtime state dump.

## Requirements

| Item | Required for | Notes |
|------|-------------|-------|
| **Godot 4.4+** | All | |
| **Same addon build** | All | All players must share a compatible `protocol_version`. |
| **GodotSteam GDExtension 4.4+** | Steam backend only | Plugin by [Gramps](https://godotsteam.com/). |
| **Steam client running** | Steam backend only | Must be open on the player's machine. |

---

<!-- doc-shell:page slug="instalacion" -->

# Installation

## Step 1 — Copy the addon

Copy the `addons/linkux/` folder into your Godot project:

```
your-project/
└── addons/
    └── linkux/       ← paste the entire folder here
        ├── linkux.gd
        ├── plugin.gd
        ├── core/
        ├── backends/
        └── ...
```

## Step 2 — Enable the plugin

Go to **Project → Project Settings → Plugins** and enable **LinkUx**:

```
[ LinkUx ]  [ enable ]
```

When enabled, the plugin:
1. Registers the `LinkUxEntity`, `LinkUxSynchronizer` and `LinkUxSpawner` nodes in the editor.
2. Adds the **`LinkUx`** autoload to the root scene (available globally as `LinkUx`).

## Step 3 — Verify the autoload

In **Project → Project Settings → Autoloads** you should see:

```
LinkUx   res://addons/linkux/linkux.tscn   ✓
```

> The autoload is registered automatically when the plugin is enabled. You don't need to add it manually.

## Step 4 — Initialize in your game

LinkUx **does not configure itself automatically**. You must call `LinkUx.initialize()` once at startup (typically in your own Autoload):

```gdscript
# GLOBAL.gd — Your game autoload
extends Node

func _ready() -> void:
    # Initialize Steam first if you plan to use the Steam backend
    LinkUx.initialize_steam(480)   # 480 = Spacewar (use your real App ID)

    _init_linkux()
    LinkUx.scene_load_requested.connect(_on_scene_load_requested)
    LinkUx.session_closed.connect(_on_session_closed)

func _init_linkux() -> void:
    if LinkUx.get_config() != null:
        return  # Already initialized; avoid duplicate calls
    var config := LinkUxConfig.new()
    config.network = NetworkConfig.new()
    config.network.tick_rate = 30
    config.log_level = 3  # 0=none, 1=error, 2=warn, 3=info, 4=debug
    LinkUx.initialize(config)

func _on_scene_load_requested(scene_path: String) -> void:
    get_tree().change_scene_to_file(scene_path)

func _on_session_closed() -> void:
    get_tree().change_scene_to_file("res://scenes/menu.tscn")
```

> `initialize_steam()` is only needed if you plan to use `NetworkEnums.BackendType.STEAM`. It is safe to call even if Steam is not running — it returns `false` and LinkUx continues normally.

---

<!-- doc-shell:page slug="configuracion" -->

# Configuration

LinkUx is configured via the `LinkUxConfig` resource, which groups four independent configuration classes.

## LinkUxConfig

Main resource. Created with `LinkUxConfig.new()` and passed to `LinkUx.initialize(config)`.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `default_backend` | `NetworkEnums.BackendType` | `NONE` | Backend activated when calling `initialize()`. If `NONE`, you must call `set_backend()` manually before creating or joining a session. |
| `network` | `NetworkConfig` | `null` | Network timing and optimization settings. |
| `lan` | `LanBackendConfig` | `null` | LAN backend specific configuration. |
| `steam` | `SteamBackendConfig` | `null` | Steam backend specific configuration. |
| `advanced` | `AdvancedConfig` | `null` | Advanced options. |
| `debug_enabled` | `bool` | `false` | Enables internal debug hooks. |
| `log_level` | `int` | `3` | Log verbosity level (see values below). |

**Log levels:**

| Value | Constant | Description |
|-------|----------|-------------|
| `0` | `NONE` | No logs |
| `1` | `ERROR` | Errors only |
| `2` | `WARN` | Errors and warnings |
| `3` | `INFO` | General information (recommended for production) |
| `4` | `DEBUG` | Detailed |
| `5` | `TRACE` | Very verbose (development only) |

---

## NetworkConfig

Controls network timing and transport optimizations.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `tick_rate` | `int` (1–128) | `20` | Network ticks per second. Common values: 20 (smooth action), 30 (FPS), 60 (high precision). |
| `interpolation_delay_ms` | `float` | `100.0` | Interpolation delay in ms. Increasing reduces jitter; decreasing reduces perceived latency. |
| `extrapolation_limit_ms` | `float` | `250.0` | Extrapolation limit in ms when no packets arrive. |
| `max_snapshot_buffer_size` | `int` (5–120) | `30` | Maximum snapshot buffer size for interpolation. |
| `heartbeat_interval_ms` | `float` | `5000.0` | Interval between heartbeats for disconnection detection. |
| `disconnect_timeout_ms` | `float` | `15000.0` | Time without heartbeat before considering a peer disconnected. |
| `packet_batch_enabled` | `bool` | `true` | Groups multiple packets per tick to reduce overhead. |
| `delta_compression_enabled` | `bool` | `true` | Only sends properties that changed since the last tick. |
| `max_packet_size` | `int` | `4096` | Maximum packet size in bytes. |
| `max_rpc_per_tick` | `int` (1–100) | `10` | Maximum RPCs processed per tick. |

---

## LanBackendConfig

LAN backend specific options (ENet).

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `default_port` | `int` | `7777` | ENet server base port. |
| `max_clients` | `int` | `16` | Maximum clients the server can accept. |
| `lan_port_stride` | `int` | `2` | Port increment if the base port is already in use. Allows multiple hosts on the same machine. |
| `max_lan_host_attempts` | `int` | `8` | Number of alternative ports to try when creating a server. |
| `in_bandwidth` | `int` | `0` | Maximum incoming bandwidth in bytes/s. `0` = unlimited. |
| `out_bandwidth` | `int` | `0` | Maximum outgoing bandwidth in bytes/s. `0` = unlimited. |
| `connection_timeout` | `float` | `3.0` | Maximum seconds to complete ENet client connection. If it expires, `connection_failed` is emitted. |

---

## SteamBackendConfig

Steam backend specific options.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `connection_timeout` | `float` | `10.0` | Maximum seconds to wait for `SteamMultiplayerPeer` to report `CONNECTION_CONNECTED`. If it expires, `connection_failed` is emitted. |
| `lobby_type` | `int` | `2` | Steam lobby visibility. `0` = Private, `1` = Friends Only, `2` = Public, `3` = Invisible. |

---

## AdvancedConfig

Advanced behavioral options.

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `enable_host_migration` | `bool` | `false` | Allows host migration if the original host disconnects. |
| `max_reconnect_attempts` | `int` | `3` | Automatic reconnection attempts on unexpected disconnections. |
| `reconnect_timeout_ms` | `float` | `10000.0` | Maximum time per reconnection attempt in ms. |
| `ghost_player_timeout_ms` | `float` | `30000.0` | Time before cleaning up a "ghost" peer that didn't complete the handshake. |
| `max_bandwidth_per_second` | `int` | `1024000` | Global bandwidth limit in bytes/s (~1 MB/s). |
| `max_state_updates_per_entity_per_tick` | `int` | `1` | Maximum state updates per entity per tick. |

---

## Full configuration example

```gdscript
func _init_linkux() -> void:
    var config := LinkUxConfig.new()

    # Network
    config.network = NetworkConfig.new()
    config.network.tick_rate = 30
    config.network.heartbeat_interval_ms = 3000.0
    config.network.disconnect_timeout_ms = 10000.0

    # LAN backend
    config.lan = LanBackendConfig.new()
    config.lan.default_port = 7777
    config.lan.connection_timeout = 5.0

    # Steam backend
    config.steam = SteamBackendConfig.new()
    config.steam.connection_timeout = 12.0
    config.steam.lobby_type = 2  # Public

    # Advanced
    config.advanced = AdvancedConfig.new()

    # Debug
    config.debug_enabled = true
    config.log_level = 4  # DEBUG

    LinkUx.initialize(config)
```

---

<!-- doc-shell:page slug="sesiones" -->

# Sessions

A **session** is an active game room. The player who creates it is the **host**; the rest are **clients**. All multiplayer flow revolves around the session.

## Typical lifecycle

```
set_backend()  →  create_session() / join_session_by_room_code()
      ↓
[signal session_started]
      ↓
request_scene_load()  →  report_scene_ready()  →  [signal scene_all_ready]
      ↓
[gameplay]
      ↓
close_session()  →  [signal session_closed]
```

---

## Session methods

### `LinkUx.set_backend(backend_type: int) → void`

Activates the specified network backend. Must be called before creating or joining a session.

```gdscript
LinkUx.set_backend(NetworkEnums.BackendType.LAN)    # Local network
LinkUx.set_backend(NetworkEnums.BackendType.STEAM)  # Online via Steam
```

---

### `LinkUx.create_session(session_name, max_players, metadata) → int`

Creates a new session (makes the caller the host).

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_name` | `String` | Visible room name. |
| `max_players` | `int` | Player limit (default `16`). |
| `metadata` | `Dictionary` | Optional metadata (`{"private": true}`, etc.). |

**Returns:** `NetworkEnums.ErrorCode` — `SUCCESS (0)` if successful.

```gdscript
var err := LinkUx.create_session("My Room", 4)
if err != NetworkEnums.ErrorCode.SUCCESS:
    print("Error: ", err)
```

---

### `LinkUx.join_session(session_info: SessionInfo) → int`

Joins a session using a `SessionInfo` object (obtained from a session listing).

```gdscript
var err := LinkUx.join_session(session_info)
```

---

### `LinkUx.join_session_by_room_code(room_code: String) → int`

Joins a session using the room code. Works for both LAN and Steam backends — the active backend determines the expected code format.

| Backend | Code format | Example |
|---------|-------------|---------|
| **LAN** | 8-character hex (encodes IP:port) | `A3F7KQ2P` |
| **Steam** | 6-character alphanumeric (A–Z, 0–9) | `K7PQ3A` |

```gdscript
var code := $CodeInput.text.strip_edges().to_upper()
var err := LinkUx.join_session_by_room_code(code)
if err != NetworkEnums.ErrorCode.SUCCESS:
    print("Invalid code or room not found")
```

---

### `LinkUx.close_session() → void`

Closes the active session and disconnects all peers. Emits the `session_closed` signal.

```gdscript
LinkUx.close_session()
```

---

### `LinkUx.prepare_for_new_session() → void`

Clears internal state to allow creating/joining a new session. Call this when returning to the main menu to avoid invalid state errors.

```gdscript
# In the main menu, on enter
func _ready() -> void:
    LinkUx.prepare_for_new_session()
```

---

### Session queries

| Method | Returns | Description |
|--------|---------|-------------|
| `get_current_session()` | `SessionInfo` | Active session object (`null` if no session). |
| `get_room_code()` | `String` | Room code of the current session. |
| `has_room()` | `bool` | `true` if there is an active session. |
| `is_in_session()` | `bool` | `true` if the connection is established and in session. |
| `is_host()` | `bool` | `true` if the local peer is the host. |
| `is_client()` | `bool` | `true` if the local peer is a client (not host). |
| `is_singleplayer()` | `bool` | `true` if there is only one player in the session. |
| `is_multiplayer()` | `bool` | `true` if there is more than one player. |
| `is_lan()` | `bool` | `true` if the active backend is LAN. |
| `is_online()` | `bool` | `true` if the active backend is Steam (online). |

---

## SessionInfo — Data structure

| Property | Type | Description |
|----------|------|-------------|
| `session_id` | `String` | Unique session ID. |
| `session_name` | `String` | Room name. |
| `host_peer_id` | `int` | Network ID of the host (always `1` in LAN and Steam). |
| `max_players` | `int` | Player limit configured at creation. |
| `room_code` | `String` | Alphanumeric room code. |
| `backend_data` | `Dictionary` | Backend-opaque data (e.g. IP:port in LAN, Steam lobby ID in Steam). |

---

## Error codes (NetworkEnums.ErrorCode)

| Code | Value | Description |
|------|-------|-------------|
| `SUCCESS` | `0` | Successful operation. |
| `NETWORK_UNAVAILABLE` | `101` | No network available. |
| `SESSION_NOT_FOUND` | `102` | Room code not found. |
| `SESSION_FULL` | `103` | The room is full. |
| `AUTHORITY_DENIED` | `104` | Authority operation rejected. |
| `PROTOCOL_VERSION_MISMATCH` | `106` | Incompatible protocol version. |
| `BACKEND_NOT_SET` | `113` | No backend configured before creating/joining. |
| `ALREADY_IN_SESSION` | `114` | Session already active. Call `close_session()` first. |

---

## Example — Main menu (both backends)

```gdscript
extends Control

func _ready() -> void:
    LinkUx.prepare_for_new_session()
    LinkUx.session_started.connect(_on_session_started)
    LinkUx.connection_failed.connect(_on_connection_failed)

# ── LAN ──────────────────────────────────────────────
func _on_lan_host_pressed() -> void:
    LinkUx.set_local_player_name($NicknameInput.text)
    LinkUx.set_backend(NetworkEnums.BackendType.LAN)
    LinkUx.create_session("Room of %s" % $NicknameInput.text, 4)

func _on_lan_join_pressed() -> void:
    LinkUx.set_local_player_name($NicknameInput.text)
    LinkUx.set_backend(NetworkEnums.BackendType.LAN)
    LinkUx.join_session_by_room_code($CodeInput.text.strip_edges())

# ── Steam Online ──────────────────────────────────────
func _on_online_host_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        _show_error("Steam is not running."); return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.create_session("Room of %s" % LinkUx.get_steam_user(), 8)

func _on_online_join_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        _show_error("Steam is not running."); return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.join_session_by_room_code($CodeInput.text.strip_edges().to_upper())

# ── Common ─────────────────────────────────────────────
func _on_session_started() -> void:
    $CodeLabel.text = "Code: " + LinkUx.get_room_code()
    if LinkUx.is_host():
        LinkUx.request_scene_load("res://scenes/level.tscn")

func _on_connection_failed(error: String) -> void:
    _show_error("Connection error: " + error)
```

---

<!-- doc-shell:page slug="jugadores" -->

# Players & Profiles

LinkUx maintains a registry of all connected players as `PlayerInfo` objects. You can query and modify this data at any time.

## Local player profile

### `LinkUx.set_local_player_name(display_name: String) → void`

Sets the local player's name. Call this **before** creating or joining a session.

```gdscript
LinkUx.set_local_player_name("Zara")
```

### `LinkUx.set_player_profile(display_name, metadata, data) → void`

Extended version allowing metadata and custom data.

```gdscript
LinkUx.set_player_profile(
    "Zara",
    {"avatar": "warrior"},      # metadata: visible to all
    {"score": 0, "level": 1}    # data: internal state
)
```

### `LinkUx.get_local_player_name() → String`

Returns the local player's name.

---

## Querying players

| Method | Returns | Description |
|--------|---------|-------------|
| `get_local_player()` | `PlayerInfo` | Local player data. |
| `get_player_info(peer_id)` | `PlayerInfo` | Data for a specific peer by ID. `null` if not found. |
| `get_players()` | `Array[PlayerInfo]` | All connected players (including local). |
| `get_remote_players()` | `Array[PlayerInfo]` | Remote players only (excludes local). |
| `get_host_player()` | `PlayerInfo` | The host player's data. |
| `get_client_players()` | `Array[PlayerInfo]` | Clients only (excludes host). |
| `get_local_peer_id()` | `int` | Local peer's network ID. |
| `get_connected_peers()` | `Array[int]` | IDs of all connected peers. |
| `is_local_player_id(peer_id)` | `bool` | `true` if `peer_id` belongs to the local player. |
| `is_local_player_info(info)` | `bool` | `true` if the `PlayerInfo` belongs to the local player. |
| `is_player_connected(peer_id)` | `bool` | `true` if the peer is in the active session. |

---

## Dynamic player data

### `LinkUx.update_local_player_data(key: String, value: Variant) → void`

```gdscript
LinkUx.update_local_player_data("score", 150)
LinkUx.update_local_player_data("ready", true)
```

### `LinkUx.remove_local_player_data(key: String) → void`

```gdscript
LinkUx.remove_local_player_data("ready")
```

### `LinkUx.get_player_data(peer_id) → Dictionary`

```gdscript
var data := LinkUx.get_player_data(peer_id)
print("Score: ", data.get("score", 0))
```

---

## Kicking a player

### `LinkUx.kick_player(peer_id: int, reason: String) → void`

Only the host can kick players. The kicked client receives `reason` as an error message in the `connection_failed` signal. Works identically on both LAN and Steam backends.

```gdscript
if LinkUx.is_host():
    LinkUx.kick_player(peer_id, "Inappropriate behavior")
```

---

## PlayerInfo — Data structure

| Property | Type | Description |
|----------|------|-------------|
| `peer_id` | `int` | Player's unique network ID in the session. |
| `display_name` | `String` | Player's visible name. |
| `is_host` | `bool` | `true` if this player is the host. |
| `metadata` | `Dictionary` | Metadata set in `set_player_profile`. |
| `data` | `Dictionary` | Dynamic data updatable at runtime. |

---

<!-- doc-shell:page slug="entidades" -->

# Entities — LinkUxEntity

`LinkUxEntity` is a node that turns its parent into a **replicated entity**. By adding it as a child of any node, LinkUx will begin synchronizing its properties with all peers in the session.

## When to use LinkUxEntity vs LinkUxSynchronizer

| | `LinkUxEntity` | `LinkUxSynchronizer` |
|--|--|--|
| **Interpolation** | No (remote state applied immediately) | Yes (smooth `_process` blending for non-owners) |
| **Typical use** | Host-driven props; discrete state | Characters / props with continuous motion on remote screens |
| **Property list** | `replicated_properties` (`PackedStringArray` in the inspector) | `sync_properties` (`Array[String]` + custom inspector UI) |
| **Default replication** | `ON_CHANGE` | `ALWAYS` |

Use **`LinkUxSynchronizer`** when remote players must see **smooth** motion. Use **`LinkUxEntity`** when state can **snap** (doors, switches, counters, animation triggers driven by discrete values).

### Do not stack both on the same root

`LinkUxEntity` and `LinkUxSynchronizer` each call `LinkUx.register_entity()` on their **parent** node. Adding **both** children under the same `CharacterBody3D` (or any single root) makes them fight over the same registration — expect warnings and broken replication. Pick **one component per replicated root**.

---

## Exported properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `authority_mode` | `NetworkEnums.AuthorityMode` | `HOST` | Sets LinkUx **authority policy** for the parent (`HOST`, `OWNER`, `TRANSFERABLE`). |
| `replicated_properties` | `PackedStringArray` | `[]` | Names of **parent** properties to replicate. Only simple names (`"health"`, `"global_position"`). |
| `replication_mode` | `NetworkEnums.ReplicationMode` | `ON_CHANGE` | `ALWAYS`: send every tick; `ON_CHANGE`: send when values differ; `MANUAL`: requires explicit flushing. |

### Authority modes (AuthorityMode)

| Mode | Description |
|------|-------------|
| `HOST` | The host has absolute authority. Clients receive state but cannot modify it. Ideal for world objects. |
| `OWNER` | The peer that spawned the entity has authority. Ideal for player characters. |
| `TRANSFERABLE` | Authority can be dynamically transferred between peers. Useful for pickable objects. |

---

## Example — World object controlled by host

```gdscript
# door.gd
extends StaticBody3D

var is_open: bool = false

func toggle() -> void:
    if not LinkUx.is_host():
        return  # Only the host can open/close
    is_open = not is_open
    $AnimationPlayer.play("open" if is_open else "close")
```

```
StaticBody3D (door.gd)
└── LinkUxEntity
    authority_mode: HOST
    replicated_properties: ["is_open"]
    replication_mode: ON_CHANGE
```

---

<!-- doc-shell:page slug="sincronizador" -->

# Synchronizer — LinkUxSynchronizer

`LinkUxSynchronizer` targets **continuously simulated entities** (especially characters). Compared to `LinkUxEntity`, it keeps a `_pending_state` buffer and, for non-owners, **eases** toward each network snapshot in `_process` instead of snapping instantly.

## Exported properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `sync_properties` | `Array[String]` | `[]` | Property paths watched on the **parent** of the synchronizer. Use plain names (`"velocity"`) or child paths (`"CameraPivot:rotation"`). |
| `replication_mode` | `NetworkEnums.ReplicationMode` | `ALWAYS` | Defaults to per-tick updates — ideal for motion. |
| `interpolate` | `bool` | `true` | When `true`, remote peers **lerp** vectors and **lerp_angle** rotations. |
| `remote_smoothing_hz` | `float` (2–45) | `16.0` | Blend factor per frame. Higher values track packets more aggressively; lower values smooth more. |

## `sync_properties` format

```gdscript
sync_properties = [
    "position",                        # Direct property of the parent
    "rotation",                        # Direct property of the parent
    "health",                          # Any exported property
    "Head:rotation",                   # Property of a child node named "Head"
    "WeaponHolder/Gun:position",       # Child node subpath
]
```

> **Note:** Only synchronizes properties Godot can serialize: `int`, `float`, `bool`, `String`, `Vector2`, `Vector3`, `Quaternion`, `Color`, etc.

---

## Example — Player with smooth movement

```
Player (CharacterBody3D)
└── LinkUxSynchronizer
    sync_properties: ["position", "rotation", "player_nickname", "Pivot:rotation"]
    interpolate: true
    remote_smoothing_hz: 24.0
```

```gdscript
# player.gd
extends CharacterBody3D

@export var player_peer_id: int = 0
@export var player_nickname: String = ""

func _ready() -> void:
    if not LinkUx.is_local_player_id(player_peer_id):
        set_process_unhandled_input(false)
        return
    $Camera3D.current = true

func _physics_process(_delta: float) -> void:
    if not LinkUx.is_local_player_id(player_peer_id):
        return
    var dir := Vector3(
        Input.get_axis("move_left", "move_right"),
        0,
        Input.get_axis("move_forward", "move_back")
    )
    velocity = dir.normalized() * 5.0
    move_and_slide()
```

---

<!-- doc-shell:page slug="spawner" -->

# Spawner — LinkUxSpawner

`LinkUxSpawner` replicates **scene spawn and despawn** across all peers in the session. It is the recommended system for creating players, projectiles, items and any dynamic object in multiplayer.

## Exported properties

| Property | Type | Description |
|----------|------|-------------|
| `spawn_path` | `NodePath` | Path (relative to the spawner) to the node used as parent for instances. Leave **empty** to parent new instances directly to the **spawner's own parent**. |
| `spawnable_scenes` | `Array[PackedScene]` | Scenes you can spawn by index. Keep ordering identical in every build. |

---

## Methods

### `spawn(scene_index, properties, authority_peer) → Node`

Instantiates the scene at the given index, applies the properties and replicates to all peers.

```gdscript
var player := $PlayerSpawner.spawn(
    0,  # index 0 in spawnable_scenes = player.tscn
    {
        "player_peer_id": LinkUx.get_local_peer_id(),
        "player_nickname": LinkUx.get_local_player_name(),
        "global_position": Vector3(0, 1, 0),
    },
    LinkUx.get_local_peer_id()
)
```

### `despawn(entity: Node) → void`

Removes the entity locally and notifies all peers to remove it too.

```gdscript
$PlayerSpawner.despawn(my_player)
```

---

## Automatic behavior

- **Despawn on disconnect**: When a peer disconnects, the Spawner automatically removes all entities owned by that peer.
- **Late join**: When a player joins a session in progress, the host automatically sends all existing spawns to the new player. Works on both LAN and Steam backends.

---

## Editor setup

```
Level (Node3D)
├── PlayerSpawner (LinkUxSpawner)
│   spawn_path: Players
│   spawnable_scenes: [player.tscn]
└── Players (Node3D)
```

---

<!-- doc-shell:page slug="rpc" -->

# RPC System

LinkUx provides a **remote procedure call** (RPC) system completely independent from Godot's built-in RPC system. LinkUx RPCs are routed through the `RpcRelay` and support both reliable and unreliable modes. Works identically on LAN and Steam backends.

## Send methods

```gdscript
# Send to a specific peer
LinkUx.send_rpc(peer_id, "my_method", [arg1, arg2], true)

# Send to host
LinkUx.send_rpc_to_host("my_method", [arg1])

# Send to all peers (including local)
LinkUx.broadcast_rpc("update_score", [peer_id, score])
```

### High-level shortcuts

| Method | Description |
|--------|-------------|
| `send_to_all(method, payload, reliable)` | Broadcast to everyone. |
| `send_to_host(method, payload, reliable)` | To the host (peer ID 1). |
| `send_to_player(peer_id, method, payload, reliable)` | To a specific player. |
| `send_to_clients(method, payload, reliable)` | To all clients (excludes host). |

---

## Registering handlers

```gdscript
func _ready() -> void:
    LinkUx.register_rpc("receive_message", _on_receive_message)

func _on_receive_message(from_peer: int, message: String) -> void:
    print("[%s]: %s" % [from_peer, message])

func _exit_tree() -> void:
    LinkUx.unregister_rpc("receive_message")
```

> **Note:** The first argument of the handler is always `from_peer: int` (the sender's ID), followed by the sent arguments.

---

## Reliable vs unreliable

| Parameter | Value | Description |
|-----------|-------|-------------|
| `reliable` | `true` | Guaranteed delivery and order. Use for critical events (spawn, death, score). |
| `reliable` | `false` | No delivery guarantee. Lower latency. Use for frequent position updates. |

---

<!-- doc-shell:page slug="estado-global" -->

# Global State

The **global state** is a key-value dictionary synchronized with all peers. Only the host can modify it; clients receive it automatically and can query it at any time.

## Methods

### `LinkUx.set_global_state(key: String, value: Variant) → void`

```gdscript
# Host only
if LinkUx.is_host():
    LinkUx.set_global_state("game_phase", "combat")
    LinkUx.set_global_state("time_remaining", 120)
    LinkUx.set_global_state("scores", {"Zara": 0, "Leo": 0})
```

### `LinkUx.get_global_state(key: String, default: Variant = null) → Variant`

```gdscript
var phase: String = LinkUx.get_global_state("game_phase", "waiting")
var time: int     = LinkUx.get_global_state("time_remaining", 0)
```

---

## Signal `global_state_changed`

```gdscript
func _ready() -> void:
    LinkUx.global_state_changed.connect(_on_state_changed)

func _on_state_changed(key: String, value: Variant) -> void:
    match key:
        "game_phase":
            _change_phase(value)
        "time_remaining":
            $HUD/Timer.text = str(value)
```

---

## Use cases

- **Game phases**: `"lobby"`, `"countdown"`, `"combat"`, `"end"`
- Synchronized **countdown timer**
- Global **scoreboard**
- **Match configuration**: rounds, difficulty, chosen map
- **Random seeds** for consistent procedural generation

---

<!-- doc-shell:page slug="autoridad" -->

# Entity Authority

**Authority** determines which peer controls a replicated entity. Only the peer with authority can send state updates for that entity; others receive those updates and apply them.

## Methods

### `LinkUx.set_entity_authority(entity: Node, peer_id: int) → void`

```gdscript
LinkUx.set_entity_authority($CentralObject, 2)
```

### `LinkUx.is_entity_authority(entity: Node) → bool`

```gdscript
func _physics_process(_delta: float) -> void:
    if not LinkUx.is_entity_authority(self):
        return  # Don't process if we're not the owner
    # ... physics logic
```

### `LinkUx.transfer_authority(entity: Node, to_peer_id: int) → void`

```gdscript
# Host transfers control of a vehicle to the mounting player
LinkUx.transfer_authority($Vehicle, driver_peer_id)
```

---

## Authority modes compared

| Mode | Control | Use case |
|------|---------|----------|
| `HOST` | Host always has authority. | World objects, server-controlled enemies, doors, traps. |
| `OWNER` | The peer that spawned the entity. | Player characters, player projectiles. |
| `TRANSFERABLE` | Can be passed between peers. | Vehicles, pickable items, interaction objects. |

---

<!-- doc-shell:page slug="senales" -->

# Signals

LinkUx exposes **26 signals** covering the entire multiplayer lifecycle. Connect to them to react to network events without polling.

## Session

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `session_created` | `session_info: SessionInfo` | Host has successfully created the session. |
| `session_joined` | `session_info: SessionInfo` | Client has successfully joined the session. |
| `session_started` | — | Session is ready (emitted on both host AND client). Use it to start scene loading. |
| `session_closed` | — | Session closed (by `close_session()` or host disconnect). |
| `session_ended` | — | Emitted internally when session cleanup finishes. |

## Players

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `player_joined` | `player_info: PlayerInfo` | A new peer (including yourself) entered the session. |
| `player_left` | `peer_id: int, reason: int` | A peer disconnected (before cleanup). |
| `player_left_processed` | `player_info: PlayerInfo, reason: int` | A peer disconnected (after their state was cleaned up). Use for UI. |
| `player_updated` | `player_info: PlayerInfo` | A player's data changed. |

## Connection

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `connection_failed` | `error: String` | Connection/session creation attempt failed. |
| `connection_state_changed` | `new_state: int` | Connection state changed. |
| `protocol_version_mismatch` | `local: int, remote: int` | Client has an incompatible protocol version with the host. |
| `backend_incompatible` | `reason: String` | Active backend doesn't support required capabilities. |

## Scene loading

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `scene_load_requested` | `scene_path: String` | Host requests all peers to load a scene. |
| `scene_all_ready` | `scene_path: String` | All peers reported scene ready via `report_scene_ready()`. |

## Entities and state

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `authority_changed` | `entity: Node, new_authority: int` | An entity's authority changed. |
| `global_state_changed` | `key: String, value: Variant` | A global state value changed. |

## Network

| Signal | Parameters | When emitted |
|--------|------------|--------------|
| `network_tick` | `tick_number: int, delta: float` | Every network tick. |

---

## Example — Connecting essential signals

```gdscript
func _ready() -> void:
    LinkUx.session_started.connect(_on_session_started)
    LinkUx.session_closed.connect(_on_session_closed)
    LinkUx.connection_failed.connect(_on_connection_failed)
    LinkUx.player_joined.connect(_on_player_joined)
    LinkUx.player_left_processed.connect(_on_player_left)

func _on_session_started() -> void:
    print("Session started. Host: ", LinkUx.is_host())

func _on_session_closed() -> void:
    get_tree().change_scene_to_file("res://scenes/menu.tscn")

func _on_connection_failed(error: String) -> void:
    $UI/ErrorLabel.text = "Error: " + error

func _on_player_joined(info: PlayerInfo) -> void:
    print("Player joined: ", info.display_name, " (", info.peer_id, ")")

func _on_player_left(info: PlayerInfo, _reason: int) -> void:
    print("Player left: ", info.display_name)
```

---

<!-- doc-shell:page slug="api" -->

# API Reference

Complete reference of all public methods of the `LinkUx` Autoload.

## Configuration & Steam

| Method | Returns | Description |
|--------|---------|-------------|
| `initialize(config)` | `int` | Initializes LinkUx with the given config. Returns `ErrorCode`. |
| `initialize_steam(app_id)` | `bool` | Initializes GodotSteam for the Steam backend. Writes `steam_appid.txt` automatically. Returns `true` on success. |
| `is_steam_initialized()` | `bool` | Whether Steam was successfully initialized. |
| `get_steam_user()` | `String` | Local Steam display name, or `"Player"` if unavailable. |
| `set_backend(backend_type)` | `void` | Activates the specified backend. |
| `get_config()` | `LinkUxConfig` | Active config (`null` if not initialized). |
| `get_version()` | `String` | Addon version string read from `plugin.cfg` (e.g. `"2.1.0"`). |
| `get_protocol_version()` | `int` | Internal network protocol version (integer). |
| `get_backend_type()` | `int` | Active backend type (`NetworkEnums.BackendType`). |
| `get_backend_name()` | `String` | Human-readable name of the active backend (`"LAN"` or `"Steam"`). |

## Session

| Method | Returns | Description |
|--------|---------|-------------|
| `prepare_for_new_session()` | `void` | Clears state for a new session. |
| `create_session(name, max, meta)` | `int` | Creates session as host. |
| `join_session(session_info)` | `int` | Joins as client with SessionInfo. |
| `join_session_by_room_code(code)` | `int` | Joins by room code (LAN=8 chars, Steam=6 chars). |
| `close_session()` | `void` | Closes the active session. |
| `get_current_session()` | `SessionInfo` | Active session or `null`. |
| `get_room_code()` | `String` | Active room code. |
| `has_room()` | `bool` | Is there an active session? |
| `is_in_session()` | `bool` | In session? |
| `is_host()` | `bool` | Am I host? |
| `is_client()` | `bool` | Am I client? |
| `is_singleplayer()` | `bool` | Only one player? |
| `is_multiplayer()` | `bool` | More than one player? |
| `is_lan()` | `bool` | LAN backend active? |
| `is_online()` | `bool` | Steam backend active? |

## Players

| Method | Returns | Description |
|--------|---------|-------------|
| `set_local_player_name(name)` | `void` | Local player name. |
| `set_player_profile(name, meta, data)` | `void` | Full local player profile. |
| `get_local_player_name()` | `String` | Local player name. |
| `get_local_peer_id()` | `int` | Local peer's network ID. |
| `get_local_player()` | `PlayerInfo` | Local player's PlayerInfo. |
| `get_player_info(peer_id)` | `PlayerInfo` | A peer's PlayerInfo. |
| `get_players()` | `Array[PlayerInfo]` | All players. |
| `get_remote_players()` | `Array[PlayerInfo]` | Remote players only. |
| `get_host_player()` | `PlayerInfo` | Host's PlayerInfo. |
| `get_client_players()` | `Array[PlayerInfo]` | Clients only. |
| `get_connected_peers()` | `Array[int]` | Connected peer IDs. |
| `is_local_player_id(peer_id)` | `bool` | Is local peer? |
| `is_local_player_info(info)` | `bool` | Is local PlayerInfo? |
| `is_player_connected(peer_id)` | `bool` | Is connected? |
| `kick_player(peer_id, reason)` | `void` | Kick a player (host only). |
| `update_local_player_data(key, val)` | `void` | Update local data. |
| `remove_local_player_data(key)` | `void` | Remove local data entry. |
| `set_player_data(peer, key, val)` | `bool` | Modify any player's data. |
| `get_player_data(peer_id)` | `Dictionary` | Read a player's data. |
| `remove_player_data(peer, key)` | `bool` | Remove player data entry. |

## Authority

| Method | Returns | Description |
|--------|---------|-------------|
| `set_entity_authority(entity, peer)` | `void` | Assign authority. |
| `get_entity_authority(entity)` | `int` | Returns the authority peer. |
| `is_entity_authority(entity)` | `bool` | Do I have authority over this entity? |
| `request_authority(entity)` | `void` | Request authority (TRANSFERABLE). |
| `transfer_authority(entity, peer)` | `void` | Transfer authority to another peer. |
| `validate_authority_change(entity, peer)` | `bool` | Is transfer valid? |

## State replication

| Method | Returns | Description |
|--------|---------|-------------|
| `register_entity(node, props, mode, id)` | `void` | Register entity in the replicator. |
| `unregister_entity(node)` | `void` | Unregister entity. |
| `allocate_entity_network_id()` | `int` | Generate a unique network entity ID. |
| `set_global_state(key, value)` | `void` | Write to global state (host only). |
| `get_global_state(key, default)` | `Variant` | Read from global state. |

## Scene loading

| Method | Returns | Description |
|--------|---------|-------------|
| `request_scene_load(scene_path)` | `void` | Host requests scene load for all. |
| `report_scene_ready()` | `void` | Local peer reports it loaded the scene. |
| `replay_late_join_spawns_now(peer_id)` | `void` | Force spawn replay for a peer. |
| `run_late_join_snapshot_only(peer_id)` | `void` | Send world snapshot only (no spawns). |

## RPC

| Method | Returns | Description |
|--------|---------|-------------|
| `send_rpc(peer, method, args, reliable)` | `void` | Send to a peer. |
| `send_rpc_to_host(method, args, reliable)` | `void` | Send to host. |
| `broadcast_rpc(method, args, reliable)` | `void` | Send to all. |
| `send_to_all(method, payload, reliable)` | `void` | Broadcast shortcut. |
| `send_to_host(method, payload, reliable)` | `void` | Host shortcut. |
| `send_to_player(peer, method, payload, reliable)` | `void` | Player shortcut. |
| `send_to_clients(method, payload, reliable)` | `void` | Clients shortcut. |
| `register_rpc(method, callable)` | `void` | Register handler. |
| `unregister_rpc(method)` | `void` | Unregister handler. |

## Debug

| Method | Returns | Description |
|--------|---------|-------------|
| `debug_mode(enabled)` | `void` | Enable/disable debug mode. |
| `get_debug_metrics()` | `Dictionary` | Basic metrics: state, tick, peers, backend. |
| `dump_network_state()` | `Dictionary` | Full network state dump. |
| `get_connection_state()` | `int` | Internal state machine state. |
| `get_feedback_logs(limit, min_level)` | `Array[Dictionary]` | Filtered log entries. |
| `get_logs(limit)` | `Array[Dictionary]` | All log entries. |
| `get_logs_type(level_name, limit)` | `Array[Dictionary]` | Log filtered by level. |
| `set_feedback_log_capacity(max)` | `void` | Log entry limit. |
| `clear_feedback_logs()` | `void` | Clear the log. |

---

<!-- doc-shell:page slug="debug" -->

# Debug & Diagnostics

LinkUx includes a complete diagnostics system you can use both during development and for in-game debug panels.

## Enable debug mode

```gdscript
# In configuration
config.debug_enabled = true
config.log_level = 4  # DEBUG

# Or at runtime
LinkUx.debug_mode(true)   # Enables debug and raises log to DEBUG
LinkUx.debug_mode(false)  # Disables and restores config log_level
```

---

## Basic metrics

```gdscript
var metrics := LinkUx.get_debug_metrics()
# {
#   "state":   "RUNNING",    ← Internal state machine state
#   "tick":    247,          ← Current network tick
#   "peers":   3,            ← Connected peers
#   "backend": "Steam"       ← Active backend ("LAN" or "Steam")
# }
```

---

## Full state dump

```gdscript
var state := LinkUx.dump_network_state()
print(JSON.stringify(state, "  "))
```

---

## Log system

```gdscript
# Last 50 entries of any level
var logs := LinkUx.get_logs(50)

# Errors only
var errors := LinkUx.get_logs_type("ERROR")

# INFO level and above
var info_logs := LinkUx.get_feedback_logs(0, DebugLogger.LogLevel.INFO)
```

Each log entry is a `Dictionary`:

```gdscript
{
    "level":          3,
    "level_name":     "INFO",
    "context":        "Core",
    "message":        "Initialized with backend: Steam",
    "formatted":      "[INFO][Core] Initialized with backend: Steam",
    "timestamp_msec": 12430
}
```

---

## In-game debug panel (example)

```gdscript
# debug_panel.gd
extends PanelContainer

func _process(_delta: float) -> void:
    if not visible:
        return
    var m := LinkUx.get_debug_metrics()
    $Grid/State.text   = m.get("state", "—")
    $Grid/Tick.text    = str(m.get("tick", 0))
    $Grid/Peers.text   = str(m.get("peers", 0))
    $Grid/Backend.text = m.get("backend", "—")
    $Grid/PeerID.text  = str(LinkUx.get_local_peer_id())
    $Grid/IsHost.text  = str(LinkUx.is_host())
    $Grid/Version.text = LinkUx.get_version()
```

---

<!-- doc-shell:page slug="backend-lan" -->

# LAN Backend

The LAN backend uses Godot's **ENetMultiplayerPeer** and allows local network play with direct IP and port connection. No internet connection or external accounts required.

## How it works

```
Host (ENet Server)
│
├── ENet ←→ Client 1
├── ENet ←→ Client 2
└── ENet ←→ Client 3

Star topology: clients don't connect to each other.
The host relays packets from each client to the rest.
```

## Room code

When creating a LAN session, LinkUx automatically generates an **8-character hex room code** (e.g. `"A3F7KQ2P"`). Internally, the code encodes the host's IP address and ENet port. Clients resolve the code and connect directly — no IP knowledge required.

```gdscript
func _on_session_started() -> void:
    var code := LinkUx.get_room_code()  # e.g. "A3F7KQ2P"
    $CodeLabel.text = "Room code: " + code
```

## Port stride — Multiple hosts on the same machine

For testing with multiple instances on the same PC, `LanBackendConfig.lan_port_stride` lets LinkUx try consecutive ports if the base port is already occupied.

```gdscript
config.lan = LanBackendConfig.new()
config.lan.default_port          = 7777
config.lan.lan_port_stride       = 2
config.lan.max_lan_host_attempts = 8
# Tries: 7777, 7779, 7781, 7783, 7785, 7787, 7789, 7791
```

## Connection timeout

If the ENet connection cannot be established within `connection_timeout` seconds, LinkUx emits `connection_failed`:

```gdscript
config.lan.connection_timeout = 3.0  # seconds
```

## Note on `multiplayer_poll`

LinkUx uses custom binary packets through ENet, separate from Godot's RPC system. When the LAN backend is enabled, LinkUx **automatically disables** `SceneTree.multiplayer_poll`. This is transparent — you don't need to do it manually.

## Activate the LAN backend

```gdscript
LinkUx.set_backend(NetworkEnums.BackendType.LAN)
# — or in initial config —
config.default_backend = NetworkEnums.BackendType.LAN
```

---

<!-- doc-shell:page slug="backend-steam" -->

# Steam Backend

The Steam backend enables **online multiplayer** via Valve's Steam platform. It uses **Steam Lobbies** for session discovery and **SteamMultiplayerPeer** as the transport layer. Relay, NAT traversal, and encryption are handled automatically by Steam.

## Requirements

| Item | Details |
|------|---------|
| **GodotSteam GDExtension 4.4+** | Official plugin by [Gramps](https://godotsteam.com/). Must be installed and enabled in your project. |
| **Steam client** | Must be running on the player's machine before the game starts. |
| **Valid Steam App ID** | Use `480` (Spacewar) for local testing. Use your real App ID in production. |
| **Separate Steam accounts** | Two instances with the **same Steam account** on the same machine cannot connect via P2P — this is a Steam limitation. |

## Install GodotSteam

1. Download **GodotSteam GDExtension 4.4+** from [godotsteam.com](https://godotsteam.com/).
2. Copy the addon folder into your project under `res://addons/godotsteam/`.
3. Enable the plugin in **Project → Project Settings → Plugins**.
4. Set `initialization/initialize_on_startup = false` in the `[steam]` section of `project.godot` — LinkUx handles initialization itself.

> GodotSteam registers the `Steam` global class and `SteamMultiplayerPeer` node automatically.

## Initialize Steam

Call `LinkUx.initialize_steam(app_id)` **once at startup**, before calling `LinkUx.initialize()`:

```gdscript
# GLOBAL.gd — Your game autoload
extends Node

func _ready() -> void:
    # initialize_steam() is safe to call even if Steam is not running.
    # It returns false and LinkUx continues without the Steam backend.
    LinkUx.initialize_steam(480)   # 480 = Spacewar for testing

    _init_linkux()
    LinkUx.scene_load_requested.connect(_on_scene_load_requested)
    LinkUx.session_closed.connect(_on_session_closed)

func _init_linkux() -> void:
    if LinkUx.get_config() != null:
        return
    var config := LinkUxConfig.new()
    config.network = NetworkConfig.new()
    config.network.tick_rate = 30

    config.steam = SteamBackendConfig.new()
    config.steam.connection_timeout = 10.0

    LinkUx.initialize(config)
```

`initialize_steam()` does three things automatically:
1. Creates `steam_appid.txt` in the correct path (project root in editor, executable directory in exports).
2. Calls GodotSteam's `steamInitEx(false, app_id)` to start the Steam SDK.
3. Sets `is_steam_initialized()` to `true` on success.

## Host an online session

```gdscript
func _on_online_host_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        show_error("Steam is not running.")
        return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.create_session(LinkUx.get_steam_user() + "'s Room", 8, {})
```

After `session_started`, read the **6-character room code** and display it:

```gdscript
func _on_session_started() -> void:
    var code := LinkUx.get_room_code()   # e.g. "K7PQ3A"
    $CodeLabel.text = "Room code: " + code
    if LinkUx.is_host():
        LinkUx.request_scene_load("res://scenes/level.tscn")
```

## Join an online session

```gdscript
func _on_online_join_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        show_error("Steam is not running.")
        return
    var code := $CodeInput.text.strip_edges().to_upper()
    if code.length() != 6:
        show_error("Steam room codes are 6 characters.")
        return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    var err := LinkUx.join_session_by_room_code(code)
    if err != NetworkEnums.ErrorCode.SUCCESS:
        show_error("Invalid code format.")
```

## Room codes — LAN vs Steam

| Backend | Format | Length | Encodes |
|---------|--------|--------|---------|
| **LAN** | Hexadecimal | 8 chars | Host IP + ENet port |
| **Steam** | Alphanumeric (A–Z, 0–9) | 6 chars | Steam Lobby metadata |

6 chars × 36 symbols = **~2.2 billion** unique combinations.

## How it works internally

```
HOST
  1. set_backend(STEAM) + create_session()
  2. LinkUx calls Steam.createLobby(PUBLIC, max_players)
  3. On lobby_created:
     → generates 6-char random room code
     → stores code as Steam lobby metadata (key: "linkux_room_code")
     → creates SteamMultiplayerPeer.create_host(0)
     → emits session_started

CLIENT
  1. set_backend(STEAM) + join_session_by_room_code("K7PQ3A")
  2. LinkUx calls Steam.addRequestLobbyListStringFilter("linkux_room_code", "K7PQ3A")
     → Steam.requestLobbyList()
  3. On lobby_match_list: calls Steam.joinLobby(lobbies[0])
  4. On lobby_joined:
     → gets host Steam ID via Steam.getLobbyOwner()
     → creates SteamMultiplayerPeer.create_client(host_steam_id, 0)
     → emits session_started
```

## Check the active backend

```gdscript
if LinkUx.is_online():
    print("Using Steam backend")

print(LinkUx.get_backend_name())  # "Steam"
```

## Steam-related API functions

| Method | Returns | Description |
|--------|---------|-------------|
| `initialize_steam(app_id)` | `bool` | Initializes GodotSteam. Returns `true` on success. |
| `is_steam_initialized()` | `bool` | Whether Steam was successfully initialized. |
| `get_steam_user()` | `String` | Local Steam display name, or `"Player"` if unavailable. |
| `is_online()` | `bool` | `true` if the active backend is Steam. |
| `get_version()` | `String` | Current addon version string (e.g. `"2.1.0"`). |

## Limitations

- **Same machine, same account**: Two game instances sharing the same Steam account cannot connect to each other via Steam P2P. Use separate Steam accounts for same-machine testing.
- **Steam client required**: If Steam is not running, `initialize_steam()` returns `false` and the STEAM backend is unavailable.
- **Internet required for lobby search**: Steam Lobby discovery requires an internet connection. Actual game traffic may use Steam Relay if direct P2P is unavailable.
- **`multiplayer_poll` is disabled**: Same as the LAN backend, LinkUx disables `SceneTree.multiplayer_poll` when the Steam backend is active. This is transparent.

---

<!-- doc-shell:page slug="backends" -->

# Backends

LinkUx is designed from the ground up to support multiple backends. The `NetworkBackend` base class defines the interface any backend must implement, and the rest of the system (API, subsystems, nodes) is completely transport-agnostic.

## Current status

| Backend | Status | Transport | Room code |
|---------|--------|-----------|-----------|
| **LAN (ENet)** | ✅ Available | ENetMultiplayerPeer | 8-char hex |
| **Steam Online** | ✅ Available | SteamMultiplayerPeer | 6-char alphanumeric |

## Switch backends without touching game code

This is LinkUx's core promise. Your signals, RPCs, Spawner, and Synchronizer work identically regardless of the active backend. The only change when switching is:

```gdscript
# LAN session:
LinkUx.set_backend(NetworkEnums.BackendType.LAN)
LinkUx.create_session("Room", 4)

# Steam Online session (same API, different backend):
LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
LinkUx.create_session("Room", 8)
```

## How a backend is integrated

Each backend is a script extending `NetworkBackend` that implements methods like `_backend_create_session()`, `_backend_join_session_by_room_code()`, `_backend_kick_peer()`, etc. LinkUx detects backend capabilities via `BackendCapabilityChecker`.

To add support for a new backend in the future:
1. Create the backend script under `addons/linkux/backends/your_backend/`.
2. Add the corresponding case to the `BackendType` enum in `network_enums.gd`.
3. Register the backend in `set_backend()` inside `linkux.gd`.

---

<!-- doc-shell:page slug="ejemplo-completo" -->

# Full Example

A minimal but fully functional multiplayer game using LinkUx. Includes menu with LAN and Steam Online, synchronized scene loading, player spawning and basic first-person movement.

## Project structure

```
project/
├── autoloads/
│   └── GLOBAL.gd          ← Autoload: initializes LinkUx + Steam
├── scenes/
│   ├── menu.tscn           ← Main menu
│   ├── level.tscn          ← Multiplayer level
│   └── player.tscn         ← Player prefab
└── scripts/
    ├── menu.gd
    ├── level.gd
    └── player.gd
```

---

## GLOBAL.gd — Game autoload

```gdscript
# autoloads/GLOBAL.gd
extends Node

func _ready() -> void:
    # Initialize Steam (safe to call even if Steam is not running)
    LinkUx.initialize_steam(480)

    _init_linkux()
    LinkUx.scene_load_requested.connect(_on_scene_load_requested)
    LinkUx.session_closed.connect(_on_session_closed)

func _init_linkux() -> void:
    if LinkUx.get_config() != null:
        return
    var config := LinkUxConfig.new()
    config.network = NetworkConfig.new()
    config.network.tick_rate = 30
    config.steam = SteamBackendConfig.new()
    config.log_level = 3
    LinkUx.initialize(config)

func _on_scene_load_requested(scene_path: String) -> void:
    get_tree().change_scene_to_file(scene_path)

func _on_session_closed() -> void:
    get_tree().change_scene_to_file("res://scenes/menu.tscn")
```

---

## menu.gd — Main menu

```gdscript
# scripts/menu.gd
extends Control

const DEFAULT_MAX_PLAYERS := 8

func _ready() -> void:
    LinkUx.prepare_for_new_session()
    LinkUx.session_started.connect(_on_session_started)
    LinkUx.connection_failed.connect(_on_connection_failed)

# ─── LAN ────────────────────────────────────────────────────
func _on_lan_host_pressed() -> void:
    var nickname: String = $NicknameInput.text.strip_edges()
    if nickname.is_empty(): return
    LinkUx.set_local_player_name(nickname)
    LinkUx.set_backend(NetworkEnums.BackendType.LAN)
    LinkUx.create_session("Room of " + nickname, DEFAULT_MAX_PLAYERS)

func _on_lan_join_pressed() -> void:
    var nickname: String = $NicknameInput.text.strip_edges()
    var code: String     = $CodeInput.text.strip_edges().to_upper()
    if nickname.is_empty() or code.is_empty(): return
    LinkUx.set_local_player_name(nickname)
    LinkUx.set_backend(NetworkEnums.BackendType.LAN)
    var err := LinkUx.join_session_by_room_code(code)
    if err != NetworkEnums.ErrorCode.SUCCESS:
        $StatusLabel.text = "Room not found (invalid code)"

# ─── Steam Online ────────────────────────────────────────────
func _on_online_host_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        $StatusLabel.text = "Steam is not running."; return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.create_session(LinkUx.get_steam_user() + "'s Room", DEFAULT_MAX_PLAYERS)

func _on_online_join_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        $StatusLabel.text = "Steam is not running."; return
    var code: String = $CodeInput.text.strip_edges().to_upper()
    if code.is_empty(): return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    var err := LinkUx.join_session_by_room_code(code)
    if err != NetworkEnums.ErrorCode.SUCCESS:
        $StatusLabel.text = "Invalid code format."

# ─── Common ──────────────────────────────────────────────────
func _on_session_started() -> void:
    $CodeLabel.text = "Code: " + LinkUx.get_room_code()
    if LinkUx.is_host():
        LinkUx.request_scene_load("res://scenes/level.tscn")

func _on_connection_failed(error: String) -> void:
    $StatusLabel.text = "Error: " + error
```

---

## level.gd — Game level

```gdscript
# scripts/level.gd
extends Node3D

@onready var spawner: LinkUxSpawner = $PlayerSpawner

func _ready() -> void:
    LinkUx.player_joined.connect(_on_player_joined)
    LinkUx.player_left_processed.connect(_on_player_left)

    _spawn_player(LinkUx.get_local_peer_id())
    LinkUx.report_scene_ready()

func _on_player_joined(info: PlayerInfo) -> void:
    if info.peer_id == LinkUx.get_local_peer_id():
        return
    _spawn_player(info.peer_id)

func _on_player_left(_info: PlayerInfo, _reason: int) -> void:
    pass  # LinkUxSpawner cleans up automatically

func _spawn_player(peer_id: int) -> void:
    var pos := Vector3(randf_range(-3.0, 3.0), 1.0, randf_range(-3.0, 3.0))
    var p := LinkUx.get_player_info(peer_id)
    spawner.spawn(0, {
        "player_peer_id":   peer_id,
        "player_nickname":  p.display_name if p else "?",
        "global_position":  pos,
    }, peer_id)

func _unhandled_key_input(event: InputEvent) -> void:
    if event.is_action_pressed("ui_cancel"):
        LinkUx.close_session()
```

---

## player.gd — Player character

```gdscript
# scripts/player.gd
extends CharacterBody3D

@export var player_peer_id: int = 0
@export var player_nickname: String = ""

const SPEED := 5.0
const MOUSE_SENSITIVITY := 0.002

func _ready() -> void:
    if not LinkUx.is_local_player_id(player_peer_id):
        set_process_unhandled_input(false)
        return
    $Camera3D.current = true
    Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseMotion:
        rotate_y(-event.relative.x * MOUSE_SENSITIVITY)
        $Camera3D.rotate_x(-event.relative.y * MOUSE_SENSITIVITY)
        $Camera3D.rotation.x = clamp($Camera3D.rotation.x, -1.4, 1.4)
    if event.is_action_pressed("ui_cancel"):
        Input.mouse_mode = Input.MOUSE_MODE_VISIBLE

func _physics_process(_delta: float) -> void:
    if not LinkUx.is_local_player_id(player_peer_id):
        return
    var dir := Vector3(
        Input.get_axis("move_left", "move_right"),
        0,
        Input.get_axis("move_forward", "move_back")
    )
    dir = dir.rotated(Vector3.UP, rotation.y)
    velocity = dir.normalized() * SPEED
    if not is_on_floor():
        velocity.y -= 9.8 * _delta
    move_and_slide()
```

---

## Scene trees

```
CharacterBody3D  (player.gd)
├── CollisionShape3D
├── MeshInstance3D
├── Camera3D
└── LinkUxSynchronizer
    sync_properties:     ["position", "rotation"]
    interpolate:         true
    remote_smoothing_hz: 24.0
```

```
Node3D  (level.gd)
├── WorldEnvironment
├── DirectionalLight3D
├── MeshInstance3D (floor)
├── PlayerSpawner  (LinkUxSpawner)
│   spawn_path: Players
│   spawnable_scenes: [player.tscn]
└── Players  (Node3D)
```

---

## Complete flow

```
1. Player A → create_session() (LAN or Steam)  →  [session_started]
   └── request_scene_load("level.tscn")
   └── [scene_load_requested] → GLOBAL loads the scene

2. Player B → join_session_by_room_code(code)  →  [session_started]
   └── GLOBAL receives [scene_load_requested] → loads the scene

3. Both call report_scene_ready()
   └── [scene_all_ready] → tick loop starts

4. Each level.gd spawns its local player via LinkUxSpawner
   └── Spawner replicates the spawn to all peers

5. LinkUxSynchronizer sends position/rotation every tick
   └── Remote peers receive and apply with interpolation

6. On exit → close_session() → [session_closed] → menu
```
