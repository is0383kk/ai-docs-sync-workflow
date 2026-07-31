---
read_when:
    - Live visuele QA bouwen of uitvoeren voor OpenClaw-bugs
    - Verificatie vóór en na toevoegen voor een pull request
    - Discord-, Slack-, WhatsApp- of andere livetransportscenario's toevoegen
    - Gerichte browservalidatie voor de Control UI uitvoeren voor een kandidaatref.
    - QA-runs debuggen waarvoor schermafbeeldingen, browserautomatisering of VNC-toegang nodig zijn
summary: Mantis legt visueel end-to-endbewijs vast voor live transportvergelijkingen en gerichte browserbewijzen voor uitsluitend kandidaten, en voegt de artefacten vervolgens toe aan PR's.
title: Mantis
x-i18n:
    generated_at: "2026-07-27T06:11:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 48a1b306e37aba7e8c67139df61f3680a9aec066361aa196d88c81270337bc1b
    source_path: concepts/mantis.md
    workflow: 16
---

Mantis publiceert visueel CI-bewijs en een PR-opmerking voor OpenClaw-gedrag.
Live-transportscenario's vergelijken een bekende foutieve basislijn met een kandidaatref;
gerichte browserlanen kunnen in plaats daarvan één kandidaat toetsen aan een deterministisch
gemockt transport. Discord was de eerste uitgebrachte integratie, met echte botauthenticatie, guild-kanalen,
reacties, threads en een browsergetuige. Er bestaan ook lanen voor Slack, Telegram en gerichte Control
UI-chat; WhatsApp en Matrix zijn niet geïmplementeerd.

## Eigenaarschap

- OpenClaw (`extensions/qa-lab/src/mantis/*`): scenarioruntime, `pnpm openclaw qa mantis <command>`-CLI, bewijsschema.
- QA Lab (`extensions/qa-lab/src/live-transports/*`): live-transportharnas, driver-/SUT-bots, rapport-/bewijsschrijvers.
- Crabbox (`openclaw/crabbox`): opgewarmde Linux-machines, leases, VNC, `crabbox media preview`.
- GitHub Actions (`.github/workflows/mantis-*.yml`): externe toegangspunten, bewaring van artefacten.
- ClawSweeper: parseert PR-opdrachten van beheerders, start workflows en plaatst de definitieve PR-opmerking.

## CLI-opdrachten

Alle opdrachten zijn `pnpm openclaw qa mantis <command>`, gedefinieerd in
`extensions/qa-lab/src/mantis/cli.ts`. Vereist `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1`
tijdens bouwen/uitvoeren (gebundelde workflows stellen vóór het bouwen `OPENCLAW_BUILD_PRIVATE_QA=1` en
`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` in).

| Opdracht                        | Doel                                                                                                                                                      |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discord-smoke`                 | Controleren of de Mantis Discord-bot de guild/het kanaal kan zien, berichten kan plaatsen en kan reageren.                                                |
| `run`                           | Een voor-/na-scenario uitvoeren met basislijn- en kandidaatrefs (alleen Discord).                                                                          |
| `desktop-browser-smoke`         | Een Crabbox-desktop leasen/hergebruiken, een zichtbare browser openen en een schermafbeelding + video vastleggen.                                          |
| `slack-desktop-smoke`           | Een Crabbox-desktop leasen/hergebruiken, daarin Slack-QA uitvoeren, Slack Web openen en bewijs vastleggen.                                                 |
| `telegram-desktop-builder`      | Een Crabbox-desktop leasen/hergebruiken, Telegram Desktop installeren en optioneel een OpenClaw-Gateway configureren.                                      |
| `visual-task` / `visual-driver` | Algemene Crabbox-desktopvastlegging met optionele controles voor beeldbegrip; `visual-driver` is de driverhelft die onder `crabbox record --while` wordt gestart. |

Elke opdracht accepteert `--repo-root <path>` en `--output-dir <path>`; Crabbox-
opdrachten accepteren ook `--crabbox-bin`, `--provider`, `--machine-class`/`--class`,
`--lease-id`, `--idle-timeout`, `--ttl` en `--keep-lease`. Lokale CLI-standaardwaarden
voor provider/klasse zijn `hetzner`/`beast`, tenzij anders vermeld; CI-workflows
overschrijven doorgaans beide.

### `discord-smoke`

```bash
pnpm openclaw qa mantis discord-smoke \
  --output-dir .artifacts/qa-e2e/mantis/discord-smoke
```

Roept de Discord REST-API (`https://discord.com/api/v10`) aan om de bot-
gebruiker, de guild, de kanalen van de guild en het doelkanaal op te halen, controleert of het
kanaal bij de guild hoort en plaatst vervolgens (tenzij `--skip-post`) een bericht en
voegt een `👀`-reactie toe. Schrijft `mantis-discord-smoke-summary.json` en
`mantis-discord-smoke-report.md`.

Volgorde voor tokenresolutie: waarde van `--token-file`, vervolgens `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
(overschrijven met `--token-env`), daarna een bestand dat door `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN_FILE` wordt aangeduid
(overschrijven met `--token-file-env`). Guild-/kanaal-id's komen uit
`OPENCLAW_QA_DISCORD_GUILD_ID` / `OPENCLAW_QA_DISCORD_CHANNEL_ID` (overschrijven met
`--guild-id` / `--channel-id`) en moeten Discord-snowflakes van 17-20 cijfers zijn. Stel
`OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` in om bot-/guild-/kanaal-/bericht-id's
en namen in de gepubliceerde samenvatting en het rapport te vervangen door `<redacted>`.

### `run`

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-status-reactions-tool-only \
  --baseline origin/main \
  --candidate HEAD \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-status-reactions
```

`--transport` accepteert momenteel alleen `discord`. `--scenario` is een van twee
ingebouwde id's, elk met een eigen standaardref voor de basislijn en verwachte voor-/na-
labels (`extensions/qa-lab/src/mantis/run.runtime.ts`):

| Scenario                                   | Standaardbasislijn                         | Basislijn verwacht                       | Kandidaat verwacht            |
| ------------------------------------------ | ------------------------------------------ | ---------------------------------------- | ----------------------------- |
| `discord-status-reactions-tool-only`       | `0bf06e953fdda290799fc9fb9244a8f67fdae593` | `queued-only`                            | `queued -> thinking -> done` |
| `discord-thread-reply-filepath-attachment` | `81349cdc2a9d5143fd0991ed858b739e7d96e05c` | threadantwoord laat `filePath`-bijlage weg | threadantwoord bevat deze     |

`--candidate` is standaard `HEAD`. Andere vlaggen: `--credential-source`
(standaard `convex`), `--credential-role` (standaard `ci`), `--provider-mode`
(standaard `live-frontier`), `--fast` (standaard ingeschakeld), `--skip-install`, `--skip-build`.

De runner maakt losgekoppelde `git worktree`-check-outs voor de basislijn en
kandidaat onder `<output-dir>/worktrees/`, voert `pnpm install`/`pnpm build` in
elk daarvan uit (tenzij overgeslagen) en voert vervolgens
`pnpm openclaw qa discord --scenario <id> --model openai/gpt-5.4 --alt-model openai/gpt-5.4 --allow-failures`
uit voor elke worktree. Elke lane schrijft `discord-qa-reaction-timelines.json`
plus een `<scenario-id>-timeline.html`/`.png`-paar; de runner kopieert dit
bewijs terug onder `baseline/`/`candidate/`, schrijft `comparison.json`,
`mantis-report.md` en `mantis-evidence.json` in de uitvoermap en
sluit af met een niet-nulcode als de vergelijking niet is geslaagd (basislijn `fail` en kandidaat
`pass`).

Het tweede Discord-scenario (`discord-thread-reply-filepath-attachment`) plaatst
een bovenliggend bericht met de driverbot, maakt een echte thread, roept de
`message.thread-reply`-actie van de SUT aan met een repo-lokale `filePath` en controleert vervolgens herhaaldelijk
de thread op het antwoord en de bestandsnaam van de bijlage. Het verwacht een bijlage
met de naam `mantis-thread-report.md`.

### `desktop-browser-smoke`

```bash
pnpm openclaw qa mantis desktop-browser-smoke \
  --output-dir .artifacts/qa-e2e/mantis/desktop-browser
```

Leaset of hergebruikt een Crabbox-desktop, start een browser binnen de VNC-sessie
die naar `--browser-url` (standaard `https://openclaw.ai`) of een gerenderde
`--html-file` wijst, wacht, maakt een schermafbeelding met `scrot`, neemt optioneel een MP4 op met
`ffmpeg` en synchroniseert `desktop-browser-smoke.png` / `.mp4` / `remote-metadata.json` via rsync
terug naar `--output-dir`.

Vlaggen:

- `--lease-id <cbx_...>` hergebruikt een opgewarmde desktop in plaats van er een te maken.
- `--browser-profile-dir <remote-path>` hergebruikt een externe Chrome-user-data-dir, zodat een permanente desktop tussen uitvoeringen aangemeld blijft (gebruikt voor een langlevend Discord Web-weergaveprofiel).
- `--browser-profile-archive-env <name>` herstelt vóór het starten een base64-`.tgz`-archief met een Chrome-profiel uit die omgevingsvariabele (standaard `OPENCLAW_MANTIS_BROWSER_PROFILE_TGZ_B64`); gebruikt voor aangemelde getuigen zoals Discord Web.
- `--video-duration <seconds>` bepaalt de duur van de MP4-opname (standaard 10s).
- `--keep-lease` (of `OPENCLAW_MANTIS_KEEP_VM=1`) houdt een lease die tijdens deze uitvoering is gemaakt open voor VNC-inspectie; mislukte uitvoeringen die een lease hebben gemaakt, houden deze standaard ook open.

Voor bewijs uit Discord Web gebruikt Mantis een speciaal weergaveaccount, geen bot-
token. De Discord REST-orakel (via `qa discord`) blijft gezaghebbend; wanneer
`OPENCLAW_QA_DISCORD_CAPTURE_UI_METADATA=1` is ingesteld, schrijft het scenario ook een
Discord Web-URL-artefact en houdt `OPENCLAW_QA_DISCORD_KEEP_THREADS=1` de
thread lang genoeg open zodat de browser deze kan openen.

De GitHub-workflow geeft de voorkeur aan een permanent weergaveprofiel via
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` (volledige profielarchieven kunnen groter zijn dan
de geheimgroottelimiet van GitHub); voor kleine/bootstrap-profielen kan deze in plaats daarvan een
base64-`.tgz` uit `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` herstellen. Wanneer
geen van beide bronnen is geconfigureerd, publiceert de workflow nog steeds de deterministische
schermafbeeldingen van basislijn/kandidaat en logt deze dat de aangemelde getuige is
overgeslagen.

### `slack-desktop-smoke`

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --output-dir .artifacts/qa-e2e/mantis/slack-desktop \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Leaset of hergebruikt een Crabbox-desktop, synchroniseert de check-out naar de VM, voert
`pnpm openclaw qa slack` daarin uit, opent Slack Web in de VNC-browser,
legt de desktop vast en kopieert zowel de Slack-QA-artefacten (`slack-qa/`) als
de VNC-schermafbeelding/-video lokaal terug. Dit is de enige Mantis-vorm waarbij de
SUT-Gateway en de browser beide binnen dezelfde VM worden uitgevoerd.

Met `--gateway-setup` maakt de opdracht een permanente wegwerpbare OpenClaw-
thuismap op `$HOME/.openclaw-mantis/slack-openclaw` in de VM, past de Slack-
Socket Mode-configuratie aan voor het doelkanaal, start
`openclaw gateway run --dev --allow-unconfigured --port 38973` en laat
Chrome actief in de VNC-sessie; als `--gateway-setup` wordt weggelaten, wordt in plaats daarvan de normale
bot-naar-bot-Slack-QA-lane uitgevoerd.

Vereiste omgevingsvariabelen voor `--credential-source env` (lokale standaardwaarde is `env`; standaardwaarde voor rol
is `maintainer`):

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`
- `OPENCLAW_LIVE_OPENAI_KEY` voor de externe modellane (als lokaal alleen `OPENAI_API_KEY`
  is ingesteld, kopieert Mantis deze naar `OPENCLAW_LIVE_OPENAI_KEY` voordat
  Crabbox wordt aangeroepen)

Met `--credential-source convex` leaset Mantis de Slack-SUT-aanmeldgegevens uit
de gedeelde pool voordat de VM wordt gemaakt en geeft het kanaal-id, de app-token en
de bot-token door aan de VM als `OPENCLAW_MANTIS_SLACK_*`-omgevingsvariabelen, zodat GitHub-
workflows alleen het Convex-brokergeheim nodig hebben, niet de onbewerkte Slack-tokens.

Andere vlaggen: `--slack-url <url>` opent een specifieke URL (anders leidt Mantis
`https://app.slack.com/client/<team>/<channel>` af uit `auth.test`);
`--slack-channel-id <id>` stelt het kanaal voor de Gateway-toestaanlijst in;
`OPENCLAW_MANTIS_SLACK_BROWSER_PROFILE_DIR` beheert het permanente Chrome-
profiel binnen de VM (standaard `$HOME/.config/openclaw-mantis/slack-chrome-profile`);
`--approval-checkpoints` voert de native Slack-goedkeuringsscenario's uit
(`slack-approval-exec-native`, `slack-approval-plugin-native`) en rendert
schermafbeeldingen van wachtende/afgehandelde controlepunten in plaats van Gateway-instelling (wederzijds
uitsluitend met `--gateway-setup`); `--hydrate-mode source|prehydrated`,
`--provider-mode`, `--model`, `--alt-model` en `--fast` worden doorgegeven aan de
live Slack-lane.

Schermafbeeldingen van goedkeuringscontrolepunten worden gerenderd uit het Slack-API-bericht dat het
scenario heeft waargenomen, niet uit de live Slack-UI; `slack-desktop-smoke.png` is alleen
bewijs van Slack Web zelf wanneer het browserprofiel van de lease al was aangemeld.

### `telegram-desktop-builder`

```bash
pnpm openclaw qa mantis telegram-desktop-builder \
  --credential-source convex \
  --credential-role maintainer \
  --keep-lease
```

Leaset of hergebruikt een Crabbox-desktop, installeert de native Linux-versie van Telegram Desktop,
herstelt optioneel een gebruikerssessiearchief, configureert OpenClaw met de
geleasete Telegram-SUT-bottoken, start
`openclaw gateway run --dev --allow-unconfigured --port 38974`, plaatst een
gereedheidsbericht van de driverbot in de geleasete privégroep en legt vervolgens een
schermafbeelding en MP4 vast. Een bottoken configureert alleen OpenClaw; hiermee wordt
nooit aangemeld bij Telegram Desktop. De desktopviewer is een afzonderlijke Telegram-gebruikerssessie
die wordt hersteld uit `--telegram-profile-archive-env <name>` of handmatig
via VNC wordt aangemeld en actief wordt gehouden met `--keep-lease`.

Vlaggen: `--lease-id <cbx_...>` voert opnieuw uit op een VM die al is aangemeld bij
Telegram Desktop; `--telegram-profile-archive-env <name>` herstelt vóór het starten een base64-
`.tgz`-profielarchief; `--telegram-profile-dir <remote-path>`
stelt de externe profielmap in (standaard `$HOME/.local/share/TelegramDesktop`);
`--no-gateway-setup` installeert en opent alleen Telegram Desktop;
`--credential-source`/`--credential-role` zijn standaard `convex`/`maintainer`.

## Bewijsmanifest

Elk scenario dat naar een PR publiceert, schrijft `mantis-evidence.json` naast
het rapport:

```json
{
  "schemaVersion": 1,
  "id": "discord-status-reactions",
  "title": "QA van Mantis Discord-statusreacties",
  "summary": "Voor mensen leesbare hoofdsamenvatting voor de PR-opmerking.",
  "scenario": "discord-status-reactions-tool-only",
  "comparison": {
    "baseline": { "sha": "...", "status": "fail", "expected": "queued-only" },
    "candidate": { "sha": "...", "status": "pass", "expected": "queued -> thinking -> done" },
    "pass": true
  },
  "artifacts": [
    {
      "kind": "timeline",
      "lane": "baseline",
      "label": "Baseline alleen in wachtrij",
      "path": "baseline/timeline.png",
      "targetPath": "baseline.png",
      "alt": "Discord-tijdlijn van baseline",
      "width": 420
    }
  ]
}
```

Artefact-`path` is relatief ten opzichte van de map van het manifest; `targetPath` is
relatief ten opzichte van het geconfigureerde R2/S3-artefactvoorvoegsel. `scripts/mantis/publish-pr-evidence.mjs`
weigert padtraversal en slaat vermeldingen met `"required": false` over wanneer het
bestand ontbreekt.

Artefactsoorten: `timeline` (deterministische schermafbeelding vóór/na),
`desktopScreenshot` (VNC-/browserschermafbeelding), `motionPreview` (inline geanimeerde
GIF uit de opname), `motionClip` (op beweging bijgesneden MP4), `fullVideo` (volledige
opname), `metadata` (JSON-/log-zijbestand), `report` (Markdown-rapport).

De artefactindeling van een uitvoering op schijf:

```text
.artifacts/qa-e2e/mantis/<run-id>/
  mantis-report.md
  mantis-evidence.json
  baseline/
  candidate/
  comparison.json
```

Schermafbeeldingen zijn bewijsmateriaal, geen geheimen, maar vereisen nog steeds zorgvuldige redactie:
namen van privékanalen, gebruikersnamen of berichtinhoud kunnen zichtbaar zijn. Stel
`OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` in voor openbare artefactuploads; dit is
standaard ingeschakeld in de GitHub-workflows voor Discord/Slack/Telegram.

## GitHub-automatisering

`scripts/mantis/publish-pr-evidence.mjs` is de herbruikbare publicatiefunctie. Workflows
roepen deze aan met het manifest, de doel-PR, de hoofdmap voor doelartefacten, de opmerkingsmarkering,
de artefact-URL, de uitvoerings-URL en de aanvraagbron. Deze uploadt gedeclareerde artefacten naar
de Mantis R2-bucket, bouwt een PR-opmerking met de samenvatting voorop, inline
afbeeldingen/voorbeelden en gekoppelde video's, en werkt vervolgens de bestaande markeringsopmerking bij of
maakt een nieuwe. Vereiste omgevingsvariabelen:

- `MANTIS_ARTIFACT_R2_ACCESS_KEY_ID`
- `MANTIS_ARTIFACT_R2_SECRET_ACCESS_KEY`
- `MANTIS_ARTIFACT_R2_BUCKET` (workflows stellen `openclaw-crabbox-artifacts` in)
- `MANTIS_ARTIFACT_R2_ENDPOINT`
- `MANTIS_ARTIFACT_R2_REGION` (workflows stellen `auto` in)
- `MANTIS_ARTIFACT_R2_PUBLIC_BASE_URL` (workflows stellen `https://artifacts.openclaw.ai` in)

Opmerkingen worden geplaatst via de Mantis GitHub App (`MANTIS_GITHUB_APP_ID` /
`MANTIS_GITHUB_APP_PRIVATE_KEY`), niet via `github-actions[bot]`, waarbij een verborgen
markeringsopmerking als upsert-sleutel wordt gebruikt.

| Workflow                          | Trigger                                                                                    | Wat deze doet                                                                                                                                                                                                                                                                                                     |
| --------------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Mantis Discord Smoke`            | handmatige activering                                                                            | Voert `discord-smoke` uit op een gekozen ref.                                                                                                                                                                                                                                                                       |
| `Mantis Discord Status Reactions` | PR-opmerking of handmatige activering                                                              | Bouwt afzonderlijke baseline-/candidate-worktrees, voert `discord-status-reactions-tool-only` op elk uit, rendert de tijdlijn van elke lane in een Crabbox-desktopbrowser, genereert op beweging bijgesneden GIF-/MP4-voorbeelden met `crabbox media preview`, uploadt artefacten en plaatst inline PR-bewijsmateriaal.                                 |
| `Mantis Scenario`                 | handmatige activering                                                                            | Algemene dispatcher: ontvangt `scenario_id` (`discord-status-reactions-tool-only`, `discord-thread-reply-filepath-attachment`, `slack-desktop-smoke`, `telegram-live`, `telegram-desktop-proof`, `web-ui-chat-proof`), `baseline_ref`, `candidate_ref`, `pr_number` en stuurt deze door naar de overeenkomende scenarioworkflow. |
| `Mantis Slack Desktop Smoke`      | handmatige activering                                                                            | Leaset een Crabbox Linux-desktop (standaard `aws`, keuze uit `hetzner`), voert `slack-desktop-smoke --gateway-setup` uit op de candidate, neemt het bureaublad op, genereert een bewegingsvoorbeeld, uploadt artefacten en plaatst PR-bewijsmateriaal wanneer een PR-nummer is opgegeven.                                                      |
| `Mantis Telegram Live`            | PR-opmerking of handmatige activering                                                              | Voert de live QA-lane voor de Telegram-bot-API uit (`openclaw qa telegram`), schrijft `mantis-evidence.json` vanuit de QA-samenvatting, rendert geredigeerde bewijs-HTML via een Crabbox-desktopbrowser, genereert een bewegings-GIF en plaatst PR-bewijsmateriaal. Aanmelden bij Telegram Web is voor deze lane niet vereist.                               |
| `Mantis Telegram Desktop Proof`   | PR-label van maintainer (`mantis: telegram-visible-proof`) plus PR-opmerking, of handmatige activering | Agentisch systeemeigen Telegram Desktop-bewijs vóór/na. Geeft de PR, baseline-/candidate-refs en instructies van de maintainer door aan Codex, dat de Crabbox Telegram Desktop-bewijslane voor echte gebruikers voor beide refs uitvoert en een PR-bewijstabel met 2 kolommen plaatst.                                                              |
| `Mantis Web UI Chat Proof`        | PR-opmerking of handmatige activering                                                              | Voert het gerichte Playwright-bewijs voor OpenClaw Control UI-chat uit op de candidate, verifieert dat de browser via de nagebootste Gateway verzendt, legt schermafbeeldings-/videoartefacten vast en plaatst PR-bewijsmateriaal. Deze lane is alleen bewijs voor webchat, niet voor WinUI/systeemeigen apps of willekeurig visueel bewijs.                           |

`Mantis Discord Status Reactions` en `Mantis Telegram Live` accepteren beide
`baseline_ref`/`candidate_ref` (of `baseline=`/`candidate=` in een PR-opmerking)
en valideren dat de opgeloste SHA een voorouder van `origin/main`, een
releasetag (`v*`) of de head van een open PR is voordat ze worden uitgevoerd met
referenties die geheimen bevatten.

Opmerkingstriggers vanuit een PR met schrijf-/beheer-/beheerderstoegang:

```text
@openclaw-mantis discord status reactions
@openclaw-mantis discord status reactions baseline=origin/main candidate=HEAD
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
@openclaw-mantis web ui chat
@openclaw-mantis web-ui-chat candidate=HEAD
```

Telegram-opmerkingstriggers gebruiken standaard de head-SHA van de PR als candidate en
`telegram-status-command` als scenario; ze accepteren `provider=aws|hetzner` en
`lease=<cbx_...>` om een specifieke Crabbox-provider of een vooraf opgewarmde
desktop te kiezen. `Mantis Telegram Desktop Proof` reageert alleen op een PR-opmerking wanneer
de PR al het label `mantis: telegram-visible-proof` draagt.

Opmerkingstriggers voor web-UI-chat gebruiken standaard de head-SHA van de PR als candidate. Ze voeren
het chatbewijs voor Control UI met een nagebootste Gateway uit en publiceren browserartefacten; gebruik
normaal Playwright-/browserbewijs, schermafbeeldingen van maintainers, Crabbox of lokale
artefacten voor andere webpagina's en systeemeigen app-oppervlakken.

ClawSweeper kan een scenario ook rechtstreeks activeren:

```text
@clawsweeper mantis discord discord-status-reactions-tool-only
```

## Machines en geheimen

De standaardwaarden voor lokale CLI-Crabbox zijn `--provider hetzner --class beast`; overschrijf deze
met `--provider`, `--class`/`--machine-class` of
`OPENCLAW_MANTIS_CRABBOX_PROVIDER` / `OPENCLAW_MANTIS_CRABBOX_CLASS`. GitHub-
workflows overschrijven doorgaans beide (bijvoorbeeld `--class standard` en de
providerkeuze-invoer `aws`/`hetzner` van de Slack-workflow). Als een provider te
traag of niet beschikbaar is, voeg deze dan toe achter dezelfde Crabbox-interface in plaats van
een fallback hard te coderen.

VM-baseline: Linux met een desktopgeschikte Chrome/Chromium, CDP-toegang, VNC/
noVNC, Node 22.22.3+, 24.15+ of 25.9+ en pnpm, een OpenClaw-checkout en
uitgaande toegang tot het doeltransport, GitHub, modelproviders en de
referentiebroker.

Namen van referenties en omgevingsvariabelen die in Mantis-opdrachten en -workflows worden gebruikt:

- `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- Lokale `qa mantis run --credential-source env` vereist ook
  `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`, `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
  en `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID`. GitHub-workflows gebruiken normaal
  `--credential-source convex` en de brokerreferenties hieronder in plaats van onbewerkte
  Discord-bottokens.
- `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` voor openbare artefactuploads
- `OPENCLAW_QA_CONVEX_SITE_URL`, `OPENCLAW_QA_CONVEX_SECRET_CI`
- `OPENAI_API_KEY` (of de voor Telegram Desktop-bewijs specifieke
  `OPENCLAW_MANTIS_AGENT_OPENAI_API_KEY`)
- `CRABBOX_COORDINATOR` / `CRABBOX_COORDINATOR_TOKEN` (workflows accepteren ook
  `OPENCLAW_QA_MANTIS_CRABBOX_COORDINATOR` / `_TOKEN` als fallback en wijzen
  deze toe aan de gewone namen voordat Crabbox wordt aangeroepen)
- `CRABBOX_ACCESS_CLIENT_ID`, `CRABBOX_ACCESS_CLIENT_SECRET`
- `MANTIS_GITHUB_APP_ID`, `MANTIS_GITHUB_APP_PRIVATE_KEY`

De Mantis-runner mag nooit Discord-/Slack-/Telegram-bottokens,
API-sleutels van providers, browsercookies, inhoud van auth-profielen, VNC-wachtwoorden of
onbewerkte referentiepayloads afdrukken. Als een token in een issue, PR, chat of log lekt,
roteer het dan nadat het vervangende geheim is opgeslagen.

## Uitvoeringsresultaten

Transportscenario's vóór/na onderscheiden deze resultaten, zodat een instabiele
omgeving niet als een productregressie wordt geïnterpreteerd:

- **Bug gereproduceerd**: de baseline mislukte op de manier die het scenario verwacht.
- **Harnessfout**: omgevingsconfiguratie, referenties, transport-API, browser
  of provider mislukte voordat de oracle betekenisvol was.

Browserbewijs met alleen een candidate rapporteert of de candidate slaagde voor de nagebootste
Gateway en zichtbare UI-asserties; het claimt niet dat de baseline is gereproduceerd.

## Een scenario toevoegen

Live transportscenario's worden per transport in TypeScript gedefinieerd (zie
`MANTIS_SCENARIO_CONFIGS` in `extensions/qa-lab/src/mantis/run.runtime.ts` voor
de Discord-vorm vóór/na), niet als een zelfstandige declaratieve bestandsindeling.
Elk scenario heeft het volgende nodig: id en titel, transport, vereiste referenties, beleid voor
baseline-ref, beleid voor candidate-ref, OpenClaw-configuratiepatch, configuratie-/stimulusstappen,
verwachte oracle voor baseline en candidate, doelen voor visuele vastlegging, time-outbudget
en opschoonstappen.

Gericht browserbewijs met alleen een candidate kan een speciale deterministische E2E-test
en workflow gebruiken. Houd het bereik expliciet, valideer de candidate-ref vóór
uitvoering, isoleer publicatie met geheimen en gebruik hetzelfde
bewijsmanifestcontract.

Geef de voorkeur aan kleine, getypeerde oracles boven vision-controles: Discord-reactiestatus of
berichtreferenties, Slack-thread-`ts`/reactie-API-status, e-mailbericht-id's
en headers. Gebruik browserschermafbeeldingen wanneer de UI het enige betrouwbare waarneembare resultaat is
en houd vision-controles aanvullend op een platform-API-oracle waar die bestaat.

Na Discord, Slack en Telegram kan dezelfde runnervorm worden uitgebreid naar WhatsApp
(QR-aanmelding, heridentificatie, bezorging, media, reacties) en Matrix
(versleutelde kamers, thread-/antwoordrelaties, hervatting na herstart); geen van beide is
nog geïmplementeerd.

## Open vragen

- Welke Discord-bot moet de driver zijn en welke de SUT wanneer de bestaande Mantis-
  bot opnieuw wordt gebruikt?
- Hoelang moet GitHub Mantis-artefacten voor pull requests bewaren?
- Wanneer moet ClawSweeper automatisch een Mantis-scenario aanbevelen in plaats van
  op een opdracht van een maintainer te wachten?
- Moeten schermafbeeldingen vóór het uploaden voor openbare pull requests worden geredigeerd of bijgesneden?
