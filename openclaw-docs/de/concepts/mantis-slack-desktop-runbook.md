---
read_when:
    - Mantis-Slack-Desktop-QA über GitHub oder lokal ausführen
    - Langsame Mantis-Ausführungen in der Slack-Desktop-App debuggen
    - Auswahl des Quell-, vorhydrierten oder Warm-Lease-Modus
    - Screenshot- und Videonachweise in einem PR veröffentlichen
summary: 'Betriebshandbuch für die Desktop-QA von Mantis Slack: GitHub-Auslösung, lokale CLI, vorbereitete VNC-Leases, Hydratationsmodi, Interpretation der Zeitmessungen, Artefakte und Fehlerbehandlung.'
title: Mantis-Runbook für Slack Desktop
x-i18n:
    generated_at: "2026-07-26T18:24:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack Desktop-QA ist der Real-UI-Testpfad für Fehler der Slack-Klasse, die einen
Linux-Desktop, VNC-Wiederherstellung, Slack Web, ein echtes OpenClaw-Gateway, Screenshots,
Videos und einen PR-Evidenzkommentar erfordern. Verwenden Sie ihn, wenn Unit-Tests oder der Headless-
Slack-Live-Testpfad den Fehler nicht nachweisen können.

## Speichermodell

Mantis verwendet drei Speicherebenen:

- **Provider-Image** – im Besitz von Crabbox und im Cloud-Provider-Konto gespeichert.
  Enthält Maschinenfunktionen (Chrome/Chromium, ffmpeg, scrot,
  Node/corepack/pnpm, native Build-Werkzeuge) und leere Cache-Verzeichnisse.
- **Status der warmen Lease** – im Besitz der aktuellen Operatorsitzung. Kann ein
  angemeldetes Browserprofil, `/var/cache/crabbox/pnpm` und einen vorbereiteten Quellcode-
  Checkout enthalten, solange die Lease aktiv ist.
- **Mantis-Artefakte** – im Besitz des OpenClaw-Laufs. Befinden sich unter
  `.artifacts/qa-e2e/mantis/...`; GitHub Actions lädt sie hoch, und die Mantis
  GitHub App kommentiert Inline-Evidenz im PR.

Betten Sie niemals Secrets, Browser-Cookies, Slack-Anmeldestatus, Repository-Checkouts,
`node_modules` oder `dist/` in ein Provider-Image ein.

## GitHub-Auslösung

Führen Sie den Workflow über `main` aus:

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

`candidate_ref` ist eingeschränkt, da der Workflow Live-Anmeldedaten verwendet: Er
muss auf die aktuelle Abstammung von `main`, ein Release-Tag oder den Head eines offenen PRs in
`openclaw/openclaw` aufgelöst werden.

Der Workflow erzeugt:

- hochgeladenes Artefakt `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- Inline-PR-Kommentar der Mantis GitHub App
- `slack-desktop-smoke.png`, `slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`, `slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`, `mantis-slack-desktop-smoke-report.md`
- Remote-Protokolle: `slack-desktop-command.log`, `openclaw-gateway.log`, `chrome.log`, `ffmpeg.log`

Der PR-Kommentar wird über die verborgene Markierung `<!-- mantis-slack-desktop-smoke -->` direkt aktualisiert.

## Lokale CLI

Nachweis mit kaltem Quellcode:

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

VM für die VNC-Wiederherstellung beibehalten:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

VNC öffnen:

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

Eine warme Lease wiederverwenden:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

Verwenden Sie `--hydrate-mode prehydrated` nur, wenn der wiederverwendete Remote-Arbeitsbereich bereits
über `node_modules` und ein gebautes `dist/` verfügt; andernfalls verweigert Mantis die Ausführung.

Native Slack-Genehmigungs-UI nachweisen:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` und `--gateway-setup` schließen sich gegenseitig aus. Dabei werden
die optionalen Szenarien `slack-approval-exec-native` und `slack-approval-plugin-native`
ausgeführt, sofern Sie nicht ausdrücklich einen Genehmigungs-Checkpoint `--scenario` übergeben; andere
Slack-Szenarien werden abgelehnt, bevor die VM startet. Der Slack-QA-Runner schreibt
jede Checkpoint-JSON-Datei aus der tatsächlich beobachteten Slack-API-Nachricht; anschließend
rendert der Remote-Watcher diese Nachricht in
`approval-checkpoints/<scenario>-pending.png` und
`approval-checkpoints/<scenario>-resolved.png`. Der Lauf schlägt fehl, wenn eine
Checkpoint-JSON-Datei, Nachrichtenevidenz, Bestätigungs-JSON-Datei oder ein gerenderter Screenshot fehlt
oder leer ist.

Kalte GitHub-Actions-Leases besitzen keine Slack-Web-Cookies, daher kann ihre Browseraufnahme
auf dem Slack-Anmeldebildschirm landen. Verlassen Sie sich für den Nachweis von Genehmigungs-Checkpoints auf die
gerenderten Checkpoint-Bilder und Slack-QA-Artefakte statt auf
`slack-desktop-smoke.png`. Verwenden Sie nur dann eine beibehaltene warme Lease mit einem manuell
angemeldeten Slack-Web-Profil, wenn der Browser-Screenshot selbst
Slack Web zeigen muss.

## Hydratisierungsmodi

| Modus          | Verwenden, wenn                                  | Remote-Verhalten                                                                       | Nachteil                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | Normaler PR-Nachweis, kalte Maschinen, CI        | Führt `pnpm install --frozen-lockfile --prefer-offline` und `pnpm build` innerhalb der VM aus | Am langsamsten, stärkster Nachweis anhand des Quellcode-Checkouts                 |
| `prehydrated` | Sie haben absichtlich eine wiederverwendete Lease vorbereitet | Erfordert vorhandenes `node_modules` und `dist/`; überspringt Installation/Build                     | Schnell, aber nur für operatorgesteuerte warme Leases gültig |

GitHub Actions bereitet den Kandidaten-Checkout immer vor dem VM-Lauf vor. Sein
pnpm-Store wird nach Betriebssystem, Node-Version und Lockdatei zwischengespeichert. Der `source`-Lauf der VM
verwendet außerdem `/var/cache/crabbox/pnpm` wieder, sofern vorhanden.

## Interpretation der Zeitmessung

`mantis-slack-desktop-smoke-report.md` enthält Phasenzeiten:

- `crabbox.warmup` – Start des Cloud-Providers, Desktop-/Browserbereitschaft, SSH.
- `crabbox.inspect` – Abruf der Lease-Metadaten.
- `credentials.prepare` – Abruf der Convex-Anmeldedaten-Lease.
- `crabbox.remote_run` – Synchronisierung, Browserstart, Installation/Build von OpenClaw oder
  Hydratisierungsvalidierung, Gateway-Start, Screenshot- und Videoaufnahme.
- `artifacts.copy` – Rücksynchronisierung von der VM per rsync.

`crabbox.remote_run` kann `accepted` anzeigen, wenn Crabbox einen von null verschiedenen
Remote-Status zurückgibt, Mantis jedoch Metadaten kopiert hat, die belegen, dass entweder die Einrichtung des OpenClaw-Gateways
abgeschlossen wurde oder der Slack-QA-Befehl selbst erfolgreich beendet wurde. Behandeln Sie
`accepted` als bestanden mit Erläuterung, nicht als fehlgeschlagenes Szenario.

Wenn ein Lauf langsam ist:

- Warmup dominiert: Erstellen Sie vorab ein besseres Crabbox-Provider-Image oder stufen Sie eines hoch.
- `remote_run` dominiert in `source`: Verwenden Sie eine warme Lease, verbessern Sie die Wiederverwendung des pnpm-Stores
  oder verschieben Sie Maschinenvoraussetzungen in das Provider-Image.
- `remote_run` dominiert in `prehydrated`: Der Remote-Arbeitsbereich war nicht
  tatsächlich bereit, oder die Einrichtung von Gateway, Browser oder Slack ist langsam.
- Das Kopieren von Artefakten dominiert: Prüfen Sie die Videogröße und den Inhalt des Artefaktverzeichnisses.

## Evidenz-Checkliste

Ein guter PR-Kommentar zeigt:

- Szenario-ID und Kandidaten-SHA
- URL des GitHub-Actions-Laufs und Artefakt-URL
- Inline-Screenshot des Genehmigungs-Checkpoints oder einen Slack-Web-Screenshot aus einer
  angemeldeten warmen Lease
- animierte Inline-Vorschau, sofern verfügbar
- Links zur vollständigen und gekürzten MP4-Datei
- Bestanden-/Fehlgeschlagen-Status und die Zeitübersicht des Berichts

Committen Sie keine Screenshots oder Videos in das Repository. Bewahren Sie sie in
GitHub-Actions-Artefakten oder im PR-Kommentar auf.

## Fehlerbehandlung

Wenn der Workflow vor dem VM-Lauf fehlschlägt, prüfen Sie zuerst den Actions-Job.
Typische Ursachen: nicht vertrauenswürdiges `candidate_ref`, fehlende Umgebungs-Secrets oder ein
Fehler bei Installation/Build des Kandidaten.

Wenn der VM-Lauf fehlschlägt, die Screenshots jedoch zurückkopiert wurden, prüfen Sie:

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

Wenn der Lauf die Lease beibehalten hat, öffnen Sie VNC mit dem Befehl `crabbox vnc ...`
aus dem Bericht und stoppen Sie anschließend die Lease:

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

Wenn die Slack-Anmeldung abgelaufen ist, reparieren Sie sie per VNC auf einer beibehaltenen Lease und führen Sie den Lauf erneut mit
`--lease-id` aus. Betten Sie dieses Browserprofil nicht in ein Provider-Image ein.

## Verwandte Themen

- [QA-Übersicht](/de/concepts/qa-e2e-automation)
- [Slack-Kanal](/de/channels/slack)
- [Tests](/de/help/testing)
