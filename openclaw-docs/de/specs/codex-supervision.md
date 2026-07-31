---
read_when:
    - Codex-Sitzungserkennung, -fortsetzung oder -archivierung entwerfen
    - Ändern der nativen Benutzeroberfläche des Sitzungskatalogs oder der Gateway-RPCs
    - Ausweitung der Codex-Überwachung auf gekoppelte Nodes
summary: Architektur und Produktgrenze für die Überwachung nativer Codex-Sitzungen aus OpenClaw.
title: Codex-Überwachung
x-i18n:
    generated_at: "2026-07-26T18:47:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e259badc8f7fdec6fa093785a1dd04394e12287ae61f00474bcd45e7b95352d
    source_path: specs/codex-supervision.md
    workflow: 16
---

# Codex-Überwachung

## Ziel

Mit der Codex-Überwachung kann ein OpenClaw-Operator native Codex-Sitzungen erkennen und,
sofern dies sicher ist, über die normale OpenClaw-Chat-Oberfläche einen lokalen Branch erstellen.
Codex App Server bleibt Eigentümer des Threads und des Modellzyklus. OpenClaw stellt den
Flottenkatalog, die authentifizierte Operator-Benutzeroberfläche, die Sitzungsbindung und die Kanalauslieferung bereit.

Die Funktion gehört zum offiziellen Plugin `codex`. Es gibt kein separates
Supervisor-Plugin und keine zweite Implementierung des Codex-Protokolls.

## Produktgrenze

Der Katalog wird immer registriert, wenn das Codex-Plugin aktiv ist, sofern die Erkennung
nativer Sitzungen nicht ausdrücklich wie folgt deaktiviert wurde:

```text
plugins.entries.codex.config.sessionCatalog.enabled = false
```

Aktivieren Sie die Überwachungswerkzeuge für Agenten mit:

```text
plugins.entries.codex.config.supervision.enabled = true
```

Das derzeit aktive Ausgangsprodukt ist bewusst kleiner als der langfristige
Flottenplan:

- Nur nicht archivierte Codex-Threads auflisten.
- Lokale Zeilen und Zeilen angemeldeter gekoppelter Nodes anhand einer stabilen Hostidentität gruppieren.
- Aus einem gespeicherten oder inaktiven Gateway-lokalen Thread einen normalen, modellgebundenen Chat-Branch
  erstellen, dessen vollständigen Codex-Harness-Thread beim ersten Durchlauf starten oder den für einen früheren Branch
  erstellten Chat öffnen.
- Einen gespeicherten oder inaktiven Gateway-lokalen Thread erst nach ausdrücklicher
  Bestätigung archivieren, dass kein anderer Runner vorhanden ist.
- Aktive lokale Quellen ohne Steuerelemente für neue Branches oder die Archivierung anzeigen und dabei weiterhin
  das Öffnen eines vorhandenen überwachten Chats ermöglichen.
- Die neuesten Zeilen jedes Hosts in der Hauptseitenleiste anzeigen, den vollständigen Katalog auf
  der Sitzungsseite beibehalten und begrenzte, cursorpaginierte Transkriptabrufe für
  lokale Zeilen und Zeilen gekoppelter Nodes bereitstellen.
- Katalogfehler nach Host isolieren.

Der Katalog ist die Sammlung der nicht archivierten Elemente. Eine darin enthaltene Zeile kann dennoch
den Durchlaufstatus „inaktiv“, „aktiv“, `notLoaded` oder „Fehler“ aufweisen.

Die agentenseitige Überwachung bleibt optional. Das geführte Onboarding versucht, sie zu installieren und zu aktivieren,
nachdem die native Codex-Installation erfolgreich erkannt wurde und das ausgewählte Inferenz-Backend
seine Live-Prüfung bestanden hat, unabhängig davon, welches primäre Backend der Benutzer
auswählt. Die Überwachung wird nur aktiviert, wenn diese opportunistische Plugin-Einrichtung
erfolgreich ist. Ein ausdrücklich deaktiviertes Plugin, eine Richtliniensperre oder
`supervision.enabled: false` bleibt für Überwachungswerkzeuge maßgeblich, deaktiviert jedoch
nicht den Operatorsitzungskatalog. `sessionCatalog.enabled: false`
deaktiviert die Operatorerkennung und Katalogbefehle für gekoppelte Nodes; der Codex-
Provider und der Harness bleiben aktiv.

## Zuständigkeit

Das Plugin `codex` ist für das gesamte Verhalten von Codex App Server zuständig:

- Endpunkterkennung und Verbindungslebenszyklus
- Protokollinitialisierung und Versionsprüfungen
- Auflisten, Lesen, Fortsetzen und Archivieren von Threads sowie Ereignisverarbeitung
- Brücken für Genehmigungen und Benutzereingaben
- Bindungen nativer Threads an OpenClaw-Sitzungen
- Durchsetzung des ausschließlich für Codex geltenden Modells und Harness nach der Fortsetzung

Die Control UI und das Gateway verwenden diesen Plugin-eigenen Dienst. Sie lesen
Codex-Rollout-Dateien nicht direkt und implementieren keinen weiteren App-Server-Client.

Die lokale Standardtopologie lautet:

```text
Codex Desktop -> privater stdio App Server -> Codex-Benutzerverzeichnis
                                               ^
OpenClaw Codex-Plugin -> Überwachungsverbindung zu App Server
  (standardmäßig verwaltetes stdio des Benutzerverzeichnisses; explizite appServer-Einstellungen werden berücksichtigt)
  -> passiver Quellenkatalog und Lesezugriff
  -> Snapshot-Fixierung -> kanonischer Branch der appServer-Quelle
  -> Einbindung des sichtbaren Verlaufs und jeder späteren überwachten Chat-Nachricht

Gewöhnliche OpenClaw-Codex-Sitzungen -> standardmäßig verwaltetes stdio des Agentenverzeichnisses
  -> gewöhnliche vollständige Harness-Threads -> OpenClaw Chat und Kanalauslieferung
```

Das Aktivieren der Überwachung ändert den gewöhnlichen Codex-Harness nicht: Er bleibt
standardmäßig agentenspezifisch. Die separate Überwachungsverbindung verwendet standardmäßig
verwaltetes stdio des Benutzerverzeichnisses, sodass ihre Katalog- und Snapshot-Vorgänge native
gespeicherte Threads erfassen. Explizite Verbindungseinstellungen für `appServer` werden berücksichtigt. Wenn
`homeScope` nicht gesetzt ist, löst die Überwachungsverbindung den Wert für stdio
oder Unix zu `"user"` und für WebSocket zu `"agent"` auf. Setzen Sie `appServer.homeScope: "user"`
nur dann ausdrücklich, wenn auch der gewöhnliche Harness das native Codex-
Benutzerverzeichnis gemeinsam verwenden soll. Eine Ausnahme bildet ein Chat, der aus der Codex-Gruppe der Seitenleiste übernommen wurde: Seine private
Überwachungsbindung hält Quellenabrufe, die Erstellung des kanonischen Branches und spätere
Nachrichten auf der Überwachungsverbindung. Live-Status und Zuständigkeit bleiben
prozesslokal; ein Thread, der dem Überwachungsprozess von OpenClaw unbekannt ist, ist `notLoaded`,
selbst wenn Codex Desktop ihn aktiv ausführt.

Codex verfügt über einen experimentellen kanonischen lokalen Daemon mit einem separaten,
vom Installationsprogramm verwalteten Bootstrap-Vertrag. Diese Funktion darf diesen Daemon nicht implizit
starten, für sich beanspruchen oder voraussetzen.

## Katalogablauf

Die generische Gateway-Methode `sessions.catalog.list` leitet an den Katalog-Provider `codex`
weiter, der immer `archived: false` anfordert und App Server dessen
Standardauswahl interaktiver Quellen anwenden lässt: `cli`, `vscode`, Atlas und ChatGPT. Sie
kombiniert:

1. Gateway-lokale Ergebnisse von `thread/list` aus dem Überwachungs-App-Server,
   der standardmäßig verwaltetes stdio des Benutzerverzeichnisses verwendet.
2. Ergebnisse von `codex.appServer.threads.list.v1` aus jedem verbundenen, angemeldeten Node.

Die Transkriptauswahl verwendet lokal `thread/turns/list` mit `itemsView: "full"` oder
den versionierten Befehl `codex.appServer.thread.turns.list.v1` auf dem ausgewählten
Node. Jede Antwort enthält höchstens 20 persistierte Durchläufe sowie opake
Vorwärts- und Rückwärtscursor. Die Control UI fordert die Seiten beginnend mit den neuesten an, stellt jede Seite in
chronologischer Reihenfolge dar und fügt ältere Seiten am Anfang ein. Sie greift niemals auf ein
unbegrenztes `thread/read` zurück. OpenClaw weist außerdem jede serialisierte Elementseite über
20 MiB zurück, bevor sie den Node- oder Gateway-Transport passieren kann.

Die native Implementierung für gekoppelte macOS-Nodes unterstützt nur einen nicht gesetzten/standardmäßigen oder
expliziten Wert `appServer.transport: "stdio"` mit nicht gesetztem/standardmäßigem Überwachungsumfang oder
explizitem `appServer.homeScope: "user"`. Sie übergibt die konfigurierten Werte `command`, `args`
und den normalisierten Wert `clearEnv` an den untergeordneten Prozess. Bei `"unix"`, `"websocket"`
oder explizitem `homeScope: "agent"` kündigt sie weder die Katalogfunktion
noch den Befehl an; ein direkter Aufruf schlägt ebenfalls sicher geschlossen fehl. Sie darf das Codex-
Benutzerverzeichnis niemals für eine agentenspezifische Konfiguration verfügbar machen oder lokales stdio anstelle eines
expliziten Endpunkts verwenden.

Die Katalogprojektion normalisiert Kennungen, Titel, cwd, Status, aktive Warte-
Flags, Zeitstempel, Quelle, Modell-Provider, Codex-Version und Git-Branch. Sie
gibt keine Transkriptvorschauen, Durchläufe, Rollout-Pfade, Pfade des Codex-Benutzerverzeichnisses,
Git-Remotes, Commit-SHAs, Rohendpunkte oder unverarbeiteten App-Server-Fehler zurück. Transkriptantworten
enthalten nur die ausdrücklich angeforderte App-Server-Elementseite und ihre
opaken Cursor.

Hostfehler bleiben auf das jeweilige Hostergebnis beschränkt. Ein offline befindlicher Node oder ein nicht verfügbarer
lokaler App Server entfernt keine funktionsfähigen Hosts von der Seite. Konnektivität ist eine
Hosteigenschaft und kein Threadstatus: Ein fehlgeschlagenes Hostergebnis enthält keine aktuellen
Sitzungszeilen und projiziert `offline` nicht auf native Threads.

Die Control UI fordert fortlaufende Katalogaktualisierungen an. Jeder lokale oder gekoppelte Host
wird angezeigt, sobald seine eigene App-Server-Auflistung abgeschlossen ist; die aggregierte Antwort bleibt
der Kompatibilitäts- und Wiederherstellungs-Snapshot. Die sichtbare Seite wird nach
Konnektivitätsänderungen, beim Fokussieren und höchstens alle 30 Sekunden abgeglichen, wobei nach
Änderungen ein schnellerer Durchlauf erfolgt. Native Codex-Sitzungen, die in einem anderen Client erstellt wurden, werden daher
schließlich erkannt, ohne sie in den OpenClaw-Speicher zu importieren.

Die Katalogerkennung ist passiv. Beim Auflisten oder Lesen von Metadaten dürfen weder
`thread/resume` aufgerufen, der OpenClaw-Client für Live-Thread-Anfragen angemeldet noch
eine Genehmigung beantwortet werden.

Die Suche erfolgt ausschließlich im Titel und berücksichtigt keine Groß-/Kleinschreibung. Für jede zurückgegebene Katalogseite durchsuchen das
Gateway und der gekoppelte Mac eine begrenzte Anzahl nativer Seiten, ohne
die Abfrage an App Server weiterzugeben, da die native Suche auch Übereinstimmungen in Transkriptvorschauen
finden kann. Mit dem zurückgegebenen nativen Cursor können Aufrufer den Suchlauf fortsetzen.

## Grenze der Operator-CLI

Das Plugin registriert drei Gateway-gestützte Shell-Befehle:

```text
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [gateway-options]
openclaw codex continue <thread-id> [--json] [gateway-options]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [gateway-options]
```

`[gateway-options]` ist `--url <url>`, `--token <token>`, `--timeout <ms>` und
der übernommene Schalter `--expect-final`. Die Sitzungsauflistung verwendet standardmäßig 75,000 ms;
Fortsetzen und Archivieren verwenden standardmäßig 30,000 ms;
`--expect-final` hat für diese unären RPCs keine zusätzliche Wirkung. Die Sitzungssuche
erfolgt ausschließlich im Titel und berücksichtigt keine Groß-/Kleinschreibung; jede Antwort durchsucht eine begrenzte Kette nativer Seiten,
und `--cursor` setzt die Suche nach älteren Ergebnissen fort. Das Limit beträgt standardmäßig 50 pro Host
und akzeptiert Werte von 1 bis 100; ein Cursor erfordert ein stabiles Ziel `--host`.
Kein Befehl akzeptiert eine Option für archivierte Elemente oder deren Einbeziehung. Nur `sessions` kann gekoppelte Hosts ansprechen;
`continue` und `archive` senden immer `hostId: "gateway:local"`, und die Archivierung
erfordert das ausdrückliche Bestätigungs-Flag.

Der Shell-Namensraum ist nicht der Laufzeitnamensraum `/codex` innerhalb des Chats. Insbesondere
listet `/codex sessions --host <node>` Codex-CLI-Sitzungsdateien auf einem
Node auf, `/codex threads` listet App-Server-Threads für die aktuelle Konversationsverbindung
auf und `/codex resume` oder `/codex bind` ändert die Bindung dieser Konversation.
Diese Befehle ersetzen `sessions.catalog.continue` nicht, und es gibt keinen
Laufzeitbefehl `/codex continue` oder `/codex archive`.

## Lokale Fortsetzung

Für eine gespeicherte oder inaktive Gateway-lokale Zeile ruft die Benutzeroberfläche
`sessions.catalog.continue` mit `catalogId: "codex"` sowie den Host- und Thread-
Kennungen auf. Das Plugin:

1. Verwendet den vorhandenen überwachten Chat erneut, wenn die Quelle bereits über einen verfügt.
2. Projiziert andernfalls einen begrenzten Verlauf von Benutzer- und Assistentennachrichten bis zum letzten
   terminalen persistierten Durchlauf der Quelle (abgeschlossen, unterbrochen oder fehlgeschlagen) in einen neuen
   OpenClaw Chat und zeichnet einen ausstehenden Harness-Branch auf.
3. Speichert die ausstehende, ausschließlich für Codex geltende Modellbindung, jedoch keine konkrete Modell- oder
   Providerauswahl, zusammen mit dem privaten Umfang der Überwachungsverbindung und
   gibt den OpenClaw-Wert `sessionKey` zurück.

Die Verlaufsprojektion wählt den neuesten Abschnitt sichtbarer Benutzer- und Assistenten-
nachrichten aus, mit festen Grenzen von 200 Nachrichten, insgesamt 512 KiB UTF-8-Text und
64 KiB pro Nachricht. Sie ersetzt Bild- und lokale Bildeingaben durch
`[Image attachment]`, kopiert niemals Bildnutzdaten oder Pfade und lässt Schlussfolgerungen,
Werkzeugaufrufe und Werkzeugergebnisse aus.

Die Benutzeroberfläche navigiert mit diesem Sitzungsschlüssel zum normalen Chat. Es existiert noch kein kanonischer Harness-
Thread. Beim ersten normalen Chat-Durchlauf installiert der Harness die tatsächlichen
Codex-Handler für Genehmigungen, Erfragungen, Ereignisse und Auslieferungen und führt anschließend Folgendes aus:

1. Verwendet die Überwachungsverbindung, um das native `thread/fork` ohne Modell-
   oder Providerüberschreibung aufzurufen und den persistierten Quell-Snapshot zu fixieren. Der aktuelle
   Zustand `ConfigManager` von Codex wählt Modell und Provider aus, und die Fork-Antwort
   meldet das tatsächlich verwendete Paar. Wenn sich das Modell vom zuletzt in der Quelle aufgezeichneten Modell
   unterscheidet, gibt Codex seine normale Warnung über die Modellabweichung aus.
2. Startet über dieselbe Verbindung den kanonischen vollständigen Codex-Harness-Thread mit
   `threadSource: "appServer"`, dem cwd, der Richtlinie, der Konfiguration und der Umgebung von OpenClaw, der
   vollständigen Werkzeugoberfläche des OpenClaw-Harness sowie exakt dem Modell und Provider,
   die vom Fork für diesen ersten Start zurückgegeben wurden.
3. Bindet den begrenzten sichtbaren Verlauf der Benutzer- und Assistentennachrichten über diese
   Verbindung ein, schreibt die kanonische Bindung fest, ohne ihren Überwachungsumfang zu verwerfen,
   führt den Durchlauf aus und archiviert den temporären Fork.

Vor dem ersten Turn ist der Chat ein gesperrter ausstehender Branch mit einem sichtbaren
Verlaufsabbild; danach läuft jeder Modell-Turn über den kanonischen Codex-
Harness-Thread auf der Supervision-Verbindung. Der Branch ist kein vollständiger nativer
Rollout-Klon: Quell-Reasoning, Tool-Aufrufe und Tool-Ergebnisse werden bewusst
ausgelassen. Wenn das Anheften des Snapshots oder die Erstellung des kanonischen Threads fehlschlägt, bleibt der ausstehende
Branch erneut versuchbar. Ein Bindungs-Wettlauf, deaktivierte Supervision oder eine nicht verfügbare
beziehungsweise nicht übereinstimmende Supervision-Verbindung schlägt vor der Ausführung des Turns geschlossen fehl,
statt auf den gewöhnlichen Harness im Agent-Home zurückzufallen.

Dies garantiert eine Codex-eigene Auswahl, nicht die Beibehaltung des historischen
Modells der Quelle. Das vom Fork zurückgegebene Paar wird für den Start des kanonischen Threads
verwendet, und Codex speichert das native Modell und den Provider dieses Threads. Bei späteren Fortsetzungen
werden Modell- und Provider-Überschreibungen von OpenClaw ausgelassen, sodass Codex das gespeicherte Paar wiederherstellt.
Wenn eine separate native Codex-Steuerung den kanonischen Thread ändert, akzeptiert OpenClaw
diese nativ gespeicherte Auswahl. Das äußere OpenClaw-Modell und die Fallback-Kette
ersetzen sie niemals.

Modelländerungen, das Löschen von Sitzungen und das Zurücksetzen beziehungsweise Neuerstellen von Sitzungen schlagen
für den überwachten, modellgesperrten Chat geschlossen fehl. Änderungen an `/codex model <model>`, `/codex
bind`, `/codex resume` (einschließlich Node `--bind here`) und `/codex detach` oder
`/codex unbind` schlagen ebenfalls geschlossen fehl, da sie die Bindung ersetzen oder löschen. Die
Abfrage `/codex model` sowie `/codex fast`, `/codex permissions` und `/codex
threads` bleiben verfügbar. Das Agent-Tool `codex_threads` kann keinen neuen
Fork anhängen oder den gebundenen nativen Thread archivieren. Listen- und reine Metadaten-Lesevorgänge bleiben
verfügbar; Transkriptfelder erfordern `supervision.allowRawTranscripts`, während
Umbenennen, Wiederherstellen aus dem Archiv, ein losgelöster Fork und die Archivierung eines nicht zugehörigen Threads
`supervision.allowWriteControls` erfordern. Keine der beiden Optionen kann die gesperrte Bindung ersetzen.
Das Löschen oder Zurücksetzen des OpenClaw-Eintrags würde andernfalls die native
Bindung verwerfen und hinter einer wie Codex aussehenden Sitzung einen generischen Thread erstellen oder zulassen.
Die Aufbewahrungswartung erhält daher modellgesperrte Einträge auch dann, wenn sie
gewöhnliche Alters-, Anzahl- oder Speicherbudgetgrenzen überschreiten. Auch beim Deaktivieren oder Deinstallieren des
zuständigen Plugins bleiben die Sperre und die Plugin-Eigentumsmarkierung erhalten. Der Chat bleibt
nicht verfügbar und schlägt geschlossen fehl, bis dasselbe Plugin wieder aktiviert wird; eine Bereinigung
wandelt ihn niemals in eine gewöhnliche Modellsitzung um.

Die Quelle wird durch diese Aktion niemals fortgesetzt oder verändert. Der temporäre Fork heftet einen
Snapshot an; er ist nicht der dauerhafte Fortsetzungs-Thread. Das Starten eines separaten
kanonischen Harness-Threads beim ersten Turn verhindert, dass OpenClaw zu einem
konkurrierenden Schreiber der Quelle wird, nur weil der prozesslokale Status einen
Desktop-eigenen Turn nicht erkannt hat. Das sichtbare Verlaufsabbild und der angeheftete Snapshot können Arbeit
auslassen, die in einer aktiven Quelle noch nicht abgeschlossen ist. Die ursprüngliche CLI-, VS-Code-,
Atlas- oder ChatGPT-Quelle bleibt sowohl für native als auch für OpenClaw-Kataloge zugelassen.
Der kanonische Branch bleibt ein nativer Codex-Thread im Supervision-Speicher,
native Clients können jedoch seine Quellart `appServer` herausfiltern, sodass die Sichtbarkeit in Codex Desktop
kein Vertrag ist.

## Archivierungsverhalten

Für eine gespeicherte oder inaktive Gateway-lokale Zeile erfordert `sessions.catalog.archive` mit
`catalogId: "codex"`
eine ausdrückliche Angabe von `confirmNoOtherRunner: true`, liest den aktuellen prozesslokalen
Status neu ein, fährt nur bei `idle` oder `notLoaded` fort, ruft nativ `thread/archive` auf
und gibt erst dann Erfolg zurück, wenn Codex den Vorgang akzeptiert hat. Die Zeile verlässt anschließend
den nicht archivierten Katalog.

Ein aktiver Status oder Fehlerstatus aus dem neuen Lesevorgang lehnt die Archivierung ab. Dasselbe gilt für einen
initialisierenden oder ausstehenden überwachten Branch der Quelle: Der erste Chat-Turn
muss seinen kanonischen Branch materialisieren, bevor die Quelle archiviert werden kann. Ein
bekannter aktiver Eigentümer einer OpenClaw-Bindung für das exakte Ziel oder ein nicht archivierter
erzeugter Nachkomme lehnt die Archivierung ebenfalls ab. OpenClaw paginiert die experimentelle
Beziehung `thread/list ancestorThreadId` von Codex und schlägt bei Anfrage- oder Antwortfehlern,
Cursor- oder Thread-Zyklen sowie beim Ausschöpfen von Sicherheitsgrenzen geschlossen fehl. Die native Archivierung kann
geladene übergeordnete und nachgeordnete Arbeiten beenden, daher ist die Archivierung keine Abkürzung
für eine Unterbrechung. Der Lesevorgang, die Aufzählung der Nachkommen und die Archivierungsaufrufe sind nicht atomar.
Ein unabhängiger Client kann weiterhin Arbeit an einer Zeile besitzen oder starten, die lokal inaktiv oder
`notLoaded` erscheint. Die Bestätigung, dass kein anderer Runner vorhanden ist, deckt unbekannte Clients und
diesen Wettlauf ab, bis Codex eine bedingte Archivierung oder eine prozessübergreifende Lease bereitstellt.
Die Archivierung gekoppelter Nodes ist untersagt.

Im Codex-Katalog gibt es keine Archivansicht. Ein Thread, der mit
`thread/unarchive` in einer anderen vom Eigentümer autorisierten Codex-Oberfläche wiederhergestellt wurde, ist
wieder für den nicht archivierten Katalog zugelassen.

## Sicherheit aktiver Threads

Codex serialisiert Änderungen an einem Thread zwischen Clients eines App Servers, stellt jedoch
weder einen exklusiven prozessübergreifenden Runner noch eine Lease für den Genehmigungseigentümer bereit.
Unabhängige stdio-App-Server können an denselben Rollout anhängen, während jeder
nur seinen eigenen speicherinternen Status sieht. Genehmigungsanfragen können außerdem jeden Abonnenten
eines Servers erreichen, wobei die erste gültige Antwort die Anfrage abschließt.

Daher gilt:

- passive Katalog-Clients abonnieren Genehmigungen nicht und lehnen sie nicht automatisch ab
- aktuell als aktiv gemeldete Zeilen bieten weder einen neuen Branch noch Archivieren an
- eine nicht zugeordnete Quelle wird zu einem Branch mit sichtbarem Verlauf, dessen kanonischer Harness-
  Thread die Quelle niemals fortsetzt
- `notLoaded` wird als Aktivität unbekannt angezeigt und kann erst nach
  informierter Bestätigung, dass kein anderer Runner vorhanden ist, archiviert werden
- die lokale Archivierung erfordert diese Bestätigung sowie einen neuen Lesevorgang von `idle` oder `notLoaded`,
  wobei der Protokoll-Wettlauf zwischen Lesen und Archivieren anerkannt wird

Unterbrechung und Übergabe zwischen mehreren Clients sind künftige Produktentscheidungen. Sie werden nicht
durch die Anzeige einer aktiven Zeile impliziert.

## Grenze gekoppelter Nodes

Node Invoke unterstützt derzeit ausschließlich Anfrage/Antwort. Es kann sicher begrenzte
Katalogmetadaten und Seiten mit Transkript-Turns zurückgeben, aber nicht den langlebigen Ereignisstrom, Genehmigungs-
anfragen, Tool-Aufrufe, Abbruch und Assistenten-Deltas übertragen, die für einen Codex-
Harness-Lauf erforderlich sind.

Der Node-Vertrag unterstützt daher Listen- und Transkript-Turn-Seiten. Entfernte
Zeilen bleiben lesbar, aber **Fortsetzen** und **Archivieren** sind unabhängig vom Inaktivitätsstatus nicht verfügbar. Eine
echte entfernte Fortsetzung erfordert einen Node-seitigen Runner und eine Streaming-Brücke, die
dieselben Genehmigungs- und Bindungsinvarianten wie der lokale Harness wahrt.

## Berechtigungen

Jeder Computer stimmt lokal zu. Das Aktivieren des Gateways autorisiert keinen anderen
Node zum Lesen seiner Codex-Metadaten. Die Node-Fähigkeit muss die normale Kopplung
und Genehmigung durch die Befehlsrichtlinie durchlaufen.

Flottenauflistung und Transkriptanzeige verwenden den Gateway-Bereich `operator.write`,
da sie gekoppelte Nodes aufrufen. Lokale Fortsetzung und Archivierung sind
authentifizierte Operatoraktionen und unterliegen weiterhin Host- und Statusprüfungen.

Der Zugriff durch autonome Agenten und eigenständiges MCP ist separat geregelt. Die ausgelieferten
Tool-Verträge `codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` und `codex_session_interrupt` bleiben Eigentum
des Plugins `codex`. Bei aktivierter Supervision erfordern rohe Transkript-
Lesevorgänge von `codex_threads` und aus Transkripten abgeleitete Listenfelder ebenfalls
`supervision.allowRawTranscripts`; jeder Fork, jede Umbenennung, Archivierung
oder Wiederherstellung aus dem Archiv über `codex_threads` erfordert `supervision.allowWriteControls`. Beide Richtlinien sind
standardmäßig deaktiviert.

## Kompatibilität

`openclaw doctor --fix` migriert die ausgelieferte Konfiguration `plugins.entries.codex-supervisor`,
einschließlich Endpunkten und Transkript-/Schreibrichtlinien sowie Plugin-
Zulassungs-/Ablehnungsreferenzen nach
`plugins.entries.codex.config.supervision`. Explizite Werte am kanonischen Ziel
gewinnen bei Konflikten. Laufzeitcode verwendet nach der Migration ausschließlich die kanonische Plugin-
Form `codex`.

Das offizielle Plugin behält genau fünf Supervisor-Kompatibilitätstools:
`codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` und `codex_session_interrupt`. Die Sitzungsliste enthält standardmäßig nur geladene
Sitzungen; es gibt keinen Parameter `loaded_only`. `include_stored: true` fügt
nicht archivierte Zeilen der Statusdatenbank hinzu, pro Endpunkt begrenzt durch `max_stored_sessions`
(Standardwert 200, zulässiger Bereich 1 bis 1.000); geladene Zeilen werden durch diese
Einstellung nicht begrenzt. Aus Transkripten abgeleitete Felder und Lesevorgänge bleiben durch
`allowRawTranscripts` geschützt; Senden und Unterbrechen bleiben durch `allowWriteControls` geschützt.

Kompatibilitätssenden startet oder setzt niemals einen inaktiven Thread fort. `mode: "start"` wird
immer abgelehnt; `"auto"` und `"steer"` steuern ausschließlich einen lesbaren aktiven Turn.
Auch eine Unterbrechung erfordert einen aktiven lesbaren Turn. Die Fortsetzung eines inaktiven Threads wird
an den nativen Codex-Katalog weitergeleitet, damit der vollständige Harness Genehmigungen, Tools und die Bindung verwaltet.
Der eigenständige Legacy-MCP-Adapter löst dieselben Tools aus dem offiziellen
Plugin auf und ist der einzige Pfad, der die beibehaltenen Legacy-Richtlinien-Umgebungsvariablen
berücksichtigt.

Die Katalog-Benutzeroberfläche vom Juli, die Gateway-Methode, die Node-Fähigkeit und die CLI-Registrierung waren
unter der alten Plugin-ID nicht ausgeliefert worden. Sie wechseln direkt in die Eigentümerschaft von `codex`,
ohne eine zweite Laufzeitfassade.

## Künftige Arbeiten

- Node-seitiger Streaming-Runner und Ereignisbrücke für die entfernte Fortsetzung
- explizite Leases für Runner und Genehmigungseigentümer zur gleichzeitigen Client-Übergabe
- entfernte Archivierung, sobald eine Lease für Runner-Eigentümerschaft oder eine gleichwertige Abschirmung existiert
- Unterbrechung und umfassendere Beobachtung aktiver Sitzungen
- auditierte Übergabe zwischen Codex Desktop, CLI und OpenClaw

Das Durchsuchen archivierter Threads ist nicht Teil der geplanten Supervision-Seitenleiste. Native Codex-
Oberflächen bleiben der Wiederherstellungspfad für archivierte Threads.

## Akzeptanztests

- Das Aktivieren der Überwachung listet nicht archivierte lokale Sitzungen auf.
- Archivierte Sitzungen erscheinen weder in der Katalogantwort noch in der Benutzeroberfläche.
- Funktionsfähige Hosts bleiben sichtbar, wenn ein anderer Host ausfällt; ein nicht verfügbarer Host
  gibt keine aktuellen Zeilen zurück, statt einen Offline-Sitzungsstatus zu erfinden.
- Eine gespeicherte oder inaktive lokale Zeile erstellt eine Chat-Spiegelung mit einer ausschließlich
  für Codex geltenden Modell-/Runtime-Sperre; der erste Durchlauf fixiert einen temporären Snapshot und startet den
  kanonischen vollständigen Harness-Thread, und erneutes Ausführen von Continue öffnet den vorhandenen Chat.
- Der erste Durchlauf lässt Modell-/Provider-Überschreibungen beim Snapshot-Fork aus und fixiert
  den kanonischen Start auf genau das von Codex zurückgegebene Paar, selbst wenn Codex warnt,
  dass sein aktuelles Modell vom zuletzt aufgezeichneten Modell der Quelle abweicht.
- Ausstehende und bestätigte überwachte Bindungen verwenden die Überwachungsverbindung für den
  Zugriff auf die Quelle, die Erstellung des kanonischen Branches und jeden späteren Durchlauf; gewöhnliche
  Codex-Sitzungen bleiben auf den Agenten beschränkt.
- Spätere Fortsetzungen lassen OpenClaw-Modell-/Provider-Überschreibungen aus, bewahren die
  kanonische persistierte Auswahl von Codex, akzeptieren separate native Änderungen an diesem Thread
  und ersetzen sie niemals durch das äußere OpenClaw-Modell oder die Fallback-Kette.
- Das Deaktivieren der Überwachung oder der Verlust des Bindungs-/Verbindungslebenszyklus führt zu einem
  sicheren Abbruch, statt den Chat in das gewöhnliche Harness des Agentenverzeichnisses zu verschieben.
- Ein überwachter, modellgesperrter Chat kann nicht gelöscht werden, solange er die native
  Bindung schützt.
- Der Chat spiegelt höchstens 200 Benutzer- und Assistentennachrichten, insgesamt 512 KiB und
  64 KiB pro Nachricht. Bilder werden zu Platzhaltern; Reasoning der Quelle, Tool-Aufrufe,
  Tool-Ergebnisse, Bildnutzdaten und lokale Pfade werden nicht geklont.
- Der Branch-Ablauf setzt den Quell-Thread niemals fort.
- Die ursprüngliche Quelle bleibt für beide Kataloge verfügbar. Der kanonische native
  Branch verwendet die Quellart `appServer` und erscheint nicht garantiert in
  Codex Desktop.
- Aktive lokale Quellen können weder einen Branch erstellen noch archiviert werden; ein vorhandener
  überwachter Chat kann weiterhin geöffnet werden.
- Zeilen mit unbekannter Aktivität können ohne Bestätigung einen Branch erstellen; die Archivierung erfordert
  eine ausdrückliche Bestätigung, dass kein anderer Runner aktiv ist.
- Eine Quelle mit einem initialisierenden oder ausstehenden überwachten Branch kann nicht archiviert werden,
  bis der erste Chat-Durchlauf den kanonischen Branch materialisiert.
- Ein bekannter aktiver Bindungseigentümer für das exakte Ziel oder einen nicht archivierten erzeugten
  Nachfahren blockiert die Archivierung; Fehler bei der Aufzählung von Nachfahren führen zu einem sicheren Abbruch, und
  die ausdrückliche Bestätigung bleibt für unbekannte Clients und das Zeitfenster zwischen
  Statusprüfung und Archivierung verantwortlich.
- Die bestätigte lokale Archivierung einer gespeicherten oder inaktiven Sitzung entfernt die Zeile nach nativem Erfolg.
- Zeilen gekoppelter Nodes bleiben ohne Continue oder Archive sichtbar.
- Die passive Auflistung abonniert oder beantwortet niemals Thread-Genehmigungen.
- Die veraltete Supervisor-Konfiguration wird in die kanonische Codex-Konfigurationsform migriert.
- Die veraltete Liste wird standardmäßig nur geladen, die Aufzählung gespeicherter Einträge hält ihre Obergrenze
  pro Endpunkt ein, und der Kompatibilitätsversand startet oder setzt niemals einen inaktiven Thread fort.
