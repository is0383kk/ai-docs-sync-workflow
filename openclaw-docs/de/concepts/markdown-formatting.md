---
read_when:
    - Sie ändern die Markdown-Formatierung oder Segmentierung für ausgehende Kanäle
    - Sie fügen einen neuen Kanalformatierer oder eine neue Stilzuordnung hinzu
    - Sie debuggen Formatierungsregressionen über mehrere Kanäle hinweg
summary: Markdown-Formatierungspipeline für ausgehende Kanäle
title: Markdown-Formatierung
x-i18n:
    generated_at: "2026-07-26T17:47:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9a35fd9a6386068e1e3bec73ec6e692f49239b468f42dd737f919b1c6a88e41
    source_path: concepts/markdown-formatting.md
    workflow: 16
---

OpenClaw konvertiert ausgehendes Markdown vor dem Rendern kanalspezifischer Ausgaben
in eine gemeinsame Zwischendarstellung (IR). Die IR enthält Klartext sowie
Stil-/Link-Spannen, sodass ein einziger Analyseschritt alle Kanäle bedient und die
Segmentierung Formatierungen niemals innerhalb einer Spanne trennt.

## Pipeline

1. **Markdown in IR analysieren** (`markdownToIR`) – Klartext plus Stilspannen
   (fett, kursiv, durchgestrichen, Code, Codeblock, Spoiler, Blockzitat,
   Überschrift 1–6) und Link-Spannen. Offsets sind UTF-16-Codeeinheiten, sodass
   Signal-Stilbereiche direkt mit dessen API übereinstimmen. Tabellen werden nur
   analysiert, wenn für den Kanal ein Tabellenmodus aktiviert ist.
2. **IR segmentieren** (`chunkMarkdownIR` / `renderMarkdownIRChunksWithinLimit`)
   - Die Aufteilung erfolgt vor dem Rendern anhand des IR-Texts, sodass Inline-Stile und
     Links für jedes Segment zugeschnitten werden, statt an einer Grenze unterbrochen zu werden.
3. **Pro Kanal rendern** (`renderMarkdownWithMarkers`) – eine Zuordnung von Stilmarkierungen
   wandelt Spannen in das native Markup des Kanals um.

| Kanal                                                            | Renderer                                                                             | Hinweise                                                                                 |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Slack                                                            | mrkdwn-Tokens (`*bold*`, `_italic_`, `` `code` ``, Code-Fences)                      | Links werden zu `<url\|label>`; automatische Verlinkung ist während der Analyse deaktiviert, um doppelte Links zu vermeiden |
| Telegram                                                         | HTML-Tags (`<b>`, `<i>`, `<s>`, `<code>`, `<pre><code>`, `<a href>`, `<tg-spoiler>`) | Unterstützt außerdem Rich-Message-Tabellen und -Überschriften (`<h1>`–`<h6>`), wenn `richMessages` aktiviert ist |
| Signal                                                           | Klartext + `text-style`-Bereiche                                                     | Links werden als `label (url)` gerendert, wenn sich die Beschriftung von der URL unterscheidet |
| Discord, WhatsApp, iMessage, Microsoft Teams und andere Kanäle   | Klartext                                                                             | Keine IR-basierte Formatierung; die Markdown-Tabellenkonvertierung erfolgt weiterhin über `convertMarkdownTables` |

## IR-Beispiel

Eingabe-Markdown:

```markdown
Hallo **Welt** – siehe [Dokumentation](https://docs.openclaw.ai).
```

IR (schematisch):

```json
{
  "text": "Hallo Welt – siehe Dokumentation.",
  "styles": [{ "start": 6, "end": 10, "style": "bold" }],
  "links": [{ "start": 19, "end": 32, "href": "https://docs.openclaw.ai" }]
}
```

## Tabellenverarbeitung

`markdown.tables` steuert pro Kanal und optional pro Konto, wie ein Kanal
Markdown-Tabellen konvertiert:

| Modus     | Verhalten                                                                            |
| --------- | ------------------------------------------------------------------------------------ |
| `code`    | Als ausgerichtete ASCII-Tabelle in einem Codeblock rendern (Fallback-Standard)        |
| `bullets` | Jede Zeile in `label: value`-Aufzählungspunkte konvertieren                          |
| `block`   | Native Tabellen beibehalten, sofern der Transport sie unterstützt; andernfalls Fallback auf `code` |
| `off`     | Tabellenanalyse deaktivieren; unverarbeiteter Tabellentext wird unverändert weitergegeben |

Plugin-Standardeinstellungen pro Kanal: Signal, WhatsApp und Matrix verwenden standardmäßig
`bullets`; Mattermost verwendet standardmäßig `off`; Telegram verwendet standardmäßig `block` (was
zu `code` aufgelöst wird, sofern für das Konto nicht `richMessages` aktiviert ist). Jeder
Kanal ohne explizite Plugin-Standardeinstellung verwendet als Fallback `code`.

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## Segmentierungsregeln

- Segmentgrenzen stammen aus den Kanaladaptern bzw. der Konfiguration und gelten für
  IR-Text, nicht für die gerenderte Ausgabe.
- Codeblöcke mit Fences werden als einzelner Block mit abschließendem Zeilenumbruch
  beibehalten, damit Kanäle die schließende Fence korrekt rendern.
- Listen- und Blockzitatpräfixe sind Teil des IR-Texts, sodass die Segmentierung
  niemals innerhalb eines Präfixes trennt.
- Inline-Stile werden niemals über mehrere Segmente getrennt; der Renderer öffnet
  einen offenen Stil am Anfang des nächsten Segments erneut.

Weitere Informationen zu Segmentgrenzen und zum Zustellungsverhalten über verschiedene
Kanäle hinweg finden Sie unter [Streaming und Segmentierung](/concepts/streaming).

## Link-Richtlinie

- **Slack:** `[label](url)` -> `<url|label>`; reine URLs bleiben unverändert.
- **Telegram:** `[label](url)` -> `<a href="url">label</a>` (HTML-Analysemodus).
- **Signal:** `[label](url)` -> `label (url)`, sofern die Beschriftung nicht bereits
  mit der URL übereinstimmt.

## Spoiler

Spoiler-Markierungen (`||spoiler||`) werden für Signal analysiert (Zuordnung zu
`SPOILER`-Stilbereichen) und für Telegram (Zuordnung zu `<tg-spoiler>`). Andere Kanäle behandeln
`||...||` als Klartext.

## Kanalformatierer hinzufügen oder aktualisieren

1. **Einmal analysieren** mit `markdownToIR(...)` und dabei kanalgeeignete
   Optionen übergeben (`autolink`, `headingStyle`, `blockquotePrefix`, `tableMode`).
2. **Rendern** mit `renderMarkdownWithMarkers(...)` und einer Zuordnung von Stilmarkierungen (oder
   benutzerdefinierter Stilbereichslogik für Transporte wie Signal).
3. **Segmentieren** mit `chunkMarkdownIR(...)` oder
   `renderMarkdownIRChunksWithinLimit(...)`, bevor jedes Segment gerendert wird.
4. **Adapter anbinden**, sodass der neue Segmentierer und Renderer über den
   ausgehenden Sendepfad aufgerufen werden.
5. **Testen** mit Formatierungstests sowie einem Test der ausgehenden Zustellung, wenn der
   Kanal segmentiert.

## Häufige Fallstricke

- Slack-Tokens in spitzen Klammern (`<@U123>`, `<#C123>`, `<https://...>`) müssen
  die Maskierung überstehen; unverarbeitetes HTML muss weiterhin sicher maskiert werden.
- Telegram-HTML erfordert die Maskierung von Text außerhalb der Tags, um fehlerhaftes Markup zu vermeiden.
- Signal-Stilbereiche verwenden UTF-16-Offsets, nicht Codepunkt-Offsets.
- Behalten Sie abschließende Zeilenumbrüche bei Codeblöcken mit Fences bei, damit die
  schließende Markierung in einer eigenen Zeile steht.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Streaming und Segmentierung" href="/de/concepts/streaming" icon="bars-staggered">
    Verhalten beim ausgehenden Streaming, Segmentgrenzen und kanalspezifische Zustellung.
  </Card>
  <Card title="System-Prompt" href="/de/concepts/system-prompt" icon="message-lines">
    Was das Modell vor der Unterhaltung sieht, einschließlich eingebundener Workspace-Dateien.
  </Card>
</CardGroup>
