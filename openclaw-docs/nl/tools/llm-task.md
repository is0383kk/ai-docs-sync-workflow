---
read_when:
    - Je wilt een LLM-stap met uitsluitend JSON binnen workflows
    - Je hebt schemavalideerde LLM-uitvoer nodig voor automatisering
summary: LLM-taken met uitsluitend JSON voor workflows (optionele plugintool)
title: LLM-taak
x-i18n:
    generated_at: "2026-07-27T05:54:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 78ea533f43546fbdd66c7f7138b8dea0b12b02d38925689324b390a12d0c4c5a
    source_path: tools/llm-task.md
    workflow: 16
---

`llm-task` is een gebundelde **optionele plugintool** die één LLM-aanroep met uitsluitend JSON uitvoert
en gestructureerde uitvoer retourneert, die optioneel wordt gevalideerd aan de hand van een JSON
Schema. Hiermee krijgen workflow-engines zoals Lobster een LLM-stap zonder aangepaste
OpenClaw-code per workflow.

## Inschakelen

1. Schakel de plugin in:

```json
{
  "plugins": {
    "entries": {
      "llm-task": { "enabled": true }
    }
  }
}
```

2. Sta de tool toe:

```json
{
  "tools": {
    "alsoAllow": ["llm-task"]
  }
}
```

`alsoAllow` voegt `llm-task` toe boven op het actieve toolprofiel zonder
andere kerntools te beperken. Gebruik in plaats daarvan alleen `tools.allow` als je een beperkende
toestaanlijstmodus wilt.

## Configuratie (optioneel)

```json
{
  "plugins": {
    "entries": {
      "llm-task": {
        "enabled": true,
        "config": {
          "defaultProvider": "openai",
          "defaultModel": "gpt-5.6-sol",
          "defaultAuthProfileId": "main",
          "allowedModels": ["openai/gpt-5.6-sol"],
          "maxTokens": 800,
          "timeoutMs": 30000
        }
      }
    }
  }
}
```

`allowedModels` is een toestaanlijst van `provider/model`-tekenreeksen; een aanvraag voor elk
ander model wordt geweigerd. Alle andere sleutels zijn terugvalwaarden per aanroep die worden gebruikt wanneer de
toolaanroep die parameter weglaat.

## Toolparameters

| Parameter       | Type   | Opmerkingen                                                                                                                                         |
| --------------- | ------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `prompt`        | string | Vereist. Taakinstructie voor het LLM.                                                                                                       |
| `input`         | any    | Optionele payload; wordt naar JSON geserialiseerd en aan de prompt toegevoegd.                                                                              |
| `schema`        | object | Optioneel JSON Schema waaraan de geparseerde uitvoer moet voldoen.                                                                                 |
| `provider`      | string | Overschrijft `defaultProvider` / de standaardprovider van de agent.                                                                                   |
| `model`         | string | Overschrijft `defaultModel`; accepteert kale model-id's, aliassen of een `provider/model`-verwijzing (een dubbel providervoorvoegsel wordt automatisch verwijderd). |
| `thinking`      | string | Redeneerniveau (bijv. `low`, `medium`); moet door het gevonden model worden ondersteund.                                                          |
| `authProfileId` | string | Overschrijft `defaultAuthProfileId`.                                                                                                             |
| `temperature`   | number | Naar beste vermogen; niet alle providers respecteren dit.                                                                                                      |
| `maxTokens`     | number | Limiet naar beste vermogen voor uitvoertokens.                                                                                                             |
| `timeoutMs`     | number | Time-out voor uitvoering; standaard `30000`.                                                                                                                 |

## Uitvoer

Retourneert `details.json` (de geparseerde, aan het schema getoetste JSON) plus `details.provider`
en `details.model`, die aangeven wat daadwerkelijk is uitgevoerd.

## Voorbeeld: Lobster-workflowstap

### Belangrijke beperking

In het onderstaande voorbeeld wordt ervan uitgegaan dat de **zelfstandige Lobster CLI** wordt uitgevoerd waar
`openclaw.invoke` al de juiste Gateway-URL/authenticatiecontext heeft.

Voor de gebundelde **ingebedde** Lobster-runner in OpenClaw is dit geneste CLI-
patroon **momenteel niet betrouwbaar**:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{ ... }'
```

Totdat ingebedde Lobster een ondersteunde brug voor deze flow heeft, geef je de voorkeur aan:

- rechtstreekse `llm-task`-toolaanroepen buiten Lobster, of
- Lobster-stappen die niet afhankelijk zijn van geneste `openclaw.invoke`-aanroepen.

Voorbeeld voor de zelfstandige Lobster CLI:

```lobster
openclaw.invoke --tool llm-task --action json --args-json '{
  "prompt": "Geef op basis van de ingevoerde e-mail de intentie en een concept terug.",
  "thinking": "low",
  "input": {
    "subject": "Hallo",
    "body": "Kun je helpen?"
  },
  "schema": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "draft": { "type": "string" }
    },
    "required": ["intent", "draft"],
    "additionalProperties": false
  }
}'
```

## Veiligheidsopmerkingen

- **Alleen JSON**: het model krijgt de instructie om uitsluitend een JSON-waarde te retourneren, zonder code-
  fences en zonder commentaar.
- **Geen tools**: voor de onderliggende uitvoering zijn tools uitgeschakeld, zodat het model
  tijdens de taak geen externe aanroepen kan doen.
- Behandel de uitvoer als niet-vertrouwd, tenzij je deze valideert met `schema`.
- Plaats goedkeuringen vóór elke stap met neveneffecten (verzenden, plaatsen, uitvoeren) die deze
  uitvoer gebruikt.

## Gerelateerd

- [Redeneerniveaus](/nl/tools/thinking)
- [Subagenten](/nl/tools/subagents)
- [Slash-opdrachten](/nl/tools/slash-commands)
