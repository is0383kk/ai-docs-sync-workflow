---
read_when:
    - Skills hinzufügen oder ändern
    - Skill-Zugriffsbeschränkungen, Zulassungslisten oder Laderegeln ändern
    - Grundlegendes zur Priorität von Skills und zum Snapshot-Verhalten
sidebarTitle: Skills
summary: Skills bringen Ihrem Agenten bei, wie er Tools verwendet. Erfahren Sie, wie sie geladen werden, wie die Prioritätsregeln funktionieren und wie Sie Zugriffsbeschränkungen, Positivlisten und die Umgebungsinjektion konfigurieren.
title: Skills
x-i18n:
    generated_at: "2026-07-26T18:15:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills sind Markdown-Anweisungsdateien, die dem Agenten vermitteln, wie und wann
Werkzeuge verwendet werden sollen. Jeder Skill befindet sich in einem Verzeichnis mit einer `SKILL.md`-Datei mit YAML-
Frontmatter und einem Markdown-Textkörper. OpenClaw lädt gebündelte Skills sowie lokale
Überschreibungen und filtert sie beim Laden anhand von Umgebung, Konfiguration und
vorhandenen Binärdateien.

<CardGroup cols={2}>
  <Card title="Skills erstellen" href="/de/tools/creating-skills" icon="hammer">
    Erstellen und testen Sie einen benutzerdefinierten Skill von Grund auf.
  </Card>
  <Card title="Skill Workshop" href="/de/tools/skill-workshop" icon="flask">
    Prüfen und genehmigen Sie vom Agenten entworfene Skill-Vorschläge.
  </Card>
  <Card title="Skills-Konfiguration" href="/de/tools/skills-config" icon="gear">
    Vollständiges `skills.*`-Konfigurationsschema und Agenten-Zulassungslisten.
  </Card>
  <Card title="ClawHub" href="/de/clawhub" icon="cloud">
    Durchsuchen und installieren Sie Community-Skills.
  </Card>
</CardGroup>

## Ladereihenfolge

OpenClaw lädt aus diesen Quellen, **beginnend mit der höchsten Priorität**. Wenn derselbe
Skill-Name an mehreren Stellen vorkommt, hat die Quelle mit der höchsten Priorität Vorrang.

| Priorität    | Quelle                       | Pfad                                    |
| ------------ | ---------------------------- | --------------------------------------- |
| 1 — höchste  | Workspace-Skills             | `<workspace>/skills`                    |
| 2            | Projekt-Agenten-Skills       | `<workspace>/.agents/skills`            |
| 3            | Persönliche Agenten-Skills   | `~/.agents/skills`                      |
| 4            | Verwaltete / lokale Skills   | `~/.openclaw/skills`                    |
| 5            | Gebündelte Skills            | mit der Installation ausgeliefert       |
| 6 — niedrigste | Zusätzliche Verzeichnisse  | `skills.load.extraDirs` + Plugin-Skills |

Skill-Stammverzeichnisse unterstützen gruppierte Layouts. OpenClaw erkennt einen Skill, sobald
`SKILL.md` irgendwo unter einem konfigurierten Stammverzeichnis erscheint (bis zu 6 Ebenen tief):

```text
<workspace>/skills/research/SKILL.md          ✓ als "research" gefunden
<workspace>/skills/personal/research/SKILL.md ✓ ebenfalls als "research" gefunden
```

Der Ordnerpfad dient ausschließlich der Organisation. Der Name und der Slash-Befehl
des Skills stammen aus dem Frontmatter-Feld `name` (oder aus dem Verzeichnisnamen, wenn `name`
fehlt). Agenten-Zulassungslisten (siehe unten) gleichen ebenfalls dieses `name` ab.

<Note>
  Das native `$CODEX_HOME/skills`-Verzeichnis der Codex CLI ist **kein**
  Skill-Stammverzeichnis von OpenClaw. Verwenden Sie `openclaw migrate plan codex`, um diese Skills zu erfassen, und anschließend
  `openclaw migrate codex`, um sie in Ihren OpenClaw-Workspace zu kopieren.
</Note>

## Auf Nodes gehostete Skills

Ein verbundener Headless-Node kann Skills veröffentlichen, die in seinem aktiven OpenClaw-
Skills-Verzeichnis installiert sind (standardmäßig `~/.openclaw/skills`; Überschreibungen durch die Profilumgebung
gelten). Sie erscheinen in der normalen Skill-Liste des Agenten, solange der Node verbunden ist,
und verschwinden, wenn seine Verbindung getrennt wird. Bei einer Namenskollision behält ein lokaler oder Gateway-Skill
seinen Namen; der Node-Skill erhält einen deterministischen Namen mit Node-Präfix.
Bei auf Nodes gehosteten Skills der Version v1 muss der Verzeichnisname mit dem Frontmatter-Feld
`name` des Skills übereinstimmen.

Der Skill-Eintrag enthält den Node-Locator. Seine Dateien, relativen Verweise und
Binärdateien befinden sich auf dem Node; laden und führen Sie ihn daher mit
`exec host=node node=<node-id>` aus. Starten Sie den Node-Host nach Änderungen an seinen Skill-
Dateien neu. Informationen zur Kopplung und zu Deaktivierungsoptionen finden Sie unter [Nodes](/de/nodes#node-hosted-skills).

## Agentenspezifische und gemeinsam genutzte Skills

In Multi-Agenten-Konfigurationen verfügt jeder Agent über einen eigenen Workspace. Verwenden Sie den Pfad, der
der gewünschten Sichtbarkeit entspricht:

| Geltungsbereich        | Pfad                         | Sichtbar für                         |
| ---------------------- | ---------------------------- | ------------------------------------ |
| Agentenspezifisch      | `<workspace>/skills`         | Nur diesen Agenten                   |
| Projekt-Agent          | `<workspace>/.agents/skills` | Nur den Agenten dieses Workspace     |
| Persönlicher Agent     | `~/.agents/skills`           | Alle Agenten auf diesem Rechner      |
| Gemeinsam verwaltet    | `~/.openclaw/skills`         | Alle Agenten auf diesem Rechner      |
| Zusätzliche Verzeichnisse | `skills.load.extraDirs`      | Alle Agenten auf diesem Rechner      |

## Agenten-Zulassungslisten

Der **Speicherort** eines Skills (Priorität) und seine **Sichtbarkeit** (welcher Agent ihn verwenden kann)
sind getrennte Steuerungsmöglichkeiten. Verwenden Sie Zulassungslisten, um einzuschränken, welche Skills ein Agent sieht,
unabhängig davon, woher sie geladen werden.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // gemeinsame Ausgangsbasis
    },
    list: [
      { id: "writer" }, // übernimmt github, weather
      { id: "docs", skills: ["docs-search"] }, // ersetzt die Standardwerte vollständig
      { id: "locked-down", skills: [] }, // keine Skills
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="Regeln für Zulassungslisten">
    - Lassen Sie `agents.defaults.skills` weg, damit standardmäßig alle Skills uneingeschränkt verfügbar bleiben.
    - Lassen Sie `agents.entries.*.skills` weg, um `agents.defaults.skills` zu übernehmen.
    - Setzen Sie `agents.entries.*.skills: []`, damit für diesen Agenten keine Skills verfügbar sind.
    - Eine nicht leere `agents.entries.*.skills`-Liste ist die **endgültige** Menge — sie wird nicht
      mit den Standardwerten zusammengeführt.
    - Die wirksame Zulassungsliste gilt für die Prompt-Erstellung, die Erkennung von Slash-Befehlen,
      die Sandbox-Synchronisierung und Skill-Snapshots.
    - Dies ist keine Autorisierungsgrenze für die Host-Shell. Wenn derselbe Agent
      `exec` verwenden kann, schränken Sie diese Shell separat durch Sandboxing, Betriebssystembenutzer-
      Isolierung, Ausführungs-Sperr-/Zulassungslisten und ressourcenspezifische Anmeldedaten ein.
  </Accordion>
</AccordionGroup>

## Plugins und Skills

Plugins können eigene Skills ausliefern, indem sie `skills`-Verzeichnisse in
`openclaw.plugin.json` auflisten (Pfade relativ zum Plugin-Stammverzeichnis). Plugin-Skills werden geladen,
wenn das Plugin aktiviert ist — beispielsweise liefert das Browser-Plugin einen
`browser-automation`-Skill für die mehrstufige Browsersteuerung aus.

Plugin-Skill-Verzeichnisse werden auf derselben niedrigen Prioritätsstufe wie
`skills.load.extraDirs` zusammengeführt. Daher überschreibt ein gleichnamiger gebündelter, verwalteter, Agenten- oder Workspace-
Skill diese Verzeichnisse. Steuern Sie die Eignung eines Plugin-Skills über
`metadata.openclaw.requires` in seinem Frontmatter, genau wie bei jedem anderen Skill.

Das vollständige Plugin-System ist unter [Plugins](/de/tools/plugin) und [Werkzeuge](/de/tools) beschrieben.

## Skill Workshop

Der [Skill Workshop](/de/tools/skill-workshop) ist eine Vorschlagswarteschlange zwischen dem Agenten
und Ihren aktiven Skill-Dateien. Wenn der Agent wiederverwendbare Arbeit erkennt, erstellt er einen
Vorschlag, statt direkt in `SKILL.md` zu schreiben. Sie prüfen und genehmigen ihn,
bevor Änderungen vorgenommen werden.

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Unter [Skill Workshop](/de/tools/skill-workshop) finden Sie den vollständigen Lebenszyklus, die CLI-
Referenz und die Konfiguration.

## Installation aus ClawHub

[ClawHub](https://clawhub.ai) ist das öffentliche Skills-Register. Verwenden Sie
`openclaw skills`-Befehle für Installation und Aktualisierung oder die `clawhub`-CLI zum
Veröffentlichen und Synchronisieren.

| Aktion                                  | Befehl                                                 |
| --------------------------------------- | ------------------------------------------------------ |
| Einen Skill im Workspace installieren   | `openclaw skills install @owner/<slug>`                |
| Aus einem Git-Repository installieren   | `openclaw skills install git:owner/repo@ref`           |
| Ein lokales Skill-Verzeichnis installieren | `openclaw skills install ./path/to/skill --as my-tool` |
| Für alle lokalen Agenten installieren   | `openclaw skills install @owner/<slug> --global`       |
| Alle Workspace-Skills aktualisieren     | `openclaw skills update --all`                         |
| Einen gemeinsam verwalteten Skill aktualisieren | `openclaw skills update @owner/<slug> --global`        |
| Alle gemeinsam verwalteten Skills aktualisieren | `openclaw skills update --all --global`                |
| Vertrauensrahmen eines Skills überprüfen | `openclaw skills verify @owner/<slug>`                 |
| Die generierte Skill-Karte ausgeben     | `openclaw skills verify @owner/<slug> --card`          |
| Über die ClawHub-CLI veröffentlichen / synchronisieren | `clawhub sync --all`                                   |

<AccordionGroup>
  <Accordion title="Installationsdetails">
    `openclaw skills install` installiert standardmäßig in das Verzeichnis `skills/`
    des aktiven Workspace. Fügen Sie `--global` hinzu, um in das gemeinsam genutzte
    Verzeichnis `~/.openclaw/skills` zu installieren, das für alle lokalen Agenten sichtbar ist, sofern Agenten-
    Zulassungslisten dies nicht einschränken.

    Git- und lokale Installationen erwarten `SKILL.md` im Stammverzeichnis der Quelle. Der Slug stammt,
    sofern gültig, aus `name` im Frontmatter `SKILL.md`; andernfalls wird der
    Verzeichnis- oder Repository-Name verwendet. Verwenden Sie `--as <slug>`, um ihn zu überschreiben.
    `openclaw skills update` verfolgt ausschließlich ClawHub-Installationen — installieren Sie Git- oder
    lokale Quellen erneut, um sie zu aktualisieren.

  </Accordion>
  <Accordion title="Verifizierung und Sicherheitsscans">
    `openclaw skills verify @owner/<slug>` fragt bei ClawHub den
    `clawhub.skill.verify.v1`-Vertrauensrahmen des Skills ab. Installierte ClawHub-Skills werden
    anhand der in `.clawhub/origin.json` erfassten Version und Registry verifiziert.
    Reine Slugs werden für bereits installierte oder eindeutige Skills weiterhin akzeptiert, aber
    um den Eigentümer ergänzte Referenzen vermeiden Mehrdeutigkeiten beim Herausgeber.

    ClawHub-Skill-Seiten zeigen vor der Installation den aktuellen Status des Sicherheitsscans
    sowie Detailseiten für VirusTotal, ClawScan und statische Analyse. Der
    Befehl wird mit einem von null verschiedenen Status beendet, wenn ClawHub die Verifizierung als fehlgeschlagen kennzeichnet. Herausgeber
    können Fehlalarme über das ClawHub-Dashboard oder
    `clawhub skill rescan @owner/<slug>` beheben.

  </Accordion>
  <Accordion title="Installation privater Archive">
    Gateway-Clients, die eine Bereitstellung außerhalb von ClawHub benötigen, können ein ZIP-Skill-Archiv
    mit `skills.upload.begin`, `skills.upload.chunk` und `skills.upload.commit` bereitstellen
    und es anschließend mit `skills.install({ source: "upload", ... })` installieren. Dieser Pfad ist
    standardmäßig deaktiviert und erfordert `skills.install.allowUploadedArchives: true` in
    `openclaw.json`. Normale ClawHub-Installationen benötigen diese Einstellung nie.
  </Accordion>
</AccordionGroup>

## Sicherheit

<Warning>
  Behandeln Sie Skills von Drittanbietern als **nicht vertrauenswürdigen Code**. Lesen Sie sie vor der Aktivierung.
  Bevorzugen Sie Sandbox-Ausführungen für nicht vertrauenswürdige Eingaben und riskante Werkzeuge. Unter
  [Sandboxing](/de/gateway/sandboxing) finden Sie agentenseitige Steuerungsmöglichkeiten.
</Warning>

<AccordionGroup>
  <Accordion title="Pfadbegrenzung">
    Bei der Skill-Erkennung für Workspace-, Projekt-Agenten- und zusätzliche Verzeichnisse werden nur Skill-
    Stammverzeichnisse akzeptiert, deren aufgelöster Realpath innerhalb des konfigurierten Stammverzeichnisses bleibt, sofern
    `skills.load.allowSymlinkTargets` einem Zielstammverzeichnis nicht ausdrücklich vertraut.
    Skill Workshop schreibt nur über diese vertrauenswürdigen Ziele, wenn
    `skills.workshop.allowSymlinkTargetWrites` aktiviert ist.
    Das verwaltete `~/.openclaw/skills` und das persönliche `~/.agents/skills` dürfen
    über symbolische Links eingebundene Skill-Ordner enthalten, aber jeder Realpath von `SKILL.md` muss weiterhin
    innerhalb seines aufgelösten Skill-Verzeichnisses bleiben.
  </Accordion>
  <Accordion title="Installationsrichtlinie des Betreibers">
    Konfigurieren Sie `security.installPolicy`, um einen vertrauenswürdigen lokalen Richtlinienbefehl
    auszuführen, bevor Skill-Installationen fortgesetzt werden. Die Richtlinie erhält Metadaten und den bereitgestellten
    Quellpfad, gilt für ClawHub-, Upload-, Git-, lokale, Aktualisierungs- und
    Abhängigkeitsinstallationspfade und verweigert den Vorgang, wenn der Befehl keine
    gültige Entscheidung zurückgeben kann.
  </Accordion>
  <Accordion title="Geltungsbereich der Secret-Injektion">
    `skills.entries.*.env` und `skills.entries.*.apiKey` injizieren Secrets ausschließlich für diesen Agentendurchlauf in den
    **Host**-Prozess — nicht in die Sandbox. Halten Sie
    Secrets aus Prompts und Protokollen heraus.
  </Accordion>
</AccordionGroup>

Das umfassendere Bedrohungsmodell und Sicherheitschecklisten finden Sie unter
[Sicherheit](/de/gateway/security).

## SKILL.md-Format

Jeder Skill benötigt im Frontmatter mindestens ein `name` und ein `description`:

```markdown
---
name: image-lab
description: Bilder über einen Provider-gestützten Bild-Workflow generieren oder bearbeiten
---

Wenn der Benutzer die Generierung eines Bildes anfordert, verwenden Sie das Werkzeug `image_generate`...
```

<Note>
  OpenClaw folgt der [AgentSkills](https://agentskills.io)-Spezifikation. Frontmatter
  wird zunächst als YAML geparst; wenn dies fehlschlägt, wird auf einen Parser zurückgegriffen, der ausschließlich
  einzelne Zeilen unterstützt. Verschachtelte `metadata`-Blöcke (einschließlich mehrzeiliger YAML-Zuordnungen) werden
  zu einer JSON-Zeichenfolge verflacht und erneut als JSON5 geparst, sodass die unter
  [Zugriffssteuerung](#gating) gezeigte Blockform funktioniert. Verwenden Sie `{baseDir}` im Textkörper, um auf den Pfad des
  Skill-Ordners zu verweisen.
</Note>

### Optionale Frontmatter-Schlüssel

<ParamField path="homepage" type="string">
  URL, die in der macOS-Skills-Benutzeroberfläche als "Website" angezeigt wird. Wird auch über
  `metadata.openclaw.homepage` unterstützt.
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  Wenn `true`, wird das Skill als vom Benutzer aufrufbarer Slash-Befehl bereitgestellt.
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  Wenn `true`, nimmt OpenClaw die Anweisungen des Skills nicht in den normalen
  Prompt des Agenten auf. Das Skill ist weiterhin als Slash-Befehl verfügbar, wenn `user-invocable`
  ebenfalls `true` ist.
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  Wenn auf `tool` gesetzt, umgeht der Slash-Befehl das Modell und leitet
  den Aufruf direkt an ein registriertes Tool weiter.
</ParamField>

<ParamField path="command-tool" type="string">
  Name des aufzurufenden Tools, wenn `command-dispatch: tool` gesetzt ist.
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  Bei der Weiterleitung an ein Tool wird die unverarbeitete Argumentzeichenfolge ohne
  Parsing durch den Kern an das Tool weitergegeben. Das Tool empfängt
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.
</ParamField>

## Zugriffsbeschränkung

OpenClaw filtert Skills beim Laden mithilfe von `metadata.openclaw` (einem in
das Frontmatter eingebetteten JSON5-Objekt; siehe den Parsing-Hinweis oben). Ein Skill ohne
`metadata.openclaw`-Block ist immer zulässig, sofern es nicht ausdrücklich deaktiviert wurde.

```markdown
---
name: image-lab
description: Bilder über einen Provider-gestützten Bild-Workflow generieren oder bearbeiten
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  Wenn `true`, wird das Skill immer einbezogen und alle anderen Zugriffsschranken werden übersprungen.
</ParamField>

<ParamField path="emoji" type="string">
  Optionales Emoji, das in der macOS-Oberfläche für Skills angezeigt wird.
</ParamField>

<ParamField path="homepage" type="string">
  Optionale URL, die in der macOS-Oberfläche für Skills als „Website“ angezeigt wird.
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  Plattformfilter. Wenn festgelegt, ist das Skill nur auf einem aufgeführten Betriebssystem zulässig.
</ParamField>

<ParamField path="requires.bins" type="string[]">
  Jede Binärdatei muss in `PATH` vorhanden sein.
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  Mindestens eine Binärdatei muss in `PATH` vorhanden sein.
</ParamField>

<ParamField path="requires.env" type="string[]">
  Jede Umgebungsvariable muss im Prozess vorhanden oder über die Konfiguration bereitgestellt sein.
</ParamField>

<ParamField path="requires.config" type="string[]">
  Jeder `openclaw.json`-Pfad muss einen Wahrheitswert ergeben.
</ParamField>

<ParamField path="primaryEnv" type="string">
  Name der mit `skills.entries.<name>.apiKey` verknüpften Umgebungsvariable.
</ParamField>

<ParamField path="install" type="object[]">
  Optionale Installationsspezifikationen für die macOS-Oberfläche für Skills (brew / node / go / uv / download).
</ParamField>

<Note>
  Veraltete `metadata.clawdbot`-Blöcke werden weiterhin akzeptiert, wenn
  `metadata.openclaw` fehlt, sodass ältere installierte Skills ihre
  Abhängigkeitsschranken und Installationshinweise beibehalten. Neue Skills sollten
  `metadata.openclaw` verwenden.
</Note>

### Installationsspezifikationen

Installationsspezifikationen teilen der macOS-Oberfläche für Skills mit, wie eine Abhängigkeit installiert wird:

```markdown
---
name: gemini
description: Gemini CLI für Programmierunterstützung und Google-Suchabfragen verwenden.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Gemini CLI installieren (brew)",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="Regeln zur Auswahl des Installationsprogramms">
    - Wenn mehrere Installationsprogramme aufgeführt sind, wählt das Gateway eine bevorzugte
      Option aus (brew, falls verfügbar, andernfalls node).
    - Wenn alle Installationsprogramme `download` sind, führt OpenClaw jeden Eintrag auf, damit Sie
      alle verfügbaren Artefakte sehen können.
    - Spezifikationen können `os: ["darwin"|"linux"|"win32"]` enthalten, um nach Plattform zu filtern.
    - Node-Installationen berücksichtigen `skills.install.nodeManager` in `openclaw.json`
      (Standard: npm; Optionen: npm / pnpm / yarn / bun). Dies wirkt sich nur auf Skill-
      Installationen aus; die Gateway-Laufzeit sollte weiterhin Node sein.
    - Installationspräferenz des Gateways: Homebrew → uv → konfigurierter Node-Manager →
      go → Download.
  </Accordion>
  <Accordion title="Details zu den einzelnen Installationsprogrammen">
    - **Homebrew:** OpenClaw installiert Homebrew nicht automatisch und übersetzt brew-
      Formeln nicht in Systempaketbefehle. In Linux-Containern ohne
      `brew` werden Installationsprogramme, die ausschließlich brew unterstützen, ausgeblendet; verwenden Sie ein benutzerdefiniertes Image oder installieren Sie
      die Abhängigkeit manuell.
    - **Go:** OpenClaw benötigt Go 1.21 oder neuer für automatische Skill-Installationen.
      Wenn `go` fehlt und Homebrew verfügbar ist, installiert OpenClaw zunächst Go über
      Homebrew; unter Linux ohne Homebrew kann stattdessen `apt-get`
      als Root oder über passwortloses `sudo` verwendet werden, wenn der aktualisierte `golang-go`-
      Kandidat die Mindestversion erfüllt. Das tatsächliche `go install` für die
      Abhängigkeit zielt immer auf ein dediziertes, von OpenClaw verwaltetes Binärdateiverzeichnis
      (bei einer Neuinstallation `bin` von Homebrew, andernfalls `~/.local/bin`) statt auf
      Ihr konfiguriertes `GOBIN` — Ihre eigenen Umgebungsvariablen `GOBIN`, `GOPATH` und `GOTOOLCHAIN`
      werden gelesen, aber niemals überschrieben.
    - **Download:** `url` (erforderlich), `archive` (`tar.gz` | `tar.bz2` | `zip`),
      `extract` (Standard: automatisch, wenn ein Archiv erkannt wird), `stripComponents`,
      `targetDir` (Standard: `~/.openclaw/tools/<skillKey>`).
  </Accordion>
  <Accordion title="Hinweise zur Sandbox">
    `requires.bins` wird beim Laden des Skills auf dem **Host** geprüft. Wenn ein Agent
    in einer Sandbox ausgeführt wird, muss die Binärdatei auch **innerhalb des Containers** vorhanden sein.
    Installieren Sie sie über `agents.defaults.sandbox.docker.setupCommand` oder ein benutzerdefiniertes
    Image. `setupCommand` wird einmal nach der Containererstellung ausgeführt und erfordert
    ausgehenden Netzwerkzugriff, ein beschreibbares Root-Dateisystem und einen Root-Benutzer in der Sandbox.
  </Accordion>
</AccordionGroup>

## Konfigurationsüberschreibungen

Aktivieren, deaktivieren und konfigurieren Sie gebündelte oder verwaltete Skills unter `skills.entries` in
`~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false` deaktiviert das Skill, selbst wenn es gebündelt oder installiert ist. Das gebündelte Skill
  `coding-agent` muss explizit aktiviert werden — setzen Sie `skills.entries.coding-agent.enabled: true`
  und stellen Sie sicher, dass `claude`, `codex`, `opencode` oder eine andere unterstützte CLI
  installiert und authentifiziert ist.
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  Komfortfeld für Skills, die `metadata.openclaw.primaryEnv` deklarieren.
  Unterstützt eine Klartextzeichenfolge oder ein SecretRef-Objekt.
</ParamField>

<ParamField path="env" type="Record<string, string>">
  Umgebungsvariablen, die für die Agentenausführung injiziert werden. Sie werden nur injiziert, wenn die
  Variable nicht bereits im Prozess gesetzt ist.
</ParamField>

<ParamField path="config" type="object">
  Optionales Objekt für benutzerdefinierte Konfigurationsfelder pro Skill.
</ParamField>

<ParamField path="allowBundled" type="string[]">
  Optionale Positivliste ausschließlich für **gebündelte** Skills. Wenn festgelegt, sind nur gebündelte Skills
  aus der Liste zulässig. Verwaltete Skills und Workspace-Skills sind davon nicht betroffen.
</ParamField>

<Note>
  Konfigurationsschlüssel entsprechen standardmäßig dem **Skill-Namen**. Wenn ein Skill
  `metadata.openclaw.skillKey` definiert, verwenden Sie stattdessen diesen Schlüssel unter `skills.entries`.
  Setzen Sie Namen mit Bindestrichen in Anführungszeichen: JSON5 erlaubt Schlüssel in Anführungszeichen.
</Note>

## Injektion von Umgebungsvariablen

Wenn eine Agentenausführung beginnt, führt OpenClaw folgende Schritte aus:

<Steps>
  <Step title="Skill-Metadaten lesen">
    OpenClaw ermittelt die effektive Skill-Liste für den Agenten und wendet dabei Zugriffsschranken,
    Positivlisten und Konfigurationsüberschreibungen an.
  </Step>
  <Step title="Umgebungsvariablen und API-Schlüssel injizieren">
    `skills.entries.<key>.env` und `skills.entries.<key>.apiKey` werden für die Dauer
    der Ausführung auf `process.env` angewendet.
  </Step>
  <Step title="System-Prompt erstellen">
    Zulässige Skills werden in einem kompakten XML-Block zusammengefasst und in den
    System-Prompt injiziert.
  </Step>
  <Step title="Umgebung wiederherstellen">
    Nach dem Ende der Ausführung wird die ursprüngliche Umgebung wiederhergestellt.
  </Step>
</Steps>

<Warning>
  Die Injektion von Umgebungsvariablen ist auf die Agentenausführung auf dem **Host** beschränkt, nicht auf die Sandbox. Innerhalb einer
  Sandbox haben `env` und `apiKey` keine Wirkung. Unter
  [Skills-Konfiguration](/de/tools/skills-config#sandboxed-skills-and-env-vars) erfahren Sie, wie
  Secrets an Ausführungen in einer Sandbox übergeben werden.
</Warning>

Für das gebündelte `claude-cli`-Backend materialisiert OpenClaw denselben
Snapshot der zulässigen Skills außerdem als temporäres Claude-Code-Plugin und übergibt ihn über
`--plugin-dir`. Andere CLI-Backends verwenden ausschließlich den Prompt-Katalog.

## Snapshots und Aktualisierung

OpenClaw erstellt einen Snapshot der zulässigen Skills, **wenn eine Sitzung beginnt**, und verwendet diese
Liste für alle nachfolgenden Interaktionen in der Sitzung erneut. Änderungen an Skills oder der Konfiguration werden
in der nächsten neuen Sitzung wirksam.

Skills werden während einer Sitzung in zwei Fällen aktualisiert:

- Der Skills-Watcher erkennt eine Änderung an `SKILL.md`.
- Ein neuer zulässiger Remote-Node stellt eine Verbindung her.

Die aktualisierte Liste wird bei der nächsten Agenteninteraktion übernommen. Wenn sich die effektive
Positivliste des Agenten ändert, aktualisiert OpenClaw den Snapshot, damit die sichtbaren Skills
übereinstimmen.

<AccordionGroup>
  <Accordion title="Skills-Watcher">
    Standardmäßig überwacht OpenClaw Skill-Ordner und aktualisiert den Snapshot, wenn sich
    `SKILL.md`-Dateien ändern. Konfigurieren Sie dies unter `skills.load`:

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // Standard
        },
      },
    }
    ```

    Watcher-Ereignisse verwenden eine integrierte Entprellzeit von 250 ms. Verwenden Sie `allowSymlinkTargets`
    für beabsichtigte Layouts mit symbolischen Links, bei denen ein symbolischer Link des Skill-
    Stammverzeichnisses außerhalb des konfigurierten Stammverzeichnisses liegt, beispielsweise
    `<workspace>/skills/manager -> ~/Projects/manager/skills`.
    Aktivieren Sie `skills.workshop.allowSymlinkTargetWrites` nur, wenn Skill Workshop
    Vorschläge ebenfalls über diese vertrauenswürdigen Pfade mit symbolischen Links anwenden soll.

  </Accordion>
  <Accordion title="Remote-macOS-Nodes (Linux-Gateway)">
    Wenn das Gateway unter Linux ausgeführt wird, aber ein **macOS-Node** mit zugelassenem
    `system.run` verbunden ist, kann OpenClaw ausschließlich für macOS verfügbare Skills als zulässig betrachten, wenn
    die erforderlichen Binärdateien auf diesem Node vorhanden sind. Der Agent sollte diese
    Skills über das Tool `exec` mit `host=node` ausführen.

    Offline-Nodes machen ausschließlich remote verfügbare Skills **nicht** sichtbar. Wenn ein Node nicht mehr
    auf Binärdateiabfragen antwortet, löscht OpenClaw die zwischengespeicherten Binärdateiübereinstimmungen dieses Nodes.

  </Accordion>
</AccordionGroup>

## Token-Auswirkungen

Wenn Skills zulässig sind, injiziert OpenClaw einen kompakten XML-Block in den System-
Prompt. Die Kosten sind deterministisch und skalieren linear pro Skill:

- **Grundaufwand** (nur wenn mindestens 1 Skill zulässig ist): ein fester Block aus einleitendem
  Text sowie dem `<available_skills>`-Wrapper.
- **Pro Skill:** ~97 Zeichen plus die Längen Ihrer Felder `name`, `description` und `location`.
- XML-Escaping erweitert `& < > " '` zu Entitäten und fügt pro
  Vorkommen einige Zeichen hinzu.
- Bei ~4 Zeichen/Token entsprechen 97 Zeichen vor Berücksichtigung der Feldlängen ungefähr 24 Token pro Skill.

Wenn der gerenderte Block das konfigurierte Prompt-Budget
(`skills.limits.maxSkillsPromptChars`) überschreiten würde, behält OpenClaw zunächst so viele
Skill-Identitäten (Name, Speicherort und Version) bei, wie in das beschreibungsfreie
Kompaktformat passen. Anschließend wird das verbleibende Budget für gekürzte
Beschreibungen verwendet. Wenn kein Budget für Beschreibungen verbleibt, werden
sie weggelassen. Der Prompt enthält einen Hinweis auf `openclaw skills check`, wenn
eine kompakte Formatierung oder das Kürzen der Liste erforderlich ist.

Halten Sie Beschreibungen kurz und aussagekräftig, um den Prompt-Overhead zu minimieren.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Skills erstellen" href="/de/tools/creating-skills" icon="hammer">
    Schritt-für-Schritt-Anleitung zum Erstellen eines benutzerdefinierten Skills.
  </Card>
  <Card title="Skill-Workshop" href="/de/tools/skill-workshop" icon="flask">
    Vorschlagswarteschlange für von Agenten entworfene Skills.
  </Card>
  <Card title="Skills-Konfiguration" href="/de/tools/skills-config" icon="gear">
    Vollständiges `skills.*`-Konfigurationsschema und Agenten-Zulassungslisten.
  </Card>
  <Card title="Slash-Befehle" href="/de/tools/slash-commands" icon="terminal">
    Wie Slash-Befehle von Skills registriert und weitergeleitet werden.
  </Card>
  <Card title="ClawHub" href="/de/clawhub" icon="cloud">
    Durchsuchen und veröffentlichen Sie Skills im öffentlichen Register.
  </Card>
  <Card title="Plugins" href="/de/tools/plugin" icon="plug">
    Plugins können Skills zusammen mit den Tools bereitstellen, die sie dokumentieren.
  </Card>
</CardGroup>
