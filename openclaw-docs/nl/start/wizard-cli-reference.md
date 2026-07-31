---
read_when:
    - Je hebt gedetailleerd gedrag nodig voor een specifieke stap van `openclaw onboard`
    - Je debugt onboardingresultaten of integreert onboardingclients
sidebarTitle: CLI reference
summary: 'Stapsgewijs gedrag van openclaw onboard: wat elke stap doet, welke configuratie deze schrijft en de interne werking'
title: Referentie voor CLI-configuratie
x-i18n:
    generated_at: "2026-07-27T06:13:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 41bb9243ac7276b383274f4c27e3782b29e8ecf9d883229a44e3ab59aca5a34f
    source_path: start/wizard-cli-reference.md
    workflow: 16
---

Deze pagina behandelt stapsgewijs het onboardinggedrag, de uitvoer en de interne werking.
Zie [Onboarding (CLI)](/nl/start/wizard) voor een rondleiding. Zie voor de volledige
referentie van CLI-vlaggen (elke `--flag`, niet-interactieve voorbeelden,
providerspecifieke opdrachten) [`openclaw onboard`](/nl/cli/onboard).

## Wat de wizard doet

De lokale modus (standaard) begeleidt je bij:

- Model- en authenticatieconfiguratie (Anthropic, OAuth voor een OpenAI Code-abonnement, xAI, OpenCode, aangepaste eindpunten en meer authenticatieflows die eigendom zijn van providers)
- Werkruimtelocatie en bootstrapbestanden
- Gateway-instellingen (poort, binding, authenticatie, Tailscale)
- Kanalen en providers (Discord, Feishu, Google Chat, iMessage, Mattermost, Microsoft Teams, QQ Bot, Signal, Slack, Telegram, WhatsApp en andere gebundelde kanalen of Plugin-kanalen)
- Provider voor zoeken op het web (optioneel)
- Daemoninstallatie (LaunchAgent, systemd-gebruikerseenheid of systeemeigen Windows Scheduled Task met terugval op de map Startup)
- Statuscontrole
- Configuratie van Skills

De externe modus configureert deze machine om verbinding te maken met een Gateway
elders. Er wordt niets op de externe host geïnstalleerd of gewijzigd.

## Details van de lokale flow

<Steps>
  <Step title="Detectie van bestaande configuratie">
    - Als `~/.openclaw/openclaw.json` bestaat, kies je **Huidige waarden behouden**, **Controleren en bijwerken** of **Opnieuw instellen vóór configuratie**.
    - Als je de wizard opnieuw uitvoert, wordt niets gewist tenzij je expliciet Opnieuw instellen kiest (of `--reset` doorgeeft).
    - CLI `--reset` is standaard ingesteld op `config+creds+sessions`; gebruik `--reset-scope full` om ook de werkruimte te verwijderen.
    - Als de configuratie ongeldig is of verouderde sleutels bevat, stopt de wizard en wordt je gevraagd `openclaw doctor` uit te voeren voordat je doorgaat.
    - Bij opnieuw instellen wordt de status naar de prullenmand verplaatst (nooit rechtstreeks verwijderd) en kun je uit de volgende bereiken kiezen:
      - Alleen configuratie
      - Configuratie + referenties + sessies
      - Volledig opnieuw instellen (verwijdert ook de werkruimte)

  </Step>
  <Step title="Model en authenticatie">
    - De volledige optiematrix staat in [Authenticatie- en modelopties](#auth-and-model-options).

  </Step>
  <Step title="Werkruimte">
    - Standaard `~/.openclaw/workspace` (configureerbaar).
    - Maakt de werkruimtebestanden aan die nodig zijn voor de bootstrap bij de eerste uitvoering.
    - Bij opnieuw uitvoeren behoudt een bestaand agentenregister zijn vlootbrede werkruimte, tenzij
      je de verplaatsing expliciet bevestigt. Niet-interactieve heruitvoeringen waarschuwen en behouden
      de huidige waarde.
    - Indeling van de werkruimte: [Agentwerkruimte](/nl/concepts/agent-workspace).

  </Step>
  <Step title="Gateway">
    - Vraagt om de poort, binding, authenticatiemodus en Tailscale-blootstelling.
    - Aanbevolen: houd tokenauthenticatie zelfs voor loopback ingeschakeld, zodat lokale WS-clients zich moeten authenticeren.
    - In tokenmodus biedt de interactieve configuratie:
      - **Token in platte tekst genereren/opslaan** (standaard)
      - **SecretRef gebruiken** (optioneel)
    - In wachtwoordmodus ondersteunt de interactieve configuratie ook opslag in platte tekst of als SecretRef.
    - Niet-interactief token-SecretRef-pad: `--gateway-token-ref-env <ENV_VAR>`.
      - Vereist een niet-lege omgevingsvariabele in de procesomgeving van de onboarding.
      - Kan niet worden gecombineerd met `--gateway-token`.
    - Schakel authenticatie alleen uit als je elk lokaal proces volledig vertrouwt.
    - Bindingen die geen loopback gebruiken, vereisen nog steeds authenticatie.

  </Step>
  <Step title="Kanalen">
    - [WhatsApp](/nl/channels/whatsapp): optionele QR-aanmelding
    - [Telegram](/nl/channels/telegram): bottoken
    - [Discord](/nl/channels/discord): bottoken
    - [Google Chat](/nl/channels/googlechat): JSON van serviceaccount + webhookdoelgroep
    - [Mattermost](/nl/channels/mattermost): bottoken + basis-URL
    - [Signal](/nl/channels/signal): optionele installatie van `signal-cli` + accountconfiguratie
    - [iMessage](/nl/channels/imessage): pad naar de `imsg`-CLI + toegang tot de Messages-database; gebruik een SSH-wrapper wanneer de Gateway niet op een Mac draait
    - DM-beveiliging: standaard wordt koppeling gebruikt. De eerste DM verzendt een code; keur deze goed via
      `openclaw pairing approve <channel> <code>` of gebruik toelatingslijsten.
  </Step>
  <Step title="Zoeken op het web">
    - Kies een provider (Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, Tavily) of sla deze stap over.
    - Sla deze stap over met `--skip-search`; configureer deze later opnieuw met `openclaw configure --section web`.

  </Step>
  <Step title="Daemoninstallatie">
    - macOS: LaunchAgent
      - Vereist een aangemelde gebruikerssessie; gebruik voor een headless-systeem een aangepaste LaunchDaemon (niet meegeleverd).
    - Linux en Windows via WSL2: systemd-gebruikerseenheid
      - De wizard probeert `loginctl enable-linger <user>`, zodat de Gateway actief blijft na afmelden.
      - Kan om sudo vragen (schrijft `/var/lib/systemd/linger`); eerst wordt het zonder sudo geprobeerd.
    - Systeemeigen Windows: eerst Scheduled Task
      - Als het maken van de taak wordt geweigerd, valt OpenClaw terug op een aanmeldingsitem per gebruiker in de map Startup en wordt de Gateway onmiddellijk gestart.
      - Scheduled Tasks blijven de voorkeur houden omdat ze een betere supervisorstatus bieden.
    - Runtimeselectie: Node is vereist omdat het canonieke opslagmedium voor de runtimestatus van OpenClaw `node:sqlite` gebruikt.

  </Step>
  <Step title="Statuscontrole">
    - Start de Gateway (indien nodig) en voert `openclaw health` uit.
    - `openclaw status --deep` voegt de live statusprobe van de Gateway toe aan de statusuitvoer, inclusief kanaalprobes wanneer die worden ondersteund.

  </Step>
  <Step title="Skills">
    - Leest beschikbare Skills en controleert de vereisten.
    - Laat je een Node-beheerder kiezen: npm, pnpm of bun.
    - Installeert optionele afhankelijkheden voor vertrouwde gebundelde Skills wanneer het vereiste
      installatieprogramma beschikbaar is.
    - Slaat niet-beschikbare installatieprogramma's voor Homebrew, uv en Go over en groepeert vervolgens de betrokken
      Skills met instructies voor handmatige configuratie. Voer `openclaw doctor` uit nadat je
      de ontbrekende vereisten hebt geïnstalleerd.

  </Step>
  <Step title="Voltooien">
    - Samenvatting en vervolgstappen, inclusief opties voor iOS-, Android- en macOS-apps.

  </Step>
</Steps>

<Note>
Als er geen GUI wordt gedetecteerd, drukt de wizard instructies voor SSH-poortdoorschakeling voor de Control UI af in plaats van een browser te openen.
Als Control UI-assets ontbreken, probeert de wizard ze te bouwen; de terugvaloptie is `pnpm ui:build` (installeert automatisch UI-afhankelijkheden).
</Note>

## Details van de externe modus

De externe modus configureert deze machine om verbinding te maken met een Gateway
elders. Er wordt niets op de externe host geïnstalleerd of gewijzigd.

Wat je instelt:

- URL van externe Gateway (`ws://...` of `wss://...`)
- Token, wachtwoord of geen authenticatie, overeenkomstig de configuratie van de externe Gateway

<Steps>
  <Step title="Detectie (optioneel)">
    Als `dns-sd` (macOS) of `avahi-browse` (Linux) beschikbaar is, biedt de onboarding
    aan om naar Bonjour/mDNS-bakens van Gateways te zoeken voordat wordt teruggevallen op
    handmatige URL-invoer. Indien geconfigureerd, wordt ook detectie via wide-area DNS-SD
    geprobeerd. Documentatie: [Gateway-detectie](/nl/gateway/discovery), [Bonjour](/nl/gateway/bonjour).
  </Step>
  <Step title="Verbindingsmethode">
    Wanneer een baken is geselecteerd, kies je voor een rechtstreekse WebSocket of een SSH-tunnel:
    - **Rechtstreeks**: maakt verbinding via `wss://` en vraagt je de gedetecteerde
      TLS-vingerafdruk te vertrouwen (vastzetten op basis van vertrouwen bij eerste gebruik; alleen vastgezet als je deze accepteert).
    - **SSH-tunnel**: drukt een `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`-
      opdracht af die je eerst moet uitvoeren en maakt daarna verbinding met het lokale tunneleindpunt.
  </Step>
  <Step title="Authenticatie">
    Kies een token (aanbevolen), wachtwoord of geen authenticatie en sla dit vervolgens desgewenst
    op als SecretRef in plaats van als platte tekst.
  </Step>
</Steps>

<Note>
Als de Gateway alleen via loopback bereikbaar en niet detecteerbaar is, gebruik je handmatig een SSH-tunnel of tailnet.
`ws://` in platte tekst wordt geaccepteerd voor loopback, letterlijke privé-IP-adressen, `.local` en Tailnet-URL's met `*.ts.net`; andere privé-DNS-namen vereisen `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1`.
</Note>

## Authenticatie- en modelopties

Als een providerconfiguratiestap tijdens interactieve onboarding mislukt (bijvoorbeeld een optie voor hergebruik van de CLI
zonder lokale aanmelding), toont de wizard de fout en keert terug naar de providerkiezer
in plaats van af te sluiten. Expliciete uitvoeringen van `--auth-choice` blijven snel mislukken voor automatisering.

<AccordionGroup>
  <Accordion title="Anthropic-API-sleutel">
    Gebruikt `ANTHROPIC_API_KEY` indien aanwezig of vraagt om een sleutel en slaat deze vervolgens op voor gebruik door de daemon.
  </Accordion>
  <Accordion title="Anthropic Claude CLI">
    Voorkeursroute voor lokaal gebruik tijdens interactieve onboarding/configuratie; hergebruikt indien beschikbaar een bestaande aanmelding bij de Claude CLI.
  </Accordion>
  <Accordion title="OpenAI Code-abonnement (OAuth)">
    Browserflow; plak `code#state`.

    Bij een nieuwe configuratie zonder primair model wordt `agents.defaults.model`
    via de Codex-runtime ingesteld op `openai/gpt-5.6-sol`.

  </Accordion>
  <Accordion title="OpenAI Code-abonnement (apparaatkoppeling)">
    Browserkoppelingsflow met een kortlevende apparaatcode.

    Bij een nieuwe configuratie zonder primair model wordt `agents.defaults.model`
    via de Codex-runtime ingesteld op `openai/gpt-5.6-sol`.

  </Accordion>
  <Accordion title="OpenAI-API-sleutel">
    Gebruikt `OPENAI_API_KEY` indien aanwezig of vraagt om een sleutel en slaat de referentie vervolgens op in authenticatieprofielen.

    Bij een nieuwe configuratie zonder primair model wordt `agents.defaults.model`
    ingesteld op `openai/gpt-5.6`; de kale model-ID voor de rechtstreekse API wordt omgezet naar de Sol-laag.

    Wanneer OpenAI wordt toegevoegd of opnieuw wordt geauthenticeerd, blijft een bestaand expliciet primair
    model behouden, waaronder `openai/gpt-5.5`. Als het account geen GPT-5.6 beschikbaar stelt,
    selecteer je expliciet `openai/gpt-5.5`; OpenClaw verlaagt dit niet stilzwijgend.

  </Accordion>
  <Accordion title="xAI (Grok) OAuth">
    Aanmelden via de browser voor geschikte SuperGrok- of X Premium-accounts. Dit is voor
    de meeste gebruikers het aanbevolen xAI-traject. OpenClaw slaat het resulterende authenticatieprofiel
    op voor Grok-modellen, Grok `web_search`, `x_search` en `code_execution`.
  </Accordion>
  <Accordion title="xAI (Grok)-apparaatcode">
    Browseraanmelding die geschikt is voor externe systemen, met een korte code in plaats van een localhost-
    callback. Gebruik dit vanaf SSH-, Docker- of VPS-hosts.
  </Accordion>
  <Accordion title="xAI (Grok)-API-sleutel">
    Vraagt om `XAI_API_KEY` en configureert xAI als modelprovider. Gebruik dit
    wanneer je een API-sleutel van de xAI Console wilt in plaats van OAuth via een abonnement.
  </Accordion>
  <Accordion title="OpenCode">
    Vraagt om `OPENCODE_API_KEY` (of `OPENCODE_ZEN_API_KEY`) en laat je de Zen- of Go-catalogus kiezen (één API-sleutel dekt beide).
    Installatie-URL: [opencode.ai/auth](https://opencode.ai/auth).
  </Accordion>
  <Accordion title="API-sleutel (algemeen)">
    Slaat de sleutel voor je op.
  </Accordion>
  <Accordion title="Vercel AI Gateway">
    Vraagt om `AI_GATEWAY_API_KEY`.
    Meer informatie: [Vercel AI Gateway](/nl/providers/vercel-ai-gateway).
  </Accordion>
  <Accordion title="Cloudflare AI Gateway">
    Vraagt om account-ID, gateway-ID en `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    Meer informatie: [Cloudflare AI Gateway](/nl/providers/cloudflare-ai-gateway).
  </Accordion>
  <Accordion title="MiniMax">
    De configuratie wordt automatisch geschreven. De standaard voor hosting is `MiniMax-M3`; installatie met een API-sleutel gebruikt
    `minimax/...` en installatie met OAuth gebruikt `minimax-portal/...`.
    Meer informatie: [MiniMax](/nl/providers/minimax).
  </Accordion>
  <Accordion title="StepFun">
    De configuratie wordt automatisch geschreven voor StepFun Standard of Step Plan op Chinese of wereldwijde eindpunten.
    Standard bevat momenteel `step-3.5-flash` en Step Plan bevat ook `step-3.5-flash-2603`.
    Meer informatie: [StepFun](/nl/providers/stepfun).
  </Accordion>
  <Accordion title="Synthetic (Anthropic-compatibel)">
    Vraagt om `SYNTHETIC_API_KEY`.
    Meer informatie: [Synthetic](/nl/providers/synthetic).
  </Accordion>
  <Accordion title="Ollama (cloud- en lokale open modellen)">
    Vraagt eerst om `Cloud + Local`, `Cloud only` of `Local only`.
    `Cloud only` gebruikt `OLLAMA_API_KEY` met `https://ollama.com`.
    De hostgebaseerde modi vragen om de basis-URL (standaard `http://127.0.0.1:11434`), detecteren beschikbare modellen en stellen standaardwaarden voor.
    `Cloud + Local` controleert ook of die Ollama-host is aangemeld voor cloudtoegang.
    Meer informatie: [Ollama](/nl/providers/ollama).
  </Accordion>
  <Accordion title="Moonshot en Kimi Coding">
    Configuraties voor Moonshot (Kimi K2) en Kimi Coding worden automatisch geschreven.
    Meer informatie: [Moonshot AI (Kimi + Kimi Coding)](/nl/providers/moonshot).
  </Accordion>
  <Accordion title="Aangepaste provider">
    Werkt met OpenAI-compatibele, OpenAI Responses-compatibele en Anthropic-compatibele eindpunten.

    Interactieve onboarding ondersteunt dezelfde opslagkeuzes voor API-sleutels als andere API-sleuteltrajecten voor providers:
    - **API-sleutel nu plakken** (platte tekst)
    - **Geheimverwijzing gebruiken** (omgevingsverwijzing of geconfigureerde providerverwijzing, met voorafgaande validatie)

    Onboarding leidt afbeeldingsondersteuning af voor gangbare ID's van visiemodellen (GPT-4o/4.1/5.x, Claude 3/4, Gemini, Qwen-VL, LLaVA, Pixtral en vergelijkbare modellen) en vraagt dit alleen wanneer de modelnaam onbekend is.

    Niet-interactieve vlaggen:
    - `--auth-choice custom-api-key`
    - `--custom-base-url`
    - `--custom-model-id`
    - `--custom-api-key` (optioneel; valt terug op `CUSTOM_API_KEY`)
    - `--custom-provider-id` (optioneel)
    - `--custom-compatibility <openai|openai-responses|anthropic>` (optioneel; standaard `openai`)
    - `--custom-image-input` / `--custom-text-input` (optioneel; overschrijft de afgeleide invoercapaciteit van het model)

  </Accordion>
  <Accordion title="Overslaan">
    Laat authenticatie ongeconfigureerd.
  </Accordion>
</AccordionGroup>

Modelgedrag:

- Kies het standaardmodel uit de gedetecteerde opties of voer de provider en het model handmatig in.
- Wanneer onboarding begint vanuit een keuze voor providerauthenticatie, geeft de modelkiezer
  automatisch de voorkeur aan die provider. Voor Volcengine en BytePlus komt die voorkeur
  ook overeen met hun varianten voor programmeerplannen (`volcengine-plan/*`,
  `byteplus-plan/*`).
- Als dat voorkeursproviderfilter leeg zou zijn, valt de kiezer terug op
  de volledige catalogus in plaats van geen modellen te tonen.
- De wizard voert een modelcontrole uit en waarschuwt als het geconfigureerde model onbekend is of authenticatie ontbreekt.

Paden voor aanmeldgegevens en profielen:

- Authenticatieprofielen (API-sleutels + OAuth): `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- Verouderde OAuth-import: `~/.openclaw/credentials/oauth.json`

Opslagmodus voor aanmeldgegevens:

- Bij standaardonboarding worden API-sleutels als plattetekstwaarden in authenticatieprofielen opgeslagen.
- `--secret-input-mode ref` schakelt de verwijzingsmodus in in plaats van opslag van sleutels als platte tekst.
  Bij interactieve installatie kun je kiezen uit:
  - verwijzing naar een omgevingsvariabele (bijvoorbeeld `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)
  - geconfigureerde providerverwijzing (`file` of `exec`) met provideralias + ID
- De interactieve verwijzingsmodus voert vóór het opslaan een snelle voorafgaande validatie uit.
  - Omgevingsverwijzingen: valideert de variabelenaam + een niet-lege waarde in de huidige onboardingomgeving.
  - Providerverwijzingen: valideert de providerconfiguratie en herleidt de gevraagde ID.
  - Als de voorafgaande validatie mislukt, toont onboarding de fout en kun je het opnieuw proberen.
- In de niet-interactieve modus wordt `--secret-input-mode ref` alleen door de omgeving ondersteund.
  - Stel de omgevingsvariabele van de provider in de procesomgeving voor onboarding in.
  - Inline sleutelvlaggen (bijvoorbeeld `--openai-api-key`) vereisen dat die omgevingsvariabele is ingesteld; anders mislukt onboarding onmiddellijk.
  - Voor aangepaste providers slaat de niet-interactieve modus `ref` `models.providers.<id>.apiKey` op als `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`.
  - In dat geval met een aangepaste provider vereist `--custom-api-key` dat `CUSTOM_API_KEY` is ingesteld; anders mislukt onboarding onmiddellijk.
- Gateway-authenticatiegegevens ondersteunen bij interactieve installatie zowel platte tekst als SecretRef:
  - Tokenmodus: **Platteteksttoken genereren/opslaan** (standaard) of **SecretRef gebruiken**.
  - Wachtwoordmodus: platte tekst of SecretRef.
- Niet-interactief pad voor token-SecretRef: `--gateway-token-ref-env <ENV_VAR>`.
- Bestaande installaties met platte tekst blijven ongewijzigd werken.

<Note>
Tip voor headless systemen en servers: voltooi OAuth op een machine met een browser en kopieer vervolgens
de `auth-profiles.json` van die agent (bijvoorbeeld
`~/.openclaw/agents/<agentId>/agent/auth-profiles.json` of het overeenkomende
`$OPENCLAW_STATE_DIR/...`-pad) naar de gatewayhost. `credentials/oauth.json`
is alleen een verouderde importbron.
</Note>

## Uitvoer en interne werking

Typische velden in `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.skipBootstrap` wanneer `--skip-bootstrap` wordt doorgegeven
- `agents.defaults.model` / `models.providers` (als Minimax is gekozen)
- `tools.profile` (lokale onboarding gebruikt standaard `"coding"` wanneer dit niet is ingesteld; bestaande expliciete waarden blijven behouden)
- `gateway.*` (modus, binding, authenticatie, Tailscale)
- `session.dmScope` (onboarding behoudt expliciete waarden en laat dit anders oningesteld, zodat de standaardwaarde `main` alle privéberichten van verschillende kanalen in de doorlopende hoofdsessie van de agent bewaart—de standaard voor een persoonlijke agent. Gebruik voor gedeelde inboxen of inboxen met meerdere gebruikers `per-channel-peer`; `openclaw security audit` beveelt isolatie aan wanneer verkeer van privéberichten met meerdere gebruikers wordt gedetecteerd)
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Toestemmingslijsten voor kanalen (Discord, iMessage, Signal, Slack, Telegram, WhatsApp) wanneer je hier tijdens de prompts voor kiest; Discord en Slack zetten ingevoerde namen ook om naar ID's
- `skills.install.nodeManager`
  - De vlag `setup --node-manager` accepteert `npm`, `pnpm` of `bun`.
  - Bij handmatige configuratie kan `skills.install.nodeManager: "yarn"` later nog worden ingesteld.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add` schrijft `agents.entries.*` en optioneel `bindings`.

WhatsApp-aanmeldgegevens komen onder `~/.openclaw/credentials/whatsapp/<accountId>/`.
Actieve sessies en transcripties worden opgeslagen in
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. De map
`~/.openclaw/agents/<agentId>/sessions/` wordt gebruikt voor invoer voor verouderde migraties
en archief-/ondersteuningsartefacten.

<Note>
Sommige kanalen worden als plugins geleverd. Wanneer je deze tijdens de installatie selecteert, vraagt de wizard
om de plugin te installeren (npm of lokaal pad) voordat het kanaal wordt geconfigureerd.
</Note>

### Aanbevelingen voor geïnstalleerde apps

Nadat de controle voor modeltoegang is geslaagd, scant de klassieke interactieve onboarding op macOS namen en bundle-ID's van applicaties zonder om privacytoestemming van macOS te vragen. De officiële plugincatalogi en ClawHub worden doorzocht, waarna het geconfigureerde model wordt gevraagd onjuiste naamovereenkomsten af te wijzen en relevante plugins of skills aan te bevelen. Aanbevolen overeenkomsten zijn standaard geselecteerd; optionele overeenkomsten vereisen een expliciete selectie.

Het resultatenscherm vermeldt de gedetecteerde applicaties en toont: "App-namen zijn gekoppeld met behulp van je geconfigureerde model en de zoekfunctie van ClawHub." Stel `wizard.appRecommendations` in op `false` om zowel deze onboardingstap als Gateway-toegang tot app-inventarissen van nodes uit te schakelen. De scan wordt niet gebruikt bij quickstart- of niet-macOS-onboarding.

## Niet-interactieve installatie

`--non-interactive` vereist `--accept-risk` (bevestigt dat agents
krachtig zijn en volledige systeemtoegang riskant is):

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY"
```

Volledige naslag voor vlaggen en providerspecifieke voorbeelden: [`openclaw onboard`](/nl/cli/onboard), [CLI-automatisering](/nl/start/wizard-cli-automation).

## RPC van de Gateway-wizard

- `wizard.start`
- `wizard.next`
- `wizard.cancel`
- `wizard.status`

Clients (macOS-app en Control UI) kunnen stappen weergeven zonder de onboardinglogica opnieuw te implementeren.

## Gedrag bij installatie van Signal

- Downloadt het juiste release-artefact uit de officiële GitHub-releases van `signal-cli` (native build, alleen Linux x86-64)
- Installeert op andere platforms (macOS, niet-x64 Linux) in plaats daarvan via Homebrew
- Slaat de installatie van het release-artefact op onder `~/.openclaw/tools/signal-cli/<version>/`
- Schrijft `channels.signal.transport.cliPath` met `kind: "managed-native"` in de configuratie
- Native Windows wordt nog niet ondersteund; voer onboarding uit binnen WSL2 om het Linux-installatiepad te verkrijgen

## Gerelateerde documentatie

- Onboardingcentrum: [Onboarding (CLI)](/nl/start/wizard)
- Automatisering en scripts: [CLI-automatisering](/nl/start/wizard-cli-automation)
- Opdrachtnaslag: [`openclaw onboard`](/nl/cli/onboard)
