---
read_when:
    - Je wilt de standaardbackend voor geheugen begrijpen
    - Je wilt embeddingproviders of hybride zoeken configureren
summary: De standaard op SQLite gebaseerde geheugenbackend met zoeken op trefwoorden, vectoren en een hybride combinatie
title: Ingebouwde geheugenengine
x-i18n:
    generated_at: "2026-07-27T05:02:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3efb6f1449d9b55717b3c117444ba7d4519d0111b842b48790ad85551511433
    source_path: concepts/memory-builtin.md
    workflow: 16
---

De ingebouwde engine is de standaardbackend voor geheugen. Deze slaat je geheugenindex
op in een SQLite-database per agent en vereist geen extra afhankelijkheden om
aan de slag te gaan.

## Wat deze biedt

- **Zoeken op trefwoorden** via FTS5-indexering van volledige tekst (BM25-score).
- **Vectorzoeken** via embeddings van elke ondersteunde provider.
- **Hybride zoeken** dat beide combineert voor de beste resultaten.
- **CJK-ondersteuning** via trigramtokenisatie voor Chinees, Japans en Koreaans.
- **sqlite-vec-versnelling** voor vectorquery's in de database (optioneel).

## Aan de slag

Standaard gebruikt de ingebouwde engine OpenAI-embeddings. Als `OPENAI_API_KEY` of
`models.providers.openai.apiKey` al is geconfigureerd, werkt vectorzoeken
zonder extra geheugenconfiguratie.

Een provider expliciet instellen:

```json5
{
  memory: {
    search: {
      provider: "openai",
    },
  },
}
```

Zonder embeddingprovider is alleen zoeken op trefwoorden beschikbaar.

Installeer de officiële llama.cpp-provider-
plugin om lokale GGUF-embeddings af te dwingen en laat `local.modelPath` vervolgens naar een GGUF-bestand verwijzen:

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

```json5
{
  memory: {
    search: {
      provider: "local",
      fallback: "none",
      local: {
        modelPath: "~/.node-llama-cpp/models/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

## Ondersteunde embeddingproviders

| Provider          | ID                  | Opmerkingen                              |
| ----------------- | ------------------- | ---------------------------------------- |
| Bedrock           | `bedrock`           | Gebruikt de AWS-referentieketen          |
| DeepInfra         | `deepinfra`         | Standaard: `BAAI/bge-m3`                 |
| Gemini            | `gemini`            | Ondersteunt multimodaal (beeld + audio)  |
| GitHub Copilot    | `github-copilot`    | Gebruikt je Copilot-abonnement            |
| LM Studio         | `lmstudio`          | Lokaal/zelfgehost                        |
| Lokaal            | `local`             | `@openclaw/llama-cpp-provider`      |
| Mistral           | `mistral`           |                                        |
| Ollama            | `ollama`            | Lokaal/zelfgehost                        |
| OpenAI            | `openai`            | Standaard: `text-embedding-3-small`   |
| OpenAI-compatibel | `openai-compatible` | Algemeen `/v1/embeddings`-eindpunt   |
| Voyage            | `voyage`            |                                        |

Stel `memory.search.provider` in om van OpenAI over te schakelen.

## Hoe indexering werkt

OpenClaw indexeert `MEMORY.md` en `memory/*.md` in segmenten (standaard 400 tokens met
een overlap van 80 tokens) en slaat deze op in een SQLite-database per agent.

- **Indexlocatie:** de database van de eigenaaragent op
  `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- **Opslagonderhoud:** SQLite WAL-nevenbestanden worden begrensd met periodieke controlepunten en
  controlepunten bij het afsluiten.
- **Bestandsbewaking:** wijzigingen in geheugenbestanden activeren een vertraagde herindexering
  (standaard 1,5 s).
- **Automatische herindexering:** de index wordt automatisch opnieuw opgebouwd wanneer de embeddingprovider,
  het model, de segmenteringsconfiguratie, de geconfigureerde bronnen of het bereik veranderen.
- **Herindexering op aanvraag:** `openclaw memory index --force`

<Info>
Je kunt ook Markdown-bestanden buiten de werkruimte indexeren met
`memory.search.extraPaths`. Zie de
[configuratiereferentie](/nl/reference/memory-config#additional-memory-paths).
</Info>

## Wanneer te gebruiken

De ingebouwde engine is de juiste keuze voor de meeste gebruikers:

- Werkt direct zonder extra afhankelijkheden.
- Verwerkt zoeken op trefwoorden en vectorzoeken goed.
- Ondersteunt alle embeddingproviders.
- Hybride zoeken combineert het beste van beide zoekbenaderingen.

Overweeg over te schakelen op [QMD](/nl/concepts/memory-qmd) als je herordening, query-
uitbreiding nodig hebt of mappen buiten de werkruimte wilt indexeren.

Overweeg [Honcho](/nl/concepts/memory-honcho) als je geheugen over sessies heen wilt
met automatische gebruikersmodellering.

## Problemen oplossen

**Geheugenzoeken uitgeschakeld?** Controleer `openclaw memory status`. Als er geen provider wordt
gedetecteerd, stel je er expliciet een in of voeg je een API-sleutel toe.

**Lokale provider niet gedetecteerd?** Controleer of het lokale pad bestaat en voer het volgende uit:

```bash
openclaw memory status --deep --agent main
openclaw memory index --force --agent main
```

Zowel zelfstandige CLI-opdrachten als de Gateway gebruiken dezelfde provider-ID `local`.
Stel `memory.search.provider: "local"` in wanneer je lokale embeddings wilt.

**Verouderde resultaten?** Voer `openclaw memory index --force` uit om de index opnieuw op te bouwen. De bewaking
kan in zeldzame randgevallen wijzigingen missen.

**Wordt sqlite-vec niet geladen?** OpenClaw valt automatisch terug op cosinus-
gelijkenis binnen het proces. `openclaw memory status --deep` rapporteert de lokale
vectoropslag afzonderlijk van de embeddingprovider, dus `Vector store:
unavailable` verwijst naar het laden van sqlite-vec, terwijl `Embeddings: unavailable`
verwijst naar de gereedheid van de provider/authenticatie of het model. Controleer de logboeken op de specifieke laadfout.

## Configuratie

Zie voor het instellen van embeddingproviders, het afstemmen van hybride zoeken (gewichten, MMR, temporeel
verval), batchindexering, multimodaal geheugen, sqlite-vec, extra paden en alle
overige configuratieopties de
[referentie voor geheugenconfiguratie](/nl/reference/memory-config).

## Gerelateerd

- [Geheugenoverzicht](/nl/concepts/memory)
- [Geheugen doorzoeken](/nl/concepts/memory-search)
- [Active Memory](/nl/concepts/active-memory)
