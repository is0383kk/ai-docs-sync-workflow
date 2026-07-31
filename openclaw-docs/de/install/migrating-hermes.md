---
read_when:
    - Sie wechseln von Hermes und möchten Ihre Modellkonfiguration, Prompts, Ihren Speicher und Ihre Skills beibehalten
    - Sie möchten wissen, was OpenClaw automatisch importiert und was ausschließlich im Archiv verbleibt
    - Sie benötigen einen sauberen, skriptgesteuerten Migrationspfad (CI, neuer Laptop, Automatisierung)
summary: Wechseln Sie von Hermes zu OpenClaw mit einem vorab geprüften, reversiblen Import
title: Migration von Hermes
x-i18n:
    generated_at: "2026-07-26T18:30:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8cdb7a77cfb8ecb0504ccc322b5600c6ed671a8bf9ac866d964fdf4b3494000
    source_path: install/migrating-hermes.md
    workflow: 16
---

Der gebündelte Hermes-Migrations-Provider folgt `HERMES_HOME` und dem aktiven Hermes-Profil und greift unter macOS/Linux auf `~/.hermes` beziehungsweise unter Windows auf `%LOCALAPPDATA%\hermes` zurück. Er zeigt jede Änderung vor der Anwendung in einer Vorschau an und schwärzt Secrets in Plänen und Berichten. Das eigenständige `openclaw migrate` schreibt ein verifiziertes Backup; beim neuen Onboarding-Pfad werden Konfiguration, Anmeldedaten und Dateien zunächst bereitgestellt und erst veröffentlicht, nachdem die importierte Inferenz erfolgreich verifiziert wurde. Ein expliziter `--from`-Pfad hat immer Vorrang.

<Note>
Importe erfordern eine neue OpenClaw-Einrichtung. Wenn bereits ein lokaler OpenClaw-Zustand vorhanden ist, setzen Sie zunächst Konfiguration, Anmeldedaten, Sitzungen und den Workspace zurück oder verwenden Sie `openclaw migrate apply hermes` nach Prüfung des Plans direkt mit `--overwrite`.
</Note>

## Zwei Importmöglichkeiten

<Tabs>
  <Tab title="Onboarding-Assistent">
    Erkennt das aktive Hermes-Home/-Profil und zeigt vor der Anwendung eine Vorschau an.

    ```bash
    openclaw onboard --flow import
    ```

    Oder geben Sie eine bestimmte Quelle an:

    ```bash
    openclaw onboard --import-from hermes --import-source ~/.hermes
    ```

  </Tab>
  <Tab title="CLI">
    Verwenden Sie `openclaw migrate` für skriptgesteuerte oder wiederholbare Ausführungen. Die vollständige Referenz finden Sie unter [`openclaw migrate`](/de/cli/migrate).

    ```bash
    openclaw migrate hermes --dry-run    # nur Vorschau
    openclaw migrate apply hermes --yes  # anwenden, ohne Bestätigung abzufragen
    ```

    Fügen Sie `--from <path>` hinzu, um die Erkennung von Hermes-Home/-Profil zu überschreiben.

  </Tab>
</Tabs>

## Was importiert wird

<AccordionGroup>
  <Accordion title="Modellkonfiguration">
    - Standardmodellauswahl aus Hermes `config.yaml`.
    - Konfigurierte Modell-Provider und benutzerdefinierte Endpunkte aus `model`, `providers` und `custom_providers`, einschließlich der aktuellen Hermes-Transporte Chat Completions, Codex Responses und Anthropic Messages.

  </Accordion>
  <Accordion title="MCP-Server">
    MCP-Serverdefinitionen aus `mcp_servers` oder `mcp.servers`, einschließlich Deaktivierungsstatus, Zeitüberschreitungen, Unterstützung paralleler Tools, OAuth-Geltungsbereich, kompatibler TLS-Felder und Richtlinien für native Tools, Ressourcen-Tools und Prompt-Tools. Literale Umgebungsvariablen und Header erfordern die Zustimmung zum Import von Anmeldedaten. Nur in Hermes vorhandene Einstellungen für Lebenszyklus, Sampling, Elicitation, Preflight, Keepalive, CA-Bundles, passwortgeschützte Client-Schlüssel und vorab registrierte OAuth-Clients werden anstelle einer ungültigen OpenClaw-Konfiguration zu Elementen für die manuelle Prüfung.
  </Accordion>
  <Accordion title="Workspace-Dateien">
    - `SOUL.md` und `AGENTS.md` werden in den OpenClaw-Agent-Workspace kopiert.
    - `memories/MEMORY.md` und `memories/USER.md` werden an die entsprechenden OpenClaw-Speicherdateien **angehängt**, anstatt sie zu überschreiben.
    - Reine Speicheroberflächen verhalten sich anders: Die Speicherseite des Onboardings und die Seite für Speicherimporte in der Control UI kopieren diese beiden Dateien zur indizierten Erinnerung unter `memory/imports/hermes/` und lassen den vorhandenen Workspace-Speicher unverändert.

  </Accordion>
  <Accordion title="Speicherkonfiguration">
    Standardwerte der Speicherkonfiguration für den OpenClaw-Dateispeicher. Externe Speicher-Provider wie Honcho werden als Archiv- oder manuell zu prüfende Elemente erfasst, damit Sie sie gezielt verschieben können.
  </Accordion>
  <Accordion title="Skills">
    Skills mit einer `SKILL.md`-Datei an einer beliebigen Stelle unter `skills/` werden rekursiv erkannt, in das Skill-Verzeichnis des OpenClaw-Workspace überführt und zusammen mit ihren unterstützenden Dateien kopiert. Skill-spezifische Konfigurationswerte aus `skills.config` bleiben erhalten.
  </Accordion>
  <Accordion title="Authentifizierungsdaten">
    Das interaktive `openclaw migrate` fragt vor dem Import von Authentifizierungsdaten nach, wobei „Ja“ standardmäßig ausgewählt ist. Akzeptierte Importe umfassen aktuelle OAuth-Einträge für Hermes OpenAI Codex, OpenAI-OAuth- und GitHub-Copilot-Einträge aus OpenCode sowie die [unterstützten Hermes-Schlüssel `.env`](/de/cli/migrate#supported-env-keys). Verwenden Sie `--include-secrets` für einen nicht interaktiven Import, `--no-auth-credentials`, um Anmeldedaten zu überspringen, oder das Onboarding-Flag `--import-secrets`. Lassen Sie Hermes und OpenClaw nach dem Import von Hermes-OAuth nicht denselben Refresh Grant verwenden; authentifizieren Sie eine Seite erneut, bevor beide gleichzeitig ausgeführt werden.
  </Accordion>
</AccordionGroup>

## Was ausschließlich archiviert wird

Der Provider kopiert Folgendes zur manuellen Prüfung in das Verzeichnis des Migrationsberichts, lädt es jedoch **nicht** in die aktive OpenClaw-Konfiguration oder die Anmeldedaten:

- `plugins/`
- `sessions/`
- `logs/`
- `cron/`
- `mcp-tokens/`
- `plans/`, `workspace/`, `skins/` und `kanban/`
- `pairing/`- und `platforms/`-Speicher sowie Gateway-Routing- und Prozesszustand
- `state.db`, `hermes_state.db`, `projects.db`, `response_store.db`, `memory_store.db`, `verification_evidence.db`, `kanban.db` und `retaindb_queue.db`

OpenClaw weigert sich, diesen Zustand automatisch auszuführen oder ihm zu vertrauen, da Formate und Vertrauensannahmen zwischen Systemen voneinander abweichen können. Verschieben Sie die benötigten Elemente nach Prüfung des Archivs manuell.

## Empfohlener Ablauf

<Steps>
  <Step title="Planvorschau anzeigen">
    ```bash
    openclaw migrate hermes --dry-run
    ```

    Der Plan führt alle Änderungen auf, einschließlich Konflikten, übersprungenen Elementen und sensiblen Elementen. Verschachtelte Schlüssel, die wie Secrets aussehen, werden in der Ausgabe geschwärzt.

  </Step>
  <Step title="Mit Backup anwenden">
    ```bash
    openclaw migrate apply hermes --yes
    ```

    OpenClaw erstellt und verifiziert vor der Anwendung ein Backup. Dieses nicht interaktive Beispiel importiert ausschließlich nicht geheimen Zustand. Führen Sie den Befehl ohne `--yes` aus, um die Abfrage zu Anmeldedaten interaktiv zu beantworten, oder fügen Sie `--include-secrets` hinzu, um unterstützte Anmeldedaten in eine unbeaufsichtigte Ausführung einzubeziehen.

  </Step>
  <Step title="Doctor ausführen">
    ```bash
    openclaw doctor
    ```

    [Doctor](/de/gateway/doctor) wendet ausstehende Konfigurationsmigrationen erneut an und prüft auf Probleme, die während des Imports entstanden sind.

  </Step>
  <Step title="Neu starten und überprüfen">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Vergewissern Sie sich, dass das Gateway fehlerfrei funktioniert und Ihr importiertes Modell, Ihr Speicher und Ihre Skills geladen sind.

  </Step>
</Steps>

## Konfliktbehandlung

Die Anwendung wird nicht fortgesetzt, wenn der Plan Konflikte meldet (eine Datei oder ein Konfigurationswert ist am Ziel bereits vorhanden).

<Warning>
Führen Sie den Vorgang nur dann erneut mit `--overwrite` aus, wenn das vorhandene Ziel absichtlich ersetzt werden soll. Provider können für überschriebene Dateien dennoch Backups auf Elementebene in das Verzeichnis des Migrationsberichts schreiben.
</Warning>

Bei einer neuen Installation sind Konflikte ungewöhnlich. Sie treten üblicherweise auf, wenn Sie den Import für eine Einrichtung erneut ausführen, die bereits Benutzeränderungen enthält.

Wenn während der Anwendung ein Konflikt auftritt (beispielsweise durch einen unerwarteten Wettlauf bei einer Konfigurationsdatei), wird das betreffende Element als Konflikt gemeldet, während unabhängige Dateien, Skills, Anmeldedaten, Archive und Konfigurationseinträge weiterverarbeitet werden. Beheben Sie das in Konflikt stehende Element und führen Sie den Import erneut aus; identische Speicherimporte sind idempotent.

## Secrets

Das interaktive `openclaw migrate` fragt, ob erkannte Authentifizierungsdaten importiert werden sollen, wobei „Ja“ standardmäßig ausgewählt ist.

- Bei Zustimmung werden aktuelle OAuth-Einträge für Hermes OpenAI Codex, OpenAI-OAuth- und GitHub-Copilot-Einträge aus OpenCode sowie die [unterstützten Schlüssel `.env`](/de/cli/migrate#supported-env-keys) importiert.
- Verwenden Sie `--no-auth-credentials` oder antworten Sie bei der Abfrage mit „Nein“, um ausschließlich nicht geheimen Zustand zu importieren.
- Verwenden Sie `--include-secrets`, um Anmeldedaten bei einer unbeaufsichtigten Ausführung von `--yes` zu importieren.
- Verwenden Sie das Flag `--import-secrets` des Onboarding-Assistenten, um Anmeldedaten über den Assistenten zu importieren.

## JSON-Ausgabe für die Automatisierung

```bash
openclaw migrate hermes --dry-run --json
openclaw migrate apply hermes --json --yes
```

Mit `--json` und ohne `--yes` gibt die Anwendung den Plan aus und verändert keinen Zustand – der sicherste Modus für CI und gemeinsam genutzte Skripte.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Die Anwendung wird aufgrund von Konflikten verweigert">
    Prüfen Sie die Planausgabe. Jeder Konflikt gibt den Quellpfad und das vorhandene Ziel an. Entscheiden Sie für jedes Element, ob es übersprungen, das Ziel bearbeitet oder der Vorgang mit `--overwrite` erneut ausgeführt werden soll.
  </Accordion>
  <Accordion title="Hermes befindet sich außerhalb von ~/.hermes">
    Übergeben Sie `--from /actual/path` (CLI) oder `--import-source /actual/path` (Onboarding).
  </Accordion>
  <Accordion title="Onboarding verweigert den Import in eine vorhandene Einrichtung">
    Onboarding-Importe erfordern eine neue Einrichtung. Setzen Sie entweder den Zustand zurück und führen Sie das Onboarding erneut durch oder verwenden Sie `openclaw migrate apply hermes` direkt. Dies unterstützt `--overwrite` und eine explizite Backup-Steuerung.
  </Accordion>
  <Accordion title="API-Schlüssel wurden nicht importiert">
    Das interaktive `openclaw migrate` importiert API-Schlüssel nur, wenn Sie der Abfrage zu Anmeldedaten zustimmen. Nicht interaktive Ausführungen von `--yes` benötigen `--include-secrets`; Onboarding-Importe benötigen `--import-secrets`. Nur die [unterstützten Schlüssel `.env`](/de/cli/migrate#supported-env-keys) werden erkannt – andere Variablen von `.env` werden ignoriert.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [`openclaw migrate`](/de/cli/migrate): vollständige CLI-Referenz, Plugin-Vertrag und JSON-Strukturen.
- [Onboarding](/de/cli/onboard): Ablauf des Assistenten und nicht interaktive Flags.
- [Migration](/de/install/migrating): Verschieben einer OpenClaw-Installation zwischen Computern.
- [Doctor](/de/gateway/doctor): Zustandsprüfung nach der Migration.
- [Agent-Workspace](/de/concepts/agent-workspace): Speicherort von `SOUL.md`, `AGENTS.md` und Speicherdateien.
