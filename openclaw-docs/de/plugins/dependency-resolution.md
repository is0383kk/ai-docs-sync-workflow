---
read_when:
    - Sie debuggen die Installation von Plugin-Paketen
    - Sie ändern das Startverhalten von Plugins, Doctor oder die Installation über den Paketmanager
    - Sie verwalten paketierte OpenClaw-Installationen oder gebündelte Plugin-Manifeste
sidebarTitle: Dependencies
summary: Wie OpenClaw Plugin-Pakete installiert und Plugin-Abhängigkeiten auflöst
title: Auflösung von Plugin-Abhängigkeiten
x-i18n:
    generated_at: "2026-07-26T17:56:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw verarbeitet Plugin-Abhängigkeiten ausschließlich zum Installations-/Aktualisierungszeitpunkt. Beim Laden zur Laufzeit wird niemals ein Paketmanager ausgeführt, ein Abhängigkeitsbaum repariert oder das OpenClaw-Paketverzeichnis verändert.

## Aufteilung der Zuständigkeiten

Plugin-Pakete verwalten ihren eigenen Abhängigkeitsgraphen:

- Laufzeitabhängigkeiten befinden sich in `dependencies` oder
  `optionalDependencies` des Plugin-Pakets.
- SDK-/Core-Importe sind Peer- oder von OpenClaw bereitgestellte Importe.
- Lokale Entwicklungs-Plugins bringen ihre eigenen bereits installierten Abhängigkeiten mit.
- npm- und git-Plugins werden in OpenClaw-eigenen Paket-Stammverzeichnissen installiert.

OpenClaw verwaltet nur den Plugin-Lebenszyklus:

- Die Plugin-Quelle ermitteln.
- Das Paket installieren oder aktualisieren, wenn dies ausdrücklich angefordert wird.
- Installationsmetadaten erfassen.
- Den Plugin-Einstiegspunkt laden.
- Bei fehlenden Abhängigkeiten mit einem Fehler abbrechen, der eine konkrete Lösung nennt.

## Installations-Stammverzeichnisse

OpenClaw verwendet stabile Stammverzeichnisse pro Quelle:

- npm-Pakete werden in projektspezifischen Verzeichnissen pro Plugin unter
  `~/.openclaw/npm/projects/<encoded-package>` installiert.
- git-Pakete werden unter `~/.openclaw/git` geklont.
- Lokale/Pfad-/Archivinstallationen werden ohne Reparatur der Abhängigkeiten
  kopiert oder referenziert.

npm-Installationen werden in diesem Projekt-Stammverzeichnis pro Plugin mit Folgendem ausgeführt:

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>` verwendet dasselbe npm-Projekt-Stammverzeichnis pro Plugin
für einen lokalen npm-pack-Tarball: OpenClaw liest die npm-Metadaten des Tarballs,
fügt ihn dem verwalteten Projekt als kopierte `file:`-Abhängigkeit hinzu, führt
die oben gezeigte normale npm-Installation aus und überprüft anschließend die Metadaten
der installierten Sperrdatei, bevor es dem Plugin vertraut. Dieser Pfad dient der
Paketabnahme und dem Nachweis für Release-Kandidaten, bei denen sich ein lokales Pack-Artefakt
wie das simulierte Registry-Artefakt verhalten soll.

Verwenden Sie `npm-pack:`, wenn Sie offizielle oder externe Plugin-Pakete vor der
Veröffentlichung testen. Eine direkte Archiv- oder Pfadinstallation eignet sich für die lokale
Fehlersuche, weist jedoch nicht denselben Abhängigkeitspfad wie ein installiertes npm- oder
ClawHub-Paket nach. `npm-pack:` weist die Struktur der verwalteten Paketinstallation nach;
dies ist für sich genommen jedoch kein Nachweis dafür, dass das Plugin katalogverknüpfter
offizieller Inhalt ist.

Wenn das Verhalten vom Status als gebündeltes Plugin oder vertrauenswürdiges offizielles Plugin
abhängt, kombinieren Sie den lokalen Paketnachweis mit einer kataloggestützten offiziellen
Installation oder einem veröffentlichten Paketpfad, der das offizielle Vertrauen erfasst.
Der Zugriff auf privilegierte Hilfsfunktionen und die Behandlung des vertrauenswürdigen
offiziellen Geltungsbereichs sollten über diesen vertrauenswürdigen Installationspfad validiert
und nicht aus einer lokalen Tarball-Installation abgeleitet werden.

Wenn ein Plugin zur Laufzeit aufgrund eines fehlenden Imports fehlschlägt, korrigieren Sie das
Paketmanifest, statt das verwaltete Projekt manuell zu reparieren. Laufzeitimporte gehören in
`dependencies` oder `optionalDependencies` des Plugin-Pakets; `devDependencies`
werden für verwaltete Laufzeitprojekte nicht installiert. Ein lokales `npm install` innerhalb
von `~/.openclaw/npm/projects/<encoded-package>` kann eine vorübergehende
Diagnose ermöglichen, ist jedoch kein Nachweis für die Paketabnahme, da das Projekt bei der
nächsten Installation oder Aktualisierung anhand der Paketmetadaten neu erstellt wird.

npm kann transitive Abhängigkeiten in `node_modules` des projektspezifischen
Plugin-Verzeichnisses neben das Plugin-Paket hochstufen. OpenClaw überprüft das verwaltete
Projekt-Stammverzeichnis, bevor es der Installation vertraut, und entfernt dieses Projekt bei
der Deinstallation. Daher verbleiben hochgestufte Laufzeitabhängigkeiten innerhalb der
Bereinigungsgrenze dieses Plugins.

Veröffentlichte npm-Plugin-Pakete können `npm-shrinkwrap.json` mitliefern; npm verwendet diese
veröffentlichbare Sperrdatei während der Installation, und das von OpenClaw verwaltete
npm-Projekt-Stammverzeichnis unterstützt sie über den normalen Installationspfad. Veröffentlichbare
Plugin-Pakete im Besitz von OpenClaw müssen eine paketlokale Shrinkwrap-Datei enthalten, die aus
dem veröffentlichten Abhängigkeitsgraphen dieses Pakets erzeugt wurde:

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

Der Generator entfernt Plugin-`devDependencies`, wendet die Workspace-Überschreibungsrichtlinie
an und schreibt `extensions/<id>/npm-shrinkwrap.json` für jedes Plugin mit
`openclaw.release.publishToNpm: true`. Plugin-Pakete von Drittanbietern können ebenfalls
eine Shrinkwrap-Datei mitliefern; OpenClaw verlangt dies nicht für Community-Pakete, aber npm
berücksichtigt sie, wenn sie vorhanden ist.

Bevor Sie ein lokales Paket als Nachweis für einen Release-Kandidaten betrachten, prüfen Sie den
zu installierenden Tarball:

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

Überprüfen Sie bei Änderungen an Abhängigkeiten außerdem, ob eine Produktionsinstallation die
Laufzeitpakete ohne Entwicklungsabhängigkeiten auflösen kann:

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

npm-Plugin-Pakete im Besitz von OpenClaw können außerdem mit explizitem
`bundledDependencies` veröffentlicht werden. Der npm-Veröffentlichungspfad überlagert die
Namensliste der Laufzeitabhängigkeiten, entfernt reine Entwicklungs-Workspace-Metadaten aus dem
veröffentlichten Manifest, führt eine skriptfreie npm-Installation für die paketlokalen
Laufzeitabhängigkeiten aus und verpackt oder veröffentlicht anschließend den Plugin-Tarball
einschließlich dieser Abhängigkeitsdateien. Pakete mit umfangreichen nativen Komponenten
(Codex, ACPX, Copilot, llama.cpp, memory-lancedb, Tlon) deaktivieren dies mit
`openclaw.release.bundleRuntimeDependencies: false`; sie liefern weiterhin eine
Shrinkwrap-Datei aus, aber npm löst Laufzeitabhängigkeiten während der Installation auf, statt
jede Plattformbinärdatei in den Plugin-Tarball einzubetten. Das Stammverzeichnis-Paket
`openclaw` bündelt nicht seinen vollständigen Abhängigkeitsbaum.

Plugins, die `openclaw/plugin-sdk/*` importieren, deklarieren `openclaw` als
Peer-Abhängigkeit. OpenClaw lässt npm keine separate Registry-Kopie des Host-Pakets in einem
verwalteten Projekt installieren, da ein veraltetes Host-Paket die Peer-Auflösung von npm
innerhalb dieses Plugins beeinflussen kann. Verwaltete npm-Installationen überspringen die
npm-Peer-Auflösung/-Materialisierung, und OpenClaw stellt nach der Installation oder
Aktualisierung die pluginlokalen `node_modules/openclaw`-Verknüpfungen für installierte Pakete,
die den Host-Peer deklarieren, erneut her.

git-Installationen klonen oder aktualisieren das Repository und führen anschließend Folgendes aus:

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

Das installierte Plugin wird anschließend aus diesem Paketverzeichnis geladen, sodass die
Auflösung paketlokaler und übergeordneter `node_modules` genauso funktioniert wie bei
einem normalen Node-Paket.

## Lokale Plugins

Lokale Plugins sind von Entwicklern verwaltete Verzeichnisse. OpenClaw führt für sie niemals
`npm install`, `pnpm install` oder eine Reparatur der Abhängigkeiten aus. Wenn ein
lokales Plugin Abhängigkeiten besitzt, installieren Sie diese vor dem Laden im betreffenden Plugin.

Lokale TypeScript-Plugins von Drittanbietern werden als Notfallpfad über Jiti geladen.
Paketierte JavaScript-Plugins und gebündelte interne Plugins werden stattdessen über den nativen
Import/Require-Mechanismus geladen.

## Start und Neuladen

Beim Start des Gateways und beim Neuladen der Konfiguration werden niemals Plugin-Abhängigkeiten
installiert. Dabei werden die Plugin-Installationsdatensätze gelesen, der Einstiegspunkt berechnet
und das Plugin geladen.

Eine zur Laufzeit fehlende Abhängigkeit lässt das Laden des Plugins mit einem Fehler fehlschlagen,
der den Betreiber auf eine explizite Lösung verweist:

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix` bereinigt veralteten, von OpenClaw erzeugten Abhängigkeitszustand und kann
herunterladbare Plugins wiederherstellen, die in lokalen Installationsdatensätzen fehlen, wenn
die Konfiguration weiterhin auf sie verweist. Doctor repariert keine Abhängigkeiten für ein
bereits installiertes lokales Plugin.

## Gebündelte Plugins

Leichtgewichtige und für den Core kritische gebündelte Plugins werden als Teil von OpenClaw
ausgeliefert. Sie sollten entweder keinen umfangreichen Laufzeitabhängigkeitsbaum besitzen oder
als herunterladbares Paket auf ClawHub/npm ausgelagert werden.

Die aktuelle generierte Liste der Plugins, die im Core-Paket ausgeliefert, extern installiert
oder ausschließlich im Quellcode gehalten werden, finden Sie im
[Plugin-Inventar](/de/plugins/plugin-inventory).

Manifeste gebündelter Plugins dürfen kein Staging von Abhängigkeiten anfordern. Umfangreiche oder
optionale Plugin-Funktionalität sollte als normales Plugin paketiert und über denselben
npm-/git-/ClawHub-Pfad wie Plugins von Drittanbietern installiert werden.

In Quellcode-Checkouts behandelt OpenClaw das Repository als pnpm-Monorepo.
Nach `pnpm install` werden gebündelte Plugins aus `extensions/<id>` geladen, damit
paketlokale Workspace-Abhängigkeiten verfügbar sind und Änderungen direkt übernommen werden.
Die Entwicklung in Quellcode-Checkouts unterstützt ausschließlich pnpm; ein einfaches
`npm install` im Repository-Stammverzeichnis bereitet die Abhängigkeiten gebündelter Plugins
nicht vor.

| Installationsstruktur                    | Speicherort des gebündelten Plugins               | Eigentümer der Abhängigkeiten                                                     |
| -------------------------------- | ------------------------------------- | -------------------------------------------------------------------- |
| `npm install -g openclaw`        | Erstellter Laufzeitbaum innerhalb des Pakets | OpenClaw-Paket und explizite Plugin-Installations-/Aktualisierungs-/Doctor-Abläufe     |
| Git-Checkout plus `pnpm install` | `extensions/<id>`-Workspace-Pakete  | Der pnpm-Workspace einschließlich der eigenen Abhängigkeiten jedes Plugin-Pakets |
| `openclaw plugins install ...`   | Verwaltetes npm-Projekt-/git-/ClawHub-Stammverzeichnis  | Der Plugin-Installations-/Aktualisierungsablauf                                       |

## Bereinigung veralteter Strukturen

Ältere OpenClaw-Versionen erzeugten beim Start oder während der Doctor-Reparatur
Abhängigkeits-Stammverzeichnisse für gebündelte Plugins. Die aktuelle Doctor-Bereinigung entfernt
diese veralteten Verzeichnisse und symbolischen Verknüpfungen mit `--fix`, einschließlich
alter `plugin-runtime-deps`-Stammverzeichnisse, globaler Paket-Symlinks im Node-Präfix, die auf
entfernte `plugin-runtime-deps`-Ziele verweisen, `.openclaw-runtime-deps*`-Manifeste, erzeugte
Plugin-`node_modules`, Installations-Staging-Verzeichnisse und paketlokale pnpm-Stores.
Die paketierte Postinstallationsroutine entfernt diese globalen Symlinks ebenfalls, bevor die
veralteten Ziel-Stammverzeichnisse bereinigt werden, sodass Aktualisierungen keine verwaisten
ESM-Paketimporte hinterlassen.

Ältere npm-Installationen verwendeten außerdem ein gemeinsames `~/.openclaw/npm/node_modules`-Stammverzeichnis.
Die aktuellen Installations-, Aktualisierungs-, Deinstallations- und Doctor-Abläufe erkennen dieses
veraltete flache Stammverzeichnis weiterhin, jedoch ausschließlich zur Wiederherstellung und
Bereinigung. Neue npm-Installationen erstellen stattdessen Projekt-Stammverzeichnisse pro Plugin.
