---
read_when:
    - Je wilt één API-sleutel voor veel LLM's
    - Je wilt modellen via Kilo Gateway uitvoeren in OpenClaw
summary: Gebruik de uniforme API van Kilo Gateway om toegang te krijgen tot veel modellen in OpenClaw
title: Kilo Gateway
x-i18n:
    generated_at: "2026-07-27T06:09:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0246a1a77f4265168b213e0167360e1cd89dc2ca864997f08cae5331037f9e89
    source_path: providers/kilocode.md
    workflow: 16
---

Kilo Gateway routeert aanvragen naar veel modellen achter één OpenAI-compatibel eindpunt en één API-sleutel.

| Eigenschap | Waarde                             |
| ---------- | ---------------------------------- |
| Provider   | `kilocode`                 |
| Authenticatie | `KILOCODE_API_KEY`              |
| API        | OpenAI-compatibel                  |
| Basis-URL  | `https://api.kilo.ai/api/gateway/`                 |

## Plugin installeren

```bash
openclaw plugins install @openclaw/kilocode-provider
openclaw gateway restart
```

## Instellen

<Steps>
  <Step title="Een account aanmaken">
    Ga naar [app.kilo.ai](https://app.kilo.ai), meld je aan of maak een account aan en genereer vervolgens een API-sleutel.
  </Step>
  <Step title="Onboarding uitvoeren">
    ```bash
    openclaw onboard --auth-choice kilocode-api-key
    ```

    Of stel de omgevingsvariabele rechtstreeks in:

    ```bash
    export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
    ```

  </Step>
  <Step title="Controleren of het model beschikbaar is">
    ```bash
    openclaw models list --provider kilocode
    ```
  </Step>
</Steps>

## Standaardmodel en catalogus

Het standaardmodel is `kilocode/kilo-auto/balanced`, de gebalanceerde slimme routeringslaag van Kilo Gateway.
OpenClaw publiceert hiervoor geen toewijzing van taken aan upstreammodellen; de routering achter
`kilo-auto/balanced` wordt beheerd door Kilo Gateway.

Bij het opstarten bevraagt OpenClaw `GET https://api.kilo.ai/api/gateway/models` en voegt het ontdekte modellen
vóór een statische terugvalcatalogus samen. De statische terugvalcatalogus bevat alleen
`kilocode/kilo-auto/balanced` (`Auto Balanced`, `input: ["text", "image"]`, `reasoning: true`,
`contextWindow: 1000000`, `maxTokens: 65536`).

Elk model op de Gateway is adresseerbaar als `kilocode/<upstream-id>` (bijvoorbeeld
`kilocode/anthropic/claude-sonnet-4`, `kilocode/openai/gpt-5.5`). Voer `/models kilocode` of
`openclaw models list --provider kilocode` uit om de volledige lijst met ontdekte modellen te bekijken.

## Configuratievoorbeeld

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo-auto/balanced" },
    },
  },
}
```

## Opmerkingen over het gedrag

<AccordionGroup>
  <Accordion title="Transport en compatibiliteit">
    Kilo Gateway is compatibel met OpenRouter en gebruikt daarom het proxy-achtige OpenAI-compatibele aanvraagpad
    in plaats van systeemeigen OpenAI-aanvraagvorming (geen `store`, geen OpenAI-payload voor redeneerinspanning).

    - Kilo-verwijzingen die door Gemini worden ondersteund, blijven op het proxy-Gemini-pad: OpenClaw schoont daar Gemini-gedachtehandtekeningen
      op, maar schakelt systeemeigen Gemini-validatie voor opnieuw afspelen of bootstrap-herschrijvingen niet in.
    - Aanvragen gebruiken een Bearer-token dat op basis van je API-sleutel is samengesteld.

  </Accordion>

  <Accordion title="Streamwrapper en redeneren">
    De Kilo-streamwrapper voegt een `X-KILOCODE-FEATURE`-aanvraagheader toe (standaard `openclaw`,
    te overschrijven met de omgevingsvariabele `KILOCODE_FEATURE`) en normaliseert payloads voor redeneerinspanning voor
    modellen die dit ondersteunen.

    <Warning>
    Verwijzingen naar `kilocode/kilo-auto/balanced` en `x-ai/*` slaan de injectie van redeneerinspanning over. Gebruik een concrete
    modelverwijzing, zoals `kilocode/anthropic/claude-sonnet-4`, als je ondersteuning voor redeneren nodig hebt.
    </Warning>

  </Accordion>

  <Accordion title="Problemen oplossen">
    - Als modeldetectie bij het opstarten mislukt, valt OpenClaw terug op de statische catalogus met `kilocode/kilo-auto/balanced`.
    - Controleer of je API-sleutel geldig is en of de gewenste modellen voor je Kilo-account zijn ingeschakeld.
    - Wanneer Gateway als daemon wordt uitgevoerd, moet `KILOCODE_API_KEY` beschikbaar zijn voor dat proces (bijvoorbeeld in `~/.openclaw/.env` of via `env.shellEnv`).

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelverwijzingen en terugvalgedrag kiezen.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledige configuratiereferentie voor OpenClaw.
  </Card>
  <Card title="Kilo Gateway" href="https://app.kilo.ai" icon="arrow-up-right-from-square">
    Dashboard, API-sleutels en accountbeheer van Kilo Gateway.
  </Card>
</CardGroup>
