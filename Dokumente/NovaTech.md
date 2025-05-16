Technische Gesamtspezifikation und EntwicklungsrichtlinienI. EinleitungDieses Dokument dient als umfassende technische Spezifikation und Sammlung von Entwicklungsrichtlinien für die Erstellung einer neuartigen Linux-Desktop-Umgebung. Ziel ist es, eine moderne, schnelle, intuitive und KI-gestützte Benutzererfahrung zu schaffen, die sowohl für Entwickler, Kreative als auch für alltägliche Nutzer optimiert ist.Der Zweck dieser Spezifikation ist es, eine solide technische Grundlage zu legen und klare Richtlinien für das Entwicklungsteam bereitzustellen. Sie definiert die übergeordnete Architektur, den Technologie-Stack, die Kernkomponenten und die grundlegenden Entwicklungsprinzipien. Dieses Dokument ist die Basis für die nachfolgenden, detaillierten Implementierungsleitfäden für jede Architekturschicht und soll sicherstellen, dass Entwickler die Umgebung direkt implementieren können, ohne grundlegende technologische oder architektonische Entscheidungen selbst treffen zu müssen.Die vorgeschlagene Desktop-Umgebung basiert auf einer klaren, geschichteten Architektur, um Modularität, Wartbarkeit und Testbarkeit zu maximieren.II. Architektonischer Überblick (Schichtenarchitektur)Das System ist in vier logische Schichten unterteilt, die jeweils spezifische Verantwortlichkeiten tragen und über definierte Schnittstellen miteinander kommunizieren. Diese Trennung fördert die Kohäsion innerhalb der Schichten und reduziert die Kopplung zwischen ihnen.
Kernschicht (Core Layer):

Verantwortlichkeiten: Enthält die grundlegendsten Datentypen, Dienstprogramme, Konfigurationsgrundlagen, Logging-Infrastruktur und allgemeine Fehlerdefinitionen, die von allen anderen Schichten verwendet werden. Diese Schicht hat keine Abhängigkeiten zu den anderen Schichten.
Interaktionen: Stellt Funktionalität für alle darüberliegenden Schichten bereit.


Domänenschicht (Domain Layer):

Verantwortlichkeiten: Beinhaltet die Kernlogik und die Geschäftsregeln der Desktop-Umgebung. Dies umfasst die Verwaltung von Workspaces ("Spaces"), das Theming-System, die Logik für KI-Interaktionen (inkl. Einwilligungsmanagement), die Verwaltung von Benachrichtigungen und die Definition von Richtlinien für das Fenstermanagement (z.B. Tiling-Regeln). Diese Schicht ist unabhängig von spezifischen UI-Implementierungen oder Systemdetails wie D-Bus oder Wayland.
Interaktionen: Nutzt Funktionalität der Kernschicht. Stellt Logik und Zustand für die System- und Benutzeroberflächenschicht bereit.


Systemschicht (System Layer):

Verantwortlichkeiten: Implementiert die Interaktion mit dem zugrundeliegenden Betriebssystem und externen Diensten. Dies beinhaltet den Wayland-Compositor, die Eingabeverarbeitung (libinput), die Kommunikation über D-Bus mit Systemdiensten (Netzwerk, Energie, Audio, Secrets, PolicyKit), die Implementierung von Wayland-Protokollen, die XWayland-Integration, die MCP-Client-Implementierung und die Interaktion mit XDG Desktop Portals.
Interaktionen: Nutzt Funktionalität der Kern- und Domänenschicht (z.B. um Domänenregeln anzuwenden oder Zustände abzufragen). Stellt systemnahe Dienste und Ereignisse für die Benutzeroberflächenschicht bereit. Implementiert die konkrete "Mechanik" des Fenstermanagements basierend auf den Richtlinien der Domänenschicht.


Benutzeroberflächenschicht (User Interface Layer):

Verantwortlichkeiten: Ist verantwortlich für die Darstellung der Benutzeroberfläche und die Interaktion mit dem Benutzer. Dies umfasst die Shell-UI (Panels, Dock, etc.), das Control Center, das Widget-System, den Übersichtsmodus, die Befehlspalette und alle anderen sichtbaren Elemente. Diese Schicht verwendet das gewählte GUI-Toolkit (GTK4) und interagiert mit der Systemschicht, um Aktionen auszuführen und Systemzustände darzustellen.
Interaktionen: Nutzt Funktionalität aller darunterliegenden Schichten, insbesondere der Systemschicht (um Fenster zu verwalten, Eingaben zu empfangen, Systemdienste anzusprechen) und der Domänenschicht (um Zustände darzustellen und Geschäftslogik auszulösen).


Diese Schichtung stellt sicher, dass Änderungen in einer Schicht (z.B. der Austausch des GUI-Toolkits) minimale Auswirkungen auf andere Schichten haben, insbesondere auf die Kern- und Domänenschicht.III. Technologie-StackDie Auswahl des Technologie-Stacks basiert auf den Anforderungen an Modernität, Leistung, Sicherheit, Wartbarkeit und der Verfügbarkeit von qualitativ hochwertigen Bibliotheken und Werkzeugen im Linux-Ökosystem.
3.1. Programmiersprache:

Entscheidung: Rust.1
Begründung: Rust bietet eine überzeugende Kombination aus Leistung ("close-to-the-metal" ähnlich C++) und Speichersicherheit ohne Garbage Collector, was für Systemsoftware wie eine Desktop-Umgebung entscheidend ist.1 Sein starkes Typsystem und das Ownership-Modell verhindern viele gängige Fehlerklassen (z.B. Null Pointer, Data Races) bereits zur Kompilierzeit, was die Stabilität und Sicherheit erhöht.1 Rust fördert zudem moderne Nebenläufigkeitskonzepte, was für eine reaktionsschnelle UI wichtig ist.3 Obwohl C++ eine längere Historie im Embedded-/Systembereich hat und über umfangreichere Bibliotheken verfügt 1, wird Rust aufgrund seiner Sicherheitsgarantien und seiner wachsenden Popularität und Ökosystemreife als zukunftssicherere Wahl für dieses Projekt angesehen.1 Die Lernkurve für Rust, insbesondere das Ownership-Modell, wird als investitionswürdig betrachtet, um die langfristigen Vorteile in Bezug auf Codequalität und Wartbarkeit zu realisieren.3


3.2. Build-System:

Entscheidung: Meson.4
Begründung: Meson ist ein modernes Meta-Build-System, das oft als einfacher und schneller als CMake wahrgenommen wird, insbesondere für Projekte, die nicht tief in der CMake-Welt verwurzelt sind.4 Es bietet eine klare Syntax und ein "opinionated" Design, das zu konsistenteren Build-Skripten führen kann.4 Meson hat gute Unterstützung für Rust und C/C++ und kann CMake-basierte Abhängigkeiten über sein Subprojekt-System (WrapDB, CMake-Modul) integrieren.5 Obwohl CMake eine breitere Akzeptanz und mächtigere Features für komplexe Szenarien haben mag, wird Meson aufgrund seiner Einfachheit, Geschwindigkeit und guten Integration verschiedener Sprachen als passender für dieses Projekt eingeschätzt. Die Verwaltung von Abhängigkeiten über WrapDB und Subprojekte wird als ausreichend für die Projektanforderungen bewertet.5


3.3. GUI-Toolkit:

Entscheidung: GTK4.6
Begründung: GTK4 ist ein modernes, aktiv entwickeltes Toolkit mit erstklassiger Wayland-Unterstützung.6 Es bietet offizielle und gut gepflegte Rust-Bindings (gtk4-rs). GTK4 verwendet CSS für das Theming, was eine flexible und standardisierte Anpassung des Erscheinungsbildes ermöglicht und gut zur geplanten Token-basierten Theming-Architektur passt.6 Während Qt6 ebenfalls eine leistungsstarke Alternative mit guter Wayland-Unterstützung und breiter Plattformabdeckung (inkl. Mobile/Embedded) darstellt 6, wird GTK4 aufgrund seiner starken Integration in das GNOME-Ökosystem (aus dem viele Linux-Technologien stammen), seiner LGPL-Lizenzierung 6 und der als etwas einfacher empfundenen Lernkurve für reine Desktop-Anwendungen bevorzugt. Die Möglichkeit, das Erscheinungsbild von Qt-Anwendungen an GTK anzupassen (und umgekehrt), existiert, erfordert aber zusätzliche Konfiguration.8 Die Fokussierung auf GTK4 vereinfacht die Entwicklung und sorgt für eine konsistente Basis. Die Fähigkeit von GTK4, Themes zur Laufzeit zu wechseln (z.B. über CSS-Änderungen oder GtkStyleContext), ist entscheidend für die dynamische Theme-Umschaltung.9


3.4. Wayland Compositor & Bibliotheken:

Entscheidung: Smithay Toolkit.11
Begründung: Smithay ist eine in Rust geschriebene Bibliothek zum Erstellen von Wayland-Compositors.11 Dies passt perfekt zur Wahl von Rust als Hauptprogrammiersprache und vermeidet die Notwendigkeit von C-Bindings für Kernkomponenten des Compositors. Smithay ist modular aufgebaut ("kein Framework, das einschränkt") und ermöglicht die selektive Nutzung benötigter Komponenten.11 Es bietet Unterstützung für wichtige Wayland-Protokolle, XWayland-Integration 13 und Abstraktionen für Backends (DRM, libinput, etc.). Obwohl wlroots 15 eine etablierte C-Bibliothek ist, die von vielen bekannten Compositors (wie Sway, Hyprland) genutzt wird 15, bietet Smithay den Vorteil einer nativen Rust-Implementierung, was die Integration in das Rust-basierte Projekt vereinfacht und potenzielle Probleme an der Sprachgrenze reduziert. Die aktive Entwicklung und der Fokus auf Sicherheit und Modularität machen Smithay zu einer vielversprechenden Wahl.11


3.5. Essentielle Wayland-Protokolle:

Kern: wayland.xml (Basisprotokoll).17
Fensterverwaltung: xdg-shell (Standard für Desktop-Fenster).
Shell-UI-Elemente: wlr-layer-shell-unstable-v1 (Für Panels, Docks, Benachrichtigungen etc., die über normalen Fenstern liegen).18 Smithay bietet Unterstützung hierfür.
Fensterdekorationen: xdg-decoration-unstable-v1 (Verhandlung zwischen Client und Server über Fensterrahmen/-steuerungen).20
Taskleisten/Docks: wlr-foreign-toplevel-management-unstable-v1 (Auflisten und Steuern von Fenstern anderer Anwendungen).22
Display-Konfiguration: wlr-output-management-unstable-v1 (Lesen und Ändern von Monitorkonfigurationen wie Auflösung, Position, Aktivierungsstatus).26 Smithay unterstützt dies.
Energieverwaltung (Displays): wlr-output-power-management-unstable-v1 (Steuern des Energiesparmodus von Monitoren).26 Smithay unterstützt dies.
Eingabemethoden: input-method-unstable-v1, text-input-unstable-v3 (Für On-Screen-Keyboards, CJK-Eingabe etc.).
Zwischenablage & DND: Wayland Core (wl_data_device_manager), ggf. wlr-data-control-unstable-v1.
Screencasting/Screenshots: XDG Desktop Portal APIs (siehe unten) oder ggf. wlr-screencopy-unstable-v1.
X11-Kompatibilität: XWayland (unterstützt durch Smithay).13
Weitere: presentation-time, viewporter, linux-dmabuf-unstable-v1, idle-notify-unstable-v1, etc. je nach Bedarf.
Hinweis: Die Verwendung von wlr-*-Protokollen bindet die Funktionalität primär an Compositors, die diese unterstützen (wlroots-basiert, Smithay-basiert, KWin).19 GNOME verwendet oft eigene Protokolle oder Ansätze. Dies ist eine bewusste Entscheidung zugunsten der verfügbaren Funktionalität in der Rust/Smithay-Welt.


3.6. Inter-Prozess-Kommunikation (IPC):

Entscheidung: D-Bus.30
Begründung: D-Bus ist der De-facto-Standard für IPC im Linux-Desktop-Bereich.30 Es wird von essenziellen Systemdiensten wie NetworkManager, UPower (Energieverwaltung), logind (Session-Management), PolicyKit (Rechteverwaltung), dem Benachrichtigungsdienst (org.freedesktop.Notifications) und dem Secret Service (org.freedesktop.secrets) verwendet. Die Nutzung von D-Bus ermöglicht eine nahtlose Integration mit diesen Diensten. Es gibt etablierte Rust-Bibliotheken für D-Bus (z.B. zbus, oder das ältere dbus 31).


3.7. KI-Integration:

Entscheidung: Model Context Protocol (MCP).32
Begründung: MCP ist ein offener Standard, der speziell für die sichere und standardisierte Verbindung von KI-Modellen (LLMs) mit externen Werkzeugen, Datenquellen und Anwendungen (wie der Desktop-Umgebung) entwickelt wurde.32 Es bietet eine Client-Server-Architektur und definierte Nachrichtenformate (Requests, Responses, Notifications) 33, was die Integration vereinfacht und Vendor Lock-in vermeidet.32 Dies passt gut zur Anforderung, sowohl lokale als auch Cloud-basierte Modelle sicher anzubinden und dem Benutzer die Kontrolle über Datenzugriffe zu geben (Einwilligungsmanagement).


3.8. Eingabeverarbeitung:

Entscheidung: libinput [User Query].
Begründung: libinput ist die Standardbibliothek unter Linux für die Verarbeitung von Eingabeereignissen von Tastaturen, Mäusen, Touchpads etc. auf niedrigem Niveau.34 Wayland-Compositors wie die, die mit Smithay oder wlroots gebaut werden, integrieren libinput typischerweise, um rohe Hardware-Events zu verarbeiten und in Wayland-Events umzusetzen.15 libinput bietet robuste Unterstützung für Touchpad-Gesten (Pinch, Swipe, Hold).35 Die Verwendung von libinput über den Compositor stellt eine konsistente und präzise Eingabebehandlung sicher.


3.9. Audio-Management:

Entscheidung: PipeWire.34
Begründung: PipeWire ist der moderne Standard für Audio- (und Video-) Management unter Linux und ersetzt zunehmend PulseAudio und JACK.34 Es bietet geringe Latenz, flexible Routing-Möglichkeiten und eine sandboxed Architektur. Es stellt Kompatibilitätsschichten für PulseAudio-, JACK- und ALSA-Anwendungen bereit.34 Für die Interaktion mit PipeWire aus Rust gibt es Bibliotheken wie pipewire-rs oder spezifischere Wrapper/Anwendungen.37 Die Nutzung von PipeWire ermöglicht die Steuerung von Lautstärke, Geräteauswahl und Anwendungs-Streams.


3.10. Geheimnisverwaltung:

Entscheidung: Freedesktop Secret Service API.
Begründung: Dies ist der Standardmechanismus auf Linux-Desktops zum sicheren Speichern von sensiblen Daten wie Passwörtern oder API-Schlüsseln.39 Implementierungen wie GNOME Keyring oder KWallet stellen diesen Dienst über D-Bus bereit (org.freedesktop.secrets).39 Die Verwendung dieser API stellt sicher, dass sensible Daten (z.B. API-Keys für Cloud-LLMs) nicht im Klartext gespeichert werden und durch die Mechanismen des Systems geschützt sind. Es gibt Rust-Bibliotheken (secret-service-rs) für die Interaktion mit dieser API.39


3.11. Rechteverwaltung:

Entscheidung: PolicyKit (polkit).
Begründung: PolicyKit ist der Standardmechanismus unter Linux, um privilegierten Aktionen (wie Systemaktualisierungen, Änderung von Energieeinstellungen, Installation von Software) durch unprivilegierte Anwendungen zu erlauben, typischerweise nach Authentifizierung des Benutzers [User Query]. Die Interaktion erfolgt über D-Bus. Die Nutzung von PolicyKit ist unerlässlich, um sicherzustellen, dass administrative Aufgaben nur mit expliziter Benutzerzustimmung durchgeführt werden.


3.12. Theming-Implementierung:

Entscheidung: Token-basiertes System, implementiert über GTK4 CSS Custom Properties (var()) und @define-color (als Fallback/Ergänzung).7
Begründung: Ein Token-basiertes System bietet eine Abstraktionsebene über konkreten Werten (Farben, Schriftgrößen, Abstände).41 Designentscheidungen werden als benannte Tokens gespeichert (z.B. --background-color, --accent-color). GTK4 unterstützt CSS Custom Properties (--name: value;) und deren Verwendung (var(--name)) 7, was eine direkte Umsetzung dieses Konzepts ermöglicht. Änderungen an den Token-Definitionen (z.B. Wechsel von Hell- zu Dunkelmodus) können zur Laufzeit angewendet werden, indem die CSS-Definitionen aktualisiert werden, was die dynamische Theme-Umschaltung ohne Neustart ermöglicht.41 Tokens werden in Schichten organisiert (Foundation/Core -> Alias/Semantic -> Component-specific) 42, um Konsistenz und Wartbarkeit zu gewährleisten. Die generierten CSS-Dateien werden über GtkCssProvider geladen.


3.13. Desktop Portals:

Entscheidung: XDG Desktop Portals.43
Begründung: XDG Desktop Portals bieten standardisierte D-Bus-Schnittstellen für sandboxed Anwendungen (wie Flatpaks, aber auch native Apps), um sicher auf Ressourcen außerhalb ihrer Sandbox zuzugreifen.43 Dies umfasst Funktionen wie Datei öffnen/speichern-Dialoge, Kamera-/Mikrofonzugriff, Standortdienste, Hintergrunddienste, Screencasting/Screenshots und mehr.43 Die Nutzung dieser Portale ist der empfohlene Weg, um solche Funktionen unter Wayland sicher zu implementieren, anstatt direkte System-APIs zu verwenden. Es gibt Rust-Bibliotheken (xdg-portal) für die Interaktion mit diesen Portalen.43 Die Desktop-Umgebung muss ggf. Backend-Implementierungen für einige dieser Portale bereitstellen oder sicherstellen, dass vorhandene (z.B. xdg-desktop-portal-gtk) korrekt funktionieren.


Zusammenfassung Technologiestack:
BereichTechnologie/StandardBegründungReferenzenProgrammierspracheRustSicherheit, Performance, Concurrency, Modernität1Build-SystemMesonModern, Einfach, Rust/C++ Integration, WrapDB4GUI-ToolkitGTK4Modern, Wayland, Rust Bindings, CSS Theming6Compositor-ToolkitSmithayRust-nativ, Modular, Wayland Protokolle, XWayland11Display-Server-ProtokollWaylandModern, Sicher, Performant17Wayland-ProtokolleXDG Shell, WLR Layer Shell, XDG Decoration, etc.Standardisierte Schnittstellen für Desktop-Funktionen19IPCD-BusLinux Standard, Integration mit Systemdiensten30KI-IntegrationModel Context Protocol (MCP)Offener Standard, Sichere LLM-Anbindung, Flexibel32Eingabeverarbeitunglibinput (via Smithay)Standard Linux Eingabe, Gestenunterstützung34AudioPipeWireModerner Linux Standard, Flexibel, Kompatibel34GeheimnisverwaltungFreedesktop Secret Service APIStandard, Sicher, Integration mit Keyring/KWallet39RechteverwaltungPolicyKit (polkit)Standard für privilegierte Aktionen[User Query]ThemingToken-basiert via GTK4 CSS Custom PropertiesFlexibel, Standardisiert, Laufzeit-Änderungen7Sandboxing-InteraktionXDG Desktop PortalsSicherer Zugriff auf Systemressourcen für Anwendungen43
IV. EntwicklungsrichtlinienDiese Richtlinien sollen Konsistenz, Qualität und Wartbarkeit des Codes über das gesamte Projekt hinweg sicherstellen.
4.1. Coding Style & Formatierung:

Entscheidung: Standard rustfmt Konfiguration plus Einhaltung der Rust API Guidelines.45
Begründung: Die Verwendung eines automatischen Formatierungswerkzeugs wie rustfmt stellt einen konsistenten Code-Stil sicher, reduziert Diskussionen über Formatierung und verbessert die Lesbarkeit.45 Die Rust API Guidelines bieten bewährte Praktiken für die Gestaltung von öffentlichen Schnittstellen in Rust-Bibliotheken und -Anwendungen.46
Implementierung: rustfmt wird als Teil des CI-Prozesses ausgeführt, um die Einhaltung zu überprüfen. Entwickler müssen rustfmt vor dem Committen ausführen. Die Richtlinien umfassen Einrückung (4 Leerzeichen), maximale Zeilenlänge (100 Zeichen), Kommentierungsstil (bevorzugt //), Modulorganisation und Namenskonventionen (RFC 430 Casing, as_, to_, into_, iter, iter_mut, into_iter etc.).45 Doc Comments (/// oder //!) werden für die Dokumentation verwendet.45


4.2. API-Design:

Entscheidung: Befolgung der Rust API Guidelines Checklist.46
Begründung: Konsistente und idiomatisches Rust-APIs verbessern die Benutzerfreundlichkeit und Wartbarkeit der Bibliotheken und Komponenten des Desktops. Die Guidelines decken Aspekte wie Namensgebung, Trait-Implementierungen, Fehlerbehandlung, Dokumentation, Flexibilität und Typsicherheit ab.46
Implementierung: Entwickler müssen sich mit den Guidelines vertraut machen. Wichtige Punkte sind:

Implementierung gängiger Traits (Debug, Clone, Send, Sync, Default, From, AsRef, etc.) wo sinnvoll.46
Sinnvolle Fehlertypen (std::error::Error).46
Klare Unterscheidung zwischen &self, &mut self und self Methoden.
Verwendung von Generics zur Minimierung von Annahmen über Parameter.46
Verwendung von Newtypes für Typsicherheit und Kapselung.46
Vermeidung von bool oder Option für Argumente, wo spezifischere Typen (Enums, Structs, Bitflags) aussagekräftiger sind.46
Bereitstellung von Builder-Patterns für komplexe Typen.46
Private Felder in Structs und versiegelte Traits (#[doc(hidden)] pub trait Sealed {}) zum Schutz vor Breaking Changes.46




4.3. Fehlerbehandlung:

Entscheidung: Verwendung des thiserror Crates zur Definition von benutzerdefinierten, spezifischen Fehler-Enums pro Modul.48
Begründung: thiserror vereinfacht die Erstellung idiomatischer Rust-Fehlertypen, die das std::error::Error Trait implementieren und gute Fehlermeldungen über das #[error(...)] Attribut bereitstellen.48 Die Verwendung von spezifischen Enums pro Modul (z.B. ThemingError, CompositorError) bietet einen guten Kompromiss zwischen Granularität und Verwaltbarkeit.49 Es ermöglicht dem aufrufenden Code, Fehler spezifisch zu behandeln (falls nötig) oder sie einfach mittels des ?-Operators weiterzugeben.48 Panics durch unwrap() oder expect() sind in Bibliotheks-/Kerncode zu vermeiden; sie sind nur für unrecoverable Fehler oder in Tests/Beispielen akzeptabel.48 Das #[from] Attribut in thiserror erleichtert das Umwandeln von Fehlern aus Abhängigkeiten oder unteren Schichten.48 Diese strukturierte Herangehensweise ist der Verwendung eines einzigen, globalen Fehler-Enums oder unspezifischer Fehler (wie Box<dyn Error>) in den meisten Fällen vorzuziehen, da sie mehr Kontext und Typsicherheit bietet.49 Die Konsistenz in der Fehlerbehandlungsstruktur über ein großes Projekt hinweg ist entscheidend. Ohne eine klare Regel zur Granularität könnten verschiedene Teams unterschiedliche Ansätze verfolgen, was zu einer inkonsistenten und schwerer zu nutzenden API-Oberfläche für die Fehlerbehandlung führt. Eine Regel pro Modul bietet eine vernünftige Balance zwischen Spezifität und der Vermeidung einer übermäßigen Verbreitung von Fehlertypen.
Implementierung: In der Kernschicht wird ggf. ein Basis-Fehler-Enum oder Trait definiert. Jedes Modul (z.B. compositor, theming_engine, mcp_client) definiert sein eigenes Error-Enum mit #. Varianten repräsentieren spezifische Fehlerfälle des Moduls. Fehler aus Abhängigkeiten werden mittels #[from] in passende Varianten des Modul-Fehlers konvertiert.


4.4. Logging & Tracing:

Entscheidung: Nutzung des tracing Crate-Frameworks.50
Begründung: tracing bietet strukturiertes, kontextbezogenes Logging mit Unterstützung für Spans (Zeitmessung von Codeabschnitten), was leistungsfähiger ist als das traditionelle log-Crate für die Fehlersuche in komplexen, asynchronen Systemen wie einer Desktop-Umgebung.51 Es ermöglicht die Unterscheidung zwischen Ereignissen (Momente) und Zeitspannen und deren hierarchische Verschachtelung.51 tracing integriert sich gut in asynchronen Rust-Code und verfügt über ein wachsendes Ökosystem von Subscribers und Layern für verschiedene Ausgabeformate (Text, JSON, Bunyan) und Analysewerkzeuge (OpenTelemetry, Jaeger, etc.).50
Implementierung: Standard-Logging-Levels (ERROR, WARN, INFO, DEBUG, TRACE) werden definiert. Die Verwendung von Spans (z.B. via #[tracing::instrument] Makro oder manuell erzeugten Spans) für wichtige Funktionen, Operationen oder Bearbeitungszyklen ist vorgeschrieben, um Kontext zu liefern. Ein Standard-Subscriber (z.B. tracing_subscriber::fmt für menschenlesbare Ausgabe oder tracing-bunyan-formatter 51 für strukturierte Logs) wird für Entwicklungs-Builds konfiguriert. Es werden Richtlinien definiert, was auf welchem Level geloggt werden soll (z.B. Fehler und Warnungen immer, INFO für wichtige Lebenszyklusereignisse, DEBUG für detaillierte Abläufe, TRACE für sehr feingranulare Informationen) und dass sensible Daten (Passwörter, Geheimnisse) niemals geloggt werden dürfen.


4.5. Versionskontrolle & Branching:

Entscheidung: Verwendung von Git mit dem GitHub Flow Branching-Modell.52
Begründung: GitHub Flow ist ein schlankeres Modell als das komplexere Gitflow. Es fördert Agilität und Continuous Integration/Delivery, da der main-Branch stets den neuesten stabilen (potenziell auslieferbaren) Zustand repräsentiert.52 Feature-Entwicklung geschieht in kurzlebigen Branches, die direkt von main abzweigen und nach Code-Review und erfolgreichen CI-Prüfungen wieder in main gemerged werden.52 Dies passt gut zu einem Projekt, das auf schnelle Iteration abzielt. Langwierige develop- oder release-Branches werden vermieden, was den Prozess vereinfacht.53 Die Wahl von GitHub Flow, das die kontinuierliche Auslieferung betont, erfordert zwingend eine robuste Continuous Integration Pipeline, um die Stabilität des main-Branches sicherzustellen.52 Jeder Merge in main sollte idealerweise auslieferbar sein. Ohne starke automatisierte Tests steigt das Risiko, Regressionen in die Hauptentwicklungslinie einzuführen. Die explizite Verknüpfung der Wahl des Branching-Modells mit der Notwendigkeit einer umfassenden Teststrategie unterstreicht diese Abhängigkeit und stellt sicher, dass Tests priorisiert werden.
Implementierung: Der main-Branch ist geschützt (direkte Pushes sind verboten). Alle Entwicklungsarbeiten erfolgen in Feature-Branches, die von main erstellt werden (z.B. feature/neues-widget, fix/crash-im-compositor). Pull Requests (PRs) sind für das Mergen in main obligatorisch. Jeder PR erfordert mindestens ein positives Code-Review von einem anderen Teammitglied und das Bestehen aller CI-Checks (Kompilierung, Tests, Linter, Formatter). Es werden Konventionen für Commit-Nachrichten festgelegt (z.B. Conventional Commits), um die Historie nachvollziehbar zu gestalten und automatisierte Changelog-Generierung zu ermöglichen.


4.6. Teststrategie:

Unit-Tests: Umfassende Unit-Tests für Komponenten der Kern- und Domänenschicht sind obligatorisch. Tests sollen direkt neben dem zu testenden Code im selben Modul oder in einem Untermodul tests liegen (#[cfg(test)]). Eine hohe Codeabdeckung durch Unit-Tests wird angestrebt.
Integrationstests: Integrationstests sind im tests/-Verzeichnis auf Crate-Ebene erforderlich. Sie testen das Zusammenspiel verschiedener Module oder Schichten (z.B. ein UI-Befehl löst eine Aktion in der Systemschicht aus, die den Zustand in der Domänenschicht ändert) und die Interaktion mit externen Schnittstellen (z.B. gemockte D-Bus-Dienste, simulierte Wayland-Ereignisse).
Compositor-Tests: Die Testbarkeit des Compositors (Systemschicht) ist eine Herausforderung. Es sollen Möglichkeiten evaluiert werden, wie z.B. die Verwendung von verschachtelten/headless Wayland-Backends (falls von Smithay oder Abhängigkeiten unterstützt) oder spezialisierte Test-Clients, um Kernlogiken des Compositors (Fensterplatzierung, Protokollimplementierungen) zu verifizieren.
UI-Tests: Das Testen der grafischen Benutzeroberfläche (UI-Schicht) ist komplex. Der Ansatz muss noch definiert werden. Mögliche Strategien umfassen die Nutzung von Accessibility-APIs zur Inspektion und Steuerung von Widgets, die Simulation von Eingabeereignissen auf Wayland-Ebene oder den Einsatz von Frameworks für visuelle Regressionstests (falls für GTK4/Rust verfügbar). Aufgrund der Komplexität wird zunächst ein Fokus auf Tests der unteren Schichten gelegt.
CI-Pipeline: Alle definierten Tests (Unit, Integration) müssen in der Continuous Integration (CI) Pipeline erfolgreich durchlaufen, bevor Code in den main-Branch gemerged werden darf. Die CI-Pipeline beinhaltet außerdem Prüfungen für Code-Formatierung (cargo fmt --check), Linting (cargo clippy -- -D warnings) und Sicherheitslücken in Abhängigkeiten (cargo audit).


4.7. Dokumentation:

Code-Dokumentation: Umfassende rustdoc-Kommentare sind für alle öffentlichen APIs (Crates, Module, Funktionen, Typen, Felder) zwingend erforderlich.46 Die Dokumentation soll nicht nur beschreiben, was eine Komponente tut, sondern auch warum und wie sie verwendet wird. Sie muss Erklärungen zur Logik, zu Invarianten, zu möglichen Fehlerbedingungen (Errors-Sektion) und zu Sicherheitsaspekten (# Safety-Sektion für unsafe Code) enthalten.46 Aussagekräftige Code-Beispiele (#[doc(inline)] oder in separaten examples/-Verzeichnissen) sind essenziell.46
Architektur-Dokumentation: High-Level-Architekturdiagramme (z.B. C4-Modell, Schichtendiagramme, Komponentendiagramme) müssen erstellt und aktuell gehalten werden, um das Gesamtsystem zu beschreiben. Dieses Spezifikationsdokument dient als zentrales Referenzdokument.
READMEs: Jedes Crate und jede größere Komponente muss eine README.md-Datei besitzen, die den Zweck, die Hauptfunktionen und grundlegende Nutzungsanweisungen beschreibt.
Metadaten: Alle Cargo.toml-Dateien müssen vollständige und korrekte Metadaten enthalten, einschließlich authors, description, license, repository, homepage, documentation, keywords und categories.46 Dies ist wichtig für die Veröffentlichung und Auffindbarkeit.


V. Initiale Schichtspezifikationen (Grobe Komponentenübersicht)Diese Übersicht dient dazu, Entwicklern eine erste Vorstellung von den Hauptkomponenten innerhalb jeder Schicht zu geben und den Kontext für die detaillierten Implementierungsleitfäden zu schaffen.
5.1. Kernschicht (Core Layer) Komponenten:

core::types: Fundamentale Datenstrukturen (z.B. Point, Rect, Size, Color), gemeinsam genutzte Enums (z.B. Orientation).
core::errors: Basis-Fehlertypen/-Traits (z.B. ein generischer CoreError), ggf. unter Verwendung von thiserror.
core::logging: Makros und Initialisierungsfunktionen für das tracing-Framework.
core::config: Primitive zum Laden und Parsen von Basiskonfigurationen (z.B. aus Dateien).
core::utils: Allgemeine Hilfsfunktionen (String-Manipulation, mathematische Helfer, etc.).


5.2. Domänenschicht (Domain Layer) Komponenten:

domain::theming: Logik der Theming-Engine, Verwaltung von Design-Tokens, Regeln zur Anwendung von Themes (Hell/Dunkel, Akzentfarben).
domain::workspaces: Verwaltungslogik für Arbeitsbereiche (Erstellung, Löschung, Wechsel, Zuweisung von Fenstern). Implementiert Konzepte wie "Spaces".
domain::ai: Kernlogik für KI-Interaktionen, Verwaltung von Einwilligungsrichtlinien, Kontextaufbau für MCP-Anfragen.
domain::settings: Repräsentation und Logik zur Verwaltung des Zustands der Desktop-Einstellungen (unabhängig von der Speicherung oder UI).
domain::notifications: Logik zur Verwaltung des Benachrichtigungszustands, Historie, Filterung und Priorisierung.
domain::window_management: High-Level-Regeln und Richtlinien für Fensterplatzierung, Logik für automatisches Tiling und Snapping (die "Policy").


5.3. Systemschicht (System Layer) Komponenten:

system::compositor: Die eigentliche Smithay-basierte Wayland-Compositor-Implementierung. Verwaltet Wayland-Surfaces, integriert Rendering-Backends (z.B. OpenGL ES via EGL), implementiert Wayland-Protokolle (xdg-shell, wlr-layer-shell, xdg-decoration, etc.).
system::input: Integration von libinput, Verarbeitung von Tastatur-, Maus- und Touchpad-Ereignissen, Gestenerkennung, Verwaltung von Tastaturlayouts, Fokus-Handling auf Wayland-Ebene.
system::dbus: Schnittstellenmodule für die Interaktion mit spezifischen System-D-Bus-Diensten (NetworkManager, UPower, logind, org.freedesktop.Notifications, Secret Service, PolicyKit). Enthält D-Bus Service- und Client-Implementierungen.
system::outputs: Implementierung der wlr-output-management und wlr-output-power-management Protokolle serverseitig; Handhabung von Display-Konfigurationsänderungen (via D-Bus oder direkt).
system::xwayland: Logik zur Integration und Verwaltung des XWayland-Servers unter Verwendung der von Smithay bereitgestellten Hilfsmittel.13
system::audio: Client-Integration für PipeWire zur Steuerung der Systemlautstärke, Auswahl von Audio-Geräten und ggf. Verwaltung von Anwendungs-Streams.37
system::mcp: Implementierung des MCP-Clients, der die sichere Kommunikation mit MCP-Servern (lokal oder remote) abwickelt.32
system::portals: Logik zur Interaktion mit XDG Desktop Portal D-Bus-Schnittstellen, um Anfragen von Anwendungen (z.B. Datei öffnen) zu bedienen.43


5.4. Benutzeroberflächenschicht (User Interface Layer) Komponenten:

ui::shell: Implementierung der Haupt-Shell-UI mit GTK4 und wlr-layer-shell. Enthält Panels/Leisten, die dynamische Tab-Leiste/Panel, Anwendungsmenü/-starter, System-Tray-Äquivalent.
ui::control_center: GTK4-Implementierung der modularen Einstellungsoberfläche. Kommuniziert über D-Bus oder interne APIs mit der System-/Domänenschicht.
ui::widgets: Implementierung des Widget-Systems. Enthält die Seitenleiste, Logik zum Laden, Anzeigen und Verwalten von Widgets (inkl. Drag & Drop). Beinhaltet Standard-Widgets (Uhr, Kalender, Wetter, Systemmonitor) als GTK4-Komponenten.
ui::window_manager_frontend: Verantwortlich für UI-Aspekte des Fenstermanagements: Client-seitige Dekorationen (falls CSD verwendet wird, Verhandlung via xdg-decoration), Fensterwechsler-UI (Übersichtsmodus/"Exposé", Alt-Tab). Interagiert mit dem Compositor (via wlr-foreign-toplevel-management Client-seitig) und internen APIs.
ui::notifications: Frontend-Komponente (GTK4) zur Anzeige von Benachrichtigungen, die über den D-Bus-Benachrichtigungsdienst empfangen werden. Implementiert das Benachrichtigungszentrum.
ui::theming_gtk: Wendet die von der Domänenschicht (Theming Engine) generierten CSS-Stile auf GTK-Widgets an. Implementiert die Logik zur Anforderung von Theme-Wechseln zur Laufzeit.
ui::components: Sammlung wiederverwendbarer, anwendungsspezifischer GTK4-Widgets und UI-Komponenten (z.B. spezielle Buttons, Layout-Container).
ui::speed_dial: GTK4-Implementierung der Speed-Dial-Startansicht.
ui::command_palette: GTK4-Implementierung der kontextuellen Befehlspaletten-UI.


VI. Deployment-ÜberlegungenDie Auslieferung und Installation einer vollständigen Desktop-Umgebung ist komplexer als die einer einzelnen Anwendung. Es erfordert eine tiefere Integration in das Basissystem.
6.1. Paketierung:

Es müssen Ziel-Paketierungsformate definiert werden (z.B. native Pakete wie .deb/.rpm, möglicherweise auch Flatpak für Teile der Umgebung oder für die SDK-Bereitstellung). Der Build-Prozess zur Erstellung dieser Artefakte muss spezifiziert werden, inklusive der Handhabung von Abhängigkeiten. Standardverfahren für Linux-Systembereitstellungen 54 können als Referenz dienen, müssen aber für die Bereitstellung einer Kern-Desktop-Umgebung (statt nur eines Agenten) angepasst werden. Die Bereitstellung einer Desktop-Umgebung erfordert eine tiefere Systemintegration (z.B. Display-Manager-Konfiguration, systemd User-Sessions, PAM-Module) als die in den Referenzen beschriebene Agenten-Installation. Diese Komplexität muss bei der Planung der Installationsroutine berücksichtigt werden.


6.2. Konfiguration:

Die Bereitstellung von Standardkonfigurationen und die Möglichkeit für Benutzer, diese zu überschreiben, muss spezifiziert werden. Dabei ist die XDG Base Directory Specification strikt einzuhalten ($XDG_CONFIG_HOME, $XDG_DATA_HOME, $XDG_STATE_HOME) 34, um eine saubere Trennung von Benutzerdaten und Systemkonfiguration zu gewährleisten.


6.3. Updates:

Eine Strategie für die Auslieferung von Updates muss definiert werden (z.B. über die Paketmanager der Distributionen, Flatpak-Updates). Dies beinhaltet Überlegungen zur Versionierung und zur Handhabung von Konfigurationsänderungen bei Updates.


VII. SchlussfolgerungDiese technische Gesamtspezifikation legt das Fundament für die Entwicklung der neuen Linux-Desktop-Umgebung. Sie definiert eine klare, geschichtete Architektur, wählt einen modernen und robusten Technologie-Stack basierend auf Rust, Wayland, GTK4 und Smithay und etabliert wesentliche Entwicklungsrichtlinien für Qualität, Konsistenz und Wartbarkeit.Die getroffenen Entscheidungen bezüglich Architektur, Technologien (wie Rust für Sicherheit und Leistung, Smithay für einen nativen Wayland-Compositor, GTK4 für die UI und Theming, MCP für KI-Integration) und Richtlinien (wie GitHub Flow, thiserror für Fehlerbehandlung, tracing für Logging) bilden die Grundlage für die nachfolgenden, detaillierten Implementierungsleitfäden der einzelnen Schichten und Komponenten.Durch die konsequente Einhaltung dieser Spezifikation soll eine hochwertige, moderne, sichere und anpassungsfähige Desktop-Umgebung entstehen, die den Anforderungen von Entwicklern, Kreativen und alltäglichen Nutzern gerecht wird und eine intuitive, effiziente und persönliche Benutzererfahrung bietet.