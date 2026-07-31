---
read_when:
    - Hinzufügen oder Ändern der Darstellung von Nachrichtenkarten, Diagrammen, Tabellen, Schaltflächen oder Auswahlfeldern
    - Erstellen eines Kanal-Plugins mit Unterstützung für umfangreiche ausgehende Nachrichten
    - Darstellung des Nachrichten-Tools oder Zustellungsfunktionen ändern
    - Debuggen providerspezifischer Regressionen bei der Darstellung von Karten, Blöcken und Komponenten
summary: Semantische Nachrichtencards, Diagramme, Tabellen, Steuerelemente, Fallback-Text und Zustellungshinweise für Kanal-Plugins
title: Nachrichtendarstellung
x-i18n:
    generated_at: "2026-07-26T18:37:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

Die Nachrichtendarstellung ist der gemeinsame Vertrag von OpenClaw für umfangreiche Benutzeroberflächen ausgehender Chatnachrichten.
Damit können Agenten, CLI-Befehle, Genehmigungsabläufe und Plugins die
Nachrichtenabsicht einmal beschreiben, während jedes Channel-Plugin die bestmögliche native Form rendert.

Verwenden Sie die Darstellung für portable Nachrichtenoberflächen: Textabschnitte, kurze Kontext-/Fußzeilentexte,
Trennlinien, Diagramme, Tabellen, Schaltflächen, Auswahlmenüs sowie Kartentitel und -tonalität.

Fügen Sie dem gemeinsamen Nachrichtentool keine neuen Provider-nativen Felder wie Discord `components`, Slack
`blocks`, Telegram `buttons`, Teams `card` oder Feishu `card` hinzu. Diese sind Ausgaben des Renderers,
für die das Channel-Plugin zuständig ist.

## Vertrag

Plugin-Autoren importieren den öffentlichen Vertrag aus:

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Struktur:

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** Veralteter Callback-Wert. Bevorzugen Sie action für neue Steuerelemente. */
  value?: string;
  /** @deprecated Verwenden Sie eine Aktion mit dem Typ "url". */
  url?: string;
  /** @deprecated Verwenden Sie eine Aktion mit dem Typ "web-app". */
  webApp?: { url: string };
  /** @deprecated Verwenden Sie eine Aktion mit dem Typ "web-app". */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** Veralteter Callback-Wert. Bevorzugen Sie action für neue Steuerelemente. */
  value?: string;
};

type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

Semantik von Schaltflächen:

- `action.type: "command"` führt einen nativen Slash-Befehl über den Befehlspfad
  des Kerns aus. Verwenden Sie dies für integrierte Befehlsschaltflächen und Menüs.
- `action.type: "callback"` überträgt undurchsichtige Plugin-Daten über den
  Interaktionspfad des Channels. Channel-Plugins dürfen Callback-Daten nicht als Slash-
  Befehle neu interpretieren.
- `action.type: "approval"` identifiziert eine dauerhafte Genehmigung durch den Bediener, deren
  explizite Art `exec` oder `plugin` und die angeforderte Entscheidung. Channel-Plugins
  codieren diese Aktion in einen transportspezifischen privaten Callback und lösen sie über
  den Genehmigungsdienst auf; sie dürfen weder `/approve`-Befehlstext analysieren noch
  die Art aus der ID ableiten.
- `action.type: "question"` identifiziert eine Auswahl für eine aktive, zur Laufzeit erstellte
  `ask_user`-Frage. Wie `approval` ist dies eine OpenClaw-Laufzeitaktion;
  Agenten und Plugins dürfen keine Frage-IDs erzeugen. Telegram, Discord und
  Slack ordnen sie transportspezifischen privaten nativen Callbacks zu und lösen die Auswahl
  über das Gateway auf. Sobald die Frage beantwortet wurde, abgelaufen ist oder
  abgebrochen wurde, bearbeiten diese Channels die zugestellte Nachricht, entfernen ihre Aktionen
  und hängen den Endstatus an. WhatsApp, Signal und iMessage rendern bis zu
  vier Einfachauswahlen als Reaktionen von `1️⃣` bis `4️⃣`. Andere Frageformen
  werden auf Beschriftungstext zurückgestuft, und der Benutzer kann mit einer Nur-Text-
  Antwort antworten.
- `action.type: "url"` öffnet einen normalen Link.
- `action.type: "web-app"` startet eine Channel-native Web-App. Legen Sie `url` für eine
  URL-basierte App oder `widgetId` für ein von OpenClaw gehostetes Widget fest, dessen Startmechanismen
  dem Channel unterliegen; mindestens eines davon ist erforderlich. Sind beide
  vorhanden, kann ein Channel den nativen Start des gehosteten Widgets bevorzugen und die URL
  verwenden, wenn dieser Mechanismus nicht verfügbar ist.
- `value` ist der veraltete undurchsichtige Callback-Wert. Neue Steuerelemente sollten `action`
  verwenden, damit Channel-Plugins Befehle und Callbacks zuordnen können, ohne anhand des Texts zu raten.
- `url`, `webApp` und `web_app` werden weiterhin als veraltete Eingaben an der Schnittstellengrenze akzeptiert.
  Normalisierer bewahren diese Felder, damit Renderer zwischen ausgelieferter veralteter
  Semantik und expliziten typisierten Aktionen unterscheiden können. Neue Erzeuger sollten `action` verwenden.
- `label` ist erforderlich und wird auch beim Text-Fallback verwendet.
- `style` hat empfehlenden Charakter. Renderer sollten nicht unterstützte Stile einem sicheren
  Standard zuordnen, statt das Senden fehlschlagen zu lassen.
- `priority` ist optional. Wenn ein Channel Aktionslimits angibt und Steuerelemente
  verworfen werden müssen, behält der Kern zuerst Schaltflächen mit höherer Priorität bei und bewahrt
  die ursprüngliche Reihenfolge von Schaltflächen mit gleicher Priorität. Wenn alle Steuerelemente passen, bleibt die
  vom Autor festgelegte Reihenfolge erhalten.
- `disabled` ist optional. Channels müssen dies mit `supportsDisabled` aktivieren; andernfalls
  stuft der Kern das deaktivierte Steuerelement zu nicht interaktivem Fallback-Text zurück. Eine
  deaktivierte Schaltfläche wird im Fallback-Text immer nur mit ihrer Beschriftung gerendert, selbst wenn sie
  eine `command`-Aktion enthält.
- `reusable` ist optional. Channels, die wiederverwendbare native Callbacks unterstützen, können
  die Aktion nach einer erfolgreichen Interaktion weiterhin verfügbar halten. Verwenden Sie dies für
  wiederholbare oder idempotente Aktionen wie Aktualisieren, Prüfen oder weitere Details;
  lassen Sie es für normale einmalige Genehmigungen und destruktive Aktionen ungesetzt.

Semantik von Auswahlmenüs:

- `options[].action` akzeptiert nur `command` oder `callback`; Genehmigungs- und Linkaktionen sind ausschließlich für Schaltflächen vorgesehen.
- `options[].value` ist der veraltete ausgewählte Anwendungswert.
- `placeholder` hat empfehlenden Charakter und kann von Channels ohne native
  Auswahlunterstützung ignoriert werden.
- Wenn ein Channel keine Auswahlmenüs unterstützt, listet der Fallback-Text die Beschriftungen auf.

Semantik von Diagrammen:

- `pie` erfordert positive Segmentwerte.
- `bar`, `area` und `line` verwenden ein geordnetes `categories`-Array. Jede Datenreihe
  stellt für jede Kategorie genau einen endlichen Wert in derselben Reihenfolge bereit.
- Kategoriebeschriftungen und Namen von Datenreihen müssen eindeutig sein. Ungültige oder unvollständige Diagramm-
  blöcke werden während der Normalisierung verworfen, statt Daten stillschweigend zu ändern.
- Das native Rendern von Diagrammen wird über `presentationCapabilities.charts` aktiviert.
  Andere Channels erhalten Diagrammtitel, Achsen, Kategorien, Datenreihen und Werte
  als deterministischen Text. Dies ist zugleich das Barrierefreiheits-Fallback.

Semantik von Tabellen:

- `caption` ist eine erforderliche kurze Überschrift. `headers` muss mindestens eine
  eindeutige, nicht leere Spaltenbeschriftung enthalten.
- `rows` muss mindestens eine Zeile enthalten. Jede Zeile muss genau eine Zelle pro
  Überschrift besitzen, und jede Zelle muss eine nicht leere Zeichenfolge oder eine endliche Zahl sein.
- `rowHeaderColumnIndex` ist ein optionaler nullbasierter Index, der die Spalte
  identifiziert, deren Zellen von nativen Renderern als Zeilenüberschriften bereitgestellt werden sollen.
- Die Tabellennormalisierung erfolgt atomar. Eine ungültige Beschriftung, Überschrift, Zeilenbreite, Zelle
  oder ein ungültiger Zeilenüberschriftenindex führt dazu, dass der Tabellenblock verworfen wird, statt seine Daten zu kürzen oder zu reparieren.
- Das native Rendern von Tabellen wird über `presentationCapabilities.tables` aktiviert.
  Andere Channels erhalten die Beschriftung und jede Zeile als deterministischen linearen
  Text, wobei interne Leerzeichen zusammengefasst werden:

  ```text
  Offene Pipeline (Tabelle)
  - Konto: Acme; Phase: Gewonnen; ARR: 125000
  - Konto: Globex; Phase: Prüfung; ARR: 82000
  ```

Es gibt keinen separaten `report`-Diskriminator. Stellen Sie einen Bericht aus `title`,
`tone`, `text`, `context`, `chart`, `table` und Aktionsblöcken zusammen. Dadurch bleibt jeder
Block unabhängig renderbar, und der vollständige Bericht erhält dasselbe
deterministische Text-Fallback.

## Beispiele für Erzeuger

Einfache Karte:

```json
{
  "title": "Bereitstellung genehmigen",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "Canary kann jetzt hochgestuft werden." },
    { "type": "context", "text": "Build 1234, Staging erfolgreich." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Genehmigen",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "Ablehnen",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

Schaltfläche mit ausschließlich einem URL-Link:

```json
{
  "blocks": [
    { "type": "text", "text": "Die Versionshinweise sind verfügbar." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Hinweise öffnen",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Schaltfläche für eine Telegram Mini App:

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Starten",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

Auswahlmenü:

```json
{
  "title": "Umgebung auswählen",
  "blocks": [
    {
      "type": "select",
      "placeholder": "Umgebung",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "Produktion", "value": "env:prod" }
      ]
    }
  ]
}
```

Diagramm:

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "Quartalsumsatz",
      "categories": ["Q1", "Q2", "Q3"],
      "series": [
        { "name": "Produkt", "values": [120, 145, 138] },
        { "name": "Dienstleistungen", "values": [80, 95, 104] }
      ],
      "xLabel": "Quartal",
      "yLabel": "Umsatz"
    }
  ]
}
```

Tabellenbericht:

```json
{
  "title": "Pipeline-Bericht",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "Aktuelle Verkaufschancen nach Phase." },
    {
      "type": "table",
      "caption": "Offene Pipeline",
      "headers": ["Konto", "Phase", "ARR"],
      "rows": [
        ["Acme", "Gewonnen", 125000],
        ["Globex", "Prüfung", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "Aus dem CRM-Snapshot aktualisiert." }
  ]
}
```

Senden per CLI:

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "Bereitstellung genehmigen" \
  --presentation '{"title":"Bereitstellung genehmigen","tone":"warning","blocks":[{"type":"text","text":"Canary ist bereit."},{"type":"buttons","buttons":[{"label":"Genehmigen","value":"deploy:approve","style":"success"},{"label":"Ablehnen","value":"deploy:decline","style":"danger"}]}]}'
```

Angeheftete Zustellung:

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "Topic opened" \
  --pin
```

Angeheftete Zustellung mit explizitem JSON:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## Renderer-Vertrag

Channel-Plugins deklarieren die Render-Unterstützung in ihrem ausgehenden Adapter:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

Boolesche Fähigkeiten beschreiben, welche Elemente der Renderer interaktiv umsetzen kann. Optionale
`limits` beschreiben die generische Hülle, die der Kern anpassen kann, bevor er den
Renderer aufruft:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};
```

Der Kern wendet vor dem Rendern generische Begrenzungen auf semantische Steuerelemente an. Renderer
bleiben dennoch für die abschließende providerspezifische Validierung und Kürzung hinsichtlich nativer Blockanzahl,
Kartengröße, URL-Begrenzungen und Provider-Besonderheiten verantwortlich, die sich nicht im
generischen Vertrag ausdrücken lassen. Wenn durch Begrenzungen alle Steuerelemente eines Blocks entfernt werden, behält der Kern
die Beschriftungen als nicht interaktiven Kontexttext bei, damit die zugestellte Nachricht weiterhin eine
sichtbare Ausweichdarstellung enthält.

## Render-Ablauf im Kern

Im kanonischen ausgehenden Pfad, den die CLI und Standard-Nachrichtenaktionen verwenden, führt der Kern Folgendes aus:

1. Normalisiert die Präsentationsnutzlast.
2. Löst den ausgehenden Adapter des Ziel-Channels auf.
3. Liest `presentationCapabilities`.
4. Wendet generische Fähigkeitsbegrenzungen wie Anzahl der Aktionen, Länge der Beschriftungen und
   Anzahl der Auswahloptionen an, wenn der Adapter diese angibt. Diagramm- und Tabellenblöcke
   werden zu deterministischem Text, sofern der Adapter nicht ausdrücklich
   `charts: true` beziehungsweise `tables: true` angibt.
5. Ruft `renderPresentation` auf, wenn der Adapter die Nutzlast rendern kann.
6. Weicht auf konservativen Text aus, wenn der Adapter fehlt oder nicht rendern kann.
7. Sendet die resultierende Nutzlast über den normalen Zustellungspfad des Channels.
8. Wendet Zustellungsmetadaten wie `delivery.pin` nach der ersten erfolgreich
   gesendeten Nachricht an.

Channel-lokale Antwort- oder Vorschaupfade, die `ReplyPayload` direkt verarbeiten,
müssen entweder in diesen kanonischen Pfad eintreten oder dieselbe Präsentations-Ausweichdarstellung
erzeugen, bevor sie die Nutzlast auf reinen Text beziehungsweise Medien projizieren.

Der Kern ist für das Ausweichverhalten verantwortlich, damit Produzenten Channel-unabhängig bleiben können. Channel-
Plugins sind für natives Rendering und die Interaktionsverarbeitung verantwortlich.

## Regeln für eingeschränkte Darstellung

Die Präsentation muss auch auf eingeschränkten Channels sicher versendet werden können.

Der Ausweichtext enthält:

- `title` als erste Zeile
- `text`-Blöcke als normale Absätze
- `context`-Blöcke als kompakte Kontextzeilen
- `divider`-Blöcke als visuelle Trennlinie
- Schaltflächenbeschriftungen, einschließlich URLs für Link-Schaltflächen
- Beschriftungen der Auswahloptionen
- Diagrammtitel, Typ, Achsen, Kategorien, Datenreihen und Werte
- Tabellenüberschrift, Spaltenüberschriften und jeden Zeilenwert

### Sichtbarkeit von Schaltflächenwerten in der Ausweichdarstellung

Wenn ein Channel keine interaktiven Steuerelemente rendern kann, werden Schaltflächen- und Auswahlwerte
als reiner Text dargestellt. Das Ausweichverhalten erhält die Bedienbarkeit und
hält gleichzeitig nicht transparente Callback-Daten privat:

- **Aktionen vom Typ `command`** werden als `` label: `command` `` dargestellt, damit Benutzer
  den Befehl kopieren und manuell im Eingabefeld des Channels ausführen können.
- **Aktionen vom Typ `callback`** und veraltete **`value`**-Felder werden
  nur mit ihrer Beschriftung dargestellt. Der nicht transparente Callback-Wert wird im Ausweichtext nicht offengelegt.
- **Aktionen vom Typ `approval`** werden nur mit ihrer Beschriftung dargestellt. Genehmigungs-IDs und Entscheidungen sind
  Transportdaten und werden weder über generische Skalar-Hilfsfunktionen noch über Ausweichtext
  offengelegt.
- **`url`-Aktionen**, URL-gestützte **`web-app`-Aktionen** und veraltete **`url` /
  `webApp` / `web_app`**-Eingaben stellen den URL-Text neben der Schaltflächenbeschriftung dar,
  da die URL für Benutzer sichtbar ist. Aktionen ausschließlich für gehostete Widgets werden auf
  Channels ohne nativen Widget-Start nur mit ihrer Beschriftung dargestellt.
- **Auswahloptionen** werden nur mit ihrer Beschriftung dargestellt. Der zugrunde liegende Optionswert wird
  im Ausweichtext nicht offengelegt.

Channel-Adapter, die in ihrer Ausweich-Benutzeroberfläche Hinweise zur manuellen Befehlsausführung hinzufügen (z. B.
Anweisungen für Feishu-Dokumentkommentare), müssen die Prüfung auf vorhandene Befehle
aus denselben Präsentationsblöcken ableiten, die der Ausweich-Renderer verwendet, damit der
Hinweistext nur erscheint, wenn tatsächlich ein manueller Befehl angezeigt wird.

Nicht unterstützte native Steuerelemente sollten auf eine einfachere Darstellung zurückfallen, statt den gesamten Versand fehlschlagen zu lassen.
Beispiele:

- Telegram sendet bei deaktivierten Inline-Schaltflächen einen Text als Ausweichdarstellung.
- Ein Channel ohne Auswahlunterstützung führt die Auswahloptionen als Text auf.
- Ein Channel ohne native Diagrammunterstützung führt die Diagrammdaten als Text auf.
- Ein Channel ohne native Tabellenunterstützung führt jede Tabellenzeile als Text auf.
- Eine reine URL-Schaltfläche wird entweder zu einer nativen Link-Schaltfläche oder einer URL-Zeile als Ausweichdarstellung.
- Fehler beim optionalen Anheften lassen die zugestellte Nachricht nicht fehlschlagen.

Die wichtigste Ausnahme ist `delivery.pin.required: true`; wenn das Anheften als
erforderlich angefordert wird und der Channel die gesendete Nachricht nicht anheften kann, meldet die Zustellung einen Fehler.

## Provider-Zuordnung

Aktuelle gebündelte Renderer:

| Channel         | Natives Render-Ziel                       | Hinweise                                                                                                                                                                                                          |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | Komponenten und Komponentencontainer      | Behält veraltetes `channelData.discord.components` für bestehende Provider-native Nutzlastproduzenten bei, neue gemeinsame Sendevorgänge sollten jedoch `presentation` verwenden.                                            |
| Feishu          | Interaktive Karten                        | Der Kartenkopf kann `title` verwenden; der Textkörper vermeidet eine Wiederholung dieses Titels.                                                                                                       |
| Matrix          | Textausweichdarstellung plus strukturiertes Ereignisfeld | Schaltflächen/Auswahlfelder werden als unterstützt angegeben, aber jeder Block wird derzeit als `renderMessagePresentationFallbackText`-Ausgabe in einem `com.openclaw.presentation`-Ereignisfeld dargestellt, nicht als native interaktive Widgets. |
| Mattermost      | Text plus interaktive Eigenschaften       | Auswahlfelder und Trennlinien werden nicht unterstützt; diese Blöcke werden als Text dargestellt.                                                                                                                 |
| Microsoft Teams | Adaptive Cards                            | Reiner `message`-Text wird zusammen mit der Karte eingefügt, wenn beides bereitgestellt wird. Auswahlfelder, Stile und der deaktivierte Zustand werden nicht unterstützt.                                 |
| Slack           | Block Kit                                 | Rendert `chart` als natives `data_visualization` und `table` als natives `data_table`; behält veraltetes `channelData.slack.blocks` bei, neue gemeinsame Sendevorgänge sollten jedoch `presentation` verwenden. |
| Telegram        | Text plus Inline-Tastaturen               | Schaltflächen/Auswahlfelder erfordern Inline-Schaltflächen-Unterstützung für die Zieloberfläche; andernfalls wird die Textausweichdarstellung verwendet.                                                           |
| Reine Text-Channels | Textausweichdarstellung                | Channels ohne Renderer erhalten weiterhin eine lesbare Ausgabe.                                                                                                                                                   |

Die Kompatibilität mit Provider-nativen Nutzlasten ist eine Übergangshilfe für bestehende
Antwortproduzenten. Sie ist kein Grund, neue gemeinsame native Felder hinzuzufügen.

## Präsentation im Vergleich zu InteractiveReply

`InteractiveReply` ist die ältere interne Teilmenge, die von Hilfsfunktionen für Genehmigungen und Interaktionen
verwendet wird. Sie unterstützt:

- Text
- Schaltflächen
- Auswahlfelder

`MessagePresentation` ist der kanonische gemeinsame Sendevertrag. Er ergänzt:

- Titel
- Ton
- Kontext
- Trennlinie
- Diagramm
- Tabelle
- reine URL-Schaltflächen
- generische Zustellungsmetadaten über `ReplyPayload.delivery`

Verwenden Sie beim Überbrücken älteren
Codes Hilfsfunktionen aus `openclaw/plugin-sdk/interactive-runtime`:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Neuer Code sollte `MessagePresentation` direkt akzeptieren oder erzeugen. Bestehende
`interactive`-Nutzlasten sind eine veraltete Teilmenge von `presentation`; die Laufzeitunterstützung
für ältere Produzenten bleibt bestehen.

Wichtige, nicht veraltete Hilfsfunktionen:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  validieren und konvertieren eine untypisierte Nutzlast (zum Beispiel JSON aus dem CLI-Flag
  `--presentation`) in `MessagePresentation`.
- `isMessagePresentationInteractiveBlock(block)` schränkt einen Block auf die
  Union `buttons` | `select` ein.
- `resolveMessagePresentationButtonAction(button)` und
  `resolveMessagePresentationOptionAction(option)` geben die kanonische typisierte
  Aktion zurück und akzeptieren dabei veraltete Grenzfelder. Ein explizites `action`
  hat immer Vorrang.
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` lesen ausschließlich skalare Befehls-/Callback-Werte.
  Eine nicht skalare kanonische Aktion fällt niemals auf ein veraltetes Schattenfeld
  `value` zurück, sodass Genehmigungs-IDs und Linkziele typisiert bleiben.
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` rendern einen strukturierten
  Datenblock als deterministischen Text für kanalspezifische Fallback-Pfade.

Die alten `InteractiveReply*`-Typen und Konvertierungshilfen sind im SDK als
`@deprecated` gekennzeichnet:

- `InteractiveReply`, `InteractiveReplyBlock`, `InteractiveReplyButton` und
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` und
`presentationToInteractiveControlsReply(...)` bleiben als Renderer-Brücken
für alte Kanalimplementierungen verfügbar. Neuer Producer-Code sollte sie nicht
aufrufen; senden Sie `presentation` und überlassen Sie das Rendering der Kern-/Kanalanpassung.

Auch für Genehmigungshilfen gibt es darstellungsorientierte Ersatzlösungen:

- verwenden Sie `buildApprovalPresentation(...)` anstelle von
  `buildApprovalInteractiveReply(...)`
- verwenden Sie `buildExecApprovalPresentation(...)` anstelle von
  `buildExecApprovalInteractiveReply(...)`

Diese ausgelieferten Builder bleiben aus Gründen der Plugin-Kompatibilität befehlsbasiert. Gateway-
und gebündelter Kanalcode, der einen dauerhaften Genehmigungstyp besitzt, sollte
`buildTypedApprovalPresentation(...)`,
`buildTypedExecApprovalPendingReplyPayload(...)` oder
`buildTypedPluginApprovalPendingReplyPayload(...)` verwenden, damit Transporte eine
explizite `approval`-Aktion erhalten, anstatt die Semantik aus `/approve`-Text abzuleiten.

`renderMessagePresentationFallbackText(...)` gibt für
Darstellungsblöcke ohne Text-Fallback eine leere Zeichenfolge zurück, etwa für eine
Darstellung, die ausschließlich aus einer Trennlinie besteht. Transporte, die einen nicht leeren Sendetext benötigen,
können `emptyFallback` übergeben, um einen minimalen Text zu aktivieren, ohne den standardmäßigen Fallback-
Vertrag zu ändern.

## Anheften bei der Zustellung

Das Anheften ist ein Zustellungsverhalten, keine Darstellung. Verwenden Sie `delivery.pin` anstelle von
Provider-nativen Feldern wie `channelData.telegram.pin`.

Semantik:

- `pin: true` heftet die erste erfolgreich zugestellte Nachricht an.
- `pin.notify` verwendet standardmäßig `false`.
- `pin.required` verwendet standardmäßig `false`.
- Optionale Fehler beim Anheften werden toleriert und lassen die gesendete Nachricht unverändert.
- Erforderliche Fehler beim Anheften führen zum Fehlschlagen der Zustellung.
- Bei aufgeteilten Nachrichten wird der erste zugestellte Teil angeheftet, nicht der letzte Teil.

Manuelle Nachrichtenaktionen für `pin`, `unpin` und `pins` sind weiterhin für vorhandene
Nachrichten verfügbar, sofern der Provider diese Operationen unterstützt.

## Checkliste für Plugin-Autoren

- Deklarieren Sie `presentation` aus `describeMessageTool(...)`, wenn der Kanal
  semantische Darstellung rendern oder sicher degradieren kann.
- Fügen Sie `presentationCapabilities` zum ausgehenden Runtime-Adapter hinzu.
- Implementieren Sie `renderPresentation` im Runtime-Code, nicht im
  Plugin-Einrichtungscode der Steuerungsebene.
- Halten Sie native UI-Bibliotheken aus häufig ausgeführten Einrichtungs-/Katalogpfaden heraus.
- Deklarieren Sie allgemeine Fähigkeitsgrenzen in `presentationCapabilities.limits`, sofern
  sie bekannt sind.
- Behalten Sie die endgültigen Plattformgrenzen im Renderer und in den Tests bei.
- Fügen Sie Fallback-Tests für nicht unterstützte Diagramme, Tabellen, Schaltflächen, Auswahlfelder, URL-
  Schaltflächen, Titel-/Textduplizierung sowie gemischte Sendevorgänge mit `message` und `presentation`
  hinzu.
- Fügen Sie Unterstützung für das Anheften bei der Zustellung über `deliveryCapabilities.pin` und
  `pinDeliveredMessage` nur hinzu, wenn der Provider die ID der gesendeten Nachricht anheften kann.
- Machen Sie keine neuen Provider-nativen Karten-/Block-/Komponenten-/Schaltflächenfelder über
  das gemeinsame Nachrichtenaktionsschema verfügbar.

## Zugehörige Dokumentation

- [Nachrichten-CLI](/de/cli/message)
- [Übersicht über das Plugin SDK](/de/plugins/sdk-overview)
- [Plugin-Architektur](/de/plugins/architecture-internals#message-tool-schemas)
- [Refactoring-Plan für die Kanaldarstellung](/de/plan/ui-channels)
