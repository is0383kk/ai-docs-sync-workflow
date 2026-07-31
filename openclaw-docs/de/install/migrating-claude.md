---
read_when:
    - Sie wechseln von Claude Code oder Claude Desktop und möchten Anweisungen, MCP-Server und Skills beibehalten
    - Sie müssen verstehen, was OpenClaw automatisch importiert und was ausschließlich im Archiv verbleibt.
summary: Lokalen Status von Claude Code und Claude Desktop mit einem Import samt Vorschau in OpenClaw übertragen
title: Migration von Claude
x-i18n:
    generated_at: "2026-07-26T18:26:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d5a5e63727e1583fc3fa27ac45215c72df9074b21d7c5f6b33800bec916769b
    source_path: install/migrating-claude.md
    workflow: 16
---

OpenClaw importiert lokalen Claude-Status über den gebündelten Claude-Migrations-Provider. Der Provider zeigt vor jeder Statusänderung eine Vorschau aller Elemente an und schwärzt Geheimnisse in Plänen und Berichten. Der eigenständige Aufruf `openclaw migrate` erstellt eine verifizierte Sicherung; beim Ablauf für eine neue Ersteinrichtung wird der Import zunächst vorbereitet und erst nach erfolgreicher Verifizierung veröffentlicht.

<Note>
Importe während der Ersteinrichtung erfordern eine neue OpenClaw-Einrichtung. Wenn bereits lokaler OpenClaw-Status vorhanden ist, setzen Sie zuerst Konfiguration, Anmeldedaten, Sitzungen und den Arbeitsbereich zurück, oder verwenden Sie `openclaw migrate` nach Prüfung des Plans direkt mit `--overwrite`.
</Note>

## Zwei Importmöglichkeiten

<Tabs>
  <Tab title="Einrichtungsassistent">
    Der Assistent bietet Claude an, wenn er lokalen Claude-Status erkennt.

    ```bash
    openclaw onboard --flow import
    ```

    Alternativ kann eine bestimmte Quelle angegeben werden:

    ```bash
    openclaw onboard --import-from claude --import-source ~/.claude
    ```

  </Tab>
  <Tab title="CLI">
    Verwenden Sie `openclaw migrate` für skriptgesteuerte oder wiederholbare Ausführungen. Die vollständige Referenz finden Sie unter [`openclaw migrate`](/de/cli/migrate).

    ```bash
    openclaw migrate claude --dry-run
    openclaw migrate apply claude --yes
    ```

    Fügen Sie `--from <path>` hinzu, um ein bestimmtes Claude-Code-Stammverzeichnis oder Projektstammverzeichnis zu importieren.

  </Tab>
</Tabs>

## Was importiert wird

<AccordionGroup>
  <Accordion title="Anweisungen und Speicher">
    - Inhalte aus dem Projekt `CLAUDE.md` und `.claude/CLAUDE.md` werden in den OpenClaw-Agentenarbeitsbereich `AGENTS.md` kopiert oder daran angehängt.
    - Inhalte aus dem Benutzerverzeichnis `~/.claude/CLAUDE.md` werden an den Arbeitsbereich `USER.md` angehängt.

  </Accordion>
  <Accordion title="MCP-Server">
    Definitionen von MCP-Servern werden, sofern vorhanden, aus dem Projekt `.mcp.json`, aus Claude Code `~/.claude.json` und aus Claude Desktop `claude_desktop_config.json` importiert.
  </Accordion>
  <Accordion title="Skills und Befehle">
    - Claude-Skills mit einer Datei `SKILL.md` werden in das Skills-Verzeichnis des OpenClaw-Arbeitsbereichs kopiert.
    - Markdown-Dateien mit Claude-Befehlen unter `.claude/commands/` oder `~/.claude/commands/` werden mit `disable-model-invocation: true` in OpenClaw-Skills umgewandelt.

  </Accordion>
</AccordionGroup>

## Was ausschließlich archiviert wird

Der Provider kopiert Folgendes zur manuellen Prüfung in den Migrationsbericht, lädt es jedoch **nicht** in die aktive OpenClaw-Konfiguration:

- Claude-Hooks
- Claude-Berechtigungen und umfassende Zulassungslisten für Tools
- Claude-Standardwerte für die Umgebung
- `CLAUDE.local.md`
- `.claude/rules/`
- Claude-Unteragenten unter `.claude/agents/` oder `~/.claude/agents/`
- Caches, Pläne und Projektverlaufsverzeichnisse von Claude Code
- Claude-Desktop-Erweiterungen und im Betriebssystem gespeicherte Anmeldedaten

OpenClaw weigert sich, Hooks auszuführen, Berechtigungs-Zulassungslisten zu vertrauen oder undurchsichtige OAuth- und Desktop-Anmeldedaten automatisch zu dekodieren. Übertragen Sie benötigte Elemente nach Prüfung des Archivs manuell.

## Quellenauswahl

Ohne `--from` prüft OpenClaw das standardmäßige Claude-Code-Stammverzeichnis unter `~/.claude`, die stichprobenartig ausgewählte Claude-Code-Statusdatei `~/.claude.json` und die MCP-Konfiguration von Claude Desktop unter macOS.

Wenn `--from` auf ein Projektstammverzeichnis verweist, importiert OpenClaw nur die Claude-Dateien dieses Projekts, beispielsweise `CLAUDE.md`, `.claude/settings.json`, `.claude/commands/`, `.claude/skills/` und `.mcp.json`. Bei einem Import aus einem Projektstammverzeichnis wird das globale Claude-Stammverzeichnis nicht gelesen.

## Empfohlener Ablauf

<Steps>
  <Step title="Vorschau des Plans anzeigen">
    ```bash
    openclaw migrate claude --dry-run
    ```

    Der Plan führt alle bevorstehenden Änderungen auf, einschließlich Konflikten, übersprungenen Elementen und sensiblen Werten, die in verschachtelten MCP-Feldern `env` oder `headers` geschwärzt wurden.

  </Step>
  <Step title="Mit Sicherung anwenden">
    ```bash
    openclaw migrate apply claude --yes
    ```

    OpenClaw erstellt und verifiziert vor der Anwendung eine Sicherung.

  </Step>
  <Step title="Doctor ausführen">
    ```bash
    openclaw doctor
    ```

    [Doctor](/de/gateway/doctor) prüft nach dem Import auf Probleme mit der Konfiguration oder dem Status.

  </Step>
  <Step title="Neu starten und überprüfen">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    Vergewissern Sie sich, dass das Gateway fehlerfrei funktioniert und die importierten Anweisungen, MCP-Server und Skills geladen sind.

  </Step>
</Steps>

## Konfliktbehandlung

Die Anwendung wird verweigert, wenn der Plan Konflikte meldet, weil am Ziel bereits eine Datei oder ein Konfigurationswert vorhanden ist.

<Warning>
Führen Sie den Vorgang nur dann erneut mit `--overwrite` aus, wenn das vorhandene Ziel absichtlich ersetzt werden soll. Provider können für überschriebene Dateien weiterhin Sicherungen auf Elementebene im Verzeichnis des Migrationsberichts erstellen.
</Warning>

Bei einer neuen OpenClaw-Installation sind Konflikte ungewöhnlich. Sie treten normalerweise auf, wenn der Import für eine Einrichtung erneut ausgeführt wird, die bereits Benutzeränderungen enthält.

## JSON-Ausgabe für die Automatisierung

```bash
openclaw migrate claude --dry-run --json
openclaw migrate apply claude --json --yes
```

`--yes` ist für `migrate apply` außerhalb eines interaktiven Terminals erforderlich. Ohne diese Angabe gibt OpenClaw einen Fehler aus, statt die Änderungen anzuwenden. Skripte und die CI müssen `--yes` daher ausdrücklich übergeben. Zeigen Sie zuerst mit `--dry-run --json` eine Vorschau an und wenden Sie die Änderungen anschließend mit `--json --yes` an, sobald der Plan korrekt aussieht.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Der Claude-Status befindet sich außerhalb von ~/.claude">
    Übergeben Sie `--from /actual/path` für die CLI oder `--import-source /actual/path` für die Ersteinrichtung.
  </Accordion>
  <Accordion title="Die Ersteinrichtung verweigert den Import in eine vorhandene Einrichtung">
    Importe während der Ersteinrichtung erfordern eine neue Einrichtung. Setzen Sie entweder den Status zurück und führen Sie die Ersteinrichtung erneut durch, oder verwenden Sie direkt `openclaw migrate apply claude`, das `--overwrite` und eine ausdrückliche Steuerung der Sicherung unterstützt.
  </Accordion>
  <Accordion title="MCP-Server aus Claude Desktop wurden nicht importiert">
    Claude Desktop liest `claude_desktop_config.json` aus einem plattformspezifischen Pfad. Lassen Sie `--from` auf das Verzeichnis dieser Datei verweisen, falls OpenClaw es nicht automatisch erkannt hat.
  </Accordion>
  <Accordion title="Claude-Befehle wurden zu Skills mit deaktiviertem Modellaufruf">
    Dies ist beabsichtigt. Claude-Befehle werden durch Benutzer ausgelöst, daher importiert OpenClaw sie als Skills mit `disable-model-invocation: true`. Bearbeiten Sie das Frontmatter jedes Skills, wenn der Agent sie automatisch aufrufen soll.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [`openclaw migrate`](/de/cli/migrate): vollständige CLI-Referenz, Plugin-Vertrag und JSON-Strukturen.
- [Migrationsleitfaden](/de/install/migrating): alle Migrationspfade.
- [Migration von Hermes](/de/install/migrating-hermes): der andere systemübergreifende Importpfad.
- [Ersteinrichtung](/de/cli/onboard): Assistentenablauf und Flags für die nicht interaktive Verwendung.
- [Doctor](/de/gateway/doctor): Zustandsprüfung nach der Migration.
- [Agentenarbeitsbereich](/de/concepts/agent-workspace): Speicherort von `AGENTS.md`, `USER.md` und Skills.
