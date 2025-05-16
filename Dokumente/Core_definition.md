# Pflichtenheft für die Coreschicht

**Version:** 1.0

**Datum:** 2024-05-16

**Autor:** Dr. rer. nat. Expertenteam

**Status:** Entwurf

## Inhaltsverzeichnis

1. Einleitung 1.1. Zweck des Dokuments 1.2. Geltungsbereich 1.3. Zielgruppe 1.4. Definitionen und Akronyme 1.5. Referenzierte Dokumente und Standards
2. Datenbankdesign-Spezifikation 2.1. Data Dictionary 2.1.1. Tabellenstruktur und Felddefinitionen 2.1.2. Datentypen 2.1.3. Primär- und Fremdschlüssel 2.1.4. Constraints und Validierungsregeln 2.2. Entity-Relationship-Diagramme (ERD) 2.2.1. Notation 2.2.2. Werkzeuge 2.2.3. Konzeptionelles ER-Diagramm (Beschreibung) 2.2.4. Best Practices für ER-Diagramme
3. Schnittstellendesign-Spezifikation (API) 3.1. Interne Schnittstellen 3.1.1. REST-API für Kernfunktionalitäten 3.1.1.1. Basis-URL 3.1.1.2. Authentifizierung und Autorisierung 3.1.1.3. Datenformate (Request/Response) 3.1.1.4. HTTP-Statuscodes 3.1.1.5. Endpunktspezifikationen (Beispiele) 3.1.1.6. Versionierung 3.2. Externe Schnittstellen (falls zutreffend) 3.3. API-Dokumentation (OpenAPI/Swagger) 3.4. Best Practices für API-Design
4. UI/UX-Design-Spezifikationen (für Administrations- und Konfigurationsoberflächen) 4.1. Wireframes 4.1.1. Zweck und Detailgrad 4.1.2. Werkzeuge 4.1.3. Beispiele (Beschreibung der wichtigsten Ansichten) 4.1.4. Informationsarchitektur und Navigation 4.2. Mockups 4.2.1. Visuelles Design und Detailtiefe 4.2.2. Werkzeuge 4.2.3. Beispiele (Beschreibung der wichtigsten Ansichten) 4.3. Prototypen (klickbar) 4.3.1. Interaktivität und Benutzerflüsse 4.3.2. Werkzeuge 4.3.3. Zu testende Szenarien 4.4. Styleguide 4.4.1. Farbpalette 4.4.2. Typografie 4.4.3. Ikonografie 4.4.4. Abstände und Layout-Raster 4.4.5. UI-Komponentenbibliothek
5. Sicherheitskonzept 5.1. Grundlagen und Prinzipien 5.1.1. Layered Security (Defense in Depth) 5.1.2. Prinzip der geringsten Rechte (Principle of Least Privilege) 5.2. Authentifizierung und Autorisierung 5.2.1. Authentifizierungsmechanismen 5.2.2. Autorisierungsmodell (z.B. RBAC) 5.3. Datensicherheit 5.3.1. Verschlüsselung von Daten "at rest" 5.3.2. Verschlüsselung von Daten "in transit" 5.4. API-Sicherheitsmaßnahmen 5.5. Sichere Eingabevalidierung und -verarbeitung 5.6. Protokollierung (Logging) und Überwachung sicherheitsrelevanter Ereignisse
6. Schlussfolgerungen
7. Anhang 7.1. Glossar 7.2. Referenzierte Dokumente und Standards (erweitert)

---

## 1. Einleitung

Dieses Pflichtenheft (auch als funktionale Spezifikation oder Sollkonzept bezeichnet 1) definiert die Anforderungen und Spezifikationen für die Entwicklung der Coreschicht einer Softwareanwendung. Es dient als verbindliche Grundlage für die Implementierung und stellt sicher, dass das Endprodukt die definierten Bedürfnisse und Erwartungen erfüllt.2 Die Coreschicht umfasst die zentrale Geschäftslogik, die Datenhaltung sowie die internen und externen Schnittstellen des Systems.

### 1.1. Zweck des Dokuments

Der Hauptzweck dieses Dokuments ist die detaillierte und unmissverständliche Beschreibung der zu realisierenden Coreschicht. Es legt fest, _was_ entwickelt werden soll und _wie_ die einzelnen Komponenten strukturiert und implementiert werden müssen, um die übergeordneten Projektziele zu erreichen.3 Dieses Pflichtenheft bildet die Grundlage für Design, Entwicklung, Test und Abnahme der Coreschicht. Es dient der Vermeidung von Missverständnissen zwischen Auftraggeber und Auftragnehmer und reduziert das Risiko von Fehlentwicklungen und Nacharbeiten.5

### 1.2. Geltungsbereich

Dieses Pflichtenheft bezieht sich ausschließlich auf die Entwicklung und Implementierung der Coreschicht. Dies beinhaltet:

- **Datenbankdesign:** Struktur, Tabellen, Felder, Datentypen und Beziehungen der primären Datenbank.
- **Schnittstellendesign:** Spezifikation der internen APIs (insbesondere REST-APIs) für die Kommunikation mit anderen Systemkomponenten (z.B. Frontend, andere Backend-Services) und ggf. externen Diensten.
- **UI/UX-Design-Spezifikationen:** Konzeption und Gestaltung von Benutzeroberflächen, die direkt mit der Coreschicht interagieren, typischerweise Administrations- oder Konfigurationsoberflächen.
- **Sicherheitskonzept:** Maßnahmen zur Gewährleistung der Vertraulichkeit, Integrität und Verfügbarkeit der Daten und Funktionen der Coreschicht.

Aspekte wie die detaillierte Ausgestaltung von Frontend-Applikationen, die nicht direkt Administrationszwecken der Coreschicht dienen, oder die übergeordnete Infrastrukturplanung sind nicht primärer Gegenstand dieses Dokuments, können aber referenziert werden, wo Interdependenzen bestehen.

### 1.3. Zielgruppe

Dieses Dokument richtet sich an folgende Stakeholder:

- **Entwicklerteam:** Als detaillierte Vorgabe für die Implementierung.
- **Projektmanager:** Zur Planung, Steuerung und Überwachung des Entwicklungsprozesses.
- **Qualitätssicherungsteam:** Als Grundlage für die Erstellung von Testfällen und die Durchführung von Tests.
- **Systemarchitekten:** Zur Sicherstellung der Kompatibilität mit der Gesamtarchitektur.
- **Auftraggeber/Produktmanagement:** Zur Überprüfung, ob die spezifizierten Funktionalitäten den Anforderungen entsprechen und zur formalen Abnahme.

Ein klares Verständnis der Inhalte ist für alle Beteiligten essenziell, um den Projekterfolg sicherzustellen.2

### 1.4. Definitionen und Akronyme

Eine umfassende Liste der in diesem Dokument verwendeten Fachbegriffe, Abkürzungen und Akronyme befindet sich im Anhang (Kapitel 7.1 Glossar).5 Dies dient der Vermeidung von Missverständnissen und der Sicherstellung eines einheitlichen Sprachgebrauchs.

### 1.5. Referenzierte Dokumente und Standards

Dieses Pflichtenheft basiert auf und referenziert verschiedene externe Dokumente, Normen und interne Vorgaben. Eine detaillierte Auflistung findet sich im Anhang (Kapitel 7.2 Referenzierte Dokumente und Standards).1 Zu den wichtigsten gehören Normen wie DIN 69901-5 für Projektmanagement und Pflichtenhefte 1 sowie ggf. der IEEE 830 Standard für Software Requirements Specifications (SRS) 8, falls internationale Kontexte relevant sind.

## 2. Datenbankdesign-Spezifikation

Die Datenbank bildet das Fundament der Coreschicht und ist für die persistente Speicherung und Verwaltung aller relevanten Anwendungsdaten zuständig. Eine sorgfältige Planung und Dokumentation des Datenbankdesigns ist entscheidend für die Performance, Integrität und Skalierbarkeit des Gesamtsystems. Dieses Kapitel beschreibt die Struktur der Datenbank, einschließlich des Data Dictionary und der Entity-Relationship-Diagramme (ERD). Die hier getroffenen Entscheidungen müssen Aspekte der Datensicherheit, wie Verschlüsselung und Zugriffskontrolle, von Beginn an berücksichtigen, um ein "Secure by Design"-Prinzip zu gewährleisten.

### 2.1. Data Dictionary

Das Data Dictionary dient als zentrales Nachschlagewerk für alle Datenelemente der Datenbank. Es dokumentiert detailliert jede Tabelle, deren Felder, Datentypen, Beziehungen und Constraints.11 Ein gut gepflegtes Data Dictionary ist unerlässlich für das Verständnis der Datenstruktur, die Entwicklung konsistenter Abfragen und die Wartung des Systems.12

#### 2.1.1. Tabellenstruktur und Felddefinitionen

Für jede Tabelle im Datenbankschema werden die folgenden Informationen spezifiziert:

- **Tabellenname:** Ein eindeutiger und deskriptiver Name für die Tabelle (z.B. `Kunden`, `Bestellungen`, `Produkte`). Es empfiehlt sich, konsistente Namenskonventionen zu verwenden (z.B. Singular- oder Pluralformen).14
- **Beschreibung:** Eine kurze Erläuterung des Zwecks und Inhalts der Tabelle.
- **Felder (Attribute):** Für jedes Feld in der Tabelle:
    - **Feldname:** Eindeutiger Name des Feldes innerhalb der Tabelle (z.B. `KundenID`, `Vorname`, `Erstelldatum`). Auch hier sind konsistente Namenskonventionen wichtig (z.B. CamelCase oder snake_case).12
    - **Beschreibung:** Bedeutung und Zweck des Feldes.
    - **Datentyp:** Der spezifische Datentyp des Feldes (siehe Kapitel 2.1.2).
    - **Primärschlüssel (PK):** Kennzeichnung, ob das Feld Teil des Primärschlüssels ist.
    - **Fremdschlüssel (FK):** Kennzeichnung, ob das Feld ein Fremdschlüssel ist, inklusive der Referenztabelle und des Referenzfeldes.
    - **NotNull-Constraint:** Angabe, ob das Feld einen Wert enthalten muss (`TRUE`/`FALSE`).
    - **Unique-Constraint:** Angabe, ob der Wert des Feldes innerhalb der Tabelle eindeutig sein muss (`TRUE`/`FALSE`).
    - **Standardwert:** Ein optionaler Standardwert, der verwendet wird, wenn beim Einfügen eines neuen Datensatzes kein Wert für dieses Feld angegeben wird.
    - **Beispielwert:** Ein typischer Beispielwert zur Veranschaulichung.
    - **Sensitivität:** Klassifizierung der Sensitivität der Daten im Feld (z.B. öffentlich, intern, vertraulich, streng vertraulich) als Grundlage für Sicherheitsmaßnahmen.

Die Definition der Tabellen und Felder erfolgt in einer strukturierten Form, beispielsweise durch eine Serie von Tabellen, die jeweils eine Datenbanktabelle beschreiben.

**Tabelle 2.1: Beispielstruktur für eine Data Dictionary Tabelle (am Beispiel `Kunden`)**

|   |   |   |   |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|---|
|**Attribut**|**Feldname**|**Datentyp**|**PK**|**FK (Tabelle.Feld)**|**NotNull**|**Unique**|**Standardwert**|**Beispielwert**|**Sensitivität**|**Beschreibung**|
|Kundenidentifikation|`KundenID`|`INTEGER`|Ja||Ja|Ja|AUTOINCREMENT|1001|Intern|Eindeutiger Identifikator des Kunden|
|Vorname des Kunden|`Vorname`|`VARCHAR(50)`|Nein||Ja|Nein||Max|Vertraulich|Vorname des Kunden|
|Nachname des Kunden|`Nachname`|`VARCHAR(50)`|Nein||Ja|Nein||Mustermann|Vertraulich|Nachname des Kunden|
|E-Mail-Adresse|`Email`|`VARCHAR(100)`|Nein||Ja|Ja||max.mustermann@example.com|Vertraulich|E-Mail-Adresse des Kunden|
|Registrierungsdatum|`RegistriertAm`|`TIMESTAMP`|Nein||Ja|Nein|CURRENT_TIMESTAMP|2023-10-26 10:00:00|Intern|Zeitpunkt der Registrierung des Kunden|
|Letzter Login|`LetzterLogin`|`TIMESTAMP`|Nein||Nein|Nein||2024-05-15 14:30:00|Vertraulich|Zeitpunkt des letzten Logins des Kunden|

Die konsequente Anwendung von Namenskonventionen und die detaillierte Beschreibung jedes Elements sind entscheidend. Dies erleichtert nicht nur die Entwicklung, sondern auch die spätere Wartung und Erweiterung des Systems, da neue Teammitglieder sich schneller einarbeiten können und die Datenstrukturen klar verständlich sind.12 Die Klassifizierung der Datensensitivität ist ein wichtiger Input für das Sicherheitskonzept, insbesondere für Verschlüsselungsstrategien und Zugriffskontrollen.

#### 2.1.2. Datentypen

Die Auswahl des korrekten Datentyps für jedes Feld ist fundamental, da sie bestimmt, welche Art von Daten gespeichert werden können, wie viel Speicherplatz benötigt wird und welche Operationen auf den Daten ausgeführt werden können.15 Die Datentypen müssen dem gewählten Datenbanksystem entsprechen. Gängige Datentypen umfassen:

- **Zeichenketten:**
    - `CHAR(n)`: Feste Länge, geeignet für kurze, immer gleich lange Zeichenketten.
    - `VARCHAR(n)`: Variable Länge bis zu `n` Zeichen, effizient für unterschiedlich lange Texte.
    - `TEXT` / `CLOB`: Für sehr lange Textdaten.
- **Numerische Typen:**
    - `INTEGER` / `INT`: Ganze Zahlen.
    - `SMALLINT`, `BIGINT`: Ganze Zahlen mit kleinerem bzw. größerem Wertebereich.
    - `NUMERIC(p,s)` / `DECIMAL(p,s)`: Festkommazahlen mit Präzision `p` und `s` Nachkommastellen, ideal für Währungsbeträge.
    - `FLOAT`, `REAL`, `DOUBLE PRECISION`: Gleitkommazahlen.
- **Datum und Zeit:**
    - `DATE`: Speichert nur das Datum.
    - `TIME`: Speichert nur die Uhrzeit.
    - `TIMESTAMP` / `DATETIME`: Speichert Datum und Uhrzeit, oft mit Zeitzoneninformation.
- **Boole'sche Werte:**
    - `BOOLEAN`: Speichert `TRUE` oder `FALSE`.
- **Binärdaten:**
    - `BLOB` / `BYTEA`: Für die Speicherung von Binärdaten wie Bildern oder Dateien (obwohl die Speicherung großer Binärdateien direkt in der Datenbank oft kritisch hinterfragt werden sollte).
- **Spezifische Typen:**
    - `UUID`: Universally Unique Identifier.
    - `JSON` / `JSONB`: Zur Speicherung von JSON-Dokumenten direkt in der Datenbank.
    - `XML`: Zur Speicherung von XML-Daten.

Die Wahl des Datentyps beeinflusst auch die Datenvalidierung auf Datenbankebene; beispielsweise kann in ein `INTEGER`-Feld kein Text eingegeben werden.15 Die Feldgröße (z.B. bei `VARCHAR(n)` oder die Wahl zwischen `INTEGER` und `BIGINT`) sollte so gewählt werden, dass sie den erwarteten Datenmengen entspricht, aber keinen unnötigen Speicherplatz verschwendet.16

#### 2.1.3. Primär- und Fremdschlüssel

- **Primärschlüssel (PK):** Jede Tabelle muss einen Primärschlüssel besitzen, der jeden Datensatz eindeutig identifiziert. Ein Primärschlüssel kann aus einem oder mehreren Feldern bestehen (zusammengesetzter Primärschlüssel). Üblicherweise werden hierfür `INTEGER`- oder `UUID`-Felder mit automatischer Generierung (z.B. `AUTO_INCREMENT` oder Sequenzen) verwendet.16
- **Fremdschlüssel (FK):** Fremdschlüssel dienen dazu, Beziehungen zwischen Tabellen herzustellen und die referentielle Integrität sicherzustellen. Ein Fremdschlüssel in einer Tabelle verweist auf den Primärschlüssel einer anderen (oder derselben) Tabelle. Es muss sichergestellt werden, dass für jeden Fremdschlüsselwert ein entsprechender Primärschlüsselwert in der referenzierten Tabelle existiert. Datenbanken bieten Optionen wie `ON DELETE CASCADE` oder `ON DELETE SET NULL`, um das Verhalten bei Löschoperationen in der referenzierten Tabelle zu steuern.

Die klare Definition von Primär- und Fremdschlüsseln ist die Basis für die Abbildung von Entitätsbeziehungen im relationalen Modell und somit ein Kernstück des Datenbankdesigns.11

#### 2.1.4. Constraints und Validierungsregeln

Zusätzlich zu `NotNull`- und `Unique`-Constraints können weitere Validierungsregeln auf Datenbankebene definiert werden, um die Datenintegrität zu gewährleisten:

- **CHECK-Constraints:** Erlauben die Definition spezifischer Bedingungen, die Daten erfüllen müssen (z.B. `Alter >= 18`, `Preis > 0`).
- **Default-Werte:** Wie in 2.1.1 erwähnt, können Standardwerte für Felder definiert werden.

Obwohl viele Validierungen auch auf Applikationsebene (in der Coreschicht-Logik) stattfinden, bieten datenbankseitige Constraints eine zusätzliche Sicherheitsebene und stellen die Datenkonsistenz auch bei direkten Datenbankzugriffen sicher. Die Definition dieser Regeln im Data Dictionary ist daher von großer Bedeutung.17

### 2.2. Entity-Relationship-Diagramme (ERD)

Entity-Relationship-Diagramme (ERDs) visualisieren die Struktur der Datenbank, indem sie Entitäten (Tabellen), deren Attribute (Felder) und die Beziehungen zwischen ihnen darstellen.18 Sie sind ein unverzichtbares Werkzeug für das Verständnis und die Kommunikation des Datenbankmodells.

#### 2.2.1. Notation

Es existieren verschiedene Notationen für ERDs. Die gebräuchlichsten sind:

- **Chen-Notation:** Stellt Entitäten als Rechtecke, Attribute als Ovale und Beziehungen als Rauten dar. Sie ist sehr ausdrucksstark und detailliert.18
- **Crow's Foot Notation (Krähenfußnotation):** Stellt Entitäten als Rechtecke dar und Beziehungen als Linien zwischen ihnen, wobei die Kardinalität (z.B. eins-zu-viele, viele-zu-viele) durch spezielle Symbole (Krähenfüße) am Ende der Linien angezeigt wird. Diese Notation ist oft intuitiver und wird in vielen modernen Modellierungswerkzeugen verwendet.18
- **UML-Klassendiagramm-Notation:** Kann ebenfalls zur Darstellung von Datenmodellen verwendet werden, wobei Klassen Entitäten repräsentieren.18

Für dieses Projekt wird die **Crow's Foot Notation** empfohlen, da sie eine klare und weit verbreitete Methode zur Darstellung von Kardinalitäten bietet und von vielen gängigen ERD-Werkzeugen unterstützt wird.

#### 2.2.2. Werkzeuge

Zur Erstellung und Pflege von ER-Diagrammen stehen zahlreiche Werkzeuge zur Verfügung, sowohl Desktop-Anwendungen als auch webbasierte Lösungen. Einige Beispiele sind:

- **Lucidchart:** Ein populäres Online-Tool mit Funktionen für Datenimport und Kollaboration.20
- **ERDPlus:** Ein webbasiertes Tool, das die automatische Konvertierung von ER-Diagrammen in relationale Schemata ermöglicht und SQL exportieren kann.20
- **Visual Paradigm Online:** Bietet eine breite Palette an Diagrammtypen, einschließlich ERDs, mit vielen Vorlagen.20
- **SmartDraw:** Ermöglicht die automatische Erstellung von ERDs aus bestehenden Datenbanken.20
- **Creately:** Ein weiteres kollaboratives Online-Diagrammwerkzeug.20
- **dbdiagram.io:** Ermöglicht die Erstellung von ERDs durch das Schreiben einer einfachen textbasierten Sprache (DSL).21
- **ClickUp:** Bietet Whiteboard-Funktionen und Vorlagen für ER-Diagramme als Teil einer umfassenderen Projektmanagement-Plattform.23

Die Auswahl des Werkzeugs sollte auf Basis der Projektanforderungen, der Teampräferenzen und ggf. vorhandener Lizenzen erfolgen. Wichtig ist, dass das Werkzeug den Export in gängige Bildformate oder als Vektorgrafik ermöglicht und idealerweise eine Versionierung oder Integration mit Versionskontrollsystemen unterstützt.

#### 2.2.3. Konzeptionelles ER-Diagramm (Beschreibung)

Das konzeptionelle ER-Diagramm stellt die Hauptentitäten des Systems und ihre Beziehungen auf einer hohen Abstraktionsebene dar. Es dient dazu, ein grundlegendes Verständnis der Datenstruktur zu vermitteln, ohne sich in Implementierungsdetails zu verlieren.

Für die Coreschicht werden folgende Hauptentitäten und ihre Beziehungen erwartet (Beispielhaft, muss an das spezifische Projekt angepasst werden):

- **Benutzer (`Users`):** Enthält Informationen über die Benutzer des Systems (z.B. `UserID`, `Username`, `PasswortHash`, `Email`, `Rolle`).
- **[Hauptentität 1] (z.B. `Produkte`):** Enthält Attribute spezifisch für die Kernfunktionalität (z.B. `ProduktID`, `Name`, `Beschreibung`, `Preis`).
- **[Hauptentität 2] (z.B. `Bestellungen`):** (z.B. `BestellID`, `KundenID` (FK zu `Users`), `Bestelldatum`, `Status`).
- ** (z.B. `Bestellpositionen`):** Verbindet `Bestellungen` und `Produkte` (z.B. `BestellpositionsID`, `BestellID` (FK), `ProduktID` (FK), `Menge`, `Einzelpreis`).

**Beziehungen (Beispiele):**

- Ein `Benutzer` kann viele `Bestellungen` aufgeben (1:N).
- Eine `Bestellung` gehört zu genau einem `Benutzer` (N:1).
- Eine `Bestellung` kann viele `Produkte` über `Bestellpositionen` enthalten (M:N, realisiert über die Verknüpfungstabelle `Bestellpositionen`).
- Ein `Produkt` kann in vielen `Bestellungen` über `Bestellpositionen` enthalten sein (M:N).

Das tatsächliche ER-Diagramm wird als separate grafische Darstellung beigefügt oder im gewählten Werkzeug gepflegt und hier referenziert. Es muss alle im Data Dictionary definierten Tabellen und ihre durch Fremdschlüssel spezifizierten Beziehungen abbilden.

#### 2.2.4. Best Practices für ER-Diagramme

Bei der Erstellung und Pflege von ER-Diagrammen sollten folgende Best Practices beachtet werden, um Klarheit, Lesbarkeit und Korrektheit zu gewährleisten 14:

- **Standardisierte Symbole und Notationen verwenden:** Dies erleichtert das Verständnis für alle Beteiligten.24
- **Überlappende Linien vermeiden:** Linienkreuzungen können die Lesbarkeit stark beeinträchtigen. Diagramme sollten ausreichend Platz bieten.14
- **Konsistente Ausrichtung und Anordnung:** Objekte sollten logisch gruppiert und ausgerichtet werden, um die Struktur hervorzuheben.24
- **Farbcodierung (optional):** Farben können verwendet werden, um verschiedene Arten von Entitäten oder Beziehungen hervorzuheben, sollten aber sparsam und konsistent eingesetzt werden.14
- **Klare und konsistente Namenskonventionen:** Tabellen- und Attributnamen sollten den im Data Dictionary definierten Namen entsprechen und verständlich sein.14
- **Korrekte Darstellung von Kardinalitäten:** Die Beziehungen (1:1, 1:N, M:N) müssen korrekt abgebildet werden, da sie wesentliche Geschäftsregeln widerspiegeln.24
- **Keine Instanzen, sondern Entitätstypen darstellen:** ERDs zeigen Entitätsmengen (Tabellen), nicht einzelne Datensätze.19
- **Attribute nicht mit Entitäten verwechseln:** Attribute sind Eigenschaften einer Entität, keine eigenständigen Entitäten (es sei denn, es handelt sich um komplexe Attribute, die als eigene Entität modelliert werden sollten).24
- **Lesefluss von links nach rechts oder oben nach unten beibehalten:** Ein konsistenter Lesefluss verbessert die Verständlichkeit.24

Die Einhaltung dieser Praktiken stellt sicher, dass das ER-Diagramm ein effektives Kommunikationsmittel bleibt und die zugrundeliegende Datenbankstruktur präzise repräsentiert.

## 3. Schnittstellendesign-Spezifikation (API)

Die Schnittstellen der Coreschicht sind entscheidend für die Interaktion mit anderen Systemteilen (z.B. Frontend-Anwendungen, mobile Apps, andere Backend-Dienste) sowie potenziell mit externen Systemen. Eine klare, konsistente und gut dokumentierte API-Spezifikation ist unerlässlich für eine effiziente Entwicklung, Integration und Wartung.25 Dieses Kapitel fokussiert sich primär auf RESTful APIs, die als moderner Standard für Web-Schnittstellen gelten.27

### 3.1. Interne Schnittstellen

Interne Schnittstellen ermöglichen die Kommunikation und den Datenaustausch zwischen der Coreschicht und anderen Komponenten innerhalb der Gesamtarchitektur der Anwendung.29

#### 3.1.1. REST-API für Kernfunktionalitäten

Die primäre interne Schnittstelle wird als RESTful API (Representational State Transfer) realisiert. REST-APIs nutzen Standard-HTTP-Methoden, um Operationen auf Ressourcen auszuführen, die durch URIs identifiziert werden.27

##### 3.1.1.1. Basis-URL

Alle Endpunkte der Coreschicht-API werden unter einer gemeinsamen Basis-URL erreichbar sein, die auch die Versionierung der API beinhaltet.

Beispiel: https://api.example.com/core/v1/

Die Verwendung von v1 im Pfad kennzeichnet die erste Hauptversion der API.30

##### 3.1.1.2. Authentifizierung und Autorisierung

Jeder Zugriff auf die API muss authentifiziert und autorisiert werden.

- **Authentifizierung:**
    
    - Die primäre Authentifizierungsmethode für Endbenutzer-initiierte Anfragen (z.B. vom Frontend) erfolgt über **OAuth 2.0** (Authorization Code Flow oder Client Credentials Flow, je nach Anwendungsfall).31 JWTs (JSON Web Tokens) werden als Bearer-Token im `Authorization`-Header übertragen. Die JWTs müssen serverseitig validiert werden (Signatur, Ablaufdatum, Aussteller, Zielgruppe).31
    - Für rein serverseitige Kommunikation zwischen vertrauenswürdigen internen Diensten können **API-Keys** verwendet werden, die ebenfalls im HTTP-Header (z.B. `X-API-Key`) übertragen werden.31 API-Keys müssen sicher generiert, gespeichert und rotiert werden können.
    - Basisauthentifizierung (Username/Passwort im Header) ist aufgrund ihrer Unsicherheit zu vermeiden, es sei denn, sie wird ausschließlich über HTTPS in stark kontrollierten, nicht-produktiven Umgebungen eingesetzt.31
- **Autorisierung:**
    
    - Nach erfolgreicher Authentifizierung wird die Autorisierung auf Basis von Rollen und Berechtigungen durchgeführt (Role-Based Access Control - RBAC).32 Die im JWT enthaltenen `scopes` oder `roles` (oder aus einer Benutzerdatenbank abgerufene Rollen) bestimmen, auf welche Ressourcen und Operationen der Benutzer oder Dienst zugreifen darf.
    - Detaillierte Spezifikationen zu Rollen und Berechtigungen sind im Sicherheitskonzept (Kapitel 5.2.2) definiert.

Die Sicherheit der Authentifizierungs- und Autorisierungsmechanismen ist von höchster Priorität. Passwörter und API-Schlüssel dürfen niemals in Klartext übertragen oder gespeichert werden und sollten stets über HTTPS gesendet werden.27

##### 3.1.1.3. Datenformate (Request/Response)

- **Request Body Format:** Für Anfragen, die Daten im Body übertragen (z.B. POST, PUT, PATCH), wird ausschließlich das **JSON (JavaScript Object Notation)** Format verwendet.27 Der `Content-Type`-Header der Anfrage muss auf `application/json` gesetzt sein.
- **Response Body Format:** Antworten der API werden ebenfalls im JSON-Format ausgeliefert.27 Der `Content-Type`-Header der Antwort wird auf `application/json; charset=utf-8` gesetzt. XML wird aufgrund der geringeren Verbreitung und des höheren Verarbeitungsaufwands im Kontext moderner Web-APIs nicht standardmäßig unterstützt.28

Die JSON-Strukturen für Requests und Responses müssen klar definiert sein (siehe OpenAPI-Spezifikation in Kapitel 3.3).

##### 3.1.1.4. HTTP-Statuscodes

Die API verwendet Standard-HTTP-Statuscodes, um das Ergebnis einer Anfrage anzuzeigen.27 Dies ermöglicht es Clients, standardisiert auf verschiedene Situationen zu reagieren. Wichtige Statuscodes sind:

- **2xx (Erfolg):**
    - `200 OK`: Standardantwort für erfolgreiche GET-, PUT-, PATCH- oder DELETE-Anfragen.
    - `201 Created`: Antwort für erfolgreiche POST-Anfragen, die eine neue Ressource erstellt haben. Die Antwort sollte einen `Location`-Header mit der URI der neu erstellten Ressource enthalten.
    - `204 No Content`: Antwort für erfolgreiche Anfragen, die keinen Antwortkörper zurückgeben (z.B. erfolgreiche DELETE-Anfrage).
- **3xx (Umleitung):**
    - `304 Not Modified`: Wird verwendet, um Caching-Mechanismen zu unterstützen.
- **4xx (Client-Fehler):**
    - `400 Bad Request`: Die Anfrage war fehlerhaft oder konnte nicht verarbeitet werden (z.B. ungültiges JSON, fehlende Pflichtfelder, Validierungsfehler). Die Antwort sollte eine detailliertere Fehlermeldung im JSON-Format enthalten.28
    - `401 Unauthorized`: Authentifizierung ist fehlgeschlagen oder erforderlich, aber nicht vorhanden.28
    - `403 Forbidden`: Der authentifizierte Benutzer hat keine Berechtigung, auf die angeforderte Ressource oder Operation zuzugreifen.28
    - `404 Not Found`: Die angeforderte Ressource existiert nicht.28
    - `405 Method Not Allowed`: Die verwendete HTTP-Methode ist für die angeforderte Ressource nicht zulässig.
    - `409 Conflict`: Die Anfrage konnte aufgrund eines Konflikts mit dem aktuellen Zustand der Ressource nicht abgeschlossen werden (z.B. Versuch, eine eindeutige Ressource zu erstellen, die bereits existiert).
    - `429 Too Many Requests`: Der Client hat zu viele Anfragen in einem bestimmten Zeitraum gesendet (Ratenbegrenzung).
- **5xx (Server-Fehler):**
    - `500 Internal Server Error`: Ein unerwarteter Fehler ist auf dem Server aufgetreten. Es sollten keine sensiblen Fehlerdetails an den Client weitergegeben werden.28
    - `503 Service Unavailable`: Der Server ist temporär nicht verfügbar (z.B. wegen Überlastung oder Wartung).

Eine konsistente Verwendung von HTTP-Statuscodes ist entscheidend für die Interoperabilität und Robustheit der API-Clients.

##### 3.1.1.5. Endpunktspezifikationen (Beispiele)

Die Endpunkte der API folgen den REST-Prinzipien, wobei Pfade Substantive im Plural verwenden, um Sammlungen von Ressourcen darzustellen, und HTTP-Methoden die auszuführenden Aktionen definieren.27

**Beispiel: Ressourcenverwaltung für "Produkte"**

- **`GET /core/v1/produkte`**
    
    - **Beschreibung:** Ruft eine Liste aller Produkte ab.
    - **Parameter (Query):**
        - `limit` (optional, integer): Maximale Anzahl der zurückzugebenden Produkte (für Paginierung).
        - `offset` (optional, integer): Anzahl der zu überspringenden Produkte (für Paginierung).
        - `sortBy` (optional, string): Feld, nach dem sortiert werden soll (z.B. `name`, `preis`).
        - `sortOrder` (optional, enum: `asc`, `desc`): Sortierreihenfolge.
        - Filterparameter (z.B. `kategorie`, `minPreis`).
    - **Erfolgsantwort (`200 OK`):** JSON-Array von Produktobjekten.
    - **Fehlerantworten:** `401`, `403`.
- **`POST /core/v1/produkte`**
    
    - **Beschreibung:** Erstellt ein neues Produkt.
    - **Request Body (JSON):** Produktobjekt mit den erforderlichen Attributen (z.B. `name`, `beschreibung`, `preis`).
    - **Erfolgsantwort (`201 Created`):** JSON-Objekt des neu erstellten Produkts. `Location`-Header mit URI des neuen Produkts.
    - **Fehlerantworten:** `400` (Validierungsfehler), `401`, `403`, `409` (falls Produkt mit gleichem eindeutigen Bezeichner bereits existiert).
- **`GET /core/v1/produkte/{produktId}`**
    
    - **Beschreibung:** Ruft ein spezifisches Produkt anhand seiner ID ab.
    - **Parameter (Path):** `produktId` (string/integer, je nach ID-Format).
    - **Erfolgsantwort (`200 OK`):** JSON-Objekt des Produkts.
    - **Fehlerantworten:** `401`, `403`, `404` (Produkt nicht gefunden).
- **`PUT /core/v1/produkte/{produktId}`**
    
    - **Beschreibung:** Aktualisiert ein bestehendes Produkt vollständig. Alle Felder des Produkts müssen im Request Body mitgesendet werden.
    - **Parameter (Path):** `produktId`.
    - **Request Body (JSON):** Vollständiges Produktobjekt mit den aktualisierten Werten.
    - **Erfolgsantwort (`200 OK`):** JSON-Objekt des aktualisierten Produkts.
    - **Fehlerantworten:** `400` (Validierungsfehler), `401`, `403`, `404`.
- **`PATCH /core/v1/produkte/{produktId}`**
    
    - **Beschreibung:** Aktualisiert ein bestehendes Produkt partiell. Nur die zu ändernden Felder müssen im Request Body mitgesendet werden.
    - **Parameter (Path):** `produktId`.
    - **Request Body (JSON):** Produktobjekt mit den zu aktualisierenden Feldern.
    - **Erfolgsantwort (`200 OK`):** JSON-Objekt des aktualisierten Produkts.
    - **Fehlerantworten:** `400` (Validierungsfehler), `401`, `403`, `404`.
- **`DELETE /core/v1/produkte/{produktId}`**
    
    - **Beschreibung:** Löscht ein spezifisches Produkt.
    - **Parameter (Path):** `produktId`.
    - **Erfolgsantwort (`204 No Content`):** Kein Antwortkörper.
    - **Fehlerantworten:** `401`, `403`, `404`.

Diese Struktur wird für alle Ressourcen der Coreschicht analog angewendet. Die genauen Definitionen aller Endpunkte, ihrer Parameter, Request- und Response-Schemata werden in der OpenAPI-Spezifikation (Kapitel 3.3) dokumentiert. Die Implementierung von Filterung, Sortierung und Paginierung ist entscheidend für die Benutzerfreundlichkeit und Performance bei großen Datenmengen.26

##### 3.1.1.6. Versionierung

API-Versionierung ist notwendig, um Änderungen an der API vorzunehmen, ohne bestehende Clients zu beeinträchtigen. Die Hauptversion der API wird im URI-Pfad angegeben (z.B. `/v1/`, `/v2/`).30 Kleinere, abwärtskompatible Änderungen (Minor- und Patch-Versionen) erfordern keine neue URI-Version, sollten aber in der API-Dokumentation vermerkt werden.30 Breaking Changes führen immer zu einer neuen Hauptversion im URI.

### 3.2. Externe Schnittstellen (falls zutreffend)

Falls die Coreschicht mit externen Diensten oder Systemen von Drittanbietern kommunizieren muss (z.B. Zahlungsanbieter, externe Datenquellen, Benachrichtigungsdienste), werden diese Schnittstellen hier spezifiziert. Für jede externe Schnittstelle sind folgende Informationen zu dokumentieren:

- **Name des externen Dienstes/Systems.**
- **Zweck der Integration.**
- **Kommunikationsprotokoll** (z.B. REST, SOAP, gRPC).
- **Authentifizierungsmethode** (z.B. API-Key, OAuth 2.0).
- **Datenformate.**
- **Wichtige Endpunkte/Funktionen, die genutzt werden.**
- **Fehlerbehandlung und Retry-Mechanismen.**
- **Performance- und Zuverlässigkeitserwartungen.**

Die Spezifikation externer Schnittstellen ist oft von der Dokumentation des Drittanbieters abhängig.

### 3.3. API-Dokumentation (OpenAPI/Swagger)

Eine umfassende und aktuelle API-Dokumentation ist für Entwickler, die die API nutzen, unerlässlich.25 Die API der Coreschicht wird gemäß der **OpenAPI Specification (OAS)**, Version 3.x (früher bekannt als Swagger) 35, dokumentiert. Die OpenAPI-Spezifikation ist eine standardisierte, sprachunabhängige Beschreibung von REST-APIs, die sowohl von Menschen als auch von Maschinen gelesen werden kann.35

Die OpenAPI-Definitionsdatei (üblicherweise im YAML- oder JSON-Format) wird folgende Informationen für jeden Endpunkt enthalten 37:

- **Pfade und Operationen:** Alle verfügbaren Endpunkte und die unterstützten HTTP-Methoden (GET, POST, PUT, PATCH, DELETE etc.).
- **Parameter:** Definition von Pfad-, Query-, Header- und Cookie-Parametern, einschließlich ihrer Namen, Datentypen, ob sie erforderlich sind und Beschreibungen.
- **Request Bodies:** Beschreibung der erwarteten Request-Payloads, einschließlich Medientypen (z.B. `application/json`) und Schemadefinitionen für die Datenstrukturen.
- **Responses:** Beschreibung möglicher HTTP-Statuscodes für jede Operation und die zugehörigen Response Bodies, einschließlich Medientypen und Schemadefinitionen.
- **Schemadefinitionen (Components):** Wiederverwendbare Definitionen für Datenmodelle (z.B. `Produkt`, `Benutzer`), die in Request- und Response-Bodies verwendet werden. Dies fördert Konsistenz und reduziert Redundanz.
- **Sicherheitsdefinitionen (Security Schemes):** Beschreibung der verwendeten Authentifizierungs- und Autorisierungsmechanismen (z.B. OAuth 2.0, API-Key).
- **Metainformationen:** Titel, Version, Beschreibung der API, Kontaktinformationen, Lizenzinformationen.

Werkzeuge wie Swagger Editor oder Swagger UI können verwendet werden, um die OpenAPI-Spezifikation zu erstellen, zu validieren und interaktiv darzustellen.35 Swagger UI ermöglicht es Entwicklern, die API direkt im Browser zu testen. Postman-Templates können ebenfalls als Grundlage für die Dokumentation dienen.25 Die OpenAPI-Datei sollte versioniert und zusammen mit dem Quellcode der Coreschicht verwaltet werden.

### 3.4. Best Practices für API-Design

Bei der Gestaltung der APIs der Coreschicht werden folgende Best Practices berücksichtigt, um eine hohe Qualität, Benutzerfreundlichkeit und Wartbarkeit sicherzustellen:

- **Konsistenz:** Einheitliche Namenskonventionen für Pfade, Parameter und Felder. Konsistente Verwendung von HTTP-Methoden und Statuscodes.27
- **Ressourcenorientierung:** Design der API um Ressourcen herum, nicht um Aktionen (Verben in Pfaden vermeiden, außer für spezifische nicht-CRUD-Operationen).27
- **Korrekte Verwendung von HTTP-Methoden:** GET für Abruf, POST für Erstellung, PUT für vollständige Aktualisierung, PATCH für partielle Aktualisierung, DELETE für Löschung.28
- **Statuslose Kommunikation:** Jede Anfrage vom Client an den Server muss alle Informationen enthalten, die zur Bearbeitung der Anfrage erforderlich sind. Der Server speichert keinen Client-Kontext zwischen Anfragen (fundamental für REST).32
- **Sicherheit:** Implementierung robuster Authentifizierung und Autorisierung, Verwendung von HTTPS für die gesamte Kommunikation, Validierung aller Eingaben.27 Sensible Daten wie Passwörter oder API-Keys dürfen niemals in URLs erscheinen.27
- **Paginierung, Filterung und Sortierung:** Für Endpunkte, die Listen von Ressourcen zurückgeben, müssen Mechanismen zur Paginierung (z.B. `limit`/`offset` oder cursor-basiert), Filterung und Sortierung bereitgestellt werden, um die Performance zu optimieren und die Datenmenge zu kontrollieren.26
- **Fehlerbehandlung:** Klare und informative Fehlermeldungen im JSON-Format zurückgeben, ohne interne Implementierungsdetails preiszugeben.28
- **Caching-Unterstützung:** Verwendung von HTTP-Headern wie `ETag` und `Last-Modified` zur Unterstützung von Caching auf Client- oder Proxy-Ebene, um die Serverlast zu reduzieren und die Antwortzeiten zu verbessern.27
- **Ratenbegrenzung (Rate Limiting):** Schutz der API vor Missbrauch und Überlastung durch Implementierung von Ratenbegrenzungen pro Client/Benutzer.26
- **Dokumentation:** Umfassende und aktuelle Dokumentation unter Verwendung von Standards wie OpenAPI.25

Die Beachtung dieser Prinzipien führt zu APIs, die nicht nur funktional, sondern auch sicher, performant und einfach zu integrieren sind. Die API-Spezifikation sollte als "Vertrag" zwischen dem API-Anbieter (Coreschicht) und den API-Konsumenten betrachtet werden.26

## 4. UI/UX-Design-Spezifikationen (für Administrations- und Konfigurationsoberflächen)

Obwohl die Coreschicht primär Backend-Funktionalitäten bereitstellt, können spezifische Benutzeroberflächen (UIs) für Administrations-, Konfigurations- oder Überwachungsaufgaben erforderlich sein. Diese UIs interagieren direkt mit den APIs der Coreschicht. Dieses Kapitel definiert die Spezifikationen für das User Interface (UI) und die User Experience (UX) dieser speziellen Oberflächen. Der Designprozess folgt typischerweise einer Progression von Wireframes über Mockups zu klickbaren Prototypen.39

### 4.1. Wireframes

Wireframes sind grundlegende, schematische Darstellungen der Bildschirmlayouts und der Informationsarchitektur. Sie fokussieren auf Struktur, Inhaltshierarchie und grundlegende Funktionalität, ohne visuelle Designelemente wie Farben oder detaillierte Grafiken.39

#### 4.1.1. Zweck und Detailgrad

Der Zweck von Wireframes in diesem Kontext ist:

- Definition der grundlegenden Struktur und des Layouts der Administrationsseiten.
- Festlegung der Anordnung von UI-Elementen (z.B. Navigationsmenüs, Tabellen, Formulare, Schaltflächen).
- Visualisierung der Benutzerflüsse für typische Administrationsaufgaben.
- Frühzeitige Abstimmung über Funktionalität und Informationsarchitektur mit den Stakeholdern.

Wireframes für die Coreschicht-Administrationsoberflächen werden in der Regel als Low-Fidelity-Darstellungen erstellt, um schnelle Iterationen und Feedbackschleifen zu ermöglichen.39 Sie sollten jedoch genügend Details enthalten, um die Kernfunktionalitäten und die Navigation klar zu vermitteln.

#### 4.1.2. Werkzeuge

Für die Erstellung von Wireframes können verschiedene Werkzeuge eingesetzt werden:

- **Balsamiq:** Bekannt für seinen skizzenhaften Stil, der den Low-Fidelity-Charakter betont und Diskussionen auf die Struktur lenkt.42
- **Figma:** Ein kollaboratives All-in-One-Tool, das auch für Wireframing gut geeignet ist und einen nahtlosen Übergang zu Mockups und Prototypen ermöglicht.42
- **Moqups:** Einsteigerfreundliches Web-Tool für Wireframes, Mockups und Diagramme.42
- **ClickUp:** Bietet Whiteboard-Funktionen und Vorlagen für Wireframing im Rahmen einer Projektmanagement-Plattform.46
- **Visily:** Bietet KI-gestützte Wireframing-Funktionen, z.B. Umwandlung von Screenshots in editierbare Wireframes.42
- Auch einfache Werkzeuge wie Stift und Papier oder Whiteboards können für erste Entwürfe verwendet werden.41

Die Wahl des Werkzeugs hängt von den Präferenzen des Teams und den Anforderungen an Kollaboration und Detailgrad ab. Figma wird aufgrund seiner Vielseitigkeit und Kollaborationsmöglichkeiten oft bevorzugt.

#### 4.1.3. Beispiele (Beschreibung der wichtigsten Ansichten)

Die Wireframes werden die wichtigsten Ansichten der Administrationsoberfläche abdecken. Typische Ansichten könnten sein:

- **Dashboard/Übersichtsseite:** Anzeige wichtiger Systemstatistiken, Benachrichtigungen oder schneller Zugriff auf häufig genutzte Funktionen.
- **Benutzerverwaltung:** Liste der Benutzer, Formulare zum Anlegen/Bearbeiten von Benutzern, Zuweisung von Rollen und Berechtigungen.
- **Datenmanagement-Ansichten:** Tabellarische Darstellung von Kerndatenobjekten (z.B. Produkte, Bestellungen), mit Funktionen zum Suchen, Filtern, Erstellen, Bearbeiten und Löschen von Einträgen.
- **Konfigurationsseiten:** Formulare zur Einstellung systemspezifischer Parameter der Coreschicht.
- **Log-Ansicht:** Anzeige von System- oder Audit-Logs mit Filter- und Suchfunktionen.

Jeder Wireframe wird die Platzierung von Navigationselementen, Hauptinhaltsbereichen, Aktionsschaltflächen und wichtigen Datenfeldern skizzieren. Die Verwendung von echtem oder realitätsnahem Beispieltext anstelle von "Lorem Ipsum" wird empfohlen, um die Struktur besser beurteilen zu können.41

#### 4.1.4. Informationsarchitektur und Navigation

Ein wesentlicher Bestandteil der Wireframing-Phase ist die Definition der Informationsarchitektur (IA) und des Navigationskonzepts.48 Dies beinhaltet:

- **Strukturierung der Inhalte:** Logische Gruppierung von Funktionen und Informationen.
- **Navigationsmenüs:** Definition der Hauptnavigation (z.B. Seitenleiste, Top-Menü) und ggf. Unternavigation.
- **Benutzerflüsse:** Darstellung, wie Benutzer typische Aufgaben innerhalb der Administrationsoberfläche erledigen (z.B. Anlegen eines neuen Benutzers, Ändern einer Konfigurationseinstellung).

Die IA sollte intuitiv sein und es Administratoren ermöglichen, gesuchte Informationen und Funktionen schnell zu finden und zu bedienen.49 Die Konsistenz in der Navigation und Struktur über verschiedene Ansichten hinweg ist dabei entscheidend.41

### 4.2. Mockups

Mockups sind detailliertere, oft farbige, aber nicht interaktive Entwürfe der Benutzeroberfläche. Sie bauen auf den Wireframes auf und fügen visuelle Designelemente hinzu, um ein realistischeres Bild des Endprodukts zu vermitteln.39

#### 4.2.1. Visuelles Design und Detailtiefe

Mockups für die Administrationsoberfläche der Coreschicht werden folgende Aspekte des visuellen Designs konkretisieren:

- **Farbpalette:** Anwendung der im Styleguide (siehe Kapitel 4.4) definierten Farben.
- **Typografie:** Verwendung der festgelegten Schriftarten, -größen und -schnitte.
- **Ikonografie:** Einsatz spezifischer Icons für Aktionen und Navigationselemente.
- **Layout und Abstände:** Präzisere Definition von Rastern, Abständen und Ausrichtung der Elemente.
- **Visuelle Darstellung von UI-Komponenten:** Detaillierte Gestaltung von Schaltflächen, Formularfeldern, Tabellen, Benachrichtigungen etc.

Mockups sind statisch und dienen primär der Abstimmung des visuellen Erscheinungsbildes.39 Sie sollten High-Fidelity sein, um eine genaue Vorstellung vom Look-and-Feel zu geben.

#### 4.2.2. Werkzeuge

Viele der für Wireframing genannten Werkzeuge eignen sich auch für die Erstellung von Mockups, insbesondere solche, die einen fließenden Übergang von Low- zu High-Fidelity ermöglichen:

- **Figma:** Sehr stark für detailliertes UI-Design und Mockup-Erstellung.42
- **Sketch:** Ein weiteres professionelles UI-Design-Tool, primär für macOS.43
- **Adobe XD:** Teil der Adobe Creative Cloud, bietet umfangreiche Design- und Prototyping-Funktionen.50
- **Visily:** Kann auch für High-Fidelity Mockups verwendet werden, insbesondere durch seine KI-Funktionen und Vorlagen.47
- **Moqups:** Unterstützt ebenfalls den Übergang von Wireframes zu detaillierteren Mockups.45

Die Wahl des Werkzeugs wird oft durch die bereits im Team etablierten Tools und die Notwendigkeit der Kollaboration bestimmt.

#### 4.2.3. Beispiele (Beschreibung der wichtigsten Ansichten)

Für alle in Kapitel 4.1.3 beschriebenen Wireframe-Ansichten werden entsprechende Mockups erstellt. Diese zeigen die finale visuelle Gestaltung:

- **Dashboard:** Mit realitätsnahen Diagrammen, Farbschemata und Icons.
- **Benutzerverwaltung:** Formulare und Tabellen im finalen Design, inklusive Statusanzeigen (z.B. aktiv/inaktiv).
- **Datenmanagement-Ansichten:** Tabellen mit korrekter Typografie, Farbcodierung für bestimmte Zustände, gestaltete Aktionsschaltflächen.
- **Konfigurationsseiten:** Klar strukturierte Formulare mit ansprechenden Eingabeelementen.

Die Mockups dienen als Vorlage für die Frontend-Entwicklung der Administrationsoberfläche.

### 4.3. Prototypen (klickbar)

Klickbare Prototypen sind interaktive Modelle der Benutzeroberfläche, die den Benutzerfluss simulieren und es ermöglichen, die User Experience (UX) vor der eigentlichen Implementierung zu testen.39

#### 4.3.1. Interaktivität und Benutzerflüsse

Die Prototypen für die Administrationsoberfläche werden folgende Interaktionen ermöglichen:

- **Navigation:** Klickbare Menüpunkte, die zu den entsprechenden Seiten führen.
- **Formularinteraktionen:** Simulation von Eingaben in Formularfelder (ohne tatsächliche Datenverarbeitung), Auswahl aus Dropdowns.
- **Schaltflächen-Interaktionen:** Klickbare Schaltflächen, die zu anderen Ansichten navigieren oder Zustandsänderungen simulieren (z.B. Anzeige einer Erfolgs- oder Fehlermeldung).
- **Tabelleninteraktionen:** Simulation von Sortier- oder Filterfunktionen.

Ziel ist es, die wichtigsten Benutzerflüsse (Use Cases) durch die Administrationsoberfläche erlebbar zu machen.39 Die Prototypen sind in der Regel High-Fidelity in Bezug auf das visuelle Design (basierend auf den Mockups), aber die Interaktionen sind simuliert und greifen nicht auf das tatsächliche Backend zu.

#### 4.3.2. Werkzeuge

Viele moderne UI/UX-Designwerkzeuge bieten integrierte Prototyping-Funktionen:

- **Figma:** Ermöglicht das Verknüpfen von Frames und das Definieren von Übergängen und einfachen Interaktionen, um klickbare Prototypen zu erstellen.43
- **Adobe XD:** Bietet ebenfalls starke Prototyping-Funktionen, einschließlich Auto-Animate für komplexere Übergänge.50
- **Sketch:** In Kombination mit Plugins oder anderen Werkzeugen wie InVision oder Marvel für Prototyping nutzbar.43
- **Justinmind:** Spezialisiert auf interaktive Prototypen, ermöglicht auch komplexere Logik und Datenmanipulationen in Prototypen.42
- **ProtoPie:** Ein leistungsstarkes Werkzeug für High-Fidelity-Prototyping mit Fokus auf komplexe Interaktionen, kann Designs aus Figma oder Sketch importieren.43
- **Marvel:** Ein weiteres Tool für schnelles Prototyping und Testing.43
- **UXPin:** Ermöglicht die Erstellung von Prototypen, die sehr nah an das Endprodukt herankommen, inklusive Code-basierter Designelemente.42

Figma und Adobe XD sind oft ausreichend für die meisten klickbaren Prototypen im Administrationsbereich. Spezialisierte Werkzeuge wie ProtoPie oder Justinmind kommen bei Bedarf für komplexere Interaktionssimulationen in Frage.

#### 4.3.3. Zu testende Szenarien

Die klickbaren Prototypen werden verwendet, um spezifische Benutzerszenarien zu testen und Feedback zur Usability zu sammeln. Beispiele für Testszenarien:

- **Anlegen eines neuen Benutzers:** Kann ein Administrator den Prozess von Anfang bis Ende intuitiv durchlaufen?
- **Ändern einer wichtigen Systemeinstellung:** Ist der Pfad zur Einstellung klar? Sind die Optionen verständlich?
- **Suchen und Filtern von Daten in einer Tabelle:** Funktionieren die Interaktionen wie erwartet und sind sie effizient?
- **Verständlichkeit von Fehlermeldungen und Hinweisen (simuliert).**

Die Ergebnisse dieser Tests fließen direkt in die Optimierung des UI/UX-Designs ein, bevor Entwicklungsressourcen gebunden werden.39

### 4.4. Styleguide

Der Styleguide definiert die visuellen und gestalterischen Grundlagen für die Administrationsoberfläche. Er stellt Konsistenz über alle Ansichten und Komponenten hinweg sicher und dient als Referenz für Designer und Entwickler.52 Er ist ein zentrales Element eines umfassenderen Design Systems.54

#### 4.4.1. Farbpalette

- **Primärfarben:** Die Hauptfarben, die das Branding der Administrationsoberfläche prägen (z.B. für Hintergründe, Navigationselemente).
- **Sekundärfarben:** Akzentfarben zur Hervorhebung von aktiven Elementen, Links oder wichtigen Informationen.
- **Statusfarben:** Farben für Erfolgsmeldungen (grün), Warnungen (gelb/orange), Fehlermeldungen (rot) und Informationshinweise (blau).
- **Neutrale Farben:** Graustufen für Text, Hintergründe, Trennlinien und deaktivierte Elemente.

Für jede Farbe werden die exakten Farbwerte (z.B. HEX, RGB, HSL) spezifiziert.52

#### 4.4.2. Typografie

- **Schriftfamilien:** Definition der primären Schriftart für Überschriften und der sekundären Schriftart für Fließtext und UI-Elemente.
- **Schriftschnitte und -größen:** Festlegung verschiedener Schriftschnitte (z.B. Regular, Bold, Italic) und einer Hierarchie von Schriftgrößen für unterschiedliche Textelemente (z.B. H1, H2, H3, Paragraph, Label).
- **Zeilenhöhe und Zeichenabstand:** Vorgaben für optimale Lesbarkeit.
- **Textfarben:** Definition der Standardtextfarbe und Farben für Links oder hervorgehobenen Text in Abstimmung mit der Farbpalette.

Die typografischen Regeln gewährleisten ein einheitliches und gut lesbares Erscheinungsbild.52

#### 4.4.3. Ikonografie

- **Icon-Set:** Auswahl oder Erstellung eines konsistenten Icon-Sets (z.B. Material Design Icons, Font Awesome oder ein benutzerdefiniertes Set).
- **Stil:** Definition des visuellen Stils der Icons (z.B. outlined, filled, two-tone).
- **Größen:** Standardgrößen für Icons in verschiedenen Kontexten (z.B. in Schaltflächen, Menüs, Tabellen).
- **Verwendungsrichtlinien:** Beispiele für die korrekte Anwendung von Icons.52

Icons tragen maßgeblich zur intuitiven Bedienbarkeit bei.

#### 4.4.4. Abstände und Layout-Raster

- **Grid-System:** Definition eines Basisrasters (z.B. 8-Punkt-Grid), das für die Ausrichtung und Platzierung aller UI-Elemente verwendet wird. Dies sorgt für visuelle Harmonie und Konsistenz.
- **Abstandsregeln (Spacing):** Festlegung von Standardabständen zwischen Elementen (z.B. Margin, Padding) basierend auf dem Grid-System.
- **Responsive Design Vorgaben:** Wie sich das Layout und die Abstände auf verschiedenen Bildschirmgrößen anpassen (falls die Administrationsoberfläche responsiv sein soll).53

Ein durchdachtes Raster- und Abstandssystem ist fundamental für ein professionelles und aufgeräumtes UI-Design.52

#### 4.4.5. UI-Komponentenbibliothek

Die UI-Komponentenbibliothek ist eine Sammlung wiederverwendbarer UI-Elemente, die in der Administrationsoberfläche zum Einsatz kommen. Sie ist ein praktischer Teil des Styleguides und oft der Kern eines Design Systems.54 Für jede Komponente werden definiert:

- **Name der Komponente** (z.B. Button, Input Field, Dropdown, Table, Modal, Notification).
- **Visuelle Spezifikation:** Aussehen basierend auf Farben, Typografie, Icons und Abständen des Styleguides.
- **Zustände:** Definition verschiedener Zustände der Komponente (z.B. für einen Button: default, hover, active, disabled, loading).52
- **Verhaltensregeln:** Wie die Komponente auf Benutzerinteraktionen reagiert.
- **Anwendungsbeispiele ("Do's and Don'ts"):** Richtlinien für den korrekten Einsatz der Komponente.52

Beispiele für UI-Komponenten:

- **Schaltflächen (Buttons):** Primär-, Sekundär-, Tertiär-Buttons; Buttons mit Icons.
- **Formularelemente:** Textfelder, Textareas, Checkboxen, Radiobuttons, Select-Dropdowns, Datepicker.
- **Tabellen:** Darstellung, Sortier- und Filterindikatoren, Paginierungselemente.
- **Navigationselemente:** Menüs, Tabs, Breadcrumbs.
- **Feedback-Elemente:** Modale Dialoge, Popovers, Toasts/Notifications, Ladeindikatoren.

Diese Bibliothek stellt sicher, dass Entwickler auf standardisierte und bereits gestaltete Elemente zurückgreifen können, was die Entwicklungszeit verkürzt und die Konsistenz der UI erhöht.49 Die UI-Komponenten sollten so gestaltet sein, dass sie die Prinzipien einer guten Admin-UI erfüllen: Benutzerfreundlichkeit, klare Layouts und Anpassbarkeit.49

## 5. Sicherheitskonzept

Das Sicherheitskonzept beschreibt die geplanten Maßnahmen zur Gewährleistung der Vertraulichkeit, Integrität und Verfügbarkeit der Daten und Funktionen der Coreschicht. Sicherheit ist kein nachträglicher Gedanke, sondern ein integraler Bestandteil des gesamten Entwicklungszyklus ("Secure by Design" und "Secure Software Development Lifecycle" - SDLC).57 Die hier definierten Maßnahmen müssen in allen Phasen des Datenbankdesigns, der API-Entwicklung und der UI/UX-Gestaltung für Administrationstools berücksichtigt werden. Dieses Konzept orientiert sich an etablierten Sicherheitspraktiken und adressiert gängige Bedrohungen.

### 5.1. Grundlagen und Prinzipien

#### 5.1.1. Layered Security (Defense in Depth)

Das Sicherheitskonzept basiert auf dem Prinzip der "Layered Security" oder "Defense in Depth".58 Dies bedeutet, dass mehrere Sicherheitsebenen implementiert werden, sodass ein Angreifer, der eine einzelne Schutzmaßnahme überwindet, immer noch auf weitere Barrieren trifft. Diese Ebenen umfassen typischerweise Netzwerk-, Anwendungs-, Daten- und physische Sicherheit.57 Für die Coreschicht sind insbesondere die Anwendungs- und Datensicherheit relevant, die durch Maßnahmen auf Netzwerkebene (z.B. Firewalls, die hier nicht im Detail spezifiziert werden, aber vorausgesetzt werden) ergänzt werden.

#### 5.1.2. Prinzip der geringsten Rechte (Principle of Least Privilege)

Benutzern und Systemkomponenten werden nur die minimal notwendigen Berechtigungen erteilt, die sie zur Erfüllung ihrer Aufgaben benötigen.31 Dies minimiert den potenziellen Schaden im Falle einer Kompromittierung eines Kontos oder einer Komponente. Dieses Prinzip wird durch das Autorisierungsmodell (siehe 5.2.2) konsequent umgesetzt.

### 5.2. Authentifizierung und Autorisierung

Eine robuste Authentifizierung und Autorisierung ist fundamental, um sicherzustellen, dass nur legitime Benutzer und Systeme auf die Coreschicht zugreifen und nur die ihnen erlaubten Aktionen durchführen können.32

#### 5.2.1. Authentifizierungsmechanismen

Die Coreschicht muss starke Authentifizierungsmechanismen für alle Zugriffe implementieren.

- **Für Benutzer (z.B. über Administrations-UI oder clientseitige Anwendungen):**
    
    - **OAuth 2.0 mit OpenID Connect (OIDC):** Als Standard für die delegierte Authentifizierung.31 Dies ermöglicht es Clients (z.B. Web-Frontends, mobile Apps), Benutzer sicher zu authentifizieren, ohne direkten Zugriff auf deren Anmeldedaten zu haben.
    - **JSON Web Tokens (JWTs):** Nach erfolgreicher Authentifizierung werden JWTs ausgestellt, die als Bearer-Token für nachfolgende API-Anfragen verwendet werden.31 JWTs müssen signiert (z.B. mit RS256 oder ES256) und validiert werden (Signatur, Ablaufdatum, Aussteller, Zielgruppe, `kid`-Header gegen JWK).33
    - **Multi-Faktor-Authentifizierung (MFA):** Für administrative Zugriffe und sensible Operationen ist MFA zwingend erforderlich.60 Dies fügt eine zusätzliche Sicherheitsebene über das reine Passwort hinaus hinzu.
    - **Passwortrichtlinien:** Starke Passwortrichtlinien (Mindestlänge, Komplexität, keine gängigen Passwörter) müssen durchgesetzt werden.60 Passwörter müssen sicher gehasht (z.B. mit Argon2id oder bcrypt) und gesalzen gespeichert werden; niemals in Klartext.33
    - **Konto-Sperrungsrichtlinien:** Nach einer definierten Anzahl fehlgeschlagener Anmeldeversuche wird das Konto temporär gesperrt, um Brute-Force-Angriffe zu erschweren.60
- **Für serverseitige System-zu-System-Kommunikation (interne Dienste):**
    
    - **OAuth 2.0 Client Credentials Flow:** Geeignet für vertrauenswürdige Server-Anwendungen, die im eigenen Namen auf Ressourcen zugreifen.31
    - **API-Keys:** Für einfachere Szenarien können API-Keys verwendet werden. Diese müssen eine hohe Entropie aufweisen, sicher übertragen (z.B. im HTTP-Header `X-API-Key`), serverseitig validiert und regelmäßig rotiert werden.31 API-Keys sollten nicht im Code fest verdrahtet, sondern sicher verwaltet werden (z.B. über Secret-Management-Systeme).

Alle Authentifizierungsdaten (Passwörter, Token, API-Keys) müssen stets über verschlüsselte Verbindungen (HTTPS) übertragen werden.27

#### 5.2.2. Autorisierungsmodell (z.B. RBAC)

Nach erfolgreicher Authentifizierung erfolgt die Autorisierung, um zu bestimmen, welche Aktionen ein authentifizierter Benutzer oder Dienst durchführen darf. Es wird ein **Role-Based Access Control (RBAC)** Modell implementiert.32

- **Rollen:** Definieren Gruppen von Berechtigungen, die typischen Benutzerkategorien oder Systemfunktionen entsprechen (z.B. `Administrator`, `ReadOnlyUser`, `CoreServiceCommunicator`).
- **Berechtigungen:** Spezifische Rechte, die Aktionen auf bestimmten Ressourcen erlauben (z.B. `produkt:lesen`, `produkt:erstellen`, `benutzer:verwalten`).
- **Zuweisung:** Benutzern oder Dienst-Identitäten werden eine oder mehrere Rollen zugewiesen.

Die Autorisierungsentscheidungen werden bei jedem API-Aufruf basierend auf der Rolle/den Berechtigungen des anfragenden Subjekts getroffen. Dies adressiert direkt die OWASP API Security Risiken #1 (Broken Object Level Authorization) und #5 (Broken Function Level Authorization).61

**Tabelle 5.1: Rollen- und Rechteübersicht (Beispiel)**

|   |   |   |   |   |
|---|---|---|---|---|
|**Rolle**|**Beschreibung der Rolle**|**Berechtigungen für Datenzugriff (CRUD - Beispiel: Produkt, Benutzer)**|**Berechtigungen für API-Endpunkte (Beispiel)**|**Zugeordnete Authentifizierungsmethoden**|
|`SystemAdministrator`|Vollständige Kontrolle über das System, Benutzerverwaltung, Konfiguration.|Produkt:CRUDE, Benutzer:CRUDE, Konfiguration:CRUDE|`GET /produkte`, `POST /produkte`, `PUT /produkte/{id}`, `DELETE /produkte/{id}`, `GET /benutzer`, `POST /benutzer`, etc. (alle administrativen Endpunkte)|OAuth 2.0 (mit MFA)|
|`DatenAnalyst`|Kann alle Daten lesen, aber keine Änderungen vornehmen.|Produkt:R, Benutzer:R, Bestellung:R|`GET /produkte`, `GET /produkte/{id}`, `GET /benutzer`, `GET /benutzer/{id}`, `GET /bestellungen`|OAuth 2.0|
|`FrontendService`|Stellt Daten für die Hauptanwendung bereit, kann im Namen von Benutzern Bestellungen erstellen.|Produkt:R, Bestellung:CR (im Kontext des Benutzers), Benutzer:R (eingeschränkt auf eigene Daten des Benutzers)|`GET /produkte`, `GET /produkte/{id}`, `POST /bestellungen` (im Benutzerkontext), `GET /benutzer/me`|OAuth 2.0 (Authorization Code Flow für Benutzer, Client Credentials für eigene Operationen)|
|`InternerBatchService`|Führt Hintergrundaufgaben aus, z.B. Datenaggregation.|Produkt:R, Bestellung:R, AggregierteDaten:CRU|`GET /produkte/all`, `POST /aggregierteDaten`|API-Key oder OAuth 2.0 Client Credentials|

Diese Tabelle ist ein kritisches Werkzeug, um das Prinzip der geringsten Rechte systematisch anzuwenden. Sie muss detailliert für alle relevanten Rollen, Datenobjekte und API-Endpunkte ausgearbeitet werden. Die Granularität der Berechtigungen muss bis auf die Ebene einzelner Objekte und deren Eigenschaften reichen (Object Level und Object Property Level Authorization), um Risiken wie API1:2023 und API3:2023 der OWASP Top 10 zu mitigieren.61

### 5.3. Datensicherheit

Der Schutz der in der Coreschicht gespeicherten und verarbeiteten Daten ist von höchster Bedeutung.

#### 5.3.1. Verschlüsselung von Daten "at rest"

Alle sensiblen Daten, die in der Datenbank oder anderen persistenten Speichern der Coreschicht abgelegt werden, müssen verschlüsselt werden ("encryption at rest").63

- **Algorithmen:** Es sind starke, etablierte symmetrische Verschlüsselungsalgorithmen wie **AES-256 (Advanced Encryption Standard mit 256-Bit Schlüssellänge)** zu verwenden.63
- **Anwendungsbereich:** Dies betrifft insbesondere personenbezogene Daten (PII), Finanzdaten, Authentifizierungsdaten (obwohl Passwörter gehasht und nicht nur verschlüsselt werden) und andere geschäftskritische Informationen.
- **Schlüsselmanagement:** Ein sicheres Schlüsselmanagement ist entscheidend. Die Verschlüsselungsschlüssel müssen sicher generiert, gespeichert, rotiert und verwaltet werden. Hierfür sollte ein dediziertes **Hardware Security Module (HSM)** oder ein **Key Management Service (KMS)** (z.B. AWS KMS, Azure Key Vault, Google Cloud KMS) eingesetzt werden.63 Der Zugriff auf die Schlüssel muss streng kontrolliert und protokolliert werden. Regelmäßige Schlüsselrotation ist ein Muss.
- **Datenbankseitige vs. Applikationsseitige Verschlüsselung:** Je nach Sensitivität und Anforderungen kann die Verschlüsselung auf Datenbankebene (Transparent Data Encryption - TDE) oder auf Applikationsebene (Client-Side Encryption, bevor die Daten in die DB geschrieben werden) erfolgen.64 Applikationsseitige Verschlüsselung bietet oft mehr Kontrolle, erfordert aber sorgfältige Implementierung.

#### 5.3.2. Verschlüsselung von Daten "in transit"

Jegliche Datenübertragung zur und von der Coreschicht sowie zwischen internen Komponenten der Coreschicht (falls diese über ein Netzwerk kommunizieren) muss verschlüsselt werden ("encryption in transit").63

- **Protokoll:** Für die API-Kommunikation (extern und intern) ist ausschließlich **HTTPS (HTTP Secure)** zu verwenden, basierend auf **TLS (Transport Layer Security)** in einer aktuellen Version (mindestens TLS 1.2, bevorzugt TLS 1.3).27
- **Zertifikate:** Es sind gültige digitale Zertifikate von vertrauenswürdigen Zertifizierungsstellen (CAs) zu verwenden.
- **Cipher Suites:** Nur starke und aktuell als sicher geltende Cipher Suites dürfen konfiguriert werden. Veraltete oder schwache Algorithmen (z.B. SSLv3, frühe TLS-Versionen, MD5, SHA1) sind zu deaktivieren.
- **Interne Kommunikation:** Auch die Kommunikation zwischen Microservices oder verschiedenen Instanzen der Coreschicht sollte, wenn sie über ein Netzwerk erfolgt, mittels TLS gesichert werden.

Die konsequente Verschlüsselung von Daten "at rest" und "in transit" schützt vor unbefugtem Zugriff und Datenlecks.

### 5.4. API-Sicherheitsmaßnahmen

Die APIs der Coreschicht sind potenzielle Angriffsvektoren. Daher müssen spezifische Sicherheitsmaßnahmen implementiert werden, die sich insbesondere an den **OWASP API Security Top 10** orientieren.61

- **`API1:2023 Broken Object Level Authorization (BOLA)`:** Strikte Überprüfung bei jedem Zugriff auf ein Objekt (z.B. `GET /produkte/{id}`), ob der authentifizierte Benutzer tatsächlich die Berechtigung hat, auf _dieses spezifische Objekt_ zuzugreifen, nicht nur auf die Objektart allgemein. Dies wird durch das RBAC-Modell und detaillierte Berechtigungsprüfungen in der Geschäftslogik erreicht.61
- **`API2:2023 Broken Authentication`:** Implementierung der robusten Authentifizierungsmechanismen wie in 5.2.1 beschrieben (OAuth 2.0, JWT-Validierung, MFA, sichere Passwort-Policies und -Speicherung).61
- **`API3:2023 Broken Object Property Level Authorization (BOPLA)`:**
    - **Excessive Data Exposure:** API-Antworten dürfen nur die Datenfelder enthalten, die für den jeweiligen Benutzer und Anwendungsfall tatsächlich benötigt und erlaubt sind. Sensible Felder müssen herausgefiltert werden.
    - **Mass Assignment:** Bei Operationen, die Datenobjekte entgegennehmen (z.B. PUT, POST, PATCH), dürfen nur die Felder aktualisiert werden, die vom Benutzer geändert werden dürfen. Eine Whitelist erlaubter Felder ist zu verwenden, um das Überschreiben interner oder schützenswerter Felder zu verhindern.61
- **`API4:2023 Unrestricted Resource Consumption`:** Implementierung von Maßnahmen zur Begrenzung der Ressourcennutzung:
    - **Ratenbegrenzung (Rate Limiting):** Begrenzung der Anzahl der Anfragen, die ein Client innerhalb eines bestimmten Zeitraums stellen kann.26
    - **Quotas:** Begrenzung der Gesamtmenge an Ressourcen (z.B. Speicherplatz, Anzahl Objekte), die ein Benutzer/Tenant nutzen darf.
    - **Größenbeschränkungen:** Validierung und Begrenzung der Größe von Request- und Response-Payloads sowie hochgeladenen Dateien.61
    - **Timeout-Konfigurationen:** Angemessene Timeouts für Anfragen.
- **`API5:2023 Broken Function Level Authorization (BFLA)`:** Strikte Trennung der Berechtigungen für administrative Funktionen (z.B. Benutzerverwaltung, Systemkonfiguration) von regulären Benutzerfunktionen. Administrative Endpunkte müssen besonders geschützt und nur für autorisierte Rollen zugänglich sein.61
- **`API6:2023 Unrestricted Access to Sensitive Business Flows`:** Identifizierung und besondere Absicherung von Geschäftsabläufen, die über APIs ausgelöst werden und ein hohes Missbrauchspotenzial haben (z.B. Massenbestellungen, Kontoerstellungen). Dies kann zusätzliche Validierungen, Überwachung oder menschliche Interaktion erfordern.61
- **`API7:2023 Server-Side Request Forgery (SSRF)`:** Wenn die API serverseitig Anfragen an andere URLs stellt (basierend auf Benutzereingaben), müssen diese URLs rigoros validiert und auf eine Whitelist erlaubter Ziele beschränkt werden, um zu verhindern, dass Angreifer interne Systeme scannen oder angreifen können.61
- **Weitere OWASP-Punkte:**
    - **`API8:2023 Security Misconfiguration`:** Sorgfältiges Konfigurationsmanagement, Deaktivierung unnötiger Features, regelmäßige Sicherheitsüberprüfungen der Konfigurationen.62
    - **`API9:2023 Improper Inventory Management`:** Führen eines aktuellen Inventars aller API-Endpunkte, Versionen und deren Sicherheitsstatus. "Shadow APIs" oder veraltete, ungesicherte Endpunkte sind zu vermeiden.62
    - **`API10:2023 Unsafe Consumption of APIs`:** Wenn die Coreschicht selbst externe APIs konsumiert, müssen auch hier Sicherheitsaspekte wie Validierung der Antworten, sichere Authentifizierung und Fehlerbehandlung beachtet werden.62

Die Implementierung dieser Maßnahmen erfordert eine kontinuierliche Aufmerksamkeit während des gesamten API-Lebenszyklus.

### 5.5. Sichere Eingabevalidierung und -verarbeitung

Alle von externen Quellen (insbesondere API-Requests) stammenden Daten müssen serverseitig in der Coreschicht rigoros validiert werden, bevor sie weiterverarbeitet oder gespeichert werden. Clientseitige Validierung dient lediglich der Verbesserung der User Experience, bietet aber keinen Sicherheitsschutz, da sie leicht umgangen werden kann.67

**Arten der Validierung:**

- **Typprüfung:** Sicherstellen, dass die Daten dem erwarteten Datentyp entsprechen (z.B. String, Integer, Boolean, Array, Objekt).67
- **Formatprüfung:** Überprüfung, ob Daten spezifischen Formaten entsprechen (z.B. E-Mail-Adresse, Datum (ISO 8601), UUID, Telefonnummer).67
- **Längen-/Größenprüfung:** Validierung der Mindest- und Maximallänge von Zeichenketten, der Anzahl von Elementen in Arrays oder der Größe von Dateien.67
- **Bereichsprüfung:** Sicherstellen, dass numerische Werte innerhalb eines erlaubten Bereichs liegen (z.B. `Alter >= 0`, `Preis > 0`).
- **Prüfung auf erlaubte Zeichen/Werte (Whitelisting):** Bevorzugt sollte eine Whitelist von erlaubten Zeichen oder Werten verwendet werden, anstatt eine Blacklist von verbotenen Zeichen zu pflegen. Dies ist sicherer, da es schwieriger ist, alle potenziell schädlichen Eingaben vorherzusehen.67
- **Konsistenzprüfung:** Überprüfung, ob zusammengehörige Daten logisch konsistent sind (z.B. Startdatum vor Enddatum).67

**Schutz vor Injection-Angriffen:**

- **SQL-Injection (und NoSQL-Injection):** Verwendung von Prepared Statements (parametrisierten Abfragen) oder ORM-Frameworks, die dies intern handhaben, ist zwingend erforderlich, um SQL-Injection-Angriffe zu verhindern.65 Benutzereingaben dürfen niemals direkt in SQL-Abfragen konkateniert werden.
- **Command-Injection:** Vermeidung der Ausführung von Betriebssystembefehlen, die direkt oder indirekt aus Benutzereingaben konstruiert werden. Wenn unvermeidbar, müssen Eingaben extrem sorgfältig validiert und saniert werden.
- **Cross-Site Scripting (XSS):** Obwohl XSS primär ein Frontend-Problem ist, kann die Coreschicht dazu beitragen, indem sie Daten, die später im Frontend angezeigt werden, korrekt validiert und ggf. vor der Speicherung saniert oder bei der Ausgabe kontextbezogen kodiert (z.B. HTML-Encoding).

Fehlgeschlagene Validierungen müssen zu einer klaren Fehlermeldung an den Client führen (z.B. HTTP `400 Bad Request`), ohne sensible interne Details preiszugeben.67

### 5.6. Protokollierung (Logging) und Überwachung sicherheitsrelevanter Ereignisse

Eine umfassende Protokollierung aller sicherheitsrelevanten Ereignisse ist notwendig, um Sicherheitsvorfälle erkennen, analysieren und darauf reagieren zu können.60

**Zu protokollierende Ereignisse umfassen mindestens:**

- **Authentifizierungsversuche:** Erfolgreiche und fehlgeschlagene Anmeldungen (Benutzername, Quell-IP, Zeitstempel).60
- **Autorisierungsentscheidungen:** Verweigerte Zugriffsversuche auf Ressourcen oder Funktionen (Benutzerkennung, angeforderte Ressource/Funktion, Zeitstempel).
- **Wichtige Konfigurationsänderungen:** Änderungen an Sicherheitseinstellungen, Benutzerrollen oder Berechtigungen (wer hat was wann geändert).
- **API-Anfragen mit Fehlern:** Insbesondere solche, die auf potenzielle Angriffe hindeuten (z.B. wiederholte `401`/`403`-Fehler, Validierungsfehler).
- **Fehler bei der kryptographischen Schlüsselverwaltung.**
- **Erkannte Angriffsversuche oder Anomalien.**

**Anforderungen an die Protokolle:**

- **Ausreichender Detaillierungsgrad:** Logs müssen genügend Informationen enthalten, um den Kontext eines Ereignisses zu verstehen (z.B. Zeitstempel mit Zeitzone, Quell-IP-Adresse, betroffene Benutzerkennung, Ereignistyp, Ergebnis, betroffene Ressource).
- **Integrität und Schutz:** Log-Daten müssen vor unbefugtem Zugriff und Manipulation geschützt werden. Sie sollten idealerweise an ein zentrales, gesichertes Log-Management-System (z.B. SIEM - Security Information and Event Management) gesendet werden.
- **Regelmäßige Auswertung:** Logs müssen regelmäßig (automatisiert und manuell) auf verdächtige Aktivitäten und Muster überwacht werden.60
- **Aufbewahrungsfristen:** Definition von Aufbewahrungsfristen für Log-Daten gemäß rechtlicher und betrieblicher Anforderungen.

Die Protokollierung unterstützt nicht nur die Reaktion auf Vorfälle, sondern auch proaktive Sicherheitsanalysen und die Einhaltung von Compliance-Vorgaben. Die Dokumentation der Logging-Mechanismen und der Zugriff auf Logs ist ebenfalls Teil eines umfassenden Sicherheitsansatzes.57

## 6. Schlussfolgerungen

Dieses Pflichtenheft legt die detaillierten Spezifikationen für die Entwicklung der Coreschicht fest und dient als zentrale Referenz für alle Projektbeteiligten. Es umfasst die genauen Anforderungen an das Datenbankdesign, die API-Schnittstellen, die UI/UX-Gestaltung für administrative Zwecke sowie ein umfassendes Sicherheitskonzept.

Die **Datenbankdesign-Spezifikation** mit einem detaillierten Data Dictionary und konzeptionellen ER-Diagrammen bildet die Grundlage für eine robuste und skalierbare Datenhaltung. Die konsequente Anwendung von Namenskonventionen, die sorgfältige Auswahl von Datentypen und die klare Definition von Beziehungen und Constraints sind hierbei unerlässlich.

Die **Schnittstellendesign-Spezifikation** definiert primär eine RESTful API unter Verwendung von JSON als Datenformat und OAuth 2.0 sowie API-Keys für die Authentifizierung. Die Dokumentation mittels OpenAPI (Swagger) gewährleistet eine klare und maschinenlesbare Beschreibung aller Endpunkte und Datenstrukturen, was die Integration und Nutzung der API erleichtert. Die Einhaltung von Best Practices im API-Design ist entscheidend für die Erstellung einer sicheren, performanten und benutzerfreundlichen Schnittstelle.

Für eventuell notwendige **Administrations- und Konfigurationsoberflächen** werden UI/UX-Spezifikationen bereitgestellt, die den Prozess von Wireframes über Mockups bis hin zu klickbaren Prototypen beschreiben. Ein detaillierter Styleguide inklusive einer UI-Komponentenbibliothek stellt die visuelle Konsistenz und eine effiziente Entwicklung dieser Oberflächen sicher.

Das **Sicherheitskonzept** ist als integraler Bestandteil des gesamten Pflichtenhefts zu verstehen. Es basiert auf den Prinzipien der Layered Security und der geringsten Rechte. Starke Authentifizierungs- und Autorisierungsmechanismen (RBAC), die Verschlüsselung von Daten "at rest" (AES-256) und "in transit" (TLS), die Adressierung der OWASP API Security Top 10, rigorose Eingabevalidierung und umfassende Protokollierung sind Kernkomponenten dieses Konzepts. Die Sicherheit der Coreschicht muss in jeder Phase des Designs und der Entwicklung berücksichtigt werden, um ein "Secure by Design"-Produkt zu gewährleisten.

Die erfolgreiche Umsetzung der in diesem Pflichtenheft definierten Spezifikationen erfordert eine enge Zusammenarbeit aller Beteiligten und eine kontinuierliche Überprüfung der Anforderungen im Projektverlauf. Dieses Dokument ist als "lebendes Dokument" zu betrachten, das bei Bedarf und nach formaler Abstimmung angepasst werden kann, um Änderungen und neue Erkenntnisse zu reflektieren.4 Die Einhaltung der hier festgelegten Vorgaben ist entscheidend für die Entwicklung einer qualitativ hochwertigen, sicheren und wartbaren Coreschicht, die den Anforderungen des Gesamtsystems gerecht wird.

## 7. Anhang

### 7.1. Glossar

|   |   |   |
|---|---|---|
|**Begriff**|**Definition**|**Quelle (falls zutreffend)**|
|API|Application Programming Interface (Programmierschnittstelle)|29|
|CRUD|Create, Read, Update, Delete (Grundlegende Datenoperationen)|27|
|ERD|Entity-Relationship-Diagram (Diagramm zur Darstellung von Datenbankstrukturen)|18|
|GUI|Graphical User Interface (Grafische Benutzeroberfläche)|29|
|HSM|Hardware Security Module (Hardwaremodul zur sicheren Schlüsselspeicherung und -verwaltung)|63|
|HTTP|Hypertext Transfer Protocol (Protokoll zur Übertragung von Daten im Web)||
|HTTPS|Hypertext Transfer Protocol Secure (Sichere Variante von HTTP durch TLS/SSL-Verschlüsselung)|27|
|JWT|JSON Web Token (Standard zur Übertragung von Claims zwischen Parteien als JSON-Objekt)|31|
|JSON|JavaScript Object Notation (Leichtgewichtiges Datenaustauschformat)|27|
|KMS|Key Management Service (Dienst zur Verwaltung kryptographischer Schlüssel)|63|
|MFA|Multi-Factor Authentication (Authentifizierungsmethode mit mehreren Faktoren)|60|
|OAuth 2.0|Open Authorization 2.0 (Offenes Protokoll für delegierte Autorisierung)|31|
|OIDC|OpenID Connect (Identitätsschicht aufbauend auf OAuth 2.0)|31|
|OpenAPI|Standardisierte Speifikation zur Beschreibung von REST-APIs (früher Swagger)|35|
|ORM|Object-Relational Mapping (Technik zur Abbildung von Objekten auf relationale Datenbanken)||
|OWASP|Open Web Application Security Project (Non-Profit-Organisation mit Fokus auf Softwaresicherheit)|61|
|PII|Personally Identifiable Information (Personenbezogene Daten)||
|PK|Primary Key (Primärschlüssel in einer Datenbanktabelle)||
|FK|Foreign Key (Fremdschlüssel in einer Datenbanktabelle)||
|RBAC|Role-Based Access Control (Rollenbasiertes Zugriffskontrollmodell)|32|
|REST|Representational State Transfer (Architekturstil für verteilte Hypermedia-Systeme)|27|
|SDLC|Software Development Lifecycle (Softwareentwicklungslebenszyklus)|57|
|SIEM|Security Information and Event Management (System zur Sammlung und Analyse von Sicherheitsinformationen)||
|SQL|Structured Query Language (Standardsprache zur Verwaltung relationaler Datenbanken)||
|SRS|Software Requirements Specification (Software-Anforderungsspezifikation)|1|
|SSRF|Server-Side Request Forgery (Sicherheitslücke, bei der ein Server dazu gebracht wird, Anfragen an beliebige Ziele zu senden)|61|
|Swagger|Werkzeugsatz zur Implementierung der OpenAPI-Spezifikation (siehe OpenAPI)|35|
|TLS|Transport Layer Security (Verschlüsselungsprotokoll zur sicheren Datenübertragung)|31|
|UI|User Interface (Benutzerschnittstelle)|47|
|UML|Unified Modeling Language (Standardisierte Modellierungssprache)|2|
|URI|Uniform Resource Identifier (Eindeutiger Bezeichner für eine Ressource)||
|URL|Uniform Resource Locator (Spezifische Art von URI, die den Ort einer Ressource angibt)||
|UX|User Experience (Benutzererlebnis)|72|
|UUID|Universally Unique Identifier (Eindeutiger 128-Bit-Identifikator)||
|XSS|Cross-Site Scripting (Art von Sicherheitslücke in Webanwendungen)|67|
|YAML|YAML Ain't Markup Language (Menschenlesbares Datenformat, oft für Konfigurationsdateien verwendet)|37|

### 7.2. Referenzierte Dokumente und Standards (erweitert)

- DIN 69901-5: Projektmanagement – Projektmanagementsysteme – Teil 5: Begriffe 1
- VDI Richtlinie 2519 Blatt 1: Vorgehensweise bei der Planung und Ausführung von Automatisierungsprojekten 1
- VDI Richtlinie 3694: Anforderungen an Automatisierungssysteme 1
- IEEE Std 830-1998: IEEE Recommended Practice for Software Requirements Specifications (ggf. ersetzt durch ISO/IEC/IEEE 29148) 8
- ISO/IEC/IEEE 29148:2018: Systems and software engineering — Life cycle processes — Requirements engineering 7
- OpenAPI Specification (Version 3.x): Standard zur Beschreibung von REST-APIs 35
- OWASP API Security Top 10: Regelmäßig aktualisierte Liste der kritischsten Sicherheitsrisiken für APIs 61
- OWASP Secure Coding Practices
- NIST Special Publications (z.B. SP 800-53 für Sicherheitskontrollen, NIST Cybersecurity Framework) 57
- RFCs (Request for Comments) relevant für HTTP, TLS, JWT, OAuth 2.0 (z.B. RFC 2616, RFC 8446, RFC 7519, RFC 6749)
- [Internes Dokument XYZ]: Lastenheft für das Projekt (falls vorhanden und Basis für dieses Pflichtenheft)
- : Übergeordnete Systemarchitektur