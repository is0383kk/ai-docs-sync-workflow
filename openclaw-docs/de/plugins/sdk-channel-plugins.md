---
read_when:
    - Sie entwickeln ein neues Plugin für einen Messaging-Kanal
    - Sie möchten OpenClaw mit einer Messaging-Plattform verbinden
    - Sie müssen die Adapteroberfläche von ChannelPlugin verstehen
sidebarTitle: Channel Plugins
summary: Schritt-für-Schritt-Anleitung zum Erstellen eines Messaging-Kanal-Plugins für OpenClaw
title: Channel-Plugins erstellen
x-i18n:
    generated_at: "2026-07-26T19:28:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

Dieser Leitfaden erstellt ein Channel-Plugin, das OpenClaw mit einer Messaging-
Plattform verbindet: DM-Sicherheit, Kopplung, Antwort-Threads und ausgehende Nachrichten.

<Info>
  Neu bei OpenClaw-Plugins? Lesen Sie zuerst [Erste Schritte](/de/plugins/building-plugins),
  um mehr über die Paketstruktur und die Einrichtung des Manifests zu erfahren.
</Info>

## Zuständigkeiten Ihres Plugins

Channel-Plugins implementieren keine Tools zum Senden, Bearbeiten oder Reagieren; der Core stellt ein
gemeinsames `message`-Tool bereit. Ihr Plugin ist zuständig für:

- **Konfiguration** – Kontoauflösung und Einrichtungsassistent
- **Sicherheit** – DM-Richtlinie und Zulassungslisten
- **Kopplung** – DM-Genehmigungsablauf
- **Sitzungsgrammatik** – wie providerspezifische Konversations-IDs Basischats,
  Thread-IDs und übergeordneten Fallbacks zugeordnet werden
- **Ausgehend** – Senden von Text, Medien und Umfragen an die Plattform
- **Threading** – wie Antworten in Threads eingeordnet werden
- **Heartbeat-Tippanzeige** – optionale Tipp-/Beschäftigt-Signale für Heartbeat-Zustellungs-
  ziele

Der Core ist zuständig für das gemeinsame Nachrichtentool, die Prompt-Verdrahtung, die äußere Form des Sitzungsschlüssels,
die generische `:thread:`-Verwaltung und den Dispatch.

## Nachrichtenadapter

Stellen Sie einen `message`-Adapter mit `defineChannelMessageAdapter` aus
`openclaw/plugin-sdk/channel-outbound` bereit. Deklarieren Sie nur die dauerhaften Fähigkeiten für den endgültigen Versand,
die Ihr nativer Transport tatsächlich unterstützt, abgesichert durch einen Vertragstest,
der den nativen Nebeneffekt und die zurückgegebene Empfangsbestätigung nachweist. Leiten Sie Text-/Medien-
sendungen an dieselben Transportfunktionen weiter, die der ältere `outbound`-Adapter verwendet. Den
vollständigen API-Vertrag, die Fähigkeitsmatrix, Regeln für Empfangsbestätigungen, die Finalisierung von Live-Vorschauen,
die Richtlinie für Empfangsbestätigungen, Tests und die Migrationstabelle finden Sie unter
[API für ausgehende Channel-Nachrichten](/de/plugins/sdk-channel-outbound).

Wenn Ihr bestehender `outbound`-Adapter bereits über die richtigen Sendemethoden und
Fähigkeitsmetadaten verfügt, leiten Sie den `message`-Adapter mit
`createChannelMessageAdapterFromOutbound(...)` ab, anstatt eine weitere
Brücke manuell zu schreiben. Adapter-Sendevorgänge geben `MessageReceipt`-Werte zurück. Leiten Sie ältere IDs
mit `listMessageReceiptPlatformIds(...)` oder
`resolveMessageReceiptPrimaryId(...)` ab, anstatt parallele `messageIds`-
Felder beizubehalten.

Deklarieren Sie Live- und Finalisierungsfähigkeiten präzise – der Core entscheidet anhand dieser Angaben,
wozu ein Channel fähig ist, und Abweichungen zwischen dem deklarierten und dem tatsächlichen Verhalten führen zum
Fehlschlagen eines Vertragstests:

| Oberfläche                            | Werte                                                                                            |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`, `previewFinalization`, `progressUpdates`, `nativeStreaming`, `quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`, `normalFallback`, `discardPending`, `previewReceipt`, `retainOnAmbiguousFailure`    |

Channels, die einen Vorschauentwurf direkt finalisieren, sollten die Laufzeitlogik
über `defineFinalizableLivePreviewAdapter(...)` sowie
`deliverWithFinalizableLivePreviewAdapter(...)` leiten und die deklarierten
Fähigkeiten durch `verifyChannelMessageLiveCapabilityAdapterProofs(...)`-
und `verifyChannelMessageLiveFinalizerProofs(...)`-Tests absichern, damit das Verhalten nativer Vorschauen,
des Fortschritts, der Bearbeitung, des Fallbacks/der Aufbewahrung, der Bereinigung und der Empfangsbestätigungen nicht unbemerkt
abweichen kann.

Eingangsempfänger, die Plattformbestätigungen verzögern, sollten
`message.receive.defaultAckPolicy` und `supportedAckPolicies` deklarieren, anstatt den
Bestätigungszeitpunkt im lokalen Zustand des Monitors zu verbergen. Decken Sie jede deklarierte Richtlinie mit
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` ab.

Ältere Antworthelfer wie `dispatchInboundReplyWithBase` und
`recordInboundSessionAndDispatchReply` bleiben für kompatible
Dispatcher verfügbar. Verwenden Sie sie nicht für neuen Channel-Code; beginnen Sie stattdessen mit dem `message`-
Adapter, Empfangsbestätigungen und Hilfsfunktionen für den Empfangs-/Sendelebenszyklus unter
`openclaw/plugin-sdk/channel-outbound`.

### Eingehender Eingangspfad (experimentell)

Channels, die die eingehende Autorisierung migrieren, können den experimentellen
`openclaw/plugin-sdk/channel-ingress-runtime`-Unterpfad aus den Empfangspfaden der Laufzeit
verwenden. Er akzeptiert Plattformfakten, unaufbereitete Zulassungslisten, Routendeskriptoren, Befehls-
fakten und die Konfiguration von Zugriffsgruppen und gibt anschließend Projektionen für Absender, Route, Befehl und Aktivierung
sowie den geordneten Eingangsgraphen zurück, während Plattformsuche und Neben-
effekte im Plugin verbleiben. Behalten Sie die Normalisierung der Plugin-Identität in dem
Deskriptor bei, den Sie an den Resolver übergeben; serialisieren Sie keine unaufbereiteten Übereinstimmungswerte aus
dem aufgelösten Zustand oder der Entscheidung. Unter
[API für eingehende Channel-Nachrichten](/de/plugins/sdk-channel-ingress) finden Sie das API-Design,
die Zuständigkeitsgrenze und die Testerwartungen.

### Dauerhafter Eingang und Deduplizierung bei Wiederholungen

Channels, die einen dauerhaften Eingang einführen, sollten `createChannelIngressMonitor`
aus `openclaw/plugin-sdk/channel-outbound` verwenden, sofern sie keinen wesentlich
anderen Zulassungs- oder Pumpvertrag benötigen. Stellen Sie den unaufbereiteten Transportumschlag an einem
einzigen Empfangsengpass in die Warteschlange (keine Normalisierung zum Empfangszeitpunkt), knüpfen Sie bei Webhook-Transporten die
Transportbestätigung an das dauerhafte Anhängen, leiten Sie pro Konversation eine
serialisierte Lane ab und markieren Sie das Ereignis bei der Übernahme durch den Dispatch als abgeschlossen.
Der Primärschlüssel der Warteschlange ist `(queue_name, event_id)`, und beim Abschluss wird
die Zeile mit einem Tombstone versehen, anstatt sie zu löschen, sodass eine verspätete erneute Zustellung desselben
`event_id` durch die Plattform für das Aufbewahrungsfenster des Tombstones dauerhaft abgelehnt wird.
Unter [API für ausgehende Channel-Nachrichten](/de/plugins/sdk-channel-outbound#durable-ingress-monitors)
finden Sie die Monitor-API und den Vertrag für das Herunterfahren.

Dieser Tombstone ist die Schichtungsregel für Wiederholungsschutzmechanismen
(`openclaw/plugin-sdk/persistent-dedupe`): Ein geleerter Channel behält nur dann einen separaten
Wiederholungsschutz bei, wenn dessen Identität oder Aufbewahrungsdauer die der Warteschlange
übersteigt – ein logischer Nachrichtenschlüssel, der sich von der Transportzustellungs-ID unterscheidet (Telegram
dedupliziert `chat_id:message_id`, weil Debounce-Zusammenführungen eine Nachricht
unter einem neuen `update_id` erneut erscheinen lassen können), oder ein längeres Fenster als die Tombstone-
Aufbewahrung des Channels. Wenn Ihr Schutzschlüssel dem `event_id` des Drains entsprechen würde, löschen Sie den
Schutz bei der Einführung des Drains und dimensionieren Sie `completedTtlMs`/`completedMaxEntries`
stattdessen so, dass sie das alte Schutzfenster abdecken. Schutzmechanismen ohne Deduplizierungsfunktion wie Alters-
grenzen sind von dieser Regel unabhängig. Stabile IDs ausgehender Nachrichten verwenden die gemeinsame
Registrierung für ausgehende Echos aus `openclaw/plugin-sdk/channel-outbound` anstelle eines
Channel-lokalen TTL-Caches.

#### Transportklassen und Aufbewahrung

Klassifizieren Sie einen Transport anhand der Wiederherstellungsgarantie an seiner Empfangsgrenze:

- **Bestätigungsgebundene Webhook- oder Ereigniszustellung:** Bestätigen Sie die Zustellung oder geben Sie erst dann Erfolg zurück,
  nachdem das dauerhafte Anhängen abgeschlossen ist. Bei einem Fehler beim Anhängen muss die Zustellung weiterhin
  wiederholbar bleiben oder die Empfangsgrenze fehlschlagen. Zu dieser Klasse gehören Slack, SMS, Zalo,
  Microsoft Teams, Google Chat, LINE und Synology Chat.
- **Abgewartete Polling- oder Stream-Zustellung:** Bewegen Sie den entfernten Cursor erst weiter oder senden Sie die
  Transportbestätigung erst nach dem Anhängen. Wenn kein expliziter Cursor vorhanden ist, halten Sie den
  Empfangs-Callback serialisiert und warten Sie ihn ab, damit ein Fehler beim Anhängen die
  Empfangsschleife nicht vorauseilen lässt. Telegram-Polling, Signal und Tlon verwenden diese Klasse;
  die Telegram-Webhook-Zustellung folgt der obigen bestätigungsgebundenen Regel.
- **Sockets ohne Wiederholungsfunktion:** IRC, Mattermost, Twitch und Zalo Personal können die
  Plattform nicht auffordern, ein akzeptiertes Ereignis erneut zuzustellen. Ihre dauerhafte Warteschlange schützt das
  Zeitfenster eines Prozessabsturzes und unterstützt die lokale Wiederherstellung nach einem Neustart; Abschluss-
  Tombstones sind gegenüber Plattformwiederholungen nahezu wirkungslos.

Verwenden Sie 30 Tage als flotteweite Konvention für die Tombstone-TTL, nicht als SDK-Standard. Ein
Wiederholungsfenster mit hohem Volumen verwendet normalerweise eine Obergrenze von 20,000 abgeschlossenen Einträgen;
abgewartete Transporte und Transporte ohne Wiederholungsfunktion mit geringerem Volumen verwenden normalerweise 1,000-2,000.
Zu den aktuellen Ausnahmen gehören die Obergrenzen von LINE mit 4,096 Einträgen, die 24-stündige TTL für abgeschlossene
SMS-Einträge und die ausschließlich durch eine Obergrenze bestimmte Aufbewahrung abgeschlossener Tlon-Einträge. Obergrenzen für fehlgeschlagene Zeilen können ebenfalls niedriger
als Obergrenzen für abgeschlossene Zeilen sein. Sowohl TTL als auch Obergrenze bereinigen Zeilen, daher endet die effektive Aufbewahrung,
sobald die erste Grenze erreicht ist. Weichen Sie nur aufgrund eines dokumentierten Wiederholungshorizonts der Plattform,
eines beibehaltenen ausgelieferten Wiederholungsschutzfensters, des erwarteten Volumens oder Speicherbudgets
oder eines Transports ohne Wiederholungsfunktion davon ab und sichern Sie den Aufbewahrungsvertrag mit Tests ab.

#### Mindestens-einmal-Nebeneffekte

Der Drain-Dispatch führt Befehlsnebenwirkungen aus, bevor die Eingangszeile ihren
Abschluss-Tombstone erreicht. Ein Prozessabsturz zwischen diesen Schritten wiederholt die Zeile und
kann den Nebeneffekt erneut ausführen. Dieses Mindestens-einmal-Absturzfenster ist der
Standardvertrag. Verwenden Sie für nicht idempotente Arbeiten wie Konfigurationsschreibvorgänge, das
Leeren von Speichern oder sichtbare Bestätigungen außerhalb der Antwort-Lane
`createIngressEffectOnce(...)` aus
`openclaw/plugin-sdk/ingress-effect-once`. Übergeben Sie jedem Aufruf den stabilen Eingangs-
`eventId` sowie einen Effektnamen. Erstellen Sie einen Helfer pro Eingangswarteschlange/Konto und
verwenden Sie für diesen Geltungsbereich einen stabilen, eindeutigen `namespacePrefix`, da Transportereignis-
IDs möglicherweise nur innerhalb der Warteschlange eindeutig sind. Der Helfer schreibt seinen dauerhaften Anspruch erst fest, nachdem der
Effekt erfolgreich war; ein ausgelöster Effekt gibt den Anspruch frei, sodass ein Drain-Wiederholungsversuch
ihn erneut ausführen kann, während gleichzeitige Aufrufer auf den aktiven Anspruch warten. Fehler des dauerhaften
Zustands rufen `onDiskError` auf, sofern angegeben, und werden abgelehnt, anstatt auf den
Prozessspeicher zurückzufallen.

Setzen Sie `ttlMs` des Helfers mindestens auf die Tombstone-Aufbewahrung des Channels für den Eingang
zuzüglich der maximalen Verzögerung zwischen der Festschreibung des Effekts und dem Abschluss der Zeile, einschließlich
begrenzter Ausfallzeiten und Drain-Wiederholungsversuche. Die TTL des Effektdatensatzes beginnt bei der Festschreibung,
während die Tombstone-Aufbewahrung später beim Abschluss beginnt; wenn die Lebensdauer ausstehender Zeilen
unbegrenzt ist, kann keine endliche TTL beliebige Ausfallzeiten abdecken. Nachdem der Tombstone
die Zeile nicht mehr wiederholen kann, sind ältere Effektdatensätze nutzlos. Dimensionieren Sie
`stateMaxEntries` für jeden unterschiedlichen Ereignis-/Effektschlüssel, der in diesem
Aufbewahrungsfenster vorhanden sein kann, unter Berücksichtigung der Obergrenze abgeschlossener Einträge der Warteschlange und der
maximalen Anzahl von Effekten pro Ereignis. Eine niedrigere Obergrenze entfernt den ältesten Datensatz vor Ablauf seiner TTL
und ermöglicht die erneute Ausführung dieses Effekts. Verbleibende Mindestens-einmal-Fenster bestehen,
wenn der Prozess beendet wird oder die Persistierung fehlschlägt, nachdem der Effekt erfolgreich war, aber bevor
der Anspruch festgeschrieben wurde, oder wenn der Datensatz abläuft, während seine Eingangszeile noch
aussteht.

#### Kontobezogener Neustartvertrag

Änderungen an der Channel-Konfiguration starten standardmäßig den gesamten Channel neu. Ein Channel mit mehreren Konten
darf `reload.accountScopedRestart: true` nur setzen, wenn die Konfigurations-
auflösung channelweite gemeinsame Felder sowie das ausgewählte Konto liest, niemals ein
gleichgeordnetes Konto, und der Gateway eine `(channel, accountId)`-
Laufzeit stoppen und starten kann, ohne gleichgeordnete Laufzeiten zu ersetzen.

Der begrenzte Pfad gilt nur für Änderungen unter
`channels.<channel>.accounts.<non-default-id>.*`. Änderungen an gemeinsamen Channel-
Feldern, `accounts.default`, entfernten oder nicht auflösbaren Konten und gemischte Änderungen,
die sich auf die Vererbung auswirken können, werden zu einem Neustart des gesamten Channels hochgestuft. Plugins,
die diese Funktion nicht aktivieren, verwenden immer den Pfad für den gesamten Channel.

Bei Channels, die den dauerhaften Eingangs-Drain verwenden, muss der Stopppfad des Kontomonitors
zuerst alle akzeptierten Transportzulassungen abschließen und anschließend seinen
Drain freigeben und abwarten. Beim Start des Kontos wird dieselbe kontobezogene Warteschlange geöffnet, deren anfänglicher
Drain nicht versandte dauerhafte Zeilen wiederherstellt. Fügen Sie keinen zweiten, neuladespezifischen
Wiederholungsdurchlauf hinzu; die Warteschlangenwiederherstellung ist der kanonische Neustartpfad.

Behandeln Sie dieses Flag als Fähigkeitszusicherung, nicht als Leistungspräferenz. Vertrags-
tests sollten nachweisen, dass das Hinzufügen und Bearbeiten eines benannten Kontos die aufgelöste Konfiguration eines gleichgeordneten Kontos
unverändert lässt, das Stoppen eines Kontos nur den Monitor und Drain dieses Kontos
abschließt und ein neuer Monitor die Zeilen dieses Kontos genau
einmal wiederherstellt. Wenn eine dieser Garantien nicht nachgewiesen werden kann, lassen Sie das Flag weg.

### Tippanzeigen

Wenn Ihr Channel Tippanzeigen außerhalb eingehender Antworten unterstützt, stellen Sie
`heartbeat.sendTyping(...)` im Channel-Plugin bereit. Der Core ruft es mit dem
aufgelösten Heartbeat-Zustellungsziel auf, bevor der Heartbeat-Modelllauf beginnt, und
verwendet den gemeinsamen Lebenszyklus für die Aufrechterhaltung und Bereinigung der Tippanzeige. Fügen Sie
`heartbeat.clearTyping(...)` hinzu, wenn die Plattform ein explizites Stoppsignal benötigt.

### Parameter für Medienquellen

Wenn Ihr Channel dem Nachrichtentool Parameter hinzufügt, die Medienquellen enthalten, stellen Sie
diese Parameternamen über `plugin.actions.describeMessageTool(...).mediaSourceParams` bereit.
Der Core verwendet diese explizite Liste für die Normalisierung von Sandbox-Pfaden und die Zugriffsrichtlinie für ausgehende
Medien, sodass Plugins keine gemeinsam genutzten Core-Sonderfälle für
providerspezifische Avatar-, Anhangs- oder Titelbildparameter benötigen.

Bevorzugen Sie eine nach Aktionen gegliederte Map wie `{ "set-profile": ["avatarUrl", "avatarPath"] }`,
damit nicht zusammenhängende Aktionen nicht die Medienargumente einer anderen Aktion übernehmen. Ein flaches Array
funktioniert weiterhin für Parameter, die absichtlich von jeder bereitgestellten Aktion gemeinsam verwendet werden.

Kanäle, die für einen plattformseitigen Medienabruf eine temporäre öffentliche URL
bereitstellen müssen, können `createHostedOutboundMediaStore(...)` aus
`openclaw/plugin-sdk/outbound-media` mit Plugin-Zustandsspeichern verwenden. Belassen Sie das Parsen von Plattformrouten
und die Token-Durchsetzung im Kanal-Plugin; der gemeinsame Helper
ist nur für das Laden von Medien, Ablaufmetadaten, Chunk-Zeilen und die Bereinigung zuständig.

Eingehende Anhänge verwenden geordnete Fakten, keine parallelen `Media*`-Felder. Normalisieren Sie
Kanaldatensätze mit `toInboundMediaFacts(...)` aus
`openclaw/plugin-sdk/channel-inbound` und übergeben Sie sie beim Erstellen des
eingehenden Kontexts als `media`. Wenn ein Plugin lokale Medienlesevorgänge autorisieren muss, importieren Sie
`getAgentScopedMediaLocalRoots(...)` oder
`getAgentScopedMediaLocalRootsForSources(...)` aus dem gezielten
`openclaw/plugin-sdk/media-local-roots`-Unterpfad. Der alte
`agent-media-payload`-Builder bzw. die Root-Fassade dient nur noch der veralteten Kompatibilität.

### Formung nativer Payloads

Wenn Ihr Kanal eine Provider-spezifische Formung für `message(action="send")` benötigt,
bevorzugen Sie `actions.prepareSendPayload(...)`. Legen Sie native Karten, Blöcke, Einbettungen oder
andere dauerhafte Daten unter `payload.channelData.<channel>` ab und lassen Sie den Core
über den Outbound-/Nachrichtenadapter senden. Verwenden Sie `actions.handleAction(...)` zum Senden
nur als Kompatibilitäts-Fallback für Payloads, die nicht serialisiert und
erneut versucht werden können.

### Grammatik von Sitzungsunterhaltungen

Wenn Ihre Plattform zusätzlichen Geltungsbereich in Konversations-IDs speichert, belassen Sie dieses Parsen
mit `messaging.resolveSessionConversation(...)` im Plugin. Dies ist der
kanonische Hook für die Zuordnung von `rawId` zur Basis-Konversations-ID, einer optionalen
Thread-ID, einem expliziten `baseConversationId` und etwaigen
`parentConversationCandidates`. Wenn Sie `parentConversationCandidates` zurückgeben,
ordnen Sie sie vom engsten übergeordneten Element bis zur umfassendsten bzw. Basis-Konversation.

`messaging.resolveParentConversationCandidates(...)` ist ein veralteter
Kompatibilitäts-Fallback für Plugins, die nur übergeordnete Fallbacks zusätzlich zur
generischen bzw. unverarbeiteten ID benötigen. Wenn beide Hooks vorhanden sind, verwendet der Core zuerst
`resolveSessionConversation(...).parentConversationCandidates` und greift nur auf
`resolveParentConversationCandidates(...)` zurück, wenn der kanonische
Hook sie auslässt.

Gebündelte Plugins, die dasselbe Parsen benötigen, bevor die Kanalregistrierung startet,
können eine `session-key-api.ts`-Datei auf oberster Ebene mit einem passenden
`resolveSessionConversation(...)`-Export bereitstellen (siehe die Plugins für Feishu und Telegram).
Der Core verwendet diese bootstrap-sichere Oberfläche nur, wenn die Laufzeit-Plugin-
Registrierung noch nicht verfügbar ist.

Verwenden Sie `openclaw/plugin-sdk/channel-route`, wenn Plugin-Code routenähnliche
Felder normalisieren, einen untergeordneten Thread mit seiner übergeordneten Route vergleichen oder aus
`{ channel, to, accountId, threadId }` einen stabilen Deduplizierungsschlüssel erstellen muss. Der Helper
normalisiert numerische Thread-IDs genauso wie der Core; bevorzugen Sie ihn daher gegenüber ad-hoc
`String(threadId)`-Vergleichen. Plugins mit Provider-spezifischer Zielgrammatik
sollten `messaging.resolveOutboundSessionRoute(...)` bereitstellen, damit der Core
Provider-native Sitzungs- und Thread-Identitäten ohne Parser-Shims erhält.

### Unterstützung kontobezogener Konversationsbindungen

Setzen Sie `conversationBindings.supportsCurrentConversationBinding`, wenn der Kanal
generische Bindungen für die aktuelle Konversation unterstützt. `createChatChannelPlugin(...)`
setzt diese statische Fähigkeit standardmäßig auf `true`.

Wenn die Unterstützung je nach konfiguriertem Konto abweicht, implementieren Sie zusätzlich
`conversationBindings.isCurrentConversationBindingSupported({ accountId })`.
Der Core wertet diesen synchronen Hook erst aus, nachdem die statische Fähigkeit
aktiviert wurde. Die Rückgabe von `false` macht generische Funktionen für die aktuelle Konversation
zum Ermitteln von Fähigkeiten, Binden, Nachschlagen, Auflisten, Aktualisieren und Aufheben von Bindungen für dieses Konto nicht verfügbar.
Wird der Hook ausgelassen, gilt die statische Fähigkeit für jedes Konto.

Ermitteln Sie die Antwort aus der bereits geladenen Kontokonfiguration oder dem Laufzeitzustand. Dieser
Hook steuert nur generische Bindungen für die aktuelle Konversation; er ersetzt weder
konfigurierte Bindungsregeln noch das sitzungsbezogene Routing des Plugins. Vertragstests
sollten mindestens ein unterstütztes und ein nicht unterstütztes Konto über den von
`openclaw/plugin-sdk/channel-core` exportierten
`ChannelPlugin["conversationBindings"]`-Vertrag abdecken.

## Genehmigungen und Kanalfähigkeiten

Die meisten Kanal-Plugins benötigen keinen genehmigungsspezifischen Code. Der Core verwaltet
`/approve` im selben Chat, gemeinsame Payloads für Genehmigungsschaltflächen und die generische Fallback-Zustellung.
`ChannelPlugin.approvals` wurde entfernt; legen Sie Fakten zu Genehmigungszustellung, nativer Darstellung, Rendering und Autorisierung
stattdessen in einem einzigen `approvalCapability`-Objekt ab. `plugin.auth` dient nur
zur Anmeldung und Abmeldung – der Core liest aus diesem Objekt keine Hooks zur Genehmigungsautorisierung mehr.

Verwenden Sie `approvalCapability.delivery` nur für natives Genehmigungsrouting oder die Unterdrückung von Fallbacks
und `approvalCapability.render` nur, wenn ein Kanal tatsächlich
benutzerdefinierte Genehmigungs-Payloads anstelle des gemeinsamen Renderers benötigt.

### Genehmigungsautorisierung

- `approvalCapability.authorizeActorAction` und
  `approvalCapability.getActionAvailabilityState` bilden die kanonische
  Schnittstelle für die Genehmigungsautorisierung.
- Verwenden Sie `getActionAvailabilityState` für die Verfügbarkeit der Genehmigungsautorisierung im selben Chat.
  Halten Sie konfigurierte Genehmigende für `/approve` verfügbar, auch wenn die native Zustellung
  deaktiviert ist; verwenden Sie stattdessen den Zustand der nativen auslösenden Oberfläche für Hinweise
  zur Zustellung und Einrichtung.
- Wenn Ihr Kanal native Ausführungsgenehmigungen bereitstellt, verwenden Sie
  `approvalCapability.getExecInitiatingSurfaceState` für den
  Zustand der auslösenden Oberfläche bzw. des nativen Clients, wenn dieser von der Genehmigungsautorisierung
  im selben Chat abweicht. Der Core verwendet diesen ausführungsspezifischen Hook, um zwischen `enabled` und
  `disabled` zu unterscheiden, zu entscheiden, ob der auslösende Kanal native Ausführungsgenehmigungen
  unterstützt, und den Kanal in die Fallback-Hinweise für native Clients aufzunehmen.
  `createApproverRestrictedNativeApprovalCapability(...)` füllt dies für
  den üblichen Fall aus.
- Wenn ein Kanal aus der vorhandenen Konfiguration stabile, inhaberähnliche DM-Identitäten ableiten kann,
  verwenden Sie `createResolvedApproverActionAuthAdapter` aus
  `openclaw/plugin-sdk/approval-runtime`, um `/approve` im selben Chat einzuschränken,
  ohne genehmigungsspezifische Core-Logik hinzuzufügen.
- Wenn eine benutzerdefinierte Genehmigungsautorisierung absichtlich nur einen Fallback im selben Chat zulässt, geben Sie
  `markImplicitSameChatApprovalAuthorization({ authorized: true })` aus
  `openclaw/plugin-sdk/approval-auth-runtime` zurück; andernfalls behandelt der Core das
  Ergebnis als explizite Autorisierung eines Genehmigenden.
- Wenn ein kanaleigener nativer Callback Genehmigungen direkt auflöst, verwenden Sie vor dem Auflösen
  `isImplicitSameChatApprovalAuthorization(...)`, damit der implizite
  Fallback weiterhin die normale Akteursautorisierung des Kanals durchläuft.

### Payload-Lebenszyklus und Einrichtungshinweise

- Verwenden Sie `outbound.shouldSuppressLocalPayloadPrompt` oder
  `outbound.beforeDeliverPayload` für kanalspezifisches Verhalten im Payload-Lebenszyklus,
  etwa zum Ausblenden doppelter lokaler Genehmigungsaufforderungen oder zum Senden von Tippindikatoren
  vor der Zustellung.
- Verwenden Sie `approvalCapability.describeExecApprovalSetup`, wenn der Kanal möchte,
  dass die Antwort für den deaktivierten Pfad die genauen Konfigurationsoptionen erläutert, die zum Aktivieren
  nativer Ausführungsgenehmigungen erforderlich sind. Der Hook empfängt `{ channel, channelLabel, accountId }`;
  Kanäle mit benannten Konten sollten kontobezogene Pfade wie
  `channels.<channel>.accounts.<id>.execApprovals.*` statt allgemeiner
  Standardwerte auf oberster Ebene darstellen.
- Verwenden Sie `approvalCapability.describePluginApprovalSetup`, wenn Hinweise zu
  Fehlern bei Plugin-Genehmigungen bei Ausfällen aufgrund fehlender Routen oder Zeitüberschreitungen gefahrlos angezeigt werden können.
  `createApproverRestrictedNativeApprovalCapability(...)` leitet dies nicht
  aus `describeExecApprovalSetup` ab; übergeben Sie denselben Helper nur dann explizit,
  wenn Plugin- und Ausführungsgenehmigungen tatsächlich dieselbe native Einrichtung verwenden.

### Native Genehmigungszustellung

Wenn ein Kanal eine native Genehmigungszustellung benötigt, konzentrieren Sie den Kanalcode auf
die Zielnormalisierung sowie Fakten zu Transport und Darstellung. Verwenden Sie
`createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`,
`createChannelApproverDmTargetResolver` und
`createApproverRestrictedNativeApprovalCapability` aus
`openclaw/plugin-sdk/approval-runtime`. Legen Sie die kanalspezifischen Fakten hinter
`approvalCapability.nativeRuntime` ab, idealerweise über
`createChannelApprovalNativeRuntimeAdapter(...)` oder
`createLazyChannelApprovalNativeRuntimeAdapter(...)`, damit der Core den
Handler zusammensetzen und Anforderungsfilterung, Routing, Deduplizierung, Ablauf, Gateway-
Abonnements und Hinweise auf anderweitiges Routing verwalten kann.

`nativeRuntime` ist in einige kleinere Schnittstellen aufgeteilt:

- `availability` – ob das Konto konfiguriert ist und ob eine Anfrage
  verarbeitet werden soll
- `presentation` – ordnet das gemeinsame Ansichtsmodell für Genehmigungen
  ausstehenden, aufgelösten oder abgelaufenen nativen Payloads beziehungsweise abschließenden Aktionen zu
- `transport` – bereitet Ziele vor und sendet, aktualisiert oder löscht native
  Genehmigungsnachrichten
- `interactions` – optionale Hooks zum Binden, Aufheben von Bindungen und Löschen von Aktionen für native Schaltflächen
  oder Reaktionen sowie ein optionaler `cancelDelivered`-Hook. Implementieren Sie
  `cancelDelivered`, wenn `deliverPending` prozessinternen oder persistenten
  Zustand registriert (etwa einen Speicher für Reaktionsziele), damit dieser Zustand freigegeben werden kann, wenn
  das Stoppen eines Handlers die Zustellung abbricht, bevor `bindPending` ausgeführt wird, oder wenn
  `bindPending` kein Handle zurückgibt
- `observe` – optionale Hooks für Zustellungsdiagnosen

Weitere Helper für Genehmigungen:

- Verwenden Sie `createNativeApprovalChannelRouteGates` aus
  `openclaw/plugin-sdk/approval-native-runtime`, wenn ein Kanal sowohl
  sitzungsursprüngliche native Zustellung als auch explizite Weiterleitungsziele für Genehmigungen unterstützt. Der
  Helper zentralisiert die Auswahl der Genehmigungskonfiguration, die Verarbeitung von `mode`, Agenten-/Sitzungsfilter,
  Kontobindung, den Abgleich von Sitzungszielen und den Abgleich von Ziellisten,
  während die Aufrufer weiterhin für Kanal-ID, standardmäßigen Weiterleitungsmodus, Kontosuche,
  Prüfung der Transportaktivierung, Zielnormalisierung und die Auflösung des turnbezogenen
  Quellziels zuständig sind. Verwenden Sie ihn nicht, um kanalspezifische, vom Core verwaltete Richtlinienstandardwerte
  zu erstellen; übergeben Sie den dokumentierten Standardmodus des Kanals explizit.
- `createChannelNativeOriginTargetResolver` verwendet standardmäßig den gemeinsamen Kanalrouten-
  Matcher für `{ to, accountId, threadId }`-Ziele. Übergeben Sie
  `targetsMatch` nur, wenn ein Kanal Provider-spezifische Äquivalenzregeln besitzt,
  beispielsweise den Abgleich von Slack-Zeitstempelpräfixen. Übergeben Sie `normalizeTargetForMatch`, wenn
  der Kanal Provider-IDs kanonisieren muss, bevor der standardmäßige Routen-
  Matcher oder ein benutzerdefinierter `targetsMatch`-Callback ausgeführt wird, während das
  ursprüngliche Ziel für die Zustellung erhalten bleibt. Verwenden Sie `normalizeTarget` nur, wenn das aufgelöste
  Zustellungsziel selbst kanonisiert werden soll.
- Wenn der Kanal laufzeiteigene Objekte wie einen Client, ein Token, eine Bolt-
  App oder einen Webhook-Empfänger benötigt, registrieren Sie sie über
  `openclaw/plugin-sdk/channel-runtime-context`. Die generische Laufzeitkontext-
  Registrierung ermöglicht dem Core, fähigkeitsgesteuerte Handler aus dem Kanalstartzustand
  zu initialisieren, ohne genehmigungsspezifischen Wrapper-Verbindungscode hinzuzufügen.
- Greifen Sie nur dann auf die niedrigeren Ebenen `createChannelApprovalHandler` oder
  `createChannelNativeApprovalRuntime` zurück, wenn die fähigkeitsgesteuerte Schnittstelle
  noch nicht ausdrucksstark genug ist.
- Kanäle für native Genehmigungen müssen sowohl `accountId` als auch `approvalKind`
  über diese Helper routen. `accountId` beschränkt Genehmigungsrichtlinien für mehrere Konten
  auf das richtige Bot-Konto, und `approvalKind` hält das Verhalten von Ausführungs- gegenüber Plugin-
  Genehmigungen für den Kanal verfügbar, ohne fest codierte Verzweigungen im
  Core.
- Der Core verwaltet auch Hinweise zur Umleitung von Genehmigungen. Kanal-Plugins sollten
  keine eigenen Folgenachrichten wie „Genehmigung wurde an DMs/einen anderen Kanal gesendet“ aus
  `createChannelNativeApprovalRuntime` senden; stellen Sie stattdessen präzises Routing für Ursprung und
  Genehmigenden-DMs über die gemeinsamen Helper für Genehmigungsfähigkeiten bereit und lassen Sie den
  Core die tatsächlichen Zustellungen aggregieren, bevor ein Hinweis an den
  auslösenden Chat zurückgesendet wird.
- Bewahren Sie die Art der zugestellten Genehmigungs-ID durchgängig. Native Clients sollten
  das Routing von Ausführungs- gegenüber Plugin-Genehmigungen nicht anhand des kanallokalen
  Zustands erraten oder umschreiben.
- Übergeben Sie diesen expliziten `approvalKind` an `resolveApprovalOverGateway`. Dadurch wird
  der kanonische `approval.resolve`-Dienst verwendet und der aufgezeichnete Gewinner zurückgegeben, wenn
  eine andere Oberfläche zuerst antwortet. Die ältere explizite `resolveMethod`-Eingabe
  bleibt für befehlsbasierte Steuerelemente bestehen; neue native Aktionen dürfen sie nicht verwenden oder
  die Art aus einer ID ableiten.
- Verschiedene Genehmigungsarten können absichtlich unterschiedliche native
  Oberflächen bereitstellen. Aktuelle gebündelte Beispiele: Matrix behält dasselbe native DM-/Kanal-
  Routing und dieselbe Reaktions-UX für Ausführungs- und Plugin-Genehmigungen bei, während sich
  die Autorisierung weiterhin je nach Genehmigungsart unterscheiden kann; Slack hält natives Genehmigungsrouting
  sowohl für Ausführungs- als auch für Plugin-IDs verfügbar.
- `createApproverRestrictedNativeApprovalAdapter` besteht weiterhin als
  Kompatibilitäts-Wrapper, neuer Code sollte jedoch den Capability-Builder bevorzugen
  und `approvalCapability` auf dem Plugin bereitstellen.

### Engere Unterpfade der Genehmigungslaufzeit

Bevorzugen Sie für häufig aufgerufene Kanaleinstiegspunkte diese engeren Unterpfade gegenüber dem breiteren
`approval-runtime`-Barrel, wenn Sie nur einen Teil dieser Familie benötigen:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Bevorzugen Sie entsprechend `openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` und
`openclaw/plugin-sdk/reply-chunking` gegenüber breiteren übergeordneten Schnittstellen, wenn Sie
nicht alle benötigen.

### Unterpfade für die Einrichtung

- `openclaw/plugin-sdk/setup-runtime` umfasst die laufzeitsicheren Einrichtungshilfen:
  `createSetupTranslator`, importsichere Adapter für Einrichtungspatches
  (`createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), die Ausgabe von Suchhinweisen,
  `promptResolvedAllowFrom`, `splitSetupEntries` und die delegierten
  Builder für Einrichtungs-Proxys.
- `openclaw/plugin-sdk/channel-setup` umfasst die Einrichtungs-Builder für optionale Installationen
  sowie einige einrichtungssichere Primitive: `createOptionalChannelSetupSurface`,
  `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`,
  `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`,
  `setSetupChannelEnabled` und `splitSetupEntries`.
- Verwenden Sie die breitere Schnittstelle `openclaw/plugin-sdk/setup` nur, wenn Sie auch
  die umfangreicheren gemeinsamen Einrichtungs-/Konfigurationshilfen wie
  `moveSingleAccountChannelSectionToDefaultAccount(...)` benötigen.

Wenn Ihr Kanal auf Einrichtungsoberflächen nur darauf hinweisen soll, „zuerst dieses Plugin zu installieren“,
bevorzugen Sie `createOptionalChannelSetupSurface(...)`. Der generierte
Adapter/Assistent verweigert Konfigurationsschreibvorgänge und den Abschluss sicher
und verwendet dieselbe Meldung über die erforderliche Installation für Validierung, Abschluss
und den Text des Dokumentationslinks.

Wenn Ihr Kanal eine umgebungsvariablengesteuerte Einrichtung oder Authentifizierung unterstützt, stellen Sie diese über das
Kanalkonfigurationsschema und die Einrichtungsdeskriptoren bereit. Verwenden Sie die Kanallaufzeit `envVars` oder
lokale Konstanten nur für an Betreiber gerichtete Texte.

Wenn Ihr Kanal in `status`, `channels list`, `channels status` oder
SecretRef-Scans erscheinen kann, bevor die Plugin-Laufzeit startet, fügen Sie `openclaw.setupEntry` in
`package.json` hinzu. Dieser Einstiegspunkt sollte sich in schreibgeschützten Befehlspfaden
sicher importieren lassen und die Kanalmetadaten, den einrichtungssicheren Konfigurationsadapter,
den Statusadapter und die Metadaten zu den geheimen Kanalzielen zurückgeben, die für diese
Zusammenfassungen erforderlich sind. Starten Sie über den Einrichtungseinstieg keine Clients,
Listener oder Transportlaufzeiten.

Halten Sie auch den Importpfad des Hauptkanaleinstiegs schlank. Die Erkennung kann
den Einstieg und das Kanal-Plugin-Modul auswerten, um Fähigkeiten zu registrieren, ohne
den Kanal zu aktivieren. Dateien wie `channel-plugin-api.ts` sollten
das Kanal-Plugin-Objekt exportieren, ohne Einrichtungsassistenten, Transport-
Clients, Socket-Listener, Starter für Unterprozesse oder Module für den Dienststart zu importieren.
Platzieren Sie diese Laufzeitbestandteile in Modulen, die über `registerFull(...)`, Laufzeit-
Setter oder verzögert geladene Fähigkeitsadapter geladen werden.

### Weitere schlanke Kanalunterpfade

Bevorzugen Sie für andere häufig genutzte Kanalpfade die schlanken Hilfen gegenüber breiteren älteren
Schnittstellen:

- `openclaw/plugin-sdk/account-core`, `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` und
  `openclaw/plugin-sdk/account-helpers` für Konfigurationen mit mehreren Konten und
  den Rückgriff auf das Standardkonto
- `openclaw/plugin-sdk/inbound-envelope` und
  `openclaw/plugin-sdk/channel-inbound` für eingehende Routen/Umschläge sowie die
  Verkabelung zum Aufzeichnen und Weiterleiten
- `openclaw/plugin-sdk/channel-targets` für Hilfen zum Parsen von Zielen
- `openclaw/plugin-sdk/channel-outbound` für Delegaten für ausgehende Identitäten/Sendungen
  und die typisierte Nutzlastplanung
- `buildThreadAwareOutboundSessionRoute(...)` aus
  `openclaw/plugin-sdk/channel-core`, wenn eine ausgehende Route eine explizite
  `replyToId`/`threadId` beibehalten oder die aktuelle `:thread:`-
  Sitzung wiederherstellen soll, nachdem der Basissitzungsschlüssel weiterhin übereinstimmt. Provider-Plugins können
  Priorität, Suffixverhalten und die Normalisierung der Thread-ID überschreiben, wenn
  ihre Plattform über native Semantik für die Thread-Zustellung verfügt.
- `openclaw/plugin-sdk/thread-bindings-runtime` für den Lebenszyklus der Thread-Bindung
  und die Adapterregistrierung

Kanäle, die ausschließlich der Authentifizierung dienen, können in der Regel beim Standardpfad bleiben: Der Kern übernimmt
Genehmigungen, und das Plugin stellt lediglich Fähigkeiten für ausgehende Vorgänge und Authentifizierung bereit. Native
Genehmigungskanäle wie Matrix, Slack, Telegram und benutzerdefinierte Chat-Transporte
sollten die gemeinsamen nativen Hilfen verwenden, statt einen eigenen Genehmigungslebenszyklus
zu implementieren.

## Richtlinie für Erwähnungen in eingehenden Nachrichten

Teilen Sie die Verarbeitung eingehender Erwähnungen in zwei Ebenen auf:

- Erfassung von Nachweisen im Besitz des Plugins
- gemeinsame Richtlinienauswertung

Verwenden Sie `openclaw/plugin-sdk/channel-mention-gating` für Entscheidungen der Erwähnungsrichtlinie.
Verwenden Sie `openclaw/plugin-sdk/channel-inbound` nur, wenn Sie das breitere
Hilfsmodul für eingehende Nachrichten benötigen.

Gut für Plugin-lokale Logik geeignet:

- Erkennung von Antworten an den Bot
- Erkennung zitierter Bot-Nachrichten
- Prüfungen der Thread-Teilnahme
- Ausschlüsse von Dienst-/Systemnachrichten
- plattformnative Caches, die zum Nachweis der Bot-Teilnahme erforderlich sind

Gut für die gemeinsame Hilfe geeignet:

- `requireMention`
- Ergebnis der expliziten Erwähnung
- Positivliste für implizite Erwähnungen
- Umgehung für Befehle
- endgültige Entscheidung zum Überspringen

Bevorzugter Ablauf:

1. Berechnen Sie lokale Fakten zu Erwähnungen.
2. Übergeben Sie diese Fakten an `resolveInboundMentionDecision({ facts, policy })`.
3. Verwenden Sie `decision.effectiveWasMentioned`, `decision.shouldBypassMention` und
   `decision.shouldSkip` in Ihrem Gate für eingehende Nachrichten.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` gibt einen booleschen Wert zurück. `hasAnyMention`,
`isExplicitlyMentioned` und `canResolveExplicit` stammen aus den eigenen
nativen Erwähnungsmetadaten des Kanals (Nachrichtenentitäten, Markierungen für Antworten an den Bot und Ähnliches);
geben Sie `false`/`undefined`-Werte an, wenn Ihre Plattform diese nicht erkennen kann.

`api.runtime.channel.mentions` stellt dieselben gemeinsamen Hilfen für Erwähnungen für
gebündelte Kanal-Plugins bereit, die bereits von Laufzeitinjektion abhängen:
`buildMentionRegexes`, `matchesMentionPatterns`, `matchesMentionWithExplicit`,
`implicitMentionKindWhen`, `resolveInboundMentionDecision`.

Wenn Sie nur `implicitMentionKindWhen` und `resolveInboundMentionDecision` benötigen,
importieren Sie aus `openclaw/plugin-sdk/channel-mention-gating`, um das Laden
nicht zugehöriger Laufzeithilfen für eingehende Nachrichten zu vermeiden.

## Anleitung

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paket und Manifest">
    Erstellen Sie die standardmäßigen Plugin-Dateien. Das Feld `channels` in
    `openclaw.plugin.json` (nicht ein Feld `kind`) kennzeichnet ein Manifest als
    Besitzer eines Kanals. Die vollständige Oberfläche für Paketmetadaten finden Sie unter
    [Plugin-Einrichtung und -Konfiguration](/de/plugins/sdk-setup#openclaw-channel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "OpenClaw mit Acme Chat verbinden."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Kanal-Plugin für Acme Chat",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "Bot-Token",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema` validiert `plugins.entries.acme-chat.config`. Verwenden Sie es für
    Plugin-eigene Einstellungen, die nicht zur Kanalkontokonfiguration gehören.
    `channelConfigs.acme-chat.schema` validiert `channels.acme-chat` und ist die
    Quelle für selten ausgeführte Pfade, die vom Konfigurationsschema, der Einrichtung und den UI-Oberflächen verwendet wird, bevor die
    Plugin-Laufzeit geladen wird. Die vollständige Referenz der Felder auf oberster Ebene finden Sie unter [Plugin-Manifest](/de/plugins/manifest).

  </Step>

  <Step title="Kanal-Plugin-Objekt erstellen">
    Die Schnittstelle `ChannelPlugin` verfügt über viele optionale Adapteroberflächen. Beginnen Sie mit
    dem Minimum – `id`, `config` und `setup` – und fügen Sie Adapter nach
    Bedarf hinzu.

    Erstellen Sie `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    Verwenden Sie für Kanäle, die sowohl kanonische DM-Schlüssel auf oberster Ebene als auch ältere verschachtelte Schlüssel akzeptieren, die Hilfsfunktionen aus `plugin-sdk/channel-config-helpers`: `resolveChannelDmAccess`, `resolveChannelDmPolicy`, `resolveChannelDmAllowFrom` und `normalizeChannelDmPolicy` sorgen dafür, dass kontolokale Werte Vorrang vor geerbten Stammwerten haben. Koppeln Sie denselben Resolver über `normalizeLegacyDmAliases` mit der Doctor-Reparatur, damit Laufzeit und Migration denselben Vertrag lesen.

    <Accordion title="Was createChatChannelPlugin für Sie erledigt">
      Statt Low-Level-Adapter-Schnittstellen manuell zu implementieren, übergeben Sie
      deklarative Optionen, die der Builder zusammensetzt:

      | Option | Was sie verknüpft |
      | --- | --- |
      | `security.dm` | Bereichsbezogener DM-Sicherheits-Resolver aus Konfigurationsfeldern |
      | `pairing.text` | Textbasierter DM-Kopplungsablauf mit Codeaustausch |
      | `threading` | Resolver für den Antwortmodus (fest, kontobezogen oder benutzerdefiniert) |
      | `outbound.attachedResults` | Sendefunktionen, die Ergebnismetadaten (Nachrichten-IDs) zurückgeben; erfordert eine gleichgeordnete `channel`-ID, damit der Kern das zurückgegebene Zustellergebnis kennzeichnen kann |

      Sie können statt der deklarativen Optionen auch unverarbeitete Adapterobjekte
      übergeben, wenn Sie vollständige Kontrolle benötigen.

      Unverarbeitete ausgehende Adapter können eine `chunker(text, limit, ctx)`-Funktion definieren.
      Die optionale `ctx.formatting` enthält Formatierungsentscheidungen zum Zustellzeitpunkt,
      beispielsweise `maxLinesPerMessage`; wenden Sie sie vor dem Senden an, damit Antwort-Threading
      und Segmentgrenzen durch die gemeinsame ausgehende Zustellung nur einmal aufgelöst werden.
      Sendekontexte enthalten außerdem `replyToIdSource` (`implicit` oder `explicit`),
      wenn ein natives Antwortziel aufgelöst wurde, sodass Payload-Hilfsfunktionen
      explizite Antwort-Tags beibehalten können, ohne einen impliziten, einmalig verwendbaren Antwortplatz zu verbrauchen.
    </Accordion>

    ### Adapter für Gruppen-Tool-Richtlinien

    Ein Kanal, der `group.resolveToolPolicy` implementiert und
    `toolsBySender` unterstützt, muss den vollständigen `ChannelGroupContext` an seinen
    gemeinsam genutzten Richtlinien-Resolver weiterleiten. Berücksichtigen Sie insbesondere `senderPolicyMode: "never"`,
    indem Sie senderspezifische Überlagerungen sowohl im Bereich der übereinstimmenden Gruppe als auch im
    Platzhalterbereich überspringen, während die grundlegende `tools`-Richtlinie weiterhin angewendet wird.

    OpenClaw aktiviert diesen Modus nur für vertrauenswürdige Ausführungen außerhalb des Eingangswegs, bei denen die
    Autorität des Absenders bereits in einem serverseitig verwalteten Umschlag erfasst wurde, beispielsweise bei einer
    ausdrücklich begrenzten geplanten Ausführung. Plugins dürfen den Modus nicht aus
    eingehenden Metadaten ableiten, ihn als Kanalzustand speichern oder als Konfiguration verfügbar machen. Fügen Sie
    einen Adaptertest hinzu, der nachweist, dass der Modus einen Platzhalter-Eintrag `toolsBySender`
    überspringt, ohne die passende grundlegende Einschränkung `tools` zu verwerfen.

  </Step>

  <Step title="Einstiegspunkt verdrahten">
    Erstellen Sie `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme-Chat-Kanal-Plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme-Chat-Verwaltung");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme-Chat-Verwaltung",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Legen Sie kanaleigene CLI-Deskriptoren in `registerCliMetadata(...)` ab, damit OpenClaw
    sie in der Hilfe auf der obersten Ebene anzeigen kann, ohne die vollständige Kanallaufzeit zu aktivieren,
    während normale vollständige Ladevorgänge weiterhin dieselben Deskriptoren für die tatsächliche Befehlsregistrierung
    übernehmen. Behalten Sie `registerFull(...)` für reine Laufzeitaufgaben bei.
    `defineChannelPluginEntry` verarbeitet die Aufteilung nach Registrierungsmodus automatisch.
    Wenn `registerFull(...)` Gateway-RPC-Methoden registriert, verwenden Sie ein
    Plugin-spezifisches Präfix. Die administrativen Kern-Namensräume (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) bleiben reserviert und werden immer
    in `operator.admin` aufgelöst. Alle Optionen finden Sie unter
    [Einstiegspunkte](/de/plugins/sdk-entrypoints#definechannelpluginentry).

  </Step>

  <Step title="Setup-Einstieg hinzufügen">
    Erstellen Sie `setup-entry.ts` für leichtgewichtiges Laden während des Onboardings:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw lädt diesen Einstiegspunkt anstelle des vollständigen Einstiegspunkts, wenn der Kanal deaktiviert
    oder nicht konfiguriert ist. Dadurch wird vermieden, dass während der Einrichtungsabläufe umfangreicher Laufzeitcode geladen wird.
    Weitere Informationen finden Sie unter [Einrichtung und Konfiguration](/de/plugins/sdk-setup#setup-entry).

    Gebündelte Workspace-Kanäle, die einrichtungssichere Exporte in Sidecar-
    Module aufteilen, können `defineBundledChannelSetupEntry(...)` aus
    `openclaw/plugin-sdk/channel-entry-contract` verwenden, wenn sie außerdem einen
    expliziten Laufzeit-Setter für die Einrichtungsphase benötigen.

  </Step>

  <Step title="Eingehende Nachrichten verarbeiten">
    Ihr Plugin muss Nachrichten von der Plattform empfangen und an
    OpenClaw weiterleiten. Das typische Muster ist ein Webhook, der die Anfrage überprüft und
    über den Handler Ihres Kanals für eingehende Nachrichten weiterleitet:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // vom Plugin verwaltete Authentifizierung (Signaturen selbst überprüfen)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Ihr Handler für eingehende Nachrichten leitet die Nachricht an OpenClaw weiter.
          // Die genaue Verkabelung hängt von Ihrem Plattform-SDK ab –
          // ein reales Beispiel finden Sie im gebündelten Plugin-Paket für Microsoft Teams oder Google Chat.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      Die Verarbeitung eingehender Nachrichten ist kanalspezifisch. Jedes Kanal-Plugin ist für
      seine eigene Pipeline für eingehende Nachrichten verantwortlich. Reale Muster finden Sie in gebündelten Kanal-Plugins
      (zum Beispiel im Plugin-Paket für Microsoft Teams oder Google Chat).
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Testen">
Schreiben Sie Tests direkt in `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat-Plugin", () => {
      it("löst das Konto aus der Konfiguration auf", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("prüft das Konto, ohne Geheimnisse zu materialisieren", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("meldet eine fehlende Konfiguration", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    Informationen zu gemeinsamen Test-Hilfsfunktionen finden Sie unter [Testen](/de/plugins/sdk-testing).

</Step>
</Steps>

## Dateistruktur

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel-Metadaten
├── openclaw.plugin.json      # Manifest mit Konfigurationsschema
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Öffentliche Exporte (optional)
├── runtime-api.ts            # Interne Laufzeit-Exporte (optional)
└── src/
    ├── channel.ts            # ChannelPlugin über createChatChannelPlugin
    ├── channel.test.ts       # Tests
    ├── client.ts             # Plattform-API-Client
    └── runtime.ts            # Laufzeitspeicher (falls erforderlich)
```

## Fortgeschrittene Themen

<CardGroup cols={2}>
  <Card title="Threading-Optionen" icon="git-branch" href="/de/plugins/sdk-entrypoints#registration-mode">
    Feste, kontobezogene oder benutzerdefinierte Antwortmodi
  </Card>
  <Card title="Integration des Nachrichtenwerkzeugs" icon="puzzle" href="/de/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool und Aktionserkennung
  </Card>
  <Card title="Zielauflösung" icon="crosshair" href="/de/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType, looksLikeId, reservedLiterals, resolveTarget
  </Card>
  <Card title="Runtime-Hilfsfunktionen" icon="settings" href="/de/plugins/sdk-runtime">
    TTS, STT, Medien und Subagent über api.runtime
  </Card>
  <Card title="API für eingehende Kanalereignisse" icon="bolt" href="/de/plugins/sdk-channel-inbound">
    Gemeinsamer Lebenszyklus eingehender Ereignisse: erfassen, auflösen, aufzeichnen, weiterleiten, abschließen
  </Card>
</CardGroup>

<Note>
Einige gebündelte Hilfsschnittstellen bestehen weiterhin zur Wartung und
Kompatibilität gebündelter Plugins. Sie sind nicht das empfohlene Muster für neue Kanal-Plugins;
bevorzugen Sie die generischen Unterpfade für Kanal, Einrichtung, Antworten und Runtime der gemeinsamen SDK-
Oberfläche, sofern Sie diese Familie gebündelter Plugins nicht direkt warten.
</Note>

## Nächste Schritte

- [Provider-Plugins](/de/plugins/sdk-provider-plugins) – wenn Ihr Plugin auch Modelle bereitstellt
- [SDK-Übersicht](/de/plugins/sdk-overview) – vollständige Referenz der Unterpfadimporte
- [SDK-Tests](/de/plugins/sdk-testing) – Testhilfsprogramme und Vertragstests
- [Plugin-Manifest](/de/plugins/manifest) – vollständiges Manifestschema

## Verwandte Themen

- [Einrichtung des Plugin-SDK](/de/plugins/sdk-setup)
- [Plugins erstellen](/de/plugins/building-plugins)
- [Plugins für das Agent-Harness](/de/plugins/sdk-agent-harness)
