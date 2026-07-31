---
read_when:
    - Sie möchten verstehen, über welche Sitzungswerkzeuge der Agent verfügt
    - Sie möchten sitzungsübergreifenden Zugriff oder das Starten von Sub-Agenten konfigurieren
    - Sie möchten den Status gestarteter Sub-Agenten prüfen
summary: Agentenwerkzeuge für sitzungsübergreifenden Status, Erinnerungsabruf, Nachrichtenaustausch und die Orchestrierung von Sub-Agenten
title: Sitzungswerkzeuge
x-i18n:
    generated_at: "2026-07-26T18:25:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ceaf48addc9fc57afe2f6428cda03ed8b19f4efce93b13b58b7ef493a41c62fe
    source_path: concepts/session-tool.md
    workflow: 16
---

OpenClaw stellt Agenten Werkzeuge bereit, um sitzungsübergreifend zu arbeiten, den Status zu prüfen und Sub-Agenten zu orchestrieren.

## Verfügbare Werkzeuge

| Werkzeug                 | Funktion                                                                |
| -------------------- | --------------------------------------------------------------------------- |
| `sessions`           | Sichtbare Sitzungseinstellungen ändern und den globalen Sitzungskatalog verwalten  |
| `sessions_list`      | Sitzungen mit optionalen Filtern auflisten (Art, Bezeichnung, Agent, Archiv, Vorschau)  |
| `sessions_search`    | Sichtbare Sitzungstranskripte durchsuchen und passende Auszüge zurückgeben             |
| `sessions_history`   | Das Transkript einer bestimmten Sitzung lesen                                   |
| `sessions_send`      | Eine andere Sitzung auf demselben Gateway ausführen und optional warten                 |
| `conversations_list` | Stabile externe Konversationsadressen auflisten                                 |
| `conversations_send` | An genau eine externe Konversation senden, ohne eine lokale Sitzung auszuführen     |
| `conversations_turn` | An genau eine externe Konversation senden und auf die zugehörige Antwort warten   |
| `sessions_spawn`     | Eine isolierte Sub-Agenten-Sitzung für Hintergrundarbeiten starten                     |
| `sessions_yield`     | Den aktuellen Turn beenden und auf nachfolgende Ergebnisse von Sub-Agenten warten               |
| `subagents`          | Hintergrundarbeiten in diesem Sitzungsbaum auflisten oder abbrechen                         |
| `session_status`     | Eine Karte im Stil von `/status` anzeigen und optional eine sitzungsspezifische Modellüberschreibung festlegen |

Diese Werkzeuge unterliegen weiterhin dem aktiven Werkzeugprofil und der Zulassungs-/Verweigerungsrichtlinie. `tools.profile: "coding"` umfasst den vollständigen Satz zur Sitzungsorchestrierung. `tools.profile: "messaging"` umfasst die Selbstverwaltung von Sitzungen, Erkennung, Abruf, sitzungsübergreifende Nachrichtenübermittlung, Werkzeuge für externe Konversationen und den vollständigen Startlebenszyklus (`sessions_spawn`, `sessions_yield` und `subagents`). Die ausschließlich für die Benutzeroberfläche vorgesehenen Werkzeuge für Aufgabenvorschläge `spawn_task` und `dismiss_task` bleiben Werkzeuge des Coding-Profils.

Gruppen-, Provider-, Sandbox- und agentenspezifische Richtlinien können diese Werkzeuge nach der Profilphase weiterhin entfernen. Verwenden Sie `/tools` in der betroffenen Sitzung, um die tatsächlich verfügbare Werkzeugliste zu prüfen.

## Sitzungen auflisten und lesen

`sessions_list` gibt fokussierte Erkennungszeilen zurück: Sitzungsschlüssel, Agent, Art, Kanal, Bezeichnungs-/Titel-/Vorschaufelder, Beziehungen zu über- und untergeordneten Sitzungen, letzte Aktualisierung, Archivierungs-/Anheftungsstatus, Statusversion, Modell, Kontext-/Gesamtzahl der Token, Ausführungsstatus und ob die letzte Ausführung abgebrochen wurde. Filtern Sie nach `kinds` (Array; akzeptierte Werte: `main`, `group`, `cron`, `hook`, `node`, `other`), genauem `label`, genauem `agentId`, `search`-Text oder Aktualität (`activeMinutes`). Standardmäßig werden aktive Sitzungen zurückgegeben; übergeben Sie stattdessen `archived: true`, um archivierte Sitzungen zu prüfen. Legen Sie `includeDerivedTitles`, `includeLastMessage` oder `messageLimit` (begrenzt auf 20) fest, wenn Sie eine postfachähnliche Triage benötigen: einen vom Sichtbarkeitsumfang abhängigen abgeleiteten Titel, einen Vorschauauszug der letzten Nachricht oder eine begrenzte Anzahl aktueller Nachrichten pro Zeile. Zustellungsrouting, interne Sitzungs-IDs, laufbezogene Zeitmessungen/Einstellungen, Kostenschätzungen und Transkriptpfade werden bewusst ausgelassen; verwenden Sie für diese eigentümerspezifischen Details `session_status`, Konversationswerkzeuge und `sessions_history`. Abgeleitete Titel und Vorschauen werden nur für Sitzungen erzeugt, die der Aufrufer gemäß der konfigurierten Sichtbarkeitsrichtlinie für Sitzungswerkzeuge bereits sehen kann, sodass nicht zugehörige Sitzungen verborgen bleiben. Bei eingeschränkter Sichtbarkeit gibt `sessions_list` optionale `visibility`-Metadaten zurück, die den effektiven Modus und einen Hinweis darauf enthalten, dass die Ergebnisse möglicherweise auf den Geltungsbereich beschränkt sind.

`sessions_history` ruft das Konversationstranskript für eine bestimmte Sitzung ab. Standardmäßig sind Werkzeugergebnisse ausgeschlossen; übergeben Sie `includeTools: true`, um sie anzuzeigen. Verwenden Sie `limit` für den neuesten begrenzten Endabschnitt. Übergeben Sie `offset: 0`, wenn Sie Paginierungsmetadaten benötigen, und übergeben Sie anschließend die zurückgegebenen `nextOffset`-Werte, um rückwärts durch ältere OpenClaw-Transkriptfenster zu blättern, ohne rohe Transkriptdateien zu lesen. Explizite Offset-Seiten führen keine externen CLI-Fallback-Importe zusammen; verwenden Sie die standardmäßige Ansicht des neuesten Endabschnitts (ohne `offset`), wenn Sie diesen zusammengeführten Anzeigeverlauf benötigen.

Die zurückgegebene Ansicht ist bewusst begrenzt und sicherheitsgefiltert:

- Assistententext wird vor dem Abruf normalisiert:
  - Thinking-Tags werden entfernt
  - `<relevant-memories>`- / `<relevant_memories>`-Gerüstblöcke werden entfernt
  - XML-Nutzlastblöcke für Werkzeugaufrufe im Klartext wie `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>` und `<function_calls>...</function_calls>` werden entfernt, einschließlich abgeschnittener Nutzlasten, die nie ordnungsgemäß geschlossen werden
  - herabgestufte Gerüststrukturen für Werkzeugaufrufe/-ergebnisse wie `[Tool Call: ...]`, `[Tool Result ...]` und `[Historical context ...]` werden entfernt
  - offengelegte Modellsteuerungs-Token wie `<|assistant|>`, andere ASCII-`<|...|>`-Token und vollbreite `<｜...｜>`-Varianten werden entfernt
  - fehlerhaftes MiniMax-Werkzeugaufruf-XML wie `<invoke ...>` / `</minimax:tool_call>` wird entfernt
- Text, der Anmeldedaten oder Token ähnelt, wird vor der Rückgabe geschwärzt
- lange Textblöcke werden gekürzt
- bei sehr großen Verläufen können ältere Zeilen entfallen oder eine übergroße Zeile durch `[sessions_history omitted: message too large]` ersetzt werden
- das Werkzeug meldet Zusammenfassungskennzeichen wie `truncated`, `droppedMessages`, `contentTruncated`, `contentRedacted`, `bytes` sowie Paginierungsmetadaten

Verwenden Sie den zurückgegebenen **Sitzungsschlüssel** (wie `"main"`) mit `sessions_history`, `sessions_send` und `session_status`. Diese Zielwerkzeuge können auch eine bekannte Sitzungs-ID auflösen, aber `sessions_list` legt keine internen IDs offen.

Wenn Sie das exakte Rohtranskript benötigen, prüfen Sie die Transkriptzeilen in der SQLite-Datenbank innerhalb des Geltungsbereichs, statt `sessions_history` als ungefilterte Ausgabe zu behandeln.

Verwenden Sie [`sessions_search`](/de/concepts/session-search) für den exakten Volltextabruf in sichtbarem Transkripttext von Benutzern und Assistenten. Die Ergebnisse enthalten ein `sessionKey` für einen nachfolgenden `sessions_history`-Aufruf; Sichtbarkeitsfilterung, Schwärzung von Auszügen und Ausgabegrenzen entsprechen der Verlaufsgrenze.

## Sitzungseinstellungen und Gruppen verwalten

Das eigentümergeschützte Werkzeug `sessions` stellt zwei begrenzte Selbstverwaltungsbereiche bereit:

- `action: "patch"` ändert standardmäßig die aktuelle Sitzung oder eine andere sichtbare Sitzung, die mit `sessionKey` ausgewählt wurde. Es kann die Bezeichnung, das Seitenleistensymbol, den Anheftungs-/Archivierungsstatus, das Modell und die Thinking-Stufe festlegen. Es stellt keine Aktionen zum Zurücksetzen, Löschen oder zur Compaction bereit.
- `group_list`, `group_set`, `group_rename` und `group_delete` verwalten den globalen geordneten Sitzungskatalog. `group_set` ersetzt die geordnete Namensliste, anstatt einen einzelnen Eintrag zu ändern.

Eine von einem Agenten ausgewählte Modelländerung bleibt rückgängig zu machen, bis mit dieser Auswahl eine Ausführung erfolgreich abgeschlossen wurde. Wenn das ausgewählte Modell aufgrund eines Authentifizierungs-, Abrechnungs- oder „Modell nicht gefunden“-Fehlers definitiv nicht verwendbar ist, stellt OpenClaw das vorherige Modell wieder her und schreibt einen sichtbaren Systemhinweis. Vorübergehende Fehler durch Ratenbegrenzung, Überlastung, Zeitüberschreitung, Netzwerk oder Server machen die Auswahl nicht rückgängig.

## Sitzungen im Vergleich zu Konversationen

Eine **Sitzung** ist lokaler Modellkontext. Eine **Konversation** ist eine exakte externe Adresse, etwa ein einzelner Kommunikationspartner, Kanal oder Thread. Beide sind verknüpft, aber nicht austauschbar: Direktnachrichten können eine gemeinsame `main`-Sitzung verwenden und gleichzeitig separate Konversationsadressen behalten.

`conversations_list` gibt undurchsichtige `conversationRef`-Werte für den aktiven Agenten zurück. Mit einem expliziten `channel` aktualisiert das Gateway außerdem Adressen aus dem lokalen Verzeichnis dieses Kanals, etwa genehmigte Reef-Kommunikationspartner; verwenden Sie `query`, um einen bestimmten Kommunikationspartner außerhalb der aktuellen Ergebnisseite zu finden. Die Erkennung katalogisiert die Adresse, ohne eine Modellkontextsitzung zu erstellen; die zugrunde liegende Sitzung wird erst erstellt, wenn die Zustellung oder der eingehende Kontext sie benötigt. Konversationserkennung und -zustellung sind ausschließlich dem Eigentümer vorbehalten, da sie die Kanalanmeldedaten des Gateways verwenden. Verwenden Sie `conversations_send` für eine Zustellung ohne Warten auf das Ergebnis. Verwenden Sie `conversations_turn`, wenn die entfernte Antwort zum aktuellen Modell-Turn gehört: Das Gateway reserviert eine Transportnachrichten-ID, persistiert einen Zustellungsvorgang und eine Warteschlangenabsicht vor der Transport-E/A und gibt die zugehörige Antwort über das Werkzeug zurück, anstatt einen zweiten lokalen Agenten-Turn zu starten. Zustellungsvorgänge liegen außerhalb von Modelltranskripten; eine erfasste Antwort wird nur als Nebenartefakt aufbewahrt, während das Werkzeugergebnis den Modellkontext enthält. Wenn das Gateway nach dem Einreihen in die Warteschlange neu startet, kann die Zustellung wiederhergestellt werden, eine spätere Antwort folgt jedoch der gewöhnlichen Verarbeitung eingehender Nachrichten, da der prozesslokale Wartemechanismus nicht mehr vorhanden ist. Unaufgeforderte eingehende Nachrichten durchlaufen immer den normalen Kanalverarbeitungspfad.

Verwenden Sie das gemeinsame Werkzeug `message`, wenn Sie bereits ein explizites rohes Kanalziel haben oder eine kanalspezifische Aktion benötigen. Konversationsreferenzen sind auf den aktiven Agenten beschränkt und sollten über `conversations_list` bezogen und nicht aus Sitzungsschlüsseln konstruiert werden.

Im Code-Modus verwenden die Konversationswerkzeuge ihre exakten Gateway-Ausgabeverträge erneut. Eine einzelne `exec`-Zelle kann Adressen auflisten, ein zurückgegebenes `conversationRef` auswählen und `conversations_send` oder `conversations_turn` aufrufen; die normalen Werkzeugrichtlinien und Genehmigungen gelten weiterhin für die verschachtelten Aufrufe.

## Sitzungsübergreifende Nachrichten senden

`sessions_send` führt eine andere Sitzung auf demselben Gateway aus und wartet optional auf die Antwort. Sein `sessionKey`, `label` oder `agentId` wählt lokalen Modellkontext aus, kein externes Ziel. Die resultierende Antwort kann weiterhin über den bestehenden Zustellungskontext des Anforderers oder Ziels angekündigt werden; dieses bestehende Verhalten bleibt unverändert. Verwenden Sie für die exakte externe Zustellung ein Konversationswerkzeug oder `message` mit einem expliziten Kanal und Ziel.

- **Ohne Warten auf das Ergebnis:** Legen Sie `timeoutSeconds: 0` fest, um den Auftrag einzureihen und sofort zurückzukehren.
- **Auf Antwort warten:** Legen Sie eine Zeitüberschreitung fest und erhalten Sie die Antwort direkt.

Auf Threads beschränkte Chatsitzungen, etwa Schlüssel mit der Endung `:thread:<id>`, sind keine gültigen `sessions_send`-Ziele. Verwenden Sie für die agentenübergreifende Koordination den Sitzungsschlüssel des übergeordneten Kanals, damit über Werkzeuge weitergeleitete Nachrichten nicht innerhalb eines aktiven, für Menschen sichtbaren Threads erscheinen.

Nachrichten und A2A-Folgeantworten werden in der empfangenden Eingabeaufforderung (`[Inter-session message ... isUser=false]`) und in der Transkriptherkunft als sitzungsübergreifende Daten gekennzeichnet. Der empfangende Agent sollte sie als über Werkzeuge weitergeleitete Daten behandeln, nicht als direkte Anweisung eines Endbenutzers.

Nachdem das Ziel geantwortet hat, kann OpenClaw eine **Rückantwortschleife** ausführen, bei der die Agenten bis zum integrierten Limit abwechselnd Nachrichten senden. Der Zielagent kann mit `REPLY_SKIP` antworten, um die Schleife vorzeitig zu beenden.

Übergeben Sie `watch: true`, um den Absender zusätzlich als Beobachter von Statusänderungen des Ziels zu registrieren: Wenn später ein anderer Akteur dem Ziel eine direkte menschliche Nachricht sendet oder sein Ziel ändert, erhält der Absender einen Systemhinweis, der auf `session_status` `changesSince` verweist. Die Registrierung erfolgt nach erfolgreicher Weiterleitung, bezieht sich auf die Sitzung, die die Nachricht tatsächlich erhalten hat, und beginnt bei deren aktueller Statusversion, sodass nur spätere Änderungen Hinweise erzeugen. Das Ergebnis meldet `watched: true`, wenn die Registrierung erfolgreich war. Siehe [Bewusstsein für den Sitzungsstatus](/de/concepts/session-state).

## Status- und Orchestrierungshilfen

`session_status` ist das leichtgewichtige, zu `/status` äquivalente Werkzeug für die aktuelle oder eine andere sichtbare Sitzung. Es meldet Nutzung, Zeit, Modell-/Laufzeitstatus und gegebenenfalls den Kontext verknüpfter Hintergrundaufgaben. Wie `/status` kann es lückenhafte Token-/Cache-Zähler anhand des neuesten Transkriptnutzungseintrags ergänzen, und `model=default` entfernt eine sitzungsspezifische Überschreibung. Verwenden Sie `sessionKey="current"` für die aktuelle Sitzung des Aufrufers; sichtbare Clientbezeichnungen wie `openclaw-tui` sind keine Sitzungsschlüssel.

Wenn Routenmetadaten verfügbar sind, enthält `session_status` außerdem einen sichtbaren `Route context`-JSON-Block und entsprechende strukturierte `details`-Felder. Diese Felder unterscheiden den Sitzungsschlüssel von der Route, die derzeit den laufenden Live-Durchlauf verarbeitet:

- `origin` gibt an, wo die Sitzung erstellt wurde, oder den aus einem zustellbaren Sitzungsschlüsselpräfix abgeleiteten Provider, wenn in einem älteren Zustand keine Ursprungsmetadaten gespeichert sind.
- `active` ist die aktuelle Route des Live-Durchlaufs. Sie wird nur für die Live-Sitzung oder die aktuell verarbeitete Sitzung gemeldet.
- `deliveryContext` ist die in der Sitzung gespeicherte persistente Zustellroute, die OpenClaw für spätere Zustellungen wiederverwenden kann, selbst wenn die aktive Oberfläche davon abweicht.

## Änderungen des Sitzungszustands

OpenClaw führt ein dauerhaftes Signalprotokoll wesentlicher Änderungen des Sitzungszustands (direkte menschliche Nachrichten an überwachte Sitzungen, Ergebnisse untergeordneter Durchläufe, Zieländerungen, Compaction). `sessions_list`-Zeilen und `session_status` stellen die `stateVersion` der Sitzung bereit, und `session_status` akzeptiert `changesSince: <version>`, um die typisierten Ereignisse nach dieser Version zurückzugeben. Dabei signalisiert `historyGap` exakt, wenn die angeforderte Version älter als der aufbewahrte Verlauf ist. Überwachende Instanzen – automatisch übergeordnete Erstellungsinstanzen, explizit über `sessions_send watch: true` – erhalten einen einzigen zusammengefassten Hinweis auf einen veralteten Zustand, wenn ein anderer Akteur eine überwachte Sitzung ändert.

Zustandsänderungsereignisse lassen wiederholte Sitzungs-/Agenten-IDs aus und stellen nur für das Modell relevante Nutzlastfelder bereit (`outcome`, `channel` oder `turns`). Die Ereigniszusammenfassung sowie die Akteur-/Durchlaufkennungen bleiben für die Abstimmung verfügbar.

Das vollständige Modell mit Ereignisarten, Registrierung überwachender Instanzen, dem Anti-Spam-Hinweisprotokoll, Abstimmungsablauf und aktuellen Einschränkungen finden Sie unter [Bewusstsein für den Sitzungszustand](/de/concepts/session-state).

`sessions_yield` beendet absichtlich den aktuellen Turn, damit die nächste Nachricht das erwartete Folgeereignis sein kann. Verwenden Sie dies nach dem Erstellen von Unteragenten, wenn Abschlussergebnisse als nächste Nachricht eintreffen sollen, anstatt Polling-Schleifen zu erstellen.

`subagents` ist die Sitzungssbaumansicht nativer Unteragentendurchläufe und des gemeinsamen Ledgers für Hintergrundaufgaben. `action: "list"` meldet aktive/kürzlich ausgeführte Unteragenten sowie bereichsbezogene ACP-, CLI-/Medien- und Cron-Aufgaben. `action: "cancel"` akzeptiert einen zurückgegebenen `taskId` und kann nur Arbeit innerhalb des vom Aufrufer kontrollierten Sitzungsbaums beenden; Unteragenten auf Blattebene können die Aufgabe einer anderen Sitzung nicht abbrechen.

## Erstellen von Unteragenten

`sessions_spawn` erstellt standardmäßig eine isolierte Sitzung für eine Hintergrundaufgabe. Der Vorgang ist immer nicht blockierend und gibt sofort eine `runId` und eine `childSessionKey` zurück. Native Unteragentendurchläufe erhalten die delegierte Aufgabe in der ersten sichtbaren `[Subagent Task]`-Nachricht der untergeordneten Sitzung, während der System-Prompt nur Laufzeitregeln für Unteragenten und Routingkontext enthält.

Wichtige Optionen:

- `runtime: "subagent"` (Standard) oder `"acp"` für externe Harness-Agenten.
- Überschreibungen für `model` und `thinking` der untergeordneten Sitzung.
- `thread: true`, um die Erstellung an einen Chat-Thread (Discord, Slack usw.) zu binden.
- `sandbox: "require"`, um Sandboxing für die untergeordnete Sitzung durchzusetzen.
- `context: "fork"` für native Unteragenten, wenn die untergeordnete Sitzung das aktuelle Transkript des Anfragenden benötigt; lassen Sie die Option weg oder verwenden Sie `context: "isolated"` für eine bereinigte untergeordnete Sitzung. `context: "fork"` ist nur mit `runtime: "subagent"` gültig. Threadgebundene native Unteragenten verwenden standardmäßig `context: "fork"`, sofern `threadBindings.defaultSpawnContext` nichts anderes angibt.
- `visible: true`, um statt einer verborgenen Unteragentensitzung eine persistente Dashboard-Sitzung zu erstellen. Sichtbare Erstellungen unterstützen ein explizites Modell, Arbeitsverzeichnis, einen Transkript-Fork desselben Agenten und einen optionalen [verwalteten Arbeitsbaum](/de/concepts/managed-worktrees). Die genauen Kompatibilitätsgrenzen finden Sie unter [Unteragenten](/de/tools/subagents#tool-parameters).

Standardmäßige Unteragenten auf Blattebene erhalten keine Sitzungswerkzeuge. Wenn `maxSpawnDepth >= 2`, erhalten Orchestrator-Unteragenten der Tiefe 1 zusätzlich `sessions_spawn`, `subagents`, `sessions_list` und `sessions_history`, damit sie ihre eigenen untergeordneten Agenten verwalten können. Durchläufe auf Blattebene erhalten weiterhin keine Werkzeuge für rekursive Orchestrierung.

Nach Abschluss veröffentlicht ein Ankündigungsschritt das Ergebnis im Kanal des Anfragenden. Die Abschlusszustellung behält nach Möglichkeit das gebundene Thread-/Themen-Routing bei. Wenn der Abschlussursprung nur einen Kanal identifiziert, kann OpenClaw weiterhin die gespeicherte Route der Sitzung des Anfragenden (`lastChannel` / `lastTo`) für die direkte Zustellung wiederverwenden.

ACP-spezifisches Verhalten ist unter [ACP-Agenten](/de/tools/acp-agents) beschrieben.

## Sichtbarkeit

Sitzungswerkzeuge sind im Umfang beschränkt, um zu begrenzen, was der Agent sehen kann:

| Ebene   | Umfang                                                      |
| ------- | ---------------------------------------------------------- |
| `self`  | Nur die aktuelle Sitzung                                   |
| `tree`  | Aktuell + erstellt; Lesezugriffe umfassen überwachte Gruppen desselben Agenten |
| `agent` | Alle Sitzungen dieses Agenten                                |
| `all`   | Alle Sitzungen (agentenübergreifend, sofern konfiguriert)                   |

Standard ist `tree`. Sitzungen in der Sandbox sind unabhängig von der Konfiguration auf `tree` begrenzt.
Mit der Standardeinstellung `session.dmScope: "main"` macht Gruppenaktivität überwachte
Gruppensitzungen desselben Agenten aus der Hauptsitzung lesbar.

## Weiterführende Informationen

- [Sitzungsverwaltung](/de/concepts/session): Routing, Lebenszyklus, Wartung
- [Unteragenten](/de/tools/subagents): Lebenszyklus und Zustellung untergeordneter Sitzungen
- [ACP-Agenten](/de/tools/acp-agents): Erstellen externer Harness-Agenten
- [Multi-Agent](/de/concepts/multi-agent): Multi-Agent-Architektur
- [Gateway-Konfiguration](/de/gateway/configuration): Konfigurationsoptionen für Sitzungswerkzeuge

## Verwandte Themen

- [Sitzungsverwaltung](/de/concepts/session)
- [Bereinigung von Sitzungen](/de/concepts/session-pruning)
