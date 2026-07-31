---
read_when:
    - Sie möchten Session-Routing und -Isolation verstehen
    - Sie möchten den DM-Geltungsbereich für Mehrbenutzerkonfigurationen festlegen
    - Sie debuggen tägliche oder durch Inaktivität ausgelöste Sitzungszurücksetzungen
summary: So verwaltet OpenClaw Konversationssitzungen
title: Sitzungsverwaltung
x-i18n:
    generated_at: "2026-07-26T18:25:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de85fe5a623bdbc6d5564d822b39e9077a582b0816b62ab30d2f7245bd097000
    source_path: concepts/session.md
    workflow: 16
---

OpenClaw leitet jede eingehende Nachricht basierend auf ihrer Herkunft an eine **Sitzung** weiter:
Direktnachrichten, Gruppenchats, Cron-Aufträge usw. Der gesamte Sitzungsstatus gehört dem
**Gateway**; UI-Clients fragen Sitzungsdaten beim Gateway ab.

Informationen zum Standard für persönliche Agenten – eine fortlaufende, von all Ihren
Direktnachrichtenkanälen gemeinsam genutzte Unterhaltung, in die Gruppenaktivitäten und Hintergrundarbeit einfließen – finden Sie unter
[Die Hauptsitzung](/de/concepts/main-session).

## So werden Nachrichten weitergeleitet

| Quelle          | Verhalten                  |
| --------------- | ------------------------- |
| Direktnachrichten | Standardmäßig gemeinsame Sitzung |
| Gruppenchats     | Für jede Gruppe isoliert        |
| Räume/Kanäle  | Für jeden Raum isoliert         |
| Cron-Aufträge       | Neue Sitzung pro Ausführung     |
| Webhooks        | Für jeden Hook isoliert         |

## Isolierung von Direktnachrichten

Standardmäßig nutzen alle Direktnachrichten eine gemeinsame Sitzung, um Kontinuität zu gewährleisten. Dies eignet sich für
Einzelnutzerkonfigurationen.

<Warning>
Wenn mehrere Personen Ihrem Agenten Nachrichten senden können, aktivieren Sie die Isolierung von Direktnachrichten. Ohne sie
teilen sich alle Benutzer denselben Unterhaltungskontext, sodass Alices private Nachrichten für
Bob sichtbar wären.
</Warning>

```json5
{
  session: {
    dmScope: "per-channel-peer", // nach Kanal + Absender isolieren
  },
}
```

`session.dmScope`-Optionen:

| Wert                      | Verhalten                                                 |
| -------------------------- | -------------------------------------------------------- |
| `main` (Standard)           | Alle Direktnachrichten nutzen die [Hauptsitzung](/de/concepts/main-session) gemeinsam |
| `per-peer`                 | Kanalübergreifend nach Absender isolieren                       |
| `per-channel-peer`         | Nach Kanal + Absender isolieren (empfohlen)                |
| `per-account-channel-peer` | Nach Konto + Kanal + Absender isolieren                    |

<Tip>
Wenn dieselbe Person Sie über mehrere Kanäle kontaktiert, verwenden Sie
`session.identityLinks`, um ihre Identitäten einer kanonischen Peer-ID zuzuordnen, damit
sie eine Sitzung gemeinsam nutzen.
</Tip>

### Verknüpfte Kanäle andocken

Andockbefehle verschieben die Antwortweiterleitung der aktuellen Direktchat-Sitzung zu einem anderen
verknüpften Kanal, ohne eine neue Sitzung zu starten. Beispiele, Konfiguration und
Fehlerbehebung finden Sie unter [Kanäle andocken](/de/concepts/channel-docking).

Überprüfen Sie Ihre Konfiguration mit `openclaw security audit`.

## Inkognito-Sitzungen

Inkognito-Sitzungen sind nur über den Bildschirm **Neuer Thread** der Control UI verfügbar. Aktivieren Sie **Inkognito**, bevor Sie den Thread starten, damit der Sitzungseintrag, das Transkript und der Compaction-Status im Prozessspeicher statt auf dem Datenträger gespeichert werden. Der Thread verschwindet beim Neustart des Gateway, führt die automatische Speicherleerung von OpenClaw nicht aus und erstellt beim Zurücksetzen oder Löschen kein Transkriptarchiv. Codex-gestützte Ausführungen starten ihren Harness-Thread ebenfalls im flüchtigen Modus, sodass Codex keine Rollout- oder lokalen Sitzungsstatusdateien schreibt; andere Modell-Provider verwenden HTTP-APIs und führen in OpenClaw kein lokales Provider-Transkript.

Das Segment `incognito-` ist für Dashboard-, Subagent- und verborgene interne Sitzungsschlüssel reserviert; `openclaw doctor --fix` benennt kollidierende persistente Legacy-Schlüssel um.

Inkognito schränkt die normalen Werkzeuge des Agenten nicht ein. Eine ausdrückliche Aufforderung zum Speichern von Informationen oder jeder werkzeuggesteuerte Schreibvorgang in eine Datei kann weiterhin Daten außerhalb des Inkognito-Sitzungsspeichers dauerhaft speichern. Ihr konfigurierter Modell-Provider verarbeitet weiterhin die von Ihnen gesendeten Nachrichten, die Diagnoseprotokollierung bleibt unverändert und OpenClaw zeichnet weiterhin inhaltsfreie Audit-Metadaten wie HMAC-Referenzen auf.

Auf Mehrbenutzer-Gateways sind Inkognito-Threads nur für Verbindungen mit Administratorbereich sichtbar und werden niemals über die Agentensitzungswerkzeuge oder die Transkriptsuche einer anderen Sitzung angezeigt. Dies schützt sie vor der Speicherung und anderen über das Gateway vermittelten Benutzern, nicht jedoch vor dem Eigentümer des Gateway oder dem Prozessbetreiber, die Live-Sitzungen jederzeit beobachten können.

## Über Unterhaltungen hinweg merken

Separate Transkripte steuern den lokalen Verlauf jeder Unterhaltung. Für einen persönlichen
oder vollständig vertrauenswürdigen Agenten fügt `memory.search.rememberAcrossConversations: true`
einen optionalen Abrufschritt über die anderen privaten
Unterhaltungen dieses Agenten hinweg hinzu; die Transkripte werden dabei nicht zusammengeführt.

Private Direktunterhaltungen und persistente explizite UI-Unterhaltungen können einander relevanten
Kontext bereitstellen. Gruppen und Kanäle bleiben in beide Richtungen getrennt:
Ihre Transkripte dienen nicht als private Erinnerungsquellen und Antworten in diesen
Unterhaltungen erhalten keinen Kontext aus privaten Transkripten. Die aktuelle
Unterhaltung wird ebenfalls ausgeschlossen, da ihr Verlauf bereits geladen ist.

Diese Einstellung ändert weder Sitzungsschlüssel, Direktnachrichtenbereich, Weiterleitung und Zustellung noch
`tools.sessions.visibility`. Der gemeinsame Arbeitsbereichsspeicher in `MEMORY.md` und
`memory/*.md` behält ebenfalls sein bisheriges Verhalten bei. Der aktuelle Speicher-Provider
muss den geschützten Abruf privater Transkripte unterstützen; Kontext-Engines wie
Lossless Claw bleiben unabhängig und können parallel dazu ausgeführt werden. Einzelheiten zur Einrichtung
und Laufzeit finden Sie unter [Active Memory](/de/concepts/active-memory#remember-across-conversations).

## Sitzungslebenszyklus

Sitzungen werden wiederverwendet, bis Sie sie manuell zurücksetzen oder eine automatische Rücksetzrichtlinie aktivieren:

- **Kein automatisches Zurücksetzen** (Standard `mode: "none"`) – Sitzungen behalten dieselbe
  `sessionId`; Compaction verwaltet den aktiven Kontext, während die Unterhaltung wächst.
- **Tägliches Zurücksetzen** (`mode: "daily"`) – aktiviert eine neue Sitzung zu einer konfigurierten lokalen
  Stunde (`session.reset.atHour`, Standard `4`, 0-23) auf dem Gateway-Host. Die tägliche
  Aktualität basiert darauf, wann die aktuelle `sessionId` begann, nicht auf späteren
  Metadatenschreibvorgängen.
- **Zurücksetzen bei Inaktivität** (`mode: "idle"`) – aktiviert nach `session.reset.idleMinutes`
  Inaktivität eine neue Sitzung. Die Aktualität bezüglich Inaktivität basiert auf der letzten tatsächlichen Benutzer-/Kanalinteraktion,
  sodass Heartbeat-, Cron- und Exec-Systemereignisse die
  Sitzung nicht aktiv halten.
- **Manuelles Zurücksetzen** – geben Sie im Chat `/new` oder `/reset` ein. `/new <model>`
  wechselt außerdem das Modell.

Wenn sowohl tägliches Zurücksetzen als auch Zurücksetzen bei Inaktivität konfiguriert sind, gilt das Ereignis, das zuerst abläuft.
Heartbeat-, Cron-, Exec- und andere Systemereignisvorgänge können Sitzungsmetadaten schreiben,
diese Schreibvorgänge verlängern jedoch nicht die Aktualität für tägliches oder inaktivitätsbasiertes Zurücksetzen. Wenn ein Zurücksetzen
die Sitzung wechselt, werden in der Warteschlange befindliche Systemereignishinweise für die alte Sitzung
verworfen, damit veraltete Hintergrundaktualisierungen nicht dem ersten Prompt der
neuen Sitzung vorangestellt werden.

Sitzungen mit einer aktiven Provider-eigenen CLI-Sitzung verwenden ebenfalls standardmäßig
kein automatisches Zurücksetzen. Verwenden Sie `/reset` oder konfigurieren Sie `session.reset` ausdrücklich, wenn diese Sitzungen
nach einer festgelegten Zeit ablaufen sollen.

Aktivieren Sie automatische Rücksetzungen global und überschreiben Sie sie anschließend je nach Chattyp oder Kanal:

```json5
{
  session: {
    reset: { mode: "daily", atHour: 4 },
    resetByType: {
      group: { mode: "idle", idleMinutes: 120 },
      thread: { mode: "daily", atHour: 6 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
  },
}
```

`resetByType` unterstützt `direct`, `group` und `thread`. Doctor migriert veraltete `dm`-Einträge zu `direct` und `session.idleMinutes` zu `session.reset.idleMinutes`; das Schema lehnt beide außer Betrieb genommenen Formen ab.

## Speicherort des Status

- **Laufzeit-Sitzungszeilen:** `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **Archivierte Transkriptdateien:** `~/.openclaw/agents/<agentId>/sessions/`
- **Quelle für die Migration von Legacy-Zeilen:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`

Die Sitzungszeilen in der agentenspezifischen SQLite-Datenbank führen separate
Lebenszyklus-Zeitstempel:

- `sessionStartedAt`: Zeitpunkt, zu dem die aktuelle `sessionId` begann; wird für das tägliche Zurücksetzen verwendet.
- `lastInteractionAt`: letzte Benutzer-/Kanalinteraktion, die die Inaktivitätslebensdauer verlängert.
- `updatedAt`: letzte Änderung der Speicherzeile; nützlich für Auflistung und Bereinigung, aber nicht
  maßgeblich für die Aktualität des täglichen oder inaktivitätsbasierten Zurücksetzens.

Während der Migration älterer Installationen importieren der Gateway-Start und `openclaw doctor
--fix` automatisch Legacy-`sessions.json`-Zeilen sowie den aktiven JSONL-Transkriptverlauf in
SQLite. Zeilen ohne `sessionStartedAt` werden, sofern verfügbar, anhand des
Sitzungs-Headers des Legacy-JSONL-Transkripts aufgelöst. Wenn einer älteren Zeile außerdem
`lastInteractionAt` fehlt, greift die Aktualität bezüglich Inaktivität auf die Startzeit dieser Sitzung zurück,
nicht auf spätere Buchführungsschreibvorgänge. Verwenden Sie `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` und die [Doctor-Migrationssequenz](/de/cli/doctor#session-sqlite-migration), wenn Sie eine ausdrückliche
Inspektion oder einen Validierungsnachweis wünschen.

## Sitzungswartung

OpenClaw begrenzt den Sitzungsspeicher im Laufe der Zeit über `session.maintenance`; die Standardwerte
sind dargestellt:

```json5
{
  session: {
    maintenance: {
      mode: "enforce", // "enforce" führt die Bereinigung durch; "warn" meldet nur
      pruneAfter: "30d",
      maxEntries: 500,
    },
  },
}
```

Bei `maxEntries`-Grenzwerten in Produktionsgröße verwenden Laufzeitschreibvorgänge des Gateway einen kleinen
Hochwasserpuffer und bereinigen stapelweise bis zur konfigurierten Obergrenze.
Lesevorgänge im Sitzungsspeicher bereinigen oder begrenzen beim Start des Gateway keine Einträge, sodass
der Start und isolierte Cron-Sitzungen nicht die Kosten einer vollständigen Speicherbereinigung tragen.
`openclaw sessions cleanup --enforce` wendet die Obergrenze sofort an.

Gateway-Testsitzungen für Modellausführungen sind standardmäßig kurzlebig. Zeilen, die
`agent:*:explicit:model-run-<uuid>` entsprechen, verwenden eine feste Aufbewahrungsdauer von `24h`, die Bereinigung ist jedoch
druckabhängig: Veraltete Testzeilen werden nur entfernt, wenn der Wartungs-/Obergrenzendruck
für Sitzungseinträge erreicht ist; dies erfolgt vor dem allgemeineren altersbasierten Grenzwert
für veraltete Einträge und der Eintragsobergrenze. Normale Direkt-, Gruppen-, Thread-, Cron-, Hook-, Heartbeat-,
ACP- und Subagent-Sitzungen übernehmen diese 24h-Aufbewahrung nicht.

Die Wartung bewahrt persistente externe Unterhaltungszeiger, einschließlich Gruppen-
und Thread-bezogener Chatsitzungen, während synthetische Cron-, Hook-, Heartbeat-,
ACP- und Subagent-Einträge weiterhin veralten und entfernt werden können.

Archivierte Sitzungen wurden von Benutzern abgelegt und sind von jedem automatischen Wartungspfad
ausgenommen, einschließlich altersbasierter Bereinigung, Eintragsobergrenzen, Bereinigung von Modellausführungen und
Verdrängung aufgrund des Datenträgerbudgets. Sie bleiben archiviert, bis Sie sie aus dem Archiv holen oder ausdrücklich
löschen.

Wenn Sie zuvor die Isolierung von Direktnachrichten verwendet und `session.dmScope` später wieder auf
`main` gesetzt haben, zeigen Sie veraltete Peer-bezogene Direktnachrichtenzeilen mit
`openclaw sessions cleanup --dry-run --fix-dm-scope` in einer Vorschau an. Durch Anwendung desselben Flags
werden diese alten Direktnachrichtenzeilen außer Betrieb genommen und ihre Transkripte als gelöschte
Archive aufbewahrt.

Zeigen Sie jeden Wartungslauf mit `openclaw sessions cleanup --dry-run` in einer Vorschau an.

## Sitzungen überprüfen

| Befehl                    | Anzeige                                           |
| -------------------------- | ----------------------------------------------- |
| `openclaw status`          | Pfad des Sitzungsspeichers und letzte Aktivität          |
| `openclaw sessions --json` | Alle Sitzungen (mit `--active <minutes>` filtern) |
| `/status` im Chat          | Kontextnutzung, Modell und Umschalter               |
| `/context list`            | Inhalt des System-Prompts                    |

## Weiterführende Informationen

- [Sitzungssuche](/de/concepts/session-search) – Volltextabruf über frühere Transkripte hinweg
- [Sitzungsbereinigung](/de/concepts/session-pruning) – Kürzen von Werkzeugergebnissen
- [Compaction](/de/concepts/compaction) – Zusammenfassen langer Unterhaltungen
- [Sitzungswerkzeuge](/de/concepts/session-tool) – Agentenwerkzeuge für sitzungsübergreifende Arbeit
- [Vertiefung zur Sitzungsverwaltung](/de/reference/session-management-compaction) –
  Speicherschema, Transkripte, Senderichtlinie, Ursprungsmetadaten und erweiterte Konfiguration
- [Mehrere Agenten](/de/concepts/multi-agent) – Weiterleitung und Sitzungsisolierung über Agenten hinweg
- [Hintergrundaufgaben](/de/automation/tasks) – wie getrennte Arbeit Aufgabendatensätze mit Sitzungsreferenzen erstellt
- [Kanalweiterleitung](/de/channels/channel-routing) – wie eingehende Nachrichten an Sitzungen weitergeleitet werden

## Verwandte Themen

- [Sitzungsbereinigung](/de/concepts/session-pruning)
- [Sitzungswerkzeuge](/de/concepts/session-tool)
- [Befehlswarteschlange](/de/concepts/queue)
