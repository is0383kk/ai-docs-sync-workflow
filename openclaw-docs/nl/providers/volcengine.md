---
read_when:
    - Je wilt Volcano Engine- of Doubao-modellen gebruiken met OpenClaw
    - Je moet de Volcengine-API-sleutel instellen
    - Je wilt tekst-naar-spraak van Volcengine Speech gebruiken
summary: Volcano Engine-configuratie (Doubao-modellen, coding-eindpunten en Seed Speech TTS)
title: Volcengine (Doubao)
x-i18n:
    generated_at: "2026-07-27T05:20:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89538772b704499547ecf0274c5bb9bf8f68cc267dc7f484d3236921a9c89681
    source_path: providers/volcengine.md
    workflow: 16
---

De Volcengine-provider biedt toegang tot Doubao-modellen en modellen van derden die op Volcano Engine worden gehost, met afzonderlijke eindpunten voor algemene en programmeerworkloads. Dezelfde gebundelde Plugin registreert ook Volcengine Speech als TTS-provider.

| Detail            | Waarde                                                     |
| ----------------- | ---------------------------------------------------------- |
| Providers         | `volcengine` (algemeen + TTS), `volcengine-plan` (programmeren) |
| Modelauthenticatie | `VOLCANO_ENGINE_API_KEY`                                        |
| TTS-authenticatie | `VOLCENGINE_TTS_API_KEY` of `BYTEPLUS_SEED_SPEECH_API_KEY`                   |
| API               | OpenAI-compatibele modellen, BytePlus Seed Speech TTS      |

## Aan de slag

<Steps>
  <Step title="Stel de API-sleutel in">
    Voer de interactieve onboarding uit:

    ```bash
    openclaw onboard --auth-choice volcengine-api-key
    ```

    Hiermee worden zowel de algemene provider (`volcengine`) als de programmeerprovider (`volcengine-plan`) met één API-sleutel geregistreerd.

  </Step>
  <Step title="Stel een standaardmodel in">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "volcengine-plan/ark-code-latest" },
        },
      },
    }
    ```
  </Step>
  <Step title="Controleer of het model beschikbaar is">
    ```bash
    openclaw models list --provider volcengine
    openclaw models list --provider volcengine-plan
    ```
  </Step>
</Steps>

<Tip>
Geef voor niet-interactieve configuratie (CI, scripts) de sleutel rechtstreeks door:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice volcengine-api-key \
  --volcengine-api-key "$VOLCANO_ENGINE_API_KEY"
```

</Tip>

## Providers en eindpunten

| Provider          | Eindpunt                                  | Gebruiksscenario     |
| ----------------- | ----------------------------------------- | -------------------- |
| `volcengine` | `ark.cn-beijing.volces.com/api/v3`                       | Algemene modellen    |
| `volcengine-plan` | `ark.cn-beijing.volces.com/api/coding/v3`                       | Programmeermodellen  |

<Note>
Beide providers worden met één API-sleutel geconfigureerd. Tijdens de configuratie worden beide automatisch geregistreerd en de modelkiezer van de programmeerprovider hergebruikt ook de authenticatie van de algemene provider (`volcengine-plan` is een authenticatiealias van `volcengine`).
</Note>

## Ingebouwde catalogus

<Tabs>
  <Tab title="Algemeen (volcengine)">
    | Modelreferentie                               | Naam                            | Invoer      | Context |
    | --------------------------------------------- | ------------------------------- | ----------- | ------- |
    | `volcengine/deepseek-v3-2-251201`                            | DeepSeek V3.2                   | tekst, afbeelding | 128,000 |
    | `volcengine/doubao-seed-1-8-251228`                            | Doubao Seed 1.8                 | tekst, afbeelding | 256,000 |
    | `volcengine/doubao-seed-code-preview-251028`                            | doubao-seed-code-preview-251028 | tekst, afbeelding | 256,000 |
    | `volcengine/glm-4-7-251222`                            | GLM 4.7                         | tekst, afbeelding | 200,000 |
    | `volcengine/kimi-k2-5-260127`                            | Kimi K2.5                       | tekst, afbeelding | 256,000 |
  </Tab>
  <Tab title="Programmeren (volcengine-plan)">
    | Modelreferentie                               | Naam                     | Invoer | Context |
    | --------------------------------------------- | ------------------------ | ------ | ------- |
    | `volcengine-plan/ark-code-latest`                            | Ark Coding Plan          | tekst  | 256,000 |
    | `volcengine-plan/doubao-seed-code`                            | Doubao Seed Code         | tekst  | 256,000 |
  </Tab>
</Tabs>

Beide catalogi zijn statisch (geen `/models`-detectieaanroep) en ondersteunen OpenAI-compatibele gebruiksregistratie bij streaming. Toolschema's voor beide providers verwijderen automatisch de sleutelwoorden `minLength`, `maxLength`, `minItems`, `maxItems`, `minContains` en `maxContains`, omdat de tool-call-API van Volcengine deze afwijst.

## Tekst-naar-spraak

Volcengine TTS gebruikt de BytePlus Seed Speech HTTP-API (`voice.ap-southeast-1.bytepluses.com`) en wordt afzonderlijk geconfigureerd van de API-sleutel voor de OpenAI-compatibele Doubao-model-API. Open in de BytePlus-console Seed Speech > Settings > API Keys, kopieer de API-sleutel en stel vervolgens het volgende in:

```bash
export VOLCENGINE_TTS_API_KEY="byteplus_seed_speech_api_key"
export VOLCENGINE_TTS_RESOURCE_ID="seed-tts-1.0"
```

Schakel deze vervolgens in via `openclaw.json`:

```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "byteplus_seed_speech_api_key",
        voice: "en_female_anna_mars_bigtts",
        speedRatio: 1.0,
      },
    },
  },
}
```

Beschikbare velden onder `tts.providers.volcengine`: `apiKey`, `voice`, `speedRatio` (0.2-3.0), `emotion`, `cluster`, `resourceId`, `appKey` en `baseUrl`. `!emotion=<value>` werkt ook als inline steminstructie wanneer overschrijvingen van de steminstelling zijn toegestaan.

Voor doelen voor spraakberichten vraagt OpenClaw de providerspecifieke `ogg_opus` aan. Voor normale audiobijlagen vraagt het `mp3` aan. De provideraliassen `bytedance` en `doubao` verwijzen ook naar deze spraakprovider.

De standaardresource-id is `seed-tts-1.0`, de gebruiksmachtiging die BytePlus standaard aan nieuw aangemaakte Seed Speech-API-sleutels verleent. Als je project een TTS 2.0-gebruiksmachtiging heeft, stel je `VOLCENGINE_TTS_RESOURCE_ID=seed-tts-2.0` in.

<Warning>
`VOLCANO_ENGINE_API_KEY` is bedoeld voor de ModelArk-/Doubao-modeleindpunten en is geen Seed Speech-API-sleutel. Voor TTS is een Seed Speech-API-sleutel uit de BytePlus Speech Console vereist, of een verouderd AppID/token-paar uit de Speech Console.
</Warning>

Verouderde AppID/token-authenticatie blijft ondersteund voor oudere Speech Console-applicaties:

```bash
export VOLCENGINE_TTS_APPID="speech_app_id"
export VOLCENGINE_TTS_TOKEN="speech_access_token"
export VOLCENGINE_TTS_CLUSTER="volcano_tts"
```

Andere optionele TTS-omgevingsvariabelen: `VOLCENGINE_TTS_VOICE`, `VOLCENGINE_TTS_APP_KEY` en `VOLCENGINE_TTS_BASE_URL` overschrijven de overeenkomstige `tts.providers.volcengine`-configuratievelden wanneer ze zijn ingesteld.

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Standaardmodel na onboarding">
    `openclaw onboard --auth-choice volcengine-api-key` stelt `volcengine-plan/ark-code-latest` in als standaardmodel en registreert tegelijk de algemene `volcengine`-catalogus.
  </Accordion>

  <Accordion title="Terugvalgedrag van de modelkiezer">
    Tijdens de modelselectie bij onboarding/configuratie geeft de Volcengine-authenticatiekeuze de voorkeur aan zowel de rijen `volcengine/*` als `volcengine-plan/*`. Als die modellen nog niet zijn geladen, valt OpenClaw terug op de ongefilterde catalogus in plaats van een lege providergebonden modelkiezer weer te geven.
  </Accordion>

  <Accordion title="Omgevingsvariabelen voor daemonprocessen">
    Als de Gateway als daemon wordt uitgevoerd (launchd/systemd), zorg er dan voor dat omgevingsvariabelen voor modellen en TTS, zoals `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `VOLCENGINE_TTS_APPID` en `VOLCENGINE_TTS_TOKEN`, beschikbaar zijn voor dat proces (bijvoorbeeld in `~/.openclaw/.env` of via `env.shellEnv`).
  </Accordion>
</AccordionGroup>

<Warning>
Wanneer OpenClaw als achtergrondservice wordt uitgevoerd, worden omgevingsvariabelen die in je interactieve shell zijn ingesteld niet automatisch overgenomen. Zie de opmerking over de daemon hierboven.
</Warning>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration" icon="gear">
    Volledige configuratiereferentie voor agents, modellen en providers.
  </Card>
  <Card title="Problemen oplossen" href="/nl/help/troubleshooting" icon="wrench">
    Veelvoorkomende problemen en stappen voor foutopsporing.
  </Card>
  <Card title="Veelgestelde vragen" href="/nl/help/faq" icon="circle-question">
    Veelgestelde vragen over het configureren van OpenClaw.
  </Card>
</CardGroup>
