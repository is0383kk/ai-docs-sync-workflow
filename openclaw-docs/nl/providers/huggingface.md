---
read_when:
    - Je wilt Hugging Face Inference met OpenClaw gebruiken
    - Je hebt de omgevingsvariabele voor het HF-token of de CLI-authenticatiekeuze nodig
summary: Hugging Face Inference instellen (authenticatie + modelselectie)
title: Hugging Face (inferentie)
x-i18n:
    generated_at: "2026-07-27T05:19:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 92c400b78c5ad2cc724ad4029560dccc5bc2006fdeae400fc6b58998e727e17c
    source_path: providers/huggingface.md
    workflow: 16
---

[Hugging Face Inference Providers](https://huggingface.co/docs/inference-providers) biedt een OpenAI-compatibele router voor chatvoltooiingen vóór veel gehoste modellen (DeepSeek, Llama en meer) onder één token. OpenClaw communiceert alleen met het **eindpunt voor chatvoltooiingen**; gebruik voor tekst-naar-afbeelding, embeddings of spraak rechtstreeks de [HF-inferenceclients](https://huggingface.co/docs/api-inference/quicktour).

| Eigenschap    | Waarde                                                                                                                      |
| ------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Provider-id   | `huggingface`                                                                                                          |
| Plugin        | gebundeld (standaard ingeschakeld, geen installatiestap)                                                                    |
| Auth-omgevingsvariabele | `HUGGINGFACE_HUB_TOKEN` of `HF_TOKEN` (fijnmazig token)                                                        |
| API           | OpenAI-compatibel (`https://router.huggingface.co/v1`)                                                                                       |
| Facturering   | Eén HF-token; [prijzen](https://huggingface.co/docs/inference-providers/pricing) volgen de tarieven van de provider met een gratis niveau |

## Aan de slag

<Steps>
  <Step title="Een fijnmazig token maken">
    Ga naar [Hugging Face Settings Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) en maak een nieuw fijnmazig token.

    <Warning>
    Voor het token moet de machtiging **Make calls to Inference Providers** zijn ingeschakeld, anders worden API-aanvragen geweigerd.
    </Warning>

  </Step>
  <Step title="Onboarding uitvoeren">
    Kies **Hugging Face** in de providerkeuzelijst en voer vervolgens je API-sleutel in wanneer daarom wordt gevraagd:

    ```bash
    openclaw onboard --auth-choice huggingface-api-key
    ```

  </Step>
  <Step title="Een standaardmodel selecteren">
    Kies een model in de keuzelijst **Standaardmodel van Hugging Face**. De lijst wordt vanuit de Inference API geladen wanneer je token geldig is; anders toont OpenClaw de ingebouwde catalogus hieronder. Je keuze wordt opgeslagen als `agents.defaults.model.primary`:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
        },
      },
    }
    ```

  </Step>
  <Step title="Controleren of het model beschikbaar is">
    ```bash
    openclaw models list --provider huggingface
    ```
  </Step>
</Steps>

### Niet-interactieve configuratie

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

Stelt `huggingface/deepseek-ai/DeepSeek-R1` in als standaardmodel.

## Model-ID's

Modelverwijzingen gebruiken de vorm `huggingface/<org>/<model>` (ID's in Hub-stijl). De ingebouwde catalogus van OpenClaw:

| Model         | Verwijzing (voorvoegen met `huggingface/`) |
| ------------- | -------------------------------- |
| DeepSeek R1   | `deepseek-ai/DeepSeek-R1`        |
| DeepSeek V3.1 | `deepseek-ai/DeepSeek-V3.1`      |
| GPT-OSS 120B  | `openai/gpt-oss-120b`            |

<Tip>
Wanneer je token geldig is, detecteert OpenClaw tijdens de onboarding en bij het starten van de Gateway ook alle andere modellen via **GET** `https://router.huggingface.co/v1/models`, zodat je catalogus veel meer dan de drie bovenstaande modellen kan bevatten. Je kunt `:fastest` of `:cheapest` aan elk model-id toevoegen; de router van HF routeert naar de overeenkomende inferenceprovider. Stel je standaardvolgorde van providers in bij de [instellingen voor Inference Providers](https://hf.co/settings/inference-providers).
</Tip>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Modeldetectie en onboardingkeuzelijst">
    OpenClaw detecteert modellen met:

    ```bash
    GET https://router.huggingface.co/v1/models
    Authorization: Bearer $HUGGINGFACE_HUB_TOKEN   # of $HF_TOKEN
    ```

    Het antwoord heeft de OpenAI-stijl: `{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }`.

    Met een geconfigureerde sleutel (onboarding, `HUGGINGFACE_HUB_TOKEN` of `HF_TOKEN`) wordt de keuzelijst **Standaardmodel van Hugging Face** tijdens de interactieve configuratie vanuit dit eindpunt gevuld. Bij het starten van de Gateway wordt dezelfde aanroep herhaald om de catalogus te vernieuwen. Gedetecteerde modellen worden samengevoegd met de ingebouwde catalogus hierboven (die wordt gebruikt voor metagegevens zoals het contextvenster en de kosten wanneer een id overeenkomt). Als de aanvraag mislukt, geen gegevens retourneert of er geen sleutel is ingesteld, valt OpenClaw uitsluitend terug op de ingebouwde catalogus.

    Detectie uitschakelen zonder de provider te verwijderen:

    ```bash
    openclaw config set plugins.entries.huggingface.config.discovery.enabled false
    ```

  </Accordion>

  <Accordion title="Modelnamen, aliassen en beleidsachtervoegsels">
    - **Naam uit de API:** gedetecteerde modellen gebruiken indien aanwezig de `name`, `title` of `display_name` van de API; anders leidt OpenClaw een naam af van het model-id (bijvoorbeeld `deepseek-ai/DeepSeek-R1` wordt "DeepSeek R1").
    - **Weergavenaam overschrijven:** stel per model een aangepast label in de configuratie in:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
          },
        },
      },
    }
    ```

    - **Beleidsachtervoegsels:** `:fastest` en `:cheapest` zijn conventies van de HF-router, niet iets wat OpenClaw herschrijft: het achtervoegsel wordt letterlijk als onderdeel van het model-id verzonden en de router van HF kiest de overeenkomende inferenceprovider. Voeg elke variant als afzonderlijke vermelding toe onder `models.providers.huggingface.models` (of in `model.primary`) als je per achtervoegsel een afzonderlijke alias wilt.
    - **Configuratiesamenvoeging:** bestaande vermeldingen in `models.providers.huggingface.models` (bijvoorbeeld in `models.json`) blijven bij het samenvoegen van de configuratie behouden, zodat aangepaste `name`, `alias` of modelopties die je daar instelt ook na herstarts behouden blijven.

  </Accordion>

  <Accordion title="Omgevings- en daemonconfiguratie">
    Als de Gateway als daemon draait (launchd/systemd), zorg er dan voor dat `HUGGINGFACE_HUB_TOKEN` of `HF_TOKEN` beschikbaar is voor dat proces (bijvoorbeeld in `~/.openclaw/.env` of via `env.shellEnv`).

    <Note>
    OpenClaw accepteert zowel `HUGGINGFACE_HUB_TOKEN` als `HF_TOKEN`. Als beide zijn ingesteld, heeft `HUGGINGFACE_HUB_TOKEN` voorrang.
    </Note>

  </Accordion>

  <Accordion title="Configuratie: DeepSeek R1 met terugvalmodel">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-R1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Configuratie: DeepSeek met goedkoopste en snelste varianten">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheapest)" },
            "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fastest)" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Configuratie: DeepSeek + GPT-OSS met aliassen">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-V3.1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-V3.1": { alias: "DeepSeek V3.1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Overzicht van alle providers, modelverwijzingen en failovergedrag.
  </Card>
  <Card title="Modelselectie" href="/nl/concepts/models" icon="brain">
    Modellen kiezen en configureren.
  </Card>
  <Card title="Documentatie voor Inference Providers" href="https://huggingface.co/docs/inference-providers" icon="book">
    Officiële documentatie voor Hugging Face Inference Providers.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration" icon="gear">
    Volledige configuratiereferentie.
  </Card>
</CardGroup>
