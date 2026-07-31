---
read_when:
    - Je wilt één beheerde sleutel voor meerdere modelproviders
    - Je hebt ClawRouter-modeldetectie of quotarapportage in OpenClaw nodig
summary: Routeer modellen met een specifieke referentie door ClawRouter en toon beheerde quota's
title: ClawRouter
x-i18n:
    generated_at: "2026-07-27T06:07:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 929a93e8d1d003e21f792d0fdab9542553ffab374f59d4d0505819b0f719591f
    source_path: providers/clawrouter.md
    workflow: 16
---

ClawRouter geeft OpenClaw één beleidsspecifieke sleutel voor meerdere upstreammodelproviders. De gebundelde Plugin `clawrouter` detecteert alleen de modellen die voor die sleutel zijn toegestaan, routeert elk model via het opgegeven protocol en rapporteert het budget en het totale gebruik van de sleutel in de gebruiksoverzichten van OpenClaw.

Upstreamreferenties en providerspecifieke doorsturing blijven in ClawRouter, zodat je nooit elke upstreamprovider-Plugin op de OpenClaw-host hoeft te installeren of te authenticeren. De Plugin wordt gebundeld met OpenClaw geleverd (`enabledByDefault: true`); je hebt alleen een uitgegeven ClawRouter-referentie nodig.

| Eigenschap    | Waarde                                   |
| ------------- | ---------------------------------------- |
| Provider      | `clawrouter`                       |
| Plugin        | gebundeld (opgenomen in OpenClaw)        |
| Authenticatie | `CLAWROUTER_API_KEY`                       |
| Standaard-URL | `https://clawrouter.openclaw.ai`                       |
| Modelcatalogus | Referentiespecifiek via `/v1/catalog` |
| Quota's       | Maandbudget en gebruik via `/v1/usage` |

## Aan de slag

<Steps>
  <Step title="Een specifieke referentie verkrijgen">
    Vraag je ClawRouter-beheerder om een referentie waarvan het beleid de
    providers, modellen en het maandbudget omvat die je moet gebruiken. Referenties
    worden bij uitgifte één keer weergegeven.
  </Step>
  <Step title="OpenClaw configureren">
    ```bash
    export CLAWROUTER_API_KEY="..."
    openclaw onboard --auth-choice clawrouter-api-key
    openclaw plugins enable clawrouter
    ```

    `clawrouter` is gebundeld en standaard ingeschakeld. Als je configuratie
    `plugins.allow` instelt, voeg je `clawrouter` aan die lijst toe voordat
    je de Plugin inschakelt. Stel voor een aangepaste implementatie
    `models.providers.clawrouter.baseUrl` in op de ClawRouter-origin; de standaardwaarde is
    `https://clawrouter.openclaw.ai`.

  </Step>
  <Step title="Toegekende modellen weergeven">
    ```bash
    openclaw models list --all --provider clawrouter
    ```

    Gebruik de geretourneerde modelreferenties exact zoals weergegeven. Ze behouden
    de upstreamnaamruimte, zoals `clawrouter/openai/gpt-5.5`,
    `clawrouter/anthropic/claude-sonnet-4-6` of
    `clawrouter/google/gemini-3.5-flash`. Als `agents.defaults.modelPolicy.allow`
    is geconfigureerd, voeg je elke geselecteerde ClawRouter-referentie eraan toe.

  </Step>
  <Step title="Een model selecteren">
    ```bash
    openclaw models set clawrouter/<provider>/<model>
    ```

    Je kunt voor één uitvoering ook een geretourneerd model selecteren met
    `openclaw agent --model clawrouter/<provider>/<model> --message "..."`.

  </Step>
</Steps>

## Beheerde niet-interactieve implementatie

Bewaar de proxysleutel in de geheime-injectie van de workload en sla in
`openclaw.json` alleen een SecretRef op. De canonieke beheerde velden zijn:

| Doel          | Configuratie- of omgevingsveld                                            |
| ------------- | ------------------------------------------------------------------------- |
| Router-origin | `models.providers.clawrouter.baseUrl`                                                        |
| Referentie    | `models.providers.clawrouter.apiKey` -> SecretRef uit omgeving                             |
| Geheime waarde | `CLAWROUTER_API_KEY` in de procesomgeving van de Gateway                   |
| Standaardmodel | `agents.defaults.model.primary` -> `clawrouter/<provider>/<model>`                                 |
| Workloadtag   | `models.providers.clawrouter.headers.X-ClawRouter-Project-Id` (optioneel)                                           |

Een implementatiecontroller kan bijvoorbeeld deze JSON5-patch beheren:

```json5
{
  plugins: {
    entries: { clawrouter: { enabled: true } },
  },
  models: {
    providers: {
      clawrouter: {
        baseUrl: "https://clawrouter.internal.example",
        apiKey: {
          source: "env",
          provider: "default",
          id: "CLAWROUTER_API_KEY",
        },
        headers: {
          "X-ClawRouter-Project-Id": "fakeco",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "clawrouter/openai/gpt-5.5" },
    },
  },
}
```

Als de implementatie `plugins.allow` instelt, behoud je de bestaande vermeldingen
en voeg je `clawrouter` toe. Valideer en pas toe zonder interactieve wizard:

```bash
openclaw config patch --file ./clawrouter.patch.json5 --dry-run --json
openclaw config patch --file ./clawrouter.patch.json5
```

De proefuitvoering lost de SecretRef op, maar drukt de waarde nooit af. Om de
referentie te roteren, werk je de externe Secret bij die `CLAWROUTER_API_KEY`
levert en start je de Gateway-workload opnieuw, zodat de nieuwe procesomgeving
wordt geladen. Het configuratiebestand en de modelreferentie veranderen niet.

Voor een zelfstandig vanuit broncode gebouwde Docker-Gateway is ClawRouter al
opgenomen in de root-runtime. Selecteer alleen de kanaal-Plugin waarvoor aparte
verpakking nodig is, zoals `OPENCLAW_EXTENSIONS=clickclack`, `slack` of
`msteams`; zie
[vanuit broncode gebouwde images met geselecteerde Plugins](/nl/install/docker#source-built-images-with-selected-plugins).
Archief-/appliance-implementaties moeten dezelfde opgenomen broncode via hun
eigen artefactpijplijn verpakken in plaats van de OCI-image te gebruiken.

## Gereedheid en live bewijs

Deze controles bewijzen verschillende grenzen; vervang de ene niet door de andere:

```bash
# Alleen de status van het ClawRouter-proces; er wordt geen referentie of upstreammodel gebruikt.
curl -fsS https://clawrouter.internal.example/v1/health

# Alleen de opstartgereedheid van de OpenClaw-Gateway; er wordt geen modelaanroep uitgevoerd.
curl -fsS http://127.0.0.1:18789/readyz

# Referentiespecifieke catalogusdetectie.
openclaw models list --all --provider clawrouter --json

# Minimale echte inferentieprobe via de geconfigureerde ClawRouter-provider.
openclaw models status --probe --probe-provider clawrouter --probe-max-tokens 8 --json

# Workload-canary met een exacte toegekende modelreferentie.
openclaw agent --agent main \
  --model clawrouter/openai/gpt-5.5 \
  --message "Antwoord exact: CLAWROUTER_CANARY_OK" \
  --json
```

Gebruik een model dat door de specifieke catalogus wordt geretourneerd in plaats
van het voorbeeldmodel klakkeloos te kopiëren. Een geslaagd
`/readyz`-antwoord betekent dat de Gateway aanvragen kan verwerken;
het bewijst niet dat ClawRouter, de bijbehorende referentie of een
upstreamprovider gereed is. De modelprobe en agent-canary vormen het
inferentiebewijs.

Voer voor live diagnostiek de canary uit en inspecteer de standaardlogs van de
Gateway. De bestaande diagnostiek voor modeltransport met alleen metadata
produceert regels met de volgende vorm:

```text
[model-fetch] start provider=clawrouter api=openai-responses model=openai/gpt-5.5 method=POST url=https://clawrouter.internal.example/v1/responses
[model-fetch] response provider=clawrouter api=openai-responses model=openai/gpt-5.5 status=200
```

De Plugin verzendt begrensde headers `X-ClawRouter-Client`,
`X-ClawRouter-Agent-Id` en `X-ClawRouter-Session-Id` wanneer die identificatoren
beschikbaar zijn. De Plugin koppelt ook de diagnostische
`callId` (`<run-id>:model:<n>`) van de modelaanroep aan
`X-Request-ID`, zodat een modelaanroepgebeurtenis van OpenClaw kan worden
gekoppeld aan het auditspoor van ClawRouter dat alleen metadata bevat. Waarden
binnen het budget van 128 tekens voor de aanvraag-ID zijn identiek. Langere
waarden behouden het achtervoegsel `:model:<n>` en een deterministische
hash, zodat afzonderlijke aanroepen begrensd en koppelbaar blijven. Statische
implementatiemetadata zoals `X-ClawRouter-Project-Id` kunnen worden ingesteld in de
provider-map `headers`. Headers voor agent- en sessietoeschrijving
behouden hun afzonderlijke limiet van 256 tekens. Automatische aanvraag-ID's
met tekens buiten de ASCII-identificatorenset van ClawRouter gebruiken dezelfde
deterministische begrensde vorm.
Expliciet geconfigureerde headers, inclusief elke variant in hoofdlettergebruik
van `X-Request-ID`, hebben voorrang op automatische waarden. De
transportdiagnostiek registreert routerings- en antwoordmetadata; deze logt
geen referenties, aanvraag-ID's, prompts of voltooiingen. De eigen
auditgebeurtenis van ClawRouter bevat de geselecteerde upstreamprovider en de
status voor het bewaren van inhoud.

## Modeldetectie

`GET /v1/catalog` retourneert `{ providers: [...] }`, waarbij elke
providervermelding de eigen `models[]` vermeldt (met upstream-ID,
mogelijkheden en prijzen) en de ondersteunde aanvraagroutes. OpenClaw levert
geen tweede, vaste lijst met ClawRouter-modellen. Een catalogusmodel wordt als
OpenClaw-model aangeboden wanneer:

- het beleid van de referentie de provider ervan toestaat;
- het catalogusmodel een ondersteunde LLM-mogelijkheid aankondigt
  (`llm.responses`, `llm.chat`, `llm.messages` of
  `llm.stream` met een overeenkomende streamingroute); en
- de provider een overeenkomende route beschikbaar stelt voor
  een van de onderstaande transporten.

Voor het toevoegen van een model aan een ondersteunde ClawRouter-provider is
geen OpenClaw-release nodig: de volgende catalogusvernieuwing (60 seconden per
referentiebereik in de cache) detecteert het. Voor een model waarvoor een nieuw
wire-protocol nodig is, moet eerst ondersteuning aan de Plugin worden toegevoegd.

## Protocol- en provider-Plugins

ClawRouter beheert upstreamreferenties; de catalogus vertelt OpenClaw welk
transport moet worden gebruikt, zodat je nooit de authenticatie-Plugin van elk
upstreambedrijf hoeft te installeren.

| Catalogusmogelijkheid/-route                            | OpenClaw-transport     |
| ------------------------------------------------------ | ---------------------- |
| `llm.responses` (OpenAI-compatibele provider)       | `openai-responses`     |
| `llm.chat` (OpenAI-compatibele provider)       | `openai-completions`     |
| `llm.messages` + route `anthropic.messages`          | `anthropic-messages`     |
| `llm.stream` + streamingroute `google.generate_content` | `google-generative-ai`     |

De Plugin past ook het bijbehorende beleid voor opnieuw afspelen en
toolschema's toe op die families (compatibiliteit van toolschema's voor
OpenAI/DeepSeek/Gemini/Perplexity; native beleid voor opnieuw afspelen van
Anthropic en Google Gemini). Perplexity-modellen krijgen een strikte
schemaherschrijving: `patternProperties` en `additionalProperties` worden
verwijderd en elk objectschema declareert `properties`, omdat Perplexity
toolschema's zonder deze declaraties afwijst. Een catalogusprovider die alleen
een niet-ondersteunde aanvraagindeling beschikbaar stelt, wordt bewust niet
aangeboden als OpenClaw-tekstmodel. Normaliseer die providers in ClawRouter
naar een van de ondersteunde contracten in plaats van een incompatibele
payload te verzenden.

## Quota's en gebruik

Het antwoord `/v1/usage` van ClawRouter voedt de normale
providergebruiksweergaven van OpenClaw: totalen voor aanvragen, tokens en
uitgaven, plus een maandbudgetvenster wanneer de sleutel een limiet heeft.
Sleutels zonder meting tonen nog steeds het totale gebruik, maar zonder
percentagevenster.

Bij het opzoeken van quota's wordt dezelfde specifieke sleutel gebruikt als bij
modeldetectie. Een mislukte quotaopzoeking blokkeert de modeluitvoering niet.

Controleer de live momentopname met:

```bash
openclaw status --usage
openclaw models status
```

Dezelfde providermomentopname is beschikbaar voor `/status` in de chat
en in de gebruiksinterface van OpenClaw. Het budget geldt voor het hele beleid,
dus aanvragen van een andere client die hetzelfde ClawRouter-beleid gebruikt,
kunnen het resterende percentage veranderen.

## Probleemoplossing

| Symptoom                                 | Controle                                                                                                                                       |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Geen ClawRouter-modellen                 | Controleer of de Plugin is ingeschakeld en toegestaan door `plugins.allow` en controleer vervolgens of de referentie actief is en toegang geeft tot ten minste één gereedstaande provider. |
| Een geconfigureerd ClawRouter-model ontbreekt | Inspecteer de mogelijkheid `/v1/catalog` en de routeondersteuning. Niet-ondersteunde transportcontracten worden bewust uitgefilterd. |
| Modeloverschrijving geweigerd door beleid | Voeg de exacte catalogusreferentie of `clawrouter/*` toe aan `agents.defaults.modelPolicy.allow`. |
| `401` of `403` uit catalogus of gebruik | Geef de ClawRouter-referentie opnieuw uit of pas het bereik aan; OpenClaw valt niet terug op upstreamprovidersleutels. |
| Modelaanroep mislukt na detectie         | Controleer in ClawRouter de providerverbinding en de status van de upstreamprovider en probeer het opnieuw nadat de gereedheidsstatus is hersteld. |
| Gebruik bevat totalen maar geen percentage | Het beleid heeft geen meting; voeg in ClawRouter een maandbudget toe om een percentagevenster beschikbaar te maken. |

## Beveiligingsgedrag

- Catalogusdetectie is beperkt tot de geconfigureerde proxysleutel en wordt per referentiebereik in de cache opgeslagen (agentmap, werkruimtemap, authenticatieprofiel-id en basis-URL).
- De proxysleutel wordt alleen bij het verzenden van de aanvraag toegevoegd; deze wordt niet opgeslagen in de modelmetadata.
- Waarden voor automatische toeschrijving en aanvraagcorrelatie worden vóór verzending ontdaan van witruimte en bij controletekens geweigerd. Toeschrijvingswaarden zijn beperkt tot 256 tekens; aanvraag-id's tot 128.
- Diagnostische gegevens over het modeltransport bevatten alleen metadata en nooit de proxysleutel of modelinhoud.
- Model-id's van native Anthropic- en Gemini-modellen worden alleen bij verzending herschreven naar hun upstream-id's.
- Niet-ondersteunde catalogusrijen of catalogusrijen waarvoor geen toestemming is verleend, worden standaard geweigerd en kunnen niet worden geselecteerd.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Providerconfiguratie en modelselectie.
  </Card>
  <Card title="Gebruiksregistratie" href="/nl/concepts/usage-tracking" icon="chart-line">
    Gebruiks- en statusoverzichten van OpenClaw.
  </Card>
</CardGroup>
