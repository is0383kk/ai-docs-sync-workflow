---
read_when:
    - Je wilt weten welke functies mogelijk betaalde API's aanroepen
    - Je moet sleutels, kosten en inzicht in het gebruik controleren
    - Je legt de kostenrapportage van /status of /usage uit
summary: Controleer waaraan geld kan worden uitgegeven, welke sleutels worden gebruikt en hoe je het gebruik kunt bekijken
title: API-gebruik en kosten
x-i18n:
    generated_at: "2026-07-27T06:11:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22caad8b8fa168739563223b3663a04adceeef7e83576a53dc9cdf885a35750d
    source_path: reference/api-usage-costs.md
    workflow: 16
---

Overzicht van OpenClaw-functies die betaalde provider-API's kunnen aanroepen, waar elke functie haar referenties vandaan haalt en waar de resulterende kosten worden weergegeven.

## Waar kosten worden weergegeven

**`/status`** (momentopname per sessie)

- Toont het huidige sessiemodel, het contextgebruik en de tokens van het laatste antwoord.
- Voegt **geschatte kosten** voor het laatste antwoord toe wanneer OpenClaw gebruiksmetadata en lokale prijsgegevens voor het actieve model heeft, inclusief expliciet geprijsde providers zonder API-sleutel, zoals Bedrock `aws-sdk`-modellen.
- Als de live momentopname van de sessie weinig gegevens bevat, haalt `/status` de token-/cachetellers en het label van het actieve model op uit de nieuwste gebruiksvermelding in het transcript. Bestaande live waarden die niet nul zijn, krijgen voorrang op transcriptgegevens; een transcripttotaal ter grootte van de prompt kan nog steeds voorrang krijgen wanneer het opgeslagen totaal ontbreekt of kleiner is.

**`/usage`** (voettekst per bericht)

- `/usage full` voegt aan elk antwoord een gebruiksvoettekst toe, inclusief **geschatte kosten** wanneer lokale prijsgegevens zijn geconfigureerd en gebruiksmetadata beschikbaar zijn.
- `/usage tokens` toont alleen tokens. OAuth-/token- en CLI-runtimes in abonnementsvorm tonen alleen tokens, tenzij ze compatibele gebruiksmetadata plus een expliciete lokale prijs leveren.
- `/usage cost` toont een lokaal kostenoverzicht; `/usage off` schakelt de voettekst uit.
- Opmerking over Gemini CLI: zowel uitvoer van `stream-json` als van de oudere `json` bevat gebruiksgegevens onder `stats`. OpenClaw normaliseert `stats.cached` naar `cacheRead` en leidt indien nodig invoertokens af uit `stats.input_tokens - stats.cached`.

**Control UI → Usage** (analyse over meerdere sessies)

- Toont uit transcripten afgeleide totalen voor tokens en geschatte kosten voor het geselecteerde datumbereik, met uitsplitsingen per provider, model, agent, kanaal en tokentype.
- Vergelijkt kortere kalenderperioden die eindigen op de einddatum van het geselecteerde bereik. Ontbrekende datums tellen als kalenderdagen met nul gebruik; ze worden niet overgeslagen om een compacter bereik te creëren.
- Geeft de schaal van de dagelijkse grafiek rechtstreeks aan. Een `√`-badge betekent dat vierkantswortelcompressie dagen met weinig gebruik zichtbaar houdt.
- Deze totalen beschrijven de beschikbare lokale sessiegeschiedenis, niet een providerfactuur of een levenslang factureringsoverzicht. De UI waarschuwt wanneer prijsgegevens voor bepaalde vermeldingen ontbreken.

**CLI-gebruiksperioden** (providerquota, geen kosten per bericht)

- `openclaw status --usage` en `openclaw channels list` tonen **gebruiksperioden** van providers als `X% left`.
- Huidige providers voor gebruiksperioden: Anthropic, ClawRouter, DeepSeek, GitHub Copilot, Gemini CLI, MiniMax, OpenAI (omvat ChatGPT/Codex OAuth-/tokenauthenticatie), Xiaomi en z.ai. Zie [Modellen-CLI](/nl/cli/models) en [Kanalen-CLI](/nl/cli/channels) voor de volledige lijst met providers en vlaggen.
- De onbewerkte velden `usage_percent` / `usagePercent` van MiniMax rapporteren het resterende quotum, dus OpenClaw keert ze om; op aantallen gebaseerde velden krijgen voorrang wanneer ze aanwezig zijn. Als het antwoord een `model_remains`-array bevat, selecteert OpenClaw de vermelding voor het chatmodel, leidt het indien nodig het label van de periode af uit tijdstempels en neemt het de modelnaam op in het planlabel.
- Gebruiksauthenticatie komt waar mogelijk uit providerspecifieke hooks; anders valt OpenClaw terug op overeenkomende OAuth-/API-sleutelreferenties uit authenticatieprofielen, omgevingsvariabelen of configuratie.

Zie [Tokengebruik en kosten](/nl/reference/token-use) voor gedetailleerde voorbeelden.

<Note>
Anthropic heeft bevestigd dat hergebruik van Claude CLI (inclusief `claude -p`) een goedgekeurd integratiepatroon is, tenzij het een nieuw beleid publiceert. Anthropic biedt geen schatting in dollars per bericht, dus `/usage full` kan geen kosten voor Claude CLI-gebruik tonen.
</Note>

## Hoe sleutels worden gevonden

- **Authenticatieprofielen**: per agent, opgeslagen in `auth-profiles.json`.
- **Omgevingsvariabelen**: bijvoorbeeld `OPENAI_API_KEY`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`.
- **Configuratie**: `models.providers.*.apiKey`, `plugins.entries.*.config.webSearch.apiKey`, `plugins.entries.firecrawl.config.webFetch.apiKey`, `memory.search.*`, `talk.providers.*.apiKey`.
- **Skills**: `skills.entries.<name>.apiKey`, waarmee de sleutel naar de procesomgeving van de skill kan worden geëxporteerd.

## Functies die sleutels kunnen gebruiken

### Antwoorden van het kernmodel (chat + tools)

Elk antwoord of elke toolaanroep wordt uitgevoerd via de huidige modelprovider. Dit is de belangrijkste bron van gebruik en kosten, inclusief gehoste abonnementen die buiten de lokale UI van OpenClaw worden gefactureerd: OpenAI Codex, Alibaba Cloud Model Studio Coding Plan, MiniMax Coding Plan, Z.AI/GLM Coding Plan en het Claude-aanmeldingstraject van Anthropic met Extra Usage ingeschakeld.

Zie [Modellen](/nl/providers/models) voor prijsconfiguratie en [Tokengebruik en kosten](/nl/reference/token-use) voor de weergave.

### Mediabegrip (audio/afbeelding/video)

Binnenkomende media kunnen via een provider-API worden samengevat of getranscribeerd voordat de antwoordpijplijn wordt uitgevoerd. Providerondersteuning wordt per plugin geregistreerd en verandert wanneer plugins worden toegevoegd; zie [Mediabegrip](/nl/nodes/media-understanding) voor de huidige lijst en configuratie.

### Afbeeldingen en video's genereren

`image_generate` en `video_generate` routeren naar de beschikbare geauthenticeerde provider. Beide kunnen een door authenticatie ondersteunde standaardprovider afleiden wanneer hun `agents.defaults.mediaModels`-vermelding niet is ingesteld.

Zie [Afbeeldingen genereren](/nl/tools/image-generation) en [Video's genereren](/nl/tools/video-generation) voor de huidige lijst met providers.

### Geheugenembeddings en semantisch zoeken

Semantisch zoeken in het geheugen gebruikt embedding-API's wanneer `memory.search.provider` een externe adapter noemt (bijvoorbeeld `openai`, `gemini`, `voyage`, `mistral`, `deepinfra`, `github-copilot`, `amazon-bedrock`). `memory.search.provider = "lmstudio"` of `"ollama"` wordt uitgevoerd op een lokale/zelfgehoste server en brengt doorgaans geen kosten voor gehost gebruik met zich mee. `memory.search.provider = "local"` houdt alles op het apparaat zonder API-gebruik. Een optionele `memory.search.fallback`-provider kan fouten bij lokale embeddings opvangen.

Zie [Geheugen](/nl/concepts/memory).

### Tool voor zoeken op het web

`web_search` kan gebruikskosten veroorzaken, afhankelijk van de geselecteerde provider. Elke provider leest de sleutel eerst uit een omgevingsvariabele en vervolgens uit `plugins.entries.<id>.config.webSearch.apiKey`:

| Provider               | Omgevingsvariabele(n)                                                                                                                                                 |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brave Search           | `BRAVE_API_KEY`                                                                                                                                                        |
| DuckDuckGo             | zonder sleutel; niet-officieel, gebaseerd op HTML, geen facturering                                                                                                   |
| Exa                    | `EXA_API_KEY`                                                                                                                                                          |
| Firecrawl              | `FIRECRAWL_API_KEY`                                                                                                                                                    |
| Gemini (Google Search) | `GEMINI_API_KEY`                                                                                                                                                       |
| Grok (xAI)             | xAI OAuth-profiel of `XAI_API_KEY`                                                                                                                                     |
| Kimi (Moonshot)        | `KIMI_API_KEY` of `MOONSHOT_API_KEY`                                                                                                                                   |
| MiniMax Search         | `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN` of `MINIMAX_API_KEY`                                                                         |
| Ollama Web Search      | zonder sleutel voor een bereikbare, lokaal aangemelde host; rechtstreeks zoeken via `https://ollama.com` gebruikt `OLLAMA_API_KEY`; hosts met authenticatiebeveiliging hergebruiken de gebruikelijke bearer-authenticatie van de Ollama-provider |
| Parallel               | `PARALLEL_API_KEY`                                                                                                                                                     |
| Perplexity Search API  | `PERPLEXITY_API_KEY` of `OPENROUTER_API_KEY`                                                                                                                           |
| SearXNG                | `SEARXNG_BASE_URL`; zonder sleutel/zelfgehost, geen facturering voor gehost gebruik                                                                                   |
| Tavily                 | `TAVILY_API_KEY`                                                                                                                                                       |

Oudere `tools.web.search.*`-configuratiepaden worden nog steeds geladen via een compatibiliteitsshim, maar zijn niet langer de aanbevolen interface.

**Gratis tegoed van Brave Search**: elk abonnement bevat $5/maand aan gratis tegoed dat maandelijks wordt vernieuwd. Het Search-abonnement kost $5 per 1,000 aanvragen, dus het tegoed dekt 1,000 aanvragen/maand zonder kosten. Stel in het Brave-dashboard een gebruikslimiet in om onverwachte kosten te voorkomen.

Zie [Webtools](/nl/tools/web).

### Tool voor ophalen van het web (Firecrawl)

`web_fetch` kan Firecrawl aanroepen met sleutelvrije starterstoegang; voeg `FIRECRAWL_API_KEY` (of `plugins.entries.firecrawl.config.webFetch.apiKey`) toe voor hogere limieten. Als Firecrawl niet is geconfigureerd, valt de tool terug op rechtstreeks ophalen plus de gebundelde `web-readability`-plugin (geen betaalde API). Schakel `plugins.entries.web-readability.enabled` uit om lokale Readability-extractie over te slaan.

Zie [Webtools](/nl/tools/web).

### Momentopnamen van providergebruik (status/gezondheid)

`openclaw status --usage` en `openclaw models status --json` roepen gebruikseindpunten van providers aan om quotaperioden of de authenticatiestatus te tonen. De aanroepen vinden weinig plaats, maar gebruiken nog steeds provider-API's.

Zie [Modellen-CLI](/nl/cli/models).

### Beveiligde samenvatting bij Compaction

De beveiliging voor Compaction kan de sessiegeschiedenis samenvatten met het huidige model, waarbij provider-API's worden aangeroepen wanneer deze wordt uitgevoerd.

Zie [Sessiebeheer en Compaction](/nl/reference/session-management-compaction).

### Modelscan/-controle

`openclaw models scan` kan OpenRouter-modellen controleren en gebruikt `OPENROUTER_API_KEY` wanneer controle is ingeschakeld.

Zie [Modellen-CLI](/nl/cli/models).

### Praten (spraak)

De praatmodus kan ElevenLabs aanroepen wanneer dit is geconfigureerd: `ELEVENLABS_API_KEY` of `talk.providers.elevenlabs.apiKey`.

Zie [Praatmodus](/nl/nodes/talk).

### Skills (API's van derden)

Skills kunnen `apiKey` opslaan in `skills.entries.<name>.apiKey`. Als een skill die sleutel gebruikt voor een externe API, volgen de kosten de provider van de skill.

Zie [Skills](/nl/tools/skills).

## Gerelateerd

- [Tokengebruik en kosten](/nl/reference/token-use)
- [Promptcaching](/nl/reference/prompt-caching)
- [Gebruiksregistratie](/nl/concepts/usage-tracking)
