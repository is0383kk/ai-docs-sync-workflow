---
read_when:
    - Sie benötigen den Laufzeitunterstützungsvertrag für das Codex-Harness.
    - Sie debuggen native Codex-Tools, Hooks, Compaction oder den Feedback-Upload
    - Sie ändern das Plugin-Verhalten über OpenClaw- und Codex-Harness-Durchläufe hinweg
summary: Laufzeitgrenzen, Hooks, Tools, Berechtigungen und Diagnosefunktionen für das Codex-Harness
title: Codex-Harness-Laufzeit
x-i18n:
    generated_at: "2026-07-26T17:56:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d18d42683df0d827b776547f7b45f60f572cf39410d00533f53f8fdcdccb0d2
    source_path: plugins/codex-harness-runtime.md
    workflow: 16
---

Laufzeitvertrag für Codex-Harness-Turns. Informationen zu Einrichtung und Routing finden Sie unter
[Codex-Harness](/de/plugins/codex-harness). Informationen zu Konfigurationsfeldern finden Sie in der
[Codex-Harness-Referenz](/de/plugins/codex-harness-reference).

## Überblick

Codex übernimmt die native Modellschleife, das native Fortsetzen von Threads, die native
Fortsetzung von Tools und die native Compaction. OpenClaw übernimmt das Channel-Routing, Sitzungsdateien,
die sichtbare Nachrichtenzustellung, dynamische OpenClaw-Tools, Genehmigungen, die
Medienzustellung und eine Transkriptspiegelung um diese Grenze herum.

Das Prompt-Routing richtet sich nach der ausgewählten Laufzeit, nicht nur nach der Provider-Zeichenfolge. Ein
nativer Codex-Turn erhält die Entwickleranweisungen des Codex App Server; eine explizite
OpenClaw-Kompatibilitätsroute behält den normalen OpenClaw-System-Prompt bei, selbst wenn
sie Codex-spezifische OpenAI-Authentifizierung oder -Übertragung verwendet.

OpenClaw startet native Codex-Threads und setzt sie fort, wobei die integrierte
Persönlichkeit von Codex deaktiviert ist (`personality: "none"`), damit Persönlichkeitsdateien des Arbeitsbereichs
und die OpenClaw-Agentenidentität maßgeblich bleiben. Native Codex-Ausführungen behalten ansonsten die Codex-eigenen
Basis-/Modellanweisungen und das Laden der Projektdokumentation bei. Leichtgewichtige
OpenClaw-Ausführungen (beispielsweise Cron) unterdrücken weiterhin das Laden der Projektdokumentation.

Die OpenClaw-Entwickleranweisungen decken Belange der OpenClaw-Laufzeit ab: Zustellung über den Quell-Channel,
dynamische OpenClaw-Tools, ACP-Delegierung, Adapterkontext und die
aktiven Arbeitsbereichsprofildateien des Agenten. Skill-Kataloge und über Tools weitergeleitete
`MEMORY.md`-Verweise werden als auf den Turn beschränkte Entwickleranweisungen zur Zusammenarbeit
projiziert. Wenn Speicher-Tools nicht verfügbar sind, werden aktive `BOOTSTRAP.md`-Inhalte
und vollständige `MEMORY.md` stattdessen als einfacher Turn-Eingabekontext bereitgestellt.

Die meisten dynamischen OpenClaw-Tools verwenden den durchsuchbaren `openclaw`-Namespace. Mit
`catalogMode: "direct-only"` gekennzeichnete Tools verwenden `openclaw_direct`, das Codex
direkt als `DirectModelOnly` für das Modell sichtbar hält, statt es einer verschachtelten
Code-Mode-Ausführung zugänglich zu machen.

## Thread-Bindungen und Modellwechsel

Wenn eine OpenClaw-Sitzung an einen bestehenden Codex-Thread angehängt ist, sendet der nächste
Turn das aktuell ausgewählte Modell, die Genehmigungsrichtlinie, die Sandbox,
den Genehmigungsprüfer und die Dienststufe erneut an den App Server. Ein Wechsel von
`openai/gpt-5.5` zu `openai/gpt-5.2` behält die Thread-Bindung bei, fordert Codex jedoch auf,
mit dem neu ausgewählten Modell fortzufahren.

Überwachte Bindungen bilden die Ausnahme. Die OpenClaw-Modellauswahl bleibt gesperrt,
und beim Fortsetzen werden Modell- und Provider-Überschreibungen ausgelassen, damit Codex das persistierte
Modell und den Provider des kanonischen Threads wiederherstellt. Eine separate native Codex-Steuerung kann
dieses persistierte Paar ändern, und der anfängliche Snapshot kann die normale
Codex-Warnung zu Modellunterschieden auslösen; das äußere OpenClaw-Modell und die Fallback-Kette
ersetzen keines der beiden.

## Überwachung und sichere Fortsetzung

Die Codex-Überwachung ist eine optionale Funktion desselben `codex`-Plugins. Sie erkennt
native Threads über eine separate Verbindung und projiziert nur nicht archivierte
Sitzungen in den Gateway-Katalog. Ohne explizite `appServer`-Verbindungseinstellungen
verwendet diese Verbindung verwaltete stdio-Kommunikation im Benutzerverzeichnis, während das gewöhnliche
Harness agentenspezifisch bleibt. Auflistungs- und Metadaten-Lesevorgänge sind passiv: Sie
setzen keinen Thread fort, abonnieren OpenClaw nicht für dessen Live-Ereignisse und beantworten
keine Genehmigungsanfragen.

Für eine gespeicherte oder inaktive Sitzung auf dem Gateway-Computer erstellt **Als Branch fortsetzen**
einen normalen, modellgebundenen Chat und spiegelt einen begrenzten Verlauf von Benutzer- und Assistentennachrichten
bis einschließlich des letzten terminalen persistierten Turns der Quelle. Der erste normale
Chat-Turn installiert die tatsächlichen Genehmigungs-Handler und verwendet einen temporären nativen Fork,
um den Snapshot ohne Modell- oder Provider-Überschreibung festzuhalten. Codex App Server verwendet
seine aktuelle native Konfiguration und gibt das ausgewählte Paar zurück; er gibt seine
normale Warnung aus, wenn dieses Modell vom zuletzt aufgezeichneten Modell der Quelle abweicht.
Über dieselbe Überwachungsverbindung startet OpenClaw den kanonischen
Codex-Harness-Thread der `appServer`-Quelle unter dessen Arbeitsverzeichnis und Laufzeitrichtlinie mit
genau dem zurückgegebenen Modell und Provider für diesen ersten Start, fügt den
begrenzten sichtbaren Verlauf ein und archiviert den temporären Fork. Die Quelle wird niemals
fortgesetzt. Der kanonische Thread verfügt über die vollständige Tool-Oberfläche des OpenClaw-Harness;
Reasoning, Tool-Aufrufe und Tool-Ergebnisse aus der Quelle werden nicht in ihn kopiert.
Der private Verbindungsbereich bleibt über ausstehende und festgeschriebene Bindungszustände hinweg bestehen, sodass
jeder spätere Turn mit nativer Authentifizierung und Provider-Konfiguration über diese Verbindung verbleibt.
Deaktivierte Überwachung oder eine Abweichung der Bindung bzw. Verbindung führt zu einem sicheren Abbruch,
statt zum gewöhnlichen Harness im Agentenverzeichnis zu wechseln.

Die ursprüngliche CLI-, VS-Code-, Atlas- oder ChatGPT-Quelle bleibt für beide
Kataloge geeignet. Der kanonische Branch ist ein nativer Codex-Thread, seine Quellart lautet jedoch
`appServer`; native Clients können diese Quellart herausfiltern, sodass sein Erscheinen in
Codex Desktop nicht garantiert ist.

Aktive Quellen können weder einen neuen Branch starten noch archiviert werden; ein vorhandener überwachter
Chat kann weiterhin geöffnet werden. `notLoaded` bedeutet, dass die Aktivität unbekannt und nicht inaktiv ist;
OpenClaw erlaubt die Archivierung einer lokalen `idle`- oder `notLoaded`-Zeile erst nach einer expliziten
Bestätigung, dass keine andere Ausführung läuft, und einer aktuellen prozesslokalen Statusabfrage. Codex
serialisiert Thread-Mutationen innerhalb eines App-Server-Prozesses, stellt jedoch
keine exklusive prozessübergreifende Lease für Ausführungen oder Genehmigungsverantwortliche bereit, sodass diese Abfrage nicht
beweisen kann, dass kein anderer Prozess den Thread verwendet. OpenClaw blockiert einen bekannten
aktiven Bindungsverantwortlichen für das exakte Ziel oder jeden nicht archivierten erzeugten Nachfolger,
den die paginierte Codex-Nachfolgerabfrage zurückgibt. Aufzählungsfehler, Zyklen und
das Ausschöpfen von Sicherheitsgrenzen führen zu einem sicheren Abbruch. Die native Archivierung kann weiterhin mit einem neuen Turn
in einem anderen Prozess kollidieren, daher deckt die Bestätigung unbekannte Clients und die Lücke zwischen
Statusabfrage und Archivierung ab. Ein überwachter modellgebundener Chat kann nicht gelöscht werden, solange
er die native Bindung schützt.

Kataloge gekoppelter Nodes bleiben in der ersten Version reine Metadatenkataloge. Die aktuelle
Node-Aufrufgrenze arbeitet nach dem Anfrage-/Antwortprinzip und kann die langlebigen Turn-Ereignisse,
Genehmigungsanfragen oder Streaming-Ausgaben nicht übertragen, die eine echte Codex-Harness-Bindung
erfordert. Die entfernten Aktionen **Fortsetzen** und **Archivieren** bleiben daher auch dann nicht verfügbar,
wenn die Zeile inaktiv ist.

Informationen zur Einrichtung durch Betreiber und zum sichtbaren Verhalten der Control UI finden Sie unter
[Codex-Überwachung](/de/plugins/codex-supervision).

## Sichtbare Antworten und Heartbeats

Direkte bzw. Quell-Chat-Turns über das Codex-Harness verwenden standardmäßig die automatische abschließende
Assistentenzustellung für interne WebChat-Oberflächen und entsprechen damit dem Vertrag des Pi-Harness:
Der Agent antwortet normal, und OpenClaw veröffentlicht den abschließenden Text in der
Quellkonversation. Setzen Sie `messages.visibleReplies: "message_tool"`, um den
abschließenden Assistententext privat zu halten, sofern der Agent nicht `message(action="send")` aufruft.

Codex-Heartbeat-Turns erhalten standardmäßig `heartbeat_respond` im durchsuchbaren OpenClaw-Tool-Katalog,
damit der Agent festhalten kann, ob der Weckvorgang still bleiben oder eine Benachrichtigung auslösen soll.
Hinweise zur Heartbeat-Initiative werden als Entwickleranweisung für den Codex-Zusammenarbeitsmodus gesendet,
die auf den Heartbeat-Turn beschränkt ist; gewöhnliche Chat-Turns verbleiben im Codex-Standardmodus.
Wenn `HEARTBEAT.md` nicht leer ist, verweisen die Heartbeat-Anweisungen Codex auf die Datei,
statt deren Inhalt direkt einzubetten.

## Hook-Grenzen

| Ebene                                 | Verantwortlicher          | Zweck                                                               |
| ------------------------------------- | ------------------------ | ------------------------------------------------------------------- |
| OpenClaw-Plugin-Hooks                 | OpenClaw                 | Produkt-/Plugin-Kompatibilität zwischen OpenClaw- und Codex-Harnesses. |
| Codex-App-Server-Erweiterungsmiddleware | Gebündelte OpenClaw-Plugins | Adapterverhalten pro Turn um dynamische OpenClaw-Tools herum.        |
| Native Codex-Hooks                    | Codex                    | Low-Level-Codex-Lebenszyklus und native Tool-Richtlinie aus der Codex-Konfiguration. |

OpenClaw verwendet keine projektbezogenen oder globalen Codex-`hooks.json`-Dateien, um
Plugin-Verhalten zu routen. Für die Brücke für native Tools und Berechtigungen injiziert OpenClaw
eine Codex-Konfiguration pro Thread für `PreToolUse`, `PostToolUse`, `PermissionRequest`
und `Stop`.

Wenn Codex-App-Server-Genehmigungen aktiviert sind (`approvalPolicy` ist nicht
`"never"`), lässt die standardmäßig injizierte native Hook-Konfiguration `PermissionRequest`
aus, damit der App-Server-Prüfer von Codex und die OpenClaw-Genehmigungsbrücke echte
Eskalationen nach der Prüfung bearbeiten. Fügen Sie `permission_request` zu
`nativeHookRelay.events` hinzu, um die Kompatibilitätsweiterleitung dennoch zu erzwingen. Andere Codex-
Hooks wie `SessionStart` und `UserPromptSubmit` bleiben Steuerungen auf Codex-Ebene;
sie werden im v1-Vertrag nicht als OpenClaw-Plugin-Hooks bereitgestellt.

Bei dynamischen OpenClaw-Tools führt OpenClaw das Tool aus, nachdem Codex den
Aufruf angefordert hat, sodass Plugin- und Middleware-Verhalten im Harness-Adapter ausgeführt wird. Codex
Code Mode empfängt generische dynamische Ergebnisse als Text und serialisiert verschachtelte
dynamische Aufrufe; Aufrufer müssen JSON-ähnliche Ergebnisse analysieren und können sich bei der
gleichzeitigen Übermittlung nicht auf `Promise.all` verlassen. Bei Codex-nativen Tools verwaltet Codex den
kanonischen Tool-Datensatz; OpenClaw kann ausgewählte Ereignisse spiegeln, den
nativen Thread jedoch nicht umschreiben, sofern Codex dies nicht über App-Server- oder native Hook-
Callbacks verfügbar macht.

`PreToolUse`-Ereignisse im Berichtsmodus des Codex App Server verschieben die Plugin-Genehmigung auf die
entsprechende App-Server-Genehmigung. Wenn ein OpenClaw-`before_tool_call`-Hook
`requireApproval` zurückgibt, während die native Nutzlast `openclaw_approval_mode:
"report"` setzt, zeichnet die native Hook-Weiterleitung die Plugin-Genehmigungsanforderung auf und
gibt keine native Entscheidung zurück. Wenn Codex später die App-Server-Genehmigungsanfrage
für dieselbe Tool-Verwendung sendet, öffnet OpenClaw die Plugin-Genehmigungsaufforderung und
ordnet die Entscheidung Codex zu. Codex-`PermissionRequest`-Ereignisse bilden einen
separaten Genehmigungspfad und können weiterhin über OpenClaw-Genehmigungen geroutet werden,
wenn diese Brücke entsprechend konfiguriert ist.

Elementbenachrichtigungen des Codex App Server stellen außerdem asynchrone `after_tool_call`-
Beobachtungen für Abschlüsse nativer Tools bereit, die noch nicht von der nativen
`PostToolUse`-Weiterleitung abgedeckt werden. Diese dienen ausschließlich der Telemetrie und Kompatibilität; sie können den
nativen Tool-Aufruf weder blockieren, verzögern noch verändern.

Compaction- und LLM-Lebenszyklusprojektionen stammen aus Benachrichtigungen des Codex App Server
und dem Zustand des OpenClaw-Adapters, nicht aus nativen Codex-Hook-Befehlen.
`before_compaction`, `after_compaction`, `llm_input` und `llm_output` sind
Beobachtungen auf Adapterebene und keine bytegenauen Erfassungen der internen
Anfrage- oder Compaction-Nutzlasten von Codex.

Native Codex-App-Server-Benachrichtigungen `hook/started` und `hook/completed` werden
als `codex_app_server.hook`-Agentenereignisse für Verlauf und
Debugging projiziert. Sie rufen keine OpenClaw-Plugin-Hooks auf.

## v1-Unterstützungsvertrag

In Codex-Laufzeit v1 unterstützt:

| Oberfläche                                       | Unterstützung                                                                          | Begründung                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenAI-Modellschleife über Codex               | Unterstützt                                                                        | Der Codex-App-Server verwaltet den OpenAI-Turn, die native Fortsetzung von Threads und die native Fortsetzung von Tools.                                                                                                                                                                                                                                                                                                                                                                                          |
| OpenClaw-Kanalrouting und -zustellung         | Unterstützt                                                                        | Telegram, Discord, Slack, WhatsApp, iMessage und andere Kanäle bleiben außerhalb der Modelllaufzeit.                                                                                                                                                                                                                                                                                                                                                                                    |
| Dynamische OpenClaw-Tools                        | Unterstützt                                                                        | Codex fordert OpenClaw auf, diese Tools auszuführen, sodass OpenClaw Teil des Ausführungspfads bleibt.                                                                                                                                                                                                                                                                                                                                                                                                |
| Prompt- und Kontext-Plugins                    | Unterstützt                                                                        | OpenClaw projiziert OpenClaw-spezifische Prompts und Kontexte in den Codex-Turn, während Codex-eigene Basis-, Modell- und konfigurierte Projektdokument-Prompts im nativen Codex-Pfad verbleiben. OpenClaw deaktiviert die integrierte Persönlichkeit von Codex für native Threads, sodass Persönlichkeitsdateien im Agent-Arbeitsbereich maßgeblich bleiben. Native Codex-Entwickleranweisungen akzeptieren nur Befehlsanleitungen, deren Geltungsbereich ausdrücklich auf `codex_app_server` beschränkt ist; ältere globale Befehlshinweise bleiben für Nicht-Codex-Prompt-Oberflächen erhalten. |
| Lebenszyklus der Kontext-Engine                      | Unterstützt                                                                        | Zusammenstellung, Aufnahme und Wartung nach dem Turn werden um Codex-Turns herum ausgeführt. Kontext-Engines ersetzen nicht die native Codex-Compaction.                                                                                                                                                                                                                                                                                                                                                        |
| Hooks für dynamische Tools                            | Unterstützt                                                                        | `before_tool_call`, `after_tool_call` und die Tool-Ergebnis-Middleware werden um OpenClaw-eigene dynamische Tools herum ausgeführt.                                                                                                                                                                                                                                                                                                                                                                          |
| Lebenszyklus-Hooks                               | Als Adapterbeobachtungen unterstützt                                                | `llm_input`, `llm_output`, `agent_end`, `before_compaction` und `after_compaction` werden mit unverfälschten Nutzdaten für den Codex-Modus ausgelöst.                                                                                                                                                                                                                                                                                                                                                           |
| Überarbeitungs-Gate für die endgültige Antwort                    | Über native Hook-Weiterleitung unterstützt                                              | Codex `Stop` wird an `before_agent_finalize` weitergeleitet; `revise` fordert Codex vor dem Abschluss zu einem weiteren Modelldurchlauf auf.                                                                                                                                                                                                                                                                                                                                                                |
| Native Shell-, Patch- und MCP-Blockierung oder -Beobachtung | Über native Hook-Weiterleitung unterstützt                                              | Codex `PreToolUse` und `PostToolUse` werden für bestätigte native Tool-Oberflächen weitergeleitet, einschließlich MCP-Nutzdaten mit Codex-App-Server `0.142.0` oder neuer. Blockierung wird unterstützt, das Umschreiben von Argumenten jedoch nicht.                                                                                                                                                                                                                                                                               |
| Native Berechtigungsrichtlinie                      | Über Genehmigungen des Codex-App-Servers und die kompatible native Hook-Weiterleitung unterstützt | Genehmigungsanfragen des Codex-App-Servers werden nach der Codex-Prüfung über OpenClaw geleitet. Die native Hook-Weiterleitung `PermissionRequest` ist für native Genehmigungsmodi optional, da Codex sie vor der Guardian-Prüfung ausgibt.                                                                                                                                                                                                                                                                          |
| Erfassung der App-Server-Trajektorie                 | Unterstützt                                                                        | OpenClaw zeichnet die an den App-Server gesendete Anfrage und die empfangenen App-Server-Benachrichtigungen auf.                                                                                                                                                                                                                                                                                                                                                                                    |

In Codex-Laufzeit v1 nicht unterstützt:

| Oberfläche                                             | V1-Grenze                                                                                                                                     | Zukünftiger Pfad                                                                               |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Mutation nativer Tool-Argumente                       | Native Codex-Hooks vor der Tool-Ausführung können blockieren, OpenClaw schreibt jedoch keine Codex-nativen Tool-Argumente um.                                               | Erfordert Codex-Hook-/Schemasupport für ersetzende Tool-Eingaben.                            |
| Bearbeitbarer Codex-nativer Transkriptverlauf            | Codex verwaltet den kanonischen nativen Thread-Verlauf. OpenClaw verwaltet eine Spiegelung und kann zukünftigen Kontext projizieren, sollte aber nicht unterstützte Interna nicht verändern. | Explizite Codex-App-Server-APIs hinzufügen, falls Eingriffe in native Threads erforderlich sind.                    |
| `tool_result_persist` für Codex-native Tool-Datensätze | Dieser Hook transformiert OpenClaw-eigene Transkriptschreibvorgänge, nicht Codex-native Tool-Datensätze.                                                           | Transformierte Datensätze könnten gespiegelt werden, für das kanonische Umschreiben ist jedoch Codex-Unterstützung erforderlich.              |
| Umfangreiche native Compaction-Metadaten                     | OpenClaw kann native Compaction anfordern, erhält jedoch keine stabile Liste beibehaltener/verwerfener Elemente, kein Token-Delta, keine Abschlusszusammenfassung und keine Zusammenfassungsnutzdaten.   | Erfordert umfangreichere Codex-Compaction-Ereignisse.                                                     |
| Eingriff in die Compaction                             | OpenClaw erlaubt Plugins oder Kontext-Engines nicht, native Codex-Compaction abzulehnen, umzuschreiben oder zu ersetzen.                                             | Codex-Hooks vor/nach der Compaction hinzufügen, falls Plugins native Compaction ablehnen oder umschreiben müssen. |
| Bytegenaue Erfassung von Modell-API-Anfragen             | OpenClaw kann App-Server-Anfragen und -Benachrichtigungen erfassen, aber der Codex-Kern erstellt die endgültige OpenAI-API-Anfrage intern.                      | Erfordert ein Codex-Ereignis zur Nachverfolgung von Modellanfragen oder eine Debug-API.                                   |

## Native Berechtigungen und MCP-Abfragen

Für `PermissionRequest` gibt OpenClaw nur dann ausdrückliche Zulassen- oder Ablehnen-
Entscheidungen zurück, wenn die Richtlinie eine Entscheidung trifft. Ein Ergebnis ohne Entscheidung ist keine Zulassung: Codex
behandelt es als fehlende Hook-Entscheidung und greift auf den eigenen Guardian- oder Benutzer-
Genehmigungspfad zurück.

In den Genehmigungsmodi des Codex-App-Servers wird dieser native Hook standardmäßig ausgelassen. Dies
gilt, sofern `permission_request` nicht ausdrücklich in
`nativeHookRelay.events` enthalten ist oder eine Kompatibilitätslaufzeit ihn installiert.

Wenn ein Betreiber `allow-always` für eine native Codex-Berechtigungs-
anfrage auswählt, merkt sich OpenClaw diesen exakten Fingerabdruck aus Provider/Sitzung/Tool-Eingabe/cwd
für ein begrenztes Sitzungszeitfenster. Die gespeicherte Entscheidung gilt
absichtlich nur bei exakter Übereinstimmung: Ein geänderter Befehl, geänderte Argumente, Tool-Nutzdaten oder ein
geändertes cwd führen zu einer neuen Genehmigung.

Genehmigungsabfragen für Codex-MCP-Tools werden über den Plugin-Genehmigungs-
ablauf von OpenClaw geleitet, wenn Codex `_meta.codex_approval_kind` als `"mcp_tool_call"` kennzeichnet. Codex
`request_user_input` registriert eine providerneutrale Gateway-Frage für die
ursprüngliche Sitzung. Die Control UI zeigt die Gateway-Fragekarte an, und eine
einzelne nicht geheime Auswahl verwendet typisierte Kanalschaltflächen, wenn der Kanal sie unterstützt.
Das Antippen einer Schaltfläche, Antworten in der Control UI und die nächste in der Warteschlange befindliche Klartextantwort
lösen alle denselben Gateway-Datensatz auf, bevor OpenClaw die App-Server-Antwort zurückgibt.
Die automatische Auflösung durch Codex und abgebrochene Versuche begrenzen die Wartezeit und brechen den Datensatz ab.
Geheime Fragen verbleiben vollständig im mit einer Warnung versehenen Pfad für Textantworten. Andere MCP-
Abfrageanfragen werden sicher abgelehnt.

Den allgemeinen Plugin-Genehmigungsablauf, der diese Aufforderungen überträgt, finden Sie unter
[Plugin-Berechtigungsanfragen](/de/plugins/plugin-permission-requests).

## Warteschlangensteuerung

Die Steuerung der Warteschlange aktiver Ausführungen wird Codex app-server `turn/steer` zugeordnet. Mit dem
standardmäßigen `messages.queue.mode: "steer"` bündelt OpenClaw Chatnachrichten im Steuerungsmodus
für das konfigurierte Ruhefenster und sendet sie in der Reihenfolge ihres Eingangs als eine `turn/steer`-
Anfrage.

Codex-Reviews und manuelle Compaction-Durchläufe können die Steuerung im selben Durchlauf ablehnen. In
diesem Fall wartet OpenClaw, bis die aktive Ausführung abgeschlossen ist, bevor der
Prompt gestartet wird. Verwenden Sie `/queue followup` oder `/queue collect`, wenn Nachrichten
standardmäßig in die Warteschlange eingereiht statt zur Steuerung verwendet werden sollen. Siehe [Steuerungswarteschlange](/de/concepts/queue-steering).

## Hochladen von Codex-Feedback

Wenn `/diagnostics [note]` für eine Sitzung im nativen Codex-
Harness genehmigt wird, ruft OpenClaw für relevante
Codex-Threads zusätzlich Codex app-server `feedback/upload` auf, einschließlich Protokollen für jeden aufgeführten Thread und erzeugte Codex-
Unterthreads, sofern verfügbar.

Das Hochladen erfolgt über den regulären Feedbackpfad von Codex zu den OpenAI-Servern. Wenn
Codex-Feedback in diesem app-server deaktiviert ist, gibt der Befehl den
app-server-Fehler zurück. Die Antwort der abgeschlossenen Diagnose führt die Kanäle,
OpenClaw-Sitzungs-IDs, Codex-Thread-IDs und lokalen `codex resume <thread-id>`-
Befehle für die gesendeten Threads auf.

Wenn Sie die Genehmigung verweigern oder ignorieren, gibt OpenClaw diese Codex-IDs
nicht aus und sendet kein Codex-Feedback. Das Hochladen ersetzt nicht den lokalen
Export der Gateway-Diagnose. Informationen zu Genehmigung, Datenschutz, lokalem Paket und
Verhalten in Gruppenchats finden Sie unter [Diagnoseexport](/de/gateway/diagnostics).

Verwenden Sie `/codex diagnostics [note]` nur, wenn Sie Codex-Feedback
für den aktuell angehängten Thread ohne das vollständige Gateway-Diagnosepaket hochladen
möchten.

## Compaction und Transkriptspiegel

Wenn das ausgewählte Modell das Codex-Harness verwendet, gehört die native Thread-Compaction
zum Codex app-server. OpenClaw führt für
Codex-Durchläufe keine Compaction vorab aus, ersetzt die Codex-Compaction nicht durch die Compaction der Kontext-Engine und
greift nicht auf eine Zusammenfassung durch OpenClaw oder die öffentliche OpenAI-Schnittstelle zurück, wenn die native Compaction nicht
gestartet werden kann. OpenClaw verwaltet einen Transkriptspiegel für den Kanalverlauf, die Suche,
`/new`, `/reset` und zukünftige Wechsel des Modells oder Harness.

Explizite Compaction-Anfragen wie `/compact` oder ein von einem Plugin angeforderter manueller
Compaction-Vorgang starten die native Codex-Compaction mit `thread/compact/start`.
OpenClaw hält die Anfrage und die Lease des gemeinsam verwendeten Clients offen, bis Codex das
zugehörige Abschlusselement `contextCompaction` ausgibt, und meldet den Compaction-
Durchlauf anschließend als abgeschlossen. Wenn dieser abschließende Durchlauf das konfigurierte Compaction-
Zeitlimit überschreitet, fordert OpenClaw eine native Unterbrechung des Durchlaufs an. Die Lease und die Thread-spezifische
Compaction-Sperre bleiben bestehen, bis Codex den Endzustand meldet oder
den Unterbrechungs-RPC bestätigt. Wenn Codex dies nicht innerhalb der Kulanzfrist für die Unterbrechung
bestätigt, nimmt OpenClaw die Verbindung außer Betrieb, bevor die Sperre aufgehoben wird. Bei Remote-
Verbindungen wird außerdem die zugehörige Thread-Bindung getrennt, damit spätere Arbeiten
nicht mit einem unbestätigten Remote-Durchlauf überlappen können. Andere Durchläufe auf einer außer Betrieb genommenen Verbindung schlagen
fehl und können mit einem neuen Client wiederholt werden. Das Schließen des Clients, der Abbruch der Anfrage oder ein
fehlgeschlagener Compaction-Durchlauf führt zu einem fehlgeschlagenen Vorgang. Die automatische Compaction bei
Kontextdruck ist Aufgabe von Codex; OpenClaw startet die native Compaction nur bei manuell
angeforderten Auslösern.

Wenn eine Kontext-Engine die Thread-Bootstrap-Projektion für Codex anfordert, projiziert OpenClaw
Namen und IDs von Tool-Aufrufen, Eingabeformen und redigierte Inhalte von Tool-Ergebnissen
in den neuen Codex-Thread. Die Rohwerte der Argumente von Tool-Aufrufen werden
nicht in diese Projektion kopiert.

Der Spiegel umfasst den Benutzer-Prompt, den endgültigen Assistententext und kompakte
Codex-Datensätze zu Gedankengängen oder Plänen, wenn der app-server sie ausgibt. OpenClaw
zeichnet den Beginn und den Endstatus der nativen Compaction auf, stellt jedoch
weder eine für Menschen lesbare Compaction-Zusammenfassung noch eine überprüfbare Liste der
Einträge bereit, die Codex nach der Compaction beibehalten hat.

Da Codex Eigentümer des kanonischen nativen Threads ist, schreibt `tool_result_persist`
Codex-native Tool-Ergebnisdatensätze nicht um. Dies gilt nur, wenn OpenClaw
ein Tool-Ergebnis in ein OpenClaw-eigenes Sitzungstranskript schreibt.

## Medien und Zustellung

OpenClaw ist weiterhin für die Medienzustellung und die Auswahl des Medien-Providers zuständig. Bild-,
Video-, Musik-, PDF- und TTS-Funktionen sowie das Medienverständnis verwenden entsprechende Provider-/Modell-
Einstellungen wie `agents.defaults.mediaModels.image`,
`agents.defaults.mediaModels.video`, `pdfModel` und `tts`.

Text, Bilder, Video, Musik, TTS, Genehmigungen und Ausgaben von Messaging-Tools werden weiterhin
über den regulären Zustellungspfad von OpenClaw verarbeitet; die Mediengenerierung erfordert
die Legacy-Laufzeit nicht. Wenn Codex ein natives Bildgenerierungselement mit einem
`savedPath` ausgibt, leitet OpenClaw genau diese Datei über den regulären Antwortmedien-
Pfad weiter, selbst wenn der Codex-Durchlauf keinen Assistententext enthält.

## Verwandte Themen

- [Codex-Harness](/de/plugins/codex-harness)
- [Codex-Harness-Referenz](/de/plugins/codex-harness-reference)
- [Codex-Überwachung](/de/plugins/codex-supervision)
- [Native Codex-Plugins](/de/plugins/codex-native-plugins)
- [Plugin-Hooks](/de/plugins/hooks)
- [Agent-Harness-Plugins](/de/plugins/sdk-agent-harness)
- [Diagnoseexport](/de/gateway/diagnostics)
- [Trajektorienexport](/de/tools/trajectory)
