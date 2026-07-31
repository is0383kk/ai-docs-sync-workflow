---
doc-schema-version: 1
read_when:
    - Suche nach Definitionen öffentlicher Release-Kanäle
    - Release-Validierung oder Paketabnahme ausführen
    - Auf der Suche nach Versionsbenennung und Veröffentlichungsrhythmus
summary: Release-Kanäle, Betreiber-Checkliste, Validierungsboxen, Versionsbenennung und Veröffentlichungsrhythmus
title: Veröffentlichungsrichtlinie
x-i18n:
    generated_at: "2026-07-26T18:45:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw stellt vier benutzerseitige Update-Kanäle bereit:

- stable: das hochgestufte reguläre Release auf npm `latest`
- extended-stable: die `.33+`-Wartungslinie des letzten abgeschlossenen Monats auf
  npm `extended-stable`
- beta: Vorabversions-Tags auf npm `beta`
- dev: der sich fortlaufend ändernde Head von `main`

Extended-stable stellt den Gateway, die offiziellen npm-Plugins und die
Docker-Images des letzten Monats bereit, ohne die regulären Selektoren `latest` oder `main` zu verschieben.

Tideclaw-Alpha-Builds sind ein separater interner Vorabversionskanal (npm-Dist-Tag `alpha`), der unter [NPM-Workflow-Eingaben](#npm-workflow-inputs) und [Release-Testboxen](#release-test-boxes) behandelt wird.

## Versionsbenennung

- Monatliche Extended-Stable-Release-Version des Gateways: `YYYY.M.PATCH`, mit `PATCH >= 33`, Git-Tag `vYYYY.M.PATCH`
- Tägliche/reguläre finale Release-Version: `YYYY.M.PATCH`, mit `PATCH < 33`, Git-Tag `vYYYY.M.PATCH`
- Reguläre Fallback-Korrektur-Release-Version: `YYYY.M.PATCH-N`, Git-Tag `vYYYY.M.PATCH-N`
- Beta-Vorabversionsversion: `YYYY.M.PATCH-beta.N`, Git-Tag `vYYYY.M.PATCH-beta.N`
- Alpha-Vorabversionsversion: `YYYY.M.PATCH-alpha.N`, Git-Tag `vYYYY.M.PATCH-alpha.N`
- Monat oder Patch niemals mit führenden Nullen auffüllen
- `PATCH` ist eine fortlaufende monatliche Release-Train-Nummer, kein Kalendertag. Reguläre finale und Beta-Releases setzen den aktuellen Train fort; reine Alpha-Tags verbrauchen oder erhöhen niemals die Beta-/reguläre Patchnummer. Ignorieren Sie daher ältere reine Alpha-Tags mit höheren Patchnummern, wenn Sie einen Beta- oder regulären Train auswählen.
- Alpha-/Nightly-Builds verwenden den nächsten noch nicht veröffentlichten Patch-Train und erhöhen bei wiederholten Builds nur `alpha.N`. Sobald für diesen Patch eine Beta existiert, wechseln neue Alpha-Builds zum darauffolgenden Patch.
- npm-Versionen sind unveränderlich: Löschen, veröffentlichen oder verwenden Sie einen veröffentlichten Tag niemals erneut. Erstellen Sie stattdessen die nächste Vorabversionsnummer oder den nächsten monatlichen Patch.
- `latest` folgt weiterhin der aktuellen regulären/täglichen npm-Linie; `beta` ist das aktuelle Beta-Installationsziel
- `extended-stable` bezeichnet die unterstützte Gateway-Distribution des letzten Monats, beginnend mit Patch `33`; Patch `34` und spätere sind Wartungs-Releases dieser monatlichen Linie
- Reguläre finale und reguläre Korrektur-Releases werden standardmäßig unter npm `beta` veröffentlicht; Release-Verantwortliche können ausdrücklich `latest` als Ziel festlegen oder später einen geprüften Beta-Build hochstufen
- Gateway Extended-Stable veröffentlicht Core, jedes auf npm veröffentlichbare offizielle Plugin
  und die zugehörigen Docker-Images in exakt derselben Version; siehe den dedizierten Workflow unten.
- Jedes reguläre finale Release stellt das npm-Paket, die macOS-App, die signierte eigenständige Android-APK und die signierten Windows-Hub-Installationsprogramme gemeinsam bereit. Beta-Releases validieren und veröffentlichen normalerweise zuerst den npm-/Paketpfad; Build, Signierung, Beglaubigung und Hochstufung nativer Apps bleiben regulären finalen Releases vorbehalten, sofern sie nicht ausdrücklich angefordert werden.

## Release-Takt

- Releases durchlaufen zuerst die Beta-Phase; Stable folgt erst, nachdem die neueste Beta validiert wurde
- Maintainer erstellen Releases normalerweise aus einem von der aktuellen Version `main` erstellten Branch `release/YYYY.M.PATCH`, damit Release-Validierung und Fehlerbehebungen die neue Entwicklung auf `main` nicht blockieren
- Wenn ein Beta-Tag gepusht oder veröffentlicht wurde und korrigiert werden muss, erstellen Maintainer das nächste Tag `-beta.N`, anstatt das alte zu löschen oder neu zu erstellen
- Detaillierte Release-Verfahren, Genehmigungen, Anmeldedaten und Wiederherstellungshinweise sind ausschließlich für Maintainer bestimmt

## Monatliche Extended-Stable-Veröffentlichung des Gateways

Erstellen Sie für den abgeschlossenen Monat `YYYY.M` den Branch `extended-stable/YYYY.M.33` und veröffentlichen Sie
`.33+` von diesem Branch. Tag, Branch, Checkout, Paketversion, Vorprüfung und
Validierung müssen denselben Commit bezeichnen. Vor `.33` muss der geschützte Branch `main`
die finale Version eines späteren Monats unterhalb von Patch `33` enthalten; spätere Wartungs-Patches bleiben
zulässig.

### Kandidaten vorbereiten und stabilisieren

Prüfen Sie den noch nicht auditierten Mainline-Bereich, gleichen Sie private Sicherheitsarbeiten ab, genehmigen Sie
eine begrenzte Backport-Menge und führen Sie einen koordinierten PR zusammen. Pushen Sie nicht direkt auf den kanonischen
Branch.

Setzen Sie auf dem kanonischen Branch `YYYY.M.P`, führen Sie `pnpm release:prep` aus und verlangen Sie
diese Version in jedem veröffentlichbaren offiziellen Plugin. Generieren und committen Sie anhand des genehmigten Verzeichnisses
einen vollständigen Abschnitt `## YYYY.M.P` mit `### Highlights`,
`### Changes` und `### Fixes`; verweisen Sie bei gleichwertigen Backports auf die ursprünglich zusammengeführten PRs `main`.
Die Vorprüfung weist einen fehlenden oder leeren Abschnitt zurück.

Übernehmen Sie die vollständige Docker-Release-Kanaleinheit des aktuellen Main-Branchs: Workflow, Hochstufungslogik,
Richtlinie, gemeinsamen Klassifikator, Tests und Workflow-Validierung. GitHub lädt Tag-
Workflows aus dem getaggten Commit; eine unvollständige Kopie kann nach dem Build fehlschlagen oder
reguläre Aliasse verschieben. Führen Sie gezielte Prüfungen aus.

Fixieren Sie den vollständigen SHA der Branch-Spitze. Prüfen Sie vor dem Tagging die exakten npm-Bytes
vorab und führen Sie die vollständige Release-Validierung für diesen SHA aus:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

Die SHA-Form ist ausschließlich für die Vorprüfung vorgesehen. Führen Sie die Validierung auf dem kanonischen Branch aus; die Veröffentlichung
bindet ihre Workflow-Referenz, den Head-/Ziel-SHA, die Ausführungs-ID und den Versuch. Speichern Sie beide IDs und
den erfolgreichen `run_attempt`; weisen Sie Nachweise zu `release-ci/*` zurück.

Klassifizieren Sie Fehler vor der Bearbeitung:

- Produkt: Führen Sie einen weiteren genehmigten Backport-PR zusammen.
- Werkzeuge für das fixierte Ziel: Übernehmen Sie nur die kleinste Kompatibilitätskorrektur als Backport, die
  das alte Produkt unverändert testet.
- Provider, Genehmigung, Runner oder Dienst: Lassen Sie den Kandidaten unverändert und verwenden Sie
  den begrenzten Wiederholungspfad.

Jede Branch-Änderung macht beide Prüfungen ungültig. Sobald sie erfolgreich sind, muss die Spitze weiterhin
`RELEASE_SHA` entsprechen; pushen Sie anschließend das signierte Tag `vYYYY.M.P`. Spätere Änderungen benötigen den nächsten
Patch; verschieben oder löschen Sie das Tag niemals. Sein Push startet `Docker Release`.

### npm-Pakete veröffentlichen

Veröffentlichen Sie jedes auf npm veröffentlichbare offizielle Plugin aus demselben SHA und speichern Sie die
ID der erfolgreichen Ausführung:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

Der Workflow deckt alle `all-publishable`-Pakete einschließlich unveränderter Pakete ab
und überprüft jede exakte Version und jeden Selektor. Wiederholungen verwenden bereits veröffentlichte Versionen erneut.

Veröffentlichen Sie anschließend den vorbereiteten Core-Tarball mit allen drei gespeicherten Ausführungsidentitäten:

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

Fügen Sie ausschließlich für Probeläufe außerhalb der Produktion
`-f bypass_extended_stable_guard=true` zur Vorprüfung und Veröffentlichung hinzu. Dies umgeht
nur die Monatsprüfung, niemals die Prüfungen der kanonischen Referenz, SHA-/Tag-/Versionsgleichheit, Herkunft,
Genehmigung oder Rücklesung. Verwenden Sie dies niemals für die Produktion.

### Überprüfen und wiederherstellen

Führen Sie aus einem separaten sauberen Checkout des aktuellen Branchs `main`, nicht aus dem fixierten Branch, Folgendes aus:

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

Verlangen Sie Signaturen und npm-Herkunftsnachweise für den kanonischen Branch sowie die Bindung von Veröffentlichung,
Vorprüfung und Tarball-Digest an den Release-SHA. Beide Befehle müssen
`YYYY.M.P` zurückgeben. Überprüfen Sie jedes vorbereitete Core-Paket und jedes der `all-publishable`
offiziellen Plugins mit seiner exakten Version und seinem Selektor.

Wenn nur der Root-Selektor fehlschlägt, verwenden Sie den generierten
Reparaturbefehl `npm dist-tag add openclaw@YYYY.M.P extended-stable`, der in
der Workflow-Zusammenfassung ausgegeben wird. Reparieren Sie vorhandene Plugin- oder andere vorbereitete Core-Selektoren
mithilfe genehmigter Werkzeuge mit isolierten Anmeldedaten; die OIDC-Quelle kann sie nicht verändern.
Veröffentlichen Sie eine unveränderliche Version niemals erneut.

Verlangen Sie, dass `Docker Release` die exakten Standard-, Slim-, Browser- und architekturspezifischen
Images in GHCR und Docker Hub einschließlich Beglaubigungen und Plattformversionen überprüft. Es darf ausschließlich
`extended-stable`, `extended-stable-slim` und `extended-stable-browser`
anhand des Digests aktualisieren; reguläre Aliasse bleiben unverändert und ein automatisches Rollback wird abgelehnt.

Führen Sie zur Alias-Reparatur den genehmigungspflichtigen Workflow `Docker Channel Promotion` vom aktuellen
Branch `main` mit dem Tag aus. Er wiederholt die Digest-, Beglaubigungs- und Plattformprüfungen, erlaubt
ein ausdrückliches Rollback und erstellt niemals Images neu.

Slack, Discord und Codex sind die anfänglich dokumentierten Support-Oberflächen, keine
Release-Zulassungsliste: Jedes auf npm veröffentlichbare offizielle Plugin wird ausgeliefert. Ausschließlich die reguläre
Checkliste ist für Beta/`latest`, GitHub Releases, ClawHub, native Apps, Mobilgeräte,
Website und private Dist-Tags zuständig; führen Sie diese Schritte für diesen Gateway-Pfad nicht aus.

## Checkliste für reguläre Release-Verantwortliche

Diese Checkliste bildet den öffentlichen Ablauf des Release-Prozesses ab. Private Anmeldedaten, Signierung, Beglaubigung, Wiederherstellung von Dist-Tags und Details zu Notfall-Rollbacks verbleiben im ausschließlich für Maintainer bestimmten Release-Runbook.

1. Beginnen Sie mit dem aktuellen Branch `main`: Rufen Sie den neuesten Stand ab, bestätigen Sie, dass der Ziel-Commit gepusht wurde, und bestätigen Sie, dass die CI von `main` ausreichend grün ist, um davon einen Branch zu erstellen.
2. Erstellen Sie `release/YYYY.M.PATCH` aus diesem Commit. Backports sind optional; wenden Sie nur die vom Release-Verantwortlichen ausgewählte Menge an. Erhöhen Sie jede erforderliche Versionsangabe, führen Sie `pnpm release:prep` aus, schließen Sie Release-Korrekturen und erforderliche Forward-Ports ab und prüfen Sie `src/plugins/compat/registry.ts` sowie `src/commands/doctor/shared/deprecation-compat.ts`.
3. Fixieren Sie den produktvollständigen Commit vor der Changelog-Änderung als **Code-SHA**. Führen Sie die deterministische Quell-Vorprüfung aus und verwenden Sie anschließend `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH`. Dadurch werden vertrauenswürdige Workflow-Werkzeuge fixiert, während die vollständige Vitest-, Docker-, QA-, Paket- und Leistungsmatrix exakt auf den Code-SHA ausgerichtet ist.
4. Klassifizieren Sie Fehler vor der Bearbeitung. Ein Produkt-/Codefehler erzeugt einen neuen Code-SHA und erfordert eine erfolgreiche vollständige Validierung dieses SHA. Ein Fehler in Workflow, Testumgebung, Anmeldedaten, Genehmigung oder Infrastruktur wird in der jeweils zuständigen Oberfläche behoben und mit demselben Code-SHA erneut ausgeführt.
5. Generieren Sie erst dann, wenn der Code-SHA erfolgreich validiert wurde, den obersten Abschnitt `CHANGELOG.md` aus zusammengeführten PRs und direkten Commits seit dem letzten erreichbaren ausgelieferten Tag. Formulieren Sie Einträge benutzerorientiert und ohne Duplikate. Wenn ein abweichendes ausgeliefertes Tag oder ein späterer Forward-Port bereits veröffentlichte PRs neu zuordnet, übergeben Sie es ausdrücklich als `--shipped-ref`.
6. Committen Sie ausschließlich `CHANGELOG.md`. Dieser Commit ist der **Release-SHA**. Der vollständige Diff vom Code-SHA zum Release-SHA muss exakt `CHANGELOG.md` entsprechen; jeder andere geänderte Pfad setzt das Release auf Schritt 2 zurück.
7. Führen Sie die SHA-fixierte vollständige Release-Validierung für den Release-SHA mit aktivierter Wiederverwendung von Nachweisen aus. Der leichtgewichtige übergeordnete Lauf muss `changelog-only-release-v1` erfassen, auf den erfolgreichen Code-SHA verweisen und darf keine untergeordneten Produkt-Lanes starten. Dadurch werden Produktnachweise wiederverwendet, nicht jedoch Paketbytes.
8. Führen Sie `OpenClaw NPM Release` mit `preflight_only=true` für den Release-SHA bzw. das Tag aus. Speichern Sie `preflight_run_id` des erfolgreichen Laufs. Dadurch werden exakt die Paketbytes erstellt und geprüft, die den finalen Changelog enthalten.
9. Taggen Sie den Release-SHA und führen Sie anschließend das Kandidaten-Hilfsprogramm mit dem erfolgreichen übergeordneten Release-SHA-Validierungslauf und der npm-Vorprüfung aus, anstatt einen der beiden erneut zu starten:

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   Für stabile Releases übergeben Sie außerdem `--windows-node-tag vX.Y.Z`. Das Hilfsprogramm überprüft die Herkunft der Release Notes, die npm-Preflight-Bytes, den Parallels-Installations-/Aktualisierungsnachweis, den Telegram-Paketnachweis und die Plugin-Veröffentlichungspläne und gibt anschließend den Veröffentlichungsbefehl aus.

   `OpenClaw Release Publish` übermittelt die ausgewählten oder alle veröffentlichungsfähigen Plugin-Pakete parallel an npm und dieselbe Gruppe an ClawHub und stuft anschließend das vorbereitete OpenClaw-npm-Preflight-Artefakt mit dem passenden Dist-Tag hoch, sobald die npm-Veröffentlichung der Plugins erfolgreich war. Der Release-Checkout bleibt der Produkt-/Datenstamm, während Planung und abschließende Überprüfung aus dem exakten vertrauenswürdigen Workflow-Quell-Checkout ausgeführt werden, damit ein älterer Release-Commit nicht unbemerkt veraltete Release-Werkzeuge verwenden kann. Bevor ein untergeordneter Veröffentlichungsvorgang startet, rendert und speichert der Workflow den exakten GitHub-Release-Text zwischen. Wenn der vollständige passende Abschnitt `CHANGELOG.md` innerhalb des GitHub-Limits von 125,000 Zeichen und der entsprechenden Sicherheitsobergrenze des Renderers von 125,000 Byte liegt, enthält die Seite genau diesen Abschnitt `## YYYY.M.PATCH` einschließlich seiner Überschrift. Wenn der Quellabschnitt nicht hineinpasst, behält die Seite die exakten gruppierten redaktionellen Hinweise bei und ersetzt den zu großen Beitragsdatensatz durch einen stabilen Link zum vollständigen Datensatz in `CHANGELOG.md`, der an den Tag gebunden ist; unvollständige Datensätze und abgeschnittene Aufzählungspunkte werden niemals veröffentlicht. Der Workflow wählt diesen vollständigen oder kompakten Text aus, bevor `### Release verification` hinzugefügt wird; würde der Nachweisanhang das Limit überschreiten, behält er den kanonischen Text bei und stützt sich stattdessen auf die unveränderlichen angehängten Nachweise. Stabile Releases, die unter `latest` auf npm veröffentlicht werden, werden zum neuesten GitHub-Release, während stabile Wartungsreleases, die unter `beta` auf npm verbleiben, mit GitHub `latest=false` erstellt werden. Der Workflow lädt außerdem die Preflight-Abhängigkeitsnachweise, das vollständige Validierungsmanifest und die Nachweise der Registry-Überprüfung nach der Veröffentlichung zum GitHub-Release hoch, um die Reaktion auf Vorfälle nach dem Release zu unterstützen. Er gibt die IDs der untergeordneten Ausführungen sofort aus, genehmigt automatisch die Release-Umgebungssperren, die das Workflow-Token genehmigen darf, fasst fehlgeschlagene untergeordnete Jobs samt Log-Enden zusammen, erstellt die GitHub-Release-Seite vorab als Entwurf und überträgt Windows- und Android-Artefakte gleichzeitig mit der npm-Veröffentlichung von OpenClaw, schließt die Release-Seite und die Abhängigkeitsnachweise ab, sobald diese Phasen erfolgreich waren, wartet auf ClawHub, wenn OpenClaw auf npm veröffentlicht wird, führt anschließend den Beta-Verifizierer des vertrauenswürdigen Hauptzweigs aus und lädt Nachweise nach der Veröffentlichung für das GitHub-Release, das npm-Paket, die ausgewählten Plugin-npm-Pakete, die ausgewählten ClawHub-Pakete, die IDs der untergeordneten Workflow-Ausführungen und die optionale ID der NPM-Telegram-Ausführung hoch. Der ClawHub-Bootstrap-Verifizierer erfordert den exakten vertrauenswürdigen Workflow-Pfad und SHA des Hauptzweigs, die Producer- und terminalen Ausführungsversuche, den Release-SHA, die angeforderte Paketgruppe, das unveränderliche Tupel des Paketartefakts und das terminale Artefakt des Registry-Rücklesens; eine erfolgreiche ältere Ausführung über eine Release-Referenz wird nicht akzeptiert.

   Führen Sie anschließend die Paketabnahme nach der Veröffentlichung für das veröffentlichte Paket `openclaw@YYYY.M.PATCH-beta.N` oder `openclaw@beta` aus. Wenn ein übertragener oder veröffentlichter Vorabrelease korrigiert werden muss, erstellen Sie die nächste passende Vorabrelease-Nummer; löschen oder überschreiben Sie niemals die alte.

10. Bei einem fehlgeschlagenen Veröffentlichungsversuch bleibt der Release-SHA unverändert, sofern der Fehler nicht einen Produkt- oder Changelog-Defekt belegt. Setzen Sie erfolgreiche unveränderliche untergeordnete Vorgänge und Artefakte fort; erstellen oder veröffentlichen Sie niemals eine bereits erfolgreich veröffentlichte Paketversion erneut.
11. Fahren Sie bei einem stabilen Release erst fort, wenn der geprüfte Beta- oder Release-Kandidat über die erforderlichen Validierungsnachweise verfügt. Die stabile npm-Veröffentlichung läuft ebenfalls über `OpenClaw Release Publish` und verwendet dabei das erfolgreiche Preflight-Artefakt über `preflight_run_id` erneut. Die Bereitschaft für ein stabiles macOS-Release erfordert außerdem die paketierten `.zip`, `.dmg`, `.dSYM.zip` und das aktualisierte `appcast.xml` auf `main`; der macOS-Veröffentlichungsworkflow veröffentlicht den signierten Appcast automatisch unter dem öffentlichen `main`, nachdem die Release-Artefakte überprüft wurden, oder öffnet beziehungsweise aktualisiert einen Appcast-PR, wenn der Branch-Schutz die direkte Übertragung blockiert. Die Bereitschaft des stabilen Windows Hub erfordert die signierten Artefakte `OpenClawCompanion-Setup-x64.exe`, `OpenClawCompanion-Setup-arm64.exe` und `OpenClawCompanion-SHA256SUMS.txt` im OpenClaw-GitHub-Release. Übergeben Sie den exakten signierten Release-Tag `openclaw/openclaw-windows-node` als `windows_node_tag` und seine vom Kandidaten genehmigte Installer-Digest-Zuordnung als `windows_node_installer_digests`; `OpenClaw Release Publish` behält den Release-Entwurf bei, startet `Windows Node Release` und überprüft alle drei Artefakte vor der Veröffentlichung.
12. Führen Sie nach der Veröffentlichung den npm-Verifizierer für die Nachveröffentlichung aus, optional einen eigenständigen Telegram-E2E-Test mit dem veröffentlichten npm-Paket, wenn Sie einen Kanalnachweis nach der Veröffentlichung benötigen, nehmen Sie bei Bedarf die Dist-Tag-Hochstufung vor, überprüfen Sie die generierte GitHub-Release-Seite, führen Sie die Schritte zur Release-Ankündigung aus und schließen Sie anschließend [Abschluss des stabilen Hauptzweigs](#stable-main-closeout) ab, bevor Sie ein stabiles Release als fertig bezeichnen.

## Abschluss des stabilen Hauptzweigs

Die stabile Veröffentlichung ist erst abgeschlossen, wenn `main` den tatsächlich ausgelieferten Release-Zustand enthält.

1. Beginnen Sie mit einem aktuellen `main`. Prüfen Sie `release/YYYY.M.PATCH` dagegen und übertragen Sie echte Korrekturen vorwärts, die in `main` fehlen. Führen Sie nicht blind ausschließlich für das Release bestimmte Kompatibilitäts-, Test- oder Validierungsadapter in das neuere `main` zusammen.
2. Setzen Sie für den normalen Ablauf `main` auf die ausgelieferte stabile Version. Bei einem verspäteten Abschluss kann `main` verwendet werden, nachdem es auf eine spätere stabile OpenClaw-CalVer-Version fortgeschritten ist; stufen Sie einen bereits begonnenen Release-Zyklus nicht allein zum Abschluss des vorherigen Releases zurück. Der Validator verlangt weiterhin den exakten ausgelieferten Changelog-Abschnitt und Appcast-Eintrag und zeichnet die tatsächliche Version und den SHA von `main` auf. Führen Sie nach jeder Änderung der Stammversion `pnpm release:prep` und anschließend `pnpm deps:shrinkwrap:generate` aus.
3. Sorgen Sie dafür, dass der Abschnitt `## YYYY.M.PATCH` von `CHANGELOG.md` auf `main` exakt mit dem getaggten Release-Branch übereinstimmt. Nehmen Sie die stabile Aktualisierung von `appcast.xml` auf, wenn das Mac-Release eine veröffentlicht hat.
4. Fügen Sie `main` weder `YYYY.M.PATCH+1` noch eine Beta-Version oder einen leeren zukünftigen Changelog-Abschnitt hinzu, bevor der Operator diesen Release-Zyklus ausdrücklich startet.
5. Führen Sie `pnpm release:generated:check`, `pnpm deps:shrinkwrap:check` und `OPENCLAW_TESTBOX=1 pnpm check:changed` aus. Übertragen Sie die Änderungen und überprüfen Sie anschließend, dass `origin/main` die ausgelieferte Version und den Changelog enthält, bevor Sie das stabile Release als abgeschlossen bezeichnen.
6. Halten Sie die Repository-Variablen `RELEASE_ROLLBACK_DRILL_ID` und `RELEASE_ROLLBACK_DRILL_DATE` nach jeder privaten Rollback-Übung aktuell.

`OpenClaw Stable Main Closeout` beginnt mit der Übertragung von `main`, die nach der stabilen Veröffentlichung die ausgelieferte Version, den Changelog und den Appcast enthält. Der Vorgang liest unveränderliche Nachweise nach der Veröffentlichung, um den ausgelieferten Tag an seine Ausführungen der vollständigen Release-Validierung und Veröffentlichung zu binden, und überprüft anschließend den stabilen Zustand des Hauptzweigs, das Release, die obligatorische stabile Beobachtungsphase und die blockierenden Leistungsnachweise. Er hängt dem GitHub-Release ein unveränderliches Abschlussmanifest und dessen Prüfsumme an. Der automatische Übertragungsauslöser überspringt ältere Releases, die vor unveränderlichen Nachweisen nach der Veröffentlichung entstanden sind, und behandelt dieses Überspringen niemals als abgeschlossenen Abschluss.

Ein vollständiger Abschluss erfordert beide Artefakte und eine passende Prüfsumme. Ein unvollständiges Manifest spielt seinen aufgezeichneten SHA `main` und die Rollback-Übung erneut ab, um identische Bytes zu erzeugen, und hängt anschließend die fehlende Prüfsumme an; ein ungültiges Paar oder eine Prüfsumme ohne Manifest bleibt blockierend. Eine durch eine Übertragung ausgelöste Ausführung ohne Repository-Variablen für die Rollback-Übung wird übersprungen, ohne den Abschluss zu vollenden; ein fehlender oder mehr als 90 Tage alter Übungsdatensatz blockiert weiterhin den manuellen nachweisgestützten Abschluss. Private Wiederherstellungsbefehle verbleiben im ausschließlich für Maintainer bestimmten Runbook. Verwenden Sie die manuelle Auslösung nur, um einen nachweisgestützten stabilen Abschluss zu reparieren oder erneut abzuspielen.

Wenn der übergeordnete Release-Veröffentlichungsvorgang erst fehlgeschlagen ist, nachdem unveränderliche npm-/Plugin-Nachweise angehängt wurden, reparieren und veröffentlichen Sie zunächst alle stabilen Plattformartefakte. Anschließend kann ein Maintainer den Abschluss manuell mit `allow_failed_publish_recovery=true` auslösen; dieser Modus akzeptiert nur einen abgeschlossenen fehlgeschlagenen übergeordneten Vorgang und erfordert zusätzlich die exakten Android- und Windows-Artefaktverträge, GitHub-SHA-256-Digests, die Prüfsummenüberprüfung, die Android-Herkunft und eine erfolgreiche, vom übergeordneten Vorgang ausgelöste Windows-Übertragung, deren Authenticode-Prüfungen und vom Kandidaten genehmigte Digests mit den veröffentlichten Installern übereinstimmen, zusätzlich zu den normalen macOS-/Appcast-Prüfungen. Der automatische Abschluss bei einer Übertragung aktiviert diesen Wiederherstellungsmodus niemals.

Ein älterer Fallback-Korrektur-Tag darf Nachweise des Basispakets nur wiederverwenden, wenn der Korrektur-Tag auf denselben Quell-Commit wie der stabile Basis-Tag verweist. Sein Android-Release verwendet die verifizierte APK des Basis-Tags erneut und fügt einen Herkunftsnachweis für den Korrektur-Tag hinzu. Eine Korrektur mit einer anderen Quelle muss eigene Paketnachweise veröffentlichen und überprüfen sowie einen höheren Android-`versionCode` verwenden.

## Release-Preflight

- Führen Sie `pnpm check:test-types` vor dem Release-Preflight aus, damit Test-TypeScript außerhalb der schnelleren lokalen `pnpm check`-Sperre weiterhin abgedeckt bleibt.
- Führen Sie `pnpm check:architecture` vor dem Release-Preflight aus, damit die umfassenderen Prüfungen auf Importzyklen und Architekturgrenzen außerhalb der schnelleren lokalen Sperre erfolgreich sind.
- Führen Sie `pnpm build && pnpm ui:build` vor `pnpm release:check` aus, damit die erwarteten Release-Artefakte `dist/*` und das Control-UI-Bundle für den Paketvalidierungsschritt vorhanden sind.
- Führen Sie `pnpm release:prep` nach der Erhöhung der Stammversion und vor dem Tagging aus. Der Vorgang führt jeden deterministischen Release-Generator aus, bei dem es nach einer Versions-, Konfigurations- oder API-Änderung häufig zu Abweichungen kommt: Plugin-Versionen, npm-Shrinkwraps, Plugin-Inventar, Basiskonfigurationsschema, Konfigurationsmetadaten gebündelter Kanäle, Basisstand der Konfigurationsdokumentation, Plugin-SDK-Exporte, das API-Vertragsmanifest des Plugin-SDK und Locale-Bundles der Control UI. Er blockiert außerdem, bis die Übersetzungen nativer Apps und die von den Plattformen generierten Locale-Ressourcen mit dem Quellinventar übereinstimmen; wenn sie zurückliegen, warten Sie auf `Native App Locale Refresh` oder starten Sie es, bevor Sie den Code-SHA festschreiben. `pnpm release:check` führt diese Prüfungen erneut im Prüfmodus aus, einschließlich der strikten Locale-Sperren und des Oberflächenbudgets des Plugin-SDK, und meldet alle Fehler durch Abweichungen generierter Dateien in einem Durchlauf, bevor die Paket-Release-Prüfungen ausgeführt werden.
- Die Synchronisierung der Plugin-Version aktualisiert standardmäßig das veröffentlichungsfähige Laufzeitpaket `@openclaw/ai`, die Versionen offizieller Plugin-Pakete und vorhandene Untergrenzen von `openclaw.compat.pluginApi` auf die OpenClaw-Release-Version. Behandeln Sie dieses Feld als Untergrenze der Plugin-SDK-/Laufzeit-API und nicht nur als Kopie der Paketversion: Behalten Sie bei reinen Plugin-Releases, die absichtlich mit älteren OpenClaw-Hosts kompatibel bleiben, die Untergrenze bei der ältesten unterstützten Host-API und dokumentieren Sie diese Entscheidung im Plugin-Release-Nachweis.
- Führen Sie den manuellen Workflow `Full Release Validation` vor der Release-Genehmigung aus, um alle Testumgebungen vor dem Release über einen einzigen Einstiegspunkt zu starten. Er akzeptiert einen Branch, Tag oder vollständigen Commit-SHA, startet manuell `CI` und startet `OpenClaw Release Checks` für Installations-Smoke-Tests, Paketabnahme, betriebssystemübergreifende Paketprüfungen, QA-Lab-Parität sowie Matrix- und Telegram-Prüfläufe. Stabile und vollständige Ausführungen enthalten stets umfassende Live-/E2E-Tests und eine Docker-Beobachtungsphase für den Release-Pfad; `run_release_soak=true` bleibt für eine ausdrückliche Beta-Beobachtungsphase erhalten. Die Paketabnahme stellt während der Kandidatenvalidierung den kanonischen Telegram-E2E-Test des Pakets bereit und vermeidet so einen zweiten gleichzeitig laufenden Live-Poller.

  Geben Sie nach der Veröffentlichung einer Beta `release_package_spec` an, um das ausgelieferte npm-Paket in Release-Prüfungen, der Paketabnahme und dem Telegram-E2E-Test des Pakets wiederzuverwenden, ohne den Release-Tarball erneut zu erstellen. Geben Sie `npm_telegram_package_spec` nur an, wenn Telegram ein anderes veröffentlichtes Paket als der übrige Teil der Release-Validierung verwenden soll. Geben Sie `package_acceptance_package_spec` an, wenn die Paketabnahme ein anderes veröffentlichtes Paket als die Release-Paketspezifikation verwenden soll. Geben Sie `evidence_package_spec` an, wenn der Release-Nachweisbericht belegen soll, dass die Validierung einem veröffentlichten npm-Paket entspricht, ohne einen Telegram-E2E-Test zu erzwingen.

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- Führen Sie den manuellen `Package Acceptance`-Workflow aus, wenn Sie einen unabhängigen Nachweis für einen Paketkandidaten benötigen, während die Release-Arbeiten fortgesetzt werden. Verwenden Sie `source=npm` für `openclaw@beta`, `openclaw@latest` oder eine exakte Release-Version; `source=ref`, um einen vertrauenswürdigen `package_ref`-Branch/-Tag/-SHA mit dem aktuellen `workflow_ref`-Testsystem zu paketieren; `source=url` für einen öffentlichen HTTPS-Tarball mit erforderlicher SHA-256-Prüfsumme und strenger Richtlinie für öffentliche URLs; `source=trusted-url` für eine benannte Richtlinie für vertrauenswürdige Quellen mit erforderlichem `trusted_source_id` und SHA-256; oder `source=artifact` für einen Tarball, der von einem anderen GitHub-Actions-Lauf hochgeladen wurde.

  Der Workflow löst den Kandidaten zu `package-under-test` auf, verwendet den Docker-E2E-Release-Scheduler erneut für diesen Tarball und kann mit `telegram_mode=mock-openai` oder `telegram_mode=live-frontier` Telegram-QA für denselben Tarball ausführen. Wenn die ausgewählten Docker-Lanes `published-upgrade-survivor` enthalten, ist das Paketartefakt der Kandidat und `published_upgrade_survivor_baseline` wählt die veröffentlichte Referenzversion aus. `update-restart-auth` verwendet das Kandidatenpaket sowohl als installierte CLI als auch als zu testendes Paket, sodass der verwaltete Neustartpfad des Update-Befehls des Kandidaten ausgeführt wird.

  Beispiel:

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  Häufig verwendete Profile:
  - `smoke`: Lanes für Installation/Kanal/Agent, Gateway-Netzwerk und erneutes Laden der Konfiguration
  - `package`: artefaktnative Lanes für Paket/Update/Neustart/Plugin ohne OpenWebUI oder Live-ClawHub
  - `product`: Paketprofil plus MCP-Kanäle, Cron-/Subagent-Bereinigung, OpenAI-Websuche und OpenWebUI
  - `full`: Docker-Releasepfad-Abschnitte mit OpenWebUI
  - `custom`: exakte Auswahl von `docker_lanes` für eine gezielte Wiederholung

- Führen Sie den manuellen `CI`-Workflow direkt aus, wenn Sie nur eine deterministische normale CI-Abdeckung für den Release-Kandidaten benötigen. Manuelle CI-Ausführungen umgehen die Eingrenzung auf Änderungen und erzwingen die Linux-Node-Shards, die Shards der gebündelten Plugins, die Plugin- und Kanalvertrag-Shards, die Node-22-Kompatibilität, `check-*`, `check-additional-*`, Smoke-Tests für erstellte Artefakte, Dokumentationsprüfungen, Python-Skills, Windows, macOS und die i18n-Lanes der Control UI. Eigenständige manuelle CI-Läufe führen Android nur aus, wenn sie mit `include_android=true` gestartet werden; `Full Release Validation` übergibt diese Eingabe an den untergeordneten CI-Workflow.

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- Führen Sie `pnpm qa:otel:smoke` aus, wenn Sie die Release-Telemetrie validieren. Dies führt QA-lab über einen lokalen OTLP/HTTP-Empfänger aus und überprüft den Export von Traces, Metriken und Protokollen sowie begrenzte Trace-Attribute und die Schwärzung von Inhalten und Bezeichnern, ohne Opik, Langfuse oder einen anderen externen Collector zu benötigen.
- Führen Sie `pnpm qa:otel:collector-smoke` aus, wenn Sie die Collector-Kompatibilität validieren. Dies leitet denselben OTLP-Export von QA-lab durch einen echten Docker-Container mit OpenTelemetry Collector, bevor die Prüfungen des lokalen Empfängers erfolgen.
- Führen Sie `pnpm qa:prometheus:smoke` aus, wenn Sie geschütztes Prometheus-Scraping validieren. Dies führt QA-lab aus, weist nicht authentifizierte Scrapes zurück und überprüft, dass releasekritische Metrikfamilien frei von Prompt-Inhalten, unverarbeiteten Bezeichnern, Authentifizierungstoken und lokalen Pfaden bleiben.
- Führen Sie `pnpm qa:observability:smoke` aus, um die Smoke-Lanes für OpenTelemetry und Prometheus aus dem Quellcode-Checkout direkt nacheinander auszuführen.
- Führen Sie `pnpm release:check` vor jedem mit einem Tag versehenen Release aus.
- Der `OpenClaw NPM Release`-Preflight erzeugt Nachweise zur Freigabe von Abhängigkeiten, bevor er den npm-Tarball paketiert. Das npm-Advisory-Schwachstellen-Gate blockiert das Release bei Fehlern. Die Berichte zu Risiken im transitiven Manifest, zur Eigentümerschaft und Installationsoberfläche von Abhängigkeiten sowie zu Änderungen an Abhängigkeiten dienen nur als Release-Nachweise. Der Bericht zu Änderungen an Abhängigkeiten vergleicht den Release-Kandidaten mit dem vorherigen erreichbaren Release-Tag. Der Preflight lädt die Abhängigkeitsnachweise als `openclaw-release-dependency-evidence-<tag>` hoch und bettet sie außerdem unter `dependency-evidence/` in das vorbereitete npm-Preflight-Artefakt ein. Der tatsächliche Veröffentlichungspfad verwendet dieses Preflight-Artefakt erneut und hängt anschließend dieselben Nachweise als `openclaw-<version>-dependency-evidence.zip` an das GitHub-Release an.
- Führen Sie `OpenClaw Release Publish` für die verändernde Veröffentlichungssequenz aus, nachdem das Tag vorhanden ist. Starten Sie reguläre Beta- und stabile Veröffentlichungen vom vertrauenswürdigen `main`; das Release-Tag wählt weiterhin den exakten Ziel-Commit aus und kann auf `release/YYYY.M.PATCH` verweisen. Tideclaw-Alpha-Veröffentlichungen verbleiben auf ihrem entsprechenden Alpha-Branch. Übergeben Sie den erfolgreichen OpenClaw-npm-`preflight_run_id`, den erfolgreichen `full_release_validation_run_id` und den exakten `full_release_validation_run_attempt`, und behalten Sie den standardmäßigen Plugin-Veröffentlichungsumfang `all-publishable` bei, sofern Sie nicht bewusst eine gezielte Reparatur ausführen. Der Workflow führt die npm-Veröffentlichung der Plugins, die ClawHub-Veröffentlichung der Plugins und die npm-Veröffentlichung von OpenClaw nacheinander aus, damit das Kernpaket nicht vor seinen externalisierten Plugins veröffentlicht wird; die Windows- und Android-Promotion läuft gleichzeitig mit der Veröffentlichung des Kernpakets auf npm und verwendet dabei die Entwurfsseite des Releases. Wiederholungen der Veröffentlichung können fortgesetzt werden: Eine bereits veröffentlichte npm-Version des Kernpakets überspringt die Kernausführung, nachdem der Workflow nachgewiesen hat, dass der Registry-Tarball dem Preflight-Artefakt des Tags entspricht. Die Windows-/Android-Promotion wird übersprungen, wenn das Release bereits den verifizierten Asset-Vertrag enthält, sodass bei einem erneuten Versuch nur die fehlgeschlagenen Phasen wiederholt werden. Gezielte Reparaturen ausschließlich für Plugins erfordern `plugin_publish_scope=selected` und eine nicht leere Plugin-Liste. Ausschließlich auf Plugins bezogene `all-publishable`-Läufe erfordern vollständige, unveränderliche Nachweise aus Preflight und vollständiger Release-Validierung; unvollständige Nachweise werden abgelehnt.
- Das stabile `OpenClaw Release Publish` erfordert einen exakten `windows_node_tag`, nachdem das entsprechende `openclaw/openclaw-windows-node`-Release ohne Vorabversionskennzeichnung vorhanden ist, sowie die für den Kandidaten genehmigte `windows_node_installer_digests`-Zuordnung. Vor dem Start eines untergeordneten Veröffentlichungs-Workflows überprüft es, dass dieses Quell-Release veröffentlicht und keine Vorabversion ist, die erforderlichen x64-/ARM64-Installationsprogramme enthält und weiterhin dieser genehmigten Zuordnung entspricht. Anschließend startet es `Windows Node Release`, während das OpenClaw-Release noch ein Entwurf ist, und übergibt dabei die festgelegte Zuordnung der Installationsprogramm-Digests unverändert. Der untergeordnete Workflow lädt die signierten Installationsprogramme von Windows Hub von exakt diesem Tag herunter, gleicht sie mit den festgelegten Digests ab, überprüft auf einem Windows-Runner, dass ihre Authenticode-Signaturen den erwarteten Unterzeichner OpenClaw Foundation verwenden, erstellt ein SHA-256-Manifest und lädt die Installationsprogramme samt Manifest in das kanonische OpenClaw-GitHub-Release hoch. Anschließend lädt er die übernommenen Assets erneut herunter und überprüft ihre Zugehörigkeit zum Manifest sowie ihre Hashes. Der übergeordnete Workflow überprüft vor der Veröffentlichung den aktuellen Vertrag für x64-, ARM64- und Prüfsummen-Assets. Die direkte Wiederherstellung weist unerwartete `OpenClawCompanion-*`-Asset-Namen zurück, bevor die erwarteten Vertrags-Assets durch die festgelegten Bytes der Quelle ersetzt werden.

  Starten Sie `Windows Node Release` nur zur Wiederherstellung manuell und übergeben Sie stets ein exaktes Tag, niemals `latest`, sowie die explizite `expected_installer_digests`-JSON-Zuordnung aus dem genehmigten Quell-Release. Download-Links auf der Website sollten auf exakte URLs der OpenClaw-Release-Assets für das aktuelle stabile Release verweisen oder nur dann auf `releases/latest/download/...`, nachdem überprüft wurde, dass die Weiterleitung von GitHub für das neueste Release auf dasselbe Release verweist; verlinken Sie nicht ausschließlich auf die Release-Seite des begleitenden Repositorys.

- Release-Prüfungen werden jetzt in einem separaten manuellen Workflow ausgeführt: `OpenClaw Release Checks`. Er führt außerdem die QA-Lab-Lane für Mock-Parität sowie das Matrix-Release-Profil und die Telegram-QA-Lane vor der Release-Freigabe aus. Die Live-Lanes verwenden die Umgebung `qa-live-shared`; Telegram verwendet zusätzlich Convex-CI-Credential-Leases. Führen Sie den manuellen Workflow `QA-Lab - All Lanes` mit `matrix_profile=all` aus, wenn Sie alle gepflegten Matrix-Szenarien ausführen möchten; der Workflow verteilt diese Auswahl auf die Transport-, Medien- und E2EE-Profile, damit der vollständige Nachweis innerhalb der Zeitüberschreitungen pro Job bleibt.
- Die betriebssystemübergreifende Laufzeitvalidierung von Installation und Upgrade ist Teil der öffentlichen Workflows `OpenClaw Release Checks` und `Full Release Validation`, die den wiederverwendbaren Workflow `.github/workflows/openclaw-cross-os-release-checks-reusable.yml` direkt aufrufen. Diese Trennung ist beabsichtigt: Der echte npm-Release-Pfad bleibt kurz, deterministisch und auf Artefakte ausgerichtet, während langsamere Live-Prüfungen in ihrer eigenen Lane verbleiben, damit sie die Veröffentlichung weder verzögern noch blockieren.
- Release-Prüfungen, die Secrets verwenden, sollten über `Full Release Validation` oder vom Workflow-Ref `main`/release ausgelöst werden, damit Workflow-Logik und Secrets kontrolliert bleiben.
- `OpenClaw Release Checks` akzeptiert einen Branch, ein Tag oder einen vollständigen Commit-SHA, solange der aufgelöste Commit von einem OpenClaw-Branch oder Release-Tag aus erreichbar ist.
- Der reine Validierungs-Preflight von `OpenClaw NPM Release` akzeptiert außerdem den aktuellen vollständigen, 40 Zeichen langen Commit-SHA des Workflow-Branches, ohne ein gepushtes Tag zu erfordern. Dieser SHA-Pfad dient ausschließlich der Validierung und kann nicht zu einer echten Veröffentlichung hochgestuft werden. Im SHA-Modus erzeugt der Workflow `v<package.json version>` ausschließlich für die Prüfung der Paketmetadaten; eine echte Veröffentlichung erfordert weiterhin ein echtes Release-Tag.
- Beide Workflows belassen den echten Veröffentlichungs- und Hochstufungspfad auf von GitHub gehosteten Runnern, während der nicht verändernde Validierungspfad die größeren Blacksmith-Linux-Runner verwenden kann.
- Dieser Workflow führt `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache` mit den beiden Workflow-Secrets `OPENAI_API_KEY` und `ANTHROPIC_API_KEY` aus.
- Der npm-Release-Preflight wartet nicht mehr auf die separate Lane für Release-Prüfungen.
- Führen Sie vor dem lokalen Taggen eines Release-Kandidaten `RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check` aus. Das Hilfsprogramm führt die schnellen Release-Schutzprüfungen, die npm-/ClawHub-Release-Prüfungen für Plugins, den Build, den UI-Build und `release:openclaw:npm:check` in einer Reihenfolge aus, die häufige, die Freigabe blockierende Fehler erkennt, bevor der GitHub-Veröffentlichungsworkflow startet.
- Führen Sie vor der Freigabe `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts` (oder das entsprechende Vorabrelease-/Korrektur-Tag) aus.
- Führen Sie nach der npm-Veröffentlichung `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH` (oder die entsprechende Beta-/Korrekturversion) aus, um den veröffentlichten Registry-Installationspfad in einem neuen temporären Präfix zu verifizieren.
- Führen Sie nach einer Beta-Veröffentlichung `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live` aus, um das Onboarding des installierten Pakets, die Telegram-Einrichtung und echte Telegram-E2E-Tests mit dem veröffentlichten npm-Paket und dem gemeinsam genutzten Pool geleaster Telegram-Credentials zu verifizieren. Für einmalige lokale Maintainer-Ausführungen können die Convex-Variablen entfallen und die drei `OPENCLAW_QA_TELEGRAM_*`-Umgebungs-Credentials direkt übergeben werden.
- Verwenden Sie `pnpm release:beta-smoke -- --beta betaN`, um den vollständigen Beta-Smoke-Test nach der Veröffentlichung von einem Maintainer-Rechner auszuführen. Das Hilfsprogramm führt die Parallels-Validierung für npm-Update und neues Ziel aus, löst `NPM Telegram Beta E2E` aus, fragt den exakten Workflow-Lauf ab, lädt das Artefakt herunter und gibt den Telegram-Bericht aus.
- Maintainer können dieselbe Prüfung nach der Veröffentlichung über den manuellen Workflow `NPM Telegram Beta E2E` in GitHub Actions ausführen. Er ist bewusst ausschließlich manuell und wird nicht bei jedem Merge ausgeführt.
- Die Release-Automatisierung für Maintainer verwendet das Prinzip „Preflight, dann Hochstufung“:
  - Eine echte npm-Veröffentlichung muss einen erfolgreichen npm-`preflight_run_id` durchlaufen.
  - Die reguläre Orchestrierung und der Preflight für Beta- und stabile Veröffentlichungen verwenden vertrauenswürdiges `main` für das exakte Ziel-Tag. Die Veröffentlichung und der Preflight für Tideclaw Alpha verwenden den entsprechenden Alpha-Branch.
  - Stabile npm-Releases verwenden standardmäßig `beta`; die stabile npm-Veröffentlichung kann über eine Workflow-Eingabe explizit auf `latest` abzielen.
  - Die tokenbasierte Änderung des npm-Dist-Tags befindet sich in `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml`, weil `npm dist-tag add` weiterhin `NPM_TOKEN` benötigt, während das Quell-Repository ausschließlich OIDC-basierte Veröffentlichungen beibehält.
  - Der öffentliche Workflow `macOS Release` dient ausschließlich der Validierung; wenn sich ein Tag nur auf einem Release-Branch befindet, der Workflow jedoch von `main` ausgelöst wird, setzen Sie `public_release_branch=release/YYYY.M.PATCH`.
  - Eine echte macOS-Veröffentlichung muss erfolgreiche macOS-`preflight_run_id` und `validate_run_id` durchlaufen.
  - Echte Veröffentlichungspfade stufen vorbereitete Artefakte hoch, anstatt sie erneut zu erstellen.
- Bei stabilen Korrektur-Releases wie `YYYY.M.PATCH-N` prüft der Verifizierer nach der Veröffentlichung außerdem denselben Upgrade-Pfad mit temporärem Präfix von `YYYY.M.PATCH` auf `YYYY.M.PATCH-N`, damit Release-Korrekturen ältere globale Installationen nicht unbemerkt auf dem ursprünglichen stabilen Payload belassen.
- Der npm-Release-Preflight schlägt sicher geschlossen fehl, sofern das Tarball nicht sowohl `dist/control-ui/index.html` als auch einen nicht leeren `dist/control-ui/assets/`-Payload enthält, damit nicht erneut ein leeres Browser-Dashboard ausgeliefert wird.
- Die Verifizierung nach der Veröffentlichung prüft außerdem, ob die veröffentlichten Plugin-Einstiegspunkte und Paketmetadaten im installierten Registry-Layout vorhanden sind. Ein Release, bei dem Plugin-Laufzeit-Payloads fehlen, lässt den Postpublish-Verifizierer fehlschlagen und kann nicht zu `latest` hochgestuft werden.
- `pnpm test:install:smoke` erzwingt außerdem das npm-Pack-Budget `unpackedSize` für das Kandidaten-Update-Tarball, damit Installer-E2E-Tests versehentliches Anwachsen des Pakets vor dem Release-Veröffentlichungspfad erkennen.
- Wenn die Release-Arbeit die CI-Planung, Zeitmanifestdateien von Erweiterungen oder Testmatrizen von Erweiterungen berührt hat, generieren und prüfen Sie vor der Freigabe die vom Planer verwalteten `plugin-prerelease-extension-shard`-Matrixausgaben aus `.github/workflows/plugin-prerelease.yml` neu, damit die Release Notes kein veraltetes CI-Layout beschreiben.
- Die Bereitschaft für stabile macOS-Releases umfasst außerdem die Updater-Oberflächen: Das GitHub-Release muss letztlich die paketierten Dateien `.zip`, `.dmg` und `.dSYM.zip` enthalten; `appcast.xml` auf `main` muss nach der Veröffentlichung auf die neue stabile ZIP-Datei verweisen (der macOS-Veröffentlichungsworkflow committet sie automatisch oder öffnet einen Appcast-PR, wenn ein direkter Push blockiert ist); die paketierte App muss eine Nicht-Debug-Bundle-ID, eine nicht leere Sparkle-Feed-URL und eine `CFBundleVersion` auf oder über der kanonischen Sparkle-Build-Untergrenze für diese Release-Version beibehalten.

## Release-Testboxen

`Full Release Validation` ermöglicht es Operatoren, die vollständige Produktmatrix über einen einzigen Einstiegspunkt zu starten. Verwenden Sie das Hilfsprogramm, damit jeder untergeordnete Workflow von einem temporären Branch ausgeführt wird, der auf einen vertrauenswürdigen `main`-Workflow-SHA festgelegt ist, während der angeforderte Commit der zu testende Kandidat bleibt:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

Das Hilfsprogramm ruft den aktuellen Stand von `origin/main` ab, pusht `release-ci/<workflow-sha>-...` an diesem vertrauenswürdigen Workflow-Commit, leitet `beta` aus Alpha-/Beta-Paketversionen und andernfalls `stable` ab, löst `Full Release Validation` vom temporären Branch mit `ref=<target-sha>` aus, verifiziert, dass `headSha` jedes untergeordneten Workflows mit dem fixierten SHA des übergeordneten Workflows übereinstimmt, und löscht anschließend den temporären Branch. Übergeben Sie `-f reuse_evidence=false`, um einen neuen Lauf zu erzwingen, `-f release_profile=full` für die umfassende beratende Prüfung oder `--workflow-sha <trusted-main-sha>`, um einen älteren Commit zu fixieren, der vom aktuellen `origin/main` aus weiterhin erreichbar ist. Der Workflow selbst schreibt niemals Repository-Refs. Dadurch bleibt die ausschließlich auf Main verfügbare Release-Werkzeugausstattung nutzbar, ohne dem Kandidaten Tooling-Commits hinzuzufügen, und es wird vermieden, versehentlich einen neueren untergeordneten `main`-Lauf als Nachweis zu verwenden.

Nachdem der Code-SHA grün ist, committen Sie ausschließlich `CHANGELOG.md` und führen dasselbe Hilfsprogramm mit dem Release-SHA aus:

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

Der zweite übergeordnete Workflow verwendet Produktnachweise nur dann wieder, wenn GitHub nachweist, dass der Release-SHA vom Code-SHA abstammt und die vollständige Menge geänderter Pfade exakt `CHANGELOG.md` entspricht. Er zeichnet `changelog-only-release-v1` auf und löst keine untergeordneten Produkt-Workflows aus. Der npm-Preflight und die Paket-/Installationsakzeptanz werden weiterhin für den Release-SHA ausgeführt, da sich seine Tarball-Bytes geändert haben.

Für einen neuen Code-SHA löst der Workflow das Ziel auf, löst den manuellen Workflow `CI` und anschließend `OpenClaw Release Checks` aus. `OpenClaw Release Checks` verteilt Installations-Smoke-Tests, betriebssystemübergreifende Release-Prüfungen, Live-/E2E-Docker-Abdeckung des Release-Pfads bei aktiviertem Soak, Paketakzeptanz mit dem kanonischen Telegram-Paket-E2E, QA-Lab-Parität, Live-Matrix und Live-Telegram. Ein vollständiger/all-Lauf ist nur akzeptabel, wenn die Zusammenfassung `Full Release Validation` `normal_ci`, `plugin_prerelease` und `release_checks` als erfolgreich ausweist, es sei denn, bei einer gezielten Wiederholung wurde der separate untergeordnete Workflow `Plugin Prerelease` absichtlich übersprungen. Verwenden Sie den eigenständigen untergeordneten Workflow `npm-telegram` nur für eine gezielte Wiederholung mit veröffentlichtem Paket und `release_package_spec` oder `npm_telegram_package_spec`. Die abschließende Zusammenfassung des Verifizierers enthält Tabellen der langsamsten Jobs für jeden untergeordneten Lauf, sodass die Release-Verantwortlichen den aktuellen kritischen Pfad sehen können, ohne Protokolle herunterzuladen.

Der untergeordnete Workflow für die Produktleistung ist in diesem Release-Pfad ausschließlich artefaktbasiert. Der
übergeordnete Workflow löst ihn mit `publish_reports=false` aus, und die Validierung wird abgelehnt,
sofern seine reine Artefakt-Schutzprüfung nicht nachweist, dass der Clawgrit-Berichts-Publisher
übersprungen blieb.

Unter [Vollständige Release-Validierung](/de/reference/full-release-validation) finden Sie die vollständige Phasenmatrix, die exakten Workflow-Jobnamen, die Unterschiede zwischen stabilem und vollständigem Profil, Artefakte und Optionen für gezielte Wiederholungen.

Untergeordnete Workflows werden vom SHA-fixierten vertrauenswürdigen Ref ausgelöst, der `Full Release Validation` ausführt. Jeder untergeordnete Lauf muss exakt den SHA des übergeordneten Workflows verwenden. Verwenden Sie für Release-Nachweise keine direkten `--ref main -f ref=<sha>`-Auslösungen; verwenden Sie `pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH`.

Verwenden Sie `release_profile`, um den Umfang der Live-/Provider-Abdeckung auszuwählen:

- `beta`: schnellster releasekritischer OpenAI-/Core-Live- und Docker-Pfad
- `stable`: Beta- plus stabile Provider-/Backend-Abdeckung für die Release-Freigabe
- `full`: stabile plus umfassende beratende Provider-/Medienabdeckung

Die stabile und die vollständige Validierung führen vor der Hochstufung immer die umfassenden Live-/E2E-Prüfungen, den Docker-Release-Pfad und die begrenzte Überlebensprüfung für Upgrades veröffentlichter Pakete aus. Verwenden Sie `run_release_soak=true`, um dieselbe Prüfung für eine Beta anzufordern. Diese Prüfung umfasst die neuesten vier stabilen Pakete sowie die fixierten Baselines `2026.4.23` und `2026.5.2` und zusätzlich die Abdeckung älterer `2026.4.15`-Versionen; doppelte Baselines werden entfernt und jede Baseline wird einem eigenen Docker-Runner-Job zugewiesen.

`OpenClaw Release Checks` verwendet den vertrauenswürdigen Workflow-Ref, um den Ziel-Ref einmalig als `release-package-under-test` aufzulösen, und verwendet dieses Artefakt bei ausgeführtem Soak in betriebssystemübergreifenden Prüfungen, der Paketakzeptanz und den Docker-Prüfungen des Release-Pfads erneut. Dadurch verwenden alle paketbezogenen Boxen dieselben Bytes und wiederholte Paket-Builds werden vermieden. Nachdem eine Beta bereits auf npm verfügbar ist, setzen Sie `release_package_spec=openclaw@YYYY.M.PATCH-beta.N`, damit die Release-Prüfungen das ausgelieferte Paket einmal herunterladen, seinen Build-Quell-SHA aus `dist/build-info.json` extrahieren und dieses Artefakt für betriebssystemübergreifende Prüfungen, Paketakzeptanz, Release-Pfad-Docker und Telegram-Paket-Lanes wiederverwenden.

Der betriebssystemübergreifende OpenAI-Installations-Smoke-Test verwendet `OPENCLAW_CROSS_OS_OPENAI_MODEL`, wenn die Repository-/Organisationsvariable gesetzt ist, andernfalls `openai/gpt-5.6-luna`, da diese Lane die Paketinstallation, das Onboarding, den Gateway-Start und einen Live-Agentendurchlauf nachweist, anstatt das leistungsfähigste Modell zu benchmarken. Die umfassendere Live-Provider-Matrix bleibt der Ort für modellspezifische Abdeckung.

Verwenden Sie je nach Release-Phase die folgenden Varianten:

```bash
# Den produktvollständigen Code-SHA validieren.
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# Den ausschließlich das Änderungsprotokoll betreffenden Release-SHA durch Wiederverwendung der Produktnachweise des Code-SHA validieren.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# Nach der Veröffentlichung einer Beta Telegram-E2E für das veröffentlichte Paket hinzufügen.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

Verwenden Sie den vollständigen übergeordneten Lauf nicht als ersten erneuten Lauf nach einer gezielten Korrektur. Wenn eine Box fehlschlägt, verwenden Sie für den nächsten Nachweis den fehlgeschlagenen untergeordneten Workflow, Job, Docker-Lane, das Paketprofil, den Modell-Provider oder die QA-Lane. Führen Sie den vollständigen übergeordneten Lauf nur dann erneut aus, wenn die Korrektur die gemeinsame Release-Orchestrierung geändert oder frühere Nachweise aller Boxen ungültig gemacht hat. Der abschließende Prüfer des übergeordneten Laufs überprüft die aufgezeichneten Ausführungs-IDs der untergeordneten Workflows erneut. Führen Sie daher nach dem erfolgreichen erneuten Lauf eines untergeordneten Workflows nur den fehlgeschlagenen übergeordneten Job `Verify full validation` erneut aus.

`rerun_group=all` kann einen früheren erfolgreichen übergeordneten Lauf wiederverwenden, wenn das Release-Profil,
die effektive Soak-Einstellung und die Validierungseingaben übereinstimmen und entweder der Ziel-SHA
identisch ist oder das neue Ziel ein Nachfolger ist, dessen vollständige Menge geänderter Pfade
genau `CHANGELOG.md` entspricht. Bei der Wiederverwendung des exakten Ziels wird
`exact-target-full-validation-v1` aufgezeichnet; beim Release-SHA nach der Validierung wird
`changelog-only-release-v1` aufgezeichnet. Letzteres verwendet nur die Produktvalidierung wieder. Npm-
Vorprüfung, Paketbytes, Herkunft der Release-Hinweise und Akzeptanz von Installation/Aktualisierung
müssen weiterhin für den Release-SHA ausgeführt werden. Jede Änderung am Ziel bezüglich Version, Quelle, generierter
Dateien, Abhängigkeiten, Paket oder Workflow erfordert einen neuen Code-SHA
und eine neue vollständige Validierung. Neuere übergeordnete Läufe für dieselbe `release/*`-Referenz und
Gruppe erneuter Läufe ersetzen laufende Ausführungen automatisch. Übergeben Sie
`reuse_evidence=false`, um einen neuen vollständigen Lauf zu erzwingen.

Übergeben Sie für eine begrenzte Wiederherstellung `rerun_group` an den übergeordneten Lauf. `all` ist der tatsächliche Release-Kandidatenlauf, `ci` führt nur den normalen untergeordneten CI-Lauf aus, `plugin-prerelease` führt nur den ausschließlich für Releases vorgesehenen untergeordneten Plugin-Lauf aus, `release-checks` führt jede Release-Box aus, und die enger gefassten Release-Gruppen sind `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` und `npm-telegram`. Gezielte erneute `npm-telegram`-Läufe erfordern `release_package_spec` oder `npm_telegram_package_spec`; vollständige/alle Läufe verwenden das kanonische Paket-Telegram-E2E innerhalb von Package Acceptance. Gezielte betriebssystemübergreifende erneute Läufe können `cross_os_suite_filter=windows/packaged-upgrade` oder einen anderen Betriebssystem-/Suite-Filter hinzufügen. Fehler bei QA-Release-Prüfungen blockieren die normale Release-Validierung, einschließlich Abweichungen dynamischer OpenClaw-Tools in der Kern-Runtime-Paar-Lane. Tideclaw-Alpha-Läufe können Release-Prüfungs-Lanes, die nicht der Paketsicherheit dienen, weiterhin als beratend behandeln. Mit `release_profile=beta` sind die Live-Provider-Suites `Run repo/live E2E validation` beratend (Warnungen, keine Blocker); stabile und vollständige Profile behandeln sie weiterhin als blockierend. Wenn `live_suite_filter` ausdrücklich eine zugangsbeschränkte QA-Live-Lane wie Discord, WhatsApp oder Slack anfordert, muss die entsprechende Repo-Variable `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` aktiviert sein; andernfalls schlägt die Erfassung der Eingaben fehl, statt die Lane stillschweigend zu überspringen.

### Vitest

Die Vitest-Box ist der manuelle untergeordnete Workflow `CI`. Die manuelle CI umgeht absichtlich die Eingrenzung nach Änderungen und erzwingt den normalen Testgraphen für den Release-Kandidaten: Linux-Node-Shards, Shards gebündelter Plugins, Plugin- und Kanalvertrag-Shards, Node-22-Kompatibilität, `check-*`, `check-additional-*`, Smoke-Prüfungen erstellter Artefakte, Dokumentationsprüfungen, Python-Skills, Windows, macOS und Control-UI-i18n. Android ist enthalten, wenn `Full Release Validation` die Box ausführt, da der übergeordnete Lauf `include_android=true` übergibt; eine eigenständige manuelle CI erfordert `include_android=true` für die Android-Abdeckung.

Verwenden Sie diese Box, um die Frage „Hat der Quellbaum die vollständige normale Testsuite bestanden?“ zu beantworten. Sie ist nicht mit der Produktvalidierung des Release-Pfads identisch. Aufzubewahrende Nachweise:

- `Full Release Validation`-Zusammenfassung, die die URL des ausgelösten `CI`-Laufs zeigt
- erfolgreicher `CI`-Lauf für den exakten Ziel-SHA
- Namen fehlgeschlagener oder langsamer Shards aus den CI-Jobs bei der Untersuchung von Regressionen
- Vitest-Zeitmessungsartefakte wie `.artifacts/vitest-shard-timings.json`, wenn für einen Lauf eine Leistungsanalyse erforderlich ist

Führen Sie die manuelle CI nur dann direkt aus, wenn das Release eine deterministische normale CI, aber nicht die Docker-, QA-Lab-, Live-, betriebssystemübergreifenden oder Paket-Boxen benötigt. Verwenden Sie den ersten Befehl für eine direkte CI ohne Android. Fügen Sie `include_android=true` hinzu, wenn die direkte Release-Kandidaten-CI Android abdecken muss:

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

Die Docker-Box befindet sich in `OpenClaw Release Checks` bis `openclaw-live-and-e2e-checks-reusable.yml` sowie im Release-Modus-Workflow `install-smoke`. Sie validiert den Release-Kandidaten über paketierte Docker-Umgebungen statt ausschließlich über Tests auf Quellebene.

Die Docker-Abdeckung für Releases umfasst:

- vollständigen Installations-Smoke-Test mit aktiviertem langsamem globalem Bun-Installations-Smoke-Test
- Vorbereitung/Wiederverwendung des Smoke-Test-Images des Root-Dockerfiles nach Ziel-SHA, wobei QR-, Root-/Gateway- und Installer-/Bun-Smoke-Jobs als separate Installations-Smoke-Shards ausgeführt werden
- E2E-Lanes des Repositorys
- Docker-Blöcke des Release-Pfads: `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, `plugins-runtime-install-a` bis `plugins-runtime-install-h` und `openwebui`
- OpenWebUI-Abdeckung auf einem dedizierten Runner mit großem Datenträger, wenn angefordert
- aufgeteilte Installations-/Deinstallations-Lanes gebündelter Plugins von `bundled-plugin-install-uninstall-0` bis `bundled-plugin-install-uninstall-23`
- Live-/E2E-Provider-Suites und Docker-Live-Modellabdeckung, wenn die Release-Prüfungen Live-Suites umfassen

Verwenden Sie Docker-Artefakte vor einem erneuten Lauf. Der Scheduler des Release-Pfads lädt `.artifacts/docker-tests/` mit Lane-Protokollen, `summary.json`, `failures.json`, Phasenzeitmessungen, dem Scheduler-Plan als JSON und Befehlen für erneute Läufe hoch. Verwenden Sie für eine gezielte Wiederherstellung `docker_lanes=<lane[,lane]>` im wiederverwendbaren Live-/E2E-Workflow, statt alle Release-Blöcke erneut auszuführen. Generierte Befehle für erneute Läufe enthalten vorherige `package_artifact_run_id`- und vorbereitete Docker-Image-Eingaben, sofern verfügbar, sodass eine fehlgeschlagene Lane denselben Tarball und dieselben GHCR-Images wiederverwenden kann.

### QA Lab

Die QA-Lab-Box ist ebenfalls Teil von `OpenClaw Release Checks`. Sie ist das Release-Gate für agentisches Verhalten und Verhalten auf Kanalebene, getrennt von Vitest und den Paketmechanismen von Docker.

Die QA-Lab-Abdeckung für Releases umfasst:

- Mock-Paritäts-Lane, die die OpenAI-Kandidaten-Lane mithilfe des Pakets für agentische Parität mit der `anthropic/claude-opus-4-8`-Baseline vergleicht
- Matrix-Live-Adapter-Release-Profil unter Verwendung der `qa-live-shared`-Umgebung
- Live-Telegram-QA-Lane unter Verwendung von Convex-CI-Zugangsdaten-Leases
- `pnpm qa:otel:smoke`, `pnpm qa:otel:collector-smoke`, `pnpm qa:prometheus:smoke` oder `pnpm qa:observability:smoke`, wenn die Release-Telemetrie einen ausdrücklichen lokalen Nachweis benötigt

Verwenden Sie diese Box, um die Frage „Verhält sich das Release in QA-Szenarien und Live-Kanalabläufen korrekt?“ zu beantworten. Bewahren Sie bei der Freigabe des Releases die Artefakt-URLs für die Paritäts-, Matrix- und Telegram-Lanes auf. Die vollständige Matrix-Abdeckung bleibt als manueller QA-Lab-Lauf mit Sharding verfügbar, statt als standardmäßige releasekritische Lane.

### Paket

Die Paket-Box ist das Gate für das installierbare Produkt. Sie basiert auf `Package Acceptance` und dem Resolver `scripts/resolve-openclaw-package-candidate.mjs`. Der Resolver normalisiert einen Kandidaten in den von Docker-E2E verwendeten `package-under-test`-Tarball, validiert den Paketbestand, zeichnet Paketversion und SHA-256 auf und hält die Workflow-Harness-Referenz von der Paketquellenreferenz getrennt.

Unterstützte Kandidatenquellen:

- `source=npm`: `openclaw@beta`, `openclaw@latest` oder eine exakte OpenClaw-Release-Version
- `source=ref`: einen vertrauenswürdigen `package_ref`-Branch, -Tag oder vollständigen Commit-SHA mit dem ausgewählten `workflow_ref`-Harness paketieren
- `source=url`: eine öffentliche HTTPS-`.tgz` mit erforderlichem `package_sha256` herunterladen; URL-Zugangsdaten, nicht standardmäßige HTTPS-Ports, private/interne/für besondere Zwecke reservierte Hostnamen oder aufgelöste Adressen sowie unsichere Weiterleitungen werden abgelehnt
- `source=trusted-url`: eine HTTPS-`.tgz` mit erforderlichem `package_sha256` und `trusted_source_id` aus einer benannten Richtlinie in `.github/package-trusted-sources.json` herunterladen; verwenden Sie dies für von Maintainern verwaltete Unternehmensspiegel oder private Paket-Repositorys, statt `source=url` eine eingabebezogene Umgehung für private Netzwerke hinzuzufügen
- `source=artifact`: einen von einem anderen GitHub-Actions-Lauf hochgeladenen `.tgz` wiederverwenden

`OpenClaw Release Checks` führt Package Acceptance mit `source=artifact`, dem vorbereiteten Release-Paketartefakt, `suite_profile=custom`, `docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape`, `telegram_mode=mock-openai` aus. Package Acceptance führt Migration, Aktualisierung, Root-verwaltetes VPS-Upgrade, Neustart nach einer Aktualisierung mit konfigurierter Authentifizierung, Installation eines Live-ClawHub-Skills, Bereinigung veralteter Plugin-Abhängigkeiten, Offline-Plugin-Fixtures, Plugin-Aktualisierung, Absicherung des Escapings für Plugin-Befehlsbindungen und Telegram-Paket-QA mit demselben aufgelösten Tarball durch. Blockierende Release-Prüfungen verwenden standardmäßig die neueste veröffentlichte Paket-Baseline; das Beta-Profil mit `run_release_soak=true`, `release_profile=stable` oder `release_profile=full` erweitert die Prüfung überlebender veröffentlichter Upgrades auf `last-stable-4` sowie die fixierten Baselines `2026.4.23`, `2026.5.2` und `2026.4.15` mit `reported-issues`-Szenarien. Verwenden Sie Package Acceptance mit `source=npm` für einen bereits ausgelieferten Kandidaten, `source=ref` für einen SHA-basierten lokalen npm-Tarball vor der Veröffentlichung, `source=trusted-url` für einen von Maintainern verwalteten Unternehmens-/Privatspiegel oder `source=artifact` für einen vorbereiteten Tarball, der von einem anderen GitHub-Actions-Lauf hochgeladen wurde.

Es ist der GitHub-native Ersatz für den Großteil der Paket-/Aktualisierungsabdeckung, die zuvor Parallels erforderte. Betriebssystemübergreifende Release-Prüfungen sind weiterhin für betriebssystemspezifisches Onboarding-, Installer- und Plattformverhalten relevant, für die Produktvalidierung von Paketen/Aktualisierungen sollte jedoch Package Acceptance bevorzugt werden.

Die kanonische Checkliste für die Validierung von Aktualisierungen und Plugins ist [Aktualisierungen und Plugins testen](/de/help/testing-updates-plugins). Verwenden Sie sie bei der Entscheidung, welche lokale, Docker-, Package-Acceptance- oder Release-Prüfungs-Lane die Installation/Aktualisierung eines Plugins, eine Doctor-Bereinigung oder eine Migration veröffentlichter Pakete nachweist. Die vollständige Migration veröffentlichter Aktualisierungen aus jedem stabilen `2026.4.23+`-Paket ist ein separater manueller `Update Migration`-Workflow und nicht Teil der vollständigen Release-CI.

Die Nachsicht der Legacy-Paketakzeptanz ist absichtlich zeitlich begrenzt. Pakete bis einschließlich `2026.4.25` dürfen den Kompatibilitätspfad für Metadatenlücken verwenden, die bereits auf npm veröffentlicht wurden: private QA-Bestandseinträge, die im Tarball fehlen, fehlendes `gateway install --wrapper`, fehlende Patch-Dateien im aus dem Tarball abgeleiteten Git-Fixture, fehlendes persistiertes `update.channel`, veraltete Speicherorte für Plugin-Installationsdatensätze, fehlende Persistenz von Marketplace-Installationsdatensätzen und die Migration von Konfigurationsmetadaten während `plugins update`. Das veröffentlichte Paket `2026.4.26` darf bei bereits ausgelieferten lokalen Build-Metadaten-Stempeldateien warnen. Spätere Pakete müssen die modernen Paketverträge erfüllen; dieselben Lücken führen dann zum Fehlschlagen der Release-Validierung.

Verwenden Sie umfassendere Package-Acceptance-Profile, wenn sich die Release-Frage auf ein tatsächlich installierbares Paket bezieht:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

Übliche Paketprofile:

- `smoke`: schnelle Pfade für Paketinstallation/Kanal/Agent, Gateway-Netzwerk und erneutes Laden der Konfiguration
- `package`: Verträge für Installation/Aktualisierung/Neustart/Plugin-Pakete sowie Live-Nachweis der ClawHub-Skill-Installation; dies ist der Standard für die Release-Prüfung
- `product`: `package` plus MCP-Kanäle, Bereinigung von Cron/Subagenten, OpenAI-Websuche und OpenWebUI
- `full`: Abschnitte des Docker-Release-Pfads mit OpenWebUI
- `custom`: exakte `docker_lanes`-Liste für gezielte Wiederholungsläufe

Aktivieren Sie für den Telegram-Nachweis eines Paketkandidaten `telegram_mode=mock-openai` oder `telegram_mode=live-frontier` in Package Acceptance. Der Workflow übergibt den aufgelösten `package-under-test`-Tarball an den Telegram-Pfad; der eigenständige Telegram-Workflow akzeptiert weiterhin eine veröffentlichte npm-Spezifikation für Prüfungen nach der Veröffentlichung.

## Automatisierung der regulären Release-Veröffentlichung

Für Beta, `latest`, Plugin, GitHub Release und Plattformveröffentlichung
ist `OpenClaw Release Publish` der normale verändernde Einstiegspunkt. Der monatliche
erweiterte stabile `.33+`-Gateway-Pfad verwendet diesen Orchestrator nicht. Der
reguläre Workflow orchestriert die Trusted-Publisher-Workflows in der für das
Release erforderlichen Reihenfolge:

1. Checken Sie das Release-Tag aus und ermitteln Sie dessen Commit-SHA.
2. Prüfen Sie, ob das Tag von `main` oder `release/*` aus erreichbar ist (oder für Alpha-Vorabversionen von einem Tideclaw-Alpha-Branch).
3. Führen Sie `pnpm plugins:sync:check` aus.
4. Lösen Sie `Plugin NPM Release` mit `publish_scope=all-publishable` und `ref=<release-sha>` aus.
5. Lösen Sie `Plugin ClawHub Release` mit demselben Umfang und derselben SHA aus.
6. Lösen Sie `OpenClaw NPM Release` mit dem Release-Tag, dem npm-Dist-Tag und dem gespeicherten `preflight_run_id` aus, nachdem Sie den gespeicherten `full_release_validation_run_id` und den exakten Ausführungsversuch geprüft haben.
7. Erstellen oder aktualisieren Sie für stabile Releases das GitHub-Release als Entwurf, lösen Sie `Windows Node Release` mit dem expliziten `windows_node_tag` und dem vom Kandidaten genehmigten `windows_node_installer_digests` aus und prüfen Sie die kanonischen Windows-Installer-/Prüfsummen-Assets. Lösen Sie außerdem `Android Release` aus, um die signierte APK für das exakte Tag sowie Prüfsumme und Provenienz zu erstellen. Prüfen Sie beide nativen Asset-Verträge, bevor Sie den Entwurf veröffentlichen.

Beispiel für eine Beta-Veröffentlichung:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Stabile Veröffentlichung mit dem standardmäßigen Beta-Dist-Tag:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Die direkte stabile Hochstufung zu `latest` erfolgt explizit:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

Verwenden Sie die untergeordneten Workflows `Plugin NPM Release` und `Plugin ClawHub Release` nur für gezielte Reparatur- oder Wiederveröffentlichungsarbeiten. `OpenClaw Release Publish` lehnt `plugin_publish_scope=selected` ab, wenn `publish_openclaw_npm=true`, damit das Kernpaket nicht ohne jedes veröffentlichungsfähige offizielle Plugin ausgeliefert werden kann, einschließlich `@openclaw/diffs-language-pack`. Legen Sie für eine ausgewählte Plugin-Reparatur `publish_openclaw_npm=false` mit `plugin_publish_scope=selected` und `plugins=@openclaw/name` fest oder lösen Sie den untergeordneten Workflow direkt aus.

Das ClawHub-Bootstrapping bei der Erstveröffentlichung ist die Ausnahme: Lösen Sie `Plugin ClawHub New`
vom vertrauenswürdigen `main` aus und übergeben Sie die vollständige SHA des Ziel-Releases über `ref`.
Führen Sie den Bootstrap-Workflow selbst niemals vom Release-Tag oder -Branch aus:

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

Die Validierung vor dem Taggen erfordert `dry_run=true`, lehnt Eingaben für Release-Tags und übergeordnete Ausführungen
ab und akzeptiert nur ein exaktes Ziel, das von `main` oder `release/*` aus erreichbar ist.
Sie lädt keine ClawHub-Anmeldedaten, veröffentlicht keine Paketbytes und ändert keine
Trusted-Publisher-Konfiguration. Der Workflow ermittelt dennoch den Live-Registry-Plan,
checkt das Ziel nur in einem geheimnisfreien Job aus und packt es, materialisiert die
gesperrte ClawHub-Toolchain und validiert das unveränderliche Artefakt sowie
Paket-Slug/-Identität, bevor das Release-Tag vorhanden ist. Genehmigen Sie die
`clawhub-plugin-bootstrap`-Umgebung erst, nachdem die geheimnisfreien Pack-Jobs
abgeschlossen sind; dieser geschützte Validierungsjob enthält weder Anmeldedaten noch verändernde Befehle.

Ein genehmigter Probelauf oder ein echtes Bootstrapping nach dem Taggen muss das exakte
Release-Tag sowie die ID, den Versuch und den
Branch der übergeordneten `OpenClaw Release Publish`-Ausführung enthalten. Die übergeordnete Ausführung bestätigt ihre eigene Workflow-SHA und eine separate exakte vertrauenswürdige
`main`-SHA für `Plugin ClawHub New`; die untergeordnete Ausführung und jede Genehmigung einer geschützten
Umgebung müssen mit dieser genehmigten untergeordneten SHA übereinstimmen. Das Release-Tag wird
vor jedem Veröffentlichungsversuch und jeder Trusted-Publisher-Änderung erneut geprüft.

Der Pack-Job
lädt ein unveränderliches Artefakt hoch, dessen Name, Actions-Artefakt-ID/-Digest,
erzeugende Ausführung/Versuch, Ziel-SHA und SHA-256/Größe des Tarballs je Paket
in die Validierungs- und geschützten Jobs übernommen werden. Der geschützte Job checkt ausschließlich vertrauenswürdige
`main`-Tools aus, validiert das Artefakt-Tupel über die GitHub-API, lädt
anhand der exakten Artefakt-ID herunter, bildet für jeden Tarball erneut den Hash und validiert lokale TAR-Pfade und
die Paketidentität anhand der USTAR-Kanonisierungsregeln der angehefteten CLI. Jeder
Kandidat durchläuft anschließend den Probelauf der angehefteten CLI zur Veröffentlichung, der vor
Registry-Abfrage oder Authentifizierung zurückkehrt. Der Vorfilter des Anmeldedaten-Jobs begrenzt komprimierte ClawPacks
auf 120 MiB, die gesamte Datei-Nutzlast auf 50 MiB, entpackte TAR-Daten auf 64 MiB und
die Anzahl der TAR-Einträge auf 10,000. Die Trusted-Publisher-Reparatur vorhandener Pakete bleibt
reine Konfiguration, packt jedoch weiterhin das Ziel und erfordert das angeforderte Tag
sowie die exakte Übereinstimmung von Registry-Bytes und -Metadaten, bevor die Trusted-Publisher-
Konfiguration geändert wird. Die Prüfung nach der Veröffentlichung lädt das ClawHub-Artefakt herunter und
erfordert dieselbe SHA-256 und Größe. Eine Wiederherstellung durch erneutes Ausführen fehlgeschlagener Jobs darf das Paketartefakt eines früheren
Versuchs nur wiederverwenden, wenn der exakte erzeugende Job erfolgreich
abgeschlossen wurde. Der abschließende Nachweis bindet außerdem die gesperrte ClawHub-Version, die Sperrdatei-
SHA-256 und die npm-Integrität ein. Eine Abweichung erfordert eine neue Paketversion.

## Eingaben des NPM-Workflows

`OpenClaw NPM Release` akzeptiert die folgenden vom Operator gesteuerten Eingaben:

- `tag`: erforderliches Release-Tag wie `v2026.4.2`, `v2026.4.2-1`, `v2026.4.2-beta.1` oder `v2026.4.2-alpha.1`; wenn `preflight_only=true`, darf es für einen ausschließlich validierenden Vorabtest auch die aktuelle vollständige 40-stellige Commit-SHA des Workflow-Branches sein
- `preflight_only`: `true` nur für Validierung/Build/Paket, `false` für den echten Veröffentlichungspfad
- `preflight_run_id`: ID einer vorhandenen erfolgreichen Vorabtest-Ausführung, auf dem echten Veröffentlichungspfad erforderlich, damit der Workflow den vorbereiteten Tarball wiederverwendet, statt ihn neu zu erstellen
- `full_release_validation_run_id`: ID einer erfolgreichen `Full Release Validation`-Ausführung für dieses Tag/diese SHA, für eine echte Veröffentlichung erforderlich. Beta-Veröffentlichungen dürfen allein auf Grundlage des Vorabtests mit einer Warnung fortgesetzt werden, aber die stabile/`latest`-Hochstufung erfordert sie weiterhin.
- `full_release_validation_run_attempt`: exakter positiver Ausführungsversuch, gekoppelt mit `full_release_validation_run_id`; erforderlich, wenn die Ausführungs-ID angegeben wird, damit Wiederholungsläufe den Autorisierungsnachweis während der Veröffentlichung nicht ändern können.
- `release_publish_run_id`: ID der genehmigten `OpenClaw Release Publish`-Ausführung; erforderlich, wenn dieser Workflow von dieser übergeordneten Ausführung ausgelöst wird (echte Veröffentlichungsaufrufe durch Bot-Akteure)
- `plugin_npm_run_id`: ID einer erfolgreichen Exact-Head-`Plugin NPM Release`-Ausführung; erforderlich für eine echte `extended-stable`-Veröffentlichung des Kernpakets
- `npm_dist_tag`: npm-Ziel-Tag für den Veröffentlichungspfad; akzeptiert `alpha`, `beta`, `latest` oder `extended-stable` und verwendet standardmäßig `beta`. Der endgültige Patch `33` und spätere müssen `extended-stable` verwenden; standardmäßig lehnt `extended-stable` frühere Patches ab und lehnt nicht endgültige Tags immer ab.
- `bypass_extended_stable_guard`: boolescher Wert nur für Tests, Standardwert `false`; umgeht mit `npm_dist_tag=extended-stable` die monatliche Berechtigung für den erweiterten stabilen Pfad, während Prüfungen der Release-Identität, des Artefakts, der Genehmigung und des Zurücklesens erhalten bleiben.

`Plugin NPM Release` akzeptiert `npm_dist_tag=default` für das bestehende Release-
Verhalten oder `npm_dist_tag=extended-stable` für den abgesicherten monatlichen Pfad. Die
Option für den erweiterten stabilen Pfad erfordert `publish_scope=all-publishable`, eine leere
`plugins`-Eingabe, einen endgültigen Patch ab `33` und den kanonischen
`extended-stable/YYYY.M.33`-Branch an dessen exakter Spitze. Sie verschiebt niemals die Plugin-
`latest` oder `beta`. Neue Paketversionen erhalten `extended-stable` atomar
über die vertrauenswürdige OIDC-Veröffentlichung (`npm publish --tag extended-stable`); dieser
Quell-Workflow verwendet kein tokenauthentifiziertes `npm dist-tag add`. Wiederholungsversuche
überspringen exakte Versionen, die bereits in npm vorhanden sind, und schlagen dann geschlossen fehl, sofern nicht das vollständige
Zurücklesen bestätigt, dass jedes exakte Paket und `extended-stable`-Tag konvergiert sind.

`OpenClaw Release Publish` akzeptiert die folgenden vom Operator gesteuerten Eingaben:

- `tag`: erforderliches Release-Tag; muss bereits vorhanden sein
- `preflight_run_id`: ID einer erfolgreichen `OpenClaw NPM Release`-Vorabtest-Ausführung; erforderlich, wenn `publish_openclaw_npm=true` oder `plugin_publish_scope=all-publishable`
- `full_release_validation_run_id`: ID einer erfolgreichen `Full Release Validation`-Ausführung; erforderlich, wenn `publish_openclaw_npm=true` oder `plugin_publish_scope=all-publishable`
- `full_release_validation_run_attempt`: exakter positiver Versuch, gekoppelt mit `full_release_validation_run_id`; erforderlich, wenn die Ausführungs-ID angegeben wird
- `windows_node_tag`: exaktes Nicht-Vorabversions-`openclaw/openclaw-windows-node`-Release-Tag; für die stabile OpenClaw-Veröffentlichung erforderlich
- `windows_node_installer_digests`: vom Kandidaten genehmigte kompakte JSON-Zuordnung der aktuellen Windows-Installer-Namen zu ihren angehefteten `sha256:`-Digests; für die stabile OpenClaw-Veröffentlichung erforderlich
- `npm_telegram_run_id`: optionale ID einer erfolgreichen `NPM Telegram Beta E2E`-Ausführung, die in den abschließenden Release-Nachweis aufgenommen werden soll
- `npm_dist_tag`: npm-Ziel-Tag für das OpenClaw-Paket, eines von `alpha`, `beta` oder `latest`
- `plugin_publish_scope`: verwendet standardmäßig `all-publishable`; verwenden Sie `selected` nur für gezielte reine Plugin-Reparaturarbeiten mit `publish_openclaw_npm=false`
- `plugins`: durch Kommas getrennte `@openclaw/*`-Paketnamen, wenn `plugin_publish_scope=selected`
- `publish_openclaw_npm`: verwendet standardmäßig `true`; legen Sie `false` nur fest, wenn Sie den Workflow als reinen Plugin-Reparatur-Orchestrator verwenden
- `release_profile`: Release-Abdeckungsprofil für Zusammenfassungen der Release-Nachweise; verwendet standardmäßig `from-validation`, wodurch es aus dem Validierungsmanifest gelesen wird, oder überschreiben Sie es mit `beta`, `stable` oder `full`
- `wait_for_clawhub`: verwendet standardmäßig `false`, damit die npm-Verfügbarkeit nicht durch den ClawHub-Sidecar blockiert wird; legen Sie `true` nur fest, wenn der Abschluss des Workflows den Abschluss von ClawHub einschließen muss

`OpenClaw Release Checks` akzeptiert die folgenden vom Operator gesteuerten Eingaben:

- `ref`: Branch, Tag oder vollständiger Commit-SHA für die Validierung. Prüfungen mit Geheimnissen setzen voraus, dass der aufgelöste Commit von einem OpenClaw-Branch oder Release-Tag aus erreichbar ist.
- `run_release_soak`: aktiviert für Beta-Release-Prüfungen umfassende Live-/E2E-Prüfungen, den Docker-Release-Pfad sowie einen Dauertest aller seitdem hinzugekommenen Upgrade-Survivor. Wird durch `release_profile=stable` und `release_profile=full` erzwungen.

Regeln:

- Reguläre finale Versionen und Korrekturversionen unterhalb von Patch `33` dürfen entweder unter `beta` oder `latest` veröffentlicht werden. Finale Versionen ab Patch `33` müssen unter `extended-stable` veröffentlicht werden; Versionen mit Korrektursuffix an dieser Grenze werden abgelehnt.
- Beta-Prerelease-Tags dürfen nur unter `beta` veröffentlicht werden; Alpha-Prerelease-Tags dürfen nur unter `alpha` veröffentlicht werden.
- Für `OpenClaw NPM Release` ist die Eingabe eines vollständigen Commit-SHA nur zulässig, wenn `preflight_only=true`
- `OpenClaw Release Checks` und `Full Release Validation` dienen immer ausschließlich der Validierung.
- Der tatsächliche Veröffentlichungspfad muss denselben `npm_dist_tag` verwenden wie der Preflight; der Workflow überprüft diese Metadaten, bevor die Veröffentlichung fortgesetzt wird.

## Reguläre Release-Abfolge für Beta/neueste stabile Version

Diese bisherige Abfolge gilt für das reguläre orchestrierte Release, das auch Plugins, GitHub Release, Windows und weitere Plattformarbeiten umfasst. Sie gilt nicht für den oben auf dieser Seite dokumentierten monatlichen erweiterten stabilen Gateway-Pfad `.33+`.

Beim Erstellen eines regulären orchestrierten stabilen Releases:

1. Führen Sie `OpenClaw NPM Release` mit `preflight_only=true` aus. Solange noch kein Tag existiert, können Sie den aktuellen vollständigen Commit-SHA des Workflow-Branches für einen ausschließlich der Validierung dienenden Probelauf des Preflight-Workflows verwenden.
2. Wählen Sie `npm_dist_tag=beta` für den normalen Ablauf mit vorheriger Beta oder `latest` nur dann, wenn Sie bewusst direkt eine stabile Version veröffentlichen möchten.
3. Führen Sie `Full Release Validation` auf dem Release-Branch, dem Release-Tag oder dem vollständigen Commit-SHA aus, wenn Sie die normale CI zusammen mit Live-Abdeckung für Prompt-Cache, Docker, QA Lab, Matrix und Telegram über einen einzigen manuellen Workflow ausführen möchten. Wenn Sie bewusst nur den deterministischen normalen Testgraphen benötigen, führen Sie stattdessen den manuellen Workflow `CI` auf der Release-Referenz aus.
4. Wählen Sie genau den Nicht-Prerelease-Release-Tag `openclaw/openclaw-windows-node` aus, dessen signierte x64- und ARM64-Installationsprogramme ausgeliefert werden sollen. Speichern Sie ihn als `windows_node_tag` und die validierte Digest-Zuordnung der Installationsprogramme als `windows_node_installer_digests`. Das Release-Candidate-Hilfsprogramm erfasst beide Werte und nimmt sie in den generierten Veröffentlichungsbefehl auf.
5. Speichern Sie die erfolgreichen Werte `preflight_run_id`, `full_release_validation_run_id` und den exakten Wert `full_release_validation_run_attempt`.
6. Führen Sie `OpenClaw Release Publish` aus dem vertrauenswürdigen `main` mit demselben `tag`, demselben `npm_dist_tag`, dem ausgewählten `windows_node_tag`, dem dafür gespeicherten `windows_node_installer_digests`, dem gespeicherten `preflight_run_id`, `full_release_validation_run_id` und `full_release_validation_run_attempt` aus. Dadurch werden externalisierte Plugins auf npm und ClawHub veröffentlicht, bevor das OpenClaw-npm-Paket hochgestuft wird.
7. Wenn das Release unter `beta` veröffentlicht wurde, verwenden Sie den Workflow `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml`, um diese stabile Version von `beta` nach `latest` hochzustufen.
8. Wenn das Release bewusst direkt unter `latest` veröffentlicht wurde und `beta` unmittelbar auf denselben stabilen Build verweisen soll, verwenden Sie denselben Release-Workflow, um beide Dist-Tags auf die stabile Version zu setzen, oder lassen Sie `beta` später durch dessen geplante selbstheilende Synchronisierung verschieben.

Die Dist-Tag-Änderung befindet sich im Release-Ledger-Repository, da sie weiterhin `NPM_TOKEN` benötigt, während das Quell-Repository ausschließlich per OIDC veröffentlicht. Dadurch bleiben sowohl der direkte Veröffentlichungspfad als auch der Hochstufungspfad mit vorheriger Beta dokumentiert und für Bedienende sichtbar.

Wenn ein Maintainer auf lokale npm-Authentifizierung zurückgreifen muss, führen Sie alle Befehle der 1Password CLI (`op`) ausschließlich in einer dafür vorgesehenen tmux-Sitzung aus. Rufen Sie `op` nicht direkt aus der Haupt-Shell des Agenten auf. Die Ausführung innerhalb von tmux macht Eingabeaufforderungen, Warnmeldungen und die OTP-Verarbeitung sichtbar und verhindert wiederholte Host-Warnmeldungen.

## Öffentliche Referenzen

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Maintainer verwenden die privaten Release-Dokumente unter [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) als tatsächliches Runbook.

## Verwandte Themen

- [Release-Kanäle](/de/install/development-channels)
