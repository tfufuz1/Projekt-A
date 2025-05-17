## Git Spickzettel - Deutsch
## Git Spickzettel - Deutsch

Yo, check mal! Hier sind die krassesten Git-Befehle, die wir gecheckt haben, in easy-peasy Deutsch, damit du Git voll drauf hast!

**BLUF: Für einfache Synchronisation deines lokalen Repositorys mit einem Remote-Repository brauchst du diese Git-Befehle:**

- **`git status`**: Checkt, welche Änderungen du lokal gemacht hast.
    
```
git status

```

- **`git add`**: Fügt deine Änderungen zum Staging-Bereich hinzu, damit sie im nächsten Commit enthalten sind.
    
```
git add .
```

- **`git commit -m "Deine Commit-Nachricht"`**: Speichert deine Änderungen mit einer Beschreibung.
    
```
git commit -m "Deine Commit-Nachricht"
```

- **`git pull origin <branch-name>`**: Holt die neuesten Änderungen vom Remote-Repository und führt sie mit deinen lokalen Änderungen zusammen.
    
```
git pull origin main
```

- **`git push origin <branch-name>`**: Lädt deine lokalen Commits ins Remote-Repository hoch.
    
```
git push origin main
```


```
git push --force origin main
```

```
git push --force-with-lease origin main
```


```
tfufuz1
```

```
Eselsh@ut61
```

### Grundlegende Konfiguration

- **Benutzernamen und E-Mail-Adresse festlegen:**


```
git config --global user.name "tfufuz1"
```

```
git config --global user.email "tfufuz@gmail.com" 
```


    Mit diesen Befehlen legst du deinen Namen und deine E-Mail-Adresse global für alle deine Git-Sachen fest. Super wichtig, weil Git diese Infos für deine Commits braucht. Ohne das geht's nicht! Dein Name und deine E-Mail-Adresse werden im Commit-Verlauf gespeichert und sind für jeden sichtbar, der sich das Repository ansieht. Mit der Option `--global` gelten diese Einstellungen für alle Git-Repositories auf deinem System. Du kannst sie aber auch nur für ein einzelnes Repository festlegen, indem du `--local` statt `--global` verwendest. Dann gelten die Einstellungen nur für das Repository im aktuellen Ordner.
    

### Ein Repository starten

- **Ein neues Repository in einem Verzeichnis erstellen:**
    
    ```
    git init
    ```
    
    Dieser Befehl initialisiert ein neues, leeres Git-Repository im angegebenen Ordner. Ein Git-Repository ist einfach ein Ordner, der alle deine Dateien und die Historie aller Änderungen an diesen Dateien enthält. Durch die Initialisierung eines Repositorys wird ein versteckter Ordner namens `.git` im angegebenen Ordner erstellt. Dieser Ordner enthält alle Metadaten, die Git zur Verwaltung des Repositorys benötigt. Wichtig: `git init` muss in dem Ordner ausgeführt werden, der als Wurzelverzeichnis deines Projekts dienen soll. Wenn du dich bereits in einem Unterordner befindest, wird das Repository in diesem Unterordner erstellt, was wahrscheinlich nicht das ist, was du willst.
    
- **Ein Repository von einem Remote-Server klonen:**
    
    ```
    git clone <URL-des-Repositorys>
    ```
    
    Dieser Befehl wird verwendet, um eine Kopie eines bestehenden Git-Repositorys von einem Remote-Server auf deinen Rechner zu erstellen. Ein Remote-Server ist ein Server, auf dem das Repository gehostet wird, z. B. GitHub, GitLab oder Bitbucket. Durch das Klonen eines Repositorys wird nicht nur die neueste Version der Dateien heruntergeladen, sondern auch die gesamte Historie aller Änderungen an diesen Dateien. So kannst du die Historie des Projekts einsehen und bei Bedarf zu früheren Versionen zurückkehren. Der Befehl `git clone` erstellt einen neuen Ordner mit dem Namen des Repositorys und lädt die Dateien und die Historie in diesen Ordner herunter.
    

### Änderungen speichern

- **Den Status deines Arbeitsverzeichnisses anzeigen:**
    
    ```
    git status
    ```
    
    Der Befehl `git status` gibt dir einen Überblick über den aktuellen Zustand deines Arbeitsverzeichnisses. Das Arbeitsverzeichnis ist der Ordner auf deinem Rechner, in dem sich die Dateien deines Projekts befinden. `git status` zeigt dir, welche Dateien geändert wurden, welche Dateien sich im Staging-Bereich befinden und welche Dateien nicht von Git verfolgt werden. Diese Infos sind super wichtig, um den Überblick über deine Änderungen zu behalten und zu entscheiden, welche Änderungen in den nächsten Commit aufgenommen werden sollen. Die Ausgabe von `git status` ist in verschiedene Abschnitte unterteilt, die dir unterschiedliche Infos anzeigen. Zum Beispiel werden Dateien, die geändert, aber noch nicht zum Staging-Bereich hinzugefügt wurden, unter der Überschrift "Changes not staged for commit" aufgelistet, während Dateien im Staging-Bereich unter "Changes to be committed" angezeigt werden.
    
- **Eine oder mehrere Dateien zum Staging-Bereich hinzufügen:**
    
    ```
    git add <Datei1> <Datei2> ...
    git add . # Alle Änderungen im aktuellen Verzeichnis hinzufügen
    ```
    
    Der Befehl `git add` wird verwendet, um Änderungen an Dateien zum Staging-Bereich hinzuzufügen. Der Staging-Bereich ist ein Bereich, in dem Änderungen für den nächsten Commit vorbereitet werden. Wenn du eine Datei änderst, bedeutet das nicht automatisch, dass diese Änderung im nächsten Commit enthalten ist. Stattdessen musst du die Änderung explizit mit `git add` zum Staging-Bereich hinzufügen. Dadurch kannst du auswählen, welche Änderungen in einen Commit einfließen sollen, und Commits in logische Einheiten unterteilen. Du kannst `git add` mit einem oder mehreren Dateinamen als Argumente aufrufen, um bestimmte Dateien zum Staging-Bereich hinzuzufügen. Oder du benutzt den Punkt `.` als Argument, um alle Änderungen im aktuellen Verzeichnis und seinen Unterverzeichnissen zum Staging-Bereich hinzuzufügen. Aber Vorsicht, damit fügst du nicht versehentlich Änderungen hinzu, die nicht in den nächsten Commit sollen!
    
- **Änderungen im Staging-Bereich anzeigen:**
    
    ```
    git diff --staged
    ```
    
    Der Befehl `git diff` wird verwendet, um die Unterschiede zwischen verschiedenen Versionen einer Datei anzuzeigen. In seiner einfachsten Form zeigt `git diff` die Unterschiede zwischen dem Arbeitsverzeichnis und dem Staging-Bereich an. Mit der Option `--staged` zeigt der Befehl stattdessen die Unterschiede zwischen dem Staging-Bereich und dem letzten Commit an. Das ist praktisch, um genau zu sehen, welche Änderungen im nächsten Commit enthalten sein werden. Die Ausgabe von `git diff` wird im sogenannten "Diff-Format" angezeigt, das die hinzugefügten, entfernten und geänderten Zeilen hervorhebt.
    
- **Einen Commit mit den Änderungen im Staging-Bereich erstellen:**
    
    ```
    git commit -m "Beschreibung der Änderungen"
    ```
    
    Der Befehl `git commit` wird verwendet, um einen neuen Commit zu erstellen, der die Änderungen im Staging-Bereich enthält. Ein Commit ist ein Schnappschuss des Repositorys zu einem bestimmten Zeitpunkt. Jeder Commit enthält eine eindeutige ID, einen Zeitstempel, den Autor des Commits und eine optionale Commit-Nachricht. Die Commit-Nachricht ist eine kurze Beschreibung der im Commit enthaltenen Änderungen. Sie ist super wichtig, um die Historie des Projekts zu dokumentieren und es anderen (und dir selbst in der Zukunft) zu ermöglichen, die vorgenommenen Änderungen zu verstehen. Die Option `-m` wird verwendet, um die Commit-Nachricht direkt in der Befehlszeile anzugeben. Wenn du diese Option nicht verwendest, öffnet Git einen Texteditor, in dem du die Commit-Nachricht eingeben kannst. Es wird dringend empfohlen, aussagekräftige Commit-Nachrichten zu verfassen, die die Änderungen präzise und prägnant beschreiben.
    

### Branches und Merges

- **Alle lokalen Branches anzeigen:**
    
    ```
    git branch
    ```
    
    Ein Branch in Git stellt eine unabhängige Entwicklungslinie dar. Mit Branches kannst du an neuen Features arbeiten, Fehler beheben oder experimentieren, ohne die Hauptentwicklungslinie zu beeinträchtigen. Der Befehl `git branch` listet alle lokalen Branches im Repository auf. Der aktuell aktive Branch ist durch ein Sternchen `*` gekennzeichnet. Lokale Branches sind Branches, die nur auf deinem Rechner existieren. Es gibt auch Remote-Branches, die auf einem Remote-Server existieren.
    
- **Einen neuen Branch erstellen:**
    
    ```
    git branch <Name-des-Branches>
    ```
    
    Mit diesem Befehl erstellst du einen neuen Branch mit dem angegebenen Namen. Der neue Branch wird von dem Commit erstellt, auf dem du dich gerade befindest. Das bedeutet, dass der neue Branch zunächst genau den gleichen Inhalt wie der aktuelle Branch hat. Danach kannst du mit dem Befehl `git checkout` zu dem neuen Branch wechseln und daran arbeiten.
    
- **Zu einem anderen Branch wechseln:**
    
    ```
    git checkout <Name-des-Branches>
    ```
    
    Der Befehl `git checkout` wird verwendet, um zu einem anderen Branch zu wechseln oder einen Commit auszuchecken. Wenn ein Branch ausgecheckt wird, wird das Arbeitsverzeichnis so aktualisiert, dass es den Dateien und der Historie dieses Branches entspricht. Alle neuen Commits werden dann zu diesem Branch hinzugefügt. Wenn stattdessen ein Commit ausgecheckt wird, wird das Arbeitsverzeichnis in den Zustand dieses Commits versetzt, und alle neuen Commits würden von diesem Commit abzweigen, wodurch ein "detached HEAD"-Zustand entsteht, der für die meisten Benutzer unerwünscht ist.
    
- **Einen Branch erstellen und sofort zu ihm wechseln:**
    
    ```
    git checkout -b <Name-des-Branches>
    ```
    
    Dieser Befehl ist eine Abkürzung für die Erstellung eines neuen Branches mit `git branch` und das anschließende Wechseln zu diesem Branch mit `git checkout`. Die Option `-b` weist `git checkout` an, einen neuen Branch zu erstellen, falls er noch nicht existiert, und dann zu diesem Branch zu wechseln.
    
- **Änderungen von einem Branch in einen anderen mergen:**
    
    ```
    git merge <Name-des-Branches-der-gemergt-wird>
    ```
    
    Der Befehl `git merge` wird verwendet, um Änderungen von einem Branch in einen anderen zu integrieren. Wenn ein Branch in einen anderen gemergt wird, werden die Änderungen vom Quell-Branch in den Ziel-Branch übernommen. Der Ziel-Branch ist der Branch, der gerade ausgecheckt ist, wenn der Befehl `git merge` aufgerufen wird. Git versucht, die Änderungen automatisch zu integrieren. In manchen Fällen kann es jedoch zu Konflikten kommen, wenn dieselben Zeilen in beiden Branches unterschiedlich geändert wurden. In diesem Fall musst du die Konflikte manuell beheben, bevor der Merge abgeschlossen werden kann.
    

### Remote-Repositories

- **Ein Remote-Repository hinzufügen:**
    
    ```
    git remote add origin <URL-des-Remote-Repositorys>
    ```
    
    Ein Remote-Repository ist eine Version des Projekts, die auf einem Server gehostet wird, auf den mehrere Benutzer zugreifen können. Mit dem Befehl `git remote add` wird eine Verbindung zu einem Remote-Repository hergestellt. Das Schlüsselwort `origin` ist ein Kurzname für das Remote-Repository und wird üblicherweise für das Haupt-Repository verwendet, von dem das Projekt ursprünglich geklont wurde. Du kannst aber auch einen anderen Namen vergeben. Die `<URL-des-Remote-Repositorys>` ist die Adresse des Remote-Repositorys. Diese URL kann entweder eine HTTPS-URL oder eine SSH-URL sein.
    
- **Alle Remote-Repositories anzeigen:**
    
    ```
    git remote -v
    ```
    
    Dieser Befehl listet alle Remote-Repositories auf, mit denen das lokale Repository verbunden ist. Für jedes Remote-Repository zeigt der Befehl seinen Namen und seine URL an. Die Option `-v` steht für "verbose" und bewirkt, dass der Befehl die URLs sowohl für das Holen als auch für das Pushen anzeigt.
    
- **Änderungen vom Remote-Repository herunterladen:**
    
    ```
    git pull origin <Name-des-Branches>
    git pull # Holt Änderungen vom Standard-Remote-Branch
    ```
    
    Der Befehl `git pull` wird verwendet, um Änderungen von einem Remote-Repository in das lokale Repository herunterzuladen und sie mit den lokalen Änderungen zusammenzuführen. Der Befehl erwartet den Namen des Remote-Repositorys (in der Regel `origin`) und den Namen des Branches, von dem die Änderungen heruntergeladen werden sollen. Wenn kein Branchname angegeben wird, ruft Git die Änderungen vom konfigurierten Upstream-Branch ab. Dies ist normalerweise der Branch, von dem der aktuelle Branch abzweigt. `git pull` ist im Grunde eine Abkürzung für die Ausführung von `git fetch` gefolgt von `git merge`. Zuerst lädt `git fetch` die Änderungen vom Remote-Repository herunter, und dann integriert `git merge` die heruntergeladenen Änderungen in den aktuellen Branch.
    
- **Lokale Commits zum Remote-Repository hochladen:**
    
    ```
    git push origin <Name-des-Branches>
    git push # Lädt Änderungen zum Standard-Remote-Branch hoch
    ```
    
    Der Befehl `git push` wird verwendet, um lokale Commits zu einem Remote-Repository hochzuladen. Der Befehl erwartet den Namen des Remote-Repositorys (in der Regel `origin`) und den Namen des Branches, in den die Commits hochgeladen werden sollen. Wenn kein Branchname angegeben wird, lädt Git die Commits in den konfigurierten Upstream-Branch hoch. `git push` lädt nur die Commits hoch, die sich im lokalen Repository befinden, aber nicht im Remote-Repository.
    
- **Änderungen im Remote-Repository überschreiben (nach einem Rebase):**
    
    ```
    git push origin <Name-des-Branches> --force-with-lease
    ```
    
    Nach einem Rebase wird die Historie des lokalen Branches geändert, wodurch er sich von der Historie des entsprechenden Branches im Remote-Repository unterscheidet. Ein normaler `git push`-Befehl würde in dieser Situation fehlschlagen, um dich davor zu bewahren, versehentlich Änderungen im Remote-Repository zu überschreiben. Die Option `--force-with-lease` ist eine sicherere Alternative zur Option `--force`. Anstatt den Remote-Branch blind zu überschreiben, überprüft `--force-with-lease`, ob der lokale Branch auf dem gleichen Stand wie der Remote-Branch ist, den er zu überschreiben versucht. Dies verhindert, dass du versehentlich Änderungen überschreibst, die von anderen vorgenommen wurden, während du an deinem Rebase gearbeitet hast.
    

### Rebase

- **Einen Branch auf einen anderen rebasen:**
    
    ```
    git rebase <Basis-Branch>
    ```
    
    Der Befehl `git rebase` wird verwendet, um die Basis eines Branches zu ändern. Beim Rebasen werden die Commits eines Branches genommen und so getan, als wären sie auf einem anderen Branch erstellt worden. Dies unterscheidet sich vom Mergen, bei dem die Änderungen von zwei Branches zusammengeführt werden, wobei ein neuer Merge-Commit entsteht. Beim Rebasen wird die Commit-Historie des aktuellen Branches geändert, wodurch sie linearer und übersichtlicher wird. Das Rebasen wird häufig verwendet, um einen Feature-Branch auf dem neuesten Stand des Haupt-Branches zu halten.
    
- **Einen interaktiven Rebase starten:**
    
    ```
    git rebase -i <Basis-Branch>
    ```
    
    Die Option `-i` steht für "interactive". Ein interaktiver Rebase ermöglicht es dir, die Commits auszuwählen, die neu angeordnet, bearbeitet, zusammengefasst oder entfernt werden sollen. Dies ist ein leistungsstarkes Werkzeug, um die Commit-Historie vor dem Mergen in einen Haupt-Branch zu bereinigen. Während eines interaktiven Rebase öffnet Git einen Editor, in dem du eine Liste von Commits und die Aktionen siehst, die für jeden Commit ausgeführt werden sollen.
    
- **Während eines Rebase: Den Rebase fortsetzen:**
    
    ```
    git rebase --continue
    ```
    
    Nachdem Konflikte während eines Rebase behoben wurden oder nachdem Commits im interaktiven Modus neu angeordnet, bearbeitet oder zusammengefasst wurden, wird der Befehl `git rebase --continue` verwendet, um den Rebase-Vorgang fortzusetzen. Dieser Befehl weist Git an, mit dem nächsten Commit im Rebase-Vorgang fortzufahren.
    
- **Während eines Rebase: Einen Commit überspringen:**
    
    ```
    git rebase --skip
    ```
    
    Der Befehl `git rebase --skip` wird verwendet, um einen Commit während eines Rebase zu überspringen. Das bedeutet, dass die Änderungen in diesem Commit nicht in den resultierenden Rebase aufgenommen werden. Dies kann nützlich sein, um einen Commit zu entfernen, der Fehler enthält oder nicht mehr benötigt wird. Es ist jedoch Vorsicht geboten, da das Überspringen von Commits zu Datenverlust führen kann.
    
- **Während eines Rebase: Den Rebase abbrechen:**
    
    ```
    git rebase --abort
    ```
    
    Der Befehl `git rebase --abort` wird verwendet, um den Rebase-Vorgang abzubrechen und das Repository in den Zustand vor dem Start des Rebase zurückzusetzen. Dies ist nützlich, wenn während des Rebase Fehler auftreten oder du entscheidest, dass du den Rebase nicht fortsetzen möchtest.
    

### SSH-Schlüssel für die Authentifizierung

- **Überprüfen, ob SSH-Schlüssel vorhanden sind:**
    
    ```
    ls -al ~/.ssh
    ```
    
    SSH-Schlüssel werden verwendet, um eine sichere Verbindung zu einem Remote-Server herzustellen, ohne dass ein Passwort eingegeben werden muss. Der Befehl `ls -al ~/.ssh` listet alle Dateien im Verzeichnis `.ssh` in deinem Home-Verzeichnis auf. In diesem Verzeichnis werden deine SSH-Schlüssel gespeichert. Wenn Dateien namens `id_rsa` oder `id_ed25519` vorhanden sind, bedeutet dies, dass du bereits SSH-Schlüssel erstellt hast.
    
- **Einen neuen EdDSA-SSH-Schlüssel erstellen (empfohlen):**
    
    ```
    ssh-keygen -t ed25519 -C "deine_email@beispiel.com"
    ```
    
    Dieser Befehl generiert ein neues Paar von SSH-Schlüsseln mit dem EdDSA-Algorithmus. EdDSA ist ein modernerer und sichererer Algorithmus als RSA. Die Option `-t` gibt den Typ des zu erstellenden Schlüssels an, und die Option `-C` gibt einen Kommentar an, der im öffentlichen Schlüssel gespeichert wird. Es wird dringend empfohlen, für die SSH-Schlüssel eine Passphrase festzulegen, um die Sicherheit zu erhöhen.
    
- **Einen neuen RSA-SSH-Schlüssel erstellen:**
    
    ```
    ssh-keygen -t rsa -b 4096 -C "deine_email@beispiel.com"
    ```
    
    Dieser Befehl generiert ein neues Paar von SSH-Schlüsseln mit dem RSA-Algorithmus. RSA ist ein älterer und weniger sicherer Algorithmus als EdDSA, wird aber von vielen Systemen immer noch unterstützt. Die Option `-t` gibt den Typ des zu erstellenden Schlüssels an, und die Option `-b` gibt die Anzahl der Bits im Schlüssel an. Ein höherer Wert für die Anzahl der Bits bedeutet einen sichereren Schlüssel, erfordert aber auch mehr Rechenleistung. Es wird dringend empfohlen, für die SSH-Schlüssel eine Passphrase festzulegen, um die Sicherheit zu erhöhen.
    
- **SSH-Verbindung zu GitHub testen:**
    
    ```
    ssh -T git@github.com
    ```
    
    Nachdem die SSH-Schlüssel erstellt und der öffentliche Schlüssel deinem GitHub-Konto hinzugefügt wurde, kann dieser Befehl verwendet werden, um zu überprüfen, ob die SSH-Verbindung zu GitHub ordnungsgemäß funktioniert. Die Option `-T` weist SSH an, keinen Pseudo-Terminal zuzuweisen. Wenn die Verbindung erfolgreich ist, zeigt GitHub eine Meldung an, die deinen Benutzernamen bestätigt.
    
- **Die Remote-URL auf SSH ändern:**
    
    ```
    git remote set-url origin git@github.com:DeinUsername/Projekt-A.git
    ```
    
    Standardmäßig verwenden Git-Remote-URLs das HTTPS-Protokoll. Das bedeutet, dass du bei jeder Interaktion mit dem Remote-Repository, z. B. beim Pushen oder Pullen von Änderungen, zur Eingabe deines Benutzernamens und Passworts aufgefordert wirst. Durch die Verwendung von SSH-Schlüsseln kann dieser Authentifizierungsprozess automatisiert werden, sodass du dein Passwort nicht jedes Mal eingeben musst. Um die Remote-URL von HTTPS auf SSH zu ändern, musst du den Befehl `git remote set-url` mit der SSH-URL des Repositorys verwenden.
    

### Nützliche Tipps

- **Versteckte Dateien anzeigen:** In den meisten Dateimanagern kannst du versteckte Dateien anzeigen, indem du die Tastenkombination `Strg + H` drückst. Das ist wichtig, weil Git-Repositorys einen versteckten Ordner `.git` im Stammverzeichnis des Projekts enthalten.
    
- **Terminalbefehle abbrechen:** Es kann vorkommen, dass ein laufender Terminalbefehl durch Drücken von `Strg + C` abgebrochen werden muss. Das ist besonders nützlich, wenn ein Befehl unerwartet lange dauert oder eine Endlosschleife auslöst.
    
- **Den Standard-Editor verlassen:**
    
    - **Nano:** Drück `Strg + X`, dann `J`, dann `Enter`. Wenn Änderungen vorhanden sind, fordert dich Nano auf, diese zu speichern, bevor es beendet wird.
        
    - **Vi/Vim:** Drück `Esc`, dann gib `:wq` ein und drück `Enter`. Der Befehl `:wq` weist Vim an, die Änderungen zu schreiben und das Programm zu beenden. Es gibt auch andere Befehle, um Vim zu beenden, z. B. `:q` zum Beenden ohne Speichern von Änderungen und `:q!` zum Beenden ohne Speichern von Änderungen zu erzwingen.
        
- **Lokale Änderungen sichern:** Es wird empfohlen, deinen Projektordner an einem sicheren Ort zu speichern, bevor potenziell riskante Git-Operationen durchgeführt werden. Zu diesen Operationen gehören z. B. das Rebasen, das Ändern der Commit-Historie oder das erzwungene Pushen von Änderungen in ein Remote-Repository. Wenn du deine Änderungen sicherst, kannst du sie bei einem Fehler oder einem unerwarteten Ergebnis wiederherstellen.
### Grundlegende Konfiguration

- **Benutzernamen und E-Mail-Adresse festlegen:**
    
    ```
    git config --global user.name "Dein Name"
    git config --global user.email "deine_email@beispiel.com"
    ```
    
    Yo, mit diesen Befehlen legst du deinen Namen und deine E-Mail-Adresse für alle deine Git-Dinger fest. Mega wichtig, weil Git das braucht, um deine Commits zuzuordnen. Ohne diese Infos geht gar nix! Dein Name und deine E-Mail-Adresse werden im Commit-Verlauf gespeichert und sind für jeden sichtbar, der das Repository checkt. Die Option `--global` sorgt dafür, dass diese Einstellungen nicht nur für das aktuelle Repository gelten, sondern für alle Git-Repositories auf deinem System. Du kannst die Einstellungen auch nur für ein einzelnes Repository festlegen, indem du `--local` anstelle von `--global` benutzt. Dann gelten die Einstellungen nur für das Repository im aktuellen Ordner.
    

### Ein Repository starten

- **Ein neues Repository in einem Verzeichnis erstellen:**
    
    ```
    git init
    ```
    
    Dieser Befehl initialisiert ein neues, leeres Git-Repository im angegebenen Ordner. Ein Git-Repository ist einfach ein Ordner, der alle deine Dateien und die Historie aller Änderungen an diesen Dateien enthält. Durch die Initialisierung eines Repositorys wird ein versteckter Ordner namens `.git` im angegebenen Ordner erstellt. Dieser Ordner enthält alle Metadaten, die Git für die Verwaltung des Repositorys benötigt. Wichtig: `git init` muss in dem Ordner ausgeführt werden, der als Wurzelverzeichnis deines Projekts dienen soll. Wenn du dich bereits in einem Unterordner befindest, wird das Repository in diesem Unterordner erstellt, was wahrscheinlich nicht das ist, was du willst.
    
- **Ein Repository von einem Remote-Server klonen:**
    
    ```
    git clone <URL-des-Repositorys>
    ```
    
    Dieser Befehl wird verwendet, um eine Kopie eines bestehenden Git-Repositorys von einem Remote-Server auf deinen Rechner zu erstellen. Ein Remote-Server ist ein Server, auf dem das Repository gehostet wird, z. B. GitHub, GitLab oder Bitbucket. Durch das Klonen eines Repositorys wird nicht nur die neueste Version der Dateien heruntergeladen, sondern auch die gesamte Historie aller Änderungen an diesen Dateien. Dadurch kannst du die Historie des Projekts einsehen und bei Bedarf zu früheren Versionen zurückkehren. Der Befehl `git clone` erstellt einen neuen Ordner mit dem Namen des Repositorys und lädt die Dateien und die Historie in diesen Ordner herunter.
    

### Änderungen speichern

- **Den Status deines Arbeitsverzeichnisses anzeigen:**
    
    ```
    git status
    ```
    
    Der Befehl `git status` gibt dir einen Überblick über den aktuellen Zustand deines Arbeitsverzeichnisses. Das Arbeitsverzeichnis ist der Ordner auf deinem Rechner, in dem sich die Dateien deines Projekts befinden. `git status` zeigt dir, welche Dateien geändert wurden, welche Dateien sich im Staging-Bereich befinden und welche Dateien nicht von Git verfolgt werden. Diese Infos sind mega wichtig, um den Überblick über deine Änderungen zu behalten und zu entscheiden, welche Änderungen in den nächsten Commit aufgenommen werden sollen. Die Ausgabe von `git status` ist in verschiedene Abschnitte unterteilt, die dir unterschiedliche Infos anzeigen. Zum Beispiel werden Dateien, die geändert, aber noch nicht zum Staging-Bereich hinzugefügt wurden, unter der Überschrift "Changes not staged for commit" aufgelistet, während Dateien im Staging-Bereich unter "Changes to be committed" angezeigt werden.
    
- **Eine oder mehrere Dateien zum Staging-Bereich hinzufügen:**
    
    ```
    git add <Datei1> <Datei2> ...
    git add . # Alle Änderungen im aktuellen Verzeichnis hinzufügen
    ```
    
    Der Befehl `git add` wird verwendet, um Änderungen an Dateien zum Staging-Bereich hinzuzufügen. Der Staging-Bereich ist ein Bereich, in dem Änderungen für den nächsten Commit vorbereitet werden. Wenn du eine Datei änderst, bedeutet das nicht automatisch, dass diese Änderung im nächsten Commit enthalten ist. Stattdessen musst du die Änderung explizit mit `git add` zum Staging-Bereich hinzufügen. Dadurch kannst du auswählen, welche Änderungen in einen Commit einfließen sollen, und Commits in logische Einheiten unterteilen. Du kannst `git add` mit einem oder mehreren Dateinamen als Argumente aufrufen, um bestimmte Dateien zum Staging-Bereich hinzuzufügen. Alternativ kannst du den Befehl mit einem Punkt `.` als Argument aufrufen, um alle Änderungen im aktuellen Verzeichnis und seinen Unterverzeichnissen zum Staging-Bereich hinzuzufügen. Sei vorsichtig mit dem Punkt, um nicht versehentlich Änderungen hinzuzufügen, die nicht in den nächsten Commit aufgenommen werden sollen.
    
- **Änderungen im Staging-Bereich anzeigen:**
    
    ```
    git diff --staged
    ```
    
    Der Befehl `git diff` wird verwendet, um die Unterschiede zwischen verschiedenen Versionen einer Datei anzuzeigen. In seiner einfachsten Form zeigt `git diff` die Unterschiede zwischen dem Arbeitsverzeichnis und dem Staging-Bereich an. Mit der Option `--staged` kannst du den Befehl jedoch so ändern, dass er stattdessen die Unterschiede zwischen dem Staging-Bereich und dem letzten Commit anzeigt. Das ist nützlich, um genau zu sehen, welche Änderungen im nächsten Commit enthalten sein werden. Die Ausgabe von `git diff` wird im sogenannten "Diff-Format" angezeigt, das die hinzugefügten, entfernten und geänderten Zeilen hervorhebt.
    
- **Einen Commit mit den Änderungen im Staging-Bereich erstellen:**
    
    ```
    git commit -m "Beschreibung der Änderungen"
    ```
    
    Der Befehl `git commit` wird verwendet, um einen neuen Commit zu erstellen, der die Änderungen im Staging-Bereich enthält. Ein Commit ist ein Schnappschuss des Repositorys zu einem bestimmten Zeitpunkt. Jeder Commit enthält eine eindeutige ID, einen Zeitstempel, den Autor des Commits und eine optionale Commit-Nachricht. Die Commit-Nachricht ist eine kurze Beschreibung der im Commit enthaltenen Änderungen. Sie ist mega wichtig, um die Historie des Projekts zu dokumentieren und es anderen (und dir selbst in der Zukunft) zu ermöglichen, die vorgenommenen Änderungen zu verstehen. Die Option `-m` wird verwendet, um die Commit-Nachricht direkt auf der Befehlszeile anzugeben. Wenn du diese Option nicht verwendest, öffnet Git einen Texteditor, in dem du die Commit-Nachricht eingeben kannst. Es wird dringend empfohlen, aussagekräftige Commit-Nachrichten zu verfassen, die die Änderungen präzise und prägnant beschreiben.
    

### Branches und Merges

- **Alle lokalen Branches anzeigen:**
    
    ```
    git branch
    ```
    
    Ein Branch in Git stellt eine unabhängige Entwicklungslinie dar. Branches ermöglichen es dir, an neuen Features zu arbeiten, Fehler zu beheben oder Experimente durchzuführen, ohne die Hauptentwicklungslinie zu beeinträchtigen. Der Befehl `git branch` listet alle lokalen Branches im Repository auf. Der aktuell aktive Branch ist durch ein Sternchen `*` gekennzeichnet. Lokale Branches sind Branches, die nur auf deinem Rechner existieren. Es gibt auch Remote-Branches, die auf einem Remote-Server existieren.
    
- **Einen neuen Branch erstellen:**
    
    ```
    git branch <Name-des-Branches>
    ```
    
    Mit diesem Befehl erstellst du einen neuen Branch mit dem angegebenen Namen. Der neue Branch wird von dem Commit erstellt, auf dem du dich gerade befindest. Das bedeutet, dass der neue Branch zunächst genau den gleichen Inhalt wie der aktuelle Branch hat. Nachdem der Branch erstellt wurde, kannst du mit dem Befehl `git checkout` zu ihm wechseln und mit der Arbeit an dem neuen Branch beginnen.
    
- **Zu einem anderen Branch wechseln:**
    
    ```
    git checkout <Name-des-Branches>
    ```
    
    Der Befehl `git checkout` wird verwendet, um zu einem anderen Branch zu wechseln oder einen Commit auszuchecken. Wenn ein Branch ausgecheckt wird, wird das Arbeitsverzeichnis so aktualisiert, dass es den Dateien und der Historie dieses Branches entspricht. Alle neuen Commits werden dann zu diesem Branch hinzugefügt. Wenn ein Commit anstelle eines Branches ausgecheckt wird, wird das Arbeitsverzeichnis in den Zustand dieses Commits versetzt, und alle neuen Commits würden von diesem Commit abzweigen, wodurch ein "detached HEAD"-Zustand entsteht, der für die meisten Benutzer nicht erwünscht ist.
    
- **Einen Branch erstellen und sofort zu ihm wechseln:**
    
    ```
    git checkout -b <Name-des-Branches>
    ```
    
    Dieser Befehl ist eine Abkürzung für die Erstellung eines neuen Branches mit `git branch` und das anschließende Wechseln zu diesem Branch mit `git checkout`. Die Option `-b` weist `git checkout` an, einen neuen Branch zu erstellen, falls er noch nicht existiert, und dann zu diesem Branch zu wechseln.
    
- **Änderungen von einem Branch in einen anderen mergen:**
    
    ```
    git merge <Name-des-Branches-der-gemergt-wird>
    ```
    
    Der Befehl `git merge` wird verwendet, um Änderungen von einem Branch in einen anderen zu integrieren. Wenn ein Branch in einen anderen gemergt wird, werden die Änderungen vom Quell-Branch in den Ziel-Branch übernommen. Der Ziel-Branch ist der Branch, der gerade ausgecheckt ist, wenn der Befehl `git merge` aufgerufen wird. Git versucht, die Änderungen automatisch zu integrieren. In manchen Fällen kann es jedoch zu Konflikten kommen, wenn dieselben Zeilen in beiden Branches unterschiedlich geändert wurden. In diesem Fall musst du die Konflikte manuell beheben, bevor der Merge abgeschlossen werden kann.
    

### Remote-Repositories

- **Ein Remote-Repository hinzufügen:**
    
    ```
    git remote add origin <URL-des-Remote-Repositorys>
    ```
    
    Ein Remote-Repository ist eine Version des Projekts, die auf einem Server gehostet wird, auf den von mehreren Benutzern zugegriffen werden kann. Mit dem Befehl `git remote add` wird eine Verbindung zu einem Remote-Repository hergestellt. Das Schlüsselwort `origin` ist ein Kurzname für das Remote-Repository und wird üblicherweise für das Haupt-Repository verwendet, von dem das Projekt ursprünglich geklont wurde. Du kannst aber auch einen anderen Namen vergeben. Die `<URL-des-Remote-Repositorys>` ist die Adresse des Remote-Repositorys. Diese URL kann entweder eine HTTPS-URL oder eine SSH-URL sein.
    
- **Alle Remote-Repositories anzeigen:**
    
    ```
    git remote -v
    ```
    
    Dieser Befehl listet alle Remote-Repositories auf, mit denen das lokale Repository verbunden ist. Für jedes Remote-Repository zeigt der Befehl seinen Namen und seine URL an. Die Option `-v` steht für "verbose" und bewirkt, dass der Befehl die URLs sowohl für das Holen als auch für das Pushen anzeigt.
    
- **Änderungen vom Remote-Repository herunterladen:**
    
    ```
    git pull origin <Name-des-Branches>
    git pull # Holt Änderungen vom Standard-Remote-Branch
    ```
    
    Der Befehl `git pull` wird verwendet, um Änderungen von einem Remote-Repository in das lokale Repository herunterzuladen und sie mit den lokalen Änderungen zusammenzuführen. Der Befehl erwartet den Namen des Remote-Repositorys (in der Regel `origin`) und den Namen des Branches, von dem die Änderungen heruntergeladen werden sollen. Wenn kein Branchname angegeben wird, ruft Git die Änderungen vom konfigurierten Upstream-Branch ab. Dies ist in der Regel der Branch, von dem der aktuelle Branch abgezweigt ist. `git pull` ist im Wesentlichen eine Abkürzung für die Ausführung von `git fetch` gefolgt von `git merge`. Zuerst lädt `git fetch` die Änderungen vom Remote-Repository herunter, und dann integriert `git merge` die heruntergeladenen Änderungen in den aktuellen Branch.
    
- **Lokale Commits zum Remote-Repository hochladen:**
    
    ```
    git push origin <Name-des-Branches>
    git push # Lädt Änderungen zum Standard-Remote-Branch hoch
    ```
    
    Der Befehl `git push` wird verwendet, um lokale Commits zu einem Remote-Repository hochzuladen. Der Befehl erwartet den Namen des Remote-Repositorys (in der Regel `origin`) und den Namen des Branches, in den die Commits hochgeladen werden sollen. Wenn kein Branchname angegeben wird, lädt Git die Commits in den konfigurierten Upstream-Branch hoch. `git push` lädt nur die Commits hoch, die sich im lokalen Repository befinden, aber nicht im Remote-Repository.
    
- **Änderungen im Remote-Repository überschreiben (nach einem Rebase):**
    
    ```
    git push origin <Name-des-Branches> --force-with-lease
    ```
    
    Nach einem Rebase wird die Historie des lokalen Branches geändert, wodurch er sich von der Historie des entsprechenden Branches im Remote-Repository unterscheidet. Ein normaler `git push`-Befehl würde in dieser Situation fehlschlagen, um dich davor zu bewahren, versehentlich Änderungen im Remote-Repository zu überschreiben. Die Option `--force-with-lease` ist eine sBLicherere Alternative zur Option `--force`. Anstatt den Remote-Branch blind zu überschreiben, überprüft `--force-with-lease`, ob der lokale Branch auf dem gleichen Stand wie der Remote-Branch ist, den er zu überschreiben versucht. Dies verhindert, dass du versehentlich Änderungen überschreibst, die von anderen vorgenommen wurden, während du an deinem Rebase gearbeitet hast.
    

### Rebase

- **Einen Branch auf einen anderen rebasen:**
    
    ```
    git rebase <Basis-Branch>
    ```
    
    Der Befehl `git rebase` wird verwendet, um die Basis eines Branches zu ändern. Beim Rebasen werden die Commits eines Branches genommen und so getan, als wären sie auf einem anderen Branch erstellt worden. Dies unterscheidet sich vom Mergen, bei dem die Änderungen von zwei Branches zusammengeführt werden, wobei ein neuer Merge-Commit entsteht. Beim Rebasen wird die Commit-Historie des aktuellen Branches geändert, wodurch sie linearer und übersichtlicher wird. Das Rebasen wird häufig verwendet, um einen Feature-Branch auf dem neuesten Stand des Haupt-Branches zu halten.
    
- **Einen interaktiven Rebase starten:**
    
    ```
    git rebase -i <Basis-Branch>
    ```
    
    Die Option `-i` steht für "interactive". Ein interaktiver Rebase ermöglicht es dir, die Commits auszuwählen, die neu angeordnet, bearbeitet, zusammengefasst oder entfernt werden sollen. Dies ist ein leistungsstarkes Werkzeug, um die Commit-Historie vor dem Mergen in einen Haupt-Branch zu bereinigen. Während eines interaktiven Rebase öffnet Git einen Editor, in dem du eine Liste von Commits und die Aktionen siehst, die für jeden Commit ausgeführt werden sollen.
    
- **Während eines Rebase: Den Rebase fortsetzen:**
    
    ```
    git rebase --continue
    ```
    
    Nachdem Konflikte während eines Rebase behoben wurden oder nachdem Commits im interaktiven Modus neu angeordnet, bearbeitet oder zusammengefasst wurden, wird der Befehl `git rebase --continue` verwendet, um den Rebase-Vorgang fortzusetzen. Dieser Befehl weist Git an, mit dem nächsten Commit im Rebase-Vorgang fortzufahren.
    
- **Während eines Rebase: Einen Commit überspringen:**
    
    ```
    git rebase --skip
    ```
    
    Der Befehl `git rebase --skip` wird verwendet, um einen Commit während eines Rebase zu überspringen. Dies bedeutet, dass die Änderungen in diesem Commit nicht in den resultierenden Rebase aufgenommen werden. Dies kann nützlich sein, um einen Commit zu entfernen, der Fehler enthält oder nicht mehr benötigt wird. Es ist jedoch Vorsicht geboten, da das Überspringen von Commits zu Datenverlust führen kann.
    
- **Während eines Rebase: Den Rebase abbrechen:**
    
    ```
    git rebase --abort
    ```
    
    Der Befehl `git rebase --abort` wird verwendet, um den Rebase-Vorgang abzubrechen und das Repository in den Zustand vor dem Start des Rebase zurückzusetzen. Dies ist nützlich, wenn während des Rebase Fehler auftreten oder du entscheidest, dass du den Rebase nicht fortsetzen möchtest.
    

### SSH-Schlüssel für die Authentifizierung

- **Überprüfen, ob SSH-Schlüssel vorhanden sind:**
    
    ```
    ls -al ~/.ssh
    ```
    
    SSH-Schlüssel werden verwendet, um eine sichere Verbindung zu einem Remote-Server herzustellen, ohne dass ein Passwort eingegeben werden muss. Der Befehl `ls -al ~/.ssh` listet alle Dateien im Verzeichnis `.ssh` in deinem Home-Verzeichnis auf. In diesem Verzeichnis werden deine SSH-Schlüssel gespeichert. Wenn Dateien namens `id_rsa` oder `id_ed25519` vorhanden sind, bedeutet dies, dass du bereits SSH-Schlüssel erstellt hast.
    
- **Einen neuen EdDSA-SSH-Schlüssel erstellen (empfohlen):**
    
    ```
    ssh-keygen -t ed25519 -C "deine_email@beispiel.com"
    ```
    
    Dieser Befehl generiert ein neues Paar von SSH-Schlüsseln mit dem EdDSA-Algorithmus. EdDSA ist ein modernerer und sichererer Algorithmus als RSA. Die Option `-t` gibt den Typ des zu erstellenden Schlüssels an, und die Option `-C` gibt einen Kommentar an, der im öffentlichen Schlüssel gespeichert wird. Es wird dringend empfohlen, für die SSH-Schlüssel eine Passphrase festzulegen, um die Sicherheit zu erhöhen.
    
- **Einen neuen RSA-SSH-Schlüssel erstellen:**
    
    ```
    ssh-keygen -t rsa -b 4096 -C "deine_email@beispiel.com"
    ```
    
    Dieser Befehl generiert ein neues Paar von SSH-Schlüsseln mit dem RSA-Algorithmus. RSA ist ein älterer und weniger sicherer Algorithmus als EdDSA, wird aber von vielen Systemen immer noch unterstützt. Die Option `-t` gibt den Typ des zu erstellenden Schlüssels an, und die Option `-b` gibt die Anzahl der Bits im Schlüssel an. Ein höherer Wert für die Anzahl der Bits bedeutet einen sichereren Schlüssel, erfordert aber auch mehr Rechenleistung. Es wird dringend empfohlen, für die SSH-Schlüssel eine Passphrase festzulegen, um die Sicherheit zu erhöhen.
    
- **SSH-Verbindung zu GitHub testen:**
    
    ```
    ssh -T git@github.com
    ```
    
    Nachdem die SSH-Schlüssel erstellt und der öffentliche Schlüssel deinem GitHub-Konto hinzugefügt wurde, kann dieser Befehl verwendet werden, um zu überprüfen, ob die SSH-Verbindung zu GitHub ordnungsgemäß funktioniert. Die Option `-T` weist SSH an, keinen Pseudo-Terminal zuzuweisen. Wenn die Verbindung erfolgreich ist, zeigt GitHub eine Meldung an, die deinen Benutzernamen bestätigt.
    
- **Die Remote-URL auf SSH ändern:**
    
    ```
    git remote set-url origin git@github.com:DeinUsername/Projekt-A.git
    ```
    
    Standardmäßig verwenden Git-Remote-URLs das HTTPS-Protokoll. Dies bedeutet, dass du bei jeder Interaktion mit dem Remote-Repository, z. B. beim Pushen oder Pullen von Änderungen, zur Eingabe deines Benutzernamens und Passworts aufgefordert wirst. Durch die Verwendung von SSH-Schlüsseln kann dieser Authentifizierungsprozess automatisiert werden, sodass du dein Passwort nicht jedes Mal eingeben musst. Um die Remote-URL von HTTPS auf SSH zu ändern, musst du den Befehl `git remote set-url` mit der SSH-URL des Repositorys verwenden.
    

### Nützliche Tipps

- **Versteckte Dateien anzeigen:** In den meisten Dateimanagern kannst du versteckte Dateien anzeigen, indem du die Tastenkombination `Strg + H` drückst. Das ist wichtig, weil Git-Repositorys einen versteckten Ordner `.git` im Stammverzeichnis des Projekts enthalten.
    
- **Terminalbefehle abbrechen:** Es kann vorkommen, dass ein laufender Terminalbefehl durch Drücken von `Strg + C` abgebrochen werden muss. Das ist besonders nützlich, wenn ein Befehl unerwartet lange dauert oder eine Endlosschleife auslöst.
    
- **Den Standard-Editor verlassen:**
    
    - **Nano:** Drück `Strg + X`, dann `J`, dann `Enter`. Wenn Änderungen vorhanden sind, fordert dich Nano auf, diese zu speichern, bevor es beendet wird.
        
    - **Vi/Vim:** Drück `Esc`, dann gib `:wq` ein und drück `Enter`. Der Befehl `:wq` weist Vim an, die Änderungen zu schreiben und das Programm zu beenden. Es gibt auch andere Befehle, um Vim zu beenden, z. B. `:q` zum Beenden ohne Speichern von Änderungen und `:q!` zum Beenden ohne Speichern von Änderungen zu erzwingen.
        
- **Lokale Änderungen sichern:** Es wird empfohlen, deinen Projektordner an einem sicheren Ort zu speichern, bevor potenziell riskante Git-Operationen durchgeführt werden. Zu diesen Operationen gehören z. B. das Rebasen, das Ändern der Commit-Historie oder das erzwungene Pushen von Änderungen in ein Remote-Repository. Durch das Sichern deiner Änderungen stellst du sicher, dass du bei einem Fehler oder unerwarteten Ergebnis die Änderungen wiederherstellen kannst.

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