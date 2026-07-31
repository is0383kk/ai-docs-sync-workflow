---
read_when:
    - Refactoring der Benutzeroberfläche für Kanalnachrichten, interaktiver Payloads oder nativer Kanal-Renderer
    - Funktionen des Nachrichten-Tools, Zustellungshinweise oder kontextübergreifende Markierungen ändern
    - Debugging des Import-Fan-outs von Discord Carbon oder der verzögerten Laufzeitinitialisierung des Channel-Plugins
summary: Semantische Nachrichtendarstellung von nativen UI-Renderern der Kanäle entkoppeln.
title: Refaktorierungsplan für die Channel-Darstellung
x-i18n:
    generated_at: "2026-07-26T17:56:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## Status

Implementiert für den gemeinsamen Agenten, die CLI, Plugin-Fähigkeiten und ausgehende Zustellungsoberflächen:

- `ReplyPayload.presentation` überträgt die semantische Nachrichten-UI.
- `ReplyPayload.delivery.pin` überträgt Anfragen zum Anheften gesendeter Nachrichten.
- Gemeinsame Nachrichtenaktionen stellen `presentation`, `delivery` und `pin` anstelle der Provider-nativen `components`, `blocks`, `buttons` oder `card` bereit.
- Der Kern rendert die Darstellung über vom Plugin deklarierte ausgehende Fähigkeiten oder stuft sie automatisch herab.
- Die Renderer von Discord, Slack, Telegram, Mattermost, MS Teams und Feishu verwenden den generischen Vertrag.
- Der Control-Plane-Code des Discord-Kanals importiert keine Carbon-basierten UI-Container mehr.

Die kanonische Dokumentation befindet sich jetzt unter [Nachrichtendarstellung](/de/plugins/message-presentation).
Bewahren Sie diesen Plan als historischen Implementierungskontext auf; aktualisieren Sie den kanonischen Leitfaden
bei Änderungen am Vertrag, Renderer- oder Fallback-Verhalten.

## Problem

Die Kanal-UI ist derzeit auf mehrere inkompatible Oberflächen verteilt:

- Der Kern besitzt über `buildCrossContextComponents` einen Discord-förmigen kontextübergreifenden Renderer-Hook.
- Discord `channel.ts` kann über `DiscordUiContainer` die native Carbon-UI importieren, wodurch UI-Laufzeitabhängigkeiten in die Control Plane des Kanal-Plugins gelangen.
- Der Agent und die CLI stellen native Payload-Ausweichmöglichkeiten bereit, etwa Discord `components`, Slack `blocks`, Telegram oder Mattermost `buttons` sowie Teams oder Feishu `card`.
- `ReplyPayload.channelData` überträgt sowohl Transporthinweise als auch native UI-Umschläge.
- Das generische Modell `interactive` ist vorhanden, aber weniger umfangreich als die bereits von Discord, Slack, Teams, Feishu, LINE, Telegram und Mattermost verwendeten komplexeren Layouts.

Dadurch kennt der Kern native UI-Strukturen, die verzögerte Laufzeitinitialisierung von Plugins wird geschwächt und Agenten erhalten zu viele Provider-spezifische Möglichkeiten, dieselbe Nachrichtenabsicht auszudrücken.

## Ziele

- Der Kern bestimmt anhand deklarierter Fähigkeiten die beste semantische Darstellung für eine Nachricht.
- Erweiterungen deklarieren Fähigkeiten und rendern die semantische Darstellung in native Transport-Payloads.
- Die Web-Control-UI bleibt von der nativen Chat-UI getrennt.
- Native Kanal-Payloads werden nicht über die gemeinsame Nachrichtenoberfläche des Agenten oder der CLI bereitgestellt.
- Nicht unterstützte Darstellungsfunktionen werden automatisch auf die bestmögliche Textdarstellung herabgestuft.
- Zustellungsverhalten wie das Anheften einer gesendeten Nachricht besteht aus generischen Zustellungsmetadaten und ist keine Darstellung.

## Nichtziele

- Kein Abwärtskompatibilitäts-Shim für `buildCrossContextComponents`.
- Keine öffentlichen nativen Ausweichmöglichkeiten für `components`, `blocks`, `buttons` oder `card`.
- Keine Kernimporte nativer UI-Bibliotheken von Kanälen.
- Keine Provider-spezifischen SDK-Schnittstellen für gebündelte Kanäle.

## Zielmodell

Fügen Sie `ReplyPayload` ein kerneigenes Feld `presentation` hinzu.

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

`interactive` wird während der Migration zu einer Teilmenge von `presentation`:

- Der Textblock `interactive` wird `presentation.blocks[].type = "text"` zugeordnet.
- Der Schaltflächenblock `interactive` wird `presentation.blocks[].type = "buttons"` zugeordnet.
- Der Auswahlblock `interactive` wird `presentation.blocks[].type = "select"` zugeordnet.

Die externen Agenten- und CLI-Schemas verwenden jetzt `presentation`; `interactive` bleibt ein interner Legacy-Helfer für das Parsen und Rendern bestehender Antwortproduzenten.
Die öffentliche, an Produzenten gerichtete API behandelt `interactive` als veraltet. Die Laufzeitunterstützung
bleibt bestehen, damit vorhandene Genehmigungshelfer und ältere Plugins weiterhin
funktionieren, während neuer Code `presentation` ausgibt.

## Zustellungsmetadaten

Fügen Sie für Sendeverhalten, das nicht zur UI gehört, ein kerneigenes Feld `delivery` hinzu.

```ts
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

Semantik:

- `delivery.pin = true` bedeutet, dass die erste erfolgreich zugestellte Nachricht angeheftet wird.
- `notify` verwendet standardmäßig `false`.
- `required` verwendet standardmäßig `false`; bei nicht unterstützten Kanälen oder fehlgeschlagenem Anheften wird die Zustellung automatisch fortgesetzt.
- Manuelle Nachrichtenaktionen `pin`, `unpin` und `list-pins` bleiben für vorhandene Nachrichten bestehen.

Die aktuelle Telegram-ACP-Themenbindung sollte von `channelData.telegram.pin = true` nach `delivery.pin = true` verschoben werden.

## Vertrag für Laufzeitfähigkeiten

Fügen Sie dem ausgehenden Laufzeitadapter Hooks für das Rendern der Darstellung und die Zustellung hinzu, nicht dem Control-Plane-Kanal-Plugin.

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
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

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

Kernverhalten:

- Zielkanal und Laufzeitadapter auflösen.
- Darstellungsfähigkeiten abfragen.
- Nicht unterstützte Blöcke herabstufen und vor dem Rendern generische Fähigkeitsgrenzen
  anwenden.
- `renderPresentation` aufrufen.
- Wenn kein Renderer vorhanden ist, die Darstellung in eine Textausweichdarstellung umwandeln.
- Nach erfolgreichem Senden `pinDeliveredMessage` aufrufen, wenn `delivery.pin` angefordert wird und unterstützt ist.

## Kanalzuordnung

Discord:

- `presentation` in reinen Laufzeitmodulen als Components v2 und Carbon-Container rendern.
- Hilfsfunktionen für Akzentfarben in schlanken Modulen belassen.
- Importe von `DiscordUiContainer` aus dem Control-Plane-Code des Kanal-Plugins entfernen.

Slack:

- `presentation` als Block Kit rendern.
- Eingabe von `blocks` aus Agent und CLI entfernen.

Telegram:

- Text, Kontext und Trennlinien als Text rendern.
- Aktionen und Auswahl als Inline-Tastaturen rendern, sofern sie konfiguriert und für die Zieloberfläche zulässig sind.
- Die Textausweichdarstellung verwenden, wenn Inline-Schaltflächen deaktiviert sind.
- Das Anheften von ACP-Themen nach `delivery.pin` verschieben.

Mattermost:

- Aktionen als interaktive Schaltflächen rendern, sofern konfiguriert.
- Andere Blöcke als Textausweichdarstellung rendern.

MS Teams:

- `presentation` als Adaptive Cards rendern.
- Manuelle Aktionen zum Anheften, Lösen und Auflisten angehefteter Nachrichten beibehalten.
- `pinDeliveredMessage` optional implementieren, wenn die Graph-Unterstützung für die Zielkonversation zuverlässig ist.

Feishu:

- `presentation` als interaktive Karten rendern.
- Manuelle Aktionen zum Anheften, Lösen und Auflisten angehefteter Nachrichten beibehalten.
- `pinDeliveredMessage` optional für das Anheften gesendeter Nachrichten implementieren, wenn das API-Verhalten zuverlässig ist.

LINE:

- `presentation` nach Möglichkeit als Flex- oder Vorlagennachrichten rendern.
- Bei nicht unterstützten Blöcken auf Text zurückgreifen.
- LINE-UI-Payloads aus `channelData` entfernen.

Einfache oder eingeschränkte Kanäle:

- Die Darstellung mit zurückhaltender Formatierung in Text umwandeln.

## Refaktorierungsschritte

1. Den Discord-Release-Fix erneut anwenden, der `ui-colors.ts` von der Carbon-basierten UI trennt und `DiscordUiContainer` aus `extensions/discord/src/channel.ts` entfernt.
2. `presentation` und `delivery` zu `ReplyPayload`, der Normalisierung ausgehender Payloads, Zustellungszusammenfassungen und Hook-Payloads hinzufügen.
3. Schema- und Parser-Hilfsfunktionen für `MessagePresentation` in einem eng begrenzten SDK-/Laufzeit-Unterpfad hinzufügen.
4. Die Nachrichtenfähigkeiten `buttons`, `cards`, `components` und `blocks` durch semantische Darstellungsfähigkeiten ersetzen.
5. Hooks zum Rendern der Darstellung und zum Anheften bei der Zustellung zum ausgehenden Laufzeitadapter hinzufügen.
6. Die kontextübergreifende Komponentenkonstruktion durch `buildCrossContextPresentation` ersetzen.
7. `src/infra/outbound/channel-adapters.ts` löschen und `buildCrossContextComponents` aus den Typen des Kanal-Plugins entfernen.
8. `maybeApplyCrossContextMarker` so ändern, dass `presentation` anstelle nativer Parameter angehängt wird.
9. Die Sendepfade der Plugin-Verteilung so aktualisieren, dass sie ausschließlich semantische Darstellung und Zustellungsmetadaten verarbeiten.
10. Native Payload-Parameter aus Agent und CLI entfernen: `components`, `blocks`, `buttons` und `card`.
11. SDK-Hilfsfunktionen entfernen, die native Schemas für Nachrichtenwerkzeuge erstellen, und sie durch Hilfsfunktionen für Darstellungsschemas ersetzen.
12. UI-/native Umschläge aus `channelData` entfernen; ausschließlich Transportmetadaten beibehalten, bis jedes verbleibende Feld überprüft wurde.
13. Die Renderer für Discord, Slack, Telegram, Mattermost, MS Teams, Feishu und LINE migrieren.
14. Die Dokumentation für Nachrichten-CLI, Kanalseiten, Plugin-SDK und das Fähigkeits-Cookbook aktualisieren.
15. Import-Fanout-Profiling für Discord und betroffene Kanaleinstiegspunkte ausführen.

Die Schritte 1–11 und 13–14 sind in dieser Refaktorierung für den gemeinsamen Agenten, die CLI, Plugin-Fähigkeiten und Verträge ausgehender Adapter implementiert. Schritt 12 bleibt ein tiefergehender interner Bereinigungsdurchlauf für Provider-private `channelData`-Transportumschläge. Schritt 15 bleibt eine nachgelagerte Validierung, falls quantifizierte Import-Fanout-Zahlen über das Typ-/Test-Gate hinaus benötigt werden.

## Tests

Hinzufügen oder aktualisieren:

- Tests zur Darstellungsnormalisierung.
- Tests zur automatischen Herabstufung der Darstellung bei nicht unterstützten Blöcken.
- Tests für kontextübergreifende Markierungen bei Plugin-Verteilung und Kernzustellungspfaden.
- Tests der Kanal-Render-Matrix für Discord, Slack, Telegram, Mattermost, MS Teams, Feishu, LINE und die Textausweichdarstellung.
- Tests des Nachrichtenwerkzeug-Schemas, die belegen, dass native Felder entfernt wurden.
- CLI-Tests, die belegen, dass native Flags entfernt wurden.
- Regressionstest für die verzögerte Importinitialisierung des Discord-Einstiegspunkts unter Einbeziehung von Carbon.
- Tests zum Anheften bei der Zustellung für Telegram und den generischen Fallback.

## Offene Fragen

- Sollte `delivery.pin` im ersten Durchlauf für Discord, Slack, MS Teams und Feishu implementiert werden oder zunächst nur für Telegram?
- Sollte `delivery` künftig vorhandene Felder wie `replyToId`, `replyToCurrent`, `silent` und `audioAsVoice` aufnehmen oder weiterhin auf das Verhalten nach dem Senden ausgerichtet bleiben?
- Sollte die Darstellung Bilder oder Dateiverweise direkt unterstützen oder sollten Medien vorerst vom UI-Layout getrennt bleiben?

## Verwandte Themen

- [Übersicht über Kanäle](/de/channels)
- [Nachrichtendarstellung](/de/plugins/message-presentation)
