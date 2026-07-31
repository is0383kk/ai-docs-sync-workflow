---
read_when:
    - Sie möchten Gateway-Plugins oder kompatible Bundles installieren oder verwalten
    - Sie möchten ein einfaches Tool-Plugin erstellen oder validieren
    - Sie möchten Fehler beim Laden von Plugins debuggen
sidebarTitle: Plugins
summary: CLI-Referenz für `openclaw plugins` (initialisieren, erstellen, validieren, auflisten, installieren, Marketplace, deinstallieren, aktivieren/deaktivieren, Doctor)
title: Plugins
x-i18n:
    generated_at: "2026-07-26T18:52:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a1acba76fb1bc0ddae75e51fe573d3c2ac8f694607836e0c072ec7ca8fc0e262
    source_path: cli/plugins.md
    workflow: 16
---

Verwalten Sie Gateway-Plugins, Hook-Pakete und kompatible Bundles.

<CardGroup cols={2}>
  <Card title="Plugin-System" href="/de/tools/plugin">
    Benutzerhandbuch zum Installieren, Aktivieren und Beheben von Problemen mit Plugins.
  </Card>
  <Card title="Plugins verwalten" href="/de/plugins/manage-plugins">
    Kurze Beispiele für Installation, Auflistung, Aktualisierung, Deinstallation und Veröffentlichung.
  </Card>
  <Card title="Plugin-Bundles" href="/de/plugins/bundles">
    Kompatibilitätsmodell für Bundles.
  </Card>
  <Card title="Plugin-Manifest" href="/de/plugins/manifest">
    Manifestfelder und Konfigurationsschema.
  </Card>
  <Card title="Sicherheit" href="/de/gateway/security">
    Sicherheitshärtung für Plugin-Installationen.
  </Card>
</CardGroup>

## Befehle

```bash
openclaw plugins list [--enabled] [--verbose] [--json]
openclaw plugins search <query> [--limit <n>] [--json]
openclaw plugins install <path-or-spec> [--link] [--force] [--pin] [--marketplace <source>]
openclaw plugins inspect <id> [--runtime] [--json]
openclaw plugins inspect --all [--runtime] [--json]
openclaw plugins info <id>                    # Alias für inspect
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id> [--dry-run] [--keep-files] [--force]
openclaw plugins update <id-or-npm-spec> | --all [--dry-run]
openclaw plugins registry [--refresh] [--json]
openclaw plugins doctor
openclaw plugins init <id> [--name <name>] [--type tool|provider] [--directory <path>]
openclaw plugins build [--entry <path>] [--check]
openclaw plugins validate [--entry <path>]
openclaw plugins marketplace entries [--offline] [--feed-profile <name>] [--json]
openclaw plugins marketplace list <source> [--json]
openclaw plugins marketplace refresh [--feed-profile <name>] [--expected-sha256 <sha256>] [--json]
```

Führen Sie zur Untersuchung langsamer Installations-, Prüf-, Deinstallations- oder Registry-Aktualisierungsvorgänge den
Befehl mit `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` aus. Der Trace schreibt die Zeitmessungen der Phasen
nach stderr und hält die JSON-Ausgabe analysierbar. Siehe [Debugging](/de/help/debugging#plugin-lifecycle-trace).

<Note>
Im Nix-Modus (`OPENCLAW_NIX_MODE=1`) ist `openclaw.json` unveränderlich. `install`, `update`, `uninstall`, `enable` und `disable` verweigern alle die Ausführung. Bearbeiten Sie stattdessen die Nix-Quelle für diese Installation (`programs.openclaw.config` oder `instances.<name>.config` für nix-openclaw) und erstellen Sie sie anschließend neu. Siehe den agentenorientierten [Schnellstart](https://github.com/openclaw/nix-openclaw#quick-start).
</Note>

<Note>
Gebündelte Plugins werden mit OpenClaw ausgeliefert. Einige sind standardmäßig aktiviert (beispielsweise gebündelte Modell-Provider, gebündelte Sprachanbieter und das gebündelte Browser-Plugin); andere erfordern `plugins enable`.

Native OpenClaw-Plugins liefern `openclaw.plugin.json` mit einem eingebetteten JSON-Schema (`configSchema`, auch wenn es leer ist) aus. Kompatible Bundles verwenden stattdessen ihre eigenen Bundle-Manifeste.

`plugins list` zeigt `Format: openclaw` oder `Format: bundle`. Die ausführliche Listen-/Infoausgabe zeigt außerdem den Bundle-Untertyp (`codex`, `claude` oder `cursor`) sowie die erkannten Bundle-Funktionen.
</Note>

## Entwicklung

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm run plugin:build
npm run plugin:validate
```

`plugins init` erstellt standardmäßig ein minimales TypeScript-Tool-Plugin. Das erste
Argument ist die Plugin-ID; `--name` legt den Anzeigenamen fest. OpenClaw verwendet die
ID für das standardmäßige Ausgabeverzeichnis und die Paketbenennung. Tool-Gerüste verwenden
`defineToolPlugin` und erzeugen die `package.json`-Skripte `plugin:build` und
`plugin:validate`, die zunächst bauen und anschließend `openclaw plugins build`/`validate` aufrufen.

`plugins build` importiert den gebauten Einstiegspunkt, liest dessen statische Tool-Metadaten, schreibt
`openclaw.plugin.json` und hält `openclaw.extensions` von `package.json` synchron.
`plugins validate` prüft, ob das generierte Manifest, die Paketmetadaten und
der aktuelle Export des Einstiegspunkts weiterhin übereinstimmen. Den vollständigen Entwicklungsablauf finden Sie unter
[Tool-Plugins](/de/plugins/tool-plugins).

Das Gerüst schreibt TypeScript-Quellcode, erzeugt die Metadaten jedoch aus dem gebauten
`./dist/index.js`-Einstiegspunkt, sodass der Ablauf auch mit der veröffentlichten CLI funktioniert. Verwenden Sie
`--entry <path>`, wenn der Einstiegspunkt nicht dem standardmäßigen Paketeinstiegspunkt entspricht. Verwenden Sie
`plugins build --check` in der CI, damit veraltete generierte Metadaten einen Fehler verursachen, ohne
Dateien neu zu schreiben.

### Provider-Gerüst

```bash
openclaw plugins init acme-models --name "Acme Models" --type provider
cd acme-models
npm install
npm run build
npm test
npm run validate
```

Provider-Gerüste erstellen ein generisches, OpenAI-kompatibles Modell-Provider-Plugin
mit API-Schlüssel-Authentifizierungslogik, einem `npm run validate`-Skript, das
`clawhub package validate` ausführt, ClawHub-Paketmetadaten und einem manuell
ausgelösten GitHub-Actions-Workflow für eine zukünftige vertrauenswürdige Veröffentlichung über GitHub
OIDC. Provider-Gerüste erzeugen keine Skills und verwenden nicht
`openclaw plugins build`/`validate`; diese Befehle sind für den Pfad der generierten Metadaten
des Tool-Gerüsts vorgesehen.

Ersetzen Sie vor der Veröffentlichung die Platzhalter für die API-Basis-URL, den Modellkatalog, die Dokumentationsroute,
den Anmeldedatentext und den README-Text durch echte Provider-Details. Verwenden Sie die
generierte README für die erstmalige Veröffentlichung auf ClawHub und die Einrichtung eines vertrauenswürdigen Herausgebers.

## Installation

```bash
openclaw plugins search "calendar"                      # ClawHub-Plugins durchsuchen
openclaw plugins install @openclaw/<package>            # vertrauenswürdiger offizieller Katalog
openclaw plugins install <package>                       # beliebiges npm-Paket
openclaw plugins install clawhub:<package>                # nur ClawHub
openclaw plugins install npm:<package>                    # nur npm
openclaw plugins install npm-pack:<path.tgz>               # lokales npm-pack-Tarball
openclaw plugins install git:github.com/<owner>/<repo>     # Git-Repository
openclaw plugins install git:github.com/<owner>/<repo>@<ref>
openclaw plugins install <path>                            # lokaler Pfad oder Archiv
openclaw plugins install -l <path>                         # verknüpfen statt kopieren
openclaw plugins install <plugin>@<marketplace>             # Marketplace-Kurzform
openclaw plugins install <plugin> --marketplace <name>      # Marketplace (explizit)
openclaw plugins install <package> --force                  # Quelle bestätigen/vorhandene Installation überschreiben
openclaw plugins install <package> --pin                    # aufgelöste npm-Version fixieren
openclaw plugins install clawhub:<package> --acknowledge-clawhub-risk
openclaw plugins install <package> --dangerously-force-unsafe-install
```

Maintainer, die Installationen während der Einrichtung testen, können die automatischen Quellen für Plugin-Installationen
mit abgesicherten Umgebungsvariablen überschreiben. Siehe
[Überschreibungen für Plugin-Installationen](/de/plugins/install-overrides).

<Warning>
Während der Umstellung beim Start werden einfache Paketnamen standardmäßig von npm installiert, sofern sie nicht mit der ID eines gebündelten oder offiziellen Plugins übereinstimmen. In diesem Fall verwendet OpenClaw die lokale/offizielle Kopie, statt auf die npm-Registry zuzugreifen. Verwenden Sie `npm:<package>`, wenn Sie bewusst stattdessen ein externes npm-Paket verwenden möchten. Verwenden Sie `clawhub:<package>` für ClawHub. Behandeln Sie Plugin-Installationen wie die Ausführung von Code; bevorzugen Sie festgelegte Versionen.
</Warning>

<Warning>
ClawHub-Pakete und der gebündelte/offizielle Katalog von OpenClaw sind vertrauenswürdige Installationsquellen. Bei einer neuen beliebigen npm-, `npm-pack:`-, Git-, lokalen Pfad-/Archiv- oder
Marketplace-Quelle wird eine Warnung angezeigt und vor dem Fortfahren eine Bestätigung verlangt. Nicht interaktive beliebige
Installationen müssen nach Prüfung und Vertrauensbestätigung der Quelle `--force` übergeben. Dasselbe
Flag überschreibt bei Bedarf ein vorhandenes Installationsziel. Normale Aktualisierungen einer
bereits nachverfolgten Installation benötigen es nicht. Diese Bestätigung ist unabhängig von
`--acknowledge-clawhub-risk`, das nur für Vertrauenswarnungen bei riskanten ClawHub-Releases gilt.
`--force` umgeht weder `security.installPolicy` noch die verbleibenden
Sicherheitsprüfungen der Installation.
</Warning>

`plugins search` fragt ClawHub nach installierbaren `code-plugin`- und
`bundle-plugin`-Paketen ab (nicht nach Skills; verwenden Sie dafür `openclaw skills search`).
Der Standardwert für `--limit` ist 20, begrenzt auf 100. Es wird ausschließlich der Remote-Katalog gelesen: keine
Prüfung des lokalen Zustands, Änderung der Konfiguration, Paketinstallation oder Laden der Plugin-Laufzeit.
Die Ergebnisse enthalten den ClawHub-Paketnamen, die Familie, den Kanal, die Version,
eine Zusammenfassung und einen Installationshinweis wie `openclaw plugins install clawhub:<package>`.

<Note>
ClawHub ist für die meisten Plugins die primäre Oberfläche für Verteilung und Auffindbarkeit. Npm
bleibt ein unterstützter Ausweich- und Direktinstallationspfad. OpenClaw-eigene
`@openclaw/*`-Plugin-Pakete werden wieder auf npm veröffentlicht; die aktuelle Liste finden Sie
unter [npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) oder im
[Plugin-Inventar](/de/plugins/plugin-inventory). Stabile Installationen verwenden `latest`.
Installationen und Aktualisierungen im Beta-Kanal bevorzugen das npm-Dist-Tag `beta`, sofern verfügbar,
und greifen andernfalls auf `latest` zurück. Im Extended-Stable-Kanal werden offizielle npm-Plugins
mit einfacher/standardmäßiger oder `latest`-Absicht auf exakt die installierte Core-Version
aufgelöst. Exakte Fixierungen und explizite Tags ungleich `latest`, Drittanbieterpakete und
Nicht-npm-Quellen werden nicht umgeschrieben.
</Note>

<AccordionGroup>
  <Accordion title="Konfigurations-Includes und Reparatur ungültiger Konfigurationen">
    Wenn Ihr Abschnitt `plugins` durch ein `$include` mit einer einzelnen Datei bereitgestellt wird, schreibt `plugins install/update/enable/disable/uninstall` direkt in diese eingebundene Datei und lässt `openclaw.json` unverändert. Root-Includes, Include-Arrays und Includes mit gleichgeordneten Überschreibungen schlagen sicher fehl, statt sie zusammenzuführen. Die unterstützten Formen finden Sie unter [Konfigurations-Includes](/de/gateway/configuration).

    Wenn die Konfiguration vor der Installation ungültig ist, schlägt `plugins install` normalerweise sicher fehl und fordert Sie auf, zuerst `openclaw doctor --fix` auszuführen. Während des Gateway-Starts und des Hot Reloads schlagen ungültige Plugin-Konfigurationen wie jede andere ungültige Konfiguration sicher fehl; `openclaw doctor --fix` kann den ungültigen Plugin-Eintrag isolieren. Die einzige Ausnahme für eine bereits vorhandene Konfiguration ist ein eng begrenzter Wiederherstellungspfad für gebündelte Plugins, die sich ausdrücklich für `openclaw.install.allowInvalidConfigRecovery` entscheiden.

    Wenn die vorhandene Host-Konfiguration gültig ist, aber die eigene Konfiguration des neu installierten Plugins fehlt, zeichnet OpenClaw die Installation als deaktiviert auf, statt einen ungültigen aktivierten Eintrag zu schreiben. Konfigurieren Sie `plugins.entries.<id>.config` und führen Sie anschließend `openclaw plugins enable <id>` aus. Wenn ein vorhandener Plugin-Konfigurationseintrag existiert, aber ungültig ist, schlägt die Installation fehl, ohne ihn umzuschreiben.

  </Accordion>
  <Accordion title="--force-Bestätigung und Neuinstallation gegenüber Aktualisierung">
    `--force` bestätigt eine Nicht-ClawHub-Quelle ohne Rückfrage. Es umgeht weder `security.installPolicy` noch die verbleibenden Sicherheitsprüfungen der Installation. Wenn das Plugin oder Hook-Paket bereits installiert ist, verwendet es außerdem das vorhandene Ziel erneut und überschreibt es direkt. Verwenden Sie es nach der Prüfung einer beliebigen npm-, lokalen, Archiv-, Git- oder Marketplace-Quelle oder wenn Sie dieselbe ID bewusst neu installieren. Für routinemäßige Upgrades eines bereits nachverfolgten npm-Plugins ist `openclaw plugins update <id-or-npm-spec>` vorzuziehen.

    Wenn Sie `plugins install` für eine bereits installierte Plugin-ID ausführen, hält OpenClaw an und verweist Sie für ein normales Upgrade auf `plugins update <id-or-npm-spec>` oder auf `plugins install <package> --force`, wenn Sie die aktuelle Installation tatsächlich aus einer anderen Quelle überschreiben möchten. Bei beliebigen Quellen wird weiterhin die interaktive Herkunftswarnung angezeigt; nicht interaktive Installationen müssen nach der Prüfung `--force` übergeben. Vertrauenswürdige Quellen von ClawHub und aus dem OpenClaw-Katalog benötigen dies nicht. Mit `--link` bestätigt `--force` die Quelle, ändert jedoch nicht den Installationsmodus mit verknüpftem Pfad.

  </Accordion>
  <Accordion title="Geltungsbereich von --pin">
    `--pin` gilt nur für npm-Installationen und zeichnet die aufgelöste exakte `<name>@<version>` auf. Es wird weder bei Installationen mit `git:` unterstützt (heften Sie stattdessen die Referenz in der Spezifikation an, z. B. `git:github.com/acme/plugin@v1.2.3`) noch mit `--marketplace` (Marketplace-Installationen speichern Marketplace-Quellmetadaten anstelle einer npm-Spezifikation).
  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` ist veraltet und hat jetzt keine Wirkung mehr. OpenClaw führt bei Plugin-Installationen keine integrierte Blockierung gefährlichen Codes zur Installationszeit mehr aus.

    Verwenden Sie die vom Betreiber verwaltete Oberfläche `security.installPolicy`, wenn eine hostspezifische Installationsrichtlinie erforderlich ist. Plugin-Hooks vom Typ `before_install` sind Lebenszyklus-Hooks der Plugin-Laufzeit und nicht die primäre Richtliniengrenze für CLI-Installationen.

    Wenn ein von Ihnen auf ClawHub veröffentlichtes Plugin durch einen Registry-Scan ausgeblendet oder blockiert wird, führen Sie die Schritte für Herausgeber unter [Veröffentlichung auf ClawHub](/de/clawhub/publishing) aus. `--dangerously-force-unsafe-install` fordert ClawHub nicht dazu auf, das Plugin erneut zu scannen oder eine blockierte Version öffentlich zugänglich zu machen.

  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk">
    Bei Community-Installationen von ClawHub wird vor dem Herunterladen der Vertrauensdatensatz der ausgewählten Version geprüft. Wenn ClawHub den Download für die Version deaktiviert, bösartige Scanergebnisse meldet oder die Version in einen blockierenden Moderationsstatus versetzt (unter Quarantäne gestellt, widerrufen), lehnt OpenClaw sie unabhängig von diesem Flag grundsätzlich ab. Bei nicht blockierenden riskanten Scan- oder Moderationsstatus zeigt OpenClaw die Vertrauensdetails an und fordert vor dem Fortfahren eine Bestätigung an.

    Verwenden Sie `--acknowledge-clawhub-risk` nur, nachdem Sie die ClawHub-Warnung geprüft und entschieden haben, ohne interaktive Eingabeaufforderung fortzufahren. Ausstehende oder veraltete (noch nicht unbedenkliche) Scanergebnisse lösen eine Warnung aus, erfordern jedoch keine Bestätigung. Offizielle ClawHub-Pakete und gebündelte OpenClaw-Plugin-Quellen umgehen diese Vertrauensprüfung für Versionen vollständig.

  </Accordion>
  <Accordion title="Hook-Pakete und npm-Spezifikationen">
    `plugins install` ist auch die Installationsoberfläche für Hook-Pakete, die `openclaw.hooks` in `package.json` bereitstellen. Verwenden Sie `openclaw hooks` für die gefilterte Sichtbarkeit und Aktivierung einzelner Hooks, nicht für die Paketinstallation.

    Npm-Spezifikationen sind **ausschließlich für Registrys** vorgesehen (Paketname sowie optional eine **exakte Version** oder ein **dist-tag**). Git-/URL-/Dateispezifikationen und Semver-Bereiche werden abgelehnt. Abhängigkeitsinstallationen werden aus Sicherheitsgründen in einem verwalteten npm-Projekt pro Plugin mit `--ignore-scripts` ausgeführt, selbst wenn Ihre Shell globale npm-Installationseinstellungen verwendet. Verwaltete npm-Projekte für Plugins übernehmen die npm-Einstellung `overrides` auf Paketebene von OpenClaw, sodass Sicherheitsfestlegungen des Hosts auch für hochgezogene Plugin-Abhängigkeiten gelten.

    Verwenden Sie `npm:<package>`, um die npm-Auflösung explizit festzulegen. Reine Paketspezifikationen werden während der Umstellung zum Start ebenfalls direkt von npm installiert, sofern sie nicht mit einer offiziellen Plugin-ID übereinstimmen.

    Unverarbeitete `@openclaw/*`-Spezifikationen, die mit gebündelten Plugins übereinstimmen, werden vor einem npm-Fallback zur image-eigenen gebündelten Kopie aufgelöst. Beispielsweise verwendet `openclaw plugins install @openclaw/discord@2026.5.20 --pin` das gebündelte Discord-Plugin aus dem aktuellen OpenClaw-Build, anstatt eine verwaltete npm-Überschreibung zu erstellen. Um die Verwendung des externen npm-Pakets zu erzwingen, verwenden Sie `openclaw plugins install npm:@openclaw/discord@2026.5.20 --pin`.

    Reine Spezifikationen und `@latest` verbleiben auf dem stabilen Kanal. Datumsbasierte Korrekturversionen von OpenClaw wie `2026.5.3-1` gelten bei dieser Prüfung als stabil. Wenn npm eine der beiden Formen zu einer Vorabversion auflöst, wird OpenClaw angehalten und fordert Sie auf, sich explizit mit einem Vorabversions-Tag (`@beta`/`@rc`) oder einer exakten Vorabversion (`@1.2.3-beta.4`) dafür zu entscheiden.

    Bei npm-Installationen ohne exakte Version (`npm:<package>` oder `npm:<package>@latest`) prüft OpenClaw vor der Installation die Metadaten des aufgelösten Pakets. Wenn das neueste stabile Paket eine neuere OpenClaw-Plugin-API oder eine höhere Mindestversion des Hosts erfordert, prüft OpenClaw ältere stabile Versionen und installiert stattdessen die neueste kompatible Version. Exakte Versionen und explizite dist-tags werden strikt behandelt: Eine inkompatible Auswahl schlägt fehl und fordert Sie auf, OpenClaw zu aktualisieren oder eine kompatible Version auszuwählen.

    Wenn eine reine Installationsspezifikation mit einer offiziellen Plugin-ID übereinstimmt (zum Beispiel `diffs`), installiert OpenClaw den Katalogeintrag direkt. Um ein gleichnamiges npm-Paket zu installieren, verwenden Sie eine explizite Spezifikation mit Geltungsbereich (zum Beispiel `@scope/diffs`).

  </Accordion>
  <Accordion title="Git-Repositorys">
    Verwenden Sie `git:<repo>`, um direkt aus einem Git-Repository zu installieren. Unterstützte Formen: `git:github.com/owner/repo`, `git:owner/repo`, vollständige `https://`-, `ssh://`-, `git://`-, `file://`- und `git@host:owner/repo.git`-Clone-URLs. Fügen Sie `@<ref>` oder `#<ref>` hinzu, um vor der Installation einen Branch, ein Tag oder einen Commit auszuchecken.

    Bei Git-Installationen wird das Repository in ein temporäres Verzeichnis geklont, die angeforderte Referenz wird, sofern vorhanden, ausgecheckt und anschließend das normale Installationsprogramm für Plugin-Verzeichnisse verwendet. Dadurch verhalten sich Manifestvalidierung, Installationsrichtlinie des Betreibers, Installationsvorgänge des Paketmanagers und Installationsdatensätze wie bei npm-Installationen. Aufgezeichnete Git-Installationen enthalten die Quell-URL/-Referenz sowie den aufgelösten Commit, sodass `openclaw plugins update` die Quelle später erneut auflösen kann.

    Verwenden Sie nach der Installation aus Git `openclaw plugins inspect <id> --runtime --json`, um Laufzeitregistrierungen wie Gateway-Methoden und CLI-Befehle zu überprüfen. Wenn das Plugin mit `api.registerCli` einen CLI-Stammbefehl registriert hat, führen Sie diesen Befehl direkt über die OpenClaw-Stamm-CLI aus, zum Beispiel `openclaw demo-plugin ping`.

  </Accordion>
  <Accordion title="Archive">
    Unterstützte Archive: `.zip`, `.tgz`, `.tar.gz`, `.tar`. Native OpenClaw-Plugin-Archive müssen ein gültiges `openclaw.plugin.json` im Stammverzeichnis des extrahierten Plugins enthalten; Archive, die nur `package.json` enthalten, werden abgelehnt, bevor OpenClaw Installationsdatensätze schreibt.

    Verwenden Sie `npm-pack:<path.tgz>`, wenn es sich bei der Datei um einen npm-pack-Tarball handelt und Sie
    denselben verwalteten npm-Projektpfad pro Plugin verwenden möchten, den Registry-Installationen nutzen,
    einschließlich der Überprüfung von `package-lock.json`, des Scannens hochgezogener Abhängigkeiten
    und der npm-Installationsdatensätze. Einfache Archivpfade werden weiterhin als lokale
    Archive unter dem Erweiterungsstammverzeichnis der Plugins installiert.

    Installationen aus dem Claude Marketplace werden ebenfalls unterstützt.

  </Accordion>
</AccordionGroup>

ClawHub-Installationen verwenden einen expliziten `clawhub:<package>`-Locator:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

Reine npm-kompatible Plugin-Spezifikationen werden während der Umstellung zum Start standardmäßig von npm installiert, sofern sie nicht mit einer offiziellen Plugin-ID übereinstimmen:

```bash
openclaw plugins install openclaw-codex-app-server
```

Verwenden Sie `npm:`, um eine ausschließliche npm-Auflösung explizit festzulegen:

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@openclaw/discord@2026.5.20
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw prüft vor der Installation die angegebene Plugin-API-/Mindestkompatibilität mit dem Gateway. Wenn die ausgewählte ClawHub-Version ein ClawPack-Artefakt veröffentlicht, lädt OpenClaw das versionierte npm-pack-`.tgz` herunter, überprüft den ClawHub-Digest-Header und den Artefakt-Digest und installiert es anschließend über den normalen Archivpfad. Ältere ClawHub-Versionen ohne ClawPack-Metadaten werden weiterhin über den bisherigen Prüfpfad für Paketarchive installiert. Aufgezeichnete Installationen bewahren ihre ClawHub-Quellmetadaten, den Artefakttyp, die npm-Integrität, die npm-Prüfsumme, den Tarball-Namen und die ClawPack-Digest-Daten für spätere Aktualisierungen auf.
ClawHub-Installationen ohne Versionsangabe behalten eine nicht versionierte aufgezeichnete Spezifikation bei, sodass `openclaw plugins update` neueren ClawHub-Versionen folgen kann; explizite Versions- oder Tag-Selektoren wie `clawhub:pkg@1.2.3` und `clawhub:pkg@beta` bleiben an diesen Selektor angeheftet.

### Marketplace-Kurzform

Verwenden Sie die Kurzform `plugin@marketplace`, wenn der Marketplace-Name im lokalen Registry-Cache von Claude unter `~/.claude/plugins/known_marketplaces.json` vorhanden ist:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

Verwenden Sie `--marketplace`, um die Marketplace-Quelle explizit anzugeben:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

<Tabs>
  <Tab title="Marketplace-Quellen">
    - ein Claude bekannter Marketplace-Name aus `~/.claude/plugins/known_marketplaces.json`
    - ein lokales Marketplace-Stammverzeichnis oder ein `marketplace.json`-Pfad
    - eine Kurzform für ein GitHub-Repository wie `owner/repo`
    - eine GitHub-Repository-URL wie `https://github.com/owner/repo`
    - eine Git-URL

  </Tab>
  <Tab title="Regeln für Remote-Marketplaces">
    Bei Remote-Marketplaces, die von GitHub oder über Git geladen werden, müssen Plugin-Einträge innerhalb des geklonten Marketplace-Repositorys verbleiben. OpenClaw akzeptiert relative Pfadquellen aus diesem Repository und lehnt HTTP(S)-, absolute Pfad-, Git-, GitHub- und andere Nicht-Pfad-Plugin-Quellen aus Remote-Manifesten ab.
  </Tab>
</Tabs>

Bei lokalen Pfaden und Archiven erkennt OpenClaw automatisch:

- native OpenClaw-Plugins (`openclaw.plugin.json`)
- Codex-kompatible Bundles (`.codex-plugin/plugin.json`)
- Claude-kompatible Bundles (`.claude-plugin/plugin.json` oder das standardmäßige Komponentenlayout von Claude, wenn diese Manifestdatei fehlt)
- Cursor-kompatible Bundles (`.cursor-plugin/plugin.json`)

Verwaltete lokale Installationen müssen Plugin-Verzeichnisse oder Archive sein. Eigenständige Plugin-Dateien vom Typ `.js`,
`.mjs`, `.cjs` und `.ts` werden von `plugins install` nicht in das verwaltete Plugin-
Stammverzeichnis kopiert und auch nicht geladen, wenn sie direkt in
`~/.openclaw/extensions` oder `<workspace>/.openclaw/extensions` abgelegt werden; diese
automatisch erkannten Stammverzeichnisse laden Plugin-Paket- oder Bundle-Verzeichnisse und überspringen
Skriptdateien der obersten Ebene als lokale Hilfsdateien. Führen Sie eigenständige Dateien stattdessen explizit in
`plugins.load.paths` auf.

<Note>
Kompatible Bundles werden im normalen Plugin-Stammverzeichnis installiert und durchlaufen denselben Ablauf zum Auflisten, Anzeigen von Informationen, Aktivieren und Deaktivieren. Derzeit werden Bundle-Skills, Claude-Befehls-Skills, Claude-Standardeinstellungen für `settings.json`, Claude-Standardeinstellungen für `.lsp.json` bzw. im Manifest deklarierte Standardeinstellungen für `lspServers`, Cursor-Befehls-Skills und kompatible Codex-Hook-Verzeichnisse unterstützt; andere erkannte Bundle-Funktionen werden in der Diagnose und den Informationen angezeigt, sind jedoch noch nicht mit der Laufzeitausführung verbunden.
</Note>

Verwenden Sie `-l`/`--link`, um ohne Kopieren auf ein lokales Plugin-Verzeichnis zu verweisen (wird
zu `plugins.load.paths` hinzugefügt):

```bash
openclaw plugins install -l ./my-plugin
```

`--link` wird bei Installationen mit `--marketplace` oder `git:` nicht unterstützt und
erfordert einen bereits vorhandenen lokalen Pfad. Übergeben Sie für eine nicht interaktive lokale Verknüpfung
nach Prüfung der Quelle `--force`; dies bestätigt die Herkunft, kopiert oder überschreibt das verknüpfte Verzeichnis jedoch
nicht.

<Note>
Plugins mit Workspace-Ursprung, die in einem Workspace-Erweiterungsstammverzeichnis erkannt werden, werden erst
importiert oder ausgeführt, nachdem sie explizit aktiviert wurden. Führen Sie für die lokale Entwicklung
`openclaw plugins enable <plugin-id>` aus oder legen Sie
`plugins.entries.<plugin-id>.enabled: true` fest; wenn Ihre Konfiguration
`plugins.allow` verwendet, nehmen Sie dort auch dieselbe Plugin-ID auf. Diese Fail-Closed-Regel
gilt auch, wenn die Kanaleinrichtung explizit ein Plugin mit Workspace-Ursprung zum
reinen Laden für die Einrichtung auswählt. Dadurch wird der Einrichtungscode des lokalen Kanal-Plugins nicht ausgeführt, solange dieses
Workspace-Plugin deaktiviert oder von der Zulassungsliste ausgeschlossen bleibt. Verknüpfte Installationen
und explizite `plugins.load.paths`-Einträge folgen der normalen Richtlinie für ihren
aufgelösten Plugin-Ursprung. Siehe
[Plugin-Richtlinie konfigurieren](/de/tools/plugin#configure-plugin-policy)
und [Konfigurationsreferenz](/de/gateway/configuration-reference#plugins).

Verwenden Sie `--pin` bei npm-Installationen, um die aufgelöste exakte Spezifikation (`name@version`) im verwalteten Plugin-Index zu speichern, während das Standardverhalten weiterhin nicht angeheftet bleibt.
</Note>

## Auflisten

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

<ParamField path="--enabled" type="boolean">
  Nur aktivierte Plugins anzeigen.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Von der Tabellenansicht zu Detailzeilen pro Plugin mit Metadaten zu Format, Quelle, Herkunft, Version und Aktivierung wechseln.
</ParamField>
<ParamField path="--json" type="boolean">
  Maschinenlesbares Inventar sowie Registry-Diagnosen und Installationsstatus der Paketabhängigkeiten.
</ParamField>

<Note>
`plugins list` liest zuerst die persistierte lokale Plugin-Registry und verwendet eine ausschließlich aus dem Manifest abgeleitete Ausweichlösung, wenn die Registry fehlt oder ungültig ist. Dies ist nützlich, um zu prüfen, ob ein Plugin installiert, aktiviert und für die Planung eines Kaltstarts sichtbar ist, stellt jedoch keine Live-Laufzeitprüfung eines bereits ausgeführten Gateway-Prozesses dar. Starten Sie nach Änderungen an Plugin-Code, Aktivierung, Hook-Richtlinie oder `plugins.load.paths` das Gateway neu, das den Kanal bereitstellt, bevor Sie erwarten, dass neuer `register(api)`-Code oder neue Hooks ausgeführt werden. Stellen Sie bei Remote-/Container-Bereitstellungen sicher, dass Sie tatsächlich den untergeordneten `openclaw gateway run`-Prozess und nicht nur einen Wrapper-Prozess neu starten.

`plugins list --json` enthält für jedes Plugin dessen `dependencyStatus` aus `package.json`
`dependencies` und `optionalDependencies`. OpenClaw prüft, ob diese Paketnamen
entlang des normalen Node-`node_modules`-Suchpfads des Plugins vorhanden sind;
es importiert keinen Plugin-Laufzeitcode, führt keinen Paketmanager aus und repariert
keine fehlenden Abhängigkeiten.
</Note>

Wenn beim Start `plugins.allow is empty; discovered non-bundled plugins may auto-load: ...` protokolliert wird,
führen Sie `openclaw plugins list --enabled --verbose` oder
`openclaw plugins inspect <id>` mit einer aufgeführten Plugin-ID aus, um die Plugin-
IDs zu bestätigen, und kopieren Sie vertrauenswürdige IDs in `plugins.allow` in `openclaw.json`. Wenn die
Warnung alle erkannten Plugins auflisten kann, gibt sie ein direkt einfügbares
`plugins.allow`-Snippet aus, das diese IDs bereits enthält. Wenn ein Plugin
ohne Herkunftsnachweis für Installation oder Ladepfad geladen wird, prüfen Sie diese Plugin-ID und verankern Sie
anschließend entweder die vertrauenswürdige ID in `plugins.allow` oder installieren Sie das Plugin aus einer vertrauenswürdigen Quelle neu,
damit OpenClaw die Installationsherkunft aufzeichnet.

Binden Sie für Arbeiten an gebündelten Plugins innerhalb eines paketierten Docker-Images das Plugin-
Quellverzeichnis per Bind-Mount über den entsprechenden paketierten Quellpfad ein, beispielsweise
`/app/extensions/synology-chat`. OpenClaw erkennt dieses eingebundene Quell-Overlay
vor `/app/dist/extensions/synology-chat`; ein lediglich kopiertes Quellverzeichnis
bleibt inaktiv, sodass normale paketierte Installationen weiterhin die kompilierte Distribution verwenden.

Zur Fehlerdiagnose von Laufzeit-Hooks:

- `openclaw plugins inspect <id> --runtime --json` zeigt registrierte Hooks und Diagnosen aus einem Inspektionsdurchlauf mit geladenem Modul. Die Laufzeitinspektion installiert niemals Abhängigkeiten; verwenden Sie `openclaw doctor --fix`, um veraltete Abhängigkeitszustände zu bereinigen oder fehlende herunterladbare Plugins wiederherzustellen, auf die in der Konfiguration verwiesen wird.
- `openclaw gateway status --deep --require-rpc` bestätigt die erreichbare Gateway-URL bzw. das Profil, Dienst-/Prozesshinweise, den Konfigurationspfad und den RPC-Zustand.
- Nicht gebündelte Konversations-Hooks (`llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize`, `agent_end`) erfordern `plugins.entries.<id>.hooks.allowConversationAccess=true`.

### Plugin-Index

Plugin-Installationsmetadaten sind maschinell verwalteter Zustand und keine Benutzerkonfiguration. Installationen und Aktualisierungen schreiben sie in die gemeinsam genutzte SQLite-Zustandsdatenbank unter dem aktiven OpenClaw-Zustandsverzeichnis. Die Zeile `installed_plugin_index` speichert dauerhafte `installRecords`-Metadaten, einschließlich Datensätzen für beschädigte oder fehlende Plugin-Manifeste, sowie einen aus dem Manifest abgeleiteten Cache der Kaltstart-Registry, der von `openclaw plugins update`, der Deinstallation, der Diagnose und der Kaltstart-Plugin-Registry verwendet wird.

`plugins.installs` ist eine außer Betrieb genommene Konfigurationsoberfläche für manuell erstellte Konfigurationen. Laufzeit- und Aktualisierungsbefehle lesen ausschließlich den SQLite-Index installierter Plugins. Führen Sie `openclaw doctor --fix` aus, um alte Konfigurationsdatensätze in den Index zu importieren und den außer Betrieb genommenen Schlüssel vor der normalen Laufzeitnutzung zu entfernen.

## Deinstallation

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
openclaw plugins uninstall <id> --force
```

`uninstall` entfernt Plugin-Datensätze aus `plugins.entries`, dem persistierten Plugin-Index, Einträge in Plugin-Zulassungs-/Sperrlisten sowie gegebenenfalls verknüpfte `plugins.load.paths`-Einträge. Sofern `--keep-files` nicht gesetzt ist, entfernt die Deinstallation außerdem das nachverfolgte verwaltete Installationsverzeichnis, jedoch nur, wenn dessen aufgelöster Pfad innerhalb des Plugin-Erweiterungsstammverzeichnisses von OpenClaw liegt. Wenn das Plugin derzeit den Slot `memory` oder `contextEngine` belegt, wird dieser Slot auf seinen Standardwert zurückgesetzt (`memory-core` für Speicher, `legacy` für die Kontext-Engine).

`uninstall` zeigt eine Vorschau der zu entfernenden Elemente an und fragt anschließend mit `Uninstall plugin "<id>"?` nach einer Bestätigung, bevor Änderungen vorgenommen werden. Übergeben Sie `--force`, um die Bestätigungsabfrage zu überspringen (nützlich für Skripte und nicht interaktive Ausführungen); ohne diese Option erfordert die Deinstallation ein interaktives TTY. `--dry-run` zeigt dieselbe Vorschau an und beendet den Vorgang, ohne nachzufragen oder Änderungen vorzunehmen.

<Note>
`--keep-config` wird als veralteter Alias für `--keep-files` unterstützt.
</Note>

## Aktualisierung

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call
openclaw plugins update @acme/demo
openclaw plugins update openclaw-codex-app-server --acknowledge-clawhub-risk
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

Aktualisierungen gelten für nachverfolgte Plugin-Installationen im verwalteten Plugin-Index und für nachverfolgte Hook-Paketinstallationen im gemeinsam genutzten SQLite-Zustand. Sie verwenden erneut die Quelle, die der Benutzer bereits bei der Installation des Plugins ausgewählt hat, sodass keine zweite Bestätigung der Quelle erforderlich ist.

<AccordionGroup>
  <Accordion title="Auflösen von Plugin-ID und npm-Spezifikation">
    Wenn Sie eine Plugin-ID übergeben, verwendet OpenClaw erneut die für dieses Plugin aufgezeichnete Installationsspezifikation. Das bedeutet, dass zuvor gespeicherte Distributions-Tags wie `@beta` und exakt verankerte Versionen auch bei späteren `update <id>`-Ausführungen weiterhin verwendet werden.

    Während `update <id> --dry-run` bleiben exakt verankerte npm-Installationen verankert. Wenn OpenClaw auch die standardmäßige Registry-Versionslinie des Pakets auflösen kann und diese Standardlinie neuer als die installierte verankerte Version ist, meldet der Probelauf die Verankerung und gibt den expliziten Paketaktualisierungsbefehl `@latest` aus, mit dem der standardmäßigen Registry-Versionslinie gefolgt werden kann.

    Diese Regel für gezielte Aktualisierungen unterscheidet sich vom Massenwartungspfad `openclaw plugins update --all`. Massenaktualisierungen berücksichtigen weiterhin gewöhnliche nachverfolgte Installationsspezifikationen, vertrauenswürdige offizielle OpenClaw-Plugin-Datensätze können jedoch mit dem aktuellen Ziel des offiziellen Katalogs synchronisiert werden, statt bei einem veralteten exakten offiziellen Paket zu verbleiben. Verwenden Sie die gezielte Variante `update <id>`, wenn eine exakte oder mit einem Tag versehene offizielle Spezifikation absichtlich unverändert bleiben soll.

    Bei npm-Installationen können Sie außerdem eine explizite npm-Paketspezifikation mit einem Distributions-Tag oder einer exakten Version übergeben. OpenClaw löst diesen Paketnamen wieder zum nachverfolgten Plugin-Datensatz auf, aktualisiert das installierte Plugin und zeichnet die neue npm-Spezifikation für zukünftige ID-basierte Aktualisierungen auf.

    Wenn Sie den npm-Paketnamen ohne Version oder Tag übergeben, wird dieser ebenfalls wieder zum nachverfolgten Plugin-Datensatz aufgelöst. Verwenden Sie dies, wenn ein Plugin auf eine exakte Version verankert wurde und Sie es wieder auf die standardmäßige Release-Linie der Registry umstellen möchten.

  </Accordion>
  <Accordion title="Aktualisierungen im Beta-Kanal">
    Die gezielte Variante `openclaw plugins update <id-or-npm-spec>` verwendet die nachverfolgte Plugin-Spezifikation erneut, sofern Sie keine neue Spezifikation übergeben. Die Massenversion `openclaw plugins update --all` verwendet das konfigurierte `update.channel`, wenn sie vertrauenswürdige offizielle Plugin-Datensätze mit dem Ziel des offiziellen Katalogs synchronisiert, sodass Installationen aus dem Beta-Kanal auf der Beta-Release-Linie verbleiben können, statt stillschweigend auf stable/latest normalisiert zu werden.

    `openclaw update` kennt außerdem den aktiven OpenClaw-Aktualisierungskanal: Im Beta-Kanal versuchen npm- und ClawHub-Plugin-Datensätze der Standardlinie zuerst `@beta`. Sie greifen auf die aufgezeichnete default/latest-Spezifikation zurück, wenn keine Beta-Version des Plugins vorhanden ist; npm-Plugins greifen auch darauf zurück, wenn das Beta-Paket vorhanden ist, aber die Installationsvalidierung fehlschlägt. Diese Ausweichlösung wird als Warnung gemeldet und lässt die Kernaktualisierung nicht fehlschlagen. Exakte Versionen und explizite Tags bleiben bei gezielten Aktualisierungen an diesen Selektor gebunden.

  </Accordion>
  <Accordion title="Versionsprüfungen und Integritätsabweichungen">
    Vor einer aktiven npm-Aktualisierung prüft OpenClaw die installierte Paketversion anhand der Metadaten der npm-Registry. Wenn die installierte Version und die aufgezeichnete Artefaktidentität bereits mit dem aufgelösten Ziel übereinstimmen, wird die Aktualisierung übersprungen, ohne `openclaw.json` herunterzuladen, neu zu installieren oder neu zu schreiben.

    Wenn ein gespeicherter Integritäts-Hash vorhanden ist und sich der Hash des abgerufenen Artefakts ändert, behandelt OpenClaw dies als Abweichung des npm-Artefakts. Der interaktive Befehl `openclaw plugins update` zeigt den erwarteten und den tatsächlichen Hash an und fordert vor dem Fortfahren eine Bestätigung an. Nicht interaktive Aktualisierungshilfen brechen sicher ab, sofern der Aufrufer keine explizite Fortsetzungsrichtlinie angibt.

  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install bei Aktualisierungen">
    `--dangerously-force-unsafe-install` wird aus Kompatibilitätsgründen auch für `plugins update` akzeptiert, ist jedoch veraltet und ändert das Verhalten von Plugin-Aktualisierungen nicht mehr. `security.installPolicy` des Betreibers kann Aktualisierungen weiterhin blockieren; Plugin-`before_install`-Hooks gelten nur in Prozessen, in denen Plugin-Hooks geladen sind.
  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk bei Aktualisierungen">
    Community-Plugin-Aktualisierungen auf ClawHub-Basis führen vor dem Herunterladen des Ersatzpakets dieselbe Vertrauensprüfung für die exakte Version wie Installationen durch. Verwenden Sie `--acknowledge-clawhub-risk` für geprüfte Automatisierungen, die fortfahren sollen, wenn die ausgewählte ClawHub-Version eine riskante Vertrauenswarnung aufweist. Offizielle ClawHub-Pakete und gebündelte OpenClaw-Plugin-Quellen umgehen diese Aufforderung zur Bestätigung des Release-Vertrauens.
  </Accordion>
</AccordionGroup>

## Prüfen

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
```

Die Prüfung zeigt standardmäßig Identität, Ladestatus, Quelle, Manifestfähigkeiten, Richtlinienflags, Diagnosen, Installationsmetadaten, Bundle-Fähigkeiten und jegliche erkannte Unterstützung für MCP- oder LSP-Server an, ohne die Plugin-Laufzeit zu importieren. Die JSON-Ausgabe enthält die Verträge des Plugin-Manifests, beispielsweise `contracts.agentToolResultMiddleware` und `contracts.trustedToolPolicies`, sodass Betreiber Deklarationen vertrauenswürdiger Oberflächen prüfen können, bevor sie ein Plugin aktivieren oder neu starten. Fügen Sie `--runtime` hinzu, um das Plugin-Modul zu laden und registrierte Hooks, Werkzeuge, Befehle, Dienste, Gateway-Methoden und HTTP-Routen einzubeziehen. Die Laufzeitinspektion meldet fehlende Plugin-Abhängigkeiten direkt; Installationen und Reparaturen verbleiben in `openclaw plugins install`, `openclaw plugins update` und `openclaw doctor --fix`.

Plugin-eigene CLI-Befehle werden üblicherweise als `openclaw`-Befehlsgruppen auf Stammebene installiert, Plugins können jedoch auch verschachtelte Befehle unter einem Kernbefehl wie `openclaw nodes` registrieren. Nachdem `inspect --runtime` einen Befehl unter `cliCommands` anzeigt, führen Sie ihn unter dem aufgeführten Pfad aus; ein Plugin, das beispielsweise `demo-git` registriert, kann mit `openclaw demo-git ping` überprüft werden.

Jedes Plugin wird danach klassifiziert, was es tatsächlich zur Laufzeit registriert:

| Form                | Bedeutung                                                                  |
| ------------------- | -------------------------------------------------------------------------- |
| `plain-capability`  | genau ein Fähigkeitstyp (z. B. ein reines Provider-Plugin)                 |
| `hybrid-capability` | mehr als ein Fähigkeitstyp (z. B. Text + Sprache + Bilder)                 |
| `hook-only`         | nur Hooks, keine Fähigkeiten, Werkzeuge, Befehle, Dienste oder Routen      |
| `non-capability`    | Werkzeuge/Befehle/Dienste, aber keine Fähigkeiten                          |

Weitere Informationen zum Fähigkeitsmodell finden Sie unter [Plugin-Formen](/de/plugins/architecture#plugin-shapes).

<Note>
Das Flag `--json` gibt einen maschinenlesbaren Bericht aus, der sich für Skripte und Prüfungen eignet. `inspect --all` stellt eine flottenweite Tabelle mit Spalten für Form, Fähigkeitsarten, Kompatibilitätshinweise, Bundle-Fähigkeiten und Hook-Zusammenfassung dar. `info` ist ein Alias für `inspect`.
</Note>

## Diagnose

```bash
openclaw plugins doctor
```

`doctor` meldet Fehler beim Laden von Plugins, Manifest-/Discovery-Diagnosen, Kompatibilitätshinweise und veraltete Plugin-Konfigurationsreferenzen wie fehlende Plugin-Slots. Wenn der Installationsbaum und die Plugin-Konfiguration fehlerfrei sind, wird `No plugin issues detected.` ausgegeben. Wenn noch veraltete Konfigurationen vorhanden sind, der Installationsbaum ansonsten jedoch fehlerfrei ist, weist die Zusammenfassung darauf hin, statt einen vollständig fehlerfreien Plugin-Zustand zu suggerieren.

Wenn ein konfiguriertes Plugin auf dem Datenträger vorhanden ist, aber durch die Pfadsicherheitsprüfungen des Loaders blockiert wird, behält die Konfigurationsvalidierung den Plugin-Eintrag bei und meldet ihn als `present but blocked`. Beheben Sie die vorhergehende Diagnose zum blockierten Plugin, beispielsweise hinsichtlich des Pfadeigentümers oder der Schreibberechtigungen für alle Benutzer, statt die Konfiguration `plugins.entries.<id>` oder `plugins.allow` zu entfernen.

Führen Sie den Befehl bei Fehlern der Modulstruktur, etwa fehlenden Exporten `register`/`activate`, erneut mit `OPENCLAW_PLUGIN_LOAD_DEBUG=1` aus, um eine kompakte Zusammenfassung der Exportstruktur in die Diagnoseausgabe aufzunehmen.

## Registry

```bash
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins registry --json
```

Die lokale Plugin-Registry ist das persistierte Cold-Read-Modell von OpenClaw für die Identität installierter Plugins, deren Aktivierungsstatus, Quellmetadaten und die Zuständigkeit für Beiträge. Der normale Startvorgang, die Suche nach Provider-Zuständigen, die Klassifizierung der Kanaleinrichtung und das Plugin-Inventar können daraus lesen, ohne Plugin-Laufzeitmodule zu importieren.

Verwenden Sie `plugins registry`, um zu prüfen, ob die persistierte Registry vorhanden, aktuell oder veraltet ist. Verwenden Sie `--refresh`, um sie anhand des persistierten Plugin-Index, der Konfigurationsrichtlinie und der Manifest-/Paketmetadaten neu zu erstellen. Dies ist ein Reparaturpfad und kein Pfad zur Laufzeitaktivierung.

`openclaw doctor --fix` behebt außerdem Abweichungen in verwalteten npm-Installationen im Umfeld der Registry. Wenn ein verwaistes oder wiederhergestelltes `@openclaw/*`-Paket unter einem verwalteten npm-Projekt eines Plugins oder im veralteten flachen Stammverzeichnis für verwaltete npm-Pakete ein gebündeltes Plugin überschattet, entfernt Doctor dieses veraltete Paket und erstellt die Registry neu, sodass der Startvorgang anhand des gebündelten Manifests validiert wird. Wenn ein maßgeblicher Installationsdatensatz eine verwaltete Generation auswählt, aber ältere flache Verzeichnisse oder Generationsverzeichnisse verbleiben, nimmt Doctor diese veralteten Verzeichnisbäume außer Betrieb, damit sie nach dem Neustart des Gateways bereinigt werden können. Doctor verknüpft außerdem das `openclaw`-Paket des Hosts erneut mit verwalteten npm-Plugins, die `peerDependencies.openclaw` deklarieren, sodass paketlokale Laufzeitimporte wie `openclaw/plugin-sdk/*` nach Aktualisierungen oder npm-Reparaturen aufgelöst werden.

## Marketplace

```bash
openclaw plugins marketplace entries
openclaw plugins marketplace entries --offline
openclaw plugins marketplace entries --json
openclaw plugins marketplace entries --feed-profile <name>
openclaw plugins marketplace entries --feed-url <url>
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
openclaw plugins marketplace refresh
openclaw plugins marketplace refresh --feed-profile <name>
openclaw plugins marketplace refresh --feed-url <url>
openclaw plugins marketplace refresh --expected-sha256 <sha256> --json
```

`plugins marketplace entries` listet Einträge aus dem konfigurierten Marketplace-Feed von OpenClaw auf. Standardmäßig wird versucht, den gehosteten Feed abzurufen; andernfalls wird auf den zuletzt akzeptierten Snapshot oder die gebündelten Daten zurückgegriffen. Verwenden Sie `--feed-profile <name>`, um ein bestimmtes konfiguriertes Profil zu lesen, `--feed-url <url>`, um eine explizite URL eines gehosteten Feeds zu lesen, und `--offline`, um den zuletzt akzeptierten Snapshot zu lesen, ohne den Feed abzurufen.

`plugins marketplace refresh` aktualisiert den konfigurierten Snapshot des gehosteten Feeds und meldet, ob OpenClaw gehostete Daten, einen gehosteten Snapshot oder gebündelte Ausweichdaten akzeptiert hat. Verwenden Sie `--expected-sha256`, wenn ein Aufrufer verlangt, dass der Befehl fehlschlägt, sofern nicht eine aktuelle gehostete Nutzlast mit einer festgelegten Prüfsumme übereinstimmt.

Marketplace `list` akzeptiert einen lokalen Marketplace-Pfad, einen `marketplace.json`-Pfad, eine GitHub-Kurzform wie `owner/repo`, eine GitHub-Repository-URL oder eine Git-URL. `--json` gibt die aufgelöste Quellbezeichnung sowie das geparste Marketplace-Manifest und die Plugin-Einträge aus.

Die Marketplace-Aktualisierung lädt einen gehosteten Marketplace-Feed von OpenClaw und persistiert die
validierte Antwort als lokalen Snapshot des gehosteten Feeds. Ohne Optionen wird
das konfigurierte standardmäßige Feed-Profil verwendet. Verwenden Sie `--feed-profile <name>`, um ein
bestimmtes konfiguriertes Profil zu aktualisieren, `--feed-url <url>`, um einen expliziten gehosteten
Feed über seine URL zu aktualisieren, `--expected-sha256 <sha256>`, um eine übereinstimmende Nutzlastprüfsumme zu verlangen
(`sha256:<hex>` oder einen reinen 64-stelligen Hexadezimalwert), und `--json` für
maschinenlesbare Ausgaben. Explizite URLs gehosteter Feeds dürfen weder
Anmeldedaten noch Abfragezeichenfolgen oder Fragmente enthalten. Aktualisierungen ohne festgelegte Prüfsumme können ein
Ergebnis aus einem gehosteten Snapshot oder gebündelten Ausweichdaten melden, ohne dass der Befehl fehlschlägt. Aktualisierungen
mit festgelegter Prüfsumme schlagen fehl, sofern sie keine aktuelle gehostete Nutzlast akzeptieren, und erfolgreiche gehostete
Aktualisierungen schlagen fehl, wenn OpenClaw den validierten Snapshot nicht persistieren kann.

Das integrierte Profil `clawhub-public` erwartet die Nutzlastidentität
`clawhub-official`. OpenClaw wird den öffentlichen Produktionsschlüssel von ClawHub bündeln, nachdem
ClawHub diesen Schlüssel generiert und übergeben hat. Bis dahin
erteilt das integrierte Profil keine Berechtigung zur Installation über signierte Feeds. Öffentliche Schlüssel müssen aus einem vertrauenswürdigen
Release- oder Betreiberkanal stammen und nicht von einem Schlüssel-Endpunkt auf dem Feed-Host.

OpenClaw verifiziert den DSSE-Umschlag und verlangt, wenn ein Profil `feedId` deklariert,
dass die decodierte Nutzlast-ID damit übereinstimmt. Das integrierte Profil `clawhub-public`
deklariert stets seine Identität und verhindert dadurch, dass ein gültiges Dokument für einen anderen
Feed über dieses Profil wiederverwendet wird.

Während der schrittweisen Einführung behalten vorhandene benutzerdefinierte signierte Profile, die `feedId`
auslassen, die Signaturprüfung ohne Bindung an die Nutzlastidentität bei. Neue benutzerdefinierte
Profile sollten `feedId` deklarieren. Die Konfigurationsoberfläche für Feed-Profile wird
separat zusammen mit den von der Control UI benötigten Darstellungsmetadaten eingeführt; ihre
Doctor-Diagnose muss den Betreiber auffordern, eine fehlende Identität anzugeben, und darf
keine aus der Feed-URL ableiten. Diese Vertrauensbindung stellt den außer Betrieb genommenen
Stammschlüssel `marketplaces` nicht wieder her.

## Verwandte Themen

- [Plugins erstellen](/de/plugins/building-plugins)
- [CLI-Referenz](/de/cli)
- [ClawHub](/de/clawhub)
