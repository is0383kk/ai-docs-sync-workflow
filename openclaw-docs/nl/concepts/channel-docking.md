---
read_when:
    - Je wilt dat antwoorden voor één actieve sessie worden verplaatst van Telegram naar Discord, Slack, Mattermost of een ander gekoppeld kanaal
    - Je configureert session.identityLinks voor kanaaloverschrijdende directe berichten
    - Een /dock-opdracht meldt dat de afzender niet is gekoppeld of dat er geen actieve sessie bestaat
summary: Verplaats de antwoordroute van één OpenClaw-sessie tussen gekoppelde chatkanalen
title: Kanaaldocking
x-i18n:
    generated_at: "2026-07-27T05:02:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d7af3a59b95b2c73cb74a9529584e51caed055719db2df8aad2ba8e8c9b0593
    source_path: concepts/channel-docking.md
    workflow: 16
---

Kanaaldocking is oproepdoorschakeling voor één OpenClaw-sessie. Dezelfde
gesprekscontext blijft behouden, maar de locatie waar toekomstige antwoorden voor die sessie
worden afgeleverd, verandert. Docking werkt alleen vanuit een privéchat; het werkt niet vanuit een
groepschat.

## Voorbeeld

Alice kan OpenClaw berichten sturen via Telegram en Discord:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456"],
    },
  },
}
```

Als Alice dit vanuit een privéchat op Telegram verstuurt:

```text
/dock_discord
```

behoudt OpenClaw de huidige sessiecontext en wijzigt het de antwoordroute:

| Vóór docking               | Na `/dock_discord`       |
| ---------------------------- | --------------------------- |
| Antwoorden gaan naar Telegram `123` | Antwoorden gaan naar Discord `456` |

De sessie wordt niet opnieuw aangemaakt. De transcriptgeschiedenis blijft gekoppeld aan
dezelfde sessie.

## Waarom je dit gebruikt

Gebruik docking wanneer een taak in de ene chatapp begint, maar de volgende antwoorden
ergens anders moeten aankomen.

Gebruikelijke flow:

1. Start een agenttaak vanuit Telegram.
2. Ga naar Discord, waar je het werk coördineert.
3. Verstuur `/dock_discord` vanuit de privéchat op Telegram.
4. Behoud dezelfde OpenClaw-sessie, maar ontvang toekomstige antwoorden in Discord.

## Vereiste configuratie

Voor docking is `session.identityLinks` vereist. De afzender in het bronkanaal en de doelpeer
moeten deel uitmaken van dezelfde identiteitsgroep:

```json5
{
  session: {
    identityLinks: {
      alice: ["telegram:123", "discord:456", "slack:U123"],
    },
  },
}
```

De waarden zijn peer-id's met een kanaalvoorvoegsel:

| Waarde          | Betekenis                      |
| -------------- | ---------------------------- |
| `telegram:123` | Afzender-id van Telegram `123`     |
| `discord:456`  | Directe peer-id van Discord `456` |
| `slack:U123`   | Gebruikers-id van Slack `U123`         |

De canonieke sleutel (`alice` hierboven) is alleen de naam van de gedeelde identiteitsgroep. Dock-
opdrachten gebruiken de waarden met kanaalvoorvoegsel om te bewijzen dat de afzender in het bronkanaal en
de doelpeer dezelfde persoon zijn.

## Opdrachten

OpenClaw genereert één `/dock-<channel>`-opdracht voor elke geladen kanaalplugin
die systeemeigen opdrachten ondersteunt, waardoor de lijst groeit wanneer plugins worden toegevoegd. Meegeleverde
plugins die dit momenteel ondersteunen:

| Doelkanaal | Opdracht            | Alias              |
| -------------- | ------------------ | ------------------ |
| Discord        | `/dock-discord`    | `/dock_discord`    |
| Mattermost     | `/dock-mattermost` | `/dock_mattermost` |
| Slack          | `/dock-slack`      | `/dock_slack`      |
| Telegram       | `/dock-telegram`   | `/dock_telegram`   |

De vorm met een liggend streepje is ook de naam van de systeemeigen opdracht op oppervlakken zoals Telegram
die slash-opdrachten rechtstreeks beschikbaar stellen.

## Wat verandert

Docking werkt de afleveringsvelden van de actieve sessie bij:

| Sessieveld   | Voorbeeld na `/dock_discord`            |
| --------------- | ---------------------------------------- |
| `lastChannel`   | `discord`                                |
| `lastTo`        | `456`                                    |
| `lastAccountId` | het account van het doelkanaal, of `default` |

Die velden worden opgeslagen in de sessieopslag en gebruikt voor het afleveren van latere
antwoorden voor die sessie.

## Wat niet verandert

Docking doet het volgende niet:

- kanaalaccounts aanmaken
- een nieuwe bot voor Discord, Telegram, Slack of Mattermost verbinden
- een gebruiker toegang verlenen
- toelatingslijsten van kanalen of beleid voor privéberichten omzeilen
- transcriptgeschiedenis naar een andere sessie verplaatsen
- niet-gerelateerde gebruikers een sessie laten delen

Het wijzigt alleen de afleveringsroute voor de huidige sessie.

## Problemen oplossen

**De opdracht meldt dat de afzender niet is gekoppeld.**

Voeg zowel de huidige afzender als de doelpeer toe aan dezelfde
`session.identityLinks`-groep. Als Telegram-afzender `123` bijvoorbeeld moet docken
naar Discord-peer `456`, neem je zowel `telegram:123` als `discord:456` op.

**De opdracht meldt dat docking alleen beschikbaar is vanuit privéchats.**

Verstuur de dockopdracht vanuit een privéchat met OpenClaw, niet vanuit een groepschat.

**De opdracht meldt dat er geen actieve sessie bestaat.**

Dock vanuit een bestaande privéchatsessie. De opdracht heeft een actieve sessie-
vermelding nodig om de nieuwe route te kunnen opslaan.

**Antwoorden gaan nog steeds naar het oude kanaal.**

Controleer of de opdracht met een succesbericht heeft geantwoord en bevestig dat de id van de doelpeer
overeenkomt met de id die door dat kanaal wordt gebruikt. Docking wijzigt alleen de route van de actieve
sessie; een andere sessie kan nog steeds ergens anders naartoe routeren.

**Ik moet terugschakelen.**

Verstuur vanuit een gekoppelde afzender de bijbehorende opdracht voor het oorspronkelijke kanaal, zoals `/dock_telegram` of
`/dock-telegram`.
