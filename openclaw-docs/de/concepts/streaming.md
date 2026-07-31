---
read_when:
    - Erläuterung der Funktionsweise von Streaming oder Chunking in Kanälen
    - Verhalten beim Block-Streaming oder Channel-Chunking ändern
    - Fehlerbehebung bei doppelten/verfrühten Blockantworten oder beim Vorschau-Streaming in Kanälen
summary: Streaming- und Chunking-Verhalten (Blockantworten, Vorschau-Streaming im Kanal, Moduszuordnung)
title: Streaming und Aufteilung in Blöcke
x-i18n:
    generated_at: "2026-07-26T17:48:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a498f2e490ae6f2ecdebba92f0b992f2e16d212eae6a437eb3a0ef8a59354e13
    source_path: concepts/streaming.md
    workflow: 16
---

OpenClaw verfügt über zwei unabhängige Streaming-Ebenen, und derzeit gibt es **kein echtes
Token-Delta-Streaming** für Channel-Nachrichten:

- **Block-Streaming (Channels):** Gibt abgeschlossene **Blöcke** aus, während der Assistent
  schreibt. Dabei handelt es sich um normale Channel-Nachrichten, nicht um Token-Deltas.
- **Vorschau-Streaming (Telegram/Discord/Slack/Matrix/Mattermost/MS Teams):**
  Aktualisiert während der Generierung eine temporäre **Vorschaunachricht** (Senden + Bearbeitungen/Anhängen).

## Startstatus der Control UI

Nachdem `chat.send` einen aktiven Lauf bestätigt hat, kann das Gateway einen typisierten,
groben Startstatus senden, bevor Text des Assistenten oder Tool-Aktivität sichtbar ist. Die
Control UI zeigt diesen Status neben der Arbeitsanzeige an, mit Phasen für die
Vorbereitung des Workspace, die Bereitstellung der Umgebung, die Vorbereitung des Kontexts und
den Modellstart.

Das erste Assistenten-Delta oder der erste Tool-Start ersetzt den Startstatus für
diesen Lauf dauerhaft. Der Genehmigungsstatus hat Vorrang, während ein Tool auf eine Aktion
des Operators wartet. Die Erstellung des Worktree und die anfängliche Cloud-Weiterleitung erfolgen, bevor ein Chat-Lauf
existiert. Daher wird ihr RPC-Fortschritt vor dem Lauf nicht als Startstatus des Laufs dargestellt;
die Bereitstellung der Umgebung erscheint hier nur, wenn ein aktiver Lauf einen
zurückgewonnenen Worker erneut bereitstellt.

## Block-Streaming (Channel-Nachrichten)

Beim Block-Streaming wird die Ausgabe des Assistenten in groben Abschnitten gesendet, sobald sie verfügbar ist.

```text
Modellausgabe
  └─ text_delta/Ereignisse
       ├─ (blockStreamingBreak=text_end)
       │    └─ Chunker gibt Blöcke aus, während der Puffer wächst
       └─ (blockStreamingBreak=message_end)
            └─ Chunker leert den Puffer bei message_end
                   └─ Channel-Versand (Blockantworten)
```

- `text_delta/events`: Modell-Stream-Ereignisse (können bei Modellen ohne Streaming spärlich sein).
- `chunker`: `EmbeddedBlockChunker` unter Anwendung von Mindest-/Höchstgrenzen und Trennpräferenz.
- `channel send`: tatsächlich ausgehende Nachrichten (Blockantworten).

**Steuerelemente** (alle unter `agents.defaults`, sofern nicht anders angegeben):

| Schlüssel                                                    | Werte/Form                                                            | Standardwert |
| ------------------------------------------------------------ | ----------------------------------------------------------------------- | ---------- |
| `blockStreamingDefault`                                      | `"on"` / `"off"`                                                        | `"off"`    |
| `blockStreamingBreak`                                        | `"text_end"` / `"message_end"`                                          | -          |
| `blockStreamingChunk`                                        | `{ minChars, maxChars, breakPreference? }`                              | -          |
| `blockStreamingCoalesce`                                     | `{ minChars?, maxChars?, idleMs? }` (gestreamte Blöcke vor dem Senden zusammenführen) | -          |
| `*.streaming.block.enabled` (Channel-Überschreibung)               | `true` / `false`, erzwingt Block-Streaming pro Channel (und pro Konto)  | -          |
| `*.textChunkLimit` (z. B. `channels.whatsapp.textChunkLimit`) | Zahl, feste Obergrenze                                                        | 4000       |
| `*.streaming.chunkMode`                                      | `"length"` / `"newline"`                                                | `"length"` |
| `channels.discord.maxLinesPerMessage`                        | Zahl, weiche Zeilenobergrenze, die hohe Antworten aufteilt, um ein Abschneiden in der UI zu vermeiden     | 17         |

`streaming.chunkMode: "newline"` teilt an Leerzeilen (Absatzgrenzen),
nicht an jedem Zeilenumbruch. Erst wenn der Text den Grenzwert überschreitet,
wird ersatzweise nach Länge aufgeteilt.

Gebündelte Channels schreiben diese Überschreibungen als
`channels.<id>.streaming.{chunkMode,block.enabled,block.coalesce}`. Die flachen
Schreibweisen `*.chunkMode` / `*.blockStreaming` / `*.blockStreamingCoalesce` sind
bei jedem gebündelten Channel veraltet: `openclaw doctor --fix` migriert sie in
die verschachtelte Form, und die Channel-Schemas lehnen sie ab. Konfigurationen externer SDK-Plugins,
die weiterhin die flachen Schreibweisen verwenden, funktionieren über einen veralteten
Fallback (mit einer Laufzeitwarnung) bis zum nächsten Release-Zyklus weiter.

**Grenzsemantik** für `blockStreamingBreak`:

- `text_end`: Streamt Blöcke, sobald der Chunker sie ausgibt; leert den Puffer bei jedem `text_end`.
- `message_end`: Wartet, bis die Assistentennachricht abgeschlossen ist, und leert dann die gepufferte
  Ausgabe. Verwendet weiterhin den Chunker, wenn der gepufferte Text `maxChars` überschreitet, sodass
  am Ende mehrere Abschnitte ausgegeben werden können.

### Medienübermittlung mit Block-Streaming

Streaming-Medien müssen strukturierte Nutzlastfelder wie `mediaUrl` oder
`mediaUrls` verwenden; gestreamter Text wird nicht als Anhangsbefehl ausgewertet. Wenn Block-
Streaming Medien vorzeitig sendet, merkt sich OpenClaw diese Übermittlung für den Turn. Wenn
die endgültige Assistenten-Nutzlast dieselbe Medien-URL wiederholt, entfernt die endgültige Übermittlung
das doppelte Medium, anstatt den Anhang erneut zu senden.

Exakt identische endgültige Nutzlasten werden unterdrückt. Wenn die endgültige Nutzlast
zusätzlichen Text um bereits gestreamte Medien enthält, sendet OpenClaw dennoch den
neuen Text, während das Medium nur einmal übermittelt wird. Dies verhindert doppelte Sprachnachrichten
oder Dateien in Channels wie Telegram.

## Aufteilungsalgorithmus (Unter-/Obergrenzen)

Die Blockaufteilung wird durch `EmbeddedBlockChunker` implementiert:

- **Untergrenze:** Keine Ausgabe, bis der Puffer >= `minChars` ist (außer bei erzwungener Ausgabe).
- **Obergrenze:** Trennungen vor `maxChars` bevorzugen; bei erzwungener Trennung an `maxChars` aufteilen.
- **Reihenfolge der Trennpräferenzen:** `paragraph` -> `newline` -> `sentence` ->
  Leerraum -> harte Trennung.
- **Code-Fences:** Niemals innerhalb von Fences trennen; bei erzwungener Trennung an `maxChars`
  die Fence schließen und erneut öffnen, damit das Markdown gültig bleibt.

`maxChars` wird auf `textChunkLimit` des Channels begrenzt, sodass die
Channel-spezifischen Obergrenzen nicht überschritten werden können.

## Zusammenführung (gestreamte Blöcke zusammenführen)

Wenn Block-Streaming aktiviert ist, kann OpenClaw **aufeinanderfolgende Block-
Abschnitte** vor dem Senden zusammenführen. Dadurch werden einzelne kurze Nachrichten reduziert, während
die Ausgabe weiterhin schrittweise erfolgt.

- Die Zusammenführung wartet vor dem Leeren auf **Inaktivitätsintervalle** (`idleMs`).
- Puffer werden durch `maxChars` begrenzt und geleert, wenn sie diesen Wert überschreiten.
- `minChars` verhindert, dass winzige Fragmente gesendet werden, bevor sich ausreichend Text angesammelt hat
  (beim abschließenden Leeren wird verbleibender Text immer gesendet).
- Das Verbindungszeichen wird aus `blockStreamingChunk.breakPreference` abgeleitet: `paragraph` ->
  `\n\n`, `newline` -> `\n`, `sentence` -> Leerzeichen.
- Channel-Überschreibungen sind über `*.streaming.block.coalesce` verfügbar (einschließlich
  kontospezifischer Konfigurationen).
- Discord, Signal und Slack verwenden standardmäßig `{ minChars: 1500, idleMs: 1000 }` für die Zusammenführung,
  sofern dies nicht überschrieben wird.

## Menschlich wirkende Pausen zwischen Blöcken

Wenn Block-Streaming aktiviert ist, wird nach dem ersten Block eine **zufällige Pause**
zwischen Blockantworten eingefügt, damit Antworten mit mehreren Nachrichten natürlicher wirken.

| `agents.defaults.humanDelay.mode` | Verhalten                |
| --------------------------------- | ----------------------- |
| `off` (Standardwert)                   | Keine Pause                |
| `natural`                         | Zufällige Pause von 800-2500ms |
| `custom`                          | `minMs`/`maxMs`         |

Pro Agent über `agents.entries.*.humanDelay` überschreibbar. Gilt nur für **Block-
antworten**, nicht für endgültige Antworten oder Tool-Zusammenfassungen.

## „Abschnitte streamen oder alles“

- **Abschnitte streamen:** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"`
  (während der Generierung ausgeben). Channels außer Telegram benötigen außerdem
  `*.streaming.block.enabled: true`.
- **Alles am Ende streamen:** `blockStreamingBreak: "message_end"` (Puffer
  einmal leeren, bei sehr langem Inhalt möglicherweise in mehreren Abschnitten).
- **Kein Block-Streaming:** `blockStreamingDefault: "off"` (nur endgültige Antwort).

Block-Streaming ist **deaktiviert, sofern** `*.streaming.block.enabled` nicht ausdrücklich
auf `true` gesetzt ist (Ausnahme: QQ Bot besitzt keine `streaming.block`-Schlüssel und streamt
Blockantworten, sofern `channels.qqbot.streaming.mode` nicht `"off"` ist). Channels können
eine Live-Vorschau streamen (`channels.<channel>.streaming.mode`), ohne Block-
antworten zu senden. Die Standardwerte für `blockStreaming*` befinden sich unter `agents.defaults`, nicht im
Konfigurationsstamm.

## Modi des Vorschau-Streamings

Kanonischer Schlüssel: `channels.<channel>.streaming` (verschachtelt `{ mode, ... }`; veraltete
boolesche/Zeichenketten-Schreibweisen auf oberster Ebene werden durch `openclaw doctor --fix` umgeschrieben).

| Modus       | Verhalten                                                              |
| ---------- | --------------------------------------------------------------------- |
| `off`      | Vorschau-Streaming deaktivieren                                             |
| `partial`  | Einzelne Vorschau wird durch den neuesten Text ersetzt                              |
| `block`    | Vorschau wird in aufgeteilten/angehängten Schritten aktualisiert                             |
| `progress` | Fortschritts-/Statusvorschau während der Generierung, endgültige Antwort nach Abschluss |

`streaming.mode: "block"` ist ein Vorschau-Streaming-Modus für bearbeitungsfähige
Channels wie Discord und Telegram; allein aktiviert er dort nicht die
Blockübermittlung des Channels. Verwenden Sie `streaming.block.enabled` für normale Blockantworten.
Microsoft Teams bildet die
Ausnahme: Es verfügt nicht über einen Blocktransport für Entwurfsvorschauen. Daher deaktiviert `streaming.mode:
"block"` natives Streaming vollständig, und die Antwort wird stattdessen als reguläre
Blockübermittlung gesendet, nicht als natives Teil-/Fortschritts-Streaming. Mattermost unterscheidet sich ebenfalls:
Im Modus `block` wechselt die Vorschau zwischen abgeschlossenem Text und
Tool-Aktivitätsblöcken, sodass frühere Blöcke als separate Beiträge sichtbar bleiben,
anstatt in einem bearbeitbaren Entwurf überschrieben zu werden.

### Channel-Zuordnung

| Channel    | `off` | `partial` | `block` | `progress`              |
| ---------- | ----- | --------- | ------- | ----------------------- |
| Telegram   | Ja   | Ja       | Ja     | bearbeitbarer Fortschrittsentwurf |
| Discord    | Ja   | Ja       | Ja     | bearbeitbarer Fortschrittsentwurf |
| Slack      | Ja   | Ja       | Ja     | Ja                     |
| Mattermost | Ja   | Ja       | Ja     | Ja                     |
| MS Teams   | Ja   | Ja       | Ja     | nativer Fortschrittsstream  |

Die Konfiguration der Vorschauabschnitte (`streaming.preview.chunk.*`, z. B. unter
`channels.discord.streaming` oder `channels.telegram.streaming`) verwendet standardmäßig
`minChars: 200`, `maxChars: 800` (begrenzt auf `textChunkLimit` des Channels) und
`breakPreference: "paragraph"`.

Nur Slack:

- `channels.slack.streaming.nativeTransport` schaltet Aufrufe der nativen Streaming-API von Slack
  (`chat.startStream`/`chat.appendStream`/`chat.stopStream`) um, wenn
  `channels.slack.streaming.mode="partial"` (Standardwert: `true`).
- Das native Streaming von Slack und der Thread-Status des Slack-Assistenten erfordern ein
  Antwort-Thread-Ziel. Direktnachrichten auf oberster Ebene zeigen diese threadartige Vorschau nicht an, können
  aber weiterhin Slack-Entwurfsvorschau-Beiträge und -Bearbeitungen verwenden.

### Migration veralteter Schlüssel

| Kanal   | Alte Schlüssel                                             | Status                                                                                                                                               |
| -------- | ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| Telegram | `streamMode`, skalarer/boolescher `streaming`                    | Wird von `openclaw doctor --fix` in `streaming.mode` umgeschrieben; wird zur Laufzeit nicht gelesen                                                                        |
| Discord  | `streamMode`, boolescher `streaming`                           | Wird von `openclaw doctor --fix` in `streaming.mode` umgeschrieben; wird zur Laufzeit nicht gelesen                                                                        |
| Slack    | `streamMode`; boolescher `streaming`; veralteter `nativeStreaming` | Wird von `openclaw doctor --fix` in `streaming.mode` (und bei den booleschen/veralteten Formen in `streaming.nativeTransport`) umgeschrieben; wird zur Laufzeit nicht gelesen         |
| Matrix   | skalarer/boolescher `streaming`                                  | Wird von `openclaw doctor --fix` in `streaming.mode` umgeschrieben (einschließlich des Matrix-Modus `"quiet"`); wird zur Laufzeit nicht gelesen                                    |
| Feishu   | boolescher `streaming`                                         | Wird von `openclaw doctor --fix` in `streaming.mode` umgeschrieben; wird zur Laufzeit nicht gelesen                                                                        |
| QQ Bot   | boolescher `streaming`; `streaming.c2cStreamApi`               | Wird von `openclaw doctor --fix` in `streaming.mode` (und bei den booleschen/`c2cStreamApi`-Formen in `streaming.nativeTransport`) umgeschrieben; wird zur Laufzeit nicht gelesen |

## Laufzeitverhalten

### Telegram

- Verwendet Vorschauaktualisierungen über `sendMessage` und `editMessageText` hinweg in Direktnachrichten und
  Gruppen/Themen; der endgültige Text bearbeitet die aktive Vorschau direkt. Die
  flüchtigen 30-sekündigen „Tippen“-Entwürfe von Telegram (`sendMessageDraft`) werden nicht für das
  Antwort-Streaming verwendet.
- Kurze anfängliche Vorschauen werden für eine bessere UX bei Push-Benachrichtigungen weiterhin entprellt, erscheinen
  jedoch nach einer begrenzten Verzögerung, damit aktive Ausführungen nicht visuell stumm bleiben.
- Lange endgültige Antworten verwenden die Vorschaunachricht für den ersten Abschnitt erneut und senden nur die
  verbleibenden Abschnitte.
- Der Modus `block` überführt die Vorschau bei
  `streaming.preview.chunk.maxChars` in eine neue Nachricht (Standardwert 800, begrenzt durch das
  Telegram-Bearbeitungslimit von 4096); in anderen Modi wächst eine einzelne Vorschau auf bis zu 4096 Zeichen.
- Der Modus `progress` hält den Werkzeugfortschritt in einem bearbeitbaren Statusentwurf fest, zeigt
  die Statusbezeichnung an, wenn das Antwort-Streaming aktiv, aber noch keine Werkzeugzeile
  verfügbar ist, löscht den Entwurf bei Abschluss und sendet die endgültige Antwort
  über die normale Zustellung.
- Falls die abschließende Bearbeitung fehlschlägt, bevor der vollständige Text bestätigt wurde, verwendet OpenClaw
  die normale endgültige Zustellung und bereinigt die veraltete Vorschau.
- Das Vorschau-Streaming wird übersprungen, wenn Telegram-Block-Streaming ausdrücklich
  aktiviert ist, um doppeltes Streaming zu vermeiden.
- `/reasoning stream` kann Überlegungen in eine vorübergehende Vorschau schreiben, die
  nach der endgültigen Zustellung gelöscht wird.
- Antworten auf ausgewählte Telegram-Zitate bilden eine Ausnahme: Wenn `replyToMode` nicht
  `"off"` ist und ausgewählter Zitattext vorhanden ist, überspringt OpenClaw für diesen Durchlauf den
  Antwort-Vorschaustream (die endgültige Antwort muss den nativen Pfad für Zitatantworten
  verwenden), sodass Vorschauzeilen zum Werkzeugfortschritt nicht dargestellt werden können. Antworten auf die aktuelle Nachricht
  ohne ausgewählten Zitattext behalten das Vorschau-Streaming bei. Weitere Informationen finden Sie in der
  [Dokumentation zum Telegram-Kanal](/de/channels/telegram).

### Discord

- Verwendet das Senden und Bearbeiten von Vorschaunachrichten.
- Der Modus `block` verwendet die Entwurfsaufteilung (`draftChunk`).
- Das Vorschau-Streaming wird übersprungen, wenn Discord-Block-Streaming ausdrücklich
  aktiviert ist.
- Der Modus `progress` hängt einen kleinen Aktivitätsnachweis `-#` (Anzahl der Überlegungen/Werkzeugaufrufe
  und verstrichene Zeit) an die endgültige Antwort an und löscht den Statusentwurf,
  sobald diese Antwort zugestellt wurde, sodass in stark frequentierten Kanälen kein verwaistes Werkzeugprotokoll
  über der Antwort verbleibt. Bei endgültigen Fehlerantworten bleibt der Entwurf als Aufzeichnung des fehlgeschlagenen
  Durchlaufs erhalten.
- Endgültige Medien-, Fehler- und explizite Antwort-Payloads verwerfen ausstehende Vorschauen,
  ohne einen neuen Entwurf zu übertragen, und verwenden anschließend die normale Zustellung.

### Slack

- `partial` kann natives Slack-Streaming (`chat.startStream`/`append`/`stop`)
  verwenden, sofern verfügbar.
- `block` verwendet Entwurfsvorschauen im Anhangsstil.
- `progress` verwendet Statusvorschautext und anschließend die endgültige Antwort.
- Direktnachrichten auf oberster Ebene ohne Antwort-Thread verwenden Entwurfsvorschau-Beiträge und deren Bearbeitung
  anstelle des nativen Slack-Streamings.
- Natives Streaming und Entwurfsvorschau-Streaming unterdrücken Blockantworten für diesen Durchlauf, sodass eine
  Slack-Antwort nur über einen einzigen Zustellungspfad gestreamt wird.
- Endgültige Medien-/Fehler-Payloads und abschließende Fortschrittsmeldungen erzeugen keine kurzlebigen Entwurfsnachrichten;
  nur endgültige Text-/Blockantworten, die die Vorschau bearbeiten können, übertragen ausstehenden
  Entwurfstext.

### Mattermost

- Im Modus `partial` werden Überlegungen und unvollständiger Antworttext in einen einzelnen Entwurfsvorschau-Beitrag
  gestreamt, der direkt abgeschlossen wird, sobald die endgültige Antwort sicher gesendet werden kann.
- Im Modus `progress` werden Überlegungen und Werkzeugaktivitäten in eine einzelne Statusvorschau
  gestreamt, die direkt abgeschlossen wird, sobald die endgültige Antwort sicher gesendet werden kann.
- Im Modus `block` wird zwischen Beiträgen mit vollständigem Text und Werkzeugaktivitäten gewechselt;
  parallele und aufeinanderfolgende Werkzeugaktualisierungen verwenden gemeinsam den aktuellen Werkzeugaktivitätsbeitrag.
- Es wird ersatzweise ein neuer endgültiger Beitrag gesendet, falls der Vorschaubeitrag gelöscht wurde oder
  zum Abschlusszeitpunkt anderweitig nicht verfügbar ist.
- Endgültige Medien-/Fehler-Payloads verwerfen ausstehende Vorschauaktualisierungen vor der normalen
  Zustellung, anstatt einen temporären Vorschaubeitrag zu übertragen.

### Matrix

- Entwurfsvorschauen werden direkt abgeschlossen, wenn der endgültige Text das Vorschauereignis
  wiederverwenden kann.
- Reine Medienantworten, Fehler und endgültige Antworten mit abweichendem Antwortziel verwerfen ausstehende Vorschauaktualisierungen
  vor der normalen Zustellung; eine bereits sichtbare veraltete Vorschau wird geschwärzt.

## Vorschauaktualisierungen zum Werkzeugfortschritt

Das Vorschau-Streaming kann auch Aktualisierungen zum **Werkzeugfortschritt** enthalten: kurze Statuszeilen
wie „Web wird durchsucht“, „Datei wird gelesen“ oder „Werkzeug wird aufgerufen“, die während der
Werkzeugausführung vor der endgültigen Antwort in derselben Vorschaunachricht erscheinen.
Im Codex-App-Server-Modus verwenden Codex-Präambel-/Kommentarnachrichten denselben
Vorschaupfad, sodass kurze Fortschrittshinweise wie „Ich prüfe gerade ...“ in den
bearbeitbaren Entwurf gestreamt werden können, ohne Teil der endgültigen Antwort zu werden. Dadurch bleiben
mehrstufige Werkzeugdurchläufe visuell aktiv, statt zwischen der ersten
Überlegungsvorschau und der endgültigen Antwort stumm zu bleiben.

Lange laufende Werkzeuge können vor ihrer Rückgabe typisierten Fortschritt ausgeben. Beispielsweise
startet `web_fetch` beim Beginn einen Fünf-Sekunden-Timer: Falls der Abruf weiterhin
aussteht, zeigt die Vorschau `Fetching page content...` an; falls der Abruf vorher abgeschlossen oder
abgebrochen wird, wird keine Fortschrittszeile ausgegeben. Das spätere endgültige Werkzeugergebnis
wird weiterhin normal an das Modell übermittelt.

Unterstützte Oberflächen:

- **Discord**, **Slack**, **Telegram** und **Matrix** streamen Werkzeugfortschritt und
  Codex-Präambelaktualisierungen standardmäßig in die laufende Vorschaubearbeitung, wenn das Vorschau-
  Streaming aktiv ist. Microsoft Teams verwendet in persönlichen Chats seinen nativen Fortschrittsstream.
- Telegram wird seit `v2026.4.22` mit aktivierten Vorschauaktualisierungen zum Werkzeugfortschritt
  ausgeliefert; sie aktiviert zu lassen, bewahrt dieses veröffentlichte Verhalten.
- **Mattermost** fasst Werkzeugaktivitäten in den Modi `partial` und
  `progress` in einem Vorschaubeitrag zusammen oder im Modus `block`
  in einem Werkzeugaktivitätsbeitrag zwischen Textblöcken (siehe oben).
- Bearbeitungen zum Werkzeugfortschritt folgen dem aktiven Vorschau-Streaming-Modus; sie werden
  übersprungen, wenn das Vorschau-Streaming `off` ist oder das Block-Streaming die
  Nachricht übernommen hat. Bei Telegram ist `streaming.mode: "off"` ausschließlich für endgültige Antworten bestimmt:
  Allgemeine Fortschrittsmeldungen werden ebenfalls unterdrückt, statt als eigenständige Statusnachrichten
  zugestellt zu werden, während Genehmigungsaufforderungen, Medien-Payloads und Fehler weiterhin
  normal weitergeleitet werden.
- Um das Vorschau-Streaming beizubehalten, aber Werkzeugfortschrittszeilen auszublenden, setzen Sie
  `streaming.preview.toolProgress` für diesen Kanal auf `false` (Standardwert
  `true`). Um Werkzeugfortschrittszeilen sichtbar zu lassen, aber Befehls-/Ausführungstext auszublenden,
  setzen Sie `streaming.preview.commandText` auf `"status"` oder
  `streaming.progress.commandText` auf `"status"`; der Standardwert ist `"raw"`, um
  veröffentlichtes Verhalten beizubehalten. Diese Richtlinie gilt gemeinsam für Entwurfs-/Fortschrittskanäle,
  die den kompakten Fortschrittsrenderer von OpenClaw verwenden, einschließlich Discord, Matrix,
  Microsoft Teams, Mattermost, Slack-Entwurfsvorschauen und Telegram. Um
  Vorschaubearbeitungen vollständig zu deaktivieren, setzen Sie `streaming.mode` auf `off`.

## Darstellung von Fortschrittsentwürfen

Entwürfe im Fortschrittsmodus (`streaming.progress.*`) sind begrenzt und pro
Kanal konfigurierbar:

| Schlüssel                         | Standardwert  | Verhalten                                                      |
| --------------------------------- | ------------- | -------------------------------------------------------------- |
| `streaming.progress.maxLines`     | `8`           | Maximale Anzahl kompakter Fortschrittszeilen unter der Entwurfsbezeichnung          |
| `streaming.progress.maxLineChars` | `120`         | Maximale Zeichenzahl pro kompakter Zeile vor der Kürzung (wortsensitiv) |
| `streaming.progress.label`        | `"auto"`      | Entwurfstitel; eine benutzerdefinierte Zeichenfolge oder `false`, um ihn auszublenden            |
| `streaming.progress.labels`       | integrierter Pool | Mögliche Bezeichnungen, die verwendet werden, wenn `label: "auto"`                     |

### Kommentar-Fortschrittsspur

Über den Werkzeugfortschritt hinaus kann der kompakte Fortschrittsrenderer im Entwurf
eine weitere Spur anzeigen:

- **`streaming.progress.commentary`** – stellt den **Kommentar** des Modells vor der Werkzeugausführung
  (eine kurze Erläuterung wie „Ich prüfe ... und dann ...“) zwischen den
  Werkzeugzeilen im Fortschrittsentwurf dar. Bei Discord und Telegram im Fortschrittsmodus
  liefert dieselbe Präambel die Statusüberschrift, selbst wenn diese optionale Spur
  deaktiviert ist; andere Kanäle behalten ihr bestehendes Fortschrittsverhalten bei. Siehe
  [Fortschrittsentwürfe](/de/concepts/progress-drafts#status-headline).

```json
{
  "channels": {
    "discord": {
      "streaming": { "mode": "progress", "progress": { "commentary": true } }
    }
  }
}
```

Fortschrittszeilen sichtbar lassen, aber unverarbeiteten Befehls-/Ausführungstext ausblenden:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "partial",
        "preview": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

Verwenden Sie dieselbe Struktur unter einem anderen Schlüssel für kompakte Fortschrittskanäle, beispielsweise
`channels.discord`, `channels.matrix`, `channels.msteams`,
`channels.mattermost` oder Slack-Entwurfsvorschauen. Für den Fortschrittsentwurfsmodus platzieren Sie
dieselbe Richtlinie unter `streaming.progress`:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

## Verwandte Themen

- [Refaktorierung des Nachrichtenlebenszyklus](/de/concepts/message-lifecycle-refactor) – gemeinsames Zieldesign für Vorschau, Bearbeitung, Streaming und Abschluss
- [Fortschrittsentwürfe](/de/concepts/progress-drafts) – sichtbare Nachrichten zu laufenden Arbeiten, die während langer Durchläufe aktualisiert werden
- [Nachrichten](/de/concepts/messages) – Nachrichtenlebenszyklus und Zustellung
- [Wiederholungsversuch](/de/concepts/retry) – Wiederholungsverhalten bei Zustellungsfehlern
- [Kanäle](/de/channels) – Streaming-Unterstützung pro Kanal
