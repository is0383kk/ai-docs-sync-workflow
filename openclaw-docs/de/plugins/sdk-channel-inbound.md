---
read_when:
    - Sie erstellen oder überarbeiten den Empfangspfad eines Messaging-Kanal-Plugins.
    - Sie benötigen eine gemeinsame Erstellung des Kontexts für eingehende Nachrichten, Sitzungsaufzeichnung oder einen vorbereiteten Antwortversand
    - Sie migrieren alte Hilfsfunktionen für Channel-Turns zu Inbound-/Message-APIs
summary: 'Hilfsfunktionen für eingehende Ereignisse in Kanal-Plugins: Kontexterstellung, gemeinsame Runner-Orchestrierung, Sitzungsdatensatz und vorbereiteter Antwortversand'
title: API für eingehende Kanalnachrichten
x-i18n:
    generated_at: "2026-07-26T18:00:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

Channel-Empfangspfade folgen einem einzigen Ablauf:

```text
Plattformereignis -> eingehende Fakten/Kontext -> Agentenantwort -> Nachrichtenzustellung
```

Verwenden Sie `openclaw/plugin-sdk/channel-inbound` für die Normalisierung eingehender Ereignisse,
Formatierung, Roots und Orchestrierung. Verwenden Sie
`openclaw/plugin-sdk/channel-outbound` für natives Senden, Empfangsbestätigungen, dauerhafte
Zustellung und das Verhalten der Live-Vorschau.

## Core-Hilfsfunktionen

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`: überführt normalisierte Channel-Fakten
  in den Prompt-/Sitzungskontext. Übergeben Sie Channel-eigene Absender-/Chat-Metadaten
  über `channelContext`, die Plugin-Hooks als `ctx.channelContext` sehen.
  Ergänzen Sie `PluginHookChannelSenderContext` oder `PluginHookChannelChatContext`
  aus diesem Unterpfad um Channel-spezifische Felder.
- `runChannelInboundEvent(...)`: führt Aufnahme, Klassifizierung, Vorprüfung, Auflösung,
  Aufzeichnung, Dispatch und Abschluss für ein eingehendes Plattformereignis aus.
- `dispatchChannelInboundReply(...)`: zeichnet eine bereits
  zusammengestellte eingehende Antwort auf und versendet sie mit einem Zustellungsadapter.

Bei eingehenden Ereignissen, die ausschließlich Medien enthalten, lassen Sie Nachrichtentext und Befehlstext leer und
übergeben Sie pro nativem Anhang einen `ChannelInboundMediaInput`-Fakt. Wenn eine umgebende
Verlaufszeile oder ein anderer reiner Textträger diese Fakten beschreiben muss, verwenden Sie
`formatMediaPlaceholderText(media)`. Die Klassifizierung jedes Fakts erfolgt anhand von `kind`, MIME-
Typ und anschließend der Pfad- oder URL-Erweiterung; noch nicht heruntergeladene native Anhänge sollten dennoch
jeweils einen Fakt enthalten, der nur den Typ angibt. Verwenden Sie den Formatierer nicht, um den
primären eingehenden Text zu erzeugen.

Normalisieren Sie Plugin-eigene Anhangsdatensätze mit `toInboundMediaFacts(...)` und
übergeben Sie anschließend das resultierende geordnete Array über das Feld `media` des Kontexts:

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

Die Array-Position ist die Identität des Anhangs. Die faktbezogenen Felder `transcribed`, `messageId` und
`workspaceDir` ersetzen die veralteten parallelen Index-/Workspace-Felder. Die
Kontextfelder `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` und `MediaStaged`
sowie `buildChannelInboundMediaPayload(...)` bleiben nur als veraltete
Kompatibilitätsfelder verfügbar. Neue Plugins sollten sie weder erstellen noch lesen.

Gebündelte/native Channels, die bereits das injizierte Plugin-Laufzeitobjekt
erhalten, können dieselben Hilfsfunktionen unter `runtime.channel.inbound.*` aufrufen, statt
diesen Unterpfad direkt zu importieren:

```ts
await runtime.channel.inbound.run({
  channel: "demo",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest: normalizePlatformEvent,
    resolveTurn: resolveInboundReply,
  },
});
```

Stellen Sie `dispatchChannelInboundReply(...)`-Eingaben für Kompatibilitäts-
Dispatcher zusammen, bei denen die Plattformzustellung im Zustellungsadapter verbleibt. Neue Sendepfade
sollten stattdessen Nachrichtenadapter und Hilfsfunktionen für dauerhafte Nachrichten aus
`channel-outbound` verwenden.

## Vertrag für den Zustellungsabschluss

`ChannelInboundTurnPlan.delivery` ist für das native Senden jedes logischen Antwort-
Payloads zuständig. Core ist für die Reihenfolge ausgehender Hooks und, wenn der Adapter dies aktiviert,
die abschließende `message_sent`-Beobachtung zuständig. Halten Sie diese Verantwortlichkeiten getrennt, damit
ein Payload nicht zu doppelten Abschlussereignissen führen kann.

Die Felder des Zustellungsergebnisses haben folgende Bedeutung:

| Feld                    | Vertrag                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | Vom Provider akzeptierter sichtbarer Text für das logische Payload nach nativer Formatierung oder Finalisierung. Lassen Sie das Feld weg, um den vorbereiteten Payload-Text für die Abschlussbeobachtung zu verwenden. Bei Sendungen, die ausschließlich Medien enthalten, kann es weggelassen werden.                             |
| `messageIds` / `receipt` | Tatsächliche Provider-Identitäten für den sichtbaren Sendevorgang. Bevorzugen Sie ein `MessageReceipt`; Core verwendet dessen primäre Provider-ID für `message_sent`.                                                                                            |
| `visibleReplySent`       | Setzen Sie dies nur dann auf `false`, wenn der Provider weder eine sichtbare Vorschau noch eine endgültige Nachricht erzeugt hat. Core gibt für dieses Ergebnis kein erfolgreiches `message_sent` aus.                                                                          |
| `finalization`           | Ein Promise für den verzögerten nativen Abschluss desselben logischen Payloads, etwa beim Schließen oder Bearbeiten einer direkt aktualisierten Streaming-Karte. Die aufgelösten Felder überschreiben vor der Abschlussbeobachtung und `onDelivered` das unmittelbare Ergebnis. |

Setzen Sie die Option `observeMessageSent` des Zustellungsadapters auf `true`, wenn Core
für die nicht dauerhaften Sendungen dieses Adapters die kanonischen Plugin- und internen `message_sent`-Ereignisse
ausgeben soll. Geben Sie diese Option nicht aus `deliver` zurück und
geben Sie diese Ereignisse nicht zusätzlich im Plugin aus. Dauerhafte Sendungen werden bereits über
die gemeinsam genutzte Ausgangskomponente ausgegeben und nicht dupliziert.

Geben Sie ein Ergebnis pro logischem Payload zurück. `finalization` ist kein zweiter Sendevorgang und
darf `reply_payload_sending` oder `message_sending` nicht erneut ausführen. Sobald
`deliver` zurückkehrt, beobachtet Core eine Ablehnung des Finalisierungs-Promise, damit sie
nicht unbehandelt bleiben kann; Core wartet dennoch auf das ursprüngliche Promise, nachdem der Antwort-
Dispatch abgeschlossen ist. Anschließend gibt Core pro Payload höchstens eine Abschlussbeobachtung
mit dem finalisierten Inhalt und der Provider-ID aus. `onDelivered` erhält, sofern vorhanden,
das abgeschlossene Ergebnis nach dieser Beobachtung.

Lassen Sie `deliver` oder `finalization` fehlschlagen, wenn die native Zustellung fehlschlägt. Wenn kein Sendeversuch
beim Provider erfolgte, lösen Sie `PlatformMessageNotDispatchedError` aus
`openclaw/plugin-sdk/error-runtime` aus; Core unterdrückt ein fälschliches `message_sent`-
Ereignis. Wenn ein nativer Sendevorgang sichtbar wurde, bevor ein späterer Vorgang fehlschlug,
bewahren Sie die sichtbare Teilmenge im Fehler auf:

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

Core gibt eine fehlgeschlagene Abschlussbeobachtung mit diesem für den Provider sichtbaren Inhalt und
dieser Identität aus und lässt die Zustellung anschließend weiterhin als fehlgeschlagen gelten, damit Aufrufer einen teilweisen
Erfolg nicht mit einem fehlerfreien Sendevorgang verwechseln. Melden Sie `visibleReplySent: false` nicht, nachdem eine
Vorschau, ein Entwurf, ein Anhang oder eine endgültige Nachricht sichtbar geworden ist.

Wenn `reply_payload_sending` oder `message_sending` registriert ist, müssen diese Hooks
abgeschlossen sein, bevor etwas für den Provider Sichtbares erstellt wird, da jeder Hook
das logische Payload umschreiben oder abbrechen kann. Eine voreilige native Vorschau würde
Inhalt vor der Umschreibung offenlegen oder einen abgebrochenen Entwurf hinterlassen. Puffern Sie Vorschauinhalte,
bis das akzeptierte Payload `deliver` erreicht; Kompatibilitäts-Dispatcher, die
Vorschauen früher starten, müssen diese voreilige Vorschau unterdrücken, solange einer der Hooks
registriert ist. Verwenden Sie für neue Vorschaupfade die finalisierbaren Hilfsfunktionen für Live-Vorschauen aus der
[API für ausgehende Channel-Nachrichten](/de/plugins/sdk-channel-outbound).

## Migration

`runtime.channel.turn.*`-Laufzeit-Aliasse wurden entfernt. Verwenden Sie:

- `runtime.channel.inbound.run(...)` für rohe eingehende Ereignisse.
- `runtime.channel.inbound.dispatchReply(...)` für zusammengestellte Antwortkontexte.
- `runtime.channel.inbound.buildContext(...)` für eingehende Kontext-Payloads.
- `runtime.channel.inbound.runPreparedReply(...)`, veraltet, nur für
  Channel-eigene vorbereitete Dispatch-Pfade, die bereits ihre eigene
  Dispatch-Closure zusammenstellen.

Neuer Plugin-Code sollte keine mit `turn` benannten Channel-APIs einführen. Beschränken Sie die Terminologie für Modell- oder
Agentendurchläufe auf Agenten-/Provider-Code; Channel-Plugins verwenden Begriffe für Eingang,
Nachricht, Zustellung und Antwort.
