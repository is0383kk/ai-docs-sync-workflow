---
read_when:
    - Je wilt OpenAI-modellen gebruiken in OpenClaw
    - Je wilt Codex-abonnementsauthenticatie in plaats van API-sleutels
    - Je hebt strikter uitvoeringsgedrag voor GPT-5-agents nodig
summary: Gebruik OpenAI via API-sleutels of een Codex-abonnement in OpenClaw
title: OpenAI
x-i18n:
    generated_at: "2026-07-27T06:06:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 612a36760899e01126364ddca523f0a6340036253cf349ae2755ba15c6451ba6
    source_path: providers/openai.md
    workflow: 16
---

OpenClaw gebruikt één provider-id, `openai`, voor zowel directe authenticatie met een API-sleutel als
ChatGPT/Codex-abonnementsauthenticatie. `openai/*` is de canonieke modelroute.
Voor ingebedde agentbeurten waarbij het runtimebeleid niet is ingesteld of `auto` is, bepalen de routegegevens
van OpenAI of OpenClaw impliciet de gebundelde Codex-app-serverruntime
mag selecteren. Alleen het voorvoegsel `openai/*` selecteert geen runtime.

- **Agentmodellen** - `openai/*` via de runtime die is geselecteerd door expliciete
  `agentRuntime`-configuratie of het impliciete routebeleid van OpenAI. Meld je aan met Codex-
  authenticatie om een ChatGPT/Codex-abonnement te gebruiken, of configureer een authenticatieprofiel
  met API-sleutel wanneer je facturering op basis van een sleutel wilt.
- **OpenAI-API's zonder agent** - directe toegang tot OpenAI Platform, gefactureerd per gebruik,
  via `OPENAI_API_KEY` of een `openai`-authenticatieprofiel met API-sleutel.
- **Verouderde configuratie** - verwijzingen naar `codex/*` en `openai-codex/*` worden door
  `openclaw doctor --fix` hersteld naar `openai/*` plus modelgebonden
  `agentRuntime.id: "codex"`.

OpenAI ondersteunt expliciet het gebruik van OAuth-abonnementen in externe tools en
workflows zoals OpenClaw.

## Gebruiks- en kostentracering

OpenClaw houdt abonnementsquota en facturering voor de Platform-API gescheiden:

- ChatGPT/Codex OAuth toont het abonnementsplan, quotumperioden en creditsaldo.
- `OPENAI_ADMIN_KEY` toont 30 dagen aan door de provider gerapporteerde organisatiekosten en completions-gebruik in **Gebruik** van de Control UI, inclusief dagelijkse uitgaven, totalen voor aanvragen/tokens, meestgebruikte modellen en kostencategorieën.
- `OPENAI_PROJECT_ID` beperkt de geschiedenis van de Admin API optioneel tot één project.
- OpenClaw stuurt nooit `OPENAI_API_KEY` of een `openai`-inferentieprofiel naar organisatie-API's; die aanmeldgegevens kunnen bij aangepaste, Azure- of agentlokale eindpunten horen.

Een expliciete beheerderssleutel heeft voorrang op OAuth. Door de provider gerapporteerde geschiedenis wordt niet samengevoegd met de uit sessies afgeleide geschatte kosten van OpenClaw; deze kan API-activiteit van andere clients en factureringscorrecties van de provider bevatten.

De documentatie van OpenAI over het [dashboard voor API-gebruik](https://help.openai.com/en/articles/10478918) beschrijft de vereisten voor organisatie-eigenaars en expliciete machtigingen voor het Usage Dashboard om gebruiksgegevens te bekijken.

Provider, model, runtime en kanaal zijn afzonderlijke lagen. Als deze labels
door elkaar raken, lees dan [Agentruntimes](/nl/concepts/agent-runtimes) voordat je
de configuratie wijzigt.

## Snelle keuze

| Doel                                              | Gebruik                                                                | Opmerkingen                                                               |
| ------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| ChatGPT/Codex-abonnement, systeemeigen Codex-runtime  | `openai/gpt-5.6-sol`                                               | Nieuwe abonnementsconfiguratie; meld je aan met Codex-authenticatie.                  |
| Directe facturering met API-sleutel voor agentbeurten            | `openai/gpt-5.6` plus een geordend authenticatieprofiel met API-sleutel              | Nieuwe configuratie met API-sleutel; de kale directe API-id wordt omgezet naar Sol.        |
| Een exacte GPT-5.6-laag kiezen                      | `openai/gpt-5.6-sol`, `-terra` of `-luna`                         | Controleer `models list` voor de lagen die voor dit account beschikbaar zijn.        |
| Account zonder toegang tot GPT-5.6                    | `openai/gpt-5.5`                                                   | Expliciete herstelkeuze; OpenClaw schakelt niet stilzwijgend terug.     |
| Directe facturering met API-sleutel, expliciete OpenClaw-runtime | `openai/gpt-5.6` plus provider/model `agentRuntime.id: "openclaw"` | Selecteer een normaal `openai`-authenticatieprofiel met API-sleutel.                           |
| Alias voor het nieuwste ChatGPT Instant-model                | `openai/chat-latest`                                               | Alleen directe API-sleutel; veranderlijke alias, niet de stabiele standaard.          |
| Afbeeldingen genereren of bewerken                       | `openai/gpt-image-2`                                               | Werkt met `OPENAI_API_KEY` of Codex OAuth.                         |
| Afbeeldingen met transparante achtergrond                     | `openai/gpt-image-1.5`                                             | Stel `outputFormat` in op `png` of `webp` en `background=transparent`. |

## Naamgevingsschema

| Naam die je ziet                            | Laag             | Betekenis                                                                                  |
| --------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------- |
| `openai`                                | Providervoorvoegsel   | Canonieke OpenAI-modelroute; routegegevens bepalen de impliciete runtime.                |
| `codex`-plugin                          | Plugin            | Gebundelde plugin die de systeemeigen Codex-app-serverruntime en `/codex`-chatbesturing biedt. |
| provider/model `agentRuntime.id: codex` | Agentruntime     | Dwing de systeemeigen Codex-app-serverharnas af voor overeenkomende ingebedde beurten.                   |
| `/codex ...`                            | Chatopdrachtenset  | Koppel en beheer Codex-app-serverthreads vanuit een gesprek.                               |
| `runtime: "acp", agentId: "codex"`      | ACP-sessieroute | Expliciet terugvalpad dat Codex via ACP/acpx uitvoert.                                 |

## Impliciete agentruntime

Wanneer het provider/modelbeleid voor `agentRuntime` niet is ingesteld of `auto` is, kiest het
providergebonden routebeleid van OpenAI de impliciete runtime op basis van het effectieve
eindpunt en de adapter:

| Effectieve routegegevens                                                                                                                                                  | Impliciete runtime      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| Exact officieel Platform-HTTPS-eindpunt met `openai-responses`, of exact officieel ChatGPT-HTTPS-eindpunt met `openai-chatgpt-responses`; geen zelf ingestelde aanvraagoverschrijving | Codex kan worden geselecteerd |
| Zelf ingestelde `openai-completions`-adapter                                                                                                                                  | OpenClaw              |
| Aangepast eindpunt                                                                                                                                                        | OpenClaw              |
| Expliciet exact officieel eindpunt via HTTP                                                                                                                            | Geweigerd              |
| Route met een zelf ingestelde provider/model-aanvraagoverschrijving                                                                                                                 | OpenClaw              |

Een expliciete niet-standaard provider/model-`agentRuntime.id` blijft bepalend.
Zo houdt `agentRuntime.id: "openclaw"` een route die anders voor Codex in aanmerking komt
op OpenClaw, terwijl `agentRuntime.id: "codex"` Codex vereist en
gesloten faalt wanneer de effectieve route niet als Codex-compatibel is gedeclareerd.
Runtimeselectie verandert het type aanmeldgegevens of de facturering niet: authenticatie met een API-sleutel
voor de Platform-API en ChatGPT/Codex-abonnementsauthenticatie blijven gescheiden.

`openclaw doctor --fix` migreert verouderde modelverwijzingen naar `codex/*` en `openai-codex/*`,
verouderde Codex-authenticatieprofiel-id's en verouderde Codex-vermeldingen voor authenticatievolgorde naar de
canonieke `openai`-route. Gemigreerde modelverwijzingen krijgen modelgebonden
`agentRuntime.id: "codex"`; gebruik `auth.order.openai` voor nieuwe configuratie van de authenticatievolgorde.

<Note>
Een nieuwe OpenAI-configuratie past alleen een GPT-5.6-primair model toe wanneer er geen primair model is
geconfigureerd. Het toevoegen of vernieuwen van OpenAI-authenticatie behoudt een bestaande expliciete
selectie, inclusief `openai/gpt-5.5`, tenzij je expliciet
`models auth login --set-default` of `models set` gebruikt. Gebruik alleen een authenticatieprofiel met API-sleutel
wanneer je authenticatie met een API-sleutel voor een agentmodel wilt.
</Note>

## Beperkte preview van GPT-5.6

OpenClaw herkent de exacte model-id's `openai/gpt-5.6-sol`,
`openai/gpt-5.6-terra` en `openai/gpt-5.6-luna`. Alle drie bieden
`xhigh`- en `max`-redenering in de huidige catalogus. OpenAI beschrijft Sol als
de vlaggenschiplaag, Terra als de uitgebalanceerde laag en Luna als de snelle,
goedkopere laag. Zie de
[aankondiging van de lancering van GPT-5.6](https://openai.com/index/previewing-gpt-5-6-sol/)
en de [toegangsgids](https://help.openai.com/en/articles/20001325-a-preview-of-gpt-5-6-sol-terra-and-luna).

Bij directe OpenAI-authenticatie met een API-sleutel is de kale `openai/gpt-5.6`-id een alias voor
Sol en de standaard voor een nieuwe configuratie. De systeemeigen Codex-catalogus past
die directe API-alias niet aan de clientzijde toe; afhankelijk van de werkruimtetoegang kan deze
de exacte Sol-, Terra- en Luna-id's tonen. Een nieuwe ChatGPT/Codex OAuth-configuratie gebruikt daarom
`openai/gpt-5.6-sol`. Controleer het huidige account met:

```bash
openclaw models list --provider openai
```

Toegang voor de API-organisatie en de Codex-werkruimte kan verschillen. Als GPT-5.6 niet
beschikbaar is, selecteer GPT-5.5 dan expliciet:

```bash
openclaw models set openai/gpt-5.5
```

OpenClaw toont de upstream-toegangsfout en vervangt een
GPT-5.6-selectie niet stilzwijgend door GPT-5.5.

<Note>
Exacte officiële HTTPS-routes die in aanmerking komen, kunnen de gebundelde Codex-app-serverplugin
selecteren wanneer het runtimebeleid niet is ingesteld of `auto` is; zelf ingestelde Completions-routes,
aangepaste eindpunten en overschrijvingen van aanvraagtransport blijven op OpenClaw. Officiële HTTP-eindpunten
met platte tekst worden geweigerd. Expliciete provider/model-runtimeconfiguratie blijft
bepalend. Voer `openclaw doctor --fix` uit om verouderde Codex-modelverwijzingen,
`codex-cli/*`-verwijzingen of oude runtimesessiepins te herstellen die niet door
expliciete runtimeconfiguratie zijn ingesteld.
</Note>

## Functiedekking van OpenClaw

| OpenAI-mogelijkheid         | OpenClaw-oppervlak                                                                              | Status                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Chat / Responses          | `openai/<model>`-modelprovider                                                               | Ja                                                             |
| Codex-abonnementsmodellen | `openai/<model>` met OpenAI OAuth                                                            | Ja                                                             |
| Verouderde Codex-modelverwijzingen   | oude Codex-modelverwijzingen, `codex-cli/<model>`                                                     | Door doctor hersteld naar `openai/<model>`                          |
| Codex-app-serverharnas  | Codex-compatibele HTTPS-route met runtime niet ingesteld/`auto`, of expliciete `agentRuntime.id: codex`  | Ja                                                             |
| Webzoeken aan serverzijde    | Ingebouwde OpenAI Responses-tool                                                                  | Ja, wanneer webzoeken is ingeschakeld en geen andere provider is vastgezet |
| Afbeeldingen                    | `image_generate`                                                                              | Ja                                                             |
| Video's                    | `video_generate`                                                                              | Ja                                                             |
| Tekst-naar-spraak            | `tts.provider: "openai"` / `tts`                                                              | Ja                                                             |
| Batchgewijze spraak-naar-tekst      | `tools.media.audio` / mediabegrip                                                     | Ja                                                             |
| Streamende spraak-naar-tekst  | Voice Call `streaming.provider: "openai"`                                                     | Ja                                                             |
| Realtime spraak            | Voice Call `realtime.provider: "openai"` / Control UI Talk `talk.realtime.provider: "openai"` | Ja (OpenAI Platform-API-sleutel)                                   |
| Embeddings                | provider voor geheugenembeddings                                                                     | Ja                                                             |

<Note>
OpenAI Realtime-spraak verloopt via de openbare **OpenAI Platform Realtime
API** en vereist een Platform-API-sleutel. Codex OAuth-tokens verifiëren in
plaats daarvan de ChatGPT Codex-backend; ze zijn niet uitwisselbaar met Platform-API-
sleutels voor de openbare Realtime-eindpunten.

Als authenticatie met een API-sleutel meldt dat facturering ontbreekt, vul je Platform-tegoed aan via
[platform.openai.com/account/billing](https://platform.openai.com/account/billing)
voor de organisatie achter je realtime-referenties wanneer je authenticatie met een API-sleutel
gebruikt. Realtime-spraak accepteert het `openai`-authenticatieprofiel met API-sleutel dat is aangemaakt door
`openclaw onboard --auth-choice openai-api-key`, een Platform-API-sleutel die via
`talk.realtime.providers.openai.apiKey` is ingesteld voor Control UI Talk, of
`plugins.entries.voice-call.config.realtime.providers.openai.apiKey` voor Voice
Call, of de omgevingsvariabele `OPENAI_API_KEY`.

In Control UI Video Talk ontvangt OpenAI WebRTC op aanvraag cameracontext:
wanneer het model `describe_view` aanroept, verzendt de browser één begrensde JPEG via
het realtime-datakanaal. OpenClaw koppelt geen continue cameratrack
aan de OpenAI-sessie.
</Note>

## Geheugenembeddings

OpenClaw kan OpenAI, of een OpenAI-compatibel embedding-eindpunt, gebruiken voor
`memory_search`-indexering en query-embeddings:

```json5
{
  memory: {
    search: {
      provider: "openai",
      model: "text-embedding-3-small",
    },
  },
}
```

Stel voor OpenAI-compatibele eindpunten die asymmetrische embeddinglabels vereisen
`queryInputType` en `documentInputType` in onder `memory.search`. OpenClaw
stuurt deze door als providerspecifieke `input_type`-aanvraagvelden: query-
embeddings gebruiken `queryInputType`; geïndexeerde geheugenfragmenten en batchindexering gebruiken
`documentInputType`. Zie de
[Referentie voor geheugenconfiguratie](/nl/reference/memory-config#provider-specific-config)
voor het volledige voorbeeld.

## Aan de slag

<Tabs>
  <Tab title="API-sleutel (OpenAI Platform)">
    **Meest geschikt voor:** directe API-toegang en facturering op basis van gebruik.

    <Steps>
      <Step title="Je API-sleutel ophalen">
        Maak of kopieer een API-sleutel vanuit het [OpenAI Platform-dashboard](https://platform.openai.com/api-keys).
      </Step>
      <Step title="Onboarding uitvoeren">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        Of geef de sleutel rechtstreeks door:

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="Controleren of het model beschikbaar is">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### Routesamenvatting

    | Modelverwijzing        | Runtimebeleid of routefeiten                                 | Route                     | Authenticatie                              |
    | ---------------- | ------------------------------------------------------------- | ------------------------- | --------------------------------- |
    | `openai/gpt-5.6` | niet ingesteld/`auto`, exacte officiële systeemeigen HTTPS-route, geen aanvraagoverschrijving | Codex kan worden geselecteerd     | Geordend authenticatieprofiel met API-sleutel      |
    | `openai/gpt-5.6` | provider/model `agentRuntime.id: "openclaw"`                  | Ingebouwde OpenClaw-runtime | Geselecteerd `openai`-profiel met API-sleutel |
    | `openai/gpt-5.5` | expliciete provider/model `agentRuntime.id`                     | Geselecteerde agentruntime    | Geselecteerd OpenAI-profiel met API-sleutel   |
    | `openai/*`       | geschreven Completions, aangepast of aanvraagoverschrijving | Ingebouwde OpenClaw-runtime | Referentietype blijft ongewijzigd |
    | `openai/*`       | officieel HTTP-eindpunt met platte tekst                  | Geweigerd                 | Referentie wordt niet verzonden             |

    <Note>
    Als de runtime niet is ingesteld of `auto` is, mag alleen een geschikte, exacte officiële systeemeigen
    HTTPS-route impliciet het Codex-app-serverharnas selecteren. Maak voor authenticatie met een API-sleutel
    op een agentmodel een `openai`-authenticatieprofiel met API-sleutel en orden dit met
    `auth.order.openai`; `OPENAI_API_KEY` blijft de rechtstreekse terugvaloptie voor
    OpenAI API-oppervlakken die niet voor agents zijn. Voer `openclaw doctor --fix` uit om oudere
    verouderde vermeldingen voor de Codex-authenticatievolgorde te migreren.
    </Note>

    ### Configuratievoorbeeld

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
    }
    ```

    De kale directe-API-id `gpt-5.6` wordt omgezet naar het Sol-niveau. Als deze API-
    organisatie GPT-5.6 niet aanbiedt, stel je het primaire model expliciet in op
    `openai/gpt-5.5`.

    Stel het model in op `openai/chat-latest` om het huidige Instant-model van ChatGPT via de OpenAI API te proberen:

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` is een veranderende alias. Een nieuwe installatie met een OpenAI-API-sleutel gebruikt in plaats daarvan
    `openai/gpt-5.6`, waarvan de kale directe-API-id wordt omgezet naar Sol. Bestaande
    expliciete primaire modellen, waaronder `openai/gpt-5.5`, blijven ongewijzigd. De
    alias `chat-latest` accepteert alleen tekstuitgebreidheid `medium`; OpenClaw dwingt
    elke andere aangevraagde uitgebreidheid voor dit model af op `medium`.

    <Warning>
    OpenClaw stelt `gpt-5.3-codex-spark` **niet** beschikbaar via de rechtstreekse OpenAI-
    route met API-sleutel. Het is alleen beschikbaar via vermeldingen in de Codex-abonnementscatalogus
    wanneer je aangemelde account het beschikbaar stelt.
    </Warning>

  </Tab>

  <Tab title="Codex-abonnement">
    **Meest geschikt voor:** je ChatGPT/Codex-abonnement gebruiken met systeemeigen uitvoering via de Codex-
    app-server in plaats van een afzonderlijke API-sleutel. Codex-cloud vereist
    aanmelding bij ChatGPT.

    <Steps>
      <Step title="Codex OAuth uitvoeren">
        ```bash
        openclaw onboard --auth-choice openai
        ```

        Of voer OAuth rechtstreeks uit:

        ```bash
        openclaw models auth login --provider openai
        ```

        Voeg voor headless-installaties of installaties waar callbacks problemen opleveren `--device-code` toe om
        je aan te melden met een ChatGPT-apparaatcodestroom in plaats van de browsercallback
        via localhost:

        ```bash
        openclaw models auth login --provider openai --device-code
        ```
      </Step>
      <Step title="De canonieke OpenAI-modelroute gebruiken">
        ```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.6-sol
        ```

        Voor deze exacte officiële systeemeigen HTTPS-route is geen runtimeconfiguratie
        vereist. Deze kan automatisch de Codex-app-serverruntime selecteren, en
        OpenClaw installeert of herstelt de meegeleverde Codex-plugin wanneer die runtime
        wordt gekozen.
      </Step>
      <Step title="Controleren of Codex-authenticatie beschikbaar is">
        ```bash
        openclaw models list --provider openai
        ```

        Nadat de Gateway actief is, verzend je `/codex status` of `/codex models`
        in de chat om de systeemeigen app-serverruntime te controleren.
      </Step>
    </Steps>

    ### Routesamenvatting

    | Modelverwijzing                | Runtimebeleid of routefeiten                                 | Route                                                    | Authenticatie                                               |
    | ------------------------ | ------------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------- |
    | `openai/gpt-5.6-sol`     | niet ingesteld/`auto`, exacte officiële systeemeigen HTTPS-route, geen aanvraagoverschrijving | Codex kan worden geselecteerd                                    | Codex-aanmelding, of een geordend `openai`-authenticatieprofiel |
    | `openai/gpt-5.6-terra`   | niet ingesteld/`auto`, exacte officiële systeemeigen HTTPS-route, geen aanvraagoverschrijving | Codex kan worden geselecteerd                                    | Codex-aanmelding wanneer de catalogus Terra beschikbaar stelt       |
    | `openai/gpt-5.6-luna`    | niet ingesteld/`auto`, exacte officiële systeemeigen HTTPS-route, geen aanvraagoverschrijving | Codex kan worden geselecteerd                                    | Codex-aanmelding wanneer de catalogus Luna beschikbaar stelt        |
    | `openai/gpt-5.6-sol`     | provider/model `agentRuntime.id: "openclaw"`                  | Ingebouwde OpenClaw-runtime, intern Codex-authenticatievervoer | Geselecteerd `openai` OAuth-profiel                    |
    | `openai/gpt-5.5`         | expliciete provider/model `agentRuntime.id`                     | Geselecteerde agentruntime                                   | Geselecteerd OpenAI-authenticatieprofiel                       |
    | `openai/*`               | geschreven Completions, aangepast of aanvraagoverschrijving | Ingebouwde OpenClaw-runtime                                | Referentievereiste blijft routespecifiek      |
    | `openai/*`               | officieel HTTP-eindpunt met platte tekst                  | Geweigerd                                                 | Referentie wordt niet verzonden                              |
    | Verouderde Codex GPT-5.5-verwijzing | door doctor hersteld                                            | Herschreven naar `openai/gpt-5.5`                            | Gemigreerd OpenAI OAuth-profiel                      |
    | `codex-cli/gpt-5.5`      | door doctor hersteld                                            | Herschreven naar `openai/gpt-5.5`                            | Codex-app-serverauthenticatie                              |

    <Warning>
    Een nieuwe configuratie op basis van een abonnement gebruikt exact `openai/gpt-5.6-sol`; de
    systeemeigen Codex-catalogus kan ook exacte Terra- of Luna-referenties aanbieden. Als het
    account GPT-5.6 niet aanbiedt, selecteer dan expliciet `openai/gpt-5.5`. Oudere
    Codex GPT-referenties zijn verouderde OpenClaw-routes, niet het systeemeigen runtimepad
    van Codex; voer `openclaw doctor --fix` uit om ze te migreren zonder een
    bestaande expliciete GPT-5.5-selectie te upgraden. `gpt-5.3-codex-spark` blijft beperkt
    tot accounts waarvan de Codex-abonnementscatalogus dit aanbiedt; rechtstreekse OpenAI-
    API-sleutel- en Azure-referenties hiervoor blijven onderdrukt.
    </Warning>

    <Note>
    Nieuwe configuraties moeten de verificatievolgorde voor OpenAI-agents onder `auth.order.openai` plaatsen;
    doctor migreert oudere verouderde vermeldingen voor de Codex-verificatievolgorde.
    </Note>

    ### Configuratievoorbeeld

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
    }
    ```

    Met een API-sleutel als reserve houd je het geselecteerde model onder `openai/*` en plaats je
    de verificatievolgorde onder `openai`. OpenClaw probeert eerst het abonnement en vervolgens
    de API-sleutel, terwijl het de Codex-harness blijft gebruiken:

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
      auth: {
        order: {
          openai: [
            "openai:user@example.com",
            "openai:api-key-backup",
          ],
        },
      },
    }
    ```

    <Note>
    Onboarding importeert geen OAuth-materiaal meer uit `~/.codex`. Meld je aan met
    OAuth via de browser (standaard) of de apparaatcodestroom hierboven; OpenClaw beheert de
    resulterende referenties in de eigen verificatieopslag van de agent.
    </Note>

    ### Codex OAuth-routering controleren en herstellen

    ```bash
    openclaw models status
    openclaw models auth list --provider openai
    openclaw config get agents.defaults.model --json
    openclaw config get models.providers.openai.agentRuntime --json
    ```

    Voeg voor een specifieke agent `--agent <id>` toe:

    ```bash
    openclaw models status --agent <id>
    openclaw models auth list --agent <id> --provider openai
    ```

    Als een oudere configuratie nog verouderde Codex GPT-referenties bevat, of een achterhaalde
    runtime-sessievastlegging voor OpenAI zonder expliciete runtimeconfiguratie, herstel je deze:

    ```bash
    openclaw doctor --fix
    openclaw config validate
    ```

    Als `models auth list --provider openai` geen bruikbaar profiel toont, meld je dan
    opnieuw aan:

    ```bash
    openclaw models auth login --provider openai
    openclaw models status --probe --probe-provider openai
    ```

    Gebruik `--profile-id` voor meerdere Codex OAuth-aanmeldingen binnen dezelfde agent en
    beheer ze vervolgens via de verificatievolgorde of `/model ...@<profileId>`:

    ```bash
    openclaw models auth login --provider openai --profile-id openai:ritsuko
    openclaw models auth login --provider openai --profile-id openai:lain
    ```

    Voer `openclaw doctor --fix` uit om oudere verouderde profiel-ID's en volgordevermeldingen
    met een OpenAI Codex-voorvoegsel te migreren voordat je op de profielvolgorde vertrouwt.

    ### Statusindicator

    Chat `/status` toont welke modelruntime actief is voor de huidige
    sessie. De meegeleverde Codex-app-serverharness wordt weergegeven als
    `Runtime: OpenAI Codex` wanneer deze wordt geselecteerd door een geschikte impliciete route of een expliciet
    runtimebeleid voor provider/model.

    ### Doctor-waarschuwing

    Als verouderde Codex-modelreferenties of achterhaalde OpenAI-runtimevastleggingen in de configuratie
    of sessiestatus achterblijven, herschrijft `openclaw doctor --fix` deze naar `openai/*` met
    de Codex-runtime, tenzij OpenClaw expliciet is geconfigureerd.

    ### Standaardwaarden voor het contextvenster en aanmelding voor lange context

    OpenClaw behandelt de systeemeigen modelcapaciteit en het actieve runtimebudget als
    afzonderlijke waarden:

    - `contextWindow` declareert het totale modelvenster van de provider.
    - `contextTokens` beperkt hoeveel van dat venster OpenClaw voor actieve invoer gebruikt.

    ChatGPT/Codex OAuth volgt de actuele Codex-accountcatalogus. De huidige
    catalogus vermeldt voor GPT-5.6 doorgaans een actief venster van `272000` tokens.
    Rechtstreekse GPT-5.5- en GPT-5.6-modellen met een API-sleutel gebruiken ook standaard
    `272000` `contextTokens`, hoewel de Platform API een groter systeemeigen
    venster aanbiedt. Hierdoor blijven het normale profiel voor latentie, kwaliteit en kosten
    consistent tussen de verificatiemethoden. Een geconfigureerde waarde voor `agents.defaults.contextTokens` kan
    dat budget verder verlagen, maar kan een model niet boven de geconfigureerde
    limiet van `contextTokens` verhogen.

    Voor rechtstreekse GPT-5.5 en GPT-5.6 met een API-sleutel documenteert OpenAI een providervenster
    van `1050000` tokens en maximaal `128000` uitvoertokens. Als de
    volledige uitvoerruimte wordt gereserveerd, blijven `922000` tokens over voor invoer. Dit is een afgeleid
    werkbudget, geen afzonderlijke door de provider gepubliceerde invoerlimiet. Zie de
    officiële [modelvergelijking](https://developers.openai.com/api/docs/models/compare)
    en de [GPT-5.5-modelpagina](https://developers.openai.com/api/docs/models/gpt-5.5).
    In het volgende voorbeeld wordt voor één Terra-model deze ruimte ingeschakeld en
    OpenAI gevraagd om Compaction uit te voeren bij `700000` actieve tokens:

    ```json5
    {
      models: {
        providers: {
          openai: {
            models: [
              {
                id: "gpt-5.6-terra",
                name: "GPT-5.6 Terra",
                contextWindow: 1050000,
                contextTokens: 922000,
                maxTokens: 128000,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-terra" },
          models: {
            "openai/gpt-5.6-terra": {
              agentRuntime: { id: "openclaw" },
              params: {
                responsesServerCompaction: true,
                responsesCompactThreshold: 700000,
              },
            },
          },
        },
      },
    }
    ```

    `agentRuntime.id: "openclaw"` is in dit voorbeeld bewust gekozen. Hiermee wordt aangetoond dat het
    ingebedde OpenClaw Responses-pad de bovenstaande modelmetadata en instellingen voor
    Compaction aan de serverzijde gebruikt. Een systeemeigen Codex-harnessthread beheert het eigen contextbudget
    in plaats daarvan in de Codex-configuratie; zie
    [Lange context voor de Codex-harness](/nl/plugins/codex-harness#direct-api-long-context).

    <Warning>
    OpenAI past hogere tarieven voor lange context toe zodra een GPT-5.5- of GPT-5.6-
    verzoek meer dan `272000` invoertokens bevat: het volledige kwalificerende verzoek wordt
    gefactureerd tegen 2× het invoertarief en 1,5× het uitvoertarief. Grote prompts worden bij volgende
    beurten opnieuw verzonden of gecompacteerd, waardoor een sessie waarvoor dit is ingeschakeld aanzienlijk duurder kan zijn
    dan de standaard, zelfs als het zichtbare antwoord kort is. Zie
    [OpenAI API-tarieven](https://developers.openai.com/api/docs/pricing). De API
    blijft bepalend voor accounttoegang, werkelijke limieten en facturering.
    </Warning>

    ### Catalogusherstel

    OpenClaw gebruikt upstream Codex-catalogusmetadata voor `gpt-5.5` wanneer deze
    aanwezig is. Als de actuele Codex-detectie de rij `gpt-5.5` weglaat terwijl het account
    is geverifieerd, maakt OpenClaw die OAuth-modelrij aan, zodat uitvoeringen via Cron,
    subagents en het geconfigureerde standaardmodel niet mislukken met
    `Unknown model`.

  </Tab>
</Tabs>

## Verificatie voor de systeemeigen Codex-app-server

De systeemeigen Codex-app-serverharness gebruikt `openai/*`-modelreferenties wanneer deze impliciet wordt
geselecteerd door een geschikte exacte officiële HTTPS-route, of wanneer provider/model
`agentRuntime.id: "codex"` deze expliciet selecteert. De verificatie blijft
accountgebaseerd. OpenClaw selecteert verificatie in deze volgorde:

1. Geordende OpenAI-verificatieprofielen voor de agent, bij voorkeur onder
   `auth.order.openai`. Voer `openclaw doctor --fix` uit om oudere verouderde
   Codex-verificatieprofiel-ID's en de verificatievolgorde te migreren.
2. Het bestaande account van de app-server, zoals een lokale ChatGPT-
   aanmelding bij de Codex CLI. Voor de standaard geïsoleerde thuismap van de agent koppelt OpenClaw dat systeemeigen
   CLI-account via de aanmeldings-RPC aan de app-server; het deelt niet de
   configuratie, plugins of threadopslag van de CLI.
3. Alleen voor lokale app-serverstarts via stdio, en uitsluitend wanneer de app-server
   meldt dat er geen account is: `CODEX_API_KEY`, gevolgd door `OPENAI_API_KEY`.

Een lokale aanmelding met een ChatGPT/Codex-abonnement wordt niet vervangen alleen omdat het
Gateway-proces ook `OPENAI_API_KEY` bevat voor rechtstreekse OpenAI-modellen of
embeddings. De terugval op een API-sleutel uit de omgeving geldt uitsluitend voor het lokale stdio-pad
zonder account; deze sleutel wordt nooit via WebSocket-verbindingen met de app-server verzonden. Wanneer een
Codex-profiel in abonnementsstijl is geselecteerd, houdt OpenClaw ook
`CODEX_API_KEY` en `OPENAI_API_KEY` buiten het gestarte stdio-app-serverkindproces
en verzendt het de geselecteerde referenties in plaats daarvan via de aanmeldings-RPC van de app-server.

Wanneer dat abonnementsprofiel wordt geblokkeerd door een Codex-gebruikslimiet, markeert OpenClaw
het profiel als geblokkeerd tot de door Codex vermelde hersteltijd en laat het de verificatievolgorde
doorschakelen naar het volgende `openai:*`-profiel, zonder het geselecteerde
model te wijzigen of de Codex-harness te verlaten. Zodra de hersteltijd is verstreken, komt het
abonnementsprofiel weer in aanmerking.

## Afbeeldingen genereren

De meegeleverde Plugin `openai` registreert het genereren van afbeeldingen via de
tool `image_generate`. Deze ondersteunt zowel het genereren van afbeeldingen met een OpenAI-API-sleutel
als met Codex OAuth via dezelfde modelreferentie `openai/gpt-image-2`.

| Mogelijkheid              | OpenAI-API-sleutel                 | Codex OAuth                          |
| ------------------------- | ---------------------------------- | ------------------------------------ |
| Modelreferentie           | `openai/gpt-image-2`               | `openai/gpt-image-2`                 |
| Verificatie               | `OPENAI_API_KEY`                   | Aanmelding met OpenAI Codex OAuth    |
| Transport                 | OpenAI Images API                  | Codex Responses-backend              |
| Maximumaantal afbeeldingen per verzoek | 4                       | 4                                    |
| Bewerkingsmodus           | Ingeschakeld (maximaal 5 referentieafbeeldingen) | Ingeschakeld (maximaal 5 referentieafbeeldingen) |
| Overschrijvingen voor formaat | Ondersteund, inclusief 2K/4K-formaten | Ondersteund, inclusief 2K/4K-formaten |
| Beeldverhouding/resolutie | Niet doorgestuurd naar OpenAI Images API | Indien veilig toegewezen aan een ondersteund formaat |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-2" },
    },
  },
}
```

<Note>
Zie [Afbeeldingen genereren](/nl/tools/image-generation) voor gedeelde toolparameters,
providerselectie en terugvalgedrag.
</Note>

`gpt-image-2` is de standaard voor het genereren en bewerken van afbeeldingen
op basis van tekst met OpenAI. `gpt-image-1.5`, `gpt-image-1` en `gpt-image-1-mini` blijven bruikbaar
als expliciete modeloverschrijvingen. Gebruik `openai/gpt-image-1.5` voor
PNG/WebP-uitvoer met een transparante achtergrond; de huidige `gpt-image-2`-API weigert
`background: "transparent"`.

Roep voor een verzoek met transparante achtergrond `image_generate` aan met
`model: "openai/gpt-image-1.5"`, `outputFormat: "png"` of `"webp"`, en
`background: "transparent"`; de oudere provideroptie `openai.background` wordt
nog steeds geaccepteerd. OpenClaw beschermt ook de openbare routes voor OpenAI en OpenAI Codex OAuth
door standaard transparante verzoeken voor `openai/gpt-image-2` te herschrijven naar
`gpt-image-1.5`; Azure en aangepaste OpenAI-compatibele eindpunten behouden hun
geconfigureerde implementatie- en modelnamen.

Dezelfde instelling is beschikbaar voor headless CLI-uitvoeringen:

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "Een eenvoudige sticker met een rode cirkel op een transparante achtergrond" \
  --json
```

Gebruik dezelfde vlaggen `--output-format` en `--background` met
`openclaw infer image edit` wanneer je met een invoerbestand begint.
`--openai-background` blijft beschikbaar als een OpenAI-specifieke alias. Gebruik
`--quality low|medium|high|auto` om de kwaliteit en kosten van OpenAI Images te regelen.
Gebruik `--openai-moderation low|auto` om de moderatiehint van OpenAI door te geven vanuit
`image generate` of `image edit`.

Voor ChatGPT/Codex OAuth-installaties behoud je dezelfde `openai/gpt-image-2`-ref. Wanneer
een `openai` OAuth-profiel is geconfigureerd, haalt OpenClaw dat opgeslagen OAuth-
toegangstoken op en verstuurt het afbeeldingsverzoeken via de Codex Responses-backend; het
probeert niet eerst `OPENAI_API_KEY` en valt niet stilzwijgend terug op een API-sleutel.
Configureer `models.providers.openai` expliciet met een API-sleutel, aangepaste basis-
URL of Azure-eindpunt wanneer je in plaats daarvan de directe route via de OpenAI Images API
wilt gebruiken. Als dat aangepaste afbeeldingseindpunt zich op een vertrouwd LAN/privéadres bevindt,
stel je ook `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` in; OpenClaw
houdt privé/interne OpenAI-compatibele afbeeldingseindpunten geblokkeerd tenzij deze
opt-in aanwezig is.

Genereren:

```
/tool image_generate model=openai/gpt-image-2 prompt="Een verzorgde lanceringsposter voor OpenClaw op macOS" size=3840x2160 count=1
```

Een transparante PNG genereren:

```
/tool image_generate model=openai/gpt-image-1.5 prompt="Een eenvoudige sticker met een rode cirkel op een transparante achtergrond" outputFormat=png background=transparent
```

Bewerken:

```
/tool image_generate model=openai/gpt-image-2 prompt="Behoud de vorm van het object en verander het materiaal in doorschijnend glas" image=/path/to/reference.png size=1024x1536
```

## Videogeneratie

De meegeleverde `openai`-plugin registreert videogeneratie via de
tool `video_generate`.

| Mogelijkheid       | Waarde                                                                              |
| ---------------- | ---------------------------------------------------------------------------------- |
| Standaardmodel    | `openai/sora-2`                                                                    |
| Modi            | Tekst-naar-video, afbeelding-naar-video, bewerking van één video                                   |
| Referentie-invoer | 1 afbeelding of 1 video                                                                 |
| Formaatoverschrijvingen   | Ondersteund voor tekst-naar-video en afbeelding-naar-video                                     |
| Beeldverhouding     | Geconverteerd naar het dichtstbijzijnde ondersteunde formaat, niet ongewijzigd doorgestuurd                         |
| Andere overschrijvingen  | `resolution`, `audio`, `watermark` worden niet ondersteund en met een toolwaarschuwing weggelaten |

OpenAI-verzoeken voor afbeelding-naar-video gebruiken `POST /v1/videos` met een
`input_reference`-afbeelding. Bewerkingen van één video gebruiken `POST /v1/videos/edits` met de
geüploade video in het veld `video`.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
Zie [Videogeneratie](/nl/tools/video-generation) voor gedeelde toolparameters,
providerselectie en failovergedrag.

De OpenAI-provider declareert `supportsSize`, maar niet `supportsAspectRatio` of
`supportsResolution`. De gedeelde normalisatielaag van OpenClaw converteert een
aangevraagde `aspectRatio` naar de best overeenkomende OpenAI-`size` voordat het
verzoek de provider bereikt, zodat verzoeken voor beeldverhoudingen doorgaans blijven werken.
`resolution` heeft geen terugvalformaat en wordt weggelaten; dit wordt aan de aanroeper gemeld als
`Ignored unsupported overrides for openai/<model>: resolution=<value>`.
</Note>

## GPT-5-promptbijdrage

OpenClaw voegt een gedeelde GPT-5-promptbijdrage toe voor modellen uit de GPT-5-familie bij
de provider `openai` (inclusief verouderde Codex-refs van vóór de reparatie die worden genormaliseerd
naar `openai/*`). Andere providers die ook model-id's uit de GPT-5-familie aanbieden, zoals
OpenRouter- of opencode-routes, ontvangen deze overlay niet; deze wordt bepaald door
provider-id `openai`, niet alleen door de model-id. Oudere GPT-4.x-modellen
ontvangen deze nooit.

De native Codex-app-serverharnas ontvangt het gedragscontract voor persona/tool-
discipline of de vriendelijke overlay voor interactiestijl niet via
ontwikkelaarsinstructies; native Codex behoudt het door Codex beheerde gedrag voor basis, model en
projectdocumentatie, en OpenClaw schakelt de ingebouwde persoonlijkheid van Codex uit voor
native threads, zodat persoonlijkheidsbestanden in de agentwerkruimte gezaghebbend blijven.
OpenClaw draagt alleen runtimecontext bij aan native Codex-threads: kanaal-
aflevering, dynamische OpenClaw-tools, ACP-delegatie, werkruimtecontext en
OpenClaw Skills. De Heartbeat-begeleidingstekst uit dezelfde bijdrage is de
enige uitzondering: native Codex-Heartbeat-beurten ontvangen deze wel, geïnjecteerd als afzonderlijke
samenwerkingsinstructies in plaats van via de gedeelde hook voor promptbijdragen.

De GPT-5-bijdrage voegt een getagd gedragscontract toe voor persona-
persistentie, uitvoeringsveiligheid, tooldiscipline, uitvoervorm, voltooiings-
controles en verificatie bij overeenkomende door OpenClaw samengestelde prompts. Kanaal-
specifiek antwoord- en stilberichtgedrag blijft in de gedeelde OpenClaw-systeem-
prompt en het beleid voor uitgaande aflevering. De vriendelijke laag voor interactiestijl is
afzonderlijk en configureerbaar.

| Waarde                  | Effect                                      |
| ---------------------- | ------------------------------------------- |
| `"friendly"` (standaard) | De vriendelijke laag voor interactiestijl inschakelen |
| `"on"`                 | Alias voor `"friendly"`                      |
| `"off"`                | Alleen de vriendelijke stijllaag uitschakelen       |

<Tabs>
  <Tab title="Configuratie">
    ```json5
    {
      agents: {
        defaults: {
          promptOverlays: {
            gpt5: { personality: "friendly" },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="CLI">
    ```bash
    openclaw config set agents.defaults.promptOverlays.gpt5.personality off
    ```
  </Tab>
</Tabs>

<Tip>
Waarden zijn tijdens runtime niet hoofdlettergevoelig, dus zowel `"Off"` als `"off"` schakelen de
vriendelijke stijllaag uit.
</Tip>

<Note>
De verouderde `plugins.entries.openai.config.personality` wordt nog steeds gelezen als
compatibiliteitsterugval wanneer de gedeelde instelling
`agents.defaults.promptOverlays.gpt5.personality` niet is ingesteld.
</Note>

## Stem en spraak

<AccordionGroup>
  <Accordion title="Spraaksynthese (TTS)">
    De meegeleverde `openai`-plugin registreert spraaksynthese voor het
    `tts`-oppervlak.

    | Instelling      | Configuratiepad                                            | Standaard                          |
    | ------------- | --------------------------------------------------------- | ----------------------------------- |
    | Model        | `tts.providers.openai.model`                  | `gpt-4o-mini-tts`                |
    | Stem        | `tts.providers.openai.speakerVoice`           | `coral`                          |
    | Snelheid        | `tts.providers.openai.speed`                  | (niet ingesteld)                          |
    | Instructies | `tts.providers.openai.instructions`           | (niet ingesteld, alleen `gpt-4o-mini-tts`)  |
    | Formaat       | `tts.providers.openai.responseFormat`         | `opus` voor spraaknotities, `mp3` voor bestanden |
    | API-sleutel      | `tts.providers.openai.apiKey`                 | Valt terug op `OPENAI_API_KEY`   |
    | Basis-URL     | `tts.providers.openai.baseUrl`                | `https://api.openai.com/v1`      |
    | Extra body   | `tts.providers.openai.extraBody` / `extra_body` | (niet ingesteld)                        |

    Beschikbare modellen: `gpt-4o-mini-tts`, `tts-1`, `tts-1-hd`. Beschikbare stemmen:
    `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `fable`, `juniper`,
    `marin`, `onyx`, `nova`, `sage`, `shimmer`, `verse`.

    `extraBody` wordt na de door OpenClaw
    gegenereerde velden samengevoegd in de JSON van het `/audio/speech`-verzoek; gebruik dit dus voor OpenAI-compatibele eindpunten die
    aanvullende sleutels vereisen, zoals `lang`. Prototypesleutels worden genegeerd.

    ```json5
    {
      tts: {
        providers: {
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "coral" },
        },
      },
    }
    ```

    <Note>
    Stel `OPENAI_TTS_BASE_URL` in om de TTS-basis-URL te overschrijven zonder
    het eindpunt van de chat-API te beïnvloeden. OpenAI TTS en Realtime-spraak worden beide geconfigureerd
    via een API-sleutel van het OpenAI Platform; installaties met alleen OAuth kunnen nog steeds
    door Codex ondersteunde chatmodellen gebruiken, maar geen live terugspraak van OpenAI.
    </Note>

  </Accordion>

  <Accordion title="Spraak-naar-tekst">
    De meegeleverde `openai`-plugin registreert batchgewijze spraak-naar-tekst via
    het transcriptieoppervlak voor mediabegrip van OpenClaw.

    - Standaardmodel: `gpt-4o-transcribe`
    - Eindpunt: OpenAI REST `/v1/audio/transcriptions`
    - Invoerpad: multipart-upload van audiobestand
    - Gebruikt overal waar transcriptie van inkomende audio `tools.media.audio` leest,
      inclusief segmenten van Discord-spraakkanalen en audio-bijlagen van kanalen

    Om OpenAI af te dwingen voor transcriptie van inkomende audio:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "openai",
                model: "gpt-4o-transcribe",
              },
            ],
          },
        },
      },
    }
    ```

    Taal- en promptaanwijzingen worden doorgestuurd naar OpenAI wanneer ze worden geleverd door de
    gedeelde audiomediaconfiguratie of het transcriptieverzoek per aanroep.

  </Accordion>

  <Accordion title="Realtime-transcriptie">
    De meegeleverde `openai`-plugin registreert realtime-transcriptie voor de
    Voice Call-plugin.

    | Instelling          | Configuratiepad                                                          | Standaard |
    | ----------------- | ----------------------------------------------------------------------- | --------- |
    | Model            | `plugins.entries.voice-call.config.streaming.providers.openai.model` | `gpt-4o-transcribe` |
    | Taal         | `...openai.language`                                                 | (niet ingesteld) |
    | Prompt           | `...openai.prompt`                                                   | (niet ingesteld) |
    | Stilteduur | `...openai.silenceDurationMs`                                        | `800`   |
    | VAD-drempel    | `...openai.vadThreshold`                                             | `0.5`   |
    | Authenticatie             | `...openai.apiKey`, `OPENAI_API_KEY` of API-sleutelprofiel `openai`    | Platform-API-sleutel vereist |

    <Note>
    Gebruikt een WebSocket-verbinding met `wss://api.openai.com/v1/realtime` met
    G.711 u-law-audio (`g711_ulaw` / `audio/pcmu`). Voor een API-sleutelprofiel van `openai`
    maakt de Gateway een tijdelijk Realtime-transcriptieclient-
    geheim aan voordat de WebSocket wordt geopend. Deze streamingprovider is bedoeld voor het realtime-transcriptiepad
    van Voice Call; Discord-spraak neemt momenteel korte
    segmenten op en gebruikt in plaats daarvan het batchtranscriptiepad `tools.media.audio`.
    </Note>

  </Accordion>

  <Accordion title="Realtime-spraak">
    De meegeleverde `openai`-plugin registreert realtime-spraak voor de Voice Call-
    plugin.

    | Instelling                             | Configuratiepad                                                             | Standaardwaarde        |
    | --------------------------------------- | ---------------------------------------------------------------------------- | ---------------------- |
    | Model                                  | `plugins.entries.voice-call.config.realtime.providers.openai.model`     | `gpt-realtime-2.1`  |
    | Stem                                   | `...openai.voice`                                                       | `alloy`             |
    | Temperatuur (Azure-implementatiebridge) | `...openai.temperature`                                                 | `0.8`               |
    | VAD-drempel                            | `...openai.vadThreshold`                                                | `0.5`                |
    | Stilteduur                             | `...openai.silenceDurationMs`                                           | `500`                |
    | Voorloopopvulling                      | `...openai.prefixPaddingMs`                                             | `300`                |
    | Redeneerinspanning                     | `...openai.reasoningEffort`                                             | (niet ingesteld)     |
    | Authenticatie                          | `openai` API-sleutelprofiel, `...openai.apiKey` of `OPENAI_API_KEY` | OpenAI Platform-API-sleutel vereist |

    Beschikbare ingebouwde Realtime-stemmen voor `gpt-realtime-2.1`: `alloy`, `ash`,
    `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`.
    OpenAI beveelt `marin` en `cedar` aan voor de beste Realtime-kwaliteit. Dit
    is een andere reeks dan de tekst-naar-spraakstemmen hierboven; een stem die alleen voor TTS is bedoeld,
    zoals `fable`, `nova` of `onyx`, is niet geldig voor Realtime-sessies.
    Stel het model expliciet in op `gpt-realtime-2.1-mini` als je de
    kleinere, goedkopere Realtime 2.1-variant verkiest.

    <Note>
    **GPT-Live (binnenkort beschikbaar).** OpenAI's full-duplexmodellen `gpt-live-1` en
    `gpt-live-1-mini` vervingen de spraakmodus van ChatGPT in juli 2026; de
    ontwikkelaars-API wordt uitgerold naar organisaties met vroege toegang. OpenClaw
    herkent de modelfamilie, maar voert deze nog niet uit: GPT-Live-sessies werken
    uitsluitend via WebRTC, beheren zelf de beurtwisseling (zonder VAD) en delegeren agentwerk
    via een overdrachtsgebeurtenisprotocol dat de Realtime-transporten van OpenClaw
    nog niet implementeren. Het configureren van een `gpt-live-*`-model wordt veilig geweigerd met
    aanwijzingen voor zowel de WebSocket-bridge als Talk-browsersessies, in plaats van
    stilzwijgend audio te verbinden zonder agenttoegang. API-toegang wordt tijdens
    de vroege toegang ook per OpenAI-organisatie beperkt. Behoud `gpt-realtime-2.1` (de
    standaardwaarde) totdat ondersteuning voor GPT-Live beschikbaar is.
    </Note>

    <Note>
    OpenAI Realtime-bridges in de backend gebruiken de GA-sessiestructuur voor Realtime via WebSocket,
    die `session.temperature` niet accepteert. Azure OpenAI-
    implementaties blijven beschikbaar via `azureEndpoint` en `azureDeployment` en
    behouden de implementatiecompatibele sessiestructuur (inclusief `temperature`).
    Ondersteunt bidirectionele toolaanroepen en G.711 u-law-audio.
    </Note>

    <Note>
    De Realtime-stem wordt geselecteerd wanneer de sessie wordt aangemaakt. OpenAI staat toe dat de meeste
    sessievelden later worden gewijzigd, maar de stem kan niet meer worden gewijzigd nadat het
    model in die sessie audio heeft uitgevoerd. OpenClaw stelt momenteel de
    id's van ingebouwde Realtime-stemmen beschikbaar als tekenreeksen.
    </Note>

    <Note>
    Control UI Talk gebruikt OpenAI Realtime-browsersessies met een door de Gateway
    uitgegeven tijdelijk clientgeheim en een rechtstreekse WebRTC SDP-uitwisseling vanuit de browser
    met de OpenAI Realtime-API. De Gateway maakt dat clientgeheim aan met
    de geselecteerde `openai`-referentie. Geconfigureerde sleutels, API-sleutelprofielen en
    `OPENAI_API_KEY` krijgen voorrang; een `openai` OAuth-profiel of externe
    Codex-aanmelding dient als terugvaloptie. De Gateway-relay en Realtime-
    WebSocket-bridges van de Voice Call-backend gebruiken dezelfde volgorde van referenties voor oorspronkelijke OpenAI-eindpunten.
    Liveverificatie voor beheerders is beschikbaar met
    `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts`;
    de OpenAI-trajecten verifiëren zowel de WebSocket-bridge van de backend als de WebRTC
    SDP-uitwisseling van de browser zonder geheimen te loggen.
    Geef `--openai-only` door om deze twee trajecten zonder Google-referenties uit te voeren.
    </Note>

  </Accordion>
</AccordionGroup>

## Azure OpenAI-eindpunten

De meegeleverde provider `openai` kan voor het genereren van afbeeldingen een Azure OpenAI-resource
gebruiken door de basis-URL te overschrijven. In het pad voor het genereren van afbeeldingen detecteert OpenClaw
Azure-hostnamen in `models.providers.openai.baseUrl` en schakelt het automatisch over op
de aanvraagstructuur van Azure.

<Note>
Realtime-spraak gebruikt een afzonderlijk configuratiepad
(`plugins.entries.voice-call.config.realtime.providers.openai.azureEndpoint`)
en wordt niet beïnvloed door `models.providers.openai.baseUrl`. Bekijk het uitklapgedeelte **Realtime-
spraak** onder [Spraak en gesproken tekst](#voice-and-speech) voor de Azure-
instellingen.
</Note>

Gebruik Azure OpenAI wanneer:

- Je al een Azure OpenAI-abonnement, quotum of zakelijke
  overeenkomst hebt
- Je regionale gegevensopslag of door Azure geboden nalevingsmaatregelen nodig hebt
- Je verkeer binnen een bestaande Azure-tenant wilt houden

### Configuratie

Voor het genereren van Azure-afbeeldingen via de meegeleverde provider `openai` stel je
`models.providers.openai.baseUrl` in op je Azure-resource en `apiKey` op
de Azure OpenAI-sleutel (niet een OpenAI Platform-sleutel):

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://<your-resource>.openai.azure.com",
        apiKey: "<azure-openai-api-key>",
      },
    },
  },
}
```

OpenClaw herkent deze Azure-hostachtervoegsels voor de Azure-route voor het genereren van
afbeeldingen:

- `*.openai.azure.com`
- `*.services.ai.azure.com`
- `*.cognitiveservices.azure.com`

Voor aanvragen voor het genereren van afbeeldingen op een herkende Azure-host doet OpenClaw het volgende:

- Verzendt de header `api-key` in plaats van `Authorization: Bearer`
- Gebruikt implementatiespecifieke paden (`/openai/deployments/{deployment}/...`)
- Voegt `?api-version=...` toe aan elke aanvraag
- Gebruikt een standaardtime-out van 600s voor Azure-aanroepen voor het genereren van afbeeldingen.
  Waarden voor `timeoutMs` per aanroep overschrijven deze standaardwaarde nog steeds.

Andere basis-URL's (openbare OpenAI, OpenAI-compatibele proxy's) behouden de standaard
OpenAI-aanvraagstructuur voor afbeeldingen.

<Note>
Azure-routering voor het pad voor het genereren van afbeeldingen van de provider `openai` vereist
OpenClaw 2026.4.22 of nieuwer. Eerdere versies behandelen elke aangepaste
`openai.baseUrl` als het openbare OpenAI-eindpunt en mislukken bij Azure-
afbeeldingsimplementaties.
</Note>

### API-versie

Stel `AZURE_OPENAI_API_VERSION` in om een specifieke Azure-preview- of GA-versie
vast te zetten voor het Azure-pad voor het genereren van afbeeldingen:

```bash
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
```

De standaardwaarde is `2024-12-01-preview` wanneer de variabele niet is ingesteld.

### Modelnamen zijn implementatienamen

Azure OpenAI koppelt modellen aan implementaties. Voor aanvragen voor het genereren van Azure-afbeeldingen
die via de meegeleverde provider `openai` worden gerouteerd, moet het veld `model` in OpenClaw
de **Azure-implementatienaam** zijn die je in de Azure-portal hebt geconfigureerd, niet
de openbare OpenAI-model-id.

Als je een implementatie met de naam `gpt-image-2-prod` maakt die `gpt-image-2` aanbiedt:

```
/tool image_generate model=openai/gpt-image-2-prod prompt="Een strakke poster" size=1024x1024 count=1
```

Dezelfde regel voor implementatienamen geldt voor elke aanroep voor het genereren van afbeeldingen die
via de meegeleverde provider `openai` wordt gerouteerd.

### Regionale beschikbaarheid

Het genereren van Azure-afbeeldingen is momenteel alleen beschikbaar in een deel van de regio's
(bijvoorbeeld `eastus2`, `swedencentral`, `polandcentral`, `westus3`,
`uaenorth`). Controleer de actuele regiolijst van Microsoft voordat je een
implementatie maakt en controleer of het specifieke model in jouw regio wordt aangeboden.

### Parameterverschillen

Azure OpenAI en openbaar OpenAI accepteren niet altijd dezelfde afbeeldingsparameters.
Azure kan opties weigeren die openbaar OpenAI wel toestaat (bijvoorbeeld bepaalde
waarden voor `background` bij `gpt-image-2`) of deze alleen beschikbaar stellen voor specifieke modelversies.
Deze verschillen zijn afkomstig van Azure en het onderliggende model, niet van
OpenClaw. Als een Azure-aanvraag mislukt met een validatiefout, controleer dan in de
Azure-portal de parameterset die door jouw specifieke implementatie en API-versie wordt ondersteund.

<Note>
Azure OpenAI gebruikt oorspronkelijk transport en compatibiliteitsgedrag, maar ontvangt
de verborgen toeschrijvingsheaders van OpenClaw niet — bekijk het uitklapgedeelte **Oorspronkelijke versus OpenAI-compatibele
routes** onder [Geavanceerde configuratie](#advanced-configuration).

Gebruik voor chat- of Responses-verkeer op Azure (naast het genereren van afbeeldingen) de
onboardingflow of een specifieke Azure-providerconfiguratie; alleen `openai.baseUrl`
neemt de Azure-API-/authenticatiestructuur niet over. Er bestaat een afzonderlijke provider
`azure-openai-responses/*`; bekijk het uitklapgedeelte over server-side Compaction
hieronder.
</Note>

## Geavanceerde configuratie

De onderstaande voorbeelden voor `params` per model bepalen de ingesloten provideraanvraag
van OpenClaw. Het configureren ervan is expliciet gedefinieerd aanvraaggedrag, waardoor een anders geschikte
route `auto` bij OpenClaw blijft in plaats van Codex impliciet te selecteren. De oorspronkelijke
Codex-app-serverharness beheert zijn eigen transport- en aanvraaginstellingen; expliciete
`agentRuntime.id: "codex"` wordt veilig geweigerd wanneer de effectieve route niet als
Codex-compatibel is gedeclareerd.

<AccordionGroup>
  <Accordion title="Transport (WebSocket versus SSE)">
    OpenClaw gebruikt voor `openai/*` eerst WebSocket, met SSE als terugvaloptie (`"auto"`).

    In de modus `"auto"` doet OpenClaw het volgende:
    - Probeert één vroege WebSocket-fout opnieuw voordat op SSE wordt teruggevallen
    - Markeert WebSocket na een fout gedurende 60 seconden als verslechterd en gebruikt SSE
      tijdens de afkoelperiode
    - Voegt stabiele headers voor sessie- en beurtidentiteit toe bij nieuwe pogingen en
      nieuwe verbindingen
    - Normaliseert gebruikstellers (`input_tokens` / `prompt_tokens`) voor alle
      transportvarianten

    | Waarde                | Gedrag                          |
    | ---------------------- | ------------------------------------ |
    | `"auto"` (standaard)   | Eerst WebSocket, SSE als terugvaloptie |
    | `"sse"`              | Alleen SSE afdwingen             |
    | `"websocket"`        | Alleen WebSocket afdwingen       |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    Gerelateerde OpenAI-documentatie:
    - [Realtime-API met WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
    - [API-antwoorden streamen (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="Snelle modus">
    OpenClaw biedt een gedeelde schakeloptie voor de snelle modus van `openai/*`:

    - **Chat/UI:** `/fast status|auto|on|off`
    - **Configuratie:** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    Wanneer deze is ingeschakeld, koppelt OpenClaw de snelle modus aan prioriteitsverwerking van OpenAI
    (`service_tier = "priority"`). Bestaande waarden voor `service_tier` blijven
    behouden en de snelle modus herschrijft `reasoning` of
    `text.verbosity` niet. `fastMode: "auto"` start nieuwe modelaanroepen in de snelle modus tot de
    automatische afkapgrens en start latere nieuwe pogingen, terugval-, toolresultaat- of
    vervolgaanroepen daarna zonder snelle modus. De afkapgrens is standaard 60 seconden;
    stel `params.fastAutoOnSeconds` in op het actieve model om deze te wijzigen.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { fastMode: "auto", fastAutoOnSeconds: 30 } },
          },
        },
      },
    }
    ```

    <Note>
    Sessieoverschrijvingen krijgen voorrang op de configuratie. Als je de sessieoverschrijving in de
    Sessions UI wist, keert de sessie terug naar de geconfigureerde standaardwaarde.
    </Note>

  </Accordion>

  <Accordion title="Prioriteitsverwerking (service_tier)">
    De API van OpenAI biedt prioriteitsverwerking via `service_tier`. Stel dit per
    model in OpenClaw in:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    Ondersteunde waarden: `auto`, `default`, `flex`, `priority`.

    <Warning>
    `serviceTier` wordt alleen doorgestuurd naar systeemeigen OpenAI-eindpunten
    (`api.openai.com`) en systeemeigen Codex-eindpunten (`chatgpt.com/backend-api`).
    Als je een van beide providers via een proxy routeert, laat OpenClaw
    `service_tier` ongewijzigd.
    </Warning>

  </Accordion>

  <Accordion title="Compaction aan de serverzijde (Responses API)">
    Voor directe OpenAI Responses-modellen (`openai/*` op `api.openai.com`) schakelt
    de OpenClaw-streamwrapper van de OpenAI-plugin automatisch Compaction aan de
    serverzijde in:

    - Dwingt `store: true` af (tenzij modelcompatibiliteit `supportsStore: false` instelt)
    - Injecteert `context_management: [{ type: "compaction", compact_threshold: ... }]`
    - Standaardwaarde voor `compact_threshold`: 70% van `contextWindow` (of `80000` wanneer
      niet beschikbaar)

    Dit is van toepassing op het ingebouwde runtimepad van OpenClaw en op
    OpenAI-providerhooks die door ingesloten uitvoeringen worden gebruikt. De
    systeemeigen Codex-app-serverharness beheert zijn eigen context via Codex
    en wordt niet door deze instelling beïnvloed.

    <Tabs>
      <Tab title="Expliciet inschakelen">
        Nuttig voor compatibele eindpunten zoals Azure OpenAI Responses:

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.5": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Aangepaste drempelwaarde">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: {
                    responsesServerCompaction: true,
                    responsesCompactThreshold: 120000,
                  },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="Uitschakelen">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` regelt alleen de injectie van `context_management`.
    Directe OpenAI Responses-modellen dwingen nog steeds `store: true` af, tenzij
    compatibiliteit `supportsStore: false` instelt.
    </Note>

  </Accordion>

  <Accordion title="Strikte agentische GPT-modus">
    Voor GPT-5-familiemodellen van de provider `openai` die via de ingesloten
    runtime van OpenClaw worden uitgevoerd, gebruikt OpenClaw standaard al een
    strikter uitvoeringscontract met de naam `strict-agentic`. Het wordt
    automatisch geactiveerd wanneer de opgeloste provider `openai` is
    en de model-id overeenkomt met de GPT-5-familie, tenzij de configuratie dit
    expliciet weer uitschakelt:

    ```json5
    {
      agents: {
        defaults: {
          embeddedAgent: { executionContract: "default" },
        },
      },
    }
    ```

    Het expliciet instellen van `"strict-agentic"` heeft geen effect op een ondersteund pad
    (het is al de standaardwaarde) en blijft inactief bij niet-ondersteunde
    provider-modelparen.

    Wanneer `strict-agentic` actief is, doet OpenClaw het volgende:
    - Schakelt `update_plan` automatisch in voor substantieel werk
    - Probeert structureel lege beurten of beurten met alleen redenering opnieuw met een
      voortzetting die een zichtbaar antwoord oplevert
    - Gebruikt expliciete plangebeurtenissen van de harness wanneer de geselecteerde harness
      deze biedt

    OpenClaw classificeert assistentproza niet om te bepalen of een beurt een
    plan, voortgangsupdate of definitief antwoord is.

    <Note>
    Dit contract bevindt zich volledig in de ingesloten agentrunner van
    OpenClaw. Het is niet van toepassing op de systeemeigen Codex-app-serverharness,
    die zijn eigen beurt- en plangedrag beheert; voor systeemeigen
    Codex-uitvoeringen is de harnessselectie belangrijker dan de instelling van
    het uitvoeringscontract.
    </Note>

  </Accordion>

  <Accordion title="Systeemeigen versus OpenAI-compatibele routes">
    OpenClaw behandelt directe OpenAI-, Codex- en Azure OpenAI-eindpunten
    anders dan generieke OpenAI-compatibele `/v1`-proxy's:

    **Systeemeigen routes** (`openai/*`, Azure OpenAI):
    - Behouden `reasoning: { effort: "none" }` alleen voor modellen die de OpenAI-inspanning
      `none` ondersteunen
    - Laten uitgeschakelde redenering weg voor modellen of proxy's die
      `reasoning.effort: "none"` weigeren
    - Gebruiken standaard de strikte modus voor toolschema's
    - Voegen verborgen attributieheaders alleen toe op geverifieerde systeemeigen hosts (Azure
      OpenAI krijgt deze headers niet, ook al is dit een systeemeigen route)
    - Behouden OpenAI-specifieke aanvraagvormgeving (`service_tier`, `store`,
      redeneringscompatibiliteit, hints voor de promptcache)

    **Proxy-/compatibele routes:**
    - Gebruiken soepeler compatibiliteitsgedrag
    - Verwijderen Completions-`store` uit niet-systeemeigen `openai-completions`-payloads
    - Accepteren geavanceerde `params.extra_body`/`params.extraBody`-doorgifte-JSON
      voor OpenAI-compatibele Completions-proxy's
    - Accepteren `params.chat_template_kwargs` voor OpenAI-compatibele Completions-
      proxy's zoals vLLM
    - Dwingen geen strikte toolschema's of uitsluitend systeemeigen headers af

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Gedeelde parameters voor afbeeldingstools en providerselectie.
  </Card>
  <Card title="Video's genereren" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor videotools en providerselectie.
  </Card>
  <Card title="OAuth en authenticatie" href="/nl/gateway/authentication" icon="key">
    Authenticatiedetails en regels voor het hergebruik van aanmeldgegevens.
  </Card>
</CardGroup>
