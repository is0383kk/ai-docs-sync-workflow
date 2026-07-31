---
read_when:
    - Je wilt privacygerichte inferentie in OpenClaw
    - Je wilt installatie-instructies voor Venice AI
summary: Gebruik de privacygerichte modellen van Venice AI in OpenClaw
title: Venice AI
x-i18n:
    generated_at: "2026-07-27T06:10:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 13c32b783394eb3092ff94a532b69e34c00624127b0e76e4e2812751d39073a1
    source_path: providers/venice.md
    workflow: 16
---

[Venice AI](https://venice.ai) biedt privacygerichte inferentie: open modellen worden uitgevoerd
zonder logging, plus geanonimiseerde proxytoegang tot Claude, GPT, Gemini en Grok.
Alle eindpunten zijn OpenAI-compatibel (`/v1`).

## Privacymodi

| Modus              | Gedrag                                                                      | Modellen                                                      |
| ------------------ | --------------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Privé**          | Prompts/antwoorden worden nooit opgeslagen of gelogd. Tijdelijk.            | Llama, Qwen, DeepSeek, Kimi, MiniMax, Venice Uncensored, enz. |
| **Geanonimiseerd** | Via Venice doorgestuurd, waarbij metadata vóór het doorsturen wordt gewist. | Claude, GPT, Gemini, Grok                                     |

<Warning>
Geanonimiseerde modellen zijn niet volledig privé. Venice verwijdert metadata vóór het doorsturen, maar de onderliggende provider (OpenAI, Anthropic, Google, xAI) verwerkt het verzoek nog steeds. Gebruik privémodellen wanneer volledige privacy vereist is.
</Warning>

## Aan de slag

<Steps>
  <Step title="Installeer de plugin">
    ```bash
    openclaw plugins install @openclaw/venice-provider
    ```
  </Step>
  <Step title="Verkrijg je API-sleutel">
    1. Meld je aan bij [venice.ai](https://venice.ai)
    2. Ga naar **Settings > API Keys > Create new key**
    3. Kopieer je API-sleutel (indeling: `vapi_xxxxxxxxxxxx`)
  </Step>
  <Step title="Configureer OpenClaw">
    <Tabs>
      <Tab title="Interactief (aanbevolen)">
        ```bash
        openclaw onboard --auth-choice venice-api-key
        ```

        Vraagt om de API-sleutel (of hergebruikt een bestaande `VENICE_API_KEY`), toont de beschikbare Venice-modellen en stelt je standaardmodel in.
      </Tab>
      <Tab title="Omgevingsvariabele">
        ```bash
        export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
        ```
      </Tab>
      <Tab title="Niet-interactief">
        ```bash
        openclaw onboard --non-interactive \
          --auth-choice venice-api-key \
          --venice-api-key "vapi_xxxxxxxxxxxx"
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="Controleer de configuratie">
    ```bash
    openclaw agent --model venice/kimi-k2-5 --message "Hallo, werk je?"
    ```
  </Step>
</Steps>

## Modelselectie

- **Standaard**: `venice/kimi-k2-5` (privé, redeneren, beeld).
- **Sterkste geanonimiseerde optie**: `venice/claude-opus-4-6`.

```bash
openclaw models set venice/kimi-k2-5
openclaw models list --all --provider venice
```

Je kunt ook `openclaw configure` uitvoeren en **Model/auth provider > Venice AI** kiezen.

<Tip>
| Gebruikssituatie        | Model                                        | Waarom                                        |
| ----------------------- | -------------------------------------------- | --------------------------------------------- |
| Algemene chat (standaard) | `kimi-k2-5`                         | Sterk privé redeneervermogen plus beeld       |
| Beste algehele kwaliteit  | `claude-opus-4-6`                         | Sterkste geanonimiseerde Venice-optie         |
| Privacy + programmeren    | `qwen3-coder-480b-a35b-instruct-turbo`                         | Privéprogrammeermodel met grote context       |
| Snel + goedkoop           | `llama-3.2-3b`                         | Compact privémodel                            |
| Complexe privétaken       | `deepseek-v3.2`                         | Sterk redeneervermogen; toolaanroepen uitgeschakeld |
| Ongecensureerd            | `venice-uncensored-1-2`                         | Huidig ongecensureerd Venice-model            |
</Tip>

## Ingebouwde catalogus (30 modellen)

<AccordionGroup>
  <Accordion title="Privémodellen (20) — volledig privé, geen logging">
    | Model-ID                               | Naam                                  | Context | Opmerkingen                       |
    | -------------------------------------- | ------------------------------------- | ------- | --------------------------------- |
    | `kimi-k2-5`                    | Kimi K2.5                             | 256k    | Standaard, redeneren, beeld       |
    | `llama-3.3-70b`                    | Llama 3.3 70B                         | 128k    | Algemeen                          |
    | `llama-3.2-3b`                    | Llama 3.2 3B                          | 128k    | Algemeen                          |
    | `hermes-3-llama-3.1-405b`                    | Hermes 3 Llama 3.1 405B               | 128k    | Algemeen, tools uitgeschakeld     |
    | `qwen3-235b-a22b-thinking-2507`                    | Qwen3 235B Thinking                   | 128k    | Redeneren                         |
    | `qwen3-235b-a22b-instruct-2507`                    | Qwen3 235B Instruct                   | 128k    | Algemeen                          |
    | `qwen3-coder-480b-a35b-instruct-turbo`                    | Qwen3 Coder 480B Turbo                | 256k    | Programmeren                      |
    | `qwen3-5-35b-a3b`                    | Qwen3.5 35B A3B                       | 256k    | Redeneren, beeld                  |
    | `qwen3-next-80b`                    | Qwen3 Next 80B                        | 256k    | Algemeen                          |
    | `qwen3-vl-235b-a22b`                    | Qwen3 VL 235B (Vision)                | 256k    | Beeld                             |
    | `deepseek-v3.2`                    | DeepSeek V3.2                         | 160k    | Redeneren, tools uitgeschakeld    |
    | `google-gemma-3-27b-it`                    | Google Gemma 3 27B Instruct           | 198k    | Beeld                             |
    | `openai-gpt-oss-120b`                    | OpenAI GPT OSS 120B                   | 128k    | Algemeen                          |
    | `nvidia-nemotron-3-nano-30b-a3b`                    | NVIDIA Nemotron 3 Nano 30B            | 128k    | Algemeen                          |
    | `olafangensan-glm-4.7-flash-heretic`                    | GLM 4.7 Flash Heretic                 | 128k    | Redeneren                         |
    | `zai-org-glm-4.6`                    | GLM 4.6                               | 198k    | Algemeen                          |
    | `zai-org-glm-4.7`                    | GLM 4.7                               | 198k    | Redeneren                         |
    | `zai-org-glm-4.7-flash`                    | GLM 4.7 Flash                         | 128k    | Redeneren                         |
    | `zai-org-glm-5`                    | GLM 5                                 | 198k    | Redeneren                         |
    | `minimax-m25`                    | MiniMax M2.5                          | 198k    | Redeneren                         |
  </Accordion>

  <Accordion title="Geanonimiseerde modellen (10) — via de Venice-proxy">
    | Model-ID                        | Naam                            | Context | Opmerkingen                    |
    | ------------------------------- | ------------------------------- | ------- | ------------------------------ |
    | `claude-opus-4-6`              | Claude Opus 4.6 (via Venice)    | 1M      | Redeneren, beeld               |
    | `claude-sonnet-4-6`              | Claude Sonnet 4.6 (via Venice)  | 1M      | Redeneren, beeld               |
    | `openai-gpt-54`              | GPT-5.4 (via Venice)            | 1M      | Redeneren, beeld               |
    | `openai-gpt-53-codex`              | GPT-5.3 Codex (via Venice)      | 400k    | Redeneren, beeld, programmeren |
    | `openai-gpt-52`              | GPT-5.2 (via Venice)            | 256k    | Redeneren                      |
    | `openai-gpt-52-codex`              | GPT-5.2 Codex (via Venice)      | 256k    | Redeneren, beeld, programmeren |
    | `openai-gpt-4o-2024-11-20`              | GPT-4o (via Venice)             | 128k    | Beeld                          |
    | `openai-gpt-4o-mini-2024-07-18`              | GPT-4o Mini (via Venice)        | 128k    | Beeld                          |
    | `gemini-3-1-pro-preview`              | Gemini 3.1 Pro (via Venice)     | 1M      | Redeneren, beeld               |
    | `gemini-3-flash-preview`              | Gemini 3 Flash (via Venice)     | 256k    | Redeneren, beeld               |
  </Accordion>
</AccordionGroup>

Door Grok ondersteunde Venice-modellen (`grok-4-3` en vergelijkbare modellen) krijgen dezelfde
compatibiliteitspatch voor het toolschema als de systeemeigen xAI-provider, omdat ze dezelfde bovenliggende
indeling voor toolaanroepen gebruiken.

## Modeldetectie

De gebundelde catalogus hierboven is een door een manifest ondersteunde basislijst. Tijdens runtime vernieuwt OpenClaw
deze via de Venice-API `/models` en valt het terug op de basislijst als
de API onbereikbaar is. Het eindpunt `/models` is openbaar (geen authenticatie nodig voor
het weergeven van de lijst), maar inferentie vereist een geldige API-sleutel.

Venice kan uitgefaseerde model-ID's blijven accepteren als door de provider beheerde aliassen. De
OpenClaw-catalogus toont alleen de canonieke model-ID's die door `/models` worden geretourneerd.

## Replaygedrag van DeepSeek V4

Als Venice DeepSeek V4-modellen beschikbaar stelt, zoals `deepseek-v4-pro` of
`deepseek-v4-flash`, vult OpenClaw het vereiste replayveld `reasoning_content`
in assistentberichten in wanneer Venice dit weglaat, en verwijdert het `thinking`/
`reasoning`/`reasoning_effort` uit de aanvraagpayload (Venice weigert
het systeemeigen besturingselement `thinking` van DeepSeek voor deze modellen). Deze replayoplossing staat
los van de eigen denkbesturing van de systeemeigen DeepSeek-provider.

## Ondersteuning voor streaming en tools

| Functie          | Ondersteuning                                             |
| ---------------- | --------------------------------------------------------- |
| Streaming        | Alle modellen                                             |
| Functieaanroepen | De meeste modellen; per model uitgeschakeld waar hierboven vermeld |
| Beeld/afbeeldingen | Modellen die hierboven als "Beeld" zijn gemarkeerd      |
| JSON-modus       | Via `response_format`                                    |

## Prijzen

Venice gebruikt een systeem op basis van tegoeden. Geanonimiseerde modellen kosten ongeveer hetzelfde als
directe API-prijzen plus een kleine Venice-toeslag. Zie
[venice.ai/prijzen](https://venice.ai/pricing) voor de actuele tarieven.

## Gebruiksvoorbeelden

```bash
# Standaard privémodel
openclaw agent --model venice/kimi-k2-5 --message "Snelle statuscontrole"

# Claude Opus via Venice (geanonimiseerd)
openclaw agent --model venice/claude-opus-4-6 --message "Vat deze taak samen"

# Ongecensureerd model
openclaw agent --model venice/venice-uncensored-1-2 --message "Stel opties op"

# Beeldmodel met afbeelding
openclaw agent --model venice/qwen3-vl-235b-a22b --message "Beoordeel de bijgevoegde afbeelding"

# Programmeermodel
openclaw agent --model venice/qwen3-coder-480b-a35b-instruct-turbo --message "Herstructureer deze functie"
```

## Problemen oplossen

<AccordionGroup>
  <Accordion title="API-sleutel wordt niet herkend">
    ```bash
    echo $VENICE_API_KEY
    openclaw models list | grep venice
    ```

    Controleer of de sleutel begint met `vapi_`.

  </Accordion>

  <Accordion title="Model niet beschikbaar">
    Voer `openclaw models list --all --provider venice` uit om de momenteel
    beschikbare modellen te bekijken; de catalogus verandert wanneer Venice modellen toevoegt of uitfaseert.
  </Accordion>

  <Accordion title="Verbindingsproblemen">
    De Venice-API bevindt zich op `https://api.venice.ai/api/v1`. Controleer of je netwerk HTTPS naar die host toestaat.
  </Accordion>
</AccordionGroup>

<Note>
Meer hulp: [Problemen oplossen](/nl/help/troubleshooting) en [Veelgestelde vragen](/nl/help/faq).
</Note>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Voorbeeld van configuratiebestand">
    ```json5
    {
      env: { VENICE_API_KEY: "vapi_..." },
      agents: { defaults: { model: { primary: "venice/kimi-k2-5" } } },
      models: {
        mode: "merge",
        providers: {
          venice: {
            baseUrl: "https://api.venice.ai/api/v1",
            apiKey: "${VENICE_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2-5",
                name: "Kimi K2.5",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 256000,
                maxTokens: 65536,
              },
            ],
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
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Venice AI" href="https://venice.ai" icon="globe">
    Homepage van Venice AI en accountregistratie.
  </Card>
  <Card title="API-documentatie" href="https://docs.venice.ai" icon="book">
    Venice API-referentie en ontwikkelaarsdocumentatie.
  </Card>
  <Card title="Prijzen" href="https://venice.ai/pricing" icon="credit-card">
    Huidige Venice-krediettarieven en abonnementen.
  </Card>
</CardGroup>
