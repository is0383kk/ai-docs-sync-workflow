---
read_when:
    - Ein Plugin für einen Messaging-Kanal erstellen oder migrieren
    - Ändern von Zulassungslisten für Direktnachrichten oder Gruppen, Routing-Sperren, Befehlsauthentifizierung, Ereignisauthentifizierung oder Erwähnungsaktivierung
    - Überprüfung der Schwärzung eingehender Kanalnachrichten oder der SDK-Kompatibilitätsgrenzen
sidebarTitle: Channel Ingress
summary: Experimentelle Channel-Ingress-API für die Autorisierung eingehender Nachrichten
title: Channel-Ingress-API
x-i18n:
    generated_at: "2026-07-26T17:59:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

Channel Ingress ist die experimentelle Zugriffskontrollgrenze für eingehende
Channel-Ereignisse. Plugins sind für Plattformeigenschaften und Seiteneffekte zuständig; der Core ist für
generische Richtlinien zuständig: Zulassungslisten für DMs/Gruppen, DM-Einträge im Pairing-Speicher, Route-Gates,
Command-Gates, Ereignisauthentifizierung, Erwähnungsaktivierung, geschwärzte Diagnosen und
Zulassung.

Verwenden Sie `openclaw/plugin-sdk/channel-ingress-runtime` für Empfangspfade.

## Runtime-Resolver

```ts
import {
  defineStableChannelIngressIdentity,
  resolveChannelMessageIngress,
} from "openclaw/plugin-sdk/channel-ingress-runtime";

const identity = defineStableChannelIngressIdentity({
  key: "platform-user-id",
  normalize: normalizePlatformUserId,
  sensitivity: "pii",
});

const result = await resolveChannelMessageIngress({
  channelId: "my-channel",
  accountId,
  identity,
  subject: { stableId: platformUserId },
  conversation: { kind: isGroup ? "group" : "direct", id: conversationId },
  event: { kind: "message", authMode: "inbound", mayPair: !isGroup },
  policy: {
    dmPolicy: config.dmPolicy,
    groupPolicy: config.groupPolicy,
    groupAllowFromFallbackToAllowFrom: true,
  },
  allowFrom: config.allowFrom,
  groupAllowFrom: config.groupAllowFrom,
  accessGroups: cfg.accessGroups,
  route,
  readStoreAllowFrom,
  command: hasControlCommand ? { allowTextCommands: true, hasControlCommand } : undefined,
});
```

Berechnen Sie effektive Zulassungslisten, Command-Verantwortliche oder Command-Gruppen nicht vorab.
Der Resolver leitet sie aus unverarbeiteten Zulassungslisten, Speicher-Callbacks, Route-
Deskriptoren, Zugriffsgruppen, Richtlinien und der Konversationsart ab.

## Ergebnis

Gebündelte Plugins sollten moderne Projektionen direkt verwenden:

| Feld               | Bedeutung                                                          |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | geordnete Gate-Entscheidung und Zulassung                          |
| `senderAccess`     | nur Absender-/Konversationsautorisierung                            |
| `routeAccess`      | Route- und Route-Absender-Projektion                                |
| `commandAccess`    | Command-Autorisierung; `requested: false`, wenn kein Command-Gate ausgeführt wurde |
| `activationAccess` | Ergebnis der Erwähnung/Aktivierung                                  |

Die Ereignisautorisierung bleibt in der geordneten `ingress.graph` und der
entscheidenden `ingress.reasonCode` verfügbar; es wird keine separate Ereignisprojektion ausgegeben.

Veraltete SDK-Hilfsfunktionen von Drittanbietern können ältere Strukturen intern wiederherstellen. Neue
gebündelte Empfangspfade sollten moderne Ergebnisse nicht zurück in lokale
DTOs übersetzen.

## Zugriffsgruppen

`accessGroup:<name>`-Einträge bleiben geschwärzt. Der Core löst statische
`message.senders`-Gruppen selbst auf und ruft `resolveAccessGroupMembership` nur
für dynamische Gruppen auf, die eine Plattformabfrage erfordern. Fehlende, nicht unterstützte und
fehlgeschlagene Gruppen führen standardmäßig zur Ablehnung.

## Ereignismodi

| `authMode`       | Bedeutung                                        |
| ---------------- | ------------------------------------------------ |
| `inbound`        | normale Absender-Gates für eingehende Ereignisse |
| `command`        | Command-Gates für Callbacks oder kontextgebundene Schaltflächen |
| `origin-subject` | der Akteur muss dem Subjekt der ursprünglichen Nachricht entsprechen |
| `route-only`     | nur Route-Gates für vertrauenswürdige Ereignisse im Route-Kontext |
| `none`           | Plugin-eigene interne Ereignisse umgehen die gemeinsame Authentifizierung |

Verwenden Sie `mayPair: false` für Reaktionen, Schaltflächen, Callbacks und native Commands.

## Routen und Aktivierung

Verwenden Sie Route-Deskriptoren für Richtlinien zu Räumen, Themen, Guilds, Threads oder verschachtelten Routen:

```ts
route: {
  id: "room",
  allowed: roomAllowed,
  enabled: roomEnabled,
  senderPolicy: "replace",
  senderAllowFrom: roomAllowFrom,
  blockReason: "room_sender_not_allowlisted",
}
```

Verwenden Sie `channelIngressRoutes(...)`, wenn ein Plugin mehrere optionale Route-
Deskriptoren hat; es filtert deaktivierte Zweige heraus, während Route-Eigenschaften generisch bleiben
und nach dem jeweiligen `precedence` des Deskriptors geordnet werden.

Das Erwähnungs-Gating ist ein Aktivierungs-Gate. Eine fehlende Erwähnung gibt
`admission: "skip"` zurück, damit der Turn-Kernel keinen Turn verarbeitet, der nur zur Beobachtung dient.
Bei den meisten Channels sollte die Aktivierung nach Absender- und Command-Gates erfolgen. Öffentliche
Chat-Oberflächen, die nicht erwähnten Datenverkehr unterdrücken müssen, bevor die Absender-Zulassungsliste
Störmeldungen erzeugt, können sich für `activation.order: "before-sender"` entscheiden, wenn die Umgehung durch
Text-Commands deaktiviert ist. Channels mit impliziter Aktivierung, etwa Antworten in Bot-
Threads, lösen `channels.defaults.implicitMentions` sowie Channel- und Account-
Überschreibungen mit `resolveChannelImplicitMentions(...)` auf und übergeben das Ergebnis anschließend als
`activation.implicitMentions`. Die projizierte
`activationAccess.shouldBypassMention` meldet, wenn ein Command oder eine implizite
Aktivierung eine explizite Erwähnung umgangen hat.

## Schwärzung

Unverarbeitete Absenderwerte und unverarbeitete Zulassungslisteneinträge dienen nur als Eingabe für den Resolver. Sie
dürfen nicht im aufgelösten Zustand, in Entscheidungen, Diagnosen, Snapshots oder
Kompatibilitätseigenschaften erscheinen. Verwenden Sie undurchsichtige Subjekt-IDs, Eintrags-IDs, Route-IDs und
Diagnose-IDs.

## Verifizierung

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
