---
read_when:
    - Je wilt Fireworks met OpenClaw gebruiken
    - Je hebt de omgevingsvariabele voor de Fireworks-API-sleutel of de standaardmodel-id nodig
    - Je debugt het gedrag van Kimi met uitgeschakeld denkproces op Fireworks
summary: Fireworks-configuratie (authenticatie + modelselectie)
title: Fireworks
x-i18n:
    generated_at: "2026-07-27T05:30:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7720b23b69aa716d2e2903f5644bb74f81ca1c5e753f71d72d4d7a25c0747884
    source_path: providers/fireworks.md
    workflow: 16
---

[Fireworks](https://fireworks.ai) biedt open-weight- en gerouteerde modellen aan via een OpenAI-compatibele API. Installeer de officiële Fireworks-providerplugin om tijdens runtime twee vooraf gecatalogiseerde Kimi-modellen en elk Fireworks-model of elke router-id te gebruiken.

| Eigenschap      | Waarde                                                 |
| --------------- | ------------------------------------------------------ |
| Provider-id     | `fireworks` (alias: `fireworks-ai`)                    |
| Pakket          | `@openclaw/fireworks-provider`                         |
| Omgevingsvariabele voor authenticatie | `FIREWORKS_API_KEY`                                    |
| Onboarding-vlag | `--auth-choice fireworks-api-key`                      |
| Directe CLI-vlag | `--fireworks-api-key <key>`                            |
| API             | OpenAI-compatibel (`openai-completions`)               |
| Basis-URL       | `https://api.fireworks.ai/inference/v1`                |
| Standaardmodel  | `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` |
| Standaardalias  | `Kimi K2.6 Turbo`                                      |

## Aan de slag

<Steps>
  <Step title="Installeer de plugin">
    ```bash
    openclaw plugins install @openclaw/fireworks-provider
    ```
  </Step>
  <Step title="Stel de Fireworks-API-sleutel in">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice fireworks-api-key
```

```bash Directe vlag
openclaw onboard --non-interactive \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY"
```

```bash Alleen omgeving
export FIREWORKS_API_KEY=fw-...
```

    </CodeGroup>

    Onboarding slaat de sleutel voor de provider `fireworks` op in je authenticatieprofielen en stelt de Kimi K2.6 Turbo-router **Fire Pass** in als standaardmodel.

  </Step>
  <Step title="Controleer of het model beschikbaar is">
    ```bash
    openclaw models list --provider fireworks
    ```

    De lijst moet `Kimi K2.6` en `Kimi K2.6 Turbo (Fire Pass)` bevatten. Als `FIREWORKS_API_KEY` niet kan worden omgezet, meldt `openclaw models status --json` de ontbrekende referentie onder `auth.unusableProfiles`.

  </Step>
</Steps>

## Niet-interactieve configuratie

Geef voor gescripte installaties of CI-installaties alles op via de opdrachtregel:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## Ingebouwde catalogus

| Modelreferentie                                        | Naam                        | Invoer       | Context | Maximale uitvoer | Thinking             |
| ------------------------------------------------------ | --------------------------- | ------------ | ------- | ---------------- | -------------------- |
| `fireworks/accounts/fireworks/models/kimi-k2p6`        | Kimi K2.6                   | tekst + afbeelding | 262,144 | 262,144    | Gedwongen uit        |
| `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` | Kimi K2.6 Turbo (Fire Pass) | tekst + afbeelding | 256,000 | 256,000    | Gedwongen uit (standaard) |

<Note>
  OpenClaw zet alle Fireworks Kimi-modellen vast op `thinking: off`, omdat Kimi op Fireworks de redeneerketen in het zichtbare antwoord kan laten uitlekken, tenzij het verzoek Thinking expliciet uitschakelt. Als hetzelfde model rechtstreeks via [Moonshot](/nl/providers/moonshot) wordt gerouteerd, blijft de redeneeruitvoer van Kimi behouden. Zie [Thinking-modi](/nl/tools/thinking) voor het wisselen tussen providers.
</Note>

## Aangepaste Fireworks-model-id's

OpenClaw accepteert tijdens runtime elke Fireworks-model- of router-id. Gebruik de exacte id die Fireworks toont en voeg het voorvoegsel `fireworks/` toe. Dynamische omzetting kloont de Fire Pass-sjabloon (tekst- en afbeeldingsinvoer, OpenAI-compatibele API, standaardkosten nul) en schakelt Thinking automatisch uit wanneer de id overeenkomt met het Kimi-patroon. Dynamische GLM-id's worden gemarkeerd als uitsluitend tekst, tenzij je een aangepaste modelvermelding met afbeeldingsinvoer configureert.

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/models/<your-model-id>",
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Hoe het voorvoegsel voor model-id's werkt">
    Elke Fireworks-modelreferentie in OpenClaw begint met `fireworks/`, gevolgd door de exacte id of het routerpad van het Fireworks-platform. Bijvoorbeeld:

    - Routermodel: `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo`
    - Rechtstreeks model: `fireworks/accounts/fireworks/models/<model-name>`

    OpenClaw verwijdert het voorvoegsel `fireworks/` bij het samenstellen van het API-verzoek en stuurt het resterende pad als het OpenAI-compatibele veld `model` naar het Fireworks-eindpunt.

  </Accordion>

  <Accordion title="Waarom Thinking voor Kimi gedwongen wordt uitgeschakeld">
    Fireworks levert Kimi zonder afzonderlijk redeneerkanaal, waardoor de redeneerketen zichtbaar kan worden in de `content`-stream. Bij elk Fireworks Kimi-verzoek verzendt OpenClaw `thinking: { type: "disabled" }` en verwijdert het `reasoning`, `reasoning_effort` en `reasoningEffort` uit de payload (`extensions/fireworks/stream.ts`). Het providerbeleid (`extensions/fireworks/thinking-policy.ts`) biedt voor Kimi-model-id's alleen het Thinking-niveau `off` aan, zodat handmatige `/think`-schakelaars en providerbeleidsonderdelen afgestemd blijven op het runtimecontract.

    Om Kimi-redenering van begin tot eind te gebruiken, configureer je de [Moonshot-provider](/nl/providers/moonshot) en routeer je hetzelfde model via deze provider.

  </Accordion>

  <Accordion title="Beschikbaarheid van de omgeving voor de daemon">
    Als de Gateway als beheerde service wordt uitgevoerd (launchd, systemd, Docker), moet de Fireworks-sleutel zichtbaar zijn voor dat proces, niet alleen voor je interactieve shell.

    <Warning>
      Een sleutel die alleen in een interactieve shell is geëxporteerd, helpt een launchd- of systemd-daemon niet, tenzij die omgeving daar ook wordt geïmporteerd. Stel de sleutel in via `~/.openclaw/.env` of `env.shellEnv`, zodat het gatewayproces deze kan lezen.
    </Warning>

    OpenClaw laadt `~/.openclaw/.env` wanneer de configuratie wordt geladen, zodat sleutels die daar zijn opgeslagen op elk platform de beheerde gatewayservices bereiken. Start de gateway opnieuw (of voer `openclaw doctor --fix` opnieuw uit) nadat je de sleutel hebt vervangen.

  </Accordion>
</AccordionGroup>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Providers, modelreferenties en failovergedrag kiezen.
  </Card>
  <Card title="Thinking-modi" href="/nl/tools/thinking" icon="brain">
    `/think`-niveaus, providerbeleid en het routeren van modellen met redeneermogelijkheden.
  </Card>
  <Card title="Moonshot" href="/nl/providers/moonshot" icon="moon">
    Voer Kimi met native Thinking-uitvoer uit via de eigen API van Moonshot.
  </Card>
  <Card title="Probleemoplossing" href="/nl/help/troubleshooting" icon="wrench">
    Algemene probleemoplossing en veelgestelde vragen.
  </Card>
</CardGroup>
