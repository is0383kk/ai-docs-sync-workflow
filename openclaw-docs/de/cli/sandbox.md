---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: Sandbox-Laufzeitumgebungen verwalten und die geltende Sandbox-Richtlinie prüfen
title: Sandbox-CLI
x-i18n:
    generated_at: "2026-07-26T18:23:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8311de7702222295f3ba8753304e30f6ed21958e2843f62db5d064f06e24ae
    source_path: cli/sandbox.md
    workflow: 16
---

Verwalten Sie Sandbox-Laufzeitumgebungen für die isolierte Agent-Ausführung: Docker-Container, SSH-Ziele oder OpenShell-Backends.

## Befehle

### `openclaw sandbox list`

Listet Sandbox-Laufzeitumgebungen mit Status, Backend, Konfigurationsübereinstimmung, Alter, Leerlaufzeit und zugehöriger Sitzung bzw. zugehörigem Agent auf.

```bash
openclaw sandbox list
openclaw sandbox list --browser  # nur Browser-Container
openclaw sandbox list --json
```

### `openclaw sandbox recreate`

Entfernt Sandbox-Laufzeitumgebungen, um deren Neuerstellung mit der aktuellen Konfiguration zu erzwingen. Die Laufzeitumgebungen werden automatisch neu erstellt, wenn der Agent das nächste Mal verwendet wird.

```bash
openclaw sandbox recreate --all
openclaw sandbox recreate --agent mybot        # schließt Untersitzungen vom Typ agent:mybot:* ein
openclaw sandbox recreate --session "agent:main:main"
openclaw sandbox recreate --browser --all      # nur Browser-Container
openclaw sandbox recreate --all --force        # Bestätigung überspringen
```

Optionen:

- `--all`: alle Sandbox-Container neu erstellen
- `--session <key>`: die Laufzeitumgebung mit diesem exakten Bereichsschlüssel neu erstellen (wie von `sandbox list` angezeigt); keine Erweiterung von Kurznamen
- `--agent <id>`: Laufzeitumgebungen für einen Agent neu erstellen (entspricht `agent:<id>` und `agent:<id>:*`)
- `--browser`: nur Browser-Container betreffen
- `--force`: Bestätigungsabfrage überspringen

Übergeben Sie genau eine der Optionen `--all`, `--session` oder `--agent`.

Bei `ssh` und OpenShell `remote` ist die Neuerstellung wichtiger als bei Docker: Nach der anfänglichen Befüllung ist der entfernte Arbeitsbereich maßgeblich, `recreate` löscht diesen maßgeblichen entfernten Arbeitsbereich für den ausgewählten Bereich, und beim nächsten Lauf wird er aus dem aktuellen lokalen Arbeitsbereich neu befüllt.

### `openclaw sandbox explain`

Prüft den effektiven Sandbox-Modus und -Bereich, den Arbeitsbereichszugriff, die Richtlinie für Sandbox-Werkzeuge und die Schranken für Werkzeuge mit erhöhten Rechten (einschließlich Konfigurationsschlüsselpfaden zur Behebung).

Der Bericht behält `workspaceRoot` als konfiguriertes Sandbox-Stammverzeichnis bei und zeigt separat den effektiven Host-Arbeitsbereich, das Arbeitsverzeichnis der Backend-Laufzeitumgebung und die Docker-Einhängetabelle an. Bei `workspaceAccess: "rw"` ist der effektive Host-Arbeitsbereich der Agent-Arbeitsbereich und kein Verzeichnis unterhalb von `workspaceRoot`.

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Anders als `recreate --session` akzeptiert dieser Befehl kurze Sitzungsnamen (beispielsweise `main`) und erweitert sie anhand des aufgelösten Agent.

## Warum eine Neuerstellung erforderlich ist

Das Aktualisieren der Sandbox-Konfiguration wirkt sich nicht auf laufende Container aus: Bestehende Laufzeitumgebungen behalten ihre alten Einstellungen bei, und inaktive Laufzeitumgebungen werden erst nach `prune.idleHours` bereinigt (standardmäßig 24h). Regelmäßig verwendete Agents können veraltete Laufzeitumgebungen unbegrenzt aktiv halten. `openclaw sandbox recreate` entfernt die alte Laufzeitumgebung, sodass sie bei der nächsten Verwendung anhand der aktuellen Konfiguration neu erstellt wird.

<Tip>
Verwenden Sie vorzugsweise `openclaw sandbox recreate` statt einer manuellen, Backend-spezifischen Bereinigung. Der Befehl verwendet die Laufzeitregistrierung des Gateways und vermeidet Abweichungen, wenn sich Bereichs- oder Sitzungsschlüssel ändern.
</Tip>

## Häufige Auslöser

| Änderung                                                                                                                                                        | Befehl                                                              |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Docker-Image-Aktualisierung (`agents.defaults.sandbox.docker.image`)                                                                                                                | `openclaw sandbox recreate --all`                                                  |
| Sandbox-Konfiguration (`agents.defaults.sandbox.*`)                                                                                                                      | `openclaw sandbox recreate --all`                                                  |
| SSH-Ziel/Authentifizierung (`agents.defaults.sandbox.ssh.{target,workspaceRoot,identityFile,certificateFile,knownHostsFile,identityData,certificateData,knownHostsData}`)                                                                                                                 | `openclaw sandbox recreate --all`                                                  |
| OpenShell-Quelle/-Richtlinie/-Modus (`plugins.entries.openshell.config.{from,mode,policy}`)                                                                                                        | `openclaw sandbox recreate --all`                                                  |
| `setupCommand`                                                                                                                                             | `openclaw sandbox recreate --all` (oder `--agent <id>` für einen Agent)        |

<Note>
Laufzeitumgebungen werden automatisch neu erstellt, wenn der Agent das nächste Mal verwendet wird.
</Note>

## Registrierungsmigration

Die Metadaten der Sandbox-Laufzeitumgebungen befinden sich in der gemeinsam genutzten SQLite-Zustandsdatenbank. Ältere Installationen können veraltete Registrierungsdateien enthalten, die bei regulären Lesevorgängen nicht mehr neu geschrieben werden:

- `~/.openclaw/sandbox/containers.json`
- `~/.openclaw/sandbox/browsers.json`
- ein JSON-Fragment pro Container/Browser unter `~/.openclaw/sandbox/containers/` oder `~/.openclaw/sandbox/browsers/`

Führen Sie `openclaw doctor --fix` aus, um gültige veraltete Einträge nach SQLite zu migrieren. Ungültige veraltete Dateien werden unter Quarantäne gestellt, damit eine beschädigte alte Registrierung keine aktuellen Laufzeitumgebungseinträge verbergen kann.

## Konfiguration

Die Sandbox-Einstellungen befinden sich in `~/.openclaw/openclaw.json` unter `agents.defaults.sandbox` (Agent-spezifische Überschreibungen werden in `agents.entries.*.sandbox` eingetragen):

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // aus, nicht Haupt-Agent, alle
        "backend": "docker", // docker, ssh, openshell (von Plugin bereitgestellt)
        "scope": "agent", // Sitzung, Agent, gemeinsam
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... weitere Docker-Optionen
        },
        "prune": {
          "idleHours": 24, // nach 24h Leerlauf automatisch bereinigen
          "maxAgeDays": 7, // nach 7 Tagen automatisch bereinigen
        },
      },
    },
  },
}
```

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Sandboxing](/de/gateway/sandboxing)
- [Agent-Arbeitsbereich](/de/concepts/agent-workspace)
- [Doctor](/de/gateway/doctor): prüft die Sandbox-Einrichtung.
