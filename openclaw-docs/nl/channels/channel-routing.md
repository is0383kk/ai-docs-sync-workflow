---
read_when:
    - Kanaalroutering of inboxgedrag wijzigen
summary: Routeringsregels per kanaal (WhatsApp, Telegram, Discord, Slack) en gedeelde context
title: Kanaalroutering
x-i18n:
    generated_at: "2026-07-27T04:56:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa03f04a55015bf17e0fe1f3a9bc422875124bb64af5891c898a98bc6917d9e8
    source_path: channels/channel-routing.md
    workflow: 16
---

# Kanalen en routering

OpenClaw routeert antwoorden **terug naar het kanaal waar een bericht vandaan kwam**. Het
model kiest geen kanaal; de routering is deterministisch en wordt bepaald door de
hostconfiguratie. Binnen het standaardbereik voor privéberichten komen privéberichten van elk
kanaal samen in de [hoofdsessie](/concepts/main-session) van de agent.

## Belangrijke termen

- **Kanaal**: een meegeleverde kanaalplugin zoals `discord`, `googlechat`, `imessage`, `irc`, `line`, `signal`, `slack`, `telegram` of `whatsapp`, plus geïnstalleerde pluginkanalen. `webchat` is het interne WebChat-UI-kanaal en is geen configureerbaar uitgaand kanaal.
- **AccountId**: accountinstantie per kanaal (indien ondersteund).
- Optioneel standaardaccount voor het kanaal: `channels.<channel>.defaultAccount` bepaalt
  welk account wordt gebruikt wanneer een uitgaand pad geen `accountId` opgeeft.
  - Stel in configuraties met meerdere accounts een expliciete standaard in (`defaultAccount` of een account met de naam `default`) wanneer twee of meer accounts zijn geconfigureerd. Zonder deze instelling kan reserveroutering de eerste genormaliseerde account-ID kiezen.
- **AgentId**: een geïsoleerde werkruimte + sessieopslag ("brein").
- **SessionKey**: de bucketsleutel die wordt gebruikt om context op te slaan en gelijktijdigheid te beheren.

## Voorvoegsels voor uitgaande doelen

Expliciete uitgaande doelen kunnen een providervoorvoegsel bevatten, zoals `telegram:123` of `tg:123`. Core behandelt dat voorvoegsel alleen als aanwijzing voor kanaalselectie wanneer het geselecteerde kanaal `last` of anderszins onopgelost is, en alleen wanneer de geladen plugin dat voorvoegsel aanbiedt. Als de aanroeper al een expliciet kanaal heeft geselecteerd, moet het providervoorvoegsel overeenkomen met dat kanaal; kanaaloverschrijdende combinaties, zoals bezorging via WhatsApp aan `telegram:123`, mislukken vóór pluginspecifieke normalisatie van het doel.

Voorvoegsels voor doelsoorten en diensten, zoals `channel:<id>`, `user:<id>`, `room:<id>`, `thread:<id>`, `imessage:<handle>` en `sms:<number>`, blijven binnen de grammatica van het geselecteerde kanaal. Ze selecteren niet zelfstandig de provider.

## Vormen van sessiesleutels (voorbeelden)

Privéberichten worden standaard samengevoegd in de **hoofdsessie** van de agent:

- `agent:<agentId>:<mainKey>` (standaard: `agent:main:main`)

`session.dmScope` bepaalt het samenvoegen van privéberichten: `main` (standaard) deelt één hoofdsessie,
terwijl `per-peer`, `per-channel-peer` en `per-account-channel-peer`
privéberichten in afzonderlijke sessies houden. Een routeringsbinding kan het bereik voor de
overeenkomende peers overschrijven via `bindings[].session.dmScope`.

Zelfs wanneer de gespreksgeschiedenis van privéberichten met de hoofdsessie wordt gedeeld, gebruiken het sandbox-
en toolbeleid voor externe privéberichten een afgeleide runtimesleutel per account voor privéchats,
zodat van kanalen afkomstige berichten niet worden behandeld als lokale uitvoeringen van de hoofdsessie.

Groepen en kanalen blijven per kanaal geïsoleerd:

- Groepen: `agent:<agentId>:<channel>:group:<id>`
- Kanalen/ruimten: `agent:<agentId>:<channel>:channel:<id>`

Threads:

- Slack-/Discord-threads voegen `:thread:<threadId>` toe aan de basissleutel.
- Telegram-forumonderwerpen nemen `:topic:<topicId>` op in de groepssleutel.

Voorbeelden:

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## Vastzetten van de hoofdroute voor privéberichten

Wanneer `session.dmScope` gelijk is aan `main`, kunnen privéberichten één hoofdsessie delen.
Om te voorkomen dat de `lastRoute` van de sessie wordt overschreven door privéberichten van niet-eigenaren,
leidt OpenClaw een vastgezette eigenaar af uit `allowFrom` wanneer aan al deze voorwaarden is voldaan:

- `allowFrom` bevat precies één item dat geen jokerteken is.
- Het item kan voor dat kanaal worden genormaliseerd tot een concrete afzender-ID.
- De afzender van het inkomende privébericht komt niet overeen met die vastgezette eigenaar.

Bij zo'n verschil registreert OpenClaw nog steeds de inkomende sessiemetadata, maar
slaat het bijwerken van `lastRoute` in de hoofdsessie over.

## Beveiligde registratie van inkomende berichten

Kanaalplugins kunnen een inkomende sessieregistratie markeren als `createIfMissing: false`
wanneer een beveiligd pad geen nieuwe OpenClaw-sessie mag aanmaken. In die modus
kan OpenClaw metadata en `lastRoute` voor een bestaande sessie bijwerken, maar
maakt het niet alleen omdat een bericht is waargenomen een sessie-item aan dat uitsluitend voor routering dient.

## Routeringsregels (hoe een agent wordt gekozen)

De routering kiest **één agent** voor elk inkomend bericht:

1. **Exacte peer-overeenkomst** (`bindings` met `peer.kind` + `peer.id`).
2. **Overeenkomst met bovenliggende peer** (thread-overerving).
3. **Overeenkomst met peer-jokerteken** (`peer.id: "*"` voor een peersoort).
4. **Overeenkomst met guild + rollen** (Discord) via `guildId` + `roles`.
5. **Guild-overeenkomst** (Discord) via `guildId`.
6. **Teamovereenkomst** (Slack) via `teamId`.
7. **Accountovereenkomst** (`accountId` op het kanaal).
8. **Kanaalovereenkomst** (elk account op dat kanaal, `accountId: "*"`).
9. **Standaardagent** (`agents.entries.*.default`, anders het eerste lijstitem, met `main` als reserve).

Wanneer een binding meerdere overeenkomstvelden bevat (`peer`, `guildId`, `teamId`, `roles`), **moeten alle opgegeven velden overeenkomen** voordat die binding wordt toegepast.

De overeenkomende agent bepaalt welke werkruimte en sessieopslag worden gebruikt.

## Uitzendgroepen (meerdere agents uitvoeren)

Met uitzendgroepen kun je **meerdere agents** uitvoeren voor dezelfde peer **wanneer OpenClaw normaal gesproken zou antwoorden** (bijvoorbeeld: in WhatsApp-groepen, na controle op vermelding/activering).

Configuratie:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

Zie: [Uitzendgroepen](/nl/channels/broadcast-groups).

## Configuratieoverzicht

- `agents.entries`: benoemde agentdefinities (werkruimte, model enz.).
- `bindings`: wijst inkomende kanalen/accounts/peers toe aan agents.

Voorbeeld:

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## Sessieopslag

Runtimesessierijen bevinden zich in de SQLite-database van elke agent onder de statusmap
(standaard `~/.openclaw`):

- `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`

Oudere installaties kunnen verouderde JSONL-bestanden met transcripties en een `sessions.json`-rijopslag
onder `~/.openclaw/agents/<agentId>/sessions/` bevatten. Bij het starten van de Gateway en via
`openclaw doctor --fix` worden actieve verouderde rijen/geschiedenis automatisch in SQLite
geïmporteerd. Gebruik `openclaw doctor --session-sqlite inspect
--session-sqlite-all-agents` en de
validatiereeks van [Doctor](/nl/cli/doctor#session-sqlite-migration) wanneer je
expliciet bewijs van de migratie nodig hebt.
Je kunt nog steeds een verouderd opslagpad selecteren via `session.store` en `{agentId}`-
sjablonen voor migratie- en offline-onderhoudsworkflows.

De sessiedetectie van Gateway en ACP scant ook schijfgebaseerde agentopslag onder de
standaardhoofdmap `agents/` en onder gesjabloneerde hoofdmappen van `session.store`. Gedetecteerde
opslag moet binnen die herleide hoofdmap van de agent blijven en een regulier verouderd
`sessions.json`-bestand gebruiken. Symbolische koppelingen en paden buiten de hoofdmap worden genegeerd.

## WebChat-gedrag

WebChat wordt gekoppeld aan de **geselecteerde agent** en gebruikt standaard de hoofdsessie
van de agent. Hierdoor kun je in WebChat de kanaaloverschrijdende context voor die
agent op één plaats bekijken.

## Antwoordcontext

Inkomende antwoorden bevatten:

- `ReplyToId`, `ReplyToBody` en `ReplyToSender` wanneer beschikbaar.
- Geciteerde context wordt als een `[Replying to ...]`-blok toegevoegd aan `Body`.

Dit is consistent voor alle kanalen.

## Gerelateerd

- [Groepen](/nl/channels/groups)
- [Uitzendgroepen](/nl/channels/broadcast-groups)
- [Koppelen](/nl/channels/pairing)
