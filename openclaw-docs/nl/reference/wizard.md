---
read_when:
    - Een specifieke onboardingstap of vlag opzoeken
    - Onboarding automatiseren met de niet-interactieve modus
    - Onboardinggedrag debuggen
sidebarTitle: Onboarding Reference
summary: 'Volledig naslagwerk voor onboarding via de CLI: elke stap, vlag en elk configuratieveld'
title: Referentie voor onboarding
x-i18n:
    generated_at: "2026-07-27T05:34:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e5e7e42fa3fc1a6d85ad422d0d28dfeda233c89a4d7e97eee4fb974831816372
    source_path: reference/wizard.md
    workflow: 16
---

Dit is de volledige referentie voor `openclaw onboard`.
Zie [Onboarding (CLI)](/nl/start/wizard) voor een overzicht op hoofdlijnen. Zie [Referentie voor CLI-installatie](/nl/start/wizard-cli-reference) voor stapsgewijs
gedrag en uitvoer.

## Details van de flow (lokale modus)

<Steps>
  <Step title="Opnieuw instellen (optioneel)">
    - `--reset` stelt de status opnieuw in voordat de installatie wordt uitgevoerd; zonder deze optie behoudt het opnieuw uitvoeren van de onboarding
      de bestaande configuratie en wordt deze opnieuw als standaardinstelling gebruikt.
    - `--reset-scope` bepaalt wat `--reset` verwijdert: `config` (alleen het configuratiebestand
      ), `config+creds+sessions` (standaard) of `full` (verwijdert ook de
      werkruimte).
    - Als het configuratiebestand ongeldig is, stopt de onboarding en wordt je gevraagd eerst
      `openclaw doctor` uit te voeren en daarna de installatie opnieuw uit te voeren.
    - Bij opnieuw instellen wordt de status naar de prullenmand verplaatst (nooit rechtstreeks verwijderd).

  </Step>
  <Step title="Risico erkennen">
    - Bij de eerste uitvoering (of elke uitvoering voordat `wizard.securityAcknowledgedAt` is ingesteld)
      wordt je gevraagd te bevestigen dat je begrijpt dat agents krachtig zijn en volledige
      systeemtoegang riskant is.
    - `--non-interactive` vereist expliciet `--accept-risk`; zonder deze optie
      wordt de onboarding met een fout afgesloten in plaats van om invoer te vragen.
    - Bij interactieve uitvoeringen verschijnt een bevestigingsvraag in plaats van de vlag; bij weigering
      wordt de installatie geannuleerd.

  </Step>
  <Step title="Model/authenticatie">
    - **Anthropic-API-sleutel**: gebruikt `ANTHROPIC_API_KEY` indien aanwezig of vraagt om een sleutel en slaat deze vervolgens op voor gebruik door de daemon.
    - **Anthropic Claude CLI**: lokaal voorkeurspad wanneer er al een Claude CLI-aanmelding bestaat; OpenClaw ondersteunt als alternatief nog steeds authenticatie met een Anthropic-installatietoken.
    - **OpenAI Code-abonnement (Codex) (OAuth)**: browserflow; plak de `code#state`.
      - Bij een nieuwe installatie zonder primair model wordt `agents.defaults.model` via de Codex-runtime ingesteld op `openai/gpt-5.6-sol`.
    - **OpenAI Code-abonnement (Codex) (apparaatkoppeling)**: browserflow voor koppeling met een kort geldige apparaatcode.
      - Bij een nieuwe installatie zonder primair model wordt `agents.defaults.model` via de Codex-runtime ingesteld op `openai/gpt-5.6-sol`.
    - **OpenAI-API-sleutel**: gebruikt `OPENAI_API_KEY` indien aanwezig of vraagt om een sleutel en slaat deze vervolgens op in authenticatieprofielen.
      - Bij een nieuwe installatie zonder primair model wordt `agents.defaults.model` ingesteld op `openai/gpt-5.6`; de kale model-id voor de directe API wordt omgezet naar het Sol-niveau.
    - Bij het toevoegen van of opnieuw authenticeren bij OpenAI blijft een bestaand expliciet primair model behouden, waaronder `openai/gpt-5.5`. Als het account GPT-5.6 niet beschikbaar stelt, selecteer dan expliciet `openai/gpt-5.5`; OpenClaw verlaagt het model niet stilzwijgend.
    - **xAI OAuth**: aanmelding via de browser met een apparaatcode, zonder vereiste localhost-callback, zodat dit ook via SSH/Docker/VPS werkt (`--auth-choice xai-oauth`).
    - **xAI-API-sleutel**: vraagt om `XAI_API_KEY` (`--auth-choice xai-api-key`).
    - `--auth-choice xai-device-code` werkt nog steeds als alleen handmatig te gebruiken compatibiliteitsalias voor dezelfde xAI OAuth-flow met apparaatcode; gebruik `xai-oauth` voor nieuwe scripts.
    - **OpenCode**: vraagt om `OPENCODE_API_KEY` (of `OPENCODE_ZEN_API_KEY`, verkrijgbaar via https://opencode.ai/auth) en laat je de Zen- of Go-catalogus kiezen.
    - **Ollama**: biedt eerst **Cloud + lokaal**, **Alleen cloud** of **Alleen lokaal** aan. `Cloud only` vraagt om `OLLAMA_API_KEY` en gebruikt `https://ollama.com`; de hostgebaseerde modi vragen om de Ollama-basis-URL (standaard `http://127.0.0.1:11434`), detecteren beschikbare modellen en halen het geselecteerde lokale model indien nodig automatisch op; `Cloud + Local` controleert ook of die Ollama-host is aangemeld voor cloudtoegang.
    - Meer informatie: [Ollama](/nl/providers/ollama)
    - **API-sleutel**: slaat de sleutel voor je op.
    - **Vercel AI Gateway (proxy voor meerdere modellen)**: vraagt om `AI_GATEWAY_API_KEY`.
    - Meer informatie: [Vercel AI Gateway](/nl/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: vraagt om Account ID, Gateway ID en `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - Meer informatie: [Cloudflare AI Gateway](/nl/providers/cloudflare-ai-gateway)
    - **MiniMax**: de configuratie wordt automatisch geschreven; de standaardwaarde voor hosting is `MiniMax-M3`.
      Installatie met een API-sleutel gebruikt `minimax/...` en installatie met OAuth gebruikt
      `minimax-portal/...`.
    - Meer informatie: [MiniMax](/nl/providers/minimax)
    - **StepFun**: de configuratie wordt automatisch geschreven voor StepFun Standard of Step Plan op Chinese of wereldwijde eindpunten.
    - Standard gebruikt momenteel standaard `step-3.5-flash`; Step Plan bevat ook `step-3.5-flash-2603`.
    - Meer informatie: [StepFun](/nl/providers/stepfun)
    - **Synthetic (compatibel met Anthropic)**: vraagt om `SYNTHETIC_API_KEY`.
    - Meer informatie: [Synthetic](/nl/providers/synthetic)
    - **Moonshot (Kimi K2)**: de configuratie wordt automatisch geschreven.
    - **Kimi Coding**: de configuratie wordt automatisch geschreven.
    - Meer informatie: [Moonshot AI (Kimi + Kimi Coding)](/nl/providers/moonshot)
    - **Aangepaste provider**: werkt met eindpunten die compatibel zijn met OpenAI, OpenAI Responses of Anthropic. Niet-interactieve vlaggen: `--auth-choice custom-api-key`, `--custom-base-url`, `--custom-model-id`, `--custom-api-key` (optioneel; valt terug op `CUSTOM_API_KEY`), `--custom-provider-id` (optioneel; automatisch afgeleid van de basis-URL), `--custom-compatibility openai|openai-responses|anthropic` (standaard `openai`), `--custom-image-input` / `--custom-text-input` (overschrijft de afgeleide detectie van visiemodellen).
    - **Overslaan**: er is nog geen authenticatie geconfigureerd.
    - Kies een standaardmodel uit de gedetecteerde opties (of voer de provider en het model handmatig in). Kies voor de beste kwaliteit en een lager risico op promptinjectie het krachtigste beschikbare model van de nieuwste generatie in je providerstack.
    - De onboarding voert een modelcontrole uit en waarschuwt als het geconfigureerde model onbekend is of authenticatie ontbreekt.
    - De opslagmodus voor API-sleutels gebruikt standaard platte tekstwaarden in authenticatieprofielen. Gebruik `--secret-input-mode ref` om in plaats daarvan omgevingsvariabeleverwijzingen op te slaan (bijvoorbeeld `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`); de omgevingsvariabele waarnaar wordt verwezen, moet al zijn ingesteld, anders mislukt de onboarding onmiddellijk.
    - Authenticatieprofielen bevinden zich in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (API-sleutels + OAuth). `~/.openclaw/credentials/oauth.json` dient alleen voor het importeren van verouderde gegevens.
    - Meer informatie: [OAuth](/nl/concepts/oauth)
    <Note>
    Tip voor headless systemen/servers: voltooi OAuth op een computer met een browser en kopieer vervolgens
    `auth-profiles.json` van die agent (bijvoorbeeld
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`, of het overeenkomstige
    `$OPENCLAW_STATE_DIR/...`-pad) naar de Gateway-host. `credentials/oauth.json`
    is alleen een verouderde importbron.
    </Note>
  </Step>
  <Step title="Werkruimte">
    - Standaard `~/.openclaw/workspace` (configureerbaar).
    - Vult de werkruimte met de bestanden die nodig zijn voor het bootstrapritueel van de agent.
    - Volledige indeling van de werkruimte + back-uphandleiding: [Agentwerkruimte](/nl/concepts/agent-workspace)

  </Step>
  <Step title="Gateway">
    - Poort (standaard **18789**), binding, authenticatiemodus, blootstelling via Tailscale.
    - Authenticatieadvies: behoud **Token**, zelfs voor loopback, zodat lokale WS-clients zich moeten authenticeren.
    - In de tokenmodus biedt de interactieve installatie:
      - **Token in platte tekst genereren/opslaan** (standaard)
      - **SecretRef gebruiken** (optioneel)
      - Quickstart hergebruikt bestaande `gateway.auth.token` SecretRefs van de providers `env`, `file` en `exec` voor de onboardingcontrole en het bootstrappen van het dashboard.
      - Als die SecretRef is geconfigureerd maar niet kan worden omgezet, mislukt de onboarding vroegtijdig met een duidelijk herstelbericht in plaats van de runtime-authenticatie stilzwijgend te verzwakken.
    - In de wachtwoordmodus ondersteunt de interactieve installatie ook opslag als platte tekst of SecretRef.
    - Niet-interactief SecretRef-pad voor tokens: `--gateway-token-ref-env <ENV_VAR>`.
      - Vereist een niet-lege omgevingsvariabele in de procesomgeving van de onboarding.
      - Kan niet worden gecombineerd met `--gateway-token`.
    - Schakel authenticatie alleen uit als je elk lokaal proces volledig vertrouwt.
    - Bindingen buiten loopback vereisen nog steeds authenticatie.

  </Step>
  <Step title="Kanalen">
    - [WhatsApp](/nl/channels/whatsapp): optionele QR-aanmelding.
    - [Telegram](/nl/channels/telegram): bottoken.
    - [Discord](/nl/channels/discord): bottoken.
    - [Google Chat](/nl/channels/googlechat): JSON van het serviceaccount + webhookdoelgroep.
    - [Mattermost](/nl/channels/mattermost) (plugin): bottoken + basis-URL.
    - [Signal](/nl/channels/signal) (plugin): optionele installatie van `signal-cli` + accountconfiguratie.
    - [iMessage](/nl/channels/imessage): CLI-pad voor `imsg` + toegang tot de Messages-database; gebruik een SSH-wrapper wanneer de Gateway niet op een Mac wordt uitgevoerd.
    - Discord, Feishu, Microsoft Teams, QQ Bot, Slack en andere kanalen worden geleverd als
      plugins die de onboarding voor je kan installeren. Volledige catalogus: [Kanalen](/nl/channels).
    - DM-beveiliging: standaard wordt koppeling gebruikt. De eerste DM verstuurt een code; keur deze goed via `openclaw pairing approve <channel> <code>` of gebruik toelatingslijsten.

  </Step>
  <Step title="Zoeken op internet">
    - Kies een ondersteunde provider, zoals Brave, Codex (Hosted Search), DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Parallel, Perplexity, SearXNG of Tavily (of sla deze stap over).
    - Providers met een API kunnen omgevingsvariabelen of de bestaande configuratie gebruiken voor een snelle installatie; providers zonder sleutel gebruiken in plaats daarvan hun providerspecifieke vereisten.
    - Sla over met `--skip-search`.
    - Later configureren: `openclaw configure --section web`.

  </Step>
  <Step title="Daemon installeren">
    - macOS: LaunchAgent
      - Vereist een aangemelde gebruikerssessie; gebruik voor headless systemen een aangepaste LaunchDaemon (niet meegeleverd).
    - Linux (en Windows via WSL2): systemd-gebruikerseenheid
      - De onboarding probeert lingering in te schakelen via `loginctl enable-linger <user>`, zodat de Gateway actief blijft na afmelden.
      - Kan om sudo vragen (schrijft `/var/lib/systemd/linger`); eerst wordt het zonder sudo geprobeerd.
    - Native Windows: eerst een Scheduled Task; als het maken van de taak wordt geweigerd, valt OpenClaw terug op een aanmelditem per gebruiker in de map Startup en wordt de Gateway onmiddellijk gestart.
    - **Runtimeselectie:** Node is vereist omdat de canonieke opslag voor runtimestatus `node:sqlite` gebruikt. Verouderde Bun-services worden tijdens herstel naar Node gemigreerd.
    - Als tokenauthenticatie een token vereist en `gateway.auth.token` door SecretRef wordt beheerd, valideert de daemoninstallatie dit, maar worden de omgezette plattetekstwaarden van het token niet opgeslagen in de omgevingsmetadata van de supervisorservice.
    - Als tokenauthenticatie een token vereist en de geconfigureerde SecretRef voor het token niet kan worden omgezet, wordt de daemoninstallatie geblokkeerd met uitvoerbare instructies.
    - Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd en `gateway.auth.mode` niet is ingesteld, wordt de daemoninstallatie geblokkeerd totdat de modus expliciet is ingesteld.

  </Step>
  <Step title="Statuscontrole">
    - Start de Gateway (indien nodig) en voert `openclaw health` uit.
    - Tip: `openclaw status --deep` voegt de live statuscontrole van de Gateway toe aan de statusuitvoer, inclusief kanaalcontroles indien ondersteund (vereist een bereikbare Gateway).

  </Step>
  <Step title="Skills (aanbevolen)">
    - Leest de beschikbare skills en controleert de vereisten.
    - Laat je een nodebeheerder kiezen: **npm / pnpm / bun**.
    - Installeert automatisch optionele afhankelijkheden voor vertrouwde gebundelde skills (sommige gebruiken Homebrew op macOS).
    - Slaat skills over waarvan de vereiste Homebrew-, uv- of Go-installer niet beschikbaar is, groepeert ze met instructies voor handmatige installatie en verwijst je naar `openclaw doctor` zodra de vereiste is geïnstalleerd.

  </Step>
  <Step title="Voltooien">
    - Samenvatting + vervolgstappen, inclusief de vraag **Hoe wil je je agent laten uitkomen?** voor Terminal, Browser of later.

  </Step>
</Steps>

<Note>
Als er geen GUI wordt gedetecteerd, toont de onboarding SSH-instructies voor port forwarding naar de Control UI in plaats van een browser te openen.
Als de assets van de Control UI ontbreken, probeert de onboarding ze te bouwen; de fallback is `pnpm ui:build` (installeert automatisch de UI-afhankelijkheden).
</Note>

## Niet-interactieve modus

Gebruik `--non-interactive --accept-risk` om de onboarding te automatiseren of via scripts uit te voeren (de
vlag is de vereiste risicoverklaring; de onboarding wordt met een fout afgesloten
zonder deze vlag):

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

Voeg `--json` toe voor een machineleesbare samenvatting.

Gateway-token-SecretRef in niet-interactieve modus:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` en `--gateway-token-ref-env` sluiten elkaar wederzijds uit.

<Note>
`--json` impliceert **geen** niet-interactieve modus. Gebruik `--non-interactive --accept-risk` (en `--workspace`) voor scripts.
</Note>

Providerspecifieke opdrachtvoorbeelden staan in [CLI-automatisering](/nl/start/wizard-cli-automation#provider-specific-examples).
Gebruik deze referentiepagina voor de semantiek van vlaggen en de volgorde van stappen.

### Agent toevoegen (niet-interactief)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`main` is een gereserveerde agent-id en kan niet worden gebruikt voor `openclaw agents add`.

## RPC van de Gateway-wizard

De Gateway stelt de onboardingflow beschikbaar via RPC (`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`).
Clients (macOS-app, Control UI) kunnen stappen weergeven zonder de onboardinglogica opnieuw te implementeren.

## Signal instellen (signal-cli)

De onboarding detecteert of `signal-cli` zich in `PATH` bevindt en biedt aan dit te installeren als het ontbreekt:

- Linux x86-64: downloadt de officiële native GraalVM-build uit de GitHub-releases van `signal-cli` en slaat deze op onder `~/.openclaw/tools/signal-cli/<version>/`.
- macOS en andere architecturen: installeert in plaats daarvan via Homebrew.
- Native Windows: wordt nog niet ondersteund; voer de onboarding uit in WSL2 om het Linux-installatiepad te gebruiken.
- Schrijft in beide gevallen `channels.signal.transport.cliPath` met `kind: "managed-native"`.

## Wat de wizard schrijft

Gebruikelijke velden in `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.skipBootstrap` wanneer `--skip-bootstrap` wordt doorgegeven
- `agents.defaults.model` / `models.providers` (als Minimax is gekozen)
- `tools.profile` (lokale onboarding gebruikt standaard `"coding"` wanneer dit niet is ingesteld; bestaande expliciete waarden blijven behouden)
- `gateway.*` (modus, binding, authenticatie, Tailscale)
- `session.dmScope` (de onboarding behoudt expliciete waarden en laat dit anders oningesteld, zodat de standaardwaarde `"main"` alle directe berichten van alle kanalen in de doorlopende hoofdsessie van de agent bewaart—de standaardinstelling voor een persoonlijke agent. Gebruik voor gedeelde inboxen of inboxen met meerdere gebruikers `"per-channel-peer"`; `openclaw security audit` beveelt isolatie aan wanneer verkeer van directe berichten van meerdere gebruikers wordt gedetecteerd. Details: [CLI-installatiereferentie](/nl/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Toelatingslijsten voor directe berichten per kanaal wanneer je hiervoor kiest tijdens de kanaalprompts. Discord, Matrix, Microsoft Teams en Slack zetten namen waar mogelijk om naar id's; andere kanalen gebruiken rechtstreeks id's (bijvoorbeeld numerieke afzender-id's van Telegram of telefoonnummers van WhatsApp).
- `skills.install.nodeManager`
  - `setup --node-manager` accepteert `npm`, `pnpm` of `bun`.
  - Bij handmatige configuratie kan `yarn` nog steeds worden gebruikt door `skills.install.nodeManager` rechtstreeks in te stellen.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add` schrijft `agents.entries.*` en optioneel `bindings`.

WhatsApp-inloggegevens worden opgeslagen onder `~/.openclaw/credentials/whatsapp/<accountId>/`.
Actieve sessies en transcripten worden opgeslagen in
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. De map
`~/.openclaw/agents/<agentId>/sessions/` wordt gebruikt voor invoer voor verouderde migraties
en archief-/ondersteuningsartefacten.

Sommige kanalen worden geleverd als plugins. Wanneer je er tijdens de installatie een kiest, vraagt de onboarding
om deze te installeren (npm of een lokaal pad) voordat deze kan worden geconfigureerd.

## Gerelateerde documentatie

- Overzicht van de onboarding: [Onboarding (CLI)](/nl/start/wizard)
- Referentie voor CLI-installatie: [Referentie voor CLI-installatie](/nl/start/wizard-cli-reference)
- Onboarding van de macOS-app: [Onboarding](/nl/start/onboarding)
- Configuratiereferentie: [Gateway-configuratie](/nl/gateway/configuration)
- Providers: [WhatsApp](/nl/channels/whatsapp), [Telegram](/nl/channels/telegram), [Discord](/nl/channels/discord), [Google Chat](/nl/channels/googlechat), [Signal](/nl/channels/signal), [iMessage](/nl/channels/imessage)
- Skills: [Skills](/nl/tools/skills), [Skills-configuratie](/nl/tools/skills-config)
