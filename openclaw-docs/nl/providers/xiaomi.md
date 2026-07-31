---
read_when:
    - Je wilt Xiaomi MiMo-modellen in OpenClaw
    - Je hebt Xiaomi MiMo-authenticatie of een Token Plan-configuratie nodig
summary: Gebruik de pay-as-you-go- en Token Plan-modellen van Xiaomi MiMo met OpenClaw
title: Xiaomi MiMo
x-i18n:
    generated_at: "2026-07-27T06:07:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef79dea8332903c726f076c91b3b458e2d98534d402a412e7c156c06b2912a69
    source_path: providers/xiaomi.md
    workflow: 16
---

Xiaomi MiMo is het API-platform voor **MiMo**-modellen. De meegeleverde `xiaomi`
plugin (`enabledByDefault: true`, geen installatiestap) registreert twee tekstproviders
plus een spraakprovider (TTS):

- `xiaomi` - sleutels met betalen naar gebruik (`sk-...`)
- `xiaomi-token-plan` - Token Plan-sleutels (`tp-...`) met regionale endpointvoorinstellingen

| Eigenschap       | Waarde                                                                                                                                             |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Provider-id's    | `xiaomi` (betalen naar gebruik), `xiaomi-token-plan` (Token Plan)                                                                         |
| Auth-omgevingsvariabelen | `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`                                                                                                      |
| Onboarding-vlaggen | `--auth-choice xiaomi-api-key`, `--auth-choice xiaomi-token-plan-cn`, `--auth-choice xiaomi-token-plan-sgp`, `--auth-choice xiaomi-token-plan-ams` |
| Directe CLI-vlaggen | `--xiaomi-api-key <key>`, `--xiaomi-token-plan-api-key <key>`                                                                                      |
| API              | OpenAI-compatibele chatvoltooiingen (`openai-completions`)                                                                                          |
| Spraakcontract   | `speechProviders: ["xiaomi"]`                                                                                                                      |
| Basis-URL's      | Betalen naar gebruik: `https://api.xiaomimimo.com/v1`; Token Plan: `token-plan-{cn,sgp,ams}.xiaomimimo.com/v1`                                            |
| Standaardmodellen | `xiaomi/mimo-v2.5`, `xiaomi-token-plan/mimo-v2.5-pro`                                                                                              |
| TTS-standaard    | `mimo-v2.5-tts`, stem `mimo_default`; stemontwerpmodel `mimo-v2.5-tts-voicedesign`                                                               |

## Aan de slag

<Steps>
  <Step title="Verkrijg de juiste sleutel">
    Maak een sleutel voor betalen naar gebruik aan in de [Xiaomi MiMo-console](https://platform.xiaomimimo.com/#/console/api-keys), of open je Token Plan-abonnementspagina en kopieer de regionale OpenAI-compatibele basis-URL plus de bijbehorende `tp-...`-sleutel.
  </Step>

  <Step title="Voer onboarding uit">
    Betalen naar gebruik:

    ```bash
    openclaw onboard --auth-choice xiaomi-api-key
    ```

    Token Plan:

    ```bash
    openclaw onboard --auth-choice xiaomi-token-plan-sgp
    ```

    Of geef de sleutels rechtstreeks door:

    ```bash
    openclaw onboard --auth-choice xiaomi-api-key --xiaomi-api-key "$XIAOMI_API_KEY"
    openclaw onboard --auth-choice xiaomi-token-plan-sgp --xiaomi-token-plan-api-key "$XIAOMI_TOKEN_PLAN_API_KEY"
    ```

  </Step>
  <Step title="Controleer of het model beschikbaar is">
    ```bash
    openclaw models list --provider xiaomi
    openclaw models list --provider xiaomi-token-plan
    ```
  </Step>
</Steps>

<Tip>
Onboarding valideert de vorm van de sleutel en waarschuwt wanneer een `tp-...`-sleutel wordt ingevoerd in het traject voor betalen naar gebruik, of een `sk-...`-sleutel wordt ingevoerd in het Token Plan-traject.
</Tip>

## Catalogus voor betalen naar gebruik

| Modelreferentie        | Invoer      | Context   | Maximale uitvoer | Redeneren | Opmerkingen    |
| ---------------------- | ----------- | --------- | ---------------- | --------- | -------------- |
| `xiaomi/mimo-v2.5`     | tekst, afbeelding | 1,048,576 | 131,072    | Ja        | Standaardmodel |
| `xiaomi/mimo-v2.5-pro` | tekst        | 1,048,576 | 131,072    | Ja        | Vlaggenschip   |

## Token Plan-catalogus

Kies de Token Plan-authenticatiekeuze die overeenkomt met de regionale basis-URL die in de abonnementsinterface van Xiaomi wordt weergegeven:

| Authenticatiekeuze      | Basis-URL                                  |
| ----------------------- | ------------------------------------------ |
| `xiaomi-token-plan-cn`  | `https://token-plan-cn.xiaomimimo.com/v1`  |
| `xiaomi-token-plan-sgp` | `https://token-plan-sgp.xiaomimimo.com/v1` |
| `xiaomi-token-plan-ams` | `https://token-plan-ams.xiaomimimo.com/v1` |

| Modelreferentie                   | Invoer      | Context   | Maximale uitvoer | Redeneren | Opmerkingen    |
| --------------------------------- | ----------- | --------- | ---------------- | --------- | -------------- |
| `xiaomi-token-plan/mimo-v2.5-pro` | tekst        | 1,048,576 | 131,072    | Ja        | Standaardmodel |
| `xiaomi-token-plan/mimo-v2.5`     | tekst, afbeelding | 1,048,576 | 131,072    | Ja        | Multimodaal    |

`xiaomi-token-plan` heeft een regionale basis-URL nodig om te worden omgezet. Het ondersteunde traject
is een meegeleverde Token Plan-onboardingkeuze of een expliciet
`models.providers.xiaomi-token-plan`-configuratieblok waarin `baseUrl` is ingesteld; de
provider wordt zonder een van beide niet aangeboden.

## Redeneermodellen

`mimo-v2.5` en `mimo-v2.5-pro` ondersteunen
OpenClaws [`/think`-instructie](/nl/tools/thinking) met de niveaus `off`,
`minimal`, `low`, `medium`, `high`, `xhigh` en `max` (standaard `high`).

## Tekst-naar-spraak

De meegeleverde `xiaomi`-plugin registreert Xiaomi MiMo ook als spraakprovider
voor `tts`. Deze roept Xiaomi's TTS-contract voor chatvoltooiingen aan met de
tekst als een `assistant`-bericht en optionele stijlaanwijzingen als een `user`-
bericht.

| Eigenschap | Waarde                                   |
| ---------- | ---------------------------------------- |
| TTS-id     | `xiaomi` (`mimo`-alias)                  |
| Authenticatie | `XIAOMI_API_KEY`                         |
| API        | `POST /v1/chat/completions` met `audio` |
| Standaard  | `mimo-v2.5-tts`, stem `mimo_default`    |
| Uitvoer    | Standaard MP3; WAV wanneer geconfigureerd |

```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "xiaomi_api_key",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
        style: "Bright, natural, conversational tone.",
      },
    },
  },
}
```

Ingebouwde stemmen: `mimo_default`, `default_zh`, `default_en`, `Mia`, `Chloe`,
`Milo`, `Dean`. Het model met vooraf ingestelde stemmen `mimo-v2.5-tts` gebruikt `audio.voice`, dus
OpenClaw verzendt `speakerVoice` voor dat model.

Het stemontwerpmodel `mimo-v2.5-tts-voicedesign` genereert de stem op basis van een
stijlprompt in natuurlijke taal in plaats van een vooraf ingestelde stem-id. Stel `style` in op
de gewenste stembeschrijving; OpenClaw verzendt deze als het `user`-bericht, verzendt
de uitgesproken tekst als het `assistant`-bericht en laat `audio.voice` weg voor dit
model.

```json5
{
  tts: {
    provider: "xiaomi",
    providers: {
      xiaomi: {
        model: "mimo-v2.5-tts-voicedesign",
        format: "wav",
        style: "Warm, natural female voice with clear pronunciation.",
      },
    },
  },
}
```

Voor kanalen die een synthesedoel voor spraakberichten aanvragen (Discord, Feishu,
Matrix, Telegram en WhatsApp), transcodeert OpenClaw de Xiaomi-uitvoer vóór levering naar 48kHz
mono-Opus met `ffmpeg`.

## Configuratievoorbeeld

```json5
{
  env: { XIAOMI_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "xiaomi/mimo-v2.5" } } },
  models: {
    mode: "merge",
    providers: {
      xiaomi: {
        baseUrl: "https://api.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_API_KEY",
        models: [
          {
            id: "mimo-v2.5",
            name: "Xiaomi MiMo V2.5",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
          {
            id: "mimo-v2.5-pro",
            name: "Xiaomi MiMo V2.5 Pro",
            reasoning: true,
            input: ["text"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Prijs- en compatibiliteitsvlaggen zijn afkomstig uit het meegeleverde pluginmanifest, daarom laat het configuratievoorbeeld `cost` en `compat` weg om afwijkingen van het runtimegedrag te voorkomen.

Token Plan:

```json5
{
  env: { XIAOMI_TOKEN_PLAN_API_KEY: "tp-your-key" },
  agents: { defaults: { model: { primary: "xiaomi-token-plan/mimo-v2.5-pro" } } },
  models: {
    mode: "merge",
    providers: {
      "xiaomi-token-plan": {
        baseUrl: "https://token-plan-sgp.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_TOKEN_PLAN_API_KEY",
        models: [
          {
            id: "mimo-v2.5-pro",
            name: "Xiaomi MiMo V2.5 Pro",
            reasoning: true,
            input: ["text"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
          {
            id: "mimo-v2.5",
            name: "Xiaomi MiMo V2.5",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

Prijzen zijn afkomstig uit het meegeleverde manifest (Token Plan-modellen bevatten gelaagde prijzen voor cachelezingen), daarom laat het configuratievoorbeeld `cost` weg.

<AccordionGroup>
  <Accordion title="Gedrag voor automatische injectie">
    De provider `xiaomi` wordt automatisch ingeschakeld wanneer `XIAOMI_API_KEY` in je omgeving is ingesteld of wanneer er een authenticatieprofiel bestaat. `xiaomi-token-plan` heeft een regionale basis-URL nodig, dus het ondersteunde traject is de meegeleverde Token Plan-onboardingkeuze of een expliciet `models.providers.xiaomi-token-plan`-configuratieblok.
  </Accordion>

  <Accordion title="Modeldetails">
    - **mimo-v2.5** - standaard voor betalen naar gebruik en multimodale V2.5-route van Token Plan.
    - **mimo-v2.5-pro** - toonaangevend redeneermodel en standaard voor Token Plan.

    <Note>
    Modellen voor betalen naar gebruik gebruiken het voorvoegsel `xiaomi/`. Token Plan-modellen gebruiken het voorvoegsel `xiaomi-token-plan/`.
    </Note>

  </Accordion>

  <Accordion title="Probleemoplossing">
    - Als modellen niet verschijnen, controleer dan of de relevante omgevingsvariabele voor de sleutel of het authenticatieprofiel aanwezig en geldig is.
    - Controleer voor Token Plan of de gekozen onboardingregio overeenkomt met de basis-URL op de abonnementspagina en of de sleutel begint met `tp-`.
    - Wanneer de Gateway als daemon wordt uitgevoerd, moet de sleutel beschikbaar zijn voor dat proces (bijvoorbeeld in `~/.openclaw/.env` of via `env.shellEnv`).

    <Warning>
    Sleutels die alleen in je interactieve shell zijn ingesteld, zijn niet zichtbaar voor door een daemon beheerde Gateway-processen. Gebruik `~/.openclaw/.env` of de `env.shellEnv`-configuratie voor permanente beschikbaarheid.
    </Warning>

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Denk­niveaus" href="/nl/tools/thinking" icon="brain">
    Syntaxis van de `/think`-instructie en niveautoewijzing.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledige configuratiereferentie voor OpenClaw.
  </Card>
  <Card title="Xiaomi MiMo-console" href="https://platform.xiaomimimo.com" icon="arrow-up-right-from-square">
    Xiaomi MiMo-dashboard en beheer van API-sleutels.
  </Card>
</CardGroup>
