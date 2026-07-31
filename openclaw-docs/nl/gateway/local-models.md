---
read_when:
    - Je wilt modellen aanbieden vanaf je eigen GPU-machine
    - Je koppelt LM Studio of een OpenAI-compatibele proxy aan
    - Je hebt richtlijnen nodig voor het veiligste lokale model
summary: Voer OpenClaw uit op lokale LLM's (LM Studio, vLLM, LiteLLM, aangepaste OpenAI-eindpunten)
title: Lokale modellen
x-i18n:
    generated_at: "2026-07-27T05:00:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: af76c9e97bd1d3c9665c347944511b4f466f0b620bb8af7b5f95b1e9145aadec
    source_path: gateway/local-models.md
    workflow: 16
---

Lokale modellen werken, maar stellen hogere eisen aan hardware, contextgrootte en bescherming tegen promptinjectie: kleine of agressief gekwantiseerde modellen kappen context af en slaan veiligheidsfilters aan providerzijde over. Deze pagina behandelt geavanceerdere lokale stacks en aangepaste OpenAI-compatibele servers. Begin voor de eenvoudigste aanpak met [LM Studio](/nl/providers/lmstudio) of [Ollama](/nl/providers/ollama) en `openclaw onboard`.

Zie [Lokale modelservices](/nl/gateway/local-model-services) voor lokale servers die alleen moeten starten wanneer een geselecteerd model ze nodig heeft.

## Minimale hardwarevereisten

Streef naar **2+ maximaal uitgeruste Mac Studios of een gelijkwaardige GPU-installatie (~$30k+)** voor een soepel werkende agentlus. Eén GPU van **24 GB** kan alleen lichtere prompts verwerken, met een hogere latentie. Gebruik altijd de **grootste variant / variant op volledige grootte die je kunt hosten** - kleine of sterk gekwantiseerde checkpoints verhogen het risico op promptinjectie (zie [Beveiliging](/nl/gateway/security)).

## Kies een backend

| Backend                                              | Gebruiken wanneer                                                                 |
| ---------------------------------------------------- | --------------------------------------------------------------------------------- |
| [ds4](/nl/providers/ds4)                                | Lokale DeepSeek V4 Flash op macOS Metal met OpenAI-compatibele toolaanroepen       |
| [LM Studio](/nl/providers/lmstudio)                     | Eerste lokale configuratie, GUI-lader, systeemeigen Responses API                  |
| LiteLLM / OAI-proxy / aangepaste OpenAI-compatibele proxy | Je een andere model-API ontsluit en OpenClaw die als OpenAI moet behandelen   |
| MLX / vLLM / SGLang                                  | Zelfgehoste bediening met hoge doorvoer en een OpenAI-compatibel HTTP-eindpunt     |
| [Ollama](/nl/providers/ollama)                          | CLI-workflow, modelbibliotheek, onderhoudsvrije systemd-service                    |

Gebruik `api: "openai-responses"` wanneer de backend dit ondersteunt (LM Studio doet dat). Gebruik anders `api: "openai-completions"`. Als `api` wordt weggelaten bij een aangepaste provider met een `baseUrl`, gebruikt OpenClaw standaard `openai-completions`.

<Warning>
**WSL2 + Ollama + NVIDIA/CUDA:** het officiële Ollama-installatieprogramma voor Linux schakelt een systemd-service met `Restart=always` in. Bij WSL2-GPU-configuraties kan automatisch starten tijdens het opstarten het laatst gebruikte model opnieuw laden en hostgeheugen vastzetten, waardoor de VM herhaaldelijk opnieuw wordt gestart. Zie [WSL2-crashlus](/nl/providers/ollama#troubleshooting).
</Warning>

## LM Studio + groot lokaal model (Responses API)

Dit is momenteel de beste lokale stack. Laad een groot model in LM Studio (een volledige Qwen-, DeepSeek- of Llama-build), schakel de lokale server in (standaard `http://127.0.0.1:1234`) en gebruik de Responses API om redeneringen gescheiden te houden van de definitieve tekst.

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "lmstudio/my-local-model": { alias: "Local" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Configuratiechecklist:

- Installeer LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- Download de **grootste beschikbare modelbuild** (vermijd "small"/sterk gekwantiseerde varianten), start de server en controleer of `http://127.0.0.1:1234/v1/models` deze vermeldt.
- Vervang `my-local-model` door de daadwerkelijke model-ID die in LM Studio wordt weergegeven.
- Houd het model geladen; koud laden voegt opstartlatentie toe.
- Pas `contextWindow`/`maxTokens` aan als jouw LM Studio-build afwijkt.
- Blijf voor WhatsApp de Responses API gebruiken, zodat alleen de definitieve tekst wordt verzonden.
- Behoud `models.mode: "merge"`, zodat gehoste modellen als terugvalopties beschikbaar blijven.

### Hybride configuratie: gehost primair model, lokale terugvaloptie

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Voor een lokale-eerstconfiguratie met een gehost vangnet verwissel je de volgorde van `primary`/`fallbacks` en behoud je hetzelfde `providers`-blok en `models.mode: "merge"`.

### Regionale hosting / gegevensroutering

Gehoste MiniMax/Kimi/GLM-varianten bestaan ook op OpenRouter met aan een regio gebonden eindpunten (bijvoorbeeld gehost in de VS). Kies de regionale variant om verkeer binnen het gekozen rechtsgebied te houden en behoud `models.mode: "merge"` voor Anthropic/OpenAI-terugvalopties. Alleen lokaal blijft de beste keuze voor privacy; gehoste regionale routering is de middenweg wanneer je providerfuncties nodig hebt, maar controle over de gegevensstroom wilt behouden.

## Andere OpenAI-compatibele lokale proxy's

MLX (`mlx_lm.server`), vLLM, SGLang, LiteLLM, OAI-proxy of een aangepaste Gateway werkt als deze een OpenAI-achtig `/v1/chat/completions`-eindpunt aanbiedt. Gebruik `openai-completions`, tenzij de backend expliciet ondersteuning voor `/v1/responses` documenteert.

```json5
{
  agents: {
    defaults: {
      model: { primary: "local/my-local-model" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Vermeldingen voor aangepaste/lokale providers vertrouwen hun exact geconfigureerde `baseUrl`-oorsprong voor beveiligde modelaanvragen, waaronder loopback-, LAN-, tailnet- en privé-DNS-hosts. Oorsprongen voor metadata/link-local worden altijd geblokkeerd. Voor aanvragen naar andere privé-oorsprongen is `models.providers.<id>.request.allowPrivateNetwork: true` nog steeds vereist; stel de vertrouwensvlag in op `false` om het vertrouwen in de exacte oorsprong uit te schakelen.

`models.providers.<id>.models[].id` is providerspecifiek - neem het providervoorvoegsel niet op. Voor een MLX-server die is gestart met `mlx_lm.server --model mlx-community/Qwen3-30B-A3B-6bit`:

- `models.providers.mlx.models[].id: "mlx-community/Qwen3-30B-A3B-6bit"`
- `agents.defaults.model.primary: "mlx/mlx-community/Qwen3-30B-A3B-6bit"`

Stel `input: ["text", "image"]` in voor lokale of via proxy aangeboden visiemodellen, zodat afbeeldingsbijlagen in agentbeurten worden ingevoegd. Interactieve onboarding van aangepaste providers herkent veelgebruikte ID's van visiemodellen en vraagt alleen naar onbekende namen; niet-interactieve onboarding gebruikt dezelfde herkenning, met `--custom-image-input` / `--custom-text-input` om deze te overschrijven.

Gebruik `models.providers.<id>.timeoutSeconds` voor trage lokale/externe modelservers voordat je `agents.defaults.timeoutSeconds` verhoogt. De providertime-out omvat verbinding, headers, streaming van de hoofdtekst en het volledig afbreken van beveiligd ophalen, uitsluitend voor HTTP-modelaanvragen - als de time-out van de agent/run lager is, verhoog die dan ook, omdat de providertime-out niet de volledige run kan verlengen.

<Note>
Voor aangepaste OpenAI-compatibele providers wordt een niet-geheime lokale markering zoals `apiKey: "ollama-local"` geaccepteerd wanneer `baseUrl` wordt omgezet naar loopback, een privé-LAN, `.local` of een kale hostnaam - OpenClaw behandelt deze als geldige lokale referentie in plaats van een ontbrekende sleutel te melden. Gebruik een echte waarde voor elke provider die een openbare hostnaam accepteert.
</Note>

Gedragsopmerkingen voor lokale/via proxy aangeboden `/v1`-backends:

- OpenClaw behandelt deze als OpenAI-compatibele proxyroutes, niet als systeemeigen OpenAI-eindpunten.
- Aanvraagvorming die uitsluitend voor systeemeigen OpenAI geldt, wordt niet toegepast: geen `service_tier`, geen Responses-`store`, geen vormgeving van OpenAI-compatibele payloads voor redeneringen, geen hints voor promptcaching.
- Verborgen OpenClaw-toeschrijvingsheaders (`originator`, `version`, `User-Agent`) worden niet ingevoegd bij aangepaste proxy-URL's.

Compat-declaraties gelden alleen voor het aangepaste eindpunt dat door deze providerregel wordt beschreven. Routes die in de catalogus bekend zijn, gebruiken in plaats daarvan mogelijkheden die eigendom zijn van de provider; zie de [handleiding voor mogelijkheden van aangepaste providers](/nl/gateway/config-tools#custom-provider-capability-declarations).

Compat-overschrijvingen voor strengere OpenAI-compatibele backends:

- **Alleen tekenreeksinhoud**: sommige servers accepteren alleen `messages[].content` als tekenreeks, geen gestructureerde arrays van inhoudsdelen. Stel `models.providers.<provider>.models[].compat.requiresStringContent: true` in.
- **Strikte berichtsleutels**: als de server berichtvermeldingen met meer dan `role`/`content` weigert, stel je `compat.strictMessageKeys: true` in.
- **Tooltekst tussen blokhaken**: sommige lokale modellen geven zelfstandige toolaanvragen als tekst tussen blokhaken weer, zoals `[tool_name]`, gevolgd door JSON en `[END_TOOL_REQUEST]`. OpenClaw zet deze alleen om in echte toolaanroepen wanneer de naam exact overeenkomt met een voor de beurt geregistreerde tool; anders blijft deze als verborgen, niet-ondersteunde tekst staan.
- **Ongestructureerde tekst die op een toolaanroep lijkt**: als een model JSON/XML/ReAct-achtige tekst genereert die op een toolaanroep lijkt, maar geen gestructureerde aanroep was, laat OpenClaw deze als tekst staan en registreert het een waarschuwing met de run-ID, provider/het model, het gedetecteerde patroon en, indien beschikbaar, de toolnaam. Dit is incompatibiliteit van de provider/het model, geen voltooide toolrun.
- **Toolgebruik afdwingen**: als tools als assistenttekst verschijnen (onbewerkte JSON/XML/ReAct of een lege `tool_calls`-array), controleer dan eerst of de chatsjabloon/parser van de server toolaanroepen ondersteunt. Als de parser alleen werkt wanneer toolgebruik wordt afgedwongen, overschrijf je de standaardproxywaarde van `tool_choice: "auto"` per model:

  ```json5
  {
    agents: {
      defaults: {
        models: {
          "local/my-local-model": {
            params: {
              extra_body: {
                tool_choice: "required",
              },
            },
          },
        },
      },
    },
  }
  ```

  Gebruik dit alleen wanneer elke normale beurt een tool moet aanroepen. Vervang `local/my-local-model` door de exacte verwijzing uit `openclaw models list`, of stel deze in via de CLI:

  ```bash
  openclaw config set agents.defaults.models '{"local/my-local-model":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
  ```

- **Extra redeneerinspanningen**: als een aangepast OpenAI-compatibel model OpenAI-redeneerinspanningen buiten het ingebouwde profiel accepteert, declareer je deze in het compat-blok van het model. Door `"xhigh"` toe te voegen, wordt deze voor die modelverwijzing beschikbaar in `/think xhigh`, sessiekiezers, Gateway-validatie en `llm-task`-validatie:

  ```json5
  {
    models: {
      providers: {
        local: {
          baseUrl: "http://127.0.0.1:8000/v1",
          apiKey: "sk-local",
          api: "openai-responses",
          models: [
            {
              id: "gpt-5.4",
              name: "GPT 5.4 via lokale proxy",
              reasoning: true,
              input: ["text"],
              cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
              contextWindow: 196608,
              maxTokens: 8192,
              compat: {
                supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
                reasoningEffortMap: { xhigh: "xhigh" },
              },
            },
          ],
        },
      },
    },
  }
  ```

## Kleinere of strengere backends

Als het model zonder problemen wordt geladen, maar volledige agentbeurten zich verkeerd gedragen, werk dan van boven naar beneden: controleer eerst het transport en beperk vervolgens het oppervlak.

1. **Controleer of het lokale model reageert** - geen tools, geen agentcontext:

   ```bash
   openclaw infer model run --local --model <provider/model> --prompt "Antwoord exact met: pong" --json
   ```

2. **Controleer Gateway-routering** - verzendt alleen de prompt en slaat het transcript, de AGENTS-bootstrap, de samenstelling van de context-engine, tools en gebundelde MCP-servers over, maar test nog steeds Gateway-routering, authenticatie en providerselectie:

   ```bash
   openclaw infer model run --gateway --model <provider/model> --prompt "Antwoord exact met: pong" --json
   ```

3. **Probeer de zuinige modus** als beide tests slagen, maar echte agentbeurten mislukken door ongeldige toolaanroepen of te grote prompts: stel `agents.defaults.experimental.localModelLean: true` in. Hiermee worden zware browser-, cron-, bericht-, mediageneratie-, spraak- en PDF-tools weggelaten, tenzij ze expliciet vereist zijn, en worden grotere toolcatalogi standaard achter gestructureerde Tool Search-besturingselementen geplaatst, terwijl `exec` direct zichtbaar blijft. Zie [Experimentele functies -> Zuinige modus voor lokale modellen](/nl/concepts/experimental-features#local-model-lean-mode) voor details en hoe je controleert of deze actief is.

4. **Schakel tools als laatste redmiddel volledig uit** door `models.providers.<provider>.models[].compat.supportsTools: false` voor dat model in te stellen - de agent wordt dan zonder toolaanroepen uitgevoerd.

5. **Daarna ligt de bottleneck upstream.** Als de backend na de zuinige modus en `supportsTools: false` nog steeds alleen bij grotere OpenClaw-uitvoeringen mislukt, ligt het resterende probleem meestal bij het model of de server zelf - contextvenster, GPU-geheugen, verwijdering uit de kv-cache of een backendbug - en niet bij de transportlaag van OpenClaw.

## Problemen oplossen

- **Kan de Gateway de proxy niet bereiken?** `curl http://127.0.0.1:1234/v1/models`.
- **LM Studio-model niet meer geladen?** Laad het opnieuw; een koude start is een veelvoorkomende oorzaak van 'vastlopen'.
- **Meldt de lokale server `terminated`, `ECONNRESET`, of sluit deze de stream halverwege een beurt?** OpenClaw registreert in de diagnostiek een `model.call.error.failureKind` met lage cardinaliteit plus een momentopname van het RSS-/heapgebruik van het OpenClaw-proces. Vergelijk bij geheugendruk in LM Studio/Ollama die tijdstempel met het serverlogboek of een macOS-crash-/jetsamlogboek om te bevestigen of de modelserver is beëindigd.
- **Contextfouten?** OpenClaw leidt de drempelwaarden voor de preflightcontrole van het contextvenster af van het gedetecteerde modelvenster (of het begrensde venster wanneer `agents.defaults.contextTokens` dit verlaagt), met een waarschuwing onder 20% en een ondergrens van **8k**, en een harde blokkering onder 10% met een ondergrens van **4k** (begrensd tot het effectieve contextvenster, zodat te grote modelmetadata een geldige gebruikerslimiet niet kan afwijzen). Verlaag `contextWindow` of verhoog de contextlimiet van de server/het model.
- **`messages[].content ... expected a string`?** Voeg `compat.requiresStringContent: true` toe aan die modelvermelding.
- **`validation.keys`, of 'berichtvermeldingen staan alleen `role` en `content` toe'?** Voeg `compat.strictMessageKeys: true` toe aan die modelvermelding.
- **Werken directe `/v1/chat/completions`-aanroepen, maar mislukt `openclaw infer model run --local` bij Gemma of een ander lokaal model?** Controleer eerst de provider-URL, de modelreferentie, de authenticatiemarkering en de serverlogboeken - `model run` slaat agenttools volledig over. Als `model run` slaagt, maar grotere agentbeurten mislukken, verklein dan het tooloppervlak met `localModelLean` of `compat.supportsTools: false`.
- **Verschijnen toolaanroepen als onbewerkte JSON-/XML-/ReAct-tekst, of retourneert de provider een lege `tool_calls`-array?** Voeg geen proxy toe die assistenttekst blindelings omzet in tooluitvoering - herstel eerst het chatsjabloon/de parser van de server. Als het model alleen werkt wanneer toolgebruik wordt afgedwongen, voeg dan de bovenstaande `params.extra_body.tool_choice: "required"`-override toe en gebruik die modelvermelding alleen voor sessies waarin bij elke beurt een toolaanroep wordt verwacht.
- **Veiligheid**: lokale modellen slaan filters aan de providerzijde over. Houd agents beperkt en laat Compaction ingeschakeld om het bereik van promptinjecties te beperken.

## Gerelateerd

- [Configuratiereferentie](/nl/gateway/configuration-reference)
- [Model-failover](/nl/concepts/model-failover)
