# Blaupause, Roadmap, Entwicklungsplan, Implementierungsplan: Definitionen, Unterschiede und Einordnung in den Softwareentwicklungs-Workflow

## 1. Einleitung

Die Softwareentwicklung ist ein komplexes Unterfangen, das von der initialen Idee bis zur Auslieferung und Wartung eines Produkts reicht. Um diese Komplexität zu beherrschen, sind strukturierte Planung und klare Kommunikation unerlässlich.1 Verschiedene Planungs- und Dokumentationsinstrumente kommen dabei zum Einsatz, um unterschiedliche Aspekte des Entwicklungsprozesses zu steuern und zu kommunizieren. Im deutschen Sprachraum, insbesondere im Kontext der Softwareentwicklung, tauchen dabei Begriffe wie 'Blaupause', 'Roadmap', 'Entwicklungsplan' und 'Implementierungsplan' auf. Obwohl sie alle eine Form der Planung darstellen, dienen sie unterschiedlichen Zwecken, richten sich an verschiedene Zielgruppen und weisen variierende Detaillierungsgrade auf.

Die Existenz dieser unterschiedlichen Planungsdokumente spiegelt die vielschichtige Natur der Softwareentwicklung wider. Ein einzelner Plan kann selten alle notwendigen Perspektiven abdecken – von der langfristigen strategischen Ausrichtung über die operative Projektplanung bis hin zur detaillierten technischen Vorbereitung der Inbetriebnahme. Es bedarf verschiedener Abstraktionsebenen und Fokusbereiche (strategisch, taktisch, operativ, technisch), um den gesamten Lebenszyklus effektiv zu managen und die Kommunikation zwischen diversen Stakeholdern, wie Management, Produktteams, Entwicklern und Betriebspersonal, sicherzustellen.8

Dieser Bericht zielt darauf ab, Klarheit über die Bedeutung, den Inhalt und die Abgrenzung dieser vier zentralen Begriffe im Kontext der Softwareentwicklung zu schaffen. Es werden die einzelnen Begriffe definiert, ihre Unterschiede herausgearbeitet und ihre typische Rolle sowie die anderer wichtiger Dokumente (wie Anforderungsspezifikation, Architekturdesign, Testplan und Sprint Backlog) im Rahmen eines allgemeinen Softwareentwicklungs-Workflows (Software Development Lifecycle, SDLC) beleuchtet. Abschließend wird auf die Unterschiede in der Nutzung und Ausprägung dieser Dokumente in klassischen (z.B. Wasserfall) und agilen (z.B. Scrum) Vorgehensmodellen eingegangen.

## 2. Definitionen der Kernbegriffe

Um die Rolle und Abgrenzung der Planungsartefakte zu verstehen, ist eine klare Definition jedes Begriffs notwendig.

### 2.1. Blaupause (Blueprint)

Der Begriff "Blaupause" stammt ursprünglich aus dem technischen Zeichnungswesen, wo er ein mittels Blaudruck (Zyanotypie) erstelltes, detailgetreues Abbild einer technischen Zeichnung bezeichnete. Im Kontext der Softwareentwicklung wird der Begriff jedoch meist **metaphorisch** verwendet und bezeichnet in der Regel **kein formal definiertes, standardisiertes Artefakt** innerhalb gängiger Vorgehensmodelle wie Scrum oder des V-Modells.11

**Zweck und Inhalt:** Eine Blaupause steht in der Softwareentwicklung oft für ein **frühes Konzept**, eine **grundlegende Idee** oder einen **abstrakten Plan** für die Konstruktion eines Systems.11 Sie kann eine **hochrangige Architekturübersicht** 11, eine **Vorlage** oder ein **Muster** für wiederkehrende Strukturen (z.B. eine Standardarchitektur für eine bestimmte Art von Lösung 12 oder sogar die Definition einer Klasse als "Blaupause" für Objekte 17) repräsentieren. Ihr Hauptzweck liegt oft in der **Kommunikation** einer grundlegenden Idee oder Struktur in einer frühen Phase 15, bevor detaillierte Anforderungen oder technische Spezifikationen ausgearbeitet sind. Sie kann auch als **Dokumentation** grundlegender Designentscheidungen dienen.15 Charakteristisch ist oft ein **hoher Abstraktionsgrad** 11 und ein Mangel an Implementierungsdetails, die für die direkte Umsetzung notwendig wären.14

**Zielgruppe:** Die Zielgruppe ist meist intern und technisch orientiert: Softwarearchitekten, leitende Entwickler, eventuell Produktverantwortliche oder frühe Stakeholder, die an der initialen Konzeptfindung beteiligt sind.

Die informelle und metaphorische Natur des Begriffs "Blaupause" spiegelt gut die Unsicherheit und den explorativen Charakter der frühen Phasen der Softwareentwicklung wider. In dieser Phase sind Anforderungen oft noch vage und technische Lösungen werden erst konzipiert. Die Blaupause dient dazu, die grundlegende Absicht und Struktur festzuhalten, bevor formale und detaillierte Artefakte wie ein vollständiges Architekturdesign oder ein detaillierter Entwicklungsplan erstellt werden können.11 Diese Flexibilität erlaubt es, Ideen schnell zu skizzieren, birgt aber auch die Gefahr von Mehrdeutigkeiten, was typisch für frühe Projektstadien ist.

### 2.2. Roadmap (Produkt-Roadmap)

Eine Roadmap, im Softwarekontext meist als Produkt-Roadmap verstanden, ist eine **strategische, oft visuelle Darstellung der geplanten Entwicklung eines Produkts oder einer Initiative über einen längeren Zeitraum**.9 Sie dient als hochrangiger Fahrplan 19 und verbindet die Produktvision 22 und Unternehmensziele mit konkreteren Entwicklungsinitiativen, wobei sie Marktanforderungen, Kundenbedürfnisse und technologische Möglichkeiten berücksichtigt.19

**Zweck:** Der Hauptzweck einer Roadmap liegt in der **strategischen Ausrichtung und Kommunikation**.9 Sie kommuniziert die Vision und die strategische Richtung des Produkts an alle relevanten Stakeholder, schafft ein gemeinsames Verständnis, unterstützt die Priorisierung von Initiativen und dient als Rahmen für die Koordination zwischen verschiedenen Teams (z.B. Entwicklung, Marketing, Vertrieb).18 Sie hilft dabei, Erwartungen zu managen und Kompromissentscheidungen zu treffen.18

**Typischer Inhalt:** Eine Roadmap fokussiert auf das „Was“ und „Warum“ auf strategischer Ebene. Typische Inhalte sind 9:

- Übergeordnete Ziele (Goals) oder strategische Initiativen
- Themen (Themes) oder Epics (größere Funktionsblöcke)
- Wichtige geplante Features oder Produkterweiterungen (auf hohem Niveau)
- Grobe Zeitrahmen (z.B. Quartale, Halbjahre, Jahre) 18
- Wichtige Meilensteine 9
- Manchmal auch Abhängigkeiten oder strategische Risiken 18 Eine Roadmap ist explizit **keine detaillierte Liste von Features** 18 und **kein detaillierter Projektplan**.18

**Zielgruppe:** Die Zielgruppe einer Roadmap ist oft breit gefächert und umfasst interne sowie manchmal externe Stakeholder 9:

- Unternehmensführung/Management
- Produktmanagement
- Entwicklungsteams
- Marketing- und Vertriebsteams
- Kunden und Partner (bei externen Roadmaps) Oft werden unterschiedliche Ansichten oder Versionen einer Roadmap für verschiedene Zielgruppen erstellt, die jeweils einen angepassten Detaillierungsgrad und Fokus aufweisen.9

Roadmaps sind im Kern Kommunikationswerkzeuge, die darauf abzielen, diverse Gruppen auf eine gemeinsame strategische Ausrichtung einzuschwören.9 Sie schlagen die Brücke zwischen der übergeordneten Strategie und der konkreten Umsetzung 19, indem sie das 'Warum' (strategische Ziele) und das grobe 'Was' (Initiativen, Themen) kommunizieren, anstatt sich im detaillierten 'Wie' zu verlieren. Die Existenz verschiedener Roadmap-Typen für spezifische Zielgruppen 9 unterstreicht diese primäre Funktion. Ihr Wert liegt darin, ein gemeinsames Verständnis und die Zustimmung zur Produktrichtung im gesamten Unternehmen zu schaffen.

### 2.3. Entwicklungsplan (Development Plan / Project Plan)

Der Entwicklungsplan, oft auch als Projektplan für die Entwicklung bezeichnet, ist ein **detaillierter, operativer Plan, der die spezifischen Aufgaben, den Zeitplan, die benötigten Ressourcen und die Meilensteine für die Implementierung eines Softwareprodukts oder bestimmter Features festlegt**.8 Er operationalisiert die strategischen Vorgaben aus der Roadmap oder detaillierte Anforderungen aus einer Spezifikation.

**Zweck:** Der Hauptzweck ist die **operative Planung und Steuerung der Entwicklungsaktivitäten**.8 Er dient der Koordination von Aufgaben, der Nachverfolgung des Fortschritts, der Zuweisung und Verwaltung von Ressourcen sowie dem Management von Projektrisiken auf operativer Ebene.

**Typischer Inhalt:** Ein Entwicklungsplan ist deutlich detaillierter als eine Roadmap und enthält typischerweise 8:

- Detaillierte Aufgaben, Arbeitspakete oder User Stories
- Zeitpläne (z.B. als Gantt-Diagramm 50 oder in Form von Sprints/Iterationen 54)
- Ressourcenzuweisung (Personal, Budget, Werkzeuge) 8
- Konkrete Meilensteine und Liefergegenstände (Deliverables) 9
- Abhängigkeiten zwischen Aufgaben 29
- Gegebenenfalls eine detaillierte Risikoanalyse und Maßnahmenplanung 39
- Beschreibung des Vorgehensmodells und der Rollen/Aufgaben 39 Der Entwicklungsplan steht in engem Zusammenhang mit der Ressourcenplanung und dem Ressourcenmanagement.8

**Zielgruppe:** Die primäre Zielgruppe ist das **Entwicklungsteam und dessen unmittelbares Management**: Projektleiter, Scrum Master, Entwickler, Tester, eventuell Teamleiter.8

Der Entwicklungsplan fungiert als Brücke zwischen der strategischen Absicht (aus Roadmap oder Anforderungen) und der tatsächlichen Umsetzung durch das Entwicklungsteam. Er übersetzt das 'Was' und 'Warum' in das 'Wie', 'Wer' und 'Wann' auf operativer Ebene.30 Die Granularität und Starrheit dieses Plans variieren jedoch erheblich je nach gewähltem Vorgehensmodell. In klassischen Ansätzen wie dem Wasserfallmodell wird versucht, einen möglichst vollständigen und detaillierten Plan zu Beginn zu erstellen, der dann sequenziell abgearbeitet wird.6 Im agilen Kontext, beispielsweise mit Scrum, ist der initiale Plan oft weniger detailliert (abgeleitet aus dem Product Backlog 67). Die Feinplanung erfolgt iterativ in Sprint-Planungssitzungen, deren Ergebnis der Sprint Backlog ist – ein detaillierter Plan für die nächste kurze Iteration.54 Der Entwicklungsplan transformiert sich somit von einem statischen Dokument zu einem dynamischen, rollierenden Planungsinstrument im agilen Umfeld.38

### 2.4. Implementierungsplan (Implementation/Deployment/Release Plan)

Ein Implementierungsplan (auch Deployment Plan oder Release Plan genannt) ist ein **technischer Plan, der die konkreten Schritte, Prozeduren und Verantwortlichkeiten für die Bereitstellung (Deployment) einer Softwareversion in einer Zielumgebung (z.B. Test-, Staging- oder Produktionsumgebung) detailliert beschreibt**.70 Er ist ein zentrales Element des Release Managements.70

**Zweck:** Das Hauptziel ist die **Sicherstellung einer reibungslosen, kontrollierten und risikoarmen Überführung der entwickelten Software aus der Entwicklungsumgebung in den operativen Betrieb**.70 Er dient dazu, den Release-Prozess zu managen, Ausfallzeiten zu minimieren und die Stabilität der Zielumgebung zu gewährleisten.74

**Typischer Inhalt:** Der Implementierungsplan ist sehr technisch und prozedural ausgerichtet. Er enthält typischerweise 70:

- Genaue Abfolge der Installationsschritte
- Notwendige Konfigurationsanpassungen in der Zielumgebung
- Schritte zur Datenmigration (falls erforderlich)
- Tests, die vor, während und nach dem Deployment durchgeführt werden müssen (Pre-/Post-Deployment Checks)
- Eine detaillierte Rollback-Strategie für den Fall von Problemen
- Einen Zeitplan für die einzelnen Deployment-Aktivitäten (oft außerhalb der Hauptgeschäftszeiten)
- Zuweisung von Verantwortlichkeiten für die einzelnen Schritte
- Einen Kommunikationsplan für die beteiligten Teams und Stakeholder
- Verweise auf die zu deployenden Software-Artefakte (Builds, Pakete) und deren Versionen.82

**Zielgruppe:** Die Zielgruppe besteht primär aus den **technischen Teams**, die am Deployment-Prozess beteiligt sind: Entwickler (insbesondere DevOps-Rollen), Systemadministratoren/Operations-Teams, Datenbankadministratoren, Qualitätssicherungs-/Testteams und eventuell Support-Mitarbeiter.70

Der Implementierungsplan stellt die letzte technische Brücke zwischen der abgeschlossenen Entwicklung und der tatsächlichen Nutzung der Software dar. Sein Fokus ist rein technisch und prozedural, ausgerichtet auf die Minimierung von Risiken und Störungen während der kritischen Phase des Deployments.70 Er unterscheidet sich damit klar vom Entwicklungsplan, der den Bau der Software beschreibt, und von der Roadmap, die die strategische Richtung vorgibt. Die Notwendigkeit eines solchen Plans unterstreicht die Bedeutung der Koordination zwischen Entwicklung und Betrieb – eine Herausforderung, die durch DevOps-Praktiken adressiert wird 87, um sicherzustellen, dass die entwickelte Software in ihrer Zielumgebung wie erwartet funktioniert.

## 3. Vergleichende Analyse der Planungsdokumente

Die vier besprochenen Begriffe – Blaupause, Roadmap, Entwicklungsplan und Implementierungsplan – repräsentieren unterschiedliche Ebenen und Zwecke der Planung im Softwareentwicklungsprozess. Ein direkter Vergleich anhand zentraler Kriterien verdeutlicht ihre Abgrenzung und ihr Zusammenspiel.

**Tabelle 1: Vergleich der Planungsdokumente**

|   |   |   |   |   |
|---|---|---|---|---|
|**Merkmal**|**Blaupause**|**Roadmap (Produkt-Roadmap)**|**Entwicklungsplan (Development Plan)**|**Implementierungsplan (Deployment/Release Plan)**|
|**Hauptzweck**|Konzeptualisierung, frühes Design, Kommunikation|Strategische Ausrichtung, Kommunikation der Vision, Priorisierung 18|Operative Planung der Umsetzung, Aufgabensteuerung, Ressourcenmanagement 30|Technische Planung & Steuerung der Inbetriebnahme/Auslieferung, Risikominimierung 70|
|**Abstraktionsgrad**|Konzeptionell, hoch 11|Strategisch, hoch 18|Operativ, mittel bis detailliert 8|Technisch, prozedural, sehr detailliert 70|
|**Zeitrahmen**|Frühe Phase, konzeptionell|Langfristig (Quartale, Jahre) 19|Mittelfristig (Projektlaufzeit, Sprints) 39|Kurzfristig, spezifisches Ereignis (Release-Datum/-Zeitfenster) 24|
|**Detailgrad**|Gering bis mittel 14|Gering bis mittel, strategisch 18|Hoch, operativ 39|Sehr hoch, technisch-prozedural 70|
|**Typische Zielgruppe**|Architekten, Lead-Entwickler, frühe Stakeholder|Management, Stakeholder, Teams (Entwicklung, Vertrieb, Marketing), Kunden 9|Projektleiter, Entwicklungsteam, Scrum Master 8|Entwicklungsteam, Operations/DevOps, QA-Team 70|
|**Typischer Inhalt (Kurzform)**|Grundkonzept, Architektur-Skizze, Muster 11|Ziele, Initiativen, Themen, grobe Zeitpläne, Meilensteine 18|Detaillierte Aufgaben, Zeitplan, Ressourcen, Meilensteine, Abhängigkeiten, Vorgehen 39|Installations-/Konfigurationsschritte, Tests, Rollback-Plan, Deployment-Zeitplan 70|

Diese Dokumente bilden eine Art Planungshierarchie, die von der breiten strategischen Absicht bis hin zu spezifischen technischen Ausführungsschritten reicht. Die **Roadmap** 18 setzt den strategischen Rahmen und kommuniziert die langfristige Vision. Die **Blaupause** 11 fungiert oft als informelle Brücke zwischen dieser Vision und der konkreteren Planung; sie hält frühe konzeptionelle oder architektonische Ideen fest, bevor alle Details ausgearbeitet sind. Der **Entwicklungsplan** 30 übersetzt die strategischen oder konzeptionellen Vorgaben in einen operativen Plan für das Entwicklungsteam, der detailliert beschreibt, wie die Software gebaut werden soll. Schließlich fokussiert der **Implementierungsplan** 70 auf den letzten Schritt: die technische Überführung der fertigen Software in die Zielumgebung. Jedes dieser Instrumente erfüllt einen spezifischen Zweck für eine bestimmte Zielgruppe und ist in einer bestimmten Phase des Gesamtprozesses von besonderer Bedeutung. Die Notwendigkeit dieser gestaffelten Planung ergibt sich aus der Komplexität, die es erfordert, eine abstrakte Vision in ein funktionierendes, ausgeliefertes Softwaresystem zu verwandeln.

## 4. Der Softwareentwicklungs-Workflow (SDLC - Software Development Lifecycle)

Um die Rolle der genannten Dokumente besser zu verstehen, ist es hilfreich, sie in einen typischen Softwareentwicklungs-Workflow, auch Software Development Lifecycle (SDLC) genannt, einzuordnen. Der SDLC beschreibt die Phasen, die ein Softwareprodukt von der Konzeption bis zur Außerbetriebnahme durchläuft.1 Obwohl es verschiedene Modelle gibt, die diese Phasen unterschiedlich strukturieren und gewichten, lassen sich gemeinsame Kernaktivitäten identifizieren.

**Typische Phasen eines SDLC:**

1. **Planung & Anforderungsanalyse (Planning & Requirement Analysis):**
    
    - Diese initiale Phase beginnt mit der Projektidee oder -vision. Es werden Machbarkeitsstudien durchgeführt, um technische, wirtschaftliche und organisatorische Realisierbarkeit zu prüfen.97
    - Die Projektziele werden definiert 30, und die relevanten Stakeholder (Interessengruppen wie Kunden, Nutzer, Management etc.) werden identifiziert und analysiert.98
    - Die Anforderungen an die Software werden systematisch erhoben (Elicitation), analysiert, spezifiziert und validiert.30 Das Ergebnis ist oft eine Anforderungsspezifikation.
    - Der Projektumfang (Scope) wird festgelegt 1, und eine erste grobe Planung von Zeit, Kosten und Ressourcen erfolgt.1
2. **Design & Architektur (Design & Architecture):**
    
    - Basierend auf den Anforderungen wird die technische Struktur der Software entworfen.1 Dies umfasst die Definition der Softwarearchitektur, d.h. die Zerlegung des Systems in Komponenten und die Beschreibung ihrer Beziehungen und Interaktionen.11
    - Es werden Entscheidungen über Technologien, Frameworks und Plattformen getroffen.92
    - Schnittstellen zwischen Komponenten und zu externen Systemen werden spezifiziert.112
    - Das Design der Datenbank(en) und der Benutzeroberfläche (UI/UX) wird erstellt.113
    - Architekturmuster (z.B. Schichtenarchitektur 120, Microservices 122, MVC 126, MVVM 129) und Designprinzipien (z.B. SOLID 135, KISS 144, DRY 147) werden angewendet, um Qualitätsziele wie Wartbarkeit, Skalierbarkeit und Robustheit zu erreichen.
3. **Implementierung/Entwicklung (Implementation/Development):**
    
    - In dieser Phase wird der eigentliche Programmcode gemäß den Design- und Architekturspezifikationen geschrieben.1
    - Entwickler nutzen spezifische Programmiersprachen und -paradigmen (z.B. Objektorientierte Programmierung (OOP) 150, Funktionale Programmierung 153, Prozedurale Programmierung 150).
    - Unit-Tests (Modultests) werden oft direkt von den Entwicklern geschrieben und ausgeführt, um die Korrektheit einzelner Codeeinheiten zu überprüfen.163
    - Versionskontrollsysteme (z.B. Git) werden eingesetzt, um Änderungen am Code nachzuverfolgen und die Zusammenarbeit im Team zu ermöglichen.82
    - Aspekte wie Nebenläufigkeit (Concurrency) müssen berücksichtigt und korrekt implementiert werden.168
4. **Testen (Testing):**
    
    - Diese Phase dient der systematischen Überprüfung der Software auf Fehler und der Sicherstellung, dass die Anforderungen erfüllt werden.1
    - Es werden verschiedene Teststufen durchlaufen 163:
        - **Integrationstests:** Überprüfung des Zusammenspiels mehrerer Komponenten oder Module.
        - **Systemtests:** Testen des gesamten integrierten Systems gegen die spezifizierten Anforderungen.
        - **Akzeptanztests (User Acceptance Testing, UAT):** Tests durch Endbenutzer oder Kunden, um zu prüfen, ob die Software ihre Bedürfnisse erfüllt und akzeptabel ist.163
    - Qualitätssicherung (Quality Assurance, QA) fokussiert auf die Prozesse zur Fehlervermeidung, während Qualitätskontrolle (Quality Control, QC) auf die Fehlerfindung im Produkt abzielt.33
    - Testplanung (Erstellung von Testplänen 48), Testfallerstellung und Testdurchführung sind zentrale Aktivitäten. Gefundene Fehler (Bugs) werden dokumentiert und deren Behebung verfolgt.179
5. **Bereitstellung/Deployment (Deployment):**
    
    - Die getestete und freigegebene Software wird in die Zielumgebung (z.B. Produktion) überführt und den Endbenutzern zugänglich gemacht.1
    - Dies beinhaltet oft Konfigurationsmanagement, um sicherzustellen, dass die Software in der Zielumgebung korrekt eingerichtet ist.82
    - Der Prozess wird durch einen Implementierungs- oder Deployment-Plan gesteuert.
6. **Wartung & Betrieb (Maintenance & Operation):**
    
    - Nach der Inbetriebnahme beginnt die Phase des Betriebs und der Wartung.1
    - Dies umfasst die laufende Überwachung des Systems auf Leistung und Fehler, die Behebung von neu entdeckten Fehlern (Bugfixing), die Bereitstellung von Updates und Patches sowie den Benutzersupport.
    - Feedback von Benutzern wird gesammelt und fließt oft in die Planung zukünftiger Versionen oder Verbesserungen ein.4

Vorgehensmodelle:

Die Art und Weise, wie diese Phasen durchlaufen werden, wird durch das gewählte Vorgehensmodell bestimmt.

- **Klassische Modelle** wie das **Wasserfallmodell** 65 oder das **V-Modell** 65 sehen einen linearen, sequenziellen Durchlauf der Phasen vor, wobei jede Phase abgeschlossen sein muss, bevor die nächste beginnt.
- Das **Spiralmodell** 65 ist iterativ und risikogetrieben, durchläuft die Phasen aber in Zyklen.
- **Agile Methoden** wie **Scrum** 185, **Kanban** 185 oder **Extreme Programming (XP)** 185 verfolgen einen iterativen und inkrementellen Ansatz, bei dem die Phasen in kurzen Zyklen (Sprints bei Scrum) wiederholt durchlaufen werden, um schnell funktionierende Softwareteile zu liefern und flexibel auf Änderungen reagieren zu können.2

Obwohl der SDLC einen notwendigen konzeptionellen Rahmen bietet, ist seine praktische Anwendung stark kontextabhängig. Die spezifische Ausgestaltung der Phasen, ihre Reihenfolge und die Übergänge zwischen ihnen variieren erheblich je nach Projektkomplexität, der gewählten Methodik (agil vs. klassisch) und organisatorischen Faktoren.2 Agile Methoden weichen explizit von einer streng linearen Abfolge ab und betonen Iteration und Feedbackschleifen.2 Die Wahl des passenden Modells hängt entscheidend von Faktoren wie der Stabilität der Anforderungen und der Komplexität des Projekts ab.65 Der SDLC ist somit weniger eine starre Vorschrift als vielmehr eine flexible Landkarte, deren konkrete Route durch die spezifischen Gegebenheiten des Projekts bestimmt wird.

## 5. Einordnung der Dokumente in den Workflow

Die im SDLC entstehenden Informationen und Entscheidungen werden in verschiedenen Dokumenten und Artefakten festgehalten. Ein **Artefakt** ist dabei generell ein von Menschen erzeugter Gegenstand oder ein Ergebnis eines Arbeitsprozesses.195 Im Software-Kontext können dies Code, Modelle, Dokumente oder ausführbare Dateien sein.195 Diese Artefakte dienen der Kommunikation, Dokumentation, Planung und Steuerung des Projekts.

Die Zuordnung der besprochenen Planungsdokumente und weiterer wichtiger Artefakte zu den SDLC-Phasen sieht typischerweise wie folgt aus, wobei Unterschiede zwischen klassischen und agilen Ansätzen (hier am Beispiel Scrum) bestehen:

**Tabelle 2: Einordnung der Dokumente/Artefakte in den SDLC**

|   |   |   |
|---|---|---|
|**SDLC-Phase**|**Typische Dokumente/Artefakte (Klassisch)**|**Typische Dokumente/Artefakte (Agil - Scrum Beispiel)**|
|**Initiierung/Planung**|Projektantrag, Machbarkeitsstudie 97, Projektziele 30, Stakeholder-Analyse 98, **Produktvision** 22, **Roadmap** 18, **Blaupause** (Konzept) 11|**Produktvision** 22, **Roadmap** 9, **Product Backlog** (initial) 67, Stakeholder-Analyse 98|
|**Anforderung**|**Lastenheft** (Auftraggeber) 200, **Pflichtenheft** (Auftragnehmer) 200, Anforderungsspezifikation 200|**Product Backlog** (detailliert, priorisiert) 47, User Stories, Akzeptanzkriterien|
|**Architektur/Design**|**Architekturdesign-Dokument** 106, **Blaupause** (Architekturskizze) 11, Detail-Design-Spezifikationen, UI/UX-Entwürfe 108|**Architekturdesign** (oft emergent, iterativ), **Blaupause** (als Skizze), UI/UX-Prototypen, Spikes (zur Technologieerkundung)|
|**Implementierung/Entwicklung**|**Entwicklungsplan** 30, Quellcode 196, Kompilierte Binärdateien 196, Bibliotheken 196, Unit-Tests 167|**Sprint Backlog** 47, Quellcode, Kompilierte Binärdateien, Bibliotheken, Unit-Tests, **Increment** (potenziell auslieferbar) 67, **Definition of Done (DoD)** 47|
|**Test**|**Testplan** 48, Testfälle 48, Testprotokolle/Berichte 48|**Testplan** (oft integriert/iterativ), Testfälle (oft automatisiert), Testprotokolle/Berichte, DoD beinhaltet Testkriterien 206|
|**Deployment**|**Implementierungsplan** (Deployment/Release Plan) 70, Release Notes 198, Installationsanleitung 39|**Implementierungsplan** (oft automatisiert via CI/CD 70), Release Notes, DoD kann Deployment-Kriterien enthalten 207|
|**Wartung/Betrieb**|Bug Reports, Änderungsanträge (Change Requests) 198, Wartungsdokumentation|Bug Reports, User Feedback, Monitoring-Daten 70, Aktualisiertes Product Backlog|

**Abhängigkeiten und Informationsfluss:**

Die Artefakte bauen logisch aufeinander auf und ermöglichen einen Informationsfluss durch den Entwicklungsprozess:

- **Klassischer Fluss (vereinfacht):** Die Produktvision 22 und strategische Ziele fließen in die **Roadmap**.18 Diese bildet die Basis für das **Lastenheft** 200, das die Anforderungen des Auftraggebers detailliert. Der Auftragnehmer antwortet mit dem **Pflichtenheft** 200, das beschreibt, wie die Anforderungen umgesetzt werden sollen. Darauf basierend entsteht das **Architekturdesign-Dokument**.106 Der **Entwicklungsplan** 30 detailliert die Umsetzungsschritte. Während der Implementierung entsteht der Code und die Unit-Tests. Der **Testplan** 166 leitet die Verifikations- und Validierungsaktivitäten. Schließlich steuert der **Implementierungsplan** 70 die Auslieferung.
- **Agiler Fluss (Scrum, vereinfacht):** Die **Produktvision** 22 und die **Roadmap** 9 liefern den Input für das **Product Backlog**.67 Im **Sprint Planning** 55 werden Items aus dem Product Backlog ausgewählt und im **Sprint Backlog** 67 detailliert geplant. Das Entwicklungsteam arbeitet an der Umsetzung und erstellt ein **Increment** 67, das der **Definition of Done** 47 entspricht. Im **Sprint Review** 55 wird das Increment den Stakeholdern präsentiert und Feedback gesammelt, das wieder ins Product Backlog fließt. Die **Sprint Retrospektive** 55 dient der Prozessverbesserung. Der **Implementierungsplan** kommt ins Spiel, wenn ein Increment tatsächlich ausgeliefert wird (was potenziell nach jedem Sprint geschehen kann).

Die Abfolge und die formale Ausgestaltung dieser Dokumente und Artefakte werden maßgeblich durch die gewählte Entwicklungsmethodik geprägt. Während ein logischer Informationsfluss von der Anforderung über das Design zur Implementierung und Auslieferung immer existiert, komprimieren oder iterieren agile Praktiken diese Schritte. Klassische Modelle tendieren dazu, umfassende, statische Dokumente als Ergebnis abgeschlossener Phasen zu produzieren (z.B. Lasten-/Pflichtenheft, detaillierter Entwicklungsplan).65 Agile Ansätze ersetzen oder transformieren viele dieser traditionellen Dokumente in dynamischere, schlankere Artefakte wie Backlogs und die Definition of Done, die den iterativen Arbeitsfluss unterstützen und kontinuierliches Feedback ermöglichen.47 Die Roadmap behält oft auch im agilen Umfeld ihre strategische Bedeutung, dient aber als Input für das sich ständig weiterentwickelnde Product Backlog.9

## 6. Unterschiede: Klassische vs. Agile Vorgehensmodelle

Die Wahl des Vorgehensmodells – klassisch (wie Wasserfall oder V-Modell) oder agil (wie Scrum oder Kanban) – hat tiefgreifende Auswirkungen darauf, wie Planung und Dokumentation im Softwareentwicklungsprozess gehandhabt werden.

**Planungshorizont & Detailgrad:**

- **Klassisch (Wasserfall):** Charakteristisch ist eine **detaillierte Gesamtplanung zu Beginn** des Projekts.6 Der gesamte Funktionsumfang, der Zeitplan und die Ressourcen werden möglichst vollständig im Voraus festgelegt. Der Entwicklungsplan ist oft ein umfassendes Dokument, das den gesamten Projektablauf abbildet.
- **Agil (Scrum):** Die Planung erfolgt **iterativ und adaptiv**. Zu Beginn gibt es eine übergeordnete Planung (Produktvision, Roadmap, initiales Product Backlog).9 Die **detaillierte Planung findet jedoch nur für den nächsten kurzen Zyklus (Sprint)** statt (Sprint Planning), dessen Ergebnis der Sprint Backlog ist.38 Der Planungshorizont ist somit kurzfristiger, die Planung rollierend.

**Dokumentation:**

- **Klassisch (Wasserfall):** Legt großen Wert auf **umfassende, formale Dokumentation** als Ergebnis jeder Phase (z.B. Lastenheft, Pflichtenheft, Designspezifikation, Testplan).6 Diese Dokumente dienen oft als Vertragsgrundlage und als "Gate" zur nächsten Phase.6
- **Agil (Scrum):** Priorisiert **funktionierende Software über umfassende Dokumentation**.7 Dokumentation wird als Mittel zum Zweck gesehen ("just enough") und ist oft schlanker und dynamischer (z.B. Backlogs, Wiki-Seiten, automatisierte Tests als Dokumentation).210 Die zentralen **Scrum-Artefakte (Product Backlog, Sprint Backlog, Increment mit Definition of Done)** sind die wesentlichen Informationsträger und ersetzen viele traditionelle Dokumente.47

**Umgang mit Änderungen:**

- **Klassisch (Wasserfall):** Änderungen während des Projektablaufs werden als störend empfunden und sind oft **schwierig und kostspielig** zu integrieren.6 Sie erfordern formale Änderungsmanagementprozesse (Change Control).
- **Agil (Scrum):** Änderungen werden als **normal und sogar willkommen** betrachtet.88 Das iterative Vorgehen ermöglicht es, neue Anforderungen oder geänderte Prioritäten in nachfolgenden Sprints zu berücksichtigen und das Produkt flexibel anzupassen.

**Rolle der Dokumente:**

- **Klassisch (Wasserfall):** Dokumente dienen als **detaillierte Spezifikation**, als **Vertragsgrundlage**, zur **Abnahme von Phasen** und zur Sicherstellung der Nachvollziehbarkeit.6
- **Agil (Scrum):** Dokumente (bzw. Artefakte) dienen primär der **Kommunikation im Team und mit Stakeholdern**, als **Planungshilfe für Iterationen**, zur **Visualisierung des Fortschritts** und als **lebendige Grundlage** für die weitere Entwicklung.210

**Flexibilität vs. Vorhersagbarkeit:**

- **Klassisch (Wasserfall):** Zielt auf hohe **Vorhersagbarkeit** bezüglich Umfang, Zeit und Kosten basierend auf der initialen Planung.6 Dies funktioniert gut bei stabilen Anforderungen.
- **Agil (Scrum):** Priorisiert **Flexibilität und Anpassungsfähigkeit** an sich ändernde Bedingungen und Anforderungen.6 Die Vorhersagbarkeit bezieht sich eher auf kurze Zyklen (Sprints).

Der fundamentale Unterschied zwischen klassischen und agilen Modellen liegt in ihrer Philosophie bezüglich Planung und Veränderung. Klassische Modelle versuchen, Unsicherheit und Veränderung durch umfassende Vorabplanung und detaillierte Dokumentation zu kontrollieren.6 Sie basieren auf der Annahme, dass Anforderungen zu Beginn weitgehend bekannt und stabil sind. Agile Modelle hingegen akzeptieren und begrüßen Veränderung als integralen Bestandteil der Softwareentwicklung.7 Sie nutzen iterative Planungszyklen und leichtgewichtige, anpassungsfähige Artefakte, um kontinuierlich Wert zu liefern und auf Feedback zu reagieren. Scrum implementiert dies durch klar definierte Rollen, Ereignisse und Artefakte, die einen empirischen Prozess der Inspektion und Anpassung ermöglichen.57 Dies stellt einen Paradigmenwechsel dar: weg von der Vorhersage und Kontrolle hin zur Anpassung und empirischen Prozesssteuerung. Die Wahl der Methodik bestimmt somit maßgeblich, welche Planungsdokumente wie detailliert, wann und zu welchem Zweck erstellt und verwendet werden.

## 7. Fazit

Die Begriffe 'Blaupause', 'Roadmap', 'Entwicklungsplan' und 'Implementierungsplan' bezeichnen im Kontext der Softwareentwicklung distinkte Planungs- und Kommunikationsinstrumente, die jeweils unterschiedliche Zwecke erfüllen und sich an verschiedene Zielgruppen richten.

- Die **Blaupause** ist meist eine informelle, konzeptionelle Skizze oder ein früher Entwurf, der eine grundlegende Idee oder Architektur umreißt.
- Die **Roadmap** dient der strategischen Ausrichtung und Kommunikation der Produktvision und -entwicklung über einen längeren Zeitraum.
- Der **Entwicklungsplan** fokussiert auf die operative Planung und Steuerung der konkreten Umsetzungsaufgaben durch das Entwicklungsteam.
- Der **Implementierungsplan** schließlich beschreibt detailliert die technischen Schritte zur Inbetriebnahme einer Softwareversion.

Die wesentlichen Unterschiede liegen im **Abstraktionsgrad** (strategisch bis technisch-prozedural), im **Zeitrahmen** (langfristig bis kurzfristig/ereignisbezogen), im **Detailgrad** (grob bis sehr detailliert) und im **Hauptzweck** (strategische Kommunikation bis technische Ausführung).

Die Art, der Detaillierungsgrad, der Zeitpunkt der Erstellung und die Bedeutung dieser Dokumente werden maßgeblich durch das gewählte **Vorgehensmodell** beeinflusst. Klassische Modelle wie Wasserfall setzen auf umfassende Vorabplanung und formale, oft statische Dokumente als Phasenergebnisse. Agile Modelle wie Scrum bevorzugen iterative Planung, schlanke, dynamische Artefakte (wie Backlogs und die Definition of Done) und priorisieren funktionierende Software und die Reaktion auf Veränderungen über umfassende Dokumentation.

Ein klares Verständnis dieser Begriffe und ihrer jeweiligen Rolle im Softwareentwicklungsprozess ist für Projektleiter, Entwickler und andere Stakeholder entscheidend. Die bewusste Auswahl und der angemessene Einsatz dieser Planungsinstrumente, angepasst an den spezifischen Projektkontext und die gewählte Methodik, fördern eine effektive Kommunikation, schaffen Klarheit über Ziele und Vorgehen und tragen somit maßgeblich zum Erfolg von Softwareprojekten bei.