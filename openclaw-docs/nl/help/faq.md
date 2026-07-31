---
read_when:
    - Veelgestelde ondersteuningsvragen over configuratie, installatie, onboarding of runtime beantwoorden
    - Door gebruikers gemelde problemen triëren vóór diepgaandere foutopsporing
summary: Veelgestelde vragen over de installatie, configuratie en het gebruik van OpenClaw
title: Veelgestelde vragen
x-i18n:
    generated_at: "2026-07-27T06:18:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7bddbf851a0e25323aa7e7cfc3882b33cc0d33a2aa223cccf00328af477ab4c4
    source_path: help/faq.md
    workflow: 16
---

Snelle antwoorden plus diepgaandere probleemoplossing voor praktijksituaties (lokale ontwikkeling, VPS, meerdere agents, OAuth/API-sleutels, model-failover). Zie [Probleemoplossing](/nl/gateway/troubleshooting) voor runtimediagnostiek. Zie [Configuratie](/nl/gateway/configuration) voor de volledige configuratiereferentie.

## Eerste 60 seconden als er iets niet werkt

<Steps>
  <Step title="Snelle status">
    ```bash
    openclaw status
    ```
    Snelle lokale samenvatting: besturingssysteem + update, bereikbaarheid van Gateway/service, agents/sessies, providerconfiguratie + runtimeproblemen (wanneer de Gateway bereikbaar is).
  </Step>
  <Step title="Plakbaar rapport (veilig om te delen)">
    ```bash
    openclaw status --all
    ```
    Alleen-lezen diagnose met het einde van het logboek (tokens geredigeerd).
  </Step>
  <Step title="Daemon- en poortstatus">
    ```bash
    openclaw gateway status
    ```
    Toont de runtime van de supervisor versus RPC-bereikbaarheid, de doel-URL van de probe en welke configuratie de service waarschijnlijk heeft gebruikt.
  </Step>
  <Step title="Diepgaande probes">
    ```bash
    openclaw status --deep
    ```
    Live statusprobe van de Gateway, inclusief kanaalprobes wanneer ondersteund (vereist een bereikbare Gateway). Zie [Status](/nl/gateway/health).
  </Step>
  <Step title="Volg het nieuwste logboek">
    ```bash
    openclaw logs --follow
    ```
    Als RPC niet beschikbaar is, val dan terug op:
    ```bash
    tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
    # Voorbeeld van benoemd profiel:
    tail -f "/tmp/openclaw/openclaw-dev-$(date +%F).log"
    ```
    Bestandslogboeken staan los van servicelogboeken; zie [Logboekregistratie](/nl/logging) en [Probleemoplossing](/nl/gateway/troubleshooting).
  </Step>
  <Step title="Voer de doctor uit (reparaties)">
    ```bash
    openclaw doctor
    ```
    Repareert/migreert configuratie en status en voert vervolgens statuscontroles uit. Zie [Doctor](/nl/gateway/doctor).
  </Step>
  <Step title="Gateway-snapshot (alleen WS)">
    ```bash
    openclaw health --json
    openclaw health --verbose   # toont bij fouten de doel-URL + het configuratiepad
    ```
    Vraagt de actieve Gateway om een volledige snapshot. Zie [Status](/nl/gateway/health).
  </Step>
</Steps>

## Snel starten en configuratie bij de eerste uitvoering

Vragen en antwoorden over de eerste uitvoering — installatie, onboarding, authenticatieroutes, abonnementen en aanvankelijke fouten — staan in de [FAQ over de eerste uitvoering](/nl/help/faq-first-run).

## Wat is OpenClaw?

<AccordionGroup>
  <Accordion title="Wat is OpenClaw, in één alinea?">
    OpenClaw is een persoonlijke AI-assistent die je op je eigen apparaten uitvoert. Deze antwoordt via de berichtenplatforms die je al gebruikt (Discord, Google Chat, iMessage, Mattermost, Signal, Slack, Telegram, WebChat, WhatsApp en meegeleverde kanaalplugins zoals QQ Bot) en ondersteunt op geschikte platforms ook spraak plus een live Canvas. De **Gateway** is de altijd actieve besturingslaag; de assistent is het product.
  </Accordion>

  <Accordion title="Waardepropositie">
    OpenClaw is niet "alleen maar een wrapper voor Claude". Het is een **local-first besturingslaag** die een capabele assistent uitvoert op **je eigen hardware**, bereikbaar via de chatapps die je al gebruikt, met stateful sessies, geheugen en hulpmiddelen — zonder je workflows over te dragen aan een gehoste SaaS.

    - **Jouw apparaten, jouw gegevens**: voer de Gateway uit waar je maar wilt (Mac, Linux, VPS) en houd de werkruimte en sessiegeschiedenis lokaal.
    - **Echte kanalen, geen websandbox**: Discord/iMessage/Signal/Slack/Telegram/WhatsApp/etc., plus mobiele spraak en Canvas op ondersteunde platforms.
    - **Modelonafhankelijk**: gebruik Anthropic, MiniMax, OpenAI, OpenRouter, enzovoort, met routering en failover per agent.
    - **Optie voor uitsluitend lokaal gebruik**: voer lokale modellen uit zodat alle gegevens op je apparaat kunnen blijven.
    - **Routering met meerdere agents**: afzonderlijke agents per kanaal, account of taak, elk met een eigen werkruimte en standaardinstellingen.
    - **Open source en aanpasbaar**: inspecteer, breid uit en host zelf zonder leveranciersafhankelijkheid.

    Documentatie: [Gateway](/nl/gateway), [Kanalen](/nl/channels), [Meerdere agents](/nl/concepts/multi-agent), [Geheugen](/nl/concepts/memory).

  </Accordion>

  <Accordion title="Ik heb het net ingesteld — wat kan ik het beste eerst doen?">
    Goede eerste projecten: bouw een website (WordPress, Shopify of een statische site); maak een prototype van een mobiele app (opzet, schermen, API-plan); organiseer bestanden en mappen; verbind Gmail en automatiseer samenvattingen of opvolging.

    Het kan grote taken uitvoeren, maar werkt het beste wanneer deze in fasen worden opgesplitst, met subagents voor parallel werk.

  </Accordion>

  <Accordion title="Wat zijn de vijf belangrijkste alledaagse toepassingen van OpenClaw?">
    - **Persoonlijke briefings**: samenvattingen van je inbox, agenda en nieuws dat voor jou relevant is.
    - **Onderzoek en conceptteksten**: snel onderzoek, samenvattingen en eerste versies van e-mails of documenten.
    - **Herinneringen en opvolging**: door Cron of Heartbeat aangestuurde seintjes en checklists.
    - **Browserautomatisering**: formulieren invullen, gegevens verzamelen en webtaken herhalen.
    - **Coördinatie tussen apparaten**: verstuur een taak vanaf je telefoon, laat de Gateway deze op een server uitvoeren en ontvang het resultaat terug in de chat.

  </Accordion>

  <Accordion title="Kan OpenClaw helpen met leadgeneratie, outreach, advertenties en blogs voor een SaaS?">
    Ja, voor **onderzoek, kwalificatie en het opstellen van concepten**: websites scannen, shortlists maken, potentiële klanten samenvatten en concepten voor outreach of advertentieteksten schrijven.

    Houd voor **outreach- of advertentiecampagnes** altijd een mens betrokken. Vermijd spam, volg lokale wetgeving en platformbeleid en controleer alles voordat het wordt verzonden. Laat OpenClaw een concept opstellen; jij keurt het goed.

    Documentatie: [Beveiliging](/nl/gateway/security).

  </Accordion>

  <Accordion title="Wat zijn de voordelen ten opzichte van Claude Code voor webontwikkeling?">
    OpenClaw is een **persoonlijke assistent** en coördinatielaag, geen vervanging voor een IDE. Gebruik Claude Code of Codex voor de snelste directe programmeercyclus binnen een repository. Gebruik OpenClaw voor duurzaam geheugen, toegang vanaf meerdere apparaten en orkestratie van hulpmiddelen.

    - Permanent geheugen en een persistente werkruimte tussen sessies.
    - Toegang vanaf meerdere platforms (Telegram, WhatsApp, TUI, WebChat).
    - Orkestratie van hulpmiddelen (browser, bestanden, planning, hooks).
    - Altijd actieve Gateway (uitvoeren op een VPS, communiceren vanaf elke locatie).
    - Nodes voor lokale browser/scherm/camera/uitvoering.

    Showcase: [https://openclaw.ai/showcase](https://openclaw.ai/showcase).

  </Accordion>
</AccordionGroup>

## Skills en automatisering

<AccordionGroup>
  <Accordion title="Hoe pas ik Skills aan zonder de repository vervuild te houden?">
    Gebruik beheerde overrides in plaats van de kopie in de repository te bewerken. Plaats wijzigingen in `~/.openclaw/skills/<name>/SKILL.md` (of voeg een map toe via `skills.load.extraDirs` in `~/.openclaw/openclaw.json`). Prioriteit: `<workspace>/skills` -> `<workspace>/.agents/skills` -> `~/.agents/skills` -> `~/.openclaw/skills` -> meegeleverd -> `skills.load.extraDirs`, zodat beheerde overrides voorrang krijgen op meegeleverde Skills zonder git aan te raken. Om globaal te installeren maar de zichtbaarheid tot bepaalde agents te beperken, bewaar je de gedeelde kopie in `~/.openclaw/skills` en beheer je de zichtbaarheid met `agents.defaults.skills` / `agents.entries.*.skills`. Alleen wijzigingen die geschikt zijn voor upstream, moeten als pull requests voor de kopie in de repository worden ingediend.
  </Accordion>

  <Accordion title="Kan ik Skills vanuit een aangepaste map laden?">
    Ja: voeg mappen toe via `skills.load.extraDirs` in `~/.openclaw/openclaw.json` (laagste prioriteit in de bovenstaande volgorde). `clawhub` installeert standaard in `./skills`, die OpenClaw tijdens de volgende sessie als `<workspace>/skills` behandelt. Combineer dit met `agents.defaults.skills` of `agents.entries.*.skills` om de zichtbaarheid tot bepaalde agents te beperken.
  </Accordion>

  <Accordion title="Hoe kan ik verschillende modellen of instellingen voor verschillende taken gebruiken?">
    Ondersteunde patronen:

    - **Cron-taken**: geïsoleerde taken kunnen per taak een `model`-override instellen.
    - **Agents**: routeer taken naar afzonderlijke agents met verschillende standaardmodellen, denkniveaus en streamparameters.
    - **Schakelen op verzoek**: `/model` schakelt op elk moment het model van de huidige sessie om.

    Voorbeeld — hetzelfde model, verschillende instellingen per agent:

    ```json5
    {
      agents: {
        list: [
          {
            id: "coder",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "high",
            params: { temperature: 0.1 },
          },
          {
            id: "chat",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "off",
            params: { temperature: 0.8 },
          },
        ],
      },
    }
    ```

    Plaats gedeelde standaardinstellingen per model in `agents.defaults.models["provider/model"].params` en vervolgens agentspecifieke overrides in de platte `agents.entries.*.params`. Dupliceer hetzelfde model niet onder de geneste `agents.entries.*.models["provider/model"].params`; dat pad is bedoeld voor de modelcatalogus en runtime-overrides per agent.

    Zie [Cron-taken](/nl/automation/cron-jobs), [Routering met meerdere agents](/nl/concepts/multi-agent), [Configuratie](/nl/gateway/config-agents), [Slash-opdrachten](/nl/tools/slash-commands).

  </Accordion>

  <Accordion title="De bot loopt vast tijdens zwaar werk. Hoe besteed ik dat uit?">
    Gebruik **subagents** voor langdurige of parallelle taken: ze worden in hun eigen sessie uitgevoerd, retourneren een samenvatting en houden je hoofdchat responsief. Vraag de bot om "voor deze taak een subagent te starten" of gebruik `/subagents`. Gebruik `/status` om te zien of de Gateway momenteel bezig is.

    Zowel langdurige taken als subagents verbruiken tokens; stel via `agents.defaults.subagents.model` een goedkoper model voor subagents in als de kosten belangrijk zijn.

    Documentatie: [Subagents](/nl/tools/subagents), [Achtergrondtaken](/nl/automation/tasks).

  </Accordion>

  <Accordion title="Hoe werken aan threads gebonden subagentsessies op Discord?">
    Koppel een Discord-thread aan een subagent of sessiedoel, zodat vervolgberichten daarin bij die gekoppelde sessie blijven.

    - Start met `sessions_spawn` en gebruik `thread: true` (optioneel `mode: "session"` voor permanente opvolging).
    - Of koppel handmatig met `/focus <target>`.
    - `/agents` controleert de koppelingsstatus.
    - `/session idle <duration|off>` en `/session max-age <duration|off>` beheren het automatisch opheffen van de focus.
    - `/unfocus` ontkoppelt de thread.

    Configuratie: `session.threadBindings.enabled` (globale schakelaar), `session.threadBindings.idleHours` (standaard `24`, `0` schakelt uit), `session.threadBindings.maxAgeHours` (standaard `0` = geen harde limiet) en `session.threadBindings.spawnSessions` voor automatisch koppelen bij het starten (standaard `true`).

    Documentatie: [Subagents](/nl/tools/subagents), [Discord](/nl/channels/discord), [Configuratiereferentie](/nl/gateway/configuration-reference), [Slash-opdrachten](/nl/tools/slash-commands).

  </Accordion>

  <Accordion title="Een subagent is voltooid, maar de voltooiingsupdate ging naar de verkeerde plek of is nooit geplaatst. Wat moet ik controleren?">
    Controleer de opgeloste route van de aanvrager:

    - De bezorging van subagents in voltooiingsmodus geeft de voorkeur aan een gekoppelde thread- of gespreksroute wanneer die bestaat.
    - Als de oorsprong van de voltooiing alleen een kanaal bevat, valt OpenClaw terug op de opgeslagen route van de sessie van de aanvrager (`lastChannel` / `lastTo` / `lastAccountId`), zodat directe bezorging alsnog kan slagen.
    - Geen gekoppelde route en geen bruikbare opgeslagen route: directe bezorging kan mislukken en het resultaat valt terug op bezorging via de sessiewachtrij in plaats van onmiddellijk te worden geplaatst.
    - Ongeldige of verouderde doelen kunnen ook een terugval naar de wachtrij of een definitieve bezorgingsfout veroorzaken.
    - Als het laatste zichtbare assistentantwoord van het kind exact `NO_REPLY` / `no_reply` of `ANNOUNCE_SKIP` is, onderdrukt OpenClaw de aankondiging bewust in plaats van verouderde eerdere voortgang te plaatsen.

    Foutopsporing: `openclaw tasks show <lookup>` waarbij `<lookup>` een taak-id, uitvoerings-id of sessiesleutel is.

    Documentatie: [Subagents](/nl/tools/subagents), [Achtergrondtaken](/nl/automation/tasks), [Sessiehulpmiddelen](/nl/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron of herinneringen worden niet geactiveerd. Wat moet ik controleren?">
    Cron wordt uitgevoerd binnen het Gateway-proces; het wordt niet geactiveerd als de Gateway niet continu actief is.

    - Controleer of Cron is ingeschakeld (`cron.enabled`) en `OPENCLAW_SKIP_CRON` niet is ingesteld.
    - Controleer of de Gateway 24/7 actief blijft (geen slaapstand/herstarts).
    - Controleer de tijdzone van de taak (`--tz` versus de tijdzone van de host).

    Foutopsporing:
    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentatie: [Cron-taken](/nl/automation/cron-jobs), [Automatisering](/nl/automation).

  </Accordion>

  <Accordion title="Cron is uitgevoerd, maar er is niets naar het kanaal verzonden. Waarom?">
    Controleer de afleveringsmodus:

    - `--no-deliver` / `delivery.mode: "none"`: er wordt geen fallbackverzending door de runner verwacht.
    - Ontbrekend of ongeldig aankondigingsdoel (`channel` / `to`): de runner heeft uitgaande aflevering overgeslagen.
    - Authenticatiefouten voor het kanaal (`unauthorized`, `Forbidden`): de runner probeerde af te leveren, maar de inloggegevens verhinderden dit.
    - Een stil geïsoleerd resultaat (alleen `NO_REPLY` / `no_reply`) wordt als opzettelijk niet-afleverbaar beschouwd, waardoor fallbackaflevering vanuit de wachtrij ook wordt onderdrukt.

    Bij geïsoleerde Cron-taken kan de agent nog steeds rechtstreeks verzenden met de tool `message` wanneer een chatroute beschikbaar is. `--announce` beheert alleen de fallbackaflevering door de runner van definitieve tekst die de agent niet al zelf heeft verzonden.

    Foutopsporing:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <lookup>
    ```

    Documentatie: [Cron-taken](/nl/automation/cron-jobs), [Achtergrondtaken](/nl/automation/tasks).

  </Accordion>

  <Accordion title="Waarom wisselde een geïsoleerde Cron-uitvoering van model of probeerde deze het één keer opnieuw?">
    Dit is het live pad voor modelwisseling, geen dubbele planning. Geïsoleerde Cron slaat een runtime-overdracht van het model permanent op en probeert het opnieuw wanneer de actieve uitvoering `LiveSessionModelSwitchError` genereert. Daarbij blijven de gewisselde provider en het gewisselde model (en een eventuele gewisselde overschrijving van het authenticatieprofiel) behouden voordat de nieuwe poging begint.

    Prioriteitsvolgorde voor modelselectie: eerst de modeloverschrijving van de Gmail-hook (`hooks.gmail.model`), daarna `model` per taak, vervolgens een opgeslagen modeloverschrijving voor de Cron-sessie en ten slotte de normale modelselectie van de agent of het standaardmodel.

    De lus voor nieuwe pogingen is beperkt tot de eerste poging plus 2 nieuwe pogingen na een wisseling; daarna breekt Cron af in plaats van eindeloos door te gaan.

    Foutopsporing:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    ```

    Documentatie: [Cron-taken](/nl/automation/cron-jobs), [Cron-CLI](/nl/cli/cron).

  </Accordion>

  <Accordion title="Hoe installeer ik Skills op Linux?">
    Gebruik de ingebouwde `openclaw skills`-opdrachten of plaats Skills in je werkruimte; de macOS-interface voor Skills is niet beschikbaar op Linux. Bekijk Skills op [https://clawhub.ai](https://clawhub.ai).

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install @owner/<skill-slug>
    openclaw skills install @owner/<skill-slug> --version <version>
    openclaw skills install @owner/<skill-slug> --force
    openclaw skills install @owner/<skill-slug> --global
    openclaw skills update --all
    openclaw skills update --all --global
    openclaw skills list --eligible
    openclaw skills check
    ```

    De ingebouwde `openclaw skills install` schrijft standaard naar de map `skills/` van de actieve werkruimte. Voeg `--global` toe om te installeren in de gedeelde beheerde map voor Skills voor alle lokale agents. Installeer de afzonderlijke `clawhub`-CLI alleen om je eigen Skills te publiceren of synchroniseren. Gebruik `agents.defaults.skills` of `agents.entries.*.skills` om te beperken welke agents gedeelde Skills zien.

  </Accordion>

  <Accordion title="Kan OpenClaw taken volgens een planning of continu op de achtergrond uitvoeren?">
    Ja, via de planner van de Gateway:

    - **Cron-taken** voor geplande of terugkerende taken (blijven behouden na herstarts).
    - **Heartbeat** voor periodieke controles van de hoofdsessie.
    - **Geïsoleerde taken** voor autonome agents die samenvattingen plaatsen of bij chats afleveren.

    Documentatie: [Cron-taken](/nl/automation/cron-jobs), [Automatisering](/nl/automation), [Heartbeat](/nl/gateway/heartbeat).

  </Accordion>

  <Accordion title="Kan ik uitsluitend voor Apple macOS bestemde Skills uitvoeren vanaf Linux?">
    Niet rechtstreeks. macOS-Skills worden beperkt door `metadata.openclaw.os` en de vereiste binaire bestanden, en worden alleen geladen wanneer ze geschikt zijn op de **Gateway-host**. Op Linux worden Skills die uitsluitend voor `darwin` zijn bedoeld (`apple-notes`, `apple-reminders`, `things-mac`) niet geladen, tenzij je deze beperking overschrijft.

    Drie ondersteunde patronen:

    **Optie A - voer de Gateway uit op een Mac (het eenvoudigst)**. Voer de Gateway uit waar de macOS-binaire bestanden aanwezig zijn en maak vervolgens verbinding vanaf Linux in de [externe modus](#gateway-ports-already-running-and-remote-mode) of via Tailscale. Skills worden normaal geladen omdat de Gateway-host macOS gebruikt.

    **Optie B - gebruik een macOS-Node (zonder SSH)**. Voer de Gateway uit op Linux, koppel een macOS-Node (menubalkapp) en stel **Node Run Commands** op de Mac in op "Always Ask" of "Always Allow". OpenClaw beschouwt uitsluitend voor macOS bestemde Skills als geschikt wanneer de vereiste binaire bestanden op de Node aanwezig zijn; de agent voert ze uit via de tool `nodes`. Als je bij "Always Ask" in de vraag "Always Allow" goedkeurt, wordt die opdracht aan de toelatingslijst toegevoegd.

    **Optie C - leid macOS-binaire bestanden via SSH door (geavanceerd)**. Laat de Gateway op Linux draaien, maar zorg dat de vereiste binaire CLI-bestanden verwijzen naar SSH-wrappers die op een Mac worden uitgevoerd. Overschrijf vervolgens de Skill om Linux toe te staan, zodat deze geschikt blijft.

    1. Maak een SSH-wrapper voor het binaire bestand (voorbeeld: `memo` voor Apple Notes):
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```
    2. Plaats de wrapper in `PATH` op de Linux-host (bijvoorbeeld `~/bin/memo`).
    3. Overschrijf de metadata van de Skill (in de werkruimte of `~/.openclaw/skills`) om Linux toe te staan:
       ```markdown
       ---
       name: apple-notes
       description: Beheer Apple Notes via de memo-CLI op macOS.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```
    4. Start een nieuwe sessie zodat de momentopname van de Skills wordt vernieuwd.

  </Accordion>

  <Accordion title="Is er een integratie met Notion of HeyGen?">
    Momenteel niet ingebouwd. Mogelijkheden:

    - **Aangepaste Skill / Plugin**: het beste voor betrouwbare API-toegang (beide hebben API's).
    - **Browserautomatisering**: werkt zonder code, maar is langzamer en kwetsbaarder.

    Voor context per klant in de stijl van een bureau: houd één Notion-pagina per klant bij (context + voorkeuren + actief werk) en vraag de agent die pagina aan het begin van een sessie op te halen.

    Open voor een ingebouwde integratie een functieverzoek of bouw een Skill op basis van die API's.

    ```bash
    openclaw skills install @owner/<skill-slug>
    openclaw skills update --all
    ```

    Ingebouwde installaties komen in de map `skills/` van de actieve werkruimte terecht; gebruik `--global` voor alle lokale agents of configureer `agents.defaults.skills` / `agents.entries.*.skills` om de zichtbaarheid te beperken. Sommige Skills verwachten binaire bestanden die met Homebrew zijn geïnstalleerd; op Linux betekent dit Linuxbrew.

    Zie [Skills](/nl/tools/skills), [Skills-configuratie](/nl/tools/skills-config), [ClawHub](/tools/clawhub).

  </Accordion>

  <Accordion title="Hoe gebruik ik mijn bestaande aangemelde Chrome met OpenClaw?">
    Gebruik het ingebouwde browserprofiel `user`, dat via Chrome DevTools MCP verbinding maakt:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    Maak voor een aangepaste naam een expliciet MCP-profiel:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    Dit kan de lokale browser op de host of een verbonden browser-Node gebruiken. Als de Gateway elders draait, voer je een Node-host uit op de browsercomputer of gebruik je in plaats daarvan externe CDP.

    Huidige beperkingen van `existing-session`- / `user`-profielen ten opzichte van het beheerde profiel `openclaw`:

    - `click`, `type`, `hover`, `scrollIntoView`, `drag` en `select` vereisen verwijzingen naar momentopnamen, geen CSS-selectors.
    - Uploadhooks vereisen `ref` of `inputRef`, één bestand tegelijk, zonder CSS-`element`.
    - `responsebody`, PDF-export, onderschepping van downloads en batchacties vereisen nog steeds het beheerde browserpad.

    Zie [Browser](/nl/tools/browser#existing-session-via-chrome-devtools-mcp) voor de volledige vergelijking.

  </Accordion>
</AccordionGroup>

## Sandboxing en geheugen

<AccordionGroup>
  <Accordion title="Is er speciale documentatie over sandboxing?">
    Ja: [Sandboxing](/nl/gateway/sandboxing). Zie [Docker](/nl/install/docker) voor Docker-specifieke configuratie (de volledige Gateway in Docker of sandboxinstallatiekopieën).
  </Accordion>

  <Accordion title="Docker voelt beperkt aan - hoe schakel ik alle functies in?">
    De standaardinstallatiekopie geeft prioriteit aan beveiliging en wordt uitgevoerd als de gebruiker `node`. Systeempakketten, Homebrew en meegeleverde browsers zijn daarom uitgesloten. Voor een completere configuratie:

    - Maak `/home/node` persistent met `OPENCLAW_HOME_VOLUME`, zodat caches behouden blijven.
    - Neem systeemafhankelijkheden op in de installatiekopie met `OPENCLAW_IMAGE_APT_PACKAGES`.
    - Installeer Playwright-browsers via de meegeleverde CLI: `node /app/node_modules/playwright-core/cli.js install chromium`.
    - Stel `PLAYWRIGHT_BROWSERS_PATH` in en maak dat pad persistent.

    Documentatie: [Docker](/nl/install/docker), [Browser](/nl/tools/browser).

  </Accordion>

  <Accordion title="Kan ik privéberichten persoonlijk houden, maar groepen met één agent openbaar en in een sandbox uitvoeren?">
    Ja, als privéverkeer uit **privéberichten** bestaat en openbaar verkeer uit **groepen**. Stel `agents.defaults.sandbox.mode: "non-main"` zo in dat groeps-/kanaalsessies (niet-hoofdsleutels) in de geconfigureerde sandboxbackend worden uitgevoerd, terwijl de hoofdprivéberichtsessie op de host blijft. Docker is de standaardbackend zodra sandboxing is ingeschakeld. Beperk via `tools.sandbox.tools` welke tools in sandboxsessies beschikbaar zijn.

    Configuratiehandleiding: [Groepen: persoonlijke privéberichten + openbare groepen](/nl/channels/groups#pattern-personal-dms-public-groups-single-agent). Belangrijk naslagwerk: [Gateway-configuratie](/nl/gateway/config-agents#agentsdefaultssandbox).

  </Accordion>

  <Accordion title="Hoe koppel ik een hostmap aan de sandbox?">
    Stel `agents.defaults.sandbox.docker.binds` in op `["host:container:mode"]` (bijvoorbeeld `"/home/user/src:/src:ro"`). Globale koppelingen en koppelingen per agent worden samengevoegd; koppelingen per agent worden genegeerd wanneer `scope: "shared"`. Gebruik `:ro` voor alles wat gevoelig is; koppelingen omzeilen de grenzen van het sandboxbestandssysteem.

    OpenClaw valideert bronnen voor koppelingen aan de hand van zowel het genormaliseerde pad als het canonieke pad dat via de diepste bestaande bovenliggende map is bepaald. Ontsnappingen via bovenliggende symlinks worden daardoor standaard geweigerd, zelfs wanneer het laatste padsegment nog niet bestaat.

    Zie [Sandboxing](/nl/gateway/sandboxing#custom-bind-mounts) en [Sandbox versus toolbeleid versus verhoogde rechten](/nl/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check).

  </Accordion>

  <Accordion title="Hoe werkt het geheugen?">
    Het geheugen van OpenClaw bestaat uit Markdown-bestanden in de werkruimte van de agent: dagelijkse notities in `memory/YYYY-MM-DD.md` en samengestelde langetermijnnotities in `MEMORY.md` (alleen hoofd-/privésessies).

    OpenClaw voert ook stil een **geheugenflush vóór Compaction** uit voordat Compaction de conversatie samenvat, waarbij het model eraan wordt herinnerd eerst duurzame notities te schrijven. Dit wordt alleen uitgevoerd wanneer de werkruimte beschrijfbaar is (alleen-lezen-sandboxes slaan dit over); schakel dit uit met `agents.defaults.compaction.memoryFlush.enabled: false`. Zie [Geheugen](/nl/concepts/memory).

  </Accordion>

  <Accordion title="Het geheugen blijft dingen vergeten. Hoe zorg ik dat ze worden onthouden?">
    Vraag de bot om **het feit naar het geheugen te schrijven**: langetermijnnotities komen in `MEMORY.md` en kortetermijncontext in `memory/YYYY-MM-DD.md`. Het model eraan herinneren herinneringen op te slaan, lost dit meestal op. Als het dingen blijft vergeten, controleer dan of de Gateway bij elke uitvoering dezelfde werkruimte gebruikt.

    Documentatie: [Geheugen](/nl/concepts/memory), [Werkruimte van de agent](/nl/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Blijft geheugen voor altijd bewaard? Wat zijn de limieten?">
    Geheugenbestanden staan op schijf en blijven bewaard totdat ze worden verwijderd; de limiet is je opslagruimte, niet het model. **De sessiecontext** wordt nog steeds beperkt door het contextvenster van het model, waardoor lange gesprekken kunnen worden gecompacteerd of afgekapt - daarom bestaat zoeken in het geheugen: alleen de relevante delen worden teruggehaald naar de context.

    Documentatie: [Geheugen](/nl/concepts/memory), [Context](/nl/concepts/context).

  </Accordion>

  <Accordion title="Is voor semantisch zoeken in het geheugen een OpenAI API-sleutel vereist?">
    Alleen als je **OpenAI-embeddings** gebruikt, de standaardprovider. Codex OAuth ondersteunt chat/aanvullingen en verleent **geen** toegang tot embeddings. Aanmelden met Codex (OAuth of de aanmelding via de Codex CLI) schakelt semantisch zoeken in het geheugen dus niet in. Voor OpenAI-embeddings is nog steeds een echte API-sleutel nodig (`OPENAI_API_KEY` of `models.providers.openai.apiKey`).

    Als je alles lokaal wilt houden, stel je `memory.search.provider: "local"` (GGUF/llama.cpp) in. Andere ondersteunde providers: Bedrock, DeepInfra, Gemini (`GEMINI_API_KEY` of `memory.search.remote.apiKey`), GitHub Copilot, LM Studio, Mistral, Ollama, OpenAI-compatibel en Voyage. Raadpleeg [Geheugen](/nl/concepts/memory) en [Zoeken in het geheugen](/nl/concepts/memory-search) voor configuratiedetails.

  </Accordion>
</AccordionGroup>

## Waar alles op schijf staat

<AccordionGroup>
  <Accordion title="Worden alle met OpenClaw gebruikte gegevens lokaal opgeslagen?">
    Nee: **de eigen status van OpenClaw is lokaal**, maar **externe services zien nog steeds wat je naar ze verzendt**.

    - **Standaard lokaal**: sessies, geheugenbestanden, configuratie en werkruimte staan op de Gateway-host (`~/.openclaw` plus je werkruimtemap).
    - **Noodzakelijk op afstand**: berichten die naar modelproviders (Anthropic/OpenAI/enz.) worden verzonden, gaan naar hun API's en chatplatforms (Slack/Telegram/WhatsApp/enz.) slaan berichtgegevens op hun servers op.
    - **Je bepaalt de omvang**: lokale modellen houden prompts op je machine, maar kanaalverkeer loopt nog steeds via de servers van het kanaal.

    Gerelateerd: [Agentwerkruimte](/nl/concepts/agent-workspace), [Geheugen](/nl/concepts/memory).

  </Accordion>

  <Accordion title="Waar slaat OpenClaw zijn gegevens op?">
    Alles staat onder `$OPENCLAW_STATE_DIR` (standaard: `~/.openclaw`):

    | Pad                                                               | Doel                                                               |
    | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                                 | Hoofdconfiguratie (JSON5)                                          |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                        | Verouderde OAuth-import (bij het eerste gebruik naar authenticatieprofielen gekopieerd) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json`     | Authenticatieprofielen (OAuth, API-sleutels, optioneel `keyRef`/`tokenRef`) |
    | `$OPENCLAW_STATE_DIR/secrets.json`                                  | Optionele bestandsgebaseerde geheime payload voor `file` SecretRef-providers |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`              | Verouderd compatibiliteitsbestand (statische `api_key`-vermeldingen verwijderd) |
    | `$OPENCLAW_STATE_DIR/credentials/`                                  | Providerstatus (bijvoorbeeld `whatsapp/<accountId>/creds.json`)                    |
    | `$OPENCLAW_STATE_DIR/agents/`                                       | Status per agent (agentDir plus verouderde/gearchiveerde sessieartefacten) |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/openclaw-agent.sqlite`  | SQLite-status per agent, inclusief sessierijen en transcripties     |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                    | Verouderde bronnen voor sessiemigratie en archief-/ondersteuningsartefacten |

    Het verouderde pad voor één agent `~/.openclaw/agent/*` wordt gemigreerd door `openclaw doctor`.

    Je **werkruimte** (AGENTS.md, geheugenbestanden, Skills enz.) staat apart en wordt geconfigureerd via `agents.defaults.workspace` (standaard: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="Waar horen AGENTS.md / SOUL.md / USER.md / MEMORY.md te staan?">
    Deze staan in de **agentwerkruimte**, niet in `~/.openclaw`.

    - **Werkruimte (per agent)**: `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md`, `MEMORY.md`, `memory/YYYY-MM-DD.md`, optioneel `HEARTBEAT.md`. De hoofdmap met kleine letters `memory.md` dient alleen als invoer voor herstel van verouderde gegevens; `openclaw doctor --fix` kan deze samenvoegen in `MEMORY.md` wanneer beide bestaan.
    - **Statusmap (`~/.openclaw`)**: configuratie, kanaal-/providerstatus, authenticatieprofielen, sessies, logboeken en gedeelde Skills (`~/.openclaw/skills`).

    De standaardwerkruimte is `~/.openclaw/workspace` en kan worden geconfigureerd:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    Als de bot na een herstart dingen „vergeet”, controleer dan of de Gateway bij elke start dezelfde werkruimte gebruikt (de externe modus gebruikt de werkruimte van de **gateway-host**, niet die van je lokale laptop).

    Tip: vraag de bot om duurzaam gedrag of een voorkeur **in AGENTS.md of MEMORY.md te schrijven** in plaats van te vertrouwen op de chatgeschiedenis.

    Raadpleeg [Agentwerkruimte](/nl/concepts/agent-workspace) en [Geheugen](/nl/concepts/memory).

  </Accordion>

  <Accordion title="Kan ik SOUL.md groter maken?">
    Ja. `SOUL.md` is een van de opstartbestanden uit de werkruimte die in de agentcontext worden geïnjecteerd. De standaardlimiet voor injectie per bestand is `20000` tekens; het totale opstartbudget voor alle bestanden is `60000` tekens.

    Wijzig de gedeelde standaardwaarden:

    ```json5
    {
      agents: {
        defaults: {
          bootstrapMaxChars: 50000,
          bootstrapTotalMaxChars: 300000,
        },
      },
    }
    ```

    Of overschrijf één agent onder `agents.entries.*.bootstrapMaxChars` / `bootstrapTotalMaxChars`.

    Gebruik `/context` om de onbewerkte en geïnjecteerde grootten te controleren en te zien of er afkapping heeft plaatsgevonden. Houd `SOUL.md` gericht op stem, houding en persoonlijkheid; plaats operationele regels in `AGENTS.md` en duurzame feiten in het geheugen.

    Raadpleeg [Context](/nl/concepts/context) en [Agentconfiguratie](/nl/gateway/config-agents).

  </Accordion>

  <Accordion title="Aanbevolen back-upstrategie">
    Plaats je **agentwerkruimte** in een **privé**-git-repository en maak ergens privé een back-up (bijvoorbeeld op GitHub als privérepository). Hiermee leg je het geheugen plus de AGENTS-/SOUL-/USER-bestanden vast en kun je later de „geest” van de assistent herstellen.

    Commit **niets** onder `~/.openclaw` (inloggegevens, sessies, tokens, versleutelde geheime payloads). Maak voor een volledig herstel afzonderlijk een back-up van de werkruimte en de statusmap.

    Documentatie: [Agentwerkruimte](/nl/concepts/agent-workspace).

  </Accordion>

  <Accordion title="Hoe verwijder ik OpenClaw volledig?">
    Raadpleeg [Verwijderen](/nl/install/uninstall).
  </Accordion>

  <Accordion title="Kunnen agents buiten de werkruimte werken?">
    Ja. De werkruimte is de **standaard-cwd** en het geheugenanker, geen harde sandbox. Relatieve paden worden binnen de werkruimte opgelost; absolute paden kunnen andere locaties op de host benaderen, tenzij sandboxing is ingeschakeld. Gebruik voor isolatie [`agents.defaults.sandbox`](/nl/gateway/sandboxing) of sandboxinstellingen per agent. Als je een repository als standaardwerkmap wilt instellen, laat je `workspace` van die agent naar de hoofdmap van de repository verwijzen - de OpenClaw-repository zelf bevat alleen broncode, dus houd de werkruimte apart, tenzij je bewust wilt dat de agent erin werkt.

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Externe modus: waar staat de sessieopslag?">
    De sessiestatus wordt beheerd door de **gateway-host**. In de externe modus staat de relevante sessieopslag op de externe machine, niet op je lokale laptop. Raadpleeg [Sessiebeheer](/nl/concepts/session).
  </Accordion>
</AccordionGroup>

## Basisprincipes van configuratie

<AccordionGroup>
  <Accordion title="Welke indeling gebruikt de configuratie? Waar staat deze?">
    OpenClaw leest een optionele **JSON5**-configuratie uit `$OPENCLAW_CONFIG_PATH` (standaard: `~/.openclaw/openclaw.json`). Als het bestand ontbreekt, gebruikt het redelijk veilige standaardwaarden, waaronder een standaardwerkruimte van `~/.openclaw/workspace`.
  </Accordion>

  <Accordion title='Ik heb gateway.bind ingesteld op "lan" (of "tailnet") en nu luistert er niets / meldt de UI dat ik niet geautoriseerd ben'>
    Bindingen buiten loopback **vereisen een geldig authenticatiepad voor de gateway**: authenticatie met een gedeeld geheim (token of wachtwoord), of `gateway.auth.mode: "trusted-proxy"` achter een correct geconfigureerde identiteitsbewuste reverse proxy.

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    - `gateway.remote.token` / `.password` schakelen lokale gateway-authenticatie **niet** zelfstandig in; lokale aanroeppaden kunnen `gateway.remote.*` alleen als terugvaloptie gebruiken wanneer `gateway.auth.*` niet is ingesteld.
    - Stel voor wachtwoordauthenticatie `gateway.auth.mode: "password"` plus `gateway.auth.password` (of `OPENCLAW_GATEWAY_PASSWORD`) in.
    - Als `gateway.auth.token` / `.password` expliciet via SecretRef is geconfigureerd en niet kan worden opgelost, wordt de toegang standaard geweigerd (zonder maskering door een externe terugvaloptie).
    - Control UI-configuraties met een gedeeld geheim authenticeren via `connect.params.auth.token` of `connect.params.auth.password` (opgeslagen in de app-/UI-instellingen). Modi met identiteitsgegevens, zoals Tailscale Serve of `trusted-proxy`, gebruiken in plaats daarvan aanvraagheaders - plaats geen gedeelde geheimen in URL's.
    - Met `gateway.auth.mode: "trusted-proxy"` vereisen loopback-reverse-proxy's op dezelfde host een expliciete `gateway.auth.trustedProxy.allowLoopback = true` en een loopbackvermelding in `gateway.trustedProxies`.

  </Accordion>

  <Accordion title="Waarom heb ik nu een token nodig op localhost?">
    OpenClaw dwingt gateway-authenticatie standaard af, ook voor loopback. Als er geen expliciet authenticatiepad is geconfigureerd, wordt bij het opstarten de tokenmodus gekozen en uitsluitend voor die start een token gegenereerd. Lokale WS-clients moeten zich daarom authenticeren. Dit voorkomt dat andere lokale processen de Gateway aanroepen.

    Configureer `gateway.auth.token`, `gateway.auth.password`, `OPENCLAW_GATEWAY_TOKEN` of `OPENCLAW_GATEWAY_PASSWORD` expliciet wanneer clients na herstarts hetzelfde geheim nodig hebben. Je kunt ook kiezen voor de wachtwoordmodus of `trusted-proxy` voor identiteitsbewuste reverse proxy's. Stel voor een open loopback expliciet `gateway.auth.mode: "none"` in. `openclaw doctor --generate-gateway-token` genereert op elk moment een token.

  </Accordion>

  <Accordion title="Moet ik opnieuw opstarten nadat ik de configuratie heb gewijzigd?">
    De Gateway bewaakt de configuratie en ondersteunt direct herladen: `gateway.reload.mode: "hybrid"` (standaard) past veilige wijzigingen direct toe en start opnieuw op bij kritieke wijzigingen. `hot`, `restart` en `off` worden ook ondersteund. De meeste wijzigingen aan `tools.*`, het `agents.*`-beleid, `session.*` en `messages.*` worden onmiddellijk toegepast zonder enige herlaadactie; wijzigingen aan de binding/poort van `gateway.*` vereisen een herstart.
  </Accordion>

  <Accordion title="Hoe schakel ik zoeken op het web (en ophalen van het web) in?">
    `web_fetch` werkt zonder API-sleutel. `web_search` is afhankelijk van de geselecteerde provider:

    | Provider | Zonder sleutel | Omgevingsvariabele(n) |
    | --- | --- | --- |
    | Brave | Nee | `BRAVE_API_KEY` |
    | DuckDuckGo | Ja (niet-officieel, op HTML gebaseerd) | - |
    | Exa | Nee | `EXA_API_KEY` |
    | Firecrawl | Nee | `FIRECRAWL_API_KEY` |
    | Gemini | Nee | `GEMINI_API_KEY` |
    | Grok | Nee (xAI OAuth of sleutel) | `XAI_API_KEY` |
    | Kimi | Nee | `KIMI_API_KEY` of `MOONSHOT_API_KEY` |
    | MiniMax Search | Nee | `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` of `MINIMAX_API_KEY` |
    | Ollama Web Search | Ja (vereist `ollama signin`) | - |
    | Perplexity | Nee | `PERPLEXITY_API_KEY` of `OPENROUTER_API_KEY` |
    | SearXNG | Ja (zelf gehost) | `SEARXNG_BASE_URL` |
    | Tavily | Nee | `TAVILY_API_KEY` |

    Grok kan ook xAI OAuth uit modelauthenticatie (`openclaw onboard --auth-choice xai-oauth`) hergebruiken.

    **Aanbevolen**: `openclaw configure --section web` en kies een provider.

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            enabled: true,
            provider: "brave",
            maxResults: 5,
          },
          fetch: {
            enabled: true,
            provider: "firecrawl", // optioneel; weglaten voor automatische detectie
          },
        },
      },
    }
    ```

    Providerspecifieke configuratie voor zoeken op het web bevindt zich onder `plugins.entries.<plugin>.config.webSearch.*`. Verouderde providerpaden van `tools.web.search.*` worden voor compatibiliteit nog steeds geladen, maar mogen niet in nieuwe configuraties worden gebruikt. De configuratie van Firecrawl als terugvaloptie voor ophalen van het web bevindt zich onder `plugins.entries.firecrawl.config.webFetch.*`.

    - Toelatingslijsten: voeg `web_search`/`web_fetch`/`x_search` toe, of `group:web` voor alle drie.
    - `web_fetch` is standaard ingeschakeld.
    - Als `tools.web.fetch.provider` wordt weggelaten, detecteert OpenClaw automatisch de eerste beschikbare terugvalprovider voor ophalen aan de hand van de beschikbare inloggegevens; de officiële Firecrawl-plugin biedt die terugvaloptie.
    - Daemons lezen omgevingsvariabelen uit `~/.openclaw/.env` (of de serviceomgeving).

    Documentatie: [Webtools](/nl/tools/web).

  </Accordion>

  <Accordion title="config.apply heeft mijn configuratie gewist. Hoe herstel en voorkom ik dit?">
    `config.apply` vervangt de **volledige configuratie**; bij een gedeeltelijk object wordt al het overige verwijderd.

    De huidige versie van OpenClaw beschermt tegen de meeste onbedoelde overschrijvingen:

    - Door OpenClaw uitgevoerde configuratiewijzigingen valideren vóór het schrijven de volledige resulterende configuratie.
    - Ongeldige of destructieve schrijfbewerkingen door OpenClaw worden geweigerd en opgeslagen als `openclaw.json.rejected.*`.
    - Als een rechtstreekse bewerking het opstarten of direct herladen verstoort, sluit de Gateway zich veilig af of slaat deze het herladen over; `openclaw.json` wordt niet herschreven.
    - `openclaw doctor --fix` beheert het herstel, kan de laatst bekende werkende versie terugzetten en slaat het geweigerde bestand op als `openclaw.json.clobbered.*`.

    Herstellen:

    - Controleer `openclaw logs --follow` op `Invalid config at`, `Config write rejected:` of `config reload skipped (invalid config)`.
    - Inspecteer de nieuwste `openclaw.json.clobbered.*` of `openclaw.json.rejected.*` naast de actieve configuratie.
    - Voer `openclaw config validate` en `openclaw doctor --fix` uit.
    - Kopieer alleen de bedoelde sleutels terug met `openclaw config set` of `config.patch`.
    - Geen laatst bekende werkende versie of geweigerde payload: herstel vanuit een back-up, of voer `openclaw doctor` opnieuw uit en configureer kanalen/modellen opnieuw.
    - Onverwacht verlies: meld een bug met je laatst bekende configuratie of een back-up. Een lokale codeeragent kan vaak een werkende configuratie reconstrueren aan de hand van logboeken of de geschiedenis.

    Voorkom dit: gebruik `openclaw config set` voor kleine wijzigingen, `openclaw configure` voor interactieve bewerkingen, `config.schema.lookup` om een onbekend pad te inspecteren (retourneert een oppervlakkig schemaknooppunt plus samenvattingen van directe onderliggende elementen) en `config.patch` voor gedeeltelijke RPC-bewerkingen. Reserveer `config.apply` voor vervanging van de volledige configuratie. De agentgerichte runtimetool `gateway` weigert `tools.exec.ask` / `tools.exec.security` te herschrijven, zelfs via verouderde aliassen van `tools.bash.*`.

    Documentatie: [Configuratie](/nl/cli/config), [Configureren](/nl/cli/configure), [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting#gateway-rejected-invalid-config), [Doctor](/nl/gateway/doctor).

  </Accordion>

  <Accordion title="Hoe voer ik een centrale Gateway uit met gespecialiseerde workers op verschillende apparaten?">
    Gebruikelijk patroon: **één Gateway** (bijvoorbeeld een Raspberry Pi) plus **nodes** en **agents**.

    - **Gateway (centraal)**: beheert kanalen (Signal/WhatsApp), routering en sessies.
    - **Nodes (apparaten)**: Macs/iOS/Android maken verbinding als randapparaten en stellen lokale tools beschikbaar (`system.run`, `canvas`, `camera`).
    - **Agents (workers)**: afzonderlijke breinen/werkruimten voor speciale rollen (bijvoorbeeld beheer versus persoonlijke gegevens).
    - **Subagents**: starten achtergrondwerk vanuit een hoofdagent om parallel te werken.
    - **TUI**: maakt verbinding met de Gateway en wisselt tussen agents/sessies.

    Documentatie: [Nodes](/nl/nodes), [Externe toegang](/nl/gateway/remote), [Routering met meerdere agents](/nl/concepts/multi-agent), [Subagents](/nl/tools/subagents), [TUI](/nl/web/tui).

  </Accordion>

  <Accordion title="Kan de OpenClaw-browser headless worden uitgevoerd?">
    Ja:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    De standaardwaarde is `false` (met zichtbare interface). Headless activeert op sommige sites vaker antibotcontroles (X/Twitter blokkeert headless-sessies vaak). Het gebruikt dezelfde Chromium-engine en werkt voor de meeste automatisering; het belangrijkste verschil is dat er geen zichtbaar browservenster is (gebruik schermafbeeldingen voor visuele controle). Zie [Browser](/nl/tools/browser).

  </Accordion>

  <Accordion title="Hoe gebruik ik Brave voor browserbesturing?">
    Stel `browser.executablePath` in op je Brave-binaire bestand (of een andere op Chromium gebaseerde browser) en start de Gateway opnieuw. Zie [Browser](/nl/tools/browser#use-brave-or-another-chromium-based-browser).
  </Accordion>
</AccordionGroup>

## Externe gateways en nodes

<AccordionGroup>
  <Accordion title="Hoe worden opdrachten doorgegeven tussen Telegram, de gateway en nodes?">
    Telegram-berichten worden verwerkt door de **gateway**, die de agent uitvoert en pas daarna nodes via de **Gateway WebSocket** aanroept wanneer een nodetool nodig is:

    Telegram -> Gateway -> Agent -> `node.*` -> Node -> Gateway -> Telegram

    Nodes zien geen inkomend providerverkeer; ze ontvangen alleen RPC-aanroepen voor nodes.

  </Accordion>

  <Accordion title="Hoe krijgt mijn agent toegang tot mijn computer als de Gateway extern wordt gehost?">
    Koppel je computer als een **node**. De Gateway wordt elders uitgevoerd, maar kan via de Gateway WebSocket `node.*`-tools (scherm, camera, systeem) op je lokale computer aanroepen.

    1. Voer de Gateway uit op de host die altijd actief is (VPS/thuisserver).
    2. Plaats de Gateway-host en je computer op hetzelfde tailnet.
    3. Zorg dat de Gateway-WS bereikbaar is (binding aan het tailnet of SSH-tunnel).
    4. Open de macOS-app lokaal en maak verbinding in de modus **Remote over SSH** (of rechtstreeks via het tailnet), zodat deze als node wordt geregistreerd.
    5. Keur de node goed:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Er is geen afzonderlijke TCP-bridge nodig; nodes maken verbinding via de Gateway WebSocket.

    Beveiligingsherinnering: door een macOS-node te koppelen, wordt `system.run` op die machine toegestaan. Koppel alleen apparaten die je vertrouwt; raadpleeg [Beveiliging](/nl/gateway/security).

    Documentatie: [Nodes](/nl/nodes), [Gateway-protocol](/nl/gateway/protocol), [Externe modus van macOS](/nl/platforms/mac/remote), [Beveiliging](/nl/gateway/security).

  </Accordion>

  <Accordion title="Tailscale is verbonden, maar ik krijg geen antwoorden. Wat nu?">
    Controleer de basiszaken:

    ```bash
    openclaw gateway status
    openclaw status
    openclaw channels status
    ```

    Controleer vervolgens authenticatie en routering: als je Tailscale Serve gebruikt, controleer dan of `gateway.auth.allowTailscale` correct is ingesteld; als je via een SSH-tunnel verbinding maakt, controleer dan of de tunnel actief is en naar de juiste poort verwijst; controleer of je account in de toelatingslijsten voor privéberichten/groepen staat.

    Documentatie: [Tailscale](/nl/gateway/tailscale), [Externe toegang](/nl/gateway/remote), [Kanalen](/nl/channels).

  </Accordion>

  <Accordion title="Kunnen twee OpenClaw-instanties met elkaar communiceren (lokaal + VPS)?">
    Ja, hoewel er geen ingebouwde bridge tussen bots is.

    **Eenvoudigste optie**: gebruik een normaal chatkanaal waartoe beide bots toegang hebben (Slack/Telegram/WhatsApp). Laat Bot A een bericht naar Bot B sturen en laat Bot B vervolgens zoals gewoonlijk antwoorden.

    **CLI-bridge (algemeen)**: voer een script uit dat met `openclaw agent --message ... --deliver` de andere Gateway aanroept en zich richt op een chat waarin de andere bot luistert. Als één bot op een externe VPS staat, richt je CLI dan via SSH/Tailscale op die externe Gateway (zie [Externe toegang](/nl/gateway/remote)):

    ```bash
    openclaw agent --message "Hallo van de lokale bot" --deliver --channel telegram --reply-to <chat-id>
    ```

    Voeg een beveiliging toe zodat de twee bots niet eindeloos blijven reageren (alleen bij vermeldingen, toelatingslijsten voor kanalen of een regel "niet antwoorden op botberichten").

    Documentatie: [Externe toegang](/nl/gateway/remote), [Agent-CLI](/nl/cli/agent), [Verzenden door agents](/nl/tools/agent-send).

  </Accordion>

  <Accordion title="Heb ik afzonderlijke VPS'en nodig voor meerdere agents?">
    Nee. Eén Gateway host meerdere agents, elk met een eigen werkruimte, standaardmodellen en routering. Dit is de normale configuratie en veel goedkoper/eenvoudiger dan één VPS per agent. Gebruik alleen afzonderlijke VPS'en voor strikte isolatie (beveiligingsgrenzen) of zeer verschillende configuraties die je niet wilt delen.
  </Accordion>

  <Accordion title="Heeft het voordelen om een node op mijn persoonlijke laptop te gebruiken in plaats van SSH vanaf een VPS?">
    Ja: nodes zijn de primaire manier om je laptop vanaf een externe Gateway te bereiken en bieden meer mogelijkheden dan alleen shelltoegang. De Gateway draait op macOS/Linux (Windows via WSL2) en is lichtgewicht (een kleine VPS of een apparaat van Raspberry Pi-klasse volstaat; 4 GB RAM is ruim voldoende), dus een gebruikelijke configuratie is een host die altijd actief is, met je laptop als node.

    - **Geen inkomende SSH vereist** - nodes maken via apparaatkoppeling een uitgaande verbinding met de Gateway WebSocket.
    - **Veiligere uitvoeringscontroles** - `system.run` wordt op die laptop beperkt door toelatingslijsten/goedkeuringen voor nodes.
    - **Meer apparaathulpmiddelen** - naast `system.run` stellen nodes ook `canvas`, `camera` en `screen` beschikbaar.
    - **Lokale browserautomatisering** - houd de Gateway op een VPS, maar voer Chrome lokaal uit via een nodehost, of maak via Chrome MCP verbinding met de lokale Chrome-installatie.

    SSH is geschikt voor incidentele shelltoegang; nodes zijn eenvoudiger voor doorlopende agentworkflows en apparaatautomatisering.

    Documentatie: [Nodes](/nl/nodes), [Nodes-CLI](/nl/cli/nodes), [Browser](/nl/tools/browser).

  </Accordion>

  <Accordion title="Voeren nodes een gatewayservice uit?">
    Nee. Er mag slechts **één gateway** per host worden uitgevoerd, tenzij je bewust geïsoleerde profielen uitvoert (zie [Meerdere gateways](/nl/gateway/multiple-gateways)). Nodes zijn randapparaten die verbinding maken met de gateway (iOS-/Android-nodes of de macOS-"nodemodus" in de menubalk-app). Zie [Nodehost-CLI](/nl/cli/node) voor headless-nodehosts en CLI-besturing.

    Een volledige herstart is vereist voor `gateway`, `discovery` en wijzigingen aan het oppervlak van gehoste plugins.

  </Accordion>

  <Accordion title="Is er een API-/RPC-methode om configuratie toe te passen?">
    Ja:

    - `config.schema.lookup`: inspecteer vóór het schrijven één configuratiesubstructuur met het oppervlakkige schemaknooppunt, de overeenkomende UI-hint en samenvattingen van directe onderliggende elementen.
    - `config.get`: haal de huidige momentopname plus hash op.
    - `config.patch`: veilige gedeeltelijke update (aanbevolen voor de meeste RPC-bewerkingen); herlaadt direct wanneer mogelijk en start opnieuw wanneer vereist.
    - `config.apply`: valideer en vervang de volledige configuratie; herlaadt direct wanneer mogelijk en start opnieuw wanneer vereist.
    - De agentgerichte runtimetool `gateway` weigert nog steeds `tools.exec.ask` / `tools.exec.security` te herschrijven; verouderde aliassen van `tools.bash.*` worden genormaliseerd naar dezelfde beschermde paden.

  </Accordion>

  <Accordion title="Minimale verstandige configuratie voor een eerste installatie">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    Stelt je werkruimte in en beperkt wie de bot kan activeren.

  </Accordion>

  <Accordion title="Hoe stel ik Tailscale in op een VPS en maak ik verbinding vanaf mijn Mac?">
    1. **Installeren en aanmelden op de VPS**:
       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```
    2. **Installeren en aanmelden op je Mac** met de Tailscale-app, op hetzelfde tailnet.
    3. **MagicDNS inschakelen** in de Tailscale-beheerconsole, zodat de VPS een stabiele naam heeft.
    4. **De tailnet-hostnaam gebruiken**: SSH `ssh user@your-vps.tailnet-xxxx.ts.net`; Gateway-WS `ws://your-vps.tailnet-xxxx.ts.net:18789`.

    Gebruik Tailscale Serve op de VPS voor de Control UI zonder SSH:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Hierdoor blijft de Gateway aan loopback gebonden en wordt HTTPS via Tailscale beschikbaar gesteld. Zie [Tailscale](/nl/gateway/tailscale).

  </Accordion>

  <Accordion title="Hoe verbind ik een Mac-Node met een externe Gateway (Tailscale Serve)?">
    Serve stelt de **Gateway Control UI + WS** beschikbaar; Nodes maken verbinding via hetzelfde Gateway-WS-eindpunt.

    1. Zorg dat de VPS en Mac zich op hetzelfde tailnet bevinden.
    2. Gebruik de macOS-app in de externe modus (het SSH-doel kan de tailnet-hostnaam zijn). Deze maakt een tunnel voor de Gateway-poort en maakt verbinding als Node.
    3. Keur de Node goed:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    Documentatie: [Gateway-protocol](/nl/gateway/protocol), [Detectie](/nl/gateway/discovery), [externe modus van macOS](/nl/platforms/mac/remote).

  </Accordion>

  <Accordion title="Moet ik OpenClaw op een tweede laptop installeren of alleen een Node toevoegen?">
    Voor alleen **lokale tools** (scherm/camera/exec) op de tweede laptop voeg je deze toe als **Node**: één Gateway, zonder dubbele configuratie. Lokale Node-tools zijn momenteel alleen beschikbaar voor macOS. Installeer alleen een tweede Gateway voor **strikte isolatie** of twee volledig afzonderlijke bots.

    Documentatie: [Nodes](/nl/nodes), [Nodes-CLI](/nl/cli/nodes), [Meerdere Gateways](/nl/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## Omgevingsvariabelen en het laden van .env

<AccordionGroup>
  <Accordion title="Hoe laadt OpenClaw omgevingsvariabelen?">
    OpenClaw leest omgevingsvariabelen uit het bovenliggende proces (shell, launchd/systemd, CI enzovoort) en laadt daarnaast:

    - `.env` uit de huidige werkmap.
    - een algemene terugvaloptie `.env` uit `~/.openclaw/.env` (`$OPENCLAW_STATE_DIR/.env`).

    Geen van beide `.env`-bestanden overschrijft bestaande omgevingsvariabelen. Sleutels voor providerreferenties en eindpuntroutering vormen een uitzondering voor `.env` in de werkruimte: sleutels zoals `GEMINI_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY` en elke sleutel die eindigt op `_ENDPOINT` (evenals andere omgevingsvariabelen voor authenticatie of eindpunten van meegeleverde providers) worden genegeerd in `.env` van de werkruimte en horen thuis in de procesomgeving, `~/.openclaw/.env` of configuratie `env`.

    In de configuratie opgenomen omgevingsvariabelen worden alleen toegepast als ze in de procesomgeving ontbreken:

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    Zie [/environment](/nl/help/environment) voor de volledige prioriteitsvolgorde en bronnen.

  </Accordion>

  <Accordion title="Ik heb de Gateway via de service gestart en mijn omgevingsvariabelen zijn verdwenen. Wat nu?">
    Twee oplossingen:

    1. Plaats de ontbrekende sleutels in `~/.openclaw/.env`, zodat ze ook worden geladen wanneer de service je shell-omgeving niet overneemt.
    2. Schakel shell-import in (optioneel voor extra gemak):
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
       Hiermee wordt je aanmeldingsshell uitgevoerd en worden alleen ontbrekende verwachte sleutels geïmporteerd (bestaande waarden worden nooit overschreven). Overeenkomstige omgevingsvariabelen: `OPENCLAW_LOAD_SHELL_ENV=1`, `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='Ik heb COPILOT_GITHUB_TOKEN ingesteld, maar de modelstatus toont "Shell env: off." Waarom?'>
    `openclaw models status` meldt of **import van de shell-omgeving** is ingeschakeld. "Shell env: off" betekent **niet** dat je omgevingsvariabelen ontbreken; het betekent alleen dat OpenClaw je aanmeldingsshell niet automatisch laadt.

    Als de Gateway als service wordt uitgevoerd (launchd/systemd), neemt deze je shell-omgeving niet over. Los dit op door het token in `~/.openclaw/.env` te plaatsen, `env.shellEnv.enabled: true` in te schakelen of het toe te voegen aan configuratie `env` (wordt alleen toegepast als het ontbreekt). Start daarna de Gateway opnieuw en controleer nogmaals:

    ```bash
    openclaw models status
    ```

    Copilot-tokens worden in deze volgorde opgezocht: `OPENCLAW_GITHUB_TOKEN`, vervolgens `COPILOT_GITHUB_TOKEN`, daarna `GH_TOKEN` en ten slotte `GITHUB_TOKEN`.

    Zie [/concepts/model-providers](/nl/concepts/model-providers) en [/environment](/nl/help/environment).

  </Accordion>
</AccordionGroup>

## Sessies en meerdere chats

<AccordionGroup>
  <Accordion title="Hoe start ik een nieuw gesprek?">
    Stuur `/new` of `/reset` als afzonderlijk bericht. Zie [Sessiebeheer](/nl/concepts/session).
  </Accordion>

  <Accordion title="Worden sessies automatisch opnieuw ingesteld als ik nooit /new stuur?">
    Nee, niet standaard. Sessies behouden dezelfde `sessionId` en Compaction begrenst de actieve modelcontext naarmate gesprekken langer worden. `/new` en `/reset` blijven beschikbaar, of je kunt automatische resets inschakelen met `mode: "daily"` of `mode: "idle"`. De dagelijkse modus schakelt om op `session.reset.atHour` (standaard `4`, 0-23) op de Gateway-host; de inactiviteitsmodus gebruikt `session.reset.idleMinutes` sinds de laatste echte interactie, niet sinds Heartbeat-/Cron-/exec-systeemgebeurtenissen.

    ```json5
    {
      session: {
        reset: { mode: "daily", atHour: 4 },
        resetByType: {
          group: { mode: "idle", idleMinutes: 120 },
          thread: { mode: "daily", atHour: 6 },
        },
        resetByChannel: {
          discord: { mode: "idle", idleMinutes: 10080 },
        },
      },
    }
    ```

    `resetByType` ondersteunt `direct`, `group` en `thread`. Doctor migreert verouderde `dm`-vermeldingen naar `direct`; het schema weigert `dm`. De verouderde `session.idleMinutes` op het hoogste niveau werkt nog als compatibiliteitsalias voor een standaardwaarde in inactiviteitsmodus wanneer geen `session.reset`-/`resetByType`-blok is ingesteld. Zie [Sessiebeheer](/nl/concepts/session) voor de volledige levenscyclus.

  </Accordion>

  <Accordion title="Kan ik een team van OpenClaw-instanties maken (één CEO en veel agents)?">
    Ja, via **routering met meerdere agents** en **subagents**: één coördinerende agent plus meerdere uitvoerende agents met hun eigen werkruimten en modellen.

    Dit kun je het beste zien als een leuk experiment: het verbruikt veel tokens en is vaak minder efficiënt dan één bot met afzonderlijke sessies. Het gebruikelijke model is één bot waarmee je praat, met verschillende sessies voor parallel werk en subagents die waar nodig worden gestart.

    Documentatie: [Routering met meerdere agents](/nl/concepts/multi-agent), [Subagents](/nl/tools/subagents), [Agents-CLI](/nl/cli/agents).

  </Accordion>

  <Accordion title="Waarom is de context halverwege een taak afgekapt? Hoe voorkom ik dit?">
    De sessiecontext wordt beperkt door het contextvenster van het model. Lange chats, omvangrijke tooluitvoer of veel bestanden kunnen Compaction of afkapping veroorzaken.

    - Vraag de bot om de huidige status samen te vatten en naar een bestand te schrijven.
    - Gebruik `/compact` vóór lange taken en `/new` wanneer je van onderwerp wisselt.
    - Bewaar belangrijke context in de werkruimte en vraag de bot deze opnieuw te lezen.
    - Gebruik subagents voor lang of parallel werk, zodat de hoofdchat kleiner blijft.
    - Kies een model met een groter contextvenster als dit vaak gebeurt.

  </Accordion>

  <Accordion title="Hoe stel ik OpenClaw volledig opnieuw in zonder het te verwijderen?">
    ```bash
    openclaw reset
    ```

    Volledige niet-interactieve reset:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    Voer daarna de installatie opnieuw uit:

    ```bash
    openclaw onboard --install-daemon
    ```

    Onboarding biedt ook **Opnieuw instellen** aan als een bestaande configuratie wordt gedetecteerd; zie [Onboarding (CLI)](/nl/start/wizard). Als je profielen hebt gebruikt (`--profile` / `OPENCLAW_PROFILE`), stel dan elke statusmap opnieuw in (standaard `~/.openclaw-<profile>`). Reset alleen voor ontwikkeling: `openclaw gateway --dev --reset` wist de ontwikkelconfiguratie, referenties, sessies en werkruimte.

  </Accordion>

  <Accordion title='Ik krijg fouten met "context too large". Hoe stel ik opnieuw in of pas ik Compaction toe?'>
    - **Compaction** (behoudt het gesprek en vat oudere beurten samen): `/compact` of `/compact <instructions>` om de samenvatting te sturen.
    - **Opnieuw instellen** (nieuwe sessie-ID voor dezelfde chatsleutel): `/new` of `/reset`.

    Als dit blijft gebeuren, pas dan **sessieopschoning** (`agents.defaults.contextPruning`) aan om oude tooluitvoer in te korten, of gebruik een model met een groter contextvenster.

    Documentatie: [Compaction](/nl/concepts/compaction), [Sessieopschoning](/nl/concepts/session-pruning), [Sessiebeheer](/nl/concepts/session).

  </Accordion>

  <Accordion title='Waarom zie ik "LLM request rejected: messages.content.tool_use.input field required"?'>
    Validatiefout van de provider: het model heeft een `tool_use`-blok uitgevoerd zonder de vereiste `input`. Dit betekent doorgaans dat de sessiegeschiedenis verouderd of beschadigd is, vaak na lange threads of een wijziging aan een tool of schema.

    Oplossing: start een nieuwe sessie met `/new` (als afzonderlijk bericht).

  </Accordion>

  <Accordion title="Waarom krijg ik elke 30 minuten Heartbeat-berichten?">
    Heartbeats worden standaard elke **30m** uitgevoerd, of elke **1h** wanneer de vastgestelde authenticatiemodus Anthropic OAuth-/tokenauthenticatie is (inclusief hergebruik van de Claude-CLI) en `heartbeat.every` niet is ingesteld. Pas dit aan of schakel het uit:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // of "0m" om uit te schakelen
          },
        },
      },
    }
    ```

    Als `HEARTBEAT.md` bestaat maar feitelijk leeg is (alleen lege regels, Markdown-/HTML-opmerkingen, ATX-koppen, fence-markeringen of lege lijstitems), slaat OpenClaw de Heartbeat-uitvoering over om API-aanroepen te besparen. Als het bestand ontbreekt, wordt de Heartbeat nog steeds uitgevoerd en bepaalt het model wat er moet gebeuren.

    Overschrijvingen per agent gebruiken `agents.entries.*.heartbeat`. Documentatie: [Heartbeat](/nl/gateway/heartbeat).

  </Accordion>

  <Accordion title='Moet ik een "botaccount" aan een WhatsApp-groep toevoegen?'>
    Nee. OpenClaw wordt uitgevoerd op **je eigen account**: als je in de groep zit, kan OpenClaw deze zien. Standaard worden groepsantwoorden geblokkeerd totdat je afzenders toestaat (`groupPolicy: "allowlist"`).

    Zo beperk je groepsantwoorden tot alleen jezelf:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Hoe vind ik de JID van een WhatsApp-groep?">
    Het snelst: volg de logboeken en stuur een testbericht in de groep.

    ```bash
    openclaw logs --follow --json
    ```

    Zoek naar `chatId` (of `from`) die eindigt op `@g.us`, zoals `1234567890-1234567890@g.us`.

    Als de groep al is geconfigureerd of op de toelatingslijst staat, geef je de groepen uit de configuratie weer:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    Documentatie: [WhatsApp](/nl/channels/whatsapp), [Directory](/nl/cli/directory), [Logboeken](/nl/cli/logs).

  </Accordion>

  <Accordion title="Waarom antwoordt OpenClaw niet in een groep?">
    Twee veelvoorkomende oorzaken: de vermeldingsbeperking is standaard ingeschakeld (je moet de bot met @ vermelden of overeenkomen met `mentionPatterns`), of je hebt `channels.whatsapp.groups` zonder `"*"` geconfigureerd en de groep staat niet op de toelatingslijst.

    Zie [Groepen](/nl/channels/groups) en [Groepsberichten](/nl/channels/group-messages).

  </Accordion>

  <Accordion title="Delen groepen/threads context met privéberichten?">
    Rechtstreekse chats worden standaard samengevoegd met de hoofdsessie. Groepen/kanalen hebben hun eigen sessiesleutels en Telegram-onderwerpen / Discord-threads zijn afzonderlijke sessies. Zie [Groepen](/nl/channels/groups) en [Groepsberichten](/nl/channels/group-messages).
  </Accordion>

  <Accordion title="Hoeveel werkruimten en agents kan ik maken?">
    Er zijn geen harde limieten: tientallen of zelfs honderden zijn mogelijk, maar let op:

    - **Schijfgroei**: actieve sessies en transcripties bevinden zich in de SQLite-database per agent; verouderde/archiefartefacten kunnen zich nog steeds ophopen onder `~/.openclaw/agents/<agentId>/sessions/`.
    - **Tokenkosten**: meer agents betekent meer gelijktijdig modelgebruik.
    - **Operationele overhead**: authenticatieprofielen, werkruimten en kanaalroutering per agent.

    Houd één **actieve** werkruimte per agent (`agents.defaults.workspace`), ruim oude sessies op met `openclaw sessions cleanup` als het schijfgebruik toeneemt (bewerk de actieve SQLite-status niet handmatig) en gebruik `openclaw doctor` om achtergebleven werkruimten en niet-overeenkomende profielen op te sporen.

  </Accordion>

  <Accordion title="Kan ik meerdere bots of chats tegelijk uitvoeren (Slack), en hoe moet ik dat instellen?">
    Ja, via **routering met meerdere agents**: voer meerdere geïsoleerde agents uit en routeer inkomende berichten op basis van kanaal/account/peer. Slack wordt als kanaal ondersteund en kan aan specifieke agents worden gekoppeld.

    Browsertoegang is krachtig, maar kan niet zomaar "alles doen wat een mens kan" — antibotmaatregelen, CAPTCHA's en MFA kunnen automatisering nog steeds blokkeren. Gebruik voor de betrouwbaarste besturing lokale Chrome MCP op de host, of CDP op de machine waarop de browser daadwerkelijk wordt uitgevoerd.

    Aanbevolen configuratie: een Gateway-host die altijd actief is (VPS/Mac mini), één agent per rol (koppelingen), Slack-kanaal of -kanalen die aan deze agents zijn gekoppeld, en indien nodig een lokale browser via Chrome MCP of een Node.

    Documentatie: [Routering met meerdere agents](/nl/concepts/multi-agent), [Slack](/nl/channels/slack), [Browser](/nl/tools/browser), [Nodes](/nl/nodes).

  </Accordion>
</AccordionGroup>

## Modellen, failover en authenticatieprofielen

Vragen en antwoorden over modellen — standaardinstellingen, selectie, aliassen, omschakelen, failover en authenticatieprofielen — staan in de [veelgestelde vragen over modellen](/nl/help/faq-models).

## Gateway: poorten, "wordt al uitgevoerd" en externe modus

<AccordionGroup>
  <Accordion title="Welke poort gebruikt de Gateway?">
    `gateway.port` beheert de enkele gemultiplexte poort voor WebSocket + HTTP (Control UI, hooks enzovoort). Prioriteitsvolgorde:

    ```text
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > standaard 18789
    ```

  </Accordion>

  <Accordion title='Waarom meldt openclaw gateway status "Runtime: running", maar "Connectivity probe: failed"?'>
    "Running" is de weergave van de **supervisor** (launchd/systemd/schtasks); bij de connectiviteitscontrole maakt de CLI daadwerkelijk verbinding met de WebSocket van de Gateway. Vertrouw op deze regels uit `openclaw gateway status`: `Probe target:` (de URL die de controle gebruikte), `Listening:` (wat daadwerkelijk aan de poort is gebonden), `Last gateway error:` (veelvoorkomende hoofdoorzaak wanneer het proces actief is, maar de poort niet luistert).
  </Accordion>

  <Accordion title='Waarom toont openclaw gateway status verschillende waarden voor "Config (cli)" en "Config (service)"?'>
    Je bewerkt het ene configuratiebestand terwijl de service een ander gebruikt (vaak door een verschil tussen `--profile` en `OPENCLAW_STATE_DIR`).

    Oplossing: voer dit uit vanuit dezelfde `--profile` / omgeving die de service moet gebruiken:

    ```bash
    openclaw gateway install --force
    ```

  </Accordion>

  <Accordion title='Wat betekent "another gateway instance is already listening"?'>
    OpenClaw dwingt een runtimevergrendeling af door de WebSocket-listener onmiddellijk bij het opstarten te binden (standaard `ws://127.0.0.1:18789`). Als het binden mislukt met `EADDRINUSE`, wordt `GatewayLockError` ("another gateway instance is already listening") gegenereerd.

    Oplossing: stop de andere instantie, maak de poort vrij of voer uit met `openclaw gateway --port <port>`.

  </Accordion>

  <Accordion title="Hoe voer ik OpenClaw uit in externe modus (waarbij de client verbinding maakt met een Gateway elders)?">
    Stel `gateway.mode: "remote"` in en verwijs naar een externe WebSocket-URL, eventueel met externe referenties voor een gedeeld geheim:

    ```json5
    {
      gateway: {
        mode: "remote",
        remote: {
          url: "ws://gateway.tailnet:18789",
          token: "your-token",
          password: "your-password",
        },
      },
    }
    ```

    - `openclaw gateway` wordt alleen gestart wanneer `gateway.mode` gelijk is aan `local` (of wanneer je een overschrijvingsvlag meegeeft).
    - De macOS-app bewaakt het configuratiebestand en schakelt direct tussen modi wanneer deze waarden veranderen.
    - `gateway.remote.token` / `.password` zijn uitsluitend externe referenties aan de clientzijde; ze schakelen op zichzelf geen lokale Gateway-authenticatie in.

  </Accordion>

  <Accordion title='De Control UI meldt "unauthorized" (of blijft opnieuw verbinding maken). Wat nu?'>
    Het authenticatiepad van je Gateway en de authenticatiemethode van de UI komen niet overeen.

    Feiten (uit de code):

    - De Control UI bewaart het token in `sessionStorage`, beperkt tot het huidige browsertabblad en de geselecteerde Gateway-URL, zodat vernieuwen in hetzelfde tabblad blijft werken zonder langdurige tokenopslag in localStorage.
    - Bij `AUTH_TOKEN_MISMATCH` kunnen vertrouwde clients één begrensde nieuwe poging uitvoeren met een gecachet apparaattoken wanneer de Gateway aanwijzingen voor een nieuwe poging retourneert (`canRetryWithDeviceToken=true`, `recommendedNextStep=retry_with_device_token`).
    - Die nieuwe poging met het gecachete token hergebruikt de gecachete goedgekeurde bereiken die bij het apparaattoken zijn opgeslagen; expliciete aanroepers van `deviceToken` / expliciete aanroepers van `scopes` behouden hun aangevraagde reeks bereiken in plaats van gecachete bereiken over te nemen.
    - Buiten dat pad voor een nieuwe poging geldt bij verbindingsauthenticatie deze prioriteitsvolgorde: eerst het expliciete gedeelde token/wachtwoord, vervolgens expliciete `deviceToken`, daarna het opgeslagen apparaattoken en ten slotte het bootstrap-token.
    - De ingebouwde bootstrap met installatiecode retourneert een Node-apparaattoken met `scopes: []`, plus een begrensd operatoroverdrachtstoken voor vertrouwde mobiele onboarding. De operatoroverdracht kan tijdens de installatie de native configuratie lezen, maar verleent geen bereiken voor koppelingswijzigingen of `operator.admin`.

    Oplossing:

    - Snelste optie: `openclaw dashboard` (drukt de dashboard-URL af en kopieert deze, probeert deze te openen en toont een SSH-tip als er geen grafische omgeving is).
    - Nog geen token: `openclaw doctor --generate-gateway-token`.
    - Extern: maak eerst een tunnel met `ssh -N -L 18789:127.0.0.1:18789 user@host` en open vervolgens `http://127.0.0.1:18789/`.
    - Modus met gedeeld geheim: stel `gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` of `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` in en plak vervolgens het overeenkomende geheim in de instellingen van de Control UI.
    - Tailscale Serve-modus: controleer of `gateway.auth.allowTailscale` is ingeschakeld en of je de Serve-URL opent, niet een rechtstreekse loopback-/tailnet-URL die de Tailscale-identiteitsheaders omzeilt.
    - Modus met vertrouwde proxy: controleer of je verbinding maakt via de geconfigureerde identiteitsbewuste proxy. Loopback-proxy's op dezelfde host hebben ook `gateway.auth.trustedProxy.allowLoopback = true` nodig.
    - Blijft het verschil na die ene nieuwe poging bestaan, roteer het gekoppelde apparaattoken of keur het opnieuw goed:
      ```bash
      openclaw devices list
      openclaw devices rotate --device <id> --role operator
      ```
    - Rotatie geweigerd: sessies van gekoppelde apparaten kunnen alleen hun **eigen** apparaat roteren, tenzij ze ook `operator.admin` hebben, en expliciete waarden voor `--scope` mogen de huidige operatorbereiken van de aanroeper niet overschrijden.
    - Nog steeds vastgelopen: `openclaw status --all` plus [Problemen oplossen](/nl/gateway/troubleshooting). Zie [Dashboard](/nl/web/dashboard) voor authenticatiedetails.

  </Accordion>

  <Accordion title="Ik heb gateway.bind ingesteld op tailnet, maar er wordt alleen op loopback geluisterd">
    De binding `tailnet` kiest een Tailscale-IP uit je netwerkinterfaces (100.64.0.0/10). Als de machine niet met Tailscale is verbonden (of de interface niet actief is), valt de Gateway terug op loopback in plaats van een andere netwerkinterface beschikbaar te stellen.

    Oplossing: start Tailscale op die host en herstart de Gateway, of schakel expliciet over naar `gateway.bind: "loopback"` / `"lan"`.

    `tailnet` is expliciet; `auto` geeft de voorkeur aan loopback. Gebruik `gateway.bind: "tailnet"` om blootstelling buiten loopback te beperken tot de Tailnet, terwijl de vereiste `127.0.0.1`-listener op dezelfde host behouden blijft.

  </Accordion>

  <Accordion title="Kan ik meerdere Gateways op dezelfde host uitvoeren?">
    Gewoonlijk niet — één Gateway kan meerdere berichtenkanalen en agents uitvoeren. Gebruik meerdere Gateways alleen voor redundantie (bijvoorbeeld een reddingsbot) of strikte isolatie, en isoleer elke Gateway met een eigen `OPENCLAW_CONFIG_PATH`, `OPENCLAW_STATE_DIR`, `agents.defaults.workspace` en unieke `gateway.port`.

    Aanbevolen: `openclaw --profile <name> ...` per instantie (maakt automatisch `~/.openclaw-<name>` aan), een unieke `gateway.port` per profielconfiguratie (of `--port` voor handmatige uitvoeringen) en een service per profiel met `openclaw --profile <name> gateway install`.

    Profielen voegen ook achtervoegsels aan servicenamen toe: launchd `ai.openclaw.<profile>`, systemd `openclaw-gateway-<profile>.service`, Windows `OpenClaw Gateway (<profile>)`. De systemd-eenheid `openclaw-gateway` zonder kwalificatie bestaat alleen voor het standaardprofiel; de verouderde naam van de systemd-eenheid van vóór de naamswijziging, `clawdbot-gateway`, wordt automatisch gemigreerd.

    Volledige handleiding: [Meerdere Gateways](/nl/gateway/multiple-gateways).

  </Accordion>

  <Accordion title='Wat betekent "invalid handshake" / code 1008?'>
    De Gateway is een **WebSocket-server** en verwacht dat het eerste bericht een `connect`-frame is. Iets anders sluit de verbinding met **code 1008** (beleidsschending).

    Veelvoorkomende oorzaken: je hebt de **HTTP**-URL in een browser geopend in plaats van een WS-client te gebruiken, je hebt de verkeerde poort/het verkeerde pad gebruikt, of een proxy/tunnel heeft authenticatieheaders verwijderd of een verzoek verzonden dat niet voor de Gateway bestemd was.

    Oplossing: gebruik de WS-URL (`ws://<host>:18789`, of `wss://...` via HTTPS), open de WS-poort niet in een normaal browsertabblad en neem het token/wachtwoord op in het `connect`-frame wanneer authenticatie is ingeschakeld. CLI/TUI-voorbeeld:

    ```bash
    openclaw tui --url ws://<host>:18789 --token <token>
    ```

    Protocoldetails: [Gateway-protocol](/nl/gateway/protocol).

  </Accordion>
</AccordionGroup>

## Logboekregistratie en foutopsporing

<AccordionGroup>
  <Accordion title="Waar staan de logboeken?">
    Bestandslogboeken (gestructureerd): `/tmp/openclaw/openclaw-YYYY-MM-DD.log` voor het standaardprofiel, of `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` voor een benoemd profiel. Stel een stabiel pad in via `logging.file`; het logniveau voor bestanden via `logging.level`; de uitgebreidheid van console-uitvoer via `--verbose` en `logging.consoleLevel`.

    Snelst volgen:

    ```bash
    openclaw logs --follow
    ```

    Service-/supervisorlogboeken (wanneer de Gateway via launchd/systemd wordt uitgevoerd):

    - macOS launchd-stdout: `~/Library/Logs/openclaw/gateway.log` (profielen gebruiken `gateway-<profile>.log`; stderr wordt onderdrukt).
    - Linux: `journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`.
    - Windows: `schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`.

    Zie [Problemen oplossen](/nl/gateway/troubleshooting) voor meer informatie.

  </Accordion>

  <Accordion title="Hoe start, stop of herstart ik de Gateway-service?">
    ```bash
    openclaw gateway status
    openclaw gateway restart
    ```

    Als je de Gateway handmatig uitvoert, kan `openclaw gateway --force` de poort terugvorderen. Zie [Gateway](/nl/gateway).

  </Accordion>

  <Accordion title="Ik heb mijn terminal in Windows gesloten — hoe herstart ik OpenClaw?">
    Drie Windows-installatiemodi:

    **1) Lokale configuratie van Windows Hub**: de native app beheert een lokale WSL-Gateway waarvan de app eigenaar is. Open **OpenClaw Companion** vanuit het menu Start of het systeemvak en gebruik vervolgens **Gateway Setup** of het tabblad Connections.

    **2) Handmatige WSL2-Gateway**: de Gateway wordt binnen Linux uitgevoerd.
    ```powershell
    wsl
    openclaw gateway status
    openclaw gateway restart
    ```
    Als je de service nooit hebt geïnstalleerd, start je deze op de voorgrond: `openclaw gateway run`.

    **3) Native Windows-CLI/Gateway**: wordt rechtstreeks in Windows uitgevoerd.
    ```powershell
    openclaw gateway status
    openclaw gateway restart
    ```
    Als je deze handmatig uitvoert (zonder service): `openclaw gateway run`.

    Documentatie: [Windows](/nl/platforms/windows), [draaiboek voor de Gateway-service](/nl/gateway).

  </Accordion>

  <Accordion title="De Gateway is actief, maar antwoorden komen nooit aan. Wat moet ik controleren?">
    Snelle statuscontrole:

    ```bash
    openclaw status
    openclaw models status
    openclaw channels status
    openclaw logs --follow
    ```

    Veelvoorkomende oorzaken: modelauthenticatie is niet geladen op de **Gateway-host** (controleer `models status`), kanaalkoppeling/toelatingslijst blokkeert antwoorden (controleer de kanaalconfiguratie en logboeken), of WebChat/Dashboard is geopend zonder het juiste token. Controleer bij externe toegang of de tunnel-/Tailscale-verbinding actief is en of de WebSocket van de Gateway bereikbaar is.

    Docs: [Kanalen](/nl/channels), [Probleemoplossing](/nl/gateway/troubleshooting), [Externe toegang](/nl/gateway/remote).

  </Accordion>

  <Accordion title='"Verbinding met gateway verbroken: geen reden" - wat nu?'>
    Dit betekent meestal dat de UI de WebSocket-verbinding heeft verloren. Controleer: draait de Gateway (`openclaw gateway status`)? Werkt deze correct (`openclaw status`)? Heeft de UI het juiste token (`openclaw dashboard`)? Als de Gateway extern draait, is de tunnel-/Tailscale-verbinding actief?

    Volg daarna de logs:

    ```bash
    openclaw logs --follow
    ```

    Docs: [Dashboard](/nl/web/dashboard), [Externe toegang](/nl/gateway/remote), [Probleemoplossing](/nl/gateway/troubleshooting).

  </Accordion>

  <Accordion title="Telegram setMyCommands mislukt. Wat moet ik controleren?">
    ```bash
    openclaw channels status
    openclaw channels logs --channel telegram
    ```

    Zoek daarna de bijbehorende fout:

    - `BOT_COMMANDS_TOO_MUCH`: het Telegram-menu bevat te veel items. OpenClaw kort het menu al in tot de limiet van Telegram en probeert het opnieuw met minder opdrachten, maar sommige menu-items kunnen alsnog worden weggelaten. Verminder het aantal Plugin-/skill-/aangepaste opdrachten of schakel `channels.telegram.commands.native` uit als je het menu niet nodig hebt.
    - `TypeError: fetch failed`, `Network request for 'setMyCommands' failed!` of vergelijkbare netwerkfouten: controleer op een VPS of achter een proxy of uitgaand HTTPS-verkeer is toegestaan en DNS werkt voor `api.telegram.org`.

    Als de Gateway extern draait, controleer je de logs op de Gateway-host.

    Docs: [Telegram](/nl/channels/telegram), [Probleemoplossing voor kanalen](/nl/channels/troubleshooting).

  </Accordion>

  <Accordion title="De TUI toont geen uitvoer. Wat moet ik controleren?">
    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    Gebruik in de TUI `/status` om de huidige status te bekijken. Als je antwoorden in een chatkanaal verwacht, controleer dan of bezorging is ingeschakeld (`/deliver on`).

    Docs: [TUI](/nl/web/tui), [Slash-opdrachten](/nl/tools/slash-commands).

  </Accordion>

  <Accordion title="Hoe stop ik de Gateway volledig en start ik deze daarna opnieuw?">
    Als je de service hebt geïnstalleerd (launchd op macOS, systemd op Linux):

    ```bash
    openclaw gateway stop
    openclaw gateway start
    ```

    Stop op de voorgrond met Ctrl-C en voer daarna `openclaw gateway run` uit.

    Docs: [Runbook voor de Gateway-service](/nl/gateway).

  </Accordion>

  <Accordion title="Eenvoudig uitgelegd: openclaw gateway restart versus openclaw gateway">
    `openclaw gateway restart` herstart de **achtergrondservice** (launchd/systemd). `openclaw gateway` voert de gateway **op de voorgrond** uit voor deze terminalsessie. Gebruik de gateway-subopdrachten als je de service hebt geïnstalleerd; gebruik de gewone uitvoering op de voorgrond voor een eenmalige sessie.
  </Accordion>

  <Accordion title="Snelste manier om meer details te krijgen wanneer iets mislukt">
    Start de Gateway met `--verbose` voor meer details in de console en bekijk daarna het logbestand voor fouten met kanaalauthenticatie, modelroutering en RPC.
  </Accordion>
</AccordionGroup>

## Media en bijlagen

<AccordionGroup>
  <Accordion title="Mijn skill heeft een afbeelding/PDF gegenereerd, maar er is niets verzonden">
    Uitgaande bijlagen van de agent moeten gestructureerde mediavelden gebruiken, zoals `media`, `mediaUrl`, `path` of `filePath`. Zie [OpenClaw-assistent instellen](/nl/start/openclaw) en [Verzenden door agent](/nl/tools/agent-send).

    ```bash
    openclaw message send --target +15555550123 --message "Alsjeblieft" --media /path/to/file.png
    ```

    Controleer ook of het doelkanaal uitgaande media ondersteunt en niet door toelatingslijsten wordt geblokkeerd; of het bestand binnen de groottelimieten van de provider valt (afbeeldingen worden verkleind tot een maximale zijde van 2048px); `tools.fs.workspaceOnly=true` beperkt verzendingen via lokale paden tot bestanden in de werkruimte, tijdelijke/mediaopslag en door de sandbox gevalideerde bestanden; `tools.fs.workspaceOnly=false` (standaard) staat toe dat gestructureerde verzendingen van lokale media bestanden op de host gebruiken die de agent al kan lezen, voor media en veilige documenttypen (afbeeldingen, audio, video, PDF, Office-documenten en gevalideerde tekstdocumenten zoals Markdown/MD, TXT, JSON, YAML/YML). Dit is geen scanner voor geheimen: een voor de agent leesbaar `secret.txt` of `config.json` kan als bijlage worden toegevoegd wanneer de extensie en inhoudsvalidatie overeenkomen. Bewaar gevoelige bestanden buiten paden die de agent kan lezen, of behoud `tools.fs.workspaceOnly=true` voor strengere verzendingen via lokale paden.

    Zie [Afbeeldingen](/nl/nodes/images).

  </Accordion>
</AccordionGroup>

## Beveiliging en toegangsbeheer

<AccordionGroup>
  <Accordion title="Is het veilig om OpenClaw toegankelijk te maken voor inkomende privéberichten?">
    Behandel inkomende privéberichten als niet-vertrouwde invoer. De standaardinstellingen beperken het risico:

    - Het standaardgedrag op kanalen die privéberichten ondersteunen is **koppelen**: onbekende afzenders ontvangen een koppelingscode en hun bericht wordt niet verwerkt. Keur goed met `openclaw pairing approve --channel <channel> [--account <id>] <code>`. Er zijn maximaal **3 per kanaal** openstaande verzoeken toegestaan; controleer `openclaw pairing list --channel <channel> [--account <id>]` als er geen code is aangekomen.
    - Privéberichten openbaar toestaan vereist expliciete activering (`dmPolicy: "open"` en toelatingslijst `"*"`).

    Voer `openclaw doctor` uit om riskant beleid voor privéberichten zichtbaar te maken.

  </Accordion>

  <Accordion title="Is promptinjectie alleen een probleem voor openbare bots?">
    Nee. Promptinjectie gaat om **niet-vertrouwde inhoud**, niet alleen om wie de bot privéberichten kan sturen. Als je assistent externe inhoud leest (zoeken/ophalen op het web, browserpagina's, e-mails, documenten, bijlagen, geplakte logs), kan die inhoud instructies bevatten die proberen het model over te nemen, zelfs als jij de enige afzender bent.

    Het grootste risico ontstaat wanneer tools zijn ingeschakeld: het model kan worden misleid om context te lekken of namens jou tools aan te roepen. Beperk de impact:

    - gebruik een alleen-lezenagent of een agent zonder tools als 'lezer' om niet-vertrouwde inhoud samen te vatten
    - schakel `web_search` / `web_fetch` / `browser` uit voor agents met ingeschakelde tools
    - behandel gedecodeerde bestands-/documenttekst ook als niet-vertrouwd: zowel OpenResponses `input_file` als de extractie van mediabijlagen plaatsen geëxtraheerde tekst tussen expliciete grensmarkeringen voor externe inhoud in plaats van onbewerkte bestandstekst door te geven
    - gebruik een sandbox en strikte toelatingslijsten voor tools

    Details: [Beveiliging](/nl/gateway/security).

  </Accordion>

  <Accordion title="Is OpenClaw minder veilig omdat het TypeScript/Node gebruikt in plaats van Rust/WASM?">
    Taal en runtime zijn van belang, maar vormen niet het belangrijkste risico voor een persoonlijke agent. De praktische risico's zijn blootstelling van de gateway, wie de bot berichten kan sturen, promptinjectie, het bereik van tools, omgang met aanmeldgegevens, browsertoegang, uitvoeringstoegang en het vertrouwen in skills/plugins van derden.

    Rust en WASM kunnen voor sommige codecategorieën sterkere isolatie bieden, maar lossen promptinjectie, slechte toelatingslijsten, openbare blootstelling van de gateway, te ruime tools of een browserprofiel dat al bij gevoelige accounts is aangemeld niet op. Beschouw dit als de belangrijkste beheersmaatregelen: houd de Gateway privé of beveiligd met authenticatie, gebruik koppeling en toelatingslijsten voor privéberichten/groepen, weiger riskante tools voor niet-vertrouwde invoer of voer ze uit in een sandbox, installeer alleen vertrouwde plugins en skills en voer `openclaw security audit --deep` uit na configuratiewijzigingen.

    Details: [Beveiliging](/nl/gateway/security), [Sandboxing](/nl/gateway/sandboxing).

  </Accordion>

  <Accordion title="Ik heb berichten gezien over blootgestelde OpenClaw-instanties. Wat moet ik controleren?">
    ```bash
    openclaw security audit --deep
    openclaw gateway status
    ```

    Een veiligere basisconfiguratie: Gateway gebonden aan `loopback`, of alleen toegankelijk via geauthenticeerde privétoegang (tailnet, SSH-tunnel, token-/wachtwoordauthenticatie of een correct geconfigureerde vertrouwde proxy); privéberichten in de modus `pairing` of `allowlist`; groepen op een toelatingslijst en alleen geactiveerd bij vermelding, tenzij elk lid wordt vertrouwd; tools met een hoog risico (`exec`, `browser`, `gateway`, `cron`) geweigerd of strikt beperkt voor agents die niet-vertrouwde inhoud lezen; sandboxing ingeschakeld waar uitvoering van tools een kleinere impactzone vereist.

    Openbare bindingen zonder authenticatie, open privéberichten/groepen met tools en blootgestelde browserbediening zijn de bevindingen die je als eerste moet oplossen. Details: [openclaw security audit](/nl/gateway/security#openclaw-security-audit).

  </Accordion>

  <Accordion title="Zijn ClawHub-skills en plugins van derden veilig om te installeren?">
    Behandel skills en plugins van derden als code die je bewust vertrouwt. Skillpagina's van ClawHub tonen vóór installatie de scanstatus, maar scans vormen geen volledige beveiligingsgrens. OpenClaw voert tijdens de installatie of update van plugins/skills geen ingebouwde lokale blokkering van gevaarlijke code uit; gebruik door de operator beheerde `security.installPolicy` voor lokale beslissingen over toestaan/blokkeren.

    Veiliger patroon: geef de voorkeur aan vertrouwde auteurs en vastgezette versies, lees de skill/plugin voordat je deze inschakelt, houd toelatingslijsten voor plugins/skills beperkt, voer workflows met niet-vertrouwde invoer uit in een sandbox met minimale tools en geef code van derden geen brede toegang tot het bestandssysteem, uitvoering, de browser of geheimen.

    Details: [Skills](/nl/tools/skills), [Plugins](/nl/tools/plugin), [Beveiliging](/nl/gateway/security).

  </Accordion>

  <Accordion title="Moet mijn bot een eigen e-mailadres, GitHub-account of telefoonnummer hebben?">
    Ja, voor de meeste configuraties. Door de bot met afzonderlijke accounts en telefoonnummers te isoleren, beperk je de impact als er iets misgaat en kun je eenvoudiger aanmeldgegevens vervangen of toegang intrekken zonder je persoonlijke accounts te beïnvloeden.

    Begin klein: geef alleen toegang tot de tools en accounts die je daadwerkelijk nodig hebt en breid dit later indien nodig uit.

    Docs: [Beveiliging](/nl/gateway/security), [Koppelen](/nl/channels/pairing).

  </Accordion>

  <Accordion title="Kan ik de bot autonomie geven over mijn tekstberichten en is dat veilig?">
    We raden volledige autonomie over je persoonlijke berichten **niet** aan. Het veiligste patroon: houd privéberichten in de **koppelingsmodus** of gebruik een strikte toelatingslijst, gebruik een **afzonderlijk nummer of account** als de bot namens jou berichten moet sturen en laat de bot concepten opstellen die jij **vóór verzending goedkeurt**.

    Experimenteer hiervoor met een speciaal, geïsoleerd account. Zie [Beveiliging](/nl/gateway/security).

  </Accordion>

  <Accordion title="Kan ik goedkopere modellen gebruiken voor taken van een persoonlijke assistent?">
    Ja, **als** de agent alleen chat en de invoer wordt vertrouwd. Kleinere modelcategorieën zijn gevoeliger voor het overnemen van instructies, dus vermijd ze voor agents met ingeschakelde tools of bij het lezen van niet-vertrouwde inhoud. Als je een kleiner model moet gebruiken, beperk dan de tools en voer het uit in een sandbox. Zie [Beveiliging](/nl/gateway/security).
  </Accordion>

  <Accordion title="Ik heb /start uitgevoerd in Telegram, maar kreeg geen koppelingscode">
    Koppelingscodes worden **alleen** verzonden wanneer een onbekende afzender de bot een bericht stuurt en `dmPolicy: "pairing"` is ingeschakeld; alleen `/start` genereert geen code.

    Controleer openstaande verzoeken:

    ```bash
    openclaw pairing list telegram
    ```

    Voor directe toegang voeg je je afzender-id toe aan de toelatingslijst of stel je `dmPolicy: "open"` in voor dat account.

  </Accordion>

  <Accordion title="WhatsApp: stuurt de bot berichten naar mijn contacten? Hoe werkt koppelen?">
    Nee. Het standaardbeleid voor privéberichten van WhatsApp is **koppelen**. Onbekende afzenders krijgen alleen een koppelingscode; hun bericht wordt **niet verwerkt**. OpenClaw antwoordt alleen op chats die het ontvangt of op expliciete verzendingen die je activeert.

    ```bash
    openclaw pairing approve whatsapp <code>
    openclaw pairing list whatsapp
    ```

    De vraag om een telefoonnummer in de wizard stelt je **toelatingslijst/eigenaar** in, zodat je eigen privéberichten zijn toegestaan; dit nummer wordt niet gebruikt voor automatisch verzenden. Gebruik voor je persoonlijke WhatsApp-nummer datzelfde nummer en schakel `channels.whatsapp.selfChatMode` in.

  </Accordion>
</AccordionGroup>

## Chatopdrachten, taken afbreken en "het stopt niet"

<AccordionGroup>
  <Accordion title="Hoe voorkom ik dat interne systeemberichten in de chat verschijnen?">
    De meeste interne/toolberichten verschijnen alleen wanneer **uitgebreid**, **tracering** of **redenering** voor die sessie is ingeschakeld.

    Los dit op in de chat waarin je ze ziet:

    ```text
    /verbose off
    /trace off
    /reasoning off
    ```

    Nog steeds te veel berichten: controleer de sessie-instellingen in de Control UI en stel uitgebreid in op **overnemen**; controleer of je geen botprofiel gebruikt met `verboseDefault: "on"` in de configuratie.

    Docs: [Denken en uitgebreide uitvoer](/nl/tools/thinking), [Beveiliging](/nl/gateway/security/index#reasoning-and-verbose-output-in-groups).

  </Accordion>

  <Accordion title="Hoe stop/annuleer ik een actieve taak?">
    Stuur een van deze opties **als een op zichzelf staand bericht** (zonder slash) om afbreken te activeren: `stop`, `stop action`, `stop current action`, `stop run`, `stop current run`, `stop agent`, `stop the agent`, `stop openclaw`, `openclaw stop`, `stop don't do anything`, `stop do not do anything`, `stop doing anything`, `do not do that`, `please stop`, `stop please`, `abort`, `esc`, `exit`, `interrupt`, `halt`. Veelgebruikte niet-Engelse triggers (Frans, Duits, Spaans, Chinees, Japans, Hindi, Arabisch, Russisch) werken ook.

    Vraag de agent voor achtergrondprocessen die door de exec-tool zijn gestart om het volgende uit te voeren:

    ```text
    process action:kill sessionId:XXX
    ```

    De meeste slash-opdrachten moeten worden verzonden als een **op zichzelf staand** bericht dat begint met `/`, maar enkele snelkoppelingen (zoals `/status`) werken voor afzenders op de toelatingslijst ook binnen een bericht. Zie [Slash-opdrachten](/nl/tools/slash-commands).

  </Accordion>

  <Accordion title='Hoe stuur ik een Discord-bericht vanuit Telegram? ("Berichten tussen contexten geweigerd")'>
    OpenClaw blokkeert standaard berichten **tussen providers**. Als een toolaanroep aan Telegram is gekoppeld, wordt er geen bericht naar Discord verzonden tenzij je dit expliciet toestaat. Dit wordt onmiddellijk van kracht; de Gateway hoeft niet opnieuw te worden gestart:

    ```json5
    {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[van {channel}] " },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title='Waarom lijkt het alsof de bot snel achter elkaar verzonden berichten "negeert"?'>
    Prompts die tijdens een uitvoering worden verzonden, worden standaard naar de actieve uitvoering geleid. Gebruik `/queue` om het gedrag van de actieve uitvoering te kiezen:

    - `steer` (standaard) - stuur de actieve uitvoering bij bij de volgende modelgrens.
    - `followup` - plaats berichten in de wachtrij en voer ze een voor een uit nadat de huidige uitvoering is beëindigd.
    - `collect` - plaats compatibele berichten in de wachtrij en antwoord één keer nadat de huidige uitvoering is beëindigd.
    - `interrupt` - breek de huidige uitvoering af en begin opnieuw.

    Voeg opties toe aan wachtrijmodi, zoals `debounce:0.5s cap:25 drop:summarize`. Zie [Opdrachtwachtrij](/nl/concepts/queue) en [Bijsturingswachtrij](/nl/concepts/queue-steering).

  </Accordion>
</AccordionGroup>

## Diversen

<AccordionGroup>
  <Accordion title='Wat is het standaardmodel voor Anthropic met een API-sleutel?'>
    Referenties en modelselectie staan los van elkaar. Het instellen van `ANTHROPIC_API_KEY` (of het opslaan van een Anthropic-API-sleutel in authenticatieprofielen) maakt authenticatie mogelijk, maar het daadwerkelijke standaardmodel is wat je configureert in `agents.defaults.model.primary` (bijvoorbeeld `anthropic/claude-sonnet-4-6` of `anthropic/claude-opus-4-6`). `No credentials found for profile "anthropic:default"` betekent dat de Gateway geen Anthropic-referenties kon vinden in de verwachte `auth-profiles.json` voor de actieve agent.
  </Accordion>
</AccordionGroup>

---

Kom je er nog steeds niet uit? Vraag het in [Discord](https://discord.com/invite/clawd) of open een [GitHub-discussie](https://github.com/openclaw/openclaw/discussions).

## Gerelateerd

- [Veelgestelde vragen over de eerste uitvoering](/nl/help/faq-first-run) - installatie, onboarding, authenticatie, abonnementen, vroege fouten
- [Veelgestelde vragen over modellen](/nl/help/faq-models) - modelselectie, failover, authenticatieprofielen
- [Problemen oplossen](/nl/help/troubleshooting) - symptoomgerichte triage
