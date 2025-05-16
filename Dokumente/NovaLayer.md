1. **Definierte APIs (Traits & öffentliche Funktionen):**
    
    - **Kernschicht:** Stellt ihre Funktionalität über `pub` Funktionen und direkt nutzbare `pub` Typen bereit.
    - **Domänenschicht:** Definiert ihre Service-Schnittstellen primär über `pub trait ...: Send + Sync {}` (oft mit `#[async_trait::async_trait]`). Diese Traits werden von konkreten Service-Strukturen implementiert, die typischerweise mit `Arc<tokio::sync::Mutex/RwLock<InternalState>>` arbeiten, um Zustand zu verwalten.
    - **Systemschicht:** Definiert ebenfalls Service-Schnittstellen über Traits für Funktionen, die von der UI-Schicht aufgerufen werden können (z.B. `SystemPowerControlTrait`). Für interne Operationen oder Callbacks von externen Bibliotheken (z.B. Smithay Handler) werden konkrete `impl` Blöcke für die zentralen Zustandsstrukturen (`DesktopState`) verwendet.
    - **UI-Schicht:** Nutzt die Traits der Domänen- und Systemschicht. Interne UI-Komponenten kommunizieren über GTK-Signale, Rust-Methodenaufrufe und ggf. interne Event-Bus-Mechanismen (z.B. `glib::Sender/Receiver` für thread-übergreifende Kommunikation innerhalb der UI).
2. **Event-System (Nachrichtenbasierte Kommunikation):**
    
    - **Domänenschicht-Events:** Werden über `tokio::sync::broadcast` Kanäle von Domänen-Services publiziert. Subscriber (in Domäne, System oder UI) erhalten Klone der Event-Datenstrukturen. Event-Strukturen sind `#[derive(Debug, Clone, Serialize, Deserialize)]` (Serialize/Deserialize für Logging oder potenzielle IPC).
    - **Systemschicht-Events (intern & extern):**
        - Intern: Der `system::event_bridge` (mit `tokio::sync::broadcast`) verteilt System-interne Ereignisse (z.B. `UPowerEvent`, `AudioEvent`, `InputDeviceHotplugEvent`).
        - Extern (für UI): Die Systemschicht übersetzt relevante Low-Level-Systemereignisse (z.B. Wayland-Events, D-Bus-Signale) in spezifische, UI-konsumierbare Events oder aktualisiert Zustände, die von der UI periodisch abgefragt oder über Callbacks (z.B. `glib::idle_add`) verarbeitet werden. Wayland-Events (wie `xdg_surface.configure`) sind ein primärer Kommunikationsweg vom Compositor zu den Anwendungsfenstern (Clients) und indirekt zur UI-Shell.
    - **UI-Schicht-Events:** GTK4-Signale (`button.connect_clicked(...)`), `glib::Property` Benachrichtigungen, benutzerdefinierte Signale auf Widgets.
3. **Direktionale Abhängigkeiten:**
    
    - Kern → (von Domäne, System, UI genutzt)
    - Domäne → (von System, UI genutzt)
    - System → (von UI genutzt)
    - Keine zyklischen Abhängigkeiten zwischen den Hauptschichten-Crates (`novade-core`, `novade-domain`, `novade-system`, `novade-ui`).
4. **Datenfluss und Transformation:**
    
    - Daten, die zwischen Schichten ausgetauscht werden, sind klar definierte Rust-Structs oder -Enums.
    - Wenn eine Schicht Daten von einer tieferen Schicht empfängt, kann sie diese in ihre eigenen internen Repräsentationen transformieren (z.B. D-Bus-`zvariant::Value` wird in der Systemschicht zu einer Rust-Struktur, die dann ggf. an die Domänenschicht als Domänenobjekt weitergegeben wird).

---

**Schnittstellen im Detail (Ultra-Feinspezifikation):**

---

**1. Schnittstelle: Kernschicht (`novade-core`) zu höheren Schichten**

Diese Schnittstelle ist unidirektional. Die Kernschicht stellt Funktionalität bereit und hat keine Kenntnis von den höheren Schichten.

**1.1. `core::types`**

- **Datentypen (Beispiele mit exakten Definitionen):**
    - `pub struct Point<T: Copy + Debug + PartialEq + Default + Send + Sync + 'static> { pub x: T, pub y: T }`
    - `pub struct Size<T: Copy + Debug + PartialEq + Default + Send + Sync + 'static> { pub width: T, pub height: T }`
    - `pub struct Rect<T: Copy + Debug + PartialEq + Default + Send + Sync + 'static> { pub origin: Point<T>, pub size: Size<T> }`
    - `#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, Default)] pub struct RectInt { pub x: i32, pub y: i32, pub width: u32, pub height: u32 }`
    - `#[derive(Debug, Clone, Copy, PartialEq, Serialize, Deserialize)] pub struct Color { pub r: f32, pub g: f32, pub b: f32, pub a: f32 }` (Invarianten: 0.0-1.0, Durchsetzung in `new()` und `serde`-Implementierung).
    - `#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, Default)] pub enum Orientation { #[default] Horizontal, Vertical }`
    - `pub use uuid::Uuid;`
    - `pub use chrono::{DateTime, Utc};`
- **Nutzung (Ultra-Fein):**
    - **Domäne:**
        - `domain::theming::types::AccentColor::value: core::types::Color`
        - `domain::window_management_policy::types::GapSettings::screen_outer_horizontal: u16` (Primitiver Typ, aber Werte könnten von `core::types::Size<u16>` inspiriert sein).
    - **System:**
        - `system::compositor::core::state::DesktopState::pointer_location: core::types::Point<f64, smithay::utils::Logical>`
        - `system::compositor::surface_management::RenderableElement::WaylandSurface::position: core::types::Point<i32, smithay::utils::Logical>`
        - Wayland-Geometrien (z.B. `xdg_surface.configure(.., width: i32, height: i32, ..)`) verwenden primitive Typen, werden aber oft in `core::types::RectInt` für interne Logik umgewandelt.
    - **UI:**
        - `ui::widgets::PlacedWidgetWidget::position: core::types::Point<i32>` (Hypothetisch)
        - CSS-Farben aus `domain::theming::AppliedThemeState::resolved_tokens` werden von der UI-Schicht geparst (ggf. zurück in `core::types::Color`, wenn nötig).

**1.2. `core::errors`**

- **Fehlertypen (Beispiele mit exakten Definitionen):**
    - `#[derive(Debug, thiserror::Error)] pub enum CoreError { #[error("I/O error accessing path '{path}': {source}")] Io { path: PathBuf, #[source] source: std::io::Error }, ... }`
    - `#[derive(Debug, Clone, PartialEq, Eq, thiserror::Error)] pub enum ColorParseError { #[error("Invalid hex color string format for '{0}'")] InvalidHexFormat(String), ... }`
- **Nutzung (Ultra-Fein):**
    - **Domäne:**
        - `domain::theming::errors::ThemingError::TokenFileIoError { path: PathBuf, #[source] source: std::io::Error }` (Könnte `CoreError::Io` wrappen oder direkt `std::io::Error` verwenden und `From<std::io::Error>` implementieren).
        - Wenn `domain::theming::logic::load_raw_tokens_from_file` eine `core::config::ConfigError::FileReadError { path, source }` zurückgibt (falls Dateilesen in `core::config` gekapselt ist), könnte `ThemingError` dies direkt via `#[from]` übernehmen oder spezifischer mappen.
    - **System:**
        - `system::dbus_interfaces::common::errors::DBusInterfaceError::ConnectionFailed(#[from] zbus::Error)` (Wrappt externen Fehler). Wenn `zbus::Error` selbst `std::io::Error` wrappt, könnte die Kette bis `CoreError::Io` reichen.
        - Fehler beim Lesen einer Shader-Datei im `system::compositor::renderer` könnte `CoreError::Io` sein, das in einen `RendererError::ShaderLoadFailed` gewrappt wird.
    - **UI:**
        - Fehler aus der Domänen- oder Systemschicht werden an die UI weitergegeben. Z.B. ein `GlobalSettingsError::PersistenceError { source: CoreError::Io { ... } }` wird von der UI abgefangen. Die UI entscheidet dann, dem Benutzer eine generische "Speichern fehlgeschlagen" Meldung anzuzeigen und loggt den vollständigen Fehler (`error!(error = ?err, "...")`).

**1.3. `core::logging`**

- **Logging-API (Beispiele mit exakten Signaturen):**
    - `pub fn initialize_logging(level_filter: tracing::LevelFilter, format: core::logging::types::LogFormat, writer: impl std::io::Write + Send + Sync + 'static) -> Result<(), core::logging::errors::LoggingError>` (Writer für Flexibilität).
    - Konvention: Alle Module verwenden `tracing::{trace, debug, info, warn, error, instrument};`.
- **Nutzung (Ultra-Fein):**
    - **Anwendungs-Root (`main.rs` in `novade-ui` oder dem Haupt-Binary):**
        
        Rust
        
        ```
        // In main.rs
        fn main() -> Result<(), Box<dyn std::error::Error>> {
            // Konfig laden, um Log-Level und Format zu bestimmen
            // let core_config = novade_core::config::get_global_core_config(); // Annahme: Konfig wurde geladen
            // let log_level = core_config.log_level.into(); // Konvertierung zu tracing::LevelFilter
            // let log_format = core_config.log_format.into(); // Konvertierung zu core::logging::LogFormat
            // novade_core::logging::initialize_logging(log_level, log_format, std::io::stdout())?;
            // ... Rest der Anwendung ...
            Ok(())
        }
        ```
        
    - **Alle Schichten/Module:**
        
        Rust
        
        ```
        // In irgendeiner Funktion
        #[tracing::instrument(skip(sensitive_data))]
        fn process_data(data: &DataType, sensitive_data: &Secret) -> Result<(), MyModuleError> {
            tracing::debug!(input_data_len = data.len(), "Verarbeite Daten.");
            if data.is_empty() {
                tracing::warn!("Leere Daten empfangen.");
                return Err(MyModuleError::EmptyData);
            }
            // ... Logik ...
            tracing::info!(result = "Erfolgreich", "Daten verarbeitet.");
            Ok(())
        }
        ```
        

**1.4. `core::config`**

- **Konfigurations-API (Beispiele mit exakten Signaturen):**
    - **Trait `core::config::ConfigServiceAsync` (Neu, für asynchrones Laden):**
        
        Rust
        
        ```
        #[async_trait::async_trait]
        pub trait ConfigServiceAsync: Send + Sync {
            async fn read_config_file_string(&self, key_or_path: &str) -> Result<String, CoreError>; // CoreError::Io oder CoreError::NotFound
            async fn write_config_file_string(&self, key_or_path: &str, content: String) -> Result<(), CoreError>; // CoreError::Io
        }
        ```
        
        - Eine Implementierung (`FilesystemConfigServiceAsync`) würde `tokio::fs` verwenden.
    - `pub fn load_config_deserialize<T: for<'de> serde::Deserialize<'de>>(content_str: &str, source_description: &str) -> Result<T, CoreError>` (Generische Deserialisierungsfunktion).
    - Globale `CoreConfig` über `pub fn get_global_core_config() -> &'static CoreConfig`. Initialisierung über `pub fn initialize_global_core_config(paths: &[PathBuf], config_service: Arc<dyn ConfigServiceAsync>) -> Result<(), CoreError>`.
- **Nutzung (Ultra-Fein):**
    - **Domäne (`domain::theming::logic`):**
        
        Rust
        
        ```
        // ThemingEngine::internal_load_themes_and_tokens_locked
        async fn load_theme_definition_from_file(
            &self,
            path: &Path,
            config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>,
        ) -> Result<ThemeDefinition, ThemingError> {
            let content = config_service.read_config_file_string(path.to_str().unwrap())
                .await
                .map_err(|core_err| ThemingError::FilesystemError(core_err))?; // Wrapping
            let theme_def: ThemeDefinition = novade_core::config::load_config_deserialize(&content, path.to_string_lossy().as_ref())
                .map_err(|core_err| ThemingError::ThemeFileLoadError { /* ... source: core_err ... */})?;
            Ok(theme_def)
        }
        ```
        
    - **System (`system::mcp_client::connection_manager`):**
        - `McpConnectionManager` könnte `get_global_core_config()` verwenden, um Standardpfade oder Features für MCP-Server zu laden, falls diese in `CoreConfig` definiert sind.


# Kernschicht Implementierungsleitfaden: Modul 1 - Fundamentale Datentypen (core::types)

1. Modulübersicht: core::types
    1. 1.1. Zweck und Verantwortlichkeit Dieses Modul, `core::types`, bildet das Fundament der Kernschicht (core) und somit des gesamten Systems.
        
    2. Seine primäre Verantwortung liegt in der Definition grundlegender, universell einsetzbarer Datentypen, die von allen anderen Schichten und Modulen der Desktop-Umgebung benötigt werden.
        
    3. Dazu gehören geometrische Primitive (wie Punkte, Größen, Rechtecke), Farbdarstellungen und allgemeine Enumerationen (wie Orientierungen).
        
    4. Die in diesem Modul definierten Typen sind bewusst einfach gehalten und repräsentieren reine Datenstrukturen ohne komplexe Geschäftslogik oder Abhängigkeiten zu höheren Schichten oder externen Systemen.
        
    5. Sie dienen als Bausteine für komplexere Operationen und Zustandsrepräsentationen in den Domänen-, System- und Benutzeroberflächenschichten.
        
    6. 1.2. Designphilosophie Das Design von `core::types` folgt den Prinzipien der Modularität, Wiederverwendbarkeit und minimalen Kopplung.
        
    7. Die Typen sind generisch gehalten (wo sinnvoll, z.B. bei geometrischen Primitiven), um Flexibilität für verschiedene numerische Darstellungen (z.B. `i32` für Koordinaten, `f32` für Skalierungsfaktoren) zu ermöglichen.
        
    8. Ein wesentlicher Aspekt ist die klare Trennung von Datenrepräsentation (in `core::types`) und Fehlerbehandlung.
        
    9. Während dieses Modul die Datenstrukturen definiert, werden die spezifischen Fehler, die bei Operationen mit diesen Typen auftreten können (z.B. durch ungültige Werte), in den Modulen definiert, die diese Operationen durchführen (typischerweise in `core::errors` oder modulspezifischen Fehler-Enums höherer Schichten).
        
    10. 1.3. Zusammenspiel mit Fehlerbehandlung Obwohl `core::types` selbst keine Error-Typen definiert, ist das Design der hier enthaltenen Typen entscheidend für eine robuste und konsistente Fehlerbehandlungsstrategie im gesamten Projekt.
        
    11. Die übergeordnete Richtlinie sieht die Verwendung des `thiserror`-Crates vor, um spezifische Fehler-Enums pro Modul zu definieren.
        
    12. Dies ermöglicht eine granulare Fehlerbehandlung, ohne die Komplexität übermäßig zu erhöhen.
        
    13. Die Typen in `core::types` unterstützen diese Strategie, indem sie:
        
        - Standard-Traits implementieren: Alle Typen implementieren grundlegende Traits wie `Debug` und `Display`.
            
        - Dies ist essenziell, damit Instanzen dieser Typen effektiv in Fehlermeldungen und Log-Ausgaben eingebettet werden können, die von höheren Schichten unter Verwendung von `thiserror` generiert werden.
            
        - Eine gute Fehlerdarstellung ist entscheidend für die Fehlersuche und das Verständnis von Problemen im Laufzeitbetrieb.
            
        - Invarianten dokumentieren: Für Typen wie `Rect<T>` existieren logische Invarianten (z.B. nicht-negative Breite und Höhe). Diese Invarianten werden klar dokumentiert.
            
        - Validierung ermöglichen: Wo sinnvoll, werden Methoden zur Überprüfung der Gültigkeit bereitgestellt (z.B. `Rect::is_valid()`).
            
        - Diese Methoden erlauben es aufrufendem Code in höheren Schichten, Zustände zu überprüfen, bevor Operationen ausgeführt werden, die fehlschlagen könnten.
            
        - Keine Panics in Kernfunktionen: Konstruktoren und einfache Zugriffsmethoden in `core::types` lösen keine Panics aus und geben keine `Result`-Typen zurück, um die API auf dieser fundamentalen Ebene einfach und vorhersagbar zu halten.
            
        - Die Verantwortung für die Handhabung potenziell ungültiger Zustände (z.B. ein `Rect` mit negativer Breite, das an eine Rendering-Funktion übergeben wird) liegt bei den konsumierenden Funktionen, die dann die definierten Fehlerpfade (mittels `Result<T, E>` 3 und den `thiserror`-basierten E-Typen) nutzen.
            
    14. Diese Designentscheidungen stellen sicher, dass die fundamentalen Typen nahtlos in das übergeordnete Fehlerbehandlungskonzept integriert werden können, ohne selbst die Komplexität der Fehlerdefinition tragen zu müssen.
        
    15. Die gewählte Fehlerstrategie mit `thiserror` pro Modul wird als ausreichend für die Bedürfnisse der Kernschicht erachtet, auch wenn alternative Ansätze wie `snafu` für komplexere Szenarien existieren, in denen z.B. die Unterscheidung von Fehlern aus derselben Quelle kritisch ist. Für die Kernschicht wird die Einfachheit und Direktheit von `thiserror` bevorzugt.
        
    16. 1.4. Modulabhängigkeiten Dieses Modul ist darauf ausgelegt, minimale externe Abhängigkeiten zu haben, um seine grundlegende Natur und breite Anwendbarkeit zu gewährleisten.
        
    17. Erlaubte Abhängigkeiten:
        
        - `std` (Rust Standardbibliothek)
            
    18. Optionale Abhängigkeiten (derzeit nicht verwendet):
        
        - `num-traits`: Nur hinzufügen, falls generische numerische Operationen benötigt werden, die über `std::ops` hinausgehen.
        - `serde` (mit `derive`-Feature): Nur hinzufügen, wenn Serialisierung/Deserialisierung dieser Basistypen direkt auf dieser Ebene zwingend erforderlich ist (z.B. für Konfigurationsdateien, die diese Typen direkt verwenden).
            
        - Aktuell wird davon ausgegangen, dass Serialisierungslogik in höheren Schichten implementiert wird, um unnötige Abhängigkeiten zu vermeiden.
    19. 1.5. Ziel-Dateistruktur Die Implementierung dieses Moduls erfolgt innerhalb des core-Crates mit folgender Verzeichnisstruktur:
        
        ```
        core/
        └── src/
            ├── Cargo.toml        # (Definiert das 'core' Crate)
            └── src/
                ├── lib.rs          # (Deklariert Kernmodule: pub mod types; pub mod errors; ...)
                └── types/
                    ├── mod.rs       # (Deklariert und re-exportiert Typen: pub mod geometry; pub mod color; ...)
                    ├── geometry.rs # (Enthält Point<T>, Size<T>, Rect<T>)
                    ├── color.rs      # (Enthält Color)
                    └── enums.rs      # (Enthält Orientation, etc.)
        ```
        
2. 2. Spezifikation: Geometrische Primitive (geometry.rs) Diese Datei definiert grundlegende 2D-Geometrietypen, die für Layout, Positionierung und Rendering unerlässlich sind.
    3. 2.1. Struct: `Point<T>`
        1. 2.1.1. Definition und Zweck: Repräsentiert einen Punkt im 2D-Raum mit x- und y-Koordinaten.
            
        2. Generisch über den Typ T.
        3. 2.1.2. Felder:
            - `pub x: T`
            - `pub y: T`
        4. 2.1.3. Assoziierte Konstanten:
            - `pub const ZERO_I32: Point<i32> = Point { x: 0, y: 0 };`
                
            - `pub const ZERO_U32: Point<u32> = Point { x: 0, y: 0 };`
                
            - `pub const ZERO_F32: Point<f32> = Point { x: 0.0, y: 0.0 };`
                
            - `pub const ZERO_F64: Point<f64> = Point { x: 0.0, y: 0.0 };`
                
        5. 2.1.4. Methoden:
            - `pub const fn new(x: T, y: T) -> Self`
                - Erstellt einen neuen Punkt.
                    
            - `pub fn distance_squared(&self, other: &Point<T>) -> T`
                - Berechnet das Quadrat der euklidischen Distanz.
                    
                - Constraints: `T: Copy + std::ops::Add<Output=T> + std::ops::Sub<Output=T> + std::ops::Mul<Output=T>`
                    
            - `pub fn distance(&self, other: &Point<T>) -> T`
                - Berechnet die euklidische Distanz.
                    
                - Constraints: `T: Copy + std::ops::Add<Output=T> + std::ops::Sub<Output=T> + std::ops::Mul<Output=T> + num_traits::Float` (Implementierung nur für Float-Typen sinnvoll oder über sqrt-Funktion). Vorerst nur für f32,f64 implementieren.
            - `pub fn manhattan_distance(&self, other: &Point<T>) -> T`
                - Berechnet die Manhattan-Distanz (`|x1 - x2| + |y1 - y2|`).
                    
                - Constraints: `T: Copy + std::ops::Add<Output=T> + std::ops::Sub<Output=T> + num_traits::Signed` (Benötigt abs()).
        6. 2.1.5. Trait Implementierungen:
            - `impl<T: Copy + Debug + Default> Default for Point<T> { ... }`
                - Bedingung: T muss die jeweiligen Traits ebenfalls implementieren. Default setzt x und y auf T::default().
                    
            - `impl<T: Send + 'static> Send for Point<T> {}`
                
            - `impl<T: Sync + 'static> Sync for Point<T> {}`
                
            - `impl<T: std::ops::Add<Output = T>> std::ops::Add for Point<T>`
                
            - `impl<T: std::ops::Sub<Output = T>> std::ops::Sub for Point<T>`
                
        7. 2.1.6. Generische Constraints (Basis): `T: Copy + Debug + PartialEq + Default + Send + Sync + 'static`.
            
            - Weitere Constraints werden pro Methode spezifiziert.
    4. 2.2. Struct: `Size<T>`
        1. 2.2.1. Definition und Zweck: Repräsentiert eine 2D-Dimension (Breite und Höhe). Generisch über den Typ T.
            
        2. 2.2.2. Felder:
            - `pub width: T`
            - `pub height: T`
        3. 2.2.3. Assoziierte Konstanten:
            - `pub const ZERO_I32: Size<i32> = Size { width: 0, height: 0 };`
                
            - `pub const ZERO_U32: Size<u32> = Size { width: 0, height: 0 };`
                
            - `pub const ZERO_F32: Size<f32> = Size { width: 0.0, height: 0.0 };`
                
            - `pub const ZERO_F64: Size<f64> = Size { width: 0.0, height: 0.0 };`
                
        4. 2.2.4. Methoden:
            - `pub const fn new(width: T, height: T) -> Self`
                - Erstellt eine neue Größe.
                    
            - `pub fn area(&self) -> T`
                - Berechnet die Fläche (`width * height`).
                    
                - Constraints: `T: Copy + std::ops::Mul<Output=T>`
            - `pub fn is_empty(&self) -> bool`
                - Prüft, ob Breite oder Höhe null ist.
                    
                - Constraints: `T: PartialEq + num_traits::Zero`
            - `pub fn is_valid(&self) -> bool`
                - Prüft, ob Breite und Höhe nicht-negativ sind. Nützlich für Typen wie i32.
                    
                - Constraints: `T: PartialOrd + num_traits::Zero`
        5. 2.2.5. Trait Implementierungen:
            - `impl<T: Copy + Debug + Default> Default for Size<T> { ... }`
                - Bedingung: T muss die jeweiligen Traits ebenfalls implementieren. Default setzt width und height auf T::default().
                    
            - `impl<T: Send + 'static> Send for Size<T> {}`
                
            - `impl<T: Sync + 'static> Sync for Size<T> {}`
                
        6. 2.2.6. Generische Constraints (Basis): `T: Copy + Debug + PartialEq + Default + Send + Sync + 'static`.
            
            - Weitere Constraints werden pro Methode spezifiziert. Die Invariante nicht-negativer Dimensionen wird durch `is_valid` prüfbar gemacht, aber nicht durch den Typ erzwungen.
    5. 2.3. Struct: `Rect<T>`
        1. 2.3.1. Definition und Zweck: Repräsentiert ein 2D-Rechteck, definiert durch einen Ursprungspunkt (oben-links) und eine Größe.
            
        2. Generisch über den Typ T.
        3. 2.3.2. Felder:
            - `pub origin: Point<T>`
            - `pub size: Size<T>`
        4. 2.3.3. Assoziierte Konstanten:
            - `pub const ZERO_I32: Rect<i32> = Rect { origin: Point::ZERO_I32, size: Size::ZERO_I32 };`
                
            - `pub const ZERO_U32: Rect<u32> = Rect { origin: Point::ZERO_U32, size: Size::ZERO_U32 };`
                
            - `pub const ZERO_F32: Rect<f32> = Rect { origin: Point::ZERO_F32, size: Size::ZERO_F32 };`
                
            - `pub const ZERO_F64: Rect<f64> = Rect { origin: Point::ZERO_F64, size: Size::ZERO_F64 };`
                
        5. 2.3.4. Methoden:
            - `pub const fn new(origin: Point<T>, size: Size<T>) -> Self`
                
            - `pub fn from_coords(x: T, y: T, width: T, height: T) -> Self`
                - Constraints: T muss die Constraints von `Point::new` und `Size::new` erfüllen.
                    
            - `pub fn x(&self) -> T` (Constraints: `T: Copy`)
                
            - `pub fn y(&self) -> T` (Constraints: `T: Copy`)
                
            - `pub fn width(&self) -> T` (Constraints: `T: Copy`)
                
            - `pub fn height(&self) -> T` (Constraints: `T: Copy`)
                
            - `pub fn top(&self) -> T` (Alias für y, Constraints: `T: Copy`)
                
            - `pub fn left(&self) -> T` (Alias für x, Constraints: `T: Copy`)
                
            - `pub fn bottom(&self) -> T` (y + height, Constraints: `T: Copy + std::ops::Add<Output=T>`)
                
            - `pub fn right(&self) -> T` (x + width, Constraints: `T: Copy + std::ops::Add<Output=T>`)
                
            - `pub fn center(&self) -> Point<T>`
                - Berechnet den Mittelpunkt.
                    
                - Constraints: `T: Copy + std::ops::Add<Output=T> + std::ops::Div<Output=T> + num_traits::FromPrimitive` (Benötigt Division durch 2).
            - `pub fn contains_point(&self, point: &Point<T>) -> bool`
                - Prüft, ob der Punkt innerhalb des Rechtecks liegt (Grenzen inklusiv für top/left, exklusiv für bottom/right).
                    
                - Constraints: `T: Copy + PartialOrd + std::ops::Add<Output=T>`
            - `pub fn intersects(&self, other: &Rect<T>) -> bool`
                - Prüft, ob sich dieses Rechteck mit einem anderen überschneidet.
                    
                - Constraints: `T: Copy + PartialOrd + std::ops::Add<Output=T>`
            - `pub fn intersection(&self, other: &Rect<T>) -> Option<Rect<T>>`
                - Berechnet das Schnittrechteck. Gibt `None` zurück, wenn keine Überschneidung vorliegt.
                    
                - Constraints: `T: Copy + Ord + std::ops::Add<Output=T> + std::ops::Sub<Output=T> + num_traits::Zero`
            - `pub fn union(&self, other: &Rect<T>) -> Rect<T>`
                - Berechnet das umschließende Rechteck beider Rechtecke.
                    
                - Constraints: `T: Copy + Ord + std::ops::Add<Output=T> + std::ops::Sub<Output=T>`
            - `pub fn translated(&self, dx: T, dy: T) -> Rect<T>`
                - Verschiebt das Rechteck um (dx, dy).
                    
                - Constraints: `T: Copy + std::ops::Add<Output=T>`
            - `pub fn scaled(&self, sx: T, sy: T) -> Rect<T>`
                - Skaliert das Rechteck relativ zum Ursprung (0, 0).
                    
                - Beachtet, dass dies Ursprung und Größe skaliert.
                - Constraints: `T: Copy + std::ops::Mul<Output=T>`
            - `pub fn is_valid(&self) -> bool`
                - Prüft, ob `size.is_valid()` wahr ist.
                    
                - Constraints: `T: PartialOrd + num_traits::Zero`
        6. 2.3.5. Trait Implementierungen:
            - `impl<T: Copy + Debug + Default> Default for Rect<T> { ... }`
                - Bedingung: T muss die jeweiligen Traits ebenfalls implementieren. Default verwendet `Point::default()` und `Size::default()`.
                    
            - `impl<T: Send + 'static> Send for Rect<T> {}`
                
            - `impl<T: Sync + 'static> Sync for Rect<T> {}`
                
        7. 2.3.6. Generische Constraints (Basis): `T: Copy + Debug + PartialEq + Default + Send + Sync + 'static`.
            
            - Weitere Constraints werden pro Methode spezifiziert.
        8. 2.3.7. Invarianten und Validierung (Verbindung zur Fehlerbehandlung):
            - Invariante: Logisch sollten width und height der size-Komponente nicht-negativ sein.
                
            - Kontext: Die Verwendung von vorzeichenbehafteten Typen wie i32 für Koordinaten ist üblich, erlaubt aber technisch negative Dimensionen.
                
            - Eine Erzwingung nicht-negativer Dimensionen auf Typebene (z.B. durch u32) wäre zu restriktiv für Koordinatensysteme.
                
            - Konsequenz: Die Flexibilität, `Rect<i32>` zu verwenden, verlagert die Verantwortung für die Validierung auf die Nutzer des Rect-Typs.
                
            - Funktionen in höheren Schichten (z.B. Layout-Algorithmen, Rendering-Engines), die ein `Rect` konsumieren, müssen potenziell ungültige Rechtecke (mit negativer Breite oder Höhe) behandeln.
                
            - Solche Fälle stellen Laufzeitfehler dar, die über das etablierte Fehlerbehandlungssystem (basierend auf `Result<T, E>` und `thiserror`-definierten E-Typen) signalisiert werden müssen.
                
            - Implementierung in `core::types`: Das Modul erzwingt die Invariante nicht zur Compilezeit oder in Konstruktoren.
                
            - Stattdessen wird die Methode `pub fn is_valid(&self) -> bool` bereitgestellt.
                
            - Nutzer von `Rect<T>` (insbesondere mit `T=i32`) sollten diese Methode aufrufen, um die Gültigkeit sicherzustellen, bevor Operationen durchgeführt werden, die eine positive Breite und Höhe voraussetzen.
                
            - Die Dokumentation des Rect-Typs muss explizit auf diese Invariante und die Notwendigkeit der Validierung durch den Aufrufer hinweisen.
                
            - Die Verantwortung für das Melden eines Fehlers bei Verwendung eines ungültigen `Rect` liegt beim Aufrufer, der dafür die Fehlerinfrastruktur (z.B. `core::errors` oder modulspezifische Fehler) nutzt.
                
    6. 2.4. Struct: `RectInt`
        - (aus einer anderen Quelldatei, aber thematisch passend)
        - Repräsentiert ein achsenparalleles Rechteck mit ganzzahligen Koordinaten (`x: i32`, `y: i32`) und Dimensionen (`width: u32`, `height: u32`).
            
            - Methoden u.a. `new(...)`, `from_points(...)`, `top_left()`, `size()`, `right()`, `bottom()`, `contains_point(...)`, `intersects(...)`, `intersection(...)`, `union(...)`, `translate(...)`, `inflate(...)`, `is_empty()`
                
            - Verwendet `saturating_add` / `saturating_sub` um Überläufe zu vermeiden.
                
            - Traits: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Default`
                
3. 3. Spezifikation: Farbdarstellung (color.rs) Diese Datei definiert einen Standard-Farbtyp für die Verwendung im gesamten System.
    4. 3.1. Struct: `Color` (RGBA)
        1. 3.1.1. Definition und Zweck: Repräsentiert eine Farbe mit Rot-, Grün-, Blau- und Alpha-Komponenten.
            
        2. Verwendet `f32`-Komponenten im Bereich `[0.0, 1.0]` für hohe Präzision und Flexibilität bei Farboperationen wie Mischen und Transformationen.
            
        3. 3.1.2. Felder:
            - `pub r: f32` (Rotkomponente, 0.0 bis 1.0)
                
            - `pub g: f32` (Grünkomponente, 0.0 bis 1.0)
                
            - `pub b: f32` (Blaukomponente, 0.0 bis 1.0)
                
            - `pub a: f32` (Alphakomponente, 0.0=transparent bis 1.0=opak)
                
        4. 3.1.3. Assoziierte Konstanten:
            - `pub const TRANSPARENT: Color = Color { r: 0.0, g: 0.0, b: 0.0, a: 0.0 };`
                
            - `pub const BLACK: Color = Color { r: 0.0, g: 0.0, b: 0.0, a: 1.0 };`
                
            - `pub const WHITE: Color = Color { r: 1.0, g: 1.0, b: 1.0, a: 1.0 };`
                
            - `pub const RED: Color = Color { r: 1.0, g: 0.0, b: 0.0, a: 1.0 };`
                
            - `pub const GREEN: Color = Color { r: 0.0, g: 1.0, b: 0.0, a: 1.0 };`
                
            - `pub const BLUE: Color = Color { r: 0.0, g: 0.0, b: 1.0, a: 1.0 };`
                
            - (Weitere Standardfarben nach Bedarf hinzufügen)
                
        5. 3.1.4. Methoden:
            - `pub const fn new(r: f32, g: f32, b: f32, a: f32) -> Self`
                - Erstellt eine neue Farbe.
                    
                - Werte außerhalb `[0.0, 1.0]` werden nicht automatisch geklemmt, dies liegt in der Verantwortung des Aufrufers oder nachfolgender Operationen. `debug_assert!` kann zur Laufzeitprüfung in Debug-Builds verwendet werden.
                    
            - `pub fn from_rgba8(r: u8, g: u8, b: u8, a: u8) -> Self`
                - Konvertiert von 8-Bit-Ganzzahlkomponenten (0-255) zu f32 (0.0-1.0).
                    
                - `value / 255.0`.
            - `pub fn to_rgba8(&self) -> (u8, u8, u8, u8)`
                - Konvertiert von f32 zu 8-Bit-Ganzzahlkomponenten.
                    
                - Klemmt Werte auf `[0.0, 1.0]` und skaliert dann auf (value.clamp(0.0,1.0)∗255.0).round()asu8.
                    
            - `pub fn with_alpha(&self, alpha: f32) -> Self`
                - Erstellt eine neue Farbe mit dem angegebenen Alpha-Wert, wobei RGB beibehalten wird.
                    
                - Klemmt Alpha auf `[0.0, 1.0]`.
                    
            - `pub fn blend(&self, background: &Color) -> Color`
                - Führt Alpha-Blending ("source-over") dieser Farbe über einer Hintergrundfarbe durch.
                    
                - Formel: `C_out = C_fg * α_fg + C_bg * α_bg * (1 - α_fg)`. `α_out = α_fg + α_bg * (1 - α_fg)`. Annahme: Farben sind nicht vormultipliziert.
                    
            - `pub fn lighten(&self, amount: f32) -> Color`
                - Hellt die Farbe um einen Faktor amount auf (z.B. durch lineare Interpolation zu Weiß).
                    
                - Klemmt das Ergebnis auf gültige Farbwerte. amount im Bereich `[0.0, 1.0]`.
                    
            - `pub fn darken(&self, amount: f32) -> Color`
                - Dunkelt die Farbe um einen Faktor amount ab (z.B. durch lineare Interpolation zu Schwarz).
                    
                - Klemmt das Ergebnis. amount im Bereich `[0.0, 1.0]`.
                    
        6. 3.1.5. Trait Implementierungen:
            - `impl PartialEq for Color { ... }`
                - PartialEq: Verwendet den Standard-Float-Vergleich.
                    
                - Für präzisere Vergleiche könnten benutzerdefinierte Implementierungen mit Epsilon erforderlich sein, dies wird jedoch für die Kernschicht als unnötige Komplexität betrachtet.
                    
            - `impl Default for Color { ... }`
                - Default: Implementiert Default manuell, um `Color::TRANSPARENT` zurückzugeben.
                    
            - `impl Send for Color {}`
                
            - `impl Sync for Color {}`
                
4. 4. Spezifikation: Allgemeine Enumerationen (enums.rs) Diese Datei enthält häufig verwendete, einfache Enumerationen.
        
    5. 4.1. Enum: `Orientation`
        1. 4.1.1. Definition und Zweck: Repräsentiert eine horizontale oder vertikale Ausrichtung, häufig verwendet in UI-Layouts und Widgets.
            
        2. 4.1.2. Varianten:
            - `Horizontal`
                
            - `Vertical`
                
        3. 4.1.3. Methoden:
            - `pub fn toggle(&self) -> Self`
                - Gibt die jeweils andere Orientierung zurück (Horizontal -> Vertical, Vertical -> Horizontal).
                    
        4. 4.1.4. Trait Implementierungen:
            - `impl Default for Orientation { fn default() -> Self { Orientation::Horizontal } }` (Standard ist Horizontal).
                
            - `impl Send for Orientation {}`
                
            - `impl Sync for Orientation {}`
                
5. 5. Zusammenfassung: Standard Trait Implementierungen Die folgende Tabelle gibt einen Überblick über die Implementierung gängiger Standard-Traits für die in diesem Modul definierten Typen. Dies dient als
        

 Quellen
---

**2. Schnittstelle: Domänenschicht (`novade-domain`) zu Systemschicht und UI-Schicht**

**2.1. Service-APIs (Traits) – Ultra-Fein-Definitionen (Beispiele):**

- **`domain::theming::service::ThemingService`** (Umbenannt von `ThemingEngine` für Klarheit als Trait)
    
    Rust
    
    ```
    use crate::core::types::{Color as CoreColor}; // novade-core Typen
    use super::types::{AppliedThemeState, ThemeIdentifier, ColorSchemeType, ThemingConfiguration, ThemeDefinition};
    use super::errors::ThemingError;
    use tokio::sync::broadcast;
    use uuid::Uuid;
    
    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)] // Für Event und Cache
    pub struct ThemeChangedEvent {
        pub event_id: Uuid,
        pub timestamp: DateTime<Utc>,
        pub new_state: AppliedThemeState,
    }
    
    #[async_trait::async_trait]
    pub trait ThemingService: Send + Sync {
        /// Gibt den aktuell angewendeten Theme-Zustand zurück.
        async fn get_current_theme_state(&self) -> Result<AppliedThemeState, ThemingError>;
    
        /// Gibt eine Liste aller verfügbaren (geladenen) Theme-Definitionen zurück.
        async fn get_available_themes(&self) -> Result<Vec<ThemeDefinition>, ThemingError>;
    
        /// Gibt die aktuell vom Benutzer gewählte Theming-Konfiguration zurück.
        async fn get_current_theming_configuration(&self) -> Result<ThemingConfiguration, ThemingError>;
    
        /// Aktualisiert die Theming-Konfiguration des Benutzers.
        /// Löst eine Neuberechnung des `AppliedThemeState` und ein `ThemeChangedEvent` aus, falls sich der Zustand ändert.
        async fn update_theming_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>;
    
        /// Lädt alle Theme- und Token-Definitionen von den konfigurierten Pfaden neu.
        /// Dies kann z.B. nützlich sein, wenn der Benutzer neue Themes installiert.
        /// Aktualisiert den `AppliedThemeState` basierend auf der aktuellen Konfiguration.
        async fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>;
    
        /// Abonniert Änderungen am angewendeten Theme-Zustand.
        fn subscribe_to_theme_changes(&self) -> broadcast::Receiver<ThemeChangedEvent>;
    }
    ```
    
    - **Publisher von `ThemeChangedEvent`**: `DefaultThemingService` (die Implementierung).
    - **Subscriber von `ThemeChangedEvent`**:
        - `ui::theming_gtk::ThemeManagerGtk`: Aktualisiert GTK CSS Provider.
        - Andere UI-Komponenten, die direkt auf Theme-Änderungen reagieren müssen (z.B. benutzerdefinierte Widgets, die Farben nicht über CSS beziehen).
        - Ggf. `system::compositor` wenn serverseitige Dekorationen stark vom Theme abhängen.
- **`domain::global_settings_and_state_management::service::GlobalSettingsService`**
    
    Rust
    
    ```
    use super::types::GlobalDesktopSettings;
    use super::paths::SettingPath;
    use super::errors::GlobalSettingsError;
    use tokio::sync::broadcast;
    use uuid::Uuid;
    use serde_json::Value as JsonValue;
    
    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    pub struct SettingChangedEvent {
        pub event_id: Uuid,
        pub timestamp: DateTime<Utc>,
        pub path: SettingPath,
        pub old_value: Option<JsonValue>, // Kann None sein, wenn der vorherige Wert nicht verfügbar war
        pub new_value: JsonValue,
    }
    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    pub struct SettingsLoadedEvent { /* ... */ pub settings: GlobalDesktopSettings }
    #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
    pub struct SettingsSavedEvent { /* ... */ }
    
    
    #[async_trait::async_trait]
    pub trait GlobalSettingsService: Send + Sync {
        /// Lädt die globalen Einstellungen vom Persistenz-Provider.
        /// Löst ein `SettingsLoadedEvent` aus.
        async fn load_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsError>;
    
        /// Speichert die aktuellen globalen Einstellungen über den Persistenz-Provider.
        /// Löst ein `SettingsSavedEvent` aus.
        async fn save_settings(&self) -> Result<(), GlobalSettingsError>;
    
        /// Gibt einen Klon der aktuell geladenen `GlobalDesktopSettings` zurück.
        async fn get_current_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsError>;
    
        /// Aktualisiert einen spezifischen Einstellungswert über seinen `SettingPath`.
        /// Validiert den neuen Wert.
        /// Löst ein `SettingChangedEvent` aus und persistiert die Änderungen.
        async fn update_setting(&self, path: SettingPath, new_value: JsonValue) -> Result<(), GlobalSettingsError>;
    
        /// Liest den Wert einer spezifischen Einstellung.
        async fn get_setting_value(&self, path: &SettingPath) -> Result<JsonValue, GlobalSettingsError>;
    
        /// Setzt alle Einstellungen auf ihre Standardwerte zurück.
        /// Löst mehrere `SettingChangedEvent`s aus und persistiert.
        async fn reset_all_settings_to_defaults(&self) -> Result<(), GlobalSettingsError>;
    
        /// Setzt eine spezifische Einstellung auf ihren Standardwert zurück.
        async fn reset_setting_to_default(&self, path: &SettingPath) -> Result<(), GlobalSettingsError>;
    
        /// Abonniert Änderungen an spezifischen Einstellungen oder allen Einstellungen.
        fn subscribe_to_setting_changes(&self) -> broadcast::Receiver<SettingChangedEvent>;
        // fn subscribe_to_settings_loaded_event(...) -> broadcast::Receiver<SettingsLoadedEvent>;
        // fn subscribe_to_settings_saved_event(...) -> broadcast::Receiver<SettingsSavedEvent>;
    }
    ```
    
    - **Publisher von Events**: `DefaultGlobalSettingsService`.
    - **Subscriber von `SettingChangedEvent`**:
        - **Domäne:** `ThemingService` (für Theme-relevante Einstellungen), `WorkspaceManagerService` (für Workspace-bezogene Einstellungen), `NotificationRulesEngine` (für `SettingIsTrue`-Bedingungen), `PowerManagementService` (für Energieeinstellungen).
        - **System:** `system::input` (für Tastatur-Layout, Maus-Sensitivität), `system::compositor::output_management` (für Standard-Auflösung/Skalierung), `system::audio_management` (für Standard-Lautstärke).
        - **UI:** `ui::control_center` (um sich selbst zu aktualisieren), diverse `ui::shell`-Widgets, die Einstellungen direkt darstellen oder verwenden.

**2.2. Nutzung durch die Systemschicht (Ultra-Fein):**

- **`system::compositor::core::state::DesktopState::new(...)`** nimmt `Arc<dyn WindowManagementPolicyService>` und `Arc<dyn WorkspaceManagerService>` als Parameter.
- **`system::compositor::xdg_shell::handlers::XdgShellHandler::new_toplevel()` (für `DesktopState`)**:
    1. Erhält `ToplevelSurface` von Smithay.
    2. Erzeugt eine `DomainWindowIdentifier`.
    3. Ruft `self.workspace_manager_service.get_active_workspace_id().await` (oder eine äquivalente synchrone Methode, wenn im Compositor-Thread).
    4. Ruft `self.window_policy_service.get_initial_window_geometry(domain_id, workspace_id, available_area, ...).await` um die initiale Geometrie und den Zustand zu bestimmen.
    5. Konfiguriert die `ToplevelSurface` entsprechend.
    6. Informiert `self.workspace_manager_service.assign_window_to_workspace(workspace_id, domain_id, ...).await`.
- **`system::dbus_interfaces::notifications_server::FreedesktopNotificationsServer::notify()`**:
    1. Empfängt D-Bus-Parameter.
    2. Konvertiert diese in eine `domain::user_centric_services::notifications_core::types::NotificationInput`-Struktur.
    3. Ruft `self.notification_service_domain_handle.lock().await.post_notification(input).await`.
    4. Konvertiert das Ergebnis (Domain `Uuid`) in eine D-Bus `u32` ID.
- **`system::input::event_dispatcher::process_input_event()` für `InputEvent::GestureSwipeUpdate`**:
    1. Ermittelt die Geste.
    2. Ruft `self.desktop_state.workspace_manager_service.switch_to_next_or_prev_workspace(direction).await`.

**2.3. Nutzung durch die UI-Schicht (Ultra-Fein):**

- **`ui::control_center::appearance_page::AppearancePageModel::new()`**:
    1. Nimmt `Arc<dyn GlobalSettingsService>` und `Arc<dyn ThemingService>` als Parameter.
    2. Abonniert `global_settings_service.subscribe_to_setting_changes()` und `theming_service.subscribe_to_theme_changes()`.
    3. Ruft `global_settings_service.get_current_settings().await` und `theming_service.get_current_theming_configuration().await` sowie `get_available_themes().await`, um den initialen Zustand der UI-Elemente (ComboBoxes für Themes, Switches für Dark Mode etc.) zu füllen.
- **Event-Handler in `AppearancePageModel` für `SettingChangedEvent`**:
    1. Prüft, ob `event.path` für die Appearance-Seite relevant ist.
    2. Aktualisiert das entsprechende UI-Element (z.B. einen `gtk::Switch` für `SettingPath::Appearance(AppearanceSettingPath::PreferDarkTheme)`) mit `event.new_value`.
- **Aktion in `AppearancePageModel` (z.B. Klick auf "Akzentfarbe ändern"):**
    1. Öffnet einen Farbauswahldialog.
    2. Nach Auswahl: `new_color: core::types::Color`.
    3. `current_theming_config = self.theming_service.get_current_theming_configuration().await?;`
    4. `updated_config = current_theming_config.with_new_accent(new_color);`
    5. `self.theming_service.update_theming_configuration(updated_config).await?;` (Löst `ThemeChangedEvent` aus).
- **`ui::shell::workspace_switcher::WorkspaceSwitcherWidget::on_active_workspace_changed_event(event: domain::workspaces::manager::events::ActiveWorkspaceChangedData)`**:
    1. Iteriert über seine `SpaceIconWidget`s.
    2. Setzt den visuellen "aktiv"-Zustand für das Widget, das `event.new_active_id` entspricht.

---

**3. Schnittstelle: Systemschicht (`novade-system`) zu UI-Schicht (`novade-ui`)**

Diese Schnittstelle ist oft ereignisbasiert oder durch von der UI-Schicht gehaltene Handles zu System-Services.

**3.1. Systemereignisse und -zustände (Ultra-Fein):**

- **Fensterinformationen (bereitgestellt durch einen `WindowManagerSystemFacade` Trait, implementiert von einer Struktur, die auf `DesktopState` zugreift):**
    - **Trait `WindowManagerSystemFacade`**:
        
        Rust
        
        ```
        use crate::core::types::RectInt;
        use crate::domain::workspaces::core::types::{WindowIdentifier as DomainWindowIdentifier, WorkspaceId};
        use crate::system::compositor::xdg_shell::types::ManagedWindowPublicState; // Neuer Typ
        
        #[derive(Debug, Clone, Serialize, Deserialize)] // Für UI-Nutzung
        pub struct ManagedWindowPublicState { // Daten, die UI benötigt
            pub domain_id: DomainWindowIdentifier,
            pub app_id: Option<String>,
            pub title: Option<String>,
            pub geometry: RectInt, // Relative Workspace-Koordinaten
            pub workspace_id: Option<WorkspaceId>,
            pub is_mapped: bool,
            pub is_focused: bool, // Bezogen auf den Seat der UI-Shell
            pub is_maximized: bool,
            pub is_fullscreen: bool,
            // Ggf. Icon-Daten oder Name
        }
        
        #[async_trait::async_trait]
        pub trait WindowManagerSystemFacade: Send + Sync {
            /// Gibt eine Liste aller aktuell verwalteten (gemappten) Fenster mit ihren relevanten Zuständen zurück.
            async fn get_all_managed_windows(&self) -> Result<Vec<ManagedWindowPublicState>, SystemFacadeError>;
        
            /// Gibt den Zustand eines spezifischen Fensters zurück.
            async fn get_window_state(&self, id: &DomainWindowIdentifier) -> Result<Option<ManagedWindowPublicState>, SystemFacadeError>;
        
            /// Fordert den Fokus für ein bestimmtes Fenster an.
            async fn request_focus(&self, id: &DomainWindowIdentifier) -> Result<(), SystemFacadeError>;
        
            /// Fordert das Schließen eines Fensters an (sendet Close-Request an Client).
            async fn request_close_window(&self, id: &DomainWindowIdentifier) -> Result<(), SystemFacadeError>;
        
            // Weitere Aktionen: request_minimize, request_maximize, request_move_to_workspace etc.
        
            /// Abonniert Änderungen an Fensterzuständen.
            fn subscribe_to_window_changes(&self) -> tokio::sync::broadcast::Receiver<WindowSystemEvent>;
        }
        
        #[derive(Debug, Clone)] // System-intern oder für UI, Serialize/Deserialize falls nötig
        pub enum WindowSystemEvent {
            WindowCreated(ManagedWindowPublicState),
            WindowClosed(DomainWindowIdentifier),
            WindowUpdated(ManagedWindowPublicState), // Geometrie, Titel, Fokus etc. geändert
            FocusChanged(Option<DomainWindowIdentifier> /* neues Fokusfenster */, Option<DomainWindowIdentifier> /* altes Fokusfenster */),
        }
        // SystemFacadeError definieren
        ```
        
    - **Implementierung:** Eine Struktur in `novade-system` implementiert diesen Trait. Sie greift auf `system::compositor::core::DesktopState` (via `Arc<Mutex<...>>` oder `LoopHandle::insert_idle`) zu, um die Daten der `ManagedWindow`s zu lesen und in `ManagedWindowPublicState` zu transformieren. Sie publiziert `WindowSystemEvent`s über den `SystemEventBridge`.
- **Eingabeereignisse:** Werden nicht direkt als Rust-Strukturen an die UI-Schicht gesendet. Stattdessen:
    - **Wayland-Protokoll:** Der `system::compositor` sendet `wl_keyboard.key`, `wl_pointer.motion`, `wl_touch.down` etc. an die Wayland-Oberflächen der UI-Elemente (GTK-Fenster). GTK4 (`gdk::Display`) empfängt diese Wayland-Events und übersetzt sie in GTK-Events (`GdkEvent`), die dann von Widgets verarbeitet werden.
    - **Spezielle Aktionen:** Globale Hotkeys oder Gesten, die nicht an ein spezifisches Fenster gebunden sind (z.B. `Super+Space` für Command Palette, Workspace-Switch-Gesten), werden vom `system::input` erkannt. Der `system::input` kann dann direkt eine Domänenaktion auslösen (z.B. `domain::command_palette_service::show()`) oder ein spezifisches `SystemLayerEvent` über den `SystemEventBridge` senden, das von der UI-Shell abonniert wird.
- **Status von Systemdiensten:**
    - `system::dbus_interfaces::upower_client::UPowerClientService` publiziert `UPowerEvent`s über den `SystemEventBridge`.
    - Die UI (z.B. `ui::shell::PowerIndicatorWidget`) abonniert `SystemLayerEvent::UPower` vom `SystemEventBridge`.
    - Bei Empfang: `match event { SystemLayerEvent::UPower(UPowerEvent::DeviceChanged(path)) => { let details = block_on(self.upower_client_facade.get_device_details_by_path(path)); /* UI aktualisieren */ } ... }`

**3.2. Ausführung von UI-Befehlen (Ultra-Fein):**

- **UI-Schicht (`ui::shell::window_title_bar::CloseButton::on_clicked`)**:
    1. Ruft `self.window_manager_facade.request_close_window(self.associated_window_id).await;`.
- **Systemschicht (`DefaultWindowManagerSystemFacade::request_close_window`)**:
    1. Sperrt `DesktopState`.
    2. Findet das `ManagedWindow` (als `Arc<smithay::desktop::Window>`) für die `DomainWindowIdentifier`.
    3. Wenn `WindowSurface::Toplevel(toplevel)`: `toplevel.send_close();`.
    4. (Der Client sollte dann unmappen; `XdgShellHandler::unmap_toplevel` wird ausgelöst).
- **UI-Schicht (`ui::control_center::display_page::ApplyButton::on_clicked`)**:
    1. Sammelt neue Monitorkonfiguration (`Vec<MonitorConfigUiData>`).
    2. Konvertiert zu `Vec<system::output_management::types::OutputConfigRequestData>`.
    3. Ruft `self.output_management_facade.apply_output_configuration(requests).await;`.
- **Systemschicht (`DefaultOutputManagementFacade::apply_output_configuration`)**:
    1. Interagiert mit `system::compositor::output_management` (oder `DesktopState`), um `wlr-output-management` Protokoll-Requests zu senden oder direkt DRM-Modi zu setzen.

---

**Zusammenfassende Prinzipien der Schnittstellen (Ultra-Fein):**

1. **Typsicherheit und explizite Datenstrukturen:** Alle über Schnittstellen ausgetauschten Daten (Parameter, Rückgabewerte, Event-Payloads) sind explizit definierte Rust-`struct`s oder `enum`s. Primitive Typen werden verwendet, wo angemessen, aber für komplexere Daten werden spezifische Typen bevorzugt.
2. **Trait-basierte Service-Abstraktion:** Domänen- und Systemdienste, die von höheren Schichten genutzt werden, implementieren `#[async_trait::async_trait]` Traits. Dies ermöglicht Mocking für Tests und Flexibilität bei Implementierungsänderungen. Handles zu diesen Services sind typischerweise `Arc<dyn ServiceTrait>`.
3. **Granulare Events:** Events sind spezifisch und enthalten alle notwendigen Daten, um Subscribern eine sinnvolle Reaktion zu ermöglichen, ohne zusätzliche Abfragen tätigen zu müssen (außer bei Bedarf für sehr große Datenmengen). Jedes Event hat eine eindeutige ID (`Uuid`) und einen Zeitstempel (`DateTime<Utc>`).
4. **Fehler-Wrapping und -Kontext:** Fehler werden über Schichten hinweg mit `#[from]` oder `#[source]` gewrappt, um die ursprüngliche Ursache nachvollziehbar zu halten. Fehlertypen enthalten kontextspezifische Informationen (z.B. Pfade, IDs).
5. **Asynchronität für potenziell blockierende Operationen:** Alle Operationen, die I/O beinhalten oder länger dauern könnten (insbesondere in Domänen- und Systemschicht-APIs), sind `async`. Die UI-Schicht verwendet `tokio::spawn` oder `glib::MainContext::spawn_local` um diese `async` Funktionen aufzurufen, ohne die UI zu blockieren.
6. **Threadsicherheit:** Geteilter Zustand in Service-Implementierungen wird mit `Arc<tokio::sync::Mutex<...>>` oder `Arc<tokio::sync::RwLock<...>>` geschützt.
7. **Kommunikation mit dem Compositor-Thread:** Operationen, die direkten Zugriff auf `DesktopState` (oder andere nicht-`Send` Smithay-Strukturen) benötigen und von `async` Kontexten außerhalb des Compositor-`calloop`-Threads ausgelöst werden, müssen sorgfältig gehandhabt werden:
    - **Option A (Bevorzugt für kurze Operationen):** `LoopHandle::insert_idle()` oder `LoopHandle::send_event()` um eine Funktion/einen Task im `calloop`-Thread auszuführen.
    - **Option B (Für komplexe Interaktionen):** Dedizierte `mpsc` oder `oneshot` Kanäle zwischen `async` Tasks und dem Compositor-Thread.
    - **Option C (Wenn Daten `Send` sind):** `Arc<Mutex<TeilVonDesktopState>>`, wobei der Mutex `parking_lot::Mutex` sein kann für blockierende Operationen aus dem Compositor-Thread und `tokio::sync::Mutex` für `async` Tasks.

Diese ultra-feingranulare Definition der Schnittstellen ist der Schlüssel zu einer disziplinierten Entwicklung. Jede Interaktion ist spezifiziert, was die Integration der Schichten erleichtert und die Testbarkeit verbessert.

**1. Schnittstelle: Kernschicht (Core Layer) zu höheren Schichten**

- **1.1.1. `core::types` Schnittstelle:**
    - **Exakte Typdefinitionen referenzieren:** Verlinken Sie oder wiederholen Sie die exakten Rust-Definitionen von `Point<T>`, `Size<T>`, `Rect<T>`, `RectInt`, `Color`, `Orientation` aus `A1 Kernschicht.md` / `1. Core-Details.md` / `Infrastruktur ( UI fehlt).md`. Inklusive aller abgeleiteten Traits (`Debug`, `Clone`, `Serialize`, etc.) und implementierten Methoden mit Signaturen.
    - **Nutzungsbeispiele präzisieren:** "Domänenschicht verwendet `Color` für `TokenValue::Color(String)` und `AccentColor.value: CoreColor`."
- **1.1.2. `core::errors` Schnittstelle:**
    - **Exakte Fehlerdefinition referenzieren:** Verlinken oder wiederholen Sie die Definition von `CoreError` und `ColorParseError` aus `A2 Kernschicht.md` / `A3 Kernschicht Fehlerbehandlung.md` / `A4 Kernschicht.md`. Inklusive aller Varianten, `#[error(...)]`-Attribute und `#[source]` / `#[from]`-Annotationen.
    - **Fehler-Wrapping-Regel:** "JEDE höhere Schicht, die eine Funktion aus der Kernschicht aufruft, die `Result<_, CoreError>` (oder einen spezifischeren Kernfehler wie `ConfigError`) zurückgibt, MUSS diesen Fehler entweder:
        - Explizit behandeln (match).
        - Oder in ihren eigenen Fehlertyp wrappen. Diese Wrapper-Variante in den höheren Schichten MUSS `#[from] CoreError` oder `#[source] source: CoreError` verwenden, um die Fehlerkette zu erhalten."
- **1.1.3. `core::logging` Schnittstelle:**
    - **Exakte Signatur:** `pub fn initialize_logging(level_filter: tracing::LevelFilter, format: LogFormat) -> Result<(), LoggingError>` (aus `A4 Kernschicht.md`). Die Enums `LogFormat` und `LoggingError` MÜSSEN hier ebenfalls exakt definiert werden.
    - **Verwendungsrichtlinie:** "Alle Module in allen Schichten MÜSSEN die Makros `tracing::{trace, debug, info, warn, error}` für Logging verwenden. Sensible Daten DÜRFEN NICHT geloggt werden."
- **1.1.4. `core::config` Schnittstelle:**
    - **Exakte Signaturen und Strukturen referenzieren:** `load_core_config(...) -> Result<CoreConfig, ConfigError>`, `get_global_core_config() -> &'static CoreConfig`, `initialize_global_core_config(config: CoreConfig) -> Result<(), ConfigError>` (aus `A4 Kernschicht.md`). Die Struktur `CoreConfig` (und alle ihre Unterstrukturen wie `LogLevelConfig`, `FeatureFlags`) und der `ConfigError`-Enum MÜSSEN hier exakt spezifiziert werden.
    - **Suchreihenfolge für `load_core_config`:** (Wie in meiner vorherigen Analyse vorgeschlagen) "1. `custom_path` (falls `Some`), 2. `$XDG_CONFIG_HOME/novade/core.toml`, 3. `/etc/novade/core.toml`. Wenn keine Datei gefunden wird, wird `ConfigError::NoConfigurationFileFound` zurückgegeben. Die aufrufende Schicht (typischerweise beim Anwendungsstart) MUSS dann `CoreConfig::default()` verwenden und dies loggen."
- **1.1.5. `core::utils` Schnittstelle:**
    - Für JEDE geplante Utility-Funktion: Exakte Signatur, Parameter, Rückgabetyp (`Result<_, UtilityXYError>`), Vor-/Nachbedingungen und eine kurze Beschreibung ihrer Logik. Jede `UtilityXYError` muss definiert sein.

**2. Schnittstelle: Domänenschicht (Domain Layer) zu System- und UI-Schicht**

Hier wird es komplexer, da die Domänenschicht viele Services bereitstellt.

- **Für jeden Domänen-Service-Trait (z.B. `ThemingEngine`, `WorkspaceManagerService`, `AIInteractionLogicService`, `NotificationService`, `GlobalSettingsService`, `WindowManagementPolicyService`):**
    
    - **Exakte Trait-Definition:** Alle Methoden mit exakten Signaturen (`async fn ...`), Parameter-Typen (die bereits in `core::types`, `domain::shared_types` oder den `types.rs` des jeweiligen Domänenmoduls definiert sein müssen) und Rückgabetypen (`Result<OkTyp, DomänenModulFehlerTyp>`).
    - **Datenstrukturen der API:** Alle Structs und Enums, die in den Methodensignaturen als Parameter oder Rückgabewerte verwendet werden (z.B. `AppliedThemeState`, `WorkspaceWindowLayout`, `AIInteractionContext`, `NotificationInput`), MÜSSEN exakt definiert werden (Felder, Typen, Ableitungen). Verweisen Sie auf die Moduldokumente (B1-B4, "2. Domain-Details.md").
    - **Events von diesem Service:**
        - Für jeden Event-Typ (z.B. `ThemeChangedEvent`, `WorkspaceEvent::WorkspaceCreated`, `NotificationEventEnum::NotificationPosted`): Die exakte Payload-Struktur muss definiert sein.
        - **Event-Bus-Spezifikation:** "Jeder Domänen-Service, der Events publiziert, hält einen `tokio::sync::broadcast::Sender<EventType>`. Die `subscribe_to_xxx_events()`-Methode des Services gibt einen `tokio::sync::broadcast::Receiver<EventType>` zurück. Die Kapazität des Broadcast-Kanals wird beim Erstellen des Service festgelegt (z.B. `new(..., broadcast_capacity: usize)`)."
    - **Fehlertypen dieses Services:** Der spezifische Fehler-Enum des Domänenmoduls (z.B. `ThemingError`, `WorkspaceManagerError`) MUSS hier als Teil der Schnittstelle explizit genannt werden.
    - **Beispielhafte Nutzung durch System-/UI-Schicht präzisieren:**
        - "Systemschicht: Der `system::mcp_client::service::DefaultSystemMcpService` ruft `ai_interaction_service.get_consent_status_for_interaction(...)` auf. Der zurückgegebene `AIInteractionError` wird in einen `SystemMcpError::DomainInteractionFailed` gewrappt."
        - "UI-Schicht: Das `ui::shell::PanelWidget` (oder dessen Submodule) abonniert `ThemingEngine::subscribe_to_theme_changes()` und erhält `ThemeChangedEvent`. Bei Empfang wird `ui::theming_gtk::apply_resolved_tokens(event.new_state.resolved_tokens)` aufgerufen."
- **Spezifische Ultra-Feinspezifikation für `domain::user_centric_services::notification_service_impl` und `core::config::loader` (aus Ihrem Plan):**
    
    - Diese Detailtiefe ist exzellent und genau das, was für _alle_ Module und deren öffentliche APIs (und internen, wenn für die Schnittstelle relevant) benötigt wird. Ihr Plan, dies für ausgewählte Module zu tun, ist gut. Dieser Detailgrad muss auf die _Schnittstellenbeschreibung zwischen den Schichten_ erweitert werden.
    - **Inkonsistenz-Beispiel und Lösung im Plan:**
        - `notification_api.rs` definiert `replaces_id: Option<NotificationId>` für `post_notification`.
        - Die D-Bus-Schnittstelle (`notification_dbus_service.rs`) nimmt `replaces_id_dbus: u32`.
        - Die Schnittstellendokumentation muss das **Mapping** spezifizieren: "Der `system::dbus::notification_dbus_service` MUSS `replaces_id_dbus: u32` in `Option<NotificationId>` für den Aufruf von `NotificationService::post_notification` umwandeln. Wenn `replaces_id_dbus == 0`, dann ist das Ergebnis `None`. Andernfalls MUSS der `IdMapper` konsultiert werden, um die `u32` in eine `NotificationId` (Uuid) zu konvertieren. Wenn keine Uuid für die `u32` existiert, ist dies ein `NotificationDBusError::InvalidReplacesId`."
        - Ähnlich für die Rückgabe der neuen `NotificationId` (Uuid) als `u32` D-Bus-ID.

**3. Schnittstelle: Systemschicht (System Layer) zu Benutzeroberflächenschicht (UI Layer)**

- **Für jede von der Systemschicht bereitgestellte Information/Funktion:**
    - **Exakter API-Aufruf (oft ein Trait, den die Systemschicht implementiert und die UI-Schicht nutzt) oder Event:**
        - **Fensterinformationen:** `system::compositor_facade::CompositorViewService::get_visible_windows_for_space(space_id: WorkspaceId) -> Result<Vec<WindowViewData>, SystemError>`. Die Struktur `WindowViewData { id: DomainWindowIdentifier, title: String, app_id: ApplicationId, rect: RectInt, has_focus: bool, decoration_mode: DecorationMode }` muss definiert sein.
        - **Eingabeereignisse:** "Die UI-Schicht registriert Callback-Handler bei `system::input::InputManager::register_keyboard_event_handler(Box<dyn Fn(KeyEventData) + Send>)`. Die Struktur `KeyEventData { keysym: u32, utf8: Option<String>, modifiers: ModifiersState, state: KeyState }` muss definiert sein." (Alternativ: Systemschicht sendet Events über den `SystemEventBridge`).
        - **Status von Systemdiensten:** Z.B. `system::dbus::upower_client_service::UPowerService::get_current_power_state() -> Result<PowerStateData, SystemError>`.
    - **Fehlertypen:** Jeder Service der Systemschicht, den die UI aufruft, gibt einen spezifischen Systemfehler zurück.
    - **Event-Strukturen und Event-Bus:** Wenn die Systemschicht Events an die UI sendet (z.B. `OutputConfigurationChangedEvent`), müssen diese Events und der Bus spezifiziert werden (wahrscheinlich der `SystemEventBridge`).

**4. Allgemeine Kommunikationsmuster (aus Ihrem Plan)**

Diese sind gut, aber müssen für die KI konkretisiert werden:

- **Synchrone Aufrufe:** Nur innerhalb eines Threads oder wenn der aufgerufene Code garantiert nicht blockiert. Für schichtübergreifende Aufrufe selten, außer für einfache Getter auf bereits gecachte/gehaltene Daten.
- **Asynchrone Operationen:** Alle schichtübergreifenden Aufrufe, die I/O oder potenziell längere Berechnungen beinhalten, MÜSSEN `async` sein und `Result<_, _>` zurückgeben.
- **Event-Broadcasting:** Der `SystemEventBridge` (für System-interne und System-zu-UI Events) und die `tokio::sync::broadcast`-Kanäle in Domänen-Services sind die festgelegten Mechanismen.
    - _Spezifizieren:_ Welche Events werden von welchem Modul über welchen Sender/Kanal gesendet? Wer sind die erwarteten Subscriber?
- **Fehlerpropagation:** Die Regel "Die `source()`-Kette MUSS erhalten bleiben" ist kritisch.

**Inkonsistenzen erkennen und beheben durch diesen detaillierten Plan:**

Indem Sie jede Schnittstelle so detailliert definieren, werden Inkonsistenzen aufgedeckt:

1. **Typ-Mismatch:** Wenn `domain::theming` eine `core::types::Color` erwartet, aber `ui::control_center` versucht, einen `String` zu senden, wird dies beim Definieren der `GlobalSettingsService::update_setting`-Signatur und der `SettingPath`-Logik klar.
2. **Fehlende Fehlerbehandlung:** Wenn eine Funktion `Result` zurückgibt, aber die aufrufende Schicht den `Err`-Fall nicht behandelt oder nicht in ihren eigenen Fehlertyp mapped, wird dies beim Definieren des Fehler-Mappings auffallen.
3. **Semantische Unterschiede:** (Wie beim `replaces_id`-Beispiel oben) Die D-Bus-Spezifikation verwendet `u32`, die Domäne `Option<Uuid>`. Die Schnittstellendefinition muss die Transformation explizit machen.
4. **Fehlende Daten:** Wenn die UI-Schicht eine bestimmte Information zum Darstellen eines Fensters benötigt (z.B. "minimiert"-Status), aber die von `system::compositor` bereitgestellte `WindowViewData`-Struktur dieses Feld nicht enthält, wird die Lücke offensichtlich.
5. **Unklare Verantwortlichkeiten für Zustandsänderungen:** Wenn sowohl die Domäne als auch die Systemschicht versuchen, denselben Zustand zu modifizieren, ohne klare Koordination, wird dies bei der Definition der API-Aufrufe und Event-Flüsse sichtbar.
# Detaillierte Analyse und Implementierungsstrategien für die Systemschicht von NovaDE

Dieser Bericht analysiert die Kernkomponenten der Systemschicht (novade-system) der Linux-Desktop-Umgebung NovaDE. Er fokussiert auf bewährte Verfahren, typische Fallstricke und Lösungen für potenzielle Implementierungsprobleme unter Verwendung der spezifizierten Technologien wie Smithay, libinput, zbus und PipeWire.

## 1. Smithay Compositor-Entwicklung (system::compositor)

Die Entwicklung eines Wayland-Compositors mit Smithay erfordert eine sorgfältige Planung des Zustandsmanagements, der Fensterrepräsentation und der Protokollimplementierungen.

### 1.1. Zustandsmanagement in `DesktopState`

Die `DesktopState`-Struktur ist das Herzstück des Compositors und implementiert eine Vielzahl von Smithay-Handler-Traits (z.B. `CompositorHandler`, `XdgShellHandler`, `ShmHandler`, `SeatHandler`). Dies birgt die Gefahr, dass `DesktopState` zu einem "God Object" mit übermäßiger Komplexität wird.1

**Best Practices zur Strukturierung:**

- **Modularisierung durch Komposition:** Anstatt alle Logik direkt in `DesktopState` zu implementieren, sollten spezialisierte Sub-Strukturen für zusammengehörige Funktionalitäten verwendet werden. Beispielsweise könnte eine `XdgShellState` die Logik für `xdg_shell` kapseln und von `DesktopState` referenziert werden. Smithay selbst fördert diesen Ansatz durch spezifische Zustandsstrukturen für viele Protokolle (z.B. `smithay::wayland::shell::xdg::XdgShellState`, `smithay::wayland::seat::SeatState`).2
- **Trait-basierte Abstraktion:** Für komplexe, wiederverwendbare Logik innerhalb der Handler können eigene Traits definiert und von `DesktopState` oder dessen Sub-Strukturen implementiert werden.
- **Klare Verantwortlichkeiten:** Jeder Handler-Trait sollte primär für die Interaktion mit dem entsprechenden Wayland-Protokoll zuständig sein. Anwendungslogik, die darüber hinausgeht (z.B. spezifische Fensterverwaltungsregeln), sollte in separate Module ausgelagert und von den Handlern aufgerufen werden.

**Strategien zur Verwaltung von `Arc<Mutex/RwLock<...>>`:**

Der Zugriff auf geteilte Zustände innerhalb der asynchronen Handler-Methoden erfordert Synchronisation. `Arc<Mutex<T>>` oder `Arc<RwLock<T>>` sind hierfür gängige Muster in Rust.4

- **Granularität der Locks:** Sperren sollten so feingranular wie möglich gehalten werden, um Contention zu minimieren. Anstatt den gesamten `DesktopState` zu sperren, sollten nur die tatsächlich benötigten Teile des Zustands gesperrt werden.
- **`RwLock` bevorzugen:** Für Zustände, die häufig gelesen und selten geschrieben werden, ist `RwLock` performanter als `Mutex`, da es mehrere gleichzeitige Leser erlaubt.
- **Vermeidung von Deadlocks:** Besondere Vorsicht ist bei verschachtelten Locks oder dem Aufruf von externem Code (z.B. Client-Callbacks) innerhalb eines Locks geboten. Die Reihenfolge der Lock-Akquirierung sollte konsistent sein.
- **Kurze Lock-Dauer:** Locks sollten nur so lange wie unbedingt nötig gehalten werden. Operationen, die außerhalb des kritischen Abschnitts durchgeführt werden können, sollten dies auch tun.

Smithay selbst empfiehlt, einen zentralen, mutablen Zustand über `calloop` an die Callbacks zu übergeben, was die Notwendigkeit von `Arc<Mutex<T>>` für den Hauptzustand reduzieren kann, da Callbacks sequenziell ausgeführt werden.6 Für Zustände, die von nebenläufigen Tasks (z.B. D-Bus-Handler in Tokio) modifiziert werden und vom Compositor-Thread gelesen werden müssen, bleiben `Arc<Mutex/RwLock<T>>` jedoch relevant.

**Beispiel (konzeptionell):**

Rust

```
pub struct DesktopState {
    pub compositor_state: smithay::wayland::compositor::CompositorState,
    pub xdg_shell_state: smithay::wayland::shell::xdg::XdgShellState,
    pub seat_state: smithay::wayland::seat::SeatState<Self>,
    pub space: Arc<Mutex<smithay::desktop::Space<smithay::desktop::Window>>>,
    //... weitere Zustands-Subkomponenten
}

// Implementierung der Handler-Traits für DesktopState,
// die oft auf die Sub-Zustände delegieren.
// smithay::delegate_compositor!(DesktopState);
// smithay::delegate_xdg_shell!(DesktopState);
//...
```

Die `delegate_*!` Makros von Smithay sind essentiell, um die Implementierung der Handler-Traits an die entsprechenden Zustandsstrukturen zu delegieren und Boilerplate-Code zu reduzieren.2

### 1.2. `ManagedWindow` und `smithay::desktop::Space`

`ManagedWindow` ist eine Abstraktion über Wayland-Fenster (XDG Toplevels, Popups) und XWayland-Fenster. Es implementiert das `smithay::desktop::Window`-Trait.

**Korrekte Implementierung des `Window`-Traits:**

Das `smithay::desktop::Window`-Trait 9 erfordert die Implementierung verschiedener Methoden, um die Eigenschaften und das Verhalten eines Fensters zu definieren:

- `bbox()`: Gibt die Bounding Box des Fensters inklusive aller Subsurfaces und Popups zurück.
- `is_wayland()` / `is_x11()`: Unterscheidet zwischen Wayland- und X11-Fenstern.
- `toplevel()`: Gibt eine Referenz auf die zugrundeliegende `ToplevelSurface` (für XDG-Fenster) zurück.
- `send_frame()`: Sendet Frame-Callbacks an die Oberflächen des Fensters.
- `surface_under()`: Findet die Oberfläche unter einem gegebenen Punkt.
- `set_activated()`: Setzt den Aktivierungszustand des Fensters.
- `geometry()`: Gibt die Kerngeometrie des Fensters zurück (ohne Dekorationen).
- `on_commit()`: Wird aufgerufen, wenn die zugrundeliegende Oberfläche einen Commit durchführt.

**Repräsentation von Fenstergeometrien und -zuständen:**

- **Geometrie:** Die Geometrie (`Rectangle<i32, Logical>`) wird typischerweise direkt in der `ManagedWindow`-Struktur gespeichert und bei `configure`-Events vom Client oder durch Aktionen des Compositors aktualisiert. Sie repräsentiert die von der Anwendung gewünschte Größe und Position des Fensterinhalts.
- **Zustände (maximiert, aktiv, etc.):** Diese Zustände sollten ebenfalls in `ManagedWindow` gespeichert werden (z.B. als Booleans oder Enums).
    - Der **maximierte Zustand** wird durch Anfragen des Clients (z.B. `xdg_toplevel::set_maximized`) oder Aktionen des Compositors gesetzt. Die tatsächliche Geometrie wird dann vom Compositor basierend auf dem verfügbaren Platz des Outputs angepasst.
    - Der **aktive Zustand** wird vom Compositor verwaltet (typischerweise im `SeatHandler` in Reaktion auf Benutzereingaben) und an das Fenster propagiert (z.B. über `Window::set_activated(true)`).
- **Dekorationen (CSD/SSD via `xdg-decoration`):**
    - Das `xdg-decoration-unstable-v1`-Protokoll ermöglicht die Aushandlung zwischen Client-Side Decorations (CSD) und Server-Side Decorations (SSD).12
    - `ManagedWindow` (oder eine assoziierte Struktur) muss den ausgehandelten Dekorationsmodus (`zxdg_toplevel_decoration_v1::mode`) speichern.
    - Wenn SSD aktiv ist, ist der Compositor für das Zeichnen der Dekorationen verantwortlich. `ManagedWindow` sollte Informationen bereitstellen, die für das Rendern der SSDs benötigt werden (z.B. Titel, aktive/inaktive Zustände).
    - Wenn CSD aktiv ist, zeichnet der Client die Dekorationen selbst. Der Compositor muss die Geometrie des Fensters entsprechend berücksichtigen.
    - Die `XdgDecorationState` und `XdgDecorationHandler` von Smithay helfen bei der Implementierung dieses Protokolls.15 Die `XdgDecorationHandler::request_mode` und `new_decoration` Methoden sind hierbei zentral. Der Server (Compositor) sendet `configure`-Events an den `zxdg_toplevel_decoration_v1`, um den Modus festzulegen. Der Client antwortet mit `set_mode` oder `unset_mode`.

**Synchronisation mit `XdgToplevelSurfaceData`:**

`XdgToplevelSurfaceData` 3 speichert den Zustand, der spezifisch für eine `xdg_toplevel`-Oberfläche ist (z.B. angeforderte Größe, Zustände wie maximiert, minimiert, fullscreen, App-ID, Titel).

- Wenn der Client Zustandsänderungen anfordert (z.B. `set_title`, `set_maximized`), werden diese im `XdgToplevelSurfaceData` (oft im `pending_state`) gespeichert.
- Bei einem `wl_surface.commit` werden diese Änderungen durch den `XdgShellHandler` verarbeitet.
- Die Logik im `XdgShellHandler` (oder in `ManagedWindow::on_commit`) muss dann den Zustand von `ManagedWindow` mit den Daten aus `XdgToplevelSurfaceData` synchronisieren. Beispielsweise wird ein `xdg_toplevel.set_maximized` im `XdgToplevelSurfaceData` vermerkt, und der Compositor entscheidet dann, ob und wie er das Fenster maximiert und sendet entsprechende `configure`-Events. Die `send_configure`-Methode auf `ToplevelSurface` ist hierbei zentral, um dem Client die neue Konfiguration mitzuteilen.20 Der Client muss diese Konfiguration mit `ack_configure` bestätigen.

**Interaktion mit `Space`:**

Der `smithay::desktop::Space` 9 verwaltet die Anordnung von Fenstern und anderen Elementen im 2D-Raum.

- `Space::map_window(window, location, activate)`: Fügt ein `ManagedWindow` zum `Space` an einer bestimmten `location` hinzu und legt die Stapelreihenfolge fest. `activate` gibt an, ob das Fenster aktiviert werden soll.
- `Space::unmap_window(window)`: Entfernt ein Fenster aus dem `Space`.
- `Space::raise_window(window, activate)`: Bringt ein Fenster in der Stapelreihenfolge nach vorne.
- **Schadensverfolgung (Damage Tracking):**
    - `Space::damage_window(window, new_damage, old_damage)`: Wird verwendet, um dem `Space` mitzuteilen, dass sich Bereiche eines Fensters geändert haben. `new_damage` sind die beschädigten Regionen in Fensterkoordinaten. `old_damage` kann verwendet werden, wenn sich die Position oder Größe des Fensters geändert hat.
    - `Space::damage_output(output, damage)`: Markiert einen Bereich auf einem bestimmten Output als beschädigt.
    - Wenn ein Client einen neuen Puffer committet (`wl_surface.commit`), wird die `SurfaceAttributes.damage`-Region aktualisiert.26 Der Compositor muss diese Information dann an den `Space` weiterleiten, typischerweise über `damage_window`.
    - `render_output` verwendet die akkumulierten Schadensinformationen, um nur die notwendigen Bereiche eines Outputs neu zu zeichnen und so die Performance zu optimieren.9

### 1.3. Wayland Protokoll-Implementierung (spezifisch)

**1.3.1. `wlr-output-management-unstable-v1`** 6

Dieses Protokoll ermöglicht Clients (z.B. Display-Konfigurationstools wie `kanshi` 34 oder `wdisplays` 33) das Abfragen und Ändern der Konfiguration von Ausgabegeräten (Heads).

- **`apply` und `test` Requests auf `zwlr_output_configuration_v1`:**
    - Ein Client erstellt ein `zwlr_output_configuration_v1`-Objekt mit einem bestimmten `serial` (erhalten vom `zwlr_output_manager_v1.done`-Event).
    - Auf diesem Konfigurationsobjekt kann der Client Änderungen für einzelne `zwlr_output_head_v1`-Objekte vornehmen (z.B. Modus, Position, Skalierung, Aktivierungsstatus setzen).
    - **`test` Request:** Der Client sendet einen `test`-Request, um die Konfiguration zu validieren, ohne sie anzuwenden. Der Compositor prüft, ob die Konfiguration prinzipiell möglich ist.
    - **`apply` Request:** Der Client sendet einen `apply`-Request, um die Konfiguration anzuwenden.
    - **Antworten des Compositors:** In beiden Fällen (`test` und `apply`) muss der Compositor mit einem der folgenden Events antworten:
        - `succeeded`: Die Konfiguration wurde erfolgreich getestet/angewendet. Der Client sollte das Konfigurationsobjekt zerstören. Wenn die Konfiguration angewendet wurde und sich der Zustand geändert hat, sendet der Manager `head`-Events und ein `done`-Event mit einem neuen Serial.
        - `failed`: Die Konfiguration konnte nicht getestet/angewendet werden (z.B. ungültiger Modus). Der Compositor sollte alle Änderungen, die durch einen `apply`-Request ausgelöst wurden, zurückrollen. Der Client sollte das Konfigurationsobjekt zerstören.
        - `cancelled`: Die Konfiguration wurde vom Compositor abgebrochen, weil sich der Zustand eines Outputs geändert hat und die Informationen des Clients veraltet sind (z.B. durch Hotplugging). Der Client kann eine neue Konfiguration mit einem neueren Serial erstellen und es erneut versuchen. Der Client sollte das Konfigurationsobjekt zerstören.
- **Atomarität und Serial-Nummern:**
    - Der `serial` spielt eine Schlüsselrolle für die Atomarität. Der `zwlr_output_manager_v1` sendet ein `done`-Event mit einem `serial`, nachdem alle aktuellen Zustandsinformationen gesendet wurden.29
    - Der Client muss diesen `serial` beim Erstellen eines `zwlr_output_configuration_v1`-Objekts verwenden. Dies stellt sicher, dass die Konfigurationsänderungen des Clients auf einem aktuellen Stand des Compositor-Zustands basieren.
    - Wenn der Compositor eine Konfiguration mit einem veralteten `serial` erhält, kann er diese mit `cancelled` ablehnen.
    - Die Änderungen innerhalb eines `apply`-Requests sollten atomar angewendet werden. Das bedeutet, entweder alle Änderungen werden erfolgreich übernommen, oder bei einem Fehler wird der Zustand vor dem `apply`-Request wiederhergestellt (Rollback).31
- **Smithay Integration:**
    - Smithays `Output` und `OutputManagerState` 34 verwalten die Eigenschaften von Outputs (Modi, Skalierung, Transformation, aktivierter Zustand) und können diese über `wl_output` und `xdg_output` an Clients bekannt machen.
    - Zur Implementierung von `wlr-output-management` muss ein `OutputHandler` (oder eine ähnliche Logik) die Anfragen des Clients verarbeiten, die Konfigurationen validieren und anwenden und die entsprechenden `succeeded`, `failed` oder `cancelled` Events senden. Der `OutputManagerState` würde die globalen Objekte bereitstellen und die `OutputHandler`-Implementierung würde die spezifischen Anfragen bearbeiten.
    - Die Atomarität erfordert sorgfältiges Zustandsmanagement im Compositor. Bevor eine Konfiguration angewendet wird, sollte der aktuelle Zustand zwischengespeichert werden, um im Fehlerfall ein Rollback durchführen zu können.

**1.3.2. `wlr-output-power-management-unstable-v1`** 30

Dieses Protokoll ermöglicht Clients (typischerweise Desktop-Shells) die Steuerung der Energieverwaltungsmodi (DPMS) von Ausgabegeräten, die Teil des Compositor-Raums sind.

- **Exklusive Kontrolle pro Output (`get_output_power`):**
    - Der Client fordert über `zwlr_output_power_manager_v1::get_output_power` ein `zwlr_output_power_v1`-Objekt für ein bestimmtes `wl_output` an.
    - Der Compositor kann diesen Request ablehnen (und ein `failed`-Event auf dem `zwlr_output_power_v1`-Objekt senden), wenn z.B. der Output kein Power-Management unterstützt oder ein anderer Client bereits exklusive Kontrolle hat.30 Dies impliziert, dass der Compositor den Besitz von Power-Management-Kontrollen pro Output verfolgen muss.
- **Reaktion auf vom Compositor initiierte DPMS-Änderungen:**
    - Das `zwlr_output_power_v1::mode`-Event wird gesendet, wenn sich der Power-Modus eines Outputs ändert. Dies kann durch einen `set_mode`-Request des Clients oder durch eine Entscheidung des Compositors selbst geschehen (z.B. durch Inaktivitäts-Timer).49
    - Wenn der Compositor den DPMS-Zustand eines Outputs ändert (z.B. über `system::power_management`), muss er das `mode`-Event an alle Clients senden, die ein `zwlr_output_power_v1`-Objekt für diesen Output halten.
- **Smithay Integration:**
    - Die `OutputHandler`-Implementierung 6 müsste erweitert werden, um die Anfragen dieses Protokolls zu verarbeiten.
    - Der `OutputManagerState` 36 würde das globale `zwlr_output_power_manager_v1`-Objekt verwalten.
    - Der Zustand (aktueller Power-Modus, exklusive Kontrolle) müsste pro Output im Compositor-Zustand (`DesktopState` oder einer Substruktur) gespeichert werden.

**1.3.3. `wlr-layer-shell-unstable-v1`** 53

Dieses Protokoll erlaubt es Clients, Oberflächen (Layer Surfaces) zu erstellen, die über, unter oder zwischen normalen Fenstern gerendert werden, typischerweise für Panels, Hintergrundbilder, Benachrichtigungen und Sperrbildschirme.

- **Management von `LayerSurface`-Daten:**
    - Für jede `LayerSurface` müssen Informationen wie die gewünschte Schicht (`zwlr_layer_shell_v1::layer`), Anker (`anchor`), exklusive Zone (`exclusive_zone`), Ränder (`margin`) und Interaktivitätseinstellungen (`keyboard_interactivity`) gespeichert werden. Diese Daten sind oft Teil einer `SurfaceData`-ähnlichen Struktur, die mit der `wl_surface` der `LayerSurface` assoziiert ist.
    - Smithays `WlrLayerShellState` und `WlrLayerShellHandler` 59 sind für die Verwaltung dieser Zustände und die Bearbeitung von Client-Anfragen zuständig.
- **Interaktion mit `PopupManager` für Layer-Popups:**
    - `LayerSurface`s können Popups haben (z.B. ein Kalender-Popup von einem Panel). Das `zwlr_layer_surface_v1::get_popup`-Request wird verwendet, um ein `xdg_popup` einer `LayerSurface` zuzuordnen.54
    - Der `smithay::desktop::PopupManager` 9 sollte so erweitert oder verwendet werden, dass er auch Popups von `LayerSurface`s korrekt verwalten und positionieren kann. Dies bedeutet, dass der `PopupManager` die Hierarchie von Popups relativ zu ihren Eltern-`LayerSurface`s verstehen muss.
- **Korrekte Berechnung von exklusiven Zonen:**
    - Eine `LayerSurface` kann eine exklusive Zone anfordern (`set_exclusive_zone`). Dies ist ein Bereich entlang eines oder mehrerer Ränder des Outputs, den andere Oberflächen (insbesondere maximierte Fenster) nicht verdecken sollten.55
    - Der Compositor muss die exklusiven Zonen aller `LayerSurface`s auf einem Output aggregieren. Dies ist besonders wichtig für das `system::window_mechanics`-Modul, um Fenster korrekt zu positionieren und zu maximieren, ohne über Panels oder andere Layer-Elemente zu ragen.
    - Die Berechnung muss die Schicht (`layer`), den Anker und die Ränder der `LayerSurface` berücksichtigen. Ein Wert von -1 für die exklusive Zone bedeutet, dass die Oberfläche nicht verschoben werden soll, auch wenn sie andere exklusive Zonen überlappt.56
    - Die `smithay::desktop::Space`-Logik muss diese exklusiven Zonen bei der Berechnung des verfügbaren Arbeitsbereichs für normale Fenster berücksichtigen.

### 1.4. Renderer-Abstraktion

Eine robuste Renderer-Abstraktion ist entscheidend für Flexibilität und Wartbarkeit. Die Traits `FrameRenderer` und `RenderableTexture` sind hierfür vorgesehen.

- **Robuste Trait-Definitionen:**
    - **`FrameRenderer`:**
        - Sollte Methoden zum Beginnen und Beenden eines Frames bereitstellen.
        - Methoden zum Rendern von `RenderableTexture`s an bestimmten Positionen mit Transformationen (Skalierung, Rotation), Alpha-Blending und Clipping-Regionen.
        - Methoden zum Löschen des Framebuffers oder Teilen davon.
        - Potenziell Methoden zur Verwaltung von Shader-Programmen oder anderen Renderer-spezifischen Ressourcen.
    - **`RenderableTexture`:**
        - Repräsentiert eine Textur, die vom `FrameRenderer` gezeichnet werden kann.
        - Sollte Methoden zum Abrufen der Dimensionen und des Formats der Textur bereitstellen.
        - Muss die zugrundeliegenden Pufferdaten (SHM oder DMABUF) kapseln.
- **Effiziente Pufferübergabe und Texturverwaltung:**
    - **SHM-Puffer:** Für SHM-Puffer (`wl_shm_buffer`) müssen die Pixeldaten in eine GPU-Textur hochgeladen werden. Dies kann bei jedem Frame oder nur bei Änderungen erfolgen.
    - **DMABUF-Puffer:** DMABUFs können oft direkt von der GPU importiert werden, was den Kopiervorgang CPU -> GPU vermeidet. Smithays `DmabufFeedback` und `ImportDma` 60 sind hier relevant.
    - **Textur-Caching:** Um das wiederholte Hochladen identischer Pufferinhalte zu vermeiden, kann ein Textur-Cache implementiert werden. Texturen können anhand eines Identifikators des Puffers (z.B. `wl_buffer`-Objekt-ID oder ein Hash des Inhalts für SHM) zwischengespeichert werden.
    - **Freigabe:** Wenn ein `wl_buffer` vom Client freigegeben wird (`wl_buffer.release`), sollten auch die zugehörigen Texturen im Cache freigegeben oder als wiederverwendbar markiert werden.
- **Kommunikation von Schadensverfolgung (Damage Tracking):**
    - Der Compositor (speziell `DesktopState` oder `Space`) verfolgt die beschädigten Regionen jeder Oberfläche (`wl_surface.damage_buffer` oder `wl_surface.damage_surface`) und des gesamten Outputs.9
    - Diese Schadensinformation (typischerweise eine Liste von Rechtecken) muss an den `FrameRenderer` übergeben werden.
    - Der `FrameRenderer` sollte diese Information nutzen, um nur die beschädigten Bereiche des Framebuffers neu zu zeichnen (z.B. durch Setzen einer Scissor-Region oder durch differenzielles Rendern).
    - Smithay bietet im Modul `smithay::backend::renderer::damage` Hilfsmittel für das Damage Tracking auf Output-Ebene.60 Die `Frame`-Implementierung des Renderers sollte die akkumulierten Schäden berücksichtigen.

**Beispiel (konzeptionelle Trait-Definitionen):**

Rust

```
use smithay::utils::{Point, Rectangle, Scale, Transform};
use smithay::backend::renderer::Renderer; // Basis-Renderer-Trait von Smithay

pub trait RenderableTexture {
    fn width(&self) -> u32;
    fn height(&self) -> u32;
    // Evtl. Methode, um den zugrundeliegenden Puffer-Typ zu identifizieren (SHM, DMABUF)
    // fn buffer_type(&self) -> BufferType;
    // Evtl. Methode, um die Textur-ID für den spezifischen Renderer zu erhalten
    // fn renderer_id(&self) -> RendererTextureId;
}

pub trait FrameRenderer<T: RenderableTexture, R: Renderer> { // R ist der Backend-Renderer von Smithay
    fn begin_frame(&mut self, output_damage: Option<&>) -> Result<(), R::Error>;
    fn render_texture(
        &mut self,
        texture: &T,
        src_rect: Rectangle<f64, smithay::utils::Buffer>, // Quellrechteck auf der Textur
        dst_rect: Rectangle<i32, smithay::utils::Physical>, // Zielrechteck auf dem Framebuffer
        transform: Transform,
        alpha: f32,
        damage: &, // Schaden relativ zur Textur im Ziel-Framebuffer
    ) -> Result<(), R::Error>;
    fn clear(&mut self, color: [f32; 4], region: &) -> Result<(), R::Error>;
    fn finish_frame(&mut self) -> Result<(), R::Error>; // Beinhaltet Swap-Buffers o.ä.
}
```

Die Integration mit Smithays eigenem `Renderer`-Trait und den `RenderElement`-Strukturen 60 ist hierbei zu beachten. Die `FrameRenderer`-Abstraktion könnte auf Smithays `Frame`-Trait aufbauen oder dieses wrappen.

### 1.5. XWayland-Integration

Die Integration von XWayland ermöglicht die Ausführung von X11-Anwendungen in der Wayland-Umgebung.

- **Herausforderungen und Lösungen:**
    - **Starten und Verwalten des XWayland-Prozesses:**
        - Smithays `smithay::xwayland::XWayland::new()` 61 startet den XWayland-Server als Kindprozess. Der Compositor muss die Wayland- und X11-Sockets bereitstellen, über die XWayland kommuniziert.
        - Die `XWayland`-Struktur muss im `DesktopState` gehalten und bei Beendigung des Compositors ordnungsgemäß beendet werden.
    - **X11 Window Manager (`X11Wm`):**
        - Der Compositor muss als X11 Window Manager für die von XWayland erzeugten X11-Fenster agieren. Smithay stellt das `XwmHandler`-Trait und die `X11Wm`-Struktur bereit.61
        - Der `XwmHandler` muss Callbacks implementieren, um auf X11-Ereignisse wie das Erstellen, Zerstören, Konfigurieren und Verschieben von Fenstern zu reagieren.
        - Die `X11Wm`-Instanz verwaltet die Verbindung zum XWayland-Server und die X11-Fenster.
    - **Darstellung von X11-Fenstern (`X11Surface`) im Wayland-Space:**
        - XWayland erstellt für jedes X11-Top-Level-Fenster eine `wl_surface`. Smithays `X11Surface` 61 repräsentiert ein solches Fenster und ist mit der zugrundeliegenden X11-Fenster-ID und der `wl_surface` verbunden.
        - Diese `X11Surface`-Objekte müssen in das `smithay::desktop::Window`-Trait gewrappt und im `smithay::desktop::Space` verwaltet werden, genau wie native Wayland-Fenster.
        - Die Geometrie, Stapelreihenfolge und der Aktivierungszustand von X11-Fenstern müssen zwischen dem X11-Protokoll (via `X11Wm`) und dem Wayland-Space synchronisiert werden. Beispielsweise muss ein `XConfigureWindow`-Request vom `X11Wm` verarbeitet und die Position des `X11Surface` im `Space` entsprechend angepasst werden.
    - **Input-Routing:** Eingabeereignisse (Tastatur, Maus) für X11-Fenster müssen korrekt an XWayland weitergeleitet werden. Dies geschieht, indem der Fokus auf die `wl_surface` des `X11Surface` gesetzt wird. Der `SeatHandler` spielt hier eine wichtige Rolle.
    - **Synchronisation von Fensterzuständen:** Zustände wie Maximierung oder Aktivierung müssen zwischen dem Wayland-Compositor und dem X11-Zustand des Fensters synchron gehalten werden. Der `X11Wm` ist dafür verantwortlich, entsprechende X11-Properties (z.B. `_NET_WM_STATE`) zu setzen oder zu interpretieren.
- **Referenzimplementierungen:** Die Anvil-Demo-Compositor von Smithay enthält eine XWayland-Integration, die als gutes Beispiel dienen kann.6

## 2. Eingabeverarbeitung (system::input)

Die Eingabeverarbeitung ist eine kritische Komponente für die Benutzerinteraktion. NovaDE setzt hier auf `libinput` und `xkbcommon`.

### 2.1. `libinput` und `calloop`

- **Best Practices für die Integration des `LibinputInputBackend` in die `calloop`-Ereignisschleife:**
    - Der `LibinputInputBackend` wird typischerweise als eine `calloop::EventSource` in die `calloop`-Schleife eingefügt.
    - Die `dispatch`-Methode des Backends verarbeitet die anstehenden `libinput`-Events.
    - Um die Schleife nicht zu blockieren, sollte die Verarbeitung der einzelnen Events (z.B. Übersetzung in Wayland-Events, Fokus-Logik) möglichst effizient sein. Langwierige Operationen sollten vermieden oder in separate Tasks ausgelagert werden (obwohl dies im Input-Pfad selten vorkommt).
    - Smithay bietet in `smithay::backend::input` und `smithay::input` Abstraktionen, die die Integration erleichtern.
- **Effiziente Verarbeitung von `libinput`-Events:**
    - Events sollten so früh wie möglich gefiltert werden, wenn sie nicht relevant sind.
    - Die Übersetzung von `libinput`-Events in Wayland-Protokoll-Events sollte direkt und ohne unnötige Allokationen erfolgen.
    - Zustandsänderungen (z.B. Tastatur-Modifier-Status) sollten effizient aktualisiert werden.

### 2.2. `xkbcommon`

`xkbcommon` ist für die Interpretation von Tastatur-Layouts und die Übersetzung von Keycodes in Keysyms und UTF-8-Strings zuständig.

- **Korrekte Initialisierung und Aktualisierung von `xkb::State` pro Tastatur/Seat:**
    - Für jede Tastatur (oder allgemeiner, für jeden `Seat` mit Tastatur-Capability) muss eine eigene `xkb::Context` und `xkb::Keymap` erstellt werden.
    - Aus der `Keymap` wird dann eine `xkb::State` abgeleitet. Diese `XkbKeyboardData` (mit `xkb::State`) wird im `SeatState` gehalten.
    - **Layout-Wechsel:** Bei einem Layout-Wechsel (z.B. durch den Benutzer oder eine Systemkonfiguration) muss die `xkb::Keymap` neu geladen und eine neue `xkb::State` daraus erstellt und aktualisiert werden. Dies erfordert typischerweise die Interaktion mit Systemdiensten, die Layout-Änderungen signalisieren (z.B. über D-Bus).
- **Zuverlässige Implementierung von Modifikatoren und Tastenwiederholung:**
    - **Modifikatoren:** `xkb::State::update_key()` aktualisiert den Zustand der Modifikatoren (Shift, Ctrl, Alt, etc.) basierend auf den gedrückten/losgelassenen Tasten. Der `SeatHandler` muss diesen Zustand verwenden, um korrekte Keysyms und Modifier-Masken an die Clients zu senden.
    - **Tastenwiederholung (Key Repeat):**
        - `libinput` selbst sendet keine Wiederholungsereignisse. Der Compositor muss dies implementieren.
        - Wenn eine Taste gedrückt wird, für die Wiederholung aktiviert ist (konfigurierbar über `xkb::Keymap::key_repeats()`), startet ein Timer.
        - Der Timer wird in `calloop` verwaltet (z.B. `calloop::timer::Timer`).
        - Nach einer initialen Verzögerung (`repeat_delay`) sendet der Timer periodisch (mit `repeat_rate`) Events, die der Compositor als erneute Tastendrücke interpretiert und an den fokussierten Client weiterleitet.
        - Der Timer muss gestoppt werden, wenn die Taste losgelassen wird oder der Fokus wechselt.
        - Die `XkbKeyboardData` in der Spezifikation deutet darauf hin, dass die Konfiguration für die Tastenwiederholung (Rate, Verzögerung) aus `xkbcommon` bezogen wird.

### 2.3. Fokusmanagement

Das Fokusmanagement ist komplex und erfordert eine enge Koordination zwischen `SeatHandler`, `system::input` und `system::window_mechanics`.

- **Detaillierte Logik für `SeatHandler::focus_changed`:**
    - Diese Methode wird vom `Seat` aufgerufen, wenn sich der Fokus ändert.
    - Sie muss den alten und neuen Fokus-Target (typischerweise eine `WlSurface`) an die entsprechenden Clients kommunizieren (z.B. `wl_keyboard::leave`/`enter`, `wl_pointer::leave`/`enter`).
    - Der `SeatHandler` sollte auch den internen Zustand des Compositors aktualisieren (z.B. welches Fenster gerade aktiv ist).
- **Konsistente Handhabung des Fokus zwischen Tastatur, Zeiger und Touch:**
    - **Tastaturfokus:** Folgt typischerweise dem "aktiven Fenster". Ein Klick auf ein Fenster oder eine explizite Aktivierungsanforderung setzt den Tastaturfokus.
    - **Zeigerfokus ("Pointer Focus"):** Folgt der Mausbewegung. Wenn der Zeiger eine neue Oberfläche betritt, erhält diese den Zeigerfokus. Dies löst `wl_pointer::enter` und `wl_pointer::leave` aus.
    - **Touchfokus:** Ist oft an einen bestimmten Touchpunkt (Slot) gebunden. Wenn ein Touchpunkt auf einer Oberfläche landet, erhält diese Oberfläche den Touchfokus für diesen Slot.
    - Die Konsistenz wird dadurch erreicht, dass der `Seat` den Zustand für alle Eingabemethoden hält und der `SeatHandler` die entsprechenden Wayland-Events basierend auf diesem Zustand sendet.
- **Interaktion mit `system::window_mechanics::set_application_focus`:**
    - `set_application_focus` in `system::window_mechanics` ist für die _technische Umsetzung_ des Fokuswechsels zuständig (z.B. Anfordern der Aktivierung eines XDG-Toplevels, Aktualisieren des internen Zustands im `DesktopState`).
    - Der `SeatHandler` (oder die Logik, die Fokusentscheidungen trifft) ruft `set_application_focus` auf, um einen Fokuswechsel zu initiieren.
    - Um Redundanz oder Konflikte zu vermeiden:
        - `SeatHandler::focus_changed` sollte primär für das Senden der Wayland-Protokoll-Events zuständig sein, nachdem der Fokuswechsel _entschieden und technisch umgesetzt_ wurde.
        - Die Entscheidung, welches Fenster den Fokus erhält (z.B. durch Klick), und die Aktivierung dieses Fensters (über `set_application_focus`) sollten vor dem Aufruf von `focus_changed` erfolgen.
- **Finden der korrekten Oberfläche unter dem Zeiger/Touchpunkt (`find_surface_and_coords_at_global_point`):**
    - Diese Funktion ist entscheidend für das korrekte Routing von Zeiger- und Touch-Events.
    - Sie muss die Fenster im `smithay::desktop::Space` in umgekehrter Stapelreihenfolge (von oben nach unten) durchlaufen.
    - Für jedes Fenster muss geprüft werden, ob der globale Punkt innerhalb seiner Bounding Box liegt.
    - Anschließend muss der Punkt in die Koordinaten der Fensteroberfläche transformiert werden.
    - `smithay::desktop::Window::surface_under()` kann verwendet werden, um die spezifische `WlSurface` (inkl. Subsurfaces und Popups) unter dem lokalen Punkt zu finden.
    - **Eingaberegionen (`wl_surface.set_input_region`):** Die Eingaberegion einer Oberfläche definiert, welche Teile der Oberfläche auf Eingaben reagieren. `surface_under` muss dies berücksichtigen. Nur wenn der Punkt innerhalb der Eingaberegion liegt, gilt die Oberfläche als getroffen.
    - **Stapelreihenfolge:** Die Iteration durch die Fenster im `Space` muss die korrekte z-Ordnung beachten, um sicherzustellen, dass die oberste sichtbare Oberfläche an diesem Punkt ausgewählt wird. `smithay::desktop::Space::elements_under()` kann hierfür nützlich sein.

### 2.4. Gesten

Die Implementierung robuster Gestenerkennung (Pinch, Swipe) auf Basis von `libinput`-Events ist eine fortgeschrittene Aufgabe.

- **Ansätze:**
    - **Direkte Verarbeitung von `libinput`-Gestenereignissen:** `libinput` selbst erkennt bereits einige grundlegende Gesten (z.B. `GesturePinchEvent`, `GestureSwipeEvent`). Diese Events enthalten Informationen wie Skalierungsfaktor, Winkeländerung (Pinch) oder Verschiebungsvektoren (Swipe).
        - Der `LibinputInputBackend` leitet diese Events an den Compositor weiter.
        - Der Compositor muss diese Events interpretieren und in höherwertige Aktionen oder Wayland-Protokoll-Events (z.B. über `zwp_pointer_gestures_v1`) umwandeln.
    - **Eigene Gestenerkennung:** Für komplexere oder benutzerdefinierte Gesten, die `libinput` nicht direkt bereitstellt, müsste der Compositor eine eigene Gestenerkennungslogik implementieren. Dies würde das Sammeln und Analysieren von Folgen von Touch- oder Zeigerereignissen über die Zeit erfordern. Dies ist deutlich aufwendiger.
- **Übersetzung in Domänenaktionen:**
    - Erkannte Gesten (z.B. ein "Pinch-to-Zoom" auf einem Fenster) müssen in Aktionen der Domänenschicht übersetzt werden (z.B. "ZoomFenster", "WechsleWorkspace").
    - Dies erfordert eine klare Schnittstelle zwischen `system::input` und der Domänenschicht, möglicherweise über den `SystemEventBridge`.
- **Smithay-Unterstützung:** Smithay bietet Unterstützung für das `zwp_pointer_gestures_v1`-Protokoll, was die Weiterleitung von Pinch- und Swipe-Gesten an Clients ermöglicht. Der Compositor muss die `libinput`-Events in die entsprechenden Wayland-Events dieses Protokolls übersetzen.
- **Herausforderungen:**
    - **Eindeutigkeit:** Sicherstellen, dass Gesten nicht mit normalen Zeiger- oder Touch-Interaktionen kollidieren.
    - **Konfigurierbarkeit:** Benutzer erwarten oft, Gesten konfigurieren zu können.
    - **Performance:** Die Gestenerkennung darf die Eingabeverarbeitung nicht verlangsamen.

## 3. D-Bus Interaktion (system::dbus_interfaces mit zbus)

Die Kommunikation mit Systemdiensten über D-Bus ist ein integraler Bestandteil moderner Desktop-Umgebungen. `zbus` wird für die asynchrone D-Bus-Kommunikation mit Tokio verwendet.

### 3.1. Proxy-Management

Die Erstellung und Verwaltung von `zbus::Proxy`-Instanzen, insbesondere für dynamische Objektpfade, erfordert sorgfältige Überlegungen zum Lebenszyklus.

- **Best Practices:**
    - **Caching von Proxies:** Für häufig verwendete Proxies zu bekannten Objektpfaden (z.B. `/org/freedesktop/UPower`, `/org/freedesktop/NetworkManager`) ist es sinnvoll, die Proxy-Instanzen nach der ersten Erstellung zwischenzuspeichern (z.B. in einem `OnceLock` oder einem `Arc<Mutex<...>>` Feld im jeweiligen `DBusClientService`). Dies vermeidet den Overhead wiederholter Proxy-Erstellung.
        - Die `DBusConnectionManager` mit `OnceLock` für Session/System-Bus ist ein guter Ansatz für die zugrundeliegende `zbus::Connection`.
    - **Dynamische Objektpfade:** Für Proxies zu dynamischen Objektpfaden (z.B. `UPowerDeviceProxy` für `/org/freedesktop/UPower/devices/display_device`, `LogindSessionProxy` für `/org/freedesktop/login1/session/self`):
        - Diese Proxies sollten bei Bedarf erstellt werden, wenn der spezifische Objektpfad bekannt wird (z.B. durch ein Signal vom Manager-Objekt oder eine Abfrage).
        - Eine Möglichkeit ist, sie nicht langfristig zu cachen, wenn ihre Lebensdauer stark an das zugrundeliegende D-Bus-Objekt gebunden ist und dieses häufig wechseln kann.
        - Alternativ kann ein Cache (z.B. `HashMap<OwnedObjectPath, Arc<MyDeviceProxy>>`) verwendet werden, aber es muss eine Strategie zur Invalidierung geben, falls das D-Bus-Objekt verschwindet. Signale wie `InterfacesRemoved` vom `org.freedesktop.DBus.ObjectManager` können hier helfen, sind aber nicht immer verfügbar.
    - **Lebenszyklusmanagement:**
        - Proxies sind an die Lebensdauer der `zbus::Connection` gebunden. Wenn die Verbindung getrennt wird, werden die Proxies ungültig.
        - Für Dienste, die neu gestartet werden können, muss der Client-Service in der Lage sein, die Verbindung und die Proxies neu zu initialisieren. Das Abonnieren von `NameOwnerChanged`-Signalen auf dem D-Bus kann helfen, Neustarts von Diensten zu erkennen.
- **Beispiel (Caching eines Manager-Proxys):**
    
    Rust
    
    ```
    use zbus::Proxy;
    use tokio::sync::OnceCell;
    use std::sync::Arc;
    
    static UPOWER_PROXY: OnceCell<Arc<UPowerProxy<'static>>> = OnceCell::const_new();
    
    async fn get_upower_proxy(connection: &Arc<zbus::Connection>) -> Result<Arc<UPowerProxy<'static>>, zbus::Error> {
        UPOWER_PROXY.get_or_try_init(|| async {
            let proxy = UPowerProxy::new(connection).await?;
            Ok(Arc::new(proxy))
        }).await.cloned()
    }
    ```
    
    Für dynamische Proxies wäre ein `HashMap` im Service-Struct, geschützt durch einen `Mutex` oder `RwLock`, ein gängiger Ansatz.

### 3.2. Asynchrone Signalbehandlung

Die Verarbeitung von D-Bus-Signalen erfolgt in `tokio::tasks`, die `zbus::SignalStream`s konsumieren.

- **Robuste Implementierung:**
    - Jeder `SignalStream` sollte in einem eigenen `tokio::task` verarbeitet werden, um Blockierungen zu vermeiden.
    - Der Task muss den Stream kontinuierlich pollen (z.B. mit `while let Some(signal) = stream.next().await`).
    - **Fehler im Stream:** `stream.next().await` kann `None` zurückgeben, wenn der Stream beendet wird (z.B. Verbindung verloren, Dienst beendet). Der Task sollte dies erkennen und sich sauber beenden oder versuchen, den Stream neu zu initialisieren (z.B. durch erneutes Abonnieren der Signale nach Wiederherstellung der Verbindung). Fehler während des Stream-Pollings (z.B. `zbus::Error`) sollten geloggt und behandelt werden.
    - **Parsen von Signaldaten:** Die Argumente eines Signals kommen als `zbus::zvariant::Value` oder spezifischere Typen an. Diese müssen sorgfältig in die internen Event-Typen der Systemschicht gemappt werden. Fehler beim Parsen (z.B. unerwarteter Typ) sollten abgefangen und geloggt werden.
    - **Umwandlung in interne Events:** Nach erfolgreichem Parsen werden die Signaldaten in anwendungsspezifische Events (z.B. `UPowerEvent`, `NetworkManagerEvent`) umgewandelt und typischerweise über den `SystemEventBridge` publiziert.
- **Beispiel (konzeptionell):**
    
    Rust
    
    ```
    // In UPowerClientService::initialize()
    // let upower_proxy = self.get_proxy().await?;
    // let mut device_added_stream = upower_proxy.receive_device_added().await?;
    // let event_bridge = global_system_event_bridge();
    
    // tokio::spawn(async move {
    //     while let Some(signal) = device_added_stream.next().await {
    //         match signal.args() {
    //             Ok(args) => {
    //                 let device_path = args.device_path; // Annahme basierend auf Spezifikation
    //                 // Hole Details für device_path, erstelle UPowerEvent::DeviceAdded
    //                 // event_bridge.publish(SystemLayerEvent::UPower(upower_event)).await;
    //             },
    //             Err(e) => {
    //                 // log_error!("Failed to parse UPower DeviceAdded signal args: {:?}", e);
    //             }
    //         }
    //     }
    //     // log_info!("UPower DeviceAdded signal stream ended.");
    // });
    ```
    
- **Referenz:** Die `zbus` Dokumentation zu `SignalStream` und asynchroner Verarbeitung ist hier relevant.

### 3.3. `org.freedesktop.Notifications` Server

Die Implementierung eines D-Bus-Servers für `org.freedesktop.Notifications` 75 erfordert ein Mapping zwischen D-Bus `u32` IDs und den internen `Uuid`s der Domänenschicht sowie das korrekte Emittieren von Signalen.

- **ID-Mapping (`D-Bus u32 <-> Domain Uuid`):**
    - Wenn der Server eine Benachrichtigung über `Notify` empfängt, generiert er eine neue, eindeutige `u32` ID und gibt diese an den Client zurück.
    - Diese `u32` ID muss intern mit der `Uuid` der Domänen-Benachrichtigung (die vom `domain::NotificationService` kommt oder dorthin gesendet wird) assoziiert werden. Ein `HashMap<u32, Uuid>` und ggf. ein `HashMap<Uuid, u32>` (für schnelle Rückwärts-Lookups) im Zustand des `FreedesktopNotificationsServer` sind hierfür geeignet. Diese Maps müssen durch einen `Mutex` oder `RwLock` geschützt werden, da D-Bus-Methodenaufrufe und Domain-Event-Handler potenziell nebenläufig sind.
- **Verwendung von `zbus::SignalContext` zum Emittieren von Signalen:**
    - Wenn ein Domänen-Event (z.B. `DomainNotificationEvent::Closed`) eintritt, das ein D-Bus-Signal auslösen soll (z.B. `NotificationClosed`), muss der Listener dieses Domänen-Events den `FreedesktopNotificationsServer` darüber informieren.
    - Der Server-Task, der die D-Bus-Schnittstelle implementiert, ist derjenige, der Signale emittieren kann.
    - Um ein Signal aus einem anderen Task (dem Domain-Event-Listener) zu emittieren, muss der `SignalContext` des entsprechenden Interfaces verwendet werden.
    - Eine Möglichkeit ist, dass der Domain-Event-Listener eine Nachricht (z.B. über einen `tokio::sync::mpsc::channel`) an den D-Bus-Server-Task sendet. Dieser Task ruft dann die Signal-Emissionsmethode auf seinem Interface-Objekt auf.
    - **Beispiel (konzeptionell für `NotificationClosed`):**
        
        Rust
        
        ```
        // In FreedesktopNotificationsServer (zbus::Interface Implementierung)
        // #[dbus_interface(name = "org.freedesktop.Notifications")]
        // struct NotificationsServerLogic {
        //     //... id_map: Arc<Mutex<HashMap<u32, Uuid>>>,...
        //     // signal_context: Option<SignalContext<'static>> // Wird von zbus bereitgestellt
        // }
        
        // impl NotificationsServerLogic {
        //     #[dbus_interface(signal)]
        //     async fn notification_closed(&self, ctxt: &SignalContext<'_>, id: u32, reason: u32) -> zbus::Result<()>;
        
        //     // Diese Methode wird vom Domain-Event-Listener aufgerufen (z.B. via MPSC-Kanal)
        //     async fn handle_domain_notification_closed(&self, dbus_id: u32, reason: u32) {
        //         if let Some(ctxt) = self.signal_context() { // Korrekten Weg zum SignalContext finden
        //             if let Err(e) = self.notification_closed(ctxt, dbus_id, reason).await {
        //                 // log_error!("Failed to emit NotificationClosed signal: {:?}", e);
        //             }
        //         }
        //     }
        // }
        ```
        
        Die `SwayNotificationCenter` 77 und `mako` 78 sind Beispiele für Notification Daemons, deren D-Bus Implementierung (falls in Rust/zbus oder C/GDBus) als Inspiration dienen kann. Die Freedesktop Notification Specification 75 ist die maßgebliche Quelle.

### 3.4. Fehlerbehandlung

- **Umgang mit `zbus::Error` und `zbus::fdo::Error`:**
    - `zbus::Error` ist der allgemeine Fehlertyp von `zbus`, der I/O-Fehler, Serialisierungsfehler, Verbindungsfehler etc. umfassen kann.79
    - `zbus::fdo::Error` ist ein spezifischer D-Bus-Fehlertyp, der von Methodenaufrufen zurückgegeben werden kann (z.B. `org.freedesktop.DBus.Error.ServiceUnknown`).
    - Diese Fehler müssen in den jeweiligen Client-Services (z.B. `UPowerClientService`) abgefangen werden.
- **Mapping in eigenes `DBusInterfaceError`-Schema unter Beibehaltung des Fehlerkontexts:**
    - Das `DBusInterfaceError`-Enum sollte Varianten für spezifische Fehlerfälle (z.B. `ServiceUnavailable`, `MethodCallFailed`, `SignalStreamError`, `InvalidResponse`) und eine generische Variante für unerwartete `zbus`-Fehler haben.
    - Die `thiserror`-Crate ist hierfür ideal.
    - Die ursprüngliche `zbus::Error` oder `zbus::fdo::Error` sollte als `source` im `DBusInterfaceError` gespeichert werden, um den Kontext nicht zu verlieren.
    - **Beispiel (konzeptionell):**
        
        Rust
        
        ```
        #
        pub enum DBusInterfaceError {
            #
            ServiceUnavailable { service: String, #[source] source: zbus::Error },
        
            #[error("Method call {method:?} on {interface:?} failed")]
            MethodCallFailed {
                interface: String,
                method: String,
                #[source]
                source: zbus::fdo::Error,
            },
        
            #
            MessageSendError(#[from] zbus::Error),
        
            #
            ResponseParseError{ method: String, #[source] source: zbus::zvariant::Error },
            //... weitere Varianten
        }
        
        // In einem Client-Service:
        // match some_proxy.some_method().await {
        //     Ok(reply) => Ok(reply),
        //     Err(zbus::Error::FDO(fdo_error)) => Err(DBusInterfaceError::MethodCallFailed {
        //         interface: P::INTERFACE.to_string(), // P ist der Proxy-Typ
        //         method: "some_method".to_string(),
        //         source: *fdo_error,
        //     }),
        //     Err(e) => Err(DBusInterfaceError::MessageSendError(e)),
        // }
        ```
        
        Die Granularität der Fehlertypen ist ein wichtiger Aspekt. Eine zu feine Aufteilung von `DBusInterfaceError` kann unhandlich werden. Eine gute Balance zwischen spezifischen, für die Domänenschicht relevanten Fehlern und generischen Catch-All-Varianten ist anzustreben. Die Domänenschicht interessiert sich beispielsweise eher dafür, _ob_ ein Dienst nicht verfügbar ist, um ein Feature zu deaktivieren, als für die genaue Art eines I/O-Fehlers auf dem D-Bus-Socket. Das `DBusInterfaceError` sollte diese Abstraktionsebene widerspiegeln. Bei bestimmten Fehlern, wie `ServiceUnavailable` oder temporären Netzwerkproblemen, könnte eine Retry-Logik in den Client-Services implementiert werden, bevor der Fehler an die Domänenschicht weitergegeben wird. Dies erhöht die Robustheit der Systemschicht gegenüber transienten Fehlern.

Tabelle für Sektion 3.1: Proxy-Management-Strategien

Eine durchdachte Strategie für das Management von D-Bus-Proxies ist entscheidend für Performance und Stabilität. Die naive Erstellung eines Proxys bei jedem Methodenaufruf ist ineffizient. Andererseits können langlebige Proxies veralten, wenn der zugrundeliegende Dienst neu startet oder Objekte verschwinden.

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**Strategie**|**Beschreibung**|**Vorteile**|**Nachteile**|**zbus-Mechanismen**|**Eignung für NovaDE-Dienste (Beispiele)**|
|**On-Demand Creation**|Proxy wird bei jedem benötigten Aufruf neu erstellt.|Immer aktuell; einfach zu implementieren.|Hoher Overhead bei häufigen Aufrufen.|`Proxy::new(conn).await`|Selten genutzte Dienste oder einmalige Aufrufe.|
|**Cached Static Proxies**|Proxies für bekannte, statische Objektpfade werden einmal erstellt und wiederverwendet.|Geringer Overhead nach Initialisierung.|Proxy kann veralten, wenn Dienst neu startet (Neuerstellung erforderlich).|`OnceCell`, `Arc<Proxy>`|Manager-Proxies: `UPowerProxy`, `LogindManagerProxy`, `NetworkManagerProxy`, `SecretServiceProxy`, `PolicyKitAuthorityProxy`.|
|**Cached Dynamic Proxies**|Proxies für dynamische Objektpfade werden bei Bedarf erstellt und in einem Cache (z.B. `HashMap`) gehalten.|Reduzierter Overhead für häufig zugegriffene dynamische Objekte.|Cache-Invalidierung komplex; Proxy kann veralten.|`Arc<Mutex<HashMap<ObjectPath, Arc<Proxy>>>>`|`UPowerDeviceProxy`, `LogindSessionProxy`, `NMDeviceProxy`, `SecretCollectionProxy`, `SecretItemProxy`. Erfordert Überwachung von Signalen (z.B. `InterfacesRemoved`).|
|**Proxies via Manager Signals**|Proxies für dynamische Objekte werden nur erstellt, wenn ihre Existenz durch Signale des Managers bekannt wird.|Erstellt Proxies nur für tatsächlich existierende Objekte.|Abhängig von der Zuverlässigkeit und Semantik der Manager-Signale.|Signal-Handler, die Proxies erstellen/entfernen.|Wie "Cached Dynamic Proxies", aber die Erstellung/Entfernung wird durch Signale wie `DeviceAdded`/`DeviceRemoved` (UPower) oder `SessionNew`/`SessionRemoved` (Logind) gesteuert.|
|**Server-seitige Objekt-Ref.**|(Für den Notifications-Server) Referenzen auf exportierte Objekte werden von zbus verwaltet.|Lebenszyklus wird von zbus gehandhabt.|Weniger direkte Kontrolle.|`#[dbus_interface]`, `ObjectServer`|`FreedesktopNotificationsServer`.|

## 4. PipeWire Integration (system::audio_management mit `pipewire-rs`)

Die Integration von PipeWire 80 für Audio-Management in NovaDE erfolgt über die `pipewire-rs`-Bibliothek. Dies erfordert die Verwaltung des PipeWire-MainLoops und die Kommunikation zwischen dem asynchronen Service und dem PipeWire-Thread.

### 4.1. `MainLoop` Management

PipeWire benötigt einen laufenden MainLoop. Die `pipewire-rs`-Bibliothek abstrahiert dies.

- **Strategien zur Integration der `pipewire::MainLoop` in eine tokio-basierte Anwendung:**
    
    - **Option 1: Dedizierter Thread für PipeWire `MainLoop`:** Dies ist der gängigste und oft robusteste Ansatz. Ein separater Systemthread wird gestartet, der ausschließlich `pipewire::MainLoop::run()` ausführt.
        - _Vorteile:_ Klare Trennung der Audioverarbeitung vom Rest der Anwendung, blockiert nicht die Tokio-Runtime.
        - _Nachteile:_ Erfordert Inter-Thread-Kommunikation (IPC) für Befehle und Zustandsaktualisierungen.
    - **Option 2: Integration des PipeWire-FDs in eine tokio-kompatible Event-Loop:** Theoretisch könnte, falls `pipewire-rs` einen Dateideskriptor des PipeWire-Mainloops exponiert, dieser in den Tokio-Eventloop integriert werden (ähnlich wie `calloop` FDs integriert). Dies ist jedoch mit `pipewire-rs` **unwahrscheinlich**, da es oft auf GLib im Hintergrund setzt oder einen eigenen Loop managed. Die Dokumentation von `pipewire-rs` ist hier entscheidend.
        - _Vorteile (hypothetisch):_ Potenziell geringerer Threading-Overhead.
        - _Nachteile (hypothetisch):_ Deutlich komplexere Implementierung, falls nicht direkt von `pipewire-rs` unterstützt.
    - **Empfehlung für NovaDE:** Ein dedizierter Thread für den PipeWire `MainLoop` wird empfohlen, da dies ein etabliertes Muster ist und die Komplexität der direkten FD-Integration vermeidet, es sei denn, `pipewire-rs` bietet explizite Unterstützung für eine Tokio-Integration.
- **Sichere Kommunikation zwischen async-Service-Methoden und dem PipeWire-Thread:**
    
    - **Befehle (Tokio-Service -> PipeWire-Thread):** Ein `tokio::sync::mpsc::channel` ist hierfür ideal. Der `PipeWireClientService` (laufend im Tokio-Kontext) sendet Befehle (z.B. `SetVolumeCommand { device_id: String, volume: f32 }`) an den PipeWire-Thread. Der PipeWire-Thread besitzt den `Receiver`-Teil des Kanals und verarbeitet die eintreffenden Befehle synchron innerhalb seines `MainLoop`-Kontexts.
    - **Zustandsupdates und Events (PipeWire-Thread -> Tokio-Service):**
        - Für lesbare Zustände (z.B. aktuelle Geräteliste, Lautstärken): Ein `Arc<RwLock<AudioSystemState>>` kann verwendet werden. Der PipeWire-Thread hat Schreibzugriff und aktualisiert diesen Zustand, wenn sich etwas ändert (z.B. nach einem Befehl oder durch externe PipeWire-Events). Der Tokio-Service hat Lesezugriff.
        - Für Event-Benachrichtigungen (z.B. `DeviceListChanged`, `DefaultDeviceChanged`): Ein `tokio::sync::watch::channel` (für einzelne Werte, die sich ändern) oder ein `tokio::sync::broadcast::channel` (für Ereignisse, die von mehreren Stellen konsumiert werden könnten) kann vom PipeWire-Thread verwendet werden, um Updates an den Tokio-Service zu senden. Der `PipeWireClientService` abonniert diese Kanäle.
        - Alternativ kann der PipeWire-Thread Callbacks verwenden (siehe Abschnitt 4.2), die dann ihrerseits Nachrichten über einen weiteren `mpsc`-Kanal zurück an den Tokio-Thread senden. Dieser kann dann die `AudioEvent`s über den `SystemEventBridge` publizieren.
- **Code-Beispiel (konzeptionell für dedizierten Thread und MPSC):**
    
    Rust
    
    ```
    use pipewire::Loop; // Beispiel, tatsächliche Typen können variieren
    use std::sync::{Arc, RwLock};
    use tokio::sync::mpsc;
    
    enum PipeWireCommand {
        SetVolume(String, f32),
        // Weitere Befehle
    }
    
    struct InternalAudioState {
        //... z.B. Vec<AudioDevice>
    }
    
    pub struct PipeWireClientService {
        command_tx: mpsc::Sender<PipeWireCommand>,
        audio_state: Arc<RwLock<InternalAudioState>>,
        //... event_rx: broadcast::Receiver<AudioEvent>...
    }
    
    impl PipeWireClientService {
        pub fn new(/*... */) -> Self {
            let (command_tx, mut command_rx) = mpsc::channel(32);
            let audio_state = Arc::new(RwLock::new(InternalAudioState { /*... */ }));
            let audio_state_clone = audio_state.clone();
            // Hier fehlt der Event-Sender vom PipeWire-Thread zum SystemEventBridge
    
            std::thread::spawn(move |
    ```
    

| {

// Diese Initialisierung ist stark vereinfacht und dient nur dem Konzept

let main_loop = pipewire::MainLoop::new(None)

.expect("Failed to create PipeWire MainLoop");

let context = pipewire::Context::new(&main_loop)

.expect("Failed to create PipeWire Context");

let core = context.connect(None)

.expect("Failed to connect to PipeWire Core");

let registry = core.get_registry()

.expect("Failed to get PipeWire Registry");

````
            // Hier würden Listener für Core- und Registry-Events registriert (siehe 4.2)
            // Diese Listener würden audio_state_clone aktualisieren und/oder Events senden

            loop {
                // Verarbeite Befehle vom Tokio-Service
                // main_loop.iterate(false) blockiert nicht lange, erlaubt command_rx.try_recv()
                // oder man nutzt einen blocking_recv mit Timeout in einer Schleife,
                // die auch main_loop.iterate() aufruft.
                match command_rx.blocking_recv() { // In einem echten Szenario ggf. non-blocking oder mit Timeout
                    Some(command) => {
                        // PipeWire-Aktionen basierend auf dem Befehl ausführen
                        // z.B. core.set_volume(...)
                        // audio_state_clone.write().unwrap()... aktualisieren
                    }
                    None => break, // Kanal wurde geschlossen
                }

                // PipeWire-Events verarbeiten (non-blocking)
                if main_loop.iterate(false) < 0 {
                    // Fehler in der PipeWire-Schleife
                    break;
                }
            }
        });

        Self { command_tx, audio_state /*,... */ }
    }

    // Async Methoden, die command_tx verwenden
    pub async fn set_device_volume(&self, device_id: String, volume: f32) -> Result<(), AudioManagementError> {
        self.command_tx.send(PipeWireCommand::SetVolume(device_id, volume)).await
           .map_err(|_| AudioManagementError::InternalThreadCommunicationError)?;
        Ok(())
    }
}
```
Die Thread-Sicherheit der von `pipewire-rs` bereitgestellten Objekte ist fundamental. Objekte wie `pipewire::Core` oder `pipewire::Registry` sind oft nicht `Send` oder `Sync`. Das bedeutet, sie dürfen nicht direkt über `Arc<Mutex<T>>` zwischen Threads geteilt und von verschiedenen Threads modifiziert werden. Alle Operationen, die diese Objekte verändern, müssen im Kontext des PipeWire-MainLoops (also im dedizierten PipeWire-Thread) ausgeführt werden. Befehle vom Tokio-Service müssen daher über einen Kanal an diesen Thread gesendet werden.
````

### 4.2. Callback-Management

PipeWire verwendet ein Callback-basiertes System für Events (z.B. über `RegistryListener`, `CoreListener`, `NodeListener`, `StreamListener`).

- **Verwaltung von Listener-Strukturen, die `'static` sein müssen:**
    
    - Wenn `add_listener_local` verwendet wird, erwartet `pipewire-rs` oft, dass der Listener eine `'static` Lebensdauer hat, da der Listener potenziell länger existiert als der Scope, in dem er erstellt wurde.
    - **Problem mit `self`-Referenzen:** Wenn der Listener Methoden des `PipeWireClientService` oder dessen internen Zustands aufrufen muss, wird es schwierig, da `self` nicht einfach in einen `'static` Closure verschoben werden kann, wenn der Listener selbst Teil von `self` ist oder `self` eine kürzere Lebensdauer hat.
    - **Lösungen:**
        1. **`Arc` und schwache Referenzen (`Weak`):** Der Zustand, auf den der Listener zugreifen muss (z.B. der `mpsc::Sender` für Events zurück an Tokio oder der `Arc<RwLock<InternalAudioState>>`), kann in einen `Arc` gepackt werden. Der Listener-Closure erhält einen Klon dieses `Arc`. Wenn der Listener den `PipeWireClientService` selbst referenzieren muss (z.B. um Methoden aufzurufen), kann ein `Weak<PipeWireClientService>` verwendet werden, das vor der Verwendung zu einem `Arc` aufgewertet wird, um Zyklen zu vermeiden.
        2. **Kanalbasierte Kommunikation:** Der Listener-Callback sendet die empfangenen Daten/Events über einen `mpsc::Sender` (der in den Callback geklont wurde) an einen zentralen Verarbeitungspunkt im PipeWire-Thread oder direkt zurück an den Tokio-Thread.
        3. **Globale oder statische Handler (mit Vorsicht):** In seltenen Fällen könnten globale Handler verwendet werden, aber dies erschwert das Zustandsmanagement erheblich und ist meist nicht empfehlenswert.
- **Beenden von Listenern:**
    
    - Die von `add_listener_local` zurückgegebenen `Listener` structs implementieren `Drop`. Wenn der `Listener` aus dem Scope geht, wird der Listener automatisch von PipeWire entfernt.
    - Es ist wichtig, diese `Listener`-Objekte so lange im Speicher zu halten, wie die Callbacks aktiv sein sollen (z.B. als Felder im PipeWire-Thread-Zustand). Wenn sie vorzeitig gedroppt werden, werden keine Callbacks mehr empfangen.
- **Beispiel (konzeptionell für Registry-Listener):**
    
    Rust
    
    ```
    // Im PipeWire-Thread
    // let registry_listener = registry.add_listener_local()
    //    .global({
    //         let audio_state_clone = audio_state.clone(); // Arc<RwLock<InternalAudioState>>
    //         // let event_tx_clone = event_tx.clone(); // mpsc::Sender<AudioEvent>
    //         move |global_obj| {
    //             if let Some(props) = &global_obj.props {
    //                 if props.get(pipewire::keys::OBJECT_TYPE) == Some("PipeWire:Interface:Device") {
    //                     // Neues Audiogerät gefunden
    //                     // audio_state_clone.write().unwrap().add_device(...);
    //                     // event_tx_clone.send(AudioEvent::DeviceAdded(...)).unwrap();
    //                 }
    //             }
    //         }
    //     })
    //     //... weitere Callbacks...
    //    .register();
    // // registry_listener muss am Leben gehalten werden.
    ```
    

### 4.3. SPA Pod Erstellung und Anwendung

SPA (Simple Plugin API) Pods werden verwendet, um Parameter für PipeWire-Objekte (Nodes, Streams) zu setzen, z.B. für Lautstärkeregelung.

- **Detaillierte Beispiele für die Erstellung von `SpaPod`s:**
    - `pipewire-rs` bietet Builder-APIs oder direkte Konstruktionsmethoden für Pods.
    - **Lautstärkeregelung (`Props` mit `channelVolumes`, `mute`):**
        - Ein `Props`-Pod (oft ein `Object`-Pod mit `ParamProps`-Typ) wird erstellt.
        - Dieser enthält Eigenschaften wie `channelVolumes` (ein Array von Floats, z.B. `[0.5, 0.5]` für Stereo bei 50%) und `mute` (ein Boolean).
        - Die genauen Namen der Eigenschaften (`SpaId`) und deren Typen müssen der PipeWire-Spezifikation für Audio-Nodes entsprechen (z.B. `SpaId::from_str("channelVolumes")`).
    - **Routenparameter auf Devices:** Ähnlich können Pods für Routing-Parameter erstellt werden, um z.B. ein Default-Profil für ein Gerät zu setzen.
- **Korrekte Verwendung mit `node.set_param()`:**
    - Die `set_param`-Methode eines `Node`-Objekts (oder `Stream`-Objekts) erwartet die ID des Parameters, der gesetzt werden soll (z.B. `SpaId::EnumParamPortConfig` für Port-Konfiguration, `SpaId::EnumParamProps` für allgemeine Eigenschaften) und den `SpaPod` mit den neuen Werten.
    - **Beispiel (konzeptionell):**
        
        Rust
        
        ```
        // Annahme: node ist ein pipewire::Node
        // use pipewire::spa::pod::{Pod, PodObject, PodBuilder, Value};
        // use pipewire::spa::param::ParamType;
        // use pipewire::spa::utils::SpaTypes; // Für SpaId-Konstanten
        
        // // Lautstärke setzen (vereinfacht)
        // let mut props_builder = PodBuilder::new_object(
        //     SpaTypes::Props, // Builder-Typ
        //     ParamType::Props.as_raw() // Pod-Typ-ID
        // );
        // props_builder.add(
        //     SpaTypes::PropChannelVolumes, // Key-ID
        //     &Value::Array(vec![Value::Float(0.7), Value::Float(0.7)]) // Wert
        // ).unwrap();
        // props_builder.add(
        //     SpaTypes::PropMute,
        //     &Value::Bool(false)
        // ).unwrap();
        // let props_pod = props_builder.end();
        
        // node.set_param(ParamType::Props.as_raw(), 0, &props_pod).unwrap();
        ```
        
        Die genaue API für die Pod-Erstellung in `pipewire-rs` muss der Dokumentation entnommen werden. Die Verwendung von `spa_sys` oder höheren Abstraktionen in `pipewire-rs` ist möglich.

### 4.4. Metadaten-Interface

Das Metadaten-Interface in PipeWire wird verwendet, um systemweite Einstellungen wie Standard-Audio-Geräte zu verwalten.

- **Interaktion mit dem `Metadata`-Objekt:**
    - PipeWire exponiert ein globales `Metadata`-Objekt (oft mit der ID `PIPEWIRE_METADATA_OBJECT_ID_SETTINGS` oder über die Registry auffindbar).
    - Dieses Objekt hat Eigenschaften (Properties), die über `metadata.set_property()` und `metadata.property()` (oder äquivalente Methoden in `pipewire-rs`) gelesen und geschrieben werden können.
- **Setzen und Lesen von Standard-Audio-Geräten (`default.audio.sink/source`):**
    - Die Namen der Properties für Standardgeräte sind typischerweise `"default.audio.sink"` und `"default.audio.source"`.
    - Der Wert dieser Properties ist die ID (oft eine `u32`) des entsprechenden PipeWire-Node-Objekts, das als Standard-Sink oder -Source dienen soll.
    - Um das Standardgerät zu ändern, wird `metadata.set_property("default.audio.sink", "neue_node_id_als_string")` aufgerufen.
    - Um das aktuelle Standardgerät zu lesen, wird `metadata.property("default.audio.sink")` verwendet.
- **Events:** Änderungen an Metadaten (einschließlich Standardgeräten) lösen Events auf dem `Metadata`-Objekt oder der `Registry` aus, die der `PipeWireClientService` abonnieren muss, um `DefaultDeviceChanged`-Events zu generieren.

## 5. MCP Client (system::mcp_client mit `mcp_client_rs`)

Der Model Context Protocol (MCP) Client ermöglicht die Anbindung an KI-Dienste. NovaDE startet lokale MCP-Server-Prozesse und verwaltet die Kommunikation.

### 5.1. Prozessmanagement

Das Starten und Überwachen lokaler MCP-Server-Prozesse erfolgt mit `tokio::process::Command`.

- **Robuste Methoden:**
    
    - **Starten:** `tokio::process::Command::new("path/to/mcp_server_executable")` wird verwendet, um den Prozess zu konfigurieren (Argumente, Umgebungsvariablen). `Command::spawn()` startet den Prozess asynchron.
    - **Überwachen:**
        - Das von `spawn()` zurückgegebene `Child`-Objekt bietet eine `wait()`-Methode (als Future), um auf das Beenden des Prozesses zu warten und den Exit-Status zu erhalten.
        - Regelmäßige Health-Checks (falls vom MCP-Server unterstützt) oder das Überwachen der Stdio-Pipes können ebenfalls zur Überwachung dienen.
        - Bei unerwartetem Beenden des Prozesses sollte ein Fehler geloggt und ggf. ein Neustartversuch unternommen werden (mit Backoff-Strategie).
- **Handhabung von Stdio (`stdin`/`stdout`):**
    
    - `mcp_client_rs::McpClient` kommuniziert typischerweise über `stdin` und `stdout` mit dem Serverprozess.
    - `Command::stdin(Stdio::piped())` und `Command::stdout(Stdio::piped())` müssen gesetzt werden.
    - Die von `child.stdin.take().unwrap()` und `child.stdout.take().unwrap()` zurückgegebenen `ChildStdin` und `ChildStdout` (die `AsyncWrite` bzw. `AsyncRead` implementieren) werden dann an den `McpClient` übergeben (z.B. beim Konstruktor des Clients).
- **Beispiel (konzeptionell):**
    
    Rust
    
    ```
    use tokio::process::Command;
    use std::process::Stdio;
    
    // async fn start_local_mcp_server(config: &McpServerConfig::LocalExecutable) -> Result<mcp_client_rs::McpClient, McpSystemClientError> {
    //     let mut command = Command::new(&config.executable_path);
    //     command.args(&config.args);
    //     command.stdin(Stdio::piped());
    //     command.stdout(Stdio::piped());
    //     // command.stderr(Stdio::piped()); // Für Fehler-Logging
    
    //     let mut child = command.spawn().map_err(McpSystemClientError::ProcessSpawnFailed)?;
    
    //     let stdin = child.stdin.take().ok_or(McpSystemClientError::StdioUnavailable("stdin".into()))?;
    //     let stdout = child.stdout.take().ok_or(McpSystemClientError::StdioUnavailable("stdout".into()))?;
    
    //     let mcp_client = mcp_client_rs::McpClient::new(stdout, stdin); // Annahme einer solchen API
    
    //     // Optional: Task zur Überwachung des Kindprozesses
    //     tokio::spawn(async move {
    //         match child.wait().await {
    //             Ok(status) => { /* log_info!("MCP server exited with status: {}", status); */ },
    //             Err(e) => { /* log_error!("Failed to wait for MCP server: {}", e); */ }
    //         }
    //         // Hier ggf. ServerConnectionStateChanged-Event senden
    //     });
    
    //     Ok(mcp_client)
    // }
    ```
    

### 5.2. Notification Handling

Asynchrone `McpMessage::Notification`-Nachrichten vom Server müssen in einem Hintergrund-Task überwacht werden.

- **Effiziente Überwachung:**
    - Der `mcp_client_rs::McpClient` sollte eine Methode wie `receive_message()` oder `notifications_stream()` bereitstellen, die einen Stream von `McpMessage`s zurückgibt.
    - Dieser Stream wird in einem dedizierten `tokio::task` konsumiert.
    - `while let Some(message) = stream.next().await` wird verwendet, um Nachrichten zu empfangen.
    - Wenn `message` eine `McpMessage::Notification` ist, wird sie in ein `McpClientSystemEvent::NotificationReceived` umgewandelt und über den `SystemEventBridge` publiziert.
- **Beispiel (konzeptionell):**
    
    Rust
    
    ```
    // In McpConnection::new() oder einer ähnlichen Methode
    // let client_clone = self.mcp_client.clone(); // Annahme, McpClient ist Arc-wrapped oder hat eine clone-bare Listener-API
    // let event_bridge = global_system_event_bridge();
    // tokio::spawn(async move {
    //     // Annahme: client_clone.message_stream() liefert einen Stream<Result<McpMessage, _>>
    //     let mut stream = client_clone.message_stream();
    //     while let Some(result) = stream.next().await {
    //         match result {
    //             Ok(McpMessage::Notification(notification)) => {
    //                 event_bridge.publish(SystemLayerEvent::McpClient(
    //                     McpClientSystemEvent::NotificationReceived {
    //                         server_id: self.server_config.id, // Annahme: Server-ID ist verfügbar
    //                         notification,
    //                     }
    //                 )).await;
    //             }
    //             Ok(_) => { /* Andere Nachrichtentypen ignorieren oder behandeln */ }
    //             Err(e) => {
    //                 // log_error!("Error receiving MCP message: {:?}", e);
    //                 // event_bridge.publish(SystemLayerEvent::McpClient(
    //                 //     McpClientSystemEvent::CommunicationError { server_id: self.server_config.id, error: e.into() }
    //                 // )).await;
    //                 break; // Stream beenden bei Fehler
    //             }
    //         }
    //     }
    // });
    ```
    

### 5.3. Request Timeouts

Die Implementierung von Timeouts für `client.send_request_json()` ist wichtig, um auf nicht antwortende Server zu reagieren.

- **`tokio::time::timeout`:** Die `tokio::time::timeout(duration, future)`-Funktion ist hierfür ideal.
    - Sie wrappt den `Future` des `send_request_json()`-Aufrufs.
    - Wenn der `Future` innerhalb der `duration` nicht auflöst, gibt `timeout` ein `Err(Elapsed)` zurück.
    - Anderenfalls gibt es das `Result` des inneren `Future`s zurück.
- **Beispiel:**
    
    Rust
    
    ```
    use tokio::time::{timeout, Duration};
    
    // async fn send_request_with_timeout(
    //     client: &mcp_client_rs::McpClient,
    //     request: McpRequest, // Annahme: MCP-Request-Typ
    //     timeout_duration: Duration,
    // ) -> Result<McpResponse, McpSystemClientError> { // Annahme: MCP-Response-Typ
    //     match timeout(timeout_duration, client.send_request_json(&request)).await {
    //         Ok(Ok(response)) => Ok(response),
    //         Ok(Err(mcp_err)) => Err(McpSystemClientError::RequestFailed(mcp_err)), // Fehler vom McpClient
    //         Err(_elapsed) => Err(McpSystemClientError::Timeout), // Timeout-Fehler
    //     }
    // }
    ```
    
    Dieses Ergebnis wird dann ggf. als `McpClientSystemEvent::RequestFailed` publiziert.

### 5.4. API-Key Management

API-Keys für `RemoteHttp`-Verbindungen werden über den `SecretsServiceClientService` abgerufen und müssen sicher an den `McpClient` übergeben werden.

- **Sichere Übergabe:**
    - Der `McpConnectionManager` oder `DefaultSystemMcpService` fordert den API-Key vom `SecretsServiceClientService` an, wenn eine `RemoteHttp`-Verbindung konfiguriert wird.
    - Der abgerufene Key (typischerweise ein `String` oder `Vec<u8>`) wird dann an den `mcp_client_rs::McpClient` übergeben, wenn dieser für die Remote-Verbindung initialisiert wird.
    - `mcp_client_rs` sollte eine Methode oder Konfigurationsoption bieten, um den API-Key für HTTP-Header (z.B. `Authorization: Bearer <key>`) zu setzen.
    - Der API-Key sollte **niemals** hartcodiert oder unverschlüsselt in Konfigurationsdateien gespeichert werden. Die Verwendung des Secret Service ist hier die korrekte Vorgehensweise.
    - Es ist wichtig sicherzustellen, dass der API-Key nur im Speicher gehalten wird, solange er benötigt wird, und nicht unnötig geloggt oder anderweitig exponiert wird.

## 6. Schichtübergreifende Synchronisation und Fehlerbehandlung

Die Koordination zwischen verschiedenen Event-Loops und eine konsistente Fehlerbehandlung sind für die Stabilität des Gesamtsystems unerlässlich.

### 6.1. Event-Loops (calloop vs. tokio vs. GLib)

NovaDE verwendet `calloop` für Wayland/Compositor-Events und `tokio` für D-Bus, MCP und interne Timer. Eine potenzielle GLib-Main-Loop könnte von der UI-Schicht (nicht Teil der Systemschicht-Spezifikation, aber relevant für die Integration) verwendet werden.

- **Best Practices für Kommunikation und Datenübergabe:**
    - **`calloop` (Compositor-Thread) <-> `tokio` (Service-Threads):**
        - **Von `tokio` zu `calloop`:**
            - Wenn ein Tokio-Task (z.B. D-Bus-Signal-Handler) eine Aktion im Compositor auslösen muss (z.B. Fenster neu zeichnen lassen), kann dies über einen `calloop::channel` geschehen. Der Tokio-Task sendet eine Nachricht, der `calloop`-Eventloop empfängt sie und führt eine entsprechende Aktion im Compositor-Zustand aus.
            - Alternativ kann ein `Arc<Mutex<T>>` für geteilten Zustand verwendet werden, wobei der Tokio-Task den Zustand modifiziert und der Compositor-Thread ihn periodisch prüft oder durch ein Signal (z.B. `calloop::ping`) aufgeweckt wird.
        - **Von `calloop` zu `tokio`:**
            - Wenn eine Aktion im Compositor (z.B. Benutzerinteraktion) einen Tokio-basierten Service aufrufen muss, kann der Compositor-Code einen `tokio::task` spawnen (wenn der `DesktopState` Zugriff auf eine Tokio-Runtime hat) oder eine Nachricht über einen `std::sync::mpsc` (oder `crossbeam_channel`) an einen lauschenden Tokio-Task senden.
            - Der `SystemEventBridge` (basierend auf `tokio::sync::broadcast`) ist der primäre Mechanismus für Events vom Compositor (oder anderen Systemmodulen) zu Tokio-basierten Listenern.
    - **`tokio` (Systemschicht) <-> GLib (UI-Schicht, falls GTK4):**
        - GTK4 ist nicht threadsicher. Alle UI-Operationen müssen im GLib-Main-Thread ausgeführt werden.82
        - **Von `tokio` zu GLib/GTK4:**
            - `glib::MainContext::spawn_local()` oder `glib::idle_add_local()` sind die Standardmechanismen, um Code aus einem Tokio-Thread im GLib-Main-Thread auszuführen.
            - Ein Tokio-Task kann ein Ergebnis berechnen und dann `glib::MainContext::default().spawn_local(...)` verwenden, um einen Closure auszuführen, der die UI aktualisiert.
            - Die `SystemEventBridge`-Events (`tokio::sync::broadcast::Receiver`) müssen im GLib-Main-Thread konsumiert werden. Ein `glib::MainContext::spawn_local` kann einen `async` Block starten, der auf dem `broadcast::Receiver` lauscht und bei neuen Events UI-Updates durchführt.88
                
                Rust
                
                ```
                // In der UI-Schicht (GTK4)
                // let system_event_rx = global_system_event_bridge().subscribe();
                // glib::MainContext::default().spawn_local(async move {
                //     loop {
                //         match system_event_rx.recv().await {
                //             Ok(system_event) => {
                //                 // UI basierend auf system_event aktualisieren
                //             }
                //             Err(broadcast::error::RecvError::Lagged(_)) => { /*... */ }
                //             Err(broadcast::error::RecvError::Closed) => break,
                //         }
                //     }
                // });
                ```
                
        - **Von GLib/GTK4 zu `tokio`:**
            - Ein GTK-Signal-Handler (der im GLib-Main-Thread läuft) kann einen `tokio::task` auf einer globalen Tokio-Runtime spawnen, um eine langlaufende Operation auszuführen.
            - Das Ergebnis kann dann wieder über `glib::MainContext::spawn_local()` an die UI zurückgespielt werden.
- **Wichtige Überlegungen:**
    - **Blockierung vermeiden:** Kein Thread sollte den Event-Loop eines anderen Threads blockieren. Insbesondere dürfen `calloop`-Callbacks und GLib-Main-Loop-Handler keine langwierigen synchronen Operationen ausführen.
    - **Daten-Ownership und Thread-Safety:** `Send` und `Sync` müssen beachtet werden, wenn Daten zwischen Threads übergeben werden. `Arc<Mutex/RwLock<T>>` für geteilten Zustand oder Kanäle für Nachrichten sind üblich.

### 6.2. Fehler-Wrapping

Eine konsistente Strategie zum Wrappen von Fehlern aus unteren Schichten in die spezifischen Fehler-Enums der Systemschicht-Module ist entscheidend für Robustheit und Debugging.

- **Konsistente Strategien:**
    - **`thiserror` verwenden:** Jedes Modul der Systemschicht (`system::compositor`, `system::input`, etc.) sollte sein eigenes Fehler-Enum definieren (z.B. `CompositorCoreError`, `InputError`), das mit `#[derive(thiserror::Error)]` versehen ist.
    - **`#[from]`-Annotation:** Für Fehler, die direkt von einer Abhängigkeits-Crate stammen und 1:1 übernommen werden können oder eine klare Entsprechung haben, kann `#[from]` verwendet werden.
        
        Rust
        
        ```
        // #
        // pub enum McpSystemClientError {
        //     #[error("MCP client internal error: {0}")]
        //     ClientError(#[from] mcp_client_rs::Error),
        //     //...
        // }
        ```
        
    - **`#[source]`-Annotation:** Wenn ein Fehler aus einer unteren Schicht gewrappt und mit zusätzlichem Kontext versehen wird, sollte der ursprüngliche Fehler mit `#[source]` als Ursache beibehalten werden. Dies ist entscheidend für die `source()`-Kette.
        
        Rust
        
        ```
        // #
        // pub enum AudioManagementError {
        //     #[error("PipeWire connection failed")]
        //     ConnectionError(#[source] pipewire::Error), // Annahme: pipewire::Error ist der Fehlertyp
        //     //...
        // }
        ```
        
    - **Spezifische Fehlervarianten:** Erstellen Sie spezifische Fehlervarianten für häufige oder für die Domänenschicht relevante Fehlerfälle. Dies ermöglicht eine präzisere Fehlerbehandlung in höheren Schichten.
    - **Weitergabe an die Domänenschicht:** Die Domänenschicht sollte idealerweise nicht direkt mit den spezifischen Fehlertypen der Systemschicht-Module konfrontiert werden. Eine übergeordnete `SystemLayerError`-Enum, die alle modul-spezifischen Systemfehler wrappt, oder ein generischerer Ansatz (z.B. `anyhow::Error` mit Kontext) kann verwendet werden, um Fehler an die Domänenschicht zu propagieren. Die `source()`-Kette bleibt dabei erhalten und ermöglicht bei Bedarf eine detaillierte Fehleranalyse.
    - **Logging:** Fehler sollten auf der Systemschicht geloggt werden, bevor sie weitergegeben werden, insbesondere wenn sie nicht trivial sind oder wichtige Kontextinformationen enthalten.

## 7. Schlussfolgerungen und Empfehlungen

Die Entwicklung der Systemschicht für NovaDE stellt eine komplexe Aufgabe dar, die ein tiefes Verständnis der zugrundeliegenden Technologien und Protokolle erfordert. Die gewählte Architektur mit einer klaren Trennung in Module und der Verwendung etablierter Bibliotheken wie Smithay, libinput, zbus und pipewire-rs bildet eine solide Grundlage.

**Zentrale Herausforderungen und Lösungsansätze:**

- **Zustandsmanagement im Compositor:** Die Komplexität des `DesktopState` sollte durch Modularisierung und klare Verantwortlichkeiten reduziert werden. Die `delegate_*!`-Makros von Smithay sind hierbei wertvolle Werkzeuge. Geteilter Zustand erfordert sorgfältige Synchronisation mit `Arc<Mutex/RwLock<T>>`, wobei feingranulare Locks und die Vermeidung von Deadlocks essentiell sind.
- **Wayland-Protokollimplementierung:** Die korrekte Implementierung spezifischer Protokolle wie `wlr-output-management` (Atomarität, Serial-Handling), `wlr-output-power-management` (exklusive Kontrolle) und `wlr-layer-shell` (exklusive Zonen) ist entscheidend für die Funktionalität und Stabilität des Desktops. Die Protokoll-Spezifikationen und existierende Implementierungen (z.B. in wlroots-basierten Compositors) dienen als wichtige Referenzen.
- **Asynchronität und Event-Loop-Integration:** Die Koordination zwischen `calloop` (Compositor) und `tokio` (D-Bus, MCP, Timer) erfordert robuste IPC-Mechanismen wie Kanäle (`tokio::sync::mpsc`, `calloop::channel`) und geteilten Zustand (`Arc<Mutex<T>>`). Für die UI-Integration (GLib) sind `glib::MainContext::spawn_local` und `glib::idle_add_local` Schlüsselkomponenten.
- **Fehlerbehandlung:** Eine konsistente Fehler-Wrapping-Strategie mit `thiserror` unter Beibehaltung der `source()`-Kette ist für Diagnose und Robustheit unerlässlich.

**Empfehlungen für die weitere Entwicklung:**

1. **Priorisierung von Tests:** Angesichts der Komplexität, insbesondere bei der Interaktion verschiedener Protokolle und Event-Loops, sollten Unit- und Integrationstests frühzeitig und umfassend implementiert werden.
2. **Dokumentation und Code-Beispiele:** Die interne Dokumentation sollte detailliert sein. Das Studium von Open-Source-Compositors, die ähnliche Technologien verwenden (z.B. Anvil für Smithay, andere Compositors für wlroots-Protokolle), kann wertvolle Einblicke liefern.
3. **Performance-Analyse:** Regelmäßige Performance-Analysen, insbesondere in kritischen Pfaden wie Rendering und Eingabeverarbeitung, sind notwendig, um Engpässe frühzeitig zu erkennen.
4. **Sicherheitsaspekte:** Bei der Interaktion mit externen Diensten (D-Bus, MCP) und der Verarbeitung von Client-Eingaben müssen Sicherheitsaspekte (Input-Validierung, Berechtigungsmanagement via PolicyKit) berücksichtigt werden.
5. **Iterative Entwicklung:** Aufgrund der Komplexität ist ein iterativer Ansatz, bei dem einzelne Module und Protokolle schrittweise implementiert und getestet werden, empfehlenswert.

Durch die konsequente Anwendung von Best Practices und eine sorgfältige Implementierung der spezifizierten Komponenten kann die Systemschicht von NovaDE eine stabile und leistungsfähige Basis für die darüberliegenden Schichten bilden.

# NovaDE Ultra-Feinspezifikation: Domänenschicht & Wayland/Smithay-Integration

Dieses Dokument definiert die detaillierten technischen Spezifikationen für die Domänenschicht des NovaDE-Projekts und deren Integration mit dem Wayland-Compositor, der unter Verwendung des Smithay-Toolkits implementiert wird. Es dient als direkter Implementierungsleitfaden für Entwickler und beschreibt die Kernarchitektur, Datenstrukturen, APIs und Interaktionsmuster.

## 1. Domänenschicht-Spezifikationen

Die Domänenschicht kapselt die Kernlogik und den Zustand der NovaDE-Desktop-Umgebung. Sie ist bewusst unabhängig von UI-Details und systemspezifischen Implementierungen konzipiert, um eine klare Trennung der Verantwortlichkeiten zu gewährleisten und die Wartbarkeit sowie Testbarkeit des Systems zu verbessern. Diese Schicht bildet das Herzstück von NovaDE und definiert die grundlegenden Entitäten, Regeln und Prozesse, die das Verhalten der Desktop-Umgebung bestimmen.

### 1.1. Modul: `domain::theming`

Dieses Modul ist verantwortlich für die Logik des Erscheinungsbilds (Theming), die Verwaltung von Design-Tokens, die Interpretation von Theme-Definitionen und die Ermöglichung dynamischer Theme-Wechsel. Es stellt die Grundlage für ein konsistentes und anpassbares Benutzererlebnis dar.

Referenzierte Dokumente: B1 Domänenschicht.md

#### 1.1.1. Datenstrukturen (`domain::theming::types`)

Die folgenden Datenstrukturen definieren die Kernentitäten für das Theming-System:

- **`TokenIdentifier`**: Ein Wrapper für `String` zur eindeutigen Identifizierung von Design-Tokens (z.B. `"color.background.primary"`).
    
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Serialize`, `Deserialize`, `PartialOrd`, `Ord`, `Display`, `From<&str>`.
    - Invarianten: Darf nicht leer sein und darf nur ASCII-Alphanumerische Zeichen, Punkte (`.`) und Bindestriche (`-`) enthalten. Diese strikten Invarianten sind entscheidend, um Parsing-Fehler zu vermeiden und eine konsistente Referenzierung von Tokens im gesamten System sicherzustellen, was die Token-Verwaltung vereinfacht und potenzielle Laufzeitprobleme reduziert.
- **`TokenValue`**: Ein Enum zur Repräsentation der verschiedenen Typen von Token-Werten.
    
    - Varianten: `Color(String)`, `Dimension(String)`, `FontSize(String)`, `FontFamily(String)`, `FontWeight(String)`, `LineHeight(String)`, `LetterSpacing(String)`, `Border(String)`, `Shadow(String)`, `Radius(String)`, `Spacing(String)`, `ZIndex(i32)`, `Opacity(f64)` (Wertebereich 0.0-1.0), `Text(String)`, `Reference(TokenIdentifier)`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`.
    - Die `Opacity`-Werte werden bei der Zuweisung oder Verarbeitung automatisch auf den Bereich [0.0,1.0] geklemmt, um ungültige Zustände zu verhindern. Die `Reference` Variante ermöglicht es, Tokens aufeinander verweisen zu lassen, was eine flexible und wartbare Theme-Struktur unterstützt.
- **`RawToken`**: Eine Struktur zur Repräsentation eines Tokens, wie es typischerweise aus einer Konfigurationsdatei gelesen wird.
    
    - Felder: `id: TokenIdentifier`, `value: TokenValue`, `description: Option<String>`, `group: Option<String>`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`.
- **`TokenSet`**: Ein Typalias für `std::collections::BTreeMap<TokenIdentifier, RawToken>`.
    
    - Die Verwendung von `BTreeMap` stellt sicher, dass Tokens bei der Verarbeitung oder Anzeige konsistent sortiert sind, was die Fehlersuche und das Management erleichtert.
- **`ThemeIdentifier`**: Ein Wrapper für `String` zur eindeutigen Identifizierung von Themes.
    
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash`, `Serialize`, `Deserialize`, `PartialOrd`, `Ord`, `Display`, `From<&str>`.
    - Invarianten: Darf nicht leer sein. Empfohlene Zeichen sind `a-z`, `A-Z`, `0-9`, `.` und `-`, um Kompatibilität über verschiedene Dateisysteme und Kontexte hinweg zu gewährleisten.
- **`ColorSchemeType`**: Ein Enum zur Unterscheidung zwischen hellen und dunklen Farbschemata.
    
    - Varianten: `Light`, `Dark`.
    - Implementiert: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Serialize`, `Deserialize`, `Default` (standardmäßig `Light`).
- **`AccentColor`**: Eine Struktur zur Repräsentation einer Akzentfarbe innerhalb eines Themes.
    
    - Felder: `name: Option<String>`, `value: novade_core::types::Color`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`.
    - Für `Eq` und `Hash` auf dem Feld `value` (vom Typ `novade_core::types::Color`) ist möglicherweise eine manuelle Implementierung oder eine Konvertierung in einen kanonischen Hex-String erforderlich, um konsistentes Verhalten in Hash-basierten Sammlungen sicherzustellen.
- **`ThemeVariantDefinition`**: Definiert spezifische Token-Werte für eine bestimmte Theme-Variante (z.B. helles oder dunkles Schema).
    
    - Felder: `applies_to_scheme: ColorSchemeType`, `tokens: TokenSet`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`.
- **`ThemeDefinition`**: Die vollständige Definition eines Themes, inklusive Basis-Tokens, Varianten und Akzentfarb-Informationen.
    
    - Felder: `id: ThemeIdentifier`, `name: String`, `description: Option<String>`, `author: Option<String>`, `version: Option<String>`, `base_tokens: TokenSet`, `variants: Vec<ThemeVariantDefinition>`, `supported_accent_colors: Option<Vec<AccentColor>>`, `accentable_tokens: Option<HashMap<TokenIdentifier, AccentModificationType>>`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`.
    - Das Feld `accentable_tokens` ermöglicht eine feingranulare Steuerung, wie Akzentfarben auf einzelne Tokens angewendet werden.
- **`AccentModificationType`**: Ein Enum, das beschreibt, wie ein Token durch eine Akzentfarbe modifiziert werden soll.
    
    - Varianten: `DirectReplace`, `Lighten(f32)`, `Darken(f32)`.
    - Die Parameter für `Lighten` und `Darken` (z.B. `f32`) geben den Grad der Aufhellung oder Abdunkelung an.
- **`AppliedThemeState`**: Repräsentiert den aktuell aktiven, vollständig aufgelösten Theme-Zustand, der an die UI-Schicht übergeben wird.
    
    - Felder: `theme_id: ThemeIdentifier`, `color_scheme: ColorSchemeType`, `active_accent_color: Option<AccentColor>`, `resolved_tokens: std::collections::BTreeMap<TokenIdentifier, String>`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`.
    - Invarianten: `resolved_tokens` darf keine `TokenValue::Reference` mehr enthalten; alle Werte müssen zu finalen Strings aufgelöst sein. Dies ist kritisch, da die UI-Schicht keine Token-Referenzen interpretieren soll.
- **`ThemingConfiguration`**: Benutzerspezifische Einstellungen für das Theming.
    
    - Felder: `selected_theme_id: ThemeIdentifier`, `preferred_color_scheme: ColorSchemeType`, `selected_accent_color: Option<novade_core::types::Color>`, `custom_user_token_overrides: Option<TokenSet>`.
    - Implementiert: `Debug`, `Clone`, `PartialEq`, `Serialize`, `Deserialize`, `Default`.
    - Die `Default`-Implementierung stellt sicher, dass das System auch ohne explizite Benutzerkonfiguration einen validen Ausgangszustand hat.

#### 1.1.2. Fehlerbehandlung (`domain::theming::errors`)

Das `ThemingError` Enum, definiert mit `thiserror`, kapselt alle potenziellen Fehler, die im Theming-Modul auftreten können. Dies ermöglicht eine präzise und kontextbezogene Fehlerbehandlung.

- **Varianten**:
    - `TokenFileParseError { path: PathBuf, source: serde_json::Error }`: Fehler beim Parsen einer Token-JSON-Datei.
    - `TokenFileIoError { path: PathBuf, source: std::io::Error }`: Fehler beim Lesen/Schreiben einer Token-Datei.
    - `InvalidTokenData { path: PathBuf, message: String }`: Ungültige Datenstruktur in einer Token-Datei.
    - `InvalidTokenValue { token_id: TokenIdentifier, value_string: String, reason: String }`: Ein Token-Wert entspricht nicht dem erwarteten Format oder Typ.
    - `CyclicTokenReference { token_id: TokenIdentifier, cycle_path: Vec<TokenIdentifier> }`: Zyklische Abhängigkeit bei der Token-Referenzierung entdeckt.
    - `ThemeFileLoadError { theme_id: ThemeIdentifier, path: PathBuf, source: serde_json::Error }`: Fehler beim Parsen einer Theme-JSON-Datei.
    - `ThemeFileIoError { theme_id: ThemeIdentifier, path: PathBuf, source: std::io::Error }`: Fehler beim Lesen/Schreiben einer Theme-Datei.
    - `InvalidThemeData { theme_id: ThemeIdentifier, path: PathBuf, message: String }`: Ungültige Datenstruktur in einer Theme-Datei.
    - `ThemeNotFound { theme_id: ThemeIdentifier }`: Das angeforderte Theme konnte nicht gefunden werden.
    - `MissingTokenReference { referencing_token_id: TokenIdentifier, target_token_id: TokenIdentifier }`: Ein referenziertes Token existiert nicht.
    - `MaxReferenceDepthExceeded { token_id: TokenIdentifier, depth: u8 }`: Die maximale Tiefe für Token-Referenzen wurde überschritten, um Endlosschleifen zu verhindern.
    - `ThemeApplicationError { message: String, source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`: Allgemeiner Fehler bei der Anwendung eines Themes.
    - `FallbackThemeLoadError { source: Box<dyn std::error::Error + Send + Sync + 'static>> }`: Fehler beim Laden des Fallback-Themes.
    - `InitialConfigurationError(String)`: Fehler bei der Initialisierung der Theming-Konfiguration.
    - `InternalStateError(String)`: Unerwarteter interner Fehler im Theming-Modul.
    - `EventSubscriptionError(String)`: Fehler bei der Registrierung für Theme-Änderungs-Events.
    - `AccentColorApplicationError { theme_id: ThemeIdentifier, accent_color: novade_core::types::Color, details: String }`: Fehler bei der Anwendung einer Akzentfarbe.
    - `TokenResolutionError { token_id: TokenIdentifier, message: String, source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`: Fehler während der Auflösung eines spezifischen Tokens.
    - `FilesystemError(#[from] novade_core::errors::CoreError)`: Fehler im Zusammenhang mit Dateisystemoperationen, die von `novade_core` stammen.

#### 1.1.3. Kernlogik (`domain::theming::logic`)

Die Kernlogik des Theming-Moduls umfasst das Laden, Validieren und Auflösen von Themes und Tokens.

- **Konstante**: `MAX_TOKEN_RESOLUTION_DEPTH: u8 = 16`. Dieser Wert begrenzt die Rekursionstiefe bei der Auflösung von Token-Referenzen, um Endlosschleifen und übermäßigen Ressourcenverbrauch zu verhindern. Der Wert 16 stellt einen pragmatischen Kompromiss zwischen Flexibilität und Sicherheit dar.
    
- **Token/Theme Laden & Validieren**:
    
    - `async fn load_raw_tokens_from_file(path: &Path, config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>) -> Result<TokenSet, ThemingError>`: Liest eine JSON-Datei vom angegebenen Pfad, parst deren Inhalt zu einem `TokenSet` und validiert auf Duplikate sowie das grundlegende Format der Tokens. Nutzt den `ConfigServiceAsync` für asynchrone Dateioperationen.
    - `async fn load_and_validate_token_files(paths: &, config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>) -> Result<TokenSet, ThemingError>`: Lädt Tokens von mehreren Pfaden, führt sie zusammen (wobei Benutzer-Tokens System-Tokens überschreiben) und validiert das Ergebnis auf Zyklen und andere Inkonsistenzen.
    - `async fn load_theme_definition_from_file(path: &Path, theme_id_override: Option<ThemeIdentifier>, config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>) -> Result<ThemeDefinition, ThemingError>`: Liest eine JSON-Datei und parst sie zu einer `ThemeDefinition`. Ermöglicht das Überschreiben der Theme-ID, falls diese nicht im Dateinamen enthalten ist oder anders abgeleitet werden muss.
    - `async fn load_and_validate_theme_files(paths: &, available_tokens: &TokenSet, config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>) -> Result<Vec<ThemeDefinition>, ThemingError>`: Lädt mehrere Theme-Definitionen von den angegebenen Pfaden und validiert die darin enthaltenen Token-Referenzen gegen den Satz der global verfügbaren Tokens (`available_tokens`).
    - `fn validate_tokenset_for_cycles(tokens: &TokenSet) -> Result<(), ThemingError>`: Verwendet eine Tiefensuche (Depth-First Search, DFS), um zyklische Referenzen innerhalb eines `TokenSet` zu erkennen.
- **Token Resolution Pipeline**:
    
    - `fn resolve_tokens_for_config(config: &ThemingConfiguration, theme_def: &ThemeDefinition, global_tokens: &TokenSet, accentable_tokens_map: &HashMap<TokenIdentifier, AccentModificationType>, max_depth: u8) -> Result<BTreeMap<TokenIdentifier, String>, ThemingError>`: Diese zentrale Funktion ist verantwortlich für die Auflösung aller Tokens zu ihren finalen String-Werten basierend auf der aktuellen Konfiguration.
        
        1. Startet mit einer Basis-Token-Menge, die aus globalen Tokens und den Basis-Tokens des ausgewählten Themes besteht.
        2. Wendet die Tokens der passenden Theme-Variante (hell/dunkel) an, die die Basis-Tokens überschreiben.
        3. Appliziert die ausgewählte Akzentfarbe. Dies geschieht durch Modifikation der Tokens, die im `accentable_tokens_map` des Themes definiert sind, gemäß dem `AccentModificationType`.
        4. Wendet benutzerspezifische Token-Overrides an, die die höchste Priorität haben.
        5. Führt eine rekursive Auflösung für alle Tokens vom Typ `TokenValue::Reference` durch. Dabei wird die Funktion `resolve_single_token_value` (nicht explizit in der API, aber impliziert durch die Logik) verwendet, die eine Zyklenerkennung und eine Begrenzung der Rekursionstiefe (`max_depth`) implementiert.
        6. Konvertiert alle aufgelösten `TokenValue`-Instanzen in ihre finalen String-Repräsentationen (z.B. Farbcodes, Dimensionsangaben).
        
        - Das Ergebnis ist ein `BTreeMap<TokenIdentifier, String>`, das keine Referenzen mehr enthält und direkt von der UI-Schicht verwendet werden kann.
- **Caching**:
    
    - Ein Cache für `AppliedThemeState` wird implementiert, um die wiederholte, potenziell aufwendige Auflösung von Tokens zu vermeiden.
    - Der Cache-Schlüssel (`CacheKey`) wird gebildet aus: `(ThemeIdentifier, ColorSchemeType, Option<String> /* hex von AccentColor */, u64 /* hash von Overrides */)`.
    - Diese Strategie stellt sicher, dass bei relevanten Änderungen der Konfiguration ein neuer Zustand generiert wird, während bei unveränderter Konfiguration auf den gecachten Zustand zurückgegriffen werden kann.
- **Fallback**:
    
    - `fn generate_fallback_applied_state() -> AppliedThemeState`: Generiert einen minimalen, aber funktionsfähigen `AppliedThemeState` aus einkompilierten JSON-Definitionen. Dies stellt sicher, dass das System auch dann ein grundlegendes Erscheinungsbild hat, wenn keine Theme-Dateien geladen werden können oder Konfigurationsfehler auftreten.

#### 1.1.4. API (`domain::theming::service::ThemingEngine`)

Die `ThemingEngine` ist die zentrale Schnittstelle für andere Teile des Systems, um auf Theming-Funktionalitäten zuzugreifen.

- **Struktur `ThemingEngineInternalState`**: Kapselt den internen, veränderlichen Zustand der Engine.
    
    - Felder: `current_config: ThemingConfiguration`, `available_themes: Vec<ThemeDefinition>`, `global_raw_tokens: TokenSet`, `applied_state: AppliedThemeState`, `theme_load_paths: Vec<PathBuf>`, `token_load_paths: Vec<PathBuf>`, `resolved_state_cache: HashMap<CacheKey, AppliedThemeState>`, `config_service: Arc<dyn novade_core::config::ConfigServiceAsync>`.
    - Die Verwendung von `Arc` für `config_service` ermöglicht die gemeinsame Nutzung dieses Dienstes.
- **Struktur `ThemingEngine`**: Die öffentliche API-Struktur.
    
    - Felder: `internal_state: Arc<tokio::sync::Mutex<ThemingEngineInternalState>>`, `event_sender: tokio::sync::broadcast::Sender<ThemeChangedEvent>`.
    - Die Verwendung von `Arc<tokio::sync::Mutex<...>>` gewährleistet Thread-sicheren Zugriff auf den internen Zustand in einer asynchronen Umgebung.
    - `tokio::sync::broadcast::Sender` wird für die Veröffentlichung von Theme-Änderungs-Events verwendet.
- **Methoden**:
    
    - `async fn new(initial_config: ThemingConfiguration, theme_load_paths: Vec<PathBuf>, token_load_paths: Vec<PathBuf>, config_service: Arc<dyn novade_core::config::ConfigServiceAsync>, broadcast_capacity: usize) -> Result<Self, ThemingError>`: Konstruktor, der die Engine initialisiert, initiale Themes und Tokens lädt und den ersten `AppliedThemeState` generiert.
    - `async fn get_current_theme_state(&self) -> AppliedThemeState`: Gibt den aktuell angewendeten und aufgelösten Theme-Zustand zurück.
    - `async fn get_available_themes(&self) -> Vec<ThemeDefinition>`: Gibt eine Liste aller verfügbaren Theme-Definitionen zurück.
    - `async fn get_current_configuration(&self) -> ThemingConfiguration`: Gibt die aktuelle Benutzerkonfiguration für das Theming zurück.
    - `async fn update_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>`: Aktualisiert die Benutzerkonfiguration, löst die Tokens neu auf, aktualisiert den Cache und publiziert ein `ThemeChangedEvent`.
    - `async fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>`: Lädt alle Theme- und Token-Dateien von den konfigurierten Pfaden neu, validiert sie, aktualisiert den internen Zustand und den Cache und publiziert ggf. ein `ThemeChangedEvent`.
    - `fn subscribe_to_theme_changes(&self) -> tokio::sync::broadcast::Receiver<ThemeChangedEvent>`: Ermöglicht anderen Modulen, auf Änderungen des Themes zu reagieren.

#### 1.1.5. Events (`domain::theming::events`)

- **`ThemeChangedEvent`**: Wird publiziert, wenn sich der angewendete Theme-Zustand ändert.
    - Payload: `new_state: AppliedThemeState`.
    - Dieses Event ist entscheidend für die dynamische Aktualisierung der Benutzeroberfläche, ohne dass Komponenten den `ThemingEngine` ständig pollen müssen.

### 1.2. Modul: `domain::workspaces`

Dieses Modul ist für die Logik und Verwaltung von Arbeitsbereichen (auch "Spaces" oder "virtuelle Desktops" genannt) zuständig. Es definiert, wie Arbeitsbereiche erstellt, modifiziert, gelöscht und wie Fenster ihnen zugewiesen werden.

Referenzierte Dokumente: B2 Domänenschicht.md

#### 1.2.1. Untermodul: `domain::workspaces::core`

Das `core`-Untermodul definiert die grundlegenden Datenstrukturen, Entitäten und Kernfehler für Arbeitsbereiche.

- **Datenstrukturen (`domain::workspaces::core::types`)**:
    
    - `WorkspaceId`: Ein Typalias für `uuid::Uuid`, um Arbeitsbereiche eindeutig zu identifizieren. Die Verwendung von UUIDs stellt globale Eindeutigkeit sicher, was bei der Synchronisation oder bei verteilten Szenarien vorteilhaft sein kann.
    - `WindowIdentifier`: Ein Wrapper für `String` zur Identifizierung von Fenstern.
        - Implementiert `new(id: impl Into<String>) -> Result<Self, WorkspaceCoreError>`. Die Validierung (z.B. nicht leer) erfolgt hier. Dies stellt sicher, dass Fensterbezeichner immer in einem validen Zustand sind.
    - `WorkspaceLayoutType`: Ein Enum zur Definition der verschiedenen Layout-Modi eines Arbeitsbereichs.
        - Varianten: `Floating`, `TilingHorizontal`, `TilingVertical`, `Maximized`.
        - Implementiert `Default` (standardmäßig `Floating`).
- **Entität `Workspace` (`domain::workspaces::core::mod.rs`)**: Die zentrale Entität, die einen Arbeitsbereich repräsentiert.
    
    - Felder:
        - `id: WorkspaceId`: Eindeutiger Identifikator.
        - `name: String`: Vom Benutzer lesbarer Name des Arbeitsbereichs (z.B. "Arbeit", "Web").
        - `persistent_id: Option<String>`: Eine optionale, persistente ID, die über Sitzungen hinweg stabil bleiben kann (z.B. für das Wiederherstellen von Arbeitsbereichs-Setups).
        - `layout_type: WorkspaceLayoutType`: Der aktuelle Layout-Modus.
        - `window_ids: HashSet<WindowIdentifier>`: Eine Menge der IDs der Fenster, die diesem Arbeitsbereich zugeordnet sind.
        - `created_at: DateTime<Utc>`: Zeitstempel der Erstellung.
        - `icon_name: Option<String>`: Optionaler Name eines Icons zur visuellen Repräsentation.
        - `accent_color_hex: Option<String>`: Optionale Akzentfarbe im Hex-Format für den Arbeitsbereich.
    - Methoden:
        - `new(name: String, persistent_id: Option<String>, layout_type: WorkspaceLayoutType, icon_name: Option<String>, accent_color_hex: Option<String>) -> Result<Self, WorkspaceCoreError>`: Konstruktor für neue Arbeitsbereiche. Validiert Eingaben wie `name` und `accent_color_hex`.
        - `id() -> WorkspaceId`: Gibt die ID des Arbeitsbereichs zurück.
        - `name() -> &str`: Gibt den Namen des Arbeitsbereichs zurück.
        - `rename(&mut self, new_name: String) -> Result<(), WorkspaceCoreError>`: Ändert den Namen des Arbeitsbereichs. Validiert den neuen Namen.
        - `set_layout_type(&mut self, layout_type: WorkspaceLayoutType)`: Ändert den Layout-Typ.
        - `add_window_id(&mut self, window_id: WindowIdentifier)` (`crate`-intern): Fügt eine Fenster-ID hinzu. Die Sichtbarkeit ist auf `crate` beschränkt, da die Fensterzuweisung über den `assignment`-Service erfolgen soll.
        - `remove_window_id(&mut self, window_id: &WindowIdentifier) -> bool` (`crate`-intern): Entfernt eine Fenster-ID.
        - `set_icon_name(&mut self, icon_name: Option<String>)`: Setzt den Icon-Namen.
        - `set_accent_color_hex(&mut self, accent_color_hex: Option<String>) -> Result<(), WorkspaceCoreError>`: Setzt die Akzentfarbe und validiert das Format.
- **Event-Payloads (`domain::workspaces::core::event_data`)**: Strukturen, die als Daten für die verschiedenen Arbeitsbereichs-Events dienen.
    
    - `WorkspaceRenamedData { new_name: String }`
    - `WorkspaceLayoutChangedData { new_layout: WorkspaceLayoutType }`
    - `WindowAddedToWorkspaceData { window_id: WindowIdentifier }`
    - `WindowRemovedFromWorkspaceData { window_id: WindowIdentifier }`
    - `WorkspacePersistentIdChangedData { new_persistent_id: Option<String> }`
    - `WorkspaceIconChangedData { new_icon_name: Option<String> }`
    - `WorkspaceAccentChangedData { new_accent_color_hex: Option<String> }`
- **Fehlerbehandlung (`domain::workspaces::core::errors`)**:
    
    - `WorkspaceCoreError`: Enum für Fehler, die direkt bei der Manipulation einer `Workspace`-Entität auftreten können.
        - Varianten: `InvalidName`, `NameCannotBeEmpty`, `NameTooLong`, `InvalidPersistentId`, `WindowIdentifierEmpty`, `InvalidAccentColorFormat`.

#### 1.2.2. Untermodul: `domain::workspaces::assignment`

Dieses Untermodul stellt die API für die Zuordnung von Fenstern zu Arbeitsbereichen bereit. Es stellt sicher, dass die Regeln für Fensterzuweisungen eingehalten werden.

- **API (`domain::workspaces::assignment::mod.rs`)**: Funktionen, die auf einer veränderlichen `HashMap<WorkspaceId, Workspace>` operieren. Dieser Ansatz ermöglicht es, die Zuordnungslogik von der Hauptverwaltung der Arbeitsbereiche zu entkoppeln, macht aber eine sorgfältige Zustandsverwaltung erforderlich, um Konsistenz zu gewährleisten.
    
    - `assign_window_to_workspace(workspaces: &mut HashMap<WorkspaceId, Workspace>, target_workspace_id: WorkspaceId, window_id: &WindowIdentifier, ensure_unique_assignment: bool) -> Result<(), WindowAssignmentError>`: Weist ein Fenster einem spezifischen Arbeitsbereich zu. Der Parameter `ensure_unique_assignment` steuert, ob ein Fenster zuvor von allen anderen Arbeitsbereichen entfernt werden muss.
    - `remove_window_from_workspace(workspaces: &mut HashMap<WorkspaceId, Workspace>, workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<bool, WindowAssignmentError>`: Entfernt ein Fenster von einem spezifischen Arbeitsbereich. Gibt `true` zurück, wenn das Fenster entfernt wurde.
    - `move_window_to_workspace(workspaces: &mut HashMap<WorkspaceId, Workspace>, window_id: &WindowIdentifier, source_workspace_id: WorkspaceId, target_workspace_id: WorkspaceId) -> Result<(), WindowAssignmentError>`: Verschiebt ein Fenster von einem Quell- zu einem Zielarbeitsbereich.
    - `find_workspace_for_window(workspaces: &HashMap<WorkspaceId, Workspace>, window_id: &WindowIdentifier) -> Option<WorkspaceId>`: Findet den Arbeitsbereich, dem ein bestimmtes Fenster zugewiesen ist.
- **Fehlerbehandlung (`domain::workspaces::assignment::errors`)**:
    
    - `WindowAssignmentError`: Enum für Fehler, die bei der Fensterzuweisung auftreten können.
        - Varianten: `WorkspaceNotFound`, `WindowAlreadyAssigned` (wenn `ensure_unique_assignment` true ist und das Fenster bereits auf dem Ziel-Workspace ist oder auf einem anderen und nicht entfernt werden konnte), `WindowNotAssignedToWorkspace`, `SourceWorkspaceNotFound`, `TargetWorkspaceNotFound`, `WindowNotOnSourceWorkspace`, `CannotMoveToSameWorkspace`, `RuleViolation` (für zukünftige, komplexere Zuweisungsregeln), `Internal`.

#### 1.2.3. Untermodul: `domain::workspaces::config`

Verantwortlich für die Persistenz und das Laden von Arbeitsbereichs-Konfigurationen.

- **Datenstrukturen (`domain::workspaces::config::mod.rs`)**:
    
    - `WorkspaceSnapshot`: Eine serialisierbare Repräsentation eines Arbeitsbereichs für die Speicherung. Enthält nur persistente Eigenschaften.
        - Felder: `persistent_id: String`, `name: String`, `layout_type: WorkspaceLayoutType`, `icon_name: Option<String>`, `accent_color_hex: Option<String>`.
    - `WorkspaceSetSnapshot`: Eine serialisierbare Repräsentation eines Sets von Arbeitsbereichen und des aktiven Arbeitsbereichs.
        - Felder: `workspaces: Vec<WorkspaceSnapshot>`, `active_workspace_persistent_id: Option<String>`.
- **API (`domain::workspaces::config::mod.rs`)**:
    
    - Trait `WorkspaceConfigProvider`: Definiert die Schnittstelle für das Laden und Speichern von Arbeitsbereichs-Konfigurationen.
        - `async fn load_workspace_config(&self) -> Result<WorkspaceSetSnapshot, WorkspaceConfigError>`
        - `async fn save_workspace_config(&self, snapshot: &WorkspaceSetSnapshot) -> Result<(), WorkspaceConfigError>`
    - Struktur `FilesystemConfigProvider`: Eine Implementierung von `WorkspaceConfigProvider`, die `novade_core::config::ConfigServiceAsync` für Dateioperationen verwendet. Dies entkoppelt die Workspace-Logik von den Details der Dateispeicherung.
- **Fehlerbehandlung (`domain::workspaces::config::errors`)**:
    
    - `WorkspaceConfigError`: Enum für Fehler im Zusammenhang mit der Konfigurationspersistenz.
        - Varianten: `LoadError { source: novade_core::errors::CoreError }`, `SaveError { source: novade_core::errors::CoreError }`, `InvalidData { message: String }`, `SerializationError { source: serde_json::Error }`, `DeserializationError { source: serde_json::Error }`, `PersistentIdNotFoundInLoadedSet { persistent_id: String }`, `DuplicatePersistentIdInLoadedSet { persistent_id: String }`.

#### 1.2.4. Untermodul: `domain::workspaces::manager`

Der `WorkspaceManagerService` ist die Hauptschnittstelle zur Verwaltung von Arbeitsbereichen. Er orchestriert die `core`-, `assignment`- und `config`-Logik.

- **API (`domain::workspaces::manager::mod.rs`)**:
    
    - Trait `WorkspaceManagerService`: Definiert die öffentliche API für die Arbeitsbereichsverwaltung.
        - Methoden: `async fn create_workspace(&self, name: String, persistent_id: Option<String>, layout_type: Option<WorkspaceLayoutType>, icon: Option<String>, accent_color: Option<String>) -> Result<Workspace, WorkspaceManagerError>`, `async fn delete_workspace(&self, id: WorkspaceId, fallback_workspace_id: Option<WorkspaceId>) -> Result<(), WorkspaceManagerError>`, `async fn get_workspace(&self, id: WorkspaceId) -> Option<Workspace>`, `async fn all_workspaces_ordered(&self) -> Vec<Workspace>`, `async fn active_workspace_id(&self) -> Option<WorkspaceId>`, `async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>`, `async fn assign_window_to_active_workspace(&self, window_id: WindowIdentifier) -> Result<(), WorkspaceManagerError>`, `async fn assign_window_to_specific_workspace(&self, workspace_id: WorkspaceId, window_id: WindowIdentifier) -> Result<(), WorkspaceManagerError>`, `async fn remove_window_from_its_workspace(&self, window_id: &WindowIdentifier) -> Result<bool, WorkspaceManagerError>`, `async fn move_window_to_specific_workspace(&self, window_id: &WindowIdentifier, target_workspace_id: WorkspaceId) -> Result<(), WorkspaceManagerError>`, `async fn rename_workspace(&self, id: WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>`, `async fn set_workspace_layout(&self, id: WorkspaceId, layout: WorkspaceLayoutType) -> Result<(), WorkspaceManagerError>`, `async fn set_workspace_icon(&self, id: WorkspaceId, icon_name: Option<String>) -> Result<(), WorkspaceManagerError>`, `async fn set_workspace_accent_color(&self, id: WorkspaceId, accent_color_hex: Option<String>) -> Result<(), WorkspaceManagerError>`, `async fn save_configuration(&self) -> Result<(), WorkspaceManagerError>`, `fn subscribe_to_workspace_events(&self) -> tokio::sync::broadcast::Receiver<WorkspaceEvent>`, `async fn reorder_workspace(&self, workspace_id: WorkspaceId, new_index: usize) -> Result<(), WorkspaceManagerError>`.
- **Implementierung `DefaultWorkspaceManager`**: Hält einen `Arc<tokio::sync::Mutex<WorkspaceManagerInternalState>>`.
    
    - `WorkspaceManagerInternalState`: Kapselt den internen Zustand.
        - Felder: `workspaces: HashMap<WorkspaceId, Workspace>`, `active_workspace_id: Option<WorkspaceId>`, `ordered_workspace_ids: Vec<WorkspaceId>`, `next_workspace_number: u32` (für Standardnamen neuer Workspaces), `config_provider: Arc<dyn WorkspaceConfigProvider>`, `event_publisher: tokio::sync::broadcast::Sender<WorkspaceEvent>`, `ensure_unique_window_assignment: bool`.
        - Die Option `ensure_unique_window_assignment` steuert, ob ein Fenster nur einem Arbeitsbereich gleichzeitig zugewiesen sein darf. Dies ist eine wichtige Policy-Entscheidung.
- **Events (`domain::workspaces::manager::events`)**:
    
    - `WorkspaceEvent`: Enum, das alle relevanten Änderungen an Arbeitsbereichen und Fensterzuweisungen repräsentiert.
        - Varianten: `WorkspaceCreated { workspace: Workspace }`, `WorkspaceDeleted { workspace_id: WorkspaceId, windows_moved_to: Option<WorkspaceId> }`, `ActiveWorkspaceChanged { old_id: Option<WorkspaceId>, new_id: WorkspaceId }`, `WorkspaceRenamed { workspace_id: WorkspaceId, data: WorkspaceRenamedData }`, `WorkspaceLayoutChanged { workspace_id: WorkspaceId, data: WorkspaceLayoutChangedData }`, `WindowAddedToWorkspace { workspace_id: WorkspaceId, data: WindowAddedToWorkspaceData }`, `WindowRemovedFromWorkspace { workspace_id: WorkspaceId, data: WindowRemovedFromWorkspaceData }`, `WorkspaceOrderChanged { ordered_ids: Vec<WorkspaceId> }`, `WorkspacesReloaded { new_workspaces: Vec<Workspace>, new_active_id: Option<WorkspaceId> }`, `WorkspacePersistentIdChanged { workspace_id: WorkspaceId, data: WorkspacePersistentIdChangedData }`, `WorkspaceIconChanged { workspace_id: WorkspaceId, data: WorkspaceIconChangedData }`, `WorkspaceAccentChanged { workspace_id: WorkspaceId, data: WorkspaceAccentChangedData }`.
- **Fehlerbehandlung (`domain::workspaces::manager::errors`)**:
    
    - `WorkspaceManagerError`: Enum für Fehler auf der Manager-Ebene.
        - Varianten: `WorkspaceNotFound { id: WorkspaceId }`, `CannotDeleteLastWorkspace`, `DeleteRequiresFallbackForWindows { num_windows: usize }`, `FallbackWorkspaceNotFound { id: WorkspaceId }`, `CoreError(#[from] WorkspaceCoreError)`, `AssignmentError(#[from] WindowAssignmentError)`, `ConfigError(#[from] WorkspaceConfigError)`, `SetActiveWorkspaceNotFound { id: WorkspaceId }`, `NoActiveWorkspace`, `DuplicatePersistentId { id: String }`, `Internal { message: String }`.

### 1.3. Modul: `domain::user_centric_services`

Dieses Modul bündelt Domänenlogik für Dienste, die direkt auf Benutzerinteraktionen und -bedürfnisse ausgerichtet sind, insbesondere KI-Interaktionen und Benachrichtigungen.

Referenzierte Dokumente: B3 Domänenschicht.md, B4 Domänenschicht.md

#### 1.3.1. Untermodul: `domain::user_centric_services::ai_interaction`

Verantwortlich für die Logik rund um KI-gestützte Interaktionen, Einwilligungsmanagement und die Verwaltung von KI-Modellprofilen.

- **Datenstrukturen (`domain::user_centric_services::ai_interaction::types`)**:
    
    - `AIDataCategory`: Enum zur Kategorisierung von Daten, die für KI-Interaktionen verwendet werden (z.B. `UserProfile`, `ApplicationUsage`, `FileSystemRead`).
    - `AIConsentStatus`: Enum für den Einwilligungsstatus (z.B. `Granted`, `Denied`, `PendingUserAction`). Die Variante `NotRequired` ist nützlich für Interaktionen, die keine explizite Einwilligung erfordern.
    - `AttachmentData`: Struktur für Anhänge an KI-Interaktionen (z.B. Dateien, Textauszüge). Die Felder `source_uri`, `content_base64`, `text_content` bieten Flexibilität für verschiedene Anhangstypen.
    - `InteractionParticipant`: Enum zur Kennzeichnung der Teilnehmer einer Interaktion (`User`, `Assistant`, `System`).
    - `InteractionHistoryEntry`: Repräsentiert einen einzelnen Eintrag im Interaktionsverlauf.
    - `AIInteractionContext`: Die zentrale Entität, die den gesamten Kontext einer KI-Interaktion kapselt, inklusive Verlauf, Anhängen und Einwilligungsstatus.
    - `AIConsentScope`: Enum zur Definition des Gültigkeitsbereichs einer Einwilligung (z.B. `SessionOnly`, `PersistentUntilRevoked`).
    - `AIConsent`: Struktur zur Speicherung detaillierter Einwilligungsinformationen.
    - `AIModelCapability`: Enum zur Beschreibung der Fähigkeiten eines KI-Modells (z.B. `TextGeneration`, `ImageAnalysis`).
    - `AIModelProfile`: Struktur zur Definition der Eigenschaften und Fähigkeiten eines KI-Modells. Das Feld `sort_order` ermöglicht eine benutzerdefinierte Sortierung der Modelle in der UI.
- **Fehlerbehandlung (`domain::user_centric_services::ai_interaction::errors`)**:
    
    - `AIInteractionError`: Enum für Fehler im KI-Interaktionsmodul (z.B. `ConsentCheckFailed`, `ApiKeyNotFoundInSecrets`, `ModelEndpointUnreachable`, `NoDefaultModelConfigured`, `CoreConfigError(#[from] novade_core::errors::CoreError)`).
- **Persistenz-Interfaces (`domain::user_centric_services::ai_interaction::persistence_iface`)**:
    
    - Traits `AIConsentProvider` und `AIModelProfileProvider`: Definieren Schnittstellen für das Laden und Speichern von Einwilligungen und Modellprofilen, was eine Entkopplung von der konkreten Speicherimplementierung ermöglicht.
- **API (`domain::user_centric_services::ai_interaction::service::AIInteractionLogicService`)**:
    
    - Trait mit Methoden wie `initiate_interaction`, `get_interaction_context`, `provide_consent`, `load_model_profiles`.
- **Implementierung `DefaultAIInteractionLogicService`**: Konkrete Implementierung des `AIInteractionLogicService`.
    
- **Events (`domain::user_centric_services::ai_interaction::events::AIInteractionEventEnum`)**:
    
    - Events wie `AIInteractionInitiatedEvent`, `AIConsentUpdatedEvent`, `AIContextUpdatedEvent`, `AIModelProfilesReloadedEvent` zur Benachrichtigung anderer Systemteile über relevante Änderungen.

#### 1.3.2. Untermodul: `domain::user_centric_services::notifications_core`

Verantwortlich für die Kernlogik der Benachrichtigungsverwaltung, inklusive Erstellung, Speicherung, Filterung und Interaktion mit Benachrichtigungen.

- **Datenstrukturen (`domain::user_centric_services::notifications_core::types`)**:
    
    - `NotificationId`: Typalias für `uuid::Uuid`.
    - `NotificationUrgency`: Enum für die Dringlichkeit (`Low`, `Normal`, `Critical`).
    - `NotificationActionType`: Enum für Aktionstypen (`Callback`, `OpenLink`).
    - `NotificationAction`: Struktur für Aktionen, die einer Benachrichtigung zugeordnet sind.
    - `Notification`: Die Hauptentität, die eine Benachrichtigung repräsentiert. Enthält Felder wie `id`, `application_name`, `summary`, `body`, `actions`, `urgency`, `timestamp`, `is_read`, `is_dismissed`, `transient`, `category`, `hints`, `timeout_ms`.
    - `NotificationInput`: Eine Struktur zum Erstellen neuer Benachrichtigungen, ohne laufzeitgenerierte Felder wie `id` oder `timestamp`. Dies vereinfacht die API zum Posten von Benachrichtigungen.
    - `NotificationFilterCriteria`: Ein Enum für komplexe Filterkriterien, das logische UND/ODER/NICHT-Operationen sowie Filter auf Zeitbereiche unterstützt. Dies ermöglicht mächtige Abfragen des Benachrichtigungsverlaufs.
    - `NotificationSortOrder`: Enum für Sortierkriterien.
    - `NotificationStats`: Struktur zur Bereitstellung von Statistiken (z.B. `num_active`, `num_unread`), nützlich für UI-Anzeigen.
    - `DismissReason`: Enum zur Angabe des Grundes für das Schließen einer Benachrichtigung (z.B. `User`, `Timeout`), nützlich für Analysen und intelligente Benachrichtigungslogik.
- **Fehlerbehandlung (`domain::user_centric_services::notifications_core::errors`)**:
    
    - `NotificationError`: Enum für Fehler im Benachrichtigungssystem (z.B. `NotFound`, `InvalidData`, `HistoryFull`).
- **API (`domain::user_centric_services::notifications_core::service::NotificationService`)**:
    
    - Trait mit Methoden wie `post_notification(input: NotificationInput)`, `get_notification`, `mark_as_read`, `dismiss_notification`, `get_active_notifications`, `get_notification_history`, `set_do_not_disturb`, `invoke_action`, `get_stats`.
- **Implementierung `DefaultNotificationService`**: Konkrete Implementierung des `NotificationService`.
    
- **Events (`domain::user_centric_services::notifications_core::events::NotificationEventEnum`)**:
    
    - Events wie `NotificationPostedEvent`, `NotificationDismissedEvent`, `NotificationReadEvent`, `DoNotDisturbModeChangedEvent` zur Benachrichtigung der UI und anderer Systemteile.

### 1.4. Modul: `domain::notifications_rules`

Ermöglicht eine regelbasierte Verarbeitung von eingehenden Benachrichtigungen, um deren Verhalten (z.B. Unterdrückung, Änderung der Dringlichkeit) basierend auf benutzerdefinierten Kriterien anzupassen.

Referenzierte Dokumente: B4 Domänenschicht.md

#### 1.4.1. Datenstrukturen (`domain::notifications_rules::types`)

- `RuleConditionValue`: Enum für die Werte in Regelbedingungen (z.B. `String(String)`, `Urgency(NotificationUrgency)`).
- `RuleConditionOperator`: Enum für Vergleichsoperatoren (z.B. `Is`, `Contains`, `MatchesRegex`).
- `RuleConditionField`: Enum zur Spezifizierung des Benachrichtigungsfeldes, auf das sich eine Bedingung bezieht (z.B. `ApplicationName`, `Summary`, `HintExists(String)`). Die Varianten `HintExists` und `HintValue` erlauben flexible Bedingungen basierend auf den `hints` einer Benachrichtigung.
- `SimpleRuleCondition`: Eine einfache Bedingung bestehend aus Feld, Operator und Wert.
- `RuleCondition`: Ein Enum, das komplexe, verschachtelte Bedingungen durch logische Operatoren (`And`, `Or`, `Not`) ermöglicht. Die Variante `SettingIsTrue(SettingPath)` erlaubt es, Regeln basierend auf globalen Einstellungen zu definieren, was eine starke Integration mit dem `GlobalSettingsService` ermöglicht.
- `RuleAction`: Enum für Aktionen, die ausgeführt werden, wenn eine Regel zutrifft (z.B. `SuppressNotification`, `SetUrgency`, `PlaySound`). Die Aktion `StopProcessingFurtherRules` bietet eine explizite Kontrolle über den Regelauswertungsfluss.
- `NotificationRule`: Repräsentiert eine einzelne Regel mit ID, Name, Bedingung, Aktionen, Aktivierungsstatus und Priorität. Die `Default`-Implementierung kann für das einfache Erstellen neuer Regeln in einer UI nützlich sein.
- `NotificationRuleSet`: Typalias für `Vec<NotificationRule>`. Die Reihenfolge in diesem Vektor ist relevant für die Regelauswertung, da Regeln typischerweise basierend auf ihrer Priorität und/oder Reihenfolge verarbeitet werden.

#### 1.4.2. Fehlerbehandlung (`domain::notifications_rules::errors`)

- `NotificationRulesError`: Enum für Fehler im Regelmodul (z.B. `InvalidRuleDefinition`, `ConditionEvaluationError`, `SettingsAccessError(#[from] GlobalSettingsError)`, `InvalidRegex`).

#### 1.4.3. API (`domain::notifications_rules::engine::NotificationRulesEngine`)

- Trait `NotificationRulesEngine`:
    - Methoden: `async fn reload_rules(&mut self)`, `async fn process_notification(&self, notification: Notification) -> Result<RuleProcessingResult, NotificationRulesError>`, `async fn get_rules(&self) -> Result<NotificationRuleSet, NotificationRulesError>`, `async fn update_rules(&mut self, new_rules: NotificationRuleSet) -> Result<(), NotificationRulesError>`.
- Implementierung `DefaultNotificationRulesEngine`: Hält einen `Arc<tokio::sync::RwLock<NotificationRuleSet>>`, einen `Arc<dyn NotificationRulesProvider>` und einen `Arc<dyn GlobalSettingsService>`. Die Verwendung von `RwLock` ermöglicht konkurrierenden Lesezugriff auf die Regeln, während Schreibzugriffe exklusiv sind.
- Enum `RuleProcessingResult`: Gibt das Ergebnis der Regelverarbeitung an.
    - Varianten: `Allow(Notification)` (die Benachrichtigung darf angezeigt werden, möglicherweise modifiziert), `Suppress { rule_id: Uuid }` (die Benachrichtigung soll unterdrückt werden).

#### 1.4.4. Persistenz (`domain::notifications_rules::persistence_iface`)

- Trait `NotificationRulesProvider`: Definiert die Schnittstelle zum Laden und Speichern von Regelwerken.
    - Methoden: `async fn load_rules(&self) -> Result<NotificationRuleSet, NotificationRulesError>`, `async fn save_rules(&self, rules: &NotificationRuleSet) -> Result<(), NotificationRulesError>`.
- Implementierung `FilesystemNotificationRulesProvider`: Speichert Regeln im Dateisystem, typischerweise als JSON.

### 1.5. Modul: `domain::global_settings_and_state_management`

Verwaltet globale Desktop-Einstellungen und deren Persistenz. Dieses Modul ist zentral für die Konfiguration vieler Aspekte der Desktop-Umgebung.

Referenzierte Dokumente: B3 Domänenschicht.md, B4 Domänenschicht.md

#### 1.5.1. Datenstrukturen (`domain::global_settings_and_state_management::types`)

- `GlobalDesktopSettings`: Die Hauptstruktur, die alle globalen Einstellungen enthält.
    - Felder: `appearance: AppearanceSettings`, `workspace_config: WorkspaceSettings`, `input_behavior: InputBehaviorSettings`, `power_management_policy: PowerManagementPolicySettings`, `default_applications: DefaultApplicationsSettings`.
    - Alle Felder sind mit `#[serde(default)]` annotiert. Dies ist eine wichtige Designentscheidung, die die Robustheit des Systems gegenüber unvollständigen oder fehlerhaften Konfigurationsdateien erhöht, da für fehlende Abschnitte Standardwerte verwendet werden.
- Detaillierte Unterstrukturen (z.B. `AppearanceSettings`, `FontSettings`) und Enums (z.B. `ColorScheme`, `MouseAccelerationProfile`) sind wie in den referenzierten Dokumenten B3 und B4 definiert.
- **Validierung**: Jede Einstellungs-Unterstruktur implementiert eine Methode `fn validate(&self) -> Result<(), String>`. Die `GlobalDesktopSettings`-Struktur verfügt über eine Methode `validate_recursive()`, die die Validierung für alle Unterstrukturen aufruft und so die Datenintegrität sicherstellt.

#### 1.5.2. Pfade (`domain::global_settings_and_state_management::paths`)

- `SettingPath`: Ein hierarchischer Enum zur typsicheren Adressierung einzelner Einstellungen innerhalb der `GlobalDesktopSettings`-Struktur.
    - Implementiert `Display` (zur Konvertierung in einen lesbaren String-Pfad, z.B. für UI oder Logging) und `TryFrom<&str>` (zum Parsen eines String-Pfades zurück in den Enum).
    - Die Verwendung eines typsicheren Pfad-Enums anstelle von reinen Strings reduziert das Risiko von Tippfehlern und Laufzeitfehlern beim Zugriff auf Einstellungen.

#### 1.5.3. Fehlerbehandlung (`domain::global_settings_and_state_management::errors`)

- `GlobalSettingsError`: Enum für Fehler im Einstellungsmodul.
    - Varianten: `PathNotFound { path: SettingPath }`, `InvalidValueType { path: SettingPath, expected: String, found: String }`, `ValidationError { path: SettingPath, reason: String }`, `SerializationError { source: serde_json::Error }`, `DeserializationError { source: serde_json::Error }`, `PersistenceError(#[from] GlobalSettingsPersistenceError)`.
    - Die Verwendung von `SettingPath` in den Fehlervarianten ermöglicht eine präzise Fehlerlokalisierung.

#### 1.5.4. Persistenz-Interface (`domain::global_settings_and_state_management::persistence_iface`)

- Trait `SettingsPersistenceProvider`: Definiert die Schnittstelle zum Laden und Speichern der globalen Einstellungen.
    - Methoden: `async fn load_global_settings(&self) -> Result<GlobalDesktopSettings, GlobalSettingsPersistenceError>`, `async fn save_global_settings(&self, settings: &GlobalDesktopSettings) -> Result<(), GlobalSettingsPersistenceError>`.
    - Anmerkung: Die Verwendung eines spezifischen `GlobalSettingsPersistenceError` ermöglicht eine saubere `#[from]`-Konvertierung in `GlobalSettingsError::PersistenceError`.
- Implementierung `FilesystemSettingsProvider`: Nutzt `novade_core::config::ConfigServiceAsync` für Dateioperationen.

#### 1.5.5. API (`domain::global_settings_and_state_management::service::GlobalSettingsService`)

- Trait `GlobalSettingsService`: Die Hauptschnittstelle für den Zugriff auf und die Manipulation von globalen Einstellungen.
    - Methoden: `async fn load_settings(&self) -> Result<(), GlobalSettingsError>`, `async fn save_settings(&self) -> Result<(), GlobalSettingsError>`, `fn get_current_settings(&self) -> Arc<GlobalDesktopSettings>`, `async fn update_setting(&self, path: SettingPath, value: serde_json::Value) -> Result<(), GlobalSettingsError>`, `fn get_setting(&self, path: &SettingPath) -> Result<serde_json::Value, GlobalSettingsError>`, `async fn reset_to_defaults(&self) -> Result<(), GlobalSettingsError>`, `fn subscribe_to_setting_changes(&self) -> tokio::sync::broadcast::Receiver<SettingChangedEvent>`.
- Implementierung `DefaultGlobalSettingsService`: Hält die Einstellungen in einem `Arc<tokio::sync::RwLock<GlobalDesktopSettings>>`, einen `Arc<dyn SettingsPersistenceProvider>` und einen `tokio::sync::broadcast::Sender` für Events.

#### 1.5.6. Events (`domain::global_settings_and_state_management::events`)

- `SettingChangedEvent { path: SettingPath, new_value: serde_json::Value }`: Wird publiziert, wenn eine Einstellung geändert wird.
- `SettingsLoadedEvent { settings: Arc<GlobalDesktopSettings> }`: Wird publiziert, nachdem die Einstellungen erfolgreich geladen wurden.
- `SettingsSavedEvent`: Wird publiziert, nachdem die Einstellungen erfolgreich gespeichert wurden.

### 1.6. Modul: `domain::window_management_policy`

Dieses Modul definiert High-Level-Regeln und Richtlinien für die Fensterplatzierung, Tiling-Verhalten, Snapping und Fokusmanagement. Es entkoppelt die Policy-Entscheidungen von den Mechanismen der Fensterverwaltung, die im Compositor implementiert sind.

Referenzierte Dokumente: B3 Domänenschicht.md

#### 1.6.1. Datenstrukturen (`domain::window_management_policy::types`)

- `TilingMode`: Enum für verschiedene Tiling-Strategien (z.B. `Manual`, `Columns`, `Rows`, `Spiral`, `MaximizedFocused`).
- `GapSettings`: Definiert Abstände um Bildschirme und zwischen Fenstern.
    - Felder: `screen_outer_horizontal: u16`, `screen_outer_vertical: u16`, `window_inner: u16`.
- `WindowSnappingPolicy`: Konfiguriert das Snapping-Verhalten von Fenstern.
    - Felder: `snap_to_screen_edges: bool`, `snap_to_other_windows: bool`, `snap_to_workspace_gaps: bool`, `snap_distance_px: u16`.
- `WindowGroupingPolicy`: Einstellungen für manuelles Gruppieren von Fenstern.
    - Felder: `enable_manual_grouping: bool`.
- `NewWindowPlacementStrategy`: Enum für die Platzierungsstrategie neuer Fenster (z.B. `Smart`, `Center`, `Cascade`, `UnderMouse`).
- `FocusStealingPreventionLevel`: Enum zur Steuerung, wie aggressiv das System "Focus Stealing" verhindert (`None`, `Moderate`, `Strict`).
- `FocusPolicy`: Umfassende Einstellungen zum Fokusverhalten.
    - Felder: `focus_follows_mouse: bool`, `click_to_focus: bool`, `focus_new_windows_on_creation: bool`, `focus_new_windows_on_workspace_switch: bool`, `focus_stealing_prevention: FocusStealingPreventionLevel`.
- `WindowPolicyOverrides`: Optionale, fensterspezifische Überschreibungen globaler Policies (z.B. `preferred_tiling_mode`, `is_always_floating`).
- `WorkspaceWindowLayout`: Beschreibt das berechnete Layout aller Fenster auf einem Arbeitsbereich.
    - Felder: `window_geometries: HashMap<WindowIdentifier, novade_core::types::RectInt>`, `occupied_area: Option<novade_core::types::RectInt>`, `tiling_mode_applied: TilingMode`.
- `WindowLayoutInfo`: Enthält Informationen von einem Fenster, die für Layout-Berechnungen relevant sind.
    - Felder: `id: WindowIdentifier`, `requested_min_size: Option<novade_core::types::Size<u32>>`, `is_fullscreen_requested: bool`, `is_maximized_requested: bool`.

#### 1.6.2. Fehlerbehandlung (`domain::window_management_policy::errors`)

- `WindowPolicyError`: Enum für Fehler im Policy-Modul.
    - Varianten: `LayoutCalculationError { details: String }`, `InvalidPolicyConfiguration { setting_path: Option<String>, reason: String }`, `WindowNotFoundForPolicy { window_id: WindowIdentifier }`.

#### 1.6.3. API (`domain::window_management_policy::service::WindowManagementPolicyService`)

- Trait `WindowManagementPolicyService`: Definiert die Schnittstelle für Policy-Abfragen und Layout-Berechnungen.
    - Methoden: `async fn calculate_workspace_layout(&self, workspace_id: WorkspaceId, windows_info: Vec<WindowLayoutInfo>, available_area: novade_core::types::RectInt) -> Result<WorkspaceWindowLayout, WindowPolicyError>`, `async fn get_initial_window_geometry(&self, app_id: Option<ApplicationId>, title: Option<String>, requested_size: Option<novade_core::types::Size<u32>>, output_area: novade_core::types::RectInt) -> Result<novade_core::types::RectInt, WindowPolicyError>`, `async fn calculate_snap_target(&self, window_id: WindowIdentifier, current_geometry: novade_core::types::RectInt, workspace_layout: &WorkspaceWindowLayout, output_area: novade_core::types::RectInt) -> Option<novade_core::types::RectInt>`, `async fn get_effective_tiling_mode_for_workspace(&self, workspace_id: WorkspaceId) -> Result<TilingMode, WindowPolicyError>`, `async fn get_effective_gap_settings_for_workspace(&self, workspace_id: WorkspaceId) -> Result<GapSettings, WindowPolicyError>`, `async fn get_effective_snapping_policy(&self) -> Result<WindowSnappingPolicy, WindowPolicyError>`, `async fn get_effective_focus_policy(&self) -> Result<FocusPolicy, WindowPolicyError>`, `async fn get_effective_new_window_placement_strategy(&self) -> Result<NewWindowPlacementStrategy, WindowPolicyError>`.
- Implementierung `DefaultWindowManagementPolicyService`: Hält einen `Arc<dyn GlobalSettingsService>`, um auf die globalen Policy-Einstellungen zuzugreifen. Diese Abhängigkeit ermöglicht dynamische Policy-Änderungen. Die Komplexität der `calculate_workspace_layout`-Funktion erfordert sorgfältige Implementierung und gründliche Tests, insbesondere im Hinblick auf verschiedene Tiling-Modi und Fenster-Overrides.

### 1.7. Modul: `domain::common_events`

Definiert Domänen-übergreifende Events, die von verschiedenen Modulen publiziert oder abonniert werden können.

Referenzierte Dokumente: B3 Domänenschicht.md

#### 1.7.1. Datenstrukturen (`domain::common_events::events`)

- `UserActivityType`: Enum zur Klassifizierung von Benutzeraktivitäten.
    - Varianten: `MouseMoved`, `MouseClicked`, `MouseWheelScrolled`, `KeyPressed`, `TouchInteraction`, `WorkspaceSwitched`, `ApplicationFocused`, `WindowOpened`, `WindowClosed`. Die Granularität dieser Typen ist für die Implementierung von Features wie Inaktivitäts-Timer oder Nutzungsstatistiken relevant.
- `UserActivityDetectedEvent`: Event, das bei Erkennung einer Benutzeraktivität ausgelöst wird.
    - Felder: `event_id: Uuid`, `timestamp: DateTime<Utc>`, `activity_type: UserActivityType`, `current_session_state: UserSessionState`, `active_application_id: Option<ApplicationId>`, `active_workspace_id: Option<WorkspaceId>`.
- `ShutdownReason`: Enum, das den Grund für ein Herunterfahren des Systems angibt.
    - Varianten: `UserRequest`, `PowerButtonPress`, `LowBattery`, `SystemUpdate`, `ApplicationRequest`, `OsError`, `Unknown`.
- `SystemShutdownInitiatedEvent`: Event, das ausgelöst wird, wenn ein System-Shutdown initiiert wird.
    - Felder: `event_id: Uuid`, `timestamp: DateTime<Utc>`, `reason: ShutdownReason`, `is_reboot: bool`, `delay_seconds: Option<u32>`, `message: Option<String>`.

### 1.8. Modul: `domain::shared_types`

Stellt wiederverwendbare, domänenspezifische Typen bereit, die in mehreren Domänenmodulen verwendet werden, um Konsistenz zu fördern und Duplizierung zu vermeiden.

Referenzierte Dokumente: B3 Domänenschicht.md

#### 1.8.1. Datenstrukturen (`domain::shared_types::types`)

- `ApplicationId`: Wrapper für `String` zur eindeutigen Identifizierung von Anwendungen.
- `UserSessionState`: Enum zur Repräsentation des Zustands der Benutzersitzung.
    - Varianten: `Active`, `Locked`, `Idle`.
- `ResourceIdentifier`: Eine generische Struktur zur Identifizierung von Ressourcen innerhalb des Systems.
    - Felder: `r#type: String` (z.B. "window", "workspace"), `id: String`, `label: Option<String>`.

Die klare Definition dieser gemeinsam genutzten Typen ist grundlegend für die Interoperabilität und das korrekte Zusammenspiel der verschiedenen Domänenmodule.

**Tabelle 1: Domain Layer Modules Overview**

|   |   |   |
|---|---|---|
|**Modul-Pfad**|**Kernverantwortlichkeit**|**Wichtige Services/Entitäten**|
|`domain::theming`|Verwaltet alle Aspekte des visuellen Stylings, der Theme-Anwendung und der Token-Auflösung.|`ThemingEngine`, `ThemeDefinition`, `AppliedThemeState`, `TokenIdentifier`, `TokenValue`|
|`domain::workspaces`|Logik und Verwaltung von Arbeitsbereichen ("Spaces"), inklusive Fensterzuweisung und Persistenz.|`WorkspaceManagerService`, `Workspace`, `WorkspaceId`, `WindowIdentifier`|
|`domain::user_centric_services::ai_interaction`|Handhabt KI-gestützte Interaktionen, Einwilligungsmanagement und KI-Modellprofile.|`AIInteractionLogicService`, `AIInteractionContext`, `AIConsent`, `AIModelProfile`|
|`domain::user_centric_services::notifications_core`|Kernlogik für das Erstellen, Speichern, Filtern und Interagieren mit Benachrichtigungen.|`NotificationService`, `Notification`, `NotificationInput`, `NotificationFilterCriteria`|
|`domain::notifications_rules`|Ermöglicht regelbasierte Verarbeitung und Modifikation von Benachrichtigungen.|`NotificationRulesEngine`, `NotificationRule`, `RuleCondition`, `RuleAction`|
|`domain::global_settings_and_state_management`|Verwaltung globaler Desktop-Einstellungen, deren Persistenz und Validierung.|`GlobalSettingsService`, `GlobalDesktopSettings`, `SettingPath`|
|`domain::window_management_policy`|Definiert High-Level-Regeln für Fensterplatzierung, Tiling, Snapping und Fokus.|`WindowManagementPolicyService`, `TilingMode`, `FocusPolicy`, `WorkspaceWindowLayout`|
|`domain::common_events`|Definiert Domänen-übergreifende Events wie Benutzeraktivität oder System-Shutdown.|`UserActivityDetectedEvent`, `SystemShutdownInitiatedEvent`|
|`domain::shared_types`|Stellt wiederverwendbare domänenspezifische Typen bereit.|`ApplicationId`, `UserSessionState`, `ResourceIdentifier`|

## 2. Wayland/Smithay-Integration (Systemschicht-Perspektive)

Dieser Abschnitt beschreibt, wie die Systemschicht, insbesondere der Wayland-Compositor, der mit dem Smithay-Toolkit implementiert wird, mit Wayland-Protokollen umgeht und mit der oben definierten Domänenschicht interagiert. Die korrekte Integration ist entscheidend für die Funktionalität und Stabilität des Desktops.

Referenzierte Dokumente: C1 System Implementierungsplan.md bis C4 Systemschicht.md, 3. System-Details.md.

### 2.1. Kern-Compositor-Struktur (`system::compositor::core::state::DesktopState`)

`DesktopState` ist die zentrale Zustandsstruktur des Compositors. Sie implementiert die verschiedenen Handler-Traits von Smithay und hält sowohl Smithay-spezifische Zustände als auch Handles zu den Domänendiensten. Diese Struktur ist das Herzstück der Compositor-Implementierung und agiert als zentraler Dispatcher und State-Manager für Wayland-bezogene Operationen.1 Die Verwendung einer zentralen, veränderbaren Zustandsstruktur, die an Callbacks übergeben wird, ist ein von Smithay empfohlenes Muster, das durch `calloop` ermöglicht wird und die Notwendigkeit komplexer Synchronisationsmechanismen reduziert.2

- **Wichtige Felder** (Auswahl, basierend auf typischer Smithay-Architektur):
    - `display_handle: DisplayHandle`: Das zentrale Handle zur Wayland-Display-Instanz, notwendig für die Erstellung von Globals und die Client-Kommunikation.5
    - `loop_handle: LoopHandle<'static, Self>`: Handle zum `calloop`-Event-Loop, der den Compositor antreibt.
    - `clock: Clock<u64>`: Eine Zeitquelle für den Compositor.
    - `compositor_state: CompositorState`: Verwaltet `wl_compositor`- und `wl_subcompositor`-bezogene Zustände.6
    - `shm_state: ShmState`: Verwaltet `wl_shm`-bezogene Zustände für Shared Memory Buffer.8
    - `xdg_shell_state: XdgShellState`: Verwaltet Zustände für das `xdg_shell`-Protokoll (Fenster, Popups).9
    - `output_manager_state: OutputManagerState`: Verwaltet `wl_output`- und `zxdg_output_v1`-Zustände.16
    - `seat_state: SeatState<Self>`: Verwaltet `wl_seat`-Zustände und Eingabefähigkeiten.9
    - `seat: Seat<Self>`: Die konkrete Seat-Instanz.
    - `data_device_state: DataDeviceState`: Verwaltet Zustände für `wl_data_device` (Clipboard, Drag & Drop).8
    - `space: Space<WindowElement>`: Smithays 2D-Raum zur Verwaltung von Fenstern, deren Stapelreihenfolge und zum Rendern.1 `WindowElement` ist dabei ein Enum, das verschiedene Fenstertypen wie native Wayland-Fenster (`Window`), X11-Fenster (`X11Surface`) und Layer-Shell-Oberflächen (`LayerSurface`) kapselt.
    - `windows: HashMap<DomainWindowIdentifier, Arc<WindowElement>>`: Ein Mapping von den abstrakten `DomainWindowIdentifier` der Domänenschicht zu den konkreten `WindowElement`-Instanzen des Compositors. Dies ist eine entscheidende Brücke zwischen Domänen- und Systemschicht.
    - Domänen-Service-Handles: `Arc<dyn WindowManagementPolicyService>`, `Arc<dyn WorkspaceManagerService>`, `Arc<dyn GlobalSettingsService>`. Diese `Arc`-Handles ermöglichen den Zugriff auf die Domänenlogik aus den Wayland-Handlern heraus. Die Thread-Sicherheit dieser Interaktion muss gewährleistet sein, insbesondere da Domänendienste asynchron sind, während Wayland-Handler typischerweise synchron im `calloop`-Thread ausgeführt werden.33
    - `event_bridge: Arc<SystemEventBridge>`: Ein Mechanismus zur internen Event-Kommunikation innerhalb der Systemschicht und potenziell zur Weiterleitung von Systemereignissen an die Domänenschicht.

Obwohl `DesktopState` viele Verantwortlichkeiten bündelt, was Bedenken hinsichtlich eines "God Object" 29 aufwerfen könnte, ist dieses Muster im Kontext von Smithay und `calloop` üblich und wird durch die Verwendung von Delegate-Makros handhabbar gemacht. Diese Makros leiten die Protokollbehandlung an spezifische Zustands-Subobjekte und deren Handler weiter, anstatt die gesamte Logik in `DesktopState` selbst zu implementieren.1

### 2.2. Wayland-Protokoll-Implementierungen

Die `DesktopState`-Struktur implementiert verschiedene Handler-Traits von Smithay, um die Wayland-Protokolle zu bedienen.

- **`wl_compositor`, `wl_subcompositor`**:
    
    - Gehandhabt durch `CompositorState` und die Implementierung von `CompositorHandler` in `DesktopState`.4
    - `CompositorHandler::commit(surface: &WlSurface)`: Diese Methode wird bei jedem `wl_surface.commit` eines Clients aufgerufen. Sie ist verantwortlich für:
        - Aktualisierung von `SurfaceData` (Puffer, Schadensregionen (Damage)).
        - Benachrichtigung von `domain::window_management_policy` oder `domain::workspaces` bei relevanten Änderungen (z.B. könnte eine neue Puffergröße das Layout beeinflussen). Diese Interaktion ist ein Beispiel für die Kommunikation von der Systemschicht zur Domänenschicht für Policy-Entscheidungen.
        - Markierung der Oberfläche für eine Neuzeichnung durch den Renderer.
- **`wl_shm`**:
    
    - Gehandhabt durch `ShmState` und die Implementierung von `ShmHandler` in `DesktopState`.4
    - `BufferHandler::buffer_destroyed(buffer: &WlBuffer)`: Informiert den Renderer, dass die mit diesem Puffer verbundenen Ressourcen freigegeben werden können. Dies ist wichtig für das Speichermanagement.
- **`xdg_shell` (`xdg_wm_base`, `xdg_surface`, `xdg_toplevel`, `xdg_popup`)**:
    
    - Gehandhabt durch `XdgShellState` und die Implementierung von `XdgShellHandler` in `DesktopState`.4
    - `XdgShellHandler::new_toplevel(surface: ToplevelSurface)`: Ein kritischer Integrationspunkt.
        1. Erstellt ein `WindowElement::Wayland(Window::new(ToplevelKind::Xdg(surface)))`. Das `Window` hier ist eine Smithay-Abstraktion.29
        2. Generiert einen `DomainWindowIdentifier` für die Domänenschicht.
        3. Ruft `domain::window_management_policy::get_initial_window_geometry()` auf, um die initiale Position und Größe des Fensters gemäß den globalen und anwendungsspezifischen Richtlinien zu bestimmen.
        4. Ruft `domain::workspaces::assign_window_to_active_workspace()` auf, um das neue Fenster dem aktuell aktiven Arbeitsbereich zuzuordnen.
        5. Konfiguriert und mappt das Fenster im `Space`.
    - `XdgShellHandler::map_toplevel(surface: &ToplevelSurface)`: Informiert `domain::workspaces`, dass das Fenster nun sichtbar (gemappt) ist.
    - Client-Anfragen wie `set_title`, `set_app_id`, `set_maximized`, `move`, `resize` werden an `domain::window_management_policy` delegiert. Die Domänenschicht entscheidet über die Policy-konforme Reaktion. Das Ergebnis dieser Entscheidung (z.B. neue Geometrie, neuer Zustand) wird dann vom Compositor auf das `ToplevelSurface` angewendet, typischerweise durch Senden eines `configure`-Events an den Client (`toplevel_surface.send_configure(...)`).
    - **Domäne -> System für Layout**: `domain::window_management_policy` gibt eine `WorkspaceWindowLayout`-Struktur zurück. Ein systemseitiger Mechanismus (z.B. `system::window_mechanics::apply_workspace_layout`) iteriert über die `window_geometries` dieser Struktur und ruft `toplevel_surface.send_configure(...)` für jedes betroffene Fenster auf, um die vom Domänenmodul berechneten Größen und Positionen anzuwenden.
    - Die Synchronisation von Zuständen wie "maximiert" oder "aktiv" zwischen `XdgToplevelSurfaceData` und einer internen `ManagedWindow`-Repräsentation erfolgt über den Configure-Ack-Mechanismus.1 Der Compositor sendet einen `configure`-Event mit dem neuen Zustand, und der Client muss diesen mit `ack_configure` bestätigen, bevor der Zustand als angewendet gilt.
- **`wlr-layer-shell-unstable-v1`**:
    
    - Gehandhabt durch `WlrLayerShellState` und die Implementierung von `LayerShellHandler` in `DesktopState`.4
    - `LayerShellHandler::new_layer_surface(surface: LayerSurface,...)`: Erstellt ein `WindowElement::Layer(surface)`. Informiert die Domänenschicht (ggf. ein spezifischer Service für Shell-Elemente wie Panels oder Hintergrundbilder). Die Platzierung im `Space` erfolgt gemäß den Layer-Shell-Regeln (Layer, Anchor, Exclusive Zone). Die Interaktion mit `PopupManager` und die Handhabung von `exclusive_zone` sind hier relevant.53
- **`wl_output` / `xdg-output-unstable-v1`**:
    
    - Gehandhabt durch `OutputManagerState` und die Implementierung von `OutputHandler` in `DesktopState`.2
    - Änderungen an Outputs (Hotplug, Modusänderung) werden vom `OutputHandler` verarbeitet.
    - Diese Änderungen werden an `domain::global_settings_and_state_management` gemeldet (z.B. um Persistenz anzustoßen oder Display-Profile zu aktualisieren) und an `domain::workspaces` (da Änderungen der Output-Konfiguration das Fensterlayout beeinflussen können).
    - Die Implementierung von `zwlr_output_manager_v1` (Teil von `wlr-output-management-unstable-v1` 2) in `system::outputs::output_manager` (oder einer ähnlichen Struktur) würde die `apply`-Methode dieses Protokolls implementieren. Bei einem Aufruf von `apply` wird `Output::change_current_state()` aufgerufen. Die Domänenschicht wird über ein `OutputConfigurationChangedEvent` informiert. Die korrekte Handhabung von Serials ist hierbei entscheidend für die atomare Anwendung von Konfigurationen.61
- **`wl_seat` / Eingabeprotokolle (`wl_keyboard`, `wl_pointer`, `wl_touch`)**:
    
    - Gehandhabt durch `SeatState`, `SeatHandler`, `KeyboardHandle`, `PointerHandle`, `TouchHandle`.4
    - Eingabe-Events von `system::input` (typischerweise `libinput` 81) werden verarbeitet.
    - `SeatHandler::focus_changed(seat: &Seat<Self>, focused: Option<&WlSurface>)`: Identifiziert den `DomainWindowIdentifier` für die fokussierte Oberfläche. Informiert `domain::window_management_policy` oder `domain::workspaces` über die Fokusänderung. Dies ist ein kritischer Punkt für die Fokusverwaltung.84
    - `KeyboardHandle::input(keycode, state,...)`: Sendet `wl_keyboard.key`-Events an den fokussierten Client.
    - `PointerHandle::motion(...)`: Sendet `wl_pointer.motion`, `enter`, `leave`-Events.
    - **Domäne -> System für Fokus**: `domain::window_management_policy` kann einen Fokuswechsel anstoßen (z.B. als Reaktion auf eine Policy oder einen Benutzerbefehl). Ein systemseitiger Mechanismus (z.B. `system::window_mechanics::set_application_focus`) ruft dann `keyboard.set_focus(...)` auf, um den Fokus im Wayland-Seat zu setzen.

### 2.3. Smithay-Typen und ihre Verwendung

Die Integration mit Smithay erfordert die korrekte Verwendung und Erweiterung seiner Kern-Typen.

- **`DisplayHandle`**: Das zentrale Handle zur Wayland-Display-Instanz. Wird für die Erstellung von Globals, Client-Kommunikation etc. verwendet.5
    
- **`CompositorState`**: Verwaltet `wl_compositor` und `wl_subcompositor` Globals und Client-Zustände.6
    
- **`ShmState`**: Verwaltet das `wl_shm` Global und SHM-Puffer.
    
- **`XdgShellState`**: Verwaltet das `xdg_wm_base` Global und XDG-Oberflächen (Toplevels, Popups).9
    
- **`OutputManagerState`**: Verwaltet `wl_output` und `zxdg_output_manager_v1` Globals.16
    
- **`SeatState<DesktopState>` / `Seat<DesktopState>`**: Verwalten `wl_seat` und die Eingabefähigkeiten (Tastatur, Maus, Touch). `SeatHandler` wird von `DesktopState` implementiert.9
    
- **`KeyboardHandle`, `PointerHandle`, `TouchHandle`**: Abstraktionen für die Interaktion mit spezifischen Eingabegeräten des Seats.
    
- **`Space<WindowElement>`**: Smithays 2D-Raum zur Verwaltung von Fenstern, deren Stapelreihenfolge und zum Rendern.1 `WindowElement` muss den `smithay::desktop::Window` Trait implementieren.29
    
- **`SurfaceData`** (aus `smithay::wayland::compositor`): Wird an `WlSurface`-Instanzen angehängt und speichert Pufferinformationen, die zugewiesene Rolle, Schadensregionen (Damage) etc..7 NovaDE wird dies erweitern (z.B. über `UserDataMap` oder eine dedizierte Erweiterungsstruktur wie `SurfaceDataExt`), um anwendungsspezifische Daten wie `DomainWindowIdentifier` und Renderer-spezifische Textur-Handles zu speichern. Die Verwaltung von doppelt gepuffertem Zustand und Commit-Hooks (Pre/Post) ist ein Kernaspekt von `SurfaceData`.4
    
- **`ClientData`** (aus `smithay::reexports::wayland_server::backend`): Wird an `Client`-Instanzen angehängt und kann Client-spezifische Zustände speichern (z.B. `CompositorClientState`, `XdgWmBaseClientData`).
    
- **Änderungen an Smithay-Typen und deren Konsequenzen**:
    
    - Das direkte Ändern von Smithay-Typen ist in der Regel nicht vorgesehen, da sie Teil der Bibliothek sind. Die Erweiterung erfolgt primär durch die Verwendung von `UserDataMap` an Objekten wie `WlSurface` oder `Client` oder durch die Implementierung der von Smithay bereitgestellten Handler-Traits.
    - Wenn Smithay eine neue Version mit geänderten Typen oder Trait-Signaturen veröffentlicht, müssen die Handler-Implementierungen und die Nutzung dieser Typen in NovaDE entsprechend angepasst werden. Dies erfordert sorgfältige Migration und umfassende Tests, um die Kompatibilität und korrekte Funktionalität sicherzustellen.
    - Das `UserDataMap`-Muster ist der primäre Mechanismus, um anwendungsspezifische Daten mit Smithay-Objekten zu assoziieren, ohne die Typen selbst modifizieren zu müssen. Dies ist entscheidend für die Aufrechterhaltung einer sauberen Trennung zwischen der Bibliothekslogik und der anwendungsspezifischen Logik des Compositors.

### 2.4. Kommunikation Domäne <-> Compositor

Die Kommunikation zwischen der Domänenschicht und der Systemschicht (Compositor) ist bidirektional und erfolgt über definierte Schnittstellen, primär Events und Service-Aufrufe.

- **Compositor (System) -> Domäne**:
    
    - **Events**: Der Compositor generiert Systemereignisse, die für die Domänenschicht relevant sind. Diese werden typischerweise über einen `SystemEventBridge` oder einen ähnlichen Mechanismus an die Domänenschicht weitergeleitet.
        - Neues Fenster (z.B. in `XdgShellHandler::new_toplevel`): Sendet ein `WindowCreatedEvent { domain_id, initial_geometry, app_id, title }` an `domain::workspaces::manager` oder einen allgemeinen Event-Bus.
        - Fenster geschlossen (z.B. in `XdgShellHandler::toplevel_destroyed`): Sendet ein `WindowClosedEvent { domain_id }`.
        - Fokusänderung durch Benutzer (z.B. in `SeatHandler::focus_changed`): Sendet ein `FocusChangedEvent { new_focus_domain_id, old_focus_domain_id }`.
        - Benutzereingabe für spezielle Funktionen (z.B. Befehlspalette): Kann ein spezifisches Event senden oder direkt einen Domänendienst aufrufen.
        - Output-Änderung (z.B. in `OutputHandler`): Sendet ein `OutputConfigurationChangedEvent {... }`.
    - **Service-Aufrufe (indirekt)**: Bei bestimmten Aktionen kann der Compositor Domänenlogik benötigen, z.B. um die Standardplatzierung für ein neues Fenster zu erfragen (`domain::window_management_policy::get_initial_window_geometry`).
- **Domäne -> Compositor (System)**:
    
    - **Service-Aufrufe**: Die Domänenschicht kann Aktionen im Compositor auslösen, indem sie Methoden auf Schnittstellen aufruft, die von der Systemschicht bereitgestellt werden.
        - `domain::window_management_policy` gibt eine `WorkspaceWindowLayout`-Struktur an einen systemseitigen Mechanismus (z.B. `system::window_mechanics`).
        - `system::window_mechanics::apply_workspace_layout` ruft dann spezifische Smithay-Funktionen auf, z.B. `toplevel_surface.send_configure(...)` oder `space.map_window(...)`.
        - `domain::workspaces::manager` ruft z.B. `system::window_mechanics::set_application_focus` auf, was intern `keyboard.set_focus(...)` verwendet.
    - Die Systemschicht stellt hierfür klar definierte Traits bereit (z.B. eine `WindowManipulationApi`), die von Domänendiensten genutzt werden können. Diese API-Aufrufe werden dann in spezifische Smithay-Operationen übersetzt, um die gewünschten Aktionen im Compositor auszuführen.

Diese klare Trennung der Kommunikationspfade und -mechanismen ist entscheidend für die Entkopplung der Schichten und die Aufrechterhaltung einer sauberen Architektur.

## 3. Protokoll-Implementierungen (D-Bus, XDG Desktop Portals, MCP)

Die Systemschicht ist verantwortlich für die Implementierung von Clients und Servern für verschiedene externe Protokolle und deren Integration mit der Domänenschicht, um eine umfassende Desktop-Funktionalität zu gewährleisten.

### 3.1. D-Bus (`zbus`)

Die D-Bus-Integration erfolgt über die `zbus`-Bibliothek, die moderne asynchrone D-Bus-Kommunikation in Rust ermöglicht.87 Die Wahl von `zbus` gegenüber älteren Alternativen wie `dbus-rs` ist auf dessen asynchrone Natur und bessere Integration in moderne Rust-Ökosysteme zurückzuführen.

- **Module in `system::dbus_interfaces`**:
    
    - `connection_manager`: Stellt einen `DBusConnectionManager` bereit, der den Zugriff auf den Session-Bus (`session_bus()`) und den System-Bus (`system_bus()`) kapselt und vereinfacht.89
    - **Clients** (z.B. `upower_client`, `logind_client`, `network_manager_client`, `secrets_service_client`, `policykit_client`):
        - Definieren `zbus::proxy`-Strukturen für die jeweiligen D-Bus-Interfaces der externen Dienste (z.B. `org.freedesktop.UPower` 90, `org.freedesktop.NetworkManager` 91).
        - Implementieren einen Service-Wrapper (z.B. `UPowerClientService`), der den Proxy kapselt und eine saubere API für die Domänenschicht bereitstellt.
        - Abonnieren relevante D-Bus-Signale (z.B. `PropertiesChanged` von UPower) und konvertieren diese entweder in interne `SystemLayerEvents` (die über den `SystemEventBridge` publiziert werden) oder rufen direkt Methoden von Domänendiensten auf.
        - Bieten asynchrone Methoden für die Domänenschicht, um D-Bus-Methoden aufzurufen (z.B. `NetworkManager.ActivateConnection`) oder Eigenschaften abzufragen (z.B. `UPower.OnBattery`).
    - **Server** (z.B. `notifications_server`):
        - Implementiert das `org.freedesktop.Notifications`-Interface unter Verwendung des `#[dbus_interface]`-Makros von `zbus`.94
        - Ruft Methoden des `domain::user_centric_services::NotificationService` auf, um Aktionen wie das Anzeigen oder Schließen von Benachrichtigungen auszuführen.
        - Abonniert Domänen-Events (z.B. `NotificationDismissedEvent`, `NotificationActionInvokedEvent` aus `domain::user_centric_services::notifications_core::events`) und sendet entsprechende D-Bus-Signale (`NotificationClosed`, `ActionInvoked`) an D-Bus-Clients.
        - Verwaltet ein ID-Mapping zwischen den `u32`-basierten D-Bus-Benachrichtigungs-IDs und den `Uuid`-basierten internen `NotificationId`s der Domänenschicht.
- **Integration mit der Domäne**:
    
    - Domänendienste (z.B. `PowerManagementPolicy` in `domain::global_settings_and_state_management`) abonnieren die von den D-Bus-Client-Wrappern publizierten `SystemLayerEvents` oder halten direkte Handles zu den Client-Services, um auf Systemänderungen zu reagieren oder Aktionen auszulösen.
    - Domänendienste (z.B. `NotificationService`) werden von den D-Bus-Server-Implementierungen aufgerufen, um Anfragen von externen Anwendungen zu bedienen.

### 3.2. XDG Desktop Portals (`system::portals`)

Implementiert die Backend-Logik für ausgewählte XDG Desktop Portals, um sandboxed Anwendungen den Zugriff auf Systemressourcen auf eine kontrollierte Weise zu ermöglichen. Die `ashpd`-Bibliothek 96 dient hierbei als Referenz oder kann direkt genutzt werden, falls sie Backend-Logik für Portalserver bereitstellt; andernfalls erfolgt eine direkte `zbus`-Implementierung der Portal-D-Bus-Interfaces.

- Definiert einen Trait `SystemPortalInterface` (oder ähnlich benannt), den die D-Bus-Objekte der Portale (die in einem separaten Prozess oder Dienst laufen könnten) aufrufen.
- **`FileChooser` Portal (`org.freedesktop.portal.FileChooser`)** 96:
    - Empfängt `OpenFile`- oder `SaveFile`-Anfragen vom Portal-D-Bus-Objekt. Diese Anfragen enthalten typischerweise einen `WindowIdentifier`, um den Dialog korrekt dem anfragenden Anwendungsfenster zuzuordnen.97
    - Sendet einen Befehl an die UI-Schicht (über den `SystemEventBridge` oder eine dedizierte Schnittstelle), um einen nativen Dateidialog (z.B. mit GTK) anzuzeigen.
    - Empfängt das Ergebnis des Dateidialogs von der UI-Schicht und sendet es über D-Bus an den anfragenden Client zurück.
- **`Screenshot` Portal (`org.freedesktop.portal.Screenshot`)**:
    - Empfängt `Screenshot`- oder `PickColor`-Anfragen.
    - Interagiert mit `system::compositor::screencopy` (für Screenshots) oder einer spezifischen Compositor-Funktion (für die Farbpipette), um die angeforderte Aktion auszuführen.
    - Sendet das Ergebnis (Bilddaten oder Farbwert) über D-Bus zurück.

### 3.3. Model Context Protocol (MCP) (`system::mcp_client`)

Implementiert die Client-Seite des Model Context Protocol für die Interaktion mit KI-Modellen oder -Diensten, die dieses Protokoll verwenden. Die `mcp_client_rs`-Bibliothek 101 wird hierfür als Grundlage genutzt.

- Implementiert den Trait `SystemMcpService` aus der Domänenschicht.
- Nutzt `mcp_client_rs::McpClient` für die eigentliche Protokollkommunikation.
- **`connection_manager`**:
    - Verwaltet Verbindungen zu MCP-Servern, basierend auf `McpServerConfig`-Strukturen.
    - Kann lokale MCP-Server-Prozesse starten und deren `stdin`/`stdout` für die IPC verwalten, typischerweise unter Verwendung von `tokio::process::Command`.104
    - Holt API-Schlüssel für die Authentifizierung bei MCP-Servern über den `secrets_service_client` (siehe D-Bus-Integration).
- **`service::DefaultSystemMcpService`**:
    - Empfängt Anfragen (z.B. `CallToolParams`) vom `domain::user_centric_services::AIInteractionLogicService`.
    - Sendet diese Anfragen über `McpClient::send_request_json` an den entsprechenden MCP-Server.
    - Implementiert Timeout-Mechanismen für MCP-Anfragen.
    - Lauscht auf `McpClient::receive_message()` für Antworten und serverseitige Benachrichtigungen (Notifications).
    - Konvertiert MCP-Antworten und -Notifications in `McpClientSystemEvents` und publiziert diese über den `SystemEventBridge`. Der `AIInteractionLogicService` abonniert diese Events, um auf Ergebnisse oder serverseitige Updates zu reagieren. Die robuste Handhabung des Notification-Streams, inklusive Fehlerbehandlung und Deserialisierung, ist hierbei entscheidend.103

## 4. Implementierbare Lösungen

Dieser Abschnitt skizziert konkrete Implementierungsansätze für ausgewählte Komponenten und Interaktionen, um die Spezifikationen greifbarer zu machen.

### 4.1. Code-Skizzen (Englisch)

Die folgenden Code-Skizzen dienen zur Veranschaulichung der Struktur und Interaktion einiger Kernkomponenten. Sie sind nicht als vollständig kompilierbarer oder fehlerfreier Code gedacht, sondern sollen die in den Spezifikationen definierten Typen und Methoden im Kontext zeigen.

Hinweis zur Verwendung von Arc<Mutex<T>> und block_on in Handlern:

Die direkte Verwendung von Arc<Mutex<DesktopState>> und dessen Weitergabe an tokio::spawn muss sorgfältig auf Thread-Sicherheit und Deadlocks geprüft werden; oft sind spezifische Daten zu extrahieren oder Kanäle für die Kommunikation mit dem Haupt-Compositor-Thread zu verwenden.33 block_on in synchronen Smithay-Handlern zur Ausführung asynchroner Domänenlogik ist generell zu vermeiden. Stattdessen sollten Kommunikationsmuster wie Kanäle zwischen dem synchronen Handler-Thread und einem Tokio-Runtime-Thread verwendet werden, um Ergebnisse asynchroner Operationen zurück an den Handler zu leiten.107

Rust

```
// ==== Domain Layer: domain::workspaces::manager ====
// #[async_trait]
// pub trait WorkspaceManagerService: Send + Sync {
//     async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>;
//     //... other methods
// }

// pub struct DefaultWorkspaceManager {
//     internal: Arc<tokio::sync::Mutex<WorkspaceManagerInternalState>>,
// }

// impl WorkspaceManagerInternalState {
//     async fn set_active_workspace_impl(&mut self, id: WorkspaceId, current_active: Option<WorkspaceId>) -> Result<(), WorkspaceManagerError> {
//         if!self.workspaces.contains_key(&id) {
//             return Err(WorkspaceManagerError::SetActiveWorkspaceNotFound(id));
//         }
//         if self.active_workspace_id == Some(id) {
//             return Ok(());
//         }
//         let old_id = self.active_workspace_id.take();
//         self.active_workspace_id = Some(id);
//         self.ordered_workspace_ids.retain(|ws_id| *ws_id!= id);
//         self.ordered_workspace_ids.insert(0, id); // Example: move to front
//         let event = WorkspaceEvent::ActiveWorkspaceChanged { old_id, new_id: id };
//         if self.event_publisher.send(event).is_err() {
//             tracing::warn!("No active subscribers for ActiveWorkspaceChanged event.");
//         }
//         // self.save_configuration_internal_locked().await?; // Persist change
//         Ok(())
//     }
// }

// ==== System Layer: system::compositor::core::state::DesktopState ====
// impl SeatHandler for DesktopState {
//     type KeyboardFocus = WlSurface;
//     type PointerFocus = WlSurface;
//     type TouchFocus = WlSurface;

//     fn seat_state(&mut self) -> &mut SeatState<Self> {
//         &mut self.seat_state
//     }

//     fn focus_changed(&mut self, seat: &Seat<Self>, focused: Option<&WlSurface>) {
//         let new_focus_domain_id = focused.and_then(|s| {
//             // Logic to get DomainWindowIdentifier from WlSurface UserData
//             // Example: s.data_map().get::<SurfaceDataExt>().map(|data_ext| data_ext.domain_id.clone())
//             None // Placeholder
//         });
//         tracing::debug!(seat = %seat.name(), new_focus =?new_focus_domain_id, "Keyboard focus changed (Wayland)");
//         // Notify domain layer (e.g., window_management_policy or workspace_manager)
//         // This might involve sending an event through the SystemEventBridge or calling a domain service
//         // let event = SystemLayerEvent::InputFocusChanged { new_focus_domain_id };
//         // self.event_bridge.publish(event);

//         // For XDG Shell activation (simplified)
//         if let Some(surface_to_activate) = focused {
//             if let Some(toplevel) = self.space.window_for_surface(surface_to_activate, WindowSurfaceType::TOPLEVEL)
//                .and_then(|win_elem| match win_elem.as_ref() { // Assuming WindowElement is Arc<ManagedWindow>
//                     WindowElement::Wayland(w) => w.toplevel().ok(),
//                     _ => None,
//                 }) {
//                 toplevel.send_configure(); // To signal state change (active)
//             }
//         }
//         // Deactivate previously focused window
//         //...
//     }

//     fn cursor_image(&mut self, _seat: &Seat<Self>, image: CursorImageStatus) {
//         // Update self.current_cursor_status for the renderer
//         // *self.current_cursor_status.lock().unwrap() = image;
//         // self.damage_cursor_area(); // Signal renderer to update cursor
//     }
// }

// ==== System Layer: system::dbus_interfaces::upower_client::service ====
// #[async_trait]
// pub trait SystemUPowerClient: Send + Sync {
//    async fn is_on_battery(&self) -> Result<bool, DBusInterfaceError>;
// }

// pub struct UPowerClientService {
//     proxy: UPowerProxy<'static>, // Generated by zbus
//     event_publisher: Arc<SystemEventBridge>,
// }

// impl UPowerClientService {
//     pub async fn new(conn: Arc<Connection>, event_publisher: Arc<SystemEventBridge>) -> Result<Self, DBusInterfaceError> {
//         let proxy = UPowerProxy::new(&conn).await?;
//         let client = Self { proxy, event_publisher };
//         client.start_signal_listeners().await?;
//         Ok(client)
//     }

//     async fn start_signal_listeners(&self) -> Result<(), DBusInterfaceError> {
//         let mut props_changed_stream = self.proxy.receive_properties_changed().await?;
//         let publisher_clone = self.event_publisher.clone();
//         let proxy_clone = self.proxy.clone(); // Proxy is Arc-like or cloneable for tasks

//         tokio::spawn(async move {
//             while let Some(signal) = props_changed_stream.next().await {
//                 if let Ok(args) = signal.args() {
//                     if args.interface_name() == "org.freedesktop.UPower" {
//                         if let Some(Value::Bool(is_on_battery)) = args.changed_properties().get("OnBattery") {
//                             publisher_clone.publish(SystemLayerEvent::UPower(UPowerEvent::OnBatteryChanged(*is_on_battery)));
//                         }
//                         // Handle LidIsClosed, etc.
//                     }
//                 }
//             }
//         });
//         Ok(())
//     }
// }

// #[async_trait]
// impl SystemUPowerClient for UPowerClientService {
//     async fn is_on_battery(&self) -> Result<bool, DBusInterfaceError> {
//         Ok(self.proxy.on_battery().await?)
//     }
// }
```

### 4.2. Algorithmen

- **Token Resolution Pipeline (`domain::theming::logic`)**:
    
    1. **Sammle Basis-Tokens**: Kombiniere globale Tokens (systemweit oder anwendungsweit definiert) mit den Basis-Tokens des aktuell ausgewählten Themes (`theme_def.base_tokens`).
    2. **Varianten-Tokens anwenden**: Basierend auf dem `preferred_color_scheme` (z.B. `Light` oder `Dark`) aus der `ThemingConfiguration`, wähle die entsprechende `ThemeVariantDefinition` aus `theme_def.variants`. Überschreibe die Basis-Tokens mit den Tokens aus dieser Variante.
    3. **Akzentfarbe anwenden**: Falls eine `selected_accent_color` in der `ThemingConfiguration` gesetzt ist und das Theme Akzentfarben unterstützt (`theme_def.accentable_tokens_map`), modifiziere die entsprechenden Tokens. Für jedes Token im `accentable_tokens_map`:
        - Bei `AccentModificationType::DirectReplace`: Ersetze den Token-Wert direkt durch die Akzentfarbe.
        - Bei `AccentModificationType::Lighten(factor)` oder `AccentModificationType::Darken(factor)`: Helle den bestehenden Farbwert des Tokens um den Faktor auf oder dunkle ihn ab, unter Verwendung der Akzentfarbe als Basis oder Modifikator.
    4. **Benutzer-Overrides anwenden**: Überschreibe die bisherigen Token-Werte mit Tokens aus `config.custom_user_token_overrides`, falls vorhanden. Diese haben die höchste Priorität.
    5. **Rekursive Referenzauflösung**: Iteriere durch alle Tokens. Für jedes Token, dessen `value` ein `TokenValue::Reference(target_id)` ist:
        - Rufe `resolve_single_token_value(target_id, &mut visited_path, current_depth + 1, max_depth,...)` auf.
        - `visited_path`: Eine Liste der bisher besuchten `TokenIdentifier` in der aktuellen Auflösungskette, um Zyklen zu erkennen. Wenn `target_id` bereits in `visited_path` enthalten ist, liegt ein Zyklus vor -> `ThemingError::CyclicTokenReference`.
        - `current_depth`: Die aktuelle Rekursionstiefe. Wenn `current_depth > max_depth` (`MAX_TOKEN_RESOLUTION_DEPTH`), dann -> `ThemingError::MaxReferenceDepthExceeded`.
        - Wenn das `target_id` nicht im kombinierten Token-Set gefunden wird -> `ThemingError::MissingTokenReference`.
        - Ersetze das `TokenValue::Reference` durch den aufgelösten Wert des `target_id`.
    6. **Finale Werte konvertieren**: Konvertiere alle `TokenValue`-Varianten (die jetzt keine Referenzen mehr sein sollten) in ihre finalen String-Repräsentationen (z.B. Hex-Farbcodes, Pixelwerte mit "px"-Suffix, Schriftartnamen). Das Ergebnis ist ein `BTreeMap<TokenIdentifier, String>`.
- **Fenster-Layout-Berechnung (Beispiel für `domain::window_management_policy` bei `TilingMode::Columns`)**:
    
    1. Ermittle die verfügbare Breite für Fenster: `available_width = available_area.width - (num_windows - 1) * gap_settings.window_inner - 2 * gap_settings.screen_outer_horizontal`. (Annahme: `num_windows > 0`).
    2. Berechne die Breite pro Fenster: `width_per_window = available_width / num_windows`.
    3. Iteriere über die zu kachelnden Fenster (basierend auf `windows_info`):
        - Setze `window_height = available_area.height - 2 * gap_settings.screen_outer_vertical`.
        - Setze `window_width = width_per_window`.
        - Berücksichtige `WindowLayoutInfo.requested_min_size` und `WindowPolicyOverrides` (z.B. `min_size_override`, `fixed_size`). Passe `window_width` und `window_height` entsprechend an, falls die berechneten Werte die Minima unterschreiten oder feste Größen vorgegeben sind.
        - Positioniere das Fenster: `x_position = gap_settings.screen_outer_horizontal + previous_window_x_plus_width + gap_settings.window_inner`. `y_position = gap_settings.screen_outer_vertical`.
        - Speichere die berechnete Geometrie (`RectInt`) in `WorkspaceWindowLayout.window_geometries`.
    4. Aktualisiere `WorkspaceWindowLayout.occupied_area`.
    5. Setze `WorkspaceWindowLayout.tiling_mode_applied = TilingMode::Columns`.
- **Wayland Event zu Domänen-Event (Beispiel `XdgShellHandler::new_toplevel`)**:
    
    1. Ein Wayland-Client sendet `xdg_wm_base.get_xdg_surface` und anschließend `xdg_surface.get_toplevel`.
    2. Smithay ruft die Methode `XdgShellHandler::new_toplevel(toplevel_surface)` in `DesktopState` auf.
    3. Innerhalb des Handlers in `DesktopState`:
        - Generiere einen neuen, eindeutigen `DomainWindowIdentifier` (z.B. mittels `Uuid::new_v4()`).
        - Extrahiere initiale Client-Hinweise (z.B. `app_id`, `title` vom `toplevel_surface`).
        - Rufe asynchron (ggf. über einen Task und Channel-Kommunikation, um den synchronen Handler nicht zu blockieren) `domain_service.get_initial_window_geometry(domain_id, client_hints)` auf, um die empfohlene Startgeometrie zu erhalten.
        - Rufe asynchron `domain_service.assign_window_to_active_workspace(domain_id)` (oder eine spezifischere Zuweisungsmethode) auf.
        - Erstelle ein `WindowElement` (z.B. `WindowElement::Wayland(Window::new(toplevel_surface))`).
        - Füge das `WindowElement` zu `DesktopState.space` hinzu (z.B. `space.map_window(window_element, initial_position, activate)`).
        - Speichere das Mapping `DomainWindowIdentifier -> Arc<WindowElement>` in `DesktopState.windows`.
        - Sende einen `configure`-Event an das `toplevel_surface` mit der von der Domäne bestimmten Geometrie und Zuständen (z.B. aktiviert/maximiert).
        - Publiziere ein `SystemLayerEvent::WindowCreated { domain_id,... }` über den `SystemEventBridge`, damit die Domänenschicht (z.B. `WorkspaceManagerService`) darauf reagieren kann.

### 4.3. Relevante Rust-Bibliotheken

- **Domänenschicht**:
    - `uuid`: Für eindeutige Identifikatoren (`WorkspaceId`, `NotificationId`, etc.).
    - `chrono`: Für Zeitstempel (`DateTime<Utc>`).
    - `serde`, `serde_json`: Für Serialisierung und Deserialisierung von Datenstrukturen (z.B. Konfigurationen, Events).
    - `thiserror`: Zur einfachen Erstellung von Fehler-Enums.
    - `async-trait`: Für asynchrone Traits in Services.
    - `tokio` (speziell `tokio::sync`): Für Synchronisationsprimitive wie `Mutex`, `RwLock` und `broadcast` Channels für die Event-Verteilung.
- **Systemschicht**:
    - **Compositor**:
        - `smithay`: Das Kern-Toolkit für den Wayland-Compositor.2
        - `wayland-server`, `wayland-protocols`: Für die Wayland-Protokoll-Implementierung.122
        - `calloop`: Der Event-Loop, der von Smithay verwendet wird.36
        - `tracing`, `tracing-subscriber`: Für Logging.
    - **Input**:
        - `libinput` (C-Bibliothek) und `input-rs` (Rust-Bindings, von Smithay genutzt) 81: Für die Verarbeitung von Eingabeereignissen von Geräten.
        - `xkbcommon` (C-Bibliothek) und `xkbcommon-rs` (Rust-Bindings) 126: Für Tastatur-Layout-Management.
    - **D-Bus**:
        - `zbus`: Für die asynchrone D-Bus-Kommunikation.87
    - **Audio**:
        - `pipewire` (C-Bibliothek) und `pipewire-rs` (Rust-Bindings) 137: Für Audio-Management und Interaktion mit dem PipeWire-Server.
    - **MCP**:
        - `mcp_client_rs`: Für die Implementierung des Model Context Protocol Clients.101
    - **XDG Desktop Portals**:
        - `ashpd`: Als Referenz oder zur direkten Nutzung für die Implementierung von Portal-Backends, falls es die serverseitige Logik unterstützt. Andernfalls direkte `zbus`-Implementierung der Portal-D-Bus-Interfaces.96
- **Kernschicht (`novade_core`)**:
    - `thiserror`: Für Fehlerdefinitionen.
    - `tracing`, `serde`, `toml`, `once_cell`, `uuid`, `chrono`: Allgemeine Hilfsbibliotheken.

### 4.4. Annahmen, Einschränkungen, Alternativen

- **Annahme**: Die `novade_core`-Schicht stellt alle in der Domänenschicht referenzierten Typen (z.B. `novade_core::types::RectInt`, `novade_core::types::Color`, `novade_core::errors::CoreError`, `novade_core::config::ConfigServiceAsync`, Logging-Initialisierung) bereit.
- **Annahme**: Ein `SystemEventBridge` (basierend auf `tokio::sync::broadcast`) existiert für die interne Event-Kommunikation in der Systemschicht und als Adapter zur Domänenschicht. Dies ermöglicht eine entkoppelte Kommunikation zwischen Systemkomponenten und der Domäne.
- **Annahme**: Die Domänenschicht-Services sind asynchron implementiert und werden über `Arc<dyn TraitName>`-Handles zugänglich gemacht, um Thread-Sicherheit und flexible Injektion zu ermöglichen.
- **Einschränkung**: Detaillierte Spezifikationen zur GPU-Interaktion und die konkrete Implementierung des Renderers (z.B. spezifische OpenGL/Vulkan-Nutzung) sind in diesem Dokument nicht enthalten und müssen separat definiert werden.4
- **Einschränkung**: Die Komplexität des XWayland-Managements ist nur angedeutet. Eine vollständige XWayland-Integration erfordert eine detaillierte Implementierung eines X11 Window Managers (`X11Wm`) innerhalb des Compositors.4
- **Alternative (D-Bus)**: `dbus-rs` (älter, primär synchron) wurde zugunsten von `zbus` (modern, asynchron-fokussiert) nicht gewählt, um eine bessere Integration in das `tokio`-basierte asynchrone Ökosystem von NovaDE zu gewährleisten.87
- **Alternative (Compositor-Toolkit)**: Die direkte Nutzung von `libweston` (C-Bibliothek) wäre in Rust komplexer zu integrieren und würde weniger idiomatische Rust-APIs bieten als das Smithay-Toolkit. Smithay wurde aufgrund seiner Rust-Nativität und modularen Architektur gewählt.2

## 5. Maximale Detailtiefe

Die vorliegenden Spezifikationen versuchen, für jedes Modul und jede wesentliche Interaktion die folgenden Detailebenen abzudecken, um eine klare und direkte Implementierungsgrundlage zu schaffen:

- **Datenstrukturen**: Alle Felder werden mit exakten Rust-Typen spezifiziert. Sichtbarkeitsmodifikatoren (wo relevant für die Kapselung), abgeleitete Traits (wie `Debug`, `Clone`, `Serialize`, `Deserialize`, `PartialEq`, `Eq`, `Hash`, `Ord`, `Display`, `From`, `Default`) und wichtige Invarianten werden explizit genannt.
- **APIs (Traits und Funktionen)**: Signaturen werden präzise definiert, inklusive Parameternamen, Parametertypen, Rückgabetypen und relevanten Trait-Bounds wie `async`, `Send`, `Sync`. Vor- und Nachbedingungen für kritische Funktionen werden, wo nötig, beschrieben.
- **Events**: Die Struktur des Event-Payloads (alle Felder und deren Typen) wird definiert. Es wird angegeben, welche Komponenten typischerweise Events publizieren und welche sie abonnieren, sowie die Bedingungen, unter denen Events ausgelöst werden.
- **Fehlerbehandlung**: Spezifische Fehler-Enums werden pro Modul unter Verwendung von `thiserror` definiert. Jede Fehlervariante wird mit ihrer Bedeutung erläutert, und gegebenenfalls werden `#[source]`- oder `#[from]`-Attribute zur Fehlerverkettung spezifiziert.
- **Logik/Algorithmen**: Kritische Abläufe und Algorithmen (z.B. Token-Auflösung, Fenster-Layout-Berechnung) werden schrittweise beschrieben, um die Implementierungslogik zu verdeutlichen.
- **Interaktionen**: Die Kommunikation zwischen Modulen und Schichten über APIs und Events wird klar definiert, um das Zusammenspiel der Systemkomponenten nachvollziehbar zu machen.

Diese Detailtiefe zielt darauf ab, Interpretationsspielraum während der Entwicklung zu minimieren und eine konsistente, qualitativ hochwertige Implementierung zu ermöglichen.

## Schlussfolgerungen

Dieses Spezifikationsdokument legt eine solide Grundlage für die Entwicklung der Domänenschicht und deren Integration mit dem Wayland/Smithay-basierten Compositor für das NovaDE-Projekt. Die klare Trennung zwischen Domänenlogik und Systemimplementierungsdetails, die detaillierte Definition von Datenstrukturen, APIs und Fehlerzuständen sowie die Skizzierung von Kernalgorithmen und Interaktionsmustern bieten einen umfassenden Leitfaden für Entwickler.

Die Modularität der Domänenschicht, beispielsweise durch die Aufteilung in `domain::theming`, `domain::workspaces` und `domain::user_centric_services`, fördert die Kapselung und Wiederverwendbarkeit von Code. Die Verwendung von Rusts Typsystem und Features wie `async/await`, `thiserror` und `serde` unterstützt die Entwicklung eines robusten und wartbaren Systems.

Die Integration mit Smithay nutzt dessen Stärken im Bereich der Wayland-Protokollbehandlung und des Zustandsmanagements. Die `DesktopState`-Struktur als zentraler Punkt im Compositor, die verschiedene Handler-Traits implementiert und Smithay-Zustandsobjekte verwaltet, ist ein etabliertes Muster. Die Kommunikation zwischen Dom

## NovaDE Ultra-Feinspezifikation: Domänenschicht & Wayland/Smithay-Integration

Dieses Dokument definiert die detaillierten technischen Spezifikationen für die Domänenschicht des NovaDE-Projekts und deren Integration mit dem Wayland-Compositor, der unter Verwendung des Smithay-Toolkits implementiert wird. Es dient als direkter Implementierungsleitfaden für Entwickler.

### 1. Domänenschicht-Spezifikationen

Die Domänenschicht kapselt die Kernlogik und den Zustand der NovaDE-Desktop-Umgebung. Sie ist unabhängig von UI- und Systemdetails konzipiert.

#### 1.1. Modul: `domain::theming`

Dieses Modul ist verantwortlich für die Logik des Erscheinungsbilds (Theming), die Verwaltung von Design-Tokens, die Interpretation von Theme-Definitionen und die Ermöglichung dynamischer Theme-Wechsel.

- **Referenzierte Dokumente:** `B1 Domänenschicht.md`

##### 1.1.1. Datenstrukturen (`domain::theming::types`)

- **`TokenIdentifier`**: Wrapper für `String` zur Identifizierung von Design-Tokens (z.B. "color.background.primary").
    - Implementiert: `Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize, PartialOrd, Ord, Display, From<&str>`.
    - **Invarianten**: Nicht leer, enthält nur ASCII-Alphanumerisch, Punkte, Bindestriche.
- **`TokenValue`**: Enum zur Repräsentation von Token-Werten.
    - Varianten: `Color(String)`, `Dimension(String)`, `FontSize(String)`, `FontFamily(String)`, `FontWeight(String)`, `LineHeight(String)`, `LetterSpacing(String)`, `Border(String)`, `Shadow(String)`, `Radius(String)`, `Spacing(String)`, `ZIndex(i32)`, `Opacity(f64)` (0.0-1.0), `Text(String)`, `Reference(TokenIdentifier)`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize`.
    - `Opacity`-Werte werden auf 0.0-1.0 geklemmt.
- **`RawToken`**: Struktur zur Repräsentation eines Tokens aus einer Konfigurationsdatei.
    - Felder: `id: TokenIdentifier`, `value: TokenValue`, `description: Option<String>`, `group: Option<String>`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize`.
- **`TokenSet`**: Typalias für `std::collections::BTreeMap<TokenIdentifier, RawToken>`.
- **`ThemeIdentifier`**: Wrapper für `String` zur Identifizierung von Themes.
    - Implementiert: `Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize, PartialOrd, Ord, Display, From<&str>`.
    - **Invarianten**: Nicht leer, empfohlene Zeichen: `a-zA-Z0-9-`.
- **`ColorSchemeType`**: Enum (`Light`, `Dark`).
    - Implementiert: `Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, Default`.
- **`AccentColor`**: Struktur zur Repräsentation einer Akzentfarbe.
    - Felder: `name: Option<String>`, `value: novade_core::types::Color`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize`. (Eq/Hash für `value` ggf. manuell oder über Hex-String).
- **`ThemeVariantDefinition`**: Definition spezifischer Token-Werte für eine Theme-Variante.
    - Felder: `applies_to_scheme: ColorSchemeType`, `tokens: TokenSet`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize`.
- **`ThemeDefinition`**: Vollständige Definition eines Themes.
    - Felder: `id: ThemeIdentifier`, `name: String`, `description: Option<String>`, `author: Option<String>`, `version: Option<String>`, `base_tokens: TokenSet`, `variants: Vec<ThemeVariantDefinition>`, `supported_accent_colors: Option<Vec<AccentColor>>`, `accentable_tokens: Option<HashMap<TokenIdentifier, AccentModificationType>>`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize`.
- **`AccentModificationType`**: Enum (`DirectReplace`, `Lighten(f32)`, `Darken(f32)`).
- **`AppliedThemeState`**: Repräsentiert den aktuell aktiven, aufgelösten Theme-Zustand.
    - Felder: `theme_id: ThemeIdentifier`, `color_scheme: ColorSchemeType`, `active_accent_color: Option<AccentColor>`, `resolved_tokens: std::collections::BTreeMap<TokenIdentifier, String>`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize`.
    - **Invarianten**: `resolved_tokens` enthält keine Referenzen mehr.
- **`ThemingConfiguration`**: Benutzerspezifische Theming-Einstellungen.
    - Felder: `selected_theme_id: ThemeIdentifier`, `preferred_color_scheme: ColorSchemeType`, `selected_accent_color: Option<novade_core::types::Color>`, `custom_user_token_overrides: Option<TokenSet>`.
    - Implementiert: `Debug, Clone, PartialEq, Serialize, Deserialize, Default`.

##### 1.1.2. Fehlerbehandlung (`domain::theming::errors`)

- **`ThemingError`**: Enum (definiert mit `thiserror`).
    - Varianten: `TokenFileParseError { path: PathBuf, source: serde_json::Error }`, `TokenFileIoError { path: PathBuf, source: std::io::Error }`, `InvalidTokenData { path: PathBuf, message: String }`, `InvalidTokenValue { token_id: TokenIdentifier, value_string: String, reason: String }`, `CyclicTokenReference { token_id: TokenIdentifier, cycle_path: Vec<TokenIdentifier> }`, `ThemeFileLoadError { theme_id: ThemeIdentifier, path: PathBuf, source: serde_json::Error }`, `ThemeFileIoError { theme_id: ThemeIdentifier, path: PathBuf, source: std::io::Error }`, `InvalidThemeData { theme_id: ThemeIdentifier, path: PathBuf, message: String }`, `ThemeNotFound { theme_id: ThemeIdentifier }`, `MissingTokenReference { referencing_token_id: TokenIdentifier, target_token_id: TokenIdentifier }`, `MaxReferenceDepthExceeded { token_id: TokenIdentifier, depth: u8 }`, `ThemeApplicationError { message: String, source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`, `FallbackThemeLoadError { source: Box<dyn std::error::Error + Send + Sync + 'static> }`, `InitialConfigurationError(String)`, `InternalStateError(String)`, `EventSubscriptionError(String)`, `AccentColorApplicationError { theme_id: ThemeIdentifier, accent_color: novade_core::types::Color, details: String }`, `TokenResolutionError { token_id: TokenIdentifier, message: String, source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`, `FilesystemError(#[from] novade_core::errors::CoreError)`.

##### 1.1.3. Kernlogik (`domain::theming::logic`)

- **Konstante**: `MAX_TOKEN_RESOLUTION_DEPTH: u8 = 16;`
- **Token/Theme Laden & Validieren**:
    - `async fn load_raw_tokens_from_file(path: &Path, config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>) -> Result<TokenSet, ThemingError>`: Liest JSON, parst zu `TokenSet`, validiert Duplikate/Basisformat.
    - `async fn load_and_validate_token_files(paths: &[PathBuf], config_service: &Arc<dyn novade_core::config::ConfigServiceAsync>) -> Result<TokenSet, ThemingError>`: Lädt von Pfaden, mergt (Benutzer überschreibt System), validiert Zyklen.
    - `async fn load_theme_definition_from_file(...) -> Result<ThemeDefinition, ThemingError>`: Liest JSON, parst zu `ThemeDefinition`.
    - `async fn load_and_validate_theme_files(...) -> Result<Vec<ThemeDefinition>, ThemingError>`: Lädt Themes, validiert Token-Referenzen.
    - `fn validate_tokenset_for_cycles(tokens: &TokenSet) -> Result<(), ThemingError>`: Tiefensuche zur Zyklenerkennung.
- **Token Resolution Pipeline**:
    - `fn resolve_tokens_for_config(config: &ThemingConfiguration, theme_def: &ThemeDefinition, global_tokens: &TokenSet, accentable_tokens_map: &HashMap<TokenIdentifier, AccentModificationType>, max_depth: u8) -> Result<BTreeMap<TokenIdentifier, String>, ThemingError>`:
        1. Start mit globalen Tokens + Theme-Basis-Tokens.
        2. Varianten-Tokens anwenden.
        3. Akzentfarben anwenden (basierend auf `accentable_tokens_map` und `config.selected_accent_color`).
        4. Benutzer-Overrides anwenden.
        5. Rekursive Referenzauflösung (`resolve_single_token_value`) mit Zyklenerkennung und Tiefenbegrenzung.
        6. Finale Werte in Strings konvertieren.
- **Caching**: Cache-Schlüssel `(ThemeIdentifier, ColorSchemeType, Option<String> /* hex von AccentColor */, u64 /* hash von Overrides */)` für `AppliedThemeState`.
- **Fallback**: `fn generate_fallback_applied_state() -> AppliedThemeState` aus einkompilierten JSONs.

##### 1.1.4. API (`domain::theming::service::ThemingEngine`)

- **Struktur `ThemingEngineInternalState`**:
    - Felder: `current_config: ThemingConfiguration`, `available_themes: Vec<ThemeDefinition>`, `global_raw_tokens: TokenSet`, `applied_state: AppliedThemeState`, `theme_load_paths: Vec<PathBuf>`, `token_load_paths: Vec<PathBuf>`, `resolved_state_cache: HashMap<CacheKey, AppliedThemeState>`, `config_service: Arc<dyn novade_core::config::ConfigServiceAsync>`.
- **Struktur `ThemingEngine`**:
    - Felder: `internal_state: Arc<tokio::sync::Mutex<ThemingEngineInternalState>>`, `event_sender: tokio::sync::broadcast::Sender<ThemeChangedEvent>`.
- **Methoden**:
    - `async fn new(initial_config: ThemingConfiguration, theme_load_paths: Vec<PathBuf>, token_load_paths: Vec<PathBuf>, config_service: Arc<dyn novade_core::config::ConfigServiceAsync>, broadcast_capacity: usize) -> Result<Self, ThemingError>`
    - `async fn get_current_theme_state(&self) -> AppliedThemeState`
    - `async fn get_available_themes(&self) -> Vec<ThemeDefinition>`
    - `async fn get_current_configuration(&self) -> ThemingConfiguration`
    - `async fn update_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>`
    - `async fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>`
    - `fn subscribe_to_theme_changes(&self) -> tokio::sync::broadcast::Receiver<ThemeChangedEvent>`

##### 1.1.5. Events (`domain::theming::events`)

- **`ThemeChangedEvent`**:
    - Payload: `new_state: AppliedThemeState`.

#### 1.2. Modul: `domain::workspaces`

Verantwortlich für die Logik und Verwaltung von Arbeitsbereichen ("Spaces").

- **Referenzierte Dokumente:** `B2 Domänenschicht.md`

##### 1.2.1. Untermodul: `domain::workspaces::core`

- **Datenstrukturen (`domain::workspaces::core::types`)**:
    - `WorkspaceId`: Typalias für `uuid::Uuid`.
    - `WindowIdentifier`: Wrapper für `String`. Implementiert `new(id: impl Into<String>) -> Result<Self, WorkspaceCoreError>`.
    - `WorkspaceLayoutType`: Enum (`Floating`, `TilingHorizontal`, `TilingVertical`, `Maximized`). Implementiert `Default`.
- **Entität `Workspace` (`domain::workspaces::core::mod.rs`)**:
    - Felder: `id: WorkspaceId`, `name: String`, `persistent_id: Option<String>`, `layout_type: WorkspaceLayoutType`, `window_ids: HashSet<WindowIdentifier>`, `created_at: DateTime<Utc>`, `icon_name: Option<String>`, `accent_color_hex: Option<String>`.
    - Methoden: `new(...) -> Result<Self, WorkspaceCoreError>`, `id()`, `name()`, `rename(...)`, `set_layout_type(...)`, `add_window_id(...)` (crate-intern), `remove_window_id(...)` (crate-intern), `set_icon_name(...)`, `set_accent_color_hex(...) -> Result<...>`.
- **Event-Payloads (`domain::workspaces::core::event_data`)**: `WorkspaceRenamedData`, `WorkspaceLayoutChangedData`, `WindowAddedToWorkspaceData`, `WindowRemovedFromWorkspaceData`, `WorkspacePersistentIdChangedData`, `WorkspaceIconChangedData`, `WorkspaceAccentChangedData`.
- **Fehlerbehandlung (`domain::workspaces::core::errors`)**:
    - `WorkspaceCoreError`: `InvalidName`, `NameCannotBeEmpty`, `NameTooLong`, `InvalidPersistentId`, `WindowIdentifierEmpty`, `InvalidAccentColorFormat`.

##### 1.2.2. Untermodul: `domain::workspaces::assignment`

- **API (`domain::workspaces::assignment::mod.rs`)**: Funktionen operieren auf `&mut HashMap<WorkspaceId, Workspace>`.
    - `assign_window_to_workspace(...) -> Result<(), WindowAssignmentError>`
    - `remove_window_from_workspace(...) -> Result<bool, WindowAssignmentError>`
    - `move_window_to_workspace(...) -> Result<(), WindowAssignmentError>`
    - `find_workspace_for_window(...) -> Option<WorkspaceId>`
- **Fehlerbehandlung (`domain::workspaces::assignment::errors`)**:
    - `WindowAssignmentError`: `WorkspaceNotFound`, `WindowAlreadyAssigned`, `WindowNotAssignedToWorkspace`, `SourceWorkspaceNotFound`, `TargetWorkspaceNotFound`, `WindowNotOnSourceWorkspace`, `CannotMoveToSameWorkspace`, `RuleViolation`, `Internal`.

##### 1.2.3. Untermodul: `domain::workspaces::config`

- **Datenstrukturen (`domain::workspaces::config::mod.rs`)**:
    - `WorkspaceSnapshot`: `persistent_id: String`, `name: String`, `layout_type: WorkspaceLayoutType`, `icon_name: Option<String>`, `accent_color_hex: Option<String>`.
    - `WorkspaceSetSnapshot`: `workspaces: Vec<WorkspaceSnapshot>`, `active_workspace_persistent_id: Option<String>`.
- **API (`domain::workspaces::config::mod.rs`)**:
    - Trait `WorkspaceConfigProvider`: `async fn load_workspace_config() -> Result<WorkspaceSetSnapshot, WorkspaceConfigError>`, `async fn save_workspace_config(snapshot: &WorkspaceSetSnapshot) -> Result<(), WorkspaceConfigError>`.
    - Struktur `FilesystemConfigProvider`: Implementiert `WorkspaceConfigProvider`, nutzt `core::config::ConfigServiceAsync`.
- **Fehlerbehandlung (`domain::workspaces::config::errors`)**:
    - `WorkspaceConfigError`: `LoadError`, `SaveError`, `InvalidData`, `SerializationError`, `DeserializationError`, `PersistentIdNotFoundInLoadedSet`, `DuplicatePersistentIdInLoadedSet`.

##### 1.2.4. Untermodul: `domain::workspaces::manager`

- **API (`domain::workspaces::manager::mod.rs`)**: Trait `WorkspaceManagerService`.
    - Methoden: `async create_workspace(...)`, `async delete_workspace(...)`, `get_workspace(...) -> Option<Workspace>`, `all_workspaces_ordered() -> Vec<Workspace>`, `active_workspace_id()`, `async set_active_workspace(...)`, `async assign_window_to_active_workspace(...)`, `async assign_window_to_specific_workspace(...)`, `async remove_window_from_its_workspace(...)`, `async move_window_to_specific_workspace(...)`, `async rename_workspace(...)`, `async set_workspace_layout(...)`, `async set_workspace_icon(...)`, `async set_workspace_accent_color(...)`, `async save_configuration()`, `subscribe_to_workspace_events() -> tokio::sync::broadcast::Receiver<WorkspaceEvent>`, `async reorder_workspace(...)`.
- **Implementierung `DefaultWorkspaceManager`**: Hält `Arc<tokio::sync::Mutex<WorkspaceManagerInternalState>>`.
    - `WorkspaceManagerInternalState`: `workspaces: HashMap<WorkspaceId, Workspace>`, `active_workspace_id: Option<WorkspaceId>`, `ordered_workspace_ids: Vec<WorkspaceId>`, `next_workspace_number: u32`, `config_provider: Arc<dyn WorkspaceConfigProvider>`, `event_publisher: tokio::sync::broadcast::Sender<WorkspaceEvent>`, `ensure_unique_window_assignment: bool`.
- **Events (`domain::workspaces::manager::events`)**:
    - `WorkspaceEvent`: `WorkspaceCreated`, `WorkspaceDeleted`, `ActiveWorkspaceChanged`, `WorkspaceRenamed`, `WorkspaceLayoutChanged`, `WindowAddedToWorkspace`, `WindowRemovedFromWorkspace`, `WorkspaceOrderChanged`, `WorkspacesReloaded`, `WorkspacePersistentIdChanged`, `WorkspaceIconChanged`, `WorkspaceAccentChanged`.
- **Fehlerbehandlung (`domain::workspaces::manager::errors`)**:
    - `WorkspaceManagerError`: `WorkspaceNotFound`, `CannotDeleteLastWorkspace`, `DeleteRequiresFallbackForWindows`, `FallbackWorkspaceNotFound`, `CoreError(#[from] WorkspaceCoreError)`, `AssignmentError(#[from] WindowAssignmentError)`, `ConfigError(#[from] WorkspaceConfigError)`, `SetActiveWorkspaceNotFound`, `NoActiveWorkspace`, `DuplicatePersistentId`, `Internal`.

#### 1.3. Modul: `domain::user_centric_services`

Bündelt Logik für KI-Interaktionen und Benachrichtigungen.

- **Referenzierte Dokumente:** `B3 Domänenschicht.md`, `B4 Domänenschicht.md`

##### 1.3.1. Untermodul: `domain::user_centric_services::ai_interaction`

- **Datenstrukturen (`...::types`)**:
    - `AIDataCategory`: Enum (`UserProfile`, `ApplicationUsage`, `FileSystemRead`, `ClipboardAccess`, `LocationData`, `GenericText`, `GenericImage`).
    - `AIConsentStatus`: Enum (`Granted`, `Denied`, `PendingUserAction`, `NotRequired`).
    - `AttachmentData`: Felder `id: Uuid`, `mime_type: String`, `source_uri: Option<String>`, `content_base64: Option<String>`, `text_content: Option<String>`, `description: Option<String>`.
    - `InteractionParticipant`: Enum (`User`, `Assistant`, `System`).
    - `InteractionHistoryEntry`: Felder `entry_id: Uuid`, `timestamp: DateTime<Utc>`, `participant: InteractionParticipant`, `content: String`, `related_attachment_ids: Vec<Uuid>`.
    - `AIInteractionContext`: Felder `id: Uuid`, `creation_timestamp: DateTime<Utc>`, `active_model_id: Option<String>`, `consent_status: AIConsentStatus`, `associated_data_categories: Vec<AIDataCategory>`, `history_entries: Vec<InteractionHistoryEntry>`, `attachments: Vec<AttachmentData>`, `user_prompt_template: Option<String>`, `is_active: bool`.
    - `AIConsentScope`: Enum (`SessionOnly`, `PersistentUntilRevoked`, `SpecificDuration`).
    - `AIConsent`: Felder `id: Uuid`, `user_id: String`, `model_id: String`, `data_category: AIDataCategory`, `granted_timestamp: DateTime<Utc>`, `expiry_timestamp: Option<DateTime<Utc>>`, `is_revoked: bool`, `last_used_timestamp: Option<DateTime<Utc>>`, `consent_scope: AIConsentScope`.
    - `AIModelCapability`: Enum (`TextGeneration`, `CodeGeneration`, `Summarization`, `Translation`, `ImageAnalysis`, `FunctionCalling`).
    - `AIModelProfile`: Felder `model_id: String`, `display_name: String`, `description: String`, `provider: String`, `required_consent_categories: Vec<AIDataCategory>`, `capabilities: Vec<AIModelCapability>`, `supports_streaming: bool`, `endpoint_url: Option<String>`, `api_key_secret_name: Option<String>`, `is_default_model: bool`, `sort_order: i32`.
- **Fehlerbehandlung (`...::errors`)**: `AIInteractionError` (`ConsentCheckFailed`, `ApiKeyNotFoundInSecrets`, `ModelEndpointUnreachable`, `NoDefaultModelConfigured`, `CoreConfigError`).
- **Persistenz-Interfaces (`...::persistence_iface`)**: Traits `AIConsentProvider`, `AIModelProfileProvider`.
- **API (`...::service::AIInteractionLogicService`)**: Trait mit Methoden `initiate_interaction`, `get_interaction_context`, `provide_consent`, `get_consent_status_for_interaction`, `get_default_model`, `update_interaction_history`, `store_consent`, `get_all_user_consents`, `load_model_profiles`.
- **Implementierung `DefaultAIInteractionLogicService`**.
- **Events (`...::events::AIInteractionEventEnum`)**: `AIInteractionInitiatedEvent`, `AIConsentUpdatedEvent`, `AIContextUpdatedEvent`, `AIModelProfilesReloadedEvent`.

##### 1.3.2. Untermodul: `domain::user_centric_services::notifications_core`

- **Datenstrukturen (`...::types`)**:
    - `NotificationId`: Typalias für `uuid::Uuid`.
    - `NotificationUrgency`: Enum (`Low`, `Normal`, `Critical`).
    - `NotificationActionType`: Enum (`Callback`, `OpenLink`).
    - `NotificationAction`: Felder `key: String`, `label: String`, `action_type: NotificationActionType`.
    - `Notification`: Felder `id: Uuid`, `application_name: String`, `application_icon: Option<String>`, `summary: String`, `body: Option<String>`, `actions: Vec<NotificationAction>`, `urgency: NotificationUrgency`, `timestamp: DateTime<Utc>`, `is_read: bool`, `is_dismissed: bool`, `transient: bool`, `category: Option<String>`, `hints: HashMap<String, serde_json::Value>`, `timeout_ms: Option<u32>`.
    - `NotificationInput`: Struktur für das Erstellen von Notifications (ohne `id`, `timestamp`, `is_read`, `is_dismissed`).
    - `NotificationFilterCriteria`: Enum (mit Varianten `Unread`, `Application`, `Urgency`, `Category`, `HasAction`, `BodyContains`, `SummaryContains`, `IsTransient`, `AndTimeRange`, `And`, `Or`, `Not`).
    - `NotificationSortOrder`: Enum (`TimestampAscending`, `TimestampDescending`, `Urgency`, `ApplicationNameAscending`, `SummaryAscending`).
    - `NotificationStats`: `num_active: usize`, `num_history: usize`, `num_unread: usize`.
    - `DismissReason`: Enum (`User`, `Timeout`, `Replaced`, `AppClosed`, `SystemRequest`).
- **Fehlerbehandlung (`...::errors`)**: `NotificationError` (`NotFound`, `InvalidData`, `HistoryFull`, `ActionNotFound`, `InvalidFilterCriteria`, `ActionInvocationFailed`).
- **API (`...::service::NotificationService`)**: Trait mit Methoden `post_notification(input: NotificationInput)`, `get_notification`, `mark_as_read`, `dismiss_notification`, `get_active_notifications`, `get_notification_history`, `clear_history`, `set_do_not_disturb`, `is_do_not_disturb_enabled`, `invoke_action`, `get_stats`, `clear_all_for_app`.
- **Implementierung `DefaultNotificationService`**.
- **Events (`...::events::NotificationEventEnum`)**: `NotificationPostedEvent`, `NotificationDismissedEvent`, `NotificationReadEvent`, `NotificationActionInvokedEvent`, `DoNotDisturbModeChangedEvent`, `NotificationHistoryClearedEvent`, `NotificationPopupExpiredEvent`.

#### 1.4. Modul: `domain::notifications_rules`

Regelbasierte Verarbeitung von Benachrichtigungen.

- **Referenzierte Dokumente:** `B4 Domänenschicht.md`

##### 1.4.1. Datenstrukturen (`domain::notifications_rules::types`)

- `RuleConditionValue`: Enum (`String(String)`, `Integer(i64)`, `Boolean(bool)`, `Urgency(NotificationUrgency)`, `Regex(String)`).
- `RuleConditionOperator`: Enum (`Is`, `IsNot`, `Contains`, `NotContains`, `StartsWith`, `EndsWith`, `MatchesRegex`, `NotMatchesRegex`, `GreaterThan`, `LessThan`, `GreaterThanOrEqual`, `LessThanOrEqual`).
- `RuleConditionField`: Enum (`ApplicationName`, `Summary`, `Body`, `Urgency`, `Category`, `HintExists(String)`, `HintValue(String)`).
- `SimpleRuleCondition`: Felder `field: RuleConditionField`, `operator: RuleConditionOperator`, `value: RuleConditionValue`.
- `RuleCondition`: Enum (`Simple(SimpleRuleCondition)`, `SettingIsTrue(SettingPath)`, `And(Vec<RuleCondition>)`, `Or(Vec<RuleCondition>)`, `Not(Box<RuleCondition>`).
- `RuleAction`: Enum (`SuppressNotification`, `SetUrgency`, `AddActionToNotification`, `SetHint`, `PlaySound`, `MarkAsPersistent`, `SetTimeoutMs`, `SetCategory`, `SetSummary`, `SetBody`, `SetIcon`, `SetAccentColor`, `StopProcessingFurtherRules`, `LogMessage`).
- `NotificationRule`: Felder `id: Uuid`, `name: String`, `condition: RuleCondition`, `actions: Vec<RuleAction>`, `is_enabled: bool`, `priority: i32`. Implementiert `Default`.
- `NotificationRuleSet`: Typalias für `Vec<NotificationRule>`.

##### 1.4.2. Fehlerbehandlung (`domain::notifications_rules::errors`)

- `NotificationRulesError`: `InvalidRuleDefinition`, `ConditionEvaluationError`, `ActionApplicationError`, `SettingsAccessError(#[from] GlobalSettingsError)`, `RulePersistenceError(#[from] novade_core::errors::CoreError)`, `InvalidRegex`.

##### 1.4.3. API (`domain::notifications_rules::engine::NotificationRulesEngine`)

- **Trait `NotificationRulesEngine`**:
    - Methoden: `async reload_rules()`, `async process_notification(notification: Notification) -> Result<RuleProcessingResult, NotificationRulesError>`, `async get_rules() -> Result<NotificationRuleSet, _>`, `async update_rules(new_rules: NotificationRuleSet) -> Result<_,_>`.
- **Implementierung `DefaultNotificationRulesEngine`**: Hält `Arc<tokio::sync::RwLock<NotificationRuleSet>>`, `Arc<dyn NotificationRulesProvider>`, `Arc<dyn GlobalSettingsService>`.
- **Enum `RuleProcessingResult`**: `Allow(Notification)`, `Suppress { rule_id: Uuid }`.

##### 1.4.4. Persistenz (`domain::notifications_rules::persistence_iface`)

- **Trait `NotificationRulesProvider`**: `async fn load_rules() -> Result<NotificationRuleSet, _>`, `async fn save_rules(rules: &NotificationRuleSet) -> Result<_,_>`.
- **Implementierung `FilesystemNotificationRulesProvider`**.

#### 1.5. Modul: `domain::global_settings_and_state_management`

Verwaltung globaler Desktop-Einstellungen.

- **Referenzierte Dokumente:** `B3 Domänenschicht.md`, `B4 Domänenschicht.md`

##### 1.5.1. Datenstrukturen (`...::types`)

- `GlobalDesktopSettings`: Hauptstruktur mit Feldern wie `appearance: AppearanceSettings`, `workspace_config: WorkspaceSettings`, `input_behavior: InputBehaviorSettings`, `power_management_policy: PowerManagementPolicySettings`, `default_applications: DefaultApplicationsSettings`. Alle Felder mit `#[serde(default)]`.
- Detaillierte Unterstrukturen (`AppearanceSettings`, `FontSettings`, etc.) und Enums (`ColorScheme`, `MouseAccelerationProfile`, etc.) wie in `B3 Domänenschicht.md` und `B4 Domänenschicht.md` definiert.
- **Validierung**: Jede Einstellungs-Unterstruktur implementiert `fn validate(&self) -> Result<(), String>`. `GlobalDesktopSettings` hat `validate_recursive()`.

##### 1.5.2. Pfade (`...::paths`)

- `SettingPath`: Hierarchischer Enum zur typsicheren Adressierung aller Einstellungen. Implementiert `Display` und `TryFrom<&str>`.

##### 1.5.3. Fehlerbehandlung (`...::errors`)

- `GlobalSettingsError`: `PathNotFound { path: SettingPath }`, `InvalidValueType { path: SettingPath, ... }`, `ValidationError { path: SettingPath, reason: String }`, `SerializationError`, `DeserializationError`, `PersistenceError(#[from] GlobalSettingsPersistenceError)`.

##### 1.5.4. Persistenz-Interface (`...::persistence_iface`)

- **Trait `SettingsPersistenceProvider`**: `async fn load_global_settings() -> Result<GlobalDesktopSettings, GlobalSettingsError>`, `async fn save_global_settings(settings: &GlobalDesktopSettings) -> Result<(), GlobalSettingsError>`.
    - **Anmerkung:** Die Fehlerart hier sollte ggf. ein spezifischerer `GlobalSettingsPersistenceError` sein, um `#[from]` in `GlobalSettingsError::PersistenceError` zu ermöglichen.
- **Implementierung `FilesystemSettingsProvider`**.

##### 1.5.5. API (`...::service::GlobalSettingsService`)

- **Trait `GlobalSettingsService`**:
    - Methoden: `async load_settings()`, `async save_settings()`, `get_current_settings() -> GlobalDesktopSettings`, `async update_setting(path: SettingPath, value: serde_json::Value) -> Result<(),_>`, `get_setting(path: &SettingPath) -> Result<serde_json::Value, _>`, `async reset_to_defaults()`, `subscribe_to_setting_changes() -> tokio::sync::broadcast::Receiver<SettingChangedEvent>`.
- **Implementierung `DefaultGlobalSettingsService`**: Hält `settings: Arc<tokio::sync::RwLock<GlobalDesktopSettings>>`, `persistence_provider: Arc<dyn SettingsPersistenceProvider>`, `event_sender`.

##### 1.5.6. Events (`...::events`)

- `SettingChangedEvent { path: SettingPath, new_value: serde_json::Value }`
- `SettingsLoadedEvent { settings: GlobalDesktopSettings }`
- `SettingsSavedEvent`

#### 1.6. Modul: `domain::window_management_policy`

High-Level-Regeln und Richtlinien für Fensterplatzierung, Tiling, etc.

- **Referenzierte Dokumente:** `B3 Domänenschicht.md`

##### 1.6.1. Datenstrukturen (`...::types`)

- `TilingMode`: Enum (`Manual`, `Columns`, `Rows`, `Spiral`, `MaximizedFocused`).
- `GapSettings`: `screen_outer_horizontal: u16`, `screen_outer_vertical: u16`, `window_inner: u16`.
- `WindowSnappingPolicy`: `snap_to_screen_edges: bool`, `snap_to_other_windows: bool`, `snap_to_workspace_gaps: bool`, `snap_distance_px: u16`.
- `WindowGroupingPolicy`: `enable_manual_grouping: bool`.
- `NewWindowPlacementStrategy`: Enum (`Smart`, `Center`, `Cascade`, `UnderMouse`).
- `FocusStealingPreventionLevel`: Enum (`None`, `Moderate`, `Strict`).
- `FocusPolicy`: `focus_follows_mouse: bool`, `click_to_focus: bool`, `focus_new_windows_on_creation: bool`, `focus_new_windows_on_workspace_switch: bool`, `focus_stealing_prevention: FocusStealingPreventionLevel`.
- `WindowPolicyOverrides`: Optionale, fensterspezifische Overrides (`preferred_tiling_mode`, `is_always_floating`, `fixed_size`, `fixed_position`, `prevent_focus_stealing`, `min_size_override`, `max_size_override`).
- `WorkspaceWindowLayout`: `window_geometries: HashMap<WindowIdentifier, novade_core::types::RectInt>`, `occupied_area: Option<RectInt>`, `tiling_mode_applied: TilingMode`.
- `WindowLayoutInfo`: `id: WindowIdentifier`, `requested_min_size: Option<novade_core::types::Size<u32>>`, `is_fullscreen_requested: bool`, `is_maximized_requested: bool`.

##### 1.6.2. Fehlerbehandlung (`...::errors`)

- `WindowPolicyError`: `LayoutCalculationError`, `InvalidPolicyConfiguration`, `WindowNotFoundForPolicy`.

##### 1.6.3. API (`...::service::WindowManagementPolicyService`)

- **Trait `WindowManagementPolicyService`**:
    - Methoden: `async calculate_workspace_layout(...) -> Result<WorkspaceWindowLayout, _>`, `async get_initial_window_geometry(...) -> Result<RectInt, _>`, `async calculate_snap_target(...) -> Option<RectInt>`, `async get_effective_tiling_mode_for_workspace(...) -> Result<TilingMode, _>`, `async get_effective_gap_settings_for_workspace(...) -> Result<GapSettings, _>`, `async get_effective_snapping_policy() -> Result<WindowSnappingPolicy, _>`, `async get_effective_focus_policy() -> Result<FocusPolicy, _>`, `async get_effective_new_window_placement_strategy() -> Result<NewWindowPlacementStrategy, _>`.
- **Implementierung `DefaultWindowManagementPolicyService`**: Hält `Arc<dyn GlobalSettingsService>`.

#### 1.7. Modul: `domain::common_events`

Definition von Domänen-übergreifenden Events.

- **Referenzierte Dokumente:** `B3 Domänenschicht.md`

##### 1.7.1. Datenstrukturen (`domain::common_events::events`)

- `UserActivityType`: Enum (`MouseMoved`, `MouseClicked`, `MouseWheelScrolled`, `KeyPressed`, `TouchInteraction`, `WorkspaceSwitched`, `ApplicationFocused`, `WindowOpened`, `WindowClosed`).
- `UserActivityDetectedEvent`: Felder `event_id: Uuid`, `timestamp: DateTime<Utc>`, `activity_type: UserActivityType`, `current_session_state: UserSessionState`, `active_application_id: Option<ApplicationId>`, `active_workspace_id: Option<WorkspaceId>`.
- `ShutdownReason`: Enum (`UserRequest`, `PowerButtonPress`, `LowBattery`, `SystemUpdate`, `ApplicationRequest`, `OsError`, `Unknown`).
- `SystemShutdownInitiatedEvent`: Felder `event_id: Uuid`, `timestamp: DateTime<Utc>`, `reason: ShutdownReason`, `is_reboot: bool`, `delay_seconds: Option<u32>`, `message: Option<String>`.

#### 1.8. Modul: `domain::shared_types`

Wiederverwendbare domänenspezifische Typen.

- **Referenzierte Dokumente:** `B3 Domänenschicht.md`

##### 1.8.1. Datenstrukturen (`domain::shared_types::types`)

- `ApplicationId`: Wrapper für `String`.
- `UserSessionState`: Enum (`Active`, `Locked`, `Idle`).
- `ResourceIdentifier`: Felder `r#type: String`, `id: String`, `label: Option<String>`.

### 2. Wayland/Smithay-Integration (Systemschicht-Perspektive)

Dieser Abschnitt beschreibt, wie die Systemschicht, insbesondere der Compositor, mit Wayland-Protokollen unter Verwendung von Smithay umgeht und mit der Domänenschicht interagiert.

- **Referenzierte Dokumente:** `C1 System Implementierungsplan.md` bis `C4 Systemschicht.md`, `3. System-Details.md`.

#### 2.1. Kern-Compositor-Struktur (`system::compositor::core::state::DesktopState`)

- `DesktopState` ist die zentrale Zustandsstruktur, die Smithay-Handler-Traits implementiert.
- **Wichtige Felder (Auswahl, basierend auf Smithay-Architektur):**
    - `display_handle: DisplayHandle`
    - `loop_handle: LoopHandle<'static, Self>`
    - `clock: Clock<u64>`
    - `compositor_state: CompositorState`
    - `shm_state: ShmState`
    - `xdg_shell_state: XdgShellState`
    - `output_manager_state: OutputManagerState`
    - `seat_state: SeatState<Self>`
    - `seat: Seat<Self>`
    - `data_device_state: DataDeviceState`
    - `space: Space<WindowElement>` (wobei `WindowElement` ein Enum ist, das `Wayland(Window)`, `X11(X11Surface)`, `Layer(LayerSurface)` wrappt)
    - `windows: HashMap<DomainWindowIdentifier, Arc<WindowElement>>` (Mapping von Domänen-IDs zu Compositor-Fenstern)
    - Domänen-Service-Handles: `Arc<dyn WindowManagementPolicyService>`, `Arc<dyn WorkspaceManagerService>`, `Arc<dyn GlobalSettingsService>`.
    - `event_bridge: Arc<SystemEventBridge>`

#### 2.2. Wayland-Protokoll-Implementierungen

- **`wl_compositor`, `wl_subcompositor`**: Gehandhabt durch `CompositorState` und `CompositorHandler` in `DesktopState`.
    - `CompositorHandler::commit(surface: &WlSurface)`:
        1. Aktualisiert `SurfaceData` (Puffer, Schaden).
        2. Informiert `domain::window_management_policy` oder `domain::workspaces` bei relevanten Änderungen (z.B. neue Puffergröße könnte Layout beeinflussen).
        3. Markiert Oberfläche für Neuzeichnung.
- **`wl_shm`**: Gehandhabt durch `ShmState` und `ShmHandler` in `DesktopState`.
    - `BufferHandler::buffer_destroyed(buffer: &WlBuffer)`: Informiert Renderer, Ressourcen freizugeben.
- **`xdg_shell` (`xdg_wm_base`, `xdg_surface`, `xdg_toplevel`, `xdg_popup`)**: Gehandhabt durch `XdgShellState` und `XdgShellHandler` in `DesktopState`.
    - `XdgShellHandler::new_toplevel(surface: ToplevelSurface)`:
        1. Erstellt `WindowElement::Wayland(Window::new(ToplevelKind::Xdg(surface))))`.
        2. Generiert `DomainWindowIdentifier`.
        3. Ruft `domain::window_management_policy::get_initial_window_geometry()` auf, um Position/Größe zu bestimmen.
        4. Ruft `domain::workspaces::assign_window_to_active_workspace()` auf.
        5. Konfiguriert und mappt das Fenster im `Space`.
    - `XdgShellHandler::map_toplevel(surface: &ToplevelSurface)`: Informiert `domain::workspaces`, Fenster als gemappt zu markieren.
    - Anfragen (`set_title`, `set_app_id`, `set_maximized`, `move`, `resize`) werden an `domain::window_management_policy` delegiert, um die Policy-konforme Reaktion zu bestimmen. Das Ergebnis wird dann auf das `ToplevelSurface` angewendet (`send_configure`).
    - **Domäne -> System**: `domain::window_management_policy` gibt `WorkspaceWindowLayout` zurück. `system::window_mechanics::apply_workspace_layout` iteriert über `window_geometries` und ruft `toplevel.send_configure(...)` für jedes Fenster auf.
- **`wlr-layer-shell-unstable-v1`**: Gehandhabt durch `WlrLayerShellState` und `LayerShellHandler`.
    - `LayerShellHandler::new_layer_surface(surface: LayerSurface, ...)`: Erstellt `WindowElement::Layer(surface)`. Informiert Domäne (ggf. spezifischer Service für Shell-Elemente), platziert im `Space` gemäß Layer-Regeln.
- **`wl_output` / `xdg-output-unstable-v1`**: Gehandhabt durch `OutputManagerState`.
    - Änderungen an Outputs (Hotplug, Modusänderung) werden von `OutputHandler` verarbeitet.
    - Meldet Änderungen an `domain::global_settings_and_state_management` (um z.B. Persistenz anzustoßen) und `domain::workspaces` (kann Layout beeinflussen).
    - `system::outputs::output_manager` implementiert `wlr-output-management-unstable-v1`.
        - `apply()`: Ruft `Output::change_current_state()`. Domäne wird über `OutputConfigurationChangedEvent` informiert.
- **`wl_seat` / Eingabeprotokolle**: Gehandhabt durch `SeatState`, `SeatHandler`, `KeyboardHandle`, `PointerHandle`, `TouchHandle`.
    - Eingabe-Events von `system::input` (Libinput) werden verarbeitet.
    - `SeatHandler::focus_changed(seat: &Seat<Self>, focused: Option<&WlSurface>)`:
        1. Identifiziert `DomainWindowIdentifier` für `focused_surface`.
        2. Informiert `domain::window_management_policy` oder `domain::workspaces` über Fokusänderung.
    - `KeyboardHandle::input(keycode, state, ...)`: Sendet `wl_keyboard.key`.
    - `PointerHandle::motion(...)`: Sendet `wl_pointer.motion`, `enter`, `leave`.
    - **Domäne -> System**: `domain::window_management_policy` kann Fokuswechsel anstoßen. `system::window_mechanics::set_application_focus` ruft `keyboard.set_focus(...)`.

#### 2.3. Smithay-Typen und ihre Verwendung

- **`DisplayHandle`**: Zentrales Handle zur Wayland-Display-Instanz. Wird für die Erstellung von Globals, Client-Kommunikation etc. verwendet.
- **`CompositorState`**: Verwaltet `wl_compositor` und `wl_subcompositor` Globals und Client-Zustände.
- **`ShmState`**: Verwaltet `wl_shm` Global und SHM-Puffer.
- **`XdgShellState`**: Verwaltet `xdg_wm_base` Global und XDG-Oberflächen.
- **`OutputManagerState`**: Verwaltet `wl_output` und `zxdg_output_v1` Globals.
- **`SeatState<DesktopState>` / `Seat<DesktopState>`**: Verwalten `wl_seat` und Eingabefähigkeiten. `SeatHandler` wird von `DesktopState` implementiert.
- **`KeyboardHandle`, `PointerHandle`, `TouchHandle`**: Abstraktionen für die Interaktion mit spezifischen Eingabegeräten des Seats.
- **`Space<WindowElement>`**: Smithay's 2D-Raum zur Verwaltung von Fenstern, deren Stapelreihenfolge und zum Rendern. `WindowElement` muss das `smithay::desktop::Window` Trait implementieren.
- **`SurfaceData` (aus `smithay::wayland::compositor`)**: Hängt an `WlSurface` und speichert Puffer, Rolle, Schaden etc. NovaDE wird dies um `novade_system::compositor::surface_management::SurfaceDataExt` (oder ähnliches in `UserDataMap`) erweitern, um anwendungsspezifische Daten wie `DomainWindowIdentifier` und Renderer-Textur-Handles zu speichern.
- **`ClientData` (aus `smithay::reexports::wayland_server::backend`)**: Hängt an `Client` und kann Client-spezifische Zustände speichern (z.B. `CompositorClientState`, `XdgWmBaseClientData`).

**Änderungen an Smithay-Typen und deren Konsequenzen:**

- Das Ändern von Smithay-Typen ist in der Regel nicht vorgesehen, da sie Teil der Bibliothek sind. Man erweitert sie durch UserData oder implementiert Handler-Traits.
- Wenn Smithay eine neue Version mit geänderten Typen veröffentlicht, müssen die Handler-Implementierungen und die Nutzung dieser Typen in NovaDE angepasst werden. Dies erfordert sorgfältige Migration und Tests.
- Das UserData-Muster (`UserDataMap` an `WlSurface`, `Client` etc.) ist der primäre Mechanismus, um anwendungsspezifische Daten mit Smithay-Objekten zu assoziieren, ohne die Typen selbst zu ändern.

#### 2.4. Kommunikation Domäne &lt;-> Compositor

- **Compositor (System) -> Domäne:**
    - **Events**:
        - Neues Fenster (`XdgShellHandler::new_toplevel`): Sendet `WindowCreatedEvent { domain_id, initial_geometry, app_id, title }` an `domain::workspaces::manager` oder einen allgemeinen Event-Bus.
        - Fenster geschlossen (`XdgShellHandler::toplevel_destroyed`): Sendet `WindowClosedEvent { domain_id }`.
        - Fokusänderung durch Benutzer (`SeatHandler::focus_changed`): Sendet `FocusChangedEvent { new_focus_domain_id, old_focus_domain_id }`.
        - Benutzereingabe für Befehlspalette: Sendet spezifisches Event oder ruft Domänen-Service direkt.
        - Output-Änderung (`OutputHandler`): Sendet `OutputConfigurationChangedEvent { ... }`.
    - **Service-Aufrufe (indirekt):** Bei bestimmten Aktionen kann der Compositor Domänenlogik benötigen, z.B. um Standardplatzierung für ein neues Fenster zu erfragen.
- **Domäne -> Compositor (System):**
    - **Service-Aufrufe**:
        - `domain::window_management_policy` gibt `WorkspaceWindowLayout` an `system::window_mechanics`.
        - `system::window_mechanics::apply_workspace_layout` ruft dann z.B. `toplevel_surface.send_configure(...)` oder `space.map_window(...)`.
        - `domain::workspaces::manager` ruft `system::window_mechanics::set_application_focus` auf.
    - Die Systemschicht stellt Traits bereit (z.B. `WindowManipulationApi`), die von Domänendiensten aufgerufen werden können, um Aktionen im Compositor auszulösen. Diese API-Aufrufe werden dann in spezifische Smithay-Operationen übersetzt.

### 3. Protokoll-Implementierungen (D-Bus, XDG Desktop Portals, MCP)

Die Systemschicht implementiert Clients und Server für diese Protokolle und integriert sie mit der Domänenschicht.

#### 3.1. D-Bus (`zbus`)

- **Module in `system::dbus_interfaces`**:
    - **`connection_manager`**: Stellt `DBusConnectionManager` bereit (`session_bus()`, `system_bus()`).
    - **Clients (`upower_client`, `logind_client`, `network_manager_client`, `secrets_service_client`, `policykit_client`)**:
        - Definieren `zbus::proxy` für die D-Bus-Interfaces.
        - Implementieren einen Service (z.B. `UPowerClientService`), der den Proxy nutzt.
        - Abonnieren D-Bus-Signale und konvertieren sie in interne `SystemLayerEvent`s (publiziert über `SystemEventBridge`) oder rufen direkt Domänen-Services auf.
        - Bieten `async` Methoden für die Domänenschicht, um D-Bus-Methoden aufzurufen oder Eigenschaften abzufragen.
    - **Server (`notifications_server`)**:
        - Implementiert das `org.freedesktop.Notifications` Interface mit `#[dbus_interface]`.
        - Ruft Methoden des `domain::user_centric_services::NotificationService`.
        - Abonniert Domänen-Events (`NotificationDismissedEvent`, `NotificationActionInvokedEvent`) und sendet entsprechende D-Bus-Signale (`NotificationClosed`, `ActionInvoked`).
        - Verwaltet ID-Mapping zwischen D-Bus `u32` und Domänen `Uuid`.
- **Integration Domäne**:
    - Domänendienste (z.B. `PowerManagementPolicy`) abonnieren die von D-Bus-Clients publizierten `SystemLayerEvent`s oder haben direkte Handles zu den Client-Services.
    - Domänendienste (z.B. `NotificationService`) werden von D-Bus-Server-Implementierungen aufgerufen.

#### 3.2. XDG Desktop Portals (`system::portals`)

- Implementiert Backend-Logik für ausgewählte Portale (FileChooser, Screenshot).
- Definiert einen Trait `SystemPortalInterface` (o.ä.), den die D-Bus-Objekte der Portale (die in einem separaten Prozess oder Dienst laufen könnten) aufrufen.
- **`FileChooser`**:
    - Empfängt `OpenFile` / `SaveFile` Anfrage von Portal D-Bus Objekt.
    - Sendet Befehl an UI-Schicht (via `SystemEventBridge` oder dedizierter Schnittstelle) zur Anzeige eines GTK-Dateidialogs.
    - Empfängt Ergebnis von UI und sendet es via D-Bus zurück.
- **`Screenshot`**:
    - Empfängt `Screenshot` / `PickColor` Anfrage.
    - Interagiert mit `system::compositor::screencopy` (für Screenshots) oder einer spezifischen Compositor-Funktion (für Farbpipette).
    - Sendet Ergebnis zurück.

#### 3.3. Model Context Protocol (MCP) (`system::mcp_client`)

- Implementiert den `SystemMcpService`-Trait.
- Nutzt `mcp_client_rs::McpClient`.
- **`connection_manager`**: Verwaltet Verbindungen zu MCP-Servern (`McpServerConfig`), startet lokale Prozesse, holt API-Keys via `secrets_service_client`.
- **`service::DefaultSystemMcpService`**:
    - Empfängt Anfragen von `domain::user_centric_services::AIInteractionLogicService` (z.B. `CallToolParams`).
    - Sendet Anfragen via `McpClient::send_request_json`.
    - Handhabt Timeouts.
    - Lauscht auf `McpClient::receive_message()` für Antworten und Notifications.
    - Konvertiert MCP-Antworten/Notifications in `McpClientSystemEvent`s und publiziert sie. `AIInteractionLogicService` abonniert diese Events.

### 4. Implementierbare Lösungen

#### 4.1. Code-Skizzen (Englisch)

Rust

```
// ==== Domain Layer: domain::workspaces::manager ====
// #[async_trait]
// pub trait WorkspaceManagerService: Send + Sync {
//     async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>;
//     // ... other methods
// }

// pub struct DefaultWorkspaceManager {
//     internal: Arc<tokio::sync::Mutex<WorkspaceManagerInternalState>>,
// }

// impl WorkspaceManagerInternalState {
//     async fn set_active_workspace_impl(&mut self, id: WorkspaceId, current_active: Option<WorkspaceId>) -> Result<(), WorkspaceManagerError> {
//         if !self.workspaces.contains_key(&id) {
//             return Err(WorkspaceManagerError::SetActiveWorkspaceNotFound(id));
//         }
//         if self.active_workspace_id == Some(id) {
//             return Ok(());
//         }
//         let old_id = self.active_workspace_id.take();
//         self.active_workspace_id = Some(id);
//         self.ordered_workspace_ids.retain(|ws_id| *ws_id != id);
//         self.ordered_workspace_ids.insert(0, id); // Example: move to front

//         let event = WorkspaceEvent::ActiveWorkspaceChanged { old_id, new_id: id };
//         if self.event_publisher.send(event).is_err() {
//             tracing::warn!("No active subscribers for ActiveWorkspaceChanged event.");
//         }
//         // self.save_configuration_internal_locked().await?; // Persist change
//         Ok(())
//     }
// }

// ==== System Layer: system::compositor::core::state::DesktopState ====
// impl SeatHandler for DesktopState {
//     type KeyboardFocus = WlSurface;
//     type PointerFocus = WlSurface;
//     type TouchFocus = WlSurface;

//     fn seat_state(&mut self) -> &mut SeatState<Self> {
//         &mut self.seat_state
//     }

//     fn focus_changed(&mut self, seat: &Seat<Self>, focused: Option<&WlSurface>) {
//         let new_focus_domain_id = focused.and_then(|s| {
//             // Logic to get DomainWindowIdentifier from WlSurface UserData
//             // Example: s.data_map().get::<SurfaceDataExt>().map(|data_ext| data_ext.domain_id.clone())
//             None // Placeholder
//         });
//         tracing::debug!(seat = %seat.name(), new_focus = ?new_focus_domain_id, "Keyboard focus changed (Wayland)");

//         // Notify domain layer (e.g., window_management_policy or workspace_manager)
//         // This might involve sending an event through the SystemEventBridge or calling a domain service
//         // let event = SystemLayerEvent::InputFocusChanged { new_focus_domain_id };
//         // self.event_bridge.publish(event);

//         // For XDG Shell activation (simplified)
//         if let Some(surface_to_activate) = focused {
//             if let Some(toplevel) = self.space.window_for_surface(surface_to_activate, WindowSurfaceType::TOPLEVEL)
//                 .and_then(|win_elem| match win_elem.as_ref() { // Assuming WindowElement is Arc<ManagedWindow>
//                     WindowElement::Wayland(w) => w.toplevel().ok(),
//                     _ => None,
//                 }) {
//                 toplevel.send_configure(); // To signal state change (active)
//             }
//         }
//         // Deactivate previously focused window
//         // ...
//     }

//     fn cursor_image(&mut self, _seat: &Seat<Self>, image: CursorImageStatus) {
//         // Update self.current_cursor_status for the renderer
//         // *self.current_cursor_status.lock().unwrap() = image;
//         // self.damage_cursor_area(); // Signal renderer to update cursor
//     }
// }

// ==== System Layer: system::dbus_interfaces::upower_client::service ====
// #[async_trait]
// pub trait SystemUPowerClient: Send + Sync {
//    async fn is_on_battery(&self) -> Result<bool, DBusInterfaceError>;
// }

// pub struct UPowerClientService {
//     proxy: UPowerProxy<'static>, // Generated by zbus
//     event_publisher: Arc<SystemEventBridge>,
// }

// impl UPowerClientService {
//     pub async fn new(conn: Arc<Connection>, event_publisher: Arc<SystemEventBridge>) -> Result<Self, DBusInterfaceError> {
//         let proxy = UPowerProxy::new(&conn).await?;
//         let client = Self { proxy, event_publisher };
//         client.start_signal_listeners().await?;
//         Ok(client)
//     }

//     async fn start_signal_listeners(&self) -> Result<(), DBusInterfaceError> {
//         let mut props_changed_stream = self.proxy.receive_properties_changed().await?;
//         let publisher_clone = self.event_publisher.clone();
//         let proxy_clone = self.proxy.clone(); // Proxy is Arc-like or cloneable for tasks

//         tokio::spawn(async move {
//             while let Some(signal) = props_changed_stream.next().await {
//                 if let Ok(args) = signal.args() {
//                     if args.interface_name() == "org.freedesktop.UPower" {
//                         if let Some(Value::Bool(is_on_battery)) = args.changed_properties().get("OnBattery") {
//                             publisher_clone.publish(SystemLayerEvent::UPower(UPowerEvent::OnBatteryChanged(*is_on_battery)));
//                         }
//                         // Handle LidIsClosed, etc.
//                     }
//                 }
//             }
//         });
//         Ok(())
//     }
// }

// #[async_trait]
// impl SystemUPowerClient for UPowerClientService {
//     async fn is_on_battery(&self) -> Result<bool, DBusInterfaceError> {
//         Ok(self.proxy.on_battery().await?)
//     }
// }
```

#### 4.2. Algorithmen

- **Token Resolution Pipeline (`domain::theming::logic`)**:
    1. Sammle Basis-Tokens (globale + Theme-Basis).
    2. Überschreibe mit Varianten-Tokens (Hell/Dunkel).
    3. Wende Akzentfarbe an (ersetze definierte Akzent-Token-IDs).
    4. Überschreibe mit Benutzer-Overrides.
    5. Löse rekursiv `TokenValue::Reference` auf:
        - Verfolge besuchten Pfad zur Zyklenerkennung.
        - Prüfe maximale Rekursionstiefe.
        - Bei fehlender Referenz: Fehler.
    6. Konvertiere alle Werte in finale CSS-Strings.
- **Fenster-Layout-Berechnung (`domain::window_management_policy`)**:
    - Für `TilingMode::Columns`:
        1. Ermittle verfügbare Breite (`available_area.width` - Summe der `gap_settings.window_inner` - `2 * gap_settings.screen_outer_horizontal`).
        2. Teile verfügbare Breite durch Anzahl der zu kachelnden Fenster (`width_per_window`).
        3. Iteriere über Fenster, setze `width = width_per_window`, `height = available_area.height - 2 * gap_settings.screen_outer_vertical`. Positioniere sequentiell mit `gap_settings.window_inner`.
        4. Berücksichtige `WindowLayoutInfo.requested_min_size` und `WindowPolicyOverrides`.
- **Wayland Event zu Domänen-Event (`system::compositor` Handler)**:
    - Beispiel `XdgShellHandler::new_toplevel`:
        1. Wayland-Client sendet `xdg_wm_base.get_xdg_surface` und `xdg_surface.get_toplevel`.
        2. Smithay ruft `XdgShellHandler::new_toplevel(toplevel_surface)`.
        3. Handler in `DesktopState`:
            - Generiert `DomainWindowIdentifier`.
            - Ruft `domain_service.get_initial_window_geometry(domain_id, client_hints)`.
            - Ruft `domain_service.assign_window_to_active_workspace(domain_id)`.
            - Erstellt `WindowElement`, fügt zu `DesktopState.space` und `DesktopState.windows` hinzu.
            - Sendet `configure` Event an `toplevel_surface` mit der von der Domäne bestimmten Geometrie.
            - Publiziert `SystemLayerEvent::WindowCreated` über `SystemEventBridge`.

#### 4.3. Relevante Rust-Bibliotheken

- **Domänenschicht**: `uuid`, `chrono`, `serde`, `serde_json`, `thiserror`, `async-trait`, `tokio` (für `sync`-Primitive wie `Mutex`, `RwLock`, `broadcast`).
- **Systemschicht**:
    - Compositor: `smithay`, `wayland-server`, `wayland-protocols`, `calloop`, `tracing`.
    - Input: `libinput` (C-Lib), `input-rs` (Rust-Bindings für libinput, wird von Smithay genutzt), `xkbcommon` (C-Lib), `xkbcommon-rs` (Rust-Bindings).
    - D-Bus: `zbus`.
    - Audio: `pipewire` (C-Lib), `pipewire-rs` (Rust-Bindings).
    - MCP: `mcp_client_rs`.
    - XDG Portals: `ashpd` (als Referenz oder Nutzung, wenn es Backend-Logik für Portalserver bereitstellt, sonst direkte `zbus`-Implementierung der Portal-D-Bus-Interfaces).
- **Kernschicht**: `thiserror`, `tracing`, `tracing-subscriber`, `serde`, `toml`, `once_cell`, `uuid`, `chrono`.

#### 4.4. Annahmen, Einschränkungen, Alternativen

- **Annahme**: Die `novade-core` Schicht stellt alle spezifizierten Typen (`RectInt`, `Color`, `CoreError`, `ConfigError`, Logging-Init) bereit.
- **Annahme**: Ein `SystemEventBridge` (basierend auf `tokio::sync::broadcast`) existiert für die interne Event-Kommunikation in der Systemschicht und als Adapter zur Domänenschicht.
- **Annahme**: Die Domänenschicht-Services sind `async` und hinter `Arc<dyn TraitName>` zugänglich.
- **Einschränkung**: Detaillierte GPU-Interaktion und Renderer-Implementierung sind noch nicht spezifiziert.
- **Einschränkung**: Komplexität des XWayland-Managements ist nur angedeutet.
- **Alternative (D-Bus)**: `dbus-rs` (älter, synchron) vs. `zbus` (modern, async). `zbus` wird bevorzugt.
- **Alternative (Compositor)**: Direkte Nutzung von `libweston` (C) wäre komplexer in Rust zu integrieren als Smithay.

### 5. Maximale Detailtiefe

Die Spezifikationen versuchen, für jedes Modul und jede Interaktion die folgenden Details abzudecken:

- **Datenstrukturen**: Alle Felder mit exakten Rust-Typen, Sichtbarkeitsmodifikatoren, abgeleitete Traits (`Debug`, `Clone`, `Serialize`, etc.), Invarianten.
- **APIs (Traits/Funktionen)**: Exakte Signaturen (Parameter mit Name und Typ, Rückgabetyp, `async`, `Send`, `Sync`). Vor- und Nachbedingungen.
- **Events**: Struktur des Event-Payloads (alle Felder und Typen). Wer publiziert, wer abonniert (typischerweise). Auslösebedingungen.
- **Fehler**: Spezifische Fehler-Enums pro Modul (`thiserror`), jede Variante mit Bedeutung und ggf. `#[source]` oder `#[from]`.
- **Logik/Algorithmen**: Schritt-für-Schritt-Beschreibung kritischer Abläufe.
- **Interaktionen**: Klare Definition, wie Module und Schichten über APIs und Events kommunizieren.

Diese Detailtiefe zielt darauf ab, Interpretationsspielraum zu minimieren und eine direkte, konsistente Implementierung zu ermöglichen.

---

**Anmerkung zu den bereitgestellten Code-Skizzen:** Die Code-Skizzen sind zur Veranschaulichung und nicht als vollständig kompilierbarer oder fehlerfreier Code gedacht. Sie dienen dazu, die Struktur, Typen und Interaktionen gemäß der Spezifikation aufzuzeigen. Die tatsächliche Implementierung erfordert weitere Details, Fehlerbehandlung und Tests. `block_on` in Smithay-Handlern ist zu vermeiden; stattdessen sollten synchrone Domänenaufrufe oder eine Umstrukturierung für `async` erfolgen. Die Verwendung von `Arc<Mutex<DesktopState>>` und dessen Weitergabe an `tokio::spawn` muss sorgfältig auf Thread-Sicherheit und Deadlocks geprüft werden; oft sind spezifische Daten zu extrahieren oder Kanäle für die Kommunikation mit dem Haupt-Compositor-Thread zu verwenden.


## Technische Spezifikation der Domänenschicht

Basierend auf den bereitgestellten Dokumenten ("Domänenschicht: Theming-Engine – Ultra-Feinspezifikation (Teil 1/4)", "Domänenschicht: Implementierungsleitfaden Teil 2/4 – Workspaces (domain::workspaces)", "Domänenschicht: Detaillierte Spezifikation – Teil 3/4: Benutzerzentrierte Dienste und Globale Einstellungsverwaltung" und "Domänenschicht (Domain Layer) – Teil 4/4: Einstellungs- und Benachrichtigungs-Subsysteme" ) wird hiermit eine eigenständige, konkrete und ausführliche technische Spezifikation für die Entwicklung der Domänenschicht erstellt. Diese Spezifikation beinhaltet allgemeine Entwicklungsrichtlinien sowie spezifische Details zu den einzelnen Modulen der Domänenschicht.

### 1. Allgemeine Prinzipien und Entwicklungsrichtlinien der Domänenschicht

Die Domänenschicht ist das Herzstück der Anwendungslogik und repräsentiert die Geschäftsregeln und -konzepte der Desktop-Umgebung. Sie ist UI-unabhängig und entkoppelt von spezifischen Systemdetails oder Infrastrukturbelangen.

**Entwicklungsrichtlinien:**

- **Sprache und Tooling:** Rust wird als primäre Programmiersprache verwendet.
    - **Fehlerbehandlung:** `thiserror` wird für die Definition spezifischer, benutzerdefinierter Fehler-Enums pro Modul verwendet. Dies ermöglicht eine klare Kommunikation von Fehlerzuständen. Fehler werden über `Result<T, E>` zurückgegeben; `unwrap()` und `expect()` sind zu vermeiden, außer in absoluten Ausnahmefällen. Die `source()`-Kette von Fehlern soll durch korrekte Verwendung von `#[source]` und `#[from]` erhalten bleiben.
        
    - **Serialisierung/Deserialisierung:** `serde` (mit `serde_json` für JSON) wird für das Laden und Speichern von Konfigurationen und Datenstrukturen verwendet. Attribute wie `#[serde(rename_all = "kebab-case")]`, `#[serde(default)]` und `#[serde(skip_serializing_if = "Option::is_none")]` sollen konsistent genutzt werden.
        
    - **Asynchronität:** Wo Operationen potenziell blockierend sind (z.B. I/O beim Laden von Konfigurationen, Kommunikation mit externen Diensten), werden `async/await` und `async_trait` verwendet. Für nebenläufigen Zugriff auf geteilte Zustände sind `tokio::sync` Mechanismen wie `RwLock` und `Mutex` einzusetzen.
        
    - **Eindeutige IDs:** `uuid` (Version 4) wird zur Generierung eindeutiger Identifikatoren für Entitäten verwendet.
        
    - **Zeitstempel:** `chrono::DateTime<Utc>` wird für Zeitstempel verwendet, um Konsistenz zu gewährleisten.
        
    - **Event-Handling:** `tokio::sync::broadcast` wird für ein entkoppeltes, internes Event-System genutzt, um Änderungen an andere Systemteile zu kommunizieren.
        
- **Modularität und Kohäsion:** Die Domänenschicht ist in klar abgegrenzte Module unterteilt, die jeweils spezifische Verantwortlichkeiten haben (z.B. `domain::theming`, `domain::workspaces`, `domain::user_centric_services`, `domain::global_settings_and_state_management`, `domain::notifications_core`, `domain::notifications_rules`). Jedes Modul sollte eine hohe Kohäsion aufweisen und lose mit anderen Modulen gekoppelt sein.
    
- **Typsicherheit:** Newtypes und spezifische Enums werden verwendet, um die Typsicherheit zu erhöhen und die Semantik von Daten klarer zu gestalten (z.B. `TokenIdentifier`, `WorkspaceId`, `SettingKey` ).
    
- **Abstraktion und Schnittstellen:** Öffentliche APIs von Modulen werden oft durch Traits definiert, um Implementierungsdetails zu kapseln und Testbarkeit durch Mocking zu ermöglichen (z.B. `AIInteractionLogicService`, `NotificationService`, `GlobalSettingsService`, `SettingsProvider` ).
    
- **Zustandsverwaltung:** Veränderliche Zustände innerhalb von Services werden typischerweise mit `Arc<Mutex<...>>` oder `Arc<RwLock<...>>` gekapselt, um Thread-Sicherheit zu gewährleisten.
    
- **Validierung:** Eingabedaten und Einstellungsänderungen werden aktiv validiert, um die Konsistenz und Integrität der Domänendaten sicherzustellen.
    
- **Logging:** Das `tracing`-Framework soll für strukturiertes Logging und Debugging verwendet werden.
    
- **Dokumentation:** Öffentliche Typen, Methoden und Felder müssen umfassend mit `rustdoc`-Kommentaren dokumentiert werden, inklusive Vor- und Nachbedingungen, Fehler und Beispiele.
    
- **Testbarkeit:** Unit-Tests sind parallel zur Implementierung zu erstellen und sollen eine hohe Codeabdeckung anstreben. Mocking von Abhängigkeiten (insbesondere von Schnittstellen zur Kern- oder Systemschicht) ist entscheidend.
    

### 2. Struktur und Kernkomponenten der Domänenschicht

Die Domänenschicht besteht aus mehreren Kernmodulen, die spezifische Aufgabenbereiche abdecken:

#### 2.1. Modul: `domain::theming`

- **Verantwortlichkeit:** Logik des Erscheinungsbilds (Theming), Verwaltung von Design-Tokens, Interpretation von Theme-Definitionen, dynamische Theme-Wechsel (Farbschema, Akzentfarben).
    
- **Datenstrukturen:**
    - `TokenIdentifier` (String-Wrapper für hierarchische Token-IDs wie "color.background.primary").
        
    - `TokenValue` (Enum für Token-Wertetypen: Color, Dimension, FontSize, FontFamily, FontWeight, LineHeight, LetterSpacing, Border, Shadow, Radius, Spacing, ZIndex, Opacity, Text, Reference zu anderem Token).
        
    - `RawToken` (Struct: id, value, optionale description, group).
        
    - `TokenSet` (Typalias für `HashMap<TokenIdentifier, RawToken>`).
        
    - `ThemeIdentifier` (String-Wrapper für Theme-IDs).
        
    - `ColorSchemeType` (Enum: Light, Dark).
        
    - `AccentColor` (Struct: optionaler name, value als CSS-Farbwert).
        
    - `ThemeVariantDefinition` (Struct: applies_to_scheme, tokens als TokenSet für Überschreibungen).
        
    - `ThemeDefinition` (Struct: id, name, description, author, version, base_tokens, variants, supported_accent_colors).
        
    - `AppliedThemeState` (Struct: theme_id, color_scheme, active_accent_color, resolved_tokens als `HashMap<TokenIdentifier, String>`).
        
    - `ThemingConfiguration` (Struct: selected_theme_id, preferred_color_scheme, selected_accent_color, custom_user_token_overrides).
        
- **Kernlogik (`ThemingEngine` Service):**
    
    - Laden, Parsen und Validieren von Token- (_.tokens.json) und Theme-Definitionen (_.theme.json) von standardisierten Pfaden (System- und Benutzer-spezifisch). Validierung beinhaltet Eindeutigkeit von Token-IDs und Erkennung zyklischer Referenzen.
        
    - Token Resolution Pipeline: Auflösung von Token-Referenzen und Anwendung von Überschreibungen (Theme-Basis, Variante, Akzentfarbe, Benutzer-Overrides) in definierter Reihenfolge. Ergebnis ist der `AppliedThemeState`.
        
    - Dynamische Theme-Wechsel basierend auf Änderungen in `ThemingConfiguration`.
        
    - Caching von aufgelösten `AppliedThemeState`s.
        
- **Öffentliche API (`ThemingEngine`):**
    - `new(initial_config, theme_load_paths, token_load_paths)`: Konstruktor.
        
    - `get_current_theme_state()`: Gibt aktuellen `AppliedThemeState` zurück.
        
    - `get_available_themes()`: Gibt `Vec<ThemeDefinition>` zurück.
        
    - `get_current_configuration()`: Gibt aktuelle `ThemingConfiguration` zurück.
        
    - `update_configuration(new_config)`: Aktualisiert Konfiguration und löst Neuberechnung aus.
        
    - `reload_themes_and_tokens()`: Lädt alle Definitionen neu.
        
    - `subscribe_to_theme_changes()`: Gibt einen `mpsc::Receiver<ThemeChangedEvent>` zurück.
        
- **Events:** `ThemeChangedEvent { new_state: AppliedThemeState }`.
    
- **Fehlerbehandlung:** `ThemingError` Enum (z.B. `TokenFileParseError`, `CyclicTokenReference`, `ThemeNotFound`, `MissingTokenReference`).
    
- **Dateistruktur:** `domain/theming/{mod.rs, types.rs, errors.rs, logic.rs, default_themes/}`.
    

#### 2.2. Modul: `domain::workspaces`

Verantwortlich für die Logik und Verwaltung von Arbeitsbereichen ("Spaces" oder virtuelle Desktops). Unterteilt in `core`, `assignment`, `manager`, und `config`.

- **`workspaces::core`**: Fundamentale Workspace-Definition.
    
    - **Datenstrukturen:**
        - `WorkspaceId` (Typalias für `uuid::Uuid`).
            
        - `WindowIdentifier` (Newtype für `String`, repräsentiert Fenster-IDs).
            
        - `WorkspaceLayoutType` (Enum: Floating, TilingHorizontal, TilingVertical, Maximized; Default: Floating).
            
        - `Workspace` (Struct: id, name, persistent_id, layout_type, window_ids: `HashSet<WindowIdentifier>`, created_at). Validierungen für `name` (nicht leer, Maximallänge) und `persistent_id`.
            
    - **API (`impl Workspace`):** `new()`, `id()`, `name()`, `rename()`, `layout_type()`, `set_layout_type()`, `add_window_id()` (crate-intern), `remove_window_id()` (crate-intern), `window_ids()`, `persistent_id()`, `set_persistent_id()`, `created_at()`.
        
    - **Event-Payloads (Definiert in `core::event_data`):** `WorkspaceRenamedData`, `WorkspaceLayoutChangedData`, `WindowAddedToWorkspaceData`, `WindowRemovedFromWorkspaceData`, `WorkspacePersistentIdChangedData`.
        
    - **Fehlerbehandlung:** `WorkspaceCoreError` (z.B. `InvalidName`, `NameCannotBeEmpty`, `NameTooLong`, `InvalidPersistentId`).
        
- **`workspaces::assignment`**: Logik zur Fensterzuweisung.
    
    - **API (Freistehende Funktionen):**
        - `assign_window_to_workspace(workspaces: &mut HashMap<WorkspaceId, Workspace>, target_workspace_id, window_id, ensure_unique_assignment: bool)`
            
        - `remove_window_from_workspace(workspaces: &mut HashMap<WorkspaceId, Workspace>, source_workspace_id, window_id)`
            
        - `move_window_to_workspace(workspaces: &mut HashMap<WorkspaceId, Workspace>, source_workspace_id, target_workspace_id, window_id)`
            
        - `find_workspace_for_window(workspaces: &HashMap<WorkspaceId, Workspace>, window_id) -> Option<WorkspaceId>`
            
    - **Fehlerbehandlung:** `WindowAssignmentError` (z.B. `WorkspaceNotFound`, `WindowAlreadyAssigned`, `WindowNotOnSourceWorkspace`, `CannotMoveToSameWorkspace`, `RuleViolation`).
        
- **`workspaces::manager`**: Orchestrierung und übergeordnete Verwaltung.
    
    - **Zustand (`WorkspaceManager` Struct):** `workspaces: HashMap<WorkspaceId, Workspace>`, `active_workspace_id: Option<WorkspaceId>`, `ordered_workspace_ids: Vec<WorkspaceId>`, `next_workspace_number`, `config_provider: Arc<dyn WorkspaceConfigProvider>`, `event_publisher: Arc<dyn EventPublisher<WorkspaceEvent>>`, `ensure_unique_window_assignment: bool`.
        
    - **API (`impl WorkspaceManager`):** `new()`, `create_workspace()`, `delete_workspace()`, `get_workspace()`, `get_workspace_mut()`, `all_workspaces_ordered()`, `active_workspace_id()`, `set_active_workspace()`, `assign_window_to_active_workspace()`, `assign_window_to_specific_workspace()`, `remove_window_from_its_workspace()`, `move_window_to_specific_workspace()`, `rename_workspace()`, `set_workspace_layout()`, `save_configuration()`.
        
    - **Events (`WorkspaceEvent` Enum):** `WorkspaceCreated`, `WorkspaceDeleted`, `ActiveWorkspaceChanged`, `WorkspaceRenamed`, `WorkspaceLayoutChanged`, `WindowAddedToWorkspace`, `WindowRemovedFromWorkspace`, `WorkspaceOrderChanged`, `WorkspacesReloaded`, `WorkspacePersistentIdChanged`.
        
    - **Fehlerbehandlung:** `WorkspaceManagerError` (z.B. `WorkspaceNotFound`, `CannotDeleteLastWorkspace`, `NoActiveWorkspace`, Wraps: `WorkspaceCoreError`, `WindowAssignmentError`, `WorkspaceConfigError`).
        
- **`workspaces::config`**: Konfigurations- und Persistenzlogik.
    
    - **Datenstrukturen (Snapshots für Persistenz):**
        - `WorkspaceSnapshot` (Struct: persistent_id, name, layout_type).
            
        - `WorkspaceSetSnapshot` (Struct: workspaces: `Vec<WorkspaceSnapshot>`, active_workspace_persistent_id).
            
    - **Schnittstelle (`WorkspaceConfigProvider` Trait):**
        - `load_workspace_config() -> Result<WorkspaceSetSnapshot, WorkspaceConfigError>`
            
        - `save_workspace_config(config_snapshot: &WorkspaceSetSnapshot) -> Result<(), WorkspaceConfigError>`
            
    - **Beispielimplementierung:** `FilesystemConfigProvider` (nutzt `core::config::ConfigService`).
        
    - **Fehlerbehandlung:** `WorkspaceConfigError` (z.B. `LoadError`, `SaveError`, `InvalidData`, `SerializationError`, `DeserializationError`, `PersistentIdNotFound`, `DuplicatePersistentId`).
        

#### 2.3. Modul: `domain::user_centric_services`

Bündelt Logik für KI-Interaktionen (inkl. Einwilligungsmanagement) und ein umfassendes Benachrichtigungssystem.

- **KI-Interaktionsmanagement:**
    - **Datenstrukturen:**
        - `AIInteractionContext` (Struct: id: Uuid, creation_timestamp, active_model_id, consent_status: `AIConsentStatus`, associated_data_categories: `Vec<AIDataCategory>`, interaction_history, attachments: `Vec<AttachmentData>`).
            
        - `AIConsent` (Struct: id: Uuid, user_id, model_id, data_categories: `Vec<AIDataCategory>`, granted_timestamp, expiry_timestamp, is_revoked).
            
        - `AIModelProfile` (Struct: model_id, display_name, description, provider, required_consent_categories: `Vec<AIDataCategory>`, capabilities).
            
        - `AttachmentData` (Struct: id: Uuid, mime_type, source_uri, content, description).
            
        - `AIConsentStatus` (Enum: Granted, Denied, PendingUserAction, NotRequired).
            
        - `AIDataCategory` (Enum: UserProfile, ApplicationUsage, FileSystemRead, ClipboardAccess, LocationData, GenericText, GenericImage).
            
    - **API (`AIInteractionLogicService` Trait):** `initiate_interaction()`, `get_interaction_context()`, `provide_consent()`, `get_consent_for_model()`, `add_attachment_to_context()`, `list_available_models()`, `store_consent()`, `get_all_user_consents()`, `load_model_profiles()`.
        
    - **Events:** `AIInteractionInitiatedEvent`, `AIConsentUpdatedEvent`.
        
    - **Fehlerbehandlung:** `AIInteractionError` (z.B. `ContextNotFound`, `ConsentRequired`, `ModelNotFound`, `ConsentStorageError`, `ModelProfileLoadError`).
        
- **Benachrichtigungsmanagement:**
    - **Datenstrukturen:**
        - `Notification` (Struct: id: Uuid, application_name, application_icon, summary, body, actions: `Vec<NotificationAction>`, urgency: `NotificationUrgency`, timestamp, is_read, is_dismissed, transient).
            
        - `NotificationAction` (Struct: key, label, action_type: `NotificationActionType`).
            
        - `NotificationUrgency` (Enum: Low, Normal, Critical).
            
        - `NotificationActionType` (Enum: Callback, OpenLink).
            
        - `NotificationFilterCriteria` (Enum: Unread, Application(String), Urgency(NotificationUrgency)).
            
        - `NotificationSortOrder` (Enum: TimestampAscending, TimestampDescending, Urgency).
            
    - **API (`NotificationService` Trait):** `post_notification()`, `get_notification()`, `mark_as_read()`, `dismiss_notification()`, `get_active_notifications()`, `get_notification_history()`, `clear_history()`, `set_do_not_disturb()`, `is_do_not_disturb_enabled()`, `invoke_action()`.
        
    - **Events:** `NotificationPostedEvent`, `NotificationDismissedEvent`, `NotificationReadEvent`, `DoNotDisturbModeChangedEvent`.
        
    - **Fehlerbehandlung:** `NotificationError` (z.B. `NotFound`, `InvalidData`, `HistoryFull`, `ActionNotFound`).
        
- **Dateistruktur:** `domain/user_centric_services/{mod.rs, ai_interaction_service.rs, notification_service.rs, types.rs, errors.rs}`.
    

#### 2.4. Modul: `domain::global_settings_and_state_management` (auch `domain::settings_core` + `domain::settings_persistence_iface`)

Verantwortlich für die Repräsentation, Verwaltung und Konsistenz globaler Desktop-Einstellungen.

- **`domain::settings_core`**: Kernlogik der Einstellungsverwaltung.
    
    - **Datenstrukturen:**
        - `SettingKey` (Newtype für `String`, für Einstellungsschlüssel wie "appearance.theme.name").
            
        - `SettingValue` (Enum: Boolean, Integer, Float, String, Color, FilePath, List, Map).
            
        - `SettingMetadata` (Struct: description, default_value, value_type_hint, possible_values, validation_regex, min_value, max_value, is_sensitive, requires_restart).
            
        - `Setting` (Struct: id: Uuid, key, current_value, metadata, last_modified, is_dirty).
            
        - `GlobalDesktopSettings` (Hauptstruktur, die alle globalen Einstellungen kategorisiert, z.B. `AppearanceSettings`, `WorkspaceSettings`, `InputBehaviorSettings`, `PowerManagementPolicySettings`, `DefaultApplicationsSettings`). Jede Unterstruktur enthält spezifische Einstellungsfelder.
            
        - `SettingPath` (Enum-Hierarchie zur typsicheren Adressierung von Einstellungen, z.B. `SettingPath::Appearance(AppearanceSettingPath::FontSettings(FontSettingPath::DefaultFontSize))`).
            
    - **API (`SettingsCoreManager` oder `GlobalSettingsService` Trait):**
        - `new(provider, initial_metadata, event_channel_capacity)` / `load_settings()`
            
        - `save_settings()`
            
        - `get_current_settings()` / `get_setting_value(key)` / `get_setting(path)`
            
        - `set_setting_value(key, value)` / `update_setting(path, value: JsonValue)`
            
        - `reset_setting_to_default(key)` / `reset_to_defaults()`
            
        - `register_setting_metadata(key, metadata)`
            
        - `get_all_settings_with_metadata()`
            
        - `subscribe_to_changes()` / `subscribe_to_setting_changes()`
            
    - **Events:** `SettingChangedEvent { key/path, new_value }`, `SettingsLoadedEvent { settings }`, `SettingsSavedEvent`.
        
    - **Fehlerbehandlung:** `SettingsCoreError` / `GlobalSettingsError` (z.B. `SettingNotFound`, `ValidationError`, `PersistenceError`, `PathNotFound`, `InvalidValueType`).
        
- **`domain::settings_persistence_iface`**: Persistenzabstraktion.
    
    - **Schnittstelle (`SettingsProvider` Trait):**
        - `load_setting(key) -> Result<Option<SettingValue>, SettingsPersistenceError>`
            
        - `save_setting(key, value) -> Result<(), SettingsPersistenceError>`
            
        - `load_all_settings() -> Result<Vec<(SettingKey, SettingValue)>, SettingsPersistenceError>`
            
        - `delete_setting(key) -> Result<(), SettingsPersistenceError>`
            
        - `setting_exists(key) -> Result<bool, SettingsPersistenceError>`
            
    - **Fehlerbehandlung:** `SettingsPersistenceError` (z.B. `BackendUnavailable`, `StorageAccessError`, `SerializationError`, `DeserializationError`, `IoError`).
        
- **Dateistruktur (Global Settings):** `domain/global_settings_management/{mod.rs, service.rs, types.rs, paths.rs, errors.rs}`.
    
- **Dateistruktur (Settings Core & Persistence Interface):** `domain/src/settings_core/{mod.rs, types.rs, error.rs}`, `domain/src/settings_persistence_iface/{mod.rs, error.rs}`.
    

#### 2.5. Modul: `domain::notifications_rules`

Implementiert die Logik zur dynamischen Verarbeitung von Benachrichtigungen basierend auf konfigurierbaren Regeln.

- **Verantwortlichkeit:** Definition von Benachrichtigungsregeln (`NotificationRule`), deren Bedingungen (`RuleCondition`) und Aktionen (`RuleAction`); Bereitstellung einer Engine (`NotificationRulesEngine`) zur Regelauswertung und -anwendung.
    
- **Datenstrukturen:**
    - `RuleCondition` (Enum: AppNameIs, AppNameMatches (Regex), SummaryContains, UrgencyIs, CategoryIs, HintExists, HintValueIs, SettingIsTrue, LogicalAnd, LogicalOr, LogicalNot etc.).
        
    - `RuleAction` (Enum: SuppressNotification, SetUrgency, AddAction, SetHint, PlaySound, MarkAsPersistent, SetExpiration, LogMessage etc.).
        
    - `NotificationRule` (Struct: id, description, conditions: `RuleCondition`, actions: `Vec<RuleAction>`, is_enabled, priority, stop_after).
        
- **Kernlogik (`NotificationRulesEngine` Service):**
    - Lädt und verwaltet Regeldefinitionen (sortiert nach Priorität).
        
    - `process_notification(notification)`: Wertet Regeln gegen eine eingehende Benachrichtigung aus.
        
        - Gibt `RuleProcessingResult` zurück: `Allow(modified_notification)` oder `Suppress(rule_id)`.
            
    - `evaluate_condition(condition, notification, rule)`: Rekursive Auswertung von Regelbedingungen. Interagiert mit `SettingsCoreManager` für `Setting*`-Bedingungen.
        
    - `apply_action(action, notification, rule)`: Anwendung von Regelaktionen auf eine Benachrichtigung.
        
    - Reagiert auf `SettingChangedEvent` (optional, zur Cache-Invalidierung oder Neubewertung).
        
- **Öffentliche API (`NotificationRulesEngine`):**
    - `new(settings_manager, initial_rules, settings_event_receiver)`
        
    - `load_rules(new_rules)`
        
    - `process_notification(notification) -> Result<RuleProcessingResult, NotificationRulesError>`
        
    - `handle_setting_changed(event)` (intern aufgerufen).
        
- **Fehlerbehandlung:** `NotificationRulesError` (z.B. `InvalidRuleDefinition`, `ConditionEvaluationError`, `ActionApplicationError`, `SettingsAccessError`).
    
- **Dateistruktur:** `domain/src/notifications_rules/{mod.rs, types.rs, error.rs}`.

### 3. Interaktionen und Abhängigkeiten

- **Domänenmodule untereinander:**
    - `NotificationCoreManager` nutzt `NotificationRulesEngine` zur Verarbeitung von Benachrichtigungen.
        
    - `NotificationRulesEngine` nutzt `SettingsCoreManager` (oder `GlobalSettingsService`), um regelbedingte Einstellungen abzufragen.
        
    - `ThemingEngine` reagiert auf `SettingChangedEvent` von `SettingsCoreManager` für themenrelevante Einstellungen.
        
    - Services aus `domain::user_centric_services` und `domain::workspaces` können globale Einstellungen von `GlobalSettingsService` lesen.
        
- **Abhängigkeiten zur Kernschicht (`core::*`):**
    - `core::config`: Wird von `domain::settings_persistence_iface`-Implementierungen und `domain::workspaces::config` für das Lesen/Schreiben von Konfigurationsdateien genutzt.
        
    - `core::errors`: Basisfehlertypen können in Domänenfehler gewrappt werden.
        
    - `core::types`: Fundamentale Typen wie `Uuid`, `DateTime<Utc>`.
        
    - `core::logging` (`tracing`): Wird für Logging verwendet.
        
- **Schnittstellen zu höheren Schichten (System- und UI-Schicht):**
    - Die Domänenschicht stellt ihre Funktionalität über öffentliche APIs (oft Traits) ihrer Service-Komponenten bereit.
        
    - Die UI-Schicht (z.B. `ui::control_center`, `ui::shell` ) konsumiert diese APIs und reagiert auf Events aus der Domänenschicht.
        
    - Die Systemschicht (z.B. MCP-Client, D-Bus Handler, Compositor ) interagiert ebenfalls mit den Domänendiensten und leitet Systemereignisse an diese weiter oder setzt deren Anweisungen um.
        

### 4. Zusammenfassende Betrachtungen

Die Domänenschicht ist als eine Sammlung modularer, voneinander entkoppelter Komponenten konzipiert, die jeweils klar definierte Verantwortlichkeiten besitzen. Durch die konsequente Anwendung von Prinzipien wie Typsicherheit, expliziter Fehlerbehandlung, Event-basierter Kommunikation und der Abstraktion von Persistenz- und UI-Belangen wird eine robuste, wartbare und erweiterbare Grundlage für die Desktop-Umgebung geschaffen. Die detaillierten Spezifikationen der einzelnen Module, ihrer Datenstrukturen, APIs und Fehlerfälle dienen als direkter Leitfaden für die Implementierung.

## Technische Spezifikation und Entwicklungsrichtlinien: Systemschicht

Die Systemschicht bildet das Fundament der Desktop-Umgebung und ist verantwortlich für die Kernfunktionalitäten, die Interaktion mit der Hardware und die Bereitstellung grundlegender Dienste für höhere Schichten. Sie implementiert die "Mechanik" der Desktop-Umgebung, indem sie übergeordnete Richtlinien und Benutzerinteraktionen in konkrete Systemaktionen übersetzt. Dies erfordert eine präzise und robuste Interaktion mit Wayland-Protokollen, D-Bus-Systemdiensten und potenziell direkter Hardware-Interaktion (z.B. DRM). Die Stabilität und Reaktionsfähigkeit der gesamten Umgebung hängt maßgeblich von der Zuverlässigkeit dieser Schicht ab. Daher sind Resilienz, asynchrone Operationen, umfassendes Fehlermanagement und sorgfältige Zustandssynchronisation zentrale Entwurfsprinzipien.

### 1. Entwicklungsrichtlinien

Die Entwicklung aller Komponenten der Systemschicht muss strengen Richtlinien folgen, um Konsistenz, Qualität und Wartbarkeit zu gewährleisten.

- **Programmiersprache:** Rust ist die primäre Programmiersprache.
    
- **Coding Style & Formatierung:** Verbindliche Nutzung von `rustfmt` mit Standardkonfiguration. Einhaltung der Rust API Guidelines ist erforderlich.
    
- **API-Design:** Befolgung der Rust API Guidelines Checklist für konsistente und idiomatische Schnittstellen.
    
- **Fehlerbehandlung:** Konsequente Verwendung des `thiserror`-Crates zur Definition spezifischer Fehler-Enums pro Modul (z.B. `CompositorCoreError`, `ShmError`, `XdgShellError`, `DBusError`, `OutputError`, `AudioError`, `McpError`, `PortalsError`). Alle öffentlichen Funktionen in den Modulen sollten `Result<T, SpecificError>` zurückgeben.
    
- **Logging & Tracing:** Einsatz des `tracing`-Crate-Frameworks (`tracing::{info, debug, error}`) für strukturiertes, kontextbezogenes Logging und Tracing von Operationen.
    
- **Dokumentkonventionen:** Eine Tabelle mit definierten Begriffen und Konventionen (z.B. `DesktopState`, `WlFoo`, `XdgFoo`) wird verwendet, um Klarheit und Konsistenz in der Terminologie sicherzustellen.
    
- **Asynchronität:** Asynchrone Operationen sind für alle potenziell blockierenden E/A-Vorgänge zu verwenden, insbesondere bei D-Bus-Aufrufen (mittels `zbus`) und Wayland-Ereignisverarbeitung.
    
- **Strikte Einhaltung der Gesamtspezifikation:** Die hier beschriebenen Spezifikationen sind eine detaillierte Erweiterung der "Technischen Gesamtspezifikation und Entwicklungsrichtlinien". Deren Einhaltung ist bindend.
    

### 2. Modul: `system::compositor` (Smithay-basierter Wayland Compositor)

Dieses Modul implementiert die Kernlogik des Wayland-Compositors unter Verwendung des Smithay-Toolkits. Es ist verantwortlich für die Verwaltung von Wayland-Client-Verbindungen, Oberflächen, Pufferbehandlung und die Integration mit Shell-Protokollen. Die Designphilosophie von Smithay erfordert ein signifikantes Zustandsmanagement innerhalb der zentralen `DesktopState`-Struktur.

#### 2.1. Submodul: `system::compositor::core`

Dieses Submodul etabliert die grundlegenden Elemente für die Verwaltung von Wayland-Oberflächen und die Kernoperationen des Compositors.

**Dateien:**

- `compositor_state.rs`:
    - **Zweck:** Definiert und verwaltet den primären Zustand für die Globals `wl_compositor` und `wl_subcompositor` und handhabt den Client-spezifischen Compositor-Zustand.
        
    - **Struktur `CompositorCoreError`:** Definiert Fehler spezifisch für Kernoperationen des Compositors mittels `thiserror`. Varianten umfassen `GlobalCreationFailed`, `RoleError`, `ClientDataMissing`, `SurfaceDataMissing`, `InvalidSurfaceState`.
        
    - **Struktur `DesktopState` (teilweise):** Kapselt den zentralen Zustand. Enthält `compositor_state: CompositorState`, `display_handle: DisplayHandle`, `loop_handle: LoopHandle<Self>`. Ein Konstruktor `new(display_handle, loop_handle)` initialisiert diese Felder.
        
    - **Struktur `ClientCompositorData`:** Speichert `CompositorClientState` pro Client.
        
    - **Implementierung `CompositorHandler` für `DesktopState`:** Delegiert Compositor-Ereignisse an die Anwendungslogik.
        
        - `compositor_state()`: Gibt `&mut self.compositor_state` zurück.
            
        - `client_compositor_state()`: Ruft `ClientCompositorData` aus `Client::get_data()` ab und gibt `&compositor_state` daraus zurück. Loggt Fehler und panicked bei fehlenden Daten.
            
        - `commit()`: Greift auf `SurfaceData` zu, prüft auf neue Puffer, aktualisiert Schadensverfolgung, löst rollenspezifische Logik aus und markiert die Oberfläche für Neuzeichnung. Verwendet `smithay::wayland::compositor::with_states`.
            
        - `new_surface()`: Initialisiert `SurfaceData` für die neue `WlSurface` und speichert sie in deren `data_map`. Fügt einen Zerstörungs-Hook hinzu.
            
        - `new_subsurface()`: Aktualisiert `SurfaceData` der neuen und der Eltern-Oberfläche, um die Eltern-Kind-Beziehung herzustellen.
            
        - `destroyed()`: Die primäre Bereinigung wird von Smithay gehandhabt. Externe Referenzen müssen hier oder über Zerstörungs-Hooks bereinigt werden.
            
    - **Implementierung `GlobalDispatch<WlCompositor, ()>` für `DesktopState`:**
        - `bind()`: Protokolliert die Bind-Anfrage, initialisiert bei Bedarf `ClientCompositorData` für den Client und initialisiert die Ressource mit `data_init.init()`. `CompositorState::new()` erstellt das Global.
            
    - **Implementierung `GlobalDispatch<WlSubcompositor, ()>` für `DesktopState`:**
        - `bind()`: Protokolliert die Bind-Anfrage und initialisiert die Ressource. `CompositorState::new()` erstellt das Global.
            
- `surface_management.rs`:
    - **Zweck:** Definiert `SurfaceData` und zugehörige Hilfsfunktionen.
        
    - **Struktur `SurfaceData`:** Gespeichert in der `UserDataMap` jeder `WlSurface`. Enthält Felder wie `id: uuid::Uuid`, `role: Option<String>`, `client_id`, `current_buffer`, `pending_buffer`, `texture_id` (als `Option<Box<dyn RenderableTexture>>`), `last_commit_serial`, `damage_regions_buffer_coords`, `opaque_region`, `input_region`, `user_data_ext`, `parent`, `children`, `pre_commit_hooks`, `post_commit_hooks`, `destruction_hooks`. `RenderableTexture` muss `Send + Sync` sein.
        
    - **Methoden für `SurfaceData`:** `new()`, `set_role()`, `get_role()`, `attach_buffer()`, `commit_buffer()`, `add_damage_buffer_coords()`, `take_damage_buffer_coords()`.
        
    - **Enum `SurfaceRoleError`:** Variante `RoleAlreadySet`.
        
    - **Funktionen:** `get_surface_data()`, `with_surface_data()` (kapselt Mutex-Locking), `give_surface_role()` (verwendet `smithay::wayland::compositor::give_role`), `get_surface_role()` (verwendet `smithay::wayland::compositor::get_role`).
        
- `global_objects.rs`:
    - **Zweck:** Zentralisiert die Erstellung der Kern-Wayland-Globals.
        
    - **Funktion `create_core_compositor_globals()`:** Erstellt `CompositorState` und speichert es in `DesktopState`. Dies registriert intern `wl_compositor` (Version 6) und `wl_subcompositor` (Version 1).
        

#### 2.2. Submodul: `system::compositor::shm` (SHM-Pufferbehandlung)

Dieses Submodul implementiert die Unterstützung für `wl_shm`, wodurch Clients Shared-Memory-Puffer mit dem Compositor teilen können.

**Dateien:**

- `shm_state.rs`:
    - **Zweck:** Verwaltet das `wl_shm`-Global und handhabt die Erstellung und den Zugriff auf SHM-Puffer.
        
    - **Struktur `ShmError`:** Definiert Fehler für SHM-Operationen (z.B. `PoolCreationFailed`, `BufferCreationFailed`, `InvalidFormat`, `AccessError`).
        
    - **`DesktopState` (teilweise):** Enthält `shm_state: ShmState` und `shm_global: GlobalId`.
        
    - **Implementierung `ShmHandler` für `DesktopState`:**
        - `shm_state()`: Gibt `&self.shm_state` zurück.
            
    - **Implementierung `BufferHandler` für `DesktopState`:** (Gilt für alle `wl_buffer`, nicht nur SHM)
        
        - `buffer_destroyed()`: Protokolliert, benachrichtigt das Rendering-Backend zur Ressourcenfreigabe und entfernt Puffer-Referenzen aus `SurfaceData`-Instanzen.
            
    - **Implementierung `GlobalDispatch<WlShm, ()>` für `DesktopState`:**
        - `bind()`: Protokolliert die Bindung und initialisiert die Ressource. `ShmState` sendet `format`-Ereignisse.
            
    - **Funktion `create_shm_global()`:** Definiert unterstützte SHM-Formate (Standard: ARGB8888, XRGB8888), erstellt `ShmState`, speichert `shm_state` und `shm_global` (zurückgegeben von `shm_state.global().clone()`) in `DesktopState`.
        
- `shm_buffer_access.rs`:
    - **Zweck:** Bietet sicheren Zugriff auf Inhalte von SHM-Puffern.
        
    - **Funktion `with_shm_buffer_contents()`:** Verwendet intern `smithay::wayland::shm::with_buffer_contents`, um einen Callback mit Zeiger und Länge der Pufferdaten aufzurufen. Kapselt die Unsicherheit der Zeiger-Dereferenzierung.
        

#### 2.3. Submodul: `system::compositor::xdg_shell` (XDG-Shell-Integration)

Implementiert das `xdg_shell`-Protokoll zur Verwaltung moderner Desktop-Fenster (Toplevels und Popups). Das Protokoll ist komplex und erfordert signifikante Logik in den Handler-Methoden, um Oberflächenzustände, Interaktionen mit der Fenstermanagement-Richtlinie und das Senden korrekter Wayland-Ereignisse zu verwalten. Eine robuste Fehlerbehandlung und Zustandsvalidierung sind entscheidend.

**Dateien:**

- `xdg_shell_state.rs`:
    - **Zweck:** Verwaltet das `xdg_wm_base`-Global und die zugehörigen XDG-Oberflächenzustände.
        
    - **Struktur `XdgShellError`:** Definiert Fehler wie `InvalidSurfaceRole`, `WindowHandlingError`, `PopupPositioningError`, `InvalidAckConfigureSerial`, `ToplevelNotFound`, `PopupNotFound`.
        
    - **`DesktopState` (teilweise):** Enthält `xdg_shell_state: XdgShellState`, `xdg_shell_global: GlobalId`, `toplevels: HashMap<WlSurface, Arc<Mutex<ManagedToplevel>>>`, `popups: HashMap<WlSurface, Arc<Mutex<ManagedPopup>>>`.
        
    - **Implementierung `XdgShellHandler` für `DesktopState`:** Die Implementierung der Methoden wird in `xdg_handlers.rs` detailliert. `xdg_shell_state()` gibt `&mut self.xdg_shell_state` zurück.
        
    - **Implementierung `GlobalDispatch<XdgWmBase, GlobalId>` für `DesktopState`:**
        - `bind()`: Protokolliert, ruft `state.xdg_shell_state.new_client()` auf, um `ShellClientUserData` zu erhalten, und initialisiert die Ressource damit. `XdgShellState` handhabt das Ping/Pong-Verhalten.
            
    - **Funktion `create_xdg_shell_global()`:** Erstellt `XdgShellState`, ruft dessen `global()`-Methode auf und speichert beides in `DesktopState`.
        
- `toplevel_management.rs`:
    - **Zweck:** Definiert Datenstrukturen und Logik spezifisch für XDG-Toplevel-Fenster.
        
    - **Struktur `ManagedToplevel`:** Kapselt `ToplevelSurface` von Smithay und fügt anwendungsspezifische Zustände hinzu. Felder umfassen `id: uuid::Uuid`, `surface_handle: ToplevelSurface`, `wl_surface: WlSurface`, `app_id`, `title`, `current_state: ToplevelWindowState`, `pending_state: ToplevelWindowState`, `window_geometry`, `min_size`, `max_size`, `parent` (für transiente Fenster), `client_provides_decorations`, `last_configure_serial`, `acked_configure_serial`. Methoden: `new()`, `send_configure()` (sendet `xdg_toplevel.configure` und `xdg_surface.configure`), `ack_configure()`.
        
    - **Struktur `ToplevelWindowState`:** Felder: `size`, `maximized`, `fullscreen`, `resizing`, `activated`, `suspended`, `decorations`.
        
    - **Struktur `ToplevelSurfaceUserData`:** In `WlSurface::data_map()` gespeichert, enthält `managed_toplevel_id`.
        
- `popup_management.rs`:
    - **Zweck:** Definiert Datenstrukturen und Logik spezifisch für XDG-Popup-Fenster.
        
    - **Struktur `ManagedPopup`:** Kapselt `PopupSurface` von Smithay. Felder: `id: uuid::Uuid`, `surface_handle: PopupSurface`, `wl_surface: WlSurface`, `parent_wl_surface`, `positioner_state: PositionerState`, `current_geometry`, `last_configure_serial`, `acked_configure_serial`. Methoden: `new()`, `send_configure()`, `ack_configure()`, `calculate_geometry()`.
        
    - **Struktur `PopupSurfaceUserData`:** In `WlSurface::data_map()` gespeichert, enthält `managed_popup_id`.
        
- `xdg_handlers.rs`:
    - **Zweck:** Detaillierte Implementierung der `XdgShellHandler`-Methoden für `DesktopState`.
        
    - `new_toplevel()`: Erstellt `ManagedToplevel`, speichert es in `state.toplevels` und dessen ID in `ToplevelSurfaceUserData` der `WlSurface`. Sendet initiale Konfiguration.
        
    - `new_popup()`: Erstellt `ManagedPopup`, speichert es in `state.popups` und dessen ID in `PopupSurfaceUserData`. Sendet initiale Konfiguration.
        
    - `map_toplevel()`: Führt Logik für das Mapping des Toplevels aus (Sichtbarkeit, initiale Position/Größe), ruft ggf. `send_configure()` auf.
        
    - `ack_configure()`: Verarbeitet `ack_configure` vom Client für Toplevels oder Popups, ruft `ack_configure()` auf der `ManagedEntity` auf.
        
    - `toplevel_request_set_title()`: Aktualisiert `title` im `ManagedToplevel` und benachrichtigt die UI-Schicht.
        
    - Weitere Handler (z.B. für `set_app_id`, `set_maximized`, `move`, `resize`) werden analog implementiert, aktualisieren den Zustand von `ManagedToplevel`/`ManagedPopup` und lösen ggf. neue `configure`-Zyklen aus oder interagieren mit dem Input-System.
        

#### 2.4. Submodul: `system::compositor::display_loop` (Display und Ereignisschleife)

Verantwortlich für die Einrichtung des Wayland-Display-Kernobjekts und dessen Integration in die `calloop`-Ereignisschleife. Die `calloop`-Ereignisschleife ist zentral für Smithay. Langlaufende Operationen in Callbacks müssen vermieden werden.

**Dateien:**

- `display_setup.rs`:
    - **Zweck:** Initialisiert das Wayland Display und `DisplayHandle`.
        
    - **Struktur `ClientData`:** Assoziiert mit `wayland_server::Client`. Enthält `id: uuid::Uuid`, `client_name: Option<String>`, `user_data: UserDataMap`.
        
    - **Funktion `init_wayland_display_and_loop()` (konzeptionell):** Erstellt `EventLoop<DesktopState>` und `Display<DesktopState>`. `DisplayHandle` und `LoopHandle` werden in `DesktopState` gespeichert.
        
    - **Enum `InitError`:** Varianten `WaylandDisplayCreationFailed`, `EventLoopCreationFailed`.
        
- `event_loop_integration.rs`:
    - **Zweck:** Integriert die Wayland-Anzeige in die `calloop`-Ereignisschleife.
        
    - **Funktion `register_wayland_source()`:** Ruft den Dateideskriptor der Wayland-Anzeige ab. Erstellt eine `calloop::generic::Generic<FileDescriptor>`-Ereignisquelle. Fügt die Quelle in die Ereignisschleife ein (`loop_handle.insert_source()`). Der Callback ruft `shared_data.display_handle.dispatch_clients(shared_data)` auf und bei Erfolg `shared_data.display_handle.flush_clients()`. `display_handle.flush_clients()` muss regelmäßig aufgerufen werden.
        

#### 2.5. Submodul: `system::compositor::renderer_interface` (Renderer-Schnittstelle)

Definiert abstrakte Schnittstellen für Rendering-Operationen, um die Kernlogik des Compositors von spezifischen Rendering-Backends zu entkoppeln (z.B. DRM/GBM, Winit/EGL). Schadensverfolgung ist für effizientes Rendering unerlässlich und muss integriert werden.

**Dateien:**

- `abstraction.rs`:
    - **Zweck:** Definiert Traits für Rendering-Operationen.
        
    - **Trait `FrameRenderer`:**
        - Methoden: `new()`, `render_frame()` (nimmt `RenderElement`), `present_frame()`, `create_texture_from_shm()`, `create_texture_from_dmabuf()` (für spätere Teile), `screen_size()`.
            
    - **Trait `RenderableTexture` (`Send + Sync + Debug`):**
        - Methoden: `id() -> uuid::Uuid`, `bind()`, `width_px()`, `height_px()`, `format()`.
            
    - **Enum `RenderElement` (konzeptionell, analog zu Smithays `Element`):** Varianten `Surface` (mit `texture: Arc<dyn RenderableTexture>`), `SolidColor`, `Cursor`.
        
    - **Struktur `Color`:** Felder `r, g, b, a` als `f32`.
        
    - **Enum `RendererError`:** Varianten `ContextCreationFailed`, `ShaderCompilationFailed`, `TextureUploadFailed`, `BufferSwapFailed`, `InvalidBufferType`, `DrmError`, `EglError`, `Generic`.
        

### 3. Modul: `system::input` (Libinput-basierte Eingabeverarbeitung)

Verantwortlich für die gesamte Verarbeitung von Benutzereingaben von Geräten wie Tastaturen, Mäusen und Touchpads. Nutzt `libinput` für Rohdaten und Smithay-Abstraktionen (`LibinputInputBackend`, `SeatState`, `SeatHandler`) für Seat- und Fokusmanagement. Latenz oder fehlerhafte Verarbeitung hier beeinträchtigen die UX erheblich. Die Transformation von `libinput`-Ereignissen in Wayland-Ereignisse, inklusive Koordinatentransformationen und Fokuslogik, muss präzise sein. Erweiterte Eingabefunktionen wie Gesten müssen berücksichtigt werden. `xkbcommon` ist fundamental für die korrekte Interpretation von Tastatureingaben.

#### 3.1. Submodul: `system::input::seat_manager`

Definiert und verwaltet `SeatState` und `SeatHandler` für Eingabefokus und die Bekanntmachung von Fähigkeiten.

**Dateien:**

- `seat_state.rs`:
    - **Struktur `InputError`:** Definiert Fehler wie `SeatCreationFailed`, `CapabilityAdditionFailed`, `XkbConfigError`, `LibinputError`, `SeatNotFound`, `KeyboardHandleNotFound`, etc..
        
    - **`DesktopState` (teilweise):** Enthält `seat_state: SeatState<Self>`, `seats: HashMap<String, Seat<Self>>`, `active_seat_name: Option<String>`, `keyboards: HashMap<String, XkbKeyboardData>`.
        
    - **Implementierung `SeatHandler` für `DesktopState`:**
        - `KeyboardFocus = WlSurface`, `PointerFocus = WlSurface`, `TouchFocus = WlSurface`.
            
        - `seat_state()`: Gibt `&mut self.seat_state` zurück.
            
        - `focus_changed()`: Protokolliert Fokusänderung. Ruft `KeyboardHandle::leave()` für alten Fokus und `KeyboardHandle::enter()` für neuen Fokus auf (mit aktuellen Tasten und Modifikatoren). Aktualisiert interne Fenstermanagement-Zustände.
            
        - `cursor_image()`: Handhabt `CursorImageStatus::Hidden`, `CursorImageStatus::Surface` (prüft Rolle "cursor", ruft Puffer/Hotspot ab) und `CursorImageStatus::Named` (verwendet Cursor-Theming-Bibliothek). Weist Renderer an, den Cursor zu zeichnen/aktualisieren.
            
    - **Funktion `create_seat()`:** Ruft `state.seat_state.new_wl_seat()` auf. Fügt Fähigkeiten hinzu (`seat.add_keyboard()`, `seat.add_pointer()`, `seat.add_touch()`). Speichert `Seat`-Objekt und `XkbKeyboardData`. Setzt ggf. `active_seat_name`.
        

#### 3.2. Submodul: `system::input::libinput_handler`

Initialisiert und konfiguriert das `LibinputInputBackend` und verarbeitet dessen Ereignisse.

**Dateien:**

- `backend_config.rs`:
    - **Struktur `LibinputSessionInterface`:** Wrapper für `input::LibinputInterface` zum Öffnen/Schließen von Geräten über ein Session-Objekt (z.B. `DirectSession`, `LogindSession`).
        
    - **Funktion `init_libinput_backend()`:** Erstellt `libinput::Libinput`-Kontext mit `Libinput::new_from_path(session_interface)`. Weist dem Kontext einen Seat zu (`udev_assign_seat("seat0")`). Erstellt und gibt `LibinputInputBackend` zurück.
        
- `event_dispatcher.rs`:
    - **Zweck:** Verarbeitet `InputEvent<LibinputInputBackend>` und leitet an spezifische Handler weiter.
        
    - **Funktion `process_input_event()`:** Wird vom `calloop`-Callback aufgerufen. Ruft den aktiven Seat ab. Verarbeitet verschiedene `InputEvent`-Varianten (`Keyboard`, `PointerMotion`, `PointerButton`, `PointerAxis`, `TouchDown`, `TouchUp`, `TouchMotion`, `TouchFrame`, `TouchCancel`, `Gesture*`, `DeviceAdded`, `DeviceRemoved`) durch Aufruf entsprechender Handler-Funktionen in den jeweiligen Submodulen (`keyboard::key_event_translator`, `pointer::pointer_event_translator`, `touch::touch_event_translator`). Gestenereignisse werden initial nur protokolliert. Bei `DeviceAdded`/`DeviceRemoved` werden Seat-Fähigkeiten ggf. aktualisiert.
        

#### 3.3. Submodul: `system::input::keyboard`

Verwaltet XKB-Keymap und -Status für Tastaturen, übersetzt `KeyboardKeyEvent` und handhabt Tastaturfokus.

**Dateien:**

- `xkb_config.rs`:
    - **Struktur `XkbKeyboardData`:** Enthält `context: xkbcommon::xkb::Context`, `keymap: xkbcommon::xkb::Keymap`, `state: xkbcommon::xkb::State`, `repeat_timer: Option<calloop::TimerHandle>`, `repeat_info`, `focused_surface_on_seat`, `repeat_key_serial`.
        
    - **Funktion `new_xkb_keyboard_data()`:** Erstellt `xkbcommon::xkb::Context`, `xkbcommon::xkb::RuleNames` (aus `XkbConfig`), `xkbcommon::xkb::Keymap` und `xkbcommon::xkb::State`.
        
    - **Funktion `update_xkb_state_from_modifiers()`:** Ruft `xkb_state.update_mask()` auf.
        
- `key_event_translator.rs`:
    - **Funktion `handle_keyboard_key_event()`:** Ruft Keyboard-Handle und `XkbKeyboardData` ab. Aktualisiert `xkb_data.state` mit `update_key()`. Ruft `ModifiersState` von `xkb_data.state` ab und informiert `KeyboardHandle`. Sendet `keyboard_handle.input()` mit Keysym und UTF-8 (für `KeyState::Pressed`). Richtet Tastenwiederholung mittels `calloop::Timer` ein/bricht sie ab, basierend auf `keyboard_handle.repeat_info()`.
        
- `focus_handler_keyboard.rs`:
    - **Funktion `set_keyboard_focus()`:** Ruft Seat und Keyboard-Handle ab. Ermittelt alten Fokus. Sendet `keyboard_handle.leave()` für alten Fokus und `keyboard_handle.enter()` (mit gedrückten Tasten/Modifikatoren) für neuen Fokus. Aktualisiert `xkb_data.focused_surface_on_seat`. Ruft `keyboard_handle.set_focus()` auf.
        

#### 3.4. Submodul: `system::input::pointer`

Verarbeitet Zeigerereignisse, handhabt Zeigerfokus und Cursor-Aktualisierungen.

**Dateien:**

- `pointer_event_translator.rs`:
    - **Funktion `handle_pointer_motion_event()`:** Ruft Pointer-Handle ab. Aktualisiert globale Cursorposition. Bestimmt neuen Zeigerfokus basierend auf globaler Position und sichtbaren Toplevel-Oberflächen/Eingaberegionen (`find_surface_at_global_coords()`). Ruft `update_pointer_focus_and_send_motion()` auf. Aktualisiert Renderer-Cursorposition.
        
    - **Funktion `handle_pointer_motion_absolute_event()`:** Ähnlich wie Motion, aber verwendet absolute Koordinaten (`event.x_transformed()`, `event.y_transformed()`).
        
    - **Funktion `handle_pointer_button_event()`:** Ruft Pointer-Handle ab. Sendet `pointer_handle.button()`. Ändert ggf. Tastaturfokus (Click-to-Focus) und behandelt Fenstermanagement-Interaktionen (z.B. Start von Move/Resize-Grab).
        
    - **Funktion `handle_pointer_axis_event()`:** Ruft Pointer-Handle ab. Bestimmt Achsenquelle. Sendet `pointer_handle.axis()` für vertikales/horizontales Scrollen mit diskreten und kontinuierlichen Werten.
        
- `focus_handler_pointer.rs`:
    - **Funktion `update_pointer_focus_and_send_motion()`:** Ruft aktuellen Fokus vom Pointer-Handle ab. Sendet `pointer_handle.leave()` für alten Fokus und `pointer_handle.enter()` für neuen Fokus. Sendet `pointer_handle.motion()`, wenn neuer Fokus existiert.
        
- `cursor_updater.rs`:
    - **Zweck:** Enthält Logik von `SeatHandler::cursor_image`, ggf. Hilfsfunktionen für Cursor-Themen.
        

#### 3.5. Submodul: `system::input::touch`

Verarbeitet Touch-Ereignisse und handhabt Touch-Fokus. Fokus ist implizit im Oberflächenargument für `down`/`motion`. Zustand, welche Oberfläche von welchem Slot berührt wird, muss in `DesktopState` oder assoziierten Strukturen verwaltet werden.

**Dateien:**

- `touch_event_translator.rs`:
    - **Funktion `handle_touch_down_event()`:** Ruft Touch-Handle ab. Bestimmt fokussierte Oberfläche unter dem Touchpunkt. Sendet `touch_handle.down()`.
        
    - **Funktion `handle_touch_up_event()`:** Ruft Touch-Handle ab. Sendet `touch_handle.up()`.
        
    - **Funktion `handle_touch_motion_event()`:** Ruft Touch-Handle ab. Ruft fokussierte Oberfläche für den Touch-Slot ab. Transformiert Koordinaten. Sendet `touch_handle.motion()`.
        
    - **Funktion `handle_touch_frame_event()`:** Ruft Touch-Handle ab. Sendet `touch_handle.frame()`.
        
    - **Funktion `handle_touch_cancel_event()`:** Ruft Touch-Handle ab. Sendet `touch_handle.cancel()`.
        
- `focus_handler_touch.rs`:
    - **Zweck:** Verwaltet Touch-Fokus, ähnlich wie Zeigerfokus, aber pro Touchpunkt/Slot.
        

### 4. Modul: `system::dbus` (Interaktion mit System-D-Bus-Diensten)

Verantwortlich für die Kommunikation mit Standard-D-Bus-Diensten wie UPower, systemd-logind, NetworkManager, Freedesktop Secret Service und PolicyKit unter Verwendung der `zbus`-Bibliothek.

#### 4.1. Submodul: `system::dbus::error`

Definiert spezifische Fehlertypen für D-Bus-Interaktionen mittels `thiserror`.

**Datei:** `system/dbus/error.rs`

- **Enum `DBusError`:** Varianten wie `ConnectionFailed`, `MethodCallFailed`, `ProxyCreationFailed`, `SignalSubscriptionFailed`, `InvalidResponse`, `DataDeserializationError`, `PropertyAccessFailed`, `NameTaken`, `Timeout`.
    
- **Enum `BusType`:** `Session`, `System`.
    

#### 4.2. Submodul: `system::dbus::connection`

Stellt einen zentralen Manager für D-Bus-Verbindungen bereit.

**Datei:** `system/dbus/connection.rs`

- **Struktur `DBusConnectionManager`:**
    - Felder: `session_bus: tokio::sync::OnceCell<Arc<zbus::Connection>>`, `system_bus: tokio::sync::OnceCell<Arc<zbus::Connection>>` für verzögerte Initialisierung und Wiederverwendung.
        
    - Methoden: `new()`, `get_session_bus()` (verwendet `zbus::Connection::session().await`), `get_system_bus()` (verwendet `zbus::Connection::system().await`).
        

#### 4.3. Submodul: `system::dbus::upower_client` (oder `upower_interface`)

Interagiert mit `org.freedesktop.UPower` für Energieinformationen.

**Dateien:** `system/dbus/upower_client.rs` (oder `upower_interface/client.rs`), `system/dbus/upower_types.rs` (oder `upower_interface/types.rs`)

- **`upower_types.rs`:**
    - Enums: `PowerDeviceType`, `PowerState`, `PowerWarningLevel`, `PowerDeviceTechnology`. `TryFrom<u32>` für Enums implementieren.
        
    - Strukturen: `PowerDeviceDetails` (enthält Felder wie `object_path`, `vendor`, `model`, `device_type`, `state`, `percentage`, `time_to_empty`, `time_to_full`, etc.), `UPowerProperties` (oder `UPowerManagerProperties`) (`on_battery`, `lid_is_closed`, `lid_is_present`, `daemon_version`).
        
    - Internes Enum `UPowerEvent`: `DeviceAdded`, `DeviceRemoved`, `DeviceUpdated`, `ManagerPropertiesChanged`.
        
- **`upower_client.rs`:**
    - **Proxy-Definitionen (`#[zbus::proxy]`)**:
        - `UPowerManagerProxy` für `org.freedesktop.UPower`: Methoden `enumerate_devices`, `get_display_device`, `get_critical_action`; Properties `on_battery`, `lid_is_closed`, `lid_is_present`, `daemon_version`; Signale `device_added`, `device_removed`, `properties_changed`.
            
        - `UPowerDeviceProxy` für `org.freedesktop.UPower.Device`: Properties wie `type_`, `state`, `percentage`, `time_to_empty`, `time_to_full`, `is_present`, `icon_name`, etc.; Signal `properties_changed`.
            
    - **Struktur `UPowerClient`:**
        - Felder: `connection_manager: Arc<DBusConnectionManager>` (oder `connection: zbus::Connection`), `manager_proxy: Arc<UPowerManagerProxy>`, `devices: Arc<Mutex<HashMap<ObjectPath<'static>, PowerDeviceDetails>>>`, `display_device_path: Arc<Mutex<Option<ObjectPath<'static>>>>`, `manager_properties: Arc<Mutex<UPowerManagerProperties>>`, `internal_event_sender: tokio::sync::broadcast::Sender<UPowerEvent>`.
            
        - Methoden: `new()` (oder `connect_and_initialize()`), `get_manager_proxy()`, `get_device_proxy()`, `get_properties()` (für Manager), `enumerate_devices()`, `get_display_device_path()`, `get_device_details()`, `on_battery()`, `subscribe_device_added()`, `subscribe_device_removed()`, `subscribe_upower_properties_changed()` (Manager), `subscribe_device_properties_changed()`.
            
    - **Signalbehandlung:** Asynchrone Tasks (`tokio::spawn`) für Manager-Signale (`DeviceAdded`, `DeviceRemoved`, `PropertiesChanged`) und Geräte-Signale (`PropertiesChanged`). Aktualisiert interne Zustände (`devices`, `manager_properties`) und sendet `UPowerEvent` über den Broadcast-Kanal. PropertiesChanged-Signale sollten detailliert ausgewertet werden, um nur betroffene Felder zu aktualisieren.
        

#### 4.4. Submodul: `system::dbus::logind_client` (oder `logind_interface`)

Interagiert mit `org.freedesktop.login1` für Sitzungsmanagement und Systemereignisse.

**Dateien:** `system/dbus/logind_client.rs` (oder `logind_interface/client.rs`), `system/dbus/logind_types.rs` (oder `logind_interface/types.rs`)

- **`logind_types.rs`:**
    - Strukturen: `SessionInfo` (`id`, `user_id`, `user_name`, `seat_id`, `object_path`, `is_active`, `is_locked_hint`), `UserInfo`.
        
    - Enums: `SessionState`, `LogindEvent` (`PrepareForSleep { starting: bool }`, `ActiveSessionLocked`, `ActiveSessionUnlocked`, `ActiveSessionChanged`, `SessionListChanged`).
        
- **`logind_client.rs`:**
    - **Proxy-Definitionen (`#[zbus::proxy]`)**:
        - `LogindManagerProxy` für `org.freedesktop.login1.Manager`: Methoden `get_session`, `get_session_by_pid`, `get_user`, `list_sessions`, `lock_session`, `unlock_session`, `lock_sessions`, `unlock_sessions`, `inhibit`; Signale `session_new`, `session_removed`, `prepare_for_sleep`.
            
        - `LogindSessionProxy` für `org.freedesktop.login1.Session`: Properties `active`, `locked_hint`, `id`, `user`, `seat`; Signale `Lock`, `Unlock`, `PropertyChanged`.
            
    - **Struktur `LogindClient`:**
        - Felder: `connection: zbus::Connection`, `manager_proxy: Arc<LogindManagerProxy>`, `active_session_id: Arc<Mutex<Option<String>>>`, `active_session_path`, `active_session_proxy`, `sleep_inhibitor_lock: Arc<Mutex<Option<zbus::zvariant::OwnedFd>>>`, `internal_event_sender: tokio::sync::broadcast::Sender<LogindEvent>`.
            
        - Methoden: `new()` (oder `connect_and_initialize()`), `get_manager_proxy()`, `get_session_proxy()`, `list_sessions()` (konvertiert zu `Vec<SessionInfo>`), `get_session_details()`, `lock_session()`, `unlock_session()`, `lock_all_sessions()`, `unlock_all_sessions()`, `subscribe_session_new()`, `subscribe_session_removed()`, `subscribe_prepare_for_sleep()`, `subscribe_session_lock()`, `subscribe_session_unlock()`, `request_lock_active_session()`, `request_unlock_active_session()`, `subscribe_events()`, `release_sleep_inhibitor()`.
            
    - **Initialisierung:** Identifiziert die aktive Sitzung durch `ListSessions` und Prüfung der `Active`-Eigenschaft jeder Session.
        
    - **Signalbehandlung (`PrepareForSleep`):** Bei `start == true`, `inhibit()` aufrufen und FD speichern. `LogindEvent::PrepareForSleep { starting: true }` senden. Bei `start == false`, Inhibit-Lock freigeben (`drop(fd)`) und `LogindEvent::PrepareForSleep { starting: false }` senden. Die korrekte Freigabe des Inhibit-Locks ist kritisch.
        
    - **Signalbehandlung (`SessionNew`/`SessionRemoved`):** Aktualisiert interne Sitzungsliste, prüft auf Änderung der aktiven Sitzung und sendet `LogindEvent`.
        
    - **Signalbehandlung (`Lock`/`Unlock` oder `LockedHint` der aktiven Session):** Sendet `LogindEvent::ActiveSessionLocked/Unlocked`. Die DE ist i.d.R. selbst für den Sperrbildschirm zuständig; dieses Modul kann `LockedHint` überwachen.
        

#### 4.5. Submodul: `system::dbus::networkmanager_client`

Interagiert mit `org.freedesktop.NetworkManager` für Netzwerkinformationen.

**Dateien:** `system/dbus/networkmanager_client.rs`, `system/dbus/networkmanager_types.rs`

- **`networkmanager_types.rs`:**
    - Enums: `NetworkManagerState`, `NetworkDeviceType`, `NetworkConnectivityState`.
        
    - Strukturen: `NetworkDevice`, `ActiveConnection`, `NetworkManagerProperties`.
        
- **`networkmanager_client.rs`:**
    - **Proxy-Definitionen (`#[zbus::proxy]`)**: `NetworkManagerProxy`, `NMDeviceProxy`, `NMActiveConnectionProxy` mit relevanten Methoden, Properties und Signalen (z.B. `GetDevices`, `GetActiveConnections`, `StateChanged`, `DeviceAdded`, `DeviceRemoved`).
        
    - **Struktur `NetworkManagerClient`:** Enthält `connection_manager`, `manager_proxy_path`. Methoden `new()`, `get_manager_proxy()`, `get_device_proxy()`, `get_active_connection_proxy()`, `get_properties()`, `get_devices()` (ruft Details für jeden Pfad ab), `get_active_connections()`, `subscribe_state_changed()`, `subscribe_device_added()` (ruft Details für neuen Pfad ab), `subscribe_device_removed()`.
        
    - Reaktive Aktualisierung bei Signalempfang ist wichtig.
        

#### 4.6. Submodul: `system::dbus::secrets_client`

Interagiert mit `org.freedesktop.secrets` zum Speichern/Abrufen sensibler Daten (z.B. API-Schlüssel).

**Dateien:** `system/dbus/secrets_client.rs`, `system/dbus/secrets_types.rs`

- **`secrets_types.rs`:**
    - Strukturen: `Secret`, `SecretItemInfo`, `SecretCollectionInfo`.
        
    - Enum: `PromptCompletedResult`.
        
- **`secrets_client.rs`:**
    - **Proxy-Definitionen (`#[zbus::proxy]`)**: `SecretServiceProxy`, `SecretCollectionProxy`, `SecretItemProxy`, `SecretPromptProxy` mit relevanten Methoden, Properties und Signalen (z.B. `OpenSession`, `CreateCollection`, `SearchItems`, `Unlock`, `GetSecrets`, `CreateItem`, `GetSecret`, `Prompt`, `Completed`).
        
    - **Struktur `SecretsClient`:** Enthält `connection_manager`, `service_proxy_path`. Methoden `new()`, Proxies abrufen, `open_session()`, `get_default_collection()`, `store_secret()` (behandelt Prompt), `retrieve_secret()` (behandelt Unlock/Prompt), `search_items()`, `handle_prompt_if_needed()` (ruft `PromptProxy::Prompt()` mit `window_id` von UI-Schicht und wartet auf `Completed`-Signal).
        

#### 4.7. Submodul: `system::dbus::policykit_client`

Interagiert mit `org.freedesktop.PolicyKit1.Authority` zur Berechtigungsprüfung.

**Dateien:** `system/dbus/policykit_client.rs`, `system/dbus/policykit_types.rs`

- **`policykit_types.rs`:**
    - Bitflags-Struktur `PolicyKitCheckAuthFlags` (`AllowUserInteraction`, etc.).
        
    - Strukturen: `PolicyKitSubject` (`kind`, `details`), `PolicyKitAuthorizationResult` (`is_authorized`, `is_challenge`, `details`).
        
- **`policykit_client.rs`:**
    - **Proxy-Definition (`#[zbus::proxy]`)**: `PolicyKitAuthorityProxy` für `org.freedesktop.PolicyKit1.Authority`. Methode `CheckAuthorization`.
        
    - **Struktur `PolicyKitClient`:** Enthält `connection_manager`, `authority_proxy_path`. Methoden `new()`, `get_authority_proxy()`, `check_authorization()` (erstellt `PolicyKitSubject` mit PID des aktuellen Prozesses oder übergebenem PID, setzt Flags, ruft Proxy-Methode auf). Die korrekte Definition des `subject` ist sicherheitskritisch.
        

### 5. Modul: `system::outputs` (Verwaltung der Anzeigeausgänge)

Verantwortlich für Erkennung, Konfiguration und Verwaltung von Anzeigeausgängen (Monitoren). Implementiert serverseitige Logik für Wayland-Protokolle (`wl_output`, `xdg-output-unstable-v1`, `wlr-output-management-unstable-v1`, `wlr-output-power-management-unstable-v1`) unter Verwendung von Smithay-Abstraktionen.

#### 5.1. Submodul: `system::outputs::error`

Definiert spezifische Fehlertypen für Output-Operationen.

**Datei:** `system/outputs/error.rs`

- **Enum `OutputError`:** Varianten `DeviceAccessFailed`, `ProtocolError`, `ConfigurationConflict`, `ResourceCreationFailed`, `SmithayOutputError`, `OutputNotFound`, `ModeNotSupported`.
    

#### 5.2. Submodul: `system::outputs::output_device`

Kapselt Zustand und Logik eines einzelnen physischen Anzeigeausgangs.

**Datei:** `system/outputs/output_device.rs`

- **Struktur `OutputDevice`:**
    - Felder: `name: String`, `smithay_output: smithay::output::Output`, `wl_output_global: Option<GlobalId>`, `xdg_output_global: Option<GlobalId>`, `wlr_head_global: Option<GlobalId>`, `wlr_power_global: Option<GlobalId>`, `enabled: bool`, `current_dpms_state: DpmsState`, `pending_config_serial: Option<u32>` (für wlr-output-management).
        
    - Methoden: `new()` (initialisiert `smithay::output::Output`, fügt Modi hinzu, setzt initialen Zustand), `name()`, `smithay_output()`, `current_mode()`, `current_transform()`, `current_scale()`, `current_position()`, `is_enabled()`, `apply_state()` (ruft `smithay_output.change_current_state()`), `set_dpms_state()` (interagiert mit DRM, aktualisiert `current_dpms_state`), `supported_modes()`, `physical_properties()`, `add_mode()`, `set_preferred_mode()`, Methoden zum Setzen/Abrufen von Global-IDs, `destroy_globals()`.
        
- **Struktur `OutputDevicePendingState` (für wlr-output-management):** Felder `mode`, `position`, `transform`, `scale`, `enabled`, `adaptive_sync_enabled`.
    
- **Enum `DpmsState`:** `On`, `Standby`, `Suspend`, `Off`.
    

#### 5.3. Submodul: `system::outputs::manager` (oder `output_manager` für wlr-output-management)

Verwaltet eine Liste aller `OutputDevice`-Instanzen und behandelt Hotplug-Events.

**Datei:** `system/outputs/manager.rs` (oder `output_manager/manager_global.rs` und andere für wlr-output-management)

- **Struktur `OutputManager` (oder `OutputManagerModuleState` / `WlrOutputManagementState`):**
    - Felder: `outputs: HashMap<String, Arc<Mutex<OutputDevice>>>`, `udev_event_source_token: Option<RegistrationToken>`, `output_manager_global` (für wlr), `active_configurations` (für wlr), `compositor_output_serial` (für wlr) / `global_serial`.
        
    - Methoden: `new()`, `add_output()`, `remove_output()` (zerstört Globals), `find_output_by_name()`, `all_outputs()`, `handle_hotplug_event()` (erstellt/entfernt `OutputDevice`, ruft `output_device_created/removed_notifications` auf).
        
    - Hilfsmethoden `output_device_created_notifications()` und `output_device_removed_notifications()`: Erstellen/Zerstören Globals (`wl_output`, `zxdg_output_v1`, `zwlr_output_head_v1`) und benachrichtigen relevante Handler (WlrOutputManagementState, WlrOutputPowerManagementState).
        
- **Enum `HotplugEvent`:** `DeviceAdded` (mit Name, Pfad, Eigenschaften, Modi, etc.), `DeviceRemoved`.
    
- **Struktur `OutputConfigurationRequest` (für wlr-output-management):** `serial`, `client`, `pending_changes: HashMap<String, HeadChangeRequest>`, `config_resource`.
    
- **Struktur `HeadChangeRequest` (für wlr-output-management):** `mode`, `position`, `transform`, `scale`, `enabled`, `adaptive_sync_enabled`.
    
- **`wlr-output-management` spezifische UserData-Strukturen:** `WlrOutputManagerGlobalData`, `WlrOutputHeadGlobalData`, `WlrOutputModeGlobalData`, `WlrOutputConfigurationUserData`, `WlrOutputConfigurationHeadUserData`.
    

#### 5.4. Submodul: `system::outputs::wl_output_handler`

Handhabung des `wl_output`-Protokolls, meist durch Smithay's `Output`-Typ und `OutputHandler`-Trait.

**Datei:** Integration in globalen Compositor-Zustand und `system::outputs::manager.rs`.

- **Smithay Integration:** Globaler Compositor-Zustand implementiert `smithay::wayland::output::OutputHandler`. `smithay::delegate_output!` wird deklariert. Beim Hinzufügen eines Outputs wird `output_dev.smithay_output().create_global()` aufgerufen; `GlobalId` wird in `OutputDevice::wl_output_global` gespeichert.
    
- **Implementierung `OutputHandler`:**
    - `output_state()`: Gibt Referenz zum `OutputManagerState` des Compositors zurück.
        
    - `new_output()`: Kann Client-spezifischen Zustand initialisieren.
        
    - `output_destroyed()`: Wird bei Zerstörung eines `wl_output`-Globals aufgerufen.
        
    - Smithay sendet `geometry`, `mode`, `scale`, `done` Events automatisch bei `Output::change_current_state()`.
        

#### 5.5. Submodul: `system::outputs::wlr_output_management_handler` (oder in `system::outputs::output_manager`)

Implementiert `wlr-output-management-unstable-v1`.

**Dateien:** `system/outputs/wlr_output_management/*` oder integriert.

- **Zustandsstrukturen:** `WlrOutputManagementState` (enthält `OutputManager`-Referenz, `configurations`, `global_serial`). `OutputConfigurationRequest` (enthält `serial`, `client`, `pending_changes`, `config_resource`). `HeadChangeRequest` (enthält `mode`, `position`, etc.).
    
- **Smithay Integration:** Globaler Compositor-Zustand implementiert `GlobalDispatch` und `Dispatch` für `ZwlrOutputManagerV1`, `ZwlrOutputHeadV1`, `ZwlrOutputModeV1`, `ZwlrOutputConfigurationV1`, `ZwlrOutputConfigurationHeadV1`.
    
- **Initialisierung:** `WlrOutputManagementState` erstellen, `zwlr_output_manager_v1`-Global registrieren.
    
- **Anfragebehandlung `zwlr_output_manager_v1`:**
    - `bind`: Sendet aktuellen Zustand aller Outputs (Heads, Modi) und `done(serial)`.
        
    - `create_configuration()`: Erstellt `OutputConfigurationRequest`, speichert es. Sendet aktuellen Output-Zustand an das neue `config_resource`.
        
    - `stop()`: Zerstört das Manager-Global, sendet `finished`.
        
- **Anfragebehandlung `zwlr_output_configuration_head_v1`:** `enable()`, `disable()`, `set_mode()`, `set_custom_mode()`, `set_position()`, `set_transform()`, `set_scale()`, `set_adaptive_sync()` aktualisieren `HeadChangeRequest` im `OutputConfigurationRequest`. Prüfen auf `is_applied_or_tested`.
    
- **Anfragebehandlung `zwlr_output_configuration_v1`:**
    - `destroy()`: Verwirft Anfrage.
        
    - `apply()`: Prüft `serial` gegen `compositor_output_serial` (bei Mismatch -> `cancelled`). Validiert Änderungen. Versucht, Konfiguration auf `OutputDevice` anzuwenden (via `OutputManager` und `output.change_current_state()`). Bei Erfolg: `succeeded`, `compositor_output_serial` inkrementieren, alle Manager-Clients benachrichtigen. Bei Fehler: `failed`, ggf. Rollback.
        
    - `test()`: Validiert Konfiguration. Sendet `succeeded` oder `failed`.
        
- **Event-Generierung:** Bei Änderungen des Output-Zustands (Hotplug, erfolgreiches `apply`) müssen `head`, `mode`, `done(new_serial)` an alle Manager gesendet werden.
    
- **Fehlerbehandlung (`OutputManagerError`):** Varianten wie `InvalidWlOutput`, `AlreadyProcessed`, `SerialMismatch`, `UnknownOutput`, `TestFailed`, `ApplyFailed`, `Cancelled`, `ProtocolError`.
    

#### 5.6. Submodul: `system::outputs::wlr_output_power_management_handler` (oder `system::outputs::power_manager`)

Implementiert `wlr-output-power-management-unstable-v1` für Energiezustand von Monitoren.

**Dateien:** `system/outputs/wlr_output_power_management/*` oder integriert.

- **Zustandsstrukturen:** `WlrOutputPowerManagementState` (enthält `OutputManager`-Referenz, `active_controllers: HashMap<String, Resource<ZwlrOutputPowerV1>>`). `OutputPowerControlState` (enthält `wl_output_resource`, `compositor_output_name`, `current_mode: InternalPowerMode`).
    
- **Enum `InternalPowerMode`**: `On`, `Off`.
    
- **UserData-Strukturen:** `WlrOutputPowerManagerGlobalData`, `WlrOutputPowerControlUserData`.
    
- **Smithay Integration:** Globaler Compositor-Zustand implementiert `GlobalDispatch` und `Dispatch` für `ZwlrOutputPowerManagerV1`, `ZwlrOutputPowerV1`.
    
- **Initialisierung:** `WlrOutputPowerManagementState` erstellen, `zwlr_output_power_manager_v1`-Global registrieren.
    
- **Anfragebehandlung `zwlr_output_power_manager_v1`:**
    - `get_output_power()`: Ermittelt `OutputDevice`. Prüft auf exklusive Kontrolle (nur ein Controller pro Output). Speichert `ZwlrOutputPowerV1`-Ressource in `active_controllers`. Sendet initiales `mode`-Event an Ressource.
        
    - `destroy()`: Zerstört Manager-Global.
        
- **Anfragebehandlung `zwlr_output_power_v1`:**
    - `destroy()`: Entfernt Controller aus `active_controllers`.
        
    - `set_mode()`: Ermittelt `OutputDevice`. Konvertiert `mode` in `DpmsState`. Ruft `output_device.set_dpms_state()`. Sendet `mode(mode)` oder `failed`.
        
- **Event-Generierung:** Bei externer Änderung des DPMS-Zustands oder Entfernung eines Outputs, `mode` oder `failed` an aktive Controller senden.
    
- **Fehlerbehandlung (`OutputPowerError`):** Varianten `OutputDoesNotSupportPowerManagement`, `BackendSetModeFailed`, `OutputVanished`, `ExclusiveControlConflict`, `InvalidWlOutput`, `ProtocolError`.
    

#### 5.7. Submodul: `system::outputs::xdg_output_handler`

Implementiert `xdg-output-unstable-v1` für detaillierte logische Geometrieinformationen.

**Datei:** `system/outputs/xdg_output_handler.rs` oder integriert.

- **Smithay Integration:** Globaler Compositor-Zustand implementiert `GlobalDispatch` und `Dispatch` für `ZxdgOutputManagerV1`, `ZxdgOutputV1`. `XdgOutputManagerGlobalData`, `XdgOutputGlobalData` als UserData. Erstellung von Globals kann über `OutputManagerState::new_with_xdg_output()` oder manuell erfolgen.
    
- **Anfragebehandlung `zxdg_output_manager_v1`:**
    - `get_xdg_output()`: Ermittelt `OutputDevice`. Initialisiert `ZxdgOutputV1`-Ressource mit logischen Daten (`logical_position`, `logical_size`, `name`, `description`), sendet diese und `done`.
        
- **Event-Generierung:** Bei Änderungen an logischer Position, Größe, Name, Beschreibung eines `OutputDevice` müssen entsprechende Events an gebundene `zxdg_output_v1`-Instanzen gesendet werden, gefolgt von `done`. Dies wird i.d.R. von Smithay bei `Output::change_current_state()` gehandhabt.
    

### 6. Modul: `system::audio` (PipeWire Client-Integration)

Verantwortlich für alle audiobezogenen Operationen mittels PipeWire und `pipewire-rs`. Zentralisiert PipeWire-Interaktionslogik, ereignisgesteuerte Architektur. Eigene PipeWire MainLoop in dediziertem Thread, Kommunikation über asynchrone Kanäle (`tokio::sync::mpsc`, `tokio::sync::broadcast`). Unterscheidet Master-Lautstärke (Device-Routen) und Anwendungs-Stream-Lautstärke (Node-Props).

#### 6.1. Submodul: `system::audio::client`

Verwaltet Low-Level-Verbindung zu PipeWire, startet und unterhält PipeWire-MainLoop-Thread, leitet Befehle weiter und verteilt Ereignisse.

**Datei:** `system/audio/client.rs`

- **Struktur `PipeWireClient`:**
    - Felder: `core: Arc<pipewire::Core>`, `mainloop_thread_handle: Option<std::thread::JoinHandle<()>>`, `command_sender: tokio::sync::mpsc::Sender<AudioCommand>`, `internal_event_sender: tokio::sync::mpsc::Sender<InternalAudioEvent>`.
        
    - Methode `new()`: Initialisiert PipeWire (`pipewire::init()`), erstellt MPSC-Kanäle, startet `run_pipewire_loop` in neuem OS-Thread, wartet auf Initialisierungssignal vom Thread, speichert Handles und Sender.
        
    - Methode `get_command_sender()`: Gibt Klon des `command_sender` zurück.
        
- **Struktur `PipeWireLoopData` (für internen Thread-Zustand):** `core`, `registry: Arc<pipewire::Registry>`, `audio_event_broadcaster: tokio::sync::broadcast::Sender<AudioEvent>`, `command_receiver`, `internal_event_receiver`, `active_devices: HashMap<u32, MonitoredDevice>`, `active_streams: HashMap<u32, MonitoredStream>`, `default_sink_id`, `default_source_id`, `pipewire_mainloop`, `pipewire_context`, `metadata_proxy`, `metadata_listener_hook`.
    
- **Struktur `MonitoredDevice`:** `proxy: Arc<dyn ProxyT>`, `proxy_id`, `global_id`, `properties`, `param_listener_hook`, `info: AudioDevice`.
    
- **Struktur `MonitoredStream`:** `proxy: Arc<pipewire::node::Node>`, `proxy_id`, `global_id`, `properties`, `param_listener_hook`, `info: StreamInfo`.
    
- **Enum `InternalAudioEvent`:** `PwGlobalAdded`, `PwGlobalRemoved`, `PwNodeParamChanged`, `PwDeviceParamChanged`, `PwMetadataPropsChanged`.
    
- **Private Funktion `run_pipewire_loop()`:** Initialisiert `MainLoop`, `Context`, `Core`, `Registry`. Registriert Listener auf Registry (`global`, `global_remove` Callbacks senden `InternalAudioEvent`). Startet Timer, der `process_external_messages` aufruft. Ruft `mainloop.run()`.
    
    - `process_external_messages()`: Verarbeitet `AudioCommand` (ruft Control-Funktionen) und `InternalAudioEvent` (ruft Manager-Funktionen).
        

#### 6.2. Submodul: `system::audio::manager`

Verarbeitet PipeWire-Registry-Ereignisse, verwaltet `AudioDevice`/`StreamInfo`, behandelt Eigenschaftsänderungen.

**Datei:** `system/audio/manager.rs`

- **Funktion `handle_pipewire_global_added()`:** Loggt. Abhängig von `global.type_`:
    - `ObjectType::Node`: Extrahiert Properties. Unterscheidet Gerät (Sink/Source) von Anwendungsstream. Bindet `pipewire::node::Node`-Proxy. Ruft initiale Parameter ab (Lautstärke/Mute aus `SPA_PARAM_Props`). Erstellt `AudioDevice` oder `StreamInfo`. Registriert `param_changed`-Listener (sendet `InternalAudioEvent::PwNodeParamChanged`). Speichert in `active_devices` oder `active_streams`. Sendet `AudioEvent::DeviceAdded` oder `StreamAdded`.
        
    - `ObjectType::Device`: Extrahiert Properties. Bindet `pipewire::device::Device`-Proxy. Ruft initiale `SPA_PARAM_Route`-Parameter ab. Erstellt `AudioDevice`. Registriert `param_changed`-Listener (sendet `InternalAudioEvent::PwDeviceParamChanged`). Speichert in `active_devices`. Sendet `AudioEvent::DeviceAdded`.
        
    - `ObjectType::Metadata`: Wenn `metadata.name == "default"`. Bindet `pipewire::metadata::Metadata`-Proxy. Speichert Proxy. Parst Standardgeräte-IDs aus Properties. Registriert `props`-Listener (sendet `InternalAudioEvent::PwMetadataPropsChanged`). Sendet `AudioEvent::DefaultSink/SourceChanged`.
        
- **Funktion `handle_pipewire_global_removed()`:** Loggt. Entfernt `MonitoredDevice` oder `MonitoredStream` aus `active_devices`/`active_streams`. Listener werden automatisch entfernt. Sendet `AudioEvent::DeviceRemoved` oder `StreamRemoved`. Behandelt Entfernung des Metadata-Proxy.
    
- **Funktion `handle_node_param_changed()`:** Loggt. Sucht `MonitoredDevice`/`Stream`. Wenn `SPA_PARAM_Props` geändert: Parst Lautstärke/Mute aus Pod. Aktualisiert `AudioDevice`/`StreamInfo`. Sendet `AudioEvent::Device/StreamVolumeChanged/MuteChanged`.
    
- **Funktion `handle_device_param_changed()`:** Loggt. Sucht `MonitoredDevice`. Wenn `SPA_PARAM_Route` geändert: Parst Lautstärke/Mute der aktiven Route. Aktualisiert `AudioDevice`. Sendet `AudioEvent::DeviceVolumeChanged/MuteChanged`.
    
- **Funktion `handle_metadata_props_changed()`:** Loggt. Extrahiert neue Standard-Sink/Source-IDs aus Properties. Aktualisiert `is_default`-Flags der betroffenen `AudioDevice`-Instanzen und sendet `AudioEvent::DeviceUpdated`. Aktualisiert `default_sink_id`/`default_source_id`. Sendet `AudioEvent::DefaultSink/SourceChanged`.
    

#### 6.3. Submodul: `system::audio::control`

Implementiert Logik zum Senden von Steuerbefehlen an PipeWire.

**Datei:** `system/audio/control.rs`

- **Funktion `set_node_volume()`:** Sucht `MonitoredDevice`/`Stream`. Passt `volume.channel_volumes` gemäß `VolumeCurve` an. Erstellt `SPA_PARAM_Props`-Pod mit `channelVolumes` (via `spa_pod_utils`). Ruft `node_proxy.set_param()`.
    
- **Funktion `set_node_mute()`:** Sucht `MonitoredDevice`/`Stream`. Erstellt `SPA_PARAM_Props`-Pod mit `mute`. Ruft `node_proxy.set_param()`.
    
- **Funktion `set_device_volume()`:** Sucht `MonitoredDevice` (Device-Proxy). Passt `volume.channel_volumes` an. Ermittelt aktive Route. Erstellt `SPA_PARAM_Route`-Pod mit `channelVolumes` für die Route. Ruft `device_proxy.set_param()`.
    
- **Funktion `set_device_mute()`:** Sucht `MonitoredDevice`. Ermittelt aktive Route. Erstellt `SPA_PARAM_Route`-Pod mit `mute` für die Route. Ruft `device_proxy.set_param()`.
    
- **Funktion `set_default_device()`:** Prüft `metadata_proxy`. Bestimmt Property-Namen (`default.audio.sink/source`). Ruft `metadata_proxy.set_property()` mit globaler ID als String.
    

#### 6.4. Submodul: `system::audio::types`

Definiert primäre Datenstrukturen für das Audio-Modul.

**Datei:** `system/audio/types.rs`

- **Enums:** `AudioDeviceType` (`Sink`, `Source`, `Unknown`), `VolumeCurve` (`Linear`, `Cubic`), `AudioCommand` (siehe Tabelle), `AudioEvent` (siehe Tabelle).
    
- **Strukturen:**
    - `Volume`: Enthält `channel_volumes: Vec<f32>` (0.0-1.0).
        
    - `AudioDevice`: Enthält `id` (globale PW ID), `proxy_id`, `name`, `description`, `device_type`, `volume`, `is_muted`, `is_default`, `ports` (optional), `properties_spa` (optional), `is_hardware_device`, `api_name`.
        
    - `StreamInfo`: Enthält `id` (globale PW ID), `name`, `application_name`, `process_id`, `volume`, `is_muted`, `media_class`, `node_id_pw` (interne PW Node ID).
        

#### 6.5. Submodul: `system::audio::spa_pod_utils`

Hilfsfunktionen zur Erstellung von `pipewire::spa::Pod`-Objekten.

**Datei:** `system/audio/spa_pod_utils.rs`

- `build_volume_props_pod()`: Erstellt Pod für `SPA_PARAM_Props` mit `SPA_PROP_channelVolumes`.
    
- `build_mute_props_pod()`: Erstellt Pod für `SPA_PARAM_Props` mit `SPA_PROP_mute`.
    
- `build_route_volume_pod()`: Erstellt Pod für `SPA_PARAM_Route` mit `SPA_PARAM_ROUTE_index`, `SPA_PARAM_ROUTE_device` und verschachtelten `SPA_PARAM_ROUTE_props` (mit `SPA_PROP_channelVolumes`).
    
- `build_route_mute_pod()`: Analog für `SPA_PROP_mute` in Route-Props.
    
- `parse_props_volume_mute()`: Extrahiert `Volume` und `mute` aus `SPA_PARAM_Props`-Pod.
    
- `parse_route_props_volume_mute()`: Extrahiert `Volume` und `mute` aus `SPA_PARAM_ROUTE`-Pod (via dessen `SPA_PARAM_ROUTE_props`).
    

#### 6.6. Submodul: `system::audio::error`

Definiert `AudioError` mittels `thiserror`.

**Datei:** `system/audio/error.rs`

- **Enum `AudioError`:** Varianten wie `PipeWireInitFailed`, `MainLoopCreationFailed`, `ContextCreationFailed`, `CoreConnectionFailed`, `RegistryCreationFailed`, `PipeWireThreadPanicked`, `ProxyBindFailed`, `ParameterEnumerationFailed`, `SpaPodParseFailed`, `SpaPodBuildFailed`, `PipeWireCommandFailed`, `DeviceOrStreamNotFound`, `NoActiveRouteFound`, `MetadataProxyNotAvailable`, `InternalChannelSendError`, `InternalBroadcastSendError`.
    

### 7. Modul: `system::mcp` (Model Context Protocol Client)

Implementiert einen Client für das Model Context Protocol (MCP) zur Kommunikation mit lokalen oder Cloud-basierten MCP-Servern für KI-gestützte Funktionen. Kommunikation typischerweise über Stdio mit JSON-RPC. Verwendet `mcp_client_rs` Crate.

#### 7.1. Submodul: `system::mcp::client`

Kernlogik für Interaktion mit MCP-Server: Starten des Serverprozesses, Senden von Anfragen, Verarbeiten von Antworten/Benachrichtigungen.

**Datei:** `system/mcp/client.rs`

- **Struktur `McpClient`:**
    - Felder: `client_handle: Option<mcp_client_rs::client::Client>`, `server_process: Option<tokio::process::Child>`, `command_sender: tokio::sync::mpsc::Sender<McpCommand>`, `notification_broadcaster: tokio::sync::broadcast::Sender<McpNotification>`, `status_broadcaster: tokio::sync::broadcast::Sender<McpClientStatus>`, `request_id_counter: Arc<AtomicU64>`, `pending_requests: Arc<Mutex<HashMap<String, tokio::sync::oneshot::Sender<Result<serde_json::Value, McpError>>>>>`, `listen_task_handle: Option<tokio::task::JoinHandle<()>>`.
        
    - Methode `new()`: Erstellt MPSC-Kanal für `McpCommand`. Startet MCP-Serverprozess (`tokio::process::Command`) mit Pipes für Stdin/Stdout/Stderr. Erstellt `mcp_client_rs::transport::stdio::StdioTransport` und `mcp_client_rs::client::Client`. Startet `listen_task`. Sendet `Initialize`-Befehl und wartet auf Antwort. Sendet `McpClientStatus`-Updates.
        
    - Private Methode `listen_task()`: Lauscht auf `StdioTransportReceiver`. Deserialisiert Nachrichten. Bei Antwort: Sendet an passenden `oneshot::Sender` aus `pending_requests`. Bei Benachrichtigung: Sendet an `notification_broadcaster`. Behandelt Fehler und Verbindungsabbruch, sendet `McpClientStatus`.
        
    - Private Methode `send_request_generic()`: Generiert `request_id`. Speichert `oneshot::Sender` in `pending_requests`. Sendet JSON-RPC-Anfrage über `client_handle` (intern via `mcp_client_rs`). Wartet auf Antwort.
        
    - Öffentliche Methoden für MCP-Requests (`list_resources`, `read_resource`, `call_tool`): Rufen `send_request_generic` auf.
        
    - Methode `shutdown()`: Sendet Shutdown-Anfrage (falls unterstützt). Bricht `listen_task` ab. Beendet Serverprozess (`child.kill()`). Sendet `McpClientStatus::Disconnected`.
        
    - Methode `get_command_sender()`: Gibt Klon des `command_sender` zurück.
        
- **Struktur `McpServerConfig`:** `command`, `args`, `working_directory`.
    
- **Enums:** `McpClientStatus` (`Disconnected`, `Connecting`, `Connected`, `Error`), `McpCommand` (z.B. `Initialize`, `ListResources`, `ReadResource`, `CallTool`, `Shutdown`).
    

#### 7.2. Submodul: `system::mcp::transport`

Abstraktionsebene für Kommunikationstransport. Meist durch `mcp_client_rs::transport::stdio::StdioTransport` abgedeckt.

#### 7.3. Submodul: `system::mcp::types`

Definitionen für MCP-Anfragen, -Antworten, -Benachrichtigungen. Meist Re-Exporte oder dünne Wrapper um `mcp_client_rs::protocol` und `mcp_client_rs::types`.

**Datei:** `system/mcp/types.rs`

- **Struktur `McpNotification`:** `method: String`, `params: Option<serde_json::Value>`.
    

#### 7.4. Submodul: `system::mcp::error`

Definiert `McpError` mittels `thiserror`.

**Datei:** `system/mcp/error.rs`

- **Enum `McpError`:** Varianten `ServerSpawnFailed`, `TransportError` (aus `mcp_client_rs::Error`), `NotConnected`, `InitializationFailed`, `SerializationFailed` (aus `serde_json::Error`), `RequestTimeout`, `ServerReturnedError` (mit `code`, `message`, `data`), `UnexpectedResponse`, `ResponseChannelDropped`, `CommandSendError`.
    

### 8. Modul: `system::portals` (XDG Desktop Portals Backend)

Implementiert Backend-Logik für XDG Desktop Portals (`org.freedesktop.portal.FileChooser`, `org.freedesktop.portal.Screenshot`). Agiert als D-Bus-Dienst, der Anfragen von Client-Anwendungen bearbeitet. Nutzt `zbus` für D-Bus-Implementierung.

#### 8.1. Submodul: `system::portals::file_chooser`

Implementiert `org.freedesktop.portal.FileChooser` D-Bus-Interface.

**Datei:** `system/portals/file_chooser.rs`

- **Struktur `FileChooserPortal`:** Enthält `connection: Arc<zbus::Connection>`, `ui_event_sender: tokio::sync::mpsc::Sender<UiPortalCommand>` (zur Kommunikation mit UI-Schicht für Dialoganzeige).
    
- **D-Bus Interface Implementierung (`#[zbus::interface(name = "org.freedesktop.portal.FileChooser")]`)**:
    - `OpenFile()`: Extrahiert Optionen. Sendet Befehl an UI-Schicht, um Dateiauswahldialog anzuzeigen. Wartet auf Antwort (ausgewählte URIs). Gibt `Ok((0, {"uris": Value::from(vec!["file:///..."])})` oder Fehlercode/D-Bus-Fehler zurück.
        
    - `SaveFile()`: Ähnlich `OpenFile`, UI zeigt "Speichern"-Dialog. Gibt einzelnen URI zurück.
        
    - `SaveFiles()`: UI zeigt Ordnerauswahldialog. Backend konstruiert vollständige URIs aus ausgewähltem Ordner und übergebenen Dateinamen (`options["files"]`).
        
    - Antworten erfolgen direkt, nicht über separates Request-Objekt für einfache Fälle.
        

#### 8.2. Submodul: `system::portals::screenshot`

Implementiert `org.freedesktop.portal.Screenshot` D-Bus-Interface.

**Datei:** `system/portals/screenshot.rs`

- **Struktur `ScreenshotPortal`:** Enthält `connection: Arc<zbus::Connection>`, `compositor_command_sender: tokio::sync::mpsc::Sender<CompositorScreenshotCommand>` (zur Kommunikation mit Compositor).
    
- **D-Bus Interface Implementierung (`#[zbus::interface(name = "org.freedesktop.portal.Screenshot")]`)**:
    - `Screenshot()`: Extrahiert `interactive`. Sendet Befehl an Compositor, Screenshot zu erstellen (interaktiv oder gesamter Bildschirm). Compositor speichert temporär, gibt Pfad zurück. Konvertiert Pfad zu URI. Gibt `Ok((0, {"uri": Value::from(screenshot_uri)}))` zurück.
        
    - `PickColor()`: Sendet Befehl an Compositor, Farbauswahlmodus zu starten. Compositor meldet RGB-Werte. Gibt `Ok((0, {"color": Value::from((r,g,b))}))` zurück.
        
    - Property `version()`: Gibt implementierte Portal-Version zurück (z.B. 2).
        

#### 8.3. Submodul: `system::portals::common`

Gemeinsame Hilfsmittel und D-Bus-Handhabung.

**Datei:** `system/portals/common.rs`

- **Funktion `run_portal_service()`:**
    - Erstellt D-Bus-Verbindung zum Session-Bus. Registriert Dienstnamen `org.freedesktop.portal.Desktop`.
        
    - Erstellt Instanzen der Portal-Implementierungen (z.B. `FileChooserPortal`, `ScreenshotPortal`).
        
    - Registriert eine einzelne `DesktopPortal`-Struktur (die alle Portal-Interfaces implementiert oder delegiert) unter Pfad `/org/freedesktop/portal/desktop` beim `ObjectServer`.
        
    - Hält Dienst am Laufen.
        
- **Funktion `generate_request_handle()`:** Erzeugt eindeutigen Handle-String für Portal-Anfragen.
    
- **Hilfsstrukturen/Enums für UI/Compositor-Kommunikation:** `UiPortalCommand` (mit `OpenFileOptions`, `SaveFileOptions`), `PortalUiError`, `CompositorScreenshotCommand`.
    

#### 8.4. Submodul: `system::portals::error`

Definiert `PortalsError` mittels `thiserror`.

**Datei:** `system/portals/error.rs`

- **Enum `PortalsError`:** Varianten `DBusConnectionFailed` (aus `zbus::Error`), `DBusNameAcquisitionFailed`, `DBusInterfaceRegistrationFailed`, `UiCommandSendError`, `CompositorCommandSendError`, `UiInteractionFailed`, `CompositorInteractionFailed`, `InvalidOptions`.
    

### Schlussfolgerung

Diese detaillierte technische Spezifikation für die Systemschicht, basierend auf den bereitgestellten Dokumenten, deckt die Kernmodule Compositor, Input, D-Bus-Interaktion, Output-Management, Audio-Integration, MCP-Client und XDG-Desktop-Portal-Backends ab. Die konsequente Anwendung der Entwicklungsrichtlinien und die Nutzung moderner Technologien wie Rust, Smithay, PipeWire und D-Bus sollen eine robuste, performante und wartbare Systemschicht gewährleisten, die als solide Grundlage für die gesamte Desktop-Umgebung dient.

## Technische Spezifikation und Entwicklungsrichtlinien: Kernschicht

Diese Spezifikation beschreibt die fundamentalen Komponenten und Richtlinien für die Entwicklung der Kernschicht der Desktop-Umgebung. Die Kernschicht bildet das Fundament für alle darüberliegenden Schichten und umfasst Module für grundlegende Datentypen (`core::types`), Fehlerbehandlung (`core::errors`), Logging (`core::logging`), Konfigurationsmanagement (`core::config`) und allgemeine Hilfsfunktionen (`core::utils`).

### 1. Modul: `core::types` (Fundamentale Datentypen)

**1.1. Zweck und Verantwortlichkeit** Das Modul `core::types` definiert grundlegende, universell einsetzbare Datentypen, die von allen anderen Schichten und Modulen benötigt werden. Dazu gehören geometrische Primitive, Farbdarstellungen und allgemeine Enumerationen. Diese Typen sind reine Datenstrukturen ohne komplexe Geschäftslogik oder Abhängigkeiten zu höheren Schichten.

**1.2. Designphilosophie** Das Design folgt den Prinzipien der Modularität, Wiederverwendbarkeit und minimalen Kopplung. Typen sind generisch gehalten, wo sinnvoll (z.B. `Point<T>`, `Size<T>`, `Rect<T>`), um Flexibilität für verschiedene numerische Darstellungen zu ermöglichen (z.B. `i32` für Koordinaten, `f32` für Skalierungsfaktoren). Es besteht eine klare Trennung von Datenrepräsentation und Fehlerbehandlung.

**1.3. Ziel-Dateistruktur**

```
core/
└── src/
    ├── lib.rs         # Deklariert Kernmodule: pub mod types; pub mod errors; ...
    └── types/
        ├── mod.rs     # Deklariert und re-exportiert Typen aus geometry.rs, color.rs, etc.
        ├── geometry.rs # Enthält Point<T>, Size<T>, Rect<T>
        ├── color.rs    # Enthält Color
        └── enums.rs    # Enthält Orientation, etc.
```

**1.4. Spezifikation: Geometrische Primitive (`geometry.rs`)**

- **`Point<T>`**: Repräsentiert einen Punkt im 2D-Raum mit `x: T` und `y: T`.
    
    - Konstanten wie `ZERO_I32`, `ZERO_F32` etc..
        
    - Methoden: `new(x: T, y: T)`, `distance_squared(...)`, `distance(...)` (für Float-Typen), `manhattan_distance(...)`.
        
    - Basis-Constraints für `T`: `Copy + Debug + PartialEq + Default + Send + Sync + 'static`.
        
- **`Size<T>`**: Repräsentiert eine 2D-Dimension mit `width: T` und `height: T`.
    
    - Konstanten wie `ZERO_I32`, `ZERO_F32` etc..
        
    - Methoden: `new(width: T, height: T)`, `area()`, `is_empty()`, `is_valid()` (für nicht-negative Dimensionen).
        
    - Basis-Constraints für `T`: `Copy + Debug + PartialEq + Default + Send + Sync + 'static`. Die Invariante nicht-negativer Dimensionen wird durch `is_valid()` prüfbar gemacht, aber nicht durch den Typ erzwungen.
        
- **`Rect<T>`**: Repräsentiert ein 2D-Rechteck, definiert durch `origin: Point<T>` und `size: Size<T>`.
    
    - Konstanten wie `ZERO_I32`, `ZERO_F32` etc..
        
    - Methoden: `new(origin, size)`, `from_coords(x,y,width,height)`, Zugriffsmethoden (`x()`, `y()`, `width()`, `height()`, `top()`, `left()`, `bottom()`, `right()`), `center()`, `contains_point(...)`, `intersects(...)`, `intersection(...)`, `union(...)`, `translated(...)`, `scaled(...)`, `is_valid()`.
        
    - Basis-Constraints für `T`: `Copy + Debug + PartialEq + Default + Send + Sync + 'static`.
        
    - **Invariante**: Logisch sollten `width` und `height` nicht-negativ sein. Die Methode `is_valid()` wird bereitgestellt; Nutzer (besonders mit `T=i32`) sollten diese aufrufen. Die Verantwortung für das Melden eines Fehlers bei Verwendung eines ungültigen `Rect` liegt beim Aufrufer.
        
- **`RectInt`**: (aus einer anderen Quelldatei, aber thematisch passend) Repräsentiert ein achsenparalleles Rechteck mit ganzzahligen Koordinaten (`x: i32`, `y: i32`) und Dimensionen (`width: u32`, `height: u32`).
    
    - Methoden u.a. `new(...)`, `from_points(...)`, `top_left()`, `size()`, `right()`, `bottom()`, `contains_point(...)`, `intersects(...)`, `intersection(...)`, `union(...)`, `translate(...)`, `inflate(...)`, `is_empty()`.
        
    - Verwendet `saturating_add` / `saturating_sub` um Überläufe zu vermeiden.
        
    - Traits: `Debug`, `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Default`.
        

**1.5. Spezifikation: Farbdarstellung (`color.rs`)**

- **`Color` (RGBA)**: Repräsentiert eine Farbe mit `r: f32`, `g: f32`, `b: f32`, `a: f32` Komponenten im Bereich `[0.0, 1.0]`.
    
    - Konstanten: `TRANSPARENT`, `BLACK`, `WHITE`, `RED`, `GREEN`, `BLUE` etc..
        
    - Methoden: `new(r,g,b,a)` (klemmt Werte nicht automatisch, Aufruferverantwortung), `from_rgba8(r,g,b,a)`, `to_rgba8()`, `with_alpha(alpha)` (klemmt Alpha), `blend(background)`, `lighten(amount)`, `darken(amount)`.
        
    - `Default` wird manuell implementiert, um `Color::TRANSPARENT` zurückzugeben.
        
    - Soll `Serialize` und `Deserialize` implementieren, um als Hex-String (z.B. "#RRGGBBAA") in Konfigurationsdateien dargestellt zu werden. Dies erfordert eine `ColorParseError`-Behandlung.
        

**1.6. Spezifikation: Allgemeine Enumerationen (`enums.rs`)**

- **`Orientation`**: Repräsentiert eine horizontale oder vertikale Ausrichtung.
    
    - Varianten: `Horizontal`, `Vertical`.
        
    - Methoden: `toggle()`.
        
    - `Default` ist `Orientation::Horizontal`.
        

**1.7. Standard Trait Implementierungen** Alle Typen sollen grundlegende Traits wie `Debug`, `Clone`, `Copy` (wo anwendbar und `T` es unterstützt), `PartialEq`, `Default` (sinnvoll definiert), `Send` und `Sync` implementieren. `Eq` und `Hash` sind für Fließkommazahlen generell nicht geeignet.

**1.8. Modulabhängigkeiten** Minimale externe Abhängigkeiten: `std`. Optional `num-traits` (für erweiterte numerische Operationen) und `serde` (mit `derive`-Feature, falls Serialisierung direkt hier benötigt wird, aktuell aber eher in höheren Schichten vorgesehen).

### 2. Modul: `core::errors` (Fehlerbehandlung)

**2.1. Zweck und Geltungsbereich** Spezifiziert die verbindliche Strategie und Implementierung der Fehlerbehandlung innerhalb der Kernschicht. Ziel ist eine lückenlose, präzise Spezifikation für Entwickler.

**2.2. Grundlagen und Prinzipien**

- **Verwendung von `thiserror`**: Obligatorisch für die Definition von benutzerdefinierten Fehlertypen. `thiserror` reduziert Boilerplate-Code für `std::error::Error` und `std::fmt::Display`. Alle benutzerdefinierten Fehler-Enums in der Kernschicht müssen `thiserror::Error` ableiten.
    
- **`Result<T, E>` vs. `panic!`**: Strikte Trennung.
    
    - `Result<T, E>`: Standard für erwartete, potenziell behebbare Fehlerzustände (z.B. I/O-Fehler, ungültige Eingaben). Funktionen müssen `Result<T, E>` zurückgeben, wobei `E` typischerweise `CoreError` oder ein spezifischerer Modul-Fehler ist.
        
    - `panic!`: Ausschließlich für nicht behebbare Programmierfehler (Bugs), Verletzung von Vorbedingungen oder logisch unmögliche interne Zustände.
        
- **Umgang mit `.unwrap()` und `.expect()`**: In produktivem Code der Kernschicht strengstens zu vermeiden, da sie die strukturierte Fehlerbehandlung umgehen.
    
    - Ausnahme für `expect()`: Nur wenn ein `Err`- oder `None`-Zustand nachweislich einen Bug darstellt (interne Invariante verletzt). Die Nachricht muss dem "expect as precondition"-Stil folgen und erklären, _warum_ ein `Ok`- oder `Some`-Wert erwartet wurde.
        
- **Anforderungen an Fehlermeldungen (`#[error("...")]`)**:
    - Klarheit und Präzision, eindeutige Problembeschreibung.
        
    - Kontextinformationen durch eingebettete Feldwerte (`{field_name}`).
        
    - Zielgruppe: Entwickler (für Logging/Debugging).
        
    - Format: Knappe, klein geschriebene Sätze ohne abschließende Satzzeichen (Rust API Guidelines).
        
- **Keine sensiblen Daten in Fehlermeldungen**: Niemals Passwörter, API-Schlüssel, private Benutzerdaten etc. in Fehlermeldungen oder Kontextfeldern. Daten müssen maskiert, entfernt oder durch Platzhalter ersetzt werden.
    

**2.3. Strategie: Ein Fehler-Enum pro Modul** Jedes signifikante Modul innerhalb der Kernschicht (und höheren Schichten) definiert sein eigenes, spezifisches Fehler-Enum mit `thiserror`. Dies vermeidet Überladung des zentralen `CoreError` und adressiert `thiserror`-Einschränkungen bezüglich mehrdeutiger `#[from]`-Konvertierungen desselben Quelltyps.

**2.4. Definition des Basis-Fehlertyps: `CoreError`** Ein zentrales, öffentliches Enum `CoreError` in `core::errors` dient als primäre Schnittstelle für Fehler, die von öffentlichen Funktionen der Kernschicht propagiert werden. Es aggregiert allgemeine Fehlerarten und spezifischere Fehler aus Untermodulen (via `#[from]`).

- **Spezifikation `CoreError`** (Beispielvarianten):
    
    - `Io { path: PathBuf, #[source] source: std::io::Error }`
        
    - `Configuration(#[from] ConfigError)`
        
    - `Serialization { description: String }`
        
    - `InvalidId { invalid_id: String }`
        
    - `NotFound { resource_description: String }`
        
    - `Internal(String)` (sollte vermieden und durch spezifischere Varianten ersetzt werden)
        
    - `InitializationFailed { component: String, #[source] source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`
        
- **Ableitungen**: Mindestens `Debug` und `thiserror::Error`.
    
- **Fehlerverkettung (`source()`)**: Wird von `thiserror` automatisch für `#[source]` und `#[from]` annotierte Felder implementiert, um die Ursache zurückzuverfolgen.
    

**2.5. Modul-spezifische Fehler und Integration** Module definieren eigene Fehler-Enums (z.B. `ConfigError`, `UtilsError`) die `thiserror::Error` ableiten.

- **Integrationsmechanismus**: Eine dedizierte Variante in `CoreError`, die den Modul-Fehler kapselt und `#[from]` verwendet, ist der bevorzugte Weg. Beispiel: `Configuration(#[from] ConfigError)` in `CoreError`.
    
- Dies etabliert eine zweistufige Fehlerhierarchie.
    

**2.6. Fehlerkontext und Diagnose** Fehlervarianten sollen relevante Kontextinformationen als Felder enthalten (Dateipfade, ungültige Werte etc.).

**2.7. Implementierungsleitfaden für Entwickler (Fehlerdefinition und -behandlung)**

- **Neue Variante zu `CoreError` hinzufügen**: Prüfen, ob der Fehlerfall allgemeine Bedeutung hat oder besser in einem Modul-Fehler aufgehoben ist. Variante, `#[error]`-Meldung und Kontextfelder definieren. `#[source]` oder `#[from]` für Kapselung verwenden.
    
- **Neuen Modul-Fehler erstellen**: `errors.rs` im Modulverzeichnis anlegen. Enum definieren, `thiserror::Error` ableiten, Varianten und Meldungen spezifizieren. In `CoreError` über eine `#[from]`-Variante integrieren.
    
- **Verwendung des `?`-Operators**: Standard für Fehlerpropagation. Funktioniert nahtlos bei identischen Fehlertypen oder existierender `From`-Implementierung.
    
- **Fehler-Matching (`match`)**: Für spezifische Behandlung (Standardwerte, alternative Pfade, Anreicherung).
    
- **Umgang mit externen Crates**: Fehler von externen Bibliotheken müssen in einen Kernschicht-Fehlertyp (`CoreError` oder Modul-Fehler) gekapselt werden. Bevorzugt mit `#[from]` oder `#[source]` (manuelle Erzeugung via `.map_err()`).
    

### 3. Modul: `core::logging` (Logging-Infrastruktur)

**3.1. Grundlagen und Wahl von `tracing`** Die Desktop-Umgebung verwendet das `tracing`-Crate für strukturiertes Logging. `core::logging` stellt Initialisierungsroutinen bereit.

**3.2. `tracing` Framework Integrationsdetails**

- **Initialisierung**: Eine Funktion `initialize_logging(level_filter: tracing::LevelFilter, format: LogFormatEnum) -> Result<(), LoggingError>` wird früh im Anwendungsstart aufgerufen. `LogFormatEnum` könnte `PlainTextDevelopment`, `JsonProduction` definieren. `LoggingError` ist ein `thiserror`-Enum in `core::logging`.
    
- **Subscriber-Konfiguration**:
    - Entwicklung: `tracing_subscriber::fmt()` mit menschenlesbarer Ausgabe (`with_ansi(true)`, `with_target(true)`, `with_file(true)`, `with_line_number(true)`).
        
    - Release: Strukturiertes JSON-Format für Log-Aggregation und maschinelle Analyse (`tracing_subscriber::fmt::json()` oder `tracing-bunyan-formatter`).
        
- **Dynamische Log-Level-Änderungen**: Für zukünftige Erweiterungen berücksichtigen (z.B. via `tracing_subscriber::filter::EnvFilter` oder `RUST_LOG`).
    

**3.3. Standardisierte Log-Makros und `tracing::instrument` Verwendung**

- **Standard-Makros**: Direkte Verwendung von `trace!`, `debug!`, `info!`, `warn!`, `error!` ist verbindlich.
    
- **Log-Nachrichtenstruktur**: Prägnant und beschreibend. Schlüssel-Wert-Paare für strukturierte Daten: `tracing::info!(user_id = %user.id, " Nachricht")` (% für Display, ? für Debug). Fehler mit `error = ?err` loggen, um die Debug-Repräsentation (inkl. `source`-Kette) zu erfassen.
    
- **`#[tracing::instrument]` Verwendung**: Erzeugt Spans für Funktionen/Codeblöcke, gruppiert Log-Ereignisse.
    
    - Anwendung auf öffentliche API-Funktionen, I/O-Operationen, komplexe Berechnungen, abgeschlossene operative Einheiten.
        
    - `skip(...)` / `skip_all` für sensible/ausführliche Argumente.
        
    - `fields(...)` für spezifischen Kontext im Span.
        
    - `err` zur automatischen Fehlerprotokollierung bei `Result::Err`.
        
    - `level` zur Steuerung des Span-Levels.
        

**3.4. Logging von Fehlern** Jeder Fehler (`Result::Err`) sollte an seiner Ursprungsstelle oder einer geeigneten übergeordneten Stelle mit ausreichend Kontext geloggt werden, mindestens auf `ERROR`-Level (`tracing::error!`). Dies sollte typischerweise _vor_ der Propagation geschehen. Den Fehler selbst als strukturiertes Feld mitgeben: `error!(error = %core_err, "Nachricht")`.

**3.5. Log-Daten Sensibilität** Absolutes Verbot, sensible Daten (Passwörter, API-Schlüssel, PII etc.) im Klartext zu loggen. Daten redigieren oder auslassen. Vorsicht bei `Debug`-Implementierungen für Strukturen mit sensiblen Daten; ggf. manuelle Redaktion in `Debug` oder `skip_all` in `#[tracing::instrument]` verwenden.

### 4. Modul: `core::config` (Konfigurationsprimitive)

**4.1. Zweck** Definiert, wie grundlegende Konfigurationseinstellungen geladen, geparst und zugegriffen werden. Fokus auf Einfachheit, Robustheit.

**4.2. Konfigurationsdateiformat und Parsing-Logik**

- **Format**: TOML (Tom's Obvious, Minimal Language) wegen Lesbarkeit und einfacher Verarbeitung.
    
- **Parsing-Bibliothek**: `serde` in Verbindung mit `toml`-Crate (`serde_toml`).
    
- **Ladelogik**:
    - Definition von Standard-Konfigurationspfaden (z.B. systemweit, Entwicklungstests). XDG-Pfade für benutzerspezifische Konfigurationen in höheren Schichten berücksichtigen.
        
    - Eine Funktion wie `load_core_config(custom_path: Option<PathBuf>) -> Result<CoreConfig, ConfigError>` implementiert eine Suchreihenfolge, liest und deserialisiert die TOML-Datei.
        
    - Fehlerbehandlung mit `core::config::ConfigError` (definiert mit `thiserror`), Varianten wie `FileReadError`, `DeserializationError`, `NoConfigurationFileFound`.
        

**4.3. Konfigurationsdatenstrukturen (Ultra-Fein)**

- **`CoreConfig`-Struktur**: Eine primäre Struktur (z.B. `CoreConfig`) hält alle spezifischen Konfigurationen der Kernschicht.
    
    - Felder mit explizit definierten Typen.
        
    - Muss `serde::Deserialize` ableiten.
        
    - `#[serde(default = "path")]` oder `#[serde(default)]` umfassend verwenden für Standardwerte bei fehlenden Feldern.
        
    - `#[serde(deny_unknown_fields)]` erzwingen, um Tippfehler oder unbekannte Felder in Konfigurationsdateien zu verhindern.
        
- **Validierung**: Grundlegende Validierung durch Typen. Komplexere Validierungen nach Deserialisierung (z.B. via `TryFrom` Muster oder `validate()`-Methode). Für Kernschicht kann initiale Validierung auf `serde`-Fähigkeiten beschränkt sein.
    

**4.4. Konfigurationszugriffs-API**

- **Globaler Zugriff**: Geladene `CoreConfig`-Instanz threadsicher speichern, typischerweise mittels `once_cell::sync::OnceCell`.
    
    - `initialize_core_config(config: CoreConfig) -> Result<(), CoreConfig>` zum einmaligen Setzen.
        
    - `get_core_config() -> &'static CoreConfig` für den Zugriff; paniert, wenn nicht initialisiert (Programmierfehler).
        
- **Immutabilität**: Global zugängliche Konfiguration sollte nach Initialisierung unveränderlich sein. `CoreConfig` sollte `Clone` ableiten für Momentaufnahmen oder Tests.
    

### 5. Modul: `core::utils` (Allgemeine Hilfsfunktionen)

**5.1. Zweck** Beherbergt allgemeine Hilfsfunktionen und kleine, in sich geschlossene Utilities, die nicht in spezifischere Module passen, aber breit verwendet werden.

**5.2. Allgemeine Richtlinien**

- **Geltungsbereich**: Nur wirklich allgemeine Utilities.
- **Einfachheit**: Einfache Funktionen bevorzugen.
- **Reinheit**: Reine Funktionen (Ausgabe hängt nur von Eingabe ab, keine Seiteneffekte) bevorzugen.
- **Fehlerbehandlung**: Jede fehleranfällige Utility-Funktion gibt `Result<T, YourUtilError>` zurück, wobei `YourUtilError` mit `thiserror` im Utility-Submodul definiert wird.
- **Dokumentation**: Umfassende `rustdoc`-Kommentare mit Beispielen.
- **Tests**: Gründliche Unit-Tests.

### 6. Allgemeine Entwicklungsrichtlinien (Kernschicht)

**6.1. Dokumentation (`rustdoc`)** Alle öffentlichen Elemente (Module, Structs, Enums, Felder, Konstanten, Methoden) müssen `///`-Dokumentationskommentare haben.

- Modul-Level: Zweck des Moduls.
- Typ-Level: Zweck und Invarianten.
- Feld-Level: Bedeutung des Feldes.
- Methoden-Level: Was die Methode tut, Parameter, Rückgabewerte, mögliche Panics (idealerweise keine außer in Tests), Vor-/Nachbedingungen, Algorithmen.
- `# Examples`-Abschnitte verwenden.
- Strikte Einhaltung der Rust API Guidelines.
- `cargo doc --open` zur Überprüfung.

**6.2. Unit-Testing**

- Ein `#[cfg(test)]`-Modul innerhalb jeder Implementierungsdatei.
- Tests für Konstruktoren, Konstanten, Methodenlogik, Grenzfälle, Trait-Implementierungen, Invariantenprüfungen.
- Anstreben einer hohen Testabdeckung.

**6.3. Immutabilität und Stabilität** Die API der Kernschicht sollte nach Stabilisierung als äußerst stabil behandelt werden. Änderungen haben weitreichende Auswirkungen. Komponenten sind so konzipiert, dass sie `Send + Sync` sind, wo sinnvoll, für Multithreading.

**6.4. Schichtübergreifende Integrationsrichtlinien**

- **Fehlerbehandlung**: Höhere Schichten definieren eigene `thiserror`-Enums. Fehler aus der Kernschicht werden behandelt oder via `?` propagiert (ggf. mit `#[from]` in eigene Fehlertypen konvertiert), Fehlerkette (`source()`) muss erhalten bleiben.
- **Logging**: Alle Schichten nutzen `tracing`-Makros. `core::logging::initialize_logging()` wird vom Hauptbinary aufgerufen. Einhaltung von Log-Leveln und Datensensibilität ist zwingend.
- **Konfiguration**: Höhere Schichten können eigene Konfigs definieren. Zugriff auf Kern-Konfig via `core::config::get_core_config()`. Kern-Konfig nicht zur Laufzeit modifizieren.
- **Typen und Utilities**: Kerndatentypen und -utilities direkt verwenden. Bei Spezialisierung Komposition oder Newtype-Wrapper um Kerntypen in Betracht ziehen.

Diese Spezifikation legt den Grundstein für eine robuste, wartbare und performante Kernschicht. Die disziplinierte Einhaltung dieser Richtlinien ist für den Erfolg des Projekts entscheidend.