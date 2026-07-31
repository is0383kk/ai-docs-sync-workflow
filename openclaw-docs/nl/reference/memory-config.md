---
read_when:
    - Je wilt providers voor geheugenzoekopdrachten of embeddingmodellen configureren
    - Je wilt de QMD-backend instellen
    - Je wilt hybride zoekopdrachten, MMR of tijdsverval inschakelen
    - Je wilt multimodale geheugenindexering inschakelen
sidebarTitle: Memory config
summary: Providers voor geheugenzoekopdrachten, ophaalmodi, QMD en multimodale indexering
title: Referentie voor geheugenconfiguratie
x-i18n:
    generated_at: "2026-07-27T05:49:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91f843b1516093c49e18b3d659ab24ea9cb7be32aaaac722205eca8bc3f2ca5b
    source_path: reference/memory-config.md
    workflow: 16
---

Deze pagina vermeldt elke configuratieoptie voor het doorzoeken van het geheugen van OpenClaw. Zie voor conceptuele overzichten:

<CardGroup cols={2}>
  <Card title="Geheugenoverzicht" href="/nl/concepts/memory">
    Hoe het geheugen werkt.
  </Card>
  <Card title="Ingebouwde engine" href="/nl/concepts/memory-builtin">
    Standaard SQLite-backend.
  </Card>
  <Card title="QMD-engine" href="/nl/concepts/memory-qmd">
    Local-first-sidecar.
  </Card>
  <Card title="Geheugen doorzoeken" href="/nl/concepts/memory-search">
    Zoekpijplijn en afstemming.
  </Card>
  <Card title="Active Memory" href="/nl/concepts/active-memory">
    Geheugensubagent voor interactieve sessies.
  </Card>
</CardGroup>

Alle gedeelde geheugeninstellingen staan onder `memory` op het hoogste niveau in `openclaw.json`. Zoekstandaarden gebruiken `memory.search`; zoekspecifieke overschrijvingen per agent gebruiken `agents.entries.*.memory.search`.

<Note>
Gebruik voor de aanbevolen workflow voor persoonlijke agents
`memory.search.rememberAcrossConversations`. Geavanceerde instellingen voor doelbepaling,
model, prompt en latentie van Active Memory staan onder `plugins.entries.active-memory`.

Zie [Active Memory](/nl/concepts/active-memory) voor beide activeringspaden,
persistentie van transcripten en richtlijnen voor een veilige uitrol.
</Note>

---

## Onthouden tussen gesprekken

| Sleutel                       | Type      | Standaard                                                  | Beschrijving                                                                  |
| ----------------------------- | --------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `rememberAcrossConversations` | `boolean` | Aan voor persoonlijke installaties; uit bij geconfigureerde DM-isolatie | Gebruik relevante context uit andere herkende privégesprekken van deze agent. |

Configureer dit per agent wanneer alleen een vertrouwde persoonlijke agent
transcripten uit andere gesprekken mag ophalen:

```json5
{
  agents: {
    entries: {
      personal: {
        memory: {
          search: {
            rememberAcrossConversations: true,
          },
        },
      },
    },
  },
}
```

De waarde volgt de normale overerving van `memory.search` met een
overschrijving per agent. Wanneer deze niet is ingesteld, is deze standaard alleen ingeschakeld als globale
`session.dmScope` niet is ingesteld of `"main"` is en geen binding een overschrijving voor `session.dmScope`
heeft. Elke geconfigureerde DM-isolatie schakelt deze standaard uit. Een expliciete `true` of
`false` heeft altijd voorrang. Als dit wordt ingeschakeld, wordt sessietranscriptindexering geactiveerd en
wordt `sessions` toegevoegd aan de opgeloste geheugenbronnen van de agent. Met QMD wordt ook
de sessie-export van die agent ingeschakeld; voor deze modus is geen afzonderlijke instelling
`memory.qmd.sessions.enabled` vereist.

De ingebouwde geheugenprovider van OpenClaw ondersteunt dit beveiligde pad met zowel de
ingebouwde backend als de QMD-backend. Alternatieve geheugenproviders kunnen hun eigen
ophaalhooks en geavanceerde Active Memory-tools blijven gebruiken, maar deze instelling wordt overgeslagen
tenzij de huidige provider beveiligd ophalen van privétranscripten ondersteunt.
`openclaw doctor` meldt een niet-ondersteunde provider of een expliciete Active Memory-lijst
`toolsAllow` waarin `memory_search` ontbreekt.

De ophaalgrens is beperkter dan bij algemeen zoeken in sessies:

- alleen herkende privégesprekken van dezelfde agent komen in aanmerking
- het gesprek dat wordt beantwoord, wordt uitgesloten
- groepen en kanalen worden uitgesloten als bronnen en bestemmingen
- onbekende gesprekstypen worden standaard geweigerd
- ophalen in een sandbox kan de speciale autorisatie voor meerdere gesprekken niet gebruiken

De instelling wijzigt `tools.sessions.visibility`, sessiesleutels,
transcriptopslag, afleveringsroutering of de machtigingen van `sessions_list`,
`sessions_history` en `sessions_send` niet. Active Memory voert een begrensde
alleen-lezen-ophaalronde uit; niet-beschikbaar of door een time-out afgebroken ophalen blokkeert het
antwoord niet.

---

## Providerselectie

| Sleutel    | Type      | Standaard        | Beschrijving                                                                                                                                                                                                                                                                               |
| ---------- | --------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`  | `boolean` | `true`           | Geheugenzoekfunctie in- of uitschakelen                                                                                                                                                                                                                                                     |
| `provider` | `string`  | `"openai"`       | ID van de embeddingadapter, zoals `bedrock`, `deepinfra`, `gemini`, `github-copilot`, `local`, `mistral`, `ollama`, `openai`, `openai-compatible` of `voyage`; kan ook een geconfigureerde `models.providers.<id>` zijn waarvan `api` verwijst naar een geheugenembeddingadapter of OpenAI-compatibele model-API |
| `model`    | `string`  | standaard van provider | Naam van embeddingmodel                                                                                                                                                                                                                                                                    |
| `fallback` | `string`  | `"none"`         | ID van fallbackadapter wanneer de primaire adapter mislukt                                                                                                                                                                                                                                 |

Wanneer `provider` niet is ingesteld, gebruikt OpenClaw embeddings van OpenAI. Stel `provider`
expliciet in om Bedrock, DeepInfra, Gemini, GitHub Copilot, Mistral, Ollama,
Voyage, een lokaal GGUF-model of een OpenAI-compatibel `/v1/embeddings`-eindpunt te gebruiken.
Verouderde configuraties die nog `provider: "auto"` vermelden, worden omgezet naar `openai`.

<Warning>
Als je de embeddingprovider, het model, de providerinstellingen, bronnen, het bereik,
de segmentering of de tokenizer wijzigt, kan de bestaande SQLite-vectorindex incompatibel worden.
OpenClaw pauzeert het zoeken met vectoren en meldt een waarschuwing over de indexidentiteit in plaats van
alles automatisch opnieuw van embeddings te voorzien. Bouw de index opnieuw wanneer je klaar bent met
`openclaw memory status --index --agent <id>` of
`openclaw memory index --force --agent <id>`.
</Warning>

Wanneer `provider` niet is ingesteld, de verouderde `provider: "auto"` aanwezig is, of
`provider: "none"` bewust de modus met alleen FTS selecteert, kan het ophalen uit het geheugen nog steeds
lexicale FTS-rangschikking gebruiken wanneer embeddings niet beschikbaar zijn.

Expliciete niet-lokale providers worden standaard geweigerd. Als je `memory.search.provider` instelt op
een concrete provider met een externe backend, zoals Bedrock, DeepInfra, Gemini, GitHub
Copilot, LM Studio, Mistral, Ollama, OpenAI, Voyage of een OpenAI-compatibele
aangepaste provider, en die provider tijdens runtime niet beschikbaar is, retourneert `memory_search`
een resultaat dat aangeeft dat deze niet beschikbaar is, in plaats van stilzwijgend alleen FTS-ophalen te gebruiken. Herstel de
provider-/authenticatieconfiguratie, schakel over naar een bereikbare provider of stel
`provider: "none"` in als je bewust alleen FTS wilt gebruiken voor het ophalen.

### Aangepaste provider-ID's

`memory.search.provider` kan verwijzen naar een aangepaste `models.providers.<id>`-vermelding voor geheugenspecifieke provideradapters zoals `ollama`, of voor OpenAI-compatibele model-API's zoals `openai-responses` / `openai-completions`. OpenClaw bepaalt de eigenaar van `api` van die provider voor de embeddingadapter, terwijl de aangepaste provider-ID behouden blijft voor de verwerking van eindpunten, authenticatie en modelvoorvoegsels. Hierdoor kunnen opstellingen met meerdere GPU's of hosts geheugenembeddings aan een specifiek lokaal eindpunt toewijzen:

```json5
{
  models: {
    providers: {
      "ollama-5080": {
        api: "ollama",
        baseUrl: "http://gpu-box.local:11435",
        apiKey: "ollama-local",
        models: [{ id: "qwen3-embedding:0.6b", name: "Qwen3 Embedding 0.6B" }],
      },
    },
  },
  memory: {
    search: {
      provider: "ollama-5080",
      model: "qwen3-embedding:0.6b",
    },
  },
}
```

### API-sleutel bepalen

Externe embeddings vereisen een API-sleutel. Bedrock gebruikt in plaats daarvan de standaardreferentieketen van de AWS SDK (instancerollen, SSO, toegangssleutels of een Bedrock-API-sleutel).

| Provider       | Omgevingsvariabele                                | Configuratiesleutel                 |
| -------------- | --------------------------------------------------- | ----------------------------------- |
| Bedrock        | AWS-referentieketen of `AWS_BEARER_TOKEN_BEDROCK` | Geen API-sleutel nodig              |
| DeepInfra      | `DEEPINFRA_API_KEY`                                 | `models.providers.deepinfra.apiKey` |
| Gemini         | `GEMINI_API_KEY`                                    | `models.providers.google.apiKey`    |
| GitHub Copilot | `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`, `GITHUB_TOKEN`  | Auth-profiel via apparaataanmelding |
| Mistral        | `MISTRAL_API_KEY`                                   | `models.providers.mistral.apiKey`   |
| Ollama         | `OLLAMA_API_KEY` (tijdelijke aanduiding)            | --                                  |
| OpenAI         | `OPENAI_API_KEY`                                    | `models.providers.openai.apiKey`    |
| Voyage         | `VOYAGE_API_KEY`                                    | `models.providers.voyage.apiKey`    |

<Note>
Codex OAuth dekt alleen chat/voltooiingen en voldoet niet aan embeddingverzoeken.
</Note>

---

## Configuratie van extern eindpunt

Gebruik `provider: "openai-compatible"` voor een generieke OpenAI-compatibele
`/v1/embeddings`-server die geen globale OpenAI-chatreferenties mag overnemen.

<ParamField path="remote.baseUrl" type="string">
  Aangepaste basis-URL voor de API.
</ParamField>
<ParamField path="remote.apiKey" type="string">
  API-sleutel overschrijven.
</ParamField>
<ParamField path="remote.headers" type="object">
  Extra HTTP-headers (samengevoegd met de standaardwaarden van de provider).
</ParamField>

```json5
{
  memory: {
    search: {
      provider: "openai-compatible",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_KEY",
      },
    },
  },
}
```

---

## Providerspecifieke configuratie

<AccordionGroup>
  <Accordion title="Gemini">
    | Sleutel                | Type     | Standaard              | Beschrijving                               |
    | ---------------------- | -------- | ---------------------- | ------------------------------------------- |
    | `model`                | `string` | `gemini-embedding-001` | Ondersteunt ook `gemini-embedding-2-preview` |
    | `outputDimensionality` | `number` | `3072`                 | Voor Embedding 2: 768, 1536 of 3072        |

    <Warning>
    Als je het model of `outputDimensionality` wijzigt, verandert de indexidentiteit. OpenClaw
    pauzeert het zoeken met vectoren totdat je de geheugenindex expliciet opnieuw opbouwt.
    </Warning>

  </Accordion>
  <Accordion title="Invoertypen voor OpenAI-compatibele eindpunten">
    OpenAI-compatibele embeddingeindpunten kunnen providerspecifieke aanvraagvelden voor `input_type` inschakelen. Dit is nuttig voor asymmetrische embeddingmodellen die verschillende labels vereisen voor embeddings van zoekopdrachten en documenten.

    | Sleutel             | Type     | Standaard    | Beschrijving                                                   |
    | ------------------- | -------- | ------------ | -------------------------------------------------------------- |
    | `inputType`         | `string` | niet ingesteld | Gedeelde `input_type` voor embeddings van zoekopdrachten en documenten |
    | `queryInputType`    | `string` | niet ingesteld | `input_type` tijdens het zoeken; overschrijft `inputType`         |
    | `documentInputType` | `string` | niet ingesteld | `input_type` voor index/document; overschrijft `inputType`       |

    ```json5
    {
      memory: {
        search: {
          provider: "openai-compatible",
          remote: {
            baseUrl: "https://embeddings.example/v1",
            apiKey: "${EMBEDDINGS_API_KEY}",
          },
          model: "asymmetric-embedder",
          queryInputType: "query",
          documentInputType: "passage",
        },
      },
    }
    ```

    Het wijzigen van deze waarden beïnvloedt de identiteit van de embeddingcache voor batchindexering door de provider en moet worden gevolgd door een herindexering van het geheugen wanneer het upstreammodel de labels anders behandelt.

  </Accordion>
  <Accordion title="Bedrock">
    ### Configuratie voor Bedrock-embeddings

    Bedrock gebruikt de standaardreferentieketen van de AWS SDK plus een door OpenClaw gecontroleerd bearertoken, zodat er geen API-sleutels in de configuratie worden opgeslagen. Als OpenClaw op EC2 draait met een voor Bedrock ingeschakelde instantiëringsrol, hoef je alleen de provider en het model in te stellen:

    ```json5
    {
      memory: {
        search: {
          provider: "bedrock",
          model: "amazon.titan-embed-text-v2:0",
        },
      },
    }
    ```

    | Sleutel                | Type     | Standaard                       | Beschrijving                       |
    | ---------------------- | -------- | ------------------------------- | ---------------------------------- |
    | `model`                | `string` | `amazon.titan-embed-text-v2:0` | Elke Bedrock-embeddingmodel-ID     |
    | `outputDimensionality` | `number` | modelstandaard                 | Voor Titan V2: 256, 512 of 1024    |

    **Ondersteunde modellen** (met detectie van de familie en standaarddimensies):

    | Model-ID                                    | Provider   | Standaarddimensies | Configureerbare dimensies  |
    | ------------------------------------------- | ---------- | ------------------ | -------------------------- |
    | `amazon.titan-embed-text-v2:0`             | Amazon     | 1024         | 256, 512, 1024             |
    | `amazon.titan-embed-text-v1`               | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-g1-text-02`            | Amazon     | 1536         | --                          |
    | `amazon.titan-embed-image-v1`              | Amazon     | 1024         | --                          |
    | `amazon.nova-2-multimodal-embeddings-v1:0` | Amazon     | 1024         | 256, 384, 1024, 3072       |
    | `cohere.embed-english-v3`                  | Cohere     | 1024         | --                          |
    | `cohere.embed-multilingual-v3`             | Cohere     | 1024         | --                          |
    | `cohere.embed-v4:0`                        | Cohere     | 1536         | 256, 384, 512, 768, 1024, 1536 |
    | `twelvelabs.marengo-embed-3-0-v1:0`        | TwelveLabs | 512          | --                          |
    | `twelvelabs.marengo-embed-2-7-v1:0`        | TwelveLabs | 1024         | --                          |

    Varianten met een throughputachtervoegsel (bijvoorbeeld `amazon.titan-embed-text-v1:2:8k`) en inferentieprofiel-ID's met een regiovoorvoegsel (bijvoorbeeld `us.amazon.titan-embed-text-v2:0`) nemen de configuratie van het basismodel over.

    **Regio:** wordt in deze volgorde bepaald: de overschrijving `memory.search.remote.baseUrl`, de configuratie `models.providers.amazon-bedrock.baseUrl`, `AWS_REGION`, `AWS_DEFAULT_REGION`, en vervolgens de standaardwaarde `us-east-1`.

    **Authenticatie:** OpenClaw controleert eerst op `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` of `AWS_BEARER_TOKEN_BEDROCK` en valt daarna terug op de standaardketen van referentieproviders van de AWS SDK:

    1. Omgevingsvariabelen (`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`), tenzij `AWS_PROFILE` ook is ingesteld
    2. SSO (alleen wanneer SSO-velden zijn geconfigureerd)
    3. Gedeelde referentie- en configuratiebestanden (`fromIni`, inclusief `AWS_PROFILE`)
    4. Referentieproces (`credential_process` in het AWS-configuratiebestand)
    5. Referenties voor webidentiteitstokens
    6. Referenties uit ECS- of EC2-instantiemetadata

    **IAM-machtigingen:** de IAM-rol of -gebruiker heeft het volgende nodig:

    ```json
    {
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "*"
    }
    ```

    Beperk voor minimale bevoegdheden `InvokeModel` tot het specifieke model:

    ```text
    arn:aws:bedrock:*::foundation-model/amazon.titan-embed-text-v2:0
    ```

  </Accordion>
  <Accordion title="Lokaal (GGUF + llama.cpp)">
    | Sleutel               | Type               | Standaard               | Beschrijving                                                                                                                                                                                                                                                                                                          |
    | --------------------- | ------------------ | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | `local.modelPath`     | `string`           | automatisch gedownload | Pad naar het GGUF-modelbestand                                                                                                                                                                                                                                                                                        |
    | `local.modelCacheDir` | `string`           | standaard van node-llama-cpp | Cachemap voor gedownloade modellen                                                                                                                                                                                                                                                                                |
    | `local.contextSize`   | `number \| "auto"` | `4096`                 | Grootte van het contextvenster voor de embeddingcontext. 4096 dekt gebruikelijke fragmenten (128-512 tokens) en begrenst tegelijkertijd VRAM dat niet voor gewichten wordt gebruikt. Verlaag dit op beperkte hosts naar 1024-2048. `"auto"` gebruikt het getrainde maximum van het model -- niet aanbevolen voor modellen van 8B+ (Qwen3-Embedding-8B: maximaal 40 960 tokens kunnen het VRAM-gebruik tot ~32 GB verhogen). |

    Installeer eerst de officiële llama.cpp-provider: `openclaw plugins install @openclaw/llama-cpp-provider`.
    Standaardmodel: `embeddinggemma-300m-qat-Q8_0.gguf` (~0.6 GB, automatisch gedownload). Voor broncodecheck-outs blijft goedkeuring voor een native build vereist: `pnpm approve-builds` en vervolgens `pnpm rebuild node-llama-cpp`.

    Gebruik de zelfstandige CLI om hetzelfde providerpad te verifiëren dat de Gateway gebruikt:

    ```bash
    openclaw memory status --deep --agent main
    openclaw memory index --force --agent main
    ```

    Numerieke waarden voor `local.contextSize` sturen ook de automatische plaatsing van GPU-lagen door node-llama-cpp aan, zodat de modelgewichten en de aangevraagde embeddingcontext samen passen. `openclaw memory status --deep` rapporteert de laatst bekende llama.cpp-backend, het apparaat, de offload, de aangevraagde context en geheugengegevens met tijdstempel nadat de runtime is geladen; passieve status laadt geen model.

    Stel `provider: "local"` expliciet in voor lokale GGUF-embeddings. `hf:` en HTTP(S)-modelverwijzingen worden ondersteund voor expliciete lokale configuraties (via de modelresolutie van node-llama-cpp), maar ze wijzigen de standaardprovider niet.

  </Accordion>
</AccordionGroup>

## Indexeringsgedrag

Geheugenengines beheren synchronisatie, batchverwerking, bewaking en heuristieken
voor indexering na Compaction. OpenClaw houdt dit gedrag ingeschakeld met onderhouden
standaardwaarden in plaats van tijdschakelaars per installatie beschikbaar te stellen.

## Configuratie voor hybride zoeken

Alles onder `memory.search.query`:

| Sleutel      | Type     | Standaard | Beschrijving                                         |
| ------------ | -------- | --------- | ---------------------------------------------------- |
| `maxResults` | `number` | `6`     | Maximaal aantal geheugenresultaten vóór injectie     |
| `minScore`   | `number` | `0.35`  | Minimale relevantiescore om een resultaat op te nemen |

Hybride opvraging blijft ingeschakeld; MMR en tijdsverval blijven uitgeschakeld door
het ingebouwde enginebeleid.

### Volledig voorbeeld

```json5
{
  memory: {
    search: {
      query: {
        maxResults: 6,
        minScore: 0.35,
      },
    },
  },
}
```

---

## Aanvullende geheugenpaden

| Sleutel      | Type       | Beschrijving                                  |
| ------------ | ---------- | --------------------------------------------- |
| `extraPaths` | `string[]` | Aanvullende mappen of bestanden om te indexeren |

```json5
{
  memory: {
    search: {
      extraPaths: ["../team-docs", "/srv/shared-notes"],
    },
  },
}
```

Paden kunnen absoluut of relatief ten opzichte van de werkruimte zijn. Mappen worden recursief gescand op `.md`-bestanden. De afhandeling van symbolische koppelingen hangt af van de actieve backend: de ingebouwde engine slaat symbolische koppelingen over, terwijl QMD het gedrag van de onderliggende QMD-scanner volgt.

Gebruik voor agentgebonden zoekopdrachten in transcripties van andere agents `agents.entries.*.memory.search.qmd.extraCollections` in plaats van `memory.qmd.paths`. Die aanvullende verzamelingen volgen dezelfde `{ path, name, pattern? }`-structuur, maar worden per agent samengevoegd en kunnen expliciete gedeelde namen behouden wanneer het pad buiten de huidige werkruimte wijst. Als hetzelfde opgeloste pad zowel in `memory.qmd.paths` als in `memory.search.qmd.extraCollections` voorkomt, behoudt QMD de eerste vermelding en slaat het duplicaat over.

---

## Multimodaal geheugen (Gemini)

Indexeer afbeeldingen en audio naast Markdown met Gemini Embedding 2:

| Sleutel                   | Type       | Standaard  | Beschrijving                                  |
| ------------------------- | ---------- | ---------- | --------------------------------------------- |
| `multimodal.enabled`      | `boolean`  | `false`    | Multimodale indexering inschakelen             |
| `multimodal.modalities`   | `string[]` | --         | `["image"]`, `["audio"]` of `["all"]` |
| `multimodal.maxFileBytes` | `number`   | `10485760` | Maximale bestandsgrootte voor indexering (10 MiB) |

<Note>
Alleen van toepassing op bestanden in `extraPaths`. Standaardgeheugenroots blijven uitsluitend voor Markdown. Vereist `gemini-embedding-2-preview`. `fallback` moet `"none"` zijn.
</Note>

Ondersteunde indelingen: `.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.heic`, `.heif` (afbeeldingen); `.mp3`, `.wav`, `.ogg`, `.opus`, `.m4a`, `.aac`, `.flac` (audio).

---

## Embeddingcache

| Sleutel         | Type      | Standaard | Beschrijving                         |
| --------------- | --------- | --------- | ------------------------------------ |
| `cache.enabled` | `boolean` | `true`  | Embeddings van fragmenten cachen in SQLite |

Voorkomt dat ongewijzigde tekst opnieuw wordt ingebed tijdens herindexering of transcriptupdates.

---

## Batchindexering

| Sleutel                      | Type      | Standaard | Beschrijving                     |
| ---------------------------- | --------- | --------- | -------------------------------- |
| `remote.nonBatchConcurrency` | `number`  | `4`     | Parallelle inline-embeddings     |
| `remote.batch.enabled`       | `boolean` | `false` | Batch-embedding-API inschakelen  |

Beschikbaar voor `gemini`, `openai` en `voyage`. OpenAI-batchverwerking is doorgaans het snelst en goedkoopst voor grote aanvullingen met historische gegevens.

Gelijktijdigheid, polling en time-outgedrag worden door de provider beheerd.

---

## Zoeken in sessiegeheugen

Indexeer sessietranscripties en maak ze beschikbaar via `memory_search`:

| Sleutel                       | Type       | Standaard    | Beschrijving                                      |
| ----------------------------- | ---------- | ------------ | ------------------------------------------------- |
| `rememberAcrossConversations` | `boolean`  | `false`      | Privéherinnering tussen gesprekken toestaan       |
| `sources`                     | `string[]` | `["memory"]` | Voeg `"sessions"` toe om transcripties op te nemen |

<Warning>
Sessie-indexering is opt-in en wordt asynchroon uitgevoerd. Resultaten kunnen enigszins verouderd zijn. Sessielogboeken staan op schijf, dus beschouw toegang tot het bestandssysteem als de vertrouwensgrens.
</Warning>

Gewone, door het model aangeroepen zoekopdrachten in sessietranscripten volgen
[`tools.sessions.visibility`](/nl/gateway/config-tools#toolssessions). De standaardzichtbaarheid
`tree` maakt de huidige sessie zichtbaar, evenals sessies die hierdoor zijn gestart en
groepssessies van dezelfde agent die via omgevingsbewustzijn van groepen worden gevolgd. Voor andere,
niet-gerelateerde sessies is zichtbaarheid `agent` vereist (of `all` alleen wanneer herinneringen
tussen agents ook vereist zijn en het agent-naar-agentbeleid dit toestaat).

`rememberAcrossConversations` verruimt die instelling niet. Het biedt
een afzonderlijke, uitsluitend voor runtime geldende autorisatie die tijdens de begrensde Active Memory-pass
beperkt is tot privétranscripten van dezelfde agent.

In de onderstaande voorbeelden staan deze instellingen onder `memory.search` op het hoogste niveau. Je kunt ook
gelijkwaardige instellingen toepassen in een `memory.search`-override per agent wanneer slechts één
agent sessietranscripten moet indexeren en doorzoeken.

Voor herinneringen van dezelfde agent van gateway naar DM:

<Tabs>
  <Tab title="Ingebouwde backend">
    ```json5
    {
      memory: {
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
  <Tab title="QMD-backend">
    ```json5
    {
      memory: {
        backend: "qmd",
        search: {
          experimental: { sessionMemory: true },
          sources: ["memory", "sessions"],
        },
        qmd: {
          sessions: { enabled: true },
        },
      },
      tools: {
        sessions: { visibility: "agent" },
      },
    }
    ```
  </Tab>
</Tabs>

Bij gebruik van QMD exporteert `sources: ["sessions"]` op zichzelf geen transcripten naar QMD. Stel ook
`memory.qmd.sessions.enabled: true` in. De instelling op hoger niveau
`rememberAcrossConversations: true` vormt de uitzondering: deze impliceert de
vereiste export van QMD-sessies voor die agent. Impliciete exports blijven privé:
ze gebruiken altijd de standaard interne exportlocatie (een geconfigureerde
`sessions.exportDir` geldt alleen voor expliciete exports), worden alleen doorzocht
tijdens de herinneringen tussen gesprekken van die agent en gewone `memory_get`
kan ze niet lezen. Expliciete
`memory.qmd.sessions.enabled: true` behoudt het bestaande gedrag en maakt
geëxporteerde transcripten onderdeel van de gewone geheugencorpus.

---

## SQLite-vectorversnelling (sqlite-vec)

| Sleutel                      | Type      | Standaard | Beschrijving                             |
| ---------------------------- | --------- | --------- | ---------------------------------------- |
| `store.vector.enabled`       | `boolean` | `true`  | sqlite-vec gebruiken voor vectorquery's |
| `store.vector.extensionPath` | `string`  | meegeleverd | Pad naar sqlite-vec overschrijven       |

Wanneer sqlite-vec niet beschikbaar is, valt OpenClaw automatisch terug op cosinusgelijkenis binnen het proces.

---

## Indexopslag

Ingebouwde geheugenindexen staan in de OpenClaw SQLite-database van elke agent op
`agents/<agentId>/agent/openclaw-agent.sqlite`.

| Sleutel                | Type     | Standaard   | Beschrijving                                     |
| ---------------------- | -------- | ----------- | ------------------------------------------------ |
| `store.fts.tokenizer` | `string` | `unicode61` | FTS5-tokenizer (`unicode61` of `trigram`) |

---

## Configuratie van de QMD-backend

Stel `memory.backend = "qmd"` in om deze in te schakelen. Alle QMD-instellingen staan onder `memory.qmd`:

| Sleutel                   | Type      | Standaard | Beschrijving                                                                                      |
| ------------------------- | --------- | --------- | ------------------------------------------------------------------------------------------------- |
| `command`                | `string`  | `qmd`    | Pad naar het uitvoerbare QMD-bestand; stel een absoluut pad in wanneer service-`PATH` afwijkt van je shell |
| `searchMode`             | `string`  | `search` | Zoekopdracht: `search`, `vsearch`, `query`                                          |
| `rerank`                 | `boolean` | --       | Stel in op `false` met `searchMode: "query"` en QMD 2.1+ om QMD-herrangschikking over te slaan          |
| `includeDefaultMemory`   | `boolean` | `true`   | `MEMORY.md` + `memory/**/*.md` automatisch indexeren                                             |
| `paths[]`                | `array`   | --       | Extra paden: `{ name, path, pattern? }`                                               |
| `sessions.enabled`       | `boolean` | `false`  | Sessietranscripten naar QMD exporteren                                                       |
| `sessions.retentionDays` | `number`  | --       | Bewaartermijn van transcripten                                                               |
| `sessions.exportDir`     | `string`  | --       | Exportmap                                                                                    |

`searchMode: "search"` is uitsluitend lexicaal/BM25. OpenClaw voert voor die modus geen gereedheidscontroles voor semantische vectoren of onderhoud van QMD-embeddings uit, ook niet tijdens `memory status --deep`; `vsearch` en `query` blijven QMD-vectorgereedheid en embeddings vereisen.

`rerank: false` wijzigt alleen de QMD-`query`-modus en vereist QMD 2.1 of nieuwer. In directe CLI-modus geeft OpenClaw `--no-rerank` door; in door mcporter ondersteunde MCP-modus geeft het `rerank: false` door aan QMD's uniforme querytool. Laat dit oningesteld om het standaardgedrag van QMD voor het herrangschikken van query's te gebruiken.

OpenClaw geeft de voorkeur aan actuele QMD-collectie- en MCP-querystructuren, maar houdt oudere QMD-releases werkend door waar nodig compatibele vlaggen voor collectiepatronen en oudere MCP-toolnamen te proberen. Wanneer QMD ondersteuning voor meerdere collectiefilters aangeeft, worden collecties met dezelfde bron met één QMD-proces doorzocht; oudere QMD-builds behouden het compatibiliteitspad per collectie. Dezelfde bron betekent dat duurzame geheugencollecties (standaardgeheugenbestanden plus aangepaste paden) samen worden gegroepeerd, terwijl collecties met sessietranscripten een afzonderlijke groep blijven, zodat brondiversificatie nog steeds beide invoerbronnen heeft.

<Note>
Overrides voor QMD-modellen blijven aan de QMD-zijde en niet in de OpenClaw-configuratie. Als je de modellen van QMD globaal moet overschrijven, stel dan omgevingsvariabelen zoals `QMD_EMBED_MODEL`, `QMD_RERANK_MODEL` en `QMD_GENERATE_MODEL` in de runtimeomgeving van de Gateway in.
</Note>

<AccordionGroup>
  <Accordion title="Limieten">
    | Sleutel                    | Type     | Standaard | Beschrijving                |
    | ------------------------- | -------- | --------- | --------------------------- |
    | `limits.maxResults`       | `number` | `4`     | Maximaal aantal zoekresultaten |
    | `limits.maxSnippetChars`  | `number` | `450`   | Lengte van fragment begrenzen |
    | `limits.maxInjectedChars` | `number` | `2200`  | Totaal aantal ingevoegde tekens begrenzen |
    | `limits.timeoutMs`        | `number` | `4000`  | Time-out van QMD-opdracht tijdens door QMD ondersteund zoeken, inclusief `memory_search`; installatie, synchronisatie, ingebouwde fallback en aanvullend werk behouden de standaarddeadline van de tool |
  </Accordion>
  <Accordion title="Bereik">
    Bepaalt welke sessies QMD-zoekresultaten kunnen ontvangen. Hetzelfde schema als [`session.sendPolicy`](/nl/gateway/config-agents#session):

    ```json5
    {
      memory: {
        qmd: {
          scope: {
            default: "deny",
            rules: [{ action: "allow", match: { chatType: "direct" } }],
          },
        },
      },
    }
    ```

    De meegeleverde standaard staat alleen DM/direct toe en weigert groepen en andere kanaaltypen. `match.keyPrefix` komt overeen met de genormaliseerde sessiesleutel; `match.rawKeyPrefix` komt overeen met de onbewerkte sleutel inclusief `agent:<id>:`.

  </Accordion>
  <Accordion title="Citaten">
    `memory.citations` geldt voor alle backends:

    | Waarde           | Gedrag                                               |
    | ---------------- | ---------------------------------------------------- |
    | `auto` (standaard) | `Source: <path#line>`-voettekst opnemen in fragmenten |
    | `on`             | Voettekst altijd opnemen                           |
    | `off`            | Voettekst weglaten (pad wordt intern nog steeds aan agent doorgegeven) |

  </Accordion>
</AccordionGroup>

QMD wordt lui geïnitialiseerd wanneer het geheugen voor het eerst wordt gebruikt; de adapter beheert de vernieuwings- en embeddingschema's.

### Volledig QMD-voorbeeld

```json5
{
  memory: {
    backend: "qmd",
    citations: "auto",
    qmd: {
      includeDefaultMemory: true,
      update: { interval: "5m", debounceMs: 15000 },
      limits: { maxResults: 4, timeoutMs: 4000 },
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

---

## Dreaming

Dreaming wordt geconfigureerd onder `plugins.entries.memory-core.config.dreaming`, niet onder `memory.search`.

Dreaming wordt uitgevoerd als één geplande controlecyclus en gebruikt interne lichte/diepe/REM-fasen als implementatiedetail.

Zie [Dreaming](/nl/concepts/dreaming) voor conceptueel gedrag en slash-opdrachten.

### Gebruikersinstellingen

| Sleutel                                  | Type      | Standaard     | Beschrijving                                                                                                                           |
| ---------------------------------------- | --------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`                              | `boolean` | `false`       | Dreaming volledig in- of uitschakelen                                                                                                 |
| `frequency`                            | `string`  | `0 3 * * *`   | Optioneel Cron-schema voor de volledige Dreaming-cyclus                                                                               |
| `model`                                | `string`  | standaardmodel | Optionele override van het Dream Diary-subagentmodel                                                                                  |
| `phases.deep.maxPromotedSnippetTokens` | `number`  | `160`         | Maximaal geschat aantal tokens dat wordt behouden uit elk kortetermijnherinneringsfragment dat naar `MEMORY.md` wordt gepromoveerd; herkomstmetagegevens blijven zichtbaar |

### Voorbeeld

```json5
{
  plugins: {
    entries: {
      "memory-core": {
        subagent: {
          allowModelOverride: true,
          allowedModels: ["anthropic/claude-sonnet-4-6"],
        },
        config: {
          dreaming: {
            enabled: true,
            frequency: "0 3 * * *",
            model: "anthropic/claude-sonnet-4-6",
          },
        },
      },
    },
  },
}
```

<Note>
- Dreaming schrijft machinestatus naar `memory/.dreams/`.
- Dreaming schrijft voor mensen leesbare verhalende uitvoer naar `DREAMS.md` (of bestaande `dreams.md`).
- `dreaming.model` gebruikt de bestaande vertrouwenspoort voor pluginsubagents; stel `plugins.entries.memory-core.subagent.allowModelOverride: true` in voordat je dit inschakelt.
- Dream Diary probeert het eenmaal opnieuw met het standaardsessiemodel wanneer het geconfigureerde model niet beschikbaar is. Fouten met vertrouwen of toelatingslijsten worden geregistreerd en niet stilzwijgend opnieuw geprobeerd.
- Het beleid en de drempelwaarden voor de lichte/diepe/REM-fasen zijn intern gedrag, geen gebruikersconfiguratie.

</Note>

## Gerelateerd

- [Configuratiereferentie](/nl/gateway/configuration-reference)
- [Geheugenoverzicht](/nl/concepts/memory)
- [Geheugen doorzoeken](/nl/concepts/memory-search)
