---
read_when:
    - Je configureert de Plugin memory-lancedb
    - Je wilt langetermijngeheugen op basis van LanceDB met automatisch ophalen of automatisch vastleggen
    - Je gebruikt lokale OpenAI-compatibele embeddings, zoals Ollama
sidebarTitle: Memory LanceDB
summary: Configureer de officiële externe LanceDB-geheugenplugin, inclusief lokale Ollama-compatibele embeddings
title: Geheugen-LanceDB
x-i18n:
    generated_at: "2026-07-27T05:24:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdb7208925ac6c76430ee36dfcd9733041530e0f2ee175950b3cdb8010d67b24
    source_path: plugins/memory-lancedb.md
    workflow: 16
---

`memory-lancedb` is een officiële externe plugin die langetermijngeheugen opslaat in
LanceDB met vectorzoekopdrachten. De plugin kan vóór een modelbeurt automatisch relevante
herinneringen ophalen en na een antwoord automatisch belangrijke feiten vastleggen.

Gebruik de plugin voor een lokale vectordatabase, een OpenAI-compatibel embedding-eindpunt of
een geheugenopslag buiten de standaard ingebouwde geheugenbackend.

## Installatie

```bash
openclaw plugins install @openclaw/memory-lancedb
```

De plugin wordt gepubliceerd naar npm; deze is niet gebundeld in de runtime-image
van OpenClaw. Bij installatie wordt de pluginvermelding geschreven, de plugin ingeschakeld en
`plugins.slots.memory` overgeschakeld naar `memory-lancedb`. Als een andere plugin momenteel
het geheugenslot beheert, wordt die plugin met een waarschuwing uitgeschakeld.

<Note>
Aanvullende plugins zoals `memory-wiki` kunnen naast `memory-lancedb` worden uitgevoerd,
maar slechts één plugin beheert tegelijk het actieve geheugenslot.
</Note>

<Note>
`memory_recall` van LanceDB ontvangt niet de beveiligde autorisatie voor privétranscripten
die door `memory.search.rememberAcrossConversations` wordt gebruikt. Gebruik `autoRecall` van LanceDB
of het hulpprogramma `memory_recall` via
[geavanceerd Active Memory](/nl/concepts/active-memory#lancedb-memory).
`openclaw doctor` meldt wanneer Onthouden tussen gesprekken niet beschikbaar is
met de huidige geheugenprovider.
</Note>

## Snel aan de slag

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small",
          },
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

Start de Gateway opnieuw nadat je de pluginconfiguratie hebt gewijzigd en controleer vervolgens of deze is geladen:

```bash
openclaw gateway restart
openclaw plugins list
```

## Embeddingconfiguratie

`embedding` is vereist en moet ten minste één veld bevatten. `provider`
is standaard `openai`; `model` is standaard `text-embedding-3-small`.

| Veld                   | Type          | Opmerkingen                                                               |
| ---------------------- | ------------- | ------------------------------------------------------------------------- |
| `embedding.provider`   | tekenreeks    | Adapter-id, bijvoorbeeld `openai`, `github-copilot`, `ollama`. Standaard `openai`. |
| `embedding.model`      | tekenreeks    | Standaard `text-embedding-3-small`.                                       |
| `embedding.apiKey`     | tekenreeks    | Optioneel; ondersteunt uitbreiding van `${ENV_VAR}`.                       |
| `embedding.baseUrl`    | tekenreeks    | Optioneel; ondersteunt uitbreiding van `${ENV_VAR}`.                       |
| `embedding.dimensions` | geheel getal (>=1) | Vereist voor modellen die niet in de ingebouwde tabel staan (zie hieronder). |

Er bestaan twee aanvraagpaden:

- **Pad via provideradapter** (standaard): stel `embedding.provider` in en laat
  `embedding.apiKey`/`embedding.baseUrl` weg. De plugin herleidt het geconfigureerde
  autorisatieprofiel, de omgevingsvariabele of
  `models.providers.<provider>.apiKey` van de provider via dezelfde geheugenembeddingadapters
  die `memory-core` gebruikt. Dit is het pad voor `github-copilot`, `ollama`
  en elke andere gebundelde provider met ondersteuning voor embeddings.
- **Pad via directe OpenAI-compatibele client**: laat `embedding.provider` oningesteld
  (of `"openai"`) en stel `embedding.apiKey` plus `embedding.baseUrl` in. Gebruik dit
  voor een onbewerkt OpenAI-compatibel embedding-eindpunt zonder gebundelde
  provideradapter.

OpenAI Codex-/ChatGPT-OAuth is geen referentie voor OpenAI Platform-embeddings.
Gebruik voor OpenAI-embeddings een autorisatieprofiel met een OpenAI-API-sleutel, `OPENAI_API_KEY` of
`models.providers.openai.apiKey`. Gebruikers die uitsluitend OAuth gebruiken, moeten een andere
provider met embeddingondersteuning kiezen, zoals `github-copilot` of `ollama`.

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "github-copilot",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

Sommige OpenAI-compatibele embedding-eindpunten weigeren de parameter `encoding_format`;
andere negeren deze en retourneren altijd `number[]`. `memory-lancedb`
laat `encoding_format` weg uit aanvragen en accepteert zowel arrays met kommagetallen als
base64-gecodeerde float32-antwoorden, zodat beide antwoordvormen zonder configuratie werken.

### Dimensies

OpenClaw heeft alleen een ingebouwde dimensie voor `text-embedding-3-small` (1536) en
`text-embedding-3-large` (3072). Voor elk ander model is een expliciete
`embedding.dimensions` nodig, zodat LanceDB de vectorkolom kan maken, bijvoorbeeld
ZhiPu `embedding-3` met 2048 dimensies:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            apiKey: "${ZHIPU_API_KEY}",
            baseUrl: "https://open.bigmodel.cn/api/paas/v4",
            model: "embedding-3",
            dimensions: 2048,
          },
        },
      },
    },
  },
}
```

## Ollama-embeddings

Gebruik het pad via de gebundelde Ollama-provideradapter (`embedding.provider: "ollama"`).
Dit roept het systeemeigen `/api/embed`-eindpunt van Ollama aan en volgt dezelfde regels voor autorisatie en de basis-URL
als de provider [Ollama](/nl/providers/ollama).

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "ollama",
            baseUrl: "http://127.0.0.1:11434",
            model: "mxbai-embed-large",
            dimensions: 1024,
          },
          recallMaxChars: 400,
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

`mxbai-embed-large` staat niet in de ingebouwde dimensietabel, dus `dimensions` is
vereist. Verlaag voor kleine lokale embeddingmodellen `recallMaxChars` als de
lokale server fouten over de contextlengte retourneert.

## Limieten voor ophalen en vastleggen

| Instelling         | Standaard | Bereik                       | Van toepassing op                                          |
| ----------------- | --------- | ---------------------------- | ---------------------------------------------------------- |
| `recallMaxChars`  | `1000`  | 100-10000                    | Tekst die voor ophalen naar de embedding-API wordt verzonden. |
| `captureMaxChars` | `500`   | 100-10000                    | Berichtlengte die in aanmerking komt voor automatisch vastleggen. |
| `customTriggers`  | `[]`    | 0-50 items, elk <=100 tekens | Letterlijke zinnen waardoor automatisch vastleggen een bericht in overweging neemt. |

`recallMaxChars` begrenst de `before_prompt_build`-query voor automatisch ophalen, het
hulpprogramma `memory_recall`, het `memory_forget`-querypad en `openclaw ltm
search`. Automatisch ophalen maakt een embedding van het nieuwste gebruikersbericht uit de beurt en
valt alleen terug op de volledige prompt wanneer er geen gebruikersbericht aanwezig is, zodat kanaalmetadata
en grote promptblokken buiten de embeddingaanvraag blijven.

`captureMaxChars` bepaalt of een gebruikersbericht uit de `agent_end`-gebeurtenis
van de beurt kort genoeg is om voor automatisch vastleggen in aanmerking te komen; dit heeft geen invloed op
ophaalquery's.

`customTriggers` voegt letterlijke zinnen voor automatisch vastleggen toe zonder reguliere expressies. Ingebouwde
triggers omvatten veelgebruikte Engelse, Tsjechische, Chinese, Japanse en Koreaanse
geheugenzinnen (`remember`, `prefer`, `记住`, `覚えて`, `기억해` en vergelijkbare).

Automatisch vastleggen weigert ook tekst die lijkt op envelop-/transportmetadata,
promptinjectiepayloads of reeds geïnjecteerde `<relevant-memories>`-context,
en legt maximaal 3 herinneringen per agentbeurt vast.

Elke herinnering is eigendom van één agent. Ophalen, detectie van duplicaten, vastleggen,
weergeven, onbewerkte query's en verwijderen dwingen allemaal die eigenaar af voordat rijen worden
geretourneerd of gewijzigd. Een agent met `memory.search.enabled: false` in zijn `agents.entries.*`-
vermelding, of een agent die een uitgeschakelde zoekfunctie op het hoogste niveau overneemt, krijgt ook geen van de hulpprogramma's `memory_recall`, `memory_store`
of `memory_forget` en neemt niet deel aan automatisch ophalen of
vastleggen, zelfs wanneer de `autoRecall`/`autoCapture`-vlaggen op pluginniveau zijn ingeschakeld.

## Opdrachten

`memory-lancedb` registreert de CLI-naamruimte `ltm` wanneer deze is geïnstalleerd
(niet alleen wanneer deze het actieve geheugenslot beheert):

```bash
openclaw ltm list [--agent <id>] [--limit <n>] [--order-by-created-at]
openclaw ltm search <query> [--agent <id>] [--limit <n>]
openclaw ltm stats [--agent <id>]
```

`ltm query` voert rechtstreeks een query zonder vectoren uit op de LanceDB-tabel:

```bash
openclaw ltm query --agent research --cols id,text,createdAt --limit 20
openclaw ltm query --filter "category = 'preference'" --order-by createdAt:desc
```

| Vlag                              | Standaard                               | Opmerkingen                                                                                                                                |
| --------------------------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `--agent <id>`                    | geconfigureerde standaardagent          | Selecteert de privénaamruimte van de agent. Beschikbaar voor `list`, `search`, `query` en `stats`.                                                |
| `--cols <columns>`                | `id,text,importance,category,createdAt` | Door komma's gescheiden acceptatielijst met kolommen.                                                                                       |
| `--filter <condition>`            | geen                                    | Eén vergelijking op een uitvoerkolom, zoals `category = 'preference'` of `importance >= 0.8`. Tekenreekswaarden moeten tussen aanhalingstekens staan. |
| `--limit <n>`                     | `10`                                    | Positief geheel getal.                                                                                                                       |
| `--order-by <column>:<asc\|desc>` | geen                                    | Wordt in het geheugen gesorteerd nadat het filter is uitgevoerd; de sorteerkolom wordt automatisch aan de projectie toegevoegd en uit de uitvoer verwijderd als deze niet was aangevraagd. |

Agents krijgen drie hulpprogramma's van de actieve geheugenplugin:

- `memory_recall`: vectorzoekopdracht in opgeslagen herinneringen.
- `memory_store`: sla een feit, voorkeur, beslissing of entiteit op (weigert tekst
  die op een promptinjectiepayload lijkt; slaat bijna-duplicaten over).
- `memory_forget`: verwijder op basis van `memoryId`, of op basis van `query` (verwijdert automatisch één
  overeenkomst met een score boven 90%, of geeft anders kandidaat-id's weer om de keuze te verduidelijken).

## Opslag

LanceDB-gegevens worden standaard opgeslagen in `~/.openclaw/memory/lancedb`. Overschrijf dit met `dbPath`:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "~/.openclaw/memory/lancedb",
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

De plugin onderhoudt één LanceDB-tabel en slaat voor elke rij een genormaliseerde agenteigenaar
op. Dit is een opslaggrens, geen filter na het zoeken: agenteigendom wordt
toegepast vóór de vectorrangschikking en wordt opgenomen in predicaten voor weergeven, query's, tellen en verwijderen.
`ltm query --filter` accepteert één gevalideerde vergelijking op de
openbare uitvoerkolommen. De opslag bouwt die vergelijking afzonderlijk op van het
verplichte eigenaarspredicaat, zodat een filter de query niet kan uitbreiden naar een andere
agent.

Databases die vóór eigendom per agent zijn gemaakt, hebben geen betrouwbare herkomstgegevens per rij.
Bij een upgrade wijst `openclaw doctor --fix` die verouderde rijen eenmaal toe aan de
geconfigureerde standaardagent. Runtimetoegang wordt bij fouten geblokkeerd totdat die migratie is
voltooid; andere agents nemen de oude gedeelde rijen nooit over.

`storageOptions` accepteert sleutel-/waardeparen van tekenreeksen voor LanceDB-opslagbackends
(bijv. S3-compatibele objectopslag) en ondersteunt uitbreiding van `${ENV_VAR}`:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "s3://memory-bucket/openclaw",
          storageOptions: {
            access_key: "${AWS_ACCESS_KEY_ID}",
            secret_key: "${AWS_SECRET_ACCESS_KEY}",
            endpoint: "${AWS_ENDPOINT_URL}",
          },
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

## Runtimeafhankelijkheden en platformondersteuning

`memory-lancedb` is afhankelijk van het native pakket `@lancedb/lancedb`, dat eigendom is van het
pluginpakket (niet van de kerndistributie van OpenClaw). Bij het opstarten herstelt de Gateway geen
pluginafhankelijkheden. Als de native afhankelijkheid ontbreekt of niet kan worden geladen,
installeer of werk het pluginpakket dan opnieuw bij en start de Gateway opnieuw.

`@lancedb/lancedb` publiceert geen native build voor `darwin-x64` (Intel
Mac). Op dat platform registreert de plugin tijdens het laden dat LanceDB niet beschikbaar is;
gebruik de standaardgeheugenbackend, voer de Gateway uit op een ondersteund
platform/architectuur of schakel `memory-lancedb` uit.

## Problemen oplossen

### Invoerlengte overschrijdt de contextlengte

Het embeddingmodel heeft de opvraagquery geweigerd:

```text
memory-lancedb: opvragen mislukt: Fout: 400 de invoerlengte overschrijdt de contextlengte
```

Verlaag `recallMaxChars` en start vervolgens de Gateway opnieuw:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        config: {
          recallMaxChars: 400,
        },
      },
    },
  },
}
```

Controleer voor Ollama ook of de embeddingserver vanaf de Gateway-host bereikbaar is
via het native embedding-eindpunt:

```bash
curl http://127.0.0.1:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"mxbai-embed-large","input":"hello"}'
```

### Niet-ondersteund embeddingmodel

Zonder `embedding.dimensions` zijn alleen de ingebouwde OpenAI-embeddingdimensies
bekend (`text-embedding-3-small`, `text-embedding-3-large`). Stel voor elk ander
model `embedding.dimensions` in op de vectorgrootte die dat model rapporteert.

### Plugin wordt geladen, maar er verschijnen geen herinneringen

Controleer of `plugins.slots.memory` naar `memory-lancedb` verwijst en voer vervolgens uit:

```bash
openclaw ltm stats
openclaw ltm search "recent preference"
```

Als `autoCapture` is uitgeschakeld, haalt de plugin nog steeds bestaande herinneringen op, maar
slaat deze niet automatisch nieuwe op. Gebruik de tool `memory_store` of schakel
`autoCapture` in.

## Gerelateerd

- [Overzicht van geheugen](/nl/concepts/memory)
- [Active Memory](/nl/concepts/active-memory)
- [Geheugen doorzoeken](/nl/concepts/memory-search)
- [Geheugenwiki](/nl/plugins/memory-wiki)
- [Ollama](/nl/providers/ollama)
