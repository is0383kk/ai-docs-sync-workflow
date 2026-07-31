---
read_when:
    - Fallbackgedrag van modellen of de selectie-UX wijzigen
    - Foutopsporing voor ‘model is niet toegestaan’ of een verouderde terugvaloptie voor de standaardprovider
    - Werken aan het samenvoegings- en geheimengedrag van models.json
sidebarTitle: Models CLI
summary: Hoe OpenClaw provider-/modelverwijzingen, configuratiesleutels en de chatopdracht `/model` omzet
title: Modellen-CLI
x-i18n:
    generated_at: "2026-07-27T05:31:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cd13a2aae6575bdfeefb477b7fe8be740b77c66cb76454b07d82481f6612152
    source_path: concepts/models.md
    workflow: 16
---

<CardGroup cols={2}>
  <Card title="Model-failover" href="/nl/concepts/model-failover">
    Rotatie van authenticatieprofielen, afkoelperiodes en de interactie daarvan met fallbacks.
  </Card>
  <Card title="Modelproviders" href="/nl/concepts/model-providers">
    Kort overzicht van providers en voorbeelden.
  </Card>
  <Card title="CLI-referentie voor modellen" href="/nl/cli/models">
    Volledige referentie voor de opdracht en vlaggen van `openclaw models`.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-agents#agent-defaults">
    Modelconfiguratiesleutels, standaardwaarden en voorbeelden.
  </Card>
</CardGroup>

Een modelreferentie (`provider/model`) kiest een provider en model, niet de onderliggende
agentruntime. Als het runtimebeleid niet is ingesteld of `auto` is, kan het providereigen
routebeleid van OpenAI Codex alleen selecteren voor een exacte officiële HTTPS Platform
Responses- of ChatGPT Responses-route zonder door de auteur ingestelde verzoekoverschrijving; alleen het
voorvoegsel `openai/*` selecteert nooit Codex. Completions-adapters, aangepaste
eindpunten en door de auteur ingesteld verzoekgedrag blijven op OpenClaw. Officiële
HTTP-eindpunten met platte tekst worden geweigerd. Zie [Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime).

Voor Copilot-referenties met abonnement (`github-copilot/*`) kan expliciet worden gekozen voor de externe
GitHub Copilot-agentruntime-Plugin, maar dat pad is altijd expliciet (en wordt nooit
geselecteerd door `auto`). Runtimeoverschrijvingen horen bij provider-/modelbeleid, niet bij
de volledige agent of sessie. Runtimeselectie bepaalt de facturering niet:
OpenAI-API-sleutels en ChatGPT-/Codex-abonnementsreferenties blijven gescheiden. Zie
[Agentruntimes](/nl/concepts/agent-runtimes) en
[GitHub Copilot-agentruntime](/nl/plugins/copilot).

## Selectievolgorde

<Steps>
  <Step title="Primair model">
    `agents.defaults.model.primary` (of `agents.defaults.model` als gewone tekenreeks).
  </Step>
  <Step title="Fallbacks">
    `agents.defaults.model.fallbacks`, geprobeerd in de opgegeven volgorde.
  </Step>
  <Step title="Authenticatiefailover">
    Rotatie van authenticatieprofielen vindt binnen een provider plaats voordat OpenClaw naar het volgende fallbackmodel gaat.
  </Step>
</Steps>

Gerelateerde modelconfiguratie-oppervlakken:

- `agents.defaults.models` slaat aliassen en instellingen per model op. Het toevoegen van een vermelding beperkt modeloverschrijvingen niet.
- `agents.defaults.modelPolicy.allow` is de optionele acceptatielijst voor overschrijvingen. Gebruik exacte referenties of afsluitende jokertekens voor voorvoegsels, zoals `provider/*` en `provider/namespace/*`; laat deze weg of stel `[]` in om elk model toe te staan. `agents.entries.*.modelPolicy.allow` per agent vervangt het standaardbeleid voor die agent.
- `agents.defaults.utilityModel` is een optioneel goedkoper model voor korte interne taken, zoals gegenereerde titels van dashboardsessies, ondersteunde thread-/onderwerptitels van kanalen en voortgangsbeschrijvingen. `agents.entries.*.utilityModel` per agent overschrijft dit. Als dit niet is ingesteld, gebruikt OpenClaw de opgegeven standaardwaarde voor kleine modellen van de primaire provider wanneer die bestaat (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`), anders het primaire model van de agent; stel dit in op een lege tekenreeks om hulproutering uit te schakelen. Gegenereerde titels worden bij een fout in een afzonderlijk hulpmodel eenmaal opnieuw geprobeerd met het primaire model. Voor dashboardtitels volgen automatische hulpmodelafleiding en de reguliere fallback de effectieve sessieprovider en het authenticatieprofiel; een expliciet hulpmodel behoudt de geconfigureerde provider/authenticatie. Een leeg hulpmodel slaat alleen de alternatieve route voor kleine modellen over, niet het genereren van dashboardtitels. Hulptaken zijn afzonderlijke modelaanroepen en kunnen begrensde taakinhoud naar de geselecteerde modelprovider sturen.
- `agents.defaults.imageModel` wordt alleen gebruikt wanneer het primaire model geen afbeeldingen kan accepteren.
- `agents.defaults.pdfModel` wordt gebruikt door de tool `pdf`. Als dit niet is ingesteld, valt de tool terug op `imageModel` en daarna op het bepaalde sessie-/standaardmodel.
- `agents.defaults.mediaModels.{image,music,video}` ondersteunt de gedeelde tools voor mediageneratie. Als dit niet is ingesteld, leidt elke tool een standaardprovider met beschikbare authenticatie af: eerst de huidige standaardprovider en daarna de overige geregistreerde providers voor die mogelijkheid, in volgorde van provider-id. Fallback tussen providers is het vaste standaardgedrag.
- `agents.entries.*.model` per agent (plus bindingen) overschrijft `agents.defaults.model` — zie [Routering met meerdere agents](/nl/concepts/multi-agent).

Volledige sleutelreferentie, standaardwaarden en JSON5-voorbeelden: [Configuratiereferentie](/nl/gateway/config-agents#agent-defaults).

## Selectiebron en striktheid van fallbacks

Dezelfde `provider/model` gedraagt zich anders, afhankelijk van waar deze vandaan kwam:

| Bron                                                                  | Gedrag                                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Geconfigureerde standaardwaarde (`agents.defaults.model.primary`, primair model per agent) | Normaal beginpunt; gebruikt `agents.defaults.model.fallbacks`.                                                                                                                                                                                                 |
| Automatische fallback                                                           | Tijdelijke herstelstatus, opgeslagen als `modelOverrideSource: "auto"`. OpenClaw test periodiek het oorspronkelijke primaire model opnieuw, wist de automatische selectie na herstel en kondigt fallback-/herstelovergangen eenmaal per statuswijziging aan.                              |
| Gebruikersselectie voor sessie                                                  | Exact en strikt. `/model`, de modelkiezer, `session_status(model=...)` en `sessions.patch` slaan `modelOverrideSource: "user"` op. Als die provider of dat model onbereikbaar wordt, mislukt de uitvoering zichtbaar in plaats van terug te vallen op een ander geconfigureerd model. |
| Cron `--model` / payload `model`                                        | Primair model per taak. Gebruikt nog steeds geconfigureerde fallbacks, tenzij de taak een eigen payload `fallbacks` opgeeft (`fallbacks: []` dwingt een strikte uitvoering af).                                                                                                                    |

Andere selectieregels:

- Het wijzigen van `agents.defaults.model.primary` herschrijft bestaande sessievastzettingen niet. Als de status `This session is pinned to X; config primary Y will apply to new/unpinned sessions.` meldt, voer je `/model default` uit om de vastzetting te wissen.
- CLI-kiezers voor het standaardmodel en de acceptatielijst respecteren `models.mode: "replace"` door alleen `models.providers.*.models` weer te geven in plaats van de volledige ingebouwde catalogus.
- De modelkiezer in de Control UI vraagt de Gateway om de geconfigureerde modelweergave. Een expliciete `modelPolicy.allow` filtert deze, inclusief vermeldingen met afsluitende jokertekens voor voorvoegsels; anders worden geconfigureerde modellen plus providers met bruikbare authenticatie weergegeven. De volledige ingebouwde catalogus is voorbehouden aan expliciete bladerweergaven (`models.list` met `view: "all"`, of `openclaw models list --all`).
- Gebruikersinterfaces voor providerinventarissen gebruiken `models.list` met `view: "provider-config"` om door de bron opgegeven `models.providers.*.models`-rijen weer te geven zonder acceptatielijsten van kiezers toe te passen.

Volledige werking: [Model-failover](/nl/concepts/model-failover).

## Kort modelbeleid

- Stel je primaire model in op het krachtigste model van de nieuwste generatie dat voor jou beschikbaar is.
- Gebruik fallbacks voor kosten-/latentiegevoelige taken en chats met minder grote gevolgen.
- Vermijd oudere/zwakkere modelniveaus voor agents met ingeschakelde tools of niet-vertrouwde invoer.

## Onboarding

```bash
openclaw onboard
```

Stelt model en authenticatie voor veelgebruikte providers in zonder de configuratie handmatig te bewerken, waaronder OAuth voor een OpenAI Codex-abonnement en Anthropic (API-sleutel of hergebruik van de Claude CLI).

Als er geen primair model is geconfigureerd, selecteert een nieuwe installatie met een OpenAI-API-sleutel
`openai/gpt-5.6`; de kale id voor de directe API wordt omgezet naar het Sol-niveau. Een nieuwe
ChatGPT-/Codex-OAuth-installatie selecteert de exacte catalogusreferentie `openai/gpt-5.6-sol`.
Opnieuw authenticeren behoudt een bestaand expliciet primair model, waaronder
`openai/gpt-5.5`. Als GPT-5.6 niet beschikbaar is voor het account, selecteer je
expliciet `openai/gpt-5.5`; OpenClaw verlaagt het niveau niet stilzwijgend.

## "Model is not allowed" (en waarom antwoorden stoppen)

Als `agents.defaults.modelPolicy.allow` niet leeg is, wordt dit de acceptatielijst voor `/model`, sessieoverschrijvingen en `--model`. Als je een model buiten die acceptatielijst selecteert, wordt de verwerking beëindigd voordat een normaal antwoord wordt gegenereerd. `agents.entries.*.modelPolicy.allow` per agent vervangt het standaardbeleid voor die agent.

```text
Modeloverschrijving "provider/model" is niet toegestaan door agents.defaults.modelPolicy.allow.
Voeg "provider/model", "provider/*" of een specifieker voorvoegsel "provider/namespace/*" toe aan agents.defaults.modelPolicy.allow, of verwijder/leeg de lijst om elk model toe te staan.
```

Los dit op door het model of een providerjokerteken toe te voegen aan de genoemde sleutel `modelPolicy.allow`, die lijst te verwijderen/leeg te maken, of een model uit `/model list` te kiezen. Als de geweigerde opdracht een runtimeoverschrijving bevatte, zoals `/model openai/gpt-5.5 --runtime codex`, corrigeer dan eerst de acceptatielijst en probeer daarna dezelfde opdracht opnieuw.

Voor lokale/GGUF-modellen moet de acceptatielijst de volledige referentie met providervoorvoegsel bevatten, bijvoorbeeld `ollama/gemma4:26b` of `lmstudio/Gemma4-26b-a4-it-gguf` — controleer `openclaw models list --provider <provider>` voor de exacte tekenreeks. Alleen bestandsnamen of weergavenamen zijn niet voldoende zodra de acceptatielijst actief is.

Gebruik vermeldingen met afsluitende jokertekens voor voorvoegsels om providers te beperken zonder elk model op te sommen. Een providerbrede `provider/*` komt overeen met elk model van die provider; een specifieker voorvoegsel, zoals `clawrouter/anthropic/*`, komt alleen overeen met die naamruimte:

```json5
{
  agents: {
    defaults: {
      modelPolicy: {
        allow: ["openai/*", "vllm/*"],
      },
    },
  },
}
```

`/model`, `/models` en modelkiezers tonen dan alleen de ontdekte catalogus voor die providers, en nieuwe modellen kunnen verschijnen zonder de acceptatielijst te bewerken. Combineer exacte `provider/model`-vermeldingen met `provider/*`-vermeldingen om één specifiek model van een andere provider op te nemen.

Voorbeeld van een acceptatielijst met aliassen en instellingen per model:

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
      modelPolicy: {
        allow: ["anthropic/claude-sonnet-4-6", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
}
```

<Accordion title="De acceptatielijst expliciet bewerken">
Stel de volledige lijst rechtstreeks in:

```bash
openclaw config set agents.defaults.modelPolicy.allow '["openai/gpt-5.4","anthropic/*"]' --strict-json
```

`openclaw models set`, providerinstallatie en `openclaw models aliases add` kunnen vermeldingen toevoegen onder `agents.defaults.models`, maar wijzigen nooit `modelPolicy.allow`. Hierdoor blijven modelmetadata en aliassen onafhankelijk van het overschrijvingsbeleid.
</Accordion>

## `/model` in de chat

```text
/model
/model list
/model 3
/model openai/gpt-5.4
/model default
/model status
```

- `/model` en `/model list` tonen een compacte genummerde keuzelijst (modelfamilie + beschikbare providers); `/model <#>` maakt hieruit een keuze. Op Discord worden hiermee vervolgkeuzelijsten voor providers/modellen geopend, gevolgd door een Submit-stap; op Telegram gelden keuzes in de keuzelijst alleen voor de sessie en herschrijven ze nooit de permanente standaardinstelling van de agent in `openclaw.json`. `/models add` is verouderd en retourneert een bericht in plaats van modellen vanuit de chat te registreren.
- `/model` slaat de nieuwe sessiekeuze onmiddellijk op. Als de agent inactief is, gebruikt de volgende uitvoering deze direct; als er al een uitvoering actief is, wordt de omschakeling in de wachtrij geplaatst voor het volgende schone punt voor een nieuwe poging (of een later punt als toolactiviteit of de uitvoer van het antwoord al is begonnen).
- `/model default` wist de sessiekeuze, zodat deze de geconfigureerde primaire keuze weer overneemt.
- Een door de gebruiker geselecteerde `/model`-referentie is strikt voor die sessie: als deze onbereikbaar wordt, mislukt het antwoord zichtbaar in plaats van stilzwijgend terug te vallen via `agents.defaults.model.fallbacks`. Geconfigureerde standaardinstellingen en primaire modellen van Cron-taken blijven terugvalketens gebruiken.
- `/model status` is de gedetailleerde weergave: authenticatiekandidaten per provider en (indien geconfigureerd) het provider-eindpunt `baseUrl` plus de `api`-modus.
- Modelreferenties worden geparseerd door ze bij de eerste `/` te splitsen; typ `provider/model`. Als de model-ID zelf `/` bevat (OpenRouter-stijl), neem dan het providervoorvoegsel op, bijvoorbeeld `/model openrouter/moonshotai/kimi-k2`. Als je de provider weglaat, probeert OpenClaw: (1) een overeenkomst met een alias, (2) een unieke overeenkomst met een geconfigureerde provider voor die exacte model-ID zonder voorvoegsel, (3) de geconfigureerde standaardprovider (verouderde terugvaloptie) — en als die provider het geconfigureerde standaardmodel niet meer aanbiedt, in plaats daarvan de eerste geconfigureerde provider en het eerste geconfigureerde model, om te voorkomen dat een verouderde standaardinstelling voor een verwijderde provider zichtbaar wordt.
- Modelreferenties worden genormaliseerd naar kleine letters; provider-ID's zijn verder exact, dus gebruik de ID die door de plugin wordt vermeld.

Volledig opdrachtgedrag en configuratie: [Slash-opdrachten](/nl/tools/slash-commands).

## CLI

```bash
openclaw models status
openclaw models list
openclaw models set <provider/model>
openclaw models set-image <provider/model>
openclaw models scan
openclaw models aliases list|add|remove
openclaw models fallbacks list|add|remove|clear
openclaw models image-fallbacks list|add|remove|clear
openclaw models auth list|add|login|paste-api-key|paste-token|setup-token|order
```

`openclaw models` zonder subopdracht is een snelkoppeling voor `models status`, die ook het verlopen van OAuth voor authenticatieopslagprofielen toont (standaard wordt binnen 24h gewaarschuwd). Volledige vlaggen, JSON-structuren en subopdrachten voor authenticatieprofielen: [CLI-referentie voor modellen](/nl/cli/models).

<AccordionGroup>
  <Accordion title="Scannen (gratis OpenRouter-modellen)">
    `openclaw models scan` onderzoekt de openbare catalogus van gratis modellen van OpenRouter en kan kandidaten live testen op ondersteuning voor tools en afbeeldingen. De catalogus zelf is openbaar, dus scans met alleen metagegevens (`--no-probe`) hebben geen sleutel nodig; live tests en `--set-default`/`--set-image` vereisen een OpenRouter-API-sleutel (authenticatieprofiel of `OPENROUTER_API_KEY`) en beperken zich zonder sleutel strikt tot uitvoer met alleen metagegevens.

    Resultaten worden gerangschikt op: ondersteuning voor afbeeldingen, vervolgens toollatentie, daarna contextgrootte en ten slotte het aantal parameters. In een TTY vragen geteste resultaten om een interactieve terugvalkeuze; de niet-interactieve modus vereist `--yes` om de standaardinstellingen te accepteren.

  </Accordion>
</AccordionGroup>

## Modellenregister (`models.json`)

Aangepaste providers die onder `models.providers` zijn geconfigureerd, worden naar `models.json` in de agentmap geschreven (standaard `~/.openclaw/agents/<agentId>/agent/models.json`). Catalogi van providerplugins worden afzonderlijk opgeslagen als gegenereerde, door plugins beheerde catalogusfragmenten en automatisch geladen. Dit bestand wordt standaard met de configuratie samengevoegd; stel `models.mode: "replace"` in om alleen je geconfigureerde providers te gebruiken.

<AccordionGroup>
  <Accordion title="Prioriteit in samenvoegmodus">
    Voor overeenkomende provider-ID's:

    - Een niet-lege `baseUrl` die al aanwezig is in de `models.json` van de agent, heeft voorrang.
    - Een niet-lege `apiKey` in `models.json` heeft alleen voorrang wanneer die provider in de huidige context van configuratie/authenticatieprofiel niet door SecretRef wordt beheerd.
    - Door SecretRef beheerde `apiKey`-waarden worden vernieuwd vanuit bronmarkeringen in plaats van opgeloste geheimen permanent op te slaan: de naam van de omgevingsvariabele voor omgevingsreferenties, `secretref-managed` voor bestands-/uitvoerreferenties.
    - Door SecretRef beheerde headerwaarden worden op dezelfde manier vernieuwd, met `secretref-env:ENV_VAR_NAME` voor omgevingsreferenties.
    - Lege of ontbrekende `apiKey`/`baseUrl` in `models.json` vallen terug op configuratie-`models.providers`.
    - Andere providervelden worden vernieuwd vanuit de configuratie en genormaliseerde catalogusgegevens.

  </Accordion>
</AccordionGroup>

Het permanent opslaan van markeringen volgt de bron als gezaghebbende bron: telkens wanneer OpenClaw `models.json` opnieuw genereert — ook via opdrachtgestuurde paden zoals `openclaw agent` — schrijft het markeringen uit de actieve momentopname van de bronconfiguratie (vóór oplossing), niet uit opgeloste geheime runtimewaarden.

## Gerelateerd

- [Agent-runtimes](/nl/concepts/agent-runtimes) — OpenClaw, Codex en andere runtimes voor agentlussen
- [Configuratiereferentie](/nl/gateway/config-agents#agent-defaults) — configuratiesleutels voor modellen
- [Afbeeldingen genereren](/nl/tools/image-generation) — configuratie van afbeeldingsmodellen
- [Model-failover](/nl/concepts/model-failover) — terugvalketens
- [Modelproviders](/nl/concepts/model-providers) — providerroutering en authenticatie
- [CLI-referentie voor modellen](/nl/cli/models) — volledige referentie voor opdrachten en vlaggen
- [Muziek genereren](/nl/tools/music-generation) — configuratie van muziekmodellen
- [Video genereren](/nl/tools/video-generation) — configuratie van videomodellen
