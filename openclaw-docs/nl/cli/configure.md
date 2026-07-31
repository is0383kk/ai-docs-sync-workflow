---
read_when:
    - Je wilt referenties, apparaten of standaardinstellingen voor agents interactief aanpassen
summary: CLI-referentie voor `openclaw configure` (interactieve configuratieprompts)
title: Configureren
x-i18n:
    generated_at: "2026-07-27T05:45:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5980d06e75a5df9e5269d0ef78431f730d6f5fd050dca74784ef3426fb0433d8
    source_path: cli/configure.md
    workflow: 16
---

# `openclaw configure`

Interactieve prompts voor gerichte wijzigingen aan een bestaande installatie: referenties, apparaten, standaardinstellingen voor agents, Gateway, kanalen, plugins, Skills en statuscontroles.

Gebruik `openclaw onboard` of `openclaw setup` voor het volledige begeleide traject bij de eerste uitvoering, `openclaw setup --baseline` alleen voor de basisconfiguratie/-werkruimte en `openclaw channels add` wanneer je alleen kanaalaccounts hoeft in te stellen.

<Tip>
`openclaw config` zonder subopdracht opent dezelfde wizard. Gebruik `openclaw config get|set|unset` voor niet-interactieve wijzigingen.
</Tip>

## Opties

`--section <section>`: herhaalbaar sectiefilter. Beschikbare secties:

`workspace`, `model`, `web`, `gateway`, `daemon`, `channels`, `plugins`, `skills`, `health`

```bash
openclaw configure
openclaw configure --section web
openclaw configure --section model --section channels
openclaw configure --section gateway --section daemon
```

Als je `gateway`, `daemon` of `health` selecteert (of de volledige wizard uitvoert zonder `--section`), wordt gevraagd waar de Gateway draait en wordt `gateway.mode` bijgewerkt. Sectiefilters die alle drie overslaan, gaan rechtstreeks naar de gevraagde configuratie zonder prompt voor de gatewaymodus. Als je de externe gatewaymodus kiest, wordt de externe configuratie weggeschreven en wordt de wizard onmiddellijk afgesloten; uitsluitend lokale stappen, zoals het installeren van plugins, worden niet uitgevoerd.

<Note>
`openclaw configure` vereist een interactieve terminal (zowel stdin als stdout moeten TTY's zijn). Zonder interactieve terminal worden de gelijkwaardige niet-interactieve `openclaw config get|set|patch|validate`-opdrachten weergegeven en wordt het programma met een fout afgesloten in plaats van gedeeltelijk te worden uitgevoerd.
</Note>

## Modelsectie

<Note>
**Model** bevat een meervoudige selectie voor de expliciete lijst `agents.defaults.modelPolicy.allow` (wat wordt weergegeven in `/model` en de modelkiezer). Providergebonden configuratiekeuzes voegen de geselecteerde modellen samen met de bestaande lijst, in plaats van niet-gerelateerde providers die al in de configuratie staan te vervangen. Aliassen en parameters per model blijven onder `agents.defaults.models`; deze vermeldingen beperken modeloverschrijvingen op zichzelf niet.

Als je providerauthenticatie opnieuw uitvoert vanuit configure, blijft een bestaande `agents.defaults.model.primary` behouden, zelfs wanneer de authenticatiestap van de provider een configuratiepatch met een eigen aanbevolen standaardmodel retourneert. Door een provider toe te voegen of opnieuw te authenticeren, worden de modellen ervan beschikbaar zonder je huidige primaire model over te nemen. Gebruik `openclaw models auth login --provider <id> --set-default` of `openclaw models set <model>` om het standaardmodel bewust te wijzigen.
</Note>

Wanneer configure vanuit een keuze voor providerauthenticatie wordt gestart, geven de kiezers voor het standaardmodel en modelbeleid automatisch de voorkeur aan die provider. Voor gekoppelde providers zoals Volcengine en BytePlus geldt dezelfde voorkeur ook voor hun coding-planvarianten (`volcengine-plan/*`, `byteplus-plan/*`). Als het filter voor de voorkeursprovider een lege lijst zou opleveren, valt configure terug op de ongefilterde catalogus in plaats van een lege kiezer weer te geven.

## Websectie

`openclaw configure --section web` selecteert een provider voor zoeken op het web en configureert de referenties daarvan. Sommige providers tonen providerspecifieke vervolgvragen:

- **Grok** kan optionele configuratie voor `x_search` aanbieden met hetzelfde xAI OAuth-profiel of dezelfde API-sleutel, en je een `x_search`-model laten kiezen.
- **Kimi** kan vragen naar de Moonshot API-regio (`api.moonshot.ai` tegenover `api.moonshot.cn`) en het standaardmodel van Kimi voor zoeken op het web.

## Overige opmerkingen

- Na lokale configuratiewijzigingen installeert configure geselecteerde downloadbare plugins wanneer dit voor het gekozen configuratietraject vereist is. Bij een externe Gateway-configuratie worden geen lokale pluginpakketten geïnstalleerd.
- Kanaalgerichte services (Slack/Discord/Matrix/Microsoft Teams) vragen tijdens de configuratie om acceptatielijsten voor kanalen/ruimten. Je kunt namen of ID's invoeren; de wizard zet namen waar mogelijk om in ID's.
- Als je de installatiestap voor de daemon uitvoert, vereist tokenauthenticatie een token. Als `gateway.auth.token` door SecretRef wordt beheerd, valideert configure de SecretRef, maar slaat het opgeloste token niet als leesbare tekst op in de omgevingsmetadata van de supervisorservice; als de SecretRef niet kan worden opgelost, blokkeert configure de installatie van de daemon met uitvoerbare herstelrichtlijnen.
- Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd en `gateway.auth.mode` niet is ingesteld, blokkeert configure de installatie van de daemon totdat je de modus expliciet instelt.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Configuratie](/nl/gateway/configuration)
- Config-CLI: [Config](/nl/cli/config)
