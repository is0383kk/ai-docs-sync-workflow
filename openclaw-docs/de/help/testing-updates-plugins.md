---
read_when:
    - Ändern des Verhaltens von OpenClaw bei Updates, Doctor, Paketannahme oder Plugin-Installation
    - Vorbereiten oder Genehmigen eines Release Candidates
    - Debugging von Paketaktualisierungen, Bereinigung von Plugin-Abhängigkeiten oder Regressionen bei der Plugin-Installation
sidebarTitle: Update and plugin tests
summary: Wie OpenClaw Aktualisierungspfade, Paketmigrationen und das Installations- und Aktualisierungsverhalten von Plugins validiert
title: 'Tests: Updates und Plugins'
x-i18n:
    generated_at: "2026-07-26T18:25:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96a11fe42472f758d4fd1cc568486e301f7460982fdb547cab8b39de04a8dabe
    source_path: help/testing-updates-plugins.md
    workflow: 16
---

Checkliste für die Validierung von Updates und Plugins: Nachweisen, dass das installierbare Paket
echten Benutzerzustand aktualisieren, veralteten Legacy-Zustand über `doctor` reparieren und weiterhin
Plugins aus jeder unterstützten Quelle installieren, laden, aktualisieren und deinstallieren kann.

Die umfassendere Übersicht der Test-Runner finden Sie unter [Tests](/de/help/testing). Informationen zu Schlüsseln für Live-Provider
und Testsuiten mit Netzwerkzugriff finden Sie unter [Live-Tests](/de/help/testing-live).

## Was wir schützen

- Ein Paket-Tarball ist vollständig, besitzt eine gültige `dist/postinstall-inventory.json`
  und hängt nicht von entpackten Repository-Dateien ab.
- Benutzer können von einem älteren veröffentlichten Paket zum Kandidatenpaket wechseln,
  ohne Konfiguration, Agenten, Sitzungen, Arbeitsbereiche, Plugin-Zulassungslisten oder
  Kanalkonfiguration zu verlieren.
- `openclaw doctor --fix --non-interactive` ist für die Bereinigungs- und Reparaturpfade
  von Legacy-Zuständen zuständig. Beim Start sollten keine verborgenen Kompatibilitätsmigrationen für veralteten
  Plugin-Zustand hinzukommen.
- Plugin-Installationen funktionieren aus lokalen Verzeichnissen, Git-Repositorys, npm-Paketen und über den
  ClawHub-Registrierungspfad.
- npm-Abhängigkeiten von Plugins werden in einem verwalteten npm-Projekt pro Plugin installiert,
  vor der Vertrauensgewährung geprüft und bei der Deinstallation des Plugins über `npm uninstall`
  entfernt, damit hochgezogene Abhängigkeiten nicht zurückbleiben.
- Ein Plugin-Update ist eine wirkungslose Operation, wenn sich nichts geändert hat: Installationsdatensätze, aufgelöste
  Quelle, Layout der installierten Abhängigkeiten und Aktivierungsstatus bleiben unverändert.

## Lokaler Nachweis während der Entwicklung

Beginnen Sie gezielt:

```bash
pnpm changed:lanes --json
pnpm check:changed
pnpm test:changed
```

Führen Sie bei Änderungen an Plugin-Installation, -Deinstallation, Abhängigkeiten oder Paketbestand zusätzlich
die gezielten Tests aus, die die bearbeitete Schnittstelle abdecken:

```bash
pnpm test src/plugins/uninstall.test.ts src/infra/package-dist-inventory.test.ts test/scripts/package-acceptance-workflow.test.ts
```

Bevor ein Paket-Docker-Testlauf einen Tarball verwendet, weisen Sie das Paketartefakt nach:

```bash
pnpm release:check
```

`release:check` führt Prüfungen auf Abweichungen bei Konfiguration, Dokumentation und API aus (Konfigurationsschema, Basisstand der Konfigurationsdokumentation,
API-Vertragsmanifest und Exporte des Plugin-SDK, Plugin-Versionen/-Bestand),
schreibt den Paket-Distributionsbestand, führt `npm pack --dry-run` aus, lehnt unzulässige
gepackte Dateien ab, installiert den Tarball in einem temporären Präfix, führt Postinstall aus und
unterzieht die Einstiegspunkte gebündelter Kanäle einem Smoke-Test.

## Docker-Testläufe

Die Docker-Testläufe bilden den Nachweis auf Produktebene. Sie installieren oder aktualisieren ein echtes
Paket in Linux-Containern und prüfen das Verhalten über CLI-Befehle,
Gateway-Start, HTTP-Prüfungen, RPC-Status und Dateisystemzustand.

Verwenden Sie während der Iteration gezielte Testläufe:

```bash
pnpm test:docker:plugins
pnpm test:docker:plugin-lifecycle-matrix
pnpm test:docker:plugin-update
pnpm test:docker:upgrade-survivor
pnpm test:docker:published-upgrade-survivor
pnpm test:docker:update-restart-auth
pnpm test:docker:update-migration
```

Wichtige Testläufe:

- `test:docker:plugins` deckt Smoke-Tests für Plugin-Installationen, Installationen aus lokalen Ordnern,
  Überspringverhalten bei Aktualisierungen lokaler Ordner, lokale Ordner mit vorinstallierten
  Abhängigkeiten, Installationen von `file:`-Paketen, Git-Installationen mit CLI-Ausführung, Git-
  Aktualisierungen beweglicher Referenzen, Installationen aus der npm-Registry mit hochgezogenen transitiven
  Abhängigkeiten, wirkungslose npm-Updates, die Ablehnung fehlerhafter npm-Paketmetadaten,
  Installationen lokaler ClawHub-Fixtures und wirkungslose Updates, das Aktualisierungsverhalten des Marketplace
  sowie Aktivierung/Inspektion des Claude-Bundles ab. Setzen Sie `OPENCLAW_PLUGINS_E2E_CLAWHUB=0`, um
  den ClawHub-Block hermetisch/offline zu halten.
- `test:docker:plugin-lifecycle-matrix` installiert das Kandidatenpaket in einem leeren
  Container und führt ein npm-Plugin durch Installation, Inspektion, Deaktivierung, Aktivierung,
  explizites Upgrade, explizites Downgrade und Deinstallation nach dem Löschen des Plugin-
  Codes. Für jede Phase werden RSS- und CPU-Metriken protokolliert.
- `test:docker:plugin-update` validiert, dass ein unverändertes installiertes Plugin
  während `openclaw plugins update` weder neu installiert wird noch Installationsmetadaten verliert.
- `test:docker:upgrade-survivor` installiert den Kandidaten-Tarball über einer verunreinigten
  Fixture eines alten Benutzers, führt die Paketaktualisierung sowie Doctor nicht interaktiv aus, startet anschließend
  ein Loopback-Gateway und prüft die Beibehaltung des Zustands.
- `test:docker:published-upgrade-survivor` installiert zunächst einen veröffentlichten Basisstand,
  konfiguriert ihn über ein eingebettetes `openclaw config set`-Rezept, aktualisiert ihn auf den
  Kandidaten-Tarball, führt Doctor aus, prüft die Legacy-Bereinigung, startet das Gateway und
  prüft `/healthz`, `/readyz` sowie den RPC-Status.
- `test:docker:update-restart-auth` installiert das Kandidatenpaket, startet ein
  verwaltetes Gateway mit Token-Authentifizierung, entfernt für
  `openclaw update --yes --json` die Gateway-Authentifizierungsumgebung des Aufrufers und verlangt, dass der Aktualisierungsbefehl des Kandidaten
  das Gateway vor den regulären Prüfungen neu startet.
- `test:docker:update-migration` ist der bereinigungsintensive Testlauf für veröffentlichte Updates. Er
  beginnt mit einem konfigurierten Benutzerzustand nach Art von Discord/Telegram, führt den Doctor des Basisstands aus,
  damit sich konfigurierte Plugin-Abhängigkeiten materialisieren können, legt für ein konfiguriertes verpacktes Plugin
  veraltete Überreste von Plugin-Abhängigkeiten an, aktualisiert auf
  den Kandidaten-Tarball und verlangt, dass Doctor nach der Aktualisierung die veralteten
  Abhängigkeitsstammverzeichnisse entfernt.

Nützliche Varianten für das Überleben veröffentlichter Upgrades:

```bash
OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@2026.4.23 \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=versioned-runtime-deps \
pnpm test:docker:published-upgrade-survivor

OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@latest \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=bootstrap-persona \
pnpm test:docker:published-upgrade-survivor
```

Verfügbare Szenarien: `base`, `acpx-openclaw-tools-bridge`, `feishu-channel`,
`bootstrap-persona`, `channel-post-core-restore`, `plugin-deps-cleanup`,
`configured-plugin-installs`, `stale-source-plugin-shadow`, `tilde-log-path`
und `versioned-runtime-deps`. Bei aggregierten Ausführungen wird `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues`
(Alias `far-reaching`) auf alle Szenarien erweitert, einschließlich der
Installationsmigration für konfigurierte Plugins.

Die vollständige Aktualisierungsmigration ist bewusst von der vollständigen Release-CI getrennt. Verwenden Sie den
manuellen `Update Migration`-Workflow, wenn die Release-Frage lautet: „Kann jede
seit 2026.4.23 veröffentlichte stabile Version auf diesen Kandidaten aktualisiert werden und
Überreste von Plugin-Abhängigkeiten bereinigen?“:

```bash
gh workflow run update-migration.yml \
  --ref main \
  -f workflow_ref=main \
  -f package_ref=main \
  -f baselines=all-since-2026.4.23 \
  -f scenarios=plugin-deps-cleanup
```

## Paketabnahme

Die Paketabnahme ist die GitHub-native Paketprüfung. Sie löst ein Kandidatenpaket
in einen `package-under-test`-Tarball auf, zeichnet Version und SHA-256 auf und
führt anschließend wiederverwendbare Docker-E2E-Testläufe für genau diesen Tarball aus. Die Referenz des Workflow-Testgerüsts
ist von der Referenz der Paketquelle getrennt, sodass die aktuelle Testlogik ältere
vertrauenswürdige Releases validieren kann.

Kandidatenquellen:

- `source=npm`: `openclaw@extended-stable`, `openclaw@beta`,
  `openclaw@latest` oder eine exakt veröffentlichte Version validieren.
- `source=ref`: Einen vertrauenswürdigen Branch, Tag oder Commit mit dem ausgewählten aktuellen
  Testgerüst packen.
- `source=url`: Einen öffentlichen HTTPS-Tarball mit erforderlichem `package_sha256` validieren.
  Dieser Pfad lehnt URL-Anmeldedaten, vom Standard abweichende HTTPS-Ports, private/interne
  Hostnamen oder DNS-/IP-Ergebnisse, IP-Adressräume für besondere Zwecke und unsichere Weiterleitungen ab.
- `source=trusted-url`: Einen HTTPS-Tarball mit erforderlichem
  `package_sha256` und `trusted_source_id` anhand der von den Maintainern verwalteten Richtlinie
  in `.github/package-trusted-sources.json` validieren. Verwenden Sie dies für unternehmensinterne/private
  Mirrors, anstatt `source=url` durch einen eingabebasierten Schalter zum Zulassen privater Quellen abzuschwächen.
  Wenn Bearer-Authentifizierung durch die Richtlinie konfiguriert ist, verwendet sie das feste
  `OPENCLAW_TRUSTED_PACKAGE_TOKEN`-Secret.
- `source=artifact`: Einen von einer anderen Actions-Ausführung hochgeladenen Tarball wiederverwenden.

Die vollständige Release-Validierung verwendet standardmäßig `source=artifact`, erstellt aus dem
aufgelösten Release-SHA. Übergeben Sie für den Nachweis nach der Veröffentlichung
`package_acceptance_package_spec=openclaw@YYYY.M.PATCH`, damit dieselbe Upgrade-Matrix
stattdessen auf das ausgelieferte npm-Paket zielt.

Release-Prüfungen rufen die Paketabnahme mit der Paket-/Update-/Neustart-/Plugin-Gruppe auf:

```text
doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape
```

Wenn der Release-Dauertest aktiviert ist (für `release_profile=stable` und
`full` zwingend aktiviert), übergeben sie außerdem:

```text
published_upgrade_survivor_baselines=last-stable-4 2026.4.23 2026.5.2 2026.4.15
published_upgrade_survivor_scenarios=reported-issues
telegram_mode=mock-openai
```

Dadurch werden Paketmigration, Wechsel des Aktualisierungskanals, Toleranz gegenüber beschädigten verwalteten Plugins,
Bereinigung veralteter Plugin-Abhängigkeiten, Offline-Plugin-Abdeckung, Verhalten bei Plugin-
Aktualisierungen und Telegram-Paket-QA für dasselbe aufgelöste Artefakt ausgeführt, ohne
dass die standardmäßige Release-Paketprüfung jede veröffentlichte Version durchlaufen muss.

`last-stable-4` wird in die vier neuesten stabilen, auf npm veröffentlichten OpenClaw-
Releases aufgelöst. Die Release-Paketabnahme legt `2026.4.23` als erste Kompatibilitätsgrenze
für Plugin-Aktualisierungen, `2026.5.2` als Grenze für Änderungen an der Plugin-Architektur und
`2026.4.15` als älteren Basisstand aus der Reihe 2026.4.1x für veröffentlichte Updates fest; der Resolver
entfernt doppelte feste Versionen, die bereits zu den neuesten vier gehören. Verwenden Sie für eine vollständige
Abdeckung der Migration veröffentlichter Updates `all-since-2026.4.23` im separaten Workflow für die Aktualisierungs-
migration anstelle der vollständigen Release-CI. `release-history` bleibt
für manuelle breitere Stichproben verfügbar, wenn Sie zusätzlich den Legacy-Anker
vor dem Stichtag einbeziehen möchten.

Wenn mehrere Basisstände für das Überleben veröffentlichter Upgrades ausgewählt sind, teilt der wiederverwendbare
Docker-Workflow jeden Basisstand in einen eigenen gezielten Runner-Job auf. Jeder
Basisstand-Shard führt weiterhin die ausgewählte Szenariengruppe aus, Protokolle und Artefakte bleiben jedoch
je Basisstand getrennt, und die Gesamtdauer wird durch den langsamsten Shard begrenzt statt durch einen großen
seriellen Job.

Führen Sie ein Paketprofil manuell aus, wenn Sie einen Kandidaten vor dem Release validieren:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=package \
  -f published_upgrade_survivor_baselines="last-stable-4 2026.4.23 2026.5.2 2026.4.15" \
  -f published_upgrade_survivor_scenarios=reported-issues \
  -f telegram_mode=mock-openai
```

Setzen Sie für einen veröffentlichten Extended-Stable-Canary
`package_spec=openclaw@extended-stable`. Die Paketabnahme löst diesen
Selektor in einen exakten Tarball auf, bevor die Docker-Testläufe beginnen.

Verwenden Sie `suite_profile=product`, wenn die Release-Frage MCP-Kanäle,
Cron-/Subagent-Bereinigung, OpenAI-Websuche oder OpenWebUI umfasst. Verwenden Sie `suite_profile=full`
nur, wenn Sie eine vollständige Docker-Abdeckung des Release-Pfads benötigen.

## Release-Standard

Für Release-Kandidaten besteht der standardmäßige Nachweisstapel aus:

1. `pnpm check:changed` und `pnpm test:changed` für Regressionen auf Quellcodeebene.
2. `pnpm release:check` für die Integrität des Paketartefakts.
3. Dem `package`-Profil der Paketabnahme oder den benutzerdefinierten Paket-
   Testläufen der Release-Prüfung für Installations-, Aktualisierungs-, Neustart- und Plugin-Verträge.
4. Betriebssystemübergreifenden Release-Prüfungen für betriebssystemspezifisches Installations-, Onboarding- und Plattform-
   verhalten.
5. Live-Testsuiten nur, wenn die geänderte Oberfläche das Verhalten eines Providers oder gehosteten Dienstes
   betrifft.

Auf Maintainer-Rechnern sollten umfassende Prüfungen und Docker-/Paketnachweise auf Produktebene
in Testbox ausgeführt werden, sofern nicht ausdrücklich ein lokaler Nachweis erfolgt.

## Legacy-Kompatibilität

Die Kompatibilitätstoleranz ist eng begrenzt und zeitlich befristet:

- Pakete bis einschließlich `2026.4.25`, darunter `2026.4.25-beta.*`, dürfen
  in der Paketabnahme bereits ausgelieferte Lücken in den Paketmetadaten tolerieren.
- Das veröffentlichte `2026.4.26`-Paket darf bei bereits ausgelieferten lokalen Stempeldateien
  für Build-Metadaten Warnungen ausgeben.
- Spätere Pakete müssen moderne Verträge erfüllen. Dieselben Lücken führen zu Fehlern, statt
  nur zu warnen oder übersprungen zu werden.

Fügen Sie für diese alten Formen keine neuen Startmigrationen hinzu. Fügen Sie eine Doctor-
Reparatur hinzu oder erweitern Sie sie und weisen Sie sie anschließend mit `upgrade-survivor`, `published-upgrade-survivor` oder
`update-restart-auth` nach, wenn der Aktualisierungsbefehl für den Neustart zuständig ist.

## Abdeckung hinzufügen

Bei Änderungen am Update- oder Plugin-Verhalten muss die Abdeckung auf der niedrigsten Ebene ergänzt werden, die aus dem richtigen Grund fehlschlagen kann:

- Reine Pfad- oder Metadatenlogik: Unit-Test neben dem Quellcode.
- Paketbestand oder Verhalten gepackter Dateien: `package-dist-inventory` oder Test des Tarball-Prüfers.
- CLI-Installations-/Update-Verhalten: Assertion oder Fixture in einer Docker-Lane.
- Migrationsverhalten veröffentlichter Releases: Szenario `published-upgrade-survivor`.
- Update-gesteuertes Neustartverhalten: `update-restart-auth`.
- Verhalten der Registry-/Paketquelle: Fixture `test:docker:plugins` oder ClawHub-Fixture-Server.
- Verhalten des Abhängigkeitslayouts oder der Bereinigung: Sowohl die Laufzeitausführung als auch die Dateisystemgrenze prüfen. npm-Abhängigkeiten können innerhalb des verwalteten npm-Projekts des Plugins nach oben verlagert werden. Daher müssen Tests nachweisen, dass dieses Projekt durchsucht/bereinigt wird, statt anzunehmen, dass nur der Plugin-paketlokale `node_modules`-Baum betroffen ist.

Neue Docker-Fixtures müssen standardmäßig hermetisch bleiben. Verwenden Sie lokale Fixture-Registrys und gefälschte Pakete, sofern nicht gerade das Verhalten einer Live-Registry Gegenstand des Tests ist.

## Fehleranalyse

Beginnen Sie mit der Artefaktidentität:

- Package-Acceptance-Zusammenfassung für `resolve_package`: Quelle, Version, SHA-256 und Artefaktname.
- Docker-Artefakte: `.artifacts/docker-tests/**/summary.json`, `failures.json`, Lane-Protokolle und Befehle zur erneuten Ausführung.
- Zusammenfassung der Upgrade-Überlebensprüfung: `.artifacts/upgrade-survivor/summary.json`, einschließlich Basisversion, Kandidatenversion, Szenario, Phasenlaufzeiten und Abdeckung der Konfigurationsrezepte.

Führen Sie vorzugsweise exakt die fehlgeschlagene Lane mit demselben Paketartefakt erneut aus, statt den gesamten übergeordneten Release-Ablauf zu wiederholen.
