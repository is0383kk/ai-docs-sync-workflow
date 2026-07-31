---
read_when:
    - Sie möchten einen Quellcode-Checkout sicher aktualisieren
    - Sie debuggen die Ausgabe oder Optionen von `openclaw update`
    - Sie müssen das Kurzschreibweisenverhalten von `--update` verstehen
summary: CLI-Referenz für `openclaw update` (weitgehend sichere Aktualisierung aus dem Quellcode + automatischer Neustart des Gateways)
title: Aktualisieren
x-i18n:
    generated_at: "2026-07-26T18:19:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

Aktualisieren Sie OpenClaw und wechseln Sie zwischen den Kanälen stable/extended-stable/beta/dev.

Wenn Sie die Installation über **npm/pnpm/bun** vorgenommen haben (globale Installation, keine Git-Metadaten),
erfolgen Aktualisierungen über den unter
[Aktualisieren](/de/install/updating) beschriebenen Paketmanager-Ablauf.

## Verwendung

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update` wird in `openclaw update` umgeschrieben (nützlich für Shells und
Launcher-Skripte).

## Optionen

| Flag                                             | Beschreibung                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | Überspringt den Neustart des Gateway-Dienstes nach einer erfolgreichen Aktualisierung. Bei Paketmanager-Aktualisierungen mit Neustart wird überprüft, ob der neu gestartete Dienst die erwartete Version meldet, bevor der Befehl erfolgreich abgeschlossen wird.                                                                                |
| `--channel <stable\|extended-stable\|beta\|dev>` | Legt den Aktualisierungskanal fest und speichert ihn nach erfolgreicher Aktualisierung des Kerns dauerhaft. Extended-stable ist nur für Paketinstallationen verfügbar.                                                                                                                                                                          |
| `--tag <dist-tag\|version\|spec>`                | Überschreibt das Paketziel nur für diese Aktualisierung. Dies kann nicht mit einem wirksamen Kanal `extended-stable` kombiniert werden, dessen verifiziertes exaktes Ziel obligatorisch ist. Bei anderen Paketinstallationen wird `main` auf `github:openclaw/openclaw#main` abgebildet; GitHub-/Git-Quellspezifikationen werden vor der gestaffelten globalen npm-Installation in ein temporäres Tarball gepackt. |
| `--dry-run`                                      | Zeigt eine Vorschau der geplanten Aktionen (Kanal/Tag/Ziel/Neustartablauf) an, ohne die Konfiguration zu schreiben, etwas zu installieren, Plugins zu synchronisieren oder einen Neustart durchzuführen.                                                                                                                                          |
| `--json`                                         | Gibt maschinenlesbares `UpdateRunResult`-JSON aus. Enthält `postUpdate.plugins.warnings`, wenn ein verwaltetes Plugin repariert werden muss, Details zum Plugin-Fallback des Beta-Kanals und `postUpdate.plugins.integrityDrifts`, wenn bei der Synchronisierung nach der Aktualisierung eine Abweichung des npm-Plugin-Artefakts erkannt wird.                                     |
| `--timeout <seconds>`                            | Zeitlimit pro Schritt. Standardwert: `1800`.                                                                                                                                                                                                                                                                                      |
| `--yes`                                          | Überspringt Bestätigungsaufforderungen (beispielsweise die Bestätigung eines Downgrades).                                                                                                                                                                                                                                                     |
| `--acknowledge-clawhub-risk`                     | Ermöglicht der Plugin-Synchronisierung nach der Aktualisierung, Warnungen zur Vertrauenswürdigkeit von Community-Paketen auf ClawHub ohne interaktive Aufforderung zu übergehen. Ohne diese Option werden riskante Community-Versionen übersprungen und unverändert belassen, wenn OpenClaw keine Aufforderung anzeigen kann. Offizielle ClawHub-Pakete und gebündelte Plugin-Quellen umgehen diese Aufforderung. |

Es gibt kein Flag `--verbose`. Verwenden Sie `--dry-run` für eine Vorschau der geplanten Aktionen,
`--json` für maschinenlesbare Ergebnisse und `openclaw update status --json`
nur für Kanal und Verfügbarkeit. Die Ausführlichkeit der Gateway-Konsole (`--verbose`) und
die Protokollierungsstufe für Dateien (`logging.level: "debug"`/`"trace"`) sind unabhängige Einstellungen; siehe
[Gateway-Protokollierung](/de/gateway/logging).

<Note>
Im Nix-Modus (`OPENCLAW_NIX_MODE=1`) sind verändernde Ausführungen von `openclaw update` deaktiviert. Aktualisieren Sie stattdessen die Nix-Quelle oder die Flake-Eingabe für diese Installation; verwenden Sie für nix-openclaw den Agent-zentrierten [Schnellstart](https://github.com/openclaw/nix-openclaw#quick-start). `openclaw update status` und `openclaw update --dry-run` bleiben schreibgeschützt.
</Note>

<Warning>
Downgrades erfordern eine Bestätigung, da ältere Versionen die Konfiguration beschädigen können.
Wenn die Installation Sitzungen bereits zu SQLite migriert hat, stellen Sie archivierte ältere
Transkriptartefakte wieder her, bevor Sie eine ältere dateibasierte Version starten. Siehe
[Doctor: Downgrade nach der SQLite-Sitzungsmigration](/de/cli/doctor#downgrading-after-session-sqlite-migration).
</Warning>

## `update status`

Zeigt den aktiven Aktualisierungskanal, das Git-Tag, den Branch und den SHA (nur bei Quellcode-Checkouts)
sowie die Verfügbarkeit von Aktualisierungen an.

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| Flag                  | Standardwert | Beschreibung                         |
| --------------------- | ------------ | ------------------------------------ |
| `--json`              | `false` | Gibt maschinenlesbares Status-JSON aus. |
| `--timeout <seconds>` | `3`     | Zeitlimit für Prüfungen.                 |

Bei Extended-stable-Paketinstallationen verwendet die Statusprüfung denselben öffentlichen Selektor
und dieselbe Prüfung des exakten Pakets wie eine Aktualisierung im Vordergrund. Sie kann
`ahead of extended-stable` melden, wenn die installierte Version neuer ist. JSON-Fehler
enthalten `registry.reason` (`selector_missing`, `selector_query_failed`,
`exact_package_mismatch` oder `unsupported_git_channel`).

## `update repair`

Führt den Abschluss der Aktualisierung erneut aus, nachdem das Kernpaket bereits geändert wurde, spätere
Reparaturarbeiten jedoch nicht ordnungsgemäß abgeschlossen wurden. Dies ist der unterstützte Wiederherstellungspfad, wenn
`openclaw update` das neue Kernpaket installiert hat, die anschließende Plugin-Synchronisierung,
die Metadaten verwalteter npm-Plugins, die Aktualisierung der Registry oder die Doctor-Reparatur jedoch
nicht konvergiert sind.

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| Flag                                             | Beschreibung                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | Speichert den Aktualisierungskanal des Kerns vor der Reparatur dauerhaft. Bei Extended-stable verwenden geeignete offizielle npm-Plugins mit unveränderter/standardmäßiger oder `latest`-Zielvorgabe die exakt installierte Kernversion als Ziel. Die Extended-stable-Reparatur wird bei Git-Checkouts abgelehnt, ohne die Konfiguration zu ändern. |
| `--json`                                         | Gibt maschinenlesbares JSON zum Abschluss aus.                                                                                                                                                                                                                       |
| `--timeout <seconds>`                            | Zeitlimit für Reparaturschritte. Standardwert: `1800`.                                                                                                                                                                                                   |
| `--yes`                                          | Überspringt Bestätigungsaufforderungen.                                                                                                                                                                                                                              |
| `--acknowledge-clawhub-risk`                     | Dasselbe Verhalten wie bei `openclaw update`.                                                                                                                                                                                                                       |
| `--no-restart`                                   | Wird zur Wahrung der Gleichheit akzeptiert; die Reparatur startet das Gateway niemals neu.                                                                                                                                                                           |

`update repair` führt `openclaw doctor --fix` aus, lädt die reparierte Konfiguration und
die Installationsdatensätze neu, synchronisiert nachverfolgte Plugins für den aktiven Aktualisierungskanal, aktualisiert
verwaltete npm-Plugin-Installationen, repariert fehlende konfigurierte Plugin-Nutzdaten,
aktualisiert die Plugin-Registry und schreibt konvergierte Metadaten der Installationsdatensätze.
Es installiert kein neues Kernpaket und startet das Gateway nicht neu.

## `update wizard`

Interaktiver Ablauf zur Auswahl eines Aktualisierungskanals und zur Bestätigung, ob das
Gateway anschließend neu gestartet werden soll (standardmäßig erfolgt ein Neustart). Wenn `dev` ohne Git-
Checkout ausgewählt wird, wird angeboten, einen zu erstellen.

| Flag                  | Standardwert | Beschreibung                       |
| --------------------- | ------------ | ---------------------------------- |
| `--timeout <seconds>` | `1800`  | Zeitlimit für jeden Aktualisierungsschritt. |

## Funktionsweise

Beim expliziten Wechseln des Kanals (`--channel ...`) wird auch die Installationsmethode
entsprechend angepasst:

- `dev` -> stellt einen Git-Checkout sicher (standardmäßig `~/openclaw` oder
  `$OPENCLAW_HOME/openclaw`, wenn `OPENCLAW_HOME` festgelegt ist; Überschreibung mit
  `OPENCLAW_GIT_DIR`), aktualisiert ihn und installiert die globale CLI aus diesem
  Checkout.
- `stable` -> installiert aus npm unter Verwendung von `latest`.
- `extended-stable` -> löst den öffentlichen npm-Selektor `extended-stable` auf,
  überprüft das exakt ausgewählte Paket und installiert genau diese Version. Es
  erfolgt kein Fallback auf einen anderen Selektor, und die Option wird bei Git-Checkouts abgelehnt.
- `beta` -> bevorzugt das npm-Dist-Tag `beta` und greift auf `latest` zurück, wenn die Beta-Version
  fehlt oder älter als die aktuelle stabile Version ist.

### Neustartübergabe

Der automatische Aktualisierer des Gateway-Kerns (wenn über die Konfiguration aktiviert) startet den CLI-
Aktualisierungspfad außerhalb des Anfrage-Handlers des laufenden Gateways. Paketmanager-Aktualisierungen der Steuerungsebene
`update.run` und überwachte Aktualisierungen von Git-Checkouts verwenden
dieselbe Übergabe an den verwalteten Dienst, anstatt den Paketbaum zu ersetzen oder
`dist/` innerhalb des laufenden Gateway-Prozesses neu zu erstellen: Das Gateway startet einen
abgetrennten Hilfsprozess und wird beendet; dieser Hilfsprozess führt `openclaw update --yes --json`
außerhalb des Gateway-Prozessbaums aus. Wenn die Übergabe nicht verfügbar ist,
gibt `update.run` eine strukturierte Antwort mit dem sicheren Shell-Befehl zurück, der
manuell ausgeführt werden kann.

Gespeicherte Extended-Stable-Auswahlen erhalten schreibgeschützte Start- und 24-stündige
Aktualisierungshinweise, wenn `update.checkOnStart` aktiviert ist. Diese Prüfungen wenden niemals eine Aktualisierung an,
starten keine Übergabe, starten das Gateway nicht neu, verwenden weder Stable-Verzögerung/-Jitter noch den
Beta-Abfragezyklus. Explizite Vordergrundaktualisierungen, einfache Vordergrundaktualisierungen mit
gespeichertem `update.channel: "extended-stable"`, Statusabfragen bei Bedarf und deren verwaltete
Gateway-Übergabe werden weiterhin unterstützt.

Wenn ein lokaler verwalteter Gateway-Dienst installiert und der Neustart aktiviert ist,
stoppen Paketmanager- und Git-Checkout-Aktualisierungen den laufenden Dienst, bevor
der Paketbaum ersetzt oder die Checkout-/Build-Ausgabe geändert wird. Das Aktualisierungsprogramm
aktualisiert anschließend die Dienstmetadaten, startet den Dienst neu und überprüft das
neu gestartete Gateway, bevor `Gateway: restarted and verified.` gemeldet wird.
Paketmanager-Aktualisierungen überprüfen zusätzlich, ob das neu gestartete Gateway die
erwartete Paketversion meldet; Git-Checkout-Aktualisierungen überprüfen nach dem erneuten Build
den Gateway-Zustand und die Dienstbereitschaft.

Paketmanager-Aktualisierungen verwenden normalerweise weiterhin die im
verwalteten Dienst aufgezeichnete Node-Binärdatei. Wenn dieses Node die Zielversion nicht ausführen kann,
das aktuelle CLI-Node dies jedoch kann und der Dienst nachweislich zu dem aktualisierten Paket gehört,
verwendet eine Aktualisierung mit aktiviertem Neustart das aktuelle Node für den Abschluss und schreibt
die Dienstmetadaten für diese Laufzeit neu. `--no-restart` kann Dienstmetadaten nicht reparieren,
daher führt dieselbe Laufzeitabweichung vor der Paketänderung zum Abbruch.

Unter macOS überprüft die Prüfung nach der Aktualisierung außerdem, ob der LaunchAgent für das
aktive Profil geladen/aktiv ist und der konfigurierte Loopback-Port ordnungsgemäß funktioniert.
Wenn die plist installiert ist, aber nicht von launchd überwacht wird, bootstrappt OpenClaw
den LaunchAgent automatisch neu und führt die Zustands-/Versions-/
Kanalbereitschaftsprüfungen erneut aus (ein frischer Bootstrap lädt den `RunAtLoad`-Job direkt,
sodass die Wiederherstellung das neu gestartete Gateway nicht sofort `kickstart -k`). Wenn
das Gateway weiterhin keinen ordnungsgemäßen Zustand erreicht, wird der Befehl mit einem von null
abweichenden Status beendet und gibt den Pfad zum Neustartprotokoll sowie Anweisungen für Neustart,
Neuinstallation und Paket-Rollback aus.

Wenn der Neustart nicht ausgeführt werden kann, gibt der Befehl `Gateway: restart skipped (...)` oder
`Gateway: restart failed: ...` mit einem Hinweis zur manuellen Ausführung von `openclaw gateway restart` aus.
Mit `--no-restart` werden der Paketaustausch oder der erneute Git-Build weiterhin ausgeführt,
der verwaltete Dienst wird jedoch weder gestoppt noch neu gestartet, sodass das laufende Gateway
den alten Code weiterverwendet, bis Sie es manuell neu starten.

### Form der Control-Plane-Antwort

Wenn `update.run` über die Gateway-Control-Plane bei einer Paketmanager-
Installation oder einem überwachten Git-Checkout ausgeführt wird, meldet der Handler die Einleitung
der Übergabe getrennt von der CLI-Aktualisierung, die nach dem Beenden des Gateways fortgesetzt wird:

- `ok: true`, `result.status: "skipped"`,
  `result.reason: "managed-service-handoff-started"` und
  `handoff.status: "started"`: Das Gateway hat die Übergabe an den verwalteten Dienst erstellt
  und seinen eigenen Neustart geplant, damit der getrennte Helfer
  `openclaw update --yes --json` außerhalb des laufenden Dienstprozesses ausführen kann.
- `ok: false`, `result.reason: "managed-service-handoff-unavailable"` und
  `handoff.status: "unavailable"`: OpenClaw konnte keine überwachende
  Dienstgrenze und dauerhafte Dienstidentität für eine sichere Übergabe finden (beispielsweise
  erfordert die systemd-Übergabe die Identität der `OPENCLAW_SYSTEMD_UNIT`-Unit,
  nicht nur vorhandene systemd-Prozessmarker). Die Antwort enthält
  `handoff.command`, den Shell-Befehl, der außerhalb des Gateways auszuführen ist.
- `ok: false`, `result.reason: "managed-service-handoff-failed"`: Das Gateway
  hat versucht, die Übergabe zu erstellen, konnte den getrennten Helfer jedoch nicht starten.

Die `sentinel`-Nutzlast wird geschrieben, bevor das Gateway beendet wird, und die CLI-
Übergabe aktualisiert denselben Neustart-Sentinel, nachdem die Zustandsprüfungen nach dem
Neustart des verwalteten Dienstes abgeschlossen sind. Während der Übergabe kann der Sentinel
`stats.reason: "restart-health-pending"` ohne Erfolgsfortsetzung enthalten; das
neu gestartete Gateway fragt ihn ab und löst die Fortsetzung erst aus, nachdem die CLI
den Dienstzustand überprüft und den Sentinel mit dem endgültigen `ok`-Ergebnis neu geschrieben hat.
`openclaw status` und `openclaw status --all` zeigen eine `Update restart`-Zeile,
solange dieser Sentinel aussteht oder fehlgeschlagen ist, und `update.status` aktualisiert
den Sentinel und gibt seinen neuesten Stand zurück.

## Git-Checkout-Ablauf

### Kanalauswahl

- `stable`: Checkt das neueste Nicht-Beta-Tag aus und führt anschließend Build und Doctor aus.
- `beta`: Bevorzugt das neueste `-beta`-Tag und greift auf das neueste Stable-Tag zurück,
  wenn Beta fehlt oder älter ist.
- `dev`: Checkt `main` aus und führt anschließend Fetch und Rebase aus.
- `extended-stable`: Wird für Git-Checkouts nicht unterstützt; der Checkout
  wird nicht geändert.

### Aktualisierungsschritte

<Steps>
  <Step title="Sauberen Arbeitsbaum überprüfen">
    Erfordert, dass keine nicht committeten Änderungen vorhanden sind.
  </Step>
  <Step title="Kanal wechseln">
    Wechselt zum ausgewählten Kanal (Tag oder Branch).
  </Step>
  <Step title="Upstream abrufen">
    Nur für Dev.
  </Step>
  <Step title="Build-Vorabprüfung (nur Dev)">
    Führt den TypeScript-Build in einem temporären Arbeitsbaum aus. Wenn die Spitze fehlschlägt, werden bis zu 10 Commits zurückverfolgt, um den neuesten buildfähigen Commit zu finden. Setzen Sie `OPENCLAW_UPDATE_PREFLIGHT_LINT=1`, um bei dieser Vorabprüfung zusätzlich Lint auszuführen; Lint läuft in einem eingeschränkten seriellen Modus, da Hosts für Benutzeraktualisierungen häufig kleiner als CI-Runner sind.
  </Step>
  <Step title="Rebase ausführen">
    Führt einen Rebase auf den ausgewählten Commit aus (nur Dev).
  </Step>
  <Step title="Abhängigkeiten installieren">
    Verwendet den Paketmanager des Repositorys. Bei pnpm-Checkouts bootstrappt das Aktualisierungsprogramm `pnpm` bei Bedarf (zuerst über `corepack`, dann über einen temporären `npm install pnpm@11`-Fallback), statt `npm run build` innerhalb eines pnpm-Arbeitsbereichs auszuführen. Wenn der pnpm-Bootstrap weiterhin fehlschlägt, bricht das Aktualisierungsprogramm frühzeitig mit einem paketmanagerspezifischen Fehler ab, statt `npm run build` im Checkout zu versuchen.
  </Step>
  <Step title="Control UI bauen">
    Baut das Gateway und die Control UI.
  </Step>
  <Step title="Doctor ausführen">
    `openclaw doctor` wird als abschließende Prüfung für eine sichere Aktualisierung ausgeführt.
  </Step>
  <Step title="Plugins synchronisieren">
    Synchronisiert Plugins mit dem aktiven Kanal. Dev verwendet gebündelte Plugins; Stable und Beta verwenden npm. Aktualisiert nachverfolgte Plugin-Installationen.
  </Step>
</Steps>

### Details zur Plugin-Synchronisierung

Im Beta-Kanal versuchen nachverfolgte npm- und ClawHub-Plugin-Installationen, die der
Default-/Latest-Linie folgen, zuerst eine Plugin-Version `@beta`. Wenn für das Plugin keine
Beta-Version verfügbar ist, greift OpenClaw auf die aufgezeichnete Default-/Latest-Spezifikation zurück
und meldet eine Warnung. Bei npm-Plugins greift OpenClaw auch dann zurück, wenn das Beta-
Paket vorhanden ist, die Installationsvalidierung jedoch fehlschlägt. Diese Fallback-Warnungen führen
nicht zum Fehlschlagen der Kernaktualisierung. Exakte Versionen und explizite Tags werden niemals neu geschrieben.

<Warning>
Wenn eine exakt angeheftete npm-Plugin-Aktualisierung zu einem Artefakt aufgelöst wird, dessen Integrität vom gespeicherten Installationsdatensatz abweicht, bricht `openclaw update` die Aktualisierung dieses Plugin-Artefakts ab, statt es zu installieren. Installieren oder aktualisieren Sie das Plugin erst dann explizit neu, nachdem Sie überprüft haben, dass Sie dem neuen Artefakt vertrauen.
</Warning>

<Note>
Fehler bei der Plugin-Synchronisierung nach der Aktualisierung, die auf ein verwaltetes Plugin beschränkt sind und die der Synchronisierungspfad umgehen kann (beispielsweise eine nicht erreichbare npm-Registry für ein nicht wesentliches Plugin), werden als Warnungen gemeldet, nachdem die Kernaktualisierung erfolgreich abgeschlossen wurde. Das JSON-Ergebnis behält auf oberster Ebene den Aktualisierungswert `status: "ok"` bei und meldet `postUpdate.plugins.status: "warning"` mit Hinweisen zu `openclaw update repair` und `openclaw plugins inspect <id> --runtime --json`. Unerwartete Ausnahmen des Aktualisierungsprogramms oder der Synchronisierung lassen das Aktualisierungsergebnis weiterhin fehlschlagen. Beheben Sie den Fehler bei der Plugin-Installation oder -Aktualisierung und führen Sie anschließend `openclaw update repair` erneut aus. Wenn eine fehlgeschlagene Aktualisierung ein verwaltetes Plugin unbrauchbar macht, deaktiviert OpenClaw dessen Laufzeiteintrag und setzt aktive Slots zurück, ohne die vom Operator verfasste Richtlinie `plugins.allow` oder `plugins.deny` zu ändern.

Nach dem Synchronisierungsschritt für jedes Plugin führt `openclaw update` vor dem Neustart des Gateways einen obligatorischen **Konvergenzdurchlauf nach der Kernaktualisierung** aus: Fehlende Nutzlasten konfigurierter Plugins werden repariert, jeder _aktive_ nachverfolgte Installationsdatensatz auf dem Datenträger wird validiert und es wird statisch überprüft, ob sein `package.json` geparst werden kann (und ob jeder explizit deklarierte `main` vorhanden ist). Fehler aus diesem Durchlauf sowie ein ungültiger Konfigurations-Snapshot geben `postUpdate.plugins.status: "error"` zurück und ändern den Aktualisierungswert `status` auf oberster Ebene in `"error"`, sodass `openclaw update` mit einem von null abweichenden Status beendet und das Gateway _nicht_ mit einem ungeprüften Plugin-Satz neu gestartet wird. Der Fehler enthält strukturierte `postUpdate.plugins.warnings[].guidance`-Zeilen, die auf `openclaw update repair` und `openclaw plugins inspect <id> --runtime --json` verweisen. Deaktivierte Plugin-Einträge und Datensätze, die keine mit vertrauenswürdigen Quellen verknüpften offiziellen Synchronisierungsziele sind, werden hier übersprungen (entsprechend der `skipDisabledPlugins`-Richtlinie der Prüfung auf fehlende Nutzlasten), sodass ein veralteter Datensatz eines deaktivierten Plugins eine ansonsten gültige Aktualisierung nicht blockieren kann.

Wenn das aktualisierte Gateway startet, dient das Laden von Plugins ausschließlich der Überprüfung: Beim Start werden keine Paketmanager ausgeführt und keine Abhängigkeitsbäume geändert. Paketmanager-Neustarts über `update.run` werden an den CLI-Pfad für verwaltete Dienste übergeben, sodass der Paketaustausch außerhalb des alten Gateway-Prozesses erfolgt und die Dienstzustandsprüfungen entscheiden, ob die Aktualisierung als abgeschlossen gemeldet werden kann.
</Note>

Nach erfolgreicher Extended-Stable-Kernaktualisierung zielen die Plugin-Integritäts-
und Konvergenzprüfungen nach der Kernaktualisierung auf geeignete offizielle npm-Plugins
mit exakt der installierten Kernversion. Bei Default-/`latest`-Absicht fragt OpenClaw
weder Plugin-`@extended-stable` ab noch greift es auf npm-`latest` zurück; die Paketversion
wird aus dem installierten Kern abgeleitet. Explizite Versionsanheftungen, explizite Nicht-`latest`-Tags,
Drittanbieterpakete und Nicht-npm-Quellen behalten ihre bestehende Absicht bei.

Bei Paketmanager-Installationen löst `openclaw update` die Zielpaketversion auf,
bevor der Paketmanager aufgerufen wird. Globale npm-Installationen verwenden eine gestufte
Installation: OpenClaw installiert das neue Paket in einem temporären npm-Präfix,
lässt das Kandidatenpaket während `preinstall` die Node-Version des Hosts validieren
und überprüft dort das paketierte `dist`-Inventar. Ein paketierter Abschlusswächter
bleibt außerhalb dieses Inventars, bis `preinstall` erfolgreich abgeschlossen ist, sodass Paketmanager,
die Lebenszyklusskripte überspringen, ebenfalls vor der Aktivierung anhalten. Unter npm 12 und neuer
genehmigt das Aktualisierungsprogramm ausschließlich den Lebenszyklus des OpenClaw-Kandidaten;
Skripte transitiver Abhängigkeiten bleiben blockiert. OpenClaw tauscht anschließend den sauberen Paketbaum
in das tatsächliche globale Präfix ein. Wenn die Überprüfung fehlschlägt, werden Doctor nach der Aktualisierung,
Plugin-Synchronisierung und Neustartarbeiten nicht aus dem verdächtigen Baum ausgeführt. Selbst wenn die
installierte Version bereits dem Ziel entspricht, aktualisiert der Befehl die globale Paketinstallation
und führt anschließend die Plugin-Synchronisierung, eine Aktualisierung der Vervollständigung für Kernbefehle
und Neustartarbeiten aus. Dadurch bleiben paketierte Sidecars und kanaleigene
Plugin-Datensätze mit dem installierten OpenClaw-Build abgestimmt, während vollständige
Vervollständigungs-Neubuilds für Plugin-Befehle expliziten
Ausführungen von `openclaw completion --write-state` vorbehalten bleiben.

## Verwandte Themen

- `openclaw doctor` (bietet bei Git-Checkouts an, zuerst die Aktualisierung auszuführen)
- [Entwicklungskanäle](/de/install/development-channels)
- [Aktualisierung](/de/install/updating)
- [CLI-Referenz](/de/cli)
