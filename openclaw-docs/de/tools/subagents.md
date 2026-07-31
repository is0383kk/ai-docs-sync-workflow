---
read_when:
    - Sie möchten Hintergrund- oder parallele Arbeit über den Agenten ausführen
    - Sie ändern die Richtlinie für `sessions_spawn` oder das Subagenten-Tool.
    - Sie implementieren oder beheben Probleme bei Thread-gebundenen Subagent-Sitzungen
sidebarTitle: Sub-agents
summary: Starten Sie isolierte Agent-Läufe im Hintergrund, die ihre Ergebnisse im Chat des Anfragenden bekannt geben.
title: Unteragenten
x-i18n:
    generated_at: "2026-07-26T18:51:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e45b32fdb177c52ed785287712b9b6c2c30bbe392f0ce975970910ff91ed30ed
    source_path: tools/subagents.md
    workflow: 16
---

Sub-Agenten sind im Hintergrund ausgeführte Agentenläufe, die aus einem bestehenden Agentenlauf gestartet werden.
Jeder wird in einer eigenen Sitzung (`agent:<agentId>:subagent:<uuid>`) ausgeführt und
gibt nach Abschluss sein Ergebnis im anfragenden Chatkanal **bekannt**.
Jeder Sub-Agentenlauf wird als [Hintergrundaufgabe](/de/automation/tasks) verfolgt.

Ziele:

- Recherche, langwierige Aufgaben und langsame Tool-Arbeit parallelisieren, ohne den Hauptlauf zu blockieren.
- Sub-Agenten standardmäßig isoliert halten (Sitzungstrennung, optionales Sandboxing).
- Die Tool-Oberfläche vor Fehlbedienung schützen: Sub-Agenten erhalten standardmäßig **keine** Sitzungs- oder Nachrichten-Tools.
- Konfigurierbare Verschachtelungstiefe für Orchestrator-Muster unterstützen.

<Note>
**Kostenhinweis:** Jeder Sub-Agent verfügt standardmäßig über einen eigenen
Kontext und eine eigene Token-Nutzung. Legen Sie für aufwendige oder sich
wiederholende Aufgaben ein günstigeres Modell für Sub-Agenten fest und
verwenden Sie für Ihren Haupt-Agenten über `agents.defaults.subagents.model` oder
agentenspezifische Überschreibungen ein hochwertigeres Modell. Wenn ein
untergeordneter Agent tatsächlich das aktuelle Transkript des Anfragenden
benötigt, starten Sie ihn mit `context: "fork"`. Threadgebundene
Sub-Agenten-Sitzungen verwenden standardmäßig `context: "fork"`, da sie
die aktuelle Unterhaltung in einen Folge-Thread verzweigen.
</Note>

## Slash-Befehl

`/subagents` prüft Sub-Agentenläufe für die **aktuelle Sitzung**:

```text
/subagents list
/subagents log <id|#> [limit] [tools]
/subagents info <id|#>
```

`/subagents info` zeigt Laufmetadaten (Status, Zeitstempel, Sitzungs-ID,
Transkriptpfad, Bereinigung). `/subagents log` gibt die letzten Chatbeiträge
eines Laufs aus; fügen Sie das Token `tools` hinzu, um Nachrichten
zu Tool-Aufrufen und deren Ergebnissen einzubeziehen (standardmäßig
ausgelassen). Verwenden Sie `sessions_history` für eine begrenzte,
sicherheitsgefilterte Rückschau innerhalb eines Agentenbeitrags oder prüfen
Sie den Transkriptpfad auf dem Datenträger, um das vollständige Rohtranskript
anzuzeigen.

In der Control UI verfügen übergeordnete Sitzungen mit kürzlich ausgeführten
untergeordneten Läufen über eine aufklappbare Seitenleistenzeile. Die
verschachtelten Zeilen zeigen Status und Laufzeit des untergeordneten Laufs;
durch Auswählen wird dessen Chat geöffnet, während die übergeordnete
Hierarchie erhalten bleibt.

### Steuerelemente für die Threadbindung

Diese Befehle funktionieren in Kanälen mit dauerhaften Threadbindungen. Siehe
unten [Kanäle mit Threadunterstützung](#thread-supporting-channels).

```text
/focus <subagent-label|session-key|session-id|session-label>
/unfocus
/agents
/session idle <duration|off>
/session max-age <duration|off>
```

### Startverhalten

Agenten starten Sub-Agenten im Hintergrund mit dem Tool `sessions_spawn`.
Abschlüsse werden als interne Ereignisse der übergeordneten Sitzung
zurückgegeben; der übergeordnete/anfragende Agent entscheidet, ob eine
benutzersichtbare Aktualisierung erforderlich ist.

<AccordionGroup>
  <Accordion title="Nicht blockierender, Push-basierter Abschluss">
    - `sessions_spawn` blockiert nicht; es gibt sofort eine Lauf-ID zurück.
    - Nach Abschluss meldet sich der Sub-Agent bei der übergeordneten/anfragenden Sitzung zurück.
    - Agentenbeiträge, die Ergebnisse untergeordneter Agenten benötigen, sollten nach dem Starten der erforderlichen Arbeit `sessions_yield` aufrufen. Dadurch wird der aktuelle Beitrag beendet und das Abschlussereignis kann als nächste für das Modell sichtbare Nachricht eintreffen.
    - Der Abschluss erfolgt Push-basiert. Fragen Sie nach dem Start **nicht** `/subagents list`, `sessions_list` oder `sessions_history` in einer Schleife ab, nur um auf den Abschluss zu warten; prüfen Sie den Status nur bei Bedarf zur Fehlerdiagnose.
    - Die Ausgabe des untergeordneten Agenten ist ein Bericht/Nachweis, den der anfragende Agent zusammenführen soll. Sie ist kein vom Benutzer verfasster Anweisungstext und kann System-, Entwickler- oder Benutzerrichtlinien nicht außer Kraft setzen.
    - Nach Abschluss schließt OpenClaw nach Möglichkeit die von dieser Sub-Agenten-Sitzung geöffneten und verfolgten Browser-Tabs/-Prozesse, bevor der Bekanntgabe-Bereinigungsablauf fortgesetzt wird.

  </Accordion>
  <Accordion title="Zustellung des Abschlusses">
    - OpenClaw übergibt Abschlüsse über einen `agent`-Beitrag mit einem stabilen Idempotenzschlüssel an die anfragende Sitzung zurück.
    - Wenn der anfragende Lauf noch aktiv ist, versucht OpenClaw zunächst, diesen Lauf aufzuwecken/zu steuern, anstatt einen zweiten sichtbaren Antwortpfad zu starten.
    - Wenn ein aktiver Anfragender nicht aufgeweckt werden kann, greift OpenClaw auf eine Übergabe an den anfragenden Agenten mit demselben Abschlusskontext zurück, anstatt die Bekanntgabe zu verwerfen.
    - Eine erfolgreiche Übergabe an den übergeordneten Agenten schließt die Zustellung des Sub-Agenten ab, selbst wenn der übergeordnete Agent entscheidet, dass keine sichtbare Benutzeraktualisierung erforderlich ist.
    - Native Sub-Agenten erhalten das Nachrichten-Tool nicht. Sie geben einfachen Assistententext an den übergeordneten/anfragenden Agenten zurück; für Menschen sichtbare Antworten unterliegen weiterhin der normalen Zustellrichtlinie des übergeordneten/anfragenden Agenten.
    - Wenn eine direkte Übergabe nicht verwendet werden kann, greift die Zustellung zunächst auf Warteschlangenrouting und anschließend auf eine kurze Wiederholung der Bekanntgabe mit exponentiellem Backoff zurück, bevor sie endgültig aufgegeben wird.
    - Die Zustellung behält die aufgelöste Route des Anfragenden bei: Thread- oder unterhaltungsgebundene Abschlussrouten haben Vorrang, sofern verfügbar. Wenn der Ursprung des Abschlusses nur einen Kanal bereitstellt, ergänzt OpenClaw das fehlende Ziel/Konto anhand der aufgelösten Route der anfragenden Sitzung (`lastChannel` / `lastTo` / `lastAccountId`), damit die direkte Zustellung weiterhin funktioniert.

  </Accordion>
  <Accordion title="Metadaten der Abschlussübergabe">
    Die Abschlussübergabe an die anfragende Sitzung ist ein zur Laufzeit
    generierter interner Kontext (kein vom Benutzer verfasster Text) und
    enthält:

    - `Result` — den neuesten sichtbaren `assistant`-Antworttext des untergeordneten Agenten. Ausgaben von Tool/ToolResult werden nicht in Ergebnisse des untergeordneten Agenten übernommen. Endgültig fehlgeschlagene Läufe verwenden erfassten Antworttext nicht erneut.
    - `Status` — `completed; ready for parent review` / `failed` / `timed out` / `unknown`.
    - Kompakte Laufzeit-/Token-Statistiken.
    - Eine Prüfanweisung, die den anfragenden Agenten auffordert, das Ergebnis zu verifizieren, bevor er entscheidet, ob die ursprüngliche Aufgabe abgeschlossen ist.
    - Hinweise zum weiteren Vorgehen, die den anfragenden Agenten auffordern, die Aufgabe fortzusetzen oder eine Folgeaufgabe zu erfassen, wenn das Ergebnis des untergeordneten Agenten weitere Maßnahmen erfordert.
    - Eine Anweisung zur abschließenden Aktualisierung für den Fall, dass keine weiteren Maßnahmen erforderlich sind, formuliert in normaler Assistentensprache, ohne interne Rohmetadaten weiterzugeben.

  </Accordion>
  <Accordion title="Modi und ACP-Laufzeit">
    - `--model` und `--thinking` überschreiben die Standardwerte für diesen spezifischen Lauf.
    - Verwenden Sie `info`/`log`, um Details und Ausgabe nach Abschluss zu prüfen.
    - Verwenden Sie für dauerhafte threadgebundene Sitzungen `sessions_spawn` mit `thread: true` und `mode: "session"`.
    - Wenn der anfragende Kanal keine Threadbindungen unterstützt, verwenden Sie `mode: "run"`, anstatt eine unmögliche threadgebundene Kombination erneut zu versuchen.
    - Verwenden Sie für ACP-Harness-Sitzungen (Claude Code, Gemini CLI, OpenCode oder explizites Codex ACP/acpx) `sessions_spawn` mit `runtime: "acp"`, wenn das Tool diese Laufzeit ausweist. Siehe [ACP-Zustellmodell](/de/tools/acp-agents#delivery-model) zur Fehlerdiagnose bei Abschlüssen oder Agent-zu-Agent-Schleifen. Wenn das Plugin `codex` aktiviert ist, sollte die Codex-Chat-/Threadsteuerung `/codex ...` gegenüber ACP bevorzugen, sofern der Benutzer nicht ausdrücklich ACP/acpx anfordert.
    - OpenClaw blendet `runtime: "acp"` aus, bis ACP aktiviert ist, der Anfragende nicht in einer Sandbox ausgeführt wird und ein Backend-Plugin wie `acpx` geladen ist. `runtime: "acp"` erwartet eine externe ACP-Harness-ID oder einen `agents.entries.*`-Eintrag mit `runtime.type="acp"`; verwenden Sie die standardmäßige Sub-Agenten-Laufzeit für normale OpenClaw-Konfigurationsagenten aus `agents_list`.

  </Accordion>
</AccordionGroup>

## Kontextmodi

Native Sub-Agenten starten isoliert, sofern der Aufrufer nicht ausdrücklich
eine Verzweigung des aktuellen Transkripts anfordert.

| Modus       | Verwendungszweck                                                                                                                         | Verhalten                                                                          |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `isolated` | Neue Recherche, unabhängige Implementierung, langsame Tool-Arbeit oder alles, was im Aufgabentext vollständig beschrieben werden kann | Erstellt ein leeres Transkript des untergeordneten Agenten. Dies ist die Standardeinstellung und hält die Token-Nutzung geringer. |
| `fork`     | Arbeit, die von der aktuellen Unterhaltung, früheren Tool-Ergebnissen oder differenzierten Anweisungen im Transkript des Anfragenden abhängt | Verzweigt das Transkript des Anfragenden in die Sitzung des untergeordneten Agenten, bevor dieser startet. |

Verwenden Sie `fork` sparsam. Es dient zur kontextabhängigen
Delegation und ersetzt keine klare Aufgabenbeschreibung.

## Tool: `sessions_spawn`

Startet einen Sub-Agentenlauf mit `deliver: false` auf der globalen
`subagent`-Lane, führt anschließend einen Bekanntgabeschritt aus und
sendet die Bekanntgabeantwort an den anfragenden Chatkanal.

Die Verfügbarkeit hängt von der effektiven Tool-Richtlinie des Aufrufers ab.
Die integrierten Profile `coding` und `messaging` enthalten
`sessions_spawn`, `sessions_yield` und `subagents`;
`minimal` hingegen nicht. `full` erlaubt jedes Tool.
Fügen Sie diese Tools mit `tools.alsoAllow` hinzu oder verwenden Sie eines
der oben genannten Profile für einen Agenten mit einem benutzerdefinierten,
restriktiveren Profil, der dennoch Arbeit delegieren soll.
Kanal-/Gruppen-, Provider-, Sandbox- und agentenspezifische Zulassungs-/
Verweigerungsrichtlinien können das Tool nach der Profilphase weiterhin
entfernen. Verwenden Sie `/tools` aus derselben Sitzung, um die
effektive Tool-Liste zu bestätigen.

**Standardwerte:**

- **Modell:** Native Sub-Agenten übernehmen das Modell des Aufrufers, sofern Sie nicht `agents.defaults.subagents.model` (oder das agentenspezifische `agents.entries.*.subagents.model`) festlegen. Starts der ACP-Laufzeit verwenden dasselbe konfigurierte Sub-Agenten-Modell, sofern vorhanden; andernfalls behält das ACP-Harness seinen eigenen Standard bei. Ein explizites `sessions_spawn.model` hat weiterhin Vorrang.
- **Denken:** Native Sub-Agenten übernehmen die Denkeinstellung des Aufrufers, sofern Sie nicht `agents.defaults.subagents.thinking` (oder das agentenspezifische `agents.entries.*.subagents.thinking`) festlegen. Starts der ACP-Laufzeit wenden außerdem `agents.defaults.models["provider/model"].params.thinking` auf das ausgewählte Modell an. Ein explizites `sessions_spawn.thinking` hat weiterhin Vorrang.
- **Laufzeitlimit:** OpenClaw verwendet `agents.defaults.subagents.runTimeoutSeconds`, wenn es festgelegt ist; andernfalls greift es auf `0` zurück (kein Zeitlimit). `sessions_spawn` akzeptiert keine aufrufspezifischen Zeitlimitüberschreibungen.
- **Prozesslebensdauer:** Ein abgetrennter OpenClaw-Sub-Agent hat einen eigenen Lauflebenszyklus. Eine Hintergrundaufgabe, die innerhalb eines externen CLI-Backends erstellt wird, ist etwas anderes: Sie teilt sich den übergeordneten CLI-Unterprozess und wird beendet, wenn dieser übergeordnete Prozess `agents.defaults.timeoutSeconds` erreicht.
- **Aufgabenzustellung:** Native Sub-Agenten erhalten die delegierte Aufgabe in ihrer ersten sichtbaren `[Subagent Task]`-Nachricht. Der System-Prompt des Sub-Agenten enthält Laufzeitregeln und Routingkontext, nicht ein verborgenes Duplikat der Aufgabe.

Akzeptierte Starts nativer Sub-Agenten enthalten die aufgelösten
Modellmetadaten des untergeordneten Agenten im Tool-Ergebnis:
`resolvedModel` enthält die angewendete Modellreferenz und
`resolvedProvider` das Provider-Präfix, sofern die Referenz eines enthält.

### Modus des Delegations-Prompts

`agents.defaults.subagents.delegationMode` steuert nur die Prompt-Anleitung; es ändert weder die Tool-Richtlinie noch erzwingt es eine Delegation.

- `suggest` (Standard): Behält den standardmäßigen Prompt-Hinweis bei, für umfangreichere oder langsamere Arbeit Sub-Agenten zu verwenden.
- `prefer`: Weist den Haupt-Agenten an, reaktionsfähig zu bleiben und alles, was über eine direkte Antwort hinausgeht, über `sessions_spawn` zu delegieren.

Agentenspezifische Überschreibung: `agents.entries.*.subagents.delegationMode`.

```json5
{
  agents: {
    defaults: {
      subagents: {
        delegationMode: "prefer",
        maxConcurrent: 4,
      },
    },
    list: [
      {
        id: "coordinator",
        subagents: { delegationMode: "prefer" },
      },
    ],
  },
}
```

### Tool-Parameter

<ParamField path="task" type="string" required>
  Die Aufgabenbeschreibung für den Sub-Agenten.
</ParamField>
<ParamField path="taskName" type="string">
  Optionaler stabiler Bezeichner zur Identifizierung eines bestimmten untergeordneten Agenten in späteren Statusausgaben. Muss `[a-z][a-z0-9_-]{0,63}` entsprechen und darf kein reserviertes Ziel wie `last` oder `all` sein.
</ParamField>
<ParamField path="label" type="string">
  Optionale menschenlesbare Bezeichnung.
</ParamField>
<ParamField path="agentId" type="string">
  Unter einer anderen konfigurierten Agenten-ID starten, sofern durch `subagents.allowAgents` erlaubt.
</ParamField>
<ParamField path="cwd" type="string">
  Optionales Arbeitsverzeichnis für die Aufgabe des untergeordneten Laufs. Native Sub-Agenten laden Bootstrap-Dateien weiterhin aus dem Arbeitsbereich des Zielagenten; `cwd` ändert nur, wo Laufzeitwerkzeuge und CLI-Harnesses die delegierte Arbeit ausführen.
</ParamField>
<ParamField path="runtime" type='"subagent" | "acp"' default="subagent">
  `acp` ist ausschließlich für externe ACP-Harnesses (`claude`, `droid`, `gemini`, `opencode` oder ausdrücklich angefordertes Codex ACP/acpx) sowie für `agents.entries.*`-Einträge vorgesehen, deren `runtime.type` `acp` ist.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Nur ACP. Setzt eine vorhandene ACP-Harness-Sitzung fort, wenn `runtime: "acp"`; wird bei nativen Sub-Agent-Starts ignoriert.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  Nur ACP. Streamt die Ausgabe des ACP-Laufs an die übergeordnete Sitzung, wenn `runtime: "acp"`; bei nativen Sub-Agent-Starts weglassen.
</ParamField>
<ParamField path="model" type="string">
  Überschreibt das Modell des Sub-Agenten. Ungültige Werte werden übersprungen, und der Sub-Agent wird mit dem Standardmodell ausgeführt; das Werkzeugergebnis enthält eine Warnung.
</ParamField>
<ParamField path="thinking" type="string">
  Überschreibt die Denkstufe für den Sub-Agent-Lauf. Mit `visible: true` nicht verfügbar.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  Fordert bei `true` die Bindung dieser Sub-Agent-Sitzung an einen Kanal-Thread an.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  Wenn `thread: true` und `mode` weggelassen wird, wird `session` zum Standard. `mode: "session"` erfordert `thread: true`.
  Wenn für den Kanal des Anforderers keine Thread-Bindung verfügbar ist, verwenden Sie stattdessen `mode: "run"`.
  Lassen Sie bei `visible: true` `mode` weg; sichtbare Sitzungen sind persistent und unterstützen `mode: "run"` nicht.
</ParamField>
<ParamField path="cleanup" type='"delete" | "keep"' default="keep">
  `"delete"` archiviert die Sitzung unmittelbar nach der Ankündigung (das Transkript bleibt durch Umbenennen erhalten).
</ParamField>
<ParamField path="sandbox" type='"inherit" | "require"' default="inherit">
  `require` lehnt den Start ab, sofern die Laufzeit des untergeordneten Zielagenten nicht in einer Sandbox ausgeführt wird.
</ParamField>
<ParamField path="context" type='"isolated" | "fork"' default="isolated">
  `fork` verzweigt das aktuelle Transkript des Anforderers in die untergeordnete Sitzung. Nur für native Sub-Agenten. Thread-gebundene Starts verwenden standardmäßig `fork`; Starts ohne Thread verwenden standardmäßig `isolated`. Ein sichtbarer Fork muss denselben Agenten wie der Anforderer als Ziel verwenden.
</ParamField>
<ParamField path="visible" type="boolean" default="false">
  Erstellt eine persistente Dashboard-Sitzung, die in der Control UI geöffnet werden kann. Sichtbare Starts unterstützen nur `runtime: "subagent"` und behalten die erstellte Sitzung immer bei.
</ParamField>
<ParamField path="worktree" type="boolean" default="false">
  Stellt einen verwalteten Git-Worktree für die neue Dashboard-Sitzung bereit. Erfordert `visible: true`.
</ParamField>
<ParamField path="worktreeName" type="string">
  Optionaler Name des verwalteten Worktrees. Erfordert `visible: true` und `worktree: true`.
</ParamField>
<ParamField path="worktreeBaseRef" type="string">
  Optionale Git-Basisreferenz für den verwalteten Worktree. Erfordert `visible: true` und `worktree: true`.
</ParamField>

<Warning>
`sessions_spawn` akzeptiert **keine** Parameter für die Kanalauslieferung (`target`,
`channel`, `to`, `threadId`, `replyTo`, `transport`). Native Sub-Agenten melden
ihre jeweils letzte Assistentenantwort an den Anforderer zurück; die externe Auslieferung verbleibt
beim übergeordneten bzw. anfordernden Agenten.
</Warning>

Mit `visible: true` werden `model`, `cwd` und ein agentengleiches `context: "fork"` unterstützt. Ein Ziel in einer Sandbox beschränkt `cwd` auf den Arbeitsbereich dieses Agenten. Thread-Bindung, `mode`, Überschreibungen der Denkstufe, `lightContext`, `attachments` und `attachAs` sind auf diesem Pfad nicht verfügbar, da sichtbare Sitzungen persistente Dashboard-Sitzungen sind, die über `sessions.create` erstellt werden. Sichtbares Starten wird abgelehnt, wenn der Anforderer selbst mit einer geerbten Werkzeug-Zulassungs- oder -Sperrliste gestartet wurde; diese Einschränkung wird beim Start festgelegt und kann nicht per Konfiguration überschrieben werden. Das Auflisten und Adressieren von Sitzungen richtet sich nach `tools.sessions.visibility`; der standardmäßige Geltungsbereich `tree` umfasst die aktuelle Sitzung und deren eigenen Start-Unterbaum. Informationen zur Benennung von Checkouts sowie zum Einrichten, Bereinigen und Wiederherstellen finden Sie unter [Verwaltete Worktrees](/de/concepts/managed-worktrees).

### Aufgabennamen und Zielauswahl

`taskName` ist ein modellseitiger Bezeichner für die Orchestrierung und kein Sitzungsschlüssel.
Verwenden Sie ihn für stabile Namen untergeordneter Agenten wie `review_subagents`,
`linux_validation` oder `docs_update`, wenn ein Koordinator den
untergeordneten Agenten später möglicherweise überprüfen muss.

Die Zielauflösung akzeptiert exakte Übereinstimmungen mit `taskName` und eindeutige
Präfixe. Der Abgleich ist auf dasselbe aktive bzw. kürzlich verwendete Zielfenster beschränkt,
das auch von nummerierten `/subagents`-Zielen verwendet wird, sodass ein veralteter abgeschlossener untergeordneter Agent einen
wiederverwendeten Bezeichner nicht mehrdeutig macht. Wenn zwei aktive oder kürzlich verwendete untergeordnete Agenten denselben
`taskName` verwenden, ist das Ziel mehrdeutig; verwenden Sie stattdessen den Listenindex, den Sitzungsschlüssel oder
die Lauf-ID.

Die reservierten Ziele `last` und `all` sind keine gültigen `taskName`-Werte,
da sie bereits Steuerungsbedeutungen haben.

## Werkzeug: `sessions_yield`

Beendet den aktuellen Modellturn und wartet darauf, dass Laufzeitereignisse, vor allem
Abschlussereignisse von Sub-Agenten, als nächste Nachricht eintreffen. Verwenden Sie es nach dem
Start erforderlicher untergeordneter Arbeiten, wenn der Anforderer erst nach deren Abschluss eine endgültige
Antwort erstellen kann.

`sessions_yield` ist das Grundelement zum Warten. Ersetzen Sie es nicht durch Polling-
Schleifen über `subagents`, `sessions_list`, `sessions_history`, Shell-
`sleep` oder Prozess-Polling, nur um den Abschluss untergeordneter Agenten zu erkennen.

Verwenden Sie `sessions_yield` nur, wenn die effektive Werkzeugliste der Sitzung
es enthält. Einige minimale oder benutzerdefinierte Werkzeugprofile stellen möglicherweise `sessions_spawn` und
`subagents` bereit, ohne `sessions_yield` bereitzustellen; erfinden Sie in diesem Fall
keine Polling-Schleife, nur um auf den Abschluss zu warten.

Wenn aktive untergeordnete Agenten vorhanden sind, fügt OpenClaw in normalen Turns einen kompakten, zur Laufzeit erzeugten
`Active Subagents`-Promptblock ein, damit der Anforderer
die aktuellen untergeordneten Sitzungen, Lauf-IDs, Statusangaben, Bezeichnungen, Aufgaben und
`taskName`-Aliasse ohne Polling sehen kann. Die Aufgaben- und Bezeichnungsfelder in diesem
Block werden als Daten und nicht als Anweisungen zitiert, da sie aus vom Benutzer oder Modell bereitgestellten Startargumenten
stammen können.

## Werkzeug: `subagents`

Listet gestartete Sub-Agent-Läufe und Datensätze zu Hintergrundaufgaben auf, die dem
Sitzungsbaum des Anforderers gehören. Die Aufgabenzeilen umfassen native Sub-Agenten, ACP-Läufe,
Gateway-CLI-/Medienarbeiten und Cron-Ausführungen. Der Geltungsbereich ist auf den aktuellen
Anforderer beschränkt; ein untergeordneter Agent kann nur seine eigenen kontrollierten untergeordneten Agenten sehen.

Verwenden Sie `subagents` für Statusabfragen und Fehlerdiagnosen bei Bedarf. Verwenden Sie `sessions_yield`, um
auf Abschlussereignisse zu warten.

Verwenden Sie `action: "cancel"` mit einer von `action: "list"` zurückgegebenen `taskId`, um
eine Aufgabe zu stoppen. Abbrüche sind auf den kontrollierten Sitzungsbaum beschränkt; ein
Sub-Agent am Ende des Baums kann keine Arbeit abbrechen, die einer anderen Sitzung gehört.

## Thread-gebundene Sitzungen

Wenn Thread-Bindungen für einen Kanal aktiviert sind, kann ein Sub-Agent an
einen Thread gebunden bleiben, sodass nachfolgende Benutzernachrichten in diesem Thread weiterhin an
dieselbe Sub-Agent-Sitzung geleitet werden.

### Kanäle mit Thread-Unterstützung

Ein Kanal unterstützt persistente Thread-gebundene Sub-Agent-Sitzungen
(`sessions_spawn` mit `thread: true`), wenn er einen Adapter für Konversationsbindungen
registriert. Mitgelieferte Kanäle mit dieser Unterstützung: **Discord**,
**iMessage**, **Matrix** und **Telegram**. Discord und Matrix erstellen standardmäßig
einen untergeordneten Thread; Telegram und iMessage binden standardmäßig die
aktuelle Konversation. Verwenden Sie die kanalspezifischen `threadBindings`-Konfigurationsschlüssel für
Aktivierung, Zeitlimits und `spawnSessions`.

### Schnellablauf

<Steps>
  <Step title="Starten">
    `sessions_spawn` mit `thread: true` (und optional `mode: "session"`).
  </Step>
  <Step title="Binden">
    OpenClaw erstellt oder bindet im aktiven Kanal einen Thread an dieses Sitzungsziel.
  </Step>
  <Step title="Folgenachrichten weiterleiten">
    Antworten und Folgenachrichten in diesem Thread werden an die gebundene Sitzung weitergeleitet.
  </Step>
  <Step title="Zeitlimits prüfen">
    Verwenden Sie `/session idle`, um die automatische Aufhebung des Fokus bei Inaktivität zu prüfen oder zu aktualisieren, und
    `/session max-age`, um die feste Obergrenze zu steuern.
  </Step>
  <Step title="Trennen">
    Verwenden Sie `/unfocus`, um die Bindung manuell zu trennen.
  </Step>
</Steps>

### Manuelle Steuerung

| Befehl             | Wirkung                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------- |
| `/focus <target>`  | Bindet den aktuellen Thread an ein Sub-Agent-/Sitzungsziel oder erstellt dafür einen Thread |
| `/unfocus`         | Entfernt die Bindung für den aktuell gebundenen Thread                                    |
| `/agents`          | Listet aktive Läufe und den Bindungsstatus auf (`binding:<id>`, `unbound` oder `bindings unavailable`) |
| `/session idle`    | Prüft/aktualisiert die automatische Aufhebung des Fokus bei Inaktivität (nur fokussierte gebundene Threads) |
| `/session max-age` | Prüft/aktualisiert die feste Obergrenze (nur fokussierte gebundene Threads)                |

### Konfigurationsschalter

- **Globaler Standard:** `session.threadBindings.enabled`, `session.threadBindings.idleHours`, `session.threadBindings.maxAgeHours`.
- **Kanalspezifische Überschreibungs- und automatische Bindungsschlüssel beim Start** sind adapterspezifisch. Siehe oben [Kanäle mit Thread-Unterstützung](#thread-supporting-channels).

Aktuelle Adapterdetails finden Sie in der [Konfigurationsreferenz](/de/gateway/configuration-reference) und unter
[Slash-Befehle](/de/tools/slash-commands).

### Zulassungsliste

<ParamField path="agents.entries.*.subagents.allowAgents" type="string[]">
  Liste konfigurierter Agenten-IDs, die über explizites `agentId` als Ziel verwendet werden können (`["*"]` erlaubt jedes konfigurierte Ziel). Standardmäßig ist nur der anfordernde Agent zulässig. Wenn Sie eine Liste festlegen und weiterhin möchten, dass der Anforderer sich mit `agentId` selbst startet, nehmen Sie die ID des Anforderers in die Liste auf.
</ParamField>
<ParamField path="agents.defaults.subagents.allowAgents" type="string[]">
  Standardmäßige Zulassungsliste konfigurierter Zielagenten, die verwendet wird, wenn der anfordernde Agent kein eigenes `subagents.allowAgents` festlegt.
</ParamField>
<ParamField path="agents.defaults.subagents.requireAgentId" type="boolean" default="false">
  Blockiert `sessions_spawn`-Aufrufe, bei denen `agentId` fehlt (erzwingt die ausdrückliche Profilauswahl). Agentenspezifische Überschreibung: `agents.entries.*.subagents.requireAgentId`.
</ParamField>
<ParamField path="agents.defaults.subagents.announceTimeoutMs" type="number" default="120000">
  Aufrufbezogenes Zeitlimit für Zustellversuche von Gateway-`agent`-Ankündigungen. Werte sind positive ganzzahlige Millisekundenwerte und werden auf das plattformsichere Timermaximum begrenzt. Vorübergehende Wiederholungsversuche können dazu führen, dass die gesamte Wartezeit für die Ankündigung länger als ein konfiguriertes Zeitlimit ist.
</ParamField>

Wenn die Sitzung des Anforderers in einer Sandbox ausgeführt wird, lehnt `sessions_spawn` Ziele ab,
die ohne Sandbox ausgeführt würden.

### Ermittlung

Verwenden Sie `agents_list`, um zu sehen, welche Agenten-IDs derzeit für
`sessions_spawn` zulässig sind. Die Antwort enthält für jeden aufgeführten Agenten das effektive
Modell und eingebettete Laufzeitmetadaten, damit Aufrufer zwischen OpenClaw, dem Codex-
App-Server und anderen konfigurierten nativen Laufzeiten unterscheiden können.

`allowAgents`-Einträge müssen auf konfigurierte Agenten-IDs in `agents.entries.*` verweisen.
`["*"]` bedeutet jeden konfigurierten Zielagenten einschließlich des anfragenden Agenten. Wenn eine Agentenkonfiguration
gelöscht wird, ihre ID aber in `allowAgents` verbleibt, lehnt `sessions_spawn` diese ID ab
und `agents_list` lässt sie aus. Führen Sie `openclaw doctor --fix` aus, um veraltete
Positivlisteneinträge zu bereinigen, oder fügen Sie einen minimalen `agents.entries.*`-Eintrag hinzu, wenn das Ziel
weiterhin erzeugt werden können soll und dabei Standardwerte erbt.

### Automatische Archivierung

- Sub-Agenten-Sitzungen werden nach `agents.defaults.subagents.archiveAfterMinutes` (Standardwert: `60`) automatisch archiviert.
- Die Archivierung verwendet `sessions.delete` und benennt das Transkript in `*.deleted.<timestamp>` um (im selben Ordner).
- `cleanup: "delete"` archiviert unmittelbar nach der Bekanntgabe (das Transkript bleibt durch die Umbenennung erhalten).
- Die automatische Archivierung erfolgt nach bestem Bemühen; ausstehende Timer gehen bei einem Neustart des Gateways verloren.
- Konfigurierte Laufzeitüberschreitungen führen **nicht** zur automatischen Archivierung; sie beenden lediglich den Lauf. Die Sitzung bleibt bis zur automatischen Archivierung erhalten.
- Die automatische Archivierung gilt gleichermaßen für Sitzungen der Tiefe 1 und 2.
- Die Browserbereinigung erfolgt getrennt von der Archivbereinigung: Nachverfolgte Browser-Tabs und -Prozesse werden nach bestem Bemühen geschlossen, wenn der Lauf endet, selbst wenn das Transkript bzw. der Sitzungseintrag beibehalten wird.

## Verschachtelte Sub-Agenten

Standardmäßig können Sub-Agenten keine eigenen Sub-Agenten erzeugen
(`maxSpawnDepth: 1`). Setzen Sie `maxSpawnDepth: 2`, um eine
Verschachtelungsebene zu aktivieren – das **Orchestrator-Muster**: Hauptagent → Orchestrator-Sub-Agent →
Worker-Sub-Sub-Agenten.

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxSpawnDepth: 2, // Sub-Agenten dürfen untergeordnete Agenten erzeugen (Standardwert: 1, Bereich 1-5)
        maxChildrenPerAgent: 5, // maximale Anzahl aktiver untergeordneter Agenten pro Agentensitzung (Standardwert: 5, Bereich 1-20)
        maxConcurrent: 8, // globale Obergrenze der Parallelitätsspur (Standardwert: 8)
        runTimeoutSeconds: 900, // standardmäßige Zeitüberschreitung für sessions_spawn (0 = keine Zeitüberschreitung)
        announceTimeoutMs: 120000, // Gateway-Zeitüberschreitung für Bekanntgaben pro Aufruf
      },
    },
  },
}
```

### Tiefenebenen

| Tiefe | Form des Sitzungsschlüssels                            | Rolle                                          | Kann Agenten erzeugen?                   |
| ----- | -------------------------------------------- | --------------------------------------------- | ---------------------------- |
| 0     | `agent:<id>:main`                            | Hauptagent                                    | Immer                       |
| 1     | `agent:<id>:subagent:<uuid>`                 | Sub-Agent (Orchestrator, wenn Tiefe 2 zulässig ist) | Nur wenn `maxSpawnDepth >= 2` |
| 2     | `agent:<id>:subagent:<uuid>:subagent:<uuid>` | Sub-Sub-Agent (Worker auf Blattebene)                   | Nie                        |

### Bekanntgabekette

Ergebnisse fließen die Kette aufwärts zurück:

1. Worker der Tiefe 2 wird fertig → gibt das Ergebnis an seinen übergeordneten Agenten (Orchestrator der Tiefe 1) bekannt.
2. Der Orchestrator der Tiefe 1 empfängt die Bekanntgabe, fasst die Ergebnisse zusammen und wird fertig → gibt das Ergebnis an den Hauptagenten bekannt.
3. Der Hauptagent empfängt die Bekanntgabe und übermittelt sie an den Benutzer.

Jede Ebene sieht nur Bekanntgaben ihrer direkt untergeordneten Agenten.

<Note>
**Betriebshinweis:** Starten Sie untergeordnete Aufgaben einmal und warten Sie auf Abschlussereignisse,
anstatt Abfrageschleifen um `sessions_list`,
`sessions_history`, `/subagents list` oder `exec`-Schlafbefehle zu erstellen.
`sessions_list` und `/subagents list` halten die Beziehungen zu untergeordneten Sitzungen
auf aktive Aufgaben ausgerichtet – aktive untergeordnete Agenten bleiben zugeordnet, beendete untergeordnete Agenten bleiben
für ein kurzes Zeitfenster sichtbar und veraltete, nur im Speicher vorhandene Verknüpfungen zu untergeordneten Agenten werden
nach Ablauf ihres Aktualitätsfensters ignoriert. Dadurch wird verhindert, dass alte `spawnedBy`-/
`parentSessionKey`-Metadaten nach einem Neustart nicht mehr vorhandene untergeordnete Agenten wiederherstellen.
Wenn ein Abschlussereignis eines untergeordneten Agenten eintrifft, nachdem Sie bereits die
endgültige Antwort gesendet haben, ist die korrekte Folgeaktion exakt das stille Token
`NO_REPLY` / `no_reply`.
</Note>

### Tool-Richtlinie nach Tiefe

- Ein untergeordneter Agent erfasst beim Erzeugen die effektive Absenderrichtlinie des Anfragenden. Läufe untergeordneter Agenten ohne Absender sowie von authentifizierten Operatoren fortgesetzte Läufe behalten diese Momentaufnahme bei, selbst wenn sich `toolsBySender` später ändert; aktuelle globale Einschränkungen sowie Einschränkungen für Agenten, Provider, Sandbox und Sub-Agenten gelten weiterhin. Bei einem neuen externen Kanalvorgang, der an den untergeordneten Agenten gerichtet ist, wird stattdessen die aktuelle Absenderrichtlinie neu aufgelöst.
- Rolle und Steuerungsumfang werden beim Erzeugen in die Sitzungsmetadaten geschrieben. Dadurch wird verhindert, dass flache oder wiederhergestellte Sitzungsschlüssel versehentlich erneut Orchestrator-Berechtigungen erhalten.
- **Tiefe 1 (Orchestrator, wenn `maxSpawnDepth >= 2`):** erhält `sessions_spawn`, `subagents`, `sessions_list`, `sessions_history`, damit er untergeordnete Agenten erzeugen und deren Status prüfen kann. Andere Sitzungs-/Systemtools bleiben gesperrt.
- **Tiefe 1 (Blattebene, wenn `maxSpawnDepth == 1`):** keine Sitzungstools (aktuelles Standardverhalten).
- **Tiefe 2 (Worker auf Blattebene):** keine Sitzungstools – `sessions_spawn` ist auf Tiefe 2 immer gesperrt. Weitere untergeordnete Agenten können nicht erzeugt werden.

### Erzeugungslimit pro Agent

Jede Agentensitzung (auf jeder Tiefe) kann gleichzeitig höchstens `maxChildrenPerAgent`
(Standardwert: `5`) aktive untergeordnete Agenten haben. Dadurch wird eine unkontrollierte Auffächerung
durch einen einzelnen Orchestrator verhindert.

### Kaskadierendes Beenden

Beim Beenden eines Orchestrators der Tiefe 1 werden automatisch alle seine untergeordneten Agenten
der Tiefe 2 beendet:

- `/stop` im Hauptchat beendet alle Agenten der Tiefe 1 und kaskadiert zu deren untergeordneten Agenten der Tiefe 2.

## Authentifizierung

Die Authentifizierung von Sub-Agenten wird anhand der **Agenten-ID** aufgelöst, nicht anhand des Sitzungstyps:

- Der Sitzungsschlüssel des Sub-Agenten lautet `agent:<agentId>:subagent:<uuid>`.
- Der Authentifizierungsspeicher wird aus `agentDir` dieses Agenten geladen.
- Die Authentifizierungsprofile des Hauptagenten werden als **Fallback** zusammengeführt; bei Konflikten haben Agentenprofile Vorrang vor den Profilen des Hauptagenten.

Die Zusammenführung ist additiv, sodass die Profile des Hauptagenten stets als
Fallbacks verfügbar sind. Eine vollständig isolierte Authentifizierung pro Agent wird noch nicht unterstützt.

## Bekanntgabe

Sub-Agenten melden Ergebnisse über einen Bekanntgabeschritt zurück:

- Der Bekanntgabeschritt wird innerhalb der Sub-Agenten-Sitzung ausgeführt (nicht in der Sitzung des Anfragenden).
- Wenn der Sub-Agent exakt mit `ANNOUNCE_SKIP` antwortet, wird nichts veröffentlicht.
- Wenn der neueste Assistententext exakt dem stillen Token `NO_REPLY` / `no_reply` entspricht, wird die Bekanntgabeausgabe unterdrückt, selbst wenn zuvor sichtbare Fortschrittsmeldungen vorhanden waren.

Die Zustellung hängt von der Tiefe des Anfragenden ab:

- Sitzungen von Anfragenden auf oberster Ebene verwenden einen nachfolgenden `agent`-Aufruf mit externer Zustellung (`deliver=true`).
- Verschachtelte Sub-Agenten-Sitzungen des Anfragenden erhalten eine interne nachfolgende Einspeisung (`deliver=false`), damit der Orchestrator die Ergebnisse untergeordneter Agenten innerhalb der Sitzung zusammenfassen kann.
- Wenn eine verschachtelte Sub-Agenten-Sitzung des Anfragenden nicht mehr vorhanden ist, greift OpenClaw nach Möglichkeit auf den Anfragenden dieser Sitzung zurück.

Bei Sitzungen von Anfragenden auf oberster Ebene löst die direkte Zustellung im Abschlussmodus zunächst
eine gebundene Konversations-/Thread-Route und eine Hook-Überschreibung auf und ergänzt anschließend
fehlende Kanalzielfelder aus der gespeicherten Route der Sitzung des Anfragenden.
Dadurch bleiben Abschlussmeldungen im richtigen Chat/Thema, selbst wenn der Ursprung
des Abschlusses nur den Kanal identifiziert.

Beim Erstellen verschachtelter Abschlussbefunde wird die Aggregation der Abschlüsse untergeordneter Agenten auf den aktuellen Lauf des Anfragenden
begrenzt. Dadurch wird verhindert, dass Ausgaben untergeordneter Agenten aus früheren Läufen
in die aktuelle Bekanntgabe gelangen. Bekanntgabeantworten behalten die
Thread-/Themenweiterleitung bei, sofern sie auf Kanaladaptern verfügbar ist.

### Bekanntgabekontext

Der Bekanntgabekontext wird in einen stabilen internen Ereignisblock normalisiert:

| Feld          | Quelle                                                                                                   |
| -------------- | -------------------------------------------------------------------------------------------------------- |
| Quelle         | `subagent` oder `cron`                                                                                     |
| Sitzungs-IDs    | Sitzungsschlüssel/-ID des untergeordneten Agenten                                                                                     |
| Typ           | Bekanntgabetyp + Aufgabenbezeichnung                                                                               |
| Status         | Aus dem Laufzeitergebnis abgeleitet (`ok`, `error`, `timeout` oder `unknown`) – **nicht** aus Modelltext abgeleitet |
| Ergebnisinhalt | Neuester sichtbarer Assistententext des untergeordneten Agenten                                                             |
| Folgeaktion      | Anweisung, wann geantwortet bzw. still geblieben werden soll                                                      |

Fehlgeschlagene abgeschlossene Läufe melden den Fehlerstatus, ohne erfassten
Antworttext erneut wiederzugeben. Tool-/ToolResult-Ausgaben werden nicht zum Ergebnistext des untergeordneten Agenten hochgestuft.

### Statistikzeile

Bekanntgabe-Nutzlasten enthalten am Ende eine Statistikzeile (auch bei Zeilenumbruch):

- Laufzeit (z. B. `runtime 5m12s`).
- Token-Nutzung (Eingabe/Ausgabe/gesamt).
- Geschätzte Kosten, wenn die Modellpreise konfiguriert sind (`models.providers.*.models[].cost`).
- `sessionKey`, `sessionId` und Transkriptpfad, damit der Hauptagent den Verlauf über `sessions_history` abrufen oder die Datei auf dem Datenträger prüfen kann.

Interne Metadaten sind ausschließlich für die Orchestrierung bestimmt; benutzerseitige Antworten
sollten in normaler Assistentensprache neu formuliert werden.

### Warum `sessions_history` bevorzugt werden sollte

`sessions_history` ist der sicherere Orchestrierungsweg, um das Transkript eines untergeordneten Agenten
innerhalb eines Agentenvorgangs zu lesen:

- Schwärzt Anmeldedaten-/Token-ähnlichen Text, selbst wenn die allgemeine Protokollschwärzung deaktiviert ist.
- Kürzt lange Textblöcke (4000 Zeichen pro Block) und entfernt Denksignaturen, Wiedergabe-Nutzlasten für Schlussfolgerungen und Inline-Bilddaten.
- Erzwingt eine Antwortobergrenze von 80 KB; übergroße Zeilen werden durch `[sessions_history omitted: message too large]` ersetzt.
- Verwenden Sie `nextOffset`, sofern vorhanden, um rückwärts durch ältere Transkriptfenster zu blättern.
- `sessions_history` entfernt **keine** Schlussfolgerungs-Tags, `<relevant-memories>`-Gerüststrukturen oder Tool-Aufruf-XML aus Nachrichtentexten – es gibt strukturierte Inhaltsblöcke nahe an der Rohform des Transkripts zurück, lediglich geschwärzt und größenbegrenzt. `/subagents log` wendet die umfassendere Prosa-Bereinigung an (entfernt Schlussfolgerungs-Tags, Speichergerüststrukturen und Tool-Aufruf-XML), da es einfache Chatzeilen statt strukturierter Blöcke rendert.
- Die direkte Prüfung des auf dem Datenträger gespeicherten Rohtranskripts ist der Fallback, wenn Sie das vollständige bytegenaue Transkript benötigen.

## Tool-Richtlinie

Sub-Agenten verwenden zunächst dieselbe Profil- und Tool-Richtlinien-Pipeline wie der übergeordnete
oder Zielagent. Anschließend wendet OpenClaw die Einschränkungsebene für Sub-Agenten
an.

Sub-Agenten verlieren unabhängig von Tiefe oder Rolle stets `gateway`, `agents_list`, `session_status` und
`cron` (Tools auf Systemebene/interaktive Tools oder
Tools, die der Hauptagent koordinieren sollte). Sub-Agenten auf Blattebene (standardmäßiges Verhalten auf Tiefe 1
und immer auf Tiefe 2) verlieren zusätzlich `subagents`,
`sessions_list`, `sessions_history` und `sessions_spawn`. Sub-Agenten erhalten niemals
das Tool `message` – es wird beim Erzeugen deaktiviert und nicht durch
diese Sperrliste gefiltert – und `sessions_send` bleibt gesperrt, sodass Sub-Agenten
ausschließlich über die Bekanntgabekette kommunizieren.

`sessions_history` bleibt auch hier eine begrenzte, bereinigte Erinnerungsansicht –
es handelt sich nicht um eine Ausgabe des Rohtranskripts.

Wenn `maxSpawnDepth >= 2`, erhalten Orchestrator-Sub-Agenten der Tiefe 1 zusätzlich
`sessions_spawn`, `subagents`, `sessions_list` und
`sessions_history`, damit sie ihre untergeordneten Agenten verwalten können.

### Überschreiben über die Konfiguration

```json5
{
  agents: {
    defaults: {
      subagents: {
        maxConcurrent: 1,
      },
    },
  },
  tools: {
    subagents: {
      tools: {
        // Verweigerung hat Vorrang
        deny: ["gateway", "cron"],
        // wenn „allow“ festgelegt ist, sind ausschließlich diese Einträge zulässig („deny“ hat weiterhin Vorrang)
        // allow: ["read", "exec", "process"]
      },
    },
  },
}
```

`tools.subagents.tools.allow` ist ein abschließender Filter, der ausschließlich Zulässiges berücksichtigt. Er kann die
bereits aufgelöste Werkzeugmenge einschränken, aber kein durch
`tools.profile` entferntes Werkzeug **wieder hinzufügen**. Beispielsweise enthält `tools.profile: "coding"`
`web_search`/`web_fetch`, jedoch nicht das Werkzeug `browser`. Damit
Unteragenten mit Coding-Profil Browserautomatisierung verwenden können, fügen Sie „browser“ in der
Profilphase hinzu:

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

Verwenden Sie `agents.entries.*.tools.alsoAllow: ["browser"]` pro Agent, wenn nur ein
Agent Browserautomatisierung erhalten soll.

## Nebenläufigkeit

Unteragenten verwenden eine dedizierte prozessinterne Warteschlangenspur:

- **Spurname:** `subagent`
- **Nebenläufigkeit:** `agents.defaults.subagents.maxConcurrent` (Standardwert: `8`)

## Aktivität und Wiederherstellung

OpenClaw betrachtet das Fehlen von `endedAt` nicht als dauerhaften Beweis dafür, dass ein
Unteragent noch aktiv ist. Nicht beendete Läufe, die älter als das Zeitfenster für veraltete Läufe sind
(2 Stunden oder das konfigurierte Laufzeitlimit zuzüglich einer kurzen Kulanzfrist,
je nachdem, welcher Zeitraum länger ist), werden in `/subagents list`,
Statuszusammenfassungen, der Abschlussprüfung für Nachkommen und den
Nebenläufigkeitsprüfungen pro Sitzung nicht mehr als aktiv/ausstehend gezählt.

Nach einem Neustart des Gateways werden wiederhergestellte, veraltete und nicht beendete Läufe entfernt, sofern
ihre untergeordnete Sitzung nicht als `abortedLastRun: true` gekennzeichnet ist. Durch einen Neustart abgebrochene
Läufe bleiben für den Wiederherstellungsablauf verwaister Unteragenten registriert: Veraltete
Läufe werden ohne Fortsetzung abgeschlossen, während aktuelle untergeordnete Sitzungen eine
synthetische Fortsetzungsnachricht erhalten, bevor die Abbruchmarkierung gelöscht wird.

Die automatische Wiederherstellung nach einem Neustart ist pro untergeordneter Sitzung begrenzt. Wenn derselbe
untergeordnete Unteragent innerhalb des Zeitfensters für schnelle erneute Blockierungen wiederholt
für die Wiederherstellung verwaister Prozesse akzeptiert wird, speichert OpenClaw in dieser
Sitzung einen Wiederherstellungs-Tombstone und setzt sie bei späteren Neustarts nicht mehr automatisch fort. Führen Sie
`openclaw tasks maintenance --apply` aus, um den Aufgabendatensatz abzugleichen, oder
`openclaw doctor --fix`, um veraltete Abbruch-Wiederherstellungsmarkierungen in
Sitzungen mit Tombstone zu löschen.

<Note>
Wenn das Starten eines Unteragenten mit Gateway `PAIRING_REQUIRED` /
`scope-upgrade` fehlschlägt, prüfen Sie den RPC-Aufrufer, bevor Sie den Kopplungsstatus bearbeiten.
Die interne `sessions_spawn`-Koordination wird prozessintern ausgeführt, wenn der
Aufrufer bereits im Kontext einer Gateway-Anfrage läuft. Daher öffnet sie
weder einen Loopback-WebSocket noch ist sie vom Umfangs-Basiswert gekoppelter Geräte der CLI
abhängig. Aufrufer außerhalb des Gateway-Prozesses verwenden weiterhin den WebSocket-
Fallback als `client.id: "gateway-client"` mit `client.mode: "backend"`
über direkte Loopback-Authentifizierung mit gemeinsamem Token/Passwort. Entfernte Aufrufer, explizite
`deviceIdentity`, explizite Geräte-Token-Pfade sowie Browser-/Node-Clients
benötigen für Umfangserweiterungen weiterhin die normale Gerätegenehmigung.
</Note>

## Beenden

- Das Senden von `/stop` im Chat des Anforderers bricht dessen Sitzung ab und beendet alle daraus gestarteten aktiven Unteragentenläufe, einschließlich verschachtelter Nachkommen.

## Einschränkungen

- Die Ankündigung von Unteragenten erfolgt nach dem **Best-Effort-Prinzip**. Wenn das Gateway neu startet, gehen ausstehende Arbeiten zur Rückmeldung verloren.
- Unteragenten nutzen weiterhin dieselben Prozessressourcen des Gateways gemeinsam; behandeln Sie `maxConcurrent` als Sicherheitsventil.
- `sessions_spawn` ist immer nicht blockierend: Es gibt `{ status: "accepted", runId, childSessionKey }` sofort zurück.
- Der Kontext von Unteragenten fügt nur `AGENTS.md` und `TOOLS.md` ein (kein `SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `HEARTBEAT.md` oder `BOOTSTRAP.md`). Codex-native Unteragenten folgen derselben Grenze: `TOOLS.md` verbleibt in den geerbten Codex-Thread-Anweisungen, während ausschließlich für den übergeordneten Agenten bestimmte Persona-, Identitäts- und Benutzerdateien als auf den Turn beschränkte Zusammenarbeitsanweisungen eingefügt werden, damit untergeordnete Agenten sie nicht klonen.
- Die maximale Verschachtelungstiefe beträgt 5 (Bereich für `maxSpawnDepth`: 1-5). Für die meisten Anwendungsfälle wird Tiefe 2 empfohlen.
- `maxChildrenPerAgent` begrenzt die aktiven untergeordneten Agenten pro Sitzung (Standardwert: `5`, Bereich: `1-20`).

## Verwandte Themen

- [Sitzungswerkzeuge und Statusänderungen](/de/concepts/session-tool)
- [ACP-Agenten](/de/tools/acp-agents)
- [Senden durch Agenten](/de/tools/agent-send)
- [Hintergrundaufgaben](/de/automation/tasks)
- [Sandbox-Werkzeuge für mehrere Agenten](/de/tools/multi-agent-sandbox-tools)
