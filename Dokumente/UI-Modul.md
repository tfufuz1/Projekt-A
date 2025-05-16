# NovaDE: Ultra-Granulare Definition Interner Schnittstellen

## Präambel: Grundlegende Prinzipien

Die Architektur von NovaDE basiert auf einer Reihe fundamentaler Kommunikations- und Strukturprinzipien, die darauf abzielen, ein robustes, modulares und wartbares System zu schaffen. Diese Prinzipien sind entscheidend für das Erreichen der übergeordneten Projektziele wie Performance, intuitive Bedienung, Modernität, Modularität, Anpassbarkeit, sichere KI-Integration und Stabilität.

- **Kernprinzipien der Kommunikation:**
    - Die Interaktion zwischen den verschiedenen Schichten des Systems – Kern (Core), Domäne (Domain), System und Benutzeroberfläche (UI) – erfolgt primär über klar definierte öffentliche Programmierschnittstellen (APIs). Diese APIs werden in der Regel als Rust-Traits spezifiziert und von Service-Strukturen innerhalb der jeweiligen Schicht implementiert. Dieser Ansatz fördert ein vertragsbasiertes Interaktionsmodell, das die Kopplung zwischen den Schichten minimiert und die Austauschbarkeit von Implementierungen erleichtert.
    - Ein integraler Bestandteil der Architektur ist ein robustes, ereignisgesteuertes System. Dieses System dient der Benachrichtigung über Zustandsänderungen und ermöglicht entkoppelte Interaktionen zwischen den Schichten. Komponenten können auf relevante Ereignisse reagieren, ohne direkte Abhängigkeiten zu den Ereigniserzeugern aufbauen zu müssen. Dies ist besonders für die UI-Schicht von Bedeutung, da es reaktive Updates auf Änderungen im Domänenzustand oder Systemstatus ohne Polling-Mechanismen ermöglicht.
    - Eine strikte Schichtenarchitektur wird durchgesetzt: Höhere Schichten dürfen typischerweise nur von unmittelbar tieferliegenden Schichten abhängen. Dieses hierarchische Abhängigkeitsmodell verbessert die Modularität, Testbarkeit und Wartbarkeit, indem es zirkuläre Abhängigkeiten und unkontrollierte Interaktionen verhindert.
    - **Fehlerbehandlung:** Eine konsistente Fehlerbehandlungsstrategie wird über alle Schichten hinweg angewendet. Die `thiserror`-Bibliothek wird für die Definition von spezifischen, benutzerdefinierten Fehlertypen genutzt. `core::errors::CoreError` dient als fundamentaler Basistyp für Fehler. Es ist zwingend erforderlich, dass Fehlerketten (abrufbar über die `source()`-Methode) bei der Weitergabe oder beim Wrappen von Fehlern erhalten bleiben, um eine umfassende Fehlerdiagnose zu gewährleisten.

Die Kombination aus strikter Schichtung, einem API-First-Design (mittels Rust-Traits) und einem ereignisgesteuerten Ansatz bildet ein leistungsfähiges Fundament für NovaDE. Das Ereignissystem ist insbesondere für die UI-Schicht von entscheidender Bedeutung, da es ihr ermöglicht, reaktiv auf zugrundeliegende Änderungen im Domänenzustand oder Systemstatus zu reagieren, ohne auf ineffiziente Polling-Mechanismen zurückgreifen zu müssen. Beispielsweise kann die UI dynamisch aktualisiert werden, wenn ein `ThemeChangedEvent` vom `ThemingEngine` ausgelöst wird, anstatt periodisch den aktuellen Theme-Status abfragen zu müssen. Diese Entkopplung reduziert die Systemlast und verbessert die Reaktionsfähigkeit der Benutzeroberfläche. Die nachfolgende ultra-granulare Definition wird daher nicht nur die synchronen API-Methoden detailliert beschreiben, sondern auch die präzise Struktur, den Payload und die Semantik jedes Ereignisses, das zwischen den Schichten ausgetauscht wird. Dies stellt sicher, dass sowohl API-Konsumenten als auch Ereignis-Abonnenten ein klares und eindeutiges Verständnis der Daten haben, die sie senden oder empfangen werden.

---

**Teil 1: Ultra-Granulare Schnittstellendefinition der UI-Schicht**

Dieser Teil adressiert die primäre Anforderung einer detaillierten Schnittstellendefinition für die UI-Schicht und konzentriert sich darauf, wie diese mit der Domänen- und der Systemschicht interagiert. Die UI-Schicht ist für die Präsentation von Informationen für den Benutzer und die Übersetzung von Benutzereingaben in Aktionen auf den unteren Schichten verantwortlich.

- **1.1. Interaktionsprinzipien der UI-Schicht mit unteren Schichten**
    
    - **UI als Konsument von Domänenlogik und Systemdiensten:**
        - Die UI-Schicht agiert als primärer Interaktionspunkt mit dem Benutzer. Sie übersetzt Benutzereingaben (z.B. Klicks, Tastatureingaben, Gesten) in spezifische, wohldefinierte Aufrufe von Methoden, die von Diensten der Domänenschicht (für Geschäftslogik und Anwendungszustandsmanipulation) oder der Systemschicht (für Betriebssystem- oder Hardwareinteraktionen) bereitgestellt werden.
        - Ein entscheidender Aspekt ist, dass die UI-Schicht Ereignisse abonniert, die sowohl von der Domänen- als auch von der Systemschicht stammen. Diese Ereignisse signalisieren Zustandsänderungen oder das Auftreten bemerkenswerter Systemaktivitäten. Dies ermöglicht es der UI, ihre Darstellung dynamisch und reaktiv zu aktualisieren und so den aktuellen Zustand der Anwendung und des zugrundeliegenden Systems widerzuspiegeln.
        - Obwohl die interne Implementierung von UI-Komponenten stark auf einem Toolkit wie GTK4 basieren wird, werden die in diesem Dokument definierten Schnittstellen (zwischen UI und Domäne/System) in Form von reinen Rust-Typen, Traits und Ereignis-Strukturen spezifiziert. Diese Abstraktion stellt sicher, dass die Kernlogik von NovaDE nicht intrinsisch an ein spezifisches UI-Toolkit gebunden ist. Beispielsweise wird ein `core::types::Color` in API-Signaturen verwendet, nicht ein Toolkit-spezifischer Typ wie `gdk4::RGBA`.
    - **Zustandsverwaltung in der UI:**
        - Die UI-Schicht ist für die Verwaltung ihres eigenen Präsentationszustands verantwortlich (oft als ViewModels oder UI-spezifischer Zustand bezeichnet). Dieser Zustand wird von dem Zustand abgeleitet, der von der Domänen- und Systemschicht verwaltet wird, und muss mit diesem synchron gehalten werden.
        - Eine robuste und effiziente UI-Zustandsverwaltung ist für eine reaktionsschnelle und konsistente Benutzererfahrung von größter Bedeutung. Während die UI-Schicht ihre visuelle Darstellung und temporäre Interaktionszustände verwaltet, befindet sich die autoritative "Wahrheitsquelle" (Source of Truth) für geschäftskritische Daten innerhalb der Domänenschicht und für den Systemstatus innerhalb der Systemschicht. Die UI muss so aufgebaut sein, dass sie effizient auf Änderungen reagiert, die von diesen unteren Schichten signalisiert werden.
        - Die Benutzeranfrage liefert ein Beispiel: `ui::theming_gtk` konsumiert `AppliedThemeState` von `ThemingEngine`. Dieses Interaktionsmuster impliziert einen reaktiven Aktualisierungsmechanismus. Verschiedene Ansätze zur Zustandsverwaltung in `gtk-rs`-Anwendungen, einschließlich der Verwendung von `Rc<RefCell<T>>` für Single-Threaded Shared Mutability oder Mustern wie TEA (The Elm Architecture) oder Relm4, werden in der Forschung diskutiert.1 Für die geschichtete Architektur von NovaDE ist der am besten geeignete Ansatz, dass die UI-Schicht Domänen- und Systemereignisse abonniert. Nach dem Empfang eines Ereignisses aktualisiert die UI ihren lokalen Zustand (der GObject-Eigenschaften, Relm4-Modellfelder oder andere UI-Toolkit-spezifische Zustandscontainer sein können). Dieser ereignisgesteuerte Ansatz vermeidet eine enge Kopplung, bei der die Domänen- oder Systemschicht direkte Kenntnis von GTK-GObjects oder anderen UI-internen Details haben müsste.
        - Ereignisbehandlungsmechanismen innerhalb von GTK4 und verschiedene Zustandsverwaltungsmuster 1 sind für die interne Implementierung der UI-Schicht relevant. Dieses Schnittstellendefinitionsdokument konzentriert sich jedoch auf die Ereignisse, die die UI von unteren Schichten konsumiert, und die Daten-Payloads, die diese Ereignisse transportieren.
    - **Strategie zur Fehlerpräsentation:**
        - Die UI-Schicht trägt die Verantwortung für die Übersetzung von Fehlern, die von der Domänen- und Systemschicht empfangen werden, in benutzerfreundliche und kontextuell angemessene Benachrichtigungen, Dialoge oder visuelle Hinweise. Rohe technische Fehlerdetails sollten dem Endbenutzer nicht direkt präsentiert werden.
        - Intern könnte die UI `gtk::AlertDialog` 6 oder benutzerdefinierte Benachrichtigungs-Widgets (potenziell unter Verwendung von `gtk4-layer-shell` 9 für nicht-modale Popups) zur Darstellung dieser übersetzten Fehler verwenden. Diese Schnittstellendefinition wird spezifizieren, wie Fehlertypen (z.B. `ThemingError`, `CompositorError`) von unteren Schichten erwartet werden, von der UI behandelt und potenziell auf standardisierte benutzerorientierte Fehlerkategorien oder -nachrichten abgebildet zu werden.
        - Eine konsistente und verständliche Fehlerpräsentationsstrategie ist grundlegend für eine gute Benutzererfahrung. Die UI muss als Filter und Übersetzer fungieren und Benutzer vor komplexen oder kryptischen internen Fehlerdetails schützen. Die Benutzeranfrage spezifiziert, dass alle Schichten `thiserror`-basierte Fehler-Enums verwenden. Die UI-Schicht wird diese spezifischen Fehlertypen empfangen. Wenn beispielsweise der `WorkspaceManager` einen `WorkspaceManagerError::WorkspaceNotFound` zurückgibt, sollte die UI-Schicht diese rohe Enum-Variante nicht anzeigen. Stattdessen sollte sie dies in eine für Menschen lesbare Nachricht wie "Der ausgewählte Arbeitsbereich konnte nicht gefunden werden." übersetzen oder gegebenenfalls einen Dialog mit Wiederherstellungsoptionen anbieten. Diese Übersetzungs- und Präsentationslogik ist eine Kernaufgabe der UI-Schicht. Die Schnittstellendefinition sollte dies leiten, indem sie Fehlerkategorien vorschlägt, die die UI von jedem Dienst zu behandeln bereit sein sollte.
- 1.2. Schnittstelle der UI-Schicht mit der Domänenschicht
    
    Dieser Abschnitt definiert ultra-granular die Interaktionen der UI-Schicht mit jedem von der Domänenschicht bereitgestellten Dienst.
    
    - **1.2.1. `ThemingEngine`-API-Nutzung durch die UI:**
        
        - **Zweck:** Ermöglicht der UI, das aktuelle visuelle Thema der NovaDE-Umgebung abzufragen und zu ändern.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `update_configuration`: `domain::theming::ThemeConfigurationUpdate { theme_name: Option<String>, variant_name: Option<String>, custom_color_palette: Option<HashMap<String, core::types::Color>> }`
            - Ausgabe von `get_current_theme_state`: `domain::theming::AppliedThemeState { theme_name: String, variant_name: String, primary_accent_color: core::types::Color, secondary_accent_color: core::types::Color, default_background_color: core::types::Color, default_text_color: core::types::Color, default_font_family: String, default_font_size: u16, border_radius: u8, shadow_intensity: f32 }`
            - Ausgabe von `get_available_themes`: `Vec<domain::theming::ThemeIdentifier { theme_id: String, display_name: String, available_variants: Vec<String>, preview_icon_path: Option<PathBuf>, author: Option<String>, description: Option<String> }>`
            - Ausgabe von `get_current_configuration`: `domain::theming::CurrentThemeConfiguration { theme_name: String, variant_name: String, custom_palette: HashMap<String, core::types::Color> }`
        - **Von der UI aufgerufene API-Methoden (aus `domain::ThemingEngine`-Trait):**
            - `fn get_current_theme_state(&self) -> Result<domain::theming::AppliedThemeState, domain::ThemingError>;`
            - `fn get_available_themes(&self) -> Result<Vec<domain::theming::ThemeIdentifier>, domain::ThemingError>;`
            - `fn get_current_configuration(&self) -> Result<domain::theming::CurrentThemeConfiguration, domain::ThemingError>;`
            - `fn update_configuration(&self, config_update: domain::theming::ThemeConfigurationUpdate) -> Result<(), domain::ThemingError>;`
            - `fn reload_themes_and_tokens(&self) -> Result<(), domain::ThemingError>;`
            - `// Die Ereignisabonnierung erfolgt über einen globalen EventBus oder eine dedizierte Abonnementmethode, falls eine feingranulare Steuerung erforderlich ist.`
        - **Von der UI konsumierte Event-Payloads:**
            - `domain::theming::ThemeChangedEvent { new_state: domain::theming::AppliedThemeState }`
        - **Fehlerbehandlung durch die UI:** Die UI muss `domain::ThemingError`-Varianten (z.B. `ThemeNotFound`, `InvalidConfiguration`, `AssetLoadFailure`) auf angemessene benutzerorientierte Nachrichten oder Wiederherstellungsaktionen abbilden.
    - **1.2.2. `WorkspaceManager`-API-Nutzung durch die UI:**
        
        - **Zweck:** Ermöglicht der UI die Verwaltung virtueller Arbeitsbereiche, einschließlich deren Erstellung, Löschung, Aktivierung und Zuweisung von Fenstern.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `create_workspace`: `domain::workspaces::CreateWorkspaceParams { name: Option<String>, layout_hint: Option<domain::workspaces::WorkspaceLayoutType>, icon_name: Option<String> }`
            - Eingabe für `assign_window_to_workspace`: `window_id: core::types::WindowId`, `workspace_id: core::types::WorkspaceId`
            - Eingabe für `rename_workspace`: `workspace_id: core::types::WorkspaceId`, `new_name: String`
            - Ausgabe von `get_active_workspace`, `get_workspace_by_id`, `list_workspaces`: `domain::workspaces::WorkspaceDescriptor { id: core::types::WorkspaceId, name: String, display_name: String, icon_name: Option<String>, layout: domain::workspaces::WorkspaceLayoutType, window_ids: Vec<core::types::WindowId>, is_active: bool, creation_timestamp: core::types::DateTime<Utc> }`
            - Enum `domain::workspaces::WorkspaceLayoutType { TilingHorizontal, TilingVertical, Floating, Maximized }`
        - **Von der UI aufgerufene API-Methoden (aus `domain::WorkspaceManager`-Trait):**
            - `fn create_workspace(&self, params: domain::workspaces::CreateWorkspaceParams) -> Result<core::types::WorkspaceId, domain::WorkspaceManagerError>;`
            - `fn delete_workspace(&self, id: core::types::WorkspaceId) -> Result<(), domain::WorkspaceManagerError>;`
            - `fn set_active_workspace(&self, id: core::types::WorkspaceId) -> Result<(), domain::WorkspaceManagerError>;`
            - `fn get_active_workspace(&self) -> Result<Option<domain::workspaces::WorkspaceDescriptor>, domain::WorkspaceManagerError>;`
            - `fn get_workspace_by_id(&self, id: core::types::WorkspaceId) -> Result<Option<domain::workspaces::WorkspaceDescriptor>, domain::WorkspaceManagerError>;`
            - `fn list_workspaces(&self) -> Result<Vec<domain::workspaces::WorkspaceDescriptor>, domain::WorkspaceManagerError>;`
            - `fn assign_window_to_workspace(&self, window_id: core::types::WindowId, workspace_id: core::types::WorkspaceId) -> Result<(), domain::WorkspaceManagerError>;`
            - `fn remove_window_from_workspace(&self, window_id: core::types::WindowId, workspace_id: core::types::WorkspaceId) -> Result<(), domain::WorkspaceManagerError>;`
            - `fn get_window_assignment(&self, window_id: core::types::WindowId) -> Result<Option<core::types::WorkspaceId>, domain::WorkspaceManagerError>;`
            - `fn save_workspace_configuration(&self) -> Result<(), domain::WorkspaceManagerError>;`
            - `fn rename_workspace(&self, workspace_id: core::types::WorkspaceId, new_name: String) -> Result<(), domain::WorkspaceManagerError>;`
            - `fn set_workspace_layout(&self, workspace_id: core::types::WorkspaceId, layout: domain::workspaces::WorkspaceLayoutType) -> Result<(), domain::WorkspaceManagerError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `domain::workspaces::WorkspaceCreatedEvent { descriptor: domain::workspaces::WorkspaceDescriptor }`
            - `domain::workspaces::WorkspaceDeletedEvent { workspace_id: core::types::WorkspaceId, previous_name: String }`
            - `domain::workspaces::ActiveWorkspaceChangedEvent { old_workspace_id: Option<core::types::WorkspaceId>, new_workspace_id: core::types::WorkspaceId }`
            - `domain::workspaces::WorkspaceLayoutChangedEvent { workspace_id: core::types::WorkspaceId, new_layout: domain::workspaces::WorkspaceLayoutType }`
            - `domain::workspaces::WindowAssignedToWorkspaceEvent { window_id: core::types::WindowId, new_workspace_id: core::types::WorkspaceId, old_workspace_id: Option<core::types::WorkspaceId> }`
            - `domain::workspaces::WindowRemovedFromWorkspaceEvent { window_id: core::types::WindowId, workspace_id: core::types::WorkspaceId }`
            - `domain::workspaces::WorkspaceRenamedEvent { workspace_id: core::types::WorkspaceId, new_name: String, old_name: String }`
            - `domain::workspaces::WorkspaceOrderChangedEvent { ordered_workspace_ids: Vec<core::types::WorkspaceId> }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `domain::WorkspaceManagerError`-Varianten (z.B. `WorkspaceNotFound`, `WindowNotManaged`, `NameConflict`) auf Benutzerfeedback.
    - **1.2.3. `AIInteractionLogicService`-API-Nutzung durch die UI:**
        
        - **Zweck:** Ermöglicht der UI, KI-gestützte Interaktionen zu initiieren, Kontextinformationen abzurufen und Benutzereinwilligungen zu verwalten.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `initiate_interaction`: `domain::ai::InteractionRequest { prompt_text: String, context_type: domain::ai::AIContextType, attachments: Option<Vec<domain::ai::AIAttachment>>, preferred_model_id: Option<String> }`
            - Enum `domain::ai::AIContextType { GeneralText, CodeGeneration, ImageAnalysis, /*... */ }`
            - Struct `domain::ai::AIAttachment { mime_type: String, data_uri: String /* or PathBuf if local */, name: Option<String> }`
            - Ausgabe von `initiate_interaction`: `domain::ai::InteractionResponseHandle { interaction_id: core::types::Uuid }` (Weitere Ergebnisse/Streams könnten über Events oder eine Streaming-API kommen)
            - Ausgabe von `get_interaction_context`: `domain::ai::InteractionContext { interaction_id: core::types::Uuid, status: domain::ai::AIInteractionStatus, history: Vec<domain::ai::AIMessage>, available_actions: Vec<domain::ai::AIActionSuggestion> }`
            - Enum `domain::ai::AIInteractionStatus { PendingInput, Processing, AwaitingConsent, Completed, Error }`
            - Struct `domain::ai::AIMessage { role: domain::ai::AIRole, content: String, timestamp: core::types::DateTime<Utc> }`
            - Enum `domain::ai::AIRole { User, Assistant, System }`
            - Struct `domain::ai::AIActionSuggestion { action_id: String, display_text: String, action_type: domain::ai::AIActionType }`
            - Enum `domain::ai::AIActionType { ExecuteCommand, InsertText, OpenLink }`
            - Eingabe für `provide_consent`: `interaction_id: core::types::Uuid, model_id: String, consent_granted: bool`
            - Ausgabe von `get_consent_for_model`: `domain::ai::AIConsentStatus { model_id: String, granted: bool, last_updated: core::types::DateTime<Utc> }`
            - Struct `domain::ai::AIModelProfile { model_id: String, display_name: String, provider: String, capabilities: Vec<domain::ai::AIContextType>, requires_consent: bool }`
        - **Von der UI aufgerufene API-Methoden (aus `domain::AIInteractionLogicService`-Trait):**
            - `async fn initiate_interaction(&self, request: domain::ai::InteractionRequest) -> Result<domain::ai::InteractionResponseHandle, domain::AIInteractionError>;`
            - `async fn get_interaction_context(&self, interaction_id: core::types::Uuid) -> Result<domain::ai::InteractionContext, domain::AIInteractionError>;`
            - `async fn provide_consent(&self, interaction_id: core::types::Uuid, model_id: String, consent_granted: bool) -> Result<(), domain::AIInteractionError>;`
            - `async fn get_consent_for_model(&self, model_id: String) -> Result<domain::ai::AIConsentStatus, domain::AIInteractionError>;`
            - `async fn store_consent(&self, model_id: String, granted: bool) -> Result<(), domain::AIInteractionError>;`
            - `async fn load_model_profiles(&self) -> Result<Vec<domain::ai::AIModelProfile>, domain::AIInteractionError>;`
            - `async fn send_message_to_interaction(&self, interaction_id: core::types::Uuid, message: domain::ai::AIMessage) -> Result<(), domain::AIInteractionError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `domain::ai::AIInteractionInitiatedEvent { interaction_id: core::types::Uuid, initial_prompt: String }`
            - `domain::ai::AIConsentUpdatedEvent { model_id: String, new_status: domain::ai::AIConsentStatus }`
            - `domain::ai::AIInteractionStatusChangedEvent { interaction_id: core::types::Uuid, new_status: domain::ai::AIInteractionStatus, details: Option<String> }`
            - `domain::ai::NewAIMessageEvent { interaction_id: core::types::Uuid, message: domain::ai::AIMessage }`
            - `domain::ai::AIInteractionErrorEvent { interaction_id: Option<core::types::Uuid>, error_message: String, is_fatal: bool }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `domain::AIInteractionError`-Varianten (z.B. `ModelUnavailable`, `ConsentRequired`, `NetworkError`) auf Benutzerfeedback.
    - **1.2.4. `NotificationService`-API-Nutzung durch die UI (primär für das Posten von UI-generierten Benachrichtigungen):**
        
        - **Zweck:** Ermöglicht der UI, anwendungsspezifische Benachrichtigungen zu erstellen und anzuzeigen, sowie bestehende Benachrichtigungen zu verwalten.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `post_notification`: `domain::notifications::NotificationData { app_name: String, app_icon: Option<String>, summary: String, body: Option<String>, actions: Vec<domain::notifications::NotificationAction>, urgency: domain::notifications::NotificationUrgency, timeout_ms: Option<u32>, category: Option<String>, transient: bool }`
            - Struct `domain::notifications::NotificationAction { id: String, display_text: String }`
            - Enum `domain::notifications::NotificationUrgency { Low, Normal, Critical }`
            - Ausgabe von `post_notification`: `notification_id: core::types::Uuid`
            - Ausgabe von `get_notification`, `list_notifications`: `domain::notifications::Notification { id: core::types::Uuid, data: domain::notifications::NotificationData, timestamp: core::types::DateTime<Utc>, is_read: bool, is_dismissed: bool }`
        - **Von der UI aufgerufene API-Methoden (aus `domain::NotificationService`-Trait):**
            - `fn post_notification(&self, notification_data: domain::notifications::NotificationData) -> Result<core::types::Uuid, domain::NotificationError>;`
            - `fn get_notification(&self, id: core::types::Uuid) -> Result<Option<domain::notifications::Notification>, domain::NotificationError>;`
            - `fn list_notifications(&self, filter: Option<domain::notifications::NotificationFilter>) -> Result<Vec<domain::notifications::Notification>, domain::NotificationError>;` // Filter könnte `unread_only`, `app_name` etc. sein
            - `fn mark_as_read(&self, id: core::types::Uuid) -> Result<(), domain::NotificationError>;`
            - `fn dismiss_notification(&self, id: core::types::Uuid) -> Result<(), domain::NotificationError>;`
            - `fn invoke_action_on_notification(&self, notification_id: core::types::Uuid, action_id: String) -> Result<(), domain::NotificationError>;`
            - `fn set_do_not_disturb(&self, enabled: bool) -> Result<(), domain::NotificationError>;`
            - `fn get_do_not_disturb_status(&self) -> Result<bool, domain::NotificationError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `domain::notifications::NotificationPostedEvent { notification: domain::notifications::Notification }`
            - `domain::notifications::NotificationDismissedEvent { notification_id: core::types::Uuid, reason: domain::notifications::DismissReason }`
            - `domain::notifications::NotificationActionInvokedEvent { notification_id: core::types::Uuid, action_id: String }`
            - `domain::notifications::DoNotDisturbChangedEvent { is_enabled: bool }`
            - Enum `domain::notifications::DismissReason { User, Timeout, Programmatic }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `domain::NotificationError`-Varianten (z.B. `NotificationNotFound`, `ActionNotFound`) auf Benutzerfeedback.
    - **1.2.5. `GlobalSettingsService`-API-Nutzung durch die UI:**
        
        - **Zweck:** Ermöglicht der UI, globale Anwendungseinstellungen zu lesen, zu ändern und auf Standardwerte zurückzusetzen.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `update_setting`: `key: String`, `value: glib::Variant`
            - Ausgabe von `get_setting`: `Result<Option<glib::Variant>, domain::GlobalSettingsError>`
            - Ausgabe von `get_current_settings`: `Result<HashMap<String, glib::Variant>, domain::GlobalSettingsError>`
            - Struct `domain::settings::SettingDescriptor { key: String, display_name: String, description: Option<String>, value_type: glib::VariantType, current_value: glib::Variant, default_value: glib::Variant, possible_values: Option<Vec<glib::Variant>>, range: Option<(glib::Variant, glib::Variant)>, category: String }`
        - **Von der UI aufgerufene API-Methoden (aus `domain::GlobalSettingsService`-Trait):**
            - `fn load_settings(&self) -> Result<(), domain::GlobalSettingsError>;`
            - `fn save_settings(&self) -> Result<(), domain::GlobalSettingsError>;`
            - `fn get_current_settings(&self) -> Result<HashMap<String, glib::Variant>, domain::GlobalSettingsError>;`
            - `fn get_setting_descriptors(&self) -> Result<Vec<domain::settings::SettingDescriptor>, domain::GlobalSettingsError>;`
            - `fn update_setting(&self, key: String, value: glib::Variant) -> Result<(), domain::GlobalSettingsError>;`
            - `fn get_setting(&self, key: String) -> Result<Option<glib::Variant>, domain::GlobalSettingsError>;`
            - `fn reset_setting_to_default(&self, key: String) -> Result<(), domain::GlobalSettingsError>;`
            - `fn reset_all_settings_to_defaults(&self) -> Result<(), domain::GlobalSettingsError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `domain::settings::SettingChangedEvent { key: String, new_value: glib::Variant, old_value: glib::Variant }`
            - `domain::settings::SettingsLoadedEvent { settings: HashMap<String, glib::Variant> }`
            - `domain::settings::SettingsSavedEvent {}`
        - **Fehlerbehandlung durch die UI:** Abbildung von `domain::GlobalSettingsError`-Varianten (z.B. `SettingNotFound`, `InvalidValueType`, `PersistenceError`) auf Benutzerfeedback. Die Verwendung von `glib::Variant` für Einstellwerte 12 ermöglicht eine flexible Handhabung verschiedener Datentypen für Einstellungen.
    - **Schlüsseltabellen für Teil 1.2:**
        
        Für jede Methode der Domänendienst-API:
        
        1. **Methodensignatur-Tabelle:**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp**|**Kurzbeschreibung**|
|`ThemingEngine::get_current_theme_state`|`&self`|`Result<domain::theming::AppliedThemeState, domain::ThemingError>`|Ruft den aktuell angewendeten Theme-Zustand ab.|
|`WorkspaceManager::create_workspace`|`&self, params: domain::workspaces::CreateWorkspaceParams`|`Result<core::types::WorkspaceId, domain::WorkspaceManagerError>`|Erstellt einen neuen Arbeitsbereich.|
|...|...|...|...|

```
    2.  **Parameterdetail-Tabelle (Beispiel für `ThemingEngine::update_configuration`):**
```

|   |   |   |   |
|---|---|---|---|
|**Parametername**|**Typ**|**Erforderlich**|**Beschreibung**|
|`config_update`|`domain::theming::ThemeConfigurationUpdate`|Ja|Ein Objekt, das die gewünschten Änderungen an der Theme-Konfiguration enthält.|

```
    3.  **Rückgabewertdetail-Tabelle (Beispiel für `ThemingEngine::get_current_theme_state`):**
```

|   |   |   |
|---|---|---|
|**Erfolgsfall (Ok)**|**Fehlerfall (Err)**|**Beschreibung**|
|`domain::theming::AppliedThemeState`|`domain::ThemingError`|Enthält Details zum aktuell angewendeten Theme oder einen Fehler bei der Abfrage.|

```
    Für jedes von der UI konsumierte Domänenereignis:
    4.  **Ereignis-Payload-Detail-Tabelle (Beispiel für `ThemeChangedEvent`):**
```

|   |   |   |
|---|---|---|
|**Feldname**|**Typ**|**Beschreibung**|
|`new_state`|`domain::theming::AppliedThemeState`|Der neue Zustand des Themes nach der Änderung.|

```
    Diese Tabellen sind unerlässlich, um Entwicklern eine eindeutige und leicht zugängliche Referenz zu bieten. Sie erzwingen Konsistenz bei der Typverwendung und Methodenaufrufen und reduzieren so Integrationsfehler zwischen der UI- und der Domänenschicht erheblich. Beispielsweise beschreibt eine detaillierte Tabelle für `WorkspaceDescriptor` klar alle Informationen, die die UI über einen Arbeitsbereich erwarten kann, was die Entwicklung von UI-Komponenten wie Arbeitsbereichswechslern erleichtert.
```

- 1.3. Schnittstelle der UI-Schicht mit der Systemschicht
    
    Dieser Abschnitt definiert ultra-granular die Interaktionen der UI-Schicht mit Diensten, die von der Systemschicht bereitgestellt werden. Die Systemschicht abstrahiert Details des zugrundeliegenden Betriebssystems, der Hardware und des Fenstersystems (Wayland/X11).
    
    - **1.3.1. Compositor-Interaktion (`system::compositor_iface`) durch die UI:**
        
        - **Zweck:** Ermöglicht der UI, Fensterinformationen abzufragen, Fensterzustände (Fokus, Geometrie) zu verwalten und Aktualisierungen über den Lebenszyklus und die Eigenschaften von Fenstern zu erhalten. Diese Schnittstelle abstrahiert die zugrundeliegenden Wayland-Compositor-Interaktionen. Die UI interagiert nicht direkt mit Wayland-Protokollen wie `wlr-foreign-toplevel-management` 17 oder `ext-foreign-toplevel-list-v1` 17; stattdessen konsumiert sie die von der Systemschicht bereitgestellten Abstraktionen.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `request_focus`: `window_id: core::types::WindowId`
            - Eingabe für `configure_window`: `window_id: core::types::WindowId`, `config: system::compositor::WindowConfigurationRequest { new_position: Option<core::types::PointInt>, new_size: Option<core::types::SizeInt>, new_state: Option<system::compositor::CompositorWindowState>, new_workspace_id: Option<core::types::WorkspaceId> }`
            - Enum `system::compositor::CompositorWindowState { Maximized, Minimized, Fullscreen, Normal, TiledLeft, TiledRight }`
            - Ausgabe von `get_window_info`, `list_managed_windows`: `system::compositor::WindowInfo { id: core::types::WindowId, title: String, app_id: core::types::AppId, class: Option<String>, geometry: core::types::RectInt, is_focused: bool, is_maximized: bool, is_minimized: bool, is_fullscreen: bool, workspace_id: Option<core::types::WorkspaceId>, can_resize: bool, can_maximize: bool, can_minimize: bool, can_close: bool, parent_window_id: Option<core::types::WindowId>, transient_for: Option<core::types::WindowId> }`
            - Ausgabe von `get_output_info`, `list_outputs`: `system::compositor::OutputInfo { id: String, name: String, description: String, geometry: core::types::RectInt, scale_factor: f32, refresh_rate_mhz: u32, current_resolution: core::types::SizeInt, available_resolutions: Vec<core::types::SizeInt>, is_primary: bool, is_enabled: bool }`
        - **Von der UI aufgerufene API-Methoden (aus `system::CompositorInterface`-Trait):**
            - `fn request_focus(&self, window_id: core::types::WindowId) -> Result<(), system::CompositorError>;`
            - `fn configure_window(&self, window_id: core::types::WindowId, config: system::compositor::WindowConfigurationRequest) -> Result<(), system::CompositorError>;`
            - `fn get_window_info(&self, window_id: core::types::WindowId) -> Result<Option<system::compositor::WindowInfo>, system::CompositorError>;`
            - `fn list_managed_windows(&self) -> Result<Vec<system::compositor::WindowInfo>, system::CompositorError>;`
            - `fn close_window(&self, window_id: core::types::WindowId) -> Result<(), system::CompositorError>;`
            - `fn list_outputs(&self) -> Result<Vec<system::compositor::OutputInfo>, system::CompositorError>;`
            - `fn get_output_info(&self, output_id: String) -> Result<Option<system::compositor::OutputInfo>, system::CompositorError>;`
            - `fn get_cursor_position(&self) -> Result<core::types::PointInt, system::CompositorError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `system::compositor::WindowCreatedEvent { info: system::compositor::WindowInfo }`
            - `system::compositor::WindowClosedEvent { window_id: core::types::WindowId }`
            - `system::compositor::WindowFocusedEvent { window_id: core::types::WindowId }`
            - `system::compositor::WindowUnfocusedEvent { window_id: core::types::WindowId }`
            - `system::compositor::WindowPropertiesChangedEvent { window_id: core::types::WindowId, changed_title: Option<String>, changed_app_id: Option<core::types::AppId>, changed_geometry: Option<core::types::RectInt>, changed_state: Option<system::compositor::CompositorWindowState> }`
            - `system::compositor::OutputAddedEvent { info: system::compositor::OutputInfo }`
            - `system::compositor::OutputRemovedEvent { output_id: String }`
            - `system::compositor::OutputConfigurationChangedEvent { info: system::compositor::OutputInfo }`
            - `system::compositor::CursorPositionChangedEvent { position: core::types::PointInt }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `system::CompositorError`-Varianten (z.B. `WindowNotFound`, `WaylandProtocolError`, `InvalidConfiguration`) auf Benutzerfeedback.
    - **1.3.2. Eingabeverarbeitung (`system::input_iface`) durch die UI:**
        
        - **Zweck:** Liefert der UI verarbeitete Eingabeereignisse (Tastatur, Maus, Touch, Gesten) und ermöglicht es der UI, den Fokus zu steuern.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `request_keyboard_focus`: `window_id: core::types::WindowId` (kann auch über `compositor_iface` laufen, hier zur Verdeutlichung der Input-Intention)
            - Event-Payloads für `KeyboardEvent`, `PointerEvent`, `TouchEvent`, `GestureEvent` (detaillierte Definitionen erforderlich, z.B. `KeyEvent { key_code: u32, modifiers: KeyModifiers, state: KeyState, utf8: Option<String> }`).
        - **Von der UI aufgerufene API-Methoden (aus `system::InputInterface`-Trait):**
            - `fn request_keyboard_focus(&self, window_id: core::types::WindowId) -> Result<(), system::InputError>;` (kann Duplikat zu Compositor-API sein, aber semantisch hier relevant)
            - `fn set_cursor_theme(&self, theme_name: String, size: u32) -> Result<(), system::InputError>;`
            - `fn get_current_keyboard_layout(&self) -> Result<String, system::InputError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `system::input::KeyboardEvent { window_id: Option<core::types::WindowId>, /*... event details... */ }`
            - `system::input::PointerButtonEvent { window_id: Option<core::types::WindowId>, /*... event details... */ }`
            - `system::input::PointerMotionEvent { window_id: Option<core::types::WindowId>, /*... event details... */ }`
            - `system::input::PointerAxisEvent { window_id: Option<core::types::WindowId>, /*... event details... */ }`
            - `system::input::TouchEvent { window_id: Option<core::types::WindowId>, /*... event details... */ }`
            - `system::input::GestureEvent { window_id: Option<core::types::WindowId>, gesture_type: system::input::GestureType, /*... event details... */ }`
            - `system::input::FocusChangedEvent { new_focus_window_id: Option<core::types::WindowId>, old_focus_window_id: Option<core::types::WindowId> }` (kann auch vom Compositor kommen)
            - `system::input::KeyboardLayoutChangedEvent { new_layout_name: String }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `system::InputError`-Varianten.
    - **1.3.3. D-Bus Client-Schnittstellen (`system::dbus::*`) Nutzung durch die UI:**
        
        - **Allgemein:** Die UI interagiert mit D-Bus-Diensten über typsichere Rust-Wrapper, die von der Systemschicht bereitgestellt werden. Die UI konsumiert Ereignisse und ruft Methoden dieser Wrapper auf. Die Komplexität der D-Bus-Kommunikation (z.B. mit `zbus` 19 oder `gio::DBusConnection` 22) wird von der Systemschicht gekapselt.
        - **`system::dbus::upower_client_iface`:**
            - Events: `system::dbus::UPowerDeviceChangedEvent { device_id: String, percentage: f64, state: UPowerDeviceState, time_to_empty_s: Option<u64>, time_to_full_s: Option<u64> }`
        - **`system::dbus::logind_client_iface`:**
            - Methoden: `fn lock_session(&self) -> Result<(), system::DBusError>;`
            - Events: `system::dbus::LogindSessionEvent { event_type: LogindEventType /* Lock, Unlock, Sleep, Wakeup */ }`
        - **`system::dbus::networkmanager_client_iface`:**
            - Methoden: `fn list_access_points(&self, device_path: String) -> Result<Vec<system::dbus::AccessPointInfo>, system::DBusError>;`, `fn connect_to_access_point(&self, device_path: String, ap_bssid: String, password: Option<String>) -> Result<(), system::DBusError>;`
            - Datenstrukturen: `AccessPointInfo { bssid: String, ssid: String, strength: u8, security_flags: u32, is_active: bool }` (basierend auf NetworkManager D-Bus API 23)
            - Events: `system::dbus::NetworkStateChangedEvent { connectivity: NetworkConnectivityState, primary_connection_type: Option<String> }`, `system::dbus::AccessPointsChangedEvent { device_path: String }`
        - **`system::dbus::secrets_client_iface`:**
            - Methoden: `async fn prompt_for_secret(&self, prompt_message: String) -> Result<Option<String>, system::DBusError>;` (interagiert mit UI für Prompts)
    - **1.3.4. Output-Management (`system::outputs_iface`) durch die UI:**
        
        - **Zweck:** Ermöglicht der UI, Informationen über angeschlossene Displays abzurufen und deren Konfiguration (Auflösung, Skalierung, Position) zu ändern.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `configure_output`: `output_id: String`, `config: system::outputs::OutputConfigurationRequest { resolution: Option<core::types::SizeInt>, scale: Option<f32>, position: Option<core::types::PointInt>, enabled: Option<bool>, refresh_rate_mhz: Option<u32> }`
            - Ausgabe von `list_outputs`, `get_output_details`: `system::outputs::OutputDetails` (ähnlich `system::compositor::OutputInfo`, aber ggf. mit mehr Konfigurationsdetails wie verfügbare Modi, aktuelle Skalierung, etc.)
        - **Von der UI aufgerufene API-Methoden (aus `system::OutputInterface`-Trait):**
            - `fn list_outputs(&self) -> Result<Vec<system::outputs::OutputDetails>, system::OutputError>;`
            - `fn get_output_details(&self, output_id: String) -> Result<Option<system::outputs::OutputDetails>, system::OutputError>;`
            - `fn configure_output(&self, output_id: String, config: system::outputs::OutputConfigurationRequest) -> Result<(), system::OutputError>;`
            - `fn set_primary_output(&self, output_id: String) -> Result<(), system::OutputError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `system::outputs::OutputConfigurationChangedEvent { output_id: String, new_details: system::outputs::OutputDetails }` (kann auch vom Compositor kommen, hier ggf. spezifischer für Output-Management-Aktionen)
            - `system::outputs::OutputConnectedEvent { details: system::outputs::OutputDetails }`
            - `system::outputs::OutputDisconnectedEvent { output_id: String }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `system::OutputError`-Varianten.
    - **1.3.5. Audio-Management (`system::audio_iface`) durch die UI:**
        
        - **Zweck:** Ermöglicht der UI, Audio-Geräte und -Streams zu verwalten, Lautstärken zu regeln und Audio-Ereignisse zu empfangen. Abstrahiert PipeWire-Interaktionen.26
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `set_volume`: `device_id: String`, `stream_id: Option<String>`, `volume_percent: f32`, `is_muted: Option<bool>`
            - Ausgabe von `list_audio_devices`, `get_audio_device_details`: `system::audio::AudioDeviceDetails { id: String, name: String, device_type: AudioDeviceType /* Input, Output */, current_volume_percent: f32, is_muted: bool, available_ports: Vec<String>, active_port: Option<String> }`
            - Ausgabe von `list_audio_streams`: `system::audio::AudioStreamDetails { id: String, application_name: String, media_role: String, current_volume_percent: f32, is_muted: bool }`
        - **Von der UI aufgerufene API-Methoden (aus `system::AudioInterface`-Trait):**
            - `fn list_audio_devices(&self, device_type_filter: Option<system::audio::AudioDeviceType>) -> Result<Vec<system::audio::AudioDeviceDetails>, system::AudioError>;`
            - `fn get_audio_device_details(&self, device_id: String) -> Result<Option<system::audio::AudioDeviceDetails>, system::AudioError>;`
            - `fn set_device_volume(&self, device_id: String, volume_percent: f32) -> Result<(), system::AudioError>;`
            - `fn set_device_mute(&self, device_id: String, muted: bool) -> Result<(), system::AudioError>;`
            - `fn set_default_output_device(&self, device_id: String) -> Result<(), system::AudioError>;`
            - `fn set_default_input_device(&self, device_id: String) -> Result<(), system::AudioError>;`
            - `fn list_audio_streams(&self) -> Result<Vec<system::audio::AudioStreamDetails>, system::AudioError>;`
            - `fn set_stream_volume(&self, stream_id: String, volume_percent: f32) -> Result<(), system::AudioError>;`
            - `fn set_stream_mute(&self, stream_id: String, muted: bool) -> Result<(), system::AudioError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `system::audio::AudioDeviceChangedEvent { device_details: system::audio::AudioDeviceDetails }`
            - `system::audio::AudioDeviceAddedEvent { device_details: system::audio::AudioDeviceDetails }`
            - `system::audio::AudioDeviceRemovedEvent { device_id: String }`
            - `system::audio::AudioStreamAddedEvent { stream_details: system::audio::AudioStreamDetails }`
            - `system::audio::AudioStreamRemovedEvent { stream_id: String }`
            - `system::audio::AudioStreamVolumeChangedEvent { stream_id: String, new_volume_percent: f32, is_muted: bool }`
            - `system::audio::DefaultDeviceChangedEvent { device_type: system::audio::AudioDeviceType, new_default_device_id: String }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `system::AudioError`-Varianten.
    - **1.3.6. MCP-Client (`system::mcp_client_iface`) Nutzung durch die UI:**
        
        - **Zweck:** Stellt der UI KI-Funktionen zur Verfügung, z.B. für die Befehlspalette oder andere intelligente UI-Elemente.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `request_ai_completion`: `prompt: String`, `context_data: HashMap<String, glib::Variant>`, `max_tokens: Option<u32>`
            - Ausgabe von `request_ai_completion`: `system::mcp::AICompletionResponse { completion_id: core::types::Uuid, text_result: Option<String>, structured_result: Option<glib::Variant>, error_message: Option<String> }`
        - **Von der UI aufgerufene API-Methoden (aus `system::MCPClientInterface`-Trait):**
            - `async fn request_ai_completion(&self, prompt: String, context_data: HashMap<String, glib::Variant>, options: system::mcp::AICompletionOptions) -> Result<system::mcp::AICompletionResponse, system::MCPError>;`
            - `async fn get_ai_capabilities(&self) -> Result<Vec<system::mcp::AICapabilityDescriptor>, system::MCPError>;`
        - **Von der UI konsumierte Event-Payloads:**
            - `system::mcp::AIModelStatusChangedEvent { model_id: String, is_available: bool }`
        - **Fehlerbehandlung durch die UI:** Abbildung von `system::MCPError`-Varianten.
    - **1.3.7. XDG Desktop Portals (`system::portals_iface`) Nutzung durch die UI:**
        
        - **Zweck:** Ermöglicht der UI die Interaktion mit XDG Desktop Portals für Aktionen wie Dateiauswahl, Screenshots, etc., auf eine sandkastenfreundliche Weise. Die Systemschicht abstrahiert die direkte D-Bus-Kommunikation mit den Portal-Diensten. Die `ashpd`-Bibliothek 28 ist ein gutes Beispiel für eine solche Abstraktionsebene, die die Systemschicht intern nutzen könnte.
        - **Ausgetauschte Datenstrukturen (Typen):**
            - Eingabe für `open_file_dialog`: `parent_window_identifier: system::portals::WindowIdentifier`, `title: String`, `options: system::portals::OpenFileOptions { multiple: bool, directory: bool, filters: Vec<system::portals::FileFilter> }`
            - Struct `system::portals::FileFilter { name: String, patterns: Vec<String> /* z.B. "*.txt" */ }`
            - Ausgabe von `open_file_dialog`: `Vec<PathBuf>` (URIs werden von der Systemschicht in Pfade konvertiert)
            - Eingabe für `save_file_dialog`: `parent_window_identifier: system::portals::WindowIdentifier`, `title: String`, `options: system::portals::SaveFileOptions { current_name: Option<String>, current_folder: Option<PathBuf>, filters: Vec<system::portals::FileFilter> }`
            - Ausgabe von `save_file_dialog`: `Option<PathBuf>`
            - Typ `system::portals::WindowIdentifier`: Ein plattformagnostischer Handle für ein Fenster, der intern zu Wayland- oder X11-spezifischen Handles aufgelöst wird, wie von `ashpd` benötigt.
        - **Von der UI aufgerufene API-Methoden (aus `system::PortalsInterface`-Trait):**
            - `async fn open_file_dialog(&self, parent_window_identifier: system::portals::WindowIdentifier, title: String, options: system::portals::OpenFileOptions) -> Result<Vec<PathBuf>, system::PortalError>;`
            - `async fn save_file_dialog(&self, parent_window_identifier: system::portals::WindowIdentifier, title: String, options: system::portals::SaveFileOptions) -> Result<Option<PathBuf>, system::PortalError>;`
            - `async fn take_screenshot(&self, parent_window_identifier: system::portals::WindowIdentifier, interactive: bool, include_cursor: bool) -> Result<Option<PathBuf>, system::PortalError>;`
            - `async fn pick_color(&self, parent_window_identifier: system::portals::WindowIdentifier) -> Result<Option<core::types::Color>, system::PortalError>;`
        - **Von der UI konsumierte Event-Payloads:** (Portals sind typischerweise Request-Response, weniger eventbasiert für die UI direkt, außer Status-Events des Portal-Dienstes selbst, die hier aber nicht im Fokus stehen).
        - **Fehlerbehandlung durch die UI:** Abbildung von `system::PortalError`-Varianten (z.B. `UserCancelled`, `PortalNotAvailable`, `BackendError`).
    - **Schlüsseltabellen für Teil 1.3:** Ähnlich wie in Teil 1.2 werden Tabellen für Methodensignaturen, Parameterdetails, Rückgabewertdetails und Ereignis-Payload-Details für jede definierte Systemschicht-Schnittstelle erstellt.
        
        - Die Interaktionen der Systemschicht sind oft komplex und beinhalten viele Low-Level-Details (z.B. D-Bus-Varianten, Wayland-Protokollspezifika). Tabellen machen die abstrahierte Rust-API klar und reduzieren die Mehrdeutigkeit für UI-Entwickler.

---

**Teil 2: Ultra-Granulare Schnittstellendefinition der Kernschicht (Erweiterung des Benutzerdokuments)**

- 2.1. Kernschicht zu allen höheren Schichten
    
    Die Kernschicht (core) stellt fundamentale Bausteine und Dienste bereit, die von allen höheren Schichten (Domäne, System, UI) genutzt werden. Ihre Schnittstellen müssen daher besonders stabil und klar definiert sein.
    
    - **`core::types`:**
        
        - **Datentypen:** Stellt eine Sammlung grundlegender, wiederverwendbarer Datentypen bereit.
            - `Point<T> { x: T, y: T }` (generisch für numerische Typen wie `f32`, `i32`)
            - `Size<T> { width: T, height: T }` (generisch)
            - `Rect<T> { origin: Point<T>, size: Size<T> }` (generisch)
            - `RectInt` ist ein Typalias für `Rect<i32>`.
            - `Color { r: u8, g: u8, b: u8, a: u8 }` (RGBA-Format)
            - `Orientation { Horizontal, Vertical }` (Enum)
            - `Uuid` (Wrapper um die `uuid::Uuid`-Bibliothek, um eine konsistente Verwendung sicherzustellen)
            - `DateTime<Utc>` (Wrapper um `chrono::DateTime<chrono::Utc>`, um Zeitzonenprobleme zu standardisieren)
            - `WindowId` (Typalias, z.B. `pub type WindowId = u64;` oder eine spezifischere Struktur, falls Metadaten benötigt werden)
            - `WorkspaceId` (Typalias, z.B. `pub type WorkspaceId = core::types::Uuid;`)
            - `AppId` (Typalias, z.B. `pub type AppId = String;`, oft im Reverse-DNS-Format wie `org.novade.FileExplorer`)
        - **Nutzung:** Diese Typen werden direkt in den öffentlichen APIs und internen Logiken der höheren Schichten für Geometrieberechnungen, Farbangaben, eindeutige Identifikatoren, Zeitstempel und spezifische Entitäts-IDs verwendet. Die Standardisierung dieser Typen in der Kernschicht verhindert Inkonsistenzen und erleichtert die Interoperabilität zwischen den Modulen.
    - **`core::errors`:**
        
        - **Basis-Fehlertyp:**
            
            Rust
            
            ```
            #
            pub enum CoreError {
                #[error("I/O error: {source}")]
                Io { #[from] source: std::io::Error },
                #[error("Configuration parsing error: {0}")]
                ConfigParseError(String),
                #[error("Invalid state: {0}")]
                InvalidState(String),
                #[error("Functionality not implemented")]
                NotImplemented,
                #[error("Custom error: {0}")]
                Custom(String),
                #
                SerializationJson { #[from] source: serde_json::Error },
                // Weitere generische Fehlerkategorien hier hinzufügen
            }
            ```
            
        - **Modul-spezifische Fehlerstrategie:** Höhere Schichten und Module innerhalb der Kernschicht definieren ihre eigenen spezifischen Fehler-Enums mit `thiserror`. Diese spezifischen Fehler sollten `CoreError` oder andere relevante Fehler wrappen, um die Fehlerkette zu erhalten.
            - Beispiel: `core::config::ConfigError` könnte eine Variante haben wie:
                
                Rust
                
                ```
                #
                pub enum ConfigError {
                    #[error("Failed to open configuration file '{path}': {source}")]
                    FileOpenFailed { path: PathBuf, #[source] source: CoreError },
                    #[error("Failed to parse configuration: {message}")]
                    ParseFailed { message: String, #[source] source: Option<serde_json::Error> },
                    //...
                }
                ```
                
        - **Fehler-Wrapping-Illustration:**
            
            Rust
            
            ```
            // In einem hypothetischen Modul
            #
            pub enum MyModuleError {
                #[error("Failed to read file for stuff: {source}")]
                FileReadFailed(#[source] CoreError),
                //...
            }
            
            fn read_file_content(path: &PathBuf) -> Result<String, CoreError> {
                std::fs::read_to_string(path).map_err(CoreError::from)
            }
            
            fn load_stuff(file_path: &PathBuf) -> Result<(), MyModuleError> {
                let _content = read_file_content(file_path).map_err(MyModuleError::FileReadFailed)?;
                //... weitere Logik...
                Ok(())
            }
            ```
            
            Diese Strategie stellt sicher, dass der ursprüngliche Kontext des Fehlers (`source()`) verfügbar bleibt, was die Fehlersuche und -behebung erheblich erleichtert.
    - **`core::logging`:**
        
        - **Initialisierungsroutine:**
            
            Rust
            
            ```
            pub enum LogOutput {
                Stdout,
                Stderr,
                File(PathBuf),
            }
            
            pub fn initialize_logging(level: tracing::Level, output: LogOutput, format: LogFormat) -> Result<(), CoreError>;
            pub enum LogFormat { Json, Pretty, Compact }
            ```
            
            Diese Funktion konfiguriert das globale Logging-System (basierend auf `tracing`).
        - **Konvention für `tracing`-Makros:**
            - Alle Schichten verwenden die Standard-`tracing`-Makros (`trace!`, `debug!`, `info!`, `warn!`, `error!`).
            - Strukturierte Log-Nachrichten werden bevorzugt, um kontextbezogene Daten effizient zu erfassen: `trace!(target: "core::module_name", "Processing item {}", item_id, item_type = %item.kind, user_id = %current_user.id);`
            - Das `target`-Feld sollte das Modul oder die Komponente angeben, aus der die Log-Nachricht stammt (z.B. `core::config`, `domain::theming`).
            - Spezifische Felder (z.B., `window.id = %window_id`, `event.name = "ThemeChanged"`) sollten verwendet werden, um das Filtern und Analysieren von Logs zu erleichtern.
    - **`core::config`:**
        
        - **`CoreConfig`-Struktur:** Definiert die globalen Kernkonfigurationen. Diese Struktur wird beim Start der Anwendung geladen und als unveränderlich betrachtet.
            
            Rust
            
            ```
            #
            pub struct LoggingConfig {
                pub level: String, // z.B. "info", "debug"
                pub output: String, // z.B. "stdout", "/var/log/novade.log"
                pub format: String, // z.B. "json", "pretty"
            }
            
            #
            pub struct ThemeConfig {
                pub default_theme_name: String,
                pub default_variant: String,
                pub icon_theme: String,
            }
            
            #
            pub struct AIServiceConfig {
                pub service_url: Option<String>,
                pub default_model_id: Option<String>,
                pub consent_required_by_default: bool,
            }
            
            #
            pub struct CoreConfig {
                pub logging: LoggingConfig,
                pub theming: ThemeConfig,
                pub ai: AIServiceConfig,
                pub default_workspace_layout: String, // z.B. "Tiling", "Floating"
                // Weitere Kernkonfigurationen
            }
            ```
            
        - **Konfigurations-API:**
            - `pub fn load_core_config(path: &PathBuf) -> Result<CoreConfig, ConfigError>;` Lädt die Konfiguration aus einer Datei (z.B. TOML oder JSON).
            - `pub fn get_core_config() -> Result<&'static CoreConfig, CoreError>;` Bietet globalen, schreibgeschützten Zugriff auf die geladene Kernkonfiguration. Dies setzt voraus, dass die Konfiguration nach der Initialisierung in einem globalen statischen Speicher (z.B. `once_cell::sync::OnceCell` oder `std::sync::OnceLock`) gehalten wird.
        - **`ConfigError`-Enum:**
            
            Rust
            
            ```
            #
            pub enum ConfigError {
                #[error("Configuration file not found at path: {0}")]
                FileNotFound(PathBuf),
                #[error("Failed to open configuration file '{path}': {source}")]
                FileOpenFailed { path: PathBuf, #[source] source: CoreError },
                #[error("Failed to read configuration file '{path}': {source}")]
                FileReadFailed { path: PathBuf, #[source] source: CoreError },
                #[error("Failed to parse configuration from '{path}': {message}")]
                ParseFailed { path: PathBuf, message: String, #[source] source: Option<Box<dyn std::error::Error + Send + Sync>> }, // Box für Flexibilität bei Parser-Fehlern
                #[error("Configuration validation failed: {0}")]
                ValidationError(String),
            }
            ```
            
            (Hinweis: `Box<dyn std::error::Error + Send + Sync>` für `source` in `ParseFailed` ermöglicht das Wrappen verschiedener Parser-Fehler, z.B. `serde_json::Error`, `toml::de::Error`).
    - **`core::utils`:**
        
        - **Hilfsfunktionen:** Stellt allgemeine, zustandslose Hilfsfunktionen bereit, die von mehreren Schichten genutzt werden können. Diese sollten in thematisch gruppierten Untermodulen organisiert sein.
            - `core::utils::string_utils`: Funktionen zur String-Manipulation (z.B. `truncate_with_ellipsis(text: &str, max_len: usize) -> String;`).
            - `core::utils::geometry_utils`: Einfache geometrische Berechnungen, die nicht in `core::types` selbst implementiert sind (z.B. `fn calculate_rect_area(rect: &core::types::RectInt) -> i32;`).
            - `core::utils::time_utils`: Formatierungs- oder Vergleichsfunktionen für `DateTime<Utc>` (z.B. `fn format_datetime_human_readable(dt: &core::types::DateTime<Utc>) -> String;` unter Verwendung von `chrono::format::strftime`).
            - `core::utils::uuid_utils`: Funktionen zur Erzeugung oder Validierung von UUIDs (z.B. `fn generate_v4_uuid() -> core::types::Uuid;`).
        - **Nutzung:** Direkte Verwendung durch alle höheren Schichten nach Bedarf. Da diese Funktionen zustandslos sind, führen sie keine Seiteneffekte ein und sind einfach zu testen. Die Signaturen dieser Hilfsfunktionen sind Teil der internen API der Kernschicht und sollten bei Stabilitätserwägungen berücksichtigt werden.
    - **Schlüsseltabellen für Teil 2.1:**
        
        1. **`core::types` Detail-Tabelle:**

|   |   |   |   |
|---|---|---|---|
|**Typname**|**Struktur / Enum-Definition**|**Generische Parameter**|**Beschreibung**|
|`Point<T>`|`struct Point<T> { x: T, y: T }`|`T: Copy + Num`|Repräsentiert einen Punkt im 2D-Raum.|
|`Color`|`struct Color { r: u8, g: u8, b: u8, a: u8 }`|-|Repräsentiert eine RGBA-Farbe.|
|`Uuid`|`struct Uuid(uuid::Uuid);`|-|Wrapper für einen universell eindeutigen Identifikator.|
|`DateTime<Utc>`|`struct DateTime<Utc>(chrono::DateTime<chrono::Utc>);`|-|Repräsentiert einen Zeitpunkt in UTC.|
|...|...|...|...|

```
    2.  **`CoreError` Varianten-Tabelle:**
```

|   |   |   |
|---|---|---|
|**Variante**|**Assoziierte Daten (#[from], #[source], Felder)**|**#[error("...")] Format-String**|
|`Io`|`#[from] source: std::io::Error`|`"I/O error: {source}"`|
|`ConfigParseError`|`String` (Meldung)|`"Configuration parsing error: {0}"`|
|`SerializationJson`|`#[from] source: serde_json::Error`|`"Serialization error (serde_json): {source}"`|
|...|...|...|

```
    3.  **`CoreConfig` Felder-Tabelle:**
```

|   |   |   |   |
|---|---|---|---|
|**Feldname**|**Typ**|**Beschreibung**|**Standardwert (falls zutreffend)**|
|`logging`|`LoggingConfig`|Konfiguration für das Logging-System.|Siehe `LoggingConfig`|
|`theming`|`ThemeConfig`|Standard-Theme-Einstellungen.|Siehe `ThemeConfig`|
|`default_workspace_layout`|`String`|Standard-Layout für neue Arbeitsbereiche (z.B. "Tiling").|`"Floating"`|
|...|...|...|...|

```
    Diese Tabellen bieten eine klare, strukturierte Übersicht über die von der Kernschicht bereitgestellten fundamentalen Elemente und sind für Entwickler aller höheren Schichten von entscheidender Bedeutung.
```

---

**Teil 3: Ultra-Granulare Schnittstellendefinition der Domänenschicht (Erweiterung des Benutzerdokuments)**

Die Domänenschicht enthält die Kernlogik und den Zustand der Anwendung. Ihre Schnittstellen definieren, wie höhere Schichten (System und UI) diese Logik nutzen und auf Zustandsänderungen reagieren können.

- **3.1. Domänenschicht zu System- & UI-Schichten (Allgemeine Prinzipien)**
    
    - **Logik und Zustand:** Jede Domänen-Service-Komponente kapselt einen spezifischen Aspekt der Geschäftslogik und des zugehörigen Zustands. Diese werden über öffentliche Methoden der Service-Traits und durch das Aussenden von domänenspezifischen Ereignissen bereitgestellt.
        - Für jeden Domänendienst wird explizit der von ihm verwaltete Zustand (z.B. für `ThemingEngine`: aktuelles Theme, verfügbare Themes, benutzerdefinierte Farbpaletten) und die wichtigsten Geschäftsregeln, die er durchsetzt (z.B. für `WorkspaceManager`: keine zwei Arbeitsbereiche mit demselben Namen, ein Fenster kann nur einem Arbeitsbereich zugewiesen sein), dokumentiert.
    - **Fehlertypen:** Jedes Modul innerhalb der Domänenschicht definiert seine eigenen, auf `thiserror` basierenden Fehler-Enums. Diese Fehler sind spezifisch für die jeweilige Domänenlogik und bieten detaillierte Informationen über aufgetretene Probleme.
        - Beispiel für `ThemingError`:
            
            Rust
            
            ```
            #
            pub enum ThemingError {
                #
                ThemeNotFound { name: String, path: Option<PathBuf> },
                #
                VariantNotFound { theme_name: String, variant_name: String },
                #[error("Failed to load theme assets for '{theme_name}': {source}")]
                AssetLoadError { theme_name: String, #[source] source: core::errors::CoreError },
                #[error("Invalid theme configuration: {message}")]
                InvalidConfiguration { message: String },
                #
                TokenNotFound { theme_name: String, token_name: String },
                #[error("Failed to parse color value '{value}' for token '{token_name}'")]
                ColorParseError { token_name: String, value: String },
            }
            ```
            
            (Diese detaillierte Definition wird für alle im Benutzerdokument genannten Fehler wie `WorkspaceCoreError`, `WindowAssignmentError`, etc. wiederholt.)
    - **Events:** Domänenspezifische Ereignisse werden ausgelöst, um andere Schichten über signifikante Zustandsänderungen oder abgeschlossene Aktionen zu informieren. Die Payload jedes Ereignisses ist so gestaltet, dass sie alle relevanten Informationen enthält, die ein Konsument benötigt, um ohne weitere Abfragen reagieren zu können.
        - Beispiel für `ThemeChangedEvent`:
            
            Rust
            
            ```
            # // Serialize/Deserialize falls Events über IPC gehen könnten
            pub struct PaletteDetails {
                pub primary_accent: core::types::Color,
                pub secondary_accent: core::types::Color,
                pub foreground: core::types::Color,
                pub background: core::types::Color,
                //... weitere relevante Farbdefinitionen
            }
            
            #
            pub struct ThemeChangedEvent {
                pub new_theme_name: String,
                pub new_variant_name: String,
                pub applied_palette: PaletteDetails, // Statt nur IDs, die volle Information
                pub font_details: FontDescription, // Annahme einer Struktur für Font-Details
            }
            
            #
            pub struct FontDescription {
                pub family: String,
                pub size_pt: f32,
                pub style: String, // z.B. "Regular", "Bold"
            }
            ```
            
            (Diese detaillierte Definition wird für alle im Benutzerdokument genannten Ereignisse wie `WorkspaceEvent`, `AIInteractionInitiatedEvent`, etc. wiederholt.)
- **3.2. Spezifische Domänen-Service-Schnittstellen (Ultra-Granular):**
    
    - **`ThemingEngine` API:**
        
        - **Trait Definition:**
            
            Rust
            
            ```
            pub trait ThemingEngine: Send + Sync {
                /// Ruft den aktuell angewendeten und aufgelösten Theme-Zustand ab.
                fn get_current_theme_state(&self) -> Result<AppliedThemeState, ThemingError>;
            
                /// Listet alle verfügbaren Themes und deren Varianten auf.
                fn get_available_themes(&self) -> Result<Vec<ThemeIdentifier>, ThemingError>;
            
                /// Ruft die aktuelle, persistierte Theme-Konfiguration ab (z.B. ausgewählter Theme-Name, Variante, Overrides).
                fn get_current_configuration(&self) -> Result<ThemeConfiguration, ThemingError>;
            
                /// Aktualisiert die Theme-Konfiguration. Änderungen werden persistiert und lösen ggf. ein `ThemeChangedEvent` aus.
                fn update_configuration(&self, config_update: ThemeConfigurationUpdate) -> Result<(), ThemingError>;
            
                /// Lädt alle Theme-Definitionen und Farb-Token neu vom Dateisystem oder der konfigurierten Quelle.
                fn reload_themes_and_tokens(&self) -> Result<(), ThemingError>;
            
                /// Abonniert Änderungen am Theme-Zustand.
                /// Gibt einen Receiver-Endpunkt eines Broadcast-Kanals zurück, über den `ThemeChangedEvent`s empfangen werden können.
                /// Alternativ könnte dies über einen zentralen EventBus erfolgen.
                fn subscribe_to_theme_changes(&self) -> Result<tokio::sync::broadcast::Receiver<ThemeChangedEvent>, ThemingError>;
            }
            ```
            
        - **Datenstrukturen:**
            - `AppliedThemeState { name: String, variant: String, primary_color: core::types::Color, text_color: core::types::Color, background_color: core::types::Color, accent_colors: HashMap<String, core::types::Color>, font_family: String, font_size_pt: f32, raw_tokens: HashMap<String, String> /* Für Debugging oder fortgeschrittene Nutzung */ }`
            - `ThemeIdentifier { id: String, name: String, display_name: String, description: Option<String>, author: Option<String>, variants: Vec<ThemeVariantIdentifier> }`
            - `ThemeVariantIdentifier { id: String, name: String, display_name: String, preview_image_path: Option<PathBuf> }`
            - `ThemeConfiguration { current_theme_id: String, current_variant_id: String, custom_palette_overrides: HashMap<String, core::types::Color>, font_scale_factor: Option<f32> }`
            - `ThemeConfigurationUpdate { theme_id: Option<String>, variant_id: Option<String>, set_custom_palette_overrides: Option<HashMap<String, core::types::Color>>, remove_custom_palette_overrides: Option<Vec<String>>, font_scale_factor: Option<f32> }`
        - **Events:**
            - `ThemeChangedEvent { new_state: AppliedThemeState }` (wie oben definiert, Payload enthält alle relevanten Infos)
        - **Fehler:** `ThemingError` (Varianten wie oben detailliert definiert: `ThemeNotFound`, `VariantNotFound`, `AssetLoadError`, `InvalidConfiguration`, `TokenNotFound`, `ColorParseError`).
    - **`WorkspaceManager` API:**
        
        - **Trait Definition:**
            
            Rust
            
            ```
            pub trait WorkspaceManager: Send + Sync {
                fn create_workspace(&self, params: CreateWorkspaceParams) -> Result<WorkspaceDescriptor, WorkspaceManagerError>;
                fn delete_workspace(&self, workspace_id: core::types::WorkspaceId) -> Result<(), WorkspaceManagerError>;
                fn set_active_workspace(&self, workspace_id: core::types::WorkspaceId) -> Result<(), WorkspaceManagerError>;
                fn get_active_workspace(&self) -> Result<Option<WorkspaceDescriptor>, WorkspaceManagerError>;
                fn get_workspace_by_id(&self, workspace_id: core::types::WorkspaceId) -> Result<Option<WorkspaceDescriptor>, WorkspaceManagerError>;
                fn list_workspaces(&self) -> Result<Vec<WorkspaceDescriptor>, WorkspaceManagerError>;
                fn assign_window_to_workspace(&self, window_id: core::types::WindowId, workspace_id: core::types::WorkspaceId) -> Result<(), WorkspaceManagerError>;
                fn assign_window_to_active_workspace(&self, window_id: core::types::WindowId) -> Result<(), WorkspaceManagerError>;
                fn remove_window_from_workspace(&self, window_id: core::types::WindowId, workspace_id: core::types::WorkspaceId) -> Result<(), WorkspaceManagerError>;
                fn get_window_assignment(&self, window_id: core::types::WindowId) -> Result<Option<core::types::WorkspaceId>, WorkspaceManagerError>;
                fn save_configuration(&self) -> Result<(), WorkspaceManagerError>; // Persistiert die aktuelle Arbeitsbereichsanordnung und -konfiguration
                fn load_configuration(&self) -> Result<(), WorkspaceManagerError>; // Lädt die Konfiguration beim Start
                fn rename_workspace(&self, workspace_id: core::types::WorkspaceId, new_name: String) -> Result<(), WorkspaceManagerError>;
                fn set_workspace_layout(&self, workspace_id: core::types::WorkspaceId, layout: WorkspaceLayoutType) -> Result<(), WorkspaceManagerError>;
                fn reorder_workspaces(&self, ordered_workspace_ids: Vec<core::types::WorkspaceId>) -> Result<(), WorkspaceManagerError>;
                fn subscribe_to_workspace_events(&self) -> Result<tokio::sync::broadcast::Receiver<WorkspaceEvent>, WorkspaceManagerError>;
            }
            ```
            
        - **Datenstrukturen:**
            - `CreateWorkspaceParams { name: Option<String>, layout_hint: Option<WorkspaceLayoutType>, icon_name: Option<String>, activate_when_created: bool }`
            - `WorkspaceDescriptor { id: core::types::WorkspaceId, name: String, display_name: String, icon_name: Option<String>, layout: WorkspaceLayoutType, window_ids: Vec<core::types::WindowId>, is_active: bool, creation_timestamp: core::types::DateTime<Utc>, last_accessed_timestamp: core::types::DateTime<Utc> }`
            - `WorkspaceLayoutType { TilingHorizontal, TilingVertical, Floating, MaximizedSingleWindow }`
        - **Events (als Enum `WorkspaceEvent` zusammengefasst für einfacheres Abonnement):**
            
            Rust
            
            ```
            #
            pub enum WorkspaceEvent {
                WorkspaceCreated { descriptor: WorkspaceDescriptor },
                WorkspaceDeleted { workspace_id: core::types::WorkspaceId, previous_name: String },
                ActiveWorkspaceChanged { old_workspace_id: Option<core::types::WorkspaceId>, new_workspace_id: core::types::WorkspaceId, new_descriptor: WorkspaceDescriptor },
                WorkspaceLayoutChanged { workspace_id: core::types::WorkspaceId, new_layout: WorkspaceLayoutType },
                WindowAssignedToWorkspace { window_id: core::types::WindowId, new_workspace_id: core::types::WorkspaceId, old_workspace_id: Option<core::types::WorkspaceId> },
                WindowRemovedFromWorkspace { window_id: core::types::WindowId, workspace_id: core::types::WorkspaceId },
                WorkspaceRenamed { workspace_id: core::types::WorkspaceId, new_name: String, old_name: String },
                WorkspaceOrderChanged { ordered_workspace_ids: Vec<core::types::WorkspaceId> },
                WorkspaceConfigurationSaved,
                WorkspaceConfigurationLoaded,
            }
            ```
            
        - **Fehler:** `WorkspaceManagerError`, `WorkspaceConfigError`, `WindowAssignmentError` (jeweils mit detaillierten Varianten wie `WorkspaceNotFound`, `NameAlreadyExists`, `WindowNotKnownToCompositor`, `ConfigSaveFailed`, `ConfigLoadFailed`).
    - **(Diese Detailtiefe wird für `AIInteractionLogicService`, `NotificationService` und `GlobalSettingsService` wiederholt, basierend auf den im Benutzerdokument und in Teil 1.2 dieses Dokuments skizzierten Schnittstellen. Für `GlobalSettingsService` ist die Verwendung von `glib::Variant` für Einstellwerte hervorzuheben, um Flexibilität bei den Datentypen zu gewährleisten.12)**
        
    - Schlüsseltabellen für Teil 3.2:
        
        Ähnlich wie in Teil 1.2 und Teil 2.1 werden für jeden Domänendienst detaillierte Tabellen für Methodensignaturen, Parameter, Rückgabewerte und Ereignis-Payloads erstellt.
        
        - **Beispiel: `AIInteractionLogicService` Methodenübersicht**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp (async Result<_, _>)**|**Kurzbeschreibung**|
|`initiate_interaction`|`&self, request: domain::ai::InteractionRequest`|`Result<domain::ai::InteractionResponseHandle, domain::AIInteractionError>`|Startet eine neue KI-Interaktion.|
|`get_interaction_context`|`&self, interaction_id: core::types::Uuid`|`Result<domain::ai::InteractionContext, domain::AIInteractionError>`|Ruft den aktuellen Kontext und Verlauf einer Interaktion ab.|
|`provide_consent`|`&self, interaction_id: core::types::Uuid, model_id: String, granted: bool`|`Result<(), domain::AIInteractionError>`|Gibt die Zustimmung für ein bestimmtes Modell im Kontext einer Interaktion.|
|...|...|...|...|

```
    *   **Beispiel: `AIInteractionStatusChangedEvent` Payload**
```

|   |   |   |
|---|---|---|
|**Feldname**|**Typ**|**Beschreibung**|
|`interaction_id`|`core::types::Uuid`|ID der Interaktion, deren Status sich geändert hat.|
|`new_status`|`domain::ai::AIInteractionStatus`|Der neue Status der Interaktion.|
|`details`|`Option<String>`|Zusätzliche Details oder Fehlermeldung zum Statuswechsel.|

```
    Die präzise Definition dieser Domänenschnittstellen ist fundamental, da sie die Kernfunktionalität von NovaDE kapseln und die Verträge für die Interaktion mit der System- und UI-Schicht festlegen.
```

---

**Teil 4: Ultra-Granulare Schnittstellendefinition der Systemschicht (Erweiterung des Benutzerdokuments)**

Die Systemschicht ist verantwortlich für die Interaktion mit dem zugrundeliegenden Betriebssystem, der Hardware und Low-Level-Diensten wie dem Compositor, Eingabegeräten und D-Bus-Diensten. Sie stellt der UI-Schicht abstrahierte Schnittstellen zu diesen systemnahen Funktionen bereit.

- **4.1. Systemschicht zu UI-Schicht (Allgemeine Prinzipien)**
    
    - **Systemnahe Dienste und Ereignisse:** Die Systemschicht stellt der UI-Schicht Informationen und Ereignisse bereit, die direkt vom Betriebssystem, der Hardware oder systemnahen Daemons stammen. Für jede Kategorie (z.B. Fenstergeometrie, Eingabeereignisse, Systemstatusänderungen) werden spezifische Ereignisse mit detaillierten Payloads definiert.
    - **Technische Umsetzung von UI-Befehlen:** Die Systemschicht empfängt Befehle von der UI-Schicht (z.B. Fenster verschieben, Arbeitsbereich wechseln, Fokus anfordern) und setzt diese technisch um, indem sie mit dem Compositor, dem Fenstermanager oder anderen Systemkomponenten interagiert. Für jeden solchen Befehl wird eine exakte Methodensignatur in der API der Systemschicht definiert, die von der UI aufgerufen wird.
    - **Renderer-Schnittstelle (`system::compositor::renderer_interface`):** Obwohl nicht direkt von der UI-Schicht konsumiert, stellt diese Schnittstelle eine Abstraktion für das Rendering bereit. Wenn diese Schnittstelle Implikationen dafür hat, wie UI-Elemente letztendlich dargestellt werden (z.B. unterstützte Oberflächenformate, Anforderungen an Textur-Sharing), werden diese kurz vermerkt, da sie indirekt die UI-Implementierung beeinflussen können.
    - **Fehlertypen:** Jedes Modul der Systemschicht (z.B. `system::compositor`, `system::input`) definiert eigene, auf `thiserror` basierende Fehler-Enums (z.B. `CompositorError`, `InputError`). Diese Fehler-Enums listen alle spezifischen Fehlerzustände auf, die bei der Interaktion mit den jeweiligen Systemkomponenten auftreten können.
- **4.2. Spezifische Systemschicht-Schnittstellen (Ultra-Granular, für UI-Interaktion):**
    
    - **Compositor-Schnittstelle (`system::compositor_iface`):**
        
        - Diese Schnittstelle wird als Trait `CompositorInterface` definiert, um eine klare API für die UI-Schicht bereitzustellen.
        - **Methoden, die von der UI aufgerufen werden:**
            
            Rust
            
            ```
            pub trait CompositorInterface: Send + Sync {
                /// Fordert den Fokus für ein bestimmtes Fenster an.
                fn request_focus(&self, window_id: core::types::WindowId) -> Result<(), CompositorError>;
            
                /// Verschiebt ein Fenster an eine neue Position.
                fn move_window(&self, window_id: core::types::WindowId, new_pos: core::types::PointInt) -> Result<(), CompositorError>;
            
                /// Ändert die Größe eines Fensters.
                fn resize_window(&self, window_id: core::types::WindowId, new_size: core::types::SizeInt) -> Result<(), CompositorError>;
            
                /// Fordert einen neuen Zustand für ein Fenster an (z.B. Maximieren, Minimieren).
                fn set_window_state(&self, window_id: core::types::WindowId, state: CompositorWindowState) -> Result<(), CompositorError>;
            
                /// Schließt ein Fenster.
                fn close_window(&self, window_id: core::types::WindowId) -> Result<(), CompositorError>;
            
                /// Ruft detaillierte Informationen zu einem bestimmten Fenster ab.
                fn get_window_info(&self, window_id: core::types::WindowId) -> Result<Option<WindowInfo>, CompositorError>;
            
                /// Listet alle aktuell vom Compositor verwalteten Fenster auf.
                fn list_windows(&self) -> Result<Vec<WindowInfo>, CompositorError>;
            
                /// Ruft Informationen zu allen verfügbaren Outputs (Monitoren) ab.
                fn list_outputs(&self) -> Result<Vec<OutputInfo>, CompositorError>;
            
                /// Konfiguriert einen Output (Auflösung, Skalierung, etc.).
                fn configure_output(&self, output_id: String, config: OutputConfigurationRequest) -> Result<(), CompositorError>;
            
                /// Abonniert Compositor-Ereignisse.
                fn subscribe_to_compositor_events(&self) -> Result<tokio::sync::broadcast::Receiver<CompositorEvent>, CompositorError>;
            }
            ```
            
        - **Datenstrukturen:**
            - `WindowInfo { id: core::types::WindowId, title: String, app_id: core::types::AppId, class: Option<String>, geometry: core::types::RectInt, is_focused: bool, is_fullscreen: bool, is_maximized: bool, is_minimized: bool, workspace_id: Option<core::types::WorkspaceId>, parent_id: Option<core::types::WindowId>, transient_for_id: Option<core::types::WindowId>, can_resize: bool, can_maximize: bool, can_minimize: bool, can_close: bool, pid: Option<u32> }`
            - `CompositorWindowState { Maximized, Minimized, Fullscreen, Normal, TiledLeft, TiledRight, Floating }`
            - `OutputInfo { id: String, name: String, description: String, manufacturer: Option<String>, model: Option<String>, serial_number: Option<String>, physical_size_mm: Option<core::types::Size<u32>>, geometry_pixels: core::types::RectInt, scale_factor: f32, current_refresh_rate_mhz: u32, current_mode: OutputMode, available_modes: Vec<OutputMode>, is_primary: bool, is_enabled: bool, transform: core::types::Orientation }`
            - `OutputMode { width_px: u32, height_px: u32, refresh_rate_mhz: u32, is_preferred: bool }`
            - `OutputConfigurationRequest { resolution: Option<core::types::SizeInt>, scale: Option<f32>, position: Option<core::types::PointInt>, enabled: Option<bool>, refresh_rate_mhz: Option<u32>, transform: Option<core::types::Orientation>, is_primary: Option<bool> }`
        - **Events (als Enum `CompositorEvent` für das Abonnement):**
            
            Rust
            
            ```
            #
            pub enum CompositorEvent {
                WindowCreated { info: WindowInfo },
                WindowClosed { window_id: core::types::WindowId, app_id: core::types::AppId },
                WindowFocused { window_id: core::types::WindowId },
                WindowUnfocused { window_id: core::types::WindowId },
                WindowPropertiesChanged { window_id: core::types::WindowId, changed_properties: WindowPropertiesDelta },
                OutputAdded { info: OutputInfo },
                OutputRemoved { output_id: String },
                OutputConfigurationChanged { info: OutputInfo }, // Enthält den neuen Zustand des Outputs
                // Weitere Ereignisse wie Workspace-Wechsel, wenn vom Compositor direkt gemeldet
            }
            
            #
            pub struct WindowPropertiesDelta {
                pub title: Option<String>,
                pub app_id: Option<core::types::AppId>,
                pub class: Option<Option<String>>, // Option<Option<T>> um 'wurde auf None gesetzt' von 'wurde nicht geändert' zu unterscheiden
                pub geometry: Option<core::types::RectInt>,
                pub is_fullscreen: Option<bool>,
                pub is_maximized: Option<bool>,
                pub is_minimized: Option<bool>,
                pub workspace_id: Option<Option<core::types::WorkspaceId>>,
            }
            ```
            
        - **Fehler:** `CompositorError` mit Varianten wie `WindowNotFound`, `OutputNotFound`, `InvalidOperationForWindowState`, `WaylandCommunicationError(String)`, `X11CommunicationError(String)`.
        - Die Systemschicht abstrahiert hier die Komplexität von Wayland-Protokollen wie `wlr-foreign-toplevel-management` 17 oder `ext-foreign-toplevel-list-v1` 17, die für Taskbars und Docks zur Abfrage von Fensterlisten fundamental sind. Die UI konsumiert die `WindowInfo`-Struktur, die von dieser Abstraktion bereitgestellt wird. Für die Implementierung von UI-Elementen wie Panels oder Docks, die eine feste Position auf dem Bildschirm einnehmen, wird die Systemschicht intern `gtk4-layer-shell` 9 verwenden und eine abstrahierte API dafür anbieten.
    - **(Diese Detailtiefe wird für `system::input_iface`, `system::dbus::upower_client_iface`, `system::dbus::logind_client_iface`, `system::dbus::networkmanager_client_iface` (unter Berücksichtigung von D-Bus-Spezifikationen für NetworkManager 23 und BlueZ 20), `system::outputs_iface` (bereits teilweise in Compositor-Schnittstelle enthalten, hier ggf. spezifischere Konfigurationsmethoden), `system::audio_iface` (Abstraktion von PipeWire 26), `system::mcp_client_iface` und `system::portals_iface` (Abstraktion von XDG Desktop Portals, potenziell unter Verwendung von Bibliotheken wie `ashpd` 28) wiederholt, wie in Teil 1, Abschnitt 1.3 skizziert.)**
        
        - Für `system::portals_iface` ist die `WindowIdentifier`-Struktur, die von `ashpd` verwendet wird, ein wichtiger Aspekt, um Portal-Dialoge korrekt dem aufrufenden Anwendungsfenster zuzuordnen. Die Systemschicht muss einen Mechanismus bereitstellen, um diese Kennung von der UI zu erhalten oder abzuleiten.
    - Schlüsseltabellen für Teil 4.2:
        
        Ähnlich wie in den vorherigen Teilen werden für jede Systemschicht-Schnittstelle detaillierte Tabellen für Methodensignaturen, Parameter, Rückgabewerte und Ereignis-Payloads erstellt.
        
        - **Beispiel: `InputInterface` Methodenübersicht**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp (Result<_, _>)**|**Kurzbeschreibung**|
|`set_cursor_theme`|`&self, theme_name: String, size: u32`|`Result<(), system::InputError>`|Setzt das Cursor-Theme und die Größe.|
|`get_current_keyboard_layout`|`&self`|`Result<String, system::InputError>`|Ruft das aktuell aktive Tastaturlayout ab.|
|`subscribe_to_input_events`|`&self`|`Result<tokio::sync::broadcast::Receiver<system::input::InputEventEnum>, system::InputError>`|Abonniert verarbeitete Eingabeereignisse.|

```
    *   **Beispiel: `system::input::InputEventEnum` Payload (Auszug)**
        ```rust
        #
        pub enum InputEventEnum {
            Keyboard(KeyboardEvent),
            PointerButton(PointerButtonEvent),
            PointerMotion(PointerMotionEvent),
            //... weitere Event-Typen
        }

        #
        pub struct KeyboardEvent {
            pub window_id: Option<core::types::WindowId>, // Fenster, das den Fokus hatte, falls zutreffend
            pub key_code: u32,          // System-agnostischer Keycode
            pub raw_key_code: u32,      // Hardware-Keycode
            pub modifiers: KeyModifiers, // Bitmaske für Shift, Ctrl, Alt, Super
            pub state: KeyState,        // Pressed, Released
            pub utf8: Option<String>,   // Decodiertes Zeichen, falls vorhanden
            pub timestamp_us: u64,      // Zeitstempel in Mikrosekunden
        }
        // Definitionen für KeyModifiers, KeyState, PointerButtonEvent etc. folgen
        ```

    Die klare Definition dieser Systemschicht-APIs ist entscheidend, da sie die Brücke zwischen der plattformunabhängigen UI-Logik und den plattformspezifischen Implementierungen schlägt.
```

---

**Teil 5: Verfeinerte Schichtübergreifende Schnittstellenprinzipien und Fortgeschrittene Themen**

Dieser Abschnitt behandelt übergreifende Prinzipien und fortgeschrittene Aspekte, die für das Design und die Implementierung der internen Schnittstellen von NovaDE von Bedeutung sind.

- 5.1. Asynchrone Operationen und UI-Synchronisation
    
    Die Reaktionsfähigkeit der Benutzeroberfläche ist ein primäres Ziel. Viele Operationen, insbesondere solche, die Netzwerkzugriffe, Dateisystemoperationen oder komplexe Berechnungen beinhalten (z.B. Aufrufe an AIInteractionLogicService oder MCPClientInterface), sind inhärent asynchron. Es ist unerlässlich, dass die UI nicht blockiert, während auf den Abschluss dieser Operationen gewartet wird.
    
    - **Muster für von der UI initiierte asynchrone Aufrufe:**
        
        - Die UI-Schicht ruft eine `async fn` Methode eines Domänen- oder Systemdienstes auf (z.B. `async fn perform_long_task() -> Result<Data, ServiceError>;`).
        - Um ein Blockieren des UI-Threads zu verhindern, sollte die UI-Komponente, die die Operation initiiert (z.B. bei einem Button-Klick), die asynchrone Funktion in einer separaten Task ausführen. In einer `gtk4-rs` Umgebung wird hierfür typischerweise `tokio::spawn` (oder ein äquivalenter Executor) verwendet, falls die Service-Methode selbst `Send` ist, oder `glib::MainContext::spawn_local` für nicht-`Send` Futures, die auf dem Hauptthread laufen müssen, aber dennoch asynchron sind.
    - **Übermittlung von Ergebnissen/Fehlern an den UI-Thread:**
        
        - **`glib::MainContext::spawn_local`:** Dies ist der bevorzugte Mechanismus in `gtk4-rs`, um UI-Aktualisierungen sicher auf dem Haupt-GTK-Thread zu planen, sobald eine asynchrone Operation abgeschlossen ist.3
            - **Ablaufbeispiel:**
                1. Eine UI-Komponente (z.B. ein Button-Klick-Handler) startet eine Tokio-Task (`tokio::spawn`).
                2. Diese Tokio-Task führt `await` auf den Aufruf des Domänen- oder Systemdienstes aus.
                3. Nach Abschluss der Operation (ob `Ok` oder `Err`) verwendet die Tokio-Task `main_context.spawn_local(...)`, um eine Closure zu planen.
                4. Diese Closure, die nun garantiert auf dem Hauptthread ausgeführt wird, aktualisiert die GTK-Widgets mit dem Ergebnis oder dem Fehler.
        - **Kanäle (z.B. `tokio::sync::mpsc`, `async_channel`):**
            - Der UI-Thread hält das `Receiver`-Ende des Kanals. Die asynchrone Task (z.B. in einem Tokio-Thread) hält das `Sender`-Ende.
            - Nach Abschluss der asynchronen Operation wird das Ergebnis oder der Fehler über den Kanal gesendet.
            - Der UI-Thread, typischerweise in einer `glib::idle_add_local`-Schleife oder einem `glib::MainContext::spawn_local`-Future, das den Receiver abfragt, verarbeitet eingehende Nachrichten und aktualisiert die Widgets.
            - Es ist wichtig zu beachten, dass `glib::MainContext::channel()` in neueren GLib-Versionen entfernt wurde.39 Der empfohlene Ansatz ist die Verwendung von Standard-Async-Kanälen (wie `async_channel` oder `tokio::sync::mpsc`) und dann `glib::MainContext::default().spawn_local()`, um die Ergebnisse auf den Hauptthread für UI-Aktualisierungen zu bringen. Diese Änderung vereinfacht die Codebasis, da sie auf etablierte asynchrone Rust-Patterns setzt, anstatt eine GLib-spezifische Kanallösung zu verwenden. Die Schnittstellendefinition selbst schreibt diese interne UI-Implementierung nicht vor, aber die Dokumentation für UI-Entwickler, die diese asynchronen APIs konsumieren, sollte dieses Muster empfehlen.
        - **Fehlerbehandlung in asynchroner UI:** Fehler von asynchronen Operationen müssen ebenfalls auf den Hauptthread gemarshallt und angemessen dargestellt werden. Dies kann beispielsweise durch die Anzeige eines `gtk::AlertDialog` 6 oder einer nicht-modalen Benachrichtigung geschehen.
    - **Tabelle: Asynchrones Interaktionsmuster**
        

|   |   |   |   |   |
|---|---|---|---|---|
|**Schritt**|**Aktion**|**Thread**|**Mechanismus**|**Anmerkungen**|
|1|Benutzerinteraktion|UI-Thread|GTK-Signal|z.B. Button-Klick|
|2|Asynchronen Aufruf initiieren|UI-Thread|`tokio::spawn` (für `Send` Futures)|Startet eine neue Task für den Service-Aufruf, um den UI-Thread nicht zu blockieren.|
|3|Service-Methode ausführen|Worker-Thread (Tokio)|`await service.method()`|Ruft die asynchrone Methode des Domänen-/Systemdienstes auf.|
|4a|Ergebnis/Fehler senden (Kanal-Ansatz)|Worker-Thread (Tokio)|`sender.send(result).await`|Sendet das Ergebnis an einen Kanal, der vom UI-Thread überwacht wird.|
|4b|UI-Update planen (Direkter Spawn-Ansatz)|Worker-Thread (Tokio)|`main_context.spawn_local(async move {... })`|Plant direkt eine Closure zur UI-Aktualisierung auf dem Hauptthread.|
|5a|Empfangen & Verarbeiten (Kanal-Ansatz)|UI-Thread|`receiver.recv().await` (in `spawn_local`)|Die Closure in `spawn_local` empfängt die Nachricht und aktualisiert GTK-Widgets.|
|5b|UI-Update ausführen (Direkter Spawn-Ansatz)|UI-Thread|Ausführung der Closure|Die geplante Closure wird ausgeführt und aktualisiert GTK-Widgets.|

```
    Diese Tabelle verdeutlicht das Threading-Modell und den Kommunikationsfluss für asynchrone Operationen, die von der UI initiiert werden. Dies ist ein häufiger Bereich für Komplexität und Fehlerquellen, daher ist ein klares Verständnis dieses Musters entscheidend.
```

- 5.2. Ereignissystem: Granulare Semantik
    
    Das Ereignissystem ist ein Rückgrat für die reaktive Natur von NovaDE. Eine präzise Definition der Ereignissemantik ist unerlässlich.
    
    - **Ereignisdefinition:** Jede Ereignisstruktur (`struct`) muss vollständig definiert sein, einschließlich aller Felder und deren exakter Rust-Typen. Die Nutzdaten (Payload) eines Ereignisses sollten umfassend sein, um es den Konsumenten zu ermöglichen, ohne zusätzliche Abfragen an den Ereigniserzeuger zu reagieren. Beispielsweise sollte ein `ThemeChangedEvent` nicht nur eine ID des neuen Themes enthalten, sondern direkt die `AppliedThemeState`-Struktur, die alle relevanten Details des neuen Themes beinhaltet. Dies minimiert den Kommunikationsaufwand und verbessert die Performance.
    - **Payload-Serialisierung:** Für die interne Kommunikation zwischen den Schichten innerhalb desselben Prozesses ist eine direkte Übergabe von Rust-Strukturen ausreichend und performant. Sollten Ereignisse jedoch zukünftig über Prozessgrenzen hinweg kommuniziert werden (z.B. über D-Bus an externe Komponenten), muss ein Serialisierungsformat (z.B. Serde mit JSON/CBOR oder GVariant für D-Bus) definiert werden. Die aktuellen Ereignisstrukturen sollten mit `#` annotiert werden, um diese zukünftige Erweiterbarkeit vorzubereiten.
    - **Ereignisgarantien:**
        - **Reihenfolge:** Innerhalb eines bestimmten Geltungsbereichs (z.B. Ereignisse von einem einzelnen Service) ist typischerweise davon auszugehen, dass Ereignisse in der Reihenfolge empfangen werden, in der sie ausgelöst wurden, vorausgesetzt, der Emitter ist single-threaded oder verwendet geeignete Synchronisationsmechanismen. Für einen systemweiten Event-Bus kann die globale Reihenfolge komplexer sein und hängt von der Implementierung des Busses ab.
        - **Zustellung:** Für einen In-Prozess-Event-Bus wird typischerweise eine "At-least-once"-Zustellung angestrebt, solange die Abonnenten aktiv sind und keine Fehler im Bus selbst auftreten. Garantien für "Exactly-once" sind komplexer zu implementieren.
    - **Abonnement-Mechanismus:**
        - Es muss eine klare API für Komponenten geben, um spezifische Ereignistypen zu abonnieren. Dies könnte über einen zentralen `EventBus` erfolgen: `event_bus.subscribe::<ThemeChangedEvent>(|event: &ThemeChangedEvent| { /* UI-Logik hier */ });`.
        - Der Event-Bus sollte Filterfunktionen unterstützen, z.B. das Abonnieren von `WorkspaceEvent` nur für eine bestimmte `workspace_id` oder das Filtern von `SettingChangedEvent` nach einem bestimmten `setting_key`.
        - Die zurückgegebenen `tokio::sync::broadcast::Receiver` von den `subscribe_to_..._events` Methoden der Service-Traits sind ein Beispiel für einen solchen Mechanismus.
    - **Broadcasting vs. Gezielte Ereignisse:** Die meisten Domänen- und Systemereignisse werden als Broadcast an alle interessierten Abonnenten gesendet. Gezielte Ereignisse (nur an einen bestimmten Listener) sind seltener und erfordern einen komplexeren Registrierungs- und Routing-Mechanismus. Für die interne Schichtkommunikation ist Broadcasting meist ausreichend.
    - Die Reaktivität der Benutzeroberfläche hängt stark von einem wohldefinierten und zuverlässigen Ereignissystem ab. Granulare Ereignis-Payloads sind der Schlüssel zur Minimierung von Daten-Refetching und zur Gewährleistung einer performanten UI. GTK4-Signale und Ereignisbehandlungsmechanismen werden intern von der UI-Schicht verwendet, um auf diese anwendungsweiten Ereignisse zu reagieren, die von den unteren Schichten über den Event-Bus oder direkte Abonnements empfangen werden.
- **5.3. API-Versionierung und Evolution (Kurzer Hinweis)**
    
    - APIs unterliegen im Laufe der Zeit Änderungen. Um die Stabilität und Wartbarkeit des Systems langfristig zu gewährleisten, sollten Strategien für die API-Evolution berücksichtigt werden:
        - **Hinzufügen neuer Methoden zu Traits:** Dies ist für bestehende Implementierer nicht-brechend, wenn die neuen Methoden Standardimplementierungen haben (obwohl dies für Kern-APIs seltener der Fall ist).
        - **Hinzufügen neuer optionaler Felder zu Strukturen/Event-Payloads:** Erfordert eine sorgfältige Behandlung durch die Konsumenten (z.B. Verwendung von `Option<T>` und `#[serde(default)]`).
        - **Einführung neuer Versionen von Traits (z.B. `ThemingEngineV2`):** Für signifikante, brechende Änderungen ist dies der sauberste Ansatz.
    - Diese Überlegungen sind zunächst auf hoher Ebene, aber wichtig für die langfristige Gesundheit des Projekts. Eine klare Dokumentation von Änderungen und Deprecation-Policys ist ebenfalls entscheidend.

---

**Teil 6: Schnittstellendetails UI-Komponenten-spezifisch**

Dieser Abschnitt widmet sich der ultra-granularen Definition der Interaktionen spezifischer UI-Komponenten mit den zuvor definierten Domänen- und System-APIs. Er beleuchtet, wie Schlüsselkomponenten der Benutzeroberfläche diese Schnittstellen nutzen, um ihre Funktionalität zu realisieren.

- **6.1. Window Manager Frontend (`ui::window_manager_frontend`)**
    - **Verantwortlichkeiten:** Anzeige von Fensterrepräsentationen (z.B. in einer Taskleiste, Alt-Tab-Umschalter), Bearbeitung von Fokusanfragen für Fenster von UI-Elementen.
    - **Interaktion mit `system::compositor_iface`:**
        - Konsumiert `system::compositor::WindowCreatedEvent`, `WindowClosedEvent`, `WindowFocusedEvent`, `WindowUnfocusedEvent`, `WindowPropertiesChangedEvent`, um seine interne Liste der Fenster und deren Zustände zu aktualisieren.
        - Ruft `compositor_iface.request_focus(window_id)` auf, wenn der Benutzer ein Fenster auswählt (z.B. durch Klick in der Taskleiste).
        - Ruft `compositor_iface.set_window_state(window_id, CompositorWindowState::Minimized)` oder `close_window(window_id)` auf, basierend auf Benutzeraktionen.
    - **Interne UI-Modelldatenstrukturen:**
        - `UIWindowRepresentation { id: core::types::WindowId, title: String, app_id: core::types::AppId, icon_name: Option<String>, // Name für Themed Icon gtk_icon: Option<gtk::gdk_pixbuf::Pixbuf>, // Geladenes Icon is_focused: bool, is_minimized: bool, workspace_tag: Option<String> /* z.B. "WS1" */ }`
    - **GTK-Implementierungshinweise:**
        - Könnte `gtk::ListView` 40 oder `gtk::FlowBox` 45 mit benutzerdefinierten Widgets für jedes Fenster verwenden. Jedes benutzerdefinierte Widget würde an eine `UIWindowRepresentation` binden.
        - Für die Fenster-Thumbnails oder Live-Vorschauen (wie in einem Alt-Tab-Switcher) sind Wayland-spezifische Protokolle oder Compositor-Features notwendig. Die Systemschicht müsste eine API bereitstellen, um solche Texturen oder Oberflächen-Handles zu liefern, die dann in einem GTK-Widget (ggf. ein spezielles Wayland-Oberflächen-Widget oder ein `gtk::Picture` mit aktualisierten Snapshots) dargestellt werden könnten. Wakefield 47 ist ein Beispiel für einen GTK-basierten Compositor, der Client-Oberflächen in GTK-Widgets einbetten kann; ähnliche Konzepte könnten für Vorschauen relevant sein, auch wenn NovaDE kein verschachtelter Compositor ist. Die `ext-foreign-toplevel-list-v1` und `wlr-foreign-toplevel-management-unstable-v1` Protokolle 17 sind fundamental für Taskleisten, um Fensterlisten und Metadaten zu erhalten. Die Systemschicht abstrahiert diese Protokolle, und die UI konsumiert die `WindowInfo`-Strukturen.
    - **Tabellen:**
        - **Interaktion `ui::window_manager_frontend` mit `system::compositor_iface`**

|   |   |   |   |
|---|---|---|---|
|**UI-Aktion**|**Aufgerufene compositor_iface-Methode**|**Konsumierte CompositorEvents**|**Aktualisierte UI-Daten**|
|Fenster in Taskleiste anklicken|`request_focus(window_id)`|`WindowFocusedEvent`, `WindowUnfocusedEvent`|`UIWindowRepresentation::is_focused`|
|Fenster minimieren|`set_window_state(id, Minimized)`|`WindowPropertiesChangedEvent` (mit `is_minimized = true`)|`UIWindowRepresentation::is_minimized`|
|Neues Fenster erscheint|-|`WindowCreatedEvent`|Neue `UIWindowRepresentation`|
|Fenstertitel ändert sich|-|`WindowPropertiesChangedEvent` (mit `changed_title`)|`UIWindowRepresentation::title`|

- **6.2. Desktop Shell (`ui::shell`) - Panels, Docks**
    
    - **Verantwortlichkeiten:** Anzeige von Panels (oben, unten, seitlich), Docks, Arbeitsbereichswechslern, Uhr, Systemindikatoren (Netzwerk, Lautstärke, Akku etc.).
    - **Interaktion mit `gtk4-layer-shell` (über Systemschicht-Abstraktion):**
        - Die Systemschicht stellt eine API bereit, z.B. `system::desktop_shell_iface`, um Layer-Oberflächen zu erstellen und zu verwalten. Diese API kapselt die direkte Verwendung von `gtk4-layer-shell`.9
        - `fn create_layer_surface(&self, params: LayerSurfaceParams) -> Result<LayerSurfaceHandle, ShellError>;`
            - `LayerSurfaceParams { window_handle: gtk::WindowHandle, // Das GTK-Fenster, das als Layer-Oberfläche dient anchor: LayerShellAnchor, // Bitflags: Top, Bottom, Left, Right exclusivity: LayerShellExclusivity, // Exclusive, OnDemand keyboard_interactivity: LayerShellKeyboardInteractivity // None, Exclusive, OnDemand margins: (i32, i32, i32, i32) // top, right, bottom, left }`
        - Die UI verwendet dieses `LayerSurfaceHandle`, um ihre GTK-Panel- oder Dock-Widgets in der Layer-Oberfläche zu positionieren und zu verankern.
        - Die Verwendung von `gtk4-layer-shell` ist entscheidend für die korrekte Integration von Panels und Docks in Wayland-Umgebungen, da es Apps erlaubt, sich über oder unter normalen Fenstern zu positionieren und Platz auf dem Bildschirm zu reservieren.9
    - **Arbeitsbereichswechsler:**
        - Konsumiert `domain::workspaces::WorkspaceEvent`-Ereignisse vom `WorkspaceManager`.
        - Zeigt `domain::workspaces::WorkspaceDescriptor`-Daten an (Name, Icon, aktive Fenster).
        - Ruft `WorkspaceManager::set_active_workspace(workspace_id)` bei Benutzerinteraktion auf.
        - GTK-Implementierung: Könnte eine `gtk::Box` mit `gtk::ToggleButton`s oder benutzerdefinierten gezeichneten Elementen für jeden Arbeitsbereich sein. `chunks-rs` 50 und allgemeine Pager-Beispiele 51 zeigen Konzepte für die Implementierung von Arbeitsbereichswechslern in GTK.
    - **Uhr / Kalenderanzeige:**
        - Verwendet `core::utils::time_utils` (basierend auf `chrono`) zur Formatierung von Datum und Uhrzeit.
        - Kann ein `gtk::Calendar` in einem `gtk::Popover` anzeigen, wenn auf die Uhr geklickt wird. Das Popover wird relativ zum Uhr-Widget positioniert.
    - **Systemindikatoren (Lautstärke, Netzwerk, Akku, Bluetooth etc.):**
        - **Lautstärke:**
            - Konsumiert `system::audio::AudioDeviceChangedEvent`, `AudioStreamVolumeChangedEvent` etc. vom `system::audio_iface`.
            - Zeigt aktuelle Lautstärke mit `gtk::Scale` (als Slider) oder `gtk::VolumeButton` 52 an.
            - Sendet `system::audio_iface::set_device_volume()` bei Benutzerinteraktion.
        - **Netzwerk:**
            - Konsumiert `system::dbus::NetworkStateChangedEvent`, `AccessPointsChangedEvent` vom `system::dbus::networkmanager_client_iface`.
            - Zeigt Verbindungsstatus (Icon), SSID, verfügbare Netzwerke in einem `gtk::Popover` mit einer `gtk::ListBox` an.
            - Ruft `system::dbus::networkmanager_client_iface::connect_to_access_point()` auf.
        - **Akku:**
            - Konsumiert `system::dbus::UPowerDeviceChangedEvent` vom `system::dbus::upower_client_iface`.
            - Zeigt Akkustand (Icon, Prozent) und geschätzte Restlaufzeit an.
        - **Bluetooth:**
            - Konsumiert Ereignisse von einem `system::dbus::bluez_adapter_iface` (analog zu NetworkManager).
            - Zeigt Bluetooth-Status, gekoppelte und verfügbare Geräte an.
            - Ruft Methoden wie `pair_device()`, `connect_device()` auf der BlueZ-Schnittstelle auf.20
        - **StatusNotifierItem/Ayatana Indicators (System Tray):**
            - Die Implementierung eines System Trays unter Wayland ist komplex. Die Spezifikation `StatusNotifierItem` (SNI) 48 ist der De-facto-Standard, der über D-Bus funktioniert.
            - Die `ui::shell` müsste als `StatusNotifierHost` agieren. Dies beinhaltet das Lauschen auf den D-Bus nach Diensten, die `StatusNotifierWatcher.RegisterStatusNotifierItem` aufrufen, und dann die Interaktion mit jedem `StatusNotifierItem` über dessen D-Bus-Schnittstelle, um Icon, Tooltip, Menü etc. abzurufen und darzustellen.
            - Die UI-Elemente für jedes Tray-Icon (typischerweise ein `gtk::Button` mit einem `gtk::Image` und einem `gtk::MenuButton` oder `gtk::Popover` für das Kontextmenü) würden dynamisch in einem Bereich des Panels erstellt.
            - Alternative Ansätze oder Herausforderungen unter Wayland werden diskutiert.58 NovaDE wird sich auf die `StatusNotifierItem`-Spezifikation konzentrieren, da sie am weitesten verbreitet ist. Die `libayatana-indicator`-Bibliothek 63 bietet eine Implementierung, die als Referenz dienen kann, obwohl NovaDE seine eigene D-Bus-Interaktion wahrscheinlich direkt mit `zbus` oder `gio::DBusConnection` implementieren würde, abstrahiert durch die Systemschicht.
            - Die Systemschicht würde eine `system::status_notifier_host_iface` bereitstellen, die Ereignisse wie `StatusNotifierItemRegisteredEvent { service_name: String, object_path: String }` und `StatusNotifierItemUnregisteredEvent` aussendet. Die UI würde diese abonnieren und dann die `system::status_notifier_item_proxy_iface` verwenden, um mit den einzelnen Items zu interagieren (z.B. `get_icon_pixmap()`, `get_tooltip()`, `activate()`, `context_menu()`).
- **6.3. Benachrichtigungs-Popups (`ui::notification_popups`)**
    
    - **Verantwortlichkeiten:** Anzeige von transienten Benachrichtigungs-Popups.
    - **Interaktion mit `domain::NotificationService`:**
        - Konsumiert `domain::notifications::NotificationPostedEvent`.
        - Zeigt die `domain::notifications::Notification`-Daten (Zusammenfassung, Textkörper, Icon, Aktionen) an.
    - **Interaktion mit `gtk4-layer-shell` (über Systemschicht-Abstraktion):**
        - Verwendet das Layer-Shell-Protokoll, um Benachrichtigungs-Popups an einer bestimmten Bildschirmkante (z.B. oben rechts) ohne Interferenz mit anderen Fenstern zu positionieren. Dies ist entscheidend für nicht-invasive Benachrichtigungen.9
        - Die Systemschicht stellt eine Methode bereit, z.B. `system::desktop_shell_iface::create_notification_layer_surface(params: NotificationSurfaceParams) -> Result<LayerSurfaceHandle, ShellError>;`, wobei `NotificationSurfaceParams` Details wie Anker, Dauer und Größe enthalten kann.
    - **Benutzerinteraktion:**
        - Klick auf eine Benachrichtigung könnte `NotificationService::dismiss_notification(id)` aufrufen oder eine Standardaktion auslösen (falls definiert).
        - Klick auf einen Aktionsbutton in einer Benachrichtigung (z.B. "Antworten", "Archivieren") resultiert in einem Aufruf an `NotificationService::invoke_action_on_notification(notification_id, action_id)`, was wiederum ein `domain::notifications::NotificationActionInvokedEvent` auslösen kann, auf das andere Teile der Anwendung reagieren können.
    - **GTK-Implementierungshinweise:**
        - Jedes Popup ist ein eigenes `gtk::Window`, das als Layer-Oberfläche konfiguriert ist. Es enthält typischerweise `gtk::Image` für das Icon, `gtk::Label` für Text und `gtk::Button` für Aktionen.
        - Mehrere Popups müssen verwaltet werden (z.B. in einer Warteschlange oder gestapelt auf dem Bildschirm).65
    - NovaDE wird seine eigenen Benachrichtigungen zeichnen, um volle Kontrolle über das Erscheinungsbild und die Integration mit dem Theming-System zu haben, anstatt sich auf einen externen Freedesktop-Benachrichtigungsdaemon zu verlassen.67 Der `domain::NotificationService` dient als zentrale Anlaufstelle und leitet die `NotificationPostedEvent` an die `ui::notification_popups`-Komponente weiter.
- **6.4. Anwendungsstarter (z.B. Dock, Anwendungsraster)**
    
    - **Verantwortlichkeiten:** Anzeige verfügbarer Anwendungen, Starten von Anwendungen.
    - **Interaktion mit `system::app_launcher_iface`:**
        - Ruft `system::app_launcher_iface::list_applications()` auf, um eine `Vec<system::applications::ApplicationEntry>` zu erhalten.
        - Zeigt diese Einträge an (Icon, Name). Die Icons werden basierend auf dem Icon-Namen und dem aktuellen Theme geladen.
        - Ruft `system::app_launcher_iface::launch_application(app_id, files_to_open: Option<Vec<PathBuf>>)` bei Benutzeraktivierung (Klick) oder per Drag-and-Drop auf.
    - **Interne UI-Modelldatenstrukturen:**
        - `UIAppEntry { id: String, // Desktop-Datei-ID oder AppStream-ID name: String, generic_name: Option<String>, comment: Option<String>, icon_name: Option<String>, gtk_icon: Option<gtk::gdk_pixbuf::Pixbuf>, categories: Vec<String>, keywords: Vec<String> }`
        - Die `system::app_launcher_iface` würde intern `freedesktop_entry_parser` 71 oder ähnliche Mechanismen verwenden, um `.desktop`-Dateien zu parsen und die `ApplicationEntry`-Daten bereitzustellen. Die UI konsumiert diese abstrahierten Daten.
    - **GTK-Implementierungshinweise:**
        - `gtk::FlowBox` 45 oder `gtk::GridView` (mit `gtk::SignalListItemFactory` 42) eignen sich gut zur Anzeige von App-Icons.
        - Jedes Icon könnte ein benutzerdefiniertes `gtk::Button` sein, das ein `gtk::Image` und optional ein `gtk::Label` enthält.
    - **Drag-and-Drop zum Starten:**
        - App-Icons fungieren als `GtkDropTarget`.74
        - Bei einem Drop werden Datei-URIs/Pfade extrahiert und an `system::app_launcher_iface::launch_application(app_id, Some(dropped_files))` übergeben.
        - Die `system::app_launcher_iface` verwendet intern `gio::AppInfo` und verwandte APIs, um Anwendungen zu starten [76,
# Implementierungsleitfaden für die UI-Schicht: Modul 1 – Applikations-Shell und Kernnavigation

Dieses Dokument beschreibt die Ultra-Feinspezifikationen und hochdetaillierten Implementierungspläne für das erste Modul der UI-Schicht. Das Modul "Applikations-Shell und Kernnavigation" umfasst grundlegende Komponenten, die für den Start, die Hauptfensterverwaltung und zentrale UI-Dienste der Applikation zuständig sind. Jede Komponente ist so detailliert spezifiziert, dass Entwickler sie direkt implementieren können, ohne eigene Entscheidungen treffen oder Logiken entwerfen zu müssen.

## 2. UI-Schicht: Implementierungsplan

### Modul 1: Applikations-Shell und Kernnavigation

Die folgenden Abschnitte definieren die Spezifikationen für die Kernkomponenten der Applikations-Shell und der grundlegenden Navigationsinfrastruktur.

#### 2.1. `AppInitializer` (Service/Non-Widget GObject)

##### 2.1.1. Übersicht und Zweck

Der `AppInitializer` ist ein zentraler Service, der für die Initialisierung der GTK-Applikation verantwortlich ist. Seine Hauptaufgaben umfassen das Einrichten globaler Ressourcen, die Registrierung der Applikations-ID und die Behandlung der `activate`- und `startup`-Signale der `gtk::Application`. Darüber hinaus verwaltet der `AppInitializer` den Lebenszyklus des globalen `ApplicationState`-Objekts, das anwendungsweite Zustandsinformationen und Dienste bündelt. Diese Komponente stellt sicher, dass die Applikation korrekt gestartet wird und das Hauptfenster bei Bedarf erstellt und angezeigt wird. Die Kapselung dieser Logik in `AppInitializer` fördert die Modularität und entkoppelt die applikationsspezifische Startlogik von der allgemeinen GTK-Struktur.

##### 2.1.2. Dateistruktur

Die Implementierung des `AppInitializer` wird in folgenden Dateien organisiert:

- `src/app_shell/app_initializer/mod.rs`: Öffentliches Modul und Wrapper-Definition.
- `src/app_shell/app_initializer/imp.rs`: Private Implementierungsdetails und GObject-Subklassifizierung.

##### 2.1.3. GObject-Definition

- **Wrapper-Typ:**
    
    Rust
    
    ```
    use gtk::glib;
    use gtk::gio;
    use std::cell::{Cell, OnceCell, RefCell};
    use std::rc::Rc;
    use crate::app_shell::main_window::MainWindow; // Annahme: MainWindow ist definiert
    use crate::app_shell::action_handler::MasterActionHandler; // Annahme: MasterActionHandler ist definiert
    use crate::app_shell::settings_service::SettingsService; // Annahme: SettingsService ist definiert
    
    glib::wrapper! {
        pub struct AppInitializer(ObjectSubclass<imp::AppInitializerImp>)
            @extends glib::Object;
    }
    ```
    
- **Implementierungs-Struktur (`imp`):**
    
    Rust
    
    ```
    // In src/app_shell/app_initializer/imp.rs
    use gtk::glib;
    use gtk::gio;
    use gtk::prelude::*;
    use gtk::subclass::prelude::*;
    use std::cell::{Cell, OnceCell, RefCell};
    use std::rc::Rc;
    use crate::app_shell::main_window::MainWindow;
    use crate::app_shell::action_handler::MasterActionHandler;
    use crate::app_shell::settings_service::SettingsService;
    
    #
    pub struct ApplicationState {
        pub settings_service: Option<SettingsService>,
        // Weitere globale Zustände oder Dienste hier
    }
    
    #
    pub struct AppInitializerImp {
        app: OnceCell<gtk::Application>,
        main_window: Cell<Option<MainWindow>>,
        app_state: OnceCell<Rc<RefCell<ApplicationState>>>,
        master_action_handler: OnceCell<Rc<MasterActionHandler>>,
    }
    
    #[glib::object_subclass]
    impl ObjectSubclass for AppInitializerImp {
        const NAME: &'static str = "MyAppInitializer";
        type Type = super::AppInitializer;
        type ParentType = glib::Object;
    
        fn new() -> Self {
            Self {
                app: OnceCell::new(),
                main_window: Cell::new(None),
                app_state: OnceCell::new(),
                master_action_handler: OnceCell::new(),
            }
        }
    }
    
    impl ObjectImpl for AppInitializerImp {
        fn constructed(&self) {
            self.parent_constructed();
            // Initialisiere app_state hier, da es von anderen Komponenten benötigt wird
            let initial_app_state = ApplicationState {
                settings_service: None, // Wird später in on_startup initialisiert
                // Initialisiere andere Felder von ApplicationState
            };
            self.app_state.set(Rc::new(RefCell::new(initial_app_state)))
               .expect("ApplicationState konnte nicht initialisiert werden.");
    
            let action_handler = Rc::new(MasterActionHandler::new());
            self.master_action_handler.set(action_handler)
               .expect("MasterActionHandler konnte nicht initialisiert werden.");
        }
    }
    ```
    
    Die Initialisierung von `ApplicationState` und `MasterActionHandler` in `constructed` stellt sicher, dass diese Kernkomponenten frühzeitig verfügbar sind. `ApplicationState` wird als `Rc<RefCell<ApplicationState>>` gespeichert, um gemeinsam genutzten, veränderlichen Zugriff zu ermöglichen.
    

##### 2.1.4. Eigenschaften

Für den `AppInitializer` selbst sind keine öffentlich exponierten GObject-Eigenschaften vorgesehen, die über die interne Zustandsverwaltung hinausgehen. Der Zustand wird intern durch die Felder in `AppInitializerImp` verwaltet.

##### 2.1.5. Methoden

- **`pub fn new(application_id: &str, flags: gio::ApplicationFlags) -> Self`**
    
    - **Signatur:** `pub fn new(application_id: &str, flags: gio::ApplicationFlags) -> super::AppInitializer`
    - **Zugriffsmodifikator:** `pub`
    - **Parameter:**
        - `application_id`: `&str` – Die eindeutige ID der Applikation (z.B. "org.example.myapp").
        - `flags`: `gio::ApplicationFlags` – Flags zur Konfiguration des Applikationsverhaltens (z.B. `gio::ApplicationFlags::HANDLES_OPEN`).
    - **Rückgabewert:** `super::AppInitializer` – Eine neue Instanz des `AppInitializer`.
    - **Vorbedingungen:** `application_id` darf nicht leer sein und muss einem gültigen Format für Applikations-IDs entsprechen.
    - **Nachbedingungen:** Eine `gtk::Application`-Instanz wurde erstellt und im `AppInitializerImp` gespeichert. Die Signale `startup` und `activate` der `gtk::Application` sind mit den entsprechenden internen Methoden des `AppInitializer` verbunden. Der `AppInitializer` ist bereit, die Applikation auszuführen.
    - **Implementierungslogik (Schritt-für-Schritt):**
        1. Erstelle eine neue `glib::Object`-Instanz vom Typ `super::AppInitializer`. Dies ruft `AppInitializerImp::constructed` auf.
        2. Hole die private Implementierung (`imp`) der erstellten `AppInitializer`-Instanz.
        3. Erstelle die `gtk::Application`:
            
            Rust
            
            ```
            let gtk_app = gtk::Application::builder()
               .application_id(application_id)
               .flags(flags)
               .build();
            ```
            
        4. Speichere `gtk_app` im Feld `imp.app`. Es ist wichtig, `set` nur einmal aufzurufen, da `OnceCell` dies erzwingt.
            
            Rust
            
            ```
            imp.app.set(gtk_app.clone()).expect("gtk::Application konnte nicht gesetzt werden.");
            ```
            
        5. Verbinde das `startup`-Signal der `gtk::Application`. Das Klonen von `app_initializer` ist notwendig, um es in den Closure zu bewegen.
            
            Rust
            
            ```
            let app_initializer_clone_startup = obj.clone(); // obj ist die Self-Instanz
            gtk_app.connect_startup(move |app| {
                app_initializer_clone_startup.imp().on_startup(app);
            });
            ```
            
        6. Verbinde das `activate`-Signal der `gtk::Application`.
            
            Rust
            
            ```
            let app_initializer_clone_activate = obj.clone(); // obj ist die Self-Instanz
            gtk_app.connect_activate(move |app| {
                app_initializer_clone_activate.imp().on_activate(app);
            });
            ```
            
        7. Optional: Registriere die Applikation bei D-Bus, falls dies für die Funktionalität (z.B. Single-Instance-Verhalten über D-Bus) erforderlich ist. Dies geschieht typischerweise im `startup`-Handler, nachdem die Applikation die Kontrolle über die ID erlangt hat.1
            
            Rust
            
            ```
            // Beispielhafte D-Bus Registrierung (kann in on_startup erfolgen)
            // if flags.contains(gio::ApplicationFlags::IS_SERVICE) {
            //     gtk_app.register(None::<&gio::Cancellable>).expect("D-Bus Registrierung fehlgeschlagen");
            // }
            ```
            
        8. Gib die erstellte `AppInitializer`-Instanz zurück.
- **`pub fn run(&self) -> glib::ExitCode`**
    
    - **Signatur:** `pub fn run(&self) -> glib::ExitCode`
    - **Zugriffsmodifikator:** `pub`
    - **Parameter:** Keine.
    - **Rückgabewert:** `glib::ExitCode` – Der Exit-Code der Applikation nach deren Beendigung.
    - **Vorbedingungen:** Die `new`-Methode muss erfolgreich aufgerufen worden sein.
    - **Nachbedingungen:** Die GTK-Applikation wird gestartet und die Hauptschleife betreten. Die Methode blockiert, bis die Applikation beendet wird.
    - **Implementierungslogik (Schritt-für-Schritt):**
        1. Hole die `gtk::Application`-Instanz aus `self.imp().app`.
            
            Rust
            
            ```
            let gtk_app = self.imp().app.get().expect("gtk::Application wurde nicht initialisiert.");
            ```
            
        2. Rufe `gtk_app.run()` auf. Da die GTK-Dokumentation für `gtk_application_run` keine Argumente für `argc` und `argv` in der Rust-Binding-Variante direkt zeigt, wird die parameterlose Variante verwendet, die die Kommandozeilenargumente implizit handhabt.3
            
            Rust
            
            ```
            gtk_app.run()
            ```
            
- **`fn on_startup(&self, app: &gtk::Application)`**
    
    - **Signatur:** `fn on_startup(&self, app: &gtk::Application)`
    - **Zugriffsmodifikator:** Privat (intern von `AppInitializerImp`)
    - **Parameter:**
        - `app`: `&gtk::Application` – Die GTK-Applikationsinstanz.
    - **Rückgabewert:** Keiner.
    - **Vorbedingungen:** Wird vom `startup`-Signal der `gtk::Application` aufgerufen.
    - **Nachbedingungen:** Globale Dienste wie `MasterActionHandler` und `SettingsService` sind initialisiert und konfiguriert. Applikationsweite Aktionen sind registriert.
    - **Implementierungslogik (Schritt-für-Schritt):**
        1. Hole den `MasterActionHandler` aus `self.master_action_handler`.
            
            Rust
            
            ```
            let action_handler = self.master_action_handler.get().expect("MasterActionHandler nicht initialisiert.").clone();
            ```
            
        2. Initialisiere und registriere applikationsweite Aktionen über den `MasterActionHandler`.
            
            Rust
            
            ```
            // Beispiel: action_handler.register_app_action(...);
            // Hier sollten globale Aktionen wie "quit", "about" etc. definiert werden.
            // Siehe Spezifikation für MasterActionHandler (2.4).
            let about_action = gio::SimpleAction::new("about", None);
            let self_clone_for_about = self.clone();
            about_action.connect_activate(move |_, _| {
                self_clone_for_about.show_about_dialog();
            });
            action_handler.register_app_action(app, &about_action);
            ```
            
        3. Initialisiere den `SettingsService`. Der Schema-Name muss mit der gschema.xml-Datei übereinstimmen.
            
            Rust
            
            ```
            let settings_service = SettingsService::new("org.example.myapp")
               .expect("SettingsService konnte nicht initialisiert werden.");
            // Speichere settings_service im ApplicationState
            let app_state = self.app_state.get().expect("ApplicationState nicht initialisiert.");
            app_state.borrow_mut().settings_service = Some(settings_service);
            ```
            
        4. Führe weitere einmalige Initialisierungsaufgaben durch (z.B. Laden von Ressourcen, Konfiguration von Logging).
        5. Stelle sicher, dass die Applikation die Kontrolle über die ID hat, bevor D-Bus-Operationen durchgeführt werden, falls relevant. Die `startup`-Phase ist dafür geeignet.
- **`fn on_activate(&self, app: &gtk::Application)`**
    
    - **Signatur:** `fn on_activate(&self, app: &gtk::Application)`
    - **Zugriffsmodifikator:** Privat (intern von `AppInitializerImp`)
    - **Parameter:**
        - `app`: `&gtk::Application` – Die GTK-Applikationsinstanz.
    - **Rückgabewert:** Keiner.
    - **Vorbedingungen:** Wird vom `activate`-Signal der `gtk::Application` aufgerufen. Dies kann mehrfach geschehen, z.B. wenn versucht wird, eine bereits laufende Instanz erneut zu starten.
    - **Nachbedingungen:** Das Hauptfenster der Applikation (`MainWindow`) wird erstellt (falls noch nicht vorhanden) und angezeigt.
    - **Implementierungslogik (Schritt-für-Schritt):**
        
        1. Überprüfe, ob bereits ein Hauptfenster (`self.main_window`) existiert.
            
            Rust
            
            ```
            if self.main_window.get().is_none() {
                // Hauptfenster existiert noch nicht, erstelle es.
                let app_state = self.app_state.get().expect("ApplicationState nicht initialisiert.").clone();
                let main_window_instance = MainWindow::new(app, app_state);
                self.main_window.set(Some(main_window_instance));
            }
            ```
            
        2. Hole das Hauptfenster (entweder das neu erstellte oder das existierende).
            
            Rust
            
            ```
            let window_to_present = self.main_window.get().clone().expect("Hauptfenster konnte nicht abgerufen werden.");
            ```
            
        3. Zeige das Hauptfenster an und bringe es in den Vordergrund.
            
            Rust
            
            ```
            window_to_present.present();
            ```
            
        
        Die Logik stellt sicher, dass bei wiederholten `activate`-Signalen nicht mehrere Fenster erzeugt werden, sondern das existierende Fenster in den Vordergrund gebracht wird. Dies ist ein typisches Verhalten für Desktop-Applikationen.3
- **`fn show_about_dialog(&self)`**
    
    - **Signatur:** `fn show_about_dialog(&self)`
    - **Zugriffsmodifikator:** Privat (kann aber von einer Action aufgerufen werden)
    - **Parameter:** Keine.
    - **Rückgabewert:** Keiner.
    - **Implementierungslogik (Schritt-für-Schritt):**
        1. Erstelle einen `gtk::AboutDialog`.
            
            Rust
            
            ```
            let about_dialog = gtk::AboutDialog::new();
            ```
            
        2. Setze die Eigenschaften des Dialogs (Programmname, Version, Copyright, Webseite, Autoren etc.). Diese Informationen sollten zentral verwaltet oder aus Build-Informationen bezogen werden.
            
            Rust
            
            ```
            about_dialog.set_program_name("Meine Anwendung");
            about_dialog.set_version(Some("1.0.0"));
            about_dialog.set_copyright(Some("© 2025 Mein Name"));
            about_dialog.set_website(Some("https://example.com"));
            // about_dialog.set_authors(&["Autor 1", "Autor 2"]);
            // about_dialog.set_logo_icon_name(Some("application-x-executable")); // Beispiel Icon
            ```
            
        3. Setze das transiente Elternfenster, falls das Hauptfenster bereits existiert.
            
            Rust
            
            ```
            if let Some(main_window) = self.main_window.get().as_ref() {
                about_dialog.set_transient_for(Some(main_window));
                about_dialog.set_modal(true);
            }
            ```
            
        4. Zeige den Dialog an.
            
            Rust
            
            ```
            about_dialog.present();
            ```
            

##### 2.1.6. Signale

- **Emittierte Signale:** Der `AppInitializer` selbst emittiert keine eigenen GObject-Signale.
- **Behandelte Signale:**
    - `gtk::Application::startup`: Verbunden mit `AppInitializerImp::on_startup`. Dient zur einmaligen Initialisierung der Applikation.
    - `gtk::Application::activate`: Verbunden mit `AppInitializerImp::on_activate`. Dient zur Erstellung und/oder Anzeige des Hauptfensters.

##### 2.1.7. Datenstrukturen

- **`ApplicationState`**
    - **Definition:**
        
        Rust
        
        ```
        // Bereits in imp.rs definiert
        // pub struct ApplicationState {
        //     pub settings_service: Option<SettingsService>,
        //     // Weitere globale Zustände oder Dienste hier
        // }
        ```
        
    - **Attribute:**
        - `settings_service`: `Option<SettingsService>`, `pub` – Hält eine Instanz des `SettingsService` für den Zugriff auf Anwendungseinstellungen. Wird in `on_startup` initialisiert.
    - **Invarianten:** Nach `on_startup` sollte `settings_service` `Some` sein.
    - **Zweck:** Dient als Container für global verfügbare Dienste und Zustandsinformationen, die über `Rc<RefCell<ApplicationState>>` an verschiedene UI-Komponenten weitergegeben werden können. Dies ist eine gängige Methode zur Zustandsverwaltung in GTK-Anwendungen, um Prop-Drilling zu vermeiden und einen zentralen Zugriffspunkt für gemeinsam genutzte Daten zu schaffen.5

##### 2.1.8. UI Layout und Styling

Nicht anwendbar, da `AppInitializer` ein nicht-visueller Service ist.

##### 2.1.9. Exception Handling / Error Types

Fehler bei der Initialisierung (z.B. Laden von Settings, D-Bus-Registrierung) werden über `Result` oder `expect` behandelt. Kritische Fehler während des Starts sollten zum kontrollierten Beenden der Applikation führen.

##### 2.1.10. Asynchrone Operationen

Der `AppInitializer` führt primär synchrone Operationen während der Startup- und Activate-Phasen aus. Asynchrone Operationen könnten bei der D-Bus-Registrierung oder beim Laden von Ressourcen auftreten, würden aber typischerweise innerhalb der Signalhandler mit `glib::MainContext::block_on` oder durch Spawnen auf den Main-Context gehandhabt, falls sie nicht blockierend sein dürfen.

##### 2.1.11. Abhängigkeiten

- `MainWindow`: Für die Erstellung des Hauptfensters.
- `MasterActionHandler`: Für die Registrierung globaler Aktionen.
- `SettingsService`: Für den Zugriff auf und die Initialisierung von Anwendungseinstellungen.
- `gtk::Application`, `gtk::gio`, `gtk::glib`.

##### 2.1.12. Testplan

- **Unit-Tests:**
    - Verifiziere, dass `new()` eine gültige `AppInitializer`-Instanz zurückgibt und die `gtk::Application` korrekt konfiguriert ist (ID, Flags).
    - Teste das Verhalten von `on_startup` (Mocking von Abhängigkeiten wie `SettingsService` könnte notwendig sein, um die korrekte Initialisierung zu überprüfen).
    - Teste das Verhalten von `on_activate`:
        - Erstellt `MainWindow` beim ersten Aufruf.
        - Zeigt existierende `MainWindow` bei nachfolgenden Aufrufen an.
- **Integrationstests:**
    - Simuliere den Applikationsstart und überprüfe, ob das Hauptfenster erscheint.
    - Teste die Reaktion auf mehrfache Aktivierungssignale.
    - Überprüfe die korrekte Registrierung von D-Bus-Diensten (falls implementiert).

#### 2.2. `MainWindow` (Benutzerdefiniertes GTK Widget)

##### 2.2.1. Übersicht und Zweck

Das `MainWindow` ist das Hauptfenster der Applikation und dient als primärer Container für alle anderen UI-Elemente. Es definiert die grundlegende Layoutstruktur, typischerweise bestehend aus einer Seitenleiste für die Navigation, einem Hauptinhaltsbereich und einem Integrationspunkt für die Befehlspalette. Als Unterklasse von `gtk::ApplicationWindow` integriert es sich nahtlos in den Lebenszyklus der `gtk::Application`.3 Die Verwendung eines benutzerdefinierten Widgets für das Hauptfenster ermöglicht eine bessere Kapselung der fensterspezifischen Logik und des Layouts.

##### 2.2.2. Dateistruktur

- `src/app_shell/main_window/mod.rs`: Öffentliches Modul und Wrapper-Definition.
- `src/app_shell/main_window/imp.rs`: Private Implementierungsdetails und GObject-Subklassifizierung.
- `resources/ui/main_window.ui`: (Optional) UI-Definitionsdatei für das Fensterlayout, falls Composite Templates verwendet werden.

##### 2.2.3. GObject-Definition

- **Wrapper-Typ:**
    
    Rust
    
    ```
    // In src/app_shell/main_window/mod.rs
    use gtk::glib;
    use gtk::subclass::prelude::*;
    use gtk::{gio, ApplicationWindow, Widget}; // Weitere Imports nach Bedarf
    use std::rc::Rc;
    use std::cell::RefCell;
    use crate::app_shell::app_initializer::ApplicationState; // Pfad anpassen
    
    glib::wrapper! {
        pub struct MainWindow(ObjectSubclass<imp::MainWindowImp>)
            @extends ApplicationWindow, gtk::Window, Widget,
            @implements gio::ActionGroup, gio::ActionMap, gtk::Accessible,
                        gtk::Buildable, gtk::ConstraintTarget, gtk::Native,
                        gtk::Root, gtk::ShortcutManager;
    }
    ```
    
    Die Implementierung der zahlreichen Interfaces (`gio::ActionGroup`, `gio::ActionMap`, etc.) wird durch `glib::wrapper!` und die Vererbung von `gtk::ApplicationWindow` bereitgestellt.7
    
- **Implementierungs-Struktur (`imp`):**
    
    Rust
    
    ```
    // In src/app_shell/main_window/imp.rs
    use gtk::glib::{self, Properties};
    use gtk::subclass::prelude::*;
    use gtk::{
        ApplicationWindow, Box as GtkBox, Revealer, CompositeTemplate, TemplateChild,
        gio
    };
    use std::cell::{Cell, RefCell};
    use std::rc::Rc;
    use crate::app_shell::app_initializer::ApplicationState; // Pfad anpassen
    use crate::app_shell::command_palette::CommandPaletteWidget; // Annahme
    use crate::app_shell::sidebar::SidebarNavigationWidget; // Annahme
    use crate::app_shell::content_view::ContentViewManager; // Annahme
    
    
    #
    #[template(resource = "/org/example/myapp/main_window.ui")] // Pfad zur UI-Datei
    #[properties(wrapper_type = super::MainWindow)]
    pub struct MainWindowImp {
        #[template_child]
        pub main_box: TemplateChild<GtkBox>,
    
        #[template_child]
        pub content_area_box: TemplateChild<GtkBox>, // Container für Sidebar und Content
    
        #[template_child]
        pub command_palette_revealer: TemplateChild<Revealer>,
    
        #[template_child]
        pub command_palette_widget: TemplateChild<CommandPaletteWidget>, // Instanz der CommandPalette
    
        // Interner Zustand
        #[property(get, set, name = "is-command-palette-visible", nick="Command Palette Visible", blurb="Whether the command palette is currently visible", default=false)]
        is_command_palette_visible_prop: Cell<bool>,
    
        app_state: RefCell<Option<Rc<RefCell<ApplicationState>>>>,
        actions: gio::SimpleActionGroup,
    }
    
    #[glib::object_subclass]
    impl ObjectSubclass for MainWindowImp {
        const NAME: &'static str = "MyMainWindow";
        type Type = super::MainWindow;
        type ParentType = ApplicationWindow;
    
        fn class_init(klass: &mut Self::Class) {
            // UI-Template binden
            klass.bind_template();
            // Template Callbacks binden, falls vorhanden
            // klass.bind_template_callbacks();
        }
    
        fn instance_init(obj: &glib::subclass::InitializingObject<Self>) {
            obj.init_template();
        }
    }
    
    #[glib::derived_properties]
    impl ObjectImpl for MainWindowImp {
        fn constructed(&self) {
            self.parent_constructed();
    
            // Aktionen initialisieren und hinzufügen
            self.obj().setup_actions();
            self.obj().setup_keyboard_shortcuts();
    
            // Command Palette initialisieren und zum Revealer hinzufügen
            // self.command_palette_revealer.set_child(Some(&*self.command_palette_widget));
            // Die obige Zeile ist nicht notwendig, wenn command_palette_widget als Kind von
            // command_palette_revealer in der.ui Datei definiert ist.
    
            // Initialisiere Sidebar und Content View Manager und füge sie zum content_area_box hinzu
            // let sidebar = SidebarNavigationWidget::new(self.app_state.borrow().as_ref().unwrap().clone());
            // let content_manager = ContentViewManager::new(self.app_state.borrow().as_ref().unwrap().clone());
            // self.content_area_box.append(&sidebar);
            // self.content_area_box.append(&content_manager);
            // Diese Initialisierung sollte idealerweise in der UI-Datei oder durch Template-Callbacks erfolgen,
            // um die Logik hier sauber zu halten.
        }
    }
    
    impl WidgetImpl for MainWindowImp {
        // Überschreibe size_allocate, falls notwendig für benutzerdefinierte Layout-Logik
    }
    
    impl WindowImpl for MainWindowImp {
        // Behandle Fensterzustandsänderungen, z.B. Schließen-Anfrage
        fn close_request(&self) -> glib::Propagation {
            // Hier könnte Logik zum Speichern des Zustands oder Bestätigungsdialoge implementiert werden
            println!("Close request received for MainWindow");
            // Um das Schließen tatsächlich zu verhindern:
            // return glib::Propagation::Stop;
            self.parent_close_request()
        }
    }
    impl ApplicationWindowImpl for MainWindowImp {}
    ```
    
    Die Verwendung von `#[derive(Properties)]` und `#[properties(wrapper_type =...)]` ermöglicht die Definition von GObject-Eigenschaften direkt in der `imp`-Struktur.8 Die `#[template_child]`-Annotationen verbinden Felder mit Widgets, die in der UI-Datei definiert sind.10 Die `constructed`-Methode ist der geeignete Ort, um Aktionen einzurichten und initiale UI-Konfigurationen vorzunehmen.7
    
- **UI-Definition (`main_window.ui`):**
    
    XML
    
    ```
    <?xml version="1.0" encoding="UTF-8"?>
    <interface>
      <requires lib="gtk" version="4.0"/>
      <template class="MyMainWindow" parent="GtkApplicationWindow">
        <property name="title" translatable="yes">Meine Anwendung</property>
        <property name="default-width">1024</property>
        <property name="default-height">768</property>
        <child>
          <object class="GtkBox" id="main_box">
            <property name="orientation">vertical</property>
            <child>
              <object class="GtkHeaderBar" id="header_bar">
                </object>
            </child>
            <child>
              <object class="GtkBox" id="content_area_box">
                <property name="orientation">horizontal</property>
                <property name="hexpand">true</property>
                <property name="vexpand">true</property>
                </object>
            </child>
            <child>
              <object class="GtkRevealer" id="command_palette_revealer">
                <property name="transition-type">slide-down</property>
                <property name="transition-duration">250</property>
                <property name="reveal-child">false</property>
                <child>
                  <object class="CommandPaletteWidget" id="command_palette_widget"/>
                </child>
              </object>
            </child>
          </object>
        </child>
      </template>
    </interface>
    ```
    
    Die UI-Datei definiert die Struktur des Fensters. `TemplateChild`-Felder in `MainWindowImp` werden mit den Widgets mit den entsprechenden IDs (`main_box`, `command_palette_revealer`, etc.) verbunden. Die Verwendung von UI-Templates fördert die Trennung von UI-Definition und Logik.10
    

##### 2.2.4. Eigenschaften

- **`is-command-palette-visible`**:
    - **GObject-Name:** `is-command-palette-visible`
    - **Rust-Typ:** `bool` (intern `Cell<bool>`)
    - **Zugriff:** Lesen/Schreiben (R/W)
    - **Standardwert:** `false`
    - **Beschreibung:** Steuert die Sichtbarkeit des `command_palette_revealer`. Änderungen an dieser Eigenschaft lösen ein `notify::is-command-palette-visible`-Signal aus.
    - **Implementierung:** Die Eigenschaft wird durch das `#[property(...)]`-Makro auf `is_command_palette_visible_prop` in `MainWindowImp` generiert. Der Getter liest aus `is_command_palette_visible_prop`. Der Setter schreibt in `is_command_palette_visible_prop` und ruft `self.obj().notify("is-command-palette-visible")` auf, sowie `self.command_palette_revealer.set_reveal_child(new_value)`.

##### 2.2.5. Methoden

- **`pub fn new(app: &gtk::Application, app_state: Rc<RefCell<ApplicationState>>) -> Self`**
    
    - **Signatur:** `pub fn new(app: &gtk::Application, app_state: Rc<RefCell<ApplicationState>>) -> Self`
    - **Zugriffsmodifikator:** `pub`
    - **Parameter:**
        - `app`: `&gtk::Application` – Die Hauptapplikationsinstanz.
        - `app_state`: `Rc<RefCell<ApplicationState>>` – Der globale Anwendungszustand.
    - **Rückgabewert:** `Self` – Eine neue Instanz von `MainWindow`.
    - **Vorbedingungen:** `app` und `app_state` sind gültige, initialisierte Instanzen.
    - **Nachbedingungen:** Ein neues `MainWindow` wird erstellt, mit der Applikation verknüpft und der `app_state` intern gespeichert.
    - **Implementierungslogik (Schritt-für-Schritt):**
        1. Erstelle das Objekt mit `glib::Object::builder().property("application", app).build()`.
        2. Hole die `imp`-Struktur des neu erstellten Fensters.
        3. Speichere `app_state` in `imp.app_state.borrow_mut().replace(app_state);`.
- **`pub fn toggle_command_palette(&self)`**
    
    - **Signatur:** `pub fn toggle_command_palette(&self)`
    - **Zugriffsmodifikator:** `pub`
    - **Parameter:** Keine.
    - **Rückgabewert:** Keiner.
    - **Vorbedingungen:** Das `MainWindow` und seine internen Widgets (`command_palette_revealer`, `command_palette_widget`) sind initialisiert.
    - **Nachbedingungen:** Die Sichtbarkeit der Befehlspalette wird umgeschaltet. Die Eigenschaft `is-command-palette-visible` wird aktualisiert und das Signal `command_palette_toggled` wird emittiert.
    - **Implementierungslogik (Schritt-für-Schritt):**
        1. Hole die `imp`-Struktur.
        2. Lese den aktuellen Zustand von `imp.command_palette_revealer.reveals_child()`.
        3. Setze den neuen Zustand: `imp.command_palette_revealer.set_reveal_child(!current_state)`.
        4. Aktualisiere die Eigenschaft: `self.set_is_command_palette_visible(!current_state)`. Dies löst automatisch das `notify`-Signal für die Eigenschaft aus.
        5. Wenn die Palette sichtbar wird, setze den Fokus auf das Eingabefeld der `CommandPaletteWidget`: `imp.command_palette_widget.grab_focus_entry();` (Annahme: `grab_focus_entry` ist eine Methode von `CommandPaletteWidget`).
        6. Emittiere das `command_palette_toggled`-Signal: `self.emit_by_name::<()>("command-palette-toggled", &[&!current_state]);`.
- **`fn setup_actions(&self)`** (Privat, aufgerufen in `constructed`)
    
    - **Signatur:** `fn setup_actions(&self)`
    - **Implementierungslogik:**
        1. Erstelle `gio::SimpleActionGroup`.
        2. Definiere Aktionen spezifisch für das Fenster (z.B. "toggle-command-palette").
            
            Rust
            
            ```
            let imp = self.imp();
            let toggle_cp_action = gio::SimpleAction::new("toggle-command-palette", None);
            let window_clone = self.clone();
            toggle_cp_action.connect_activate(move |_, _| {
                window_clone.toggle_command_palette();
            });
            imp.actions.add_action(&toggle_cp_action);
            ```
            
        3. Füge die `SimpleActionGroup` zum Fenster hinzu: `self.insert_action_group("win", Some(&imp.actions));`. Der Präfix "win" ist Konvention für fensterspezifische Aktionen.12
- **`fn setup_keyboard_shortcuts(&self)`** (Privat, aufgerufen in `constructed`)
    
    - **Signatur:** `fn setup_keyboard_shortcuts(&self)`
    - **Implementierungslogik:**
        
        1. Definiere Tastenkürzel für Fensteraktionen. Zum Beispiel `Ctrl+P` für die Befehlspalette.
            
            Rust
            
            ```
            let app = self.application().expect("Anwendung nicht gefunden für MainWindow");
            app.set_accels_for_action("win.toggle-command-palette", &["<Primary>P"]);
            // Weitere Kürzel hier definieren
            ```
            
        
        Die Aktionen müssen bereits in der `gio::ActionGroup` des Fensters (hier "win") registriert sein.12

##### 2.2.6. Signale

- **Emittierte Signale:**
    
    - **Signal Name:** `command-palette-toggled`
        - **GObject Name:** `command-palette-toggled`
        - **Parameter:**
            - `visible`: `bool` (Rust), `glib::VariantType::BOOLEAN` (`glib::Variant`) – Gibt an, ob die Palette sichtbar (`true`) oder unsichtbar (`false`) wurde.
        - **Beschreibung:** Wird emittiert, nachdem die Sichtbarkeit der Befehlspalette geändert wurde.
        - **Definition in `ObjectImpl::signals()`:**
            
            Rust
            
            ```
            // In MainWindowImp
            fn signals() -> &'static {
                static SIGNALS: OnceCell<Vec<glib::subclass::Signal>> = OnceCell::new();
                SIGNALS.get_or_init(|| {
                    vec!)
                           .build(),
                    ]
                })
            }
            ```
            
            Die Definition von Signalen erfolgt typischerweise in der `ObjectImpl` der Implementierungsstruktur.11
- **Behandelte Signale:**
    
    - Verbindungen zu Signalen von Kind-Widgets (z.B. `command_palette_widget.command-executed`) werden intern in `constructed` oder bei der Initialisierung der Kind-Widgets eingerichtet.

##### 2.2.7. Datenstrukturen

Keine spezifischen öffentlichen Datenstrukturen, die von `MainWindow` direkt exponiert werden, außer den GObject-Eigenschaften. Der `ApplicationState` wird intern referenziert.

##### 2.2.8. UI Layout und Styling

- **Widget-Hierarchie:**
    - `MyMainWindow` (`gtk::ApplicationWindow`)
        - `main_box` (`gtk::Box`, vertikal)
            - `header_bar` (`gtk::HeaderBar`) (optional, falls nicht CSD)
            - `content_area_box` (`gtk::Box`, horizontal)
                - `SidebarNavigationWidget` (benutzerdefiniert, links)
                - `ContentViewManager` (benutzerdefiniert, rechts, `hexpand=true`)
            - `command_palette_revealer` (`gtk::Revealer`)
                - `command_palette_widget` (`CommandPaletteWidget`, benutzerdefiniert)
- **Layout Manager:** Primär `gtk::Box` für die Hauptstrukturierung.
- **CSS-Klassen und -Namen:**
    - `MainWindow`: CSS-Name `main-window`.
    - `content_area_box`: CSS-Klasse `content-area`.
    - `sidebar_nav` (falls als TemplateChild): CSS-Klasse `sidebar-navigation`.
    - `command_palette_revealer`: CSS-Klasse `command-palette-container`. Styling erfolgt über eine zentrale CSS-Datei, die von der Applikation geladen wird. Spezifische Stile für `MainWindow` und seine Hauptbereiche werden über diese Klassen und Namen ermöglicht.14

##### 2.2.9. Exception Handling / Error Types

Fehler bei der UI-Erstellung (z.B. Laden der UI-Datei) sollten durch `expect` oder `Result` behandelt und geloggt werden. Laufzeitfehler sind typischerweise unwahrscheinlich, es sei denn, es gibt Probleme mit Kind-Widgets.

##### 2.2.10. Asynchrone Operationen

Das `MainWindow` selbst führt keine langlaufenden asynchronen Operationen durch. Es kann jedoch UI-Aktualisierungen als Reaktion auf Ereignisse von asynchronen Diensten (via `ApplicationState` und `AsyncUIManager`) durchführen.

##### 2.2.11. Abhängigkeiten

- `CommandPaletteWidget`: Für die Anzeige und Interaktion mit der Befehlspalette.
- `SidebarNavigationWidget`: Für die Hauptnavigation (Spezifikation folgt).
- `ContentViewManager`: Für die Verwaltung und Anzeige verschiedener Inhaltsansichten (Spezifikation folgt).
- `ApplicationState`: Für den Zugriff auf globale Zustände und Dienste.
- `MasterActionHandler`: Für die Verbindung von UI-Elementen mit globalen Aktionen.
- `gtk::Application`, `gtk::ApplicationWindow`, `gtk::Box`, `gtk::Revealer`, `gtk::gio`.

##### 2.2.12. Testplan

- **Unit-Tests:**
    - Teste die Erstellung des `MainWindow`.
    - Teste die `toggle_command_palette`-Methode:
        - Sichtbarkeit des `command_palette_revealer` ändert sich korrekt.
        - Eigenschaft `is-command-palette-visible` wird aktualisiert.
        - Signal `command_palette_toggled` wird mit korrektem Parameter emittiert.
    - Überprüfe die korrekte Einrichtung der Aktionen (`win.toggle-command-palette`).
- **Integrationstests (mit GTK-Testumgebung):**
    - Überprüfe das korrekte Laden der UI-Datei (falls verwendet).
    - Teste die Reaktion auf das Tastenkürzel zum Öffnen der Befehlspalette.
    - Teste das Schließen des Fensters und die korrekte Ausführung von `close_request`.

#### 2.3. `CommandPaletteWidget` (Benutzerdefiniertes GTK Widget)

##### 2.3.1. Übersicht und Zweck

Das `CommandPaletteWidget` ist eine zentrale UI-Komponente, die es Benutzern ermöglicht, schnell nach Befehlen, Aktionen oder anderen navigierbaren Elementen innerhalb der Applikation zu suchen und diese auszuführen. Es verwendet eine Texteingabe mit Fuzzy-Suche, um eine dynamisch gefilterte Liste von Ergebnissen anzuzeigen. Dieses Konzept ist inspiriert von ähnlichen Funktionen in modernen Entwicklungsumgebungen und Produktivitätswerkzeugen.15 Die Palette wird typischerweise als Pop-up oder einblendbares Element (`gtk::Revealer`) im `MainWindow` integriert.

##### 2.3.2. Dateistruktur

- `src/app_shell/command_palette/mod.rs`: Öffentliches Modul und Wrapper-Definition.
- `src/app_shell/command_palette/imp.rs`: Private Implementierungsdetails und GObject-Subklassifizierung.
- `src/app_shell/command_palette/fuzzy_searcher.rs`: Modul für die Fuzzy-Suchlogik.
- `src/app_shell/command_palette/item.rs`: Definition des `CommandPaletteItem`-GObjects.
- `resources/ui/command_palette_widget.ui`: (Optional) UI-Definitionsdatei.
- `resources/ui/command_palette_item.ui`: (Optional) UI-Definitionsdatei für ein einzelnes Listenelement.

##### 2.3.3. GObject-Definition

- **Wrapper-Typ:**
    
    Rust
    
    ```
    // In src/app_shell/command_palette/mod.rs
    use gtk::glib;
    use gtk::subclass::prelude::*;
    use gtk::{Box as GtkBox, Widget}; // Weitere Imports nach Bedarf
    use crate::app_shell::action_handler::MasterActionHandler; // Annahme
    use std::rc::Rc;
    
    glib::wrapper! {
        pub struct CommandPaletteWidget(ObjectSubclass<imp::CommandPaletteWidgetImp>)
            @extends GtkBox, Widget,
            @implements gtk::Accessible, gtk::Buildable, gtk::ConstraintTarget, gtk::Orientable;
    }
    
    impl CommandPaletteWidget {
        pub fn new(action_handler: Rc<MasterActionHandler>) -> Self {
            let widget: Self = glib::Object::builder().build();
            widget.imp().action_handler.set(Some(action_handler)).expect("ActionHandler konnte nicht gesetzt werden");
            widget
        }
    
        pub fn grab_focus_entry(&self) {
            self.imp().search_entry.grab_focus();
        }
    }
    ```
    
- **Implementierungs-Struktur (`imp`):**
    
    Rust
    
    ```
    // In src/app_shell/command_palette/imp.rs
    use gtk::glib::{self, Properties, Propagation, Variant};
    use gtk::prelude::*;
    use gtk::subclass::prelude::*;
    use gtk::{
        Box as GtkBox, SearchEntry, ListView, ScrolledWindow, SignalListItemFactory,
        ListItem, SingleSelection, FilterListModel, CustomFilter, NoSelection,
        gio, Orientation, Align,
    };
    use std::cell::{Cell, OnceCell, RefCell};
    use std::rc::Rc;
    use crate::app_shell::action_handler::MasterActionHandler;
    use super::fuzzy_searcher::{FuzzySearchAlgorithm, FuzzySearcher}; // Annahme
    use super::item::CommandPaletteItem; // Annahme
    
    #
    #[template(resource = "/org/example/myapp/command_palette_widget.ui")]
    #[properties(wrapper_type = super::CommandPaletteWidget)]
    pub struct CommandPaletteWidgetImp {
        #[template_child]
        pub search_entry: TemplateChild<SearchEntry>,
    
        #[template_child]
        pub results_scrolled_window: TemplateChild<ScrolledWindow>,
    
        #[template_child]
        pub results_list_view: TemplateChild<ListView>,
    
        // Interner Zustand
        list_store: OnceCell<gio::ListStore>, // Speichert CommandPaletteItem GObjects
        filter_model: OnceCell<FilterListModel>,
        selection_model: OnceCell<NoSelection>, // Oder SingleSelection, falls gewünscht
    
        fuzzy_searcher: FuzzySearcher, // Enthält den Algorithmus
        action_handler: OnceCell<Rc<MasterActionHandler>>,
        all_commands: RefCell<Vec<Rc<CommandPaletteItem>>>, // Cache aller verfügbaren Befehle
    }
    
    
    #[glib::object_subclass]
    impl ObjectSubclass for CommandPaletteWidgetImp {
        const NAME: &'static str = "MyCommandPaletteWidget";
        type Type = super::CommandPaletteWidget;
        type ParentType = GtkBox;
    
        fn class_init(klass: &mut Self::Class) {
            klass.bind_template();
            // Template Callbacks für UI-Datei-Events, falls benötigt
            // klass.bind_template_callback("search_entry_search_changed_cb", |widget: &super::CommandPaletteWidget, entry: &SearchEntry| {
            // widget.imp().on_search_changed(entry);
            // });
        }
    
        fn instance_init(obj: &glib::subclass::InitializingObject<Self>) {
            obj.init_template();
        }
    }
    
    #[glib::derived_properties]
    impl ObjectImpl for CommandPaletteWidgetImp {
        fn constructed(&self) {
            self.parent_constructed();
            let obj = self.obj(); // Wrapper-Instanz
    
            // Initialisiere list_store
            let store = gio::ListStore::new::<CommandPaletteItem>();
            self.list_store.set(store).expect("ListStore konnte nicht gesetzt werden.");
    
            // Initialisiere FilterListModel
            let filter = CustomFilter::new(|item_obj| {
                // Anfänglich alle Items zulassen, Filterung erfolgt durch Neusetzen der Ergebnisse
                // oder durch eine komplexere Filterlogik, die auf einer Eigenschaft des CommandPaletteItem basiert.
                // Für Fuzzy-Suche ist es oft einfacher, die `list_store` direkt neu zu befüllen.
                // Alternativ: item_obj.downcast_ref::<CommandPaletteItem>().unwrap().is_match() (wenn is_match eine Eigenschaft wäre)
                true
            });
            let filter_model = FilterListModel::new(Some(self.list_store.get().unwrap()), Some(filter));
            self.filter_model.set(filter_model).expect("FilterModel konnte nicht gesetzt werden.");
    
            // Initialisiere SelectionModel
            let selection_model = NoSelection::new(Some(self.filter_model.get().unwrap()));
            // Oder SingleSelection, wenn eine Auswahl gewünscht ist:
            // let selection_model = SingleSelection::new(Some(self.filter_model.get().unwrap()));
            self.selection_model.set(selection_model).expect("SelectionModel konnte nicht gesetzt werden.");
    
            // Konfiguriere ListView
            let factory = SignalListItemFactory::new();
            factory.connect_setup(move |_, list_item_gtk| {
                // Erstelle das Widget für ein Listenelement (z.B. eine GtkBox mit Icon und Label)
                // Dies kann auch über eine.ui-Datei für das ListItem geschehen (BuilderListItemFactory)
                let item_widget = gtk::Label::new(None); // Vereinfachtes Beispiel
                item_widget.set_halign(Align::Start);
                list_item_gtk.set_child(Some(&item_widget));
            });
    
            factory.connect_bind(move |_, list_item_gtk| {
                let list_item_gtk = list_item_gtk.downcast_ref::<ListItem>().expect("Muss ListItem sein");
                let command_item = list_item_gtk.item()
                   .and_downcast::<CommandPaletteItem>()
                   .expect("Item muss CommandPaletteItem sein");
    
                let child_widget = list_item_gtk.child().and_downcast::<gtk::Label>().expect("Child muss Label sein");
                // Binde Eigenschaften von command_item an child_widget
                // Hier nur der Name als Beispiel
                child_widget.set_label(&command_item.display_name());
    
                // Hier könnte man auch ein komplexeres Widget binden, das Icon etc. anzeigt
            });
    
            self.results_list_view.set_model(Some(self.selection_model.get().unwrap()));
            self.results_list_view.set_factory(Some(&factory));
    
            // Signale verbinden
            self.search_entry.connect_search_changed(glib::clone!(@weak obj => move |entry| {
                obj.imp().on_search_changed(entry);
            }));
    
            // Aktivierung eines Eintrags in der ListView
            self.results_list_view.connect_activate(glib::clone!(@weak obj => move |list_view, position| {
                obj.imp().on_item_activated(list_view, position);
            }));
    
            // Tastaturnavigation für ListView und SearchEntry
            self.setup_keyboard_navigation();
    
    
            // Lade initial alle verfügbaren Befehle
            self.load_all_commands();
        }
    }
    
    impl WidgetImpl for CommandPaletteWidgetImp {
        fn grab_focus(&self) -> bool {
            self.search_entry.grab_focus()
        }
    }
    impl BoxImpl for CommandPaletteWidgetImp {}
    
    // Private Methoden für CommandPaletteWidgetImp
    impl CommandPaletteWidgetImp {
        fn load_all_commands(&self) {
            let action_handler = self.action_handler.get().expect("ActionHandler nicht verfügbar").clone();
            let mut commands = Vec::new();
    
            // Hier app-weite und fenster-spezifische Aktionen vom MasterActionHandler abrufen
            // und in CommandPaletteItem-Objekte umwandeln.
            // Dies ist eine vereinfachte Darstellung. In einer echten Anwendung
            // müsste MasterActionHandler eine Methode bereitstellen, um beschreibende
            // Informationen über Aktionen zu liefern (Name, Icon, Parameter-Typen etc.).
    
            // Beispielhafte statische Befehle:
            commands.push(Rc::new(CommandPaletteItem::new(
                "quit".to_string(),
                "Anwendung beenden".to_string(),
                Some("application-exit".to_string()),
                "app.quit".to_string(), // Gio Action Name
                None
            )));
            commands.push(Rc::new(CommandPaletteItem::new(
                "about".to_string(),
                "Über diese Anwendung".to_string(),
                Some("help-about".to_string()),
                "app.about".to_string(), // Gio Action Name
                None
            )));
            //... weitere Befehle aus verschiedenen Quellen hinzufügen
    
            *self.all_commands.borrow_mut() = commands;
            self.update_displayed_results(&self.all_commands.borrow());
        }
    
        fn on_search_changed(&self, entry: &SearchEntry) {
            let query = entry.text().to_lowercase();
            if query.is_empty() {
                self.update_displayed_results(&self.all_commands.borrow());
                self.results_scrolled_window.set_visible(false);
            } else {
                let all_cmds = self.all_commands.borrow();
                let filtered_results: Vec<Rc<CommandPaletteItem>> = all_cmds
                   .iter()
                   .filter_map(|item| {
                        // Hier die Fuzzy-Suche anwenden.
                        // Die FuzzySearcher-Klasse sollte eine Methode wie `calculate_score` haben.
                        let score = self.fuzzy_searcher.calculate_score(&item.display_name().to_lowercase(), &query);
                        if score > 0.5 { // Schwellenwert für die Anzeige
                            let mut new_item_data = item.item_data().clone(); // Annahme: CommandPaletteItemData ist Clone
                            new_item_data.score = score;
                            Some(Rc::new(CommandPaletteItem::from_data(new_item_data))) // Erzeuge neues Item mit Score
                        } else {
                            None
                        }
                    })
                   .collect::<Vec<_>>();
    
                // Sortiere nach Score (höchster zuerst)
                // let mut sorted_results = filtered_results;
                // sorted_results.sort_by(|a, b| b.score().partial_cmp(&a.score()).unwrap_or(std::cmp::Ordering::Equal));
                // Da CommandPaletteItem ein GObject ist, kann man nicht einfach sortieren, wenn es schon im Store ist.
                // Besser: Den Store leeren und neu befüllen oder einen SortListModel verwenden.
                // Für dieses Beispiel wird der Store direkt mit gefilterten (aber nicht unbedingt sortierten) Ergebnissen befüllt.
                // Eine robustere Lösung würde einen GtkSortListModel verwenden.
    
                self.update_displayed_results(&filtered_results);
                self.results_scrolled_window.set_visible(!filtered_results.is_empty());
            }
        }
    
        fn update_displayed_results(&self, results: &) {
            let list_store = self.list_store.get().unwrap();
            list_store.remove_all();
            for item_rc in results {
                // Da CommandPaletteItem ein GObject ist, muss es geklont werden, wenn es
                // an mehreren Stellen gehalten wird (z.B. all_commands und list_store).
                // Wenn es nur für den list_store erstellt wird, ist kein Klonen nötig.
                // Hier wird angenommen, dass `results` bereits `Rc<CommandPaletteItem>` enthält.
                // Für `gio::ListStore::append` wird ein Verweis auf das GObject benötigt.
                list_store.append(item_rc.as_ref());
            }
        }
    
        fn on_item_activated(&self, _list_view: &ListView, position: u32) {
            let model = self.filter_model.get().unwrap(); // Oder direkt list_store, wenn kein Filter aktiv
            if let Some(item_obj) = model.item(position) {
                if let Ok(command_item) = item_obj.downcast::<CommandPaletteItem>() {
                    let action_name = command_item.action_name();
                    let param = command_item.action_param_variant(); // Methode, die Option<Variant> zurückgibt
    
                    if let Some(action_handler) = self.action_handler.get() {
                        action_handler.activate_action(&action_name, param.as_ref());
                        self.obj().emit_by_name::<()>("command-executed", &[&action_name, &param]);
                        // Palette ausblenden, typischerweise durch das MainWindow gesteuert
                        if let Some(revealer) = self.obj().parent().and_then(|p| p.downcast::<Revealer>().ok()) {
                             revealer.set_reveal_child(false);
                             let main_window = revealer.ancestor(MainWindow::static_type()).and_then(|a| a.downcast::<MainWindow>().ok());
                             if let Some(mw) = main_window {
                                 mw.set_is_command_palette_visible(false);
                             }
                        } else if let Some(popover) = self.obj().ancestor(gtk::Popover::static_type()).and_then(|p| p.downcast::<gtk::Popover>().ok()){
                            popover.popdown();
                             let main_window = popover.ancestor(MainWindow::static_type()).and_then(|a| a.downcast::<MainWindow>().ok());
                             if let Some(mw) = main_window {
                                 mw.set_is_command_palette_visible(false);
                             }
                        }
                    }
                }
            }
        }
    
        fn setup_keyboard_navigation(&self) {
            let search_entry = self.search_entry.get().unwrap();
            let list_view = self.results_list_view.get().unwrap();
    
            search_entry.add_controller({
                let key_controller = gtk::EventControllerKey::new();
                let list_view_clone = list_view.clone();
                key_controller.connect_key_pressed(move |_, keyval, _, _| {
                    match keyval {
                        gdk::Key::Down => {
                            // Fokus auf ListView setzen und erstes Element auswählen
                            list_view_clone.grab_focus();
                            // Ggf. erstes Element programmatisch auswählen, falls NoSelection verwendet wird
                            // oder die Logik für die Auswahl in SingleSelection anpassen.
                            // Für NoSelection ist dies komplexer, da es keine "aktuelle Auswahl" gibt.
                            // Man könnte den Fokus auf das erste Kind-Widget der ListView setzen.
                            Propagation::Stop
                        }
                        gdk::Key::Escape => {
                            // Palette schließen (durch MainWindow gesteuert)
                             if let Some(revealer) = search_entry.ancestor(gtk::Revealer::static_type()).and_then(|p| p.downcast::<gtk::Revealer>().ok()) {
                                 revealer.set_reveal_child(false);
                                 let main_window = revealer.ancestor(MainWindow::static_type()).and_then(|a| a.downcast::<MainWindow>().ok());
                                 if let Some(mw) = main_window {
                                     mw.set_is_command_palette_visible(false);
                                 }
                             } else if let Some(popover) = search_entry.ancestor(gtk::Popover::static_type()).and_then(|p| p.downcast::<gtk::Popover>().ok()){
                                popover.popdown();
                                 let main_window = popover.ancestor(MainWindow::static_type()).and_then(|a| a.downcast::<MainWindow>().ok());
                                 if let Some(mw) = main_window {
                                     mw.set_is_command_palette_visible(false);
                                 }
                            }
                            Propagation::Stop
                        }
                        _ => Propagation::Proceed,
                    }
                });
                key_controller
            });
    
            // ListView benötigt ebenfalls einen KeyController für Enter/Escape
            list_view.add_controller({
                let key_controller = gtk::EventControllerKey::new();
                let search_entry_clone = search_entry.clone();
                key_controller.connect_key_pressed(move |event_controller_key, keyval, _, _| {
                    let list_view_widget = event_controller_key.widget().unwrap().downcast::<ListView>().unwrap();
                    match keyval {
                        gdk::Key::Return | gdk::Key::KP_Enter => {
                            // Aktiviere das aktuell fokussierte/ausgewählte Element
                            // Dies erfordert, dass SingleSelection verwendet wird und ein Element fokussiert ist.
                            // Die `activate` Signalverbindung oben sollte dies bereits behandeln, wenn
                            // die ListView den Fokus hat und ein Element ausgewählt/aktiviert werden kann.
                            // Hier könnte man explizit die Aktion `list.activate-item` aufrufen.
                            // GtkListView::activate_action ist nicht direkt verfügbar, aber das Signal wird ausgelöst.
                            // Man könnte die aktuelle Cursorposition abfragen und das activate Signal manuell emittieren
                            // oder auf das existierende `activate` Signal vertrauen.
                            Propagation::Stop
                        }
                        gdk::Key::Escape => {
                            search_entry_clone.grab_focus(); // Fokus zurück zum Suchfeld
                            Propagation::Stop
                        }
                        _ => Propagation::Proceed,
                    }
                });
                key_controller
            });
        }
    }
    ```
    
    Die Verwendung von `gio::ListStore` in Kombination mit `gtk::ListView` und `gtk::SignalListItemFactory` ist ein moderner Ansatz in GTK4 zur Darstellung dynamischer Listen.21 `GtkFilterListModel` kann verwendet werden, um die im `ListStore` enthaltenen Elemente basierend auf der Suchanfrage zu filtern, obwohl für eine echte Fuzzy-Suche oft das Neupopulieren des Stores oder ein spezialisierter Filter effektiver ist. Die `fuzzy-search`-Bibliothek 26 bietet Algorithmen wie SymSpell, die hierfür geeignet sind.
    
- **UI-Definition (`command_palette_widget.ui`):**
    
    XML
    
    ```
    <?xml version="1.0" encoding="UTF-8"?>
    <interface>
      <requires lib="gtk" version="4.0"/>
      <template class="MyCommandPaletteWidget" parent="GtkBox">
        <property name="orientation">vertical</property>
        <property name="spacing">6</property>
        <child>
          <object class="GtkSearchEntry" id="search_entry">
            <property name="placeholder-text" translatable="yes">Befehl eingeben...</property>
            <property name="hexpand">true</property>
          </object>
        </child>
        <child>
          <object class="GtkScrolledWindow" id="results_scrolled_window">
            <property name="hscrollbar-policy">never</property>
            <property name="vscrollbar-policy">automatic</property>
            <property name="min-content-height">100</property> <property name="max-content-height">300</property> <property name="vexpand">false</property> <property name="visible">false</property> <child>
              <object class="GtkListView" id="results_list_view">
                <property name="css-classes">results-list</property>
              </object>
            </child>
          </object>
        </child>
      </template>
    </interface>
    ```
    

##### 2.3.4. Eigenschaften

Keine öffentlich exponierten GObject-Eigenschaften. Der interne Zustand wird durch die Felder in `CommandPaletteWidgetImp` verwaltet.

##### 2.3.5. Methoden

- **`pub fn new(action_handler: Rc<MasterActionHandler>) -> Self`** (bereits im Wrapper definiert)
    
    - **Nachbedingungen:** Das Widget ist initialisiert, der `action_handler` ist gespeichert. Die `list_store`, `filter_model`, `selection_model` und `results_list_view` sind konfiguriert. `search_entry` ist mit `on_search_changed` verbunden. `results_list_view` ist mit `on_item_activated` verbunden. Tastaturnavigation ist eingerichtet. Initiale Befehlsliste ist geladen.
- **`pub fn show_palette(&self)`**
    
    - **Signatur:** `pub fn show_palette(&self)`
    - **Zugriffsmodifikator:** `pub`
    - **Nachbedingungen:** Das `search_entry` erhält den Fokus, vorherige Suchanfragen und Ergebnisse werden gelöscht, und die Ergebnisliste wird initial (ggf. leer oder mit allen Befehlen) angezeigt und sichtbar gemacht, falls Ergebnisse vorhanden sind.
    - **Implementierungslogik:**
        1. Hole `imp`.
        2. `imp.search_entry.set_text("")`.
        3. `imp.search_entry.grab_focus()`.
        4. `imp.update_displayed_results(&imp.all_commands.borrow())` (oder leere Liste, je nach gewünschtem Verhalten).
        5. `imp.results_scrolled_window.set_visible(!imp.list_store.get().unwrap().n_items() == 0)`.
- **`pub fn hide_palette(&self)`**
    
    - **Signatur:** `pub fn hide_palette(&self)`
    - **Zugriffsmodifikator:** `pub`
    - **Nachbedingungen:** Die Palette wird ausgeblendet (typischerweise durch das `MainWindow`, das den `Revealer` steuert). Suchfeld wird geleert.
    - **Implementierungslogik:**
        1. Hole `imp`.
        2. `imp.search_entry.set_text("")`.
        3. Die eigentliche Ausblendlogik (z.B. `revealer.set_reveal_child(false)`) sollte vom `MainWindow` gesteuert werden, welches diese Methode aufruft oder auf ein Signal reagiert.
- **`fn on_search_changed(&self, entry: &gtk::SearchEntry)`** (Privat, in `CommandPaletteWidgetImp` implementiert)
    
    - **Implementierungslogik Details:**
        1. Hole den Suchbegriff von `entry.text()`.
        2. Wenn der Suchbegriff leer ist:
            - Zeige alle Befehle oder keine (je nach Designentscheidung). `self.update_displayed_results(&self.all_commands.borrow());`
            - `self.results_scrolled_window.set_visible(!self.list_store.get().unwrap().n_items() == 0);`
            - Beende.
        3. Hole die gecachte Liste aller `CommandPaletteItem`s (`self.all_commands.borrow()`).
        4. Iteriere über `all_commands`:
            - Für jedes Kommando, berechne den Fuzzy-Match-Score mit `self.fuzzy_searcher.calculate_score(&command_item.display_name().to_lowercase(), &query)`. Die `fuzzy-search`-Bibliothek 26 bietet hierfür geeignete Algorithmen.
            - Wenn der Score einen Schwellenwert überschreitet, erstelle ein neues `CommandPaletteItem`-GObject (oder klone das existierende und aktualisiere dessen `score`-Eigenschaft). Füge es zu einer temporären Ergebnisliste hinzu.
        5. Sortiere die temporäre Ergebnisliste nach Score (absteigend).
        6. Aktualisiere `self.list_store` mit den sortierten Ergebnissen mittels `self.update_displayed_results(&sorted_results)`.
        7. `self.results_scrolled_window.set_visible(!sorted_results.is_empty());`
- **`fn on_item_activated(&self, _list_view: &gtk::ListView, position: u32)`** (Privat, in `CommandPaletteWidgetImp` implementiert)
    
    - **Implementierungslogik Details:**
        1. Hole das `CommandPaletteItem`-GObject an der gegebenen `position` aus dem `self.filter_model.get().unwrap()` (oder `self.list_store.get().unwrap()`, falls kein Filtermodell verwendet wird). 27
        2. Extrahiere `action_name` und `action_param` (als `Option<glib::Variant>`) aus dem `CommandPaletteItem`.
        3. Rufe `self.action_handler.get().unwrap().activate_action(&action_name, action_param.as_ref())` auf.
        4. Emittiere das `command_executed`-Signal: `self.obj().emit_by_name::<()>("command-executed", &[&action_name, &action_param]);`.
        5. Veranlasse das Schließen der Palette (z.B. durch ein Signal an `MainWindow` oder direkten Aufruf von `MainWindow::hide_palette`, falls eine Referenz vorhanden ist und dies architektonisch sinnvoll ist. Eine sauberere Methode ist, dass `MainWindow` auf `command-executed` hört und die Palette schließt). Hier wird die Logik aus `setup_keyboard_navigation` für Escape wiederverwendet.

##### 2.3.6. Signale

- **Emittierte Signale:**
    - **Signal Name:** `command-executed`
        - **GObject Name:** `command-executed`
        - **Parameter:**
            - `action_name`: `String` (Rust), `glib::VariantType::STRING` (`glib::Variant`) – Der Name der ausgeführten Aktion.
            - `parameter`: `Option<glib::Variant>` (Rust), `glib::VariantType::VARIANT` (`glib::Variant`) – Der Parameter, mit dem die Aktion ausgeführt wurde (kann `None` sein).
        - **Beschreibung:** Wird emittiert, nachdem ein Befehl aus der Palette erfolgreich ausgewählt und dessen Aktion ausgelöst wurde.
        - **Definition in `ObjectImpl::signals()` (in `CommandPaletteWidgetImp`):**
            
            Rust
            
            ```
            fn signals() -> &'static {
                static SIGNALS: OnceCell<Vec<glib::subclass::Signal>> = OnceCell::new();
                SIGNALS.get_or_init(|| {
                    vec!) // glib::Variant kann None sein
                           .build(),
                    ]
                })
            }
            ```
            

##### 2.3.7. Datenstrukturen

- **`CommandPaletteItem` (GObject)**
    
    - **Datei:** `src/app_shell/command_palette/item.rs`
    - **Wrapper:** `pub struct CommandPaletteItem(ObjectSubclass<imp::CommandPaletteItemImp>)`
    - **Implementierung (`imp`):**
        
        Rust
        
        ```
        // In src/app_shell/command_palette/item/imp.rs
        use gtk::glib::{self, Properties, Variant};
        use gtk::prelude::*;
        use gtk::subclass::prelude::*;
        use std::cell::{Cell, RefCell};
        
        # // Clone für item_data
        #[properties(wrapper_type = super::CommandPaletteItem)]
        pub struct CommandPaletteItemData {
            #
            pub id: RefCell<String>,
            #
            pub display_name: RefCell<String>,
            #[property(get, set, name="icon-name", nick="Icon Name", blurb="Optional icon name for the command")]
            pub icon_name: RefCell<Option<String>>,
            #[property(get, set, name="action-name", nick="Action Name", blurb="gio::Action name to execute")]
            pub action_name: RefCell<String>,
            #[property(get, set, name="action-param", nick="Action Parameter", blurb="Optional parameter for the action")]
            pub action_param: RefCell<Option<Variant>>, // Speichert glib::Variant direkt
            #
            pub score: Cell<f32>,
        }
        
        #
        pub struct CommandPaletteItemImp {
            #[property(get, set)]
            data: RefCell<CommandPaletteItemData>,
        }
        
        
        #[glib::object_subclass]
        impl ObjectSubclass for CommandPaletteItemImp {
            const NAME: &'static str = "MyCommandPaletteItem";
            type Type = super::CommandPaletteItem;
            type ParentType = glib::Object;
        }
        
        // Manuelle Implementierung von ObjectImpl, um Properties von CommandPaletteItemData weiterzuleiten
        // oder #[glib::derived_properties] auf CommandPaletteItemImp verwenden, wenn data direkt Properties hätte.
        // Da data ein RefCell<CommandPaletteItemData> ist und CommandPaletteItemData #[derive(Properties)] hat,
        // benötigen wir eine Brücke oder eine flachere Struktur.
        // Einfacher ist es, die Properties direkt in CommandPaletteItemImp zu definieren, wenn möglich,
        // oder Getter/Setter im Wrapper zu implementieren, die auf `data` zugreifen.
        // Für dieses Beispiel gehen wir davon aus, dass die Properties direkt auf CommandPaletteItemImp definiert werden
        // oder dass der Wrapper entsprechende Zugriffsmethoden bereitstellt.
        // Die #[property] Attribute in CommandPaletteItemData sind für den Fall, dass CommandPaletteItemData selbst
        // als GObject verwendet würde. Hier ist es eingebettet.
        // Eine bessere Struktur wäre, die Felder direkt in CommandPaletteItemImp zu haben und dort #[property] zu verwenden.
        // Wir passen das an, um die Properties direkt im GObject zu haben:
        
        // In src/app_shell/command_palette/item/imp.rs (angepasst)
        //... imports...
        #
        #[properties(wrapper_type = super::CommandPaletteItem)]
        pub struct CommandPaletteItemImp {
            #
            pub id: RefCell<String>,
            #
            pub display_name: RefCell<String>,
            #[property(get, set, name="icon-name", nick="Icon Name", blurb="Optional icon name for the command")]
            pub icon_name: RefCell<Option<String>>,
            #[property(get, set, name="action-name", nick="Action Name", blurb="gio::Action name to execute")]
            pub action_name: RefCell<String>,
            #[property(get, set, name="action-param-variant", nick="Action Parameter Variant", blurb="Optional parameter for the action as GVariant")]
            pub action_param_variant: RefCell<Option<Variant>>,
            #
            pub score: Cell<f32>,
        }
        
        #[glib::object_subclass]
        impl ObjectSubclass for CommandPaletteItemImp {
            const NAME: &'static str = "MyCommandPaletteItem";
            type Type = super::CommandPaletteItem;
            type ParentType = glib::Object;
        }
        
        #[glib::derived_properties]
        impl ObjectImpl for CommandPaletteItemImp {}
        ```
        
    - **Wrapper-Methoden für `CommandPaletteItem` (in `item/mod.rs`):**
        
        Rust
        
        ```
        // In src/app_shell/command_palette/item/mod.rs
        use gtk::glib::{self, Variant};
        use gtk::subclass::prelude::*;
        
        mod imp;
        
        glib::wrapper! {
            pub struct CommandPaletteItem(ObjectSubclass<imp::CommandPaletteItemImp>)
                @extends glib::Object;
        }
        
        impl CommandPaletteItem {
            pub fn new(
                id: String,
                display_name: String,
                icon_name: Option<String>,
                action_name: String,
                action_param: Option<Variant>,
            ) -> Self {
                glib::Object::builder()
                   .property("item-id", &id)
                   .property("display-name", &display_name)
                   .property("icon-name", &icon_name)
                   .property("action-name", &action_name)
                   .property("action-param-variant", &action_param)
                   .build()
            }
            // Getter für die Properties werden durch #[glib::derived_properties] und #[property(get)] generiert.
            // z.B. self.item_id(), self.display_name() etc.
        }
        ```
        
    - Die Verwendung eines dedizierten GObject für Listeneinträge ist notwendig für `gio::ListStore` und ermöglicht saubere Datenbindung und Property-Management.23
- **`FuzzySearchAlgorithm` Enum (in `fuzzy_searcher.rs`):**
    
    Rust
    
    ```
    pub enum FuzzySearchAlgorithm {
        Levenshtein,
        SymSpell, // Von fuzzy-search crate [26]
        // Weitere Algorithmen...
    }
    
    pub struct FuzzySearcher {
        algorithm: FuzzySearchAlgorithm,
        // Ggf. initialisierte Strukturen für den gewählten Algorithmus, z.B. SymSpell Dictionary
    }
    
    impl FuzzySearcher {
        pub fn new(algorithm: FuzzySearchAlgorithm) -> Self {
            // Initialisiere den Sucher basierend auf dem Algorithmus
            Self { algorithm, /*... */ }
        }
    
        pub fn calculate_score(&self, text: &str, query: &str) -> f32 {
            match self.algorithm {
                FuzzySearchAlgorithm::Levenshtein => {
                    let distance = levenshtein::levenshtein(query, text);
                    // Konvertiere Distanz in einen Score (z.B. 1.0 - (distance / max_len))
                    // Je höher der Score, desto besser der Match.
                    // Dies ist eine einfache Heuristik.
                    let max_len = std::cmp::max(text.len(), query.len()) as f32;
                    if max_len == 0.0 { return if query == text {1.0} else {0.0}; }
                    1.0 - (distance as f32 / max_len)
                }
                FuzzySearchAlgorithm::SymSpell => {
                    // Verwendung der fuzzy-search crate [26]
                    // Die Crate gibt typischerweise direkt passende Strings oder Distanzen zurück.
                    // Hier müsste eine Adaption erfolgen, um einen Score zu erhalten.
                    // Beispiel: Wenn ein Match gefunden wird, Score > 0.5, sonst 0.
                    // Die `fuzzy-search` Crate bietet `levenshtein` als Funktion.
                    // Für SymSpell müsste man die `SymSpell::lookup` Methode verwenden und die Ergebnisse interpretieren.
                    // Für dieses Beispiel wird Levenshtein als Fallback verwendet, wenn SymSpell nicht direkt einen Score liefert.
                    let distance = levenshtein::levenshtein(query, text);
                    let max_len = std::cmp::max(text.len(), query.len()) as f32;
                    if max_len == 0.0 { return if query == text {1.0} else {0.0}; }
                    1.0 - (distance as f32 / max_len)
                }
            }
        }
    }
    ```
    
- **Tabelle: `CommandPaletteItem` Eigenschaften**
    

|   |   |   |   |   |
|---|---|---|---|---|
|**Eigenschaftsname (GObject)**|**Rust-Feld (Imp)**|**Rust-Typ**|**Standardwert**|**Beschreibung**|
|`item-id`|`id`|`RefCell<String>`|`""`|Eindeutige ID des Befehlseintrags.|
|`display-name`|`display_name`|`RefCell<String>`|`""`|Für den Benutzer sichtbarer Name des Befehls.|
|`icon-name`|`icon_name`|`RefCell<Option<String>>`|`None`|Optionaler Icon-Name für den Befehl (gemäß Freedesktop Icon Naming Spec).|
|`action-name`|`action_name`|`RefCell<String>`|`""`|Der Name der `gio::Action`, die ausgeführt werden soll.|
|`action-param-variant`|`action_param_variant`|`RefCell<Option<Variant>>`|`None`|Optionaler Parameter für die Aktion, als `glib::Variant`.|
|`score`|`score`|`Cell<f32>`|`0.0`|Score des Fuzzy-Suchalgorithmus für diesen Eintrag (höher ist besser).|

```
Diese Tabelle dokumentiert die Struktur der `CommandPaletteItem`-Objekte, die in der Ergebnisliste angezeigt werden. Sie ist entscheidend für Entwickler, die neue Befehle zur Palette hinzufügen oder das Verhalten der Palette anpassen möchten.
```

##### 2.3.8. UI Layout und Styling

- **Widget-Hierarchie:** (Wie in `command_palette_widget.ui` definiert)
    - `MyCommandPaletteWidget` (`gtk::Box`, vertikal, `spacing=6`)
        - `search_entry` (`gtk::SearchEntry`, `hexpand=true`, `placeholder-text="Befehl eingeben..."`)
        - `results_scrolled_window` (`gtk::ScrolledWindow`, `hscrollbar-policy=never`, `vscrollbar-policy=automatic`, `min-content-height`, `max-content-height`, `vexpand=false`, `visible=false`)
            - `results_list_view` (`gtk::ListView`, CSS-Klasse `results-list`)
- **Styling:**
    - `CommandPaletteWidget`: CSS-Name `command-palette`.
    - `search_entry`: Standard-Styling oder CSS-Klasse `command-palette-search-entry`.
    - `results_list_view`: CSS-Klasse `results-list`.
    - `ListItem`-Widgets (in der Factory erstellt): CSS-Klasse `command-palette-item`. Aktive/Hover-Zustände können über CSS definiert werden. Die Verwendung von `vexpand=false` für das `ScrolledWindow` und `min/max-content-height` stellt sicher, dass die Ergebnisliste nicht den gesamten verfügbaren Platz einnimmt, sondern sich dynamisch bis zu einer maximalen Höhe anpasst.

##### 2.3.9. Exception Handling / Error Types

Fehler bei der Interaktion mit dem `MasterActionHandler` oder bei der D-Bus-Kommunikation (falls Aktionen über D-Bus ausgelöst werden) sollten abgefangen und geloggt werden. UI-Fehler (z.B. fehlerhafte Item-Erstellung) sollten ebenfalls behandelt werden.

##### 2.3.10. Asynchrone Operationen

Die Fuzzy-Suche selbst könnte bei sehr vielen Befehlen potenziell blockierend sein. Falls dies der Fall ist, sollte die Suchlogik in `on_search_changed` in einen separaten Thread oder eine asynchrone Aufgabe ausgelagert werden, die dann die `list_store` über den `AsyncUIManager` aktualisiert. Für die typische Anzahl von Befehlen in einer Palette ist dies jedoch oft nicht notwendig.

##### 2.3.11. Abhängigkeiten

- `MasterActionHandler`: Zum Abrufen verfügbarer Aktionen und zum Auslösen ausgewählter Aktionen.
- `fuzzy-search` Crate (oder eine ähnliche Bibliothek) für die Fuzzy-Suchlogik.26
- `gtk::SearchEntry`, `gtk::ListView`, `gtk::ScrolledWindow`, `gio::ListStore`, `gtk::FilterListModel`, `gtk::SignalListItemFactory`.

##### 2.3.12. Testplan

- **Unit-Tests (`fuzzy_searcher.rs`):**
    - Teste verschiedene Suchanfragen gegen eine bekannte Menge von Befehlsnamen.
    - Überprüfe Korrektheit der Scores und der Sortierung.
    - Teste Randfälle (leere Query, keine Treffer, exakte Treffer).
- **Unit-Tests (`CommandPaletteItem`):**
    - Teste Erstellung und Property-Zugriff.
- **Widget-Tests (`CommandPaletteWidget` mit GTK-Testumgebung):**
    - Teste `show_palette()` und `hide_palette()` (indirekt über `MainWindow`).
    - Simuliere Texteingabe in `search_entry` und überprüfe:
        - Korrekte Filterung und Anzeige der Ergebnisse in `results_list_view`.
        - Korrekte Sortierung der Ergebnisse.
        - Sichtbarkeit des `results_scrolled_window`.
    - Simuliere Item-Aktivierung (`results_list_view.activate`) und überprüfe:
        - Aufruf von `MasterActionHandler::activate_action` mit korrekten Parametern.
        - Emission des `command_executed`-Signals.
        - Schließen der Palette.
    - Teste Tastaturnavigation (Pfeiltasten, Enter, Escape).

#### 2.4. `MasterActionHandler` (Service/Non-Widget GObject)

##### 2.4.1. Übersicht und Zweck

Der `MasterActionHandler` ist ein zentraler Dienst innerhalb der Applikation, der als Registrierungsstelle und Auslöser für globale und fensterspezifische `gio::Action`-Instanzen dient. Er ermöglicht es verschiedenen Komponenten, Aktionen zu definieren und bereitzustellen, ohne dass eine direkte Kopplung zwischen dem Auslöser (z.B. einem Menüpunkt, einem Button oder der `CommandPaletteWidget`) und dem Aktionsempfänger bestehen muss. Dies fördert die Entkopplung und Wiederverwendbarkeit von Aktionen. GTK selbst nutzt `gio::Action` intensiv für Menüs, Tastenkürzel und UI-Elemente, die Aktionen auslösen können.12 Der `MasterActionHandler` stellt eine Abstraktionsebene darüber bereit, um die Verwaltung dieser Aktionen applikationsweit zu vereinfachen.

##### 2.4.2. Dateistruktur

- `src/app_shell/action_handler/mod.rs`: Öffentliches Modul und Wrapper-Definition.
- `src/app_shell/action_handler/imp.rs`: Private Implementierungsdetails und GObject-Subklassifizierung.

##### 2.4.3. GObject-Definition

- **Wrapper-Typ:**
    
    Rust
    
    ```
    // In src/app_shell/action_handler/mod.rs
    use gtk::glib;
    use gtk::gio;
    use gtk::subclass::prelude::*;
    use std::collections::HashMap;
    use std::cell::RefCell;
    use gtk::ApplicationWindow; // Für window_actions
    
    glib::wrapper! {
        pub struct MasterActionHandler(ObjectSubclass<imp::MasterActionHandlerImp>)
            @extends glib::Object;
    }
    
    impl MasterActionHandler {
        pub fn new() -> Self {
            glib::Object::builder().build()
        }
    
        // Wrapper-Methoden für die Logik in imp
        pub fn register_app_action(&self, app: &gtk::Application, action: &impl IsA<gio::Action>) {
            self.imp().register_app_action(app, action);
        }
    
        pub fn register_window_action(&self, window: &ApplicationWindow, action_name_prefix: &str, action: &impl IsA<gio::Action>) {
            self.imp().register_window_action(window, action_name_prefix, action);
        }
    
        pub fn activate_action(&self, active_window: Option<&ApplicationWindow>, action_name: &str, parameter: Option<&glib::Variant>) {
            self.imp().activate_action(active_window, action_name, parameter);
        }
    
        pub fn create_action_entry<F>(
            name: &str,
            parameter_type: Option<&glib::VariantTy>,
            state: Option<&glib::Variant>,
            activate_callback: F,
        ) -> gio::ActionEntry
        where
            F: Fn(&gio::SimpleAction, Option<&glib::Variant>) + 'static,
        {
            // Diese Methode ist statisch oder Teil des Imp, da sie keine Self-Referenz benötigt,
            // wenn sie nur eine ActionEntry erstellt.
            // Wenn sie jedoch Zugriff auf den MasterActionHandler selbst benötigt (z.B. um
            // this.obj() in der Closure zu verwenden), dann muss sie eine &self Referenz haben.
            // Für eine reine Factory-Methode ist static ok.
    
            let action = gio::SimpleAction::new_stateful(name, parameter_type, state.unwrap_or(&Variant::unit()));
            action.connect_activate(move |act, param| {
                activate_callback(act, param);
            });
            // ActionEntry::builder() ist oft einfacher für die direkte Verwendung mit add_action_entries.
            // Diese Methode hier ist eher für die manuelle Erstellung und Registrierung einzelner SimpleActions.
            // Die Anforderung war ActionEntry, daher:
            ActionEntry::builder(name)
               .parameter_type(parameter_type.cloned())
               .state(state.cloned())
               .activate(move |_target_obj: &glib::Object, act: &gio::SimpleAction, param: Option<&Variant>| {
                    // Die Closure-Signatur für ActionEntry::activate ist anders.
                    // Sie erhält das Zielobjekt (App oder Window), die Aktion und den Parameter.
                    // Der Aufruf von activate_callback muss hier angepasst werden.
                    // Da activate_callback hier eine generische Fn ist, die SimpleAction erwartet,
                    // ist die direkte Verwendung in ActionEntry::activate schwierig ohne Anpassung.
                    // Einfacher ist es, SimpleAction zu erstellen und dann add_action zu verwenden.
    
                    // Für ActionEntry wäre es eher so:
                    // activate: move |target_obj, simple_action, param_variant| {
                    //     // Logik hier, die auf target_obj (App/Window), simple_action und param_variant zugreift
                    // }
                    // Die bereitgestellte activate_callback Signatur passt besser zu SimpleAction::connect_activate.
                    // Wir gehen davon aus, dass die ActionEntry dann manuell einer ActionMap hinzugefügt wird.
                    // Oder, die activate_callback wird so angepasst, dass sie mit ActionEntry kompatibel ist.
                })
               .build()
        }
    }
    
    impl Default for MasterActionHandler {
        fn default() -> Self {
            Self::new()
        }
    }
    ```
    
- **Implementierungs-Struktur (`imp`):**
    
    Rust
    
    ```
    // In src/app_shell/action_handler/imp.rs
    use gtk::glib::{self, Variant, VariantTy};
    use gtk::gio::{self, prelude::*, ActionMap, SimpleActionGroup};
    use gtk::subclass::prelude::*;
    use gtk::ApplicationWindow;
    use std::collections::HashMap;
    use std::cell::RefCell;
    use std::rc::Rc; // Für das Signal
    
    #
    pub struct MasterActionHandlerImp {
        // app_actions wird direkt auf gtk::Application gesetzt, daher hier nicht unbedingt nötig.
        // Stattdessen halten wir eine Referenz auf die App, um Aktionen zu setzen.
        app: RefCell<Option<gtk::Application>>,
        window_actions: RefCell<HashMap<String, gio::SimpleActionGroup>>, // Key: Window ID (z.B. GObject Pointer als String oder eine eindeutige ID)
    }
    
    #[glib::object_subclass]
    impl ObjectSubclass for MasterActionHandlerImp {
        const NAME: &'static str = "MyMasterActionHandler";
        type Type = super::MasterActionHandler;
        type ParentType = glib::Object;
    }
    
    impl ObjectImpl for MasterActionHandlerImp {
        fn signals() -> &'static {
            static SIGNALS: OnceCell<Vec<glib::subclass::Signal>> = OnceCell::new();
            SIGNALS.get_or_init(|| {
                vec!) // action_name, success
                       .build(),
                ]
            })
        }
    }
    
    impl MasterActionHandlerImp {
        pub fn register_app_action(&self, app: &gtk::Application, action: &impl IsA<gio::Action>) {
            // Speichere die App-Referenz, falls noch nicht geschehen
            if self.app.borrow().is_none() {
                *self.app.borrow_mut() = Some(app.clone());
            }
            app.add_action(action); // gtk::Application implementiert gio::ActionMap
        }
    
        pub fn register_window_action(
            &self,
            window: &ApplicationWindow,
            action_name_prefix: &str, // z.B. "win"
            action: &impl IsA<gio::Action>,
        ) {
            // Eindeutige ID für das Fenster generieren (z.B. Pfad oder eine interne ID)
            // Für dieses Beispiel verwenden wir den GObject-Pfad, falls verfügbar, oder eine einfache Zählung.
            // Eine robustere Lösung wäre, eine eindeutige ID pro Fenster zu verwalten.
            let window_id = window.path().map_or_else(
    ```
    

|

| format!("{:p}", window.as_ptr()), // Fallback: Pointer-Adresse als ID

|p| p.to_string()

);

````
        let mut window_actions_map = self.window_actions.borrow_mut();
        let action_group = window_actions_map
           .entry(window_id.clone())
           .or_insert_with(SimpleActionGroup::new);
        
        action_group.add_action(action);

        // Stelle sicher, dass die ActionGroup auch tatsächlich am Fenster registriert ist.
        // GTK fügt automatisch eine "win" ActionGroup zu ApplicationWindows hinzu.
        // Wenn wir eine eigene Gruppe verwenden wollen, müssen wir sie einfügen.
        // Für Standard "win." Aktionen:
        if action_name_prefix == "win" {
             window.lookup_action(&action.name().unwrap_or_default().to_string()); // Prüfen ob schon da
             window.add_action(action); // Fügt zur Standard "win" Gruppe hinzu
        } else {
            // Für benutzerdefinierte Gruppen
            if window.lookup_action_group(action_name_prefix).is_none() {
                window.insert_action_group(action_name_prefix, Some(action_group));
            }
        }
    }

    pub fn activate_action(
        &self,
        active_window: Option<&ApplicationWindow>,
        action_name: &str,
        parameter: Option<&Variant>,
    ) {
        let mut success = false;
        // Versuche zuerst, die Aktion im Kontext des aktiven Fensters auszuführen
        if let Some(window) = active_window {
            if let Some(action) = window.lookup_action(action_name) {
                action.activate(parameter);
                success = true;
            }
            // Man könnte auch nach präfixierten Aktionen suchen, z.B. "win.my-action"
            // oder spezifische ActionGroups durchsuchen, die am Fenster registriert sind.
            // Die aktuelle Implementierung von register_window_action fügt Aktionen
            // direkt zur Standard "win" Gruppe des Fensters hinzu oder zu einer benutzerdefinierten Gruppe.
            // `window.lookup_action` sollte also "prefix.action" finden, wenn so registriert.
        }

        // Wenn nicht im Fensterkontext gefunden oder kein aktives Fenster, versuche App-Aktionen
        if!success {
            if let Some(app) = self.app.borrow().as_ref() {
                if let Some(action) = app.lookup_action(action_name) {
                    action.activate(parameter);
                    success = true;
                }
            }
        }

        if!success {
            eprintln!("Aktion '{}' konnte nicht gefunden oder aktiviert werden.", action_name);
        }
        
        self.obj().emit_by_name::<()>("action-activated", &[&action_name, &success]);
    }
}
```
Die Verwaltung von Aktionen ist zentral für GTK-Anwendungen. `gio::Action` und `gio::ActionMap` (implementiert von `gtk::Application` und `gtk::ApplicationWindow`) sind die Kernkomponenten.[12, 30] `gio::SimpleActionGroup` kann verwendet werden, um Aktionen zu bündeln.[31, 32] Der `MasterActionHandler` kapselt die Logik zur Registrierung und Aktivierung dieser Aktionen.
````

##### 2.4.4. Eigenschaften

Keine öffentlich exponierten GObject-Eigenschaften.

##### 2.4.5. Methoden

- **`pub fn new() -> Self`** (bereits im Wrapper definiert)
    
    - **Nachbedingungen:** Ein neuer `MasterActionHandler` wird erstellt. Interne Maps für Fensteraktionen sind initialisiert.
- **`pub fn register_app_action(&self, app: &gtk::Application, action: &impl IsA<gio::Action>)`** (Wrapper für `imp`-Methode)
    
    - **Signatur (Imp):** `pub fn register_app_action(&self, app: &gtk::Application, action: &impl IsA<gio::Action>)`
    - **Parameter:**
        - `app`: `&gtk::Application` – Die Applikationsinstanz.
        - `action`: `&impl IsA<gio::Action>` – Die zu registrierende App-Aktion.
    - **Nachbedingungen:** Die Aktion ist in der `ActionMap` der `gtk::Application` registriert und kann global aufgerufen werden (z.B. mit Präfix "app.").
    - **Implementierungslogik (Imp):**
        1. Rufe `app.add_action(action)` auf. `gtk::Application` implementiert `gio::ActionMap`.30
- **`pub fn register_window_action(&self, window: &ApplicationWindow, action_name_prefix: &str, action: &impl IsA<gio::Action>)`** (Wrapper für `imp`-Methode)
    
    - **Signatur (Imp):** `pub fn register_window_action(&self, window: &ApplicationWindow, action_name_prefix: &str, action: &impl IsA<gio::Action>)`
    - **Parameter:**
        - `window`: `&ApplicationWindow` – Das Fenster, für das die Aktion registriert wird.
        - `action_name_prefix`: `&str` – Ein Präfix für die Aktionsgruppe (z.B. "win").
        - `action`: `&impl IsA<gio::Action>` – Die zu registrierende Fenster-Aktion.
    - **Nachbedingungen:** Die Aktion ist in einer `ActionMap` registriert, die mit dem Fenster assoziiert ist. Sie kann über das Fenster aufgerufen werden (z.B. mit Präfix "win.").
    - **Implementierungslogik (Imp):**
        1. Generiere eine eindeutige ID für das `window` (z.B. `window.path()` oder Pointer-Adresse).
        2. Hole oder erstelle eine `gio::SimpleActionGroup` für diese Fenster-ID aus `self.window_actions`.
        3. Füge `action` zur `SimpleActionGroup` hinzu.
        4. Füge die `SimpleActionGroup` zur `ActionMap` des `window` hinzu, falls sie nicht schon existiert, unter Verwendung des `action_name_prefix`: `window.insert_action_group(action_name_prefix, Some(&action_group));`.31 Wenn der Präfix "win" ist, kann die Aktion direkt zum Fenster hinzugefügt werden, da `ApplicationWindow` bereits eine "win"-Gruppe hat: `window.add_action(action);`.
- **`pub fn activate_action(&self, active_window: Option<&ApplicationWindow>, action_name: &str, parameter: Option<&glib::Variant>)`** (Wrapper für `imp`-Methode)
    
    - **Signatur (Imp):** `pub fn activate_action(&self, active_window: Option<&ApplicationWindow>, action_name: &str, parameter: Option<&glib::Variant>)`
    - **Parameter:**
        - `active_window`: `Option<&ApplicationWindow>` – Das aktuell aktive Fenster, falls vorhanden.
        - `action_name`: `&str` – Der vollständige Name der zu aktivierenden Aktion (z.B. "win.close" oder "app.about").
        - `parameter`: `Option<&glib::Variant>` – Der Parameter für die Aktion.
    - **Nachbedingungen:** Die angeforderte Aktion wird ausgeführt, falls sie gefunden wird. Das `action_activated`-Signal wird emittiert.
    - **Implementierungslogik (Imp):**
        1. `success = false`.
        2. Wenn `active_window` `Some(window)` ist:
            - Versuche, die Aktion `action_name` direkt auf `window` zu finden und zu aktivieren: `window.activate_action(action_name, parameter)`. `gtk::ApplicationWindow` implementiert `gio::ActionGroup` und `gio::ActionMap`.30
            - Wenn erfolgreich, setze `success = true`.
        3. Wenn nicht erfolgreich und `action_name` app-spezifisch ist (z.B. beginnt mit "app.") oder global gesucht werden soll:
            - Hole die `gtk::Application`-Instanz aus `self.app`.
            - Versuche, die Aktion `action_name` auf der Applikation zu finden und zu aktivieren: `app.activate_action(action_name, parameter)`.
            - Wenn erfolgreich, setze `success = true`.
        4. Wenn immer noch nicht erfolgreich, logge einen Fehler.
        5. Emittiere das `action-activated`-Signal mit `action_name` und `success`.
- **`pub fn create_action_entry<F>(name: &str, parameter_type: Option<&glib::VariantTy>, state: Option<&glib::Variant>, activate_callback: F) -> gio::ActionEntry`** (Statische Hilfsmethode oder Methode auf `MasterActionHandlerImp`)
    
    - **Signatur:** Wie im Wrapper definiert. `F: Fn(&gio::SimpleAction, Option<&glib::Variant>) + 'static`
    - **Beschreibung:** Eine Hilfsfunktion zur einfacheren Erstellung von `gio::ActionEntry`-Strukturen, die für `add_action_entries` verwendet werden können. Dies ist besonders nützlich für zustandsbehaftete Aktionen oder Aktionen mit Parametern.12
    - **Implementierungslogik:**
        
        1. Erstelle eine `gio::SimpleAction` mit `gio::SimpleAction::new_stateful(name, parameter_type, state.unwrap_or_else(Variant::unit))`.
        2. Verbinde die `activate_callback` mit dem `activate`-Signal der `SimpleAction`.
        3. Erstelle eine `gio::ActionEntry` manuell, die diese `SimpleAction` repräsentiert, oder passe die Logik an, um `ActionEntry::builder()` zu verwenden, was oft direkter ist.
            
            Rust
            
            ```
            // Verwendung von ActionEntry::builder ist oft prägnanter:
            // ActionEntry::builder(name)
            //    .parameter_type(parameter_type.cloned())
            //    .state(state.cloned())
            //    .activate(move |obj, action_ref, param_variant| {
            //         // Die activate_callback müsste hier aufgerufen werden.
            //         // Die Signatur von activate_callback (Fn(&SimpleAction,...)) passt nicht direkt
            //         // zur Signatur von ActionEntry::activate (Fn(&TargetType, &SimpleAction,...)).
            //         // Es ist einfacher, SimpleActions zu erstellen und sie mit.add_action() hinzuzufügen.
            //         // Diese Hilfsmethode ist daher komplexer als nötig, wenn ActionEntry das Ziel ist.
            //     })
            //    .build()
            // Stattdessen sollte diese Methode vielleicht direkt eine konfigurierte SimpleAction zurückgeben,
            // die dann mit add_action registriert wird.
            // Wenn das Ziel eine ActionEntry ist, muss die Callback-Struktur angepasst werden.
            ```
            
        
        Die gtk-rs Dokumentation 12 zeigt, dass `ActionEntry::builder()` einen `activate`-Callback erwartet, der das Zielobjekt (App oder Window), die Aktion selbst und den Parameter erhält. Die `create_action_entry` Methode müsste entweder eine `SimpleAction` zurückgeben, die dann manuell hinzugefügt wird, oder die `activate_callback` müsste so angepasst werden, dass sie mit der Signatur von `ActionEntry` kompatibel ist. Für die direkte Verwendung mit `add_action_entries` ist `ActionEntry::builder()` der bevorzugte Weg.

##### 2.4.6. Signale

- **Emittierte Signale:**
    - **Signal Name:** `action-activated`
        - **GObject Name:** `action-activated`
        - **Parameter:**
            - `action_name`: `String` (Rust), `glib::VariantType::STRING` – Der Name der Aktion, die versucht wurde zu aktivieren.
            - `success`: `bool` (Rust), `glib::VariantType::BOOLEAN` – Gibt an, ob die Aktivierung erfolgreich war.
        - **Beschreibung:** Wird emittiert, nachdem versucht wurde, eine Aktion zu aktivieren.
        - **Definition in `ObjectImpl::signals()` (in `MasterActionHandlerImp`):**
            
            Rust
            
            ```
            // Bereits in imp.rs definiert
            // static SIGNALS: OnceCell<Vec<glib::subclass::Signal>> = OnceCell::new();
            // SIGNALS.get_or_init(|| {
            //     vec!)
            //            .build(),
            //     ]
            // })
            ```
            

##### 2.4.7. Datenstrukturen

Keine spezifischen öffentlichen Datenstrukturen, außer den intern verwendeten `HashMap` für fensterspezifische Aktionsgruppen.

##### 2.4.11. Abhängigkeiten

- `gtk::Application`, `gtk::ApplicationWindow`
- `gtk::gio` (insbesondere `Action`, `SimpleAction`, `ActionMap`, `SimpleActionGroup`, `ActionEntry`)
- `gtk::glib` (insbesondere `Variant`, `VariantTy`)

##### 2.4.12. Testplan

- **Unit-Tests:**
    - Teste `register_app_action`: Überprüfe, ob die Aktion korrekt zur `gtk::Application` hinzugefügt wird.
    - Teste `register_window_action`: Überprüfe, ob die Aktion korrekt zur `ActionGroup` des Fensters hinzugefügt wird.
    - Teste `activate_action`:
        - Aktivierung einer App-Aktion.
        - Aktivierung einer Fenster-Aktion (mit und ohne aktives Fenster).
        - Aktivierung einer nicht existierenden Aktion.
        - Korrekte Parameterübergabe.
        - Emission des `action-activated`-Signals mit korrekten Werten.
    - Teste `create_action_entry` (falls als Factory für `ActionEntry` implementiert): Korrekte Erstellung der `ActionEntry`.

#### 2.5. `SettingsService` (Service/Non-Widget GObject)

##### 2.5.1. Übersicht und Zweck

Der `SettingsService` dient als zentraler Zugriffspunkt für anwendungsspezifische Einstellungen, die persistent gespeichert werden. Er kapselt die Funktionalität von `gio::Settings` und stellt eine typsichere API zum Lesen, Schreiben und Überwachen von Einstellungen bereit. Dies ermöglicht eine saubere Trennung der Einstellungsverwaltung von anderen UI-Komponenten und erleichtert das Testen. Die Einstellungen werden typischerweise in einem GSchema XML-File definiert, das bei der Installation der Anwendung im System registriert wird.33

##### 2.5.2. Dateistruktur

- `src/app_shell/settings_service/mod.rs`: Öffentliches Modul und Wrapper-Definition.
- `src/app_shell/settings_service/imp.rs`: Private Implementierungsdetails und GObject-Subklassifizierung.
- `resources/schemas/org.example.myapp.gschema.xml`: GSettings-Schemadatei (Beispielname).

##### 2.5.3. GObject-Definition

- **Wrapper-Typ:**
    
    Rust
    
    ```
    // In src/app_shell/settings_service/mod.rs
    use gtk::glib;
    use gtk::gio;
    use gtk::subclass::prelude::*;
    
    glib::wrapper! {
        pub struct SettingsService(ObjectSubclass<imp::SettingsServiceImp>)
            @extends glib::Object;
    }
    
    impl SettingsService {
        pub fn new(schema_id: &str) -> Result<Self, glib::Error> {
            let service: Self = glib::Object::builder().build();
            service.imp().init_settings(schema_id)?;
            Ok(service)
        }
    
        // Wrapper für get_string, get_bool, etc.
        pub fn get_string(&self, key: &str) -> String {
            self.imp().settings.get().expect("Settings nicht initialisiert").string(key)
        }
        pub fn get_boolean(&self, key: &str) -> bool {
            self.imp().settings.get().expect("Settings nicht initialisiert").boolean(key)
        }
        pub fn get_int(&self, key: &str) -> i32 {
            self.imp().settings.get().expect("Settings nicht initialisiert").int(key)
        }
    
        // Wrapper für set_string, set_bool, etc.
        pub fn set_string(&self, key: &str, value: &str) -> Result<(), glib::Error> {
            self.imp().settings.get().expect("Settings nicht initialisiert").set_string(key, value)
        }
        pub fn set_boolean(&self, key: &str, value: bool) -> Result<(), glib::Error> {
            self.imp().settings.get().expect("Settings nicht initialisiert").set_boolean(key, value)
        }
        pub fn set_int(&self, key: &str, value: i32) -> Result<(), glib::Error> {
            self.imp().settings.get().expect("Settings nicht initialisiert").set_int(key, value)
        }
    
        pub fn bind_property<P: IsA<glib::Object>>(
            &self,
            key: &str,
            object: &P,
            property_name: &str,
            flags: gio::SettingsBindFlags,
        ) {
            self.imp().settings.get().expect("Settings nicht initialisiert").bind(key, object, property_name)
               .flags(flags)
               .build();
        }
    
        pub fn connect_changed<F: Fn(&Self, &str) + 'static + Clone>(
            &self,
            key: Option<&str>,
            callback: F,
        ) -> glib::SignalHandlerId {
            let self_clone = self.clone();
            self.imp().settings.get().expect("Settings nicht initialisiert").connect_changed(key, move |_, changed_key| {
                callback(&self_clone, changed_key);
                self_clone.emit_by_name::<()>("setting-changed", &[&changed_key]);
            })
        }
    }
    ```
    
- **Implementierungs-Struktur (`imp`):**
    
    Rust
    
    ```
    // In src/app_shell/settings_service/imp.rs
    use gtk::glib;
    use gtk::gio::{self, prelude::SettingsExtManual};
    use gtk::subclass::prelude::*;
    use std::cell::OnceCell;
    
    
    #
    pub struct SettingsServiceImp {
        settings: OnceCell<gio::Settings>,
    }
    
    #[glib::object_subclass]
    impl ObjectSubclass for SettingsServiceImp {
        const NAME: &'static str = "MySettingsService";
        type Type = super::SettingsService;
        type ParentType = glib::Object;
    }
    
    impl ObjectImpl for SettingsServiceImp {
        fn signals() -> &'static {
            static SIGNALS: OnceCell<Vec<glib::subclass::Signal>> = OnceCell::new();
            SIGNALS.get_or_init(|| {
                vec!) // key
                       .build(),
                ]
            })
        }
    }
    
    impl SettingsServiceImp {
        pub fn init_settings(&self, schema_id: &str) -> Result<(), glib::Error> {
            match gio::Settings::new(schema_id) {
                Ok(s) => {
                    self.settings.set(s).map_err(|_| glib::Error::new(glib::FileError::Failed, "Settings bereits initialisiert"))?;
                    Ok(())
                }
                Err(e) => Err(e),
            }
        }
    }
    ```
    
    Die `gio::Settings`-Instanz wird in `OnceCell` gespeichert, um sicherzustellen, dass sie nur einmal initialisiert wird. Die `init_settings`-Methode wird vom Wrapper `new` aufgerufen.
    
- **GSettings Schema (`org.example.myapp.gschema.xml`):**
    
    XML
    
    ```
    <schemalist>
      <schema id="org.example.myapp" path="/org/example/myapp/">
        <key name="window-width" type="i">
          <default>1024</default>
          <summary>Main window width</summary>
          <description>The last saved width of the main application window.</description>
        </key>
        <key name="window-height" type="i">
          <default>768</default>
          <summary>Main window height</summary>
          <description>The last saved height of the main application window.</description>
        </key>
        <key name="dark-mode" type="b">
          <default>false</default>
          <summary>Enable dark mode</summary>
          <description>Whether the application should use a dark theme variant.</description>
        </key>
        <key name="last-opened-file" type="s">
          <default>''</default>
          <summary>Last opened file</summary>
          <description>Path to the last successfully opened file.</description>
        </key>
        </schema>
    </schemalist>
    ```
    
    Diese Datei muss während des Build-Prozesses kompiliert (mit `glib-compile-schemas`) und an einem Ort installiert werden, an dem GSettings sie finden kann (z.B. `$XDG_DATA_DIRS/glib-2.0/schemas/`).33
    

##### 2.5.4. Eigenschaften

Keine öffentlich exponierten GObject-Eigenschaften für den `SettingsService` selbst. Die Einstellungen sind über die get/set-Methoden zugänglich.

##### 2.5.5. Methoden

- **`pub fn new(schema_id: &str) -> Result<Self, glib::Error>`** (bereits im Wrapper definiert)
    
    - **Nachbedingungen:** Der `SettingsService` ist initialisiert und hat das GSettings-Schema mit der gegebenen `schema_id` geladen. Gibt `Err` zurück, falls das Schema nicht geladen werden konnte.
- **`pub fn get_string(&self, key: &str) -> String`** (Wrapper für `gio::Settings::string`)
    
    - **Parameter:** `key: &str` – Der Schlüssel der Einstellung.
    - **Rückgabewert:** `String` – Der Wert der Einstellung.
    - **Vorbedingungen:** `key` muss im Schema als String-Typ definiert sein.
    - **Fehlerbehandlung:** Panik, wenn der Schlüssel nicht existiert oder einen anderen Typ hat.
- **`pub fn get_boolean(&self, key: &str) -> bool`** (Wrapper für `gio::Settings::boolean`)
    
    - **Parameter:** `key: &str` – Der Schlüssel der Einstellung.
    - **Rückgabewert:** `bool` – Der Wert der Einstellung.
    - **Vorbedingungen:** `key` muss im Schema als Boolean-Typ definiert sein.
- **`pub fn get_int(&self, key: &str) -> i32`** (Wrapper für `gio::Settings::int`)
    
    - **Parameter:** `key: &str` – Der Schlüssel der Einstellung.
    - **Rückgabewert:** `i32` – Der Wert der Einstellung.
    - **Vorbedingungen:** `key` muss im Schema als Integer-Typ definiert sein.
- **`pub fn set_string(&self, key: &str, value: &str) -> Result<(), glib::Error>`** (Wrapper für `gio::Settings::set_string`)
    
    - **Parameter:**
        - `key: &str` – Der Schlüssel der Einstellung.
        - `value: &str` – Der zu setzende Wert.
    - **Rückgabewert:** `Result<(), glib::Error>` – `Ok(())` bei Erfolg, sonst `Err`.
    - **Vorbedingungen:** `key` muss im Schema als String-Typ definiert sein.
- **`pub fn set_boolean(&self, key: &str, value: bool) -> Result<(), glib::Error>`** (Wrapper für `gio::Settings::set_boolean`)
    
- **`pub fn set_int(&self, key: &str, value: i32) -> Result<(), glib::Error>`** (Wrapper für `gio::Settings::set_int`)
    
- **`pub fn bind_property<W: IsA<glib::Object>>(&self, key: &str, object: &W, property_name: &str, flags: gio::SettingsBindFlags)`** (Wrapper für `gio::Settings::bind`)
    
    - **Parameter:**
        - `key: &str` – Der GSettings-Schlüssel.
        - `object: &W` – Das GObject, dessen Eigenschaft gebunden werden soll.
        - `property_name: &str` – Der Name der GObject-Eigenschaft.
        - `flags: gio::SettingsBindFlags` – Flags zur Steuerung der Bindung (z.B. `DEFAULT`, `BIDIRECTIONAL`).
    - **Nachbedingungen:** Die GObject-Eigenschaft ist an die GSetting gebunden. Änderungen werden synchronisiert gemäß den `flags`.33
- **`pub fn connect_changed<F: Fn(&Self, &str) + 'static + Clone>(&self, key: Option<&str>, callback: F) -> glib::SignalHandlerId`** (Wrapper für `gio::Settings::connect_changed`)
    
    - **Parameter:**
        - `key: Option<&str>` – Der zu überwachende Schlüssel. `None` für alle Schlüssel.
        - `callback: F` – Die Funktion, die bei Änderungen aufgerufen wird. Erhält den `SettingsService` und den geänderten Schlüssel als Parameter.
    - **Rückgabewert:** `glib::SignalHandlerId` – Zur möglichen Trennung der Verbindung.
    - **Nachbedingungen:** Die `callback`-Funktion wird aufgerufen, wenn sich der spezifizierte Schlüssel (oder ein beliebiger Schlüssel, falls `key` `None` ist) ändert. Das `setting-changed`-Signal des `SettingsService` wird ebenfalls emittiert.

##### 2.5.6. Signale

- **Emittierte Signale:**
    - **Signal Name:** `setting-changed`
        - **GObject Name:** `setting-changed`
        - **Parameter:**
            - `key`: `String` (Rust), `glib::VariantType::STRING` – Der Name des geänderten Schlüssels.
        - **Beschreibung:** Wird emittiert, nachdem sich eine Einstellung geändert hat. Dies ist ein Wrapper um das `changed`-Signal von `gio::Settings`.
        - **Definition in `ObjectImpl::signals()` (in `SettingsServiceImp`):**
            
            Rust
            
            ```
            // Bereits in imp.rs definiert
            // static SIGNALS: OnceCell<Vec<glib::subclass::Signal>> = OnceCell::new();
            // SIGNALS.get_or_init(|| {
            //     vec!)
            //            .build(),
            //     ]
            // })
            ```
            
- **Behandelte Signale:**
    - `gio::Settings::changed`: Intern verbunden, um das `setting-changed`-Signal des `SettingsService` auszulösen.

##### 2.5.7. Datenstrukturen

- **Tabelle: GSettings Schema (`org.example.myapp.gschema.xml`)**

|   |   |   |   |   |
|---|---|---|---|---|
|**Schlüsselname**|**GVariant Typ**|**Standardwert**|**Zusammenfassung**|**Beschreibung**|
|`window-width`|`i`|`1024`|Fensterbreite|Die zuletzt gespeicherte Breite des Hauptfensters.|
|`window-height`|`i`|`768`|Fensterhöhe|Die zuletzt gespeicherte Höhe des Hauptfensters.|
|`dark-mode`|`b`|`false`|Dunkelmodus aktivieren|Ob die Anwendung eine dunkle Theme-Variante verwenden soll.|
|`last-opened-file`|`s`|`''`|Zuletzt geöffnete Datei|Pfad zur zuletzt erfolgreich geöffneten Datei.|
|`sidebar-visible`|`b`|`true`|Seitenleiste sichtbar|Ob die Navigationsseitenleiste standardmäßig sichtbar ist.|
|`command-palette-hotkey`|`s`|`<Primary>P`|Hotkey Befehlspalette|Tastenkombination zum Öffnen der Befehlspalette.|

```
Diese Tabelle ist essentiell, da sie die persistenten Einstellungen der Anwendung definiert.
```

##### 2.5.11. Abhängigkeiten

- `gtk::gio` (insbesondere `Settings`, `SettingsBindFlags`)
- `gtk::glib`

##### 2.5.12. Testplan

- **Unit-Tests:**
    - Teste `new()`: Erfolgreiches Laden eines Test-Schemas, Fehlerbehandlung bei ungültigem Schema-ID.
    - Teste `get_` und `set_` Methoden für jeden Typ (String, bool, i32): Korrektes Lesen und Schreiben von Werten.
    - Teste `bind_property`: Bindung an eine Eigenschaft eines Mock-GObjects, Überprüfung der Synchronisation in beide Richtungen (falls `BIDIRECTIONAL`).
    - Teste `connect_changed`: Registrierung eines Callbacks, Überprüfung, ob der Callback und das `setting-changed`-Signal bei Wertänderungen ausgelöst werden.
- **Integrationstests:** (Erfordert eine Testumgebung, in der GSettings funktionieren)
    - Überprüfe, ob Änderungen an Einstellungen persistent sind nach Neustart der Test-Applikation.

#### 2.6. `NotificationService` (Service/Non-Widget GObject)

##### 2.6.1. Übersicht und Zweck

Der `NotificationService` ist verantwortlich für die Anzeige von Benachrichtigungen innerhalb der Applikation und optional für die Interaktion mit dem systemweiten Freedesktop-Benachrichtigungsdienst. Er unterscheidet zwischen In-App-Benachrichtigungen (z.B. als `gtk::Popover` 34 oder über `gtk4-layer-shell` 37 für benutzerdefinierte Popups) und Desktop-Benachrichtigungen. In-App-Benachrichtigungen bieten mehr Kontrolle über Aussehen und Verhalten im Kontext der Anwendung, während Desktop-Benachrichtigungen die Konsistenz mit der Desktop-Umgebung des Benutzers wahren.39

##### 2.6.2. Dateistruktur

- `src/app_shell/notification_service/mod.rs`: Öffentliches Modul und Wrapper-Definition.
- `src/app_shell/notification_service/imp.rs`: Private Implementierungsdetails.
- `src/app_shell/notification_service/desktop_notifier.rs`: (Optional) Modul für die Interaktion mit `org.freedesktop.Notifications` via D-Bus.

##### 2.6.3. GObject-Definition

- **Wrapper-Typ:**
    
    Rust
    
    ```
    // In src/app_shell/notification_service/mod.rs
    use gtk::glib;
    use gtk::gio; // für Cancellable
    use gtk::subclass::prelude::*;
    use gtk::{Widget, Popover}; // Für In-App Notifikationen
    use std::cell::RefCell;
    use std::collections::VecDeque;
    use zbus; // Für Desktop-Notifikationen
    
    // Enum für interne Verwendung im Service
    #
    pub enum NotificationLevel {
        Info,
        Warning,
        Error,
        Success, // Hinzugefügt für positive Rückmeldungen
    }
    
    #
    pub enum DesktopNotificationUrgency {
        Low,
        Normal,
        Critical,
    }
    
    glib::wrapper! {
        pub struct NotificationService(ObjectSubclass<imp::NotificationServiceImp>)
            @extends glib::Object;
    }
    
    impl NotificationService {
        pub fn new(app_name: String, app_icon: String) -> Self {
            let service: Self = glib::Object::builder().build();
            service.imp().init_desktop_notifier(app_name, app_icon);
            service
        }
    
        pub fn show_in_app_notification(
            &self,
            parent_widget: &impl IsA<Widget>,
            message: &str,
            level: NotificationLevel,
            autohide_ms: Option<u32>,
        ) {
            self.imp().show_in_app_notification(parent_widget, message, level, autohide_ms);
        }
    
        pub async fn show_desktop_notification(
            &self,
            summary: &str,
            body: &str,
            icon: Option<&str>,
            actions: Option<&[(&str, &str)]>, // Vec aus (action_key, display_name)
            urgency: Option<DesktopNotificationUrgency>,
            timeout_ms: Option<i32>, // Hinzugefügt für Timeout
        ) -> Result<u32, zbus::Error> {
            self.imp().show_desktop_notification(summary, body, icon, actions, urgency, timeout_ms).await
        }
    }
    ```
    
- **Implementierungs-Struktur (`imp`):**
    
    Rust
    
    ```
    // In src/app_shell/notification_service/imp.rs
    use gtk::glib;
    use gtk::gio;
    use gtk::prelude::*;
    use gtk::subclass::prelude::*;
    use gtk::{Widget, Popover, Box as GtkBox, Label, Image, Align, Orientation};
    use std::cell::RefCell;
    use std::collections::VecDeque;
    use zbus::Connection; // Für Desktop-Notifikationen
    use once_cell::sync::Lazy; // Für statische Connection
    
    use super::{NotificationLevel, DesktopNotificationUrgency}; // Importiere die Enums
    
    const DEFAULT_IN_APP_AUTOHIDE_MS: u32 = 5000; // 5 Sekunden
    
    // Statische D-Bus Verbindung für Desktop-Notifikationen
    static DBUS_CONNECTION: Lazy<Result<Connection, zbus::Error>> = Lazy::new(Connection::session);
    
    
    #
    pub struct NotificationServiceImp {
        active_popovers: RefCell<VecDeque<Popover>>, // Verwende VecDeque als einfache Queue
        app_name_for_desktop: RefCell<
    ```


# Implementierungsleitfaden UI-Schicht NovaDE: Modul 2 - `PanelWidget`

## 1. Einleitung und Zielsetzung

### 1.1. Zweck dieses Dokuments

Dieses Dokument stellt Teil 2 einer Serie von 8 bis 16 Modulspezifikationen für die Implementierung der Benutzeroberflächenschicht (UI-Schicht) der Linux-Desktop-Umgebung "NovaDE" dar. Es definiert die Ultra-Feinspezifikationen und hochdetaillierten Implementierungspläne für das Modul `PanelWidget`. Ziel ist es, einen lückenlosen Entwickler-Implementierungsleitfaden bereitzustellen, der so detailliert ist, dass Entwickler ihn direkt zur Implementierung verwenden können, ohne eigenständige Designentscheidungen treffen oder Kernlogiken und Algorithmen selbst entwerfen zu müssen. Alle relevanten Aspekte wurden recherchiert, entschieden und werden hier präzise spezifiziert.

### 1.2. Geltungsbereich: `PanelWidget` als Kernkomponente der `ui::shell`

Das `PanelWidget` ist eine zentrale Komponente innerhalb des `ui::shell`-Moduls von NovaDE. Es repräsentiert die Hauptkontroll- und Systemleiste(n) des Desktops. Dieses Dokument umfasst die vollständige Spezifikation des `PanelWidget`, einschließlich seiner internen Struktur, seiner öffentlichen Schnittstelle (Eigenschaften, Signale, Methoden), seines Verhaltens, seines Erscheinungsbilds (Layout und Theming) sowie seiner Interaktion mit anderen Modulen und Schichten von NovaDE. Die Spezifikationen für die einzelnen Elemente, die _innerhalb_ des Panels angezeigt werden (z.B. Uhr, System-Tray, AppMenu-Button), sind Gegenstand separater Moduldokumentationen; dieses Dokument definiert jedoch die Schnittstellen und Mechanismen für deren Integration in das `PanelWidget`.

### 1.3. Bezug zur Gesamtarchitektur von NovaDE

NovaDE basiert auf einer strengen Vier-Schichten-Architektur (Kern, Domäne, System, Benutzeroberfläche). Das `PanelWidget` ist integraler Bestandteil der **Benutzeroberflächenschicht**. Es interagiert mit:

- Anderen UI-Modulen innerhalb von `ui::shell` (z.B. `QuickSettingsPanelWidget`, `NotificationCenterPanelWidget`) und `ui::widgets`.
- Der **Domänenschicht**, insbesondere `domain::theming` für das Erscheinungsbild und `domain::global_settings_and_state_management` für Konfigurationen.
- Indirekt mit der **Systemschicht**, beispielsweise wenn im Panel gehostete Indikator-Widgets Daten von Systemdiensten (via D-Bus) anzeigen.

Die Implementierung des `PanelWidget` muss die in der "Technischen Gesamtspezifikation & Richtlinien" von NovaDE festgelegten Prinzipien der Modularität, losen Kopplung und hohen Kohäsion strikt einhalten.

## 2. Modul `ui::shell::panel_widget` - Spezifikation und Implementierungsplan

### 2.1. Verantwortlichkeiten und Features des `PanelWidget`

Das `PanelWidget` ist verantwortlich für:

- Die Bereitstellung einer horizontalen (oder potenziell vertikalen, obwohl initial nicht primär gefordert) Leiste am Bildschirmrand.
- Das Hosting und die Anordnung von verschiedenen UI-Elementen wie Anwendungsmenü-Button, Workspace-Indikator, Uhr, System-Tray, Schnelleinstellungs-Button, Benachrichtigungszentrum-Button sowie diversen Statusindikatoren (Netzwerk, Energie, Audio).
- Die Implementierung eines "eleganten Leisten-Designs mit optionalem Leuchtakzent", dessen Farbe sich dynamisch an das System-Theme anpasst.
- Die Konfigurierbarkeit hinsichtlich Position (z.B. oben, unten am Bildschirm) und Höhe.
- Die Reaktion auf Theme-Änderungen (Hell/Dunkel-Modus, Akzentfarben) aus `domain::theming`.

Es ist _nicht_ verantwortlich für die Implementierung der Logik der einzelnen gehosteten Elemente (z.B. das Anzeigen der Schnelleinstellungen selbst), sondern nur für deren Integration und die Bereitstellung der Auslöser (z.B. Buttons).

### 2.2. Dateistruktur und Modul-Setup

#### 2.2.1. Verzeichnisstruktur

Die empfohlene Verzeichnisstruktur für das `PanelWidget`-Modul innerhalb des `ui::shell`-Verzeichnisses ist wie folgt:

```
nova_de/
└── src/
    └── ui/
        └── shell/
            ├── panel_widget/
            │   ├── mod.rs         // Öffentliche API des PanelWidget, Wrapper-Definition
            │   ├── imp.rs         // Interne Implementierung (ObjectSubclass)
            │   ├── panel_widget.ui // GTK Composite Template UI-Definition
            │   ├── panel_widget.css // Modulspezifische CSS-Stile (optional)
            │   └── error.rs       // Modulspezifische Fehlerdefinitionen
            └──...                // Andere Shell-Module
```

Diese Struktur folgt etablierten Konventionen in der gtk-rs Community, wie sie beispielsweise in Dokumentationen und Beispielen zu finden sind.1 Eine solche Standardisierung fördert die Lesbarkeit und Wartbarkeit des Codes, da sich Entwickler, die mit GTK4 und Rust vertraut sind, schneller zurechtfinden. Die Trennung der öffentlichen API (`mod.rs`) von der internen `ObjectSubclass`-Implementierung (`imp.rs`) ist ein gängiges Muster, das die Einarbeitungszeit für neue Projektmitglieder reduziert.

#### 2.2.2. `Cargo.toml` Konfiguration

Die relevanten Abhängigkeiten für das `PanelWidget` und die UI-Schicht im Allgemeinen sind in der `Cargo.toml` des `nova_de_ui`-Crates (oder des entsprechenden Sub-Crates) zu deklarieren:

Ini, TOML

```
[dependencies]
gtk = { package = "gtk4", version = "0.8", features = ["v4_10"] } # Aktuellste stabile Version, v4_10 für PopoverMenuFlags, etc.
gdk = { package = "gdk4", version = "0.8" }
glib = { package = "glib", version = "0.19" } # Aktuellste stabile Version
gio = { package = "gio", version = "0.19" }
tokio = { version = "1", features = ["full"] } # Für asynchrone Operationen, falls benötigt
tracing = "0.1"
thiserror = "1.0"

# Workspace-interne Abhängigkeiten (Beispiele, genaue Pfade anpassen)
# nova_de_core = { path = "../../../core" }
# nova_de_domain_theming = { path = "../../../domain/theming" }
```

Die Spezifikation von Feature-Flags für GTK, wie z.B. `features = ["v4_10"]` 3, ist entscheidend, um Zugriff auf neuere API-Elemente zu erhalten. Diese können für bestimmte Funktionalitäten, wie erweiterte Optionen für `GtkPopoverMenu` 4 (relevant für im Panel getriggerte Elemente wie das Schnelleinstellungs-Panel), erforderlich sein. Eine frühzeitige Berücksichtigung stellt sicher, dass die notwendigen APIs verfügbar sind.

### 2.3. `PanelWidget`: Rust Struct-Definition und Initialisierung

#### 2.3.1. Definition des `PanelWidget` Structs (Verwendung von `gtk::glib::wrapper!`, `CompositeTemplate`)

Das `PanelWidget` wird als benutzerdefiniertes GTK-Widget implementiert. Die öffentliche Struktur wird in `panel_widget/mod.rs` definiert:

Rust

```
use gtk::glib;
use gtk::subclass::prelude::*;

glib::wrapper! {
    pub struct PanelWidget(ObjectSubclass<imp::PanelWidget>)
        @extends gtk::Widget, gtk::Box; // gtk::Box als Basis für einfaches Layout der Kinder
}

impl PanelWidget {
    pub fn new() -> Self {
        glib::Object::new(&) // Erzeugt eine neue Instanz des PanelWidget
    }
}

// Enum zur Definition der Panel-Position (Beispiel, kann auch globaler sein)
#
#
pub enum PanelPosition {
    #
    Top,
    #
    Bottom,
    // Ggf. Left, Right für vertikale Panels später
}

impl Default for PanelPosition {
    fn default() -> Self {
        PanelPosition::Top
    }
}

// Enum zur Definition der Bereiche für Kind-Widgets
#
pub enum PanelChildrenArea {
    Left,
    Center,
    Right,
}
```

Die Entscheidung, von `gtk::Box` abzuleiten, ermöglicht eine unkomplizierte Anordnung von Kind-Widgets in horizontalen oder vertikalen Segmenten (links, zentriert, rechts). Für die typische Struktur einer Panelleiste ist `gtk::Box` oft ausreichend und einfacher zu handhaben als beispielsweise `gtk::Grid`, welches für komplexere, gitterbasierte Layouts vorgesehen ist.

#### 2.3.2. Implementierung von `imp::PanelWidget` (`ObjectSubclass`, `WidgetImpl`, `BoxImpl` etc.)

Die interne Implementierung des `PanelWidget` erfolgt in `panel_widget/imp.rs`:

Rust

```
use gtk::glib::{self, subclass::InitializingObject, Cell, RefCell};
use gtk::subclass::prelude::*;
use gtk::{gio, gdk, CompositeTemplate, Box as GtkBox}; // GtkBox hier importieren für Klarheit

use super::PanelPosition; // PanelPosition aus mod.rs importieren

#
#[template(resource = "/org/nova_de/ui/shell/panel_widget.ui")] // Pfad zur.ui Datei
pub struct PanelWidget {
    #[template_child]
    pub(super) left_box: TemplateChild<GtkBox>,
    #[template_child]
    pub(super) center_box: TemplateChild<GtkBox>,
    #[template_child]
    pub(super) right_box: TemplateChild<GtkBox>,

    // Interne Zustandsvariablen für GObject-Properties
    #[property(get, set, explicit_notify)]
    position: Cell<PanelPosition>,
    #[property[10]] // Beispiel für Property-Attribute
    height_request_prop: Cell<i32>, // Name mit Suffix _prop zur Unterscheidung von Widget-Property
    #[property(get, set, explicit_notify)]
    auto_hide: Cell<bool>,
    #[property(get, set, explicit_notify)]
    show_glow_accent: Cell<bool>,
    #[property(get, /* no set from outside */ explicit_notify)]
    glow_accent_color_prop: RefCell<Option<gdk::RGBA>>,
    #[property(get, set, explicit_notify)] // Init-only wird in Setter-Logik behandelt
    monitor_index: Cell<i32>,

    // Weitere interne Zustandsvariablen
    // settings: RefCell<Option<gio::Settings>>, // Für GSettings
    // children_configs: RefCell<Vec<PanelChildConfig>>, // Für dynamische Kinder-Konfiguration
}

#[glib::object_subclass]
impl ObjectSubclass for PanelWidget {
    const NAME: &'static str = "NovaDEPanelWidget";
    type Type = super::PanelWidget;
    type ParentType = GtkBox; // Muss mit @extends oben übereinstimmen

    fn class_init(klass: &mut Self::Class) {
        klass.bind_template();
        // CSS laden, falls modulspezifisch vorhanden
        // Self::Type::load_css(); // Methode muss in PanelWidget (mod.rs) definiert werden

        // GObject Properties manuell installieren, wenn #[property] nicht alles abdeckt
        // oder man mehr Kontrolle braucht. Hier durch #[derive(Properties)] und #[property] abgedeckt.
        // Siehe 2.4.
    }

    fn instance_init(obj: &InitializingObject<Self>) {
        obj.init_template();
    }
}

impl ObjectImpl for PanelWidget {
    fn properties() -> &'static {
        // Diese Methode wird benötigt, wenn man Properties manuell definiert.
        // Mit #[derive(Properties)] und #[property(...)] wird dies oft automatisch generiert.
        // Hier explizit für Klarheit und falls Anpassungen nötig sind.
        use once_cell::sync::Lazy;
        static PROPERTIES: Lazy<Vec<glib::ParamSpec>> = Lazy::new(|| vec!);
        PROPERTIES.as_ref()
    }

    fn set_property(&self, _id: usize, value: &glib::Value, pspec: &glib::ParamSpec) {
        match pspec.name() {
            "position" => self.position.set(value.get().expect("Value must be of type PanelPosition")),
            "height-request" => self.height_request_prop.set(value.get().expect("Value must be of type i32")),
            "auto-hide" => self.auto_hide.set(value.get().expect("Value must be of type bool")),
            "show-glow-accent" => self.show_glow_accent.set(value.get().expect("Value must be of type bool")),
            // "glow-accent-color" ist nur lesbar, kein Setter von außen
            "monitor-index" => {
                // Hier könnte Logik für "Init-only" implementiert werden,
                // z.B. prüfen, ob das Widget bereits konstruiert ist.
                self.monitor_index.set(value.get().expect("Value must be of type i32"));
            }
            _ => unimplemented!(),
        }
        self.obj().notify_by_pspec(pspec); // Wichtig für explizites Notify
    }

    fn property(&self, _id: usize, pspec: &glib::ParamSpec) -> glib::Value {
        match pspec.name() {
            "position" => self.position.get().to_value(),
            "height-request" => self.height_request_prop.get().to_value(),
            "auto-hide" => self.auto_hide.get().to_value(),
            "show-glow-accent" => self.show_glow_accent.get().to_value(),
            "glow-accent-color" => self.glow_accent_color_prop.borrow().to_value(),
            "monitor-index" => self.monitor_index.get().to_value(),
            _ => unimplemented!(),
        }
    }

    fn constructed(&self) {
        self.parent_constructed(); // Wichtig: Aufruf der Methode der Elternklasse
        // Initialisierung nach Template-Initialisierung
        // z.B. Standard-Widgets hinzufügen, Signale verbinden, CSS laden
        let obj = self.obj();
        obj.set_property("height-request", self.height_request_prop.get()); // Initialen Wert setzen
        // Weitere Initialisierungen...
        // self.connect_theme_signals();
        // self.setup_default_children();
    }

    fn dispose(&self) {
        // Aufräumarbeiten, z.B. explizites Freigeben von Kind-Widgets
        // die nicht automatisch durch GTK verwaltet werden oder
        // um Referenzzyklen aufzulösen.
        while let Some(child) = self.obj().first_child() {
            child.unparent();
        }
    }
}

impl WidgetImpl for PanelWidget {} // Standardimplementierung oft ausreichend
impl BoxImpl for PanelWidget {}    // Standardimplementierung oft ausreichend
```

Die Verwendung von `#` und `#[template_child]` vereinfacht die Anbindung der im UI-File definierten Elemente an das Rust-Struct.1 Die `left_box`, `center_box` und `right_box` dienen als klar definierte Container für die typische Dreiteilung eines Panels, was das Layoutmanagement erheblich vereinfacht.

Die `dispose`-Methode spielt eine wichtige Rolle im Lebenszyklus von GTK4-Widgets. Für Container-Widgets wie das `PanelWidget`, die viele Kind-Widgets verwalten, ist eine sorgfältige Implementierung von `dispose` unerlässlich. GTK4 hat `gtk_widget_destroy()` für Nicht-Toplevel-Widgets entfernt und setzt stattdessen auf reguläres Referenzzählen.6 Die GTK-Dokumentation und Diskussionen legen nahe, dass benutzerdefinierte Container-Widgets in ihrer `dispose`-Methode `gtk_widget_unparent()` (oder das Äquivalent in den Bindings) für alle ihre Kinder aufrufen sollten.7 Dies hilft, Referenzzyklen zu durchbrechen und stellt sicher, dass Ressourcen korrekt freigegeben werden, insbesondere wenn Kind-Widgets komplexe interne Zustände oder eigene Referenzen halten.

#### 2.3.3. Interne Zustandsvariablen (private Felder)

Zusätzlich zu den über `#[template_child]` angebundenen UI-Elementen und den GObject-Properties (siehe 2.3.2 und 2.4) kann `PanelWidget` weitere interne Zustandsvariablen benötigen:

- `settings: RefCell<Option<gio::Settings>>`: Für den Zugriff auf GSettings, um Panel-spezifische Einstellungen (z.B. Auto-Hide-Verhalten, falls nicht über Property gesteuert) zu laden und zu beobachten.8
- `accent_color_cache: RefCell<Option<gdk::RGBA>>`: Ein interner Cache für die vom Theme bereitgestellte Akzentfarbe, um schnelle Aktualisierungen des "Leuchtakzents" zu ermöglichen. `glow_accent_color_prop` dient als GObject-Property-Spiegelung.
- `children_configs: RefCell<Vec<PanelChildConfig>>`: Potenziell eine Struktur, die die Konfiguration (Widget-Typ, Position, Reihenfolge) der dynamisch hinzugefügten Kind-Elemente speichert. Dies ist relevant, wenn das Panel hochgradig dynamisch konfigurierbar sein soll.

#### 2.3.4. Konstruktor (`fn new() -> Self`) und Initialisierungslogik (`constructed`, `class_init`, `instance_init`)

- `PanelWidget::new()`: Erstellt eine neue Instanz des `PanelWidget` über `glib::Object::new()`.
- `class_init()`: Wird einmal pro Klasse aufgerufen. Hier wird das Composite Template (`.ui`-Datei) mittels `klass.bind_template()` an die Klasse gebunden. Falls das Template Signal-Handler-Namen enthält, die auf Rust-Methoden gemappt werden sollen (Template Callbacks), muss hier auch `klass.bind_template_callbacks()` aufgerufen werden.2 Hier ist auch der Ort, um modulspezifisches CSS zu laden oder GObject-Properties und -Signale zu registrieren, falls dies nicht deklarativ über Makros geschieht.
- `instance_init()`: Wird für jede neue Instanz aufgerufen. Hier wird das Template für die spezifische Instanz mittels `obj.init_template()` initialisiert.2
- `constructed()`: Wird aufgerufen, nachdem das Objekt vollständig konstruiert und das Template für die Instanz initialisiert wurde. An dieser Stelle können:
    - Standard-Kind-Widgets (Platzhalter oder finale Widgets für AppMenu, Uhr etc.) programmatisch erstellt und in die `left_box`, `center_box`, `right_box` eingefügt werden.
    - Verbindungen zu Signalen der Domänenschicht (z.B. Theme-Änderungen von `domain::theming` via `gio::Settings` oder D-Bus) hergestellt werden.
    - Initialwerte für interne Zustände gesetzt und GObject-Properties initialisiert werden (z.B. die initiale Höhe des Panels).

Die konsequente Nutzung von Composite Templates wird in der gtk-rs Dokumentation 2 und Beispielen 1 stark betont und von GTK-Entwicklern empfohlen.9 Dieser Ansatz sollte als Standard für alle komplexeren Widgets in NovaDE etabliert werden, um Konsistenz zu fördern und die Einarbeitung zu erleichtern. Wenn das `PanelWidget` als eines der ersten komplexen UI-Module diesen Weg beschreitet, setzt es einen wichtigen Präzedenzfall.

### 2.4. `PanelWidget`: GObject-Eigenschaften (Properties)

GObject-Eigenschaften ermöglichen die Konfiguration des `PanelWidget` von außen, beispielsweise durch das `ui::control_center` oder programmatisch durch andere Teile der Anwendung. Sie werden üblicherweise mit dem `#[derive(Properties)]` Makro (nicht direkt in gtk-rs, aber `glib::Properties` ist das Äquivalent) und `#[property]` Attributen innerhalb der `imp::PanelWidget` Struktur definiert oder explizit über die `properties()`, `set_property()`, `property()` Methoden im `ObjectImpl` Trait implementiert.10

#### 2.4.1. Tabelle der definierten Eigenschaften

Die folgende Tabelle listet die für das `PanelWidget` definierten GObject-Eigenschaften auf.

**Tabelle 2.4.1: `PanelWidget` GObject-Eigenschaften**

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**Eigenschaftsname (Rust / XML)**|**Rust-Datentyp (intern)**|**GObject-Typ (extern)**|**Lese-/Schreibzugriff**|**Standardwert**|**Beschreibung**|
|`position` / `position`|`Cell<PanelPosition>`|`NovaDEPanelPosition` (Enum)|R/W|`PanelPosition::Top`|Position des Panels auf dem Bildschirm (Oben, Unten). `PanelPosition` ist ein `glib::Enum`.|
|`height-request` / `height-request`|`Cell<i32>`|`gint`|R/W|`48`|Angeforderte Höhe des Panels in Pixeln. Min: 24, Max: 128.|
|`auto-hide` / `auto-hide`|`Cell<bool>`|`gboolean`|R/W|`false`|Ob das Panel automatisch versteckt werden soll. Die detaillierte Logik ist extern.|
|`show-glow-accent` / `show-glow-accent`|`Cell<bool>`|`gboolean`|R/W|`true`|Ob der "Leuchtakzent" angezeigt werden soll.|
|`glow-accent-color` / `glow-accent-color`|`RefCell<Option<gdk::RGBA>>`|`GdkRGBA*` (boxed)|R (intern W)|`None` (wird vom Theme abgeleitet)|Die Farbe des Leuchtakzents. Wird primär intern gesetzt basierend auf dem System-Theme.|
|`monitor-index` / `monitor-index`|`Cell<i32>`|`gint`|R/W (Init-only Logik)|`0` (Primärmonitor)|Index des Monitors, auf dem dieses Panel angezeigt wird. Minimum: 0.|

Diese Tabelle ist entscheidend für Entwickler, die das Panel verwenden oder konfigurieren (z.B. aus dem Control Center). Sie liefert alle notwendigen Informationen auf einen Blick: wie die Eigenschaft in Rust und XML heißt, welcher Datentyp erwartet wird, ob sie gelesen/geschrieben werden kann, was der Standard ist und was sie bewirkt. Dies ist essentiell für die korrekte Ansteuerung und das Verständnis des Widgets und dient als Checkliste während der Implementierung.

#### 2.4.2. Detaillierte Spezifikation jeder Eigenschaft

- `position`:
    - Verwendet den in `panel_widget/mod.rs` definierten `PanelPosition` Enum, der als `glib::Enum` registriert sein muss, um in GObject-Properties verwendet zu werden.
    - Der Setter sollte den Wert validieren. Bei einem ungültigen Wert (falls der Enum nicht korrekt übergeben wird) wird eine Warnung via `tracing::warn` ausgegeben und der Wert nicht geändert.
    - Eine Änderung der Position löst eine Neuzeichnung und potenziell eine Neupositionierung des Panels durch die übergeordnete Shell-Logik aus. Das Panel selbst passt seine CSS-Klassen an (z.B. `.panel-top`, `.panel-bottom`).
- `height-request`:
    - Der Setter validiert den übergebenen Wert gegen die definierten Min/Max-Grenzen (24-128 Pixel). Werte außerhalb dieses Bereichs führen zu einer Warnung und werden ggf. auf den nächstgelegenen gültigen Wert korrigiert oder ignoriert.
    - Eine Änderung dieser Eigenschaft führt dazu, dass das Widget eine neue Größenanforderung (`gtk_widget_queue_resize`) auslöst.
- `auto-hide`:
    - Diese Eigenschaft dient als reiner Schalter. Die komplexe Logik für das automatische Verstecken und Anzeigen (z.B. Reaktion auf Maus am Bildschirmrand, Überlappung mit maximierten Fenstern, Animationen) ist nicht Teil des `PanelWidget` selbst, sondern wird von einer dedizierten Policy oder einem Manager in der `ui::shell` gesteuert. Diese Trennung von Konfiguration (Eigenschaft) und Mechanismus (Logik) hält das `PanelWidget` fokussiert und modular.
- `show-glow-accent`:
    - Steuert die Sichtbarkeit des CSS-basierten Leuchtakzents. Bei Änderung wird eine CSS-Klasse (z.B. `glow-accent-visible`) am Widget gesetzt oder entfernt, was eine Neuzeichnung auslöst.
- `glow-accent-color`:
    - Diese Eigenschaft ist primär von außen lesbar. Intern wird sie aktualisiert, wenn sich die System-Akzentfarbe ändert. Das Panel muss hierfür auf entsprechende Signale von `domain::theming` (z.B. über `gio::Settings` oder einen D-Bus Service) hören. Eine Änderung dieser internen Property aktualisiert die CSS-Variable, die für den Akzentstil verwendet wird, oder eine spezifische CSS-Regel.
- `monitor-index`:
    - Diese Eigenschaft sollte idealerweise nur bei der Initialisierung des Panels gesetzt werden. Eine spätere Änderung ist komplex, da sie eine vollständige Neuinitialisierung des Panels auf einem anderen Monitor bedeuten würde. Der Setter kann eine Warnung ausgeben, wenn versucht wird, den Wert nach der `constructed`-Phase zu ändern.

Die Verwendung eines spezifischen Enum-Typs (`PanelPosition`) anstelle von einfachen Strings oder Integern für die Position ist robuster und typsicherer, was durch `glib::ParamSpecEnum` unterstützt wird. Dies verhindert ungültige Werte und macht den Code selbstdokumentierender.

### 2.5. `PanelWidget`: Signale

Signale werden vom `PanelWidget` emittiert, um andere Teile der Anwendung über Zustandsänderungen oder spezifische Benutzerinteraktionen innerhalb des Panels zu informieren. Sie werden typischerweise im `ObjectImpl`-Block mit dem `#[signal]`-Attribut (oder durch manuelle Registrierung in `class_init`) definiert.8

#### 2.5.1. Tabelle der definierten Signale

**Tabelle 2.5.1: `PanelWidget` Signale**

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**Signalname (Rust / XML)**|**Parameter (Name, Typ)**|**Beschreibung**|**Typische Publisher**|**Typische Subscriber**|**Emissionsbedingungen**|
|`quick-settings-toggled` / `quick-settings-toggled`|`is_open: bool`|Informiert darüber, dass der Schnelleinstellungs-Button betätigt wurde und der gewünschte neue Zustand des Popups.|`PanelWidget` (intern, bei Klick auf den entsprechenden Button)|`ui::shell` Logik oder `QuickSettingsPanelWidget` direkt|Klick auf den Schnelleinstellungs-Button im Panel.|
|`notification-center-toggled` / `notification-center-toggled`|`is_open: bool`|Informiert darüber, dass der Benachrichtigungszentrum-Button betätigt wurde und der gewünschte neue Zustand des Popups.|`PanelWidget` (intern, bei Klick auf den entsprechenden Button)|`ui::shell` Logik oder `NotificationCenterPanelWidget` direkt|Klick auf den Benachrichtigungszentrum-Button im Panel.|
|`app-menu-toggled` / `app-menu-toggled`|`is_open: bool`|Informiert darüber, dass der AppMenu-Button betätigt wurde und der gewünschte neue Zustand des Menüs.|`PanelWidget` (intern, bei Klick auf den entsprechenden Button)|`ui::shell` (AppMenu-Logik)|Klick auf den AppMenu-Button im Panel.|

Diese Tabelle ist entscheidend für Entwickler, die auf Ereignisse innerhalb des Panels reagieren müssen. Sie listet klar auf, welche Signale existieren, welche Daten sie mitliefern und unter welchen Umständen sie ausgelöst werden. Dies ermöglicht es anderen Modulen, sich korrekt mit dem Panel zu integrieren.

#### 2.5.2. Detaillierte Spezifikation jedes Signals

- `quick-settings-toggled (is_open: bool)`:
    - Wird emittiert, wenn der Benutzer auf den Button zum Öffnen/Schließen der Schnelleinstellungen im Panel klickt.
    - Der Parameter `is_open` signalisiert den _gewünschten_ Zustand des Schnelleinstellungs-Popups (true für öffnen, false für schließen). Die Logik, die auf dieses Signal hört, ist dafür verantwortlich, den tatsächlichen Zustand des Popups entsprechend zu ändern.
- `notification-center-toggled (is_open: bool)`:
    - Analog zu `quick-settings-toggled`, jedoch für das Benachrichtigungszentrum.
- `app-menu-toggled (is_open: bool)`:
    - Analog zu `quick-settings-toggled`, jedoch für das Hauptanwendungsmenü.

Das ursprünglich in der Grobplanung erwogene Signal `child-widget-requested` wird nicht implementiert. Widgets sollten typischerweise nicht ihre eigenen Kinder "anfordern", sondern diese entweder intern erstellen oder von einer übergeordneten Instanz hinzugefügt bekommen. Die Verantwortung für die Komposition der Shell-Elemente und das Bestücken des Panels liegt bei der `ui::shell`-Logik, nicht beim Panel selbst. Dies vermeidet eine Umkehrung des Kontrollflusses und potenzielle schwer nachvollziehbare Abhängigkeiten.

Die hier definierten `*-toggled`-Signale sind ein exzellentes Beispiel für lose Kopplung. Das `PanelWidget` muss nicht wissen, wie das Schnelleinstellungs-Panel (`QuickSettingsPanelWidget`) implementiert ist oder wo es sich befindet. Es signalisiert lediglich die Absicht des Benutzers, dieses zu öffnen oder zu schließen. Die übergeordnete Shell-Logik oder das `QuickSettingsPanelWidget` selbst kann auf dieses Signal reagieren. Diese ereignisbasierte Kommunikation erhöht die Modularität, Flexibilität und Testbarkeit des Gesamtsystems erheblich.

### 2.6. `PanelWidget`: Methoden (Öffentliche API und interne Logik)

Die Methoden des `PanelWidget` definieren seine programmatische Schnittstelle und interne Hilfsfunktionen.

#### 2.6.1. Öffentliche Methoden

Diese Methoden bilden die öffentliche API des `PanelWidget` und werden in `panel_widget/mod.rs` innerhalb des `impl PanelWidget`-Blocks definiert.

**Tabelle 2.6.1: `PanelWidget` Öffentliche API-Methoden**

|   |   |   |   |   |
|---|---|---|---|---|
|**Methodensignatur (Rust)**|**Beschreibung**|**Vorbedingungen**|**Nachbedingungen**|**Ausgelöste Fehler (Typ $PanelWidgetError$)**|
|`pub fn new() -> Self`|Konstruktor. Erstellt eine neue Instanz des `PanelWidget`.|GTK-Umgebung ist initialisiert.|Eine gültige, initialisierte `PanelWidget`-Instanz wird zurückgegeben.|-|
|`pub fn add_child_widget(&self, child: &impl IsA<gtk::Widget>, area: PanelChildrenArea, order: i32)`|Fügt ein Kind-Widget einem bestimmten Bereich (`Left`, `Center`, `Right`) des Panels hinzu. `order` dient als Sortierhinweis.|`child` ist ein gültiges `gtk::Widget`. `area` ist ein gültiger `PanelChildrenArea`-Wert.|`child` wird dem spezifizierten Bereich des Panels hinzugefügt und ist sichtbar (wenn das Panel sichtbar ist).|`ChildWidgetAddFailed` (z.B. wenn `child` bereits ein Parent hat)|
|`pub fn remove_child_widget(&self, child: &impl IsA<gtk::Widget>) -> Result<(), PanelWidgetError>`|Entfernt ein spezifisches Kind-Widget aus dem Panel.|`child` ist ein aktuelles Kind-Widget des Panels.|`child` wird aus dem Panel entfernt und dessen Parent-Beziehung aufgelöst.|`ChildWidgetRemoveFailed` (z.B. wenn `child` nicht gefunden wird)|
|`pub fn update_theme_dependent_elements(&self, accent_color: &gdk::RGBA, theme_name: &str)`|Aktualisiert UI-Elemente, die vom Theme abhängen (insbesondere den Leuchtakzent und Hell/Dunkel-Modus Anpassungen).|`accent_color` ist gültig. `theme_name` gibt das aktuelle GTK-Theme an.|Leuchtakzent und andere themenabhängige Stile werden entsprechend der neuen Akzentfarbe und des Themes aktualisiert.|-|

Die Tabelle der öffentlichen API-Methoden dient als "Vertragsdokument" für andere Teile der UI-Schicht, die mit dem Panel interagieren. Sie definiert klar die unterstützten Operationen, deren Verhalten und Parameter.

#### 2.6.2. Interne Hilfsmethoden

Diese Methoden sind nicht Teil der öffentlichen API und dienen der internen Strukturierung und Logik. Sie werden typischerweise in `panel_widget/imp.rs` definiert oder als private Methoden in `panel_widget/mod.rs`.

**Tabelle 2.6.2: `PanelWidget` Interne Hilfsmethoden**

|   |   |   |
|---|---|---|
|**Methodensignatur (Rust) (Beispielhaft)**|**Beschreibung**|**Kontext der Verwendung**|
|`fn setup_initial_state(obj: &super::PanelWidget)`|Setzt initiale Property-Werte und verbindet notwendige Signale nach der Konstruktion.|Aufgerufen am Ende von `ObjectImpl::constructed()` für `imp::PanelWidget`.|
|`fn load_specific_css(obj: &super::PanelWidget)`|Lädt die `panel_widget.css` Datei (falls vorhanden) und wendet sie auf das Widget an.|Aufgerufen in `ObjectImpl::constructed()` oder `class_init`.|
|`fn update_glow_accent_style(obj: &super::PanelWidget, color: Option<&gdk::RGBA>, position: PanelPosition)`|Aktualisiert dynamisch die CSS-Klasse oder Inline-Stile für den Leuchtakzent.|Aufgerufen von `update_theme_dependent_elements` oder bei Änderung der `show-glow-accent` Eigenschaft.|
|`fn connect_theme_signals(obj: &super::PanelWidget)`|Verbindet mit Signalen von `domain::theming` (z.B. `gio::Settings` für `gtk-theme-name` und `color-scheme`) um Theme-Änderungen zu empfangen.|Aufgerufen in `setup_initial_state` oder `constructed`.|
|`fn get_target_box(obj: &super::PanelWidget, area: PanelChildrenArea) -> GtkBox`|Gibt eine geklonte Referenz auf die `left_box`, `center_box` oder `right_box` zurück.|Von `add_child_widget`.|

Interne Hilfsmethoden verbessern die Organisation und Wartbarkeit des Panel-Codes, indem sie wiederverwendbare Logik kapseln.

#### 2.6.3. Detaillierte Schritt-für-Schritt Implementierungslogik für ausgewählte Methoden

- `PanelWidget::add_child_widget(&self, child: &impl IsA<gtk::Widget>, area: PanelChildrenArea, order: i32)`:
    
    1. Ermittle das `imp::PanelWidget`-Objekt (interner Zustand) aus `self`.
    2. Rufe `get_target_box(area)` auf, um den Ziel-`gtk::Box` Container (`left_box`, `center_box` oder `right_box`) zu erhalten. Diese Boxen sind `TemplateChild<GtkBox>`.
    3. Prüfe, ob `child` bereits einen Parent hat. Wenn ja, und dieser Parent nicht das `PanelWidget` selbst ist, gib `$Result::Err(PanelWidgetError::ChildWidgetAddFailed("Widget already has a parent".into()))$` zurück.
    4. Setze eine benutzerdefinierte Eigenschaft oder ein `qdata`-Feld auf dem `child` mit dem Wert `order`. Dies ist optional und dient dazu, die Sortierreihenfolge zu speichern, falls eine dynamische Neusortierung implementiert wird. GTK `Box` selbst sortiert Kinder nicht dynamisch basierend auf einer Property; die Reihenfolge wird durch `append`, `prepend` oder `reorder_child` bestimmt.
    5. Füge `child` dem Ziel-`gtk::Box` hinzu: `target_box.append(child);`.
    6. Falls eine explizite Sortierung basierend auf `order` erforderlich ist und nicht durch die Reihenfolge des Hinzufügens gewährleistet wird:
        - Iteriere über die Kinder der `target_box`.
        - Entferne `child` temporär.
        - Finde die korrekte Einfügeposition basierend auf den `order`-Werten der anderen Kinder.
        - Füge `child` an der korrekten Position wieder ein (`target_box.insert_child_after(child, Option<previous_sibling>)` oder `target_box.reorder_child(child, new_index)`).
        - **Entscheidung für die Erstimplementierung:** Die Sortierung erfolgt primär durch die Wahl der `PanelChildrenArea` und die Reihenfolge, in der Widgets hinzugefügt werden. Die `order`-Eigenschaft dient als Vorbereitung für eventuell komplexere, zukünftige Sortieranforderungen.
    7. Gib `$Result::Ok(())$` zurück.
- `PanelWidget::update_theme_dependent_elements(&self, accent_color: &gdk::RGBA, theme_name: &str)`:
    
    1. Ermittle das `imp::PanelWidget`-Objekt.
    2. Aktualisiere die interne `glow_accent_color_prop` Property: `imp.glow_accent_color_prop.replace(Some(accent_color.clone()));`. Emittiere `notify::glow-accent-color`.
    3. Rufe `update_glow_accent_style(self, Some(accent_color), self.position())` auf.
    4. Passe ggf. CSS-Klassen für Hell/Dunkel-Modus an, basierend auf `theme_name` (z.B. wenn das Panel spezifische Stile für Dark-Mode hat, die nicht global greifen). Dies geschieht meist automatisch durch das GTK-Theming-System, aber manuelle Anpassungen sind hier möglich.
        
        Rust
        
        ```
        // In panel_widget/mod.rs
        //...
        // fn update_glow_accent_style(&self, color: Option<&gdk::RGBA>, position: PanelPosition) {
        //     let style_context = self.style_context();
        //     style_context.remove_class("glow-accent-visible"); // Alte Klasse entfernen
        //
        //     if self.show_glow_accent() && color.is_some() {
        //         style_context.add_class("glow-accent-visible");
        //
        //         // Dynamisches Setzen einer CSS-Variable für die Akzentfarbe
        //         // Dies ist der bevorzugte Weg, um Farben an CSS zu übergeben.
        //         let c = color.unwrap();
        //         let css_color = format!("rgba({},{},{},{})",
        //                                (c.red() * 255.0) as u8,
        //                                (c.green() * 255.0) as u8,
        //                                (c.blue() * 255.0) as u8,
        //                                c.alpha());
        //
        //         let provider = gtk::CssProvider::new();
        //         let css_data = format!(
        //             ".nova-panel-widget.glow-accent-visible {{ --nova-panel-glow-color: {}; }}",
        //             css_color
        //         );
        //         provider.load_from_data(&css_data);
        //
        //         // Provider mit hoher Priorität hinzufügen, um globale Theme-Variablen zu überschreiben,
        //         // oder besser: Theme stellt --nova-accent-color bereit und CSS nutzt diese.
        //         // Hier wird angenommen, dass das Panel seine eigene Akzentfarbe setzt.
        //         gtk::StyleContext::add_provider_for_display(
        //             &self.display(),
        //             &provider,
        //             gtk::STYLE_PROVIDER_PRIORITY_APPLICATION,
        //         );
        //
        //         // Panel-spezifische Klassen für Position (oben/unten)
        //         style_context.remove_class("panel-top");
        //         style_context.remove_class("panel-bottom");
        //         match position {
        //             PanelPosition::Top => style_context.add_class("panel-top"),
        //             PanelPosition::Bottom => style_context.add_class("panel-bottom"),
        //         }
        //     }
        // }
        ```
        
        Die bevorzugte Methode zur Anwendung der Akzentfarbe ist jedoch die Verwendung von CSS-Variablen, die vom globalen Theme bereitgestellt werden.12 Das `PanelWidget` würde dann nur eine Klasse wie `glow-accent-visible` setzen, und die Farbe käme automatisch vom Theme (`var(--nova-accent-color)` im CSS). Die obige `update_glow_accent_style` Methode sollte sich darauf konzentrieren, die notwendigen CSS-Klassen zu setzen und sicherzustellen, dass das Widget neu gezeichnet wird. Die direkte Manipulation von CSS-Providern für Farbwerte sollte vermieden werden, wenn das Theming-System dies global handhaben kann.

Die Methoden `add_child_widget` und `remove_child_widget` sind entscheidend für die Flexibilität des Panels. NovaDE könnte es Nutzern erlauben, Panel-Elemente zur Laufzeit anzupassen, oder verschiedene Profile könnten unterschiedliche Elementkombinationen laden. Die Methode `update_theme_dependent_elements` ist kritisch für die Einhaltung des visuellen Stils von NovaDE, insbesondere für den "Leuchtakzent", und muss bei jeder relevanten Theme-Änderung aufgerufen werden.

### 2.7. `PanelWidget`: UI-Layout und Theming (GTK4 Composite Template & CSS)

Das Layout und das grundlegende Erscheinungsbild des `PanelWidget` werden durch eine GTK4 Composite Template UI-Datei (`.ui`) und zugehörige CSS-Regeln definiert.

#### 2.7.1. Struktur der `panel_widget.ui` Datei

Die UI-Definition basiert auf `GtkBox` als Wurzel-Template-Klasse. Die Datei befindet sich unter `resources/ui/shell/panel_widget.ui` (oder einem ähnlichen Pfad, der über `gio::resources_register_include!` im Build-Skript eingebunden wird) und wird im Code mit `@template(resource = "/org/nova_de/ui/shell/panel_widget.ui")` referenziert.

XML

```
<?xml version="1.0" encoding="UTF-8"?>
<interface domain="nova-de"> <template class="NovaDEPanelWidget" parent="GtkBox">
    <property name="orientation">horizontal</property>
    <property name="css_classes">nova-panel-widget</property>
    <child>
      <object class="GtkBox" id="left_box">
        <property name="orientation">horizontal</property>
        <property name="halign">start</property>
        <property name="hexpand">true</property> <property name="css_classes">panel-area panel-left-area</property>
        <style>
          <class name="spacing-small"/> </style>
      </object>
    </child>
    <child>
      <object class="GtkBox" id="center_box">
        <property name="orientation">horizontal</property>
        <property name="halign">center</property>
        <property name="hexpand">false</property> <property name="css_classes">panel-area panel-center-area</property>
        <style>
          <class name="spacing-small"/>
        </style>
      </object>
    </child>
    <child>
      <object class="GtkBox" id="right_box">
        <property name="orientation">horizontal</property>
        <property name="halign">end</property>
        <property name="hexpand">true</property> <property name="css_classes">panel-area panel-right-area</property>
        <style>
          <class name="spacing-small"/>
        </style>
      </object>
    </child>
  </template>
</interface>
```

Die `hexpand`-Eigenschaften sind so konfiguriert, dass die `center_box` zentriert bleibt, während `left_box` und `right_box` den verfügbaren Raum dynamisch aufteilen. Die Klasse `spacing-small` (oder ähnlich) könnte im globalen CSS definiert sein, um konsistente Abstände zwischen den Panel-Elementen zu gewährleisten.

**Tabelle 2.7.1: `PanelWidget.ui` Kind-Widget-IDs und Typen (Primärcontainer)**

|   |   |   |
|---|---|---|
|**ID (#[template_child])**|**GTK-Widget-Typ**|**Zweck**|
|`left_box`|`GtkBox`|Container für linksbündige Panel-Elemente (z.B. AppMenu, WorkspaceSwitcher).|
|`center_box`|`GtkBox`|Container für zentrierte Panel-Elemente (z.B. Uhr).|
|`right_box`|`GtkBox`|Container für rechtsbündige Panel-Elemente (z.B. Tray, Indikatoren).|

Die deklarative Natur der `.ui`-Datei 2 erlaubt eine visuell nachvollziehbare Definition der Panel-Grundstruktur. Layout-Änderungen können oft direkt im XML erfolgen, ohne Rust-Code anpassen zu müssen.9 Dies beschleunigt die UI-Entwicklung. Die Tabelle 2.7.1 dokumentiert die Brücke zwischen XML-Deklaration und Rust-Code (`#[template_child]`).

#### 2.7.2. CSS-Klassen und -IDs für das `PanelWidget` und seine primären Elemente

Für das Styling werden CSS-Klassen verwendet. IDs sollten primär für die `#[template_child]`-Anbindung genutzt werden und im CSS nur sparsam, um die Spezifität niedrig zu halten.14

**Tabelle 2.7.2: `PanelWidget` CSS-Klassen**

|   |   |   |
|---|---|---|
|**CSS-Klasse**|**Anwendungsbereich**|**Beschreibung**|
|`nova-panel-widget`|Das Haupt-Widget des Panels (`NovaDEPanelWidget`)|Basis-Styling für das Panel (Hintergrund, Padding, Schrift etc.).|
|`panel-top`|`nova-panel-widget` (wenn Position oben)|Spezifische Stile für ein oben positioniertes Panel (z.B. unterer Rand/Schatten für Leuchtakzent).|
|`panel-bottom`|`nova-panel-widget` (wenn Position unten)|Spezifische Stile für ein unten positioniertes Panel (z.B. oberer Rand/Schatten für Leuchtakzent).|
|`panel-area`|`left_box`, `center_box`, `right_box`|Gemeinsames Styling für die Hauptbereiche (z.B. vertikales Padding).|
|`panel-left-area`|`left_box`|Spezifisches Styling für den linken Bereich.|
|`panel-center-area`|`center_box`|Spezifisches Styling für den mittleren Bereich.|
|`panel-right-area`|`right_box`|Spezifisches Styling für den rechten Bereich.|
|`glow-accent-visible`|`nova-panel-widget`|Wird dynamisch via Code gesetzt, um den Leuchtakzent sichtbar zu machen. Das CSS definiert dann den eigentlichen Stil.|
|`panel-child-item`|Alle direkten Kinder in `left/center/right_box`|Einheitliches Styling für Panel-Elemente (z.B. Margins, Paddings, vertikale Ausrichtung).|
|`spacing-small` / `spacing-medium`|`GtkBox` Container oder einzelne Elemente|Definiert Standardabstände (z.B. `column-spacing` für Boxen, `margin` für Items).|

Diese CSS-Klassen (Tabelle 2.7.2) sind unverzichtbar für Theme-Entwickler und UI-Designer, die das Aussehen des Panels anpassen wollen, da sie die "Haken" für das CSS-Styling bereitstellen.

#### 2.7.3. Anwendung von Design-Tokens (`domain::theming`) via `GtkCssProvider`

Das globale NovaDE-Theme, definiert in domain::theming, stellt Design-Tokens als CSS-Variablen bereit (z.B. --nova-accent-color, --nova-panel-background, --nova-text-color). Diese werden über GtkSettings:gtk-theme-name und GtkStyleContext::add_provider_for_display systemweit geladen.12

Das PanelWidget und sein spezifisches panel_widget.css (falls vorhanden) nutzen diese vordefinierten CSS-Variablen.

Beispielhafte Verwendung in `panel_widget.css` oder im globalen Theme:

CSS

```
.nova-panel-widget {
    background-color: var(--nova-panel-background, #23272e); /* Fallback-Farbe */
    color: var(--nova-fg-color, white);
    padding: 0 6px; /* Globales horizontales Padding für das Panel */
    min-height: 24px; /* Mindesthöhe sicherstellen */
}

.panel-area >.panel-child-item { /* Styling für Elemente direkt in den Bereichen */
    padding: 4px 6px; /* Vertikales und horizontales Padding für jedes Element */
    margin: 0 2px;    /* Kleiner horizontaler Abstand zwischen Elementen */
}

.nova-panel-widget.panel-top.glow-accent-visible {
    border-bottom: 2px solid var(--nova-accent-color, #5294e2); /* Standard-Akzentfarbe als Fallback */
    box-shadow: 0px 2px 5px 0px alpha(var(--nova-accent-color, #5294e2), 0.3);
}

.nova-panel-widget.panel-bottom.glow-accent-visible {
    border-top: 2px solid var(--nova-accent-color, #5294e2);
    box-shadow: 0px -2px 5px 0px alpha(var(--nova-accent-color, #5294e2), 0.3);
}

/* Beispiel für Abstands-Klassen, die im.ui File verwendet werden könnten */
.spacing-small > * { /* Gilt für direkte Kinder einer Box mit.spacing-small */
    margin-left: 4px;
    margin-right: 4px;
}
.spacing-small > *:first-child {
    margin-left: 0;
}
.spacing-small > *:last-child {
    margin-right: 0;
}
```

Die konsequente Nutzung von CSS-Variablen 15 ist der Schlüssel zur Erfüllung der Anforderungen an Anpassbarkeit und dynamische Theme-Wechsel. Das `PanelWidget` wird so zu einem integralen Bestandteil des visuell konsistenten NovaDE-Ökosystems.

#### 2.7.4. Spezifikation des "optionalen Leuchtakzents"

Der Leuchtakzent ist ein Kernmerkmal des Panel-Designs und wird primär über CSS realisiert:

- Eine CSS-Klasse, z.B. `glow-accent-visible`, wird dem `NovaDEPanelWidget`-Wurzelelement dynamisch hinzugefügt oder entfernt, basierend auf dem Wert der GObject-Eigenschaft `show-glow-accent`.
- Zusätzliche Klassen wie `panel-top` oder `panel-bottom` (basierend auf der `position`-Eigenschaft) bestimmen, ob der Akzent als unterer oder oberer Rand/Schatten erscheint.
- Die Farbe des Akzents wird durch die CSS-Variable `--nova-accent-color` (oder eine spezifischere wie `--nova-panel-glow-color`, falls abweichend vom globalen Akzent gewünscht) gesteuert. Diese Variable wird vom Theming-System (`domain::theming`) bereitgestellt und bei Theme-Wechseln aktualisiert.
- Der Akzent selbst wird durch eine Kombination aus `border` (z.B. `border-bottom` oder `border-top`) und einem subtilen `box-shadow` in der Akzentfarbe implementiert, um einen "Leucht"-Effekt zu erzeugen.
- Die genaue visuelle Ausprägung (Dicke des Borders, Farbe, Weichzeichnung und Ausdehnung des Schattens) wird in der CSS-Datei des Themes oder in einer spezifischen `panel_widget.css` definiert, die die Theme-Variablen nutzt.

Dieser Ansatz der Trennung von Struktur (UI-XML), Stil (CSS) und Verhalten (Rust), gefördert durch GTK4 und Composite Templates 2, ist eine bewährte Methode zur Entwicklung wartbarer und flexibler Benutzeroberflächen.

### 2.8. `PanelWidget`: Integration von Sub-Modulen (Platzhalter-Schnittstellen)

Das `PanelWidget` dient als Container. Die eigentlichen funktionalen Elemente (AppMenu-Button, Uhr, Indikatoren etc.) sind eigenständige Widgets/Module, die separat spezifiziert werden. Das `PanelWidget` stellt lediglich die Mechanismen bereit, um diese Kind-Widgets aufzunehmen und anzuordnen.

Die Integration erfolgt über die öffentliche Methode `PanelWidget::add_child_widget(&self, child: &impl IsA<gtk::Widget>, area: PanelChildrenArea, order: i32)`. Die übergeordnete `ui::shell`-Logik ist dafür verantwortlich, Instanzen der Sub-Modul-Widgets zu erstellen und sie dem entsprechenden `PanelWidget` (potenziell eines pro Monitor) hinzuzufügen.

- **2.8.1. AppMenu-Integration:** Ein `AppMenuButtonWidget` (separat spezifiziert) wird erstellt und via `add_child_widget` der `PanelChildrenArea::Left` hinzugefügt. Dieser Button löst das `app-menu-toggled`-Signal aus.
- **2.8.2. Workspace-Indikator-Integration:** Ein `WorkspaceIndicatorWidget` (separat spezifiziert) wird erstellt und der `PanelChildrenArea::Left` (oder `Center`, je nach Designentscheidung) hinzugefügt.
- **2.8.3. Uhr-Integration:** Ein `ClockWidget` (separat spezifiziert) wird erstellt und der `PanelChildrenArea::Center` (oder `Right`) hinzugefügt.
- **2.8.4. System-Tray-Integration:** Ein `SystemTrayWidget` (separat spezifiziert, potenziell komplex aufgrund des `StatusNotifierItem`-Protokolls) wird erstellt und der `PanelChildrenArea::Right` hinzugefügt.
- **2.8.5. Schnelleinstellungen-Button-Integration:** Ein `QuickSettingsButtonWidget` (separat spezifiziert, könnte ein einfacher `gtk::Button` oder `gtk::MenuButton` sein) wird erstellt und der `PanelChildrenArea::Right` hinzugefügt. Dieser Button löst das `quick-settings-toggled`-Signal aus, woraufhin das `QuickSettingsPanelWidget` (ein Popover-Typ 16) angezeigt wird.
- **2.8.6. Benachrichtigungszentrum-Button-Integration:** Ein `NotificationCenterButtonWidget` (analog zum Schnelleinstellungen-Button) wird erstellt und der `PanelChildrenArea::Right` hinzugefügt. Dieser Button löst das `notification-center-toggled`-Signal aus.
- **2.8.7. Netzwerk-, Energie-, Audio-Indikator-Integration:** Jeder Indikator (z.B. `NetworkIndicatorWidget`, `PowerIndicatorWidget`, `AudioIndicatorWidget` – alle separat spezifiziert) wird als eigenständiges Widget erstellt und der `PanelChildrenArea::Right` hinzugefügt. Diese Widgets sind dafür verantwortlich, Daten von relevanten Systemdiensten (z.B. NetworkManager, UPower 18, PipeWire) über D-Bus 11 oder andere Mechanismen der System- oder Domänenschicht zu beziehen und darzustellen.

Die Entscheidung, Sub-Module als separate Widgets zu behandeln, die über eine generische `add_child_widget`-Methode integriert werden, ist ein starkes Bekenntnis zur Modularität. Es erzwingt, dass jedes dieser Sub-Module eine wohldefinierte `gtk::Widget`-Schnittstelle besitzt und keine versteckten Abhängigkeiten zum `PanelWidget` aufbaut. Das Panel bleibt ein reiner Container, und die funktionalen Elemente können unabhängig entwickelt, getestet und potenziell wiederverwendet werden. Viele der durch Panel-Buttons ausgelösten Interaktionen, wie das Anzeigen von Schnelleinstellungen oder Kalendern, werden typischerweise als `GtkPopover` oder `GtkPopoverMenu` implementiert.4 Das Panel selbst hostet nur die Auslöser (Buttons), nicht die Popovers; die Signale des Panels sind das Bindeglied.

### 2.9. `PanelWidget`: Fehlerbehandlung und Ausnahmeklassen

Eine robuste Fehlerbehandlung ist essentiell für die Stabilität der UI.

#### 2.9.1. Definition der `PanelWidgetError` Enum (via `thiserror`)

In `panel_widget/error.rs` wird eine spezifische Fehler-Enum für das `PanelWidget`-Modul definiert:

Rust

```
use thiserror::Error;

#
pub enum PanelWidgetError {
    #[error("Failed to add child widget to panel: {0}")]
    ChildWidgetAddFailed(String),

    #[error("Failed to remove child widget from panel: {0}")]
    ChildWidgetRemoveFailed(String),

    #[error("Invalid configuration provided for panel widget: {0}")]
    InvalidConfiguration(String),

    #[error("Failed to apply style to panel widget: {0}")]
    StylingError(String),
    // Weitere spezifische Fehlerarten nach Bedarf
}
```

Diese Enum, die `thiserror::Error` ableitet, ermöglicht eine klare und typisierte Fehlerkommunikation. Aufrufer können spezifisch auf Fehler reagieren.

#### 2.9.2. Fehlerbehandlungsstrategien innerhalb des Moduls

- Öffentliche Methoden, die fehlschlagen können (z.B. `add_child_widget`, `remove_child_widget`), geben `$Result<T, PanelWidgetError>$` zurück, wie in den Entwicklungsrichtlinien von NovaDE vorgesehen.
- Interne Fehler, die nicht nach außen propagiert werden müssen oder können (z.B. Fehler beim Laden optionaler Ressourcen, die zu einem Fallback-Verhalten führen), werden mit `tracing::error!` oder `tracing::warn!` geloggt.
- Panics sind strikt zu vermeiden, außer bei nicht behebbaren Programmierfehlern (z.B. Nichterfüllung von Vorbedingungen, die durch `debug_assert!` in Entwicklungs-Builds geprüft werden könnten). Solche Fälle deuten auf Fehler in der Logik hin und sollten während der Entwicklung identifiziert und behoben werden.

Die konsequente Anwendung dieser Fehlerbehandlungsstrategien erhöht die Robustheit und Wartbarkeit des `PanelWidget` und des Gesamtsystems.

### 2.10. `PanelWidget`: Interaktion mit Domänen- und Systemschicht

Das `PanelWidget` muss die in der NovaDE-Architektur definierten Schichtengrenzen wahren.

#### 2.10.1. Abruf von Daten

- **Theme-Informationen:** Das `PanelWidget` reagiert auf Änderungen der globalen Akzentfarbe und des Hell/Dunkel-Modus. Diese Informationen werden von `domain::theming` bereitgestellt. Die Anbindung erfolgt typischerweise über:
    - Beobachtung von `gio::Settings`-Schlüsseln (z.B. `org.gnome.desktop.interface gtk-theme`, `org.gnome.desktop.interface color-scheme`).
    - Beobachtung eines NovaDE-spezifischen `gio::Settings`-Schlüssels oder eines D-Bus-Signals für die Akzentfarbe, falls diese nicht über Standardmechanismen abgedeckt ist. Die Methode `connect_theme_signals` im `PanelWidget` ist für die Einrichtung dieser Beobachter zuständig. Bei Empfang einer Änderung wird `update_theme_dependent_elements` aufgerufen.
- **Workspace-Informationen:** Das `PanelWidget` selbst benötigt keine direkten Workspace-Daten. Diese werden vom `WorkspaceIndicatorWidget` (oder einem ähnlichen, im Panel gehosteten Widget) verarbeitet, das seinerseits mit `domain::workspaces` kommuniziert.
- **Status-Indikatoren (Netzwerk, Energie, etc.):** Analog zu den Workspace-Informationen beziehen die jeweiligen Indikator-Widgets ihre Daten von der System- oder Domänenschicht (z.B. UPower-Status via D-Bus 18). Das `PanelWidget` stellt nur den Container bereit.

#### 2.10.2. Auslösen von Aktionen

- Das `PanelWidget` löst primär UI-interne Aktionen aus, wie das Anzeigen von Popovers (z.B. Schnelleinstellungen, Benachrichtigungszentrum) über seine GObject-Signale.
- Das Öffnen von Systemeinstellungsmodulen (z.B. Klick auf den Netzwerk-Indikator öffnet die Netzwerkeinstellungen im `ui::control_center`) wird von den jeweiligen Indikator-Widgets selbst initiiert. Diese können hierfür D-Bus-Methodenaufrufe an `ui::control_center` senden oder andere IPC-Mechanismen nutzen, die von der `ui::shell` oder `ui::control_center` bereitgestellt werden.

Diese klare Trennung stellt sicher, dass das `PanelWidget` ein reines UI-Element bleibt und nicht mit systemspezifischer Logik oder direkten Datenabfragen aus tieferen Schichten überfrachtet wird. Dies erhält die Modularität und Testbarkeit der Gesamtarchitektur. Viele Daten, die im Panel angezeigt werden (Netzwerkstatus, Akkustand), fließen nicht direkt in das Panel, sondern in die spezialisierten Indikator-Widgets, die das Panel hostet. Das Panel ist somit eher ein "Präsentator von Präsentatoren".

### 2.11. `PanelWidget`: Testaspekte und -szenarien

Umfassende Tests sind gemäß den Entwicklungsrichtlinien von NovaDE für alle Module erforderlich.

#### 2.11.1. Unit-Tests für kritische Logik

- **Property-Handling:** Testen der Setter und Getter für alle GObject-Properties, einschließlich Validierungslogik (z.B. für `height-request`) und korrekte Emission von `notify::*`-Signalen bei Wertänderungen.
- **Kind-Widget-Management:** Testen der Logik zum Hinzufügen (`add_child_widget`) und Entfernen (`remove_child_widget`) von Kind-Widgets. Dies beinhaltet die Überprüfung, ob Widgets korrekt den Bereichen `left_box`, `center_box`, `right_box` hinzugefügt und wieder entfernt werden und ob Fehlerfälle (z.B. Hinzufügen eines Widgets, das bereits einen Parent hat) korrekt mit `$PanelWidgetError$` behandelt werden.
- **Zustandsänderungen:** Testen interner Zustandsänderungen als Reaktion auf den Aufruf öffentlicher Methoden oder die Änderung von Properties (z.B. Setzen der CSS-Klasse `glow-accent-visible` bei Änderung von `show-glow-accent`).
- **Abhängigkeits-Mocking:** Wo das `PanelWidget` von externen Signalen abhängt (z.B. Theme-Änderungen), sollten diese Abhängigkeiten in Unit-Tests gemockt werden, um die Reaktion des Panels (z.B. Aufruf von `update_theme_dependent_elements`) isoliert zu testen.

#### 2.11.2. Integrationsszenarien mit (simulierten) Sub-Modulen

- **Layout-Tests:** Testen des korrekten Layouts von hinzugefügten (Mock-)Kind-Widgets in den drei Hauptbereichen. Überprüfung, ob `hexpand`-Einstellungen und Ausrichtung wie erwartet funktionieren.
- **Signal-Emission:** Testen, ob das `PanelWidget` die definierten Signale (z.B. `quick-settings-toggled`) korrekt emittiert, wenn simulierte Kind-Buttons (die diese Aktionen auslösen würden) "geklickt" werden (programmatische Aktivierung im Test).
- **Interaktion mit Popovers:** Obwohl die Popovers selbst separate Widgets sind, kann getestet werden, ob die Signale des Panels korrekt von einer Mock-Logik empfangen werden, die ein Popover simulieren würde.

#### 2.11.3. Überlegungen zu UI-Tests

Für UI-Tests können Werkzeuge aus dem gtk-rs Ökosystem oder externe Test-Frameworks in Betracht gezogen werden. Szenarien umfassen:

- **Sichtbarkeit und Erscheinungsbild:** Überprüfung der korrekten Anzeige des Panels und seiner Hauptbereiche auf einem simulierten Bildschirm. Visuelle Überprüfung (ggf. automatisiert durch Screenshot-Vergleiche) des Leuchtakzents und seiner Reaktion auf Theme-Änderungen.
- **Interaktion:** Simulation von Benutzerinteraktionen wie Klicks auf Buttons im Panel und Überprüfung, ob die entsprechenden Aktionen (z.B. das (simulierte) Öffnen eines Popovers oder Menüs) ausgelöst werden.
- **Dynamische Anpassungen:** Testen der Reaktion des Panels auf Laufzeitänderungen von Eigenschaften (z.B. Höhe, Position).

Das Testen von Widgets, die Composite Templates verwenden, erfordert möglicherweise spezifische Ansätze, um Template-Kinder zu inspizieren oder zu mocken. Die Test-Utilities von gtk-rs oder Ansätze, die eine laufende GTK-Anwendungsumgebung für Tests bereitstellen, können hier nützlich sein. Durch die Definition von Testszenarien bereits in der Spezifikationsphase wird sichergestellt, dass Testbarkeit von Anfang an ein Designkriterium ist, was spätere Probleme bei der Testautomatisierung vermeiden hilft.

## 3. Anhang

### 3.1. Referenzen zur "Technischen Gesamtspezifikation & Richtlinien"

Dieses Dokument ist im Kontext der übergeordneten "Technischen Gesamtspezifikation & Richtlinien für NovaDE" zu verstehen. Dieses Hauptdokument enthält verbindliche Vorgaben bezüglich der Gesamtarchitektur, des Technologie-Stacks, der Code-Qualitätsstandards und der allgemeinen Entwicklungsrichtlinien, die auch für die Implementierung des `PanelWidget` gelten.

### 3.2. Glossar für `PanelWidget`-spezifische Begriffe

- **Leuchtakzent:** Ein visueller Effekt, typischerweise ein farbiger Rand und/oder Schatten, am oberen oder unteren Rand des Panels, dessen Farbe sich dynamisch an die Akzentfarbe des System-Themes anpasst.
- **`PanelChildrenArea`:** Eine Rust-Enum (`Left`, `Center`, `Right`), die zur Spezifizierung dient, in welchem Hauptbereich eines `PanelWidget` ein Kind-Widget platziert werden soll.
- **Composite Template:** Eine XML-basierte UI-Definitionsdatei (`.ui`), die die Struktur und das initiale Layout eines benutzerdefinierten GTK-Widgets beschreibt und im Rust-Code über Makros an die Widget-Implementierung gebunden wird.2
- **Design-Token:** Abstrakte Variablen (oft als CSS-Variablen implementiert), die Designentscheidungen wie Farben, Schriftgrößen oder Abstände repräsentieren und eine konsistente und anpassbare Gestaltung über die gesamte Anwendung hinweg ermöglichen.15

**Technische Gesamtspezifikation & Richtlinien: Linux Desktop-Umgebung "NovaDE" (Essenz)**

**I. Vision und Kernziele**

- **Vision:** NovaDE ist eine innovative Linux-Desktop-Umgebung, die eine moderne, schnelle, intuitive und KI-gestützte Benutzererfahrung für Entwickler, Kreative und alltägliche Nutzer schafft. Ziel ist die Maximierung von Produktivität und Freude an der Systeminteraktion.
- **Kernziele:**
    - **Performance:** Durchgehend schnelle, reaktionsschnelle Umgebung.
    - **Intuition:** Natürlich anfühlende, leicht erlernbare und intelligent unterstützende UI.
    - **Modernität:** Einsatz aktueller Technologien und Designprinzipien.
    - **Modularität & Wartbarkeit:** Klare Architektur für einfache Erweiterung, Testbarkeit und Wartung.
    - **Anpassbarkeit:** Weitreichende Personalisierungsmöglichkeiten für Optik und Verhalten.
    - **KI-Integration:** Nahtlose, sichere KI-Assistenz unter voller Benutzerkontrolle.
    - **Stabilität & Sicherheit:** Höchste Priorität durch geeignete Technologien und Entwicklungspraktiken.

**II. Architektonischer Überblick: Vier-Schichten-Architektur**

NovaDE nutzt eine strikte, vier-schichtige Architektur für Modularität, lose Kopplung und hohe Kohäsion. Kommunikation zwischen Schichten erfolgt ausschließlich über wohldefinierte Schnittstellen.

1. **Kernschicht (Core Layer): Systemfundament**
    
    - **Verantwortlichkeiten:** Fundamentale Datentypen (`Point<T>`, `Color`, `TokenIdentifier`), allgemeine Dienstprogramme, Basis-Infrastruktur für Konfigurationsparsing und -zugriff (TOML, Serde), globales Logging-Framework (`tracing`), allgemeine Fehlerdefinitionen (`thiserror`).
    - **Abhängigkeiten:** Minimal (Rust std, uuid, chrono, thiserror, tracing, serde, toml, once_cell). Keine NovaDE-internen Abhängigkeiten.
    - **Interaktionen:** Stellt Funktionalität für alle höheren Schichten bereit.
2. **Domänenschicht (Domain Layer): Geschäftslogik & Regeln**
    
    - **Verantwortlichkeiten:** UI-unabhängige Kernlogik.
        - `domain::theming`: Logik der Theming-Engine, Design-Token-Verwaltung, dynamische Theme-Wechsel, `AppliedThemeState`.
        - `domain::workspaces`: Logik für Arbeitsbereiche ("Spaces"), Fensterzuweisung (`WindowIdentifier`, `WorkspaceId`), Workspace-Management, Persistenz.
        - `domain::user_centric_services`: KI-Interaktionslogik (`AIInteractionContext`), Einwilligungsmanagement (`AIConsent`, `AIDataCategory`, `AIModelProfile`), Benachrichtigungslogik (`Notification`, `NotificationUrgency`).
        - `domain::notifications_rules`: Regelbasierte Benachrichtigungsverarbeitung.
        - `domain::global_settings_and_state_management`: Verwaltung globaler Desktop-Einstellungen (`GlobalDesktopSettings`, `SettingValue`).
        - `domain::window_management_policy`: High-Level-Richtlinien für Fensterplatzierung, Tiling (Spalten, Spiralen), Snapping, Gaps.
    - **Abhängigkeiten:** Nur Kernschicht.
    - **Interaktionen:** Stellt Logik und Zustand für System- und UI-Schicht bereit (oft via Traits und Events wie `ThemeChangedEvent`).
3. **Systemschicht (System Layer): OS-Interaktion & technische Umsetzung**
    
    - **Verantwortlichkeiten:** Interaktion mit OS, Hardware, externen Diensten. Setzt Domänenrichtlinien technisch um.
        - `system::compositor`: Smithay-basierter Wayland-Compositor (`xdg-shell`, `wlr-layer-shell`, etc.), XWayland.
        - `system::input`: `libinput`-basierte Eingabeverarbeitung, Gesten, Seat-Management (`xkbcommon`).
        - `system::dbus`: `zbus`-Schnittstellen zu Systemdiensten (NetworkManager, UPower, logind, org.freedesktop.Notifications, org.freedesktop.secrets, PolicyKit).
        - `system::outputs`: Monitorkonfiguration (`wlr-output-management`, DPMS).
        - `system::audio`: `pipewire-rs` Client für Audio-Management.
        - `system::mcp`: `mcp_client_rs` für KI-Modell-Kommunikation basierend auf `domain::user_centric_services`.
        - `system::portals`: Backend für XDG Desktop Portals (FileChooser, Screenshot).
        - `system::window_mechanics`: Technische Umsetzung des Fenstermanagements (Positionierung, Tiling, Fokus) gemäß `domain::window_management_policy`. Technische Basis für "Intelligente Tab-Leiste".
    - **Abhängigkeiten:** Kern- und Domänenschicht.
    - **Interaktionen:** Stellt systemnahe Dienste/Events für UI-Schicht bereit. Empfängt UI-Befehle.
4. **Benutzeroberflächenschicht (User Interface Layer): Darstellung & Benutzerinteraktion**
    
    - **Verantwortlichkeiten:** Grafische Darstellung und direkte Benutzerinteraktion. Basiert auf GTK4 (`gtk4-rs`).
        - `ui::shell`: Haupt-Shell-UI (PanelWidget, SmartTabBarWidget pro "Space", QuickSettingsPanelWidget, WorkspaceSwitcherWidget, QuickActionDockWidget, NotificationCenterPanelWidget).
        - `ui::control_center`: Modulare GTK4-Anwendung für Systemeinstellungen.
        - `ui::widgets`: Widget-System (RightSidebarWidget, WidgetManagerService, WidgetPickerPopover, diverse Widgets wie ClockWidget).
        - `ui::window_manager_frontend`: UI-Aspekte des Fenstermanagements (CSD, OverviewModeWidget, AltTabSwitcherWidget).
        - `ui::notifications_frontend`: Pop-up-Benachrichtigungen (NotificationPopupWidget).
        - `ui::theming_gtk`: Anwendung von CSS (aus `domain::theming`) via `GtkCssProvider`.
        - `ui::components`: Wiederverwendbare GTK4-Widgets.
        - `ui::speed_dial`: Startansicht für leere Workspaces.
        - `ui::command_palette`: Kontextuelle Befehlspalette.
    - **Abhängigkeiten:** Alle darunterliegenden Schichten.
    - **Interaktionen:** Empfängt System-Events. Visualisiert Daten aus Domäne/System. Löst Aktionen in Domäne/System aus.

**III. Technologie-Stack (Verbindliche Auswahl)**

|Bereich|Technologie/Standard|
|:--|:--|
|Programmiersprache|Rust|
|Build-System|Meson|
|GUI-Toolkit|GTK4 (mit `gtk4-rs` Bindings)|
|Wayland Compositor & Bibliotheken|Smithay Toolkit|
|Essentielle Wayland-Protokolle|`wayland.xml`, `xdg-shell`, `wlr-layer-shell-unstable-v1`, `xdg-decoration-unstable-v1`, `wlr-foreign-toplevel-management-unstable-v1`, `wlr-output-management-unstable-v1`, etc.|
|Inter-Prozess-Kommunikation (IPC)|D-Bus (mit `zbus` Crate)|
|KI-Integration|Model Context Protocol (MCP) (mit `mcp_client_rs` Crate)|
|Eingabeverarbeitung|`libinput` (integriert via Smithay)|
|Audio-Management|PipeWire (mit `pipewire-rs` Crate)|
|Geheimnisverwaltung|Freedesktop Secret Service API (via D-Bus)|
|Rechteverwaltung|PolicyKit (polkit) (via D-Bus)|
|Theming-Implementierung (UI)|Token-basiert, GTK4 CSS Custom Properties (`var()`)|
|Sandboxing-Interaktion|XDG Desktop Portals (via D-Bus)|

In Google Sheets exportieren

**IV. Entwicklungsrichtlinien (Verbindlich)**

- **Rust:**
    - **Stil:** `rustfmt` (Standard), max. 100 Zeichen/Zeile, 4 Leerzeichen Einrückung.
    - **API-Design:** Rust API Guidelines Checklist.
    - **Fehlerbehandlung:** `thiserror` pro Modul, `Result<T, E>`, Panics nur für interne Invariantenverletzungen/Tests. Klare `#[error]`-Nachrichten, keine sensiblen Daten. `source()`-Kette erhalten.
    - **Logging & Tracing:** `tracing` Crate, Spans (`#[tracing::instrument]`), Standard-Level, keine sensiblen Daten.
    - **Nebenläufigkeit:** `async/await` (tokio, `glib::MainContext::spawn_local`), thread-sichere Datenstrukturen.
    - **Crate-Struktur:** Logische Aufteilung (z.B. `novade-core`, `novade-domain`), klare APIs.
- **Versionskontrolle:** Git, GitHub Flow, PRs mit Review & CI-Checks, Conventional Commits.
- **Teststrategie:** Unit-Tests (Kern, Domäne, Logik-Komponenten), Integrationstests, Compositor-Tests (Smithay Headless Backends), UI-Tests (Accessibility-APIs).
- **CI-Pipeline:** Tests, `cargo fmt --check`, `cargo clippy -D warnings`, `cargo audit`.
- **Dokumentation:**
    - **Code (rustdoc):** Umfassend für alle öffentlichen APIs (Was, Warum, Wie, Fehler, Sicherheit, Beispiele).
    - **Architektur:** Dieses Dokument als Referenz, High-Level-Diagramme.
    - **READMEs:** Pro Crate/Komponente.
    - **Metadaten:** Vollständige `Cargo.toml`.
    - **Benutzer-/Entwicklerdoku:** Parallel erstellen.

**V. Initiale Schicht- und Komponentenspezifikationen (Struktur)**

Detaillierte Spezifikationen pro Komponente/Schicht folgen einer einheitlichen Struktur: Modul-/Komponentenübersicht, Datenstrukturen/Typdefinitionen (Rust-Syntax), Öffentliche API/Interne Schnittstellen (Signaturen, Logik, Events), Fehlerbehandlung (`thiserror`-Enum), Interaktionen/Abhängigkeiten, Implementierungsschritte, Testaspekte.

**VI. Deployment-Überlegungen**

- **Paketierung:** Native Pakete (.deb, .rpm), Flatpak (evaluieren). Meson-Build für Artefakte.
- **Systemintegration:** Display Manager (GDM, LightDM), `systemd` User Sessions, PAM, XDG Base Directory Specification.
- **Konfiguration:** Sinnvolle Standardkonfiguration, UI-basierte Anpassung (`ui::control_center`), Trennung System-/Benutzerkonfig.
- **Updates:** Distro-Paketmanager, Flatpak-Mechanismus. SemVer. Konfigurationsmigration.

**VII. Anforderungen aus der Benutzererfahrung (Essenz)**

NovaDE soll eine **intelligente, intuitive und persönliche** Erfahrung bieten, die sich wie eine natürliche Erweiterung des Nutzers anfühlt.

- **Ästhetik & Navigation:** Klare, dunkle Ästhetik mit benutzerwählbaren Akzentfarben. Flüssige Animationen (Wayland-basiert). Mühelose Navigation und Informationszugriff.
- **Personalisierung (Theming):** Dynamischer Wechsel zwischen Designs (Hell/Dunkel) und Akzentfarben zur Laufzeit (Token-basiertes Theming).
- **Fenster- & Workspace-Management ("Spaces"):**
    - **Intelligente Tab-Leiste pro Space:** Zeigt "angepinnte" Apps/Split-Views.
    - **Workspace-Switcher (Linke Seitenleiste):** Visueller Wechsel zwischen Spaces mit Icons/Namen.
    - **Fortschrittliche Fensterverwaltung:** Automatisches Tiling (Spalten, Spiralen etc.), Snapping, Stacking, Floating mischbar, konfigurierbare Gaps.
    - **Übersichtsmodus:** Visuelle Darstellung aller Fenster/Spaces, Drag & Drop zwischen Spaces.
- **Interaktionskomponenten:**
    - **Kontroll-/Systemleiste(n):** Schneller Zugriff auf Systemfunktionen, Benachrichtigungszentrum.
    - **Quick-Settings-Panel:** Schnelleinstellungen ohne Control Center.
    - **Control Center:** Zentrale, modulare Systemeinstellungen mit Live-Vorschau.
    - **Schnellaktionsdock:** Konfigurierbar für Apps, Dateien, Aktionen; intelligente Vorschläge.
    - **Adaptive Seitenleisten & Widgets:** Links für Navigation/"Spaces", rechts für informative Widgets (Uhr, Kalender, Wetter, etc.), anpassbar via Drag & Drop.
    - **Speed-Dial:** Startansicht für leere Workspaces mit Favoriten und intelligenten Vorschlägen.
    - **Kontextuelle Befehlspalette (`Super+Space`):** Schneller Zugriff auf Aktionen, Einstellungen, Apps via Texteingabe mit Fuzzy-Matching und Kontextsensitivität.
- **KI-Integration (Sicher & Kontrolliert):**
    - **Model Context Protocol (MCP):** Sicherer, standardisierter Zugriff auf lokale/Cloud-LLMs.
    - **Explizite Zustimmung:** Keine KI-Aktion ohne klares Verständnis und Erlaubnis des Nutzers. Feingranulare Berechtigungen.
    - **Praktische Funktionen:** Sprachsteuerung, automatisierte Dateiaufgaben, kontextbezogene Informationen.
    - **Sichere Speicherung:** Sensible Daten (API-Keys) via Freedesktop Secret Service.
- **Systemintegration:**
    - **Wayland & XWayland:** Moderne, sichere Darstellung.
    - **D-Bus:** Kommunikation mit Systemdiensten.
    - **PolicyKit:** Sicherheit für privilegierte Aktionen.
    - **`libinput`:** Präzise Eingabe, Gestensteuerung.
    - Multi-Monitor-Support (Auflösung, Skalierung, Anordnung).
    - Audio-Management (PipeWire).
    - Effiziente Energienutzung.

**VIII. Schlussfolgerung**

Diese Gesamtspezifikation legt das verbindliche Fundament für NovaDE. Die präzisen Entscheidungen zu Architektur, Technologien und Richtlinien, kombiniert mit den Anforderungen aus der Nutzererfahrung, bilden die Grundlage für die detaillierte Implementierung. Konsequente Einhaltung dieser Spezifikation wird eine hochwertige, moderne, sichere, performante und anpassungsfähige Desktop-Umgebung hervorbringen.

 Quellen

Jetzt bitte inklusive Beschreibung der Features wie in der ersten anfrage mitgesendet als eine Kompakte Gesamtdefinition

S

Systemingeneur

Benutzerdefiniertes Gem

Absolut. Hier ist die optimierte Essenz der technischen Gesamtspezifikation und Richtlinien für NovaDE, nun inklusive einer kompakten Beschreibung der Features, die die Anforderungen aus der Nutzererfahrung widerspiegeln:

**Technische Gesamtspezifikation & Richtlinien: Linux Desktop-Umgebung "NovaDE" (Kompakte Gesamtdefinition inkl. Features)**

**I. Vision und Kernziele**

- **Vision:** NovaDE (Nova Desktop Environment) ist eine innovative Linux-Desktop-Umgebung, die eine moderne, schnelle, intuitive und KI-gestützte Benutzererfahrung schafft. Sie ist optimiert für Entwickler, Kreative und alltägliche Nutzer und zielt darauf ab, Produktivität und Freude an der Interaktion mit dem System zu maximieren.
- **Kernziele:** Performance, Intuition, Modernität, Modularität & Wartbarkeit, Anpassbarkeit, sichere KI-Integration, Stabilität & Sicherheit.

**II. Architektonischer Überblick: Vier-Schichten-Architektur**

NovaDE basiert auf einer strengen, vier-schichtigen Architektur (Kern, Domäne, System, Benutzeroberfläche) für Modularität, lose Kopplung und hohe Kohäsion. Kommunikation erfolgt über wohldefinierte Schnittstellen.

1. **Kernschicht (Core Layer):**
    
    - **Verantwortlichkeiten:** Fundamentale Datentypen (z.B. `Point<T>`, `Color`), Dienstprogramme, Konfigurationsprimitive (TOML, Serde), Logging (`tracing`), Basis-Fehler (`thiserror`).
    - **Featurespiegelung:** Stellt die atomaren Bausteine für alle visuellen und logischen Elemente bereit.
2. **Domänenschicht (Domain Layer):**
    
    - **Verantwortlichkeiten:** UI-unabhängige Geschäftslogik.
        - `domain::theming`: Logik für das Erscheinungsbild, Design-Token-Verwaltung, dynamische Theme-Wechsel (Hell/Dunkel, Akzentfarben).
        - `domain::workspaces`: Verwaltung von Arbeitsbereichen ("Spaces"), Fensterzuweisung, Workspace-Orchestrierung und -Persistenz.
        - `domain::user_centric_services`: Logik für KI-Interaktionen (inkl. Einwilligungsmanagement für Datenkategorien wie `FileSystemRead`, `ClipboardAccess`), Benachrichtigungsverwaltung.
        - `domain::notifications_rules`: Regelbasierte, dynamische Verarbeitung von Benachrichtigungen.
        - `domain::global_settings_and_state_management`: Verwaltung globaler Desktop-Einstellungen.
        - `domain::window_management_policy`: Richtlinien für Fensterplatzierung, automatisches Tiling (Layouts: Spalten, Spiralen), Snapping, Fenstergruppierung, Gap-Management.
    - **Featurespiegelung:** Definiert _was_ personalisierbar ist (Themes, Akzente), _wie_ Arbeitsbereiche funktionieren (Spaces mit Icons, gepinnten Apps), _wie_ KI sicher und mit Zustimmung agiert und _welche_ Regeln für Fenster gelten.
3. **Systemschicht (System Layer):**
    
    - **Verantwortlichkeiten:** OS-Interaktion, technische Umsetzung der Domänenrichtlinien.
        - `system::compositor`: Smithay-basierter Wayland-Compositor (Implementierung von `xdg-shell`, `wlr-layer-shell-unstable-v1`, etc.), XWayland.
        - `system::input`: `libinput`-basierte Eingabeverarbeitung, Gestenerkennung, Seat-Management (`xkbcommon`).
        - `system::dbus`: `zbus`-Schnittstellen zu Systemdiensten (NetworkManager, UPower, logind, org.freedesktop.Notifications, org.freedesktop.secrets, PolicyKit).
        - `system::outputs`: Monitorkonfiguration (Auflösung, Skalierung, DPMS über `wlr-output-management`).
        - `system::audio`: PipeWire-Client (`pipewire-rs`) für Audio-Management.
        - `system::mcp`: MCP-Client (`mcp_client_rs`) für KI-Modell-Kommunikation.
        - `system::portals`: Backend für XDG Desktop Portals (FileChooser, Screenshot).
        - `system::window_mechanics`: Technische Umsetzung des Fenstermanagements (Positionierung, Anwendung von Tiling-Layouts, Fokus, Fensterdekorationen). Technische Basis für die "Intelligente Tab-Leiste".
    - **Featurespiegelung:** Ermöglicht flüssige Darstellung (Wayland), präzise Eingabe (`libinput`, Gesten), Integration mit Systemdiensten für Energie, Netzwerk, Sound (PipeWire) und sichere KI-Kommunikation (MCP). Setzt Fensterregeln (Tiling, Snapping) technisch um.
4. **Benutzeroberflächenschicht (User Interface Layer):**
    
    - **Verantwortlichkeiten:** Grafische Darstellung, Benutzerinteraktion (GTK4, `gtk4-rs`).
        - `ui::shell`:
            - **Kontroll-/Systemleiste(n) (PanelWidget):** Module für AppMenu, Workspace-Indikator, Uhr, System-Tray, Schnelleinstellungen, Benachrichtigungszentrum, Netzwerk-, Energie-, Audio-Indikatoren. _Elegante Leiste mit optionalem Leuchtakzent._
            - **Intelligente Tab-Leiste (SmartTabBarWidget):** Pro "Space", mit ApplicationTabWidgets für "angepinnte" Apps/Split-Views, aktive Tabs mit Akzentfarbe. _Moderne Tabs mit abgerundeten oberen Ecken._
            - **Schnelleinstellungs-Panel (QuickSettingsPanelWidget):** Ausklappbar für WLAN, Bluetooth, Lautstärke, Dark Mode.
            - **Workspace-Switcher (WorkspaceSwitcherWidget):** Adaptive linke Seitenleiste mit SpaceIconWidgets (Icons der gepinnten App oder benutzerdefiniert) für schnelle Navigation zwischen "Spaces", mit Hervorhebung des aktiven Space. _Bei Mouse-Over/Geste aufklappbar mit Namen/Vorschau._
            - **Schnellaktionsdock (QuickActionDockWidget):** Konfigurierbares Dock (schwebend/angedockt) für Apps, Dateien, Aktionen; intelligente Vorschläge, Tastaturbedienung.
            - **Benachrichtigungszentrum (NotificationCenterPanelWidget):** Anzeige von Benachrichtigungsliste und -historie.
        - `ui::control_center`: Modulare GTK4-Anwendung für alle Systemeinstellungen (Erscheinungsbild, Netzwerk, etc.) mit Live-Vorschau.
        - `ui::widgets`:
            - **Adaptive rechte Seitenleiste (RightSidebarWidget):** Optional, mit dezent transluzentem Hintergrund für informative Widgets (Uhr, Kalender, Wetter, Systemmonitor), per Drag & Drop anpassbar.
            - WidgetManagerService, WidgetPickerPopover.
        - `ui::window_manager_frontend`:
            - **Client-Side Decorations (CSD):** Logik (z.B. via `Gtk::HeaderBar`).
            - **Übersichtsmodus (OverviewModeWidget):** Fenster- und Workspace-Übersicht als interaktive Kacheln mit Live-Vorschau, Drag & Drop von Fenstern zwischen Spaces. _Hintergrund abgedunkelt/unscharf._
            - AltTabSwitcherWidget.
        - `ui::notifications_frontend`: **Pop-up-Benachrichtigungen (NotificationPopupWidget):** Dezent, im Dark Mode Stil mit Akzentfarbe für Dringlichkeit.
        - `ui::theming_gtk`: Anwendung von CSS-Stilen aus `domain::theming` via `GtkCssProvider`.
        - `ui::speed_dial`: GTK4-Implementierung der Startansicht für leere Workspaces mit Favoriten und intelligenten Vorschlägen.
        - `ui::command_palette`: GTK4-Implementierung der kontextuellen Befehlspalette (`Super+Space`).
    - **Featurespiegelung:** Setzt die gesamte beschriebene Nutzererfahrung um: dunkle Ästhetik mit Akzentfarben, Panel(s), intelligente Tab-Leiste, adaptive Seitenleisten mit Widgets, Workspace-Switcher, Schnellaktionsdock, Control Center, Speed-Dial, Übersichtsmodus und die kontextuelle Befehlspalette. Ermöglicht die Personalisierung und direkte Manipulation.

**III. Technologie-Stack (Verbindliche Auswahl)**

Rust, Meson, GTK4 (`gtk4-rs`), Smithay Toolkit, Wayland (xdg-shell, wlr-Protokolle), D-Bus (`zbus`), Model Context Protocol (MCP), `libinput`, PipeWire (`pipewire-rs`), Freedesktop Secret Service API, PolicyKit, Token-basiertes CSS-Theming, XDG Desktop Portals.

**IV. Entwicklungsrichtlinien (Verbindlich)**

- **Rust:** `rustfmt`, Rust API Guidelines, `thiserror` pro Modul, `Result<T,E>`, `tracing` für Logging, `async/await` (Tokio, GLib).
- **Allgemein:** Git (GitHub Flow), Conventional Commits, umfassende Tests (Unit, Integration, Compositor, UI), CI-Pipeline, detaillierte Dokumentation (rustdoc, Architektur, READMEs).

**V. Deployment-Überlegungen**

Native Pakete (.deb, .rpm), Flatpak (evaluieren), Integration mit Display Managern, `systemd` User Sessions, PAM, XDG Base Directory Specification. SemVer.

**VI. Schlussfolgerung**

Diese Gesamtspezifikation legt das Fundament für NovaDE, eine moderne, performante, anpassungsfähige und KI-gestützte Desktop-Umgebung. Die klare Architektur, der robuste Technologie-Stack und die detaillierten Richtlinien, in Verbindung mit den aus der Nutzererfahrung abgeleiteten Features, ermöglichen die Entwicklung einer intuitiven und effizienten Plattform.