---
read_when:
    - Je wilt lokale tekstinferentie zonder API-sleutel of modelserver
    - Je wilt embeddings voor geheugenzoekopdrachten uit een lokaal GGUF-model
    - Je configureert memory.search.provider = "local"
    - Je hebt de OpenClaw-plugin nodig die eigenaar is van de node-llama-cpp-runtime
sidebarTitle: llama.cpp Provider
summary: Voer lokale GGUF-tekstinferentie en geheugen-embeddings uit in OpenClaw met llama.cpp
title: llama.cpp-provider
x-i18n:
    generated_at: "2026-07-27T05:57:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 88e6d66943adcbc602421b8cc00359b3ed87357194c3ffaa845c1db7fbcd9c38
    source_path: plugins/llama-cpp.md
    workflow: 16
---

`llama-cpp` is de officiële externe providerplugin voor lokale GGUF-tekstinferentie
en embeddings binnen hetzelfde proces. Deze registreert tekstprovider `llama-cpp`,
embeddingprovider `local` en beheert de native runtime `node-llama-cpp`.

Installeer deze voordat je lokale inferentie of lokale geheugenembeddings gebruikt:

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

Het npm-hoofdpakket `openclaw` bevat `node-llama-cpp` niet. Door de
native afhankelijkheid in deze plugin te houden, wordt voorkomen dat normale npm-updates van OpenClaw
een handmatig geïnstalleerde runtime in de pakketmap van OpenClaw
verwijderen.

## Lokale tekstinferentie

Kies **Lokaal model (llama.cpp)** tijdens de interactieve onboarding. OpenClaw vraagt
om bevestiging voordat het standaardmodel wordt gedownload:

`hf:bartowski/Qwen_Qwen3-4B-Instruct-2507-GGUF/Qwen_Qwen3-4B-Instruct-2507-Q4_K_M.gguf`

Het bestand Qwen3 4B Instruct 2507 Q4_K_M is ongeveer 2,5 GB groot. Reken op circa 3 GB
RAM voor de modelgewichten, plus de context en overhead van de OpenClaw-runtime. De standaardcontext
wordt automatisch ingesteld met een limiet van 8.192 tokens, zodat deze praktisch bruikbaar blijft
op machines met 8 GB. Configureer alleen een grotere context als de machine voldoende
geheugen heeft.

De detectiecontrole tijdens de onboarding is alleen-lezen. llama.cpp wordt alleen automatisch
aangeboden wanneer het standaard of geconfigureerde GGUF-bestand al in de modelcache staat; tijdens
de detectie wordt nooit iets gedownload. Ollama en LM Studio blijven afzonderlijke opties voor lokale
services en behouden hun eigen detectieflows. llama.cpp handmatig kiezen
is de manier om de vraag voor het downloaden van het standaardmodel te krijgen.

De provider gebruikt de ingebouwde chatsjabloon van het GGUF-model en native
functieaanroepen van node-llama-cpp. Tekst wordt token voor token gestreamd. Toolaanroepen worden
voor uitvoering teruggestuurd naar OpenClaw in plaats van binnen node-llama-cpp te worden uitgevoerd.

### Een ander GGUF-model gebruiken

Voeg een model toe aan `models.providers.llama-cpp`. Plaats een lokaal pad of een volledige `hf:`-bestands-URI
in `params.modelPath`:

```json5
{
  models: {
    mode: "merge",
    providers: {
      "llama-cpp": {
        baseUrl: "local://llama-cpp",
        api: "openai-completions",
        params: {
          modelCacheDir: "~/.node-llama-cpp/models",
        },
        models: [
          {
            id: "my-local-model",
            name: "My local GGUF",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 2048,
            params: {
              modelPath: "~/Models/my-model.Q4_K_M.gguf",
              contextSize: 8192,
            },
            compat: { supportsTools: true },
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "llama-cpp/my-local-model" },
    },
  },
}
```

Bij inferentie wordt een ontbrekend model nooit impliciet gedownload. Download voor een aangepaste `hf:`-URI
eerst het GGUF-bestand naar `modelCacheDir`. Detectie gebruikt de
eigen alleen-lezen cache-resolver van node-llama-cpp, inclusief naamgeving voor repository's, branches en gesplitste bestanden.

## Configuratie van geheugenembeddings

Stel `memory.search.provider` in op `local`:

```json5
{
  memory: {
    search: {
      provider: "local",
      local: {
        modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

`local.modelPath` gebruikt standaard de hierboven getoonde `hf:`-URI (`embeddinggemma-300m-qat-Q8_0.gguf`).
Verwijs naar een andere `hf:`-URI of een lokaal `.gguf`-bestand om een ander
model te gebruiken. `local.modelCacheDir` overschrijft waar gedownloade modellen in de cache worden opgeslagen
(standaard: `~/.node-llama-cpp/models`) en `local.contextSize` accepteert een
geheel getal of `"auto"`.

Wanneer `local.contextSize` numeriek is, geeft de provider die vereiste ook
door aan de automatische plaatsing van GPU-lagen door node-llama-cpp. Hierdoor kan node-llama-cpp
het model en de embeddingcontext samen inpassen, terwijl de controles voor
geheugenveiligheid behouden blijven. Met `"auto"` behoudt node-llama-cpp de normale automatische plaatsing.

## Native runtime

Gebruik Node 24 voor het soepelste native installatieproces. Broncodecheck-outs die
pnpm gebruiken, moeten mogelijk de native afhankelijkheid goedkeuren en opnieuw bouwen:

```bash
pnpm approve-builds
pnpm rebuild node-llama-cpp
```

## Diagnostiek voor de geheugenruntime

Voer `openclaw memory status --deep` uit nadat de provider is geladen om
de geselecteerde backend en build, apparaatnamen, naar de GPU overgehevelde lagen, aangevraagde
contextgrootte en de laatst waargenomen momentopname van VRAM of gedeeld geheugen te inspecteren. De VRAM-
waarden bevatten een tijdstempel van de waarneming, omdat passieve statusuitlezingen het model niet
opnieuw laden en het apparaat niet pollen.

Dezelfde laatst bekende gegevens kunnen in `openclaw doctor` verschijnen wanneer de actieve
Gateway de lokale provider al heeft gebruikt. Een normale status- of doctor-opdracht
laadt geen model alleen om diagnostische gegevens te verzamelen.

## Probleemoplossing

Als `node-llama-cpp` ontbreekt of niet kan worden geladen, meldt OpenClaw de fout
met:

1. Installeer de plugin: `openclaw plugins install @openclaw/llama-cpp-provider`.
2. Gebruik Node 24 voor native installaties/updates.
3. Vanuit een pnpm-broncodecheck-out: `pnpm approve-builds`, daarna `pnpm rebuild node-llama-cpp`.

Gebruik voor lokale inferentie zonder een native afhankelijkheid binnen hetzelfde proces in plaats daarvan de provider
Ollama of LM Studio. Voor lokale embeddings met minder configuratie stel je
`memory.search.provider` in op een externe embeddingprovider zoals `lmstudio`,
`ollama`, `openai` of `voyage`.
