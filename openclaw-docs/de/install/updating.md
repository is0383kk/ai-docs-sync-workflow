---
read_when:
    - OpenClaw aktualisieren
    - Nach einem Update funktioniert etwas nicht mehr
summary: OpenClaw sicher aktualisieren (globale Installation oder Quellcode) sowie Rollback-Strategie
title: Aktualisierung
x-i18n:
    generated_at: "2026-07-26T17:51:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83444d56e0aa34f47830610538b0c3012903abb812bfe0fffb8163a5db9ac2db
    source_path: install/updating.md
    workflow: 16
---

Halten Sie OpenClaw auf dem neuesten Stand.

Informationen zum Ersetzen von Images für Docker, Podman und Kubernetes finden Sie unter
[Container-Images aktualisieren](/de/install/docker#upgrading-container-images). Das
Gateway führt vor der Bereitschaft startsichere Aktualisierungsarbeiten aus und wird beendet, wenn der eingebundene
Zustand manuell repariert werden muss.

## Empfohlen: `openclaw update`

Erkennt Ihren Installationstyp (npm, pnpm, Bun oder git), ruft die neueste Version ab, führt `openclaw doctor` aus und startet das Gateway neu.

```bash
openclaw update
```

Wechseln Sie den Kanal oder wählen Sie eine bestimmte Version:

```bash
openclaw update --channel beta
openclaw update --channel extended-stable
openclaw update --channel dev
openclaw update --dry-run   # Vorschau ohne Anwendung
```

`openclaw update` verfügt über kein Flag `--verbose` (das Installationsprogramm hingegen schon). Verwenden Sie zur Diagnose
`--dry-run` für eine Vorschau der geplanten Aktionen, `--json` für strukturierte Ergebnisse oder
`openclaw update status --json`, um den Kanal- und Verfügbarkeitsstatus zu prüfen.

`--channel beta` bevorzugt das npm-Dist-Tag „beta“, greift jedoch auf „stable/latest“ zurück,
wenn das Beta-Tag fehlt oder seine Version älter als die neueste stabile
Version ist. Verwenden Sie stattdessen `--tag beta` für eine einmalige Paketaktualisierung, die an das unverarbeitete npm-
Beta-Dist-Tag gebunden ist.

`--channel extended-stable` betrifft nur das Paket, und die Installation erfolgt weiterhin
ausschließlich im Vordergrund. OpenClaw liest den öffentlichen npm-Selektor `extended-stable`,
überprüft das exakt ausgewählte Paket und installiert genau diese Version. Fehlende
oder inkonsistente Registry-Daten führen zum sicheren Abbruch; es erfolgt niemals ein Rückgriff auf `latest`.
Wenn die ausgewählte Version älter als die installierte Version ist, gilt weiterhin
die normale Bestätigung für ein Downgrade. Die CLI speichert den Kanal nach einer
erfolgreichen Aktualisierung des Kerns; ein direktes `npm install -g openclaw@extended-stable`
aktualisiert `update.channel` nicht.
Nach dem Austausch des Kerns werden geeignete offizielle npm-Plugins mit einer leeren/standardmäßigen oder
`latest`-Vorgabe auf genau diese Kernversion angeglichen. Exakte Fixierungen und explizite
Tags ungleich `latest`, Drittanbieter-Plugins und Quellen außerhalb von npm bleiben unverändert.
Kataloginstallationen, die von aktuellen OpenClaw-Versionen erstellt wurden, behalten diese standardmäßige
Vorgabe bei. Ältere Datensätze, die nur eine exakte Version enthalten, bleiben fixiert, da
OpenClaw eine alte automatische Fixierung nicht sicher von einer Benutzerfixierung unterscheiden kann; führen Sie
`openclaw plugins update @openclaw/name` einmal im Kanal „extended-stable“ aus,
um dieses Plugin wieder für die Nachverfolgung der exakten Kernversion zu aktivieren.

`--channel dev` stellt einen persistenten, fortlaufend aktualisierten GitHub-Checkout von `main` bereit. Für eine einmalige
Paketaktualisierung ordnet `--tag main` die Paketspezifikation `github:openclaw/openclaw#main` zu
und installiert sie direkt über den gewünschten Paketmanager (npm/pnpm/bun).

Bei verwalteten Plugins ist eine fehlende Beta-Version eine Warnung und kein Fehler: Die
Kernaktualisierung kann dennoch erfolgreich abgeschlossen werden, während ein Plugin auf seine gespeicherte
standardmäßige/neueste Version zurückgreift.

Informationen zur Bedeutung der Kanäle finden Sie unter [Release-Kanäle](/de/install/development-channels).

## Zwischen npm- und git-Installationen wechseln

Verwenden Sie Kanäle, um den Installationstyp zu ändern. Das Aktualisierungsprogramm behält Ihren Zustand, Ihre Konfiguration,
Ihre Anmeldedaten und Ihren Arbeitsbereich in `~/.openclaw` bei; es ändert lediglich, welche OpenClaw-
Codeinstallation von CLI und Gateway verwendet wird.

```bash
# npm-Paketinstallation -> bearbeitbarer git-Checkout
openclaw update --channel dev

# git-Checkout -> npm-Paketinstallation
openclaw update --channel stable
```

Zeigen Sie zunächst eine Vorschau des Installationsmoduswechsels an:

```bash
openclaw update --channel dev --dry-run
openclaw update --channel stable --dry-run
```

`dev` stellt einen git-Checkout sicher, erstellt ihn und installiert die globale CLI aus diesem
Checkout. Die Kanäle `stable`, `extended-stable` und `beta` verwenden Paketinstallationen.
„Extended-stable“ wird bei einem git-Checkout abgelehnt, ohne ihn zu verändern oder
zu konvertieren. Wenn das Gateway bereits installiert ist, aktualisiert `openclaw update`
die Dienstmetadaten und startet es neu, sofern Sie nicht `--no-restart` übergeben.

Bei Paketinstallationen mit einem verwalteten Gateway-Dienst zielt `openclaw update`
auf das von diesem Dienst verwendete Paketstammverzeichnis. Wenn der Shell-Befehl `openclaw`
aus einer anderen Installation stammt, gibt das Aktualisierungsprogramm beide Stammverzeichnisse und den Node-Pfad
des verwalteten Dienstes aus und prüft diese Node-Version anhand der Anforderung
`engines.node` der Zielversion, bevor das Paket ersetzt wird.

## Server mit Quellcode-Checkout (Referenzskript)

Teams, die ein Gateway auf einem Server direkt aus einem git-Checkout ausführen, können es
innerhalb dieses Checkouts mit `scripts/update-gateway.sh` aktualisieren. Dieses Skript dient als Referenz
für eine effiziente Aktualisierung eines Quellcode-Servers: Es stellt nachverfolgte Build-Ausgaben wieder her, die
`pnpm build` überschreibt, bricht bei allen anderen lokalen Änderungen sicher ab, führt für
`main` einen Fast-Forward durch (oder rebasiert einen lokalen Server-Branch auf `origin/main`), installiert
Abhängigkeiten, führt einen sauberen Build aus und startet das Gateway neu.

```bash
ssh you@server 'cd /path/to/openclaw && scripts/update-gateway.sh'
```

Überschreiben Sie den Neustart für benutzerdefinierte Diensteinheiten oder überspringen Sie ihn vollständig:

```bash
OPENCLAW_UPDATE_RESTART_CMD='systemctl --user restart openclaw-gateway.service' scripts/update-gateway.sh
OPENCLAW_UPDATE_RESTART_CMD='' scripts/update-gateway.sh
```

Für eine einfache Einzelbenutzerinstallation aus dem Quellcode empfiehlt sich stattdessen `openclaw update --channel dev` —
es verwaltet den Checkout, den Build und den Neustart des Gateways für Sie.

## Alternative: Installationsprogramm erneut ausführen

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Fügen Sie `--no-onboard` hinzu, um das Onboarding zu überspringen. Um einen bestimmten Installationstyp zu erzwingen, übergeben Sie
`--install-method git --no-onboard` oder `--install-method npm --no-onboard`.

Wenn `openclaw update` nach der Installationsphase des npm-Pakets fehlschlägt, führen Sie stattdessen das
Installationsprogramm erneut aus. Es ruft das Aktualisierungsprogramm nicht auf, sondern führt die globale Paketinstallation
direkt aus und kann eine teilweise aktualisierte npm-Installation wiederherstellen.

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm
```

Fixieren Sie die Wiederherstellung mit `--version` auf eine bestimmte Version oder ein bestimmtes Dist-Tag:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version <version-or-dist-tag>
```

## Alternative: manuell mit npm, pnpm oder bun

```bash
npm i -g openclaw@latest
```

Bevorzugen Sie `openclaw update` für überwachte Installationen: Damit lässt sich der Paketaustausch
mit dem laufenden Gateway-Dienst koordinieren. Wenn Sie eine überwachte Installation manuell
aktualisieren, stoppen Sie zuerst das verwaltete Gateway. Paketmanager ersetzen Dateien direkt,
und ein laufendes Gateway könnte andernfalls versuchen, während des Austauschs Kern- oder Plugin-Dateien
zu laden. Starten Sie das Gateway nach Abschluss des Paketmanagers neu, damit es
die neue Installation übernimmt.

Wenn bei einer root-eigenen systemweiten Linux-Installation `openclaw update` mit
`EACCES` fehlschlägt, stellen Sie sie mit dem systemeigenen npm wieder her und lassen Sie das Gateway während des
manuellen Austauschs gestoppt. Verwenden Sie dieselben Profil-Flags bzw. dieselbe Umgebung, die Sie normalerweise für
dieses Gateway verwenden. Ersetzen Sie `/usr/bin/npm` durch das systemeigene npm, dem das
root-eigene globale Präfix auf Ihrem Host gehört:

```bash
openclaw gateway stop
sudo /usr/bin/npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

Überprüfen Sie anschließend:

```bash
openclaw --version
curl -fsS http://127.0.0.1:18789/readyz
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

Wenn `openclaw update` eine globale npm-Installation verwaltet, installiert es das Ziel
zunächst in einem temporären npm-Präfix. Das Kandidatenpaket validiert während
`preinstall` die Node-Version des Hosts; erst danach überprüft OpenClaw den paketierten
Bestand `dist` und tauscht den sauberen Paketbaum in das tatsächliche globale Präfix ein. Eine
paketierte Abschlussprüfung wird im erwarteten Bestand ausgelassen und erst entfernt,
nachdem `preinstall` erfolgreich abgeschlossen wurde, sodass übersprungene Lebenszyklusskripte ebenfalls vor dem
Austausch zum Abbruch führen. Bei npm 12 und neuer genehmigt das Aktualisierungsprogramm nur den
Lebenszyklus des OpenClaw-Kandidaten; Skripte transitiver Abhängigkeiten bleiben blockiert. Dadurch wird vermieden,
dass npm ein neues Paket über veraltete Dateien des alten Pakets legt. Wenn der Installationsbefehl
fehlschlägt, versucht OpenClaw es einmal mit `--omit=optional` erneut, was auf Hosts hilfreich ist,
auf denen native optionale Abhängigkeiten nicht kompiliert werden können.

Von OpenClaw verwaltete npm-Aktualisierungs- und Plugin-Aktualisierungsbefehle deaktivieren für den untergeordneten
npm-Prozess außerdem die Lieferkettenquarantäne `min-release-age` von npm (oder den älteren Konfigurationsschlüssel `before`).
Diese Richtlinie dient dem allgemeinen Schutz, doch eine
explizite OpenClaw-Aktualisierung bedeutet: „Die ausgewählte Version jetzt installieren.“

```bash
pnpm add -g openclaw@latest
```

Wenn OpenClaw 2026.7.1 mit pnpm 11 installiert wurde, führen Sie diesen manuellen Befehl einmal aus. Diese
Version stammt aus der Zeit vor dem isolierten Layout für globale Pakete von pnpm 11, sodass ihr Aktualisierungsprogramm
eine andere npm-Installation mit der laufenden CLI verwechseln kann. Spätere Versionen behalten
die pnpm-Zuordnung bei und folgen bei Aktualisierungen dem Stammverzeichnis des Ersatzpakets. Sie
verwenden außerdem das vom zuständigen Manager gemeldete globale Binärverzeichnis und halten vor
jeder Änderung an, wenn der verfügbare pnpm-Befehl ein anderes globales Stammverzeichnis oder eine andere Hauptversion meldet,
oder wenn das aufrufende Paket verwaist oder dort nicht die einzige aktive OpenClaw-
Installation ist.

Wenn OpenClaw eine globale pnpm-11-Installationsgruppe mit einem anderen Paket teilt, hält das
automatische Aktualisierungsprogramm an, bevor es die Gruppe ändert. Aktualisieren Sie die ursprüngliche,
durch Kommas getrennte Gruppe manuell, damit die zugehörigen Pakete und die Build-Richtlinie
intakt bleiben.

```bash
bun add -g openclaw@latest
```

### Erweiterte Themen zur npm-Installation

<AccordionGroup>
  <Accordion title="Schreibgeschützter Paketbaum">
    OpenClaw behandelt paketierte globale Installationen zur Laufzeit als schreibgeschützt, selbst wenn das globale Paketverzeichnis für den aktuellen Benutzer beschreibbar ist. Installationen von Plugin-Paketen befinden sich in OpenClaw-eigenen npm-/git-Stammverzeichnissen unter dem Benutzerkonfigurationsverzeichnis, und der Start des Gateways verändert den OpenClaw-Paketbaum nicht.

    Einige Linux-npm-Konfigurationen installieren globale Pakete unter root-eigenen Verzeichnissen wie `/usr/lib/node_modules/openclaw`. OpenClaw unterstützt dieses Layout, da Befehle zum Installieren und Aktualisieren von Plugins außerhalb dieses globalen Paketverzeichnisses schreiben.

  </Accordion>
  <Accordion title="Gehärtete systemd-Einheiten">
    Gewähren Sie OpenClaw Schreibzugriff auf seine Konfigurations-/Zustandsstammverzeichnisse, damit explizite Plugin-Installationen, Plugin-Aktualisierungen und Bereinigungen durch den Doctor ihre Änderungen dauerhaft speichern können:

    ```ini
    ReadWritePaths=/var/lib/openclaw /home/openclaw/.openclaw /tmp
    ```

  </Accordion>
  <Accordion title="Vorabprüfung des Speicherplatzes">
    Vor Paketaktualisierungen und expliziten Plugin-Installationen versucht OpenClaw nach bestem Wissen, den Speicherplatz des Zielvolumes zu prüfen. Bei wenig Speicherplatz wird eine Warnung mit dem geprüften Pfad ausgegeben, die Aktualisierung wird jedoch nicht blockiert, da sich Dateisystemkontingente, Snapshots und Netzwerkvolumes nach der Prüfung ändern können. Maßgeblich bleiben die tatsächliche Installation durch den Paketmanager und die Überprüfung nach der Installation.
  </Accordion>
</AccordionGroup>

## Automatisches Aktualisierungsprogramm

Standardmäßig deaktiviert. Aktivieren Sie es in `~/.openclaw/openclaw.json`:

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
    },
  },
}
```

| Kanal             | Verhalten                                                                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `stable`          | Wird nach einer integrierten Verzögerung mit deterministischem Jitter für eine gestaffelte Einführung angewendet.                            |
| `extended-stable` | Prüft beim Start und alle 24 Stunden auf einen schreibgeschützten Aktualisierungshinweis, wenn `checkOnStart` aktiviert ist. Wird niemals automatisch angewendet. |
| `beta`            | Prüft in einem integrierten Intervall und wendet die Aktualisierung sofort an.                                                               |
| `dev`             | Keine automatische Anwendung. Verwenden Sie `openclaw update` manuell.                                                                      |

Der Gateway protokolliert beim Start außerdem einen Aktualisierungshinweis (deaktivierbar mit
`update.checkOnStart: false`). Gespeicherte Extended-Stable-Auswahlen verwenden diesen
schreibgeschützten Hinweispfad und das bestehende 24-Stunden-Hinweisintervall, führen jedoch niemals
eine automatische Installation, Übergabe, einen Neustart, eine Stable-Verzögerung bzw. einen Stable-Jitter oder Beta-Abfragen aus.
Legen Sie für ein Downgrade oder die Wiederherstellung nach einem Vorfall `OPENCLAW_NO_AUTO_UPDATE=1` in der Gateway-Umgebung fest, um automatische Anwendungen zu blockieren, selbst wenn `update.auto.enabled` konfiguriert ist. Aktualisierungshinweise beim Start können weiterhin ausgeführt werden, sofern nicht auch `update.checkOnStart` deaktiviert ist.

Über die Live-Gateway-Steuerungsebene angeforderte Paketmanager-Aktualisierungen
(`update.run`) ersetzen den Paketbaum nicht innerhalb des laufenden Gateway-
Prozesses. Bei Installationen als verwalteter Dienst startet der Gateway eine entkoppelte Übergabe,
wird beendet und überlässt dem normalen CLI-Pfad `openclaw update --yes --json` das Stoppen des
Dienstes, das Ersetzen des Pakets, das Aktualisieren der Dienstmetadaten, den Neustart, die Überprüfung
der Gateway-Version und -Erreichbarkeit sowie, wenn möglich, die Wiederherstellung eines installierten, aber nicht geladenen
macOS-LaunchAgent. Wenn der Gateway diese Übergabe nicht sicher durchführen kann,
meldet `update.run` stattdessen einen sicheren Shell-Befehl, anstatt den Paketmanager
innerhalb des Prozesses auszuführen.

Die Aktualisierungskarte in der Seitenleiste der Control UI zeigt **Gateway aktualisieren**, wenn sie
diesen `update.run`-Ablauf direkt startet. Dies gilt für die browserbasierte Control UI, entfernte
Gateways und manuell verwaltete lokale Gateways.

In der signierten macOS-App ändert ein lokaler, der App zugeordneter Gateway diese Karte in
**Mac-App + Gateway aktualisieren**. Sparkle aktualisiert zuerst die App; nach dem erneuten Start
führt die App `openclaw update --tag <app-version> --json` aus, startet ihren Gateway neu
und überprüft dessen Funktionsfähigkeit in einem Fortschrittsfenster nach Art der Einrichtung. Das Fenster wird nur angezeigt,
wenn dieser verwaltete Gateway aktualisiert, repariert oder installiert werden muss; reine App-Aktualisierungen starten
direkt wieder in die App. Fehlerdetails bleiben zusammen mit den Aktionen „Erneut versuchen“, [Aktualisierungsanleitung](/de/install/updating) und
[Discord](https://discord.gg/clawd) sichtbar. Die App verwendet diesen koordinierten
Pfad niemals für einen entfernten oder extern verwalteten Gateway, führt niemals ein Downgrade eines neueren
Gateway durch und überschreibt niemals eine `extended-stable`-Kanalfixierung.

Nach einer erfolgreichen Aktualisierung stellt die App ein einmaliges Willkommensereignis für die
zuletzt verwendete direkte Sitzung auf oberster Ebene mit einer echten Benutzer-/Kanalinteraktion in die Warteschlange. Cron-Ausführungen,
Heartbeats und ausschließlich im Hintergrund erfolgende Sitzungsaktualisierungen ändern diese Auswahl nicht. Im
Remote-Modus aktualisiert die App nur die Laufzeitumgebung ihres lokalen Mac-Nodes und sendet das Ereignis
nur, wenn der verbundene entfernte Gateway mindestens so neu wie die App ist.

## Nach der Aktualisierung

<Steps>

### Doctor ausführen

```bash
openclaw doctor
```

Migriert die Konfiguration, prüft DM-Richtlinien und kontrolliert den Zustand des Gateway. Details: [Doctor](/de/gateway/doctor)

### Gateway neu starten

```bash
openclaw gateway restart
```

### Überprüfen

```bash
openclaw health
```

</Steps>

## Rollback

Ein Rollback umfasst zwei Ebenen:

1. Älteren OpenClaw-Code neu installieren und dabei den aktuellen Zustand beibehalten.
2. Den Zustand vor der Aktualisierung nur wiederherstellen, wenn der ältere Code eine migrierte
   Konfiguration oder Datenbank nicht verwenden kann.

Beginnen Sie mit einem reinen Code-Rollback. Bei der Wiederherstellung des Zustands gehen Änderungen verloren, die nach
der Sicherung vorgenommen wurden.

### Vor der Aktualisierung: verifizierte Sicherung erstellen

`openclaw update` bewahrt automatisch eine Konfigurationskopie vor der Aktualisierung auf, erstellt jedoch
keinen vollständigen Wiederherstellungspunkt für den Zustand. Erstellen Sie vor einer bedeutenden Aktualisierung
explizit einen:

```bash
mkdir -p ~/Backups/openclaw
openclaw backup create --output ~/Backups/openclaw --verify
```

Das Archivmanifest verzeichnet die OpenClaw-Version und die im Backup enthaltenen
Quellpfade. Das Archiv kann Anmeldedaten, Authentifizierungsprofile und Kanalzustände
enthalten. Speichern Sie es daher mit Berechtigungen ausschließlich für den Eigentümer und demselben Schutz wie das
Live-Zustandsverzeichnis. Unter [Backup](/de/cli/backup) finden Sie Informationen zu enthaltenen und absichtlich
ausgelassenen Dateien.

Für einen bitgenauen Wiederherstellungspunkt, der flüchtige Artefakte enthält, die im
portablen Archiv ausgelassen werden, stoppen Sie den Gateway und verwenden Sie einen von Ihrer Plattform
bereitgestellten Dateisystem-, Volume- oder VM-Snapshot.

### Paketinstallation zurücksetzen

Listen Sie die veröffentlichten Versionen auf, zeigen Sie dann eine Vorschau an und installieren Sie die als funktionsfähig bekannte Version:

```bash
npm view openclaw versions --json
openclaw update --tag <known-good-version> --dry-run
openclaw update --tag <known-good-version>
```

`openclaw update --tag` wird gegenüber einer direkten Installation über den Paketmanager bevorzugt. Der Befehl
erkennt das Downgrade, fordert eine Bestätigung an, führt die Konvergenz verwalteter Plugins
und Kompatibilitätsprüfungen für das installierte Ziel durch, aktualisiert die Dienstmetadaten,
startet den Gateway neu und überprüft die ausgeführte Version. Wenn der gespeicherte
Kanal `extended-stable` ist, verwenden Sie
`--channel stable --tag <known-good-version>`, da exakte einmalige Tags nicht
mit dem Selektor `extended-stable` kombiniert werden können.

Paketaktualisierungen bereiten den Kandidaten vor und überprüfen ihn vor der Aktivierung. Wenn der
Dateisystemaustausch oder das Ersetzen des Befehls-Shims fehlschlägt, stellt OpenClaw automatisch das alte
Paket wieder her. Wenn nach einem erfolgreichen Austausch später eine Gateway-Zustandsprüfung fehlschlägt,
werden die vorherige Version und Anweisungen für einen manuellen Rollback gemeldet, anstatt
das Paket erneut automatisch zu ersetzen.

Wenn der CLI-Aktualisierungspfad nicht verfügbar ist, verwenden Sie denselben Paketmanager und
Installationsumfang, denen der aktuelle Gateway zugeordnet ist:

```bash
openclaw gateway stop
npm i -g openclaw@<known-good-version>
openclaw gateway install --force
openclaw gateway restart
```

Ersetzen Sie `npm` durch `pnpm` oder `bun`, wenn dieser Manager für die Installation zuständig ist. Verhindern Sie während
der Wiederherstellung nach einem Vorfall, dass ein aktivierter automatischer Updater sofort eine
neuere Version anwendet, indem Sie `OPENCLAW_NO_AUTO_UPDATE=1` in der Gateway-Umgebung festlegen.

### Quellcode-Checkout zurücksetzen

Verwenden Sie einen sauberen Checkout und wählen Sie einen als funktionsfähig bekannten Tag oder Commit aus:

```bash
git fetch --all --tags
git checkout --detach <known-good-tag-or-commit>
pnpm install && pnpm build
openclaw gateway restart
```

So kehren Sie zur neuesten Version zurück: `git checkout main && git pull`.

Der Updater setzt einen Git-Checkout automatisch auf seinen vorherigen Branch und
SHA zurück, wenn die Abhängigkeitsinstallation, der Build, der UI-Build oder Doctor nach dem Start einer Git-
Aktualisierung fehlschlägt. Ein manueller Checkout ist weiterhin erforderlich, wenn Sie absichtlich
einen älteren Commit auswählen.

### Downgrade über die SQLite-Sitzungsmigration hinweg

Bevor Sie eine ältere dateibasierte OpenClaw-Version starten, verwenden Sie die aktuelle CLI, um
archivierte ältere Transkriptartefakte wiederherzustellen:

```bash
openclaw gateway stop
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Dadurch werden keine SQLite-Daten gelöscht. Sitzungen, die nach der SQLite-Migration erstellt wurden,
existieren nur in SQLite und werden in der älteren Laufzeitumgebung nicht angezeigt. Siehe
[Downgrade nach der SQLite-Sitzungsmigration](/de/cli/doctor#downgrading-after-session-sqlite-migration).

### Zustand nur bei Bedarf wiederherstellen

Wenn der ältere Code eine neuere Konfiguration oder ein neueres Datenbankschema nicht lesen kann, stoppen Sie den
Gateway und stellen Sie den verifizierten Dateisystem-, Volume- oder VM-Snapshot von vor der Aktualisierung wieder her.
Bewahren Sie den aktuellen Zustand vor der Wiederherstellung separat auf, da dadurch
Änderungen entfernt werden, die nach dem Snapshot vorgenommen wurden.

Umfassende `openclaw backup create`-Archive unterstützen die Erstellung und Verifizierung, jedoch
keine direkte Aktivierung des gesamten Archivs. Extrahieren Sie ein umfassendes Archiv in ein Staging-
Verzeichnis und verwenden Sie dessen `manifest.json`-Zuordnung von Quelle zu Archiv für eine Offline-
Wiederherstellung. `openclaw backup sqlite restore` schreibt ebenfalls eine verifizierte Datenbank
in ein neues Ziel; die Aktivierung dieses Ziels bleibt ein expliziter Offline-Schritt durch die Administration.

### Rollback überprüfen

```bash
openclaw --version
openclaw health
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

## Wenn Sie nicht weiterkommen

- Führen Sie `openclaw doctor` erneut aus und lesen Sie die Ausgabe sorgfältig.
- Bei `openclaw update --channel dev` in Quellcode-Checkouts richtet der Updater `pnpm` bei Bedarf automatisch ein. Wenn ein pnpm-/corepack-Bootstrap-Fehler angezeigt wird, installieren Sie `pnpm` manuell (oder aktivieren Sie `corepack` erneut) und führen Sie die Aktualisierung erneut aus.
- Prüfen Sie: [Fehlerbehebung](/de/gateway/troubleshooting)
- Fragen Sie in Discord: [https://discord.gg/clawd](https://discord.gg/clawd)

## Verwandte Themen

- [Installationsübersicht](/de/install): alle Installationsmethoden.
- [Doctor](/de/gateway/doctor): Zustandsprüfungen nach Aktualisierungen.
- [Migration](/de/install/migrating): Anleitungen zur Migration zwischen Hauptversionen.
