---
read_when:
    - CLI-onboarding uitvoeren of configureren
    - Een nieuwe machine instellen
sidebarTitle: 'Onboarding: CLI'
summary: 'CLI-onboarding: verifieer inferentie en laat daarna de resterende configuratie over aan OpenClaw'
title: Onboarding (CLI)
x-i18n:
    generated_at: "2026-07-27T06:13:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 150adfac1424b42d66fa3035339082574cc631ce0dc3db09ad32376ef139bf1c
    source_path: start/wizard.md
    workflow: 16
---

```bash
openclaw onboard
```

CLI-onboarding is de aanbevolen configuratieroute via de terminal op macOS, Linux en
Windows (native of WSL2). Standaard detecteert deze AI-toegang die al op
de machine beschikbaar is, verifieert deze met een echte voltooiing en start OpenClaw om
de werkruimte, Gateway en optionele functies te configureren. `openclaw setup` voert dezelfde flow uit ([Configuratie](/nl/cli/setup) behandelt
de variant `--baseline` die alleen de configuratie uitvoert). Gebruikers van Windows-desktop kunnen ook beginnen
via [Windows Hub](/nl/platforms/windows).

Begeleide onboarding stelt eerst inferentie in. Deze detecteert beschikbare AI-toegang,
vereist een echte voltooiing en start pas daarna [OpenClaw](/nl/cli/openclaw)
om de rest van OpenClaw te configureren. Als je **Skip for now** kiest, wordt onboarding
afgesloten zonder OpenClaw te starten.

De klassieke wizard blijft beschikbaar voor aangepaste providers, configuratie van een externe Gateway,
kanaalkoppeling, daemonbesturing, Skills en imports. Voer deze expliciet uit
met `openclaw onboard --classic`; de begeleide inferentiekiezer delegeert
niet naar deze wizard. Nadat inferentie is geslaagd, kan OpenClaw `open channel wizard for
<channel>` gebruiken om kanaalconfiguratie waarvoor geheimen nodig zijn, over te dragen aan een afgeschermde terminalwizard.
Als je de modelprovider of de authenticatie ervan wilt wijzigen, sluit je OpenClaw af en voer je
`openclaw onboard` uit; OpenClaw opent geen begeleide of klassieke providerflows.

<Info>
Snelste eerste chat: voltooi de begeleide configuratie, voer `openclaw dashboard` uit en chat in
de browser via de Control UI. Documentatie: [Dashboard](/nl/web/dashboard).
</Info>

## Landinstelling

De wizard lokaliseert vaste onboardingtekst. Deze gebruikt de eerste niet-lege waarde uit
`OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` en `LANG`, in die volgorde, en
valt daarna terug op Engels. Ondersteunde landinstellingen: `en`, `zh-CN`, `zh-TW`.

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # Expliciete Engelse overschrijving
```

Productnamen, opdrachten, configuratiesleutels, URL's, provider-ID's, model-ID's en
Plugin-/kanaallabels blijven ongeacht de landinstelling in het Engels.

Om niet-inferentie-instellingen later opnieuw te configureren:

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json` impliceert geen niet-interactieve modus. Gebruik voor scripts `--non-interactive` (zie [CLI-automatisering](/nl/start/wizard-cli-automation)).
</Note>

<Tip>
De klassieke wizard bevat een stap voor zoeken op internet waarin je een provider kunt kiezen: Brave,
DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web
Search, Perplexity, SearXNG of Tavily. Sommige vereisen een API-sleutel; andere werken
zonder sleutel. Configureer dit later met `openclaw configure --section web`. Documentatie:
[Webtools](/nl/tools/web).
</Tip>

## Begeleide standaardflow

Een gewone `openclaw onboard` volgt dit pad:

1. Accepteer de beveiligingsmelding.
2. Detecteer geconfigureerde modellen, omgevingsvariabelen voor API-sleutels, ondersteunde lokale AI-
   CLI's en reeds geïnstalleerde modellen met toolmogelijkheden van bereikbare Ollama- of LM
   Studio-servers op de Gateway-host. Deze alleen-lezen doorgang downloadt nooit een
   model. Installaties van Gemini CLI, Antigravity, Pi en OpenCode worden ook gemeld
   wanneer ze niet als herbruikbare inferentieroute voor begeleide configuratie kunnen dienen.
   Gemini en Antigravity kunnen de toolvrije test niet afdwingen; Pi en OpenCode
   zijn complete agentharnassen in plaats van inferentieroutes voor configuratie.
3. Test de eerste gedetecteerde kandidaat met een echte voltooiing. Toon bij een fout de
   reden en ga verder met de volgende bruikbare kandidaat.
4. Als de detectie niets meer oplevert, kies je OpenAI, Anthropic, xAI (Grok), Google of
   OpenRouter, of kies je **More…** voor de overige providers. De regio's,
   abonnementen en ondersteunde browser-, apparaat-, API-sleutel- of tokenmethoden van elke provider
   verschijnen in een tweede menu en worden met dezelfde echte voltooiing getest.
   Kies **Skip for now** om af te sluiten zonder OpenClaw te starten.
5. Sla alleen de geverifieerde modelroute en eventuele vereiste referentie-/Pluginstatus
   op. Instellingen voor de werkruimte en Gateway blijven ongewijzigd.
6. Start OpenClaw met het geverifieerde model, zodat het de werkruimte,
   Gateway, kanalen, agents, plugins en de resterende optionele configuratie kan instellen.

Als je de opdracht opnieuw uitvoert op een geconfigureerde installatie, wordt eerst het huidige standaardmodel
getest, waardoor de begeleide flow als verificatie- en reparatiedoorgang dient. Een mislukte
controle vervangt het geconfigureerde model nooit automatisch; onboarding stopt en
vraagt hoe verder te gaan. Voer `openclaw channels add` of `openclaw configure` uit voor
latere toevoegingen die geen inferentie betreffen; gebruik `openclaw onboard` voor wijzigingen
aan provider- of authenticatieroutes.

## Klassieke wizard: QuickStart versus Advanced

Voer `openclaw onboard --classic` uit om de volledige wizard te openen. Deze begint met een
keuze tussen **QuickStart** (standaardinstellingen) en **Advanced** (volledige controle). Geef
`--flow quickstart` of `--flow advanced` (alias `manual`) door om de klassieke
flow te selecteren en die vraag over te slaan.

<Tabs>
  <Tab title="QuickStart (standaardinstellingen)">
    - Lokale Gateway, loopbackbinding
    - Standaardwerkruimte (of bestaande werkruimte)
    - Gateway-poort **18789**
    - Gateway-authenticatie **Token** (automatisch gegenereerd, zelfs bij loopback)
    - Toolbeleid: `tools.profile: "coding"` voor nieuwe configuraties (een bestaand expliciet profiel blijft behouden)
    - DM-sessies: onboarding behoudt een expliciete `session.dmScope` en laat deze anders oningesteld, zodat de standaardinstelling `"main"` alle directe berichten uit verschillende kanalen in de doorlopende hoofdsessie van de agent houdt—de standaard voor een persoonlijke agent. Gebruik voor gedeelde inboxen of inboxen met meerdere gebruikers `"per-channel-peer"`; `openclaw security audit` raadt isolatie aan wanneer verkeer van meerdere gebruikers in DM's wordt gedetecteerd. Details: [Referentie voor CLI-configuratie](/nl/start/wizard-cli-reference#outputs-and-internals)
    - Tailscale-blootstelling **Off**
    - DM's van Telegram en WhatsApp gebruiken standaard **allowlist**: Telegram vraagt om een numerieke Telegram-gebruikers-ID, WhatsApp vraagt om een telefoonnummer

  </Tab>
  <Tab title="Advanced (volledige controle)">
    - Toont elke stap: modus, werkruimte, Gateway, kanalen, daemon, Skills

  </Tab>
</Tabs>

Externe modus (`--mode remote`) gebruikt altijd de geavanceerde flow; deze
configureert alleen deze machine om verbinding te maken met een Gateway elders en installeert
of wijzigt nooit iets op de externe host.

## Wat klassieke onboarding configureert

De lokale modus (standaard) doorloopt deze stappen:

1. **Model/authenticatie** - kies een authenticatieflow voor een provider (API-sleutel, OAuth of
   providerspecifieke handmatige authenticatie), inclusief Aangepaste provider
   (compatibel met OpenAI, compatibel met OpenAI Responses, compatibel met Anthropic of
   automatische detectie als Onbekend). Kies een standaardmodel.
   Een nieuwe OpenAI-configuratie met API-sleutel gebruikt standaard `openai/gpt-5.6` (de kale directe-API-
   ID wordt omgezet naar Sol); een nieuwe ChatGPT-/Codex-configuratie gebruikt standaard
   `openai/gpt-5.6-sol`. Als je de configuratie opnieuw uitvoert, blijft een bestaand expliciet model behouden,
   inclusief `openai/gpt-5.5`. Selecteer `openai/gpt-5.5` expliciet als het
   account geen GPT-5.6 beschikbaar stelt.
   Beveiligingsopmerking: als deze agent tools uitvoert of Webhook-/hook-
   inhoud verwerkt, gebruik dan bij voorkeur het sterkste beschikbare model van de nieuwste generatie en houd
   het toolbeleid strikt; zwakkere of oudere niveaus zijn eenvoudiger via promptinjectie te manipuleren.
   Bij niet-interactieve uitvoeringen slaat `--secret-input-mode ref` verwijzingen op die door omgevingsvariabelen
   worden ondersteund, in plaats van API-sleutelwaarden in platte tekst; de omgevingsvariabele waarnaar wordt verwezen moet al
   zijn ingesteld, anders mislukt onboarding direct. De interactieve modus voor geheime verwijzingen kan
   verwijzen naar een omgevingsvariabele of een geconfigureerde providerverwijzing (`file` of
   `exec`), met een snelle voorafgaande controle vóór het opslaan. Na de model-/authenticatieconfiguratie
   biedt de wizard een optionele live voltooiingstest; bij een fout kun je eenmaal terugkeren naar
   de model-/authenticatieconfiguratie of de fout negeren zonder de rest van de
   klassieke wizard te blokkeren. Negeren ontgrendelt OpenClaw niet; configuratie via een gesprek
   vereist nog steeds een geslaagde inferentiecontrole.
2. **Werkruimte** - map voor agentbestanden (standaard `~/.openclaw/workspace`). Maakt initiële bootstrapbestanden aan.
3. **Gateway** - poort, bindadres, authenticatiemodus, Tailscale-blootstelling. Kies in
   de interactieve tokenmodus opslag van tokens in platte tekst (standaard) of kies
   een SecretRef. Niet-interactief SecretRef-pad: `--gateway-token-ref-env <ENV_VAR>`.
4. **Kanalen** - ingebouwde chatkanalen en chatkanalen van officiële plugins, waaronder
   Discord, Feishu, Google Chat, iMessage, Mattermost, Microsoft Teams,
   QQ Bot, Signal, Slack, Telegram, WhatsApp en meer.
5. **Daemon** - installeert een LaunchAgent (macOS), een systemd-gebruikerseenheid
   (Linux/WSL2) of een native Windows Scheduled Task met een terugvaloptie
   per gebruiker via de map Startup.
   Als tokenauthenticatie vereist is en `gateway.auth.token` door SecretRef wordt beheerd,
   valideert de daemoninstallatie deze, maar slaat ze geen herleid token op in
   de omgevingsmetadata van de supervisorservice; een niet-opgeloste SecretRef blokkeert
   de installatie met instructies. Als zowel `gateway.auth.token` als
   `gateway.auth.password` zijn ingesteld terwijl `gateway.auth.mode` niet is ingesteld, wordt de installatie
   geblokkeerd totdat je de modus expliciet instelt.
6. **Statuscontrole** - start de Gateway en verifieert dat deze bereikbaar is.
7. **Skills** - installeert aanbevolen Skills en hun optionele afhankelijkheden.

<Note>
Als je onboarding opnieuw uitvoert, wordt er **niets** gewist tenzij je expliciet
**Reset** kiest (of `--reset` doorgeeft). CLI `--reset` verwijdert standaard configuratie, referenties
en sessies; gebruik `--reset-scope full` om ook de werkruimte te verwijderen. Als de
configuratie ongeldig is of verouderde sleutels bevat, vraagt onboarding je eerst
`openclaw doctor` uit te voeren.
</Note>

`--flow import` voert een gedetecteerde migratieflow (bijvoorbeeld Hermes) uit in de
klassieke wizard in plaats van een nieuwe configuratie; zie [Migreren](/nl/cli/migrate) en de migratiehandleidingen onder
[Installeren](/nl/install/migrating-hermes). `openclaw onboard --modern` is een
compatibiliteitsalias voor [OpenClaw](/nl/cli/openclaw). Deze gebruikt dezelfde
inferentiepoort als `openclaw setup`: geverifieerde inferentie start de
assistent, terwijl een interactieve fout terugkeert naar de begeleide inferentieconfiguratie.

## Nog een agent toevoegen

Gebruik `openclaw agents add <name>` om een afzonderlijke agent te maken met een eigen
werkruimte, sessies en authenticatieprofielen. Uitvoering zonder `--workspace` start
een interactieve flow voor naam, werkruimte, authenticatie, kanalen en bindingen; dit is
niet de volledige `openclaw onboard`-wizard.

Wat deze instelt:

- `agents.entries.*.name`
- `agents.entries.*.workspace`
- `agents.entries.*.agentDir`

Opmerkingen:

- Standaardwerkruimte: `~/.openclaw/workspace-<agentId>` (of onder
  `agents.defaults.workspace` als dat is ingesteld).
- Voeg `bindings` toe om inkomende berichten naar deze agent te routeren (onboarding kan dit voor je doen).
- Niet-interactieve vlaggen: `--model`, `--agent-dir`, `--bind`, `--non-interactive`.

## Volledige referentie

Zie voor gedetailleerd stapsgewijs gedrag en configuratie-uitvoer
[Referentie voor CLI-configuratie](/nl/start/wizard-cli-reference).
Zie voor niet-interactieve voorbeelden [CLI-automatisering](/nl/start/wizard-cli-automation).
Zie voor de volledige vlagreferentie [`openclaw onboard`](/nl/cli/onboard).

## Gerelateerde documentatie

- Referentie voor CLI-opdrachten: [`openclaw onboard`](/nl/cli/onboard)
- Overzicht van onboarding: [Overzicht van onboarding](/nl/start/onboarding-overview)
- Onboarding van de macOS-app: [Onboarding](/nl/start/onboarding)
- Eerste-startritueel voor agents: [Agent bootstrappen](/nl/start/bootstrapping)
