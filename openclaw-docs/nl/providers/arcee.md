---
read_when:
    - Je wilt Arcee AI gebruiken met OpenClaw
    - Je hebt de omgevingsvariabele voor de API-sleutel of de CLI-authenticatiekeuze nodig
summary: Arcee AI instellen (authenticatie + modelselectie)
title: Arcee AI
x-i18n:
    generated_at: "2026-07-27T06:04:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a4c2fc7b8d86dd0d2a300dfc48951657cbcfcd9250016f52c1804777b2966e11
    source_path: providers/arcee.md
    workflow: 16
---

[Arcee AI](https://arcee.ai) biedt de Trinity-familie van mixture-of-experts-modellen aan via een OpenAI-compatibele API. Alle Trinity-modellen vallen onder de Apache 2.0-licentie. Arcee is een officiële OpenClaw-plugin die niet met de kern wordt meegeleverd. Daarom moet je deze installeren voordat je de onboarding uitvoert.

Krijg rechtstreeks toegang tot Arcee-modellen via het Arcee-platform of via [OpenRouter](/nl/providers/openrouter).

| Eigenschap | Waarde                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| Provider | `arcee`                                                                               |
| Authenticatie     | `ARCEEAI_API_KEY` (rechtstreeks) of `OPENROUTER_API_KEY` (via OpenRouter)                   |
| API      | OpenAI-compatibel                                                                     |
| Basis-URL | `https://api.arcee.ai/api/v1` (rechtstreeks) of `https://openrouter.ai/api/v1` (OpenRouter) |

## Plugin installeren

```bash
openclaw plugins install @openclaw/arcee-provider
openclaw gateway restart
```

## Aan de slag

<Tabs>
  <Tab title="Rechtstreeks (Arcee-platform)">
    <Steps>
      <Step title="Een API-sleutel verkrijgen">
        Maak een API-sleutel aan bij [Arcee AI](https://chat.arcee.ai/).
      </Step>
      <Step title="Onboarding uitvoeren">
        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```
      </Step>
      <Step title="Een standaardmodel instellen">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="Via OpenRouter">
    <Steps>
      <Step title="Een API-sleutel verkrijgen">
        Maak een API-sleutel aan bij [OpenRouter](https://openrouter.ai/keys).
      </Step>
      <Step title="Onboarding uitvoeren">
        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```
      </Step>
      <Step title="Een standaardmodel instellen">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        Dezelfde modelreferenties werken voor zowel rechtstreekse configuraties als configuraties via OpenRouter.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Niet-interactieve configuratie

<Tabs>
  <Tab title="Rechtstreeks (Arcee-platform)">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```
  </Tab>

  <Tab title="Via OpenRouter">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```
  </Tab>
</Tabs>

## Rechtstreekse Arcee-catalogus

| Modelreferentie                      | Naam                   | Invoer | Context | Maximale uitvoer | Kosten (in/uit per 1 mln.) | Tools | Opmerkingen                                     |
| ------------------------------ | ---------------------- | ----- | ------- | ---------- | -------------------- | ----- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | tekst  | 256K    | 80K        | $0.25 / $0.90        | Nee    | Standaardmodel; uitgebreid denkproces          |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | tekst  | 128K    | 16K        | $0.25 / $1.00        | Ja   | Algemeen gebruik; 400B parameters, 13B actief  |
| `arcee/trinity-mini`           | Trinity Mini 26B       | tekst  | 128K    | 80K        | $0.045 / $0.15       | Ja   | Snel en kostenefficiënt; functieaanroepen |

<Tip>
De onboardingvoorinstelling stelt `arcee/trinity-large-thinking` in als het standaardmodel.
</Tip>

## OpenRouter-catalogus

De onboarding voor OpenRouter maakt `arcee/trinity-large-preview` en `arcee/trinity-large-thinking` beschikbaar. OpenClaw behoudt deze providergekwalificeerde modelreferenties in de configuratie en verzendt de canonieke `arcee-ai/*`-runtime-ID's van OpenRouter. Trinity Mini wordt niet langer aangeboden door OpenRouter; gebruik voor dat model de rechtstreekse Arcee-API.

## Ondersteunde functies

| Functie                                       | Ondersteund                                    |
| --------------------------------------------- | -------------------------------------------- |
| Streaming                                     | Ja                                          |
| Toolgebruik / functieaanroepen                   | Ja (Trinity Mini, Trinity Large Preview)    |
| Gestructureerde uitvoer (JSON-modus en JSON-schema) | Ja                                          |
| Uitgebreid denkproces                             | Ja (Trinity Large Thinking; tools uitgeschakeld) |

<AccordionGroup>
  <Accordion title="Opmerking over de omgeving">
    Als de Gateway als daemon (launchd/systemd) draait, zorg er dan voor dat `ARCEEAI_API_KEY`
    (of `OPENROUTER_API_KEY`) beschikbaar is voor dat proces, bijvoorbeeld in
    `~/.openclaw/.env` of via `env.shellEnv`.
  </Accordion>

  <Accordion title="OpenRouter-routering">
    OpenRouter gebruikt dezelfde `arcee/trinity-large-thinking`-modelreferentie van OpenClaw.
    OpenClaw routeert deze met de canonieke `arcee-ai/trinity-large-thinking`-
    runtime-ID van OpenRouter. Zie de
    [documentatie van de OpenRouter-provider](/nl/providers/openrouter) voor OpenRouter-specifieke
    configuratiedetails.
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="OpenRouter" href="/nl/providers/openrouter" icon="shuffle">
    Krijg met één API-sleutel toegang tot Arcee-modellen en vele andere modellen.
  </Card>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers en modelreferenties kiezen en failovergedrag configureren.
  </Card>
</CardGroup>
