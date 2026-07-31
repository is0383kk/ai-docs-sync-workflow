---
read_when:
    - Modellen kiezen of wisselen, aliassen configureren
    - Foutopsporing voor model-failover / "Alle modellen zijn mislukt"
    - Inzicht in authenticatieprofielen en hoe je ze beheert
sidebarTitle: Models FAQ
summary: 'Veelgestelde vragen: standaardmodellen, selectie, aliassen, wisselen, failover en authenticatieprofielen'
title: 'FAQ: modellen en authenticatie'
x-i18n:
    generated_at: "2026-07-27T05:56:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

Vraag en antwoord over modellen en authenticatieprofielen. Zie voor installatie, sessies, Gateway, kanalen en
probleemoplossing de hoofd-[FAQ](/nl/help/faq).

## Modellen: standaardinstellingen, selectie, aliassen, wisselen

<AccordionGroup>
  <Accordion title='Wat is het "standaardmodel"?'>
    Stel dit in met:

    ```text
    agents.defaults.model.primary
    ```

    Modellen zijn `provider/model`-verwijzingen (voorbeeld: `openai/gpt-5.5`,
    `anthropic/claude-sonnet-4-6`). Stel `provider/model` altijd expliciet in. Als
    je de provider weglaat, probeert OpenClaw eerst een overeenkomst met een alias,
    vervolgens een unieke overeenkomst met een geconfigureerde provider voor die model-id
    en valt daarna terug op de geconfigureerde standaardprovider (verouderd
    compatibiliteitspad). Als die provider het geconfigureerde standaardmodel niet
    meer heeft, valt OpenClaw terug op de eerste geconfigureerde provider/het eerste
    geconfigureerde model in plaats van op een verouderde standaardinstelling.

  </Accordion>

  <Accordion title="Welk model raden jullie aan?">
    Gebruik het krachtigste model van de nieuwste generatie dat je providerstack
    aanbiedt, vooral voor agents met toegang tot tools of agents die niet-vertrouwde
    invoer verwerken — zwakkere of sterk gekwantiseerde modellen zijn kwetsbaarder
    voor promptinjectie en onveilig gedrag (zie [Beveiliging](/nl/gateway/security)).
    Routeer goedkopere modellen op basis van de agentrol naar routinematige chats
    met een laag risico.

    Routeer modellen per agent en gebruik sub-agents om langdurige taken te
    parallelliseren (elke sub-agent verbruikt eigen tokens). Zie
    [Modellen](/nl/concepts/models), [Sub-agents](/nl/tools/subagents),
    [MiniMax](/nl/providers/minimax) en [Lokale modellen](/nl/gateway/local-models).

  </Accordion>

  <Accordion title="Hoe wissel ik van model zonder mijn configuratie te wissen?">
    Wijzig alleen de modelvelden — vervang niet de volledige configuratie.

    - `/model` in de chat (per sessie, zie [Slash-opdrachten](/nl/tools/slash-commands))
    - `openclaw models set ...` (werkt alleen de modelconfiguratie bij)
    - `openclaw configure --section model` (interactief)
    - bewerk `agents.defaults.model` rechtstreeks in `~/.openclaw/openclaw.json`

    Inspecteer bij RPC-bewerkingen eerst met `config.schema.lookup` (genormaliseerd
    pad, beknopte schemadocumentatie, samenvattingen van onderliggende onderdelen)
    en geef daarna de voorkeur aan `config.patch` boven `config.apply`
    met een gedeeltelijk object. Als je de configuratie toch hebt overschreven,
    herstel je die vanuit een back-up of voer je `openclaw doctor` uit om deze
    te repareren.

    Documentatie: [Modellen](/nl/concepts/models), [Configureren](/nl/cli/configure),
    [Configuratie](/nl/cli/config), [Doctor](/nl/gateway/doctor).

  </Accordion>

  <Accordion title="Kan ik zelfgehoste modellen gebruiken (llama.cpp, vLLM, Ollama)?">
    Ja — Ollama is de eenvoudigste optie. Snelle installatie:

    1. Installeer Ollama vanaf `https://ollama.com/download`
    2. Haal een lokaal model op, bijvoorbeeld `ollama pull gemma4`
    3. Voer voor cloudmodellen ook `ollama signin` uit
    4. Voer `openclaw onboard` uit, kies `Ollama` en vervolgens `Local` of `Cloud + Local`

    `Cloud + Local` biedt je cloudmodellen naast je lokale Ollama-modellen;
    voor cloudmodellen zoals `kimi-k2.5:cloud` hoef je niets lokaal op te halen.
    Handmatig wisselen: `openclaw models list`, daarna `openclaw models set ollama/<model>`.

    Kleinere/sterk gekwantiseerde modellen zijn kwetsbaarder voor promptinjectie.
    Gebruik grote modellen voor elke bot met toegang tot tools; als je toch kleine
    modellen gebruikt, schakel dan sandboxing en strikte lijsten met toegestane
    tools in.

    Documentatie: [Ollama](/nl/providers/ollama), [Lokale modellen](/nl/gateway/local-models),
    [Modelproviders](/nl/concepts/model-providers), [Beveiliging](/nl/gateway/security),
    [Sandboxing](/nl/gateway/sandboxing).

  </Accordion>

  <Accordion title="Hoe wissel ik direct van model (zonder opnieuw op te starten)?">
    Stuur `/model <name>` als een afzonderlijk bericht. Zie
    [Slash-opdrachten](/nl/tools/slash-commands) voor de
    volledige lijst met opdrachten, inclusief de genummerde keuzelijst
    (`/model`, `/model
    list`, `/model 3`),
    `/model default` om een sessie-override te wissen en
    `/model status` voor details over het eindpunt/de API-modus.

    Dwing per sessie een specifiek authenticatieprofiel af met `@profile`:

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    Om een met `@profile` vastgezet profiel los te maken, voer je
    `/model` opnieuw uit zonder het achtervoegsel (bijvoorbeeld
    `/model anthropic/claude-opus-4-6`), of kies je de standaardinstelling uit
    `/model`. Gebruik `/model status` om het actieve
    authenticatieprofiel te bevestigen.

  </Accordion>

  <Accordion title="Als twee providers dezelfde model-id aanbieden, welke gebruikt /model dan?">
    `/model provider/model` selecteert exact die providerroute. Zo zijn
    `qianfan/deepseek-v4-flash` en `deepseek/deepseek-v4-flash` verschillende
    verwijzingen, ook al komt de model-id overeen — OpenClaw wisselt bij een
    overeenkomst met alleen de id niet stilzwijgend van provider.

    Een door de gebruiker geselecteerde `/model`-verwijzing hanteert
    een strikte fallback: als die provider/dat model niet meer beschikbaar is,
    mislukt het antwoord zichtbaar in plaats van terug te vallen op
    `agents.defaults.model.fallbacks`. Geconfigureerde fallbackketens blijven van toepassing
    op geconfigureerde standaardinstellingen, primaire modellen van Cron-taken
    en automatisch geselecteerde fallbackstatussen. Wanneer een uitvoering
    zonder sessie-override fallback mag gebruiken, probeert OpenClaw eerst de
    aangevraagde provider/het aangevraagde model, daarna de geconfigureerde
    fallbacks en vervolgens het geconfigureerde primaire model — dubbele kale
    model-id's springen dus nooit rechtstreeks terug naar de standaardprovider.

    Zie [Modellen](/nl/concepts/models) en [Model-failover](/nl/concepts/model-failover).

  </Accordion>

  <Accordion title="Kan ik GPT 5.5 gebruiken voor dagelijkse taken en Codex 5.5 voor programmeren?">
    Ja — modelkeuze en runtimekeuze staan los van elkaar:

    - **Native Codex-programmeeragent:** stel `agents.defaults.model.primary` in op
      `openai/gpt-5.5`. Meld je aan met `openclaw models auth login --provider
      openai` voor authenticatie
      via een ChatGPT-/Codex-abonnement.
    - **Rechtstreekse OpenAI API-taken buiten de agentlus:** configureer
      `OPENAI_API_KEY` voor afbeeldingen, embeddings, spraak, realtime en andere
      OpenAI API-oppervlakken buiten agents.
    - **Authenticatie met een API-sleutel voor de OpenAI-agent:** `/model openai/gpt-5.5`
      met een geordend API-sleutelprofiel van `openai`.
    - **Sub-agents:** routeer programmeertaken naar een op Codex gerichte agent
      met een eigen `openai/gpt-5.5`-model.

    Zie [Modellen](/nl/concepts/models) en [Slash-opdrachten](/nl/tools/slash-commands).

  </Accordion>

  <Accordion title="Hoe configureer ik de snelle modus voor GPT 5.5?">
    - **Per sessie:** stuur `/fast on` terwijl je `openai/gpt-5.5` gebruikt.
    - **Standaardinstelling per model:** stel
      `agents.defaults.models["openai/gpt-5.5"].params.fastMode` in op `true`.
    - **Automatische tijdslimiet:** `/fast auto` of `params.fastMode: "auto"` voert nieuwe
      modelaanroepen tot de tijdslimiet snel uit en voert latere nieuwe pogingen,
      fallbacks, toolresultaat- of vervolgaanroepen vervolgens zonder snelle modus
      uit. De tijdslimiet is standaard 60 seconden; overschrijf deze op het model
      met `params.fastAutoOnSeconds`.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    De snelle modus komt overeen met `service_tier = "priority"` bij native OpenAI
    Responses-aanvragen; bestaande waarden van `service_tier` blijven behouden
    en de snelle modus herschrijft `reasoning` of `text.verbosity` niet.
    Sessie-overrides van `/fast` gaan vóór de configuratiestandaarden.

    Zie [Denken en snelle modus](/nl/tools/thinking) en de sectie over de snelle
    modus onder Geavanceerde configuratie op de providerpagina
    [OpenAI](/nl/providers/openai).

  </Accordion>

  <Accordion title='Waarom zie ik "Model ... is not allowed" en krijg ik daarna geen antwoord?'>
    Als `agents.defaults.modelPolicy.allow` niet leeg is, wordt dit de
    **lijst met toegestane modellen** voor `/model`,
    sessie-overrides en `--model`. Als je een model buiten die lijst
    kiest, wordt dit weergegeven in plaats van een normaal antwoord:

    ```text
    Model override "provider/model" is not allowed by agents.defaults.modelPolicy.allow.
    ```

    Oplossing: voeg het exacte model of een providerwildcard zoals
    `"provider/*"` toe aan de genoemde `modelPolicy.allow`-lijst, verwijder
    die lijst of maak deze leeg, of kies een model uit `/model list`.
    Als de opdracht ook `--runtime codex` bevatte, werk je eerst de lijst met
    toegestane modellen bij en probeer je daarna dezelfde
    `/model provider/model --runtime codex`-opdracht opnieuw.

  </Accordion>

  <Accordion title='Waarom zie ik "Unknown model: minimax/MiniMax-M3"?'>
    Als je een oudere OpenClaw-release gebruikt, voer dan eerst een upgrade uit
    (of voer vanaf de broncode `main` uit) en start de Gateway
    opnieuw — `MiniMax-M3` staat mogelijk nog niet in de catalogus van je
    geïnstalleerde release. Anders is de MiniMax-provider niet geconfigureerd
    (er is geen providervermelding of authenticatieprofiel gevonden), waardoor
    het model niet kan worden gevonden. Zie de sectie Probleemoplossing op de
    providerpagina [MiniMax](/nl/providers/minimax) voor de volledige
    controlelijst met oplossingen, de tabel met provider-/model-id's en het
    voorbeeld van een configuratieblok.

  </Accordion>

  <Accordion title="Kan ik MiniMax als standaard gebruiken en OpenAI voor complexe taken?">
    Ja. Gebruik MiniMax als standaard en wissel per sessie van model — fallbacks
    zijn bedoeld voor fouten, niet voor "moeilijke taken", dus gebruik
    `/model` of een afzonderlijke agent.

    **Optie A: wisselen per sessie**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    Daarna `/model gpt`.

    **Optie B: afzonderlijke agents** — Agent A gebruikt standaard MiniMax,
    Agent B gebruikt standaard OpenAI; routeer per agent of gebruik
    `/agent` om te wisselen.

    Documentatie: [Modellen](/nl/concepts/models),
    [Routering met meerdere agents](/nl/concepts/multi-agent),
    [MiniMax](/nl/providers/minimax), [OpenAI](/nl/providers/openai).

  </Accordion>

  <Accordion title="Zijn opus / sonnet / gpt ingebouwde snelkoppelingen?">
    Ja — ingebouwde verkorte namen, die alleen worden toegepast wanneer het
    doelmodel bestaat in `agents.defaults.models`:

    | Alias | Verwijst naar |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    Je eigen alias met dezelfde naam overschrijft de ingebouwde alias.

  </Accordion>

  <Accordion title="Hoe definieer/overschrijf ik snelkoppelingen (aliassen) voor modellen?">
    Aliassen staan in `agents.defaults.models.<modelId>.alias`:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    Daarna verwijst `/model sonnet` (of `/<alias>` wanneer
    ondersteund) naar die model-id.

  </Accordion>

  <Accordion title="Hoe voeg ik modellen toe van andere providers, zoals OpenRouter of Z.AI?">
    OpenRouter (betaling per token; veel modellen):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (GLM-modellen):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    Als de providersleutel voor een provider/model waarnaar wordt verwezen
    ontbreekt, treedt tijdens runtime een authenticatiefout op (bijvoorbeeld
    `No API key found for provider "zai"`).

    **Geen API-sleutel gevonden voor provider na het toevoegen van een nieuwe agent**

    Een nieuwe agent heeft een lege authenticatieopslag — authenticatie wordt
    per agent opgeslagen op:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Oplossing: voer `openclaw agents add <id>` uit en configureer authenticatie in de wizard, of
    kopieer alleen overdraagbare statische `api_key`/`token`-profielen uit de opslag van de
    hoofdagent. Meld je voor OAuth aan vanuit de nieuwe agent wanneer die een
    eigen account nodig heeft. Zie [Routering met meerdere agents](/nl/concepts/multi-agent) voor de
    volledige regels voor hergebruik van `agentDir` en het delen van aanmeldgegevens — hergebruik
    `agentDir` nooit tussen agents.

  </Accordion>
</AccordionGroup>

## Model-failover en "Alle modellen zijn mislukt"

<AccordionGroup>
  <Accordion title="Hoe werkt failover?">
    Twee fasen:

    1. **Rotatie van authenticatieprofielen** binnen dezelfde provider.
    2. **Model-fallback** naar het volgende model in `agents.defaults.model.fallbacks`.

    Voor mislukte profielen gelden afkoelperioden (exponentiële back-off), zodat OpenClaw
    blijft reageren wanneer een provider snelheidsbeperkingen oplegt of tijdelijk faalt.

    De bucket voor snelheidsbeperkingen omvat meer dan alleen `429`: `Too many concurrent
    requests`, `ThrottlingException`, `concurrency limit reached`, `workers_ai
    ... quota limit exceeded`, `resource exhausted` en periodieke
    limieten voor gebruiksvensters (`weekly/monthly limit reached`) tellen allemaal als
    snelheidsbeperkingen waarvoor failover nodig is.

    Factureringsreacties zijn niet altijd `402`, en sommige `402`s blijven in de
    bucket voor tijdelijke fouten/snelheidsbeperkingen in plaats van in het factureringstraject. Expliciete
    factureringstekst bij `401`/`403` kan nog steeds naar facturering worden gerouteerd; providerspecifieke
    tekstmatchers (bijv. OpenRouter `Key limit exceeded`) blijven beperkt tot hun
    eigen provider. Een `402` die lijkt op een opnieuw te proberen limiet voor een gebruiksvenster of
    uitgavenlimiet voor een organisatie/werkruimte (`daily limit reached, resets tomorrow`,
    `organization spending limit exceeded`) wordt behandeld als `rate_limit`, niet als een
    langdurige uitschakeling wegens facturering.

    Fouten door contextoverschrijding blijven volledig buiten het fallback-pad — kenmerken
    zoals `request_too_large`, `input exceeds the maximum number of tokens`,
    `input token count exceeds the maximum number of input tokens`, `input is
    too long for the model` of `ollama error: context length exceeded` leiden tot
    Compaction/opnieuw proberen in plaats van door te gaan met model-fallback.

    Algemene serverfouttekst is specifieker dan "alles met unknown/error
    erin". Providerspecifieke tijdelijke vormen die wel als failover-
    signalen tellen: Anthropic met alleen `An unknown error occurred`, OpenRouter met alleen
    `Provider returned error`, stopredenfouten zoals `Unhandled stop reason:
    error`, JSON-`api_error`-payloads met tijdelijke servertekst (`internal
    server error`, `unknown error, 520`, `upstream error`, `backend error`)
    en fouten wegens een bezette provider zoals `ModelNotReadyException` wanneer de providercontext
    overeenkomt. Algemene interne fallback-tekst zoals `LLM request failed
    with an unknown error.` blijft conservatief en activeert op
    zichzelf geen fallback.

  </Accordion>

  <Accordion title='Wat betekent "Geen aanmeldgegevens gevonden voor profiel anthropic:default"?'>
    De authenticatieprofiel-id `anthropic:default` heeft geen aanmeldgegevens in de
    verwachte authenticatieopslag.

    **Controlelijst voor de oplossing:**

    - Controleer waar profielen staan — huidig:
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`; verouderd:
      `~/.openclaw/agent/*` (gemigreerd door `openclaw doctor`).
    - Controleer of de Gateway je omgevingsvariabele laadt. `ANTHROPIC_API_KEY` die alleen in
      je shell is ingesteld, bereikt geen Gateway die via systemd/launchd wordt uitgevoerd — plaats deze in
      `~/.openclaw/.env` of schakel `env.shellEnv` in.
    - Controleer of je de juiste agent bewerkt — configuraties met meerdere agents hebben
      meerdere `auth-profiles.json`-bestanden.
    - Voer `openclaw models status` uit om geconfigureerde modellen en de
      authenticatiestatus van providers te bekijken.

    **Voor "Geen aanmeldgegevens gevonden voor profiel anthropic" (zonder e-mailachtervoegsel):**

    De uitvoering is vastgezet op een Anthropic-profiel dat de Gateway niet kan vinden.

    - Gebruik Claude CLI: voer `openclaw models auth login --provider anthropic
      --method cli --set-default` uit op de gatewayhost.
    - Liever een API-sleutel: plaats `ANTHROPIC_API_KEY` in
      `~/.openclaw/.env` op de gatewayhost en wis vervolgens elke vastgezette volgorde
      die het ontbrekende profiel afdwingt:

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - Externe modus: authenticatieprofielen staan op de gatewaymachine, niet op je
      laptop — controleer of je de opdrachten daar uitvoert.

  </Accordion>

  <Accordion title="Waarom probeerde het ook Google Gemini en mislukte dat?">
    Als je modelconfiguratie Google Gemini als fallback bevat (of je
    bent overgeschakeld naar een Gemini-verkorte notatie), probeert OpenClaw dit tijdens fallback. Als er geen
    Google-aanmeldgegevens zijn geconfigureerd, resulteert dit in `No API key found for provider
    "google"`. Oplossing: voeg Google-authenticatie toe of verwijder Google-modellen uit
    `agents.defaults.model.fallbacks`/aliassen.

    **LLM-verzoek geweigerd: handtekening voor denkproces vereist (Google Antigravity)**

    Oorzaak: de sessiegeschiedenis bevat denkblokken zonder handtekeningen (vaak
    door een afgebroken/gedeeltelijke stream); Google Antigravity vereist handtekeningen
    voor denkblokken. OpenClaw verwijdert niet-ondertekende denkblokken voor Google
    Antigravity Claude; als het probleem blijft optreden, start je een nieuwe sessie of stel je
    `/thinking off` in voor die agent.

  </Accordion>
</AccordionGroup>

## Authenticatieprofielen: wat ze zijn en hoe je ze beheert

Gerelateerd: [/concepts/oauth](/nl/concepts/oauth) (OAuth-flows, tokenopslag, patronen voor meerdere accounts)

<AccordionGroup>
  <Accordion title="Wat is een authenticatieprofiel?">
    Een benoemd record met aanmeldgegevens (OAuth of API-sleutel) dat aan een provider is gekoppeld en wordt opgeslagen
    op:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    Inspecteer opgeslagen profielen zonder geheimen weer te geven: `openclaw models auth
    list` (optioneel `--provider <id>` of `--json`). Zie
    [Modellen-CLI](/nl/cli/models#auth-profiles).

  </Accordion>

  <Accordion title="Wat zijn gebruikelijke profiel-id's?">
    Met providerprefix: `anthropic:default` (gebruikelijk als er geen e-mailidentiteit
    bestaat), `anthropic:<email>` voor OAuth-identiteiten of een aangepaste id die je
    kiest (bijv. `anthropic:work`).

  </Accordion>

  <Accordion title="Kan ik bepalen welk authenticatieprofiel eerst wordt geprobeerd?">
    Ja. De configuratie `auth.order.<provider>` stelt de rotatievolgorde per provider in
    (alleen metadata — er worden geen geheimen opgeslagen).

    OpenClaw kan een profiel overslaan tijdens een korte **afkoelperiode** (snelheidsbeperkingen,
    time-outs, authenticatiefouten) of een langere toestand **uitgeschakeld**
    (facturering/onvoldoende tegoed). Inspecteer dit met `openclaw models status
    --json` en controleer `auth.unusableProfiles`. Afkoelperioden voor snelheidsbeperkingen kunnen
    modelspecifiek zijn — een profiel dat voor één model afkoelt, kan nog steeds een
    verwant model bij dezelfde provider bedienen; vensters voor facturering/uitschakeling blokkeren het
    volledige profiel.

    Stel een volgorde-overschrijving per agent in (opgeslagen in `auth-state.json` van die agent):

    ```bash
    # Defaults to the configured default agent (omit --agent)
    openclaw models auth order get --provider anthropic

    # Lock rotation to a single profile
    openclaw models auth order set --provider anthropic anthropic:default

    # Or set an explicit order (fallback within provider)
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # Clear override (fall back to config auth.order / round-robin)
    openclaw models auth order clear --provider anthropic

    # Target a specific agent
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    Controleer wat daadwerkelijk wordt geprobeerd: `openclaw models status --probe`. Een
    opgeslagen profiel dat niet in een expliciete volgorde staat, meldt
    `excluded_by_auth_order` in plaats van stilzwijgend te worden geprobeerd.

  </Accordion>

  <Accordion title="OAuth versus API-sleutel - wat is het verschil?">
    - **OAuth-/CLI-aanmelding** gebruikt vaak abonnementstoegang wanneer de
      provider dit ondersteunt. Voor Anthropic gebruikt de Claude CLI-backend van OpenClaw
      Claude Code `claude -p`, dat Anthropic momenteel behandelt als
      Agent SDK-/programmatisch gebruik dat van de gebruikslimieten van het abonnement wordt afgetrokken —
      zie [Anthropic](/nl/providers/anthropic) voor de huidige status van de factureringspauze
      en bronlinks.
    - **API-sleutels** gebruiken facturering per token.

    De wizard ondersteunt Anthropic Claude CLI, OpenAI Codex OAuth en API-
    sleutels.

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Veelgestelde vragen](/nl/help/faq) — de belangrijkste veelgestelde vragen
- [Veelgestelde vragen — snel aan de slag en configuratie bij de eerste uitvoering](/nl/help/faq-first-run)
- [Modelselectie](/nl/concepts/model-providers)
- [Model-failover](/nl/concepts/model-failover)
