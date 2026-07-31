---
read_when:
    - Je automatiseert de onboarding in scripts of CI
    - Je hebt niet-interactieve voorbeelden nodig voor specifieke providers
sidebarTitle: CLI automation
summary: Gescripte onboarding en agentconfiguratie voor de OpenClaw CLI
title: CLI-automatisering
x-i18n:
    generated_at: "2026-07-27T05:52:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2a9fd8530379927995641f8033651ff12ada98068f106672e6655a17b8265735
    source_path: start/wizard-cli-automation.md
    workflow: 16
---

Gebruik `openclaw onboard --non-interactive` voor installatie via scripts. Hiervoor is `--accept-risk` vereist: een niet-interactieve installatie kan referenties en daemonconfiguratie zonder bevestigingsprompt schrijven, dus de vlag is de expliciete erkenning van het risico.

<Note>
`--json` impliceert geen niet-interactieve modus. Geef `--non-interactive --accept-risk` expliciet door voor scripts.
</Note>

## Niet-interactief basisvoorbeeld

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode plaintext \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-bootstrap \
  --skip-skills
```

Voeg `--json` toe voor een machineleesbare samenvatting.

- `--gateway-port` is standaard ingesteld op `18789`; geef dit alleen door om de standaardwaarde te overschrijven.
- `--skip-bootstrap` slaat het aanmaken van standaardwerkruimtebestanden over voor automatisering die vooraf een eigen werkruimte vult.
- `--secret-input-mode ref` slaat een door een omgevingsvariabele ondersteunde verwijzing (`{ source: "env", provider: "default", id: "<ENV_VAR>" }`) op in het authenticatieprofiel in plaats van de sleutel als platte tekst. In de niet-interactieve `ref`-modus moet de omgevingsvariabele van de provider al in de procesomgeving zijn ingesteld: het doorgeven van een inline sleutelvlag zonder de bijbehorende omgevingsvariabele mislukt onmiddellijk.

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref
```

## Providerspecifieke voorbeelden

<AccordionGroup>
  <Accordion title="Voorbeeld met een Anthropic API-sleutel">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice apiKey \
      --anthropic-api-key "$ANTHROPIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Cloudflare AI Gateway">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice cloudflare-ai-gateway-api-key \
      --cloudflare-ai-gateway-account-id "your-account-id" \
      --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
      --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Gemini">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice gemini-api-key \
      --gemini-api-key "$GEMINI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Mistral">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice mistral-api-key \
      --mistral-api-key "$MISTRAL_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Moonshot">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice moonshot-api-key \
      --moonshot-api-key "$MOONSHOT_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Ollama">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ollama \
      --custom-model-id "qwen3.5:27b" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met OpenCode">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice opencode-zen \
      --opencode-zen-api-key "$OPENCODE_API_KEY" \
      --gateway-bind loopback
    ```
    Schakel over naar `--auth-choice opencode-go --opencode-go-api-key "$OPENCODE_API_KEY"` voor de Go-catalogus.
  </Accordion>
  <Accordion title="Voorbeeld met Synthetic">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice synthetic-api-key \
      --synthetic-api-key "$SYNTHETIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Vercel AI Gateway">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ai-gateway-api-key \
      --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met Z.AI">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice zai-api-key \
      --zai-api-key "$ZAI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Voorbeeld met een aangepaste provider">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --custom-api-key "$CUSTOM_API_KEY" \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

    `--custom-api-key` is optioneel; sommige eindpunten vereisen geen authenticatie. Als dit wordt weggelaten, controleert het onboardingproces `CUSTOM_API_KEY` in de omgeving. `--custom-provider-id` is optioneel en wordt bij weglating automatisch afgeleid van de basis-URL. `--custom-compatibility` is standaard ingesteld op `openai` (andere waarden: `openai-responses`, `anthropic`).

    OpenClaw leidt ondersteuning voor afbeeldingsinvoer af uit bekende patronen voor vision-model-id's (`gpt-4o`, `claude-3/4`, `gemini`, achtervoegsels `-vl`/`vision` en vergelijkbare patronen). Voeg `--custom-image-input` toe om dit voor een niet-herkend vision-model geforceerd in te schakelen, of `--custom-text-input` om alleen tekst af te dwingen.

    Variant in verwijzingsmodus, waarbij `apiKey` als `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }` wordt opgeslagen:

    ```bash
    export CUSTOM_API_KEY="your-key"
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --secret-input-mode ref \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

  </Accordion>
</AccordionGroup>

Authenticatie met een Anthropic-installatietoken blijft ondersteund, maar OpenClaw geeft de voorkeur aan hergebruik van de Claude CLI wanneer lokaal een Claude CLI-aanmelding beschikbaar is. Geef voor productie de voorkeur aan een Anthropic API-sleutel.

## Nog een agent toevoegen

`openclaw agents add <name>` maakt een afzonderlijke agent met een eigen werkruimte, sessies en authenticatieprofielen. Als je dit uitvoert zonder `--workspace` (en zonder andere vlaggen), wordt de interactieve wizard gestart; als je een van `--workspace`, `--model`, `--agent-dir`, `--bind` of `--non-interactive` doorgeeft, wordt de opdracht niet-interactief uitgevoerd en is vervolgens `--workspace` vereist.

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

Configuratiesleutels die worden geschreven (`agents.entries.*`-item voor de nieuwe agent-id):

- `name`
- `workspace`
- `agentDir`
- `model` (alleen wanneer `--model` wordt doorgegeven)

Opmerkingen:

- Standaardwerkruimte (wanneer `--workspace` in de interactieve wizard wordt weggelaten): `~/.openclaw/workspace-<agentId>`.
- `--bind <channel[:accountId]>` kan worden herhaald; voeg bindingen toe om inkomende berichten naar de nieuwe agent te routeren (dit kan ook interactief via de wizard).
- De agentnaam wordt genormaliseerd tot een geldige agent-id; `main` is gereserveerd.

## Gerelateerde documentatie

- Onboardinghub: [Onboarding (CLI)](/nl/start/wizard)
- Volledige referentie: [CLI-installatiereferentie](/nl/start/wizard-cli-reference)
- Opdrachtenreferentie: [`openclaw onboard`](/nl/cli/onboard)
