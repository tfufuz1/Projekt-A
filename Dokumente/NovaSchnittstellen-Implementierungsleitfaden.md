## Spezifikation und Implementierungsanleitung für die NovaDE-Systemschicht

**Ziel:** Erstellung einer robusten, performanten und modularen Systemschicht für die Nova Desktop Environment (NovaDE) in Rust. Diese Schicht interagiert mit dem Betriebssystem, der Hardware und externen Diensten und setzt die Richtlinien der Domänenschicht technisch um. Sie stellt die Grundlage für die Benutzeroberflächenschicht dar.

**Kernmodule und ihre Hauptverantwortlichkeiten:**

1. **`system::compositor`**: Smithay-basierter Wayland-Compositor; Fenster-Lebenszyklus, Protokoll-Implementierung (XDG-Shell, Layer-Shell etc.), Renderer-Abstraktion, XWayland.
2. **`system::input`**: `libinput`-basierte Eingabeverarbeitung; Seat-Management, `xkbcommon`-Integration, Event-Übersetzung für Tastatur, Zeiger, Touch, Gesten.
3. **`system::dbus_interfaces`**: `zbus`-basierte Clients für Systemdienste (NetworkManager, UPower, logind, Secrets, PolicyKit) und Server für `org.freedesktop.Notifications`.
4. **`system::audio_management`**: `pipewire-rs`-basierte PipeWire-Integration; Geräte- und Stream-Management, Lautstärkeregelung.
5. **`system::mcp_client`**: `mcp_client_rs`-basierter Client für das Model Context Protocol; sichere KI-Modell-Kommunikation.
6. **`system::window_mechanics`**: Technische Umsetzung der Fenster-Policies aus der Domänenschicht (Positionierung, Tiling, Fokus).
7. **`system::power_management`**: DPMS-Steuerung, Reaktion auf Inaktivität und logind-Ereignisse.
8. **`system::event_bridge`**: Zentraler Hub für systeminterne Events via `tokio::sync::broadcast`.

**Technologie-Stack (Auszug für Systemschicht):** Rust, Smithay, `libinput`, `xkbcommon`, `zbus`, `pipewire-rs`, `mcp_client_rs`, `calloop`, `tokio`.

**Entwicklungsrichtlinien:** Modulare Fehlerbehandlung (`thiserror`), strukturiertes Logging (`tracing`), `async/await` für nebenläufige Operationen.

**Folgend nun die detaillierten, feingranularen Schnittstellenspezifikationen pro Modul, die als direkte Implementierungsanweisung für eine KI-gestützte Entwicklung dienen.**

---

## Ultra-Feingranulare Schnittstellenspezifikation: NovaDE Systemschicht (`novade-system`)

Dieser Implementierungsleitfaden ist so konzipiert, dass er von einem autonomen KI-Entwicklungsagenten (z.B. Manus AI) direkt zur Codegenerierung verwendet werden kann. Jede Komponente, Datenstruktur, Methode und Interaktion ist explizit definiert, um Interpretationsspielraum zu minimieren.

### Modul 0: Vorbemerkungen und Globale Systemschicht-Konventionen

- **Crate-Name:** `novade-system`
- **Abhängigkeiten (exemplarisch, in `Cargo.toml` zu definieren):**
    - `novade-core = { path = "../novade-core" }`
    - `novade-domain = { path = "../novade-domain" }`
    - `smithay = "0.10.0"` (Version prüfen und Features nach Bedarf: `renderer_gl`, `backend_libinput`, `backend_session`, `backend_udev`, `backend_drm`, `desktop`, `xwayland`)
    - `wayland-server = "0.30"` (Smithay-kompatible Version)
    - `wayland-protocols = { version = "0.30", features = ["server", "unstable_protocols"] }`
    - `calloop = "0.12"`
    - `libinput = "0.9"`
    - `xkbcommon = "0.7"`
    - `zbus = { version = "3.15", default-features = false, features = ["tokio"] }` (Version prüfen)
    - `pipewire = "0.8"` (Version prüfen, ggf. `libspa`)
    - `mcp_client_rs = "0.2.0"` (Version prüfen)
    - `tokio = { version = "1.37", features = ["full"] }`
    - `async-trait = "0.1"`
    - `thiserror = "1.0"`
    - `tracing = "0.1"`
    - `uuid = { version = "1.8", features = ["v4"] }`
    - `serde = { version = "1.0", features = ["derive"] }`
    - `serde_json = "1.0"`
- **Fehlerbehandlung:** Jedes Submodul definiert ein eigenes `Error`-Enum mit `#[derive(Debug, thiserror::Error)]`. Fehler aus Bibliotheken werden mit `#[source]` oder `#[from]` gewrappt.
- **Logging:** `tracing::{trace, debug, info, warn, error}` Makros verwenden.
- **Asynchronität:** `async fn` für Operationen, die blockieren könnten. `calloop` für die Compositor-Hauptschleife, `tokio` für D-Bus, MCP und andere nebenläufige Tasks.

### Modul 1: `system::compositor`

Zweck: Implementierung des Wayland-Compositors.

#### 1.1. Submodul: `system::compositor::errors`

Datei: `src/compositor/errors.rs`

- **Enum `CompositorError`**:
    - Varianten (mit `#[error("...")]` und ggf. `#[source]` / `#[from]`):
        - `GlobalCreationFailed { name: String, details: String }`
        - `SurfaceRoleError(#[from] smithay::wayland::compositor::SurfaceRoleError)`
        - `ClientDataMissing { client_id_str: String }` (ClientId zu String für `Display`)
        - `SurfaceDataMissing { surface_id_str: String }` (WlSurface Debug zu String)
        - `InvalidSurfaceState { surface_id_str: String, reason: String }`
        - `RendererInitializationFailed(String)`
        - `DisplayOrLoopCreationFailed(String)`
        - `XWaylandInitializationFailed(String)`
        - `DrmBackendError { details: String, #[source] source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`
        - `LibinputBackendError(String)`
        - `SessionError(String)`
        - `XdgShellError(#[from] crate::compositor::xdg_shell::errors::XdgShellError)` (aus eigenem Submodul)
        - `LayerShellError(#[from] crate::compositor::layer_shell::errors::LayerShellError)` (aus eigenem Submodul)
        - `OutputManagementError(#[from] crate::compositor::output_management::errors::OutputManagementError)` (aus eigenem Submodul)

#### 1.2. Submodul: `system::compositor::types` (oder direkt in `core::state` und `surface_management`)

Datei: `src/compositor/types.rs` (oder aufgeteilt)

- **Struct `ClientCompositorData`** (für `Client::data_map`):
    - `compositor_state: smithay::wayland::compositor::CompositorClientState`
    - `xdg_shell_client_data: smithay::wayland::shell::xdg::XdgWmBaseClientData` (oder `XdgShellClientData` je nach Smithay Version)
- **Struct `SurfaceData`** (für `WlSurface::data_map`):
    - `id: uuid::Uuid`
    - `client_id_str: String`
    - `role: std::sync::Mutex<Option<String>>`
    - `current_buffer_info: std::sync::Mutex<Option<AttachedBufferInfo>>`
    - `texture_handle: std::sync::Mutex<Option<Box<dyn crate::compositor::renderer_interface::RenderableTexture>>>`
    - `damage_buffer_coords: std::sync::Mutex<Vec<smithay::utils::Rectangle<i32, smithay::utils::Buffer>>>`
    - `opaque_region_surface_local: std::sync::Mutex<Option<smithay::utils::Region<smithay::utils::Logical>>>`
    - `input_region_surface_local: std::sync::Mutex<Option<smithay::utils::Region<smithay::utils::Logical>>>`
    - `parent: std::sync::Mutex<Option<wayland_server::Weak<wayland_server::protocol::wl_surface::WlSurface>>>`
    - `children: std::sync::Mutex<Vec<wayland_server::Weak<wayland_server::protocol::wl_surface::WlSurface>>>`
    - `surface_viewporter_state: std::sync::Mutex<smithay::wayland::viewporter::SurfaceState>`
    - `surface_presentation_state: std::sync::Mutex<smithay::wayland::presentation::SurfaceState>`
    - `user_data_map: smithay::reexports::wayland_server::backend::UserDataMap` (für Shell-spezifische Daten)
    - **Methoden:** `new(client_id_str: String) -> Self`, `set_role(&self, role: &str) -> Result<(), CompositorError>`, `get_role(&self) -> Option<String>`.
- **Struct `AttachedBufferInfo`**:
    - `buffer: wayland_server::protocol::wl_buffer::WlBuffer`
    - `scale: i32`
    - `transform: smithay::utils::Transform`
    - `dimensions: smithay::utils::Size<i32, smithay::utils::Buffer>`

#### 1.3. Submodul: `system::compositor::core`

Datei: `src/compositor/core/state.rs`

- **Struct `DesktopState`** (zentraler Zustand, Details aus Gesamtspezifikation und System-Details):
    - **Felder (Auswahl):**
        - `display_handle: smithay::reexports::wayland_server::DisplayHandle`
        - `loop_handle: smithay::reexports::calloop::LoopHandle<'static, Self>`
        - `clock: smithay::utils::Clock<u64>`
        - `compositor_state: smithay::wayland::compositor::CompositorState`
        - `shm_state: smithay::wayland::shm::ShmState`
        - `presentation_state: smithay::wayland::presentation::PresentationState`
        - `viewporter_state: smithay::wayland::viewporter::ViewporterState`
        - `xdg_shell_state: smithay::wayland::shell::xdg::XdgShellState`
        - `xdg_activation_state: smithay::wayland::xdg_activation::XdgActivationState`
        - `layer_shell_state: smithay::wayland::shell::wlr_layer::WlrLayerShellState`
        - `decoration_state: smithay::wayland::shell::xdg::decoration::XdgDecorationState`
        - `space: std::sync::Arc<std::sync::Mutex<smithay::desktop::Space<ManagedWindow>>>`
        - `windows: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<crate::domain::workspaces::core::types::WindowIdentifier, Arc<ManagedWindow>>>>`
        - `seat_state: smithay::input::SeatState<Self>`
        - `seat: smithay::input::Seat<Self>`
        - `seat_name: String`
        - `input_method_manager_state: smithay::wayland::input_method::InputMethodManagerState`
        - `keyboard_data_map: std::sync::Arc<std::sync::Mutex<std::collections::HashMap<String, crate::input::keyboard::xkb_config::XkbKeyboardData>>>`
        - `current_cursor_status: std::sync::Arc<std::sync::Mutex<smithay::input::pointer::CursorImageStatus>>`
        - `pointer_location: std::sync::Arc<std::sync::Mutex<smithay::utils::Point<f64, smithay::utils::Logical>>>`
        - `output_manager_state: smithay::wayland::output::OutputManagerState`
        - `wlr_output_manager_v1_state: smithay::wayland::output_manager::OutputManagerState` (für wlr-output-management)
        - `wlr_output_power_manager_v1_state: smithay::wayland::output_power_manager::OutputPowerManagerState` (für wlr-output-power-management)
        - `data_device_state: smithay::wayland::selection::data_device::DataDeviceState`
        - `renderer: Option<Box<dyn crate::compositor::renderer_interface::FrameRenderer>>` (wird nach Backend-Init gesetzt)
        - `domain_services: Arc<crate::domain::DomainServices>` (Sammelstruktur für Domänen-Service-Handles)
        - `system_event_bridge: Arc<crate::event_bridge::SystemEventBridge>`
        - `active_backend_data: std::sync::Mutex<Option<ActiveBackendData>>` (Enum für DRM, Winit etc. Daten)
        - `wayland_globals: Option<NovaDEWaylandGlobals>` (Struct, das `GlobalId`s hält)
    - **Konstruktor:** `pub fn new(loop_handle, display_handle, domain_services, system_event_bridge) -> Self` (initialisiert alle Smithay-States).
    - **Methoden:** `pub fn create_all_wayland_globals(&mut self) -> Result<(), CompositorError>` (registriert alle Globals).
- **Implementierung von `smithay::wayland::compositor::CompositorHandler` für `DesktopState`**:
    - `compositor_state(&mut self) -> &mut CompositorState`: `&mut self.compositor_state`
    - `client_compositor_state<'a>(&self, client: &'a Client) -> &'a CompositorClientState`: `&client.get_data::<ClientCompositorData>().unwrap().compositor_state`
    - `commit(&mut self, surface: &WlSurface)`: Logik wie in spezifiziert (Puffer-Handling, Schaden, Rollen-Dispatch, Sync-Subsurfaces). Verwendet `with_surface_data_mut` für Zugriff auf `SurfaceData`.
    - `new_surface(&mut self, surface: &WlSurface)`: `surface.data_map().insert_if_missing_threadsafe(|| Arc::new(SurfaceData::new(...))); add_destruction_hook(...)`.
    - `new_subsurface(&mut self, surface: &WlSurface, parent: &WlSurface)`: Verlinkt Parent/Child in `SurfaceData`.
    - `destroyed(&mut self, surface: &WlSurface)`: Hauptsächlich für Logging und ggf. explizite Benachrichtigung anderer Teile von `DesktopState`, die `WlSurface` direkt referenzieren (obwohl `destruction_hook` bevorzugt wird).
- **Implementierung von `smithay::wayland::shm::ShmHandler` und `smithay::wayland::buffer::BufferHandler` für `DesktopState`**:
    - `shm_state(&self) -> &ShmState`: `&self.shm_state`
    - `buffer_destroyed(&mut self, buffer: &WlBuffer)`: Benachrichtigt Renderer zur Freigabe von Texturen, die mit diesem `buffer` assoziiert sind (Identifikation über `SurfaceData`).

#### 1.4. Submodul: `system::compositor::xdg_shell`

Datei: `src/compositor/xdg_shell/errors.rs`, `types.rs`, `handlers.rs`

- **Enum `XdgShellError`**: (Wie in spezifiziert)
- **Struct `ManagedWindow`** (Implementiert `smithay::desktop::Window`):
    - **Felder:** `id: uuid::Uuid`, `domain_id: crate::domain::workspaces::core::types::WindowIdentifier`, `xdg_surface: smithay::desktop::WindowSurface` (hält `ToplevelSurface` oder `PopupSurface`), `current_geometry: smithay::utils::Rectangle<i32, smithay::utils::Logical>`, `is_mapped: bool`, `requested_size`, `min_size`, `max_size`, `parent_window_domain_id: Option<DomainWindowIdentifier>`.
    - **Methoden:** `new_toplevel(...)`, `new_popup(...)`, `wl_surface()`, Methoden des `Window`-Traits (send_configure, set_activated, etc. – rufen Methoden auf `self.xdg_surface` auf).
- **Implementierung `smithay::wayland::shell::xdg::XdgShellHandler` für `DesktopState`**:
    - `xdg_shell_state(&mut self) -> &mut XdgShellState`: `&mut self.xdg_shell_state`
    - `new_toplevel(&mut self, surface: ToplevelSurface)`: Erstellt `ManagedWindow`, fordert initiale Geometrie von `domain::window_management_policy_service` an, sendet initiales `configure`, fügt zu `self.space` und `self.windows` hinzu.
    - `new_popup(&mut self, surface: PopupSurface, positioner: PositionerState)`: Erstellt `ManagedWindow` (als Popup), berechnet Geometrie basierend auf `positioner` und Parent, sendet `configure`, fügt zu `self.windows` hinzu.
    - `map_toplevel(&mut self, surface: &ToplevelSurface)`: Setzt `managed_window.is_mapped = true`, benachrichtigt Domäne (`workspace_manager_service.assign_window_to_active_workspace`), fordert Re-Layout.
    - `unmap_toplevel(&mut self, surface: &ToplevelSurface)`: Setzt `is_mapped = false`, benachrichtigt Domäne, entfernt aus `space`.
    - `ack_configure(&mut self, surface: WlSurface, configure_data: XdgSurfaceConfigureUserData)`: Verarbeitet Client-Bestätigung.
    - **Request-Handler** (`toplevel_request_set_title`, `toplevel_request_set_maximized`, `move_request`, `resize_request`, etc.):
        1. Findet `ManagedWindow` für das `ToplevelSurface`/`PopupSurface`.
        2. Aktualisiert den Zustand im `ManagedWindow` und/oder dessen Smithay `XdgToplevelSurfaceData`/`XdgPopupSurfaceData`.
        3. Interagiert mit `self.domain_services.window_policy_service` für Validierung oder Policy-Anpassungen.
        4. Sendet bei Bedarf neue `configure`-Events an den Client.
        5. Für `move` und `resize`: Startet einen interaktiven Grab über `self.seat.start_pointer_grab(...)` mit einer spezifischen Grab-Handler-Implementierung (siehe `system::window_mechanics`).
    - `toplevel_destroyed(&mut self, toplevel: ToplevelSurface)` / `popup_destroyed(&mut self, popup: PopupSurface)`: Entfernt `ManagedWindow` aus `self.space` und `self.windows`, benachrichtigt Domäne.

#### 1.5. Submodul: `system::compositor::layer_shell`

Datei: `src/compositor/layer_shell/errors.rs`, `types.rs`, `handlers.rs`

- **Enum `LayerShellError`**: `InvalidLayerSurfaceState`, `NamespaceAlreadyTaken`.
- **Struct `ManagedLayerSurface`** (Implementiert `smithay::desktop::LayerSurface`):
    - **Felder:** `id: uuid::Uuid`, `domain_id: crate::domain::workspaces::core::types::WindowIdentifier` (für Panel/Dock-Identifikation), `layer_surface: smithay::wayland::shell::wlr_layer::LayerSurface`, `current_geometry`, `is_mapped`.
- **Implementierung `smithay::wayland::shell::wlr_layer::LayerShellHandler` für `DesktopState`**:
    - `layer_shell_state(&mut self) -> &mut WlrLayerShellState`: `&mut self.layer_shell_state`
    - `new_layer_surface(&mut self, surface: LayerSurface, output: Option<Output>, layer: Layer, namespace: String)`: Erstellt `ManagedLayerSurface`, validiert Namespace, konfiguriert Layer und Anker, sendet initiales `configure`. Fügt zu `smithay::desktop::layer_map_for_output(output).map_layer(&layer_surface)`.
    - `layer_surface_commit(&mut self, surface: &LayerSurface)`: Reagiert auf Puffer-Commits, aktualisiert ggf. Geometrie basierend auf Client-Requests.
    - `layer_surface_destroyed(&mut self, surface: &LayerSurface)`: Entfernt aus Layer Map.

#### 1.6. Weitere `system::compositor` Submodule (Struktur analog)

- **`decoration`**: `XdgDecorationHandler` für `xdg-decoration-unstable-v1`. Verwaltet `XdgDecorationState`.
- **`output_management`**: Implementierung von `smithay::wayland::output::OutputHandler`, `smithay::wayland::output_manager::OutputManagerHandler` (für `wlr-output-management-unstable-v1`) und `smithay::wayland::xdg_output::XdgOutputHandler` (für `xdg-output-unstable-v1`). `DesktopState` hält `OutputManagerState`.
- **`input_method`**: `InputMethodHandler`, `TextInputHandler`. `DesktopState` hält `InputMethodManagerState`, `TextInputManagerState`.
- **`screencopy`**: Implementierung von `wlr-screencopy-unstable-v1` (benötigt Zugriff auf Renderer).
- **`data_device`**: `DataDeviceHandler`, `PrimarySelectionHandler`. `DesktopState` hält `DataDeviceState`, `PrimarySelectionState`.
- **`xwayland`**: `XWayland`, `X11Wm`, `XwmHandler`. Starten und Verwalten des XWayland-Servers.
- **`renderer_interface`**:
    - Trait `RenderableTexture: Send + Sync + std::fmt::Debug { fn id(&self) -> uuid::Uuid; fn bind(&self, slot: u32) -> Result<()>; fn width_px(&self) -> u32; ... }`
    - Trait `FrameRenderer: Send + Sync { fn new(...) -> Result<Self>; fn render_frame(...); fn present_frame(...); fn create_texture_from_shm(...); ... }`
    - Enum `RenderElement { Surface { texture, geometry, damage, ... }, Cursor { ... } }`
- **`drm_gbm_renderer` / `winit_renderer`**: Konkrete Implementierungen von `FrameRenderer`.

### Modul 2: `system::input`

Zweck: Eingabeverarbeitung.

#### 2.1. Submodul: `system::input::errors`

Datei: `src/input/errors.rs`

- **Enum `InputError`**: (Wie in spezifiziert) `SeatCreationFailed`, `CapabilityAdditionFailed`, `XkbConfigError`, `LibinputError`, `LibinputSessionError`, `SeatNotFound`, `KeyboardHandleNotFound`, etc.

#### 2.2. Submodul: `system::input::types` (oder in `keyboard`, `pointer` etc.)

Datei: `src/input/types.rs`

- **Struct `XkbConfig`** (aus `smithay::input::keyboard::KeyboardConfig`)
- **Struct `XkbKeyboardData`**:
    - `context: xkbcommon::xkb::Context`
    - `keymap: xkbcommon::xkb::Keymap`
    - `state: xkbcommon::xkb::State`
    - `repeat_timer: Option<calloop::TimerHandle>`
    - `repeat_info: Option<(u32 /* libinput keycode */, xkbcommon::xkb::Keycode /* xkb keycode */, SmithayModifiersState, std::time::Duration, std::time::Duration)>`
    - `focused_surface_on_seat: Option<wayland_server::Weak<WlSurface>>`
    - `repeat_key_serial: Option<smithay::utils::Serial>`
    - **Methoden:** `new(config: &XkbConfig) -> Result<Self, InputError>`, `update_xkb_state_from_modifiers(...)`.
- **Event `InputDeviceHotplugEvent`** (für `system::event_bridge`):
    - `device_name: String`
    - `device_type: String` (z.B. "keyboard", "pointer", "touch")
    - `event_type: HotplugType (Added | Removed)`

#### 2.3. Submodul: `system::input::seat_manager`

Datei: `src/input/seat_manager.rs`

- **`DesktopState` Implementierung von `smithay::input::SeatHandler`**:
    - `seat_state(&mut self) -> &mut SeatState<Self>`: `&mut self.seat_state`
    - `focus_changed(&mut self, seat: &Seat<Self>, focused_surface: Option<&WlSurface>)`: Aktualisiert internen Fokus-Cache (`self.active_input_surface`), benachrichtigt Domänenschicht (`domain::window_management_policy` oder `workspace_manager`) über Fokuswechsel. _Wichtig:_ `KeyboardHandle::set_focus` löst dies aus; hier nicht erneut `set_focus` rufen.
    - `cursor_image(&mut self, seat: &Seat<Self>, image: CursorImageStatus)`: Aktualisiert `self.current_cursor_status`, benachrichtigt Renderer zur Neudarstellung des Cursors. Bei `Named(name)`: Logik zum Laden von Cursorthemes (ggf. `libwayland-cursor` oder eigene Implementierung).
- **Funktion `pub fn create_seat(state: &mut DesktopState, display_handle: &DisplayHandle, seat_name: String) -> Result<(), InputError>`**:
    1. `seat = state.seat_state.new_wl_seat(display_handle, seat_name.clone(), ...);`
    2. `state.seats.insert(seat_name.clone(), seat.clone());` (Oder `state.seat = seat.clone()` für primären Seat).
    3. Initialisiere `XkbKeyboardData` für diesen Seat in `state.keyboard_data_map`.
    4. Fähigkeiten (Keyboard, Pointer, Touch) werden später hinzugefügt, wenn `LibinputInputBackend` Geräte meldet.

#### 2.4. Submodul: `system::input::libinput_handler`

Datei: `src/input/libinput_handler/mod.rs` (und ggf. `session_interface.rs`)

- **Struct `LibinputSessionInterface`** (implementiert `smithay::backend::input::LibinputInterface`): Kapselt `smithay::backend::session::Session` (z.B. `LogindSession` oder `DirectSession`). Methoden `open_restricted`, `close_restricted`.
- **Funktion `pub fn init_libinput_backend<S: Session + 'static>(session_interface: Rc<RefCell<S>>) -> Result<LibinputInputBackend, InputError>`**: Erstellt `LibinputInputBackend`.
- **Funktion `pub fn register_libinput_event_source(loop_handle: &LoopHandle<DesktopState>, backend: LibinputInputBackend, seat_name_for_callback: String) -> Result<calloop::Source<LibinputInputBackend>, InputError>`**: Fügt Backend zur `calloop`-Schleife hinzu. Callback ruft `process_input_event`.
- **Funktion `pub fn process_input_event(desktop_state: &mut DesktopState, event: InputEvent<LibinputInputBackend>, seat_name: &str)`**:
    1. Holt `Seat<DesktopState>` für `seat_name`.
    2. `match event`:
        - `DeviceAdded { device }`: `tracing::info!(...)`. Prüft `device.capabilities()`. Wenn z.B. `Keyboard` und Seat noch keine Tastatur hat: `seat.add_keyboard(XkbConfig::default(), 200, 25)?`. Analog für Pointer, Touch. Sendet `InputDeviceHotplugEvent`.
        - `DeviceRemoved { device }`: `tracing::info!(...)`. Entfernt Capabilities vom Seat. Sendet `InputDeviceHotplugEvent`.
        - `Keyboard { event }`: Ruft `crate::input::keyboard::key_event_translator::handle_keyboard_event(...)`.
        - `PointerMotion { event }`: Ruft `crate::input::pointer::pointer_event_translator::handle_pointer_motion_event(...)`.
        - (Analog für `PointerMotionAbsolute`, `PointerButton`, `PointerAxis`, `TouchDown`, `TouchUp`, `TouchMotion`, `TouchFrame`, `TouchCancel`, `Gesture*`).

#### 2.5. Submodule: `system::input::keyboard`, `system::input::pointer`, `system::input::touch`, `system::input::gestures`

- **`keyboard::key_event_translator::handle_keyboard_event(...)`**:
    1. Holt `KeyboardHandle<DesktopState>` und `XkbKeyboardData`.
    2. Konvertiert `libinput keycode` zu `xkbcommon keycode` (`+8`).
    3. `xkb_data.state.update_key(...)`.
    4. Holt `SmithayModifiersState` von `xkb_data.state`.
    5. `keyboard_handle.modifiers(serial, mods_state, ...)`
    6. Wenn `KeyState::Pressed`: `keyboard_handle.key(serial, time, xkb_keycode, KeyState::Pressed, ...)`. Tastenwiederholung einrichten/aktualisieren (via `calloop::TimerHandle` in `XkbKeyboardData`, Callback sendet erneut `key` Event mit neuem Serial).
    7. Wenn `KeyState::Released`: `keyboard_handle.key(...)`. Tastenwiederholung abbrechen.
- **`keyboard::focus::set_keyboard_focus(...)`**:
    1. Holt `KeyboardHandle<DesktopState>` und `XkbKeyboardData`.
    2. `keyboard_handle.set_focus(target_surface_option, serial, ...)` (Smithay sendet `enter`/`leave`).
    3. Aktualisiert `xkb_data.focused_surface_on_seat`.
    4. Benachrichtigt Domäne (via `desktop_state.domain_services...` oder `SystemEventBridge`).
- **`pointer::pointer_event_translator::handle_pointer_motion_event(...)`**:
    1. Holt `PointerHandle<DesktopState>`.
    2. Aktualisiert `desktop_state.pointer_location` basierend auf `event.delta()`.
    3. `let (new_focus_surface, surface_local_coords) = find_surface_and_coords_at_global_point(desktop_state, desktop_state.pointer_location)`.
    4. `pointer_handle.motion(time, new_focus_surface.as_ref(), serial, desktop_state.pointer_location, surface_local_coords, ...)` (Smithay sendet `enter`/`leave`/`motion`).
    5. Aktualisiert `desktop_state.active_input_surface`.
- **`find_surface_and_coords_at_global_point(...)`**: Iteriert `desktop_state.space.elements_under(global_pos)`, prüft `input_region` von `SurfaceData`.
- (Analog für andere Zeiger- und Touch-Events, die entsprechende `PointerHandle`- und `TouchHandle`-Methoden aufrufen).

---

(Die Spezifikationen für die Module `system::dbus_interfaces`, `system::audio_management`, `system::mcp_client`, `system::window_mechanics`, `system::power_management` und `system::event_bridge` würden mit derselben Detailtiefe folgen, basierend auf den vorherigen Antworten und den Quelldokumenten.)

**Dieser ultra-feingranulare Implementierungsleitfaden für die Systemschicht, beginnend mit dem Compositor und der Eingabeverarbeitung, legt eine präzise Basis für die Entwicklung. Die konsequente Anwendung der definierten Strukturen, Methoden, Fehler und Events durch einen KI-Agenten soll zu einer robusten und korrekten Implementierung führen.**

Ultra-Feinspezifikation und Implementierungsleitfaden der internen Schnittstellen der Linux Desktop-Umgebung "NovaDE"EinleitungDieses Dokument dient als erweiterte technische Spezifikation und detaillierter Implementierungsleitfaden für die internen Schnittstellen zwischen den vier Hauptarchitekturschichten der Linux Desktop-Umgebung "NovaDE": Kernschicht (Core Layer), Domänenschicht (Domain Layer), Systemschicht (System Layer) und Benutzeroberflächenschicht (UI Layer). Die Kommunikation zwischen diesen Schichten erfolgt primär über wohldefinierte öffentliche APIs, die häufig als Rust-Traits implementiert sind, sowie über ein robustes Event-System zur Signalisierung von Zustandsänderungen. Ziel dieses Dokuments ist es, eine lückenlose Ultra-Feinspezifikation bereitzustellen, die Entwicklern eine direkte Implementierung ohne eigene Entwurfsentscheidungen ermöglicht. Alle relevanten Aspekte wurden recherchiert, entschieden und präzise spezifiziert, um Klarheit und Konsistenz über das gesamte System hinweg zu gewährleisten.Die hier dargelegten Spezifikationen sind entscheidend, um die kohärente Zusammenarbeit der verschiedenen Schichten sicherzustellen und die übergeordneten Projektziele – Performance, Intuition, Modernität, Modularität, Anpassbarkeit, sichere KI-Integration und Stabilität – zu erreichen.1. Schnittstelle: Kernschicht (Core Layer) zu höheren Schichten (Domäne, System, UI)Die Kernschicht stellt fundamentale Bausteine und Dienste bereit, die von allen darüberliegenden Schichten genutzt werden. Diese Dienste sind so konzipiert, dass sie maximale Wiederverwendbarkeit und minimale Abhängigkeiten für die höheren Schichten gewährleisten.1.1. Bereitgestellte Funktionalität1.1.1. core::typesDefiniert grundlegende, universell einsetzbare Datentypen.
Schnittstelle: Direkte Verwendung von Typen wie Point<T>, Size<T>, Rect<T>, RectInt, Color, Orientation sowie uuid::Uuid und chrono::DateTime<Utc> durch die höheren Schichten.
Beispielhafte Nutzung: Die Domänenschicht verwendet Color für Theming-Definitionen, die Systemschicht RectInt für Fenstergeometrien, und die UI-Schicht Point<T> für die Positionierung von Elementen.
1.1.2. core::errorsStellt eine Basis-Fehlerbehandlungsstrategie und den CoreError-Typ bereit.
Schnittstelle: Höhere Schichten können CoreError oder spezifischere Fehler aus Kernmodulen mittels #[from] oder #[source] in ihre eigenen Fehlertypen wrappen. Die Fehlerursachenkette (source()) wird dabei beibehalten.1
Beispielhafte Nutzung: Ein ConfigError in domain::workspaces::config kann einen CoreError::Io wrappen, der beim Lesen einer Datei in core::config aufgetreten ist.
1.1.3. core::loggingDefiniert die Logging-Infrastruktur basierend auf tracing.
Schnittstelle: Alle höheren Schichten verwenden die tracing-Makros (trace!, info!, etc.) für ihre Logging-Ausgaben. Die Funktion core::logging::initialize_logging() wird typischerweise einmalig von der Anwendung (z.B. UI-Schicht) beim Start aufgerufen.
1.1.4. core::configStellt Primitive zum Laden, Parsen und Zugreifen auf Kernkonfigurationen bereit.
Schnittstelle: Funktionen wie load_core_config(custom_path: Option<PathBuf>) -> Result<CoreConfig, ConfigError> und get_core_config() -> &'static CoreConfig für den globalen Zugriff. Die CoreConfig-Struktur selbst ist Teil der Schnittstelle.
Beispielhafte Nutzung: domain::settings_persistence_iface (oder eine konkrete Implementierung wie FilesystemConfigProvider) nutzt diese API, um Basiskonfigurationen zu lesen, die dann von der Domänenschicht weiterverarbeitet werden.
1.1.5. core::utilsBietet allgemeine Hilfsfunktionen.
Schnittstelle: Direkte Nutzung der öffentlichen Funktionen durch alle höheren Schichten.
1.2. Ultra-Feinspezifikation: Modul core::config::loaderDieses interne Modul ist für das Laden und Parsen von Konfigurationsdateien für das öffentliche core::config API zuständig.1.2.1. Modulübersicht
Zweck: Implementiert die Logik zum Auffinden, Lesen und Deserialisieren der Kernkonfigurationsdatei (core.json oder ein anderes gewähltes Format).
Interne Abhängigkeiten: core::types (für PathBuf), core::errors::CoreError, super::types::CoreConfig (die öffentliche Konfigurationsstruktur), super::errors::ConfigError.
Externe Abhängigkeiten: serde, serde_json (oder toml, etc.), std::fs, std::path, std::io.
1.2.2. Interne Datenstrukturen
struct RawCoreConfig

Datei: nova_de/core/src/config/raw_types.rs
Zweck: Dient der direkten Deserialisierung der Konfigurationsdatei. Dies ermöglicht Flexibilität bei der Versionierung und Validierung der Konfiguration, bevor sie in die öffentliche CoreConfig-Struktur überführt wird. Felder sind typischerweise Option<T>, um fehlende Werte abzufangen und Defaults anzuwenden.
Attribute: Spiegeln die Felder von CoreConfig, jedoch als Option<String>, Option<bool>, etc. Beispiel:
Rust// nova_de/core/src/config/raw_types.rs
use serde::Deserialize;

#
pub(super) struct RawCoreConfig {
    pub(super) log_level: Option<String>,
    pub(super) default_theme_name: Option<String>,
    pub(super) enable_experimental_features: Option<bool>,
    // Weitere Felder entsprechend CoreConfig
}


Sichtbarkeit: pub(super) innerhalb des core::config Moduls.


1.2.3. Fehlerbehandlung: ConfigError
Datei: nova_de/core/src/config/errors.rs (oder in core::errors integriert)
Definition:
Rust// nova_de/core/src/config/errors.rs
use std::io;
use std::path::PathBuf;
use thiserror::Error;

#
pub enum ConfigError {
    #
    NotFound { searched_paths: Vec<PathBuf> },

    #[error("I/O error accessing config file at {path:?}")]
    Io {
        path: PathBuf,
        #[source]
        source: io::Error,
    },

    #[error("Failed to parse config file at {path:?}")]
    ParseError {
        path: PathBuf,
        #[source]
        source: serde_json::Error, // Oder anderer Deserialisierungsfehler
        content_snippet: String, // Ein kurzer Ausschnitt des fehlerhaften Inhalts
    },

    #[error("Invalid configuration value for key '{key}': {message}")]
    InvalidValue { key: String, message: String },

    #[error("Could not determine home directory")]
    HomeDirectoryNotFound,
}

Die Verwendung von thiserror ermöglicht eine klare Fehlerdefinition und -verkettung.1
1.2.4. Interne Funktionen

fn resolve_config_path(custom_path: Option<&PathBuf>) -> Result<PathBuf, ConfigError>

Datei: nova_de/core/src/config/loader.rs
Zweck: Bestimmt den zu ladenden Konfigurationspfad gemäß der XDG Base Directory Specification und optionalen benutzerdefinierten Pfaden.
Parameter:

custom_path: Option<&PathBuf> (const): Ein optional vom Benutzer bereitgestellter Pfad.


Rückgabe: Result<PathBuf, ConfigError> - Der aufgelöste Pfad oder ein ConfigError::NotFound.
Implementierungsschritte:

Wenn custom_path Some(path) ist:

Prüfen, ob path existiert. Wenn ja, Ok(path.clone()) zurückgeben.
Wenn nicht, Err(ConfigError::NotFound { searched_paths: vec![path.clone()] }) zurückgeben (oder spezifischerer Fehler).


Andernfalls XDG-Pfade prüfen:

xdg_config_home = std::env::var("XDG_CONFIG_HOME").map(PathBuf::from).or_else(|_| dirs::home_dir().map(|h| h.join(".config"))).ok_or(ConfigError::HomeDirectoryNotFound)?;
user_path = xdg_config_home.join("novade/core.json");
Wenn user_path existiert, Ok(user_path) zurückgeben.


Systemweite XDG-Pfade prüfen:

xdg_config_dirs = std::env::var("XDG_CONFIG_DIRS").map(|dirs| dirs.split(':').map(PathBuf::from).collect::<Vec<_>>()).unwrap_or_else(|_| vec!);
Für jeden dir in xdg_config_dirs:

system_path = dir.join("novade/core.json");
Wenn system_path existiert, Ok(system_path) zurückgeben.




Fallback-Systempfad prüfen:

fallback_path = PathBuf::from("/etc/novade/core.json");
Wenn fallback_path existiert, Ok(fallback_path) zurückgeben.


Wenn keine Datei gefunden wurde, Err(ConfigError::NotFound { searched_paths:... }) mit allen geprüften Pfaden zurückgeben.


Vorbedingungen: Keine.
Nachbedingungen: Gibt einen validen, existierenden Pfad zurück oder einen Fehler.



fn read_config_file(path: &Path) -> Result<String, ConfigError>

Datei: nova_de/core/src/config/loader.rs
Zweck: Liest den Inhalt der Konfigurationsdatei.
Parameter:

path: &Path (const): Pfad zur Konfigurationsdatei.


Rückgabe: Result<String, ConfigError> - Dateiinhalt als String oder ein ConfigError::Io.
Implementierungsschritte:

std::fs::read_to_string(path).map_err(|e| ConfigError::Io { path: path.to_path_buf(), source: e })


Vorbedingungen: path ist ein valider Pfad.
Nachbedingungen: Gibt den Dateiinhalt zurück oder einen I/O-Fehler.



fn parse_raw_config(content: &str, path: &Path) -> Result<RawCoreConfig, ConfigError>

Datei: nova_de/core/src/config/loader.rs
Zweck: Parst den String-Inhalt in die RawCoreConfig-Struktur.
Parameter:

content: &str (const): Der zu parsende String-Inhalt.
path: &Path (const): Der Pfad der Konfigurationsdatei (für Fehlerberichte).


Rückgabe: Result<RawCoreConfig, ConfigError> - Die geparste Rohkonfiguration oder ein ConfigError::ParseError.
Implementierungsschritte:

serde_json::from_str(content).map_err(|e| ConfigError::ParseError { path: path.to_path_buf(), source: e, content_snippet: content.chars().take(100).collect() })


Vorbedingungen: content ist ein valider String.
Nachbedingungen: Gibt die geparste Rohkonfiguration zurück oder einen Parse-Fehler.



fn convert_raw_to_core_config(raw_config: RawCoreConfig, config_path: &Path) -> Result<CoreConfig, ConfigError>

Datei: nova_de/core/src/config/loader.rs
Zweck: Konvertiert RawCoreConfig in CoreConfig, wendet Standardwerte an und validiert Werte.
Parameter:

raw_config: RawCoreConfig: Die geparste Rohkonfiguration.
config_path: &Path: Pfad zur Konfigurationsdatei für Fehlerberichte und Kontext.


Rückgabe: Result<CoreConfig, ConfigError> - Die validierte CoreConfig oder ein ConfigError::InvalidValue.
Implementierungsschrite:

Erstelle eine CoreConfig-Instanz.
Für jedes Feld in RawCoreConfig:

Wenn Some(value), validiere value. Bei Ungültigkeit, Err(ConfigError::InvalidValue { key: "...", message: "..." }) zurückgeben.
Wenn None, verwende einen hartcodierten Standardwert für CoreConfig.
Beispiel: log_level = raw_config.log_level.unwrap_or_else(|| "info".to_string());
Validierung für log_level: Muss einer der unterstützten Werte sein (z.B. "trace", "debug", "info", "warn", "error").


Ok(core_config) zurückgeben.


Vorbedingungen: raw_config ist vorhanden.
Nachbedingungen: Gibt eine valide CoreConfig zurück oder einen Fehler bei ungültigen Werten.


1.2.5. Öffentliche Funktion (innerhalb core::config)
pub(super) fn load_and_parse_core_config(custom_path: Option<PathBuf>) -> Result<CoreConfig, ConfigError>

Datei: nova_de/core/src/config/mod.rs (ruft Funktionen aus loader.rs auf)
Zweck: Orchestriert den gesamten Lade- und Parse-Vorgang. Dies ist die Hauptfunktion, die von der öffentlichen API load_core_config genutzt wird.
Parameter:

custom_path: Option<PathBuf>: Optionaler benutzerdefinierter Pfad.


Rückgabe: Result<CoreConfig, ConfigError>.
Implementierungsschritte:

let resolved_path = loader::resolve_config_path(custom_path.as_ref())?;
let content = loader::read_config_file(&resolved_path)?;
let raw_config = loader::parse_raw_config(&content, &resolved_path)?;
loader::convert_raw_to_core_config(raw_config, &resolved_path)


Vorbedingungen: Keine.
Nachbedingungen: Gibt eine geladene und validierte CoreConfig zurück oder einen entsprechenden ConfigError.


Die Trennung von RawCoreConfig und CoreConfig ist eine bewährte Methode, um die API stabil zu halten, während interne Repräsentationen oder Konfigurationsdateiformate versioniert oder migriert werden können. Dies erhöht die Robustheit und Wartbarkeit des Konfigurationssystems.1.3. Ultra-Feinspezifikation: Modul core::types_moduleDieses Modul ist das Fundament für viele Datenaustauschoperationen innerhalb von NovaDE.1.3.1. Modulübersicht
Zweck: Definition und Implementierung fundamentaler, wiederverwendbarer Datentypen wie geometrische Primitiven (Point, Size, Rect), Color und Orientation. Stellt auch uuid::Uuid und chrono::DateTime<Utc> für das gesamte System bereit.
Dateistruktur:

nova_de/core/src/types/mod.rs: Hauptmoduldatei, exportiert alle Typen.
nova_de/core/src/types/point.rs: Definition von Point<T>.
nova_de/core/src/types/size.rs: Definition von Size<T>.
nova_de/core/src/types/rect.rs: Definition von Rect<T> und RectInt.
nova_de/core/src/types/color.rs: Definition von Color und ColorParseError.
nova_de/core/src/types/orientation.rs: Definition von Orientation.


Abhängigkeiten: uuid (mit Features "serde", "v4"), chrono (mit Feature "serde").5
1.3.2. Datentyp-SpezifikationenPoint<T>
Datei: nova_de/core/src/types/point.rs
Definition:
Rust#
#
pub struct Point<T> {
    pub x: T,
    pub y: T,
}


Attribute:

x: T - Sichtbarkeit: pub. X-Koordinate.
y: T - Sichtbarkeit: pub. Y-Koordinate.


Methoden:

pub const fn new(x: T, y: T) -> Self: Erzeugt einen neuen Punkt.
pub fn map<U, F>(&self, f: F) -> Point<U> where F: Fn(T) -> U, T: Copy: Mappt die Koordinaten zu einem neuen Typ.
Für T: std::ops::Add<Output = T> + Copy:

pub fn offset(&self, dx: T, dy: T) -> Self: Verschiebt den Punkt um (dx, dy).


Für T: std::ops::Sub<Output = T> + Copy:

pub fn vector_to(&self, other: &Point<T>) -> Point<T>: Erzeugt einen Vektor (als Punkt) von self zu other.




Invarianten: Keine über die Typ-Constraints von T hinaus.
Size<T>
Datei: nova_de/core/src/types/size.rs
Definition:
Rust#
#
pub struct Size<T> {
    pub w: T, // width
    pub h: T, // height
}


Attribute:

w: T - Sichtbarkeit: pub. Breite.
h: T - Sichtbarkeit: pub. Höhe.


Methoden:

pub const fn new(w: T, h: T) -> Self: Erzeugt eine neue Größe.
pub fn map<U, F>(&self, f: F) -> Size<U> where F: Fn(T) -> U, T: Copy: Mappt Breite und Höhe zu einem neuen Typ.
Für T: std::ops::Mul<Output = T> + Copy:

pub fn area(&self) -> T: Berechnet die Fläche.


Für T: PartialOrd + Default:

pub fn is_empty(&self) -> bool where T: Default + PartialEq: Prüft, ob Breite oder Höhe null sind (abhängig von T::default()).




Invarianten: Wenn T ein numerischer Typ ist, sollten w und h typischerweise nicht-negativ sein. Dies wird jedoch nicht aktiv erzwungen, außer durch Nutzungskonventionen oder spezifische Konstruktoren in höheren Schichten.
Rect<T>
Datei: nova_de/core/src/types/rect.rs
Definition:
Rustuse super::{Point, Size}; // Import Point und Size aus dem gleichen Modul

#
#
pub struct Rect<T> {
    pub origin: Point<T>,
    pub size: Size<T>,
}

pub type RectInt = Rect<i32>;


Attribute:

origin: Point<T> - Sichtbarkeit: pub. Ursprungspunkt (typischerweise linke obere Ecke).
size: Size<T> - Sichtbarkeit: pub. Ausmaße des Rechtecks.


Methoden (Beispiele für T: Copy + PartialOrd + std::ops::Add<Output = T> + std::ops::Sub<Output = T>):

pub const fn new(x: T, y: T, w: T, h: T) -> Self: Erzeugt ein neues Rechteck.
pub fn from_points(p1: Point<T>, p2: Point<T>) -> Self: Erzeugt ein Rechteck aus zwei gegenüberliegenden Punkten (normalisiert).
pub fn x(&self) -> T: Gibt self.origin.x zurück.
pub fn y(&self) -> T: Gibt self.origin.y zurück.
pub fn width(&self) -> T: Gibt self.size.w zurück.
pub fn height(&self) -> T: Gibt self.size.h zurück.
pub fn right(&self) -> T: Gibt self.origin.x + self.size.w zurück.
pub fn bottom(&self) -> T: Gibt self.origin.y + self.size.h zurück.
pub fn contains_point(&self, point: &Point<T>) -> bool: Prüft, ob ein Punkt innerhalb des Rechtecks liegt.
pub fn intersects(&self, other: &Rect<T>) -> bool: Prüft, ob sich dieses Rechteck mit einem anderen überschneidet.
pub fn intersection(&self, other: &Rect<T>) -> Option<Rect<T>>: Berechnet die Schnittmenge mit einem anderen Rechteck.
pub fn union(&self, other: &Rect<T>) -> Rect<T>: Berechnet die Vereinigungsmenge mit einem anderen Rechteck.
pub fn translated(&self, dx: T, dy: T) -> Self: Erzeugt ein verschobenes Rechteck.
pub fn inflated(&self, dw: T, dh: T) -> Self: Erzeugt ein Rechteck, dessen Größe um dw und dh geändert wurde (Zentrum bleibt gleich).


Invarianten: Analog zu Size<T>, sollten size.w und size.h typischerweise nicht-negativ sein.
Color
Datei: nova_de/core/src/types/color.rs
Definition:
Rustuse thiserror::Error;

#
#
pub struct Color {
    pub r: u8,
    pub g: u8,
    pub b: u8,
    pub a: u8,
}

#
pub enum ColorParseError {
    #
    InvalidFormat(String),
    #[error("Invalid hex digit in string '{input}' at component '{component}'.")]
    InvalidHexDigit { input: String, component: String },
    #[error("Hex string '{0}' has incorrect length.")]
    InvalidLength(String),
}

impl Color {
    pub const fn new(r: u8, g: u8, b: u8, a: u8) -> Self {
        Color { r, g, b, a }
    }

    pub fn from_hex(hex_string: &str) -> Result<Self, ColorParseError> {
        let s = hex_string.trim_start_matches('#');
        if!(s.len() == 6 |


| s.len() == 8) {return Err(ColorParseError::InvalidLength(hex_string.to_string()));}if!s.chars().all(|c| c.is_ascii_hexdigit()) {return Err(ColorParseError::InvalidFormat(hex_string.to_string()));}        let r = u8::from_str_radix(&s[0..2], 16).map_err(|_| ColorParseError::InvalidHexDigit{ input: hex_string.to_string(), component: "R".to_string()})?;
        let g = u8::from_str_radix(&s[2..4], 16).map_err(|_| ColorParseError::InvalidHexDigit{ input: hex_string.to_string(), component: "G".to_string()})?;
        let b = u8::from_str_radix(&s[4..6], 16).map_err(|_| ColorParseError::InvalidHexDigit{ input: hex_string.to_string(), component: "B".to_string()})?;
        let a = if s.len() == 8 {
            u8::from_str_radix(&s[6..8], 16).map_err(|_| ColorParseError::InvalidHexDigit{ input: hex_string.to_string(), component: "A".to_string()})?
        } else {
            255 // Default alpha to fully opaque
        };
        Ok(Color { r, g, b, a })
    }

    pub fn to_hex(&self, include_alpha: bool) -> String {
        if include_alpha {
            format!("#{:02X}{:02X}{:02X}{:02X}", self.r, self.g, self.b, self.a)
        } else {
            format!("#{:02X}{:02X}{:02X}", self.r, self.g, self.b)
        }
    }
}
```

Attribute: r, g, b, a: u8 - Sichtbarkeit: pub. Rot-, Grün-, Blau- und Alpha-Komponenten.
Methoden: Siehe Definition.
Invarianten: Keine.
Orientation
Datei: nova_de/core/src/types/orientation.rs
Definition:
Rust#
#
pub enum Orientation {
    Horizontal,
    Vertical,
}


Varianten: Horizontal, Vertical.
Re-exportierte Typen
Datei: nova_de/core/src/types/mod.rs
Rust//... imports für Point, Size, Rect, Color, Orientation...
pub use uuid::Uuid;
pub use chrono::{DateTime, Utc};

pub type NotificationId = Uuid;
// Weitere Typ-Aliase können hier definiert werden.

Die Verwendung von uuid::Uuid für NotificationId 5 und chrono::DateTime<Utc> für Zeitstempel 6 ist eine robuste Wahl, die Standardbibliotheken für diese Zwecke nutzt. Die Aktivierung der "serde" und "v4" Features für uuid ist wichtig für die Serialisierung und die Generierung von Typ-4-UUIDs.
Diese grundlegenden Typen bilden eine solide Basis für geometrische Berechnungen, Farbdarstellungen und eindeutige Identifikatoren im gesamten NovaDE-System. Ihre konsequente Verwendung über alle Schichten hinweg fördert die Interoperabilität und reduziert Konvertierungsaufwand.2. Schnittstelle: Domänenschicht (Domain Layer) zu Systemschicht und BenutzeroberflächenschichtDie Domänenschicht kapselt die Geschäftslogik und den Kernzustand der Desktop-Umgebung. Sie stellt ihre Funktionalität über klar definierte Service-APIs (oft als Rust-Traits), Datenstrukturen und Events bereit.2.1. Bereitgestellte Funktionalität
Service-APIs (Traits): Öffentliche Schnittstellen werden primär durch Rust-Traits definiert, die von Service-Strukturen innerhalb der Domänenmodule implementiert werden.

domain::theming::ThemingEngine: Methoden wie get_current_theme_state(), update_configuration().
domain::workspaces::WorkspaceManager: Methoden wie create_workspace(), set_active_workspace().
domain::user_centric_services::AIInteractionLogicService: Methoden wie initiate_interaction(), provide_consent().
domain::user_centric_services::NotificationService: Methoden wie post_notification(), get_active_notifications().
domain::global_settings_and_state_management::GlobalSettingsService: Methoden wie load_settings(), update_setting().


Datenstrukturen: Öffentliche Datenstrukturen, die Zustände oder Konfigurationen repräsentieren (z.B. AppliedThemeState, Workspace, Notification, GlobalDesktopSettings).
Events: Domänenspezifische Events, die Zustandsänderungen signalisieren.

Beispiele: ThemeChangedEvent, WorkspaceEvent (z.B. ActiveWorkspaceChanged), NotificationPostedEvent, SettingChangedEvent.


Fehlertypen: Modulspezifische Fehler-Enums (z.B. ThemingError, WorkspaceManagerError, AIInteractionError, GlobalSettingsError).
2.2. Nutzung durch die Systemschicht
Die Systemschicht konsumiert die Service-APIs der Domänenschicht, um Geschäftsregeln anzuwenden und Zustände abzufragen.

Der system::compositor nutzt domain::window_management_policy für Fensterplatzierungsrichtlinien.
Der system::mcp Client interagiert mit AIInteractionLogicService für Einwilligungsprüfungen und Kontextinformationen.
system::dbus (für Benachrichtigungen) interagiert mit NotificationService.


Die Systemschicht kann auf Domänen-Events reagieren, um ihr Verhalten anzupassen (z.B. Umschalten der sichtbaren Surfaces im Compositor bei ActiveWorkspaceChanged).
Fehler aus der Domänenschicht werden in der Systemschicht behandelt oder weitergeleitet.
2.3. Nutzung durch die Benutzeroberflächenschicht
Die UI-Schicht nutzt die Service-APIs der Domänenschicht, um Daten für die Darstellung abzurufen und Benutzeraktionen in Domänenlogik umzusetzen.

ui::control_center verwendet GlobalSettingsService zum Anzeigen und Ändern von Einstellungen.
ui::shell interagiert mit WorkspaceManager für die Workspace-Darstellung und -Navigation.
ui::theming_gtk reagiert auf ThemeChangedEvent und wendet Stile an.


Die UI-Schicht abonniert Domänen-Events, um ihre Ansichten dynamisch zu aktualisieren.
Fehler aus der Domänenschicht werden von der UI-Schicht behandelt und dem Benutzer ggf. in verständlicher Form präsentiert.
2.4. Ultra-Feinspezifikation: Modul domain::user_centric_services::notification_service_implDieses Modul enthält die konkrete Implementierung des NotificationService-Traits.2.4.1. Modulübersicht
Zweck: Verwaltung des Lebenszyklus von Benachrichtigungen, einschließlich Erstellung, Speicherung, Abruf aktiver Benachrichtigungen, Schließen und Auslösen von Aktionen. Veröffentlicht Domänenereignisse im Zusammenhang mit Benachrichtigungen.
Dateistruktur:

nova_de/domain/src/user_centric_services/notification_api.rs: Definiert das Trait NotificationService und zugehörige öffentliche Typen wie Notification, NotificationAction, NotificationUrgency, NotificationId, CloseReason und NotificationError.
nova_de/domain/src/user_centric_services/notification_service_impl.rs: Enthält die Struktur NotificationServiceImpl und deren Implementierung des NotificationService-Traits.
nova_de/domain/src/events.rs: Definiert das DomainEvent-Enum, das NotificationPosted und NotificationClosed enthält.


Abhängigkeiten:

Extern: uuid (für NotificationId), chrono (für Zeitstempel).
Intern: crate::core::types::{NotificationId,...} (falls dort definiert, sonst lokal), crate::core::errors::DomainError (oder spezifischer NotificationError), crate::event_bus::{EventPublisher, DomainEvent} (abstrahiert den Event-Bus).


2.4.2. Öffentliche Datentypen (definiert in notification_api.rs)

pub type NotificationId = uuid::Uuid; 5

Verwendet Uuid::new_v4() zur Generierung.



##pub struct NotificationAction {pub key: String,pub label: String,}

key: Eindeutiger Bezeichner für die Aktion innerhalb der Benachrichtigung.
label: Für den Benutzer sichtbarer Text der Aktion.



##pub enum NotificationUrgency { Low, Normal, Critical }


##pub struct Notification {pub id: NotificationId,pub app_name: String,pub app_icon: Option<String>,pub summary: String,pub body: Option<String>,pub actions: Vec<NotificationAction>,pub hints: std::collections::HashMap<String, serde_json::Value>, // Flexibel für diverse Hintspub expire_timeout: i32, // Millisekunden. -1: Server-Default, 0: Niepub urgency: NotificationUrgency,pub created_at: chrono::DateTime<chrono::Utc>,}

Attribute wie in der Freedesktop Notification Specification definiert.9
hints: Verwendet serde_json::Value für Flexibilität bei D-Bus-Varianten.



##pub enum CloseReason { Expired, DismissedByUser, ClosedByApp, Replaced }


#pub enum NotificationError {#NotFound(NotificationId),#[error("Invalid parameter for notification: {field}: {message}")]InvalidParameter { field: String, message: String },#ActionNotFound { notification_id: NotificationId, action_key: String },#[error("Internal storage error for notifications")]StorageError(#[source] Box<dyn std::error::Error + Send + Sync + 'static>),}

2.4.3. Interne Datenstrukturen (notification_service_impl.rs)

struct NotificationInternal {public_data: Notification,// Ggf. Felder für Timer-Handles für expire_timeout// Ggf. Persistenz-Status}


pub struct NotificationServiceImpl {active_notifications: std::sync::Mutex<std::collections::HashMap<NotificationId, NotificationInternal>>,event_publisher: std::sync::Arc<dyn crate::event_bus::EventPublisher<crate::events::DomainEvent> + Send + Sync>,// Ggf. ein Tokio Runtime Handle für Timer, falls expire_timeout serverseitig gehandhabt wird}

2.4.4. Implementierung des NotificationService-Traits (notification_service_impl.rs)Rustuse async_trait::async_trait;
use std::sync::{Arc, Mutex};
use std::collections::HashMap;
use uuid::Uuid;
use chrono::Utc;
use crate::event_bus::{EventPublisher, DomainEvent}; // Annahme: DomainEvent ist hier definiert
use super::notification_api::*; // Importiert Traits und öffentliche Typen

impl NotificationServiceImpl {
    pub fn new(event_publisher: Arc<dyn EventPublisher<DomainEvent> + Send + Sync>) -> Self {
        NotificationServiceImpl {
            active_notifications: Mutex::new(HashMap::new()),
            event_publisher,
        }
    }

    // Interne Hilfsfunktion zum Schließen und Benachrichtigen
    fn internal_close_notification(
        &self,
        id: NotificationId,
        reason: CloseReason,
        notifications_guard: &mut std::sync::MutexGuard<HashMap<NotificationId, NotificationInternal>>,
    ) -> Result<Notification, NotificationError> {
        if let Some(removed_notification_internal) = notifications_guard.remove(&id) {
            let event = DomainEvent::NotificationClosed {
                id,
                reason,
                app_name: removed_notification_internal.public_data.app_name.clone(), // Für Kontext im Event
            };
            if self.event_publisher.publish(&event).is_err() {
                // Fehler beim Publishen loggen, aber nicht unbedingt den Vorgang abbrechen
                tracing::error!("Failed to publish NotificationClosed event for ID: {}", id);
            }
            Ok(removed_notification_internal.public_data)
        } else {
            Err(NotificationError::NotFound(id))
        }
    }
}

#[async_trait]
impl NotificationService for NotificationServiceImpl {
    async fn post_notification(
        &self, // &self statt &mut self, da Mutex interne Mutabilität ermöglicht
        app_name: String,
        replaces_id: Option<NotificationId>, // Verwende direkt NotificationId
        app_icon: Option<String>,
        summary: String,
        body: Option<String>,
        actions: Vec<NotificationAction>,
        hints: HashMap<String, serde_json::Value>,
        expire_timeout: i32,
        urgency: NotificationUrgency,
    ) -> Result<NotificationId, NotificationError> {
        let mut notifications_guard = self.active_notifications.lock().map_err(|_| {
            NotificationError::StorageError(Box::new(std::io::Error::new(
                std::io::ErrorKind::Other,
                "Mutex poisoned",
            )))
        })?;

        let new_id = if let Some(id_to_replace) = replaces_id {
            // Versuche, die alte Benachrichtigung zu entfernen und zu benachrichtigen
            let _ = self.internal_close_notification(id_to_replace, CloseReason::Replaced, &mut notifications_guard);
            id_to_replace // Verwende die ID der ersetzten Benachrichtigung
        } else {
            Uuid::new_v4() // Generiere eine neue ID [5]
        };

        let notification_data = Notification {
            id: new_id,
            app_name,
            app_icon,
            summary,
            body,
            actions,
            hints,
            expire_timeout,
            urgency,
            created_at: Utc::now(),
        };

        let internal_notification = NotificationInternal {
            public_data: notification_data.clone(),
        };

        notifications_guard.insert(new_id, internal_notification);
        
        // TODO: Wenn expire_timeout > 0, hier Timer-Logik implementieren,
        // die self.close_notification(new_id, CloseReason::Expired) aufruft.
        // Dies erfordert typischerweise einen Tokio-Task oder ähnliches.

        let event = DomainEvent::NotificationPosted(notification_data);
        if self.event_publisher.publish(&event).is_err() {
            tracing::error!("Failed to publish NotificationPosted event for ID: {}", new_id);
            // Hier könnte man entscheiden, ob der Fehler an den Aufrufer weitergegeben wird.
            // Fürs Erste wird der Fehler nur geloggt.
        }

        Ok(new_id)
    }

    async fn get_active_notifications(&self) -> Result<Vec<Notification>, NotificationError> {
        let notifications_guard = self.active_notifications.lock().map_err(|_| {
            NotificationError::StorageError(Box::new(std::io::Error::new(
                std::io::ErrorKind::Other,
                "Mutex poisoned",
            )))
        })?;
        Ok(notifications_guard.values().map(|internal| internal.public_data.clone()).collect())
    }

    async fn close_notification(&self, id: NotificationId, reason: CloseReason) -> Result<(), NotificationError> {
        let mut notifications_guard = self.active_notifications.lock().map_err(|_| {
            NotificationError::StorageError(Box::new(std::io::Error::new(
                std::io::ErrorKind::Other,
                "Mutex poisoned",
            )))
        })?;
        
        self.internal_close_notification(id, reason, &mut notifications_guard)?;
        Ok(())
    }

    async fn invoke_action(&self, id: NotificationId, action_key: String) -> Result<(), NotificationError> {
        let notifications_guard = self.active_notifications.lock().map_err(|_| {
            NotificationError::StorageError(Box::new(std::io::Error::new(
                std::io::ErrorKind::Other,
                "Mutex poisoned",
            )))
        })?;

        match notifications_guard.get(&id) {
            Some(notification_internal) => {
                if notification_internal.public_data.actions.iter().any(|act| act.key == action_key) {
                    // Aktion existiert
                    let event = DomainEvent::NotificationActionInvoked {
                        id,
                        action_key: action_key.clone(),
                        app_name: notification_internal.public_data.app_name.clone(),
                    };
                    if self.event_publisher.publish(&event).is_err() {
                         tracing::error!("Failed to publish NotificationActionInvoked event for ID: {}", id);
                    }
                    // Die D-Bus Spezifikation sagt nicht explizit, dass eine Benachrichtigung nach ActionInvoked geschlossen werden muss.
                    // Das überlassen wir der UI-Schicht oder spezifischen Hints.
                    Ok(())
                } else {
                    Err(NotificationError::ActionNotFound { notification_id: id, action_key })
                }
            }
            None => Err(NotificationError::NotFound(id)),
        }
    }
}
2.4.5. Domänenereignisse (events.rs)
#
pub enum DomainEvent {
NotificationPosted(Notification),
NotificationClosed { id: NotificationId, reason: CloseReason, app_name: String },
NotificationActionInvoked { id: NotificationId, action_key: String, app_name: String },
//... andere Domänenereignisse
}
Publisher: NotificationServiceImpl.
Typische Subscriber:

system::dbus::notification_dbus_service: Um D-Bus Signale zu emittieren.
ui::notifications_frontend: Um die Benutzeroberfläche zu aktualisieren.


Die Implementierung des NotificationService stellt sicher, dass die Kernlogik der Benachrichtigungsverwaltung von externen Schnittstellendetails (wie D-Bus) entkoppelt ist. Die Verwendung eines internen Event-Bus ermöglicht es anderen Systemteilen, reaktiv auf Benachrichtigungsänderungen zu reagieren.3. Schnittstelle: Systemschicht (System Layer) zu Benutzeroberflächenschicht (UI Layer)Die Systemschicht stellt der UI-Schicht systemnahe Dienste und Ereignisse zur Verfügung und setzt deren Befehle technisch um.3.1. Bereitgestellte Funktionalität
Systemereignisse und -zustände:

Fensterinformationen: Geometrie, Titel, AppID, Fokusstatus von Fenstern (aus system::compositor und system::xdg_shell).
Eingabeereignisse: Verarbeitete Tastatur-, Maus-, Touch- und Gestenereignisse (aus system::input).
Output-Informationen: Verfügbare Monitore, Auflösungen, Skalierungsfaktoren (aus system::outputs).
Status von Systemdiensten: Netzwerkverbindungen (system::dbus::networkmanager_client), Energiestatus (system::dbus::upower_client), Audiostatus (system::audio).
Sitzungsereignisse: Sperren, Abmelden (von system::dbus::logind_client).


Ausführung von UI-Befehlen:

Fenstermanipulationen (Verschieben, Größe ändern, Fokus setzen), initiiert durch die UI, werden vom system::compositor und system::window_mechanics umgesetzt.
Workspace-Wechsel.
Anpassung von Systemeinstellungen (z.B. Bildschirmhelligkeit, Lautstärke), die von system::outputs bzw. system::audio ausgeführt werden.


Fehlertypen: Modulspezifische Fehler-Enums (z.B. CompositorCoreError, InputError).
3.2. Nutzung durch die Benutzeroberflächenschicht
Darstellung von Systeminformationen: Die UI visualisiert die von der Systemschicht bereitgestellten Zustände.

Fensterlisten, Titelleisten, Fokus-Hervorhebungen basieren auf Daten von system::compositor.
Netzwerk-, Batterie-, Audio-Indikatoren in ui::shell zeigen Daten von system::dbus und system::audio.


Reaktion auf Eingabeereignisse: UI-Elemente reagieren auf verarbeitete Eingabeereignisse, um Aktionen auszulösen.
Initiierung von Systemaktionen: Benutzerinteraktionen in der UI führen zu Befehlsaufrufen an die Systemschicht.

Klick auf "Lauter"-Button in ui::shell ruft eine Funktion in system::audio auf.
Auswahl eines anderen Monitorsetups in ui::control_center sendet Befehl an system::outputs.


Dialoge über XDG Portals: ui::shell oder ui::components interagieren mit system::portals für Datei-Auswahl- oder Screenshot-Dialoge.
Die UI-Schicht behandelt Fehler von der Systemschicht und informiert ggf. den Benutzer.
Die UI-Schicht reagiert auf Systemereignisse (z.B. ActiveWorkspaceChanged indirekt über Änderungen der sichtbaren Fenster, DeviceAdded für Eingabegeräte), um ihre Darstellung anzupassen.
3.3. Ultra-Feinspezifikation: Modul system::dbus::notification_dbus_serviceDieses Modul implementiert den org.freedesktop.Notifications D-Bus Standard und dient als Brücke zur Domänenschicht (NotificationService).3.3.1. Modulübersicht
Zweck: Bereitstellung der org.freedesktop.Notifications D-Bus-Schnittstelle. Nimmt D-Bus-Methodenaufrufe entgegen, leitet sie an den NotificationService der Domänenschicht weiter und emittiert D-Bus-Signale basierend auf Domänenereignissen.
Dateistruktur: nova_de/system/src/dbus/notification_dbus_service.rs
Abhängigkeiten:

Extern: zbus (für D-Bus-Interaktion) 10, async-trait.
Intern: Arc<dyn domain::user_centric_services::NotificationService>, domain::events::DomainEvent, crate::event_bus::{EventSubscriber, DomainEvent}.


3.3.2. Datenstrukturen

struct NotificationDBusService {notification_service: Arc<dyn NotificationService>,connection: Arc<zbus::Connection>,// Für die Konvertierung von NotificationId (UUID) zu u32 (D-Bus ID) und umgekehrt:id_map: Arc<Mutex<IdMapper>>,}


struct IdMapper {next_dbus_id: u32,uuid_to_dbus: HashMap<NotificationId, u32>,dbus_to_uuid: HashMap<u32, NotificationId>,}

IdMapper ist notwendig, da die D-Bus-Spezifikation UINT32 für Benachrichtigungs-IDs verwendet, während intern uuid::Uuid für NotificationId genutzt wird, um globale Eindeutigkeit zu gewährleisten.


3.3.3. D-Bus SchnittstellenimplementierungDie Implementierung erfolgt mittels des #[dbus_interface] Makros von zbus.12Rust// nova_de/system/src/dbus/notification_dbus_service.rs
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use zbus::{dbus_interface, fdo, zvariant::Value, SignalContext};
use crate::domain::user_centric_services::notification_api::{
    NotificationService, NotificationId, NotificationAction, NotificationUrgency, CloseReason, NotificationError
};
use crate::event_bus::{DomainEvent, EventSubscriber}; // Angenommen, EventSubscriber ist definiert

// Hilfsstruktur für die ID-Konvertierung
struct IdMapper {
    next_dbus_id: u32,
    uuid_to_dbus: HashMap<NotificationId, u32>,
    dbus_to_uuid: HashMap<u32, NotificationId>,
}

impl IdMapper {
    fn new() -> Self {
        IdMapper {
            next_dbus_id: 1, // D-Bus IDs sind > 0
            uuid_to_dbus: HashMap::new(),
            dbus_to_uuid: HashMap::new(),
        }
    }

    fn get_or_create_dbus_id(&mut self, uuid: NotificationId) -> u32 {
        if let Some(dbus_id) = self.uuid_to_dbus.get(&uuid) {
            *dbus_id
        } else {
            let dbus_id = self.next_dbus_id;
            self.next_dbus_id += 1;
            self.uuid_to_dbus.insert(uuid, dbus_id);
            self.dbus_to_uuid.insert(dbus_id, uuid);
            dbus_id
        }
    }

    fn get_uuid(&self, dbus_id: u32) -> Option<NotificationId> {
        self.dbus_to_uuid.get(&dbus_id).copied()
    }

    fn remove_mapping_by_uuid(&mut self, uuid: &NotificationId) {
        if let Some(dbus_id) = self.uuid_to_dbus.remove(uuid) {
            self.dbus_to_uuid.remove(&dbus_id);
        }
    }
    
    fn remove_mapping_by_dbus_id(&mut self, dbus_id: u32) {
        if let Some(uuid) = self.dbus_to_uuid.remove(&dbus_id) {
            self.uuid_to_dbus.remove(&uuid);
        }
    }
}

pub struct NotificationDBusService {
    notification_service: Arc<dyn NotificationService + Send + Sync>,
    connection: Arc<zbus::Connection>,
    id_map: Arc<Mutex<IdMapper>>,
}

impl NotificationDBusService {
    pub async fn new(
        notification_service: Arc<dyn NotificationService + Send + Sync>,
        connection: Arc<zbus::Connection>,
        event_subscriber: Arc<dyn EventSubscriber<DomainEvent> + Send + Sync>, // Für das Abonnieren von DomainEvents
    ) -> Result<Self, zbus::Error> {
        let service = Self {
            notification_service,
            connection: connection.clone(),
            id_map: Arc::new(Mutex::new(IdMapper::new())),
        };

        // Task zum Abonnieren von DomainEvents und Emittieren von D-Bus Signalen
        let id_map_clone = service.id_map.clone();
        let conn_clone = connection.clone();
        tokio::spawn(async move {
            let mut stream = event_subscriber.subscribe();
            while let Some(event_result) = stream.recv().await { // Angenommen, subscribe() gibt einen Stream/Channel
                if let Ok(event) = event_result {
                    match event {
                        DomainEvent::NotificationClosed { id, reason, app_name: _ } => {
                            let dbus_id = {
                                let mut map = id_map_clone.lock().unwrap();
                                let id_val = map.get_uuid_to_dbus_id(id); // Hilfsfunktion in IdMapper
                                if id_val.is_some() { map.remove_mapping_by_uuid(&id); }
                                id_val
                            };
                            if let Some(valid_dbus_id) = dbus_id {
                                let dbus_reason = match reason {
                                    CloseReason::Expired => 1,
                                    CloseReason::DismissedByUser => 2,
                                    CloseReason::ClosedByApp => 3,
                                    CloseReason::Replaced => 3, // Oder eine spezifische D-Bus Reason, falls vorhanden
                                };
                                if let Err(e) = NotificationDBusService::emit_notification_closed(&conn_clone, valid_dbus_id, dbus_reason).await {
                                    tracing::error!("Failed to emit NotificationClosed D-Bus signal: {}", e);
                                }
                            }
                        }
                        DomainEvent::NotificationActionInvoked { id, action_key, app_name: _ } => {
                             let dbus_id = {
                                let map = id_map_clone.lock().unwrap();
                                map.get_uuid_to_dbus_id(id) // Hilfsfunktion in IdMapper
                            };
                            if let Some(valid_dbus_id) = dbus_id {
                                if let Err(e) = NotificationDBusService::emit_action_invoked(&conn_clone, valid_dbus_id, &action_key).await {
                                    tracing::error!("Failed to emit ActionInvoked D-Bus signal: {}", e);
                                }
                            }
                        }
                        _ => {} // Andere DomainEvents ignorieren
                    }
                }
            }
        });
        
        Ok(service)
    }

    // Hilfsfunktionen zum Emittieren von Signalen
    async fn emit_notification_closed(connection: &Arc<zbus::Connection>, id: u32, reason: u32) -> zbus::Result<()> {
        let ctxt = SignalContext::new(connection, "/org/freedesktop/Notifications")?;
        NotificationDBusService::notification_closed(&ctxt, id, reason).await
    }

    async fn emit_action_invoked(connection: &Arc<zbus::Connection>, id: u32, action_key: &str) -> zbus::Result<()> {
        let ctxt = SignalContext::new(connection, "/org/freedesktop/Notifications")?;
        NotificationDBusService::action_invoked(&ctxt, id, action_key).await
    }
}


#[dbus_interface(name = "org.freedesktop.Notifications")]
impl NotificationDBusService {
    async fn Notify(
        &self,
        app_name: String,
        replaces_id_dbus: u32,
        app_icon: String,
        summary: String,
        body: String,
        actions_dbus: Vec<String>,
        hints: HashMap<String, Value<'_>>, // zbus::zvariant::Value
        expire_timeout: i32,
    ) -> fdo::Result<u32> {
        tracing::debug!(
            "D-Bus Notify called: app_name={}, replaces_id={}, summary={}",
            app_name, replaces_id_dbus, summary
        );

        let replaces_id_uuid: Option<NotificationId> = if replaces_id_dbus == 0 {
            None
        } else {
            self.id_map.lock().unwrap().get_uuid(replaces_id_dbus)
        };

        let mut parsed_actions = Vec::new();
        for i in (0..actions_dbus.len()).step_by(2) {
            if i + 1 < actions_dbus.len() {
                parsed_actions.push(NotificationAction {
                    key: actions_dbus[i].clone(),
                    label: actions_dbus[i + 1].clone(),
                });
            }
        }
        
        // Konvertiere hints von zbus::zvariant::Value zu serde_json::Value
        let mut converted_hints = HashMap::new();
        for (k, v_ref) in hints.iter() {
            // Direkte Konvertierung ist möglicherweise nicht trivial.
            // Eine einfache Variante ist, wenn die Value-Typen kompatibel sind oder eine Serialisierung/Deserialisierung erfolgt.
            // Hier wird angenommen, dass die Struktur von Value serialisierbar zu JSON ist.
            // In einer echten Implementierung wäre hier robustere Konvertierungslogik nötig.
            match serde_json::to_value(v_ref) {
                Ok(json_val) => { converted_hints.insert(k.clone(), json_val); },
                Err(e) => {
                    tracing::warn!("Could not convert hint '{}' to JSON: {}", k, e);
                    // Optional: Fehler zurückgeben oder Hint ignorieren
                    // return Err(fdo::Error::InvalidArgs("Invalid hint format".into()));
                }
            }
        }


        // TODO: Urgency aus hints extrahieren, falls vorhanden (z.B. "urgency" byte)
        let urgency = NotificationUrgency::Normal; // Default, oder aus Hints bestimmen

        match self.notification_service.post_notification(
            app_name,
            replaces_id_uuid,
            if app_icon.is_empty() { None } else { Some(app_icon) },
            summary,
            if body.is_empty() { None } else { Some(body) },
            parsed_actions,
            converted_hints,
            expire_timeout,
            urgency,
        ).await {
            Ok(new_uuid) => {
                let dbus_id = self.id_map.lock().unwrap().get_or_create_dbus_id(new_uuid);
                Ok(dbus_id)
            }
            Err(e) => {
                tracing::error!("Error posting notification via service: {:?}", e);
                Err(fdo::Error::Failed(format!("Internal error: {}", e)))
            }
        }
    }

    async fn CloseNotification(&self, id: u32) -> fdo::Result<()> {
        tracing::debug!("D-Bus CloseNotification called for ID: {}", id);
        let uuid_to_close = match self.id_map.lock().unwrap().get_uuid(id) {
            Some(uuid) => uuid,
            None => {
                // Gemäß Spezifikation: "If the notification no longer exists, an empty D-BUS Error message is sent back."
                // Ein leerer fdo::Error::Failed("") ist nicht ideal, aber zbus erfordert einen Error-Namen.
                // org.freedesktop.DBus.Error.UnknownMethod ist nicht passend.
                // Ein benutzerdefinierter Fehler oder ein stillschweigendes Ok(()) könnte hier besser sein,
                // aber die Spezifikation verlangt einen Fehler.
                // Wir verwenden hier Failed, da kein spezifischerer Fehlercode für "existiert nicht mehr" vorgesehen ist.
                return Err(fdo::Error::Failed("Notification not found.".into()));
            }
        };

        match self.notification_service.close_notification(uuid_to_close, CloseReason::ClosedByApp).await {
            Ok(()) => {
                self.id_map.lock().unwrap().remove_mapping_by_dbus_id(id); // Mapping entfernen
                Ok(())
            }
            Err(NotificationError::NotFound(_)) => {
                 self.id_map.lock().unwrap().remove_mapping_by_dbus_id(id); // Sicherstellen, dass das Mapping entfernt ist
                 Err(fdo::Error::Failed("Notification not found by service.".into())) // Gemäß Spezifikation
            }
            Err(e) => {
                tracing::error!("Error closing notification via service: {:?}", e);
                Err(fdo::Error::Failed(format!("Internal error: {}", e)))
            }
        }
    }

    async fn GetCapabilities(&self) -> fdo::Result<Vec<String>> {
        tracing::debug!("D-Bus GetCapabilities called");
        Ok(vec!)
    }

    async fn GetServerInformation(&self) -> fdo::Result<(String, String, String, String)> {
        tracing::debug!("D-Bus GetServerInformation called");
        Ok((
            "NovaDE".to_string(),
            "NovaDE Project".to_string(),
            env!("CARGO_PKG_VERSION").to_string(), // Version aus Cargo.toml
            "1.2".to_string(), // Implementierte Spezifikationsversion
        ))
    }

    // Definition der Signale
    #[dbus_interface(signal)]
    async fn NotificationClosed(ctxt: &SignalContext<'_>, id: u32, reason: u32) -> zbus::Result<()>;

    #[dbus_interface(signal)]
    async fn ActionInvoked(ctxt: &SignalContext<'_>, id: u32, action_key: &str) -> zbus::Result<()>;
}
Hinweise zur Implementierung:
ID Mapping: Eine IdMapper-Struktur (oder ein ähnlicher Mechanismus) ist entscheidend, um zwischen den internen uuid::Uuid-basierten NotificationIds und den u32-basierten IDs der D-Bus-Schnittstelle zu übersetzen. replaces_id = 0 bedeutet eine neue Benachrichtigung. Zurückgegebene IDs müssen eindeutig sein.
Action Parsing: Die actions-Parameterliste im Notify-Aufruf ist ein flaches Array von Strings (as), wobei gerade Indizes Aktionsschlüssel und ungerade Indizes die zugehörigen Labels sind.9 Dies muss in die interne Vec<NotificationAction>-Struktur geparst werden.
Hints: hints ist ein a{sv} (Dictionary String to Variant). zbus::zvariant::Value muss in den internen Typ für Hint-Werte konvertiert werden (hier serde_json::Value als Beispiel). Die urgency (Dringlichkeit) und expire_timeout können auch über Hints gesteuert werden.
Signal Emission: Die D-Bus-Signale NotificationClosed und ActionInvoked müssen emittiert werden, wenn die entsprechenden DomainEvents (NotificationClosed, NotificationActionInvoked) vom NotificationService empfangen werden. Dies erfordert, dass NotificationDBusService den internen Event-Bus abonniert und in einem separaten Task auf diese Events lauscht.
Fehlerbehandlung: Interne NotificationError-Typen müssen in zbus::fdo::Error umgewandelt werden. Die D-Bus-Spezifikation gibt vor, dass bei CloseNotification für eine nicht existierende ID ein leerer D-Bus-Fehler zurückgegeben wird.
Asynchronität: Alle D-Bus-Methoden sind async. Die Interaktion mit dem NotificationService (der ebenfalls async sein kann) erfolgt über await.
Diese Implementierung stellt sicher, dass NovaDE sich standardkonform über D-Bus für Benachrichtigungen verhält und gleichzeitig die interne Logik in der Domänenschicht sauber gekapselt bleibt.3.3.4. Tabelle: org.freedesktop.Notifications D-Bus API zu NotificationService MappingD-Bus MemberTyp (Methode/Signal)NovaDE NotificationService Methode/DomänenEventSchlüsselparameter-/Payload-Mapping (D-Bus -> Intern)AnmerkungenNotifyMethodeasync post_notification(...)app_name -> app_name, replaces_id_dbus (u32) -> replaces_id (Option<NotificationId>), app_icon -> app_icon, summary -> summary, body -> body, actions_dbus (Vec<String>) -> actions (Vec<NotificationAction>), hints (HashMap<String, zbus::zvariant::Value>) -> hints (HashMap<String, serde_json::Value>), expire_timeout -> expire_timeout, urgency (aus Hints oder Default) -> urgencyID-Mapping für replaces_id und Rückgabewert erforderlich. Parsing der Aktionsliste. Hint-Konvertierung.CloseNotificationMethodeasync close_notification(...)id (u32) -> id (NotificationId)ID-Mapping. reason wird intern als CloseReason::ClosedByApp gesetzt.GetCapabilitiesMethodeN/A (Direkt in D-Bus Modul implementiert)N/AGibt statische Liste der Server-Fähigkeiten zurück (z.B. "body", "actions").GetServerInformationMethodeN/A (Direkt in D-Bus Modul implementiert)N/AGibt statische Serverinformationen zurück (Name, Vendor, Version, Spec-Version).NotificationClosedSignalDomainEvent::NotificationClosedid (NotificationId) -> id (u32), reason (CloseReason) -> reason (u32)ID-Mapping. Enum-Mapping für reason. Emittiert, wenn internes Event empfangen wird.ActionInvokedSignalDomainEvent::NotificationActionInvoked (löst async invoke_action(...) aus, welches dieses Event publiziert)id (NotificationId) -> id (u32), action_key (String) -> action_key (String)ID-Mapping. Emittiert, wenn internes Event empfangen wird.Diese Tabelle verdeutlicht die Kopplungspunkte und notwendigen Transformationen zwischen der standardisierten D-Bus-Schnittstelle und der internen Implementierung des Benachrichtigungsdienstes in NovaDE.4. Schnittstelle: Benutzeroberflächenschicht (UI Layer) zu Domänenschicht und SystemschichtDie UI-Schicht initiiert Aktionen in den unteren Schichten basierend auf Benutzerinteraktionen und stellt Informationen aus diesen Schichten dar.4.1. Gesendete Befehle/Daten an die Domänenschicht
Theming: ThemingEngine::update_configuration(new_config: ThemingConfiguration) durch ui::control_center oder ui::theming_gtk.
Workspaces: WorkspaceManager::create_workspace(name: Option<String>), set_active_workspace(id: WorkspaceId), pin_application_to_space(space_id: WorkspaceId, app_id: ApplicationIdentifier) durch ui::shell (Workspace-Switcher, SmartTabBar) oder ui::window_manager_frontend (Übersichtsmodus).
AI: AIInteractionLogicService::initiate_interaction(context_data: AIInteractionContextData), provide_consent(consent: AIConsent) durch ui::command_palette oder spezifische KI-Widgets.
Notifications: NotificationService::dismiss_notification(id: NotificationId), invoke_action(id: NotificationId, action_key: String) durch ui::notifications_frontend oder ui::shell (NotificationCenter).
Settings: GlobalSettingsService::update_setting(path: SettingPath, value: SettingValue) durch ui::control_center oder Quick-Settings.
4.2. Gesendete Befehle/Daten an die Systemschicht
Fenstermanagement: An system::window_mechanics oder direkt an system::compositor (über eine Fassade):

request_focus(window_id: DomainWindowIdentifier)
request_close_window(window_id: DomainWindowIdentifier)
request_move_window(window_id: DomainWindowIdentifier, new_pos: Point<i32, Logical>) (oder Start eines interaktiven Moves)
request_resize_window(window_id: DomainWindowIdentifier, new_size: Size<i32, Logical>) (oder Start eines interaktiven Resizes)
request_set_window_state(window_id: DomainWindowIdentifier, state: WindowState (Maximized, Minimized, Fullscreen))


Output-Konfiguration: An system::outputs (via wlr-output-management oder eine interne API):

apply_output_configuration(config: Vec<OutputConfigChange>) durch ui::control_center.


Audio-Steuerung: An system::audio::PipeWireClientService:

set_sink_volume(device_id, volume_percent), set_sink_mute(device_id, mute) durch ui::shell (Quick-Settings) oder ui::control_center.


Systemaktionen: An system::dbus Clients:

logind_client.lock_current_session() durch ui::shell (Lock-Button).
network_manager_client.activate_connection(connection_path, device_path) durch ui::shell (Quick-Settings) oder ui::control_center.


MCP-Anfragen: An system::mcp::SystemMcpService:

call_tool(server_id, tool_name, params, interaction_id) durch ui::command_palette oder KI-Widgets.


4.3. Ultra-Feinspezifikation: Modul ui::portals::file_chooser_portal_clientDieses Modul in der UI-Schicht ist verantwortlich für die Interaktion mit dem XDG Desktop Portal für Dateiauswahldialoge. Es agiert als Client für die D-Bus-Schnittstelle org.freedesktop.portal.FileChooser.4.3.1. Modulübersicht
Zweck: Kapselt die Logik zur Kommunikation mit dem XDG FileChooser-Portal, um native Dateiauswahldialoge zu öffnen und die Ergebnisse zu verarbeiten.
Dateistruktur: nova_de/ui/src/portals/file_chooser_portal_client.rs
Abhängigkeiten:

Extern: zbus (für D-Bus-Client-Interaktion) 10, tokio (für asynchrone Operationen).
Intern: core::errors::UIError (oder spezifischer PortalError), core::types.


4.3.2. Interne Datenstrukturen und Typen

struct FileChooserPortalClient { connection: Arc<zbus::Connection> }

Hält die D-Bus-Verbindung.



pub struct OpenFileOptions {pub parent_window_id: Option<String>, // XDG Window Identifierpub title: String,pub accept_label: Option<String>,pub modal: bool,pub multiple: bool,pub directory: bool,pub filters: Vec<FileFilter>,pub current_filter: Option<FileFilter>,pub current_folder: Option<PathBuf>,}

Spiegelt die Optionen des OpenFile-D-Bus-Aufrufs.15



pub struct FileFilter {pub name: String,pub patterns: Vec<FilterPattern>, // (u32 type, String pattern)}

FilterPattern: pub enum FilterPattern { Glob(String), MimeType(String) }



pub struct SaveFileOptions {... } (analog zu OpenFileOptions)


pub enum FileChooserResponse {SelectedFiles(Vec<url::Url>),SelectedFolder(url::Url),Cancelled,Error(PortalError),}


#pub enum PortalError {#DBusConnection(#[from] zbus::Error),#DBusCall(#[from] zbus::fdo::Error),#[error("Portal request failed with code {code}: {message}")]RequestFailed { code: u32, message: String },#InvalidUri(String),#[error("Portal response malformed: {0}")]ResponseMalformed(String),}

4.3.3. D-Bus Proxy Definition (intern)Mittels zbus::proxy wird ein Proxy für org.freedesktop.portal.FileChooser erstellt.10Rust// Innerhalb von file_chooser_portal_client.rs oder einem Hilfsmodul
use zbus::{proxy, zvariant::{Value, ObjectPath, OwnedValue, Dict, Array}};
use std::collections::HashMap;

#
trait FileChooser {
    async fn OpenFile(
        &self,
        parent_window: &str, // Window identifier, "" if none
        title: &str,
        options: HashMap<&str, Value<'_>>,
    ) -> zbus::fdo::Result<ObjectPath<'static>>; // Request handle

    async fn SaveFile(
        &self,
        parent_window: &str,
        title: &str,
        options: HashMap<&str, Value<'_>>,
    ) -> zbus::fdo::Result<ObjectPath<'static>>;

    // SaveFiles nicht im Detail hier, aber analog
}

// Proxy für org.freedesktop.portal.Request, um die Antwort zu empfangen
#
trait PortalRequest {
    #[zbus(signal)]
    async fn Response(&self, response_code: u32, results: Dict<'_, String, OwnedValue>) -> zbus::Result<()>;
    // Methode zum Schließen des Requests, falls benötigt
    // async fn Close(&self) -> zbus::fdo::Result<()>;
}
4.3.4. Implementierung von FileChooserPortalClientRust// nova_de/ui/src/portals/file_chooser_portal_client.rs
use zbus::{Connection, zvariant::{Value, Dict, Array, ObjectPath, Str, Type}, fdo::Error as ZbusFdoError};
use std::collections::HashMap;
use std::path::PathBuf;
use std::sync::Arc;
use url::Url;
use futures_util::stream::StreamExt; // Für das Lauschen auf Signale

//... (Proxy-Definitionen von oben)...
//... (OpenFileOptions, FileFilter, FileChooserResponse, PortalError Definitionen)...

impl FileChooserPortalClient {
    pub fn new(connection: Arc<Connection>) -> Self {
        Self { connection }
    }

    fn build_open_file_options_dict<'a>(
        &self,
        options: &'a OpenFileOptions,
        handle_token: &'a str
    ) -> HashMap<&'a str, Value<'a>> {
        let mut dict = HashMap::<&str, Value<'_>>::new();
        dict.insert("handle_token", Value::from(handle_token));
        if let Some(label) = &options.accept_label {
            dict.insert("accept_label", Value::from(label.as_str()));
        }
        dict.insert("modal", Value::from(options.modal));
        dict.insert("multiple", Value::from(options.multiple));
        dict.insert("directory", Value::from(options.directory)); // Version 3+ [15]

        if!options.filters.is_empty() {
            let mut filters_array = Array::new(Type::Tuple(Arc::new())))])));
            for filter in &options.filters {
                let mut patterns_array = Array::new(Type::Tuple(Arc::new()));
                for pattern_enum in &filter.patterns {
                    match pattern_enum {
                        FilterPattern::Glob(p) => patterns_array.append(Value::from((0u32, p.as_str()))).unwrap(),
                        FilterPattern::MimeType(m) => patterns_array.append(Value::from((1u32, m.as_str()))).unwrap(),
                    }
                }
                filters_array.append(Value::from((Str::from(filter.name.as_str()), patterns_array))).unwrap();
            }
            dict.insert("filters", Value::from(filters_array));
        }
        
        if let Some(current_filter) = &options.current_filter {
            let mut patterns_array = Array::new(Type::Tuple(Arc::new()));
            for pattern_enum in &current_filter.patterns {
                 match pattern_enum {
                    FilterPattern::Glob(p) => patterns_array.append(Value::from((0u32, p.as_str()))).unwrap(),
                    FilterPattern::MimeType(m) => patterns_array.append(Value::from((1u32, m.as_str()))).unwrap(),
                }
            }
            dict.insert("current_filter", Value::from((Str::from(current_filter.name.as_str()), patterns_array)));
        }

        if let Some(folder) = &options.current_folder {
            if let Some(folder_str) = folder.to_str() {
                 // Pfad muss als Array von Bytes (ay) gesendet werden, null-terminiert
                 let mut path_bytes: Vec<u8> = folder_str.as_bytes().to_vec();
                 path_bytes.push(0); // Null-Terminierung
                 dict.insert("current_folder", Value::from(path_bytes));
            }
        }
        dict
    }

    pub async fn open_file_dialog(
        &self,
        options: OpenFileOptions,
    ) -> Result<FileChooserResponse, PortalError> {
        let portal_proxy = FileChooserProxy::new(&self.connection).await?;
        let handle_token = format!("novade_open_{}", uuid::Uuid::new_v4().to_simple()); // Eindeutiger Token [15]

        let options_dict = self.build_open_file_options_dict(&options, &handle_token);
        
        let parent_window_str = options.parent_window_id.as_deref().unwrap_or("");

        let request_path: ObjectPath<'_> = portal_proxy
           .OpenFile(parent_window_str, &options.title, options_dict)
           .await?;
        
        tracing::debug!("OpenFile request sent. Handle: {}", request_path.as_str());

        // Auf das Response-Signal warten
        let request_proxy = PortalRequestProxy::builder(&self.connection)
           .path(request_path.to_owned())? //.to_owned() um Lifetime-Probleme zu vermeiden
           .build()
           .await?;
            
        let mut response_stream = request_proxy.receive_Response().await?;

        if let Some(signal_args) = response_stream.next().await {
            let response_code = signal_args.args()?.response_code;
            let results_dict = signal_args.args()?.results;
            tracing::debug!("Portal Response received: code={}, results={:?}", response_code, results_dict);

            if response_code == 0 { // Erfolg
                if let Some(Value::Array(uris_array)) = results_dict.get("uris") {
                    let mut uris = Vec::new();
                    for val in uris_array.get() {
                        if let Value::Str(s) = val {
                            match Url::parse(s.as_str()) {
                                Ok(url) => uris.push(url),
                                Err(_) => return Err(PortalError::InvalidUri(s.as_str().to_string())),
                            }
                        }
                    }
                    if options.directory {
                        if let Some(first_uri) = uris.into_iter().next() {
                             return Ok(FileChooserResponse::SelectedFolder(first_uri));
                        } else {
                            return Err(PortalError::ResponseMalformed("No URI returned for directory selection".into()));
                        }
                    } else {
                        return Ok(FileChooserResponse::SelectedFiles(uris));
                    }
                } else {
                    return Err(PortalError::ResponseMalformed("Missing 'uris' in results".into()));
                }
            } else if response_code == 1 { // Abgebrochen durch Benutzer
                return Ok(FileChooserResponse::Cancelled);
            } else { // Anderer Fehler
                let message = results_dict.get("message")
                               .and_then(|v| if let Value::Str(s) = v { Some(s.to_string()) } else { None })
                               .unwrap_or_else(|| "Unknown portal error".to_string());
                return Err(PortalError::RequestFailed { code: response_code, message });
            }
        }
        // Timeout oder anderer Fehler beim Warten auf das Signal
        Err(PortalError::ResponseMalformed("No response signal received".into()))
    }

    // pub async fn save_file_dialog(...) -> Result<FileChooserResponse, PortalError> {... }
    // Ähnliche Implementierung wie open_file_dialog, aber mit SaveFile-Methode und Optionen.
}
Implementierungsdetails:
Window Identifier: Der parent_window-Parameter für Portalaufrufe ist ein String. Für Wayland-Oberflächen wird dies typischerweise als wayland:WAYLAND_SURFACE_HANDLE formatiert, wobei der Handle die Objekt-ID der Wayland-Oberfläche ist. Die UI-Schicht muss diesen Bezeichner bereitstellen.
Optionen-Dictionary: Die options für OpenFile und SaveFile werden als HashMap<&str, zbus::zvariant::Value<'_>> konstruiert. Die Schlüssel und Werttypen müssen der XDG Portal Spezifikation entsprechen.15

filters: a(sa(us)) - Array von Tupeln (String, Array von Tupeln (Uint32, String)).
current_folder: ay - Byte-Array (null-terminierter Pfad).


Antwortverarbeitung: Die Antwort des Portals kommt asynchron über das Response-Signal auf dem Request-Objekt. Der Client muss dieses Signal abonnieren und die Ergebnisse (uris, choices) parsen.
Fehlerbehandlung: PortalError kapselt D-Bus-Fehler und spezifische Portal-Fehlercodes.
Asynchronität: Alle D-Bus-Aufrufe und Signal-Listener sind async.
Die Verwendung von XDG Desktop Portals ermöglicht es NovaDE-Anwendungen, die in einer Sandbox laufen könnten, auf sichere Weise mit dem Host-System zu interagieren, z.B. um Dateien zu öffnen oder zu speichern, ohne direkten Dateisystemzugriff zu benötigen. Dies ist ein wichtiger Aspekt moderner Desktop-Sicherheit.5. Zusammenfassende Prinzipien der Schnittstellen
Klare Verantwortlichkeiten: Jede Schicht hat definierte Aufgaben, und die Schnittstellen spiegeln diese wider.
Lose Kopplung: Kommunikation erfolgt über APIs (Traits) und Events, um direkte Abhängigkeiten von Implementierungsdetails zu minimieren.
Hohe Kohäsion: Module innerhalb einer Schicht arbeiten eng zusammen, aber die Schicht als Ganzes hat eine klar definierte externe Schnittstelle.
Einheitliche Fehlerbehandlung: Die thiserror-basierte Strategie wird über alle Schichten hinweg verwendet, wobei Fehler weitergeleitet oder spezifisch behandelt werden. Die source()-Kette ist wichtig.1
Event-gesteuerte Updates: Viele Zustandsänderungen werden über Events kommuniziert, was eine reaktive und entkoppelte Architektur fördert. Die UI-Schicht ist ein primärer Konsument von Events aus der Domänen- und Systemschicht, um ihre Darstellung aktuell zu halten.
Asynchronität: Schnittstellen, die potenziell blockierende Operationen beinhalten (I/O, IPC), sind als async deklariert und nutzen Rusts async/await-Syntax.
6. SchlussfolgerungenDie detaillierte Spezifikation der internen Schnittstellen und die Ultra-Feinspezifikation ausgewählter Module bilden das Rückgrat für die Entwicklung der NovaDE. Durch die klare Definition von Verantwortlichkeiten, Datenstrukturen, Methoden, Events und Fehlerbehandlung wird eine solide Grundlage für parallele Entwicklung und zukünftige Erweiterbarkeit geschaffen.Die konsequente Anwendung von Rusts Typ- und Fehlersystem, kombiniert mit etablierten Mustern wie der Trennung von API (Traits) und Implementierung sowie einem zentralen Event-Bus, trägt maßgeblich zur Robustheit und Wartbarkeit des Systems bei. Die detaillierte Ausarbeitung der Schnittstellen zu Standard-Desktop-Diensten wie org.freedesktop.Notifications und XDG Desktop Portals gewährleistet eine gute Integration in das Linux-Ökosystem.Dieser Implementierungsleitfaden ist so konzipiert, dass Entwickler die einzelnen Module mit einem hohen Grad an Sicherheit und Klarheit implementieren können, wodurch Integrationsrisiken minimiert und die Gesamtqualität der NovaDE maximiert wird. Die durchgängige Verwendung von async/await für potenziell blockierende Operationen stellt sicher, dass die Desktop-Umgebung reaktionsschnell bleibt. Die sorgfältige Definition von Fehlerketten mittels thiserror wird die Fehlersuche und -behebung im gesamten System erheblich vereinfachen.


# NovaDE Interner Schnittstellen-Implementierungsleitfaden

Dieser Implementierungsleitfaden definiert die ultrafeinen Spezifikationen und hochdetaillierten Implementierungspläne für jede Schicht der Linux-Desktop-Umgebung "NovaDE". Er ist so detailliert konzipiert, dass Entwickler ihn direkt zur Implementierung verwenden können, ohne eigene Entscheidungen treffen oder Logiken/Algorithmen entwerfen zu müssen.

---

## 1. Schicht: Kernschicht (Core Layer)

Die Kernschicht (Core Layer) bildet das Fundament von NovaDE und stellt grundlegende, universell einsetzbare Bausteine und Dienste bereit, die von allen darüberliegenden Schichten – der Domänenschicht, der Systemschicht und der Benutzeroberflächenschicht (UI-Schicht) – genutzt werden. Die Kommunikation mit höheren Schichten erfolgt durch die direkte Verwendung der hier definierten Typen und Funktionen.

### 1.1. Modul: `core::types`

Dieses Modul definiert grundlegende Datentypen, die systemweit für geometrische Berechnungen, Farbdarstellungen und andere fundamentale Konzepte verwendet werden.

- **Verzeichnis:** `core/src/types/`

#### 1.1.1. `Point<T>` Struktur

Stellt einen Punkt im 2D-Raum dar.

- Datei: `core/src/types/point.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Point<T, U> {
        pub x: T,
        pub y: T,
        _unit: std::marker::PhantomData<U>,
    }
    
    impl<T: Default, U> Default for Point<T, U> {
        fn default() -> Self {
            Point { x: Default::default(), y: Default::default(), _unit: std::marker::PhantomData }
        }
    }
    
    // U kann z.B. Logical, Physical, BufferCoords sein
    pub struct Logical;
    pub struct Physical;
    pub struct BufferCoords;
    
    impl<T, U> Point<T, U> {
        pub const fn new(x: T, y: T) -> Self {
            Point { x, y, _unit: std::marker::PhantomData }
        }
    }
    
    // Weitere Methoden wie Add, Sub, etc. können implementiert werden.
    ```
    
- Verwendung: Positionierung von UI-Elementen, Fensterkoordinaten.

#### 1.1.2. `Size<T>` Struktur

Stellt die Dimensionen (Breite und Höhe) eines 2D-Objekts dar.

- Datei: `core/src/types/size.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Size<T, U> {
        pub w: T,
        pub h: T,
        _unit: std::marker::PhantomData<U>,
    }
    
    impl<T: Default, U> Default for Size<T, U> {
        fn default() -> Self {
            Size { w: Default::default(), h: Default::default(), _unit: std::marker::PhantomData }
        }
    }
    
    impl<T, U> Size<T, U> {
        pub const fn new(w: T, h: T) -> Self {
            Size { w, h, _unit: std.marker::PhantomData }
        }
    }
    ```
    
- Verwendung: Fenstergrößen, UI-Element-Dimensionen.

#### 1.1.3. `Rect<T>` und `RectInt` Strukturen

Stellt ein Rechteck im 2D-Raum dar, definiert durch Position und Größe.

- Datei: `core/src/types/rect.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Rect<T, U> {
        pub loc: Point<T, U>,
        pub size: Size<T, U>,
    }
    
    impl<T: Default, U> Default for Rect<T, U> {
        fn default() -> Self {
            Rect { loc: Default::default(), size: Default::default() }
        }
    }
    
    impl<T, U> Rect<T, U> {
        pub const fn new(loc: Point<T, U>, size: Size<T, U>) -> Self {
            Rect { loc, size }
        }
        // Methoden wie contains_point, intersection, etc.
    }
    
    pub type RectInt = Rect<i32, Logical>; // Beispiel für einen häufigen Typ
    ```
    
- Verwendung: Fenstergeometrien, Auswahlbereiche.

#### 1.1.4. `Color` Struktur

Repräsentiert eine Farbe im RGBA-Format.

- Datei: `core/src/types/color.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Color {
        pub r: f32, // Range 0.0 to 1.0
        pub g: f32, // Range 0.0 to 1.0
        pub b: f32, // Range 0.0 to 1.0
        pub a: f32, // Range 0.0 to 1.0 (alpha)
    }
    
    impl Color {
        pub const fn new(r: f32, g: f32, b: f32, a: f32) -> Self {
            Color { r, g, b, a }
        }
    
        pub fn to_hex_rgb(&self) -> String {
            format!(
                "#{:02x}{:02x}{:02x}",
                (self.r * 255.0) as u8,
                (self.g * 255.0) as u8,
                (self.b * 255.0) as u8
            )
        }
    
        pub fn to_hex_rgba(&self) -> String {
            format!(
                "#{:02x}{:02x}{:02x}{:02x}",
                (self.r * 255.0) as u8,
                (self.g * 255.0) as u8,
                (self.b * 255.0) as u8,
                (self.a * 255.0) as u8
            )
        }
        // Ggf. From-Implementierungen für gängige Formate (z.B. String "#RRGGBB")
    }
    ```
    
- Verwendung: Theming, UI-Elementfarben.

#### 1.1.5. `Orientation` Enum

Definiert die Ausrichtung von UI-Elementen oder Layouts.

- Datei: `core/src/types/orientation.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub enum Orientation {
        Horizontal,
        Vertical,
    }
    ```
    
- Verwendung: Layout-Manager, Ausrichtung von Bedienelementen.

#### 1.1.6. `uuid::Uuid` Integration

Universell eindeutige Identifikatoren.

- Datei: `core/src/types/mod.rs` (als Re-Export)
- Re-Export: `pub use uuid::Uuid;`
- Abhängigkeit in `core/Cargo.toml`: `uuid = { version = "1.0", features = ["v4", "serde"] }`
    - Die Features `v4` und `serde` sind essenziell. `v4` ermöglicht die Generierung zufälliger UUIDs, die für die eindeutige Identifizierung von Entitäten wie Fenstern, Workspaces oder Benachrichtigungen unerlässlich sind. `serde` wird benötigt, falls diese IDs in Konfigurationsdateien persistiert oder über IPC-Schnittstellen serialisiert werden müssen, was in einer Desktop-Umgebung sehr wahrscheinlich ist.1
- Verwendung: Eindeutige Identifizierung von Ressourcen systemweit.

#### 1.1.7. `chrono::DateTime<Utc>` Integration

Zeitstempel mit UTC-Zeitzone.

- Datei: `core/src/types/mod.rs` (als Re-Export)
- Re-Export: `pub use chrono::{DateTime, Utc};`
- Abhängigkeit in `core/Cargo.toml`: `chrono = { version = "0.4", features = ["serde"] }`
    - `chrono` ist der De-facto-Standard für Datums- und Zeitmanipulationen in Rust. Die Verwendung von `DateTime<Utc>` stellt zeitzonenagnostische Zeitstempel sicher, was für interne Repräsentationen als Best Practice gilt, um Komplexitäten bei der Zeitzonenbehandlung zu vermeiden. Das `serde`-Feature ist notwendig, wenn Zeitstempel persistiert (z.B. in Logdateien oder Konfigurationen) oder serialisiert werden sollen.3
- Verwendung: Zeitstempel für Logging, Events, "zuletzt geändert"-Informationen.

#### 1.1.8. Implementierungsschritte für `core::types`

1. Erstellen Sie das Verzeichnis `core/src/types/`.
2. Implementieren Sie `point.rs`, `size.rs`, `rect.rs`, `color.rs`, und `orientation.rs` gemäß den obigen Spezifikationen.
3. Fügen Sie `uuid` (mit Features "v4", "serde") und `chrono` (mit Feature "serde") zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
4. Fügen Sie die Re-Exports für `Uuid` und `DateTime<Utc>` in `core/src/types/mod.rs` hinzu. Alle lokalen Typen (Point, Size, etc.) sollten ebenfalls von `mod.rs` re-exportiert werden.
    
    Rust
    
    ```
    // core/src/types/mod.rs
    mod color;
    mod orientation;
    mod point;
    mod rect;
    mod size;
    
    pub use chrono::{DateTime, Utc};
    pub use color::Color;
    pub use orientation::Orientation;
    pub use point::{BufferCoords, Logical, Physical, Point};
    pub use rect::{Rect, RectInt};
    pub use size::Size;
    pub use uuid::Uuid;
    ```
    

### 1.2. Modul: `core::errors`

Dieses Modul stellt eine Basis-Fehlerbehandlungsstrategie und den `CoreError`-Typ bereit. Höhere Schichten können `CoreError` oder spezifischere Fehler aus Kernmodulen mittels `#[from]` oder `#[source]` in ihre eigenen Fehlertypen wrappen, wobei die Fehlerursachenkette (`source()`) beibehalten wird.

- **Verzeichnis:** `core/src/`
- Datei: `core/src/errors.rs`

#### 1.2.1. `CoreError` Enum

- Definition unter Verwendung von `thiserror`:
    
    Rust
    
    ```
    #
    pub enum CoreError {
        #[error("I/O error: {source}")]
        Io {
            #[from]
            source: std::io::Error,
            // Optional: context: String, // Um zusätzlichen Kontext zur IO-Operation zu geben
        },
    
        #[error("Configuration parsing error from '{path:?}': {message}")]
        ConfigParse {
            path: Option<std::path::PathBuf>,
            message: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #
        Serialization {
            message: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #
        Deserialization {
            message: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #[error("Invalid path: {path:?}")]
        InvalidPath { path: std::path::PathBuf },
    
        #[error("Initialization failed for module: {module_name}")]
        InitializationFailed {
            module_name: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #[error("Feature not implemented: {feature_name}")]
        NotImplemented { feature_name: String },
    
        #[error("Internal error: {message}")]
        Internal { message: String },
    
        #[error("Operation timed out: {operation}")]
        Timeout { operation: String },
    
        #[error("Permission denied for operation: {operation}")]
        PermissionDenied { operation: String },
    }
    ```
    
    Die Verwendung von `thiserror` 5 ist entscheidend für die Erstellung ergonomischer und informativer Fehlertypen. Das Attribut `#[from]` ermöglicht eine einfache Konvertierung von Standardbibliotheksfehlern wie `std::io::Error`. Das Attribut `#[source]` (implizit von `thiserror` verwendet, wenn ein Feld `#[from]` oder explizit `#[source]` ist) ist entscheidend für die Beibehaltung der Fehlerursachenkette, wie in der Spezifikation hervorgehoben. Eine robuste Fehlerbehandlungsstrategie ist fundamental. `thiserror` vereinfacht die Erstellung benutzerdefinierter Fehlertypen, die gut mit dem `Error`-Trait von Rust integriert sind. Die Beibehaltung der `source()`-Kette ist für das Debugging unerlässlich, da sie es Entwicklern ermöglicht, einen Fehler bis zu seinem Ursprung zurückzuverfolgen, selbst wenn er von verschiedenen Schichten mehrfach umgebrochen wird.

#### 1.2.2. Fehlerpropagierungsstrategie

Alle Funktionen in der Kernschicht, die fehlschlagen können, MÜSSEN `Result<T, CoreError>` oder `Result<T, SpecificCoreSubError>` zurückgeben, wobei `SpecificCoreSubError` via `#[from]` in `CoreError` konvertierbar sein sollte. Höhere Schichten definieren ihre eigenen Fehler-Enums (z.B. `DomainError`, `SystemError`, `UIError`) und verwenden `#[from] CoreError` oder `#[source] CoreError`, um Fehler aus dem Kern zu wrappen.

#### 1.2.3. Implementierungsschritte für `core::errors`

1. Fügen Sie `thiserror = "1.0"` zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
2. Implementieren Sie `core/src/errors.rs` mit dem `CoreError`-Enum wie oben spezifiziert.
3. Stellen Sie sicher, dass alle Module der Kernschicht `CoreError` für ihre Fehlerberichterstattung verwenden.

### 1.3. Modul: `core::logging`

Dieses Modul definiert die Logging-Infrastruktur basierend auf dem `tracing`-Crate. Alle höheren Schichten verwenden die `tracing`-Makros für ihre Logging-Ausgaben.

- **Verzeichnis:** `core/src/`
- Datei: `core/src/logging.rs`

#### 1.3.1. `initialize_logging()` Funktion

- Signatur:
    
    Rust
    
    ```
    pub fn initialize_logging(
        level_filter: tracing::Level,
        use_json: bool,
        custom_writer: Option<Box<dyn Fn() -> Box<dyn std::io::Write + Send + Sync + 'static> + Send + Sync + 'static>>
    ) -> Result<(), CoreError>
    ```
    
    - `level_filter`: Der minimale Log-Level, der aufgezeichnet werden soll (z.B. `tracing::Level::INFO`, `tracing::Level::DEBUG`).
    - `use_json`: Wenn `true`, werden Logs im JSON-Format ausgegeben, andernfalls in einem menschenlesbaren Format.
    - `custom_writer`: Eine optionale Funktion, die einen Writer für die Log-Ausgabe zurückgibt (z.B. in eine Datei, einen Netzwerk-Socket oder `std::io::stderr`). Standardmäßig wird `std::io::stderr` verwendet. Die Funktion wird verwendet, um den Writer bei Bedarf neu erstellen zu können (z.B. bei Log-Rotation).
- Implementierung:
    1. Verwendet `tracing_subscriber::fmt()` für die formatierte Protokollierung.
    2. Ermöglicht die Konfiguration des Formats (JSON oder "pretty") und des Writers.
        - Für JSON: `tracing_subscriber::fmt().json().with_writer(writer_closure).init();`
        - Für Pretty: `tracing_subscriber::fmt().pretty().with_writer(writer_closure).init();`
    3. Konfiguriert den `EnvFilter` basierend auf `level_filter` oder einer Umgebungsvariable (z.B. `RUST_LOG`).
    4. Richtet einen globalen Standard-Subscriber ein mittels `tracing::subscriber::set_global_default`.
    5. Gibt `Result<(), CoreError>` zurück, um Initialisierungsfehler zu signalisieren (z.B. wenn das Setzen des globalen Subscribers fehlschlägt).
- Die Standardisierung auf `tracing` ermöglicht eine strukturierte Protokollierung über alle Schichten hinweg, was für das Debugging und die Überwachung eines komplexen Systems wie einer Desktop-Umgebung von unschätzbarem Wert ist. Die Möglichkeit, zwischen menschenlesbaren und JSON-Formaten zu wechseln, ist gut für Entwicklung versus Produktion/Log-Aggregation. `tracing` bietet ein leistungsfähigeres und flexibleres Logging-Framework als das ältere `log`-Crate, insbesondere durch sein Konzept von Spans für kontextbezogene Protokollierung. Dies ist in einem mehrschichtigen, ereignisgesteuerten System von großem Vorteil.

#### 1.3.2. Verwendung von `tracing`-Makros

Alle Schichten verwenden die Makros `trace!`, `debug!`, `info!`, `warn!`, `error!` aus dem `tracing`-Crate. Spans (`span!`, `info_span!`, etc.) sollten verwendet werden, um kontextuelle Informationen für zusammengehörige Log-Meldungen bereitzustellen.

- Beispiel: `info!(target: "core::config", path = %file_path.display(), "Lade Kernkonfiguration.");`
- Beispiel mit Span:
    
    Rust
    
    ```
    let workspace_span = info_span!("workspace_operation", id = %workspace_id);
    let _enter = workspace_span.enter();
    //... Operationen innerhalb des Spans...
    info!("Operation abgeschlossen");
    ```
    

#### 1.3.3. Implementierungsschritte für `core::logging`

1. Fügen Sie `tracing = "0.1"` und `tracing_subscriber = { version = "0.3", features = ["fmt", "json", "env-filter"] }` zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
2. Implementieren Sie `core/src/logging.rs` mit der Funktion `initialize_logging()` wie oben spezifiziert.

### 1.4. Modul: `core::config`

Dieses Modul stellt Primitive zum Laden, Parsen und Zugreifen auf Kernkonfigurationen bereit. Es definiert die Struktur `CoreConfig` und Funktionen für deren Verwaltung.

- **Verzeichnis:** `core/src/config/`

#### 1.4.1. `CoreConfig` Struktur

Definiert die Kernkonfigurationseinstellungen.

- Datei: `core/src/config/types.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct CoreConfig {
        pub log_level: String, // z.B. "info", "debug", "trace"
        pub log_format: LogFormat,
        pub default_theme_name: Option<String>,
        pub enable_xwayland: bool,
        // Weitere kernspezifische Konfigurationen, z.B. Pfade, Feature-Flags
    }
    
    #
    #[serde(rename_all = "lowercase")]
    pub enum LogFormat {
        Json,
        Pretty,
    }
    
    impl Default for CoreConfig {
        fn default() -> Self {
            CoreConfig {
                log_level: "info".to_string(),
                log_format: LogFormat::Pretty,
                default_theme_name: Some("NovaDark".to_string()),
                enable_xwayland: true,
            }
        }
    }
    ```
    

#### 1.4.2. `ConfigError` Enum

Spezifische Fehler für das Laden und Verarbeiten von Konfigurationen.

- Datei: `core/src/config/error.rs`
- Definition:
    
    Rust
    
    ```
    use crate::core::errors::CoreError; // Assuming CoreError is in crate::core
    
    #
    pub enum ConfigError {
        #[error("Failed to determine config directory: {0}")]
        DirectoryResolutionError(String),
    
        #[error("Config file not found at {path:?}")]
        FileNotFound { path: std::path::PathBuf },
    
        #[error("Failed to read config file {path:?}")]
        FileReadError {
            path: std::path::PathBuf,
            #[source]
            source: std::io::Error,
        },
    
        #
        TomlParseError {
            path: std::path::PathBuf,
            message: String,
            #[source]
            source: Option<toml::de::Error>,
        },
        // Ggf. JsonParseError, falls JSON auch unterstützt wird
    
        #[error("Configuration not loaded or initialization failed.")]
        NotLoaded,
    
        #[error("Invalid configuration value for key '{key}': {message}")]
        InvalidValue { key: String, message: String },
    
        #[error("Failed to write default config to {path:?}")]
        DefaultConfigWriteError {
            path: std::path::PathBuf,
            #[source]
            source: std::io::Error,
        },
    
        #
        DefaultConfigTomlError(#[from] toml::ser::Error),
    
        #[error(transparent)]
        Core(#[from] CoreError), // Um CoreError wrappen zu können
    }
    ```
    

#### 1.4.3. Globale Konfigurationsvariable

Verwendet `once_cell` für eine thread-sichere, einmalig initialisierte globale Konfiguration.

- Datei: `core/src/config/mod.rs` (oder `core/src/config/global.rs`)
    
    Rust
    
    ```
    use once_cell::sync::OnceCell;
    use std::sync::RwLock;
    use super::types::CoreConfig; // Pfad anpassen
    use super::error::ConfigError; // Pfad anpassen
    
    static CORE_CONFIG: OnceCell<RwLock<CoreConfig>> = OnceCell::new();
    ```
    

#### 1.4.4. `load_core_config()` Funktion

Lädt die Kernkonfiguration aus einer Datei.

- Datei: `core/src/config/loader.rs`
- Signatur: `pub fn load_core_config(custom_path: Option<std::path::PathBuf>) -> Result<(), ConfigError>`
- Implementierung:
    
    1. Bestimme den Konfigurationsdateipfad:
        - Verwende `custom_path`, falls angegeben.
        - Andernfalls suche im XDG-Standardverzeichnis (z.B. `$XDG_CONFIG_HOME/nova-de/core.toml`) mittels `directories_next::ProjectDirs`.
        - Falls kein XDG-Verzeichnis gefunden wird, verwende einen Fallback-Pfad (z.B. `~/.config/nova-de/core.toml`).
    2. Wenn die Datei nicht existiert, erstelle eine Standardkonfiguration (`CoreConfig::default()`) und speichere sie unter dem ermittelten Pfad. Logge diesen Vorgang.
    3. Lese den Dateiinhalt. Bei Fehler `ConfigError::FileReadError`.
    4. Parse den Inhalt (z.B. TOML) in eine `CoreConfig`-Instanz. Bei Fehler `ConfigError::TomlParseError`.
    5. Initialisiere die globale `CORE_CONFIG` Variable mit der geladenen Konfiguration: `CORE_CONFIG.set(RwLock::new(loaded_config)).map_err(|_| ConfigError::NotLoaded)` (Fehlerbehandlung für den Fall, dass `set` fehlschlägt, was nur passiert, wenn bereits initialisiert).
    6. Gibt `Ok(())` bei Erfolg zurück.
    
    - Eine globale, statisch zugängliche Konfiguration ist üblich, erfordert aber eine sorgfältige Verwaltung, insbesondere bei der Initialisierung. `once_cell` für die Lazy-Initialisierung und `RwLock` für den threadsicheren Zugriff sind ein gutes Muster. Die Konfiguration muss früh im Anwendungslebenszyklus verfügbar sein und von verschiedenen Teilen des Systems zugänglich sein. `once_cell` stellt sicher, dass das Laden nur einmal erfolgt.

#### 1.4.5. `get_core_config()` Funktion

Bietet globalen Lesezugriff auf die geladene `CoreConfig`.

- Datei: `core/src/config/access.rs`
- Signatur: `pub fn get_core_config() -> Result<std::sync::RwLockReadGuard<'static, CoreConfig>, ConfigError>`
- Implementierung:
    1. Versuche, einen Read-Lock auf `CORE_CONFIG` zu erhalten: `CORE_CONFIG.get().ok_or(ConfigError::NotLoaded)?.read().map_err(|_| ConfigError::NotLoaded)` (Fehlerbehandlung für Poisoned Lock).
    2. Gibt den `RwLockReadGuard` bei Erfolg zurück.

#### 1.4.6. `update_core_config()` Funktion (Optional)

Ermöglicht die dynamische Aktualisierung der Kernkonfiguration zur Laufzeit.

- Datei: `core/src/config/access.rs`
- Signatur: `pub fn update_core_config(new_config: CoreConfig) -> Result<(), ConfigError>`
- Implementierung:
    1. Versuche, einen Write-Lock auf `CORE_CONFIG` zu erhalten.
    2. Aktualisiere die Konfiguration.
    3. Persistiert die neue Konfiguration in die Datei.
    4. **Wichtig:** Benachrichtige relevante Systemteile über die Konfigurationsänderung (z.B. über ein Event). Dies erfordert sorgfältige Überlegungen zur Thread-Sicherheit und wie Komponenten auf dynamische Änderungen reagieren.

#### 1.4.7. Implementierungsschritte für `core::config`

1. Fügen Sie `serde = { version = "1.0", features = ["derive"] }`, `toml = "0.8"`, `once_cell = "1.19"`, `directories-next = "2.0"` zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
2. Erstellen Sie die Verzeichnisstruktur `core/src/config/`.
3. Definieren Sie `CoreConfig` und `LogFormat` in `core/src/config/types.rs`.
4. Definieren Sie `ConfigError` in `core/src/config/error.rs`.
5. Implementieren Sie `loader.rs` (mit `load_core_config`) und `access.rs` (mit `get_core_config` und optional `update_core_config`).
6. Fügen Sie die globale `CORE_CONFIG`-Variable in `core/src/config/mod.rs` oder einer dedizierten `global.rs` ein und exportieren Sie die öffentlichen Funktionen.

### 1.5. Modul: `core::utils`

Dieses Modul bietet allgemeine Hilfsfunktionen, die von allen höheren Schichten genutzt werden können.

- **Verzeichnis:** `core/src/utils/`
- **Struktur:** Das Modul sollte in thematische Untermodule aufgeteilt werden, um die Übersichtlichkeit zu wahren.

#### 1.5.1. Untermodul: `core::utils::file_utils`

- Datei: `core/src/utils/file_utils.rs`
- Funktionen:
    - `pub fn read_file_to_string(path: &std::path::Path) -> Result<String, CoreError>`
        - Implementierung: Verwendet `std::fs::read_to_string`. Mappt `std::io::Error` auf `CoreError::Io`.
    - `pub fn ensure_dir_exists(path: &std::path::Path) -> Result<(), CoreError>`
        - Implementierung: Verwendet `std::fs::create_dir_all`. Mappt `std::io::Error` auf `CoreError::Io`.
    - `pub fn write_string_to_file(path: &std::path::Path, content: &str) -> Result<(), CoreError>`
        - Implementierung: Verwendet `std::fs::write`. Mappt `std::io::Error` auf `CoreError::Io`.

#### 1.5.2. Untermodul: `core::utils::string_utils`

- Datei: `core/src/utils/string_utils.rs`
- Funktionen:
    - `pub fn sanitize_filename(name: &str) -> String`
        - Implementierung: Ersetzt ungültige Zeichen für Dateinamen (z.B. `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`) durch Unterstriche oder entfernt sie.
    - `pub fn truncate_string_with_ellipsis(s: &str, max_len: usize) -> String`
        - Implementierung: Kürzt den String auf `max_len` Zeichen und fügt "..." hinzu, falls er gekürzt wurde. Achtet auf Grapheme Cluster, um Unicode-Zeichen nicht zu zerschneiden.
            
            Rust
            
            ```
            use unicode_segmentation::UnicodeSegmentation;
            //...
            // if s.graphemes(true).count() > max_len {
            //     s.graphemes(true).take(max_len - 3).collect::<String>() + "..."
            // } else {
            //     s.to_string()
            // }
            ```
            
            - Abhängigkeit: `unicode-segmentation = "1.10"`

#### 1.5.3. Untermodul: `core::utils::async_utils` (falls benötigt)

- Datei: `core/src/utils/async_utils.rs`
- Mögliche Funktionen:
    - Wrapper für `tokio::spawn` mit standardisierter Fehlerbehandlung oder Logging.
    - Hilfsfunktionen für Timeout-Management bei asynchronen Operationen.

#### 1.5.4. Implementierungsschritte für `core::utils`

1. Erstellen Sie das Verzeichnis `core/src/utils/`.
2. Erstellen Sie `core/src/utils/mod.rs` und deklarieren Sie die Untermodule:
    
    Rust
    
    ```
    pub mod file_utils;
    pub mod string_utils;
    // pub mod async_utils;
    ```
    
3. Implementieren Sie die Funktionen in den jeweiligen Untermodul-Dateien.
4. Fügen Sie ggf. Abhängigkeiten wie `unicode-segmentation` zu `core/Cargo.toml` hinzu.
5. Das `utils`-Modul kann leicht zu einer Ansammlung unzusammenhängender Funktionen werden. Es ist wichtig, die Organisation beizubehalten und sicherzustellen, dass die Funktionen eine breite Anwendbarkeit haben. Wenn eine Hilfsfunktion sehr spezifisch für eine einzelne Domäne oder ein Systemanliegen ist, sollte sie stattdessen im Hilfsmodul dieser Schicht angesiedelt sein. Eine klare Abgrenzung ist notwendig, um die Kohäsion der Module zu wahren und unnötige Kopplungen zu vermeiden.

---

## 2. Schicht: Domänenschicht (Domain Layer)

Die Domänenschicht kapselt die Geschäftslogik und den Kernzustand der Desktop-Umgebung NovaDE. Sie definiert die zentralen Konzepte, Regeln und Verhaltensweisen des Systems, unabhängig von technischen Implementierungsdetails der System- oder UI-Schicht. Die Kommunikation mit höheren Schichten (System- und UI-Schicht) erfolgt primär über wohldefinierte Service-APIs (als Rust-Traits implementiert) und durch domänenspezifische Events, die Zustandsänderungen signalisieren.

Die Domänenschicht ist zentral für die Funktionalität von NovaDE. Die Verwendung von Rust-Traits für Service-APIs fördert lose Kopplung und Testbarkeit. Ereignisse sind der Schlüssel zur reaktiven Propagierung von Zustandsänderungen. Fehlertypen sollten für jedes Domänenmodul spezifisch sein, um klaren Kontext zu bieten.

### 2.1. Service APIs (Traits)

Öffentliche Schnittstellen der Domänenschicht werden primär durch Rust-Traits definiert. Diese Traits werden von Service-Strukturen innerhalb der Domänenmodule implementiert. Methoden sind typischerweise `async`, wenn sie I/O-Operationen oder potenziell langlaufende Prozesse beinhalten, und geben `Result<T, SpecificDomainError>` zurück.

#### 2.1.1. `domain::theming::ThemingEngine` Trait

Verantwortlich für die Verwaltung und Anwendung von Themes.

- **Datei:** `domain/src/theming/engine.rs`

##### Zugehörige Datenstrukturen und Typen:

- **`ThemingConfiguration` Struct (Wertobjekt):**
    
    - Definiert die vom Benutzer oder System gewählten Theme-Einstellungen.
    - **Datei:** `domain/src/theming/config.rs`
    - **Felder:**
        - `icon_theme_name: String` (z.B. "Adwaita", "Papirus")
        - `gtk_theme_name: String` (z.B. "Adwaita-dark")
        - `cursor_theme_name: String` (z.B. "Adwaita")
        - `font_name: String` (z.B. "Cantarell 11")
        - `wallpaper_path: Option<std::path::PathBuf>`
        - `color_scheme: ColorScheme` (Enum: `Light`, `Dark`, `SystemPreference`)
        - `accent_color: core::types::Color`
    - **Invarianten:** `font_name` darf nicht leer sein, wenn gesetzt.
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`ColorScheme` Enum (Wertobjekt):**
    
    - Definiert die möglichen Farbschemata.
    - **Datei:** `domain/src/theming/config.rs` (oder `types.rs` im Theming-Modul)
    - **Varianten:** `Light`, `Dark`, `SystemPreference`
    - **Sichtbarkeit:** `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`AppliedThemeState` Struct (Wertobjekt):**
    
    - Repräsentiert den aktuell angewandten Theme-Zustand, inklusive aufgelöster Werte.
    - **Datei:** `domain/src/theming/state.rs`
    - **Felder:**
        - `current_config: ThemingConfiguration`
        - `resolved_wallpaper_path: Option<std::path::PathBuf>` (absoluter Pfad zum tatsächlich genutzten Hintergrundbild)
        - `font_config_details: String` (z.B. eine Pango-Font-Beschreibung oder Fontconfig-Pattern)
        - `effective_color_scheme: EffectiveColorScheme` (Enum: `Light`, `Dark`)
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`EffectiveColorScheme` Enum (Wertobjekt):**
    
    - Repräsentiert das tatsächlich aktive Farbschema (aufgelöst aus `SystemPreference`).
    - **Datei:** `domain/src/theming/state.rs`
    - **Varianten:** `Light`, `Dark`
    - **Sichtbarkeit:** `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`ThemingError` Enum (Fehlertyp):**
    
    - Spezifische Fehler für das Theming-Subsystem.
    - **Datei:** `domain/src/theming/error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum ThemingError {
            #[error("Invalid theming configuration: {0}")]
            InvalidConfiguration(String),
        
            #[error("Wallpaper not found at path: {0:?}")]
            WallpaperNotFound(std::path::PathBuf),
        
            #[error("Failed to apply theme '{theme_name}': {details}")]
            ThemeApplyFailed {
                theme_name: String,
                details: String,
                #[source]
                source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
            },
        
            #[error("Persistence error during theming operation")]
            PersistenceError(#[from] crate::core::errors::CoreError), // Assuming CoreError is accessible
        
            #[error("Underlying system service error for theming: {0}")]
            SystemServiceError(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn get_current_theme_state(&self) -> Result<AppliedThemeState, ThemingError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft den aktuell im System angewandten und zwischengespeicherten Theme-Zustand ab.
    - **Vorbedingungen:** Keine.
    - **Nachbedingungen:** Gibt den `AppliedThemeState` zurück oder einen `ThemingError`.
- `async fn update_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Aktualisiert die Theme-Konfiguration, persistiert sie und stößt die Anwendung des neuen Themes an.
    - **Vorbedingungen:** `new_config` muss valide sein (ggf. interne Validierung).
    - **Nachbedingungen:** Bei Erfolg wird das `ThemeChangedEvent` publiziert. Die Konfiguration wird persistiert (z.B. über `GlobalSettingsService` oder ein dediziertes Persistenz-Interface). Das System-Theme wird (versucht) anzuwenden.
    - **Geschäftsregeln:** Validiert die `new_config`. Löst das Hintergrundbild auf.
- `async fn apply_theme_to_system(&self, theme_state: &AppliedThemeState) -> Result<(), ThemingError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** (Potenziell intern, aufgerufen von `update_configuration`, oder als separate Methode für fortgeschrittene Nutzung) Wendet den gegebenen `AppliedThemeState` auf die Systemkomponenten an (z.B. Setzen von GTK-Theme, Icons, Cursor über Systemmechanismen).
    - **Vorbedingungen:** `theme_state` ist valide.
    - **Nachbedingungen:** Theme-Änderungen sind im System (versucht) aktiv.

##### Publizierte Events:

- **`ThemeChangedEvent`:**
    - **Datei:** `domain/src/theming/events.rs`
    - **Struktur:**
        
        Rust
        
        ```
        #
        pub struct ThemeChangedEvent {
            pub new_state: AppliedThemeState,
            pub old_state: Option<AppliedThemeState>, // Optional, falls der vorherige Zustand bekannt ist
        }
        ```
        
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::theming::ThemeChangedEvent`
    - **Typische Publisher:** Die Implementierung von `ThemingEngine`.
    - **Typische Subscriber:** `ui::theming_gtk` (UI-Schicht), `system::compositor` (Systemschicht, falls relevant für Darstellung).

#### 2.1.2. `domain::workspaces::WorkspaceManager` Trait

Verwaltet Workspaces (virtuelle Desktops).

- **Datei:** `domain/src/workspaces/manager.rs`

##### Zugehörige Datenstrukturen und Typen:

- **`WorkspaceId` Typalias:**
    
    - **Datei:** `domain/src/workspaces/types.rs` (oder direkt in `manager.rs`)
    - **Definition:** `pub type WorkspaceId = core::types::Uuid;`
- **`ApplicationIdentifier` Struct (Wertobjekt):**
    
    - Identifiziert eine Anwendung eindeutig.
    - **Datei:** `domain/src/workspaces/app_id.rs`
    - **Felder:**
        - `id_type: ApplicationIdType` (Enum: `DesktopFile`, `WaylandAppId`, `X11WindowClass`)
        - `identifier_str: String` (z.B. "firefox.desktop", "org.gnome.Nautilus", "Firefox")
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `serde::Serialize`, `serde::Deserialize`.
- **`Workspace` Struct (Entität):**
    
    - Repräsentiert einen einzelnen Workspace.
    - **Datei:** `domain/src/workspaces/workspace.rs`
    - **Felder:**
        - `id: WorkspaceId` (Sichtbarkeit: `pub(crate)`, Zugriff über Getter)
        - `name: String` (Sichtbarkeit: `pub`)
        - `output_name: Option<String>` (Name des Monitors, dem der Workspace primär zugeordnet ist, Sichtbarkeit: `pub`)
        - `layout_policy: WorkspaceLayoutPolicy` (Enum: `Tiling`, `Floating`, `Hybrid`, Sichtbarkeit: `pub`)
        - `pinned_applications: std::collections::HashSet<ApplicationIdentifier>` (Sichtbarkeit: `pub(crate)`, Zugriff über Methoden)
        - `creation_timestamp: core::types::DateTime<core::types::Utc>` (Sichtbarkeit: `pub(crate)`)
        - `last_accessed_timestamp: core::types::DateTime<core::types::Utc>` (Sichtbarkeit: `pub(crate)`)
    - **Initialwerte:** `id` wird bei Erstellung generiert, `name` z.B. "Workspace 1", `output_name` ist `None`, `layout_policy` wird aus globalen Einstellungen übernommen oder ist Standard (`Floating`), `pinned_applications` ist leer, `creation_timestamp` und `last_accessed_timestamp` sind Zeitpunkte der Erstellung.
    - **Invarianten:** `name` darf nicht leer sein. `id` ist unveränderlich nach Erstellung.
    - **Methoden (Beispiele):**
        - `pub fn new(name: String, layout_policy: WorkspaceLayoutPolicy) -> Self`
        - `pub fn id(&self) -> WorkspaceId`
        - `pub fn add_pinned_app(&mut self, app_id: ApplicationIdentifier) -> bool`
        - `pub fn remove_pinned_app(&mut self, app_id: &ApplicationIdentifier) -> bool`
        - `pub fn is_app_pinned(&self, app_id: &ApplicationIdentifier) -> bool`
        - `pub(crate) fn set_last_accessed(&mut self)`
    - **Beziehungen:** Keine direkten Entitätsbeziehungen in dieser Struktur, aber `pinned_applications` verweist auf `ApplicationIdentifier`.
- **`WorkspaceLayoutPolicy` Enum (Wertobjekt):**
    
    - **Datei:** `domain/src/workspaces/types.rs`
    - **Varianten:** `Tiling`, `Floating`, `Hybrid`
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`WorkspaceManagerError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/workspaces/error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum WorkspaceManagerError {
            #
            WorkspaceNotFound(super::types::WorkspaceId), // super::types::WorkspaceId
        
            #[error("Workspace with name '{0}' already exists")]
            DuplicateWorkspaceName(String),
        
            #[error("Maximum number of workspaces reached ({0})")]
            MaxWorkspacesReached(usize),
        
            #[error("Cannot delete the last workspace")]
            CannotDeleteLastWorkspace,
        
            #[error("Cannot delete active workspace; switch to another first")]
            CannotDeleteActiveWorkspace,
        
            #[error("Persistence error during workspace operation")]
            PersistenceError(#[from] crate::core::errors::CoreError),
        
            #[error("Invalid application identifier: {0}")]
            InvalidApplicationId(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn create_workspace(&self, name: Option<String>) -> Result<Workspace, WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Erstellt einen neuen Workspace. Wenn `name` `None` ist, wird ein Standardname generiert (z.B. "Workspace N").
    - **Nachbedingungen:** Bei Erfolg wird `WorkspaceEvent::Created` publiziert. Der neue Workspace wird persistiert.
    - **Geschäftsregeln:** Prüft auf Namenskonflikte und maximale Anzahl von Workspaces.
- `async fn delete_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Löscht den Workspace mit der gegebenen ID.
    - **Vorbedingungen:** Workspace darf nicht der letzte verbleibende sein. Workspace darf nicht der aktive sein (oder es muss eine Logik zum Wechsel des aktiven Workspaces implementiert sein).
    - **Nachbedingungen:** Bei Erfolg wird `WorkspaceEvent::Deleted` publiziert. Änderungen werden persistiert.
- `async fn get_workspace_by_id(&self, id: WorkspaceId) -> Result<Option<Workspace>, WorkspaceManagerError>`
    - **`noexcept`**: `false`
- `async fn get_all_workspaces(&self) -> Result<Vec<Workspace>, WorkspaceManagerError>`
    - **`noexcept`**: `false`
- `async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Vorbedingungen:** Der Workspace mit `id` muss existieren.
    - **Nachbedingungen:** `WorkspaceEvent::ActiveChanged` wird publiziert. Der `last_accessed_timestamp` des neuen und ggf. des alten aktiven Workspaces wird aktualisiert.
- `async fn get_active_workspace_id(&self) -> Result<Option<WorkspaceId>, WorkspaceManagerError>`
    - **`noexcept`**: `false`
- `async fn rename_workspace(&self, id: WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Vorbedingungen:** `new_name` darf nicht leer sein und nicht mit einem existierenden Workspace-Namen (außer dem eigenen) kollidieren.
    - **Nachbedingungen:** `WorkspaceEvent::Renamed` wird publiziert. Änderung wird persistiert.
- `async fn pin_application_to_space(&self, space_id: WorkspaceId, app_id: ApplicationIdentifier) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Nachbedingungen:** `WorkspaceEvent::AppPinned` wird publiziert. Änderung wird persistiert.
- `async fn unpin_application_from_space(&self, space_id: WorkspaceId, app_id: ApplicationIdentifier) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Nachbedingungen:** `WorkspaceEvent::AppUnpinned` wird publiziert. Änderung wird persistiert.
- `async fn move_window_to_workspace(&self, window_identifier: String /* Eindeutiger Fenster-Identifikator, z.B. Wayland Surface ID als String */, target_workspace_id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Fordert an, ein Fenster zum Ziel-Workspace zu verschieben. Die eigentliche Fensterverschiebung erfolgt in der Systemschicht.
    - **Nachbedingungen:** `WorkspaceEvent::WindowMoved` wird publiziert.

##### Publizierte Events:

- **`WorkspaceEvent` Enum:**
    - **Datei:** `domain/src/workspaces/events.rs`
    - **Varianten (Beispiele):**
        - `Created { workspace: Workspace }`
        - `Deleted { workspace_id: WorkspaceId, new_active_workspace_id: Option<WorkspaceId> }`
        - `ActiveChanged { old_id: Option<WorkspaceId>, new_id: WorkspaceId }`
        - `Renamed { workspace_id: WorkspaceId, new_name: String }`
        - `AppPinned { space_id: WorkspaceId, app_id: ApplicationIdentifier }`
        - `AppUnpinned { space_id: WorkspaceId, app_id: ApplicationIdentifier }`
        - `LayoutPolicyChanged { workspace_id: WorkspaceId, new_policy: WorkspaceLayoutPolicy }`
        - `WindowMoved { window_identifier: String, source_workspace_id: Option<WorkspaceId>, target_workspace_id: WorkspaceId }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::workspaces::WorkspaceEvent`
    - **Typische Publisher:** Die Implementierung von `WorkspaceManager`.
    - **Typische Subscriber:** `ui::shell` (UI-Schicht), `system::compositor` (Systemschicht).

#### 2.1.3. `domain::user_centric_services::AIInteractionLogicService` Trait

Steuert die Logik für Benutzerinteraktionen mit KI-Diensten, inklusive Einwilligungsmanagement.

- **Datei:** `domain/src/user_centric_services/ai_logic.rs`
- Diese Service-Schnittstelle ist entscheidend für die sichere und transparente Integration von KI-Funktionen. Die Trennung der Einwilligungslogik von der reinen MCP-Kommunikation (die in der Systemschicht angesiedelt ist) ist ein wichtiger Aspekt für Datenschutz und Benutzerkontrolle.

##### Zugehörige Datenstrukturen und Typen:

- **`AIInteractionContextData` Struct (Wertobjekt):**
    
    - Sammelt Kontextinformationen für eine KI-Anfrage.
    - **Datei:** `domain/src/user_centric_services/ai_context.rs`
    - **Felder:**
        - `active_window_title: Option<String>`
        - `active_app_id: Option<super::workspaces::app_id::ApplicationIdentifier>` // Pfad anpassen
        - `current_workspace_id: Option<super::workspaces::types::WorkspaceId>` // Pfad anpassen
        - `selected_text: Option<String>`
        - `clipboard_content_preview: Option<String>` (Vorschau, nicht der volle Inhalt)
        - `screenshot_data_uri: Option<String>` (Data-URI eines Screenshots, falls relevant)
        - `user_query: String` (Die eigentliche Anfrage des Benutzers)
        - `interaction_id: core::types::Uuid` (Eindeutige ID für diese Interaktion)
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`AIConsent` Enum (Wertobjekt):**
    
    - Repräsentiert den Einwilligungsstatus des Benutzers für KI-Interaktionen.
    - **Datei:** `domain/src/user_centric_services/ai_consent.rs`
    - **Varianten:**
        - `Granted { for_session: bool, timestamp: core::types::DateTime<core::types::Utc> }` (Einwilligung für die aktuelle Sitzung oder dauerhaft)
        - `Denied { timestamp: core::types::DateTime<core::types::Utc> }`
        - `NotSet`
    - **Sichtbarkeit:** `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`AIInteractionResponse` Struct (Wertobjekt):**
    
    - Repräsentiert die Antwort von einem KI-Dienst.
    - **Datei:** `domain/src/user_centric_services/ai_response.rs`
    - **Felder:**
        - `interaction_id: core::types::Uuid`
        - `response_text: Option<String>`
        - `suggested_actions: Vec<String>`
        - `tool_call_id: Option<String>` (Falls ein Tool vom MCP-Server aufgerufen wurde)
        - `tool_call_result: Option<serde_json::Value>` (Ergebnis des Tool-Aufrufs)
        - `error_message: Option<String>`
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`AIInteractionError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/user_centric_services/ai_error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum AIInteractionError {
            #[error("User consent not given for AI interaction")]
            ConsentNotGiven,
        
            #[error("MCP client error: {0}")]
            MCPError(String), // Wird von SystemMcpService kommen
        
            #[error("Failed to prepare context for AI interaction: {0}")]
            ContextPreparationFailed(String),
        
            #[error("AI service returned an error: {0}")]
            AIServiceError(String),
        
            #[error("Persistence error during AI consent operation")]
            PersistenceError(#[from] crate::core::errors::CoreError),
        
            #
            InteractionNotFoundOrTimedOut(crate::core::types::Uuid),
        }
        ```
        

##### Methoden des Traits:

- `async fn initiate_interaction(&self, context_data: AIInteractionContextData) -> Result<core::types::Uuid /* interaction_id */, AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Startet eine neue KI-Interaktion. Prüft die Einwilligung und leitet die Anfrage ggf. an den `SystemMcpService` weiter. Gibt eine `interaction_id` zurück, um die Antwort später zu korrelieren.
    - **Vorbedingungen:** `get_consent_status()` darf nicht `Denied` sein. Wenn `NotSet`, wird die Interaktion ggf. pausiert und ein `AIConsentRequestEvent` ausgelöst.
    - **Nachbedingungen:** `AIInteractionStartedEvent` wird publiziert.
    - **Geschäftsregeln:** Die Logik hier muss entscheiden, ob basierend auf `context_data` und `AIConsent` die Interaktion fortgesetzt, abgelehnt oder eine erneute Einwilligung angefordert wird.
- `async fn provide_consent(&self, consent: AIConsent) -> Result<(), AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Speichert die Benutzereinwilligung.
    - **Nachbedingungen:** `AIConsentChangedEvent` wird publiziert. Der Einwilligungsstatus wird persistiert.
- `async fn get_consent_status(&self) -> Result<AIConsent, AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft den aktuellen Einwilligungsstatus ab.
- `async fn process_mcp_response(&self, interaction_id: core::types::Uuid, response: Result<serde_json::Value, String> /* MCP Tool Call Result oder Fehler */) -> Result<(), AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Verarbeitet eine asynchrone Antwort vom `SystemMcpService` (typischerweise ein Tool-Aufruf-Ergebnis).
    - **Nachbedingungen:** `AIInteractionCompletedEvent` oder `AIInteractionFailedEvent` wird publiziert.

##### Publizierte Events:

- **`AIConsentChangedEvent`:**
    - **Datei:** `domain/src/user_centric_services/events.rs`
    - **Struktur:** `pub struct AIConsentChangedEvent { pub new_status: AIConsent; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIConsentChangedEvent`
- **`AIInteractionStartedEvent`:**
    - **Struktur:** `pub struct AIInteractionStartedEvent { pub interaction_id: core::types::Uuid, pub context: AIInteractionContextData; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIInteractionStartedEvent`
- **`AIInteractionCompletedEvent`:**
    - **Struktur:** `pub struct AIInteractionCompletedEvent { pub response: AIInteractionResponse; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIInteractionCompletedEvent`
- **`AIInteractionFailedEvent`:**
    - **Struktur:** `pub struct AIInteractionFailedEvent { pub interaction_id: core::types::Uuid, pub error: AIInteractionError; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIInteractionFailedEvent`
- **`AIConsentRequestEvent`:** (Falls die UI den Benutzer zur Einwilligung auffordern soll)
    - **Struktur:** `pub struct AIConsentRequestEvent { pub interaction_id: core::types::Uuid, pub for_context: AIInteractionContextData }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIConsentRequestEvent`
- **Typische Publisher:** Die Implementierung von `AIInteractionLogicService`.
- **Typische Subscriber:** `ui::command_palette` oder KI-Widgets (UI-Schicht), `system::mcp::SystemMcpService` (Systemschicht, für Antworten).

#### 2.1.4. `domain::user_centric_services::NotificationService` Trait

Verwaltet das Anzeigen und Interagieren mit Desktop-Benachrichtigungen.

- **Datei:** `domain/src/user_centric_services/notification_service.rs`
- Die Parameter für `post_notification` spiegeln eng die D-Bus-Methode `org.freedesktop.Notifications.Notify` wider.9 Dieser Dienst fungiert als Abstraktion über potenzielle D-Bus-Implementierungen in der Systemschicht. Die Domänenschicht definiert das _Konzept_ einer Benachrichtigung innerhalb von NovaDE. Die Systemschicht wäre dann dafür verantwortlich, diese Benachrichtigung tatsächlich über D-Bus zu senden, wenn NovaDE als Benachrichtigungsserver fungiert, oder auf D-Bus-Benachrichtigungen zu lauschen, wenn es diese konsumiert. Die Verwendung von `zvariant::OwnedValue` für `hints` deutet auf eine direkte Abbildung auf D-Bus-Varianten hin.

##### Zugehörige Datenstrukturen und Typen:

- **`NotificationId` Typalias:**
    
    - **Datei:** `domain/src/user_centric_services/types.rs` (oder direkt in `notification_service.rs`)
    - **Definition:** `pub type NotificationId = core::types::Uuid;`
- **`NotificationAction` Struct (Wertobjekt):**
    
    - Definiert eine Aktion, die mit einer Benachrichtigung verbunden ist.
    - **Datei:** `domain/src/user_centric_services/notification_action.rs`
    - **Felder:**
        - `key: String` (Eindeutiger Schlüssel für die Aktion, z.B. "default", "reply")
        - `label: String` (Anzeigetext für den Button)
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `serde::Serialize`, `serde::Deserialize`.
- **`NotificationUrgency` Enum (Wertobjekt):**
    
    - Definiert die Dringlichkeitsstufe einer Benachrichtigung.
    - **Datei:** `domain/src/user_centric_services/types.rs`
    - **Varianten:** `Low`, `Normal`, `Critical`
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`Notification` Struct (Entität):**
    
    - Repräsentiert eine einzelne Desktop-Benachrichtigung.
    - **Datei:** `domain/src/user_centric_services/notification.rs`
    - **Felder:**
        - `id: NotificationId` (Sichtbarkeit: `pub(crate)`, Zugriff über Getter)
        - `app_name: String` (Sichtbarkeit: `pub`)
        - `replaces_id: Option<NotificationId>` (Sichtbarkeit: `pub`)
        - `app_icon: Option<String>` (Pfad oder Name des Icons, Sichtbarkeit: `pub`)
        - `summary: String` (Kurze Zusammenfassung, Sichtbarkeit: `pub`)
        - `body: Option<String>` (Detaillierter Text, Sichtbarkeit: `pub`)
        - `actions: Vec<NotificationAction>` (Sichtbarkeit: `pub`)
        - `hints: std::collections::HashMap<String, zbus::zvariant::OwnedValue>` (Spezifische Hinweise für den Notification-Server, Sichtbarkeit: `pub`)
        - `expire_timeout_ms: i32` (-1 für Server-Default, 0 für nie, >0 für Millisekunden, Sichtbarkeit: `pub`)
        - `urgency: NotificationUrgency` (Sichtbarkeit: `pub`)
        - `timestamp: core::types::DateTime<core::types::Utc>` (Sichtbarkeit: `pub(crate)`)
        - `is_displayed: std::sync::atomic::AtomicBool` (Interner Status, ob die Benachrichtigung aktuell angezeigt wird)
    - **Initialwerte:** `id` wird generiert, `timestamp` ist aktuell, `is_displayed` ist `false`.
    - **Invarianten:** `summary` darf nicht leer sein. `id` ist unveränderlich.
    - **Methoden (Beispiele):**
        - `pub fn id(&self) -> NotificationId`
        - `pub(crate) fn mark_as_displayed(&self)`
        - `pub(crate) fn mark_as_closed(&self)`
- **`CloseReason` Enum (Wertobjekt):**
    
    - Gibt den Grund an, warum eine Benachrichtigung geschlossen wurde.
    - **Datei:** `domain/src/user_centric_services/types.rs`
    - **Varianten:** `Expired`, `DismissedByUser`, `ClosedByApp`, `ServiceShutdown`, `Unknown`
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`NotificationError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/user_centric_services/notification_error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum NotificationError {
            #
            NotificationNotFound(super::types::NotificationId), // Pfad anpassen
        
            #[error("Invalid notification format or content: {0}")]
            InvalidNotificationFormat(String),
        
            #[error("Failed to invoke action '{action_key}' for notification {notification_id}: {reason}")]
            ActionInvokeFailed {
                notification_id: super::types::NotificationId, // Pfad anpassen
                action_key: String,
                reason: String,
            },
        
            #[error("Notification service is unavailable or failed: {0}")]
            ServiceUnavailable(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn post_notification(&self, app_name: String, replaces_id: Option<NotificationId>, app_icon: Option<String>, summary: String, body: Option<String>, actions: Vec<NotificationAction>, hints: std::collections::HashMap<String, zbus::zvariant::OwnedValue>, expire_timeout_ms: i32, urgency: NotificationUrgency) -> Result<NotificationId, NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Sendet eine neue Benachrichtigung oder aktualisiert eine existierende.
    - **Nachbedingungen:** Bei Erfolg wird `NotificationPostedEvent` publiziert. Die Benachrichtigung wird (versucht) dem Benutzer angezeigt.
- `async fn get_active_notifications(&self) -> Result<Vec<Notification>, NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft eine Liste aller aktuell aktiven (angezeigten) Benachrichtigungen ab.
- `async fn close_notification(&self, id: NotificationId, reason: CloseReason) -> Result<(), NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Schließt eine spezifische Benachrichtigung.
    - **Vorbedingungen:** Benachrichtigung mit `id` muss existieren und aktiv sein.
    - **Nachbedingungen:** `NotificationClosedEvent` wird publiziert.
- `async fn invoke_action(&self, id: NotificationId, action_key: String) -> Result<(), NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Löst eine mit einer Benachrichtigung verbundene Aktion aus.
    - **Vorbedingungen:** Benachrichtigung mit `id` und Aktion mit `action_key` müssen existieren.
    - **Nachbedingungen:** `NotificationActionInvokedEvent` wird publiziert.

##### Publizierte Events:

- **`NotificationPostedEvent`:**
    - **Datei:** `domain/src/user_centric_services/events.rs`
    - **Struktur:** `pub struct NotificationPostedEvent { pub notification: Notification; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::NotificationPostedEvent`
- **`NotificationClosedEvent`:**
    - **Struktur:** `pub struct NotificationClosedEvent { pub id: NotificationId, pub reason: CloseReason; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::NotificationClosedEvent`
- **`NotificationActionInvokedEvent`:**
    - **Struktur:** `pub struct NotificationActionInvokedEvent { pub id: NotificationId, pub action_key: String; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::NotificationActionInvokedEvent`
- **Typische Publisher:** Die Implementierung von `NotificationService`.
- **Typische Subscriber:** `ui::notifications_frontend` oder `ui::shell` (UI-Schicht), `system::dbus::notification_server` (Systemschicht, falls NovaDE als Server agiert und die Events intern weiterleitet).

#### 2.1.5. `domain::global_settings_and_state_management::GlobalSettingsService` Trait

Verwaltet globale Desktop-Einstellungen und deren Persistenz.

- **Datei:** `domain/src/global_settings/service.rs`
- Dieser Dienst bietet eine strukturierte Methode zur Verwaltung einer Vielzahl von Desktop-Einstellungen. Der Persistenzmechanismus (z.B. eine Implementierung von `domain::settings_persistence_iface`, wie in der Kernschichtspezifikation erwähnt) ist hier von entscheidender Bedeutung.

##### Zugehörige Datenstrukturen und Typen:

- **`SettingPath` Typalias:**
    
    - **Datei:** `domain/src/global_settings/types.rs`
    - **Definition:** `pub type SettingPath = String;` (z.B. "desktop.background.color", "keyboard.layouts.0.name")
- **`SettingValue` Enum (Wertobjekt):**
    
    - Repräsentiert den Wert einer Einstellung. Kann verschiedene primitive Typen und verschachtelte Strukturen annehmen.
    - **Datei:** `domain/src/global_settings/value.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        #[serde(untagged)] // Für flexible (De-)Serialisierung
        pub enum SettingValue {
            String(String),
            Integer(i64),
            Float(f64),
            Boolean(bool),
            List(Vec<SettingValue>),
            Map(std::collections::HashMap<String, SettingValue>),
            Null, // Um das Fehlen eines Wertes explizit darzustellen
        }
        // Implement From<T> for SettingValue für gängige Typen
        ```
        
    - Die Verwendung von `serde_json::Value` wäre eine Alternative für maximale Flexibilität, aber ein eigenes Enum bietet mehr Typsicherheit innerhalb der Domäne.
- **`GlobalDesktopSettings` Struct (Entität):**
    
    - Repräsentiert die Gesamtheit aller verwalteten Einstellungen.
    - **Datei:** `domain/src/global_settings/settings_struct.rs`
    - **Felder:**
        - `settings: std::collections::BTreeMap<SettingPath, SettingValue>` (BTreeMap für geordnete Iteration/Serialisierung, falls gewünscht)
    - **Sichtbarkeit:** `pub(crate)` für `settings`, Zugriff über Methoden.
    - **Methoden (Beispiele):**
        - `pub fn get(&self, path: &SettingPath) -> Option<&SettingValue>`
        - `pub fn set(&mut self, path: SettingPath, value: SettingValue) -> Option<SettingValue>` (gibt alten Wert zurück)
        - `pub fn remove(&mut self, path: &SettingPath) -> Option<SettingValue>`
- **`GlobalSettingsError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/global_settings/error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum GlobalSettingsError {
            #
            SettingNotFound(super::types::SettingPath), // Pfad anpassen
        
            #
            TypeMismatch {
                path: super::types::SettingPath, // Pfad anpassen
                expected_type: String,
                found_type: String,
            },
        
            #[error("Persistence error during settings operation")]
            PersistenceError(#[from] crate::core::errors::CoreError),
        
            #[error("Validation error for setting '{path}': {message}")]
            ValidationError {
                path: super::types::SettingPath, // Pfad anpassen
                message: String,
            },
        
            #[error("Invalid setting path format: {0}")]
            InvalidPathFormat(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn load_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Lädt alle Einstellungen aus der Persistenzschicht.
    - **Nachbedingungen:** Gibt die geladenen `GlobalDesktopSettings` zurück oder einen Fehler.
- `async fn save_settings(&self, settings: &GlobalDesktopSettings) -> Result<(), GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Persistiert den gesamten Einstellungsbaum.
- `async fn get_setting(&self, path: &SettingPath) -> Result<Option<SettingValue>, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft den Wert einer spezifischen Einstellung ab.
- `async fn update_setting(&self, path: SettingPath, value: SettingValue) -> Result<Option<SettingValue /* old_value */>, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Aktualisiert eine spezifische Einstellung und persistiert die Änderung. Gibt den alten Wert zurück, falls vorhanden.
    - **Vorbedingungen:** `path` und `value` müssen validen Formaten entsprechen. Ggf. Validierung gegen ein Schema.
    - **Nachbedingungen:** Bei Erfolg wird `SettingChangedEvent` publiziert. Die Einstellung wird persistiert.
- `async fn get_all_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft eine Kopie aller aktuellen Einstellungen ab.

##### Publizierte Events:

- **`SettingChangedEvent`:**
    - **Datei:** `domain/src/global_settings/events.rs`
    - **Struktur:**
        
        Rust
        
        ```
        #
        pub struct SettingChangedEvent {
            pub path: SettingPath,
            pub old_value: Option<SettingValue>,
            pub new_value: SettingValue,
        }
        ```
        
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::global_settings::SettingChangedEvent`
- **Typische Publisher:** Die Implementierung von `GlobalSettingsService`.
- **Typische Subscriber:** `ui::control_center` (UI-Schicht), verschiedene Systemkomponenten, die auf Einstellungsänderungen reagieren müssen.

### 2.2. Datenstrukturen (Öffentlich)

Zusätzlich zu den direkt mit Service-APIs assoziierten Strukturen können weitere öffentliche Datenstrukturen existieren, die Zustände oder Konfigurationen repräsentieren und von mehreren Modulen oder Schichten verwendet werden.

- **Beispiele:** Viele der oben bereits definierten Strukturen fallen in diese Kategorie, z.B. `AppliedThemeState`, `Workspace`, `Notification`, `GlobalDesktopSettings`, `ThemingConfiguration`, `ApplicationIdentifier`, `AIInteractionContextData`, `AIConsent`, `SettingValue`.
- **Implementierungsrichtlinien:**
    - Jede Struktur/Enum in einer eigenen Datei innerhalb des relevanten Domänenmoduls (z.B. `domain/src/theming/state.rs`).
    - Ableitung von `Debug`, `Clone`, `PartialEq` ist Standard.
    - Ableitung von `serde::Serialize`, `serde::Deserialize` falls Persistenz oder IPC-Übertragung erforderlich ist.
    - Felder sollten `pub` sein, wenn sie direkt von extern gelesen werden sollen, oder `pub(crate)` mit öffentlichen Gettern/Settern für kontrollierten Zugriff.
    - Invarianten sollten durch Konstruktoren und Methoden sichergestellt werden.
    - Beziehungen zu anderen Entitäten/Wertobjekten sollten klar definiert sein (z.B. durch Komposition oder IDs).

### 2.3. Events (Domänenspezifisch)

Domänenspezifische Events signalisieren wichtige Zustandsänderungen innerhalb der Domänenschicht. Sie sind entscheidend für eine reaktive Architektur und lose Kopplung.

- **Beispiele:** Die bereits bei den Services definierten Events (`ThemeChangedEvent`, `WorkspaceEvent`, `NotificationPostedEvent`, `SettingChangedEvent`, `AIConsentChangedEvent`, etc.).
- **Struktur:**
    - Jedes Event sollte als eigene Struktur oder als Variante eines umfassenderen Event-Enums (pro Modul) definiert werden.
    - Die Event-Struktur enthält alle relevanten Daten zur Zustandsänderung.
    - Beispiel: `pub struct ActiveWorkspaceChanged { pub old_id: Option<WorkspaceId>, pub new_id: WorkspaceId }`
- **Eindeutiger Event-Name/Typ:** Eine klare Namenskonvention ist wichtig, z.B. `nova_de::domain::module::EventName`.
- **Publisher:** Die jeweilige Service-Implementierung, die die Zustandsänderung verursacht.
- **Subscriber:** Komponenten in der System- und UI-Schicht, die auf diese Änderungen reagieren müssen.
    - Beispiel: `ui::theming_gtk` abonniert `ThemeChangedEvent`.
    - Beispiel: `system::compositor` abonniert `ActiveWorkspaceChanged`.
- **Datei-Struktur:** Events können in einer `events.rs`-Datei innerhalb jedes Domänenmoduls definiert werden (z.B. `domain/src/theming/events.rs`).
- **Serialisierung:** Wenn Events über IPC-Grenzen (z.B. D-Bus) gesendet werden sollen, müssen sie `serde::Serialize` und `serde::Deserialize` implementieren.

### 2.4. Fehlertypen (Modulspezifisch)

Jedes Hauptmodul innerhalb der Domänenschicht sollte seine eigenen spezifischen Fehler-Enums definieren, um präzise Fehlerinformationen zu liefern.

- **Beispiele:** Die bereits bei den Services definierten Fehler (`ThemingError`, `WorkspaceManagerError`, `AIInteractionError`, `NotificationError`, `GlobalSettingsError`).
- **Implementierungsrichtlinien:**
    - Verwendung des `thiserror`-Crates für eine ergonomische Definition.
    - Jeder Fehlertyp in einer eigenen Datei (z.B. `domain/src/theming/error.rs`).
    - Fehlervarianten sollten klare und spezifische Fehlerzustände des Moduls beschreiben.
    - Wo sinnvoll, sollten Quellfehler (`#[source]`) oder Konvertierungen (`#[from]`) für Fehler aus der Kernschicht oder anderen Abhängigkeiten bereitgestellt werden, um die Fehlerkette zu erhalten.
- **Verwendung:** Domänenservices geben diese spezifischen Fehler in ihren `Result`-Typen zurück. Die System- und UI-Schicht behandeln diese Fehler oder leiten sie ggf. weiter, eventuell gewrappt in ihre eigenen, allgemeineren Fehlertypen.

---

## 3. Schicht: Systemschicht (System Layer)

Die Systemschicht stellt der UI-Schicht systemnahe Dienste und Ereignisse zur Verfügung und setzt deren Befehle technisch um. Sie interagiert direkt mit dem Betriebssystem, der Hardware (über Abstraktionen wie DRM/GBM, libinput), Systemdiensten (via D-Bus, PipeWire) und dem Wayland-Compositor-Backend (Smithay). Sie konsumiert APIs der Domänenschicht, um Geschäftslogik anzuwenden und Zustände abzufragen.

Diese Schicht ist die Brücke zwischen der abstrakten Domänenlogik von NovaDE und den konkreten Realitäten des Betriebssystems und der Hardware. Hier finden viele der FFI-Aufrufe, IPC-Kommunikation und Interaktionen auf niedriger Ebene statt. Smithay wird intensiv für Wayland-bezogene Komponenten genutzt. Eine robuste Fehlerbehandlung ist entscheidend, um Fehler von niedriger Ebene in abstraktere `SystemError`-Typen oder spezifische Unterfehler zu übersetzen. Die asynchrone Natur vieler Operationen (IPC, Hardwareinteraktion) ist hier kritisch.

### 3.1. Modul: `system::compositor` (Smithay-basierter Wayland Compositor Kern)

Dieses Modul ist das Herzstück der grafischen Darstellung und Fensterverwaltung in NovaDE. Es basiert auf der Smithay-Bibliothek und implementiert die Kernfunktionalitäten eines Wayland-Compositors.

- **Abhängigkeiten:** `smithay`, `wayland-server`, `wayland-protocols`, `calloop`, `input` (libinput-rs), `drm-rs`, `gbm-rs`, `libloading` (für EGL).
- **Verzeichnis:** `system/src/compositor/`

#### 3.1.1. `NovaDeSystemState` Struktur (Hauptzustand für Smithay Handler)

Diese Struktur dient als zentraler Datenhalter für den Event-Loop und implementiert die verschiedenen Handler-Traits von Smithay.

- **Datei:** `system/src/state.rs` (oder `system/src/compositor/state.rs`, je nach Gesamtstruktur)
- **Felder (Beispiele, basierend auf typischer Smithay-Nutzung und NovaDE-Anforderungen):**
    
    Rust
    
    ```
    use smithay::{
        backend::{
            allocator::dmabuf::DmabufState, // [50]
            drm::DrmDeviceFd, // [19, 24, 56, 62, 69, 93, 305, 388]
            egl::{EGLContext, EGLDisplay}, // [16, 26, 62, 65, 94, 97]
            renderer::{
                gles::GlesRenderer, // [16, 40, 62, 94, 97]
                multigpu::{gbm::GbmGlesBackend, GpuManager}, // [24, 28, 40, 62, 97, 145, 222]
            },
            libinput::LibinputInputBackend, // [23, 75, 382]
            udev::{UdevBackend, UdevEvent}, // [37]
            session::{Session, Signal as SessionSignal, logind::LogindSession}, // [362, 391]
            input::Seat,
        },
        desktop::{Space, Window, PopupManager, layer_map_for_output, LayerSurface}, // [67]
        input::{SeatState, SeatHandler, pointer::PointerHandle, keyboard::KeyboardHandle, touch::TouchHandle, SeatName},
        output::Output,
        reexports::{
            calloop::{EventLoop, LoopHandle, LoopSignal},
            wayland_server::{Display, DisplayHandle, Client,backend::GlobalId},
            wayland_protocols::xdg::shell::server::xdg_wm_base,
        },
        utils::{Clock, Logical, Point, Rectangle, Serial, Transform, SERIAL_COUNTER},
        wayland::{
            buffer::BufferHandler, // [353]
            compositor::{CompositorState, CompositorHandler, CompositorClientState, SurfaceData}, // [17, 87, 99]
            dmabuf::DmabufHandler, // [50]
            output::{OutputHandler, OutputManagerState}, // [49, 147, 280, 370, 372, 392]
            presentation::PresentationState, // [274]
            seat::WaylandSeatData,
            shell::{
                xdg::{XdgShellState, XdgShellHandler, XdgShellSurfaceUserData, XdgWmBaseUserData, decoration::XdgDecorationState}, // [13, 31, 54, 81, 167, 176, 191, 195, 199, 204, 207, 208, 215, 223, 224, 225, 227, 317, 323, 336, 349, 350, 393, 394, 395, 396, 397, 398, 399, 400]
                wlr_layer::{WlrLayerShellState, LayerShellHandler, LayerSurfaceData}, // [11, 19, 20, 41, 48, 57, 66, 67, 71, 91, 92, 95, 160, 192, 254, 292, 307, 318, 347, 401, 402]
            },
            shm::{ShmState, ShmHandler}, // [141, 165]
            selection::{
                data_device::{DataDeviceState, DataDeviceHandler}, // [11, 12, 19, 35, 44, 45, 73, 80, 141, 216, 403, 404, 405, 406]
                primary_selection::{PrimarySelectionState, PrimarySelectionHandler}, // [30, 120]
            },
            input_method::{InputMethodManagerState, InputMethodHandler, InputMethodKeyboardUserData}, // [11, 34, 35, 117, 253, 264, 268, 407, 408, 409, 410, 411]
            text_input::{TextInputManagerState, TextInputHandler, TextInputUserData}, // [86, 112, 328, 412, 413, 414]
            viewporter::ViewporterState, // [371]
            xdg_activation::XdgActivationState, // [303]
            foreign_toplevel::ForeignToplevelState, // [335]
            idle_notify::IdleNotifierState, // [58]
            explicit_synchronization::ExplicitSyncState, // [415]
            drm_lease::DrmLeaseState, // [107]
            server_decoration::ServerDecorationManagerState, // [297]
        },
        xwayland::{XWayland, XWaylandEvent, XWaylandClientData, xwm::{X11Wm, XwmHandler, X11Surface}}, // [11, 14, 19, 21, 32, 35, 36, 41, 43, 52, 60, 76, 78, 85, 117, 141, 219, 270, 310, 329, 339, 361, 363, 377, 416, 417]
    };
    use std::{collections::HashMap, sync::{Arc, Mutex}};
    use crate::system::dbus::DBusConnectionManager; // Für D-Bus Interaktionen
    use crate::system::audio::PipeWireClientService; // Für PipeWire
    use crate::domain::services::DomainLayerServices; // Wrapper für Domain-Service-Handles
    
    pub struct NovaDeSystemState {
        pub display_handle: DisplayHandle,
        pub event_loop_handle: LoopHandle<'static, Self>,
        pub loop_signal: LoopSignal,
        pub clock: Clock, // Für Timings, Animationen
    
        // Core Wayland protocol states
        pub compositor_state: Arc<Mutex<CompositorState>>,
        pub shm_state: Arc<Mutex<ShmState>>,
        pub output_manager_state: Arc<Mutex<OutputManagerState>>,
        pub seat_state: Arc<Mutex<SeatState<Self>>>,
        pub data_device_state: Arc<Mutex<DataDeviceState>>,
        pub primary_selection_state: Arc<Mutex<PrimarySelectionState>>,
    
        // Shells
        pub xdg_shell_state: Arc<Mutex<XdgShellState>>,
        pub layer_shell_state: Arc<Mutex<WlrLayerShellState>>,
        pub xdg_decoration_state: Arc<Mutex<XdgDecorationState>>,
        // Optional: xwayland_shell_state if using Smithay's xwayland shell helpers
    
        // Input related
        pub text_input_manager_state: Arc<Mutex<TextInputManagerState>>,
        pub input_method_manager_state: Arc<Mutex<InputMethodManagerState>>,
    
        // Optional Wayland protocol states
        pub presentation_time_state: Option<Arc<Mutex<PresentationState>>>,
        pub viewporter_state: Option<Arc<Mutex<ViewporterState>>>,
        pub xdg_activation_state: Option<Arc<Mutex<XdgActivationState>>>,
        pub foreign_toplevel_state: Option<Arc<Mutex<ForeignToplevelState>>>,
        pub idle_notifier_state: Option<Arc<Mutex<IdleNotifierState<Self>>>>,
        pub explicit_sync_state: Option<Arc<Mutex<ExplicitSyncState>>>,
        pub drm_lease_state: Option<Arc<Mutex<DrmLeaseState>>>,
        pub server_decoration_manager_state: Option<Arc<Mutex<ServerDecorationManagerState>>>,
    
        // Backend specific states
        pub udev_backend_data: Option<UdevBackendData>, // If using udev backend
        pub winit_data: Option<WinitData>, // If using winit backend
        pub x11_data: Option<X11BackendData>, // If using X11 backend (as a client)
    
        // XWayland
        pub xwayland: Option<XWayland>,
        pub xwm: Option<Arc<Mutex<X11Wm>>>, // X11 Window Manager for XWayland
        pub xwayland_client_data_id: Option<u32>, // To identify XWayland's client
    
        // Desktop management
        pub space: Arc<Mutex<Space<WindowElement>>>, // Manages windows and outputs
        pub popup_manager: Arc<Mutex<PopupManager>>,
    
        // Input devices
        pub seat_name: String,
        pub seat: Seat<Self>,
        pub pointer: PointerHandle<Self>,
        pub keyboard: KeyboardHandle<Self>,
        pub touch: Option<TouchHandle<Self>>, // Touch ist optional
    
        // Domain Layer Access
        pub domain_services: Arc<DomainLayerServices>,
    
        // System Services Clients
        pub dbus_conn_manager: Arc<DBusConnectionManager>,
        pub pipewire_service: Arc<PipeWireClientService>,
    
        // Internal state
        pub running: std::sync::atomic::AtomicBool,
        pub active_workspace_id: Option<crate::domain::workspaces::WorkspaceId>,
        // Weitere anwendungsspezifische Zustände
    }
    
    // Hilfsstrukturen für Backend-spezifische Daten
    pub struct UdevBackendData {
        pub session: LogindSession, // Oder eine andere Session-Implementierung
        pub gpus: GpuManager<GbmGlesBackend<GlesRenderer, DrmDeviceFd>>,
        pub primary_gpu: DrmDeviceFd,
        pub udev_backend: UdevBackend,
        // Weitere DRM/GBM/EGL spezifische Zustände
    }
    pub struct WinitData { /*... relevante Winit-Felder... */ }
    pub struct X11BackendData { /*... relevante X11-Backend-Felder... */ }
    
    // WindowElement (Beispiel, muss an NovaDE angepasst werden)
    #
    pub enum WindowElement {
        Wayland(Window), // Smithay's Window für XDG Toplevels
        X11(X11Surface), // Smithay's X11Surface
        Layer(LayerSurface), // Smithay's LayerSurface
    }
    // Implementiere notwendige Traits für WindowElement (z.B. RenderElement, SpaceElement)
    ```
    
    - Der Compositor ist das komplexeste Element der Systemschicht. Smithay 8 stellt Bausteine bereit, aber die eigentliche Fensterverwaltungslogik (Tiling, Stacking, Fokusrichtlinien basierend auf Domänenregeln) muss hier implementiert werden. Die `AnvilState` aus Smithays Anvil-Compositor dient als gute Referenz für die Strukturierung von `NovaDeSystemState` und die Integration verschiedener Smithay-Handler.

#### 3.1.2. Initialisierung (`system::compositor::initialize_compositor`)

- **Datei:** `system/src/compositor/init.rs`
- **Signatur:** `pub async fn initialize_compositor(domain_services: Arc<DomainLayerServices>, event_loop_handle: LoopHandle<'static, NovaDeSystemState>, loop_signal: LoopSignal) -> Result<NovaDeSystemState, SystemError>`
- **Schritte:**
    1. Erstelle `DisplayHandle` aus dem globalen `Display` Objekt (Annahme: `Display` wird im `main` erstellt und `DisplayHandle` wird hierher übergeben oder `NovaDeSystemState` wird direkt im `main` erstellt).
    2. Initialisiere Smithay-Backends (z.B. `LibinputInputBackend` 23, `UdevBackend` 37, DRM/GBM/EGL 16). Die Wahl des Backends (DRM/libseat für Bare-Metal, Winit/X11 für verschachtelte Entwicklung) ist eine wichtige architektonische Entscheidung. Die Implementierung sollte dies abstrahieren.
    3. Initialisiere alle relevanten Wayland-Globale unter Verwendung ihrer

---

# NovaDE Interner Schnittstellen-Implementierungsleitfaden

Dieser Implementierungsleitfaden definiert die ultrafeinen Spezifikationen und hochdetaillierten Implementierungspläne für jede Schicht der Linux-Desktop-Umgebung "NovaDE". Er ist so detailliert konzipiert, dass Entwickler ihn direkt zur Implementierung verwenden können, ohne eigene Entscheidungen treffen oder Logiken/Algorithmen entwerfen zu müssen.

---

## 1. Schicht: Kernschicht (Core Layer)

Die Kernschicht (Core Layer) bildet das Fundament von NovaDE und stellt grundlegende, universell einsetzbare Bausteine und Dienste bereit, die von allen darüberliegenden Schichten – der Domänenschicht, der Systemschicht und der Benutzeroberflächenschicht (UI-Schicht) – genutzt werden. Die Kommunikation mit höheren Schichten erfolgt durch die direkte Verwendung der hier definierten Typen und Funktionen.

### 1.1. Modul: `core::types`

Dieses Modul definiert grundlegende Datentypen, die systemweit für geometrische Berechnungen, Farbdarstellungen und andere fundamentale Konzepte verwendet werden.

- **Verzeichnis:** `core/src/types/`

#### 1.1.1. `Point<T>` Struktur

Stellt einen Punkt im 2D-Raum dar.

- Datei: `core/src/types/point.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Point<T, U> {
        pub x: T,
        pub y: T,
        _unit: std::marker::PhantomData<U>,
    }
    
    impl<T: Default, U> Default for Point<T, U> {
        fn default() -> Self {
            Point { x: Default::default(), y: Default::default(), _unit: std::marker::PhantomData }
        }
    }
    
    // U kann z.B. Logical, Physical, BufferCoords sein
    pub struct Logical;
    pub struct Physical;
    pub struct BufferCoords;
    
    impl<T, U> Point<T, U> {
        pub const fn new(x: T, y: T) -> Self {
            Point { x, y, _unit: std::marker::PhantomData }
        }
    }
    
    // Weitere Methoden wie Add, Sub, etc. können implementiert werden.
    ```
    
- Verwendung: Positionierung von UI-Elementen, Fensterkoordinaten.

#### 1.1.2. `Size<T>` Struktur

Stellt die Dimensionen (Breite und Höhe) eines 2D-Objekts dar.

- Datei: `core/src/types/size.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Size<T, U> {
        pub w: T,
        pub h: T,
        _unit: std::marker::PhantomData<U>,
    }
    
    impl<T: Default, U> Default for Size<T, U> {
        fn default() -> Self {
            Size { w: Default::default(), h: Default::default(), _unit: std::marker::PhantomData }
        }
    }
    
    impl<T, U> Size<T, U> {
        pub const fn new(w: T, h: T) -> Self {
            Size { w, h, _unit: std.marker::PhantomData }
        }
    }
    ```
    
- Verwendung: Fenstergrößen, UI-Element-Dimensionen.

#### 1.1.3. `Rect<T>` und `RectInt` Strukturen

Stellt ein Rechteck im 2D-Raum dar, definiert durch Position und Größe.

- Datei: `core/src/types/rect.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Rect<T, U> {
        pub loc: Point<T, U>,
        pub size: Size<T, U>,
    }
    
    impl<T: Default, U> Default for Rect<T, U> {
        fn default() -> Self {
            Rect { loc: Default::default(), size: Default::default() }
        }
    }
    
    impl<T, U> Rect<T, U> {
        pub const fn new(loc: Point<T, U>, size: Size<T, U>) -> Self {
            Rect { loc, size }
        }
        // Methoden wie contains_point, intersection, etc.
    }
    
    pub type RectInt = Rect<i32, Logical>; // Beispiel für einen häufigen Typ
    ```
    
- Verwendung: Fenstergeometrien, Auswahlbereiche.

#### 1.1.4. `Color` Struktur

Repräsentiert eine Farbe im RGBA-Format.

- Datei: `core/src/types/color.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct Color {
        pub r: f32, // Range 0.0 to 1.0
        pub g: f32, // Range 0.0 to 1.0
        pub b: f32, // Range 0.0 to 1.0
        pub a: f32, // Range 0.0 to 1.0 (alpha)
    }
    
    impl Color {
        pub const fn new(r: f32, g: f32, b: f32, a: f32) -> Self {
            Color { r, g, b, a }
        }
    
        pub fn to_hex_rgb(&self) -> String {
            format!(
                "#{:02x}{:02x}{:02x}",
                (self.r * 255.0) as u8,
                (self.g * 255.0) as u8,
                (self.b * 255.0) as u8
            )
        }
    
        pub fn to_hex_rgba(&self) -> String {
            format!(
                "#{:02x}{:02x}{:02x}{:02x}",
                (self.r * 255.0) as u8,
                (self.g * 255.0) as u8,
                (self.b * 255.0) as u8,
                (self.a * 255.0) as u8
            )
        }
        // Ggf. From-Implementierungen für gängige Formate (z.B. String "#RRGGBB")
    }
    ```
    
- Verwendung: Theming, UI-Elementfarben.

#### 1.1.5. `Orientation` Enum

Definiert die Ausrichtung von UI-Elementen oder Layouts.

- Datei: `core/src/types/orientation.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub enum Orientation {
        Horizontal,
        Vertical,
    }
    ```
    
- Verwendung: Layout-Manager, Ausrichtung von Bedienelementen.

#### 1.1.6. `uuid::Uuid` Integration

Universell eindeutige Identifikatoren.

- Datei: `core/src/types/mod.rs` (als Re-Export)
- Re-Export: `pub use uuid::Uuid;`
- Abhängigkeit in `core/Cargo.toml`: `uuid = { version = "1.0", features = ["v4", "serde"] }`
    - Die Features `v4` und `serde` sind essenziell. `v4` ermöglicht die Generierung zufälliger UUIDs, die für die eindeutige Identifizierung von Entitäten wie Fenstern, Workspaces oder Benachrichtigungen unerlässlich sind. `serde` wird benötigt, falls diese IDs in Konfigurationsdateien persistiert oder über IPC-Schnittstellen serialisiert werden müssen, was in einer Desktop-Umgebung sehr wahrscheinlich ist.1
- Verwendung: Eindeutige Identifizierung von Ressourcen systemweit.

#### 1.1.7. `chrono::DateTime<Utc>` Integration

Zeitstempel mit UTC-Zeitzone.

- Datei: `core/src/types/mod.rs` (als Re-Export)
- Re-Export: `pub use chrono::{DateTime, Utc};`
- Abhängigkeit in `core/Cargo.toml`: `chrono = { version = "0.4", features = ["serde"] }`
    - `chrono` ist der De-facto-Standard für Datums- und Zeitmanipulationen in Rust. Die Verwendung von `DateTime<Utc>` stellt zeitzonenagnostische Zeitstempel sicher, was für interne Repräsentationen als Best Practice gilt, um Komplexitäten bei der Zeitzonenbehandlung zu vermeiden. Das `serde`-Feature ist notwendig, wenn Zeitstempel persistiert (z.B. in Logdateien oder Konfigurationen) oder serialisiert werden sollen.3
- Verwendung: Zeitstempel für Logging, Events, "zuletzt geändert"-Informationen.

#### 1.1.8. Implementierungsschritte für `core::types`

1. Erstellen Sie das Verzeichnis `core/src/types/`.
2. Implementieren Sie `point.rs`, `size.rs`, `rect.rs`, `color.rs`, und `orientation.rs` gemäß den obigen Spezifikationen.
3. Fügen Sie `uuid` (mit Features "v4", "serde") und `chrono` (mit Feature "serde") zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
4. Fügen Sie die Re-Exports für `Uuid` und `DateTime<Utc>` in `core/src/types/mod.rs` hinzu. Alle lokalen Typen (Point, Size, etc.) sollten ebenfalls von `mod.rs` re-exportiert werden.
    
    Rust
    
    ```
    // core/src/types/mod.rs
    mod color;
    mod orientation;
    mod point;
    mod rect;
    mod size;
    
    pub use chrono::{DateTime, Utc};
    pub use color::Color;
    pub use orientation::Orientation;
    pub use point::{BufferCoords, Logical, Physical, Point};
    pub use rect::{Rect, RectInt};
    pub use size::Size;
    pub use uuid::Uuid;
    ```
    

### 1.2. Modul: `core::errors`

Dieses Modul stellt eine Basis-Fehlerbehandlungsstrategie und den `CoreError`-Typ bereit. Höhere Schichten können `CoreError` oder spezifischere Fehler aus Kernmodulen mittels `#[from]` oder `#[source]` in ihre eigenen Fehlertypen wrappen, wobei die Fehlerursachenkette (`source()`) beibehalten wird.

- **Verzeichnis:** `core/src/`
- Datei: `core/src/errors.rs`

#### 1.2.1. `CoreError` Enum

- Definition unter Verwendung von `thiserror`:
    
    Rust
    
    ```
    #
    pub enum CoreError {
        #[error("I/O error: {source}")]
        Io {
            #[from]
            source: std::io::Error,
            // Optional: context: String, // Um zusätzlichen Kontext zur IO-Operation zu geben
        },
    
        #[error("Configuration parsing error from '{path:?}': {message}")]
        ConfigParse {
            path: Option<std::path::PathBuf>,
            message: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #
        Serialization {
            message: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #
        Deserialization {
            message: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #[error("Invalid path: {path:?}")]
        InvalidPath { path: std::path::PathBuf },
    
        #[error("Initialization failed for module: {module_name}")]
        InitializationFailed {
            module_name: String,
            #[source]
            source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
        },
    
        #[error("Feature not implemented: {feature_name}")]
        NotImplemented { feature_name: String },
    
        #[error("Internal error: {message}")]
        Internal { message: String },
    
        #[error("Operation timed out: {operation}")]
        Timeout { operation: String },
    
        #[error("Permission denied for operation: {operation}")]
        PermissionDenied { operation: String },
    }
    ```
    
    Die Verwendung von `thiserror` 5 ist entscheidend für die Erstellung ergonomischer und informativer Fehlertypen. Das Attribut `#[from]` ermöglicht eine einfache Konvertierung von Standardbibliotheksfehlern wie `std::io::Error`. Das Attribut `#[source]` (implizit von `thiserror` verwendet, wenn ein Feld `#[from]` oder explizit `#[source]` ist) ist entscheidend für die Beibehaltung der Fehlerursachenkette, wie in der Spezifikation hervorgehoben. Eine robuste Fehlerbehandlungsstrategie ist fundamental. `thiserror` vereinfacht die Erstellung benutzerdefinierter Fehlertypen, die gut mit dem `Error`-Trait von Rust integriert sind. Die Beibehaltung der `source()`-Kette ist für das Debugging unerlässlich, da sie es Entwicklern ermöglicht, einen Fehler bis zu seinem Ursprung zurückzuverfolgen, selbst wenn er von verschiedenen Schichten mehrfach umgebrochen wird.

#### 1.2.2. Fehlerpropagierungsstrategie

Alle Funktionen in der Kernschicht, die fehlschlagen können, MÜSSEN `Result<T, CoreError>` oder `Result<T, SpecificCoreSubError>` zurückgeben, wobei `SpecificCoreSubError` via `#[from]` in `CoreError` konvertierbar sein sollte. Höhere Schichten definieren ihre eigenen Fehler-Enums (z.B. `DomainError`, `SystemError`, `UIError`) und verwenden `#[from] CoreError` oder `#[source] CoreError`, um Fehler aus dem Kern zu wrappen.

#### 1.2.3. Implementierungsschritte für `core::errors`

1. Fügen Sie `thiserror = "1.0"` zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
2. Implementieren Sie `core/src/errors.rs` mit dem `CoreError`-Enum wie oben spezifiziert.
3. Stellen Sie sicher, dass alle Module der Kernschicht `CoreError` für ihre Fehlerberichterstattung verwenden.

### 1.3. Modul: `core::logging`

Dieses Modul definiert die Logging-Infrastruktur basierend auf dem `tracing`-Crate. Alle höheren Schichten verwenden die `tracing`-Makros für ihre Logging-Ausgaben.

- **Verzeichnis:** `core/src/`
- Datei: `core/src/logging.rs`

#### 1.3.1. `initialize_logging()` Funktion

- Signatur:
    
    Rust
    
    ```
    pub fn initialize_logging(
        level_filter: tracing::Level,
        use_json: bool,
        custom_writer: Option<Box<dyn Fn() -> Box<dyn std::io::Write + Send + Sync + 'static> + Send + Sync + 'static>>
    ) -> Result<(), CoreError>
    ```
    
    - `level_filter`: Der minimale Log-Level, der aufgezeichnet werden soll (z.B. `tracing::Level::INFO`, `tracing::Level::DEBUG`).
    - `use_json`: Wenn `true`, werden Logs im JSON-Format ausgegeben, andernfalls in einem menschenlesbaren Format.
    - `custom_writer`: Eine optionale Funktion, die einen Writer für die Log-Ausgabe zurückgibt (z.B. in eine Datei, einen Netzwerk-Socket oder `std::io::stderr`). Standardmäßig wird `std::io::stderr` verwendet. Die Funktion wird verwendet, um den Writer bei Bedarf neu erstellen zu können (z.B. bei Log-Rotation).
- Implementierung:
    1. Verwendet `tracing_subscriber::fmt()` für die formatierte Protokollierung.
    2. Ermöglicht die Konfiguration des Formats (JSON oder "pretty") und des Writers.
        - Für JSON: `tracing_subscriber::fmt().json().with_writer(writer_closure).init();`
        - Für Pretty: `tracing_subscriber::fmt().pretty().with_writer(writer_closure).init();`
    3. Konfiguriert den `EnvFilter` basierend auf `level_filter` oder einer Umgebungsvariable (z.B. `RUST_LOG`).
    4. Richtet einen globalen Standard-Subscriber ein mittels `tracing::subscriber::set_global_default`.
    5. Gibt `Result<(), CoreError>` zurück, um Initialisierungsfehler zu signalisieren (z.B. wenn das Setzen des globalen Subscribers fehlschlägt).
- Die Standardisierung auf `tracing` ermöglicht eine strukturierte Protokollierung über alle Schichten hinweg, was für das Debugging und die Überwachung eines komplexen Systems wie einer Desktop-Umgebung von unschätzbarem Wert ist. Die Möglichkeit, zwischen menschenlesbaren und JSON-Formaten zu wechseln, ist gut für Entwicklung versus Produktion/Log-Aggregation. `tracing` bietet ein leistungsfähigeres und flexibleres Logging-Framework als das ältere `log`-Crate, insbesondere durch sein Konzept von Spans für kontextbezogene Protokollierung. Dies ist in einem mehrschichtigen, ereignisgesteuerten System von großem Vorteil.

#### 1.3.2. Verwendung von `tracing`-Makros

Alle Schichten verwenden die Makros `trace!`, `debug!`, `info!`, `warn!`, `error!` aus dem `tracing`-Crate. Spans (`span!`, `info_span!`, etc.) sollten verwendet werden, um kontextuelle Informationen für zusammengehörige Log-Meldungen bereitzustellen.

- Beispiel: `info!(target: "core::config", path = %file_path.display(), "Lade Kernkonfiguration.");`
- Beispiel mit Span:
    
    Rust
    
    ```
    let workspace_span = info_span!("workspace_operation", id = %workspace_id);
    let _enter = workspace_span.enter();
    //... Operationen innerhalb des Spans...
    info!("Operation abgeschlossen");
    ```
    

#### 1.3.3. Implementierungsschritte für `core::logging`

1. Fügen Sie `tracing = "0.1"` und `tracing_subscriber = { version = "0.3", features = ["fmt", "json", "env-filter"] }` zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
2. Implementieren Sie `core/src/logging.rs` mit der Funktion `initialize_logging()` wie oben spezifiziert.

### 1.4. Modul: `core::config`

Dieses Modul stellt Primitive zum Laden, Parsen und Zugreifen auf Kernkonfigurationen bereit. Es definiert die Struktur `CoreConfig` und Funktionen für deren Verwaltung.

- **Verzeichnis:** `core/src/config/`

#### 1.4.1. `CoreConfig` Struktur

Definiert die Kernkonfigurationseinstellungen.

- Datei: `core/src/config/types.rs`
- Definition:
    
    Rust
    
    ```
    #
    pub struct CoreConfig {
        pub log_level: String, // z.B. "info", "debug", "trace"
        pub log_format: LogFormat,
        pub default_theme_name: Option<String>,
        pub enable_xwayland: bool,
        // Weitere kernspezifische Konfigurationen, z.B. Pfade, Feature-Flags
    }
    
    #
    #[serde(rename_all = "lowercase")]
    pub enum LogFormat {
        Json,
        Pretty,
    }
    
    impl Default for CoreConfig {
        fn default() -> Self {
            CoreConfig {
                log_level: "info".to_string(),
                log_format: LogFormat::Pretty,
                default_theme_name: Some("NovaDark".to_string()),
                enable_xwayland: true,
            }
        }
    }
    ```
    

#### 1.4.2. `ConfigError` Enum

Spezifische Fehler für das Laden und Verarbeiten von Konfigurationen.

- Datei: `core/src/config/error.rs`
- Definition:
    
    Rust
    
    ```
    use crate::core::errors::CoreError; // Assuming CoreError is in crate::core
    
    #
    pub enum ConfigError {
        #[error("Failed to determine config directory: {0}")]
        DirectoryResolutionError(String),
    
        #[error("Config file not found at {path:?}")]
        FileNotFound { path: std::path::PathBuf },
    
        #[error("Failed to read config file {path:?}")]
        FileReadError {
            path: std::path::PathBuf,
            #[source]
            source: std::io::Error,
        },
    
        #
        TomlParseError {
            path: std::path::PathBuf,
            message: String,
            #[source]
            source: Option<toml::de::Error>,
        },
        // Ggf. JsonParseError, falls JSON auch unterstützt wird
    
        #[error("Configuration not loaded or initialization failed.")]
        NotLoaded,
    
        #[error("Invalid configuration value for key '{key}': {message}")]
        InvalidValue { key: String, message: String },
    
        #[error("Failed to write default config to {path:?}")]
        DefaultConfigWriteError {
            path: std::path::PathBuf,
            #[source]
            source: std::io::Error,
        },
    
        #
        DefaultConfigTomlError(#[from] toml::ser::Error),
    
        #[error(transparent)]
        Core(#[from] CoreError), // Um CoreError wrappen zu können
    }
    ```
    

#### 1.4.3. Globale Konfigurationsvariable

Verwendet `once_cell` für eine thread-sichere, einmalig initialisierte globale Konfiguration.

- Datei: `core/src/config/mod.rs` (oder `core/src/config/global.rs`)
    
    Rust
    
    ```
    use once_cell::sync::OnceCell;
    use std::sync::RwLock;
    use super::types::CoreConfig; // Pfad anpassen
    use super::error::ConfigError; // Pfad anpassen
    
    static CORE_CONFIG: OnceCell<RwLock<CoreConfig>> = OnceCell::new();
    ```
    

#### 1.4.4. `load_core_config()` Funktion

Lädt die Kernkonfiguration aus einer Datei.

- Datei: `core/src/config/loader.rs`
- Signatur: `pub fn load_core_config(custom_path: Option<std::path::PathBuf>) -> Result<(), ConfigError>`
- Implementierung:
    
    1. Bestimme den Konfigurationsdateipfad:
        - Verwende `custom_path`, falls angegeben.
        - Andernfalls suche im XDG-Standardverzeichnis (z.B. `$XDG_CONFIG_HOME/nova-de/core.toml`) mittels `directories_next::ProjectDirs`.
        - Falls kein XDG-Verzeichnis gefunden wird, verwende einen Fallback-Pfad (z.B. `~/.config/nova-de/core.toml`).
    2. Wenn die Datei nicht existiert, erstelle eine Standardkonfiguration (`CoreConfig::default()`) und speichere sie unter dem ermittelten Pfad. Logge diesen Vorgang.
    3. Lese den Dateiinhalt. Bei Fehler `ConfigError::FileReadError`.
    4. Parse den Inhalt (z.B. TOML) in eine `CoreConfig`-Instanz. Bei Fehler `ConfigError::TomlParseError`.
    5. Initialisiere die globale `CORE_CONFIG` Variable mit der geladenen Konfiguration: `CORE_CONFIG.set(RwLock::new(loaded_config)).map_err(|_| ConfigError::NotLoaded)` (Fehlerbehandlung für den Fall, dass `set` fehlschlägt, was nur passiert, wenn bereits initialisiert).
    6. Gibt `Ok(())` bei Erfolg zurück.
    
    - Eine globale, statisch zugängliche Konfiguration ist üblich, erfordert aber eine sorgfältige Verwaltung, insbesondere bei der Initialisierung. `once_cell` für die Lazy-Initialisierung und `RwLock` für den threadsicheren Zugriff sind ein gutes Muster. Die Konfiguration muss früh im Anwendungslebenszyklus verfügbar sein und von verschiedenen Teilen des Systems zugänglich sein. `once_cell` stellt sicher, dass das Laden nur einmal erfolgt.

#### 1.4.5. `get_core_config()` Funktion

Bietet globalen Lesezugriff auf die geladene `CoreConfig`.

- Datei: `core/src/config/access.rs`
- Signatur: `pub fn get_core_config() -> Result<std::sync::RwLockReadGuard<'static, CoreConfig>, ConfigError>`
- Implementierung:
    1. Versuche, einen Read-Lock auf `CORE_CONFIG` zu erhalten: `CORE_CONFIG.get().ok_or(ConfigError::NotLoaded)?.read().map_err(|_| ConfigError::NotLoaded)` (Fehlerbehandlung für Poisoned Lock).
    2. Gibt den `RwLockReadGuard` bei Erfolg zurück.

#### 1.4.6. `update_core_config()` Funktion (Optional)

Ermöglicht die dynamische Aktualisierung der Kernkonfiguration zur Laufzeit.

- Datei: `core/src/config/access.rs`
- Signatur: `pub fn update_core_config(new_config: CoreConfig) -> Result<(), ConfigError>`
- Implementierung:
    1. Versuche, einen Write-Lock auf `CORE_CONFIG` zu erhalten.
    2. Aktualisiere die Konfiguration.
    3. Persistiert die neue Konfiguration in die Datei.
    4. **Wichtig:** Benachrichtige relevante Systemteile über die Konfigurationsänderung (z.B. über ein Event). Dies erfordert sorgfältige Überlegungen zur Thread-Sicherheit und wie Komponenten auf dynamische Änderungen reagieren.

#### 1.4.7. Implementierungsschritte für `core::config`

1. Fügen Sie `serde = { version = "1.0", features = ["derive"] }`, `toml = "0.8"`, `once_cell = "1.19"`, `directories-next = "2.0"` zu den Abhängigkeiten in `core/Cargo.toml` hinzu.
2. Erstellen Sie die Verzeichnisstruktur `core/src/config/`.
3. Definieren Sie `CoreConfig` und `LogFormat` in `core/src/config/types.rs`.
4. Definieren Sie `ConfigError` in `core/src/config/error.rs`.
5. Implementieren Sie `loader.rs` (mit `load_core_config`) und `access.rs` (mit `get_core_config` und optional `update_core_config`).
6. Fügen Sie die globale `CORE_CONFIG`-Variable in `core/src/config/mod.rs` oder einer dedizierten `global.rs` ein und exportieren Sie die öffentlichen Funktionen.

### 1.5. Modul: `core::utils`

Dieses Modul bietet allgemeine Hilfsfunktionen, die von allen höheren Schichten genutzt werden können.

- **Verzeichnis:** `core/src/utils/`
- **Struktur:** Das Modul sollte in thematische Untermodule aufgeteilt werden, um die Übersichtlichkeit zu wahren.

#### 1.5.1. Untermodul: `core::utils::file_utils`

- Datei: `core/src/utils/file_utils.rs`
- Funktionen:
    - `pub fn read_file_to_string(path: &std::path::Path) -> Result<String, CoreError>`
        - Implementierung: Verwendet `std::fs::read_to_string`. Mappt `std::io::Error` auf `CoreError::Io`.
    - `pub fn ensure_dir_exists(path: &std::path::Path) -> Result<(), CoreError>`
        - Implementierung: Verwendet `std::fs::create_dir_all`. Mappt `std::io::Error` auf `CoreError::Io`.
    - `pub fn write_string_to_file(path: &std::path::Path, content: &str) -> Result<(), CoreError>`
        - Implementierung: Verwendet `std::fs::write`. Mappt `std::io::Error` auf `CoreError::Io`.

#### 1.5.2. Untermodul: `core::utils::string_utils`

- Datei: `core/src/utils/string_utils.rs`
- Funktionen:
    - `pub fn sanitize_filename(name: &str) -> String`
        - Implementierung: Ersetzt ungültige Zeichen für Dateinamen (z.B. `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`) durch Unterstriche oder entfernt sie.
    - `pub fn truncate_string_with_ellipsis(s: &str, max_len: usize) -> String`
        - Implementierung: Kürzt den String auf `max_len` Zeichen und fügt "..." hinzu, falls er gekürzt wurde. Achtet auf Grapheme Cluster, um Unicode-Zeichen nicht zu zerschneiden.
            
            Rust
            
            ```
            use unicode_segmentation::UnicodeSegmentation;
            //...
            // if s.graphemes(true).count() > max_len {
            //     s.graphemes(true).take(max_len - 3).collect::<String>() + "..."
            // } else {
            //     s.to_string()
            // }
            ```
            
            - Abhängigkeit: `unicode-segmentation = "1.10"`

#### 1.5.3. Untermodul: `core::utils::async_utils` (falls benötigt)

- Datei: `core/src/utils/async_utils.rs`
- Mögliche Funktionen:
    - Wrapper für `tokio::spawn` mit standardisierter Fehlerbehandlung oder Logging.
    - Hilfsfunktionen für Timeout-Management bei asynchronen Operationen.

#### 1.5.4. Implementierungsschritte für `core::utils`

1. Erstellen Sie das Verzeichnis `core/src/utils/`.
2. Erstellen Sie `core/src/utils/mod.rs` und deklarieren Sie die Untermodule:
    
    Rust
    
    ```
    pub mod file_utils;
    pub mod string_utils;
    // pub mod async_utils;
    ```
    
3. Implementieren Sie die Funktionen in den jeweiligen Untermodul-Dateien.
4. Fügen Sie ggf. Abhängigkeiten wie `unicode-segmentation` zu `core/Cargo.toml` hinzu.
5. Das `utils`-Modul kann leicht zu einer Ansammlung unzusammenhängender Funktionen werden. Es ist wichtig, die Organisation beizubehalten und sicherzustellen, dass die Funktionen eine breite Anwendbarkeit haben. Wenn eine Hilfsfunktion sehr spezifisch für eine einzelne Domäne oder ein Systemanliegen ist, sollte sie stattdessen im Hilfsmodul dieser Schicht angesiedelt sein. Eine klare Abgrenzung ist notwendig, um die Kohäsion der Module zu wahren und unnötige Kopplungen zu vermeiden.

---

## 2. Schicht: Domänenschicht (Domain Layer)

Die Domänenschicht kapselt die Geschäftslogik und den Kernzustand der Desktop-Umgebung NovaDE. Sie definiert die zentralen Konzepte, Regeln und Verhaltensweisen des Systems, unabhängig von technischen Implementierungsdetails der System- oder UI-Schicht. Die Kommunikation mit höheren Schichten (System- und UI-Schicht) erfolgt primär über wohldefinierte Service-APIs (als Rust-Traits implementiert) und durch domänenspezifische Events, die Zustandsänderungen signalisieren.

Die Domänenschicht ist zentral für die Funktionalität von NovaDE. Die Verwendung von Rust-Traits für Service-APIs fördert lose Kopplung und Testbarkeit. Ereignisse sind der Schlüssel zur reaktiven Propagierung von Zustandsänderungen. Fehlertypen sollten für jedes Domänenmodul spezifisch sein, um klaren Kontext zu bieten.

### 2.1. Service APIs (Traits)

Öffentliche Schnittstellen der Domänenschicht werden primär durch Rust-Traits definiert. Diese Traits werden von Service-Strukturen innerhalb der Domänenmodule implementiert. Methoden sind typischerweise `async`, wenn sie I/O-Operationen oder potenziell langlaufende Prozesse beinhalten, und geben `Result<T, SpecificDomainError>` zurück.

#### 2.1.1. `domain::theming::ThemingEngine` Trait

Verantwortlich für die Verwaltung und Anwendung von Themes.

- **Datei:** `domain/src/theming/engine.rs`

##### Zugehörige Datenstrukturen und Typen:

- **`ThemingConfiguration` Struct (Wertobjekt):**
    
    - Definiert die vom Benutzer oder System gewählten Theme-Einstellungen.
    - **Datei:** `domain/src/theming/config.rs`
    - **Felder:**
        - `icon_theme_name: String` (z.B. "Adwaita", "Papirus")
        - `gtk_theme_name: String` (z.B. "Adwaita-dark")
        - `cursor_theme_name: String` (z.B. "Adwaita")
        - `font_name: String` (z.B. "Cantarell 11")
        - `wallpaper_path: Option<std::path::PathBuf>`
        - `color_scheme: ColorScheme` (Enum: `Light`, `Dark`, `SystemPreference`)
        - `accent_color: core::types::Color`
    - **Invarianten:** `font_name` darf nicht leer sein, wenn gesetzt.
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`ColorScheme` Enum (Wertobjekt):**
    
    - Definiert die möglichen Farbschemata.
    - **Datei:** `domain/src/theming/config.rs` (oder `types.rs` im Theming-Modul)
    - **Varianten:** `Light`, `Dark`, `SystemPreference`
    - **Sichtbarkeit:** `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`AppliedThemeState` Struct (Wertobjekt):**
    
    - Repräsentiert den aktuell angewandten Theme-Zustand, inklusive aufgelöster Werte.
    - **Datei:** `domain/src/theming/state.rs`
    - **Felder:**
        - `current_config: ThemingConfiguration`
        - `resolved_wallpaper_path: Option<std::path::PathBuf>` (absoluter Pfad zum tatsächlich genutzten Hintergrundbild)
        - `font_config_details: String` (z.B. eine Pango-Font-Beschreibung oder Fontconfig-Pattern)
        - `effective_color_scheme: EffectiveColorScheme` (Enum: `Light`, `Dark`)
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`EffectiveColorScheme` Enum (Wertobjekt):**
    
    - Repräsentiert das tatsächlich aktive Farbschema (aufgelöst aus `SystemPreference`).
    - **Datei:** `domain/src/theming/state.rs`
    - **Varianten:** `Light`, `Dark`
    - **Sichtbarkeit:** `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`ThemingError` Enum (Fehlertyp):**
    
    - Spezifische Fehler für das Theming-Subsystem.
    - **Datei:** `domain/src/theming/error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum ThemingError {
            #[error("Invalid theming configuration: {0}")]
            InvalidConfiguration(String),
        
            #[error("Wallpaper not found at path: {0:?}")]
            WallpaperNotFound(std::path::PathBuf),
        
            #[error("Failed to apply theme '{theme_name}': {details}")]
            ThemeApplyFailed {
                theme_name: String,
                details: String,
                #[source]
                source: Option<Box<dyn std::error::Error + Send + Sync + 'static>>,
            },
        
            #[error("Persistence error during theming operation")]
            PersistenceError(#[from] crate::core::errors::CoreError), // Assuming CoreError is accessible
        
            #[error("Underlying system service error for theming: {0}")]
            SystemServiceError(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn get_current_theme_state(&self) -> Result<AppliedThemeState, ThemingError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft den aktuell im System angewandten und zwischengespeicherten Theme-Zustand ab.
    - **Vorbedingungen:** Keine.
    - **Nachbedingungen:** Gibt den `AppliedThemeState` zurück oder einen `ThemingError`.
- `async fn update_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Aktualisiert die Theme-Konfiguration, persistiert sie und stößt die Anwendung des neuen Themes an.
    - **Vorbedingungen:** `new_config` muss valide sein (ggf. interne Validierung).
    - **Nachbedingungen:** Bei Erfolg wird das `ThemeChangedEvent` publiziert. Die Konfiguration wird persistiert (z.B. über `GlobalSettingsService` oder ein dediziertes Persistenz-Interface). Das System-Theme wird (versucht) anzuwenden.
    - **Geschäftsregeln:** Validiert die `new_config`. Löst das Hintergrundbild auf.
- `async fn apply_theme_to_system(&self, theme_state: &AppliedThemeState) -> Result<(), ThemingError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** (Potenziell intern, aufgerufen von `update_configuration`, oder als separate Methode für fortgeschrittene Nutzung) Wendet den gegebenen `AppliedThemeState` auf die Systemkomponenten an (z.B. Setzen von GTK-Theme, Icons, Cursor über Systemmechanismen).
    - **Vorbedingungen:** `theme_state` ist valide.
    - **Nachbedingungen:** Theme-Änderungen sind im System (versucht) aktiv.

##### Publizierte Events:

- **`ThemeChangedEvent`:**
    - **Datei:** `domain/src/theming/events.rs`
    - **Struktur:**
        
        Rust
        
        ```
        #
        pub struct ThemeChangedEvent {
            pub new_state: AppliedThemeState,
            pub old_state: Option<AppliedThemeState>, // Optional, falls der vorherige Zustand bekannt ist
        }
        ```
        
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::theming::ThemeChangedEvent`
    - **Typische Publisher:** Die Implementierung von `ThemingEngine`.
    - **Typische Subscriber:** `ui::theming_gtk` (UI-Schicht), `system::compositor` (Systemschicht, falls relevant für Darstellung).

#### 2.1.2. `domain::workspaces::WorkspaceManager` Trait

Verwaltet Workspaces (virtuelle Desktops).

- **Datei:** `domain/src/workspaces/manager.rs`

##### Zugehörige Datenstrukturen und Typen:

- **`WorkspaceId` Typalias:**
    
    - **Datei:** `domain/src/workspaces/types.rs` (oder direkt in `manager.rs`)
    - **Definition:** `pub type WorkspaceId = core::types::Uuid;`
- **`ApplicationIdentifier` Struct (Wertobjekt):**
    
    - Identifiziert eine Anwendung eindeutig.
    - **Datei:** `domain/src/workspaces/app_id.rs`
    - **Felder:**
        - `id_type: ApplicationIdType` (Enum: `DesktopFile`, `WaylandAppId`, `X11WindowClass`)
        - `identifier_str: String` (z.B. "firefox.desktop", "org.gnome.Nautilus", "Firefox")
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `serde::Serialize`, `serde::Deserialize`.
- **`Workspace` Struct (Entität):**
    
    - Repräsentiert einen einzelnen Workspace.
    - **Datei:** `domain/src/workspaces/workspace.rs`
    - **Felder:**
        - `id: WorkspaceId` (Sichtbarkeit: `pub(crate)`, Zugriff über Getter)
        - `name: String` (Sichtbarkeit: `pub`)
        - `output_name: Option<String>` (Name des Monitors, dem der Workspace primär zugeordnet ist, Sichtbarkeit: `pub`)
        - `layout_policy: WorkspaceLayoutPolicy` (Enum: `Tiling`, `Floating`, `Hybrid`, Sichtbarkeit: `pub`)
        - `pinned_applications: std::collections::HashSet<ApplicationIdentifier>` (Sichtbarkeit: `pub(crate)`, Zugriff über Methoden)
        - `creation_timestamp: core::types::DateTime<core::types::Utc>` (Sichtbarkeit: `pub(crate)`)
        - `last_accessed_timestamp: core::types::DateTime<core::types::Utc>` (Sichtbarkeit: `pub(crate)`)
    - **Initialwerte:** `id` wird bei Erstellung generiert, `name` z.B. "Workspace 1", `output_name` ist `None`, `layout_policy` wird aus globalen Einstellungen übernommen oder ist Standard (`Floating`), `pinned_applications` ist leer, `creation_timestamp` und `last_accessed_timestamp` sind Zeitpunkte der Erstellung.
    - **Invarianten:** `name` darf nicht leer sein. `id` ist unveränderlich nach Erstellung.
    - **Methoden (Beispiele):**
        - `pub fn new(name: String, layout_policy: WorkspaceLayoutPolicy) -> Self`
        - `pub fn id(&self) -> WorkspaceId`
        - `pub fn add_pinned_app(&mut self, app_id: ApplicationIdentifier) -> bool`
        - `pub fn remove_pinned_app(&mut self, app_id: &ApplicationIdentifier) -> bool`
        - `pub fn is_app_pinned(&self, app_id: &ApplicationIdentifier) -> bool`
        - `pub(crate) fn set_last_accessed(&mut self)`
    - **Beziehungen:** Keine direkten Entitätsbeziehungen in dieser Struktur, aber `pinned_applications` verweist auf `ApplicationIdentifier`.
- **`WorkspaceLayoutPolicy` Enum (Wertobjekt):**
    
    - **Datei:** `domain/src/workspaces/types.rs`
    - **Varianten:** `Tiling`, `Floating`, `Hybrid`
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`WorkspaceManagerError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/workspaces/error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum WorkspaceManagerError {
            #
            WorkspaceNotFound(super::types::WorkspaceId), // super::types::WorkspaceId
        
            #[error("Workspace with name '{0}' already exists")]
            DuplicateWorkspaceName(String),
        
            #[error("Maximum number of workspaces reached ({0})")]
            MaxWorkspacesReached(usize),
        
            #[error("Cannot delete the last workspace")]
            CannotDeleteLastWorkspace,
        
            #[error("Cannot delete active workspace; switch to another first")]
            CannotDeleteActiveWorkspace,
        
            #[error("Persistence error during workspace operation")]
            PersistenceError(#[from] crate::core::errors::CoreError),
        
            #[error("Invalid application identifier: {0}")]
            InvalidApplicationId(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn create_workspace(&self, name: Option<String>) -> Result<Workspace, WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Erstellt einen neuen Workspace. Wenn `name` `None` ist, wird ein Standardname generiert (z.B. "Workspace N").
    - **Nachbedingungen:** Bei Erfolg wird `WorkspaceEvent::Created` publiziert. Der neue Workspace wird persistiert.
    - **Geschäftsregeln:** Prüft auf Namenskonflikte und maximale Anzahl von Workspaces.
- `async fn delete_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Löscht den Workspace mit der gegebenen ID.
    - **Vorbedingungen:** Workspace darf nicht der letzte verbleibende sein. Workspace darf nicht der aktive sein (oder es muss eine Logik zum Wechsel des aktiven Workspaces implementiert sein).
    - **Nachbedingungen:** Bei Erfolg wird `WorkspaceEvent::Deleted` publiziert. Änderungen werden persistiert.
- `async fn get_workspace_by_id(&self, id: WorkspaceId) -> Result<Option<Workspace>, WorkspaceManagerError>`
    - **`noexcept`**: `false`
- `async fn get_all_workspaces(&self) -> Result<Vec<Workspace>, WorkspaceManagerError>`
    - **`noexcept`**: `false`
- `async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Vorbedingungen:** Der Workspace mit `id` muss existieren.
    - **Nachbedingungen:** `WorkspaceEvent::ActiveChanged` wird publiziert. Der `last_accessed_timestamp` des neuen und ggf. des alten aktiven Workspaces wird aktualisiert.
- `async fn get_active_workspace_id(&self) -> Result<Option<WorkspaceId>, WorkspaceManagerError>`
    - **`noexcept`**: `false`
- `async fn rename_workspace(&self, id: WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Vorbedingungen:** `new_name` darf nicht leer sein und nicht mit einem existierenden Workspace-Namen (außer dem eigenen) kollidieren.
    - **Nachbedingungen:** `WorkspaceEvent::Renamed` wird publiziert. Änderung wird persistiert.
- `async fn pin_application_to_space(&self, space_id: WorkspaceId, app_id: ApplicationIdentifier) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Nachbedingungen:** `WorkspaceEvent::AppPinned` wird publiziert. Änderung wird persistiert.
- `async fn unpin_application_from_space(&self, space_id: WorkspaceId, app_id: ApplicationIdentifier) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Nachbedingungen:** `WorkspaceEvent::AppUnpinned` wird publiziert. Änderung wird persistiert.
- `async fn move_window_to_workspace(&self, window_identifier: String /* Eindeutiger Fenster-Identifikator, z.B. Wayland Surface ID als String */, target_workspace_id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Fordert an, ein Fenster zum Ziel-Workspace zu verschieben. Die eigentliche Fensterverschiebung erfolgt in der Systemschicht.
    - **Nachbedingungen:** `WorkspaceEvent::WindowMoved` wird publiziert.

##### Publizierte Events:

- **`WorkspaceEvent` Enum:**
    - **Datei:** `domain/src/workspaces/events.rs`
    - **Varianten (Beispiele):**
        - `Created { workspace: Workspace }`
        - `Deleted { workspace_id: WorkspaceId, new_active_workspace_id: Option<WorkspaceId> }`
        - `ActiveChanged { old_id: Option<WorkspaceId>, new_id: WorkspaceId }`
        - `Renamed { workspace_id: WorkspaceId, new_name: String }`
        - `AppPinned { space_id: WorkspaceId, app_id: ApplicationIdentifier }`
        - `AppUnpinned { space_id: WorkspaceId, app_id: ApplicationIdentifier }`
        - `LayoutPolicyChanged { workspace_id: WorkspaceId, new_policy: WorkspaceLayoutPolicy }`
        - `WindowMoved { window_identifier: String, source_workspace_id: Option<WorkspaceId>, target_workspace_id: WorkspaceId }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::workspaces::WorkspaceEvent`
    - **Typische Publisher:** Die Implementierung von `WorkspaceManager`.
    - **Typische Subscriber:** `ui::shell` (UI-Schicht), `system::compositor` (Systemschicht).

#### 2.1.3. `domain::user_centric_services::AIInteractionLogicService` Trait

Steuert die Logik für Benutzerinteraktionen mit KI-Diensten, inklusive Einwilligungsmanagement.

- **Datei:** `domain/src/user_centric_services/ai_logic.rs`
- Diese Service-Schnittstelle ist entscheidend für die sichere und transparente Integration von KI-Funktionen. Die Trennung der Einwilligungslogik von der reinen MCP-Kommunikation (die in der Systemschicht angesiedelt ist) ist ein wichtiger Aspekt für Datenschutz und Benutzerkontrolle.

##### Zugehörige Datenstrukturen und Typen:

- **`AIInteractionContextData` Struct (Wertobjekt):**
    
    - Sammelt Kontextinformationen für eine KI-Anfrage.
    - **Datei:** `domain/src/user_centric_services/ai_context.rs`
    - **Felder:**
        - `active_window_title: Option<String>`
        - `active_app_id: Option<super::workspaces::app_id::ApplicationIdentifier>` // Pfad anpassen
        - `current_workspace_id: Option<super::workspaces::types::WorkspaceId>` // Pfad anpassen
        - `selected_text: Option<String>`
        - `clipboard_content_preview: Option<String>` (Vorschau, nicht der volle Inhalt)
        - `screenshot_data_uri: Option<String>` (Data-URI eines Screenshots, falls relevant)
        - `user_query: String` (Die eigentliche Anfrage des Benutzers)
        - `interaction_id: core::types::Uuid` (Eindeutige ID für diese Interaktion)
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`AIConsent` Enum (Wertobjekt):**
    
    - Repräsentiert den Einwilligungsstatus des Benutzers für KI-Interaktionen.
    - **Datei:** `domain/src/user_centric_services/ai_consent.rs`
    - **Varianten:**
        - `Granted { for_session: bool, timestamp: core::types::DateTime<core::types::Utc> }` (Einwilligung für die aktuelle Sitzung oder dauerhaft)
        - `Denied { timestamp: core::types::DateTime<core::types::Utc> }`
        - `NotSet`
    - **Sichtbarkeit:** `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`AIInteractionResponse` Struct (Wertobjekt):**
    
    - Repräsentiert die Antwort von einem KI-Dienst.
    - **Datei:** `domain/src/user_centric_services/ai_response.rs`
    - **Felder:**
        - `interaction_id: core::types::Uuid`
        - `response_text: Option<String>`
        - `suggested_actions: Vec<String>`
        - `tool_call_id: Option<String>` (Falls ein Tool vom MCP-Server aufgerufen wurde)
        - `tool_call_result: Option<serde_json::Value>` (Ergebnis des Tool-Aufrufs)
        - `error_message: Option<String>`
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `serde::Serialize`, `serde::Deserialize`.
- **`AIInteractionError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/user_centric_services/ai_error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum AIInteractionError {
            #[error("User consent not given for AI interaction")]
            ConsentNotGiven,
        
            #[error("MCP client error: {0}")]
            MCPError(String), // Wird von SystemMcpService kommen
        
            #[error("Failed to prepare context for AI interaction: {0}")]
            ContextPreparationFailed(String),
        
            #[error("AI service returned an error: {0}")]
            AIServiceError(String),
        
            #[error("Persistence error during AI consent operation")]
            PersistenceError(#[from] crate::core::errors::CoreError),
        
            #
            InteractionNotFoundOrTimedOut(crate::core::types::Uuid),
        }
        ```
        

##### Methoden des Traits:

- `async fn initiate_interaction(&self, context_data: AIInteractionContextData) -> Result<core::types::Uuid /* interaction_id */, AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Startet eine neue KI-Interaktion. Prüft die Einwilligung und leitet die Anfrage ggf. an den `SystemMcpService` weiter. Gibt eine `interaction_id` zurück, um die Antwort später zu korrelieren.
    - **Vorbedingungen:** `get_consent_status()` darf nicht `Denied` sein. Wenn `NotSet`, wird die Interaktion ggf. pausiert und ein `AIConsentRequestEvent` ausgelöst.
    - **Nachbedingungen:** `AIInteractionStartedEvent` wird publiziert.
    - **Geschäftsregeln:** Die Logik hier muss entscheiden, ob basierend auf `context_data` und `AIConsent` die Interaktion fortgesetzt, abgelehnt oder eine erneute Einwilligung angefordert wird.
- `async fn provide_consent(&self, consent: AIConsent) -> Result<(), AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Speichert die Benutzereinwilligung.
    - **Nachbedingungen:** `AIConsentChangedEvent` wird publiziert. Der Einwilligungsstatus wird persistiert.
- `async fn get_consent_status(&self) -> Result<AIConsent, AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft den aktuellen Einwilligungsstatus ab.
- `async fn process_mcp_response(&self, interaction_id: core::types::Uuid, response: Result<serde_json::Value, String> /* MCP Tool Call Result oder Fehler */) -> Result<(), AIInteractionError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Verarbeitet eine asynchrone Antwort vom `SystemMcpService` (typischerweise ein Tool-Aufruf-Ergebnis).
    - **Nachbedingungen:** `AIInteractionCompletedEvent` oder `AIInteractionFailedEvent` wird publiziert.

##### Publizierte Events:

- **`AIConsentChangedEvent`:**
    - **Datei:** `domain/src/user_centric_services/events.rs`
    - **Struktur:** `pub struct AIConsentChangedEvent { pub new_status: AIConsent; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIConsentChangedEvent`
- **`AIInteractionStartedEvent`:**
    - **Struktur:** `pub struct AIInteractionStartedEvent { pub interaction_id: core::types::Uuid, pub context: AIInteractionContextData; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIInteractionStartedEvent`
- **`AIInteractionCompletedEvent`:**
    - **Struktur:** `pub struct AIInteractionCompletedEvent { pub response: AIInteractionResponse; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIInteractionCompletedEvent`
- **`AIInteractionFailedEvent`:**
    - **Struktur:** `pub struct AIInteractionFailedEvent { pub interaction_id: core::types::Uuid, pub error: AIInteractionError; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIInteractionFailedEvent`
- **`AIConsentRequestEvent`:** (Falls die UI den Benutzer zur Einwilligung auffordern soll)
    - **Struktur:** `pub struct AIConsentRequestEvent { pub interaction_id: core::types::Uuid, pub for_context: AIInteractionContextData }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::AIConsentRequestEvent`
- **Typische Publisher:** Die Implementierung von `AIInteractionLogicService`.
- **Typische Subscriber:** `ui::command_palette` oder KI-Widgets (UI-Schicht), `system::mcp::SystemMcpService` (Systemschicht, für Antworten).

#### 2.1.4. `domain::user_centric_services::NotificationService` Trait

Verwaltet das Anzeigen und Interagieren mit Desktop-Benachrichtigungen.

- **Datei:** `domain/src/user_centric_services/notification_service.rs`
- Die Parameter für `post_notification` spiegeln eng die D-Bus-Methode `org.freedesktop.Notifications.Notify` wider.9 Dieser Dienst fungiert als Abstraktion über potenzielle D-Bus-Implementierungen in der Systemschicht. Die Domänenschicht definiert das _Konzept_ einer Benachrichtigung innerhalb von NovaDE. Die Systemschicht wäre dann dafür verantwortlich, diese Benachrichtigung tatsächlich über D-Bus zu senden, wenn NovaDE als Benachrichtigungsserver fungiert, oder auf D-Bus-Benachrichtigungen zu lauschen, wenn es diese konsumiert. Die Verwendung von `zvariant::OwnedValue` für `hints` deutet auf eine direkte Abbildung auf D-Bus-Varianten hin.

##### Zugehörige Datenstrukturen und Typen:

- **`NotificationId` Typalias:**
    
    - **Datei:** `domain/src/user_centric_services/types.rs` (oder direkt in `notification_service.rs`)
    - **Definition:** `pub type NotificationId = core::types::Uuid;`
- **`NotificationAction` Struct (Wertobjekt):**
    
    - Definiert eine Aktion, die mit einer Benachrichtigung verbunden ist.
    - **Datei:** `domain/src/user_centric_services/notification_action.rs`
    - **Felder:**
        - `key: String` (Eindeutiger Schlüssel für die Aktion, z.B. "default", "reply")
        - `label: String` (Anzeigetext für den Button)
    - **Sichtbarkeit:** Alle Felder `pub`.
    - **Ableitungen:** `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `serde::Serialize`, `serde::Deserialize`.
- **`NotificationUrgency` Enum (Wertobjekt):**
    
    - Definiert die Dringlichkeitsstufe einer Benachrichtigung.
    - **Datei:** `domain/src/user_centric_services/types.rs`
    - **Varianten:** `Low`, `Normal`, `Critical`
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`Notification` Struct (Entität):**
    
    - Repräsentiert eine einzelne Desktop-Benachrichtigung.
    - **Datei:** `domain/src/user_centric_services/notification.rs`
    - **Felder:**
        - `id: NotificationId` (Sichtbarkeit: `pub(crate)`, Zugriff über Getter)
        - `app_name: String` (Sichtbarkeit: `pub`)
        - `replaces_id: Option<NotificationId>` (Sichtbarkeit: `pub`)
        - `app_icon: Option<String>` (Pfad oder Name des Icons, Sichtbarkeit: `pub`)
        - `summary: String` (Kurze Zusammenfassung, Sichtbarkeit: `pub`)
        - `body: Option<String>` (Detaillierter Text, Sichtbarkeit: `pub`)
        - `actions: Vec<NotificationAction>` (Sichtbarkeit: `pub`)
        - `hints: std::collections::HashMap<String, zbus::zvariant::OwnedValue>` (Spezifische Hinweise für den Notification-Server, Sichtbarkeit: `pub`)
        - `expire_timeout_ms: i32` (-1 für Server-Default, 0 für nie, >0 für Millisekunden, Sichtbarkeit: `pub`)
        - `urgency: NotificationUrgency` (Sichtbarkeit: `pub`)
        - `timestamp: core::types::DateTime<core::types::Utc>` (Sichtbarkeit: `pub(crate)`)
        - `is_displayed: std::sync::atomic::AtomicBool` (Interner Status, ob die Benachrichtigung aktuell angezeigt wird)
    - **Initialwerte:** `id` wird generiert, `timestamp` ist aktuell, `is_displayed` ist `false`.
    - **Invarianten:** `summary` darf nicht leer sein. `id` ist unveränderlich.
    - **Methoden (Beispiele):**
        - `pub fn id(&self) -> NotificationId`
        - `pub(crate) fn mark_as_displayed(&self)`
        - `pub(crate) fn mark_as_closed(&self)`
- **`CloseReason` Enum (Wertobjekt):**
    
    - Gibt den Grund an, warum eine Benachrichtigung geschlossen wurde.
    - **Datei:** `domain/src/user_centric_services/types.rs`
    - **Varianten:** `Expired`, `DismissedByUser`, `ClosedByApp`, `ServiceShutdown`, `Unknown`
    - **Ableitungen:** `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `serde::Serialize`, `serde::Deserialize`.
- **`NotificationError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/user_centric_services/notification_error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum NotificationError {
            #
            NotificationNotFound(super::types::NotificationId), // Pfad anpassen
        
            #[error("Invalid notification format or content: {0}")]
            InvalidNotificationFormat(String),
        
            #[error("Failed to invoke action '{action_key}' for notification {notification_id}: {reason}")]
            ActionInvokeFailed {
                notification_id: super::types::NotificationId, // Pfad anpassen
                action_key: String,
                reason: String,
            },
        
            #[error("Notification service is unavailable or failed: {0}")]
            ServiceUnavailable(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn post_notification(&self, app_name: String, replaces_id: Option<NotificationId>, app_icon: Option<String>, summary: String, body: Option<String>, actions: Vec<NotificationAction>, hints: std::collections::HashMap<String, zbus::zvariant::OwnedValue>, expire_timeout_ms: i32, urgency: NotificationUrgency) -> Result<NotificationId, NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Sendet eine neue Benachrichtigung oder aktualisiert eine existierende.
    - **Nachbedingungen:** Bei Erfolg wird `NotificationPostedEvent` publiziert. Die Benachrichtigung wird (versucht) dem Benutzer angezeigt.
- `async fn get_active_notifications(&self) -> Result<Vec<Notification>, NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft eine Liste aller aktuell aktiven (angezeigten) Benachrichtigungen ab.
- `async fn close_notification(&self, id: NotificationId, reason: CloseReason) -> Result<(), NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Schließt eine spezifische Benachrichtigung.
    - **Vorbedingungen:** Benachrichtigung mit `id` muss existieren und aktiv sein.
    - **Nachbedingungen:** `NotificationClosedEvent` wird publiziert.
- `async fn invoke_action(&self, id: NotificationId, action_key: String) -> Result<(), NotificationError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Löst eine mit einer Benachrichtigung verbundene Aktion aus.
    - **Vorbedingungen:** Benachrichtigung mit `id` und Aktion mit `action_key` müssen existieren.
    - **Nachbedingungen:** `NotificationActionInvokedEvent` wird publiziert.

##### Publizierte Events:

- **`NotificationPostedEvent`:**
    - **Datei:** `domain/src/user_centric_services/events.rs`
    - **Struktur:** `pub struct NotificationPostedEvent { pub notification: Notification; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::NotificationPostedEvent`
- **`NotificationClosedEvent`:**
    - **Struktur:** `pub struct NotificationClosedEvent { pub id: NotificationId, pub reason: CloseReason; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::NotificationClosedEvent`
- **`NotificationActionInvokedEvent`:**
    - **Struktur:** `pub struct NotificationActionInvokedEvent { pub id: NotificationId, pub action_key: String; }`
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::user_centric_services::NotificationActionInvokedEvent`
- **Typische Publisher:** Die Implementierung von `NotificationService`.
- **Typische Subscriber:** `ui::notifications_frontend` oder `ui::shell` (UI-Schicht), `system::dbus::notification_server` (Systemschicht, falls NovaDE als Server agiert und die Events intern weiterleitet).

#### 2.1.5. `domain::global_settings_and_state_management::GlobalSettingsService` Trait

Verwaltet globale Desktop-Einstellungen und deren Persistenz.

- **Datei:** `domain/src/global_settings/service.rs`
- Dieser Dienst bietet eine strukturierte Methode zur Verwaltung einer Vielzahl von Desktop-Einstellungen. Der Persistenzmechanismus (z.B. eine Implementierung von `domain::settings_persistence_iface`, wie in der Kernschichtspezifikation erwähnt) ist hier von entscheidender Bedeutung.

##### Zugehörige Datenstrukturen und Typen:

- **`SettingPath` Typalias:**
    
    - **Datei:** `domain/src/global_settings/types.rs`
    - **Definition:** `pub type SettingPath = String;` (z.B. "desktop.background.color", "keyboard.layouts.0.name")
- **`SettingValue` Enum (Wertobjekt):**
    
    - Repräsentiert den Wert einer Einstellung. Kann verschiedene primitive Typen und verschachtelte Strukturen annehmen.
    - **Datei:** `domain/src/global_settings/value.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        #[serde(untagged)] // Für flexible (De-)Serialisierung
        pub enum SettingValue {
            String(String),
            Integer(i64),
            Float(f64),
            Boolean(bool),
            List(Vec<SettingValue>),
            Map(std::collections::HashMap<String, SettingValue>),
            Null, // Um das Fehlen eines Wertes explizit darzustellen
        }
        // Implement From<T> for SettingValue für gängige Typen
        ```
        
    - Die Verwendung von `serde_json::Value` wäre eine Alternative für maximale Flexibilität, aber ein eigenes Enum bietet mehr Typsicherheit innerhalb der Domäne.
- **`GlobalDesktopSettings` Struct (Entität):**
    
    - Repräsentiert die Gesamtheit aller verwalteten Einstellungen.
    - **Datei:** `domain/src/global_settings/settings_struct.rs`
    - **Felder:**
        - `settings: std::collections::BTreeMap<SettingPath, SettingValue>` (BTreeMap für geordnete Iteration/Serialisierung, falls gewünscht)
    - **Sichtbarkeit:** `pub(crate)` für `settings`, Zugriff über Methoden.
    - **Methoden (Beispiele):**
        - `pub fn get(&self, path: &SettingPath) -> Option<&SettingValue>`
        - `pub fn set(&mut self, path: SettingPath, value: SettingValue) -> Option<SettingValue>` (gibt alten Wert zurück)
        - `pub fn remove(&mut self, path: &SettingPath) -> Option<SettingValue>`
- **`GlobalSettingsError` Enum (Fehlertyp):**
    
    - **Datei:** `domain/src/global_settings/error.rs`
    - **Definition:**
        
        Rust
        
        ```
        #
        pub enum GlobalSettingsError {
            #
            SettingNotFound(super::types::SettingPath), // Pfad anpassen
        
            #
            TypeMismatch {
                path: super::types::SettingPath, // Pfad anpassen
                expected_type: String,
                found_type: String,
            },
        
            #[error("Persistence error during settings operation")]
            PersistenceError(#[from] crate::core::errors::CoreError),
        
            #[error("Validation error for setting '{path}': {message}")]
            ValidationError {
                path: super::types::SettingPath, // Pfad anpassen
                message: String,
            },
        
            #[error("Invalid setting path format: {0}")]
            InvalidPathFormat(String),
        }
        ```
        

##### Methoden des Traits:

- `async fn load_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Lädt alle Einstellungen aus der Persistenzschicht.
    - **Nachbedingungen:** Gibt die geladenen `GlobalDesktopSettings` zurück oder einen Fehler.
- `async fn save_settings(&self, settings: &GlobalDesktopSettings) -> Result<(), GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Persistiert den gesamten Einstellungsbaum.
- `async fn get_setting(&self, path: &SettingPath) -> Result<Option<SettingValue>, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft den Wert einer spezifischen Einstellung ab.
- `async fn update_setting(&self, path: SettingPath, value: SettingValue) -> Result<Option<SettingValue /* old_value */>, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Aktualisiert eine spezifische Einstellung und persistiert die Änderung. Gibt den alten Wert zurück, falls vorhanden.
    - **Vorbedingungen:** `path` und `value` müssen validen Formaten entsprechen. Ggf. Validierung gegen ein Schema.
    - **Nachbedingungen:** Bei Erfolg wird `SettingChangedEvent` publiziert. Die Einstellung wird persistiert.
- `async fn get_all_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsError>`
    - **`noexcept`**: `false`
    - **Beschreibung:** Ruft eine Kopie aller aktuellen Einstellungen ab.

##### Publizierte Events:

- **`SettingChangedEvent`:**
    - **Datei:** `domain/src/global_settings/events.rs`
    - **Struktur:**
        
        Rust
        
        ```
        #
        pub struct SettingChangedEvent {
            pub path: SettingPath,
            pub old_value: Option<SettingValue>,
            pub new_value: SettingValue,
        }
        ```
        
    - **Eindeutiger Event-Name/Typ:** `nova_de::domain::global_settings::SettingChangedEvent`
- **Typische Publisher:** Die Implementierung von `GlobalSettingsService`.
- **Typische Subscriber:** `ui::control_center` (UI-Schicht), verschiedene Systemkomponenten, die auf Einstellungsänderungen reagieren müssen.

### 2.2. Datenstrukturen (Öffentlich)

Zusätzlich zu den direkt mit Service-APIs assoziierten Strukturen können weitere öffentliche Datenstrukturen existieren, die Zustände oder Konfigurationen repräsentieren und von mehreren Modulen oder Schichten verwendet werden.

- **Beispiele:** Viele der oben bereits definierten Strukturen fallen in diese Kategorie, z.B. `AppliedThemeState`, `Workspace`, `Notification`, `GlobalDesktopSettings`, `ThemingConfiguration`, `ApplicationIdentifier`, `AIInteractionContextData`, `AIConsent`, `SettingValue`.
- **Implementierungsrichtlinien:**
    - Jede Struktur/Enum in einer eigenen Datei innerhalb des relevanten Domänenmoduls (z.B. `domain/src/theming/state.rs`).
    - Ableitung von `Debug`, `Clone`, `PartialEq` ist Standard.
    - Ableitung von `serde::Serialize`, `serde::Deserialize` falls Persistenz oder IPC-Übertragung erforderlich ist.
    - Felder sollten `pub` sein, wenn sie direkt von extern gelesen werden sollen, oder `pub(crate)` mit öffentlichen Gettern/Settern für kontrollierten Zugriff.
    - Invarianten sollten durch Konstruktoren und Methoden sichergestellt werden.
    - Beziehungen zu anderen Entitäten/Wertobjekten sollten klar definiert sein (z.B. durch Komposition oder IDs).

### 2.3. Events (Domänenspezifisch)

Domänenspezifische Events signalisieren wichtige Zustandsänderungen innerhalb der Domänenschicht. Sie sind entscheidend für eine reaktive Architektur und lose Kopplung.

- **Beispiele:** Die bereits bei den Services definierten Events (`ThemeChangedEvent`, `WorkspaceEvent`, `NotificationPostedEvent`, `SettingChangedEvent`, `AIConsentChangedEvent`, etc.).
- **Struktur:**
    - Jedes Event sollte als eigene Struktur oder als Variante eines umfassenderen Event-Enums (pro Modul) definiert werden.
    - Die Event-Struktur enthält alle relevanten Daten zur Zustandsänderung.
    - Beispiel: `pub struct ActiveWorkspaceChanged { pub old_id: Option<WorkspaceId>, pub new_id: WorkspaceId }`
- **Eindeutiger Event-Name/Typ:** Eine klare Namenskonvention ist wichtig, z.B. `nova_de::domain::module::EventName`.
- **Publisher:** Die jeweilige Service-Implementierung, die die Zustandsänderung verursacht.
- **Subscriber:** Komponenten in der System- und UI-Schicht, die auf diese Änderungen reagieren müssen.
    - Beispiel: `ui::theming_gtk` abonniert `ThemeChangedEvent`.
    - Beispiel: `system::compositor` abonniert `ActiveWorkspaceChanged`.
- **Datei-Struktur:** Events können in einer `events.rs`-Datei innerhalb jedes Domänenmoduls definiert werden (z.B. `domain/src/theming/events.rs`).
- **Serialisierung:** Wenn Events über IPC-Grenzen (z.B. D-Bus) gesendet werden sollen, müssen sie `serde::Serialize` und `serde::Deserialize` implementieren.

### 2.4. Fehlertypen (Modulspezifisch)

Jedes Hauptmodul innerhalb der Domänenschicht sollte seine eigenen spezifischen Fehler-Enums definieren, um präzise Fehlerinformationen zu liefern.

- **Beispiele:** Die bereits bei den Services definierten Fehler (`ThemingError`, `WorkspaceManagerError`, `AIInteractionError`, `NotificationError`, `GlobalSettingsError`).
- **Implementierungsrichtlinien:**
    - Verwendung des `thiserror`-Crates für eine ergonomische Definition.
    - Jeder Fehlertyp in einer eigenen Datei (z.B. `domain/src/theming/error.rs`).
    - Fehlervarianten sollten klare und spezifische Fehlerzustände des Moduls beschreiben.
    - Wo sinnvoll, sollten Quellfehler (`#[source]`) oder Konvertierungen (`#[from]`) für Fehler aus der Kernschicht oder anderen Abhängigkeiten bereitgestellt werden, um die Fehlerkette zu erhalten.
- **Verwendung:** Domänenservices geben diese spezifischen Fehler in ihren `Result`-Typen zurück. Die System- und UI-Schicht behandeln diese Fehler oder leiten sie ggf. weiter, eventuell gewrappt in ihre eigenen, allgemeineren Fehlertypen.

---

## 3. Schicht: Systemschicht (System Layer)

Die Systemschicht stellt der UI-Schicht systemnahe Dienste und Ereignisse zur Verfügung und setzt deren Befehle technisch um. Sie interagiert direkt mit dem Betriebssystem, der Hardware (über Abstraktionen wie DRM/GBM, libinput), Systemdiensten (via D-Bus, PipeWire) und dem Wayland-Compositor-Backend (Smithay). Sie konsumiert APIs der Domänenschicht, um Geschäftslogik anzuwenden und Zustände abzufragen.

Diese Schicht ist die Brücke zwischen der abstrakten Domänenlogik von NovaDE und den konkreten Realitäten des Betriebssystems und der Hardware. Hier finden viele der FFI-Aufrufe, IPC-Kommunikation und Interaktionen auf niedriger Ebene statt. Smithay wird intensiv für Wayland-bezogene Komponenten genutzt. Eine robuste Fehlerbehandlung ist entscheidend, um Fehler von niedriger Ebene in abstraktere `SystemError`-Typen oder spezifische Unterfehler zu übersetzen. Die asynchrone Natur vieler Operationen (IPC, Hardwareinteraktion) ist hier kritisch.

### 3.1. Modul: `system::compositor` (Smithay-basierter Wayland Compositor Kern)

Dieses Modul ist das Herzstück der grafischen Darstellung und Fensterverwaltung in NovaDE. Es basiert auf der Smithay-Bibliothek und implementiert die Kernfunktionalitäten eines Wayland-Compositors.

- **Abhängigkeiten:** `smithay`, `wayland-server`, `wayland-protocols`, `calloop`, `input` (libinput-rs), `drm-rs`, `gbm-rs`, `libloading` (für EGL).
- **Verzeichnis:** `system/src/compositor/`

#### 3.1.1. `NovaDeSystemState` Struktur (Hauptzustand für Smithay Handler)

Diese Struktur dient als zentraler Datenhalter für den Event-Loop und implementiert die verschiedenen Handler-Traits von Smithay.

- **Datei:** `system/src/state.rs` (oder `system/src/compositor/state.rs`, je nach Gesamtstruktur)
- **Felder (Beispiele, basierend auf typischer Smithay-Nutzung und NovaDE-Anforderungen):**
    
    Rust
    
    ```
    use smithay::{
        backend::{
            allocator::dmabuf::DmabufState, // [50]
            drm::DrmDeviceFd, // [19, 24, 56, 62, 69, 93, 305, 388]
            egl::{EGLContext, EGLDisplay}, // [16, 26, 62, 65, 94, 97]
            renderer::{
                gles::GlesRenderer, // [16, 40, 62, 94, 97]
                multigpu::{gbm::GbmGlesBackend, GpuManager}, // [24, 28, 40, 62, 97, 145, 222]
            },
            libinput::LibinputInputBackend, // [23, 75, 382]
            udev::{UdevBackend, UdevEvent}, // [37]
            session::{Session, Signal as SessionSignal, logind::LogindSession}, // [362, 391]
            input::Seat,
        },
        desktop::{Space, Window, PopupManager, layer_map_for_output, LayerSurface}, // [67]
        input::{SeatState, SeatHandler, pointer::PointerHandle, keyboard::KeyboardHandle, touch::TouchHandle, SeatName},
        output::Output,
        reexports::{
            calloop::{EventLoop, LoopHandle, LoopSignal},
            wayland_server::{Display, DisplayHandle, Client,backend::GlobalId},
            wayland_protocols::xdg::shell::server::xdg_wm_base,
        },
        utils::{Clock, Logical, Point, Rectangle, Serial, Transform, SERIAL_COUNTER},
        wayland::{
            buffer::BufferHandler, // [353]
            compositor::{CompositorState, CompositorHandler, CompositorClientState, SurfaceData}, // [17, 87, 99]
            dmabuf::DmabufHandler, // [50]
            output::{OutputHandler, OutputManagerState}, // [49, 147, 280, 370, 372, 392]
            presentation::PresentationState, // [274]
            seat::WaylandSeatData,
            shell::{
                xdg::{XdgShellState, XdgShellHandler, XdgShellSurfaceUserData, XdgWmBaseUserData, decoration::XdgDecorationState}, // [13, 31, 54, 81, 167, 176, 191, 195, 199, 204, 207, 208, 215, 223, 224, 225, 227, 317, 323, 336, 349, 350, 393, 394, 395, 396, 397, 398, 399, 400]
                wlr_layer::{WlrLayerShellState, LayerShellHandler, LayerSurfaceData}, // [11, 19, 20, 41, 48, 57, 66, 67, 71, 91, 92, 95, 160, 192, 254, 292, 307, 318, 347, 401, 402]
            },
            shm::{ShmState, ShmHandler}, // [141, 165]
            selection::{
                data_device::{DataDeviceState, DataDeviceHandler}, // [11, 12, 19, 35, 44, 45, 73, 80, 141, 216, 403, 404, 405, 406]
                primary_selection::{PrimarySelectionState, PrimarySelectionHandler}, // [30, 120]
            },
            input_method::{InputMethodManagerState, InputMethodHandler, InputMethodKeyboardUserData}, // [11, 34, 35, 117, 253, 264, 268, 407, 408, 409, 410, 411]
            text_input::{TextInputManagerState, TextInputHandler, TextInputUserData}, // [86, 112, 328, 412, 413, 414]
            viewporter::ViewporterState, // [371]
            xdg_activation::XdgActivationState, // [303]
            foreign_toplevel::ForeignToplevelState, // [335]
            idle_notify::IdleNotifierState, // [58]
            explicit_synchronization::ExplicitSyncState, // [415]
            drm_lease::DrmLeaseState, // [107]
            server_decoration::ServerDecorationManagerState, // [297]
        },
        xwayland::{XWayland, XWaylandEvent, XWaylandClientData, xwm::{X11Wm, XwmHandler, X11Surface}}, // [11, 14, 19, 21, 32, 35, 36, 41, 43, 52, 60, 76, 78, 85, 117, 141, 219, 270, 310, 329, 339, 361, 363, 377, 416, 417]
    };
    use std::{collections::HashMap, sync::{Arc, Mutex}};
    use crate::system::dbus::DBusConnectionManager; // Für D-Bus Interaktionen
    use crate::system::audio::PipeWireClientService; // Für PipeWire
    use crate::domain::services::DomainLayerServices; // Wrapper für Domain-Service-Handles
    
    pub struct NovaDeSystemState {
        pub display_handle: DisplayHandle,
        pub event_loop_handle: LoopHandle<'static, Self>,
        pub loop_signal: LoopSignal,
        pub clock: Clock, // Für Timings, Animationen
    
        // Core Wayland protocol states
        pub compositor_state: Arc<Mutex<CompositorState>>,
        pub shm_state: Arc<Mutex<ShmState>>,
        pub output_manager_state: Arc<Mutex<OutputManagerState>>,
        pub seat_state: Arc<Mutex<SeatState<Self>>>,
        pub data_device_state: Arc<Mutex<DataDeviceState>>,
        pub primary_selection_state: Arc<Mutex<PrimarySelectionState>>,
    
        // Shells
        pub xdg_shell_state: Arc<Mutex<XdgShellState>>,
        pub layer_shell_state: Arc<Mutex<WlrLayerShellState>>,
        pub xdg_decoration_state: Arc<Mutex<XdgDecorationState>>,
        // Optional: xwayland_shell_state if using Smithay's xwayland shell helpers
    
        // Input related
        pub text_input_manager_state: Arc<Mutex<TextInputManagerState>>,
        pub input_method_manager_state: Arc<Mutex<InputMethodManagerState>>,
    
        // Optional Wayland protocol states
        pub presentation_time_state: Option<Arc<Mutex<PresentationState>>>,
        pub viewporter_state: Option<Arc<Mutex<ViewporterState>>>,
        pub xdg_activation_state: Option<Arc<Mutex<XdgActivationState>>>,
        pub foreign_toplevel_state: Option<Arc<Mutex<ForeignToplevelState>>>,
        pub idle_notifier_state: Option<Arc<Mutex<IdleNotifierState<Self>>>>,
        pub explicit_sync_state: Option<Arc<Mutex<ExplicitSyncState>>>,
        pub drm_lease_state: Option<Arc<Mutex<DrmLeaseState>>>,
        pub server_decoration_manager_state: Option<Arc<Mutex<ServerDecorationManagerState>>>,
    
        // Backend specific states
        pub udev_backend_data: Option<UdevBackendData>, // If using udev backend
        pub winit_data: Option<WinitData>, // If using winit backend
        pub x11_data: Option<X11BackendData>, // If using X11 backend (as a client)
    
        // XWayland
        pub xwayland: Option<XWayland>,
        pub xwm: Option<Arc<Mutex<X11Wm>>>, // X11 Window Manager for XWayland
        pub xwayland_client_data_id: Option<u32>, // To identify XWayland's client
    
        // Desktop management
        pub space: Arc<Mutex<Space<WindowElement>>>, // Manages windows and outputs
        pub popup_manager: Arc<Mutex<PopupManager>>,
    
        // Input devices
        pub seat_name: String,
        pub seat: Seat<Self>,
        pub pointer: PointerHandle<Self>,
        pub keyboard: KeyboardHandle<Self>,
        pub touch: Option<TouchHandle<Self>>, // Touch ist optional
    
        // Domain Layer Access
        pub domain_services: Arc<DomainLayerServices>,
    
        // System Services Clients
        pub dbus_conn_manager: Arc<DBusConnectionManager>,
        pub pipewire_service: Arc<PipeWireClientService>,
    
        // Internal state
        pub running: std::sync::atomic::AtomicBool,
        pub active_workspace_id: Option<crate::domain::workspaces::WorkspaceId>,
        // Weitere anwendungsspezifische Zustände
    }
    
    // Hilfsstrukturen für Backend-spezifische Daten
    pub struct UdevBackendData {
        pub session: LogindSession, // Oder eine andere Session-Implementierung
        pub gpus: GpuManager<GbmGlesBackend<GlesRenderer, DrmDeviceFd>>,
        pub primary_gpu: DrmDeviceFd,
        pub udev_backend: UdevBackend,
        // Weitere DRM/GBM/EGL spezifische Zustände
    }
    pub struct WinitData { /*... relevante Winit-Felder... */ }
    pub struct X11BackendData { /*... relevante X11-Backend-Felder... */ }
    
    // WindowElement (Beispiel, muss an NovaDE angepasst werden)
    #
    pub enum WindowElement {
        Wayland(Window), // Smithay's Window für XDG Toplevels
        X11(X11Surface), // Smithay's X11Surface
        Layer(LayerSurface), // Smithay's LayerSurface
    }
    // Implementiere notwendige Traits für WindowElement (z.B. RenderElement, SpaceElement)
    ```
    
    - Der Compositor ist das komplexeste Element der Systemschicht. Smithay 8 stellt Bausteine bereit, aber die eigentliche Fensterverwaltungslogik (Tiling, Stacking, Fokusrichtlinien basierend auf Domänenregeln) muss hier implementiert werden. Die `AnvilState` aus Smithays Anvil-Compositor dient als gute Referenz für die Strukturierung von `NovaDeSystemState` und die Integration verschiedener Smithay-Handler.

#### 3.1.2. Initialisierung (`system::compositor::initialize_compositor`)

- **Datei:** `system/src/compositor/init.rs`
- **Signatur:** `pub async fn initialize_compositor(domain_services: Arc<DomainLayerServices>, event_loop_handle: LoopHandle<'static, NovaDeSystemState>, loop_signal: LoopSignal) -> Result<NovaDeSystemState, SystemError>`
- **Schritte:**
    1. Erstelle `DisplayHandle` aus dem globalen `Display` Objekt (Annahme: `Display` wird im `main` erstellt und `DisplayHandle` wird hierher übergeben oder `NovaDeSystemState` wird direkt im `main` erstellt).
    2. Initialisiere Smithay-Backends (z.B. `LibinputInputBackend` 23, `UdevBackend` 37, DRM/GBM/EGL 16). Die Wahl des Backends (DRM/libseat für Bare-Metal, Winit/X11 für verschachtelte Entwicklung) ist eine wichtige architektonische Entscheidung. Die Implementierung sollte dies abstrahieren.
    3. Initialisiere alle relevanten Wayland-Globale unter Verwendung ihrer

# **UI-Schicht: Ultra-Feinspezifikation und Implementierungsleitfaden**

## **Modul: ui::shell::PanelWidget (Haupt-Panel-Implementierung)**

### **1. Modul-Identifikation und Zweck**

- **Modul-ID:** `ui::shell::PanelWidget`
- **Abhängigkeiten:**
    - `novade-core` (insbesondere `core::types` für geometrische Primitive und Farben, `core::errors` als Basis)
    - `novade-domain` (insbesondere `domain::theming` für Design-Tokens, `domain::global_settings_and_state_management` für Konfigurationen)
    - `gtk4-rs` (GTK4 Rust Bindings)
    - `gtk4-layer-shell` (für Wayland Layer Shell Integration)
    - `cairo-rs` (für benutzerdefiniertes Zeichnen, falls CSS nicht ausreicht)
    - `zbus` (für D-Bus-Kommunikation, falls das Panel direkt mit Systemdiensten interagiert, was aber eher von Submodulen gehandhabt wird)
    - `once_cell` (für statische Initialisierungen)
    - `thiserror` (für Fehlerdefinitionen innerhalb dieses Moduls, falls spezifisch)
    - `tracing` (für Logging)
- **Zweck:** Das `PanelWidget` ist die zentrale Komponente der `ui::shell`, die als primäre(s) Kontroll- und Systemleiste(n) der NovaDE dient. Es ist verantwortlich für die Bereitstellung einer oder mehrerer horizontaler Leisten am Bildschirmrand, die Aufnahme, Anordnung und Verwaltung verschiedener modularer Panel-Elemente (Submodule), die Implementierung grundlegender Panel-Eigenschaften (Höhe, Transparenz, visueller "Leuchtakzent") und die korrekte Integration in Wayland-Compositors mittels des `gtk4-layer-shell`-Protokolls.

### **2. Visuelles Design und Theming (Spezifikation für Manus AI)**

- **Positionierung:**
    - Kann am oberen (`Top`) oder unteren (`Bottom`) Bildschirmrand verankert werden.
    - Die Positionierung wird durch die Eigenschaft `position` (Typ: `PanelPosition`, siehe unten) gesteuert.
    - Standardwert: `PanelPosition::Top`.
- **Abmessungen:**
    - **Höhe (`panel-height`):** Konfigurierbare Höhe in Pixeln.
        - Datentyp: `i32`.
        - Gültiger Bereich: Minimum 24px, Maximum 128px.
        - Standardwert: 36px.
        - Die Höhe wird über die GObject-Eigenschaft `panel-height` gesetzt und beeinflusst die `default-height` des zugrundeliegenden `gtk::ApplicationWindow` sowie die `height-request` der internen `main_box`.
    - **Breite:** Das Panel erstreckt sich standardmäßig über die gesamte Breite des zugewiesenen Monitors/Outputs. Dies wird durch die Ankerung über `gtk4-layer-shell` erreicht (`Edge::Left` und `Edge::Right` auf `true` setzen).
- **Erscheinungsbild:**
    - **Hintergrund:**
        - Farbe und Stil werden primär über CSS gesteuert.
        - Das Panel kann optional einen transluzenten Hintergrund haben.
        - Die Eigenschaft `transparency-enabled` (Typ: `bool`, Standard: `false`) steuert dies.
        - Implementierung der Transparenz:
            1. Das GDK Visual des `PanelWidget` (als `gtk::Window`) muss auf ein RGBA-Visual gesetzt werden, falls Transparenz aktiviert ist: `self.set_visual(self.display().rgba_visual().as_ref())`.
            2. Der CSS-Hintergrund muss eine RGBA-Farbe verwenden (z.B. `background-color: rgba(0, 0, 0, 0.5);`).
            3. Für Wayland-Compositors, die transparente Oberflächen unterstützen, muss die `gtk4-layer-shell`-Integration sicherstellen, dass der Compositor die Transparenz korrekt handhabt. Das Setzen von `surface.set_opaque_region(None)` kann notwendig sein.
    - **"Leuchtakzent"-Effekt:**
        - Ein subtiler Leuchteffekt entlang einer Kante des Panels (typischerweise die dem Bildschirmzentrum zugewandte Kante).
        - Gesteuert durch Eigenschaften:
            - `leuchtakzent-color` (Typ: `Option<gdk::RGBA>`, Standard: `None`). Farbe des Akzents.
            - `leuchtakzent-intensity` (Typ: `f64`, Bereich: 0.0 bis 1.0, Standard: 0.5). Intensität/Opazität des Akzents.
        - **Implementierung (priorisierte Reihenfolge):**
            1. **CSS `box-shadow`:** Versuche, den Effekt mit CSS `box-shadow` zu erzielen (z.B. `box-shadow: 0px 2px 5px 0px var(--leuchtakzent-color);` angepasst für die Kante). Die Farbe `--leuchtakzent-color` wird dynamisch über einen `CssProvider` basierend auf `leuchtakzent-color` und `leuchtakzent-intensity` gesetzt.
            2. **Benutzerdefiniertes Zeichnen (Cairo auf `gtk::DrawingArea`):** Falls CSS nicht ausreicht oder Performance-Probleme verursacht:
                - Eine `gtk::DrawingArea` wird als unterste Ebene im `PanelWidget` platziert (oder das `PanelWidget` zeichnet seinen Hintergrund selbst, was komplexer ist).
                - Im `draw`-Signal-Handler der `DrawingArea`:
                    - Hintergrund des Panels zeichnen (unter Berücksichtigung von `transparency-enabled`).
                    - Den Leuchtakzent-Pfad definieren (Linie oder schmales Rechteck entlang der relevanten Kante).
                    - Akzentfarbe und -intensität aus den GObject-Eigenschaften abrufen.
                    - `cr.set_source_rgba()` mit der `leuchtakzent-color` (modifiziert durch `leuchtakzent-intensity` für die Alpha-Komponente) verwenden.
                    - Für einen weichen Effekt: `cairo::LinearGradient` verwenden, der von der Akzentfarbe zu einer transparenten Version derselben Farbe oder zur Hintergrundfarbe übergeht.
                    - `self.queue_draw()` aufrufen, wenn sich `leuchtakzent-color` oder `leuchtakzent-intensity` ändern.
- **CSS-Styling:**
    - **CSS-Knotenname:** Das `PanelWidget` (als `GtkApplicationWindow`) hat standardmäßig den CSS-Knoten `window`. Das interne Haupt-Layout-Widget (z.B. `main_box` vom Typ `gtk::Box`) hat den Knoten `box`.
    - Dem `PanelWidget` wird der CSS-Name "panelwidget" zugewiesen (`klass.set_css_name("panelwidget");`).
    - **CSS-Klassen (dynamisch und statisch):**
        - `.nova-panel` (statisch): Allgemeine Klasse für das Panel.
        - `.panel-top` / `.panel-bottom` (dynamisch): Je nach Wert der `position`-Eigenschaft.
        - `.transparent-panel` (dynamisch): Wenn `transparency-enabled` `true` ist.
        - Diese Klassen werden dem `PanelWidget` über `gtk::Widget::add_css_class()` / `gtk::Widget::remove_css_class()` zugewiesen, wenn sich die entsprechenden Eigenschaften ändern.
    - **CSS-Provider:** Die Anwendung von CSS erfolgt über einen globalen `gtk::CssProvider`, der durch ein übergeordnetes Theming-Modul (z.B. `ui::theming_gtk`) verwaltet wird. Das `PanelWidget` reagiert auf `ThemeChangedEvents` (siehe Abschnitt 2.6 Signale), um Stiländerungen zu übernehmen.

### **3. Datenstrukturen, Eigenschaften und Zustand (Spezifikation für Manus AI)**

Das `PanelWidget` wird als benutzerdefiniertes GObject-Widget implementiert, das von `gtk::ApplicationWindow` erbt.

#### **3.1. Datei: `src/ui/shell/panel_widget/mod.rs` (Öffentliche API)**

Rust

```
use gtk::glib;
use gtk::subclass::prelude::*;
use gtk::{gio, ApplicationWindow, Widget}; // Hinzugefügt für @extends und @implements
use std::cell::{Cell, RefCell}; // Für interne Veränderlichkeit in imp.rs benötigt, hier nicht direkt

mod imp; // Private Implementierung

glib::wrapper! {
    pub struct PanelWidget(ObjectSubclass<imp::PanelWidget>)
        @extends gtk::ApplicationWindow, gtk::Window, gtk::Widget,
        @implements gio::ActionGroup, gio::ActionMap, gtk::Accessible, gtk::Buildable,
                    gtk::ConstraintTarget, gtk::Native, gtk::Root, gtk::ShortcutManager;
}

impl PanelWidget {
    pub fn new(app: &gtk::Application) -> Self {
        glib::Object::builder::<Self>()
            .property("application", app)
            .build()
    }

    // ### Öffentliche Methoden zur Modulverwaltung ###

    /// Fügt ein Widget-Modul dem Panel hinzu.
    ///
    /// # Parameter
    /// * `module`: Das Widget, das als Modul hinzugefügt werden soll.
    /// * `position`: Die Position im Panel (Start, Center, End).
    /// * `order`: Die Reihenfolge des Moduls innerhalb seiner Position (kleinere Werte zuerst).
    ///
    /// `noexcept` (nicht direkt in Rust, aber Methode soll nicht paniken)
    pub fn add_module(&self, module: &impl glib::IsA<gtk::Widget>, position: imp::ModulePosition, order: i32) {
        self.imp().add_module_ordered(module, position, order);
    }

    /// Entfernt ein zuvor hinzugefügtes Modul aus dem Panel.
    ///
    /// # Parameter
    /// * `module`: Das zu entfernende Widget-Modul.
    ///
    /// `noexcept`
    pub fn remove_module(&self, module: &impl glib::IsA<gtk::Widget>) {
        self.imp().remove_module_internal(module);
    }

    // ### Öffentliche Methoden für Eigenschaften (optional, wenn GObject Properties bevorzugt) ###
    // Beispiel, falls direkte Methodenaufrufe gewünscht sind, ansonsten über Properties

    pub fn set_panel_position(&self, position: imp::PanelPosition) {
        self.set_property("position", position.to_value());
    }

    pub fn get_panel_position(&self) -> imp::PanelPosition {
        self.property("position")
    }

    pub fn set_panel_height(&self, height: i32) {
        self.set_property("panel-height", height);
    }

    pub fn get_panel_height(&self) -> i32 {
        self.property("panel-height")
    }

    pub fn set_transparency_enabled(&self, enabled: bool) {
        self.set_property("transparency-enabled", enabled);
    }

    pub fn is_transparency_enabled(&self) -> bool {
        self.property("transparency-enabled")
    }
}

// Enum Definitionen müssen hier oder in einem gemeinsamen Typmodul sein,
// damit sie in der öffentlichen API verwendet werden können.
// Für dieses Beispiel werden sie im imp-Modul deklariert und hier re-exportiert,
// oder sie sind in einem ui::shell::panel_widget::types Modul.
// pub use imp::{PanelPosition, ModulePosition}; // Falls sie in imp definiert sind

```

#### **3.2. Datei: `src/ui/shell/panel_widget/imp.rs` (Private GObject-Implementierung)**

Rust

```
use gtk::glib::{self, Properties, ParamSpec, Value, subclass::Signal};
use gtk::subclass::prelude::*;
use gtk::{CompositeTemplate, Align, Orientation as GtkOrientation, Box as GtkBox, ApplicationWindow, Widget}; // Widget hinzugefügt
use gdk::RGBA; // Für leuchtakzent-color
use std::cell::{Cell, RefCell};
use std::collections::{BTreeMap, HashMap}; // BTreeMap für geordnete Module
use once_cell::sync::Lazy;

// ### Enums für Panel-Konfiguration ###

#[derive(Debug, Default, Clone, Copy, PartialEq, Eq, glib::Enum)]
#[enum_type(name = "NovaDEPanelPosition")]
pub enum PanelPosition {
    #[default]
    Top,
    Bottom,
}

// Benötigt `glib:: القيمه` für Properties
impl From<PanelPosition> for Value {
    fn from(position: PanelPosition) -> Self {
        position.to_value()
    }
}
impl From<Value> for PanelPosition {
    fn from(value: Value) -> Self {
        glib::EnumClass::new(PanelPosition::static_type())
            .expect("PanelPosition EnumClass not found")
            .value(value.get_enum().expect("Value is not an enum"))
            .expect("Invalid PanelPosition enum value")
            .downcast::<PanelPosition>()
            .expect("Failed to downcast to PanelPosition")
            .get()
    }
}


#[derive(Debug, Default, Clone, Copy, PartialEq, Eq, glib::Enum)]
#[enum_type(name = "NovaDEModulePosition")]
pub enum ModulePosition {
    #[default]
    Start,
    Center,
    End,
}

// Hilfsstruktur für geordnete Module
#[derive(Debug, Clone)]
struct OrderedModule {
    widget: Widget,
    order: i32,
}

// ### GObject Properties Definition ###
static PANEL_PROPERTIES: Lazy<Vec<ParamSpec>> = Lazy::new(|| {
    vec![
        ParamSpec::new_enum(
            "position",
            "Position",
            "Bildschirmkante, an der das Panel verankert ist (Oben, Unten).",
            PanelPosition::static_type(),
            PanelPosition::default() as i32, // Standardwert
            glib::ParamFlags::READWRITE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        ParamSpec::new_int(
            "panel-height",
            "Panel Height",
            "Höhe des Panels in Pixeln (Min: 24, Max: 128).",
            24, // Min
            128, // Max
            36, // Standardwert
            glib::ParamFlags::READWRITE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        ParamSpec::new_boolean(
            "transparency-enabled",
            "Transparency Enabled",
            "Gibt an, ob Transparenzeffekte für das Panel aktiv sind.",
            false, // Standardwert
            glib::ParamFlags::READWRITE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        ParamSpec::new_boxed(
            "leuchtakzent-color",
            "Leuchtakzent Color",
            "Farbe des Leuchtakzents (gdk::RGBA). Wird typischerweise vom Theming-System aktualisiert.",
            RGBA::static_type(), // Typ gdk::RGBA
            glib::ParamFlags::READWRITE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        ParamSpec::new_double(
            "leuchtakzent-intensity",
            "Leuchtakzent Intensity",
            "Intensität/Opazität des Leuchtakzents (Bereich: 0.0 bis 1.0).",
            0.0, // Min
            1.0, // Max
            0.5, // Standardwert
            glib::ParamFlags::READWRITE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
    ]
});

// ### GObject Signale Definition ###
static PANEL_SIGNALS: Lazy<HashMap<String, Signal>> = Lazy::new(|| {
    let mut signals = HashMap::new();
    signals.insert(
        "module-layout-changed".to_string(),
        Signal::builder("module-layout-changed")
            .action() // Kein Rückgabewert, keine Parameter für dieses Signal
            .build(),
    );
    signals
});


#[derive(CompositeTemplate, Default)]
#[template(resource = "/org/nova_de/ui/shell/panel_widget.ui")] // Pfad zur UI-Datei
pub struct PanelWidget {
    #[template_child]
    pub main_box: TemplateChild<GtkBox>,
    #[template_child]
    pub start_box: TemplateChild<GtkBox>,
    #[template_child]
    pub center_box: TemplateChild<GtkBox>,
    #[template_child]
    pub end_box: TemplateChild<GtkBox>,

    // GObject Properties (als Felder in der Imp-Struktur)
    #[property(get, set, explicit_notify)]
    position: RefCell<PanelPosition>,
    #[property(get, set, explicit_notify)]
    panel_height: Cell<i32>,
    #[property(get, set, explicit_notify)]
    transparency_enabled: Cell<bool>,
    #[property(get, set, explicit_notify)]
    leuchtakzent_color: RefCell<Option<RGBA>>,
    #[property(get, set, explicit_notify)]
    leuchtakzent_intensity: Cell<f64>,

    // Interner Zustand für Modulverwaltung
    // Verwendet BTreeMap, um Module nach `order` und dann nach Einfügezeit (als Fallback für gleiche Order)
    // sortiert zu halten. Der innere Vec ist für den Fall, dass mehrere Widgets exakt dieselbe Order haben.
    modules_start: RefCell<BTreeMap<i32, Vec<Widget>>>,
    modules_center: RefCell<BTreeMap<i32, Vec<Widget>>>,
    modules_end: RefCell<BTreeMap<i32, Vec<Widget>>>,

    // Für benutzerdefiniertes Zeichnen des Leuchtakzents, falls CSS nicht ausreicht.
    // Wird im `constructed` initialisiert, falls benötigt.
    drawing_area_for_accent: RefCell<Option<gtk::DrawingArea>>,
}

#[glib::object_subclass]
impl ObjectSubclass for PanelWidget {
    const NAME: &'static str = "NovaDEPanelWidget";
    type Type = super::PanelWidget; // Der öffentliche Wrapper-Typ
    type ParentType = gtk::ApplicationWindow;

    fn class_init(klass: &mut Self::Class) {
        klass.bind_template();
        klass.install_properties(&PANEL_PROPERTIES);
        klass.install_signals(&PANEL_SIGNALS);
        klass.set_css_name("panelwidget"); // CSS-Name für das Widget
    }

    fn instance_init(obj: &glib::subclass::InitializingObject<Self>) {
        obj.init_template();
    }
}

#[glib::derived_properties]
impl ObjectImpl for PanelWidget {
    fn constructed(&self) {
        self.parent_constructed();
        let obj = self.obj(); // Der öffentliche Wrapper `super::PanelWidget`

        // Standardwerte für Properties setzen, falls nicht explizit beim Bauen angegeben
        // Diese Werte werden von den ParamSpec Defaults überschrieben, falls `glib::Object::builder()` verwendet wird.
        // Wenn direkt `PanelWidget::new()` aufgerufen wird, sind die Zell-Defaults (z.B. 0 für i32) aktiv.
        // Die ParamSpec Defaults sollten hier bevorzugt werden.
        // Die Initialisierung der RefCell/Cell-Felder mit den Property-Standardwerten erfolgt
        // implizit durch das GObject-Property-System, wenn das Objekt gebaut wird.
        // Manuelle Defaults hier sind nur nötig, wenn kein Property-System genutzt würde.

        // gtk4-layer-shell initialisieren
        obj.setup_layer_shell_internal();
        obj.update_module_layout_internal(); // Erstes Layout anwenden

        // CSS-Klassen basierend auf initialen Properties setzen
        self.update_css_classes();

        // Eventuell DrawingArea für Leuchtakzent initialisieren und verbinden, falls nötig
        // self.setup_drawing_area_for_accent();
    }

    // Implementierung der Properties Getters und Setters (automatisch durch #[property] und #[glib::derived_properties])
    // Die Logik für Property-Änderungen (z.B. LayerShell neu konfigurieren) kommt in die `set_property` Methode.

    fn set_property(&self, _id: usize, value: &Value, pspec: &ParamSpec) {
        match pspec.name() {
            "position" => {
                let new_position = value.get::<PanelPosition>().expect("Value must be PanelPosition");
                self.position.replace(new_position);
                self.obj().setup_layer_shell_internal(); // Layer Shell neu konfigurieren
                self.update_css_classes();
                self.obj().notify_position(); // Benachrichtigung für diese Eigenschaft
            }
            "panel-height" => {
                let new_height = value.get::<i32>().expect("Value must be i32");
                // Validierung gegen Min/Max aus ParamSpec (automatisch durch GObject, aber man könnte hier zusätzlich prüfen)
                self.panel_height.set(new_height);
                self.obj().set_default_height(new_height); // Fensterhöhe anpassen
                self.main_box.set_height_request(new_height);
                // Ggf. DrawingArea Höhe anpassen
                self.obj().queue_draw(); // Neuzeichnen anfordern
                self.obj().notify_panel_height();
            }
            "transparency-enabled" => {
                let enabled = value.get::<bool>().expect("Value must be bool");
                self.transparency_enabled.set(enabled);
                self.update_transparency_visual_internal();
                self.update_css_classes();
                self.obj().notify_transparency_enabled();
            }
            "leuchtakzent-color" => {
                // Option<RGBA> muss vorsichtig gehandhabt werden
                let color_opt = value.get::<Option<RGBA>>().expect("Value must be Option<RGBA> or None");
                self.leuchtakzent_color.replace(color_opt);
                self.obj().queue_draw(); // Neuzeichnen für Akzent
                self.obj().notify_leuchtakzent_color();
            }
            "leuchtakzent-intensity" => {
                let intensity = value.get::<f64>().expect("Value must be f64");
                self.leuchtakzent_intensity.set(intensity.clamp(0.0, 1.0)); // Sicherstellen, dass im Bereich
                self.obj().queue_draw();
                self.obj().notify_leuchtakzent_intensity();
            }
            _ => unimplemented!(),
        }
    }
}

impl WidgetImpl for PanelWidget {
    fn map(&self) {
        self.parent_map();
        // Sicherstellen, dass Layer Shell korrekt initialisiert ist, bevor das Fenster angezeigt wird
        self.obj().setup_layer_shell_internal();
    }

    fn size_allocate(&self, width: i32, height: i32, baseline: i32) {
        self.parent_size_allocate(width, height, baseline);
        // Ggf. Layout der internen Boxen hier anpassen oder DrawingArea Größe
        // Dies ist wichtig, wenn sich die Panel-Breite ändert (z.B. Multi-Monitor-Setup Wechsel)
    }

    // Falls benutzerdefiniertes Zeichnen für Akzent via DrawingArea:
    // fn snapshot(&self, snapshot: &gtk::Snapshot) {
    //     self.parent_snapshot(snapshot);
    //     // Hier könnte man den Akzent zeichnen, wenn er nicht Teil eines Kind-Widgets ist.
    // }
}

impl WindowImpl for PanelWidget {
    // Fenster-spezifische Implementierungen, z.B. Schließen-Verhalten
}

impl ApplicationWindowImpl for PanelWidget {}

// ### Interne Implementierungsmethoden für PanelWidget ###
impl PanelWidget {
    fn update_css_classes(&self) {
        let obj = self.obj();
        if *self.position.borrow() == PanelPosition::Top {
            obj.remove_css_class("panel-bottom");
            obj.add_css_class("panel-top");
        } else {
            obj.remove_css_class("panel-top");
            obj.add_css_class("panel-bottom");
        }

        if self.transparency_enabled.get() {
            obj.add_css_class("transparent-panel");
        } else {
            obj.remove_css_class("transparent-panel");
        }
    }

    fn update_transparency_visual_internal(&self) {
        let obj = self.obj();
        let visual = if self.transparency_enabled.get() {
            obj.display().rgba_visual()
        } else {
            // Zurück zum System-Standard-Visual (oder None, wenn das Fenster sowieso opak sein soll)
            obj.display().default_visual(&obj.surface().unwrap_or_else(|| obj.create_surface(obj.width(), obj.height(), None)))
        };
        obj.set_visual(visual.as_ref());
        obj.queue_draw(); // Neuzeichnen anfordern, da sich das Visual geändert hat
    }

    // Wird vom öffentlichen add_module aufgerufen
    pub(super) fn add_module_ordered(&self, module: &impl glib::IsA<gtk::Widget>, position: ModulePosition, order: i32) {
        let module_widget = module.clone().upcast::<Widget>();
        let modules_map = match position {
            ModulePosition::Start => &self.modules_start,
            ModulePosition::Center => &self.modules_center,
            ModulePosition::End => &self.modules_end,
        };

        let mut map_guard = modules_map.borrow_mut();
        map_guard.entry(order).or_default().push(module_widget);

        self.obj().update_module_layout_internal();
        self.obj().emit_by_name::<()>("module-layout-changed", &[]);
    }

    // Wird vom öffentlichen remove_module aufgerufen
    pub(super) fn remove_module_internal(&self, module_to_remove: &impl glib::IsA<gtk::Widget>) {
        let widget_ptr_to_remove = module_to_remove.as_ref().to_glib_none().0;
        let mut changed = false;

        for modules_map_refcell in [&self.modules_start, &self.modules_center, &self.modules_end] {
            let mut map_guard = modules_map_refcell.borrow_mut();
            for (_order, widgets_in_order) in map_guard.iter_mut() {
                let initial_len = widgets_in_order.len();
                widgets_in_order.retain(|m| m.to_glib_none().0 != widget_ptr_to_remove);
                if widgets_in_order.len() < initial_len {
                    changed = true;
                    break; // Modul gefunden und entfernt
                }
            }
            if changed { break; }
        }

        if changed {
            // Physisches Entfernen aus den GtkBox-Containern
            if let Some(parent_box) = module_to_remove.as_ref().parent().and_then(|p| p.downcast::<GtkBox>().ok()){
                parent_box.remove(module_to_remove.as_ref());
            }
            self.obj().update_module_layout_internal();
            self.obj().emit_by_name::<()>("module-layout-changed", &[]);
        }
    }
}

// ### Öffentliche Methoden, die von der `imp`-Struktur für den Wrapper bereitgestellt werden ###
// Diese sind jetzt im `PanelWidget`-Block in `mod.rs` implementiert oder werden
// durch GObject-Properties gehandhabt. Die `setup_layer_shell_internal` und
// `update_module_layout_internal` sind die Kernmethoden, die der Wrapper aufruft.
impl super::PanelWidget { // Bezieht sich auf den öffentlichen Wrapper
    fn setup_layer_shell_internal(&self) {
        let imp = self.imp();
        gtk_layer_shell::init_for_window(self);
        gtk_layer_shell::set_layer(self, gtk_layer_shell::Layer::Top);
        // Panels benötigen i.d.R. keinen direkten Fokus, außer ihre Kindelemente
        gtk_layer_shell::set_keyboard_mode(self, gtk_layer_shell::KeyboardMode::OnDemand); // Oder None, wenn Module Fokus explizit anfordern
        gtk_layer_shell::auto_exclusive_zone_enable(self); // Platz reservieren
        gtk_layer_shell::set_namespace(self, "NovaDEPanel");

        let position_val = *imp.position.borrow();
        match position_val {
            PanelPosition::Top => {
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Top, true);
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Left, true);
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Right, true);
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Bottom, false);
            }
            PanelPosition::Bottom => {
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Bottom, true);
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Left, true);
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Right, true);
                gtk_layer_shell::set_anchor(self, gtk_layer_shell::Edge::Top, false);
            }
        }
        self.set_default_height(imp.panel_height.get());
        // Margins könnten hier auch gesetzt werden, falls gewünscht, z.B. um das Panel leicht vom Rand abzusetzen
        // gtk_layer_shell::set_margin(self, gtk_layer_shell::Edge::Top, 5);
        // gtk_layer_shell::set_margin(self, gtk_layer_shell::Edge::Left, 5);
        // gtk_layer_shell::set_margin(self, gtk_layer_shell::Edge::Right, 5);
    }

    fn update_module_layout_internal(&self) {
        let imp = self.imp();

        // Helferfunktion zum Leeren und Befüllen einer Box
        let repopulate_box = |target_box: &GtkBox, modules_map_refcell: &RefCell<BTreeMap<i32, Vec<Widget>>>| {
            while let Some(child) = target_box.first_child() {
                target_box.remove(&child);
            }
            let map_guard = modules_map_refcell.borrow();
            for (_order, widgets_in_order) in map_guard.iter() { // BTreeMap iteriert über Schlüssel (order) sortiert
                for widget in widgets_in_order {
                    target_box.append(widget);
                }
            }
        };

        repopulate_box(&imp.start_box, &imp.modules_start);
        repopulate_box(&imp.center_box, &imp.modules_center);
        repopulate_box(&imp.end_box, &imp.modules_end);
    }
}
```

#### **3.3. UI-Definitionsdatei: `resources/ui/shell/panel_widget.ui`**

XML

```
<?xml version="1.0" encoding="UTF-8"?>
<interface>
  <requires lib="gtk" version="4.0"/>
  <template class="NovaDEPanelWidget" parent="GtkApplicationWindow">
    <property name="title" translatable="yes">NovaDE Panel</property>
    <property name="decorated">false</property>
    <child>
      <object class="GtkBox" id="main_box">
        <property name="orientation">horizontal</property>
        <property name="halign">fill</property>
        <property name="valign">fill</property> <property name="spacing">6</property> <style>
          <class name="nova-panel-main-box"/>
        </style>
        <child>
          <object class="GtkBox" id="start_box">
            <property name="orientation">horizontal</property>
            <property name="halign">start</property>
            <property name="valign">center</property>
            <property name="spacing">6</property> <style>
              <class name="nova-panel-start-box"/>
            </style>
            </object>
        </child>
        <child>
          <object class="GtkBox" id="center_box">
            <property name="orientation">horizontal</property>
            <property name="halign">center</property>
            <property name="valign">center</property>
            <property name="hexpand">true</property> <property name="spacing">6</property>
            <style>
              <class name="nova-panel-center-box"/>
            </style>
            </object>
        </child>
        <child>
          <object class="GtkBox" id="end_box">
            <property name="orientation">horizontal</property>
            <property name="halign">end</property>
            <property name="valign">center</property>
            <property name="spacing">6</property>
            <style>
              <class name="nova-panel-end-box"/>
            </style>
            </object>
        </child>
      </object>
    </child>
  </template>
</interface>
```

#### **3.4. GObject-Eigenschaften (Zusammenfassung)**

|Eigenschaftsname|Rust-Feld (Imp)|Typ (Rust / GLib)|Zugriff|Standardwert|Beschreibung|
|:--|:--|:--|:--|:--|:--|
|`application`|(Parent-Eigenschaft)|`gtk::Application`|R/W/C|N/A|Die `gtk::Application`, zu der das Fenster gehört.|
|`position`|`position`|`PanelPosition` / Enum|R/W|`Top`|Bildschirmkante für das Panel (Oben, Unten).|
|`panel-height`|`panel_height`|`i32` / GINT|R/W|36|Höhe des Panels in Pixeln (Min: 24, Max: 128).|
|`transparency-enabled`|`transparency_enabled`|`bool` / GBOOLEAN|R/W|`false`|Aktiviert/Deaktiviert Transparenzeffekte.|
|`leuchtakzent-color`|`leuchtakzent_color`|`Option<gdk::RGBA>` / GBoxed|R/W|`None`|Farbe des Leuchtakzents.|
|`leuchtakzent-intensity`|`leuchtakzent_intensity`|`f64` / GDOUBLE|R/W|0.5|Intensität des Leuchtakzents (0.0-1.0).|

In Google Sheets exportieren

_R=Read, W=Write, C=Construct-Only_

#### **3.5. Interner Zustand (Felder in `PanelWidgetImp`)**

- `main_box: TemplateChild<GtkBox>`: Hauptcontainer für die drei Modulbereiche.
- `start_box: TemplateChild<GtkBox>`: Container für Module am Anfang (links bei LTR-Layout).
- `center_box: TemplateChild<GtkBox>`: Container für Module in der Mitte. `hexpand = true`.
- `end_box: TemplateChild<GtkBox>`: Container für Module am Ende (rechts bei LTR-Layout).
- `position: RefCell<PanelPosition>`: Speichert die aktuelle Panel-Position.
- `panel_height: Cell<i32>`: Speichert die aktuelle Panel-Höhe.
- `transparency_enabled: Cell<bool>`: Speichert den Transparenzstatus.
- `leuchtakzent_color: RefCell<Option<RGBA>>`: Speichert die Akzentfarbe.
- `leuchtakzent_intensity: Cell<f64>`: Speichert die Akzentintensität.
- `modules_start: RefCell<BTreeMap<i32, Vec<Widget>>>`: Geordnete Map von Modulen für den Startbereich. Schlüssel ist `order`.
- `modules_center: RefCell<BTreeMap<i32, Vec<Widget>>>`: Geordnete Map von Modulen für den Mittelbereich.
- `modules_end: RefCell<BTreeMap<i32, Vec<Widget>>>`: Geordnete Map von Modulen für den Endbereich.
- `drawing_area_for_accent: RefCell<Option<gtk::DrawingArea>>`: (Optional) Für benutzerdefiniertes Zeichnen des Akzents.

### **4. GTK-Widget-Implementierungsstrategie (Spezifikation für Manus AI)**

- **Basis-Widget:** Das `PanelWidget` erbt von `gtk::ApplicationWindow`. Diese Wahl ist entscheidend für die Integration mit `gtk4-layer-shell`, da dessen Funktionen (`init_for_window`, `set_layer`, `set_anchor`, `auto_exclusive_zone_enable` etc.) auf einem `gtk::Window` operieren.
- **Initialisierung von `gtk4-layer-shell`:**
    - `gtk_layer_shell::init_for_window(self_wrapper)`: Muss im `constructed`-Handler oder vor dem ersten `map`-Ereignis des Fensters aufgerufen werden.
    - `gtk_layer_shell::set_layer(self_wrapper, gtk_layer_shell::Layer::Top)`: Positioniert das Panel über normalen Anwendungsfenstern.
    - `gtk_layer_shell::set_keyboard_mode(self_wrapper, gtk_layer_shell::KeyboardMode::OnDemand)`: Erlaubt dem Panel oder seinen Kindern, Tastaturfokus zu erhalten, wenn sie ihn anfordern. `None` wäre auch eine Option, wenn Module den Fokus nicht benötigen oder ihn selbst über andere Mechanismen (wie Popovers) handhaben.
    - `gtk_layer_shell::auto_exclusive_zone_enable(self_wrapper)`: Sorgt dafür, dass das Panel Platz auf dem Bildschirm reserviert und andere maximierte Fenster nicht verdeckt.
    - `gtk_layer_shell::set_namespace(self_wrapper, "NovaDEPanel")`: Setzt einen eindeutigen Namespace für das Panel.
    - **Ankerpunkte (`set_anchor`):** Basierend auf der `position`-Eigenschaft werden die Anker gesetzt:
        - `PanelPosition::Top`: `Top=true, Left=true, Right=true, Bottom=false`.
        - `PanelPosition::Bottom`: `Bottom=true, Left=true, Right=true, Top=false`.
- **Internes Layout (Composite Template):**
    - Die interne Struktur des `PanelWidget` wird über eine UI-Datei (`panel_widget.ui`) definiert und als `CompositeTemplate` geladen.
    - **`main_box` (`GtkBox`, Orientation: Horizontal):** Dient als Hauptcontainer. Erstreckt sich über die gesamte Breite und Höhe des Panel-Fensters.
    - **`start_box` (`GtkBox`, Orientation: Horizontal):** Nimmt Module auf, die am Anfang (links) des Panels positioniert sind. `halign = GtkAlign::Start`.
    - **`center_box` (`GtkBox`, Orientation: Horizontal):** Nimmt Module auf, die in der Mitte des Panels positioniert sind. `halign = GtkAlign::Center`, `hexpand = true` (damit dieser Bereich den verfügbaren Platz ausfüllt).
    - **`end_box` (`GtkBox`, Orientation: Horizontal):** Nimmt Module auf, die am Ende (rechts) des Panels positioniert sind. `halign = GtkAlign::End`.
    - Alle drei Boxen (`start_box`, `center_box`, `end_box`) haben `valign = GtkAlign::Center`, um die Module vertikal zu zentrieren.
    - Ein Abstand (`spacing`) zwischen den Boxen und zwischen den Modulen innerhalb der Boxen kann in der `.ui`-Datei oder programmatisch gesetzt werden.
- **Benutzerdefiniertes Zeichnen:** Implementierung erfolgt wie unter Abschnitt 2 beschrieben, falls notwendig.

### **5. Methoden und Funktionssignaturen (Interne und Wrapper-Logik)**

#### **5.1. Interne Methoden (in `PanelWidgetImp`)**

- `fn update_css_classes(&self)`: Aktualisiert die CSS-Klassen des Wrapper-Widgets basierend auf den aktuellen Werten von `position` und `transparency_enabled`.
- `fn update_transparency_visual_internal(&self)`: Stellt das `GdkVisual` des Fensters um, um Transparenz zu aktivieren/deaktivieren.
- `pub(super) fn add_module_ordered(&self, module: &impl glib::IsA<gtk::Widget>, position: ModulePosition, order: i32)`:
    1. Fügt das `module` der entsprechenden internen `BTreeMap` (`modules_start`, `modules_center`, `modules_end`) unter dem Schlüssel `order` hinzu. Wenn bereits Module mit derselben `order` existieren, wird das neue Modul an die `Vec<Widget>` für diese `order` angehängt.
    2. Ruft `self.obj().update_module_layout_internal()` auf.
    3. Emittiert das `module-layout-changed`-Signal.
- `pub(super) fn remove_module_internal(&self, module_to_remove: &impl glib::IsA<gtk::Widget>)`:
    1. Durchsucht alle drei Modul-Maps (`modules_start`, `modules_center`, `modules_end`).
    2. Entfernt das `module_to_remove` (Vergleich über Widget-Zeiger oder eine eindeutige ID, falls Module IDs haben) aus der entsprechenden `BTreeMap` und der zugehörigen `Vec<Widget>`.
    3. Wenn das Modul entfernt wurde:
        - Entfernt das Widget physisch aus dem Eltern-`GtkBox`-Container (`start_box`, `center_box` oder `end_box`).
        - Ruft `self.obj().update_module_layout_internal()` auf.
        - Emittiert das `module-layout-changed`-Signal.

#### **5.2. Methoden des öffentlichen Wrappers (`super::PanelWidget`)**

- `fn setup_layer_shell_internal(&self)`: Führt die Konfiguration von `gtk4-layer-shell` durch, wie in Abschnitt 4 beschrieben. Wird bei der Initialisierung und bei Änderungen der `position`-Eigenschaft aufgerufen.
- `fn update_module_layout_internal(&self)`:
    1. Leert alle drei `GtkBox`-Container (`start_box`, `center_box`, `end_box`), indem alle existierenden Kinder entfernt werden.
    2. Iteriert über die sortierten Module in `imp.modules_start.borrow()` (BTreeMap iteriert Schlüssel sortiert). Für jede `order`, iteriere über die `Vec<Widget>` und füge jedes Widget der `imp.start_box` hinzu (`append`).
    3. Wiederholt Schritt 2 für `imp.modules_center` und `imp.center_box`.
    4. Wiederholt Schritt 2 für `imp.modules_end` und `imp.end_box`.

### **6. Signale (Spezifikation für Manus AI)**

#### **6.1. Emittierte Signale**

- **Signal: `module-layout-changed`**
    - **GObject Signal Name:** `module-layout-changed`
    - **Parameter:** Keine.
    - **Signal-Flags:** `glib::SignalFlags::ACTION` (oder `RUN_FIRST`/`RUN_LAST` je nach Bedarf).
    - **Beschreibung:** Wird emittiert, nachdem Module dem Panel hinzugefügt, daraus entfernt oder ihre Anordnung (durch `update_module_layout_internal`) geändert wurde.
    - **Zweck:** Ermöglicht anderen UI-Komponenten oder Logikmodulen, auf Änderungen im Panel-Layout zu reagieren, z.B. um Größen neu zu berechnen oder Fokus-Reihenfolgen anzupassen.
    - **Auslöser:** Aufruf von `add_module_ordered` oder `remove_module_internal` nach erfolgreicher Modifikation und Layout-Aktualisierung.

#### **6.2. Verbundene Signale (Reaktionen auf externe Events)**

- **Event:** `ThemeChangedEvent` (aus `domain::theming::ThemingEngine`)
    - **Handler-Aktion im `PanelWidget`:**
        1. Die neuen Design-Tokens aus dem `ThemeChangedEvent.new_state.resolved_tokens` extrahieren, die für das Panel relevant sind (insbesondere die Werte für `--leuchtakzent-color-rgba` und `--leuchtakzent-intensity-css`).
        2. Die GObject-Eigenschaften `leuchtakzent-color` und `leuchtakzent-intensity` des `PanelWidget` entsprechend aktualisieren (z.B. `self.set_leuchtakzent_color(Some(parsed_rgba_from_token))`).
        3. `self.queue_draw()` aufrufen, um ein Neuzeichnen des Panels (und des Akzents) zu erzwingen.
- **Signale von `gtk::Settings::default()`:**
    - `notify::gtk-theme-name`:
        - **Handler-Aktion:** Kann verwendet werden, um panel-spezifisches CSS neu zu laden oder Stile anzupassen, falls das Panel-Design stark vom System-GTK-Theme abhängt und nicht vollständig durch das NovaDE-Theming-System abgedeckt wird. Für NovaDE wird primär das eigene Theming-System verwendet, daher ist dies eher ein Fallback.
    - `notify::gtk-application-prefer-dark-theme`:
        - **Handler-Aktion:** Ähnlich wie bei `gtk-theme-name`. Wenn das NovaDE-Theming-System die Light/Dark-Modi nicht vollständig über `ThemeChangedEvent` steuert, kann hierauf reagiert werden, um z.B. CSS-Klassen für Dark Mode zu setzen/entfernen.

### **7. Ereignisbehandlung (Benutzerinteraktion)**

- Das `PanelWidget` selbst ist primär ein Container und reagiert nicht direkt auf komplexe Maus- oder Tastaturereignisse.
- Mausereignisse wie `enter-notify-event` und `leave-notify-event` könnten theoretisch für Tooltips auf dem Panel selbst verwendet werden, aber Tooltips sind eher für die einzelnen Module relevant.
- Der Tastaturfokus wird von den fokussierbaren Modulen innerhalb des Panels verwaltet, nicht vom Panel selbst.

### **8. Interaktionen mit anderen Komponenten/Modulen**

- **`domain::global_settings_and_state_management` (GlobalSettingsService):**
    - Das `PanelWidget` liest beim Start (oder bei Änderungen) seine Konfiguration (z.B. Standardposition, Standardhöhe, initiale Transparenz, welche Module standardmäßig geladen werden sollen) vom `GlobalSettingsService`.
    - Änderungen an diesen Einstellungen im `GlobalSettingsService` (z.B. durch `ui::control_center`) sollten ein `SettingChangedEvent` auslösen, auf das das `PanelWidget` reagiert, um seine GObject-Properties dynamisch anzupassen.
- **`system::compositor` (indirekt über `gtk4-layer-shell`):**
    - Die primäre Interaktion erfolgt über die `gtk4-layer-shell`-Bibliothek, um das Panel als Layer-Oberfläche im Wayland-Compositor zu positionieren und zu verwalten.
- **`domain::theming::ThemingEngine`:**
    - Das `PanelWidget` abonniert das `ThemeChangedEvent` der `ThemingEngine`, um Design-Tokens (insbesondere für `leuchtakzent-color` und Hintergrundstile) zu erhalten und anzuwenden.

### **9. Ausnahmebehandlung (Fehlerdefinitionen)**

Für das `PanelWidget`-Modul selbst werden spezifische Fehler mittels `thiserror` definiert, falls Operationen fehlschlagen können, die nicht durch GTK-interne Fehler abgedeckt sind.

- **Datei:** `src/ui/shell/panel_widget/error.rs`
    
- **Enum `PanelWidgetError`**:
    
    Rust
    
    ```
    use thiserror::Error;
    use gtk::glib; // Für glib::Error
    
    #[derive(Debug, Error)]
    pub enum PanelWidgetError {
        #[error("Fehler bei der Initialisierung der gtk-layer-shell: {0}")]
        LayerShellInitializationFailed(String), // Enthält Details des Fehlers
    
        #[error("Fehler beim Laden des UI-Templates für PanelWidget: {source}")]
        TemplateLoadError { #[from] source: glib::Error },
    
        #[error("Ungültige Modulposition angegeben: {position:?}")]
        InvalidModulePosition { position: super::imp::ModulePosition }, // super::imp verweist auf PanelWidgetImp
    
        #[error("Modul konnte nicht zum Panel hinzugefügt werden: {reason}")]
        ModuleAddFailed { reason: String },
    
        #[error("Modul konnte nicht vom Panel entfernt werden: {reason}")]
        ModuleRemoveFailed { reason: String },
    }
    ```
    
- **Verwendung:** Methoden wie `add_module_ordered` oder `remove_module_internal` könnten `Result<(), PanelWidgetError>` zurückgeben, obwohl in der GTK-Welt Fehler oft durch Signale oder Logging behandelt werden, anstatt Result-Typen in Widget-APIs zu verwenden. Für kritische Setup-Fehler ist `Result` jedoch angemessen. `gtk_layer_shell`-Funktionen geben keine direkten `Result`-Typen zurück, Fehler hier würden eher zu Panics oder visuellen Problemen führen; `LayerShellInitializationFailed` wäre also ein interner Fehler, der geloggt wird, falls die Initialisierung visuell fehlschlägt.
    

### **10. Auflösung "Untersuchungsbedarf" (aus dem Quelldokument)**

Der "Untersuchungsbedarf" bezüglich `gtk4-layer-shell`-Integration und Implementierung des "Leuchtakzents" wurde in den Abschnitten 2, 4 und 8 dieser Spezifikation adressiert.

- **Best Practices `gtk4-layer-shell`:** Initialisierung vor dem `map`-Ereignis, korrekte Wahl des `KeyboardMode`, Setzen eines Namespace, Ankerung für Positionierung und `auto_exclusive_zone_enable` sind spezifiziert. Multi-Monitor-Szenarien erfordern separate `PanelWidget`-Instanzen pro Monitor, die über Änderungen in der Monitorkonfiguration (`gdk::Display` Signale) verwaltet werden.
- **Implementierung "Leuchtakzent":** Präferenz für CSS `box-shadow`. Falls unzureichend, benutzerdefiniertes Zeichnen mit Cairo auf einer `gtk::DrawingArea` oder direkt im Panel-Fenster (unter Verwendung von `set_draw_func` für Widgets oder Snapshot-Funktion für komplexere Fälle). Details zur Verwendung von `gdk::RGBA`, `cairo::LinearGradient` und Transparenz sind spezifiziert.

### **11. Dateistruktur (Zusammenfassung)**

- `src/ui/shell/panel_widget/mod.rs` (Öffentliche API, GObject Wrapper)
- `src/ui/shell/panel_widget/imp.rs` (Private GObject Implementierung, Logik)
- `src/ui/shell/panel_widget/error.rs` (Spezifische Fehlerdefinitionen)
- `resources/ui/shell/panel_widget.ui` (XML-Definition für Composite Template)

Diese Struktur fördert Modularität und Trennung von Belangen.

**Nächster Schritt:** Detaillierte Spezifikation für `ui::shell::panel_widget::AppMenuButton`.

# **UI-Schicht: Ultra-Feinspezifikation und Implementierungsleitfaden**

## **Modul: ui::shell::panel_widget::AppMenuButton (Anwendungsmenü-Button)**

### **1. Modul-Identifikation und Zweck**

- **Modul-ID:** `ui::shell::panel_widget::AppMenuButton`
- **Übergeordnetes Modul:** `ui::shell::PanelWidget`
- **Abhängigkeiten:**
    - `novade-core`
    - `novade-domain` (potenziell für das Abrufen von Anwendungsmetadaten, falls nicht direkt über Systemschicht)
    - `novade-system` (insbesondere `system::compositor` für Informationen zum aktiven Fenster/AppID und `system::dbus` für die `org.gtk.Menus`-Schnittstelle)
    - `gtk4-rs` (GTK4 Rust Bindings)
    - `zbus` (für direkte D-Bus-Kommunikation, falls nicht vollständig von `novade-system` abstrahiert)
    - `once_cell`
    - `thiserror`
    - `tracing`
- **Zweck:** Das `AppMenuButton` ist ein spezialisiertes Panel-Modul, das als `gtk::MenuButton` (oder eine benutzerdefinierte Ableitung) implementiert wird. Seine Hauptverantwortung ist die Darstellung des globalen Anwendungsmenüs (typischerweise `GMenuModel`) der aktuell fokussierten Applikation. Es muss die aktive Anwendung identifizieren, deren Menümodell über D-Bus abrufen und dieses in einem `gtk::PopoverMenu` darstellen. Das Aussehen des Buttons (Icon, Name) wird dynamisch an die aktive Anwendung angepasst.

### **2. Visuelles Design und Theming (Spezifikation für Manus AI)**

- **Darstellung als Button:**
    - Standardmäßig wird das Icon der aktiven Anwendung angezeigt (`gtk::Image` im `gtk::MenuButton`).
    - **Fallback-Icon:** Wenn kein Anwendungsicon verfügbar ist oder keine Anwendung ein Menü bereitstellt, wird ein generisches "Anwendungsmenü"-Icon angezeigt (z.B. `application-menu-symbolic`).
    - **Beschriftung (Label):** Optional kann der Name der aktiven Anwendung (`gtk::Label`) neben dem Icon angezeigt werden. Dies ist konfigurierbar (z.B. über eine globale Einstellung) und hängt vom verfügbaren Platz im Panel ab. Standardmäßig wird nur das Icon angezeigt, um Platz zu sparen.
- **Tooltip:** Der Tooltip des `AppMenuButton` zeigt immer den Namen der aktiven Anwendung an, auch wenn dieser nicht als Label sichtbar ist (`gtk::Widget::set_tooltip_text()`).
- **PopoverMenu:** Beim Klick auf den Button wird ein `gtk::PopoverMenu` angezeigt, das die Menüeinträge des `GMenuModel` der aktiven Anwendung enthält. Das Styling dieses Popovers folgt dem globalen Theme.
- **Styling (CSS):**
    - **CSS-Knotenname:** `menubutton` (wenn von `gtk::MenuButton` geerbt) oder `button` (wenn eine benutzerdefinierte Schaltfläche mit manuellem Popover verwendet wird). Das Widget selbst bekommt den CSS-Namen `appmenubutton` (`klass.set_css_name("appmenubutton");`).
    - **CSS-Klassen (dynamisch):**
        - `.app-menu-button` (statisch): Allgemeine Klasse für spezifisches Styling.
        - `.active-app-menu` (dynamisch): Wird gesetzt, wenn ein Anwendungsmenü erfolgreich geladen und an den Button gebunden wurde.
        - `.no-app-menu` (dynamisch): Wird gesetzt, wenn kein Menü für die aktive Anwendung verfügbar ist, keine Anwendung fokussiert ist oder ein Fehler beim Laden des Menüs aufgetreten ist.
        - `.loading-app-menu` (dynamisch): Wird gesetzt, während das Menü aktiv geladen wird. Dies kann für ein visuelles Feedback (z.B. Spinner-ähnliche Zustandsänderung) genutzt werden.
- **Zustandsabhängiges Aussehen:**
    - **Kein Menü / Fehler:** Button könnte leicht ausgegraut oder mit einem speziellen Indikator versehen sein.
    - **Laden:** Ein subtiler Ladeindikator könnte angezeigt werden (z.B. pulsierendes Icon).

### **3. Datenstrukturen, Eigenschaften und Zustand (Spezifikation für Manus AI)**

Das `AppMenuButton` wird als benutzerdefiniertes GObject-Widget implementiert, das von `gtk::MenuButton` erbt.

#### **3.1. Datei: `src/ui/shell/panel_widget/app_menu_button/mod.rs` (Öffentliche API)**

Rust

```
use gtk::glib;
use gtk::subclass::prelude::*;
use gtk::{gio, MenuButton, Widget}; // Widget für IsA

mod imp;

glib::wrapper! {
    pub struct AppMenuButton(ObjectSubclass<imp::AppMenuButton>)
        @extends gtk::MenuButton, gtk::Button, gtk::Widget, // gtk::Button hinzugefügt
        @implements gtk::Accessible, gtk::Actionable, gtk::Buildable, gtk::ConstraintTarget;
}

impl AppMenuButton {
    pub fn new() -> Self {
        glib::Object::builder::<Self>().build()
    }

    /// Aktualisiert die Informationen über das aktive Fenster und löst ggf. eine Menüaktualisierung aus.
    /// Diese Methode wird typischerweise von einem übergeordneten Shell-Dienst aufgerufen,
    /// der Änderungen des Fensterfokus überwacht.
    ///
    /// # Parameter
    /// * `app_id`: Optionale ID der aktiven Anwendung (z.B. "org.gnome.TextEditor").
    /// * `app_name`: Optionaler Anzeigename der aktiven Anwendung.
    /// * `icon_name`: Optionaler Icon-Name der aktiven Anwendung.
    ///
    /// `noexcept`
    pub fn update_active_window_info(&self, app_id: Option<String>, app_name: Option<String>, icon_name: Option<String>) {
        self.imp().update_active_window_info_internal(app_id, app_name, icon_name);
    }

    // Methode, um das Menü explizit neu zu laden (z.B. bei manueller Aktion oder Fehlerwiederholung)
    pub fn refresh_menu(&self) {
        self.imp().trigger_menu_update_for_current_app_internal();
    }
}

impl Default for AppMenuButton {
    fn default() -> Self {
        Self::new()
    }
}
```

#### **3.2. Datei: `src/ui/shell/panel_widget/app_menu_button/imp.rs` (Private GObject-Implementierung)**

Rust

```
use gtk::glib::{self, Properties, ParamSpec, Value, subclass::Signal, Bytes};
use gtk::subclass::prelude::*;
use gtk::{gio, MenuButton, Button, Widget, Image, Label, Box as GtkBox, Orientation, PopoverMenu, Align};
use std::cell::{Cell, RefCell};
use std::sync::Arc; // Für zbus Connection
use once_cell::sync::Lazy;
use zbus::Connection; // Für D-Bus Kommunikation
use tracing;

use crate::ui::shell::panel_widget::app_menu_button::error::AppMenuButtonError; // Error-Typ

// ### Enums für internen Zustand ###
#[derive(Debug, Default, Clone, Copy, PartialEq, Eq)]
pub enum MenuFetchStatus {
    #[default]
    Idle, // Noch kein Versuch unternommen oder zurückgesetzt
    Loading, // Menü wird gerade über D-Bus geladen
    Success, // Menü erfolgreich geladen
    NoMenuAvailable, // Anwendung hat kein Menü oder D-Bus-Dienst nicht gefunden
    Error(AppMenuButtonError), // Ein spezifischer Fehler ist aufgetreten
}

// Notwendig für Speicherung in GObject Property, falls der Status als Property verfügbar gemacht wird
// impl ToValue for MenuFetchStatus { ... }
// impl FromValue for MenuFetchStatus { ... }

// ### GObject Properties Definition ###
static APP_MENU_BUTTON_PROPERTIES: Lazy<Vec<ParamSpec>> = Lazy::new(|| {
    vec![
        ParamSpec::new_string(
            "active-app-name",
            "Active Application Name",
            "Name der Anwendung, deren Menü angezeigt wird oder angezielt ist.",
            None, // Kein Default-Wert, da dynamisch
            glib::ParamFlags::READABLE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        ParamSpec::new_string(
            "active-app-icon-name",
            "Active Application Icon Name",
            "Icon-Name der Anwendung, deren Menü angezielt ist.",
            None,
            glib::ParamFlags::READABLE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        ParamSpec::new_boolean(
            "has-menu",
            "Has Menu",
            "true, wenn ein Menü für die aktive Anwendung verfügbar und geladen ist.",
            false, // Default
            glib::ParamFlags::READABLE | glib::ParamFlags::EXPLICIT_NOTIFY,
        ),
        // Potenziell eine Property für MenuFetchStatus, aber String oder Int Enum wäre einfacher für GObject
    ]
});

#[derive(Default)]
pub struct AppMenuButton {
    // UI-Elemente (falls nicht über CompositeTemplate)
    // Stattdessen als Teil des MenuButton-Parents oder im Popover
    button_content_box: RefCell<Option<GtkBox>>, // Box für Icon und Label
    app_icon_widget: RefCell<Option<Image>>,
    app_name_widget: RefCell<Option<Label>>,
    popover_menu: RefCell<Option<PopoverMenu>>,

    // GObject Properties
    #[property(get, explicit_notify)]
    active_app_name: RefCell<Option<String>>,
    #[property(get, explicit_notify)]
    active_app_icon_name: RefCell<Option<String>>,
    #[property(get, explicit_notify)]
    has_menu: Cell<bool>,

    // Interner Zustand
    active_app_id_internal: RefCell<Option<String>>, // Unterscheidung zum Property für interne Nutzung
    menu_fetch_status_internal: RefCell<MenuFetchStatus>,
    current_menu_model_internal: RefCell<Option<gio::MenuModel>>,
    dbus_connection: RefCell<Option<Arc<Connection>>>, // Arc für Teilen mit async Tasks

    // Service-Abhängigkeit (von ui::shell oder Systemschicht)
    // Annahme: Ein Dienst liefert aktive Fensterinformationen und D-Bus-Verbindung
    // active_window_monitor: RefCell<Option<Arc<dyn ActiveWindowMonitoringService>>>,
    // Wird hier vereinfacht durch direkte D-Bus-Nutzung und Aufruf von update_active_window_info
}

#[glib::object_subclass]
impl ObjectSubclass for AppMenuButton {
    const NAME: &'static str = "NovaDEAppMenuButton";
    type Type = super::AppMenuButton; // Der öffentliche Wrapper
    type ParentType = gtk::MenuButton;

    fn class_init(klass: &mut Self::Class) {
        klass.install_properties(&APP_MENU_BUTTON_PROPERTIES);
        klass.set_css_name("appmenubutton");
    }
}

#[glib::derived_properties]
impl ObjectImpl for AppMenuButton {
    fn constructed(&self) {
        self.parent_constructed();
        let obj = self.obj(); // Der öffentliche Wrapper `super::AppMenuButton`

        // Standard-Icon und Popover setzen
        let icon = Image::from_icon_name("application-menu-symbolic");
        let label = Label::new(None);
        label.set_visible(false); // Standardmäßig kein Label
        let content_box = GtkBox::new(GtkOrientation::Horizontal, 6);
        content_box.append(&icon);
        content_box.append(&label);
        obj.set_child(Some(&content_box));

        self.button_content_box.replace(Some(content_box));
        self.app_icon_widget.replace(Some(icon));
        self.app_name_widget.replace(Some(label));

        let popover = PopoverMenu::new_from_model(None::<&gio::MenuModel>); // Initial leer
        obj.set_popover(Some(&popover));
        self.popover_menu.replace(Some(popover));

        // Initialisiere D-Bus Verbindung asynchron
        let widget_clone = obj.clone();
        glib::MainContext::default().spawn_local(async move {
            match Connection::session().await {
                Ok(conn) => {
                    widget_clone.imp().dbus_connection.replace(Some(Arc::new(conn)));
                    tracing::info!("AppMenuButton: D-Bus Session-Verbindung hergestellt.");
                    // Initiales Update versuchen, falls schon eine App fokussiert ist (Info müsste von außen kommen)
                }
                Err(e) => {
                    tracing::error!("AppMenuButton: Fehler beim Verbinden zum D-Bus Session-Bus: {}", e);
                    widget_clone.imp().menu_fetch_status_internal.replace(MenuFetchStatus::Error(AppMenuButtonError::DBusConnectionError(e.to_string())));
                    widget_clone.imp().update_button_appearance_and_state_internal();
                }
            }
        });
        self.update_button_appearance_and_state_internal(); // Initiales Aussehen setzen
    }
}

impl WidgetImpl for AppMenuButton {}
impl ButtonImpl for AppMenuButton {} // Notwendig, da gtk::MenuButton von gtk::Button erbt
impl MenuButtonImpl for AppMenuButton {}

// ### Interne Implementierungsmethoden für AppMenuButtonImp ###
impl AppMenuButton {
    pub(super) fn update_active_window_info_internal(&self, app_id: Option<String>, app_name: Option<String>, icon_name: Option<String>) {
        tracing::debug!("AppMenuButton: update_active_window_info: app_id={:?}, name={:?}, icon={:?}", app_id, app_name, icon_name);
        let current_app_id = self.active_app_id_internal.borrow().clone();

        // Nur neu laden, wenn sich die app_id geändert hat oder vorher None war
        let needs_menu_update = current_app_id != app_id || (current_app_id.is_none() && app_id.is_some());

        self.active_app_id_internal.replace(app_id.clone());
        // Die Properties direkt über den Wrapper setzen, um Notify auszulösen
        self.obj().set_property("active-app-name", app_name.to_value());
        self.obj().set_property("active-app-icon-name", icon_name.to_value());
        // self.active_app_name.replace(app_name); // Direktes Setzen ohne Notify
        // self.active_app_icon_name.replace(icon_name); // Direktes Setzen ohne Notify

        if needs_menu_update {
            if app_id.is_some() {
                self.trigger_menu_update_for_current_app_internal();
            } else {
                // Keine aktive App, Menü zurücksetzen
                self.current_menu_model_internal.replace(None);
                self.obj().set_menu_model(None::<&gio::MenuModel>);
                self.menu_fetch_status_internal.replace(MenuFetchStatus::Idle);
                self.obj().set_property("has-menu", false);
            }
        }
        self.update_button_appearance_and_state_internal();
    }

    pub(super) fn trigger_menu_update_for_current_app_internal(&self) {
        let app_id_opt = self.active_app_id_internal.borrow().clone();
        let dbus_conn_opt = self.dbus_connection.borrow().clone();

        if let (Some(app_id), Some(dbus_conn)) = (app_id_opt, dbus_conn_opt) {
            if app_id.is_empty() {
                tracing::warn!("AppMenuButton: Leere AppID erhalten, Menü-Update übersprungen.");
                self.menu_fetch_status_internal.replace(MenuFetchStatus::Error(AppMenuButtonError::MenuNotFound("Leere AppID".to_string())));
                self.update_button_appearance_and_state_internal();
                return;
            }

            tracing::info!("AppMenuButton: Starte Menü-Update für AppID: {}", app_id);
            self.menu_fetch_status_internal.replace(MenuFetchStatus::Loading);
            self.update_button_appearance_and_state_internal(); // Ladezustand anzeigen

            let widget_clone = self.obj().clone(); // Klon des Wrappers
            glib::MainContext::default().spawn_local(async move {
                let result = Self::fetch_menu_for_app_async(dbus_conn, app_id.clone()).await;
                widget_clone.imp().handle_menu_fetch_result_internal(result, &app_id);
            });
        } else {
            tracing::debug!("AppMenuButton: Keine AppID oder D-Bus-Verbindung für Menü-Update vorhanden.");
            self.menu_fetch_status_internal.replace(MenuFetchStatus::Idle);
            if app_id_opt.is_none() { // Explizit kein Menü, wenn keine AppID
                 self.obj().set_menu_model(None::<&gio::MenuModel>);
                 self.obj().set_property("has-menu", false);
            }
            self.update_button_appearance_and_state_internal();
        }
    }

    async fn fetch_menu_for_app_async(dbus_conn: Arc<Connection>, app_id: String) -> Result<gio::MenuModel, AppMenuButtonError> {
        // Primärer Mechanismus: org.gtk.Menus auf dem Bus-Namen der Anwendung
        // Standardpfade sind /org/gtk/menus/menubar oder /org/gtk/menus/appmenu
        // oder ein von GApplication festgelegter Pfad.
        // Hier wird versucht, gängige Pfade zu prüfen.
        let common_paths = [
            "/org/gtk/menus/menubar",
            "/org/gtk/menus/appmenu",
            &format!("/{}", app_id.replace('.', "/")), // z.B. /org/gnome/TextEditor
            "/", // Manchmal exportieren Apps Menüs direkt am Root-Pfad ihres Bus-Namens
        ];

        for path_str in common_paths.iter() {
            // Zuerst versuchen, direkt ein DBusMenuModel zu erstellen,
            // was die org.gtk.Menus Schnittstelle verwendet.
            match gio::DBusMenuModel::new(&dbus_conn, Some(app_id.as_str()), path_str) {
                Ok(menu_model) => {
                    // Prüfen, ob das Menü tatsächlich Items hat, da ein leeres Modell nicht nützlich ist.
                    // DBusMenuModel ist ein Live-Objekt, Änderungen werden reflektiert.
                    // Ein einfacher Check hier ist schwierig, da n_items() nicht direkt auf MenuModel existiert.
                    // Wir vertrauen darauf, dass ein erfolgreich erstelltes Modell gültig ist.
                    tracing::info!("AppMenuButton: GMenuModel für '{}' an Pfad '{}' erfolgreich via org.gtk.Menus bezogen.", app_id, path_str);
                    return Ok(menu_model.upcast::<gio::MenuModel>());
                }
                Err(e) => {
                    tracing::debug!("AppMenuButton: Kein GMenuModel für '{}' an Pfad '{}' via org.gtk.Menus gefunden: {}", app_id, path_str, e);
                }
            }
        }
        
        // Fallback: com.canonical.AppMenu.Registrar (veraltet und X11-lastig, hier nur als Referenz)
        // Für eine reine Wayland-Umgebung ist dies weniger relevant, es sei denn, XWayland-Apps nutzen es.
        // Diese Logik wird hier nicht vollständig implementiert, da sie komplex ist und oft nicht funktioniert.
        // tracing::warn!("AppMenuButton: Fallback zu com.canonical.AppMenu.Registrar nicht implementiert.");

        tracing::warn!("AppMenuButton: Kein Menü für AppID '{}' auf bekannten Pfaden gefunden.", app_id);
        Err(AppMenuButtonError::MenuNotFound(app_id))
    }

    fn handle_menu_fetch_result_internal(&self, result: Result<gio::MenuModel, AppMenuButtonError>, app_id_for_result: &str) {
        // Sicherstellen, dass das Ergebnis noch für die aktuell aktive App relevant ist.
        // Wenn der Benutzer schnell die App gewechselt hat, könnte dieses Ergebnis veraltet sein.
        let current_app_id = self.active_app_id_internal.borrow();
        if current_app_id.as_deref() != Some(app_id_for_result) {
            tracing::info!("AppMenuButton: Menü-Ergebnis für '{}' ist veraltet, aktuelle App ist '{:?}'. Ignoriere.", app_id_for_result, current_app_id);
            // Status nicht ändern, da ein neuer Ladevorgang ggf. läuft
            return;
        }

        match result {
            Ok(menu_model) => {
                tracing::info!("AppMenuButton: Menü für '{}' erfolgreich geladen.", app_id_for_result);
                self.current_menu_model_internal.replace(Some(menu_model.clone()));
                self.obj().set_menu_model(Some(&menu_model));
                self.menu_fetch_status_internal.replace(MenuFetchStatus::Success);
                self.obj().set_property("has-menu", true);
            }
            Err(e) => {
                tracing::warn!("AppMenuButton: Fehler beim Laden des Menüs für '{}': {:?}", app_id_for_result, e);
                self.current_menu_model_internal.replace(None);
                self.obj().set_menu_model(None::<&gio::MenuModel>);
                self.menu_fetch_status_internal.replace(MenuFetchStatus::Error(e.clone())); // Klonen des Fehlers
                 if matches!(e, AppMenuButtonError::MenuNotFound(_)) {
                    self.menu_fetch_status_internal.replace(MenuFetchStatus::NoMenuAvailable);
                }
                self.obj().set_property("has-menu", false);
            }
        }
        self.update_button_appearance_and_state_internal();
    }

    fn update_button_appearance_and_state_internal(&self) {
        let obj = self.obj();
        let status = *self.menu_fetch_status_internal.borrow();
        let has_actual_menu = self.current_menu_model_internal.borrow().is_some() && status == MenuFetchStatus::Success;

        // CSS-Klassen aktualisieren
        obj.remove_css_class("no-app-menu");
        obj.remove_css_class("active-app-menu");
        obj.remove_css_class("loading-app-menu");

        if has_actual_menu {
            obj.add_css_class("active-app-menu");
        } else if status == MenuFetchStatus::NoMenuAvailable || matches!(status, MenuFetchStatus::Error(_)) || self.active_app_id_internal.borrow().is_none() {
            obj.add_css_class("no-app-menu");
        } else if status == MenuFetchStatus::Loading {
            obj.add_css_class("loading-app-menu");
        }

        // Sensitivität des Buttons
        obj.set_sensitive(has_actual_menu || status == MenuFetchStatus::Loading); // Aktiv während Laden, um Klick auf leeres Popover zu verhindern

        // Icon und Label (Label wird hier nicht primär verwendet)
        let icon_widget_opt = self.app_icon_widget.borrow();
        if let Some(icon_widget) = icon_widget_opt.as_ref() {
            let icon_name_prop = self.obj().property::<Option<String>>("active-app-icon-name");
            if let Some(icon_name_str) = icon_name_prop.as_ref().filter(|s| !s.is_empty()) {
                icon_widget.set_from_icon_name(Some(icon_name_str));
            } else {
                icon_widget.set_from_icon_name(Some("application-menu-symbolic")); // Fallback
            }
        }
        
        // Tooltip aktualisieren
        let app_name_prop = self.obj().property::<Option<String>>("active-app-name");
        if let Some(name_str) = app_name_prop.as_ref().filter(|s| !s.is_empty()) {
            obj.set_tooltip_text(Some(name_str));
        } else {
            obj.set_tooltip_text(Some("Anwendungsmenü")); // Fallback-Tooltip
        }
    }
}

```

#### **3.3. GObject-Eigenschaften (Zusammenfassung)**

|Eigenschaftsname|Rust-Feld (Imp)|Typ (Rust / GLib)|Zugriff|Standardwert|Beschreibung|
|:--|:--|:--|:--|:--|:--|
|`active-app-name`|`active_app_name`|`Option<String>` / G_TYPE_STRING|R|`None`|Name der Anwendung, deren Menü angezeigt wird oder angezielt ist.|
|`active-app-icon-name`|`active_app_icon_name`|`Option<String>` / G_TYPE_STRING|R|`None`|Icon-Name der Anwendung.|
|`has-menu`|`has_menu`|`bool` / G_TYPE_BOOLEAN|R|`false`|`true`, wenn ein Menü für die aktive Anwendung verfügbar und geladen ist.|
|(Parent `menu-model`)|N/A (Parent-Eigenschaft)|`Option<gio::MenuModel>` / G_TYPE_OBJECT|R/W|`None`|Das Menümodell, das im Popover angezeigt wird. Wird von dieser Logik gesetzt.|
|(Parent `icon-name`)|N/A (Parent-Eigenschaft)|`Option<String>`|R/W|`None`|Das Icon des MenuButtons selbst. Diese Logik setzt das Icon eines Kind-Widgets.|
|(Parent `label`)|N/A (Parent-Eigenschaft)|`Option<String>`|R/W|`None`|Das Label des MenuButtons selbst. Diese Logik setzt das Label eines Kind-Widgets.|

In Google Sheets exportieren

#### **3.4. Interner Zustand (Felder in `AppMenuButtonImp`)**

- `button_content_box: RefCell<Option<GtkBox>>`: Container für Icon und optionales Label im Button.
- `app_icon_widget: RefCell<Option<Image>>`: Das `gtk::Image`-Widget für das Anwendungsicon.
- `app_name_widget: RefCell<Option<Label>>`: Das `gtk::Label`-Widget für den Anwendungsnamen (standardmäßig unsichtbar).
- `popover_menu: RefCell<Option<PopoverMenu>>`: Das `gtk::PopoverMenu`, das an den `gtk::MenuButton` gebunden ist.
- `active_app_id_internal: RefCell<Option<String>>`: Speichert die ID der aktuell fokussierten Anwendung. Dient als Trigger für Menü-Updates.
- `menu_fetch_status_internal: RefCell<MenuFetchStatus>`: Verfolgt den Zustand des Menüabrufs (Idle, Loading, Success, Error, NoMenuAvailable).
- `current_menu_model_internal: RefCell<Option<gio::MenuModel>>`: Hält das aktuell geladene `GMenuModel`.
- `dbus_connection: RefCell<Option<Arc<Connection>>>`: Die D-Bus-Verbindung für Abfragen. `Arc` wird verwendet, um die Verbindung sicher mit asynchronen Tasks zu teilen, die das Menü laden.

### **4. GTK-Widget-Implementierungsstrategie (Spezifikation für Manus AI)**

- **Basis-Widget:** Das `AppMenuButton` erbt von `gtk::MenuButton`. Diese Klasse bietet bereits die Funktionalität, ein Popover beim Klick anzuzeigen.
- **Button-Inhalt:**
    - Das `AppMenuButton` verwendet intern ein `GtkBox` (horizontal orientiert) als Kind-Widget, um ein `gtk::Image` (für das Icon) und optional ein `gtk::Label` (für den Anwendungsnamen) aufzunehmen. Standardmäßig ist nur das Icon sichtbar.
    - Das Icon wird basierend auf `active_app_icon_name` aktualisiert.
- **Popover und Menümodell:**
    - Ein `gtk::PopoverMenu` wird erstellt und als Popover für den `gtk::MenuButton` gesetzt (`obj.set_popover(Some(&popover))`).
    - Die Eigenschaft `menu-model` des `gtk::MenuButton` (oder direkt des `gtk::PopoverMenu`) wird dynamisch mit dem über D-Bus abgerufenen `gio::MenuModel` aktualisiert: `obj.set_menu_model(Some(&menu_model))`.
    - Wenn kein Menü verfügbar ist oder ein Fehler auftritt, wird `obj.set_menu_model(None::<&gio::MenuModel>)` gesetzt.

### **5. Methoden und Funktionssignaturen (Interne und Wrapper-Logik)**

#### **5.1. Interne Methoden (in `AppMenuButtonImp`)**

- `fn update_active_window_info_internal(&self, app_id: Option<String>, app_name: Option<String>, icon_name: Option<String>)`:
    1. Aktualisiert die internen Felder `active_app_id_internal`.
    2. Setzt die GObject-Properties `active-app-name` und `active-app-icon-name` des Wrapper-Objekts, um `notify`-Signale auszulösen.
    3. Wenn sich die `app_id` geändert hat oder von `None` zu `Some` wurde:
        - Wenn `app_id` `Some` ist und nicht leer: Ruft `trigger_menu_update_for_current_app_internal()` auf.
        - Wenn `app_id` `None` ist: Setzt `current_menu_model_internal` auf `None`, ruft `obj.set_menu_model(None)` auf, setzt `menu_fetch_status_internal` auf `Idle` und `has-menu` Property auf `false`.
    4. Ruft `update_button_appearance_and_state_internal()` auf.
- `fn trigger_menu_update_for_current_app_internal(&self)`:
    1. Holt `active_app_id_internal` und `dbus_connection`.
    2. Wenn beide `Some` sind und `app_id` nicht leer ist:
        - Setzt `menu_fetch_status_internal` auf `Loading`.
        - Ruft `update_button_appearance_and_state_internal()` auf, um Ladezustand anzuzeigen.
        - Startet eine asynchrone Task (`glib::MainContext::default().spawn_local`) die `Self::fetch_menu_for_app_async(dbus_conn_arc, app_id_str).await` aufruft.
        - Der Callback dieser Task ruft `self.handle_menu_fetch_result_internal(result, &original_app_id_str)` auf.
    3. Wenn keine `app_id` oder keine D-Bus-Verbindung: Setzt Status auf `Idle`, `set_menu_model(None)`, `set_property("has-menu", false)` und aktualisiert Aussehen.
- `async fn fetch_menu_for_app_async(dbus_conn: Arc<Connection>, app_id: String) -> Result<gio::MenuModel, AppMenuButtonError>`:
    1. Versucht, ein `gio::DBusMenuModel` für die gegebene `app_id` und gängige D-Bus-Objektpfade (z.B. `/org/gtk/menus/menubar`, `/org/gtk/menus/appmenu`, `/{app_id_als_pfad}`, `/`) über die `org.gtk.Menus`-Schnittstelle zu erstellen.
        - Nutzt `gio::DBusMenuModel::new(&dbus_conn, Some(app_id.as_str()), path_str)`.
    2. Wenn erfolgreich für einen Pfad: Gibt `Ok(menu_model.upcast())` zurück.
    3. Wenn alle Versuche fehlschlagen: Gibt `Err(AppMenuButtonError::MenuNotFound(app_id))` zurück.
    4. D-Bus-Verbindungsfehler werden als `AppMenuButtonError::DBusError` behandelt.
- `fn handle_menu_fetch_result_internal(&self, result: Result<gio::MenuModel, AppMenuButtonError>, app_id_for_result: &str)`:
    1. Prüft, ob das Ergebnis noch für die aktuell in `active_app_id_internal` gespeicherte App relevant ist. Wenn nicht, ignoriere das Ergebnis (Logge Warnung).
    2. Bei `Ok(menu_model)`:
        - Setzt `current_menu_model_internal` auf `Some(menu_model.clone())`.
        - Ruft `self.obj().set_menu_model(Some(&menu_model))`.
        - Setzt `menu_fetch_status_internal` auf `Success`.
        - Setzt die `has-menu` Property auf `true`.
    3. Bei `Err(e)`:
        - Setzt `current_menu_model_internal` auf `None`.
        - Ruft `self.obj().set_menu_model(None)`.
        - Setzt `menu_fetch_status_internal` auf `Error(e.clone())` oder `NoMenuAvailable` (wenn `e` ein `MenuNotFound` Fehler ist).
        - Setzt die `has-menu` Property auf `false`.
    4. Ruft `update_button_appearance_and_state_internal()` auf.
- `fn update_button_appearance_and_state_internal(&self)`:
    1. Aktualisiert CSS-Klassen (`active-app-menu`, `no-app-menu`, `loading-app-menu`) basierend auf `menu_fetch_status_internal` und ob `current_menu_model_internal` `Some` ist.
    2. Setzt die Sensitivität des Buttons (`gtk::Widget::set_sensitive()`). Der Button ist sensitiv, wenn ein Menü erfolgreich geladen wurde oder gerade geladen wird (um das Popover zu öffnen, das dann ggf. leer ist oder eine Ladeanzeige hat – hier wird es sensitiv, wenn `has_actual_menu` oder `status == Loading`).
    3. Aktualisiert das Icon im `app_icon_widget` basierend auf der `active-app-icon-name` Property (oder Fallback-Icon).
    4. Aktualisiert das Label im `app_name_widget` (falls sichtbar) basierend auf der `active-app-name` Property.
    5. Aktualisiert den Tooltip des Wrapper-Buttons (`obj.set_tooltip_text()`) mit dem Wert der `active-app-name` Property oder einem Fallback-Text.

#### **5.2. Methoden des öffentlichen Wrappers (`super::AppMenuButton`)**

- `pub fn new() -> Self`: Erstellt die Instanz.
- `pub fn update_active_window_info(&self, app_id: Option<String>, app_name: Option<String>, icon_name: Option<String>)`: Ruft die interne `update_active_window_info_internal` auf.
- `pub fn refresh_menu(&self)`: Ruft die interne `trigger_menu_update_for_current_app_internal` auf.

### **6. Signale**

- **Emittierte Signale:** Keine spezifischen benutzerdefinierten Signale für diese Komponente vorgesehen. Erbt und nutzt Signale von `gtk::MenuButton` (z.B. `clicked`, `activate` für Aktionen im Menü) und GObject-Properties (`notify::property-name`).
- **Verbundene Signale:**
    - Keine direkten Verbindungen zu externen Signalen in dieser Komponente. Es wird erwartet, dass ein übergeordneter Dienst (z.B. in `ui::shell` oder der Systemschicht) Änderungen des aktiven Fensters überwacht (z.B. mittels Wayland-Protokollen wie `wlr-foreign-toplevel-management`) und dann `AppMenuButton::update_active_window_info()` aufruft.

### **7. Interaktionen mit anderen Komponenten/Modulen**

- **`ui::shell`-Dienst (oder äquivalenter Systemdienst):**
    - Verantwortlich für die Überwachung des globalen Fensterfokus.
    - Ruft `AppMenuButton::update_active_window_info()` auf, wenn sich das aktive Anwendungsfenster ändert, und übergibt `app_id` (z.B. aus `zwlr_foreign_toplevel_handle_v1.app_id`), `app_name` (aus Fenstertitel oder App-Metadaten) und `icon_name`.
- **D-Bus (direkt oder via `novade-system`):**
    - Das `AppMenuButton` (bzw. seine `imp`-Logik) stellt direkt D-Bus-Anfragen über `zbus`, um das `GMenuModel` von Anwendungen abzurufen, die die `org.gtk.Menus`-Schnittstelle auf ihrem Anwendungs-Bus-Namen bereitstellen.
- **`domain::global_settings_and_state_management`:**
    - Könnte verwendet werden, um Konfigurationsoptionen für das `AppMenuButton` zu speichern (z.B. ob der Anwendungsname neben dem Icon angezeigt werden soll).

### **8. Ausnahmebehandlung (Fehlerdefinitionen)**

- **Datei:** `src/ui/shell/panel_widget/app_menu_button/error.rs`
    
- **Enum `AppMenuButtonError`**:
    
    Rust
    
    ```
    use thiserror::Error;
    
    #[derive(Debug, Clone, Error, PartialEq, Eq)] // Clone, PartialEq, Eq für MenuFetchStatus::Error
    pub enum AppMenuButtonError {
        #[error("D-Bus connection error: {0}")]
        DBusConnectionError(String), // Enthält zbus::Error::to_string()
    
        #[error("D-Bus call failed for app '{app_id}': {dbus_error_name} - {dbus_error_message}")]
        DBusCallFailed {
            app_id: String,
            dbus_error_name: String,
            dbus_error_message: String,
        },
    
        #[error("Menu model not found for application '{0}'.")]
        MenuNotFound(String), // app_id
    
        #[error("Failed to parse menu model for application '{app_id}': {details}")]
        MenuModelParseError { app_id: String, details: String },
    
        #[error("Active window information (app_id) is missing or invalid.")]
        MissingWindowInfo,
    
        #[error("An internal error occurred in AppMenuButton: {0}")]
        InternalError(String),
    }
    ```
    
- Fehler werden im `MenuFetchStatus::Error(AppMenuButtonError)` gespeichert und beeinflussen das Aussehen und die Sensitivität des Buttons. Fehlerdetails werden via `tracing` geloggt.
    

### **9. Auflösung "Untersuchungsbedarf" (aus dem Quelldokument)**

- **Zuverlässige Methode zur Ermittlung des aktiven Fensters/app_id unter Wayland:**
    - Dies ist _nicht_ die direkte Verantwortung des `AppMenuButton`. Ein übergeordneter Dienst (Teil von `ui::shell` oder `novade-system`) muss diese Information bereitstellen.
    - Dieser Dienst verwendet Protokolle wie `wlr-foreign-toplevel-management-unstable-v1` oder `ext-foreign-toplevel-list-v1`.
    - Das `activated`-Ereignis von `zwlr_foreign_toplevel_handle_v1` signalisiert das fokussierte Fenster. Dessen `app_id` wird dann an `AppMenuButton::update_active_window_info()` übergeben.
- **Ermittlung und Konsumierung von `GMenuModel` via D-Bus:**
    - Die Methode `Workspace_menu_for_app_async` implementiert dies.
    - **Primärer Pfad:** `gio::DBusMenuModel::new()` wird verwendet, um das Menü von `bus_name = app_id` und gängigen Objektpfaden (`/org/gtk/menus/menubar`, `/org/gtk/menus/appmenu`, anwendungsspezifischer Pfad) zu beziehen. Dies nutzt die `org.gtk.Menus`-Schnittstelle.
    - **Fallbacks (StatusNotifierItem, AppMenuRegistrar):** Sind für eine moderne Wayland-Umgebung weniger relevant. `StatusNotifierItem.Menu` (D-Bus-Pfad zu `com.canonical.dbusmenu`) könnte ein seltener Fall sein. `AppMenuRegistrar` ist X11-lastig und wird hier nicht priorisiert. Die Spezifikation fokussiert sich auf `org.gtk.Menus`.

### **10. Dateistruktur (Zusammenfassung)**

- `src/ui/shell/panel_widget/app_menu_button/mod.rs` (Öffentliche API, GObject Wrapper)
- `src/ui/shell/panel_widget/app_menu_button/imp.rs` (Private GObject Implementierung, Logik)
- `src/ui/shell/panel_widget/app_menu_button/error.rs` (Spezifische Fehlerdefinitionen)

Diese detaillierte Spezifikation für das `AppMenuButton`-Modul sollte eine klare Grundlage für die Implementierung bieten. Die Aspekte der Fensterfokus-Überwachung sind ausgelagert, was die Komplexität dieses Widgets reduziert und es auf seine Kernaufgabe – das Abrufen und Anzeigen des Anwendungsmenüs – fokussiert.