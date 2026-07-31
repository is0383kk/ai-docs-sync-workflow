---
read_when:
    - Doctor-migraties toevoegen of wijzigen
    - Invoering van incompatibele configuratiewijzigingen
sidebarTitle: Doctor
summary: 'Doctor-opdracht: statuscontroles, configuratiemigraties en herstelstappen'
title: Diagnosehulpmiddel
x-i18n:
    generated_at: "2026-07-27T06:14:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` is het reparatie- en migratiehulpprogramma voor OpenClaw. Het herstelt verouderde configuratie/status, controleert de gezondheid en biedt uitvoerbare reparatiestappen.

## Snel aan de slag

```bash
openclaw doctor
```

### Headless- en automatiseringsmodi

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    Accepteer standaardwaarden zonder prompts (inclusief stappen voor het repareren van herstarts/services/sandboxen indien van toepassing).

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    Pas aanbevolen reparaties toe zonder prompts (`--repair` is een alias).

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    Voer gestructureerde gezondheidscontroles uit voor CI of preflight-automatisering. Alleen-lezen: geen
    prompts, reparaties, migraties, herstarts of statusbewerkingen.

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    Pas ook ingrijpende reparaties toe (overschrijft aangepaste supervisorconfiguraties).

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    Voer uit zonder prompts en pas alleen veilige migraties toe (configuratienormalisatie +
    verplaatsingen van status op schijf). Slaat herstart-/service-/sandboxacties over waarvoor menselijke
    bevestiging nodig is. Verouderde statusmigraties worden bij detectie nog steeds automatisch uitgevoerd.

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    Scan systeemservices op extra Gateway-installaties (launchd/systemd/schtasks).

  </Tab>
</Tabs>

Open eerst het configuratiebestand om wijzigingen vóór het schrijven te beoordelen:

```bash
cat ~/.openclaw/openclaw.json
```

## Alleen-lezen-lintmodus

`openclaw doctor --lint` is de automatiseringsvriendelijke tegenhanger van
`openclaw doctor --fix`. Ze delen hetzelfde Doctor-regelregister, maar
selecteren regels niet op dezelfde manier en voeren er niet op dezelfde manier acties voor uit:

| Modus                     | Prompts   | Schrijft configuratie/status | Uitvoer                       | Gebruik voor                         |
| ------------------------ | --------- | ----------------------- | ---------------------- | ------------------------------- |
| `openclaw doctor`        | ja        | nee                     | begrijpelijk gezondheidsrapport | een persoon die de status controleert |
| `openclaw doctor --fix`  | soms      | ja, met reparatiebeleid | begrijpelijk reparatielogboek | goedgekeurde reparaties toepassen |
| `openclaw doctor --lint` | nee       | nee                     | gestructureerde bevindingen | CI-, preflight- en reviewpoorten |

Standaard voert `doctor --lint` het brede, veilige automatiseringsprofiel uit: controles die
statisch en lokaal zijn en nuttig zijn in CI- of preflight-uitvoer. Opt-incontroles die
adviserend, omgevingsgevoelig of afhankelijk van live-services zijn, inventarisaties van accounts/werkruimten
uitvoeren of historische opschoning betreffen, worden overgeslagen. Gebruik `doctor --lint --all` als je de
volledige geregistreerde lintaudit wilt, inclusief die opt-incontroles, of `--only <id>` voor
een gerichte controle.

`doctor --fix` gebruikt het standaardprofiel voor lint niet en accepteert
`--all` niet. Het voert het geordende reparatiepad van Doctor uit: moderne gezondheidscontroles kunnen
een optionele `repair()`-implementatie bieden, terwijl oudere gebieden nog steeds hun verouderde
Doctor-reparatieflow gebruiken. Sommige lintbevindingen zijn bewust uitsluitend diagnostisch, dus het
verschijnen van een controle in `--lint --all` betekent niet dat `--fix` dat gebied zal wijzigen.
Het contract scheidt `detect()` (rapporteert bevindingen) van `repair()` (rapporteert
wijzigingen/diffs/neveneffecten), zodat er ruimte blijft voor een toekomstige
`doctor --fix --dry-run` zonder lintcontroles in wijzigingsplanners te veranderen.

Sommige ingebouwde controles zijn intern standaard uitgeschakeld, zodat ze beschikbaar blijven voor
`--all`, `--only` en Doctor-reparatieflows zonder deel te worden van het standaard
`doctor --lint`-automatiseringsprofiel. De ernst wordt nog steeds per bevinding
weergegeven (`info`, `warning` of `error`); standaardselectie is geen
ernstniveau.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

Velden in JSON-uitvoer:

- `ok`: of een bevinding aan de geselecteerde ernstgrens voldeed
- `checksRun` / `checksSkipped`: aantallen (overgeslagen vanwege profiel, `--only` of `--skip`)
- `findings`: gestructureerde diagnostiek met `checkId`, `severity`, `message` en optioneel `path`, `line`, `column`, `ocPath`, `source`, `target`, `requirement`, `fixHint`

Afsluitcodes:

| Code | Betekenis                                                |
| ---- | -------------------------------------------------------- |
| `0`  | geen bevindingen op of boven de geselecteerde grens      |
| `1`  | een of meer bevindingen voldeden aan de geselecteerde grens |
| `2`  | opdracht-/runtimefout voordat bevindingen konden worden uitgevoerd |

Vlaggen:

- `--severity-min info|warning|error` (standaard `warning`): bepaalt zowel wat wordt afgedrukt als wat een niet-nul-afsluitcode veroorzaakt.
- `--all`: voert elke geregistreerde lintcontrole uit, inclusief opt-incontroles die van de standaard automatiseringsset zijn uitgesloten.
- `--only <id>` (herhaalbaar): voer alleen de genoemde controle-id('s) uit; een onbekende id wordt als foutbevinding gerapporteerd.
- `--skip <id>` (herhaalbaar): sluit een controle uit terwijl de rest van de uitvoering actief blijft.
- `--json`, `--severity-min`, `--all`, `--only` en `--skip` vereisen `--lint`; gewone uitvoeringen van `openclaw doctor` en `--fix` wijzen deze af.

## Wat het doet (samenvatting)

<AccordionGroup>
  <Accordion title="Gezondheid, UI en updates">
    - Optionele preflight-update voor git-installaties (alleen interactief).
    - Versheidscontrole van het UI-protocol (bouwt de Control UI opnieuw wanneer het protocolschema nieuwer is).
    - Gezondheidscontrole + prompt voor herstart.
    - Alleen opmerkingen over problematische Skills en plugins; een gezonde inventaris blijft in `openclaw skills check` en `openclaw plugins list`.

  </Accordion>
  <Accordion title="Configuratie en migraties">
    - Configuratienormalisatie voor verouderde waardevormen.
    - Migratie van Talk-configuratie van verouderde platte `talk.*`-velden naar `talk.provider` + `talk.providers.<provider>`.
    - Browsermigratiecontroles voor verouderde Chrome-extensieconfiguraties en gereedheid van Chrome MCP.
    - Waarschuwingen voor OpenCode-provideroverschrijvingen (`models.providers.opencode` / `opencode-zen` / `opencode-go`).
    - Migratie van verouderde OpenAI Codex-provider/profielen (`openai-codex` → `openai`) en overschaduwingswaarschuwingen voor verouderde `models.providers.openai-codex`.
    - Controle van OAuth TLS-vereisten voor OpenAI Codex OAuth-profielen.
    - Waarschuwingen voor plugin-/tooltoelatingslijsten wanneer `plugins.allow` restrictief is, maar het toolbeleid nog steeds om jokertekens of tools van plugins vraagt.
    - Migratie van verouderde status op schijf (sessies/agentmap/WhatsApp-authenticatie).
    - Migratie van verouderde contractsleutels in pluginmanifesten (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
    - Migratie van verouderde Cron-opslag (`jobId`, `schedule.cron`, leverings-/payloadvelden op het hoogste niveau, payload `provider`, `notify: true` Webhook-terugvaltaken).
    - Reparatie van de runtimepin voor Codex CLI (`agentRuntime.id: "codex-cli"` → `"codex"`) in `agents.defaults`, `agents.entries.*` en `models.providers.*` (inclusief vermeldingen per model).
    - Opschoning van verouderde pluginconfiguratie wanneer plugins zijn ingeschakeld; bij `plugins.enabled=false` blijven verouderde pluginverwijzingen behouden als inactieve inperkingsconfiguratie.

  </Accordion>
  <Accordion title="Status en integriteit">
    - Inspectie van sessievergrendelingsbestanden en opschoning van verouderde vergrendelingen.
    - Reparatie van sessietranscripten voor dubbele prompt-herschrijvingstakken die door getroffen builds van 2026.4.24 zijn aangemaakt.
    - Detectie van tombstones voor herstart-herstel van vastgelopen hoofdsessies en subagents. Doctor rapporteert de geblokkeerde sessies en repareert alleen verouderde afgebroken-vlaggen die conflicteren met een bestaande tombstone; automatisch herstel wordt niet opnieuw ingeschakeld.
    - Controles van statusintegriteit en machtigingen (sessies, transcripten, statusmap).
    - Controles van configuratiebestandsmachtigingen (chmod 600) bij lokale uitvoering.
    - Gezondheid van modelauthenticatie: controleert de vervaldatum van OAuth, kan bijna verlopen tokens vernieuwen en rapporteert cooldown-/uitgeschakelde statussen van authenticatieprofielen.

  </Accordion>
  <Accordion title="Gateway, services en supervisors">
    - Reparatie van sandboximages wanneer sandboxing is ingeschakeld.
    - Migratie van verouderde services en detectie van extra gateways.
    - Migratie van verouderde status van het Matrix-kanaal (in de modus `--fix` / `--repair`).
    - Runtimecontroles van de Gateway (service geïnstalleerd maar niet actief; launchd-label in cache).
    - Waarschuwingen over kanaalstatus (opgevraagd bij de actieve Gateway).
    - Kanaalspecifieke machtigingscontroles staan onder `openclaw channels capabilities`; zo worden machtigingen voor Discord-spraakkanalen gecontroleerd met `openclaw channels capabilities --channel discord --target channel:<channel-id>`.
    - Responsiviteitscontroles voor WhatsApp bij een verslechterde gezondheid van de Gateway-eventloop terwijl lokale TUI-clients nog actief zijn; `--fix` stopt alleen geverifieerde lokale TUI-clients.
    - Reparatie van Codex-routes voor verouderde `openai-codex/*`-modelverwijzingen in primaire modellen, terugvalmodellen, modellen voor het genereren van afbeeldingen/video's, overschrijvingen voor Heartbeat/subagents/Compaction, hooks, overschrijvingen van kanaalmodellen en sessieroutepins; `--fix` herschrijft ze naar `openai/*`, migreert `openai-codex:*`-authenticatieprofielen/-volgorde naar `openai:*`, verwijdert verouderde runtimepins voor sessies/volledige agents en laat de gerepareerde effectieve route bepalen of Codex compatibel is.
    - Audit van supervisorconfiguraties (launchd/systemd/schtasks) met optionele reparatie.
    - Opschoning van ingesloten proxyomgevingen voor Gateway-services die tijdens installatie of update shellwaarden voor `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` hebben vastgelegd.
    - Runtimecontroles van de Gateway (niet-ondersteunde verouderde Bun-services, paden van versiebeheerders).
    - Diagnostiek van Gateway-poortconflicten (standaard `18789`).

  </Accordion>
  <Accordion title="Authenticatie, beveiliging en koppeling">
    - Beveiligingswaarschuwingen voor open DM-beleid.
    - Gateway-authenticatiecontroles voor lokale tokenmodus (biedt het genereren van tokens aan wanneer geen tokenbron bestaat; overschrijft geen SecretRef-configuraties voor tokens).
    - Detectie van problemen met apparaatkoppeling (openstaande aanvragen voor eerste koppeling, openstaande upgrades van rollen/bereiken, afwijkingen in verouderde lokale apparaat-tokencaches en authenticatieafwijkingen in gekoppelde records).

  </Accordion>
  <Accordion title="Werkruimte en shell">
    - Controle van systemd-linger op Linux.
    - Controle van de bestandsgrootte van werkruimtebootstrapbestanden (waarschuwingen voor afkapping/bijna bereikte limiet voor contextbestanden).
    - Gereedheidscontrole van Skills voor de standaardagent; rapporteert toegestane Skills met ontbrekende binaries, omgevingsvariabelen, configuratie of OS-vereisten, en `--fix` kan niet-beschikbare Skills uitschakelen in `skills.entries`.
    - Statuscontrole en automatische installatie/upgrade van shellaanvulling.
    - Gereedheidscontrole van de embeddingprovider voor geheugenzoekopdrachten (lokaal model, externe API-sleutel of QMD-binary).
    - Controles van broninstallaties (niet-overeenkomende pnpm-werkruimte, ontbrekende UI-assets, ontbrekende tsx-binary).
    - Schrijft bijgewerkte configuratie + wizardmetadata.

  </Accordion>
</AccordionGroup>

## Aanvulling en reset van de Dreams-UI

  De scène Dreams in de Control UI bevat de acties **Backfill**, **Reset** en **Clear Grounded** voor de geaarde Dreaming-workflow. Deze gebruiken RPC-methoden in de stijl van Gateway doctor, maar maken **geen** deel uit van de CLI-reparatie/-migratie van `openclaw doctor`.

  | Actie          | Wat deze doet                                                                                                                                                         |
  | -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Backfill       | Scant historische `memory/YYYY-MM-DD.md`-bestanden in de actieve werkruimte, voert de geaarde REM-dagboekpassage uit en schrijft omkeerbare backfill-vermeldingen naar `DREAMS.md`. |
  | Reset          | Verwijdert alleen de gemarkeerde backfill-dagboekvermeldingen uit `DREAMS.md`.                                                                                 |
  | Clear Grounded | Verwijdert alleen klaargezette, uitsluitend geaarde kortetermijnvermeldingen uit historische herhaling die nog geen live herinnering of dagelijkse ondersteuning hebben opgebouwd. |

  Geen van deze acties bewerkt `MEMORY.md`, voert volledige doctor-migraties uit of zet zelfstandig geaarde kandidaten klaar in de live opslag voor kortetermijnpromotie. Gebruik in plaats daarvan de CLI-flow om geaarde historische herhaling naar het normale diepe promotietraject te sturen:

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  Hiermee worden geaarde duurzame kandidaten klaargezet in de kortetermijnopslag voor Dreaming, terwijl `DREAMS.md` het beoordelingsoppervlak blijft.

  ## Gedetailleerd gedrag en onderbouwing

  <AccordionGroup>
  <Accordion title="0. Optionele update (git-installaties)">
    Als dit een git-checkout is en doctor interactief wordt uitgevoerd, biedt deze aan om bij te werken (ophalen/rebasen/bouwen) voordat doctor wordt uitgevoerd.
  </Accordion>
  <Accordion title="1. Configuratienormalisatie">
    Doctor normaliseert verouderde waardevormen naar het huidige schema. De huidige spraakconfiguratie van Talk is `talk.provider` + `talk.providers.<provider>`, met de realtime stemconfiguratie onder `talk.realtime.*`. Doctor herschrijft oude vormen van `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` naar de provider-map en herschrijft verouderde realtime selectors op het hoogste niveau (`talk.mode`, `talk.transport`, `talk.brain`, `talk.model`, `talk.voice`) naar `talk.realtime`.

    Doctor waarschuwt ook wanneer `plugins.allow` niet leeg is en het toolbeleid jokertekens of toolvermeldingen van plugins gebruikt. `tools.allow: ["*"]` komt alleen overeen met tools van plugins die daadwerkelijk worden geladen; het omzeilt de exclusieve toestemmingslijst voor plugins niet.

  </Accordion>
  <Accordion title="2. Migraties van verouderde configuratiesleutels">
    Wanneer de configuratie een verouderde sleutel met een actieve migratie bevat, weigeren andere opdrachten te worden uitgevoerd en vragen ze je om `openclaw doctor` uit te voeren. Doctor legt uit welke verouderde sleutels zijn gevonden, toont de toegepaste migratie en herschrijft `~/.openclaw/openclaw.json` met het bijgewerkte schema. Het opstarten van de Gateway weigert verouderde configuratie-indelingen en vraagt je om `openclaw doctor --fix` uit te voeren; bij het opstarten wordt `openclaw.json` niet herschreven. Migraties van de Cron-taakopslag worden ook afgehandeld door `openclaw doctor --fix`.

    <Note>
      Doctor biedt automatische migraties slechts ongeveer twee maanden nadat een
      sleutel buiten gebruik is gesteld. Oudere verouderde sleutels (bijvoorbeeld de oorspronkelijke
      `routing.queue`, `routing.bindings`, `routing.agents`/`defaultAgentId`,
      `routing.transcribeAudio`, `agent.*` op het hoogste niveau of `identity`
      op het hoogste niveau uit de configuratievorm van vóór multi-agent) hebben geen migratiepad
      meer; configuraties die deze gebruiken, mislukken nu bij de validatie in plaats van te worden
      herschreven. Corrigeer die sleutels handmatig aan de hand van de huidige configuratiereferentie
      voordat doctor verder kan gaan.
    </Note>

    Actieve migraties:

    | Verouderde sleutel                                                                                    | Huidige sleutel                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | verwijderd (WebChat is buiten gebruik gesteld)                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours` (en per account)      | `...threadBindings.idleHours`                                               |
    | verouderde `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey`        | `talk.provider` + `talk.providers.<provider>`                               |
    | verouderde realtime Talk-selectors op het hoogste niveau (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | `tts` op het hoogste niveau                                                              |
    | `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | `messages.responsePrefix` met expliciete kanaalblokken                                           | gekopieerd naar `responsePrefix` van het geconfigureerde kanaal/account; globale terugval behouden voor impliciete/aangepaste kanalen |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`, hook-installaties, Cron-opslag, gebundelde detectie, globaal pad voor TTS-voorkeuren            | gedeelde SQLite-status                                                       |
    | TTS-sprekervelden `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>` (alle kanalen behalve Discord)                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>` (alle kanalen, inclusief Discord)                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"` (bij het starten van de Gateway worden ook providers overgeslagen waarvan `api` een toekomstige/onbekende enumwaarde is, in plaats van gesloten te falen) |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | verwijderd (verouderde relayinstelling voor de Chrome-extensie)                             |
    | `mcp.servers.*.type` (CLI-eigen aliassen)                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | omgekeerde `mcp.servers.*.enabled`                                              |
    | MCP-time-outaliassen `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | MCP-servervelden in snake_case                                                                     | MCP-servervelden in camelCase                                                   |
    | `tools.media.image/audio/video.models`                                                           | met capaciteiten gelabelde `tools.media.models`                                        |
    | `tools.media.asyncCompletion`                                                                    | verwijderd                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | `deepgram`-opties voor mediamodellen                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`, realtime `voice` van Discord                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | jokertekenbewuste `browser.ssrfPolicy.allowedHostnames`                          |
    | `enableNoVnc` van de sandboxbrowser                                                                    | `noVncEnabled`                                                                |
    | hoofd-`media`                                                                                     | `attachments`                                                                |
    | zichtbaarheidsblokken voor `heartbeat` van kanaal/account                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | hoofd-`audit`                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | standaardwaarden voor generatiemodellen                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | buiten gebruik gestelde afstelopties voor de uiteindelijke lay-out                                                               | ingebouwd standaardgedrag                                                     |
    | `channels.whatsapp.messagePrefix` en verouderde `messages.messagePrefix`                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | globale `messages.ackReaction` en `ackReactionScope` waar vertaalbaar        |
    | `cron.failureDestination`                                                                        | bestemmingsvelden op `cron.failureAlert`                                     |
    | `gateway.controlUi.chatMessageMaxWidth`, uitsluitend voor presentatie bedoelde `ui.prefs`-sleutels                       | verwijderd (tekstgrootte, chatbreedte en live activiteit in de zijbalk zijn lokaal voor de browser) |
    | `agents.list`                                                                                    | `agents.entries` met sleutels                                                        |
    | `defaultModel` op het hoogste niveau                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | `tui` op het hoogste niveau                                                                                  | verwijderd (de TUI-voettekst gebruikt de compacte standaardwaarde)                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | verwijderd (de Codex-appserver houdt Codex-eigen werkruimtetools altijd native) |
    | `commands.modelsWrite`                                                                           | verwijderd (`/models add` is verouderd)                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | verwijderd (exacte `NO_REPLY` wordt niet meer herschreven naar zichtbare terugvaltekst)  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | verwijderd (OpenClaw beheert de gegenereerde systeemprompt)                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | verwijderd (gebruik `models.providers.<id>.timeoutSeconds` voor time-outs van trage modellen/providers, onder de bovengrens voor de agent-/uitvoeringstime-out gehouden) |
    | bovenste niveau `memorySearch`, `agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path` (elk niveau)                                                            | verwijderd (geheugenindexen bevinden zich in de database van elke agent)                       |
    | bovenste niveau `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | beleids-id's voor `plugins.openai-codex`                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`, `session.parentForkMaxTokens`                                 | verwijderd (verouderd)                                                        |
    | Instellingen voor runtime- en kanaalafstemming die in 2026.7 buiten gebruik zijn gesteld                                               | verwijderd (ingebouwde productiestandaardwaarden zijn van toepassing)                               |

    <Note>
      De bovenstaande `plugins.entries.voice-call.config.*`-rijen worden bij elke configuratielading genormaliseerd door
      de Voice Call-plugin zelf, niet door `openclaw
      doctor`. De plugin registreert ook een opstartwaarschuwing die verwijst naar `openclaw
      doctor --fix`, maar doctor herschrijft momenteel
      `openclaw.json` niet voor deze sleutels; de eigen normalisatie van de plugin
      past de wijziging tijdens runtime toe.
    </Note>

    Richtlijnen voor het standaardaccount bij kanalen met meerdere accounts:

    - Als twee of meer `channels.<channel>.accounts`-vermeldingen zijn geconfigureerd zonder `channels.<channel>.defaultAccount` of `accounts.default`, waarschuwt doctor dat de fallback-routering een onverwacht account kan kiezen.
    - Als `channels.<channel>.defaultAccount` is ingesteld op een onbekende account-ID, waarschuwt doctor en toont het de geconfigureerde account-ID's.

  </Accordion>
  <Accordion title="2b. Overschrijvingen voor de OpenCode-provider">
    Als je `models.providers.opencode`, `opencode-zen` of `opencode-go` handmatig hebt toegevoegd, overschrijft dit de ingebouwde OpenCode-catalogus uit `openclaw/plugin-sdk/llm`. Daardoor kunnen modellen gedwongen worden de verkeerde API te gebruiken of kunnen kosten op nul worden gezet. Doctor waarschuwt zodat je de overschrijving kunt verwijderen en de API-routering en kosten per model kunt herstellen.
  </Accordion>
  <Accordion title="2c. Browsermigratie en gereedheid van Chrome MCP">
    Als je browserconfiguratie nog naar het verwijderde Chrome-extensiepad verwijst, normaliseert doctor dit naar het huidige hostlokale Chrome MCP-koppelingsmodel (`browser.profiles.*.driver: "extension"` → `"existing-session"`; `browser.relayBindHost` verwijderd).

    Doctor controleert ook het hostlokale Chrome MCP-pad wanneer je `defaultProfile: "user"` of een geconfigureerd `existing-session`-profiel gebruikt:

    - controleert voor standaardprofielen met automatische verbinding of Google Chrome op dezelfde host is geïnstalleerd
    - controleert de gedetecteerde Chrome-versie en waarschuwt wanneer deze lager is dan Chrome 144
    - herinnert je eraan externe foutopsporing in te schakelen op de inspectiepagina van de browser (bijvoorbeeld `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging` of `edge://inspect/#remote-debugging`)

    Doctor kan de Chrome-instelling niet voor je inschakelen. Voor hostlokale Chrome MCP is nog steeds een lokaal uitgevoerde Chromium-gebaseerde browser 144+ op de Gateway-/Node-host vereist, met externe foutopsporing ingeschakeld en de eerste toestemmingsprompt voor koppeling goedgekeurd in de browser.

    De gereedheidscontrole hier dekt alleen de lokale koppelingsvereisten. Existing-session behoudt de huidige routelimieten van Chrome MCP; geavanceerde routes zoals `responsebody`, PDF-export, onderschepping van downloads en batchacties vereisen nog steeds een beheerde browser of een onbewerkt CDP-profiel. Deze controle is niet van toepassing op Docker-, sandbox-, externe-browser- of andere headless-flows, die onbewerkt CDP blijven gebruiken.

  </Accordion>
  <Accordion title="2d. TLS-vereisten voor OAuth">
    Wanneer een OpenAI Codex OAuth-profiel is geconfigureerd, test doctor het OpenAI-autorisatie-eindpunt om te verifiëren dat de lokale TLS-stack van Node/OpenSSL de certificaatketen kan valideren. Als de test mislukt met een certificaatfout (bijvoorbeeld `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, een verlopen certificaat of een zelfondertekend certificaat), toont doctor platformspecifieke richtlijnen voor de oplossing. Op macOS met een Homebrew-versie van Node is de oplossing meestal `brew postinstall ca-certificates`. Met `--deep` wordt de test uitgevoerd, zelfs als de Gateway gezond is.
  </Accordion>
  <Accordion title="2e. Overschrijvingen voor de Codex OAuth-provider">
    Als je eerder verouderde OpenAI-transportinstellingen hebt toegevoegd onder `models.providers.openai-codex`, kunnen deze het ingebouwde providerpad voor Codex OAuth overschaduwen. Doctor waarschuwt wanneer het die oude transportinstellingen naast Codex OAuth aantreft, zodat je de verouderde transportoverschrijving kunt verwijderen of herschrijven en het huidige routeringsgedrag kunt herstellen. Aangepaste proxy's en overschrijvingen met alleen headers blijven ondersteund en activeren deze waarschuwing niet, maar deze zelf gedefinieerde aanvraagroutes komen niet in aanmerking voor impliciete Codex-selectie.
  </Accordion>
  <Accordion title="2f. Herstel van Codex-routes">
    Doctor controleert op verouderde `openai-codex/*`-modelreferenties. Systeemeigen routering via de Codex-harness gebruikt canonieke `openai/*`-modelreferenties, maar alleen het voorvoegsel selecteert Codex nooit. Als het runtimebeleid niet is ingesteld of `auto` is, komt alleen een exacte officiële HTTPS-route voor Platform Responses of ChatGPT Responses zonder zelf gedefinieerde aanvraagoverschrijving in aanmerking. Zie [Impliciete OpenAI-agentruntime](/nl/providers/openai#implicit-agent-runtime).

    In de modus `--fix` / `--repair` herschrijft doctor de betrokken referenties voor de standaardagent en afzonderlijke agents, waaronder primaire modellen, fallbacks, modellen voor beeld-/videogeneratie, overschrijvingen voor Heartbeat/subagents/Compaction, hooks, modeloverschrijvingen voor kanalen en verouderde permanente sessieroutestatus:

    - `openai-codex/gpt-*` wordt `openai/gpt-*`.
    - De Codex-intentie wordt verplaatst naar provider-/modelgebonden `agentRuntime.id: "codex"`-vermeldingen voor herstelde modelreferenties van agents.
    - Verouderde runtimeconfiguratie voor de volledige agent en permanente runtimepinnen voor sessies worden verwijderd omdat runtimeselectie provider-/modelgebonden is.
    - Bestaand runtimebeleid voor providers/modellen blijft behouden, tenzij de herstelde verouderde modelreferentie Codex-routering nodig heeft om het oude authenticatiepad te behouden.
    - Bestaande lijsten met modelfallbacks blijven behouden en hun verouderde vermeldingen worden herschreven; gekopieerde instellingen per model worden van de verouderde sleutel naar de canonieke sleutel `openai/*` verplaatst.
    - Permanente sessiegegevens voor `modelProvider`/`providerOverride`, `model`/`modelOverride`, fallbackmeldingen en authenticatieprofielpinnen worden hersteld in alle gevonden sessieopslaglocaties van agents.
    - Doctor herstelt afzonderlijk verouderde `agentRuntime.id: "codex-cli"`-pinnen (een afzonderlijke verouderde runtime-ID) naar `"codex"` in de modelvermeldingen `agents.defaults`, `agents.entries.*` en `models.providers.*`.
    - `/codex ...` betekent "een systeemeigen Codex-gesprek vanuit de chat beheren of koppelen."
    - `/acp ...` of `runtime: "acp"` betekent "de externe ACP/acpx-adapter gebruiken."

  </Accordion>
  <Accordion title="2g. Opschoning van sessieroutes">
    Doctor scant gevonden sessieopslaglocaties van agents ook op verouderde, automatisch aangemaakte routestatus nadat je geconfigureerde modellen of de runtime hebt verplaatst van een route die eigendom is van een plugin, zoals Codex.

    `openclaw doctor --fix` kan automatisch aangemaakte verouderde status wissen, zoals `modelOverrideSource: "auto"`-modelpinnen, runtimemodelmetadata, vastgezette harness-ID's, CLI-sessiekoppelingen en automatische overschrijvingen van authenticatieprofielen wanneer de bijbehorende route niet meer is geconfigureerd. Expliciete modelkeuzes van gebruikers of uit verouderde sessies worden gemeld voor handmatige controle en blijven ongewijzigd; wijzig ze met `/model ...`, `/new` of stel de sessie opnieuw in wanneer die route niet langer bedoeld is.

  </Accordion>
  <Accordion title="3. Migraties van verouderde status (schijfindeling)">
    Doctor kan oudere schijfindelingen naar de huidige structuur migreren:

    - Sessieopslag en transcripties: van `~/.openclaw/sessions/` naar `~/.openclaw/agents/<agentId>/sessions/`
    - Agentmap: van `~/.openclaw/agent/` naar `~/.openclaw/agents/<agentId>/agent/`
    - WhatsApp-authenticatiestatus (Baileys): van het verouderde `~/.openclaw/credentials/*.json` (behalve `oauth.json`) naar `~/.openclaw/credentials/whatsapp/<accountId>/...` (standaardaccount-ID: `default`)
    - Ondertekende apparaatidentiteit: van `~/.openclaw/identity/device.json` naar de `primary` `device_identities`-rij in `state/openclaw.sqlite`; het afzonderlijke bestand voor apparaatauthenticatie blijft ongewijzigd

    Deze migraties worden naar beste vermogen uitgevoerd en zijn idempotent; doctor geeft waarschuwingen wanneer verouderde mappen als back-up achterblijven. De Gateway/CLI migreert bij het opstarten ook automatisch de verouderde sessies en agentmap, zodat geschiedenis/authenticatie/modellen zonder handmatige uitvoering van doctor in het pad per agent terechtkomen. WhatsApp-authenticatie wordt opzettelijk alleen via `openclaw doctor` gemigreerd. De normalisatie van Talk-provider/provider-map vergelijkt op structurele gelijkheid, zodat verschillen die alleen de sleutelvolgorde betreffen niet langer herhaaldelijk wijzigingen zonder effect in `doctor --fix` activeren.

  </Accordion>
  <Accordion title="3a. Migraties van verouderde pluginmanifesten">
    Doctor scant alle geïnstalleerde pluginmanifesten op verouderde mogelijkhedenleutels op het hoogste niveau (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders`). Wanneer deze worden gevonden, biedt doctor aan ze naar het object `contracts` te verplaatsen en het manifestbestand ter plaatse te herschrijven. Deze migratie is idempotent; als `contracts` al dezelfde waarden bevat, wordt de verouderde sleutel verwijderd zonder gegevens te dupliceren.
  </Accordion>
  <Accordion title="3b. Migraties van verouderde Cron-opslag">
    Doctor controleert ook de verouderde opslag voor Cron-taken (`~/.openclaw/cron/jobs.json`) op oude taakstructuren voordat canonieke rijen in SQLite worden geïmporteerd.

    De huidige Cron-opschoningen omvatten:

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - payloadvelden op het hoogste niveau (`message`, `model`, `thinking`, ...) → `payload`
    - afleveringsvelden op het hoogste niveau (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
    - afleveringsaliassen in payload `provider` → expliciete `delivery.channel`
    - verouderde `notify: true`-fallbacktaken voor Webhooks → expliciete Webhook-aflevering vanuit de uitgefaseerde onbewerkte waarde `cron.webhook` wanneer deze geldig is; aankondigingstaken behouden hun chataflevering en krijgen `delivery.completionDestination`. Doctor verwijdert vervolgens de oude configuratiesleutel. Zonder een bruikbare verouderde Webhook wordt de inactieve markering `notify` op het hoogste niveau verwijderd voor taken zonder doel (bestaande aflevering, inclusief aankondigingen, blijft behouden), omdat de runtimeaflevering deze nooit leest.

    De Gateway schoont ook onjuist gevormde Cron-rijen op tijdens het laden, zodat geldige taken blijven werken. Onbewerkte onjuist gevormde rijen worden vóór verwijdering uit `jobs.json` gekopieerd naar `jobs-quarantine.json` naast de actieve opslag; doctor meldt in quarantaine geplaatste rijen, zodat je ze handmatig kunt controleren of herstellen.

    Bij het opstarten normaliseert de Gateway de runtimeprojectie en negeert deze de markering `notify` op het hoogste niveau, maar laat de permanente Cron-status intact voor herstel door doctor. Doctor verwijdert inactieve markeringen voor taken zonder migratiedoel (`delivery.mode` geen/afwezig, een onbruikbaar verouderd Webhook-doel of bestaande aankondigings-/chataflevering) en laat de bestaande aflevering ongewijzigd, zodat herhaalde uitvoeringen van `doctor --fix` niet langer opnieuw voor dezelfde taak waarschuwen.

    Op Linux waarschuwt doctor ook wanneer de crontab van de gebruiker nog steeds het verouderde `~/.openclaw/bin/ensure-whatsapp.sh` aanroept. Dat hostlokale script wordt niet onderhouden door het huidige OpenClaw en kan onterechte `Gateway inactive`-berichten naar `~/.openclaw/logs/whatsapp-health.log` schrijven wanneer Cron de systemd-gebruikersbus niet kan bereiken. Verwijder de verouderde crontab-vermelding met `crontab -e`; gebruik `openclaw channels status --probe`, `openclaw doctor` en `openclaw gateway status` voor huidige statuscontroles.

  </Accordion>
  <Accordion title="3c. Opschonen van sessievergrendelingen">
    Doctor scant elke agentsessiemap op verouderde schrijfvergrendelingsbestanden die zijn achtergebleven toen een sessie abnormaal werd beëindigd. Voor elk gevonden vergrendelingsbestand rapporteert het: het pad, de PID, of de PID nog actief is, de ouderdom van de vergrendeling en of deze als verouderd wordt beschouwd (inactieve PID, ongeldige metadata van de eigenaar, ouder dan 30 minuten of een actieve PID waarvan is aangetoond dat die bij een niet-OpenClaw-proces hoort). In de modus `--fix` / `--repair` verwijdert het automatisch vergrendelingen met inactieve, verweesde, hergebruikte, ongeldige oude of niet-OpenClaw-eigenaren. Oude vergrendelingen die nog eigendom zijn van een actief OpenClaw-proces worden gerapporteerd maar blijven staan, zodat doctor een actieve transcriptschrijver niet onderbreekt.
  </Accordion>
  <Accordion title="3d. Reparatie van sessietranscripttakken">
    Doctor scant JSONL-bestanden van agentsessies op de gedupliceerde takstructuur die is ontstaan door de bug bij het herschrijven van prompttranscripten in 2026.4.24: een verlaten gebruikersbeurt met interne OpenClaw-runtimecontext plus een actieve zustertak met dezelfde zichtbare gebruikersprompt. In de modus `--fix` / `--repair` maakt doctor naast het origineel een back-up van elk getroffen bestand en herschrijft het transcript naar de actieve tak, zodat lezers van Gateway-geschiedenis en geheugen geen dubbele beurten meer zien.
  </Accordion>
  <Accordion title="4. Integriteitscontroles van de status (sessieopslag, routering en veiligheid)">
    De statusmap is de operationele hersenstam. Als deze verdwijnt, verlies je sessies, referenties, logboeken en configuratie, tenzij je elders back-ups hebt.

    Doctor controleert:

    - **Statusmap ontbreekt**: waarschuwt voor catastrofaal statusverlies, vraagt om de map opnieuw aan te maken en herinnert je eraan dat ontbrekende gegevens niet kunnen worden hersteld.
    - **Machtigingen van statusmap**: controleert de schrijfbaarheid; biedt aan de machtigingen te herstellen (en geeft een `chown`-hint wanneer een verschil in eigenaar/groep wordt gedetecteerd).
    - **Via de cloud gesynchroniseerde statusmap op macOS**: waarschuwt wanneer de status onder iCloud Drive (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) of `~/Library/CloudStorage/...` wordt gevonden, omdat door synchronisatie ondersteunde paden tragere I/O en conflicten tussen vergrendeling en synchronisatie kunnen veroorzaken.
    - **Statusmap op Linux-SD of -eMMC**: waarschuwt wanneer de status naar een `mmcblk*`-aankoppelbron wordt herleid, omdat willekeurige I/O op SD/eMMC trager kan zijn en de opslag sneller kan slijten bij het schrijven van sessies en referenties.
    - **Vluchtige statusmap op Linux**: waarschuwt wanneer de status naar `tmpfs` of `ramfs` wordt herleid, omdat sessies, referenties, configuratie en SQLite-status (met WAL-/journaalhulpbestanden) bij opnieuw opstarten verdwijnen. Docker-`overlay`-aankoppelingen worden bewust niet gemarkeerd, omdat hun beschrijfbare lagen behouden blijven wanneer de host opnieuw wordt opgestart zolang de container blijft bestaan.
    - **Sessiemappen ontbreken**: `sessions/` en de sessieopslagmap zijn vereist om geschiedenis te bewaren en `ENOENT`-crashes te voorkomen.
    - **Transcript komt niet overeen**: waarschuwt wanneer bij recente sessie-items transcriptbestanden ontbreken.
    - **Hoofdsessie met "JSONL van 1 regel"**: markeert wanneer het hoofdtranscript slechts één regel bevat (de geschiedenis wordt niet opgebouwd).
    - **Meerdere statusmappen**: waarschuwt wanneer meerdere `~/.openclaw`-mappen in thuismappen bestaan, of wanneer `OPENCLAW_STATE_DIR` naar een andere locatie verwijst (de geschiedenis kan over installaties worden verdeeld).
    - **Herinnering voor externe modus**: als `gateway.mode=remote`, herinnert doctor je eraan om het op de externe host uit te voeren (de status bevindt zich daar).
    - **Machtigingen van configuratiebestand**: waarschuwt als `~/.openclaw/openclaw.json` leesbaar is voor de groep/iedereen en biedt aan dit te beperken tot `600`.

  </Accordion>
  <Accordion title="5. Gezondheid van modelauthenticatie (verlopen van OAuth)">
    Doctor inspecteert OAuth-profielen in de authenticatieopslag, waarschuwt wanneer tokens bijna verlopen of verlopen zijn en kan ze vernieuwen wanneer dat veilig is. Als het Anthropic OAuth-/tokenprofiel verouderd is, stelt het een Anthropic API-sleutel of het Anthropic-installatietokenpad voor. Vernieuwingsprompts verschijnen alleen bij interactieve uitvoering (TTY); `--non-interactive` slaat vernieuwingspogingen over.

    Wanneer een OAuth-vernieuwing permanent mislukt (bijvoorbeeld `refresh_token_reused`, `invalid_grant` of wanneer een provider aangeeft dat je je opnieuw moet aanmelden), meldt doctor dat herauthenticatie vereist is en drukt het de exacte uit te voeren opdracht `openclaw models auth login --provider ...` af.

    Doctor rapporteert ook authenticatieprofielen die tijdelijk onbruikbaar zijn vanwege korte afkoelperioden (snelheidslimieten/time-outs/authenticatiefouten) of langere uitschakelingen (facturerings-/tegoedproblemen).

    Verouderde Codex OAuth-profielen waarvan de tokens in macOS Keychain staan (oudere onboarding van vóór de bestandsgebaseerde hulpbestandsindeling) worden alleen door doctor gerepareerd. Voer `openclaw doctor --fix` eenmaal uit vanuit een interactieve terminal om verouderde, door Keychain beheerde tokens rechtstreeks naar `auth-profiles.json` te migreren; daarna worden ze bij ingebedde beurten (Telegram, cron, verzending naar subagents) als canonieke OpenAI OAuth-profielen herkend.

  </Accordion>
  <Accordion title="6. Validatie van het model voor hooks">
    Als `hooks.gmail.model` is ingesteld, valideert doctor de modelverwijzing aan de hand van de catalogus en toelatingslijst en waarschuwt het wanneer deze niet kan worden gevonden of niet is toegestaan.
  </Accordion>
  <Accordion title="7. Reparatie van sandbox-installatiekopieën">
    Wanneer sandboxing is ingeschakeld, controleert doctor Docker-installatiekopieën en biedt het aan om te bouwen of naar verouderde namen over te schakelen als de huidige installatiekopie ontbreekt.
  </Accordion>
  <Accordion title="7b. Opschonen van Plugin-installaties">
    Doctor verwijdert in de modus `openclaw doctor --fix` / `openclaw doctor --repair` verouderde, door OpenClaw gegenereerde tijdelijke status voor Plugin-afhankelijkheden: verouderde gegenereerde afhankelijkheidshoofdmappen, oude installatiefasemappen, pakketlokale restanten van eerdere reparatiecode voor afhankelijkheden van gebundelde Plugins en verweesde of herstelde beheerde npm-kopieën van gebundelde `@openclaw/*`-Plugins die het huidige gebundelde manifest kunnen overschaduwen. Doctor koppelt ook het `openclaw`-pakket van de host opnieuw aan beheerde npm-Plugins die `peerDependencies.openclaw` declareren, zodat pakketlokale runtime-imports zoals `openclaw/plugin-sdk/*` na updates of npm-reparaties blijven werken.

    Doctor kan ook ontbrekende downloadbare Plugins opnieuw installeren wanneer de configuratie ernaar verwijst, maar het lokale Plugin-register ze niet kan vinden (materiële `plugins.entries`, geconfigureerde kanaal-/provider-/zoekinstellingen, geconfigureerde agentruntimes). Tijdens pakketupdates voorkomt doctor dat Plugin-pakketten opnieuw worden geïnstalleerd terwijl het kernpakket wordt vervangen; voer `openclaw doctor --fix` na de update opnieuw uit als een geconfigureerde Plugin nog moet worden hersteld. Buiten de uitzondering voor het opstarten van de containerinstallatiekopie hieronder voeren het opstarten van de Gateway en het opnieuw laden van de configuratie geen pakketreparatie uit; Plugin-installaties blijven expliciet doctor-/installatie-/updatewerk.

    Het opstarten van een gecontaineriseerde Gateway heeft een beperkte upgrade-uitzondering: wanneer `openclaw gateway run` op een nieuwe OpenClaw-versie start, voert het vóór gereedheid veilige statusmigraties en de bestaande convergentie van Plugins na de kern uit en registreert het vervolgens een controlepunt per versie. Deze opstartprocedure kan verouderde records van gebundelde Plugins opschonen, lokale Plugin-koppelingen repareren, geconfigureerde Plugin-pakketten opnieuw installeren wanneer het convergentiepad dit vereist en actieve Plugin-payloads controleren. Als het opstartproces geen veilige reparatie kan uitvoeren, voer je dezelfde installatiekopie eenmaal uit met `openclaw doctor --fix` tegen dezelfde aangekoppelde status/configuratie voordat je de container normaal opnieuw opstart.

  </Accordion>
  <Accordion title="8. Migraties van Gateway-services en opschoontips">
    Doctor detecteert verouderde Gateway-services (launchd/systemd/schtasks) en biedt aan deze te verwijderen en de OpenClaw-service met de huidige Gateway-poort te installeren. Het kan ook zoeken naar extra Gateway-achtige services en opschoontips afdrukken. OpenClaw Gateway-services met een profielnaam worden als volwaardige services beschouwd en niet als "extra" gemarkeerd.

    Als op Linux de Gateway-service op gebruikersniveau ontbreekt maar er een OpenClaw Gateway-service op systeemniveau bestaat, installeert doctor niet automatisch een tweede service op gebruikersniveau. Inspecteer met `openclaw gateway status --deep` of `openclaw doctor --deep` en verwijder vervolgens het duplicaat of stel `OPENCLAW_SERVICE_REPAIR_POLICY=external` in wanneer een systeemtoezichthouder de levenscyclus van de Gateway beheert.

  </Accordion>
  <Accordion title="8b. Matrix-migratie bij het opstarten">
    Wanneer een Matrix-kanaalaccount een wachtende of uitvoerbare migratie van verouderde status heeft, maakt doctor (in de modus `--fix` / `--repair`) een momentopname vóór de migratie en voert het vervolgens de migratiestappen volgens het best-effortprincipe uit: migratie van verouderde Matrix-status en voorbereiding van verouderde versleutelde status. Beide stappen zijn niet-fataal; fouten worden geregistreerd en het opstarten gaat door. In de alleen-lezenmodus (`openclaw doctor` zonder `--fix`) wordt deze controle volledig overgeslagen.
  </Accordion>
  <Accordion title="8c. Apparaatkoppeling en authenticatieafwijkingen">
    Doctor inspecteert de status van apparaatkoppelingen als onderdeel van de normale gezondheidscontrole en rapporteert:

    - wachtende eerste koppelingsverzoeken
    - wachtende rol- of bereikupgrades voor reeds gekoppelde apparaten
    - reparaties van niet-overeenkomende openbare sleutels waarbij de apparaat-id nog overeenkomt, maar de apparaatidentiteit niet meer overeenkomt met het goedgekeurde record
    - gekoppelde records zonder actief token voor een goedgekeurde rol
    - gekoppelde tokens waarvan het bereik buiten de goedgekeurde koppelingsbasislijn afwijkt
    - lokaal gecachte apparaattokenitems voor de huidige machine die dateren van vóór een tokenrotatie aan de Gateway-zijde of verouderde bereikmetadata bevatten

    Doctor keurt koppelingsverzoeken niet automatisch goed en roteert apparaattokens niet automatisch. Het drukt de exacte vervolgstappen af:

    - inspecteer wachtende verzoeken met `openclaw devices list`
    - keur het exacte verzoek goed met `openclaw devices approve <requestId>`
    - roteer een nieuw token met `openclaw devices rotate --device <deviceId> --role <role>`
    - verwijder een verouderd record en keur het opnieuw goed met `openclaw devices remove <deviceId>`

    Hiermee wordt onderscheid gemaakt tussen een eerste koppeling, wachtende rol-/bereikupgrades en verouderde afwijkingen in tokens/apparaatidentiteiten, waarmee het veelvoorkomende probleem "al gekoppeld, maar nog steeds melding dat koppeling vereist is" wordt opgelost.

  </Accordion>
  <Accordion title="9. Beveiligingswaarschuwingen">
    Doctor geeft alleen een beveiligingsmelding als het een waarschuwing aantreft, zoals een provider die zonder toelatingslijst openstaat voor privéberichten of een gevaarlijk geconfigureerd beleid. Gebruik `openclaw security audit` voor de volledige beveiligingsinventaris.
  </Accordion>
  <Accordion title="10. systemd-linger (Linux)">
    Als doctor als een systemd-gebruikersservice wordt uitgevoerd, zorgt het ervoor dat linger is ingeschakeld, zodat de Gateway actief blijft nadat je je hebt afgemeld.
  </Accordion>
  <Accordion title="11. Werkruimtestatus (Skills, Plugins en TaskFlows)">
    Doctor drukt problemen en acties voor de standaardagent af, niet de inventaris van de gezonde status:

    - **Skills**: vermeldt toegestane maar onbruikbare Skill-namen; gebruik `openclaw skills check` voor details over vereisten en volledige aantallen.
    - **Plugins**: rapporteert alleen Plugin-ID's met fouten; gebruik `openclaw plugins list` voor een inventaris van geladen, geïmporteerde, uitgeschakelde en gebundelde Plugins.
    - **Waarschuwingen voor Plugin-compatibiliteit**: markeert Plugins die compatibiliteitsproblemen hebben met de huidige runtime.
    - **Plugin-diagnostiek**: toont waarschuwingen of fouten die tijdens het laden door het Plugin-register zijn gegenereerd.
    - **TaskFlow-herstel**: toont verdachte beheerde TaskFlows die handmatig moeten worden geïnspecteerd of geannuleerd.
    - **Claude CLI**: rapporteert alleen problemen met het binaire bestand, de authenticatie, het profiel, de werkruimte of de projectmap; details van geslaagde controles worden weggelaten.

  </Accordion>
  <Accordion title="11b. Grootte van bootstrapbestanden">
    Doctor controleert of bootstrapbestanden van de werkruimte (bijvoorbeeld `AGENTS.md`, `CLAUDE.md` of andere geïnjecteerde contextbestanden) de geconfigureerde tekenlimiet naderen of overschrijden. Het rapporteert per bestand het aantal ruwe versus geïnjecteerde tekens, het afkappingspercentage, de oorzaak van de afkapping (`max/file` of `max/total`) en het totale aantal geïnjecteerde tekens als fractie van het totale budget. Wanneer bestanden zijn afgekapt of de limiet naderen, drukt doctor tips af voor het afstemmen van `agents.defaults.bootstrapMaxChars` en `agents.defaults.bootstrapTotalMaxChars`.
  </Accordion>
  <Accordion title="11c. Shell-aanvulling">
    Doctor controleert of tabaanvulling is geïnstalleerd voor de huidige shell (zsh, bash, fish of PowerShell):

    - Als het shellprofiel een traag dynamisch aanvullingspatroon gebruikt (`source <(openclaw completion ...)`), werkt doctor dit bij naar de snellere variant met een gecachet bestand.
    - Als aanvulling in het profiel is geconfigureerd maar het cachebestand ontbreekt, genereert doctor de cache automatisch opnieuw.
    - Als er helemaal geen aanvulling is geconfigureerd, vraagt doctor om deze te installeren (alleen in interactieve modus; overgeslagen met `--non-interactive`).

    Voer `openclaw completion --write-state` uit om de cache handmatig opnieuw te genereren.

  </Accordion>
  <Accordion title="11d. Verouderde kanaalplugin opschonen">
    Wanneer `openclaw doctor --fix` een ontbrekende kanaalplugin verwijdert, verwijdert het ook de achtergebleven kanaalspecifieke configuratie die naar die plugin verwees: `channels.<id>`-vermeldingen, heartbeat-doelen waarin het kanaal werd genoemd en `agents.*.models["<channel>/*"]`-overschrijvingen. Dit voorkomt opstartlussen van de Gateway waarbij de kanaalruntime verdwenen is, maar de configuratie de Gateway nog steeds vraagt eraan te koppelen.
  </Accordion>
  <Accordion title="12. Gateway-authenticatiecontroles (lokaal token)">
    Doctor controleert of lokale Gateway-authenticatie met een token gereed is.

    - Als de tokenmodus een token vereist en er geen tokenbron bestaat, biedt doctor aan er een te genereren.
    - Als `gateway.auth.token` door SecretRef wordt beheerd maar niet beschikbaar is, waarschuwt doctor en overschrijft het deze niet met platte tekst.
    - `openclaw doctor --generate-gateway-token` dwingt alleen generatie af wanneer er geen token-SecretRef is geconfigureerd.

  </Accordion>
  <Accordion title="12b. Alleen-lezen reparaties met SecretRef-ondersteuning">
    Sommige reparatieprocessen moeten geconfigureerde aanmeldgegevens inspecteren zonder het fail-fast-gedrag van de runtime te verzwakken.

    - `openclaw doctor --fix` gebruikt voor gerichte configuratiereparaties hetzelfde alleen-lezen SecretRef-overzichtsmodel als statusgerelateerde opdrachten.
    - Voorbeeld: de reparatie van Telegram `allowFrom` / `groupAllowFrom` `@username` probeert geconfigureerde botaanmeldgegevens te gebruiken wanneer die beschikbaar zijn.
    - Als het Telegram-bottoken via SecretRef is geconfigureerd maar niet beschikbaar is in het huidige opdrachtpad, meldt doctor dat de aanmeldgegevens geconfigureerd-maar-niet-beschikbaar zijn en slaat het automatische oplossing over, in plaats van te crashen of ten onrechte te melden dat het token ontbreekt.

  </Accordion>
  <Accordion title="13. Gateway-statuscontrole en herstart">
    Doctor voert een statuscontrole uit en biedt aan de Gateway opnieuw te starten wanneer deze niet gezond lijkt.
  </Accordion>
  <Accordion title="13b. Gereedheid van geheugenzoekopdrachten">
    Doctor controleert of de geconfigureerde embeddingprovider voor geheugenzoekopdrachten gereed is voor de standaardagent. Het gedrag hangt af van de geconfigureerde backend en provider:

    - **QMD-backend**: controleert of het binaire bestand `qmd` beschikbaar is en kan worden gestart. Zo niet, dan wordt reparatieadvies weergegeven, waaronder `npm install -g @tobilu/qmd` (of het Bun-equivalent) en een optie voor een handmatig pad naar het binaire bestand.
    - **Expliciete lokale provider**: controleert op een lokaal modelbestand of een herkende externe/downloadbare model-URL. Als dit ontbreekt, wordt voorgesteld over te schakelen naar een externe provider.
    - **Expliciete externe provider** (`openai`, `voyage`, enz.): verifieert dat er een API-sleutel aanwezig is in de omgeving of authenticatieopslag. Geeft uitvoerbare reparatietips weer als deze ontbreekt.
    - **Verouderde automatische provider**: behandelt `memorySearch.provider: "auto"` als OpenAI, controleert of OpenAI gereed is en `doctor --fix` herschrijft deze naar `provider: "openai"`.

    Wanneer een gecachet resultaat van een Gateway-controle beschikbaar is (de Gateway was gezond op het moment van de controle), vergelijkt doctor het resultaat met de configuratie die zichtbaar is via de CLI en vermeldt het eventuele afwijkingen. Doctor start in het standaardpad geen nieuwe embedding-ping; gebruik de uitgebreide geheugenstatusopdracht als je een livecontrole van de provider wilt.

    Gebruik `openclaw memory status --deep` om tijdens runtime te verifiëren of embeddings gereed zijn.

  </Accordion>
  <Accordion title="14. Waarschuwingen over kanaalstatus">
    Als de Gateway gezond is, voert doctor een kanaalstatuscontrole uit en meldt het waarschuwingen met voorgestelde oplossingen.
  </Accordion>
  <Accordion title="15. Controle en reparatie van supervisorconfiguratie">
    Doctor controleert de geïnstalleerde supervisorconfiguratie (launchd/systemd/schtasks) op ontbrekende of verouderde standaardinstellingen (bijvoorbeeld systemd-afhankelijkheden voor network-online en de vertraging voor opnieuw starten). Wanneer doctor een afwijking vindt, beveelt het een update aan en kan het servicebestand of de taak worden herschreven met de huidige standaardinstellingen.

    Opmerkingen:

    - `openclaw doctor` vraagt om bevestiging voordat de supervisorconfiguratie wordt herschreven.
    - `openclaw doctor --yes` accepteert de standaardvragen voor reparaties.
    - `openclaw doctor --fix` past aanbevolen oplossingen toe zonder vragen (`--repair` is een alias).
    - `openclaw doctor --fix --force` overschrijft aangepaste supervisorconfiguraties.
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` houdt doctor alleen-lezen voor de levenscyclus van de Gateway-service. Doctor meldt nog steeds de servicestatus en voert reparaties uit die geen betrekking hebben op de service, maar slaat installatie/start/herstart/bootstrap van de service, het herschrijven van de supervisorconfiguratie en het opschonen van verouderde services over, omdat een externe supervisor die levenscyclus beheert.
    - Op Linux herschrijft doctor geen metadata van opdrachten/toegangspunten zolang de bijbehorende systemd-eenheid van de Gateway actief is. Tijdens de scan naar dubbele services negeert het ook inactieve, niet-verouderde extra Gateway-achtige eenheden, zodat aanvullende servicebestanden geen onnodige opschoonmeldingen veroorzaken.
    - Als tokenauthenticatie een token vereist en `gateway.auth.token` door SecretRef wordt beheerd, valideert de installatie/reparatie van de doctor-service de SecretRef, maar worden opgeloste tokenwaarden in platte tekst niet opgeslagen in de omgevingsmetadata van de supervisorservice.
    - Doctor detecteert beheerde `.env`-waarden/door SecretRef ondersteunde serviceomgevingswaarden die door oudere installaties van LaunchAgent, systemd of Windows Scheduled Task inline zijn ingesloten, en herschrijft de servicemetadata zodat die waarden vanuit de runtimebron worden geladen in plaats van vanuit de supervisordefinitie.
    - Doctor detecteert wanneer de serviceopdracht na wijzigingen aan `gateway.port` nog steeds een oude `--port` vastlegt en herschrijft de servicemetadata naar de huidige poort.
    - Als tokenauthenticatie een token vereist en de geconfigureerde token-SecretRef niet kan worden opgelost, blokkeert doctor het installatie-/reparatiepad met uitvoerbaar advies.
    - Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd en `gateway.auth.mode` niet is ingesteld, blokkeert doctor installatie/reparatie totdat de modus expliciet is ingesteld.
    - Voor systemd-eenheden van Linux-gebruikers omvatten de controles van doctor op tokenafwijkingen zowel `Environment=`- als `EnvironmentFile=`-bronnen bij het vergelijken van metadata voor serviceauthenticatie.
    - Reparaties door de doctor-service weigeren een Gateway-service van een ouder binair OpenClaw-bestand te herschrijven, stoppen of opnieuw te starten wanneer de configuratie het laatst door een nieuwere versie is geschreven. Zie [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting#split-brain-installs-and-newer-config-guard).
    - Je kunt altijd een volledige herschrijving afdwingen via `openclaw gateway install --force`.

  </Accordion>
  <Accordion title="16. Diagnostiek van Gateway-runtime en poort">
    Doctor inspecteert de serviceruntime (PID, laatste afsluitstatus) en waarschuwt wanneer de service is geïnstalleerd maar niet daadwerkelijk wordt uitgevoerd. Het controleert ook op poortconflicten op de Gateway-poort (standaard `18789`) en meldt waarschijnlijke oorzaken (Gateway wordt al uitgevoerd, SSH-tunnel).
  </Accordion>
  <Accordion title="17. Aanbevolen procedures voor de Gateway-runtime">
    Doctor waarschuwt wanneer de Gateway-service wordt uitgevoerd met Bun of via een door een versiebeheerder beheerd Node-pad (`nvm`, `fnm`, `volta`, `asdf`, enz.). Bun kan de `node:sqlite`-statusopslag van OpenClaw niet openen, daarom migreren reparaties verouderde Bun-services naar Node. Paden van versiebeheerders kunnen na upgrades niet meer werken, omdat de service de initialisatie van je shell niet laadt. Doctor biedt aan naar een systeeminstallatie van Node te migreren wanneer die beschikbaar is (Homebrew/apt/choco).

    Nieuw geïnstalleerde of gerepareerde macOS-LaunchAgents gebruiken een canoniek systeem-PATH (`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`) in plaats van het interactieve shell-PATH te kopiëren. Daardoor blijven door Homebrew beheerde systeembinaire bestanden beschikbaar, terwijl mappen van Volta, asdf, fnm, pnpm en andere versiebeheerders niet wijzigen welke Node-processen door onderliggende processen worden gevonden. Linux-services behouden nog steeds expliciete omgevingshoofdmappen (`NVM_DIR`, `FNM_DIR`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `BUN_INSTALL`, `PNPM_HOME`) en stabiele binaire gebruikersmappen, maar geschatte terugvalmappen van versiebeheerders worden alleen naar het service-PATH geschreven wanneer die mappen op schijf bestaan.

  </Accordion>
  <Accordion title="18. Configuratie schrijven en wizardmetadata">
    Doctor slaat alle configuratiewijzigingen op en voegt wizardmetadata toe om de doctor-uitvoering vast te leggen.
  </Accordion>
  <Accordion title="19. Werkruimtetips (back-up en geheugensysteem)">
    Doctor stelt een geheugensysteem voor de werkruimte voor wanneer dit ontbreekt en geeft een back-uptip weer als de werkruimte nog niet onder git valt.

    Zie [/concepten/agentwerkruimte](/nl/concepts/agent-workspace) voor een volledige handleiding voor de werkruimtestructuur en git-back-ups (een privérepository op GitHub of GitLab wordt aanbevolen).

  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Draaiboek voor de Gateway](/nl/gateway)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
