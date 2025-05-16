**Grundidee des Model Context Protocol (MCP):**

Ein MCP würde es verschiedenen Komponenten des Desktops (Anwendungen, Desktop-Shell, Widgets, Assistenten) ermöglichen, Informationen über den aktuellen Benutzerkontext sicher und effizient auszutauschen. "Modell" könnte sich hier auf ein Datenmodell für den Kontext oder auf KI-Modelle beziehen, die diesen Kontext nutzen.

**Phase 1: Konzeptuelle Architektur und Hypothesenformulierung**

1. **Epistemologischer Rahmen & Prämissen:**
    
    - **Ziel:** Verbesserung der Benutzererfahrung durch proaktive, kontextsensitive Unterstützung und Reduzierung repetitiver Aufgaben.
    - **Prämisse 1:** Ein standardisiertes Kontextprotokoll ist notwendig für Interoperabilität.
    - **Prämisse 2:** Benutzer müssen die volle Kontrolle über die Freigabe ihres Kontexts haben (Datenschutz).
    - **Prämisse 3:** Die Integration muss ressourcenschonend sein.
2. **Kernkonzepte & Taxonomie:**
    
    - **Context Provider:** Anwendungen (Texteditor, Browser, Kalender), Systemdienste (Standort, Netzwerk), Desktop-Shell.
    - **Context Consumer:** Desktop-Assistenten, Suchfunktionen, Automatisierungstools, App-Switcher, Benachrichtigungssysteme.
    - **Context Broker:** Eine zentrale Instanz (wahrscheinlich über D-Bus), die Kontextinformationen sammelt, filtert und verteilt.
    - **Context Data Model:** Ein standardisiertes Format (z.B. JSON-LD, ActivityStreams-ähnlich) zur Beschreibung von Kontext-Entitäten (Dokument, Aufgabe, Ort, Person, Ereignis) und deren Beziehungen.
    - **Permission Management:** System zur Verwaltung von Zugriffsrechten auf Kontextdaten.
3. **Hypothesen:**
    
    - **H1 (Sinnhaftigkeit):** Durch MCP können Anwendungen dem Benutzer relevantere Informationen und Aktionen anbieten.
    - **H2 (Benutzerfreundlichkeit):** Eine klare und granulare Kontrolle über die Kontextfreigabe erhöht die Akzeptanz.
    - **H3 (Effizienz):** MCP reduziert die Notwendigkeit für den Benutzer, Informationen manuell zwischen Anwendungen zu kopieren/übertragen.
4. **Operationalisierbare Variablen:**
    
    - Zeitersparnis bei Standardaufgaben.
    - Anzahl der Klicks/Aktionen reduziert.
    - Benutzerzufriedenheit (Umfragen).
    - Adoptionsrate des Protokolls durch Anwendungen.

**Phase 2: Systematische Literaturanalyse und Wissenskartographie**

1. **Recherche existierender Ansätze:**
    - **D-Bus:** Als zugrundeliegende IPC-Mechanismus in Linux-Desktops.
    - **Freedesktop.org-Spezifikationen:** z.B. für Benachrichtigungen, Status-Icons, MIME-Typen.
    - **Nepomuk/Baloo (KDE):** Frühere Versuche semantischer Desktops und deren Herausforderungen (Performance, Komplexität).
    - **ActivityStreams:** Web-Standard zur Beschreibung sozialer Aktivitäten, potenziell adaptierbar.
    - **Telepathy:** Framework für Echtzeitkommunikation.
    - **Mobile OS-Ansätze:** Android Intents, iOS App Intents/Shortcuts für App-Interaktion und Kontext.
2. **Identifikation von Lücken:** Aktuell kein umfassendes, desktopweites, standardisiertes Protokoll für feingranularen Anwendungskontext.

**Phase 3: Datenakquisition und Multi-Methoden-Triangulation (Design-Phase)**

Entwurf des MCP:

1. **Protokoll-Spezifikation:**
    - **Transport:** D-Bus ist die naheliegendste Wahl. Definition von D-Bus-Interfaces, -Methoden und -Signalen.
    - **Datenformat:** Z.B. JSON-basiert mit einem klaren Schema. Überlegung zu Vokabularen (Schema.org könnte Inspiration bieten).
    - **Kernkontext-Typen:** "AktivesDokument", "AusgewählterText", "AktuelleAufgabe", "Standort", "BevorstehendesEreignis", "Kommunikationspartner".
2. **API-Design:**
    - Bibliotheken (z.B. in C/GLib, Qt, Python, Vala) für Anwendungsentwickler zur einfachen Integration.
    - APIs für das Publizieren von Kontext und das Abonnieren von Kontextänderungen.
3. **Permission-Modell:**
    - Integration in bestehende Systeme (z.B. Flatpak Portals, systemweite Datenschutzeinstellungen).
    - Granulare Kontrolle: Pro Anwendung, pro Kontext-Typ.
    - Transparenz: Der Benutzer muss sehen können, welche Anwendung welchen Kontext teilt und wer darauf zugreift.

**Integration in die Linux Desktopumgebung (Sinnvoll & Benutzerfreundlich):**

1. **Zentrale Konfigurationsschnittstelle:**
    
    - Ein Modul in den Systemeinstellungen (z.B. GNOME Control Center, KDE System Settings).
    - **Benutzerfreundlich:** Klare Auflistung aller Apps, die Kontext teilen oder nutzen können. Einfache Schalter zum Aktivieren/Deaktivieren pro App und pro Kontext-Typ.
    - **Sinnvoll:** Standardeinstellungen, die einen guten Kompromiss zwischen Nutzen und Datenschutz bieten (z.B. Kontext nur mit explizit vertrauenswürdigen Systemkomponenten teilen).
2. **Integration in die Desktop-Shell (GNOME Shell, KDE Plasma, etc.):**
    
    - **Globale Suche:** Suchergebnisse basierend auf dem aktuellen Kontext priorisieren (z.B. suche "Bericht" – finde zuerst den Bericht, an dem ich gerade arbeite).
    - **Task-Switcher/Activity Overview:** Zusätzliche Kontextinformationen zu laufenden Anwendungen anzeigen.
    - **Benachrichtigungssystem:** Intelligentere Benachrichtigungen, die den aktuellen Fokus berücksichtigen (z.B. stumm schalten, wenn in Präsentation).
    - **Sinnvoll:** Macht die Shell proaktiver und informativer.
    - **Benutzerfreundlich:** Nahtlose Integration, keine zusätzliche Lernkurve.
3. **Integration in Kernanwendungen:**
    
    - **Dateimanager:** Kontextmenü-Optionen basierend auf dem globalen Kontext (z.B. "An E-Mail mit aktueller Aufgabe anhängen").
    - **Texteditor/IDE:** Code-Vervollständigung oder Dokumentationssuche basierend auf dem Projektkontext, der auch andere Tools umfasst.
    - **E-Mail-Client/Kalender:** Automatische Verknüpfung von E-Mails mit relevanten Dokumenten oder Kalendereinträgen basierend auf dem Kontext.
    - **Browser:** Vorschläge basierend auf dem Inhalt anderer aktiver Anwendungen.
    - **Sinnvoll:** Reduziert manuelle Schritte, fördert Workflows.
    - **Benutzerfreundlich:** Aktionen werden dort angeboten, wo sie gebraucht werden.
4. **Unterstützung für Desktop-Assistenten (Mycroft, Rhasspy, oder zukünftige):**
    
    - **Sinnvoll:** Ermöglicht Assistenten, wirklich "wissend" über die aktuelle Benutzeraktivität zu sein, ohne auf Screen-Scraping oder unsichere Methoden zurückgreifen zu müssen.
        - "Speichere dieses Dokument und sende einen Link an Max Mustermann bezüglich des Projekts Alpha."
        - "Erinnere mich an diese Webseite, wenn ich morgen an meinem Bericht arbeite."
    - **Benutzerfreundlich:** Natürlichsprachliche Interaktion wird mächtiger und relevanter.
5. **Entwicklerfreundlichkeit:**
    
    - **Sinnvoll:** Klare Dokumentation, Beispielimplementierungen, Test-Tools.
    - **Benutzerfreundlich (für Entwickler):** Einfach zu integrierende Bibliotheken, geringer Overhead. Anreize zur Adoption (z.B. Feature in Desktop-Umgebung hervorgehoben).
6. **Datenschutz und Sicherheit als Kernprinzip:**
    
    - **Sinnvoll & Benutzerfreundlich:**
        - **Transparenz:** Der Benutzer muss jederzeit sehen, welche Kontextdaten erfasst und von wem sie genutzt werden. Visuelle Indikatoren.
        - **Kontrolle:** Opt-in statt Opt-out für die meisten Kontextfreigaben. Möglichkeit, den Kontext-Verlauf zu löschen.
        - **Anonymisierung/Aggregation:** Wo möglich, Kontext aggregiert oder anonymisiert teilen.
        - **Sichere Speicherung:** Wenn Kontextdaten persistiert werden, dann verschlüsselt und lokal.
        - **Sandboxing:** Integration mit Flatpak/Snap-Portalen für sicheren Zugriff.

**Beispiele für sinnvolle und benutzerfreundliche Anwendungsfälle:**

- **Intelligente Zwischenablage:** Wenn Text aus einer E-Mail kopiert wird und in einen Task-Manager eingefügt wird, könnte das MCP den Betreff der E-Mail als potenziellen Task-Titel vorschlagen.
- **Automatisierte Workflows:** Beim Öffnen eines bestimmten Projektordners im Dateimanager könnte das MCP assoziierte Anwendungen (IDE, Terminal mit richtigem Pfad, Notizen-App mit Projektnotizen) vorschlagen oder automatisch starten.
- **Proaktive Hilfe:** Wenn der Benutzer in einer Anwendung auf einen Fehler stößt, könnte das MCP (mit Zustimmung) kontextrelevante Log-Dateien oder Hilfeseiten identifizieren.
- **Barrierefreiheit:** Assistive Technologien könnten den detaillierten Kontext nutzen, um Benutzern mit Einschränkungen eine reichhaltigere Interaktion zu ermöglichen.

**Herausforderungen:**

- **Standardisierung:** Einigung auf ein gemeinsames Protokoll und Datenmodell über verschiedene Desktop-Umgebungen und Toolkits hinweg. Freedesktop.org wäre die richtige Anlaufstelle.
- **Adoption:** Entwickler müssen den Mehrwert sehen, um das Protokoll in ihre Anwendungen zu integrieren.
- **Performance:** Der Kontextaustausch darf das System nicht verlangsamen.
- **Komplexität vs. Nutzen:** Das Protokoll muss mächtig genug sein, aber einfach genug für Entwickler und verständlich für Benutzer.
- **Sicherheitsrisiken:** Ein kompromittierter Context Broker könnte sensible Daten leaken.

**Fazit:**

Die Integration eines "Model Context Protocol" in eine Linux-Desktopumgebung erfordert einen durchdachten, standardisierten Ansatz mit starkem Fokus auf Benutzerkontrolle und Datenschutz. Über D-Bus als Transportmechanismus und eine klare API könnten Anwendungen und Systemdienste einen reichen, gemeinsamen Kontext aufbauen. Dies würde intelligentere, proaktivere und letztlich benutzerfreundlichere Desktop-Erfahrungen ermöglichen, indem Workflows vereinfacht, relevante Informationen zur richtigen Zeit bereitgestellt und die Effizienz gesteigert wird. Der Schlüssel zum Erfolg liegt in der offenen Zusammenarbeit, einer guten Entwicklererfahrung und dem unbedingten Schutz der Privatsphäre der Nutzer.
# Detaillierte Technische Spezifikation: KI-Integration (domain::ai)

## I. Einleitung

### A. Zweck und Geltungsbereich

Dieses Dokument spezifiziert die Implementierung des Moduls `domain::ai`, das für die Verwaltung der Interaktion der Desktop-Umgebung mit KI-Modellen verantwortlich ist.

Es legt den Schwerpunkt auf die sichere und kontrollierte Einbindung von KI-Funktionalitäten, insbesondere die Handhabung der Benutzereinwilligung bezüglich des Zugriffs auf deren Daten.

### B. Modulverantwortlichkeiten

Das `domain::ai`-Modul übernimmt folgende Aufgaben:

- Verwaltung des Lebenszyklus von KI-Interaktionskontexten.
    
    - Bereitstellung von Strukturen und Logik zur Nachverfolgung einzelner KI-Sitzungen oder Anfragen.
- Implementierung der Logik für das Einholen, Speichern und Überprüfen von Benutzereinwilligungen (AIConsent) für die Nutzung von KI-Modellen und den Zugriff auf spezifische Datenkategorien (AIDataCategory).
    
    - Definition von Mechanismen, um zu bestimmen, welche Daten für eine bestimmte KI-Aktion erforderlich sind und ob der Benutzer die Verwendung dieser Daten erlaubt hat.
- Verwaltung von Profilen verfügbarer KI-Modelle (AIModelProfile).
    
    - Katalogisierung der Fähigkeiten und Anforderungen verschiedener KI-Modelle, um eine korrekte Einwilligungsverwaltung zu gewährleisten.
- Bereitstellung einer Schnittstelle zur Initiierung von KI-Aktionen und zur Verarbeitung von deren Ergebnissen, unabhängig vom spezifischen KI-Modell oder dem MCP-Protokoll (welches in der Systemschicht implementiert wird).
    
    - Abstraktion der Kommunikation mit den KI-Modellen, um die Kompatibilität zu erhöhen und den Aufwand für andere Module zu minimieren.

### C. Nicht-Zuständigkeiten

Dieses Modul ist nicht verantwortlich für:

- Die Implementierung der UI-Elemente zur Darstellung von KI-Interaktionen oder Einwilligungsabfragen (Aufgabe der Benutzeroberflächenschicht).
    
- Die direkte Kommunikation mit KI-Modellen oder externen Diensten (Aufgabe der Systemschicht, insbesondere des MCP-Clients).
    
- Die Persistenz von Einwilligungen oder Modellprofilen (Delegiert an die Core Layer, z.B. core::config).
    

## II. Datenstrukturen

### A. Kernentitäten

1. **AIInteractionContext**
    
    - Zweck: Repräsentiert eine spezifische Interaktion oder einen Dialog mit einer KI.
        
    - Attribute:
        - `id`: `Uuid` (öffentlich): Eindeutiger Identifikator für den Kontext.
            
        - `creation_timestamp`: `DateTime<Utc>` (öffentlich): Zeitpunkt der Erstellung.
            
        - `active_model_id`: `Option<String>` (öffentlich): ID des aktuell für diesen Kontext relevanten KI-Modells.
            
        - `consent_status`: `AIConsentStatus` (öffentlich): Aktueller Einwilligungsstatus für diesen Kontext.
            
        - `associated_data_categories`: `Vec<AIDataCategory>` (öffentlich): Kategorien von Daten, die für diese Interaktion relevant sein könnten.
            
        - `interaction_history`: `Vec<String>` (privat): Eine einfache Historie der Konversation (z.B. Benutzeranfragen, KI-Antworten).
            
        - `attachments`: `Vec<AttachmentData>` (öffentlich): Angehängte Daten (z.B. Dateipfade, Text-Snippets).
            
    - Invarianten:
        - `id` und `creation_timestamp` sind nach der Erstellung unveränderlich.
            
    - Methoden (konzeptionell):
        - `new(relevant_categories: Vec<AIDataCategory>, initial_attachments: Option<Vec<AttachmentData>>) -> Self`: Erstellt einen neuen Kontext.
        - `update_consent_status(&mut self, status: AIConsentStatus)`: Aktualisiert den Einwilligungsstatus.
            
        - `set_active_model(&mut self, model_id: String)`: Legt das aktive Modell fest.
        - `add_history_entry(&mut self, entry: String)`: Fügt einen Eintrag zur Historie hinzu.
            
        - `add_attachment(&mut self, attachment: AttachmentData)`: Fügt einen Anhang hinzu.
2. **AIConsent**
    
    - Zweck: Repräsentiert die Einwilligung eines Benutzers für eine spezifische Kombination aus KI-Modell und Datenkategorien.
        
    - Attributes:
        - `id`: `Uuid` (öffentlich): Eindeutiger Identifikator für die Einwilligung.
            
        - `user_id`: `String` (öffentlich): Identifikator des Benutzers.
            
        - `model_id`: `String` (öffentlich): ID des KI-Modells, für das die Einwilligung gilt.
            
        - `data_categories`: `Vec<AIDataCategory>` (öffentlich): Datenkategorien, für die die Einwilligung erteilt wurde.
            
        - `granted_timestamp`: `DateTime<Utc>` (öffentlich): Zeitpunkt der Erteilung.
            
        - `expiry_timestamp`: `Option<DateTime<Utc>>` (öffentlich): Optionaler Ablaufzeitpunkt der Einwilligung.
            
        - `is_revoked`: `bool` (öffentlich, initial false): Gibt an, ob die Einwilligung widerrufen wurde.
            
    - Invarianten:
        - `id`, `user_id`, `model_id`, und `granted_timestamp` sind nach der Erstellung unveränderlich.
            
        - `data_categories` sollten nach der Erteilung nicht ohne expliziten Benutzerwunsch modifizierbar sein (neue Einwilligung erforderlich).
            
    - Methoden (konzeptionell):
        - `new(user_id: String, model_id: String, categories: Vec<AIDataCategory>, expiry: Option<DateTime<Utc>>) -> Self`: Erstellt eine neue Einwilligung.
        - `revoke(&mut self)`: Markiert die Einwilligung als widerrufen.
            
3. **AIModelProfile**
    
    - Zweck: Beschreibt ein verfügbares KI-Modell.
        
    - Attribute:
        - `model_id`: `String` (öffentlich): Eindeutiger Identifikator des Modells.
            
        - `display_name`: `String` (öffentlich): Anzeigename des Modells.
            
        - `description`: `String` (öffentlich): Kurze Beschreibung des Modells.
            
        - `provider`: `String` (öffentlich): Anbieter des Modells (z.B. "Local", "CloudProvider").
            
        - `required_consent_categories`: `Vec<AIDataCategory>` (öffentlich): Datenkategorien, für die dieses Modell typischerweise eine Einwilligung benötigt.
            
        - `capabilities`: `Vec<String>` (öffentlich): Liste der Fähigkeiten des Modells (z.B. "text_generation", "image_recognition").
            
    - Invarianten:
        - `model_id` ist eindeutig und unveränderlich.
            
    - Methoden (konzeptionell):
        - `new(...) -> Self`: Erstellt ein neues Modellprofil.
        - `requires_consent_for(&self, categories: &Vec<AIDataCategory>) -> bool`: Prüft, ob für die gegebenen Kategorien eine Einwilligung erforderlich ist.
            
4. **Notification**
    
    - Zweck: Repräsentiert eine einzelne Benachrichtigung.
        
    - Attribute:
        - `id`: `Uuid` (öffentlich): Eindeutiger Identifikator.
            
        - `application_name`: `String` (öffentlich): Name der Anwendung, die die Benachrichtigung gesendet hat.
            
        - `application_icon`: `Option<String>` (öffentlich): Optionaler Pfad oder Name des Icons der Anwendung.
            
        - `summary`: `String` (öffentlich): Kurze Zusammenfassung der Benachrichtigung.
            
        - `body`: `Option<String>` (öffentlich): Detaillierterer Text der Benachrichtigung.
            
        - `actions`: `Vec<NotificationAction>` (öffentlich): Verfügbare Aktionen für die Benachrichtigung.
            
        - `urgency`: `NotificationUrgency` (öffentlich): Dringlichkeitsstufe.
            
        - `timestamp`: `DateTime<Utc>` (öffentlich): Zeitpunkt des Eintreffens.
            
        - `is_read`: `bool` (privat, initial false): Status, ob gelesen.
            
        - `is_dismissed`: `bool` (privat, initial false): Status, ob vom Benutzer aktiv geschlossen.
            
        - `transient`: `bool` (öffentlich, default false): Ob die Benachrichtigung flüchtig ist und nicht in der Historie verbleiben soll.
            
    - Invarianten:
        - `id` und `timestamp` sind unveränderlich.
            
        - `summary` darf nicht leer sein.
            
    - Methoden (konzeptionell):
        - `new(app_name: String, summary: String, urgency: NotificationUrgency) -> Self`: Erstellt eine neue Benachrichtigung.
            
        - `mark_as_read(&mut self)`: Setzt den Lesestatus.
        - `dismiss(&mut self)`: Setzt den Entlassen-Status.
        - `add_action(&mut self, action: NotificationAction)`: Fügt eine Aktion hinzu.
            
5. **NotificationAction**
    
    - Zweck: Definiert eine Aktion, die im Kontext einer Benachrichtigung ausgeführt werden kann.
        
    - Attribute:
        - `key`: `String` (öffentlich): Eindeutiger Schlüssel für die Aktion (z.B. "reply", "archive").
            
        - `label`: `String` (öffentlich): Anzeigename der Aktion.
            
        - `action_type`: `NotificationActionType` (öffentlich): Typ der Aktion (z.B. Callback, Link).
            
6. **AttachmentData**
    
    - Zweck: Repräsentiert angehängte Daten an einen AIInteractionContext.
        
    - Attribute:
        - `id`: `Uuid` (öffentlich): Eindeutiger Identifikator des Anhangs.
            
        - `mime_type`: `String` (öffentlich): MIME-Typ der Daten (z.B. "text/plain", "image/png").
            
        - `source_uri`: `Option<String>` (öffentlich): URI zur Quelle der Daten (z.B. file:///path/to/file).
            
        - `content`: `Option<Vec<u8>>` (öffentlich): Direkter Inhalt der Daten, falls klein.
            
        - `description`: `Option<String>` (öffentlich): Optionale Beschreibung des Anhangs.
            

### B. Modulspezifische Enums

1. **AIConsentStatus**: Enum
    
    - Varianten: `Granted`, `Denied`, `PendingUserAction`, `NotRequired`.
        
2. **AIDataCategory**: Enum
    
    - Varianten: `UserProfile`, `ApplicationUsage`, `FileSystemRead`, `ClipboardAccess`, `LocationData`, `GenericText`, `GenericImage`.
        
3. **NotificationUrgency**: Enum
    
    - Varianten: `Low`, `Normal`, `Critical`.
        
4. **NotificationActionType**: Enum
    
    - Varianten: `Callback`, `OpenLink`.
        
5. **NotificationFilterCriteria**: Enum
    
    - Varianten: `Unread`, `Application(String)`, `Urgency(NotificationUrgency)`.
        
6. **NotificationSortOrder**: Enum
    
    - Varianten: `TimestampAscending`, `TimestampDescending`, `Urgency`.
        

### C. Modulspezifische Konstanten

- `const DEFAULT_NOTIFICATION_TIMEOUT_SECS: u64 = 5;`
    
- `const MAX_NOTIFICATION_HISTORY: usize = 100;`
    
- `const MAX_AI_INTERACTION_HISTORY: usize = 50;`
    

## III. Modulspezifische Funktionen

### A. Traits

1. **AIInteractionLogicService**
    
    Rust
    
    ```
    use crate::core::types::Uuid;
    use crate::core::errors::CoreError;
    use super::types::{AIInteractionContext, AIConsent, AIModelProfile, AIDataCategory, AttachmentData};
    use super::errors::AIInteractionError;
    use async_trait::async_trait;
    
    #[async_trait]
    pub trait AIInteractionLogicService: Send + Sync {
        /// Initiates a new AI interaction context.
        /// Returns the ID of the newly created context.
        async fn initiate_interaction(
            &mut self,
            relevant_categories: Vec<AIDataCategory>,
            initial_attachments: Option<Vec<AttachmentData>>
        ) -> Result<Uuid, AIInteractionError>;
    
        /// Retrieves an existing AI interaction context.
        async fn get_interaction_context(&self, context_id: Uuid) -> Result<AIInteractionContext, AIInteractionError>;
    
        /// Provides or updates consent for a given interaction context and model.
        async fn provide_consent(
            &mut self,
            context_id: Uuid,
            model_id: String,
            granted_categories: Vec<AIDataCategory>,
            consent_decision: bool // true for granted, false for denied
        ) -> Result<(), AIInteractionError>;
    
        /// Retrieves the consent status for a specific model and data categories,
        /// potentially within an interaction context.
        async fn get_consent_for_model(
            &self,
            model_id: &str,
            data_categories: &Vec<AIDataCategory>,
            context_id: Option<Uuid>
        ) -> Result<AIConsentStatus, AIInteractionError>;
    
        /// Adds an attachment to an existing interaction context.
        async fn add_attachment_to_context(
            &mut self,
            context_id: Uuid,
            attachment: AttachmentData
        ) -> Result<(), AIInteractionError>;
    
        /// Lists all available and configured AI model profiles.
        async fn list_available_models(&self) -> Result<Vec<AIModelProfile>, AIInteractionError>;
    
        /// Stores a user's consent decision persistently.
        /// This might be called after `provide_consent` if the consent is to be remembered globally.
        async fn store_consent(&self, consent: AIConsent) -> Result<(), AIInteractionError>;
    
        /// Retrieves all stored consents for a given user (simplified).
        async fn get_all_user_consents(&self, user_id: &str) -> Result<Vec<AIConsent>, AIInteractionError>;
    
        /// Loads AI model profiles, e.g., from a configuration managed by core::config.
        async fn load_model_profiles(&mut self) -> Result<(), AIInteractionError>;
    }
    ```
    
2. **NotificationService**
    
    Rust
    
    ```
    use crate::core::types::Uuid;
    use crate::core::errors::CoreError;
    use super::types::{Notification, NotificationUrgency, NotificationFilterCriteria, NotificationSortOrder};
    use super::errors::NotificationError;
    use async_trait::async_trait;
    
    #[async_trait]
    pub trait NotificationService: Send + Sync {
        /// Posts a new notification to the system.
        /// Returns the ID of the newly created notification.
        async fn post_notification(&mut self, notification_data: Notification) -> Result<Uuid, NotificationError>;
    
        /// Retrieves a specific notification by its ID.
        async fn get_notification(&self, notification_id: Uuid) -> Result<Notification, NotificationError>;
    
        /// Marks a notification as read.
        async fn mark_as_read(&mut self, notification_id: Uuid) -> Result<(), NotificationError>;
    
        /// Dismisses a notification, removing it from active view but possibly keeping it in history.
        async fn dismiss_notification(&mut self, notification_id: Uuid) -> Result<(), NotificationError>;
    
        /// Retrieves a list of currently active (not dismissed, potentially unread) notifications.
        /// Allows filtering and sorting.
        async fn get_active_notifications(
            &self,
            filter: Option<NotificationFilterCriteria>,
            sort_order: Option<NotificationSortOrder>
        ) -> Result<Vec<Notification>, NotificationError>;
    
        /// Retrieves the notification history.
        /// Allows filtering and sorting.
        async fn get_notification_history(
            &self,
            limit: Option<usize>,
            filter: Option<NotificationFilterCriteria>,
            sort_order: Option<NotificationSortOrder>
        ) -> Result<Vec<Notification>, NotificationError>;
    
        /// Clears all notifications from history.
        async fn clear_history(&mut self) -> Result<(), NotificationError>;
    
        /// Sets the "Do Not Disturb" mode.
        async fn set_do_not_disturb(&mut self, enabled: bool) -> Result<(), NotificationError>;
    
        /// Checks if "Do Not Disturb" mode is currently enabled.
        async fn is_do_not_disturb_enabled(&self) -> Result<bool, NotificationError>;
    
        /// Invokes a specific action associated with a notification.
        async fn invoke_action(&mut self, notification_id: Uuid, action_key: &str) -> Result<(), NotificationError>;
    }
    ```
    

### B. Methodenlogik

1. **AIInteractionLogicService::provide_consent**
    
    - Vorbedingung:
        - `context_id` muss einen existierenden AIInteractionContext referenzieren.
            
        - `model_id` muss einem bekannten AIModelProfile entsprechen.
            
    - Logik:
        1. Kontext und Modellprofil laden.
        2. Prüfen, ob die `granted_categories` eine Untermenge der vom Modell potenziell benötigten Kategorien sind.
            
        3. Einen neuen `AIConsent`-Eintrag erstellen oder einen bestehenden aktualisieren.
        4. Den `consent_status` im `AIInteractionContext` entsprechend anpassen.
            
        5. Falls `consent_decision` true ist und die Einwilligung global gespeichert werden soll, `store_consent()` aufrufen.
        6. `AIConsentUpdatedEvent` auslösen.
    - Nachbedingung:
        - Der Einwilligungsstatus des Kontexts ist aktualisiert.
        - Ein `AIConsent`-Objekt wurde potenziell erstellt/modifiziert.
        - Ein Event wurde ausgelöst.
            
2. **NotificationService::post_notification**
    
    - Vorbedingung:
        - `notification_data.summary` darf nicht leer sein.
            
    - Logik:
        1. Validieren der `notification_data`.
        2. Der `Notification` eine neue Uuid und einen `timestamp` zuweisen.
            
        3. Wenn DND-Modus aktiv ist und die `NotificationUrgency` nicht Critical ist, die Benachrichtigung ggf. unterdrücken oder nur zur Historie hinzufügen, ohne sie aktiv anzuzeigen.
            
        4. Die Benachrichtigung zur Liste der `active_notifications` hinzufügen.
        5. Wenn die Benachrichtigung nicht transient ist, sie zur `history` hinzufügen (unter Beachtung von `MAX_NOTIFICATION_HISTORY`).
        6. `NotificationPostedEvent` auslösen (ggf. mit Information, ob sie aufgrund von DND unterdrückt wurde).
    - Nachbedingung:
        - Die Benachrichtigung ist im System registriert und ein Event wurde ausgelöst.
            

## IV. Fehlerbehandlung

### A. AIInteractionError

Rust

```
use thiserror::Error;
use crate::core::types::Uuid;

pub enum AIInteractionError {
    ContextNotFound(Uuid),
    ConsentAlreadyProvided(Uuid),
    #[error("Consent required for model '{model_id}' but not granted for data categories: {missing_categories:?}")]
    ConsentRequired { model_id: String, missing_categories: Vec<String> },
    NoModelAvailable,
    ModelNotFound(String),
    InvalidAttachment(String),
    ConsentStorageError(String),
    ModelProfileLoadError(String),
    CoreError { #[from] source: crate::core::errors::CoreError },
    InternalError(String),
}
```

### B. NotificationError

Rust

```
use thiserror::Error;
use crate::core::types::Uuid;

pub enum NotificationError {
    NotFound(Uuid),
    InvalidData{ summary: String, details: String },
    #[error("Maximum notification history of {max_history} reached. Cannot add new notification: {summary}")]
    HistoryFull { max_history: usize, summary: String },
    ActionNotFound { notification_id: Uuid, action_id: String },
    CoreError { #[from] source: crate::core::errors::CoreError },
    InternalError(String),
}
```

## V. Ereignisse

### A. AIInteractionInitiatedEvent

- Payload-Struktur:
    
    Rust
    
    ```
    pub struct AIInteractionInitiatedEvent {
        pub context_id: Uuid,
        pub relevant_categories: Vec<AIDataCategory>
    }
    ```
    
- Typische Publisher: AIInteractionLogicService Implementierung.
    
- Typische Subscriber: UI-Komponenten, die eine KI-Interaktionsoberfläche öffnen oder vorbereiten; Logging-Systeme.
    
- Auslösebedingungen: Ein neuer AIInteractionContext wurde erfolgreich erstellt via initiate_interaction.
    

### B. AIConsentUpdatedEvent

- Payload-Struktur:
    
    Rust
    
    ```
    pub struct AIConsentUpdatedEvent {
        pub context_id: Option<Uuid>,
        pub model_id: String,
        pub granted_categories: Vec<AIDataCategory>,
        pub consent_status: AIConsentStatus
    }
    ```
    
- Typische Publisher: AIInteractionLogicService Implementierung.
    
- Typische Subscriber: UI-Komponenten, die den Einwilligungsstatus anzeigen oder Aktionen basierend darauf freischalten/sperren; die Komponente, die die eigentliche KI-Anfrage durchführt.
    
- Auslösebedingungen: Eine Einwilligung wurde erteilt, verweigert oder widerrufen (provide_consent, store_consent mit Widerruf).
    

### C. NotificationPostedEvent

- Payload-Struktur:
    
    Rust
    
    ```
    pub struct NotificationPostedEvent {
        pub notification: Notification,
        pub suppressed_by_dnd: bool
    }
    ```
    
- Typische Publisher: NotificationService Implementierung.
    
- Typische Subscriber: UI-Schicht (zur Anzeige der Benachrichtigung), System-Schicht (z.B. um einen Ton abzuspielen, falls nicht unterdrückt).
    
- Auslösebedingungen: Eine neue Benachrichtigung wurde erfolgreich via post_notification verarbeitet.
    

### D. NotificationDismissedEvent

- Payload-Struktur:
    
    Rust
    
    ```
    pub struct NotificationDismissedEvent {
        pub notification_id: Uuid
    }
    ```
    
- Typische Publisher: NotificationService Implementierung.
    
- Typische Subscriber: UI-Schicht (um die Benachrichtigung aus der aktiven Ansicht zu entfernen).
    
- Auslösebedingungen: Eine Benachrichtigung wurde erfolgreich via dismiss_notification geschlossen.
    

### E. NotificationReadEvent

- Payload-Struktur:
    
    Rust
    
    ```
    pub struct NotificationReadEvent {
        pub notification_id: Uuid
    }
    ```
    
- Typische Publisher: NotificationService Implementierung.
    
- Typische Subscriber: UI-Schicht (um den "gelesen"-Status zu aktualisieren).
    
- Auslösebedingungen: Eine Benachrichtigung wurde erfolgreich via mark_as_read als gelesen markiert.
    

### F. DoNotDisturbModeChangedEvent

- Payload-Struktur:
    
    Rust
    
    ```
    pub struct DoNotDisturbModeChangedEvent {
        pub dnd_enabled: bool
    }
    ```
    
- Typische Publisher: NotificationService Implementierung.
    
- Typische Subscriber: UI (DND-Statusanzeige), NotificationService (um Benachrichtigungen zu unterdrücken).
    
- Auslösebedingungen: Der DND-Modus wurde via set_do_not_disturb geändert.
    

## VI. Implementierungsrichtlinien

### A. Modulstruktur

```
src/domain/user_centric_services/
├── mod.rs                      // Deklariert Submodule, exportiert öffentliche Typen/Traits
├── ai_interaction_service.rs  // Implementierung von AIInteractionLogicService
├── notification_service.rs    // Implementierung von NotificationService
├── types.rs                    // Gemeinsame Enums und Structs
└── errors.rs                   // Definition der Fehler-Enums
```

### B. Implementierungsschritte

1. **errors.rs erstellen**: Definiere die AIInteractionError und NotificationError Enums mithilfe von `thiserror`. Stelle sicher, dass sie `Debug`, `Clone`, `PartialEq`, und `Eq` (falls benötigt) implementieren.
2. **types.rs erstellen**: Definiere alle modulspezifischen Enums (AIConsentStatus, AIDataCategory, etc.) und Structs (AIInteractionContext, AIConsent, etc.). Implementiere für diese Strukturen die notwendigen Traits: `Debug`, `Clone`, `PartialEq`, und `Serialize`/`Deserialize` (wo benötigt).
3. **ai_interaction_service.rs Basis**:
    - Definiere den Trait `AIInteractionLogicService`.
    - Erstelle eine Struktur `DefaultAIInteractionLogicService`. Diese Struktur wird Felder für den internen Zustand enthalten.
    - Beginne mit der Implementierung von `#[async_trait] impl AIInteractionLogicService for DefaultAIInteractionLogicService`.
4. **notification_service.rs Basis**:
    - Definiere den Trait `NotificationService`.
    - Erstelle eine Struktur `DefaultNotificationService`. Diese Struktur wird Felder für den internen Zustand enthalten.
    - Beginne mit der Implementierung von `#[async_trait] impl NotificationService for DefaultNotificationService`.
5. **Implementierung der AIInteractionLogicService-Methoden**: Implementiere jede Methode des Traits schrittweise. Achte auf korrekte Fehlerbehandlung und Rückgabe der definierten `AIInteractionError`-Varianten. Implementiere die Interaktion mit der Kernschicht (z.B. für Persistenz). Löse die entsprechenden Events aus.
6. **Implementierung der NotificationService-Methoden**: Implementiere jede Methode des Traits. Implementiere die Logik für DND, Historienbegrenzung, Filterung und Sortierung. Verwende `NotificationError`-Varianten für Fehlerfälle. Löse die spezifizierten Notification-Events aus.
7. **mod.rs erstellen**: Deklariere die Submodule und exportiere alle öffentlichen Typen, Traits, und Fehler-Enums, die von außerhalb dieses Moduls verwendet werden sollen.
8. **Unit-Tests**: Schreibe Unit-Tests parallel zur Implementierung jeder Methode und jeder komplexen Logikeinheit. Mocke dabei gegebenenfalls Abhängigkeiten zur Kernschicht.
# Executive Summary

Purpose and Scope: Dieses Dokument liefert eine Ultra-Feinspezifikation für sämtliche Schnittstellen und Implementierungen des Model Context Protocol (MCP) innerhalb des NovaDE-Projekts. Es dient als definitive technische Referenz für Entwickler und Architekten, die an der Integration von MCP beteiligt sind. Die Spezifikation zielt darauf ab, eine klare, präzise und unzweideutige Grundlage für die Entwicklung zu schaffen, die eine direkte Umsetzung ermöglicht.


MCP in NovaDE: Die strategische Entscheidung zur Adaption von MCP im NovaDE-Projekt basiert auf der Erwartung signifikanter Vorteile. Dazu zählen die standardisierte Integration von KI-Modellen, eine verbesserte kontextuelle Wahrnehmung für KI-Agenten und der modulare Zugriff auf die domänenspezifischen Funktionalitäten von NovaDE.1 MCP positioniert NovaDE so, dass es von einem wachsenden Ökosystem an KI-Werkzeugen und -Modellen profitieren kann, indem eine standardisierte Interaktionsebene bereitgestellt wird.1 Diese Ausrichtung deutet auf eine zukunftsorientierte Architektur hin, die auf Interoperabilität und Erweiterbarkeit abzielt. Da MCP als universeller Standard gilt und von führenden KI-Akteuren adaptiert wird 1, kann NovaDE durch dessen Nutzung einfacher mit diversen KI-Modellen integriert werden und von gemeinschaftlich entwickelten MCP-Servern oder -Clients profitieren.


Key Deliverables: Diese Spezifikation umfasst detaillierte MCP-Nachrichtenformate, NovaDE-spezifische Schnittstellendefinitionen (Ressourcen, Werkzeuge, Aufforderungen, Benachrichtigungen), Integrationsstrategien mit der (aktuell separaten) "Domänenschicht-Spezifikation", Implementierungsrichtlinien, Sicherheitsüberlegungen, Fehlerbehandlung und Versionierung.


Critical Dependencies: Es wird explizit auf die Abhängigkeit von der "Domänenschicht-Spezifikation" für die konkrete Abbildung von Domänenfunktionalitäten auf MCP-Konstrukte hingewiesen. Dieses Dokument stellt den Rahmen für solche Abbildungen bereit. Der Erfolg der MCP-Integration hängt maßgeblich von einer wohldefinierten "Domänenschicht-Spezifikation" ab; ohne diese bleiben die MCP-Schnittstellen abstrakt.


Intended Audience: Dieses Dokument richtet sich an technische Leiter, Softwarearchitekten und Senior-Entwickler des NovaDE-Projekts.

2. Model Context Protocol (MCP) Grundlagen für NovaDE

2.1. MCP Protokollübersicht

Definition und Ziele: Das Model Context Protocol (MCP) ist ein offener Standard, der entwickelt wurde, um die Art und Weise zu standardisieren, wie KI-Modelle, insbesondere Large Language Models (LLMs), mit externen Werkzeugen, Systemen und Datenquellen integriert werden und Daten austauschen.1 Es fungiert als universelle Schnittstelle für den Kontexaustausch zwischen KI-Assistenten und Software-Umgebungen, indem es modellagnostische Mechanismen zum Lesen von Dateien, Ausführen von Funktionen und Handhaben kontextueller Anfragen bereitstellt.1 Das primäre Ziel von MCP ist es, die Herausforderung isolierter Informationssilos und proprietärer Legacy-Systeme zu adressieren, die die Fähigkeiten selbst hochentwickelter KI-Modelle einschränken.1
Kernkonzepte:

Client-Host-Server-Architektur: MCP basiert auf einem Client-Host-Server-Muster.2

MCP Clients: Sind Protokoll-Clients, die typischerweise in KI-Anwendungen oder Agenten eingebettet sind und eine Eins-zu-Eins-Verbindung zu MCP-Servern herstellen. Sie sind für die Aushandlung von Fähigkeiten und die Orchestrierung von Nachrichten zwischen sich und dem Server zuständig.2
MCP Hosts: Agieren als Container oder Koordinatoren für eine oder mehrere Client-Instanzen. Sie verwalten den Lebenszyklus und die Sicherheitsrichtlinien (z.B. Berechtigungen, Benutzerautorisierung, Durchsetzung von Einwilligungsanforderungen) und überwachen, wie die KI-Integration innerhalb jedes Clients erfolgt, indem sie Kontext sammeln und zusammenführen.2 Ein Beispiel hierfür ist die Claude Desktop App.1
MCP Server: Sind Programme, die Datenquellen, APIs oder andere Dienstprogramme (wie CRM-Systeme, Git-Repositories oder Dateisysteme) umschließen und deren Fähigkeiten über die standardisierte MCP-Schnittstelle bereitstellen. Sie müssen Sicherheitsbeschränkungen und Benutzerberechtigungen, die vom Host durchgesetzt werden, einhalten.2


Ressourcen (Resources): Stellen Dateneinheiten dar, die von MCP-Servern exponiert werden. Sie können beliebige Entitäten sein – Dateien, API-Antworten, Datenbankabfragen, Systeminformationen etc..5 Sie sind vergleichbar mit GET-Endpunkten in einer Web-API und dienen dazu, Informationen in den Kontext des LLMs zu laden.6
Werkzeuge (Tools): Repräsentieren Funktionalitäten, die von MCP-Servern bereitgestellt werden und von LLMs aufgerufen werden können, um Aktionen auszuführen oder Berechnungen durchzuführen.3 Im Gegensatz zu Ressourcen wird von Werkzeugen erwartet, dass sie Seiteneffekte haben können. Sie sind vergleichbar mit POST-Endpunkten in einer REST-API.6
Aufforderungen (Prompts): Definieren wiederverwendbare Interaktionsmuster oder Vorlagen für LLM-Interaktionen, die Systemanweisungen, erforderliche Argumente, eingebettete Ressourcen und verschiedene Inhaltstypen umfassen können.5
Benachrichtigungen (Notifications): Sind asynchrone Nachrichten, die von einem MCP-Server an einen MCP-Client gesendet werden, typischerweise um über Zustandsänderungen oder Ereignisse zu informieren, ohne dass eine direkte vorherige Anfrage vom Client erfolgte.5


JSON-RPC Basis: MCP basiert auf JSON-RPC 2.0.2 Dies impliziert ein etabliertes Nachrichtenformat für Anfragen (Requests), Antworten (Responses) und Benachrichtigungen (Notifications), was die Implementierung und Interoperabilität erleichtert.



2.2. MCP-Architektur im NovaDE-Projekt

Identifizierung von MCP-Komponenten:

MCP Hosts in NovaDE: Es ist zu definieren, welche Komponenten des NovaDE-Projekts als MCP Hosts agieren werden. Dies könnte beispielsweise ein zentraler KI-Agenten-Orchestrator sein, der die Interaktionen zwischen verschiedenen KI-Modellen und den NovaDE MCP-Servern koordiniert und Sicherheitsrichtlinien durchsetzt, wie in 2 beschrieben.
MCP Server in NovaDE: Module oder Subsysteme von NovaDE, die spezifische Domänenfunktionalitäten oder Datenzugriffe bereitstellen, werden als MCP-Server implementiert. Diese Server exponieren dann über MCP definierte Ressourcen und Werkzeuge.
MCP Clients in NovaDE: Potenzielle MCP-Clients können interne KI-Agenten des NovaDE-Projekts sein oder auch externe KI-Modelle, die mit den Funktionalitäten von NovaDE interagieren sollen.


Transportmechanismen:

Stdio (Standard Input/Output): Dieser Mechanismus eignet sich für die lokale Interprozesskommunikation zwischen eng gekoppelten Komponenten innerhalb von NovaDE.3 Rust SDKs wie mcp_client_rs 7 und mcpr 10 unterstützen Stdio. Für Szenarien, in denen ein NovaDE-Host einen lokalen MCP-Server als Subprozess startet, ist Stdio eine einfache und effiziente Wahl.
HTTP/SSE (Server-Sent Events): Für die Kommunikation mit entfernten MCP-Servern oder wenn Echtzeit-Updates vom Server zum Client erforderlich sind (z.B. Benachrichtigungen über Änderungen in der Domänenschicht), ist HTTP mit SSE der empfohlene Transportmechanismus.3 Das mcpr Rust SDK 10 bietet explizite Unterstützung für SSE, einschließlich Mock-Implementierungen für Tests. Auch mcp-go unterstützt SSE.6 Die Fähigkeit, Server-Push-Benachrichtigungen zu empfangen, ist für viele KI-Anwendungen entscheidend, was SSE favorisiert.
Rationale für die Wahl: Die Auswahl des Transportmechanismus pro Komponente in NovaDE sollte auf den spezifischen Anforderungen basieren. Für eng integrierte lokale Prozesse, die keine unidirektionalen Echtzeit-Updates vom Server benötigen, kann Stdio ausreichend sein. Für alle Szenarien, die Server-Push-Benachrichtigungen oder die Anbindung externer/entfernter MCP-Server erfordern, sollte HTTP/SSE verwendet werden. Die "Domänenschicht-Spezifikation" muss analysiert werden, um festzustellen, welche Funktionalitäten asynchrone Updates erfordern, was die Wahl des Transports und potenziell des MCP-Server-SDKs für diese Teile leitet.


Datenflussdiagramme:

Diagramm 2.2.1: Allgemeiner MCP-Datenfluss in NovaDE (Illustriert einen NovaDE MCP Host, der mit einem internen NovaDE MCP Server und einem externen KI-Modell (Client) kommuniziert.)
Diagramm 2.2.2: Datenfluss für Werkzeugaufruf über Stdio
Diagramm 2.2.3: Datenfluss für Ressourcenabruf und Benachrichtigung über SSE



Die Unterscheidung zwischen Client, Host und Server im MCP-Modell 2 erfordert eine sorgfältige Zuweisung dieser Rollen innerhalb der NovaDE-Architektur. Der Host als Koordinator und Durchsetzer von Sicherheitsrichtlinien ist eine zentrale Komponente, insbesondere wenn mehrere KI-Agenten oder Clients mit verschiedenen NovaDE MCP-Servern interagieren. Das Design dieser Host-Komponente(n) wird entscheidend für die Sicherheit und Verwaltbarkeit des Gesamtsystems sein.

3. Standard-MCP-Nachrichtenspezifikationen für NovaDEDieser Abschnitt definiert die präzisen JSON-RPC 2.0 Strukturen für alle Standard-MCP-Nachrichten, angepasst mit NovaDE-spezifischen Überlegungen, wie beispielsweise gemeinsamen Metadatenfeldern. Die hier definierten Strukturen basieren auf den allgemeinen MCP-Konzepten 3 und werden durch spezifische Felder für den NovaDE-Kontext erweitert.

3.1. Initialize Request und ResponseDie Initialize-Nachricht dient dem Aufbau einer Verbindung und dem Aushandeln von Protokollversionen und Fähigkeiten zwischen Client und Server.3

InitializeParams: Parameter für den Initialize-Request.

Tabelle 3.1: InitializeParams Schema




FeldnameJSON-TypBeschreibungConstraintsprotocolVersionstringDie vom Client vorgeschlagene MCP-Protokollversion (z.B. "2025-03-26").ErforderlichclientNamestringOptionaler, menschenlesbarer Name der Client-Anwendung/Komponente.OptionalclientVersionstringOptionale Version der Client-Anwendung/Komponente.OptionalsupportedFeaturesarray of stringOptionale Liste von NovaDE-spezifischen MCP-Features, die der Client unterstützt.Optional*   **`InitializeResult`**: Ergebnis eines erfolgreichen Initialize-Requests.
    *   **Tabelle 3.2**: `InitializeResult` Schema
FeldnameJSON-TypBeschreibungConstraintsprotocolVersionstringDie vom Server gewählte und unterstützte MCP-Protokollversion.ErforderlichserverNamestringOptionaler, menschenlesbarer Name der Server-Anwendung/Komponente.OptionalserverVersionstringOptionale Version der Server-Anwendung/Komponente.OptionalsupportedFeaturesarray of stringOptionale Liste von NovaDE-spezifischen MCP-Features, die der Server unterstützt.Optionaltoolsarray of ToolDefinitionOptionale initiale Liste der vom Server bereitgestellten Werkzeuge.Optional, siehe Tabelle 3.8 für ToolDefinitionresourcesarray of ResourceDefinitionOptionale initiale Liste der vom Server bereitgestellten Ressourcen.Optional, Struktur analog zu Resource (Tabelle 3.5) aber ggf. ohne content    *Referenzen*: Die `mcp_client_rs` Bibliothek nutzt eine `spawn_and_initialize` Methode [9], und `mcpr` bietet eine `client.initialize()` Funktion [10], was die fundamentale Rolle dieser Nachricht unterstreicht.


3.2. ListResources Request und ResponseDiese Nachricht ermöglicht es einem Client, die vom Server verfügbaren Ressourcen abzufragen.3

ListResourcesParams: Parameter für den ListResources-Request.

Tabelle 3.3: ListResourcesParams Schema




FeldnameJSON-TypBeschreibungConstraintsfilterobjectOptionale, NovaDE-spezifische Kriterien zur Filterung der Ressourcen (z.B. nach Typ, Domänenentität).OptionalpageTokenstringOptionales Token zur Paginierung, um die nächste Seite der Ergebnisse abzurufen.Optional*   **`ListResourcesResult`**: Ergebnis eines erfolgreichen ListResources-Requests.
    *   **Tabelle 3.4**: `ListResourcesResult` Schema
FeldnameJSON-TypBeschreibungConstraintsresourcesarray of ResourceListe der Resource-Objekte, die den Filterkriterien entsprechen.Erforderlich, siehe Tabelle 3.5 für ResourcenextPageTokenstringOptionales Token, um die nächste Seite der Ergebnisse abzurufen, falls vorhanden.Optional*   **`Resource` Objektstruktur**: Definiert die Struktur einer einzelnen Ressource.
    *   **Tabelle 3.5**: `Resource` Objekt Schema
FeldnameJSON-TypBeschreibungConstraintsDomänenschicht-Mapping (Beispiel)uristringEindeutiger Resource Identifier (URI).ErforderlichDomainObject.IDnamestringMenschenlesbarer Name der Ressource.ErforderlichDomainObject.DisplayNamedescriptionstringOptionale, detaillierte Beschreibung der Ressource.OptionalDomainObject.DescriptionschemaobjectOptionales JSON-Schema, das die Datenstruktur des Ressourceninhalts beschreibt.Optional-novaDE_domain_typestringOptionaler Typbezeichner, der auf einen Typ in der "Domänenschicht-Spezifikation" verweist.OptionalName des DomänentypscontentTypestringOptionaler MIME-Typ oder NovaDE-spezifischer Inhaltstyp.OptionalDomainObject.MimeTypecanReadbooleanGibt an, ob die Ressource gelesen werden kann.Optional-canWritebooleanGibt an, ob die Ressource geschrieben werden kann (falls zutreffend).Optional-    *Referenzen*: Die `list_resources()` Methode in `mcp_client_rs` [9] und das allgemeine Konzept von Ressourcen in MCP [6] sind hier relevant.


3.3. CallTool Request und ResponseDiese Nachricht ermöglicht es einem Client, ein vom Server bereitgestelltes Werkzeug auszuführen.3

CallToolParams: Parameter für den CallTool-Request.

Tabelle 3.6: CallToolParams Schema




FeldnameJSON-TypBeschreibungConstraintstoolNamestringName des aufzurufenden Werkzeugs.ErforderlichargumentsobjectJSON-Objekt, das die Argumente für das Werkzeug enthält.ErforderlichprogressTokenstringOptionales Token zur Verfolgung des Fortschritts langlaufender Werkzeuge.Optional*   **`CallToolResult`**: Ergebnis eines erfolgreichen CallTool-Requests.
    *   **Tabelle 3.7**: `CallToolResult` Schema
FeldnameJSON-TypBeschreibungConstraintsresultanyOptionale Ausgabe der Werkzeugausführung. Die Struktur hängt vom Werkzeug ab.OptionalisErrorbooleanOptional. Gibt an, ob der Werkzeugaufruf zu einem anwendungsspezifischen Fehler geführt hat (Standard: false).Optional, Default falseerrorobjectOptionale, werkzeugspezifische Fehlerdetails, falls isError true ist.Optional*   **`ToolDefinition` Objektstruktur**: Definiert die Struktur eines Werkzeugs (verwendet in `InitializeResult` und potenziell in einer `ListTools`-Antwort).
    *   **Tabelle 3.8**: `ToolDefinition` Objekt Schema
FeldnameJSON-TypBeschreibungConstraintsDomänenschicht-Mapping (Beispiel)namestringEindeutiger Name des Werkzeugs.ErforderlichDomainFunction.NamedescriptionstringOptionale, menschenlesbare Beschreibung des Werkzeugs.OptionalDomainFunction.DocparametersSchemaobjectJSON-Schema, das die Eingabeparameter (arguments) des Werkzeugs beschreibt.Erforderlich-resultSchemaobjectOptionales JSON-Schema, das die erfolgreiche Ausgabe (result) des Werkzeugs beschreibt.Optional-novaDE_domain_functionstringOptionaler Bezeichner, der auf eine Funktion/Fähigkeit in der "Domänenschicht-Spezifikation" verweist.OptionalName der Domänenfunktion    *Referenzen*: Die `call_tool()` Methode in `mcp_client_rs` [9] und das Werkzeugkonzept in MCP [3, 6] sind hier relevant.


3.4. ReadResource Request und ResponseErmöglicht das Lesen des Inhalts einer spezifischen Ressource.

ReadResourceParams: Parameter für den ReadResource-Request.

Tabelle 3.9: ReadResourceParams Schema




FeldnameJSON-TypBeschreibungConstraintsuristringURI der zu lesenden Ressource.Erforderlich*   **`ReadResourceResult`**: Ergebnis eines erfolgreichen ReadResource-Requests.
    *   **Tabelle 3.10**: `ReadResourceResult` Schema
FeldnameJSON-TypBeschreibungConstraintscontentanyDer Inhalt der Ressource, konform zu ihrem Schema (falls definiert).ErforderlichcontentTypestringOptionaler MIME-Typ oder NovaDE-spezifischer Inhaltstyp der Ressource.Optional    *Referenzen*: Die `read_resource()` Methode in `mcp_client_rs`.[9]


3.5. Notification MessageAsynchrone Nachricht vom Server an den Client.5

Notification Struktur:

Tabelle 3.11: Generische Notification Struktur




FeldnameJSON-TypBeschreibungConstraintsjsonrpcstringMuss "2.0" sein.ErforderlichmethodstringName der Benachrichtigungsmethode (z.B. novaDE/resourceUpdated, novaDE/statusChanged).ErforderlichparamsobjectOptionales JSON-Objekt mit den Parametern der Benachrichtigung. Das Schema hängt von method ab.Optional    *Referenzen*: Die Notwendigkeit der Handhabung von Server-Push-Benachrichtigungen wird durch die SSE-Unterstützung in `mcpr` [10, 11, 12] und die Erwähnung in MCP-Konzepten [5] deutlich.


3.6. Response Message (Erfolg)Standard-JSON-RPC-Erfolgsantwort.

Response Struktur:

jsonrpc (string, required): "2.0".
id (string | number | null, required): Muss mit der ID der ursprünglichen Anfrage übereinstimmen.
result (any, required): Das Ergebnis der Anfrage, dessen Struktur vom jeweiligen Request-Typ abhängt (z.B. InitializeResult, ListResourcesResult).





3.7. ErrorResponse Message (Protokollfehler)Standard-JSON-RPC-Fehlerantwort.5

ErrorResponse Struktur:

Tabelle 3.12: Generische ErrorResponse Struktur




FeldnameJSON-TypBeschreibungConstraintsjsonrpcstringMuss "2.0" sein.Erforderlichidstring \number \nullerrorobjectEin Objekt, das den Fehler beschreibt.Erforderlicherror.codeintegerNumerischer Fehlercode.Erforderlicherror.messagestringMenschenlesbare Fehlerbeschreibung.Erforderlicherror.dataanyOptionale, zusätzliche Fehlerdetails.OptionalDie Standard-MCP-Nachrichten bilden ein robustes Fundament. Für NovaDE wird die Hauptaufgabe darin bestehen, spezifische Schemata für `Resource`-Inhalte, `ToolDefinition.parametersSchema`, `ToolDefinition.resultSchema` und `Notification.params` zu definieren, die auf der Domänenschicht des Projekts basieren. Die Verwendung von Rust-SDKs wie `mcp_client_rs` [7, 8, 9] unterstreicht die Bedeutung typsicherer Methoden für Kernanfragen, was wiederum voraussetzt, dass Serverantworten strikt den definierten Schemata entsprechen, um eine erfolgreiche Deserialisierung zu gewährleisten. Dies macht eine rigorose Schemavalidierung sowohl auf Client- als auch auf Serverseite unerlässlich für eine robuste Kommunikation.
4. NovaDE Domänenschicht-IntegrationsstrategieDie erfolgreiche Integration des Model Context Protocol (MCP) in das NovaDE-Projekt hängt entscheidend von einer klaren Strategie zur Abbildung der NovaDE-Domänenschicht auf MCP-Schnittstellen ab. Dieser Abschnitt legt die Methodik und Prinzipien für diese Abbildung fest und diskutiert, wie domänenspezifische Technologien, insbesondere im Kontext eines Desktop-Environments, über MCP zugänglich gemacht werden können. Da die detaillierte "Domänenschicht-Spezifikation" für NovaDE zum Zeitpunkt der Erstellung dieses Dokuments nicht vorliegt, dient dieser Abschnitt als Rahmenwerk und illustriert die Integrationsansätze beispielhaft.

4.1. Methodik zur Abbildung

Prinzipien: Der Prozess der Abbildung der "Domänenschicht-Spezifikation" auf MCP-Konstrukte erfordert eine systematische Analyse. Zunächst müssen die Kernentitäten, -funktionalitäten und -ereignisse der Domänenschicht identifiziert werden. Diese werden dann den entsprechenden MCP-Konzepten – Ressourcen (Resources), Werkzeuge (Tools) und Benachrichtigungen (Notifications) – zugeordnet. Es ist essenziell, dass diese Abbildung die Semantik der Domänenschicht korrekt widerspiegelt und gleichzeitig eine für KI-Agenten verständliche und nutzbare Schnittstelle schafft.
Granularität: Die Wahl der Granularität für MCP-Schnittstellen ist eine wichtige Designentscheidung. Es muss abgewogen werden, ob ein MCP-Server viele feingranulare Werkzeuge und Ressourcen exponiert, die spezifische, kleine Aufgaben abbilden, oder ob weniger, dafür aber grobgranularere Schnittstellen angeboten werden, die komplexere Operationen kapseln. Die optimale Granularität hängt von der Natur der NovaDE-Domänenschicht und den erwarteten Anwendungsfällen der interagierenden KI-Modelle ab. Feingranulare Schnittstellen bieten mehr Flexibilität, können aber zu komplexeren Interaktionsmustern führen, während grobgranulare Schnittstellen die Komplexität für den Client reduzieren, aber möglicherweise weniger flexibel sind.
Abstraktion vs. Direkte Abbildung: MCP ist als universeller Adapter konzipiert 3, was darauf hindeutet, dass es oft als eine Abstraktionsebene über darunterliegenden Systemen dient. Es muss entschieden werden, ob die MCP-Schnittstellen eine direkte Eins-zu-Eins-Abbildung von Funktionen der Domänenschicht darstellen oder ob sie eine höhere Abstraktionsebene bieten, die möglicherweise mehrere Domänenfunktionen zu einem kohärenten MCP-Werkzeug oder einer Ressource zusammenfasst. Eine Abstraktion kann die Komplexität für KI-Agenten reduzieren und die Schnittstelle stabiler gegenüber Änderungen in der Domänenschicht machen. Die Domänenschicht-Spezifikation ist hier der entscheidende Faktor.



4.2. Datenmodellierung für MCP-Schnittstellen

Namenskonventionen: Es müssen klare und konsistente Namenskonventionen für MCP-Ressourcen und -Werkzeuge definiert werden, die sich idealerweise an den Bezeichnern der entsprechenden Entitäten und Funktionen in der NovaDE-Domänenschicht orientieren. Dies fördert die Verständlichkeit und Wartbarkeit.
JSON-Schema-Richtlinien: Für die Inhalte von Ressourcen sowie für die Parameter und Ergebnisse von Werkzeugen müssen JSON-Schemata erstellt werden. Es sind Richtlinien für die Erstellung dieser Schemata festzulegen, um Konsistenz über alle NovaDE-MCP-Schnittstellen hinweg zu gewährleisten. Dies beinhaltet die Verwendung standardisierter Datentypen, Formatierungen und Validierungsregeln.
Datentransformation: Es ist zu analysieren, ob und welche Datentransformationen zwischen den Datenformaten der Domänenschicht und den MCP-Nachrichten-Payloads erforderlich sind. Diese Transformationen müssen klar definiert und implementiert werden, um eine korrekte Datenübertragung sicherzustellen.



4.3. Potenzielle Integrationspunkte mit Desktop-Technologien (Beispielhaft)Dieser Unterabschnitt dient als Illustration, wie domänenspezifische Technologien, die typischerweise in einem Desktop-Environment wie NovaDE vorkommen könnten, über MCP integriert werden könnten. Die konkreten Integrationspunkte hängen vollständig von der tatsächlichen "Domänenschicht-Spezifikation" von NovaDE ab.


D-Bus-Dienste: Viele Desktop-Umgebungen nutzen D-Bus für die Interprozesskommunikation und den Zugriff auf Systemdienste. Wenn die Domänenschicht von NovaDE Interaktionen mit solchen Diensten vorsieht, könnten MCP-Schnittstellen als Abstraktion dienen:

MCP-Werkzeuge (Tools) könnten D-Bus-Methodenaufrufe kapseln. Beispielsweise könnte ein Werkzeug novade/notifications/sendDesktopNotification die Methode Notify des org.freedesktop.Notifications D-Bus-Dienstes aufrufen.13 Ähnlich könnten Werkzeuge für die Interaktion mit org.freedesktop.secrets (z.B. zum Speichern oder Abrufen von Passwörtern 15), org.freedesktop.login1 (z.B. zum Sperren der Sitzung oder Abfragen von Benutzerinformationen 17) oder org.freedesktop.UPower (z.B. zum Abfragen des Batteriestatus 21) definiert werden.
MCP-Ressourcen (Resources) könnten abfragbare D-Bus-Eigenschaften oder den Zustand von D-Bus-Objekten repräsentieren. Beispielsweise könnte eine Ressource novade://power/status die Eigenschaften des org.freedesktop.UPower.Device exponieren.
MCP-Benachrichtigungen (Notifications) könnten D-Bus-Signale an MCP-Clients weiterleiten. Ein SessionLock-Signal von org.freedesktop.login1 könnte eine MCP-Benachrichtigung auslösen.
Zur Implementierung solcher MCP-Server in Rust, die mit D-Bus interagieren, ist die zbus-Bibliothek ein geeignetes Werkzeug.26



Wayland-Protokolle: Wenn NovaDE ein Wayland-Compositor ist oder tiefgreifend mit Wayland-basierten Funktionen der Domänenschicht interagiert, könnten MCP-Schnittstellen diese komplexen Protokolle abstrahieren:

MCP-Werkzeuge (Tools) könnten Aktionen wie Fensterverwaltung (Fokus setzen, Schließen, Größenänderung basierend auf xdg-shell 42), das Erstellen von Screenshots (möglicherweise über xdg-desktop-portal oder direktere Wayland-Protokolle wie wlr-screencopy-v1 falls NovaDE ein wlroots-basierter Compositor ist), oder die Synthese von Eingabeereignissen bereitstellen. Die Integration mit wlr-layer-shell 43 für Oberflächen wie Panels oder Hintergrundbilder könnte ebenfalls über MCP-Werkzeuge gesteuert werden.
MCP-Ressourcen (Resources) könnten den Zustand von Fenstern, Ausgabegeräten (Monitoren) oder Eingabegeräten repräsentieren.
Die Smithay-Bibliothek ist ein Rust-Framework, das Bausteine für Wayland-Compositoren bereitstellt und Handler für viele Wayland-Protokolle enthält.



PipeWire: Wenn die Domänenschicht von NovaDE Multimedia-Aspekte umfasst, könnten MCP-Werkzeuge PipeWire-Knoten (Sinks, Sources, Filter) für Lautstärke, Routing usw. steuern.44 MCP-Ressourcen könnten PipeWire-Objekteigenschaften darstellen. Die pipewire-rs-Bibliothek 47 bietet Rust-Bindings für PipeWire. Beispiele zeigen, wie Knoten aufgelistet 59 und Parameter wie Lautstärke gesetzt werden können.47


XDG Desktop Portals: Wenn NovaDE-Anwendungen sandboxed sind oder benutzervermittelten Zugriff auf Ressourcen (Dateien, Screenshots) benötigen, können MCP-Werkzeuge Aufrufe an XDG Desktop Portals kapseln.87 Die Schnittstellen org.freedesktop.portal.FileChooser 98 und org.freedesktop.portal.Screenshot 91 sind wohldefiniert. Rust-Crates wie xdg-portal 105 oder direkte zbus-Aufrufe können hierfür verwendet werden.


Die "Domänenschicht-Spezifikation" ist der kritischste Input für die Definition konkreter MCP-Schnittstellen. Die obigen Beispiele sind potenzielle Integrationspunkte, falls NovaDE ein Desktop-Environment ist. Die tatsächliche Domäne wird die Spezifika diktieren. Diese Spezifikation muss daher flexibel bleiben. Die Abstraktion komplexer Protokolle über einfachere MCP-Schnittstellen kann die Hürde für KI-Agenten zur Interaktion mit NovaDE signifikant senken, da Wayland 43 und D-Bus 15 komplexe APIs haben, während MCP eine standardisierte und potenziell einfachere Schnittstelle für KI anstrebt.1 Das Design der MCP-Schnittstellen sollte sich daher auf Anwendungsfälle konzentrieren, die für die KI-Interaktion relevant sind, und nicht notwendigerweise jede Nuance der zugrundeliegenden Domänenschicht-APIs exponieren.

5. NovaDE-spezifische MCP-SchnittstellendefinitionenDieser Abschnitt dient als Katalog der MCP-Server-Schnittstellen, die spezifisch für das NovaDE-Projekt entwickelt werden. Jede hier definierte Schnittstellengruppe repräsentiert eine logische Sammlung von Funktionalitäten innerhalb von NovaDE. Der Inhalt dieses Abschnitts ist als Vorlage zu verstehen und muss basierend auf der detaillierten "Domänenschicht-Spezifikation" des NovaDE-Projekts konkretisiert werden. Die Struktur orientiert sich an den Kernkonzepten von MCP (Ressourcen, Werkzeuge, Aufforderungen, Benachrichtigungen) 5, um sicherzustellen, dass alle NovaDE-spezifischen Erweiterungen auf dem Standard-MCP-Framework aufbauen.(Vorlagenstruktur - zu füllen basierend auf der Domänenschicht-Spezifikation)
5.1. Interface-Gruppe: de.nova.projekt.Kernfunktionalitaeten

Übersicht: Diese Schnittstellengruppe umfasst grundlegende Funktionalitäten des NovaDE-Kerns, die für KI-Agenten relevant sind, wie z.B. Systeminformationen oder grundlegende Konfigurationsaspekte.
Tabelle 5.1.1: MCP-Schnittstellen in Gruppe Kernfunktionalitaeten


Schnittstellen-ID (Interface ID)ZweckServer-Komponente (NovaDE-Modul)de.nova.mcp.core.systemInfoBereitstellung von SysteminformationenNovaDE.Core.SystemMonitorde.nova.mcp.core.userPreferencesZugriff auf BenutzereinstellungenNovaDE.Core.SettingsManager*   **5.1.1 Schnittstelle: `de.nova.mcp.core.systemInfo`**
    *   **Version**: `1.0.0`
    *   **Beschreibung**: Stellt Informationen über das NovaDE-System und die zugrundeliegende Hardware/Software-Umgebung bereit.
    *   **Abhängigkeiten**: Abschnitt X.Y der "Domänenschicht-Spezifikation" (Systeminformationen).
    *   **5.1.1.1 Ressourcen (Resources)**
        *   **Name**: `SystemStatus`
        *   **URI-Struktur**: `novade://core/system/status`
        *   **Tabelle 5.1.1.1.A**: Ressourcenschema für `SystemStatus`
FeldnameJSON-TypBeschreibungConstraintsDomänenschicht-EntitätosVersionstringVersion des BetriebssystemsErforderlichSystem.OS.VersionnovaDEVersionstringVersion von NovaDEErforderlichNovaDE.VersioncpuUsagenumberAktuelle CPU-Auslastung (Prozent)OptionalSystem.CPU.CurrentLoadmemoryUsageobjectInformationen zur SpeichernutzungOptionalSystem.Memory.StatsmemoryUsage.totalintegerGesamtspeicher in MBOptionalSystem.Memory.TotalmemoryUsage.availableintegerVerfügbarer Speicher in MBOptionalSystem.Memory.Available        *   **Unterstützte Operationen**: `ReadResource`.
        *   **Zugriffssteuerung**: Nur authentifizierte Systemagenten.

    *   **5.1.1.2 Werkzeuge (Tools)**: Keine für diese spezifische Schnittstelle definiert.
    *   **5.1.1.3 Aufforderungen (Prompts)**: Keine für diese spezifische Schnittstelle definiert.
    *   **5.1.1.4 Benachrichtigungen (Notifications)**
        *   **Name**: `systemLoadWarning`
        *   **Auslösebedingungen**: Wird gesendet, wenn die CPU-Auslastung für einen bestimmten Zeitraum einen Schwellenwert überschreitet.
        *   **Tabelle 5.1.1.4.A**: Payload-Schema für `systemLoadWarning`
FeldnameJSON-TypBeschreibungDomänenschicht-EreignisdatenlevelstringWarnstufe (HIGH, CRITICAL)SystemAlert.LevelcpuUsagenumberAktuelle CPU-Auslastung zum Zeitpunkt des AlarmsSystemAlert.CPULoad
5.2. Interface-Gruppe: de.nova.projekt.DesktopIntegration (Beispiel für D-Bus/Wayland)

Übersicht: Diese Schnittstellengruppe demonstriert, wie Desktop-spezifische Funktionalitäten, die typischerweise über D-Bus oder Wayland-Protokolle bereitgestellt werden, über MCP abstrahiert werden können.
Tabelle 5.2.1: MCP-Schnittstellen in Gruppe DesktopIntegration


Schnittstellen-ID (Interface ID)ZweckServer-Komponente (NovaDE-Modul)de.nova.mcp.desktop.notificationsSenden und Verwalten von Desktop-BenachrichtigungenNovaDE.NotificationServiceWrapperde.nova.mcp.desktop.secretsSicherer Speicher für GeheimnisseNovaDE.SecretsAgentde.nova.mcp.desktop.powerAbfragen und Steuern von EnergieoptionenNovaDE.PowerManagerWrapperde.nova.mcp.desktop.sessionVerwalten von BenutzersitzungenNovaDE.SessionManagerWrapperde.nova.mcp.desktop.fileChooserÖffnen von DateiauswahldialogenNovaDE.FileChooserPortalWrapperde.nova.mcp.desktop.screenshotErstellen von BildschirmfotosNovaDE.ScreenshotPortalWrapper*   **5.2.1 Schnittstelle: `de.nova.mcp.desktop.notifications`**
    *   **Version**: `1.0.0`
    *   **Beschreibung**: Ermöglicht das Senden von Desktop-Benachrichtigungen und das Abfragen von Server-Fähigkeiten, basierend auf `org.freedesktop.Notifications`.
    *   **Abhängigkeiten**: `org.freedesktop.Notifications` D-Bus Spezifikation.[13, 14]
    *   **5.2.1.1 Ressourcen (Resources)**: Keine direkt, Status wird über Werkzeuge/Benachrichtigungen gehandhabt.
    *   **5.2.1.2 Werkzeuge (Tools)**
        *   **Name**: `sendNotification`
        *   **Beschreibung**: Sendet eine Desktop-Benachrichtigung.
        *   **Tabelle 5.2.1.2.A**: Eingabeparameter für `sendNotification` (abgeleitet von `org.freedesktop.Notifications.Notify` [14])
ParameternameJSON-TypBeschreibungErforderlichDomänenschicht-Parameter (D-Bus)appNamestringName der Anwendung, die die Benachrichtigung sendet.Neinapp_name (STRING)replacesIdintegerID einer zu ersetzenden Benachrichtigung (0 für neu).Neinreplaces_id (UINT32)appIconstringPfad oder Name des Anwendungsicons.Neinapp_icon (STRING)summarystringZusammenfassung der Benachrichtigung.Jasummary (STRING)bodystringDetaillierter Text der Benachrichtigung.Neinbody (STRING)actionsarray of stringListe von Aktions-IDs und deren Beschriftungen (alternierend).Neinactions (as)hintsobjectZusätzliche Hinweise für den Server (z.B. Dringlichkeit).Neinhints (a{sv})expireTimeoutintegerTimeout in Millisekunden (-1 für Server-Default).Neinexpire_timeout (INT32)        *   **Tabelle 5.2.1.2.B**: Ausgabeparameter für `sendNotification`
ParameternameJSON-TypBeschreibungDomänenschicht-Rückgabe (D-Bus)notificationIdintegerEindeutige ID der Benachrichtigung.id (UINT32)        *   **Name**: `getNotificationCapabilities`
        *   **Beschreibung**: Frägt die Fähigkeiten des Benachrichtigungsservers ab.
        *   **Tabelle 5.2.1.2.C**: Ausgabeparameter für `getNotificationCapabilities` (abgeleitet von `org.freedesktop.Notifications.GetCapabilities` [14])
ParameternameJSON-TypBeschreibungDomänenschicht-Rückgabe (D-Bus)capabilitiesarray of stringListe der unterstützten Server-Fähigkeiten.capabilities (as)    *   **5.2.1.3 Benachrichtigungen (Notifications)**
        *   **Name**: `notificationClosed` (entspricht `org.freedesktop.Notifications.NotificationClosed` [14])
        *   **Tabelle 5.2.1.3.A**: Payload-Schema für `notificationClosed`
FeldnameJSON-TypBeschreibungDomänenschicht-Ereignisdaten (D-Bus)idintegerID der geschlossenen Benachrichtigung.id (UINT32)reasonintegerGrund für das Schließen (1=expired, 2=dismissed, 3=closed by call).reason (UINT32)*   *(Weitere Schnittstellen wie `de.nova.mcp.desktop.secrets`, `de.nova.mcp.desktop.power` etc. würden analog unter Verwendung der relevanten D-Bus Spezifikationen [15, 17, 21] und XDG Portal Spezifikationen [98, 100] detailliert werden.)*
Die explizite Abbildung auf Entitäten, Funktionen und Ereignisse der "Domänenschicht" in den Tabellen ist entscheidend, um die Nachvollziehbarkeit zu gewährleisten und zu verdeutlichen, wie die MCP-Schnittstellen mit dem zugrundeliegenden NovaDE-System zusammenhängen. Dies ist eine direkte Anforderung der Nutzeranfrage. Die Konsistenz zwischen dieser MCP-Spezifikation und der "Domänenschicht-Spezifikation" muss während der gesamten Entwicklung von NovaDE aufrechterhalten werden. Dieser Abschnitt wird der umfangreichste und detaillierteste sein und erfordert eine sorgfältige Definition von Schemata und Verhaltensweisen für jedes domänenspezifische MCP-Element, sobald die Domänenschicht-Spezifikation verfügbar ist.6. Implementierungsaspekte für NovaDEDieser Abschnitt behandelt empfohlene Technologien und Muster für die Implementierung von MCP-Clients und -Servern im NovaDE-Projekt, mit besonderem Fokus auf die Handhabung von Asynchronität und Verbindungsmanagement.

6.1. Empfohlene SDKs und BibliothekenDie Wahl der SDKs und Bibliotheken hängt von der jeweiligen Komponente und deren Anforderungen ab, insbesondere bezüglich des Transportmechanismus.

Rust:

Server-Implementierung: Für MCP-Server, die Server-Sent Events (SSE) für Benachrichtigungen nutzen müssen, wird das mcpr Crate empfohlen.10 Es bietet High-Level-Abstraktionen für Server, Werkzeuge und unterstützt verschiedene Transportmechanismen, einschließlich Stdio und SSE. Die Fähigkeit, Server-Push-Benachrichtigungen zu senden, ist für viele KI-Anwendungen kritisch, was mcpr favorisiert.
Client-Implementierung:

Das mcpr Crate 10 ist ebenfalls eine gute Wahl für Rust-basierte MCP-Clients, insbesondere wenn SSE-basierte Benachrichtigungen empfangen werden müssen. Es bietet eine konsistente API für Client- und Server-Entwicklung.
Das mcp_client_rs Crate von Darin Kishore 7 (basierend auf einer früheren Version von Derek-X-Wang/mcp-rust-sdk 109) ist eine weitere Option, primär für Stdio-basierte Kommunikation. Die Dokumentation ist jedoch weniger explizit bezüglich der Handhabung von asynchronen Server-Push-Benachrichtigungen über Stdio 9, was für reaktive Agenten ein Nachteil sein könnte. Die Unterstützung für WebSocket-Transport mit Wiederverbindungshandhabung ist zwar erwähnt, aber als "Coming Soon" markiert.109




Go: Für Komponenten des NovaDE-Projekts, die in Go implementiert werden, stellt mcp-go 6 eine valide Option dar. Dieses SDK unterstützt ebenfalls Stdio und bietet Abstraktionen für Server, Werkzeuge und Ressourcen.
Andere Sprachen: Da MCP auf JSON-RPC 2.0 basiert, können Clients und Server prinzipiell in jeder Sprache implementiert werden, die JSON-Verarbeitung und den gewählten Transportmechanismus (Stdio oder HTTP/SSE) unterstützt.



6.2. Handhabung von asynchronen Server-Sent NotificationsAsynchrone Benachrichtigungen vom Server zum Client sind ein Kernmerkmal von MCP, um KI-Agenten über Zustandsänderungen oder Ereignisse in der Domänenschicht zu informieren.3

Client-seitig:

Clients, die auf Server-Push-Benachrichtigungen reagieren müssen, sollten den SSE-Transportmechanismus verwenden. Das mcpr Crate in Rust bietet hierfür geeignete Abstraktionen, um einen SSE-Stream zu abonnieren und die eingehenden Nachrichten zu verarbeiten.10 Dies beinhaltet das Parsen der JSON-RPC-Benachrichtigungen und das Weiterleiten der params-Nutzlast an die zuständige Anwendungslogik.
Beispiele für MCP-Server, die Benachrichtigungen verwenden, wie der MCP Notify Server 122 oder die in mcp-go beschriebene Fähigkeit, Benachrichtigungen an spezifische Clients zu senden 6, unterstreichen die Wichtigkeit dieses Musters.
Für Stdio-Transporte ist die Handhabung von Server-Push-Benachrichtigungen komplexer, da Stdio primär für Request-Response-Interaktionen ausgelegt ist. mcp_client_rs müsste hierfür einen dedizierten Lesethread oder eine asynchrone Lese-Schleife implementieren, die kontinuierlich stdout des Servers auf neue Nachrichten überwacht und diese dann als Benachrichtigungen interpretiert.118 Die Dokumentation von mcp_client_rs ist hierzu nicht explizit.


Server-seitig:

NovaDE MCP-Server, die Benachrichtigungen senden müssen, sollten bei Verwendung von SSE die etablierten Mechanismen des gewählten Frameworks (z.B. mcpr in Rust oder FastAPI mit SSE-Support in Python 12) nutzen, um Nachrichten an alle oder ausgewählte verbundene Clients zu pushen.
Bei Stdio-Transport müssen Benachrichtigungen als reguläre JSON-RPC-Nachrichten auf stdout geschrieben werden, wobei der Client für das korrekte Parsen und Unterscheiden von regulären Antworten zuständig ist.





6.3. Behandlung von Verbindungsstatus-EreignissenEine robuste Behandlung von Verbindungsstatus ist essentiell für die Zuverlässigkeit.

Client-seitig:

Clients müssen Mechanismen zur Erkennung von Verbindungsabbrüchen implementieren. Dies kann durch Timeouts bei Requests, Fehler beim Lesen/Schreiben auf den Transportkanal oder spezifische Fehlermeldungen des Transport-SDKs geschehen.
Strategien für automatische Wiederverbindungsversuche sollten implementiert werden, idealerweise mit exponentiellem Backoff, um Server nicht zu überlasten.
Der mcpr-Client erwähnt die Handhabung von Prozessbeendigung und Pipe-Verbindungsproblemen bei Stdio.10 Die (geplante) WebSocket-Unterstützung in mcp_client_rs erwähnt "built-in reconnection handling".109
Allgemeine Prinzipien zur Fehlerbehebung bei Netzwerkverbindungen, wie in 123 für Azure Event Grid beschrieben (Port-Blockaden, Firewall-Regeln), können auch hier relevant sein, insbesondere bei HTTP/SSE.


Server-seitig:

MCP-Server sollten Client-Verbindungen aktiv verwalten, einschließlich Logging von Verbindungsaufbau und -abbau.
Bei Stdio-basierten Servern endet der Serverprozess typischerweise, wenn der Client die Verbindung trennt.10 Für langlebige Sitzungen muss dies bedacht werden.





6.4. Zustandsbehaftetes Sitzungsmanagement (Session Management)Einige Interaktionen mit KI-Modellen erfordern möglicherweise einen Zustand, der über mehrere MCP-Requests hinweg erhalten bleibt.

Server-seitig: Wenn NovaDE-Schnittstellen zustandsbehaftete Interaktionen erfordern, müssen MCP-Server Mechanismen zum Sitzungsmanagement implementieren. Das mcp-go SDK erwähnt explizit die Unterstützung für die Verwaltung separater Zustände für jeden verbundenen Client, das Verfolgen von Client-Sitzungen und die Möglichkeit, per-session Werkzeuganpassungen vorzunehmen.6
Dies könnte die Generierung und Verwaltung von Sitzungs-IDs beinhalten, die vom Client bei nachfolgenden Anfragen mitgesendet werden, oder die Nutzung inhärenter Sitzungsmerkmale des gewählten Transports (z.B. langlebige SSE-Verbindungen).
Die Notwendigkeit und Komplexität des Sitzungsmanagements hängt stark von den spezifischen Anwendungsfällen ab, die durch die "Domänenschicht-Spezifikation" definiert werden.


Die Wahl des SDKs und die Implementierung von Benachrichtigungs- und Verbindungsmanagement sind kritisch. Für NovaDE-Komponenten, die auf Server-Push-Benachrichtigungen angewiesen sind oder eine robustere Handhabung von Remote-Verbindungen benötigen, scheint mcpr aufgrund seiner expliziten SSE-Unterstützung die passendere Wahl in Rust zu sein. Die Client-Implementierungen in NovaDE müssen eine widerstandsfähige Logik für die Verarbeitung von Benachrichtigungsströmen und die Behandlung von Verbindungsfehlern enthalten, um die Stabilität und Reaktionsfähigkeit der KI-Agenten zu gewährleisten.7. Sicherheitsmodell für NovaDE MCP-SchnittstellenDie Sicherheit der MCP-Schnittstellen ist von größter Bedeutung, da sie potenziell Zugriff auf sensible Daten und kritische Funktionalitäten des NovaDE-Projekts ermöglichen. Das Sicherheitsmodell muss Authentifizierung, Autorisierung, Datensicherheit und Benutzereinwilligung umfassen. MCP selbst legt Wert auf Sicherheit 2, aber die konkrete Ausgestaltung obliegt dem NovaDE-Projekt.

7.1. Authentifizierung und Autorisierung

Client-Authentifizierung: Es müssen Mechanismen definiert werden, wie sich MCP-Clients gegenüber NovaDE-MCP-Servern authentifizieren.

Für Stdio-basierte Kommunikation ist die Authentifizierung oft implizit durch die Prozessgrenzen und Benutzerkontexte des Betriebssystems gegeben. Zusätzliche anwendungsspezifische Token können jedoch für eine feinere Kontrolle verwendet werden.
Für HTTP/SSE-basierte Kommunikation sind explizite Authentifizierungsmechanismen erforderlich. Optionen umfassen:

Token-basierte Authentifizierung (z.B. API-Keys, JWTs), die im HTTP-Header übertragen werden.
OAuth 2.0, falls externe Clients oder Benutzer im Namen von Benutzern agieren. MCP unterstützt prinzipiell OAuth.3
Es ist zu beachten, dass die MCP-Spezifikation zum Zeitpunkt einiger Referenzdokumente möglicherweise keinen standardisierten Authentifizierungsmechanismus für SSE-Server definierte.12 Daher muss NovaDE hier ggf. eine eigene Lösung implementieren oder auf Netzwerkebene absichern (z.B. über VPN, IP-Whitelisting oder einen Reverse-Proxy, der die Authentifizierung übernimmt).




Server-Authentifizierung: Clients müssen die Identität der NovaDE-MCP-Server überprüfen können, insbesondere bei HTTP/SSE-Kommunikation. Dies geschieht typischerweise durch TLS-Zertifikate, deren Validierung clientseitig erfolgen muss.
Autorisierungsrichtlinien: Nach erfolgreicher Authentifizierung muss die Autorisierung erfolgen. Es muss klar definiert werden, welche authentifizierten Clients (oder Benutzer, in deren Namen sie handeln) auf welche MCP-Server, Ressourcen und Werkzeuge zugreifen dürfen.

Dies erfordert eine Integration mit einem bestehenden oder neu zu definierenden Identitäts- und Zugriffsmanagementsystem (IAM) für NovaDE.
Das MCP Host-Konzept ist hier zentral: Der Host-Prozess ist für die Verwaltung von Sicherheitsrichtlinien und Benutzerautorisierung zuständig.2 Dies impliziert, dass der NovaDE MCP Host eine kritische Rolle bei der Durchsetzung von Zugriffsrechten spielt.
Wenn MCP-Server privilegierte Operationen im System ausführen (z.B. bei Integration mit Desktop-Technologien), könnte PolicyKit 108 für die Autorisierungsprüfungen auf Systemebene herangezogen werden. Der MCP-Server würde dann als Mechanismus im Sinne von PolicyKit agieren.





7.2. Berechtigungsmodell für Ressourcen und Werkzeuge

Es ist ein granulares Berechtigungsmodell zu definieren, das spezifische Aktionen auf MCP-Ressourcen (z.B. read, write, list) und die Ausführung von MCP-Werkzeugen (execute) abdeckt.
Diese Berechtigungen sollten an Rollen oder individuelle Client-Identitäten gebunden sein und vom MCP-Server bzw. dem MCP-Host bei jeder Anfrage überprüft werden.
Die Definition dieser Berechtigungen muss eng mit der "Domänenschicht-Spezifikation" und den dort definierten Zugriffsregeln verknüpft sein.



7.3. Datensicherheit

Verschlüsselung bei der Übertragung (Encryption in Transit): Für HTTP/SSE-basierte MCP-Kommunikation ist die Verwendung von TLS (HTTPS/WSS) zwingend erforderlich, um die Vertraulichkeit und Integrität der übertragenen Daten zu gewährleisten.
Verschlüsselung im Ruhezustand (Encryption at Rest): Falls NovaDE-MCP-Server Daten persistent speichern (z.B. Konfigurationen, zwischengespeicherte Ressourcendaten), müssen diese Daten im Ruhezustand verschlüsselt werden, um unbefugten Zugriff zu verhindern. Die Wahl der Verschlüsselungsmethoden sollte aktuellen Sicherheitsstandards entsprechen.
Geheimnisverwaltung (Secret Management): MCP-Server benötigen möglicherweise Geheimnisse (API-Schlüssel, Datenbank-Passwörter, Zugriffstoken für die Domänenschicht). Diese Geheimnisse müssen sicher gespeichert und verwaltet werden.

Die Nutzung der Freedesktop Secrets API über D-Bus (Schnittstelle org.freedesktop.Secrets 15) ist eine Option für NovaDE-MCP-Server, um solche Geheimnisse sicher im Benutzerkontext oder Systemkontext zu speichern und abzurufen. Dies ist besonders relevant, wenn Server im Auftrag des Benutzers auf geschützte Domänenressourcen zugreifen.





7.4. Benutzereinwilligung (User Consent)

Für Operationen, die auf sensible Benutzerdaten zugreifen oder signifikante Aktionen im Namen des Benutzers ausführen (z.B. das Ändern von Systemeinstellungen, Senden von Nachrichten), müssen Mechanismen zur Einholung der expliziten Zustimmung des Benutzers implementiert werden.
Der MCP Host-Prozess spielt auch hier eine Rolle bei der Durchsetzung von Einwilligungsanforderungen.2
Die Gestaltung der Einwilligungsdialoge muss transparent und verständlich sein, damit der Benutzer eine informierte Entscheidung treffen kann. XDG Desktop Portals 87 bieten Standardmechanismen für benutzervermittelte Zugriffsanfragen, die als Inspiration dienen oder direkt genutzt werden könnten, falls MCP-Werkzeuge solche Portale kapseln.


Die Sicherheitsarchitektur von NovaDE muss einen oder mehrere MCP Hosts definieren, die als Gatekeeper fungieren und die oben genannten Sicherheitsfunktionen koordinieren und durchsetzen. Ohne klar definierte Hosts könnten Sicherheitsrichtlinien inkonsistent angewendet werden.8. FehlerbehandlungsspezifikationEine konsistente und informative Fehlerbehandlung ist entscheidend für die Robustheit, Wartbarkeit und Benutzerfreundlichkeit der MCP-Schnittstellen im NovaDE-Projekt. Diese Spezifikation definiert Standardfehlercodes und Richtlinien für die Fehlerbehandlung.

8.1. Standard-MCP-Fehlercodes für NovaDEZusätzlich zu den Standard-JSON-RPC-2.0-Fehlercodes (Parse Error: -32700, Invalid Request: -32600, Method not found: -32601, Invalid params: -32602, Internal error: -32603) definiert NovaDE einen Satz erweiterter Fehlercodes, um spezifischere Fehlersituationen innerhalb des MCP-Kontexts zu signalisieren. Diese Codes sollten von allen NovaDE-MCP-Servern konsistent verwendet werden. Die Struktur der Fehlerantwort folgt dem Standard-JSON-RPC-Error-Objekt.5

Tabelle 8.1: NovaDE MCP Fehlercodes


CodeName (Konstante)Nachricht (Template)HTTP-Status (für SSE)Beschreibung-32000DOMAIN_SPECIFIC_ERROR"Domänenspezifischer Fehler: {details}"500Ein Fehler ist in der NovaDE-Domänenschicht aufgetreten. {details} kann spezifische Informationen enthalten.-32001RESOURCE_NOT_FOUND"Ressource '{uri}' nicht gefunden."404Die angeforderte MCP-Ressource existiert nicht oder ist nicht zugänglich.-32002TOOL_EXECUTION_FAILED"Ausführung des Werkzeugs '{toolName}' fehlgeschlagen."500Ein unerwarteter Fehler während der Ausführung eines MCP-Werkzeugs.-32003INVALID_TOOL_PARAMETERS"Ungültige Parameter für Werkzeug '{toolName}'."400Die für ein MCP-Werkzeug bereitgestellten Parameter sind ungültig oder unvollständig.-32004PERMISSION_DENIED"Zugriff für Operation '{operation}' auf '{target}' verweigert."403Dem aufrufenden Client fehlen die notwendigen Berechtigungen für die angeforderte Operation.-32005SERVER_UNAVAILABLE"MCP-Server ist temporär nicht verfügbar."503Der angefragte MCP-Server ist derzeit nicht erreichbar oder überlastet.-32006AUTHENTICATION_FAILED"Authentifizierung fehlgeschlagen."401Die Authentifizierung des Clients ist fehlgeschlagen.-32007PROTOCOL_VERSION_MISMATCH"Inkompatible Protokollversion. Client: {clientVersion}, Server unterstützt: {serverVersions}"400Client und Server konnten sich nicht auf eine gemeinsame MCP-Protokollversion einigen.Die Verwendung von Rust-Bibliotheken wie `thiserror` [125, 126, 127] oder `snafu` [128] wird für die Implementierung strukturierter Fehler in den Rust-basierten MCP-Servern von NovaDE empfohlen. Diese Bibliotheken erleichtern die Definition von Fehler-Enums, die automatische Implementierung von `std::error::Error` und `Display`, sowie das Anhängen von Kontextinformationen.


8.2. Fehlerweiterleitung (Error Propagation)

MCP-Server müssen Fehler, die in der darunterliegenden Domänenschicht oder von abhängigen Diensten (z.B. D-Bus-Dienste, externe APIs) auftreten, abfangen und in standardisierte MCP-Fehlerantworten umwandeln. Dabei ist es wichtig, eine Balance zu finden: Einerseits soll genügend Kontext für die Fehlerdiagnose bereitgestellt werden, andererseits dürfen keine sensiblen internen Implementierungsdetails oder Sicherheitsinformationen an den Client durchsickern.
Die source-Kette von Fehlern, wie sie von std::error::Error und Crates wie thiserror unterstützt wird, kann intern zur Diagnose verwendet werden, aber die an den MCP-Client gesendete Fehlernachricht sollte sorgfältig formuliert sein. Die Diskussion in 128 über das Gruppieren mehrerer Fehlertypen und das Hinzufügen von Kontext ist hier relevant.



8.3. Client-seitige Fehlerbehandlung

MCP-Clients im NovaDE-Projekt müssen robust auf Fehlerantworten reagieren. Dies beinhaltet das Parsen des error-Objekts, die Interpretation des code und der message, und gegebenenfalls die Nutzung der data-Komponente.
Abhängig vom Fehlercode und der Natur des Fehlers können verschiedene Strategien angewendet werden:

Wiederholungsversuche (Retries): Bei temporären Fehlern (z.B. SERVER_UNAVAILABLE oder bestimmten Netzwerkfehlern) können Clients Wiederholungsversuche mit exponentiellem Backoff implementieren.
Benutzerbenachrichtigung: Bei Fehlern, die eine Benutzerinteraktion erfordern oder den Benutzer über ein Problem informieren müssen (z.B. PERMISSION_DENIED, AUTHENTICATION_FAILED), sollte eine klare und verständliche Meldung angezeigt werden.
Graceful Degradation: Wenn eine Funktionalität aufgrund eines Fehlers nicht verfügbar ist, sollte der Client versuchen, in einem eingeschränkten Modus weiterzuarbeiten oder alternative Pfade anzubieten.


Die mcp_client_rs 7 und mcpr 10 SDKs stellen Result-Typen für ihre Operationen bereit, die eine Fehlerbehandlung über das Err-Variant ermöglichen.

Die Unterscheidung zwischen Protokollfehlern (die eine JSON-RPC ErrorResponse auslösen) und anwendungsspezifischen Werkzeug-Fehlern ist wichtig. Wie in 116 (impliziert durch isError in CallToolResult bei einigen SDK-Interpretationen) angedeutet, kann ein Werkzeugaufruf protokollkonform erfolgreich sein, die interne Logik des Werkzeugs jedoch fehlschlagen. In solchen Fällen sollte die CallToolResult isError: true und ein anwendungsspezifisches error-Objekt im result-Feld enthalten, anstatt einen JSON-RPC-Protokollfehler auszulösen. Dies ermöglicht eine differenziertere Fehlerbehandlung auf Client-Seite. Diese Spezifikation muss klar definieren, wann welche Art von Fehler gemeldet wird.

9. Versionierung und ProtokollevolutionUm die langfristige Wartbarkeit und Kompatibilität der MCP-Schnittstellen im NovaDE-Projekt sicherzustellen, ist eine klare Strategie für Versionierung und Protokollevolution unerlässlich.

9.1. MCP-Versionsstrategie für NovaDE

Globale MCP-Version: Das NovaDE-Projekt wird sich an der offiziellen Versionierung des Model Context Protocol orientieren, wie sie von den Standardisierungsgremien (z.B. Anthropic und die Community) vorgegeben wird. Aktuell wird auf eine Version wie "2025-03-26" referenziert.5 Die Initialize-Nachricht dient dem Aushandeln dieser Basis-Protokollversion zwischen Client und Server.3
NovaDE-spezifische Schnittstellenversionierung: Jede in Abschnitt 5 definierte, NovaDE-spezifische MCP-Schnittstelle (z.B. de.nova.mcp.core.systemInfo) erhält eine eigene semantische Versionierung (z.B. 1.0.0). Diese Version wird im serverVersion-Feld der InitializeResult-Nachricht für den jeweiligen Server und idealerweise als Teil der Metadaten einer Ressource oder eines Werkzeugs kommuniziert.
Granularität der Versionierung: Einzelne Ressourcen oder Werkzeuge innerhalb einer Schnittstelle können bei Bedarf ebenfalls versioniert werden, falls sich ihre Schemata oder Verhalten unabhängig von der Gesamtschnittstelle ändern. Dies sollte jedoch zugunsten der Einfachheit vermieden werden, wenn möglich.



9.2. Umgang mit abwärtskompatiblen ÄnderungenAbwärtskompatible Änderungen sind solche, die bestehende Clients nicht brechen.

Beispiele:

Hinzufügen neuer, optionaler Felder zu Anfrage- oder Antwort-Payloads.
Hinzufügen neuer, optionaler Parameter zu Werkzeugen.
Hinzufügen neuer Werkzeuge oder Ressourcen zu einer bestehenden Schnittstelle.
Hinzufügen neuer Werte zu Enums (Clients sollten unbekannte Enum-Werte tolerant behandeln).


Vorgehen: Solche Änderungen führen zu einer Erhöhung der Minor- oder Patch-Version der betroffenen NovaDE-spezifischen Schnittstelle (z.B. von 1.0.0 auf 1.1.0 oder 1.0.1). Clients, die für eine ältere Minor-Version entwickelt wurden, sollten weiterhin mit Servern funktionieren, die eine neuere Minor-Version derselben Major-Version implementieren.



9.3. Umgang mit abwärtsinkompatiblen ÄnderungenAbwärtsinkompatible Änderungen sind solche, die bestehende Clients potenziell brechen können.

Beispiele:

Entfernen von Feldern aus Anfrage- oder Antwort-Payloads.
Umbenennen von Feldern oder Ändern ihres Datentyps.
Ändern erforderlicher Parameter für Werkzeuge.
Entfernen von Werkzeugen oder Ressourcen.
Grundlegende Änderung der Semantik einer Operation.


Vorgehen:

Solche Änderungen erfordern eine Erhöhung der Major-Version der betroffenen NovaDE-spezifischen Schnittstelle (z.B. von 1.1.0 auf 2.0.0).
Es wird dringend empfohlen, abwärtsinkompatible Änderungen so weit wie möglich zu vermeiden.
Wenn eine solche Änderung unumgänglich ist, sollte idealerweise für eine Übergangszeit sowohl die alte als auch die neue Version der Schnittstelle parallel angeboten werden (z.B. unter einem anderen Endpunkt oder mit einer expliziten Versionsauswahl im Initialize-Request).
Eine klare Kommunikation und Migrationspfade für Clients müssen bereitgestellt werden.



Die Initialize-Nachricht spielt eine Schlüsselrolle bei der Versionierung, da sie es Clients und Servern ermöglicht, ihre unterstützten Protokollversionen und optional auch spezifische Feature-Flags auszutauschen.3 NovaDE-Clients sollten darauf vorbereitet sein, dass Server möglicherweise nicht alle angefragten Features oder die exakt gleiche Schnittstellenversion unterstützen, und entsprechend reagieren (z.B. durch Deaktivieren bestimmter Funktionalitäten oder Melden einer Inkompatibilität).

10. SchlussfolgerungenDie Implementierung des Model Context Protocol (MCP) im NovaDE-Projekt stellt einen strategisch wichtigen Schritt dar, um die Integration von KI-Funktionalitäten auf einer standardisierten, flexiblen und zukunftssicheren Basis zu ermöglichen. Diese Ultra-Feinspezifikation legt den detaillierten Rahmen für die MCP-Schnittstellen, Nachrichtenformate, Integrationsstrategien mit der Domänenschicht sowie für Implementierungs-, Sicherheits- und Fehlerbehandlungsaspekte fest.Wesentliche Erkenntnisse und Implikationen sind:
Standardisierung als Fundament: MCP bietet eine universelle Sprache für die Kommunikation zwischen KI-Modellen und den vielfältigen Datenquellen und Werkzeugen des NovaDE-Projekts.1 Dies reduziert den Aufwand für proprietäre Integrationen und fördert die Interoperabilität.
Abhängigkeit von der Domänenschicht: Die konkrete Ausgestaltung der NovaDE-spezifischen MCP-Ressourcen, -Werkzeuge und -Benachrichtigungen ist untrennbar mit der noch zu detaillierenden "Domänenschicht-Spezifikation" verbunden. Diese Spezifikation muss als Grundlage für die in Abschnitt 5 vorgesehenen Definitionen dienen.
Architektonische Entscheidungen: Die Wahl der Transportmechanismen (Stdio vs. HTTP/SSE) und die klare Definition von MCP Host-, Server- und Client-Rollen innerhalb der NovaDE-Architektur sind entscheidend für Leistung, Skalierbarkeit und Sicherheit.2 Für reaktive Agenten und Server-Push-Benachrichtigungen ist SSE der empfohlene Weg.
Rust SDKs: Für die Implementierung in Rust bieten sich mcpr 10 und mcp_client_rs 7 an, wobei mcpr aufgrund seiner expliziten SSE-Unterstützung und moderneren Anmutung für komplexere Szenarien mit Benachrichtigungen tendenziell vorzuziehen ist.
Sicherheit als Priorität: Ein robustes Sicherheitsmodell, das Authentifizierung, Autorisierung, Datensicherheit und Benutzereinwilligung umfasst, ist unerlässlich. Die Integration mit bestehenden Systemmechanismen (z.B. PolicyKit, Freedesktop Secrets API) sollte geprüft werden, falls die Domänenschicht dies erfordert.2
Konsistente Fehlerbehandlung und Versionierung: Standardisierte Fehlercodes und eine klare Versionierungsstrategie sind für die Wartbarkeit und Weiterentwicklung des Systems unabdingbar.
Empfehlungen für das weitere Vorgehen:
Priorisierung der Domänenschicht-Spezifikation: Die Fertigstellung und Detaillierung der "Domänenschicht-Spezifikation" ist der nächste kritische Schritt, um die in diesem Dokument vorbereiteten MCP-Schnittstellendefinitionen (Abschnitt 5) mit Leben zu füllen.
Prototypische Implementierung: Es wird empfohlen, frühzeitig mit der prototypischen Implementierung ausgewählter MCP-Server und -Clients zu beginnen, basierend auf den hier spezifizierten Standards und unter Verwendung der evaluierten SDKs. Dies hilft, die Konzepte zu validieren und praktische Erfahrungen zu sammeln.
Iterative Verfeinerung: Diese Spezifikation sollte als lebendes Dokument betrachtet und parallel zur Entwicklung der Domänenschicht und der MCP-Komponenten iterativ verfeinert werden.
Fokus auf Sicherheit: Sicherheitsaspekte müssen von Beginn an in Design und Implementierung aller MCP-Komponenten berücksichtigt werden.
Entwickler-Schulung: Sicherstellen, dass alle beteiligten Entwickler ein tiefes Verständnis von MCP und dieser Spezifikation erlangen.
Durch die konsequente Anwendung dieser Spezifikation kann das NovaDE-Projekt eine leistungsfähige und flexible MCP-Infrastruktur aufbauen, die es ermöglicht, das volle Potenzial moderner KI-Modelle auszuschöpfen.11. Anhang

11.1. Glossar

MCP (Model Context Protocol): Ein offener Standard zur Verbindung von KI-Modellen mit externen Datenquellen und Werkzeugen.
JSON-RPC 2.0: Ein leichtgewichtiges Remote Procedure Call Protokoll, das als Basis für MCP dient.
Ressource (Resource): Eine Dateneinheit, die von einem MCP-Server bereitgestellt und von einem Client gelesen werden kann.
Werkzeug (Tool): Eine Funktion oder Operation, die von einem MCP-Server bereitgestellt und von einem Client aufgerufen werden kann, um Aktionen auszuführen.
Aufforderung (Prompt): Eine vordefinierte Vorlage für Interaktionen mit einem LLM, die Systemanweisungen und Argumente umfassen kann.
Benachrichtigung (Notification): Eine asynchrone Nachricht vom Server an den Client, die über Ereignisse oder Zustandsänderungen informiert.
Stdio (Standard Input/Output): Ein Transportmechanismus für MCP, der auf Standard-Datenströmen basiert, typischerweise für lokale Prozesskommunikation.
SSE (Server-Sent Events): Ein Transportmechanismus für MCP über HTTP, der es einem Server ermöglicht, kontinuierlich Daten an einen Client zu senden.
Domänenschicht: Die spezifische Anwendungslogik und Datenmodelle des NovaDE-Projekts.
NovaDE: Name des Projekts, für das diese MCP-Spezifikation erstellt wird.
Client (MCP): Eine Softwarekomponente (oft Teil eines KI-Agenten oder einer Anwendung), die mit einem MCP-Server interagiert, um Kontext zu erhalten oder Aktionen auszuführen.
Server (MCP): Eine Softwarekomponente, die Daten oder Funktionalitäten über das MCP-Protokoll bereitstellt.
Host (MCP): Eine Anwendung oder Umgebung, die MCP-Clients beherbergt und deren Interaktionen mit MCP-Servern koordiniert und absichert.
URI (Uniform Resource Identifier): Eine Zeichenfolge zur eindeutigen Identifizierung einer Ressource.



11.2. JSON Schema Beispiele (Referenz)(Dieser Abschnitt würde exemplarische JSON-Schemata für typische Ressourcen oder Werkzeugparameter enthalten, um die in den Tabellen beschriebenen Strukturen zu illustrieren. Aufgrund der fehlenden Domänenschicht-Spezifikation sind dies allgemeine Beispiele.)


Beispiel: Ressourcenschema für ein einfaches Dateiobjekt
JSON{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "NovaDEFileResource",
  "description": "Repräsentiert eine Datei im NovaDE-System.",
  "type": "object",
  "properties": {
    "uri": {
      "type": "string",
      "format": "uri",
      "description": "Eindeutiger URI der Datei."
    },
    "name": {
      "type": "string",
      "description": "Dateiname."
    },
    "size": {
      "type": "integer",
      "description": "Dateigröße in Bytes.",
      "minimum": 0
    },
    "mimeType": {
      "type": "string",
      "description": "MIME-Typ der Datei."
    },
    "lastModified": {
      "type": "string",
      "format": "date-time",
      "description": "Zeitpunkt der letzten Änderung (ISO 8601)."
    },
    "contentPreview": {
      "type": "string",
      "description": "Optionale Vorschau des Dateiinhalts (z.B. erste Zeilen einer Textdatei)."
    }
  },
  "required":
}



Beispiel: Parameterschema für ein Werkzeug createDocument
JSON{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CreateDocumentToolParams",
  "description": "Parameter für das Werkzeug 'createDocument'.",
  "type": "object",
  "properties": {
    "parentFolderUri": {
      "type": "string",
      "format": "uri",
      "description": "URI des Ordners, in dem das Dokument erstellt werden soll."
    },
    "documentName": {
      "type": "string",
      "description": "Name des zu erstellenden Dokuments.",
      "minLength": 1
    },
    "initialContent": {
      "type": "string",
      "description": "Optionaler initialer Inhalt des Dokuments."
    },
    "templateId": {
      "type": "string",
      "description": "Optionale ID einer Vorlage, die für das neue Dokument verwendet werden soll."
    }
  },
  "required": [
    "parentFolderUri",
    "documentName"
  ]
}



Beispiel: Ergebnisschema für ein Werkzeug createDocument
JSON{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CreateDocumentToolResult",
  "description": "Ergebnis des Werkzeugs 'createDocument'.",
  "type": "object",
  "properties": {
    "documentUri": {
      "type": "string",
      "format": "uri",
      "description": "URI des neu erstellten Dokuments."
    },
    "timestamp": {
      "type": "string",
      "format": "date-time",
      "description": "Zeitpunkt der Erstellung (ISO 8601)."
    }
  },
  "required": [
    "documentUri",
    "timestamp"
  ]
}





# Ultra-Feinspezifikation der MCP-Schnittstellen und Implementierungen für das NovaDE-Projekt

## 1. Einleitung

### 1.1. Zweck des Dokuments

Dieses Dokument definiert die Ultra-Feinspezifikation aller Model Context Protocol (MCP) Schnittstellen und deren Implementierungen innerhalb des NovaDE-Projekts. Es dient als maßgebliche technische Referenz für die Entwicklung, Integration und Wartung von MCP-basierten Komponenten im NovaDE-Ökosystem. Die Spezifikation umfasst detaillierte Beschreibungen von Nachrichtenformaten, Datenstrukturen, Methoden, Ereignissen und Fehlerbehandlungsmechanismen. Ein besonderer Fokus liegt auf der Integration der Domänenschicht-Spezifikation des NovaDE-Projekts in die MCP-Schnittstellen.

### 1.2. Geltungsbereich

Diese Spezifikation bezieht sich auf sämtliche Aspekte des Model Context Protocol, wie es im Kontext des NovaDE-Projekts eingesetzt wird. Dies beinhaltet:

- Alle MCP-Schnittstellen, die im NovaDE-Projekt definiert oder genutzt werden.
- Die Interaktion dieser MCP-Schnittstellen mit anderen Systemkomponenten, einschließlich, aber nicht beschränkt auf D-Bus-Dienste, Wayland-Protokolle und PipeWire-Audio-Management.
- Implementierungsrichtlinien und -details, insbesondere unter Verwendung der Programmiersprache Rust und assoziierter Bibliotheken.
- Die nahtlose Einbindung der fachlichen Anforderungen und Datenmodelle aus der Domänenschicht-Spezifikation des NovaDE-Projekts.

### 1.3. Zielgruppe

Dieses Dokument richtet sich an folgende Personengruppen innerhalb des NovaDE-Projekts:

- Softwarearchitekten und -entwickler, die MCP-Schnittstellen und -Komponenten entwerfen, implementieren oder nutzen.
- Systemintegratoren, die für die Bereitstellung und Konfiguration von NovaDE-Systemen verantwortlich sind.
- Qualitätssicherungsingenieure, die MCP-Funktionalitäten testen.
- Technische Projektmanager, die die Entwicklung und Implementierung des NovaDE-Projekts überwachen.

### 1.4. Definitionen und Akronyme

- **MCP:** Model Context Protocol. Ein offener Standard zur Kommunikation zwischen KI-Modellen/Anwendungen und externen Werkzeugen oder Datenquellen.1
- **NovaDE-Projekt:** Das spezifische Projekt, für das diese MCP-Spezifikation erstellt wird. (Details zum Projekt selbst sind außerhalb des Geltungsbereichs der bereitgestellten Materialien).
- **Domänenschicht-Spezifikation:** Ein separates Dokument, das die fachlichen Entitäten, Geschäftsregeln und Datenmodelle des NovaDE-Projekts beschreibt. Diese Spezifikation wird als integraler Bestandteil der MCP-Schnittstellendefinitionen betrachtet.
- **API:** Application Programming Interface.
- **D-Bus:** Desktop Bus, ein System für Interprozesskommunikation (IPC).3
- **Wayland:** Ein Kommunikationsprotokoll zwischen einem Display-Server (Compositor) und seinen Clients.4
- **PipeWire:** Ein Multimedia-Framework für Audio- und Videoverarbeitung unter Linux.5
- **XDG Desktop Portals:** Ein Framework, das sandboxed Anwendungen den sicheren Zugriff auf Ressourcen außerhalb der Sandbox ermöglicht.6
- **JSON-RPC:** JavaScript Object Notation Remote Procedure Call. Ein leichtgewichtiges RPC-Protokoll.8
- **Stdio:** Standard Input/Output.
- **SSE:** Server-Sent Events. Eine Technologie, die es einem Server ermöglicht, Updates an einen Client über eine HTTP-Verbindung zu pushen.8
- **Smithay:** Eine Rust-Bibliothek zur Erstellung von Wayland-Compositoren.10
- **zbus:** Eine Rust-Bibliothek für die D-Bus-Kommunikation.12
- **pipewire-rs:** Rust-Bindungen für PipeWire.14
- **mcpr:** Eine Rust-Implementierung des Model Context Protocol.16
- **mcp_client_rs:** Eine weitere Rust-Client-SDK für MCP.17

### 1.5. Referenzierte Dokumente

- Model Context Protocol Specification (Version 2025-03-26 oder aktueller) 2
- Domänenschicht-Spezifikation des NovaDE-Projekts (externes Dokument)
- Freedesktop D-Bus Specification 3
- Wayland Protocol Specification 4
- PipeWire Documentation 5
- XDG Desktop Portal Documentation 6
- Spezifikationen der relevanten D-Bus-Schnittstellen (Secrets, PolicyKit, Portals, Login1, UPower, Notifications)
- Spezifikationen der relevanten Wayland-Protokolle und -Erweiterungen
- Dokumentation der verwendeten Rust-Bibliotheken (Smithay, zbus, pipewire-rs, mcpr, mcp_client_rs, tokio, serde, thiserror etc.)

## 2. Model Context Protocol (MCP) – Grundlagen

### 2.1. Überblick und Kernkonzepte

Das Model Context Protocol (MCP) ist ein offener Standard, der darauf abzielt, die Integration von Large Language Models (LLMs) mit externen Werkzeugen, Datenbanken und APIs zu standardisieren.1 Es fungiert als eine universelle Schnittstelle, die es KI-Modellen ermöglicht, dynamisch auf Kontextinformationen zuzugreifen und Aktionen in ihrer Umgebung auszuführen.9 MCP adressiert die Herausforderung der Informationssilos und proprietären Integrationen, indem es einen einheitlichen Rahmen für die KI-Tool-Kommunikation schafft.1

Die Kernprinzipien von MCP umfassen 2:

- **Standardisierte Schnittstelle:** Einheitliche Methoden für LLMs zum Zugriff auf Werkzeuge und Ressourcen.
- **Erweiterte Fähigkeiten:** Befähigung von LLMs zur Interaktion mit diversen Systemen.
- **Sicherheit und Kontrolle:** Strukturierte Zugriffsmuster mit integrierter Validierung und klaren Grenzen.
- **Modularität und Erweiterbarkeit:** Einfaches Hinzufügen neuer Fähigkeiten durch Server, ohne die Kernanwendung des LLMs modifizieren zu müssen.

MCP ist darauf ausgelegt, die Reproduzierbarkeit von KI-Interaktionen zu verbessern, indem der gesamte notwendige Kontext (Datensätze, Umgebungsspezifikationen, Hyperparameter) an einem Ort verwaltet wird.1

### 2.2. Architektur (Client-Host-Server-Modell)

MCP basiert auf einer Client-Host-Server-Architektur 8:

- **Host:** Eine LLM-Anwendung (z.B. Claude Desktop, IDEs), die Verbindungen initiiert und als Container oder Koordinator für mehrere Client-Instanzen fungiert. Der Host verwaltet den Lebenszyklus, Sicherheitsrichtlinien (Berechtigungen, Benutzerautorisierung) und die Integration des LLMs.1
- **Client:** Eine Protokoll-Client-Komponente innerhalb der Host-Anwendung, die eine 1:1-Verbindung zu einem MCP-Server herstellt. Der Client ist verantwortlich für die Aushandlung von Fähigkeiten und die Orchestrierung von Nachrichten zwischen sich und dem Server.1
- **Server:** Ein Dienst (oft ein leichtgewichtiger Prozess), der spezifische Kontexte, Werkzeuge und Prompts für den Client bereitstellt. Server können lokale Prozesse oder entfernte Dienste sein und kapseln den Zugriff auf Datenquellen, APIs oder andere Utilities.1

Diese Architektur ermöglicht eine klare Trennung der Verantwortlichkeiten und fördert die Entwicklung modularer und wiederverwendbarer MCP-Server.23 Die Kommunikation zwischen diesen Komponenten erfolgt über eine Transportschicht und eine Protokollschicht, die auf JSON-RPC aufbaut und zustandsbehaftete Sitzungen für den Kontextaustausch und das Sampling betont.1

### 2.3. Nachrichtenformate (JSON-RPC 2.0 Basis)

MCP verwendet JSON-RPC 2.0 als Grundlage für seine Nachrichtenformate.8 Dies gewährleistet eine strukturierte und standardisierte Kommunikation. Die Hauptnachrichtentypen sind 8:

- **Requests (Anfragen):** Vom Client oder Server gesendete Nachrichten, die eine Antwort erwarten. Sie enthalten typischerweise eine `method` (Methodenname) und optionale `params` (Parameter).
    - Beispiel: `{"jsonrpc": "2.0", "method": "tools/list", "id": 1}`
- **Responses (Antworten):** Erfolgreiche Antworten auf Requests. Sie enthalten ein `result`-Feld mit den Ergebnisdaten und die `id` des ursprünglichen Requests.
    - Beispiel: `{"jsonrpc": "2.0", "result": {"tools": [...]}, "id": 1}`
- **Error Responses (Fehlerantworten):** Antworten, die anzeigen, dass ein Request fehlgeschlagen ist. Sie enthalten ein `error`-Objekt mit `code`, `message` und optional `data`, sowie die `id` des ursprünglichen Requests.
    - Beispiel: `{"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": 1}`
- **Notifications (Benachrichtigungen):** Einwegnachrichten, die keine Antwort erwarten. Sie enthalten eine `method` und optionale `params`, aber keine `id`.
    - Beispiel: `{"jsonrpc": "2.0", "method": "textDocument/didChange", "params": {...}}`

Die spezifischen Methoden und Parameter für MCP-Nachrichten wie `initialize`, `tools/list`, `resources/read`, `tools/call` werden im weiteren Verlauf dieses Dokuments detailliert [2 (angenommen)].

### 2.4. Transportmechanismen

MCP unterstützt verschiedene Transportmechanismen für die Kommunikation zwischen Host/Client und Server 8:

- **Stdio (Standard Input/Output):** Dieser Mechanismus wird für die Kommunikation mit lokalen Prozessen verwendet. Der MCP-Server läuft als separater Prozess, und die Kommunikation erfolgt über dessen Standard-Eingabe- und Ausgabe-Streams. Dies ist ideal für Kommandozeilenwerkzeuge und lokale Entwicklungsszenarien.16 Die Rust-Bibliothek `mcpr` bietet beispielsweise `StdioTransport` 16, und `mcp_client_rs` fokussiert sich ebenfalls auf diesen Transportweg für lokal gespawnte Server.18
- **HTTP mit SSE (Server-Sent Events):** Dieser Mechanismus wird für netzwerkbasierte Kommunikation verwendet, insbesondere wenn der Server remote ist oder Echtzeit-Updates vom Server an den Client erforderlich sind. SSE ermöglicht es dem Server, asynchron Nachrichten an den Client zu pushen, während Client-zu-Server-Nachrichten typischerweise über HTTP POST gesendet werden.8 Die `mcpr`-Bibliothek erwähnt SSE-Transportunterstützung.16

Die Wahl des Transportmechanismus hängt von den spezifischen Anforderungen der NovaDE-Komponente ab, insbesondere davon, ob der MCP-Server lokal oder remote betrieben wird.

### 2.5. Sicherheitsaspekte

Sicherheit und Datenschutz sind zentrale Aspekte des Model Context Protocol, da es potenziell den Zugriff auf sensible Daten und die Ausführung von Code ermöglicht.2 Die Spezifikation legt folgende Schlüsselprinzipien fest 2:

- **Benutzereinwilligung und -kontrolle:**
    - Benutzer müssen explizit allen Datenzugriffen und Operationen zustimmen und diese verstehen.
    - Benutzer müssen die Kontrolle darüber behalten, welche Daten geteilt und welche Aktionen ausgeführt werden.
    - Implementierungen sollten klare Benutzeroberflächen zur Überprüfung und Autorisierung von Aktivitäten bereitstellen.
- **Datenschutz:**
    - Hosts müssen die explizite Zustimmung des Benutzers einholen, bevor Benutzerdaten an Server weitergegeben werden.
    - Hosts dürfen Ressourcendaten nicht ohne Zustimmung des Benutzers an andere Stellen übertragen.
    - Benutzerdaten sollten durch geeignete Zugriffskontrollen geschützt werden.
- **Werkzeugsicherheit (Tool Safety):**
    - Werkzeuge repräsentieren die Ausführung von beliebigem Code und müssen mit entsprechender Vorsicht behandelt werden. Beschreibungen des Werkzeugverhaltens (z.B. Annotationen) sind als nicht vertrauenswürdig zu betrachten, es sei denn, sie stammen von einem vertrauenswürdigen Server.
    - Hosts müssen die explizite Zustimmung des Benutzers einholen, bevor ein Werkzeug aufgerufen wird.
    - Benutzer sollten verstehen, was jedes Werkzeug tut, bevor sie dessen Verwendung autorisieren.
- **LLM Sampling Controls:**
    - Benutzer müssen explizit allen LLM-Sampling-Anfragen zustimmen.
    - Benutzer sollten kontrollieren, ob Sampling überhaupt stattfindet, den tatsächlichen Prompt, der gesendet wird, und welche Ergebnisse der Server sehen kann.

Obwohl MCP diese Prinzipien nicht auf Protokollebene erzwingen kann, **SOLLTEN** Implementierer robuste Zustimmungs- und Autorisierungsflüsse entwickeln, Sicherheitsimplikationen klar dokumentieren, geeignete Zugriffskontrollen und Datenschutzmaßnahmen implementieren und bewährte Sicherheitspraktiken befolgen.2 Die Architektur mit MCP-Servern als Vermittler kann eine zusätzliche Sicherheitsebene bieten, indem der Zugriff auf Ressourcen kontrolliert und potenziell in einer Sandbox ausgeführt wird.19

## 3. MCP-Schnittstellen im NovaDE-Projekt – Allgemeine Spezifikation

### 3.1. Namenskonventionen und Versionierung

Für alle MCP-Schnittstellen, die im Rahmen des NovaDE-Projekts definiert werden, gelten folgende Namenskonventionen und Versionierungsrichtlinien:

- **Schnittstellennamen:** Schnittstellennamen folgen dem Muster `nova.<KomponentenName>.<Funktionsbereich>.<Version>`. Beispiel: `nova.workspace.fileAccess.v1`. Dies gewährleistet Eindeutigkeit und Klarheit über den Ursprung und Zweck der Schnittstelle.
- **Methodennamen:** Methodennamen verwenden camelCase, beginnend mit einem Kleinbuchstaben (z.B. `listResources`, `callTool`).
- **Parameternamen:** Parameternamen verwenden ebenfalls camelCase.
- **Versionierung:** Jede MCP-Schnittstelle wird explizit versioniert. Die Version wird als Teil des Schnittstellennamens geführt (z.B. `v1`, `v2`). Änderungen, die die Abwärtskompatibilität brechen, erfordern eine Erhöhung der Hauptversionsnummer. Abwärtskompatible Erweiterungen können zu einer Erhöhung einer Nebenversionsnummer führen, falls ein solches Schema zusätzlich eingeführt wird. Das NovaDE-Projekt hält sich an die im MCP-Standard definierte Protokollversion (z.B. `2025-03-26`).2 Die aktuell unterstützte MCP-Protokollversion ist im `mcp_client_rs` Crate als `LATEST_PROTOCOL_VERSION` und `SUPPORTED_PROTOCOL_VERSIONS` definiert.27

### 3.2. Standardnachrichtenflüsse

Die Kommunikation im NovaDE-Projekt über MCP folgt etablierten Nachrichtenflüssen, die auf dem JSON-RPC 2.0 Standard basieren.8

1. **Initialisierung (Connection Lifecycle):** 8
    - Der MCP-Client (innerhalb des NovaDE-Hosts) sendet eine `initialize`-Anfrage an den MCP-Server. Diese Anfrage enthält die vom Client unterstützte Protokollversion und dessen Fähigkeiten (Capabilities).
    - Der MCP-Server antwortet mit seiner Protokollversion und seinen Fähigkeiten.
    - Der Client bestätigt die erfolgreiche Initialisierung mit einer `initialized`-Notification.
    - Anschließend beginnt der reguläre Nachrichtenaustausch.
2. **Anfrage-Antwort (Request-Response):** 8
    - Der Client sendet eine Anfrage (z.B. `tools/list`, `resources/read`, `tools/call`) mit einer eindeutigen ID.
    - Der Server verarbeitet die Anfrage und sendet entweder eine Erfolgsantwort mit dem Ergebnis (`result`) und derselben ID oder eine Fehlerantwort (`error`) mit Fehlercode, Nachricht und derselben ID.
3. **Benachrichtigungen (Notifications):** 8
    - Client oder Server können einseitige Benachrichtigungen senden, die keine direkte Antwort erwarten. Diese haben keine ID. Ein Beispiel ist die `initialized`-Notification oder serverseitige Push-Events.
4. **Beendigung (Termination):** 8
    - Die Verbindung kann durch eine `shutdown`-Anfrage vom Client initiiert werden, gefolgt von einer `exit`-Notification. Alternativ kann die Verbindung durch Schließen des zugrundeliegenden Transportkanals beendet werden.

Die Rust-Bibliotheken `mcpr` und `mcp_client_rs` implementieren diese grundlegenden Nachrichtenflüsse.16 `mcp_client_rs` beispielsweise nutzt Tokio für asynchrone Operationen und stellt Methoden wie `initialize()`, `list_resources()`, `call_tool()` zur Verfügung, die diesen Flüssen folgen.18

### 3.3. Fehlerbehandlung und Fehlercodes

Eine robuste Fehlerbehandlung ist entscheidend für die Stabilität der MCP-Kommunikation im NovaDE-Projekt. MCP-Fehlerantworten folgen dem JSON-RPC 2.0 Standard 8 und enthalten ein `error`-Objekt mit den Feldern `code` (Integer), `message` (String) und optional `data` (beliebiger Typ).

**Standard-Fehlercodes (basierend auf JSON-RPC 2.0):**

- `-32700 Parse error`: Ungültiges JSON wurde empfangen.
- `-32600 Invalid Request`: Die JSON-Anfrage war nicht wohlgeformt.
- `-32601 Method not found`: Die angeforderte Methode existiert nicht oder ist nicht verfügbar.
- `-32602 Invalid params`: Ungültige Methodenparameter.
- `-32603 Internal error`: Interner JSON-RPC-Fehler.
- `-32000` bis `-32099 Server error`: Reserviert für implementierungsspezifische Serverfehler.

NovaDE-spezifische Fehlercodes:

Zusätzlich zu den Standard-JSON-RPC-Fehlercodes definiert das NovaDE-Projekt spezifische Fehlercodes im Bereich -32000 bis -32099 für anwendungsspezifische Fehler, die während der Verarbeitung von MCP-Anfragen auftreten können. Diese Fehlercodes werden pro Schnittstelle und Methode dokumentiert.

Fehlerbehandlung in Rust-Implementierungen:

In Rust-basierten MCP-Implementierungen für NovaDE wird die Verwendung von thiserror für Bibliotheksfehler und potenziell anyhow für Anwendungsfehler empfohlen, um eine klare und kontextreiche Fehlerbehandlung zu gewährleisten.29 Die mcp_client_rs Bibliothek stellt einen Error-Typ bereit, der verschiedene Fehlerquellen kapselt.27 Die Struktur ErrorResponse und das Enum ErrorCode [240 (angenommen)] sind Teil der Protokolldefinitionen zur strukturierten Fehlerkommunikation.

**Beispiel für eine Fehlerantwort:**

JSON

```
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "NovaDE Domain Error: Ressource nicht gefunden.",
    "data": {
      "resourceUri": "nova://domain/entity/123"
    }
  },
  "id": 123
}
```

### 3.4. Integration der Domänenschicht-Spezifikation

Die Domänenschicht-Spezifikation des NovaDE-Projekts ist ein zentrales Element, das die fachlichen Entitäten, Operationen und Geschäftsregeln definiert. Die MCP-Schnittstellen im NovaDE-Projekt müssen diese Domänenspezifikation nahtlos integrieren. Dies bedeutet:

- **Abbildung von Domänenentitäten:** Datenstrukturen innerhalb der MCP-Nachrichten (Parameter von Methoden, Rückgabewerte, Event-Payloads) müssen die Entitäten der Domänenschicht widerspiegeln oder direkt verwenden. Dies stellt sicher, dass die MCP-Kommunikation die fachlichen Anforderungen korrekt abbildet.
- **Domänenoperationen als MCP-Methoden:** Viele MCP-Methoden werden direkte Abbildungen von Operationen sein, die in der Domänenschicht definiert sind. Die Parameter und Rückgabewerte dieser MCP-Methoden korrespondieren mit den Ein- und Ausgaben der Domänenoperationen.
- **Validierung und Geschäftsregeln:** Bevor MCP-Anfragen an die Domänenschicht weitergeleitet oder Ergebnisse von der Domänenschicht über MCP zurückgegeben werden, müssen Validierungen und Geschäftsregeln der Domänenschicht angewendet werden. Dies kann sowohl im MCP-Server als auch in einer zwischengeschalteten Logikschicht geschehen.
- **Konsistente Terminologie:** Die in den MCP-Schnittstellen verwendete Terminologie (Namen von Methoden, Parametern, Datenfeldern) sollte mit der Terminologie der Domänenschicht-Spezifikation übereinstimmen, um Missverständnisse zu vermeiden und die Kohärenz im gesamten System zu fördern.

Die genauen Details der Integration hängen von den spezifischen Inhalten der Domänenschicht-Spezifikation ab. Jede detaillierte MCP-Schnittstellenspezifikation in Abschnitt 4 wird explizit auf die relevanten Teile der Domänenschicht-Spezifikation verweisen und die Abbildung erläutern.

## 4. Detaillierte MCP-Schnittstellenspezifikationen

Für das NovaDE-Projekt werden spezifische MCP-Schnittstellen definiert, um die Interaktion mit verschiedenen Modulen und Diensten zu ermöglichen. Jede Schnittstelle wird nach dem folgenden Schema spezifiziert. _Da die konkreten Schnittstellen für NovaDE nicht in den bereitgestellten Materialien definiert sind, dient der folgende Abschnitt als Vorlage und Beispielstruktur._

---

**Beispiel-Schnittstelle: `nova.dataAccess.document.v1`**

### 4.1. Beschreibung und Zweck

Die Schnittstelle `nova.dataAccess.document.v1` dient dem Zugriff auf und der Verwaltung von Dokumenten innerhalb des NovaDE-Projekts. Sie ermöglicht es MCP-Clients, Dokumente basierend auf Kriterien der Domänenschicht zu lesen, zu schreiben, zu aktualisieren und zu löschen. Diese Schnittstelle interagiert eng mit den Entitäten und Operationen, die in der "Domänenschicht-Spezifikation, Abschnitt X.Y (Dokumentenverwaltung)" definiert sind.

### 4.2. Methoden

#### 4.2.1. `readDocument`

- **Beschreibung:** Liest den Inhalt und die Metadaten eines spezifischen Dokuments.
- **Parameter:**
    - `uri` (String, erforderlich): Der eindeutige URI des Dokuments, konform zum NovaDE-URI-Schema (z.B. `nova://documents/internal/doc123`).
    - `options` (Object, optional): Zusätzliche Optionen für den Lesevorgang.
        - `version` (String, optional): Die spezifische Version des Dokuments, die gelesen werden soll. Falls nicht angegeben, wird die neueste Version gelesen.
- **Rückgabewerte:**
    - `document` (Object): Ein Objekt, das das gelesene Dokument repräsentiert. Die Struktur dieses Objekts ist in der Domänenschicht-Spezifikation definiert und könnte Felder wie `uri`, `mimeType`, `content` (String oder Binary), `metadata` (Object), `version` (String), `lastModified` (Timestamp) enthalten.
- **Mögliche Fehler:**
    - `-32001`: `DOCUMENT_NOT_FOUND` - Das angeforderte Dokument existiert nicht.
    - `-32002`: `ACCESS_DENIED` - Der Client hat keine Berechtigung, das Dokument zu lesen.
    - `-32003`: `VERSION_NOT_FOUND` - Die angeforderte Version des Dokuments existiert nicht.

#### 4.2.2. `writeDocument`

- **Beschreibung:** Schreibt ein neues Dokument oder aktualisiert ein bestehendes Dokument.
- **Parameter:**
    - `uri` (String, erforderlich): Der URI, unter dem das Dokument geschrieben werden soll. Bei Aktualisierung eines bestehenden Dokuments dessen URI.
    - `content` (String oder Binary, erforderlich): Der Inhalt des Dokuments. Der Typ (String oder Base64-kodiertes Binary) hängt vom `mimeType` ab.
    - `mimeType` (String, erforderlich): Der MIME-Typ des Dokuments (z.B. `text/plain`, `application/pdf`).
    - `metadata` (Object, optional): Domänenspezifische Metadaten für das Dokument.
    - `options` (Object, optional):
        - `overwrite` (Boolean, optional, default: `false`): Wenn `true` und ein Dokument unter dem URI existiert, wird es überschrieben. Andernfalls schlägt der Aufruf fehl, wenn das Dokument existiert.
- **Rückgabewerte:**
    - `newUri` (String): Der URI des geschriebenen oder aktualisierten Dokuments (kann sich bei Neuerstellung ändern, falls der Server URIs generiert).
    - `version` (String): Die Versionskennung des geschriebenen Dokuments.
- **Mögliche Fehler:**
    - `-32002`: `ACCESS_DENIED` - Keine Schreibberechtigung.
    - `-32004`: `DOCUMENT_EXISTS` - Dokument existiert bereits und `overwrite` ist `false`.
    - `-32005`: `INVALID_CONTENT` - Der bereitgestellte Inhalt ist für den `mimeType` ungültig.

_(Weitere Methoden wie `deleteDocument`, `listDocuments` würden hier analog spezifiziert werden.)_

### 4.3. Events/Notifications

#### 4.3.1. `documentChanged`

- **Beschreibung:** Wird vom Server gesendet, wenn ein Dokument, für das der Client möglicherweise Interesse bekundet hat (z.B. durch vorheriges Lesen), geändert wurde.
- **Parameter:**
    - `uri` (String): Der URI des geänderten Dokuments.
    - `changeType` (String): Art der Änderung (z.B. `UPDATED`, `DELETED`).
    - `newVersion` (String, optional): Die neue Versionskennung, falls `changeType` `UPDATED` ist.

### 4.4. Datenstrukturen

Die für diese Schnittstelle relevanten Datenstrukturen (z.B. die Struktur eines `Document`-Objekts, `Metadata`-Objekts) werden primär durch die Domänenschicht-Spezifikation des NovaDE-Projekts definiert. MCP-Nachrichten verwenden JSON-Repräsentationen dieser domänenspezifischen Strukturen.

**Beispiel `Document` (basierend auf einer hypothetischen Domänenspezifikation):**

JSON

```
{
  "uri": "nova://documents/internal/doc123",
  "mimeType": "text/plain",
  "content": "Dies ist der Inhalt des Dokuments.",
  "metadata": {
    "author": "NovaUser",
    "tags": ["wichtig", "projektA"],
    "customDomainField": "spezifischerWert"
  },
  "version": "v1.2.3",
  "lastModified": "2024-07-15T10:30:00Z"
}
```

### 4.5. Beispiele für Nachrichten

**Anfrage `readDocument`:**

JSON

```
{
  "jsonrpc": "2.0",
  "method": "nova.dataAccess.document.v1/readDocument",
  "params": {
    "uri": "nova://documents/internal/doc123"
  },
  "id": 1
}
```

**Antwort `readDocument` (Erfolg):**

JSON

```
{
  "jsonrpc": "2.0",
  "result": {
    "document": {
      "uri": "nova://documents/internal/doc123",
      "mimeType": "text/plain",
      "content": "Dies ist der Inhalt des Dokuments.",
      "metadata": {"author": "NovaUser"},
      "version": "v1.0.0",
      "lastModified": "2024-07-15T10:00:00Z"
    }
  },
  "id": 1
}
```

### 4.6. Interaktion mit der Domänenschicht

Die Methode `readDocument` ruft intern die Funktion `DomainLayer.getDocumentByUri(uri, options.version)` der Domänenschicht auf. Die zurückgegebenen Domänenobjekte werden gemäß den MCP-Datenstrukturen serialisiert. Die Methode `writeDocument` validiert die Eingaben anhand der Geschäftsregeln der Domänenschicht (z.B. `DomainLayer.validateDocumentContent(content, mimeType)`) und ruft dann `DomainLayer.saveDocument(documentData)` auf. Berechtigungsprüfungen erfolgen ebenfalls über dedizierte Domänenschicht-Services (z.B. `DomainLayer.Security.canReadDocument(userContext, uri)`).

---

_(Dieser beispielhafte Abschnitt würde für jede spezifische MCP-Schnittstelle im NovaDE-Projekt wiederholt werden.)_

## 5. Implementierung der MCP-Schnittstellen im NovaDE-Projekt

### 5.1. Verwendete Technologien

Die Kernimplementierung der MCP-Schnittstellen und der zugehörigen Logik im NovaDE-Projekt erfolgt in **Rust**. Dies schließt sowohl Client- als auch (potenzielle) Server-seitige Komponenten ein. Die Wahl von Rust begründet sich in dessen Stärken hinsichtlich Systemsicherheit, Performance und Nebenläufigkeit, welche für ein robustes Desktop Environment Projekt wie NovaDE essentiell sind.

Folgende Rust-Bibliotheken (Crates) sind für die MCP-Implementierung von zentraler Bedeutung:

- **MCP-Protokoll-Handling:**
    - `mcp_client_rs` (von darinkishore) [17 (angenommen), 241 (angenommen), 28 (angenommen), 243 (angenommen), 244 (angenommen), 243 (angenommen), 242 (angenommen), 245 (angenommen), 246 (angenommen), 246 (angenommen)] oder alternativ `mcpr` (von conikeec) 16 für die Client-seitige Implementierung. Die Entscheidung für eine spezifische Bibliothek hängt von den detaillierten Anforderungen und der Reife der jeweiligen Bibliothek zum Zeitpunkt der Implementierung ab. Beide bieten Mechanismen zur Serialisierung/Deserialisierung von MCP-Nachrichten und zur Verwaltung der Kommunikation.
- **Asynchrone Laufzeitumgebung:** `tokio` wird als primäre asynchrone Laufzeitumgebung für die nebenläufige Verarbeitung von MCP-Nachrichten und Interaktionen mit anderen Systemdiensten verwendet.25
- **Serialisierung/Deserialisierung:** `serde` und `serde_json` für die Umwandlung von Rust-Datenstrukturen in und aus dem JSON-Format, das von JSON-RPC verwendet wird.25
- **Fehlerbehandlung:** `thiserror` für die Definition von benutzerdefinierten Fehlertypen in Bibliotheks-Code und potenziell `anyhow` für eine vereinfachte Fehlerbehandlung in Anwendungscode.29
- **UUID-Generierung:** Das `uuid` Crate mit den Features `v4` und `serde` wird für die Erzeugung und Handhabung von eindeutigen Identifikatoren verwendet, die in MCP-Nachrichten oder domänenspezifischen Daten benötigt werden könnten.41
- **D-Bus-Kommunikation:** `zbus` für die Interaktion mit Systemdiensten über D-Bus.12
- **Wayland Compositing (falls NovaDE ein Compositor ist oder tief integriert):** `smithay` als Framework für Wayland-spezifische Interaktionen.10
- **PipeWire-Integration:** `pipewire-rs` für die Interaktion mit dem PipeWire Multimedia-Framework.14

### 5.2. MCP-Client-Implementierung (Rust)

Die MCP-Client-Komponenten im NovaDE-Projekt sind für die Kommunikation mit verschiedenen MCP-Servern zuständig, die Werkzeuge und Ressourcen bereitstellen.

#### 5.2.1. Initialisierung und Verbindungsaufbau

Die Initialisierung eines MCP-Clients beginnt mit der Konfiguration des Transports und der Erstellung einer Client-Instanz. Am Beispiel von `mcp_client_rs` (darinkishore):

- **Server-Spawning (für lokale Server via Stdio):** Die `ClientBuilder`-API ermöglicht das Starten eines lokalen MCP-Serverprozesses und die Verbindung zu dessen Stdio-Kanälen.17
    
    Rust
    
    ```
    // Beispielhafte Initialisierung (Pseudocode, da Servername und Argumente spezifisch für NovaDE sind)
    // use mcp_client_rs::client::ClientBuilder;
    // let client = ClientBuilder::new("nova-mcp-server-executable")
    //    .arg("--config-path")
    //    .arg("/etc/nova/mcp_server_config.json")
    //    .spawn_and_initialize().await?;
    ```
    
    Es ist wichtig zu beachten, dass `mcp_client_rs` (darinkishore) primär für lokal gespawnte Server konzipiert ist und keine direkte Unterstützung für Remote-Server plant.17 Für Remote-Verbindungen via HTTP/SSE müsste eine andere Bibliothek oder eine Erweiterung dieses Ansatzes in Betracht gezogen werden, wie sie z.B. in `mcpr` (conikeec) angedeutet ist.16
    
- **Verwendung eines existierenden Transports:** Alternativ kann ein Client mit einem bereits existierenden Transportobjekt initialisiert werden.14
    
    Rust
    
    ```
    // use std::sync::Arc;
    // use mcp_client_rs::client::Client;
    // use mcp_client_rs::transport::stdio::StdioTransport;
    // use tokio::io::{stdin, stdout};
    //
    // let transport = StdioTransport::with_streams(stdin(), stdout());
    // let client = Client::new(Arc::new(transport));
    ```
    
- **`initialize`-Nachricht:** Nach dem Aufbau der Transportverbindung sendet der Client eine `initialize`-Anfrage an den Server, um Protokollversionen und Fähigkeiten auszutauschen.8 Die `spawn_and_initialize()`-Methode von `mcp_client_rs` (darinkishore) handhabt dies implizit.17 Die `initialize()`-Methode auf der Client-Instanz von `mcpr` (conikeec) führt dies explizit durch.16
    

Die `InitializeParams` [240 (angenommen)] würden typischerweise die `protocolVersion` (z.B. "2025-03-26"), `clientName`, `clientVersion` und `supportedFeatures` enthalten. Die `InitializeResult` [240 (angenommen)] vom Server bestätigt die `protocolVersion` und listet die `serverCapabilities` und `serverInfo` auf.

#### 5.2.2. Senden von Requests

MCP-Clients im NovaDE-Projekt senden Anfragen an Server, um Ressourcen aufzulisten, Werkzeuge aufzurufen oder andere definierte Operationen auszuführen.

- **`ListResources`:**
    - Parameter: `ListResourcesParams` [240 (angenommen)] können Filterkriterien oder Paginierungsinformationen enthalten.
    - Antwort: `ListResourcesResult` [240 (angenommen)] enthält eine Liste von `Resource`-Objekten [240 (angenommen)], die jeweils URI, Name, Beschreibung und unterstützte Operationen definieren.
    - Beispielaufruf mit `mcp_client_rs`: `let resources = client.list_resources().await?;` 18
- **`CallTool`:**
    - Parameter: `CallToolParams` [240 (angenommen)] enthalten den `toolName` (String) und `arguments` (JSON-Objekt).
    - Antwort: `CallToolResult` [240 (angenommen)] enthält das Ergebnis der Werkzeugausführung, typischerweise als JSON-Objekt.
    - Beispielaufruf mit `mcp_client_rs`: `let tool_result = client.call_tool("domain.action.calculateSum", serde_json::json!({"op1": 10, "op2": 20})).await?;` 18
    - Die Definition von Werkzeugen (`ToolDefinition` [240 (angenommen)]) umfasst Name, Beschreibung und ein JSON-Schema für die Parameter.
- **`ReadResource`:** (und andere domänenspezifische Requests)
    - Parameter: Typischerweise ein URI und optionale Parameter.
    - Antwort: Der Inhalt oder Zustand der Ressource.
    - Beispielaufruf mit `mcp_client_rs`: `let read_result = client.read_resource("nova://domain/entity/123").await?;` 18

Alle diese Anfragen werden asynchron über den konfigurierten Transportmechanismus gesendet. Die `mcp_client_rs` Bibliothek nutzt Tokio für diese asynchronen Operationen.25

#### 5.2.3. Empfangen von Responses und Notifications

Der Empfang von Nachrichten ist ein kritischer Aspekt der MCP-Client-Implementierung.

- **Responses:** Antworten auf Client-Anfragen werden typischerweise über `async/await` Konstrukte direkt als Rückgabewerte der aufrufenden Methoden empfangen (z.B. `ListResourcesResult` von `list_resources().await?`).18 Die zugrundeliegende Transportlogik (z.B. in `StdioTransport` [242 (angenommen), 244 (angenommen), 242 (angenommen)]) liest die Rohdaten, parst sie als `McpMessage` [240 (angenommen)] und leitet sie an den entsprechenden wartenden Task weiter.
- **Notifications (Server Push Events):** Asynchrone Benachrichtigungen vom Server (z.B. `documentChanged` aus dem Beispiel in Abschnitt 4.3.1) erfordern einen dedizierten Mechanismus zum Empfang und zur Verarbeitung.
    - Die `mcpr` Bibliothek (conikeec) deutet auf Unterstützung für Server-Sent Events (SSE) hin, was einen Stream von Ereignissen impliziert, den der Client verarbeiten müsste.16
    - Die `mcp_client_rs` Bibliothek (darinkishore) ist primär auf Stdio ausgerichtet. Die Handhabung von Server-Push-Benachrichtigungen über Stdio würde erfordern, dass der `StdioTransport` kontinuierlich die Standardeingabe liest und eingehende Nachrichten (die keine direkten Antworten auf Anfragen sind) als `Notification` [240 (angenommen)] identifiziert und an einen Handler oder einen Ereignis-Stream weiterleitet. Die genaue Implementierung (z.B. ein dedizierter Empfangs-Loop oder ein Stream von `McpMessage`) ist in den bereitgestellten Snippets nicht vollständig ersichtlich [17 (fehlend), 246 (fehlend), 241 (fehlend), 243 (fehlend), 243 (fehlend), 245 (fehlend), 246 (fehlend), 246 (fehlend)]. Es ist davon auszugehen, dass eine `async_stream`-basierte Lösung oder ein `tokio::sync::broadcast` Kanal 36 verwendet wird, um diese Nachrichten an interessierte Teile der Anwendung zu verteilen.
    - Die `mcp_client_rs` Version 0.1.1 erwähnt "WebSocket Transport (Coming Soon)" mit "built-in reconnection handling", was auf zukünftige robustere Mechanismen für Server-Push und Verbindungsstatus hindeutet.25
- **Connection Status Events:** Die Überwachung des Verbindungsstatus (z.B. Verbindungsabbruch, Wiederverbindung) ist für robuste Anwendungen wichtig. Explizite Mechanismen hierfür sind in den Snippets zu `mcp_client_rs` (darinkishore) nicht detailliert, könnten aber Teil des `WebSocketTransport` sein 25 oder müssten auf der Transportebene (z.B. durch Überwachung der Stdio-Pipes) implementiert werden. Für SSE-Transporte könnten HTTP-Fehlercodes oder das Schließen des Event-Streams als Indikatoren dienen.26

#### 5.2.4. Fehlerbehandlung

Fehler können auf verschiedenen Ebenen auftreten: Transportfehler, JSON-RPC-Parsingfehler, oder anwendungsspezifische Fehler, die vom Server als `ErrorResponse` [240 (angenommen)] gesendet werden.

- Die `mcp_client_rs` Bibliothek verwendet `thiserror` zur Definition ihres `Error`-Typs, der verschiedene Fehlerquellen aggregiert.27
- Client-Code sollte `Result`-Typen sorgfältig behandeln, um auf Fehler angemessen reagieren zu können (z.B. Wiederholungsversuche, Benutzerbenachrichtigung, Logging).
- Spezifische `ErrorCode`-Werte [240 (angenommen)] in `ErrorResponse`-Nachrichten ermöglichen eine differenzierte Fehlerbehandlung basierend auf der Art des serverseitigen Fehlers.

#### 5.2.5. Transport Layer

- **StdioTransport:** Für die Kommunikation mit lokalen Serverprozessen. Implementierungen in `mcpr` 16 und `mcp_client_rs` [25 (angenommen), 244 (angenommen), 242 (angenommen)] lesen von `stdin` und schreiben nach `stdout` des Subprozesses. Die `StdioTransport` in `mcp_client_rs` verwendet typischerweise `tokio::io::AsyncRead` und `tokio::io::AsyncWrite` für die asynchrone Verarbeitung. Eingehende Nachrichten werden zeilenbasiert oder durch Längenpräfixe (gemäß JSON-RPC Framing) gelesen und dann als `McpMessage` deserialisiert.
- **SSETransport (Server-Sent Events):** Für webbasierte oder Remote-Server. `mcpr` erwähnt dessen Unterstützung.16 Dies involviert einen HTTP-Client, der eine Verbindung zu einem SSE-Endpunkt des Servers herstellt und einen kontinuierlichen Stream von Ereignissen empfängt.26

### 5.3. MCP-Server-Implementierung (Rust)

Obwohl der primäre Fokus des NovaDE-Projekts auf der Client-Seite liegen mag, könnten bestimmte Komponenten des Projekts auch als MCP-Server fungieren, um Fähigkeiten für andere Teile des Systems oder externe LLMs bereitzustellen.

- **Struktur:** Ein MCP-Server in Rust, beispielsweise unter Verwendung der `mcpr`-Bibliothek 16, würde eine `ServerConfig` definieren, die Name, Version und eine Liste der bereitgestellten `Tool`-Definitionen enthält. Jedes `Tool` spezifiziert seinen Namen, eine Beschreibung und ein JSON-Schema für seine Parameter.
- **Tool-Registrierung:** Für jedes definierte Werkzeug wird ein Handler registriert, der die Werkzeugparameter entgegennimmt, die Logik ausführt und ein Ergebnis (oder einen Fehler) zurückgibt.
    
    Rust
    
    ```
    // Beispielhafte Server-Konfiguration mit mcpr (conikeec)
    // use mcpr::{server::{Server, ServerConfig}, transport::stdio::StdioTransport, Tool, error::MCPError};
    // use serde_json::Value;
    //
    // let server_config = ServerConfig::new()
    //    .with_name("NovaDE.DomainService.v1")
    //    .with_version("1.0.0")
    //    .with_tool(Tool {
    //         name: "nova.domain.getEntityDetails".to_string(),
    //         description: Some("Ruft Details zu einer Domänenentität ab.".to_string()),
    //         parameters_schema: serde_json::json!({
    //             "type": "object",
    //             "properties": {
    //                 "entityUri": {"type": "string", "description": "URI der Entität"}
    //             },
    //             "required": ["entityUri"]
    //         }),
    //     });
    // let mut server: Server<StdioTransport> = Server::new(server_config);
    // server.register_tool_handler("nova.domain.getEntityDetails", |params: Value| {
    //     // Implementierung der Domänenlogik hier
    //     //...
    //     Ok(serde_json::json!({"status": "success", "data": { /*... */ }}))
    // })?;
    // let transport = StdioTransport::new();
    // server.start(transport)?;
    ```
    

Die Serverimplementierung ist verantwortlich für das Parsen eingehender Anfragen, das Weiterleiten an die entsprechenden Handler und das Senden von Antworten oder Benachrichtigungen über den gewählten Transportmechanismus.

### 5.4. Interaktion mit Systemdiensten und Protokollen

Die MCP-Schnittstellen im NovaDE-Projekt sind nicht isoliert, sondern interagieren intensiv mit bestehenden Systemdiensten und Protokollen. Diese Interaktionen sind entscheidend für den Zugriff auf Systemressourcen, die Verwaltung von Berechtigungen und die Integration in die Desktop-Umgebung. Die folgenden Abschnitte detaillieren diese Interaktionen.

## 6. Interaktion mit D-Bus-Diensten

Die Kommunikation mit systemweiten und benutzerspezifischen Diensten im NovaDE-Projekt erfolgt primär über D-Bus. Die Rust-Bibliothek `zbus` wird für diese Interaktionen verwendet.12

### 6.1. Allgemeine D-Bus-Integration mit `zbus`

`zbus` ermöglicht eine typsichere und asynchrone Kommunikation mit D-Bus-Diensten.

- **Proxy-Generierung:** Für die Interaktion mit D-Bus-Schnittstellen werden Proxys verwendet. Das `#[dbus_proxy]` (oder `#[proxy]`) Makro von `zbus` generiert Rust-Traits und Proxy-Strukturen aus D-Bus-Interface-Definitionen [12 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich)].
    
    Rust
    
    ```
    // use zbus::{dbus_proxy, Connection, Result};
    // #
    // trait ExampleProxy {
    //     async fn some_method(&self, param: &str) -> Result<String>;
    //     #[dbus_proxy(signal)]
    //     async fn some_signal(&self, value: u32) -> Result<()>;
    // }
    ```
    
- **Verbindungsaufbau:** Eine Verbindung zum Session- oder Systembus wird mit `zbus::Connection::session().await?` bzw. `zbus::Connection::system().await?` hergestellt.45
- **Methodenaufrufe:** Methoden auf D-Bus-Schnittstellen werden asynchron über die generierten Proxy-Methoden aufgerufen.45
- **Signalempfang:** Signale werden als asynchrone Streams (`futures_util::stream::StreamExt`) über die Proxy-Methoden `receive_<signal_name>()` empfangen.46 Die Argumente des Signals können aus der `zbus::Message` deserialisiert werden [46 (nicht zugänglich), 65 (nicht zugänglich)].
- **Fehlerbehandlung:** `zbus`-Operationen geben `zbus::Result` zurück. Fehler werden durch das `zbus::Error`-Enum repräsentiert, das verschiedene Fehlerquellen wie I/O-Fehler, ungültige Nachrichten oder Fehler vom D-Bus-Dienst selbst abdeckt.52
- **Server-Implementierung:** `zbus` ermöglicht auch die Implementierung von D-Bus-Diensten. Mittels `ConnectionBuilder::serve_at` können Interfaces auf bestimmten Objektpfaden bereitgestellt werden [13 (nicht zugänglich), 57 (nicht zugänglich), 12 (nicht zugänglich), 56 (nicht zugänglich), 57 (nicht zugänglich), 12 (nicht zugänglich), 60]. Das `ObjectServer`-API kann für komplexere Szenarien mit mehreren Objekten und Interfaces auf derselben Verbindung genutzt werden [48 (nicht zugänglich), 57 (nicht zugänglich), 12 (nicht zugänglich), 56 (nicht zugänglich), 57 (nicht zugänglich), 58 (nicht zugänglich), 60].

### 6.2. `org.freedesktop.secrets` – Sichere Speicherung von Geheimnissen

Das NovaDE-Projekt nutzt die `org.freedesktop.Secrets`-Schnittstelle für die sichere Speicherung und Verwaltung von sensiblen Daten wie Passwörtern oder API-Tokens, die von MCP-Komponenten benötigt werden.70

- **Schnittstellenspezifikation:** 70
    - **`org.freedesktop.Secrets.Service`:** Verwalter von Collections und Sessions.
        - Methoden: `OpenSession`, `CreateCollection`, `SearchCollections`, `RetrieveSecrets`, `LockService`, `DeleteCollection`.
        - Signale: `CollectionCreated`, `CollectionDeleted`.
        - Properties: `Collections` (RO), `DefaultCollection` (RW).
    - **`org.freedesktop.Secrets.Collection`:** Eine Sammlung von Items (Geheimnissen).
        - Methoden: `Delete`, `SearchItems`, `CreateItem`.
        - Signale: `CreatedItem`, `DeletedItem`.
        - Properties: `Items` (RO), `Private` (RO), `Label` (RW), `Locked` (RO), `Created` (RO), `Modified` (RO).
    - **`org.freedesktop.Secrets.Item`:** Ein einzelnes Geheimnis mit Attributen.
        - Methoden: `Delete`.
        - Signale: `changed`.
        - Properties: `Locked` (RO), `Attributes` (RW), `Label` (RW), `Secret` (RW), `Created` (RO), `Modified` (RO).
    - **`org.freedesktop.Secrets.Session`:** Repräsentiert eine Sitzung zwischen Client und Dienst.
        - Methoden: `Close`, `Negotiate`, `BeginAuthenticate`, `CompleteAuthenticate`.
        - Signale: `Authenticated`.
- **Datentyp `Secret`:** 70
    - `algorithm` (String): Algorithmus zur Kodierung des Geheimnisses (z.B. "PLAIN").
    - `parameters` (Array<Byte>): Algorithmus-spezifische Parameter.
    - `value` (Array<Byte>): Der möglicherweise kodierte Geheimniswert.
- **Fehlerdomäne:** `org.freedesktop.Secrets.Error.*` (z.B. `AlreadyExists`, `IsLocked`, `NotSupported`).70
- **Nutzung im NovaDE-Projekt für MCP:**
    - MCP-Server oder -Clients im NovaDE-Projekt, die Zugriff auf persistente, sichere Anmeldeinformationen oder Tokens benötigen, interagieren mit diesem Dienst.
    - Beispiel: Ein MCP-Server, der eine Verbindung zu einem externen API herstellt, könnte das API-Token sicher über `org.freedesktop.Secrets` speichern und abrufen.
    - Die `CreateCollection`-Methode wird verwendet, um spezifische Sammlungen für NovaDE-Komponenten anzulegen, potenziell mit `private = true`, um den Zugriff einzuschränken.
    - `SearchItems` mit spezifischen `Attributes` wird verwendet, um gezielt nach Geheimnissen zu suchen.
    - Die `Negotiate`-Methode kann für eine verschlüsselte Übertragung der Geheimnisse genutzt werden, falls erforderlich.

### 6.3. `org.freedesktop.PolicyKit1.Authority` – Berechtigungsprüfung

PolicyKit wird im NovaDE-Projekt eingesetzt, um granulare Berechtigungsprüfungen für Aktionen durchzuführen, die von MCP-Schnittstellen oder den dahinterliegenden Diensten ausgelöst werden.71

- **Schnittstellenspezifikation:** `org.freedesktop.PolicyKit1.Authority` am Pfad `/org/freedesktop/PolicyKit1/Authority`.71
    - **Methoden:**
        - `CheckAuthorization(IN Subject subject, IN String action_id, IN Dict<String,String> details, IN CheckAuthorizationFlags flags, IN String cancellation_id, OUT AuthorizationResult result)`: Prüft, ob ein Subjekt berechtigt ist, eine Aktion auszuführen. `details` können verwendet werden, um kontextspezifische Informationen für die Autorisierungsentscheidung oder die dem Benutzer angezeigte Nachricht bereitzustellen. `AllowUserInteraction` im `flags`-Parameter ermöglicht eine interaktive Authentifizierung.
        - `CancelCheckAuthorization(IN String cancellation_id)`: Bricht eine laufende Autorisierungsprüfung ab.
        - `EnumerateActions(IN String locale, OUT Array<ActionDescription> action_descriptions)`: Listet alle registrierten Aktionen auf.
        - `RegisterAuthenticationAgent(IN Subject subject, IN String locale, IN String object_path)`: Registriert einen Authentifizierungsagenten.
    - **Signale:**
        - `Changed()`: Wird emittiert, wenn sich Aktionen oder Autorisierungen ändern.
- **Wichtige Datenstrukturen:** 71
    - `Subject`: Beschreibt das handelnde Subjekt (z.B. `unix-process`, `unix-session`).
    - `ActionDescription`: Beschreibt eine registrierte Aktion (ID, Beschreibung, Nachricht, Standardberechtigungen).
    - `AuthorizationResult`: Ergebnis der Prüfung (`is_authorized`, `is_challenge`, `details`).
- **Nutzung im NovaDE-Projekt für MCP-Zugriffssteuerung:**
    - Bevor eine MCP-Methode eine potenziell privilegierte Operation ausführt (z.B. Systemkonfiguration ändern, auf geschützte Benutzerdaten zugreifen), muss der MCP-Server oder die aufgerufene NovaDE-Komponente `CheckAuthorization` aufrufen.
    - Die `action_id` entspricht einer vordefinierten Aktion im NovaDE-PolicyKit-Regelwerk (z.B. `org.novade.mcp.filesystem.writeFile`).
    - Die `details` können MCP-spezifische Parameter enthalten, die für die Entscheidung oder die Benutzerinteraktion relevant sind.
    - Das Ergebnis von `CheckAuthorization` bestimmt, ob die MCP-Operation fortgesetzt oder mit einem Berechtigungsfehler abgelehnt wird.

### 6.4. XDG Desktop Portals

XDG Desktop Portals bieten sandboxed Anwendungen (und auch nicht-sandboxed Anwendungen) einen standardisierten Weg, um mit der Desktop-Umgebung zu interagieren, z.B. für Dateiauswahl oder Screenshots.6 MCP-Schnittstellen im NovaDE-Projekt können diese Portale nutzen, um solche Interaktionen sicher und konsistent zu gestalten.

#### 6.4.1. `org.freedesktop.portal.FileChooser`

Wird verwendet, um dem Benutzer Dialoge zur Datei- oder Ordnerauswahl anzuzeigen.6

- **Methoden:** 73
    - `OpenFile(IN String parent_window, IN String title, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Öffnet einen Dialog zur Auswahl einer oder mehrerer Dateien.
        - Optionen (`a{sv}`): `handle_token` (s), `accept_label` (s), `modal` (b), `multiple` (b), `directory` (b), `filters` (`a(sa(us))`), `current_filter` (`(sa(us))`), `choices` (`a(ssa(ss)s)`), `current_folder` (ay).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uris` (as), `choices` (`a(ss)`), `current_filter` (`(sa(us))`).
    - `SaveFile(IN String parent_window, IN String title, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Öffnet einen Dialog zum Speichern einer Datei.
        - Optionen (`a{sv}`): Ähnlich wie `OpenFile`, zusätzlich `current_name` (s), `current_file` (ay).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uris` (as), `choices` (`a(ss)`), `current_filter` (`(sa(us))`).
    - `SaveFiles(IN String parent_window, IN String title, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Öffnet einen Dialog zum Speichern mehrerer Dateien in einem Ordner.
        - Optionen (`a{sv}`): Ähnlich wie `SaveFile`, zusätzlich `files` (aay).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uris` (as), `choices` (`a(ss)`).
- **Nutzung im NovaDE-Projekt:** MCP-Methoden, die Dateiinteraktionen erfordern (z.B. das Hochladen eines Dokuments durch den Benutzer, das Speichern von generierten Inhalten), rufen diese Portalmethoden auf. Die `parent_window`-Kennung muss korrekt übergeben werden. Die `options` werden basierend auf dem Kontext der MCP-Aktion befüllt (z.B. Dateifilter basierend auf erwarteten MIME-Typen der Domänenschicht).

#### 6.4.2. `org.freedesktop.portal.Screenshot`

Ermöglicht das Erstellen von Screenshots und das Auswählen von Pixelfarben.3

- **Methoden (Version 2):** 79
    - `Screenshot(IN String parent_window, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Erstellt einen Screenshot.
        - Optionen (`a{sv}`): `handle_token` (s), `modal` (b, default: true), `interactive` (b, default: false, seit v2).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uri` (s) des Screenshots.
    - `PickColor(IN String parent_window, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Wählt die Farbe eines Pixels aus.
        - Optionen (`a{sv}`): `handle_token` (s).
        - Antwort über `org.freedesktop.portal.Request::Response`: `color` (`(ddd)`) als RGB-Werte .
- **Nutzung im NovaDE-Projekt:** MCP-Schnittstellen, die visuelle Informationen vom Desktop benötigen (z.B. ein Werkzeug zur Fehlerberichterstattung, das einen Screenshot anhängt, oder ein Design-Tool, das Farben vom Bildschirm aufnimmt), verwenden diese Portalmethoden.
- **Backend-Implementierung:** Für Wayland-basierte Desktops wie potenziell NovaDE ist eine Backend-Implementierung wie `xdg-desktop-portal-wlr` 6, `xdg-desktop-portal-gnome` 6, `xdg-desktop-portal-kde` 6 oder eine spezifische Implementierung wie `xdg-desktop-portal-luminous` (Rust-basiert, nutzt `libwayshot` und `zwlr_screencopy`) 83 erforderlich. `xdg-desktop-portal-luminous` ist ein Beispiel für eine Rust-basierte Implementierung, die `zbus` für D-Bus verwenden könnte und über das `zwlr_screencopy`-Protokoll mit wlroots-basierten Compositors interagiert.88

### 6.5. `org.freedesktop.login1` – Sitzungs- und Benutzerverwaltung

Der `systemd-logind`-Dienst stellt über D-Bus Informationen und Kontrollfunktionen für Benutzersitzungen, Benutzer und Seats bereit [90 (nicht zugänglich), 253 (nicht zugänglich), 254 (nicht zugänglich)]. MCP-Komponenten im NovaDE-Projekt können diese Schnittstelle nutzen, um kontextbezogene Informationen über den aktuellen Benutzer oder die Sitzung zu erhalten oder um sitzungsbezogene Aktionen auszulösen.

- **Manager-Interface (`org.freedesktop.login1.Manager` auf `/org/freedesktop/login1`):** 90
    - **Methoden:**
        - `GetSession(IN String session_id, OUT ObjectPath object_path)`
        - `GetUser(IN UInt32 uid, OUT ObjectPath object_path)`
        - `ListSessions(OUT Array<(String session_id, UInt32 user_id, String user_name, String seat_id, ObjectPath session_path)> sessions)`
        - `LockSession(IN String session_id)`
        - `UnlockSession(IN String session_id)`
    - **Signale:**
        - `SessionNew(String session_id, ObjectPath object_path)` 91
        - `SessionRemoved(String session_id, ObjectPath object_path)` 91
        - `PrepareForSleep(Boolean start)` 90
- **Session-Interface (`org.freedesktop.login1.Session` auf dem jeweiligen Session-Pfad):**
    - **Signale:**
        - `Lock()` [90 (nicht zugänglich)]
        - `Unlock()` [90 (nicht zugänglich)]
- **Nutzung im NovaDE-Projekt für MCP-Kontext:**
    - Abfrage der aktuellen Sitzungs-ID oder Benutzer-ID, um MCP-Aktionen zu personalisieren oder Berechtigungen feingranularer zu steuern.
    - Reaktion auf `PrepareForSleep`-Signale, um z.B. laufende MCP-Operationen zu pausieren oder Daten zu sichern.
    - Auslösen von `LockSession` durch eine MCP-Methode, um den Bildschirm zu sperren.

### 6.6. `org.freedesktop.UPower` – Energieverwaltung

UPower liefert Informationen über Energiequellen und deren Status.92 Dies kann für MCP-Komponenten relevant sein, die ihr Verhalten an den Energiestatus anpassen müssen.

- **UPower-Interface (`org.freedesktop.UPower` auf `/org/freedesktop/UPower`):** 93
    - **Methoden:**
        - `EnumerateDevices(OUT Array<ObjectPath> devices)`
        - `GetDisplayDevice(OUT ObjectPath device)`
        - `GetCriticalAction(OUT String action)`
    - **Signale:**
        - `DeviceAdded(ObjectPath device)` [93 (nicht zugänglich)]
        - `DeviceRemoved(ObjectPath device)` [93 (nicht zugänglich)]
        - `DeviceChanged(ObjectPath device)` (impliziert durch `PropertiesChanged` auf Device-Objekt)
    - **Properties:**
        - `DaemonVersion` (String, RO)
        - `OnBattery` (Boolean, RO)
        - `LidIsClosed` (Boolean, RO)
        - `LidIsPresent` (Boolean, RO)
- **Device-Interface (`org.freedesktop.UPower.Device` auf dem jeweiligen Gerätepfad):** 93
    - **Properties (Auswahl):**
        - `Type` (UInt32, z.B. Battery, UPS, LinePower)
        - `State` (UInt32, z.B. Charging, Discharging, FullyCharged)
        - `Percentage` (Double)
        - `TimeToEmpty` (Int64, Sekunden)
        - `TimeToFull` (Int64, Sekunden)
        - `IsPresent` (Boolean)
        - `IconName` (String)
        - `WarningLevel` (UInt32)
    - **Signale:**
        - `PropertiesChanged` (via `org.freedesktop.DBus.Properties`) [257 (nicht zugänglich)]
- **Nutzung im NovaDE-Projekt für MCP-Kontext:**
    - MCP-Werkzeuge könnten den Batteriestatus abfragen (`OnBattery`, `Percentage`, `TimeToEmpty`), um langlaufende Operationen zu vermeiden oder den Benutzer zu warnen.
    - Anpassung des Verhaltens von NovaDE-Komponenten basierend auf dem Energiestatus (z.B. Reduzierung der Hintergrundaktivität bei niedrigem Akkustand).

### 6.7. `org.freedesktop.Notifications` – Desktop-Benachrichtigungen

Diese Schnittstelle ermöglicht es Anwendungen, Desktop-Benachrichtigungen anzuzeigen.95 MCP-Komponenten im NovaDE-Projekt können dies nutzen, um Benutzer über wichtige Ereignisse, den Abschluss von Aufgaben oder Fehler zu informieren.

- **Schnittstellenspezifikation (`org.freedesktop.Notifications` auf `/org/freedesktop/Notifications`):** 96
    - **Methoden:**
        - `Notify(IN String app_name, IN UInt32 replaces_id, IN String app_icon, IN String summary, IN String body, IN Array<String> actions, IN Dict<String,Variant> hints, IN Int32 expire_timeout, OUT UInt32 notification_id)`
        - `CloseNotification(IN UInt32 id)`
        - `GetCapabilities(OUT Array<String> capabilities)`
        - `GetServerInformation(OUT String name, OUT String vendor, OUT String version, OUT String spec_version)`
    - **Signale:**
        - `NotificationClosed(UInt32 id, UInt32 reason)`
        - `ActionInvoked(UInt32 id, String action_key)`
- **Wichtige Parameter und Hinweise:**
    - `actions`: Liste von Aktions-IDs und deren lesbaren Bezeichnungen.
    - `hints`: Diktionär für zusätzliche Hinweise (z.B. `urgency`, `sound-file`, `image-data`).
    - `expire_timeout`: `-1` für Server-Default, `0` für niemals auslaufend.
- **Nutzung im NovaDE-Projekt durch MCP:**
    - Ein MCP-Tool, das eine langlaufende Aufgabe abschließt, kann `Notify` aufrufen, um den Benutzer zu informieren.
    - Fehler, die in MCP-Operationen auftreten und Benutzereingriffe erfordern, können als Benachrichtigungen signalisiert werden.
    - Aktionen in Benachrichtigungen (`actions`-Parameter) können mit spezifischen MCP-Folgeaktionen im NovaDE-Client verknüpft werden.

## 7. Interaktion mit Wayland (Smithay)

Falls das NovaDE-Projekt einen eigenen Wayland-Compositor beinhaltet oder tief mit einem solchen interagiert (z.B. für spezifische Desktop-Umgebungsfeatures), kommt das Smithay-Framework zum Einsatz.10 Smithay ist eine Rust-Bibliothek zum Erstellen von Wayland-Compositoren.

### 7.1. Smithay Architekturüberblick

Smithay bietet Bausteine für Wayland-Compositoren und ist modular aufgebaut.10

- **Display und EventLoop:** Das Herzstück ist der `Display`-Typ (aus `wayland-server`) und ein `calloop::EventLoop`.98 `DisplayHandle` wird für Interaktionen mit dem Wayland-Protokoll verwendet [214 (nicht zugänglich)]. Der `EventLoopHandle` von `calloop` dient zur Verwaltung von Event-Quellen.234
- **State Management:** Ein zentraler `State`-Typ (z.B. `AnvilState` im Smithay-Beispielcompositor Anvil) hält den Zustand des Compositors [258 (nicht zugänglich), 124 (nicht zugänglich), 124 (nicht zugänglich), 98 (nicht zugänglich), 261 (nicht zugänglich), 262 (nicht zugänglich), 170 (nicht zugänglich)]. `ClientData` (oder `UserDataMap` auf Ressourcen) wird verwendet, um client-spezifischen Zustand zu speichern [98 (nicht zugänglich)].
- **Handler und Delegation:** Für verschiedene Wayland-Protokolle und -Objekte implementiert der `State`-Typ spezifische Handler-Traits (z.B. `CompositorHandler`, `ShmHandler`, `OutputHandler`, `SeatHandler`, `DataDeviceHandler`, `XdgShellHandler`, etc.). Smithay verwendet `delegate_<protocol>!` Makros, um die Dispatch-Logik zu vereinfachen [98 (nicht zugänglich), 136 (nicht zugänglich), 201 (nicht zugänglich), 205 (nicht zugänglich), 200 (nicht zugänglich), 200 (nicht zugänglich), 145 (nicht zugänglich), 222 (nicht zugänglich), 222 (nicht zugänglich), 200 (nicht zugänglich)].

### 7.2. Wayland Core Protokolle und ihre Handhabung durch MCP

#### 7.2.1. `wl_compositor`, `wl_subcompositor`, `wl_surface`, `wl_buffer`

Diese sind grundlegend für jede Wayland-Anzeige.

- **`CompositorState` und `CompositorHandler`:** Smithay stellt `CompositorState` zur Verwaltung von `wl_surface`-Objekten und deren Hierarchien (Subsurfaces) bereit.235 Der `CompositorHandler` im NovaDE-State muss implementiert werden, um auf Surface-Commits und -Zerstörungen zu reagieren.134 `SurfaceData` [263 (nicht zugänglich)] und `CompositorClientState` [201 (nicht zugänglich)] speichern oberflächen- bzw. clientbezogene Zustände. `SurfaceAttributes` enthalten Informationen wie die zugewiesene Rolle [123 (nicht zugänglich)].
- **`wl_buffer`:** Repräsentiert den Inhalt einer Surface. `BufferHandler` [145 (nicht zugänglich)] wird implementiert, um auf die Zerstörung von Buffern zu reagieren.
- **MCP-Interaktion:** MCP-Komponenten könnten indirekt mit diesen Objekten interagieren, z.B. wenn eine MCP-gesteuerte Anwendung eine UI auf dem Desktop darstellt. Die Spezifikation von Fenstergeometrien oder das Anfordern von Neuzeichnungen könnte über MCP-Methoden erfolgen, die dann auf die entsprechenden `wl_surface`-Operationen abgebildet werden.

#### 7.2.2. `wl_shm` – Shared Memory Buffers

Ermöglicht Clients, Buffer über Shared Memory bereitzustellen.

- **`ShmState` und `ShmHandler`:** `ShmState` verwaltet den `wl_shm`-Global und die unterstützten Formate. Der `ShmHandler` im NovaDE-State stellt den Zugriff auf `ShmState` sicher.136
- **Buffer-Import und Rendering:** `with_buffer_contents` erlaubt den Zugriff auf SHM-Buffer-Daten.145 Renderer wie `GlesRenderer` können SHM-Buffer importieren (`import_shm_buffer`) und rendern.171 MCP-Aktionen, die die Anzeige von Inhalten erfordern, die von einem Client als SHM-Buffer bereitgestellt werden, nutzen diese Mechanismen.

#### 7.2.3. `wl_output` und `xdg-output` – Output Management

Verwaltung von Bildschirmausgaben.

- **`Output` und `OutputHandler`:** Ein `Output`-Objekt repräsentiert eine physische Anzeige. `Output::new()` erstellt ein Output-Objekt, `Output::create_global()` macht es für Clients sichtbar [137 (nicht zugänglich), 137]. `Output::change_current_state()` aktualisiert Modus, Transformation, Skalierung und Position. Der `OutputHandler` im NovaDE-State behandelt clientseitige Interaktionen.101
- **`OutputManagerState`:** Kann verwendet werden, um `xdg-output` zusätzlich zu `wl_output` zu verwalten [137 (nicht zugänglich)].
- **MCP-Interaktion:** MCP-Methoden könnten es ermöglichen, Informationen über verfügbare Ausgaben abzurufen oder anwendungsspezifische Fenster auf bestimmten Ausgaben zu positionieren, basierend auf den von diesen Modulen bereitgestellten Informationen.

#### 7.2.4. `wl_seat`, `wl_keyboard`, `wl_pointer`, `wl_touch` – Input Handling

Verwaltung von Eingabegeräten und Fokus.

- **`SeatState` und `SeatHandler`:** `SeatState` verwaltet einen oder mehrere `Seat`-Instanzen. Der `SeatHandler` im NovaDE-State definiert, wie auf Eingabeereignisse und Fokusänderungen reagiert wird.113
- **Fokus-Management:** `KeyboardFocus`, `PointerFocus`, `TouchFocus` werden typischerweise auf `WlSurface` gesetzt, um den Eingabefokus zu lenken.113
- **Input Grabs:** Mechanismen wie `PointerGrab` und `KeyboardGrab` ermöglichen es, Eingabeereignisse exklusiv für eine bestimmte Oberfläche oder Aktion abzufangen [187 (nicht zugänglich)].
- **MCP-Interaktion:** MCP-gesteuerte Aktionen könnten den Fokus anfordern oder auf Eingabeereignisse reagieren, die über diese Seat-Abstraktionen verarbeitet werden. Beispielsweise könnte ein MCP-Tool eine Texteingabe erfordern, was das Setzen des Tastaturfokus auf ein entsprechendes Eingabefeld des MCP-Clients zur Folge hätte.


---

# Ultra-Feinspezifikation der MCP-Schnittstellen und Implementierungen für das NovaDE-Projekt

## 1. Einleitung

### 1.1. Zweck des Dokuments

Dieses Dokument definiert die Ultra-Feinspezifikation aller Model Context Protocol (MCP) Schnittstellen und deren Implementierungen innerhalb des NovaDE-Projekts. Es dient als maßgebliche technische Referenz für die Entwicklung, Integration und Wartung von MCP-basierten Komponenten im NovaDE-Ökosystem. Die Spezifikation umfasst detaillierte Beschreibungen von Nachrichtenformaten, Datenstrukturen, Methoden, Ereignissen und Fehlerbehandlungsmechanismen. Ein besonderer Fokus liegt auf der Integration der Domänenschicht-Spezifikation des NovaDE-Projekts in die MCP-Schnittstellen.

### 1.2. Geltungsbereich

Diese Spezifikation bezieht sich auf sämtliche Aspekte des Model Context Protocol, wie es im Kontext des NovaDE-Projekts eingesetzt wird. Dies beinhaltet:

- Alle MCP-Schnittstellen, die im NovaDE-Projekt definiert oder genutzt werden.
- Die Interaktion dieser MCP-Schnittstellen mit anderen Systemkomponenten, einschließlich, aber nicht beschränkt auf D-Bus-Dienste, Wayland-Protokolle und PipeWire-Audio-Management.
- Implementierungsrichtlinien und -details, insbesondere unter Verwendung der Programmiersprache Rust und assoziierter Bibliotheken.
- Die nahtlose Einbindung der fachlichen Anforderungen und Datenmodelle aus der Domänenschicht-Spezifikation des NovaDE-Projekts.

### 1.3. Zielgruppe

Dieses Dokument richtet sich an folgende Personengruppen innerhalb des NovaDE-Projekts:

- Softwarearchitekten und -entwickler, die MCP-Schnittstellen und -Komponenten entwerfen, implementieren oder nutzen.
- Systemintegratoren, die für die Bereitstellung und Konfiguration von NovaDE-Systemen verantwortlich sind.
- Qualitätssicherungsingenieure, die MCP-Funktionalitäten testen.
- Technische Projektmanager, die die Entwicklung und Implementierung des NovaDE-Projekts überwachen.

### 1.4. Definitionen und Akronyme

- **MCP:** Model Context Protocol. Ein offener Standard zur Kommunikation zwischen KI-Modellen/Anwendungen und externen Werkzeugen oder Datenquellen.1
- **NovaDE-Projekt:** Das spezifische Projekt, für das diese MCP-Spezifikation erstellt wird. (Details zum Projekt selbst sind außerhalb des Geltungsbereichs der bereitgestellten Materialien).
- **Domänenschicht-Spezifikation:** Ein separates Dokument, das die fachlichen Entitäten, Geschäftsregeln und Datenmodelle des NovaDE-Projekts beschreibt. Diese Spezifikation wird als integraler Bestandteil der MCP-Schnittstellendefinitionen betrachtet.
- **API:** Application Programming Interface.
- **D-Bus:** Desktop Bus, ein System für Interprozesskommunikation (IPC).3
- **Wayland:** Ein Kommunikationsprotokoll zwischen einem Display-Server (Compositor) und seinen Clients.4
- **PipeWire:** Ein Multimedia-Framework für Audio- und Videoverarbeitung unter Linux.5
- **XDG Desktop Portals:** Ein Framework, das sandboxed Anwendungen den sicheren Zugriff auf Ressourcen außerhalb der Sandbox ermöglicht.6
- **JSON-RPC:** JavaScript Object Notation Remote Procedure Call. Ein leichtgewichtiges RPC-Protokoll.8
- **Stdio:** Standard Input/Output.
- **SSE:** Server-Sent Events. Eine Technologie, die es einem Server ermöglicht, Updates an einen Client über eine HTTP-Verbindung zu pushen.8
- **Smithay:** Eine Rust-Bibliothek zur Erstellung von Wayland-Compositoren.10
- **zbus:** Eine Rust-Bibliothek für die D-Bus-Kommunikation.12
- **pipewire-rs:** Rust-Bindungen für PipeWire.14
- **mcpr:** Eine Rust-Implementierung des Model Context Protocol.16
- **mcp_client_rs:** Eine weitere Rust-Client-SDK für MCP.17

### 1.5. Referenzierte Dokumente

- Model Context Protocol Specification (Version 2025-03-26 oder aktueller) 2
- Domänenschicht-Spezifikation des NovaDE-Projekts (externes Dokument)
- Freedesktop D-Bus Specification 3
- Wayland Protocol Specification 4
- PipeWire Documentation 5
- XDG Desktop Portal Documentation 6
- Spezifikationen der relevanten D-Bus-Schnittstellen (Secrets, PolicyKit, Portals, Login1, UPower, Notifications)
- Spezifikationen der relevanten Wayland-Protokolle und -Erweiterungen
- Dokumentation der verwendeten Rust-Bibliotheken (Smithay, zbus, pipewire-rs, mcpr, mcp_client_rs, tokio, serde, thiserror etc.)

## 2. Model Context Protocol (MCP) – Grundlagen

### 2.1. Überblick und Kernkonzepte

Das Model Context Protocol (MCP) ist ein offener Standard, der darauf abzielt, die Integration von Large Language Models (LLMs) mit externen Werkzeugen, Datenbanken und APIs zu standardisieren.1 Es fungiert als eine universelle Schnittstelle, die es KI-Modellen ermöglicht, dynamisch auf Kontextinformationen zuzugreifen und Aktionen in ihrer Umgebung auszuführen.9 MCP adressiert die Herausforderung der Informationssilos und proprietären Integrationen, indem es einen einheitlichen Rahmen für die KI-Tool-Kommunikation schafft.1

Die Kernprinzipien von MCP umfassen 2:

- **Standardisierte Schnittstelle:** Einheitliche Methoden für LLMs zum Zugriff auf Werkzeuge und Ressourcen.
- **Erweiterte Fähigkeiten:** Befähigung von LLMs zur Interaktion mit diversen Systemen.
- **Sicherheit und Kontrolle:** Strukturierte Zugriffsmuster mit integrierter Validierung und klaren Grenzen.
- **Modularität und Erweiterbarkeit:** Einfaches Hinzufügen neuer Fähigkeiten durch Server, ohne die Kernanwendung des LLMs modifizieren zu müssen.

MCP ist darauf ausgelegt, die Reproduzierbarkeit von KI-Interaktionen zu verbessern, indem der gesamte notwendige Kontext (Datensätze, Umgebungsspezifikationen, Hyperparameter) an einem Ort verwaltet wird.1

### 2.2. Architektur (Client-Host-Server-Modell)

MCP basiert auf einer Client-Host-Server-Architektur 8:

- **Host:** Eine LLM-Anwendung (z.B. Claude Desktop, IDEs), die Verbindungen initiiert und als Container oder Koordinator für mehrere Client-Instanzen fungiert. Der Host verwaltet den Lebenszyklus, Sicherheitsrichtlinien (Berechtigungen, Benutzerautorisierung) und die Integration des LLMs.1
- **Client:** Eine Protokoll-Client-Komponente innerhalb der Host-Anwendung, die eine 1:1-Verbindung zu einem MCP-Server herstellt. Der Client ist verantwortlich für die Aushandlung von Fähigkeiten und die Orchestrierung von Nachrichten zwischen sich und dem Server.1
- **Server:** Ein Dienst (oft ein leichtgewichtiger Prozess), der spezifische Kontexte, Werkzeuge und Prompts für den Client bereitstellt. Server können lokale Prozesse oder entfernte Dienste sein und kapseln den Zugriff auf Datenquellen, APIs oder andere Utilities.1

Diese Architektur ermöglicht eine klare Trennung der Verantwortlichkeiten und fördert die Entwicklung modularer und wiederverwendbarer MCP-Server.23 Die Kommunikation zwischen diesen Komponenten erfolgt über eine Transportschicht und eine Protokollschicht, die auf JSON-RPC aufbaut und zustandsbehaftete Sitzungen für den Kontextaustausch und das Sampling betont.1

### 2.3. Nachrichtenformate (JSON-RPC 2.0 Basis)

MCP verwendet JSON-RPC 2.0 als Grundlage für seine Nachrichtenformate.8 Dies gewährleistet eine strukturierte und standardisierte Kommunikation. Die Hauptnachrichtentypen sind 8:

- **Requests (Anfragen):** Vom Client oder Server gesendete Nachrichten, die eine Antwort erwarten. Sie enthalten typischerweise eine `method` (Methodenname) und optionale `params` (Parameter).
    - Beispiel: `{"jsonrpc": "2.0", "method": "tools/list", "id": 1}`
- **Responses (Antworten):** Erfolgreiche Antworten auf Requests. Sie enthalten ein `result`-Feld mit den Ergebnisdaten und die `id` des ursprünglichen Requests.
    - Beispiel: `{"jsonrpc": "2.0", "result": {"tools": [...]}, "id": 1}`
- **Error Responses (Fehlerantworten):** Antworten, die anzeigen, dass ein Request fehlgeschlagen ist. Sie enthalten ein `error`-Objekt mit `code`, `message` und optional `data`, sowie die `id` des ursprünglichen Requests.
    - Beispiel: `{"jsonrpc": "2.0", "error": {"code": -32601, "message": "Method not found"}, "id": 1}`
- **Notifications (Benachrichtigungen):** Einwegnachrichten, die keine Antwort erwarten. Sie enthalten eine `method` und optionale `params`, aber keine `id`.
    - Beispiel: `{"jsonrpc": "2.0", "method": "textDocument/didChange", "params": {...}}`

Die spezifischen Methoden und Parameter für MCP-Nachrichten wie `initialize`, `tools/list`, `resources/read`, `tools/call` werden im weiteren Verlauf dieses Dokuments detailliert [2 (angenommen)].

### 2.4. Transportmechanismen

MCP unterstützt verschiedene Transportmechanismen für die Kommunikation zwischen Host/Client und Server 8:

- **Stdio (Standard Input/Output):** Dieser Mechanismus wird für die Kommunikation mit lokalen Prozessen verwendet. Der MCP-Server läuft als separater Prozess, und die Kommunikation erfolgt über dessen Standard-Eingabe- und Ausgabe-Streams. Dies ist ideal für Kommandozeilenwerkzeuge und lokale Entwicklungsszenarien.16 Die Rust-Bibliothek `mcpr` bietet beispielsweise `StdioTransport` 16, und `mcp_client_rs` fokussiert sich ebenfalls auf diesen Transportweg für lokal gespawnte Server.18
- **HTTP mit SSE (Server-Sent Events):** Dieser Mechanismus wird für netzwerkbasierte Kommunikation verwendet, insbesondere wenn der Server remote ist oder Echtzeit-Updates vom Server an den Client erforderlich sind. SSE ermöglicht es dem Server, asynchron Nachrichten an den Client zu pushen, während Client-zu-Server-Nachrichten typischerweise über HTTP POST gesendet werden.8 Die `mcpr`-Bibliothek erwähnt SSE-Transportunterstützung.16

Die Wahl des Transportmechanismus hängt von den spezifischen Anforderungen der NovaDE-Komponente ab, insbesondere davon, ob der MCP-Server lokal oder remote betrieben wird.

### 2.5. Sicherheitsaspekte

Sicherheit und Datenschutz sind zentrale Aspekte des Model Context Protocol, da es potenziell den Zugriff auf sensible Daten und die Ausführung von Code ermöglicht.2 Die Spezifikation legt folgende Schlüsselprinzipien fest 2:

- **Benutzereinwilligung und -kontrolle:**
    - Benutzer müssen explizit allen Datenzugriffen und Operationen zustimmen und diese verstehen.
    - Benutzer müssen die Kontrolle darüber behalten, welche Daten geteilt und welche Aktionen ausgeführt werden.
    - Implementierungen sollten klare Benutzeroberflächen zur Überprüfung und Autorisierung von Aktivitäten bereitstellen.
- **Datenschutz:**
    - Hosts müssen die explizite Zustimmung des Benutzers einholen, bevor Benutzerdaten an Server weitergegeben werden.
    - Hosts dürfen Ressourcendaten nicht ohne Zustimmung des Benutzers an andere Stellen übertragen.
    - Benutzerdaten sollten durch geeignete Zugriffskontrollen geschützt werden.
- **Werkzeugsicherheit (Tool Safety):**
    - Werkzeuge repräsentieren die Ausführung von beliebigem Code und müssen mit entsprechender Vorsicht behandelt werden. Beschreibungen des Werkzeugverhaltens (z.B. Annotationen) sind als nicht vertrauenswürdig zu betrachten, es sei denn, sie stammen von einem vertrauenswürdigen Server.
    - Hosts müssen die explizite Zustimmung des Benutzers einholen, bevor ein Werkzeug aufgerufen wird.
    - Benutzer sollten verstehen, was jedes Werkzeug tut, bevor sie dessen Verwendung autorisieren.
- **LLM Sampling Controls:**
    - Benutzer müssen explizit allen LLM-Sampling-Anfragen zustimmen.
    - Benutzer sollten kontrollieren, ob Sampling überhaupt stattfindet, den tatsächlichen Prompt, der gesendet wird, und welche Ergebnisse der Server sehen kann.

Obwohl MCP diese Prinzipien nicht auf Protokollebene erzwingen kann, **SOLLTEN** Implementierer robuste Zustimmungs- und Autorisierungsflüsse entwickeln, Sicherheitsimplikationen klar dokumentieren, geeignete Zugriffskontrollen und Datenschutzmaßnahmen implementieren und bewährte Sicherheitspraktiken befolgen.2 Die Architektur mit MCP-Servern als Vermittler kann eine zusätzliche Sicherheitsebene bieten, indem der Zugriff auf Ressourcen kontrolliert und potenziell in einer Sandbox ausgeführt wird.19

## 3. MCP-Schnittstellen im NovaDE-Projekt – Allgemeine Spezifikation

### 3.1. Namenskonventionen und Versionierung

Für alle MCP-Schnittstellen, die im Rahmen des NovaDE-Projekts definiert werden, gelten folgende Namenskonventionen und Versionierungsrichtlinien:

- **Schnittstellennamen:** Schnittstellennamen folgen dem Muster `nova.<KomponentenName>.<Funktionsbereich>.<Version>`. Beispiel: `nova.workspace.fileAccess.v1`. Dies gewährleistet Eindeutigkeit und Klarheit über den Ursprung und Zweck der Schnittstelle.
- **Methodennamen:** Methodennamen verwenden camelCase, beginnend mit einem Kleinbuchstaben (z.B. `listResources`, `callTool`).
- **Parameternamen:** Parameternamen verwenden ebenfalls camelCase.
- **Versionierung:** Jede MCP-Schnittstelle wird explizit versioniert. Die Version wird als Teil des Schnittstellennamens geführt (z.B. `v1`, `v2`). Änderungen, die die Abwärtskompatibilität brechen, erfordern eine Erhöhung der Hauptversionsnummer. Abwärtskompatible Erweiterungen können zu einer Erhöhung einer Nebenversionsnummer führen, falls ein solches Schema zusätzlich eingeführt wird. Das NovaDE-Projekt hält sich an die im MCP-Standard definierte Protokollversion (z.B. `2025-03-26`).2 Die aktuell unterstützte MCP-Protokollversion ist im `mcp_client_rs` Crate als `LATEST_PROTOCOL_VERSION` und `SUPPORTED_PROTOCOL_VERSIONS` definiert.27

### 3.2. Standardnachrichtenflüsse

Die Kommunikation im NovaDE-Projekt über MCP folgt etablierten Nachrichtenflüssen, die auf dem JSON-RPC 2.0 Standard basieren.8

1. **Initialisierung (Connection Lifecycle):** 8
    - Der MCP-Client (innerhalb des NovaDE-Hosts) sendet eine `initialize`-Anfrage an den MCP-Server. Diese Anfrage enthält die vom Client unterstützte Protokollversion und dessen Fähigkeiten (Capabilities).
    - Der MCP-Server antwortet mit seiner Protokollversion und seinen Fähigkeiten.
    - Der Client bestätigt die erfolgreiche Initialisierung mit einer `initialized`-Notification.
    - Anschließend beginnt der reguläre Nachrichtenaustausch.
2. **Anfrage-Antwort (Request-Response):** 8
    - Der Client sendet eine Anfrage (z.B. `tools/list`, `resources/read`, `tools/call`) mit einer eindeutigen ID.
    - Der Server verarbeitet die Anfrage und sendet entweder eine Erfolgsantwort mit dem Ergebnis (`result`) und derselben ID oder eine Fehlerantwort (`error`) mit Fehlercode, Nachricht und derselben ID.
3. **Benachrichtigungen (Notifications):** 8
    - Client oder Server können einseitige Benachrichtigungen senden, die keine direkte Antwort erwarten. Diese haben keine ID. Ein Beispiel ist die `initialized`-Notification oder serverseitige Push-Events.
4. **Beendigung (Termination):** 8
    - Die Verbindung kann durch eine `shutdown`-Anfrage vom Client initiiert werden, gefolgt von einer `exit`-Notification. Alternativ kann die Verbindung durch Schließen des zugrundeliegenden Transportkanals beendet werden.

Die Rust-Bibliotheken `mcpr` und `mcp_client_rs` implementieren diese grundlegenden Nachrichtenflüsse.16 `mcp_client_rs` beispielsweise nutzt Tokio für asynchrone Operationen und stellt Methoden wie `initialize()`, `list_resources()`, `call_tool()` zur Verfügung, die diesen Flüssen folgen.18

### 3.3. Fehlerbehandlung und Fehlercodes

Eine robuste Fehlerbehandlung ist entscheidend für die Stabilität der MCP-Kommunikation im NovaDE-Projekt. MCP-Fehlerantworten folgen dem JSON-RPC 2.0 Standard 8 und enthalten ein `error`-Objekt mit den Feldern `code` (Integer), `message` (String) und optional `data` (beliebiger Typ).

**Standard-Fehlercodes (basierend auf JSON-RPC 2.0):**

- `-32700 Parse error`: Ungültiges JSON wurde empfangen.
- `-32600 Invalid Request`: Die JSON-Anfrage war nicht wohlgeformt.
- `-32601 Method not found`: Die angeforderte Methode existiert nicht oder ist nicht verfügbar.
- `-32602 Invalid params`: Ungültige Methodenparameter.
- `-32603 Internal error`: Interner JSON-RPC-Fehler.
- `-32000` bis `-32099 Server error`: Reserviert für implementierungsspezifische Serverfehler.

NovaDE-spezifische Fehlercodes:

Zusätzlich zu den Standard-JSON-RPC-Fehlercodes definiert das NovaDE-Projekt spezifische Fehlercodes im Bereich -32000 bis -32099 für anwendungsspezifische Fehler, die während der Verarbeitung von MCP-Anfragen auftreten können. Diese Fehlercodes werden pro Schnittstelle und Methode dokumentiert.

Fehlerbehandlung in Rust-Implementierungen:

In Rust-basierten MCP-Implementierungen für NovaDE wird die Verwendung von thiserror für Bibliotheksfehler und potenziell anyhow für Anwendungsfehler empfohlen, um eine klare und kontextreiche Fehlerbehandlung zu gewährleisten.29 Die mcp_client_rs Bibliothek stellt einen Error-Typ bereit, der verschiedene Fehlerquellen kapselt.27 Die Struktur ErrorResponse und das Enum ErrorCode [240 (angenommen)] sind Teil der Protokolldefinitionen zur strukturierten Fehlerkommunikation.

**Beispiel für eine Fehlerantwort:**

JSON

```
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "NovaDE Domain Error: Ressource nicht gefunden.",
    "data": {
      "resourceUri": "nova://domain/entity/123"
    }
  },
  "id": 123
}
```

### 3.4. Integration der Domänenschicht-Spezifikation

Die Domänenschicht-Spezifikation des NovaDE-Projekts ist ein zentrales Element, das die fachlichen Entitäten, Operationen und Geschäftsregeln definiert. Die MCP-Schnittstellen im NovaDE-Projekt müssen diese Domänenspezifikation nahtlos integrieren. Dies bedeutet:

- **Abbildung von Domänenentitäten:** Datenstrukturen innerhalb der MCP-Nachrichten (Parameter von Methoden, Rückgabewerte, Event-Payloads) müssen die Entitäten der Domänenschicht widerspiegeln oder direkt verwenden. Dies stellt sicher, dass die MCP-Kommunikation die fachlichen Anforderungen korrekt abbildet.
- **Domänenoperationen als MCP-Methoden:** Viele MCP-Methoden werden direkte Abbildungen von Operationen sein, die in der Domänenschicht definiert sind. Die Parameter und Rückgabewerte dieser MCP-Methoden korrespondieren mit den Ein- und Ausgaben der Domänenoperationen.
- **Validierung und Geschäftsregeln:** Bevor MCP-Anfragen an die Domänenschicht weitergeleitet oder Ergebnisse von der Domänenschicht über MCP zurückgegeben werden, müssen Validierungen und Geschäftsregeln der Domänenschicht angewendet werden. Dies kann sowohl im MCP-Server als auch in einer zwischengeschalteten Logikschicht geschehen.
- **Konsistente Terminologie:** Die in den MCP-Schnittstellen verwendete Terminologie (Namen von Methoden, Parametern, Datenfeldern) sollte mit der Terminologie der Domänenschicht-Spezifikation übereinstimmen, um Missverständnisse zu vermeiden und die Kohärenz im gesamten System zu fördern.

Die genauen Details der Integration hängen von den spezifischen Inhalten der Domänenschicht-Spezifikation ab. Jede detaillierte MCP-Schnittstellenspezifikation in Abschnitt 4 wird explizit auf die relevanten Teile der Domänenschicht-Spezifikation verweisen und die Abbildung erläutern.

## 4. Detaillierte MCP-Schnittstellenspezifikationen

Für das NovaDE-Projekt werden spezifische MCP-Schnittstellen definiert, um die Interaktion mit verschiedenen Modulen und Diensten zu ermöglichen. Jede Schnittstelle wird nach dem folgenden Schema spezifiziert. _Da die konkreten Schnittstellen für NovaDE nicht in den bereitgestellten Materialien definiert sind, dient der folgende Abschnitt als Vorlage und Beispielstruktur._

---

**Beispiel-Schnittstelle: `nova.dataAccess.document.v1`**

### 4.1. Beschreibung und Zweck

Die Schnittstelle `nova.dataAccess.document.v1` dient dem Zugriff auf und der Verwaltung von Dokumenten innerhalb des NovaDE-Projekts. Sie ermöglicht es MCP-Clients, Dokumente basierend auf Kriterien der Domänenschicht zu lesen, zu schreiben, zu aktualisieren und zu löschen. Diese Schnittstelle interagiert eng mit den Entitäten und Operationen, die in der "Domänenschicht-Spezifikation, Abschnitt X.Y (Dokumentenverwaltung)" definiert sind.

### 4.2. Methoden

#### 4.2.1. `readDocument`

- **Beschreibung:** Liest den Inhalt und die Metadaten eines spezifischen Dokuments.
- **Parameter:**
    - `uri` (String, erforderlich): Der eindeutige URI des Dokuments, konform zum NovaDE-URI-Schema (z.B. `nova://documents/internal/doc123`).
    - `options` (Object, optional): Zusätzliche Optionen für den Lesevorgang.
        - `version` (String, optional): Die spezifische Version des Dokuments, die gelesen werden soll. Falls nicht angegeben, wird die neueste Version gelesen.
- **Rückgabewerte:**
    - `document` (Object): Ein Objekt, das das gelesene Dokument repräsentiert. Die Struktur dieses Objekts ist in der Domänenschicht-Spezifikation definiert und könnte Felder wie `uri`, `mimeType`, `content` (String oder Binary), `metadata` (Object), `version` (String), `lastModified` (Timestamp) enthalten.
- **Mögliche Fehler:**
    - `-32001`: `DOCUMENT_NOT_FOUND` - Das angeforderte Dokument existiert nicht.
    - `-32002`: `ACCESS_DENIED` - Der Client hat keine Berechtigung, das Dokument zu lesen.
    - `-32003`: `VERSION_NOT_FOUND` - Die angeforderte Version des Dokuments existiert nicht.

#### 4.2.2. `writeDocument`

- **Beschreibung:** Schreibt ein neues Dokument oder aktualisiert ein bestehendes Dokument.
- **Parameter:**
    - `uri` (String, erforderlich): Der URI, unter dem das Dokument geschrieben werden soll. Bei Aktualisierung eines bestehenden Dokuments dessen URI.
    - `content` (String oder Binary, erforderlich): Der Inhalt des Dokuments. Der Typ (String oder Base64-kodiertes Binary) hängt vom `mimeType` ab.
    - `mimeType` (String, erforderlich): Der MIME-Typ des Dokuments (z.B. `text/plain`, `application/pdf`).
    - `metadata` (Object, optional): Domänenspezifische Metadaten für das Dokument.
    - `options` (Object, optional):
        - `overwrite` (Boolean, optional, default: `false`): Wenn `true` und ein Dokument unter dem URI existiert, wird es überschrieben. Andernfalls schlägt der Aufruf fehl, wenn das Dokument existiert.
- **Rückgabewerte:**
    - `newUri` (String): Der URI des geschriebenen oder aktualisierten Dokuments (kann sich bei Neuerstellung ändern, falls der Server URIs generiert).
    - `version` (String): Die Versionskennung des geschriebenen Dokuments.
- **Mögliche Fehler:**
    - `-32002`: `ACCESS_DENIED` - Keine Schreibberechtigung.
    - `-32004`: `DOCUMENT_EXISTS` - Dokument existiert bereits und `overwrite` ist `false`.
    - `-32005`: `INVALID_CONTENT` - Der bereitgestellte Inhalt ist für den `mimeType` ungültig.

_(Weitere Methoden wie `deleteDocument`, `listDocuments` würden hier analog spezifiziert werden.)_

### 4.3. Events/Notifications

#### 4.3.1. `documentChanged`

- **Beschreibung:** Wird vom Server gesendet, wenn ein Dokument, für das der Client möglicherweise Interesse bekundet hat (z.B. durch vorheriges Lesen), geändert wurde.
- **Parameter:**
    - `uri` (String): Der URI des geänderten Dokuments.
    - `changeType` (String): Art der Änderung (z.B. `UPDATED`, `DELETED`).
    - `newVersion` (String, optional): Die neue Versionskennung, falls `changeType` `UPDATED` ist.

### 4.4. Datenstrukturen

Die für diese Schnittstelle relevanten Datenstrukturen (z.B. die Struktur eines `Document`-Objekts, `Metadata`-Objekts) werden primär durch die Domänenschicht-Spezifikation des NovaDE-Projekts definiert. MCP-Nachrichten verwenden JSON-Repräsentationen dieser domänenspezifischen Strukturen.

**Beispiel `Document` (basierend auf einer hypothetischen Domänenspezifikation):**

JSON

```
{
  "uri": "nova://documents/internal/doc123",
  "mimeType": "text/plain",
  "content": "Dies ist der Inhalt des Dokuments.",
  "metadata": {
    "author": "NovaUser",
    "tags": ["wichtig", "projektA"],
    "customDomainField": "spezifischerWert"
  },
  "version": "v1.2.3",
  "lastModified": "2024-07-15T10:30:00Z"
}
```

### 4.5. Beispiele für Nachrichten

**Anfrage `readDocument`:**

JSON

```
{
  "jsonrpc": "2.0",
  "method": "nova.dataAccess.document.v1/readDocument",
  "params": {
    "uri": "nova://documents/internal/doc123"
  },
  "id": 1
}
```

**Antwort `readDocument` (Erfolg):**

JSON

```
{
  "jsonrpc": "2.0",
  "result": {
    "document": {
      "uri": "nova://documents/internal/doc123",
      "mimeType": "text/plain",
      "content": "Dies ist der Inhalt des Dokuments.",
      "metadata": {"author": "NovaUser"},
      "version": "v1.0.0",
      "lastModified": "2024-07-15T10:00:00Z"
    }
  },
  "id": 1
}
```

### 4.6. Interaktion mit der Domänenschicht

Die Methode `readDocument` ruft intern die Funktion `DomainLayer.getDocumentByUri(uri, options.version)` der Domänenschicht auf. Die zurückgegebenen Domänenobjekte werden gemäß den MCP-Datenstrukturen serialisiert. Die Methode `writeDocument` validiert die Eingaben anhand der Geschäftsregeln der Domänenschicht (z.B. `DomainLayer.validateDocumentContent(content, mimeType)`) und ruft dann `DomainLayer.saveDocument(documentData)` auf. Berechtigungsprüfungen erfolgen ebenfalls über dedizierte Domänenschicht-Services (z.B. `DomainLayer.Security.canReadDocument(userContext, uri)`).

---

_(Dieser beispielhafte Abschnitt würde für jede spezifische MCP-Schnittstelle im NovaDE-Projekt wiederholt werden.)_

## 5. Implementierung der MCP-Schnittstellen im NovaDE-Projekt

### 5.1. Verwendete Technologien

Die Kernimplementierung der MCP-Schnittstellen und der zugehörigen Logik im NovaDE-Projekt erfolgt in **Rust**. Dies schließt sowohl Client- als auch (potenzielle) Server-seitige Komponenten ein. Die Wahl von Rust begründet sich in dessen Stärken hinsichtlich Systemsicherheit, Performance und Nebenläufigkeit, welche für ein robustes Desktop Environment Projekt wie NovaDE essentiell sind.

Folgende Rust-Bibliotheken (Crates) sind für die MCP-Implementierung von zentraler Bedeutung:

- **MCP-Protokoll-Handling:**
    - `mcp_client_rs` (von darinkishore) [17 (angenommen), 241 (angenommen), 28 (angenommen), 243 (angenommen), 244 (angenommen), 243 (angenommen), 242 (angenommen), 245 (angenommen), 246 (angenommen), 246 (angenommen)] oder alternativ `mcpr` (von conikeec) 16 für die Client-seitige Implementierung. Die Entscheidung für eine spezifische Bibliothek hängt von den detaillierten Anforderungen und der Reife der jeweiligen Bibliothek zum Zeitpunkt der Implementierung ab. Beide bieten Mechanismen zur Serialisierung/Deserialisierung von MCP-Nachrichten und zur Verwaltung der Kommunikation.
- **Asynchrone Laufzeitumgebung:** `tokio` wird als primäre asynchrone Laufzeitumgebung für die nebenläufige Verarbeitung von MCP-Nachrichten und Interaktionen mit anderen Systemdiensten verwendet.25
- **Serialisierung/Deserialisierung:** `serde` und `serde_json` für die Umwandlung von Rust-Datenstrukturen in und aus dem JSON-Format, das von JSON-RPC verwendet wird.25
- **Fehlerbehandlung:** `thiserror` für die Definition von benutzerdefinierten Fehlertypen in Bibliotheks-Code und potenziell `anyhow` für eine vereinfachte Fehlerbehandlung in Anwendungscode.29
- **UUID-Generierung:** Das `uuid` Crate mit den Features `v4` und `serde` wird für die Erzeugung und Handhabung von eindeutigen Identifikatoren verwendet, die in MCP-Nachrichten oder domänenspezifischen Daten benötigt werden könnten.41
- **D-Bus-Kommunikation:** `zbus` für die Interaktion mit Systemdiensten über D-Bus.12
- **Wayland Compositing (falls NovaDE ein Compositor ist oder tief integriert):** `smithay` als Framework für Wayland-spezifische Interaktionen.10
- **PipeWire-Integration:** `pipewire-rs` für die Interaktion mit dem PipeWire Multimedia-Framework.14

### 5.2. MCP-Client-Implementierung (Rust)

Die MCP-Client-Komponenten im NovaDE-Projekt sind für die Kommunikation mit verschiedenen MCP-Servern zuständig, die Werkzeuge und Ressourcen bereitstellen.

#### 5.2.1. Initialisierung und Verbindungsaufbau

Die Initialisierung eines MCP-Clients beginnt mit der Konfiguration des Transports und der Erstellung einer Client-Instanz. Am Beispiel von `mcp_client_rs` (darinkishore):

- **Server-Spawning (für lokale Server via Stdio):** Die `ClientBuilder`-API ermöglicht das Starten eines lokalen MCP-Serverprozesses und die Verbindung zu dessen Stdio-Kanälen.17
    
    Rust
    
    ```
    // Beispielhafte Initialisierung (Pseudocode, da Servername und Argumente spezifisch für NovaDE sind)
    // use mcp_client_rs::client::ClientBuilder;
    // let client = ClientBuilder::new("nova-mcp-server-executable")
    //    .arg("--config-path")
    //    .arg("/etc/nova/mcp_server_config.json")
    //    .spawn_and_initialize().await?;
    ```
    
    Es ist wichtig zu beachten, dass `mcp_client_rs` (darinkishore) primär für lokal gespawnte Server konzipiert ist und keine direkte Unterstützung für Remote-Server plant.17 Für Remote-Verbindungen via HTTP/SSE müsste eine andere Bibliothek oder eine Erweiterung dieses Ansatzes in Betracht gezogen werden, wie sie z.B. in `mcpr` (conikeec) angedeutet ist.16
    
- **Verwendung eines existierenden Transports:** Alternativ kann ein Client mit einem bereits existierenden Transportobjekt initialisiert werden.14
    
    Rust
    
    ```
    // use std::sync::Arc;
    // use mcp_client_rs::client::Client;
    // use mcp_client_rs::transport::stdio::StdioTransport;
    // use tokio::io::{stdin, stdout};
    //
    // let transport = StdioTransport::with_streams(stdin(), stdout());
    // let client = Client::new(Arc::new(transport));
    ```
    
- **`initialize`-Nachricht:** Nach dem Aufbau der Transportverbindung sendet der Client eine `initialize`-Anfrage an den Server, um Protokollversionen und Fähigkeiten auszutauschen.8 Die `spawn_and_initialize()`-Methode von `mcp_client_rs` (darinkishore) handhabt dies implizit.17 Die `initialize()`-Methode auf der Client-Instanz von `mcpr` (conikeec) führt dies explizit durch.16
    

Die `InitializeParams` [240 (angenommen)] würden typischerweise die `protocolVersion` (z.B. "2025-03-26"), `clientName`, `clientVersion` und `supportedFeatures` enthalten. Die `InitializeResult` [240 (angenommen)] vom Server bestätigt die `protocolVersion` und listet die `serverCapabilities` und `serverInfo` auf.

#### 5.2.2. Senden von Requests

MCP-Clients im NovaDE-Projekt senden Anfragen an Server, um Ressourcen aufzulisten, Werkzeuge aufzurufen oder andere definierte Operationen auszuführen.

- **`ListResources`:**
    - Parameter: `ListResourcesParams` [240 (angenommen)] können Filterkriterien oder Paginierungsinformationen enthalten.
    - Antwort: `ListResourcesResult` [240 (angenommen)] enthält eine Liste von `Resource`-Objekten [240 (angenommen)], die jeweils URI, Name, Beschreibung und unterstützte Operationen definieren.
    - Beispielaufruf mit `mcp_client_rs`: `let resources = client.list_resources().await?;` 18
- **`CallTool`:**
    - Parameter: `CallToolParams` [240 (angenommen)] enthalten den `toolName` (String) und `arguments` (JSON-Objekt).
    - Antwort: `CallToolResult` [240 (angenommen)] enthält das Ergebnis der Werkzeugausführung, typischerweise als JSON-Objekt.
    - Beispielaufruf mit `mcp_client_rs`: `let tool_result = client.call_tool("domain.action.calculateSum", serde_json::json!({"op1": 10, "op2": 20})).await?;` 18
    - Die Definition von Werkzeugen (`ToolDefinition` [240 (angenommen)]) umfasst Name, Beschreibung und ein JSON-Schema für die Parameter.
- **`ReadResource`:** (und andere domänenspezifische Requests)
    - Parameter: Typischerweise ein URI und optionale Parameter.
    - Antwort: Der Inhalt oder Zustand der Ressource.
    - Beispielaufruf mit `mcp_client_rs`: `let read_result = client.read_resource("nova://domain/entity/123").await?;` 18

Alle diese Anfragen werden asynchron über den konfigurierten Transportmechanismus gesendet. Die `mcp_client_rs` Bibliothek nutzt Tokio für diese asynchronen Operationen.25

#### 5.2.3. Empfangen von Responses und Notifications

Der Empfang von Nachrichten ist ein kritischer Aspekt der MCP-Client-Implementierung.

- **Responses:** Antworten auf Client-Anfragen werden typischerweise über `async/await` Konstrukte direkt als Rückgabewerte der aufrufenden Methoden empfangen (z.B. `ListResourcesResult` von `list_resources().await?`).18 Die zugrundeliegende Transportlogik (z.B. in `StdioTransport` [242 (angenommen), 244 (angenommen), 242 (angenommen)]) liest die Rohdaten, parst sie als `McpMessage` [240 (angenommen)] und leitet sie an den entsprechenden wartenden Task weiter.
- **Notifications (Server Push Events):** Asynchrone Benachrichtigungen vom Server (z.B. `documentChanged` aus dem Beispiel in Abschnitt 4.3.1) erfordern einen dedizierten Mechanismus zum Empfang und zur Verarbeitung.
    - Die `mcpr` Bibliothek (conikeec) deutet auf Unterstützung für Server-Sent Events (SSE) hin, was einen Stream von Ereignissen impliziert, den der Client verarbeiten müsste.16
    - Die `mcp_client_rs` Bibliothek (darinkishore) ist primär auf Stdio ausgerichtet. Die Handhabung von Server-Push-Benachrichtigungen über Stdio würde erfordern, dass der `StdioTransport` kontinuierlich die Standardeingabe liest und eingehende Nachrichten (die keine direkten Antworten auf Anfragen sind) als `Notification` [240 (angenommen)] identifiziert und an einen Handler oder einen Ereignis-Stream weiterleitet. Die genaue Implementierung (z.B. ein dedizierter Empfangs-Loop oder ein Stream von `McpMessage`) ist in den bereitgestellten Snippets nicht vollständig ersichtlich [17 (fehlend), 246 (fehlend), 241 (fehlend), 243 (fehlend), 243 (fehlend), 245 (fehlend), 246 (fehlend), 246 (fehlend)]. Es ist davon auszugehen, dass eine `async_stream`-basierte Lösung oder ein `tokio::sync::broadcast` Kanal 36 verwendet wird, um diese Nachrichten an interessierte Teile der Anwendung zu verteilen.
    - Die `mcp_client_rs` Version 0.1.1 erwähnt "WebSocket Transport (Coming Soon)" mit "built-in reconnection handling", was auf zukünftige robustere Mechanismen für Server-Push und Verbindungsstatus hindeutet.25
- **Connection Status Events:** Die Überwachung des Verbindungsstatus (z.B. Verbindungsabbruch, Wiederverbindung) ist für robuste Anwendungen wichtig. Explizite Mechanismen hierfür sind in den Snippets zu `mcp_client_rs` (darinkishore) nicht detailliert, könnten aber Teil des `WebSocketTransport` sein 25 oder müssten auf der Transportebene (z.B. durch Überwachung der Stdio-Pipes) implementiert werden. Für SSE-Transporte könnten HTTP-Fehlercodes oder das Schließen des Event-Streams als Indikatoren dienen.26

#### 5.2.4. Fehlerbehandlung

Fehler können auf verschiedenen Ebenen auftreten: Transportfehler, JSON-RPC-Parsingfehler, oder anwendungsspezifische Fehler, die vom Server als `ErrorResponse` [240 (angenommen)] gesendet werden.

- Die `mcp_client_rs` Bibliothek verwendet `thiserror` zur Definition ihres `Error`-Typs, der verschiedene Fehlerquellen aggregiert.27
- Client-Code sollte `Result`-Typen sorgfältig behandeln, um auf Fehler angemessen reagieren zu können (z.B. Wiederholungsversuche, Benutzerbenachrichtigung, Logging).
- Spezifische `ErrorCode`-Werte [240 (angenommen)] in `ErrorResponse`-Nachrichten ermöglichen eine differenzierte Fehlerbehandlung basierend auf der Art des serverseitigen Fehlers.

#### 5.2.5. Transport Layer

- **StdioTransport:** Für die Kommunikation mit lokalen Serverprozessen. Implementierungen in `mcpr` 16 und `mcp_client_rs` [25 (angenommen), 244 (angenommen), 242 (angenommen)] lesen von `stdin` und schreiben nach `stdout` des Subprozesses. Die `StdioTransport` in `mcp_client_rs` verwendet typischerweise `tokio::io::AsyncRead` und `tokio::io::AsyncWrite` für die asynchrone Verarbeitung. Eingehende Nachrichten werden zeilenbasiert oder durch Längenpräfixe (gemäß JSON-RPC Framing) gelesen und dann als `McpMessage` deserialisiert.
- **SSETransport (Server-Sent Events):** Für webbasierte oder Remote-Server. `mcpr` erwähnt dessen Unterstützung.16 Dies involviert einen HTTP-Client, der eine Verbindung zu einem SSE-Endpunkt des Servers herstellt und einen kontinuierlichen Stream von Ereignissen empfängt.26

### 5.3. MCP-Server-Implementierung (Rust)

Obwohl der primäre Fokus des NovaDE-Projekts auf der Client-Seite liegen mag, könnten bestimmte Komponenten des Projekts auch als MCP-Server fungieren, um Fähigkeiten für andere Teile des Systems oder externe LLMs bereitzustellen.

- **Struktur:** Ein MCP-Server in Rust, beispielsweise unter Verwendung der `mcpr`-Bibliothek 16, würde eine `ServerConfig` definieren, die Name, Version und eine Liste der bereitgestellten `Tool`-Definitionen enthält. Jedes `Tool` spezifiziert seinen Namen, eine Beschreibung und ein JSON-Schema für seine Parameter.
- **Tool-Registrierung:** Für jedes definierte Werkzeug wird ein Handler registriert, der die Werkzeugparameter entgegennimmt, die Logik ausführt und ein Ergebnis (oder einen Fehler) zurückgibt.
    
    Rust
    
    ```
    // Beispielhafte Server-Konfiguration mit mcpr (conikeec)
    // use mcpr::{server::{Server, ServerConfig}, transport::stdio::StdioTransport, Tool, error::MCPError};
    // use serde_json::Value;
    //
    // let server_config = ServerConfig::new()
    //    .with_name("NovaDE.DomainService.v1")
    //    .with_version("1.0.0")
    //    .with_tool(Tool {
    //         name: "nova.domain.getEntityDetails".to_string(),
    //         description: Some("Ruft Details zu einer Domänenentität ab.".to_string()),
    //         parameters_schema: serde_json::json!({
    //             "type": "object",
    //             "properties": {
    //                 "entityUri": {"type": "string", "description": "URI der Entität"}
    //             },
    //             "required": ["entityUri"]
    //         }),
    //     });
    // let mut server: Server<StdioTransport> = Server::new(server_config);
    // server.register_tool_handler("nova.domain.getEntityDetails", |params: Value| {
    //     // Implementierung der Domänenlogik hier
    //     //...
    //     Ok(serde_json::json!({"status": "success", "data": { /*... */ }}))
    // })?;
    // let transport = StdioTransport::new();
    // server.start(transport)?;
    ```
    

Die Serverimplementierung ist verantwortlich für das Parsen eingehender Anfragen, das Weiterleiten an die entsprechenden Handler und das Senden von Antworten oder Benachrichtigungen über den gewählten Transportmechanismus.

### 5.4. Interaktion mit Systemdiensten und Protokollen

Die MCP-Schnittstellen im NovaDE-Projekt sind nicht isoliert, sondern interagieren intensiv mit bestehenden Systemdiensten und Protokollen. Diese Interaktionen sind entscheidend für den Zugriff auf Systemressourcen, die Verwaltung von Berechtigungen und die Integration in die Desktop-Umgebung. Die folgenden Abschnitte detaillieren diese Interaktionen.

## 6. Interaktion mit D-Bus-Diensten

Die Kommunikation mit systemweiten und benutzerspezifischen Diensten im NovaDE-Projekt erfolgt primär über D-Bus. Die Rust-Bibliothek `zbus` wird für diese Interaktionen verwendet.12

### 6.1. Allgemeine D-Bus-Integration mit `zbus`

`zbus` ermöglicht eine typsichere und asynchrone Kommunikation mit D-Bus-Diensten.

- **Proxy-Generierung:** Für die Interaktion mit D-Bus-Schnittstellen werden Proxys verwendet. Das `#[dbus_proxy]` (oder `#[proxy]`) Makro von `zbus` generiert Rust-Traits und Proxy-Strukturen aus D-Bus-Interface-Definitionen [12 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich), 62 (nicht zugänglich)].
    
    Rust
    
    ```
    // use zbus::{dbus_proxy, Connection, Result};
    // #
    // trait ExampleProxy {
    //     async fn some_method(&self, param: &str) -> Result<String>;
    //     #[dbus_proxy(signal)]
    //     async fn some_signal(&self, value: u32) -> Result<()>;
    // }
    ```
    
- **Verbindungsaufbau:** Eine Verbindung zum Session- oder Systembus wird mit `zbus::Connection::session().await?` bzw. `zbus::Connection::system().await?` hergestellt.45
- **Methodenaufrufe:** Methoden auf D-Bus-Schnittstellen werden asynchron über die generierten Proxy-Methoden aufgerufen.45
- **Signalempfang:** Signale werden als asynchrone Streams (`futures_util::stream::StreamExt`) über die Proxy-Methoden `receive_<signal_name>()` empfangen.46 Die Argumente des Signals können aus der `zbus::Message` deserialisiert werden [46 (nicht zugänglich), 65 (nicht zugänglich)].
- **Fehlerbehandlung:** `zbus`-Operationen geben `zbus::Result` zurück. Fehler werden durch das `zbus::Error`-Enum repräsentiert, das verschiedene Fehlerquellen wie I/O-Fehler, ungültige Nachrichten oder Fehler vom D-Bus-Dienst selbst abdeckt.52
- **Server-Implementierung:** `zbus` ermöglicht auch die Implementierung von D-Bus-Diensten. Mittels `ConnectionBuilder::serve_at` können Interfaces auf bestimmten Objektpfaden bereitgestellt werden [13 (nicht zugänglich), 57 (nicht zugänglich), 12 (nicht zugänglich), 56 (nicht zugänglich), 57 (nicht zugänglich), 12 (nicht zugänglich), 60]. Das `ObjectServer`-API kann für komplexere Szenarien mit mehreren Objekten und Interfaces auf derselben Verbindung genutzt werden [48 (nicht zugänglich), 57 (nicht zugänglich), 12 (nicht zugänglich), 56 (nicht zugänglich), 57 (nicht zugänglich), 58 (nicht zugänglich), 60].

### 6.2. `org.freedesktop.secrets` – Sichere Speicherung von Geheimnissen

Das NovaDE-Projekt nutzt die `org.freedesktop.Secrets`-Schnittstelle für die sichere Speicherung und Verwaltung von sensiblen Daten wie Passwörtern oder API-Tokens, die von MCP-Komponenten benötigt werden.70

- **Schnittstellenspezifikation:** 70
    - **`org.freedesktop.Secrets.Service`:** Verwalter von Collections und Sessions.
        - Methoden: `OpenSession`, `CreateCollection`, `SearchCollections`, `RetrieveSecrets`, `LockService`, `DeleteCollection`.
        - Signale: `CollectionCreated`, `CollectionDeleted`.
        - Properties: `Collections` (RO), `DefaultCollection` (RW).
    - **`org.freedesktop.Secrets.Collection`:** Eine Sammlung von Items (Geheimnissen).
        - Methoden: `Delete`, `SearchItems`, `CreateItem`.
        - Signale: `CreatedItem`, `DeletedItem`.
        - Properties: `Items` (RO), `Private` (RO), `Label` (RW), `Locked` (RO), `Created` (RO), `Modified` (RO).
    - **`org.freedesktop.Secrets.Item`:** Ein einzelnes Geheimnis mit Attributen.
        - Methoden: `Delete`.
        - Signale: `changed`.
        - Properties: `Locked` (RO), `Attributes` (RW), `Label` (RW), `Secret` (RW), `Created` (RO), `Modified` (RO).
    - **`org.freedesktop.Secrets.Session`:** Repräsentiert eine Sitzung zwischen Client und Dienst.
        - Methoden: `Close`, `Negotiate`, `BeginAuthenticate`, `CompleteAuthenticate`.
        - Signale: `Authenticated`.
- **Datentyp `Secret`:** 70
    - `algorithm` (String): Algorithmus zur Kodierung des Geheimnisses (z.B. "PLAIN").
    - `parameters` (Array<Byte>): Algorithmus-spezifische Parameter.
    - `value` (Array<Byte>): Der möglicherweise kodierte Geheimniswert.
- **Fehlerdomäne:** `org.freedesktop.Secrets.Error.*` (z.B. `AlreadyExists`, `IsLocked`, `NotSupported`).70
- **Nutzung im NovaDE-Projekt für MCP:**
    - MCP-Server oder -Clients im NovaDE-Projekt, die Zugriff auf persistente, sichere Anmeldeinformationen oder Tokens benötigen, interagieren mit diesem Dienst.
    - Beispiel: Ein MCP-Server, der eine Verbindung zu einem externen API herstellt, könnte das API-Token sicher über `org.freedesktop.Secrets` speichern und abrufen.
    - Die `CreateCollection`-Methode wird verwendet, um spezifische Sammlungen für NovaDE-Komponenten anzulegen, potenziell mit `private = true`, um den Zugriff einzuschränken.
    - `SearchItems` mit spezifischen `Attributes` wird verwendet, um gezielt nach Geheimnissen zu suchen.
    - Die `Negotiate`-Methode kann für eine verschlüsselte Übertragung der Geheimnisse genutzt werden, falls erforderlich.

### 6.3. `org.freedesktop.PolicyKit1.Authority` – Berechtigungsprüfung

PolicyKit wird im NovaDE-Projekt eingesetzt, um granulare Berechtigungsprüfungen für Aktionen durchzuführen, die von MCP-Schnittstellen oder den dahinterliegenden Diensten ausgelöst werden.71

- **Schnittstellenspezifikation:** `org.freedesktop.PolicyKit1.Authority` am Pfad `/org/freedesktop/PolicyKit1/Authority`.71
    - **Methoden:**
        - `CheckAuthorization(IN Subject subject, IN String action_id, IN Dict<String,String> details, IN CheckAuthorizationFlags flags, IN String cancellation_id, OUT AuthorizationResult result)`: Prüft, ob ein Subjekt berechtigt ist, eine Aktion auszuführen. `details` können verwendet werden, um kontextspezifische Informationen für die Autorisierungsentscheidung oder die dem Benutzer angezeigte Nachricht bereitzustellen. `AllowUserInteraction` im `flags`-Parameter ermöglicht eine interaktive Authentifizierung.
        - `CancelCheckAuthorization(IN String cancellation_id)`: Bricht eine laufende Autorisierungsprüfung ab.
        - `EnumerateActions(IN String locale, OUT Array<ActionDescription> action_descriptions)`: Listet alle registrierten Aktionen auf.
        - `RegisterAuthenticationAgent(IN Subject subject, IN String locale, IN String object_path)`: Registriert einen Authentifizierungsagenten.
    - **Signale:**
        - `Changed()`: Wird emittiert, wenn sich Aktionen oder Autorisierungen ändern.
- **Wichtige Datenstrukturen:** 71
    - `Subject`: Beschreibt das handelnde Subjekt (z.B. `unix-process`, `unix-session`).
    - `ActionDescription`: Beschreibt eine registrierte Aktion (ID, Beschreibung, Nachricht, Standardberechtigungen).
    - `AuthorizationResult`: Ergebnis der Prüfung (`is_authorized`, `is_challenge`, `details`).
- **Nutzung im NovaDE-Projekt für MCP-Zugriffssteuerung:**
    - Bevor eine MCP-Methode eine potenziell privilegierte Operation ausführt (z.B. Systemkonfiguration ändern, auf geschützte Benutzerdaten zugreifen), muss der MCP-Server oder die aufgerufene NovaDE-Komponente `CheckAuthorization` aufrufen.
    - Die `action_id` entspricht einer vordefinierten Aktion im NovaDE-PolicyKit-Regelwerk (z.B. `org.novade.mcp.filesystem.writeFile`).
    - Die `details` können MCP-spezifische Parameter enthalten, die für die Entscheidung oder die Benutzerinteraktion relevant sind.
    - Das Ergebnis von `CheckAuthorization` bestimmt, ob die MCP-Operation fortgesetzt oder mit einem Berechtigungsfehler abgelehnt wird.

### 6.4. XDG Desktop Portals

XDG Desktop Portals bieten sandboxed Anwendungen (und auch nicht-sandboxed Anwendungen) einen standardisierten Weg, um mit der Desktop-Umgebung zu interagieren, z.B. für Dateiauswahl oder Screenshots.6 MCP-Schnittstellen im NovaDE-Projekt können diese Portale nutzen, um solche Interaktionen sicher und konsistent zu gestalten.

#### 6.4.1. `org.freedesktop.portal.FileChooser`

Wird verwendet, um dem Benutzer Dialoge zur Datei- oder Ordnerauswahl anzuzeigen.6

- **Methoden:** 73
    - `OpenFile(IN String parent_window, IN String title, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Öffnet einen Dialog zur Auswahl einer oder mehrerer Dateien.
        - Optionen (`a{sv}`): `handle_token` (s), `accept_label` (s), `modal` (b), `multiple` (b), `directory` (b), `filters` (`a(sa(us))`), `current_filter` (`(sa(us))`), `choices` (`a(ssa(ss)s)`), `current_folder` (ay).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uris` (as), `choices` (`a(ss)`), `current_filter` (`(sa(us))`).
    - `SaveFile(IN String parent_window, IN String title, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Öffnet einen Dialog zum Speichern einer Datei.
        - Optionen (`a{sv}`): Ähnlich wie `OpenFile`, zusätzlich `current_name` (s), `current_file` (ay).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uris` (as), `choices` (`a(ss)`), `current_filter` (`(sa(us))`).
    - `SaveFiles(IN String parent_window, IN String title, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Öffnet einen Dialog zum Speichern mehrerer Dateien in einem Ordner.
        - Optionen (`a{sv}`): Ähnlich wie `SaveFile`, zusätzlich `files` (aay).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uris` (as), `choices` (`a(ss)`).
- **Nutzung im NovaDE-Projekt:** MCP-Methoden, die Dateiinteraktionen erfordern (z.B. das Hochladen eines Dokuments durch den Benutzer, das Speichern von generierten Inhalten), rufen diese Portalmethoden auf. Die `parent_window`-Kennung muss korrekt übergeben werden. Die `options` werden basierend auf dem Kontext der MCP-Aktion befüllt (z.B. Dateifilter basierend auf erwarteten MIME-Typen der Domänenschicht).

#### 6.4.2. `org.freedesktop.portal.Screenshot`

Ermöglicht das Erstellen von Screenshots und das Auswählen von Pixelfarben.3

- **Methoden (Version 2):** 79
    - `Screenshot(IN String parent_window, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Erstellt einen Screenshot.
        - Optionen (`a{sv}`): `handle_token` (s), `modal` (b, default: true), `interactive` (b, default: false, seit v2).
        - Antwort über `org.freedesktop.portal.Request::Response`: `uri` (s) des Screenshots.
    - `PickColor(IN String parent_window, IN Dict<String,Variant> options, OUT ObjectPath handle)`: Wählt die Farbe eines Pixels aus.
        - Optionen (`a{sv}`): `handle_token` (s).
        - Antwort über `org.freedesktop.portal.Request::Response`: `color` (`(ddd)`) als RGB-Werte .
- **Nutzung im NovaDE-Projekt:** MCP-Schnittstellen, die visuelle Informationen vom Desktop benötigen (z.B. ein Werkzeug zur Fehlerberichterstattung, das einen Screenshot anhängt, oder ein Design-Tool, das Farben vom Bildschirm aufnimmt), verwenden diese Portalmethoden.
- **Backend-Implementierung:** Für Wayland-basierte Desktops wie potenziell NovaDE ist eine Backend-Implementierung wie `xdg-desktop-portal-wlr` 6, `xdg-desktop-portal-gnome` 6, `xdg-desktop-portal-kde` 6 oder eine spezifische Implementierung wie `xdg-desktop-portal-luminous` (Rust-basiert, nutzt `libwayshot` und `zwlr_screencopy`) 83 erforderlich. `xdg-desktop-portal-luminous` ist ein Beispiel für eine Rust-basierte Implementierung, die `zbus` für D-Bus verwenden könnte und über das `zwlr_screencopy`-Protokoll mit wlroots-basierten Compositors interagiert.88

### 6.5. `org.freedesktop.login1` – Sitzungs- und Benutzerverwaltung

Der `systemd-logind`-Dienst stellt über D-Bus Informationen und Kontrollfunktionen für Benutzersitzungen, Benutzer und Seats bereit [90 (nicht zugänglich), 253 (nicht zugänglich), 254 (nicht zugänglich)]. MCP-Komponenten im NovaDE-Projekt können diese Schnittstelle nutzen, um kontextbezogene Informationen über den aktuellen Benutzer oder die Sitzung zu erhalten oder um sitzungsbezogene Aktionen auszulösen.

- **Manager-Interface (`org.freedesktop.login1.Manager` auf `/org/freedesktop/login1`):** 90
    - **Methoden:**
        - `GetSession(IN String session_id, OUT ObjectPath object_path)`
        - `GetUser(IN UInt32 uid, OUT ObjectPath object_path)`
        - `ListSessions(OUT Array<(String session_id, UInt32 user_id, String user_name, String seat_id, ObjectPath session_path)> sessions)`
        - `LockSession(IN String session_id)`
        - `UnlockSession(IN String session_id)`
    - **Signale:**
        - `SessionNew(String session_id, ObjectPath object_path)` 91
        - `SessionRemoved(String session_id, ObjectPath object_path)` 91
        - `PrepareForSleep(Boolean start)` 90
- **Session-Interface (`org.freedesktop.login1.Session` auf dem jeweiligen Session-Pfad):**
    - **Signale:**
        - `Lock()` [90 (nicht zugänglich)]
        - `Unlock()` [90 (nicht zugänglich)]
- **Nutzung im NovaDE-Projekt für MCP-Kontext:**
    - Abfrage der aktuellen Sitzungs-ID oder Benutzer-ID, um MCP-Aktionen zu personalisieren oder Berechtigungen feingranularer zu steuern.
    - Reaktion auf `PrepareForSleep`-Signale, um z.B. laufende MCP-Operationen zu pausieren oder Daten zu sichern.
    - Auslösen von `LockSession` durch eine MCP-Methode, um den Bildschirm zu sperren.

### 6.6. `org.freedesktop.UPower` – Energieverwaltung

UPower liefert Informationen über Energiequellen und deren Status.92 Dies kann für MCP-Komponenten relevant sein, die ihr Verhalten an den Energiestatus anpassen müssen.

- **UPower-Interface (`org.freedesktop.UPower` auf `/org/freedesktop/UPower`):** 93
    - **Methoden:**
        - `EnumerateDevices(OUT Array<ObjectPath> devices)`
        - `GetDisplayDevice(OUT ObjectPath device)`
        - `GetCriticalAction(OUT String action)`
    - **Signale:**
        - `DeviceAdded(ObjectPath device)` [93 (nicht zugänglich)]
        - `DeviceRemoved(ObjectPath device)` [93 (nicht zugänglich)]
        - `DeviceChanged(ObjectPath device)` (impliziert durch `PropertiesChanged` auf Device-Objekt)
    - **Properties:**
        - `DaemonVersion` (String, RO)
        - `OnBattery` (Boolean, RO)
        - `LidIsClosed` (Boolean, RO)
        - `LidIsPresent` (Boolean, RO)
- **Device-Interface (`org.freedesktop.UPower.Device` auf dem jeweiligen Gerätepfad):** 93
    - **Properties (Auswahl):**
        - `Type` (UInt32, z.B. Battery, UPS, LinePower)
        - `State` (UInt32, z.B. Charging, Discharging, FullyCharged)
        - `Percentage` (Double)
        - `TimeToEmpty` (Int64, Sekunden)
        - `TimeToFull` (Int64, Sekunden)
        - `IsPresent` (Boolean)
        - `IconName` (String)
        - `WarningLevel` (UInt32)
    - **Signale:**
        - `PropertiesChanged` (via `org.freedesktop.DBus.Properties`) [257 (nicht zugänglich)]
- **Nutzung im NovaDE-Projekt für MCP-Kontext:**
    - MCP-Werkzeuge könnten den Batteriestatus abfragen (`OnBattery`, `Percentage`, `TimeToEmpty`), um langlaufende Operationen zu vermeiden oder den Benutzer zu warnen.
    - Anpassung des Verhaltens von NovaDE-Komponenten basierend auf dem Energiestatus (z.B. Reduzierung der Hintergrundaktivität bei niedrigem Akkustand).

### 6.7. `org.freedesktop.Notifications` – Desktop-Benachrichtigungen

Diese Schnittstelle ermöglicht es Anwendungen, Desktop-Benachrichtigungen anzuzeigen.95 MCP-Komponenten im NovaDE-Projekt können dies nutzen, um Benutzer über wichtige Ereignisse, den Abschluss von Aufgaben oder Fehler zu informieren.

- **Schnittstellenspezifikation (`org.freedesktop.Notifications` auf `/org/freedesktop/Notifications`):** 96
    - **Methoden:**
        - `Notify(IN String app_name, IN UInt32 replaces_id, IN String app_icon, IN String summary, IN String body, IN Array<String> actions, IN Dict<String,Variant> hints, IN Int32 expire_timeout, OUT UInt32 notification_id)`
        - `CloseNotification(IN UInt32 id)`
        - `GetCapabilities(OUT Array<String> capabilities)`
        - `GetServerInformation(OUT String name, OUT String vendor, OUT String version, OUT String spec_version)`
    - **Signale:**
        - `NotificationClosed(UInt32 id, UInt32 reason)`
        - `ActionInvoked(UInt32 id, String action_key)`
- **Wichtige Parameter und Hinweise:**
    - `actions`: Liste von Aktions-IDs und deren lesbaren Bezeichnungen.
    - `hints`: Diktionär für zusätzliche Hinweise (z.B. `urgency`, `sound-file`, `image-data`).
    - `expire_timeout`: `-1` für Server-Default, `0` für niemals auslaufend.
- **Nutzung im NovaDE-Projekt durch MCP:**
    - Ein MCP-Tool, das eine langlaufende Aufgabe abschließt, kann `Notify` aufrufen, um den Benutzer zu informieren.
    - Fehler, die in MCP-Operationen auftreten und Benutzereingriffe erfordern, können als Benachrichtigungen signalisiert werden.
    - Aktionen in Benachrichtigungen (`actions`-Parameter) können mit spezifischen MCP-Folgeaktionen im NovaDE-Client verknüpft werden.

## 7. Interaktion mit Wayland (Smithay)

Falls das NovaDE-Projekt einen eigenen Wayland-Compositor beinhaltet oder tief mit einem solchen interagiert (z.B. für spezifische Desktop-Umgebungsfeatures), kommt das Smithay-Framework zum Einsatz.10 Smithay ist eine Rust-Bibliothek zum Erstellen von Wayland-Compositoren.

### 7.1. Smithay Architekturüberblick

Smithay bietet Bausteine für Wayland-Compositoren und ist modular aufgebaut.10

- **Display und EventLoop:** Das Herzstück ist der `Display`-Typ (aus `wayland-server`) und ein `calloop::EventLoop`.98 `DisplayHandle` wird für Interaktionen mit dem Wayland-Protokoll verwendet [214 (nicht zugänglich)]. Der `EventLoopHandle` von `calloop` dient zur Verwaltung von Event-Quellen.234
- **State Management:** Ein zentraler `State`-Typ (z.B. `AnvilState` im Smithay-Beispielcompositor Anvil) hält den Zustand des Compositors [258 (nicht zugänglich), 124 (nicht zugänglich), 124 (nicht zugänglich), 98 (nicht zugänglich), 261 (nicht zugänglich), 262 (nicht zugänglich), 170 (nicht zugänglich)]. `ClientData` (oder `UserDataMap` auf Ressourcen) wird verwendet, um client-spezifischen Zustand zu speichern [98 (nicht zugänglich)].
- **Handler und Delegation:** Für verschiedene Wayland-Protokolle und -Objekte implementiert der `State`-Typ spezifische Handler-Traits (z.B. `CompositorHandler`, `ShmHandler`, `OutputHandler`, `SeatHandler`, `DataDeviceHandler`, `XdgShellHandler`, etc.). Smithay verwendet `delegate_<protocol>!` Makros, um die Dispatch-Logik zu vereinfachen [98 (nicht zugänglich), 136 (nicht zugänglich), 201 (nicht zugänglich), 205 (nicht zugänglich), 200 (nicht zugänglich), 200 (nicht zugänglich), 145 (nicht zugänglich), 222 (nicht zugänglich), 222 (nicht zugänglich), 200 (nicht zugänglich)].

### 7.2. Wayland Core Protokolle und ihre Handhabung durch MCP

#### 7.2.1. `wl_compositor`, `wl_subcompositor`, `wl_surface`, `wl_buffer`

Diese sind grundlegend für jede Wayland-Anzeige.

- **`CompositorState` und `CompositorHandler`:** Smithay stellt `CompositorState` zur Verwaltung von `wl_surface`-Objekten und deren Hierarchien (Subsurfaces) bereit.235 Der `CompositorHandler` im NovaDE-State muss implementiert werden, um auf Surface-Commits und -Zerstörungen zu reagieren.134 `SurfaceData` [263 (nicht zugänglich)] und `CompositorClientState` [201 (nicht zugänglich)] speichern oberflächen- bzw. clientbezogene Zustände. `SurfaceAttributes` enthalten Informationen wie die zugewiesene Rolle [123 (nicht zugänglich)].
- **`wl_buffer`:** Repräsentiert den Inhalt einer Surface. `BufferHandler` [145 (nicht zugänglich)] wird implementiert, um auf die Zerstörung von Buffern zu reagieren.
- **MCP-Interaktion:** MCP-Komponenten könnten indirekt mit diesen Objekten interagieren, z.B. wenn eine MCP-gesteuerte Anwendung eine UI auf dem Desktop darstellt. Die Spezifikation von Fenstergeometrien oder das Anfordern von Neuzeichnungen könnte über MCP-Methoden erfolgen, die dann auf die entsprechenden `wl_surface`-Operationen abgebildet werden.

#### 7.2.2. `wl_shm` – Shared Memory Buffers

Ermöglicht Clients, Buffer über Shared Memory bereitzustellen.

- **`ShmState` und `ShmHandler`:** `ShmState` verwaltet den `wl_shm`-Global und die unterstützten Formate. Der `ShmHandler` im NovaDE-State stellt den Zugriff auf `ShmState` sicher.136
- **Buffer-Import und Rendering:** `with_buffer_contents` erlaubt den Zugriff auf SHM-Buffer-Daten.145 Renderer wie `GlesRenderer` können SHM-Buffer importieren (`import_shm_buffer`) und rendern.171 MCP-Aktionen, die die Anzeige von Inhalten erfordern, die von einem Client als SHM-Buffer bereitgestellt werden, nutzen diese Mechanismen.

#### 7.2.3. `wl_output` und `xdg-output` – Output Management

Verwaltung von Bildschirmausgaben.

- **`Output` und `OutputHandler`:** Ein `Output`-Objekt repräsentiert eine physische Anzeige. `Output::new()` erstellt ein Output-Objekt, `Output::create_global()` macht es für Clients sichtbar [137 (nicht zugänglich), 137]. `Output::change_current_state()` aktualisiert Modus, Transformation, Skalierung und Position. Der `OutputHandler` im NovaDE-State behandelt clientseitige Interaktionen.101
- **`OutputManagerState`:** Kann verwendet werden, um `xdg-output` zusätzlich zu `wl_output` zu verwalten [137 (nicht zugänglich)].
- **MCP-Interaktion:** MCP-Methoden könnten es ermöglichen, Informationen über verfügbare Ausgaben abzurufen oder anwendungsspezifische Fenster auf bestimmten Ausgaben zu positionieren, basierend auf den von diesen Modulen bereitgestellten Informationen.

#### 7.2.4. `wl_seat`, `wl_keyboard`, `wl_pointer`, `wl_touch` – Input Handling

Verwaltung von Eingabegeräten und Fokus.

- **`SeatState` und `SeatHandler`:** `SeatState` verwaltet einen oder mehrere `Seat`-Instanzen. Der `SeatHandler` im NovaDE-State definiert, wie auf Eingabeereignisse und Fokusänderungen reagiert wird.113
- **Fokus-Management:** `KeyboardFocus`, `PointerFocus`, `TouchFocus` werden typischerweise auf `WlSurface` gesetzt, um den Eingabefokus zu lenken.113
- **Input Grabs:** Mechanismen wie `PointerGrab` und `KeyboardGrab` ermöglichen es, Eingabeereignisse exklusiv für eine bestimmte Oberfläche oder Aktion abzufangen [187 (nicht zugänglich)].
- **MCP-Interaktion:** MCP-gesteuerte Aktionen könnten den Fokus anfordern oder auf Eingabeereignisse reagieren, die über diese Seat-Abstraktionen verarbeitet werden. Beispielsweise könnte ein MCP-Tool eine Texteingabe erfordern, was das Setzen des Tastaturfokus auf ein entsprechendes Eingabefeld des MCP-Clients zur Folge hätte.

#### 7.2.5. `wl_data_