---
read_when:
    - Sie möchten ein vollwertiges Backup-Archiv für den lokalen OpenClaw-Zustand
    - Sie benötigen einen kompakten, verifizierten Snapshot einer OpenClaw-SQLite-Datenbank
    - Sie möchten vor dem Zurücksetzen oder Deinstallieren eine Vorschau der einzubeziehenden Pfade anzeigen.
summary: CLI-Referenz für `openclaw backup` (Archive und SQLite-Snapshots)
title: Sicherung
x-i18n:
    generated_at: "2026-07-26T18:21:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dfb5a118545589b181cede26dab72e9d029d98a1cac5cfccedd9d9cf2c56d3b5
    source_path: cli/backup.md
    workflow: 16
---

# `openclaw backup`

Erstellen Sie ein lokales Sicherungsarchiv für OpenClaw-Status, -Konfiguration, Authentifizierungsprofile, Kanal-/Provider-Anmeldedaten, Sitzungen und optional Arbeitsbereiche.

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz
openclaw backup sqlite create --global --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite create --agent main --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite list --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id>
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id> --scratch ~/Private/openclaw-scratch
openclaw backup sqlite restore ~/Backups/openclaw-sqlite/<snapshot-id> --target ./restored/openclaw.sqlite
```

## Hinweise

- Das Archiv enthält eine eingebettete `manifest.json` mit den aufgelösten Quellpfaden und dem Archivlayout.
- Standardmäßig wird im aktuellen Arbeitsverzeichnis ein mit einem Zeitstempel versehenes `.tar.gz`-Archiv erstellt. Dateinamen mit Zeitstempel verwenden die lokale Zeitzone Ihres Rechners und enthalten den UTC-Versatz. Wenn sich das aktuelle Arbeitsverzeichnis innerhalb eines gesicherten Quellbaums befindet, verwendet OpenClaw für den standardmäßigen Archivspeicherort stattdessen Ihr Home-Verzeichnis.
- Vorhandene Archivdateien werden niemals überschrieben. Ausgabepfade innerhalb der Quellbäume für Status und Arbeitsbereiche werden abgelehnt, um eine Selbsteinbeziehung zu vermeiden.
- `openclaw backup verify <archive>` prüft, ob das Archiv genau ein Stammmanifest enthält, lehnt Archivpfade mit Verzeichnisdurchlauf sowie SQLite-Sidecars ab, bestätigt die Existenz jeder im Manifest deklarierten Nutzlast, validiert die Dateistruktur jedes SQLite-Snapshots und führt vollständige Integritäts- und Rollenprüfungen für kanonische OpenClaw-Datenbanken aus. Dedizierte Plugin-Schemata bleiben undurchsichtig, da sie möglicherweise vom Eigentümer definierte SQLite-Funktionen erfordern. `openclaw backup create --verify` führt diese Validierung unmittelbar nach dem Schreiben des Archivs aus.
- `openclaw backup create --only-config` sichert ausschließlich die aktive JSON-Konfigurationsdatei.

## SQLite-Snapshots

Verwenden Sie `openclaw backup sqlite`, wenn Sie statt eines umfassenden Statusarchivs ein portables Artefakt für eine einzelne OpenClaw-eigene SQLite-Datenbank benötigen.

Bei der Snapshot-Erstellung muss genau eine benannte Quelle angegeben werden:

| Befehl                                                          | Datenbank                    |
| --------------------------------------------------------------- | ---------------------------- |
| `openclaw backup sqlite create --global --repository <dir>`     | Gemeinsam genutzter OpenClaw-Status |
| `openclaw backup sqlite create --agent <id> --repository <dir>` | Eine Datenbank pro Agent      |

Das Repository enthält ein Verzeichnis pro übergebenem Snapshot. Jedes Snapshot-Verzeichnis enthält genau:

- `manifest.json`
- `database.sqlite`

Bei der Snapshot-Erstellung wird die aktive Datenbank vor dem Lesen überprüft, mit der Online-Backup-API von SQLite der festgeschriebene WAL-Status erfasst, ohne eine lange Lesetransaktion offen zu halten, die aktive Datenbank geschlossen, die private Kopie mit `VACUUM` komprimiert, die erzeugte Datenbank erneut überprüft und das fertige Verzeichnis veröffentlicht, ohne vorhandene Pfade zu überschreiben. Globale Snapshots entfernen vor der Komprimierung vorübergehende Zeilen der Zustellwarteschlange, damit gelöschte Warteschlangen-Nutzlasten nicht in freien Seiten verbleiben.

Kopieren Sie aktive `.sqlite`-, `-wal`-, `-shm`- oder `-journal`-Dateien nicht als Portabilitätsartefakt. Kopieren Sie ausschließlich vollständige Snapshot-Verzeichnisse.

SQLite-Snapshots können Authentifizierungsprofile, Sitzungsstatus, Plugin-Status und andere vertrauliche Datensätze enthalten. Schützen Sie Repositorys mit denselben Berechtigungen, derselben Verschlüsselung, Aufbewahrungsrichtlinie und denselben Zielbeschränkungen wie das aktive OpenClaw-Statusverzeichnis.

### Überprüfen und wiederherstellen

```bash
openclaw backup sqlite verify <snapshot-directory>
openclaw backup sqlite restore <snapshot-directory> --target <new-database-path>
```

Bei der Überprüfung werden die strikte Manifeststruktur, Artefaktgröße und SHA-256-Prüfsumme, SQLite-Integrität, Fremdschlüssel, Schemaversion, Datenbankrolle und -eigentümer sowie OpenClaw-eigene Indexdefinitionen geprüft.

Bei der Überprüfung wird eine private, an den Inhalt gebundene Kopie validiert, sodass Pfadnamen-Wettlaufsituationen die von SQLite untersuchten Bytes nicht austauschen können. Standardmäßig wird diese temporäre Kopie neben dem Snapshot-Repository erstellt und vor der Rückkehr des Befehls entfernt. Das Staging-Stammverzeichnis und seine Vorfahrenkette müssen verhindern, dass andere Benutzer es ersetzen. POSIX-Stammverzeichnisse müssen dem aktuellen Benutzer gehören und dürfen weder für die Gruppe noch für alle Benutzer schreibbar sein; Sticky-Vorfahren wie `/tmp` werden für benutzereigene untergeordnete Verzeichnisse akzeptiert. macOS-ACL-Berechtigungen, die das Staging offenlegen oder austauschbar machen, werden abgelehnt. Windows-Stammverzeichnisse und deren Vorfahren müssen dem aktuellen Benutzer oder einem vertrauenswürdigen Betriebssystemprinzipal gehören und über ACLs verfügen, die nicht vertrauenswürdigen Staging-Zugriff verweigern. Geben Sie bei einer schreibgeschützten Einbindung oder Netzwerkfreigabe `--scratch <existing-private-directory>` auf einem Speicher mit gleichwertiger Verschlüsselung und gleichwertigen Zielkontrollen an.

Die Snapshot-Erstellung wendet vor dem Staging oder Veröffentlichen von Datenbankbytes dieselben Prüfungen für Eigentümer, ACLs, Vorfahren und Pfadidentität auf das Repository an.

Die Wiederherstellung wiederholt die Überprüfung und schreibt ausschließlich in ein neues Ziel. Sie lehnt ein vorhandenes Ziel sowie `-wal`-, `-shm`- oder `-journal`-Sidecars ab und ersetzt niemals eine aktive OpenClaw-Datenbank direkt. Für das übergeordnete Zielverzeichnis gelten dieselben Pfadsicherheitsanforderungen wie für den temporären Prüfbereich. Die Aktivierung einer wiederhergestellten Datenbank bleibt ein ausdrücklicher Offline-Schritt des Betreibers.

Snapshot-Repositorys sind lokale Verzeichnisse. Zeitplanung, Upload, Aufbewahrung, inkrementelle WAL-Pakete, Failover und Wiederherstellung beim Systemstart liegen bewusst außerhalb dieses Befehls.

## Was gesichert wird

`openclaw backup create` plant Quellen aus Ihrer lokalen OpenClaw-Installation:

- Das Statusverzeichnis (üblicherweise `~/.openclaw`)
- Der Pfad der aktiven Konfigurationsdatei
- Das aufgelöste `credentials/`-Verzeichnis, wenn es außerhalb des Statusverzeichnisses vorhanden ist
- Aus der aktuellen Konfiguration ermittelte Arbeitsbereichsverzeichnisse, sofern Sie nicht `--no-include-workspace` angeben

Authentifizierungsprofile und anderer Laufzeitstatus pro Agent befinden sich in SQLite unterhalb des Statusverzeichnisses (`agents/<agentId>/agent/openclaw-agent.sqlite`) und werden daher automatisch durch den Statussicherungseintrag abgedeckt.

`--only-config` überspringt die Ermittlung von Status, Anmeldedatenverzeichnis und Arbeitsbereichen und archiviert ausschließlich den Pfad der aktiven Konfigurationsdatei.

OpenClaw kanonisiert Pfade vor dem Erstellen des Archivs: Wenn sich die Konfiguration, das Anmeldedatenverzeichnis oder ein Arbeitsbereich bereits innerhalb des Statusverzeichnisses befindet, werden sie nicht als separate Sicherungsquellen der obersten Ebene dupliziert. Fehlende Pfade werden übersprungen.

Während der Archiverstellung schließt OpenClaw bekannte, im laufenden Betrieb veränderliche Pfade aus, bevor `tar` sie liest. Dadurch werden Wettlaufsituationen zwischen der aufgezeichneten Größe einer Datei und gleichzeitigen Schreibvorgängen vermieden. Der Filter wendet unter jedem gesicherten Statusverzeichnis folgende statusrelative Regeln an:

| Statusrelativer Bereich                        | Übersprungene Dateiendungen    |
| ---------------------------------------------- | ------------------------------ |
| `sessions/**`                                | `.jsonl`, `.log`              |
| `agents/<agentId>/sessions/**`               | `.jsonl`, `.log`              |
| `cron/runs/**`                               | `.jsonl`, `.log`              |
| `logs/**`                                    | `.jsonl`, `.log`              |
| `delivery-queue/**`                          | `.json`, `.delivered`, `.tmp` |
| `session-delivery-queue/**`                  | `.json`, `.delivered`, `.tmp` |
| Jeder Pfad unterhalb des gesicherten Statusverzeichnisses | `.sock`, `.pid`, `.tmp`       |

Diese Regeln filtern keine Arbeitsbereichsdateien außerhalb des Statusverzeichnisses. Sie lassen außerdem abgeschlossene Transkript- und Protokolldateien aus, die der Tabelle entsprechen. Bewahren Sie diese Datensätze daher bei Bedarf separat auf. Das Feld `skippedVolatileCount` im JSON-Ergebnis gibt an, wie viele Dateien absichtlich ausgelassen wurden.

SQLite-Datenbanken unterhalb des Statusverzeichnisses werden mit der Online-Backup-API von SQLite erfasst und offline mit `VACUUM` komprimiert, damit Reste gelöschter Seiten nicht in das Archiv gelangen und aktive WAL-/SHM-Dateien nicht kopiert werden. Bei einer Plugin-eigenen Datenbank, die nicht verfügbare, vom Eigentümer definierte SQLite-Funktionen erfordert, wird sicher abgebrochen, statt auf eine direkte Dateikopie zurückzugreifen. SQLite-Dateien, die über Arbeitsbereichssicherungen einbezogen werden, werden als Arbeitsbereichsdateien kopiert und fallen nicht unter die Komprimierungsgarantie.

Installierte Plugin-Quell- und Manifestdateien im `extensions/`-Baum des Statusverzeichnisses werden einbezogen, ihre verschachtelten `node_modules/`-Abhängigkeitsbäume werden jedoch als neu erstellbare Installationsartefakte übersprungen. Verwenden Sie nach der Wiederherstellung eines Archivs `openclaw plugins update <id>` oder installieren Sie mit `openclaw plugins install <spec> --force` neu, wenn ein wiederhergestelltes Plugin fehlende Abhängigkeiten meldet.

Vom Installationsprogramm verwaltete und neu erstellbare Laufzeit-Stammverzeichnisse unterhalb des Statusverzeichnisses werden ebenfalls übersprungen: `dev/`, `git/`, `npm/`, das veraltete `npm-runtime/` und `tools/`. Diese enthalten verwaltete Checkouts, Paketbäume und heruntergeladene Laufzeitumgebungen anstelle maßgeblicher Benutzerdaten. Installieren oder aktualisieren Sie nach der Wiederherstellung die entsprechende Laufzeitumgebung beziehungsweise das entsprechende Plugin neu. Eine ausdrücklich konfigurierte Konfigurationsdatei, ein Anmeldedatenverzeichnis oder ein Arbeitsbereich innerhalb eines dieser Stammverzeichnisse bleibt einbezogen.

## Verhalten bei ungültiger Konfiguration

`openclaw backup` umgeht die normale Konfigurationsvorprüfung, sodass der Befehl auch bei der Wiederherstellung helfen kann. Die Arbeitsbereichsermittlung ist von einer gültigen Konfiguration abhängig. Daher bricht `openclaw backup create` frühzeitig ab, wenn die Konfigurationsdatei vorhanden, aber ungültig ist und die Arbeitsbereichssicherung weiterhin aktiviert ist.

Führen Sie für eine Teilsicherung in dieser Situation den Befehl mit `--no-include-workspace` erneut aus: Status, Konfiguration und das externe Anmeldedatenverzeichnis bleiben im Umfang enthalten, während die Arbeitsbereichsermittlung vollständig übersprungen wird.

`--only-config` funktioniert ebenfalls bei fehlerhafter Konfiguration, da die Konfiguration nicht zur Arbeitsbereichsermittlung geparst wird.

## Größe und Leistung

OpenClaw erzwingt weder eine integrierte maximale Sicherungsgröße noch eine Größenbeschränkung pro Datei. Wenn ein Archivschreibvorgang fünf Minuten lang keine Daten erzeugt, schlägt er fehl und entfernt seine unvollständige temporäre Datei, anstatt unbegrenzt zu hängen. Die praktischen Grenzen ergeben sich ansonsten aus:

- Verfügbarem Speicherplatz für den temporären Archivschreibvorgang und das endgültige Archiv
- Der Zeit zum Durchlaufen großer Arbeitsbereichsbäume und zu deren Komprimierung in ein `.tar.gz`
- Der Zeit zum erneuten Scannen des Archivs mit `--verify` oder `openclaw backup verify`
- Dem Verhalten des Zieldateisystems: OpenClaw erfordert eine Veröffentlichung über nicht überschreibende Hardlinks, damit ein endgültiger Archivpfad niemals eine noch laufende Kopie offenlegt; nicht unterstützte Dateisysteme schlagen mit einer handlungsorientierten Fehlermeldung fehl

Wenn die Bestätigung der Dauerhaftigkeit des endgültigen Verzeichnisses nach der Veröffentlichung fehlschlägt, meldet der Befehl einen Fehler, behält den vollständigen endgültigen Eintrag jedoch bei, statt das Löschen eines gleichzeitig erstellten Ersatzes zu riskieren.

Große Arbeitsbereiche sind üblicherweise der wichtigste Faktor für die Archivgröße. Verwenden Sie `--no-include-workspace` für eine kleinere und schnellere Sicherung oder `--only-config` für das kleinste Archiv.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
