---
read_when:
    - Sie möchten, dass Codex-Desktop- oder CLI-Sitzungen in OpenClaw angezeigt werden
    - Sie müssen eine gespeicherte oder inaktive lokale Codex-Sitzung verzweigen oder archivieren
    - Sie stellen Codex-Sitzungen und den Transkriptverlauf von gekoppelten Nodes bereit
sidebarTitle: Codex supervision
summary: Nicht archivierte native Codex-Sitzungen und paginierte Transkripte über OpenClaw-Nodes hinweg durchsuchen
title: Codex-Sitzungen überwachen
x-i18n:
    generated_at: "2026-07-26T18:28:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f365e3207dff092c3dfd8f7588d60d70a16f0cce484991eb4ab3fc0bd15f8051
    source_path: plugins/codex-supervision.md
    workflow: 16
---

Die Codex-Überwachung ist eine optionale Funktion des offiziellen `codex`-Plugins. Sie
zeigt nicht archivierte Quellsitzungen aus Codex CLI, VS Code, Atlas und ChatGPT vom
Gateway-Computer und von angemeldeten gekoppelten Computern in der normalen
Sitzungsseitenleiste und im Chat-Bereich an.

Die Erstveröffentlichung hält den Zuständigkeitsbereich bewusst eng:

- Eine gespeicherte oder inaktive lokale Sitzung kann aus ihrem begrenzten
  gespeicherten Verlauf von Benutzer- und Assistentennachrichten einen modellgebundenen
  OpenClaw-Chat erstellen. Die erste Nachricht startet einen nativen Snapshot-Fork und
  anschließend den vollständigen Codex-Harness-Thread mit genau dem Modell und Provider,
  die Codex App Server für diesen Fork ausgewählt hat. Bei späteren Durchläufen wird das
  gespeicherte Paar des kanonischen nativen Threads wiederhergestellt, während die
  überwachte Bindung verhindert, dass OpenClaw eine andere Runtime, ein anderes Modell
  oder einen Fallback einsetzt. Eine separate native Codex-Steuerung kann dieses
  gespeicherte Paar weiterhin ändern. Ein bereits erstellter Branch öffnet seinen
  vorhandenen Chat.
- Bei einer gespeicherten Sitzung, die von einem anderen Codex-Prozess erkannt
  wurde, ist die aktuelle Live-Aktivität unbekannt. Sie kann verzweigt oder erst
  archiviert werden, nachdem der Operator bestätigt hat, dass sie von keinem anderen
  Codex-Client verwendet wird.
- Eine aktive Quelle bleibt sichtbar, kann jedoch weder einen Branch erstellen
  noch archiviert werden, bis ihr aktueller Durchlauf abgeschlossen ist. Wenn bereits ein
  überwachter Chat vorhanden ist, bleibt **Chat öffnen** verfügbar.
- Eine Sitzung auf einem gekoppelten Node stellt ihr gespeichertes Transkript über
  begrenzte, cursorpaginierte Lesevorgänge des App Server bereit. Die Remote-Fortsetzung
  erfordert eine zukünftige Streaming-Node-Bridge; die Remote-Archivierung erfordert
  zusätzlich eine Lease für die Runner-Zuständigkeit oder eine gleichwertige Absicherung.
- Archivierte Sitzungen werden nicht aufgeführt. Eine gespeicherte oder inaktive
  lokale Sitzung kann erst archiviert werden, nachdem der Operator bestätigt hat, dass sie
  von keinem anderen Codex-Client verwendet wird.

## Bevor Sie beginnen

- Installieren Sie das offizielle `@openclaw/codex`-Plugin auf dem Gateway. Die
  OpenClaw-App für macOS kann es installieren, wenn Sie Codex-Funktionen aktivieren;
  CLI-Installationen können `openclaw plugins install @openclaw/codex` ausführen.
- Installieren Sie Codex Desktop oder Codex CLI auf jedem Computer, dessen
  Sitzungen Sie auflisten möchten, und melden Sie sich dort an.
- Koppeln Sie Remote-Computer als OpenClaw-Nodes. Jeder Computer muss die
  Funktion lokal aktivieren; die ausschließliche Aktivierung der Überwachung auf dem
  Gateway autorisiert keinen anderen Node.
- Verwenden Sie ein vom Eigentümer kontrolliertes Gateway. Sitzungstitel,
  Arbeitsverzeichnisse und Git-Branches können vertrauliche Projektinformationen
  offenlegen.

## Überwachung aktivieren

Die geführte Einrichtung über `openclaw onboard` und die Ersteinrichtung unter macOS versuchen,
die Codex-Überwachung zu installieren und zu aktivieren, nachdem eine native
Codex-Installation erkannt und das ausgewählte Inferenz-Backend erfolgreich
aktiviert wurde. Codex muss nicht das primäre Backend sein. Die Überwachung wird
verfügbar, wenn diese opportunistische Plugin-Aktivierung erfolgreich ist. Die
Verfügbarkeit von App Server wird geprüft, wenn die Überwachung erstmals eine
Verbindung herstellt. Eine explizite Deaktivierung des Codex-Plugins oder eine
Richtliniensperre verhindert die opportunistische Aktivierung, und ein vorhandenes
explizites `supervision.enabled: false` deaktiviert Überwachungstools für Agenten; der
Operatorkatalog bleibt immer registriert, wenn das Codex-Plugin aktiv ist, sofern
`sessionCatalog.enabled: false` ihn nicht deaktiviert. Dieser separate Schalter lässt den
Codex-Provider, das Harness und die agentenseitige Überwachungsrichtlinie unverändert
und entfernt zugleich die Befehle zum Auflisten und Lesen des Katalogs gekoppelter
Nodes von diesem Host.
Vorhandene Installationen können dieselbe Funktion manuell aktivieren:

Aktivieren Sie das `codex`-Plugin und seine Überwachungsfunktion in `openclaw.json`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          supervision: {
            enabled: true,
          },
        },
      },
    },
  },
}
```

Wenn `plugins.allow` vorhanden ist, schließen Sie `codex` ein. Starten Sie das Gateway
nach einer Änderung der Plugin-Aktivierung neu.

Ohne explizite `appServer`-Verbindungseinstellungen verwendet die Überwachung eine
separate verwaltete stdio-Überwachungsverbindung zum nativen Codex-Benutzerverzeichnis.
Das gewöhnliche Codex-Harness bleibt standardmäßig agentenbezogen. Dadurch werden native
Sitzungen in beiden Apps sichtbar, ohne dass gewöhnliche OpenClaw-Durchläufe nativen
Codex-Zustand gemeinsam verwenden. Legen Sie `appServer.homeScope: "user"` explizit fest, wenn das Harness
diesen Zustand ebenfalls gemeinsam verwenden soll. Die Überwachung berücksichtigt explizite
`appServer`-Verbindungseinstellungen, anstatt sie durch die lokale Standardeinstellung
für das Benutzerverzeichnis zu ersetzen.

Ein aus der Seitenleistengruppe **Codex** übernommener Chat ist keine gewöhnliche
Harness-Sitzung. Seine private Überwachungsbindung verwendet die Überwachungsverbindung
für das Lesen der Quelle, die Erstellung des kanonischen Branches, die Einspeisung des
Verlaufs und jeden späteren Durchlauf. Bei der lokalen Standardverbindung bleiben dadurch
das native Codex-Benutzerverzeichnis sowie dessen Authentifizierungs- und Provider-Konfiguration
erhalten, ohne die Standardeinstellung für andere Sitzungen zu ändern. Beobachtete
übernommene Chats nehmen außerdem an der [Sitzungszustandserkennung](/de/concepts/session-state)
teil.

Bei der lokalen Standard-Überwachungsverbindung wird der Speicher gemeinsam mit nativen
Codex-Clients verwendet. OpenClaw nimmt nicht an, dass ein anderer Client denselben aktiven
App-Server-Prozess verwendet, und die Zuständigkeit für den nativen Status ist prozesslokal.
Daher behandelt OpenClaw einen Thread, den der zugehörige Überwachungs-App-Server als
`notLoaded` meldet, als **Gespeichert / Aktivität unbekannt** und nicht als inaktiv.

Wenden Sie dieselbe optionale Aktivierung auf jedem monitorlosen Node-Host an, dessen Sitzungen
angezeigt werden sollen. Die native OpenClaw-App für macOS liest dieselbe lokale Einstellung,
wenn sie ihren Codex-Katalog beim gekoppelten Gateway bekannt gibt. Dieser gekoppelte native
Mac-Katalog unterstützt nur die Standardeinstellung oder ein explizites `appServer.transport: "stdio"` mit
einem nicht gesetzten oder expliziten `appServer.homeScope: "user"`. `command`, `args` und
`clearEnv` werden für diesen stdio-Prozess berücksichtigt. Wenn die Mac-Konfiguration
`"unix"`, `"websocket"` oder `homeScope: "agent"` auswählt, gibt die App weder die
Katalogfunktion noch den Katalogbefehl bekannt, und ein veralteter direkter Aufruf schlägt
fehl, statt das Codex-Benutzerverzeichnis offenzulegen oder einen anderen lokalen
stdio-App-Server zu starten.

Ein neu bekannt gegebener Node-Befehl ändert die genehmigte Befehlsoberfläche des Nodes.
Genehmigen Sie die Aktualisierung vom Gateway-Host aus:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Nicht archivierte Codex-Sitzungen werden außerdem in der Hauptseitenleiste der Control UI
nach Host gruppiert angezeigt. Wählen Sie eine Sitzung aus, um ihr gespeichertes Transkript
zu lesen. Der Viewer verwendet die neueste Codex-API `thread/turns/list` mit `itemsView: "full"`
und lädt höchstens 20 Durchläufe pro Anfrage; **Ältere Transkripteinträge laden** folgt dem
opaken App-Server-Cursor der neuesten Seite. Geladene Seiten werden in chronologischer
Reihenfolge dargestellt. Der Viewer lädt niemals einen unbegrenzten
`thread/read`-Verlauf. Eine Seite oberhalb der Transportsicherheitsgrenze von 20 MiB
schlägt sicher fehl, statt die Node- oder Gateway-Verbindung zu gefährden.

Öffnen Sie die Gruppe **Codex** in der normalen Sitzungsseitenleiste. Dort werden dieselben
Sitzungen nach Host gruppiert aufgeführt. **Weitere Sitzungen laden** fügt von jedem Host,
der ältere Zeilen besitzt, die nächste Seite an, und diese angefügten Zeilen bleiben auch
bei der regelmäßigen Aktualisierung der Seitenleiste erhalten. Jeder Host wird angezeigt,
sobald seine eigene native Auflistung abgeschlossen ist. Die sichtbare Seite wird nach
Änderungen der Node-Konnektivität, beim erneuten Erhalt des Fokus und spätestens alle
30 Sekunden abgeglichen; ein geändertes Ergebnis löst einen schnelleren Folgedurchlauf aus.
Sitzungen, die in Codex Desktop, der CLI oder einem anderen nativen Client erstellt wurden,
erscheinen daher ohne vollständiges Neuladen der Seite. Die erste Seite folgt der
Codex-eigenen Sortierung nach dem Zeitpunkt der letzten Aktualisierung, sodass eine neu
erstellte native Sitzung sofort berücksichtigt werden kann.
Jede zurückgegebene Suchseite durchsucht eine begrenzte Anzahl nativer Seiten pro Host,
anstatt die Abfrage an App Server zu senden, da die native Suche auch Übereinstimmungen in
Transkriptvorschauen finden kann.

Hostverfügbarkeit und Threadstatus sind voneinander getrennt. **Offline** oder
**Nicht verfügbar** beschreibt die Aktualisierung eines Hosts; ein nicht verfügbarer Host
liefert keine neuen Sitzungszeilen und ändert den nativen Status eines Threads nicht in
`offline`. Sitzungszeilen verwenden Codex-Statuswerte wie `idle`,
`active`, `notLoaded` oder „Fehler“. Ein ausgefallener Host blendet die
Ergebnisse funktionsfähiger Hosts nicht aus.

Die Warnung in der Seitenleiste enthält den Katalogfehlercode und den sicheren
zugrunde liegenden Gateway-Fehler. Öffnen Sie **Settings > Automation > Plugins > Codex > Native Session
Discovery**, um die Erkennung zu deaktivieren, ohne Codex zu deaktivieren. Vergleichen Sie bei
`NODE_LIST_FAILED` `openclaw nodes list` mit **Settings > Devices**; die detaillierte Ursache
gibt an, welcher Fehler im Kopplungsspeicher, in der Node-Registrierung, bei den Berechtigungen
oder im Gateway-Lebenszyklus behoben werden muss.

## Operator-CLI verwenden

Die Terminal-CLI stellt denselben nicht archivierten Katalog sowie Gateway-lokale Aktionen
zum Verzweigen und Archivieren bereit:

```bash
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex continue <thread-id> [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
```

Optionen für `openclaw codex sessions`:

- `--search <text>` durchsucht Sitzungstitel ohne Berücksichtigung der Groß- und Kleinschreibung.
- `--host <id>` beschränkt die Antwort auf einen stabilen Katalog-Host, beispielsweise
  `gateway:local` oder `node:<node-id>`.
- `--limit <count>` legt 1 bis 100 Zeilen pro Host fest; der Standardwert ist 50.
- `--cursor <cursor>` setzt eine Hostseite fort und erfordert daher `--host`.
- `--json` gibt die strukturierte Gateway-Antwort aus.

Alle drei Befehle übernehmen `--url`, `--token` und `--timeout <ms>` vom
Gateway-Client. Für die Sitzungsauflistung gilt standardmäßig ein Zeitlimit von 75,000 ms,
damit Kaltstarts der Kataloge gekoppelter Nodes abgeschlossen werden können; für die
Fortsetzung und Archivierung gilt standardmäßig ein Zeitlimit von 30,000 ms. Sie stellen
außerdem den gemeinsamen Schalter `--expect-final` bereit, der diese unären
Überwachungs-RPCs nicht verändert. Jeder Befehl erfordert den Gateway-Scope
`operator.write`. Die Standardausgabe `-h, --help` ist für jeden Unterbefehl
verfügbar. Es gibt keine Option für archivierte oder einschließlich archivierter Sitzungen.
`sessions` kann gekoppelte Hosts auflisten, aber `continue` und
`archive` zielen immer auf `gateway:local`; gekoppelte Zeilen können nur
aufgelistet werden. Für die Archivierung ist immer `--confirm-no-other-runner` erforderlich.

Diese Shell-Befehle unterscheiden sich von den Runtime-Befehlen `/codex` im Chat.
`/codex threads [filter]` listet App-Server-Threads auf, die für die aktuelle
Konversationsverbindung verfügbar sind. `/codex sessions --host <node>` listet fortsetzbare
Codex-CLI-Sitzungsdateien auf einem einzelnen Node auf, nicht den Katalog der gesamten
Überwachungsflotte. `/codex
resume` und `/codex bind` binden die aktuelle
Konversation ein, statt einen sicheren überwachten Branch zu erstellen, und ein
modellgebundener überwachter Chat lehnt solche Bindungsänderungen ab. Es gibt keinen
Runtime-Befehl `/codex continue` oder `/codex archive`.

## Von einer lokalen Sitzung verzweigen

Wählen Sie bei einer gespeicherten oder inaktiven Zeile des Gateway-Computers
**Als Branch fortsetzen**. OpenClaw erstellt einen normalen Chat-Eintrag, spiegelt den
begrenzten Verlauf von Benutzer- und Assistentennachrichten bis einschließlich des letzten
abschließend gespeicherten Durchlaufs der Quelle (abgeschlossen, unterbrochen oder
fehlgeschlagen), zeichnet einen ausstehenden Harness-Branch auf und öffnet den Chat. Die
generische Modellauswahl ist gesperrt, aber es wurde noch kein konkretes Modell und kein
Provider ausgewählt. Die Quelle wird nicht fortgesetzt, und der kanonische Harness-Thread
wird noch nicht gestartet. Bei Wiederholung der Aktion wird der vorhandene Chat geöffnet,
statt einen weiteren Branch zu erstellen.

Die Spiegelung behält das neueste sichtbare Ende bei, das alle drei Grenzwerte einhält:
höchstens 200 Benutzer- oder Assistentennachrichten, insgesamt 512 KiB UTF-8-Text und
64 KiB pro Nachricht. Übergroße Nachrichten werden mit einer Markierung gekürzt, und ältere
Nachrichten werden ausgelassen, sobald ein Grenzwert erreicht ist. Eine Bild- oder
lokale Bildeingabe wird zum wörtlichen Platzhalter `[Image attachment]`; Bilddaten und lokale
Pfade werden nicht kopiert.

Senden Sie die erste normale Chat-Nachricht, um mit der Arbeit zu beginnen. Das Codex-Harness installiert die
tatsächlichen Handler für Genehmigungen, Abfragen, Ereignisse und Zustellung. Es verwendet einen temporären
nativen Fork auf der Supervision-Verbindung, um den Quell-Snapshot festzuschreiben, ohne
eine Modell- oder Provider-Überschreibung anzugeben. Codex App Server wählt beides aus seiner
aktuellen nativen Konfiguration aus und gibt die tatsächliche Auswahl zurück. Auf derselben
Verbindung startet OpenClaw den kanonischen vollständigen Harness-Thread aus der Quelle `appServer`
unter dessen Arbeitsverzeichnis und Laufzeitrichtlinie mit genau diesem zurückgegebenen Paar, fügt den
begrenzten sichtbaren Verlauf ein und archiviert den temporären Fork. Der kanonische Thread
verfügt über die vollständige Tool-Oberfläche des OpenClaw-Harness. Dies ist ein Zweig des sichtbaren Verlaufs und
kein vollständiger Klon des nativen Rollouts: Schlussfolgerungen der Quelle, Tool-Aufrufe und Tool-Ergebnisse werden
ausgelassen. Dieser und jeder spätere Turn verbleibt auf der überwachten Codex-Verbindung,
anstatt eine andere OpenClaw-Modelllaufzeit oder das gewöhnliche Agent-Home-Harness zu verwenden.

Die zurückgegebene Auswahl ist kein Nachweis für das historische Modell der Quelle. Wenn die
aktuelle native Konfiguration vom für den letzten Turn der Quelle aufgezeichneten Modell
abweicht, gibt Codex seine normale Warnung über die Modellabweichung aus. OpenClaw verwendet das
zurückgegebene Paar zum Starten des kanonischen Threads. Codex speichert das native
Modell und den Provider dieses kanonischen Threads; bei späteren Fortsetzungen bleiben sie erhalten, da
OpenClaw Modell- und Provider-Überschreibungen auslässt. Wenn der kanonische Thread
über eine separate native Codex-Steuerung geändert wird, übernimmt OpenClaw die von Codex gespeicherte
Auswahl. OpenClaw ersetzt sie niemals durch sein äußeres Modell oder seine Fallback-Kette.

Der überwachte, modellgebundene Chat kann nicht gelöscht werden, das Modell wechseln, `/new`
oder `/reset` verwenden, die Gateway-Aktion zum Zurücksetzen der Sitzung aufrufen oder die generische
Aktion **Sitzung forken** verwenden. Änderungen an `/codex model <model>`, `/codex
bind`, `/codex resume` (einschließlich einer Node-Sitzung mit `--bind here`) sowie
`/codex detach` oder `/codex unbind` werden ebenfalls abgelehnt, da sie die gesperrte
native Bindung ersetzen oder löschen würden. Die Abfrage `/codex model` sowie `/codex fast`,
`/codex permissions` und `/codex threads` bleiben verfügbar. Starten Sie eine andere
gewöhnliche Sitzung, wenn Sie ein anderes Modell oder einen neuen Thread wünschen.

Lassen Sie die Supervision für diesen Chat aktiviert. Wenn die Supervision deaktiviert wird oder ihre
gespeicherte Verbindungsbindung nicht mehr verfügbar oder inkonsistent ist, schlägt der Turn
sicher geschlossen fehl, anstatt zu einer gewöhnlichen Agent-Home-Sitzung zu wechseln.

Das Deaktivieren oder Deinstallieren des Plugins `codex` gibt diese Eigentümerschaft nicht frei und
macht den Chat nicht für ein anderes Modell verfügbar. Der gesperrte Chat bleibt erhalten, ist jedoch
nicht verfügbar; installieren oder aktivieren Sie dasselbe Plugin erneut und starten Sie das Gateway neu, um
ihn fortzusetzen. Dieses bewusste Fail-Closed-Verhalten verhindert, dass eine Aufbewahrungsbereinigung oder ein
vorübergehender Plugin-Ausfall die native Bindung unbemerkt verwaist.

Das Agent-Tool `codex_threads` folgt derselben Grenze. Es kann weder einen
anderen Fork anhängen noch den gebundenen nativen Thread des Chats archivieren. Listen- und reine Metadaten-
Lesevorgänge bleiben verfügbar. Rohe Transkript-Lesevorgänge erfordern `allowRawTranscripts`.
Wenn der Rohzugriff deaktiviert ist, lehnt `codex_threads` auch die Listensuche ab, da
die native Suche Transkriptvorschauen enthält; die Control UI und die Operator-CLI
bieten weiterhin eine begrenzte Suche nur nach Titeln. Umbenennen, Dearchivieren, abgetrennter Fork und
Archivieren eines nicht zugehörigen Threads ohne Eigentümer erfordern
`allowWriteControls`. Keine der beiden Optionen umgeht die gesperrte Bindung.

OpenClaw abonniert oder beantwortet keine Genehmigungsanfragen, während lediglich
der Quell-Thread aufgelistet oder der ausstehende Chat angezeigt wird. Das Starten eines separaten kanonischen
Harness-Threads beim ersten Turn ermöglicht es einem anderen Codex-Prozess, weiterhin Eigentümer der
Quelle zu bleiben, ohne konkurrierende Rollout-Schreiber zu erzeugen.

Die ursprüngliche Quelle aus CLI, VS Code, Atlas oder ChatGPT bleibt für native
Clients und den OpenClaw-Katalog sichtbar. Der kanonische Zweig wird als nativer
Codex-Thread gespeichert, seine Quellart ist jedoch `appServer`; Codex Desktop oder ein anderer
nativer Client kann diese Quellart herausfiltern, daher ist nicht garantiert,
dass der Zweig selbst in jeder nativen Verlaufsansicht erscheint.

Eine von OpenClaws App Server als aktiv gemeldete Zeile kann keinen neuen Zweig starten. Warten Sie,
bis der aktuelle Turn beendet ist, und aktualisieren Sie den Katalog. Codex App Server
serialisiert Änderungen innerhalb eines einzelnen Prozesses, stellt jedoch keinen exklusiven
prozessübergreifenden Runner und keine Lease für den Genehmigungseigentümer bereit.

Bei einer Zeile mit **Gespeichert / Aktivität unbekannt** verwenden die Chat-Spiegelung und das Festschreiben des Snapshots
beim ersten Turn den Codex-Zustand bis zum letzten dauerhaft gespeicherten abgeschlossenen Turn. Der Quell-
Thread wird weder fortgesetzt noch unterbrochen oder archiviert. Wenn ein anderer Prozess einen
laufenden Turn hat, sind dessen neueste noch laufende Arbeiten möglicherweise nicht im Zweig enthalten.

## Eine lokale Sitzung archivieren

Wählen Sie **Archivieren** für eine gespeicherte oder inaktive Gateway-lokale Zeile und bestätigen Sie anschließend, dass kein
anderer Codex-Client oder OpenClaw-Runner diesen Thread oder seine erzeugten
Nachkommen verwendet. OpenClaw liest den prozesslokalen Status erneut, fährt nur bei
`idle` oder `notLoaded` fort, ruft die native Codex-Archivierungsoperation auf und entfernt die
Sitzung aus der Liste der nicht archivierten Sitzungen. Native Codex versucht außerdem, die
erzeugten Nachkommen des Threads zu archivieren.

Die Archivierung ist nicht verfügbar, wenn die erneute Abfrage die Sitzung als aktiv oder in einem
Fehlerzustand meldet, wenn sie zu einer gekoppelten Node gehört oder während ein neu erstellter
überwachter Chat noch einen ausstehenden Zweig aus dieser Quelle besitzt. Senden Sie die
erste Nachricht des Chats, um seinen kanonischen Zweig zu materialisieren, bevor Sie die Quelle archivieren.
Die Archivierung wird außerdem blockiert, wenn OpenClaw weiß, dass eine aktive Bindung Eigentümer des
genauen Ziel-Threads oder eines nicht archivierten erzeugten Nachkommens ist. OpenClaw folgt der
experimentellen Codex-Nachkommenabfrage über jede Seite; eine ungültige Antwort,
ein Anfragefehler, ein wiederholter Cursor oder Thread oder das Erreichen des Sicherheitslimits führt zur Ablehnung
der Archivierung.

Die Lese-, Nachkommen-Aufzählungs- und Archivierungsanfragen sind keine einzelne bedingte
Operation, sodass zwischen ihnen weiterhin ein Turn starten kann. Der App-Server-Status wird außerdem
nicht zwischen unabhängigen Prozessen geteilt. Die Bestätigung bildet daher die
Sicherheitsgrenze für unbekannte Clients und dieses Race: Beenden Sie jeden anderen Client oder
überprüfen Sie ihn anderweitig, bevor Sie bestätigen. Stellen Sie einen archivierten Thread mit Codex
Desktop, der Codex-CLI oder einem vom Eigentümer autorisierten nativen Thread-Verwaltungsablauf wieder her;
nach dem Dearchivieren erscheint er erneut.

```bash
codex unarchive <thread-id>
```

## Einschränkungen gekoppelter Nodes verstehen

Gekoppelte Nodes stellen die versionierten, schreibgeschützten Befehle
`codex.appServer.threads.list.v1` und
`codex.appServer.thread.turns.list.v1` bereit. Native Node-Hosts, auf denen die
Codex-CLI verfügbar ist, stellen außerdem den in der Zulassungsliste enthaltenen Befehl `codex.terminal.resume.v1`
bereit. Das Gateway empfängt normalisierte
Metadaten und ausdrücklich angeforderte begrenzte Transkriptseiten, niemals rohe App-Server-
Endpunkte. Das Öffnen einer Zeile im Operator-Terminal führt `codex resume <thread-id>`
auf dem besitzenden Host aus und leitet das PTY dieses Befehls weiter; es stellt weder eine allgemeine
Shell noch vom Gateway bereitgestellte argv bereit.

Die Terminal-Weiterleitung stellt weder die Verträge zur Harness-Fortsetzung noch zur Archiveigentümerschaft
bereit. Remote-Zeilen bleiben daher sichtbar, bieten jedoch weder **Fortsetzen** noch
**Archivieren**, selbst wenn der Remote-Thread inaktiv ist. Verwenden Sie Codex auf diesem Computer
über **Im Terminal öffnen** oder verwenden Sie einen zukünftigen Fortsetzungsablauf mit einer sicheren
Grenze für die Runner-Eigentümerschaft.

## Metadaten und Berechtigungen

Katalogzeilen können Folgendes enthalten:

- Thread- und Sitzungskennungen
- Titel und Arbeitsverzeichnis
- aktueller Status und aktive Warteflags
- Zeitstempel für Erstellung, Aktualisierung und Aktivität
- Quelle, Modell-Provider, Codex-CLI-Version und Git-Branch

Die Katalogprojektion schließt Transkriptvorschauen, Turns, Rollout-Pfade,
den Codex-Home-Pfad, Git-Remotes, Commit-SHAs und rohe App-Server-Fehler aus. Der Katalogzugriff
und Transkript-Lesevorgänge der Control UI erfordern den Gateway-Bereich `operator.write`,
da die Flottenaggregation den standardmäßigen Pfad `node.invoke` verwendet, obwohl
beide Node-Befehle schreibgeschützt sind.

`supervision.allowRawTranscripts` und `supervision.allowWriteControls` steuern
autonome Agent- und eigenständige MCP-Tools. Beide sind standardmäßig auf `false` gesetzt. Bei
aktivierter Supervision entfernt `codex_threads` Transkriptvorschauen und Turns aus
Listen- und reinen Metadaten-Leseergebnissen, sofern rohe Transkripte nicht erlaubt sind; ein
Lesevorgang einschließlich Turns schlägt sicher geschlossen fehl. Jeder Fork sowie jedes Umbenennen, Archivieren und Dearchivieren
erfordert Schreibsteuerungen. Diese Optionen schränken die authentifizierte Transkriptanzeige in der Control UI
nicht ein und umgehen keine Prüfungen von Bindung, Host, Status oder Bestätigung.

### Kompatibilitätstools

Das offizielle Plugin `codex` behält die fünf ausgelieferten Supervisor-Tool-Namen für
bestehende Agent- und eigenständige MCP-Clients bei:

- `codex_endpoint_probe`
- `codex_sessions_list`
- `codex_session_read`
- `codex_session_send`
- `codex_session_interrupt`

`codex_sessions_list` lädt standardmäßig nur bereits geladene Einträge; es gibt keinen Parameter `loaded_only`.
Setzen Sie `include_stored: true`, um zusätzlich nicht archivierte gespeicherte Zeilen aus
der Zustandsdatenbank von Codex zu lesen. Die optionale Obergrenze `max_stored_sessions` beträgt standardmäßig 200
und akzeptiert 1 bis 1.000 Zeilen pro Endpunkt. Sie begrenzt geladene Zeilen nicht.
Ohne Berechtigung für rohe Transkripte lassen Listenergebnisse aus Transkripten abgeleitete Namen,
Vorschauen und detaillierte Endpunktfehler aus.
`codex_session_read` erfordert `allowRawTranscripts`; `include_turns: true`
fordert von Codex zusätzlich Turns an.

`codex_session_send` und `codex_session_interrupt` erfordern
`allowWriteControls`. Senden akzeptiert `mode: "auto" | "start" | "steer"`, aber
`"start"` wird immer abgelehnt, und sowohl `"auto"` als auch `"steer"` können nur einen
lesbaren aktiven Turn steuern. Ein inaktiver Thread wird mit dem Hinweis abgelehnt, **Codex
Sessions** zu verwenden, wo das vollständige Harness vor der
Fortsetzung Genehmigungs- und Tool-Handler installiert. Eine Unterbrechung erfordert ebenfalls einen aktiven lesbaren Turn. Diese Tools
setzen einen inaktiven Quell-Thread weder fort noch starten sie ihn.

`openclaw doctor --fix` verschiebt einen veralteten Eintrag `codex-supervisor`, dessen Endpunkt-
und Berechtigungsfelder sowie Verweise auf die Plugin-Zulassungs-/Ablehnungsrichtlinie in das offizielle
Plugin `codex`, ohne explizite kanonische Einstellungen zu überschreiben. Der eigenständige
Kompatibilitäts-MCP-Adapter lädt weiterhin dieselben fünf Tools aus diesem
Plugin; veraltete Richtlinien-Umgebungsvariablen gelten nur innerhalb dieses vertrauenswürdigen
Adapters.

Informationen zu jedem Supervision-Konfigurationsfeld finden Sie in der
[Codex-Harness-Referenz](/de/plugins/codex-harness-reference#supervision).

## Fehlerbehebung

**Es werden keine Sitzungen angezeigt:** Überprüfen Sie, ob `@openclaw/codex` installiert ist, sowohl das
Plugin als auch `supervision.enabled` auf true gesetzt sind, die aktuelle Plugin-Zulassungsliste
`codex` erlaubt und die Sitzungen nicht archiviert sind. Starten Sie das Gateway oder die Node neu, nachdem
Sie die Aktivierung geändert haben.

**Fortsetzen ist deaktiviert:** Eine nicht zugeordnete Zeile ist aktiv, gehört zu einer gekoppelten Node,
deren Host ist offline oder eine andere Aktion steht aus. Gateway-lokale gespeicherte und inaktive
Zeilen bieten **Als Zweig fortsetzen** anstelle einer unsicheren Übernahme des exakten Threads. Eine Zeile,
die bereits über einen überwachten Chat verfügt, bietet **Chat öffnen**.

**Archivieren ist deaktiviert:** Die Archivierung ist nach der Bestätigung, dass kein anderer Runner aktiv ist, für Gateway-lokale Zeilen
mit gespeichertem Zustand/unbekannter Aktivität und für inaktive Gateway-lokale Zeilen verfügbar. Aktive Zeilen, Fehlerzeilen,
Offline-Zeilen, Zeilen gekoppelter Nodes, Zeilen mit ausstehendem Zweig und Zeilen mit bekanntem Eigentümer einer exakten Bindung bleiben
für die Archivierung schreibgeschützt.

**Eine archivierte Sitzung ist verschwunden:** Dies ist zu erwarten. Die Supervision-Seite verfügt
über keine Ansicht für archivierte Sitzungen. Führen Sie `codex unarchive <thread-id>` aus oder verwenden Sie Codex Desktop, um
sie wieder anzuzeigen.

**Die alte Konfiguration `codex-supervisor` ist noch vorhanden:** Führen Sie `openclaw doctor --fix` aus. Doctor
verschiebt den veralteten Plugin-Eintrag und die zugehörigen Verweise auf Plugin-Richtlinien nach
`plugins.entries.codex.config.supervision`, ohne explizite Codex-
Einstellungen zu überschreiben.

## Verwandte Themen

- [Codex-Harness](/de/plugins/codex-harness)
- [Codex-Harness-Referenz](/de/plugins/codex-harness-reference)
- [Codex-Harness-Laufzeit](/de/plugins/codex-harness-runtime)
- [Codex-Supervision-Architektur](/de/specs/codex-supervision)
- [Nodes](/de/nodes)
- [Gateway-Sicherheit](/de/gateway/security)
