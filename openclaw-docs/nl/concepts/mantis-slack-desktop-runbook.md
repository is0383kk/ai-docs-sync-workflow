---
read_when:
    - Mantis Slack-desktop-QA uitvoeren vanuit GitHub of lokaal
    - Trage Mantis-runs in de Slack-desktopapp debuggen
    - Bron-, voorgehydrateerde of warm-lease-modus kiezen
    - Screenshot- en videobewijs aan een PR toevoegen
summary: 'Runbook voor operators voor Mantis Slack-desktop-QA: GitHub-dispatch, lokale CLI, warme VNC-leases, hydratatiemodi, timinginterpretatie, artefacten en foutafhandeling.'
title: Runbook voor Mantis Slack-desktopapp
x-i18n:
    generated_at: "2026-07-27T05:48:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack-desktop-QA is het real-UI-traject voor bugs van het Slack-type waarvoor een
Linux-desktop, VNC-herstel, Slack Web, een echte OpenClaw-Gateway, schermafbeeldingen,
video's en een PR-bewijscommentaar nodig zijn. Gebruik het wanneer unittests of het headless
live Slack-traject de bug niet kunnen aantonen.

## Opslagmodel

Mantis gebruikt drie opslaglagen:

- **Provider-image** - beheerd door Crabbox, opgeslagen in het account van de cloudprovider.
  Bevat machinecapaciteiten (Chrome/Chromium, ffmpeg, scrot,
  Node/corepack/pnpm, native buildtools) en lege cachedirectories.
- **Status van warme lease** - beheerd door de huidige operatorsessie. Kan een
  aangemeld browserprofiel, `/var/cache/crabbox/pnpm` en een voorbereide broncode-checkout
  bevatten zolang de lease actief is.
- **Mantis-artefacten** - beheerd door de OpenClaw-run. Bevinden zich onder
  `.artifacts/qa-e2e/mantis/...`; GitHub Actions uploadt ze en de Mantis
  GitHub App plaatst inline bewijs als commentaar bij de PR.

Neem nooit geheimen, browsercookies, Slack-aanmeldstatus, repository-checkouts,
`node_modules` of `dist/` op in een provider-image.

## GitHub-dispatch

Voer de workflow uit vanuit `main`:

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

`candidate_ref` is beperkt omdat de workflow live-inloggegevens gebruikt: deze
moet verwijzen naar de huidige afstamming van `main`, een releasetag of de head van een open PR in
`openclaw/openclaw`.

De workflow produceert:

- geüpload artefact `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- inline PR-commentaar van de Mantis GitHub App
- `slack-desktop-smoke.png`, `slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`, `slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`, `mantis-slack-desktop-smoke-report.md`
- externe logboeken: `slack-desktop-command.log`, `openclaw-gateway.log`, `chrome.log`, `ffmpeg.log`

Het PR-commentaar wordt ter plaatse bijgewerkt via de verborgen markering `<!-- mantis-slack-desktop-smoke -->`.

## Lokale CLI

Koude broncodeverificatie:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

Behoud de VM voor VNC-herstel:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Open VNC:

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

Hergebruik een warme lease:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

Gebruik `--hydrate-mode prehydrated` alleen wanneer de hergebruikte externe werkruimte al
`node_modules` en een gebouwde `dist/` bevat; anders stopt Mantis uit voorzorg.

Toon de native Slack-goedkeuringsinterface aan:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` sluit `--gateway-setup` wederzijds uit. Hiermee worden
de optionele scenario's `slack-approval-exec-native` en `slack-approval-plugin-native`
uitgevoerd, tenzij je een expliciet goedkeuringscontrolepunt `--scenario` doorgeeft; andere
Slack-scenario's worden geweigerd voordat de VM wordt gestart. De Slack-QA-runner schrijft
elk JSON-bestand met controlepunten op basis van het echte Slack API-bericht dat deze heeft waargenomen, waarna
de externe watcher dat bericht rendert naar
`approval-checkpoints/<scenario>-pending.png` en
`approval-checkpoints/<scenario>-resolved.png`. De run mislukt als een
JSON-bestand met controlepunten, berichtbewijs, bevestigings-JSON of gerenderde schermafbeelding ontbreekt
of leeg is.

Koude GitHub Actions-leases hebben geen Slack Web-cookies, waardoor hun browseropname
op het Slack-aanmeldscherm kan uitkomen. Vertrouw voor bewijs met goedkeuringscontrolepunten op de
gerenderde controlepuntafbeeldingen en Slack-QA-artefacten in plaats van op
`slack-desktop-smoke.png`. Gebruik alleen een behouden warme lease met een handmatig
aangemeld Slack Web-profiel wanneer de browserschermafbeelding zelf
Slack Web moet tonen.

## Hydratiemodi

| Modus          | Gebruiken wanneer                                  | Extern gedrag                                                                       | Afweging                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | Normaal PR-bewijs, koude machines, CI        | Voert `pnpm install --frozen-lockfile --prefer-offline` en `pnpm build` uit binnen de VM | Traagst, sterkste bewijs vanuit broncode-checkout                 |
| `prehydrated` | Je bewust een hergebruikte lease hebt voorbereid | Vereist bestaande `node_modules` en `dist/`; slaat installatie/build over                     | Snel, maar alleen geldig voor door de operator beheerde warme leases |

GitHub Actions bereidt de checkout van de kandidaat altijd voor vóór de VM-run. De
pnpm-store wordt gecachet op basis van het besturingssysteem, de Node-versie en het lockbestand. De `source`-run van de VM
hergebruikt ook `/var/cache/crabbox/pnpm` wanneer dit aanwezig is.

## Interpretatie van tijdmetingen

`mantis-slack-desktop-smoke-report.md` bevat tijdmetingen per fase:

- `crabbox.warmup` - opstarten van de cloudprovider, gereedheid van desktop/browser, SSH.
- `crabbox.inspect` - opzoeken van leasemetagegevens.
- `credentials.prepare` - verkrijgen van een Convex-lease voor inloggegevens.
- `crabbox.remote_run` - synchronisatie, starten van de browser, installatie/build van OpenClaw of
  hydratievalidatie, opstarten van de Gateway, schermafbeelding en video-opname.
- `artifacts.copy` - rsync terug vanaf de VM.

`crabbox.remote_run` kan `accepted` tonen wanneer Crabbox een externe status anders dan nul
retourneert, maar Mantis metagegevens heeft gekopieerd die bewijzen dat de installatie van de OpenClaw-Gateway
is voltooid of dat de Slack-QA-opdracht zelf met succes is afgesloten. Behandel
`accepted` als geslaagd-met-uitleg, niet als een mislukt scenario.

Als een run traag is:

- Opwarming domineert: bouw vooraf of promoveer een betere provider-image voor Crabbox.
- `remote_run` domineert in `source`: gebruik een warme lease, verbeter het hergebruik van de pnpm-store
  of verplaats machinevereisten naar de provider-image.
- `remote_run` domineert in `prehydrated`: de externe werkruimte was niet
  daadwerkelijk gereed, of het instellen van de Gateway/browser/Slack verloopt traag.
- Het kopiëren van artefacten domineert: controleer de videogrootte en de inhoud van de artefactdirectory.

## Bewijschecklist

Goed PR-commentaar toont:

- scenario-id en kandidaat-SHA
- URL van de GitHub Actions-run en artefact-URL
- inline schermafbeelding van het goedkeuringscontrolepunt, of een Slack Web-schermafbeelding van een
  aangemelde warme lease
- inline geanimeerd voorbeeld wanneer beschikbaar
- links naar de volledige MP4 en ingekorte MP4
- status geslaagd/mislukt en het tijdsoverzicht van het rapport

Commit geen schermafbeeldingen of video's naar de repository. Bewaar ze in GitHub
Actions-artefacten of het PR-commentaar.

## Foutafhandeling

Als de workflow vóór de VM-run mislukt, controleer dan eerst de Actions-job.
Typische oorzaken: niet-vertrouwde `candidate_ref`, ontbrekende omgevingsgeheimen of een
mislukte installatie/build van de kandidaat.

Als de VM-run mislukt maar de schermafbeeldingen zijn teruggekopieerd, controleer dan:

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

Als de lease tijdens de run is behouden, open je VNC met de opdracht `crabbox vnc ...`
uit het rapport en stop je daarna de lease wanneer je klaar bent:

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

Als de Slack-aanmelding is verlopen, herstel je deze via VNC op een behouden lease en voer je de run opnieuw uit met
`--lease-id`. Neem dat browserprofiel niet op in een provider-image.

## Gerelateerd

- [QA-overzicht](/nl/concepts/qa-e2e-automation)
- [Slack-kanaal](/nl/channels/slack)
- [Testen](/nl/help/testing)
