---
read_when:
    - Sie erstellen oder validieren ein CLAW.md-Manifest
    - Sie möchten einen Agenten aus einer Claw in der Vorschau anzeigen oder hinzufügen
    - Sie müssen das Eigentum, die Abweichungen oder das Bereinigungsverhalten von Claw überprüfen.
summary: Experimentelle Claw-Agent-Pakete erstellen, hinzufügen, aktualisieren und entfernen
title: Klauen
x-i18n:
    generated_at: "2026-07-26T17:41:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da4b52bdee2b4cf4898677aadeeabb2c0cf98e7c3c53cec6f0b4c6d0b8ab3ae5
    source_path: cli/claws.md
    workflow: 16
---

# `openclaw claws`

Eine Claw ist eine versionierte Einrichtung für einen neuen OpenClaw-Agenten. Sie kann die
portable Identität des Agenten, Workspace-Dateien, Skills, Plugins, MCP-Server und
Cron-Aufträge beschreiben. Harness-spezifische Agenteneinstellungen können in einem referenzierten
Paketprofil enthalten sein. Eine Claw ersetzt oder verändert keinen bestehenden Agenten.

Claws sind experimentell. Ihr Schema, ihre Befehlsausgabe und ihr Lebenszyklus können sich ändern.
Aktivieren Sie die Befehlsoberfläche ausdrücklich:

```bash
export OPENCLAW_EXPERIMENTAL_CLAWS=1
```

Die aktuelle CLI liest ein lokales Paketverzeichnis, `CLAW.md` oder ein gruppiertes JSON-Manifest.
Das Veröffentlichen, Suchen und Installieren vollständiger Claws über ClawHub bilden einen
separaten Registry-Zweig und sind noch nicht Teil dieser Befehlsoberfläche.

## Ein Claw-Paket erstellen

Ein Paket enthält `package.json`, ein `CLAW.md`-Manifest und alle Profile oder
Workspace-Begleitdateien, auf die dieses Manifest verweist:

```json
{
  "name": "@acme/incident-triage-claw",
  "version": "1.0.0",
  "type": "module",
  "openclaw": { "claw": "CLAW.md" }
}
```

`CLAW.md` beginnt mit YAML-Frontmatter. Sein Markdown-Textkörper beschreibt die Claw
für Menschen und ist nicht Teil der Agentenkonfiguration:

```md
---
schemaVersion: 1
agent:
  id: incident-triage
  name: Vorfalltriage
metadata:
  openclaw.config: profiles/openclaw.yml
workspace:
  bootstrapFiles: {}
packages: []
mcpServers: {}
cronJobs: []
---

# Vorfalltriage

Erstellt einen Agenten zum Prüfen und Weiterleiten von Vorfällen.
```

`metadata` ist eine String-zu-String-Zuordnung für portable Verbraucherhinweise. Der
Schlüssel `openclaw.config` von OpenClaw verweist auf ein optionales, paketrelatives YAML-Profil. Der
exportierte Standardwert ist `profiles/openclaw.yml`; der Verweis ist maßgeblich, sodass ein
Paket einen anderen sicheren relativen Pfad mit `.yml` oder `.yaml` wählen kann.

```yaml
schemaVersion: 1
agent:
  tools:
    profile: coding
    alsoAllow: [cron]
    deny: [exec]
    fs:
      workspaceOnly: true
  memory:
    search:
      enabled: true
      rememberAcrossConversations: true
      sources: [memory, sessions]
```

Dieses Profil existiert nur innerhalb des Claw-Pakets. OpenClaw validiert und verwendet es
beim Prüfen, Hinzufügen, Aktualisieren und Exportieren dieser Claw; es wird nicht
in den normalen OpenClaw-Konfigurationspfad des Benutzers kopiert. Andere Harnesses können
den namensraumgebundenen Metadatenschlüssel ignorieren und die portablen Manifestfelder verwenden.

Dasselbe strikte Schema der Version 1 akzeptiert weiterhin gruppierte JSON-Manifeste.
Gruppiertes JSON verwendet denselben `metadata.openclaw.config`-Verweis, anstatt
eine zweite Kopie des OpenClaw-Profils einzubetten. Die übrigen Schemafragmente
auf dieser Seite verwenden JSON; entsprechende Schlüssel sind im `CLAW.md`-Frontmatter verfügbar.

Das OpenClaw-Paketprofil kann jedes integrierte Werkzeugprofil auswählen, das von
der ausgeführten OpenClaw-Version registriert ist, und es anschließend mit `alsoAllow`, `deny` und
`tools.fs.workspaceOnly: true` verfeinern. Eine Claw kann dieses Feld nicht auf `false` setzen und
die Dateisystembeschränkung des Hosts abschwächen. `tools.allow` bleibt als
explizite Zulassungsliste verfügbar, kann jedoch nicht mit `alsoAllow` kombiniert werden. Eine Claw kann außerdem
`memory.search.enabled` festlegen, die portablen Quellen `memory` und `sessions` auswählen
und mit `rememberAcrossConversations` konversationsübergreifenden Speicher aktivieren.
Die Angabe der Quelle `sessions` erfordert diese Aktivierung.
Die Host-Richtlinie beschränkt diese Einstellungen weiterhin, und Claws enthalten keine benutzerdefinierten
Profildefinitionen, Provider, Anmeldedaten, Bindungen oder lokalen Speicherpfade.
Das referenzierte Profil ist auf 256 KiB begrenzt, muss JSON-kompatibles YAML sein, darf
keine Aliase, Anker, Tags oder Zusammenführungsschlüssel verwenden und muss eine reguläre,
nicht symbolisch und nicht fest verknüpfte Datei innerhalb des Pakets sein.

Paket- und Workspace-Pfade müssen innerhalb des Paketstammverzeichnisses bleiben. Manifeste sind
auf 1 MiB, Paketmetadaten auf 256 KiB begrenzt, und Workspace-Quellen erzwingen
separate Grenzwerte pro Datei und insgesamt. Workspace-Quellen lehnen außerdem symbolisch verknüpfte
übergeordnete Verzeichnisse ab.

Workspace-Dateien werden nach Pfad deklariert und aus Paketbegleitdateien gelesen. Bootstrap-Dateien
wie `SOUL.md` verwenden benannte Einträge; zusätzliche Dateien verwenden paketrelative
Quellen und Workspace-relative Ziele:

```json
{
  "workspace": {
    "bootstrapFiles": {
      "SOUL.md": { "source": "workspace/SOUL.md" }
    },
    "files": [
      {
        "source": "workspace/reference/policy.md",
        "path": "reference/policy.md"
      }
    ]
  }
}
```

Skills und Plugins verwenden exakte ClawHub-Versionen:

```json
{
  "packages": [
    {
      "kind": "skill",
      "source": "clawhub",
      "ref": "incident-triage",
      "version": "1.0.0"
    },
    {
      "kind": "plugin",
      "source": "clawhub",
      "ref": "@acme/audit-plugin",
      "version": "2.0.0"
    }
  ]
}
```

Der Probelauf verwendet die bestehenden Vorabprüfpfade für Skills und Plugins, um das
exakte Artefakt, seine Integrität und etwaige ClawHub-Vertrauenswarnungen vor der Zustimmung zu ermitteln. Die
Warnung bleibt im integritätsgebundenen Plan sichtbar. Die Anwendung installiert fehlende Artefakte
oder verwendet passende erneut und zeichnet auf, ob die Claw die jeweilige Ressource eingeführt oder
referenziert hat. Plugins bleiben prozessweite OpenClaw-Funktionen und sind keine
agentenspezifischen Installationen.

Cron-Aufträge deklarieren geplante Arbeiten für den neuen Agenten:

```json
{
  "cronJobs": [
    {
      "id": "daily-summary",
      "name": "Tägliche Vorfallzusammenfassung",
      "schedule": { "cron": "0 9 * * *", "timezone": "UTC" },
      "session": "isolated",
      "message": "Fassen Sie aktive Vorfälle zusammen."
    }
  ]
}
```

Claws verwenden den bestehenden Gateway-Zeitplaner und binden erstellte Aufträge an den neuen
Agenten. Vorschau, Herkunft, Status und Entfernung decken diese Aufträge ab, ohne
das Verhalten gewöhnlicher Cron-Befehle zu ändern. Bei der Entfernung wird der aktuelle Auftrag
über das Gateway erneut gelesen und beibehalten, wenn sich seine verwaltete Definition nach
der Planung geändert hat.

MCP-Deklarationen verwenden das bestehende `mcp.servers`-Konfigurationsmodell:

```json
{
  "mcpServers": {
    "statuspage": {
      "command": "npx",
      "args": ["--yes", "@acme/statuspage-mcp@1.0.0"],
      "env": { "STATUSPAGE_TOKEN": "${STATUSPAGE_TOKEN}" }
    }
  }
}
```

Umgebungsreferenzen bleiben Referenzen; Claws betten keine aufgelösten geheimen
Werte ein. Eine kollisionsfreie Deklaration wird verwaltet, während eine exakt übereinstimmende bestehende
oder gemeinsam genutzte Deklaration referenziert wird. Vorschau, Herkunft, Status, Export und
Entfernung folgen derselben Eigentümerschaftsrichtlinie wie andere Claw-Ressourcen.

## Prüfen und Vorschau anzeigen

Validieren Sie die Quelle, ohne lokale Änderungen zu planen:

```bash
openclaw claws inspect ./incident-triage.claw.json
```

Zeigen Sie eine Vorschau aller vorgeschlagenen Lebenszyklusaktionen an:

```bash
openclaw claws add ./incident-triage.claw.json --dry-run --json
```

Der Plan meldet den abgeleiteten Agenten und Workspace, jede vorgeschlagene Aktion,
Voraussetzungen, Blockierungen, unterschiedliche Funktionserweiterungen und einen `planIntegrity`-
Digest. Funktionsdatensätze zeigen die exakte Auswirkung auf Paket, MCP, geplante Arbeit, Sandbox,
Werkzeug oder Heartbeat. Prüfen Sie den Plan, bevor Sie den Agenten erstellen:

```bash
openclaw claws add ./incident-triage.claw.json \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

`--yes` allein reicht nicht aus. OpenClaw erstellt den Plan neu und lehnt die Zustimmung ab,
wenn sich Quelle, Ziel oder aktuelle Konfiguration nach der Vorschau geändert haben. Verwenden Sie
`--agent-id` oder `--workspace` sowohl bei der Vorschau als auch bei der Anwendung, wenn Paketstandardwerte
mit dem lokalen Zustand kollidieren. Übergeben Sie für temporäre Profile und parallele Validierung
explizit `--workspace`; `OPENCLAW_STATE_DIR` verlagert den Laufzeitzustand, ändert jedoch
nicht den standardmäßigen Workspace-Speicherort.

Das Hinzufügen einer Claw erstellt den neuen Agenten und die Workspace-Konfiguration, schreibt deklarierte
Workspace-Dateien, installiert deklarierte Skill- und Plugin-Artefakte oder verwendet sie erneut und
zeichnet die Herkunft von Paket, MCP und Cron auf. Bestehende Dateien werden nicht überschrieben,
und Wiederholungsversuche schlagen sicher fehl, wenn sich verwaltete Inhalte verändert haben.

## Installierten Zustand prüfen

```bash
openclaw claws status
openclaw claws status incident-triage --json
openclaw doctor
```

`status` vergleicht den installierten Agenten und dessen aufgezeichnete Workspace-, Paket-, MCP-
und Cron-Herkunft mit dem aktuellen Zustand. Es meldet unvollständige Installationen, fehlende
Ressourcen und Abweichungen, ohne den lokalen Zustand zu ändern. `openclaw doctor` ergänzt
Claw-spezifische Diagnosen für unvollständige Eigentümerschaftsdatensätze, unsichere verwaltete
Dateien und Cron-Aufträge, die nicht durch den aktuellen Gateway-Bestand bestätigt werden können.

Die Claw-Herkunft unterscheidet zwei Beziehungen:

- **Verwaltet:** Die Claw hat die Ressource eingeführt und verwaltet sie derzeit. Sie kommt
  für eine Bereinigung infrage, wenn sie unverändert ist und kein kollidierender Eigentümer verbleibt.
- **Referenziert:** Die Ressource bestand unabhängig oder wird gemeinsam genutzt. Bei der Entfernung
  wird die Referenz dieser Claw freigegeben und die Ressource standardmäßig beibehalten.

Dies ist kein Referenzzähler. Gewöhnliche Plugin-, Skill- und Agentenbefehle behalten
ihr bestehendes Verhalten bei; Claws ergänzen darüber hinaus Herkunftsinformationen und abgesicherte Lebenszyklusoperationen.

## Eine installierte Claw aktualisieren

Standardmäßig verwendet die Aktualisierung die Quelle, die beim Hinzufügen der Claw aufgezeichnet wurde. Verwenden Sie
`--from`, wenn diese Quelle verschoben wurde oder ein anderes Paketverzeichnis getestet werden soll:

```bash
openclaw claws update incident-triage --dry-run --json
openclaw claws update incident-triage \
  --from ./incident-triage-next \
  --dry-run --json
```

Der Plan vergleicht die aktuelle Herkunft und den aktuellen Zustand mit dem Zielmanifest.
Er meldet Änderungen an Agent, Workspace, Paket, MCP, Cron und Eigentümerschaft,
einschließlich Funktionserweiterungen und Blockierungen. Funktionserweiterungen besitzen
separate maschinenlesbare Datensätze und `!`-Zeilen mit exakten geschwärzten Auswirkungen in
der menschenlesbaren Ausgabe. Aufgelöste Paketintegrität, Installationsidentität und etwaige
Vertrauenswarnungen sind enthalten. Das Entfernen einer Paketdeklaration gibt die Verknüpfung dieser Claw
frei, ohne das Artefakt während der Aktualisierung zu deinstallieren. Die abschließende
exakte Bestätigung mit `planIntegrity` bindet sowohl diesen offengelegten Satz als auch gewöhnliche
Inhaltsänderungen. Hosts können dieselben Datensätze für einen separaten Dialog oder eine
zusammengefasste Prüfung mehrerer Agenten verwenden. Wenden Sie den exakt geprüften Plan mit ausdrücklicher
Zustimmung an:

```bash
openclaw claws update incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

OpenClaw erstellt den Plan neu und führt vor jeder Änderung einen Compare-and-Swap-Vorgang für den verwalteten Zustand
durch. Entfernte Paketdeklarationen geben Abhängigkeitsverknüpfungen frei, ohne
Artefakte zu deinstallieren. Bei Cron-Änderungen wird die aktuelle Zeitplanerdefinition erneut gelesen und
bei durch Bedienpersonal verursachten Abweichungen abgebrochen. Paketinstallationsprogramme, Schreiber der Quellkonfiguration und der Gateway-Zeitplaner
bilden keine Transaktion. Wenn nach einer externen
Änderung keine Kompensation nachgewiesen werden kann, meldet OpenClaw den Fehlercode `update_partial` mit strukturierten
`status: partial`, bewahrt unsichere Herkunftsinformationen
und hält an. Prüfen Sie `claws status`, die betroffene Ressource und `openclaw doctor`;
zeigen Sie anschließend erneut eine Vorschau an, bevor Sie den Vorgang wiederholen oder etwas entfernen.

## Eine installierte Claw entfernen

Zeigen Sie vor der Auswahl der Bereinigung eine Vorschau der Entfernung an:

```bash
openclaw claws remove incident-triage --dry-run --json
openclaw claws remove incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

Standardmäßig wird geeigneter verwalteter Zustand entfernt und referenzierter Zustand freigegeben.
Geänderte Dateien und Ressourcen mit einem anderen aktuellen Eigentümer werden beibehalten oder
blockiert. Bereinigungsoptionen sind Teil des Plan-Digests; `--yes` erweitert
sie niemals. Global installierte Plugins werden beibehalten, während die Referenz dieser Claw
freigegeben wird; verwenden Sie den gewöhnlichen Plugin-Lebenszyklus separat, wenn Sie
ein prozessweites Plugin deinstallieren möchten.

Um unveränderte, von der Claw eingeführte Referenzen zu entfernen, die keinen anderen aktuellen
Eigentümer haben, geben Sie `--remove-unused` sowohl bei der Vorschau als auch bei der Anwendung an. Um stattdessen
bestimmte referenzierte Ressourcen auszuwählen, wiederholen Sie `--remove-referenced`:

```bash
openclaw claws remove incident-triage \
  --dry-run \
  --remove-referenced 'plugin:@acme/audit-plugin@2.0.0'
```

Verwenden Sie `--force-referenced` erst, nachdem Sie die angezeigten abhängigen Ressourcen,
unabhängigen Eigentümer und den bereits bestehenden Ursprung geprüft haben. Die Option erlaubt die ausgewählte Bereinigung trotz
dieser Konflikte; sie überspringt nicht die Zustimmung zur Planintegrität.

## Einen installierten Agenten exportieren

Export erstellt ein neues Paketverzeichnis und schlägt fehl, wenn das Ziel bereits vorhanden ist oder
der verwaltete Zustand abweicht:

```bash
openclaw claws export incident-triage --out ./incident-triage-export --json
```

Das Ergebnis enthält `package.json`, kanonische `CLAW.md` und Sidecar-Dateien des verwalteten
Workspace. Es ist ein portables Claw-Paket und keine Sicherung der gesamten Instanz: Nicht zugehörige
Agenten, Anmeldedaten, Sitzungen und nicht verwalteter lokaler Zustand sind ausgeschlossen.

## Befehlsreferenz

| Befehl                              | Zweck                                                      |
| ----------------------------------- | ---------------------------------------------------------- |
| `claws inspect <source>`            | Validiert ein Paketverzeichnis oder ein gruppiertes Manifest. |
| `claws add <source>`                | Zeigt eine Vorschau an oder erstellt einen neuen Agenten und Workspace. |
| `claws status [claw-or-agent]`      | Meldet installierten Zustand, Eigentümerschaft und Abweichungen. |
| `claws update <claw-or-agent>`      | Zeigt eine Vorschau an oder wendet Änderungen aus der ausgewählten Quelle an. |
| `claws remove <claw-or-agent>`      | Zeigt eine Vorschau an oder entfernt den Agenten und geeignete Ressourcen. |
| `claws export <agent> --out <path>` | Erstellt ein portables Paket aus einem installierten Agenten. |

Verwenden Sie `--json` für experimentelle maschinenlesbare Ausgaben.

## Siehe auch

- [Agenten](/de/cli/agents)
- [Skills](/de/tools/skills)
- [Plugins](/de/tools/plugin)
- [Cron-Aufträge](/de/automation/cron-jobs)
- [MCP-Konfiguration](/de/gateway/configuration-reference#mcp)
