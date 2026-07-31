---
read_when:
    - Je wilt OpenClaw verbinden met een Raft-werkruimte
    - Je configureert een externe Raft-agent
    - Je debugt de bezorging van Raft-wake-ups
sidebarTitle: Raft
summary: Ondersteuning voor externe Raft-agenten via de wake-bridge van de Raft CLI
title: Raft
x-i18n:
    generated_at: "2026-07-27T05:43:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 454d92d764a4ec3b0ec52467cba254dcad795870e04d1d32d4cf65d8b451a0de
    source_path: channels/raft.md
    workflow: 16
---

Raft verbindt een OpenClaw-agent via de lokale Raft-CLI met een externe Raft-agent.
Raft stuurt geauthenticeerde weksignalen naar de Gateway; de agent gebruikt
vervolgens de Raft-CLI om berichten te controleren en te verzenden. Alleen directe chats (geen groepen).

## Installeren

Raft is een officiële externe Plugin. Installeer deze op de Gateway-host:

```bash
openclaw plugins install @openclaw/raft
openclaw gateway restart
```

Details: [Plugins](/nl/tools/plugin)

## Vereisten

- Een Raft-werkruimte met een externe agent.
- De Raft-CLI moet zijn geïnstalleerd op dezelfde host als de OpenClaw Gateway, in het
  `PATH` van de service.
- Een Raft-CLI-profiel dat al is aangemeld en aan die
  externe agent is gekoppeld.

De Plugin slaat geen Raft-aanmeldgegevens op; de Raft-CLI bewaart die
authenticatie in zijn eigen profiel.

## Configureren

Stel het profiel in de configuratie in:

```json5
{
  channels: {
    raft: {
      enabled: true,
      profile: "openclaw",
    },
  },
}
```

Voor het standaardaccount kun je in plaats daarvan `RAFT_PROFILE` instellen in de
Gateway-omgeving:

```bash
RAFT_PROFILE=openclaw
```

Gebruik een benoemd account wanneer één Gateway verbinding maakt met meer dan één externe Raft-agent:

```json5
{
  channels: {
    raft: {
      accounts: {
        support: {
          profile: "support-agent",
        },
        engineering: {
          profile: "engineering-agent",
        },
      },
    },
  },
}
```

De interactieve installatie registreert hetzelfde profiel:

```bash
openclaw channels add --channel raft
```

## Werking

Wanneer de Gateway wordt gestart, voert de Plugin het volgende uit:

1. Opent op een tijdelijke poort een HTTP-eindpunt voor weksignalen dat alleen via loopback bereikbaar is.
2. Start `raft --profile <profile> agent bridge` met dat eindpunt en een
   token per proces.
3. Accepteert alleen geauthenticeerde weksignalen zonder inhoud en met een replay-identiteit
   van de lokale bridge.
4. Vereist voor elke wekpayload een van `eventId`, `attemptId`, `messageId`, `delivery_id`,
   `wake_id` of `id`.
5. Dedupliceert opnieuw aangeboden weksignalen gedurende 24 uur op basis van de bridge-gebeurtenis-id,
   ook na het opnieuw starten van de Gateway.
6. Retourneert een stabiele runtimesessie voor de huidige bridge en een lege
   batch voor het ophalen van activiteiten voor het Raft-CLI-protocol.
7. Start één geserialiseerde beurt van de OpenClaw-agent per geaccepteerd weksignaal.

De bridge beheert nieuwe afleverpogingen en herverbindingen voor Raft. De OpenClaw-beurt
ontvangt alleen een wekbericht, geen gekopieerde inhoud van een Raft-bericht. De beurt gebruikt de CLI
om berichten in de wachtrij te lezen en een antwoord te verzenden:

```bash
raft --profile openclaw message check
raft --profile openclaw message send
```

<Note>
Raft is geen transport voor pushberichten. OpenClaw stuurt de uiteindelijke tekst van het model niet automatisch terug via de bridge. De agent moet daarom na de verwerking van een weksignaal de Raft-CLI gebruiken.
</Note>

## Verifiëren

Controleer of OpenClaw de CLI kan vinden en een geconfigureerd profiel heeft:

```bash
openclaw channels status --probe
openclaw plugins inspect raft --runtime --json
```

Stuur vervolgens een bericht naar de externe Raft-agent. In het Gateway-logboek moet eerst
het starten van de Raft-bridge en daarna een binnenkomend weksignaal verschijnen. De agent moet
het geconfigureerde Raft-profiel gebruiken om berichten in de wachtrij te controleren.

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Raft-CLI ontbreekt">
    Installeer de Raft-CLI op de Gateway-host en zorg dat `raft` beschikbaar is in het
    `PATH` van de service. Verifieer dit met `raft --help` en start vervolgens de Gateway opnieuw.
  </Accordion>
  <Accordion title="De bridge wordt onmiddellijk afgesloten">
    Controleer of het geconfigureerde profiel is aangemeld en bij de bedoelde
    externe Raft-agent hoort. Voer `raft --profile <profile> agent bridge` rechtstreeks uit
    om de diagnostische informatie van de CLI te bekijken.
  </Accordion>
  <Accordion title="Er komt een weksignaal binnen, maar er wordt geen Raft-antwoord verzonden">
    Dit is te verwachten wanneer de agent de Raft-CLI niet aanroept. De wekbridge
    bevat geen berichtinhoud of automatische definitieve antwoorden. Controleer het
    toolbeleid van de agent en zorg dat deze `raft --profile <profile>
    message check` en `message send` kan uitvoeren.
  </Accordion>
</AccordionGroup>

## Referenties

- [Raft](https://raft.build/)
- [Raft-documentatie](https://docs.raft.build/welcome/)
- [Hermes-integratie met Raft](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/raft)
