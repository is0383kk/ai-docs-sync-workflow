---
read_when:
    - Je wilt Anthropic-modellen gebruiken in OpenClaw
    - Je wilt Claude CLI- of Claude Desktop-sessies op gekoppelde computers bekijken
summary: Gebruik Anthropic Claude via API-sleutels of de Claude CLI in OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-07-27T05:44:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic bouwt de **Claude**-modelfamilie. OpenClaw ondersteunt twee authenticatieroutes:

- **API-sleutel** - rechtstreekse toegang tot de Anthropic-API met facturering op basis van gebruik (`anthropic/*`-modellen)
- **Claude CLI** - hergebruik een bestaande Claude Code-aanmelding op dezelfde host

## Gebruiks- en kostentracering

OpenClaw detecteert de beschikbare Anthropic-referentie en selecteert het bijbehorende gebruiksoverzicht:

- Referenties voor een Claude-abonnement of -configuratie tonen quotumperiodes en een optioneel budget voor extra gebruik.
- `ANTHROPIC_ADMIN_KEY` of `ANTHROPIC_ADMIN_API_KEY` toont 30 dagen aan door de provider gerapporteerde organisatiekosten en gebruik van de Messages API in **Usage** van de Control UI, inclusief dagelijkse uitgaven, totalen voor tokens/cache, populairste modellen en kostencategorieën.
- Een `sk-ant-admin...`-referentie die in het Anthropic-providerprofiel is opgeslagen, wordt automatisch als Admin API-sleutel gedetecteerd.

De kostengeschiedenis van de Admin API komt uit Anthropics [Usage and Cost API](https://platform.claude.com/docs/en/manage-claude/usage-cost-api). Dit betreft de werkelijke facturering door de provider, los van de door OpenClaw op basis van sessies geschatte kosten.

<Warning>
De Claude CLI-backend van OpenClaw voert de geïnstalleerde Claude Code CLI uit in
niet-interactieve afdrukmodus (`claude -p`). De huidige Claude Code-documentatie van Anthropic
beschrijft die modus als gebruik via de Agent SDK/programmatisch gebruik. Anthropics
ondersteuningsupdate van 15 juni 2026 heeft de aangekondigde afzonderlijke wijziging
in de facturering van de Agent SDK opgeschort: gebruik van de Claude Agent SDK,
`claude -p` en apps van derden valt nog steeds onder de gebruikslimieten van
een aangemeld abonnement, en het eerder aangekondigde maandelijkse tegoed voor de
Agent SDK is niet beschikbaar zolang Anthropic dat plan herziet.

Interactief gebruik van Claude Code valt nog steeds onder de limieten van het
aangemelde Claude-abonnement. Authenticatie met een API-sleutel wordt rechtstreeks
op basis van gebruik gefactureerd en is niet afhankelijk van dat abonnement.
Gebruik een Anthropic API-sleutel voor langlevende Gateway-hosts, gedeelde
automatisering en voorspelbare productie-uitgaven.

De huidige ondersteuningsartikelen van Anthropic kunnen dit gedrag wijzigen
zonder een OpenClaw-release:

- [Naslaginformatie voor de Claude Code CLI](https://code.claude.com/docs/en/cli-usage)
- [De Claude Agent SDK gebruiken met je Claude-abonnement](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [Claude Code gebruiken met je Pro- of Max-abonnement](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [Claude Code gebruiken met je Team- of Enterprise-abonnement](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [Kosten van Claude Code beheren](https://code.claude.com/docs/en/costs)

</Warning>

## Aan de slag

<Tabs>
  <Tab title="API-sleutel">
    **Het meest geschikt voor:** standaard API-toegang en facturering op basis van gebruik.

    <Steps>
      <Step title="Verkrijg je API-sleutel">
        Maak een API-sleutel aan in de [Anthropic Console](https://console.anthropic.com/).
      </Step>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard
        # kies: Anthropic API key
        ```

        Of geef de sleutel rechtstreeks door:

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="Controleer of het model beschikbaar is">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### Configuratievoorbeeld

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **Het meest geschikt voor:** een bestaande Claude CLI-aanmelding hergebruiken zonder afzonderlijke API-sleutel.

    <Steps>
      <Step title="Zorg dat Claude CLI is geïnstalleerd en aangemeld">
        Controleer dit met:

        ```bash
        claude --version
        ```
      </Step>
      <Step title="Voer de onboarding uit">
        ```bash
        openclaw onboard
        # kies: Claude CLI
        ```

        OpenClaw detecteert en hergebruikt de bestaande Claude CLI-referenties.
      </Step>
      <Step title="Controleer of het model beschikbaar is">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    Configuratie- en runtimegegevens voor de Claude CLI-backend staan in [CLI-backends](/nl/gateway/cli-backends).
    </Note>

    <Warning>
    Voor hergebruik van Claude CLI moet het OpenClaw-proces op dezelfde host
    draaien als de Claude CLI-aanmelding. Docker-installaties kunnen een
    containerhome behouden en Claude Code daarin aanmelden; zie
    [Claude CLI-backend in Docker](/nl/install/docker#claude-cli-backend-in-docker).
    Andere containerinstallaties, zoals [Podman](/nl/install/podman), koppelen
    `~/.claude` van de host niet aan de configuratie of runtime; gebruik
    daar een Anthropic API-sleutel of kies een provider met door OpenClaw beheerde
    OAuth, zoals [OpenAI Codex](/nl/providers/openai).
    </Warning>

    ### Een configuratietoken verkrijgen

    Voer `claude setup-token` uit op een willekeurige machine waarop Claude Code
    is geïnstalleerd. Hiermee wordt een langlevend token afgedrukt dat begint
    met `sk-ant-oat01-`.

    Plak het token tijdens de onboarding in de macOS-app door
    **Anthropic setup-token** te kiezen onder **Connect with an API key or token**, of gebruik:

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### Configuratievoorbeeld

    Geef de voorkeur aan de canonieke Anthropic-modelreferentie plus een CLI-runtime-overschrijving:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    Verouderde `claude-cli/claude-opus-4-7`-modelreferenties blijven werken voor
    compatibiliteit, maar nieuwe configuraties moeten de provider-/modelselectie
    als `anthropic/*` behouden en de uitvoeringsbackend in het runtimebeleid
    voor de provider/het model plaatsen.

    ### Facturering en `claude -p`

    OpenClaw gebruikt het niet-interactieve `claude -p`-pad van Claude Code
    voor Claude CLI-uitvoeringen. Anthropic behandelt dat pad momenteel als gebruik
    via de Agent SDK/programmatisch gebruik:

    - Anthropics ondersteuningsupdate van 15 juni 2026 heeft het eerder aangekondigde
      afzonderlijke tegoedplan voor de Agent SDK opgeschort.
    - Gebruik van de Claude Agent SDK binnen een abonnement, `claude -p`
      en apps van derden valt nog steeds onder de gebruikslimieten van het aangemelde abonnement.
    - Het eerder aangekondigde maandelijkse tegoed voor de Agent SDK is niet
      beschikbaar zolang Anthropic dat plan herziet.
    - Aanmeldingen via de Console/API-sleutel gebruiken API-facturering op basis
      van gebruik en ontvangen het Agent SDK-tegoed van het abonnement niet.

    Zie Anthropics [artikel over het Agent SDK-abonnement](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
    voor de kennisgeving over de opschorting, en de artikelen over Claude Code-abonnementen
    voor het abonnementsgedrag van
    [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
    en
    [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan).

    Anthropic kan de facturering en het gedrag van snelheidslimieten van Claude Code
    wijzigen zonder een OpenClaw-release. Controleer `claude auth status`,
    `/status` en de gekoppelde documentatie van Anthropic wanneer
    voorspelbare facturering belangrijk is.

    <Tip>
    Gebruik voor gedeelde productieautomatisering een Anthropic API-sleutel in
    plaats van Claude CLI. OpenClaw ondersteunt ook abonnementsachtige opties van
    [OpenAI Codex](/nl/providers/openai), [Qwen Cloud](/nl/providers/qwen),
    [MiniMax](/nl/providers/minimax) en [Z.AI / GLM](/nl/providers/zai).
    </Tip>

  </Tab>
</Tabs>

## Claude-sessies op verschillende computers

De gebundelde Anthropic-plugin voegt een groep **Claude Code** toe aan de normale
sessiezijbalk. Rijen worden geopend in het normale chatvenster. De plugin vindt
niet-gearchiveerde Claude Code-sessies op de Gateway en op verbonden nodehosts:

- Claude CLI-sessies zijn afkomstig uit geldige projectindexrecords. Voor niet-geïndexeerde
  transcripten herkent een begrensde terugval op metadata gelijktijdige interactieve
  sessies zonder zijketen (`cli`) en headless CLI-sessies van de Agent SDK
  (`sdk-cli`) onder `~/.claude/projects/`.
- Claude Desktop-sessies gebruiken de Desktop-titel, activiteitstijd en
  archiefstatus wanneer de metadata naar dezelfde Claude Code-sessie-ID verwijst.
- Een sessie die alleen via de CLI bestaat, heeft geen archiefvlag en blijft
  daarom zichtbaar zolang het transcript aanwezig is.

Voor detectie is geen aanvullende OpenClaw-configuratie vereist. De Anthropic-plugin
is gebundeld en standaard ingeschakeld; een native macOS-node maakt de alleen-lezen
opdrachten voor Claude-sessies bekend wanneer de lokale map `~/.claude/projects/`
bestaat. Keur de upgrade van de nodekoppeling goed wanneer die opdrachten voor het
eerst verschijnen.

De zijbalk groepeert rijen op Gateway- of gekoppelde-nodehost en toont de nieuwste
begrensde pagina van elke host zodra die computer antwoordt. De gegevens worden
opnieuw afgestemd na wijzigingen in de hostconnectiviteit, wanneer de pagina
opnieuw focus krijgt en maximaal elke 30 seconden zolang deze zichtbaar is, zodat
Claude-sessies die buiten OpenClaw zijn gemaakt zonder herladen verschijnen.
Bij een gewijzigde catalogus volgt sneller een extra controle. Gebruik **Meer
sessies laden** onder een catalogusgroep om voor elke host met meer geschiedenis
de volgende pagina toe te voegen; toegevoegde rijen blijven zichtbaar en worden
bij vernieuwingen opnieuw tot dezelfde diepte opgehaald. Catalogusclients gebruiken
`sessions.catalog.list`; bij het openen van een rij wordt `sessions.catalog.read` gebruikt.

Bij terminalovername wordt `claude` opgezocht via de PATH van de
aanmeldingsshell van de gebruiker van de eigenaarhost, vóór de PATH van de
service/daemon. Hierdoor blijven vanuit de app gestarte sessies afgestemd op de
Claude CLI die de beheerder in een normale terminal gebruikt.

Wanneer je een rij selecteert, wordt eerst de nieuwste transcriptpagina gelezen.
**Oudere transcriptitems laden** volgt een ondoorzichtige bytecursor en leest nog
een begrensd gedeelte uit het JSONL-bestand in plaats van de volledige geschiedenis
te laden. Normale inhoud van gebruikers, assistenten, redeneringen, toolaanroepen
en toolresultaten blijft behouden. Een afzonderlijk item dat groter is dan de
veiligheidslimiet van de node/Gateway wordt duidelijk als afgekapt gemarkeerd.

Wanneer je in de normale editor van een Gateway-lokale `claude-cli`-rij typt,
wordt `sessions.catalog.continue` aangeroepen. OpenClaw zoekt het lokale catalogusrecord
opnieuw op, maakt een modelgebonden native sessie of hergebruikt deze, importeert
maximaal 200 zichtbare items of 512 KiB en initialiseert de Claude CLI-koppeling.
De eerste beurt wordt hervat met `--fork-session`; Claude wijst de fork een nieuwe
sessie-ID toe, zodat latere beurten de fork gebruiken en de bronsessie ongewijzigd
blijft.

Een headless nodehost kan zijn Claude CLI-rijen ook voortzetbaar maken door de
onderstaande node-lokale instelling in te schakelen en de nodehost opnieuw te starten:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

De node maakt `agent.cli.claude.run.v1` alleen bekend wanneer de instelling is
ingeschakeld en het lokale uitvoerbare bestand `claude` kan worden gevonden.
OpenClaw zoekt het catalogusrecord opnieuw op die node op, importeert dezelfde
begrensde geschiedenis en koppelt de overgenomen sessie aan de node en de door de
catalogus gerapporteerde werkmap. Elke beurt voert het echte `claude -p`-proces
van de node uit met de Claude-bestanden en aanmelding van die node. Het
goedkeuringsbeleid voor uitvoeropdrachten van de node blijft van toepassing; de
Gateway kan de opt-in niet afdwingen.

Nodevoortzetting v1 is uitsluitend eenmalig. Gateway-loopback-MCP-configuratie en
argumenten voor de Gateway Skills-plugin worden weggelaten, er wordt niet opnieuw
geïnitialiseerd vanuit een Gateway-transcript en bijlagen en afbeeldingen worden
geweigerd. Claude Desktop-rijen blijven alleen-lezen. Native macOS-appnodes blijven
ook alleen-lezen totdat de app de uitvoeropdracht bekendmaakt.

<Note>
Claude-sessies van gekoppelde nodes blijven alleen-lezen, tenzij de headless node
`agent.cli.claude.run.v1` expliciet bekendmaakt. OpenClaw wijzigt nooit metadata van
Claude Desktop en archiveert geen Claude-sessies. De pagina vereist een
beheerdersverbinding met schrijfrechten omdat deze geauthenticeerde
`node.invoke` gebruikt; weergeven en lezen blijven alleen-lezen, zelfs op
een node waarop voortzetting is ingeschakeld.
</Note>

Zie [Nodes: Claude-sessies en transcripties](/nl/nodes#claude-sessions-and-transcripts)
voor de Node-opdracht en beveiligingsgrens.

## Standaardinstellingen voor denken (Claude Opus 5, Sonnet 5, Mythos 5, Fable 5, 4.8 en 4.6)

`anthropic/claude-opus-5` gebruikt standaard adaptief denken met inspanningsniveau `high`.
Gebruik `/think off` om denken uit te schakelen, of `/think xhigh|max` voor de hogere
systeemeigen inspanningsniveaus van het model. OpenClaw laat handmatige denkbudgetten, aangepaste
samplingparameters, vooraf ingevulde assistenttekst en Priority Tier voor Opus 5 weg, omdat
Anthropic deze aanvraagfuncties niet ondersteunt voor dit model. De catalogus
vermeldt het contextvenster van 1.000.000 tokens, de uitvoerlimiet van 128.000 tokens, afbeeldingsinvoer
en de `$5/$25`-prijzen voor invoer/uitvoer.

`anthropic/claude-sonnet-5` gebruikt dezelfde standaardinstellingen voor adaptief denken en
aanvraagbeperkingen. De catalogus gebruikt tot en met 31 augustus 2026 de introductieprijzen van Anthropic van `$2/$10` voor invoer/uitvoer;
de standaardprijzen van `$3/$15` gaan in op 1 september 2026.

`anthropic/claude-fable-5` gebruikt altijd adaptief denken en hanteert standaard inspanningsniveau `high`.
Anthropic staat niet toe dat denken voor dit model wordt uitgeschakeld, dus
`/think off` en `/think minimal` worden in plaats daarvan toegewezen aan inspanningsniveau `low`. OpenClaw laat ook
aangepaste temperatuurwaarden weg voor Fable 5-aanvragen, omdat Anthropic
een temperatuur-override afwijst voor elke aanvraag waarbij denken is ingeschakeld.

`anthropic/claude-mythos-5` is een model met beperkte toegang en hetzelfde contract voor
altijd ingeschakeld adaptief denken. OpenClaw gebruikt standaard `high`, wijst `/think off` en
`/think minimal` toe aan `low` en laat door de aanroeper gekozen samplingparameters weg.
De catalogus vermeldt het contextvenster van 1.000.000 tokens, de uitvoerlimiet van 128.000 tokens,
afbeeldingsinvoer en de `$10/$50`-prijzen voor invoer/uitvoer.

Claude Opus 4.8 houdt denken standaard uitgeschakeld in OpenClaw. Wanneer je
adaptief denken expliciet inschakelt met `/think high|xhigh|max`, verzendt OpenClaw
de inspanningswaarden van Anthropic voor Opus 4.8; Claude 4.6-modellen (Opus 4.6 en Sonnet 4.6)
gebruiken standaard `adaptive`.

Overschrijf dit per bericht met `/think:<level>` of in de modelparameters:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
Gerelateerde documentatie van Anthropic:
- [Adaptief denken](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Uitgebreid denken](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## Terugvaloptie bij veiligheidsweigering (Claude Fable 5)

<Warning>
Claude Fable 5 gebruiken betekent ook Claude Opus 4.8 gebruiken. Fable 5 wordt geleverd met
veiligheidsclassificatoren die een aanvraag kunnen weigeren, en de door Anthropic goedgekeurde
herstelmethode is om `claude-opus-4-8` die beurt te laten afhandelen. OpenClaw schakelt dit
automatisch in voor rechtstreekse aanvragen met een API-sleutel, waardoor sommige Fable-beurten
worden beantwoord en gefactureerd als Claude Opus 4.8. Als jouw beleid of budget
geen door Opus afgehandelde beurten toestaat, selecteer dan niet `anthropic/claude-fable-5`.
</Warning>

### Waarom dit bestaat

Fable 5-classificatoren retourneren `stop_reason: "refusal"` bij aanvragen in beperkte
domeinen en leveren ook fout-positieve resultaten op bij aangrenzend, onschadelijk werk (beveiligingstools,
levenswetenschappen of zelfs de vraag aan het model om zijn onbewerkte
redenering te reproduceren). Zonder terugvaloptie eindigt de beurt met een fout, hoewel
een ander Claude-model de aanvraag zonder problemen zou afhandelen. Het weigeringsbericht van Anthropic zelf
vertelt API-integrators een terugvalmodel te configureren.

### Hoe het werkt

1. Voor elke rechtstreekse aanvraag met een API-sleutel aan `anthropic/claude-fable-5` verzendt OpenClaw
   de serverzijdige aanmelding voor terugval van Anthropic: de
   bètaheader `server-side-fallback-2026-06-01` plus
   `fallbacks: [{"model": "claude-opus-4-8"}]`. Claude Opus 4.8 is het enige
   terugvaldoel dat Anthropic toestaat voor Fable 5.
2. Alleen een weigering door een veiligheidsclassificator activeert de terugvaloptie. Snelheidslimieten,
   overbelasting en serverfouten gedragen zich precies zoals voorheen en verlopen via
   de normale [modelovername](/nl/concepts/model-failover) van OpenClaw.
3. De reddingsactie vindt binnen dezelfde aanroep plaats. Een weigering vóór enige uitvoer is
   afgezien van de latentie onzichtbaar; het volledige antwoord is afkomstig van Opus 4.8. Bij een
   weigering tijdens het streamen blijft de gedeeltelijke tekst behouden als voorvoegsel vanwaar het terugvalmodel
   verdergaat, terwijl de redenering en toolaanroepen van het weigerende model
   volgens de replayregels van Anthropic worden verwijderd (ze mogen niet worden teruggestuurd of
   uitgevoerd).
4. Als Claude Opus 4.8 eveneens weigert, wordt de weigering voor de beurt als
   fout weergegeven, precies zoals vóór deze functie.

De terugval vindt plaats op het niveau van de Anthropic-API, dus `claude-opus-4-8` hoeft niet
in je geconfigureerde modellenlijst of terugvalketen te staan: een API-sleutel die Fable ondersteunt,
kan altijd Opus afhandelen.

### Observeerbaarheid en facturering

- Een door de terugvaloptie afgehandelde beurt registreert een diagnose `provider_fallback` in het
  assistentbericht waarin `fromModel` en `toModel` worden genoemd, en de
  `responseModel` van het bericht rapporteert `claude-opus-4-8`.
- Anthropic factureert per poging: een weigering vóór uitvoer is gratis en de reddingsactie
  wordt gefactureerd tegen Claude Opus 4.8-tarieven (momenteel de helft van de Fable 5-tarieven). De
  kostenraming per beurt van OpenClaw berekent door terugval afgehandelde beurten tegen Opus-tarieven om hiermee overeen te komen.
- Bij een weigering tijdens het streamen factureert Anthropic bovendien het reeds gestreamde gedeeltelijke
  Fable-resultaat; dat gedeelte wordt gerapporteerd in het verbruik per poging van de API,
  maar niet opgenomen in de raming per beurt van OpenClaw.

### Toepassingsgebied

Van toepassing op `anthropic/claude-fable-5` met authenticatie via een API-sleutel bij
`api.anthropic.com`. OAuth (hergebruik van een Claude CLI-abonnement), proxybasis-URL's,
Bedrock-, Vertex- en Foundry-aanvragen blijven ongewijzigd en geven
weigeringen daar nog steeds als fouten weer.

Live geverifieerd: een onschuldige prompt waarin Fable 5 wordt gevraagd zijn onbewerkte redeneerketen
te reproduceren, wordt zonder terugvalopties geweigerd met `category: "reasoning_extraction"`,
terwijl dezelfde prompt via OpenClaw een normaal, door Opus afgehandeld
antwoord retourneert met de diagnose `provider_fallback` eraan gekoppeld.

Zie de [handleiding voor weigeringen en terugvalopties van
Anthropic](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
voor het onderliggende gedrag.

## Promptcaching

OpenClaw ondersteunt de promptcachingfunctie van Anthropic voor authenticatie via een API-sleutel.

| Waarde               | Cacheduur | Beschrijving                            |
| ------------------- | -------------- | -------------------------------------- |
| `"short"` (standaard) | 5 minuten      | Automatisch toegepast bij authenticatie via een API-sleutel |
| `"long"`            | 1 uur         | Uitgebreide cache                         |
| `"none"`            | Geen caching     | Promptcaching uitschakelen                 |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Cache-overrides per agent">
    Gebruik parameters op modelniveau als basis en overschrijf vervolgens specifieke agents via `agents.entries.*.params`:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    Volgorde voor het samenvoegen van configuratie:

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params` (overeenkomend met `id`, overschrijft per sleutel)

    Hierdoor kan één agent een langlevende cache behouden, terwijl een andere agent met hetzelfde model caching uitschakelt voor piekverkeer of verkeer met weinig hergebruik.

  </Accordion>

  <Accordion title="Opmerkingen over Bedrock Claude">
    - Anthropic Claude-modellen op Bedrock (`amazon-bedrock/*anthropic.claude*`) accepteren doorvoer van `cacheRetention` wanneer dit is geconfigureerd.
    - Niet-Anthropic-modellen op Bedrock worden tijdens runtime gedwongen tot `cacheRetention: "none"`.
    - Slimme standaardinstellingen voor API-sleutels vullen ook `cacheRetention: "short"` in voor Claude-on-Bedrock-verwijzingen wanneer geen expliciete waarde is ingesteld.

  </Accordion>
</AccordionGroup>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Snelle modus">
    De gedeelde `/fast`-schakelaar van OpenClaw stelt het veld `service_tier` van Anthropic in voor rechtstreeks verkeer met een API-sleutel naar `api.anthropic.com`.

    | Opdracht | Wordt toegewezen aan |
    |---------|---------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - Alleen van toepassing op rechtstreekse `api.anthropic.com`-aanvragen met een API-sleutel. Aanvragen met OAuth-/abonnementstokens en proxyroutes krijgen nooit een veld `service_tier`.
    - Expliciete parameters `serviceTier` of `service_tier` overschrijven `/fast` wanneer beide zijn ingesteld.
    - Claude Opus 5 en Sonnet 5 ondersteunen Priority Tier niet, dus OpenClaw laat `service_tier` weg voor deze modellen.
    - Voor accounts zonder capaciteit voor Priority Tier kan `service_tier: "auto"` worden omgezet in `standard`.

    </Note>

  </Accordion>

  <Accordion title="Mediabegrip (afbeeldingen en PDF)">
    De gebundelde Anthropic-Plugin registreert begrip van afbeeldingen en PDF's. OpenClaw
    bepaalt mediacapaciteiten automatisch op basis van de geconfigureerde Anthropic-authenticatie;
    aanvullende configuratie is niet nodig.

    | Eigenschap        | Waarde                 |
    | --------------- | --------------------- |
    | Standaardmodel   | `claude-opus-5`       |
    | Ondersteunde invoer | Afbeeldingen, PDF-documenten |

    Wanneer een afbeelding of PDF aan een gesprek wordt toegevoegd, leidt OpenClaw deze automatisch
    via de Anthropic-provider voor mediabegrip.

  </Accordion>

  <Accordion title="Contextvenster van 1M">
    Claude Opus 5, Sonnet 5, Mythos 5 en Fable 5 hebben een exact
    invoervenster van 1.000.000 tokens en ondersteunen maximaal 128.000 uitvoertokens.
    Het contextvenster van 1M van Anthropic is ook algemeen beschikbaar voor Claude 4.x-modellen met adaptief
    denken: Opus 4.8,
    Opus 4.7, Opus 4.6 en Sonnet 4.6. OpenClaw stelt de grootte voor deze modellen
    automatisch in; `params.context1m` is niet nodig:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    Oudere configuraties kunnen `params.context1m: true` behouden; dit is een onschadelijke no-op voor
    deze modellen en OpenClaw verzendt de uitgefaseerde
    bètaheader `context-1m-2025-08-07` niet meer, ongeacht de configuratie. Oudere `anthropicBeta`-configuratie-items
    met die waarde worden verwijderd tijdens het bepalen van aanvraagheaders, en
    niet-ondersteunde oudere Claude-modellen behouden hun normale contextvenster.

    `params.context1m: true` gedraagt zich op dezelfde manier voor de Claude CLI-backend
    (`claude-cli/*`): in aanmerking komende Opus- en Sonnet-modellen met algemene beschikbaarheid krijgen het
    1M-venster al automatisch, dus ook daar is de parameter optioneel.

    <Warning>
    Vereist toegang tot lange contexten voor je Anthropic-referentie. Authenticatie met OAuth-/abonnementstokens behoudt de vereiste Anthropic-bètaheaders, maar OpenClaw verwijdert de uitgefaseerde 1M-bètaheader als deze nog in een oudere configuratie staat.
    </Warning>

  </Accordion>

  <Accordion title="Claude Opus 5-context van 1M">
    `anthropic/claude-opus-5` en de `claude-cli`-variant hebben standaard een contextvenster van 1M;
    `params.context1m: true` is niet nodig.
  </Accordion>
</AccordionGroup>

## Probleemoplossing

<AccordionGroup>
  <Accordion title="401-fouten / token plotseling ongeldig">
    Anthropic-tokenauthenticatie verloopt en kan worden ingetrokken. Gebruik voor nieuwe configuraties in plaats daarvan een Anthropic API-sleutel.
  </Accordion>

  <Accordion title='Geen API-sleutel gevonden voor provider "anthropic"'>
    Anthropic-authenticatie is **per agent**; nieuwe agents nemen de sleutels van de hoofdagent niet over. Voer de onboarding opnieuw uit voor die agent (of configureer een API-sleutel op de Gateway-host) en verifieer dit vervolgens met `openclaw models status`.
  </Accordion>

  <Accordion title='Geen inloggegevens gevonden voor profiel "anthropic:default"'>
    Voer `openclaw models status` uit om te zien welk authenticatieprofiel actief is. Voer de onboarding opnieuw uit of configureer een API-sleutel voor dat profielpad.
  </Accordion>

  <Accordion title="Geen beschikbaar authenticatieprofiel (allemaal in afkoelperiode)">
    Controleer `openclaw models status --json` voor `auth.unusableProfiles`. Afkoelperioden vanwege Anthropic-snelheidslimieten kunnen modelspecifiek zijn, waardoor een verwant Anthropic-model mogelijk nog wel bruikbaar is. Voeg een ander Anthropic-profiel toe of wacht tot de afkoelperiode voorbij is.
  </Accordion>
</AccordionGroup>

<Note>
Meer hulp: [Probleemoplossing](/nl/help/troubleshooting) en [Veelgestelde vragen](/nl/help/faq).
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="CLI-backends" href="/nl/gateway/cli-backends" icon="terminal">
    Configuratie- en runtimedetails van de Claude CLI-backend.
  </Card>
  <Card title="Promptcaching" href="/nl/reference/prompt-caching" icon="database">
    Hoe promptcaching bij verschillende providers werkt.
  </Card>
  <Card title="OAuth en authenticatie" href="/nl/gateway/authentication" icon="key">
    Authenticatiedetails en regels voor het hergebruik van inloggegevens.
  </Card>
</CardGroup>
