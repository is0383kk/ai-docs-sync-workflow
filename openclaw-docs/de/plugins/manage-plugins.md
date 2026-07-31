---
doc-schema-version: 1
read_when:
    - Sie möchten Plugins in der Control UI durchsuchen, installieren, aktivieren oder deaktivieren
    - Sie möchten kurze Beispiele zum Auflisten, Installieren, Aktualisieren, Prüfen oder Deinstallieren von Plugins
    - Sie möchten eine Installationsquelle für ein Plugin auswählen
    - Sie benötigen die richtige Referenz für die Veröffentlichung von Plugin-Paketen
sidebarTitle: Manage plugins
summary: OpenClaw-Plugins über die Control UI oder CLI verwalten
title: Plugins verwalten
x-i18n:
    generated_at: "2026-07-26T19:07:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9101d5c3630b618a043f1e71fdf5fa083698cc23694ccdc773d295a37c4c1ef3
    source_path: plugins/manage-plugins.md
    workflow: 16
---

Die Control UI deckt den üblichen Ablauf zum Ermitteln, Installieren, Aktivieren und Deaktivieren ab. Die CLI ergänzt Aktualisierung, Deinstallation, erweiterte Konfiguration und explizite Steuerelemente für die Installationsquelle. Den vollständigen Befehlsvertrag, die Flags, Regeln zur Quellenauswahl und Sonderfälle finden Sie unter [`openclaw plugins`](/de/cli/plugins).

Typischer CLI-Ablauf: Suchen Sie ein Paket, installieren Sie es aus ClawHub, npm, git oder einem lokalen Pfad, lassen Sie den verwalteten Gateway automatisch neu starten (oder starten Sie ihn manuell neu) und überprüfen Sie anschließend die Runtime-Registrierungen des Plugins.

## Control UI verwenden

Öffnen Sie **Plugins** in der Control UI oder verwenden Sie `/settings/plugins` relativ zum konfigurierten Basispfad der Control UI. Beispielsweise verwendet ein Basispfad von `/openclaw` den Pfad `/openclaw/settings/plugins`. Die Seite umfasst zwei Registerkarten:

- **Installiert** zeigt den vollständigen lokalen Bestand, gruppiert nach Kategorie (Kanäle, Modell-Provider, Speicher, Tools). Jede Zeile öffnet eine Detailansicht; über ihr Überlaufmenü (`…`) kann das Plugin aktiviert oder deaktiviert und bei extern installierten Plugins **Entfernen** ausgewählt werden. Die Registerkarte führt außerdem die konfigurierten [MCP-Server](/de/cli/mcp) mit denselben menügesteuerten Aktionen zum Aktivieren, Deaktivieren und Entfernen auf, wobei `mcp.servers` in der Gateway-Konfiguration bearbeitet wird.
- **Entdecken** ist der Store: hervorgehobene, in OpenClaw enthaltene Plugins, offizielle externe Plugins und eine kuratierte Auswahl an Konnektoren. Konnektorkarten fügen entweder mit einem Klick einen gehosteten MCP-Server hinzu (GitHub, Notion, Linear, Sentry, Home Assistant) oder öffnen eine vorausgefüllte ClawHub-Suche. Eine Eingabe in das Suchfeld fragt [ClawHub](https://clawhub.ai/plugins) direkt ab und fügt einen Abschnitt **Von ClawHub** mit Downloadzahlen und Kennzeichnungen zur Quellenverifizierung hinzu.

Enthaltene Plugins benötigen keine Paketinstallation. Ihre Menüaktion lautet **Aktivieren** oder **Deaktivieren**. Workboard ist beispielsweise in OpenClaw enthalten und standardmäßig deaktiviert; wählen Sie daher **Aktivieren**, um es einzuschalten. Gebündelte Plugins können nicht entfernt, sondern nur deaktiviert werden.

Der Zugriff auf Katalog und Suche erfordert `operator.read`. Änderungen zum Installieren, Aktivieren, Deaktivieren und Entfernen sowie Änderungen an MCP-Servern erfordern `operator.admin`. Eine ClawHub-Installation wird vom Gateway ausgeführt und behält dessen Prüfungen für Vertrauenswürdigkeit, Integrität und Plugin-Installationsrichtlinien bei. Wenn ein installiertes Plugin als Administrator aktiviert wird, wird dieses ausdrückliche Vertrauen außerdem erfasst, indem das ausgewählte Plugin einer vorhandenen restriktiven `plugins.allow`-Liste hinzugefügt wird. Ein expliziter `plugins.deny`-Eintrag bleibt maßgeblich und muss entfernt werden, bevor das Plugin aktiviert werden kann.

Das Installieren oder Entfernen von Plugin-Code erfordert einen Neustart des Gateways. Änderungen der Aktivierung können ohne Neustart angewendet werden, wenn das installierte Plugin und die aktuelle Gateway-Runtime dies unterstützen; andernfalls weist die UI darauf hin, dass ein Neustart erforderlich ist. OAuth-gestützte MCP-Konnektoren benötigen nach dem Hinzufügen weiterhin einmalig `openclaw mcp login <name>` über die CLI.

Die Control UI installiert nicht aus beliebigen npm-, git- oder lokalen Pfadquellen, aktualisiert keine Plugins und stellt keine umfangreiche Plugin-Konfiguration bereit. Verwenden Sie für diese Vorgänge die folgenden CLI-Abläufe.

## Plugins auflisten und suchen

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins search "calendar"
```

`--json` für Skripte:

```bash
openclaw plugins list --json \
  | jq '.plugins[] | {id, enabled, format, source, dependencyStatus}'
```

`plugins list` ist eine Prüfung des Bestands im kalten Zustand: was OpenClaw anhand von Konfiguration, Manifesten und der persistenten Plugin-Registry ermitteln kann. Sie weist nicht nach, dass ein bereits laufender Gateway die Plugin-Runtime importiert hat. Die JSON-Ausgabe enthält Registry-Diagnosen und den `dependencyStatus` jedes Plugins (ob deklarierte `dependencies`/`optionalDependencies` auf dem Datenträger aufgelöst werden).

`plugins search` fragt ClawHub nach installierbaren Plugin-Paketen ab und gibt für jedes Ergebnis einen Installationshinweis (`openclaw plugins install clawhub:<package>`) aus.

## Plugins aktivieren und deaktivieren

```bash
openclaw plugins enable <plugin-id>
openclaw plugins disable <plugin-id>
```

Schaltet den Konfigurationseintrag eines Plugins um, ohne installierte Dateien zu verändern. Einige gebündelte Plugins (gebündelte Modell-/Sprach-Provider und das gebündelte Browser-Plugin) sind standardmäßig aktiviert; andere erfordern nach der Installation `enable`.

## Plugins installieren

```bash
# ClawHub nach Plugin-Paketen durchsuchen.
openclaw plugins search "calendar"

# Aus ClawHub installieren.
openclaw plugins install clawhub:<package>
openclaw plugins install clawhub:<package>@1.2.3
openclaw plugins install clawhub:<package>@beta

# Aus npm installieren.
openclaw plugins install npm:<package>
openclaw plugins install npm:@scope/openclaw-plugin@1.2.3
openclaw plugins install npm:@openclaw/codex

# Aus einem lokalen npm-pack-Artefakt installieren.
openclaw plugins install npm-pack:<path.tgz>

# Aus git oder einem lokalen Entwicklungs-Checkout installieren.
openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0
openclaw plugins install ./my-plugin
openclaw plugins install --link ./my-plugin
```

Reine Paketspezifikationen werden während der Umstellung beim Start aus npm installiert, sofern der Name nicht einer gebündelten oder offiziellen Plugin-ID entspricht; in diesem Fall verwendet OpenClaw stattdessen diese lokale/offizielle Kopie. Verwenden Sie `clawhub:`, `npm:`, `git:` oder `npm-pack:` für eine deterministische Quellenauswahl. Die gebündelten und offiziellen Katalogpakete von OpenClaw gelten ebenso wie ClawHub-Pakete als vertrauenswürdig. Neue beliebige npm-, git-, lokale Pfad-/Archiv-, `npm-pack:`- oder Marketplace-Quellen erfordern bei nicht interaktiven Installationen `--force`, nachdem Sie die Quelle geprüft und als vertrauenswürdig eingestuft haben.

`--force` bestätigt eine Nicht-ClawHub-Quelle ohne Rückfrage und überschreibt bei Bedarf ein vorhandenes Installationsziel. Verwenden Sie für routinemäßige Upgrades einer nachverfolgten npm-, ClawHub- oder Hook-Pack-Installation stattdessen `openclaw plugins update`. Mit `--link` bestätigt `--force` nur die Quelle; das verknüpfte Verzeichnis wird weder kopiert noch überschrieben.

Wenn ein neu installiertes Plugin eine noch nicht vorhandene Konfiguration benötigt, erfasst OpenClaw die Installation, lässt das Plugin jedoch deaktiviert. Konfigurieren Sie `plugins.entries.<id>.config` und führen Sie anschließend `openclaw plugins enable <id>` aus. Wenn ein vorhandener Konfigurationseintrag ungültig ist, schlägt die Installation fehl, ohne ihn neu zu schreiben.

## Neu starten und prüfen

Ein laufender verwalteter Gateway mit aktivierter Konfigurationsneuladung startet nach dem Installieren, Aktualisieren oder Deinstallieren von Plugin-Code automatisch neu. Wenn der Gateway nicht verwaltet wird oder das Neuladen deaktiviert ist, starten Sie ihn selbst neu, bevor Sie aktive Runtime-Oberflächen prüfen:

```bash
openclaw gateway restart
openclaw plugins inspect <plugin-id> --runtime --json
```

`inspect --runtime` lädt das Plugin-Modul und weist nach, dass es Runtime-Oberflächen registriert hat (Tools, Hooks, Dienste, Gateway-Methoden, HTTP-Routen und Plugin-eigene CLI-Befehle). Einfaches `inspect` und `list` sind lediglich Prüfungen von Manifest, Konfiguration und Registry im kalten Zustand.

## Plugins aktualisieren

```bash
openclaw plugins update <plugin-id>
openclaw plugins update <npm-package-or-spec>
openclaw plugins update --all
openclaw plugins update <plugin-id> --dry-run
```

Bei Übergabe einer Plugin-ID wird deren nachverfolgte Installationsspezifikation wiederverwendet: Gespeicherte dist-tags (`@beta`) und exakt angeheftete Versionen werden bei späteren `update <plugin-id>`-Ausführungen übernommen.

`openclaw plugins update --all` ist der Pfad für die Massenwartung. Er berücksichtigt weiterhin gewöhnliche nachverfolgte Installationsspezifikationen, aber vertrauenswürdige offizielle OpenClaw-Plugin-Einträge werden mit dem aktuellen Ziel des offiziellen Katalogs synchronisiert, statt an einem veralteten exakten offiziellen Paket angeheftet zu bleiben; wenn `update.channel` den Wert `beta` hat, bevorzugt diese Synchronisierung die Beta-Veröffentlichungslinie. Verwenden Sie ein gezieltes `update <plugin-id>`, damit eine exakte oder mit einem Tag versehene offizielle Spezifikation unverändert bleibt.

Übergeben Sie bei npm-Installationen eine explizite Paketspezifikation, um den nachverfolgten Eintrag zu wechseln:

```bash
openclaw plugins update @scope/openclaw-plugin@beta
openclaw plugins update @scope/openclaw-plugin
```

Der zweite Befehl verschiebt ein Plugin zurück auf die standardmäßige Veröffentlichungslinie der Registry, wenn es zuvor an eine exakte Version oder ein Tag angeheftet war.

Die genauen Fallback- und Anheftungsregeln finden Sie unter [`openclaw plugins`](/de/cli/plugins#update).

## Plugins deinstallieren

```bash
openclaw plugins uninstall <plugin-id> --dry-run
openclaw plugins uninstall <plugin-id>
openclaw plugins uninstall <plugin-id> --keep-files
```

Die Deinstallation entfernt den Konfigurationseintrag des Plugins, den persistenten Plugin-Indexeintrag, Einträge in Zulassungs-/Sperrlisten und gegebenenfalls verknüpfte `plugins.load.paths`-Einträge. Das verwaltete Installationsverzeichnis wird entfernt, sofern Sie nicht `--keep-files` übergeben. Ein laufender verwalteter Gateway startet automatisch neu, wenn die Deinstallation die Plugin-Quelle ändert.

Im Nix-Modus (`OPENCLAW_NIX_MODE=1`) sind Installation, Aktualisierung, Deinstallation, Aktivierung und Deaktivierung von Plugins vollständig deaktiviert; verwalten Sie diese Einstellungen stattdessen in der Nix-Quelle der Installation.

## Quelle auswählen

| Quelle      | Verwenden, wenn                                                               | Beispiel                                                       |
| ----------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ClawHub     | Sie OpenClaw-native Ermittlung, Scan-Zusammenfassungen, Versionen und Hinweise wünschen | `openclaw plugins install clawhub:<package>`                   |
| git         | Sie einen Branch, ein Tag oder einen Commit aus einem Repository verwenden möchten | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| lokaler Pfad | Sie ein Plugin auf demselben Computer entwickeln oder testen                 | `openclaw plugins install --link ./my-plugin`                  |
| Marketplace | Sie ein Claude-kompatibles Marketplace-Plugin installieren                   | `openclaw plugins install <plugin> --marketplace <source>`     |
| npm pack    | Sie ein lokales Paketartefakt anhand der npm-Installationssemantik prüfen     | `openclaw plugins install npm-pack:<path.tgz>`                 |
| npmjs.com   | Sie bereits JavaScript-Pakete veröffentlichen oder npm-dist-tags/eine private Registry benötigen | `openclaw plugins install npm:@acme/openclaw-plugin`           |

Verwaltete Installationen über lokale Pfade müssen Plugin-Verzeichnisse oder Archive sein. Legen Sie eigenständige Plugin-Dateien in `plugins.load.paths` ab, statt sie mit `plugins install` zu installieren.

## Plugins veröffentlichen

ClawHub ist die primäre öffentliche Oberfläche zum Ermitteln von OpenClaw-Plugins. Veröffentlichen Sie dort, wenn Benutzer vor der Installation Plugin-Metadaten, den Versionsverlauf, Ergebnisse von Registry-Scans und Installationshinweise finden sollen.

```bash
npm i -g clawhub
clawhub login
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
```

Native npm-Plugins müssen vor der Veröffentlichung ein Plugin-Manifest (`openclaw.plugin.json`) sowie `package.json`-Metadaten enthalten:

```json package.json
{
  "name": "@acme/openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

```bash
npm publish --access public
openclaw plugins install npm:@acme/openclaw-plugin
openclaw plugins install npm:@acme/openclaw-plugin@beta
openclaw plugins install npm:@acme/openclaw-plugin@1.0.0
```

Verwenden Sie für den vollständigen Veröffentlichungsvertrag diese Seiten, statt diese Seite als Veröffentlichungsreferenz zu behandeln:

- [Veröffentlichen auf ClawHub](/de/clawhub/publishing) erläutert Eigentümer, Scopes, Veröffentlichungen, Reviews, Paketvalidierung und Paketübertragung.
- [Plugins erstellen](/de/plugins/building-plugins) zeigt die vollständige Struktur eines Plugin-Pakets (einschließlich `openclaw.plugin.json`) und den Ablauf der ersten Veröffentlichung.
- [Plugin-Manifest](/de/plugins/manifest) definiert die Felder nativer Plugin-Manifeste.

Wenn dasselbe Paket sowohl auf ClawHub als auch auf npm verfügbar ist, verwenden Sie das explizite Präfix `clawhub:` oder `npm:`, um eine Quelle zu erzwingen.

## Verwandte Themen

- [Plugins](/de/tools/plugin) - installieren, konfigurieren, neu starten und Fehler beheben
- [`openclaw plugins`](/de/cli/plugins) - vollständige CLI-Referenz
- [Community-Plugins](/de/plugins/community) - öffentliche Auffindbarkeit und Veröffentlichung auf ClawHub
- [ClawHub](/de/clawhub/cli) - CLI-Operationen für die Registry
- [Plugins erstellen](/de/plugins/building-plugins) - ein Plugin-Paket erstellen
- [Plugin-Manifest](/de/plugins/manifest) - Manifest- und Paketmetadaten
