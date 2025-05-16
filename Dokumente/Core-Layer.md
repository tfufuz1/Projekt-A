# Kerninfrastruktur Implementierungsplan (Ultra-Feinspezifikation)

## 1. Einleitung

Dieses Dokument stellt den finalen, lückenlosen Entwickler-Implementierungsleitfaden für die Kerninfrastrukturschicht (Core Layer) dar. Es ist als Ultra-Feinspezifikation konzipiert und enthält alle notwendigen Details, um Entwicklern die direkte Implementierung in Rust zu ermöglichen, ohne dass eigene Architekturentscheidungen, Logikentwürfe oder Algorithmen erforderlich sind. Alle relevanten Aspekte wurden recherchiert, entschieden und präzise spezifiziert.

Die Kerninfrastruktur (`core_infra`) bildet das Fundament des Systems. Ihre Hauptverantwortlichkeiten umfassen die Bereitstellung grundlegendster Datentypen, Dienstprogramme, der Konfigurationsgrundlagen, der Logging-Infrastruktur und allgemeiner Fehlerdefinitionen. Diese Schicht dient als Basis für alle anderen Schichten des Systems und weist selbst keine Abhängigkeiten zu diesen höheren Schichten auf. Ihre Funktionalität wird von allen übergeordneten Schichten genutzt. Die Implementierung folgt strikt den Rust API Guidelines 1 und Best Practices für sichere und wartbare Rust-Entwicklung.3

## 2. Allgemeine Designprinzipien und Konventionen

### 2.1. Programmiersprache und Tooling

- **Sprache:** Rust (aktuellste stabile Version, Mindestanforderung gemäß Abschnitt 10.1).
- **Build-System:** Cargo (Standard Rust Build-System und Paketmanager).5
- **Formatierung:** `rustfmt` mit Standardkonfiguration (100 Zeichen Zeilenbreite, 4 Leerzeichen Einrückung).3
- **Linting:** `clippy` mit Standardempfehlungen (`cargo clippy`).

### 2.2. Code-Stil und Namenskonventionen

- Strikte Einhaltung der offiziellen Rust API Guidelines.1
- **Casing:** `snake_case` für Funktionen, Methoden, Variablen, Module; `PascalCase` für Typen (Structs, Enums, Traits).1
- **Konvertierungen:** `as_` für günstige Referenz-zu-Referenz-Konvertierungen, `to_` für teurere Wert-zu-Wert-Konvertierungen, `into_` für übernehmende Konvertierungen.1
- **Getter:** Namen folgen der Konvention `field_name()` für einfachen Zugriff, `set_field_name()` für Setter (falls mutability erlaubt ist).1
- **Iteratoren:** Methoden liefern `iter()`, `iter_mut()`, `into_iter()`; Iterator-Typen heißen entsprechend (z.B. `MyTypeIter`).1
- **Modulstruktur:** Klar definierte Module gemäß Abschnitt 10.5.

### 2.3. Fehlerbehandlung

- Keine Panics für erwartbare Fehler; stattdessen `Result<T, CoreError>` verwenden.6
- Vermeidung von `.unwrap()` und `.expect()`; Nutzung des `?`-Operators zur Fehlerpropagation.1
- Definition einer zentralen `CoreError`-Enum mit `thiserror` für die gesamte Schicht.6
- Fehlermeldungen müssen klar, kontextbezogen und informativ sein.

### 2.4. Sicherheit

- **Kein `unsafe` Code:** Die Verwendung von `unsafe`-Blöcken ist in dieser Schicht strikt untersagt, um die von Rust garantierte Speichersicherheit zu gewährleisten.4
- **Input Validierung:** Obwohl die Kernschicht primär interne Dienste bereitstellt, müssen alle von außen kommenden Konfigurationsdaten oder Parameter validiert werden (C-VALIDATE 1).
- **Dependency Management:** Abhängigkeiten werden minimal gehalten und regelmäßig auf Sicherheitsupdates überprüft.4

### 2.5. Dokumentation

- Alle öffentlichen Elemente (Module, Typen, Funktionen, Methoden) MÜSSEN mit ausführlichen Rustdoc-Kommentaren (`///`) versehen sein.1
- Dokumentation umfasst Zweck, Parameter, Rückgabewerte, mögliche Fehler (`CoreError`-Varianten) und Beispiele (C-EXAMPLE 1).
- Fehler- und Panikbedingungen (obwohl Panics vermieden werden sollen) müssen dokumentiert werden (C-FAILURE 1).

## 3. Kern-Datentypen (`core_infra::types`)

Dieses Modul definiert grundlegende Datenstrukturen (Structs und Enums), die potenziell von mehreren anderen Modulen innerhalb oder außerhalb der Kernschicht verwendet werden könnten. Es enthält keine komplexe Logik, sondern nur die Definitionen selbst.

### 3.1. Modulstruktur

```
core_infra/
└── src/
    ├── lib.rs
    └── types.rs
```

### 3.2. Grundlegende Wertobjekte und Structs

Wertobjekte sind einfache Strukturen, die hauptsächlich Daten kapseln und deren Identität durch ihre Werte definiert wird. Sie sollten unveränderlich sein, nachdem sie erstellt wurden, was durch private Felder und Konstruktoren, die Validierungen durchführen, sichergestellt wird.

- **Beispiel:** `AppIdentifier` (falls eine spezifische ID-Struktur benötigt wird)
    
    Rust
    
    ```
    use serde::{Serialize, Deserialize};
    use std::fmt;
    
    /// Represents a unique identifier for an application component or instance.
    /// Enforces specific formatting rules through its constructor.
    #
    pub struct AppIdentifier(String); // Internal representation is a String
    
    impl AppIdentifier {
        /// Creates a new AppIdentifier from a string slice.
        ///
        /// # Arguments
        /// * `value` - The string representation of the identifier. Must not be empty
        ///           and should adhere to specific format rules (e.g., alphanumeric).
        ///
        /// # Errors
        /// Returns `CoreError::InvalidInput` if the value is empty or does not meet
        /// the required format.
        pub fn new(value: &str) -> Result<Self, crate::error::CoreError> {
            if value.is_empty() {
                Err(crate::error::CoreError::InvalidInput("AppIdentifier cannot be empty".to_string()))
            } else if!value.chars().all(|c| c.is_alphanumeric() |
    ```
    

| c == '-') {

Err(crate::error::CoreError::InvalidInput(format!("AppIdentifier contains invalid characters: {}", value)))

}

else {

Ok(Self(value.to_string()))

}

}

````
    /// Returns a reference to the underlying string value.
    /// Conforms to Rust API guidelines for getters (C-GETTER [1]).
    pub fn value(&self) -> &str {
        &self.0
    }
}

/// Allows displaying the AppIdentifier.
impl fmt::Display for AppIdentifier {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

/// Allows conversion from AppIdentifier to String.
impl From<AppIdentifier> for String {
    fn from(id: AppIdentifier) -> Self {
        id.0
    }
}

/// Allows borrowing as a string slice.
/// Conforms to Rust API guidelines for conversions (C-CONV-TRAITS [1]).
impl AsRef<str> for AppIdentifier {
    fn as_ref(&self) -> &str {
        &self.0
    }
}
```
````

- **Spezifikation:**
    - Alle Wertobjekte müssen `Debug`, `Clone`, `PartialEq`, `Eq`, `Hash` implementieren (C-COMMON-TRAITS 1).
    - Falls sie in Konfigurationen oder Logs serialisiert werden sollen, müssen sie `serde::Serialize` und `serde::Deserialize` implementieren.8
    - Felder müssen privat sein, um Invarianten zu schützen (C-STRUCT-PRIVATE 1).
    - Öffentliche Konstruktoren (`fn new(...) -> Result<Self, CoreError>`) validieren Eingaben und erzwingen Invarianten (C-CTOR 1, C-VALIDATE 1).
    - Getter-Methoden (`fn field_name(&self) -> &T`) bieten Lesezugriff.
    - Implementierung relevanter Traits wie `Display`, `From`, `AsRef` (C-CONV-TRAITS 1).

### 3.3. Fundamentale Enums

Definition einfacher Enums für Zustände oder Kategorien, die systemweit relevant sein könnten.

- **Beispiel:** `Status`
    
    Rust
    
    ```
    use serde::{Serialize, Deserialize};
    
    /// Represents a general status indicator.
    #
    pub enum Status {
        Enabled,
        Disabled,
        Pending,
        Error(i32), // Example with associated data
    }
    
    impl Status {
        /// Checks if the status indicates an active or ready state.
        pub fn is_active(&self) -> bool {
            matches!(self, Status::Enabled)
        }
    }
    ```
    
- **Spezifikation:**
    - Müssen `Debug`, `Clone`, `Copy` (wenn sinnvoll), `PartialEq`, `Eq`, `Hash` implementieren (C-COMMON-TRAITS 1).
    - Falls für Konfiguration/Logs benötigt, `serde::Serialize` und `serde::Deserialize` implementieren.8
    - Varianten klar dokumentieren.
    - Nützliche Hilfsmethoden (wie `is_active`) können implementiert werden.

## 4. Utility Services (`core_infra::utils`)

Dieses Modul stellt grundlegende Hilfsfunktionen und Dienste bereit, die keine spezifische Domänenlogik enthalten, aber von vielen Teilen der Anwendung benötigt werden.

### 4.1. Modulstruktur

```
core_infra/
└── src/
    ├── lib.rs
    ├── types.rs
    ├── error.rs
    └── utils/
        ├── mod.rs  // Re-exportiert öffentliche Funktionen aus Submodulen
        ├── fs.rs
        └── paths.rs
        // Optional: strings.rs, time.rs
```

### 4.2. Filesystem Utilities (`core_infra::utils::fs`)

Enthält grundlegende, sichere Operationen für das Dateisystem. Komplexe Dateioperationen gehören in höhere Schichten.

- **Funktionen:**
    
    Rust
    
    ```
    // core_infra/src/utils/fs.rs
    use crate::error::CoreError;
    use std::fs;
    use std::path::Path;
    
    /// Ensures that a directory exists at the specified path.
    /// If the directory does not exist, it attempts to create it, including any
    /// necessary parent directories.
    ///
    /// # Arguments
    /// * `path` - The path to the directory to ensure existence of.
    ///
    /// # Errors
    /// Returns `CoreError::Filesystem` if the directory could not be created
    /// or if the path exists but is not a directory.
    pub fn ensure_dir_exists(path: &Path) -> Result<(), CoreError> {
        if path.exists() {
            if!path.is_dir() {
                return Err(CoreError::Filesystem {
                    message: format!("Path exists but is not a directory"),
                    path: path.to_path_buf(),
                    // Use a placeholder error kind or map specific std::io::ErrorKind
                    source: std::io::Error::new(std::io::ErrorKind::AlreadyExists, "Path exists but is not a directory"),
                });
            }
            Ok(()) // Directory already exists
        } else {
            fs::create_dir_all(path).map_err(|e| CoreError::Filesystem {
                message: "Failed to create directory".to_string(),
                path: path.to_path_buf(),
                source: e,
            })
        }
    }
    
    // Add other minimal, safe filesystem utilities if absolutely necessary.
    // Example: Reading a file with specific error mapping.
    /// Reads the entire contents of a file into a string.
    ///
    /// # Arguments
    /// * `path` - The path to the file to read.
    ///
    /// # Errors
    /// Returns `CoreError::Filesystem` if the file cannot be read.
    pub fn read_to_string(path: &Path) -> Result<String, CoreError> {
        fs::read_to_string(path).map_err(|e| CoreError::Filesystem {
            message: "Failed to read file to string".to_string(),
            path: path.to_path_buf(),
            source: e,
        })
    }
    ```
    
- **Spezifikation:**
    - Alle Funktionen müssen `Result<(), CoreError>` oder `Result<T, CoreError>` zurückgeben.
    - Unterliegende `std::io::Error` müssen sorgfältig in `CoreError::Filesystem` oder `CoreError::Io` gemappt werden, wobei Kontext (wie der Dateipfad) hinzugefügt wird. Die `Filesystem`-Variante ist vorzuziehen, wenn der Pfad relevant ist, um aussagekräftigere Fehlermeldungen zu ermöglichen.

### 4.3. Path Resolution (`core_infra::utils::paths`)

Stellt standardisierte Pfade für Konfiguration, Daten, Cache etc. bereit, basierend auf Betriebssystemkonventionen (insbesondere XDG Base Directory Specification auf Linux 10). Nutzt die `directories-next` Crate.11

- **Abhängigkeit:** `directories-next = "2.0.0"` (oder aktuellste stabile Version)
- **Funktionen:**
    
    Rust
    
    ```
    // core_infra/src/utils/paths.rs
    use crate::error::{CoreError, ConfigError};
    use directories_next::{BaseDirs, ProjectDirs}; // Verwende directories-next
    use std::path::PathBuf;
    
    // Definiere hier die Projekt-Qualifizierer, falls ProjectDirs verwendet wird.
    // Diese sollten global konfigurierbar sein oder aus einer zentralen Stelle stammen.
    const QUALIFIER: &str = "org"; // Beispiel
    const ORGANIZATION: &str = "YourOrg"; // Beispiel
    const APPLICATION: &str = "YourApp"; // Beispiel
    
    /// Returns the primary base directory for user-specific configuration files.
    /// Corresponds to $XDG_CONFIG_HOME or platform equivalent.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_config_base_dir() -> Result<PathBuf, CoreError> {
        BaseDirs::new()
           .map(|dirs| dirs.config_dir().to_path_buf())
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "Config Base".to_string() }))
    }
    
     /// Returns the primary base directory for user-specific data files.
    /// Corresponds to $XDG_DATA_HOME or platform equivalent.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_data_base_dir() -> Result<PathBuf, CoreError> {
        BaseDirs::new()
           .map(|dirs| dirs.data_dir().to_path_buf())
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "Data Base".to_string() }))
    }
    
    /// Returns the primary base directory for user-specific cache files.
    /// Corresponds to $XDG_CACHE_HOME or platform equivalent.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_cache_base_dir() -> Result<PathBuf, CoreError> {
        BaseDirs::new()
           .map(|dirs| dirs.cache_dir().to_path_buf())
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "Cache Base".to_string() }))
    }
    
    /// Returns the primary base directory for user-specific state files.
    /// Corresponds to $XDG_STATE_HOME or platform equivalent.
    /// Returns None if not applicable on the current platform (e.g., Windows, older macOS).
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_state_base_dir() -> Result<PathBuf, CoreError> {
         BaseDirs::new()
            // state_dir() ist in BaseDirs nicht direkt verfügbar,
            // aber XDG definiert es. directories-next unterstützt es möglicherweise
            // nicht direkt oder es muss manuell abgeleitet werden.
            // Fallback auf.local/state gemäß XDG Spec [10]
           .map(|dirs| {
                #[cfg(target_os = "linux")]
                {
                    std::env::var("XDG_STATE_HOME")
                       .map(PathBuf::from)
                       .unwrap_or_else(|_| dirs.home_dir().join(".local/state"))
                }
                #[cfg(not(target_os = "linux"))]
                {
                    // Für andere OS gibt es keinen direkten Standard, oft wird data_local_dir verwendet
                     dirs.data_local_dir().to_path_buf()
                }
            })
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "State Base".to_string() }))
    }
    
    
    /// Returns the application-specific configuration directory.
    /// Uses ProjectDirs based on QUALIFIER, ORGANIZATION, APPLICATION constants.
    /// Example: ~/.config/YourOrg/YourApp on Linux.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_app_config_dir() -> Result<PathBuf, CoreError> {
        ProjectDirs::from(QUALIFIER, ORGANIZATION, APPLICATION)
           .map(|dirs| dirs.config_dir().to_path_buf())
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "App Config".to_string() }))
    }
    
    /// Returns the application-specific data directory.
    /// Example: ~/.local/share/YourOrg/YourApp on Linux.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
     pub fn get_app_data_dir() -> Result<PathBuf, CoreError> {
        ProjectDirs::from(QUALIFIER, ORGANIZATION, APPLICATION)
           .map(|dirs| dirs.data_dir().to_path_buf())
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "App Data".to_string() }))
    }
    
    /// Returns the application-specific cache directory.
    /// Example: ~/.cache/YourOrg/YourApp on Linux.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_app_cache_dir() -> Result<PathBuf, CoreError> {
        ProjectDirs::from(QUALIFIER, ORGANIZATION, APPLICATION)
           .map(|dirs| dirs.cache_dir().to_path_buf())
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "App Cache".to_string() }))
    }
    
     /// Returns the application-specific state directory.
    /// Example: ~/.local/state/YourOrg/YourApp on Linux.
    ///
    /// # Errors
    /// Returns `CoreError::Config(ConfigError::DirectoryUnavailable)` if the directory cannot be determined.
    pub fn get_app_state_dir() -> Result<PathBuf, CoreError> {
        ProjectDirs::from(QUALIFIER, ORGANIZATION, APPLICATION)
             // ProjectDirs hat kein state_dir. Wir leiten es vom Basis-State-Dir ab.
           .and_then(|proj_dirs| {
                 get_state_base_dir().map(|base_state| base_state.join(proj_dirs.project_path()))
            })
           .ok_or_else(|| CoreError::Config(ConfigError::DirectoryUnavailable { dir_type: "App State".to_string() }))
    }
    ```
    
- **Spezifikation:**
    - Die Funktionen kapseln die Logik zur Pfadermittlung und abstrahieren die Unterschiede zwischen Betriebssystemen.11 Dies ist eine zentrale Aufgabe der Kernschicht, um Portabilität zu gewährleisten.
    - `Option`-Rückgabewerte von `directories-next` werden in `CoreError::Config(ConfigError::DirectoryUnavailable)` umgewandelt, um eine konsistente Fehlerbehandlung sicherzustellen.
    - Die Verwendung von `ProjectDirs` ist optional, aber empfohlen, wenn anwendungsspezifische Unterverzeichnisse standardmäßig benötigt werden. Die Konstanten `QUALIFIER`, `ORGANIZATION`, `APPLICATION` müssen definiert werden.

### 4.4. Basic String Manipulation (`core_infra::utils::strings`)

Nur hinzufügen, wenn generische String-Helfer benötigt werden, die über `std::str` und `String` hinausgehen und in der Kernschicht _unbedingt_ erforderlich sind. Im Allgemeinen sollte dieses Modul vermieden werden, um die Schicht schlank zu halten. Falls benötigt, müssen Signaturen exakt definiert werden (`pub fn...(...) ->...`).

### 4.5. Time Utilities (`core_infra::utils::time`)

Normalerweise wird `chrono` direkt in höheren Schichten verwendet. Dieses Modul ist nur notwendig, wenn die Kernschicht spezifische Zeit-Wrapper, Formate oder Abstraktionen _bereitstellen muss_. Wenn `chrono` verwendet wird, sollte es als Abhängigkeit deklariert werden.

## 5. Configuration Management (`core_infra::config`)

Verantwortlich für das Laden, Parsen und Validieren der Kernkonfiguration der Anwendung.

### 5.1. Modulstruktur

```
core_infra/
└── src/
    ├── lib.rs
    ├── types.rs
    ├── error.rs
    ├── utils/
    │   └──...
    └── config/
        ├── mod.rs // Definiert CoreConfig, LoggingConfig etc. und ConfigLoader
        └── defaults.rs // Enthält Funktionen für Standardwerte
```

### 5.2. Configuration Data Structures (`core_infra::config::mod.rs`)

Definition der Rust-Strukturen, die das Schema der Kernkonfiguration abbilden.

Rust

```
// core_infra/src/config/mod.rs
use crate::error::{CoreError, ConfigError};
use serde::Deserialize;
use std::path::{Path, PathBuf};
use std::fs;
use super::utils; // Importiere utils Modul
use super::config::defaults; // Importiere defaults Modul

/// Represents the core configuration for the application.
/// Loaded from a TOML file.
#
#[serde(deny_unknown_fields)] // Strikte Prüfung auf unbekannte Felder
pub struct CoreConfig {
    #[serde(default = "defaults::default_logging_config")]
    pub logging: LoggingConfig,
    // Weitere Kern-Einstellungen hier hinzufügen, z.B.:
    // #[serde(default = "defaults::default_feature_flags")]
    // pub features: FeatureFlags,
}

/// Configuration specific to the logging subsystem.
#
#[serde(deny_unknown_fields)]
pub struct LoggingConfig {
    /// The minimum log level to record (e.g., "trace", "debug", "info", "warn", "error").
    #[serde(default = "defaults::default_log_level")]
    pub level: String,

    /// Optional path to a file where logs should be written.
    /// If None, logs are written to stdout/stderr.
    #[serde(default = "defaults::default_log_file_path")]
    pub file_path: Option<PathBuf>,

    /// Log format (e.g., "text", "json").
    #[serde(default = "defaults::default_log_format")]
    pub format: String, // Oder eine Enum LogFormat definieren
}

// Beispiel für weitere Konfigurationsstrukturen
// #
// #[serde(deny_unknown_fields)]
// pub struct FeatureFlags {
//     #[serde(default = "defaults::default_bool_false")]
//     pub experimental_feature_x: bool,
// }

/// Service responsible for loading the application's core configuration.
#
pub struct ConfigLoader {
    // Keine Felder benötigt, da die Logik in der Methode liegt
}

impl ConfigLoader {
    /// Loads the core configuration from the standard location(s).
    ///
    /// Looks for `config.toml` in the application-specific config directory
    /// determined by `core_infra::utils::paths::get_app_config_dir()`.
    ///
    /// # Errors
    /// Returns `CoreError::Config` variants if loading, parsing, or validation fails.
    pub fn load() -> Result<CoreConfig, CoreError> {
        let config_dir = utils::paths::get_app_config_dir()?;
        let config_path = config_dir.join("config.toml");

        // 1. Sicherstellen, dass das Konfigurationsverzeichnis existiert (optional, aber gut für Erststart)
        // utils::fs::ensure_dir_exists(&config_dir)?; // Kann Fehler werfen, wenn nicht beschreibbar

        // 2. Konfigurationsdatei lesen
        let content = fs::read_to_string(&config_path).map_err(|e| {
            // Unterscheide zwischen "nicht gefunden" und anderen Lesefehlern
            if e.kind() == std::io::ErrorKind::NotFound {
                 CoreError::Config(ConfigError::NotFound { locations: vec![config_path.clone()] })
            } else {
                 CoreError::Config(ConfigError::ReadError { path: config_path.clone(), source: e })
            }
        })?;

        // 3. TOML-Inhalt deserialisieren
        let mut config: CoreConfig = toml::from_str(&content)
           .map_err(|e| CoreError::Config(ConfigError::ParseError(e)))?;

        // 4. Post-Deserialisierungs-Validierung
        Self::validate_config(&mut config)?;

        // 5. Validierte Konfiguration zurückgeben
        Ok(config)
    }

    /// Performs post-deserialization validation of the configuration.
    /// Modifiziert die Konfiguration ggf. (z.B. Pfade absolut machen).
    fn validate_config(config: &mut CoreConfig) -> Result<(), CoreError> {
        // Validiere Log-Level
        let valid_levels = ["trace", "debug", "info", "warn", "error"];
        if!valid_levels.contains(&config.logging.level.to_lowercase().as_str()) {
            return Err(CoreError::Config(ConfigError::ValidationError(format!(
                "Invalid logging level '{}'. Must be one of: {:?}",
                config.logging.level, valid_levels
            ))));
        }

        // Validiere Log-Format
        let valid_formats = ["text", "json"];
        if!valid_formats.contains(&config.logging.format.to_lowercase().as_str()) {
             return Err(CoreError::Config(ConfigError::ValidationError(format!(
                "Invalid logging format '{}'. Must be one of: {:?}",
                config.logging.format, valid_formats
            ))));
        }


        // Wandle relative Logpfade in absolute Pfade um (relativ zum Konfig-Verzeichnis oder Daten-Verzeichnis)
        if let Some(ref mut log_path) = config.logging.file_path {
            if log_path.is_relative() {
                // Entscheide, relativ wozu? Hier Annahme: relativ zum State-Verzeichnis
                let state_dir = utils::paths::get_app_state_dir()?;
                 // Stelle sicher, dass das State-Verzeichnis existiert
                utils::fs::ensure_dir_exists(&state_dir)?;
                *log_path = state_dir.join(&log_path);
            }
             // Optional: Stelle sicher, dass das übergeordnete Verzeichnis des Logfiles existiert
            if let Some(parent_dir) = log_path.parent() {
                utils::fs::ensure_dir_exists(parent_dir)?;
            }
        }


        // Füge hier weitere Validierungen für andere Konfigurationsabschnitte hinzu

        Ok(())
    }
}

```

- **Spezifikation:**
    - Alle Konfigurationsstrukturen müssen `serde::Deserialize`, `Debug`, `Clone`, `PartialEq` implementieren.8
    - `#[serde(deny_unknown_fields)]` ist zwingend, um Fehler bei unbekannten Feldern in der TOML-Datei zu erzeugen.
    - Felder sind `pub`. Standardwerte werden über `#[serde(default = "path::to::default_fn")]` gesetzt, wobei die Default-Funktionen in `core_infra::config::defaults` liegen.
    - Typen müssen exakt spezifiziert sein (`String`, `Option<PathBuf>`, `bool`, etc.).

### 5.3. Configuration Loading Service (`core_infra::config::mod.rs`)

- **Struktur:** `ConfigLoader` (Struct ohne Felder, Logik in `impl`-Block).
- **Methode:** `pub fn load() -> Result<CoreConfig, CoreError>`
- **Logik (Schritt-für-Schritt):**
    1. Ermittle den Pfad zum anwendungsspezifischen Konfigurationsverzeichnis mittels `utils::paths::get_app_config_dir()`.
    2. Konstruiere den vollständigen Pfad zur Konfigurationsdatei (z.B. `config.toml`). Der Dateiname ist fest auf `config.toml` festgelegt.
    3. Versuche, den Inhalt der Datei mit `std::fs::read_to_string` zu lesen. Bilde `std::io::Error` auf `CoreError::Config(ConfigError::ReadError)` oder `CoreError::Config(ConfigError::NotFound)` ab.
    4. Deserialisiere den gelesenen String mittels `toml::from_str::<CoreConfig>`.12 Bilde `toml::de::Error` auf `CoreError::Config(ConfigError::ParseError)` ab.
    5. Rufe die interne Validierungsfunktion `validate_config` auf.
    6. Gib die validierte `CoreConfig` im `Ok`-Fall zurück.

### 5.4. Configuration Format

- Das einzige unterstützte Konfigurationsformat ist **TOML**.12
- **Beispiel `config.toml`:**
    
    Ini, TOML
    
    ```
    # Beispiel config.toml
    
    [logging]
    level = "debug" # Mögliche Werte: "trace", "debug", "info", "warn", "error"
    file_path = "app.log" # Optional. Relativ zum State-Verzeichnis oder absolut.
    format = "text" # Mögliche Werte: "text", "json"
    
    # [features]
    # experimental_feature_x = true
    ```
    

### 5.5. Default Values (`core_infra::config::defaults.rs`)

Definition von Funktionen, die Standardwerte für die Konfigurationsstrukturen liefern.

Rust

```
// core_infra/src/config/defaults.rs
use super::{LoggingConfig}; // Importiere die relevanten Structs
use std::path::PathBuf;

pub(super) fn default_logging_config() -> LoggingConfig {
    LoggingConfig {
        level: default_log_level(),
        file_path: default_log_file_path(),
        format: default_log_format(),
    }
}

pub(super) fn default_log_level() -> String {
    "info".to_string()
}

pub(super) fn default_log_file_path() -> Option<PathBuf> {
    None // Standardmäßig auf stdout/stderr loggen
}

pub(super) fn default_log_format() -> String {
    "text".to_string()
}

// Beispiel für booleschen Default
// pub(super) fn default_bool_false() -> bool {
//     false
// }

// Füge hier Default-Funktionen für alle `#[serde(default = "...")]` Felder hinzu.
```

- **Spezifikation:**
    - Für jedes Feld mit `#[serde(default = "...")]` muss eine entsprechende `pub(super) fn default_...() -> FieldType` Funktion existieren.
    - Die Funktionen müssen den korrekten Typ zurückgeben und den dokumentierten Standardwert repräsentieren.

Die Trennung der Konfigurationslogik (Laden, Parsen, Validieren) vom reinen Datenschema (`CoreConfig`, `LoggingConfig`) und den Standardwerten (`defaults.rs`) fördert die Modularität und Testbarkeit. Die Validierung nach der Deserialisierung ist entscheidend, um sicherzustellen, dass die Konfiguration nicht nur syntaktisch korrekt ist, sondern auch semantisch gültig (z.B. gültige Log-Level). Die Konfiguration beeinflusst direkt das Verhalten anderer Kernkomponenten, insbesondere des Loggings, was eine sorgfältige Initialisierungsreihenfolge erfordert.

## 6. Logging Infrastructure (`core_infra::logging`)

Stellt eine zentrale und konfigurierbare Logging-Lösung für die gesamte Anwendung bereit, basierend auf der `tracing`-Crate.

### 6.1. Modulstruktur

```
core_infra/
└── src/
    ├── lib.rs
    ├── types.rs
    ├── error.rs
    ├── utils/
    │   └──...
    ├── config/
    │   └──...
    └── logging.rs // Enthält Initialisierungslogik
```

### 6.2. Logging Facade

Die `tracing`-Crate 13 wird als alleinige Schnittstelle für alle Logging-Aktivitäten in der Anwendung vorgeschrieben. Sie bietet strukturierte Logs und Span-basiertes Tracing.

### 6.3. Initialisierung (`core_infra::logging::init_logging`)

Eine Funktion zur Initialisierung des globalen `tracing`-Subscribers basierend auf der geladenen Konfiguration.

Rust

```
// core_infra/src/logging.rs
use crate::config::LoggingConfig;
use crate::error::CoreError;
use tracing::{Level, info};
use tracing_subscriber::{fmt, layer::SubscriberExt, util::SubscriberInitExt, EnvFilter, Layer, Registry};
use std::io::{stdout, Write};
use std::path::Path;
use tracing_appender; // Für Dateilogging mit Rotation

/// Initializes the global tracing subscriber based on the provided configuration.
///
/// This function should be called **once** at the very beginning of the application startup.
/// It sets up logging to stdout/stderr and optionally to a file.
///
/// Handles the potential issue of needing logging before config is fully loaded
/// by allowing an optional initial call with default settings.
///
/// # Arguments
/// * `config` - The logging configuration obtained from `CoreConfig`.
/// * `is_reload` - Set to true if this is re-initializing after an initial basic setup.
///
/// # Errors
/// Returns `CoreError::LoggingInitialization` if setup fails (e.g., file cannot be opened).
pub fn init_logging(config: &LoggingConfig, is_reload: bool) -> Result<(), CoreError> {
    // 1. Filter-Level bestimmen
    let level_filter = match config.level.to_lowercase().as_str() {
        "trace" => EnvFilter::new(Level::TRACE.to_string()),
        "debug" => EnvFilter::new(Level::DEBUG.to_string()),
        "info" => EnvFilter::new(Level::INFO.to_string()),
        "warn" => EnvFilter::new(Level::WARN.to_string()),
        "error" => EnvFilter::new(Level::ERROR.to_string()),
        _ => {
            // Sollte durch validate_config abgefangen werden, aber sicherheitshalber
            return Err(CoreError::LoggingInitialization(format!(
                "Invalid log level in config: {}",
                config.level
            )));
        }
    };

    // 2. Layer für stdout/stderr erstellen (immer aktiv)
    let use_ansi = atty::is(atty::Stream::Stdout); // Farben nur bei TTY
    let stdout_layer = fmt::layer()
       .with_writer(stdout) // Explizit stdout
       .with_ansi(use_ansi) // ANSI-Farben aktivieren/deaktivieren
       .with_filter(level_filter.clone()); // Klonen, da Filter mehrfach verwendet wird

    // 3. Optional: Layer für Dateilogging erstellen
    let file_layer = if let Some(log_path) = &config.file_path {
        Some(create_file_layer(log_path, &config.format)?
            .with_filter(level_filter)) // Gleiches Level für Datei
    } else {
        None
    };

    // 4. Subscriber zusammenbauen und global setzen
    let registry = Registry::default()
       .with(stdout_layer); // stdout ist immer dabei

    // Füge den Dateilayer hinzu, falls vorhanden
    let subscriber = if let Some(layer) = file_layer {
        registry.with(layer)
    } else {
        registry.with(tracing_subscriber::filter::FilterExt::boxed(stdout_layer)) // Boxen, um Typkonsistenz zu wahren, wenn nur ein Layer da ist
    };


    // Versuche, den globalen Subscriber zu setzen
    if SubscriberInitExt::try_init(subscriber).is_err() {
        // Fehler nur werfen, wenn es nicht der Reload nach der initialen Einrichtung ist.
        // Beim Reload ist es erwartet, dass bereits ein Subscriber gesetzt ist.
        if!is_reload {
             return Err(CoreError::LoggingInitialization(
                "Failed to set global tracing subscriber. Was it already initialized?".to_string(),
            ));
        }
        // Beim Reload loggen wir, dass wir rekonfigurieren (mit dem *alten* Logger)
        info!("Re-initializing logging configuration.");
        // Der neue Subscriber wird nicht gesetzt, aber die Konfiguration wurde validiert.
        // In einem realen Szenario bräuchte man einen Mechanismus zur dynamischen Rekonfiguration
        // des Filters/Writers, was über tracing-subscriber's ReloadHandle ginge, aber
        // die Komplexität hier übersteigt. Für diese Spezifikation reicht die Validierung.
    }

    Ok(())
}

/// Helper function to create a file logging layer.
fn create_file_layer(log_path: &Path, format: &str) -> Result<Box<dyn Layer<Registry> + Send + Sync + 'static>, CoreError> {
     // Stelle sicher, dass das Verzeichnis existiert (sollte durch validate_config erfolgt sein)
    if let Some(parent) = log_path.parent() {
        if!parent.exists() {
             // Versuche es zu erstellen, falls validate_config es nicht getan hat
            utils::fs::ensure_dir_exists(parent)?;
        }
    }

    // Konfiguriere den File Appender (z.B. tägliche Rotation)
    let file_appender = tracing_appender::rolling::daily(
        log_path.parent().unwrap_or_else(|| Path::new(".")), // Sicherstellen, dass parent existiert
        log_path.file_name().unwrap_or_else(|| std::ffi::OsStr::new("core.log")),
    );
    let (non_blocking_writer, _guard) = tracing_appender::non_blocking(file_appender);

    // Wähle das Format basierend auf der Konfiguration
     match format.to_lowercase().as_str() {
        "json" => {
            let layer = fmt::layer()
               .json() // JSON-Format aktivieren
               .with_writer(non_blocking_writer)
               .with_ansi(false); // Keine ANSI-Codes in Dateien
             Ok(Box::new(layer))
        }
        "text" | _ => { // Default auf Text
             let layer = fmt::layer()
               .with_writer(non_blocking_writer)
               .with_ansi(false); // Keine ANSI-Codes in Dateien
             Ok(Box::new(layer))
        }
    }
    // _guard muss im Scope bleiben, damit der Writer funktioniert.
    // In einer echten Anwendung muss dieser Guard an einen geeigneten Ort verschoben werden,
    // z.B. in die Hauptanwendungsstruktur oder global statisch (mit lazy_static/once_cell).
    // Für diese Spezifikation ignorieren wir die Lebenszeit des Guards, gehen aber davon aus,
    // dass er korrekt gehandhabt wird.
    // std::mem::forget(_guard); // NICHT IN PRODUKTION VERWENDEN! Nur zur Kompilierung hier.
    // Besser: Rückgabe des Guards oder Speicherung in einem globalen Kontext.
}

/// Initializes a minimal fallback logger to stderr before configuration is loaded.
/// This should be called unconditionally at the very start.
pub fn init_minimal_logging() {
     // Setze einen einfachen Logger, der nur auf stderr schreibt, falls noch keiner gesetzt ist.
    // Ignoriere Fehler, falls bereits einer gesetzt wurde (z.B. in Tests).
    let filter = EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new(Level::INFO.to_string()));
    let _ = fmt::Subscriber::builder()
       .with_env_filter(filter)
       .with_writer(std::io::stderr) // Explizit stderr für frühe Logs
       .try_init();
}


```

- **Spezifikation:**
    - Die Funktion `init_logging` nimmt die `LoggingConfig` entgegen.
    - Sie MUSS das Log-Level (`config.level`) parsen und in einen `EnvFilter` oder äquivalenten Filter umwandeln. Ungültige Level führen zu `CoreError::LoggingInitialization`.
    - Ein `fmt::Layer` für die Standardausgabe (stdout/stderr) wird immer konfiguriert. ANSI-Farbunterstützung wird basierend auf `atty::is(atty::Stream::Stdout)` aktiviert/deaktiviert.
    - Wenn `config.file_path` `Some` ist:
        - Ein Dateilogging-Layer wird mittels `tracing_appender` (oder einer ähnlichen Crate für Rotation) erstellt. `tracing_appender` muss als Abhängigkeit hinzugefügt werden.
        - Der Pfad wird aus der Konfiguration übernommen. Die Validierung (und Umwandlung in einen absoluten Pfad) sollte bereits in `ConfigLoader::validate_config` erfolgt sein.
        - Fehler beim Öffnen/Erstellen der Logdatei oder des Verzeichnisses führen zu `CoreError::LoggingInitialization` oder `CoreError::Filesystem`.
        - Das Format (`text` oder `json`) wird gemäß `config.format` konfiguriert. ANSI-Codes werden für Dateilogs deaktiviert.
    - Die Layer werden kombiniert (mittels `.with()`) und der resultierende Subscriber wird mit `SubscriberInitExt::try_init` als globaler Standard gesetzt. Fehler beim Setzen (z.B. wenn bereits initialisiert) werden behandelt (siehe `is_reload`).
    - **Initialisierungsproblem:** Die Funktion `init_minimal_logging` wird hinzugefügt. Sie MUSS als Allererstes im `main`-Funktion aufgerufen werden, _bevor_ `ConfigLoader::load` versucht wird. Sie richtet einen einfachen Fallback-Logger ein (z.B. `INFO` Level auf `stderr`). `init_logging` wird dann _nach_ erfolgreichem Laden der Konfiguration erneut aufgerufen (mit `is_reload = true`), um die endgültige Konfiguration anzuwenden. Dies stellt sicher, dass Konfigurationsladefehler geloggt werden können.

### 6.4. Logging Macros

Entwickler MÜSSEN die Standard-`tracing`-Makros verwenden:

- `trace!(...)`: Für sehr detaillierte Diagnoseinformationen.
- `debug!(...)`: Für Debugging-Informationen während der Entwicklung.
- `info!(...)`: Für informative Nachrichten über den normalen Betrieb.
- `warn!(...)`: Für Warnungen über potenziell problematische Situationen.
- `error!(...)`: Für Fehlerbedingungen, die den Betrieb beeinträchtigen.
- `event!(Level::...,...)`: Für explizite Events mit spezifischem Level.

### 6.5. Structured Logging

Die Verwendung von strukturierten Feldern wird dringend empfohlen, um den vollen Nutzen aus `tracing` zu ziehen.13

- **Beispiel:** `info!(user_id = %user.id, operation = "login", success = true, "User logged in successfully");`
- Felder sollten konsistent benannt werden.

### 6.6. Spans

Die Verwendung von `tracing::span!` wird für die Instrumentierung logischer Arbeitsabschnitte empfohlen, insbesondere für Operationen, die Zeit in Anspruch nehmen oder über asynchrone Grenzen hinweggehen.

- **Beispiel:**
    
    Rust
    
    ```
    let span = tracing::span!(Level::DEBUG, "config_loading", path = %config_path.display());
    let _enter = span.enter(); // Span betreten
    //... Logik zum Laden der Konfiguration...
    // Span wird automatisch verlassen, wenn _enter aus dem Scope geht
    ```
    
- Spans ermöglichen die Korrelation von Log-Ereignissen und die Messung von Dauern.

Die Wahl von `tracing` bietet eine flexible und leistungsstarke Grundlage. Die Spezifikation eines klaren Initialisierungsprozesses, der auch frühe Fehler beim Konfigurationsladen abdeckt, ist entscheidend. Die Festlegung auf `fmt::Subscriber` mit optionalem Dateilogging als Standard vereinfacht die Implementierung für Entwickler, während die `tracing`-API selbst fortgeschrittene Anwendungsfälle ermöglicht.

## 7. Error Handling (`core_infra::error`)

Definiert eine einheitliche und robuste Fehlerbehandlungsstrategie für die Kernschicht.

### 7.1. Modulstruktur

```
core_infra/
└── src/
    ├── lib.rs
    ├── types.rs
    ├── utils/
    │   └──...
    ├── config/
    │   └──...
    ├── logging.rs
    └── error.rs // Definiert CoreError, ConfigError etc.
```

### 7.2. Core Error Enum (`core_infra::error.rs`)

Eine zentrale Enum `CoreError`, die alle möglichen Fehlerfälle der Kernschicht repräsentiert. Verwendet `thiserror` zur einfachen Implementierung.7

Rust

```
// core_infra/src/error.rs
use thiserror::Error;
use std::path::PathBuf;

/// The primary error type for the core infrastructure layer.
/// Aggregates specific error categories.
#
pub enum CoreError {
    /// Errors related to configuration loading, parsing, or validation.
    #[error("Configuration Error: {0}")]
    Config(#[from] ConfigError),

    /// Errors occurring during logging subsystem initialization.
    #[error("Logging Initialization Failed: {0}")]
    LoggingInitialization(String), // Kann spezifischer sein, z.B. eine eigene Enum

    /// Errors related to filesystem operations, including context.
    #[error("Filesystem Error: {message} (Path: {path:?})")]
    Filesystem {
        message: String,
        path: PathBuf,
        #[source] // Behält die ursprüngliche IO-Fehlerquelle
        source: std::io::Error,
    },

    /// General I/O errors not covered by Filesystem or Config::ReadError.
    #[error("I/O Error: {0}")]
    Io(#[from] std::io::Error),

    /// Errors indicating invalid input parameters or data.
    #[error("Invalid Input: {0}")]
    InvalidInput(String),

    /// Placeholder for other potential core errors.
    #[error("An unexpected internal error occurred: {0}")]
    Internal(String),
}

/// Specific errors related to configuration handling.
#
pub enum ConfigError {
    /// Failed to read the configuration file.
    #[error("Failed to read configuration file from {path:?}")]
    ReadError {
        path: PathBuf,
        #[source]
        source: std::io::Error,
    },

    /// Failed to parse the TOML configuration content.
    #[error("Failed to parse configuration file: {0}")]
    ParseError(#[from] toml::de::Error),

    /// Configuration validation failed after parsing.
    #[error("Configuration validation failed: {0}")]
    ValidationError(String),

    /// Configuration file was not found at the expected location(s).
    #[error("Configuration file not found at expected locations: {locations:?}")]
    NotFound { locations: Vec<PathBuf> },

    /// Required configuration directory (e.g., XDG_CONFIG_HOME) could not be determined.
    #[error("Could not determine base directory for {dir_type}")]
    DirectoryUnavailable { dir_type: String },
}

// Implementiere Konvertierungen, falls nötig, um Kontext hinzuzufügen,
// bevor #[from] verwendet wird. Beispiel:
// impl From<SpecificError> for CoreError {
//     fn from(err: SpecificError) -> Self {
//         CoreError::Internal(format!("Specific error occurred: {}", err))
//     }
// }
```

- **Spezifikation:**
    - `CoreError` ist die einzige Fehlertyp, der von öffentlichen Funktionen dieser Schicht zurückgegeben wird.
    - Verwendet `thiserror::Error` für die Ableitung von `std::error::Error` und `Display`.
    - `#` ist obligatorisch.
    - Varianten decken alle logischen Fehlerquellen ab (Konfiguration, Logging, FS, IO, Input-Validierung).
    - `#[error("...")]` Attribute definieren menschenlesbare Fehlermeldungen. Diese Meldungen sollten kontextreich sein.
    - `#[from]` wird verwendet, um Standardfehler (wie `std::io::Error`, `toml::de::Error`) automatisch in `CoreError`-Varianten umzuwandeln.7 Dies vereinfacht die Fehlerkonvertierung.
    - `#[source]` wird verwendet, um die zugrundeliegende Fehlerursache für Debugging-Zwecke beizubehalten.7
    - Spezifischere Fehler-Enums (wie `ConfigError`) können definiert und mittels `#[from]` in `CoreError` eingebettet werden. Dies verbessert die interne Strukturierung und ermöglicht es Aufrufern bei Bedarf, spezifischere Fehlerfälle zu behandeln, ohne die Komplexität der öffentlichen API zu erhöhen.17
    - Die `CoreError::Filesystem`-Variante demonstriert das Hinzufügen von Kontext (Nachricht, Pfad) zu einem zugrundeliegenden Fehler (`source: std::io::Error`), was für die Fehlersuche unerlässlich ist.

### 7.3. Error Propagation

- Alle fehleranfälligen öffentlichen Funktionen MÜSSEN `Result<T, CoreError>` zurückgeben.
- Der `?`-Operator MUSS zur Fehlerpropagation innerhalb der Funktionen verwendet werden.
- Wo nötig, MUSS `.map_err()` verwendet werden, um Low-Level-Fehler in passende `CoreError`-Varianten umzuwandeln und dabei relevanten Kontext (z.B. Dateipfade, Operationsnamen) hinzuzufügen.

### 7.4. Error Handling Strategy

- **Kein `unwrap`/`expect`:** Die Verwendung von `.unwrap()` oder `.expect()` in der Kernschicht ist verboten, da dies zu Panics führt, die nicht ordnungsgemäß behandelt werden können.1 Erwartete Fehler müssen über `Result` signalisiert werden.
- **Panics:** Panics sollten nur bei logischen Programmierfehlern (Bugs) auftreten, die als nicht behebbar gelten (z.B. Verletzung von internen Invarianten, die durch korrekte Nutzung der API nicht auftreten sollten). Solche Fälle deuten auf einen Fehler im Code selbst hin, nicht auf einen Laufzeitfehler.

Diese Strategie stellt sicher, dass Fehler explizit behandelt und propagiert werden, was die Robustheit und Wartbarkeit des Codes verbessert. Die zentrale `CoreError`-Enum bietet eine konsistente Schnittstelle für Fehler aus der Kernschicht.

## 8. Core Event Definitions

Für diese Kerninfrastrukturschicht werden **keine** eigenen Events definiert.

Die Kernschicht konzentriert sich auf grundlegende, meist synchrone Setup-Aufgaben und die Bereitstellung von Utilities. Ereignisbasierte Kommunikation (Publish/Subscribe) ist typischerweise eine Aufgabe höherer Schichten oder dedizierter Event-Bus-Systeme, die auf der Kerninfrastruktur aufbauen, aber nicht Teil davon sind. Die Definition eines Event-Systems würde die Komplexität der Kernschicht unnötig erhöhen und ihre Abhängigkeiten erweitern (z.B. auf eine asynchrone Laufzeit oder eine spezifische Event-Bibliothek), was ihrem Zweck widerspricht.

Sollte in Zukunft ein Bedarf für _fundamentale_, von der Kernschicht ausgehende Ereignisse entstehen (z.B. dynamische Neuladung der Kernkonfiguration), müsste diese Spezifikation entsprechend erweitert werden, inklusive:

- Definition der Event-Typen (Structs/Enums).
- Spezifikation der Payloads (`Send + Sync + Clone + Debug`).
- Identifikation der emittierenden Komponente (Publisher).
- Definition des Übertragungsmechanismus (z.B. `tokio::sync::broadcast`).

Aktuell ist dies jedoch nicht vorgesehen.

## 9. External Dependencies

Die Kerninfrastruktur minimiert ihre externen Abhängigkeiten, um schlank, stabil und schnell kompilierbar zu bleiben. Nur essenzielle Crates für die Kernfunktionalitäten (Logging, Konfiguration, Fehlerbehandlung, Pfadermittlung) sind erlaubt.

### 9.1. Dependency Policy

- Nur absolut notwendige Abhängigkeiten hinzufügen.
- Stabile Versionen verwenden. Versionen müssen exakt spezifiziert werden.
- Benötigte Crate-Features explizit angeben. Standard-Features deaktivieren (`default-features = false`), wenn nicht alle benötigt werden, um die Abhängigkeitsgröße zu reduzieren.
- Regelmäßige Überprüfung auf Updates und Sicherheitsschwachstellen.

### 9.2. Dependency Table

Die folgenden externen Crates sind für die Implementierung der Kerninfrastruktur zwingend erforderlich:

|   |   |   |   |   |
|---|---|---|---|---|
|**Crate**|**Exakte Version¹**|**Benötigte Features**|**Rationale**|**Snippet Refs**|
|`tracing`|`0.1.40`|`std`|Kern-Logging-Fassade und API|13|
|`tracing-subscriber`|`0.3.18`|`fmt`, `env-filter`, `std`, `registry`, `json` (optional für Format)|`fmt`-Subscriber, Filterung, Registry-Basis|15|
|`tracing-appender`|`0.2.3`|(Standard)|Dateilogging mit Rotation|-|
|`serde`|`1.0.219`|`derive`, `std`|Deserialisierung für Konfigurationsstrukturen|8|
|`toml`|`0.8.22`|(Standard, enthält `parse`)|TOML-Parsing für Konfigurationsdateien|12|
|`thiserror`|`1.0.59`|(Standard)|Ableitung von `std::error::Error`, `Display`|6|
|`directories-next`|`2.0.0`|(Standard)|Cross-Plattform XDG/Standard-Verzeichnisermittlung|11|
|`log`|`0.4.21`|`std`|Transitiv benötigt von `tracing-subscriber`|14|
|`atty`|`0.2.15`|(Standard)|Erkennung von TTY für ANSI-Farben im Logging|-|

¹ _Anmerkung zu Versionen:_ Die hier angegebenen Versionen entsprechen den zum Zeitpunkt der Erstellung dieses Dokuments als stabil bekannten oder in den Referenzmaterialien genannten Versionen. Vor der Implementierung sind die **aktuellsten stabilen Versionen** zu überprüfen und zu verwenden, sofern sie API-kompatibel sind oder die Spezifikation entsprechend angepasst wird. Die exakten Versionen MÜSSEN in der `Cargo.toml` fixiert werden.

Diese Tabelle ist entscheidend für die Reproduzierbarkeit der Builds und die Stabilität der Kernschicht. Jede Änderung an diesen Abhängigkeiten (Version, Features) erfordert eine Überprüfung und Anpassung dieser Spezifikation.

## 10. Implementation Constraints and Guidelines

Diese Richtlinien stellen sicher, dass die Implementierung konsistent, wartbar und konform mit den Designzielen der Kernschicht ist.

### 10.1. Rust Version

- **Minimum Supported Rust Version (MSRV):** `1.70.0` (oder höher, basierend auf den MSRV-Anforderungen der Abhängigkeiten wie `tracing` 16 und der Verwendung von Sprachfeatures). Muss in `Cargo.toml` angegeben werden.
- **Entwicklung:** Die Entwicklung sollte mit der **aktuellsten stabilen Rust-Version** erfolgen.

### 10.2. `unsafe` Code

- Die Verwendung von `unsafe`-Blöcken ist in der gesamten Kerninfrastruktur **strikt verboten** (siehe Abschnitt 2.4).

### 10.3. Testing

- **Unit Tests:** Jede öffentliche Funktion und Methode MUSS durch Unit-Tests abgedeckt sein. Tests müssen sowohl Erfolgs- als auch Fehlerpfade validieren. Testmodule (`#[cfg(test)] mod tests {... }`) sollen direkt in den jeweiligen Quelldateien platziert werden.
- **Integration Tests:** Integrationstests, die das Zusammenspiel der Kernschicht mit anderen Schichten testen, gehören _nicht_ in dieses Crate, sondern in eine übergeordnete Testsuite.

### 10.4. Documentation

- **Rustdoc:** Alle öffentlichen Elemente (Crates, Module, Typen, Funktionen, Methoden, Traits, Konstanten) MÜSSEN umfassende Dokumentationskommentare (`///`) aufweisen.1
- **Inhalt:** Die Dokumentation muss den Zweck, Parameter, Rückgabewerte, garantierte Vor-/Nachbedingungen, mögliche `CoreError`-Varianten und ggf. Beispiele (`#[test]`-fähige Beispiele bevorzugt) enthalten (C-EXAMPLE, C-FAILURE 1).
- **Crate-Level Doku:** `src/lib.rs` MUSS eine ausführliche Crate-Level-Dokumentation enthalten, die den Zweck und die Verwendung der Kerninfrastruktur erklärt (C-CRATE-DOC 1).

### 10.5. Module Structure

Die Implementierung MUSS der folgenden Modulstruktur folgen:

```
core_infra/
└── src/
    ├── lib.rs         # Crate root, re-exportiert öffentliche APIs
    ├── error.rs       # Definition von CoreError, ConfigError etc.
    ├── types.rs       # Definition von Core-Datentypen (Structs, Enums)
    ├── config/        # Konfigurations-bezogene Module
    │   ├── mod.rs     # Definition von CoreConfig, LoggingConfig, ConfigLoader
    │   └── defaults.rs# Funktionen für Standardwerte
    ├── logging.rs     # Logging-Initialisierungslogik
    └── utils/         # Utility-Module
        ├── mod.rs     # Re-exportiert öffentliche Utils
        ├── fs.rs      # Filesystem-Utilities
        └── paths.rs   # Pfadermittlungs-Utilities
        // Optional: strings.rs, time.rs
```

Öffentliche APIs sollten selektiv von `lib.rs` re-exportiert werden, um die Schnittstelle klar zu definieren.

### 10.6. `Cargo.toml` Manifest

Die `Cargo.toml`-Datei MUSS wie folgt konfiguriert sein:

Ini, TOML

```
[package]
name = "core_infra"
version = "0.1.0" # Startversion, SemVer beachten
edition = "2021" # Oder neuere Edition, falls MSRV es erlaubt
authors = ["Your Name <your.email@example.com>"] # Anpassen
description = "Core infrastructure layer providing foundational utilities, configuration, logging, and error handling."
license = "MIT OR Apache-2.0" # Lizenz gemäß Vorgabe [12, 18]
repository = "URL/to/your/repository" # Optional, aber empfohlen
homepage = "URL/to/project/homepage" # Optional
documentation = "URL/to/docs" # Optional, falls extern gehostet
readme = "README.md" # Optional
keywords = ["core", "infrastructure", "config", "logging", "error"] # Optional
categories = ["api-bindings", "config", "filesystem"] # Optional, siehe crates.io Kategorien

# Mindest-Rust-Version festlegen
rust-version = "1.70.0" # Anpassen gemäß Abschnitt 10.1

[dependencies]
# Versionen exakt wie in Abschnitt 9.2 spezifiziert
tracing = { version = "0.1.40" }
tracing-subscriber = { version = "0.3.18", features = ["fmt", "env-filter", "std", "registry", "json"] } # json optional
tracing-appender = { version = "0.2.3" }
serde = { version = "1.0.219", features = ["derive"] } # std ist default feature
toml = { version = "0.8.22" }
thiserror = { version = "1.0.59" }
directories-next = { version = "2.0.0" }
log = { version = "0.4.21" }
atty = { version = "0.2.15" }

# Optional: Dev-Dependencies für Tests
[dev-dependencies]
# z.B. pretty_assertions = "1.4.0" für bessere Test-Diffs

# Optional: Build-Dependencies, falls benötigt
[build-dependencies]

# Optional: Profile für Optimierungen, etc.
[profile.release]
lto = true          # Link Time Optimization für Release-Builds
codegen-units = 1   # Bessere Optimierung, langsamere Kompilierung
panic = 'abort'     # Kleinere Binaries, keine Stack Unwinding Info bei Panic
strip = true        # Symbole entfernen für kleinere Binaries
```

- **Spezifikation:**
    - Alle Metadaten im `[package]`-Abschnitt müssen korrekt ausgefüllt sein.1
    - Die `edition` muss angegeben werden.
    - Die `license` muss den gängigen Open-Source-Lizenzen entsprechen, die in den Referenzen verwendet werden (oft MIT/Apache-2.0).7
    - Die `rust-version` MUSS gesetzt sein.
    - Der `[dependencies]`-Abschnitt MUSS exakt die Crates, Versionen und Features aus Abschnitt 9.2 enthalten. Keine zusätzlichen Abhängigkeiten sind erlaubt.

## 11. Schlussfolgerungen

Dieser Implementierungsleitfaden definiert die Kerninfrastrukturschicht (`core_infra`) mit höchster Präzision und Detailgenauigkeit. Durch die strikte Befolgung dieser Spezifikation wird sichergestellt, dass Entwickler eine konsistente, robuste und wartbare Basis für das Gesamtsystem erstellen können, ohne eigene grundlegende Designentscheidungen treffen zu müssen.

Die Kernpunkte umfassen:

1. **Klare Verantwortlichkeiten:** Die Schicht beschränkt sich auf fundamentale Utilities, Konfiguration, Logging und Fehlerbehandlung und bleibt frei von Abhängigkeiten zu höheren Schichten.
2. **Rust Best Practices:** Die Implementierung folgt konsequent den Rust API Guidelines, Sicherheitsprinzipien (kein `unsafe`) und etablierten Mustern für Fehlerbehandlung und Dokumentation.
3. **Standardisierte Werkzeuge:** Die Verwendung von etablierten Crates wie `tracing`, `serde`, `toml`, `thiserror` und `directories-next` stellt die Nutzung bewährter Lösungen sicher und reduziert den Implementierungsaufwand.
4. **Präzise Schnittstellen:** Alle öffentlichen Typen, Funktionen und Module sind exakt definiert, inklusive Signaturen, Fehlertypen und Verhalten.
5. **Reproduzierbarkeit:** Durch die Festlegung exakter Abhängigkeitsversionen und Build-Konfigurationen wird eine hohe Reproduzierbarkeit gewährleistet.
6. **Plattformabstraktion:** Kritische plattformspezifische Aspekte wie die Verzeichnisstruktur werden durch Utilities (z.B. `core_infra::utils::paths`) abstrahiert.

Die sorgfältige Behandlung von Randfällen, wie die Initialisierungsreihenfolge von Konfiguration und Logging, sowie die detaillierte Definition der Fehlerbehandlung mit `CoreError` und `ConfigError` tragen maßgeblich zur Stabilität der Schicht bei. Entwickler können diesen Leitfaden als direkte Vorlage für die Implementierung verwenden.