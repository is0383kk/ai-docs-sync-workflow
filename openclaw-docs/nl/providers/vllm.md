---
read_when:
    - Je wilt OpenClaw gebruiken met een lokale vLLM-server
    - Je wilt OpenAI-compatibele /v1-eindpunten met je eigen modellen
summary: Voer OpenClaw uit met vLLM (OpenAI-compatibele lokale server)
title: vLLM
x-i18n:
    generated_at: "2026-07-27T05:14:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM biedt open-sourcemodellen (en enkele aangepaste modellen) aan via een **OpenAI-compatibele** HTTP-API. OpenClaw maakt verbinding via de `openai-completions`-API en kan modellen **automatisch detecteren** wanneer je dit inschakelt met `VLLM_API_KEY`.

| Eigenschap       | Waarde                                     |
| ---------------- | ------------------------------------------ |
| Provider-ID      | `vllm`                         |
| API              | `openai-completions` (OpenAI-compatibel)     |
| Authenticatie    | Omgevingsvariabele `VLLM_API_KEY`      |
| Standaardbasis-URL | `http://127.0.0.1:8000/v1`                       |
| Streaminggebruik | Ondersteund (`stream_options.include_usage`)           |

## Aan de slag

<Steps>
  <Step title="Start vLLM met een OpenAI-compatibele server">
    Je basis-URL moet `/v1`-eindpunten aanbieden (`/v1/models`, `/v1/chat/completions`). vLLM draait doorgaans op:

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="Stel de omgevingsvariabele voor de API-sleutel in">
    Elke niet-lege waarde werkt als je server geen authenticatie afdwingt:

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="Selecteer een model">
    Vervang dit door een van je vLLM-model-ID's:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="Controleer of het model beschikbaar is">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
Geef voor niet-interactieve configuratie (CI, scripts) de basis-URL, sleutel en het model rechtstreeks door:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## Modeldetectie (impliciete provider)

Wanneer `VLLM_API_KEY` is ingesteld (of er een authenticatieprofiel bestaat) en `models.providers.vllm` **niet** is gedefinieerd, bevraagt OpenClaw `GET http://127.0.0.1:8000/v1/models` en zet het de geretourneerde ID's om in modelvermeldingen.

<Note>
Als je `models.providers.vllm` expliciet instelt, gebruikt OpenClaw alleen de modellen die je hebt gedeclareerd. Voeg `"vllm/*": {}` toe aan `agents.defaults.models` om OpenClaw ook het `/models`-eindpunt van die geconfigureerde provider te laten bevragen en alle aangeboden vLLM-modellen op te nemen.
</Note>

## Expliciete configuratie

Configureer dit expliciet wanneer vLLM op een andere host of poort draait, je `contextWindow`/`maxTokens` wilt vastzetten, je server een echte API-sleutel vereist of je verbinding maakt met een vertrouwd loopback-, LAN- of Tailscale-eindpunt:

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // Optioneel: verleng de time-out voor aanvragen voor langzame lokale modellen
        models: [
          {
            id: "your-model-id",
            name: "Lokaal vLLM-model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Voeg een jokerteken toe aan de zichtbare modelcatalogus om de provider dynamisch te houden zonder elk model op te sommen:

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Gedrag als proxy">
    vLLM wordt behandeld als een proxyachtige, OpenAI-compatibele `/v1`-backend, niet als een native OpenAI-eindpunt:

    | Gedrag                                  | Toegepast?                       |
    | --------------------------------------- | -------------------------------- |
    | Native vormgeving van OpenAI-aanvragen  | Nee                              |
    | `service_tier`                      | Niet verzonden                   |
    | Responses `store`            | Niet verzonden                   |
    | Aanwijzingen voor promptcache           | Niet verzonden                   |
    | OpenAI-compatibele vormgeving van reasoning-payloads | Niet toegepast          |
    | Verborgen OpenClaw-attributieheaders    | Niet geïnjecteerd bij aangepaste basis-URL's |

  </Accordion>

  <Accordion title="Besturingselementen voor het denkproces van Qwen">
    Stel voor Qwen-modellen `compat.thinkingFormat: "qwen-chat-template"` in op de modelrij wanneer de server kwargs voor de Qwen-chatsjabloon verwacht. Deze modellen bieden een binair `/think`-profiel (`off`, `on`), omdat denken via de Qwen-chatsjabloon een aan/uit-vlag is en geen inspanningsladder zoals bij OpenAI.

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw wijst `/think off` toe aan:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    Denkmodi anders dan `off` verzenden `enable_thinking: true`. Als je eindpunt in plaats daarvan vlaggen op het hoogste niveau in DashScope-stijl verwacht, gebruik je `compat.thinkingFormat: "qwen"` om `enable_thinking` in de hoofdstructuur van de aanvraag te verzenden.

  </Accordion>

  <Accordion title="Besturingselementen voor het denkproces van Nemotron 3">
    Voor `vllm/nemotron-3-*`-modellen waarvoor denken is uitgeschakeld, verzendt de gebundelde Plugin:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    Stel `chat_template_kwargs` in onder de modelparameters om deze waarden aan te passen. Als je ook `params.extra_body.chat_template_kwargs` instelt, krijgt die waarde voorrang, omdat `extra_body` de laatste overschrijving van de aanvraagbody is.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Qwen-toolaanroepen worden als tekst weergegeven">
    Controleer eerst of vLLM is gestart met de juiste parser voor toolaanroepen en de juiste chatsjabloon voor het model. vLLM documenteert `hermes` voor Qwen2.5-modellen en `qwen3_xml` voor Qwen3-Coder-modellen.

    Symptomen: skills/tools worden nooit uitgevoerd, de assistent geeft onbewerkte JSON/XML weer, zoals `{"name":"read","arguments":...}`, of vLLM retourneert een lege `tool_calls`-array wanneer OpenClaw `tool_choice: "auto"` verzendt.

    Sommige combinaties van Qwen en vLLM retourneren alleen gestructureerde toolaanroepen wanneer de aanvraag `tool_choice: "required"` gebruikt. Dwing dit per model af met `params.extra_body`:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    Vervang het model-ID door het exacte ID uit `openclaw models list --provider vllm` of pas dezelfde overschrijving toe via de CLI:

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    Dit is een optionele tijdelijke oplossing: elke beurt met tools wordt hierdoor gedwongen een toolaanroep te doen. Gebruik dit dus alleen voor een afzonderlijke modelvermelding waarbij dit acceptabel is. Stel dit niet in als algemene standaardwaarde voor alle vLLM-modellen en combineer het niet met een proxy die willekeurige assistenttekst omzet in uitvoerbare toolaanroepen.

  </Accordion>

  <Accordion title="Aangepaste basis-URL">
    Als je vLLM-server op een niet-standaardhost of -poort draait, stel je `baseUrl` in de expliciete providerconfiguratie in:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "Extern vLLM-model",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## Problemen oplossen

<AccordionGroup>
  <Accordion title="Langzame eerste reactie of time-out van externe server">
    Stel voor grote lokale modellen, externe LAN-hosts of tailnetverbindingen een time-out voor aanvragen op providerniveau in:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "Lokaal vLLM-model" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` is alleen van toepassing op HTTP-aanvragen voor vLLM-modellen: het opzetten van de verbinding, antwoordheaders, het streamen van de body en de totale afbreking van de beveiligde fetch. Dit verhoogt ook de maximale LLM-inactiviteits-/streamwatchdogtijd tot boven de impliciete standaardwaarde van ~120s voor deze provider. Geef hieraan de voorkeur boven het verhogen van `agents.defaults.timeoutSeconds`, dat de volledige agentuitvoering regelt.

  </Accordion>

  <Accordion title="Server niet bereikbaar">
    Controleer of de vLLM-server actief en toegankelijk is:

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    Als je een verbindingsfout ziet, controleer dan de host en poort en of vLLM in de OpenAI-compatibele servermodus is gestart. OpenClaw vertrouwt voor beveiligde modelaanvragen op loopback-, LAN- en Tailscale-eindpunten de exact geconfigureerde `models.providers.vllm.baseUrl`-oorsprong. Metadata- en link-local-oorsprongen blijven zonder expliciete inschakeling geblokkeerd. Stel `models.providers.vllm.request.allowPrivateNetwork: true` alleen in wanneer vLLM-aanvragen een andere privé-oorsprong moeten bereiken, of `false` om het vertrouwen van de exacte oorsprong uit te schakelen.

  </Accordion>

  <Accordion title="Authenticatiefouten bij aanvragen">
    Als aanvragen mislukken met authenticatiefouten, stel je een echte `VLLM_API_KEY` in die overeenkomt met je serverconfiguratie of configureer je de provider expliciet onder `models.providers.vllm`.

    <Tip>
    Als je vLLM-server geen authenticatie afdwingt, werkt elke niet-lege waarde voor `VLLM_API_KEY` als inschakelsignaal voor OpenClaw.
    </Tip>

  </Accordion>

  <Accordion title="Geen modellen gedetecteerd">
    Voor automatische detectie moet `VLLM_API_KEY` zijn ingesteld. Als je `models.providers.vllm` hebt gedefinieerd, gebruikt OpenClaw alleen de modellen die je hebt gedeclareerd, tenzij `agents.defaults.models` ook `"vllm/*": {}` bevat.
  </Accordion>

  <Accordion title="Tools worden als onbewerkte tekst weergegeven">
    Als een Qwen-model JSON/XML-toolsyntaxis weergeeft in plaats van een skill uit te voeren:

    - Start vLLM met de juiste parser/sjabloon voor dat model.
    - Controleer het exacte model-ID met `openclaw models list --provider vllm`.
    - Voeg alleen een afzonderlijke `params.extra_body.tool_choice: "required"`-overschrijving per model toe als `tool_choice: "auto"` nog steeds lege toolaanroepen of toolaanroepen met alleen tekst retourneert.

  </Accordion>
</AccordionGroup>

<Warning>
Meer hulp: [Problemen oplossen](/nl/help/troubleshooting) en [Veelgestelde vragen](/nl/help/faq).
</Warning>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelverwijzingen en failovergedrag kiezen.
  </Card>
  <Card title="OpenAI" href="/nl/providers/openai" icon="bolt">
    Native OpenAI-provider en gedrag van OpenAI-compatibele routes.
  </Card>
  <Card title="OAuth en authenticatie" href="/nl/gateway/authentication" icon="key">
    Authenticatiegegevens en regels voor het hergebruik van aanmeldgegevens.
  </Card>
  <Card title="Problemen oplossen" href="/nl/help/troubleshooting" icon="wrench">
    Veelvoorkomende problemen en hoe je ze oplost.
  </Card>
</CardGroup>
