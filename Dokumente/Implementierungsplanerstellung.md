
**Präambel: Verständnis von "Lückenlosigkeit" für die Ausführung durch KI-Agenten**

Die erfolgreiche Konzeption und Implementierung einer modularen Desktop-Umgebung durch einen autonomen KI-Programmieragenten hängt entscheidend von der Qualität und Vollständigkeit des zugrundeliegenden Implementierungsplans ab. Der Begriff "lückenlos" (gapless) definiert in diesem Kontext einen Plan, der keine Interpretationsspielräume für den KI-Agenten lässt und eine direkte, Zeile-für-Zeile-Umsetzung ermöglicht.

- Definition von "Lückenlos" für einen KI-Agenten:
    
    Ein als "lückenlos" definierter Plan auf der Ebene einer einzelnen Funktion oder Aufgabe muss zwingend die folgenden Elemente umfassen:
    
    - **Genaue Funktions-/Aufgabensignatur:** Eindeutiger Name, detaillierte Parameter (mit Typen und Namen) und ein klar definierter Rückgabetyp.
    - **Detaillierter, schrittweiser Algorithmus:** Formuliert als präziser Pseudocode oder eine klare, unzweideutige nummerierte Abfolge von Schritten. Jeder Schritt muss eine atomare Operation darstellen, die der KI-Agent direkt in Code umsetzen kann.
    - **Spezifikation aller Abhängigkeiten:** Eine vollständige Auflistung aller anderen Funktionen, Module oder externen APIs, die von der aktuellen Funktion/Aufgabe aufgerufen werden. Dies schließt die genauen Signaturen der aufgerufenen Abhängigkeiten ein.
    - **Vollständige Definition der Fehlerbehandlung:** Für jeden identifizierten und potenziellen Fehlerzustand muss ein exakter Fehlerbehandlungspfad spezifiziert sein. Dies beinhaltet die Erkennung des Fehlers, die zu ergreifenden Maßnahmen (z.B. Protokollierung, Ressourcenfreigabe, Rückgabe eines spezifischen Fehlercodes) und den Zustand des Systems nach der Fehlerbehandlung.
    - **Definition der verwendeten Datenstrukturen:** Präzise Beschreibung aller internen und externen Datenstrukturen, die von der Funktion/Aufgabe verwendet oder manipuliert werden, inklusive ihrer Felder, Typen und Speicherlayouts, falls relevant.
    - **Klare Testfälle:** Für jede Funktion müssen eindeutige Testfälle mit exakt definierten Eingaben und den erwarteten Ausgaben oder Systemzuständen nach der Ausführung spezifiziert werden. Diese Testfälle dienen dem KI-Agenten nicht nur zur Verifikation der Implementierung, sondern auch als konkrete Beispiele für die erwartete Funktionsweise.
    
    Die Notwendigkeit solch detaillierter Spezifikationen ergibt sich aus den Herausforderungen bei der KI-gesteuerten Softwareentwicklung, bei der unvollständige oder mehrdeutige Vorgaben zu fehlerhaften oder suboptimalen Ergebnissen führen können.1 Ein strukturierter Ansatz, wie hier definiert, ist eine Grundvoraussetzung für die erfolgreiche Codegenerierung durch KI-Systeme.3
    
- Rolle von Formalen Methoden und Ausführbaren Spezifikationen:
    
    Um die geforderte "Lückenlosigkeit" und direkte maschinelle Interpretierbarkeit zu gewährleisten, werden Prinzipien aus formalen Methoden und ausführbaren Spezifikationen adaptiert. Formale Methoden betonen die präzise Spezifikation von Systemeigenschaften, Vor- und Nachbedingungen für Operationen sowie Invarianten.4 Obwohl eine vollständige formale Verifikation des gesamten Plans den Rahmen sprengen könnte, sind die Prinzipien der präzisen und unzweideutigen Spezifikation direkt auf die Erstellung "lückenloser" Anweisungen für den KI-Agenten anwendbar. So wie mathematische Beweise in der formalen Verifikation dazu dienen, die Korrektheit eines Designs gemäß seiner Spezifikationen zu validieren 4, so dient die Detailtiefe des Plans dazu, die Korrektheit der KI-generierten Implementierung sicherzustellen.
    
    Ausführbare Spezifikationen, bei denen Anforderungen als automatisierte Tests formuliert werden 5, finden ihre Entsprechung in der Forderung nach "klaren Testfällen mit definierten Eingaben und erwarteten Ausgaben/Zuständen". Der Testtitel wird zur Anforderungsbeschreibung und die Testschritte beschreiben die Funktionalität.5
    
    Der hier entstehende "lückenlose" Implementierungsplan kann selbst als eine Form von extrem detaillierter, vorverifizierter ausführbarer Spezifikation betrachtet werden. Der KI-Programmieragent fungiert dabei als die "Ausführungs-Engine" für diese Spezifikationen. Diese Perspektive ergibt sich aus mehreren Überlegungen: Erstens umfasst die Definition von "lückenlos" Elemente wie detaillierte Algorithmen und Testfälle, was Kernbestandteile ausführbarer Spezifikationen sind.5 Zweitens ist der KI-Agent darauf ausgelegt, den Plan Zeile für Zeile umzusetzen. Dies impliziert, dass der Plan eine Präzision und Eindeutigkeit aufweisen muss, die vergleichbar ist mit bereits verifiziertem Code, wodurch er für die KI "ausführbar" wird. Folglich muss die Recherchephase nicht nur definieren, _was_ zu tun ist, sondern auch _wie_ es mit ausreichender Genauigkeit zu tun ist, um Fehlinterpretationen durch den KI-Agenten auszuschließen.
    

**Teil 1: Kern-Schicht (Core Layer) – Abstraktion von System- und Kernel-APIs**

**1.1. Einführung in die Kern-Schicht**

- **Zweck:** Die Kern-Schicht (Core Layer) bildet das Fundament der modularen Desktop-Umgebung. Ihr primärer Zweck ist die Bereitstellung einer konsistenten und abstrahierten Schnittstelle zu den zugrundeliegenden Kernel- und System-Level-APIs des Zielbetriebssystems. Diese Abstraktion ist entscheidend, um höhere Schichten der Desktop-Umgebung von plattformspezifischen Details zu entkoppeln und somit Portabilität und Modularität zu gewährleisten.
- **Hauptverantwortlichkeiten:**
    - Direkte Interaktion mit den Primitiven des Betriebssystemkernels.
    - Abstraktion von hardwarenahen Funktionalitäten, die über das Betriebssystem zugänglich sind.
    - Bereitstellung von Basisdiensten für Prozessmanagement, Speicherverwaltung, Dateisystemoperationen, Interprozesskommunikation (IPC) und Geräteinteraktion.
- **Design-Philosophie:**
    - Strikte Einhaltung des Prinzips der geringsten Rechte (Principle of Least Privilege): Die Kern-Schicht stellt nur die absolut notwendigen Abstraktionen zur Verfügung.
    - Minimierung des Overheads: Die Abstraktionen sollen die Systemleistung nicht unnötig beeinträchtigen.
    - Anpassungsfähigkeit: Das Design wird maßgeblich durch das gewählte Zielbetriebssystem und die primäre Programmiersprache beeinflusst. Die hier präsentierten Spezifikationen müssen entsprechend adaptiert werden, sobald diese fundamentalen Projektparameter definiert sind. Die Notwendigkeit einer solchen Betriebssystem-Abstraktionsschicht (OSAL) oder Hardware-Abstraktionsschicht (HAL) zur Erzielung von Portabilität und zur Handhabung von Hardware-Interaktionen ist in der Softwaretechnik etabliert.7 Der Kernel selbst spielt die zentrale Rolle bei der Verwaltung von Ressourcen und der Bereitstellung von Schnittstellen.9

**1.2. Zentrale Design-Prinzipien für Submodule der Kern-Schicht**

- **Schnittstellendesign:** Die APIs der Kern-Schicht werden auf Klarheit, minimale notwendige Funktionalität und einfache Nutzbarkeit durch die Domänen-Schicht hin optimiert. Entwurfsmuster wie Facade 11 und Adapter 13 werden in Betracht gezogen, um komplexe Kernel-Interaktionen zu vereinfachen oder bestehende Betriebssystemschnittstellen anzupassen. Das Bridge-Muster 15 könnte relevant werden, falls für eine Abstraktion mehrere Implementierungsstrategien vorgesehen sind. Die Verwendung etablierter Entwurfsmuster ist für die Strukturierung von Abstraktionsschichten unerlässlich.17
- **Fehlerbehandlungsstrategie:** Ein konsistenter Mechanismus zur Fehlerberichterstattung ist von größter Bedeutung. Dies beinhaltet die Definition von standardisierten Fehlercodes (z.B. Integer-Return-Codes oder spezifische Fehlerstrukturen) für die Kern-Schicht. Native Betriebssystem-Fehlercodes (wie POSIX `errno` 18) werden auf die in der Kern-Schicht definierten Fehlercodes abgebildet. Fehlerpropagationspfade müssen detailliert spezifiziert werden. Eine umfassende Fehlerbehandlung ist ein kritischer Aspekt für die Robustheit der KI-generierten Software.20 Die Wahl der Fehlerbehandlungsmethodik (z.B. Rückgabewerte versus Ausnahmen, falls die Zielsprache dies unterstützt) in der Kern-Schicht ist eine fundamentale Architekturentscheidung. Da die Kern-Schicht die unterste Softwareebene der neuen Umgebung darstellt und höhere Schichten (Domäne, System, UI) ihre APIs konsumieren, wird eine hier getroffene Entscheidung die Fehlerbehandlungsphilosophie des gesamten Systems prägen. Eine inkonsistente Fehlerbehandlung zwischen den Schichten würde die Aufgabe des KI-Agenten, korrekten und robusten Code zu schreiben, erheblich erschweren. Daher ist eine klare, einheitliche Strategie, die auf dieser grundlegenden Ebene festgelegt wird, entscheidend für die "Lückenlosigkeit" und die Implementierbarkeit durch den KI-Agenten.
- **Ressourcenmanagement:** Jedes Submodul, das Ressourcen vom Betriebssystem anfordert (Speicher, Handles, Deskriptoren etc.), muss für deren explizite und korrekte Freigabe verantwortlich sein, um Lecks zu vermeiden.
- **Nebenläufigkeitsaspekte:** Sofern die zugrundeliegenden Betriebssystemaufrufe nicht inhärent threadsicher sind oder die Kern-Schicht selbst gemeinsam genutzte Ressourcen verwaltet, müssen geeignete Synchronisationsprimitive bereitgestellt und deren Verwendung spezifiziert werden.

**1.3. Submodul: System Call Interface (SCI) Abstraktion**

- **Verantwortlichkeit:** Dieses Submodul stellt eine typisierte und abstrahierte Schnittstelle über den rohen Systemaufrufmechanismus des Zielbetriebssystems bereit. Dies kann das Kapseln von Systemaufrufnummern, die Verwaltung der Argumentübergabe und die Handhabung von Rückgabewertkonventionen umfassen. Die Linux-SCI beispielsweise ist eine dünne Schicht, die Funktionsaufrufe vom User Space in den Kernel ermöglicht.23 Betriebssysteme kategorisieren Systemaufrufe typischerweise in Bereiche wie Prozesssteuerung, Dateimanagement usw..24
- **Schlüsselfunktionen (Beispiele):**
    - `Kern_InvokeSystemCall(call_id, arg1, arg2,..., &result_status)`
        - `call_id`: Eine Enumeration oder ein definierter konstanter Wert, der den spezifischen Systemaufruf identifiziert.
        - `arg1, arg2,...`: Die an den Systemaufruf zu übergebenden Argumente, deren Typen exakt definiert sein müssen.
        - `&result_status`: Ein Zeiger auf eine Struktur oder einen Basistyp zur Aufnahme des Ergebnisses und/oder des Fehlerstatus des Systemaufrufs.
- **Datenstrukturen:**
    - `SystemCallIdentifier_t`: Enumeration oder Typdefinition für `call_id`.
    - `SystemCallArguments_t`: Potenziell eine Union oder Struktur, um variable Argumentlisten typsicher zu kapseln, falls die Zielsprache dies erfordert und unterstützt.
    - `SystemCallResult_t`: Struktur zur Aufnahme von Rückgabewerten und Fehlerinformationen.
- **Fehlerbehandlung:** Abbildung von OS-spezifischen Systemaufruffehlern (z.B. `EINVAL`, `EPERM` aus POSIX 18) auf klar definierte Fehlercodes der Kern-Schicht. Jeder mögliche Fehler des zugrundeliegenden Systemaufrufs muss hier berücksichtigt und abgebildet werden.
- **Abhängigkeiten:** Keine externen Abhängigkeiten innerhalb der Kern-Schicht. Direkte Interaktion mit dem Betriebssystem-Kernel.
- **Testfälle:**
    - Testfall 1.3.1: Aufruf einer gültigen `call_id` mit korrekten Argumenten. Erwartet: Erfolgreiche Ausführung, `result_status` zeigt Erfolg an, spezifische Rückgabewerte (falls zutreffend) sind korrekt.
    - Testfall 1.3.2: Aufruf einer ungültigen `call_id`. Erwartet: `result_status` zeigt spezifischen Fehlercode für "ungültiger Systemaufruf".
    - Testfall 1.3.3: Aufruf einer gültigen `call_id` mit ungültigen Argumenten (z.B. falscher Typ, ungültiger Zeiger). Erwartet: `result_status` zeigt spezifischen Fehlercode für "ungültige Argumente".
    - Testfall 1.3.4 (für jeden relevanten Systemaufruf): Simulation eines vom OS zurückgegebenen Fehlers (z.B. `EPERM`). Erwartet: `result_status` zeigt den korrekt abgebildeten Kern-Schicht-Fehlercode.

Die Abstraktion von Systemaufrufen, selbst wenn es sich nur um eine dünne Schicht handelt, ist von entscheidender Bedeutung. Erstens zentralisiert eine solche Abstraktion (`Kern_InvokeSystemCall`) die Betriebssystemspezifität, was die Portierbarkeit verbessert, da potenziell nur dieses Submodul angepasst werden muss, um ein anderes Betriebssystem mit unterschiedlichen Systemaufrufkonventionen zu unterstützen. Zweitens ermöglicht es das Mocking von Systemaufrufen während des Unit-Testings anderer Module der Kern-Schicht, was die Testbarkeit des Gesamtsystems erhöht. Drittens kann eine solche Abstraktion eine für die Domänen-Schicht benutzerfreundlichere Schnittstelle bereitstellen als rohe Systemaufrufe. Dies fördert die Modularität und Wartbarkeit, welche Schlüsselziele dieses Projekts sind.

**1.4. Submodul: Prozessmanagement-Abstraktion**

- **Verantwortlichkeit:** Abstraktion der Betriebssystemprimitive für die Erstellung, Beendigung, Statusabfrage und grundlegende Steuerung (Anhalten, Fortsetzen) von Prozessen. Kernel sind für das Prozessmanagement zuständig 9, und POSIX definiert entsprechende APIs wie `fork` und `exec`.27 Ein Prozess ist durch seinen Maschinenzustand (Speicher, Register) definiert.28
- **Schlüsselfunktionen (Beispiele):**
    - `Kern_CreateProcess(executable_path, arguments, environment_vars, creation_flags, &process_handle, &process_id)`
    - `Kern_TerminateProcess(process_handle, exit_code)`
    - `Kern_WaitForProcess(process_handle, &exit_code, timeout_ms)`
    - `Kern_GetProcessStatus(process_handle, &status_info)`
    - `Kern_SuspendProcess(process_handle)`
    - `Kern_ResumeProcess(process_handle)`
- **Datenstrukturen:**
    - `ProcessHandle_t`: Ein opaker Typ zur Repräsentation eines Prozess-Handles, das von der Kern-Schicht verwaltet wird.
    - `ProcessId_t`: Typ zur Repräsentation der Prozess-ID.
    - `ProcessStatusInfo_t`: Struktur mit Feldern wie `pid` (Typ `ProcessId_t`), `state` (Enumeration: `RUNNING`, `STOPPED`, `TERMINATED`, etc.), `exit_code` (Integer), optional `cpu_usage_percent` (Float), `memory_usage_bytes` (Integer). Dies ist eine Abstraktion über das Konzept des Process Control Blocks (PCB) des Betriebssystems.28
    - `ProcessCreationFlags_t`: Enumeration für Flags wie `CREATE_NEW_CONSOLE`, `DETACHED_PROCESS`.
- **Fehlerbehandlung:**
    - `KERN_ERR_PATH_NOT_FOUND` (Abbildung von `ENOENT`): Wenn `executable_path` nicht existiert.
    - `KERN_ERR_PERMISSION_DENIED` (Abbildung von `EACCES`): Keine Berechtigung zum Ausführen der Datei.
    - `KERN_ERR_OUT_OF_MEMORY` (Abbildung von `ENOMEM`): Nicht genügend Speicher, um den Prozess zu erstellen.
    - `KERN_ERR_INVALID_HANDLE`: Bei Operationen auf einem ungültigen `process_handle`.
    - `KERN_ERR_PROCESS_NOT_FOUND`: Wenn ein Handle oder eine ID einem nicht mehr existierenden Prozess entspricht.
    - `KERN_ERR_TIMEOUT`: Wenn `Kern_WaitForProcess` mit Timeout abläuft.
- **Abhängigkeiten:** `System Call Interface Abstraction` für die Interaktion mit dem OS-Kernel.
- **Testfälle:**
    - Testfall 1.4.1: Erstellen eines validen Prozesses. Erwartet: `KERN_SUCCESS`, gültiges `process_handle` und `process_id`. Prozess läuft.
    - Testfall 1.4.2: Versuch, einen nicht existierenden Pfad auszuführen. Erwartet: `KERN_ERR_PATH_NOT_FOUND`.
    - Testfall 1.4.3: Beenden eines laufenden Prozesses. Erwartet: `KERN_SUCCESS`. Prozess ist beendet.
    - Testfall 1.4.4: Warten auf einen Prozess, der normal terminiert. Erwartet: `KERN_SUCCESS`, korrekter `exit_code`.
    - Testfall 1.4.5: Abfragen des Status eines laufenden Prozesses. Erwartet: `KERN_SUCCESS`, korrekte `ProcessStatusInfo_t`.

Die folgende Tabelle ist entscheidend für den KI-Agenten, um zu verstehen, wie die abstrakten Kern-Schicht-APIs auf die spezifischen Systemaufrufe des gewählten Zielbetriebssystems abgebildet werden. Sie macht den "lückenlosen" Plan konkret.

**Tabelle 1: Abbildung der Prozessmanagement-API**

|   |   |   |   |   |
|---|---|---|---|---|
|**Kern-Schicht API**|**Konzeptionelle POSIX API (Beispiel)**|**Ziel-OS Systemaufruf(e) / Kernel Funktion(en) (Beispiel Linux/POSIX)**|**OS-Level Datenstrukturen (Beispiel Linux/POSIX)**|**Kern-Schicht Fehlercode-Mapping (Beispiele)**|
|`Kern_CreateProcess(path, args, env, flags, &handle, &pid)`|`fork()`, `execve()`|`clone()`, `execve()`|`task_struct`|`ENOENT` → `KERN_ERR_PATH_NOT_FOUND`, `EACCES` → `KERN_ERR_PERMISSION_DENIED`, `ENOMEM` → `KERN_ERR_OUT_OF_MEMORY`|
|`Kern_TerminateProcess(handle, exit_code)`|`kill()`|`kill()`|`task_struct`|`ESRCH` → `KERN_ERR_PROCESS_NOT_FOUND`, `EPERM` → `KERN_ERR_PERMISSION_DENIED`|
|`Kern_WaitForProcess(handle, &exit_code, timeout)`|`waitpid()`|`waitpid()`, `select()`/`poll()` für Timeout|`task_struct`|`ECHILD` → `KERN_ERR_NO_CHILD_PROCESSES` (oder spezifischer Fehler, wenn Handle ungültig), Timeout → `KERN_ERR_TIMEOUT`|
|`Kern_GetProcessStatus(handle, &status)`|(Kombination)|Lese `/proc/[pid]/stat`, `getrusage()`|`task_struct`, `/proc` Einträge|`ESRCH` → `KERN_ERR_PROCESS_NOT_FOUND`|
|`Kern_SuspendProcess(handle)`|`kill(pid, SIGSTOP)`|`kill(pid, SIGSTOP)`|`task_struct`|`ESRCH` → `KERN_ERR_PROCESS_NOT_FOUND`, `EPERM` → `KERN_ERR_PERMISSION_DENIED`|
|`Kern_ResumeProcess(handle)`|`kill(pid, SIGCONT)`|`kill(pid, SIGCONT)`|`task_struct`|`ESRCH` → `KERN_ERR_PROCESS_NOT_FOUND`, `EPERM` → `KERN_ERR_PERMISSION_DENIED`|

_Anmerkung: Die Spalten "Ziel-OS Systemaufruf(e)" und "OS-Level Datenstrukturen" müssen basierend auf dem final gewählten Zielbetriebssystem präzisiert werden._

**1.5. Submodul: Speicherverwaltungs-Abstraktion**

- **Verantwortlichkeit:** Abstraktion der Betriebssystemprimitive für Speicherallokation, -deallokation und Operationen des virtuellen Speichers (sofern diese höheren Schichten zugänglich gemacht werden). Die Speicherverwaltung ist eine Kernfunktion des Kernels.9 Linux verwendet beispielsweise GFP-Flags zur Steuerung der Allokation.30 Shared Memory erfordert spezifische Systemaufrufe wie `shmget` und `shmat`.31
- **Schlüsselfunktionen (Beispiele):**
    - `Kern_AllocateMemory(size_bytes, allocation_flags, &allocated_address)`
    - `Kern_FreeMemory(address_to_free)`
    - `Kern_MapSharedMemory(shared_memory_name, size_bytes, access_flags, &shared_memory_handle, &mapped_address)`
    - `Kern_UnmapSharedMemory(shared_memory_handle, mapped_address)`
    - `Kern_ProtectMemory(address, size_bytes, protection_flags)`
- **Datenstrukturen:**
    - `MemoryHandle_t`: Opaker Typ für Handles von Speicherobjekten (z.B. für Shared Memory).
    - `MemoryAllocationFlags_t`: Enumeration für Flags wie `MEM_READ_WRITE`, `MEM_EXECUTE`, `MEM_ZERO_INIT`.
    - `MemoryProtectionFlags_t`: Enumeration für Flags wie `PROT_READ`, `PROT_WRITE`, `PROT_EXEC`, `PROT_NONE`.
- **Fehlerbehandlung:**
    - `KERN_ERR_OUT_OF_MEMORY` (Abbildung von `ENOMEM`): Nicht genügend Speicher verfügbar.
    - `KERN_ERR_INVALID_ARGUMENT` (Abbildung von `EINVAL`): Ungültige Flags, Größe oder Adresse.
    - `KERN_ERR_ACCESS_DENIED`: Bei `Kern_ProtectMemory`, wenn die Operation nicht erlaubt ist.
    - `KERN_ERR_SHARED_MEMORY_EXISTS`: Bei `Kern_MapSharedMemory`, wenn ein Shared Memory Segment mit diesem Namen bereits existiert und exklusiver Zugriff angefordert wurde.
    - `KERN_ERR_SHARED_MEMORY_NOT_FOUND`: Wenn auf ein nicht existentes Shared Memory Segment zugegriffen wird.
- **Abhängigkeiten:** `System Call Interface Abstraction`.
- **Testfälle:**
    - Testfall 1.5.1: Allokieren und Freigeben eines Speicherblocks. Erwartet: `KERN_SUCCESS` für beide Operationen. Speicher ist nach Allokation beschreib- und lesbar (falls nicht anders geflaggt).
    - Testfall 1.5.2: Versuch, mehr Speicher zu allokieren als verfügbar. Erwartet: `KERN_ERR_OUT_OF_MEMORY`.
    - Testfall 1.5.3: Erstellen, Mappen, Beschreiben, Lesen und Unmappen eines Shared Memory Segments durch zwei Prozesse. Erwartet: `KERN_SUCCESS` für alle Operationen, Datenkonsistenz.
    - Testfall 1.5.4: Versuch, Speicher mit ungültiger Adresse freizugeben. Erwartet: `KERN_ERR_INVALID_ARGUMENT` oder plattformspezifischer Fehler.

Die Abstraktion der Speicherverwaltung muss die Eigentümerschaft und Lebensdauer des allokierten Speichers, insbesondere für Shared Memory, klar definieren. Dies ist essenziell, um Speicherlecks oder Use-after-free-Fehler durch den KI-generierten Code zu verhindern. Speicherverwaltung ist inhärent fehleranfällig. KI-Agenten, auch wenn sie potenziell gut in der Codegenerierung sind, verstehen komplexe Semantiken der Speicherverantwortung möglicherweise nicht ohne explizite Anweisungen. Der "lückenlose" Plan für Funktionen wie `Kern_AllocateMemory` und `Kern_MapSharedMemory` muss daher genau spezifizieren, wer für die Freigabe bzw. das Unmapping verantwortlich ist und unter welchen Bedingungen dies zu geschehen hat. Dies schließt die detaillierte Verwaltung und Invalidierung von Handles ein. Ein Versäumnis an dieser Stelle führt unweigerlich zu Ressourcenlecks oder Abstürzen und verletzt somit die Kriterien der "Lückenlosigkeit" und "Implementierbarkeit".

**1.6. Submodul: Dateisystem-Abstraktion**

- **Verantwortlichkeit:** Abstraktion der Betriebssystemprimitive für Datei- und Verzeichnisoperationen. Die Dateisystemverwaltung ist eine Kernelfunktion.9 Dateisystem-APIs umfassen typischerweise Operationen wie Öffnen, Schließen, Lesen, Schreiben, Metadatenverwaltung und Verzeichnismanagement.34 Das Virtual File System (VFS) in Linux stellt hierfür gängige Abstraktionen wie Inodes und Dentries bereit.35
- **Schlüsselfunktionen (Beispiele):**
    - `Kern_OpenFile(file_path, open_flags, access_mode, &file_handle)`
    - `Kern_CloseFile(file_handle)`
    - `Kern_ReadFile(file_handle, buffer, buffer_size_bytes, &bytes_read)`
    - `Kern_WriteFile(file_handle, buffer, bytes_to_write, &bytes_written)`
    - `Kern_SeekFile(file_handle, offset, seek_origin, &new_position)`
    - `Kern_GetFileInfo(path_or_handle, &file_info_struct)`
    - `Kern_CreateDirectory(directory_path, access_mode)`
    - `Kern_DeleteFile(file_path)`
    - `Kern_DeleteDirectory(directory_path)`
    - `Kern_ListDirectory(directory_path, &directory_entries_array, &entry_count)`
    - `Kern_RenameFile(old_path, new_path)`
    - `Kern_SetFilePermissions(path_or_handle, access_mode)`
- **Datenstrukturen:**
    - `FileHandle_t`: Opaker Typ für Datei-Handles.
    - `FileInfo_t`: Struktur mit Feldern wie `size_bytes` (Integer), `type` (Enumeration: `FILE_TYPE_REGULAR`, `FILE_TYPE_DIRECTORY`, `FILE_TYPE_SYMLINK`), `permissions` (Integer, oktal), `creation_time_ns` (Integer), `modification_time_ns` (Integer), `access_time_ns` (Integer).
    - `DirectoryEntry_t`: Struktur mit Feldern wie `name` (String), `type` (Enumeration wie oben).
    - `FileOpenFlags_t`: Enumeration für Flags wie `OPEN_READ_ONLY`, `OPEN_WRITE_ONLY`, `OPEN_READ_WRITE`, `OPEN_CREATE`, `OPEN_APPEND`, `OPEN_TRUNCATE`.
    - `FileAccessMode_t`: Typ für Zugriffsrechte (z.B. oktale Darstellung wie in POSIX).
    - `SeekOrigin_t`: Enumeration: `SEEK_SET`, `SEEK_CUR`, `SEEK_END`.
- **Fehlerbehandlung:**
    - `KERN_ERR_FILE_NOT_FOUND` / `KERN_ERR_PATH_NOT_FOUND` (Abbildung von `ENOENT`).
    - `KERN_ERR_PERMISSION_DENIED` (Abbildung von `EACCES`).
    - `KERN_ERR_IO_ERROR` (Abbildung von `EIO`).
    - `KERN_ERR_NO_SPACE_ON_DEVICE` (Abbildung von `ENOSPC`).
    - `KERN_ERR_IS_DIRECTORY` (Abbildung von `EISDIR`).
    - `KERN_ERR_NOT_A_DIRECTORY` (Abbildung von `ENOTDIR`).
    - `KERN_ERR_FILE_EXISTS` (Abbildung von `EEXIST`).
    - `KERN_ERR_INVALID_HANDLE`.
    - `KERN_ERR_NAME_TOO_LONG` (Abbildung von `ENAMETOOLONG`).
- **Abhängigkeiten:** `System Call Interface Abstraction`.
- **Testfälle:**
    - Testfall 1.6.1: Erstellen, Schreiben, Lesen, Suchen und Schließen einer Datei. Erwartet: `KERN_SUCCESS` für alle Operationen, Datenkonsistenz.
    - Testfall 1.6.2: Versuch, eine nicht existierende Datei zu öffnen. Erwartet: `KERN_ERR_FILE_NOT_FOUND`.
    - Testfall 1.6.3: Versuch, ohne Berechtigung in eine Datei zu schreiben. Erwartet: `KERN_ERR_PERMISSION_DENIED`.
    - Testfall 1.6.4: Erstellen und Auflisten eines Verzeichnisses. Erwartet: `KERN_SUCCESS`, korrekte Verzeichniseinträge.
    - Testfall 1.6.5: Löschen einer Datei und eines leeren Verzeichnisses. Erwartet: `KERN_SUCCESS`.
    - Testfall 1.6.6: Versuch, ein nicht leeres Verzeichnis zu löschen (ohne rekursives Flag). Erwartet: `KERN_ERR_DIRECTORY_NOT_EMPTY` (oder Äquivalent).

**1.7. Submodul: Interprozesskommunikations-Abstraktion (IPC)**

- **Verantwortlichkeit:** Abstraktion von Low-Level-Betriebssystem-IPC-Mechanismen. IPC ist ein Kerndienst.9 Gängige Mechanismen sind Shared Memory 31, Message Passing/Queues 37, Pipes 37 und Sockets für lokale IPC.37 POSIX definiert APIs für diese Mechanismen.38
- **Schlüsselfunktionen (Beispiele, abhängig von den gewählten Mechanismen):**
    - **Pipes:**
        - `Kern_CreatePipe(&read_handle, &write_handle, flags)`
        - `Kern_ReadPipe(read_handle, buffer, buffer_size_bytes, &bytes_read, timeout_ms)`
        - `Kern_WritePipe(write_handle, buffer, bytes_to_write, &bytes_written, timeout_ms)`
        - `Kern_ClosePipeHandle(pipe_handle)`
    - **Message Queues (POSIX-ähnlich):**
        - `Kern_CreateMessageQueue(queue_name, max_messages, max_message_size, flags, &mq_handle)`
        - `Kern_OpenMessageQueue(queue_name, flags, &mq_handle)`
        - `Kern_CloseMessageQueue(mq_handle)`
        - `Kern_UnlinkMessageQueue(queue_name)`
        - `Kern_SendMessage(mq_handle, message_buffer, message_size, priority, timeout_ms)`
        - `Kern_ReceiveMessage(mq_handle, buffer, buffer_size_bytes, &received_size, &priority, timeout_ms)`
    - **Shared Memory:** (Bereits abgedeckt durch Speicherverwaltungs-Abstraktion: `Kern_MapSharedMemory`, `Kern_UnmapSharedMemory`. Benötigt zusätzlich Synchronisationsprimitive, siehe potenzielles Submodul für Nebenläufigkeit.)
- **Datenstrukturen:**
    - `PipeHandle_t`: Opaker Typ für Lese- oder Schreibenden einer Pipe.
    - `MessageQueueHandle_t`: Opaker Typ für Message Queues.
    - `IPCMessage_t`: Struktur zur Kapselung von Nachrichten (z.B. Puffer und Größe).
    - `IPCFlags_t`: Enumeration für Flags (z.B. `IPC_NON_BLOCKING`).
- **Fehlerbehandlung:**
    - Pipes: `KERN_ERR_BROKEN_PIPE` (Abbildung von `EPIPE`), `KERN_ERR_TIMEOUT`, `KERN_ERR_INVALID_HANDLE`.
    - Message Queues: `KERN_ERR_QUEUE_NOT_FOUND`, `KERN_ERR_QUEUE_EXISTS`, `KERN_ERR_MESSAGE_TOO_BIG`, `KERN_ERR_QUEUE_FULL`, `KERN_ERR_QUEUE_EMPTY`, `KERN_ERR_TIMEOUT`.
- **Abhängigkeiten:** `System Call Interface Abstraction`, `Speicherverwaltungs-Abstraktion` (für Shared Memory), potenziell ein Nebenläufigkeits-Submodul für Synchronisation.
- **Testfälle:**
    - Testfall 1.7.1 (Pipes): Erstellen einer Pipe, Schreiben von Daten durch einen Prozess, Lesen durch einen anderen. Erwartet: `KERN_SUCCESS`, Datenkonsistenz.
    - Testfall 1.7.2 (Pipes): Versuch, in eine geschlossene Pipe zu schreiben. Erwartet: `KERN_ERR_BROKEN_PIPE`.
    - Testfall 1.7.3 (Message Queues): Erstellen einer Queue, Senden und Empfangen von Nachrichten zwischen Prozessen. Erwartet: `KERN_SUCCESS`, Nachrichtenintegrität und korrekte Priorisierung.
    - Testfall 1.7.4 (Message Queues): Versuch, eine Nachricht in eine volle Queue (non-blocking) zu senden. Erwartet: `KERN_ERR_QUEUE_FULL` oder entsprechender Fehler.

Die Auswahl der hier abstrahierten IPC-Mechanismen hat weitreichende Konsequenzen für die Architektur der Domänen- und Systemschichten, insbesondere wie Dienste und Anwendungen miteinander kommunizieren. Eine robuste, wohldefinierte IPC-Abstraktion ist kritisch für die Modularität der gesamten Desktop-Umgebung. Die Kern-Schicht stellt die grundlegenden IPC-Primitive bereit. Wenn sie beispielsweise nur einfache Pipes anbietet, sind höhere Schichten gezwungen, entweder nur Pipes zu verwenden oder komplexere Kommunikationsprotokolle darauf aufzubauen. Bietet sie hingegen reichhaltigere Primitive wie Message Queues oder eine Abstraktion für Shared Memory mit Synchronisation, ermöglicht dies vielfältigere und potenziell effizientere Kommunikationsmuster in den höheren Schichten. Diese Entscheidung beeinflusst das Design jedes interkommunizierenden Moduls im System.

**1.8. Submodul: Geräteschnittstellen-Abstraktion**

- **Verantwortlichkeit:** Bereitstellung einer generalisierten Methode für höhere Schichten (primär System- und UI-Schicht), um mit gängigen Gerätetypen (z.B. Eingabegeräte, Anzeigegeräte) über das Gerätemodell des Betriebssystems zu interagieren. Es geht hierbei nicht um das Schreiben von Gerätetreibern, sondern um die Abstraktion der Betriebssystemschnittstelle _zu_ den Treibern. Gerätetreiber selbst abstrahieren die Hardware 47, und der Kernel verwaltet diese Treiber.9 Die Android HAL dient als Beispiel für eine solche Abstraktionsebene.49
- **Schlüsselfunktionen (Beispiele, stark abhängig von Gerätetypen):**
    - **Eingabegeräte (Tastatur, Maus, Touch):**
        - `Kern_OpenInputDevice(device_name_or_id, &device_handle, flags)`
        - `Kern_CloseInputDevice(device_handle)`
        - `Kern_ReadInputEvent(device_handle, &input_event_struct, timeout_ms)`
        - `Kern_GetInputDeviceCapabilities(device_handle, &capabilities_struct)`
    - **Anzeigegeräte (Display/Framebuffer):**
        - `Kern_OpenDisplayDevice(display_id, &display_handle, flags)`
        - `Kern_CloseDisplayDevice(display_handle)`
        - `Kern_GetDisplayInfo(display_handle, &display_info_struct)`
        - `Kern_GetFramebuffer(display_handle, &framebuffer_address, &framebuffer_info_struct)` (falls direkter Framebuffer-Zugriff Teil der Architektur ist)
        - `Kern_SetDisplayMode(display_handle, mode_id_or_params)`
- **Datenstrukturen:**
    - `DeviceHandle_t`: Opaker Typ für Geräte-Handles.
    - `InputEvent_t`: Struktur oder Union zur Repräsentation verschiedener Eingabeereignisse (Tastendruck, Mausbewegung, Mausklick, Touch-Ereignis) mit Feldern wie `event_type`, `timestamp_ns`, `device_id`, und ereignisspezifischen Daten (z.B. `key_code`, `mouse_x`, `mouse_y`, `button_state`).
    - `InputDeviceCapabilities_t`: Struktur zur Beschreibung der Fähigkeiten eines Eingabegeräts (z.B. unterstützte Tasten, Achsen, Auflösung).
    - `DisplayInfo_t`: Struktur mit Informationen wie Auflösung (Breite, Höhe in Pixel), Farbtiefe (Bits pro Pixel), Bildwiederholfrequenz (Hz).
    - `FramebufferInfo_t`: Struktur mit Adresse des Framebuffers, Größe, Stride (Bytes pro Zeile), Pixelformat.
- **Fehlerbehandlung:**
    - `KERN_ERR_DEVICE_NOT_FOUND` (Abbildung von `ENODEV`): Gerät nicht vorhanden oder nicht zugreifbar.
    - `KERN_ERR_IO_ERROR` (Abbildung von `EIO`): Allgemeiner Fehler bei der Gerätekommunikation.
    - `KERN_ERR_PERMISSION_DENIED`: Kein Zugriff auf das Gerät erlaubt.
    - `KERN_ERR_INVALID_HANDLE`.
    - `KERN_ERR_TIMEOUT`: Bei Leseoperationen mit Timeout.
    - `KERN_ERR_UNSUPPORTED_OPERATION`: Wenn ein Gerät eine angeforderte Funktion nicht unterstützt.
- **Abhängigkeiten:** `System Call Interface Abstraction`, `Dateisystem-Abstraktion` (da Geräte oft als Dateien im OS repräsentiert werden, z.B. unter `/dev`).
- **Testfälle:**
    - Testfall 1.8.1 (Input): Öffnen eines (simulierten/gemockten) Eingabegeräts, Lesen von Ereignissen. Erwartet: `KERN_SUCCESS`, korrekte Ereignisdaten.
    - Testfall 1.8.2 (Input): Versuch, ein nicht existierendes Eingabegerät zu öffnen. Erwartet: `KERN_ERR_DEVICE_NOT_FOUND`.
    - Testfall 1.8.3 (Display): Abfragen von Displayinformationen eines (simulierten) Displays. Erwartet: `KERN_SUCCESS`, plausible `DisplayInfo_t`.
    - Testfall 1.8.4 (Display): Anfordern eines Framebuffers. Erwartet: `KERN_SUCCESS`, gültige `FramebufferInfo_t` und Adresse (falls zutreffend).

**Tabelle 2: Abstrahierte Geräte-I/O-Operationen**

|   |   |   |   |
|---|---|---|---|
|**Abstrahierter Gerätetyp**|**Kern-Schicht API Funktion**|**Konzeptionelle Operation**|**Ziel-OS Mechanismus (Beispiel Linux/POSIX)**|
|Tastatur|`Kern_ReadInputEvent`|Lesen eines Tastendrucks/-loslassens|Lesen von `/dev/input/eventX`, `ioctl(EVIOCGKEY)`|
|Maus|`Kern_ReadInputEvent`|Lesen von Mausbewegung/Klicks|Lesen von `/dev/input/eventX` (relative oder absolute Koordinaten), `ioctl(EVIOCGBIT)` für Tasten|
|Touchscreen|`Kern_ReadInputEvent`|Lesen von Touch-Punkten/-Gesten|Lesen von `/dev/input/eventX` (Multi-Touch Protokoll)|
|Display|`Kern_GetDisplayInfo`|Abfrage von Auflösung, Farbtiefe|`ioctl` auf `/dev/fb0` (Framebuffer) oder XRandR/DRM/KMS APIs|
|Display|`Kern_GetFramebuffer`|Zugriff auf den Framebuffer-Speicher|`mmap` auf `/dev/fb0`|
|Display|`Kern_SetDisplayMode`|Ändern der Auflösung/Frequenz|`ioctl` auf Framebuffer-Device oder DRM/KMS APIs|

_Anmerkung: Die Spalte "Ziel-OS Mechanismus" muss basierend auf dem final gewählten Zielbetriebssystem präzisiert werden. Diese Tabelle ist entscheidend, um dem KI-Agenten die Verbindung zwischen der abstrakten API und den konkreten OS-Interaktionen für die Gerätehandhabung aufzuzeigen._

**1.9. Submodul: Low-Level Timer- und Uhrzeit-Abstraktion**

- **Verantwortlichkeit:** Abstraktion der Betriebssystemprimitive für hochauflösende Timer, Zugriff auf die Systemuhr und potenziell Schlaf-/Verzögerungsfunktionen. POSIX bietet hierfür beispielsweise `timer_create` und `timer_settime`.
- **Schlüsselfunktionen (Beispiele):**
    - `Kern_GetCurrentTimeNanoseconds(&timestamp_ns)`
    - `Kern_GetCurrentTimeMonotonicNanoseconds(&monotonic_timestamp_ns)`
    - `Kern_CreateTimer(callback_function_ptr, callback_context_ptr, timer_flags, &timer_handle)`
    - `Kern_SetTimerRelative(timer_handle, initial_delay_ns, interval_ns)`
    - `Kern_SetTimerAbsolute(timer_handle, absolute_expiry_time_ns, interval_ns)`
    - `Kern_CancelTimer(timer_handle)`
    - `Kern_DeleteTimer(timer_handle)`
    - `Kern_SleepNanoseconds(duration_ns)`
- **Datenstrukturen:**
    - `TimerHandle_t`: Opaker Typ für Timer-Handles.
    - `TimestampNs_t`: Typ für Zeitstempel in Nanosekunden (z.B. `uint64_t`).
    - `TimerCallback_t`: Funktionszeigertyp für Timer-Callbacks: `void (*TimerCallback_t)(TimerHandle_t timer_handle, void* callback_context_ptr)`.
    - `TimerFlags_t`: Enumeration für Flags (z.B. `TIMER_ONE_SHOT`, `TIMER_PERIODIC`).
- **Fehlerbehandlung:**
    - `KERN_ERR_INVALID_ARGUMENT` (Abbildung von `EINVAL`): Ungültige Timer-Parameter (z.B. negative Delays).
    - `KERN_ERR_OUT_OF_RESOURCES`: Maximale Anzahl an Timern erreicht.
    - `KERN_ERR_INVALID_HANDLE`: Bei Operationen auf einem ungültigen `timer_handle`.
- **Abhängigkeiten:** `System Call Interface Abstraction`.
- **Testfälle:**
    - Testfall 1.9.1: Erstellen und Starten eines One-Shot-Timers. Erwartet: `KERN_SUCCESS`, Callback wird nach korrekter Verzögerung einmalig aufgerufen.
    - Testfall 1.9.2: Erstellen und Starten eines periodischen Timers, dann Abbrechen. Erwartet: `KERN_SUCCESS`, Callback wird periodisch aufgerufen, bis `Kern_CancelTimer` erfolgt.
    - Testfall 1.9.3: `Kern_SleepNanoseconds` für eine definierte Dauer. Erwartet: `KERN_SUCCESS`, Prozess blockiert für die angegebene Dauer.
    - Testfall 1.9.4: Abfragen der aktuellen Zeit mehrfach. Erwartet: Zeitstempel sind monoton steigend (für `Kern_GetCurrentTimeMonotonicNanoseconds`).

**1.10. Submodul: Interrupt-Behandlungs-Abstraktion (falls anwendbar)**

- **Verantwortlichkeit:** Falls irgendein Teil der Desktop-Umgebung (außerhalb von Gerätetreibern selbst, die üblicherweise Teil des Betriebssystems sind) direkt mit Hardware-Interrupts interagieren muss. Dies ist für eine typische Desktop-Umgebung höchst unwahrscheinlich, wird aber der Vollständigkeit halber für Kernel-Level-Abstraktionen aufgeführt. Dies würde die Abstraktion von Interrupt-Registrierungs- und Behandlungsmechanismen beinhalten. Der Kernel verwaltet Interrupts 50, und Linux-Module können Interrupts behandeln.51
- **Schlüsselfunktionen (Beispiele):**
    - `Kern_RegisterInterruptHandler(irq_number, handler_function_ptr, handler_context_ptr, &irq_handle)`
    - `Kern_UnregisterInterruptHandler(irq_handle)`
    - `Kern_EnableInterrupt(irq_handle_or_number)`
    - `Kern_DisableInterrupt(irq_handle_or_number)`
- **Datenstrukturen:**
    - `InterruptHandle_t`: Opaker Typ für registrierte Interrupt-Handler.
    - `InterruptHandler_t`: Funktionszeigertyp für Interrupt-Handler: `void (*InterruptHandler_t)(int irq_number, void* handler_context_ptr)`.
- **Fehlerbehandlung:**
    - `KERN_ERR_INVALID_IRQ_NUMBER`.
    - `KERN_ERR_IRQ_REGISTRATION_CONFLICT`.
    - `KERN_ERR_PERMISSION_DENIED`.
- **Abhängigkeiten:** `System Call Interface Abstraction`.
- **Testfälle:** (Stark abhängig von der Testbarkeit in der Zielumgebung)
    - Testfall 1.10.1: Registrieren und Deregistrieren eines (simulierten) Interrupt-Handlers. Erwartet: `KERN_SUCCESS`.
    - Testfall 1.10.2: Versuch, einen Handler für eine ungültige IRQ-Nummer zu registrieren. Erwartet: `KERN_ERR_INVALID_IRQ_NUMBER`.

Dieses Submodul wird voraussichtlich minimal oder gar nicht existent sein, es sei denn, die "Desktop-Umgebung" hat sehr ungewöhnliche, tiefgreifende Hardware-Interaktionsanforderungen, die über Standard-Betriebssystemdienste hinausgehen. Interrupt-Behandlung ist typischerweise die Domäne von Kernel-Mode-Gerätetreibern. User-Space-Anwendungen und damit auch eine Desktop-Umgebung, die auf einem Host-Betriebssystem läuft, registrieren normalerweise keine Interrupt-Handler direkt; sie verlassen sich darauf, dass Treiber Ereignisse an die Oberfläche bringen. Die Notwendigkeit dieses Submoduls hängt kritisch von der Definition der Grenze der Kern-Schicht und der Natur des Zielbetriebssystems ab. Wenn die Kern-Schicht in einem hochprivilegierten Modus oder als Teil eines benutzerdefinierten Kernels operiert, dann ist dies relevant. Andernfalls ist es wahrscheinlich nicht erforderlich.

**Teil 2: Domänen-Schicht (Domain Layer) – Systemdienste, Ressourcenmanagement**

**2.1. Einführung in die Domänen-Schicht**

- **Zweck:** Die Domänen-Schicht (Domain Layer) baut auf den Abstraktionen der Kern-Schicht auf und stellt übergeordnete Systemdienste sowie Mechanismen zur Verwaltung gemeinsam genutzter Ressourcen bereit, die für den Betrieb einer Desktop-Umgebung fundamental sind. Sie dient als Brücke zwischen den Low-Level-Funktionen des Kerns und der anwendungsspezifischen Logik der System- und UI-Schichten.
- **Hauptverantwortlichkeiten:**
    - Verwaltung von Benutzer-Sessions, einschließlich Authentifizierung und Sitzungsstatus.
    - Zentralisierte Verwaltung von systemweiten und benutzerspezifischen Konfigurationen.
    - Management des Lebenszyklus von Anwendungen (über die reine Prozesserstellung hinausgehend, z.B. Registrierung laufender Anwendungen).
    - Vermittlung von Ressourcen (z.B. exklusiver Hardwarezugriff, falls erforderlich, über die Kern-Schicht-Abstraktionen).
    - Potenziell Implementierung von Kern-System-Daemons, die Hintergrundaufgaben für die Desktop-Umgebung ausführen.
- **Interaktionsmodell:** Dienste innerhalb der Domänen-Schicht kommunizieren untereinander und mit der System- bzw. UI-Schicht primär über die von der Kern-Schicht abstrahierten IPC-Mechanismen (z.B. Message Queues, Shared Memory mit Synchronisation). Jede Funktion, die als Dienst exponiert wird, muss eine klare IPC-Schnittstellenspezifikation haben.

**2.2. Submodul: Benutzer- und Sitzungsmanagement-Dienst**

- **Verantwortlichkeit:** Verwaltung von Benutzerkonten (Abruf von Benutzerinformationen, nicht Erstellung/Löschung, da dies meist OS-Aufgabe ist), Benutzeranmeldung/-abmeldung, Verwaltung des Sitzungsstatus und potenziell benutzerspezifischer Laufzeitkonfigurationen.
- **Schlüsselfunktionen/IPC-Schnittstellen (Beispiele):**
    - IPC-Nachricht: `AUTHENTICATE_USER_REQUEST` (Parameter: `username_string`, `credentials_opaque_data`)
        - Antwort: `AUTHENTICATE_USER_RESPONSE` (Parameter: `session_token_string`, `user_id_type`, `error_code_enum`)
    - IPC-Nachricht: `START_SESSION_REQUEST` (Parameter: `user_id_type`, `display_identifier_string`)
        - Antwort: `START_SESSION_RESPONSE` (Parameter: `session_id_type`, `error_code_enum`)
    - IPC-Nachricht: `END_SESSION_REQUEST` (Parameter: `session_id_type`)
        - Antwort: `END_SESSION_RESPONSE` (Parameter: `error_code_enum`)
    - IPC-Nachricht: `GET_CURRENT_USER_INFO_REQUEST` (Parameter: `session_id_type`)
        - Antwort: `GET_CURRENT_USER_INFO_RESPONSE` (Parameter: `user_info_struct`, `error_code_enum`)
    - IPC-Nachricht: `QUERY_SESSION_STATE_REQUEST` (Parameter: `session_id_type`)
        - Antwort: `QUERY_SESSION_STATE_RESPONSE` (Parameter: `session_state_enum`, `error_code_enum`)
- **Datenstrukturen:**
    - `UserInfo_t`: Struktur (z.B. `user_id`, `username`, `home_directory_path`, `primary_group_id`).
    - `SessionInfo_t`: Struktur (z.B. `session_id`, `user_id`, `start_time`, `display_id`, `state_enum` wie `ACTIVE`, `LOCKED`).
    - `Credentials_t`: Opake Struktur oder definierter Typ für Anmeldeinformationen (Passwort-Hash, Token etc.).
    - `SessionToken_t`: Typ für Sitzungstoken.
- **Fehlerbehandlung (Beispiel-Fehlercodes für IPC-Antworten):**
    - `DOMAIN_ERR_AUTHENTICATION_FAILED`.
    - `DOMAIN_ERR_USER_NOT_FOUND`.
    - `DOMAIN_ERR_SESSION_INVALID`.
    - `DOMAIN_ERR_MAX_SESSIONS_REACHED`.
    - `DOMAIN_ERR_PERMISSION_DENIED`.
- **Abhängigkeiten:** Kern-Schicht (IPC für Kommunikation, Dateisystem für potenzielle Speicherung von Session-Daten, Timer für Session-Timeouts).
- **Zustandsverwaltung:** Dieser Dienst verwaltet den Zustand aktiver Sessions und die Zuordnung von Benutzern zu Sessions. Persistenz von Benutzerdaten (Passwörter etc.) ist typischerweise Aufgabe des OS; dieser Dienst interagiert damit.
- **Testfälle (fokussiert auf Dienstlogik und IPC):**
    - Testfall 2.2.1: Erfolgreiche Benutzerauthentifizierung. Erwartet: Gültiges `session_token`, korrekte `user_id`.
    - Testfall 2.2.2: Fehlgeschlagene Authentifizierung (falsches Passwort). Erwartet: `DOMAIN_ERR_AUTHENTICATION_FAILED`.
    - Testfall 2.2.3: Starten einer neuen Session für einen authentifizierten Benutzer. Erwartet: Gültige `session_id`.
    - Testfall 2.2.4: Beenden einer aktiven Session. Erwartet: Session wird als beendet markiert.
    - Testfall 2.2.5: Abfrage von Benutzerinformationen für eine gültige Session. Erwartet: Korrekte `UserInfo_t`.

**2.3. Submodul: Systemkonfigurations-Dienst**

- **Verantwortlichkeit:** Bereitstellung einer zentralisierten Schnittstelle zum Lesen und Schreiben von systemweiten und benutzerspezifischen Einstellungen (z.B. Desktop-Hintergrund, Tastaturlayout, Standardanwendungen, Energieeinstellungen). Persistierung dieser Einstellungen unter Verwendung der Dateisystem-Abstraktion der Kern-Schicht.
- **Schlüsselfunktionen/IPC-Schnittstellen (Beispiele):**
    - IPC-Nachricht: `GET_CONFIG_VALUE_REQUEST` (Parameter: `scope_enum` (`SYSTEM` oder `USER_SESSION_ID`), `config_key_string`)
        - Antwort: `GET_CONFIG_VALUE_RESPONSE` (Parameter: `config_value_variant_type`, `error_code_enum`)
    - IPC-Nachricht: `SET_CONFIG_VALUE_REQUEST` (Parameter: `scope_enum`, `config_key_string`, `config_value_variant_type`)
        - Antwort: `SET_CONFIG_VALUE_RESPONSE` (Parameter: `error_code_enum`)
    - IPC-Nachricht: `REGISTER_CONFIG_CHANGE_NOTIFICATION_REQUEST` (Parameter: `scope_enum`, `config_key_string`, `client_ipc_notification_handle`)
        - Antwort: `REGISTER_CONFIG_CHANGE_NOTIFICATION_RESPONSE` (Parameter: `registration_id_type`, `error_code_enum`)
    - IPC-Nachricht (Notification vom Dienst an Client): `CONFIG_VALUE_CHANGED_NOTIFICATION` (Parameter: `scope_enum`, `config_key_string`, `new_config_value_variant_type`)
- **Datenstrukturen:**
    - `ConfigKey_t`: Typ für Konfigurationsschlüssel (z.B. hierarchischer String wie " apariencia.fondo.color").
    - `ConfigValue_t`: Ein Variantentyp, der verschiedene Datentypen für Konfigurationswerte aufnehmen kann (Integer, String, Boolean, Float, Liste von Strings).
    - `ConfigScope_t`: Enumeration (`CONFIG_SCOPE_SYSTEM`, `CONFIG_SCOPE_USER`).
- **Fehlerbehandlung:**
    - `DOMAIN_ERR_CONFIG_KEY_NOT_FOUND`.
    - `DOMAIN_ERR_CONFIG_INVALID_VALUE_TYPE`.
    - `DOMAIN_ERR_CONFIG_PERMISSION_DENIED` (beim Versuch, systemweite Einstellungen ohne Admin-Rechte zu ändern).
    - `DOMAIN_ERR_CONFIG_STORAGE_ERROR` (Fehler beim Lesen/Schreiben der Konfigurationsdatei).
- **Abhängigkeiten:** Kern-Schicht (Dateisystem für Persistenz, IPC für Kommunikation), Benutzer- und Sitzungsmanagement-Dienst (zur Identifizierung des aktuellen Benutzers für benutzerspezifische Einstellungen).
- **Persistenz:** Spezifikation des Formats und Speicherorts der Konfigurationsdateien (z.B. INI-Dateien, JSON, XML, Binärformat).
- **Testfälle:**
    - Testfall 2.3.1: Setzen und Lesen eines systemweiten Konfigurationswertes. Erwartet: Wert wird korrekt gespeichert und gelesen.
    - Testfall 2.3.2: Setzen und Lesen eines benutzerspezifischen Wertes für zwei verschiedene Benutzer. Erwartet: Werte sind benutzerspezifisch korrekt.
    - Testfall 2.3.3: Versuch, einen nicht existierenden Schlüssel zu lesen. Erwartet: `DOMAIN_ERR_CONFIG_KEY_NOT_FOUND`.
    - Testfall 2.3.4: Registrieren für eine Benachrichtigung und Ändern des Wertes. Erwartet: Client erhält korrekte Benachrichtigung.

**2.4. Submodul: Ressourcenmanagement-Dienst (z.B. Energie, Netzwerkstatus)**

- **Verantwortlichkeit:** Überwachung und Bereitstellung von Abstraktionen für systemweite Ressourcen und Zustände wie Energiestatus (Akkuladung, Netzbetrieb), Netzwerkverbindungsstatus, verfügbare Speichermedien etc. Dieser Dienst kann mit den Geräteabstraktionen der Kern-Schicht interagieren, um Rohdaten zu erhalten.
- **Schlüsselfunktionen/IPC-Schnittstellen (Beispiele):**
    - IPC-Nachricht: `GET_POWER_STATUS_REQUEST`
        - Antwort: `GET_POWER_STATUS_RESPONSE` (Parameter: `power_info_struct`, `error_code_enum`)
    - IPC-Nachricht: `GET_NETWORK_STATUS_REQUEST` (Parameter: `network_interface_id_string` (optional, sonst alle))
        - Antwort: `GET_NETWORK_STATUS_RESPONSE` (Parameter: `network_info_array`, `error_code_enum`)
    - IPC-Nachricht: `REGISTER_POWER_EVENT_NOTIFICATION_REQUEST` (Parameter: `client_ipc_notification_handle`)
        - Antwort: `REGISTER_POWER_EVENT_NOTIFICATION_RESPONSE` (Parameter: `registration_id_type`, `error_code_enum`)
    - IPC-Nachricht (Notification): `POWER_EVENT_NOTIFICATION` (Parameter: `event_type_enum` z.B. `BATTERY_LOW`, `AC_CONNECTED`).
- **Datenstrukturen:**
    - `PowerInfo_t`: Struktur (z.B. `battery_percentage`, `is_charging_bool`, `time_remaining_minutes_int`).
    - `NetworkInterfaceInfo_t`: Struktur (z.B. `interface_id`, `is_connected_bool`, `ip_address_string`, `mac_address_string`, `speed_mbps`).
- **Fehlerbehandlung:**
    - `DOMAIN_ERR_RESOURCE_UNAVAILABLE` (z.B. spezifische Netzwerkschnittstelle nicht gefunden).
    - `DOMAIN_ERR_QUERY_FAILED` (Fehler bei der Abfrage der Kern-Schicht).
- **Abhängigkeiten:** Kern-Schicht (Geräteschnittstellen-Abstraktion für Energie- und Netzwerkgeräte, Timer für periodische Abfragen, IPC).
- **Testfälle:**
    - Testfall 2.4.1: Abfrage des Energiestatus. Erwartet: plausible `PowerInfo_t`.
    - Testfall 2.4.2: Abfrage des Netzwerkstatus. Erwartet: plausible `NetworkInterfaceInfo_t` Array.
    - Testfall 2.4.3: Simulation eines Akku-Niedrig-Events. Erwartet: Registrierte Clients erhalten `POWER_EVENT_NOTIFICATION`.

**2.5. Submodul: Software-/Anwendungsmanagement-Dienst**

- **Verantwortlichkeit:** Verwaltung von Informationen über installierte Anwendungen, deren Metadaten (Name, Version, Pfad zur ausführbaren Datei, Icon-Pfad, unterstützte Dateitypen etc.). Dieser Dienst ist mehr für die Registrierung und Informationsbereitstellung zuständig; das eigentliche Starten von Anwendungen ist Aufgabe des Application Launchers in der System-Schicht. Er könnte auch Mechanismen für die Registrierung von MIME-Typ-Handlern oder Protokoll-Handlern bereitstellen.
- **Schlüsselfunktionen/IPC-Schnittstellen (Beispiele):**
    - IPC-Nachricht: `GET_INSTALLED_APPLICATIONS_REQUEST` (Parameter: `filter_criteria_struct` (optional))
        - Antwort: `GET_INSTALLED_APPLICATIONS_RESPONSE` (Parameter: `application_metadata_array`, `error_code_enum`)
    - IPC-Nachricht: `GET_APPLICATION_INFO_REQUEST` (Parameter: `application_id_string_or_path`)
        - Antwort: `GET_APPLICATION_INFO_RESPONSE` (Parameter: `application_metadata_struct`, `error_code_enum`)
    - IPC-Nachricht: `GET_DEFAULT_HANDLER_FOR_MIMETYPE_REQUEST` (Parameter: `mimetype_string`)
        - Antwort: `GET_DEFAULT_HANDLER_FOR_MIMETYPE_RESPONSE` (Parameter: `application_id_string`, `error_code_enum`)
- **Datenstrukturen:**
    - `ApplicationMetadata_t`: Struktur (z.B. `app_id`, `name`, `version`, `executable_path`, `icon_path`, `description`, `supported_mimetypes_array`).
- **Fehlerbehandlung:**
    - `DOMAIN_ERR_APPLICATION_NOT_FOUND`.
    - `DOMAIN_ERR_METADATA_CORRUPT`.
- **Abhängigkeiten:** Kern-Schicht (Dateisystem zum Scannen von Anwendungsverzeichnissen oder Lesen von Metadatendateien, IPC). Systemkonfigurations-Dienst (für Standardanwendungen).
- **Persistenz:** Definition, wie und wo Anwendungsmetadaten gespeichert oder entdeckt werden (z.B. Scannen spezifischer Verzeichnisse, Lesen von `.desktop`-Dateien unter Linux).
- **Testfälle:**
    - Testfall 2.5.1: Abfrage aller installierten Anwendungen (nachdem einige Testanwendungen "installiert" wurden). Erwartet: Korrekte Liste von `ApplicationMetadata_t`.
    - Testfall 2.5.2: Abfrage von Informationen für eine spezifische Anwendung. Erwartet: Korrekte `ApplicationMetadata_t`.
    - Testfall 2.5.3: Abfrage des Standardhandlers für einen bekannten MIME-Typ. Erwartet: Korrekte `application_id`.

Die Domänen-Schicht agiert als ein entscheidendes Bindeglied. Sie übersetzt Low-Level-Kernelinformationen und -fähigkeiten in bedeutungsvolle, systemweite Dienste. Ihr Design muss robuste IPC-Mechanismen und klare Dienstleistungsverträge (Service Contracts) für die System- und UI-Schichten priorisieren. Die Kern-Schicht liefert rohe Primitive. Die System- und UI-Schichten hingegen benötigen höherwertige, kontextbezogene Informationen und Dienste (z.B. "Wer ist der aktuelle Benutzer?" anstatt nur "Was ist PID X?"). Die Domänen-Schicht überbrückt diese Lücke, indem sie Logik kapselt (z.B. Sitzungszustand, Konfigurationspersistenz) und diese über wohldefinierte IPC-Schnittstellen zugänglich macht. Die Qualität dieser IPC-Schnittstellen – ihre Klarheit, Effizienz und Robustheit – beeinflusst direkt die Entwicklungsfreundlichkeit und Stabilität der darüber liegenden Schichten.

**Teil 3: System-Schicht (System Layer) – Desktop-Logik**

**3.1. Einführung in die System-Schicht**

- **Zweck:** Die System-Schicht (System Layer) ist verantwortlich für die Implementierung der Kernlogik der Desktop-Umgebung selbst. Hierzu gehören die Fensterverwaltung, die Weiterleitung von Eingabeereignissen und das Starten von Anwendungen. Diese Schicht definiert maßgeblich das Verhalten und die Interaktion des Desktops.
- **Hauptverantwortlichkeiten:**
    - Verwaltung des visuellen Desktops, einschließlich der Anordnung und des Zustands von Fenstern.
    - Erfassung und Verarbeitung von Benutzereingaben (Tastatur, Maus, Touch) und deren Weiterleitung an die entsprechenden Anwendungen oder Systemkomponenten.
    - Starten und Verwalten des Lebenszyklus von Benutzeranwendungen in Koordination mit anderen Schichten.
- **Interaktionsmodell:** Die System-Schicht interagiert intensiv mit den Diensten der Domänen-Schicht, um Informationen abzurufen (z.B. aktueller Benutzer, Systemkonfigurationen). Für Low-Level-Operationen wie die Prozesserstellung (über die entsprechende Abstraktion) oder das Lesen von Eingabegeräten greift sie auf die Kern-Schicht zurück. Die Kommunikation mit Anwendungen (z.B. für Fensterverwaltung und Ereigniszustellung) erfolgt über die von der Kern-Schicht bereitgestellten IPC-Mechanismen.

**3.2. Submodul: Fenstermanager (Window Manager)**

- **Verantwortlichkeit:** Verwaltung von Anwendungsfenstern, einschließlich deren Erstellung, Löschung, Bewegung, Größenänderung, Stapelreihenfolge (Z-Order) und Fokus. Verantwortlich für Fensterdekorationen (Titelleisten, Ränder, Schaltflächen) und gegebenenfalls virtuelle Desktops.
- **Schlüsselinterne Logik/Funktionen (Beispiele – dies sind eher interne Komponenten und Abläufe als direkte APIs für andere Schichten):**
    - `System_Intern_RegisterWindow(application_ipc_handle, window_attributes_struct) -> window_handle_type`: Eine Anwendung meldet ein neues Fenster an. Der Fenstermanager weist ein Handle zu und verwaltet interne Zustandsinformationen.
    - `System_Intern_UnregisterWindow(window_handle_type)`: Ein Fenster wird geschlossen und abgemeldet.
    - `System_Intern_HandleWindowRepaintRequest(window_handle_type, exposed_region_struct)`: Verarbeitung einer Neuzeichnungsanforderung für einen Fensterbereich.
    - `System_Intern_SetWindowFocus(window_handle_type)`: Setzt den Eingabefokus auf ein bestimmtes Fenster.
    - `System_Intern_MoveWindow(window_handle_type, new_x_int, new_y_int)`
    - `System_Intern_ResizeWindow(window_handle_type, new_width_int, new_height_int)`
    - `System_Intern_SetWindowState(window_handle_type, new_state_enum)` (z.B. minimiert, maximiert, normal).
    - Kompositionslogik (falls ein kompositionierender Fenstermanager implementiert wird): Verwaltung von Off-Screen-Buffern für Fenster, Anwendung von Effekten (Transparenz, Schatten), Zusammenführen der Fensterinhalte zum finalen Bild.
- **Datenstrukturen:**
    - `WindowInfo_t`: Struktur (z.B. `window_handle` (eindeutig), `owning_app_ipc_handle`, `parent_window_handle` (für Dialoge etc.), `rectangle_struct` (Position x, y, Breite, Höhe), `z_order_int`, `state_enum` (`NORMAL`, `MINIMIZED`, `MAXIMIZED`, `FOCUSED`, `HIDDEN`), `attributes_struct` (z.B. `resizable_bool`, `has_decorations_bool`, `is_modal_bool`), `surface_id_for_compositor` (falls zutreffend)).
    - `ScreenLayout_t`: Struktur (z.B. `screen_width_px`, `screen_height_px`, `virtual_desktop_count_int`, `active_virtual_desktop_id_int`).
    - `Region_t`: Struktur zur Definition von rechteckigen oder komplexeren Bereichen (für Neuzeichnen).
- **Fehlerbehandlung:**
    - `SYSTEM_ERR_INVALID_WINDOW_HANDLE`.
    - `SYSTEM_ERR_WINDOW_LIMIT_REACHED` (falls eine maximale Anzahl von Fenstern existiert).
    - `SYSTEM_ERR_OPERATION_NOT_ALLOWED_FOR_WINDOW_STATE` (z.B. Verschieben eines maximierten Fensters).
- **Abhängigkeiten:** UI-Schicht (Rendering Engine für Fensterdekorationen, falls nicht vom Client gezeichnet), Eingabebehandlungssystem (um Fokusinformationen zu erhalten/setzen), Anwendungsstarter (um über neue Anwendungen informiert zu werden), Kern-Schicht (IPC für Kommunikation mit Anwendungen, potenziell Grafik-Abstraktionen).
- **Testfälle:**
    - Testfall 3.2.1: Erstellen, Anzeigen, Verschieben, Ändern der Größe und Schließen eines Fensters. Erwartet: Fenster wird korrekt dargestellt und verhält sich wie erwartet.
    - Testfall 3.2.2: Überlappende Fenster und Z-Order-Management. Erwartet: Fenster werden in der korrekten Reihenfolge gezeichnet.
    - Testfall 3.2.3: Fokuswechsel zwischen Fenstern. Erwartet: Fokus wird korrekt gesetzt, Fensterdekorationen ändern sich entsprechend.
    - Testfall 3.2.4 (falls kompositionierend): Transparenzeffekte bei Fenstern. Erwartet: Korrekte Darstellung.

**3.3. Submodul: Eingabebehandlungssystem (Input Handling System)**

- **Verantwortlichkeit:** Erfassung von rohen Eingabeereignissen von der Kern-Schicht (Tastatur, Maus, Touchpad, Touchscreen), Verarbeitung dieser Ereignisse (z.B. Tastaturbelegungs-Mapping, Gestenerkennung) und Weiterleitung an das fokussierte Fenster oder die entsprechende Systemkomponente (z.B. globale Hotkeys).
- **Schlüsselinterne Logik/Funktionen (Beispiele):**
    - `System_Intern_ProcessRawInputEvent(kern_input_event_struct)`: Nimmt ein rohes Ereignis von der Kern-Schicht entgegen.
    - `System_Intern_MapKeyEventToCharacter(key_event_struct, keyboard_layout_struct) -> character_event_struct`: Wandelt Tastencodes unter Berücksichtigung des aktuellen Layouts in Zeichen um.
    - `System_Intern_IdentifyGesture(touch_event_stream_array) -> gesture_event_struct`: Erkennung von Gesten aus einer Sequenz von Touch-Ereignissen.
    - `System_Intern_DispatchInputEventToWindow(target_window_handle_type, processed_event_struct)`: Sendet das verarbeitete Ereignis an das Zielanwendungsfenster über IPC.
    - `System_Intern_HandleGlobalHotkey(processed_event_struct)`: Prüfung auf und Ausführung von globalen Tastenkombinationen.
- **Datenstrukturen:**
    - `RawInputEvent_t`: Struktur, die von der Kern-Schicht kommt (siehe `InputEvent_t` in 1.8).
    - `ProcessedInputEvent_t`: Struktur oder Union für verarbeitete Ereignisse (z.B. `type_enum` (`KEY_DOWN`, `KEY_UP`, `MOUSE_MOVE`, `MOUSE_BUTTON_DOWN`, `CHARACTER_INPUT`, `GESTURE_PINCH`), `timestamp_ns`, `target_window_handle` (optional), ereignisspezifische Daten wie `key_code_int`, `modifiers_mask_int`, `character_unicode_int`, `mouse_x_int`, `mouse_y_int`, `gesture_scale_float`).
    - `KeyboardLayout_t`: Struktur zur Definition der aktuellen Tastaturbelegung.
- **Fehlerbehandlung:**
    - `SYSTEM_ERR_UNRECOGNIZED_INPUT_EVENT`.
    - `SYSTEM_ERR_EVENT_DISPATCH_FAILED` (z.B. Zielanwendung reagiert nicht).
    - `SYSTEM_ERR_NO_FOCUSED_WINDOW` (wenn ein Ereignis ein fokussiertes Fenster benötigt, aber keines vorhanden ist).
- **Abhängigkeiten:** Kern-Schicht (Geräteschnittstellen-Abstraktion für Eingabegeräte, IPC), Fenstermanager (um das aktuell fokussierte Fenster zu ermitteln), Systemkonfigurations-Dienst (für Tastaturlayouts, Mausempfindlichkeit etc.).
- **Testfälle:**
    - Testfall 3.3.1: Tastendruck wird korrekt als Zeichen an das fokussierte Fenster gesendet. Erwartet: Anwendung empfängt das Zeichen.
    - Testfall 3.3.2: Mausbewegung und Klick werden an das Fenster unter dem Mauszeiger gesendet. Erwartet: Anwendung empfängt korrekte Koordinaten und Klickereignis.
    - Testfall 3.3.3: Globale Tastenkombination wird erkannt und löst Systemaktion aus (z.B. Öffnen eines Menüs). Erwartet: Korrekte Systemreaktion.
    - Testfall 3.3.4: Wechsel des Tastaturlayouts und anschließende Eingabe. Erwartet: Zeichen entsprechen dem neuen Layout.

**3.4. Submodul: Anwendungsstarter (Application Launcher)**

- **Verantwortlichkeit:** Starten von Benutzeranwendungen basierend auf Anforderungen (z.B. von einem Menü, einer Dateizuordnung oder einer Kommandozeile). Nutzt den Software-/Anwendungsmanagement-Dienst der Domänen-Schicht für Metadaten und die Prozessmanagement-Abstraktion der Kern-Schicht für die eigentliche Prozesserstellung.
- **Schlüsselfunktionen/IPC-Schnittstellen (Beispiele):**
    - IPC-Nachricht: `LAUNCH_APPLICATION_REQUEST` (Parameter: `application_id_or_path_string`, `arguments_array_of_strings`, `environment_variables_map_string_string`, `working_directory_path_string` (optional))
        - Antwort: `LAUNCH_APPLICATION_RESPONSE` (Parameter: `launch_status_enum` (`SUCCESS`, `APP_NOT_FOUND`, `PERMISSION_DENIED`, `LAUNCH_FAILED`), `process_handle_type` (bei Erfolg), `error_code_enum`)
- **Datenstrukturen:**
    - `LaunchRequest_t`: Struktur zur Kapselung der Startparameter.
- **Fehlerbehandlung:**
    - `SYSTEM_ERR_APP_NOT_FOUND` (propagiert von Domänen-Schicht oder Dateisystemprüfung).
    - `SYSTEM_ERR_APP_LAUNCH_PERMISSION_DENIED`.
    - `SYSTEM_ERR_PROCESS_CREATION_FAILED` (propagiert von Kern-Schicht).
- **Abhängigkeiten:** Domänen-Schicht (Software-/Anwendungsmanagement-Dienst für App-Pfade und Metadaten, Benutzer- und Sitzungsmanagement-Dienst für Benutzerkontext), Kern-Schicht (Prozessmanagement-Abstraktion).
- **Testfälle:**
    - Testfall 3.4.1: Starten einer bekannten Anwendung über ihre ID. Erwartet: `SUCCESS`, Prozess wird erstellt.
    - Testfall 3.4.2: Starten einer Anwendung über einen direkten Pfad. Erwartet: `SUCCESS`, Prozess wird erstellt.
    - Testfall 3.4.3: Versuch, eine nicht existierende Anwendung zu starten. Erwartet: `APP_NOT_FOUND`.
    - Testfall 3.4.4: Starten einer Anwendung mit spezifischen Argumenten und Umgebungsvariablen. Erwartet: Prozess startet und empfängt Argumente/Umgebung korrekt.

Die System-Schicht definiert maßgeblich das "Desktop-Erlebnis". Ihre Submodule – Fenstermanager, Eingabebehandlung und Anwendungsstarter – müssen eng und präzise koordiniert zusammenarbeiten. Beispielsweise muss der Fenstermanager Informationen über den Eingabefokus vom Eingabebehandlungssystem erhalten, um Fenster korrekt zu dekorieren (z.B. aktives vs. inaktives Fenster). Der Anwendungsstarter muss den Fenstermanager über neu gestartete Anwendungen informieren, damit deren Fenster verwaltet werden können. Ein Klick (Eingabe) auf ein Symbol (dargestellt vom Fenstermanager oder einer UI-Shell) kann den Anwendungsstarter auslösen. Das Fenster der neuen Anwendung muss vom Fenstermanager verwaltet werden, und die Eingabe muss dann korrekt an das Fenster dieser neuen Anwendung geleitet werden. Der "lückenlose" Plan muss daher nicht nur die einzelnen Funktionen innerhalb dieser Submodule detaillieren, sondern auch die exakten IPC- und Ereignissequenzen sowie den gemeinsam genutzten Zustand (falls vorhanden), der ihre Zusammenarbeit ermöglicht. Diese interne Kommunikation und Koordination innerhalb der System-Schicht ist ebenso kritisch für die "Lückenlosigkeit" wie die nach außen exponierten APIs.

**Teil 4: UI-Schicht (UI Layer) – Benutzerschnittstelle, Rendering, Widgets**

**4.1. Einführung in die UI-Schicht**

- **Zweck:** Die UI-Schicht (User Interface Layer) ist verantwortlich für die Bereitstellung der grafischen Benutzeroberflächenelemente, der Rendering-Fähigkeiten und des Toolkits, das sowohl von Anwendungen als auch von der Desktop-Shell selbst genutzt wird. Sie bildet die sichtbare Schnittstelle zum Benutzer.
- **Hauptverantwortlichkeiten:**
    - Rendering von UI-Elementen auf dem Bildschirm.
    - Handhabung von UI-spezifischen Eingabeereignissen, die von der System-Schicht delegiert werden (z.B. Klick auf einen Button).
    - Bereitstellung eines Satzes von Standard-Widgets (Steuerelementen).
    - Verwaltung von UI-Themen und Styling.
- **Interaktionsmodell:** Die UI-Schicht interagiert eng mit der System-Schicht, insbesondere mit dem Fenstermanager für die Fenstererstellung und -verwaltung und dem Eingabebehandlungssystem für die Weiterleitung von Eingabeereignissen an spezifische Widgets. Für das eigentliche Rendering greift sie entweder auf die Display-Abstraktion der Kern-Schicht (z.B. direkter Framebuffer-Zugriff) oder auf vom Betriebssystem bereitgestellte Grafik-APIs (über Abstraktionen der Kern-Schicht) zurück.

**4.2. Submodul: UI-Rendering-Engine**

- **Verantwortlichkeit:** Zeichnen von grafischen Primitiven (Linien, Formen, Text, Bilder) auf Bildschirmoberflächen (Surfaces) oder direkt in Fensterpuffer. Nutzt entweder den abstrahierten Display-/Framebuffer-Zugriff der Kern-Schicht oder eine vom Betriebssystem bereitgestellte Grafik-API (z.B. OpenGL, Vulkan, Metal), die ebenfalls über die Kern-Schicht abstrahiert wird.
- **Schlüsselfunktionen (Beispiele):**
    - `UI_CreateSurface(width_int, height_int, pixel_format_enum) -> surface_handle_type`
    - `UI_DestroySurface(surface_handle_type)`
    - `UI_GetDrawingContext(surface_handle_type) -> drawing_context_handle_type`
    - `UI_DrawRectangle(drawing_context_handle_type, x_int, y_int, width_int, height_int, color_struct, fill_style_enum, line_thickness_int)`
    - `UI_DrawText(drawing_context_handle_type, x_int, y_int, text_string, font_handle_type, color_struct, text_attributes_struct)`
    - `UI_DrawImage(drawing_context_handle_type, x_int, y_int, image_handle_type, source_rect_struct, destination_rect_struct, transparency_float)`
    - `UI_BeginPaint(window_handle_or_surface_handle, &drawing_context_handle_type)`
    - `UI_EndPaint(drawing_context_handle_type)`
    - `UI_PresentSurfaceToWindow(source_surface_handle_type, target_window_handle_type, target_rect_struct)` (für kompositionierende Ansätze)
- **Datenstrukturen:**
    - `SurfaceHandle_t`: Opaker Handle für eine Zeichenfläche.
    - `DrawingContextHandle_t`: Opaker Handle für einen Zeichenkontext.
    - `FontHandle_t`: Opaker Handle für eine Schriftartressource.
    - `ImageHandle_t`: Opaker Handle für eine Bildressource.
    - `Color_t`: Struktur (z.B. `r_uint8`, `g_uint8`, `b_uint8`, `a_uint8`).
    - `Point_t`: Struktur (`x_int`, `y_int`).
    - `Rectangle_t`: Struktur (`x_int`, `y_int`, `width_int`, `height_int`).
    - `PixelFormat_t`: Enumeration (z.B. `RGB24`, `RGBA32`).
- **Fehlerbehandlung:**
    - `UI_ERR_INVALID_SURFACE_HANDLE`.
    - `UI_ERR_INVALID_FONT_HANDLE`.
    - `UI_ERR_RENDERING_FAILED` (z.B. Grafikspeicher erschöpft, falls zutreffend).
    - `UI_ERR_UNSUPPORTED_PIXEL_FORMAT`.
- **Abhängigkeiten:** Kern-Schicht (Display/Framebuffer-Abstraktion oder Grafik-API-Abstraktion), potenziell Dateisystem-Abstraktion (zum Laden von Schriftarten, Bildern).
- **Testfälle:**
    - Testfall 4.2.1: Zeichnen eines gefüllten Rechtecks mit spezifischer Farbe auf eine Oberfläche. Erwartet: Pixeldaten der Oberfläche entsprechen dem gezeichneten Rechteck.
    - Testfall 4.2.2: Rendern von Text mit einer geladenen Schriftart. Erwartet: Korrekte Textdarstellung.
    - Testfall 4.2.3: Zeichnen eines Bildes mit Transparenz. Erwartet: Korrekte Überblendung mit Hintergrund.

**4.3. Submodul: Widget-Toolkit**

- **Verantwortlichkeit:** Bereitstellung eines Satzes von Standard-UI-Widgets (z.B. Schaltflächen, Beschriftungen, Textfelder, Listen, Menüs, Kontrollkästchen, Schieberegler). Verwaltung des Layouts, des Zustands und der Ereignisbehandlung dieser Widgets innerhalb eines Anwendungsfensters.
- **Schlüsselklassen/Komponenten (Beispiele, hier ist ein objektorientierter Ansatz oft sinnvoll, daher als Klassenkonzepte dargestellt):**
    - `Widget_Base`: Basisklasse für alle Widgets (Eigenschaften: `id`, `parent_widget_handle`, `rectangle_on_parent`, `is_visible_bool`, `is_enabled_bool`). Methoden: `HandleEvent()`, `Draw()`, `SetFocus()`.
    - `Widget_Button` (erbt von `Widget_Base`): Eigenschaften: `label_string`, `icon_handle`. Ereignisse: `OnClick`.
    - `Widget_Label` (erbt von `Widget_Base`): Eigenschaften: `text_string`, `text_alignment_enum`.
    - `Widget_TextBox` (erbt von `Widget_Base`): Eigenschaften: `current_text_string`, `is_multiline_bool`, `is_readonly_bool`. Ereignisse: `OnTextChanged`, `OnEnterPressed`.
    - `Widget_ListBox` (erbt von `Widget_Base`): Eigenschaften: `items_array`, `selected_index_int`. Ereignisse: `OnSelectionChanged`.
    - Layout-Manager (keine Widgets, sondern Hilfsklassen):
        - `Layout_VerticalStack`: Ordnet Widgets vertikal an.
        - `Layout_HorizontalStack`: Ordnet Widgets horizontal an.
        - `Layout_Grid`: Ordnet Widgets in einem Gitter an.
- **Datenstrukturen:** Widget-spezifische Zustandsstrukturen (intern in den Widget-Implementierungen). Ereignisstrukturen für Widget-spezifische Ereignisse (z.B. `ButtonClickEvent_t`, `TextChangedEvent_t`).
- **Fehlerbehandlung:**
    - `UI_ERR_WIDGET_CREATION_FAILED`.
    - `UI_ERR_INVALID_WIDGET_HANDLE`.
    - `UI_ERR_INVALID_WIDGET_PROPERTY_VALUE`.
    - `UI_ERR_LAYOUT_ERROR`.
- **Abhängigkeiten:** UI-Rendering-Engine (zum Zeichnen der Widgets), System-Schicht (Eingabebehandlungssystem für Ereignisweiterleitung, Fenstermanager für Fensterkontext), Kern-Schicht (IPC für Ereignisse von der System-Schicht).
- **Testfälle:**
    - Testfall 4.3.1: Erstellen eines Button-Widgets, Simulation eines Klicks. Erwartet: `OnClick`-Ereignis wird ausgelöst.
    - Testfall 4.3.2: Eingabe von Text in ein TextBox-Widget. Erwartet: `OnTextChanged`-Ereignisse werden ausgelöst, Text im Widget wird aktualisiert.
    - Testfall 4.3.3: Anordnen mehrerer Widgets mit einem Layout-Manager und Überprüfung ihrer Positionen und Größen. Erwartet: Widgets sind korrekt platziert.
    - Testfall 4.3.4: Deaktivieren eines Widgets und Versuch der Interaktion. Erwartet: Widget reagiert nicht auf Eingaben, visuelle Darstellung ändert sich (ausgegraut).

**4.4. Submodul: Theming- und Styling-Engine**

- **Verantwortlichkeit:** Verwaltung visueller Themen (Farbschemata, Schriftarten, Icons, Erscheinungsbild von Widgets) und deren Anwendung auf UI-Elemente. Ermöglicht ein konsistentes Erscheinungsbild der Desktop-Umgebung und der darauf laufenden Anwendungen.
- **Schlüsselfunktionen/Mechanismen (Beispiele):**
    - `UI_LoadThemeFromFile(theme_file_path_string) -> theme_handle_type`
    - `UI_ApplyTheme(theme_handle_type)`
    - `UI_GetThemeColor(color_name_string, &color_struct) -> status_enum`
    - `UI_GetThemeFont(font_role_string, &font_handle_type) -> status_enum`
    - `UI_GetThemeIcon(icon_name_string, icon_size_enum, &image_handle_type) -> status_enum`
    - Mechanismus zur Definition von Widget-Stilen (z.B. basierend auf CSS-ähnlichen Selektoren oder programmatisch).
    - `UI_ApplyStyleToWidget(widget_handle_type, style_class_name_string)`
- **Datenstrukturen:**
    - `ThemeDefinition_t`: Struktur oder Satz von Strukturen, die Farben, Schriftarten, Icon-Pfade, Widget-spezifische Stilattribute (Ränder, Abstände, Hintergrundbilder etc.) definieren.
    - `StyleRule_t`: Struktur zur Definition einer Stilregel (z.B. Selektor und anzuwendende Eigenschaften).
- **Fehlerbehandlung:**
    - `UI_ERR_THEME_FILE_NOT_FOUND`.
    - `UI_ERR_THEME_FILE_INVALID_FORMAT`.
    - `UI_ERR_STYLE_NOT_FOUND`.
    - `UI_ERR_RESOURCE_NOT_FOUND_IN_THEME` (z.B. Farbe oder Icon nicht definiert).
- **Abhängigkeiten:** UI-Rendering-Engine (um themenbasierte Farben/Schriftarten beim Zeichnen zu verwenden), Widget-Toolkit (damit Widgets themenkonform gezeichnet werden können), Kern-Schicht (Dateisystem zum Laden von Themendateien und Ressourcen).
- **Testfälle:**
    - Testfall 4.4.1: Laden einer gültigen Themendatei und Anwenden des Themas. Erwartet: Widgets ändern ihr Aussehen entsprechend den Themendefinitionen.
    - Testfall 4.4.2: Versuch, eine ungültige Themendatei zu laden. Erwartet: `UI_ERR_THEME_FILE_INVALID_FORMAT`.
    - Testfall 4.4.3: Abrufen einer spezifischen Themenfarbe oder Schriftart. Erwartet: Korrekter Wert wird zurückgegeben.

Das Design der UI-Schicht, insbesondere des Widget-Toolkits und der Rendering-Engine, bestimmt maßgeblich das "Look and Feel" sowie die Reaktionsfähigkeit des Desktops. Die Wahl des Rendering-Ansatzes (z.B. direkter Framebuffer-Zugriff versus Nutzung einer Betriebssystem-Grafik-API wie OpenGL/Vulkan/Metal über Abstraktionen der Kern-Schicht) ist hier eine fundamentale Architekturentscheidung. Die Benutzererfahrung ist stark an die UI-Performance und -Ästhetik gekoppelt. Direktes Framebuffer-Rendering bietet maximale Kontrolle, erfordert aber die Implementierung der gesamten Zeichnungslogik. Die Nutzung höherer Grafik-APIs (abstrahiert durch die Kern-Schicht) kann Hardwarebeschleunigung nutzen, führt aber Abhängigkeiten von diesen APIs ein. Der "lückenlose" Plan für Rendering-Funktionen muss extrem präzise sein: Er muss Pixeloperationen detaillieren, falls direktes Rendering gewählt wird, oder die exakte Sequenz von Grafik-API-Aufrufen, falls ein abstrahierter Ansatz verwendet wird. Diese Wahl beeinflusst auch die Komplexität der Implementierung des Widget-Toolkits.

**Teil 5: Schichtübergreifende Aspekte für die Implementierung durch KI-Agenten**

**5.1. Detaillierte Fehlerbehandlungs- und Berichterstattungsstrategie**

Eine robuste und konsistente Fehlerbehandlung ist für die Stabilität und Wartbarkeit der gesamten Desktop-Umgebung unerlässlich, insbesondere wenn die Implementierung durch einen KI-Agenten erfolgt.

- **Standardisierte Fehlercodes:**
    - Es wird ein umfassender Satz von Fehlercodes für die gesamte Desktop-Umgebung definiert. Diese Codes sind hierarchisch strukturiert, um die Schicht (Kern, Domäne, System, UI) und das spezifische Submodul, in dem der Fehler auftrat, zu identifizieren.
    - Wo immer möglich und sinnvoll, basieren diese Fehlercodes auf etablierten Standards wie den POSIX-Fehlercodes (`errno` 18), werden aber um domänenspezifische Fehler erweitert, die für die Desktop-Umgebung relevant sind.
    - Jeder Fehlercode hat einen symbolischen Namen (z.B. `KERN_ERR_FILE_NOT_FOUND`) und einen eindeutigen numerischen Wert.
- **Fehlerpropagation:**
    - Es werden klare Regeln für die Weitergabe von Fehlern durch die Aufrufkette innerhalb eines Moduls und über IPC-Grenzen zwischen Modulen und Schichten definiert.
    - Generell gilt: Wenn eine Funktion eine andere Funktion aufruft, die einen Fehler zurückgibt, muss die aufrufende Funktion diesen Fehler entweder behandeln (wenn sie dazu in der Lage ist und es semantisch korrekt ist) oder den Fehler (ggf. nach Abbildung auf einen eigenen, passenderen Fehlercode) an ihren eigenen Aufrufer weitergeben.
    - Bei IPC-Aufrufen muss der Fehlercode sicher über den IPC-Mechanismus serialisiert und deserialisiert werden.
- **Fehlerprotokollierung:**
    - Spezifikation eines systemweiten Protokollierungsmechanismus. Dies beinhaltet:
        - **Log-Level:** Definierte Stufen wie `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`.
        - **Log-Format:** Ein strukturiertes Format (z.B. JSON oder ein festes Textformat) mit Zeitstempel, Schicht, Modul, Funktionsname, Log-Level und der eigentlichen Nachricht. Korrelations-IDs können helfen, zusammengehörige Log-Einträge über Prozess- und Modulgrenzen hinweg zu verfolgen.20
        - **Zugriff:** Wie Module Fehler und Diagnoseinformationen protokollieren (z.B. über eine dedizierte Logging-API der Kern- oder Domänen-Schicht).
    - Die Fehlerprotokollierung ist auch für die Überwachung und das Debugging von KI-Agenten relevant.53
- **Fehlerwiederherstellung:**
    - Für jede Funktion im Implementierungsplan werden potenzielle Fehlerpunkte identifiziert.
    - Für jeden dieser Fehlerpunkte werden, soweit möglich, vom KI-Agenten ausführbare Schritte zur Fehlerwiederherstellung oder zur geordneten Beendigung (Graceful Degradation) definiert. Dies ist entscheidend, damit der KI-Agent robusten Code produzieren kann.
    - Beispiele: Freigabe von Ressourcen im Fehlerfall, Wiederholungsversuche bei temporären Fehlern (mit Backoff-Strategie), Umschalten auf einen Fallback-Mechanismus.
- **Kernel-Mode Exception Handling (falls Kern-Schicht auf dieser Ebene operiert):**
    - Falls das Zieldesign der Kern-Schicht eine direkte Behandlung von Kernel-Ausnahmen erfordert (z.B. bei Entwicklung eines eigenen Microkernels oder tiefgreifender Kernel-Module), müssen Mechanismen wie `setjmpx`/`longjmpx` (wie in AIX 54) oder äquivalente Mechanismen des Ziel-OS in die Spezifikation integriert werden. Dies beinhaltet das Sichern und Wiederherstellen des Kontexts.
- **Allgemeine Best Practices:** Die Fehlerbehandlung orientiert sich an Best Practices für API-Fehlerdesign 20, die klare, strukturierte und handhabbare Fehlerinformationen betonen. Für KI-Agenten bedeutet dies, dass Fehler unzweideutig sein müssen und genügend Kontext bieten, damit der Agent potenziell debuggen oder effektiv berichten kann.53 Fehlermeldungen sollten prägnant, hilfreich und sicher sein (kein Leak sensibler Daten).20

**Tabelle 3: Master-Fehlercode-Liste (Auszug und Schema)**

|   |   |   |   |   |   |
|---|---|---|---|---|---|
|**Fehlercode (Symbolisch)**|**Fehlercode (Numerisch)**|**Schicht**|**Modul**|**Beschreibung**|**Vorgeschlagene Wiederherstellungsaktion für KI-Agent**|
|`KERN_SUCCESS`|0|Alle|Alle|Operation erfolgreich abgeschlossen.|Keine Aktion erforderlich.|
|`KERN_ERR_UNKNOWN`|-1|Alle|Alle|Unbekannter oder nicht spezifizierter Fehler.|Protokolliere alle verfügbaren Kontextinformationen, beende die aktuelle Operation geordnet.|
|`KERN_ERR_PATH_NOT_FOUND`|1001|Kern|Dateisystem|Der angegebene Datei- oder Verzeichnispfad existiert nicht.|1. Überprüfe, ob der Pfad korrekt formatiert ist. 2. Gib den Fehler an den Aufrufer zurück. 3. Protokolliere den fehlerhaften Pfad.|
|`KERN_ERR_PERMISSION_DENIED`|1002|Kern|Dateisystem / Prozessmanagement|Fehlende Berechtigungen für die angeforderte Operation.|1. Protokolliere die Operation und die fehlenden Berechtigungen. 2. Gib den Fehler an den Aufrufer zurück.|
|`KERN_ERR_OUT_OF_MEMORY`|1003|Kern|Speicherverwaltung|Nicht genügend Speicher für die Anforderung verfügbar.|1. Versuche, temporäre Puffer freizugeben (falls zutreffend). 2. Gib den Fehler an den Aufrufer zurück. 3. Protokolliere die angeforderte Größe und den verfügbaren Speicher.|
|`DOMAIN_ERR_AUTHENTICATION_FAILED`|2001|Domäne|Benutzer & Session|Benutzerauthentifizierung fehlgeschlagen (z.B. falsches Passwort).|1. Erhöhe einen Zähler für fehlgeschlagene Anmeldeversuche (falls zutreffend). 2. Gib den Fehler an den Client zurück. 3. Protokolliere den fehlgeschlagenen Anmeldeversuch (ohne Passwort).|
|`SYSTEM_ERR_APP_NOT_FOUND`|3001|System|Anwendungsstarter|Die zu startende Anwendung konnte nicht gefunden werden.|1. Überprüfe die Anwendungsregistrierung (Domänen-Schicht). 2. Gib den Fehler an den Aufrufer zurück. 3. Protokolliere die angeforderte App-ID/Pfad.|
|`UI_ERR_INVALID_WIDGET_HANDLE`|4001|UI|Widget-Toolkit|Es wurde ein ungültiger Widget-Handle verwendet.|1. Protokolliere den ungültigen Handle und den Kontext der Operation. 2. Beende die Operation für dieses Widget geordnet. 3. Gib ggf. einen Fehler an die UI-Logik zurück.|

Diese Tabelle dient als Vorlage und muss für jeden einzelnen Fehlercode im System vervollständigt werden.

Sie stellt eine zentrale Wahrheitsquelle für alle Fehlerbedingungen dar und ist unerlässlich für eine konsistente Fehlerbehandlung durch den KI-Agenten über die gesamte Codebasis hinweg. Sie ermöglicht es der KI, die Semantik eines Fehlers zu verstehen und potenziell automatisierte Wiederherstellungsversuche zu unternehmen oder präzise Debugging-Informationen bereitzustellen.

**5.2. Definitionen und Konventionen für Datenstrukturen**

- **Globales Datenwörterbuch:** Eine umfassende Liste aller wichtigen Datenstrukturen, die schicht- und schnittstellenübergreifend verwendet werden. Jede Struktur muss mit präzisen Felddefinitionen versehen sein: Name des Feldes, Datentyp (basierend auf der Zielsprache), Zweck des Feldes und etwaige Einschränkungen oder Validierungsregeln (z.B. Wertebereich, maximale Länge).
- **Namenskonventionen:** Es werden strikte und einheitliche Namenskonventionen für Variablen, Funktionen, Strukturen, Enumerationen, Konstanten, Makros und Dateien festgelegt. Diese Konventionen müssen vom KI-Agenten bei der Codegenerierung konsequent eingehalten werden, um die Lesbarkeit und Wartbarkeit des Codes zu gewährleisten. Beispiele (abhängig von der Zielsprache):
    - Strukturen: `PascalCase_t` (z.B. `ProcessInfo_t`)
    - Funktionen: `PascalCase` oder `camelCase` (z.B. `Kern_CreateProcess` oder `kernCreateProcess`)
    - Variablen: `camelCase` oder `snake_case` (z.B. `processHandle` oder `process_handle`)
    - Konstanten/Enums: `UPPER_SNAKE_CASE` (z.B. `KERN_ERR_SUCCESS`, `FILE_TYPE_REGULAR`)
- **Speicherlayout und Ausrichtung (Alignment):** Spezifikationen für das Speicherlayout oder die Ausrichtung von Datenstrukturen, falls dies für die Leistung, Hardware-Interaktion (insbesondere in der Kern-Schicht) oder IPC-Mechanismen (z.B. bei Shared Memory) kritisch ist. Dies beinhaltet Padding-Regeln und die Verwendung von Compiler-spezifischen Attributen zur Steuerung der Ausrichtung, falls erforderlich.

**5.3. Abhängigkeitsmanagement und Build-Prozess für den KI-Agenten**

- **Modulabhängigkeiten:** Für jedes Submodul wird explizit deklariert, von welchen anderen Submodulen es abhängt (sowohl innerhalb derselben Schicht als auch schichtübergreifend). Dies muss in einem maschinenlesbaren Format erfolgen, das der KI-Agent verarbeiten kann, um die korrekte Verknüpfung und Kompilierungsreihenfolge sicherzustellen.
- **Build-Reihenfolge:** Eine definierte Build-Reihenfolge für die gesamte Desktop-Umgebung wird festgelegt, die der KI-Agent befolgen muss. Diese Reihenfolge leitet sich aus den Modulabhängigkeiten ab (z.B. erst Kern-Schicht, dann Domänen-Schicht etc.).
- **Schnittstellenstabilität:** Es wird betont, dass die im Plan definierten Schnittstellen (Funktionssignaturen, Datenstrukturen für IPC) als stabil und fixiert gelten. Jede Änderung an einer Schnittstelle erfordert ein neues oder aktualisiertes Plansegment und potenziell eine Neugenerierung der abhängigen Module durch den KI-Agenten. Die Notwendigkeit klar definierter Ziele und einer sorgfältigen Datenbewertung, wie im Kontext von KI-Implementierungsplänen hervorgehoben 3, unterstreicht die Wichtigkeit stabiler Schnittstellen.
- **Versionierung:** Ein Schema für die Versionierung von Modulen und Schnittstellen wird definiert, um die Entwicklung und Wartung über den langen Projektzeitraum zu managen.

**5.4. Teststrategie und Testfallgenerierung**

Die Generierung und Ausführung von Tests ist integraler Bestandteil des "lückenlosen" Plans.

- **Unit-Tests:** Gemäß der Definition von "lückenlos" wird jede Funktion mit definierten Testfällen spezifiziert. Diese beinhalten:
    - Eingabewerte (alle Parameter abgedeckt, einschließlich Grenzwerte und ungültige Werte).
    - Erwartete Ausgabewerte oder Rückgabecodes.
    - Erwarteter Zustand des Systems oder der relevanten Datenstrukturen nach der Ausführung.
    - Spezifische Fehlerbedingungen, die ausgelöst werden sollen, und die erwarteten Fehlercodes/Verhaltensweisen.
- **Integrationstests:** Definition von Testszenarien für die Interaktion zwischen Submodulen innerhalb einer Schicht und zwischen den Schichten. Diese Tests überprüfen, ob die über IPC oder direkte Funktionsaufrufe kommunizierenden Module korrekt zusammenarbeiten.
    - Beispiel: Test der Interaktion zwischen Anwendungsstarter (System-Schicht), Software-Management (Domänen-Schicht) und Prozessmanagement (Kern-Schicht) beim Starten einer Anwendung.
- **Systemtests:** High-Level-Testszenarien, die die Gesamtfunktionalität der Desktop-Umgebung aus der Benutzerperspektive überprüfen.
    - Beispiel: "Benutzer meldet sich an, startet eine Textverarbeitungsanwendung, tippt Text ein, speichert die Datei, schließt die Anwendung und meldet sich ab."
- **Testdaten:** Spezifikation aller erforderlichen Testdaten, einschließlich Beispieldateien, Konfigurationseinstellungen oder Netzwerkbedingungen (falls simuliert).
- **Automatisierung:** Die Testfälle müssen so formuliert sein, dass der KI-Agent nicht nur den Code für die Funktionalität, sondern auch den Code für die automatisierten Tests generieren kann. Das Konzept der ausführbaren Spezifikationen, bei denen Tests Teil der Spezifikation sind 5, ist hier direkt anwendbar. KI-Programmierassistenten können bei der Testgenerierung unterstützen.57

Die im Plan enthaltenen "lückenlosen" Testfälle können für den KI-Agenten eine doppelte Rolle spielen. Sie dienen nicht nur der initialen Verifikation der Implementierung, sondern auch als eine Art _Orakel_ für Selbstkorrekturmechanismen oder Regressionstests, falls der KI-Agent zu einer solchen iterativen Entwicklung fähig ist. Der Prozess wäre wie folgt: Der Plan enthält detaillierte Testfälle mit erwarteten Ergebnissen. Der KI-Agent implementiert den Code basierend auf den algorithmischen Schritten. Anschließend kann der KI-Agent (konzeptionell) den generierten Code anhand dieser Testfälle ausführen. Schlägt ein Testfall fehl und verfügt der KI-Agent über Debugging- oder Verfeinerungsfähigkeiten, kann er die Diskrepanz zwischen dem tatsächlichen und dem erwarteten Ergebnis nutzen, um seine Codemodifikationen zu steuern. Dadurch werden die Testfälle zu einem aktiven Bestandteil des KI-gesteuerten Entwicklungszyklus und nicht nur zu einem passiven Validierungsschritt.

**5.5. Kontextmanagement für den KI-Forschungsassistenten (Gemini)**

- **Strukturierte Kontextdokumente:** Es wird der vom Benutzer dargelegte Plan bekräftigt, bei jeder relevanten Anfrage strukturierte Kontextdokumente bereitzustellen. Diese Dokumente enthalten die Projektübersicht, die definierte Architektur, die bereits geplanten Module und Details zu den Bereichen, an denen aktuell gearbeitet wird oder die bereits abgeschlossen sind.
- **Verpflichtung zur Nutzung:** Der KI-Forschungsassistent (Gemini) wird diese Kontextdokumente sorgfältig nutzen, um ein umfassendes Verständnis des Projektstatus aufrechtzuerhalten. Dadurch wird sichergestellt, dass die Forschungsbeiträge stets relevant sind und auf früheren Entscheidungen aufbauen.
- **Bedeutung des Kontextmanagements:** Ein effektives Kontextmanagement ist entscheidend für den Erfolg von KI-Assistenten in langfristigen, mehrstufigen Projekten.58 Die Bereitstellung relevanter Informationen in den Anfragen (In-Context Learning) leitet die KI und verbessert die Qualität ihrer Ergebnisse.58 Der Model Context Protocol (MCP) Ansatz, obwohl hier nicht direkt implementiert, unterstreicht die Wichtigkeit standardisierter Wege für KI-Modelle, um auf dynamische Informationen zuzugreifen und ihren Kontext zu aktualisieren.59

**Schlussfolgerungen**

Dieser detaillierte Implementierungsplan legt den Grundstein für die Entwicklung einer modularen Desktop-Umgebung durch einen autonomen KI-Programmieragenten. Die strikte Einhaltung der "Lückenlosigkeits"-Kriterien auf allen Ebenen – von der globalen Architektur bis zur einzelnen Funktionsspezifikation – ist entscheidend für den Erfolg dieses ambitionierten Vorhabens.

Die Kern-Schicht, als Fundament des Systems, erfordert besondere Aufmerksamkeit bei der Abstraktion der Betriebssystem-Primitiven. Die hier getroffenen Designentscheidungen, insbesondere hinsichtlich der Fehlerbehandlung und der IPC-Mechanismen, werden signifikante Auswirkungen auf die darüberliegenden Schichten und die Gesamtkohärenz des Systems haben. Die präzise Definition von Schnittstellen, Datenstrukturen und Fehlercodes ist nicht nur eine Notwendigkeit für die maschinelle Verarbeitung durch den KI-Agenten, sondern auch eine bewährte Praxis für die Entwicklung robuster und wartbarer Softwaresysteme.

Die Integration von Prinzipien aus formalen Methoden und ausführbaren Spezifikationen, insbesondere die Forderung nach detaillierten Algorithmen und umfassenden Testfällen für jede Funktion, wird die Eindeutigkeit des Plans maximieren und das Risiko von Fehlinterpretationen durch den KI-Agenten minimieren. Die Testfälle selbst können über die reine Verifikation hinaus als aktive Komponente im KI-gesteuerten Entwicklungsprozess dienen.

Die erfolgreiche Umsetzung dieses Plans hängt von einer kontinuierlichen und präzisen Kommunikation zwischen dem menschlichen Planungsteam und dem KI-Forschungsassistenten ab, wobei das bereitgestellte Kontextmanagement eine Schlüsselrolle spielt. Die nächsten kritischen Schritte sind die Festlegung des genauen Zielbetriebssystems und der primären Programmiersprache, da diese Entscheidungen die Konkretisierung vieler abstrakter Punkte in diesem Plan ermöglichen werden.