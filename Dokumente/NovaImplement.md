# Ultra-Detaillierter Implementierungsplan: Linux-Desktop-Umgebung

## 1. Einleitung

### 1.1. Zweck und Umfang

Dieses Dokument stellt einen detaillierten, schichtweisen Implementierungsplan für das Linux-Desktop-Umgebungsprojekt dar. Es dient als technischer Leitfaden für das Entwicklungsteam und beschreibt die Schritte, Komponenten und Überlegungen, die für die erfolgreiche Umsetzung des Projekts gemäß den (angenommenen) technischen Gesamtspezifikationen und Entwicklungsrichtlinien erforderlich sind. Der Plan deckt alle architektonischen Schichten ab – Core, Domain, System und UI – und geht detailliert auf Aspekte wie API-Definitionen, Fehlerbehandlung, Logging, Tests und Dokumentation ein. Der Umfang dieses Plans umfasst den gesamten Lebenszyklus der Desktop-Umgebungssoftware, von der Initialisierung bis zur Benutzerinteraktion und Systemintegration.

### 1.2. Angenommene Technische Grundlage

Dieser Implementierungsplan basiert auf der Annahme, dass moderne, robuste Technologien als Fundament dienen. Konkret wird davon ausgegangen, dass **Wayland** als Display-Server-Protokoll und **Rust** als primäre Implementierungssprache verwendet werden. Diese Wahl wird durch die Notwendigkeit von Sicherheit, Performance und moderner Architektur getrieben. Rust bietet Speichersicherheit ohne Garbage Collection, was für Systemsoftware wie eine Desktop-Umgebung von entscheidender Bedeutung ist. Wayland bietet gegenüber älteren Systemen wie X11 Vorteile in Bezug auf Sicherheit, Performance und Einfachheit des Protokolls.

Zentral für die Implementierung ist die Nutzung der **Smithay-Bibliothek**.1 Smithay stellt modulare Bausteine für die Erstellung von Wayland-Kompositoren in Rust bereit. Seine Designziele – gut dokumentiert, sicherheitsorientiert, modular und abstrahierend – passen gut zu den Qualitätsanforderungen dieses Projekts.2 Smithay ist explizit keine einschränkende Framework-Lösung, sondern bietet Werkzeuge, die eine flexible Architektur ermöglichen. Dies erfordert jedoch eine sorgfältige Planung der Architektur und der Komponenteninteraktion, wie sie in diesem Dokument dargelegt wird. Die Verwendung von Smithay beschleunigt die Entwicklung, indem sie grundlegende Wayland-Interaktionen und Zustandsverwaltung abstrahiert.1

### 1.3. Architektonische Übersicht

Die Architektur der Desktop-Umgebung ist in vier logische Schichten unterteilt, um eine klare Trennung der Verantwortlichkeiten und eine hohe Modularität zu gewährleisten:

- **Core:** Die unterste Schicht, verantwortlich für den grundlegenden Prozessstart, die Ereignisschleife, die Initialisierung der Wayland-Verbindung und grundlegende Dienstprogramme wie Logging und Fehlerdefinitionen.
- **Domain:** Enthält die Kernlogik und die Zustandsrepräsentation der Desktop-Umgebung. Sie verwaltet abstrakte Konzepte wie Fenster, Arbeitsbereiche, Ausgaben und Eingabegeräte unabhängig von spezifischen Protokollen oder Rendering-Details.
- **System:** Vermittelt zwischen der Domain-Schicht und der Außenwelt. Implementiert Wayland-Protokollschnittstellen, interagiert mit Grafik-Backends und Systemdiensten (z. B. `logind`).
- **UI:** Verantwortlich für die visuelle Darstellung der Desktop-Elemente (Shell-Komponenten, Fensterinhalte) und die Verarbeitung von Benutzerinteraktionen, die die Präsentation direkt beeinflussen.

Die folgenden Abschnitte detaillieren den Implementierungsplan für jede dieser Schichten sowie übergreifende Aspekte wie Fehlerbehandlung, Logging und Tests.

## 2. Gesamtarchitektur

### 2.1. Schichtverantwortlichkeiten

Eine klare Definition der Verantwortlichkeiten jeder Schicht ist entscheidend für eine saubere Architektur und Wartbarkeit:

- **Core:**
    - Initialisierung und Verwaltung der Haupt-Ereignisschleife (`calloop`).
    - Herstellen und Verwalten der Wayland-Display-Verbindung.
    - Grundlegendes Ressourcenmanagement (z. B. Speicher, Dateideskriptoren).
    - Bereitstellung fundamentaler Hilfsprogramme (zentrale Fehler-Typen, Logging-Initialisierung).
- **Domain:**
    - Definition und Verwaltung der internen Zustandsrepräsentation (z. B. Fensterhierarchien, Fokus, Arbeitsbereichszuordnungen, Ausgabekonfigurationen).
    - Implementierung der "Geschäftslogik" der Desktop-Umgebung (z. B. Fensterplatzierungsregeln, Fokuswechsel-Logik, Workspace-Management).
    - Abstraktion über rohe Wayland-Objekte (z. B. ein "Fenster"-Konzept, das eine `wl_surface` mit einer Rolle kapselt).
    - Zustandsverwaltung für Oberflächen (`wl_surface`) unter Verwendung von Smithay-Primitiven.1
- **System:**
    - Implementierung spezifischer Wayland-Protokoll-Handler (z. B. `xdg-shell`, `wlr-layer-shell`, `wl_seat`).
    - Interaktion mit dem Grafik-Subsystem (z. B. EGL, WGPU, DRM/KMS) für Buffer-Management und Rendering-Kontext.
    - Kommunikation mit Systemdiensten (z. B. `logind` für Sitzungsverwaltung und Geräteberechtigungen).2
    - Verarbeitung von Eingabeereignissen vom Backend (z. B. `libinput`) und Weiterleitung an Clients über Wayland-Protokolle.
- **UI:**
    - Implementierung der Rendering-Pipeline (Zeichnen von Fenstern, Shell-Elementen).
    - Implementierung von Damage Tracking zur Optimierung des Renderings.3
    - Entwicklung und Darstellung von Desktop-Shell-Komponenten (Panel, Launcher, Benachrichtigungen) unter Verwendung von Protokollen wie `wlr-layer-shell`.4
    - Verarbeitung direkter UI-Interaktionen (z. B. Klicks auf Buttons im Panel).
    - Anwendung von Theming und visuellen Stilen.

### 2.2. Komponenteninteraktion

Die Schichten interagieren auf klar definierten Wegen. Ein typischer Ablauf könnte wie folgt aussehen:

1. Ein Hardware-Ereignis (z. B. Tastendruck) wird vom System-Backend (z. B. `libinput`) erfasst.
2. Die Core-Ereignisschleife nimmt das Ereignis entgegen und leitet es an den entsprechenden Handler in der System-Schicht weiter.
3. Die System-Schicht (z. B. `wl_seat`-Handler) interpretiert das Ereignis im Kontext des Wayland-Protokolls.
4. Die System-Schicht interagiert mit der Domain-Schicht, um den relevanten Zustand zu ermitteln (z. B. welches Fenster hat den Fokus?).
5. Basierend auf dem Zustand entscheidet die System-Schicht, das Ereignis an einen Wayland-Client zu senden oder eine Aktion in der Domain-Schicht auszulösen (z. B. Fokuswechsel anfordern).
6. Wenn eine Zustandsänderung in der Domain-Schicht erfolgt (z. B. Fokus wechselt), wird dies registriert.
7. Die UI-Schicht wird über die Zustandsänderung informiert (direkt oder indirekt durch die Notwendigkeit eines Neuzeichnens).
8. Die UI-Schicht rendert die betroffenen Bereiche neu, möglicherweise unter Verwendung von Damage Tracking, um nur die geänderten Teile zu aktualisieren.3 Sie greift dabei auf Zustandsinformationen aus der Domain-Schicht (z. B. Fensterpositionen, Inhalte) und System-Schicht (z. B. Buffer von Clients) zu.

### 2.3. Wichtige Architekturprinzipien

Die Entwicklung folgt diesen Leitprinzipien:

- **Modularität:** Aufbauend auf Smithays Philosophie 2, werden Komponenten so entworfen, dass sie möglichst unabhängig und austauschbar sind.
- **Trennung der Verantwortlichkeiten (Separation of Concerns):** Strikte Einhaltung der Schichtgrenzen, um Abhängigkeiten zu minimieren und die Testbarkeit zu erhöhen.
- **Testbarkeit:** Design für Testbarkeit auf allen Ebenen (Unit, Integration, E2E).
- **Robustheit:** Umfassende Fehlerbehandlung, sicheres Ressourcenmanagement (unterstützt durch Rust) und graceful degradation bei Fehlern.
- **Performance:** Effiziente Algorithmen, insbesondere im Rendering-Pfad (z. B. Damage Tracking 3) und bei der Ereignisverarbeitung.

### 2.4. _Wertvolle Tabelle:_ Matrix der Schichtverantwortlichkeiten

Die folgende Matrix bietet eine Übersicht, welche Schicht primär (P) oder unterstützend (S) für wichtige Funktionalitäten verantwortlich ist.

|   |   |   |   |   |
|---|---|---|---|---|
|**Funktionalität**|**Core**|**Domain**|**System**|**UI**|
|Ereignisverarbeitung (Loop)|P||S||
|Wayland-Verbindung|P||S||
|Zustandsverwaltung (intern)||P|S|S|
|Fenstergeometrie/-logik||P|S|S|
|Workspace-Management||P|S|S|
|Wayland-Protokoll-Impl.|S|S|P||
|Grafik-Backend-Interaktion|S||P|S|
|Rendering||S|S|P|
|Damage Tracking||S|S|P|
|Eingabeverarbeitung (System)|S|S|P||
|Eingabeverarbeitung (Client)||S|P||
|Shell-Komponenten (Logik)||S|S|P|
|Shell-Komponenten (Darst.)|||S|P|
|Konfiguration (Speicher)|S|P|S||
|Konfiguration (Anwendung)||P|S|P|
|Logging|P|S|S|S|
|Fehlerbehandlung (Kern)|P|S|S|S|
|Fehlerbehandlung (Protokoll)||S|P||

Diese Matrix dient als Referenz, um sicherzustellen, dass die Logik in der korrekten Schicht implementiert wird und die Prinzipien der Modularität und Trennung der Verantwortlichkeiten eingehalten werden.

## 3. Core Layer Implementierung

### 3.1. Event Loop Integration (`calloop`)

Die zentrale Ereignisverarbeitung wird mittels der `calloop`-Bibliothek implementiert.5 `calloop` ist eine auf Rückrufen basierende Ereignisschleife, die sich gut für I/O-gebundene Anwendungen wie einen Wayland-Kompositor eignet, der die meiste Zeit auf Ereignisse wartet.

Die Initialisierung erfolgt typischerweise durch `EventLoop::try_new()`. Ein `handle` wird verwendet, um neue Ereignisquellen hinzuzufügen (`handle.insert_source()`). Jede Quelle wird mit einem Callback assoziiert, der bei Eintreten von Ereignissen ausgeführt wird. Die `event_loop.run()`-Methode startet die Schleife und verarbeitet Ereignisse. Sie ermöglicht die Übergabe gemeinsamer Daten (`shared_data`), auf die Callbacks zugreifen können, beispielsweise ein `LoopSignal` zum Beenden der Schleife.5

Eine besondere Herausforderung stellt die Integration von Wayland-Ereignisquellen dar. Der Dateideskriptor der Wayland-Display-Verbindung muss in die `calloop`-Schleife integriert werden. Es ist jedoch entscheidend, die Verwaltung der Wayland-Ereigniswarteschlangen korrekt zu handhaben, insbesondere wenn mehrere Komponenten oder Threads mit der Wayland-Verbindung interagieren könnten (z. B. durch Bibliotheken wie `winit` oder separate Threads für Aufgaben wie die Zwischenablage).6 Die Wayland-Client-Bibliothek erfordert, dass `wl_display_prepare_read_queue` aufgerufen wird, _bevor_ die Ereignisschleife blockiert (z. B. mittels `epoll_wait`), um Race Conditions zu vermeiden und sicherzustellen, dass keine Ereignisse verloren gehen oder falsch verarbeitet werden.6 Ein einfaches Hinzufügen des Wayland-FDs zu `calloop` ist möglicherweise nicht ausreichend, wenn die Warteschlange auch von anderer Stelle beeinflusst werden kann.

Daher muss eine klare Strategie für die Verwaltung der Wayland-Warteschlange definiert werden. Mögliche Ansätze sind:

a) Sicherstellen, dass alle Wayland-Interaktionen ausschließlich über den Haupt-Thread der calloop-Schleife und die von ihr verwalteten Quellen erfolgen.

b) Sorgfältige manuelle Verwendung von wl_display_prepare_read_queue, falls eine tiefere Integration notwendig ist.

c) Isolation von Komponenten mit komplexen Wayland-Interaktionen (wie der in 6 erwähnten Zwischenablage) in separate Threads mit eigenen, sorgfältig verwalteten Verbindungen/Warteschlangen, die über in calloop integrierte Kanäle (z. B. MPSC-Kanäle 5) mit der Hauptschleife kommunizieren.

Diese architektonische Entscheidung hat erhebliche Auswirkungen auf die Komplexität, Performance und das Potenzial für Deadlocks und muss frühzeitig getroffen werden.

### 3.2. Wayland Display & Core Initialisierung

Der Prozess beginnt mit dem Aufbau der Verbindung zum Wayland-Display. Dies kann entweder als verschachtelter Client innerhalb einer bestehenden Wayland- oder X11-Sitzung geschehen (nützlich für Entwicklung und Tests 2) oder direkt auf der Hardware über ein TTY mit DRM/KMS und `udev`-Unterstützung.2

Nach erfolgreicher Verbindung werden die globalen Wayland-Objekte initialisiert. Eine zentrale `State`-Struktur wird definiert, die alle kompositorweiten Ressourcen und Zustände enthält. Hierzu gehört auch die Initialisierung von Smithays `CompositorState`, wie im Beispiel in 1 gezeigt, um die Basis für die Verwaltung von Oberflächen zu legen.

### 3.3. Fundamentales Logging

Ein robustes Logging-System ist von Anfang an unerlässlich. Es wird empfohlen, die `tracing`-Bibliothek oder die `log`-Fassade mit einem geeigneten Backend wie `env_logger` 7 zu verwenden.

Zu Beginn werden grundlegende Logging-Levels (z. B. ERROR, WARN, INFO, DEBUG, TRACE) definiert. Log-Nachrichten sollten standardmäßig Zeitstempel, Modulpfad und Schweregrad enthalten. Wo immer sinnvoll, sollte strukturiertes Logging (z. B. Key-Value-Paare) verwendet werden, um die spätere Analyse und Filterung von Logs zu erleichtern. Die Konfiguration der Log-Levels sollte zur Laufzeit möglich sein (z. B. über Umgebungsvariablen).

### 3.4. Grundlegende Fehlerbehandlung

Ein konsistentes Fehlerbehandlungssystem wird etabliert. Es wird empfohlen, `thiserror` oder eine benutzerdefinierte `enum`-Struktur zu verwenden, um spezifische Fehlertypen für die Core-Schicht (`CoreError`) und potenziell andere Schichten zu definieren. Die Konvention für die Fehlerweitergabe ist `Result<T, ErrorType>`.

Ein grundlegender Panic-Handler wird implementiert, um bei unerwarteten Fehlern aussagekräftige Informationen zu loggen oder anzuzeigen, anstatt stillschweigend abzustürzen. Rusts Ownership- und Borrowing-System hilft, viele Fehlerklassen zur Kompilierzeit zu vermeiden, aber Laufzeitfehler (z. B. I/O-Fehler, Protokollverletzungen durch Clients) müssen explizit behandelt werden.

## 4. Domain Layer Implementierung

### 4.1. Compositor State Management

Die Verwaltung des Zustands von Wayland-Oberflächen (`wl_surface`) ist eine Kernaufgabe des Kompositors. Smithays `CompositorState` und das zugehörige `CompositorHandler`-Trait werden hierfür zentral genutzt.1 Diese Komponenten übernehmen die automatische Verwaltung von Oberflächen, Unteroberflächen (`subsurfaces`) und Regionen, speichern den Zustand kohärent und behandeln die Anwendung von doppelt gepufferten Zuständen.

Der Zugriff auf den Zustand einer Oberfläche erfolgt über die Funktion `with_states`, die Zugriff auf die `SurfaceData`-Instanz der Oberfläche gewährt. `SurfaceData` dient als allgemeiner Container, um anwendungsspezifische Daten (gepuffert oder nicht) mit einer Oberfläche zu assoziieren.1

Der Commit-Lebenszyklus einer Oberfläche wird durch Hooks gesteuert:

- `add_pre_commit_hook`: Wird vor der Anwendung des neuen Zustands aufgerufen. Nützlich für Protokollerweiterungen, um ungültige Zustandsanfragen von Clients zu validieren, bevor sie angewendet werden.
- `add_post_commit_hook`: Wird nach der Anwendung des neuen Zustands aufgerufen. Nützlich für Abstraktionen, die den neuen Zustand weiterverarbeiten müssen.
- `add_destruction_hook`: Wird aufgerufen, wenn die Oberfläche zerstört wird, um zugeordnete Zustände aufzuräumen.1

Die Implementierung von `CompositorHandler::commit` ist der zentrale Punkt, an dem die Domain-Logik auf den neuen, aktuellen Zustand einer Oberfläche reagieren kann, nachdem dieser (und der Zustand synchronisierter Kind-Unteroberflächen) angewendet wurde.1

Wayland verlangt, dass jede darstellbare Oberfläche eine "Rolle" zugewiesen bekommt (z. B. `toplevel`, `popup`, `subsurface`, `layer-surface`). Smithay repräsentiert dies durch einen statischen String-Identifier, der nur einmal pro Oberfläche gesetzt werden kann (`give_role`, `get_role`).1 Die Domain-Schicht ist verantwortlich für die Verwaltung dieser Rollen und die Sicherstellung, dass sie korrekt und konsistent verwendet werden.

### 4.2. Window Management Abstraction

Während Smithay die Mechanismen zur Verwaltung von `wl_surface`-Zuständen bereitstellt 1, benötigt die Desktop-Umgebung höherstufige Konzepte wie "Fenster", "Fokus" und "Stapelanordnung".3 Es ist daher notwendig, innerhalb der Domain-Schicht eine explizite Abstraktionsebene für das Fenstermanagement zu implementieren. Diese Abstraktion kapselt typischerweise eine Wurzel-`wl_surface` mit einer bestimmten Rolle (z. B. `xdg_toplevel`) und verwaltet die damit verbundenen Metadaten und Verhaltensweisen.

Diese Abstraktion ist verantwortlich für:

- **Geometrieverwaltung:** Verfolgung und Verwaltung der Position und Größe von Fenstern im logischen Koordinatensystem des Desktops.3 Dies beinhaltet auch die Handhabung von Geometrie-Constraints, wie z. B. minimale/maximale Größen oder spezielle Begrenzungsrahmen für Popups (möglicherweise pro Ausgabe, wie in 3 vorgeschlagen, um Popups auf einem Monitor zu halten).
- **Stapelanordnung (Z-Order):** Implementierung der Logik zur Verwaltung der Sichtbarkeitsreihenfolge von Fenstern. Dies ist entscheidend für das korrekte Rendering, bei dem Fenster (und ihre Unteroberflächen und Popups) in der richtigen Z-Reihenfolge durchlaufen werden müssen. Optimierungen wie das Überspringen von Oberflächen, die vollständig von anderen opaken Oberflächen verdeckt werden, sollten hier berücksichtigt werden.3
- **Fokusmanagement:** Verfolgung des aktiven Fensters (das Eingabeereignisse empfängt) und Implementierung der Logik für Fokuswechsel (z. B. durch Benutzeraktion oder programmatisch).

Die Schaffung dieser internen Abstraktion ist kritisch. Sie entkoppelt die höhere Fensterverwaltungslogik von den Details der `wl_surface`-Commit-Hooks und der Baumstruktur von Unteroberflächen.1 Ohne diese Abstraktion besteht die Gefahr, dass sich Low-Level-Oberflächenbehandlung und High-Level-Fensterlogik vermischen, was zu schwer wartbarem Code führt. Die API dieser Abstraktion muss klar definiert sein, um eine saubere Interaktion mit der System- und UI-Schicht zu ermöglichen.

### 4.3. Input Abstraction

Die Domain-Schicht verwaltet den _Zustand_ der Eingabegeräte, während die System-Schicht die _Protokoll_-Interaktion übernimmt. Hier werden interne Repräsentationen für Tastaturen, Zeigergeräte (Mäuse, Touchpads) und Touchscreens definiert.

Diese Schicht ist verantwortlich für die Verwaltung von:

- Gerätefähigkeiten (z. B. Anzahl der Maustasten, Touchscreen-Funktionen).
- Gerätezuständen (z. B. aktuell aktives Tastaturlayout, Zeigerbeschleunigungseinstellungen, Tastenzustände wie Caps Lock).
- Zuordnung von Geräten zu "Seats" (logische Gruppierungen von Eingabe- und Ausgabegeräten für einen Benutzer).

### 4.4. Output Configuration

Analog zur Eingabeabstraktion verwaltet die Domain-Schicht den internen Zustand der angeschlossenen Ausgabegeräte (Monitore).

Verantwortlichkeiten umfassen:

- Definition interner Repräsentationen für Ausgaben.
- Verwaltung von Ausgabeeigenschaften: Auflösung (Mode), Position im globalen Koordinatenraum, Skalierungsfaktor, Aktivierungszustand (enabled/disabled), Orientierung.
- Logik zur Anordnung mehrerer Monitore.

Diese Zustandsinformationen werden von der System-Schicht verwendet, um das `wl_output`-Protokoll zu implementieren und von der UI-Schicht für das Rendering.

### 4.5. Workspace/Virtual Desktop Logic

Falls das Desktop-Paradigma virtuelle Desktops oder Workspaces vorsieht, wird die entsprechende Logik in der Domain-Schicht implementiert.

Dies beinhaltet:

- Datenstrukturen zur Darstellung von Workspaces und deren Zuordnung zu Ausgaben.
- Logik zur Verwaltung der Zugehörigkeit von Fenstern zu Workspaces.
- Implementierung von Workspace-Wechsel-Aktionen und deren Auswirkungen auf Fenster-Sichtbarkeit und Fokus.

## 5. System Layer Implementierung

### 5.1. Wayland Protocol Implementation

Diese Schicht ist maßgeblich für die Implementierung der verschiedenen Wayland-Protokolle verantwortlich, die die Kommunikation zwischen dem Kompositor und den Client-Anwendungen ermöglichen. Smithay bietet hierfür zahlreiche Handler und Hilfsprogramme.

- **`wl_compositor` / `wl_subcompositor`:** Die grundlegende Funktionalität wird weitgehend durch Smithays `CompositorState` bereitgestellt.1 Die Integration in die Haupt-`State`-Struktur und die Ereignisschleife ist jedoch erforderlich.
- **Shell Protocols (`xdg-shell`):** Dies ist eines der wichtigsten Protokolle für Desktop-Anwendungen. Es müssen Handler für `xdg_wm_base` (das globale Objekt), `xdg_surface` (bindet eine `wl_surface` an `xdg-shell`), `xdg_toplevel` (für Hauptfenster) und `xdg_popup` (für Menüs, Tooltips etc.) implementiert werden. Anfragen von Clients (z. B. `move`, `resize`, `set_title`, `set_maximized`, `set_fullscreen`) müssen empfangen und in Aktionen auf der Fenstermanagement-Abstraktion der Domain-Schicht übersetzt werden. Ebenso müssen Konfigurationsereignisse (`configure`) an die Clients gesendet und deren Bestätigungen (`ack_configure`) verarbeitet werden.
- **Decoration Protocol (`xdg-decoration`):** Dieses Protokoll dient der Aushandlung zwischen Client und Kompositor, ob Fensterdekorationen vom Client (Client-Side Decorations, CSD) oder vom Kompositor (Server-Side Decorations, SSD) gezeichnet werden sollen.
    - Die Implementierung erfolgt durch Bereitstellung des `zxdg_decoration_manager_v1`-Globals mithilfe von `XdgDecorationState`.8
    - Handler für `zxdg_toplevel_decoration_v1`-Anfragen wie `set_mode` und `unset_mode` müssen implementiert werden.
    - Es ist von entscheidender Bedeutung, die _Semantik_ des Protokolls korrekt umzusetzen. Wie in 9 am Beispiel von SDL dargelegt, dient das Protokoll der _Aushandlung_, nicht ausschließlich der Signalisierung von SSD. Clients könnten fälschlicherweise annehmen, dass SSD verfügbar ist, nur weil der Manager existiert. Der Kompositor muss klar seine bevorzugte Dekorationsart (oder die Fähigkeit zum Wechsel) signalisieren und auf Client-Anfragen entsprechend reagieren. Es muss entschieden werden, ob der Kompositor nur CSD, nur SSD oder beides (ggf. dynamisch umschaltbar) unterstützt, und die Protokoll-Implementierung muss dies widerspiegeln. Die Implementierung muss robust gegenüber potenziell missverständlichen Client-Anfragen sein und die Teststrategie (Abschnitt 7.4) muss die Korrektheit dieser Aushandlung mit verschiedenen Clients überprüfen.
- **Layer Shell Protocol (`wlr-layer-shell-unstable-v1`):** Dieses Protokoll ermöglicht es Clients, Oberflächen zu erstellen, die als Teil der Desktop-Shell fungieren (z. B. Panels, Hintergrundbilder, Benachrichtigungs-Popups, Lock Screens).4
    - Das `zwlr_layer_shell_v1`-Global und die `zwlr_layer_surface_v1`-Schnittstelle müssen implementiert werden.
    - Anfragen wie `get_layer_surface`, `set_layer` (Hintergrund, Unten, Oben, Überlagerung), `set_anchor` (Anheften an Bildschirmkanten), `set_exclusive_zone` (Reservieren von Platz, der von normalen Fenstern freigehalten wird) und `set_keyboard_interactivity` müssen verarbeitet werden.4
    - Der Konfigurationszyklus (initialer Commit ohne Buffer, `configure`-Ereignis vom Kompositor, `ack_configure` vom Client, dann Buffer anhängen) muss korrekt implementiert werden.4
    - Die Eigenschaften dieser Layer-Oberflächen müssen in der Domain-Schicht repräsentiert und in die Rendering-Pipeline der UI-Schicht integriert werden. Die Möglichkeit, `xdg_popup`s relativ zu Layer-Oberflächen zu positionieren (`get_popup`), muss ebenfalls berücksichtigt werden.4
- **Seat & Input Protocols (`wl_seat`, `wl_keyboard`, `wl_pointer`, `wl_touch`):** Das `wl_seat`-Global repräsentiert eine Gruppe von Ein- und Ausgabegeräten für einen Benutzer. Zugehörige Schnittstellen (`wl_keyboard`, `wl_pointer`, etc.) dienen der Übermittlung von Eingabeereignissen an fokussierte Clients. Die System-Schicht empfängt rohe Eingabeereignisse vom Backend (z. B. `libinput`), verarbeitet sie gemäß dem Zustand in der Domain-Schicht (z. B. Fokus, Tastaturlayout) und sendet die entsprechenden Wayland-Ereignisse an die Clients. Die Verwaltung von Tastatur-Keymaps erfordert typischerweise die `libxkbcommon`-Bibliothek.7
- **Output Protocol (`wl_output`):** Das `wl_output`-Global repräsentiert einen Monitor. Die System-Schicht implementiert dieses Global und sendet Ereignisse über Geometrie (`geometry`), verfügbare Modi (`mode`), Skalierungsfaktor (`scale`) und Abschluss der Konfiguration (`done`) an die Clients, basierend auf dem Zustand, der in der Domain-Schicht verwaltet wird.
- **Andere Protokolle:** Je nach Anforderung müssen weitere Protokolle implementiert werden, z. B.:
    - Datenübertragung (`wl_data_device_manager` für Drag-and-Drop und Copy-Paste).
    - Primäre Auswahl (`gtk_primary_selection_device_manager`).
    - Präsentationszeit (`wp_presentation` für Frame-Callbacks und Timing).
    - Bildschirmaufnahme (`xdg-desktop-portal` oder spezifische Protokolle wie `wlr-screencopy-unstable-v1`).

### 5.2. _Wertvolle Tabelle:_ Wayland-Protokoll-Implementierungsplan

Diese Tabelle dient der Nachverfolgung des Implementierungsstatus und wichtiger Aspekte der einzelnen Protokolle.

|   |   |   |   |
|---|---|---|---|
|**Protokoll/Interface**|**Status**|**Smithay Handler/Util**|**Wichtige Überlegungen**|
|`wl_compositor`|Benötigt|`CompositorState`, `CompositorHandler`|Basis für Oberflächenverwaltung, von Smithay weitgehend abgedeckt.1|
|`wl_subcompositor`|Benötigt|`CompositorState`|Verwaltung von Unteroberflächen, Teil von `CompositorState`.1|
|`xdg_wm_base`|Benötigt|`XdgShellState`, `XdgShellHandler`|Zentral für Desktop-Anwendungen; komplexe Zustandsverwaltung (Konfiguration, Rollen).|
|`xdg_surface`, `xdg_toplevel`|Benötigt|`XdgShellState`, `ToplevelSurface`|Handling von Fenster-Requests (move, resize, maximize etc.), Konfigurationszyklus.|
|`xdg_popup`|Benötigt|`XdgShellState`, `PopupSurface`|Positionierung relativ zu Eltern, Greifen von Eingaben.|
|`zxdg_decoration_manager_v1`|Benötigt|`XdgDecorationState` 8|Korrekte Implementierung der CSD/SSD-Aushandlung ist kritisch, Client-Verhalten beachten.9|
|`zwlr_layer_shell_v1`|Benötigt|`LayerShellState` (oder Äquivalent)|Für Shell-UI-Elemente; Konfigurationszyklus, `exclusive_zone`, `anchor` beachten.4|
|`wl_seat`|Benötigt|`SeatState`, `SeatHandler`|Verwaltung von Fähigkeiten (Tastatur, Zeiger, Touch) und Fokus.|
|`wl_keyboard`|Benötigt|`KeyboardHandle`|Keymap-Handling (via `libxkbcommon` 7), Senden von Tastenereignissen.|
|`wl_pointer`|Benötigt|`PointerHandle`|Senden von Bewegungs-, Tasten-, Achsenereignissen; Cursor-Management.|
|`wl_touch`|Optional|`TouchHandle`|Senden von Touch-Ereignissen (down, up, motion).|
|`wl_output`|Benötigt|`OutputManagerState` (oder Äquiv.)|Senden von Geometrie, Modus, Skalierung an Clients.|
|`wl_data_device_manager`|Empfohlen|`DataDeviceState`|Für Copy-Paste und Drag-and-Drop.|
|`wp_presentation`|Empfohlen|`PresentationState` (oder Äquiv.)|Für präzise Frame-Callbacks und Timing-Informationen.|
|`wlr-screencopy-unstable-v1`|Optional|Eigene Implementierung|Für Bildschirmaufnahmen/Screenshots.|

_Status: Benötigt, Empfohlen, Optional, In Arbeit, Implementiert, Getestet_

### 5.3. Graphics Backend Abstraction

Die System-Schicht muss mit dem Grafik-Subsystem des Hosts interagieren, um Fensterinhalte darzustellen. Eine Abstraktionsschicht ist hier sinnvoll, um die Kopplung an eine spezifische Grafik-API zu minimieren.

Mögliche Backends sind:

- **EGL + OpenGL ES:** Der traditionelle Ansatz für viele Wayland-Kompositoren.
- **WGPU:** Eine moderne Grafikabstraktion in Rust, die über verschiedene native Backends (Vulkan, Metal, DirectX 12, OpenGL ES) laufen kann. Die Verwendung von `wgpu` wird in der Wayland/Smithay-Community untersucht und eingesetzt.7

Die Abstraktion muss folgende Aufgaben übernehmen:

- **Backend-Initialisierung:** Laden der Grafiktreiber, Erstellen von Kontexten (EGL-Kontext, WGPU-Instanz/Device).
- **Buffer-Management:** Handhabung von Client-bereitgestellten Buffern über `wl_buffer`. Dies umfasst Shared Memory Buffers (SHM) und idealerweise DMA-BUFs für effiziente, kopierfreie Übergabe von Grafikdaten von Clients (insbesondere GPUs) an den Kompositor.
- **Rendering-Synchronisation:** Sicherstellen, dass Frames zum richtigen Zeitpunkt (z. B. VSync) auf dem Bildschirm angezeigt werden.
- **Integration mit DRM/KMS:** Bei Ausführung direkt auf der Hardware (`--tty-udev`-Backend 2) ist eine direkte Interaktion mit dem Direct Rendering Manager (DRM) und Kernel Mode Setting (KMS) zur Ansteuerung der Monitore erforderlich. Smithay bietet hierfür Hilfsmittel.

Die Wahl des Backends hat Auswirkungen auf die Systemanforderungen und die Komplexität der Implementierung. WGPU bietet potenziell breitere Kompatibilität und modernere Features, kann aber auch komplexer in der Handhabung sein.

### 5.4. Session & Permissions Integration

Wenn der Kompositor direkt auf der Hardware läuft (nicht verschachtelt), benötigt er spezielle Berechtigungen für den Zugriff auf Eingabe- und Ausgabegeräte (DRM/KMS, `libinput`). Diese Berechtigungen werden typischerweise über einen Session Manager wie `logind` (Teil von `systemd`) oder `seatd` (eine leichtere Alternative) verwaltet.

Die System-Schicht muss:

- Mit dem aktiven Session Manager über dessen D-Bus-Schnittstelle oder Client-Bibliothek kommunizieren.
- Geräte-Dateideskriptoren sicher öffnen und Berechtigungen wieder freigeben, wenn der Kompositor beendet wird oder die Sitzung wechselt.
- Auf Ereignisse wie Geräte-Hotplugging (An-/Abstecken von Monitoren, Mäusen etc.) und Sitzungswechsel reagieren.

Smithay bietet Unterstützung für die Integration mit `libinput` und DRM/KMS, oft in Verbindung mit `logind` oder `seatd`. Die korrekte Handhabung von Berechtigungen ist entscheidend für die Sicherheit und Stabilität des Systems.2

## 6. UI Layer Implementierung

### 6.1. Rendering Pipeline

Die UI-Schicht ist verantwortlich für das tatsächliche Zeichnen jedes Frames auf dem Bildschirm. Die Rendering-Pipeline umfasst typischerweise folgende Schritte:

1. **Szenenbeschreibung erstellen:** Basierend auf dem Zustand in der Domain-Schicht (Fensterpositionen, Z-Order, Sichtbarkeit, Workspace) und System-Schicht (verfügbare Client-Buffer) wird eine Liste der darzustellenden Elemente erstellt.
2. **Oberflächen durchlaufen:** Iteration durch die sichtbaren Oberflächen (Fenster, Layer-Shell-Elemente, Popups, Unteroberflächen) in der korrekten Z-Reihenfolge (von hinten nach vorne).3
3. **Oberflächeninhalte abrufen:** Zugriff auf die Texturen oder Pixeldaten der Oberflächen. Dies können SHM-Buffer sein, die auf die GPU hochgeladen werden müssen, oder direkt verwendbare DMA-BUF-Texturen.
4. **Komposition:** Zeichnen der Oberflächeninhalte auf den Ziel-Framebuffer (des Bildschirms). Dies beinhaltet Transformationen (Positionierung, Skalierung), Clipping und Blending (für Transparenz). Opake Regionen von Oberflächen können genutzt werden, um das Zeichnen dahinterliegender Oberflächen zu überspringen.3
5. **Shell-Elemente zeichnen:** Rendern von UI-Komponenten wie Panels, Cursor usw.
6. **Frame einreichen:** Übergabe des fertigen Frames an das Grafik-Backend zur Anzeige.

**Damage Tracking:** Ein entscheidender Aspekt für die Performance ist das Damage Tracking. Anstatt jeden Frame komplett neu zu zeichnen, sollten nur die Bereiche des Bildschirms aktualisiert werden, die sich seit dem letzten Frame geändert haben.3 Dies reduziert die GPU-Last und den Energieverbrauch erheblich.

Die Implementierung von Damage Tracking ist jedoch komplex:

- Es muss nicht nur der Inhalt von Oberflächen verfolgt werden (was Smithay teilweise für SHM-Buffer unterstützt), sondern auch Änderungen an der Szene selbst: Fensterbewegungen, Größenänderungen, das Erscheinen oder Verschwinden von Fenstern/Oberflächen.3
- Ein möglicher Algorithmus (angelehnt an 3):
    1. Sammle alle "beschädigten" Rechtecke auf dem Bildschirm: die Bounding Box von neuen Fenstern, die _alte_ Bounding Box von entfernten Fenstern, die _alte und neue_ Bounding Box von bewegten/skalierten Fenstern, und die von Clients gemeldeten Schadensregionen auf Oberflächen (übersetzt in globale Koordinaten).
    2. Optimiere optional die Liste der Schadensrechtecke (z. B. durch Zusammenfassen überlappender Rechtecke).
    3. Für jedes Schadensrechteck: Ermittle alle Oberflächen, die dieses Rechteck überlappen. Zeichne die relevanten Teile dieser Oberflächen (unter Berücksichtigung der Z-Order und opaker Regionen) innerhalb des Schadensrechtecks neu.

Die Implementierung von effektivem Damage Tracking erfordert sorgfältige Zustandsverwaltung (Speichern des vorherigen Frame-Zustands) und geometrische Berechnungen. Es ist keine triviale Optimierung, sondern ein Kernmerkmal für eine performante Desktop-Umgebung. Es sollte als eigenständiges, komplexes Teilprojekt betrachtet werden, das iterative Entwicklung, Tests und Performance-Profiling erfordert.

### 6.2. Desktop Shell Components

Die UI-Schicht implementiert die sichtbaren Komponenten der Desktop-Shell. Dazu gehören typischerweise:

- **Panel/Taskleiste:** Anzeige von laufenden Anwendungen, System-Tray, Uhrzeit, Menüs.
- **Application Launcher:** Menü oder Übersicht zum Starten von Anwendungen.
- **Notification System:** Anzeige von Benachrichtigungen von Anwendungen.
- **Background Manager:** Anzeige und Verwaltung des Desktophintergrunds.
- **Lock Screen:** Bildschirmsperre bei Inaktivität oder auf Anforderung.
- **Cursor:** Darstellung des Mauszeigers.

Diese Komponenten werden oft als spezielle Wayland-Clients implementiert, die das `wlr-layer-shell-unstable-v1`-Protokoll verwenden, um sich korrekt auf dem Bildschirm zu positionieren (z. B. Panel am oberen Rand), Platz zu reservieren (`exclusive_zone`) und Eingabeverhalten zu steuern.4 Alternativ könnten einige Komponenten direkt im Kompositor-Prozess gerendert werden, was jedoch die Modularität verringern kann. Die Wahl hängt von der gewünschten Architektur ab. Bestehende Werkzeuge oder Bibliotheken aus dem Ökosystem (z. B. Konzepte aus `wlr-screen-fade` 7 für Übergänge) könnten als Inspiration dienen oder wiederverwendet werden.

### 6.3. Client Toolkit Integration (Optional)

Wenn die Shell-Komponenten selbst mit einem UI-Toolkit (wie GTK, Qt oder einem Rust-basierten Toolkit) erstellt werden, muss die Integration dieses Toolkits mit Wayland berücksichtigt werden. Wenn diese Komponenten als separate Prozesse laufen, agieren sie als normale Wayland-Clients. Das `smithay-client-toolkit` 7 könnte relevant sein, wenn UI-Komponenten ebenfalls in Rust geschrieben werden und Wayland-Client-Funktionalität benötigen.

### 6.4. Theming and Configuration API

Die UI-Schicht muss Mechanismen zur Anpassung des Erscheinungsbilds und Verhaltens bereitstellen:

- **Theming:** Laden und Anwenden von visuellen Themen (Icons, Cursor-Themes, Stile für Shell-Komponenten). Dies erfordert definierte Schnittstellen und Konventionen für Theme-Dateien.
- **Konfiguration:** Eine API oder ein Mechanismus (z. B. D-Bus-Schnittstelle, Konfigurationsdateien), über den Benutzer oder Einstellungs-Tools das Verhalten der Desktop-Umgebung anpassen können (z. B. Tastenkürzel, Erscheinungsbild, Ausgabeeinstellungen, Panel-Konfiguration). Der Zustand dieser Konfiguration wird typischerweise in der Domain-Schicht gehalten, während die UI-Schicht ihn anwendet und ggf. Einstellungsdialoge bereitstellt.

## 7. Cross-Cutting Concerns Implementation Strategy

Diese Aspekte betreffen alle Schichten und erfordern eine konsistente Strategie.

### 7.1. API Design & Documentation

- **Interne APIs:** Es werden strenge Richtlinien für das Design interner Rust-APIs festgelegt (Modulstruktur, Namenskonventionen, Nutzung des Typsystems zur Sicherstellung von Invarianten, Sichtbarkeitsregeln).
- **Dokumentation:** Umfassende Dokumentation aller öffentlichen APIs, Module und komplexen Funktionen mittels `rustdoc` ist obligatorisch. Besonderes Augenmerk liegt auf der Dokumentation der Zustandsverwaltung, der erwarteten Invarianten, der Thread-Sicherheit (falls zutreffend) und potenzieller Fehlerbedingungen. Das Ziel ist eine Dokumentationsqualität ähnlich der von Smithay selbst.2

### 7.2. Error Handling Strategy

Aufbauend auf den grundlegenden Fehlertypen der Core-Schicht wird eine detaillierte Strategie definiert:

- **Fehler-Typen:** Spezifische Fehlertypen für jede Schicht oder Hauptkomponente können definiert werden, um den Ursprung von Fehlern besser zu lokalisieren.
- **Fehlerweitergabe:** Klare Regeln für die Weitergabe von Fehlern über Schichtgrenzen hinweg. Interne Fehler sollten geloggt werden, während Protokollfehler (durch fehlerhaftes Client-Verhalten) ggf. zum Schließen der Client-Verbindung führen (`wl_display.error`).
- **Graceful Degradation:** Wo möglich, sollte das System versuchen, bei Fehlern in nicht-kritischen Komponenten weiterzuarbeiten.
- **Ressourcenbereinigung:** Robuste Mechanismen (unterstützt durch Rusts RAII) zur Freigabe von Ressourcen (Speicher, Dateideskriptoren, Wayland-Objekte) im Fehlerfall sind essenziell.

### 7.3. Structured Logging Implementation

Das grundlegende Logging wird erweitert:

- **Strukturiertes Logging:** Konsequente Verwendung von strukturiertem Logging (z. B. mit `tracing` und Feldern), um Logs maschinenlesbar und leichter analysierbar zu machen. Wichtige Identifikatoren (z. B. Client-ID, Surface-ID) sollten in den Logs enthalten sein.
- **Log-Punkte:** Definierte Log-Punkte für kritische Ereignisse (z. B. Start/Ende von Rendering-Phasen, Fokuswechsel, Fenstererstellung/-zerstörung, wichtige Protokollanfragen, Fehler).
- **Dynamische Konfiguration:** Die Log-Levels sollten zur Laufzeit anpassbar sein, um die Fehlersuche im Betrieb zu erleichtern.

### 7.4. Testing Strategy

Ein mehrschichtiger Testansatz ist erforderlich:

- **Unit Tests:** Testen isolierter Funktionen und Module, insbesondere für die reine Logik in der Domain-Schicht (z. B. Fensterplatzierungsalgorithmen, Zustandsübergänge). Rusts integriertes Test-Framework (`#[test]`) wird hierfür genutzt.
- **Integration Tests:** Testen das Zusammenspiel von Komponenten innerhalb einer Schicht oder über Schichtgrenzen hinweg. Beispiel: Überprüfen, ob eine `xdg-toplevel.set_maximized`-Anfrage (System-Schicht) korrekt den Zustand des entsprechenden Fensters in der Domain-Schicht ändert.
- **End-to-End (E2E) Tests:** Testen vollständiger Benutzerszenarien aus der Sicht eines Clients oder Benutzers. Dies ist die komplexeste Testebene.
    - **Testumgebungen:** Ähnlich wie bei Smithays Beispielkompositor Anvil 2 sollten Tests in verschiedenen Umgebungen laufen können:
        - **Verschachtelt (Nested):** Ausführung des Kompositors als Client innerhalb einer bestehenden X11- oder Wayland-Sitzung (mittels `--winit` oder `--x11` Backends). Dies ist ideal für automatisierte Tests in CI/CD-Pipelines, da keine spezielle Hardware benötigt wird.
        - **TTY/DRM:** Ausführung direkt auf der Hardware (`--tty-udev` Backend). Notwendig, um die Interaktion mit DRM/KMS, `libinput` und die Sitzungsverwaltung zu testen.
    - **Test-Clients:** Verwendung von Standard-Wayland-Clients (`weston-terminal`, `gedit`, etc.), spezialisierten Test-Clients oder Frameworks (z. B. Wayland-Konformitätstestsuiten) zur Simulation von Client-Verhalten und Überprüfung der Protokollimplementierung. Spezifische Tests sind erforderlich, um die korrekte Handhabung von Protokoll-Nuancen wie der `xdg-decoration`-Aushandlung mit verschiedenen Clients (z. B. SDL-basierten 9) sicherzustellen.

Die Einrichtung und Wartung zuverlässiger Testumgebungen, insbesondere für E2E- und Hardware-Tests, stellt eine erhebliche Herausforderung dar. Die Komplexität ergibt sich aus der Abhängigkeit von spezifischer Hardware (GPUs), Kernel-Modulen (DRM), Systembibliotheken (`libinput`, EGL/WGPU, `libxkbcommon` 7) und dem Verhalten externer Clients.10 Daher müssen explizit Ressourcen für den Aufbau und die Pflege der Testinfrastruktur eingeplant werden. Dies umfasst CI-Pipelines, die verschachtelte Tests ausführen können, und potenziell dedizierte Test-Hardware für den TTY/DRM-Backend. Manuelle Testverfahren bleiben für Aspekte wie subjektive Benutzererfahrung oder schwer automatisierbare Szenarien (z. B. komplexe Multi-Monitor-Setups 10) notwendig.

### 7.5. _Wertvolle Tabelle:_ Komponenten-Teststrategie

|   |   |   |   |   |
|---|---|---|---|---|
|**Komponente/Modul**|**Unit Test Coverage**|**Integration Test Scenarios**|**E2E Test Scenarios**|**Spezifische Tools/Backends**|
|Event Loop (`calloop`)|Mittel|Hinzufügen/Entfernen von Quellen, Timer-Auslösung|Start/Stopp des Kompositors, Reaktion auf Signale|`calloop`-API, MPSC-Kanäle|
|Window Manager (Domain)|Hoch|Fenster erstellen/zerstören, Fokuswechsel, Geometrieänderung|Fenster überlappen, maximieren, minimieren, Workspace wechseln|Interne APIs, Mock-Oberflächen|
|XDG-Shell Handler (System)|Mittel|Client verbindet sich, erstellt Toplevel/Popup, sendet Requests|Starten/Bedienen einer XDG-Client-Anwendung (z. B. `gedit`)|Wayland Test Clients, `xdg-shell`-Protokolltests|
|Layer-Shell Handler (System)|Mittel|Client erstellt Layer Surface, setzt Anchor/Layer/Zone|Starten eines Panels/Docks (als Layer-Client), Interaktion mit normalen Fenstern|Wayland Test Clients (Layer Shell), `wlr-layer-shell`-Tests|
|Renderer (UI)|Niedrig|-|Fenster anzeigen, bewegen, Transparenz, Vollbild|Verschachtelte Backends (`--winit`, `--x11`), Visueller Vergleich|
|Damage Tracker (UI)|Mittel|Berechnung von Schadensregionen für Bewegung/Resize/Inhalt|Fenster bewegen/verändern, nur beschädigte Bereiche neu zeichnen (Profiling)|Verschachtelte Backends, Frame-Analyse-Tools, Profiler|
|Panel UI (UI/System)|Mittel|Interaktion mit Layer Shell, Klick-Handler|Panel anzeigen, Fensterliste aktualisieren, auf Klicks reagieren|Layer Shell Client, Verschachtelte Backends|
|Input Handling (System)|Mittel|Verarbeitung von `libinput`-Events, Weiterleitung an Domain|Tippen in fokussiertem Fenster, Mausbewegung, Klicks|Mock `libinput`, Wayland Test Clients|

_Coverage: Ziel-Codeabdeckung (%), Schlüsselbereiche_

### 7.6. Dependency Management

- **Rust Crates:** Abhängigkeiten werden über `Cargo.toml` verwaltet. Es wird eine Strategie für regelmäßige Updates (`cargo update`) und die Überprüfung auf bekannte Sicherheitslücken benötigt. Tools wie `cargo-deny` oder externe Dienste (wie der in 2 erwähnte) können zur Überwachung eingesetzt werden. Die Auswahl der Abhängigkeiten sollte deren Wartungszustand und Popularität berücksichtigen.2
- **System Dependencies:** Externe Bibliotheken wie `libwayland-client`, `libwayland-server`, `libxkbcommon`, `pkg-config`, `libinput`, EGL/OpenGL/Vulkan-Loader und Treiber müssen auf dem Zielsystem vorhanden sein.7 Der Build-Prozess muss diese Abhängigkeiten überprüfen (z. B. mittels `pkg-config`). Dokumentation und ggf. Build-Skripte müssen die Installation dieser Abhängigkeiten erleichtern.

### 7.7. Build System & CI/CD

- **Build System:** Der Standard `cargo build`-Prozess wird verwendet. Build-Profile (Debug, Release) werden konfiguriert.
- **Continuous Integration (CI):** Eine CI-Pipeline (z. B. GitHub Actions, GitLab CI) wird eingerichtet. Diese sollte bei jedem Commit/Pull Request automatisch:
    - Den Code kompilieren (`cargo build`).
    - Linting-Checks durchführen (`cargo clippy`).
    - Code formatieren (`cargo fmt --check`).
    - Unit- und Integrationstests ausführen (`cargo test`).
    - Idealerweise E2E-Tests in einer verschachtelten Umgebung durchführen.
- **Continuous Deployment (CD):** Optional kann eine CD-Pipeline eingerichtet werden, um z. B. Nightly Builds oder Releases automatisch zu erstellen und zu veröffentlichen.

## 8. Implementation Phasing & High-Level Roadmap

### 8.1. Logical Sequencing

Eine phasierte Implementierung wird vorgeschlagen, um schrittweise Funktionalität aufzubauen und frühes Feedback zu ermöglichen:

- **Phase 1 (Core Setup & Grundstruktur):**
    - Projekt-Setup (Repository, Lizenz, Readme).
    - Implementierung der Core-Schicht: `calloop`-Ereignisschleife, grundlegende Wayland-Verbindung, Logging, Fehlerbehandlung.
    - Einrichtung des Build-Systems und der CI-Pipeline.
    - _Ziel:_ Kompositor startet, verbindet sich mit Wayland (als Client), loggt Nachrichten und kann beendet werden.
- **Phase 2 (System Basics & Rendering-Grundlagen):**
    - Implementierung der System-Schicht-Grundlagen: Grafik-Backend-Initialisierung (z. B. EGL/WGPU), grundlegende DRM/KMS-Integration (falls TTY-Backend benötigt wird).
    - Implementierung wesentlicher Wayland-Protokolle: `wl_compositor`, `wl_output`, `wl_seat`.
    - Minimales Rendering: Fähigkeit, einen einfarbigen Hintergrund oder einen einfachen Client-Buffer anzuzeigen.
    - _Ziel:_ Ein minimaler Wayland-Client (z. B. `weston-simple-shm`) kann sich verbinden und seine Oberfläche wird angezeigt.
- **Phase 3 (Domain & Windowing):**
    - Implementierung der Domain-Schicht: `CompositorState` 1, Fenstermanagement-Abstraktion (Geometrie, Fokus).
    - Implementierung von `xdg-shell` (Toplevels).
    - Grundlegende Fensterinteraktion: Fenster verschieben, Größe ändern (initial ohne Client-Feedback), Fokus per Klick setzen.
    - _Ziel:_ Mehrere `xdg-shell`-Clients können gestartet, angezeigt und rudimentär manipuliert werden.
- **Phase 4 (UI & Shell-Grundlagen):**
    - Implementierung der Rendering-Pipeline mit Komposition mehrerer Oberflächen.
    - Implementierung von grundlegendem Damage Tracking.3
    - Implementierung von `wlr-layer-shell`.4
    - Entwicklung erster Shell-Komponenten (z. B. Hintergrundmanager, einfaches Panel als Layer-Shell-Client).
    - Implementierung von `xdg-popup` und `xdg-decoration` (mit korrekter Aushandlung 9).
    - _Ziel:_ Eine grundlegende Desktop-Shell ist sichtbar, Fenster haben Dekorationen (CSD/SSD), Popups funktionieren.
- **Phase 5 (Advanced Features & Polish):**
    - Implementierung weiterer Wayland-Protokolle (Datenübertragung, etc.).
    - Verbessertes Damage Tracking und Performance-Optimierungen.
    - Vollständige Implementierung aller geplanten Shell-Features (Launcher, Notifications, etc.).
    - Implementierung von Theming und Konfigurationsmechanismen.
    - Umfassende Tests (insbesondere E2E und auf Hardware).
    - Dokumentation vervollständigen.
    - _Ziel:_ Eine funktionsreiche, performante und stabile Desktop-Umgebung.

### 8.2. Key Milestones

Messbare Meilensteine für jede Phase:

- **M1 (Ende Phase 1):** Kompositor läuft stabil in verschachtelter Umgebung, grundlegendes Logging funktioniert, CI ist grün.
- **M2 (Ende Phase 2):** Ein einfacher SHM-Client wird korrekt angezeigt (verschachtelt und/oder TTY). `wl_output` und `wl_seat` sind funktional.
- **M3 (Ende Phase 3):** Mehrere `xdg-toplevel`-Fenster können angezeigt, verschoben und fokussiert werden.
- **M4 (Ende Phase 4):** Grundlegendes Damage Tracking ist aktiv. Ein Panel (Layer Shell) ist sichtbar. Fensterdekorationen (CSD/SSD) funktionieren gemäß Aushandlung. Popups werden korrekt angezeigt.
- **M5 (Ende Phase 5):** Alle Kernfunktionen sind implementiert und getestet. Die Performance ist akzeptabel. Theming und Konfiguration sind möglich.

## 9. Schlussfolgerung

### 9.1. Zusammenfassung

Dieser Plan skizziert einen detaillierten, schichtweisen Ansatz zur Implementierung einer modernen Linux-Desktop-Umgebung. Er basiert auf der Verwendung von Wayland als Protokoll, Rust als Sprache und der Smithay-Bibliothek als Fundament.1 Der Fokus liegt auf Modularität, klarer Trennung der Verantwortlichkeiten zwischen den Schichten (Core, Domain, System, UI), Robustheit durch sorgfältige Fehlerbehandlung und umfassende Tests sowie Performance durch Techniken wie Damage Tracking.3 Besondere Aufmerksamkeit gilt der korrekten Implementierung von Wayland-Protokollen unter Berücksichtigung ihrer Semantik und potenzieller Client-Verhaltensweisen 9, der Notwendigkeit klar definierter interner Abstraktionen (z. B. Fenstermanagement) 1 und der Komplexität der Integration verschiedener Systemkomponenten wie der Ereignisschleife 5 und Testumgebungen.2

### 9.2. Nächste Schritte

Die unmittelbaren nächsten Schritte zur Umsetzung dieses Plans sind:

1. **Projekt-Setup:** Erstellung des Code-Repositorys, Wahl einer Lizenz, Einrichtung der grundlegenden Projektstruktur und der CI/CD-Pipeline gemäß Phase 1.
2. **Team-Briefing:** Vorstellung und Diskussion dieses Implementierungsplans mit dem Entwicklungsteam, um ein gemeinsames Verständnis der Architektur und der Vorgehensweise sicherzustellen.
3. **Beginn Phase 1:** Start der Implementierung der Core-Schicht-Komponenten (Ereignisschleife, Logging, Fehlerbehandlung).
4. **Spezifikationsverfeinerung:** Ggf. Verfeinerung der technischen Gesamtspezifikation basierend auf ersten Erkenntnissen aus der Prototypenentwicklung und der detaillierten Planung in diesem Dokument.
5. **Technologie-Evaluation (falls nötig):** Finale Entscheidung über spezifische Technologien (z. B. Grafik-Backend WGPU vs. EGL/GLES), falls noch nicht festgelegt.

Die konsequente Verfolgung dieses Plans, kombiniert mit agilen Praktiken zur Anpassung an neue Erkenntnisse, legt den Grundstein für die erfolgreiche Entwicklung einer robusten und performanten Linux-Desktop-Umgebung.