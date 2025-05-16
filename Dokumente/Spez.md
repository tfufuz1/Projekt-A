## NovaDE: Spezifikation und Implementierungsleitfaden – Domänenschicht

Dieser Leitfaden beschreibt die Architektur, APIs, Datenstrukturen und Interaktionen der Domänenschicht (Crate: `novade-domain`). Diese Schicht kapselt die Kernlogik und den Zustand der NovaDE-Desktop-Umgebung und ist unabhängig von UI-Toolkit-spezifischen oder direkten System-Hardware-Interaktionen.

**Allgemeine Prinzipien der Domänenschicht:**

- **UI-Unabhängigkeit:** Keine direkten Abhängigkeiten von GTK4 oder anderen UI-Frameworks.
- **Systemunabhängigkeit:** Keine direkte Interaktion mit Wayland-Protokollen, D-Bus auf Low-Level-Ebene oder Hardware. Diese werden von der Systemschicht gehandhabt.
- **Fokus auf Geschäftslogik:** Definiert die Regeln, Zustände und Prozesse der Desktop-Umgebung.
- **API-Design:** Öffentliche Schnittstellen primär über Rust-Traits, die von Service-Strukturen implementiert werden, um Testbarkeit und lose Kopplung zu fördern.
- **Zustandsverwaltung:** Veränderliche Zustände innerhalb von Services werden threadsicher gekapselt (z.B. `Arc<Mutex<...>>` oder `Arc<RwLock<...>>`).
- **Asynchronität:** `async/await` und `async_trait` für potenziell blockierende Operationen oder wenn die Interaktion mit der Systemschicht dies erfordert.
- **Events:** Ein klar definierter Event-Mechanismus (z.B. `tokio::sync::broadcast`) für die Kommunikation von Zustandsänderungen.
- **Fehlerbehandlung:** Konsequente Nutzung von `thiserror` für modulspezifische Fehler-Enums. Fehler aus der Kernschicht (`novade-core`) werden gewrappt.
- **Abhängigkeiten:** Nutzt ausschließlich `novade-core` für fundamentale Typen und Fehlerbasis.

---

### Modul 1: `domain::theming`

**Zweck:** Logik des Erscheinungsbilds (Theming), Verwaltung von Design-Tokens, Interpretation von Theme-Definitionen, dynamische Theme-Wechsel (Farbschema, Akzentfarben).

**Referenz:** Siehe Dokument `B1 Domänenschicht.md` für eine bereits sehr detaillierte Ausarbeitung dieses Moduls. Die folgenden Punkte ergänzen oder verfeinern diese.

#### 1.1. APIs (Traits)

Rust

```
// domain/src/theming/service_api.rs

use async_trait::async_trait;
use std::path::PathBuf;
use tokio::sync::broadcast; // For eventing

use novade_core::types::Color as CoreColor; // Assuming Color is in novade-core

use super::types::{
    ThemeIdentifier, ColorSchemeType, AccentColor, AppliedThemeState,
    ThemingConfiguration, ThemeDefinition,
};
use super::errors::ThemingError;

/// Event that is broadcast when the theme state changes.
#[derive(Debug, Clone, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct ThemeChangedEvent {
    pub new_state: AppliedThemeState,
    pub old_state: Option<AppliedThemeState>, // Optional: previous state for comparison
}

#[async_trait]
pub trait ThemingEngineService: Send + Sync {
    /// Initializes the theming engine with a set of paths to load themes and tokens from,
    /// and an initial user configuration.
    async fn new(
        initial_config: ThemingConfiguration,
        theme_load_paths: Vec<PathBuf>,
        token_load_paths: Vec<PathBuf>,
        // Assuming a dependency on a core configuration service for file I/O abstraction
        config_service: Arc<dyn novade_core::config::ConfigServiceAsync>, // From novade-core
        broadcast_capacity: usize,
    ) -> Result<Arc<Self>, ThemingError> where Self: Sized; // Arc<Self> for easy sharing

    /// Retrieves the currently applied and fully resolved theme state.
    /// This is the primary way for the UI layer to get current theme values.
    async fn get_current_theme_state(&self) -> AppliedThemeState;

    /// Lists all available (loaded and validated) theme definitions.
    async fn get_available_themes(&self) -> Vec<ThemeDefinition>;

    /// Retrieves the current user-defined theming configuration being used by the engine.
    async fn get_current_configuration(&self) -> ThemingConfiguration;

    /// Updates the theming configuration. This triggers a re-calculation of the
    /// AppliedThemeState and, if changed, broadcasts a ThemeChangedEvent.
    async fn update_configuration(
        &self,
        new_config: ThemingConfiguration,
    ) -> Result<(), ThemingError>;

    /// Reloads all theme definitions and token files from the configured paths.
    /// Applies the current ThemingConfiguration to the reloaded data.
    /// Broadcasts a ThemeChangedEvent if the AppliedThemeState changes.
    async fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>;

    /// Allows other parts of the system to subscribe to theme change notifications.
    fn subscribe_to_theme_changes(&self) -> broadcast::Receiver<ThemeChangedEvent>;
}
```

#### 1.2. Datenstrukturen

Definiert in `domain/src/theming/types.rs`. Basierend auf `B1 Domänenschicht.md`, mit folgenden Präzisierungen:

- **`TokenIdentifier`**:
    - Invarianten: String nicht leer, enthält nur `a-zA-Z0-9.-`. Validierung in `new()` (z.B. via `debug_assert!` oder Rückgabe von `Result`).
- **`TokenValue`**:
    - `Opacity(f64)`: Wert muss zwischen 0.0 und 1.0 liegen (Validierung bei Erstellung/Parsing).
    - Die String-Werte in `Color`, `Dimension`, etc. sollten gültige CSS-Werte sein. Die Validierung dieser CSS-Strings erfolgt primär bei der Anwendung (UI-Schicht) oder durch Linting der Theme-Dateien. Die Domänenschicht stellt sicher, dass die Typen konsistent sind.
- **`TokenSet`**: `pub type TokenSet = std::collections::BTreeMap<TokenIdentifier, RawToken>;` (Verwendung von `BTreeMap` für deterministische Reihenfolge, was Tests und Debugging erleichtert. `TokenIdentifier` muss `Ord` implementieren).
- **`AccentColor`**:
    - `value: novade_core::types::Color`: Verwendet den `Color`-Typ aus `novade-core`.
- **`ThemeDefinition`**:
    - `supported_accent_colors: Option<Vec<AccentColor>>`
    - `accentable_tokens: Option<HashMap<TokenIdentifier, AccentModificationType>>`: (Neu) Definiert, welche Tokens von Akzentfarben beeinflusst werden und wie.
        - `AccentModificationType` (Enum): `DirectReplace`, `Lighten(f32)`, `Darken(f32)`, `DesaturateAndTint(f32)`.
- **`AppliedThemeState`**:
    - `resolved_tokens: std::collections::BTreeMap<TokenIdentifier, String>;`
- **`ThemingConfiguration`**:
    - `selected_accent_color: Option<novade_core::types::Color>;`

#### 1.3. Events

- **`ThemeChangedEvent`**: (Definiert in `service_api.rs` oder `events.rs`)
    - `pub new_state: AppliedThemeState`
    - `pub old_state: Option<AppliedThemeState>`

#### 1.4. Fehlerbehandlung

- **`ThemingError`** (in `domain/src/theming/errors.rs`): Basierend auf `B1 Domänenschicht.md`, erweitert um:
    - `InvalidTokenValue { token_id: TokenIdentifier, value_string: String, reason: String }`
    - `AccentColorApplicationError { theme_id: ThemeIdentifier, accent_color: CoreColor, details: String }`
    - `TokenResolutionError { token_id: TokenIdentifier, message: String, source: Option<Box<dyn std::error::Error + Send + Sync + 'static>> }`
    - `FilesystemError(#[from] novade_core::errors::CoreError)`: Für Fehler vom `ConfigServiceAsync`.

#### 1.5. Interaktionen

- **Mit `novade-core::config::ConfigServiceAsync`**: Zum asynchronen Laden von Token- und Theme-Dateien.
- **Mit `domain::global_settings_and_state_management`**: Kann globale Einstellungen lesen, die das Theming beeinflussen (z.B. eine globale "Hoher Kontrast"-Einstellung).

---

### Modul 2: `domain::workspaces`

**Zweck:** Umfassende Verwaltungslogik für Arbeitsbereiche ("Spaces").

**Referenz:** Siehe Dokument `B2 Domänenschicht.md` für eine bereits sehr detaillierte Ausarbeitung. Die folgenden Punkte ergänzen oder verfeinern diese.

#### 2.1. APIs (Traits)

Rust

```
// domain/src/workspaces/service_api.rs
use async_trait::async_trait;
use tokio::sync::broadcast;
use uuid::Uuid; // Assuming WorkspaceId = uuid::Uuid

use novade_core::types::RectInt; // For available_area in layout calculations

use super::types::{
    WorkspaceId, WindowIdentifier, WorkspaceLayoutType, Workspace,
    WorkspaceIconChangedData, WorkspaceAccentChangedData,
    WorkspaceCreationParams, // New struct for create_workspace
};
use super::config_types::WorkspaceSetSnapshot; // For config interaction
use super::errors::WorkspaceManagerError;
use crate::window_management_policy::types::WorkspaceWindowLayout; // Assuming this path

/// Event that is broadcast when workspace-related states change.
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize, PartialEq)]
pub enum WorkspaceEvent {
    WorkspaceCreated {
        id: WorkspaceId,
        name: String,
        persistent_id: Option<String>,
        icon_name: Option<String>,
        accent_color_hex: Option<String>,
        layout_type: WorkspaceLayoutType,
        position: usize, // Position in the ordered list
    },
    WorkspaceDeleted {
        id: WorkspaceId,
        windows_moved_to_workspace_id: Option<WorkspaceId>,
    },
    ActiveWorkspaceChanged {
        old_id: Option<WorkspaceId>,
        new_id: WorkspaceId,
    },
    WorkspaceRenamed { // Uses WorkspaceRenamedData from core::event_data
        id: WorkspaceId,
        old_name: String,
        new_name: String,
    },
    WorkspaceLayoutChanged { // Uses WorkspaceLayoutChangedData
        id: WorkspaceId,
        old_layout: WorkspaceLayoutType,
        new_layout: WorkspaceLayoutType,
    },
    WindowAddedToWorkspace { // Uses WindowAddedToWorkspaceData
        workspace_id: WorkspaceId,
        window_id: WindowIdentifier,
    },
    WindowRemovedFromWorkspace { // Uses WindowRemovedFromWorkspaceData
        workspace_id: WorkspaceId,
        window_id: WindowIdentifier,
    },
    WorkspaceOrderChanged(Vec<WorkspaceId>), // New, full ordered list of IDs
    WorkspacesReloaded {
        active_workspace_id: Option<WorkspaceId>,
        ordered_ids: Vec<WorkspaceId>,
    },
    WorkspaceIconChanged(WorkspaceIconChangedData),
    WorkspaceAccentChanged(WorkspaceAccentChangedData),
    // Event when the layout of windows within a workspace is recalculated
    WorkspaceLayoutApplied { // New event
        workspace_id: WorkspaceId,
        applied_layout: WorkspaceWindowLayout, // From window_management_policy
    }
}


#[async_trait]
pub trait WorkspaceConfigProvider: Send + Sync {
    async fn load_workspace_config(&self) -> Result<WorkspaceSetSnapshot, WorkspaceManagerError>;
    async fn save_workspace_config(&self, config_snapshot: &WorkspaceSetSnapshot) -> Result<(), WorkspaceManagerError>;
}

#[async_trait]
pub trait WorkspaceManagerService: Send + Sync {
    async fn new(
        config_provider: Arc<dyn WorkspaceConfigProvider>,
        broadcast_capacity: usize,
        ensure_unique_window_assignment: bool,
    ) -> Result<Arc<Self>, WorkspaceManagerError> where Self: Sized;

    async fn create_workspace(&self, params: WorkspaceCreationParams) -> Result<WorkspaceId, WorkspaceManagerError>;
    async fn delete_workspace(&self, id: WorkspaceId, fallback_id_for_windows: Option<WorkspaceId>) -> Result<(), WorkspaceManagerError>;

    async fn get_workspace(&self, id: WorkspaceId) -> Option<Workspace>; // Returns a clone
    async fn get_all_workspaces_ordered(&self) -> Vec<Workspace>; // Returns clones
    async fn get_active_workspace_id(&self) -> Option<WorkspaceId>;
    async fn get_active_workspace(&self) -> Option<Workspace>; // Returns a clone

    async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>;

    async fn assign_window_to_active_workspace(&self, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;
    async fn assign_window_to_specific_workspace(&self, workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;
    async fn remove_window_from_its_workspace(&self, window_id: &WindowIdentifier) -> Result<Option<WorkspaceId>, WorkspaceManagerError>; // Returns Option<WorkspaceId> it was removed from
    async fn move_window_to_specific_workspace(&self, target_workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;
    async fn get_workspace_for_window(&self, window_id: &WindowIdentifier) -> Option<WorkspaceId>;


    async fn rename_workspace(&self, id: WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>;
    async fn set_workspace_layout(&self, id: WorkspaceId, layout_type: WorkspaceLayoutType) -> Result<(), WorkspaceManagerError>;
    async fn set_workspace_icon(&self, id: WorkspaceId, icon_name: Option<String>) -> Result<(), WorkspaceManagerError>;
    async fn set_workspace_accent_color(&self, id: WorkspaceId, color_hex: Option<String>) -> Result<(), WorkspaceManagerError>;
    async fn set_workspace_persistent_id(&self, id: WorkspaceId, persistent_id: Option<String>) -> Result<(), WorkspaceManagerError>;


    async fn reorder_workspace(&self, workspace_id: WorkspaceId, new_index: usize) -> Result<(), WorkspaceManagerError>;

    /// Explicitly triggers saving the current workspace configuration.
    async fn save_configuration(&self) -> Result<(), WorkspaceManagerError>;

    fn subscribe_to_workspace_events(&self) -> broadcast::Receiver<WorkspaceEvent>;

    /// (Neu) Wird von der Systemschicht aufgerufen, wenn der Compositor ein neues Fenster erkennt.
    /// Die Domänenschicht entscheidet dann, welchem Workspace es zugeordnet wird (Policy).
    async fn handle_new_system_window(&self, window_id: WindowIdentifier, initial_properties: Option<crate::window_management_policy::types::WindowLayoutInfo>) -> Result<(), WorkspaceManagerError>;

    /// (Neu) Wird von der Systemschicht aufgerufen, wenn ein Fenster vom Compositor geschlossen/zerstört wird.
    async fn handle_system_window_closed(&self, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;

}
```

#### 2.2. Datenstrukturen

Definiert in `domain/src/workspaces/types.rs` und `config_types.rs`.

- **`WindowIdentifier`**:
    
    - Die `new` Methode gibt `Result<Self, WorkspaceCoreError>` zurück und validiert, dass die ID nicht leer ist und optional weiteren Kriterien (z.B. keine Leerzeichen) genügt.
- **`Workspace`**:
    
    - Felder: `id`, `name`, `persistent_id`, `layout_type`, `window_ids: HashSet<WindowIdentifier>`, `created_at`, `icon_name: Option<String>`, `accent_color_hex: Option<String>` (validierter Hex-String).
    - Die `new` Methode validiert Namen, `persistent_id` und `accent_color_hex`.
    - Methoden für `set_icon_name` und `set_accent_color_hex` hinzugefügt (mit Validierung für Hex).
- **`WorkspaceCreationParams`** (Neu in `types.rs`):
    
    Rust
    
    ```
    #[derive(Debug, Clone, Default)]
    pub struct WorkspaceCreationParams {
        pub name: Option<String>,
        pub persistent_id: Option<String>,
        pub icon_name: Option<String>,
        pub accent_color_hex: Option<String>,
        pub layout_type_hint: Option<WorkspaceLayoutType>, // Vorschlag für das Layout
        pub activate_when_created: bool,
    }
    ```
    
- **`WorkspaceSnapshot`** (in `config_types.rs`): Enthält nun auch `icon_name` und `accent_color_hex`.
    

#### 2.3. Events

- **`WorkspaceEvent`** (in `events.rs`): Erweitert um `WorkspaceIconChanged` und `WorkspaceAccentChanged` (mit entsprechenden Payloads aus `core::event_data`). `WorkspaceLayoutApplied` hinzugefügt.

#### 2.4. Fehlerbehandlung

- **`WorkspaceCoreError`**: Erweitert um `InvalidAccentColorFormat(String)`.
- **`WorkspaceManagerError`**: Keine wesentlichen Änderungen, aber die Fehlerpropagation von den neuen Methoden muss berücksichtigt werden.

#### 2.5. Interaktionen

- **Mit `domain::window_management_policy`**: Der `WorkspaceManagerService::handle_new_system_window` ruft den `WindowManagementPolicyService` auf, um die initiale Platzierung und den Workspace für ein neues Fenster zu bestimmen.
- **Mit `domain::global_settings_service`**: Der `WorkspaceManagerService` kann globale Einstellungen lesen, die das Workspace-Verhalten beeinflussen (z.B. Standard-Layout, ob dynamische Workspaces erlaubt sind).

---

### Modul 3: `domain::user_centric_services`

**Zweck:** KI-Interaktionsmanagement (inkl. Einwilligungen) und Benachrichtigungsmanagement.

**Referenz:** Siehe Dokument `B3 Domänenschicht.md` und vorherige Antwort.

#### 3.1. Untermodul: `domain::user_centric_services::ai_interaction`

##### APIs (Traits)

Rust

```
// domain/src/user_centric_services/ai_interaction/service_api.rs
use async_trait::async_trait;
use tokio::sync::broadcast;
use uuid::Uuid;

use super::types::{
    AIInteractionContext, AIConsent, AIModelProfile, AIDataCategory, AttachmentData,
    AIConsentStatus, InteractionHistoryEntry, AIModelCapability, AIConsentScope,
    AIInteractionContextInput, // New struct for initiating interaction
};
use super::errors::AIInteractionError;
use crate::shared_types::ApplicationId; // Assuming ApplicationId is a shared type

/// Event enum for AI interaction related events.
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize, PartialEq)]
pub enum AIInteractionEvent {
    InteractionInitiated {
        context: AIInteractionContext, // Send the whole initial context
    },
    ConsentUpdated {
        user_id: String, // Or a more specific User ID type
        model_id: String,
        category: AIDataCategory,
        new_status: AIConsentStatus,
        scope: AIConsentScope,
    },
    ContextUpdated { // For changes like history additions, attachment changes
        context_id: Uuid,
        // Optionally, more specific information about what changed
        // e.g., new_history_entry_count: usize, new_attachment_count: usize
    },
    ModelProfilesReloaded {
        profiles: Vec<AIModelProfile>,
    },
    // Event to request UI to show a consent dialog
    ConsentRequiredForInteraction {
        context_id: Uuid,
        model_id: String,
        required_categories: Vec<AIDataCategory>,
        prompt_message: Option<String>, // Message for the user
    },
}


#[async_trait]
pub trait AIConsentProvider: Send + Sync {
    async fn load_consents_for_user(&self, user_id: &str) -> Result<Vec<AIConsent>, AIInteractionError>;
    async fn save_consent(&self, consent: &AIConsent) -> Result<(), AIInteractionError>;
    async fn save_consents(&self, consents: &[AIConsent]) -> Result<(), AIInteractionError>; // Batch save
    async fn revoke_consent(&self, consent_id: Uuid, user_id: &str) -> Result<(), AIInteractionError>;
    async fn get_consent(&self, user_id: &str, model_id: &str, category: AIDataCategory) -> Result<Option<AIConsent>, AIInteractionError>;
}

#[async_trait]
pub trait AIModelProfileProvider: Send + Sync {
    async fn load_model_profiles(&self) -> Result<Vec<AIModelProfile>, AIInteractionError>;
    // Optional: async fn save_model_profiles(&self, profiles: &[AIModelProfile]) -> Result<(), AIInteractionError>;
}


#[async_trait]
pub trait AIInteractionLogicService: Send + Sync {
    async fn new(
        consent_provider: Arc<dyn AIConsentProvider>,
        profile_provider: Arc<dyn AIModelProfileProvider>,
        broadcast_capacity: usize,
        // Potentially a handle to GlobalSettingsService for default user_id or global AI prefs
        // settings_service: Arc<dyn GlobalSettingsService>,
    ) -> Result<Arc<Self>, AIInteractionError> where Self: Sized;

    /// Initiates a new AI interaction context based on input.
    /// May trigger a ConsentRequiredForInteraction event if consent is missing.
    async fn initiate_interaction(
        &self,
        user_id: &str, // Explicit user context
        input: AIInteractionContextInput,
    ) -> Result<Uuid, AIInteractionError>; // Returns context_id

    async fn get_interaction_context(&self, context_id: Uuid) -> Result<Option<AIInteractionContext>, AIInteractionError>;

    /// Provides or updates consent for a specific model and data categories.
    async fn provide_user_consent(
        &self,
        user_id: &str,
        model_id: String,
        category: AIDataCategory, // Grant/deny per category
        scope: AIConsentScope,
        consent_decision: bool, // true for granted, false for denied
        expiry_timestamp: Option<DateTime<Utc>>, // Only if scope is SpecificDuration
    ) -> Result<(), AIInteractionError>;

    /// Retrieves the effective consent status for a model and a set of data categories.
    /// This will check all relevant stored AIConsent objects.
    async fn get_effective_consent_status(
        &self,
        user_id: &str,
        model_id: &str,
        data_categories: &[AIDataCategory],
    ) -> Result<AIConsentStatus, AIInteractionError>;

    async fn add_attachment_to_context(
        &self,
        context_id: Uuid,
        attachment: AttachmentData,
    ) -> Result<(), AIInteractionError>;

    async fn update_interaction_history(
        &self,
        context_id: Uuid,
        entry: InteractionHistoryEntry,
    ) -> Result<(), AIInteractionError>;

    async fn list_available_models(&self) -> Result<Vec<AIModelProfile>, AIInteractionError>;
    async fn get_model_profile(&self, model_id: &str) -> Result<Option<AIModelProfile>, AIInteractionError>;
    async fn get_default_model(&self) -> Result<Option<AIModelProfile>, AIInteractionError>;
    async fn reload_model_profiles(&self) -> Result<(), AIInteractionError>; // Triggers reload via provider

    /// Retrieves all stored (non-revoked, non-expired) consents for a user and optionally a model.
    async fn get_active_user_consents(&self, user_id: &str, model_id_filter: Option<&str>) -> Result<Vec<AIConsent>, AIInteractionError>;

    fn subscribe_to_ai_events(&self) -> broadcast::Receiver<AIInteractionEvent>;
}
```

##### Datenstrukturen

- **`AIInteractionContextInput`** (Neu in `types.rs`):
    
    Rust
    
    ```
    #[derive(Debug, Clone, Default)]
    pub struct AIInteractionContextInput {
        pub initial_prompt: Option<String>,
        pub target_model_id: Option<String>, // If a specific model is requested
        pub required_data_categories: Vec<AIDataCategory>, // What the user/action intends to use
        pub initial_attachments: Option<Vec<AttachmentData>>,
        pub source_application_id: Option<ApplicationId>, // Context: which app initiated
        pub source_workspace_id: Option<WorkspaceId>,     // Context: on which workspace
    }
    ```
    
- **`AIModelProfile`**: `capabilities` wird zu `Vec<AIModelCapability>`. `api_key_secret_name` bleibt wichtig.
    
- **`AIConsent`**: Einwilligungen sind nun pro `(user_id, model_id, data_category)`. `AIConsentScope` hinzugefügt.
    
- **`AIInteractionContext`**: `history_entries: Vec<InteractionHistoryEntry>`.
    

##### Fehlerbehandlung (`AIInteractionError`)

- Erweitert um spezifischere Fehler für Consent-Prüfung und API-Key-Handling.

##### Interaktionen

- Persistenz von `AIConsent` und `AIModelProfile` über die `AIConsentProvider` und `AIModelProfileProvider` Traits, deren Implementierungen (`Filesystem...Provider`) `novade-core::config::ConfigServiceAsync` nutzen.
- Interaktion mit `system::mcp_client::SystemMcpService`: Die `AIInteractionLogicService` entscheidet _ob_ und _mit welchen Daten_ eine KI-Anfrage (z.B. `CallTool`) gestellt werden darf. Dann ruft sie die Systemschicht-API auf.

#### 3.2. Untermodul: `domain::user_centric_services::notifications_core`

**Zweck:** Kernlogik für das Verwalten von Benachrichtigungen.

**Referenz:** Siehe Dokument `B4 Domänenschicht.md` (Abschnitt 4.3) und vorherige Antwort.

##### APIs (Traits)

Rust

```
// domain/src/user_centric_services/notifications_core/service_api.rs
use async_trait::async_trait;
use tokio::sync::broadcast;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use std::collections::HashMap;

use crate::shared_types::ApplicationId; // Assuming ApplicationId
use crate::global_settings_and_state_management::GlobalSettingsService; // To get DND, history limits
use crate::notifications_rules::NotificationRulesEngine; // For rule processing

use super::types::{
    Notification, NotificationAction, NotificationUrgency, NotificationId,
    NotificationFilterCriteria, NotificationSortOrder, NotificationInput, NotificationStats,
    DismissReason,
};
use super::errors::NotificationError;


/// Event enum for notification related events.
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize, PartialEq)]
pub enum NotificationEvent {
    NotificationPosted {
        notification: Notification,
        was_suppressed_by_dnd: bool,
        was_suppressed_by_rule: Option<String>, // rule_id if suppressed
    },
    NotificationUpdated { // For updates to an existing notification (e.g., progress)
        notification: Notification,
    },
    NotificationDismissed {
        notification_id: NotificationId,
        reason: DismissReason,
    },
    NotificationActionInvoked {
        notification_id: NotificationId,
        action_key: String,
    },
    DoNotDisturbModeChanged {
        dnd_enabled: bool,
    },
    NotificationHistoryCleared,
    NotificationPopupExpired { // When an active notification is removed due to popup limit
        notification_id: NotificationId,
    },
}


#[async_trait]
pub trait NotificationService: Send + Sync {
    async fn new(
        rules_engine: Arc<dyn NotificationRulesEngine>,
        settings_service: Arc<dyn GlobalSettingsService>, // For DND, history limits
        broadcast_capacity: usize,
    ) -> Result<Arc<Self>, NotificationError> where Self: Sized;

    /// Posts a new notification to the system.
    /// Applies rules, checks DND, and manages active/history lists.
    async fn post_notification(&self, input: NotificationInput) -> Result<NotificationId, NotificationError>;

    /// Retrieves a specific notification (active or from history if found).
    async fn get_notification(&self, notification_id: NotificationId) -> Result<Option<Notification>, NotificationError>;

    /// Marks a notification as read. Does not remove it from active list.
    async fn mark_as_read(&self, notification_id: NotificationId) -> Result<(), NotificationError>;

    /// Dismisses a notification by the user or system.
    async fn dismiss_notification(&self, notification_id: NotificationId, reason: DismissReason) -> Result<(), NotificationError>;

    async fn get_active_notifications(
        &self,
        filter: Option<NotificationFilterCriteria>,
        sort_order: Option<NotificationSortOrder>,
        limit: Option<usize>,
    ) -> Result<Vec<Notification>, NotificationError>;

    async fn get_notification_history(
        &self,
        filter: Option<NotificationFilterCriteria>,
        sort_order: Option<NotificationSortOrder>,
        limit: Option<usize>,
        offset: Option<usize>,
    ) -> Result<Vec<Notification>, NotificationError>;

    async fn clear_history(&self) -> Result<(), NotificationError>;
    async fn clear_all_for_app(&self, app_id: &ApplicationId) -> Result<usize, NotificationError>; // Returns count dismissed

    async fn set_do_not_disturb(&self, enabled: bool) -> Result<(), NotificationError>;
    async fn is_do_not_disturb_enabled(&self) -> Result<bool, NotificationError>;

    async fn invoke_action(&self, notification_id: NotificationId, action_key: &str) -> Result<(), NotificationError>;

    async fn get_stats(&self) -> Result<NotificationStats, NotificationError>;

    fn subscribe_to_notification_events(&self) -> broadcast::Receiver<NotificationEvent>;
}
```

##### Datenstrukturen

- **`NotificationInput`** (Neu in `types.rs`): Struktur für das Erstellen von Benachrichtigungen. Enthält `app_name`, `replaces_id_domain: Option<NotificationId>` (umbenannt von `replaces_id` zur Klarheit, da D-Bus `u32` verwendet), `app_icon`, `summary`, `body`, `actions`, `urgency`, `category`, `hints`, `timeout_ms`, `transient`.
- **`Notification`**: `id` ist `NotificationId`. `replaces_id` entfernt. `timestamp` ist `DateTime<Utc>`. `hints` ist `HashMap<String, serde_json::Value>`.
- **`DismissReason`** (Enum in `events.rs` oder `types.rs`): `Expired`, `UserDismissed`, `AppClosed`, `ReplacedById(NotificationId)`, `SystemShutdown`, `AppScopeClear`, `RuleSuppressed`.

##### Interaktionen

- Der `DefaultNotificationService` hält `Arc<dyn NotificationRulesEngine>` und `Arc<dyn GlobalSettingsService>`.
- **`post_notification` Logik (verfeinert):**
    1. Generiert eine neue `NotificationId` und setzt `timestamp`.
    2. Erstellt `Notification` aus `NotificationInput`.
    3. Ruft `self.rules_engine.process_notification(temp_notification).await?`.
        - Wenn `RuleProcessingResult::Suppress { rule_id }`: Sendet `NotificationPostedEvent { ..., was_suppressed_by_rule: Some(rule_id), was_suppressed_by_dnd: false }`. Fügt ggf. zu `history` hinzu, wenn `final_notification.transient == false`. `Ok(final_notification.id)`.
    4. `let mut final_notification = allowed_notification_from_rules;`
    5. Prüft DND-Status (`self.settings_service.get_setting(SettingPath::Notifications(DndEnabledPath))`).
    6. Wenn DND aktiv und `final_notification.urgency < Critical`:
        - Sendet `NotificationPostedEvent { ..., was_suppressed_by_dnd: true, was_suppressed_by_rule: None }`.
        - Fügt ggf. zu `history` hinzu. `Ok(final_notification.id)`.
    7. Sonst (nicht unterdrückt):
        - Fügt zu `active_notifications` (FIFO-Limit `max_active_popups` aus Settings beachten). Wenn ein altes Popup entfernt wird, `NotificationPopupExpiredEvent` senden und es mit `DismissReason::Expired` schließen (verschiebt es ggf. in Historie).
        - Sendet `NotificationPostedEvent { ..., was_suppressed_by_dnd: false, was_suppressed_by_rule: None }`.
        - Wenn `!final_notification.transient`, zu `history` hinzufügen (FIFO-Limit `max_history_items` aus Settings beachten).
        - `Ok(final_notification.id)`.
- **`dismiss_notification`**: Sendet `NotificationDismissedEvent`. Wenn `!notification.transient` und Grund nicht `Expired` (oder ähnliche "automatische" Gründe), füge zu `history` hinzu.

---

### Modul 4: `domain::notifications_rules`

**Zweck:** Logik zur dynamischen Verarbeitung von Benachrichtigungen basierend auf konfigurierbaren Regeln.

**Referenz:** Siehe Dokument `B4 Domänenschicht.md` (Abschnitt 4.4) und vorherige Antwort.

#### 4.1. APIs (Traits)

Rust

```
// domain/src/notifications_rules/service_api.rs
use async_trait::async_trait;
use uuid::Uuid;

use crate::user_centric_services::notifications_core::types::Notification;
use crate::global_settings_and_state_management::GlobalSettingsService; // Trait

use super::types::{NotificationRuleSet, NotificationRule};
use super::errors::NotificationRulesError;

#[derive(Debug, Clone, PartialEq)]
pub enum RuleProcessingResult {
    Allow(Notification),      // Notification (ggf. modifiziert) erlauben
    Suppress { rule_id: Uuid }, // Benachrichtigung aufgrund dieser Regel-ID unterdrücken
}

#[async_trait]
pub trait NotificationRulesProvider: Send + Sync { // Moved from persistence_iface
    async fn load_rules(&self) -> Result<NotificationRuleSet, NotificationRulesError>;
    async fn save_rules(&self, rules: &NotificationRuleSet) -> Result<(), NotificationRulesError>;
}

#[async_trait]
pub trait NotificationRulesEngine: Send + Sync {
    async fn new(
        rules_provider: Arc<dyn NotificationRulesProvider>,
        settings_service: Arc<dyn GlobalSettingsService>, // To access global settings for conditions
    ) -> Result<Arc<Self>, NotificationRulesError> where Self: Sized;

    /// Lädt oder aktualisiert die im System verwendeten Regeln.
    async fn reload_rules(&self) -> Result<usize, NotificationRulesError>; // Returns number of loaded rules

    /// Verarbeitet eine eingehende Benachrichtigung gegen die geladenen Regeln.
    /// Kann die Benachrichtigung modifizieren oder deren Unterdrückung signalisieren.
    async fn process_notification(
        &self,
        notification: Notification, // Takes ownership to modify
    ) -> Result<RuleProcessingResult, NotificationRulesError>;

    /// Gibt die aktuell geladenen Regeln zurück (z.B. für UI zur Anzeige/Bearbeitung).
    async fn get_rules(&self) -> Result<NotificationRuleSet, NotificationRulesError>;

    /// Speichert einen neuen Satz von Regeln und lädt sie neu.
    async fn update_rules(&self, new_rules: NotificationRuleSet) -> Result<(), NotificationRulesError>;

    /// Reagiert auf Änderungen globaler Einstellungen, die Regelbedingungen beeinflussen könnten.
    /// Wird typischerweise von einem Event-Listener auf SettingChangedEvent aufgerufen.
    async fn on_global_setting_changed(&self, /* setting_key: &SettingPath, new_value: &serde_json::Value */);
}
```

#### 4.2. Datenstrukturen

Definiert in `domain/src/notifications_rules/types.rs`.

- **`RuleConditionField`**: Erweitert um:
    - `ActionCount(ComparisonOperator, usize)` (z.B. Anzahl Aktionen ist `GreaterThan` 0).
    - `HintExistsWithValue { hint_key: String, operator: RuleConditionOperator, expected_value: RuleConditionValue }` (kombiniert `HintExists` und `HintValueIs`).
- **`RuleAction`**:
    - `SetHint(String, serde_json::Value)`: Verwendet `serde_json::Value` für Flexibilität, um direkt mit den `hints` der `Notification` zu arbeiten.
    - `SetAccentColor(Option<novade_core::types::Color>)`.
- **`NotificationRule`**: `id` ist `Uuid`. `name` hinzugefügt.

#### 4.3. Interaktionen

- Der `DefaultNotificationRulesEngine` wird einen `Arc<dyn GlobalSettingsService>` halten, um `SettingIsTrue`-Bedingungen auszuwerten.
- Der `DefaultNotificationService` wird den `NotificationRulesEngine::process_notification` aufrufen.

---

### Modul 5: `domain::window_management_policy`

**Zweck:** Definition von High-Level-Regeln für Fensterplatzierung, Tiling, Snapping, Fokus etc.

**Referenz:** Vorherige Antwort.

#### 5.1. APIs (Traits)

Rust

```
// domain/src/window_management_policy/service_api.rs
use async_trait::async_trait;
use std::collections::HashMap;
use uuid::Uuid;

use novade_core::types::{RectInt, Size, Point};
use crate::workspaces::types::{WorkspaceId, WindowIdentifier, WorkspaceLayoutType}; // Corrected path
use crate::global_settings_and_state_management::GlobalSettingsService; // Corrected path

use super::types::{
    TilingMode, GapSettings, WindowSnappingPolicy, NewWindowPlacementStrategy, FocusPolicy,
    WorkspaceWindowLayout, WindowPolicyOverrides, WindowLayoutInfo, InitialWindowProperties, // New
};
use super::errors::WindowPolicyError;


#[derive(Debug, Clone, Default)]
pub struct WindowStateRequest { // Represents a client's or system's request to change state
    pub is_maximized: Option<bool>,
    pub is_fullscreen: Option<bool>,
    pub is_minimized: Option<bool>, // Minimieren ist eher eine Compositor-Aktion
    // pub new_tiling_mode: Option<TilingMode>, // Wenn Fenster individuelles Tiling anfordern könnten
}

#[derive(Debug, Clone, Default)]
pub struct WindowSizePositionRequest {
    pub position: Option<Point<i32>>, // Absolute Workspace-Koordinaten
    pub size: Option<Size<u32>>,
}


#[async_trait]
pub trait WindowManagementPolicyService: Send + Sync {
    async fn new(
        settings_service: Arc<dyn GlobalSettingsService>,
        // workspace_manager: Arc<dyn WorkspaceManagerService> // To get info about other windows/layout
    ) -> Result<Arc<Self>, WindowPolicyError> where Self: Sized;

    /// Berechnet das Layout für alle Fenster auf einem gegebenen Workspace.
    async fn calculate_workspace_layout(
        &self,
        workspace_id: WorkspaceId,
        windows_on_workspace: &[(WindowIdentifier, WindowLayoutInfo)], // Info über jedes Fenster
        available_area: RectInt, // Der für Fenster verfügbare Bereich
        workspace_current_tiling_mode: TilingMode,
        focused_window_id: Option<&WindowIdentifier>,
        window_specific_overrides: &HashMap<WindowIdentifier, WindowPolicyOverrides>,
    ) -> Result<WorkspaceWindowLayout, WindowPolicyError>;

    /// Bestimmt die initiale Geometrie und den Zustand für ein neues Fenster.
    async fn get_initial_window_properties(
        &self,
        window_id: &WindowIdentifier, // Die ID des neuen Fensters
        requested_properties: &WindowLayoutInfo, // Vom Client gewünschte Start-Eigenschaften
        parent_window_id: Option<&WindowIdentifier>,
        parent_geometry: Option<RectInt>,
        target_workspace_id: WorkspaceId,
        current_layout_on_target_ws: &WorkspaceWindowLayout, // Um Überlappungen zu vermeiden
        workspace_available_area: RectInt,
        window_specific_overrides: &Option<WindowPolicyOverrides>,
    ) -> Result<InitialWindowProperties, WindowPolicyError>; // Gibt mehr als nur Geometrie zurück

    /// Berechnet ein potenzielles "Snap"-Ziel.
    async fn calculate_snap_target(
        &self,
        moving_window_id: &WindowIdentifier,
        current_geometry: RectInt,
        other_windows_geometries: &[(&WindowIdentifier, &RectInt)], // Nur Geometrien anderer Fenster
        workspace_area: RectInt,
        // Snapping-Policy und Gaps werden intern vom Service über GlobalSettings geholt
    ) -> Option<RectInt>; // Option, da nicht immer ein Snap-Ziel existiert

    // Methoden zum Abrufen der effektiven Policies
    async fn get_effective_tiling_mode_for_workspace(&self, workspace_id: WorkspaceId) -> Result<TilingMode, WindowPolicyError>;
    async fn get_effective_gap_settings(&self) -> Result<GapSettings, WindowPolicyError>; // Global oder pro WS
    async fn get_effective_snapping_policy(&self) -> Result<WindowSnappingPolicy, WindowPolicyError>;
    async fn get_effective_focus_policy(&self) -> Result<FocusPolicy, WindowPolicyError>;
    async fn get_effective_new_window_placement_strategy(&self) -> Result<NewWindowPlacementStrategy, WindowPolicyError>;

    /// Entscheidet, ob eine angeforderte Zustandsänderung für ein Fenster erlaubt ist und
    /// wie sie ggf. modifiziert wird (z.B. Tiling vs. Floating bei Maximierungsanfrage).
    async fn determine_target_state_for_request(
        &self,
        window_id: &WindowIdentifier,
        current_geometry: RectInt,
        current_xdg_state: &smithay::wayland::shell::xdg::ToplevelState, // Vom Compositor
        requested_state_change: &WindowStateRequest,
        workspace_id: WorkspaceId,
        workspace_tiling_mode: TilingMode,
        workspace_available_area: RectInt,
        window_overrides: &Option<WindowPolicyOverrides>,
    ) -> Result<(RectInt, smithay::wayland::shell::xdg::ToplevelState), WindowPolicyError>; // Ziel-Geometrie und XDG-Status

    /// Entscheidet, ob eine angeforderte Größen-/Positionsänderung erlaubt ist und wie sie modifiziert wird.
    async fn determine_target_geometry_for_request(
         &self,
        window_id: &WindowIdentifier,
        current_geometry: RectInt,
        requested_change: &WindowSizePositionRequest, // Enthält neue Pos/Size
        workspace_id: WorkspaceId,
        workspace_tiling_mode: TilingMode,
        workspace_available_area: RectInt,
        window_overrides: &Option<WindowPolicyOverrides>,
        // Evtl. Infos über andere Fenster für Kollisionsvermeidung
    ) -> Result<RectInt, WindowPolicyError>; // Nur die Ziel-Geometrie
}
```

##### Datenstrukturen

- **`InitialWindowProperties`** (Neu in `types.rs`):
    
    Rust
    
    ```
    #[derive(Debug, Clone, PartialEq)]
    pub struct InitialWindowProperties {
        pub geometry: RectInt,
        pub focus_on_creation: bool, // Soll das Fenster sofort Fokus bekommen?
        pub assigned_workspace_id: WorkspaceId, // Auf welchem Workspace es platziert wird
        pub initial_xdg_state: smithay::wayland::shell::xdg::ToplevelState, // z.B. Maximized, Activated
    }
    ```
    
- **`WindowLayoutInfo`**: Enthält nun auch `is_dialog_or_utility: bool`, `prefers_floating: bool` (Hinweise vom Client/Systemschicht).
    
- **`WindowPolicyOverrides`**: `min_size_override`, `max_size_override` verwenden `novade_core::types::Size<u32>`.
    

---

Diese Verfeinerungen und detaillierten API-Signaturen, Datenstrukturen, Event-Payloads und Fehler-Enums für die Domänenschicht bilden eine solide Grundlage für die Implementierung. Der Schlüssel ist die klare Trennung der Verantwortlichkeiten und die präzise Definition der Schnittstellen zwischen den Modulen und zu den umgebenden Schichten. Als Nächstes würde die Systemschicht-Spezifikation mit einer ähnlichen Detailtiefe für die Wayland/Smithay-Integration folgen.

Abschließend zur Domänenschicht:

**Zusammenfassende Implementierungsschritte für die gesamte Domänenschicht:**

1. **Projekt-Setup:** Erstellen Sie das `novade-domain` Crate mit den definierten Abhängigkeiten in `Cargo.toml`.
2. **Modulstruktur anlegen:** Erstellen Sie die Verzeichnisstruktur für alle spezifizierten Hauptmodule (`shared_types`, `common_events`, `theming`, `global_settings_and_state_management`, `workspaces`, `window_management_policy`, `user_centric_services`, `notifications_rules`) und deren jeweilige Untermodule (`types.rs`, `errors.rs`, `service_api.rs`, `service_impl.rs` (oder `engine.rs`), `persistence_iface.rs`, `persistence.rs` etc.).
3. **`shared_types.rs` und `common_events.rs` implementieren:** Definieren und testen Sie diese grundlegenden, modulübergreifenden Typen zuerst.
4. **Implementierung jedes Hauptmoduls (iterativ):**
    - **`types.rs`**: Alle öffentlichen und internen Datenstrukturen und Enums mit `serde`-Attributen, `Default`-Implementierungen und ggf. Validierungslogik oder Hilfsmethoden definieren. Unit-Tests für Serialisierung, Deserialisierung und Defaults.
    - **`errors.rs`**: Das modulspezifische Fehler-Enum mit `thiserror`, allen Varianten, `#[error(...)]`-Nachrichten und `#[source]` / `#[from]`-Annotationen definieren. Unit-Tests für Fehlerdarstellung.
    - **`persistence_iface.rs` (falls zutreffend):** Den oder die Provider-Trait(s) für die Persistenzlogik definieren.
    - **`persistence.rs` (intern, falls zutreffend):** Die konkrete(n) Implementierung(en) des Provider-Traits erstellen (z.B. `Filesystem...Provider`), die `Arc<dyn novade_core::config::ConfigServiceAsync>` für Dateizugriffe nutzen. Unit-Tests mit Mocking für den `ConfigServiceAsync`.
    - **`service_api.rs` (oder im Modul-`mod.rs`):** Den öffentlichen Service-Trait (z.B. `ThemingEngineService`, `WorkspaceManagerService`) mit allen `async` Methoden definieren. Die zugehörigen Event-Strukturen (oder ein gemeinsames Event-Enum für das Modul) hier oder in einem separaten `events.rs` definieren.
    - **`service_impl.rs` (oder `engine.rs`, Modul-`mod.rs`):** Die konkrete Service-Implementierung (z.B. `DefaultThemingEngineService`) erstellen.
        - Die Struktur mit ihren Feldern für internen Zustand (`Arc<tokio::sync::RwLock<...>>` oder `Arc<tokio::sync::Mutex<...>>`) und Abhängigkeiten (andere Domänenservices, Provider, `tokio::sync::broadcast::Sender`) definieren.
        - Den `new(...)`-Konstruktor implementieren, der Abhängigkeiten injiziert, initiale Zustände lädt (asynchron via Provider) und den `broadcast::Sender` initialisiert.
        - Alle Methoden des Service-Traits `async` implementieren. Dabei die spezifizierte Logik, Fehlerbehandlung und Event-Auslösung beachten.
        - Umfassende Unit-Tests für jede Methode schreiben, inklusive aller Erfolgs- und Fehlerpfade. Abhängigkeiten (andere Services, Provider) mocken. Die korrekte Verwendung von `RwLock`/`Mutex` und die Logik des Event-Versands testen.
    - **Modul-`mod.rs`**: Alle internen Submodule (`use crate::...`) deklarieren und die öffentliche API des Moduls (Service-Trait, wichtige öffentliche Typen, Fehler-Enum, Event-Typen) re-exportieren.
5. **`novade-domain/src/lib.rs` pflegen:** Alle öffentlichen Module der Domänenschicht deklarieren und die wichtigsten Service-Traits, Fehler-Enums und Event-Typen für die Nutzung durch andere Crates (Systemschicht, UI-Schicht) re-exportieren. Optional eine übergeordnete `initialize_domain_layer`-Funktion und `DomainServices`-Struktur für eine gebündelte Initialisierung bereitstellen.
6. **Kontinuierliche Integration und Testen:** Stellen Sie sicher, dass `rustfmt` und `clippy` regelmäßig ausgeführt werden und alle Tests (inklusive neuer Tests für hinzugefügte Funktionalität) in einer CI-Pipeline laufen.

Durch die konsequente Befolgung dieser Schritte und der detaillierten Spezifikationen für jedes Modul wird eine robuste, wartbare und gut strukturierte Domänenschicht für NovaDE entstehen.

---

Als Nächstes folgt die Spezifikation der **Systemschicht** mit Fokus auf die Wayland/Smithay-Integration.

## NovaDE: Spezifikation und Implementierungsleitfaden – Systemschicht

Dieser Leitfaden beschreibt die Architektur, APIs, Datenstrukturen und Interaktionen der Systemschicht (Crate: `novade-system`). Diese Schicht ist verantwortlich für die Interaktion mit dem Betriebssystem, der Hardware und externen Diensten. Sie implementiert die "Mechanik" der Desktop-Umgebung, die von der Domänenschicht definiert wird.

**Allgemeine Prinzipien der Systemschicht:**

- **Sprache:** Rust (Edition 2021+).
- **Abstraktion:** Abstrahiert Low-Level-Systemdetails und stellt klar definierte Schnittstellen für die UI-Schicht und ggf. für Rückmeldungen an die Domänenschicht bereit.
- **Technologie-Integration:** Nutzt Bibliotheken wie Smithay (Wayland Compositor), `libinput-rs` (Eingabe), `zbus` (D-Bus), `pipewire-rs` (Audio), `mcp_client_rs` (KI-Protokoll).
- **Asynchronität:** Viele Operationen sind inhärent asynchron (Event-Loops für Wayland und D-Bus, I/O). `calloop` wird oft von Smithay für die Wayland-Ereignisschleife verwendet. `tokio` kann für `zbus` und andere asynchrone Aufgaben genutzt werden. Die Integration verschiedener Event-Loops/Runtimes muss sorgfältig gehandhabt werden (z.B. über `calloop::channel` oder spezielle Adapter).
- **Fehlerbehandlung:** `thiserror` für modulspezifische Fehler. Fehler aus externen Bibliotheken oder der Domänenschicht werden gewrappt.
- **Events:** Nutzt das `system::event_bridge`-Modul für die systeminterne Event-Kommunikation und um relevante Systemereignisse an die Domänen- oder UI-Schicht weiterzuleiten.

---

### Modulübersicht der Systemschicht (`system::*`)

Basierend auf der bestehenden Dokumentation (`3. System-Details.md`):

1. **`system::compositor`**: Wayland-Compositor-Logik (Smithay-basiert).
    - `core`: Zentrale `DesktopState`-Definition, Basis-Handler.
    - `surface_management`: Verwaltung von `WlSurface`-bezogenen Daten.
    - `shm`, `xdg_shell`, `layer_shell`, `decoration`, `output_management`, `input_method`, `screencopy`, `data_device`, `xwayland`.
    - `renderer_interface`, `drm_gbm_renderer`, `winit_renderer`.
2. **`system::input`**: Eingabeverarbeitung (`libinput-rs`-basiert).
    - `seat_manager`, `libinput_handler`, `keyboard`, `pointer`, `touch`, `gestures`.
3. **`system::dbus_interfaces`**: Schnittstellen zu System-D-Bus-Diensten und Bereitstellung eigener.
    - `common` (ConnectionManager, Fehler).
    - `upower_client`, `logind_client`, `network_manager_client`, `notifications_server`, `secrets_service_client`, `policykit_client`.
    - `xdg_desktop_portal_backend_logic` (umbenannt von `xdg_desktop_portal_handler` für Klarheit, da es die Logik HINTER dem Portal-D-Bus-Server ist).
4. **`system::audio_management`**: PipeWire-Client-Integration.
    - `types`, `errors`, `client` (mit `pipewire_listener`).
5. **`system::mcp_client`**: Implementierung des Model Context Protocol Clients.
    - `types`, `errors`, `connection_manager`, `service` (implementiert `SystemMcpService`-Trait).
6. **`system::window_mechanics`**: Technische Umsetzung des Fenstermanagements.
    - `types`, `errors`, `layout_applier`, `interactive_ops`, `focus_manager`.
7. **`system::power_management`**: DPMS, Interaktion mit logind für Suspend/Hibernate.
    - `types`, `errors`, `service` (mit `PowerManagementControl`-Trait).
8. **`system::event_bridge`**: Zentrale Event-Verteilung für System-interne und System-zu-Domäne Events.

---

### Modul 1: `system::compositor`

**Zweck:** Implementierung des Wayland-Compositors unter Verwendung des Smithay-Toolkits.

**Referenz:** Siehe Dokument `C1 System Implementierungsplan.md` für eine bereits sehr detaillierte Ausarbeitung. Die folgenden Punkte ergänzen/verfeinern diese.

#### 1.1. `system::compositor::core`

##### `DesktopState` Struktur (Verfeinerungen)

- **Renderer-Integration:**
    - `pub renderer: Option<Arc<Mutex<Gles2Renderer>>>` (oder ein anderer Renderer-Typ, der den `FrameRenderer`-Trait implementiert). `Option`, da der Renderer ggf. erst nach Backend-Initialisierung verfügbar ist. `Arc<Mutex<>>` für geteilten, veränderlichen Zugriff.
    - `pub damage_tracker: Option<Arc<Mutex<smithay::backend::renderer::damage::OutputDamageTracker>>>` (Smithay's Damage-Tracking).
- **Domain Service Handles:**
    - `pub window_management_policy_service: Arc<dyn domain::window_management_policy::WindowManagementPolicyService>`
    - `pub workspace_manager_service: Arc<dyn domain::workspaces::WorkspaceManagerService>`
    - `pub global_settings_service: Arc<dyn domain::global_settings_and_state_management::GlobalSettingsService>`
    - `pub system_event_bridge: Arc<crate::event_bridge::SystemEventBridge>` (für interne Events)
- **XWayland Integration:**
    - `pub xwayland_handle: Option<smithay::xwayland::XWaylandHandle>`
    - `pub xwm: Option<Arc<Mutex<smithay::xwayland::xwm::X11Wm>>>` (Der X11 Window Manager für XWayland-Fenster).
- **Wichtige Smithay-Zustände (mit `Arc<Mutex<>>` für Handler-Methoden, die `&self` nehmen):**
    - `compositor_state: CompositorState` (oft direkt, da Handler `&mut self` nehmen)
    - `shm_state: ShmState`
    - `xdg_shell_state: XdgShellState`
    - `seat_state: SeatState<Self>`
    - `output_manager_state: OutputManagerState`
    - `data_device_state: DataDeviceState`
    - `layer_shell_state: WlrLayerShellState` (für `wlr-layer-shell`)
    - `kde_decoration_state: KdeDecorationManagerState` (für `kde-server-decoration`) oder `xdg_decoration_state: XdgDecorationState` (für `xdg-decoration`)
    - Weitere Protokoll-States...
- **Fenster-Tracking:**
    - `pub space: Space<ManagedWindow>` (Smithay's `Space` für Layout und Rendering von `ManagedWindow`-Elementen).
    - `pub windows_map: HashMap<WindowIdentifier /* domain::WindowIdentifier */, ManagedWindowIdentifier /* compositor-intern */>`
    - `ManagedWindowIdentifier` könnte `usize` (von `Window::id()`) oder `WlSurface` sein, um schnell auf das `ManagedWindow`-Objekt im `Space` zugreifen zu können.
- **Aktive Operationen:**
    - `pub current_interactive_op: Option<crate::window_mechanics::types::InteractiveOpState>` (Für Move/Resize).
- **Initialisierung von `wayland_globals`**: Im `DesktopState::new` oder einer dedizierten Funktion werden die Globals (`WlCompositor`, `WlShm`, `XdgWmBase` etc.) mit `display_handle.create_global(...)` erstellt und deren `GlobalId`s in `NovaDEWaylandState` gespeichert. Die `ClientData` für den `DisplayHandle` wird ebenfalls initialisiert.

##### `CompositorHandler` für `DesktopState`

- **`commit(&mut self, surface: &WlSurface)`**:
    1. Ruft `smithay::wayland::compositor::dispatch_client_compositor_updates(surface, self)` auf, um Puffer- und Rollenänderungen intern zu verarbeiten.
    2. Ruft `SurfaceData` ab: `let surface_data_guard = SurfaceData::from_wl(surface).unwrap();`
    3. **Schadensverfolgung:**
        - `surface_data_guard.damage_from_client(surface_attributes.damage.clone());` (Schaden in Pufferkoordinaten speichern).
    4. **Fenster-Rendering aktualisieren:**
        - Finde das zugehörige `ManagedWindow` im `self.space`.
        - `self.space.damage_element(window_ref, Option<surface_local_damage>, Option<initial_damage_screen_coords>);` (Smithay's API hierfür genau prüfen).
    5. **Subsurface-Synchronisation:** Wenn `surface.is_sync_subsurface()`, werden Commits von synchronisierten Subsurfaces zurückgehalten, bis die Elternoberfläche committet. Smithay's `CompositorState` und `SurfaceData` (`current_pre_commit_buffer`, `current_post_commit_buffer`) helfen hier. `smithay::desktop::utils::commit_surfaces_upwards` oder `commit_surfaces_downwards` kann verwendet werden.
- **`new_surface(&mut self, surface: Resource<WlSurface>, client_data: &Arc<ClientCompositorData>)`**:
    1. `SurfaceData::new()` erstellen und in `surface.data_map().insert_if_missing_threadsafe(...)` speichern.
    2. `add_destruction_hook(&surface, |data_map: &UserDataMap| { ... SurfaceData und zugehörige Ressourcen (Renderer-Texturen) bereinigen ... });`

#### 1.2. `system::compositor::xdg_shell`

##### `XdgShellHandler` für `DesktopState`

- **`new_toplevel(&mut self, surface: ToplevelSurface)`**:
    1. `domain_window_id = WindowIdentifier::new_v4();` (aus Domäne oder hier generiert und Domäne benachrichtigt).
    2. Rufe `self.workspace_manager_service.handle_new_system_window(domain_window_id, initial_props_from_client).await;` (aus Domäne). Dies sollte Workspace-Zuweisung und initiale Policy-Anwendung beinhalten.
    3. Die Domäne gibt über ein Event oder direkten Rückgabewert (schwieriger mit `async` in Handler) die `InitialWindowProperties` zurück (inkl. Geometrie, initialer XDG-Zustand).
    4. Erstelle `ManagedWindow::new_toplevel(surface.clone(), domain_window_id, initial_properties)`.
    5. Füge `XdgToplevelSurfaceData::new()` zur `surface.wl_surface().data_map()`.
    6. Sende initiale Konfiguration: `surface.send_configure(initial_xdg_state_vec, Some(initial_geometry.size));`
    7. Füge `managed_window` zu `self.space.map_window(...)` und `self.windows_map` hinzu.
    8. Wenn `initial_properties.focus_on_creation`, rufe Fokuslogik auf.
- **`new_popup(&mut self, surface: PopupSurface, _client_data: &XdgWmBaseClientData)`**:
    1. Eltern-`ManagedWindow` finden.
    2. `initial_geom = self.window_management_policy_service.get_initial_popup_geometry(parent_geom, surface.get_positioner()).await;`
    3. `ManagedWindow::new_popup(...)` erstellen, zu `space`/`windows_map` hinzufügen.
    4. `surface.send_configure(initial_geom);`
- **`map_toplevel(&mut self, surface: &ToplevelSurface)`**:
    1. Finde `ManagedWindow`. Setze `is_mapped = true`.
    2. Rufe `self.window_management_policy_service.apply_layout_for_workspace(...)` auf, um das Layout des betroffenen Workspace neu zu berechnen (oder `handle_window_mapped` in der Policy).
    3. Sende `system_event_bridge.publish(WindowMappedEvent { ... });`
- **`ack_configure(&mut self, surface: Resource<WlSurface>, configure: XdgSurfaceConfigure)`**:
    1. `smithay::desktop::utils::handle_ack_configure(&configure, &surface, &mut self.space, ...)` kann hier nützlich sein.
    2. Validiere `configure.serial`.
    3. Wenn die Konfiguration eine Größenänderung war, die vom Compositor initiiert wurde (z.B. durch Tiling), und der Client sie bestätigt hat, ist der Zustand konsistent.
    4. Wenn der Client eine neue Größe/Zustand anfordert, der von der letzten `configure`-Nachricht abweicht, muss die Policy ggf. erneut konsultiert werden.
- **`*_request_*` Methoden (z.B. `toplevel_request_set_maximized`, `move_request`, `resize_request`):**
    1. Finde `ManagedWindow`.
    2. Für `move_request`/`resize_request`:
        - Erstelle `InteractiveOpState` in `self.current_interactive_op`.
        - `self.seat.start_pointer_grab(client_seat, &surface_that_requested_grab, serial, PointerMoveResizeGrab::new(op_state, self.window_management_policy_service.clone(), self.system_event_bridge.clone()));`
    3. Für Zustandsänderungen (`set_maximized`, etc.):
        - Rufe `self.window_management_policy_service.determine_target_state_for_request(...)` auf.
        - Das Ergebnis (`target_geometry`, `target_xdg_state`) wird verwendet, um `surface.send_configure(...)` aufzurufen.
        - Aktualisiere `ManagedWindow.current_geometry` und interne Zustände.
        - `self.space.damage_window(...)`.
- **`toplevel_destroyed`, `popup_destroyed`**: Fenster aus `space` und `windows_map` entfernen. `self.workspace_manager_service.handle_system_window_closed(...)` aufrufen.

##### `PointerMoveResizeGrab` (Implementierung von `PointerGrab<DesktopState>`)

- **`motion(...)`**:
    1. Berechne neue Geometrie (`new_geom`) basierend auf `op_state.initial_window_geometry`, `event.location() - op_state.start_pointer_pos_global` und `op_state.op_type`.
    2. Rufe `self.window_policy_service.calculate_snap_target(..., new_geom, ...).await` auf. Wenn Some, verwende Snap-Ziel.
    3. Prüfe Min/Max-Größen aus `op_state.initial_window_size_constraints`.
    4. **Visuelles Feedback (ohne Configure an Client):**
        - `op_state.window_arc.current_geometry = clamped_snapped_geom;` (Aktualisiere nur die interne Repräsentation).
        - `self.desktop_state_accessor().lock().unwrap().space.damage_window(&op_state.window_arc, old_geom_for_damage, new_geom_for_damage);` (Schaden für alte und neue Position/Größe).
- **`button(...)`**:
    1. Wenn Maustaste losgelassen (`event.state == Released`):
        - `final_geometry = op_state.window_arc.current_geometry;`
        - Sende `configure` an Client mit `final_geometry` und dem aktuellen XDG-Zustand.
        - `pointer_handle.unset_grab(serial, time);`
        - `self.desktop_state_accessor().lock().unwrap().current_interactive_op = None;`
        - `self.system_event_bridge.publish(InteractiveOpEnded { ... final_geometry ... });`

#### 1.3. Integration mit der Domänenschicht

- **Fenstererstellung:**
    1. `XdgShellHandler::new_toplevel` empfängt Wayland-Anfrage.
    2. Ruft `workspace_manager_service.handle_new_system_window(window_id_compositor, client_hints)` auf.
    3. `WorkspaceManagerService` (Domäne):
        - Entscheidet (ggf. mit `WindowManagementPolicyService`), welchem Workspace das Fenster zugeordnet wird.
        - Sendet `WorkspaceEvent::WindowAddedToWorkspace`.
    4. `WindowManagementPolicyService` (Domäne):
        - Berechnet `InitialWindowProperties` (Geometrie, initialer XDG-Zustand, Fokus).
    5. `XdgShellHandler::new_toplevel` (zurück im Compositor):
        - Nimmt `InitialWindowProperties`.
        - Erstellt `ManagedWindow`, fügt zu `Space` hinzu.
        - Sendet `configure` an Wayland-Client.
- **Fokuswechsel (durch Benutzerklick in UI initiiert):**
    1. UI-Schicht -> `system::input::seat_manager::set_keyboard_focus(target_window_domain_id)`.
    2. `system::input` -> `system::window_mechanics::focus_manager::set_application_focus(target_window_domain_id)`.
    3. `set_application_focus` (System):
        - Findet `ManagedWindow` für `target_window_domain_id`.
        - Ruft `smithay::input::KeyboardHandle::set_focus(Some(target_wl_surface), serial)`.
        - Setzt `XDG_TOPLEVEL_STATE_ACTIVATED` für neues Fenster, entfernt für altes, sendet `configure`.
        - Informiert `workspace_manager_service.notify_focus_changed(target_window_domain_id)` (Domäne).
- **Layout-Änderung (durch Workspace oder Policy initiiert):**
    1. `WorkspaceManagerService` (Domäne) ändert `Workspace.layout_type` oder empfängt eine Policy-Änderung.
    2. `WorkspaceManagerService` ruft `window_management_policy_service.calculate_workspace_layout(...)` auf.
    3. Das Ergebnis (`WorkspaceWindowLayout`) wird als Event (`WorkspaceLayoutApplied`) publiziert.
    4. `system::window_mechanics::layout_applier` (System) empfängt dieses Event (oder wird direkt von Domäne/anderem Systemteil aufgerufen).
    5. `layout_applier` iteriert über die Geometrien im `WorkspaceWindowLayout`:
        - Findet `ManagedWindow`s im `DesktopState::space`.
        - Sendet `configure`-Events an die Wayland-Clients mit den neuen Geometrien/Zuständen.
        - Aktualisiert `ManagedWindow.current_geometry` und `DesktopState::space`.

#### 1.4. Wayland-Protokolle und Smithay-Typen

- **`xdg-shell`**:
    - `XdgShellState`, `XdgShellHandler` (in `DesktopState` implementiert).
    - `ToplevelSurface`, `PopupSurface` (Smithay-Typen, gewrappt in `ManagedWindow`).
    - `XdgToplevelSurfaceData`, `XdgPopupSurfaceData` (in `WlSurface::data_map()` für Zustände wie `min_size`, `max_size`, `parent`, `app_id`, `title`).
    - `PositionerState` für Popups.
- **`wlr-layer-shell-unstable-v1`**:
    - `WlrLayerShellState`, `LayerShellHandler` (in `DesktopState` implementiert).
    - `LayerSurfaceData` (in `WlSurface::data_map()`).
    - `LayerSurface` (Smithay-Typ, kann in `ManagedWindow` oder eine separate Struktur gewrappt werden).
    - Verwaltung von Layern (Background, Bottom, Top, Overlay) und Ankern.
- **`wlr-output-management-unstable-v1`**:
    - `OutputManagerState` (Teil von `DesktopState.output_manager_state`, welches `smithay::wayland::output::OutputManagerState` ist).
    - Die Handler (`GlobalDispatch`, `Dispatch` für `ZwlrOutputManagerV1`, `ZwlrOutputHeadV1`, `ZwlrOutputModeV1`, `ZwlrOutputConfigurationV1`) werden typischerweise von `DesktopState` oder einer dedizierten `OutputManagementLogic`-Struktur implementiert.
    - Interaktion mit `smithay::output::Output`-Objekten, die im `DesktopState.space` oder einer separaten Liste verwaltet werden.
- **`DisplayHandle`**:
    - Wird zum Erstellen von Globals (`create_global()`) und zum Verwalten von Clients (`dispatch_clients()`, `flush_clients()`) verwendet. Ist Teil von `DesktopState`.
- **`CompositorState`**:
    - Verwaltet `wl_compositor` und `wl_subcompositor` Globals.
    - `SurfaceAttributes` (in `WlSurface::data_map()` via `SurfaceCachedState`) hält Puffer, Skalierung, Transformation, Schaden.
- **Allgemeine Hinweise zu Smithay:**
    - **UserDataMap**: Jedes Wayland-Objekt (`WlSurface`, `WlSeat`, Protokoll-Objekte) hat eine `UserDataMap`, um anwendungsspezifische Daten zu speichern (z.B. `SurfaceData`, `XdgToplevelSurfaceData`).
    - **Handler-Traits**: Die Logik des Compositors wird durch Implementierung der Handler-Traits von Smithay (z.B. `CompositorHandler`, `XdgShellHandler`, `SeatHandler`) in `DesktopState` definiert.
    - **Ereignisschleife (`calloop`):** Wayland-Events, libinput-Events und Timer werden in einer `calloop`-Ereignisschleife verarbeitet. `DisplayHandle::dispatch_clients(&mut desktop_state)` wird im Callback für Wayland-FDs aufgerufen.

#### 1.5. Fehlerbehandlung

- **`CompositorError`** (aus `system::compositor::errors`):
    - Spezifische Varianten für Fehler bei der Globalerstellung, Rollenzuweisung, Oberflächen-Handling, Renderer-Initialisierung, XWayland-Probleme.
    - Wrappt Fehler von Smithay (z.B. `SurfaceRoleError`, `smithay::backend::renderer::RendererError`).
- Fehler aus der Domänenschicht (z.B. `WindowPolicyError`, `WorkspaceManagerError`) werden von der Systemschicht ggf. abgefangen, geloggt und in einen `CompositorError` oder einen anderen Systemfehler umgewandelt, falls die UI eine generische Fehlermeldung erhalten soll.

---

**Nächste Schritte für die Spezifikation der Systemschicht:**

Die Module `system::input`, `system::dbus_interfaces`, `system::audio_management` usw. würden mit ähnlicher Detailtiefe spezifiziert, wobei immer die Interaktion mit den relevanten externen Bibliotheken (libinput, zbus, pipewire-rs), den Smithay-Abstraktionen (falls zutreffend) und der Domänenschicht im Vordergrund steht.

**Beispiel für `system::input` (Kurzform):**

- **`LibinputInputBackend`**: Initialisierung mit einer `Session` (z.B. `LogindSession` für DRM oder eine Dummy-Session für Winit).
- **`SeatHandler` in `DesktopState`**:
    - `keyboard_focus`, `pointer_focus`, `touch_focus` verwalten.
    - `cursor_image`: Setzt `DesktopState.current_cursor_status` (Smithay-Typ), der vom Renderer verwendet wird. Für thematisierte Cursor Interaktion mit `domain::theming` (oder einer Cursor-Theming-Lib).
- **Event-Verarbeitung im `calloop`-Callback für libinput-FD:**
    - `InputEvent` von `LibinputInputBackend.dispatch_input()`.
    - `KeyboardKeyEvent`:
        - `XkbKeyboardData` (aus `DesktopState.keyboard_data_map`) verwenden.
        - `xkbcommon::State::update_key()`.
        - `smithay::input::KeyboardHandle::input()` und `modifiers()` aufrufen, um Wayland-Events an Client zu senden.
        - Tastenwiederholung mit `calloop::Timer`.
    - `PointerMotionEvent`:
        - `DesktopState.pointer_location` aktualisieren.
        - `smithay::desktop::utils::surface_under_pointer()` oder äquivalente Logik mit `DesktopState.space` verwenden, um Ziel-Surface zu finden.
        - `smithay::input::PointerHandle::motion()` aufrufen (sendet `enter`/`leave`/`motion`).
    - `PointerButtonEvent`:
        - `PointerHandle::button()`.
        - Fokuslogik auslösen: `system::window_mechanics::focus_manager::set_application_focus(...)`.
        - Wenn auf Fensterdekoration oder spezifischen Bereich geklickt: Interaktive Operation starten (`system::window_mechanics::interactive_ops::start_interactive_move(...)`).

Diese detaillierte Vorgehensweise stellt sicher, dass alle Aspekte der Integration und Logik klar definiert sind, bevor die eigentliche Implementierung beginnt.

### Protokoll-Implementierungen (D-Bus, XDG Desktop Portals, MCP)

#### 1. D-Bus (`system::dbus_interfaces`)

- **Spezifikation:** Siehe vorherige Antwort für `system::dbus_interfaces` und dessen Submodule (`common`, `upower_client`, `logind_client`, `network_manager_client`, `notifications_server`, `secrets_service_client`, `policykit_client`, `xdg_desktop_portal_backend_logic`).
- **Integration in Domänen-/Systemschichten:**
    - **Clients (z.B. `UPowerClientService`):**
        1. Werden typischerweise als Teil des `DesktopState` oder als separate, langlebige `tokio::task`s initialisiert.
        2. Nutzen den `DBusConnectionManager` für die `Arc<zbus::Connection>`.
        3. Erstellen Proxies für die Ziel-D-Bus-Interfaces.
        4. **Methodenaufrufe:** Öffentliche Methoden des Service-Wrappers (z.B. `UPowerClientService::is_on_battery()`) rufen asynchron Methoden auf dem `zbus`-Proxy auf. Fehler (`zbus::Error` oder `zbus::fdo::Error`) werden in den modulspezifischen `DBusInterfaceError` umgewandelt.
        5. **Signal-Handling:**
            - Beim Initialisieren (`initialize_and_listen()`) wird ein `tokio::task` gestartet, der `proxy.receive_my_signal().await?` in einer Schleife aufruft.
            - Empfangene Signale werden in interne System-Events (via `SystemEventBridge`) oder direkte Aufrufe an Domänenservices übersetzt. Beispiel: `UPowerProxy::PropertiesChanged` für `LidIsClosed` wird zu `SystemEventBridge::publish(SystemLayerEvent::Logind(LogindEvent::LidStateChanged{is_closed}))`.
    - **Server (z.B. `FreedesktopNotificationsServer`):**
        1. Wird als D-Bus-Objekt mit `#[dbus_interface(...)]` implementiert.
        2. Nimmt eine Referenz (`Arc<Mutex<dyn domain::user_centric_services::NotificationService>>`) auf den entsprechenden Domänenservice entgegen.
        3. D-Bus-Methodenaufrufe (z.B. `Notify`) werden an den Domänenservice delegiert.
        4. Um D-Bus-Signale zu senden (z.B. `NotificationClosed`), abonniert der D-Bus-Server-Task die entsprechenden Domänen-Events (z.B. `domain::user_centric_services::NotificationEvent::NotificationDismissed`) und sendet dann das D-Bus-Signal über den `zbus::SignalContext`.
        5. Der D-Bus-Server-Task wird beim Start von NovaDE gestartet und registriert den Dienst auf dem Session-Bus.

#### 2. XDG Desktop Portals (`system::portals` oder integriert in `system::dbus_interfaces::xdg_desktop_portal_backend_logic`)

- **Spezifikation:** Die _Backend_-Logik für Portale. Die eigentlichen D-Bus-Interfaces (`org.freedesktop.portal.FileChooser`, `Screenshot`, etc.) werden von einem separaten XDG-Desktop-Portal-Implementierungsprozess (z.B. `xdg-desktop-portal-novade`) bereitgestellt, der dann mit dieser Backend-Logik über D-Bus oder eine andere IPC kommuniziert ODER NovaDE implementiert diese Portal-D-Bus-Interfaces direkt als Teil seiner D-Bus-Services. Angesichts der Komplexität ist eine direkte Implementierung der Portal-D-Bus-Interfaces in NovaDE über `zbus` der bevorzugte Ansatz für eine engere Integration.
- **Implementierung der Portal-D-Bus-Interfaces (z.B. `org.freedesktop.portal.FileChooser`):**
    - Ein Struct (z.B. `NovaFileChooserPortal`) implementiert das D-Bus-Interface mit `#[dbus_interface(name = "org.freedesktop.portal.FileChooser")]`.
    - Benötigt Zugriff auf:
        - `system::compositor` (oder eine Abstraktion darüber): Um `parent_window` Handles zu validieren und Fensterinformationen zu erhalten.
        - UI-Schicht (indirekt): Um den nativen Dateiauswahldialog anzuzeigen. Dies geschieht typischerweise, indem der Portal-D-Bus-Handler einen Befehl an die UI-Schicht sendet (z.B. über einen dedizierten internen Mechanismus oder einen speziellen D-Bus-Aufruf an einen UI-eigenen Dienst), um den Dialog anzuzeigen. Die Antwort des Dialogs wird dann asynchron zurück an den Portal-Handler gemeldet.
    - **Methode `OpenFile(handle_token, parent_window, title, options)`:**
        1. Generiert einen eindeutigen Request-Handle für `zbus`.
        2. Sendet eine Anfrage an die UI-Schicht, einen Dateiauswahldialog mit den gegebenen Optionen anzuzeigen. Diese Anfrage enthält den Request-Handle.
        3. Gibt den D-Bus-Pfad des Request-Objekts zurück an den Portal-Client.
        4. Der Portal-Client (Anwendung) lauscht auf das `Response(u32, a{sv})`-Signal auf diesem Request-Objekt.
        5. Wenn die UI-Schicht die Auswahl des Benutzers erhält (Dateien oder Abbruch), sendet sie das Ergebnis (zusammen mit dem Request-Handle) zurück an den Portal-D-Bus-Handler.
        6. Der Portal-D-Bus-Handler sendet dann das `Response`-Signal auf dem Request-Objekt mit dem Ergebnis.
- **Integration:** Die Portal-D-Bus-Objekte werden auf dem Session-Bus registriert. Die Implementierung von `system::compositor::core::DesktopState` könnte eine Methode `get_parent_window_info(handle_string: &str) -> Option<ParentWindowDetails>` bereitstellen, die von den Portal-Handlern genutzt wird.

#### 3. Model Context Protocol (MCP) (`system::mcp_client`)

- **Spezifikation:** Siehe vorherige Antwort für `system::mcp_client` (Submodule `types`, `errors`, `connection_manager`, `service`).
- **Integration:**
    1. Der `DefaultSystemMcpService` (implementiert `SystemMcpService`-Trait) wird im `DesktopState` oder als globaler Service initialisiert.
    2. Die `server_configs: Vec<McpServerConfig>` werden von `domain::global_settings_service` geladen und an `DefaultSystemMcpService::configure_servers()` übergeben.
    3. Die Domänenschicht (`domain::user_centric_services::AIInteractionLogicService`) ruft Methoden des `SystemMcpService` auf (z.B. `call_tool`).
    4. `SystemMcpService`:
        - Nutzt den `McpConnectionManager`, um eine Verbindung zum entsprechenden `server_id` herzustellen (inkl. API-Key-Abruf via `SecretsServiceClientService`).
        - Sendet die MCP-Anfrage über die `mcp_client_rs::McpClient`-Instanz.
        - Verwendet `tokio::time::timeout` für Anfragen.
        - Verarbeitet die Antwort:
            - Erfolgreiche Antworten (`McpToolResult`) werden an den Aufrufer (Domäne) zurückgegeben und ein `McpClientSystemEvent::McpToolCallSuccessful` wird publiziert.
            - MCP-Fehlerantworten (`McpErrorResponse`) werden in `McpSystemClientError::McpServerErrorReply` umgewandelt, zurückgegeben, und ein `McpClientSystemEvent::McpRequestFailed` wird publiziert.
            - Kommunikationsfehler/Timeouts führen zu `McpSystemClientError` und `McpClientSystemEvent::McpCommunicationError`.
        - Der im `McpConnection` gestartete Task lauscht auf `client.receive_message()` für serverseitige Notifications und publiziert `McpClientSystemEvent::McpNotificationReceived`.

Diese detaillierten Integrationspläne stellen sicher, dass die Protokolle korrekt in die Schichtenarchitektur von NovaDE eingebettet werden und die Kommunikation zwischen den Komponenten klar definiert ist.

## Domänenschicht-Spezifikationen (`domain::*`)

Die Domänenschicht kapselt die Geschäftslogik und den Kernzustand von NovaDE. Sie ist unabhängig von UI-Details und direkten Systeminteraktionen.

### 1. Modul: `domain::theming`

#### 1.1. API (`ThemingEngineService` Trait)

- **Datei:** `domain/src/theming/service_api.rs`
- **Methoden (Signaturen wie in vorherigen Antworten, `async` wo zutreffend):**
    - `async fn new(...) -> Result<Arc<Self>, ThemingError>`
    - `async fn get_current_theme_state(&self) -> AppliedThemeState`
    - `async fn get_available_themes(&self) -> Vec<ThemeDefinition>`
    - `async fn get_current_configuration(&self) -> ThemingConfiguration`
    - `async fn update_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>`
    - `async fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>`
    - `fn subscribe_to_theme_changes(&self) -> broadcast::Receiver<ThemeChangedEvent>`
- **Abhängigkeiten für `new()`:** `initial_config: ThemingConfiguration`, `theme_load_paths: Vec<PathBuf>`, `token_load_paths: Vec<PathBuf>`, `config_service: Arc<dyn novade_core::config::ConfigServiceAsync>`, `broadcast_capacity: usize`.

#### 1.2. Datenstrukturen (`domain::theming::types`)

- **`TokenIdentifier`**, **`TokenValue`**, **`RawToken`**, **`TokenSet`** (`BTreeMap`), **`ThemeIdentifier`**, **`ColorSchemeType`**, **`AccentColor`** (mit `novade_core::types::Color`), **`ThemeVariantDefinition`**, **`ThemeDefinition`** (mit `accentable_tokens: Option<HashMap<TokenIdentifier, AccentModificationType>>`), **`AppliedThemeState`** (mit `resolved_tokens: BTreeMap<TokenIdentifier, String>`), **`ThemingConfiguration`** (mit `selected_accent_color: Option<novade_core::types::Color>`).
- **`AccentModificationType` Enum:** `DirectReplace`, `Lighten(f32)`, `Darken(f32)`, `DesaturateAndTint(f32)`.

#### 1.3. Events (`domain::theming::service_api` oder `events.rs`)

- **`ThemeChangedEvent` Struct:** `new_state: AppliedThemeState`, `old_state: Option<AppliedThemeState>`.

#### 1.4. Fehlerbehandlung (`domain::theming::errors`)

- **`ThemingError` Enum:** Varianten für `TokenFileParseError`, `TokenFileIoError`, `InvalidTokenData`, `InvalidTokenValue`, `CyclicTokenReference`, `ThemeFileLoadError`, `ThemeFileIoError`, `InvalidThemeData`, `ThemeNotFound`, `MissingTokenReference`, `MaxReferenceDepthExceeded`, `ThemeApplicationError`, `FallbackThemeLoadError`, `InitialConfigurationError`, `InternalStateError`, `EventSubscriptionError`, `AccentColorApplicationError`, `TokenResolutionError`, `FilesystemError(#[from] novade_core::errors::CoreError)`.

#### 1.5. Interaktionen und Datenfluss

1. **Initialisierung (`ThemingEngineService::new`):**
    - Lädt globale Tokens und verfügbare Themes asynchron über den `config_service` von den `token_load_paths` und `theme_load_paths`.
    - Validiert Tokens (Zyklen) und Themes (Referenzen).
    - Berechnet den initialen `AppliedThemeState` basierend auf `initial_config` oder einem einkompilierten Fallback-Theme.
    - Initialisiert den `tokio::sync::broadcast::Sender` für `ThemeChangedEvent`.
2. **Konfigurationsupdate (`update_configuration`):**
    - Nimmt `ThemingConfiguration` entgegen.
    - Löst die Token-Resolution-Pipeline aus:
        1. Basissatz globaler Tokens.
        2. Merge mit Tokens des ausgewählten Themes (`ThemeDefinition.base_tokens`).
        3. Merge mit Tokens der passenden `ThemeVariantDefinition` (basierend auf `preferred_color_scheme`).
        4. Wendet `selected_accent_color` an (modifiziert Tokens gemäß `accentable_tokens` und `AccentModificationType`).
        5. Merge mit `custom_user_token_overrides`.
        6. Rekursive Auflösung aller `TokenValue::Reference` zu finalen String-Werten (CSS-kompatibel) mit Zyklenerkennung und Tiefenbegrenzung.
    - Erzeugt neuen `AppliedThemeState`.
    - Wenn sich der Zustand geändert hat, wird der interne Zustand aktualisiert und ein `ThemeChangedEvent` gesendet.
    - Der aufgelöste Zustand kann gecacht werden (Schlüssel: `(ThemeIdentifier, ColorSchemeType, Option<AccentColorHash>, UserOverridesHash)`).
3. **Zustandsverwaltung:** `ThemingEngineInternalState` (geschützt durch `tokio::sync::Mutex`) hält `current_config`, `available_themes`, `global_raw_tokens`, `applied_state`, Lade-Pfade, `config_service` und den Cache.

### 2. Modul: `domain::workspaces`

#### 2.1. API (`WorkspaceManagerService` Trait)

- **Datei:** `domain/src/workspaces/service_api.rs`
- **Methoden (Signaturen wie in vorherigen Antworten, `async` wo zutreffend):**
    - `async fn new(...) -> Result<Arc<Self>, WorkspaceManagerError>`
    - `async fn create_workspace(&self, params: WorkspaceCreationParams) -> Result<WorkspaceId, WorkspaceManagerError>`
    - `async fn delete_workspace(&self, id: WorkspaceId, fallback_id_for_windows: Option<WorkspaceId>) -> Result<(), WorkspaceManagerError>`
    - `async fn get_workspace(&self, id: WorkspaceId) -> Option<Workspace>` (gibt Klon zurück)
    - `async fn get_all_workspaces_ordered(&self) -> Vec<Workspace>` (gibt Klone zurück)
    - `async fn get_active_workspace_id(&self) -> Option<WorkspaceId>`
    - `async fn get_active_workspace(&self) -> Option<Workspace>` (gibt Klon zurück)
    - `async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>`
    - `async fn assign_window_to_active_workspace(&self, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>` (interne Logik: ruft `assignment::assign_window_to_workspace`)
    - `async fn assign_window_to_specific_workspace(&self, workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>`
    - `async fn remove_window_from_its_workspace(&self, window_id: &WindowIdentifier) -> Result<Option<WorkspaceId>, WorkspaceManagerError>`
    - `async fn move_window_to_specific_workspace(&self, target_workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>`
    - `async fn get_workspace_for_window(&self, window_id: &WindowIdentifier) -> Option<WorkspaceId>`
    - `async fn rename_workspace(&self, id: WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>`
    - `async fn set_workspace_layout(&self, id: WorkspaceId, layout_type: WorkspaceLayoutType) -> Result<(), WorkspaceManagerError>`
    - `async fn set_workspace_icon(&self, id: WorkspaceId, icon_name: Option<String>) -> Result<(), WorkspaceManagerError>`
    - `async fn set_workspace_accent_color(&self, id: WorkspaceId, color_hex: Option<String>) -> Result<(), WorkspaceManagerError>`
    - `async fn set_workspace_persistent_id(&self, id: WorkspaceId, persistent_id: Option<String>) -> Result<(), WorkspaceManagerError>`
    - `async fn reorder_workspace(&self, workspace_id: WorkspaceId, new_index: usize) -> Result<(), WorkspaceManagerError>`
    - `async fn save_configuration(&self) -> Result<(), WorkspaceManagerError>`
    - `fn subscribe_to_workspace_events(&self) -> broadcast::Receiver<WorkspaceEvent>`
    - `async fn handle_new_system_window(&self, window_id: WindowIdentifier, initial_properties: Option<crate::window_management_policy::types::WindowLayoutInfo>) -> Result<(), WorkspaceManagerError>`
    - `async fn handle_system_window_closed(&self, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>`
- **`WorkspaceConfigProvider` Trait:**
    - `async fn load_workspace_config(&self) -> Result<WorkspaceSetSnapshot, WorkspaceManagerError>` (Fehlertyp angepasst)
    - `async fn save_workspace_config(&self, config_snapshot: &WorkspaceSetSnapshot) -> Result<(), WorkspaceManagerError>`

#### 2.2. Datenstrukturen (`domain::workspaces::types`, `config_types.rs`)

- **`WorkspaceId`**, **`WindowIdentifier`** (mit `Result`-basiertem `new`), **`WorkspaceLayoutType`**.
- **`Workspace` Struct:** Enthält `id`, `name`, `persistent_id`, `layout_type`, `window_ids: HashSet<WindowIdentifier>`, `created_at`, `icon_name`, `accent_color_hex`. Methoden validieren Eingaben (Name, persistente ID, Hex-Farbe).
- **`WorkspaceCreationParams` Struct:** `name`, `persistent_id`, `icon_name`, `accent_color_hex`, `layout_type_hint`, `activate_when_created`.
- **`WorkspaceSnapshot` Struct:** `persistent_id`, `name`, `layout_type`, `icon_name`, `accent_color_hex`.
- **`WorkspaceSetSnapshot` Struct:** `workspaces: Vec<WorkspaceSnapshot>`, `active_workspace_persistent_id: Option<String>`.

#### 2.3. Events (`domain::workspaces::events`)

- **`WorkspaceEvent` Enum:** Umfasst `WorkspaceCreated` (mit allen initialen Workspace-Properties), `WorkspaceDeleted`, `ActiveWorkspaceChanged`, `WorkspaceRenamed`, `WorkspaceLayoutChanged`, `WindowAddedToWorkspace`, `WindowRemovedFromWorkspace`, `WorkspaceOrderChanged`, `WorkspacesReloaded`, `WorkspaceIconChanged`, `WorkspaceAccentChanged`, `WorkspaceLayoutApplied` (mit `WorkspaceWindowLayout` von `window_management_policy`).

#### 2.4. Fehlerbehandlung (`domain::workspaces::core::errors`, `assignment::errors`, `config::errors`, `manager::errors`)

- **`WorkspaceCoreError`**: `InvalidName`, `NameCannotBeEmpty`, `NameTooLong`, `InvalidPersistentId`, `WindowIdentifierEmpty`, `InvalidAccentColorFormat`.
- **`WindowAssignmentError`**: `WorkspaceNotFound`, `WindowAlreadyAssigned`, `WindowNotAssignedToWorkspace`, `SourceWorkspaceNotFound`, `TargetWorkspaceNotFound`, `WindowNotOnSourceWorkspace`, `CannotMoveToSameWorkspace`, `RuleViolation`, `Internal`.
- **`WorkspaceConfigError`**: `LoadError` (wrappt `CoreConfigError`), `SaveError` (wrappt `CoreConfigError`), `InvalidData`, `SerializationError`, `DeserializationError`, `PersistentIdNotFoundInLoadedSet`, `DuplicatePersistentIdInLoadedSet`, `VersionMismatch`, `Internal`.
- **`WorkspaceManagerError`**: Wrappt `CoreError`, `AssignmentError`, `ConfigError`. Eigene Varianten: `WorkspaceNotFound`, `CannotDeleteLastWorkspace`, `DeleteRequiresFallbackForWindows`, `FallbackWorkspaceNotFound`, `SetActiveWorkspaceNotFound`, `NoActiveWorkspace`, `DuplicatePersistentId`, `Internal`.

#### 2.5. Interaktionen und Datenfluss

1. **Initialisierung (`WorkspaceManagerService::new`):**
    - Lädt Konfiguration über `WorkspaceConfigProvider` asynchron.
    - Wenn keine Konfig vorhanden oder fehlerhaft (außer "nicht gefunden"), wird ein Standard-Workspace ("Workspace 1") erstellt.
    - Rekonstruiert `Workspace`-Objekte, `ordered_workspace_ids`, `active_workspace_id`.
    - Sendet `WorkspacesReloaded` und `ActiveWorkspaceChanged`.
2. **Fensterzuweisung (`handle_new_system_window`):**
    - Ruft `domain::window_management_policy::WindowManagementPolicyService::get_initial_window_properties()` auf, um Ziel-Workspace und initiale Geometrie/Status zu bestimmen.
    - Ruft intern `domain::workspaces::assignment::assign_window_to_workspace()` mit der Policy-Entscheidung auf.
    - Sendet `WindowAddedToWorkspace`.
3. **Konfigurationsspeicherung (`save_configuration` intern nach Modifikationen):**
    - Erstellt `WorkspaceSetSnapshot` aus dem aktuellen Zustand (`ordered_workspace_ids` für Reihenfolge, `workspaces` für Details, `active_workspace_id` für aktiven).
    - Ruft `config_provider.save_workspace_config()` asynchron auf.
4. **Zustandsverwaltung:** `WorkspaceManagerInternalState` (geschützt durch `tokio::sync::Mutex`) hält `workspaces: HashMap`, `active_workspace_id`, `ordered_workspace_ids`, `config_provider`, `event_publisher`.

---

**(Die Spezifikationen für die Module 3 bis 8 der Domänenschicht würden analog detailliert, basierend auf den bereits existierenden Dokumenten und den hier etablierten Verfeinerungen.)**

---

## Wayland/Smithay-Integration (Systemschicht-Fokus)

Die Systemschicht (`novade-system`) ist primär für die Wayland-Compositor-Implementierung mit Smithay verantwortlich.

### 1. Smithay Compositor-Implementierung (`system::compositor`)

- **Kernstruktur `DesktopState` (in `system::compositor::core::state`):**
    - Implementiert alle relevanten Smithay Handler-Traits:
        - `smithay::wayland::compositor::CompositorHandler`
        - `smithay::wayland::shell::xdg::XdgShellHandler`
        - `smithay::wayland::shm::ShmHandler`, `smithay::wayland::buffer::BufferHandler`
        - `smithay::wayland::seat::SeatHandler`
        - `smithay::wayland::output::OutputHandler`
        - `smithay::wayland::selection::data_device::DataDeviceHandler`
        - `smithay::wayland::shell::wlr_layer::LayerShellHandler`
        - `smithay::wayland::decoration::xdg::XdgDecorationHandler` (oder `kde-server-decoration`)
        - Handler für `wlr-output-management-unstable-v1`, `wlr-output-power-management-unstable-v1`, `wlr-foreign-toplevel-management-unstable-v1`, etc.
    - Hält Instanzen der Smithay State-Objekte: `CompositorState`, `XdgShellState`, `ShmState`, `SeatState<Self>`, `OutputManagerState`, `DataDeviceState`, `WlrLayerShellState`, `XdgDecorationState`, etc. Diese werden im Konstruktor von `DesktopState` initialisiert.
    - Hält Handles zur Domänenschicht: `Arc<dyn WorkspaceManagerService>`, `Arc<dyn WindowManagementPolicyService>`, `Arc<dyn GlobalSettingsService>`.
    - Hält den Smithay `DisplayHandle` und `calloop::LoopHandle`.
    - Verwaltet eine `smithay::desktop::Space<ManagedWindow>` für das Fensterlayout und Rendering.
    - Verwaltet eine `HashMap<WindowIdentifier, ManagedWindowIdentifierCompositor>` für die Zuordnung von Domänen-Fenster-IDs zu Compositor-internen Fenster-Handles.
- **`ManagedWindow` Struct (in `system::compositor::xdg_shell::types` oder `surface_management`):**
    - Implementiert `smithay::desktop::Window` und `smithay::desktop::SpaceElement`.
    - Kapselt ein `smithay::wayland::shell::xdg::ToplevelSurface` oder `PopupSurface` oder `LayerSurface`.
    - Hält eine `domain::workspaces::core::types::WindowIdentifier` für die Verknüpfung zur Domäne.
    - Speichert die aktuelle Geometrie (`RectInt`), den XDG-Zustand (`ToplevelState`).
- **Wayland-Protokolle und Erweiterungen:**
    - **`xdg-shell` (Pflicht):** Für normale Anwendungsfenster. `XdgShellHandler` Methoden:
        - `new_toplevel`: Ruft `domain::WorkspaceManagerService::handle_new_system_window()` auf. Domäne liefert initiale Platzierung/Workspace via `WindowManagementPolicyService`. Compositor erstellt `ManagedWindow`, sendet `configure`.
        - `map_toplevel`: Informiert Domäne (`WorkspaceManagerService`), dass Fenster gemappt wurde. Löst Neuanordnung durch `WindowManagementPolicyService` aus.
        - `ack_configure`: Verarbeitet Client-Bestätigungen.
        - `*_request_*` (z.B. `set_title`, `set_app_id`, `set_maximized`, `move`, `resize`): Aktualisiert `ManagedWindow`-Zustand. Für Move/Resize wird ein `PointerGrab` (siehe `system::input`) gestartet; nach Abschluss wird `WindowManagementPolicyService` für finale Geometrie und Snapping konsultiert, dann `configure` gesendet. Für Zustandsänderungen wird `WindowManagementPolicyService::determine_target_state_for_request` aufgerufen.
    - **`wlr-layer-shell-unstable-v1` (Pflicht):** Für Panels, Wallpaper, Notifications. `LayerShellHandler` Methoden. Erstellt spezielle `LayerSurface`-Objekte. Domäne (z.B. `ThemingEngine` für Wallpaper, `NotificationService` für Popups) liefert Inhalt/Logik.
    - **`xdg-decoration-unstable-v1` (Server-Seite, Pflicht):** Für Fensterdekorationen. `XdgDecorationHandler`. Domäne (`ThemingEngine`, `GlobalSettingsService`) liefert Policy, ob SSD oder CSD bevorzugt wird.
    - **`wlr-output-management-unstable-v1` (Server-Seite, Pflicht):** `system::compositor::output_management` implementiert die Handler. Interagiert mit `smithay::output::Output`-Objekten (verwaltet im `DesktopState.space` oder einer Liste) und dem DRM-Backend. Domäne (`GlobalSettingsService`) liefert Standard-Output-Konfigs.
    - **Weitere Protokolle:** `wl_shm`, `wl_compositor`, `wl_subcompositor`, `wl_seat`, `wl_data_device`, `presentation-time`, `viewporter`, `linux-dmabuf-unstable-v1`, etc. werden durch entsprechende Smithay-States und Handler in `DesktopState` unterstützt.
- **Smithay `DisplayHandle` und `CompositorState`:**
    - `DisplayHandle` (in `DesktopState`): Dient zum Erstellen von Globals und Verwalten von Clients.
    - `CompositorState` (in `DesktopState`): Verwaltet `wl_compositor`, `wl_subcompositor`. `SurfaceAttributes` (via `SurfaceCachedState` an `WlSurface` UserData) halten Puffer, Skalierung, Transformation, Schaden.
- **Systemschicht empfängt Domänenbefehle:**
    - Der Compositor (über `DesktopState`) reagiert auf Events vom `SystemEventBridge` oder direkte Aufrufe von System-internen Services, die von Domänen-Events getriggert wurden.
    - **Workspace-Wechsel:** `WorkspaceEvent::ActiveWorkspaceChanged` von `WorkspaceManagerService`. `DesktopState` aktualisiert, welche Fenster im `Space` sichtbar sind, und wendet das Layout des neuen Workspace an (via `WindowManagementPolicyService` und `system::window_mechanics::layout_applier`). Sendet Fokus an das entsprechende Fenster.
    - **Fensterplatzierung:** `WorkspaceLayoutApplied`-Event von `WorkspaceManagerService`. `system::window_mechanics::layout_applier` iteriert die Geometrien und ruft `ManagedWindow.send_configure()` (oder äquivalente Methoden auf `ToplevelSurface`) für jedes Fenster auf.
- **Wayland-Ereignisse an Domänenschicht:**
    - Direkte Weiterleitung ist selten. Stattdessen:
        - Wayland-Client-Requests (z.B. `xdg_toplevel.set_title`) aktualisieren den Zustand im `ManagedWindow` (Systemschicht).
        - Wenn diese Änderung für die Domäne relevant ist (z.B. ein neues Fenster wird gemappt und muss einem Workspace zugeordnet werden), ruft die Systemschicht eine Methode des entsprechenden Domänenservice auf (z.B. `WorkspaceManagerService::handle_new_system_window`).
        - Domänen-Events werden dann von der Domänenschicht selbst ausgelöst (z.B. `WorkspaceEvent::WindowAddedToWorkspace`).

#### 1.6. Code-Skizzen und Algorithmen

- **`DesktopState::new(...)` (Initialisierung):**
    
    Rust
    
    ```
    // system/src/compositor/core/state.rs
    pub fn new(
        display_handle: DisplayHandle,
        loop_handle: LoopHandle<'static, Self>,
        // ... Domain Service Handles ...
        // ... SystemEventBridge ...
    ) -> Self {
        let clock = Clock::new(None); // No logger for brevity
    
        let compositor_state = CompositorState::new::<Self>(&display_handle, None);
        let shm_state = ShmState::new::<Self>(&display_handle, vec![], None); // No extra formats
        let xdg_shell_state = XdgShellState::new::<Self>(&display_handle, None);
        let mut seat_state = SeatState::new();
        let seat_name = "seat0".to_string();
        let seat = seat_state.new_wl_seat(&display_handle, seat_name.clone(), None);
        // ... Initialize other Smithay states (DataDeviceState, OutputManagerState, etc.) ...
        let space = Space::new(None);
    
        // Initialize all Wayland globals
        let shm_global = shm_state.global().clone();
        let xdg_shell_global = xdg_shell_state.global().clone();
        // ... create and store other global IDs ...
    
        let wayland_globals = Some(NovaDEWaylandState {
            shm_global, xdg_shell_global, /* ... */
        });
    
        Self {
            display_handle, loop_handle, clock,
            compositor_state, shm_state, xdg_shell_state, seat_state, seat, seat_name,
            space, windows_map: HashMap::new(), /* ... other fields ... */ wayland_globals,
            // ... Domain Service Handles ...
        }
    }
    ```
    
- **`XdgShellHandler::new_toplevel` (vereinfacht):**
    
    Rust
    
    ```
    // In DesktopState impl XdgShellHandler
    fn new_toplevel(&mut self, surface: ToplevelSurface) {
        let domain_window_id = WindowIdentifier::new_v4(); // Create a domain-level ID
        let client_app_id = surface.app_id(); // Get from client
        let client_title = surface.title();
    
        // Prepare initial info for domain policy
        let initial_props = WindowLayoutInfo {
            id: domain_window_id.clone(),
            requested_min_size: surface.min_size(), // And other client hints
            // ...
        };
    
        // This call needs to be carefully managed if it's async within a sync handler context.
        // Option 1: block_on (if handler context allows it, usually not ideal for event loop)
        // Option 2: Send a command to a separate tokio task that then calls the domain service
        // Option 3: Make the domain service call synchronous if it doesn't do I/O
        let initial_placement_result = block_on(
            self.workspace_manager_service.handle_new_system_window(domain_window_id.clone(), Some(initial_props))
            .then(|_| self.window_management_policy_service.get_initial_window_properties(
                &domain_window_id, /* ... other params from WorkspaceManager or client ... */
            ))
        );
    
        match initial_placement_result {
            Ok(initial_properties) => {
                let managed_window = ManagedWindow::new_toplevel(surface.clone(), domain_window_id, initial_properties.geometry);
                // Store XdgToplevelSurfaceData for this surface
                surface.wl_surface().data_map().insert_if_missing_threadsafe(XdgToplevelSurfaceData::new);
                // Configure the surface with initial state and geometry
                surface.send_configure(initial_properties.initial_xdg_state.into_abi(), Some(initial_properties.geometry.size));
    
                let window_arc = Arc::new(managed_window);
                self.windows_map.insert(domain_window_id, window_arc.id()); // Or map to WlSurface
                self.space.map_window(window_arc, initial_properties.geometry.loc, initial_properties.focus_on_creation);
    
                if initial_properties.focus_on_creation {
                    // Call internal focus logic
                    // self.set_focus_internal(Some(surface.wl_surface()), Serial::now());
                }
            }
            Err(e) => {
                tracing::error!("Failed to get initial placement for new toplevel: {:?}", e);
                // Potentially close the surface or mark it as unmanaged
                surface.send_close();
            }
        }
    }
    ```
    
- **Workspace-Wechsel (vereinfachter Datenfluss):**
    
    1. Domäne (`WorkspaceManagerService`) löst `WorkspaceEvent::ActiveWorkspaceChanged { new_id, .. }` aus.
    2. `SystemEventBridge` leitet dies weiter (oder ein direkter Listener im `DesktopState`).
    3. `DesktopState` empfängt das Event:
        - `current_layout = self.window_management_policy_service.calculate_workspace_layout(new_id, windows_on_new_ws, ...).await;`
        - `self.system_window_mechanics_apply_layout(current_layout);` (interne Methode)
    4. `system_window_mechanics_apply_layout`:
        - Iteriert `current_layout.window_geometries`.
        - Für jedes Fenster:
            - Sucht `ManagedWindow` im `self.space`.
            - `managed_window.xdg_surface.toplevel().send_configure(...)` mit neuer Geometrie/Status.
            - Aktualisiert Fenster im `self.space` (Sichtbarkeit, Position).
        - Setzt Fokus auf das zuletzt fokussierte Fenster des neuen Workspace.

---

Die Implementierung der Domänenschicht und deren Integration mit der Systemschicht ist ein komplexer Prozess, der sorgfältige Beachtung von Asynchronität, Zustandsmanagement und Fehlerbehandlung erfordert. Die hier skizzierten Spezifikationen bieten einen detaillierten Rahmen für die Entwicklung.

 Quellen
## Technische Spezifikationen und Implementierungsanleitungen: Domänenschicht NovaDE

Dieses Dokument detailliert die technischen Spezifikationen und Implementierungsanleitungen für die Domänenschicht des NovaDE-Projekts. Der Fokus liegt auf den internen APIs, Datenstrukturen, Ereignissen, der Fehlerbehandlung und der spezifischen Integration mit der Systemschicht, insbesondere dem Wayland/Smithay-basierten Compositor.

### 1. Domänenschicht-Spezifikationen

Die Domänenschicht (`novade-domain` Crate) kapselt die Kernlogik und den Zustand der Desktop-Umgebung, unabhängig von UI- oder Systemdetails. Sie ist in mehrere Module unterteilt, die jeweils spezifische Verantwortlichkeiten tragen.

#### 1.1. Modul: `domain::shared_types`

Dieses Modul definiert grundlegende, domänenspezifische Typen, die von mehreren anderen Domänenmodulen genutzt werden.

- **`ApplicationId`**: Ein typsicherer Wrapper für Anwendungsidentifikatoren.
    - **Definition:**
        
        Rust
        
        ```
        // src/domain/shared_types.rs
        use serde::{Serialize, Deserialize};
        use std::fmt;
        
        /// Eindeutiger Bezeichner für eine Anwendung.
        /// Repräsentiert typischerweise einen Reverse-DNS-Namen (z.B. "org.novade.FileExplorer")
        /// oder den Namen der .desktop-Datei ohne Erweiterung.
        #[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize, Default, PartialOrd, Ord)]
        pub struct ApplicationId(String);
        
        impl ApplicationId {
            /// Erstellt eine neue `ApplicationId`.
            ///
            /// # Panics
            /// Paniert, wenn die `id` leer ist (im Debug-Modus).
            pub fn new(id: impl Into<String>) -> Self {
                let id_str = id.into();
                debug_assert!(!id_str.is_empty(), "ApplicationId darf nicht leer sein.");
                // Weitere Validierungen (z.B. erlaubte Zeichen) könnten hier oder bei der Erzeugung erfolgen.
                Self(id_str)
            }
        
            /// Gibt die `ApplicationId` als String-Slice zurück.
            pub fn as_str(&self) -> &str {
                &self.0
            }
        }
        
        impl From<String> for ApplicationId {
            fn from(s: String) -> Self {
                debug_assert!(!s.is_empty(), "ApplicationId (from String) darf nicht leer sein.");
                Self(s)
            }
        }
        
        impl From<&str> for ApplicationId {
            fn from(s: &str) -> Self {
                debug_assert!(!s.is_empty(), "ApplicationId (from &str) darf nicht leer sein.");
                Self(s.to_string())
            }
        }
        
        impl fmt::Display for ApplicationId {
            fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
                write!(f, "{}", self.0)
            }
        }
        ```
        
    - **Verantwortlichkeiten**: Repräsentation einer Anwendungs-ID.
    - **Invarianten**: Interner String darf nicht leer sein.
- **`UserSessionState`**: Repräsentiert den aktuellen Zustand der Benutzersitzung.
    - **Definition:**
        
        Rust
        
        ```
        // src/domain/shared_types.rs
        use serde::{Serialize, Deserialize};
        
        /// Repräsentiert den aktuellen Zustand der Benutzersitzung aus Sicht der Domäne.
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, Default)]
        pub enum UserSessionState {
            #[default]
            Active, // Normale Benutzersitzung, Benutzer ist aktiv
            Locked, // Sitzung gesperrt (z.B. durch Bildschirmsperre)
            Idle,   // Benutzer ist für eine definierte Zeit inaktiv
        }
        ```
        
    - **Verantwortlichkeiten**: Abstraktion des Sitzungszustands.
- **`ResourceIdentifier`**: Allgemeiner Bezeichner für eine Ressource.
    - **Definition:**
        
        Rust
        
        ```
        // src/domain/shared_types.rs
        use serde::{Serialize, Deserialize};
        use uuid::Uuid;
        
        /// Allgemeiner Bezeichner für eine Ressource.
        #[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
        pub struct ResourceIdentifier {
            /// Der Typ der Ressource (z.B. "file", "contact", "calendar-event", "web-url").
            pub r#type: String,
            /// Die eindeutige ID der Ressource innerhalb ihres Typs.
            pub id: String,
            /// Optionale menschenlesbare Beschreibung oder Name der Ressource.
            #[serde(default, skip_serializing_if = "Option::is_none")]
            pub label: Option<String>,
        }
        
        impl ResourceIdentifier {
            pub fn new(resource_type: impl Into<String>, resource_id: impl Into<String>, label: Option<String>) -> Self {
                let type_str = resource_type.into();
                let id_str = resource_id.into();
                debug_assert!(!type_str.is_empty(), "ResourceIdentifier type darf nicht leer sein.");
                debug_assert!(!id_str.is_empty(), "ResourceIdentifier id darf nicht leer sein.");
                Self {
                    r#type: type_str,
                    id: id_str,
                    label,
                }
            }
        
            pub fn file(path: impl Into<String>, label: Option<String>) -> Self {
                Self::new("file", path, label)
            }
        }
        ```
        
    - **Verantwortlichkeiten**: Repräsentation einer Ressource für KI-Interaktionen oder andere Dienste.
    - **Invarianten**: `r#type` und `id` dürfen nicht leer sein.

#### 1.2. Modul: `domain::common_events`

Dieses Modul definiert Event-Typen, die von mehreren Domänenmodulen ausgelöst oder konsumiert werden können.

- **`UserActivityType`**: Kategorisiert Benutzeraktivitäten.
    - **Definition:**
        
        Rust
        
        ```
        // src/domain/common_events.rs
        use serde::{Serialize, Deserialize};
        
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
        pub enum UserActivityType {
            MouseMoved,
            MouseClicked,
            MouseWheelScrolled,
            KeyPressed,
            TouchInteraction,
            WorkspaceSwitched,
            ApplicationFocused,
            WindowOpened,
            WindowClosed,
        }
        ```
        
- **`UserActivityDetectedEvent`**: Wird ausgelöst, wenn eine Benutzeraktivität erkannt wird.
    - **Definition:**
        
        Rust
        
        ```
        // src/domain/common_events.rs
        use chrono::{DateTime, Utc};
        use serde::{Serialize, Deserialize};
        use crate::shared_types::{UserSessionState, ApplicationId}; // Pfad anpassen
        use super::UserActivityType;
        use uuid::Uuid;
        use crate::workspaces::core::types::WorkspaceId; // Pfad anpassen
        
        #[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
        pub struct UserActivityDetectedEvent {
            pub event_id: Uuid,
            pub timestamp: DateTime<Utc>,
            pub activity_type: UserActivityType,
            pub current_session_state: UserSessionState,
            #[serde(default, skip_serializing_if = "Option::is_none")]
            pub active_application_id: Option<ApplicationId>,
            #[serde(default, skip_serializing_if = "Option::is_none")]
            pub active_workspace_id: Option<WorkspaceId>,
        }
        
        impl UserActivityDetectedEvent {
            pub fn new(
                activity_type: UserActivityType,
                current_session_state: UserSessionState,
                active_application_id: Option<ApplicationId>,
                active_workspace_id: Option<WorkspaceId>
            ) -> Self {
                Self {
                    event_id: Uuid::new_v4(),
                    timestamp: Utc::now(),
                    activity_type,
                    current_session_state,
                    active_application_id,
                    active_workspace_id,
                }
            }
        }
        ```
        
    - **Payload**: `event_id: Uuid`, `timestamp: DateTime<Utc>`, `activity_type: UserActivityType`, `current_session_state: UserSessionState`, `active_application_id: Option<ApplicationId>`, `active_workspace_id: Option<WorkspaceId>`.
    - **Publisher**: Systemschicht-Adapter für `system::input`, `domain::workspaces::manager`.
    - **Subscriber**: `domain::user_centric_services::ai_interaction`, `domain::power_management_policy`.
- **`SystemShutdownInitiatedEvent`**: Signalisiert bevorstehenden System-Shutdown/Neustart.
    - **Definition:**
        
        Rust
        
        ```
        // src/domain/common_events.rs
        use chrono::{DateTime, Utc};
        use serde::{Serialize, Deserialize};
        use uuid::Uuid;
        
        #[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize, Default)]
        pub enum ShutdownReason {
            #[default]
            UserRequest,
            PowerButtonPress,
            LowBattery,
            SystemUpdate,
            ApplicationRequest,
            OsError,
            Unknown,
        }
        
        #[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
        pub struct SystemShutdownInitiatedEvent {
            pub event_id: Uuid,
            pub timestamp: DateTime<Utc>,
            pub reason: ShutdownReason,
            pub is_reboot: bool,
            #[serde(default, skip_serializing_if = "Option::is_none")]
            pub delay_seconds: Option<u32>,
            #[serde(default, skip_serializing_if = "Option::is_none")]
            pub message: Option<String>,
        }
        
        impl SystemShutdownInitiatedEvent {
            pub fn new(reason: ShutdownReason, is_reboot: bool, delay_seconds: Option<u32>, message: Option<String>) -> Self {
                Self {
                    event_id: Uuid::new_v4(),
                    timestamp: Utc::now(),
                    reason,
                    is_reboot,
                    delay_seconds,
                    message,
                }
            }
        }
        ```
        
    - **Payload**: `event_id: Uuid`, `timestamp: DateTime<Utc>`, `reason: ShutdownReason`, `is_reboot: bool`, `delay_seconds: Option<u32>`, `message: Option<String>`.
    - **Publisher**: Systemschicht-Adapter für `logind`.
    - **Subscriber**: Domänendienste, die Zustände speichern müssen.

#### 1.3. Modul: `domain::theming`

Verantwortlich für die Logik des Erscheinungsbilds (Theming).

- **Datenstrukturen** (in `src/theming/types.rs`):
    - `TokenIdentifier(String)`: Hierarchischer Bezeichner für Design-Tokens.
        - Invarianten: Nicht leer, nur ASCII-Alphanumerisch, Punkte, Bindestriche.
    - `TokenValue`: Enum für Token-Werte (Color(String), Dimension(String), ..., Reference(TokenIdentifier)).
        - Invarianten: `Opacity` zwischen 0.0 und 1.0. Strings für Farben/Dimensionen sollten CSS-kompatibel sein.
    - `RawToken { id: TokenIdentifier, value: TokenValue, description: Option<String>, group: Option<String> }`
    - `TokenSet = BTreeMap<TokenIdentifier, RawToken>`: Gewährleistet deterministische Reihenfolge.
    - `ThemeIdentifier(String)`: Bezeichner für ein Theme.
    - `ColorSchemeType`: Enum (`Light`, `Dark`).
    - `AccentColor { name: Option<String>, value: novade_core::types::Color }`: Verwendet `Color` aus der Kernschicht.
    - `ThemeVariantDefinition { applies_to_scheme: ColorSchemeType, tokens: TokenSet }`
    - `ThemeDefinition { id: ThemeIdentifier, name: String, ..., base_tokens: TokenSet, variants: Vec<ThemeVariantDefinition>, supported_accent_colors: Option<Vec<AccentColor>>, accentable_tokens: Option<HashMap<TokenIdentifier, AccentModificationType>> }`
        - `AccentModificationType`: Enum (`DirectReplace`, `Lighten(f32)`, `Darken(f32)`).
    - `AppliedThemeState { theme_id: ThemeIdentifier, color_scheme: ColorSchemeType, active_accent_color: Option<AccentColor>, resolved_tokens: BTreeMap<TokenIdentifier, String> }`
    - `ThemingConfiguration { selected_theme_id: ThemeIdentifier, preferred_color_scheme: ColorSchemeType, selected_accent_color: Option<novade_core::types::Color>, custom_user_token_overrides: Option<TokenSet> }`
- **Service API (`ThemingEngine` Trait)** (in `src/theming/service.rs` oder `mod.rs`):
    
    Rust
    
    ```
    // src/theming/service.rs
    use async_trait::async_trait;
    use std::path::PathBuf;
    use std::sync::Arc;
    use tokio::sync::broadcast;
    use super::types::{ThemingConfiguration, AppliedThemeState, ThemeDefinition, ThemeChangedEvent};
    use super::errors::ThemingError;
    use crate::core::config::ConfigServiceAsync; // Pfad anpassen
    
    #[async_trait]
    pub trait ThemingEngine: Send + Sync {
        async fn new(
            initial_config: ThemingConfiguration,
            theme_load_paths: Vec<PathBuf>,
            token_load_paths: Vec<PathBuf>,
            config_service: Arc<dyn ConfigServiceAsync>, // Für Dateizugriff
            broadcast_capacity: usize,
        ) -> Result<Self, ThemingError> where Self: Sized;
    
        async fn get_current_theme_state(&self) -> AppliedThemeState;
        async fn get_available_themes(&self) -> Vec<ThemeDefinition>;
        async fn get_current_configuration(&self) -> ThemingConfiguration;
        async fn update_configuration(&self, new_config: ThemingConfiguration) -> Result<(), ThemingError>;
        async fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>;
        fn subscribe_to_theme_changes(&self) -> broadcast::Receiver<ThemeChangedEvent>;
    }
    ```
    
- **Events** (in `src/theming/types.rs`):
    - `ThemeChangedEvent { new_state: AppliedThemeState }`
- **Fehlerbehandlung** (in `src/theming/errors.rs`):
    - `ThemingError`: Enum mit Varianten wie `TokenFileParseError`, `ThemeFileLoadError`, `CyclicTokenReference`, `MissingTokenReference`, `ThemeNotFound`, `InvalidTokenValue`, `AccentColorApplicationError`, `MaxReferenceDepthExceeded`, `FallbackThemeLoadError`, `FilesystemError(#[from] novade_core::errors::CoreError)`.
- **Interne Logik** (in `src/theming/logic.rs`): Token Resolution Pipeline (asynchrones Laden, synchrone Auflösung), Caching, Fallback-Theme-Logik.

#### 1.4. Modul: `domain::global_settings_and_state_management`

Verantwortlich für globale Desktop-Einstellungen.

- **Datenstrukturen** (in `src/global_settings/types.rs`):
    - `GlobalDesktopSettings` und Unterstrukturen (`AppearanceSettings`, `InputBehaviorSettings`, etc.) mit `serde` und `Default`.
    - Jede Unterstruktur hat eine `validate(&self) -> Result<(), String>` Methode.
- **Einstellungspfade** (in `src/global_settings/paths.rs`):
    - `SettingPath`: Hierarchischer Enum, der alle Einstellungsfelder abbildet. Implementiert `Display` und `TryFrom<&str>`.
- **Persistenzschnittstelle** (in `src/global_settings/persistence_iface.rs`):
    - `SettingsPersistenceProvider` Trait: `async fn load_global_settings() -> Result<GlobalDesktopSettings, GlobalSettingsError>`, `async fn save_global_settings(settings: &GlobalDesktopSettings) -> Result<(), GlobalSettingsError>`.
    - Implementierung `FilesystemSettingsProvider` (in `src/global_settings/persistence.rs`) nutzt `Arc<dyn novade_core::config::ConfigServiceAsync>`.
- **Service API (`GlobalSettingsService` Trait)** (in `src/global_settings/service.rs`):
    
    Rust
    
    ```
    // src/global_settings/service.rs
    use async_trait::async_trait;
    use tokio::sync::broadcast;
    use super::types::GlobalDesktopSettings;
    use super::paths::SettingPath;
    use super::errors::GlobalSettingsError;
    use super::events::SettingChangedEvent; // Events hier definieren
    use serde_json::Value as JsonValue;
    
    #[async_trait]
    pub trait GlobalSettingsService: Send + Sync {
        async fn load_settings(&self) -> Result<(), GlobalSettingsError>;
        async fn save_settings(&self) -> Result<(), GlobalSettingsError>;
        async fn get_current_settings_snapshot(&self) -> GlobalDesktopSettings; // Gibt Klon
        async fn update_setting(&self, path: SettingPath, value: JsonValue) -> Result<(), GlobalSettingsError>;
        async fn get_setting_as_json(&self, path: &SettingPath) -> Result<JsonValue, GlobalSettingsError>;
        async fn reset_to_defaults(&self) -> Result<(), GlobalSettingsError>;
        fn subscribe_to_changes(&self) -> broadcast::Receiver<SettingChangedEvent>;
    }
    ```
    
- **Events** (in `src/global_settings/events.rs`):
    - `SettingChangedEvent { path: SettingPath, new_value: JsonValue }`
    - `SettingsLoadedEvent { settings: GlobalDesktopSettings }`
    - `SettingsSavedEvent`
- **Fehlerbehandlung** (in `src/global_settings/errors.rs`):
    - `GlobalSettingsError`: Enum mit Varianten wie `PathNotFound { path: SettingPath }`, `InvalidValueType`, `ValidationError { path: SettingPath, reason: String }`, `PersistenceError { operation: String, #[source] source: novade_core::errors::CoreError }`.

#### 1.5. Modul: `domain::workspaces`

Verwaltet Arbeitsbereiche ("Spaces").

- **Datenstrukturen (`core` Submodul - `src/workspaces/core/types.rs`):**
    - `WorkspaceId = uuid::Uuid`
    - `WindowIdentifier(String)`: Invarianten: nicht leer.
    - `WorkspaceLayoutType`: Enum (`Floating`, `TilingHorizontal`, `TilingVertical`, `Maximized`).
    - `Workspace`: Struct (`id`, `name`, `persistent_id`, `layout_type`, `window_ids: HashSet<WindowIdentifier>`, `created_at`, `icon_name: Option<String>`, `accent_color_hex: Option<String>`). Validierung für `name` und `persistent_id`.
        - Methoden: `new`, `rename`, `set_layout_type`, `set_icon_name`, `set_accent_color_hex`, etc.
- **Fensterzuweisungslogik (`assignment` Submodul - `src/workspaces/assignment/mod.rs`):**
    - Funktionen: `assign_window_to_workspace`, `remove_window_from_workspace`, `move_window_to_workspace`, `find_workspace_for_window`. Operieren auf `&mut HashMap<WorkspaceId, Workspace>`.
- **Konfigurationspersistenz (`config` Submodul - `src/workspaces/config/mod.rs`):**
    - `WorkspaceSnapshot { persistent_id: String, name: String, layout_type: WorkspaceLayoutType, icon_name: Option<String>, accent_color_hex: Option<String> }`
    - `WorkspaceSetSnapshot { workspaces: Vec<WorkspaceSnapshot>, active_workspace_persistent_id: Option<String> }`
    - `WorkspaceConfigProvider` Trait: `async fn load_workspace_config()`, `async fn save_workspace_config()`.
- **Service API (`WorkspaceManagerService` Trait - `src/workspaces/manager/service.rs`):**
    
    Rust
    
    ```
    // src/workspaces/manager/service.rs
    use async_trait::async_trait;
    use tokio::sync::broadcast;
    use crate::workspaces::core::types::{WorkspaceId, WindowIdentifier, WorkspaceLayoutType, Workspace};
    use super::errors::WorkspaceManagerError;
    use super::events::WorkspaceEvent; // Events hier definieren
    
    #[async_trait]
    pub trait WorkspaceManagerService: Send + Sync {
        async fn create_workspace(&self, name: Option<String>, persistent_id: Option<String>, icon_name: Option<String>, accent_color_hex: Option<String>) -> Result<WorkspaceId, WorkspaceManagerError>;
        async fn delete_workspace(&self, id: WorkspaceId, fallback_id_for_windows: Option<WorkspaceId>) -> Result<(), WorkspaceManagerError>;
        async fn get_workspace_by_id(&self, id: WorkspaceId) -> Option<Workspace>; // Klon
        async fn get_all_workspaces_ordered(&self) -> Vec<Workspace>; // Klone
        async fn get_active_workspace_id(&self) -> Option<WorkspaceId>;
        async fn set_active_workspace(&self, id: WorkspaceId) -> Result<(), WorkspaceManagerError>;
        async fn assign_window_to_active_workspace(&self, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;
        async fn assign_window_to_specific_workspace(&self, workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;
        async fn remove_window_from_its_workspace(&self, window_id: &WindowIdentifier) -> Result<Option<WorkspaceId>, WorkspaceManagerError>;
        async fn move_window_to_specific_workspace(&self, target_workspace_id: WorkspaceId, window_id: &WindowIdentifier) -> Result<(), WorkspaceManagerError>;
        async fn rename_workspace(&self, id: WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>;
        async fn set_workspace_layout(&self, id: WorkspaceId, layout_type: WorkspaceLayoutType) -> Result<(), WorkspaceManagerError>;
        async fn set_workspace_icon(&self, id: WorkspaceId, icon_name: Option<String>) -> Result<(), WorkspaceManagerError>;
        async fn set_workspace_accent_color(&self, id: WorkspaceId, color_hex: Option<String>) -> Result<(), WorkspaceManagerError>;
        async fn reorder_workspace(&self, workspace_id: WorkspaceId, new_index: usize) -> Result<(), WorkspaceManagerError>;
        async fn save_configuration(&self) -> Result<(), WorkspaceManagerError>;
        fn subscribe_to_workspace_events(&self) -> broadcast::Receiver<WorkspaceEvent>;
    }
    ```
    
- **Events (`manager` Submodul - `src/workspaces/manager/events.rs`):**
    - `WorkspaceEvent` Enum: `WorkspaceCreated { descriptor: Workspace }`, `WorkspaceDeleted`, `ActiveWorkspaceChanged`, `WorkspaceRenamed`, `WorkspaceLayoutChanged`, `WindowAddedToWorkspace`, `WindowRemovedFromWorkspace`, `WorkspaceOrderChanged`, `WorkspacesReloaded`, `WorkspaceIconChanged`, `WorkspaceAccentChanged`.
- **Fehlerbehandlung (`core`, `assignment`, `config`, `manager` Submodule):**
    - `WorkspaceCoreError`, `WindowAssignmentError`, `WorkspaceConfigError`, `WorkspaceManagerError`.

#### 1.6. Modul: `domain::window_management_policy`

Definiert High-Level-Regeln für Fensterplatzierung, Tiling, Snapping, etc.

- **Datenstrukturen** (in `src/window_management_policy/types.rs`):
    - `TilingMode`: Enum (`Manual`, `Columns`, `Rows`, `Spiral`, `MaximizedFocused`).
    - `GapSettings { screen_outer_horizontal: u16, screen_outer_vertical: u16, window_inner: u16 }`
    - `WindowSnappingPolicy { snap_to_screen_edges: bool, snap_to_other_windows: bool, snap_to_workspace_gaps: bool, snap_distance_px: u16 }`
    - `NewWindowPlacementStrategy`: Enum (`Smart`, `Center`, `Cascade`, `UnderMouse`).
    - `FocusPolicy { focus_follows_mouse: bool, click_to_focus: bool, focus_new_windows_on_creation: bool, ... }`
    - `WindowPolicyOverrides { preferred_tiling_mode: Option<TilingMode>, is_always_floating: Option<bool>, ... }` (pro Fenster)
    - `WorkspaceWindowLayout { window_geometries: HashMap<WindowIdentifier, novade_core::types::RectInt>, tiling_mode_applied: TilingMode, occupied_area: Option<RectInt> }`
    - `WindowLayoutInfo { id: WindowIdentifier, requested_min_size: Option<novade_core::types::Size<u32>>, ... }`
- **Service API (`WindowManagementPolicyService` Trait)** (in `src/window_management_policy/service.rs`):
    
    Rust
    
    ```
    // src/window_management_policy/service.rs
    use async_trait::async_trait;
    // ... imports ...
    use crate::core::types::{RectInt, Size, Point};
    use crate::workspaces::core::types::{WorkspaceId, WindowIdentifier};
    use crate::global_settings_and_state_management::types::GlobalDesktopSettings;
    use super::types::{TilingMode, GapSettings, WindowSnappingPolicy, NewWindowPlacementStrategy, WorkspaceWindowLayout, WindowPolicyOverrides, FocusPolicy, WindowLayoutInfo};
    use super::errors::WindowPolicyError;
    use std::collections::HashMap;
    
    #[async_trait]
    pub trait WindowManagementPolicyService: Send + Sync {
        async fn calculate_workspace_layout(
            &self,
            workspace_id: WorkspaceId,
            windows_to_layout: &[WindowLayoutInfo],
            available_area: RectInt,
            workspace_current_tiling_mode: TilingMode,
            focused_window_id: Option<&WindowIdentifier>,
            window_specific_overrides: &HashMap<WindowIdentifier, WindowPolicyOverrides>,
            // Globale Settings werden intern über den GlobalSettingsService bezogen
        ) -> Result<WorkspaceWindowLayout, WindowPolicyError>;
    
        async fn get_initial_window_geometry(
            &self,
            window_info: &WindowLayoutInfo,
            is_transient_for: Option<&WindowIdentifier>,
            parent_geometry: Option<RectInt>,
            workspace_id: WorkspaceId,
            active_layout_on_workspace: &WorkspaceWindowLayout,
            available_area: RectInt,
            window_specific_overrides: &Option<WindowPolicyOverrides>,
        ) -> Result<RectInt, WindowPolicyError>;
    
        async fn calculate_snap_target(
            &self,
            moving_window_id: &WindowIdentifier,
            current_geometry: RectInt,
            other_windows_on_workspace: &[(&WindowIdentifier, &RectInt)],
            workspace_area: RectInt,
            // SnappingPolicy & GapSettings werden intern bezogen
        ) -> Option<RectInt>;
    
        async fn get_effective_focus_policy(&self) -> Result<FocusPolicy, WindowPolicyError>;
        async fn get_effective_new_window_placement_strategy(&self) -> Result<NewWindowPlacementStrategy, WindowPolicyError>;
        async fn get_effective_tiling_mode_for_workspace(&self, workspace_id: WorkspaceId) -> Result<TilingMode, WindowPolicyError>;
        async fn get_effective_gap_settings(&self) -> Result<GapSettings, WindowPolicyError>; // Global oder pro Workspace
        async fn get_effective_snapping_policy(&self) -> Result<WindowSnappingPolicy, WindowPolicyError>;
    }
    ```
    
- **Fehlerbehandlung** (in `src/window_management_policy/errors.rs`):
    - `WindowPolicyError`: Enum (`LayoutCalculationError`, `InvalidPolicyConfiguration`, `WindowNotFoundForPolicy`).

#### 1.7. Modul: `domain::user_centric_services`

Umfasst KI-Interaktion und Benachrichtigungsmanagement.

- **`ai_interaction` Submodul:**
    - Datenstrukturen (`AIConsentStatus`, `AIDataCategory`, `AttachmentData`, `AIInteractionContext`, `InteractionHistoryEntry`, `AIConsent`, `AIConsentScope`, `AIModelProfile`, `AIModelCapability`).
    - Persistenzschnittstellen: `AIConsentProvider`, `AIModelProfileProvider`.
    - Service API: `AIInteractionLogicService` Trait.
    - Events: `AIInteractionEventEnum` (`AIInteractionInitiatedEvent`, `AIConsentUpdatedEvent`, `AIContextUpdatedEvent`, `AIModelProfilesReloadedEvent`).
    - Fehler: `AIInteractionError`.
- **`notifications_core` Submodul:**
    - Datenstrukturen (`NotificationUrgency`, `NotificationActionType`, `NotificationAction`, `Notification`, `NotificationInput`, `NotificationFilterCriteria`, `NotificationSortOrder`, `NotificationStats`, `DismissReason`).
    - Service API: `NotificationService` Trait.
    - Events: `NotificationEventEnum` (`NotificationPostedEvent`, `NotificationDismissedEvent`, `NotificationReadEvent`, `NotificationActionInvokedEvent`, `DoNotDisturbModeChangedEvent`, `NotificationHistoryClearedEvent`, `NotificationPopupExpiredEvent`).
    - Fehler: `NotificationError`.

#### 1.8. Modul: `domain::notifications_rules`

Regel-Engine für Benachrichtigungen.

- **Datenstrukturen** (in `src/notifications_rules/types.rs`):
    - `RuleConditionValue`, `RuleConditionOperator`, `RuleConditionField`, `SimpleRuleCondition`, `RuleCondition` (rekursiv), `RuleAction`, `NotificationRule`, `NotificationRuleSet`.
- **Persistenzschnittstelle** (in `src/notifications_rules/persistence_iface.rs`):
    - `NotificationRulesProvider` Trait: `async fn load_rules()`, `async fn save_rules()`.
- **Engine API (`NotificationRulesEngine` Trait)** (in `src/notifications_rules/engine.rs`):
    
    Rust
    
    ```
    // src/notifications_rules/engine.rs
    use async_trait::async_trait;
    use crate::user_centric_services::notifications_core::types::Notification; // Pfad anpassen
    use super::types::NotificationRuleSet;
    use super::errors::NotificationRulesError;
    
    #[derive(Debug, Clone, PartialEq)]
    pub enum RuleProcessingResult {
        Allow(Notification),
        Suppress { rule_id: uuid::Uuid },
    }
    
    #[async_trait]
    pub trait NotificationRulesEngine: Send + Sync {
        async fn reload_rules(&self) -> Result<(), NotificationRulesError>;
        async fn process_notification(
            &self,
            notification: Notification,
        ) -> Result<RuleProcessingResult, NotificationRulesError>;
        async fn get_rules(&self) -> Result<NotificationRuleSet, NotificationRulesError>;
        async fn update_rules(&self, new_rules: NotificationRuleSet) -> Result<(), NotificationRulesError>;
    }
    ```
    
- **Fehlerbehandlung** (in `src/notifications_rules/errors.rs`):
    - `NotificationRulesError`: Enum (`InvalidRuleDefinition`, `ConditionEvaluationError`, `SettingsAccessError`, `RulePersistenceError`).

### 2. Wayland/Smithay-Integration

Die Domänenschicht selbst interagiert **nicht direkt** mit Wayland oder Smithay. Die Systemschicht (`system::compositor`, `system::input`, `system::window_mechanics`) ist für diese Integration verantwortlich.

- **Kommunikation Domäne &lt;-> Systemschicht:**
    - **Domäne zu System**: Die Domänenschicht gibt Richtlinien und Zustände vor. Z.B. `domain::window_management_policy::WindowManagementPolicyService::calculate_workspace_layout()` gibt ein `WorkspaceWindowLayout` zurück. `system::window_mechanics` nimmt dieses Ergebnis und wendet es auf die `ManagedWindow`s im `DesktopState::space` an, indem es `configure`-Events via Smithay (`xdg_toplevel.send_configure()`) an die Wayland-Clients sendet.
    - **System zu Domäne**: Die Systemschicht meldet rohe Systemereignisse oder Client-Anfragen an die Domänenschicht, die dann die Geschäftslogik anwendet.
        - Beispiel: `system::input` verarbeitet einen Tastendruck. Wenn dieser eine globale Aktion auslöst (z.B. Workspace wechseln), ruft `system::input` (oder ein Handler in `DesktopState`) eine Methode auf `domain::workspaces::WorkspaceManagerService` auf (z.B. `switch_to_next_workspace()`). Der `WorkspaceManagerService` ändert seinen Zustand und sendet ein `WorkspaceEvent::ActiveWorkspaceChanged`.
        - Beispiel: Ein Wayland-Client (`xdg_toplevel`) sendet eine `set_title`-Anfrage. Der `XdgShellHandler` in `DesktopState` empfängt dies. Er könnte direkt den Titel im `ManagedWindow` setzen oder, falls Validierung oder zusätzliche Logik nötig ist, eine Methode eines Domänendienstes (z.B. `domain::window_properties_service::set_window_title(window_id, new_title)`) aufrufen.
- **Notwendige Wayland-Protokolle (implementiert in Systemschicht):**
    - `xdg-shell`: Für Toplevel-Fenster und Popups. Die Systemschicht (`system::compositor::xdg_shell`) implementiert den `XdgShellHandler`. Zustandsänderungen an `ManagedWindow`s (die `xdg_toplevel`-Instanzen wrappen) werden von `system::window_mechanics` oder direkt im `XdgShellHandler` basierend auf Domänen-Policy durchgeführt.
    - `wlr-layer-shell-unstable-v1`: Für Panels, Docks etc. (`system::compositor::layer_shell`). Die Domänenschicht gibt keine direkten Anweisungen für Layer-Shell-Objekte, aber die globalen Einstellungen (`GlobalSettingsService`) könnten beeinflussen, welche Panels standardmäßig aktiv sind oder wie sie sich verhalten.
    - Weitere Protokolle (`wlr-output-management-unstable-v1`, etc.) werden von der Systemschicht gehandhabt. Die Domänenschicht definiert ggf. Policies oder konsumiert abstrahierte Events (z.B. `OutputConfigurationChangedEvent` könnte von `domain::theming` konsumiert werden, um Skalierungsfaktoren anzupassen).
- **Smithay-Nutzung**:
    - `DesktopState` (in `system::compositor::core`) ist der zentrale `CompositorState` und implementiert viele Smithay-Handler-Traits.
    - `DisplayHandle` wird für die Interaktion mit dem Wayland-Display verwendet.
    - Domänenlogik wird _innerhalb_ dieser Handler aufgerufen, um Entscheidungen zu treffen. Beispiel im `XdgShellHandler::new_toplevel`: Nachdem die `ToplevelSurface` von Smithay erstellt wurde, könnte `system::window_mechanics` (das `WindowManagementPolicyService` nutzt) aufgerufen werden, um die initiale Geometrie und den Workspace zu bestimmen.
- **Empfang von Domänenbefehlen durch Compositor / Übersetzung in Wayland-Operationen**:
    1. **Auslöser**: UI-Aktion oder internes Domänen-Event.
    2. **Domänenschicht**: Ein Domänen-Service (z.B. `WorkspaceManagerService.set_active_workspace(id)`) ändert seinen internen Zustand und sendet ein Domänen-Event (z.B. `WorkspaceEvent::ActiveWorkspaceChanged`).
    3. **Systemschicht-Adapter/Listener**: Eine Komponente in der Systemschicht (oft Teil des `DesktopState` oder ein dedizierter Listener, der Domänen-Events abonniert) empfängt das Domänen-Event.
    4. **Übersetzung in Wayland-Aktionen**:
        - Bei `ActiveWorkspaceChanged`: Der Listener im `DesktopState` oder `system::window_mechanics` würde:
            - Alle Fenster auf dem alten Workspace "verstecken" (z.B. `unmap` oder Sichtbarkeit ändern, falls vom Compositor unterstützt).
            - Alle Fenster auf dem neuen Workspace "anzeigen" und gemäß Layout-Policy anordnen.
            - Den Fokus auf das entsprechende Fenster im neuen Workspace setzen (`set_keyboard_focus` via `system::input`).
        - Bei Anforderung zur Fenstergrößenänderung von `domain::window_management_policy`: `system::window_mechanics` ruft `toplevel_surface.send_configure()` mit den neuen Bounds auf.
- **Weiterleitung von Wayland-Ereignissen an Domänenschicht**:
    - **Client-Requests**: Wayland-Client-Anfragen (z.B. `xdg_toplevel.set_title`) werden vom Smithay-Handler im `DesktopState` empfangen. Der Handler kann:
        1. Die Anfrage direkt bearbeiten, wenn keine komplexe Domänenlogik erforderlich ist (z.B. Fenstertitel im `ManagedWindow` aktualisieren).
        2. Eine Methode eines Domänen-Service aufrufen, um die Anfrage zu validieren oder Geschäftsregeln anzuwenden. Das Ergebnis des Domänen-Service-Aufrufs bestimmt die Antwort an den Wayland-Client.
    - **Systemereignisse**: Hardware-Events (z.B. Input von `system::input`, Monitor-Hotplug von `system::outputs`) werden in der Systemschicht verarbeitet. Diese können dann:
        1. Direkt zu Aktionen führen (z.B. Fokus verschieben).
        2. In allgemeinere System-Events (`SystemLayerEvent` via `system::event_bridge`) umgewandelt werden, die von der Domänenschicht abonniert werden können.
        3. Direkt Methoden von Domänen-Services aufrufen, wenn eine enge Kopplung sinnvoll ist.

### 3. Protokoll-Implementierungen

Die Domänenschicht interagiert nicht direkt mit diesen Protokollen, sondern über Abstraktionen in der Systemschicht.

- **D-Bus (`system::dbus_interfaces`)**:
    - Domänen-Services (z.B. `NotificationService`, `AIInteractionLogicService`, `GlobalSettingsService`) werden _nicht_ direkt D-Bus-Methoden aufrufen oder Signale senden.
    - Die Systemschicht stellt D-Bus-Clients (z.B. `UPowerClientService`, `LogindClientService`) bereit. Diese Clients:
        - Rufen D-Bus-Methoden auf.
        - Empfangen D-Bus-Signale.
        - Übersetzen diese in `SystemLayerEvent`s (via `system::event_bridge`) ODER rufen direkt Methoden von Domänen-Services auf (z.B. `LogindClientService` ruft bei `PrepareForSleep` eine Methode von `domain::power_management_policy` auf).
        - Beispiel: `UPowerClientService` sendet `UPowerSystemEvent::OnBatteryChanged`. Ein Adapter in der System- oder Domänenschicht könnte dies in ein `domain::common_events::PowerSourceChangedEvent` umwandeln, auf das `domain::power_management_policy` reagiert.
    - Der `system::dbus_interfaces::notifications_server` implementiert den `org.freedesktop.Notifications`-D-Bus-Dienst und ruft intern den `domain::user_centric_services::NotificationService`.
- **XDG Desktop Portals (`system::portals`)**:
    - Die Systemschicht implementiert das Backend für Portale (z.B. `FileChooser`, `Screenshot`).
    - Wenn ein Portal eine UI erfordert (z.B. Dateiauswahldialog), kommuniziert das Portal-Backend in der Systemschicht mit der UI-Schicht.
    - Die Domänenschicht könnte beteiligt sein, um Standardwerte oder Policies bereitzustellen (z.B. Standard-Speicherpfad von `GlobalSettingsService` für `FileChooser`).
- **Model Context Protocol (MCP) (`system::mcp_client`)**:
    - Der `domain::user_centric_services::AIInteractionLogicService` definiert die Logik für KI-Interaktionen und Einwilligungen.
    - Wenn eine KI-Aktion ausgeführt werden soll, ruft der `AIInteractionLogicService` eine Methode auf dem `system::mcp_client::SystemMcpService` (Trait-Implementierung).
    - Der `SystemMcpService` (in der Systemschicht) kümmert sich um die MCP-Kommunikation (Anfrage an Server, Empfang von Antwort/Notification) und leitet das Ergebnis (oder MCP-Fehler) an den `AIInteractionLogicService` zurück oder sendet ein `McpClientSystemEvent`.
    - API-Schlüssel für MCP-Server werden vom `SystemMcpService` über den `system::dbus_interfaces::secrets_service_client` sicher abgerufen.

### 4. Implementierbare Lösungen (Code-Skizzen, Algorithmen, Datenstrukturen)

#### 4.1. Traits der Domänenschicht (Beispiele)

Rust

```
// In src/domain/workspaces/manager/service.rs
#[async_trait]
pub trait WorkspaceManagerService: Send + Sync {
    async fn create_workspace(&self, name: Option<String>, /* ... */) -> Result<WorkspaceId, WorkspaceManagerError>;
    async fn get_active_workspace_id(&self) -> Option<WorkspaceId>;
    // ... weitere Methoden ...
    fn subscribe_to_workspace_events(&self) -> tokio::sync::broadcast::Receiver<WorkspaceEvent>;
}

// In src/domain/global_settings/service.rs
#[async_trait]
pub trait GlobalSettingsService: Send + Sync {
    async fn update_setting(&self, path: SettingPath, value: serde_json::Value) -> Result<(), GlobalSettingsError>;
    async fn get_setting_as_json(&self, path: &SettingPath) -> Result<serde_json::Value, GlobalSettingsError>;
    // ...
}
```

#### 4.2. Datenstrukturen (Beispiele)

Rust

```
// In src/domain/workspaces/core/types.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Workspace {
    id: WorkspaceId,
    name: String,
    persistent_id: Option<String>,
    layout_type: WorkspaceLayoutType,
    window_ids: HashSet<WindowIdentifier>,
    created_at: chrono::DateTime<chrono::Utc>,
    icon_name: Option<String>,
    accent_color_hex: Option<String>,
}

// In src/domain/global_settings/types.rs
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize, Default)]
#[serde(default, rename_all = "kebab-case")]
pub struct AppearanceSettings {
    pub active_theme_name: String,
    pub color_scheme: ColorScheme,
    pub accent_color_token: String, // Token-Name für die Akzentfarbe
    pub custom_accent_color_value: Option<novade_core::types::Color>, // Wenn Benutzer eine spezifische Farbe wählt
    // ...
}
```

#### 4.3. Interaktion Domäne (WorkspaceManager) &lt;-> System (Compositor/WindowMechanics)

**Szenario: Workspace wechseln**

1. **UI-Schicht**: Benutzer klickt auf Workspace-Switcher. UI ruft `WorkspaceManagerService.set_active_workspace(new_ws_id).await`.
2. **`domain::workspaces::DefaultWorkspaceManager`**:
    - Sperrt internen Zustand (`internal_state.lock().await`).
    - Validiert `new_ws_id`.
    - Aktualisiert `internal_state.active_workspace_id`.
    - Sendet `WorkspaceEvent::ActiveWorkspaceChanged { old_id, new_id }` über `event_publisher`.
    - Ruft `internal_state.config_provider.save_workspace_config(...).await` auf (um aktiven Workspace zu persistieren).
3. **`system::compositor::DesktopState` (oder ein dedizierter Listener in `system`):**
    - Abonniert `WorkspaceEvent::ActiveWorkspaceChanged`.
    - Bei Empfang des Events:
        - Identifiziert Fenster auf dem alten (`old_id`) und neuen (`new_id`) Workspace.
        - Informiert `system::window_mechanics`, das Layout für den neuen Workspace anzuwenden und Fenster entsprechend anzuzeigen/zu verstecken.
        - `system::window_mechanics` würde `domain::window_management_policy::calculate_workspace_layout()` aufrufen.
        - `system::window_mechanics` sendet dann `configure`-Events an die Wayland-Clients, um sie über neue Geometrien oder Sichtbarkeitsänderungen zu informieren.
        - Der Fokus wird auf das entsprechende Fenster im neuen Workspace gesetzt (z.B. das zuletzt fokussierte oder gemäß Policy).

**Algorithmus: Token Resolution in `domain::theming::logic::resolve_tokens_for_config`**

(Wie detailliert in der vorherigen Antwort unter Abschnitt 3.3 `domain::theming::logic` der Domänenschicht-Spezifikation, einschließlich Basis-Tokens, Varianten, Akzentfarben, User-Overrides und rekursiver Referenzauflösung mit Zyklenerkennung.)

#### 4.4. Hinweise auf Bibliotheken

- **Domänenschicht**:
    - `uuid`: Für eindeutige IDs.
    - `chrono`: Für Zeitstempel.
    - `serde`, `serde_json`, `toml`: Für Serialisierung/Deserialisierung von Konfigurationen und Datenstrukturen.
    - `thiserror`: Für Fehlerbehandlung.
    - `async-trait`: Für Traits mit `async fn`.
    - `tokio::sync::{Mutex, RwLock, broadcast}`: Für interne Zustandsverwaltung und Eventing.
    - `regex` (optional): Für `RuleCondition::AppNameMatches` in `domain::notifications_rules`.
- **Systemschicht (für Interaktion mit Domäne und Wayland/Smithay)**:
    - `smithay`: Kern-Toolkit für Wayland-Compositor.
    - `wayland-server`, `wayland-protocols`: Basis-Wayland-Bibliotheken.
    - `zbus`: Für D-Bus-Kommunikation.
    - `libinput` (via Smithay oder direkt): Für Eingabeverarbeitung.
    - `pipewire-rs`: Für Audio-Management.
    - `mcp_client_rs`: Für MCP-Client-Implementierung.
    - `ashpd` (optional): Als Referenz oder Nutzung für XDG Desktop Portal Interaktionen.

#### 4.5. Annahmen und Einschränkungen

- **Event-Bus**: Es wird ein `tokio::sync::broadcast`-basierter Event-Bus für die interne Kommunikation in der Domänenschicht und für die Benachrichtigung der System-/UI-Schicht angenommen.
- **Persistenz**: Die Domänenschicht delegiert die eigentliche Persistenz an die Kernschicht (`core::config`) über Provider-Traits. Das Format ist typischerweise TOML oder JSON.
- **Asynchronität**: Die Domänenschicht ist auf eine `async` Runtime (wie Tokio) angewiesen, die von der Anwendung bereitgestellt wird.
- **Wayland-Fokus**: Die Domänenschicht definiert Policies für den Fokus, die technische Umsetzung (Setzen des Wayland-Fokus auf eine `WlSurface`) erfolgt in der Systemschicht.

### 5. Maximale Detailtiefe

Die vorangegangenen Abschnitte und die referenzierten Dokumente (`B1-B4 Domänenschicht.md`, etc.) zielen darauf ab, diese Detailtiefe zu erreichen, indem:

- **Atomare Schritte**: Komplexe Logik (z.B. Token-Auflösung, Layout-Berechnung, Regel-Auswertung) wird in kleinere, verständliche Schritte und Funktionen zerlegt.
- **Präzise Schnittstellen**: Traits definieren klare Verträge. Datenstrukturen (`struct`, `enum`) sind mit all ihren Feldern, Typen und Ableitungen spezifiziert. Event-Payloads sind vollständig definiert. Fehlertypen sind granular und kontextreich.
- **Detaillierte technische Erklärungen**: Die "Warum"- und "Wie"-Aspekte jeder Designentscheidung und jedes Mechanismus werden erläutert, insbesondere bei Interaktionen zwischen Modulen oder Schichten.

Diese Spezifikation zielt darauf ab, Interpretationsspielräume zu minimieren und eine solide, direkt umsetzbare Grundlage für die Entwicklung der Domänenschicht von NovaDE und deren Integration mit der Systemschicht (Compositor) zu bieten.

Zusätzliche Details zur **Interaktion zwischen `domain::window_management_policy` und `system::compositor` / `system::window_mechanics`**:

1. **Fenstererstellung**:
    - Ein Wayland-Client erstellt eine neue `xdg_toplevel`-Oberfläche.
    - Der `XdgShellHandler` in `system::compositor::DesktopState` wird benachrichtigt (`new_toplevel`).
    - Der Handler erstellt ein `ManagedWindow`-Objekt (Systemschicht-Repräsentation).
    - Er ruft `domain::window_management_policy::WindowManagementPolicyService::get_initial_window_geometry()` auf und übergibt Informationen über das neue Fenster (z.B. `WindowLayoutInfo`), den aktuellen Workspace und das Layout.
    - Der Domänen-Service gibt eine initiale `RectInt` zurück.
    - `system::window_mechanics` (oder der `XdgShellHandler` direkt) sendet eine `configure`-Nachricht mit dieser Geometrie an den Client. Das `ManagedWindow` wird im `DesktopState::space` platziert.
2. **Fenster-Resize/Move (Client-initiiert)**:
    - Client sendet `xdg_toplevel.resize` oder `xdg_toplevel.move`.
    - `XdgShellHandler` empfängt dies. Er startet einen interaktiven Grab-Vorgang (`system::window_mechanics::interactive_ops`).
    - Während des Grabs ruft der Grab-Handler `domain::window_management_policy::WindowManagementPolicyService::calculate_snap_target()`, um Snapping-Vorschläge zu erhalten und die Geometrie anzupassen.
    - Nach Abschluss des Grabs wird die finale Geometrie dem Client via `configure` mitgeteilt.
3. **Automatisches Tiling durch Workspace-Layout-Änderung**:
    - Benutzer ändert Layout-Typ für einen Workspace (UI -> `domain::workspaces::WorkspaceManagerService::set_workspace_layout()`).
    - `WorkspaceManagerService` sendet `WorkspaceEvent::WorkspaceLayoutChanged`.
    - Ein Listener in `system::window_mechanics` (oder `DesktopState`) empfängt dieses Event.
    - Der Listener ruft `domain::window_management_policy::WindowManagementPolicyService::calculate_workspace_layout()` für den betroffenen Workspace auf, um die neuen Geometrien für alle Fenster zu erhalten.
    - `system::window_mechanics` wendet diese neuen Geometrien auf die `ManagedWindow`s an und sendet `configure`-Events an die Clients.

Diese Beispiele verdeutlichen den Datenfluss und die Verantwortlichkeiten: Die Domänenschicht entscheidet über die _Policy_, die Systemschicht setzt diese _mechanisch_ um und kommuniziert mit den Wayland-Clients.

### Systemschicht-Spezifikationen

Die Systemschicht (`novade-system` Crate) bildet die Brücke zwischen der abstrakten Domänenlogik und der konkreten Hardware bzw. den Low-Level-Systemdiensten. Sie ist maßgeblich für die Implementierung des Wayland-Compositors, die Eingabeverarbeitung, die Kommunikation über D-Bus und die Steuerung von Systemressourcen wie Audio und Energie.

#### 1. Modul: `system::compositor` (Smithay-basierter Wayland Compositor)

Dieses Modul implementiert die Kernlogik des Wayland-Compositors unter Verwendung des Smithay-Toolkits.

##### 1.1. Untermodul: `system::compositor::core`

Zuständig für zentrale Compositor-Zustände, die `DesktopState`-Struktur und Basis-Handler-Implementierungen.

- **Struktur `DesktopState`** (Zentrale Zustandsstruktur)
    
    - Diese Struktur ist das Herz des Compositors und implementiert die meisten Handler-Traits von Smithay. Sie hält Referenzen auf alle wichtigen Zustände und Dienste.
    - **Definition (Auszug und Erweiterung basierend auf `C1 System Implementierungsplan.md` und `3. System-Details.md`):**
        
        Rust
        
        ```
        // src/system/compositor/core/state.rs
        use smithay::{
            backend::{
                renderer::{gles2::Gles2Renderer, Frame, Renderer, Texture, ImportShm, ImportMem}, // Ggf. generischer Renderer-Trait verwenden
                allocator::dmabuf::DmabufState,
            },
            desktop::{Space, Window, WindowSurfaceType, WindowElement, PopupManager, layer_map_for_output, LayerSurface},
            input::{Seat, SeatState, SeatHandler, pointer::CursorImageStatus, keyboard::KeyboardHandle, pointer::PointerHandle, touch::TouchHandle},
            reexports::{
                calloop::{LoopHandle, EventLoop, LoopSignal, Interest, Mode, PostAction},
                wayland_server::{Display, DisplayHandle, Client, backend::{GlobalId, ClientId}, protocol::{wl_surface::WlSurface, wl_output, wl_seat, wl_shm}},
            },
            utils::{Clock, Logical, Physical, Point, Rectangle, Scale, Serial, Transform, SERIAL_COUNTER, zindagi, Buffer as SmithayBuffer},
            wayland::{
                buffer::BufferHandler,
                compositor::{CompositorState, CompositorHandler, CompositorClientState, SurfaceAttributes as WlSurfaceAttributes, add_destruction_hook, TraversalAction},
                output::{OutputManagerState, Output, PhysicalProperties, Mode as WlMode, OutputHandler},
                shell::{
                    xdg::{XdgShellState, XdgShellHandler, XdgToplevelSurfaceData, XdgPopupSurfaceData, SurfaceCachedState, XdgWmBaseClientData, ToplevelSurface, PopupSurface, PositionerState, XdgShellSurfaceUserData, XdgSurfaceConfigure, xdg_toplevel_handle::XdgToplevelUserData},
                    kde_decoration::KdeDecorationManagerState, // Beispiel für SSD
                    wlr_layer::{WlrLayerShellState, LayerShellHandler, LayerShellSurfaceData, Layer as WlrLayer, KeyboardInteractivity as WlrKeyboardInteractivity, Anchor as WlrAnchor},
                },
                shm::{ShmState, ShmHandler, BufferData as ShmBufferData, ShmFormat},
                seat::{WaylandSeatData, SeatUserApi},
                selection::data_device::{DataDeviceState, DataDeviceHandler, ServerDndGrabHandler, DataDeviceUserData},
                selection::primary_selection::{PrimarySelectionState, PrimarySelectionHandler},
                input_method::InputMethodManagerState, // Für IME
                relative_pointer::RelativePointerManagerState,
                pointer_constraints::PointerConstraintsState,
                viewporter::ViewporterState,
                presentation::PresentationState,
                xdg_activation::{XdgActivationState, XdgActivationHandler, XdgActivationTokenData, XdgActivationTokenSurfaceData},
                foreign_toplevel::ForeignToplevelManagerState, // wlr-foreign-toplevel-management
                output_manager::OutputManagerHandler, // für xdg-output
                power_manager::OutputPowerManagementState, // für wlr-output-power-management
            },
            xwayland::{XWayland, XWaylandEvent, XWaylandClientData, xwm::{X11Wm, XwmHandler, X11Surface, XwmId}},
        };
        use crate::core::types::{RectInt, PointInt as CorePointInt, SizeInt as CoreSizeInt}; // NovaDE Core Typen
        use crate::domain::workspaces::core::types::{WorkspaceId, WindowIdentifier as DomainWindowIdentifier};
        use crate::domain::window_management_policy::{WindowManagementPolicyService, WindowPolicyOverrides, TilingMode, WorkspaceWindowLayout, WindowLayoutInfo};
        use crate::domain::workspaces::manager::WorkspaceManagerService;
        use crate::domain::theming::ThemingEngine; // Für Cursor-Theming, Fensterdekorationen
        use crate::domain::global_settings_and_state_management::GlobalSettingsService;
        use crate::system::input::keyboard::xkb_config::XkbKeyboardData;
        use super::surface_management::SurfaceData;
        use std::collections::HashMap;
        use std::sync::{Arc, Mutex as StdMutex}; // Mutex aus std oder parking_lot
        use tokio::sync::Mutex as TokioMutex; // Für async Domain Services
        use uuid::Uuid;
        use std::cell::RefCell;
        use std::rc::Rc;
        use std::time::Duration;
        
        pub const CLOCK_ID: usize = 0; // Für Smithay Clock
        
        // Repräsentiert ein Fenster im Compositor, das von Space verwaltet wird.
        // Muss WindowElement implementieren.
        #[derive(Debug, Clone, PartialEq)]
        pub struct NovaWindowElement {
            pub id: DomainWindowIdentifier, // Eigene ID für die Zuordnung zur Domäne
            pub xdg_toplevel: Option<ToplevelSurface>, // Smithay ToplevelSurface
            pub xdg_popup: Option<PopupSurface>,
            pub x11_surface: Option<X11Surface>, // Für XWayland
            pub layer_surface: Option<LayerSurface>, // Für wlr-layer-shell
            // Weitere Felder für internen Zustand des Compositors, z.B. ob es gemappt ist,
            // Geometrie, Workspace-Zugehörigkeit etc.
            pub current_geometry: RectInt,
            pub workspace_id: Option<WorkspaceId>,
            pub is_mapped: bool,
        }
        
        // Implementierung von WindowElement für NovaWindowElement
        impl WindowElement for NovaWindowElement {
            fn id(&self) -> usize {
                // Eine stabile usize ID generieren, z.B. aus der Uuid der DomainWindowIdentifier
                let mut hasher = std::collections::hash_map::DefaultHasher::new();
                self.id.hash(&mut hasher);
                std::hash::Hasher::finish(&hasher) as usize
            }
        
            fn wl_surface(&self) -> Option<WlSurface> {
                if let Some(toplevel) = &self.xdg_toplevel {
                    Some(toplevel.wl_surface().clone())
                } else if let Some(popup) = &self.xdg_popup {
                    Some(popup.wl_surface().clone())
                } else if let Some(x11) = &self.x11_surface {
                    Some(x11.wl_surface().clone())
                } else if let Some(layer) = &self.layer_surface {
                    Some(layer.wl_surface().clone())
                } else {
                    None
                }
            }
            // ... (weitere Methoden von WindowElement wie bbox, is_suspended, send_frame etc.)
            // müssen sorgfältig implementiert werden, oft durch Delegation an die
            // zugrundeliegende Smithay-Surface (ToplevelSurface, PopupSurface etc.)
            // oder basierend auf current_geometry und is_mapped.
        }
        
        
        pub struct DesktopState {
            pub display_handle: DisplayHandle,
            pub loop_handle: LoopHandle<'static, Self>,
            pub loop_signal: LoopSignal,
            pub clock: Clock<u64>,
        
            // Smithay States
            pub compositor_state: CompositorState,
            pub shm_state: ShmState,
            pub output_manager_state: OutputManagerState,
            pub seat_state: SeatState<Self>,
            pub data_device_state: DataDeviceState,
            pub xdg_shell_state: XdgShellState,
            pub xdg_activation_state: XdgActivationState,
            pub layer_shell_state: WlrLayerShellState,
            pub foreign_toplevel_state: ForeignToplevelManagerState,
            pub output_power_management_state: OutputPowerManagementState,
            pub presentation_state: PresentationState,
            // ... weitere States nach Bedarf ...
        
            // Compositor-interne Daten
            pub space: Space<NovaWindowElement>, // Verwaltet Fensterpositionen und Stapelreihenfolge
            pub popups: PopupManager,
            pub seat: Seat<Self>,
            pub seat_name: String,
            pub keyboard_data_map: HashMap<String /* seat_name or device_id */, XkbKeyboardData>,
            pub current_cursor_status: Arc<StdMutex<CursorImageStatus>>, // Für Cursor-Rendering
            pub pointer_location: Point<f64, Logical>, // Globale Zeigerposition
        
            // Liste aller Fenster (eigene Verwaltung parallel zu space, für leichtere Domänen-Interaktion)
            // Key: DomainWindowIdentifier, Value: NovaWindowElement (oder Arc<NovaWindowElement>)
            pub windows: HashMap<DomainWindowIdentifier, Arc<NovaWindowElement>>,
        
            // XWayland
            pub xwayland: Option<XWayland>,
            pub xwm: Option<Arc<TokioMutex<X11Wm>>>, // X11 Window Manager für XWayland
        
            // Domänen-Service Handles (Arc<TokioMutex<...>> da Domänenservices async sein können)
            pub window_policy_service: Arc<TokioMutex<dyn WindowManagementPolicyService>>,
            pub workspace_manager_service: Arc<TokioMutex<dyn WorkspaceManagerService>>,
            pub theming_engine: Arc<TokioMutex<dyn ThemingEngine>>, // Für Cursor, Dekorationen
            pub settings_service: Arc<TokioMutex<dyn GlobalSettingsService>>,
        
            // System Services (D-Bus Clients etc. - werden hier nicht direkt gehalten, sondern über
            // das SystemEventBridge oder spezifische Handler-Methoden angesprochen)
        
            // Renderer (Beispiel, konkreter Typ hängt von gewählter Implementierung ab)
            // pub renderer: Option<Gles2Renderer>,
            // pub dmabuf_state: Option<DmabufState>, // Wenn DMABUF unterstützt wird
        
            pub last_render_time_ns: u64, // Für Animationen und Frame-Timing
            pub running: Arc<std::sync::atomic::AtomicBool>, // Um den Event-Loop zu beenden
        }
        
        impl DesktopState {
            pub fn new(
                event_loop: &mut EventLoop<'static, Self>,
                display: &mut Display<Self>,
                // Domänen-Services werden injiziert
                window_policy_service: Arc<TokioMutex<dyn WindowManagementPolicyService>>,
                workspace_manager_service: Arc<TokioMutex<dyn WorkspaceManagerService>>,
                theming_engine: Arc<TokioMutex<dyn ThemingEngine>>,
                settings_service: Arc<TokioMutex<dyn GlobalSettingsService>>,
            ) -> Self {
                let display_handle = display.handle();
                let loop_handle = event_loop.handle();
                let loop_signal = event_loop.get_signal();
        
                let clock = Clock::new(None); // Optional: Logger
        
                // Smithay States initialisieren
                let compositor_state = CompositorState::new::<Self>(&display_handle, None);
                let shm_state = ShmState::new::<Self>(&display_handle, vec![], None); // Keine zusätzlichen SHM-Formate initial
                let output_manager_state = OutputManagerState::new_with_xdg_output::<Self>(&display_handle); // Integriert xdg-output
                let mut seat_state = SeatState::new();
                let data_device_state = DataDeviceState::new::<Self>(&display_handle, None);
                let xdg_shell_state = XdgShellState::new::<Self>(&display_handle, None);
                let xdg_activation_state = XdgActivationState::new::<Self>(&display_handle, None);
                let layer_shell_state = WlrLayerShellState::new::<Self>(&display_handle, None);
                let foreign_toplevel_state = ForeignToplevelManagerState::new::<Self>(&display_handle);
                let output_power_management_state = OutputPowerManagementState::new::<Self>(&display_handle);
                let presentation_state = PresentationState::new::<Self>(&display_handle, clock.id() as u32);
        
        
                // Seat erstellen
                let seat_name = "seat0".to_string();
                let seat = seat_state.new_wl_seat(&display_handle, seat_name.clone(), None);
                // Capabilities (Keyboard, Pointer, Touch) werden später hinzugefügt, wenn das Input-Backend initialisiert ist.
        
                let space = Space::new(None);
                let popups = PopupManager::new(None);
        
                Self {
                    display_handle,
                    loop_handle,
                    loop_signal,
                    clock,
                    compositor_state,
                    shm_state,
                    output_manager_state,
                    seat_state,
                    data_device_state,
                    xdg_shell_state,
                    xdg_activation_state,
                    layer_shell_state,
                    foreign_toplevel_state,
                    output_power_management_state,
                    presentation_state,
                    space,
                    popups,
                    seat,
                    seat_name,
                    keyboard_data_map: HashMap::new(),
                    current_cursor_status: Arc::new(StdMutex::new(CursorImageStatus::Default)),
                    pointer_location: (0.0, 0.0).into(),
                    windows: HashMap::new(),
                    xwayland: None, // Wird später initialisiert
                    xwm: None,      // Wird später initialisiert
                    window_policy_service,
                    workspace_manager_service,
                    theming_engine,
                    settings_service,
                    // renderer: None,
                    // dmabuf_state: None,
                    last_render_time_ns: 0,
                    running: Arc::new(std::sync::atomic::AtomicBool::new(true)),
                }
            }
        
            /// Fügt ein neues Fenster zum DesktopState hinzu (sowohl zu `space` als auch zur `windows` Map).
            pub fn map_window(&mut self, window: Arc<NovaWindowElement>, activate: bool) {
                let initial_pos: Point<i32, Logical> = window.geometry().loc; // Oder von Policy bestimmt
                self.space.map_window(window.clone(), initial_pos, activate);
                self.windows.insert(window.id.clone(), window);
                // Ggf. Schaden an Output hinzufügen, wo das Fenster erscheint
                self.damage_outputs_for_geometry(window.geometry());
            }
        
            pub fn unmap_window(&mut self, window_id: &DomainWindowIdentifier) {
                if let Some(window_arc) = self.windows.remove(window_id) {
                    let old_geometry = window_arc.geometry();
                    self.space.unmap_window(&window_arc);
                    self.damage_outputs_for_geometry(old_geometry); // Schaden, wo Fenster war
                }
            }
        
            fn damage_outputs_for_geometry(&self, geometry: RectInt) {
                let geo_logical: Rectangle<i32, Logical> = Rectangle::from_loc_and_size(
                    (geometry.x, geometry.y),
                    (geometry.width as i32, geometry.height as i32)
                );
                for output in self.space.outputs() {
                    let output_geo = self.space.output_geometry(output).unwrap_or_default();
                    if output_geo.overlaps(geo_logical) {
                        output.damage_whole(); // Vereinfacht, oder spezifischer Schaden
                    }
                }
            }
        
            pub fn window_by_domain_id(&self, id: &DomainWindowIdentifier) -> Option<Arc<NovaWindowElement>> {
                self.windows.get(id).cloned()
            }
        }
        ```
        
    - **Handler-Implementierungen für `DesktopState`**:
        - `CompositorHandler`:
            - `client_compositor_state`: Greift auf `ClientCompositorData` zu, das bei `Client::data_map()` gespeichert wird (muss beim Client-Connect initialisiert werden).
            - `commit(surface: &WlSurface)`:
                1. Ruft `smithay::wayland::compositor::with_states(surface, |states| { ... })` auf.
                2. Holt `SurfaceData` aus `states.data_map()`.
                3. Aktualisiert `SurfaceData.current_buffer_info` mit `states.cached_state.current::<WlSurfaceAttributes>().buffer` etc.
                4. Akkumuliert Schaden aus `states.cached_state.current::<WlSurfaceAttributes>().damage` in `SurfaceData.damage_buffer_coords`.
                5. Wenn die Oberfläche eine Rolle hat (z.B. Toplevel, Popup, Layer), wird die spezifische Commit-Logik dieser Rolle (oft in deren eigenem Handler oder durch Smithay's `Space`/`Window`) ausgelöst.
                6. Markiert die Oberfläche für Neuzeichnung durch das Rendering-Backend.
            - `new_surface(surface: &WlSurface)`: Initialisiert `SurfaceData::new(...)` und fügt es zu `surface.data_map()` hinzu. Registriert `add_destruction_hook`.
            - `destroyed(surface: &WlSurface)`: Bereinigt `SurfaceData` und andere Referenzen (oft im `destruction_hook`).
        - `ShmHandler`:
            - `shm_state()`: Gibt `&self.shm_state` zurück.
        - `BufferHandler`:
            - `buffer_destroyed(buffer: &wl_buffer::WlBuffer)`: Informiert den Renderer, die Textur für diesen Puffer freizugeben. Entfernt Puffer-Referenzen aus allen `SurfaceData`-Instanzen.
        - `OutputHandler`:
            - `output_state()`: Gibt `&mut self.output_manager_state` zurück.
            - `new_output(wl_output: &wl_output::WlOutput, output_data: &smithay::wayland::output::OutputData)`: Hier wird ein `Output` von Smithay (`output_data.inner()`) mit einem `NovaWindowElement` oder einer internen Repräsentation im `DesktopState.space` assoziiert. Domänendienste werden ggf. über den neuen Output informiert.
        - `SeatHandler`:
            - `seat_state()`: Gibt `&mut self.seat_state` zurück.
            - `focus_changed(seat: &Seat<Self>, focused: Option<&WlSurface>)`: Aktualisiert den internen Fokus-Zustand. Informiert die Domänenschicht (`WindowManagementPolicyService`) über die Fokusänderung, die dann entscheidet, ob z.B. Fensterdekorationen (aktiv/inaktiv) angepasst werden müssen. Der `KeyboardHandle` sendet bereits `wl_keyboard.enter/leave`.
            - `cursor_image(seat: &Seat<Self>, image: CursorImageStatus)`: Aktualisiert `self.current_cursor_status`. Der Renderer zeichnet diesen Cursor an `self.pointer_location`. Bei `CursorImageStatus::Named`, wird `ThemingEngine` konsultiert, um den Cursor-Namen in eine Textur aufzulösen.
        - `DataDeviceHandler`, `ServerDndGrabHandler`: Implementierungen für Zwischenablage und Drag & Drop, die oft mit `Seat` und Fokus interagieren.
        - `XdgShellHandler`: Detailliert im Untermodul `system::compositor::xdg_shell`.
        - `LayerShellHandler`: Detailliert im Untermodul `system::compositor::layer_shell`.
        - `XdgActivationHandler`: Handhabt Aktivierungs-Tokens.
        - `OutputManagerHandler` (für `xdg-output`): Sendet logische Größe/Position an Clients.
        - `XwmHandler` (für XWayland): Handhabt X11-Fenster-Interaktionen.
- **Globale Erstellung**: Funktionen wie `create_core_compositor_globals`, `create_shm_global`, `create_xdg_shell_global` (aus `C1 System Implementierungsplan.md`) werden beim Start des Compositors aufgerufen, um die Wayland-Globals zu registrieren und in `DesktopState` zu speichern.
    

##### 1.2. Untermodul: `system::compositor::surface_management`

- **Struktur `SurfaceData`**
    - Wie in `C1 System Implementierungsplan.md` definiert, mit Feldern für `id (Uuid)`, `client_id`, `role`, `current_buffer_info: Option<AttachedBufferInfo>`, `texture_handle: Mutex<Option<Box<dyn RenderableTexture>>>` (wobei `RenderableTexture` aus `renderer_interface` kommt), `damage_buffer_coords`, `damage_surface_coords` (transformierter Schaden), `opaque_region_surface_local`, `input_region_surface_local`, `parent`, `children`, Hooks, `surface_viewporter_state`, `surface_presentation_state`, `surface_scale_factor`.
    - **`AttachedBufferInfo { buffer: WlBuffer, scale: i32, transform: Transform, dimensions: Size<i32, SmithayBuffer> }`**
    - Methoden: `new`, `set_role`, `attach_buffer`, `take_damage_buffer_coords`.
- **Funktionen**: `get_surface_data(surface: &WlSurface) -> Option<Arc<StdMutex<SurfaceData>>>`.

##### 1.3. Untermodul: `system::compositor::shm`

- Handhabt `wl_shm`-Protokoll (SHM-Puffer).
- `ShmError` und Integration in `DesktopState` (Implementierung von `ShmHandler`, `BufferHandler`, `GlobalDispatch<WlShm, _>`).

##### 1.4. Untermodul: `system::compositor::xdg_shell`

Implementiert das `xdg_shell`-Protokoll für Toplevel-Fenster und Popups.

- **Fehler**: `XdgShellError` (z.B. `InvalidSurfaceRole`, `WindowHandlingError`, `InvalidAckConfigureSerial`).
- **Struktur `NovaWindowElement` (anstelle von `ManagedToplevel`/`ManagedPopup` aus C1, zur direkten Nutzung mit `Space<NovaWindowElement>`):**
    - Implementiert `smithay::desktop::WindowElement`.
    - Felder: `id: DomainWindowIdentifier`, `xdg_toplevel: Option<ToplevelSurface>`, `xdg_popup: Option<PopupSurface>`, `x11_surface: Option<X11Surface>`, `layer_surface: Option<LayerSurface>`, `current_geometry: RectInt`, `workspace_id: Option<WorkspaceId>`, `is_mapped: bool`.
    - Methoden: `new_toplevel`, `new_popup`, `wl_surface()`.
- **`XdgShellHandler` für `DesktopState`**:
    - `new_toplevel(surface: ToplevelSurface)`:
        1. Erstellt `DomainWindowIdentifier`.
        2. Ruft `domain::window_management_policy::WindowManagementPolicyService::get_initial_window_geometry()` auf, um initiale Geometrie und Workspace zu bestimmen.
        3. Erstellt `NovaWindowElement::new_toplevel(surface, domain_id)`.
        4. Speichert `XdgToplevelSurfaceData` in `surface.wl_surface().data_map()`.
        5. Sendet initiale `configure`-Events an den Client (`surface.send_configure()`).
        6. Fügt das `NovaWindowElement` zu `DesktopState.space` und `DesktopState.windows` hinzu (`DesktopState.map_window()`).
        7. Informiert `domain::workspaces::WorkspaceManagerService` über das neue Fenster.
    - `new_popup(surface: PopupSurface, _client_data: &XdgWmBaseClientData)`:
        1. Erstellt `NovaWindowElement::new_popup(...)`.
        2. Berechnet Popup-Geometrie basierend auf `PositionerState` und Elter-Geometrie.
        3. Sendet `configure`-Events.
        4. Fügt zu `DesktopState.popups` (PopupManager) hinzu.
    - `map_toplevel(surface: &ToplevelSurface)`: Wird aufgerufen, wenn der Client die Oberfläche mappen will. Setzt `window.is_mapped = true`. Informiert Domänenschicht (Workspace-Zuweisung). Aktualisiert `Space`.
    - `unmap_toplevel(surface: &ToplevelSurface)`: Setzt `window.is_mapped = false`. Informiert Domänenschicht. Entfernt ggf. aus `Space`.
    - `ack_configure(surface: WlSurface, configure_data: XdgSurfaceConfigure)`: Verarbeitet Bestätigung vom Client. Aktualisiert `SurfaceCachedState`.
    - **Request-Handler** (z.B. `toplevel_request_set_title`, `toplevel_request_set_maximized`, `toplevel_request_move`):
        1. Findet das zugehörige `NovaWindowElement` über `surface.wl_surface()`.
        2. Aktualisiert den internen Zustand des `NovaWindowElement` (z.B. Titel, angeforderter maximierter Zustand).
        3. Für Zustandsänderungen, die eine Policy-Entscheidung erfordern (z.B. Maximieren, Move, Resize): Ruft `domain::window_management_policy` auf.
        4. `system::window_mechanics` wendet die neue Geometrie/den neuen Zustand an und sendet `configure`-Events an den Client.
        5. Für interaktive Moves/Resizes: Startet einen Grab über `Seat::start_pointer_grab` (siehe `system::input` und `system::window_mechanics`).
    - `toplevel_destroyed(toplevel: ToplevelSurface)`: Bereinigt das `NovaWindowElement` aus `DesktopState.space` und `DesktopState.windows`. Informiert Domänenschicht.
    - `popup_destroyed(popup: PopupSurface)`: Analog für Popups.

##### 1.5. Untermodul: `system::compositor::renderer_interface`

- Definiert abstrakte Traits `FrameRenderer` und `RenderableTexture` sowie `RenderElement` Enum (wie in `C1 System Implementierungsplan.md`). Dies entkoppelt die Compositor-Logik von spezifischen Rendering-Backends (z.B. GLES2, Vulkan).

##### 1.6. Interaktion Compositor mit Domänenschicht (Fenstermanagement)

- Der `DesktopState` hält einen `Arc<TokioMutex<dyn WindowManagementPolicyService>>`.
- Wenn ein neues Fenster gemappt wird (`XdgShellHandler::map_toplevel`) oder ein Workspace-Layout sich ändert (Event von `WorkspaceManagerService`):
    1. Der Compositor (oder `system::window_mechanics`) ruft `window_policy_service.calculate_workspace_layout(workspace_id, windows_on_ws, available_area, ...).await`.
    2. Das Ergebnis (`WorkspaceWindowLayout`) enthält die Zielgeometrien für alle Fenster.
    3. Der Compositor (via `system::window_mechanics`) aktualisiert die `current_geometry` der `NovaWindowElement`s und sendet `configure`-Events an die Wayland-Clients.

##### 1.7. Smithay Typen und ihre Verwendung (Beispiele)

- **`DisplayHandle`**: Wird verwendet, um Globals zu erstellen, Clients zu flushen, den Event-Loop-Deskriptor zu erhalten.
- **`CompositorState`**: Verwaltet `wl_compositor` und `wl_subcompositor` Globals und Client-Zustände.
- **`XdgShellState`**: Verwaltet `xdg_wm_base` Global und Client-Zustände.
- **`ToplevelSurface`, `PopupSurface`**: Smithay-Repräsentationen von XDG-Oberflächen, die Methoden zum Senden von `configure`-Events, Setzen von Zuständen (maximiert, fokussiert etc.) bereitstellen.
- **`SurfaceAttributes`**: Wird im `CompositorHandler::commit` verwendet, um angehängte Puffer, Schadenregionen etc. abzurufen.
- **`Space<NovaWindowElement>`**: Hauptkomponente von Smithay zur Verwaltung von Fenstern im 2D-Raum, inklusive Stapelreihenfolge, Output-Zuordnung und Schadensberechnung für das Rendering.
- **`Output`**: Repräsentiert einen physischen Monitor. `DesktopState.space` verwaltet eine Liste von `Output`s.
- **`Seat`**: Repräsentiert einen Satz von Eingabegeräten. `KeyboardHandle`, `PointerHandle`, `TouchHandle` werden verwendet, um Wayland-Events an Clients zu senden.

#### 2. Modul: `system::input` (Libinput-basierte Eingabeverarbeitung)

Dieses Modul verarbeitet Eingaben von Tastaturen, Mäusen, Touchpads etc.

##### 2.1. Untermodul: `system::input::seat_manager`

- **`DesktopState` (Erweiterung für Input):** Hält `SeatState<Self>`, `Seat<Self>`, `seat_name`, `keyboard_data_map: HashMap<String, XkbKeyboardData>`, `current_cursor_status`, `pointer_location`.
- **`SeatHandler` für `DesktopState`**:
    - `focus_changed(seat: &Seat<Self>, focused: Option<&WlSurface>)`: Aktualisiert internen Fokus, benachrichtigt Domäne.
    - `cursor_image(seat: &Seat<Self>, image: CursorImageStatus)`: Aktualisiert `current_cursor_status`. Der Renderer verwendet dies und `pointer_location`.
- **Funktion `create_seat(state: &mut DesktopState, ...)`**: Erstellt `Seat` und fügt Capabilities (Tastatur, Zeiger, Touch) hinzu, wenn die entsprechenden Geräte vom Backend gemeldet werden.

##### 2.2. Untermodul: `system::input::libinput_handler`

- **`LibinputSessionInterface`**: Implementiert `smithay::backend::input::LibinputInterface` für das Öffnen/Schließen von Geräten (interagiert mit `systemd-logind` oder `libseat`).
- **`init_libinput_backend(...)`**: Erstellt `LibinputInputBackend`.
- **`process_input_event(desktop_state: &mut DesktopState, event: InputEvent<LibinputInputBackend>, seat_name: &str)`**: Haupt-Dispatcher für libinput-Events.
    - `DeviceAdded`/`DeviceRemoved`: Aktualisiert Seat-Capabilities.
    - Leitet spezifische Events (Keyboard, Pointer, Touch, Gesture) an die Übersetzer-Funktionen weiter.

##### 2.3. Untermodul: `system::input::keyboard`

- **`XkbKeyboardData { context, keymap, state, repeat_timer, ... }`**: Verwaltet XKB-Kontext, Keymap und Zustand pro Tastatur.
- **`handle_keyboard_key_event(...)`**:
    1. Aktualisiert `xkbcommon::State`.
    2. Ruft `KeyboardHandle::modifiers()` und `KeyboardHandle::key()` auf, um Wayland-Events an den Client mit Fokus zu senden.
    3. Handhabt Tastenwiederholung mit `calloop::TimerHandle`.
- **`set_keyboard_focus(desktop_state: &mut DesktopState, ..., surface: Option<&WlSurface>, serial: Serial)`**: Ruft `KeyboardHandle::set_focus()`.

##### 2.4. Untermodul: `system::input::pointer`

- **`handle_pointer_motion_event(...)`**:
    1. Aktualisiert `DesktopState.pointer_location`.
    2. Findet die Oberfläche unter dem Cursor (`find_surface_and_coords_at_global_point` iteriert über `DesktopState.space.elements_under(pos)`).
    3. Ruft `PointerHandle::motion()` auf (Smithay sendet `enter`/`leave` und `motion`).
- **`handle_pointer_button_event(...)`**:
    1. Ruft `PointerHandle::button()`.
    2. Wenn Taste gedrückt: Löst ggf. Fokusänderung (`set_keyboard_focus`) oder Start einer interaktiven Fensteroperation (`system::window_mechanics` startet Grab via `Seat::start_pointer_grab`) aus.
- **`handle_pointer_axis_event(...)`**: Ruft `PointerHandle::axis()`.

##### 2.5. Untermodul: `system::input::touch`

- Handhabt Touch-Events (`Down`, `Up`, `Motion`, `Frame`, `Cancel`) analog zur Zeigerverarbeitung, aber pro Touchpunkt (`slot_id`).
- Verwaltet den Fokus für jeden aktiven Touchpunkt.

#### 3. Protokoll-Implementierungen (Systemschicht-Perspektive)

- **D-Bus**: Die `system::dbus_interfaces` Clients (z.B. `UPowerClientService`) laufen als asynchrone Tasks, abonnieren D-Bus-Signale und stellen `async fn`-Methoden für Anfragen bereit. Ergebnisse/Events werden über den `system::event_bridge` (z.B. `tokio::sync::broadcast`) an andere System- oder Domänenkomponenten gesendet. Der `FreedesktopNotificationsServer` implementiert die D-Bus-Schnittstelle und ruft intern `domain::user_centric_services::NotificationService` auf.
- **XDG Desktop Portals**: Das `system::portals`-Backend implementiert die D-Bus-Schnittstellen für Portale. Bei Methodenaufrufen (z.B. `OpenFile`), die eine UI-Interaktion erfordern, kommuniziert es mit der UI-Schicht (z.B. über einen dedizierten Mechanismus oder den `SystemEventBridge`, um einen Dialog anzufordern) und wartet auf die Antwort, bevor es dem Portal-Client antwortet. Für Screenshots/ScreenCast interagiert es mit `system::compositor::screencopy` und PipeWire.
- **MCP**: Der `system::mcp_client::DefaultSystemMcpService` nutzt `McpConnectionManager`, um Verbindungen zu MCP-Servern (lokal/remote) herzustellen und zu verwalten. Er nimmt Anfragen von `domain::user_centric_services::AIInteractionLogicService` entgegen, formuliert MCP-Nachrichten, sendet sie über `mcp_client_rs::McpClient`, verarbeitet Antworten und leitet Notifications/Ergebnisse zurück an die Domänenschicht (via `McpClientSystemEvent` über den `SystemEventBridge`). API-Schlüssel werden vom `SecretsServiceClientService` geholt.

#### 4. Implementierbare Lösungen (Systemschicht-Code-Skizzen)

**Beispiel: `DesktopState` implementiert `CompositorHandler` (Auszug `commit`)**

Rust

```
// In src/system/compositor/core/handlers.rs (oder state.rs)
impl CompositorHandler for DesktopState {
    fn compositor_state(&mut self) -> &mut CompositorState {
        &mut self.compositor_state
    }

    fn client_compositor_state<'a>(&self, client: &'a Client) -> &'a CompositorClientState {
        // Annahme: ClientCompositorData ist in Client::data_map() gespeichert
        &client.get_data::<crate::system::compositor::core::state::ClientCompositorData>().unwrap().compositor_state
    }

    fn commit(&mut self, surface: &WlSurface) {
        tracing::debug!(surface_id = ?surface.id(), "Commit for WlSurface");

        // Schadenverfolgung und Puffer-Management mit Smithay-Helpern
        if let Err(err) = smithay::wayland::compositor::handle_commit_accumulate_damage_states(surface, |_, states| {
            // Zugriff auf SurfaceData, das an `states.data_map` hängt
            let data_guard = SurfaceData::from_states(states); // Hilfsmethode
            if let Some(mut data) = data_guard {
                data.update_buffer_info_from_states(states); // Aktualisiert current_buffer_info
            }
        }) {
            tracing::warn!("Error during commit for surface {:?}: {}", surface.id(), err);
        }

        // Finde das NovaWindowElement, das diese WlSurface repräsentiert
        let mut window_to_damage: Option<Arc<NovaWindowElement>> = None;
        for win_element_arc in self.space.elements() { // Iteriere über Elemente im Space
            if win_element_arc.wl_surface().as_ref() == Some(surface) {
                window_to_damage = Some(win_element_arc.clone());
                break;
            }
        }
        // Alternativ, wenn SurfaceData eine Referenz/ID zum NovaWindowElement hält:
        // let surface_data = SurfaceData::from_surface(surface);
        // let window_to_damage = surface_data.and_then(|sd| self.windows.get(&sd.domain_id)).cloned();


        if let Some(window_arc) = window_to_damage {
            // Markiere das Fenster im Space als beschädigt, damit es neu gezeichnet wird.
            // Smithay's Space kann dies oft automatisch handhaben, wenn der Schaden
            // auf der WlSurface korrekt registriert wurde.
            // Expliziter Schaden am Space-Element:
            self.space.damage_window(&window_arc, None, None);
        } else {
            // Layer-Surfaces, Cursor-Surfaces etc. müssen ggf. anders behandelt werden.
            // Für Popups verwaltet PopupManager den Schaden.
            if self.popups.surface_under(self.pointer_location, WindowSurfaceType::Wayland).map_or(false, |(p, _)| p.wl_surface() == surface) {
                 // Popup unter Cursor, PopupManager sollte Schaden handhaben
            } else if let Some(layer) = self.layer_shell_state.layer_for_surface(surface) {
                // Layer Surface Schaden
                layer_map_for_output(layer.output()).damage_surface(surface, (0,0), None);
            }
            // Hier könnte auch der Cursor-Surface-Commit behandelt werden.
        }

        // Wenn es sich um eine XDG-Oberfläche handelt, führt XdgShellHandler::surface_commit weitere Aktionen aus.
        // Smithay leitet den Commit oft automatisch an den XdgShellHandler weiter, wenn die Rolle gesetzt ist.
        // Manuelles Aufrufen ist selten nötig, außer für sehr spezifische Logik.
        if smithay::wayland::shell::xdg::is_xdg_surface(surface) {
            if let Err(e) = smithay::wayland::shell::xdg::handle_commit_xdg(surface, self) {
                 tracing::warn!("XDG commit handling failed for {:?}: {}", surface, e);
            }
        }
        // Ähnlich für wlr-layer-shell
        if smithay::wayland::shell::wlr_layer::is_layer_surface(surface) {
             if let Err(e) = smithay::wayland::shell::wlr_layer::handle_commit_layer(surface, self) {
                 tracing::warn!("Layer-shell commit handling failed for {:?}: {}", surface, e);
             }
        }
    }
    // ... (new_surface, destroyed etc.)
}
```

**Beispiel: `DesktopState` implementiert `SeatHandler` (Auszug `cursor_image`)**

Rust

```
// In src/system/input/seat_manager.rs (oder wo SeatHandler implementiert wird)
impl SeatHandler for DesktopState {
    type KeyboardFocus = WlSurface;
    type PointerFocus = WlSurface;
    type TouchFocus = WlSurface;

    fn seat_state(&mut self) -> &mut SeatState<Self> {
        &mut self.seat_state
    }

    fn focus_changed(&mut self, seat: &Seat<Self>, focused: Option<&WlSurface>) {
        tracing::debug!(seat_name = %seat.name(), new_focus = ?focused.map(|s| s.id()), "Keyboard focus changed (SeatHandler)");
        // Logik hier sollte sich auf Compositor-interne Reaktionen auf den Fokuswechsel konzentrieren.
        // Die eigentliche Benachrichtigung des Clients (wl_keyboard.enter/leave) erfolgt
        // durch KeyboardHandle::set_focus().
        // Hier könnte man z.B. das aktive Fenster in der Domäne aktualisieren.
        let domain_window_id_option = focused.and_then(|s| {
            self.windows.iter().find_map(|(domain_id, window_element)| {
                if window_element.wl_surface().as_ref() == Some(s) {
                    Some(domain_id.clone())
                } else {
                    None
                }
            })
        });

        let workspace_manager = self.workspace_manager_service.clone(); // Arc<TokioMutex<...>>
        let focus_id_clone = domain_window_id_option.clone();
        // Dieser Aufruf muss non-blocking sein oder in einem Task erfolgen,
        // da wir uns im calloop Event-Handler befinden.
        // Hier vereinfacht als direkter Aufruf, in Realität ggf. über Kanal oder spawn_blocking.
        // tokio::runtime::Handle::current().block_on(async move {
        //     if let Err(e) = workspace_manager.lock().await.notify_keyboard_focus_changed(focus_id_clone).await {
        //         tracing::error!("Failed to notify domain about focus change: {}", e);
        //     }
        // });
        // Besser: Ein dedizierter SystemLayerEvent wird gesendet, auf den ein Domänen-Adapter hört.
         if let Some(bridge) = crate::system::event_bridge::global_system_event_bridge_option() {
            bridge.publish(crate::system::event_bridge::SystemLayerEvent::FocusChanged {
                newly_focused_window_id: domain_window_id_option,
                seat_name: seat.name().to_string(),
            });
        }
    }

    fn cursor_image(&mut self, _seat: &Seat<Self>, image: CursorImageStatus) {
        tracing::trace!(status = ?image, "Request to set cursor image");
        let mut guard = self.current_cursor_status.lock().unwrap();
        *guard = image;
        // Der Renderer wird diesen Status im nächsten Frame verwenden, um den Cursor zu zeichnen.
        // Es ist nicht nötig, hier explizit neu zu zeichnen, da der Cursor typischerweise
        // als Overlay im Rendering-Loop gehandhabt wird.
    }
}
```

Diese Code-Skizzen illustrieren die Art der Implementierung und die Interaktion mit Smithay-Komponenten und Domänendiensten. Die tatsächliche Implementierung erfordert eine sorgfältige Handhabung von Lebenszeiten, Fehlerbedingungen und der asynchronen Natur vieler Operationen.

Die maximale Detailtiefe wird durch die fortlaufende Verfeinerung jeder Komponente, Methode und Datenstruktur erreicht, wobei stets auf die Vermeidung von Ambiguitäten und die Bereitstellung klarer, umsetzbarer Anweisungen geachtet wird.

#### Zusätzliche Details zu wichtigen Protokollen in der Systemschicht

- **`wlr-output-management-unstable-v1` (system::compositor::output_management_wlr)**:
    - `DesktopState` implementiert `smithay::wayland::output_manager::OutputManagementHandler`.
    - `create_configuration(serial)`: Erstellt `OutputConfigurationRequest` (speichert `serial` und Client). Sendet aktuellen Output-Zustand (Heads, Modes) an die neue Konfigurationsressource.
    - `apply(config_resource)`:
        1. Prüft `OutputConfigurationRequest.serial` gegen `DesktopState.global_output_serial`. Bei Mismatch: `cancelled` senden.
        2. Validiert die angeforderte Konfiguration in `OutputConfigurationRequest.pending_changes` (existieren Outputs/Modes? Ist Konfig plausibel?).
        3. Versucht, die Konfiguration auf die `Output`s in `DesktopState.space` anzuwenden (via `output.change_current_state(...)` oder direkter DRM-Interaktion).
        4. Bei Erfolg: `succeeded` senden, `DesktopState.global_output_serial` inkrementieren, neuen Zustand an alle Manager-Clients senden.
        5. Bei Fehler: `failed` senden, ggf. Rollback.
- **`wlr-foreign-toplevel-management-unstable-v1` (system::compositor::foreign_toplevel)**:
    - `DesktopState` implementiert `smithay::wayland::foreign_toplevel::ForeignToplevelHandler`.
    - Wenn neue XDG-Toplevels erstellt oder ihr Zustand (Titel, AppID, maximiert etc.) sich ändert, werden die entsprechenden `zwlr_foreign_toplevel_handle_v1`-Events an Clients gesendet.
    - Handhabt Anfragen von Clients wie `request_close`, `set_maximized` etc. und leitet diese an die XDG-Toplevel-Logik weiter.
- **PipeWire-Interaktion (`system::audio_management::client`)**:
    - Der `PipeWireClientService` startet einen dedizierten Thread für den PipeWire `MainLoop`.
    - Innerhalb dieses Threads lauscht ein `RegistryListener` auf neue Nodes (Sinks, Sources) und Streams.
        - Beim Erkennen eines neuen Audio-Nodes (z.B. `media.class == "Audio/Sink"`):
            - Proxy zum Node binden: `registry.bind::<pipewire::node::Node>(...)`.
            - Eigenschaften des Nodes abrufen (`node.params("Props")` für `channelVolumes`, `mute`; `node.info()` für Beschreibung etc.).
            - `AudioDevice`-Struktur erstellen und in `PipeWireClientService.devices` (RwLock-geschützte Map) speichern.
            - Einen `Listener` auf dem Node-Proxy für `param_changed` registrieren. Wenn `param_changed` für "Props" ausgelöst wird, `AudioDevice` aktualisieren und `AudioEvent::DeviceVolumeChanged` senden.
            - `AudioEvent::DeviceListChanged` senden.
        - Analog für Streams (`media.class == "Stream/Output/Audio"`).
    - Setter-Methoden (z.B. `set_device_volume`) in `PipeWireClientService`:
        - Diese Methoden sind `async` und werden vom Haupt-Tokio-Executor aufgerufen.
        - Sie senden eine Nachricht (z.B. `AudioCommand::SetVolume { id, vol, mute }`) über einen `tokio::sync::mpsc::channel` an den PipeWire-MainLoop-Thread.
        - Der PipeWire-Thread empfängt den Befehl, holt den Node-Proxy und ruft `node.set_param("Props", ...)` mit den neuen Werten auf. Die Antwort von PipeWire (Erfolg/Fehler der Operation) wird über den `param_changed`-Listener des Nodes empfangen und löst dann das entsprechende `AudioEvent` aus.

Diese detaillierte Vorgehensweise, die klare Trennung von Verantwortlichkeiten und die Nutzung der Stärken der jeweiligen Bibliotheken (Smithay, Zbus, PipeWire-rs) bilden das Rückgrat für eine robuste und wartbare Systemschicht.