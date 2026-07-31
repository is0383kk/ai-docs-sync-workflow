---
read_when:
    - Sende- oder Empfangsverhalten von Kanälen refaktorieren
    - Ändern des Nachrichteneingangs für Kanäle, der Antwortzustellung, der Ausgangswarteschlange, des Vorschau-Streamings oder der Nachrichten-APIs des Plugin SDKs
    - Entwicklung eines neuen Kanal-Plugins, das dauerhafte Sendevorgänge, Empfangsbestätigungen, Vorschauen, Bearbeitungen oder Wiederholungsversuche benötigt
summary: 'Status des dauerhaften Lebenszyklus für den Empfang und Versand von Nachrichten: was veröffentlicht wurde, was sich gegenüber dem ursprünglichen Entwurf geändert hat und was noch offen ist'
title: Refaktorierung des Nachrichtenlebenszyklus
x-i18n:
    generated_at: "2026-07-26T18:55:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d21eda70b8be0de78677f4ff6d7547317112731d9e86a5bef58eac0268899818
    source_path: concepts/message-lifecycle-refactor.md
    workflow: 16
---

<Note>
Diese Seite entstand ursprünglich als zukunftsgerichteter Designvorschlag. Der Kern dieses
Designs wurde inzwischen in `src/channels/message/*` und den öffentlichen
Unterpfaden `openclaw/plugin-sdk/channel-outbound` / `channel-inbound` ausgeliefert. Verwenden Sie für die
aktuelle API die [API für ausgehende Kanäle](/de/plugins/sdk-channel-outbound) und die
[API für eingehende Kanäle](/de/plugins/sdk-channel-inbound). Diese Seite dokumentiert, was
ausgeliefert wurde, wo die Implementierung vom ursprünglichen Entwurf abweicht und was
noch offen ist.
</Note>

## Warum dieses Refactoring erfolgte

Der Kanal-Stack entstand aus mehreren lokalen Korrekturen: getrennten Hilfsfunktionen für eingehende
Nachrichten je nach Reifegrad (`runtime.channel.inbound.run` für einfache Adapter,
`runtime.channel.inbound.runPreparedReply` für funktionsreiche), veralteten Hilfsfunktionen für die Antwortzustellung
(`dispatchInboundReplyWithBase`, `recordInboundSessionAndDispatchReply`),
kanalspezifischem Vorschau-Streaming und nachträglich an bestehende
Antwort-Payload-Pfade angefügter Dauerhaftigkeit der endgültigen Zustellung. Diese Struktur führte zu zu vielen öffentlichen Konzepten und
zu vielen Stellen, an denen die Zustellungssemantik auseinanderdriften konnte.

Die Zuverlässigkeitslücke, die das neue Design erforderlich machte:

```text
Telegram-Polling-Aktualisierung bestätigt
  -> endgültiger Assistententext ist vorhanden
  -> Prozess startet neu, bevor sendMessage erfolgreich ausgeführt wird
  -> endgültige Antwort geht verloren
```

Zielinvariante: Sobald der Kern entscheidet, dass eine sichtbare ausgehende Nachricht vorhanden sein soll,
muss die Sendeabsicht dauerhaft gespeichert werden, bevor der Plattformaufruf versucht wird, und der
Plattformbeleg muss nach erfolgreicher Ausführung festgeschrieben werden. Dadurch wird standardmäßig eine
Wiederherstellung mit mindestens einmaliger Zustellung ermöglicht. Ein Verhalten mit exakt einmaliger Zustellung besteht nur, wenn ein Adapter
native Idempotenz nachweist oder einen Sendeversuch mit unbekanntem Ergebnis nach dem Senden vor der
Wiederholung mit dem Plattformzustand abgleicht.

## Was ausgeliefert wurde

Die interne Domäne befindet sich in `src/channels/message/*`:

| Datei                        | Zuständigkeit                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `types.ts`                  | Typverträge für Adapter, Sendekontext, Beleg und dauerhafte Absicht                                                  |
| `send.ts`                   | `withDurableMessageSendContext` / `sendDurableMessageBatch` — der dauerhafte Sendekontext                             |
| `receive.ts`                | `createMessageReceiveContext` — Zustandsautomat für die Bestätigungsrichtlinie eingehender Nachrichten                                                   |
| `live.ts`                   | Zustand der Live-Vorschau und Logik zum Abschließen an Ort und Stelle oder zum Ausweichen                                                        |
| `state.ts`                  | `classifyDurableSendRecoveryState` — Wiederherstellungsklassifizierung nach einer Unterbrechung                                    |
| `receipt.ts`                | Normalisiert Ergebnisse des Plattformversands in `MessageReceipt`                                                             |
| `capabilities.ts`           | Leitet die erforderlichen Fähigkeiten für dauerhafte endgültige Zustellungen aus einem Payload ab                                                         |
| `contracts.ts`              | Vertragsnachweisprüfung für deklarierte Adapterfähigkeiten                                                      |
| `adapter.ts`                | `defineChannelMessageAdapter`                                                                                      |
| `outbound-bridge.ts`        | `createChannelMessageAdapterFromOutbound` — umschließt veraltete Funktionen `sendText`/`sendMedia`/`sendPayload`/`sendPoll` |
| `ingress-queue.ts`          | `createChannelIngressQueue` — dauerhafte Ereigniswarteschlange für eingehende Nachrichten                                                          |
| `durable-receive.ts`        | `createDurableInboundReceiveJournal` — Journal für Annehmen/Ausstehend/Abschließen/Freigeben zur Deduplizierung eingehender Nachrichten                  |
| `inbound-reply-dispatch.ts` | `dispatchChannelInboundReply` und Wrapper mit veralteten Namen                                                            |
| `reply-pipeline.ts`         | `createChannelReplyPipeline`, Hilfsfunktionen für Antwortpräfixe und Eingabe-Callbacks                                             |

Öffentliche Oberfläche: `openclaw/plugin-sdk/channel-outbound` (Hilfsfunktionen für Versand/Beleg/Dauerhaftigkeit/Live-Vorschau/Antwort-Pipeline)
und `openclaw/plugin-sdk/channel-inbound` (Kontext eingehender Nachrichten, `runChannelInboundEvent`,
`dispatchChannelInboundReply`). Auf diesen Seiten finden Sie Adapterbeispiele, aktuelle
Typnamen und Migrationshinweise — sie sind die maßgebliche Quelle für die API-
Struktur, nicht die nachfolgenden Entwürfe.

### Sendekontext

`withDurableMessageSendContext` stellt dem Kanalcode die Schritte `render`, `previewUpdate`,
`send`, `edit`, `delete`, `commit` und `fail` für eine ausgehende
Nachricht bereit. `sendDurableMessageBatch` ist der Wrapper für den Regelfall: rendern, senden,
anschließend bei `sent`/`suppressed` festschreiben oder bei einem Fehler fehlschlagen.

`sendDurableMessageBatch` gibt eines der folgenden diskriminierten Ergebnisse zurück:

| Status           | Bedeutung                                                                          |
| ---------------- | -------------------------------------------------------------------------------- |
| `sent`           | Mindestens eine sichtbare Plattformnachricht wurde zugestellt                              |
| `suppressed`     | Keine Plattformnachricht soll als fehlend gelten (durch Hook abgebrochen, Testlauf usw.) |
| `partial_failed` | Mindestens eine Nachricht wurde zugestellt, bevor ein späteres Payload oder ein Nebeneffekt fehlschlug      |
| `failed`         | Es wurde kein Plattformbeleg erzeugt                                                 |

Die Dauerhaftigkeit ist entweder `required`, `best_effort` oder `disabled`
(`MessageDurabilityPolicy` in `src/channels/message/types.ts`). `required`
schlägt sicher fehl, wenn die dauerhafte Absicht nicht geschrieben werden kann; `best_effort` weicht
auf einen direkten Versand aus, wenn die Persistenz nicht verfügbar ist; `disabled` behält das
Verhalten des direkten Versands vor dem Refactoring bei. Veraltete Kompatibilitätshilfen verwenden standardmäßig
`disabled` und leiten `required` nicht allein daraus ab, dass ein Kanal über einen generischen
Adapter für ausgehende Nachrichten verfügt.

Die weiterhin gefährliche Grenze liegt zwischen dem erfolgreichen Plattformaufruf und dem
Festschreiben des Belegs. Wenn der Prozess in diesem Zeitraum beendet wird, kann der Kern nicht wissen, ob die
Plattformnachricht vorhanden ist, sofern der Adapter nicht `reconcileUnknownSend` deklariert.
Dieser Hook klassifiziert einen unterbrochenen Versand als `sent`, `not_sent` oder
`unresolved`; nur `not_sent` erlaubt eine Wiederholung. Kanäle ohne Abgleich
fallen auf den Zustand `unknown_after_send` zurück (`src/channels/message/state.ts`,
`src/infra/outbound/delivery-queue-recovery.ts`) und dürfen sich nur dann für eine
Wiederholung mit mindestens einmaliger Zustellung entscheiden, wenn doppelte sichtbare Nachrichten für diesen Kanal
ein akzeptabler, dokumentierter Kompromiss sind.

### Empfangskontext

`createMessageReceiveContext` verfolgt den Bestätigungs-/Ablehnungszustand pro eingehendem Ereignis mit einer
idempotenten Funktion `ack()` und einer expliziten Funktion `nack(error)`. Die Bestätigungsrichtlinie
(`ChannelMessageReceiveAckPolicy`) ist eine der folgenden:

| Richtlinie                 | Bestätigt, wenn                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| `after_receive_record` | Der Kern genügend Metadaten der eingehenden Nachricht persistiert hat, um eine erneute Zustellung zu deduplizieren/weiterzuleiten                           |
| `after_agent_dispatch` | Der Agentenlauf gestartet wurde                                                             |
| `after_durable_send`   | Der dauerhafte ausgehende Versand für diesen Durchlauf festgeschrieben wurde                                             |
| `manual`               | Der Aufrufer den Bestätigungszeitpunkt explizit steuert (Standard für Adapter, die keine Richtlinie deklarieren) |

Telegram-Polling verwendet dies, um einen sicher abgeschlossenen Aktualisierungs-Wasserstand zu persistieren
(`safeCompletedUpdateId` in `extensions/telegram/src/bot-update-tracker.ts`):
grammY erfasst weiterhin jede Aktualisierung beim Eintritt in die Middleware-Kette, aber
OpenClaw verschiebt den persistierten Wasserstand für Neustarts nur über Aktualisierungen hinaus, deren
Weiterleitung abgeschlossen wurde, sodass fehlgeschlagene oder noch ausstehende Aktualisierungen nach einem Neustart wiederholt werden.
Der vorgelagerte `getUpdates`-Offset von Telegram liegt weiterhin in der Zuständigkeit von grammY; eine vollständig
dauerhafte Polling-Quelle, die die erneute Zustellung auf Plattformebene über diesen
Wasserstand hinaus steuert, wurde nicht erstellt (siehe Offene Fragen).

### Live-Vorschau

`src/channels/message/live.ts` modelliert Vorschau/Bearbeitung/Abschluss als einen einzigen Lebenszyklus:
`createLiveMessageState`, `markLiveMessagePreviewUpdated`,
`markLiveMessageFinalized`, `markLiveMessageCancelled` und
`deliverFinalizableLivePreviewAdapter` (eine endgültige Bearbeitung aus einem Entwurf erstellen, sie anwenden
und auf einen normalen Versand ausweichen, wenn die Bearbeitung nicht möglich ist oder fehlschlägt).
`LiveMessageState.phase` ist `idle | previewing | finalizing | finalized |
cancelled`; `canFinalizeInPlace` steuert, ob eine Vorschau durch Bearbeitung statt durch einen neuen Versand zur endgültigen
Nachricht werden kann.

### Dauerhafte Belege

`MessageReceipt` (`src/channels/message/types.ts`) normalisiert eine oder mehrere
Plattformnachrichten-IDs aus einem einzelnen logischen Versand in `platformMessageIds` sowie
`parts` je Teil (Art, Index, Thread-ID, Antwort-auf-ID). Eine primäre ID wird
für Threads und spätere Bearbeitungen beibehalten. Dadurch werden mehrteilige Zustellungen (Text
plus Medien, aufgeteilter Text, Karten-Fallback) nach einem
Neustart wiederholbar und deduplizierbar.

### Reduzierung des öffentlichen SDK

Das Refactoring integrierte oder verwarf: `reply-runtime`, `reply-dispatch-runtime`,
`reply-reference`, `reply-chunking`, als öffentliche
API bereitgestellte Hilfsfunktionen `reply-payload`, `inbound-reply-dispatch`, `channel-reply-pipeline` und die meisten öffentlichen Verwendungen
der alten Fassade für ausgehende Nachrichten. `src/plugin-sdk/channel-message.ts` ist jetzt ein
`@deprecated`-Reexport-Barrel, das auf `channel-outbound` /
`channel-inbound` verweist; die Laufzeit-Aliasse `channel.turn` wurden entfernt und die alte
Dokumentationsseite `/plugins/sdk-channel-turn` leitet zur
[API für eingehende Kanäle](/de/plugins/sdk-channel-inbound) weiter. Neuer Plugin-Code sollte
direkt auf `channel-outbound` und `channel-inbound` abzielen.

## Wo die Implementierung vom ursprünglichen Design abweicht

Der nachfolgende Designentwurf wurde nie exakt wie beschrieben ausgeliefert. Er bleibt zur
historischen Genauigkeit erhalten; behandeln Sie diese Typnamen nicht als aktuelle API.

- **Kein `MessageOrigin` / `shouldDropOpenClawEcho`.** Der ursprüngliche Plan sah
  ein `source: "openclaw"`-Ursprungs-Tag für Gateway-Fehlermeldungen sowie ein
  gemeinsames Prädikat vor, das markierte, vom Bot verfasste Echos in gemeinsam genutzten Räumen
  vor der Autorisierung durch `allowBots` verwirft. Dieser Typ und dieses Prädikat sind in
  der Codebasis nicht vorhanden. `allowBots` selbst ist ein realer kanalspezifischer Konfigurationsschlüssel (Slack,
  Discord, Google Chat und andere), der dafür vorgesehene Schutzmechanismus durch Ursprungs-Tags wurde jedoch
  nie implementiert. Die Unterdrückung von Echos bei Gateway-Fehlern in
  Räumen mit aktivierten Bots bleibt eine offene Lücke und ist keine ausgelieferte Garantie.
- **Kein einheitlicher Namespace `core.messages.receive/send/live/state`.** Die
  ausgelieferten Funktionen befinden sich direkt in `src/channels/message/*`
  (`withDurableMessageSendContext`, `createMessageReceiveContext`,
  `createLiveMessageState`, `classifyDurableSendRecoveryState`) statt
  hinter einer `core.messages.*`-Fassade.
- **Kein generischer normalisierter Nachrichtentyp `ChannelMessage` / `MessageTarget` / `MessageRelation`.**
  Der Kern übergibt weiterhin konkrete Antwort-Payloads
  (`ReplyPayload`) und kanalspezifische Kontexte über die Sendeadapter,
  statt eine einzige plattformneutrale Nachrichtenstruktur mit einer `kind: "reply" |
"followup" | "broadcast" | "system"`-Beziehung zu verwenden.
- **Die Namen der Bestätigungsrichtlinien unterscheiden sich vom Entwurf.** Ausgeliefert:
  `after_receive_record | after_agent_dispatch | after_durable_send | manual`.
  Der ursprüngliche Entwurf verwendete `immediate | after-record | after-durable-send |
manual` mit einem Feld für den Grund einer Webhook-Zeitüberschreitung; diese Struktur wurde nicht implementiert.
- **Die Fähigkeitsschlüssel `DurableFinalDeliveryRequirementMap` ersetzten das entworfene
  Objekt `MessageCapabilities`.** Fähigkeiten sind flache boolesche Flags (`text`,
  `media`, `poll`, `payload`, `silent`, `replyTo`, `thread`, `nativeQuote`,
  `messageSendingHooks`, `batch`, `reconcileUnknownSend`, `afterSendSuccess`,
  `afterCommit`), die durch `verifyDurableFinalCapabilityProofs` verifiziert werden,
  statt eine verschachtelte Struktur im Stil von `text.chunking` / `attachments.voice` zu bilden.

## Konkrete Migrationsrisiken (weiterhin relevant)

Diese kanalspezifischen Nebeneffekte bestanden bereits vor dem Refactoring und müssen über
die neuen Sendepfade weiterhin funktionieren. Sie sind nicht hypothetisch: Jeder einzelne ist
heute implementiert und für den Betrieb wesentlich.

- **iMessage** (`extensions/imessage/src/monitor/echo-cache.ts`,
  `persisted-echo-cache.ts`): Der Monitor erfasst gesendete Nachrichten nach einem erfolgreichen Sendevorgang in einem Echo-
  Cache. Dauerhafte abschließende Sendevorgänge müssen diesen Cache weiterhin befüllen,
  da OpenClaw andernfalls seine eigenen Antworten erneut als eingehende Benutzernachrichten aufnehmen kann.
- **Tlon** (`extensions/tlon/src/monitor/index.ts`): Fügt eine optionale Modell-
  signatur an und erfasst nach Gruppenantworten die Threads, an denen eine Beteiligung erfolgte. Die dauerhafte
  Zustellung darf diese Effekte nicht umgehen.
- **Discord und andere vorbereitete Dispatcher** verwalten die direkte Zustellung und
  das Vorschauverhalten bereits selbst. Ein Kanal ist nicht durchgängig dauerhaft, bis sein vorbereiteter
  Dispatcher abschließende Nachrichten ausdrücklich über den Sendekontext leitet; gehen Sie nicht davon aus,
  dass der generische Adapter allein dies abdeckt.
- **Die stille Fallback-Zustellung von Telegram** muss nach der Aufteilung/Fallback-
  Projektion das gesamte projizierte Payload-Array zustellen, nicht nur den ersten Payload.
- **LINE, Zalo, Nostr** und ähnliche Hilfspfade können die Verarbeitung von Antwort-Tokens,
  Medien-Proxying, Caches für gesendete Nachrichten oder reine Callback-Ziele umfassen.
  Sie verbleiben bei der kanaleigenen Zustellung, bis diese Semantik im
  Sendeadapter abgebildet und durch Tests abgedeckt ist.
- **Hilfsfunktionen für direkte DMs** können über einen Antwort-Callback verfügen, der das einzig korrekte
  Transportziel ist. Der generische ausgehende Versand darf kein Ziel aus rohen
  Plattformfeldern ableiten und diesen Callback umgehen.

## Fehlerklassifizierung

Adapter klassifizieren Transportfehler in geschlossene Kategorien im Stil von `DeliveryFailureKind`
(vorübergehend, Ratenbegrenzung, Authentifizierung, Berechtigung, nicht gefunden, ungültiger
Payload, Konflikt, abgebrochen, unbekannt). Kernrichtlinie:

- Vorübergehende Fehler und Ratenbegrenzungsfehler erneut versuchen.
- Fehler aufgrund ungültiger Payloads nicht erneut versuchen, sofern kein Rendering-Fallback vorhanden ist.
- Authentifizierungs- oder Berechtigungsfehler erst nach einer Konfigurationsänderung erneut versuchen.
- Bei „nicht gefunden“ darf die Live-Finalisierung vom Bearbeiten auf einen neuen Sendevorgang
  zurückfallen, wenn der Kanal dies als sicher deklariert.
- Bei einem Konflikt anhand des Empfangsbestätigungs-/Idempotenzstatus entscheiden, ob die Nachricht
  bereits vorhanden ist.
- Jeder Fehler, der auftritt, nachdem der Plattformaufruf möglicherweise erfolgreich war, aber bevor die Empfangsbestätigung
  festgeschrieben wurde, wird zu `unknown_after_send`, sofern der Adapter nicht nachweist, dass der Plattformvorgang
  nicht stattgefunden hat.

## Offene Fragen

- Ob Telegram den grammY-(`1.43.0`)-Polling-
  Runner letztlich durch eine vollständig dauerhafte Polling-Quelle ersetzen sollte, die die erneute Zustellung auf Plattformebene
  steuert und nicht nur die persistierte Neustart-Wassermarke von OpenClaw
  (`safeCompletedUpdateId`).
- Ob der Live-Vorschaustatus im selben Datensatz wie die Absicht des abschließenden Sendevorgangs
  oder in einem zugehörigen Live-Statusspeicher enthalten sein sollte.
- Ob die Echo-Unterdrückung bei Gateway-Fehlern in gemeinsam genutzten Räumen mit aktivierten Bots
  den ursprünglich geplanten Mechanismus zur Herkunftskennzeichnung oder einen einfacheren kanalbezogenen
  Vertrag benötigt oder nicht in den Umfang fällt.
- Welche Kanäle native Unterstützung für Herkunft/Metadaten zur botübergreifenden Echo-
  Unterdrückung bieten und welche stattdessen eine persistierte Registrierung ausgehender Nachrichten benötigen.

## Verwandte Themen

- [Nachrichten](/de/concepts/messages)
- [Streaming und Aufteilung](/de/concepts/streaming)
- [Fortschrittsentwürfe](/de/concepts/progress-drafts)
- [Richtlinie für Wiederholungsversuche](/de/concepts/retry)
- [API für ausgehende Kanalnachrichten](/de/plugins/sdk-channel-outbound)
- [API für eingehende Kanalnachrichten](/de/plugins/sdk-channel-inbound)
