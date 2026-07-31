---
read_when:
    - Je wilt MiniMax-modellen in OpenClaw
    - Je hebt hulp nodig bij het instellen van MiniMax
summary: MiniMax-modellen gebruiken in OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-07-27T05:47:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d3e95cf9836fd0bc30ac91649422a1d0ed8e7b2908a42e241106c1ea783cbbc
    source_path: providers/minimax.md
    workflow: 16
---

De gebundelde `minimax`-Plugin registreert twee providers plus vijf mogelijkheden: chat, beeldgeneratie, muziekgeneratie, videogeneratie, beeldbegrip, spraak (T2A v2) en zoeken op het web.

| Provider-ID      | Authenticatie    | Mogelijkheden                                                                                        |
| ---------------- | ------- | --------------------------------------------------------------------------------------------------- |
| `minimax`        | API-sleutel | Tekst, beeldgeneratie, muziekgeneratie, videogeneratie, beeldbegrip, spraak, zoeken op het web |
| `minimax-portal` | OAuth   | Tekst, beeldgeneratie, muziekgeneratie, videogeneratie, beeldbegrip, spraak             |

<Tip>
Verwijzingslink voor MiniMax Coding Plan (10% korting): [MiniMax Coding Plan](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
</Tip>

## Ingebouwde catalogus

| Model                    | Type             | Beschrijving                              |
| ------------------------ | ---------------- | ---------------------------------------- |
| `MiniMax-M3`             | Chat (redeneren) | Standaard gehost redeneermodel           |
| `MiniMax-M2.7`           | Chat (redeneren) | Vorig gehost redeneermodel          |
| `MiniMax-M2.7-highspeed` | Chat (redeneren) | Snellere M2.7-redeneerlaag               |
| `MiniMax-VL-01`          | Visie           | Model voor beeldbegrip                |
| `image-01`               | Beeldgeneratie | Bewerking van tekst naar beeld en van beeld naar beeld |
| `music-2.6`              | Muziekgeneratie | Standaard muziekmodel                      |
| `MiniMax-Hailuo-2.3`     | Videogeneratie | Flows van tekst naar video en van beeld naar video   |

Modelreferenties volgen het authenticatiepad: `minimax/<model>` voor configuraties met een API-sleutel, `minimax-portal/<model>` voor OAuth-configuraties.

## Aan de slag

<Tabs>
  <Tab title="OAuth (Coding Plan)">
    **Het meest geschikt voor:** snelle configuratie met MiniMax Coding Plan via OAuth, geen API-sleutel vereist.

    <Tabs>
      <Tab title="Internationaal">
        <Steps>
          <Step title="Onboarding uitvoeren">
            ```bash
            openclaw onboard --auth-choice minimax-global-oauth
            ```

            Resulterende basis-URL van de provider: `api.minimax.io`.
          </Step>
          <Step title="Controleren of het model beschikbaar is">
            ```bash
            openclaw models list --provider minimax-portal
            ```
          </Step>
        </Steps>
      </Tab>
      <Tab title="China">
        <Steps>
          <Step title="Onboarding uitvoeren">
            ```bash
            openclaw onboard --auth-choice minimax-cn-oauth
            ```

            Resulterende basis-URL van de provider: `api.minimaxi.com`.
          </Step>
          <Step title="Controleren of het model beschikbaar is">
            ```bash
            openclaw models list --provider minimax-portal
            ```
          </Step>
        </Steps>
      </Tab>
    </Tabs>

    <Note>
    OAuth-configuraties gebruiken provider-ID `minimax-portal`. Modelreferenties hebben de vorm `minimax-portal/MiniMax-M3`.
    </Note>

  </Tab>

  <Tab title="API-sleutel">
    **Het meest geschikt voor:** gehoste MiniMax met een Anthropic-compatibele API.

    <Tabs>
      <Tab title="Internationaal">
        <Steps>
          <Step title="Onboarding uitvoeren">
            ```bash
            openclaw onboard --auth-choice minimax-global-api
            ```

            Hiermee wordt `api.minimax.io` als basis-URL geconfigureerd.
          </Step>
          <Step title="Controleren of het model beschikbaar is">
            ```bash
            openclaw models list --provider minimax
            ```
          </Step>
        </Steps>
      </Tab>
      <Tab title="China">
        <Steps>
          <Step title="Onboarding uitvoeren">
            ```bash
            openclaw onboard --auth-choice minimax-cn-api
            ```

            Hiermee wordt `api.minimaxi.com` als basis-URL geconfigureerd.
          </Step>
          <Step title="Controleren of het model beschikbaar is">
            ```bash
            openclaw models list --provider minimax
            ```
          </Step>
        </Steps>
      </Tab>
    </Tabs>

    ### Configuratievoorbeeld

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "minimax/MiniMax-M3" } } },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
              {
                id: "MiniMax-M2.7",
                name: "MiniMax M2.7",
                reasoning: true,
                input: ["text"],
                cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
                contextWindow: 204800,
                maxTokens: 131072,
              },
              {
                id: "MiniMax-M2.7-highspeed",
                name: "MiniMax M2.7 Highspeed",
                reasoning: true,
                input: ["text"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
                contextWindow: 204800,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    Het Anthropic-compatibele streaming-eindpunt van MiniMax-M2.x verzendt `reasoning_content` in deltasegmenten in OpenAI-stijl in plaats van systeemeigen denkblokken van Anthropic. Hierdoor lekt interne redenering naar de zichtbare uitvoer als denken impliciet ingeschakeld blijft. OpenClaw schakelt denken voor M2.x standaard uit, tenzij je `thinking` zelf expliciet instelt. MiniMax-M3 (en voorwaarts compatibele M3.x) is uitgezonderd: M3 verzendt correcte denkblokken van Anthropic en vereist dat denken actief is om zichtbare inhoud te produceren. Daarom houdt OpenClaw M3 op het adaptieve denkpad van de provider. Zie het gedeelte Denkstandaarden onder Geavanceerde configuratie hieronder.
    </Warning>

    <Note>
    Configuraties met een API-sleutel gebruiken provider-ID `minimax`. Modelreferenties hebben de vorm `minimax/MiniMax-M3`.
    </Note>

  </Tab>
</Tabs>

## Configureren via `openclaw configure`

<Steps>
  <Step title="De wizard starten">
    ```bash
    openclaw configure
    ```
  </Step>
  <Step title="Model/auth selecteren">
    Kies **Model/auth** in het menu.
  </Step>
  <Step title="Een MiniMax-authenticatieoptie kiezen">
    | Authenticatiekeuze            | Beschrijving                        |
    | ----------------------- | ----------------------------------- |
    | `minimax-global-oauth` | Internationale OAuth (Coding Plan)  |
    | `minimax-cn-oauth`     | OAuth voor China (Coding Plan)          |
    | `minimax-global-api`   | Internationale API-sleutel              |
    | `minimax-cn-api`       | API-sleutel voor China                      |
  </Step>
  <Step title="Je standaardmodel kiezen">
    Selecteer je standaardmodel wanneer daarom wordt gevraagd.
  </Step>
</Steps>

## Mogelijkheden

### Beeldgeneratie

De MiniMax-Plugin registreert het model `image-01` voor de tool `image_generate` op zowel `minimax` als `minimax-portal`, waarbij dezelfde `MINIMAX_API_KEY`- of OAuth-authenticatie als voor de tekstmodellen wordt hergebruikt.

- Generatie van tekst naar beeld en bewerking van beeld naar beeld (onderwerpreferentie), beide met regeling van de beeldverhouding
- Maximaal 9 uitvoerafbeeldingen per aanvraag, 1 referentieafbeelding per bewerkingsaanvraag
- Ondersteunde beeldverhoudingen: `1:1`, `16:9`, `4:3`, `3:2`, `2:3`, `3:4`, `9:16`, `21:9`

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

Beeldgeneratie gebruikt altijd het speciale beeldeindpunt van MiniMax (`/v1/image_generation`) en negeert `models.providers.minimax.baseUrl`, omdat dat veld in plaats daarvan de Anthropic-compatibele basis-URL voor chat configureert. Stel `MINIMAX_API_HOST=https://api.minimaxi.com` in om beeldgeneratie via het Chinese eindpunt te routeren; het standaard globale eindpunt is `https://api.minimax.io`.

<Note>
Zie [Beeldgeneratie](/nl/tools/image-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

### Tekst-naar-spraak

De gebundelde `minimax`-Plugin registreert MiniMax T2A v2 als spraakprovider voor `tts`.

- Standaard TTS-model: `speech-2.8-hd`
- Standaardstem: `English_expressive_narrator`
- Gebundelde model-ID's: `speech-2.8-hd`, `speech-2.8-turbo`, `speech-2.6-hd`, `speech-2.6-turbo`, `speech-02-hd`, `speech-02-turbo`, `speech-01-hd`, `speech-01-turbo`
- Volgorde voor het vaststellen van authenticatie: `tts.providers.minimax.apiKey`, vervolgens OAuth-/tokenauthenticatieprofielen voor `minimax-portal`, daarna Token Plan-omgevingssleutels (`MINIMAX_OAUTH_TOKEN`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`) en ten slotte `MINIMAX_API_KEY`
- Als er geen TTS-host is geconfigureerd, hergebruikt OpenClaw de geconfigureerde OAuth-host van `minimax-portal` en verwijdert het Anthropic-compatibele padachtervoegsels zoals `/anthropic`
- Normale audiobijlagen blijven MP3. Doelen voor spraakberichten (Feishu, Telegram en andere kanalen die om een bijlage vragen die compatibel is met spraakberichten) worden met `ffmpeg` getranscodeerd van MiniMax MP3 naar 48kHz Opus, omdat bijvoorbeeld de Feishu/Lark-bestands-API voor systeemeigen audioberichten alleen `file_type: "opus"` accepteert
- MiniMax T2A accepteert fractionele `speed` en `vol`, maar `pitch` wordt als een geheel getal verzonden; OpenClaw kapt fractionele waarden voor `pitch` af vóór de API-aanvraag

| Instelling                         | Omgevingsvariabele                | Standaard                       | Beschrijving                      |
| ------------------------------- | ---------------------- | ----------------------------- | -------------------------------- |
| `tts.providers.minimax.baseUrl` | `MINIMAX_API_HOST`     | `https://api.minimax.io`      | MiniMax T2A API-host.            |
| `tts.providers.minimax.model`   | `MINIMAX_TTS_MODEL`    | `speech-2.8-hd`               | TTS-model-ID.                    |
| `tts.providers.minimax.voiceId` | `MINIMAX_TTS_VOICE_ID` | `English_expressive_narrator` | Stem-ID gebruikt voor spraakuitvoer. |
| `tts.providers.minimax.speed`   |                        | `1.0`                         | Afspeelsnelheid, `0.5..2.0`.      |
| `tts.providers.minimax.vol`     |                        | `1.0`                         | Volume, `(0, 10]`.               |
| `tts.providers.minimax.pitch`   |                        | `0`                           | Gehele toonhoogteverschuiving, `-12..12`.  |

### Muziekgeneratie

De gebundelde MiniMax-Plugin registreert muziekgeneratie via de gedeelde tool `music_generate` voor zowel `minimax` als `minimax-portal`.

- Standaard muziekmodel: `minimax/music-2.6` (OAuth: `minimax-portal/music-2.6`)
- Ondersteunt ook `music-2.6-free`, `music-cover` en `music-cover-free`
- Promptbesturing: `lyrics`, `instrumental`
- Uitvoerformaat: `mp3`
- Door sessies ondersteunde uitvoeringen worden losgekoppeld via de gedeelde taak-/statusflow, inclusief `action: "status"`

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: { primary: "minimax/music-2.6" },
    },
  },
}
```

<Note>
Zie [Muziekgeneratie](/nl/tools/music-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

### Videogeneratie

De gebundelde MiniMax-Plugin registreert videogeneratie via de gedeelde tool `video_generate` voor zowel `minimax` als `minimax-portal`.

- Standaard videomodel: `minimax/MiniMax-Hailuo-2.3` (OAuth: `minimax-portal/MiniMax-Hailuo-2.3`)
- Ondersteunt ook `MiniMax-Hailuo-2.3-Fast`, `MiniMax-Hailuo-02`, `I2V-01-Director`, `I2V-01-live` en `I2V-01`
- Modi: tekst-naar-video en workflows met één afbeelding als referentie
- Ondersteunt `resolution` (`768P` of `1080P` op Hailuo 2.3/02-modellen); `aspectRatio` wordt niet ondersteund en wordt genegeerd

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "minimax/MiniMax-Hailuo-2.3" },
    },
  },
}
```

<Note>
Zie [Videogeneratie](/nl/tools/video-generation) voor gedeelde toolparameters, providerselectie en failovergedrag.
</Note>

### Afbeeldingen begrijpen

De MiniMax-plugin registreert het begrijpen van afbeeldingen afzonderlijk van de tekstcatalogus:

| Provider-ID      | Standaard afbeeldingsmodel | PDF-tekstextractie |
| ---------------- | ------------------- | ------------------- |
| `minimax`        | `MiniMax-VL-01`     | `MiniMax-M2.7`      |
| `minimax-portal` | `MiniMax-VL-01`     | `MiniMax-M2.7`      |

Daarom kan automatische mediaroutering MiniMax gebruiken om afbeeldingen te begrijpen, zelfs wanneer de gebundelde catalogus van tekstproviders ook M3-chatreferenties met afbeeldingsmogelijkheden bevat. Voor het begrijpen van PDF's wordt `MiniMax-M2.7` uitsluitend gebruikt voor tekstextractie; MiniMax registreert geen conversiepad van PDF naar afbeelding.

### Zoeken op internet

De MiniMax-plugin registreert ook `web_search` via de zoek-API van MiniMax Token Plan (`/v1/coding_plan/search`).

- Provider-ID: `minimax`
- Gestructureerde resultaten: titels, URL's, fragmenten, gerelateerde zoekopdrachten
- Voorkeursomgevingsvariabele: `MINIMAX_CODE_PLAN_KEY`
- Geaccepteerde omgevingsaliassen: `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`
- Compatibiliteitsfallback: `MINIMAX_API_KEY` wanneer deze al naar een tokenplanreferentie verwijst
- Hergebruik van regio: `plugins.entries.minimax.config.webSearch.region`, daarna `MINIMAX_API_HOST`, daarna de basis-URL's van de MiniMax-provider
- Zoeken blijft provider-ID `minimax` gebruiken; de OAuth-configuratie voor CN/globaal kan de regio indirect sturen via `models.providers.minimax-portal.baseUrl` en bearer-authenticatie leveren via `MINIMAX_OAUTH_TOKEN`

De configuratie staat onder `plugins.entries.minimax.config.webSearch.*`.

<Note>
Zie [MiniMax Search](/nl/tools/minimax-search) voor de volledige configuratie en het gebruik van zoeken op internet.
</Note>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Configuratieopties">
    | Optie | Beschrijving |
    | --- | --- |
    | `models.providers.minimax.baseUrl` | Geef de voorkeur aan `https://api.minimax.io/anthropic` (Anthropic-compatibel); `https://api.minimax.io/v1` is optioneel voor OpenAI-compatibele payloads |
    | `models.providers.minimax.api` | Geef de voorkeur aan `anthropic-messages`; `openai-completions` is optioneel voor OpenAI-compatibele payloads |
    | `models.providers.minimax.apiKey` | MiniMax-API-sleutel (`MINIMAX_API_KEY`) |
    | `models.providers.minimax.models` | Definieer `id`, `name`, `reasoning`, `contextWindow`, `maxTokens`, `cost` |
    | `agents.defaults.models` | Aliassen, parameters en metadata per model |
    | `agents.defaults.modelPolicy.allow` | Optionele expliciete lijst met toegestane modellen |
    | `models.mode` | Behoud `merge` als je MiniMax naast de ingebouwde providers wilt toevoegen |
  </Accordion>

  <Accordion title="Standaardinstellingen voor denkwerk">
    Bij `api: "anthropic-messages"` injecteert OpenClaw `thinking: { type: "disabled" }` voor MiniMax M2.x-modellen, tenzij een eerdere wrapper het veld `thinking` al in de payload heeft ingesteld. Dit voorkomt dat het streaming-eindpunt van M2.x `reasoning_content` uitstuurt in deltafragmenten in OpenAI-stijl, waardoor interne redeneringen in zichtbare uitvoer zouden lekken.

    MiniMax-M3 (en M3.x) is uitgezonderd: M3 retourneert een lege `content`-array met `stop_reason: "end_turn"` wanneer denkwerk is uitgeschakeld. Daarom verwijdert OpenClaw de impliciete uitgeschakelde standaardinstelling voor M3 en dwingt het `thinking: { type: "adaptive" }` af wanneer een denkniveau is ingesteld.

    Beschikbare denkniveaus per modelfamilie:

    | Modelfamilie   | Niveaus                                   | Standaard    |
    | -------------- | ----------------------------------------- | ---------- |
    | `MiniMax-M3`   | `off`, `adaptive`                        | `adaptive` |
    | `MiniMax-M2.x` | `off`, `minimal`, `low`, `medium`, `high` | `off`      |

  </Accordion>

  <Accordion title="Snelle modus">
    `/fast on` of `params.fastMode: true` herschrijft `MiniMax-M2.7` naar `MiniMax-M2.7-highspeed` in het Anthropic-compatibele streamingpad (`api: "anthropic-messages"`, provider `minimax` of `minimax-portal`).
  </Accordion>

  <Accordion title="Fallbackvoorbeeld">
    **Het meest geschikt voor:** behoud je krachtigste model van de nieuwste generatie als primair model en schakel bij uitval over naar MiniMax M2.7. In het onderstaande voorbeeld wordt Opus als concreet primair model gebruikt; vervang het door je gewenste primaire model van de nieuwste generatie.

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-..." },
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-4-6": { alias: "primary" },
            "minimax/MiniMax-M2.7": { alias: "minimax" },
          },
          model: {
            primary: "anthropic/claude-opus-4-6",
            fallbacks: ["minimax/MiniMax-M2.7"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Gebruiksdetails van Coding Plan">
    - Gebruiks-API van Coding Plan: `https://api.minimaxi.com/v1/token_plan/remains` of `https://api.minimax.io/v1/token_plan/remains` (vereist een sleutel voor een coding plan).
    - Bij geconfigureerde waarden leidt het opvragen van gebruik de host af van `models.providers.minimax-portal.baseUrl` of `models.providers.minimax.baseUrl`, zodat globale configuraties die `https://api.minimax.io/anthropic` gebruiken `api.minimax.io` opvragen. Bij ontbrekende of onjuist gevormde basis-URL's blijft voor compatibiliteit de CN-fallback behouden.
    - OpenClaw normaliseert het gebruik van een MiniMax-codingplan naar dezelfde `% left`-weergave die andere providers gebruiken. De onbewerkte velden `usage_percent` / `usagePercent` van MiniMax geven het resterende quotum aan, niet het verbruikte quotum, dus OpenClaw keert ze om. Velden op basis van aantallen krijgen voorrang wanneer ze aanwezig zijn.
    - Wanneer de API `model_remains` retourneert, geeft OpenClaw de voorkeur aan de vermelding van het chatmodel, leidt het indien nodig het vensterlabel af van `start_time` / `end_time` en neemt het de naam van het geselecteerde model op in het planlabel, zodat vensters van codingplannen gemakkelijker te onderscheiden zijn.
    - Gebruiksmomentopnamen behandelen `minimax`, `minimax-cn`, `minimax-portal` en `minimax-portal-cn` als hetzelfde MiniMax-quotumoppervlak en geven de voorkeur aan opgeslagen MiniMax OAuth voordat ze terugvallen op omgevingsvariabelen voor Coding Plan-sleutels.

  </Accordion>
</AccordionGroup>

## Opmerkingen

- Standaard chatmodel: `MiniMax-M3`. Alternatieve chatmodellen: `MiniMax-M2.7`, `MiniMax-M2.7-highspeed`
- Onboarding en rechtstreekse configuratie met een API-sleutel schrijven modeldefinities voor M3 en beide M2.7-varianten
- Voor het begrijpen van afbeeldingen wordt de door de plugin beheerde mediaprovider `MiniMax-VL-01` gebruikt
- Werk de prijswaarden in `models.json` bij als je exacte kostentracering nodig hebt
- Gebruik `openclaw models list` om het huidige provider-ID te bevestigen en schakel vervolgens over met `openclaw models set minimax/MiniMax-M3` of `openclaw models set minimax-portal/MiniMax-M3`

<Note>
Zie [Modelproviders](/nl/concepts/model-providers) voor providerregels.
</Note>

## Problemen oplossen

<AccordionGroup>
  <Accordion title='"Onbekend model: minimax/MiniMax-M3"'>
    Dit betekent meestal dat de **MiniMax-provider niet is geconfigureerd** (er is geen overeenkomende providervermelding en er is geen MiniMax-authenticatieprofiel of omgevingssleutel gevonden). Los dit als volgt op:

    - Voer `openclaw configure` uit en selecteer een **MiniMax**-authenticatieoptie, of
    - Voeg het overeenkomende blok `models.providers.minimax` of `models.providers.minimax-portal` handmatig toe, of
    - Stel `MINIMAX_API_KEY`, `MINIMAX_OAUTH_TOKEN` of een MiniMax-authenticatieprofiel in, zodat de overeenkomende provider kan worden geïnjecteerd.

    Let erop dat het model-ID **hoofdlettergevoelig** is:

    - Pad met API-sleutel: `minimax/MiniMax-M3`, `minimax/MiniMax-M2.7` of `minimax/MiniMax-M2.7-highspeed`
    - OAuth-pad: `minimax-portal/MiniMax-M3`, `minimax-portal/MiniMax-M2.7` of `minimax-portal/MiniMax-M2.7-highspeed`

    Controleer het vervolgens opnieuw met:

    ```bash
    openclaw models list
    ```

  </Accordion>
</AccordionGroup>

<Note>
Meer hulp: [Problemen oplossen](/nl/help/troubleshooting) en [Veelgestelde vragen](/nl/help/faq).
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Afbeeldingen genereren" href="/nl/tools/image-generation" icon="image">
    Gedeelde parameters voor de afbeeldingstool en providerselectie.
  </Card>
  <Card title="Muziek genereren" href="/nl/tools/music-generation" icon="music">
    Gedeelde parameters voor de muziektool en providerselectie.
  </Card>
  <Card title="Video genereren" href="/nl/tools/video-generation" icon="video">
    Gedeelde parameters voor de videotool en providerselectie.
  </Card>
  <Card title="MiniMax Search" href="/nl/tools/minimax-search" icon="magnifying-glass">
    Configuratie voor zoeken op internet via MiniMax Token Plan.
  </Card>
  <Card title="Problemen oplossen" href="/nl/help/troubleshooting" icon="wrench">
    Algemene probleemoplossing en veelgestelde vragen.
  </Card>
</CardGroup>
