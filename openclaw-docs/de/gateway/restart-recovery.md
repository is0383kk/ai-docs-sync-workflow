---
read_when:
    - Sie möchten wissen, ob durch einen Neustart des Gateways laufende Agentenaufgaben verloren gehen
    - Ein Agentenlauf wurde durch einen Neustart, einen Absturz oder das erneute Laden der Konfiguration unterbrochen
    - Sie debuggen die automatische Sitzungswiederherstellung, nachdem das Gateway wieder verfügbar ist
summary: 'Was einen Gateway-Neustart oder -Absturz übersteht: Unterbrochene Agent-Durchläufe werden automatisch fortgesetzt, Subagenten und Hintergrundaufgaben werden wiederhergestellt, ausstehende Zustellungen werden abgearbeitet'
title: Wiederherstellung nach einem Neustart
x-i18n:
    generated_at: "2026-07-26T17:48:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdea30f3a90697951f4f63a06897d2c1d936e5145138b47fed7d8ebd8b7187ad
    source_path: gateway/restart-recovery.md
    workflow: 16
---

Ein Neustart des Gateways führt nicht zum Verlust des Agent-Status. Konversationen, Transkripte,
geplante Aufträge, Datensätze von Hintergrundaufgaben und ausgehende Nachrichten in der Warteschlange
werden sämtlich auf dem Datenträger gespeichert. Arbeit, die mitten in einem Durchlauf unterbrochen wurde,
wird erkannt und automatisch fortgesetzt, nachdem das Gateway wieder verfügbar ist. Die Wiederherstellung
ist immer aktiviert und erfordert normalerweise kein manuelles Eingreifen. Bei wiederholt fehlschlagender
Wiederherstellung wird die Anzahl der Versuche begrenzt und möglicherweise eine Sitzung isoliert, bis Sie
sie prüfen oder ersetzen.

Diese Seite beschreibt, was einen Neustart übersteht, wie unterbrochene Arbeit erkannt wird
und wie die automatische Fortsetzung abläuft.

## Was einen Neustart übersteht

| Status                        | Speicherort                                  | Verhalten bei einem Neustart                                          |
| ----------------------------- | -------------------------------------------- | --------------------------------------------------------------------- |
| Konversationsverlauf          | Agent-spezifische SQLite-Datenbank           | Unverändert; Sitzungen werden anhand des gespeicherten Transkripts fortgesetzt |
| Unterbrochener Durchlauf der Hauptsitzung | Agent-spezifische SQLite-Sitzungszeile und Transkript | Wird wenige Sekunden nach dem Start automatisch fortgesetzt oder abgeglichen |
| Subagent-Ausführungen         | SQLite (gemeinsame Statusdatenbank)          | Registry wird beim Start wiederhergestellt; unterbrochene Ausführungen werden fortgesetzt |
| Hintergrundaufgaben           | SQLite (gemeinsame Statusdatenbank)          | Werden beim Start abgeglichen; verwaiste Ausführungen werden wiederhergestellt oder als verloren markiert |
| Ausgehende Zustellungen in der Warteschlange | SQLite-Zustellungswarteschlange | Wird nach dem Neustart abgearbeitet; nicht zugestellte Antworten werden erneut versucht |
| Geplante (Cron-)Aufträge      | SQLite-Cron-Speicher                         | Zeitpläne bleiben erhalten; der Scheduler wird beim Start erneut aktiviert |
| Neustartfortsetzung           | SQLite-Neustart-Sentinel                     | Einmalige Fortsetzung wird an die Sitzung gesendet, die den Neustart angefordert hat |

## Ordnungsgemäße Neustarts schließen zunächst laufende Arbeit ab

Ein angeforderter Neustart (`openclaw gateway restart`, eine Konfigurationsänderung, die
einen Neustart erfordert, oder ein Gateway-Update) beendet laufende Arbeit nicht sofort. Das
Gateway nimmt keine neue Arbeit mehr an und wartet anschließend bis zum Ende eines Zeitbudgets
für den Abschluss laufender Arbeit (standardmäßig 5 Minuten), bis aktive Agent-Durchläufe und
Hintergrundaufgaben beendet sind. Daher unterbrechen die meisten Neustarts überhaupt nichts.

Nur Arbeit, die nicht innerhalb dieses Zeitbudgets abgeschlossen werden kann (oder eine durch
einen erzwungenen Neustart oder Absturz unterbrochene Ausführung), wird abgebrochen — zuvor wird
jedoch jede betroffene Sitzung für die Wiederherstellung markiert.

## Erkennung unterbrochener Arbeit

Drei sich ergänzende Mechanismen markieren Sitzungen, deren Durchlauf nicht abgeschlossen wurde:

- **Bei der Annahme eines Durchlaufs:** Bei einem gewöhnlichen Textdurchlauf in einer bestehenden Hauptsitzung
  fügt das Gateway die Benutzernachricht an, markiert die Sitzung als laufend und zeichnet
  ihren Zustellungsanspruch für die Wiederherstellung in einer einzigen SQLite-Transaktion auf, bevor das Modell oder
  der `before_agent_reply`-Hook ausgeführt wird. Control UI führt dies aus, bevor die
  `started`-Bestätigung zurückgegeben wird; der Channel-Versand führt es aus, wenn der vorbereitete Durchlauf
  die Agent-Ausführung übernimmt.
  Befehle, Anhänge, durchlaufspezifische Überschreibungen, ausstehende Zustellungen, vorherige Abbruchhinweise,
  Plugin-eigene Sitzungen und Durchläufe mit Ausführungs-Hooks behalten ihre
  spezialisierten Annahmepfade bei.
  Wenn ein `before_agent_reply`-Hook installiert ist, zeichnet die Annahme auch dessen Phase auf.
  Die Wiederherstellung spielt niemals einen mitten im Aufruf unterbrochenen Hook erneut ab. Sobald ein nicht behandelter Hook
  abgeschlossen ist, zeichnet sein Prüfpunkt dieses Ergebnis auf. Die Wiederherstellung bleibt jedoch
  solange aus Sicherheitsgründen deaktiviert, wie dieser Hook aktiv ist: Ein Prüfpunkt kann nicht nachweisen, dass nach dem
  Neustart derselbe Plugin-Code und dieselbe Konfiguration geladen wurden. Behandelte Text- und
  stille Ergebnisse erhalten getrennte Prüfpunkte, um eine deterministische Abwicklung zu ermöglichen.
  Dauerhafte Wiederherstellungsansprüche, die von älteren Versionen geschrieben wurden, enthalten keine Markierung
  für die Eigentümerschaft der Quelle und erhalten daher während eines Upgrades dieselbe aus Sicherheitsgründen
  deaktivierende Hook-Prüfung.
- **Beim Herunterfahren:** Während laufende Arbeit vor dem Neustart abgeschlossen wird, erhält jede Sitzung mit einer aktiven Ausführung
  eine Wiederherstellungsmarkierung im Sitzungsspeicher, bevor die Ausführung
  abgebrochen wird.
- **Beim Start:** Das Gateway durchsucht die Sitzungsspeicher nach Sitzungen, die weiterhin
  als laufend gekennzeichnet sind, aber im neuen Prozess keinen aktiven Eigentümer haben. Dadurch werden
  harte Abstürze und Prozessabbrüche erfasst, bei denen kein Code zum Herunterfahren ausgeführt wurde. Veraltete
  Sperrdateien für Transkripte werden gleichzeitig bereinigt.

## Automatische Fortsetzung

Wenige Sekunden nach dem Start sendet das Gateway jede markierte Sitzung erneut
mit einer synthetischen Systemnachricht, die dem Agent mitteilt, dass sein vorheriger Durchlauf
durch einen Neustart unterbrochen wurde und er ihn anhand des vorhandenen Transkripts fortsetzen soll. Wenn
bereits eine abschließende Antwort erzeugt, aber noch nicht zugestellt wurde, wird ihr Text aufgenommen,
damit der Agent sie zustellen kann, anstatt die Arbeit erneut auszuführen.

Der Abgleich beim Start wiederholt vorübergehende Fehler bis zu dreimal mit
exponentiellem Backoff. Unabhängig davon verfügt jeder unterbrochene Hauptsitzungszyklus über ein
dauerhaftes Budget von drei angerechneten automatischen Versandversuchen, das auch über
Gateway-Neustarts hinweg erhalten bleibt. OpenClaw rechnet einen Versuch vor dem Versand an, erstattet ihn, wenn
das Gateway die Anfrage vor der Annahme ausdrücklich ablehnt, und behält die
Anrechnung bei, wenn ein Ergebnis nach dem Versand ungewiss ist, um eine erneute Ausführung der Arbeit zu vermeiden.
Vordergrundarbeit, der die Sitzung bereits gehört, verhindert eine automatische Wiederherstellung,
bis diese Arbeit abgeschlossen ist.

Nachdem das dauerhafte Budget erschöpft ist, wird die Sitzung mit einem Tombstone versehen, anstatt
endlos in einer Schleife fortzufahren. Prüfen Sie die fehlgeschlagene Sitzung und verwenden Sie `/new` oder `/reset`, um einen
Ersatz zu starten. `openclaw doctor --fix` kann eine veraltete Abbruchmarkierung reparieren, die
mit einem Tombstone in Konflikt steht, aktiviert diesen Wiederherstellungszyklus jedoch nicht erneut.

Jeder Wiederholungsversuch verwendet dieselbe dauerhafte Versandkennung, sodass ein mehrdeutiger
Verbindungsfehler dieselbe Wiederherstellung nicht zweimal starten kann. Abgeschlossene und nicht fortsetzbare
Control-UI-Durchläufe behalten ebenfalls begrenzte dauerhafte Idempotenz-Tombstones, sodass eine
erneut verbundene Outbox sie entfernen kann, ohne die Anfrage erneut auszuführen.

Antworten, die ausschließlich das Nachrichtenwerkzeug verwenden, nutzen eine zweite dauerhafte Korrelation. Bevor eine abschließende
Sendung innerhalb derselben Konversation den Channel erreicht, zeichnet das Gateway eine nicht aufgelöste
Zustellungsabsicht für die exakte Sitzung und den Quelldurchlauf auf. Ein bestätigter Erfolg beim Provider
löst sie in einen dauerhaften Zustellungsbeleg auf; ein bestätigter Fehler entfernt
sie. Die Wiederherstellung schließt einen Zustellungsbeleg ab, ohne Werkzeuge erneut auszuführen. Wenn ein Absturz
den Ausgang beim Provider unbekannt lässt, wird die Wiederherstellung aus Sicherheitsgründen beendet, anstatt
einen externen Effekt erneut auszuführen.

Die zugestellte Antwort wird außerdem mit ihrer Quellnachrichten-ID in das Transkript gespiegelt.
Abschließende Spiegelungen verwenden einen eigenen Belegschlüssel, sodass eine Fortschrittsmeldung mit
demselben Idempotenzschlüssel des Providers die abschließende Markierung nicht verdecken kann. Fortschrittsmeldungen
und Belege aus älteren Durchläufen können den aktuellen Durchlauf nicht abschließen. Nur
dauerhafte Ansprüche vom Channel-Eingang können die Berechtigung für Nachrichtenaktionen wiederherstellen. Eine fortgesetzte
Ausführung behält den ursprünglichen Modus der Quellzustellung und die Quellkorrelation einschließlich
der Identität des Anforderers und etwaiger Einschränkungen auf denselben Channel oder Thread bei, sodass derselbe Beleg
auch dann maßgeblich bleibt, wenn während der Wiederherstellung ein weiterer Neustart erfolgt. Ein
ausschließlich auf dem Nachrichtenwerkzeug basierender Durchlauf ohne rekonstruierbare Channel-Berechtigung wird
aus Sicherheitsgründen beendet und erhält den einmaligen Hinweis zum erneuten Senden.

Vor der Fortsetzung prüft das Gateway, ob das Ende des Transkripts sicher als
Fortsetzungspunkt verwendet werden kann. Ist dies nicht der Fall (wenn der Durchlauf beispielsweise mit einer veralteten ausstehenden
Genehmigung endete), wird die Sitzung nicht blind erneut ausgeführt; der Agent veröffentlicht stattdessen einen kurzen
Hinweis mit der Bitte, die letzte Anfrage erneut zu senden. Bei WebChat wird dieser Hinweis
direkt in den Sitzungsverlauf geschrieben, sodass er nach dem erneuten Verbinden sichtbar bleibt.

OpenClaw kann außerdem unterbrochene schreibgeschützte Arbeit im [Code Mode](/de/tools/code-mode)
rekonstruieren. Code Mode markiert diese Ausführungen als neustartsicher und weist Katalogwerkzeuge
oder Plugin-Namespaces mit Seiteneffekten zurück, bevor sie ausgeführt werden. Wenn ein Neustart beim
`wait`-Steuerelement erfolgt, rekonstruiert das neue Gateway den Durchlauf anhand seines Transkripts
und erzwingt, dass die rekonstruierte Ausführung neustartsicher bleibt, selbst wenn das
Modell diese Markierung auslässt oder löscht. Der Host beschränkt den gesamten rekonstruierten
Durchlauf auf geprüfte schreibgeschützte Kernwerkzeuge und ausdrücklich wiedergabesichere Plugin-Werkzeuge,
auch wenn Code Mode nach dem Neustart deaktiviert ist. Arbeit mit Seiteneffekten
bleibt durch den Hinweis zum erneuten Senden geschützt, statt einen doppelten Schreibvorgang zu riskieren.

### Subagents

Subagent-Ausführungen werden dauerhaft in der gemeinsamen SQLite-Statusdatenbank gespeichert, sodass die
Subagent-Registry den Prozess übersteht. Beim Start wird die Registry wiederhergestellt und
unterbrochene Subagent-Sitzungen werden mit ihrem ursprünglichen Aufgabenkontext fortgesetzt.
Dabei gelten zwei Sicherheitsmechanismen:

- Ausführungen, die vor mehr als 2 Stunden unterbrochen wurden, werden abgeschlossen, statt fortgesetzt, sodass
  ein Gateway, das über Nacht nicht verfügbar war, keine veraltete Arbeit wieder aufnimmt.
- Eine Sitzung, deren Wiederherstellung wiederholt fehlschlägt, wird als blockiert mit einem Tombstone versehen, damit
  die Wiederherstellung nicht endlos wiederholt werden kann.

### Hintergrundaufgaben

Die [Registry für Hintergrundaufgaben](/de/automation/tasks) basiert auf SQLite und wird
beim Start sowie in regelmäßigen Abständen abgeglichen: Dauerhafte Ergebnisse, die von
abgeschlossenen Ausführungen aufgezeichnet wurden, werden wiederhergestellt, und Ausführungen, deren zuständiger Prozess verschwunden ist,
werden nach einer Kulanzfrist als verloren markiert, statt dauerhaft hängen zu bleiben.

### Vom Agent angeforderte Neustarts

Wenn der Agent selbst einen Neustart auslöst (durch Anwenden einer Konfigurationsänderung, Aktualisieren
des Gateways oder eine ausdrückliche Neustartanforderung), wird ein Neustart-Sentinel in
SQLite geschrieben, bevor der Prozess beendet wird. Nach dem Start meldet das Gateway das Ergebnis an
den ursprünglichen Chat zurück und sendet einen einmaligen Fortsetzungsdurchlauf, damit der
Agent genau dort weitermacht, wo er aufgehört hat, und zwar im selben Channel und Thread.

Die typisierten SQLite-Spalten des Sentinels sind für die Neustartbehandlung maßgeblich;
sein `payload_json`-Wert dient nur als Wiedergabe- und Debugging-Kopie. Die Laufzeit liest, schreibt
und löscht den SQLite-Status ohne dateibasierten Fallback. Während der Speicherumstellung wird
beim Start und über Doctor eine begrenzte Statusmigration ausgeführt, um ein validiertes
`restart-sentinel.json` zu erhalten, das der ältere Prozess nach einem Update hinterlassen hat.
Die Migration überprüft die typisierte Zeile und entfernt die Quelldatei, bevor die normale
Neustartbehandlung fortgesetzt wird.

## Sicherheitsmechanismen und Beobachtbarkeit

- **Schutz vor Absturzschleifen:** 3 unsaubere Starts innerhalb von 5 Minuten lösen einen Schutzmechanismus aus, der
  beim nächsten Start den automatischen Start von Nebendiensten unterdrückt, damit ein abstürzendes Gateway
  die Auswirkungen nicht selbst verstärkt. Der Normalbetrieb wird wiederhergestellt, sobald das Zeitfenster für unsaubere Starts abgelaufen ist.
- **Versuchslimit der Hauptsitzung:** drei angerechnete automatische Versandversuche
  pro unterbrochenem Zyklus; bei Ausschöpfung wird diese Sitzung mit einem Tombstone versehen, bis sie
  geprüft und ersetzt wird.
- **Metriken:** Wiederherstellungsaktivitäten werden über
  [Prometheus](/de/gateway/prometheus) als `openclaw_session_recovery_total` und
  `openclaw_session_recovery_age_seconds` exportiert.
- **Protokolle:** Entscheidungen zur Wiederherstellung werden in den
  Subsystemen `main-session-restart-recovery` und `subagent-interrupted-resume`
  protokolliert.

## Was nicht fortgesetzt wird

- Sitzungen, die von der Wiederherstellung der Hauptsitzung ausgeschlossen sind, weil sie bereits von einem anderen Eigentümer
  verwaltet werden: Subagent-Sitzungen (Subagent-Wiederherstellung), Cron-Sitzungen (der
  Scheduler führt sie planmäßig erneut aus) und von ACP verwaltete Sitzungen (die verbundene IDE
  oder der Client ist für die Fortsetzung zuständig).
- Sitzungen, deren Transkriptende nicht sicher fortgesetzt werden kann; diese erhalten statt einer
  stillen erneuten Ausführung den oben beschriebenen Hinweis zum erneuten Senden.
- Arbeit, die nie angenommen wurde: Nachrichten, die während des Zeitfensters zum Abschluss laufender Arbeit eintreffen,
  werden mit einem ausdrücklichen Neustartfehler abgelehnt, statt unbemerkt in die Warteschlange
  eines endenden Prozesses aufgenommen zu werden.
- Eigenständige eingebettete Durchläufe können keine Hauptsitzung mit ausstehender
  Neustartwiederherstellung übernehmen, da sie nicht denselben Lebenszyklus-Eigentümer wie das Gateway verwenden.
  Führen Sie den Durchlauf über das Gateway aus oder setzen Sie ihn dort mit `/new` oder `/reset` zurück.
