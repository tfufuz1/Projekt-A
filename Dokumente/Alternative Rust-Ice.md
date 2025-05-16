# Konkreter Plan für eine Desktopumgebung mit Rust und Iced

## 1. Einleitung

### 1.1. Zielsetzung

Dieses Dokument skizziert einen konkreten technischen Plan für die Entwicklung einer modernen, performanten und robusten Desktop-Umgebung (DE), die auf der Programmiersprache Rust und dem GUI-Toolkit Iced basiert. Das primäre Zielsystem ist das Wayland-Display-Protokoll, der moderne Nachfolger des X11-Systems.1

### 1.2. Begründung der Technologiewahl

Die Wahl von Rust als Fundament für eine neue Desktop-Umgebung ist durch dessen Stärken motiviert: Speichersicherheit ohne Garbage Collection, hohe Performance, exzellente Nebenläufigkeitskonzepte und ein wachsendes Ökosystem.4 Diese Eigenschaften sind besonders vorteilhaft für Systemsoftware wie eine Desktop-Umgebung, die robust und ressourceneffizient sein muss.

Iced wird als GUI-Toolkit gewählt, da es sich um ein modernes, in Rust geschriebenes Toolkit handelt, das dem Elm-Architekturmuster (Model-View-Update) folgt.7 Es verspricht Einfachheit, Typsicherheit und eine gute Integration in das Rust-Ökosystem. Die Verwendung von Iced im COSMIC DE-Projekt von System76 8 demonstriert seine Eignung für komplexe Desktop-Anwendungen.

Wayland wird als Display-Protokoll bevorzugt, da es im Vergleich zu X11 eine modernere Architektur, bessere Sicherheitsmerkmale und eine effizientere Handhabung von Grafik und Eingabe bietet.1

### 1.3. Kernprinzipien

Die Entwicklung dieser Desktop-Umgebung sollte von folgenden Prinzipien geleitet werden:

- **Modularität**: Die Architektur sollte modular aufgebaut sein, um die Wartbarkeit zu erleichtern und die unabhängige Entwicklung von Komponenten zu ermöglichen.
- **Standardkonformität**: Eine strikte Einhaltung relevanter Freedesktop.org-Standards ist essenziell für die Interoperabilität mit dem breiteren Linux-Ökosystem.
- **Benutzererfahrung (UX)**: Trotz des Fokus auf Technologie sollte eine kohärente und intuitive Benutzererfahrung angestrebt werden.
- **Performance**: Rust und Wayland bieten das Potenzial für eine sehr performante Desktop-Umgebung; dies sollte durch effiziente Implementierungen realisiert werden.

### 1.4. Herausforderungen

Die Entwicklung einer vollständigen Desktop-Umgebung ist ein komplexes Unterfangen. Spezifische Herausforderungen bei diesem Ansatz umfassen:

- **Wayland-Komplexität**: Wayland ist primär ein Protokoll.2 Der Compositor trägt eine große Verantwortung und muss viele Interaktionen mit dem System und den Clients verwalten.
- **Rust GUI-Ökosystem**: Obwohl vielversprechend, ist das Rust GUI-Ökosystem, insbesondere für Wayland-spezifische Desktop-Komponenten, noch nicht so ausgereift wie etablierte Toolkits wie GTK oder Qt.9
- **Iced-Wayland-Integration**: Spezifische Wayland-Protokolle, die für Desktop-Shell-Elemente wie Panels oder Docks benötigt werden (z. B. `wlr-layer-shell`), werden von Iced bzw. dessen Standard-Backend `winit` nicht nativ unterstützt.19 Dies erfordert Workarounds oder spezialisierte Lösungen.
- **Theming**: Die Integration mit systemweiten Themes (GTK/Qt) stellt eine Herausforderung dar, da Iced sein eigenes Theming-System verwendet.17

### 1.5. Geltungsbereich

Dieser Plan konzentriert sich auf die architektonische Blaupause, die Definition der Kernkomponenten, die Auswahl geeigneter Technologien und Bibliotheken sowie die Integrationsstrategien. Er ist keine detaillierte Implementierungsanleitung, sondern ein technischer Fahrplan für das Projekt.

## 2. Kernarchitektur: Der Wayland Compositor

### 2.1. Rolle und Verantwortlichkeiten

Im Wayland-Modell ist der Compositor die zentrale Komponente, die die Aufgaben des traditionellen X-Servers, des Fenstermanagers und des Compositing-Managers vereint.2 Er ist allein verantwortlich für das Rendern des finalen Bildschirminhalts, die Verwaltung der Fenster (Surfaces) von Client-Anwendungen, die Verarbeitung von Hardware-Eingabeereignissen (Tastatur, Maus, Touch), die Verwaltung von Bildschirmausgaben (Monitore) und die Durchsetzung von Sicherheitsrichtlinien. Wayland selbst definiert nur das Kommunikationsprotokoll; der Compositor ist die Software, die dieses Protokoll implementiert und die Desktop-Logik bereitstellt.2

### 2.2. Wahl der Grundlage: Smithay vs. Alternativen

Die Wahl der Basisbibliothek für den Compositor ist eine fundamentale Entscheidung.

#### 2.2.1. Option 1: Smithay (Empfohlen für Rust-native DE)

- **Überblick**: Smithay ist eine in Rust geschriebene Bibliothek, die speziell für die Erstellung von Wayland-Compositors entwickelt wurde.5 Sie zielt auf Sicherheit, Modularität und die Bereitstellung von High-Level-Abstraktionen ab.30 Smithay wird aktiv entwickelt und gewinnt an Reife, was durch seine Verwendung im COSMIC DE-Projekt unterstrichen wird.30
- **Architektur**: Smithay ist modular aufgebaut und bietet dedizierte Module für Backend-Interaktionen (Grafik, Eingabe, Session), Wayland-Protokoll-Handling, Desktop-Management und mehr.27 Es nutzt die `calloop`-Bibliothek für eine Callback-orientierte Ereignisschleife, was gut zur reaktiven Natur eines Compositors passt.27
- **Vorteile**: Der entscheidende Vorteil von Smithay ist sein Rust-natives Design. Dies vermeidet die typischen Fallstricke und die Impedanzfehlanpassung, die bei der Anbindung von C-Bibliotheken wie wlroots an Rust auftreten können. Insbesondere die Handhabung von dynamischen Objektlebenszeiten und Callback-Mustern aus C heraus kann zu Problemen mit Rusts Borrow-Checker führen, wie Erfahrungen mit `wlroots-rs` gezeigt haben.5 Smithay ist von Grund auf mit Rust-Idiomen konzipiert, was zu potenziell sichereren und ergonomischeren APIs führt.
- **Reife**: Smithay hat seit seiner Entstehung erhebliche Fortschritte gemacht.27 Obwohl wlroots möglicherweise einen größeren Funktionsumfang oder eine breitere Akzeptanz hat 39, bietet Smithay eine solide und wachsende Basis für die Kernfunktionen einer Desktop-Umgebung, insbesondere wenn das Ziel ein vollständig in Rust realisiertes System ist.3
- **Beispiel**: Der `anvil`-Compositor im Smithay-Repository dient als Referenzimplementierung und Testfeld.30

#### 2.2.2. Option 2: wlroots-Bindings (z.B. `wlroots-rs` - Mit Vorsicht)

- **Überblick**: wlroots ist eine etablierte C-Bibliothek, die als Grundlage für viele Compositors dient, darunter Sway.2 Es existieren Rust-Bindings, aber deren Entwicklung und Wartung waren historisch herausfordernd.5
- **Herausforderungen**: Die primäre Schwierigkeit liegt in der sicheren und ergonomischen Überbrückung der Unterschiede zwischen C-Idiomen und Rusts Sicherheitsmodell. Dynamische Lebenszeiten von Objekten, Callback-basierte APIs und Pointer-Arithmetik in C lassen sich oft nur schwer oder mit unsicherem Code (`unsafe`) und komplexen Abstraktionen (wie Handles und `run`-Methoden 36) in Rust abbilden.5 Das Way Cooler-Projekt, das ursprünglich `wlroots-rs` nutzte, wurde eingestellt.6

#### 2.2.3. Option 3: Von Grund auf neu (Nicht empfohlen)

Die Entwicklung eines Compositors ohne eine Basisbibliothek wie Smithay oder wlroots ist extrem aufwändig. Sie erfordert tiefgreifendes Wissen über Grafik-APIs (DRM/KMS, EGL/Vulkan), Eingabeverarbeitung (libinput), Wayland-Protokolldetails und Systemintegration.6 Die Aussage von wlroots ("50.000 Zeilen Code, die man sowieso schreiben müsste" 6) unterstreicht den immensen Aufwand. Das `ewc`-Projekt 41 ist ein Beispiel für diesen anspruchsvollen Weg.

#### 2.2.4. Vergleich der Compositor-Grundlagen

Die folgende Tabelle fasst die wichtigsten Unterschiede zusammen:

|   |   |   |   |
|---|---|---|---|
|**Eigenschaft**|**Smithay**|**wlroots-rs (Historisch/Hypothetisch)**|**Von Grund auf neu**|
|**Sprachfokus**|Rust-nativ|C-Bindings|Rust (oder C/C++)|
|**Abstraktionsebene**|High-Level Helfer & Protokoll-Handling|Wrapper um C-Bibliothek|Rohe Protokolle & System-APIs|
|**Speichersicherheit**|Rust-Garantien|FFI-Herausforderungen|Manuell|
|**Reife/Adoption**|Wachsend (COSMIC DE)|Breit (Sway), Bindings problematisch|N/A|
|**Kernabhängigkeiten**|`wayland-rs`, `calloop`, Rust-Ökosystem|`libwlroots`, `wlroots-sys`|`libwayland`, Grafik, Input, etc.|
|**Eignung für Rust DE**|**Hoch**|Herausfordernd|Sehr gering|
|**Begründung der Eignung**|Idiomatisch, sicher, vermeidet FFI-Probleme|FFI-Impedanz, Borrow-Check-Probleme|Extremer Entwicklungsaufwand|

Diese Gegenüberstellung verdeutlicht, warum Smithay trotz der möglicherweise größeren Verbreitung von wlroots die strategisch sinnvollere Wahl für ein Projekt ist, das auf Rust und dessen Sicherheitsversprechen setzt. Es adressiert direkt die dokumentierten Schwierigkeiten 5 bei der Anbindung von C-Bibliotheken und ermöglicht eine kohärentere Entwicklungsumgebung.

### 2.3. Essentielle Compositor-Funktionen & Smithay-Module

Ein funktionaler Desktop-Compositor benötigt Unterstützung für diverse Wayland-Protokolle und Systeminteraktionen. Smithay bietet hierfür entsprechende Module und Abstraktionen:

- **Wayland-Kernprotokoll**: Das Modul `smithay::wayland::compositor` 29 ist fundamental. Es verwaltet `wl_surface`, `wl_subsurface` und `wl_region` Objekte und implementiert die komplexe Semantik des doppelt gepufferten Zustands (double-buffered state), die für Wayland-Surfaces und darauf aufbauende Protokollerweiterungen entscheidend ist.38
- **Shell-Integration**:
    - **XDG-Shell**: Für Standard-Anwendungsfenster wird das `xdg-shell`-Protokoll benötigt. Smithay unterstützt dies über das Modul `smithay::wayland::shell::xdg` und Delegationsmakros wie `delegate_xdg_shell!`.27 Dies umfasst Toplevel-Fenster, Popups, Fenstergeometrie und Aktivierung (`delegate_xdg_activation!`).27
    - **Layer-Shell**: Für Panels, Docks und andere Shell-Komponenten, die auf bestimmten Ebenen über oder unter normalen Fenstern liegen, wird das `wlr-layer-shell`-Protokoll verwendet. Smithay bietet Unterstützung durch `smithay::wayland::shell::wlr_layer` und das `delegate_layer_shell!` Makro.27
    - **Dekorationen**: `delegate_xdg_decoration!` 27 ermöglicht serverseitige Fensterdekorationen (Rahmen, Titelleisten) für XDG-Shell-Fenster.
- **Eingabeverarbeitung**: Das `smithay::input`-Modul 27 bietet Abstraktionen für Eingabegeräte. Die Integration mit `libinput` erfolgt über `smithay::backend::libinput`.2 Das `smithay::wayland::seat`-Modul (`delegate_seat!`) verwaltet logische "Seats" (Sammlungen von Eingabegeräten) und den Eingabefokus der Clients.29 Systemabhängigkeiten sind `libinput` und `libxkbcommon`.30
- **Ausgabeverwaltung**: Das `smithay::output`-Modul 29 und das `delegate_output!`-Makro sind für die Verwaltung von Monitoren zuständig. Dies beinhaltet das Erkennen von Ausgaben, das Setzen von Modi (Auflösung, Bildwiederholrate), Positionierung und Skalierung (wichtig für HiDPI). Die Implementierung interagiert typischerweise mit dem DRM/KMS-Subsystem des Kernels, oft über `smithay::backend::drm`.2
- **Rendering-Backend**: Smithay bietet eine Renderer-Abstraktion (`smithay::backend::renderer`) mit einer mitgelieferten GLES2-Implementierung.27 Dies ist eng gekoppelt mit der Pufferverwaltung (`smithay::backend::allocator`), die GBM (`libgbm`) und DMABUF für effizienten Pufferaustausch mit Clients unterstützt.29 Eine Vulkan-basierte Implementierung ist geplant.38 Benötigt Grafikstack-Abhängigkeiten wie `libgbm`.30
- **Session-Management**: Das `smithay::backend::session`-Modul 27 integriert mit `libseat` (oder alternativ `elogind`) 2, um Gerätezugriffsrechte sicher zu verwalten und VT-Wechsel zu ermöglichen. Dies ist entscheidend für den Betrieb außerhalb einer bestehenden grafischen Sitzung. Benötigt `libseat` als Systemabhängigkeit.30
- **XWayland-Unterstützung**: Für die Kompatibilität mit älteren X11-Anwendungen bietet Smithay das `smithay::xwayland`-Modul (`delegate_xwayland!`).2 Benötigt die `xwayland`-Systemabhängigkeit.30

Die durchgängige Verwendung von Delegationsmakros (`delegate_*!`) in Smithay 27 ist ein zentrales Designmerkmal. Diese Makros nehmen dem Entwickler die Implementierung großer Teile der Wayland-Protokolllogik ab, indem sie die Ereignisbehandlung und Zustandsverwaltung an dedizierte `*State`-Strukturen delegieren. Der Entwickler implementiert dann spezifische `*Handler`-Traits, um die anwendungsspezifische Logik bereitzustellen. Dieser Ansatz reduziert Boilerplate-Code und vereinfacht die Handhabung der komplexen, asynchronen Natur des Wayland-Protokolls erheblich.

## 3. Aufbau der Desktop Shell mit Iced

### 3.1. Iced Toolkit Übersicht

Iced ist ein in Rust geschriebenes, plattformübergreifendes GUI-Toolkit, das von der Elm-Architektur inspiriert ist und einem Model-View-Update (MVU)-Paradigma folgt.7 Es zeichnet sich durch seinen Fokus auf Einfachheit, Typsicherheit und eine idiomatische Rust-API aus. Iced bietet einen Satz von Standard-Widgets und verwendet moderne Rendering-Backends wie `wgpu` (mit Unterstützung für Vulkan, Metal, DX12) und `tiny-skia` als Software-Fallback.7

Die prominente Verwendung von Iced im COSMIC DE-Projekt von System76 8 zeigt seine grundsätzliche Eignung für den Aufbau einer Desktop-Umgebung. Dennoch ist Iced im Vergleich zu etablierten Toolkits wie GTK oder Qt jünger und in einigen Bereichen, insbesondere bei der Integration tiefgreifender Desktop-Funktionen, möglicherweise weniger ausgereift.9 Performance in Szenarien mit häufigen Neuzeichnungen kann eine Herausforderung darstellen, obwohl Optimierungen und die Nutzung im COSMIC-Projekt darauf hindeuten, dass dies handhabbar ist.48

### 3.2. Panel / Dock Implementierung

- **Anforderung**: Panels und Docks sind typische Shell-Elemente, die am Bildschirmrand verankert sind und oft über oder unter normalen Anwendungsfenstern liegen. Dies erfordert im Wayland-Kontext das `wlr-layer-shell`-Protokoll 51, das es Clients ermöglicht, Oberflächen auf bestimmten "Layern" zu platzieren und an Bildschirmkanten zu verankern.
- **Herausforderung**: Das Standard-Backend von Iced basiert auf `winit`.53 `winit` ist eine Bibliothek zur Fenstererstellung, die sich primär auf das Standard-`xdg-shell`-Protokoll für normale Anwendungsfenster konzentriert und keine native Unterstützung für `wlr-layer-shell` bietet.19 Dies stellt ein signifikantes Hindernis dar.
- **Lösung 1: `iced_layershell` Crate**: Eine mögliche Lösung ist die Verwendung der Drittanbieter-Crate `iced_layershell`.19 Diese Crate stellt explizit Bindings bereit, um Iced-Anwendungen als Layer-Shell-Oberflächen zu erstellen. Sie ermöglicht die Konfiguration von Layer, Ankerpunkten, Margins und Exklusivzonen.19 Die Verwendung dieser Crate führt jedoch eine zusätzliche Abhängigkeit ein und erfordert möglicherweise spezifische Wayland-Client-Bibliotheken.
- **Lösung 2: Pop!_OS Fork / Custom Backend**: Das COSMIC-Projekt hat dieses Problem gelöst, indem es entweder einen Fork von Iced oder ein benutzerdefiniertes Backend verwendet, das Layer-Shell-Funktionalität integriert.20 Dies unterstreicht, dass die Integration nicht trivial ist und möglicherweise tiefgreifende Anpassungen am Toolkit oder dessen Backend erfordert.
- **Implementierungsdetails**: Unabhängig von der gewählten Lösung muss die Implementierung die Semantik des Layer-Shell-Protokolls abbilden: Setzen der Ebene (`Layer::Top`, `Layer::Bottom`, etc.), der Ankerpunkte (`Anchor::Top`, `Anchor::Left`, etc.), der Ränder (`Margins`) und der exklusiven Zone (um zu verhindern, dass andere Fenster das Panel überlappen).51 Die Tastaturinteraktivität (`KeyboardInteractivity`) muss ebenfalls konfiguriert werden.51

Die Notwendigkeit externer Crates oder Forks für eine so grundlegende DE-Funktionalität wie Panels zeigt eine Reifungslücke im Rust/Iced-Ökosystem für Wayland-Desktops im Vergleich zu GTK oder Qt, die etablierte Bindings wie `gtk-layer-shell` 52 oder native Qt-Unterstützung (in neueren Versionen) bieten. Dies ist ein kritischer Punkt, der bei der Projektplanung berücksichtigt werden muss.

### 3.3. Anwendungsstarter (Application Launcher)

- **Funktionalität**: Ein Launcher muss installierte Anwendungen finden, dem Benutzer präsentieren und deren Start ermöglichen. Die Grundlage hierfür sind die `.desktop`-Dateien gemäß der Freedesktop Desktop Entry Specification.63
- **Implementierung (Iced)**: Der Launcher kann als Iced-Anwendung implementiert werden.7 Typische Komponenten sind ein Texteingabefeld für die Suche 66 und eine Liste oder ein Grid zur Anzeige der gefundenen Anwendungen. Für ein typisches "Popup"-Verhalten (z.B. über eine Tastenkombination) könnte er als Layer-Shell-Oberfläche implementiert werden, was die unter 3.2 genannten Herausforderungen mit sich bringt.
- **`.desktop`-Parsing**: Zum Einlesen und Verarbeiten der `.desktop`-Dateien eignen sich Crates wie `freedesktop-desktop-entry` 65, `freedesktop_entry_parser` 63 oder `freedesktop-file-parser`.64 Diese Dateien befinden sich üblicherweise in den Standard-XDG-Datenverzeichnissen (z.B. `/usr/share/applications`, `~/.local/share/applications`).67 Wichtige Informationen sind `Name`, `Icon`, `Exec` und `Categories`.
- **Anwendungsstart**: Der im `Exec`-Feld angegebene Befehl muss ausgeführt werden. Dabei müssen eventuelle Platzhalter (wie `%u` für URLs, `%f` für Dateien) korrekt ersetzt werden. Für Anwendungen, die über D-Bus aktiviert werden können (`DBusActivatable=true`), sollte D-Bus verwendet werden. Für sandboxed Anwendungen (z.B. Flatpak) ist möglicherweise die Verwendung von XDG Desktop Portals (`org.freedesktop.portal.Launcher`) erforderlich.71
- **Beispiele**: `cosmic-launcher` 11 und `onagre` 73 sind existierende Launcher, die mit Iced erstellt wurden. `elbey` 61 ist ein weiteres, experimentelles Beispiel.

### 3.4. Fenstermanagement-Integration (Shell-Perspektive)

- **Fensterdekorationen**: Iced-Anwendungen verwenden typischerweise Client-Side Decorations (CSDs), bei denen die Anwendung selbst ihre Titelleiste und Fensterrahmen zeichnet. Das COSMIC-Projekt hat die notwendige Unterstützung für CSDs (wie das Ziehen zum Verschieben und Größenänderung an den Rändern) in Iced implementiert.54 Server-Side Decorations (SSDs), bei denen der Compositor die Dekorationen zeichnet, wären theoretisch möglich, wenn der Compositor das `xdg-decoration`-Protokoll 27 unterstützt und die Clients dies anfordern, was aber für eine benutzerdefinierte DE weniger üblich ist.
- **Fensterinteraktion**: Grundlegende Interaktionen (Verschieben, Größenänderung über CSDs) werden von Iced und seinem Backend (`winit`) gehandhabt.54 Komplexere Fenstermanagement-Logik wie Tiling, Snapping oder die Logik zur Fokusverwaltung ist jedoch Aufgabe des _Compositors_ (siehe Abschnitt 2). Die Shell-UI (z.B. das Panel oder eine Übersichtsanzeige) kann jedoch Anfragen an den Compositor senden, um solche Aktionen auszulösen. Dies erfordert einen Kommunikationskanal, z.B. ein benutzerdefiniertes Wayland-Protokoll oder einen anderen IPC-Mechanismus (siehe Abschnitt 5.1).
- **Task-Switcher / Fensterübersicht**: Eine Funktion zum Anzeigen und Wechseln zwischen offenen Fenstern erfordert eine enge Integration mit dem Compositor. Der Compositor muss eine Liste der offenen Toplevel-Fenster bereitstellen (z.B. über das `wlr-foreign-toplevel-management`-Protokoll bei wlroots-basierten Compositors oder ein äquivalentes benutzerdefiniertes Protokoll bei Smithay) und idealerweise auch Vorschaubilder (Thumbnails) liefern können. Die UI für diese Funktion (oft Teil des Panels/Docks oder eine dedizierte Vollbildansicht) würde mit Iced erstellt.

## 4. Essentielle Desktop-Anwendungen & Dienste

Eine funktionale Desktop-Umgebung benötigt mehr als nur einen Compositor und eine Shell. Kernanwendungen wie ein Dateimanager, ein Einstellungsdialog, ein Terminal und ein Benachrichtigungsdienst sind unerlässlich.

### 4.1. Dateimanager

- **Herausforderungen**: Ein voll funktionsfähiger Dateimanager ist eine komplexe Anwendung. Features wie Dateioperationen (Kopieren, Verschieben, Löschen), Vorschaubilder, Netzwerkzugriff, Integration des Papierkorbs und Drag-and-Drop zwischen Anwendungen 75 müssen implementiert werden. Drag-and-Drop zwischen verschiedenen Anwendungen kann eine besondere Hürde für GUI-Toolkits darstellen.75
- **Implementierung mit Iced**: Grundsätzlich möglich, wie das Projekt COSMIC Files 10 zeigt, aber aufwändig. Es erfordert spezialisierte Widgets für die Verzeichnis-/Dateiauflistung (möglicherweise eine Baumansicht 77 oder Spaltenansicht wie in `fm` 78), Kontextmenüs und die Handhabung von Dateioperationen (oft asynchron).
- **Alternative Toolkits**: Viele existierende Dateimanager, auch in Rust, nutzen GTK (z.B. `rfm` 4, `fm` 78) über Bindings wie `relm4` 79 oder `gtk-rs`. Die Verwendung eines GTK-basierten Dateimanagers innerhalb der Iced-DE ist technisch möglich (via XWayland oder nativem Wayland-Backend von GTK), kann aber zu Inkonsistenzen im Erscheinungsbild und Verhalten führen.
- **Freedesktop-Integration**: Der Dateimanager muss MIME-Typen zur Dateierkennung und Anwendungsauswahl verwenden, `.desktop`-Dateien für "Öffnen mit..."-Aktionen interpretieren, die Freedesktop Trash Specification für den Papierkorb implementieren und möglicherweise XDG Portals 71 für den Zugriff auf Dateien außerhalb der Sandbox einer Anwendung nutzen (z.B. bei Flatpaks).
- **Referenz**: Der Quellcode von COSMIC Files ist auf GitHub verfügbar 76 und dient als wertvolles Beispiel für eine Iced-basierte Implementierung.

### 4.2. Einstellungsanwendung

- **Architektur**: Eine typische Einstellungsanwendung besteht aus einem GUI-Frontend und kommuniziert mit verschiedenen Systemdiensten oder Daemons im Hintergrund, um Einstellungen abzurufen und zu ändern. D-Bus ist hier der Standard-IPC-Mechanismus.47
- **GUI (Iced)**: Das Frontend wird mit Iced erstellt und bietet dem Benutzer eine Schnittstelle zur Konfiguration des Systems und der Desktop-Umgebung.87 COSMIC Settings 11 ist die Referenzimplementierung mit Iced.
- **Backend/D-Bus-Interaktion**: Die Anwendung muss mit Systemdiensten wie NetworkManager, UPower (für Energieverwaltung), systemd/logind (für Sitzungs- und Energieoptionen), PulseAudio/PipeWire (für Audioeinstellungen) etc. über D-Bus kommunizieren. Die `zbus`-Crate 83 ist hierfür die empfohlene Wahl im Rust-Ökosystem, da sie als reine Rust-Implementierung gilt und gute asynchrone Unterstützung bietet, oft bevorzugt gegenüber `dbus-rs`.97 Spezifische Bindings wie `zbus_systemd` 103 können die Interaktion mit systemd-Diensten vereinfachen.
- **XDG Portals**: Für Einstellungen, die sich auf sandboxed Anwendungen auswirken (z.B. Farbschema, Standardanwendungen, Dateizugriffsberechtigungen), sollte das `xdg-desktop-portal`-Framework 71 genutzt werden. Dies erfordert eine Portal-Implementierung (Backend) für die spezifische Desktop-Umgebung (z.B. `xdg-desktop-portal-cosmic` oder eine neue Implementierung). Die Kommunikation mit dem Portal erfolgt ebenfalls über D-Bus. Rust-Crates wie `ashpd` 104 oder `xdg-portal` 72 können hierfür verwendet werden. Die Konfiguration, welches Portal-Backend aktiv ist, hängt von der Umgebungsvariable `XDG_CURRENT_DESKTOP` und Konfigurationsdateien ab.71
- **Referenz**: Der Quellcode von COSMIC Settings 94 und dem zugehörigen Daemon 109 sind verfügbar.

### 4.3. Terminal-Emulator

- **Implementierung mit Iced**: Die Erstellung eines Terminal-Emulators mit Iced ist durch das `iced_term`-Widget 110 möglich. Dieses Widget basiert auf der Kernlogik von Alacritty (`alacritty_terminal`) 110 und bietet eine Integration in das Iced-Framework. COSMIC Terminal 10 ist die Referenzimplementierung.
- **Funktionen**: Ein Terminal-Emulator muss Pseudo-Terminals (PTYs) erzeugen und verwalten, ANSI-Escape-Sequenzen für Textformatierung und Cursor-Steuerung interpretieren, Schriften effizient rendern (COSMIC Terminal nutzt `cosmic-text` 114), Benutzereingaben verarbeiten und Scrolling ermöglichen. Moderne Terminals unterstützen oft auch Tabs, Splits und erweiterte Funktionen wie Ligaturen oder Bildanzeige (z.B. via Sixel oder Kitty Graphics Protocol 115).
- **Alternativen**: Es gibt andere bekannte Terminal-Emulatoren, die in Rust geschrieben sind, wie Alacritty oder Kitty (GPU-beschleunigt), diese verwenden jedoch nicht Iced für ihre GUI.116
- **Referenz**: Der Quellcode von COSMIC Terminal ist verfügbar.114

### 4.4. Benachrichtigungsdaemon (Notification Daemon)

- **Anforderung**: Eine Desktop-Umgebung benötigt einen Dienst, der Benachrichtigungen von Anwendungen entgegennimmt und anzeigt. Dies folgt der Freedesktop Desktop Notifications Specification.124
- **Architektur**: Typischerweise ist dies ein Hintergrunddienst (Daemon), der auf dem D-Bus lauscht und den Dienstnamen `org.freedesktop.Notifications` bereitstellt.14
- **Implementierung**: Der Daemon kann mit Rust und der `zbus`-Bibliothek 83 implementiert werden, um die D-Bus-Schnittstelle bereitzustellen. Wenn eine Benachrichtigungsanforderung empfangen wird, muss der Daemon die Benachrichtigung anzeigen.
- **Anzeige der Benachrichtigungen**: Die Popups selbst können auf verschiedene Arten realisiert werden:
    - Als Teil des Compositors: Der Daemon kommuniziert über ein benutzerdefiniertes Wayland-Protokoll oder das `wlr-layer-shell`-Protokoll 51 mit dem Compositor, der dann die Benachrichtigungsfenster als spezielle Oberflächen (z.B. Layer-Surfaces) rendert.
    - Als separate GUI-Anwendung: Der Daemon startet bei Bedarf einen separaten Prozess, der ein Iced-Fenster zur Anzeige der Benachrichtigung erstellt. Dieses Fenster wäre ein normales `xdg-shell`-Fenster, dessen Positionierung und "Always-on-Top"-Verhalten möglicherweise schwieriger zu steuern ist (siehe Abschnitt 5.5).
    - Benutzerdefinierte Anpassungen wie Templates für das Nachrichtenformat oder unterschiedliches Verhalten je nach Dringlichkeitsstufe (`low`, `normal`, `critical`) sind üblich.125
- **Client-Seite**: Anwendungen senden Benachrichtigungen typischerweise über Bibliotheken wie `notify-rust` 125 oder durch direkte D-Bus-Aufrufe an den `org.freedesktop.Notifications`-Dienst.
- **Existierende Rust-Daemons**: Es gibt bereits in Rust geschriebene Notification Daemons wie `runst` 125, `nofi` 126, `wired` 14 oder `end-rs`.136 Diese verwenden jedoch wahrscheinlich keine Iced-basierte Anzeige.

Die Entwicklung dieser Kernanwendungen verdeutlicht, dass eine Desktop-Umgebung weit mehr ist als nur die Summe ihrer GUI-Teile. Die tiefe Integration mit Systemdiensten über D-Bus und die Einhaltung von Freedesktop-Standards sind unerlässlich für eine funktionierende und interoperable Desktop-Erfahrung. Iced stellt das Werkzeug für die UI bereit, aber die darunterliegende Systemintegration erfordert zusätzliche Bibliotheken und sorgfältige Architektur.

### 4.5. Zusammenfassung der Komponentenstrategie

Die folgende Tabelle fasst die empfohlene technische Strategie für die Kernkomponenten zusammen:

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**Komponente**|**Primäres GUI Toolkit**|**Wichtige Wayland-Protokolle**|**IPC-Mechanismus**|**Wichtige Freedesktop-Spezifikationen**|**Empfohlene Rust Crates (Auswahl)**|
|**Panel / Dock**|Iced|`wlr-layer-shell`|Wayland|-|`iced`, `iced_layershell` (oder custom backend), `smithay` (Compositor-Seite)|
|**Launcher**|Iced|`xdg-shell` / `wlr-layer-shell`|Wayland / D-Bus|`.desktop`, XDG Base Dirs|`iced`, `freedesktop-desktop-entry`, `xdg`, `directories`, `zbus` (für Aktivierung)|
|**File Manager**|Iced|`xdg-shell`|D-Bus (Portals)|`.desktop`, XDG Base Dirs, MIME, Trash Spec, Portals (File Chooser)|`iced`, `xdg`, `directories`, `zbus`, `ashpd`/`xdg-portal` (für Portal-Interaktion)|
|**Settings App**|Iced|`xdg-shell`|D-Bus|XDG Base Dirs, Portals (Settings, Appearance)|`iced`, `zbus`, `zbus_systemd`, `ashpd`/`xdg-portal`, `xdg`, `directories`, Konfigurations-Crates|
|**Terminal Emulator**|Iced|`xdg-shell`|-|-|`iced`, `iced_term`, `alacritty_terminal`|
|**Notification Daemon**|Iced (für Popups)|`xdg-shell` / `wlr-layer-shell`(?)|D-Bus|Notification Spec, XDG Base Dirs|`zbus` (Daemon), `iced` (Popups), `notify-rust` (Client-Bibliothek)|

## 5. Systemintegration & Standardkonformität

### 5.1. Inter-Prozess-Kommunikation (IPC)-Strategie

Eine moderne Desktop-Umgebung besteht aus vielen interagierenden Prozessen (Compositor, Shell-Elemente, Anwendungen, Hintergrunddienste). Eine klare IPC-Strategie ist daher entscheidend.

- **Wayland-Protokolle**: Dies ist der primäre Kommunikationsweg zwischen Wayland-Clients (Anwendungen) und dem Compositor.15 Wayland ist optimiert für Aufgaben, die direkt die Anzeige, Eingabe oder Fensterverwaltung betreffen und bei denen der Compositor zwingend involviert sein muss. Es ist performant, insbesondere bei der gemeinsamen Nutzung von Grafikpuffern (z.B. über DMABUF 29). Wayland kann durch benutzerdefinierte Protokolle erweitert werden, um spezifische Kommunikation zwischen Shell-Komponenten und dem Compositor zu ermöglichen.138
- **D-Bus**: D-Bus ist der etablierte Standard für generische IPC auf dem Linux-Desktop.83 Es eignet sich hervorragend für die Kommunikation zwischen Anwendungen und Hintergrunddiensten, für Service Discovery, das Senden von Signalen (wie Benachrichtigungen) und die Bereitstellung von APIs für Systemeinstellungen. Die `zbus`-Bibliothek 83 wird für die Implementierung in Rust empfohlen.
- **Abwägung Wayland vs. D-Bus**: Die Wahl des richtigen Mechanismus hängt vom Anwendungsfall ab.85
    - **Wayland**: Immer dann verwenden, wenn der Compositor direkt beteiligt ist oder sein muss (Fensterpositionierung, Eingabefokus, Pufferaustausch, Layer-Shell-Positionierung).
    - **D-Bus**: Immer dann verwenden, wenn Dienste miteinander oder Anwendungen mit Diensten kommunizieren, die nicht der Compositor sind (Netzwerkmanager-Status abfragen, Audio-Lautstärke ändern, Benachrichtigungen senden, Systemeinstellungen ändern, Portal-Interaktionen).
    - Wayland sollte nicht als Allzweck-IPC-Mechanismus missbraucht werden.141 Für einige Anwendungsfälle gibt es Diskussionen, welcher Ansatz besser ist (z.B. Bildschirmaufnahme: dediziertes Wayland-Protokoll vs. D-Bus + PipeWire 85).

Die Notwendigkeit eines hybriden Ansatzes bedeutet, dass die Schnittstellen zwischen den Komponenten sorgfältig entworfen werden müssen. Beispielsweise kommuniziert die Iced-basierte Einstellungsanwendung über D-Bus mit einem Systemdaemon, während das Iced-basierte Panel über das Wayland-Layer-Shell-Protokoll mit dem Smithay-Compositor kommuniziert.

### 5.2. Freedesktop-Standards Compliance

Die Einhaltung von Freedesktop.org-Standards ist entscheidend für die Integration in das bestehende Linux-Ökosystem und die Kompatibilität mit Anwendungen.

- **XDG Base Directories**: Konfigurations-, Daten-, Cache- und Laufzeitdateien müssen an den standardisierten Orten abgelegt werden. Crates wie `xdg` 67 oder `directories` 68 helfen dabei, diese Pfade plattformübergreifend korrekt zu ermitteln.149
- **Desktop Entries (`.desktop`)**: Anwendungsstarter und Menüs basieren auf `.desktop`-Dateien. Diese müssen korrekt geparst werden, um Anwendungen anzuzeigen und zu starten. Crates: `freedesktop-desktop-entry` 65, `freedesktop_entry_parser` 63, `freedesktop-file-parser`.64
- **Icon Themes**: Die DE sollte die Icon Theme Specification respektieren, um konsistente Icons für Anwendungen und Systemelemente anzuzeigen. Dies beinhaltet das Suchen von Icons in den durch das Theme definierten Verzeichnissen.
- **MIME Types**: Dateitypen sollten über MIME-Typen identifiziert und die entsprechenden Standardanwendungen gemäß der `mimeapps.list`-Spezifikation gestartet werden.
- **Notifications**: Der Notification Daemon muss die Desktop Notifications Specification über D-Bus implementieren.124
- **Portals (`xdg-desktop-portal`)**: Für die sichere Interaktion mit sandboxed Anwendungen (z.B. Flatpak) sind Portale unerlässlich.71 Dies betrifft Dateizugriffe (File Chooser), das Öffnen von URIs, den Zugriff auf Systemeinstellungen (z.B. Farbschema), Screenshots und mehr. Die DE muss ein entsprechendes Portal-Backend bereitstellen (z.B. `xdg-desktop-portal-XYZ`), das die Anfragen über D-Bus entgegennimmt und mithilfe der DE-spezifischen Mechanismen (Compositor-Protokolle, Einstellungs-Daemons) beantwortet.

### 5.3. Konfigurationsmanagement

- **Formatwahl**:
    - **TOML**: Weit verbreitet im Rust-Ökosystem (z.B. `Cargo.toml`) und relativ einfach lesbar.150 Schwächen bei tief verschachtelten Hierarchien.151
    - **RON (Rusty Object Notation)**: Syntax ähnelt Rust, unterstützt Serdes Datenmodell gut.152 Kann für Nicht-Rust-Benutzer ungewohnt sein, insbesondere die explizite `Some()`-Syntax für `Option`-Typen.151 Eignet sich gut für interne oder entwicklerorientierte Konfigurationen.
    - **YAML**: Gut für Hierarchien, aber oft als komplex oder fehleranfällig empfunden.151
    - **JSON**: Sehr verbreitet, aber keine Kommentare, was für Konfigurationsdateien nachteilig ist.153
    - **KDL**: Ein neueres Format, das als Alternative zu XML/JSON/YAML positioniert wird.153
    - **Empfehlung**: Eine Kombination könnte sinnvoll sein: TOML 154 für einfache, benutzerseitige Konfigurationen und RON 152 für komplexere, interne Zustände oder wenn eine enge Anlehnung an Rust-Strukturen gewünscht ist. COSMIC verwendet RON für einige Konfigurationen (z.B. Akzentpaletten 94).
- **Bibliotheken**:
    - **Serde**: Die Grundlage für Serialisierung und Deserialisierung in Rust.151
    - **`toml`**: Die Standard-Crate für TOML-Verarbeitung.154
    - **`ron`**: Die Crate für RON-Verarbeitung.152
    - **`config-rs`**: Bietet Layering von Konfigurationsquellen (Dateien, Umgebungsvariablen, Defaults).162 Unterstützt jedoch kein Zurückschreiben von Änderungen in Konfigurationsdateien.162
    - **Schreiben von Konfiguration**: Wenn Konfigurationen programmatisch geändert und gespeichert werden sollen (z.B. durch die Einstellungsanwendung), muss dies direkt über Serde und die Serialisierungsfunktionen der jeweiligen Format-Crate (z.B. `toml::to_string`, `ron::ser::to_string`) erfolgen.
- **Strategie**: Es muss klar definiert werden, wo Konfigurationsdateien gespeichert werden (XDG-Pfade 149), wie verschiedene Konfigurationsquellen (Defaults, System, Benutzer) priorisiert und zusammengeführt werden 162 und wie Änderungen angewendet werden (Neustart erforderlich vs. Live-Reload).

### 5.4. Theming

- **Iced-Theming**: Iced bringt ein eigenes Theming-System mit.46 Es definiert Standard-Themes (z.B. Light, Dark) und ermöglicht die Erstellung benutzerdefinierter Themes durch Definition von Paletten und Styling-Regeln. Themes können zur Laufzeit dynamisch geändert werden.46 Das Styling einzelner Widgets erfolgt über die `.style()`-Methode, die entweder eine vordefinierte Stilvariante oder eine Closure bzw. eine Implementierung eines `StyleSheet`-Traits akzeptiert.167
- **System-Theme-Integration**: Eine der größten Herausforderungen ist die Integration mit systemweiten Themes, wie sie von GTK oder Qt verwendet werden. Iced bietet keine eingebaute Funktionalität, um automatisch das Aussehen des Systems zu übernehmen.17 Das COSMIC DE implementiert beispielsweise sein eigenes Theming-System _aufbauend_ auf Iced, anstatt ein bestehendes System-Theme zu adaptieren.12
- **Ansätze zur Integration**:
    1. **Farbschema-Erkennung**: Zumindest das helle/dunkle Farbschema des Systems kann oft über D-Bus (z.B. `org.freedesktop.appearance color-scheme` via Portal 105) oder plattformspezifische APIs abgefragt werden. Crates wie `dark-light` 24 können hier helfen, aber Änderungen zur Laufzeit zu erkennen, erfordert eventuell D-Bus-Signale oder andere Mechanismen.
    2. **Manuelles Parsen**: Theoretisch könnten GTK-/Qt-Theme-Dateien manuell geparst werden, um Farben, Schriftarten und Widget-Stile zu extrahieren. Dies ist jedoch extrem komplex und fehleranfällig aufgrund der Vielfalt und Komplexität der Theme-Engines.
    3. **Konsistenz durch Design**: Ein pragmatischerer Ansatz ist, ein eigenes, hochwertiges Theming-System für die Iced-DE zu entwickeln (wie COSMIC 17) und sicherzustellen, dass es gut konfigurierbar ist (z.B. Akzentfarben, Hell/Dunkel-Modus). Die visuelle Konsistenz beschränkt sich dann auf die DE-eigenen Anwendungen. Für GTK/Qt-Anwendungen, die innerhalb der DE laufen, müsste man sich auf deren Fähigkeit verlassen, ein grundlegendes Hell/Dunkel-Schema zu erkennen oder separate Themes bereitzustellen.

Die fehlende nahtlose Integration in bestehende Theme-Ökosysteme ist ein signifikanter Nachteil bei der Verwendung nicht-nativer Toolkits wie Iced für eine allgemeine Linux-Desktop-Umgebung.18 Es erfordert erheblichen Mehraufwand, um auch nur eine grundlegende visuelle Anpassung an das System zu erreichen.

### 5.5. Always-on-Top Fenster

Die Anforderung, ein Fenster immer im Vordergrund zu halten (z.B. für Widgets, OSDs 178 oder Panels), ist unter Wayland komplex.

- **Wayland-Protokolle**: Das Standard-`xdg-shell`-Protokoll sieht keine Funktion vor, mit der eine Anwendung ihr Fenster explizit "always-on-top" setzen kann.22 Dieses Verhalten wird als Aufgabe des Compositors/Fenstermanagers betrachtet.
- **Layer-Shell**: Das `wlr-layer-shell`-Protokoll 51 ermöglicht es, Oberflächen auf bestimmten Layern zu platzieren (z.B. `Overlay` oder `Top`), was einen ähnlichen Effekt erzielen kann. Dies ist der bevorzugte Weg für Panels, Docks oder Benachrichtigungen, die über anderen Fenstern liegen sollen. Wie in 3.2 diskutiert, erfordert dies spezielle Unterstützung im GUI-Toolkit bzw. dessen Backend.
- **Slint**: Die Slint-Dokumentation erwähnt eine `always-on-top`-Eigenschaft für `Window`-Elemente, weist aber darauf hin, dass diese unter Wayland (und teilweise macOS) nicht funktioniert oder nicht unterstützt wird.55 Die Implementierung hängt stark vom Backend ab.
- **Iced**: Iced selbst bietet keine direkte API für "always-on-top". Die Realisierung hängt davon ab, ob das zugrundeliegende Windowing-Backend (wie `winit` oder ein benutzerdefiniertes Backend) und der verwendete Wayland-Compositor dies über Protokolle wie Layer-Shell ermöglichen.
- **Compositor-spezifisch**: Einige Compositors (z.B. GNOME Mutter 179, KDE KWin) bieten möglicherweise interne Mechanismen oder benutzergesteuerte Optionen (z.B. über das Fenstermenü 179), um Fenster im Vordergrund zu halten, aber dies ist keine standardisierte Funktion, auf die sich Anwendungen verlassen können.

Für eine Iced/Rust DE wäre die zuverlässigste Methode, Always-on-Top-Verhalten über das `wlr-layer-shell`-Protokoll für dafür vorgesehene Komponenten (Panels, etc.) zu implementieren, was die in Abschnitt 3.2 genannten Herausforderungen bei der Iced-Integration mit sich bringt.

## 6. Konkreter Plan: Zusammenfassung & Empfehlungen

### 6.1. Phasierte Entwicklung

Ein Projekt dieser Größenordnung sollte schrittweise angegangen werden:

1. **Phase 1: Kern-Compositor**:
    - Implementierung eines minimalen Wayland-Compositors mit Smithay.
    - Fokus auf stabilem Rendering (z.B. mit GLES2-Backend), grundlegender Eingabeverarbeitung (Tastatur, Maus über libinput) und Output-Management für einen einzelnen Monitor.
    - Integration des Session-Managements (libseat) für den Standalone-Betrieb.
    - Ziel: Ein funktionierender, aber leerer Desktop.
2. **Phase 2: Grundlegende Shell**:
    - Entwicklung erster Shell-Komponenten mit Iced:
        - Ein einfaches Panel/Dock unter Verwendung von `iced_layershell` oder einem angepassten Backend (kritischer Punkt!).
        - Ein grundlegender Anwendungsstarter, der `.desktop`-Dateien parst und Anwendungen starten kann.
    - Rudimentäre Integration mit dem Compositor für Fensterfokus.
    - Implementierung der XWayland-Unterstützung im Compositor für Kompatibilität.
    - Ziel: Ein Desktop, auf dem Anwendungen gestartet und grundlegend verwaltet werden können.
3. **Phase 3: Essentielle Anwendungen**:
    - Entwicklung der Kernanwendungen mit Iced: Einstellungsanwendung (mit D-Bus-Backend für Systemdienste), Dateimanager (Grundfunktionen), Terminal-Emulator (basierend auf `iced_term`).
    - Implementierung eines Notification Daemons (D-Bus-Dienst + Iced-Popups oder Compositor-Integration).
    - Ziel: Ein benutzbarer Desktop mit grundlegenden Werkzeugen.
4. **Phase 4: Politur & Integration**:
    - Verfeinerung der UI/UX aller Komponenten.
    - Implementierung eines robusten Theming-Systems (inkl. dynamischem Hell/Dunkel-Modus, Akzentfarben). Untersuchung der System-Theme-Integration.
    - Erweiterung der Compositor-Funktionen (Multi-Monitor-Unterstützung, erweiterte Fensterverwaltung).
    - Vollständige Implementierung relevanter Freedesktop-Standards (Portals, MIME-Handling, etc.).
    - Fokus auf Barrierefreiheit (Accessibility).
    - Performance-Optimierung und Stabilitätsverbesserungen.
    - Ziel: Eine ausgereifte, alltagstaugliche Desktop-Umgebung.

### 6.2. Empfohlene Kernbibliotheken (Crates)

- **Compositor**: `smithay` 26
- **GUI Toolkit**: `iced` 7 (mit `iced_layershell` 19 oder angepasstem Backend für Panels/Docks)
- **Wayland Client**: `wayland-client` 183 (aus `wayland-rs`), `smithay-client-toolkit` 184 (falls benötigt für Shell-Komponenten-Interaktion)
- **IPC (D-Bus)**: `zbus` 83
- **Freedesktop Standards**:
    - Verzeichnisse: `xdg` 67 / `directories` 68
    - Desktop Entries: `freedesktop-desktop-entry` 65 (oder Alternativen 63)
    - Notifications (Client): `notify-rust` 129
    - Portals (Client): `ashpd` 104 / `xdg-portal` 72
- **Konfiguration**: `serde` 154, `toml` 154, `ron` 152
- **Systeminteraktion**: `libseat` (via Smithay), `udev` 187, `libinput` (via Smithay)

### 6.3. Umgang mit bekannten Herausforderungen

- **Iced & Wayland-Protokolle**: Die Integration von Iced mit `wlr-layer-shell` ist kritisch und muss frühzeitig adressiert werden. Die Optionen (`iced_layershell`, Fork, Custom Backend) müssen evaluiert und eine stabile Lösung implementiert werden. Ähnliches gilt für andere potenziell benötigte, nicht standardmäßige Protokolle.
- **Theming**: Eine klare Theming-Strategie ist von Beginn an notwendig. Der Grad der gewünschten Systemintegration muss definiert und der entsprechende Entwicklungsaufwand eingeplant werden. Eine robuste interne Theming-API in Iced ist die Grundlage.
- **Funktionsumfang**: Der Aufbau einer vollständigen DE ist ein Langzeitprojekt. Der Fokus sollte zunächst auf einem stabilen Kern und essentiellen Funktionen liegen. Iterative Entwicklung und die Nutzung vorhandener Komponenten (wo sinnvoll) sind ratsam.

### 6.4. Zukünftige Überlegungen

- **Barrierefreiheit (Accessibility)**: Muss von Anfang an mitgedacht werden. Iced hat dies auf der Roadmap.189 Die Integration mit Linux-Accessibility-Frameworks (wie AT-SPI über D-Bus) ist notwendig.
- **Sicherheit & Sandboxing**: Das Wayland-Sicherheitsmodell nutzen. XDG Portals 71 sind essentiell für die sichere Interaktion mit sandboxed Anwendungen (Flatpak).
- **Energiemanagement**: Integration mit `logind` (via D-Bus) oder ähnlichen Diensten zur Steuerung von Suspend, Hibernate etc.
- **Multi-GPU-Unterstützung**: Die Komplexität auf Systemen mit mehreren Grafikkarten (insbesondere Hybridgrafik) muss berücksichtigt werden. Smithay und die zugrundeliegenden Bibliotheken entwickeln hierfür Unterstützung.190

Dieser Plan legt eine solide technische Grundlage für die Entwicklung einer Rust- und Iced-basierten Desktop-Umgebung dar. Die Wahl von Smithay und Iced bietet Vorteile hinsichtlich Rust-Nativität und moderner Architektur, bringt aber auch spezifische Herausforderungen bei der Integration mit dem Wayland-Ökosystem mit sich, die gezielt angegangen werden müssen.