H# Git auf Manjaro nutzen: Ein Leitfaden für Terminal und VS Code

Dieser Leitfaden erklärt die Grundlagen der Git-Nutzung auf einem Manjaro-System, sowohl über die Kommandozeile als auch integriert in Visual Studio Code (VS Code). Git ermöglicht es dir, Änderungen an deinen Projekten zu verfolgen, zu verwalten und flexibel damit umzugehen.

## Was ist Git?

Git ist ein verteiltes Versionskontrollsystem. Das bedeutet, dass jeder Entwickler, der mit einem Projekt arbeitet, eine vollständige Kopie des gesamten Projektverlaufs auf seinem lokalen Rechner hat. Dies unterscheidet sich von zentralisierten Systemen, bei denen es nur eine zentrale Kopie gibt.

Die Hauptvorteile von Git sind:

- **Versionskontrolle:** Verfolge jede Änderung an deinem Code.
    
- **Zusammenarbeit:** Arbeite effizient mit anderen an Projekten.
    
- **Branching:** Erstelle unabhängige Entwicklungszweige für neue Features oder Experimente.
    
- **Flexibilität:** Wechsle einfach zwischen verschiedenen Versionen deines Projekts.
    

## Installation von Git auf Manjaro

Git ist auf Manjaro in der Regel bereits vorinstalliert. Du kannst dies im Terminal überprüfen:

```
git --version
```

Sollte Git nicht installiert sein, kannst du es über den Paketmanager Pacman installieren:

```
sudo pacman -S git
```

## Grundlegende Git-Konfiguration

Bevor du Git verwendest, solltest du deinen Namen und deine E-Mail-Adresse konfigurieren. Diese Informationen werden in deinen Commits gespeichert.

```
git config --global user.name "tfufuz1"
git config --global user.email "tfufuz@gmail.com"
```

Ersetze "Dein Name" und "deine.email@example.com" durch deine tatsächlichen Daten.

## Ein Projekt in ein Git-Repository umwandeln (Initialisierung)

Um einen bestehenden Ordner in ein Git-Repository umzuwandeln, navigiere im Terminal in diesen Ordner und führe den folgenden Befehl aus:

```
cd /pfad/zu/deinem/ordner
git init
```

Dadurch wird ein versteckter `.git`-Ordner in deinem Projektverzeichnis erstellt, der alle Informationen zur Versionskontrolle enthält.

Wenn du ein neues Projekt startest und es sofort unter Versionskontrolle stellen möchtest, kannst du zuerst ein neues Verzeichnis erstellen und dann `git init` darin ausführen:

```
mkdir mein-neues-projekt
cd mein-neues-projekt
git init
```

## Der Git-Workflow: Hinzufügen, Committen, Pushen

Der grundlegende Workflow in Git besteht aus drei Schritten:

1. **Änderungen hinzufügen (Staging):** Du wählst die Dateien aus, deren Änderungen du in den nächsten Commit aufnehmen möchtest. Dies geschieht im sogenannten "Staging Area".
    
2. **Änderungen committen:** Du erstellst einen "Commit", der einen Schnappschuss des aktuellen Zustands der gestagten Dateien darstellt. Jeder Commit hat eine eindeutige ID und eine Commit-Nachricht, die beschreibt, was geändert wurde.
    
3. **Änderungen pushen (optional):** Wenn du mit einem Remote-Repository (z. B. auf GitHub, GitLab oder Bitbucket) arbeitest, kannst du deine lokalen Commits in das Remote-Repository hochladen.
    

### Im Terminal

**1. Änderungen hinzufügen (Staging):**

Um alle geänderten Dateien im aktuellen Verzeichnis und seinen Unterverzeichnissen zum Staging Area hinzuzufügen:

```
git add .
```

Um eine bestimmte Datei hinzuzufügen:

```
git add dateiname.txt
```

Um alle Dateien in einem bestimmten Ordner hinzuzufügen:

```
git add ordnername/
```

**2. Änderungen committen:**

Erstelle einen Commit mit einer beschreibenden Nachricht:

```
<<<<<<< HEAD
git commit -m "Eine kurze Beschreibung der Änderungen"
=======
git commit -m "__ÄNDERUNG__"
```

```
git commit -m "0.0.1"
>>>>>>> 058544c (Doc-Upload)
```

Die Commit-Nachricht sollte prägnant erklären, was in diesem Commit geändert wurde.

**3. Änderungen pushen (falls ein Remote-Repository verbunden ist):**

Um deine lokalen Commits in das Remote-Repository hochzuladen (typischerweise auf den `main` oder `master` Branch):

````
git push origin main
```origin` ist der Standardname für das Remote-Repository, und `main` ist der Standardname für den Hauptentwicklungszweig.

### In Visual Studio Code

VS Code hat eine hervorragende integrierte Git-Unterstützung.

1.  **Repository öffnen:** Öffne deinen Projektordner in VS Code (`File > Open Folder`). Wenn der Ordner ein Git-Repository ist, erkennt VS Code dies automatisch.
2.  **Source Control Ansicht:** Klicke auf das Source Control Icon in der linken Seitenleiste (sieht aus wie drei verbundene Kreise). Hier siehst du alle Änderungen in deinem Repository.
3.  **Änderungen hinzufügen (Staging):** Unter "Changes" siehst du alle Dateien mit ungestagten Änderungen. Fahre mit der Maus über eine Datei und klicke auf das Plus-Symbol, um sie zum Staging Area hinzuzufügen. Du kannst auch auf das Plus-Symbol neben "Changes" klicken, um alle Änderungen zu stagen. Gestagte Änderungen erscheinen unter "Staged Changes".
4.  **Änderungen committen:** Gib eine Commit-Nachricht in das Textfeld über "Staged Changes" ein und klicke auf den Haken-Button, um den Commit zu erstellen.
5.  **Änderungen pushen:** Wenn ein Remote-Repository konfiguriert ist, erscheint ein Button mit einer Wolke und einem Pfeil nach oben in der Statusleiste unten links oder im Menü der Source Control Ansicht (drei Punkte). Klicke darauf, um deine Commits zu pushen.

## Dateien modifizieren und entfernen

Wenn du Dateien in deinem Projekt änderst oder löschst, erkennt Git diese Änderungen.

### Im Terminal

**Dateien modifizieren:**

Nachdem du eine Datei geändert hast, siehst du die Änderung mit `git status`:

```bash
git status
````

Die Datei wird als "modified" angezeigt. Um die Änderung zu committen, musst du sie wieder stagen und committen:

```
git add geaenderte_datei.txt
git commit -m "Beschreibung der Änderung"
```

**Dateien entfernen:**

Um eine Datei aus deinem Projekt und dem Git-Repository zu entfernen:

```
git rm zu_entfernende_datei.txt
```

Dieser Befehl löscht die Datei aus dem Dateisystem und staget die Entfernung. Dann musst du die Entfernung committen:

```
git commit -m "Datei entfernt"
```

Wenn du eine Datei manuell gelöscht hast (z. B. über den Dateimanager), erkennt Git dies ebenfalls. Du kannst die Entfernung dann mit `git add` stagen und committen:

```
git add geloeschte_datei.txt
git commit -m "Datei manuell entfernt und committet"
```

### In Visual Studio Code

**Dateien modifizieren:**

Geänderte Dateien erscheinen unter "Changes" in der Source Control Ansicht. Stage und committe sie wie oben beschrieben.

**Dateien entfernen:**

Wenn du eine Datei in VS Code löschst, erscheint sie ebenfalls unter "Changes" als "Deleted". Stage und committe die Änderung, um die Entfernung im Repository zu speichern.

## Flexible Änderungen mit Branching

Branching ist eine der mächtigsten Funktionen von Git. Es erlaubt dir, unabhängige Entwicklungszweige zu erstellen. Das ist ideal, um an neuen Features zu arbeiten, ohne den Hauptentwicklungszweig (oft `main` oder `master`) zu beeinträchtigen.

### Im Terminal

**Neuen Branch erstellen:**

```
git branch mein-neues-feature
```

**Zu einem Branch wechseln:**

```
git checkout mein-neues-feature
```

Oder, moderner:

```
git switch mein-neues-feature
```

**Neuen Branch erstellen und sofort wechseln:**

```
git checkout -b mein-neues-feature
```

Oder, moderner:

```
git switch -c mein-neues-feature
```

**Branches anzeigen:**

```
git branch
```

Der aktuell aktive Branch ist mit einem Sternchen markiert.

**Branches zusammenführen (Merging):**

Wenn du die Änderungen von einem Branch (z. B. `mein-neues-feature`) in einen anderen Branch (z. B. `main`) übernehmen möchtest, wechselst du zuerst zu dem Branch, in den du zusammenführen möchtest, und führst dann den Merge durch:

```
git switch main
git merge mein-neues-feature
```

**Branch löschen:**

Nachdem ein Branch zusammengeführt wurde und nicht mehr benötigt wird, kannst du ihn löschen:

```
git branch -d mein-neues-feature
```

Wenn der Branch noch nicht zusammengeführt wurde und du ihn trotzdem löschen möchtest (Vorsicht, Änderungen gehen verloren!), verwende `-D`:

```
git branch -D mein-neues-feature
```

### In Visual Studio Code

In der Source Control Ansicht von VS Code siehst du oben links den Namen des aktuellen Branches. Klicke darauf, um eine Liste der Branches anzuzeigen und zu anderen Branches zu wechseln oder neue zu erstellen.

Zum Mergen: Wechsle zum Ziel-Branch, klicke im Menü der Source Control Ansicht (drei Punkte) auf "Branch" und wähle "Merge Branch...". Wähle dann den Branch aus, den du mergen möchtest.

Zum Löschen eines Branches: Klicke im Branch-Menü auf den Branch, den du löschen möchtest, und wähle die Option zum Löschen.

## Nützliche Git-Befehle

- `git status`: Zeigt den aktuellen Status deines Repositorys an (geänderte, gestagte, ungetrackte Dateien).
    
- `git log`: Zeigt den Commit-Verlauf an.
    
- `git diff`: Zeigt die Unterschiede zwischen verschiedenen Versionen (z. B. zwischen Arbeitsverzeichnis und Staging Area oder zwischen zwei Commits).
    
- `git clone <url>`: Klont ein Remote-Repository auf deinen lokalen Rechner.
    
- `git pull`: Holt die neuesten Änderungen von einem Remote-Repository und führt sie in deinen aktuellen Branch zusammen.
    

## Git und Remote-Repositories (GitHub, GitLab, etc.)

Um dein lokales Repository mit einem Remote-Repository zu verbinden (z. B. auf GitHub), musst du das Remote hinzufügen:

```
git remote add origin <url_des_remote_repositorys>
```

Ersetze `<url_des_remote_repositorys>` durch die tatsächliche URL deines Remote-Repositorys.

Danach kannst du deine lokalen Commits mit `git push origin main` hochladen und Änderungen von anderen mit `git pull origin main` herunterladen.

## Fazit

Git ist ein unverzichtbares Werkzeug für die Softwareentwicklung. Mit den hier beschriebenen Grundlagen kannst du bereits effektiv mit Git arbeiten, deine Projektänderungen verfolgen und von den Vorteilen der Versionskontrolle profitieren. Die Integration in VS Code macht die Nutzung noch komfortabler.

## Weitere Schritte

- Erkunde fortgeschrittene Git-Konzepte wie Rebase, Stashing und Tags.
    
- Lerne, wie du Merge-Konflikte löst.
    
- Mache dich mit den Funktionen deines bevorzugten Hosting-Dienstes (GitHub, GitLab, Bitbucket) vertraut.
    
- Übe die Befehle regelmäßig, um Routine zu entwickeln.