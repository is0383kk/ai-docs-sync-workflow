---
read_when:
    - Je wilt één API-sleutel voor veel LLM's
    - Je wilt modellen via OpenRouter in OpenClaw uitvoeren
    - Je wilt OpenRouter gebruiken voor het genereren van afbeeldingen
    - Je wilt OpenRouter gebruiken voor het genereren van muziek
    - Je wilt OpenRouter gebruiken voor het genereren van video's
summary: Gebruik de uniforme API van OpenRouter voor toegang tot veel modellen in OpenClaw
title: OpenRouter
x-i18n:
    generated_at: "2026-07-27T05:14:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter routeert aanvragen naar veel modellen achter één API en één sleutel. Het is
compatibel met OpenAI, zodat OpenClaw ermee communiceert via hetzelfde
`openai-completions`-achtige transport dat voor andere proxyproviders wordt gebruikt.

## Aan de slag

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="OAuth-onboarding uitvoeren">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw opent de browseraanmeldingsflow van OpenRouter (PKCE), wisselt de
        code in voor een OpenRouter-API-sleutel en slaat deze op in het standaard
        OpenRouter-authenticatieprofiel. Op externe/headless hosts toont OpenClaw de
        aanmeldings-URL en vraagt het je om na het aanmelden de omleidings-URL te plakken.
      </Step>
      <Step title="(Optioneel) Overschakelen naar een specifiek model">
        Onboarding gebruikt standaard `openrouter/auto`. Kies later een concreet model:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="API-sleutel">
    <Steps>
      <Step title="Je API-sleutel verkrijgen">
        Maak een API-sleutel aan op [openrouter.ai/keys](https://openrouter.ai/keys).
      </Step>
      <Step title="Onboarding met API-sleutel uitvoeren">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="(Optioneel) Overschakelen naar een specifiek model">
        Onboarding gebruikt standaard `openrouter/auto`. Kies later een concreet model:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Configuratievoorbeeld

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## Modelverwijzingen

<Note>
Modelverwijzingen volgen het patroon `openrouter/<provider>/<model>`. Zie
[/concepten/modelproviders](/nl/concepts/model-providers) voor de volledige lijst met
beschikbare providers en modellen.
</Note>

Meegeleverde fallbackmodellen, gebruikt wanneer live catalogusdetectie niet beschikbaar is:

| Modelverwijzing                   | Opmerkingen                       |
| --------------------------------- | --------------------------------- |
| `openrouter/auto`                 | Automatische routering van OpenRouter |
| `openrouter/moonshotai/kimi-k2.6` | Kimi K2.6 via MoonshotAI           |
| `openrouter/moonshotai/kimi-k2.5` | Kimi K2.5 via MoonshotAI           |

Elke andere `openrouter/<provider>/<model>`-verwijzing, waaronder
`openrouter/openrouter/fusion` (zie [Fusion-router](#fusion-router)), wordt
dynamisch omgezet aan de hand van de live modelcatalogus van OpenRouter.

## Afbeeldingen genereren

OpenRouter kan de `image_generate`-tool ondersteunen. Stel een OpenRouter-afbeeldingsmodel
in onder `agents.defaults.mediaModels.image`:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw stuurt afbeeldingsaanvragen naar de chat-completions-afbeeldings-API van OpenRouter met
`modalities: ["image", "text"]`. Gemini-afbeeldingsmodellen ontvangen daarnaast
`aspectRatio`- en `resolution`-hints via `image_config` van OpenRouter; andere
afbeeldingsmodellen niet. Gebruik `agents.defaults.mediaModels.image.timeoutMs` voor
langzamere modellen; de `timeoutMs` per aanroep van de `image_generate`-tool heeft nog steeds voorrang.

## Video's genereren

OpenRouter kan de `video_generate`-tool ondersteunen via zijn asynchrone
`/videos`-API. Stel een OpenRouter-videomodel in onder
`agents.defaults.mediaModels.video`:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw dient tekst-naar-video- en afbeelding-naar-video-taken in, controleert periodiek de geretourneerde
`polling_url` en downloadt de voltooide video via `unsigned_urls` van OpenRouter
of het inhoudseindpunt van de taak. Referentieafbeeldingen worden standaard gebruikt als
afbeeldingen voor het eerste/laatste frame; afbeeldingen met de tag `reference_image` worden in plaats daarvan als
invoerreferenties verzonden. De meegeleverde standaardwaarde `google/veo-3.1-fast` ondersteunt duurwaarden van 4/6/8
seconden, resoluties van `720P`/`1080P` en beeldverhoudingen van `16:9`/`9:16`.
Video-naar-video wordt niet ondersteund: de upstream-API accepteert alleen tekst- en afbeeldingsreferenties.

## Muziek genereren

OpenRouter kan de `music_generate`-tool ondersteunen via audio-uitvoer van
chat-completions. Stel een OpenRouter-audiomodel in onder
`agents.defaults.mediaModels.music`:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

De meegeleverde OpenRouter-muziekprovider gebruikt standaard `google/lyria-3-pro-preview`
en stelt ook `google/lyria-3-clip-preview` beschikbaar. OpenClaw stuurt `modalities:
["text", "audio"]`, streamt het antwoord, verzamelt de audiofragmenten en slaat
het resultaat op als gegenereerde media voor levering via kanalen. Lyria-modellen accepteren één
referentieafbeelding via de gedeelde parameter `music_generate image=...`.
Streamingaudio, transcriptbewaring en de afgeleide SSE-eventenvelop worden
begrensd door `agents.defaults.mediaMaxMb` (de standaard audiolimiet is 16 MB).

## Tekst-naar-spraak

OpenRouter kan als TTS-provider fungeren via het OpenAI-compatibele
`/audio/speech`-endpoint.

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

Als `tts.providers.openrouter.apiKey` wordt weggelaten, valt TTS terug op
`models.providers.openrouter.apiKey` en vervolgens op `OPENROUTER_API_KEY`.

## Spraak-naar-tekst (inkomende audio)

OpenRouter kan inkomende spraak-/audiobijlagen transcriberen via het gedeelde
`tools.media.audio`-pad, met behulp van het STT-endpoint (`/audio/transcriptions`).
Dit geldt voor elke kanaalplugin die inkomende spraak/audio doorstuurt naar de
preflight voor mediabegrip.

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw verzendt OpenRouter STT-verzoeken als JSON met base64-audio onder
`input_audio` (het STT-contract van OpenRouter), niet als multipart
OpenAI-formulieruploads.

## Fusion-router

OpenRouter Fusion verzendt één OpenClaw-modelreferentie parallel naar meerdere
OpenRouter-modellen, laat OpenRouter hun antwoorden beoordelen en retourneert
één definitief antwoord via het normale OpenRouter-endpoint. De upstream-modelslug is
`openrouter/fusion`, waardoor de OpenClaw-modelreferentie zowel het
OpenClaw-providerprefix als de upstream OpenRouter-namespace bevat:

```bash
openclaw models set openrouter/openrouter/fusion
```

Configureer het panel en het beoordelingsmodel van Fusion via `params.extraBody`
van het model; deze velden worden rechtstreeks doorgestuurd naar de hoofdtekst
van het OpenRouter-verzoek voor chatvoltooiingen. Fusion werkt met onboarding
via OAuth of een API-sleutel; laat bij gebruik van OAuth de regel
`env.OPENROUTER_API_KEY` hieronder weg.

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` is het parallelle panel; `model` in de
Fusion-pluginconfiguratie is het beoordelingsmodel. Stel tijdens normale
agent-/chatbeurten `tool_choice` op het hoogste niveau niet in op
`"required"` om Fusion te proberen af te dwingen: OpenClaw-beurten kunnen
eigen tooldefinities bevatten en een verplichte toolkeuze op het hoogste niveau
kan een daarvan kiezen in plaats van de Fusion-router. Wanneer deze
Fusion-pluginconfiguratie aanwezig is, voegt OpenClaw een opgeschoonde
systeempromptnotitie toe met een lijst van de geconfigureerde analysemodellen
en het beoordelingsmodel, zodat de agent vragen over het eigen Fusion-panel kan
beantwoorden. Andere `extraBody`-velden worden niet naar de prompt
gekopieerd.

Fusion is bewust trager: OpenRouter stuurt de prompt naar meerdere
analysemodellen en voert vervolgens een beoordelings-/synthesestap uit, waardoor
de latentie hoger is dan bij een directe aanvraag aan één model. Gebruik het
voor weloverwogen antwoorden van hoge kwaliteit of escalatiepaden, niet als
latentiegevoelige standaardinstelling. Houd het panel klein en kies snellere
analyse- en beoordelingsmodellen voor snellere antwoorden.

Test een geconfigureerde referentie met een eenmalige lokale aanroep:

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "Antwoord exact met: FUSION_OK" \
  --json
```

## Authenticatie en headers

OpenRouter gebruikt een Bearer-token uit je API-sleutel. OpenRouter OAuth is een PKCE-
aanmeldingsflow die een OpenRouter API-sleutel verstrekt, zodat OpenClaw het resultaat opslaat in
hetzelfde `openrouter:default`-authenticatieprofiel voor API-sleutels dat wordt gebruikt bij handmatige
configuratie van een API-sleutel.

Om je aan te melden of de opgeslagen sleutel in een bestaande installatie te vervangen zonder
de volledige onboarding opnieuw uit te voeren:

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

Bij geverifieerde OpenRouter-verzoeken (`https://openrouter.ai/api/v1`) voegt OpenClaw
de gedocumenteerde headers voor app-toeschrijving van OpenRouter toe:

| Header                    | Waarde                                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`            | `https://openclaw.ai`                                                                                  |
| `X-OpenRouter-Title`      | `OpenClaw`                                                                                             |
| `X-OpenRouter-Categories` | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent` |

<Warning>
Als je de OpenRouter-provider naar een andere proxy of basis-URL verwijst, injecteert OpenClaw
die OpenRouter-specifieke headers of Anthropic-cachemarkeringen **niet**.
</Warning>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Antwoordcaching">
    OpenRouter-antwoordcaching is opt-in. Schakel dit per model in:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw verzendt `X-OpenRouter-Cache: true` en, indien geconfigureerd,
    `X-OpenRouter-Cache-TTL`. `responseCacheClear: true` dwingt vernieuwing af voor
    het huidige verzoek en slaat het vervangende antwoord op. Snake_case-
    aliassen (`response_cache`, `response_cache_ttl_seconds`,
    `response_cache_clear`) worden geaccepteerd, evenals `responseCacheTtl` /
    `response_cache_ttl` zonder het achtervoegsel `Seconds`.

    Dit staat los van promptcaching door de provider en van de Anthropic-
    markeringen `cache_control` van OpenRouter. Het is alleen van toepassing op geverifieerde
    `openrouter.ai`-routes, niet op aangepaste basis-URL's van proxy's.

  </Accordion>

  <Accordion title="Anthropic-cachemarkeringen">
    Op geverifieerde OpenRouter-routes behouden Anthropic-modelreferenties de Anthropic-
    markeringen `cache_control` van OpenRouter voor beter hergebruik van de promptcache voor
    promptblokken van het systeem/de ontwikkelaar.
  </Accordion>

  <Accordion title="Voorinvulling van Anthropic-redenering">
    Op geverifieerde OpenRouter-routes verwijderen Anthropic-modelreferenties waarvoor redenering is ingeschakeld
    afsluitende vooraf ingevulde beurten van de assistent voordat het verzoek
    OpenRouter bereikt, in overeenstemming met de vereiste van Anthropic dat gesprekken met redenering
    eindigen met een beurt van de gebruiker.
  </Accordion>

  <Accordion title="Injectie van denken/redeneren">
    Op ondersteunde niet-`auto`-routes zet OpenClaw het geselecteerde denkniveau
    om in redeneringspayloads voor de OpenRouter-proxy. `openrouter/auto` en niet-ondersteunde
    modelhints slaan die injectie over. Verouderde `openrouter/hunter-alpha`-refs
    slaan deze ook over, omdat OpenRouter op die uitgefaseerde route definitieve antwoordtekst
    in redeneringsvelden kon retourneren.
  </Accordion>

  <Accordion title="Herhaling van DeepSeek V4-redenering">
    Op geverifieerde OpenRouter-routes vullen `openrouter/deepseek/deepseek-v4-flash` en
    `openrouter/deepseek/deepseek-v4-pro` ontbrekende `reasoning_content` in bij
    herhaalde assistentbeurten, zodat gesprekken met denk- en toolstappen de
    vereiste vervolgstructuur van DeepSeek V4 behouden. OpenClaw verzendt voor deze routes
    door OpenRouter ondersteunde `reasoning.effort`-waarden: `xhigh`/`max` worden omgezet in `xhigh`,
    elk ander niveau dat niet uitgeschakeld is, wordt omgezet in `high`.
  </Accordion>

  <Accordion title="Alleen-OpenAI-vormgeving van aanvragen">
    OpenRouter loopt via het proxyachtige, met OpenAI compatibele pad, waardoor systeemeigen
    alleen-OpenAI-vormgeving van aanvragen, zoals `serviceTier`, Responses `store`,
    OpenAI-payloads voor compatibiliteit met redenering en hints voor de promptcache niet wordt doorgestuurd.
  </Accordion>

  <Accordion title="Door Gemini ondersteunde routes">
    Door Gemini ondersteunde OpenRouter-refs blijven op het proxy-Gemini-pad: OpenClaw behoudt
    daar de opschoning van Gemini-denkhandtekeningen, maar schakelt geen systeemeigen
    Gemini-validatie voor herhaling of bootstrap-herschrijvingen in.
  </Accordion>

  <Accordion title="Metagegevens voor providerroutering">
    OpenRouter ondersteunt een `provider`-aanvraagobject voor de routering van de onderliggende
    provider. Configureer met `models.providers.openrouter.params.provider` een standaardbeleid voor alle
    aanvragen aan OpenRouter-tekstmodellen:

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw stuurt dat object door naar OpenRouter als de `provider`-payload
    van de aanvraag. Gebruik de gedocumenteerde snake_case-velden van OpenRouter, waaronder `sort`,
    `only`, `ignore`, `order`, `allow_fallbacks`, `require_parameters`,
    `data_collection`, `quantizations`, `max_price`, `preferred_max_latency`,
    `preferred_min_throughput`, `zdr` en `enforce_distillable_text`.

    Parameters per model overschrijven het providerbrede routeringsobject:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    Dit is alleen van toepassing op chat-completions-routes van OpenRouter. Directe routes van Anthropic,
    Google, OpenAI of aangepaste providers negeren de routeringsparameters van OpenRouter.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelrefs en failovergedrag kiezen.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledige configuratiereferentie voor agents, modellen en providers.
  </Card>
</CardGroup>
