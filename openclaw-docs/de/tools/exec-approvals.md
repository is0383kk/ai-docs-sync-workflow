---
read_when:
    - Ausführungsgenehmigungen oder Zulassungslisten konfigurieren
    - Implementierung der UX für Ausführungsgenehmigungen in der macOS-App
    - Überprüfung von Sandbox-Escape-Prompts und ihren Auswirkungen
sidebarTitle: Exec approvals
summary: 'Host-Ausführungsgenehmigungen: Richtlinienoptionen, Positivlisten und der YOLO-/strikte Workflow'
title: Ausführungsgenehmigungen
x-i18n:
    generated_at: "2026-07-26T18:07:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2bd09746375061232e9094b8803d33859cac4c13c7bde14a059b7d52e48b5de8
    source_path: tools/exec-approvals.md
    workflow: 16
---

Exec-Genehmigungen sind die **Schutzvorkehrung der Begleit-App / des Node-Hosts**, mit der ein
Sandbox-Agent Befehle auf einem echten Host ausführen darf (`gateway` oder `node`). Befehle
werden nur ausgeführt, wenn Richtlinie + Zulassungsliste + (optionale) Benutzergenehmigung übereinstimmen.
Genehmigungen gelten **zusätzlich zu** Tool-Richtlinie und Elevated-Prüfung (Elevated
`full` umgeht sie).

Eine nach Modi gegliederte Übersicht über `deny`, `allowlist`, `ask`, `auto`, `full`,
die Zuordnung von Codex Guardian und ACPX-Harness-Berechtigungen finden Sie unter
[Berechtigungsmodi](/de/tools/permission-modes).

<Note>
Die effektive Richtlinie ist die **strengere** aus `tools.exec.*` und den
Genehmigungsstandardwerten: Genehmigungen können die aus der Konfiguration abgeleiteten Sicherheits-/Abfragevorgaben nur
verschärfen, niemals lockern. Wenn ein Genehmigungsfeld ausgelassen wird, wird der Wert
`tools.exec` verwendet. Host-Exec verwendet außerdem den lokalen Genehmigungsstatus dieses Rechners – ein
hostlokales `ask: "always"` in der Genehmigungsdatei des Ausführungshosts fordert
weiterhin zur Genehmigung auf, selbst wenn Sitzungs- oder Konfigurationsstandardwerte `ask: "on-miss"` anfordern.
</Note>

## Geltungsbereich

Exec-Genehmigungen werden lokal auf dem Ausführungshost erzwungen:

- **Gateway-Host** -> `openclaw`-Prozess auf dem Gateway-Rechner.
- **Node-Host** -> Node-Runner (macOS-Begleit-App oder monitorloser Node-Host).

### Vertrauensmodell

- Vom Gateway authentifizierte Aufrufer sind vertrauenswürdige Operatoren für dieses Gateway.
- Gekoppelte Nodes erweitern diese Fähigkeit vertrauenswürdiger Operatoren auf den Node-Host.
- Genehmigungen verringern das Risiko unbeabsichtigter Ausführungen, sind jedoch **keine** benutzerspezifische Authentifizierungsgrenze oder schreibgeschützte Dateisystemrichtlinie.
- Nach der Genehmigung kann ein Befehl Dateien gemäß den ausgewählten Dateisystemberechtigungen des Hosts oder der Sandbox verändern.
- Genehmigte Ausführungen auf dem Node-Host binden den kanonischen Ausführungskontext: Arbeitsverzeichnis, exakte Argumentliste, Umgebungsbindung, sofern vorhanden, und gegebenenfalls festgelegten Pfad der ausführbaren Datei.
- Bei Shell-Skripten und direkten Datei-Aufrufen über Interpreter/Laufzeiten versucht OpenClaw außerdem, genau einen konkreten lokalen Dateioperanden zu binden. Wenn sich diese Datei nach der Genehmigung, aber vor der Ausführung ändert, wird die Ausführung verweigert, statt abweichenden Inhalt auszuführen.
- Die Dateibindung erfolgt nach bestem Bemühen und bildet nicht jeden Ladepfad von Interpretern/Laufzeiten vollständig ab. Wenn nicht genau eine konkrete lokale Datei identifiziert werden kann, verweigert OpenClaw die Ausstellung einer genehmigungsgestützten Ausführung, statt eine vollständige Abdeckung vorzutäuschen.

### macOS-Aufteilung

- Der **Node-Host-Dienst** leitet `system.run` über lokale IPC an die **macOS-App** weiter.
- Die **macOS-App** erzwingt Genehmigungen und führt den Befehl im UI-Kontext aus.

## Effektive Richtlinie prüfen

| Befehl                                                           | Angezeigte Informationen                                                                 |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `openclaw approvals get` / `--gateway` / `--node <id\|name\|ip>` | Angeforderte Richtlinie, Quellen der Host-Richtlinie und das effektive Ergebnis.          |
| `openclaw exec-policy show`                                      | Zusammengeführte Ansicht des lokalen Rechners.                                           |
| `openclaw exec-policy set` / `preset`                            | Synchronisiert die lokal angeforderte Richtlinie in einem Schritt mit der lokalen Host-Genehmigungsdatei. |

<Note>
Sitzungsspezifische Überschreibungen durch `/exec` sind nicht enthalten. Führen Sie `/exec` in der betreffenden Sitzung aus, um deren aktuelle Standardwerte zu prüfen. Siehe [Sitzungsüberschreibungen](/de/tools/exec#session-overrides-exec).
</Note>

Vollständige CLI-Referenz (Flags, JSON-Ausgabe, Hinzufügen/Entfernen aus der Zulassungsliste): [Genehmigungs-CLI](/de/cli/approvals).

Wenn ein lokaler Geltungsbereich `host=node` anfordert, meldet `exec-policy show` diesen
Geltungsbereich zur Laufzeit als Node-verwaltet, statt die lokale Genehmigungsdatei
als maßgebliche Quelle zu behandeln.

Wenn die UI der Begleit-App **nicht verfügbar** ist, wird jede Anfrage, die
normalerweise eine Eingabeaufforderung auslösen würde, durch den **Abfrage-Fallback** aufgelöst (Standard: `deny`).

<Tip>
Native Chat-Genehmigungsclients können kanalspezifische Interaktionsmöglichkeiten in der
ausstehenden Genehmigungsnachricht bereitstellen. Matrix stellt Reaktionskürzel bereit (`✅` einmal zulassen,
`♾️` immer zulassen, `❌` ablehnen), während `/approve ...` weiterhin als
Fallback in der Nachricht verbleibt.
</Tip>

## Einstellungen und Speicherung

Genehmigungen befinden sich in einer lokalen JSON-Datei auf dem Ausführungshost. Wenn
`OPENCLAW_STATE_DIR` festgelegt ist, folgt die Datei diesem Statusverzeichnis;
andernfalls verwendet sie das standardmäßige OpenClaw-Statusverzeichnis:

```text
$OPENCLAW_STATE_DIR/exec-approvals.json
# andernfalls
~/.openclaw/exec-approvals.json
```

Der standardmäßige Genehmigungs-Socket verwendet dasselbe Stammverzeichnis:
`$OPENCLAW_STATE_DIR/exec-approvals.sock` oder
`~/.openclaw/exec-approvals.sock`, wenn die Variable nicht festgelegt ist.

Statusverzeichnisse sind unabhängige Vertrauensbereiche. Wenn `OPENCLAW_STATE_DIR`
auf einen anderen Speicherort verweist, importiert oder archiviert OpenClaw
`~/.openclaw/exec-approvals.json` niemals; konfigurieren Sie Genehmigungen separat für das
benutzerdefinierte Statusverzeichnis. Doctor importiert außerdem das veraltete
`plugin-binding-approvals.json` nur, wenn es zum aktiven Statusverzeichnis
gehört.

Beispielschema:

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "argPattern": "sha256:argv:...",
          "source": "allow-always",
          "lastUsedAt": 1737150000000,
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        },
        {
          "pattern": "~/Projects/**/bin/git"
        }
      ]
    }
  }
}
```

## Richtlinienoptionen

### `tools.exec.mode`

`tools.exec.mode` ist die bevorzugte normalisierte Richtlinienoberfläche für Host-Exec:

| Wert        | Verhalten                                                                                                                                                                                                             |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | Host-Exec blockieren.                                                                                                                                                                                                 |
| `allowlist` | Nur Befehle aus der Zulassungsliste ohne Nachfrage ausführen.                                                                                                                                                          |
| `ask`       | Zulassungslistenrichtlinie verwenden und bei fehlenden Übereinstimmungen nachfragen.                                                                                                                                  |
| `auto`      | Zulassungslistenrichtlinie verwenden, deterministische Übereinstimmungen direkt ausführen und fehlende Genehmigungen zunächst an den nativen automatischen Prüfer von OpenClaw und anschließend an eine menschliche Genehmigungsroute senden. |
| `full`      | Host-Exec ohne Genehmigungsaufforderungen ausführen.                                                                                                                                                                   |

Doctor migriert das außer Betrieb genommene persistierte Paar `tools.exec.security` / `tools.exec.ask`
zu `tools.exec.mode`.

### `exec.security`

<ParamField path="security" type='"deny" | "allowlist" | "full"'>
  - `deny` - alle Host-Exec-Anfragen blockieren.
  - `allowlist` - nur Befehle aus der Zulassungsliste zulassen.
  - `full` - alles zulassen (entspricht Elevated).

Der Standardwert für Gateway-/Node-Hosts ist `full`; ein `sandbox`-Host verwendet
stattdessen standardmäßig `deny`.
</ParamField>

### `exec.ask`

<ParamField path="ask" type='"off" | "on-miss" | "always"'>
  Konfigurierte Abfragerichtlinie für Host-Exec. Steuert das grundlegende Verhalten der
  Genehmigungsaufforderung aus `tools.exec.ask` und den Host-Genehmigungsstandardwerten.
  Der Standardwert ist `off`. Der aufrufspezifische Tool-Parameter `ask` (siehe
  [Exec-Tool](/de/tools/exec#parameters)) kann diese Grundlage nur verschärfen, und
  vom Kanal stammende Modellaufrufe ignorieren ihn, wenn die effektive Host-Abfrage `off` lautet.

- `off` - niemals nachfragen.
- `on-miss` - nur nachfragen, wenn die Zulassungsliste nicht übereinstimmt.
- `always` - bei jedem Befehl nachfragen. Dauerhaftes Vertrauen durch `allow-always` unterdrückt Aufforderungen **nicht**, wenn der effektive Abfragemodus `always` lautet.

</ParamField>

### `askFallback`

<ParamField path="askFallback" type='"deny" | "allowlist" | "full"'>
  Auflösung, wenn eine Eingabeaufforderung erforderlich, aber keine UI erreichbar ist (oder die
  Eingabeaufforderung das Zeitlimit überschreitet). Bei Auslassung wird standardmäßig `deny` verwendet.

- `deny` - blockieren.
- `allowlist` - nur zulassen, wenn die Zulassungsliste übereinstimmt.
- `full` - zulassen.

</ParamField>

### `tools.exec.strictInlineEval`

<ParamField path="strictInlineEval" type="boolean">
  Wenn `true`, werden Formen der Inline-Codeauswertung selbst dann als ausschließlich genehmigungspflichtig behandelt, wenn die
  Interpreter-Binärdatei selbst in der Zulassungsliste enthalten ist. Zusätzliche Schutzebene für
  Interpreter-Loader, die sich nicht eindeutig einem stabilen Dateioperanden zuordnen lassen.
</ParamField>

Beispiele, die der strikte Modus erfasst: `python -c`, `node -e`/`--eval`/`-p`,
`ruby -e`, `perl -e`/`-E`, `php -r`, `lua -e`, `osascript -e` (außerdem die Inline-Formen
`awk`, `sed`, `make`, `find -exec` und `xargs`).

Im strikten Modus benötigen diese Befehle eine Prüfer- oder ausdrückliche Genehmigung. Mit
`tools.exec.mode: "auto"` kann der Prüfer eine einzelne Ausführung mit geringem Risiko genehmigen, wenn
für den Befehl ein durchsetzbarer Plan vorliegt; andernfalls fragt OpenClaw einen Menschen.
`Codex app-server`-Befehlsgenehmigungen, die den Prüfer-Fallback erreichen, fragen einen
Menschen, da ihre Genehmigungsanfragen keine durchsetzbare aufgelöste
ausführbare Datei offenlegen.
`allow-always` speichert keine neuen Zulassungslisteneinträge für Inline-Auswertungsbefehle.

### `tools.exec.commandHighlighting`

<ParamField path="commandHighlighting" type="boolean" default="false">
  Nur Darstellung: Wenn aktiviert, kann OpenClaw vom Parser abgeleitete
  Befehlsspannen anhängen, damit Web-Genehmigungsaufforderungen Befehlstoken hervorheben können. Dies
  ändert **nicht** `security`, `ask`, den Zulassungslistenabgleich, das Verhalten bei strikter Inline-Auswertung,
  die Weiterleitung von Genehmigungen oder die Befehlsausführung.
</ParamField>

Global unter `tools.exec.commandHighlighting` oder pro Agent unter
`agents.entries.*.tools.exec.commandHighlighting` festlegen.

## YOLO-Modus (ohne Genehmigung)

Um Host-Exec ohne Genehmigungsaufforderungen auszuführen, müssen Sie **beide** Richtlinienebenen öffnen:
die angeforderte Exec-Richtlinie in der OpenClaw-Konfiguration (`tools.exec.*`) **und**
die hostlokale Genehmigungsrichtlinie in der Genehmigungsdatei des Ausführungshosts.

Ein ausgelassenes `askFallback` verwendet standardmäßig `deny`. Setzen Sie Host-`askFallback` ausdrücklich auf `full`,
wenn eine Genehmigungsaufforderung ohne UI als Fallback eine Zulassung verwenden soll.

| Ebene               | YOLO-Einstellung            |
| ------------------- | --------------------------- |
| `tools.exec.mode`  | `full` auf `gateway`/`node` |
| Host-`askFallback` | `full`                     |

<Warning>
**Wichtige Unterscheidungen:**

- `tools.exec.host=auto` legt fest, **wo** exec ausgeführt wird: in der Sandbox, sofern verfügbar, andernfalls auf dem Gateway.
- YOLO legt fest, **wie** Host-exec genehmigt wird: `security=full` plus `ask=off`.
- YOLO fügt **keine** separate heuristische Genehmigungsschranke für Befehlsverschleierung oder Ablehnungsebene für Skript-Preflights zusätzlich zur konfigurierten Host-exec-Richtlinie hinzu.
- `auto` macht das Gateway-Routing nicht zu einer frei wählbaren Überschreibung aus einer Sandbox-Sitzung. Eine `host=node`-Anforderung pro Aufruf ist von `auto` aus zulässig; `host=gateway` ist von `auto` aus nur zulässig, wenn keine Sandbox-Laufzeit aktiv ist. Für einen stabilen, nicht automatischen Standardwert legen Sie `tools.exec.host` fest oder verwenden Sie ausdrücklich `/exec host=...`.

</Warning>

CLI-basierte Provider, die einen eigenen nicht interaktiven Berechtigungsmodus bereitstellen,
können dieser Richtlinie folgen. Die Claude CLI fügt
`--permission-mode bypassPermissions` hinzu, wenn die effektive exec-
Richtlinie von OpenClaw YOLO ist. Bei von OpenClaw verwalteten Claude-Live-Sitzungen ist die
effektive exec-Richtlinie von OpenClaw gegenüber dem nativen Berechtigungsmodus von Claude maßgeblich:
YOLO normalisiert Live-Starts auf `--permission-mode bypassPermissions`, und
eine restriktive effektive exec-Richtlinie normalisiert Live-Starts auf
`--permission-mode default`, selbst wenn die unverarbeiteten Argumente des Claude-Backends einen anderen
Modus angeben.

Wenn Sie eine konservativere Einrichtung wünschen, verschärfen Sie die exec-Richtlinie von OpenClaw wieder auf
`allowlist` / `on-miss` oder `deny`.

### Dauerhafte „Nie nachfragen“-Einrichtung für den Gateway-Host

<Steps>
  <Step title="Angeforderte Konfigurationsrichtlinie festlegen">
    ```bash
    openclaw config set tools.exec.host gateway
    openclaw config set tools.exec.mode full
    openclaw gateway restart
    ```
  </Step>
  <Step title="Host-Genehmigungsdatei entsprechend anpassen">
    ```bash
    openclaw approvals set --stdin <<'EOF'
    {
      version: 1,
      defaults: {
        security: "full",
        ask: "off",
        askFallback: "full"
      }
    }
    EOF
    ```
  </Step>
</Steps>

### Lokale Kurzform

```bash
openclaw exec-policy preset yolo
```

Aktualisiert sowohl die lokale `tools.exec.host/security/ask` als auch die Standardwerte der lokalen Genehmigungsdatei
(einschließlich `askFallback: "full"`). Dies ist absichtlich
nur lokal wirksam. Um Genehmigungen für Gateway-Hosts oder Node-Hosts remote zu ändern, verwenden Sie
`openclaw approvals set --gateway` oder `openclaw approvals set --node
<id|name|ip>`.

Weitere integrierte Voreinstellungen: `cautious` (`host=gateway`, `security=allowlist`,
`ask=on-miss`, `askFallback=deny`) und `deny-all` (`host=gateway`,
`security=deny`, `ask=off`, `askFallback=deny`). Wenden Sie sie auf dieselbe Weise an:
`openclaw exec-policy preset cautious`.

Um einzelne Felder anstelle einer vollständigen Voreinstellung festzulegen, verwenden Sie
`openclaw exec-policy set --host <auto|sandbox|gateway|node> --security
<deny|allowlist|full> --ask <off|on-miss|always> --ask-fallback
<deny|allowlist|full>` mit einer beliebigen Teilmenge dieser Flags.

### Node-Host

Wenden Sie stattdessen dieselbe Genehmigungsdatei auf dem Node an:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

<Note>
**Einschränkungen bei rein lokaler Verwendung:**

- `openclaw exec-policy` synchronisiert keine Node-Genehmigungen.
- `openclaw exec-policy set --host node` wird abgelehnt.
- Node-exec-Genehmigungen werden zur Laufzeit vom Node abgerufen; daher müssen auf Nodes ausgerichtete Aktualisierungen `openclaw approvals --node ...` verwenden.

</Note>

### Nur für die Sitzung geltende Kurzform

- `/exec security=full ask=off` ändert nur die aktuelle Sitzung.
- `/elevated full` ist eine Notfall-Kurzform, die exec-Genehmigungen nur dann überspringt,
  wenn sowohl die angeforderte Richtlinie als auch die Host-Genehmigungsdatei zu
  `security: "full"` und `ask: "off"` aufgelöst werden. Bei einer strengeren Host-Datei wie `ask:
"always"` erfolgt weiterhin eine Abfrage.

Wenn die Host-Genehmigungsdatei strenger als die Konfiguration bleibt, hat weiterhin die strengere Host-
Richtlinie Vorrang.

## Positivliste (pro Agent)

Positivlisten gelten **pro Agent**. Wenn mehrere Agents vorhanden sind, wechseln Sie in der macOS-App zu dem Agent,
den Sie bearbeiten möchten. Muster werden als Glob-Ausdrücke abgeglichen.

Muster können Globs für aufgelöste Binärdateipfade oder Globs für reine Befehlsnamen sein.
Reine Namen stimmen nur mit Befehlen überein, die über `PATH` aufgerufen werden. Daher kann `rg` mit
`/opt/homebrew/bin/rg` übereinstimmen, wenn der Befehl `rg` lautet, jedoch **nicht** mit `./rg` oder
`/tmp/rg`. Verwenden Sie einen Pfad-Glob, um einem bestimmten Speicherort einer Binärdatei zu vertrauen.

Veraltete `agents.default`-Einträge werden beim Laden zu `agents.main` migriert.
Shell-Ketten wie `echo ok && pwd` erfordern weiterhin, dass jedes Segment der obersten Ebene
die Regeln der Positivliste erfüllt.

Beispiele:

- `rg`
- `~/Projects/**/bin/peekaboo`
- `~/.local/bin/*`
- `/opt/homebrew/bin/rg`

### Argumente mit argPattern einschränken

Fügen Sie `argPattern` hinzu, wenn ein Positivlisteneintrag mit einer Binärdatei und einer
bestimmten Argumentstruktur übereinstimmen soll. OpenClaw verwendet auf jedem Host die Semantik regulärer
ECMAScript-Ausdrücke (JavaScript) und wertet den Ausdruck anhand
der analysierten Befehlsargumente aus, wobei das ausführbare Token (`argv[0]`) ausgeschlossen wird.
Bei manuell erstellten Einträgen werden Argumente mit einem einzelnen Leerzeichen verbunden. Verankern Sie
daher das Muster, wenn Sie eine exakte Übereinstimmung benötigen.

```json
{
  "version": 1,
  "agents": {
    "main": {
      "allowlist": [
        {
          "pattern": "python3",
          "argPattern": "^safe\\.py$"
        }
      ]
    }
  }
}
```

Dieser Eintrag erlaubt `python3 safe.py`; `python3 other.py` stimmt nicht mit der Positivliste
überein. Wenn außerdem ein reiner Pfadeintrag für dieselbe Binärdatei vorhanden ist, können nicht übereinstimmende
Argumente weiterhin auf diesen reinen Pfadeintrag zurückfallen. Lassen Sie den reinen
Pfadeintrag weg, wenn die Binärdatei auf die angegebenen Argumente beschränkt werden soll.

Durch Genehmigungsabläufe gespeicherte Einträge verwenden ein internes Trennzeichenformat für den exakten
argv-Abgleich. Verwenden Sie vorzugsweise die Benutzeroberfläche oder den Genehmigungsablauf, um diese Einträge neu
zu erzeugen, statt den codierten Wert manuell zu bearbeiten. Wenn OpenClaw argv
für ein Befehlssegment nicht analysieren kann, stimmen Einträge mit `argPattern` nicht überein.

Generierte `allow-always`-Einträge sind an argv gebunden. Neue generierte Einträge enthalten
`argPattern`; ältere generierte reine Pfadeinträge werden ignoriert und benötigen eine neue
Genehmigung. Lassen Sie bei einer manuellen reinen Pfadregel sowohl `source` als auch `argPattern` weg.

Jeder Positivlisteneintrag unterstützt:

| Feld               | Bedeutung                                                                    |
| ------------------ | ---------------------------------------------------------------------------- |
| `pattern`          | Glob für einen aufgelösten Binärdateipfad oder reinen Befehlsnamen                       |
| `argPattern`       | ECMAScript-argv-RegEx oder generierter Hash für exakte argv; nicht angegeben bedeutet nur Pfad |
| `id`               | Stabile undurchsichtige ID; wird bei Fehlen als UUID generiert                         |
| `source`           | Quelle des generierten Eintrags, etwa `allow-always`; bei manuellen Einträgen weglassen |
| `commandText`      | Veraltete Klartexteingabe; wird beim Laden verworfen                              |
| `lastUsedAt`       | Zeitstempel der letzten Verwendung                                                |
| `lastUsedCommand`  | Letzter übereinstimmender Befehl; bei generierten gehashten argv-Einträgen nicht angegeben |
| `lastResolvedPath` | Letzter aufgelöster Binärdateipfad                                                |

## Skill-CLIs automatisch zulassen

Wenn **Skill-CLIs automatisch zulassen** (`autoAllowSkills`) aktiviert ist, werden ausführbare Dateien,
auf die bekannte Skills verweisen, auf Nodes als in der Positivliste enthalten behandelt (macOS-Node
oder Headless-Node-Host). Dazu wird `skills.bins` über den Gateway-RPC verwendet, um
die Liste der Skill-Binärdateien abzurufen. Deaktivieren Sie dies, wenn Sie strikt manuelle
Positivlisten wünschen.

<Warning>
- Dies ist eine **implizite Komfort-Positivliste**, die von manuellen Pfad-Positivlisteneinträgen getrennt ist.
- Sie ist für vertrauenswürdige Betreiberumgebungen vorgesehen, in denen Gateway und Node derselben Vertrauensgrenze angehören.
- Wenn Sie strikt explizites Vertrauen benötigen, behalten Sie `autoAllowSkills: false` bei und verwenden Sie ausschließlich manuelle Pfad-Positivlisteneinträge.

</Warning>

## Sichere Binärdateien und Weiterleitung von Genehmigungen

Einzelheiten zu sicheren Binärdateien (dem schnellen, ausschließlich stdin-basierten Pfad), zur Interpreter-Bindung und
zur Weiterleitung von Genehmigungsabfragen an Slack/Discord/Telegram (oder deren Ausführung als
native Genehmigungsclients) finden Sie unter
[Exec-Genehmigungen – erweitert](/de/tools/exec-approvals-advanced).

## Bearbeitung in der Control UI

Verwenden Sie die Karte **Control UI -> Nodes -> Exec approvals**, um Standardwerte,
agentenspezifische Überschreibungen und Positivlisten zu bearbeiten. Wählen Sie einen Bereich (Defaults oder einen Agent),
passen Sie die Richtlinie an, fügen Sie Positivlistenmuster hinzu oder entfernen Sie sie und wählen Sie anschließend **Save**. Die Benutzeroberfläche
zeigt Metadaten zur letzten Verwendung jedes Musters an, damit Sie die Liste übersichtlich halten können.

Die Zielauswahl wählt **Gateway** (lokale Genehmigungen) oder einen **Node**.
Nodes müssen `system.execApprovals.get/set` bereitstellen (macOS-App oder Headless-
Node-Host). Wenn ein Node exec-Genehmigungen noch nicht bereitstellt, bearbeiten Sie seine
lokale Genehmigungsdatei direkt.

Einige Node-Hosts, einschließlich des Windows-Begleitprogramms, verwenden ein anderes Format für Genehmigungsrichtlinien.
Die Control UI zeigt diese hostnativen Richtlinien schreibgeschützt an. Verwenden Sie die
Begleit-App oder `openclaw approvals set --node <id|name|ip>` mit der nativen
Richtlinienstruktur, um sie zu bearbeiten; siehe [Genehmigungs-CLI](/de/cli/approvals).

CLI: `openclaw approvals` unterstützt die Bearbeitung von Gateways oder Nodes – siehe
[Genehmigungs-CLI](/de/cli/approvals).

## Genehmigungsablauf

Wenn eine Abfrage erforderlich ist, überträgt das Gateway
`exec.approval.requested` an Betreiberclients. Die Control UI und die macOS-
App lösen sie über `exec.approval.resolve` auf; anschließend leitet das Gateway die
genehmigte Anforderung an den Node-Host weiter.

Bei `host=node` enthalten Genehmigungsanforderungen eine kanonische `systemRunPlan`-
Nutzlast. Das Gateway verwendet diesen Plan beim Weiterleiten genehmigter `system.run`-Anforderungen als maßgeblichen
Kontext für Befehl/cwd/Sitzung:

- Der Node-exec-Pfad erstellt im Voraus einen kanonischen Plan.
- Der Genehmigungsdatensatz speichert diesen Plan und seine Bindungsmetadaten.
- Nach der Genehmigung verwendet der abschließend weitergeleitete `system.run`-Aufruf den gespeicherten Plan erneut, anstatt späteren Änderungen des Aufrufers zu vertrauen.
- Wenn der Aufrufer `command`, `rawCommand`, `cwd`, `agentId` oder `sessionKey` ändert, nachdem die Genehmigungsanforderung erstellt wurde, lehnt das Gateway die weitergeleitete Ausführung wegen einer Genehmigungsabweichung ab.

## Systemereignisse und Ablehnungen

Der exec-Lebenszyklus sendet eine `Exec finished`-Systemnachricht an die Sitzung des Agent,
nachdem der Node den Abschluss gemeldet hat. OpenClaw kann außerdem eine
Fortschrittsmeldung ausgeben, sobald eine Genehmigung erteilt wurde und
`tools.exec.approvalRunningNoticeMs` verstrichen ist (Standardwert `10000`; `0` deaktiviert
sie). Abgelehnte exec-Genehmigungen sind für den Host-Befehl endgültig: Der Befehl
wird nicht ausgeführt.

- Bei asynchronen Genehmigungen des Haupt-Agent mit einer Ursprungssitzung sendet OpenClaw
  die Ablehnung als interne Folgenachricht an diese Sitzung zurück, damit der
  Agent nicht weiter auf den asynchronen Befehl wartet und keine Reparatur
  eines fehlenden Ergebnisses auslöst.
- Wenn keine Sitzung vorhanden ist oder die Sitzung nicht fortgesetzt werden kann, kann OpenClaw
  dem Betreiber oder dem direkten Chat-Pfad dennoch eine knappe Ablehnung
  melden.
- Ablehnungen für Subagent- und Cron-Sitzungen werden nicht an diese
  Sitzung zurückgesendet.

Exec-Genehmigungen auf dem Gateway-Host geben dasselbe Abschluss-Lebenszyklusereignis aus.
Genehmigungspflichtige exec-Vorgänge verwenden die Genehmigungs-ID erneut, um die ausstehende
Anforderung ihrer Abschluss-/Ablehnungsnachricht zuzuordnen (`Exec finished (gateway
id=...)` / `Exec denied (gateway id=...)`).

## Auswirkungen

- **`full`** ist leistungsfähig; verwenden Sie nach Möglichkeit Positivlisten.
- **`ask`** bezieht Sie weiterhin ein und ermöglicht zugleich schnelle Genehmigungen.
- Agentenspezifische Positivlisten verhindern, dass die Genehmigungen eines Agent auf andere übergreifen.
- Genehmigungen gelten nur für Host-exec-Anforderungen von **autorisierten Absendern**. Nicht autorisierte Absender können `/exec` nicht auslösen.
- `/exec security=full` ist eine Komfortfunktion auf Sitzungsebene für autorisierte Betreiber und überspringt Genehmigungen absichtlich. Um Host-exec strikt zu blockieren, setzen Sie die Genehmigungssicherheit auf `deny` oder verweigern Sie das Werkzeug `exec` über die Werkzeugrichtlinie.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Exec-Genehmigungen – erweitert" href="/de/tools/exec-approvals-advanced" icon="gear">
    Sichere Binärdateien, Interpreter-Bindung und Weiterleitung von Genehmigungen an den Chat.
  </Card>
  <Card title="Exec-Tool" href="/de/tools/exec" icon="terminal">
    Tool zur Ausführung von Shell-Befehlen.
  </Card>
  <Card title="Erweiterter Modus" href="/de/tools/elevated" icon="shield-exclamation">
    Notfallzugriff, der auch Genehmigungen überspringt.
  </Card>
  <Card title="Sandboxing" href="/de/gateway/sandboxing" icon="box">
    Sandbox-Modi und Arbeitsbereichszugriff.
  </Card>
  <Card title="Sicherheit" href="/de/gateway/security" icon="lock">
    Sicherheitsmodell und Härtung.
  </Card>
  <Card title="Sandbox vs. Tool-Richtlinie vs. erweiterter Modus" href="/de/gateway/sandbox-vs-tool-policy-vs-elevated" icon="sliders">
    Wann welches Steuerelement verwendet werden sollte.
  </Card>
  <Card title="Skills" href="/de/tools/skills" icon="sparkles">
    Durch Skills gestütztes Verhalten zur automatischen Zulassung.
  </Card>
</CardGroup>
