# Implementierungsplan Domänenschicht: Teil 1 – Kerndomäne, Konfiguration und Basissystemintegration

## I. Einleitung

### A. Zielsetzung des Dokuments

Dieses Dokument legt den detaillierten Implementierungsplan für den ersten Teil der Domänenschicht einer neu zu entwickelnden Softwareanwendung dar. Ziel ist es, eine robuste, wartbare und erweiterbare Grundlage für die Geschäftslogik zu schaffen. Der Fokus liegt auf der Definition der Kernkomponenten der Domäne, dem Aufbau eines soliden Konfigurationsmanagements, der Implementierung einer umfassenden Fehlerbehandlungs- und Logging-Strategie sowie der Integration grundlegender Systemdienste und der Etablierung von API-Designrichtlinien.

### B. Umfang und Abgrenzung

Teil 1 dieses Implementierungsplans umfasst die Entwicklung der fundamentalen Domänenlogik, einschließlich Entitäten, Wertobjekten und Repository-Schnittstellen. Des Weiteren werden das Konfigurationsmanagement, die Fehlerbehandlung und das Logging spezifiziert. Die Integration mit Systemdiensten beschränkt sich auf die Geheimnisverwaltung und die Grundlagen der Interprozesskommunikation (IPC) via D-Bus. Schließlich werden API-Designrichtlinien und projektinterne Konventionen festgelegt. Ausdrücklich nicht Teil dieses ersten Plans sind die Entwicklung der Benutzeroberfläche (UI), die Implementierung spezifischer Anwendungsdienste, die über die reine Domänenlogik hinausgehen, sowie alle Funktionalitäten, die für Teil 2 des Implementierungsplans vorgesehen sind.

### C. Zielgruppe

Dieses Dokument richtet sich primär an Softwareentwickler und -architekten, die an der Konzeption und Implementierung der Domänenschicht beteiligt sind. Es dient als technische Grundlage und Referenz für die Entwicklung.

### D. Methodik

Die Erstellung dieses Plans basiert auf einer sorgfältigen Analyse der funktionalen und nicht-funktionalen Anforderungen an die Domänenschicht. Dies beinhaltet die Auswertung von Best Practices im Bereich Softwarearchitektur und Domänendesign, insbesondere im Kontext der Programmiersprache Rust. Eine Auswahl etablierter Bibliotheken (Crates) und Technologien wurde getroffen, um eine hohe Qualität und Effizienz in der Entwicklung sicherzustellen.

## II. Grundlegende Design-Prinzipien und Architektur

### A. Architekturüberblick

Die Domänenschicht wird das Herzstück der Anwendung bilden und die Geschäftslogik sowie die Domänenobjekte kapseln. Sie wird streng nach den Prinzipien der Clean Architecture (alternativ als Hexagonal Architecture oder Ports & Adapters bekannt) entworfen. Dieses Architekturmuster gewährleistet eine klare Trennung der Belange und macht die Domänenschicht unabhängig von äußeren Schichten wie der Benutzeroberfläche, Datenbankimplementierungen oder externen Frameworks. Die Domänenschicht definiert ihre eigenen Schnittstellen (Ports), über die sie mit anderen Schichten kommuniziert, welche die entsprechenden Adapter implementieren. Diese Entkopplung fördert die Testbarkeit, Wartbarkeit und Flexibilität des Systems, da technologische Entscheidungen in den äußeren Schichten geändert werden können, ohne die Kernlogik der Domäne zu beeinträchtigen.

### B. Sprach- und Werkzeugwahl

Als primäre Programmiersprache für die Implementierung der Domänenschicht wurde Rust gewählt. Rust bietet eine einzigartige Kombination aus Performance, Speichersicherheit ohne Garbage Collector und modernen Sprachfeatures, die es besonders geeignet für systemnahe Programmierung und die Entwicklung komplexer, zuverlässiger Anwendungen machen.1 Die strengen Typsystem- und Ownership-Regeln von Rust helfen, viele gängige Programmierfehler bereits zur Kompilierzeit zu verhindern.3 Das Ökosystem von Rust, insbesondere die Verfügbarkeit hochwertiger Crates, unterstützt die schnelle Entwicklung robuster Software.

Für das Build-System und die Paketverwaltung wird Cargo, das offizielle Werkzeug von Rust, eingesetzt.5 Cargo vereinfacht das Kompilieren von Code, das Verwalten von Abhängigkeiten und das Ausführen von Tests erheblich. Zur Sicherstellung einer einheitlichen Codeformatierung wird `rustfmt` mit den Standardeinstellungen verbindlich eingesetzt. Zusätzlich wird `clippy`, ein Linter für Rust, verwendet, um idiomatischen und fehlerfreien Code zu fördern.6 Diese Werkzeuge tragen maßgeblich zur Codequalität und Lesbarkeit bei und erleichtern die Zusammenarbeit im Entwicklungsteam.

### C. Kernkonzepte der Domänenschicht

Die Domänenschicht wird um mehrere Kernkonzepte herum aufgebaut sein, die typisch für Domain-Driven Design (DDD) sind:

- **Entitäten (Entities):** Objekte, die nicht primär durch ihre Attribute, sondern durch eine eindeutige Identität und einen Lebenszyklus definiert sind. Änderungen an Entitäten werden über die Zeit verfolgt.
- **Wertobjekte (Value Objects):** Objekte, die einen beschreibenden Aspekt der Domäne darstellen und keine konzeptionelle Identität besitzen. Sie werden durch ihre Attribute definiert und sind typischerweise unveränderlich (immutable). Die Gleichheit von Wertobjekten basiert auf dem Vergleich ihrer Attribute.
- **Aggregate:** Eine Gruppe von Entitäten und Wertobjekten, die als eine einzelne Einheit für Datenänderungen behandelt werden. Jedes Aggregat hat eine Wurzelentität (Aggregate Root), die der einzige Einstiegspunkt für Modifikationen innerhalb des Aggregats ist und dessen Konsistenz sicherstellt.
- **Repositories (Schnittstellen):** Definieren Schnittstellen für den Zugriff auf und die Persistenz von Aggregaten. Sie abstrahieren die Details der Datenspeicherung und ermöglichen es der Domänenschicht, agnostisch gegenüber der konkreten Datenbanktechnologie zu bleiben.
- **Domänendienste (Domain Services):** Enthalten Domänenlogik, die nicht natürlich einer einzelnen Entität oder einem Wertobjekt zugeordnet werden kann. Solche Dienste operieren oft auf mehreren Domänenobjekten.
- **Domänenereignisse (Domain Events):** Repräsentieren signifikante Vorkommnisse innerhalb der Domäne, die für andere Teile des Systems von Interesse sein könnten.

Ein zentrales Element wird die Entwicklung einer **Ubiquitous Language** sein – einer gemeinsamen, präzisen Sprache, die von allen Teammitgliedern (Entwicklern und Domänenexperten) verwendet wird, um Konzepte der Domäne unmissverständlich zu beschreiben. Diese Sprache wird sich direkt im Code (Namen von Typen, Methoden, Modulen) widerspiegeln.

## III. Domänenmodell-Spezifikation

### A. Entitäten, Wertobjekte und Aggregate

Die korrekte Modellierung von Entitäten, Wertobjekten und Aggregaten ist von fundamentaler Bedeutung für die Fähigkeit der Anwendung, Geschäftskonzepte präzise abzubilden und zu manipulieren. Fehler in dieser frühen Phase können später zu erheblichem Refactoring-Aufwand führen.

**Entitäten** sind durch eine eindeutige Identität und einen Lebenszyklus gekennzeichnet. Ihre Attribute können sich im Laufe der Zeit ändern, aber ihre Identität bleibt bestehen. Ein Beispiel wäre ein `Benutzer` mit einer eindeutigen Benutzer-ID.

**Wertobjekte** hingegen haben keine eigene Identität über ihre Attribute hinaus. Sie beschreiben Merkmale und sind typischerweise unveränderlich. Ein `Geldbetrag` (bestehend aus Währung und Wert) oder eine `Adresse` (bestehend aus Straße, Stadt, Postleitzahl) sind klassische Beispiele. Würde eine Adresse fälschlicherweise als Entität modelliert, könnte dies zu unnötiger Komplexität bei der Identitätsverwaltung und bei Gleichheitsprüfungen führen, wo eigentlich nur ein struktureller Vergleich notwendig wäre.

**Aggregate** fassen Entitäten und Wertobjekte zu einer Konsistenzeinheit zusammen. Jedes Aggregat hat eine Wurzel (Aggregate Root), die als einziger Einstiegspunkt für Modifikationen dient und die Invarianten des Aggregats sicherstellt. Die Grenzen von Aggregaten müssen sorgfältig gewählt werden, um transaktionale Konsistenz zu gewährleisten und gleichzeitig eine übermäßige Kopplung zu vermeiden. Eine falsch gezogene Aggregatgrenze kann es erschweren, atomare Operationen konsistent durchzuführen oder führt zu unnötig großen Transaktionen.

Die folgende Tabelle fasst die initial identifizierten Kernentitäten zusammen:

**Tabelle 1: Kern-Domänenentitäten**

|   |   |   |   |
|---|---|---|---|
|**Entitätsname**|**Beschreibung**|**Schlüsselattribute**|**Identitätsmechanismus**|
|`BenutzerProfil`|Repräsentiert einen Benutzer des Systems.|`benutzer_id`, `email`|UUID (`benutzer_id`)|
|`Aufgabe`|Stellt eine zu erledigende Aufgabe dar.|`aufgaben_id`, `titel`|UUID (`aufgaben_id`)|
|`Projekt`|Gruppiert zusammengehörige Aufgaben.|`projekt_id`, `name`|UUID (`projekt_id`)|
|`Konfiguration`|Speichert benutzerspezifische Einstellungen.|`konfigurations_id`|UUID (`konfigurations_id`)|

Die Unterscheidung zwischen Entitäten und Wertobjekten ist oft subtil, aber entscheidend. Die nachstehende Tabelle listet wichtige Wertobjekte auf:

**Tabelle 2: Schlüssel-Wertobjekte**

|   |   |   |   |
|---|---|---|---|
|**Wertobjekt-Name**|**Beschreibung**|**Attribute**|**Unveränderlichkeits-Hinweise**|
|`EmailAdresse`|Repräsentiert eine E-Mail-Adresse.|`adresse` (String)|Ja, nach Erstellung fix.|
|`Zeitstempel`|Ein spezifischer Zeitpunkt.|`datum_uhrzeit` (z.B. `DateTime<Utc>`)|Ja, repräsentiert einen Punkt.|
|`Status`|Der Zustand einer Aufgabe oder eines Projekts.|`wert` (Enum: z.B. Offen, InArbeit)|Ja, Änderung erzeugt neuen Status.|
|`Prioritaet`|Die Dringlichkeit einer Aufgabe.|`stufe` (Enum: z.B. Niedrig, Hoch)|Ja.|

### B. Repository-Schnittstellen

Repository-Schnittstellen definieren die Verträge für den Zugriff auf und die Persistenz von Domänenobjekten, insbesondere Aggregaten. Sie werden als Rust-Traits implementiert und enthalten Methoden für typische CRUD-Operationen (Create, Read, Update, Delete) sowie ggf. spezifischere Abfragemethoden.

Ein fundamentaler Aspekt dieser Schnittstellen ist die Abstraktion von der konkreten Persistenztechnologie. Die Domänenschicht soll nicht wissen, ob Daten in einer SQL-Datenbank, einem NoSQL-Speicher oder einfachen Dateien abgelegt werden. Diese Unabhängigkeit wird erreicht, indem die Domänenschicht ausschließlich gegen die Repository-Traits programmiert wird. Die konkreten Implementierungen dieser Traits (z.B. ein `PostgresAufgabenRepository` oder ein `InMemoryBenutzerProfilRepository`) befinden sich außerhalb der Domänenschicht, typischerweise in der Infrastrukturschicht.

Diese Vorgehensweise bietet erhebliche Vorteile:

1. **Testbarkeit:** Für Unit-Tests der Domänenlogik können einfache In-Memory-Implementierungen der Repositories verwendet werden, was schnelle und isolierte Tests ohne externe Abhängigkeiten ermöglicht.
2. **Flexibilität:** Die Wahl der Datenbanktechnologie kann zu einem späteren Zeitpunkt geändert oder für verschiedene Umgebungen (z.B. Entwicklung vs. Produktion) unterschiedlich getroffen werden, ohne dass die Domänenschicht angepasst werden muss. Würde die Domänenschicht direkt Typen und Funktionen spezifischer Datenbank-Crates wie `sqlx` oder `diesel` verwenden, wäre sie fest an diese Technologie gebunden, was zukünftige Änderungen erschwert.

Beispielhafte Repository-Schnittstelle:

Rust

```
use std::error::Error;

// Platzhalter für Domänenfehler und Entität
type DomainError = Box<dyn Error + Send + Sync>;
struct Aufgabe { aufgaben_id: String, /*... */ }
struct Projekt { projekt_id: String, /*... */ }


pub trait AufgabenRepository {
    async fn finde_nach_id(&self, id: &str) -> Result<Option<Aufgabe>, DomainError>;
    async fn speichere(&self, aufgabe: &Aufgabe) -> Result<(), DomainError>;
    async fn loesche(&self, id: &str) -> Result<(), DomainError>;
    async fn finde_fuer_projekt(&self, projekt_id: &str) -> Result<Vec<Aufgabe>, DomainError>;
}

pub trait ProjektRepository {
    async fn finde_nach_id(&self, id: &str) -> Result<Option<Projekt>, DomainError>;
    async fn speichere(&self, projekt: &Projekt) -> Result<(), DomainError>;
}
```

### C. Serialisierung und Deserialisierung

Für die Umwandlung von Domänenobjekten in persistierbare Formate oder für die Übertragung über Prozessgrenzen hinweg (IPC) wird der `serde` Crate eingesetzt.7 `serde` ist der De-facto-Standard für Serialisierung und Deserialisierung im Rust-Ökosystem und bietet durch die Traits `Serialize` und `Deserialize` eine flexible und performante Lösung. Die aktuelle stabile Version (z.B. v1.0.219 gemäß 7) wird verwendet.

Die Wahl des konkreten Datenformats hängt vom Anwendungsfall ab:

- **JSON (JavaScript Object Notation):** Für menschenlesbare Konfigurationsdateien oder einfache IPC-Szenarien, bei denen Interoperabilität und Lesbarkeit im Vordergrund stehen.
- **CBOR (Concise Binary Object Representation) oder Bincode:** Für die effiziente binäre Persistenz großer Datenmengen oder performanzkritische IPC. Diese Formate sind kompakter und schneller zu verarbeiten als JSON, aber nicht direkt menschenlesbar.

Die Entscheidung für ein Format hat direkte Auswirkungen auf die Performance, die Speichergröße und die Debugfähigkeit. Beispielsweise wäre die Verwendung von JSON für große binäre Daten ineffizient, während ein benutzerdefiniertes Binärformat für Konfigurationsdateien deren manuelle Bearbeitung erschweren würde. `serde` stellt den Mechanismus bereit, die Wahl des Formats muss jedoch kontextspezifisch getroffen werden.

Bei der Verwendung von `serde` werden je nach Bedarf Features wie `std`, `derive`, `alloc` und `rc` aktiviert.8 Die `derive`-Makros (`#`) werden intensiv genutzt, um Boilerplate-Code für die Implementierung der Traits zu vermeiden.

## IV. Konfigurationsmanagement

### A. Struktur und Speicherort der Konfigurationsdateien

Die Konfiguration der Anwendung wird in Dateien im TOML-Format (Tom's Obvious, Minimal Language) gespeichert. TOML wurde aufgrund seiner guten Lesbarkeit und seiner Verbreitung im Rust-Ökosystem gewählt.9 Der `toml` Crate (z.B. Version 0.8.22 9) wird für das Parsen dieser Dateien verwendet. Die Konfigurationsdaten selbst werden in Rust-Strukturen abgebildet, die dann mittels `serde` deserialisiert werden.

Für die Ablage der Konfigurationsdateien wird die XDG Base Directory Specification 10 befolgt, um eine konsistente Benutzererfahrung auf Linux-Desktops zu gewährleisten. Der `directories` Crate (z.B. Version 6.0.0 11) wird verwendet, um die standardisierten Pfade systemunabhängig zu ermitteln:

- **Benutzerspezifische Konfiguration:** `$XDG_CONFIG_HOME/your_app_name/config.toml`. Falls `$XDG_CONFIG_HOME` nicht gesetzt ist, wird standardmäßig `$HOME/.config/your_app_name/config.toml` verwendet.
- **Systemweite Konfiguration (falls anwendbar):** `/etc/your_app_name/config.toml`.

Es wird eine klare Präzedenz festgelegt, wobei benutzerspezifische Konfigurationen systemweite Einstellungen überschreiben. Die Einhaltung der XDG-Standards ist von Bedeutung, da Benutzer erwarten, Konfigurationsdateien an vorhersagbaren Orten zu finden. Dies erleichtert die Verwaltung für Endanwender und Systemadministratoren und sorgt dafür, dass sich die Anwendung wie andere gut integrierte Linux-Anwendungen verhält.

Die Konfiguration kann bei Bedarf in logische Abschnitte innerhalb der `config.toml`-Datei unterteilt werden. Für den Anfang wird eine einzelne Konfigurationsdatei als ausreichend erachtet.

### B. Laden und Validieren der Konfiguration

Das Laden der Konfiguration erfolgt beim Anwendungsstart. Der Inhalt der TOML-Datei wird mithilfe des `toml` Crates und `serde` in die dafür vorgesehenen Rust-Strukturen deserialisiert.

Ein entscheidender Schritt ist die Validierung der geladenen Konfigurationswerte. Diese Validierung umfasst beispielsweise Bereichsprüfungen für numerische Werte, Formatprüfungen für Zeichenketten oder die Überprüfung der Existenz referenzierter Ressourcen. Hierfür können entweder benutzerdefinierte Validierungsfunktionen direkt in den Konfigurationsstrukturen oder, bei höherer Komplexität, ein dedizierter Validierungs-Crate implementiert werden.

Das Verhalten bei fehlgeschlagener Validierung wird klar definiert: Die Anwendung soll in einem solchen Fall mit einer aussagekräftigen Fehlermeldung beendet werden ("fail fast"). Dies ist vorzuziehen gegenüber einem Betrieb mit potenziell inkonsistenten oder fehlerhaften Einstellungen, der zu unvorhersehbarem Verhalten und schwer diagnostizierbaren Fehlern führen kann.

Standardwerte für Konfigurationsparameter werden über Implementierungen des `Default`-Traits für die Konfigurationsstrukturen oder durch explizite Fallback-Werte im Code bereitgestellt. `serde`-Attribute wie `#[serde(default)]` können hierbei nützlich sein, um das Verhalten bei fehlenden Feldern in der TOML-Datei zu steuern.

Die Frage der dynamischen Neuladung (Hot-Reloading) von Konfigurationsdateien während der Laufzeit wird als optionale, fortgeschrittene Anforderung betrachtet und ist, falls notwendig, eher Teil von Implementierungsplan Teil 2.

Die folgende Tabelle dient als Referenz für die konfigurierbaren Parameter der Domänenschicht:

**Tabelle 3: Konfigurationsparameter**

|   |   |   |   |   |
|---|---|---|---|---|
|**Parametername (Pfad in TOML)**|**Datentyp**|**Beschreibung**|**Standardwert**|**Validierungsregeln**|
|`logging.level`|String|Globaler Log-Level (z.B. "INFO", "DEBUG")|"INFO"|Muss gültiger Log-Level sein.|
|`database.url`|String|Verbindungs-URL zur Datenbank.|""|Muss valides URL-Format haben.|
|`secrets.collection_name`|String|Name der Secret-Service-Kollektion.|"default"|Darf nicht leer sein.|
|`performance.thread_pool_size`|u32|Größe des Thread-Pools für Hintergrundaufgaben.|4|Muss > 0 und < 1024 sein.|

## V. Fehlerbehandlung und Logging

### A. Fehlerarten und -hierarchie

Eine robuste Fehlerbehandlung ist essentiell für die Stabilität und Wartbarkeit der Domänenschicht. Es werden benutzerdefinierte Fehlertypen für die Domänenschicht definiert, wobei spezifische Fehler-Enums gegenüber generischen Fehler-Strings bevorzugt werden.

Der `thiserror` Crate (z.B. Version 2.0.12 12) wird zur Definition dieser Fehler-Enums verwendet. `thiserror` vereinfacht die Erstellung idiomatischer Fehlertypen erheblich:

- Das Attribut `#[error("...")]` wird genutzt, um menschenlesbare `Display`-Implementierungen für Fehler zu generieren.
- Mittels `#[from]` können zugrundeliegende Fehler (z.B. `std::io::Error`, `serde_json::Error`) automatisch in spezifische Varianten des Domänenfehlers konvertiert werden.
- Das Attribut `#[source]` ermöglicht die Verkettung von Fehlern, um den ursprünglichen Kontext für eine bessere Diagnose zu bewahren.

Bei der Definition der Fehler wird eine ausgewogene Granularität angestrebt.14 Fehler sollten spezifisch genug sein, um vom aufrufenden Code sinnvoll behandelt werden zu können, aber nicht so zahlreich, dass die Fehlerbehandlung unübersichtlich wird. Es ist denkbar, Fehler-Enums auf Modulebene zu definieren, die bei Bedarf in einem übergeordneten Domänenfehler-Enum aggregiert werden.

Ein grundlegendes Prinzip ist, dass die Domänenschicht als Bibliothek bei wiederherstellbaren Fehlern nicht paniken darf.15 Stattdessen wird `Result<T, E>` zurückgegeben. Panics sind ausschließlich für nicht wiederherstellbare Zustände reserviert, die auf einen Programmierfehler hindeuten (z.B. gebrochene Invarianten).

Gemäß den Rust API Guidelines 16 werden alle Fehlertypen die Traits `std::error::Error` und `std::fmt::Debug` implementieren (C-GOOD-ERR, C-DEBUG). Die Verwendung von `thiserror` stellt sicher, dass diese Implementierungen korrekt und mit minimalem Boilerplate-Code generiert werden, was die Ergonomie der Fehlerbehandlung sowohl für Entwickler der Domänenschicht als auch für deren Konsumenten signifikant verbessert.

Die folgende Tabelle listet beispielhaft Domänenfehlertypen auf:

**Tabelle 4: Domänenfehlertypen**

|   |   |   |   |
|---|---|---|---|
|**Fehler-Enum-Variante**|**Assoziierte Daten**|**Beschreibung (Wann tritt er auf)**|**#[source] (falls zutreffend)**|
|`KonfigurationNichtGefunden`|`Pfad: String`|Die Konfigurationsdatei konnte am erwarteten Ort nicht gefunden werden.|`std::io::Error`|
|`KonfigurationUngueltig`|`Fehler: String`|Die Konfigurationsdatei ist fehlerhaft oder enthält ungültige Werte.|`toml::de::Error`|
|`DatenbankFehler`|`Ursache: String`|Ein allgemeiner Fehler bei der Datenbankinteraktion.|Spezifischer DB-Fehler|
|`EntitaetNichtGefunden`|`Id: String, Typ: String`|Eine angeforderte Entität konnte nicht gefunden werden.|-|
|`GeheimnisdienstFehler`|`Meldung: String`|Fehler bei der Interaktion mit dem Secret Service.|`secret_service::Error`|

### B. Logging-Strategie und -Implementierung

Für ein strukturiertes und kontextbezogenes Logging wird das `tracing` Ökosystem eingesetzt, bestehend aus dem `tracing` Crate 17 und dem `tracing-subscriber` Crate.19 `tracing` bietet gegenüber dem einfacheren `log` Crate den Vorteil, reichhaltigere diagnostische Informationen zu liefern, insbesondere in asynchronen Kontexten oder komplexen Arbeitsabläufen.

Es wird zwischen `Events` (zeitpunktbezogene Ereignisse) und `Spans` (zeitraumbezogene Kontexte) unterschieden.17 Spans ermöglichen es, den Ausführungsfluss und die Kausalität von Ereignissen besser nachzuvollziehen. Das Makro `tracing::instrument` wird verwendet, um Funktionen einfach mit Spans zu versehen.

Standard-Log-Level (TRACE, DEBUG, INFO, WARN, ERROR) werden definiert und konsistent verwendet. Das Logging erfolgt strukturiert, d.h., Log-Ereignisse werden mit Schlüssel-Wert-Paaren versehen, um die maschinelle Verarbeitung und Analyse durch Log-Management-Tools zu erleichtern.

Die Konfiguration des Loggings erfolgt über `tracing-subscriber`. Die `fmt`-Schicht dient als gängiger Ausgangspunkt für die Formatierung der Log-Ausgaben (z.B. einfacher Text, JSON) und die Steuerung des Outputs (z.B. stdout, Datei). Der `env-filter` ermöglicht die Steuerung der Log-Level über eine Umgebungsvariable wie `RUST_LOG`.20 Diese flexible Konfiguration erlaubt es, die Ausführlichkeit der Logs ohne Codeänderungen an verschiedene Umgebungen (Entwicklung, Produktion) anzupassen.

Es wird sichergestellt, dass Logs relevante kontextuelle Informationen enthalten, wie z.B. Request-IDs, Benutzer-IDs (falls zutreffend) und Span-IDs, um die Fehlersuche zu unterstützen. Bei der Implementierung des Loggings werden Performance-Aspekte berücksichtigt. Obwohl `tracing` auf Effizienz ausgelegt ist, kann exzessives Logging, insbesondere auf hohen Verbositätsstufen, die Anwendungsleistung beeinträchtigen.

## VI. Integration mit Systemdiensten

### A. Geheimnisverwaltung (Secret Management)

Für die sichere Speicherung sensibler Daten wie API-Schlüssel oder Passwörter wird die Freedesktop Secret Service API genutzt. Die Integration erfolgt über den `secret-service-rs` Crate (z.B. Version 5.0.0 23). Dieser Crate ermöglicht die Interaktion mit Diensten wie GNOME Keyring oder KWallet.

Die Verwendung von `secret-service-rs` setzt eine D-Bus-Verbindung und eine asynchrone Laufzeitumgebung (Async Runtime) voraus. Die Wahl des spezifischen Runtime-Features des Crates (z.B. `rt-tokio-crypto-rust` oder `rt-async-io-crypto-openssl` 24) muss mit der übergreifenden asynchronen Strategie des Projekts abgestimmt sein. Die Kernoperationen umfassen das sichere Speichern, Abrufen und Löschen von Geheimnissen. Diese Geheimnisse werden in Kollektionen organisiert, wobei typischerweise die Standardkollektion verwendet wird. Ein typisches Nutzungsmuster beinhaltet das Verbinden zum `SecretService`, das Abrufen der Standardkollektion und dann das Erstellen, Suchen, Abrufen oder Löschen von Items, die ein Label, Attribute (für die Suche) und die eigentliche geheime Nutzlast besitzen.23

Fehler, die vom `secret-service` Crate zurückgegeben werden, müssen in domänenspezifische Fehlertypen der Anwendung verpackt werden, um eine konsistente Fehlerbehandlung zu gewährleisten. Die Abhängigkeit vom Freedesktop Secret Service bedeutet, dass die Anwendung auf Linux-Desktop-Umgebungen angewiesen ist, die diesen Dienst bereitstellen. Für die Portabilität auf Nicht-Linux-Plattformen oder in Headless-Umgebungen wären alternative Strategien zur Geheimnisverwaltung erforderlich, was potenziell plattformspezifische Abstraktionen nach sich zieht, falls plattformübergreifende Unterstützung ein langfristiges Ziel ist.

### B. Interprozesskommunikation (IPC) Grundlagen

D-Bus wird als fundamentales IPC-Mechanismus auf Linux-Desktops anerkannt 25, zumal die Geheimnisverwaltung über `secret-service-rs` bereits darauf basiert. Sollte die Domänenschicht direkt mit anderen D-Bus-Diensten interagieren müssen, die über die Abstraktionen von `secret-service-rs` hinausgehen, oder eine eigene D-Bus-Schnittstelle bereitstellen, käme der `dbus` Crate (z.B. Version 0.9.7 26) zum Einsatz. Dies würde ein Verständnis von D-Bus-Objekten, -Methoden, -Signalen und -Schnittstellen erfordern.

Die Nutzung von D-Bus für die Geheimnisverwaltung und potenziell andere IPC-Aufgaben führt eine asynchrone Abhängigkeit ein. Die Domänenschicht, oder zumindest die Teile, die mit D-Bus interagieren, müssen asynchron-fähig sein. Dies beeinflusst die Wahl der Rust Async Runtime (z.B. Tokio, async-std) für das gesamte Projekt, da D-Bus-Operationen inhärent asynchron sind und der `secret-service` Crate dies widerspiegelt.

Es ist entscheidend, eine robuste Fehlerbehandlung für Szenarien zu implementieren, in denen D-Bus-Dienste nicht verfügbar sind oder fehlerhaft reagieren. Die Domänenschicht kann nicht davon ausgehen, dass diese Dienste immer perfekt funktionieren, und muss Verbindungsfehler, Zeitüberschreitungen und dienstspezifische Fehler adäquat behandeln, beispielsweise durch das Deaktivieren abhängiger Funktionen oder durch informative Fehlermeldungen an den Benutzer.

Obwohl viele Wayland-Protokolle (z.B. `xdg-decoration` 27, `wlr-foreign-toplevel-management` 28) primär der UI- und Kompositor-Interaktion dienen und eine Form von IPC darstellen, wird die Domänenschicht diese nicht direkt implementieren. Sie könnte jedoch Daten bereitstellen oder auf Ereignisse reagieren, die von höheren Schichten orchestriert werden, welche diese Protokolle nutzen. Ähnlich verhält es sich mit XDG Desktop Portals 30, die sandboxed Anwendungen den Zugriff auf Systemressourcen über D-Bus ermöglichen; eine Interaktion hiermit würde ebenfalls auf einer höheren Ebene als der Domänenschicht stattfinden.

## VII. API-Design-Richtlinien und Best Practices

### A. Rust API Guidelines

Die offiziellen Rust API Guidelines 16 werden als verbindliche Grundlage für das Design aller öffentlichen Schnittstellen der Domänenschicht übernommen. Eine konsequente Einhaltung dieser Richtlinien ist ein starker Indikator für die langfristige Nutzbarkeit und Wartbarkeit einer Bibliothek, da sie die kognitive Last für Entwickler reduziert und Konsistenz fördert.

Besonderer Wert wird auf folgende Bereiche gelegt 16:

- **Namensgebung (C-CASE, C-CONV, C-GETTER, C-ITER, C-ITER-TY):** Einheitliche Verwendung von `snake_case` für Funktionen/Variablen und `PascalCase` für Typen. Standardisierte Namen für Konvertierungsmethoden (`as_`, `to_`, `into_`), Getter-Konventionen und Iterator-Benennungen.
- **Interoperabilität (C-COMMON-TRAITS, C-CONV-TRAITS, C-SEND-SYNC, C-GOOD-ERR):** Implementierung gängiger Traits wie `Debug`, `Clone`, `Eq`, `PartialEq`, `Default` sowie `Send` und `Sync`, wo angebracht. Verwendung von Standard-Konvertierungstraits (`From`, `AsRef`). Sicherstellung, dass Fehlertypen sich gut verhalten.
- **Dokumentation (C-CRATE-DOC, C-EXAMPLE, C-FAILURE, C-LINK, C-METADATA):** Umfassende Dokumentation auf Crate- und Item-Ebene, Beispiele für alle öffentlichen Elemente, Dokumentation von Fehlerbedingungen (Errors, Panics). Vollständige Metadaten in `Cargo.toml`.
- **Vorhersagbarkeit (C-METHOD, C-CTOR):** Funktionen mit einem klaren Empfänger (Receiver) werden als Methoden implementiert. Konstruktoren sind statische, inhärente Methoden (z.B. `new()`).
- **Flexibilität (C-GENERIC, C-CUSTOM-TYPE):** Einsatz von Generics, wo sinnvoll. Verwendung spezifischer Typen für Argumente anstelle von Booleans oder `Option`-Typen, um Bedeutung zu transportieren.
- **Verlässlichkeit (C-VALIDATE, C-DTOR-FAIL):** Funktionen validieren ihre Argumente. Destruktoren dürfen nicht fehlschlagen.
- **Debugfähigkeit (C-DEBUG, C-DEBUG-NONEMPTY):** Alle öffentlichen Typen implementieren `Debug`.
- **Zukunftssicherheit (C-STRUCT-PRIVATE, C-NEWTYPE-HIDE):** Strukturfelder sind standardmäßig privat, um interne Änderungen ohne API-Bruch zu ermöglichen. Newtypes kapseln Implementierungsdetails. Diese "Future Proofing"-Richtlinien sind besonders wichtig für eine Domänenschicht, da sich deren Kerndatenstrukturen weiterentwickeln könnten. Kapselung erlaubt solche Änderungen mit minimalen Auswirkungen auf abhängigen Code.

### B. Projektinterne Konventionen

Zusätzlich zu den offiziellen Rust API Guidelines werden folgende projektinterne Konventionen festgelegt:

- **Modulstruktur:** Eine standardisierte Modulstruktur für die Domänenschicht (z.B. `entities/`, `repositories/`, `services/`, `errors.rs`, `config.rs`).
- **Fehlerbehandlung:** Konsequente Anwendung der in Abschnitt V.A beschriebenen Muster unter Verwendung von `thiserror`.
- **Logging:** Konsequente Anwendung der in Abschnitt V.B beschriebenen Muster unter Verwendung von `tracing` für strukturiertes Logging.
- **Teststrategie:** Unit-Tests werden direkt neben dem zu testenden Code platziert. Integrationstests für Repositories verwenden Mocks oder In-Memory-Implementierungen.
- **Codeformatierung:** Die Verwendung von `rustfmt` mit den Standardeinstellungen ist obligatorisch.6
- **Clippy Lints:** Ein strenger Satz von Clippy-Lints wird durchgesetzt, um die Codequalität weiter zu erhöhen.
- **Sichere Programmierpraktiken 4:**
    - Die Verwendung von `unsafe`-Blöcken wird minimiert. Jeder Einsatz erfordert eine gründliche Überprüfung und stichhaltige Begründung.
    - Alle Daten, die Vertrauensgrenzen überschreiten, müssen validiert werden (obwohl die Domänenschicht idealerweise bereits validierte Daten von Anwendungsdiensten erhalten sollte).
    - Abhängigkeiten werden regelmäßig mit `cargo update` aktualisiert und mit Werkzeugen wie `cargo audit` auf bekannte Sicherheitslücken überprüft. Die Integration dieser Sicherheitspraktiken direkt in die Entwicklungsrichtlinien, anstatt sie als nachträglichen Gedanken zu behandeln, ist essentiell für den Aufbau eines vertrauenswürdigen Systems. Rusts Features (Ownership, Typsystem) bieten eine starke Grundlage, erfordern aber dennoch bewusste Anstrengungen.

### C. Versionsmanagement und Branching-Strategie

Für die Versionierung der Domänenschicht-Crate, insbesondere wenn sie unabhängig veröffentlicht oder versioniert wird, wird Semantic Versioning (SemVer) angewendet.

Als Git-Branching-Modell wird GitHub Flow empfohlen.33 GitHub Flow ist einfacher als GitFlow und eignet sich gut für kontinuierliche Integration und Auslieferung (CI/CD). Es basiert auf einem Haupt-Branch (z.B. `main`), von dem Feature-Branches abgeleitet werden. Nach Abschluss und Review werden diese Feature-Branches direkt zurück in den Haupt-Branch gemerged. Dies fördert schnelle Iterationen und eine stets auslieferungsbereite Codebasis.

## VIII. Anhänge

### A. Glossar

- **Aggregat (Aggregate):** Eine Gruppe von Entitäten und Wertobjekten, die als eine einzelne Einheit für Datenänderungen behandelt wird, mit einer Wurzelentität, die Konsistenz sicherstellt.
- **Clean Architecture:** Ein Softwarearchitekturmuster, das auf der Trennung von Belangen basiert und die Unabhängigkeit der Geschäftslogik von äußeren Schichten wie UI und Datenbank betont.
- **D-Bus:** Ein Interprozesskommunikationssystem, das auf Linux-Systemen weit verbreitet ist.
- **Domänenschicht (Domain Layer):** Der Teil einer Anwendung, der die Kernlogik und die Geschäftsregeln enthält.
- **Entität (Entity):** Ein Objekt in der Domäne, das durch eine eindeutige Identität und einen Lebenszyklus definiert ist.
- **Freedesktop Secret Service API:** Eine standardisierte Schnittstelle unter Linux zur sicheren Speicherung von Geheimnissen.
- **IPC (Inter-Process Communication):** Kommunikation zwischen verschiedenen Prozessen.
- **Repository:** Eine Abstraktion, die den Zugriff auf und die Persistenz von Domänenobjekten kapselt.
- **Rust:** Eine Systemprogrammiersprache, die auf Sicherheit und Performance ausgelegt ist.
- **serde:** Ein populärer Rust-Crate für Serialisierung und Deserialisierung.
- **SemVer (Semantic Versioning):** Ein Standard für die Versionierung von Software.
- **thiserror:** Ein Rust-Crate zur einfachen Erstellung von Fehler-Enums.
- **TOML (Tom's Obvious, Minimal Language):** Ein Konfigurationsdateiformat.
- **tracing:** Ein Rust-Framework für instrumentiertes, strukturiertes Logging.
- **Ubiquitous Language:** Eine gemeinsame Sprache, die von Entwicklern und Domänenexperten verwendet wird, um Domänenkonzepte präzise zu beschreiben.
- **Wertobjekt (Value Object):** Ein Objekt, das einen beschreibenden Aspekt der Domäne darstellt und keine Identität über seine Attribute hinaus besitzt; typischerweise unveränderlich.
- **XDG Base Directory Specification:** Ein Standard von freedesktop.org, der festlegt, wo benutzerspezifische Daten- und Konfigurationsdateien gespeichert werden sollen.

### B. Referenzierte Crates und Versionen

Die Entwicklung der Domänenschicht (Teil 1) wird auf der stabilen Rust-Version 1.85.0 (oder neuer, falls zum Entwicklungsstart verfügbar) und der Rust 2024 Edition basieren.35 Die Wahl einer aktuellen Edition ermöglicht die Nutzung der neuesten Sprachfeatures und Idiome. Die Pflege einer Liste der referenzierten Crates und ihrer Versionen ist entscheidend für die Reproduzierbarkeit von Builds und das Management von Abhängigkeitsupdates.

**Tabelle 7: Externe Crate-Abhängigkeiten (Teil 1)**

|   |   |   |   |   |
|---|---|---|---|---|
|**Crate-Name**|**Version (Beispiel)**|**Lizenz**|**Hauptzweck in der Domänenschicht**|**Genutzte Schlüsselfunktionen**|
|`serde`|1.0.219|MIT/Apache-2.0|Serialisierung und Deserialisierung von Datenstrukturen.|`Serialize`, `Deserialize` Traits, `derive` Makros|
|`thiserror`|2.0.12|MIT/Apache-2.0|Ergonomische Definition von benutzerdefinierten Fehlertypen.|`#[derive(Error)]`, `#[error(...)]`, `#[from]`, `#[source]`|
|`tracing`|0.1.x|MIT|Strukturiertes, kontextbezogenes Logging und Tracing.|`span!`, `event!`, `#[instrument]`|
|`tracing-subscriber`|0.3.x|MIT|Konfiguration von Logging-Ausgabe und -Filterung.|`fmt::Layer`, `EnvFilter`|
|`toml`|0.8.22|MIT/Apache-2.0|Parsen und Serialisieren von Konfigurationsdateien im TOML-Format.|`from_str`, `to_string` (via `serde`)|
|`directories`|6.0.0|MIT/Apache-2.0|Ermittlung von Standardverzeichnispfaden (XDG).|`ProjectDirs`, `UserDirs`, `BaseDirs`|
|`secret-service`|5.0.0|MIT/Apache-2.0|Sichere Speicherung und Abruf von Geheimnissen via D-Bus.|`SecretService::connect`, Collection- und Item-Operationen|
|`dbus` (optional)|0.9.7|MIT/Apache-2.0|Direkte D-Bus Interprozesskommunikation (falls benötigt).|Verbindungshandling, Methodenaufrufe, Signalempfang|

## IX. Schlussfolgerungen

Der vorliegende Implementierungsplan für Teil 1 der Domänenschicht legt eine solide Basis für die Entwicklung einer robusten und wartbaren Anwendung. Die Wahl von Rust als Programmiersprache, kombiniert mit einer klaren Architektur nach den Prinzipien der Clean Architecture, verspricht eine hohe Codequalität und Performance. Die konsequente Nutzung etablierter Crates wie `serde` für die Datenverarbeitung, `thiserror` für eine präzise Fehlerbehandlung und `tracing` für ein aufschlussreiches Logging wird die Entwicklungseffizienz steigern und die Diagnosefähigkeit des Systems verbessern.

Die Standardisierung des Konfigurationsmanagements mittels TOML und der XDG Base Directory Specification sowie die Integration mit dem Freedesktop Secret Service für die Geheimnisverwaltung gewährleisten eine gute Einbettung in Linux-Desktop-Umgebungen. Die definierten API-Designrichtlinien und projektinternen Konventionen, einschließlich der Betonung sicherer Programmierpraktiken, werden zur langfristigen Stabilität und Sicherheit der Domänenschicht beitragen.

Durch die klare Abgrenzung der Verantwortlichkeiten und die Schaffung testbarer Komponenten wird eine hohe Softwarequalität angestrebt. Die in diesem Plan getroffenen Entscheidungen zielen darauf ab, eine Domänenschicht zu schaffen, die nicht nur die aktuellen Anforderungen erfüllt, sondern auch flexibel genug ist, um zukünftige Erweiterungen und Änderungen aufzunehmen. Die sorgfältige Modellierung der Domänenkonzepte und die Abstraktion von externen Abhängigkeiten sind hierbei Schlüsselfaktoren für den Erfolg.

# Domänenschicht Implementierungsplan (Ultra-Feinspezifikation)

## 1. Grundlagen und Architektur der Domänenschicht

Die Domänenschicht bildet das Herzstück der Desktop-Umgebung und beinhaltet die Kernlogik sowie die Geschäftsregeln. Ihre Hauptverantwortung liegt in der Verwaltung von Workspaces ("Spaces"), dem Theming-System, der Logik für KI-Interaktionen einschließlich des Einwilligungsmanagements, der Verwaltung von Benachrichtigungen und der Definition von Richtlinien für das Fenstermanagement (z.B. Tiling-Regeln). Ein fundamentaler Aspekt dieser Schicht ist ihre Unabhängigkeit von spezifischen UI-Implementierungen oder Systemdetails wie D-Bus oder Wayland. Sie nutzt Funktionalitäten der Kernschicht und stellt Logik sowie Zustand für die System- und Benutzeroberflächenschicht bereit.

### 1.1. Programmiersprache und Entwicklungsumgebung

- **Sprache:** Rust (Version 1.85.0 oder neuer, Stand Februar 2025 1). Die Wahl von Rust begründet sich durch dessen Fokus auf Speichersicherheit ohne Garbage Collector, exzellente Performance und moderne Concurrency-Features, was für ein System wie eine Desktop-Umgebung von entscheidender Bedeutung ist.2
- **Build-System:** Cargo, das Standard-Build-System und Paketmanager für Rust.5 Meson wird zwar als fähig erachtet, Rust-Projekte zu handhaben 6, jedoch ist Cargo die natürliche Wahl im Rust-Ökosystem.
- **Entwicklungsrichtlinien:**
    - Rust API Guidelines Checklist 7: Strikte Einhaltung dieser Richtlinien für Namenskonventionen (C-CASE, C-CONV, C-GETTER, C-ITER), Interoperabilität (C-COMMON-TRAITS, C-SEND-SYNC), Dokumentation (C-CRATE-DOC, C-EXAMPLE) und weitere Aspekte.
    - Rust Style Guide 8: Formatierungskonventionen (Einrückung, Zeilenlänge), Kommentierungsstil.
    - Secure Coding Practices in Rust 9: Minimierung von `unsafe` Blöcken, sorgfältige Prüfung von Abhängigkeiten, Validierung von Eingabedaten (obwohl die Domänenschicht primär interne Daten verarbeitet, ist das Prinzip wichtig).
    - Git Branching Modell: GitHub Flow wird für seine Einfachheit und den Fokus auf Continuous Delivery bevorzugt, besonders wenn schnelle Iterationen und häufige Releases angestrebt werden.10 Feature-Branches werden von `main` erstellt und nach Fertigstellung und Review direkt zurück in `main` gemerged. Für stabilere Release-Zyklen könnte GitFlow in Betracht gezogen werden, aber für die iterative Entwicklung einer neuen Desktop-Umgebung erscheint GitHub Flow agiler.

Die Entscheidung für Rust basiert auf dessen Fähigkeit, systemnahe Software zu entwickeln, die sowohl sicher als auch performant ist. Die strengen Compiler-Prüfungen von Rust helfen, viele gängige Fehlerklassen bereits zur Compile-Zeit zu eliminieren, was die Stabilität der Domänenschicht maßgeblich erhöht. Die Einhaltung etablierter Entwicklungsrichtlinien stellt sicher, dass der Code verständlich, wartbar und konsistent bleibt.

### 1.2. Kernabhängigkeiten und Basistechnologien

Die Domänenschicht wird so entworfen, dass sie minimale direkte Abhängigkeiten zu externen Systembibliotheken hat. Notwendige Interaktionen mit dem System (Dateizugriff, Prozessmanagement, etc.) erfolgen über Abstraktionen (Ports), die von der Kernschicht implementiert werden.

- **1.2.1. Externe Rust-Crates**
    - Eine detaillierte Liste der verwendeten Crates mit Versionen und Features findet sich in Anhang C.
    - Die Auswahl der Crates folgt dem Prinzip der Stabilität, Verbreitung und Wartungsfreundlichkeit.

|   |   |   |   |
|---|---|---|---|
|**Crate-Name**|**Version (Beispielhaft)**|**Kurzbeschreibung/Zweck**|**Relevante Features (Beispielhaft)**|
|`serde`|`1.0.219` 12|Serialisierung und Deserialisierung von Datenstrukturen.|`derive`|
|`thiserror`|`2.0.12` 14|Einfache Erstellung von benutzerdefinierten Fehlertypen.||
|`tracing`|`0.1.40` 15|Framework für anwendungsspezifische Diagnoseaufzeichnungen (Logging und Tracing).||
|`tracing-subscriber`|`0.3.18` 15|Implementierungen für `tracing` Subscriber (z.B. für formatiertes Logging).|`env-filter`, `fmt`|
|`toml`|`0.8.22` 18|Parsen von TOML-Konfigurationsdateien (indirekt, da Domäne geparste Daten erhält).||
|`uuid`|`1.8.0` (optional)|Generierung und Handhabung von UUIDs für eindeutige IDs.|`v4`, `serde`|
|`log`|`0.4.x`|Logging-Fassade, falls `tracing-log` verwendet wird.||
|`directories-next`|`2.0.0` (oder `directories` v6.0.0 19)|Auflösung von XDG-Standardverzeichnissen (indirekt, durch Kernschicht genutzt).||

- **1.2.2. Fehlerbehandlungsstrategie**
    
    - Verwendung des `thiserror` Crates (Version 2.0.12 14) zur Definition von benutzerdefinierten Fehlertypen.
    - Jedes Modul der Domänenschicht definiert sein eigenes spezifisches Error-Enum (z.B. `WorkspaceError`, `ThemingError`).
    - Ein globales `DomainError` Enum fasst alle modulspezifischen Fehler zusammen (siehe Abschnitt 7).
    - Die `#[from]` Annotation wird genutzt, um Fehler aus abhängigen Operationen (z.B. IO-Fehler aus der Kernschicht, die durchgereicht werden) elegant in domänenspezifische Fehler umzuwandeln.20 Dies vermeidet generische Fehlertypen und macht den Code für Aufrufer verständlicher und besser handhabbar. Die klare Strukturierung von Fehlern, beginnend bei spezifischen Fehlern pro Operation oder Modul und aggregiert in einem übergeordneten `DomainError`, erleichtert sowohl die Fehlerbehandlung innerhalb der Domänenschicht als auch die Kommunikation von Fehlern an höhere Schichten.21
- **1.2.3. Logging- und Tracing-Strategie**
    
    - Verwendung des `tracing` Crates (Core: `tracing-core` Version ~0.1.31+ 16, Subscriber: `tracing-subscriber` Version ~0.3.17+ 15).
    - Strukturierte Logs mit Span-basiertem Tracing zur Nachverfolgung von Abläufen über Modulgrenzen hinweg. Die `tracing` Bibliothek ist dem `log` Crate vorzuziehen, da sie durch Spans zusätzlichen Kontext für Diagnoseinformationen bereitstellt, was besonders bei der Analyse komplexer Abläufe in der Domänenschicht vorteilhaft ist.22
    - Konfigurierbare Loglevel (TRACE, DEBUG, INFO, WARN, ERROR).
    - Die Domänenschicht emittiert Traces und Logs; die Konfiguration des Subscribers (z.B. Format, Output) obliegt der Anwendungsschicht oder dem Hauptprogramm.
    - Die Notwendigkeit eines "lückenlosen Entwickler-Implementierungsleitfadens" schließt die Beobachtbarkeit der Software im Betrieb ein. `tracing` ermöglicht die detaillierte Erfassung des Kontrollflusses und wichtiger Zustandsänderungen. Durch die Verwendung von Spans (z.B. `span!(Level::INFO, "operation_xyz");`) können Operationen, die mehrere Schritte umfassen, logisch gruppiert werden, was die Analyse von Log-Daten erheblich vereinfacht.
- **1.2.4. Konfigurationsmanagement**
    
    - Konfigurationsdaten für Domänendienste (z.B. Standard-Theme, Standard-Workspace-Layout) werden der Domänenschicht von außen (typischerweise von der Anwendungsschicht beim Start, geladen durch die Kernschicht) übergeben.
    - Die Domänenschicht definiert Strukturen für ihre Konfigurationsparameter. Diese Strukturen sollen `serde::Deserialize` implementieren.
    - Das bevorzugte Format für Konfigurationsdateien ist TOML (Crate: `toml` Version 0.8.22+ 5).
    - Pfade zu Konfigurationsdateien werden gemäß XDG Base Directory Specification 19 von der Kernschicht aufgelöst (z.B. `$XDG_CONFIG_HOME/your_app_name/domain_settings.toml`). Die `directories` Crate (Version 6.0.0 19 oder `directories-next` 2.0.0) kann hierfür von der Kernschicht genutzt werden.
    - Die Domänenschicht selbst führt keine Dateisystemoperationen durch, um ihre Unabhängigkeit und Testbarkeit zu wahren. Sie erhält Konfigurationen als bereits geparste Datenstrukturen. Diese Trennung der Verantwortlichkeiten ist entscheidend, da Dateisystemzugriffe als Systemdetails gelten, von denen die Domänenschicht abstrahiert sein soll. `serde` und `toml` sind etablierte Standards im Rust-Ökosystem für diese Aufgabe.
- **1.2.5. Serialisierung/Deserialisierung**
    
    - Verwendung des `serde` Crates (Version 1.0.219+ 12) für die Serialisierung und Deserialisierung von Datenstrukturen, die persistiert oder über Schichtgrenzen hinweg ausgetauscht werden müssen.
    - Alle relevanten Entitäten und Wertobjekte, die persistiert oder als Teil von Events/Signalen übertragen werden, müssen `serde::Serialize` und `serde::Deserialize` implementieren.
- **1.2.6. Asynchrone Operationen**
    
    - Die Domänenschicht wird primär synchron entworfen, um die Komplexität niedrig zu halten. Langlaufende Operationen oder Interaktionen mit I/O-bound Systemen (z.B. komplexe KI-Anfragen, die über die Kernschicht laufen) können jedoch asynchrone Schnittstellen erfordern.
    - Wo Asynchronität notwendig ist, wird `async/await` mit einer durch die Kerninfrastruktur vorgegebenen Runtime (z.B. Tokio) verwendet. Die Domänenschicht selbst startet keine eigenen Runtimes.
    - Kommunikation zwischen synchronen und asynchronen Teilen erfolgt über klar definierte Kanäle (z.B. `tokio::sync::mpsc` oder `async_channel` 25), die von der Kernschicht oder der Anwendungsschicht bereitgestellt werden.
    - Die primär synchrone Natur der Domänenschicht vereinfacht das Design und die Testbarkeit erheblich. Asynchronität wird nur dort eingeführt, wo sie unumgänglich ist, und die Verwaltung des Runtimes wird an die Kernschicht delegiert. Komplexe Geschäftslogik ist oft einfacher synchron zu verstehen und zu implementieren. Würde die Domänenschicht selbst durchgängig asynchron sein, müsste sie sich um Executor, Task-Spawning etc. kümmern, was ihre Komplexität erhöht und sie stärker an eine spezifische async-Runtime bindet. Indem sie synchrone Schnittstellen anbietet und für langlaufende Operationen auf von der Kernschicht bereitgestellte `Future`s oder asynchrone Funktionen zurückgreift, bleibt sie fokussierter.

### 1.3. Interaktionsmuster mit der Kernschicht und anderen Schichten

- **Mit der Kernschicht:**
    - Die Domänenschicht definiert Traits (abstrakte Schnittstellen, sogenannte Ports), die von der Kernschicht implementiert werden müssen (Adapter), um Zugriff auf systemnahe Funktionen zu erhalten. Beispiele hierfür sind `PersistencePort`, `SystemClockPort`, `SecureStoragePort`.
    - Beispiel: Das `ThemingService` (siehe Abschnitt 3) könnte ein `ThemePersistencePort` Trait definieren, um Themes zu laden und zu speichern. Die Kernschicht würde dieses Trait implementieren und dabei z.B. auf das Dateisystem zugreifen.
- **Mit der System- und UI-Schicht:**
    - Die Domänenschicht stellt konkrete Services mit wohldefinierten Methoden bereit.
    - Die Kommunikation von Zustandsänderungen und Ereignissen aus der Domänenschicht an höhere Schichten erfolgt über ein Event/Signal-System (siehe Abschnitt 1.4.2).
    - Die Verwendung von Ports und Adapters (ein Muster der Hexagonalen Architektur) für die Interaktion mit der Kernschicht stellt sicher, dass die Domänenschicht vollständig von den Implementierungsdetails der Kernschicht entkoppelt ist. Die Domänenschicht "nutzt Funktionalität der Kernschicht". Um die Unabhängigkeit zu wahren, darf die Domänenschicht die Kernschicht nicht direkt aufrufen oder deren konkrete Typen kennen. Stattdessen definiert die Domänenschicht, _was_ sie benötigt (z.B. "speichere dieses Objekt"), und die Kernschicht liefert die Implementierung dafür. Dies ist ein Kernprinzip der Inversion of Control.

### 1.4. Allgemeine Datentypen, Traits und Hilfsfunktionen der Domänenschicht

- **1.4.1. Basis-Identifikatoren**
    
    - Typalias `DomainId`: Vorerst wird `String` für Flexibilität und einfache Serialisierung gewählt.
        
        Rust
        
        ```
        pub type DomainId = String;
        ```
        
        Alternativ könnte `uuid::Uuid` verwendet werden, falls global eindeutige IDs über Systemgrenzen hinweg erforderlich sind und dies in der Kerninfrastruktur-Spezifikation festgelegt wurde. Für rein interne Zwecke könnten auch Newtype-Strukturen um `usize` in Betracht gezogen werden. Da die Domänenschicht UI- und systemunabhängig ist, sind einfache, serialisierbare IDs oft ausreichend.
- **1.4.2. Event-System Abstraktion**
    
    - Ein generischer `DomainEvent` Enum kapselt alle Domänenereignisse. Jedes Modul definiert seine eigenen spezifischen Event-Typen, die als Varianten in `DomainEvent` aufgenommen werden.
        
        Rust
        
        ```
        #
        pub enum DomainEvent {
            Workspace(crate::workspace_manager::WorkspaceEvent),
            Theming(crate::theming_manager::ThemingEvent),
            AIConsent(crate::ai_manager::consent_manager::AIConsentEvent),
            AIFeature(crate::ai_manager::feature_service::AIFeatureEvent),
            Notification(crate::notification_manager::NotificationEvent),
            WindowPolicy(crate::window_policy_engine::WindowPolicyEvent),
            // Weitere Event-Kategorien können hier hinzugefügt werden.
        }
        ```
        
    - **Beispiel für eine spezifische Event-Kategorie (WorkspaceEvent):**
        
        Rust
        
        ```
        // Definiert in workspace_manager/events.rs oder workspace_manager/mod.rs
        #
        pub enum WorkspaceEvent {
            SpaceCreated {
                space_id: DomainId,
                name: String,
                layout_type: crate::workspace_manager::LayoutType, // Vollständiger Pfad zum Typ
                // Weitere relevante Felder
            },
            SpaceDeleted { space_id: DomainId },
            SpaceRenamed { space_id: DomainId, new_name: String },
            // Weitere Workspace-spezifische Events
        }
        ```
        
        Ähnliche Enums (`ThemingEvent`, `AIConsentEvent`, etc.) werden in den jeweiligen Modulen definiert.
    - **Publisher:** Typischerweise die Services innerhalb der Domänenschicht (z.B. `SpaceService`).
    - **Subscriber:** Andere Services innerhalb der Domänenschicht, die auf bestimmte Ereignisse reagieren müssen, oder die System-/UI-Schicht, die über Änderungen informiert werden wollen. Die konkrete Event-Bus-Implementierung wird von der Kerninfrastruktur bereitgestellt; die Domänenschicht definiert nur die Events und identifiziert typische Sender und Empfänger.
    - Ein klar definiertes, typisiertes Event-System ist fundamental für eine entkoppelte Architektur. Die Verwendung von `serde` für Events ermöglicht deren einfache Serialisierung, falls sie z.B. über Prozessgrenzen hinweg gesendet oder persistiert werden müssten. Die Anforderung, "Identifikation der typischen Publisher und Subscriber für jedes Event" und "Eindeutiger Event-Name/Typ", wird durch dieses strukturierte Event-System erfüllt. Ein übergreifender `DomainEvent` Enum mit untergeordneten Enums pro Modul schafft eine klare Hierarchie und ermöglicht es Subscribern, sich gezielt für Event-Kategorien oder spezifische Events zu registrieren.
- **1.4.3. Standardisierte Rückgabetypen**
    
    - Verwendung von `Result<T, DomainError>` für alle öffentlichen Operationen der Domänenschicht, die fehlschlagen können.
        
        Rust
        
        ```
        #
        pub enum DomainError {
            #[error("Workspace error: {0}")]
            Workspace(#[from] crate::workspace_manager::WorkspaceError),
            #
            Theming(#[from] crate::theming_manager::ThemingError),
            #[error("AI consent error: {0}")]
            AIConsent(#[from] crate::ai_manager::consent_manager::AIConsentError),
            #[error("AI feature error: {0}")]
            AIFeature(#[from] crate::ai_manager::feature_service::AIFeatureError),
            #[error("Notification error: {0}")]
            Notification(#[from] crate::notification_manager::NotificationError),
            #[error("Window policy error: {0}")]
            WindowPolicy(#[from] crate::window_policy_engine::WindowPolicyError),
        
            #[error("Persistence error: {0}")]
            Persistence(String), // Fehler von der Persistenzschicht (Kernschicht)
            #[error("Configuration error: {0}")]
            Configuration(String), // Fehler beim Verarbeiten von Konfigurationen
            #[error("Invariant violation: {0}")]
            InvariantViolation(String), // Wenn eine Geschäftsregel verletzt wurde
            #[error("Unauthorized operation: {0}")]
            Unauthorized(String),
            #
            NotFound { resource_type: String, resource_id: String },
            #[error("Invalid input: {message}")]
            InvalidInput { message: String },
            #[error("An unexpected internal error occurred: {0}")]
            Internal(String), // Für nicht spezifisch behandelte Fehler
        }
        ```
        
    - Die Definition eines übergreifenden `DomainError` Enums, der spezifischere Fehler aus den einzelnen Modulen aggregiert (mittels `#[from]`), bietet eine konsistente Fehlerbehandlungsschnittstelle für die aufrufenden Schichten. Wenn jeder Service seinen eigenen, nicht verwandten Fehlertyp zurückgibt, wird die Fehlerbehandlung in der aufrufenden Schicht komplex. Ein gemeinsamer `DomainError` mit Varianten für jeden Modulfehler (z.B. `DomainError::Workspace(WorkspaceError::SpaceNotFound)`) ermöglicht es dem Aufrufer, entweder generisch auf `DomainError` zu reagieren oder spezifisch auf `WorkspaceError` oder sogar `SpaceNotFound` zu matchen. `thiserror` erleichtert diese Struktur erheblich.

---

## 2. Modul: Workspace-Management (`workspace_manager`)

### 2.1. Übersicht und Verantwortlichkeiten

Das Modul `workspace_manager` ist für die Verwaltung von "Spaces" – virtuellen Desktops oder Arbeitsbereichen – zuständig. Es kümmert sich um die Zuordnung von Fenstern, die durch abstrakte `WindowHandle`-Identifikatoren repräsentiert werden, zu diesen Spaces. Des Weiteren verwaltet es die Layout-Konfiguration für jeden Space, beispielsweise ob Fenster gekachelt (Tiling) oder frei beweglich (Floating) angeordnet werden. Das Modul stellt Informationen über den aktuellen Zustand der Spaces und der darin enthaltenen Fenster bereit. Eine wichtige Interaktion besteht mit dem `WindowPolicyEngine` (siehe Abschnitt 6), um Standardverhalten oder spezifische Regeln bei Fensteroperationen oder Wechseln zwischen Spaces anzuwenden.

Die zentrale Rolle dieses Moduls für die Organisation der Arbeitsumgebung des Benutzers erfordert eine effiziente und klare Zustandsverwaltung, die maßgeblich zur User Experience beiträgt. Die Abstraktion von "Fenstern" als `WindowHandle` ist hierbei kritisch, um die Unabhängigkeit von spezifischen Fenstersystemen wie Wayland oder X11 zu gewährleisten. Die Domänenschicht darf keine Wayland- oder X11-spezifischen Fenster-IDs direkt kennen. Ein `WindowHandle` (z.B. eine `String` oder `uuid::Uuid`) dient als stabiler, systemunabhängiger Identifikator. Die Systemschicht ist dafür verantwortlich, die Übersetzung zwischen diesem `WindowHandle` und den tatsächlichen Fenster-IDs des jeweiligen Windowing-Systems vorzunehmen.

### 2.2. Entitäten, Wertobjekte und Enums

- **2.2.1. Entität: `Space`**
    
    - Repräsentiert einen einzelnen Workspace.
    - **Attribute:**
        - `id: DomainId` (Eindeutige ID des Space, z.B. generiert via `uuid::Uuid::new_v4().to_string()`). Sichtbarkeit: `pub(crate)`. Initialwert: Generiert bei Erstellung. Invarianten: Eindeutig, unveränderlich nach Erstellung.
        - `name: String` (Benutzerdefinierter Name, z.B. "Arbeit", "Freizeit"). Sichtbarkeit: `pub`. Initialwert: Bei Erstellung übergeben. Invarianten: Nicht leer.
        - `layout_type: LayoutType` (Aktueller Layout-Modus des Space). Sichtbarkeit: `pub`. Initialwert: Bei Erstellung übergeben, Default `LayoutType::Floating`.
        - `windows: std::collections::VecDeque<WindowHandle>` (Geordnete Liste der Fenster-Handles in diesem Space; `VecDeque` für effizientes Hinzufügen/Entfernen an beiden Enden und Beibehaltung der Reihenfolge, was für Stacking-Order oder Tiling-Reihenfolge relevant sein kann). Sichtbarkeit: `pub(crate)`. Initialwert: Leer.
        - `tiling_config: Option<TilingConfiguration>` (Spezifische Konfiguration, wenn `layout_type == LayoutType::Tiling`). Sichtbarkeit: `pub`. Initialwert: `None`. Invarianten: `Some` gdw. `layout_type` eine Tiling-Variante ist.
        - `creation_timestamp: u64` (Unix-Timestamp in Millisekunden der Erstellung). Sichtbarkeit: `pub(crate)`. Initialwert: Zeitstempel bei Erstellung.
        - `last_accessed_timestamp: u64` (Unix-Timestamp in Millisekunden des letzten Zugriffs/Aktivierung). Sichtbarkeit: `pub(crate)`. Initialwert: Zeitstempel bei Erstellung.
    - **Methoden (interne Logik der `Space`-Entität):**
        - `pub(crate) fn new(id: DomainId, name: String, layout_type: LayoutType, tiling_config: Option<TilingConfiguration>, current_timestamp: u64) -> Self`
            - Vorbedingungen: `id` und `name` nicht leer. Wenn `layout_type` eine Tiling-Variante ist, muss `tiling_config` `Some` und valide sein.
            - Nachbedingungen: Ein neues `Space`-Objekt wird mit den initialen Werten erstellt.
        - `pub(crate) fn add_window(&mut self, window_handle: WindowHandle) -> Result<(), WorkspaceError>`
            - Fügt ein Fenster am Ende der `windows`-Liste hinzu.
            - Vorbedingungen: Fenster ist nicht bereits im Space.
            - Nachbedingungen: Fenster ist im Space enthalten.
            - Geschäftsregel: Verhindert Duplikate.
        - `pub(crate) fn remove_window(&mut self, window_handle: &WindowHandle) -> Result<(), WorkspaceError>`
            - Entfernt ein Fenster aus dem Space.
            - Vorbedingungen: Fenster ist im Space enthalten.
            - Nachbedingungen: Fenster ist nicht mehr im Space.
            - Geschäftsregel: Gibt Fehler zurück, wenn Fenster nicht gefunden wird.
        - `pub(crate) fn set_layout(&mut self, layout_type: LayoutType, config: Option<TilingConfiguration>) -> Result<(), WorkspaceError>`
            - Aktualisiert `layout_type` und `tiling_config`.
            - Vorbedingungen: Wenn `layout_type` eine Tiling-Variante ist, muss `config` `Some` und valide sein.
            - Nachbedingungen: Layout-Informationen sind aktualisiert.
            - Geschäftsregel: Validiert die `config` für Tiling-Layouts.
        - `pub(crate) fn is_empty(&self) -> bool`
            - Gibt `true` zurück, wenn keine Fenster im Space sind.
        - `pub(crate) fn contains_window(&self, window_handle: &WindowHandle) -> bool`
            - Prüft, ob das Fenster im Space enthalten ist.
        - `pub(crate) fn update_last_accessed(&mut self, current_timestamp: u64)`
            - Aktualisiert `last_accessed_timestamp`.
    - **Beziehungen:** Enthält eine Sammlung von `WindowHandle`s.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct Space {
            pub(crate) id: DomainId,
            pub name: String,
            pub layout_type: LayoutType,
            pub(crate) windows: std::collections::VecDeque<WindowHandle>,
            pub tiling_config: Option<TilingConfiguration>,
            pub(crate) creation_timestamp: u64,
            pub(crate) last_accessed_timestamp: u64,
        }
        ```
        
- **2.2.2. Wertobjekt: `WindowHandle`**
    
    - Eine reine ID-Abstraktion für ein Anwendungsfenster. Die Domänenschicht besitzt keine Kenntnisse über Größe, Position oder Inhalt des Fensters; diese Details werden von der UI- oder Systemschicht verwaltet.
    - **Attribute:**
        - `id: DomainId` (Eindeutiger, persistenter Identifikator). Sichtbarkeit: `pub`.
    - **Invarianten:** `id` ist nicht leer und eindeutig im Kontext aller verwalteten Fenster.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct WindowHandle {
            pub id: DomainId,
        }
        ```
        
- **2.2.3. Enum: `LayoutType`**
    
    - Definiert die möglichen Layout-Modi für einen Space.
    - **Varianten:** `Tiling`, `Floating`, `Monocle` (Vollbild für ein einzelnes Fenster im Space), `Custom(String)` (für erweiterbare, benutzerdefinierte Layouts).
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`, `Default` (z.B. `Floating`).
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum LayoutType {
            Tiling,
            #[default]
            Floating,
            Monocle,
            Custom(String),
        }
        ```
        
- **2.2.4. Wertobjekt: `TilingConfiguration`**
    
    - Spezifische Konfigurationseinstellungen für Tiling-Layouts.
    - **Attribute:**
        - `master_slave_ratio: f32` (Verhältnis der Master- zur Slave-Fläche, z.B. 0.5 für 50/50). Sichtbarkeit: `pub`. Initialwert: z.B. `0.5`. Invarianten: 0.1≤ratio≤0.9.
        - `num_master_windows: u32` (Anzahl der Fenster im Master-Bereich). Sichtbarkeit: `pub`. Initialwert: z.B. `1`. Invarianten: ≥1.
        - `orientation: TilingOrientation` (Ausrichtung der Tiling-Anordnung). Sichtbarkeit: `pub`. Initialwert: `TilingOrientation::Vertical`.
        - `gap_size: u32` (Abstand zwischen Fenstern in logischen Einheiten). Sichtbarkeit: `pub`. Initialwert: z.B. `5`. Invarianten: ≥0.
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct TilingConfiguration {
            pub master_slave_ratio: f32,
            pub num_master_windows: u32,
            pub orientation: TilingOrientation,
            pub gap_size: u32,
        }
        
        impl Default for TilingConfiguration {
            fn default() -> Self {
                Self {
                    master_slave_ratio: 0.5,
                    num_master_windows: 1,
                    orientation: TilingOrientation::Vertical,
                    gap_size: 5,
                }
            }
        }
        ```
        
- **2.2.5. Enum: `TilingOrientation`**
    
    - Definiert die Hauptausrichtung für Tiling-Layouts.
    - **Varianten:** `Horizontal` (Master-Bereich links/rechts), `Vertical` (Master-Bereich oben/unten).
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`, `Default` (z.B. `Vertical`).
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum TilingOrientation {
            Horizontal,
            #[default]
            Vertical,
        }
        ```
        
- **2.2.6. Fehler-Enum: `WorkspaceError`**
    
    - Spezifische Fehler, die im `workspace_manager`-Modul auftreten können. Definiert mit `thiserror`.
    - **Varianten:**
        - `#` `SpaceNotFound { space_id: DomainId }`
        - `#` `WindowAlreadyInSpace { window_id: DomainId, space_id: DomainId }`
        - `#` `WindowNotInSpace { window_id: DomainId, space_id: DomainId }`
        - `#[error("Invalid layout configuration: {reason}")]` `InvalidLayoutConfiguration { reason: String }`
        - `#[error("A space with the name '{name}' already exists.")]` `DuplicateSpaceName { name: String }`
        - `#[error("Maximum number of spaces reached.")]` `MaxSpacesReached`
        - `#` `CannotDeleteLastSpace`
        - `#` `CannotDeleteNonEmptySpace { space_id: DomainId, window_count: usize }`
        - `#` `EmptySpaceName`
        - `#` `MissingTilingConfiguration`
        - `#` `UnexpectedTilingConfiguration`
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum WorkspaceError {
            #
            SpaceNotFound { space_id: DomainId },
            #
            WindowAlreadyInSpace { window_id: DomainId, space_id: DomainId },
            #
            WindowNotInSpace { window_id: DomainId, space_id: DomainId },
            #[error("Invalid layout configuration: {reason}")]
            InvalidLayoutConfiguration { reason: String },
            #[error("A space with the name '{name}' already exists.")]
            DuplicateSpaceName { name: String },
            #[error("Maximum number of spaces reached.")]
            MaxSpacesReached,
            #
            CannotDeleteLastSpace,
            #
            CannotDeleteNonEmptySpace { space_id: DomainId, window_count: usize },
            #
            EmptySpaceName,
            #
            MissingTilingConfiguration,
            #
            UnexpectedTilingConfiguration,
        }
        ```
        

### 2.3. Service: `SpaceService`

Der `SpaceService` ist die zentrale Komponente für die Orchestrierung aller Operationen im Zusammenhang mit Spaces. Er hält den Zustand aller bekannten Spaces (typischerweise in einer `HashMap<DomainId, Space>`) und nutzt intern die Methoden der `Space`-Entität zur Manipulation einzelner Spaces.

- **2.3.1. Eigenschaften (Interner Zustand des `SpaceService`)**
    
    - `active_space_id: Option<DomainId>`: Die ID des aktuell aktiven/fokussierten Space. Zugriff über Methoden.
    - `spaces: std::collections::HashMap<DomainId, Space>`: Eine Map, die alle bekannten Spaces anhand ihrer ID speichert.
    - `space_order: Vec<DomainId>`: Eine geordnete Liste der Space-IDs, um eine konsistente Reihenfolge (z.B. für UI-Anzeige oder Space-Navigation) beizubehalten.
    - `max_spaces: usize`: Maximale Anzahl erlaubter Spaces (konfigurierbar, z.B. Default 10).
    - `event_publisher: Box<dyn DomainEventPublisher>`: Eine Abstraktion zum Senden von Domänenereignissen (wird bei Initialisierung injiziert).
- **2.3.2. Methoden (Öffentliche API des `SpaceService`)**
    
    Alle Methoden, die den Zustand verändern (`&mut self`), sollten bei Erfolg relevante `WorkspaceEvent`s über den `event_publisher` emittieren. Die Zeitstempel werden typischerweise von einer `Clock`-Abstraktion bezogen, die von der Kernschicht bereitgestellt und dem Service injiziert wird.
    
    - `pub fn new(max_spaces: usize, event_publisher: Box<dyn DomainEventPublisher>, initial_spaces_config: Option<Vec<InitialSpaceConfig>>) -> Self`
        - Konstruktor. Initialisiert den Service. Erstellt einen Default-Space, falls `initial_spaces_config` `None` oder leer ist.
        - `event_publisher` ist eine Trait-Implementierung, die von der Anwendungsschicht bereitgestellt wird, um Events zu publizieren.
        - `InitialSpaceConfig { name: String, layout_type: LayoutType, tiling_config: Option<TilingConfiguration> }`
    - `pub fn create_space(&mut self, name: String, layout_type: LayoutType, tiling_config: Option<TilingConfiguration>) -> Result<DomainId, WorkspaceError>`
        - **Beschreibung:** Erstellt einen neuen Space.
        - **Parameter:**
            - `name: String`: Der gewünschte Name für den neuen Space.
            - `layout_type: LayoutType`: Der initiale Layout-Typ des Space.
            - `tiling_config: Option<TilingConfiguration>`: Konfiguration für Tiling, falls `layout_type` dies erfordert.
        - **Rückgabe:** `Result<DomainId, WorkspaceError>` - Die ID des neu erstellten Space oder ein Fehler.
        - **Vorbedingungen:**
            - `name` darf nicht leer sein (`WorkspaceError::EmptySpaceName`).
            - Anzahl der Spaces < `max_spaces` (`WorkspaceError::MaxSpacesReached`).
            - `name` sollte eindeutig sein (optional, sonst `WorkspaceError::DuplicateSpaceName` oder automatische Umbenennung).
            - Wenn `layout_type == LayoutType::Tiling`, muss `tiling_config` `Some` sein (`WorkspaceError::MissingTilingConfiguration`).
            - Wenn `layout_type!= LayoutType::Tiling`, sollte `tiling_config` `None` sein (optional, sonst `WorkspaceError::UnexpectedTilingConfiguration` oder Ignorieren).
        - **Nachbedingungen:** Ein neuer `Space` existiert im Service. `space_order` wird aktualisiert.
        - **Emittiert:** `WorkspaceEvent::SpaceCreated { space_id, name, layout_type, tiling_config }`.
    - `pub fn get_space(&self, space_id: &DomainId) -> Result<&Space, WorkspaceError>`
        - **Beschreibung:** Gibt eine unveränderliche Referenz auf einen Space anhand seiner ID zurück.
        - **Parameter:** `space_id: &DomainId`.
        - **Rückgabe:** `Result<&Space, WorkspaceError>` (`WorkspaceError::SpaceNotFound`).
    - `pub fn list_spaces(&self) -> Vec<&Space>`
        - **Beschreibung:** Gibt eine Liste von Referenzen auf alle Spaces in der durch `space_order` definierten Reihenfolge zurück.
        - **Rückgabe:** `Vec<&Space>`.
    - `pub fn update_space_name(&mut self, space_id: &DomainId, new_name: String) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Aktualisiert den Namen eines existierenden Space.
        - **Parameter:** `space_id: &DomainId`, `new_name: String`.
        - **Vorbedingungen:** `new_name` nicht leer und (optional) eindeutig.
        - **Emittiert:** `WorkspaceEvent::SpaceRenamed { space_id: space_id.clone(), new_name }`.
    - `pub fn delete_space(&mut self, space_id: &DomainId, force_delete_windows: bool) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Löscht einen Space.
        - **Parameter:** `space_id: &DomainId`, `force_delete_windows: bool`.
        - **Vorbedingungen:**
            - Es muss mehr als ein Space existieren (`WorkspaceError::CannotDeleteLastSpace`).
            - Wenn `force_delete_windows == false` und der Space Fenster enthält, wird `WorkspaceError::CannotDeleteNonEmptySpace` zurückgegeben.
        - **Logik:** Wenn `force_delete_windows == true` und Fenster im Space sind, werden diese Fenster in einen Default-Space (z.B. den ersten in `space_order` oder den aktiven, falls nicht der zu löschende) verschoben. Für jedes verschobene Fenster wird `WorkspaceEvent::WindowMovedBetweenSpaces` emittiert.
        - **Nachbedingungen:** Der Space ist entfernt. `space_order` ist aktualisiert. Wenn der gelöschte Space aktiv war, wird ein anderer Space (z.B. der nächste in der Liste) aktiv gesetzt (`ActiveSpaceChanged` Event).
        - **Emittiert:** `WorkspaceEvent::SpaceDeleted { space_id: space_id.clone() }`.
    - `pub fn add_window_to_space(&mut self, space_id: &DomainId, window_handle: WindowHandle) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Fügt ein Fenster zu einem spezifischen Space hinzu. Wenn das Fenster bereits in einem anderen Space ist, wird es implizit daraus entfernt (oder es wird ein Fehler zurückgegeben, je nach Designentscheidung – hier wird angenommen, es wird verschoben).
        - **Vorbedingungen:** Der Ziel-Space existiert. Das Fenster ist nicht bereits im Ziel-Space.
        - **Logik:** Prüft, ob das Fenster in einem anderen Space ist. Falls ja, `remove_window_from_space` für den Quell-Space aufrufen. Dann zum Ziel-Space hinzufügen.
        - **Emittiert:** `WorkspaceEvent::WindowAddedToSpace { space_id: space_id.clone(), window_id: window_handle.id.clone() }`. Wenn es von einem anderen Space verschoben wurde, wird stattdessen `WindowMovedBetweenSpaces` emittiert.
    - `pub fn remove_window_from_space(&mut self, space_id: &DomainId, window_handle: &WindowHandle) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Entfernt ein Fenster aus einem spezifischen Space.
        - **Vorbedingungen:** Der Space existiert und enthält das Fenster.
        - **Emittiert:** `WorkspaceEvent::WindowRemovedFromSpace { space_id: space_id.clone(), window_id: window_handle.id.clone() }`.
    - `pub fn move_window_to_space(&mut self, window_handle: &WindowHandle, target_space_id: &DomainId) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Verschiebt ein Fenster von seinem aktuellen Space (falls vorhanden) in den `target_space_id`.
        - **Logik:** Findet den Quell-Space des Fensters. Ruft `remove_window_from_space` für den Quell-Space und `add_window_to_space` für den Ziel-Space auf.
        - **Emittiert:** `WorkspaceEvent::WindowMovedBetweenSpaces { window_id: window_handle.id.clone(), source_space_id, target_space_id: target_space_id.clone() }`.
    - `pub fn set_space_layout(&mut self, space_id: &DomainId, layout_type: LayoutType, tiling_config: Option<TilingConfiguration>) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Ändert das Layout eines Space.
        - **Vorbedingungen:** Validierung der `tiling_config` analog zu `create_space`.
        - **Emittiert:** `WorkspaceEvent::SpaceLayoutChanged { space_id: space_id.clone(), new_layout: layout_type, new_config: tiling_config }`.
    - `pub fn get_active_space_id(&self) -> Option<DomainId>`
        - **Beschreibung:** Gibt die ID des aktuell aktiven Space zurück.
    - `pub fn set_active_space(&mut self, space_id: &DomainId) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Setzt den aktiven Space.
        - **Vorbedingungen:** Der Space mit `space_id` existiert.
        - **Nachbedingungen:** `active_space_id` ist aktualisiert. `last_accessed_timestamp` des neuen aktiven Space wird aktualisiert.
        - **Emittiert:** `WorkspaceEvent::ActiveSpaceChanged { old_space_id: self.active_space_id.clone(), new_space_id: space_id.clone() }`.
    - `pub fn get_windows_in_space(&self, space_id: &DomainId) -> Result<Vec<WindowHandle>, WorkspaceError>`
        - **Beschreibung:** Gibt eine Kopie der Liste der Fenster-Handles für einen gegebenen Space zurück.
    - `pub fn find_space_for_window(&self, window_handle: &WindowHandle) -> Option<DomainId>`
        - **Beschreibung:** Gibt die ID des Space zurück, der das angegebene Fenster enthält, falls vorhanden.
    - `pub fn reorder_window_in_space(&mut self, space_id: &DomainId, window_handle: &WindowHandle, new_index: usize) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Ändert die Position eines Fensters innerhalb der `windows`-Liste eines Space (relevant für Stacking-Order oder Tiling).
        - **Emittiert:** `WorkspaceEvent::WindowReorderedInSpace { space_id: space_id.clone(), window_id: window_handle.id.clone(), new_index }`.
    - `pub fn reorder_space(&mut self, space_id: &DomainId, new_index: usize) -> Result<(), WorkspaceError>`
        - **Beschreibung:** Ändert die Position eines Space in der globalen `space_order`-Liste.
        - **Emittiert:** `WorkspaceEvent::SpaceReordered { space_id: space_id.clone(), new_index }`.
- **2.3.3. Signale/Events (als Varianten von `WorkspaceEvent` im `DomainEvent` Enum)**
    
    Diese Events werden vom `SpaceService` emittiert und über den injizierten `DomainEventPublisher` verteilt.
    
    - `SpaceCreated { space_id: DomainId, name: String, layout_type: LayoutType, tiling_config: Option<TilingConfiguration> }`
    - `SpaceDeleted { space_id: DomainId }`
    - `SpaceRenamed { space_id: DomainId, new_name: String }`
    - `SpaceLayoutChanged { space_id: DomainId, new_layout: LayoutType, new_config: Option<TilingConfiguration> }`
    - `WindowAddedToSpace { space_id: DomainId, window_id: DomainId }`
    - `WindowRemovedFromSpace { space_id: DomainId, window_id: DomainId }`
    - `WindowMovedBetweenSpaces { window_id: DomainId, source_space_id: DomainId, target_space_id: DomainId }`
    - `ActiveSpaceChanged { old_space_id: Option<DomainId>, new_space_id: DomainId }`
    - `WindowReorderedInSpace { space_id: DomainId, window_id: DomainId, new_index: usize }`
    - `SpaceReordered { space_id: DomainId, new_index: usize }`
    - **Typische Publisher:** `SpaceService`.
    - **Typische Subscriber:** UI-Schicht (zur Aktualisierung der Darstellung), `WindowPolicyEngine` (um ggf. auf Änderungen zu reagieren, z.B. um Fenster neu anzuordnen, wenn sich der aktive Space ändert), Persistenzmechanismus in der Kernschicht (um Änderungen zu speichern).
- **2.3.4. Trait: `DomainEventPublisher` (von Anwendungsschicht zu implementieren)**
    
    Rust
    
    ```
    pub trait DomainEventPublisher: Send + Sync {
        fn publish(&self, event: DomainEvent);
    }
    ```
    
    Dieser Trait wird benötigt, damit der `SpaceService` (und andere Services) Ereignisse publizieren können, ohne eine konkrete Implementierung eines Event-Busses zu kennen.
    

### 2.4. Geschäftsregeln und Invarianten (Beispiele)

- Ein Fenster kann immer nur in genau einem Space sein. Dies wird durch die Logik in `add_window_to_space` und `move_window_to_space` sichergestellt, die ein Fenster implizit aus seinem vorherigen Space entfernt.
- Es muss immer mindestens ein Space geben. `delete_space` verhindert das Löschen des letzten Space.
- Der Name eines Space sollte eindeutig sein (optional, konfigurierbar, ob Duplikate mit Suffix versehen oder abgelehnt werden). Die Methode `create_space` prüft dies.
- Spezifische Regeln für Tiling-Layouts (z.B. Mindestgröße von Fenstern, Verhalten bei Hinzufügen/Entfernen) werden innerhalb der `TilingConfiguration` und der Logik, die dieses Layout anwendet (potenziell in der UI-Schicht oder einer spezialisierten Layout-Engine, die von der Domänenschicht gesteuert wird), durchgesetzt. Die Domänenschicht speichert nur die Konfiguration.
- Maximale Anzahl von Spaces (`max_spaces`): Wird in `create_space` geprüft.

Die folgende Tabelle fasst die Attribute der `Space`-Entität zusammen:

**Tabelle 2.2.1: Entität `Space` - Attribute**

|   |   |   |   |   |
|---|---|---|---|---|
|**Attribut**|**Typ**|**Sichtbarkeit**|**Initialwert (Beispiel)**|**Invarianten**|
|`id`|`DomainId`|`pub(crate)`|Generiert|Eindeutig, unveränderlich|
|`name`|`String`|`pub`|Bei Erstellung|Nicht leer|
|`layout_type`|`LayoutType`|`pub`|`LayoutType::Floating`|Gültiger `LayoutType`|
|`windows`|`std::collections::VecDeque<WindowHandle>`|`pub(crate)`|Leer|Enthält nur gültige `WindowHandle`s|
|`tiling_config`|`Option<TilingConfiguration>`|`pub`|`None`|`Some` gdw. `layout_type` ist Tiling-Variante|
|`creation_timestamp`|`u64`|`pub(crate)`|Zeitstempel bei Erstellung||
|`last_accessed_timestamp`|`u64`|`pub(crate)`|Zeitstempel bei Erstellung||

Die öffentliche API des `SpaceService` ist in der folgenden Tabelle dargestellt:

**Tabelle 2.3.2: `SpaceService` - Öffentliche API (Auswahl)**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp**|**Kurzbeschreibung**|
|`create_space`|`name: String`, `layout: LayoutType`, `config: Option<TilingConfiguration>`|`Result<DomainId, WorkspaceError>`|Erstellt einen neuen Space.|
|`get_space`|`space_id: &DomainId`|`Result<&Space, WorkspaceError>`|Ruft einen Space anhand seiner ID ab.|
|`list_spaces`||`Vec<&Space>`|Listet alle Spaces auf.|
|`delete_space`|`space_id: &DomainId`, `force: bool`|`Result<(), WorkspaceError>`|Löscht einen Space.|
|`add_window_to_space`|`space_id: &DomainId`, `window: WindowHandle`|`Result<(), WorkspaceError>`|Fügt ein Fenster zu einem Space hinzu.|
|`move_window_to_space`|`window: &WindowHandle`, `target_space_id: &DomainId`|`Result<(), WorkspaceError>`|Verschiebt ein Fenster in einen anderen Space.|
|`set_active_space`|`space_id: &DomainId`|`Result<(), WorkspaceError>`|Setzt den aktiven Space.|
|`get_active_space_id`||`Option<DomainId>`|Gibt die ID des aktiven Space zurück.|

---

## 3. Modul: Theming-System (`theming_manager`)

### 3.1. Übersicht und Verantwortlichkeiten

Das `theming_manager`-Modul ist für die Verwaltung von Themes und deren Design-Tokens zuständig. Es ermöglicht die Anwendung eines aktiven Themes und dessen Variante (z.B. Hell/Dunkel) und stellt Mechanismen bereit, über die die UI-Schicht Token-Werte abfragen kann. Das Laden und Speichern von Theme-Definitionen erfolgt über Abstraktionen (Ports), die von der Kernschicht implementiert werden.

Ein Token-basiertes Theming-System ist hierbei von zentraler Bedeutung.26 Die Domänenschicht verwaltet die _Definition_ und _Auswahl_ der Tokens. Die _Interpretation und Anwendung_ dieser Tokens (z.B. das Rendern von CSS für GTK-Anwendungen) ist Aufgabe der UI-Schicht. GTK4-CSS-Konzepte, wie sie in 28 beschrieben sind (z.B. Custom Properties wie `--prop: value; color: var(--prop);`), dienen als starkes konzeptionelles Vorbild für die Struktur der Tokens, auch wenn die Domänenschicht selbst kein CSS direkt verarbeitet oder generiert. Die UI-Unabhängigkeit der Domänenschicht bedingt, dass das Theming-System keine direkten Zeichenoperationen durchführt. Stattdessen liefert es die notwendigen Informationen, beispielsweise einen Token `primary_background_color: "#FFFFFF"`. Die UI-Schicht fragt diesen Token-Wert ab und verwendet ihn in ihrem spezifischen Rendering-System (z.B. GTK CSS, Qt Stylesheets oder Web CSS). Dieser Ansatz fördert Konsistenz, ermöglicht schnellere Updates und verbessert die Skalierbarkeit des Theming-Systems.26

### 3.2. Entitäten, Wertobjekte und Enums

- **3.2.1. Entität: `Theme`**
    
    - Repräsentiert eine vollständige Theme-Definition.
    - **Attribute:**
        - `id: DomainId` (Eindeutige ID des Themes, z.B. "arc-dark-custom"). Sichtbarkeit: `pub`. Invarianten: Eindeutig, nicht leer.
        - `name: String` (Anzeigename, z.B. "Arc Dark Custom"). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `description: Option<String>` (Optionale Beschreibung des Themes). Sichtbarkeit: `pub`.
        - `author: Option<String>` (Optionaler Autor des Themes). Sichtbarkeit: `pub`.
        - `version: String` (Version des Themes, z.B. "1.0.0"). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `supported_variants: Vec<ThemeVariantType>` (Liste der unterstützten Varianten, z.B. ``). Sichtbarkeit: `pub`. Invarianten: Muss mindestens eine Variante enthalten.
        - `tokens: std::collections::HashMap<String, ThemeToken>` (Schlüssel ist der hierarchische Token-Name, z.B. "color.background.primary"). Sichtbarkeit: `pub(crate)`.
        - `metadata: std::collections::HashMap<String, String>` (Zusätzliche Metadaten, z.B. Pfad zur Quelldatei, Lizenz). Sichtbarkeit: `pub`.
    - **Methoden (interne Logik der `Theme`-Entität):**
        - `pub(crate) fn get_token_value(&self, token_name: &str, variant: ThemeVariantType) -> Option<&ThemeTokenValue>`
            - Sucht den Token mit `token_name`.
            - Gibt `value_dark` zurück, wenn `variant == Dark` und `value_dark` `Some` ist.
            - Gibt ansonsten `value_light` zurück, wenn der Token die Variante unterstützt (implizit, da `value_light` obligatorisch ist).
            - Gibt `None` zurück, wenn der Token nicht existiert oder die spezifische Variante nicht explizit definiert ist und keine Ableitungsregel existiert (für diese Spezifikation wird keine komplexe Ableitung angenommen).
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct Theme {
            pub id: DomainId,
            pub name: String,
            pub description: Option<String>,
            pub author: Option<String>,
            pub version: String,
            pub supported_variants: Vec<ThemeVariantType>,
            pub(crate) tokens: std::collections::HashMap<String, ThemeToken>,
            pub metadata: std::collections::HashMap<String, String>,
        }
        ```
        
- **3.2.2. Wertobjekt: `ThemeToken`**
    
    - Definiert einen einzelnen Design-Token. Die Trennung von `value_light` und `value_dark` direkt im Token ermöglicht eine explizite Definition pro Variante und vereinfacht die Abfrage.
    - **Attribute:**
        - `name: String` (Eindeutiger, hierarchischer Name, z.B. "color.text.primary", "font.body.family", "spacing.medium"). Sichtbarkeit: `pub`. Invarianten: Nicht leer, folgt einer Namenskonvention (z.B. `kebab-case` oder `snake_case`).
        - `token_type: ThemeTokenType` (Typ des Tokens). Sichtbarkeit: `pub`.
        - `value_light: ThemeTokenValue` (Wert für die helle Variante). Sichtbarkeit: `pub`. Invarianten: Muss zum `token_type` passen.
        - `value_dark: Option<ThemeTokenValue>` (Optionaler spezifischer Wert für die dunkle Variante; falls `None`, wird `value_light` verwendet). Sichtbarkeit: `pub`. Invarianten: Falls `Some`, muss zum `token_type` passen.
        - `description: Option<String>` (Beschreibung des Tokens und seines Verwendungszwecks). Sichtbarkeit: `pub`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct ThemeToken {
            pub name: String,
            pub token_type: ThemeTokenType,
            pub value_light: ThemeTokenValue,
            pub value_dark: Option<ThemeTokenValue>,
            pub description: Option<String>,
        }
        ```
        
- **3.2.3. Enum: `ThemeTokenType`**
    
    - Klassifiziert die Art eines `ThemeToken`.
    - **Varianten:** `Color` (String, z.B. "#RRGGBBAA" oder "rgba(r,g,b,a)"), `FontSize` (String, z.B. "12pt", "1.2em"), `FontFamily` (String, z.B. "Noto Sans"), `Spacing` (String, z.B. "8px"), `BorderRadius` (String, z.B. "4px"), `Shadow` (String, CSS-ähnliche Definition, z.B. "2px 2px 5px rgba(0,0,0,0.3)"), `IconSet` (String, Name eines Icon-Sets), `Custom(String)` (für anwendungsspezifische Tokens, z.B. "animation.duration.fast").
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum ThemeTokenType {
            Color,
            FontSize,
            FontFamily,
            Spacing,
            BorderRadius,
            Shadow,
            IconSet,
            Custom(String),
        }
        ```
        
- **3.2.4. Wertobjekt: `ThemeTokenValue`**
    
    - Repräsentiert den konkreten Wert eines Tokens. Für Einfachheit wird hier `String` verwendet; die UI-Schicht interpretiert den String basierend auf `ThemeTokenType`.
    - **Attribute:**
        - `value: String`. Sichtbarkeit: `pub`.
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`, `Default`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct ThemeTokenValue {
            pub value: String,
        }
        ```
        
- **3.2.5. Wertobjekt: `ActiveThemeState`**
    
    - Speichert den aktuell aktiven Theme-Zustand.
    - **Attribute:**
        - `theme_id: DomainId` (ID des aktiven Themes). Sichtbarkeit: `pub`.
        - `variant: ThemeVariantType` (Aktive Variante). Sichtbarkeit: `pub`.
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`, `Default`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct ActiveThemeState {
            pub theme_id: DomainId,
            pub variant: ThemeVariantType,
        }
        ```
        
- **3.2.6. Enum: `ThemeVariantType`**
    
    - Definiert die möglichen Varianten eines Themes.
    - **Varianten:** `Light`, `Dark`.
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`, `Default` (z.B. `Light`).
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum ThemeVariantType {
            #[default]
            Light,
            Dark,
        }
        ```
        
- **3.2.7. Fehler-Enum: `ThemingError`**
    
    - Spezifische Fehler für das `theming_manager`-Modul. Definiert mit `thiserror`.
    - **Varianten:**
        - `#` `ThemeNotFound { theme_id: DomainId }`
        - `#` `TokenNotFound { theme_id: DomainId, token_name: String }`
        - `#[error("Variant '{variant:?}' not supported by theme '{theme_id}'.")]` `VariantNotSupported { theme_id: DomainId, variant: ThemeVariantType }`
        - `#[error("Invalid token value for '{token_name}': '{value}'. Expected type: {expected_type:?}.")]` `InvalidTokenValue { token_name: String, value: String, expected_type: ThemeTokenType }`
        - `#[error("Failed to load theme from '{path}': {reason}")]` `ThemeLoadError { path: String, reason: String }`
        - `#[error("Failed to save theme to '{path}': {reason}")]` `ThemeSaveError { path: String, reason: String }`
        - `#` `ThemeIdExists { theme_id: DomainId }`
        - `#` `DefaultThemeActivationFailed { theme_id: DomainId, reason: String }`
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum ThemingError {
            #
            ThemeNotFound { theme_id: DomainId },
            #
            TokenNotFound { theme_id: DomainId, token_name: String },
            #[error("Variant '{variant:?}' not supported by theme '{theme_id}'.")]
            VariantNotSupported { theme_id: DomainId, variant: ThemeVariantType },
            #[error("Invalid token value for '{token_name}': '{value}'. Expected type: {expected_type:?}.")]
            InvalidTokenValue { token_name: String, value: String, expected_type: ThemeTokenType },
            #[error("Failed to load theme from '{path}': {reason}")]
            ThemeLoadError { path: String, reason: String },
            #[error("Failed to save theme to '{path}': {reason}")]
            ThemeSaveError { path: String, reason: String },
            #
            ThemeIdExists { theme_id: DomainId },
            #
            DefaultThemeActivationFailed { theme_id: DomainId, reason: String },
        }
        ```
        

### 3.3. Service: `ThemingService`

Der `ThemingService` verwaltet die geladenen Themes und den aktiven Theme-Zustand. Er interagiert mit einem `ThemePersistencePort` (implementiert von der Kernschicht) zum Laden und Speichern von Theme-Daten und dem aktiven Zustand.

- **3.3.1. Eigenschaften (Interner Zustand des `ThemingService`)**
    
    - `loaded_themes: std::collections::HashMap<DomainId, Theme>`: Map aller geladenen Themes.
    - `active_theme_state: ActiveThemeState`: Der aktuell ausgewählte Theme und dessen Variante.
    - `theme_persistence_port: Box<dyn ThemePersistencePort>`: Injizierter Port für Persistenz.
    - `event_publisher: Box<dyn DomainEventPublisher>`: Injizierter Publisher für Domänenereignisse.
    - `default_theme_id: DomainId`: ID eines fest einkompilierten oder als sicher bekannten Fallback-Themes.
- **3.3.2. Methoden (Öffentliche API des `ThemingService`)**
    
    - `pub fn new(default_theme_id: DomainId, default_theme: Theme, theme_persistence_port: Box<dyn ThemePersistencePort>, event_publisher: Box<dyn DomainEventPublisher>) -> Self`
        - Konstruktor. Lädt initial verfügbare Themes über den `theme_persistence_port` und den gespeicherten `ActiveThemeState`.
        - Fügt das `default_theme` zu `loaded_themes` hinzu.
        - Versucht, den gespeicherten aktiven Theme-Zustand zu laden. Falls nicht erfolgreich oder inkonsistent, wird `default_theme_id` mit `ThemeVariantType::default()` aktiviert.
    - `pub fn list_available_themes(&self) -> Vec<(DomainId, String)>`
        - Gibt eine Liste von Tupeln `(id, name)` aller geladenen Themes zurück.
    - `pub fn get_theme_details(&self, theme_id: &DomainId) -> Result<&Theme, ThemingError>`
        - Gibt Details zu einem spezifischen Theme zurück.
    - `pub fn set_active_theme(&mut self, theme_id: &DomainId) -> Result<(), ThemingError>`
        - Setzt das aktive Theme. Die Variante bleibt unverändert, sofern vom neuen Theme unterstützt, sonst wird die Default-Variante des neuen Themes gewählt.
        - Vorbedingungen: Theme mit `theme_id` muss existieren und die aktuelle (oder eine Default-) Variante unterstützen.
        - Nachbedingungen: `active_theme_state.theme_id` ist aktualisiert.
        - Emittiert: `ThemingEvent::ActiveThemeChanged { new_theme_id: theme_id.clone(), new_variant: self.active_theme_state.variant }`.
        - Persistiert den neuen Zustand über `theme_persistence_port.save_active_theme_state()`.
    - `pub fn set_active_variant(&mut self, variant: ThemeVariantType) -> Result<(), ThemingError>`
        - Setzt die aktive Theme-Variante.
        - Vorbedingungen: Das aktuell aktive Theme muss die `variant` unterstützen.
        - Nachbedingungen: `active_theme_state.variant` ist aktualisiert.
        - Emittiert: `ThemingEvent::ThemeVariantChanged { new_variant: variant }`.
        - Persistiert den neuen Zustand.
    - `pub fn get_active_theme_id(&self) -> &DomainId`
    - `pub fn get_active_variant(&self) -> ThemeVariantType`
    - `pub fn get_token_value(&self, token_name: &str) -> Result<ThemeTokenValue, ThemingError>`
        - Gibt den Wert des angeforderten Tokens für das aktuell aktive Theme und die aktive Variante zurück.
        - Nutzt intern `Theme::get_token_value()`.
        - Fallback: Wenn Token im spezifischen Theme nicht gefunden, könnte ein Fallback auf das Default-Theme erfolgen (optional, muss klar definiert sein).
    - `pub fn get_specific_token_value(&self, theme_id: &DomainId, variant: ThemeVariantType, token_name: &str) -> Result<ThemeTokenValue, ThemingError>`
        - Gibt den Wert eines Tokens für ein spezifisches (nicht notwendigerweise aktives) Theme und eine spezifische Variante zurück.
    - `pub fn reload_themes(&mut self) -> Result<usize, ThemingError>`
        - Lädt alle Themes von den bekannten Pfaden (die der `ThemePersistencePort` kennt) neu.
        - Aktualisiert `loaded_themes`. Stellt sicher, dass das `active_theme_state` gültig bleibt (ggf. Fallback auf Default-Theme).
        - Gibt die Anzahl der erfolgreich geladenen Themes zurück.
        - Emittiert: `ThemingEvent::ThemesReloaded { num_loaded }`.
    - `pub fn add_theme(&mut self, theme: Theme) -> Result<(), ThemingError>`
        - Fügt ein neues Theme dynamisch hinzu (z.B. von Benutzer importiert).
        - Vorbedingungen: `theme.id` darf nicht bereits existieren.
        - Speichert das Theme über den `ThemePersistencePort`.
        - Emittiert: `ThemingEvent::ThemeAdded { theme_id: theme.id.clone() }`.
    - `pub fn remove_theme(&mut self, theme_id: &DomainId) -> Result<(), ThemingError>`
        - Entfernt ein Theme.
        - Vorbedingungen: Das Theme darf nicht das aktive Theme sein (oder es muss ein Fallback-Mechanismus greifen). Darf nicht das Default-Theme sein.
        - Löscht das Theme über den `ThemePersistencePort`.
        - Emittiert: `ThemingEvent::ThemeRemoved { theme_id: theme_id.clone() }`.
- **3.3.3. Signale/Events (als Varianten von `ThemingEvent` im `DomainEvent` Enum)**
    
    - `ActiveThemeChanged { new_theme_id: DomainId, new_variant: ThemeVariantType }`
    - `ThemeVariantChanged { new_variant: ThemeVariantType }`
    - `ThemesReloaded { num_loaded: usize }`
    - `ThemeAdded { theme_id: DomainId }`
    - `ThemeRemoved { theme_id: DomainId }`
    - `TokenValueChanged { theme_id: DomainId, variant: ThemeVariantType, token_name: String, new_value: ThemeTokenValue }` (Nur relevant, falls einzelne Tokens zur Laufzeit änderbar sein sollen, was typischerweise nicht der Fall ist für persistierte Themes, sondern eher für dynamische Anpassungen).
    - **Typische Publisher:** `ThemingService`.
    - **Typische Subscriber:** UI-Schicht (um auf Theme-Änderungen zu reagieren und die UI neu zu zeichnen/stilisieren), andere Domänendienste, die themenabhängige Logik haben könnten (selten).

### 3.4. Geschäftsregeln

- **Fallback-Mechanismen:** Wenn ein Token im aktiven Theme nicht definiert ist oder die spezifische Variante nicht abdeckt, wird der Wert des Tokens aus dem `default_theme_id` für die entsprechende Variante verwendet. Wenn auch dort nicht vorhanden, muss ein fest kodierter, anwendungsweiter Standardwert greifen (dieser ist außerhalb des ThemingService zu definieren, z.B. in der UI-Schicht als letzte Instanz).
- **Validierung von Theme-Dateien:** Beim Laden (durch den `ThemePersistencePort`) müssen Themes auf syntaktische Korrektheit und das Vorhandensein essentieller Tokens (z.B. Basisfarben, Standardschriftgrößen) geprüft werden. Fehlerhafte Themes werden nicht geladen.
- Das System muss immer ein gültiges aktives Theme haben. Das `default_theme` (mit `default_theme_id`) dient als garantierter Fallback.

### 3.5. Trait: `ThemePersistencePort` (von Kernschicht zu implementieren)

Dieser Port definiert die Schnittstelle, über die der `ThemingService` mit der Kernschicht für die Persistenz von Theme-Daten interagiert. Dies entkoppelt den Service von der konkreten Speicherimplementierung.

- `fn load_all_themes(&self) -> Result<Vec<Theme>, DomainError>;`
    - Lädt alle verfügbaren Theme-Definitionen von konfigurierten Speicherorten.
- `fn save_theme(&self, theme: &Theme) -> Result<(), DomainError>;`
    - Speichert eine einzelne Theme-Definition.
- `fn delete_theme(&self, theme_id: &DomainId) -> Result<(), DomainError>;`
    - Löscht eine Theme-Definition.
- `fn load_active_theme_state(&self) -> Result<Option<ActiveThemeState>, DomainError>;`
    - Lädt den zuletzt gespeicherten aktiven Theme-Zustand. Gibt `Ok(None)` zurück, wenn kein Zustand gespeichert ist.
- `fn save_active_theme_state(&self, state: &ActiveThemeState) -> Result<(), DomainError>;`
    - Speichert den aktuellen aktiven Theme-Zustand.

Die Implementierung dieses Traits in der Kernschicht würde typischerweise TOML-Dateien in XDG-Verzeichnissen (z.B. `$XDG_DATA_HOME/your_app_name/themes/` für Themes, `$XDG_CONFIG_HOME/your_app_name/theming_state.toml` für den aktiven Zustand) lesen und schreiben.

Die folgenden Tabellen fassen wichtige Aspekte des Theming-Systems zusammen:

**Tabelle 3.2.1: Entität `Theme` - Attribute**

|   |   |   |   |
|---|---|---|---|
|**Attribut**|**Typ**|**Sichtbarkeit**|**Invarianten**|
|`id`|`DomainId`|`pub`|Eindeutig, nicht leer|
|`name`|`String`|`pub`|Nicht leer|
|`description`|`Option<String>`|`pub`||
|`author`|`Option<String>`|`pub`||
|`version`|`String`|`pub`|Nicht leer|
|`supported_variants`|`Vec<ThemeVariantType>`|`pub`|Mindestens eine Variante|
|`tokens`|`std::collections::HashMap<String, ThemeToken>`|`pub(crate)`|Enthält gültige `ThemeToken`-Definitionen|
|`metadata`|`std::collections::HashMap<String, String>`|`pub`||

**Tabelle 3.2.2: Wertobjekt `ThemeToken` - Attribute**

|   |   |   |   |
|---|---|---|---|
|**Attribut**|**Typ**|**Sichtbarkeit**|**Invarianten**|
|`name`|`String`|`pub`|Nicht leer, hierarchisch (z.B. "color.text.primary")|
|`token_type`|`ThemeTokenType`|`pub`|Gültiger `ThemeTokenType`|
|`value_light`|`ThemeTokenValue`|`pub`|Passend zu `token_type`|
|`value_dark`|`Option<ThemeTokenValue>`|`pub`|Falls `Some`, passend zu `token_type`|
|`description`|`Option<String>`|`pub`||

**Tabelle 3.3.2: `ThemingService` - Öffentliche API (Auswahl)**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp**|**Kurzbeschreibung**|
|`list_available_themes`||`Vec<(DomainId, String)>`|Listet IDs und Namen aller geladenen Themes.|
|`set_active_theme`|`theme_id: &DomainId`|`Result<(), ThemingError>`|Setzt das aktive Theme.|
|`set_active_variant`|`variant: ThemeVariantType`|`Result<(), ThemingError>`|Setzt die aktive Theme-Variante.|
|`get_active_theme_id`||`&DomainId`|Gibt die ID des aktiven Themes zurück.|
|`get_active_variant`||`ThemeVariantType`|Gibt die aktive Variante zurück.|
|`get_token_value`|`token_name: &str`|`Result<ThemeTokenValue, ThemingError>`|Ruft Token-Wert für aktives Theme/Variante ab.|
|`reload_themes`||`Result<usize, ThemingError>`|Lädt alle Themes neu.|
|`add_theme`|`theme: Theme`|`Result<(), ThemingError>`|Fügt ein neues Theme hinzu und persistiert es.|
|`remove_theme`|`theme_id: &DomainId`|`Result<(), ThemingError>`|Entfernt ein Theme und löscht es aus der Persistenz.|

---

## 4. Modul: KI-Interaktionen (`ai_manager`)

### 4.1. Übersicht und Verantwortlichkeiten

Das `ai_manager`-Modul dient als zentrale Anlaufstelle für die Verwaltung aller KI-gestützten Funktionen innerhalb der Desktop-Umgebung. Es ist in zwei Hauptuntermodule gegliedert: `consent_manager` für die Verwaltung von Benutzereinwilligungen bezüglich der Datennutzung und Funktionsausführung durch KI, und `feature_service` für die Implementierung der eigentlichen Logik der KI-Features. Eine Kernaufgabe des Moduls ist es, sicherzustellen, dass KI-Funktionen nur mit expliziter, informierter und granularer Zustimmung des Benutzers ausgeführt werden. Es abstrahiert die Kommunikation mit potenziellen KI-Modellen oder -Diensten, deren Anbindung über die Kernschicht erfolgt.

Die Architektur dieses Moduls kann von den Konzepten des Model Context Protocol (MCP) profitieren, wie in 31 beschrieben. MCP schlägt eine Client-Server-Architektur vor, bei der "Hosts" (Anwendungen) über "Clients" mit "MCP-Servern" kommunizieren, die Zugriff auf Datenquellen und Werkzeuge bereitstellen. In diesem Kontext könnte die Domänenschicht als "Host" (oder Teil davon) agieren, der Anfragen an KI-Funktionen stellt. Die Kernschicht wäre dann verantwortlich für die Implementierung des "MCP-Clients" und die Anbindung an externe "MCP-Server" oder äquivalente KI-Dienste. Die Domänenschicht definiert dabei die _Struktur_ der Anfragen und der erwarteten Antworten sowie die Logik des Einwilligungsmanagements. KI-Funktionen benötigen oft Kontext (Daten) und die Fähigkeit, Aktionen auszuführen. MCP zielt darauf ab, diesen Zugriff zu standardisieren. Die Domänenschicht muss klar definieren, _welchen_ Kontext eine spezifische KI-Funktion benötigt (z.B. "aktueller Workspace", "aktives Fenster", "Benachrichtigungshistorie") und welche Aktionen sie ausführen darf (z.B. "Fenster anordnen", "Text vorschlagen"). Das `ConsentManager`-Untermodul stellt sicher, dass der Zugriff auf diesen Kontext und die Ausführung dieser Aktionen nur mit gültiger Benutzereinwilligung erfolgt.

### 4.2. Untermodul: Einwilligungsmanagement (`consent_manager`)

Dieses Untermodul ist verantwortlich für die Verwaltung und Persistenz der Benutzereinwilligungen für verschiedene KI-Funktionen und Datenzugriffe.

- **4.2.1. Entität: `UserConsent`**
    
    - Repräsentiert die Einwilligung eines Benutzers für ein spezifisches KI-Feature.
    - **Attribute:**
        - `user_id: DomainId` (Identifikator des Benutzers; bei Single-User-Systemen könnte dies ein konstanter Wert sein oder weggelassen werden, hier als `DomainId` für Flexibilität). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `feature_id: String` (Eindeutige ID des KI-Features, z.B. "ai.window_organizer", "ai.smart_reply.email"). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `is_granted: bool` (Status der Einwilligung). Sichtbarkeit: `pub`. Initialwert: `false`.
        - `last_updated_timestamp: u64` (Unix-Timestamp in Millisekunden der letzten Änderung). Sichtbarkeit: `pub(crate)`.
        - `scope: Option<String>` (Optionaler Geltungsbereich, z.B. "application:org.example.Mail", "global"; dient zur Verfeinerung der Einwilligung). Sichtbarkeit: `pub`.
        - `expires_at: Option<u64>` (Optionaler Unix-Timestamp in Millisekunden für den Ablauf der Einwilligung). Sichtbarkeit: `pub`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct UserConsent {
            pub user_id: DomainId,
            pub feature_id: String,
            pub is_granted: bool,
            #[serde(default = "current_timestamp_ms")] // Helferfunktion für Default
            pub(crate) last_updated_timestamp: u64,
            pub scope: Option<String>,
            pub expires_at: Option<u64>,
        }
        
        // Beispiel für eine Helferfunktion für Default-Zeitstempel
        // Diese müsste im Kontext des Moduls verfügbar sein.
        // fn current_timestamp_ms() -> u64 {
        //     std::time::SystemTime::now()
        //        .duration_since(std::time::UNIX_EPOCH)
        //        .unwrap_or_default()
        //        .as_millis() as u64
        // }
        ```
        
- **4.2.2. Service: `ConsentService`**
    
    - Verwaltet den Zustand aller Benutzereinwilligungen. Interagiert mit einem `ConsentPersistencePort` (implementiert von der Kernschicht) zum Laden/Speichern von Einwilligungen. Die Kernschicht könnte zur sicheren Speicherung sensibler Einwilligungsdaten den Freedesktop Secret Service nutzen 33, was über den `ConsentPersistencePort` abstrakt angefordert würde.
    - **Eigenschaften (Interner Zustand):**
        - `consents: std::collections::HashMap<(DomainId, String), UserConsent>` (Schlüssel: `(user_id, feature_id)`).
        - `persistence_port: Box<dyn ConsentPersistencePort>`.
        - `event_publisher: Box<dyn DomainEventPublisher>`.
        - `clock: Box<dyn Clock>` (Für Zeitstempel, injiziert).
    - **Methoden (Öffentliche API):**
        - `pub fn new(persistence_port: Box<dyn ConsentPersistencePort>, event_publisher: Box<dyn DomainEventPublisher>, clock: Box<dyn Clock>, user_id_for_initial_load: &DomainId) -> Self`
            - Konstruktor. Lädt Einwilligungen für den `user_id_for_initial_load` beim Start.
        - `pub fn grant_consent(&mut self, user_id: DomainId, feature_id: String, scope: Option<String>, expires_at: Option<u64>) -> Result<(), AIConsentError>`
            - Erstellt oder aktualisiert eine `UserConsent`-Entität mit `is_granted = true`.
            - Setzt `last_updated_timestamp` auf die aktuelle Zeit.
            - Speichert über `persistence_port`.
            - Emittiert `AIConsentEvent::ConsentGranted { user_id, feature_id, scope, expires_at }`.
        - `pub fn revoke_consent(&mut self, user_id: &DomainId, feature_id: &str) -> Result<(), AIConsentError>`
            - Aktualisiert eine existierende `UserConsent`-Entität auf `is_granted = false`.
            - Setzt `last_updated_timestamp`.
            - Speichert über `persistence_port`.
            - Emittiert `AIConsentEvent::ConsentRevoked { user_id: user_id.clone(), feature_id: feature_id.to_string() }`.
        - `pub fn get_consent_status(&self, user_id: &DomainId, feature_id: &str) -> Result<&UserConsent, AIConsentError>`
            - Gibt die `UserConsent`-Entität zurück. Prüft intern auf Ablauf (`expires_at`). Wenn abgelaufen, wird `is_granted` als `false` interpretiert, auch wenn es `true` gespeichert ist (oder es wird ein `ConsentExpired` Fehler/Event ausgelöst).
            - Gibt `AIConsentError::ConsentNotFound` zurück, wenn keine explizite Einwilligung existiert (impliziert nicht gewährt).
        - `pub fn list_consents_for_user(&self, user_id: &DomainId) -> Vec<&UserConsent>`
            - Listet alle Einwilligungen für einen Benutzer auf.
        - `pub fn list_all_consents(&self) -> Vec<&UserConsent>`
            - Listet alle Einwilligungen im System (z.B. für administrative Zwecke).
        - `pub fn cleanup_expired_consents(&mut self) -> Result<u32, AIConsentError>`
            - Iteriert durch alle Einwilligungen und entfernt abgelaufene Einträge oder markiert sie als ungültig.
            - Gibt die Anzahl der entfernten/aktualisierten Einwilligungen zurück.
            - Emittiert `AIConsentEvent::ConsentExpired` für jede entfernte/aktualisierte Einwilligung.
            - Speichert Änderungen über `persistence_port`.
    - **Signale/Events (als Varianten von `AIConsentEvent` im `DomainEvent` Enum):**
        - `ConsentGranted { user_id: DomainId, feature_id: String, scope: Option<String>, expires_at: Option<u64> }`
        - `ConsentRevoked { user_id: DomainId, feature_id: String }`
        - `ConsentExpired { user_id: DomainId, feature_id: String }`
        - **Typische Publisher:** `ConsentService`.
        - **Typische Subscriber:** `AIFeatureService` (um zu prüfen, ob Features ausgeführt werden dürfen), UI-Schicht (um Einwilligungs-Dialoge und -Statusanzeigen zu aktualisieren).
- **4.2.3. Trait: `ConsentPersistencePort` (von Kernschicht zu implementieren)**
    
    - Definiert die Schnittstelle für die Persistenz von Einwilligungsdaten.
    - `fn load_consents_for_user(&self, user_id: &DomainId) -> Result<Vec<UserConsent>, DomainError>;`
    - `fn save_consent(&self, consent: &UserConsent) -> Result<(), DomainError>;`
    - `fn delete_consent(&self, user_id: &DomainId, feature_id: &str) -> Result<(), DomainError>;`
    - `fn load_all_consents(&self) -> Result<Vec<UserConsent>, DomainError>;` (Für Admin-Zwecke oder globalen Cleanup)
- **4.2.4. Trait: `Clock` (von Kernschicht zu implementieren)**
    
    Rust
    
    ```
    pub trait Clock: Send + Sync {
        fn current_timestamp_ms(&self) -> u64;
    }
    ```
    
- **4.2.5. Fehler-Enum: `AIConsentError`**
    
    - Spezifische Fehler für das `consent_manager`-Modul.
    - **Varianten:**
        - `#[error("Consent for user '{user_id}' and feature '{feature_id}' not found or not granted.")]` `ConsentNotFoundOrNotGranted { user_id: DomainId, feature_id: String }`
        - `#` `FeatureNotKnown { feature_id: String }`
        - `#` `StorageError { message: String }`
        - `#[error("Consent for user '{user_id}' and feature '{feature_id}' has expired.")]` `ConsentExpiredError { user_id: DomainId, feature_id: String }` // Interner Fehler, der zu ConsentNotFoundOrNotGranted führen kann.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum AIConsentError {
            #[error("Consent for user '{user_id}' and feature '{feature_id}' not found or not granted.")]
            ConsentNotFoundOrNotGranted { user_id: DomainId, feature_id: String },
            #
            FeatureNotKnown { feature_id: String },
            #
            StorageError { message: String },
            #[error("Consent for user '{user_id}' and feature '{feature_id}' has expired.")]
            ConsentExpiredError { user_id: DomainId, feature_id: String },
        }
        ```
        

### 4.3. Untermodul: KI-Funktionslogik (`feature_service`)

Dieses Untermodul enthält die Logik zur Ausführung spezifischer KI-Funktionen, nachdem die Einwilligung geprüft wurde.

- **4.3.1. Service: `AIFeatureService`**
    
    - Abhängig vom `ConsentService`, um Berechtigungen zu prüfen.
    - Definiert Schnittstellen für spezifische KI-Funktionen. Die Implementierung dieser Funktionen (d.h. die Interaktion mit den eigentlichen KI-Modellen) erfolgt in der Kernschicht oder einer dedizierten KI-Infrastrukturschicht, die über die Kernschicht angebunden ist. Der `AIFeatureService` orchestriert den Aufruf und verarbeitet die Ergebnisse.
    - **Eigenschaften (Interner Zustand):**
        - `consent_service: Arc<Mutex<ConsentService>>` (oder eine andere Form des geteilten Zugriffs, wenn `ConsentService` nicht `&mut self` für seine Methoden benötigt).
        - `ai_backend_port: Box<dyn AIBackendPort>` (Injizierter Port für die Kommunikation mit der KI-Infrastruktur).
        - `event_publisher: Box<dyn DomainEventPublisher>`.
    - **Methoden (Beispiele, stark abhängig von den konkreten KI-Features):**
        - `pub async fn suggest_window_layout(&self, user_id: &DomainId, current_windows: Vec<WindowHandle>, context: AIRequestContext) -> Result<AISuggestion<WindowLayoutSuggestion>, AIFeatureError>`
            - Prüft `self.consent_service.lock().unwrap().get_consent_status(user_id, "ai.window_organizer")`. Wenn nicht gewährt oder abgelaufen, gibt `AIFeatureError::ConsentNotGranted` zurück.
            - Bereitet die Anfrage für `ai_backend_port.request_window_layout_suggestion(...)` vor.
            - Verarbeitet die Antwort und gibt `AISuggestion` zurück.
            - Emittiert `AIFeatureEvent::SuggestionProvided`.
        - `pub async fn generate_smart_reply(&self, user_id: &DomainId, notification_content: String, context: AIRequestContext) -> Result<AISuggestion<SmartReplySuggestion>, AIFeatureError>`
            - Prüft `self.consent_service.lock().unwrap().get_consent_status(user_id, "ai.smart_reply")`.
            - Bereitet Anfrage für `ai_backend_port.request_smart_reply_suggestion(...)` vor.
            - Emittiert `AIFeatureEvent::SuggestionProvided`.
    - **Signale/Events (als Varianten von `AIFeatureEvent` im `DomainEvent` Enum):**
        - `SuggestionProvided { feature_id: String, user_id: DomainId, suggestion_id: DomainId, suggestion_payload_summary: String }` (Summary statt vollem Payload, um Eventgröße zu begrenzen)
        - `ActionTakenBasedOnAISuggestion { feature_id: String, user_id: DomainId, suggestion_id: DomainId, action_id: String }`
        - **Typische Publisher:** `AIFeatureService`.
        - **Typische Subscriber:** UI-Schicht (um Vorschläge anzuzeigen und Aktionen auszulösen), andere Domänendienste (um Aktionen basierend auf Vorschlägen auszuführen, z.B. `WorkspaceService` für Layout-Änderungen).
- **4.3.2. Trait: `AIBackendPort` (von Kernschicht zu implementieren)**
    
    - Definiert die Schnittstelle zur eigentlichen KI-Modellinteraktion.
    - `async fn request_window_layout_suggestion(&self, windows: Vec<WindowHandle>, context: AIRequestContext) -> Result<WindowLayoutSuggestion, DomainError>;`
    - `async fn request_smart_reply_suggestion(&self, text_to_reply_to: String, context: AIRequestContext) -> Result<SmartReplySuggestion, DomainError>;`
    - Weitere Methoden für andere KI-Features.
- **4.3.3. Datenstrukturen für KI-Anfragen/Antworten**
    
    - `AIRequestContext`: Enthält kontextuelle Daten, die für eine KI-Anfrage relevant sind und für die eine Einwilligung vorliegt.
        - **Attribute:** `source_application_id: Option<String>`, `current_activity_description: Option<String>`, `user_preferences: std::collections::HashMap<String, String>` (z.B. bevorzugte Sprache, Datenschutzeinstellungen für KI), `timestamp_ms: u64`.
        - **Rust-Definition:**
            
            Rust
            
            ```
            #
            pub struct AIRequestContext {
                pub source_application_id: Option<String>,
                pub current_activity_description: Option<String>,
                pub user_preferences: std::collections::HashMap<String, String>,
                pub timestamp_ms: u64,
            }
            ```
            
    - `AISuggestion<T>`: Generische Struktur für KI-Vorschläge.
        - **Attribute:** `suggestion_id: DomainId`, `feature_id: String`, `confidence_score: Option<f32>` (Wert zwischen 0.0 und 1.0), `payload: T`, `explanation: Option<String>`, `feedback_token: Option<String>` (Für implizites/explizites Feedback).
        - **Rust-Definition:**
            
            Rust
            
            ```
            #
            pub struct AISuggestion<T> {
                pub suggestion_id: DomainId,
                pub feature_id: String,
                pub confidence_score: Option<f32>,
                pub payload: T,
                pub explanation: Option<String>,
                pub feedback_token: Option<String>, // Token für Feedback-Mechanismen
            }
            ```
            
    - `WindowLayoutSuggestion`: Spezifischer Payload für Layout-Vorschläge.
        - **Attribute:** `suggested_space_id: Option<DomainId>` (Wenn ein spezifischer Space vorgeschlagen wird), `window_placements: Vec<(WindowHandle, SuggestedPlacement)>`.
        - **Rust-Definition:**
            
            Rust
            
            ```
            #
            pub struct WindowLayoutSuggestion {
                pub suggested_space_id: Option<DomainId>,
                pub window_placements: Vec<(WindowHandle, SuggestedPlacement)>,
            }
            ```
            
    - `SuggestedPlacement`: Details zur Platzierung eines Fensters.
        - **Attribute:** `target_space_id: Option<DomainId>` (Falls das Fenster in einen anderen Space verschoben werden soll), `relative_x: f32`, `relative_y: f32`, `relative_width: f32`, `relative_height: f32` (Werte zwischen 0.0 und 1.0, relativ zur Space-Größe), `stacking_order: Option<u32>`.
        - **Rust-Definition:**
            
            Rust
            
            ```
            #
            pub struct SuggestedPlacement {
                pub target_space_id: Option<DomainId>, // Wenn None, aktueller Space des Fensters
                pub relative_x: f32, // 0.0 bis 1.0
                pub relative_y: f32, // 0.0 bis 1.0
                pub relative_width: f32, // 0.0 bis 1.0
                pub relative_height: f32, // 0.0 bis 1.0
                pub stacking_order: Option<u32>, // z-Index
            }
            ```
            
    - `SmartReplySuggestion`: Spezifischer Payload für Antwortvorschläge.
        - **Attribute:** `suggested_replies: Vec<String>`.
        - **Rust-Definition:**
            
            Rust
            
            ```
            #
            pub struct SmartReplySuggestion {
                pub suggested_replies: Vec<String>,
            }
            ```
            
    - Alle diese Strukturen implementieren `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`.
- **4.3.4. Fehler-Enum: `AIFeatureError`**
    
    - Spezifische Fehler für das `feature_service`-Modul.
    - **Varianten:**
        - `#[error("Consent not granted for user '{user_id}' and feature '{feature_id}'.")]` `ConsentNotGranted { user_id: DomainId, feature_id: String }` (Kann `#[from] AIConsentError` nutzen)
        - `#[error("Error interacting with AI model/backend: {message}")]` `ModelInteractionError { message: String }` (Typischerweise von `AIBackendPort` propagiert)
        - `#[error("Invalid or insufficient context provided for AI feature '{feature_id}': {reason}")]` `InvalidContext { feature_id: String, reason: String }`
        - `#[error("No suggestion available for feature '{feature_id}' with the given context.")]` `SuggestionNotAvailable { feature_id: String }`
        - `#` `BackendPortError(String)` // Generischer Fehler vom Port
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum AIFeatureError {
            #[error("Consent not granted for user '{user_id}' and feature '{feature_id}'.")]
            ConsentNotGranted { user_id: DomainId, feature_id: String },
            #[error("Error interacting with AI model/backend: {message}")]
            ModelInteractionError { message: String },
            #[error("Invalid or insufficient context provided for AI feature '{feature_id}': {reason}")]
            InvalidContext { feature_id: String, reason: String },
            #[error("No suggestion available for feature '{feature_id}' with the given context.")]
            SuggestionNotAvailable { feature_id: String },
            #
            BackendPortError(String),
        }
        
        // Mögliche Konvertierung von AIConsentError
        impl From<AIConsentError> for AIFeatureError {
            fn from(err: AIConsentError) -> Self {
                match err {
                    AIConsentError::ConsentNotFoundOrNotGranted { user_id, feature_id } => {
                        AIFeatureError::ConsentNotGranted { user_id, feature_id }
                    }
                    // Andere Mappings oder ein generischer Fehler
                    _ => AIFeatureError::ModelInteractionError {
                        message: format!("Consent error: {}", err),
                    },
                }
            }
        }
        ```
        

### 4.4. Geschäftsregeln

- **Strikte Einwilligungsprüfung:** Vor jeder Ausführung einer KI-Funktion oder jedem Zugriff auf potenziell sensible Daten durch eine KI-Funktion _muss_ eine gültige, nicht abgelaufene Einwilligung des Benutzers für das spezifische Feature und den spezifischen Datenumfang vorliegen. Dies wird durch den `AIFeatureService` sichergestellt, der den `ConsentService` konsultiert.
- **Datenminimierung und -relevanz:** An KI-Modelle (über den `AIBackendPort`) werden nur die Daten gesendet, die für die jeweilige Funktion unbedingt notwendig sind und für die eine Einwilligung vorliegt. Die Domänenschicht definiert die Struktur dieser Daten (`AIRequestContext`).
- **Anonymisierung/Pseudonymisierung:** Falls von der Kernschicht (Implementierung des `AIBackendPort`) unterstützt, kann die Domänenschicht anfordern, dass Daten vor der Übermittlung an externe KI-Dienste anonymisiert oder pseudonymisiert werden. Die Domänenschicht selbst führt diese Operationen nicht durch, sondern spezifiziert die Notwendigkeit.
- **Fallback-Verhalten:** Wenn KI-Dienste nicht verfügbar sind (Fehler vom `AIBackendPort`) oder keine sinnvollen Vorschläge liefern (`SuggestionNotAvailable`), muss die Anwendung ein definiertes Fallback-Verhalten zeigen (z.B. Standardfunktionalität ohne KI anbieten, Fehlermeldung anzeigen). Dies wird vom Aufrufer des `AIFeatureService` gehandhabt.
- **Transparenz:** Dem Benutzer sollte (über die UI-Schicht) nachvollziehbar gemacht werden, wann und warum eine KI-Funktion aktiv wird und welche Daten dafür verwendet wurden (z.B. durch `AISuggestion::explanation`).

Die Tabellen fassen die Kernkomponenten des KI-Managements zusammen:

**Tabelle 4.2.1: Entität `UserConsent` - Attribute**

|   |   |   |   |
|---|---|---|---|
|**Attribut**|**Typ**|**Sichtbarkeit**|**Invarianten/Initialwert**|
|`user_id`|`DomainId`|`pub`|Nicht leer|
|`feature_id`|`String`|`pub`|Nicht leer|
|`is_granted`|`bool`|`pub`|Initial `false`|
|`last_updated_timestamp`|`u64`|`pub(crate)`|Aktueller Zeitstempel|
|`scope`|`Option<String>`|`pub`||
|`expires_at`|`Option<u64>`|`pub`||

**Tabelle 4.2.2: `ConsentService` - Öffentliche API (Auswahl)**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp**|**Kurzbeschreibung**|
|`grant_consent`|`user_id: DomainId`, `feature_id: String`, `scope: Option<String>`, `expires_at: Option<u64>`|`Result<(), AIConsentError>`|Erteilt eine Einwilligung.|
|`revoke_consent`|`user_id: &DomainId`, `feature_id: &str`|`Result<(), AIConsentError>`|Widerruft eine Einwilligung.|
|`get_consent_status`|`user_id: &DomainId`, `feature_id: &str`|`Result<&UserConsent, AIConsentError>`|Prüft den aktuellen Einwilligungsstatus.|
|`list_consents_for_user`|`user_id: &DomainId`|`Vec<&UserConsent>`|Listet alle Einwilligungen eines Benutzers.|

**Tabelle 4.3.1: `AIFeatureService` - Beispielhafte Öffentliche API**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter**|**Rückgabetyp**|**Kurzbeschreibung**|
|`suggest_window_layout`|`user_id: &DomainId`, `windows: Vec<WindowHandle>`, `context: AIRequestContext`|`Result<AISuggestion<WindowLayoutSuggestion>, AIFeatureError>`|Schlägt ein Fensterlayout vor.|
|`generate_smart_reply`|`user_id: &DomainId`, `notification_content: String`, `context: AIRequestContext`|`Result<AISuggestion<SmartReplySuggestion>, AIFeatureError>`|Generiert Antwortvorschläge.|

---

## 5. Modul: Benachrichtigungsverwaltung (`notification_manager`)

### 5.1. Übersicht und Verantwortlichkeiten

Das `notification_manager`-Modul ist für die Entgegennahme, Verwaltung und (logische) Anzeige von Benachrichtigungen zuständig, die von Anwendungen und Systemkomponenten stammen. Es unterstützt interaktive Aktionen innerhalb von Benachrichtigungen, ermöglicht deren Priorisierung und Deduplizierung und stellt eine Historie vergangener Benachrichtigungen bereit.

Die Domänenschicht definiert hierbei die _Struktur_ und die _Logik_ von Benachrichtigungen. Die tatsächliche visuelle Darstellung, beispielsweise als Pop-up-Fenster oder Eintrag in einer Benachrichtigungszentrale, ist Aufgabe der UI-Schicht. Die Notification API des XDG Desktop Portals 35 dient als gute Inspiration für die Definition der Felder einer Benachrichtigung, wie ID, Titel, Textkörper, Priorität und mögliche Aktionen. Die Domänenschicht verwaltet Benachrichtigungen als Datenobjekte. Essentielle Felder sind `title`, `body` und `priority`. Darüber hinaus sind `application_name` (als Quelle der Benachrichtigung) und `actions` (zur Ermöglichung von Interaktivität) wichtig. Die UI-Schicht konsumiert diese Datenobjekte und erzeugt daraus die entsprechende visuelle Repräsentation.

### 5.2. Entitäten, Wertobjekte und Enums

- **5.2.1. Entität: `Notification`**
    
    - Repräsentiert eine einzelne Benachrichtigung.
    - **Attribute:**
        - `id: DomainId` (Eindeutige ID der Benachrichtigung, z.B. generiert via `uuid::Uuid::new_v4().to_string()`). Sichtbarkeit: `pub`. Invarianten: Eindeutig, nicht leer.
        - `application_name: String` (Name der sendenden Anwendung/Komponente). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `application_icon: Option<String>` (Name oder Pfad zu einem Icon, das von der UI-Schicht interpretiert wird). Sichtbarkeit: `pub`.
        - `summary: String` (Titel/Zusammenfassung der Benachrichtigung). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `body: Option<String>` (Detaillierter Text der Benachrichtigung). Sichtbarkeit: `pub`.
        - `actions: Vec<NotificationAction>` (Liste möglicher Aktionen, die der Benutzer ausführen kann). Sichtbarkeit: `pub`.
        - `urgency: NotificationUrgency` (Dringlichkeit der Benachrichtigung). Sichtbarkeit: `pub`. Initialwert: `NotificationUrgency::Normal`.
        - `category: Option<String>` (Kategorie zur Filterung/Gruppierung, z.B. "email.new", "chat.message", "system.update.available"). Sichtbarkeit: `pub`.
        - `timestamp_ms: u64` (Unix-Timestamp in Millisekunden der Erstellung). Sichtbarkeit: `pub(crate)`.
        - `expires_timeout_ms: Option<u32>` (Zeit in Millisekunden, nach der die Benachrichtigung automatisch geschlossen wird; `0` oder `None` bedeutet, sie läuft nicht automatisch ab). Sichtbarkeit: `pub`.
        - `is_persistent: bool` (Ob die Benachrichtigung in der Historie verbleibt, auch nachdem sie geschlossen wurde). Sichtbarkeit: `pub`. Initialwert: `true`.
        - `resident: bool` (Ob die Benachrichtigung permanent sichtbar bleiben soll, bis sie explizit geschlossen wird – ähnlich "sticky" Notifications; Freedesktop-Spezifikation "resident"). Sichtbarkeit: `pub`. Initialwert: `false`.
        - `transient: bool` (Ob die Benachrichtigung nur kurz angezeigt und nicht in der Historie gespeichert werden soll, auch wenn `is_persistent` true wäre; Freedesktop-Spezifikation "transient"). Sichtbarkeit: `pub`. Initialwert: `false`.
        - `custom_data: std::collections::HashMap<String, String>` (Für anwendungsspezifische Daten, die von der sendenden Anwendung mitgegeben werden können). Sichtbarkeit: `pub`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct Notification {
            pub id: DomainId,
            pub application_name: String,
            pub application_icon: Option<String>,
            pub summary: String,
            pub body: Option<String>,
            pub actions: Vec<NotificationAction>,
            pub urgency: NotificationUrgency,
            pub category: Option<String>,
            #[serde(default = "crate::ai_manager::consent_manager::current_timestamp_ms")] // Wiederverwendung der Helferfunktion
            pub(crate) timestamp_ms: u64,
            pub expires_timeout_ms: Option<u32>,
            #[serde(default = "default_true")]
            pub is_persistent: bool,
            #[serde(default)]
            pub resident: bool,
            #[serde(default)]
            pub transient: bool,
            #[serde(default)]
            pub custom_data: std::collections::HashMap<String, String>,
        }
        
        fn default_true() -> bool { true }
        // fn current_timestamp_ms() -> u64 {... } // Siehe oben
        ```
        
- **5.2.2. Wertobjekt: `NotificationAction`**
    
    - Definiert eine Aktion, die im Kontext einer Benachrichtigung ausgeführt werden kann.
    - **Attribute:**
        - `key: String` (Eindeutiger Schlüssel für die Aktion innerhalb der Benachrichtigung, z.B. "reply", "archive", "mark-as-read"). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
        - `label: String` (Anzeigetext für den Button in der UI). Sichtbarkeit: `pub`. Invarianten: Nicht leer.
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub struct NotificationAction {
            pub key: String,
            pub label: String,
        }
        ```
        
- **5.2.3. Enum: `NotificationUrgency`**
    
    - Definiert die Dringlichkeitsstufe einer Benachrichtigung, inspiriert von der Freedesktop Notification Specification.
    - **Varianten:** `Low`, `Normal`, `Critical`.
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`, `Default` (`Normal`).
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum NotificationUrgency {
            Low,
            #[default]
            Normal,
            Critical,
        }
        ```
        
- **5.2.4. Fehler-Enum: `NotificationError`**
    
    - Spezifische Fehler für das `notification_manager`-Modul. Definiert mit `thiserror`.
    - **Varianten:**
        - `#` `NotificationNotFound { notification_id: DomainId }`
        - `#[error("Action with key '{action_key}' not found for notification '{notification_id}'.")]` `ActionNotFound { notification_id: DomainId, action_key: String }`
        - `#[error("Invalid notification data provided: {reason}")]` `InvalidNotificationData { reason: String }`
        - `#[error("Notification history is full. Maximum size: {max_size}.")]` `HistoryFull { max_size: usize }`
        - `#[error("Maximum number of active notifications reached: {max_active}.")]` `MaxActiveNotificationsReached { max_active: usize }`
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum NotificationError {
            #
            NotificationNotFound { notification_id: DomainId },
            #[error("Action with key '{action_key}' not found for notification '{notification_id}'.")]
            ActionNotFound { notification_id: DomainId, action_key: String },
            #[error("Invalid notification data provided: {reason}")]
            InvalidNotificationData { reason: String },
            #[error("Notification history is full. Maximum size: {max_size}.")]
            HistoryFull { max_size: usize },
            #[error("Maximum number of active notifications reached: {max_active}.")]
            MaxActiveNotificationsReached { max_active: usize },
        }
        ```
        

### 5.3. Service: `NotificationService`

Der `NotificationService` hält den Zustand aller aktiven und ggf. historischen Benachrichtigungen und stellt Methoden zu deren Verwaltung bereit.

- **5.3.1. Eigenschaften (Interner Zustand)**
    
    - `active_notifications: std::collections::VecDeque<Notification>`: Eine Queue für aktive, potenziell sichtbare Benachrichtigungen. `VecDeque` ermöglicht effizientes FIFO-Verhalten, wenn `max_active_notifications` erreicht ist.
    - `notification_history: std::collections::VecDeque<Notification>`: Eine Queue für die Historie geschlossener, persistenter Benachrichtigungen, begrenzt durch `max_history_size`.
    - `next_internal_id_counter: u64`: Ein interner Zähler zur Generierung sequenzieller Teile von IDs, falls UUIDs nicht allein verwendet werden oder zur Deduplizierung.
    - `max_active_notifications: usize`: Konfigurierbare maximale Anzahl aktiver Benachrichtigungen (z.B. Default 5).
    - `max_history_size: usize`: Konfigurierbare maximale Größe der Historie (z.B. Default 100).
    - `persistence_port: Box<dyn NotificationPersistencePort>`: Injizierter Port für Persistenz der Historie.
    - `event_publisher: Box<dyn DomainEventPublisher>`.
    - `clock: Box<dyn crate::ai_manager::consent_manager::Clock>`. // Wiederverwendung des Clock-Traits
- **5.3.2. Methoden (Öffentliche API)**
    
    - `pub fn new(max_active: usize, max_history: usize, persistence_port: Box<dyn NotificationPersistencePort>, event_publisher: Box<dyn DomainEventPublisher>, clock: Box<dyn crate::ai_manager::consent_manager::Clock>) -> Self`
        - Konstruktor. Lädt ggf. die Historie über den `persistence_port`.
    - `pub fn post_notification(&mut self, app_name: String, app_icon: Option<String>, summary: String, body: Option<String>, actions: Vec<NotificationAction>, urgency: NotificationUrgency, category: Option<String>, expires_ms: Option<u32>, persistent: bool, resident: bool, transient: bool, custom_data: std::collections::HashMap<String, String>) -> Result<DomainId, NotificationError>`
        - **Validierung:** Prüft, ob `app_name` und `summary` nicht leer sind (`NotificationError::InvalidNotificationData`).
        - **ID-Generierung:** Erzeugt eine eindeutige `DomainId` (z.B. `uuid::Uuid::new_v4().to_string()`).
        - **Timestamp:** Setzt `timestamp_ms` mittels `self.clock.current_timestamp_ms()`.
        - **Erstellung:** Erstellt das `Notification`-Objekt.
        - **Deduplizierung (Optional):** Implementiert Logik, um Duplikate zu erkennen und ggf. zu ersetzen oder zu ignorieren. (Für diese Spezifikation vorerst nicht detailliert).
        - **Aktive Liste:** Wenn `active_notifications.len() >= self.max_active_notifications`, wird die älteste Benachrichtigung entfernt (und ggf. in die Historie verschoben, falls `is_persistent` und nicht `transient`).
        - Fügt die neue Benachrichtigung zu `active_notifications` hinzu.
        - **Emittiert:** `NotificationEvent::NotificationPosted { notification: new_notification.clone() }`.
        - **Rückgabe:** Die ID der neuen Benachrichtigung.
    - `pub fn close_notification(&mut self, notification_id: &DomainId, reason: NotificationCloseReason) -> Result<(), NotificationError>`
        - Sucht die Benachrichtigung in `active_notifications`. Wenn nicht gefunden, `NotificationError::NotificationNotFound`.
        - Entfernt die Benachrichtigung aus `active_notifications`.
        - **Historie:** Wenn die Benachrichtigung `is_persistent` ist und nicht `transient`, wird sie zu `notification_history` hinzugefügt. Wenn `notification_history.len() >= self.max_history_size`, wird die älteste Benachrichtigung aus der Historie entfernt.
        - **Persistenz:** Speichert die aktualisierte Historie über `persistence_port.save_history()`.
        - **Emittiert:** `NotificationEvent::NotificationClosed { notification_id: notification_id.clone(), reason }`.
    - `pub fn trigger_action(&mut self, notification_id: &DomainId, action_key: &str) -> Result<(), NotificationError>`
        - Sucht die Benachrichtigung in `active_notifications`. Wenn nicht gefunden, `NotificationError::NotificationNotFound`.
        - Sucht die Aktion mit `action_key` in `notification.actions`. Wenn nicht gefunden, `NotificationError::ActionNotFound`.
        - **Emittiert:** `NotificationEvent::NotificationActionTriggered { notification_id: notification_id.clone(), action_key: action_key.to_string() }`.
        - Schließt typischerweise die Benachrichtigung danach: `self.close_notification(notification_id, NotificationCloseReason::ActionTaken)?`.
    - `pub fn get_active_notifications(&self) -> Vec<&Notification>`
        - Gibt eine Kopie der aktiven Benachrichtigungen als Slice oder Vec von Referenzen zurück.
    - `pub fn get_notification_history(&self) -> Vec<&Notification>`
        - Gibt eine Kopie der Benachrichtigungshistorie zurück.
    - `pub fn clear_history(&mut self) -> Result<(), NotificationError>`
        - Leert `notification_history`.
        - Speichert die leere Historie über `persistence_port.save_history()`.
        - **Emittiert:** `NotificationEvent::NotificationHistoryCleared`.
    - `pub fn get_notification_by_id(&self, notification_id: &DomainId) -> Option<&Notification>`
        - Sucht eine Benachrichtigung zuerst in `active_notifications`, dann in `notification_history`.
- **5.3.3. Signale/Events (als Varianten von `NotificationEvent` im `DomainEvent` Enum)**
    
    - `NotificationPosted { notification: Notification }`
    - `NotificationClosed { notification_id: DomainId, reason: NotificationCloseReason }`
    - `NotificationActionTriggered { notification_id: DomainId, action_key: String }`
    - `NotificationHistoryCleared`
    - `NotificationUpdated { notification: Notification }` (Falls Benachrichtigungen nach dem Posten noch modifizierbar sein sollen, z.B. Fortschrittsbalken. Für diese Spezifikation vorerst nicht im Fokus.)
    - **Typische Publisher:** `NotificationService`.
    - **Typische Subscriber:** UI-Schicht (zur Anzeige/Aktualisierung von Benachrichtigungen und der Historie), `AIFeatureService` (z.B. um auf neue Benachrichtigungen zu reagieren und Smart Replies vorzuschlagen).
- **5.3.4. Enum: `NotificationCloseReason`**
    
    - Gibt den Grund an, warum eine Benachrichtigung geschlossen wurde.
    - **Varianten:** `Expired` (Timeout erreicht), `DismissedByUser` (Benutzer hat sie aktiv geschlossen), `ActionTaken` (Eine Aktion wurde ausgeführt), `ProgrammaticallyClosed` (Durch die Anwendung/System geschlossen), `SourceClosed` (Die sendende Anwendung hat das Schließen angefordert).
    - **Implementiert:** `serde::Serialize`, `serde::Deserialize`, `Debug`, `Clone`, `PartialEq`, `Eq`.
    - **Rust-Definition:**
        
        Rust
        
        ```
        #
        pub enum NotificationCloseReason {
            Expired,
            DismissedByUser,
            ActionTaken,
            ProgrammaticallyClosed,
            SourceClosed, // z.B. wenn die App die Notification zurückzieht
        }
        ```
        
- **5.3.5. Trait: `NotificationPersistencePort` (von Kernschicht zu implementieren)**
    
    - `fn load_history(&self) -> Result<Vec<Notification>, DomainError>;`
    - `fn save_history(&self, history: &std::collections::VecDeque<Notification>) -> Result<(), DomainError>;`

### 5.4. Geschäftsregeln

- **Priorisierung:** Kritische Benachrichtigungen (`NotificationUrgency::Critical`) könnten an der Spitze der `active_notifications`-Queue eingefügt werden oder andere weniger wichtige Benachrichtigungen verdrängen, falls `max_active_notifications` erreicht ist. Normale und niedrige Dringlichkeiten werden am Ende der Queue hinzugefügt.
- **Deduplizierung:** (Optional, für spätere Erweiterung) Regeln, um identische oder sehr ähnliche Benachrichtigungen (z.B. gleicher `application_name`, `summary` und `category` innerhalb eines kurzen Zeitfensters) zusammenzufassen oder zu unterdrücken. Dies könnte durch einen Hash über relevante Felder oder eine "replaces_id"-Mechanik implementiert werden.
- **Maximale Anzahl aktiver Benachrichtigungen:** Wenn `max_active_notifications` überschritten wird, wird die älteste nicht-residente Benachrichtigung geschlossen (Grund: `ProgrammaticallyClosed`) und ggf. in die Historie verschoben.
- **Maximale Größe der Historie:** Wenn `max_history_size` beim Hinzufügen einer Benachrichtigung zur Historie überschritten wird, wird der älteste Eintrag aus der Historie entfernt.
- **Verhalten bei `expires_timeout_ms`:** Ein Mechanismus (z.B. ein Timer-Service in der Kernschicht, der vom `NotificationService` über den `event_publisher` oder einen dedizierten Port gesteuert wird) muss dafür sorgen, dass Benachrichtigungen mit `expires_timeout_ms` nach Ablauf der Zeit mit `NotificationCloseReason::Expired` geschlossen werden. Die Domänenschicht selbst verwaltet keine aktiven Timer.
- **`transient` vs. `is_persistent`:** Eine als `transient` markierte Benachrichtigung wird niemals in die Historie aufgenommen, unabhängig vom Wert von `is_persistent`.

Die folgenden Tabellen bieten eine Übersicht über die `Notification`-Entität und die API des `NotificationService`.

**Tabelle 5.2.1: Entität `Notification` - Attribute**

|   |   |   |   |
|---|---|---|---|
|**Attribut**|**Typ**|**Sichtbarkeit**|**Invarianten/Initialwert (Beispiel)**|
|`id`|`DomainId`|`pub`|Eindeutig, nicht leer|
|`application_name`|`String`|`pub`|Nicht leer|
|`application_icon`|`Option<String>`|`pub`||
|`summary`|`String`|`pub`|Nicht leer|
|`body`|`Option<String>`|`pub`||
|`actions`|`Vec<NotificationAction>`|`pub`||
|`urgency`|`NotificationUrgency`|`pub`|`Normal`|
|`category`|`Option<String>`|`pub`||
|`timestamp_ms`|`u64`|`pub(crate)`|Zeitstempel bei Erstellung|
|`expires_timeout_ms`|`Option<u32>`|`pub`|`None` (läuft nicht ab)|
|`is_persistent`|`bool`|`pub`|`true`|
|`resident`|`bool`|`pub`|`false`|
|`transient`|`bool`|`pub`|`false`|
|`custom_data`|`std::collections::HashMap<String, String>`|`pub`|Leer|

**Tabelle 5.3.2: `NotificationService` - Öffentliche API (Auswahl)**

|   |   |   |   |
|---|---|---|---|
|**Methode**|**Parameter (Auszug)**|**Rückgabetyp**|**Kurzbeschreibung**|
|`post_notification`|`app_name: String`, `summary: String`, `urgency: NotificationUrgency`,...|`Result<DomainId, NotificationError>`|Postet eine neue Benachrichtigung.|
|`close_notification`|`notification_id: &DomainId`, `reason: NotificationCloseReason`|`Result<(), NotificationError>`|Schließt eine aktive Benachrichtigung.|
|`trigger_action`|`notification_id: &DomainId`, `action_key: &str`|`Result<(), NotificationError>`|Löst eine Aktion einer Benachrichtigung aus.|
|`get_active_notifications`||`Vec<&Notification>`|Ruft alle aktiven Benachrichtigungen ab.|
|`get_notification_history`||`Vec<&Notification>`|Ruft die Historie der Benachrichtigungen ab.|
|`clear_history`||`Result<(), NotificationError>`|Leert die Benachrichtigungshistorie.|

---

## 6. Modul: Fenstermanagement-Richtlinien (`window_policy_engine`)

### 6.1. Übersicht und Verantwortlichkeiten

Das Modul `window_policy_engine` ist dafür zuständig, Regeln für das Verhalten von Fenstern zu definieren und anzuwenden. Diese Regeln können beispielsweise die automatische Zuweisung von Fenstern zu bestimmten Spaces, Standard-Tiling-Verhalten für spezifische Anwendungen oder andere Aspekte des Fensterverhaltens umfassen. Dieses Modul entkoppelt spezifische Fensterverwaltungsentscheidungen von der allgemeinen Workspace-Verwaltung im `WorkspaceManager`. Es reagiert auf Domänenereignisse wie "Fenster geöffnet" (signalisiert von der Systemschicht über die Kernschicht und dann als Domänenereignis weitergeleitet) oder "Space gewechselt".

Die Verwendung eines solchen Policy-Engines ermöglicht eine hohe Anpassbarkeit des Fenstermanagements, ohne die Kernlogik des `WorkspaceManager` zu verändern. Die Regeln werden als Datenstrukturen repräsentiert und können potenziell zur Laufzeit modifiziert werden (z.B. durch Benutzereingaben in einer Konfigurations-UI). Inspiration für die Definition von Fenstereigenschaften und -zuständen kann von Wayland-Protokollen wie `wlr-foreign-toplevel-management-unstable-v1` 36 oder `xdg-shell` und dessen Erweiterungen (z.B. `xdg-decoration` 40) abgeleitet werden. Konzepte wie `app_id`, `title`, `maximized`, `minimized`, `fullscreen` werden von der Domänenschicht jedoch abstrakt implementiert, ohne direkte Abhängigkeiten zu diesen Protokollen. Die Domänenschicht definiert, welche Informationen sie über ein Fenster benötigt (`WindowStateContext`), und die Systemschicht ist verantwortlich, diese Informationen aus dem jeweiligen Fenstersystem (Wayland, X11) zu extrahieren und bereitzustellen. Die Aktionen, die aus den Regeln resultieren (z.