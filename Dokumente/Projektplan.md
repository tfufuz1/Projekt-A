**Übergeordnetes Ziel:** Entwicklung einer modernen, performanten, intuitiven, anpassbaren und KI-unterstützten Benutzeroberfläche für NovaDE, basierend auf GTK4 und den `gtk4-rs` Bindings, die nahtlos mit den tieferliegenden Domänen- und Systemschichten interagiert.

---

**Teil 1: Grundlagen der UI-Schicht und Kern-Shell-Komponenten**

**I. Grundlagen der UI-Schicht (Wiederholung und Vertiefung)**

1. **Technologie-Stack (Verbindlich für UI):**
    - **GUI-Toolkit:** GTK4 mit `gtk4-rs` Bindings.
        - **Untersuchungsbedarf:**
            - Best Practices für die Strukturierung großer GTK4/Rust-Anwendungen (z.B. Aufteilung in wiederverwendbare Widgets, logische Module für UI-Bereiche).
            - State-Management-Patterns innerhalb von GTK4 (z.B. Model-View-ViewModel (MVVM)-ähnliche Ansätze, Komposition, effektive Nutzung von GTK-Properties und Bindings).
            - Fortgeschrittene Widget-Erstellung: Erben von GTK-Klassen, `#[derive(Boxed, Clone)]` für komplexe Daten in Signalen, `glib::ObjectSubclass` für benutzerdefinierte Widgets.
            - Effizienter Umgang mit dem GTK4-Event-Loop und Integration mit Rusts `async/await` (speziell `glib::MainContext::spawn_local`, `glib::clone!` Makro für sichere Callbacks).
            - Performance-Implikationen verschiedener GTK-Layout-Container und Zeichenoperationen.
    - **Theming:** Anwendung der von `domain::theming` generierten CSS-Stile über `Gtk::CssProvider`.
        - **Untersuchungsbedarf:**
            - Mechanismen zur dynamischen CSS-Aktualisierung zur Laufzeit bei Theme-Änderungen (Neuanwendung des `CssProvider`).
            - Effiziente Anwendung von Design-Tokens (als CSS Custom Properties) auf GTK-Widgets.
            - Performance-Implikationen komplexer CSS-Selektoren und -Regeln in GTK.
            - Erstellung benutzerdefinierter GTK-Widgets, die optimal auf dynamisches Theming und CSS-Styling reagieren (z.B. eigene CSS-Namen und -Eigenschaften definieren).
2. **Interaktion mit unteren Schichten (UI-Perspektive):**
    - Nutzung der APIs und Event-Systeme der Domänen- und Systemschicht.
        - **Untersuchungsbedarf:**
            - Patterns für die Datenkonvertierung und Kommunikation zwischen UI-Typen (GTK-spezifisch) und Domänen-/System-Typen (generisches Rust).
            - Fehlerbehandlung: Wie werden Fehler aus tieferen Schichten in der UI dargestellt (z.B. `Gtk::InfoBar`, `Gtk::AlertDialog`)? Wie werden technische Fehlermeldungen in benutzerfreundliche übersetzt?
            - Asynchrone Aufrufe an Domänen-/System-Services und Aktualisierung der UI bei Antwort.
3. **Allgemeine UI/UX-Prinzipien (Visionstreu):**
    - **Konsistenz:** Einheitliches Erscheinungsbild und Verhalten über alle UI-Komponenten hinweg.
    - **Feedback:** Visuelles Feedback auf Benutzeraktionen (z.B. Button-Pressed-Zustände, Ladeindikatoren).
    - **Effizienz:** Minimierung der Klicks/Aktionen für häufige Aufgaben.
    - **Zugänglichkeit (Accessibility):** Einhaltung von a11y-Standards (ATK/AT-SPI).
        - **Untersuchungsbedarf:** GTK4-spezifische Accessibility-Features (z.B. `AccessibleRole`, `AccessibleProperty`), Beschriftung von Widgets, Tastaturnavigation. Teststrategien für Accessibility mit GTK4/Rust.
    - **Performance:** Flüssige Animationen (GTK4 Animations API), schnelle Reaktionszeiten, geringer Ressourcenverbrauch der UI.
        - **Untersuchungsbedarf:** Profiling-Werkzeuge für GTK4/Rust-Anwendungen (z.B. GTK Inspector, `perf`), Optimierung von Rendering-Pfaden, Vermeidung von UI-Blockaden durch langlaufende Operationen (Delegation an Hintergrund-Threads).
    - **Anpassbarkeit:** Benutzerdefinierte Konfiguration von Layouts, Widgets und Verhalten (Details in spezifischen Komponenten).

**II. Kern-Shell-UI-Komponenten (`ui::shell` - Teil 1)**

Für jede Komponente: spezifische Verantwortlichkeiten, GTK-Widget-Struktur, Interaktionen mit anderen Modulen, visuelle Design-Aspekte (basierend auf der "Finale Beschreibung") und Implementierungsdetails definieren.

1. **`PanelWidget` (Kontroll-/Systemleiste(n)):**
    - **Design:**
        - Positionierung (oben/unten, konfigurierbar über `domain::global_settings_and_state_management`).
        - Höhe, Transparenzoptionen (ggf. basierend auf Theme-Tokens).
        - "Leuchtakzent"-Effekt (CSS-basiert).
    - **GTK-Struktur:** `Gtk::Box` oder `Gtk::CenterBox` als Hauptcontainer. Module als Kind-Widgets.
    - **Module (als einzelne, wiederverwendbare GTK-Widgets):**
        - **`AppMenuButton`:**
            - **GTK-Struktur:** `Gtk::MenuButton` oder `Gtk::Button` mit `Gtk::PopoverMenu`.
            - **Darstellung:** Zeigt ein globales Anwendungsmenü (falls vom aktiven Fenster unterstützt/exportiert, z.B. über `GActionGroup` des `Gtk::ApplicationWindow`).
            - **Interaktion (UI-Seite):** Reagiert auf Fokusänderungen der Systemschicht (`system::compositor` gibt aktive App-Info).
        - **`WorkspaceIndicatorWidget` (im Panel):**
            - **GTK-Struktur:** `Gtk::Box` mit `Gtk::Button` oder benutzerdefinierten kleinen Indikator-Widgets pro Workspace.
            - **Darstellung:** Kompakte visuelle Repräsentation der "Spaces". Hervorhebung des aktiven Space.
            - **Interaktion (UI-Seite):** Lauscht auf Events von `domain::workspaces::WorkspaceManager` (Liste und aktiver Space). Löst Workspace-Wechsel im Manager aus.
        - **`ClockDateTimeWidget`:**
            - **GTK-Struktur:** `Gtk::Label` (für Zeit/Datum), ggf. `Gtk::Button` mit `Gtk::Popover` für Kalender.
            - **Darstellung:** Anzeige von Uhrzeit und Datum. Optionales Kalender-Popup bei Klick.
            - **Konfiguration:** Datums-/Zeitformate (aus `domain::global_settings_and_state_management`).
        - **`SystemTrayEquivalentWidget`:**
            - **GTK-Struktur:** `Gtk::Box` oder `Gtk::FlowBox` für Icons.
            - **Darstellung:** Handhabung von Status-Icons (z.B. via `StatusNotifierItem` über D-Bus).
            - **Untersuchungsbedarf (UI-Fokus):** Implementierung eines `StatusNotifierWatcher` in Rust, der `StatusNotifierItem`-Proxies verwaltet und entsprechende `Gtk::Image` oder `Gtk::Button` Widgets dynamisch erzeugt/entfernt. Umgang mit Klick-Events und Menüs der Tray-Items.
        - **`QuickSettingsButtonWidget`:**
            - **GTK-Struktur:** `Gtk::Button` oder `Gtk::ToggleButton`.
            - **Darstellung:** Icon-Button.
            - **Funktionalität:** Öffnet/Schließt das `QuickSettingsPanelWidget`.
        - **`NotificationCenterButtonWidget`:**
            - **GTK-Struktur:** `Gtk::Button` oder `Gtk::ToggleButton`.
            - **Darstellung:** Icon-Button, zeigt eine Badge (z.B. `Gtk::Overlay` mit `Gtk::Label`) für neue Benachrichtigungen.
            - **Interaktion (UI-Seite):** Lauscht auf Events von `domain::user_centric_services::NotificationService` (neue/gelesene Benachrichtigungen). Öffnet/Schließt `NotificationCenterPanelWidget`.
    - **Layout:** Konfigurierbare Anordnung der Module im Panel (z.B. links, zentriert, rechts) über `Gtk::CenterBox` oder geschachtelte `Gtk::Box`-Container.

---

**Teil 2: Fortsetzung Kern-Shell-UI und Globale UI-Elemente**

**II. Kern-Shell-UI-Komponenten (`ui::shell` - Teil 2)**

1. **Panel-Module (Fortsetzung):**
    - **`NetworkIndicatorWidget`:**
        - **GTK-Struktur:** `Gtk::MenuButton` mit Icon. Popover-Menü (`Gtk::Box` mit ListBox für Verbindungen).
        - **Darstellung:** Anzeige des Netzwerkstatus (Icon, SSID). Menü für Verbindungsmanagement.
        - **Interaktion (UI-Seite):** Lauscht auf D-Bus-Signale des `NetworkManager` (über `system::dbus::networkmanager_client`) und aktualisiert Icon/Label. Löst Aktionen (Verbinden/Trennen) über die Systemschicht-API aus.
    - **`PowerIndicatorWidget`:**
        - **GTK-Struktur:** `Gtk::MenuButton` mit Icon.
        - **Darstellung:** Anzeige des Akkustatus/Ladezustands. Menü für Energieoptionen.
        - **Interaktion (UI-Seite):** Lauscht auf D-Bus-Signale von `UPower` (über `system::dbus::upower_client`).
    - **`AudioIndicatorWidget`:**
        - **GTK-Struktur:** `Gtk::MenuButton` mit Icon, `Gtk::Scale` für Lautstärke im Popover.
        - **Darstellung:** Anzeige der Lautstärke, Stummschaltungsstatus. Menü für Audioeinstellungen/-geräte.
        - **Interaktion (UI-Seite):** Lauscht auf Events/Properties von `PipeWire` (über `system::audio`). Sendet Befehle (Lautstärke ändern, Mute) an `system::audio`.
2. **`SmartTabBarWidget` (Intelligente Tab-Leiste pro "Space"):**
    - **Design:** Horizontale Leiste, abgerundete obere Ecken für Tabs. Hervorhebung des aktiven Tabs mit Akzentfarbe.
    - **GTK-Struktur:** `Gtk::Notebook` (ggf. stark angepasst oder komplett eigene Implementierung basierend auf `Gtk::Box` und benutzerdefinierten `ApplicationTabWidget`s).
    - **`ApplicationTabWidget` (benutzerdefiniertes Widget):**
        - **GTK-Struktur:** `Gtk::Box` mit `Gtk::Image` (für App-Icon) und `Gtk::Label` (für Fenstertitel).
        - **Darstellung:** Stellt einen einzelnen Tab dar. Reagiert auf Hover-Effekte, Schließen-Button.
    - **Funktionalität:**
        - Anzeige von `ApplicationTabWidget`s für "angepinnte" Apps/Split-Views (aus `domain::workspaces` Konfiguration).
        - Dynamische Anzeige von Tabs für weitere, nicht gepinnte Anwendungen im aktuellen "Space".
        - Überlauf-Logik für viele Tabs (z.B. Scrollbuttons, Dropdown-Liste).
        - Kontextmenü für Tabs (`Gtk::PopoverMenu`): "Schließen", "Zu anderem Space verschieben".
    - **Interaktion (UI-Seite):** Lauscht auf Events von `domain::workspaces::WorkspaceManager` (Fenster im aktuellen Space, Änderungen an gepinnten Apps). Holt Fensterinformationen (Icon, Titel) von `system::compositor` (oder einer Abstraktionsebene darüber). Löst Aktionen im `WorkspaceManager` oder `system::compositor` aus.
3. **`QuickSettingsPanelWidget` (Ausklappbares Schnelleinstellungs-Panel):**
    - **Design:** Ausklappbares Overlay/Menü (`Gtk::Popover` oder benutzerdefiniertes Widget), das vom `QuickSettingsButtonWidget` geöffnet wird.
    - **GTK-Struktur:** `Gtk::Box` mit verschiedenen Einstellungs-Widgets (z.B. `Gtk::Switch` für Dark Mode, `Gtk::Scale` für Lautstärke, `Gtk::ComboBoxText` für WLAN).
    - **Inhalt:** Schnellumschalter und -regler für WLAN, Bluetooth, Lautstärke, Dark Mode, etc.
    - **Interaktion (UI-Seite):** Liest und schreibt Werte über die entsprechenden Systemschicht-Schnittstellen (z.B. `system::dbus`, `system::audio`, `domain::theming::ThemingEngine` für Dark Mode).
4. **`WorkspaceSwitcherWidget` (Adaptive linke Seitenleiste):**
    - **Design:** Vertikale Leiste. Im eingeklappten Zustand nur `SpaceIconWidget`s. Aufklappbar (Mouse-Over/Geste) mit Namen/Vorschau der Spaces. Hervorhebung des aktiven Space.
    - **GTK-Struktur:** `Gtk::StackSidebar` oder `Gtk::ListBox` für die Icons/Namen. `Gtk::Revealer` für das Aufklappen.
    - **`SpaceIconWidget` (benutzerdefiniertes Widget):**
        - **GTK-Struktur:** `Gtk::Button` mit `Gtk::Image` (für App-Icon der gepinnten App oder benutzerdefiniertes Icon).
        - **Darstellung:** Visuelle Repräsentation eines einzelnen Space.
    - **Funktionalität:** Schnelle Navigation zwischen "Spaces". Drag & Drop von Fenstern auf Space-Icons zum Verschieben (im aufgeklappten Zustand mit Vorschau).
    - **Interaktion (UI-Seite):** Lauscht auf Events von `domain::workspaces::WorkspaceManager`. Löst Workspace-Wechsel und Fensterverschiebungen im Manager aus.
5. **`QuickActionDockWidget` (Konfigurierbares Schnellaktionsdock):**
    - **Design:** Schwebend oder am Bildschirmrand andockbar (konfigurierbar über `domain::global_settings_and_state_management`).
    - **GTK-Struktur:** `Gtk::Box` oder `Gtk::FlowBox`.
    - **`DockItemWidget` (benutzerdefiniertes Widget):**
        - **GTK-Struktur:** `Gtk::Button` mit `Gtk::Image` (App-Icon) und ggf. `Gtk::Label`.
        - **Darstellung:** Repräsentiert Apps, Dateien, Aktionen.
    - **Funktionalität:** Drag & Drop von Apps/Dateien ins Dock. Intelligente Vorschläge. Tastaturbedienung.
    - **Interaktion (UI-Seite):** Liest Konfiguration aus `domain::global_settings_and_state_management`. Startet Apps über die Systemschicht.

**III. Globale UI-Elemente und -Systeme**

1. **`NotificationCenterPanelWidget` (Benachrichtigungszentrum):**
    - **Design:** Ausklappbares Panel (`Gtk::Popover` oder eigenes Widget), typischerweise von der Systemleiste (`NotificationCenterButtonWidget`) erreichbar.
    - **GTK-Struktur:** `Gtk::ScrolledWindow` mit `Gtk::ListBox`.
    - **`NotificationWidget` (benutzerdefiniertes Widget):**
        - **GTK-Struktur:** `Gtk::Frame` oder `Gtk::Box` mit `Gtk::Image` (App-Icon), `Gtk::Label`s (App-Name, Zusammenfassung, Body), `Gtk::Box` für Aktionsbuttons.
        - **Darstellung:** Stellt eine einzelne Benachrichtigung dar. Layout anpassbar an Dringlichkeit/Aktionen.
    - **Funktionalität:** Anzeige einer Liste aktueller und historischer Benachrichtigungen. Möglichkeit zum Schließen/Interagieren mit Benachrichtigungen. "Bitte nicht stören"-Umschalter.
    - **Interaktion (UI-Seite):** Holt Benachrichtigungen von und sendet Aktionen an `domain::user_centric_services::NotificationService`. Lauscht auf dessen Events.
2. **`ui::theming_gtk` (Anwendung des Themes auf GTK-Widgets):**
    - **Verantwortlichkeiten:** Anwendung der von `domain::theming::ThemingEngine` generierten CSS-Stile auf alle GTK-Widgets.
    - **Kernfunktionen (UI-Seite):**
        - Abrufen des `AppliedThemeState` von der `ThemingEngine`.
        - Generieren einer CSS-Datei (String) aus den `resolved_tokens`.
        - Anwenden dieser CSS-Datei mittels `Gtk::CssProvider` auf den globalen `Gdk::Display` (`Gtk::StyleContext::add_provider_for_display`).
        - Abonnieren des `ThemeChangedEvent` von der `ThemingEngine`, um das CSS dynamisch zu aktualisieren.
    - **Untersuchungsbedarf (UI-Fokus):** Effiziente Generierung und Aktualisierung des CSS-Strings. Sicherstellung, dass benutzerdefinierte Widgets korrekt auf Theme-Änderungen reagieren und CSS-Klassen/Namen verwenden, die vom Theming-System beeinflusst werden können.
3. **`ui::components` (Sammlung wiederverwendbarer UI-Bausteine):**
    - **Verantwortlichkeiten:** Definition und Implementierung anwendungsspezifischer, aber wiederverwendbarer GTK4-Widgets, die das NovaDE-Erscheinungsbild und -Verhalten konsistent umsetzen.
    - **Beispiele (als `GtkWidget` Subklassen oder Kompositionen):**
        - **`StyledButtonWidget`:** Ein `Gtk::Button`, der konsistent mit dem NovaDE-Theming (Akzentfarben, Leuchteffekte, abgerundete Ecken) gestaltet ist und ggf. zusätzliche Style-Klassen für Varianten (z.B. "suggested-action", "destructive-action") bereitstellt.
        - **`ModalDialogWidget`:** Eine Subklasse von `Gtk::AlertDialog` oder eine benutzerdefinierte Komposition, die ein standardisiertes Aussehen und Verhalten für modale Dialoge (Header, Content-Bereich, Aktionsbuttons) sicherstellt.
        - **`SearchEntryWidget`:** Ein `Gtk::SearchEntry` mit konsistentem Design (z.B. abgerundete Ecken, integriertes Icon).
        - **`IconWidget`:** Ein Widget zur konsistenten Anzeige von Icons aus dem System-Theme mit Fallback-Logik und Unterstützung für symbolische Icons, das `Gtk::Image` kapselt und vereinfacht.
    - **Untersuchungsbedarf (UI-Fokus):** Identifikation weiterer wiederverwendbarer Komponenten während der Entwicklung der Haupt-UI-Module. Definition klarer APIs für diese Komponenten.

---

**Teil 3: Fenstermanagement-Frontend und Spezifische Ansichten**

**IV. `ui::window_manager_frontend` (UI-Aspekte des Fenstermanagements)**

Verantwortlichkeiten: Darstellung und Interaktion mit Fenstern, die von der Systemschicht (`system::compositor`) verwaltet werden. Dieses Modul stellt die visuellen Komponenten für das Fenstermanagement bereit.

1. **Client-Side Decorations (CSD) Logik und Darstellung:**
    - **Funktionalität (UI-Seite):**
        - Sicherstellung, dass von Anwendungen gezeichnete CSDs (z.B. via `Gtk::HeaderBar`) visuell mit dem globalen NovaDE-Theme harmonieren. Dies kann durch Setzen von globalen GTK-Einstellungen oder spezifischen CSS-Anweisungen für HeaderBars erreicht werden.
        - Anzeige von Fensteraktionen (Schließen, Maximieren, Minimieren) innerhalb der CSDs, falls die Anwendung diese nicht selbst bereitstellt oder falls eine konsistente Darstellung gewünscht ist (erfordert ggf. Interaktion mit `xdg-decoration`).
    - **Untersuchungsbedarf (UI-Fokus):** Wie kann das UI-Theming CSDs von GTK-Anwendungen (und ggf. Qt-Anwendungen via QGtkStyle) beeinflussen? Mechanismen zur Übernahme von Fensteraktionen, falls SSD bevorzugt wird.
2. **Server-Side Decorations (SSD) Rendering (falls vom `system::compositor` unterstützt und von der UI hier gezeichnet):**
    - **Funktionalität (UI-Seite, falls Rendering in der UI-Schicht stattfindet, was UNWAHRSCHEINLICH ist für SSDs – typischerweise macht das der Compositor):** Falls der Compositor nur die _Logik_ für SSDs bereitstellt, aber die _Darstellung_ an die Shell delegiert (seltenes Modell), müsste die UI hier Fensterrahmen und Titelleisten zeichnen.
    - **Design (UI-Seite):** Anpassbar und themenkonform. Verwendung von Theme-Tokens für Farben, Ränder, Schriftarten der Titelleiste.
    - **Interaktion (UI-Seite):** Eng mit `system::compositor` (für Fenstergeometrie, Zustand, Titel) und `domain::theming` (für Styling).
    - **Hinweis:** Typischerweise zeichnet der _Compositor_ SSDs. Die UI würde nur über `xdg-decoration` den Wunsch nach SSDs signalisieren oder deren Aussehen konfigurieren, falls das Protokoll dies zulässt.
3. **`OverviewModeWidget` (Fenster- und Workspace-Übersicht):**
    - **Design:**
        - Interaktive Kacheln mit Live-Vorschau (Thumbnails) der Fenster.
        - Hintergrund der Gesamtansicht abgedunkelt/unscharf, um den Fokus auf die Fensterkacheln zu legen.
        - Darstellung der Workspaces (z.B. als Miniaturen oder Leiste am Rand) für Drag & Drop von Fenstern.
    - **GTK-Struktur:** Benutzerdefiniertes Widget, das `Gtk::Fixed` oder einen Canvas-artigen Ansatz (`Gtk::DrawingArea` mit Cairo) für die flexible Anordnung und Skalierung der Fenster-Thumbnails verwendet. Animationen für Übergänge.
    - **Funktionalität (UI-Seite):**
        - Aktivierung durch Geste/Tastenkürzel (Signal von `system::input` weitergeleitet).
        - Fenster auswählen (führt zum Fokus des Fensters und Verlassen des Overview-Modus).
        - Fenster schließen (Button auf der Kachel).
        - Fenster per Drag & Drop zwischen Workspaces verschieben.
    - **Interaktion (UI-Seite):**
        - Holt Fensterliste und Thumbnails/Previews von `system::compositor` (z.B. über ein spezifisches Protokoll oder D-Bus-Schnittstelle, die der Compositor bereitstellt).
        - Interagiert mit `domain::workspaces::WorkspaceManager` für Workspace-Informationen und zum Auslösen von Fensterverschiebungen zwischen Workspaces.
4. **`AltTabSwitcherWidget` (Traditioneller Fensterwechsler):**
    - **Design:** Overlay-Anzeige (`Gtk::Window` vom Typ `Popup` oder `Tooltip`) mit Icons und Titeln der laufenden Anwendungen/Fenster für schnellen Wechsel.
    - **GTK-Struktur:** `Gtk::Box` mit `Gtk::ListBox` oder `Gtk::FlowBox` für die Anwendungs-Icons/-Labels.
    - **Funktionalität (UI-Seite):**
        - Aktivierung durch Alt+Tab (Signal von `system::input`).
        - Navigation durch die Liste mit Tab/Pfeiltasten.
        - Auswahl eines Fensters führt zum Fokus dieses Fensters und Schließen des Switchers.
    - **Interaktion (UI-Seite):** Holt Fensterliste und Fokusinformationen von `system::compositor`. Löst Fokuswechsel über `system::compositor` aus.

**V. Spezifische UI-Ansichten und -Dialoge**

1. **`ui::notifications_frontend` (Darstellung von Pop-up-Benachrichtigungen):**
    - **Verantwortlichkeiten:** Visuelle Präsentation von System- und Anwendungsbenachrichtigungen als Pop-ups.
    - **`NotificationPopupWidget` (benutzerdefiniertes Widget):**
        - **Design:** Dezent, im Dark Mode Stil mit Akzentfarbe je nach Dringlichkeit (aus `Notification.urgency`). Positionierung auf dem Bildschirm (z.B. obere rechte Ecke, konfigurierbar über `domain::global_settings_and_state_management`). Animationen für Erscheinen/Verschwinden.
        - **GTK-Struktur:** `Gtk::Window` (Typ `Popup`) oder ein Overlay-Layer. Enthält `Gtk::Image` (Icon), `Gtk::Label`s (App-Name, Zusammenfassung, Body), `Gtk::Box` für Aktionsbuttons.
        - **Funktionalität (UI-Seite):**
            - Anzeige von Icon, App-Name, Zusammenfassung, Body (optional), Aktionen (als `Gtk::Button`s).
            - Automatisches Schließen nach Timeout (konfigurierbar) oder manuell durch Klick auf Schließen-Button oder eine Aktion.
            - Stapeln mehrerer Pop-ups, falls sie schnell aufeinanderfolgen.
        - **Interaktion (UI-Seite):** Reagiert auf `NotificationPostedEvent` von `domain::user_centric_services::NotificationService`. Erstellt und zeigt ein `NotificationPopupWidget` an. Sendet Aktionen (Klick auf Aktionsbutton) zurück an den `NotificationService` (`invoke_action`).
2. **`ui::speed_dial` (Startansicht für leere Workspaces):**
    - **Verantwortlichkeiten:** Implementierung der GTK4-Oberfläche für die Speed-Dial-Funktionalität.
    - **Design:** Kachelbasierte Ansicht (`Gtk::FlowBox` oder `Gtk::GridView`). Jede Kachel (`Gtk::Button` mit `Gtk::Image` und `Gtk::Label`) repräsentiert einen Favoriten oder Vorschlag. Integrierte Suchfunktion (`Gtk::SearchEntry`). Anpassbares Layout (Größe/Anzahl der Kacheln).
    - **GTK-Struktur:** Hauptcontainer-Widget, das die Kacheln und das Suchfeld enthält.
    - **Interaktion (UI-Seite):**
        - Liest Favoriten und Konfiguration aus `domain::global_settings_and_state_management`.
        - Holt intelligente Vorschläge (kürzlich genutzte Apps/Dateien) von einer entsprechenden Domänen- oder Systemschicht-Komponente (ggf. `domain::user_centric_services::AIInteractionLogicService` oder eine dedizierte Verlaufs-Service).
        - Startet Anwendungen/öffnet Dateien über die Systemschicht.
3. **`ui::command_palette` (Kontextuelle Befehlspalette):**
    - **Verantwortlichkeiten:** GTK4-Implementierung der Befehlspaletten-UI.
    - **Design:** Overlay-Fenster (`Gtk::Window` Typ `Popup`, typischerweise zentriert oder am oberen Bildschirmrand), das bei Tastenkürzel (Super+Space, von `system::input` signalisiert) erscheint. `Gtk::Entry` für Texteingabe. `Gtk::ListBox` oder `Gtk::ListView` für dynamische Ergebnisliste mit Fuzzy-Matching.
    - **GTK-Struktur:** Eigenes `Gtk::Window` oder `Gtk::Popover`.
    - **Funktionalität (UI-Seite):**
        - Ausführung von Aktionen (repräsentiert als `GAction`s oder benutzerdefinierte Kommandos).
        - Starten von Apps.
        - Finden und direktes Ändern von Einstellungen.
        - Kontextsensitivität (zeigt relevante Befehle für aktive App/Desktop-Zustand).
    - **Interaktion (UI-Seite):**
        - Holt verfügbare Aktionen/Befehle von verschiedenen Quellen:
            - `domain::global_settings_and_state_management` (für Suche nach Einstellungen).
            - Systemschicht (für Starten von Apps, Ausführen von Systembefehlen).
            - Potenziell `domain::user_centric_services::AIInteractionLogicService` (für KI-gestützte Befehle oder Vorschläge).
            - Aktive Anwendung (falls sie Befehle über eine D-Bus-Schnittstelle oder `GAction`s exportiert).
        - Löst die ausgewählte Aktion aus.

---

**Teil 4: UI-Control-Center, Widgets-System und Übergreifende Aspekte**

**VI. `ui::control_center` (Modulare GTK4-Anwendung für Systemeinstellungen)**

1. **Verantwortlichkeiten:** Grafische Oberfläche zur Konfiguration aller Systemeinstellungen, die von `domain::global_settings_and_state_management` und anderen Domänen-/Systemdiensten verwaltet werden.
2. **Architektur (UI-Seite):**
    - Hauptfenster (`Gtk::ApplicationWindow`) mit einer Navigation (z.B. `Gtk::StackSidebar` oder `Gtk::ListBox` links) und einem Inhaltsbereich (`Gtk::Stack`), der das aktuell ausgewählte `SettingsModuleWidget` anzeigt.
    - Jede Einstellungskategorie (Erscheinungsbild, Netzwerk, Workspaces, etc.) ist ein eigenes, wiederverwendbares `SettingsModuleWidget` (benutzerdefiniertes GTK-Widget).
3. **Kernfunktionen (UI-Seite):**
    - **Navigation:** Klare Navigation zwischen den Einstellungsmodulen.
    - **Live-Vorschau:** Wo immer möglich, Änderungen an Einstellungen direkt in der UI des Control Centers oder auf dem Desktop (falls sicher) als Vorschau anzeigen.
    - **Suche:** Globale Suchfunktion (`Gtk::SearchEntry`) über alle Einstellungen hinweg (Interaktion mit `domain::global_settings_and_state_management`, um durchsuchbare Metadaten für Einstellungen zu erhalten).
    - **Zurücksetzen:** Buttons zum Zurücksetzen einzelner Einstellungen oder ganzer Module auf Standardwerte (löst Aktionen in `domain::global_settings_and_state_management` aus).
4. **Beispiel-Module (`SettingsModuleWidget` Implementierungen):**
    - **`AppearanceSettingsWidget`:**
        - **GTK-Struktur:** Enthält Widgets wie `Gtk::ComboBoxText` (für Theme-Auswahl), `Gtk::Switch` (für Dark Mode), Farbauswahl-Widget (für Akzentfarbe, z.B. `Gtk::ColorButton`), `Gtk::FontButton`, `Gtk::SpinButton` (für Skalierung).
        - **Interaktion (UI-Seite):** Liest und schreibt Einstellungen über `domain::theming::ThemingEngine` und `domain::global_settings_and_state_management::GlobalSettingsService`. Lauscht auf deren Events für dynamische Aktualisierungen.
    - **`NetworkSettingsWidget`:**
        - **GTK-Struktur:** `Gtk::ListBox` für verfügbare Netzwerke, Buttons für Hinzufügen/Bearbeiten, Detailansichten mit `Gtk::Entry` (Passwort), `Gtk::ComboBoxText` (Sicherheitstyp).
        - **Interaktion (UI-Seite):** Liest Netzwerkliste und -status von `system::dbus::networkmanager_client`. Löst Konfigurationsänderungen über dessen API aus.
    - **`WorkspaceSettingsWidget`:**
        - **GTK-Struktur:** `Gtk::Switch` (für dynamische Workspaces), `Gtk::SpinButton` (Standardanzahl), `Gtk::ComboBoxText` (Verhalten des Switchers).
        - **Interaktion (UI-Seite):** Liest und schreibt Einstellungen über `domain::global_settings_and_state_management::GlobalSettingsService`.
    - **Weitere Module:** Bluetooth, Audio, Energie, Eingabegeräte, Standardanwendungen, Datenschutz, Benutzerkonten, etc. Jedes Modul interagiert mit den entsprechenden Domänen- oder Systemschicht-APIs.
5. **Interaktion (UI-Seite):**
    - Primär mit `domain::global_settings_and_state_management::GlobalSettingsService` zum Lesen und Schreiben von Einstellungen (via `get_setting` und `update_setting`).
    - Für spezifische Einstellungen (Netzwerk, Audio, Theme) direkte Interaktion mit den jeweiligen Domänen- (`domain::theming`) oder Systemschicht-Schnittstellen (`system::dbus`, `system::audio`).

**VII. `ui::widgets` (System für anpassbare Widgets in Seitenleisten)**

1. **Verantwortlichkeiten:** Ermöglicht dem Benutzer das dynamische Platzieren und Konfigurieren von Informations-Widgets in den dafür vorgesehenen Seitenleisten.
2. **Komponenten und deren Untersuchungsbedarf (UI-Fokus):**
    - **`RightSidebarWidget` (und ggf. `LeftSidebarWidget`, falls dort auch Widgets platziert werden können):**
        - **Design:** Dezent transluzenter Hintergrund. Layout-Container für Widgets.
        - **GTK-Struktur:** `Gtk::Box` (vertikal), `Gtk::ScrolledWindow` falls nötig.
        - **Funktionalität (UI-Seite):** Container für `PlacedWidgetWidget`-Instanzen. Per Drag & Drop anpassbare Reihenfolge der Widgets.
    - **`WidgetManagerService` (Logische UI-Komponente, kein GTK-Widget):**
        - **Funktionalität (UI-Seite):** Verwaltet die Liste verfügbarer `ActualWidget`-Typen (Implementierungen des `ActualWidget`-Traits). Hält die Konfiguration (welche Widgets sind wo platziert, ihre spezifischen Einstellungen) und persistiert diese über `domain::global_settings_and_state_management`.
    - **`WidgetPickerPopover`:**
        - **GTK-Struktur:** `Gtk::Popover` mit einer `Gtk::ListBox` oder `Gtk::FlowBox`, die verfügbare Widget-Typen (Name, Icon, Beschreibung) anzeigt.
        - **Funktionalität (UI-Seite):** UI-Element zur Auswahl und Hinzufügung neuer Widgets zur Seitenleiste.
    - **`PlacedWidgetWidget` (benutzerdefiniertes Widget):**
        - **GTK-Struktur:** Wrapper-Widget (`Gtk::Frame` oder `Gtk::Box`), das ein spezifisches `ActualWidget` in der Seitenleiste anzeigt. Enthält Steuerelemente für Drag & Drop (via `Gtk::DragSource`, `Gtk::DropTarget`), Konfigurationszugriff (z.B. ein Einstellungs-Button, der einen Dialog öffnet) und Entfernen des Widgets.
    - **`ActualWidget` (Basis-Trait, das jedes konkrete Widget implementieren muss):**
        - **API-Definition (UI-Fokus):**
            - `fn get_gtk_widget(&self) -> Gtk::Widget;` (Gibt das darzustellende GTK-Widget zurück).
            - `fn get_config_spec() -> Vec<WidgetConfigOption>;` (Definiert konfigurierbare Optionen).
            - `fn apply_config(&mut self, config: WidgetConfigValues);`
            - `fn on_add(&mut self); fn on_remove(&mut self);` (Lebenszyklus-Hooks).
            - `fn get_name(&self) -> String; fn get_description(&self) -> String;`
        - **Beispiele für konkrete Widget-Implementierungen (als GTK-Widgets, die `ActualWidget` implementieren):**
            - **`ClockWidget`:** `Gtk::Label` oder komplexeres Layout.
            - **`CalendarWidget`:** `Gtk::Calendar` oder benutzerdefinierte Monatsansicht.
            - **`WeatherWidget`:** `Gtk::Box` mit `Gtk::Image` (Wettericon) und `Gtk::Label`s (Temperatur, Ort). Interagiert mit externen Wetter-APIs (über Systemschicht-Proxy oder direkt, falls sicher und mit `async` gehandhabt).
            - **`SystemMonitorWidget`:** `Gtk::Box` mit `Gtk::LevelBar`s oder kleinen Graphen (`Gtk::DrawingArea`) für CPU-, RAM-, Netzwerkauslastung. Liest Daten von Systemschicht-APIs oder `/proc` (asynchron!).
            - **`NotesWidget`:** `Gtk::TextView` oder `Gtk::Entry` für einfache Notizen.
            - **`MediaControllerWidget`:** `Gtk::Box` mit `Gtk::Image` (Album-Cover), `Gtk::Label`s (Titel, Künstler), `Gtk::Button`s (Play/Pause, Next, Prev). Interagiert mit MPRIS D-Bus-Schnittstelle (über `system::dbus`).
            - **`AIContextWidget` (optional):** Zeigt kontextbezogene KI-Informationen oder -Aktionen an. Interagiert mit `domain::user_centric_services::AIInteractionLogicService`.
3. **Untersuchungsbedarf (UI-Fokus):**
    - Definition einer klaren, erweiterbaren API für `ActualWidget`.
    - Implementierung eines robusten Drag & Drop-Systems für `PlacedWidgetWidget`s innerhalb der Seitenleisten.
    - Mechanismus zur (De-)Serialisierung der Widget-Konfigurationen.

**VIII. Übergreifende Untersuchungsbereiche und Abschluss für die UI-Schicht**

1. **State Management in der UI:**
    - **Problem:** Wie wird der Zustand von UI-Komponenten verwaltet, insbesondere wenn er von Daten aus tieferen Schichten abhängt oder über mehrere Widgets hinweg synchronisiert werden muss?
    - **Untersuchungsbedarf (UI-Fokus):**
        - Evaluation von Rust-spezifischen State-Management-Bibliotheken (falls vorhanden und passend für GTK, z.B. `relm4` Komponentenmodell, auch wenn NovaDE nicht komplett auf `relm4` basiert, können Ideen übernommen werden).
        - Adaption von Patterns wie MVVM (Model-View-ViewModel) oder MVC (Model-View-Controller) für GTK4/Rust. Nutzung von `glib::Object` Subclassing, um Modelle mit Properties zu erstellen, an die sich UI-Widgets binden.
        - Effektive Nutzung von GTK-Properties und Bindings (`GObject::bind_property`).
        - Verwendung von `glib::ParamSpec` für benutzerdefinierte Widget-Properties.
2. **Asynchrone Operationen und UI-Responsiveness:**
    - **Problem:** Wie werden langlaufende Operationen (z.B. Laden von Daten aus dem Netzwerk für ein Widget, komplexe Filterung von Benachrichtigungen) gehandhabt, ohne die UI zu blockieren?
    - **Untersuchungsbedarf (UI-Fokus):**
        - Konsequente Nutzung von `glib::MainContext::spawn_local` für UI-Aktualisierungen aus asynchronen Kontexten (`async fn` in Rust).
        - Kommunikation zwischen UI-Thread und Hintergrund-Threads (z.B. über `async_channel` oder `tokio::sync::mpsc`, wobei Ergebnisse dann im UI-Thread via `spawn_local` verarbeitet werden).
        - Design von UI-Feedback für laufende Operationen (z.B. `Gtk::Spinner`, `Gtk::ProgressBar`, Deaktivieren von Buttons).
3. **Teststrategien für die UI:**
    - **Problem:** Wie kann die Korrektheit und das Verhalten von GTK4-Widgets und der gesamten UI-Logik getestet werden?
    - **Untersuchungsbedarf (UI-Fokus):**
        - **Unit-Tests:** Testen der Logik von UI-Helferfunktionen und State-Management-Komponenten, die nicht direkt von GTK-Rendering abhängen. Mocking von Abhängigkeiten zu unteren Schichten und zu GTK-Interna (wo möglich).
        - **Widget-Tests:** Nutzung von GTK-Inspektionswerkzeugen (wie dem GTK Inspector) und Accessibility-APIs (`atk`) zur programmatischen Überprüfung von Widget-Zuständen, -Eigenschaften und -Hierarchien. Evaluierung von Frameworks oder Ansätzen wie `gtk4-rs-test-utils` (falls verfügbar/passend) oder Schreiben eigener Test-Helfer.
        - **Visuelle Regressionstests:** Evaluierung von Werkzeugen und Strategien für visuelle Vergleiche von UI-Zuständen (Screenshots). Dies ist oft komplex in dynamischen UIs.
        - **End-to-End-Tests (Integrationstests der UI):** Simulation von Benutzerinteraktionen auf Wayland-Ebene (sehr anspruchsvoll) oder über Accessibility-APIs und Überprüfung des UI-Verhaltens. Fokus auf kritische Benutzerpfade.
4. **Performance-Optimierung und Profiling der UI:**
    - **Problem:** Sicherstellung einer durchgehend performanten UI, insbesondere bei komplexen Layouts, vielen Widgets oder dynamischen Aktualisierungen.
    - **Untersuchungsbedarf (UI-Fokus):**
        - Identifikation und Nutzung von Profiling-Werkzeugen für Rust und GTK4 (z.B. `perf`, GTK-Debugger/Inspector, spezifische Rust-Profiler wie `flamegraph`).
        - Analyse von Rendering-Zeiten, Speicherverbrauch und CPU-Auslastung der UI-Komponenten.
        - Optimierung von Widget-Zeichnung (`Gtk::Snapshot` in GTK4), CSS-Anwendung und Datenbindung.
        - Lazy Loading von UI-Teilen oder Daten.
        - Verwendung von `Gtk::ListView` und `Gtk::GridView` für lange Listen anstelle von `Gtk::ListBox` oder `Gtk::FlowBox` wo Performance kritisch ist.
5. **Internationale und Lokalisierung (i18n/l10n) der UI:**
    - **Problem:** Vorbereitung der UI für die Übersetzung in verschiedene Sprachen und Anpassung an regionale Formate.
    - **Untersuchungsbedarf (UI-Fokus):**
        - Integration von Gettext (`gettext-rs` Crate) oder Fluent (`fluent-rs` Crate) mit GTK4/Rust.
        - Management von Übersetzungsdateien (`.po`, `.ftl`).
        - Sicherstellung, dass UI-Layouts mit unterschiedlich langen Texten umgehen können (Verwendung von GTK-Layout-Managern, die dynamische Größenanpassung unterstützen).
        - Formatierung von Zahlen, Daten und Zeiten gemäß der lokalen Einstellungen.
6. **Planungs- und Managementaspekte (UI-Fokus):**
    - **Priorisierung:** Welche UI-Komponenten und -Funktionen sind für einen ersten Prototyp/MVP (Minimum Viable Product) unerlässlich? (z.B. Panel, grundlegender Workspace-Switcher, einfache Fenster-Interaktion).
    - **Abhängigkeiten:** Welche UI-Komponenten hängen stark von der Fertigstellung bestimmter Domänen- oder Systemschichtfunktionen ab? (z.B. `NetworkIndicatorWidget` von `system::dbus::networkmanager_client`).
    - **Parallelisierung:** Welche UI-Module können parallel entwickelt werden? (z.B. `ControlCenter`-Module, verschiedene `ActualWidget`-Implementierungen).
    - **Design-Prozess:** Enge und kontinuierliche Zusammenarbeit mit UX/UI-Designern, um Mockups und Prototypen zu erstellen, zu validieren und Iterationen basierend auf Feedback durchzuführen. Verwendung von Werkzeugen, die Design-Spezifikationen (Farben, Abstände, Schriftarten als Tokens) an die Entwickler weitergeben können.
    - **Dokumentation:** Umfassende Dokumentation der UI-Komponenten, ihrer APIs, ihrer GTK-Struktur, ihrer Interaktionen und der Designentscheidungen dahinter.

Dieser detaillierte Recherche- und Untersuchungsplan für die UI-Schicht bildet die Grundlage für die erfolgreiche Konzeption und Implementierung der Benutzeroberfläche von NovaDE. Er dient als Leitfaden, um sicherzustellen, dass alle relevanten Aspekte berücksichtigt werden und eine hochwertige Benutzererfahrung entsteht.

**NovaDE Projektplan: Etappen und Aufgabenpakete**

**Etappe 0: Projekt-Setup und Fundamentale Infrastruktur**

- **Aufgabenpaket 0.1: Projektinitialisierung und Werkzeuge**
    - Einrichtung der Versionskontrolle (Git-Repository-Struktur für `novade-core`, `novade-domain`, `novade-system`, `novade-ui`).
    - Konfiguration des Build-Systems (Meson).
    - Definition der grundlegenden CI/CD-Pipeline (Formatierung mit `rustfmt`, Linting mit `clippy`, Basis-Builds).
    - Festlegung und Dokumentation von Coding-Styleguides und API-Design-Richtlinien.
    - Einrichtung der Entwicklungsumgebungen und Kommunikationskanäle.
- **Aufgabenpaket 0.2: Kernschicht – Basistypen und Fehlerbehandlung (`core::types`, `core::errors`)**
    - Implementierung fundamentaler Datentypen in `core::types` (`Point<T>`, `Size<T>`, `Rect<T>`, `RectInt`, `Color`, `Orientation`, Re-Export von `uuid::Uuid`, `chrono::DateTime<Utc>`).
    - Definition der Basis-Fehlerbehandlungsstrategie in `core::errors` (`CoreError` mit `thiserror`, `ColorParseError`, Richtlinien für Modul-spezifische Fehler).
    - Umfassende Unit-Tests für alle Typen und Fehlerdefinitionen.
- **Aufgabenpaket 0.3: Kernschicht – Logging und Konfiguration (`core::logging`, `core::config`)**
    - Implementierung der Logging-Infrastruktur in `core::logging` (`initialize_logging()` mit `tracing`, `LogFormat`, `LoggingError`).
    - Implementierung der Konfigurationsprimitive in `core::config` (`ConfigError`, `CoreConfig`-Struktur für TOML, `load_core_config()`, globaler Zugriff via `OnceCell`).
    - Unit-Tests für Logging-Initialisierung (konzeptionell) und Konfigurationsladen/-zugriff.
- **Aufgabenpaket 0.4: Kernschicht – Utilities (`core::utils`)**
    - Implementierung initial identifizierter, allgemeiner Hilfsfunktionen (z.B. `file_utils`, `string_utils`).
    - Definition spezifischer Fehler für Utilities, falls notwendig.
    - Umfassende Unit-Tests.

**Etappe 1: Domänenschicht (Domain Layer) – Definition der Kernlogik und -zustände**

- **Aufgabenpaket 1.1: Fundamentale Domänen-Typen und -Events (`domain::shared_types`, `domain::common_events`)**
    - Implementierung von `ApplicationId`, `UserSessionState`, `ResourceIdentifier`.
    - Implementierung von `UserActivityType`, `UserActivityDetectedEvent`, `ShutdownReason`, `SystemShutdownInitiatedEvent`.
- **Aufgabenpaket 1.2: Theming-Engine (`domain::theming`)**
    - Implementierung aller Theming-Datenstrukturen (von `TokenIdentifier` bis `ThemingConfiguration`).
    - Definition des `ThemingError`-Enums.
    - Implementierung der Logik für Token-Management (Laden, Validieren, Auflösungspipeline inklusive Akzentfarben, Fallback-Theme). (Dateizugriffe `async`).
    - Implementierung des `ThemingEngine`-Service-Traits und der `DefaultThemingEngine` (interner Zustand mit `tokio::sync::Mutex`, Event-Versand mit `tokio::sync::broadcast`).
- **Aufgabenpaket 1.3: Globale Einstellungen (`domain::global_settings_and_state_management`)**
    - Definition aller Einstellungsstrukturen (`GlobalDesktopSettings` etc.) und des `SettingPath`-Enums.
    - Definition des `GlobalSettingsError`-Enums.
    - Implementierung des `SettingsPersistenceProvider`-Traits und des `FilesystemSettingsProvider` (async, nutzt `core::config::ConfigServiceAsync`).
    - Implementierung des `GlobalSettingsService`-Traits und der `DefaultGlobalSettingsService` (interner Zustand mit `tokio::sync::RwLock`, Event-Versand).
- **Aufgabenpaket 1.4: Workspace-Verwaltung (`domain::workspaces`)**
    - `core`: `Workspace`-Entität, `WindowIdentifier`, `WorkspaceLayoutType`, `WorkspaceCoreError`, Event-Payloads.
    - `assignment`: Fensterzuweisungslogik, `WindowAssignmentError`.
    - `config`: `WorkspaceSnapshot`-Strukturen, `WorkspaceConfigProvider`-Trait und `FilesystemConfigProvider` (async). `WorkspaceConfigError`.
    - `manager`: `WorkspaceManagerService`-Trait und `DefaultWorkspaceManager` (async, Event-Versand). `WorkspaceEvent`-Enum.
- **Aufgabenpaket 1.5: Benachrichtigungs-Subsysteme (`domain::notifications_core`, `domain::notifications_rules`)**
    - `notifications_core`: `Notification`-Datenstrukturen, `NotificationInput`, `NotificationService`-Trait und `DefaultNotificationService` (async, interagiert mit `NotificationRulesEngine`). `NotificationError`. `NotificationEventEnum`.
    - `notifications_rules`: Regel-Datenstrukturen (`RuleCondition`, `RuleAction`, `NotificationRule`), `NotificationRulesProvider`-Trait und Implementierung. `NotificationRulesEngine`-Trait und Implementierung (async). `NotificationRulesError`.
- **Aufgabenpaket 1.6: KI-Interaktionslogik (`domain::user_centric_services::ai_interaction`)**
    - KI-Datenstrukturen (`AIInteractionContext`, `AIConsent`, `AIModelProfile`, `AttachmentData`). `AIInteractionError`.
    - `AIConsentProvider`- und `AIModelProfileProvider`-Traits und Implementierungen (async, nutzen `core::config`).
    - `AIInteractionLogicService`-Trait und `DefaultAIInteractionLogicService` (async). `AIInteractionEventEnum`.
- **Aufgabenpaket 1.7: Fensterverwaltungsrichtlinien (`domain::window_management_policy`)**
    - Policy-Datenstrukturen (`TilingMode`, `GapSettings`, `WindowSnappingPolicy`, `WorkspaceWindowLayout`). `WindowPolicyError`.
    - `WindowManagementPolicyService`-Trait und `DefaultWindowManagementPolicyService` (async, interagiert mit `GlobalSettingsService`). Implementierung der Layout-Algorithmen.

**Etappe 2: Systemschicht (System Layer) – OS-Interaktion und Technische Umsetzung**

- **Aufgabenpaket 2.1: Fundament des Wayland Compositors (`system::compositor`)**
    - `compositor::core`: `DesktopState`-Grundstruktur (mit Smithay-States), `CompositorHandler`, `SurfaceData`.
    - `compositor::shm`: SHM-Pufferbehandlung (`ShmHandler`, `ShmError`).
    - `compositor::xdg_shell`: `XdgShellHandler`, `ManagedWindow`-Struktur (implementiert `smithay::desktop::Window`), `XdgShellError`.
    - `compositor::renderer_interface`: Definition der abstrakten Renderer-Traits (`FrameRenderer`, `RenderableTexture`).
- **Aufgabenpaket 2.2: Eingabeverarbeitung (`system::input`)**
    - `input::seat_manager`: `SeatHandler`-Implementierung für `DesktopState`, `XkbKeyboardData`-Verwaltung.
    - `input::libinput_handler`: `LibinputInputBackend`-Initialisierung, `calloop`-Integration.
    - `input::keyboard`, `input::pointer`, `input::touch`: Event-Übersetzer, Fokuslogik für Basiseingaben.
    - Definition des `InputError`-Enums.
- **Aufgabenpaket 2.3: D-Bus Basisinfrastruktur und erste Dienste (`system::dbus_interfaces`)**
    - `dbus_interfaces::common`: `DBusConnectionManager` (`OnceLock` für Bus-Verbindungen), `DBusInterfaceError`.
    - `dbus_interfaces::upower_client`: Client für `org.freedesktop.UPower`. Typen, Proxies, Service-Implementierung, Event-Publishing.
    - `dbus_interfaces::logind_client`: Client für `org.freedesktop.login1`.
- **Aufgabenpaket 2.4: Hardware- und Protokollmanagement (Basis)**
    - `system::outputs`: Basis-Output-Verwaltung (`OutputDevice`), Implementierung der Handler für `wl_output` und `xdg-output-unstable-v1`.
    - `system::audio_management`: PipeWire-Client-Grundstruktur (`PipeWireClientService`), Verbindung und `MainLoop`-Management.
- **Aufgabenpaket 2.5: Event-Brücke (`system::event_bridge`)**
    - Implementierung des `SystemEventBridge` mit `tokio::sync::broadcast` für `SystemLayerEvent`.
    - Definition erster System-interner Events (z.B. `InputDeviceHotplugEvent`).

**Etappe 3: Systemschicht – Erweiterte Funktionen und Protokolle**

- **Aufgabenpaket 3.1: Erweiterte Compositor-Funktionen (`system::compositor`)**
    - `compositor::layer_shell`: Handler für `wlr-layer-shell-unstable-v1`.
    - `compositor::decoration`: Handler für `xdg-decoration-unstable-v1`.
    - `compositor::output_management`: Handler für `wlr-output-management-unstable-v1`.
    - `compositor::power_management`: Handler für `wlr-output-power-management-unstable-v1`.
    - `compositor::xwayland`: Basis-Integration für XWayland.
- **Aufgabenpaket 3.2: Weitere D-Bus-Schnittstellen (`system::dbus_interfaces`)**
    - `dbus_interfaces::network_manager_client`: Client für NetworkManager.
    - `dbus_interfaces::notifications_server`: Implementierung des `org.freedesktop.Notifications`-Servers.
    - `dbus_interfaces::secrets_service_client`: Client für `org.freedesktop.secrets`.
    - `dbus_interfaces::policykit_client`: Client für PolicyKit.
- **Aufgabenpaket 3.3: Vervollständigung Hardware-Management**
    - `system::audio_management`: Vollständige Implementierung von Geräte-/Stream-Erkennung, Lautstärkeregelung, Default-Geräte-Management.
    - `system::power_management`: Implementierung des `PowerManagementService` (DPMS, Idle-Timer, Interaktion mit `logind`).
- **Aufgabenpaket 3.4: Fortgeschrittene Systemdienste**
    - `system::mcp_client`: Implementierung des MCP-Clients, Prozessmanagement, API-Key-Handling.
    - `system::portals`: Implementierung der Backends für XDG FileChooser und Screenshot.
    - `system::window_mechanics`: Implementierung der technischen Fenster-Mechaniken (Layout-Anwendung, interaktive Operationen, Fokus).
    - `system::input::gestures`: Implementierung der Gestenerkennung.

**Etappe 4: Benutzeroberflächenschicht (UI Layer) – Darstellung und Interaktion**

- **Aufgabenpaket 4.1: UI-Grundgerüst und Kern-Shell-Elemente**
    - `ui::app_initializer`, `ApplicationState`, `MasterActionHandler`.
    - `ui::main_window` (Basis-Layout).
    - `ui::theming_gtk` (Anbindung an `domain::theming`).
    - `ui::shell::PanelWidget` (Basis-Panel, `AppMenuButton`, Uhr-Widget).
    - `ui::shell::WorkspaceSwitcherWidget` (Basis-Implementierung, Anbindung an `WorkspaceManagerService`).
- **Aufgabenpaket 4.2: Erweiterte Shell-Funktionen und Kern-Interaktionskomponenten**
    - `ui::shell::SmartTabBarWidget`.
    - `ui::shell::QuickSettingsPanelWidget` und Anbindung der System-Indikatoren (Netzwerk, Audio, Energie über Systemschicht-APIs).
    - `ui::shell::QuickActionDockWidget`.
    - `ui::command_palette` (Basis-Widget und Anbindung an `MasterActionHandler`).
    - `ui::notifications_frontend::NotificationPopupWidget` (Anzeige via `gtk4-layer-shell`).
    - `ui::shell::NotificationCenterPanelWidget`.
- **Aufgabenpaket 4.3: Fenstermanagement-UI und Systemeinstellungen**
    - `ui::window_manager_frontend::OverviewModeWidget`.
    - `ui::window_manager_frontend::AltTabSwitcherWidget`.
    - `ui::control_center` (Grundstruktur, erste Module wie "Erscheinungsbild" mit Anbindung an `GlobalSettingsService` und `ThemingEngine`).
    - `ui::speed_dial` (Startansicht).
- **Aufgabenpaket 4.4: Widgets und fortgeschrittene UI-Funktionen**
    - `ui::widgets::RightSidebarWidget` und Widget-System (WidgetManagerService, WidgetPicker).
    - Implementierung spezifischer Widgets (Kalender, Wetter, Systemmonitor).
    - Vollständige Integration von KI-Funktionen in die UI (Befehlspalette, kontextuelle Aktionen).
    - UI-Seite der XDG Desktop Portal-Nutzung (z.B. Dateiauswahldialoge über `system::portals`).

**Etappe 5: Integration, Tests, Verfeinerung und Release-Vorbereitung**

- **Aufgabenpaket 5.1: Schichtübergreifende Integration und Tests**
    - Durchführung umfassender Integrationstests für alle Kern-Workflows.
    - End-to-End-Szenariotests basierend auf `Beschreibung Nutzung.md`.
- **Aufgabenpaket 5.2: Performance-Analyse und -Optimierung**
    - Profiling kritischer Pfade (Compositor, Input, UI-Rendering, Startzeit).
    - Optimierung des Ressourcenverbrauchs.
- **Aufgabenpaket 5.3: UI/UX-Feinschliff und Benutzerfeedback**
    - Überprüfung der gesamten Benutzererfahrung auf Konsistenz, Intuitivität und Ästhetik.
    - Anpassung von Animationen, Layouts, Interaktionen.
    - Einholung und Verarbeitung von Benutzerfeedback (Alpha-/Beta-Tests).
- **Aufgabenpaket 5.4: Sicherheitsüberprüfungen und Dokumentation**
    - Sicherheitsaudit für KI-Integration, D-Bus-Schnittstellen, Portal-Backends, Geheimnisverwaltung.
    - Vervollständigung der Benutzer-, Entwickler- und Architekturdokumentation.
- **Aufgabenpaket 5.5: Release-Management**
    - Erstellung von Release Notes.
    - Vorbereitung der finalen Builds.

**Etappe 6: Deployment und Post-Release**

- **Aufgabenpaket 6.1: Paketierung und Distribution**
    - Erstellung von Paketen für Ziel-Distributionen (.deb, .rpm).
    - Evaluierung und ggf. Erstellung eines Flatpak-Pakets.
    - Testen der Installations- und Update-Prozesse.
- **Aufgabenpaket 6.2: Community-Aufbau und Wartung**
    - Einrichtung von Kanälen für Benutzer-Support und Feedback.
    - Etablierung von Prozessen für Bugfixing und die Veröffentlichung von Wartungsupdates.
    - Planung zukünftiger Feature-Releases.

---

Dieser Plan ist ein High-Level-Entwurf und muss im Laufe des Projekts weiter verfeinert und angepasst werden. Er dient jedoch als grundlegende Struktur für die Organisation der Entwicklungsaufwände.

Sie haben Recht, die bisherigen Spezifikationen konzentrieren sich auf die Kern-Desktop-Umgebung und lassen typische Desktop-Anwendungen wie einen Dateimanager, Texteditor, Bildbetrachter usw. außen vor. Die Frage, ob NovaDE eher ein spezialisierter Fenstermanager oder eine vollwertige Desktop-Umgebung mit eigenen Anwendungen sein soll, ist fundamental.

Basierend auf der Vision ("intuitive und KI-gestützte Benutzererfahrung", "optimiert für Entwickler, Kreative und alltägliche Nutzer") und den detaillierten UI-Beschreibungen (Control Center, Speed Dial etc.) deutet vieles darauf hin, dass NovaDE mehr als nur ein Fenstermanager sein soll. Es zielt auf eine **integrierte Erfahrung** ab.

**Was wäre noch für ein vollwertiges System nötig?**

Um NovaDE zu einem System zu machen, das mit GNOME oder KDE Plasma vergleichbar ist (als vollständige Desktop-Umgebungen), wären zusätzlich zu den bereits spezifizierten Komponenten mindestens folgende Elemente und Anwendungen notwendig:

**1. Kernanwendungen (Eigene oder angepasste Drittanwendungen):**

- **Dateimanager:**
    - Grundfunktionen: Durchsuchen von Verzeichnissen, Dateioperationen (Kopieren, Verschieben, Löschen, Umbenennen), Erstellen von Ordnern/Dateien, Vorschauen.
    - Integration mit NovaDE: Theming, "Spaces"-Bewusstsein (falls relevant), KI-Dateiaufgaben (Zusammenfassen, Analysieren), XDG-Portals für sicheren Zugriff.
    - Technologie: Wahrscheinlich GTK4 für Konsistenz.
- **Texteditor:**
    - Einfacher Editor für schnelle Notizen und Code-Snippets.
    - Syntax-Hervorhebung (ggf. über existierende Bibliotheken).
    - Theming-Integration.
- **Terminalemulator:**
    - Essentiell für Entwickler und Power-User.
    - Integration mit Shell-Features, ggf. Tabs, Split-Views.
    - Theming.
- **Bildbetrachter:**
    - Anzeige gängiger Bildformate.
    - Basis-Operationen (Drehen, Zoom).
- **Dokumentenbetrachter (PDF, etc.):**
    - Anzeige von PDFs und ggf. anderen Dokumentformaten.
- **Software-Center / App Store:**
    - Suchen, Installieren und Verwalten von Anwendungen (z.B. Flatpaks, native Pakete).
    - Integration mit Systemupdates.
- **Archivmanager:**
    - Erstellen und Entpacken von Archiven (zip, tar.gz etc.).
- **Systemmonitor (detaillierter):**
    - Ergänzend zum Widget, eine Anwendung zur detaillierten Anzeige von Prozessen, Ressourcenverbrauch, Netzwerkauslastung.
- **Mediaplayer (Audio/Video):**
    - Abspielen gängiger Multimedia-Formate.
    - Integration mit dem PipeWire-Backend der Systemschicht.

**2. System-Utilities und -Dienste (Erweiterungen):**

- **Druckerverwaltung:**
    - Einrichtung und Verwaltung von Druckern (oft über CUPS-Integration via D-Bus).
- **Benutzer- und Gruppenverwaltung:**
    - UI zur Verwaltung von Benutzerkonten.
- **Energieverwaltung (UI-Erweiterung):**
    - Detailliertere Einstellungen und Profile im Control Center als nur die Policies in der Domänenschicht.
- **Barrierefreiheits-Werkzeuge:**
    - Bildschirmlupe, Bildschirmtastatur, Screenreader-Integration (Orca). Die UI-Schicht muss ATK/AT-SPI unterstützen.
- **Backup-Lösung:**
    - System zur Sicherung von Benutzerdaten.
- **Login-Manager / Display-Manager (Greeter):**
    - Der Login-Bildschirm, der Benutzerauthentifizierung vor dem Start der Desktop-Session durchführt.
    - Muss mit `systemd-logind` und PAM integriert sein.
    - Sollte das NovaDE-Theming widerspiegeln.
- **Bildschirmsperre (Lock Screen):**
    - Sperrt die Sitzung bei Inaktivität oder manuell.
    - Benötigt Integration mit `logind`.
- **System-Update-Mechanismus (UI):**
    - Eine UI zur Benachrichtigung über und Durchführung von System- und Anwendungsupdates.
- **Zwischenablage-Manager (fortgeschritten):**
    - Verlauf, Anpinnen von Einträgen (aktuell ist nur die Basis-Daten-Device-Funktionalität in der Systemschicht skizziert).
- **Screenshot- und Screencast-Werkzeug (UI):**
    - Eine Benutzeroberfläche für die Screenshot-/Screencast-Funktionalität, die von `system::portals` oder `system::compositor::screencopy` bereitgestellt wird.
- **Farbwähler-Werkzeug (UI):**
    - Eine UI für die Farbauswahlfunktion des Screenshot-Portals.

**3. Entwicklungs- und Design-Philosophie für Anwendungen:**

- **Eigene Anwendungen vs. Kompatibilität:**
    - **Eigene Anwendungen:** Bieten die beste Integration mit dem NovaDE-Theming, den KI-Funktionen und der Designphilosophie. Erfordern aber erheblichen Entwicklungsaufwand.
    - **Kompatibilität mit GNOME/KDE-Apps:** NovaDE als Wayland-Compositor mit XWayland-Unterstützung wird GNOME (GTK) und KDE (Qt) Anwendungen grundsätzlich ausführen können. Die Herausforderung liegt im einheitlichen Erscheinungsbild.

**Wie kompliziert ist es, KDE (Qt) und GNOME (GTK) Apps gleich aussehen zu lassen?**

Es ist **sehr kompliziert** und oft nur bis zu einem gewissen Grad erreichbar, ein _perfekt_ einheitliches Erscheinungsbild über Qt- und GTK-Anwendungen hinweg zu erzielen, besonders wenn NovaDE selbst primär auf GTK4 basiert.

- **Theming-Engines und Stil-Abstraktionen:**
    
    - **GTK-Themes:** GTK-Anwendungen (GNOME, XFCE, etc.) verwenden GTK-Themes (CSS-basiert). NovaDEs Theming-Engine wird GTK4-CSS generieren. Das bedeutet, GTK-Anwendungen können relativ gut an das NovaDE-Theme angepasst werden, solange sie Standard-GTK-Widgets verwenden und das Theme korrekt interpretieren.
    - **Qt-Themes/Styles:** KDE Plasma und Qt-Anwendungen verwenden Qt-Styles (z.B. Breeze, Kvantum) und Farbschemata. Qt hat eigene Mechanismen zur Stilanpassung, die sich von GTK unterscheiden.
    - **Abstraktionsversuche:** Projekte wie `QGtkStyle` oder `Adwaita-Qt` versuchen, GTK-Themes auf Qt-Anwendungen anzuwenden oder das Aussehen von Qt-Anwendungen an Adwaita (das Standard-GNOME-Theme) anzupassen. Diese funktionieren oft gut für grundlegende Widget-Stile, aber es gibt Grenzen, besonders bei komplexen Anwendungen oder benutzerdefinierten Widgets.
    - **Icon-Themes:** Freedesktop-Icon-Themes (die NovaDE unterstützt) werden von den meisten GTK- und Qt-Anwendungen respektiert, was zu konsistenten Icons führt.
    - **Cursor-Themes:** Werden ebenfalls systemweit gesetzt und meist respektiert.
    - **Schriftarten:** Globale Schrifteinstellungen werden in der Regel von beiden Toolkits übernommen.
- **Herausforderungen für ein einheitliches Erscheinungsbild:**
    
    - **Fensterdekorationen (CSD vs. SSD):**
        - GNOME-Apps verwenden stark Client-Side Decorations (CSD), bei denen die Titelleiste Teil des Anwendungsfensters ist.
        - Traditionelle Qt/KDE-Apps verwenden oft Server-Side Decorations (SSD), bei denen der Fenstermanager die Titelleiste zeichnet. Unter Wayland wird mit `xdg-decoration` versucht, dies zu vereinheitlichen, wobei der Client Präferenzen äußern kann.
        - NovaDE plant die Unterstützung für `xdg-decoration`. Um ein einheitliches Aussehen zu erzielen, müsste NovaDE entweder sehr gute SSDs für Qt-Apps bereitstellen, die zum CSD-Stil der eigenen (GTK4) Apps passen, oder hoffen, dass Qt-Apps zunehmend Wayland-konforme CSDs implementieren (was bei KDE der Fall ist) oder das `xdg-decoration`-Protokoll nutzen, um dem Server die Kontrolle zu überlassen.
    - **Widget-Toolkits:** GTK und Qt haben fundamental unterschiedliche Widget-Sets und Rendering-Philosophien. Feine Unterschiede in Abständen, Animationen, Verhalten von Widgets (z.B. Scrollbalken, Menüs) sind schwer vollständig zu eliminieren.
    - **Farbschemata und Akzentfarben:** Während NovaDE eine systemweite Akzentfarbe plant, müssen sowohl GTK- als auch Qt-Theming-Engines diese korrekt interpretieren und anwenden. Für GTK ist dies über das eigene Theming-System machbar. Für Qt-Apps müsste ein passender Qt-Style oder eine Konfiguration existieren, die diese Akzentfarbe übernimmt. KDE Plasma hat eigene Mechanismen für Akzentfarben.
    - **Dialoge:** Systemdialoge (Datei öffnen/speichern) werden idealerweise über XDG Desktop Portals gehandhabt, was zu konsistenten Dialogen führt, die vom Host-System (NovaDE) bereitgestellt werden. NovaDE plant, Backend-Logik für Portale bereitzustellen.
    - **Anwendungsspezifisches Styling:** Viele Anwendungen (besonders große wie Browser, Office-Suiten) bringen ihr eigenes, stark angepasstes UI-Design mit, das sich nur begrenzt durch System-Themes beeinflussen lässt.
- **Wie machen es andere (z.B. Fenstermanager wie Hyprland)?**
    
    - **Fokus auf Fenstermanagement:** Tiling Window Manager wie Hyprland konzentrieren sich primär auf die Anordnung und Verwaltung von Fenstern (oft ohne eigene Titelleisten – "borderless"). Das Aussehen _innerhalb_ der Anwendungsfenster wird weitgehend den Anwendungen und ihren Toolkits überlassen.
    - **Konfigurationsbasierte Anpassung:** Nutzer solcher WMs passen oft Konfigurationsdateien an, um GTK- und Qt-Themes zu setzen, die visuell ähnlich sind (z.B. ein dunkles Materia-Theme für GTK und ein passendes Kvantum-Theme für Qt).
    - **Konsistenz durch Minimalismus:** Viele Nutzer von Tiling WMs bevorzugen ohnehin einen minimalistischen Look und wählen Themes, die nicht stark toolkit-spezifisch sind.
    - **Keine vollständige Kontrolle:** Sie versuchen nicht, eine pixelgenaue Übereinstimmung zu erzwingen, sondern eine generelle visuelle Harmonie über Schriftarten, Grundfarben und Icon-Themes.
    - **XSettings / GSettings Daemons:** Einige Fenstermanager oder leichtgewichtige Desktops nutzen Daemons, die GSettings (von GNOME) lesen und diese Einstellungen (Theme-Namen, Schriftarten etc.) an GTK- und teilweise auch an Qt-Anwendungen (über Qt-Platform-Themes wie `qt5ct`/`qt6ct` oder `QGnomePlatform`) weitergeben.

**Fazit für NovaDE:**

- **Vollständige DE vs. Fenstermanager:** Die Spezifikationen deuten stark auf den Anspruch einer **vollständigen Desktop-Umgebung** hin, nicht nur eines Fenstermanagers. Die Planung eigener Komponenten wie Control Center, Speed Dial, Panel-Widgets etc. unterstreicht dies.
- **Erscheinungsbild:**
    - Für **eigene NovaDE-Anwendungen (GTK4)** wird ein sehr einheitliches Erscheinungsbild erreichbar sein, da sie direkt das `domain::theming`-System nutzen.
    - Für **andere GTK-Anwendungen** wird ein gutes Maß an Konsistenz möglich sein, wenn diese Standard-GTK4-Widgets verwenden und die von NovaDE generierten CSS-Variablen (Tokens) respektieren.
    - Für **Qt/KDE-Anwendungen** wird es schwieriger. Der beste Ansatz ist:
        1. Bereitstellung eines hochwertigen **Standard-GTK-Themes** durch NovaDE.
        2. Bereitstellung oder Empfehlung eines **Qt-Styles/Kvantum-Themes**, das optisch sehr gut zum NovaDE-Standardtheme passt.
        3. Sicherstellung, dass **globale Einstellungen** (Schriftarten, Icons, Cursor, Akzentfarbe) so exponiert werden (z.B. über GSettings oder einen XSettings-Daemon), dass Qt-Platform-Themes sie lesen und an Qt-Anwendungen weitergeben können.
        4. Konsequente Nutzung von **XDG Desktop Portals** für Systemdialoge.
        5. Korrekte Handhabung von **Server-Side Decorations** für Qt-Apps, die optisch zu den CSDs der GTK-Apps passen, wenn der Client SSDs anfordert.

Ein _perfekt_ identisches Aussehen ist unrealistisch, aber eine hohe visuelle Kohärenz ist durch sorgfältige Theme-Gestaltung und Nutzung von Standards erreichbar. Der Aufwand hierfür ist beträchtlich und erfordert Expertise in GTK- und Qt-Theming.


**I. Grundlagen der UI-Schicht (Wiederholung und Vertiefung)**

1. **Technologie-Stack (Verbindlich):**
    
    - **GUI-Toolkit:** GTK4 mit `gtk4-rs` Bindings.
        - _Untersuchungsbedarf:_ Best Practices für Strukturierung großer GTK4/Rust-Anwendungen, State-Management-Patterns innerhalb von GTK4 (z.B. Model-View-ViewModel (MVVM)-ähnliche Ansätze, Komposition, Properties/Bindings), fortgeschrittene Widget-Erstellung und -Anpassung. Umgang mit dem GTK4-Event-Loop und Integration mit Rusts `async/await` (speziell `glib::MainContext::spawn_local`).
    - **Theming:** Anwendung der von `domain::theming` generierten CSS-Stile über `Gtk::CssProvider`.
        - _Untersuchungsbedarf:_ Dynamische CSS-Aktualisierung zur Laufzeit, effiziente Anwendung von Design-Tokens auf GTK-Widgets, Performance-Implikationen komplexer CSS-Selektoren und -Regeln. Erstellung benutzerdefinierter GTK-Widgets, die optimal auf dynamisches Theming reagieren.
    - **Interaktion mit unteren Schichten:**
        - Nutzung der APIs und Event-Systeme der Domänen- und Systemschicht.
        - _Untersuchungsbedarf:_ Patterns für die Kommunikation und Datenkonvertierung zwischen UI-Typen und Domänen-/System-Typen. Fehlerbehandlung und Darstellung von Fehlern aus tieferen Schichten.
2. **Allgemeine UI/UX-Prinzipien (Visionstreu):**
    
    - **Konsistenz:** Einheitliches Erscheinungsbild und Verhalten über alle UI-Komponenten hinweg.
    - **Feedback:** Visuelles und haptisches (wo sinnvoll) Feedback auf Benutzeraktionen.
    - **Effizienz:** Minimierung der Klicks/Aktionen für häufige Aufgaben.
    - **Zugänglichkeit (Accessibility):** Einhaltung von a11y-Standards (ATK/AT-SPI).
        - _Untersuchungsbedarf:_ GTK4-spezifische Accessibility-Features, Teststrategien für Accessibility.
    - **Performance:** Flüssige Animationen, schnelle Reaktionszeiten, geringer Ressourcenverbrauch.
        - _Untersuchungsbedarf:_ Profiling-Werkzeuge für GTK4/Rust, Optimierung von Rendering-Pfaden, Vermeidung von UI-Blockaden durch langlaufende Operationen.
    - **Anpassbarkeit:** Benutzerdefinierte Konfiguration von Layouts, Widgets und Verhalten.

**II. Detaillierte Untersuchung der UI-Komponenten (Module)**

Für jede der folgenden Komponenten sind die spezifischen Verantwortlichkeiten, Datenstrukturen, Interaktionen, visuellen Design-Aspekte und Implementierungsdetails zu definieren:

1. **`ui::shell` (Haupt-Shell-UI)**
    
    - **Verantwortlichkeiten:** Implementierung der primären Desktop-Interaktionselemente.
    - **Komponenten und deren Untersuchungsbedarf:**
        - **`PanelWidget` (Kontroll-/Systemleiste(n))**:
            - _Design:_ Positionierung (oben/unten, konfigurierbar), Höhe, Transparenzoptionen, "Leuchtakzent"-Effekt.
            - _Module (als einzelne GTK-Widgets):_
                - `AppMenuButton`: Darstellung eines globalen Anwendungsmenüs (falls vom aktiven Fenster unterstützt/exportiert).
                    - _Interaktion:_ Mit `system::compositor` (Information über aktive App) und ggf. D-Bus (AppMenu-Standard).
                - `WorkspaceIndicatorWidget`: Visuelle Repräsentation der "Spaces" innerhalb des Panels.
                    - _Interaktion:_ Mit `domain::workspaces::WorkspaceManager` (Liste und aktiver Space).
                - `ClockDateTimeWidget`: Anzeige von Uhrzeit und Datum, ggf. Kalender-Popup bei Klick.
                    - _Konfiguration:_ Datums-/Zeitformate.
                - `SystemTrayEquivalentWidget`: Handhabung von Status-Icons (z.B. via `StatusNotifierItem` über D-Bus oder Alternativen unter Wayland).
                    - _Untersuchungsbedarf:_ Aktuelle Best Practices für System-Tray-Äquivalente unter Wayland/GTK4.
                - `QuickSettingsButtonWidget`: Button zum Öffnen des `QuickSettingsPanelWidget`.
                - `NotificationCenterButtonWidget`: Button zum Öffnen/Schließen des `NotificationCenterPanelWidget`, Anzeige einer Badge für neue Benachrichtigungen.
                - `NetworkIndicatorWidget`: Anzeige des Netzwerkstatus (Icon, SSID), Menü für Verbindungsmanagement.
                    - _Interaktion:_ Mit `system::dbus::networkmanager_client`.
                - `PowerIndicatorWidget`: Anzeige des Akkustatus/Ladezustands, Menü für Energieoptionen.
                    - _Interaktion:_ Mit `system::dbus::upower_client`.
                - `AudioIndicatorWidget`: Anzeige der Lautstärke, Stummschaltungsstatus, Menü für Audioeinstellungen/-geräte.
                    - _Interaktion:_ Mit `system::audio`.
            - _Layout:_ Konfigurierbare Anordnung der Module im Panel.
        - **`SmartTabBarWidget` (Intelligente Tab-Leiste pro "Space")**:
            - _Design:_ Horizontale Leiste, abgerundete obere Ecken für Tabs, Hervorhebung des aktiven Tabs mit Akzentfarbe.
            - _Funktionalität:_
                - Anzeige von `ApplicationTabWidgets` für "angepinnte" Apps/Split-Views.
                - Anzeige von Tabs für weitere, nicht gepinnte Anwendungen im aktuellen "Space".
                - Überlauf-Logik für viele Tabs.
                - Kontextmenü für Tabs (z.B. "Schließen", "Zu anderem Space verschieben").
            - _Interaktion:_ Mit `domain::workspaces::WorkspaceManager` (Fenster im aktuellen Space), `system::compositor` (Fensterinformationen, App-Icons, Titel).
        - **`QuickSettingsPanelWidget` (Ausklappbares Schnelleinstellungs-Panel)**:
            - _Design:_ Ausklappbares Overlay/Menü, das vom `QuickSettingsButtonWidget` geöffnet wird.
            - _Inhalt:_ Schnellumschalter und -regler für WLAN, Bluetooth, Lautstärke, Dark Mode, etc.
            - _Interaktion:_ Mit den jeweiligen Systemdiensten über Systemschicht-Schnittstellen.
        - **`WorkspaceSwitcherWidget` (Adaptive linke Seitenleiste)**:
            - _Design:_ Vertikale Leiste, im eingeklappten Zustand nur `SpaceIconWidgets`. Aufklappbar (Mouse-Over/Geste) mit Namen/Vorschau der Spaces. Hervorhebung des aktiven Space.
            - _`SpaceIconWidget`_: Anzeige des Icons der gepinnten App oder benutzerdefiniertes Icon für den Space.
            - _Funktionalität:_ Schnelle Navigation zwischen "Spaces".
            - _Interaktion:_ Mit `domain::workspaces::WorkspaceManager`.
        - **`QuickActionDockWidget` (Konfigurierbares Schnellaktionsdock)**:
            - _Design:_ Schwebend oder am Bildschirmrand andockbar (konfigurierbar).
            - _`DockItemWidget`_: Repräsentiert Apps, Dateien, Aktionen.
            - _Funktionalität:_ Drag & Drop von Apps/Dateien ins Dock. Intelligente Vorschläge. Tastaturbedienung.
            - _Interaktion:_ Mit `domain::global_settings_and_state_management` (Konfiguration), Systemschicht (Starten von Apps).
        - **`NotificationCenterPanelWidget` (Benachrichtigungszentrum)**:
            - _Design:_ Ausklappbares Panel, typischerweise von der Systemleiste erreichbar.
            - _`NotificationWidget`_: Stellt eine einzelne Benachrichtigung dar (Icon, App-Name, Zusammenfassung, Body, Aktionen, Zeitstempel).
            - _Funktionalität:_ Anzeige einer Liste aktueller und historischer Benachrichtigungen. Möglichkeit zum Schließen/Interagieren mit Benachrichtigungen. "Bitte nicht stören"-Umschalter.
            - _Interaktion:_ Mit `domain::user_centric_services::NotificationService`.
2. **`ui::control_center` (Modulare GTK4-Anwendung für Systemeinstellungen)**
    
    - **Verantwortlichkeiten:** Grafische Oberfläche zur Konfiguration aller Systemeinstellungen.
    - **Architektur:** Modulare Struktur, bei der jede Einstellungskategorie (Erscheinungsbild, Netzwerk, Workspaces, etc.) ein eigenes GTK-Widget (`SettingsModuleWidget`) ist.
    - **Kernfunktionen:**
        - Navigation zwischen Modulen (z.B. Seitenleiste oder Icon-Grid).
        - Live-Vorschau von Änderungen, wo immer möglich.
        - Suchfunktion für Einstellungen.
        - Zurücksetzen auf Standardwerte.
    - **Beispiel-Module (`SettingsModuleWidget` Implementierungen):**
        - `AppearanceSettingsWidget`: Konfiguration von Theme, Akzentfarbe, Schriftarten, Icons, Animationen, Skalierung.
            - _Interaktion:_ Mit `domain::theming::ThemingEngine` und `domain::global_settings_and_state_management::GlobalSettingsService`.
        - `NetworkSettingsWidget`: Konfiguration von Netzwerkverbindungen (WLAN, Ethernet, VPN).
            - _Interaktion:_ Mit `system::dbus::networkmanager_client`.
        - `WorkspaceSettingsWidget`: Konfiguration von dynamischen Workspaces, Standardanzahl, Verhalten des Workspace-Switchers.
            - _Interaktion:_ Mit `domain::global_settings_and_state_management::GlobalSettingsService` (für Workspace-bezogene Einstellungen).
        - _Weitere Module:_ Bluetooth, Audio, Energie, Eingabegeräte, Standardanwendungen, Datenschutz, Benutzerkonten, etc.
    - _Interaktion:_ Primär mit `domain::global_settings_and_state_management::GlobalSettingsService` zum Lesen und Schreiben von Einstellungen. Für spezifische Einstellungen (Netzwerk, Audio) direkte Interaktion mit den entsprechenden Systemschicht-Schnittstellen.
3. **`ui::widgets` (System für anpassbare Widgets in Seitenleisten)**
    
    - **Verantwortlichkeiten:** Ermöglicht dem Benutzer das Platzieren und Konfigurieren von Informations-Widgets.
    - **Komponenten und deren Untersuchungsbedarf:**
        - **`RightSidebarWidget` (Adaptive rechte Seitenleiste)**:
            - _Design:_ Dezent transluzenter Hintergrund.
            - _Funktionalität:_ Container für `PlacedWidgetWidget`-Instanzen. Per Drag & Drop anpassbar.
        - **`WidgetManagerService` (Logische UI-Komponente)**:
            - _Funktionalität:_ Verwaltet die Liste verfügbarer Widget-Typen, deren Konfigurationen und Platzierungen.
            - _Interaktion:_ Mit `domain::global_settings_and_state_management` (Persistenz der Widget-Konfiguration).
        - **`WidgetPickerPopover`**: UI-Element zur Auswahl und Hinzufügung neuer Widgets zur Seitenleiste.
        - **`PlacedWidgetWidget`**: Wrapper-Widget, das ein spezifisches `ActualWidget` in der Seitenleiste anzeigt und dessen Drag & Drop sowie Konfigurationszugriff ermöglicht.
        - **`ActualWidget` (Basis-Trait/Klasse für konkrete Widgets)**:
            - _Beispiele für konkrete Widget-Implementierungen:_
                - `ClockWidget`: Erweiterte Uhrenfunktionen.
                - `CalendarWidget`: Monatsansicht, Termine.
                - `WeatherWidget`: Aktuelles Wetter, Vorhersage.
                    - _Interaktion:_ Mit externen Wetter-APIs (ggf. über Systemschicht-Proxy).
                - `SystemMonitorWidget`: CPU-, RAM-, Netzwerkauslastung.
                    - _Interaktion:_ Mit Systemschicht-APIs oder `/proc`.
                - `NotesWidget`: Einfache Notizen.
                - `MediaControllerWidget`: Steuerung der Medienwiedergabe.
                    - _Interaktion:_ Mit MPRIS D-Bus-Schnittstelle.
                - `AIContextWidget` (optional): Zeigt kontextbezogene KI-Informationen oder -Aktionen an.
                    - _Interaktion:_ Mit `domain::user_centric_services::AIInteractionLogicService`.
            - _Untersuchungsbedarf:_ Definition einer klaren API für `ActualWidget` (Konfiguration, Datenaktualisierung, Interaktion).
4. **`ui::window_manager_frontend` (UI-Aspekte des Fenstermanagements)**
    
    - **Verantwortlichkeiten:** Darstellung und Interaktion mit Fenstern, die von der Systemschicht verwaltet werden.
    - **Komponenten und deren Untersuchungsbedarf:**
        - **Client-Side Decorations (CSD) Logik**:
            - _Funktionalität:_ Handhabung von Fensterdekorationen, die von den Anwendungen selbst gezeichnet werden (z.B. via `Gtk::HeaderBar`). Sicherstellung der Konsistenz mit dem globalen Theme.
            - _Untersuchungsbedarf:_ Integration mit dem `xdg-decoration-unstable-v1` Protokoll, um zwischen CSD und Server-Side Decorations (SSD) zu verhandeln.
        - **Server-Side Decorations (SSD) Rendering** (falls implementiert):
            - _Funktionalität:_ Zeichnen von Fensterrahmen und Titelleisten durch den Compositor, falls Anwendungen keine CSDs bereitstellen. Design muss anpassbar und themenkonform sein.
            - _Interaktion:_ Mit `system::compositor` und `domain::theming`.
        - **`OverviewModeWidget` (Fenster- und Workspace-Übersicht)**:
            - _Design:_ Interaktive Kacheln mit Live-Vorschau der Fenster. Hintergrund abgedunkelt/unscharf. Darstellung von Workspaces für Drag & Drop von Fenstern.
            - _Funktionalität:_ Aktivierung durch Geste/Tastenkürzel. Fenster auswählen, schließen, zwischen Workspaces verschieben.
            - _Interaktion:_ Mit `system::compositor` (Fensterliste, Thumbnails/Previews), `domain::workspaces::WorkspaceManager`.
        - **`AltTabSwitcherWidget` (Traditioneller Fensterwechsler)**:
            - _Design:_ Overlay-Anzeige der laufenden Anwendungen/Fenster für schnellen Wechsel.
            - _Funktionalität:_ Aktivierung durch Alt+Tab.
            - _Interaktion:_ Mit `system::compositor`.
5. **`ui::notifications_frontend` (Darstellung von Benachrichtigungen)**
    
    - **Verantwortlichkeiten:** Visuelle Präsentation von System- und Anwendungsbenachrichtigungen.
    - **Komponenten und deren Untersuchungsbedarf:**
        - **`NotificationPopupWidget` (Pop-up-Benachrichtigungen)**:
            - _Design:_ Dezent, im Dark Mode Stil mit Akzentfarbe je nach Dringlichkeit. Positionierung auf dem Bildschirm (z.B. obere rechte Ecke). Animationen für Erscheinen/Verschwinden.
            - _Funktionalität:_ Anzeige von Icon, App-Name, Zusammenfassung, Body (optional), Aktionen. Automatisches Schließen nach Timeout oder manuell.
            - _Interaktion:_ Reagiert auf `NotificationPostedEvent` von `domain::user_centric_services::NotificationService`. Sendet Aktionen zurück an den Service.
6. **`ui::theming_gtk` (Anwendung des Themes auf GTK-Widgets)**
    
    - **Verantwortlichkeiten:** Anwendung der von `domain::theming::ThemingEngine` generierten CSS-Stile auf alle GTK-Widgets.
    - **Kernfunktionen:**
        - Abrufen des `AppliedThemeState` von der `ThemingEngine`.
        - Generieren einer CSS-Datei aus den `resolved_tokens`.
        - Anwenden dieser CSS-Datei mittels `Gtk::CssProvider` auf den globalen GTK-Kontext (`Gtk::StyleContext::add_provider_for_display`).
        - Abonnieren des `ThemeChangedEvent`, um das CSS dynamisch zu aktualisieren.
    - _Untersuchungsbedarf:_ Effiziente Generierung und Aktualisierung des CSS. Umgang mit GTK4-spezifischen CSS-Eigenschaften und -Selektoren. Sicherstellung, dass benutzerdefinierte Widgets korrekt auf Theme-Änderungen reagieren.
7. **`ui::components` (Sammlung wiederverwendbarer UI-Bausteine)**
    
    - **Verantwortlichkeiten:** Definition und Implementierung anwendungsspezifischer, aber wiederverwendbarer GTK4-Widgets.
    - **Beispiele:**
        - `StyledButtonWidget`: Ein Button, der konsistent mit dem NovaDE-Theming (Akzentfarben, Leuchteffekte) gestaltet ist.
        - `ModalDialogWidget`: Standardisiertes Aussehen und Verhalten für modale Dialoge.
        - `SearchEntryWidget`: Suchfeld mit konsistentem Design und Verhalten.
        - `IconWidget`: Widget zur konsistenten Anzeige von Icons aus dem System-Theme.
    - _Untersuchungsbedarf:_ Identifikation weiterer wiederverwendbarer Komponenten während der Entwicklung.
8. **`ui::speed_dial` (Startansicht für leere Workspaces)**
    
    - **Verantwortlichkeiten:** Implementierung der GTK4-Oberfläche für die Speed-Dial-Funktionalität.
    - **Design:** Kachelbasierte Ansicht mit Favoriten, intelligenten Vorschlägen (basierend auf Nutzungsverhalten, Kontext), integrierte Suchfunktion. Anpassbar.
    - **Interaktion:** Mit `domain::global_settings_and_state_management` (Favoriten, Konfiguration), potenziell mit `domain::user_centric_services::AIInteractionLogicService` für intelligente Vorschläge, Systemschicht (Starten von Apps).
9. **`ui::command_palette` (Kontextuelle Befehlspalette)**
    
    - **Verantwortlichkeiten:** GTK4-Implementierung der Befehlspaletten-UI.
    - **Design:** Overlay-Fenster (typischerweise zentriert oder am oberen Bildschirmrand), das bei Tastenkürzel (`Super+Space`) erscheint. Eingabefeld für Text, dynamische Ergebnisliste mit Fuzzy-Matching.
    - **Funktionalität:** Ausführung von Aktionen, Starten von Apps, Finden von Einstellungen. Kontextsensitivität (zeigt relevante Befehle für aktive App/Desktop-Zustand).
    - **Interaktion:** Mit `domain::global_settings_and_state_management` (Suche nach Einstellungen), Systemschicht (Starten von Apps, Ausführen von Systembefehlen), potenziell `domain::user_centric_services::AIInteractionLogicService` (für KI-gestützte Befehle oder Vorschläge).

**III. Übergreifende Untersuchungsbereiche für die UI-Schicht**

1. **State Management in der UI:**
    - _Problem:_ Wie wird der Zustand von UI-Komponenten verwaltet, insbesondere wenn er von Daten aus tieferen Schichten abhängt oder über mehrere Widgets hinweg synchronisiert werden muss?
    - _Untersuchungsbedarf:_ Evaluation von Rust-spezifischen State-Management-Bibliotheken (falls vorhanden und passend für GTK), Adaption von Patterns wie MVVM oder MVC für GTK4/Rust, Nutzung von GTK-Properties und Bindings, `glib::Object` Subclassing.
2. **Asynchrone Operationen und UI-Responsiveness:**
    - _Problem:_ Wie werden langlaufende Operationen (z.B. Laden von Daten aus dem Netzwerk, komplexe Berechnungen in der Domänenschicht) gehandhabt, ohne die UI zu blockieren?
    - _Untersuchungsbedarf:_ Konsequente Nutzung von `glib::MainContext::spawn_local` für UI-Aktualisierungen aus asynchronen Kontexten. Kommunikation zwischen UI-Thread und Hintergrund-Threads (z.B. über `async_channel` oder `tokio::sync::mpsc`). Design von UI-Feedback für laufende Operationen (Spinner, Fortschrittsbalken).
3. **Teststrategien für die UI:**
    - _Problem:_ Wie kann die Korrektheit und das Verhalten von GTK4-Widgets und der gesamten UI-Logik getestet werden?
    - _Untersuchungsbedarf:_
        - **Unit-Tests:** Testen der Logik von UI-Komponenten, die nicht direkt von GTK-Rendering abhängt. Mocking von Abhängigkeiten zu unteren Schichten.
        - **Widget-Tests:** Nutzung von GTK-Inspektionswerkzeugen und Accessibility-APIs zur programmatischen Überprüfung von Widget-Zuständen und -Eigenschaften. Evaluierung von Frameworks wie `gtk4-rs-test-utils` oder ähnlichen Ansätzen.
        - **Visuelle Regressionstests:** Evaluierung von Werkzeugen für visuelle Vergleiche von UI-Zuständen.
        - **End-to-End-Tests (Integrationstests):** Simulation von Benutzerinteraktionen auf Wayland-Ebene und Überprüfung des Systemverhaltens (herausfordernd, aber wichtig für kritische Pfade).
4. **Performance-Optimierung und Profiling:**
    - _Problem:_ Sicherstellung einer durchgehend performanten UI.
    - _Untersuchungsbedarf:_ Identifikation und Nutzung von Profiling-Werkzeugen für Rust und GTK4 (z.B. `perf`, GTK-Debugger, spezifische Rust-Profiler). Analyse von Rendering-Zeiten, Speicherverbrauch und CPU-Auslastung der UI-Komponenten. Optimierung von Widget-Zeichnung, CSS-Anwendung und Datenbindung.
5. **Internationale und Lokalisierung (i18n/l10n):**
    - _Problem:_ Vorbereitung der UI für die Übersetzung in verschiedene Sprachen.
    - _Untersuchungsbedarf:_ Integration von Gettext oder ähnlichen Bibliotheken mit GTK4/Rust. Management von Übersetzungsdateien. Sicherstellung, dass UI-Layouts mit unterschiedlich langen Texten umgehen können.

**IV. Planungs- und Managementaspekte**

1. **Priorisierung:** Welche UI-Komponenten sind für einen ersten Prototyp/MVP (Minimum Viable Product) unerlässlich?
2. **Abhängigkeiten:** Welche UI-Komponenten hängen stark von der Fertigstellung bestimmter Domänen- oder Systemschichtfunktionen ab?
3. **Parallelisierung:** Welche UI-Module können parallel entwickelt werden?
4. **Design-Prozess:** Enge Zusammenarbeit mit UX/UI-Designern, um Mockups und Prototypen zu erstellen und Iterationen basierend auf Feedback durchzuführen.
5. **Dokumentation:** Umfassende Dokumentation der UI-Komponenten, ihrer APIs und ihrer Interaktionen.

Dieser detaillierte Recherche- und Untersuchungsplan bildet die Grundlage für die erfolgreiche Konzeption und Implementierung der Benutzeroberflächenschicht von NovaDE. Er dient als Leitfaden, um sicherzustellen, dass alle relevanten Aspekte berücksichtigt werden und eine hochwertige Benutzererfahrung entsteht.