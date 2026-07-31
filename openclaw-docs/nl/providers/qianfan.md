---
read_when:
    - Je wilt één API-sleutel voor veel LLM's
    - Je hebt installatie-instructies voor Baidu Qianfan nodig
summary: Gebruik de uniforme API van Qianfan om toegang te krijgen tot veel modellen in OpenClaw
title: Qianfan
x-i18n:
    generated_at: "2026-07-27T05:47:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 31387a53ee4472e2d20ae939ea75cea0d6f6367501becd56a8654fd97fdf0804
    source_path: providers/qianfan.md
    workflow: 16
---

Qianfan is Baidu's MaaS-platform: een uniforme, met OpenAI compatibele API die aanvragen via één eindpunt en één API-sleutel naar vele modellen routeert. OpenClaw levert deze als de officiële externe Plugin `@openclaw/qianfan-provider`.

| Eigenschap    | Waarde                                   |
| ------------- | ---------------------------------------- |
| Provider      | `qianfan`                       |
| Authenticatie | `QIANFAN_API_KEY`                       |
| API           | OpenAI-compatibel (`openai-completions`)   |
| Basis-URL     | `https://qianfan.baidubce.com/v2`                       |
| Standaardmodel | `qianfan/deepseek-v3.2`                      |

## Plugin installeren

Installeer de officiële Plugin en start daarna de Gateway opnieuw:

```bash
openclaw plugins install @openclaw/qianfan-provider
openclaw gateway restart
```

## Aan de slag

<Steps>
  <Step title="Een Baidu Cloud-account maken">
    Registreer je of meld je aan bij de [Qianfan-console](https://console.bce.baidu.com/qianfan/ais/console/apiKey) en zorg dat toegang tot de Qianfan-API is ingeschakeld.
  </Step>
  <Step title="Een API-sleutel genereren">
    Maak een nieuwe toepassing of selecteer een bestaande en genereer vervolgens een API-sleutel. Baidu Cloud-sleutels gebruiken de indeling `bce-v3/ALTAK-...`.
  </Step>
  <Step title="Onboarding uitvoeren">
    ```bash
    openclaw onboard --auth-choice qianfan-api-key
    ```

    Niet-interactieve uitvoeringen lezen de sleutel uit `--qianfan-api-key <key>` of
    `QIANFAN_API_KEY`. Onboarding schrijft de providerconfiguratie, voegt de
    alias `QIANFAN` voor het standaardmodel toe en stelt `qianfan/deepseek-v3.2`
    in als standaardmodel wanneer er geen model is geconfigureerd.

  </Step>
  <Step title="Controleren of het model beschikbaar is">
    ```bash
    openclaw models list --provider qianfan
    ```
  </Step>
</Steps>

## Ingebouwde catalogus

| Modelreferentie                     | Invoer      | Context | Maximale uitvoer | Redeneren | Opmerkingen   |
| ------------------------------------ | ----------- | ------- | ---------------- | --------- | ------------- |
| `qianfan/deepseek-v3.2`                   | tekst       | 98,304  | 32,768           | Ja        | Standaardmodel |
| `qianfan/ernie-5.0-thinking-preview`                   | tekst, afbeelding | 119,000 | 64,000     | Ja        | Multimodaal   |

De catalogus is statisch; modellen worden niet live gedetecteerd.

<Tip>
Je hoeft `models.providers.qianfan` alleen te overschrijven wanneer je een aangepaste basis-URL of aangepaste modelmetagegevens nodig hebt.
</Tip>

## Configuratievoorbeeld

```json5
{
  env: { QIANFAN_API_KEY: "bce-v3/ALTAK-..." },
  agents: {
    defaults: {
      model: { primary: "qianfan/deepseek-v3.2" },
      models: {
        "qianfan/deepseek-v3.2": { alias: "QIANFAN" },
      },
    },
  },
  models: {
    providers: {
      qianfan: {
        baseUrl: "https://qianfan.baidubce.com/v2",
        api: "openai-completions",
        models: [
          {
            id: "deepseek-v3.2",
            name: "DEEPSEEK V3.2",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 98304,
            maxTokens: 32768,
          },
          {
            id: "ernie-5.0-thinking-preview",
            name: "ERNIE-5.0-Thinking-Preview",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 119000,
            maxTokens: 64000,
          },
        ],
      },
    },
  },
}
```

<Note>
Modelreferenties gebruiken het voorvoegsel `qianfan/` (bijvoorbeeld `qianfan/deepseek-v3.2`).
</Note>

<AccordionGroup>
  <Accordion title="Transport en compatibiliteit">
    Qianfan werkt via het met OpenAI compatibele transportpad en niet via de systeemeigen vormgeving van OpenAI-aanvragen. Standaardfuncties van de OpenAI-SDK werken, maar providerspecifieke parameters worden mogelijk niet doorgestuurd.
  </Accordion>

  <Accordion title="Problemen oplossen">
    - Zorg dat je API-sleutel begint met `bce-v3/ALTAK-` en dat toegang tot de Qianfan-API is ingeschakeld in de Baidu Cloud-console.
    - Als modellen niet worden weergegeven, controleer dan of de Qianfan-service voor je account is geactiveerd.
    - Wijzig de basis-URL alleen als je een aangepast eindpunt of een aangepaste proxy gebruikt.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/configuration-reference" icon="gear">
    Volledige OpenClaw-configuratiereferentie.
  </Card>
  <Card title="Agent instellen" href="/nl/concepts/agent" icon="robot">
    Standaardinstellingen en modeltoewijzingen voor agents configureren.
  </Card>
  <Card title="Qianfan-API-documentatie" href="https://cloud.baidu.com/doc/qianfan-api/s/3m7of64lb" icon="arrow-up-right-from-square">
    Officiële documentatie voor de Qianfan-API.
  </Card>
</CardGroup>
