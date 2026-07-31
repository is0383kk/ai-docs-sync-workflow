---
read_when:
    - Lade-, Installations- oder Zugriffsbeschränkungsverhalten für Skills konfigurieren
    - Sichtbarkeit von Skills pro Agent festlegen
    - Limits oder Genehmigungsrichtlinie des Skill Workshop anpassen
sidebarTitle: Skills config
summary: Vollständige Referenz für das Konfigurationsschema `skills.*`, Agenten-Zulassungslisten, Workshop-Einstellungen und die Handhabung von Sandbox-Umgebungsvariablen.
title: Skills-Konfiguration
x-i18n:
    generated_at: "2026-07-26T18:10:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

Die meiste Skills-Konfiguration befindet sich unter `skills` in
`~/.openclaw/openclaw.json`. Die agentenspezifische Sichtbarkeit befindet sich unter
`agents.defaults.skills` und `agents.entries.*.skills`.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
      watch: true,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",
      allowUploadedArchives: false,
    },
    workshop: {
      autonomous: { enabled: false },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<Note>
  Verwenden Sie für die integrierte Bilderzeugung `agents.defaults.mediaModels.image`
  zusammen mit dem zentralen Tool `image_generate` anstelle von `skills.entries`. Skill-Einträge
  sind ausschließlich für benutzerdefinierte oder Drittanbieter-Skill-Workflows vorgesehen.
</Note>

## Laden (`skills.load`)

<ParamField path="skills.load.extraDirs" type="string[]">
  Zusätzliche zu durchsuchende Skill-Verzeichnisse mit der niedrigsten Priorität (unterhalb
  gebündelter Skills und Plugin-Skills). Pfade werden mit Unterstützung für `~` erweitert.
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  Vertrauenswürdige tatsächliche Zielverzeichnisse, in die über Symlinks eingebundene Skill-Ordner
  aufgelöst werden dürfen, selbst wenn sich der Symlink außerhalb des konfigurierten Stammverzeichnisses
  befindet. Verwenden Sie dies für beabsichtigte Layouts mit benachbarten Repositorys wie
  `<workspace>/skills/manager -> ~/Projects/manager/skills`. Halten Sie diese Liste
  eng begrenzt — verweisen Sie nicht auf weit gefasste Stammverzeichnisse wie `~` oder `~/Projects`.
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  Überwacht Skill-Ordner und aktualisiert den Skills-Snapshot, wenn sich `SKILL.md`-Dateien
  ändern. Dies umfasst verschachtelte Dateien unter gruppierten Skill-Stammverzeichnissen.
</ParamField>

## Installation (`skills.install`)

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  Bevorzugt Homebrew-Installationsprogramme, wenn `brew` verfügbar ist.
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  Bevorzugter Node-Paketmanager für Skill-Installationen. Dies betrifft ausschließlich
  Skill-Installationen – die OpenClaw-CLI und die Gateway-Laufzeit benötigen Node, da der
  kanonische Zustandsspeicher `node:sqlite` verwendet. `openclaw setup --node-manager` und
  `openclaw onboard --node-manager` akzeptieren `npm`, `pnpm` oder `bun`; legen Sie
  `"yarn"` für Yarn-basierte Skill-Installationen direkt in der Konfiguration fest.
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  Erlaubt vertrauenswürdigen `operator.admin`-Gateway-Clients, private ZIP-Archive zu installieren,
  die über `skills.upload.*` bereitgestellt wurden. Normale ClawHub-Installationen benötigen
  diese Einstellung nicht.
</ParamField>

## Installationsrichtlinie für Betreiber (`security.installPolicy`)

Verwenden Sie `security.installPolicy`, wenn Betreiber einen vertrauenswürdigen lokalen Befehl benötigen,
um Skill- und Plugin-Installationen anhand hostspezifischer Richtlinien zu genehmigen oder zu blockieren. Die
Richtlinie wird ausgeführt, nachdem OpenClaw das Quellmaterial bereitgestellt hat und bevor die Installation
oder Aktualisierung fortgesetzt wird. Sie gilt für ClawHub-Skills, hochgeladene Skills, Git-/lokale
Skills, Installationsprogramme für Skill-Abhängigkeiten sowie Quellen für Plugin-Installationen und -Aktualisierungen.

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // Omit targets to cover every supported target.
      targets: ["skill", "plugin"],
      exec: {
        source: "exec",
        command: "/usr/local/bin/openclaw-install-policy",
        args: ["--json"],
        timeoutMs: 10000,
        noOutputTimeoutMs: 10000,
        maxOutputBytes: 1048576,
        passEnv: ["OPENCLAW_STATE_DIR", "PATH"],
        env: { POLICY_MODE: "strict" },
        trustedDirs: ["/usr/local/bin"],
      },
    },
  },
}
```

<ParamField path="security.installPolicy.enabled" type="boolean" default="false">
  Aktiviert die betreibereigene Installationsrichtlinie. Wenn sie ohne gültigen `exec`-Befehl
  aktiviert wird, werden Installationen standardmäßig blockiert.
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  Optionaler Zielfilter. Wenn er weggelassen wird, gilt die Richtlinie für jedes unterstützte
  Ziel, damit neue Installationen nicht unerwartet standardmäßig zugelassen werden.
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  Absoluter Pfad zur vertrauenswürdigen ausführbaren Richtliniendatei. OpenClaw führt sie ohne
  Shell aus und validiert den Pfad vor der Verwendung.
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  Statische Argumente, die nach `command` übergeben werden.
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  Maximale Gesamtlaufzeit für eine Richtlinienentscheidung.
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  Maximale Zeit ohne Ausgabe auf stdout oder stderr, bevor die Richtlinie
  standardmäßig blockiert.
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  Maximale kombinierte Anzahl an stdout- und stderr-Bytes, die vom Richtlinienprozess akzeptiert wird.
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  Literale Umgebungsvariablen, die dem Richtlinienprozess bereitgestellt werden.
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  Namen von Umgebungsvariablen, die aus dem OpenClaw-Prozess in den
  Richtlinienprozess kopiert werden. Nur benannte Variablen werden übergeben.
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  Optionale Positivliste von Verzeichnissen, die die ausführbare Richtliniendatei enthalten dürfen.
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  Umgeht Prüfungen der Eigentümerschaft und Berechtigungen des Befehlspfads. Verwenden Sie dies nur, wenn der
  Pfad durch einen anderen Mechanismus geschützt ist.
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  Erlaubt, dass der konfigurierte Befehlspfad ein Symlink ist. Das aufgelöste Ziel
  muss weiterhin die übrigen Pfadprüfungen erfüllen. Argumente für Interpreter-Skripte müssen
  direkte reguläre Dateien und dürfen keine Symlinks sein.
</ParamField>

Die Richtlinie empfängt über stdin ein JSON-Objekt mit `protocolVersion: 1`,
`openclawVersion`, `targetType`, `targetName`, `sourcePath`, `sourcePathKind`,
optionalem strukturiertem `source`, strukturiertem `origin` und `request`. Sie muss
über stdout ein JSON-Objekt ausgeben: `{ "protocolVersion": 1, "decision": "allow" }`
oder `{ "protocolVersion": 1, "decision": "block", "reason": "..." }`. Ein von null verschiedener
Exit-Code, eine Zeitüberschreitung, fehlerhaftes JSON, fehlende Felder oder nicht unterstützte Protokollversionen
führen zur standardmäßigen Blockierung.

OpenClaw führt die Installationsrichtlinie beim normalen Start des Gateways nicht aus.
Installationen und Aktualisierungen werden standardmäßig blockiert, wenn die Richtlinie aktiviert, aber nicht verfügbar ist.
`openclaw doctor` führt eine statische Validierung durch; `openclaw doctor --deep`
führt eine synthetische Installationsprüfung für den konfigurierten Befehl aus.

Bei Massenaktualisierungen wird die Richtlinie auf jedes Ziel einzeln angewendet: Eine blockierte Aktualisierung
eines Skills oder Plugins lässt dieses Ziel fehlschlagen, ohne die Richtlinie zu deaktivieren oder spätere Ziele
im Stapel zu überspringen.

Beispiel für stdin:

```json
{
  "protocolVersion": 1,
  "openclawVersion": "2026.6.1",
  "targetType": "skill",
  "targetName": "weather",
  "sourcePath": "/var/folders/.../openclaw-skill-clawhub/root",
  "sourcePathKind": "directory",
  "source": {
    "kind": "clawhub",
    "authority": "openclaw",
    "mutable": false,
    "network": true
  },
  "origin": {
    "type": "clawhub",
    "registry": "https://clawhub.openclaw.ai",
    "slug": "weather",
    "version": "1.0.0"
  },
  "request": {
    "kind": "skill-install",
    "mode": "install",
    "requestedSpecifier": "clawhub:weather@1.0.0"
  },
  "skill": {
    "installId": "clawhub"
  }
}
```

Minimaler Richtlinienbefehl:

```js
#!/usr/bin/env node

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  input += chunk;
});
process.stdin.on("end", () => {
  const request = JSON.parse(input);
  if (request.targetType === "plugin" && request.source?.kind === "local-path") {
    process.stdout.write(
      JSON.stringify({
        protocolVersion: 1,
        decision: "block",
        reason: "local plugin paths are not approved on this host",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## Positivliste gebündelter Skills

<ParamField path="skills.allowBundled" type="string[]">
  Optionale Positivliste ausschließlich für **gebündelte** Skills. Wenn sie festgelegt ist, kommen nur die
  gebündelten Skills in der Liste infrage. Verwaltete Skills sowie Skills auf Agenten- und Workspace-Ebene
  sind davon nicht betroffen.
</ParamField>

## Einträge pro Skill (`skills.entries`)

Schlüssel unter `entries` entsprechen standardmäßig der `name` des Skills. Wenn ein Skill
`metadata.openclaw.skillKey` definiert, verwenden Sie stattdessen diesen Schlüssel. Setzen Sie Namen mit Bindestrichen
in Anführungszeichen (JSON5 erlaubt Schlüssel in Anführungszeichen).

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  `false` deaktiviert den Skill, selbst wenn er gebündelt oder installiert ist. Der
  gebündelte Skill `coding-agent` muss explizit aktiviert werden — setzen Sie ihn auf `true` und stellen Sie sicher, dass
  `claude`, `codex`, `opencode` oder eine andere unterstützte CLI installiert und
  authentifiziert ist.
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  Komfortfeld für Skills, die `metadata.openclaw.primaryEnv` deklarieren.
  Unterstützt eine Klartextzeichenfolge oder eine SecretRef: `{ source: "env", provider: "default", id: "VAR_NAME" }`.
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  Umgebungsvariablen, die für die Agentenausführung eingefügt werden. Sie werden nur eingefügt, wenn die
  Variable im Prozess noch nicht festgelegt ist.
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  Optionales Objekt für benutzerdefinierte Konfigurationsfelder pro Skill.
</ParamField>

## Agenten-Positivlisten (`agents`)

Verwenden Sie die Agentenkonfiguration, wenn dieselben Skill-Stammverzeichnisse für Maschine und Workspace,
aber für jeden Agenten eine andere sichtbare Skill-Menge verwendet werden soll.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // shared baseline
    },
    list: [
      { id: "writer" }, // inherits github, weather
      { id: "docs", skills: ["docs-search"] }, // replaces defaults entirely
      { id: "locked-down", skills: [] }, // no skills
    ],
  },
}
```

<ParamField path="agents.defaults.skills" type="string[]">
  Gemeinsame Basis-Positivliste, die von Agenten übernommen wird, bei denen
  `agents.entries.*.skills` fehlt. Lassen Sie sie vollständig weg, damit Skills standardmäßig
  nicht eingeschränkt werden.
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  Explizite endgültige Skill-Menge für diesen Agenten. Explizite Listen **ersetzen**
  übernommene Standardwerte — sie werden nicht zusammengeführt. Setzen Sie sie auf `[]`, um für
  diesen Agenten keine Skills bereitzustellen.
</ParamField>

<Warning>
  Agenten-Skill-Positivlisten dienen als Sichtbarkeits- und Ladefilter für die OpenClaw-
  Skill-Erkennung, Prompts, Slash-Befehlserkennung, Sandbox-Synchronisierung und Skill-
  Snapshots. Sie bilden keine Autorisierungsgrenze zur Shell-Laufzeit. Wenn ein Agent
  `exec` auf dem Host ausführen kann, kann diese Shell weiterhin externe Clients ausführen oder
  Hostdateien lesen, die für den ausführenden Benutzer sichtbar sind, einschließlich MCP-Client-
  Registrierungen wie `~/.openclaw/skills/config/mcporter.json`. Kombinieren Sie für die
  agentenspezifische MCP-Isolierung Skill-Positivlisten mit einer Sandbox-/Betriebssystembenutzer-
  Isolierung, verweigern Sie die Host-Ausführung oder begrenzen Sie sie durch eine enge Positivliste und bevorzugen Sie
  agentenspezifische Anmeldedaten auf dem MCP-Server.
</Warning>

## Workshop (`skills.workshop`)

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  Wenn `true` festgelegt ist, kann OpenClaw aus dauerhaften Korrekturen ausstehende Vorschläge erstellen
  und erfolgreiche, umfangreiche abgeschlossene Arbeiten prüfen, nachdem das System
  inaktiv geworden ist. Dadurch kann nach geeigneten Durchläufen ein Modellauslauf im Hintergrund
  hinzukommen. Die vom Benutzer veranlasste Erstellung von Skills und `/learn` funktionieren weiterhin, wenn die Einstellung `false` ist.
</ParamField>

Unter [Selbstlernen](/de/tools/self-learning) finden Sie Informationen zu Eignung, Datenschutz, Kosten,
Berechtigungen nur für Vorschläge und Fehlerbehebung.

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto` ermöglicht das vom Agenten initiierte Anwenden, Ablehnen oder unter Quarantäne stellen ohne eine
  zusätzliche Genehmigungsaufforderung. `pending` erfordert die Genehmigung durch den Betreiber.
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  Ermöglicht Skill Workshop beim Anwenden, über Symlinks von Workspace-Skills zu schreiben, deren
  tatsächliches Ziel bereits durch `skills.load.allowSymlinkTargets` als vertrauenswürdig eingestuft ist. Lassen Sie
  diese Option deaktiviert, sofern das Anwenden generierter Vorschläge nicht diesen gemeinsam genutzten
  Skill-Stammordner verändern soll.
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  Maximale Anzahl ausstehender und unter Quarantäne gestellter Vorschläge, die pro Workspace aufbewahrt werden (zulässiger
  Bereich: 1-200).
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  Maximale Größe des Vorschlagstexts in Byte (zulässiger Bereich: 1024-200000). Vorschlagsbeschreibungen
  sind separat fest auf 160 Byte begrenzt, da sie in der Erkennungs- und Listenausgabe
  erscheinen.
</ParamField>

Unter [Skill Workshop](/de/tools/skill-workshop) finden Sie Informationen zum Lebenszyklus von Vorschlägen, zu CLI-
Befehlen, Parametern der Agentenwerkzeuge und Gateway-Methoden, die von dieser Konfiguration gesteuert werden.

## Über Symlinks eingebundene Skill-Stammordner

Standardmäßig sind Workspace-, Projektagenten-, Zusatzverzeichnis- und gebündelte Skill-Stammordner
Begrenzungsgrenzen. Ein über Symlink eingebundener Skill-Ordner unter `<workspace>/skills`,
der auf ein Ziel außerhalb des Stammordners verweist, wird mit einer Protokollmeldung übersprungen.

Um ein beabsichtigtes Symlink-Layout zuzulassen, deklarieren Sie das vertrauenswürdige Ziel:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

Mit dieser Konfiguration wird `<workspace>/skills/manager -> ~/Projects/manager/skills`
nach der Realpath-Auflösung akzeptiert. `extraDirs` durchsucht das benachbarte Repository
direkt; `allowSymlinkTargets` behält den über Symlink eingebundenen Pfad für bestehende
Layouts bei.

Beim Anwenden schreibt Skill Workshop standardmäßig nicht über diese Symlinks. Damit
Workshop beim Anwenden Skills unter bereits vertrauenswürdigen Symlink-Zielen verändern kann, aktivieren
Sie dies separat:

```json5
{
  skills: {
    load: {
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    workshop: {
      allowSymlinkTargetWrites: true,
    },
  },
}
```

Verwaltete `~/.openclaw/skills`- und persönliche `~/.agents/skills`-Verzeichnisse
akzeptieren Symlinks auf Skill-Verzeichnisse bereits uneingeschränkt (die Begrenzung von
`SKILL.md` pro Skill gilt weiterhin) — `allowSymlinkTargets` wird nur
für Workspace-, Zusatzverzeichnis- und Projektagenten-Stammordner (`<workspace>/.agents/skills`)
benötigt.

## Skills in Sandboxes und Umgebungsvariablen

<Warning>
  `skills.entries.<skill>.env` und `apiKey` gelten nur für Ausführungen auf dem **Host**.
  Innerhalb einer Sandbox haben sie keine Wirkung — ein Skill, der von
  `GEMINI_API_KEY` abhängt, schlägt mit `apiKey not configured` fehl, sofern die Variable
  der Sandbox nicht separat bereitgestellt wird.
</Warning>

Übergeben Sie Geheimnisse wie folgt an eine Docker-Sandbox:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { GEMINI_API_KEY: "your-key-here" },
        },
      },
    },
  },
}
```

<Note>
  Benutzer mit Zugriff auf den Docker-Daemon können `sandbox.docker.env`-Werte
  über Docker-Metadaten einsehen. Verwenden Sie eine eingebundene Geheimnisdatei, ein benutzerdefiniertes Image oder
  einen anderen Bereitstellungsweg, wenn diese Offenlegung nicht akzeptabel ist.
</Note>

## Erinnerung an die Ladereihenfolge

```text
workspace/skills      (höchste Priorität)
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
gebündelte Skills
skills.load.extraDirs (niedrigste Priorität)
```

Änderungen an Skills und Konfiguration werden in der nächsten neuen Sitzung wirksam, wenn die
Überwachung aktiviert ist, oder beim nächsten Agentendurchlauf, wenn die Überwachung eine
Änderung erkennt.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Skills-Referenz" href="/de/tools/skills" icon="puzzle-piece">
    Was Skills sind, Ladereihenfolge, Zugriffssteuerung und SKILL.md-Format.
  </Card>
  <Card title="Skills erstellen" href="/de/tools/creating-skills" icon="hammer">
    Erstellung benutzerdefinierter Workspace-Skills.
  </Card>
  <Card title="Skill Workshop" href="/de/tools/skill-workshop" icon="flask">
    Vorschlagswarteschlange für vom Agenten entworfene Skills.
  </Card>
  <Card title="Selbstlernen" href="/de/tools/self-learning" icon="brain">
    Konservative, freiwillig aktivierte Vorschläge aus abgeschlossenen Arbeiten.
  </Card>
  <Card title="Slash-Befehle" href="/de/tools/slash-commands" icon="terminal">
    Nativer Katalog von Slash-Befehlen und Chat-Direktiven.
  </Card>
</CardGroup>
