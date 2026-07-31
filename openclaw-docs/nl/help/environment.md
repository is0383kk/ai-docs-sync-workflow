---
read_when:
    - Je moet weten welke omgevingsvariabelen worden geladen en in welke volgorde.
    - Je onderzoekt ontbrekende API-sleutels in de Gateway.
    - Je documenteert providerauthenticatie of implementatieomgevingen
summary: Waar OpenClaw omgevingsvariabelen laadt en welke voorrangsvolgorde geldt
title: Omgevingsvariabelen
x-i18n:
    generated_at: "2026-07-27T05:35:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw haalt omgevingsvariabelen uit meerdere bronnen. De regel is: **bestaande waarden nooit overschrijven**.
Workspace-`.env`-bestanden zijn een bron met een lager vertrouwensniveau: OpenClaw negeert providerreferenties en beschermde runtime-instellingen uit workspace-`.env` voordat de prioriteitsvolgorde wordt toegepast.

## Prioriteitsvolgorde (van hoog naar laag)

1. **Procesomgeving** (wat het Gateway-proces al van de bovenliggende shell/daemon heeft).
2. **`.env` in de huidige werkmap** (dotenv-standaard; overschrijft niets; providerreferenties en beschermde runtime-instellingen worden genegeerd).
3. **Globale `.env`** op `~/.openclaw/.env` (ook bekend als `$OPENCLAW_STATE_DIR/.env`; aanbevolen voor API-sleutels van providers; overschrijft niets).
4. **Configuratieblok `env`** in `~/.openclaw/openclaw.json` (alleen toegepast als de waarde ontbreekt).
5. **Optionele import uit de aanmeldingsshell** (`env.shellEnv.enabled` of `OPENCLAW_LOAD_SHELL_ENV=1`), alleen toegepast voor ontbrekende verwachte sleutels.

Bij nieuwe Ubuntu-installaties die de standaardmap voor statusgegevens gebruiken, behandelt OpenClaw `~/.config/openclaw/gateway.env` ook als compatibiliteitsterugval na de globale `.env`. Als beide bestanden bestaan en van elkaar verschillen, behoudt OpenClaw `~/.openclaw/.env` en wordt een waarschuwing weergegeven.

Als het configuratiebestand volledig ontbreekt, wordt stap 4 overgeslagen; de shellimport wordt nog steeds uitgevoerd als deze is ingeschakeld.

## Ondersteunde variabelen voor beheerders

De onderstaande variabelen vormen het ondersteunde omgevingscontract voor beheerders. Niet-gedocumenteerde `OPENCLAW_*`-variabelen zijn interne implementatiedetails en kunnen zonder voorafgaande kennisgeving verdwijnen.

### Paden en instanties

| Variabele                | Doel                                                              |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | Overschrijft de thuismap die voor de standaardpaden van OpenClaw wordt gebruikt. |
| `OPENCLAW_STATE_DIR`     | Overschrijft de wijzigbare map voor statusgegevens.               |
| `OPENCLAW_CONFIG_PATH`   | Overschrijft het pad van het actieve configuratiebestand.         |
| `OPENCLAW_WORKSPACE_DIR` | Overschrijft de standaardworkspace van de agent.                   |
| `OPENCLAW_PROFILE`       | Selecteert een benoemd profiel en de geïsoleerde standaardwaarden ervan. |
| `OPENCLAW_GIT_DIR`       | Overschrijft de broncheckout die wordt gebruikt voor updates via het ontwikkelingskanaal. |
| `OPENCLAW_INCLUDE_ROOTS` | Staat toe dat `$include` vanuit aanvullende hoofdmappen wordt gevonden. |

### Gateway en authenticatie

| Variabele                   | Doel                                                            |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | Overschrijft de externe Gateway-URL die clients gebruiken.      |
| `OPENCLAW_GATEWAY_PORT`     | Overschrijft de lokale Gateway-poort.                            |
| `OPENCLAW_GATEWAY_TOKEN`    | Levert tokenauthenticatie voor Gateway-servers en -clients.      |
| `OPENCLAW_GATEWAY_PASSWORD` | Levert wachtwoordauthenticatie voor Gateway-servers en -clients. |

### Providerreferenties

De kern en meegeleverde providerplugins herkennen de volgende variabelen voor referenties en providerselectie. Geef de voorkeur aan de configuratie- of SecretRef-velden van elke provider wanneer je referenties met een beperkt bereik nodig hebt in plaats van één waarde voor het hele proces.

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY` en `Z_AI_API_KEY`.

Geïnstalleerde plugins van derden kunnen aanvullende referentievariabelen declareren in hun pluginmanifesten; deze variabelen zijn contracten van de Plugin die ze declareert, niet van de OpenClaw-kern.

### Logboekregistratie en diagnostiek

| Variabele                            | Doel                                                          |
| ------------------------------------ | ------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | Overschrijft de logniveaus voor bestanden en de console.      |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | Schakelt timingdiagnostiek voor modeltransport in.             |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | Selecteert diagnostiek van geredigeerde modelpayloads.         |
| `OPENCLAW_DEBUG_SSE`                 | Selecteert SSE-timing of diagnostiek voor het bekijken van gebeurtenissen. |
| `OPENCLAW_DEBUG_CODE_MODE`           | Schakelt diagnostiek voor code-modusoppervlakken in.           |
| `OPENCLAW_DIAGNOSTICS`               | Schakelt benoemde diagnostische vlaggen in of schakelt alle vlaggen uit met `0`. |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | Selecteert het JSONL-pad voor tijdlijndiagnostiek.             |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | Voegt event-loopsteekproeven toe aan tijdlijndiagnostiek.      |

### Functie- en runtimeschakelaars

| Variabele                            | Doel                                                                         |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | Importeert ontbrekende verwachte variabelen uit de aanmeldingsshell.          |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | Stelt de time-out voor de import uit de aanmeldingsshell in.                  |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | Schakelt snapshots van de exec-shell uit met `0`.             |
| `OPENCLAW_OFFLINE`                   | Voorkomt downloads van vastgezette binaire agenthulpprogramma's.             |
| `OPENCLAW_BROWSER_HEADLESS`          | Dwingt beheerde browserstarts met venster (`0`) of headless (`1`) af. |
| `OPENCLAW_DISABLE_BONJOUR`           | Dwingt Bonjour-adverteren aan (`0`) of uit (`1`). |
| `OPENCLAW_NO_AUTO_UPDATE`            | Schakelt het automatisch toepassen van updates uit.                           |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | Staat vertrouwde privé-DNS-`ws://`-verbindingen toe als noodoverride. |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | Staat meerdere Gateway-processen toe met behoud van eigendomsvergrendelingen per statusmap. |
| `OPENCLAW_SKIP_CHANNELS`             | Start de Gateway zonder kanaaltransporten voor probleemoplossing.             |
| `OPENCLAW_THEME`                     | Dwingt het TUI-palet af op `light` of `dark`.          |

## Providerreferenties en workspace-`.env`

Bewaar API-sleutels van providers niet uitsluitend in een workspace-`.env`. OpenClaw blokkeert een groot aantal sleutels voor providerreferenties en omleiding van eindpunten uit workspace-`.env`-bestanden, waaronder elke bekende omgevingsvariabele voor providerauthenticatie (bijvoorbeeld `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `PERPLEXITY_API_KEY`, `BRAVE_API_KEY`, `TAVILY_API_KEY`, `EXA_API_KEY`, `FIRECRAWL_API_KEY`), plus elke sleutel die eindigt op `_API_HOST`, `_BASE_URL`, `_ENDPOINT` of `_HOMESERVER`, en de volledige naamruimten `OPENCLAW_*`, `CLAWHUB_*`, `ANTHROPIC_API_KEY_*` en `OPENAI_API_KEY_*`.

Gebruik in plaats daarvan een van deze vertrouwde bronnen voor providerreferenties:

- De procesomgeving van de Gateway, zoals een shell, launchd-/systemd-eenheid, containergeheim of CI-geheim.
- Het globale runtime-dotenv-bestand op `~/.openclaw/.env` of `$OPENCLAW_STATE_DIR/.env`.
- Het configuratieblok `env` in `~/.openclaw/openclaw.json`.
- Optionele import uit de aanmeldingsshell wanneer `env.shellEnv.enabled` of `OPENCLAW_LOAD_SHELL_ENV=1` is ingeschakeld.

Als je eerder providersleutels of routeringswaarden voor eindpunten uitsluitend in een workspace-`.env` hebt opgeslagen, verplaats ze dan naar een van de bovenstaande vertrouwde bronnen. Workspace-`.env` kan nog steeds gewone projectvariabelen leveren die geen referenties, omleidingen van eindpunten, hostoverschrijvingen of `OPENCLAW_*`-runtime-instellingen zijn.

Zie [Workspace-`.env`-bestanden](/nl/gateway/security#workspace-env-files) voor de beveiligingsredenering.

## Configuratieblok `env`

Twee gelijkwaardige manieren om inline omgevingsvariabelen in te stellen (beide overschrijven niets):

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

Het configuratieblok `env` accepteert alleen letterlijke tekenreekswaarden. Het breidt
`file:...`-waarden niet uit; `XAI_API_KEY: "file:secrets/xai-api-key.txt"` wordt bijvoorbeeld
als precies die tekenreeks aan providers doorgegeven.

Gebruik voor providersleutels uit bestanden een SecretRef in het referentieveld dat
dit ondersteunt:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

Zie [Geheimenbeheer](/nl/gateway/secrets) en het
[SecretRef-referentieoppervlak](/nl/reference/secretref-credential-surface) voor
ondersteunde velden.

## Import van shellomgevingsvariabelen

`env.shellEnv` voert je aanmeldingsshell uit en importeert alleen **ontbrekende** verwachte sleutels:

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

Equivalenten als omgevingsvariabele:

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000` (standaard `15000`)

## Snapshots van de exec-shell

Op niet-Windows-Gateway-hosts gebruiken bash- en zsh-`exec`-opdrachten standaard een opstartsnapshot.
Stel `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` in de procesomgeving van de Gateway in om dit pad uit te schakelen.
De waarden `false`, `no` en `off` schakelen dit ook uit. `exec.env`-waarden per aanroep kunnen
snapshots niet in- of uitschakelen en de snapshotcache niet omleiden.

## Tijdens runtime geïnjecteerde omgevingsvariabelen

OpenClaw injecteert ook contextmarkeringen in gestarte onderliggende processen:

- `OPENCLAW_SHELL=exec`: ingesteld voor opdrachten die via de tool `exec` worden uitgevoerd.
- `OPENCLAW_SHELL=acp-client`: ingesteld voor `openclaw acp client` wanneer deze het ACP-bridgeproces start.
- `OPENCLAW_SHELL=tui-local`: ingesteld voor lokale TUI-shellopdrachten van `!`.
- `OPENCLAW_CLI=1`: ingesteld voor onderliggende processen die door het CLI-ingangspunt worden gestart.

Dit zijn runtimemarkeringen (geen vereiste gebruikersconfiguratie). Ze kunnen worden gebruikt in shell-/profiellogica
om contextspecifieke regels toe te passen.

## UI-omgevingsvariabelen

- `OPENCLAW_THEME=light`: dwing het lichte TUI-palet af wanneer je terminal een lichte achtergrond heeft.
- `OPENCLAW_THEME=dark`: dwing het donkere TUI-palet af.
- `COLORFGBG`: als je terminal dit exporteert, gebruikt OpenClaw de hint voor de achtergrondkleur om automatisch het TUI-palet te kiezen.

## Vervanging van omgevingsvariabelen in de configuratie

Je kunt rechtstreeks naar omgevingsvariabelen verwijzen in tekenreekswaarden van de configuratie met de syntaxis `${VAR_NAME}`:

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

Zie [Configuratie: vervanging van omgevingsvariabelen](/nl/gateway/configuration-reference#env-var-substitution) voor alle details.

## Secretrefs versus `${ENV}`-tekenreeksen

OpenClaw ondersteunt twee omgevingsgestuurde patronen:

- `${VAR}`-tekenreeksvervanging in configuratiewaarden.
- SecretRef-objecten (`{ source: "env", provider: "default", id: "VAR" }`) voor velden die geheimreferenties ondersteunen.

Beide worden tijdens activering vanuit de procesomgeving herleid. Details over SecretRef staan beschreven in [Beheer van geheimen](/nl/gateway/secrets).
Het configuratieblok `env` zelf herleidt geen SecretRefs of verkorte
`file:...`-waarden.

## Padgerelateerde omgevingsvariabelen

| Variabele                 | Doel                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | Overschrijf de thuismap die wordt gebruikt voor interne standaardpaden van OpenClaw (`~/.openclaw/`, agentmappen, sessies, referenties, onboarding van het installatieprogramma en de standaardontwikkelcheckout). Nuttig wanneer OpenClaw als speciale servicegebruiker wordt uitgevoerd. |
| `OPENCLAW_STATE_DIR`     | Overschrijf de statusmap (standaard `~/.openclaw`).                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | Overschrijf het pad naar het configuratiebestand (standaard `~/.openclaw/openclaw.json`).                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | Lijst met paden naar mappen waarin `$include`-richtlijnen bestanden buiten de configuratiemap mogen herleiden (standaard: geen — `$include` is beperkt tot de configuratiemap). Tildes worden uitgebreid.                                                         |

## Downloads van hulptools voor agents

Stel `OPENCLAW_OFFLINE=1` in om te voorkomen dat OpenClaw de vastgezette
hulpbinaire bestanden `fd` en `ripgrep` downloadt. Bestaande hulptools in de toolmap
van OpenClaw en werkende systeembinaire bestanden blijven bruikbaar; een ontbrekende hulptool blijft
niet beschikbaar in plaats van een netwerkverzoek te activeren.

## Logboekregistratie

| Variabele                         | Doel                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | Overschrijf het logniveau voor zowel bestanden als de console (bijv. `debug`, `trace`). Heeft voorrang op `logging.level` en `logging.consoleLevel` in de configuratie. Ongeldige waarden worden met een waarschuwing genegeerd. |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | Genereer gerichte timingdiagnostiek voor modelverzoeken en -antwoorden op niveau `info` zonder algemene debuglogboeken in te schakelen.                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | Diagnostiek van modelpayloads: `summary`, `tools` of `full-redacted`. `full-redacted` is begrensd en geredigeerd, maar kan prompt-/berichttekst bevatten.                                               |
| `OPENCLAW_DEBUG_SSE`             | Streamingdiagnostiek: `events` voor timing van de eerste gebeurtenis/voltooiing, `peek` om de eerste vijf geredigeerde SSE-gebeurtenissen op te nemen.                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | Diagnostiek van het modeloppervlak in codemodus, waaronder het verbergen van providertools en compacte afdwinging van besturing/richtlijnen.                                                                                  |

### `OPENCLAW_HOME`

Wanneer dit is ingesteld, vervangt `OPENCLAW_HOME` de thuismap van het systeem (`$HOME` / `os.homedir()`) voor interne standaardpaden van OpenClaw. Dit omvat de standaardstatusmap, het configuratiepad, agentmappen, referenties, de onboardingwerkruimte van het installatieprogramma en de standaardontwikkelcheckout die door `openclaw update --channel dev` wordt gebruikt.

**Voorrang:** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > terugval naar Termux-thuismap `PREFIX` op Android > `os.homedir()`

**Voorbeeld** (macOS LaunchDaemon):

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` kan ook worden ingesteld op een pad met een tilde (bijv. `~/svc`), dat vóór gebruik wordt uitgebreid via dezelfde terugvalketen voor de thuismap van het besturingssysteem.

Expliciete padvariabelen zoals `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH` en `OPENCLAW_GIT_DIR` hebben nog steeds voorrang. Taken voor besturingssysteemaccounts, zoals het detecteren van opstartbestanden voor de shell, het instellen van de pakketbeheerder en het uitbreiden van `~` door de host, kunnen nog steeds de echte systeemthuismap gebruiken.

## nvm-gebruikers: TLS-fouten van web_fetch

Als Node.js via **nvm** is geïnstalleerd (niet via de systeempakketbeheerder), gebruikt de ingebouwde `fetch()`
de gebundelde CA-opslag van nvm, waarin moderne root-CA's kunnen ontbreken (ISRG Root X1/X2 voor Let's Encrypt,
DigiCert Global Root G2 enz.). Hierdoor mislukt `web_fetch` op de meeste HTTPS-sites met `"fetch failed"`.

Op Linux detecteert OpenClaw nvm automatisch en past het de oplossing toe in de daadwerkelijke opstartomgeving:

- `openclaw gateway install` schrijft `NODE_EXTRA_CA_CERTS` naar de omgeving van de systemd-service
- het CLI-ingangspunt `openclaw` voert zichzelf opnieuw uit met `NODE_EXTRA_CA_CERTS` ingesteld voordat Node wordt gestart

**Handmatige oplossing (voor oudere versies of rechtstreekse starts met `node ...`):**

Exporteer de variabele voordat je OpenClaw start:

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

Vertrouw er voor deze variabele niet op dat je deze alleen naar `~/.openclaw/.env` schrijft; Node leest
`NODE_EXTRA_CA_CERTS` bij het opstarten van het proces.

## Verouderde omgevingsvariabelen

OpenClaw leest alleen `OPENCLAW_*`-omgevingsvariabelen. De verouderde voorvoegsels
`CLAWDBOT_*` en `MOLTBOT_*` uit eerdere releases worden stilzwijgend
genegeerd.

Als er bij het opstarten nog een of meer zijn ingesteld voor het Gateway-proces, genereert OpenClaw
één Node-verouderingswaarschuwing (`OPENCLAW_LEGACY_ENV_VARS`) met daarin de
gedetecteerde voorvoegsels en het totale aantal. Hernoem elke waarde door het
verouderde voorvoegsel te vervangen door `OPENCLAW_` (bijvoorbeeld `CLAWDBOT_GATEWAY_TOKEN` naar
`OPENCLAW_GATEWAY_TOKEN`); de oude namen hebben geen effect.

## Gerelateerd

- [Gateway-configuratie](/nl/gateway/configuration)
- [Veelgestelde vragen: omgevingsvariabelen en het laden van .env](/nl/help/faq#env-vars-and-env-loading)
- [Overzicht van modellen](/nl/concepts/models)
