---
read_when:
    - Sie erstellen oder überarbeiten den Sendepfad eines Messaging-Kanal-Plugins.
    - Sie benötigen eine zuverlässige Zustellung der endgültigen Antwort, Empfangsbestätigungen, den Abschluss der Live-Vorschau oder eine Richtlinie für Empfangsbestätigungen
    - Sie migrieren von Hilfsfunktionen für den Versand von Kanalnachrichten oder Legacy-Antworten.
summary: 'API für den Lebenszyklus ausgehender Nachrichten für Kanal-Plugins: Adapter, Empfangsbestätigungen, dauerhafte Sendevorgänge, Live-Vorschau und Hilfsfunktionen für die Antwort-Pipeline'
title: API für ausgehende Kanalnachrichten
x-i18n:
    generated_at: "2026-07-26T18:31:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

Channel-Plugins stellen das Verhalten für ausgehende Nachrichten über
`openclaw/plugin-sdk/channel-outbound` bereit. Verwenden Sie
`openclaw/plugin-sdk/channel-inbound` für die Orchestrierung von Empfang, Kontext und Dispatch.

Der Core ist zuständig für Warteschlangen, Dauerhaftigkeit, den dauerhaften **Ingress-Monitor und Drain**
(`createChannelIngressMonitor`, `createChannelIngressDrain` und
`openChannelIngressDrain`), die generische Wiederholungsrichtlinie, den Lebenszyklus der Turn-Übernahme
(`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`), Hooks,
Empfangsbestätigungen und das gemeinsame Tool `message`. Das Plugin ist zuständig für native
Aufrufe zum Senden/Bearbeiten/Löschen, Zielnormalisierung, plattformspezifische Threads, ausgewählte
Zitate, Benachrichtigungs-Flags, Kontostatus, Ingress-Prüfung und Payload-
Codierung, Lane-Schlüssel, Prädikate für nicht wiederholbare Fehler, optionale Supersede-
Autorisierung und plattformspezifische Nebeneffekte.

## Dauerhafte Ingress-Monitore

Verwenden Sie `createChannelIngressMonitor(...)`, wenn ein Channel akzeptierte
Transportereignisse vor dem Dispatch dauerhaft speichern muss. Es kombiniert eine Channel-Ingress-Warteschlange und einen Drain
mit dem gemeinsamen Lebenszyklus für Zulassung, Polling, Bereinigung, Zustellung und Herunterfahren.
Verwenden Sie das niedriger angesiedelte `createChannelIngressDrain(...)` nur, wenn der Transport
einen wesentlich anderen Zulassungs- oder Pump-Vertrag besitzt.

Folgende Optionen sind erforderlich:

| Option                           | Vertrag                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | Ein `ChannelIngressQueue` oder eine verzögerte Factory, die die kontobezogene Warteschlange öffnet.                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | Gibt die stabile `eventId` und die serialisierte `laneKey` zurück oder `null` für ein ignoriertes Ereignis. Die Fakten zum Claim-Zeitpunkt müssen mit der dauerhaft gespeicherten ID und Lane übereinstimmen.                                                                                                                                                                    |
| `payload`                        | Stellt die Payload-Version sowie die Serialisierung/Deserialisierung des Inhalts bereit. Verwenden Sie `storage: "raw-event"` für den standardmäßigen String-Umschlag `{ version, rawEvent }` oder stellen Sie benutzerdefinierte Encode-/Decode-Callbacks für eine vorhandene Channel-spezifische Struktur bereit. `createClaimError` klassifiziert ungültige Versionen oder eine geänderte Identität. |
| `deliver(raw, lifecycle, claim)` | Führt den Dispatch eines decodierten Ereignisses aus und empfängt den vollständigen Übernahmelebenszyklus. Es kann `completed`, `deferred`, `failed-retryable` oder nichts zurückgeben.                                                                                                                                                                |
| `pollIntervalMs`                 | Plant Wiederherstellungs-/Drain-Polls, während der Monitor ausgeführt wird.                                                                                                                                                                                                                                                     |
| `retention`                      | Legt den Bereinigungsrhythmus sowie TTLs und Eintragsobergrenzen für abgeschlossene/fehlgeschlagene Einträge fest.                                                                                                                                                                                                                                              |

Der Monitor serialisiert Zulassungen, damit der Append-Backoff die Reihenfolge einer Lane nicht umkehren kann. Die
standardmäßigen begrenzten Append-Verzögerungen betragen `0`, `100` und `300` ms; nach ihrer Ausschöpfung wird
der Transport-Callback abgelehnt, statt ein Ereignis zu dispatchen, das nicht
dauerhaft gespeichert wurde. Zum Claim-Zeitpunkt decodiert der Monitor die versionierte Payload, führt `inspect` erneut aus und
lehnt eine nicht übereinstimmende ID oder Lane vor der Zustellung ab.

`deliver` empfängt `onAdopted`, `onDeferred`, `onAdoptionFinalizing`,
`onAbandoned` und `abortSignal`. Eine Rückgabe ohne explizite Übergabe markiert ein
terminales Ereignis ohne Dispatch als übernommen. `admission` ist immer `exclusive`. Eine
verzögerte Übergabe hält den Claim aufrecht, während Herunterfahren oder Abbruch nicht übernommene
Arbeit weiterhin wiederholbar lässt. Der Monitor verfolgt die Zustellung unabhängig vom Abschluss des Claims,
da die Übernahme eine Zeile mit einem Tombstone versehen kann, bevor das Zustellungs-Promise des Channels
zurückgegeben wird.

Zu den optionalen Einstellungen gehören benutzerdefinierte Append-Verzögerungen, ein Optionsblock `drain` für
erweiterte Drain-Reihenfolge, Parallelität und Wiederholungsrichtlinie, ein externes `abortSignal`, eine
Uhr, die Meldung von Pump-Fehlern, eine Factory für Fehler im gestoppten Zustand und eine Zulassungsrichtlinie.
Der zurückgegebene Monitor stellt `admit`, `start`, `pause`, `stop`, `waitForIdle`,
`isRunning` und `isStopped` bereit. `stop` schließt zunächst akzeptierte Zulassungen ab,
bricht dann den Drain ab und gibt ihn frei, wartet auf die Pump und aktive Zustellungen und
gibt ihn erneut frei, um das Race bei der verzögerten Erstellung zu schließen.

Belassen Sie transportspezifische Schwärzung, Validierung des Roh-Umschlags, Klassifizierung als nicht wiederholbar
und die dauerhaft gespeicherte Payload-Struktur im Plugin. Webhook-Transporte
sollten erst eine Bestätigung senden, nachdem `admit` aufgelöst wurde; Transporte ohne Wiederholungsmöglichkeit sollten
die Ausschöpfung dauerhafter Append-Versuche melden, statt stillschweigend zu dispatchen.

## Adapter

Die meisten Plugins definieren einen `message`-Adapter:

```ts
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-outbound";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

Deklarieren Sie nur Fähigkeiten, die der native Transport tatsächlich erhält. Decken Sie
jede deklarierte Fähigkeit für Senden, Empfangsbestätigungen, Live-Vorschau und Empfangsbestätigung mit
den aus diesem Unterpfad exportierten Vertragshilfen ab.

## Unterdrückung ausgehender Echos

Wenn eine Plattform die eigene ausgehende Nachricht des Plugins erneut als eingehend zustellen kann, rufen Sie `recordOutboundMessageIdentity(...)` mit Channel, Konto, Konversation und einer stabilen Plattformnachrichten- oder Quellidentität auf. Der gemeinsame Pfad für eingehende Turns verwirft übereinstimmende Identitäten innerhalb eines begrenzten Zeitfensters von 30 Sekunden vor der Sitzungsaufzeichnung oder dem Agent-Dispatch; eine Quellidentität kann vor dem Senden reserviert oder beim Entfernen einer Channel-Route aktualisiert werden, um Zustellungs-Races zu schließen. `isRecentOutboundMessageIdentity(...)` stellt dieselbe Abfrage für Channel-Diagnosen und Tests bereit. Pflegen Sie für dieselbe stabile Identität keinen parallelen Channel-lokalen TTL-Cache.

## Klartextbereinigung

Verwenden Sie `sanitizeForPlainText(...)`, wenn ein Adapter für ausgehende Nachrichten die
unterstützten HTML-Formatierungs-Tags in leichtgewichtige Textauszeichnung umwandeln muss. Standardmäßig bleiben
die vorhandenen chatartigen Markierungen für Fettdruck und Durchstreichung erhalten. Übergeben Sie
`{ style: "markdown" }` nur, wenn der Channel das Ergebnis erneut als Markdown parst:

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

Der Markdown-Stil verwendet `**bold**` und `~~strikethrough~~`; Kursivschrift und Inline-
Code behalten in beiden Stilen `_italic_` und Backtick-Markierungen bei. Wählen Sie den Stil an
der Channel-Grenze aus, statt Markierungstext nach der Bereinigung umzuschreiben.

## Zustellungsnachweis

Ein `MessageReceipt` zeichnet das von einem Channel-Adapter zurückgegebene Ergebnis auf. Konkrete
Plattformnachrichten-IDs zeigen, dass der Sendeweg der Plattform die
Nachricht akzeptiert hat; sie beweisen nicht, dass das Gerät eines Empfängers sie angezeigt oder gelesen hat.
Empfangsbestätigungen ohne Plattformnachrichten-IDs sind lediglich lokale Empfangsmetadaten.
Channels mit Lesebestätigungen oder einem Gerätezustellungsstatus sollten diese Fakten
über einen separaten Channel-spezifischen Pfad verfolgen.

Wenn ein Channel-Adapter nachweisen kann, dass die Wiederholung eines Fehlers keinen
für den Empfänger sichtbaren Sendevorgang duplizieren kann und kein finalisierungsfähiger Aufruf begonnen hat, lösen Sie
`new PlatformMessageNotDispatchedError("...", { cause: error })` aus
`openclaw/plugin-sdk/error-runtime` aus. Der Core kann dann veraltete Nachweise für Sendeversuche
löschen und den Intent in der Warteschlange sicher wiederholen. Nur der Adapter, dem die
endgültige Dispatch-Grenze gehört, darf diese Zusicherung abgeben. Verwenden Sie die Markierung niemals, nachdem ein
Finalisierungs-/Sendeaufruf begonnen hat oder ein mehrdeutiges Ergebnis zurückgibt; eine falsche Markierung kann
Nachrichten duplizieren.

## Vorhandene Adapter für ausgehende Nachrichten

Wenn der Channel bereits über einen kompatiblen `outbound`-Adapter verfügt, leiten Sie den
Nachrichtenadapter daraus ab, statt den Sendecode zu duplizieren:

```ts
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-outbound";

export const messageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound,
  durableFinal: {
    capabilities: {
      text: true,
      media: true,
    },
  },
});
```

## Dauerhafte Sendevorgänge

Runtime-Sendehilfen befinden sich ebenfalls unter `channel-outbound`:

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- Hilfen für Draft-Streaming/Fortschritt wie `resolveChannelDraftStreamingChunking(...)`

`sendDurableMessageBatch(...)` gibt genau ein explizites Ergebnis zurück:

| Ergebnis          | Bedeutung                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | Mindestens eine sichtbare Plattformnachricht wurde vom Sendeweg der Plattform akzeptiert.            |
| `suppressed`     | Keine Plattformnachricht sollte als fehlend behandelt werden.                                        |
| `partial_failed` | Mindestens eine Plattformnachricht wurde akzeptiert, bevor eine spätere Payload oder ein Nebeneffekt fehlschlug. |
| `failed`         | Es wurde keine Plattform-Empfangsbestätigung erzeugt.                                                        |

Verwenden Sie `payloadOutcomes`, wenn ein Batch gesendete, unterdrückte und fehlgeschlagene
Payloads mischt. Leiten Sie die Hook-Abbrechung nicht aus einem leeren veralteten
Direktzustellungsergebnis ab.

## Zulassung verzögerter Zustellungen

Verwenden Sie `message.durableFinal.admitDeferredDelivery(...)`, wenn ein aufgelöstes Konto
vom Core verwaltete ausgehende oder verzögerte Zustellungen nicht sicher akzeptieren kann. Der Core ruft
diesen Hook synchron vor aktiver ausgehender Arbeit auf, einschließlich Pfaden, welche die
dauerhafte Speicherung in der Warteschlange überspringen, und erneut vor der Wiedergabe eines wiederhergestellten Intents. Der Kontext
enthält `cfg`, `channel`, `to`, `accountId` und ein `phase` mit `live` oder
`recovery`.

Geben Sie `{ status: "allowed" }` zurück, um fortzufahren. Geben Sie
`{ status: "permanent_rejection", reason }` zurück, wenn die Zustellung weder
dauerhaft gespeichert noch direkt gesendet oder erneut wiedergegeben werden darf. Eine aktive Ablehnung schlägt vor der Erstellung der Warteschlange,
Nachrichten-Hooks oder Plattformarbeit fehl. Eine Ablehnung bei der Wiederherstellung markiert den
Warteschlangeneintrag als fehlgeschlagen und überspringt Abgleich und Wiedergabe. Wird der Hook weggelassen,
gilt die Zustellung als zulässig.

Der Hook ist eine synchrone Zulassungsentscheidung, kein Sendepfad. Lesen Sie nur
bereits geladene Konfiguration oder bereits geladenen Laufzeitstatus; führen Sie keine Netzwerk-, Dateisystem- oder
sonstigen asynchronen E/A-Vorgänge aus. Vertragstests sollten beide Phasen und beide
Ergebnisvarianten über `ChannelMessageDurableFinalAdapter` aus
`openclaw/plugin-sdk/channel-outbound` abdecken.

## Kompatibilitäts-Dispatch

Stellen Sie den Dispatch eingehender Antworten über `dispatchChannelInboundReply(...)`
aus `channel-inbound` zusammen. Belassen Sie die Plattformzustellung im Zustelladapter; verwenden Sie
`channel-outbound` für Nachrichtenadapter, persistente Sendevorgänge, Empfangsbestätigungen, Live-
Vorschau und Optionen der Antwort-Pipeline.
