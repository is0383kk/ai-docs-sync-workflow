---
read_when:
    - Je installeert, configureert of controleert de microsoft-foundry-plugin
summary: Voegt ondersteuning voor de Microsoft Foundry-modelprovider toe aan OpenClaw.
title: Microsoft Foundry-plugin
x-i18n:
    generated_at: "2026-07-27T06:28:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f2ea554ce16cffeb4cc315e53d986d6f07b5e113fbb844c61c6575f19f8ad291
    source_path: plugins/reference/microsoft-foundry.md
    workflow: 16
---

# Microsoft Foundry-plugin

Voegt ondersteuning voor de Microsoft Foundry-modelprovider toe aan OpenClaw.

## Distributie

- Pakket: `@openclaw/microsoft-foundry`
- Installatieroute: inbegrepen bij OpenClaw

## Oppervlak

providers: `microsoft-foundry`; contracten: `imageGenerationProviders`

<!-- openclaw-plugin-reference:manual-start -->

- Provider voor afbeeldingsgeneratie: `microsoft-foundry`

## Vereisten

- Een Microsoft Foundry- of Azure AI Foundry-resource met implementaties.
- Authenticatie met een API-sleutel via `AZURE_OPENAI_API_KEY` of een geconfigureerde API-sleutel voor de provider.
- Installeer voor Entra ID-authenticatie de Azure CLI en voer `az login` uit vóór
  de onboarding. OpenClaw vernieuwt Microsoft Foundry-runtimetokens via
  `az account get-access-token`.

## Chatmodellen

Chatimplementaties van Microsoft Foundry gebruiken de modelreferentie van de provider
`microsoft-foundry/<deployment-name>`. Tijdens de onboarding worden Foundry-resources
en -implementaties met de Azure CLI gedetecteerd, waarna de geselecteerde implementatienaam
naar de modelconfiguratie wordt geschreven.

OpenClaw gebruikt het Foundry-eindpunt `/openai/v1` voor ondersteunde OpenAI-compatibele
chat-API's:

- De modelfamilies GPT, `o*`, `computer-use-preview` en DeepSeek-V4 gebruiken standaard
  `openai-responses`.
- MAI-DS-R1 en andere chat-completion-implementaties gebruiken `openai-completions`,
  tenzij expliciet een ondersteunde API is geconfigureerd.
- MAI-DS-R1 wordt geregistreerd als geschikt voor redeneren via redeneerinhoud, niet
  via `reasoning_effort`. De metadata voor context- en uitvoertokens bedraagt
  163,840 tokens.

Implementaties van Anthropic Claude in Microsoft Foundry gebruiken de vorm van de Anthropic Messages-
API, niet de OpenAI-compatibele vorm `/openai/v1`. Configureer deze als een
aangepaste provider `anthropic-messages` totdat de Microsoft Foundry-plugin over een
eigen Anthropic-runtime beschikt. Wanneer de naam van de Foundry-implementatie afwijkt van de
Claude-model-ID, stel je `params.canonicalModelId` in bij de modelvermelding, zodat OpenClaw
modelspecifieke wire-contracten kan toepassen, `/think off` correct kan toewijzen en
ondertekende denkprocessen veilig kan behouden.

## MAI-afbeeldingsgeneratie

De Plugin registreert `microsoft-foundry` voor `image_generate` met de huidige
Microsoft AI-afbeeldingsmodellen:

- `MAI-Image-2.5-Flash`
- `MAI-Image-2.5`
- `MAI-Image-2e`
- `MAI-Image-2`

Gebruik de naam van een geïmplementeerde MAI-afbeeldingsimplementatie als modelreferentie. De provider
declareert geen standaardafbeeldingsmodel, omdat de MAI-API de naam van jouw implementatie
vereist in het aanvraagveld `model`:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "microsoft-foundry/<deployment-name>",
        timeoutMs: 600000,
      },
    },
  },
}
```

Generatie met alleen een prompt roept het MAI-generatie-eindpunt van Microsoft Foundry aan:
`/mai/v1/images/generations`. Bewerkingen met een referentieafbeelding roepen
`/mai/v1/images/edits` aan en zijn beperkt tot implementaties van `MAI-Image-2.5-Flash` en
`MAI-Image-2.5`.

Voor generatie met alleen een prompt kan een aangepaste implementatienaam worden gebruikt als alleen het Foundry-
eindpunt is geconfigureerd. Selecteer voor afbeeldingsbewerkingen met een aangepaste implementatienaam de
implementatie tijdens de onboarding of voeg modelmetadata toe, zodat OpenClaw kan verifiëren
dat de implementatie wordt ondersteund door `MAI-Image-2.5-Flash` of `MAI-Image-2.5`.

Beperkingen voor MAI-afbeeldingen:

- Uitvoer: één PNG-afbeelding per aanvraag.
- Grootte: standaard `1024x1024`; zowel de breedte als de hoogte moet ten minste 768 px zijn.
- Totaal aantal pixels: breedte × hoogte mag maximaal 1,048,576 zijn.
- Bewerkingen: één PNG- of JPEG-invoerafbeelding.
- Niet-ondersteunde gedeelde hints, zoals `aspectRatio`, `resolution`, `quality`,
  `background` en niet-PNG `outputFormat`, worden niet naar Microsoft Foundry verzonden.

## Problemen oplossen

- `az: command not found`: installeer de Azure CLI of gebruik authenticatie met een API-sleutel.
- `Microsoft Foundry endpoint missing for MAI image generation`: selecteer tijdens de onboarding een
  Foundry-implementatie of voeg `models.providers.microsoft-foundry.baseUrl` toe.
- `supports MAI image deployments only`: het geselecteerde afbeeldingsmodel verwijst naar een
  niet-MAI-implementatie. Gebruik een geïmplementeerd MAI-afbeeldingsmodel voor `image_generate`.

<!-- openclaw-plugin-reference:manual-end -->
