---
read_when:
    - Je wilt kanaalaccounts toevoegen of verwijderen (Discord, Google Chat, iMessage, Matrix, Signal, Slack, Telegram, WhatsApp en meer)
    - Je wilt de kanaalstatus controleren of kanaallogs live volgen
    - Je moet een mislukte inkomende kanaalgebeurtenis inspecteren of opnieuw indienen
summary: CLI-referentie voor `openclaw channels` (accounts, status, niet-bezorgde berichten, mogelijkheden, oplossen, logboeken, aanmelden/afmelden)
title: Kanalen
x-i18n:
    generated_at: "2026-07-27T06:07:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

Beheer chatkanaalaccounts en hun runtimestatus op de Gateway.

Gerelateerde documentatie:

- Kanaalhandleidingen: [Kanalen](/nl/channels)
- Gateway-configuratie: [Configuratie](/nl/gateway/configuration)

## Veelgebruikte opdrachten

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` toont alleen chatkanalen: standaard geconfigureerde accounts, met statustags `installed`, `configured` en `enabled` per account (`--json` voor machine-uitvoer). Geef `--all` op om ook gebundelde kanalen weer te geven waarvoor nog geen account is geconfigureerd, en installeerbare cataloguskanalen die nog niet op schijf staan. Providerverificatie en modelgebruik worden elders beheerd: `openclaw models auth list` voor providerverificatieprofielen, `openclaw status` of `openclaw models list` voor gebruik/quota.

## Status / mogelijkheden / oplossen / logboeken

- `channels status`: `--channel <name>`, `--probe`, `--timeout <ms>` (standaard `10000`), `--json`
- `channels capabilities`: `--channel <name>`, `--account <id>` (vereist `--channel`), `--target <dest>` (vereist `--channel`), `--timeout <ms>` (standaard `10000`, begrensd op `30000`), `--json`
- `channels resolve <entries...>`: `--channel <name>`, `--account <id>`, `--kind <auto|user|group>` (standaard `auto`), `--json`
- `channels logs`: `--channel <name|all>` (standaard `all`), `--lines <n>` (standaard `200`), `--json`

`channels status --probe` is het livepad: op een bereikbare Gateway worden per account
`probeAccount`-controles en optionele `auditAccount`-controles uitgevoerd, zodat de uitvoer naast de transportstatus
ook testresultaten kan bevatten, zoals `works`, `probe failed`, `audit ok` of `audit failed`.
Als de Gateway onbereikbaar is, valt `channels status` terug op samenvattingen die uitsluitend op de configuratie zijn gebaseerd,
in plaats van live testuitvoer.

## Inkomende niet-bezorgbare berichten

Inkomende gebeurtenissen die hun beleid voor nieuwe pogingen uitputten, blijven gedurende de bestaande bewaartermijn voor mislukte vermeldingen van de wachtrij in de gedeelde statusdatabase staan. Inspecteer één kanaalaccount met:

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

De tekstweergave toont gebeurtenis-id's, redenen voor mislukkingen, aantallen pogingen en de ouderdom van mislukkingen. JSON-uitvoer bevat voor diagnostiek ook de bewaarde payload, metagegevens, lane en tijdstempels van pogingen.

Nadat je het onderliggende probleem hebt verholpen, plaats je één gebeurtenis opnieuw in de wachtrij met de oorspronkelijke gebeurtenis-id:

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

Voer deze opdrachten uit op de Gateway-host, zodat ze toegang hebben tot dezelfde gedeelde statusdatabase als de kanaalruntime. Bij opnieuw indienen blijven de payload, metagegevens en lane behouden, maar worden de pogingsteller en wachtrijouderdom opnieuw ingesteld. De misluktingsmarkering van die gebeurtenis wordt atomair vervangen. Als je de opdracht herhaalt terwijl de gebeurtenis in behandeling of opgeëist is, wordt deze daarom geweigerd in plaats van een tweede verzending te maken. Het actieve kanaal pikt de gebeurtenis op bij de volgende verwerking van inkomende gegevens. Voltooide gebeurtenissen blijven definitief en kunnen niet opnieuw worden ingediend. Mislukte rijen die zijn gemaakt voordat payloadbewaring werd toegevoegd, kunnen nog steeds in de lijst verschijnen, maar opnieuw indienen wordt geweigerd omdat hun payload niet beschikbaar is.

`openclaw health` rapporteert per kanaalaccount het aantal niet-bezorgbare berichten en de ouderdom van de oudste mislukking. `openclaw doctor` noemt de betrokken accounts en verwijst terug naar de inspectieopdracht.

Gebruik `openclaw sessions`, Gateway-`sessions.list` of de agenttool
`sessions_list` niet als signaal voor de socketstatus van een kanaal. Deze oppervlakken rapporteren
opgeslagen gespreksrijen, niet de runtimestatus van de provider. Nadat een Discord-provider
opnieuw is gestart, kan een verbonden maar rustig account gezond zijn, terwijl er geen Discord-sessierij
verschijnt tot de volgende inkomende of uitgaande gespreksgebeurtenis.

## Accounts toevoegen/verwijderen

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` of `openclaw channels add --channel telegram --help` toont alleen de instellingsvlaggen van Telegram. `openclaw channels add --help` toont alleen de gedeelde opdrachtschil.
</Tip>

`channels remove` werkt alleen met geïnstalleerde/geconfigureerde kanaalplugins. Gebruik eerst `channels add` voor installeerbare cataloguskanalen. Zonder `--delete` wordt gevraagd het account uit te schakelen en blijft de configuratie behouden; `--delete` verwijdert de configuratievermeldingen zonder bevestiging te vragen.
Voor kanaalplugins met runtimeondersteuning vraagt `channels remove` de actieve Gateway ook om het geselecteerde account te stoppen voordat de configuratie wordt bijgewerkt, zodat het uitschakelen of verwijderen van een account de oude listener niet actief laat tot de volgende herstart.

De gedeelde besturingsschil bevat alleen `--channel`, `--account` en de optionele accountweergave `--name`. Elke moderne kanaalplugin beheert zijn eigen referenties, transport en providerspecifieke semantiek. Zodra een kanaal is geselecteerd via een positionele id of `--channel <id>`, bouwt de CLI uitsluitend de opties van dat kanaal op uit de pakketmetagegevens van de gebundelde of geïnstalleerde plugin, zonder kanaalruntimecode te laden.

Algemeen ogende vlaggen zoals `--token`, `--url` of `--use-env` blijven eigendom van het kanaal wanneer ze door een modern contract worden afgehandeld. Wanneer een geselecteerde plugin van derden nog steeds de verouderde gedeelde instellingsadapter gebruikt, registreert de kern alleen voor dat kanaal de uitgebrachte set compatibiliteitsvlaggen, samen met de verouderde `cliAddOptions`. Niet-gerelateerde verouderde velden lekken niet naar andere kanalen en een modern geselecteerd kanaal weigert compatibiliteitsvlaggen die het niet heeft gedeclareerd.

Voorbeelden van kanaaleigen vlaggen zijn:

| Kanaal      | Vlaggen                                                                                              |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`, `--webhook-url`, `--audience-type`, `--audience`                                   |
| iMessage    | `--cli-path`, `--db-path`, `--service`, `--region`                                                   |
| Matrix      | `--homeserver`, `--user-id`, `--access-token`, `--password`, `--device-name`, `--initial-sync-limit` |
| Nostr       | `--private-key`, `--relay-urls`                                                                      |
| Signal      | `--signal-number`, `--signal-transport`, `--cli-path`, `--http-url`, `--http-host`, `--http-port`    |
| Tlon        | `--ship`, `--url`, `--code`, `--group-channels`, `--dm-allowlist`, `--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

Als tijdens een door vlaggen aangestuurde toevoegopdracht een kanaalplugin moet worden geïnstalleerd, gebruikt OpenClaw de standaardinstallatiebron van het kanaal zonder de interactieve installatieprompt voor plugins te openen.

Zowel begeleide als door vlaggen aangestuurde instelling doorloopt de parser, validatie, accountomzetting, configuratieschrijver en hooks na het schrijven van het geselecteerde kanaal. Niet-ondersteunde vlaggen mislukken met de instellingsfout van het kanaal dat ze beheert, in plaats van via een globale verzameling invoer te worden geaccepteerd.

Wanneer je `openclaw channels add` uitvoert zonder directe account-, referentie- of kanaalconfiguratievlaggen, kan de interactieve wizard vragen stellen. Zowel een positionele kanaal-id als `--channel <id>` selecteert dat kanaal vooraf zonder de begeleiding te omzeilen:

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

De wizard kan vragen om:

- account-id's per geselecteerd kanaal
- optionele weergavenamen voor die accounts
- `Route these channel accounts to agents now?`

Als je bevestigt dat de koppeling nu moet plaatsvinden, vraagt de wizard welke agent elk geconfigureerd kanaalaccount moet beheren en schrijft deze routeringskoppelingen op accountniveau.

Je kunt dezelfde routeringsregels later ook beheren met `openclaw agents bindings`, `openclaw agents bind` en `openclaw agents unbind` (zie [agents](/nl/cli/agents)).

Wanneer je een niet-standaardaccount toevoegt aan een kanaal dat nog steeds instellingen voor één account op het hoogste niveau gebruikt, promoveert OpenClaw die waarden op het hoogste niveau naar de accountmap van het kanaal voordat het nieuwe account wordt geschreven. Bij promotie wordt een bestaand benoemd account hergebruikt wanneer het kanaal er precies één heeft, of wanneer `defaultAccount` naar een account verwijst; anders komen de waarden terecht in `channels.<channel>.accounts.default`.

Het routeringsgedrag blijft consistent:

- Bestaande koppelingen die alleen een kanaal bevatten (zonder `accountId`) blijven overeenkomen met het standaardaccount.
- `channels add` maakt of herschrijft in niet-interactieve modus niet automatisch koppelingen.
- Interactieve instelling kan optioneel koppelingen op accountniveau toevoegen.

Als je configuratie al een gemengde status had (benoemde accounts aanwezig terwijl waarden voor één account op het hoogste niveau nog steeds waren ingesteld), voer je `openclaw doctor --fix` uit om accountgebonden waarden te verplaatsen naar het gepromoveerde account dat voor dat kanaal is gekozen.

## Aan- en afmelden (interactief)

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login` ondersteunt `--account <id>` en `--verbose`; `channels logout` ondersteunt `--account <id>`.
- `channels login` en `logout` kunnen het kanaal afleiden wanneer slechts één geconfigureerd kanaal die actie ondersteunt; bij meerdere kanalen geef je `--channel` op.
- `channels logout` geeft de voorkeur aan het live Gateway-pad wanneer dit bereikbaar is, zodat afmelden een actieve listener stopt voordat de verificatiestatus van het kanaal wordt gewist. Als een lokale Gateway niet bereikbaar is, valt de opdracht terug op lokale opschoning van de verificatiestatus; met `gateway.mode: "remote"` mislukt de opdracht in plaats daarvan door de Gateway-fout.
- Na een geslaagde aanmelding vraagt de CLI een bereikbare lokale Gateway om het account te starten; in externe modus slaat deze de verificatie lokaal op en vermeldt dat de externe runtime niet opnieuw is gestart.
- Voer `channels login` uit in een terminal op de Gateway-host. Agent-`exec` blokkeert deze interactieve aanmeldingsflow; kanaaleigen aanmeldingstools voor agents, zoals `whatsapp_login`, moeten waar beschikbaar vanuit de chat worden gebruikt.

## Probleemoplossing

- Voer `openclaw status --deep` uit voor een brede test.
- Gebruik `openclaw doctor` voor begeleide oplossingen.
- `openclaw channels status` valt terug op samenvattingen die uitsluitend op de configuratie zijn gebaseerd wanneer de Gateway onbereikbaar is. Als referenties voor een ondersteund kanaal via SecretRef zijn geconfigureerd maar niet beschikbaar zijn in het huidige opdrachtpad, wordt dat account gerapporteerd als geconfigureerd met opmerkingen over de beperkte werking, in plaats van als niet geconfigureerd.

## Mogelijkheidstest

Haal hints over providermogelijkheden op (intents/scopes waar beschikbaar), plus ondersteuning voor statische functies:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

Opmerkingen:

- `--channel` is optioneel; laat dit weg om elk kanaal weer te geven (inclusief kanalen die door plugins worden geleverd).
- `--account` is alleen geldig met `--channel`.
- `--target` accepteert `channel:<id>` of een onbewerkte numerieke kanaal-id en is alleen van toepassing op Discord. Voor spraakkanalen van Discord markeert de machtigingscontrole ontbrekende `ViewChannel`, `Connect`, `Speak`, `SendMessages` en `ReadMessageHistory`.
- Controles zijn providerspecifiek: Discord-botidentiteit en intents plus optionele kanaalmachtigingen; Slack-bot- en gebruikersscopes; Telegram-botvlaggen en webhook; Signal-daemonversie; Microsoft Teams-apptoken en Graph-rollen/scopes (waar bekend voorzien van annotaties). Kanalen zonder controles rapporteren `Probe: unavailable`.

## Namen omzetten naar ID's

Zet kanaal-/gebruikersnamen om naar ID's via de providerdirectory:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

Opmerkingen:

- Gebruik `--kind user|group|auto` om het doeltype af te dwingen.
- Bij meerdere vermeldingen met dezelfde naam geeft de omzetting de voorkeur aan actieve overeenkomsten.
- `channels resolve` is alleen-lezen. Als een geselecteerd account via SecretRef is geconfigureerd, maar die referentie in het huidige opdrachtpad niet beschikbaar is, retourneert de opdracht beperkte, niet-omgezette resultaten met opmerkingen in plaats van de volledige uitvoering af te breken.
- `channels resolve` installeert geen kanaalplugins. Gebruik `channels add --channel <name>` voordat je namen omzet voor een installeerbaar cataloguskanaal.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Overzicht van kanalen](/nl/channels)
