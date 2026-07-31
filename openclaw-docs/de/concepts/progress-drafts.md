---
read_when:
    - Sichtbare Fortschrittsmeldungen für lange Chat-Antworten konfigurieren
    - Auswahl zwischen den Streaming-Modi „partial“, „block“ und „progress“
    - Erläuterung, wie OpenClaw eine einzelne Kanalnachricht aktualisiert, während die Arbeit läuft
    - Entwürfe für Fortschrittsmeldungen bei der Fehlerbehebung, eigenständige Fortschrittsmeldungen oder Fallback bei der Finalisierung
summary: 'Fortschrittsentwürfe: eine sichtbare Arbeitsfortschrittsmeldung, die während der Ausführung eines Agenten aktualisiert wird'
title: Fortschrittsentwürfe
x-i18n:
    generated_at: "2026-07-26T18:21:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ef66dd4d7a31c753f5faa0b88b83ec3760beecf3118cf8aae84f5e57652e809
    source_path: concepts/progress-drafts.md
    workflow: 16
---

Fortschrittsentwürfe verwandeln eine Kanalnachricht in eine Live-Statuszeile, während ein
Agent arbeitet, statt einen Stapel temporärer „wird noch bearbeitet“-Antworten zu erzeugen. Legen Sie
`channels.<channel>.streaming.mode: "progress"` fest, und OpenClaw erstellt die
Nachricht, sobald die eigentliche Arbeit beginnt, bearbeitet sie, während der Agent liest, plant, Tools
aufruft oder auf eine Genehmigung wartet, und verwandelt sie anschließend in die endgültige Antwort.

```text
Wird bearbeitet...
📖 aus docs/concepts/progress-drafts.md
🔎 Websuche: nach "discord edit message"
🛠️ Bash: Tests ausführen
```

<Note>
  Discord verwendet bereits standardmäßig `streaming.mode: "progress"`, wenn
  `channels.discord.streaming` nicht festgelegt ist, sodass Fortschrittsentwürfe
  dort ohne Konfiguration angezeigt werden. Für jeden anderen Kanal gilt standardmäßig `partial`
  oder `off`; die vollständige Tabelle der kanalspezifischen Standardwerte finden Sie unter
  [Streaming und Aufteilung](/de/concepts/streaming#channel-mapping).
</Note>

## Schnellstart

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
      },
    },
  },
}
```

Ab hier gelten folgende Standardwerte: eine Startverzögerung von 5 Sekunden, kompakte Fortschrittszeilen,
während sinnvolle Arbeit stattfindet, und die Unterdrückung der älteren eigenständigen Fortschrittsmeldungen
für diesen Durchlauf. Entwürfe mit unverarbeiteten Tool-Zeilen verwenden
automatisch eine Einwortbezeichnung; eine Statusüberschrift lässt diesen redundanten Titel weg,
sofern Sie nicht ausdrücklich einen konfigurieren.

Diese Seite behandelt die Bedienung von Fortschrittsentwürfen und ihre Konfigurationsoptionen. Die
vollständige Matrix der Streaming-Modi, kanalspezifische Laufzeithinweise und die Migration veralteter
Schlüssel finden Sie unter [Streaming und Aufteilung](/de/concepts/streaming).

## Was Benutzende sehen

| Teil             | Zweck                                                                                                     |
| ---------------- | --------------------------------------------------------------------------------------------------------- |
| Statusüberschrift | Bei Discord und Telegram die Präambel des Modells; Discord ergänzt einen Hilfsmodell-Platzhalter.         |
| Bezeichnung      | Optionale Start-/Statuszeile wie `Working`.                                                      |
| Fortschrittszeilen | Kompakte Aktualisierungen des Durchlaufs mit denselben Tool-Symbolen und demselben Detailformatierer wie `/verbose`. |

Bei unverarbeitetem Tool-Fortschritt erscheint die Bezeichnung, sobald der Agent mit sinnvoller Arbeit beginnt
und über die anfängliche Verzögerung hinaus beschäftigt bleibt.
Sie steht oben in der fortlaufenden Liste der Fortschrittszeilen und scrollt daher aus dem sichtbaren Bereich,
sobald genügend konkrete Arbeitszeilen erscheinen. Eine Statusüberschrift zeigt nur den
Klartextstatus des Agents, sofern nicht ausdrücklich eine Bezeichnung konfiguriert ist. Reine Textantworten
zeigen niemals einen Fortschrittsentwurf; eine Zeile erscheint nur bei tatsächlichen Arbeitsaktualisierungen,
zum Beispiel `🛠️ Bash: run tests`, `🔎 Web Search: for "discord edit message"`
oder `✍️ Write: to /tmp/file`.

Die endgültige Antwort ersetzt den Entwurf direkt, wenn der Kanal dies sicher
unterstützt; andernfalls sendet OpenClaw die endgültige Antwort über die normale Zustellung und
bereinigt den Entwurf oder aktualisiert ihn nicht weiter (siehe [Abschluss](#finalization)).

## Modus auswählen

`channels.<channel>.streaming.mode` steuert das sichtbare Verhalten während der Bearbeitung:

| Modus       | Am besten geeignet für                | Anzeige im Chat                                      |
| ----------- | ------------------------------------- | ---------------------------------------------------- |
| `off` | Ruhige Kanäle                        | Nur die endgültige Antwort.                          |
| `partial` | Beobachten des entstehenden Antworttexts | Ein Entwurf, der mit dem neuesten Antworttext bearbeitet wird. |
| `block` | Größere Blöcke der Antwortvorschau   | Eine Vorschau, die in größeren Blöcken aktualisiert oder ergänzt wird. |
| `progress` | Tool-intensive oder lange Durchläufe | Ein Statusentwurf, danach die endgültige Antwort.    |

Wählen Sie `progress`, wenn Benutzenden „was gerade passiert“ wichtiger ist, als
den Antworttext Token für Token entstehen zu sehen; `partial`, wenn der Antworttext selbst
das Fortschrittssignal ist; `block` für größere Vorschaublöcke. Bei Discord und
Telegram ist `streaming.mode: "block"` weiterhin Vorschau-Streaming und keine normale
Blockantwort-Zustellung – verwenden Sie dafür `streaming.block.enabled`.

## Bezeichnungen konfigurieren

Fortschrittsbezeichnungen befinden sich unter `channels.<channel>.streaming.progress`. Die standardmäßige
Bezeichnung für unverarbeitete Tool-Zeilen ist `"auto"`, wodurch die einfache integrierte Bezeichnung `Working`
verwendet wird. Eine Statusüberschrift blendet diese implizite Bezeichnung aus; legen Sie
`label: "auto"` ausdrücklich fest, wenn Sie auch darüber eine Bezeichnung wünschen:

```text
Wird bearbeitet
```

Eine feste Bezeichnung verwenden:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "Untersuchung läuft",
        },
      },
    },
  },
}
```

Einen eigenen Bezeichnungspool verwenden (wird bei `label: "auto"` weiterhin zufällig/anhand des Seeds ausgewählt):

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto",
          labels: ["Prüfung", "Lesen", "Testen", "Abschluss"],
        },
      },
    },
  },
}
```

Die Bezeichnung ausblenden und nur Fortschrittszeilen anzeigen:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: false,
        },
      },
    },
  },
}
```

## Fortschrittszeilen steuern

Fortschrittszeilen stammen aus tatsächlichen Durchlaufereignissen: Tool-Starts, Elementaktualisierungen, Aufgabenpläne,
Genehmigungen, Befehlsausgaben, Patch-Zusammenfassungen und ähnliche Agent-Aktivitäten.
Sie sind standardmäßig aktiviert (`progress.toolProgress`, Standardwert `true`).

Tools können außerdem typisierten Fortschritt ausgeben, während ein einzelner Aufruf noch läuft. So
aktualisiert ein langsamer Abruf oder eine langsame Suche den sichtbaren Entwurf, bevor das Tool
sein endgültiges Ergebnis zurückgibt. Die Fortschrittsaktualisierung ist ein partielles Tool-Ergebnis mit
leerem Modellinhalt und expliziten öffentlichen Kanalmetadaten:

```json
{
  "content": [],
  "progress": {
    "text": "Seiteninhalt wird abgerufen...",
    "visibility": "channel",
    "privacy": "public",
    "id": "web_fetch:fetching"
  }
}
```

OpenClaw rendert nur `progress.text` in der Fortschrittsoberfläche des Kanals. Das normale
Tool-Ergebnis trifft später weiterhin als `content`/`details` ein und ist der einzige Teil,
der an das Modell zurückgegeben wird.

Wenn Sie einem Tool Fortschritt hinzufügen, geben Sie eine kurze, allgemeine Nachricht aus und verzögern Sie sie,
bis der Vorgang lange genug aussteht, damit die Anzeige nützlich ist. `web_fetch`
erledigt genau dies mit einer Verzögerung von 5 Sekunden:

```typescript
const clearProgressTimer = scheduleToolProgress(
  onUpdate,
  { text: "Seiteninhalt wird abgerufen...", id: "web_fetch:fetching" },
  5_000,
  { signal },
);

try {
  return await runToolWork();
} finally {
  clearProgressTimer();
}
```

Schnelle Aufrufe zeigen keine Fortschrittszeile; lange Aufrufe zeigen eine, solange sie noch ausstehen;
abgebrochene Aufrufe löschen den Timer, bevor veralteter Fortschritt erscheinen kann. Fortschrittstext
ist ein öffentlicher UI-Seitenkanal und darf daher niemals Geheimnisse, unverarbeitete Argumente,
abgerufene Inhalte, Befehlsausgaben oder Seitentext enthalten.

### Detailmodus

OpenClaw verwendet denselben Formatierer für Fortschrittsentwürfe und `/verbose`:

```json5
{
  agents: {
    defaults: {
      toolProgressDetail: "explain", // explain | raw
    },
  },
}
```

`"explain"` ist der Standardwert und hält Entwürfe mit prägnanten Bezeichnungen stabil.
`"raw"` hängt den zugrunde liegenden Befehl an, sofern verfügbar. Dies ist beim
Debuggen nützlich, im Chat jedoch unübersichtlicher. Beispielsweise wird ein `node --check /tmp/app.js`-Aufruf
je nach Modus unterschiedlich gerendert:

| Modus                 | Fortschrittszeile              |
| --------------------- | ------------------------------ |
| `explain`    | `🛠️ check js syntax for /tmp/app.js`             |
| `raw`    | `🛠️ check js syntax for /tmp/app.js · node --check /tmp/app.js`             |

### Befehls-/Exec-Text

`streaming.progress.commandText` (Standardwert `"raw"`) steuert unabhängig vom obigen Detailmodus,
wie viele Befehlsdetails neben Exec-/Bash-Fortschrittszeilen angezeigt werden.
Setzen Sie den Wert auf `"status"`, um eine Tool-Fortschrittszeile sichtbar zu halten und
den Befehlstext vollständig auszublenden:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          commandText: "status",
        },
      },
    },
  },
}
```

### Kommentarspur

`streaming.progress.commentary` (Standardwert `false`) verschachtelt die
Kommentar-/Präambelerzählung des Modells vor dem Tool-Aufruf (💬, zum Beispiel „Ich prüfe ... und anschließend
...“) mit den Tool-Zeilen im Entwurf. Die
gemeinsame Konfigurationsstruktur für alle Kanäle finden Sie unter
[Streaming und Aufteilung](/de/concepts/streaming#commentary-progress-lane).

Wenn die Kommentarspur aktiviert ist, werden Präambeln nur als diese verschachtelten
💬-Zeilen gerendert; die darunterliegende Statusüberschrift wird ausgeblendet, damit die Spur ihre
dokumentierte Struktur beibehält.

### Statusüberschrift

Bei Discord und Telegram wird im Fortschrittsmodus die typisierte Präambel des Modells vor dem Tool-Aufruf
zur Statusüberschrift des Entwurfs, sofern sie verfügbar ist. Andere
Kanäle im Fortschrittsmodus behalten ihr bisheriges Statusverhalten bei. Die Überschrift ist
standardmäßig aktiviert und umgeht bei kurzen Durchläufen nicht die normale Aktivitätsschwelle;
durch Aktivieren von `streaming.progress.commentary` werden Präambeln stattdessen an die verschachtelte
Kommentarspur übergeben.

Wenn bei Discord ein Hilfsmodell für den Agent aufgelöst wird – entweder ein explizites
[`utilityModel`](/de/gateway/config-agents#utilitymodel) oder der vom primären
Provider deklarierte Standardwert für kleine Modelle (OpenAI → `gpt-5.6-luna`,
Anthropic → `claude-haiku-4-5`) –, liefert es einen kurzen Klartext-Platzhalter,
wenn das Modell keine Präambel ausgibt oder ungefähr 20 Sekunden lang still war
(die Überschrift von Telegram basiert derzeit ausschließlich auf der Präambel):

```text
Das Standardmodell in Ihrer Konfiguration wird aktualisiert und anschließend wird das Gateway neu gestartet,
damit die Änderung übernommen wird. Ein Aufruf zum Auflisten der Agents ist fehlgeschlagen und wird erneut versucht.
```

Die Erzählung ist standardmäßig aktiviert (`streaming.progress.narration`, Standardwert
`true`) und greift niemals auf das primäre Modell zurück: Sie wird nur mit einem expliziten
`utilityModel` oder einem vom Provider deklarierten Standardwert für den primären
Provider des Agents ausgeführt. Setzen Sie `utilityModel: ""`, um das Routing über das Hilfsmodell vollständig zu deaktivieren. Tool-Zeilen
werden darunter weiterhin gesammelt und erscheinen wieder, wenn beide Statusquellen aussetzen. Entwurfsbearbeitungen
warten weiterhin auf die normale Aktivitätsschwelle und eine tatsächliche
Textänderung. Dadurch werden kurzes Aufblitzen bei schnellen Durchläufen vermieden und die Anzahl der Bearbeitungen in stark frequentierten
Kanälen reduziert. Setzen Sie `narration: false`, um nur den Platzhalter des Hilfsmodells zu deaktivieren; Überschriften
aus Modellpräambeln bleiben aktiviert:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          narration: false,
        },
      },
    },
  },
}
```

Die Eingabe für die Erzählung ist begrenzt und bereinigt: Das Hilfsmodell erhält den
Text der eingehenden Anfrage sowie dieselben kompakten, bereinigten Tool-Zusammenfassungen, die der Entwurf
rendern würde – niemals unverarbeitete Befehlsausgaben oder Tool-Ergebnisse. Mit
`commandText: "status"` lässt die Eingabe für die Erzählung außerdem Exec-/Bash-Befehlstext weg,
entsprechend der Anzeige im Entwurf.

### Zeilenbegrenzungen

Begrenzen Sie die Anzahl der sichtbaren Zeilen (Standardwert 8):

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 4,
        },
      },
    },
  },
}
```

Fortschrittszeilen werden automatisch komprimiert, um den Umbruch der Chatblase während
der Entwurfsbearbeitung zu reduzieren. OpenClaw kürzt außerdem lange Zeilen, damit sie bei wiederholten Entwurfsbearbeitungen
nicht bei jeder Aktualisierung anders umbrechen. Das standardmäßige Budget pro Zeile beträgt 120
Zeichen; Fließtext wird an einer Wortgrenze abgeschnitten, während lange Details wie Pfade oder
unverarbeitete Befehle mit Auslassungspunkten in der Mitte gekürzt werden, sodass das Ende sichtbar bleibt.

Das Budget pro Zeile anpassen:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLineChars: 160,
        },
      },
    },
  },
}
```

### Umfangreiche Darstellung (Slack)

Slack kann Fortschrittszeilen als strukturierte Block-Kit-Felder statt als
Klartext rendern:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          render: "rich",
        },
      },
    },
  },
}
```

Bei der umfangreichen Darstellung wird zusammen mit den Block-Kit-Feldern stets derselbe Klartextinhalt
gesendet, sodass Clients, die die umfangreichere Struktur nicht rendern können, weiterhin den kompakten
Fortschrittstext anzeigen.

### Tool-/Aufgabenzeilen ausblenden

Den einzelnen Fortschrittsentwurf beibehalten, aber Tool- und Aufgabenzeilen ausblenden:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          toolProgress: false,
        },
      },
    },
  },
}
```

Mit `toolProgress: false` unterdrückt OpenClaw für diesen Durchlauf weiterhin die älteren eigenständigen
Meldungen zum Tool-Fortschritt – der Channel bleibt visuell ruhig, bis
die endgültige Antwort erscheint, mit Ausnahme des Labels, sofern eines konfiguriert ist.

## Channel-Verhalten

| Channel         | Fortschrittsübertragung                       | Hinweise                                                                                                                                                     |
| --------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Discord         | Eine Nachricht senden und anschließend bearbeiten. | Verwendet standardmäßig den Modus `progress`; die endgültige Antwort enthält einen `-#`-Aktivitätsbeleg, und der Statusentwurf wird gelöscht, nachdem die Antwort eingegangen ist. |
| Matrix          | Ein Ereignis senden und anschließend bearbeiten. | Die Streaming-Konfiguration auf Kontoebene steuert Entwürfe auf Kontoebene.                                                                                  |
| Microsoft Teams | Nativer Teams-Stream in persönlichen Chats.   | `streaming.mode: "block"` wird stattdessen der Teams-Blockzustellung zugeordnet.                                                                                     |
| Slack           | Nativer Stream oder bearbeitbarer Entwurfsbeitrag. | Erfordert ein Antwort-Thread-Ziel; DMs auf oberster Ebene ohne ein solches Ziel erhalten weiterhin Entwurfsvorschauen und Bearbeitungen.                     |
| Telegram        | Eine Nachricht senden und anschließend bearbeiten. | Wenn zwischen dem Fortschrittsentwurf und der Antwort eine Nachricht eingeht, wird der Entwurf darunter erneut veröffentlicht (zuerst neu veröffentlichen, dann den alten löschen), statt im Client einen Sprung beim Scrollen zu verursachen. |
| Mattermost      | Bearbeitbarer Entwurfsbeitrag.                 | Im Modus `block` wird zwischen abgeschlossenem Text und Beiträgen zu Tool-Aktivitäten gewechselt; in anderen Modi werden Tool-Aktivitäten in denselben entwurfsartigen Beitrag integriert. |

Channels ohne sichere Unterstützung für Bearbeitungen greifen auf Tippindikatoren oder
die ausschließliche Zustellung der endgültigen Antwort zurück. Unter [Streaming und Aufteilung](/de/concepts/streaming) finden Sie die
vollständige Aufschlüsselung des Laufzeitverhaltens je Channel.

## Abschluss

Wenn die endgültige Antwort bereit ist, versucht OpenClaw, den Chat übersichtlich zu halten:

- Im Modus `progress` auf Discord wird die endgültige Antwort als neue Nachricht gesendet,
  an die ein kleiner `-#`-Aktivitätsbeleg angehängt wird (zum Beispiel
  `-# 🧠 2 thoughts · 🛠️ 5 tool calls · ⏱️ 12s`); der Statusentwurf wird
  gelöscht, sobald diese Antwort zugestellt wurde. In stark frequentierten Channels bleibt oberhalb
  der Antwort kein verwaistes Tool-Protokoll zurück; bei endgültigen Fehlermeldungen bleibt der Entwurf als sichtbarer Nachweis
  des fehlgeschlagenen Durchlaufs erhalten.
- Wenn der Entwurf sicher in die endgültige Antwort umgewandelt werden kann (Modi `partial`/`block`),
  bearbeitet OpenClaw ihn direkt.
- Wenn der Channel natives Fortschritts-Streaming verwendet, schließt OpenClaw diesen
  Stream ab, sobald die native Übertragung den endgültigen Text akzeptiert.
- Andernfalls (Medien, eine Genehmigungsaufforderung, ein explizites Antwortziel, zu viele
  Blöcke oder ein fehlgeschlagenes Bearbeiten/Senden) sendet OpenClaw die endgültige Antwort über den
  normalen Zustellungsweg des Channels, anstatt den Entwurf zu überschreiben.

Dieser Fallback ist beabsichtigt: Eine neue endgültige Antwort zu senden ist besser, als Text zu verlieren,
eine Antwort dem falschen Thread zuzuordnen oder einen Entwurf mit Nutzdaten zu überschreiben, die der Channel
nicht sicher darstellen kann.

## Fehlerbehebung

**Ich sehe nur die endgültige Antwort.**

Prüfen Sie, ob `channels.<channel>.streaming.mode` für das Konto
oder den Channel, das bzw. der die Nachricht verarbeitet hat, auf `progress` gesetzt ist. Bei einigen Gruppen- oder Zitatantwortpfaden werden
Entwurfsvorschauen für einen Durchlauf deaktiviert, wenn der Channel nicht sicher die richtige
Nachricht bearbeiten kann.

**Ich sehe das Label, aber keine Tool-Zeilen.**

Prüfen Sie `streaming.progress.toolProgress`. Wenn es auf `false` gesetzt ist, behält OpenClaw das
Verhalten mit einem einzelnen Entwurf bei, blendet jedoch die Fortschrittszeilen für Tools und Aufgaben aus.

**Ich sehe eine neue endgültige Nachricht statt eines bearbeiteten Entwurfs.**

Dabei handelt es sich um den unter [Abschluss](#finalization) beschriebenen Sicherheits-Fallback. Er kann
bei Medienantworten, langen Antworten, expliziten Antwortzielen, alten Telegram-
Entwürfen, fehlenden Slack-Thread-Zielen, gelöschten Vorschaunachrichten oder einer fehlgeschlagenen
Finalisierung des nativen Streams auftreten.

**Ich sehe weiterhin eigenständige Fortschrittsmeldungen.**

Der Fortschrittsmodus unterdrückt standardmäßige eigenständige Meldungen zum Tool-Fortschritt, sobald ein
Entwurf aktiv ist. Wenn weiterhin eigenständige Meldungen erscheinen, prüfen Sie, ob der Durchlauf
tatsächlich den Modus `progress` und nicht `streaming.mode: "off"` oder einen Channel-
Pfad verwendet, der für diese Nachricht keinen Entwurf erstellen kann.

**Teams verhält sich anders als Discord oder Telegram.**

Microsoft Teams verwendet in persönlichen Chats einen nativen Stream anstelle der generischen
Vorschauübertragung durch Senden und Bearbeiten und ordnet `streaming.mode: "block"` der Teams-
Blockzustellung zu, da es keinen Blockmodus für Entwurfsvorschauen wie Discord und
Telegram bietet.

## Verwandte Themen

- [Streaming und Aufteilung](/de/concepts/streaming)
- [Nachrichten](/de/concepts/messages)
- [Channel-Konfiguration](/de/gateway/config-channels)
- [Discord](/de/channels/discord)
- [Matrix](/de/channels/matrix)
- [Microsoft Teams](/de/channels/msteams)
- [Slack](/de/channels/slack)
- [Telegram](/de/channels/telegram)
- [Mattermost](/de/channels/mattermost)
