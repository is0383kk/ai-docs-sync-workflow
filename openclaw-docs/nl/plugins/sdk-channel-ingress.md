---
read_when:
    - Een Plugin voor een berichtenkanaal bouwen of migreren
    - DM- of groepslijsten met toegestane gebruikers, routepoorten, commando-authenticatie, gebeurtenis-authenticatie of activering via vermeldingen wijzigen
    - Redactie van kanaalingang of SDK-compatibiliteitsgrenzen beoordelen
sidebarTitle: Channel Ingress
summary: Experimentele API voor kanaalingang voor autorisatie van inkomende berichten
title: API voor kanaalingang
x-i18n:
    generated_at: "2026-07-27T05:17:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

Channel-ingress is de experimentele toegangscontrolegrens voor inkomende
channelgebeurtenissen. Plugins beheren platformfeiten en neveneffecten; de kern beheert
generiek beleid: toelatingslijsten voor DM's/groepen, DM-vermeldingen in de pairing-store, routepoorten,
opdrachtpoorten, gebeurtenisautorisatie, vermeldingsactivering, geredigeerde diagnostiek en
toelating.

Gebruik `openclaw/plugin-sdk/channel-ingress-runtime` voor ontvangstpaden.

## Runtime-resolver

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

Bereken effectieve toelatingslijsten, opdrachteigenaren of opdrachtgroepen niet vooraf.
De resolver leidt deze af uit onbewerkte toelatingslijsten, store-callbacks, route-
descriptors, toegangsgroepen, beleid en het gesprekstype.

## Resultaat

Gebundelde plugins moeten moderne projecties rechtstreeks gebruiken:

| Veld               | Betekenis                                                          |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | geordende poortbeslissing en toelating                             |
| `senderAccess`     | alleen autorisatie van afzender/gesprek                            |
| `routeAccess`      | projectie van route en routeafzender                               |
| `commandAccess`    | opdrachtautorisatie; `requested: false` wanneer geen opdrachtpoort is uitgevoerd |
| `activationAccess` | resultaat van vermelding/activering                                |

Gebeurtenisautorisatie blijft beschikbaar in de geordende `ingress.graph` en de
doorslaggevende `ingress.reasonCode`; er wordt geen afzonderlijke gebeurtenisprojectie uitgegeven.

Verouderde SDK-helpers van derden mogen intern oudere structuren reconstrueren. Nieuwe
gebundelde ontvangstpaden mogen moderne resultaten niet terugvertalen naar lokale
DTO's.

## Toegangsgroepen

`accessGroup:<name>`-vermeldingen blijven geredigeerd. De kern verwerkt statische
`message.senders`-groepen zelf en roept `resolveAccessGroupMembership` alleen aan
voor dynamische groepen waarvoor een platformopzoeking nodig is. Ontbrekende, niet-ondersteunde en
mislukte groepen worden standaard geweigerd.

## Gebeurtenismodi

| `authMode`       | Betekenis                                        |
| ---------------- | ------------------------------------------------ |
| `inbound`        | normale poorten voor inkomende afzenders         |
| `command`        | opdrachtpoorten voor callbacks of afgebakende knoppen |
| `origin-subject` | actor moet overeenkomen met het onderwerp van het oorspronkelijke bericht |
| `route-only`     | alleen routepoorten voor vertrouwde, routegebonden gebeurtenissen |
| `none`           | interne gebeurtenissen van plugins omzeilen gedeelde autorisatie |

Gebruik `mayPair: false` voor reacties, knoppen, callbacks en systeemeigen opdrachten.

## Routes en activering

Gebruik routedescriptors voor beleid voor ruimtes, onderwerpen, guilds, threads of geneste routes:

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

Gebruik `channelIngressRoutes(...)` wanneer een plugin meerdere optionele route-
descriptors heeft; deze filtert uitgeschakelde vertakkingen terwijl routefeiten generiek
blijven en volgens de `precedence` van elke descriptor worden geordend.

Vermeldingscontrole is een activeringspoort. Een gemiste vermelding retourneert
`admission: "skip"`, zodat de turn-kernel geen alleen-observerende beurt verwerkt.
De meeste channels moeten activering na de afzender- en opdrachtpoorten laten plaatsvinden. Openbare
chatoppervlakken die niet-vermeld verkeer moeten dempen voordat ruis van de afzendertoelatingslijst
ontstaat, kunnen `activation.order: "before-sender"` gebruiken wanneer de omzeiling
voor tekstopdrachten is uitgeschakeld. Channels met impliciete activering, zoals antwoorden in bot-
threads, verwerken `channels.defaults.implicitMentions` plus channel- en account-
overschrijvingen met `resolveChannelImplicitMentions(...)` en geven het resultaat vervolgens door als
`activation.implicitMentions`. De geprojecteerde
`activationAccess.shouldBypassMention` meldt wanneer een opdracht of impliciete
activering een expliciete vermelding heeft omzeild.

## Redactie

Onbewerkte afzenderwaarden en onbewerkte vermeldingen in toelatingslijsten dienen alleen als invoer voor de resolver. Ze
mogen niet voorkomen in verwerkte status, beslissingen, diagnostiek, snapshots of
compatibiliteitsfeiten. Gebruik ondoorzichtige onderwerp-id's, vermeldings-id's, route-id's en
diagnostische id's.

## Verificatie

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
