<!-- doc-shell:page slug="introduccion" -->

# Introducción

[![Godot 4](https://img.shields.io/badge/Godot-4.4+-478cbf?logo=godotengine&logoColor=white)](https://godotengine.org/)
[![Version](https://img.shields.io/badge/version-2.1.0-8435c4)](./plugin.cfg)

**LinkUx** es un addon de multijugador para **Godot 4** que unifica redes LAN y Online bajo una única API de alto nivel. En lugar de programar directamente contra ENet, Steam o cualquier servicio externo, tu código de juego solo habla con LinkUx — y LinkUx se encarga del resto.

## Versiones y compatibilidad

| Información | Valor |
|-------------|--------|
| **Versión del addon** | **2.1.0** (campo `version` en `addons/linkux/plugin.cfg`) |
| **Godot para el que está diseñado** | **Godot 4.4+** |
| **Protocolo de red interno** | Número entero expuesto por `LinkUx.get_protocol_version()` (ver `addons/linkux/core/protocol_version.gd`) |
| **Versión del addon (string)** | `LinkUx.get_version()` — lee de `plugin.cfg` dinámicamente |

**Importante:** todos los jugadores deben usar la **misma versión del addon** en sus proyectos. Si un cliente tiene otra versión incompatible del protocolo, la conexión puede fallar con `PROTOCOL_VERSION_MISMATCH` (`NetworkEnums.ErrorCode`).

## Filosofía

> Una sola API pública. Múltiples backends intercambiables.

El principio central de LinkUx es la **abstracción total del transporte**. Puedes cambiar de LAN a Steam Online sin tocar ni una línea de código de juego — solo cambias la configuración del backend.

## Arquitectura en capas

```
┌──────────────────────────────────────┐
│         Tu código de juego           │  ← Solo interactúa aquí
├──────────────────────────────────────┤
│         LinkUx — API pública         │  linkux.gd (Autoload)
├──────────────────────────────────────┤
│         Subsistemas internos         │  SessionManager, StateReplicator,
│                                      │  RpcRelay, SceneSync, etc.
├──────────────────────────────────────┤
│         Capa de transporte           │  TransportLayer
├──────────────────────────────────────┤
│         Backend activo               │  LAN (ENet) / Steam Online
└──────────────────────────────────────┘
```

## Backends disponibles

| Backend | Estado | Descripción |
|---------|--------|-------------|
| **LAN** | ✅ Disponible | Red local usando ENet. Código de sala = 8 caracteres hex (codifica IP:puerto). |
| **Steam** | ✅ Disponible | Multijugador online via Steam Lobbies + SteamMultiplayerPeer. Código de sala = 6 caracteres alfanuméricos. Requiere GodotSteam GDExtension 4.4+. |

## Características principales

- **API unificada** — `create_session`, `join_session`, `close_session`, señales, RPCs: todo en el mismo Autoload.
- **Backend LAN** — Juego en red local con configuración mínima.
- **Backend Steam Online** — Juego online via Steam Lobbies; relay, NAT traversal y cifrado gestionados por Steam.
- **Nodos de editor** — `LinkUxEntity`, `LinkUxSynchronizer` y `LinkUxSpawner` se configuran visualmente en el inspector.
- **Sincronización con interpolación** — Suavizado automático de posición/rotación para objetos remotos.
- **Spawning replicado** — El Spawner crea y destruye entidades en todos los peers de forma automática.
- **Late join** — Los jugadores que se unen tarde reciben el estado actual del mundo automáticamente.
- **Autoridad de entidades** — Modelo flexible: HOST, OWNER o TRANSFERABLE.
- **RPCs tipados** — Sistema de mensajes registrados con validación y routing automático.
- **Debug integrado** — Feed de logs, métricas de red y volcado de estado en tiempo de ejecución.

## Requisitos

| Elemento | Requerido para | Notas |
|----------|---------------|-------|
| **Godot 4.4+** | Todo | |
| **Misma versión del addon** | Todo | Todos los jugadores deben compartir un `protocol_version` compatible. |
| **GodotSteam GDExtension 4.4+** | Solo backend Steam | Plugin oficial de [Gramps](https://godotsteam.com/). |
| **Cliente Steam en ejecución** | Solo backend Steam | Debe estar abierto en la máquina del jugador. |

---

<!-- doc-shell:page slug="instalacion" -->

# Instalación

## Paso 1 — Copiar el addon

Copia la carpeta `addons/linkux/` dentro de tu proyecto Godot:

```
tu-proyecto/
└── addons/
    └── linkux/       ← pega aquí la carpeta completa
        ├── linkux.gd
        ├── plugin.gd
        ├── core/
        ├── backends/
        └── ...
```

## Paso 2 — Activar el plugin

Ve a **Project → Project Settings → Plugins** y activa **LinkUx**:

```
[ LinkUx ]  [ enable ]
```

Al activarlo, el plugin:
1. Registra los nodos `LinkUxEntity`, `LinkUxSynchronizer` y `LinkUxSpawner` en el editor.
2. Añade el autoload **`LinkUx`** a la escena raíz (disponible globalmente como `LinkUx`).

## Paso 3 — Verificar el autoload

En **Project → Project Settings → Autoloads** deberías ver:

```
LinkUx   res://addons/linkux/linkux.tscn   ✓
```

> El autoload se registra automáticamente al activar el plugin. No necesitas añadirlo manualmente.

## Paso 4 — Inicializar en tu juego

LinkUx **no se configura automáticamente**. Debes llamar a `LinkUx.initialize()` una vez al arrancar tu juego (normalmente en un Autoload propio):

```gdscript
# GLOBAL.gd — Tu autoload de juego
extends Node

func _ready() -> void:
    # Inicializar Steam si planeas usar el backend Steam
    LinkUx.initialize_steam(480)   # 480 = Spacewar (usa tu App ID real)

    _init_linkux()
    LinkUx.scene_load_requested.connect(_on_scene_load_requested)
    LinkUx.session_closed.connect(_on_session_closed)

func _init_linkux() -> void:
    if LinkUx.get_config() != null:
        return  # Ya inicializado; evita reinicializaciones duplicadas
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

> `initialize_steam()` solo es necesario si vas a usar `NetworkEnums.BackendType.STEAM`. Es seguro llamarlo aunque Steam no esté en ejecución — retorna `false` y LinkUx continúa normalmente.

---

<!-- doc-shell:page slug="configuracion" -->

# Configuración

LinkUx se configura mediante el recurso `LinkUxConfig`, que agrupa cuatro clases de configuración independientes.

## LinkUxConfig

Recurso principal. Se crea con `LinkUxConfig.new()` y se pasa a `LinkUx.initialize(config)`.

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `default_backend` | `NetworkEnums.BackendType` | `NONE` | Backend que se activa al llamar a `initialize()`. Si es `NONE`, debes llamar a `set_backend()` manualmente antes de crear o unirte a una sesión. |
| `network` | `NetworkConfig` | `null` | Ajustes de timing y optimización de red. |
| `lan` | `LanBackendConfig` | `null` | Configuración específica del backend LAN. |
| `steam` | `SteamBackendConfig` | `null` | Configuración específica del backend Steam. |
| `advanced` | `AdvancedConfig` | `null` | Opciones avanzadas. |
| `debug_enabled` | `bool` | `false` | Activa hooks internos de debug. |
| `log_level` | `int` | `3` | Nivel de verbosidad del log (ver valores abajo). |

**Niveles de log:**

| Valor | Constante | Descripción |
|-------|-----------|-------------|
| `0` | `NONE` | Sin logs |
| `1` | `ERROR` | Solo errores |
| `2` | `WARN` | Errores y advertencias |
| `3` | `INFO` | Información general (recomendado en producción) |
| `4` | `DEBUG` | Detallado |
| `5` | `TRACE` | Muy verboso (solo desarrollo) |

---

## NetworkConfig

Controla el timing de red y las optimizaciones de transporte.

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `tick_rate` | `int` (1–128) | `20` | Ticks de red por segundo. Valores comunes: 20 (acción suave), 30 (FPS), 60 (alta precisión). |
| `interpolation_delay_ms` | `float` | `100.0` | Retraso de interpolación en ms. Aumentar reduce el jitter; disminuir reduce la latencia perceptible. |
| `extrapolation_limit_ms` | `float` | `250.0` | Límite de extrapolación en ms cuando no llegan paquetes. |
| `max_snapshot_buffer_size` | `int` (5–120) | `30` | Tamaño máximo del buffer de snapshots para interpolación. |
| `heartbeat_interval_ms` | `float` | `5000.0` | Intervalo entre heartbeats para detectar desconexiones. |
| `disconnect_timeout_ms` | `float` | `15000.0` | Tiempo sin heartbeat antes de considerar un peer desconectado. |
| `packet_batch_enabled` | `bool` | `true` | Agrupa múltiples paquetes por tick para reducir overhead. |
| `delta_compression_enabled` | `bool` | `true` | Solo envía propiedades que cambiaron desde el último tick. |
| `max_packet_size` | `int` | `4096` | Tamaño máximo de un paquete en bytes. |
| `max_rpc_per_tick` | `int` (1–100) | `10` | Máximo de RPCs que se procesan por tick. |

---

## LanBackendConfig

Opciones específicas del backend LAN (ENet).

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `default_port` | `int` | `7777` | Puerto base del servidor ENet. |
| `max_clients` | `int` | `16` | Máximo de clientes que puede aceptar el servidor. |
| `lan_port_stride` | `int` | `2` | Incremento de puerto si el puerto base ya está ocupado. Permite múltiples hosts en la misma máquina. |
| `max_lan_host_attempts` | `int` | `8` | Número de puertos alternativos a intentar al crear un servidor. |
| `in_bandwidth` | `int` | `0` | Ancho de banda de entrada máximo en bytes/s. `0` = ilimitado. |
| `out_bandwidth` | `int` | `0` | Ancho de banda de salida máximo en bytes/s. `0` = ilimitado. |
| `connection_timeout` | `float` | `3.0` | Segundos máximos para completar la conexión ENet del cliente. Si expira, se emite `connection_failed`. |

---

## SteamBackendConfig

Opciones específicas del backend Steam.

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `connection_timeout` | `float` | `10.0` | Segundos máximos para que `SteamMultiplayerPeer` reporte `CONNECTION_CONNECTED`. Si expira, se emite `connection_failed`. |
| `lobby_type` | `int` | `2` | Visibilidad del lobby de Steam. `0` = Privado, `1` = Solo amigos, `2` = Público, `3` = Invisible. |

---

## AdvancedConfig

Opciones avanzadas de comportamiento.

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `enable_host_migration` | `bool` | `false` | Permite migrar el host si el host original se desconecta. |
| `max_reconnect_attempts` | `int` | `3` | Intentos de reconexión automática ante desconexiones inesperadas. |
| `reconnect_timeout_ms` | `float` | `10000.0` | Tiempo máximo por intento de reconexión en ms. |
| `ghost_player_timeout_ms` | `float` | `30000.0` | Tiempo antes de limpiar un peer "fantasma" que no completó la handshake. |
| `max_bandwidth_per_second` | `int` | `1024000` | Límite global de ancho de banda en bytes/s (~1 MB/s). |
| `max_state_updates_per_entity_per_tick` | `int` | `1` | Máximo de actualizaciones de estado por entidad y tick. |

---

## Ejemplo completo de configuración

```gdscript
func _init_linkux() -> void:
    var config := LinkUxConfig.new()

    # Red
    config.network = NetworkConfig.new()
    config.network.tick_rate = 30
    config.network.heartbeat_interval_ms = 3000.0
    config.network.disconnect_timeout_ms = 10000.0

    # LAN
    config.lan = LanBackendConfig.new()
    config.lan.default_port = 7777
    config.lan.connection_timeout = 5.0

    # Steam
    config.steam = SteamBackendConfig.new()
    config.steam.connection_timeout = 12.0
    config.steam.lobby_type = 2  # Público

    # Avanzado
    config.advanced = AdvancedConfig.new()

    # Debug
    config.debug_enabled = true
    config.log_level = 4  # DEBUG

    LinkUx.initialize(config)
```

---

<!-- doc-shell:page slug="sesiones" -->

# Sesiones

Una **sesión** es una sala de juego activa. El jugador que la crea es el **host**; el resto son **clientes**. Todo el flujo multijugador gira alrededor de la sesión.

## Ciclo de vida típico

```
set_backend()  →  create_session() / join_session_by_room_code()
      ↓
[señal session_started]
      ↓
request_scene_load()  →  report_scene_ready()  →  [señal scene_all_ready]
      ↓
[gameplay]
      ↓
close_session()  →  [señal session_closed]
```

---

## Métodos de sesión

### `LinkUx.set_backend(backend_type: int) → void`

Activa el backend de red especificado. Debe llamarse antes de crear o unirse a una sesión.

```gdscript
LinkUx.set_backend(NetworkEnums.BackendType.LAN)    # Red local
LinkUx.set_backend(NetworkEnums.BackendType.STEAM)  # Online via Steam
```

---

### `LinkUx.create_session(session_name, max_players, metadata) → int`

Crea una sesión nueva (convierte al llamador en host).

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| `session_name` | `String` | Nombre visible de la sala. |
| `max_players` | `int` | Límite de jugadores (default `16`). |
| `metadata` | `Dictionary` | Metadatos opcionales (`{"private": true}`, etc.). |

**Retorna:** `NetworkEnums.ErrorCode` — `SUCCESS (0)` si todo fue bien.

```gdscript
var err := LinkUx.create_session("Mi Sala", 4)
if err != NetworkEnums.ErrorCode.SUCCESS:
    print("Error: ", err)
```

---

### `LinkUx.join_session(session_info: SessionInfo) → int`

Se une a una sesión usando un objeto `SessionInfo` (obtenido de un listado de sesiones).

```gdscript
var err := LinkUx.join_session(session_info)
```

---

### `LinkUx.join_session_by_room_code(room_code: String) → int`

Se une a una sesión usando el código de sala. Funciona con ambos backends — el backend activo determina el formato esperado del código.

| Backend | Formato | Ejemplo |
|---------|---------|---------|
| **LAN** | 8 caracteres hex (codifica IP:puerto) | `A3F7KQ2P` |
| **Steam** | 6 caracteres alfanuméricos (A–Z, 0–9) | `K7PQ3A` |

```gdscript
var code := $CodeInput.text.strip_edges().to_upper()
var err := LinkUx.join_session_by_room_code(code)
if err != NetworkEnums.ErrorCode.SUCCESS:
    print("Código inválido o sala no encontrada")
```

---

### `LinkUx.close_session() → void`

Cierra la sesión activa y desconecta a todos los peers. Emite la señal `session_closed`.

```gdscript
LinkUx.close_session()
```

---

### `LinkUx.prepare_for_new_session() → void`

Limpia el estado interno para permitir crear/unirse a una nueva sesión. Llama a esto cuando vuelves al menú principal para evitar errores de estado inválido.

```gdscript
# En el menú principal, al entrar
func _ready() -> void:
    LinkUx.prepare_for_new_session()
```

---

### Consultas de sesión

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `get_current_session()` | `SessionInfo` | Objeto de la sesión activa (`null` si no hay sesión). |
| `get_room_code()` | `String` | Código de sala de la sesión actual. |
| `has_room()` | `bool` | `true` si hay una sesión activa. |
| `is_in_session()` | `bool` | `true` si la conexión está establecida y en sesión. |
| `is_host()` | `bool` | `true` si el peer local es el host. |
| `is_client()` | `bool` | `true` si el peer local es cliente (no host). |
| `is_singleplayer()` | `bool` | `true` si solo hay un jugador en la sesión. |
| `is_multiplayer()` | `bool` | `true` si hay más de un jugador. |
| `is_lan()` | `bool` | `true` si el backend activo es LAN. |
| `is_online()` | `bool` | `true` si el backend activo es Steam (online). |

---

## SessionInfo — Estructura de datos

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `session_id` | `String` | ID único de la sesión. |
| `session_name` | `String` | Nombre de la sala. |
| `host_peer_id` | `int` | ID de red del host (siempre `1` en LAN y Steam). |
| `max_players` | `int` | Límite de jugadores configurado al crear. |
| `room_code` | `String` | Código alfanumérico de sala. |
| `backend_data` | `Dictionary` | Datos opacos del backend (p.ej. IP:puerto en LAN, Steam lobby ID en Steam). |

---

## Códigos de error (NetworkEnums.ErrorCode)

| Código | Valor | Descripción |
|--------|-------|-------------|
| `SUCCESS` | `0` | Operación exitosa. |
| `NETWORK_UNAVAILABLE` | `101` | Sin red disponible. |
| `SESSION_NOT_FOUND` | `102` | Código de sala no encontrado. |
| `SESSION_FULL` | `103` | La sala está llena. |
| `AUTHORITY_DENIED` | `104` | Operación de autoridad rechazada. |
| `PROTOCOL_VERSION_MISMATCH` | `106` | Versión de protocolo incompatible. |
| `BACKEND_NOT_SET` | `113` | No se ha configurado un backend antes de crear/unirse. |
| `ALREADY_IN_SESSION` | `114` | Ya hay una sesión activa. Llama a `close_session()` primero. |

---

## Ejemplo — Menú principal (ambos backends)

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
    LinkUx.create_session("Sala de %s" % $NicknameInput.text, 4)

func _on_lan_join_pressed() -> void:
    LinkUx.set_local_player_name($NicknameInput.text)
    LinkUx.set_backend(NetworkEnums.BackendType.LAN)
    LinkUx.join_session_by_room_code($CodeInput.text.strip_edges())

# ── Steam Online ──────────────────────────────────────
func _on_online_host_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        _show_error("Steam no está en ejecución."); return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.create_session("Sala de %s" % LinkUx.get_steam_user(), 8)

func _on_online_join_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        _show_error("Steam no está en ejecución."); return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.join_session_by_room_code($CodeInput.text.strip_edges().to_upper())

# ── Común ──────────────────────────────────────────────
func _on_session_started() -> void:
    $CodeLabel.text = "Código: " + LinkUx.get_room_code()
    if LinkUx.is_host():
        LinkUx.request_scene_load("res://scenes/level.tscn")

func _on_connection_failed(error: String) -> void:
    _show_error("Error de conexión: " + error)
```

---

<!-- doc-shell:page slug="jugadores" -->

# Jugadores y Perfiles

LinkUx mantiene un registro de todos los jugadores conectados como objetos `PlayerInfo`. Puedes consultar y modificar estos datos en cualquier momento.

## Perfil del jugador local

### `LinkUx.set_local_player_name(display_name: String) → void`

Establece el nombre del jugador local. Llama a esto **antes** de crear o unirte a una sesión.

```gdscript
LinkUx.set_local_player_name("Zara")
```

### `LinkUx.set_player_profile(display_name, metadata, data) → void`

Versión extendida que permite añadir metadatos y datos personalizados.

```gdscript
LinkUx.set_player_profile(
    "Zara",
    {"avatar": "warrior"},      # metadata: visible para todos
    {"score": 0, "level": 1}    # data: estado interno
)
```

### `LinkUx.get_local_player_name() → String`

Retorna el nombre del jugador local.

---

## Consultar jugadores

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `get_local_player()` | `PlayerInfo` | Datos del jugador local. |
| `get_player_info(peer_id)` | `PlayerInfo` | Datos de un peer específico por ID. `null` si no existe. |
| `get_players()` | `Array[PlayerInfo]` | Todos los jugadores conectados (incluido el local). |
| `get_remote_players()` | `Array[PlayerInfo]` | Solo los jugadores remotos (excluye al local). |
| `get_host_player()` | `PlayerInfo` | Datos del jugador que es host. |
| `get_client_players()` | `Array[PlayerInfo]` | Solo los clientes (excluye al host). |
| `get_local_peer_id()` | `int` | ID de red del peer local. |
| `get_connected_peers()` | `Array[int]` | IDs de todos los peers conectados. |
| `is_local_player_id(peer_id)` | `bool` | `true` si el `peer_id` corresponde al jugador local. |
| `is_local_player_info(info)` | `bool` | `true` si el `PlayerInfo` corresponde al jugador local. |
| `is_player_connected(peer_id)` | `bool` | `true` si el peer está en la sesión activa. |

---

## Datos dinámicos por jugador

Puedes almacenar datos personalizados en cada jugador en tiempo de ejecución:

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
print("Puntuación: ", data.get("score", 0))
```

---

## Patear a un jugador

### `LinkUx.kick_player(peer_id: int, reason: String) → void`

Solo el host puede patear jugadores. El cliente pateado recibe `reason` como mensaje de error en la señal `connection_failed`. Funciona de forma idéntica en los backends LAN y Steam.

```gdscript
if LinkUx.is_host():
    LinkUx.kick_player(peer_id, "Comportamiento inapropiado")
```

---

## PlayerInfo — Estructura de datos

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `peer_id` | `int` | ID de red único del jugador en la sesión. |
| `display_name` | `String` | Nombre visible del jugador. |
| `is_host` | `bool` | `true` si este jugador es el host. |
| `metadata` | `Dictionary` | Metadatos configurados en `set_player_profile`. |
| `data` | `Dictionary` | Datos dinámicos actualizables en tiempo real. |

---

## Ejemplo — Mostrar lista de jugadores

```gdscript
func _actualizar_lista_jugadores() -> void:
    for info in LinkUx.get_players():
        var etiqueta := Label.new()
        var prefijo := "[HOST] " if info.is_host else ""
        var tuyo := " (tú)" if LinkUx.is_local_player_info(info) else ""
        etiqueta.text = prefijo + info.display_name + tuyo
        $ListaJugadores.add_child(etiqueta)
```

---

<!-- doc-shell:page slug="entidades" -->

# Entidades — LinkUxEntity

`LinkUxEntity` es un nodo que convierte a su padre en una **entidad replicada**. Al agregarlo como hijo de cualquier nodo, LinkUx comenzará a sincronizar sus propiedades con todos los peers de la sesión.

## Cuándo usar LinkUxEntity vs LinkUxSynchronizer

| | `LinkUxEntity` | `LinkUxSynchronizer` |
|--|--|--|
| **Interpolación** | No (el estado remoto se aplica directamente) | Sí (suavizado en `_process` para peers que no son dueños) |
| **Uso típico** | Objetos que el host controla por completo; estados discretos | Personajes y props con movimiento continuo visto en remoto |
| **Lista de propiedades** | `replicated_properties` (`PackedStringArray` en el inspector) | `sync_properties` (`Array[String]`, más editor personalizado) |
| **Modo de replicación por defecto** | `ON_CHANGE` | `ALWAYS` |

Usa **`LinkUxSynchronizer`** para personajes u objetos donde los clientes deben ver **movimiento suave**. Usa **`LinkUxEntity`** para objetos cuyo estado puede aplicarse **a saltos** (puertas, interruptores, contadores).

### No combines ambos en el mismo nodo raíz

`LinkUxEntity` y `LinkUxSynchronizer` registran al **nodo padre** con `LinkUx.register_entity(...)`. Si añades **los dos como hijos del mismo** `CharacterBody3D`, ambos intentarán registrar las mismas rutas: obtendrás comportamiento imprevisto y advertencias en consola. Elige **uno u otro** por entidad replicada.

---

## Propiedades exportadas

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `authority_mode` | `NetworkEnums.AuthorityMode` | `HOST` | Define quién tiene la **autoridad** lógica del nodo padre ante LinkUx. |
| `replicated_properties` | `PackedStringArray` | `[]` | Nombres de propiedades del **padre** que se envían por red. |
| `replication_mode` | `NetworkEnums.ReplicationMode` | `ON_CHANGE` | `ALWAYS`: cada tick; `ON_CHANGE`: solo si el valor cambió; `MANUAL`: solo cuando se invoca explícitamente. |

### Modos de autoridad (AuthorityMode)

| Modo | Descripción |
|------|-------------|
| `HOST` | El host tiene autoridad absoluta. Ideal para objetos del mundo (puertas, proyectiles del servidor). |
| `OWNER` | El peer que spawneó la entidad tiene autoridad. Ideal para personajes de jugador. |
| `TRANSFERABLE` | La autoridad puede transferirse dinámicamente entre peers. Útil para objetos recogibles. |

---

## Ejemplo — Objeto del mundo controlado por el host

```gdscript
# door.gd
extends StaticBody3D

var is_open: bool = false

func toggle() -> void:
    if not LinkUx.is_host():
        return  # Solo el host puede abrir/cerrar
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

# Sincronizador — LinkUxSynchronizer

`LinkUxSynchronizer` es el componente de sincronización pensado para **entidades con movimiento continuo** (sobre todo personajes). A diferencia de `LinkUxEntity`, aplica **interpolación** en clientes cuando el estado no es del jugador local: los valores de red no se copian de golpe, sino que convergen suavemente en `_process`.

## Propiedades exportadas

| Propiedad | Tipo | Por defecto | Descripción |
|-----------|------|-------------|-------------|
| `sync_properties` | `Array[String]` | `[]` | Lista de propiedades que el `StateReplicator` observará en el **padre** del Synchronizer. Acepta propiedades del padre (`"position"`) o de nodos hijos con la forma `"Camino/Del/Nodo:propiedad"`. |
| `replication_mode` | `NetworkEnums.ReplicationMode` | `ALWAYS` | Por defecto reenvía cada tick (adecuado a movimiento). |
| `interpolate` | `bool` | `true` | Si es `true`, los peers **que no son dueños** mezclan el estado recibido con `lerp` / `lerp_angle`. |
| `remote_smoothing_hz` | `float` (2–45) | `16.0` | Factor de suavizado por frame. Valores **más altos** siguen más de cerca el último paquete; valores **más bajos** suavizan más. |

## Formato de `sync_properties`

```gdscript
sync_properties = [
    "position",                        # Propiedad directa del padre
    "rotation",                        # Propiedad directa del padre
    "health",                          # Cualquier propiedad exportada
    "Head:rotation",                   # Propiedad de un nodo hijo llamado "Head"
    "WeaponHolder/Gun:position",       # Subruta de nodo hijo
]
```

> **Nota:** Solo sincroniza propiedades que Godot pueda serializar: `int`, `float`, `bool`, `String`, `Vector2`, `Vector3`, `Quaternion`, `Color`, etc.

---

## Añadir propiedades desde el inspector

El addon sustituye el editor por defecto de `sync_properties` por un **panel personalizado**:

1. Selecciona el nodo **`LinkUxSynchronizer`** en el árbol de escena.
2. En el inspector verás la sección **"Propiedades sincronizadas"** / *Synchronized Properties*.
3. Pulsa **`Añadir propiedad sincronizada`** para abrir el selector.
4. Elige el nodo y la propiedad — doble clic para confirmar.

---

## Ejemplo — Jugador con movimiento suave

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

`LinkUxSpawner` replica el **spawn y despawn de escenas** en todos los peers de la sesión. Es el sistema recomendado para crear jugadores, proyectiles, ítems y cualquier objeto dinámico en multijugador.

## Propiedades exportadas

| Propiedad | Tipo | Descripción |
|-----------|------|-------------|
| `spawn_path` | `NodePath` | Ruta al nodo bajo el que se colocarán las instancias. Si se deja **vacío**, las escenas se cuelgan directamente del **padre del Spawner**. |
| `spawnable_scenes` | `Array[PackedScene]` | Escenas permitidas. El índice que pasas a `spawn()` corresponde a la posición en este array. Mantén órdenes idénticos en todas las máquinas. |

---

## Métodos

### `spawn(scene_index, properties, authority_peer) → Node`

Instancia la escena en el índice dado, aplica las propiedades y replica a todos los peers.

```gdscript
var jugador := $PlayerSpawner.spawn(
    0,  # índice 0 en spawnable_scenes = player.tscn
    {
        "player_peer_id": LinkUx.get_local_peer_id(),
        "player_nickname": LinkUx.get_local_player_name(),
        "global_position": Vector3(0, 1, 0),
    },
    LinkUx.get_local_peer_id()
)
```

### `despawn(entity: Node) → void`

Elimina la entidad localmente y notifica a todos los peers para que también la eliminen.

```gdscript
$PlayerSpawner.despawn(mi_jugador)
```

---

## Comportamiento automático

- **Despawn al desconectarse**: Cuando un peer se desconecta, el Spawner elimina automáticamente todas las entidades de ese peer.
- **Late join**: Cuando un jugador se une tarde, el host envía automáticamente todos los spawns existentes al nuevo jugador. Funciona en los backends LAN y Steam.

---

## Setup en el editor

```
Level (Node3D)
├── PlayerSpawner (LinkUxSpawner)
│   spawn_path: Players
│   spawnable_scenes: [player.tscn]
└── Players (Node3D)
```

---

## Ejemplo — Spawn de jugadores en el nivel

```gdscript
# level.gd
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
    pass  # El Spawner elimina la entidad automáticamente

func _spawn_player(peer_id: int) -> void:
    var pos := Vector3(randf_range(-4.0, 4.0), 1.0, randf_range(-4.0, 4.0))
    var p := LinkUx.get_player_info(peer_id)
    spawner.spawn(0, {
        "player_peer_id": peer_id,
        "player_nickname": p.display_name if p else "?",
        "global_position": pos,
    }, peer_id)

func _unhandled_key_input(event: InputEvent) -> void:
    if event.is_action_pressed("ui_cancel"):
        LinkUx.close_session()
```

---

<!-- doc-shell:page slug="rpc" -->

# Sistema RPC

LinkUx proporciona un sistema de **llamadas a procedimiento remoto** (RPC) completamente independiente del sistema RPC de Godot. Las RPCs de LinkUx se enrutan a través del `RpcRelay` y admiten tanto modo fiable como no fiable. Funciona de forma idéntica en los backends LAN y Steam.

## Métodos de envío

```gdscript
# Enviar a un peer específico
LinkUx.send_rpc(peer_id, "mi_metodo", [arg1, arg2], true)

# Enviar al host
LinkUx.send_rpc_to_host("mi_metodo", [arg1])

# Enviar a todos los peers (incluido el local)
LinkUx.broadcast_rpc("actualizar_puntuacion", [peer_id, puntuacion])
```

### Atajos de alto nivel

| Método | Descripción |
|--------|-------------|
| `send_to_all(method, payload, reliable)` | Broadcast a todos. |
| `send_to_host(method, payload, reliable)` | Al host (peer ID 1). |
| `send_to_player(peer_id, method, payload, reliable)` | A un jugador específico. |
| `send_to_clients(method, payload, reliable)` | A todos los clientes (excluye al host). |

---

## Registrar handlers

```gdscript
func _ready() -> void:
    LinkUx.register_rpc("recibir_mensaje", _on_recibir_mensaje)

func _on_recibir_mensaje(from_peer: int, mensaje: String) -> void:
    print("[%s]: %s" % [from_peer, mensaje])

func _exit_tree() -> void:
    LinkUx.unregister_rpc("recibir_mensaje")
```

> **Nota:** El primer argumento del handler siempre es `from_peer: int` (el ID del remitente), seguido de los argumentos enviados.

---

## Modo fiable vs no fiable

| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `reliable` | `true` | ENet/Steam garantiza la entrega y el orden. Úsalo para eventos críticos (spawn, muerte, puntaje). |
| `reliable` | `false` | Sin garantía de entrega. Menor latencia. Úsalo para actualizaciones de posición frecuentes. |

---

## Ejemplo — Sistema de chat

```gdscript
# chat.gd
extends Control

func _ready() -> void:
    LinkUx.register_rpc("chat_mensaje", _on_chat_mensaje)

func _exit_tree() -> void:
    LinkUx.unregister_rpc("chat_mensaje")

func _on_send_btn_pressed() -> void:
    var msg := $Input.text.strip_edges()
    if msg.is_empty():
        return
    var nombre := LinkUx.get_local_player_name()
    LinkUx.broadcast_rpc("chat_mensaje", [nombre, msg])
    $Input.text = ""

func _on_chat_mensaje(_from_peer: int, nombre: String, mensaje: String) -> void:
    $Log.text += "\n[%s]: %s" % [nombre, mensaje]
```

---

<!-- doc-shell:page slug="estado-global" -->

# Estado Global

El **estado global** es un diccionario clave-valor sincronizado con todos los peers. Solo el host puede modificarlo; los clientes lo reciben automáticamente y pueden consultarlo en cualquier momento.

## Métodos

### `LinkUx.set_global_state(key: String, value: Variant) → void`

```gdscript
# Solo desde el host
if LinkUx.is_host():
    LinkUx.set_global_state("fase_juego", "combate")
    LinkUx.set_global_state("tiempo_restante", 120)
    LinkUx.set_global_state("puntuaciones", {"Zara": 0, "Leo": 0})
```

### `LinkUx.get_global_state(key: String, default: Variant = null) → Variant`

```gdscript
var fase: String = LinkUx.get_global_state("fase_juego", "espera")
var tiempo: int   = LinkUx.get_global_state("tiempo_restante", 0)
```

---

## Señal `global_state_changed`

```gdscript
func _ready() -> void:
    LinkUx.global_state_changed.connect(_on_estado_cambiado)

func _on_estado_cambiado(key: String, value: Variant) -> void:
    match key:
        "fase_juego":
            _cambiar_fase(value)
        "tiempo_restante":
            $HUD/Timer.text = str(value)
```

---

## Casos de uso

- **Fases del juego**: `"lobby"`, `"contando"`, `"combate"`, `"fin"`
- **Contador regresivo** sincronizado
- **Tabla de puntuaciones** global
- **Configuración de la partida**: número de rondas, dificultad, mapa elegido
- **Semillas aleatorias** para generación procedural consistente

---

<!-- doc-shell:page slug="autoridad" -->

# Autoridad de Entidades

La **autoridad** determina qué peer tiene control sobre una entidad replicada. Solo el peer con autoridad puede enviar actualizaciones de estado para esa entidad; los demás reciben esas actualizaciones y las aplican.

## Métodos

### `LinkUx.set_entity_authority(entity: Node, peer_id: int) → void`

```gdscript
LinkUx.set_entity_authority($ObjetoCentral, 2)
```

### `LinkUx.is_entity_authority(entity: Node) → bool`

```gdscript
func _physics_process(_delta: float) -> void:
    if not LinkUx.is_entity_authority(self):
        return  # No procesar si no somos el dueño
    # ... lógica de física
```

### `LinkUx.transfer_authority(entity: Node, to_peer_id: int) → void`

```gdscript
# Host transfiere control de un vehículo al jugador que lo monta
LinkUx.transfer_authority($Vehiculo, peer_id_del_conductor)
```

---

## Modos de autoridad comparados

| Modo | Control | Caso de uso |
|------|---------|-------------|
| `HOST` | El host siempre tiene autoridad. | Objetos del mundo, enemigos controlados por servidor, puertas, trampas. |
| `OWNER` | El peer que spawneó la entidad. | Personajes de jugador, proyectiles del jugador. |
| `TRANSFERABLE` | Se puede pasar entre peers. | Vehículos, ítems recogibles, objetos de interacción. |

---

<!-- doc-shell:page slug="senales" -->

# Señales

LinkUx expone **26 señales** que cubren todo el ciclo de vida multiplayer. Conéctate a ellas para reaccionar a eventos de red sin necesidad de polling.

## Sesión

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `session_created` | `session_info: SessionInfo` | El host ha creado exitosamente la sesión. |
| `session_joined` | `session_info: SessionInfo` | El cliente se ha unido exitosamente a la sesión. |
| `session_started` | — | La sesión está lista (se emite en host Y cliente). Úsala para iniciar la carga de escena. |
| `session_closed` | — | La sesión se cerró (por `close_session()` o desconexión del host). |
| `session_ended` | — | Emitida internamente al finalizar la limpieza de sesión. |

## Jugadores

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `player_joined` | `player_info: PlayerInfo` | Un peer nuevo (incluido el propio) entró a la sesión. |
| `player_left` | `peer_id: int, reason: int` | Un peer se desconectó (antes de la limpieza). |
| `player_left_processed` | `player_info: PlayerInfo, reason: int` | Un peer se desconectó (después de limpiar su estado). Úsala para UI. |
| `player_updated` | `player_info: PlayerInfo` | Los datos de un jugador cambiaron. |

## Conexión

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `connection_failed` | `error: String` | Falló el intento de conectar/crear sesión. |
| `connection_state_changed` | `new_state: int` | El estado de conexión cambió. |
| `protocol_version_mismatch` | `local: int, remote: int` | El cliente tiene una versión de protocolo incompatible con el host. |
| `backend_incompatible` | `reason: String` | El backend activo no soporta las capacidades requeridas. |

## Carga de escena

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `scene_load_requested` | `scene_path: String` | El host solicita que todos carguen una escena. |
| `scene_all_ready` | `scene_path: String` | Todos los peers reportaron escena lista con `report_scene_ready()`. |

## Entidades y estado

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `authority_changed` | `entity: Node, new_authority: int` | La autoridad de una entidad cambió. |
| `global_state_changed` | `key: String, value: Variant` | Un valor del estado global cambió. |

## Red

| Señal | Parámetros | Cuándo se emite |
|-------|------------|-----------------|
| `network_tick` | `tick_number: int, delta: float` | Cada tick de red. |

---

## Ejemplo — Conectar señales esenciales

```gdscript
func _ready() -> void:
    LinkUx.session_started.connect(_on_session_started)
    LinkUx.session_closed.connect(_on_session_closed)
    LinkUx.connection_failed.connect(_on_connection_failed)
    LinkUx.player_joined.connect(_on_player_joined)
    LinkUx.player_left_processed.connect(_on_player_left)

func _on_session_started() -> void:
    print("Sesión iniciada. Host: ", LinkUx.is_host())

func _on_session_closed() -> void:
    get_tree().change_scene_to_file("res://scenes/menu.tscn")

func _on_connection_failed(error: String) -> void:
    $UI/ErrorLabel.text = "Error: " + error

func _on_player_joined(info: PlayerInfo) -> void:
    print("Jugador entró: ", info.display_name, " (", info.peer_id, ")")

func _on_player_left(info: PlayerInfo, _reason: int) -> void:
    print("Jugador salió: ", info.display_name)
```

---

<!-- doc-shell:page slug="api" -->

# Referencia de API

Referencia completa de todos los métodos públicos del Autoload `LinkUx`.

## Configuración y Steam

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `initialize(config)` | `int` | Inicializa LinkUx con la config dada. Retorna `ErrorCode`. |
| `initialize_steam(app_id)` | `bool` | Inicializa GodotSteam para el backend Steam. Escribe `steam_appid.txt` automáticamente. Retorna `true` si tuvo éxito. |
| `is_steam_initialized()` | `bool` | Si Steam fue inicializado con éxito. |
| `get_steam_user()` | `String` | Nombre Steam del jugador local, o `"Player"` si no está disponible. |
| `set_backend(backend_type)` | `void` | Activa el backend especificado. |
| `get_config()` | `LinkUxConfig` | Config activa (`null` si no inicializado). |
| `get_version()` | `String` | Versión del addon leída de `plugin.cfg` (ej. `"2.1.0"`). |
| `get_protocol_version()` | `int` | Versión del protocolo interno de red (entero). |
| `get_backend_type()` | `int` | Tipo de backend activo (`NetworkEnums.BackendType`). |
| `get_backend_name()` | `String` | Nombre legible del backend activo (`"LAN"` o `"Steam"`). |

## Sesión

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `prepare_for_new_session()` | `void` | Limpia estado para una nueva sesión. |
| `create_session(name, max, meta)` | `int` | Crea sesión como host. |
| `join_session(session_info)` | `int` | Une como cliente con SessionInfo. |
| `join_session_by_room_code(code)` | `int` | Une por código de sala (LAN=8 chars, Steam=6 chars). |
| `close_session()` | `void` | Cierra la sesión activa. |
| `get_current_session()` | `SessionInfo` | Sesión activa o `null`. |
| `get_room_code()` | `String` | Código de la sala activa. |
| `has_room()` | `bool` | ¿Hay sesión activa? |
| `is_in_session()` | `bool` | ¿En sesión? |
| `is_host()` | `bool` | ¿Soy host? |
| `is_client()` | `bool` | ¿Soy cliente? |
| `is_singleplayer()` | `bool` | ¿Solo un jugador? |
| `is_multiplayer()` | `bool` | ¿Más de un jugador? |
| `is_lan()` | `bool` | ¿Backend LAN activo? |
| `is_online()` | `bool` | ¿Backend Steam activo? |

## Jugadores

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `set_local_player_name(name)` | `void` | Nombre del jugador local. |
| `set_player_profile(name, meta, data)` | `void` | Perfil completo del jugador local. |
| `get_local_player_name()` | `String` | Nombre del jugador local. |
| `get_local_peer_id()` | `int` | ID de red del peer local. |
| `get_local_player()` | `PlayerInfo` | PlayerInfo del jugador local. |
| `get_player_info(peer_id)` | `PlayerInfo` | PlayerInfo de un peer. |
| `get_players()` | `Array[PlayerInfo]` | Todos los jugadores. |
| `get_remote_players()` | `Array[PlayerInfo]` | Solo jugadores remotos. |
| `get_host_player()` | `PlayerInfo` | PlayerInfo del host. |
| `get_client_players()` | `Array[PlayerInfo]` | Solo clientes. |
| `get_connected_peers()` | `Array[int]` | IDs de peers conectados. |
| `is_local_player_id(peer_id)` | `bool` | ¿Es el peer local? |
| `is_local_player_info(info)` | `bool` | ¿Es el PlayerInfo local? |
| `is_player_connected(peer_id)` | `bool` | ¿Está conectado? |
| `kick_player(peer_id, reason)` | `void` | Patear a un jugador (solo host). |
| `update_local_player_data(key, val)` | `void` | Actualiza datos locales. |
| `remove_local_player_data(key)` | `void` | Elimina dato local. |
| `set_player_data(peer, key, val)` | `bool` | Modifica datos de cualquier jugador. |
| `get_player_data(peer_id)` | `Dictionary` | Lee datos de un jugador. |
| `remove_player_data(peer, key)` | `bool` | Elimina dato de jugador. |

## Autoridad

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `set_entity_authority(entity, peer)` | `void` | Asigna autoridad. |
| `get_entity_authority(entity)` | `int` | Retorna el peer con autoridad. |
| `is_entity_authority(entity)` | `bool` | ¿Tengo autoridad sobre esta entidad? |
| `request_authority(entity)` | `void` | Solicitar autoridad (TRANSFERABLE). |
| `transfer_authority(entity, peer)` | `void` | Transferir autoridad a otro peer. |
| `validate_authority_change(entity, peer)` | `bool` | ¿Es válido transferir? |

## Replicación de estado

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `register_entity(node, props, mode, id)` | `void` | Registrar entidad en el replicador. |
| `unregister_entity(node)` | `void` | Desregistrar entidad. |
| `allocate_entity_network_id()` | `int` | Genera un ID de entidad de red único. |
| `set_global_state(key, value)` | `void` | Escribe en el estado global (solo host). |
| `get_global_state(key, default)` | `Variant` | Lee del estado global. |

## Carga de escena

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `request_scene_load(scene_path)` | `void` | Host solicita carga de escena a todos. |
| `report_scene_ready()` | `void` | El peer local notifica que cargó la escena. |
| `replay_late_join_spawns_now(peer_id)` | `void` | Fuerza el replay de spawns para un peer. |
| `run_late_join_snapshot_only(peer_id)` | `void` | Envía solo el snapshot de mundo (sin spawns). |

## RPC

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `send_rpc(peer, method, args, reliable)` | `void` | Envío a un peer. |
| `send_rpc_to_host(method, args, reliable)` | `void` | Envío al host. |
| `broadcast_rpc(method, args, reliable)` | `void` | Envío a todos. |
| `send_to_all(method, payload, reliable)` | `void` | Atajo de broadcast. |
| `send_to_host(method, payload, reliable)` | `void` | Atajo a host. |
| `send_to_player(peer, method, payload, reliable)` | `void` | Atajo a jugador. |
| `send_to_clients(method, payload, reliable)` | `void` | Atajo a todos los clientes. |
| `register_rpc(method, callable)` | `void` | Registrar handler. |
| `unregister_rpc(method)` | `void` | Desregistrar handler. |

## Debug

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `debug_mode(enabled)` | `void` | Activa/desactiva modo debug. |
| `enable_debug_overlay(enabled)` | `void` | Activa hooks de debug. |
| `get_debug_metrics()` | `Dictionary` | Métricas básicas: state, tick, peers, backend. |
| `dump_network_state()` | `Dictionary` | Volcado completo del estado de red. |
| `get_connection_state()` | `int` | Estado interno de la máquina de estados. |
| `get_feedback_logs(limit, min_level)` | `Array[Dictionary]` | Entradas del log filtradas. |
| `get_logs(limit)` | `Array[Dictionary]` | Todas las entradas del log. |
| `get_logs_type(level_name, limit)` | `Array[Dictionary]` | Log filtrado por nivel. |
| `set_feedback_log_capacity(max)` | `void` | Límite de entradas en el log. |
| `clear_feedback_logs()` | `void` | Vacía el log. |

---

<!-- doc-shell:page slug="debug" -->

# Debug y Diagnóstico

LinkUx incluye un completo sistema de diagnóstico que puedes usar tanto durante el desarrollo como para paneles in-game.

## Activar modo debug

```gdscript
# En la configuración
config.debug_enabled = true
config.log_level = 4  # DEBUG

# O en tiempo de ejecución
LinkUx.debug_mode(true)   # Activa debug y sube el log a DEBUG
LinkUx.debug_mode(false)  # Desactiva y restaura el log_level de config
```

---

## Métricas básicas

```gdscript
var metrics := LinkUx.get_debug_metrics()
# {
#   "state":   "RUNNING",    ← Estado de la máquina interna
#   "tick":    247,          ← Tick de red actual
#   "peers":   3,            ← Peers conectados
#   "backend": "Steam"       ← Backend activo ("LAN" o "Steam")
# }
```

---

## Volcado completo del estado

```gdscript
var estado := LinkUx.dump_network_state()
print(JSON.stringify(estado, "  "))
```

---

## Sistema de logs

```gdscript
# Últimas 50 entradas de cualquier nivel
var logs := LinkUx.get_logs(50)

# Solo errores (limit 0 = todos)
var errores := LinkUx.get_logs_type("ERROR")

# Entradas nivel INFO o superior
var info_logs := LinkUx.get_feedback_logs(0, DebugLogger.LogLevel.INFO)
```

Cada entrada del log es un `Dictionary`:

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

## Panel de debug in-game (ejemplo)

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

# Backend LAN

El backend LAN está basado en **ENetMultiplayerPeer** de Godot y permite jugar en red local con conexión directa por IP y puerto. No requiere conexión a internet ni cuentas externas.

## Cómo funciona

```
Host (ENet Server)
│
├── ENet ←→ Cliente 1
├── ENet ←→ Cliente 2
└── ENet ←→ Cliente 3

Topología estrella: los clientes no se conectan entre sí.
El host retransmite los paquetes de cada cliente al resto.
```

## Código de sala

Al crear una sesión LAN, LinkUx genera automáticamente un **código de sala hex de 8 caracteres** (p.ej. `"A3F7KQ2P"`). Internamente, el código codifica la IP y el puerto del servidor ENet. Los clientes resuelven el código y se conectan directamente — sin necesidad de conocer la IP.

```gdscript
func _on_session_started() -> void:
    var code := LinkUx.get_room_code()  # ej. "A3F7KQ2P"
    $CodeLabel.text = "Código de sala: " + code
```

## Port stride — Múltiples hosts en la misma máquina

Para pruebas en la misma PC, `LanBackendConfig.lan_port_stride` permite crear varios servidores. Si el puerto base (`7777`) ya está ocupado, LinkUx prueba `7779`, `7781`, etc.

```gdscript
config.lan = LanBackendConfig.new()
config.lan.default_port          = 7777
config.lan.lan_port_stride       = 2
config.lan.max_lan_host_attempts = 8
# Prueba: 7777, 7779, 7781, 7783, 7785, 7787, 7789, 7791
```

## Timeout de conexión

Si no se puede establecer la conexión ENet en `connection_timeout` segundos, LinkUx cierra el intento y emite `connection_failed`:

```gdscript
config.lan.connection_timeout = 3.0  # segundos
```

## Nota sobre `multiplayer_poll`

LinkUx usa paquetes binarios personalizados a través de ENet, distintos del sistema RPC de Godot. Al activar el backend LAN, LinkUx **desactiva automáticamente** `SceneTree.multiplayer_poll`. Esto es transparente — no necesitas hacerlo manualmente.

## Activar el backend LAN

```gdscript
LinkUx.set_backend(NetworkEnums.BackendType.LAN)
# — o en la config inicial —
config.default_backend = NetworkEnums.BackendType.LAN
```

---

<!-- doc-shell:page slug="backend-steam" -->

# Backend Steam

El backend Steam habilita el **multijugador online** a través de la plataforma Steam de Valve. Usa **Steam Lobbies** para el descubrimiento de sesiones y **SteamMultiplayerPeer** como capa de transporte. El relay, NAT traversal y el cifrado son gestionados automáticamente por Steam.

## Requisitos

| Elemento | Detalles |
|----------|---------|
| **GodotSteam GDExtension 4.4+** | Plugin oficial de [Gramps](https://godotsteam.com/). Debe estar instalado y habilitado en tu proyecto. |
| **Cliente Steam** | Debe estar en ejecución en la máquina del jugador antes de iniciar el juego. |
| **Steam App ID válido** | Usa `480` (Spacewar) para pruebas locales. Usa tu App ID real en producción. |
| **Cuentas Steam distintas** | Dos instancias con la **misma cuenta Steam** en la misma máquina no pueden conectarse por P2P — es una limitación de Steam. |

## Instalar GodotSteam

1. Descarga **GodotSteam GDExtension 4.4+** desde [godotsteam.com](https://godotsteam.com/).
2. Copia la carpeta del addon en tu proyecto bajo `res://addons/godotsteam/`.
3. Activa el plugin en **Project → Project Settings → Plugins**.
4. En `project.godot`, sección `[steam]`, configura `initialization/initialize_on_startup = false` — LinkUx gestiona la inicialización.

> GodotSteam registra la clase global `Steam` y el nodo `SteamMultiplayerPeer` automáticamente.

## Inicializar Steam

Llama a `LinkUx.initialize_steam(app_id)` **una vez al inicio**, antes de llamar a `LinkUx.initialize()`:

```gdscript
# GLOBAL.gd — Tu autoload de juego
extends Node

func _ready() -> void:
    # initialize_steam() es seguro llamarlo aunque Steam no esté en ejecución.
    # Retorna false y LinkUx continúa sin el backend Steam.
    LinkUx.initialize_steam(480)   # 480 = Spacewar para pruebas

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

`initialize_steam()` hace tres cosas automáticamente:
1. Crea `steam_appid.txt` en la ruta correcta (raíz del proyecto en editor, directorio del ejecutable en exports).
2. Llama a `steamInitEx(false, app_id)` de GodotSteam para iniciar el SDK de Steam.
3. Pone `is_steam_initialized()` en `true` si tiene éxito.

## Crear una sesión online

```gdscript
func _on_online_host_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        show_error("Steam no está en ejecución.")
        return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.create_session(LinkUx.get_steam_user() + "'s Room", 8, {})
```

Después de `session_started`, lee el **código de sala de 6 caracteres** y muéstralo:

```gdscript
func _on_session_started() -> void:
    var code := LinkUx.get_room_code()   # ej. "K7PQ3A"
    $CodeLabel.text = "Código: " + code
    if LinkUx.is_host():
        LinkUx.request_scene_load("res://scenes/level.tscn")
```

## Unirse a una sesión online

```gdscript
func _on_online_join_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        show_error("Steam no está en ejecución.")
        return
    var code := $CodeInput.text.strip_edges().to_upper()
    if code.length() != 6:
        show_error("Los códigos Steam tienen 6 caracteres.")
        return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    var err := LinkUx.join_session_by_room_code(code)
    if err != NetworkEnums.ErrorCode.SUCCESS:
        show_error("Formato de código inválido.")
```

## Códigos de sala — LAN vs Steam

| Backend | Formato | Longitud | Codifica |
|---------|---------|---------|---------|
| **LAN** | Hexadecimal | 8 chars | IP del host + puerto ENet |
| **Steam** | Alfanumérico (A–Z, 0–9) | 6 chars | Metadata del Steam Lobby |

6 chars × 36 símbolos = **~2.200 millones** de combinaciones únicas.

## Cómo funciona internamente

```
HOST
  1. set_backend(STEAM) + create_session()
  2. LinkUx llama a Steam.createLobby(PUBLIC, max_players)
  3. En lobby_created:
     → genera código de sala aleatorio de 6 chars
     → guarda el código como metadata del lobby (clave: "linkux_room_code")
     → crea SteamMultiplayerPeer.create_host(0)
     → emite session_started

CLIENTE
  1. set_backend(STEAM) + join_session_by_room_code("K7PQ3A")
  2. LinkUx llama a Steam.addRequestLobbyListStringFilter("linkux_room_code", "K7PQ3A")
     → Steam.requestLobbyList()
  3. En lobby_match_list: llama a Steam.joinLobby(lobbies[0])
  4. En lobby_joined:
     → obtiene Steam ID del host via Steam.getLobbyOwner()
     → crea SteamMultiplayerPeer.create_client(host_steam_id, 0)
     → emite session_started
```

## Verificar el backend activo

```gdscript
if LinkUx.is_online():
    print("Usando backend Steam")

print(LinkUx.get_backend_name())  # "Steam"
```

## Funciones de API relacionadas con Steam

| Método | Retorna | Descripción |
|--------|---------|-------------|
| `initialize_steam(app_id)` | `bool` | Inicializa GodotSteam. Retorna `true` si tuvo éxito. |
| `is_steam_initialized()` | `bool` | Si Steam fue inicializado correctamente. |
| `get_steam_user()` | `String` | Nombre Steam del jugador local, o `"Player"` si no disponible. |
| `is_online()` | `bool` | `true` si el backend activo es Steam. |
| `get_version()` | `String` | Versión actual del addon (ej. `"2.1.0"`). |

## Limitaciones

- **Misma máquina, misma cuenta**: Dos instancias del juego con la misma cuenta Steam no pueden conectarse entre sí por P2P — limitación de Steam. Usa cuentas separadas para pruebas en la misma máquina.
- **Cliente Steam requerido**: Si Steam no está en ejecución, `initialize_steam()` retorna `false` y el backend STEAM no está disponible.
- **Internet requerido para búsqueda de lobbies**: El descubrimiento de lobbies requiere conexión a internet. El tráfico de juego puede usar Steam Relay si P2P directo no está disponible.
- **`multiplayer_poll` desactivado**: Igual que el backend LAN, LinkUx desactiva `SceneTree.multiplayer_poll` al usar el backend Steam. Esto es transparente.

---

<!-- doc-shell:page slug="backends" -->

# Backends

LinkUx está diseñado desde cero para soportar múltiples backends. La clase base `NetworkBackend` define la interfaz que cualquier backend debe implementar, y el resto del sistema (API, subsistemas, nodos) es completamente agnóstico al transporte.

## Estado actual

| Backend | Estado | Transporte | Código de sala |
|---------|--------|-----------|----------------|
| **LAN (ENet)** | ✅ Disponible | ENetMultiplayerPeer | 8 chars hex |
| **Steam Online** | ✅ Disponible | SteamMultiplayerPeer | 6 chars alfanumérico |

## Cambiar de backend sin tocar el código de juego

Esta es la promesa central de LinkUx. Tus señales, RPCs, Spawner y Synchronizer funcionan de forma idéntica independientemente del backend activo. El único cambio al cambiar de backend es:

```gdscript
# Sesión LAN:
LinkUx.set_backend(NetworkEnums.BackendType.LAN)
LinkUx.create_session("Mi Sala", 4)

# Sesión Steam Online (misma API, backend diferente):
LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
LinkUx.create_session("Mi Sala", 8)
```

## Cómo se integra un backend

Cada backend es un script que hereda de `NetworkBackend` e implementa métodos como `_backend_create_session()`, `_backend_join_session_by_room_code()`, `_backend_kick_peer()`, etc. LinkUx detecta las capacidades del backend mediante `BackendCapabilityChecker`.

Para añadir soporte a un nuevo backend en el futuro:
1. Crea el script del backend bajo `addons/linkux/backends/tu_backend/`.
2. Añade el caso correspondiente en el enum `BackendType` en `network_enums.gd`.
3. Registra el backend en `set_backend()` de `linkux.gd`.

---

<!-- doc-shell:page slug="ejemplo-completo" -->

# Ejemplo Completo

Un juego multijugador mínimo pero completamente funcional usando LinkUx. Incluye menú con LAN y Steam Online, carga de escena sincronizada, spawn de jugadores y movimiento básico en primera persona.

## Estructura del proyecto

```
proyecto/
├── autoloads/
│   └── GLOBAL.gd          ← Autoload: inicializa LinkUx + Steam
├── scenes/
│   ├── menu.tscn           ← Menú principal
│   ├── level.tscn          ← Nivel multijugador
│   └── player.tscn         ← Prefab del jugador
└── scripts/
    ├── menu.gd
    ├── level.gd
    └── player.gd
```

---

## GLOBAL.gd — Autoload de juego

```gdscript
# autoloads/GLOBAL.gd
extends Node

func _ready() -> void:
    # Inicializar Steam (seguro aunque Steam no esté en ejecución)
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

## menu.gd — Menú principal

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
    LinkUx.create_session("Sala de " + nickname, DEFAULT_MAX_PLAYERS)

func _on_lan_join_pressed() -> void:
    var nickname: String = $NicknameInput.text.strip_edges()
    var code: String     = $CodeInput.text.strip_edges().to_upper()
    if nickname.is_empty() or code.is_empty(): return
    LinkUx.set_local_player_name(nickname)
    LinkUx.set_backend(NetworkEnums.BackendType.LAN)
    var err := LinkUx.join_session_by_room_code(code)
    if err != NetworkEnums.ErrorCode.SUCCESS:
        $StatusLabel.text = "Sala no encontrada (código inválido)"

# ─── Steam Online ────────────────────────────────────────────
func _on_online_host_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        $StatusLabel.text = "Steam no está en ejecución."; return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    LinkUx.create_session(LinkUx.get_steam_user() + "'s Room", DEFAULT_MAX_PLAYERS)

func _on_online_join_pressed() -> void:
    if not LinkUx.is_steam_initialized():
        $StatusLabel.text = "Steam no está en ejecución."; return
    var code: String = $CodeInput.text.strip_edges().to_upper()
    if code.is_empty(): return
    LinkUx.set_local_player_name(LinkUx.get_steam_user())
    LinkUx.set_backend(NetworkEnums.BackendType.STEAM)
    var err := LinkUx.join_session_by_room_code(code)
    if err != NetworkEnums.ErrorCode.SUCCESS:
        $StatusLabel.text = "Formato de código inválido."

# ─── Común ──────────────────────────────────────────────────
func _on_session_started() -> void:
    $CodeLabel.text = "Código: " + LinkUx.get_room_code()
    if LinkUx.is_host():
        LinkUx.request_scene_load("res://scenes/level.tscn")

func _on_connection_failed(error: String) -> void:
    $StatusLabel.text = "Error: " + error
```

---

## level.gd — Nivel de juego

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
    pass  # LinkUxSpawner limpia automáticamente

func _spawn_player(peer_id: int) -> void:
    var pos := Vector3(randf_range(-3.0, 3.0), 1.0, randf_range(-3.0, 3.0))
    var p := LinkUx.get_player_info(peer_id)
    spawner.spawn(0, {
        "player_peer_id":  peer_id,
        "player_nickname": p.display_name if p else "?",
        "global_position": pos,
    }, peer_id)

func _unhandled_key_input(event: InputEvent) -> void:
    if event.is_action_pressed("ui_cancel"):
        LinkUx.close_session()
```

---

## player.gd — Personaje jugador

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

## Árboles de escena

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

## Flujo completo

```
1. Jugador A → create_session() (LAN o Steam)  →  [session_started]
   └── request_scene_load("level.tscn")
   └── [scene_load_requested] → GLOBAL carga la escena

2. Jugador B → join_session_by_room_code(código)  →  [session_started]
   └── GLOBAL recibe [scene_load_requested] → carga la escena

3. Ambos llaman report_scene_ready()
   └── [scene_all_ready] → tick loop inicia

4. Cada level.gd spawna su jugador local via LinkUxSpawner
   └── El Spawner replica el spawn a todos los peers

5. LinkUxSynchronizer envía position/rotation cada tick
   └── Los peers remotos reciben y aplican con interpolación

6. Al salir → close_session() → [session_closed] → menú
```
