---
read_when:
    - Bearbeiten des System-Prompt-Texts, der Werkzeugliste oder der Zeit-/Heartbeat-Abschnitte
    - Verhalten beim Workspace-Bootstrap oder bei der Skills-Injektion ändern
summary: Was der OpenClaw-System-Prompt enthält und wie er zusammengestellt wird
title: System-Prompt
x-i18n:
    generated_at: "2026-07-26T17:46:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 669fbc6f21a82a2c3c067d2ff3a6365acb3316460a85f2db165b7ad49ce79f70
    source_path: concepts/system-prompt.md
    workflow: 16
---

OpenClaw erstellt für jeden Agentenlauf einen eigenen System-Prompt; es gibt keinen standardmäßigen Laufzeit-Prompt.

Die Zusammenstellung besteht aus drei Ebenen:

- `buildAgentSystemPrompt` rendert den Prompt aus expliziten Eingaben. Die Komponente bleibt ein reiner Renderer und liest die globale Konfiguration nicht direkt.
- `resolveAgentSystemPromptConfig` löst konfigurationsgestützte Prompt-Parameter (Anzeige des Eigentümers, TTS-Hinweise, Modellaliase, Zitiermodus für den Speicher, Delegierungsmodus für Sub-Agenten) für einen bestimmten Agenten auf.
- Laufzeitadapter (eingebettet, CLI, Befehls-/Exportvorschauen, Compaction) erfassen aktuelle Fakten (Tools, Sandbox-Status, Kanalfunktionen, Kontextdateien, Prompt-Beiträge des Providers) und rufen die konfigurierte Prompt-Fassade auf.

Dadurch bleiben exportierte und Debug-Prompt-Oberflächen mit tatsächlichen Läufen konsistent, ohne jedes Laufzeitdetail in einen einzigen monolithischen Builder zu integrieren.

Provider-Plugins können cachebezogene Anweisungen beitragen, ohne den OpenClaw-eigenen Prompt zu ersetzen. Eine Provider-Laufzeit kann:

- einen von drei benannten Kernabschnitten ersetzen: `interaction_style`, `tool_call_style`, `execution_bias`
- ein **stabiles Präfix** oberhalb der Prompt-Cache-Grenze einfügen
- ein **dynamisches Suffix** unterhalb der Prompt-Cache-Grenze einfügen

Verwenden Sie Provider-eigene Beiträge für modellspezifische Optimierungen. Behalten Sie den veralteten Hook `before_prompt_build` der Kompatibilität oder wirklich globalen Prompt-Änderungen vor.

Das gebündelte Overlay der OpenAI/Codex-GPT-5-Familie (`resolveGpt5SystemPromptContribution`) verwendet diesen Mechanismus: einen Verhaltensvertrag `stablePrefix` (Ausführungsrichtlinie, Tool-Disziplin, Ausgabevertrag, Abschlussvertrag) sowie eine optionale Überschreibung `interaction_style` für einen freundlicheren Ton. Es gilt für jede über die OpenAI- oder Codex-Plugins weitergeleitete Modell-ID `gpt-5*` und wird durch `agents.defaults.promptOverlays.gpt5.personality` (`"friendly"`/`"on"` oder `"off"`) gesteuert.

## Struktur

Der Prompt ist kompakt und enthält feste Abschnitte:

- **Tools**: Hinweis auf strukturierte Tools als maßgebliche Quelle sowie Anweisungen zur Tool-Nutzung während der Laufzeit. Wenn das experimentelle Tool `update_plan` aktiviert ist (`tools.experimental.planTool`), ergänzt seine eigene Tool-Beschreibung: Verwenden Sie es nur für nicht triviale, mehrstufige Aufgaben, halten Sie höchstens einen Schritt `in_progress` und überspringen Sie es bei einfachen, einstufigen Aufgaben.
- **Ausführungspriorität**: auf umsetzbare Anfragen innerhalb des aktuellen Durchlaufs reagieren, bis zum Abschluss oder einer Blockierung fortfahren, schwache Tool-Ergebnisse ausgleichen, veränderlichen Status aktuell prüfen und vor dem Abschluss verifizieren.
- **Sicherheit**: kurzer Leitplankenhinweis gegen Machtstreben oder das Umgehen von Aufsicht.
- **Skills** (sofern verfügbar): erläutert dem Modell, wie es Skill-Anweisungen bei Bedarf lädt.
- **OpenClaw-Steuerung**: für Konfigurations-/Neustartaufgaben das Tool `gateway` bevorzugen; keine CLI-Befehle erfinden.
- **OpenClaw-Selbstaktualisierung**: Konfiguration mit `config.schema.lookup` sicher prüfen, mit `config.patch` ändern, die vollständige Konfiguration mit `config.apply` ersetzen und `update.run` nur auf ausdrückliche Benutzeranforderung ausführen. Das für Agenten verfügbare Tool `gateway` weigert sich, `tools.exec.mode` neu zu schreiben.
- **Arbeitsbereich**: Arbeitsverzeichnis (`agents.defaults.workspace`).
- **Dokumentation**: lokaler Pfad zu Dokumentation/Quellcode und Angaben dazu, wann diese gelesen werden sollen.
- **Arbeitsbereichsdateien (eingefügt)**: weist darauf hin, dass Bootstrap-Dateien nachfolgend enthalten sind.
- **Sandbox** (wenn aktiviert): Sandbox-Laufzeit, Sandbox-Pfade, Verfügbarkeit erhöhter Ausführungsrechte.
- **Aktuelles Datum und aktuelle Uhrzeit**: nur die Zeitzone (cache-stabil; die aktuelle Uhrzeit stammt aus `session_status`).
- **Anweisungen zur Assistentenausgabe**: kompakte Syntax für Anhänge, Sprachnachrichten und Antwort-Tags.
- **Heartbeats**: Heartbeat-Prompt und Bestätigungsverhalten, wenn Heartbeats für den Standardagenten aktiviert sind.
- **Laufzeit**: Host, Betriebssystem, Node, Modell, Repository-Stammverzeichnis (wenn erkannt), Denkstufe (eine Zeile).
- **Schlussfolgerung**: aktuelle Sichtbarkeitsstufe sowie Hinweis zum Umschalter `/reasoning`.

Umfangreiche stabile Inhalte (einschließlich **Projektkontext**) bleiben oberhalb der internen Prompt-Cache-Grenze. Veränderliche Abschnitte pro Durchlauf (Einbettungsanweisungen für die Steuerungsoberfläche, **Nachrichtenübermittlung**, **Sprache**, **Gruppenchatkontext**, **Reaktionen**, **Heartbeats**, **Laufzeit**) werden unterhalb dieser Grenze angefügt, damit lokale Backends mit Präfix-Caches das stabile Arbeitsbereichspräfix über Kanaldurchläufe hinweg wiederverwenden können. Tool-Beschreibungen sollten keine aktuellen Kanalnamen einbetten, wenn das akzeptierte Schema dieses Laufzeitdetail bereits enthält.

Die Tool-Anweisungen enthalten außerdem Hinweise für lang laufende Arbeiten:

- Cron für zukünftige Folgeaktionen (`check back later`, Erinnerungen, wiederkehrende Arbeiten) anstelle von `exec`-Warteschleifen, `yieldMs`-Verzögerungstricks oder wiederholtem `process`-Polling verwenden
- `exec` / `process` nur für Befehle verwenden, die jetzt starten und im Hintergrund weiterlaufen
- wenn das automatische Aufwecken bei Abschluss aktiviert ist, den Befehl einmal starten und sich auf den Push-basierten Aufweckpfad verlassen
- `process` für Protokolle, Status, Eingaben oder Eingriffe bei einem laufenden Befehl verwenden
- bei größeren Aufgaben `sessions_spawn` bevorzugen; der Abschluss eines Sub-Agenten erfolgt Push-basiert und wird dem Anfordernden automatisch mitgeteilt
- `subagents list` / `sessions_list` nicht in einer Schleife abfragen, nur um auf den Abschluss zu warten

`agents.defaults.subagents.delegationMode` (Standard `"suggest"`) kann dies verstärken. `"prefer"` fügt einen eigenen Abschnitt **Delegierung an Sub-Agenten** hinzu, der den Hauptagenten anweist, als reaktionsfähiger Koordinator zu handeln und alles, was über eine direkte Antwort hinausgeht, über `sessions_spawn` weiterzuleiten. Dies betrifft nur den Prompt; die Tool-Richtlinie steuert weiterhin, ob `sessions_spawn` verfügbar ist.

Die Sicherheitsleitplanken im System-Prompt sind Empfehlungen, keine Durchsetzungsmechanismen. Verwenden Sie Tool-Richtlinien, Ausführungsgenehmigungen, Sandboxing und Kanal-Zulassungslisten zur strikten Durchsetzung; Betreiber können Prompt-Leitplanken bewusst deaktivieren.

Auf Kanälen mit nativen Genehmigungskarten/-schaltflächen weist der Prompt den Agenten an, sich zuerst auf diese Benutzeroberfläche zu verlassen und nur dann einen manuellen Befehl `/approve` anzugeben, wenn das Tool-Ergebnis besagt, dass Chat-Genehmigungen nicht verfügbar sind oder die manuelle Genehmigung der einzige mögliche Weg ist.

## Prompt-Modi

OpenClaw rendert kleinere System-Prompts für Sub-Agenten. Die Laufzeit legt pro Lauf einen `promptMode` fest (keine benutzerseitige Konfiguration):

- `full` (Standard): alle oben genannten Abschnitte.
- `minimal`: wird für Sub-Agenten verwendet; lässt den Speicher-Prompt-Abschnitt (gebündelt als **Speicherabruf**), **OpenClaw-Selbstaktualisierung**, **Modellaliase**, **Benutzeridentität**, **Anweisungen zur Assistentenausgabe**, **Nachrichtenübermittlung**, **Stille Antworten** und **Heartbeats** aus. Tools, **Sicherheit**, **Skills** (wenn bereitgestellt), Arbeitsbereich, Sandbox, aktuelles Datum und aktuelle Uhrzeit (wenn bekannt), Laufzeit und eingefügter Kontext bleiben verfügbar.
- `none`: gibt nur die grundlegende Identitätszeile zurück.

Unter `promptMode=minimal` werden zusätzlich eingefügte Prompts als **Sub-Agenten-Kontext** statt als **Gruppenchatkontext** gekennzeichnet.

Bei automatischen Kanalantwortläufen lässt OpenClaw den allgemeinen Abschnitt **Stille Antworten** aus, wenn der Direkt-, Gruppen- oder ausschließliche Nachrichten-Tool-Kontext bereits den Vertrag für sichtbare Antworten definiert. Nur der veraltete automatische Gruppen-/Kanalmodus zeigt `NO_REPLY`; Direktchats und ausschließliche Nachrichten-Tool-Antworten überspringen Anweisungen für stille Token.

## Prompt-Snapshots

OpenClaw verwaltet festgeschriebene Prompt-Snapshots für den erfolgreichen Standardpfad der Codex-Laufzeit unter `test/fixtures/agents/prompt-snapshots/codex-runtime-happy-path/`. Sie rendern ausgewählte Thread-/Durchlaufparameter des App-Servers sowie einen rekonstruierten, modellgebundenen Prompt-Ebenenstapel für Telegram-Direkt-, Discord-Gruppen- und Heartbeat-Durchläufe: eine angeheftete Codex-Modell-Prompt-Fixture `gpt-5.5`, den Entwicklertext für Berechtigungen im erfolgreichen Codex-Standardpfad, OpenClaw-Entwickleranweisungen, durchlaufbezogene Anweisungen zum Zusammenarbeitsmodus, sofern OpenClaw diese bereitstellt, die Benutzereingabe des Durchlaufs sowie Verweise auf dynamische Tool-Spezifikationen.

Aktualisieren Sie die angeheftete Codex-Modell-Prompt-Fixture mit `pnpm prompt:snapshots:sync-codex-model`. Standardmäßig wird zuerst nach `$CODEX_HOME/models_cache.json`, dann nach `~/.codex/models_cache.json` und anschließend nach der Maintainer-Checkout-Konvention `~/code/codex/codex-rs/models-manager/models.json` gesucht; wenn nichts davon vorhanden ist, wird der Vorgang beendet, ohne die festgeschriebene Fixture zu ändern. Übergeben Sie `--catalog <path>`, um sie aus einer bestimmten Datei `models_cache.json` oder `models.json` zu aktualisieren.

Diese Snapshots sind keine Byte-für-Byte-Aufzeichnung der ursprünglichen OpenAI-Anfrage. Codex kann laufzeiteigenen Arbeitsbereichskontext (`AGENTS.md`, Umgebungskontext, Erinnerungen, App-/Plugin-Anweisungen, integrierte Anweisungen des Standard-Zusammenarbeitsmodus) hinzufügen, nachdem OpenClaw Thread- und Durchlaufparameter gesendet hat.

Generieren Sie sie mit `pnpm prompt:snapshots:gen` neu; prüfen Sie Abweichungen mit `pnpm prompt:snapshots:check`. Die CI führt die Abweichungsprüfung zusammen mit den Shards für zusätzliche Grenzen aus, sodass Prompt-Änderungen und Snapshot-Aktualisierungen im selben PR eingehen.

## Einfügung des Arbeitsbereich-Bootstraps

Bootstrap-Dateien werden aus dem aktiven Arbeitsbereich aufgelöst und an die ihrer Lebensdauer entsprechende Prompt-Oberfläche weitergeleitet:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (nur in brandneuen Arbeitsbereichen)
- `MEMORY.md`, sofern vorhanden

Im nativen Codex-Harness vermeidet OpenClaw, stabile Arbeitsbereichsdateien bei jedem Benutzerdurchlauf zu wiederholen. Codex lädt `AGENTS.md` über seine eigene Projektdokumenterkennung. `TOOLS.md` wird als geerbte Codex-Entwickleranweisung weitergeleitet. `SOUL.md`, `IDENTITY.md` und `USER.md` werden als durchlaufbezogene Entwickleranweisungen zum Zusammenarbeitsmodus weitergeleitet, damit native Codex-Sub-Agenten sie nicht erben. Der Inhalt von `HEARTBEAT.md` wird nicht direkt eingefügt; Heartbeat-Durchläufe erhalten einen Hinweis zum Zusammenarbeitsmodus, der auf die Datei verweist, wenn sie vorhanden und nicht leer ist. Auch der Inhalt von `MEMORY.md` wird nicht in jeden nativen Codex-Durchlauf eingefügt: Wenn Speicher-Tools für den Arbeitsbereich verfügbar sind, erhalten Codex-Durchläufe einen kurzen Hinweis zum Arbeitsbereichsspeicher, der das Modell zu `memory_search` oder `memory_get` verweist. Wenn Tools deaktiviert sind, die Speichersuche nicht verfügbar ist oder sich der aktive Arbeitsbereich vom Agentenspeicher-Arbeitsbereich unterscheidet, fällt `MEMORY.md` auf den normalen begrenzten Durchlaufkontextpfad zurück. `BOOTSTRAP.md` behält die normale Durchlaufkontextrolle bei.

In anderen als Codex-Harnesses werden Bootstrap-Dateien entsprechend ihren bestehenden Bedingungen in den OpenClaw-Prompt integriert. `HEARTBEAT.md` wird bei normalen Läufen ausgelassen, wenn Heartbeats für den Standardagenten deaktiviert sind oder `agents.defaults.heartbeat.includeSystemPromptSection` auf „false“ gesetzt ist. Halten Sie eingefügte Dateien kurz, insbesondere `MEMORY.md` außerhalb von Codex: Sie sollte eine kuratierte langfristige Zusammenfassung bleiben, während detaillierte tägliche Notizen in `memory/*.md` bei Bedarf über `memory_search` / `memory_get` abgerufen werden können. Übermäßig große `MEMORY.md`-Dateien außerhalb von Codex erhöhen die Prompt-Nutzung und können gemäß den nachfolgenden Größenbeschränkungen für Bootstrap-Dateien nur teilweise eingefügt werden.

<Note>
Tägliche `memory/*.md`-Dateien sind **nicht** Teil des normalen Bootstrap-Projektkontexts. Bei gewöhnlichen Durchläufen wird bei Bedarf über `memory_search` / `memory_get` auf sie zugegriffen, sodass sie nicht auf das Kontextfenster angerechnet werden, sofern das Modell sie nicht ausdrücklich liest. Unveränderte `/new`- und `/reset`-Durchläufe bilden die Ausnahme: Die Laufzeit kann aktuelle tägliche Speicherinhalte als einmaligen Startkontextblock vor diesen ersten Durchlauf setzen.
</Note>

Große Dateien werden mit einer Markierung abgeschnitten:

| Begrenzung                                    | Konfigurationsschlüssel                              | Standard |
| --------------------------------------------- | ---------------------------------------------------- | -------- |
| Maximale Zeichenzahl pro Datei                | `agents.defaults.bootstrapMaxChars`                                   | 20000    |
| Gesamtzahl über alle Dateien                  | `agents.defaults.bootstrapTotalMaxChars`                                   | 60000    |
| Kürzungswarnung (`off`\|`once`\|`always`) | `agents.defaults.bootstrapPromptTruncationWarning` | `always` |

Fehlende Dateien fügen eine kurze Markierung für fehlende Dateien ein. Detaillierte Roh-/Einfügungszähler verbleiben in Diagnosen wie `/context`, `/status`, doctor und Protokollen.

Bei Speicherdateien bedeutet Kürzung keinen Datenverlust: Die Datei bleibt auf dem Datenträger intakt. In nativem Codex wird `MEMORY.md` bei Bedarf über Speicherwerkzeuge gelesen, sofern verfügbar; andernfalls wird eine begrenzte Prompt-Ausweichlösung verwendet. In anderen Harnesses sieht das Modell nur die gekürzte eingefügte Kopie, bis es den Speicher direkt liest oder durchsucht. Wenn `MEMORY.md` wiederholt gekürzt wird, verdichten Sie die Datei zu einer kürzeren dauerhaften Zusammenfassung, verschieben Sie den detaillierten Verlauf nach `memory/*.md` oder erhöhen Sie bewusst die Bootstrap-Grenzwerte.

Sub-Agent-Sitzungen fügen nur `AGENTS.md` und `TOOLS.md` ein (andere Bootstrap-Dateien werden herausgefiltert, um den Sub-Agent-Kontext klein zu halten).

Interne Hooks können diesen Schritt über das Ereignis `agent:bootstrap` abfangen, um die eingefügten Bootstrap-Dateien zu verändern oder zu ersetzen (beispielsweise um `SOUL.md` gegen eine alternative Persona auszutauschen).

Um weniger generisch zu klingen, beginnen Sie mit dem [SOUL.md-Persönlichkeitsleitfaden](/de/concepts/soul).

Um zu prüfen, wie viel jede eingefügte Datei beiträgt (roh gegenüber eingefügt, Kürzung, Tool-Schema-Overhead), verwenden Sie `/context list` oder `/context detail`. Siehe [Kontext](/de/concepts/context).

## Zeitverarbeitung

Der Abschnitt **Aktuelles Datum und aktuelle Uhrzeit** erscheint nur, wenn die Zeitzone des Benutzers bekannt ist, und enthält ausschließlich die **Zeitzone** (keine dynamische Uhr oder Zeitformatangabe), damit der Prompt cache-stabil bleibt.

Verwenden Sie `session_status`, wenn der Agent die aktuelle Uhrzeit benötigt; die zugehörige Statuskarte enthält eine Zeitstempelzeile. Dasselbe Tool kann optional eine sitzungsspezifische Modellüberschreibung festlegen (`model=default` entfernt sie).

Konfiguration:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Ausführliche Verhaltensdetails finden Sie unter [Zeitzonen](/de/concepts/timezone) und [Datum und Uhrzeit](/de/date-time).

## Skills

Wenn geeignete Skills vorhanden sind, fügt OpenClaw eine kompakte `<available_skills>`-Liste (`formatSkillsForPrompt`) mit dem **Dateipfad** und einer aus dem Inhalt abgeleiteten `<version>sha256:...</version>`-Markierung pro Skill ein. Der Prompt weist das Modell an, mit `read` die SKILL.md am aufgeführten Speicherort (Workspace, verwaltet oder gebündelt) zu laden und einen Skill erneut zu lesen, wenn sich dessen `<version>` gegenüber einem vorherigen Durchlauf unterscheidet. Wenn keine Skills geeignet sind, wird der Abschnitt „Skills“ weggelassen.

Native Codex-Durchläufe erhalten diese Liste als durchlaufspezifische Entwickleranweisungen für die Zusammenarbeit anstelle einer Benutzereingabe pro Durchlauf; ausgenommen sind schlanke Cron-Durchläufe, die den exakten geplanten Prompt beibehalten. Andere Harnesses behalten den normalen Prompt-Abschnitt bei.

Der Speicherort kann auf einen verschachtelten Skill verweisen, etwa `skills/personal/foo/SKILL.md`. Die Verschachtelung dient nur der Organisation; der Prompt verwendet den flachen Skill-Namen aus dem `SKILL.md`-Frontmatter.

Die Eignung berücksichtigt Metadaten-Gates des Skills, Prüfungen der Laufzeitumgebung und -konfiguration sowie die effektive Skill-Zulassungsliste des Agenten, wenn `agents.defaults.skills` oder `agents.entries.*.skills` konfiguriert ist. In Plugins gebündelte Skills sind nur geeignet, wenn das zugehörige Plugin aktiviert ist. Dadurch können Tool-Plugins ausführlichere Bedienungsleitfäden bereitstellen, ohne all diese Anleitungen in jede Tool-Beschreibung einzubetten.

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
    <version>sha256:...</version>
  </skill>
</available_skills>
```

Dadurch bleibt der Basis-Prompt klein, während weiterhin eine gezielte Nutzung von Skills möglich ist. Die Größenbegrenzung liegt in der Verantwortung des Skills-Subsystems und ist von der generischen Größenbegrenzung für Laufzeit-Lesevorgänge und -Einfügungen getrennt:

| Geltungsbereich | Budget für den Skills-Prompt                         | Budget für Laufzeitauszüge          |
| --------------- | ---------------------------------------------------- | ----------------------------------- |
| Global          | `skills.limits.maxSkillsPromptChars`                                   | `agents.defaults.contextLimits.*`                  |
| Pro Agent       | `agents.entries.*.skillsLimits.maxSkillsPromptChars`                                   | `agents.entries.*.contextLimits.*`                  |

Das Budget für Laufzeitauszüge umfasst `memory_get`, Live-Tool-Ergebnisse und Aktualisierungen von `AGENTS.md` nach der Compaction.

## Dokumentation

Der Abschnitt **Dokumentation** verweist auf lokale Dokumentation, sofern verfügbar (`docs/` in einem Git-Checkout oder die Dokumentation des gebündelten npm-Pakets), und verwendet andernfalls [https://docs.openclaw.ai](https://docs.openclaw.ai). Außerdem führt er den Speicherort des OpenClaw-Quellcodes auf: Git-Checkouts stellen den lokalen Quellstamm bereit, während Paketinstallationen die GitHub-Quell-URL mit der Anweisung erhalten, dort den Quellcode zu prüfen, wenn die Dokumentation unvollständig oder veraltet ist.

Der Prompt stellt die Dokumentation als maßgebliche Quelle für OpenClaw-Selbstwissen dar, bevor das Modell versteht, wie OpenClaw funktioniert (Speicher/Tagesnotizen, Sitzungen, Tools, Gateway, Konfiguration, Befehle, Projektkontext), und weist das Modell an, `AGENTS.md`, den Projektkontext, Workspace-/Profil-/Speichernotizen und `memory_search` als Anweisungskontext oder Benutzerspeicher zu behandeln und nicht als Wissen über Entwurf oder Implementierung von OpenClaw. Wenn die Dokumentation keine Angaben enthält oder veraltet ist, soll das Modell darauf hinweisen und den Quellcode prüfen. Der Prompt weist das Modell außerdem an, `openclaw status` nach Möglichkeit selbst auszuführen und den Benutzer nur zu fragen, wenn es keinen Zugriff hat.

Speziell für die Konfiguration verweist er Agenten zunächst auf die Tool-Aktion `config.schema.lookup` des Tools `gateway`, um exakte Dokumentation und Einschränkungen auf Feldebene zu erhalten, und anschließend auf `docs/gateway/configuration.md` und `docs/gateway/configuration-reference.md` für allgemeinere Anleitungen.

## Verwandte Themen

- [Agent-Laufzeit](/de/concepts/agent)
- [Agent-Workspace](/de/concepts/agent-workspace)
- [Kontext-Engine](/de/concepts/context-engine)
