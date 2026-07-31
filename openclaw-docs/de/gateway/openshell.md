---
read_when:
    - Sie möchten cloudverwaltete Sandboxes anstelle von lokalem Docker verwenden
    - Sie richten das OpenShell-Plugin ein
    - Sie müssen zwischen dem Spiegel- und dem Remote-Arbeitsbereichsmodus wählen.
summary: Verwenden Sie OpenShell als verwaltetes Sandbox-Backend für OpenClaw-Agenten
title: OpenShell
x-i18n:
    generated_at: "2026-07-26T17:50:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell ist ein verwaltetes Sandbox-Backend: Statt Docker-Container
lokal auszuführen, delegiert OpenClaw den Sandbox-Lebenszyklus an die `openshell`-CLI, die
Remote-Umgebungen bereitstellt und Befehle über SSH ausführt.

Das Plugin verwendet denselben SSH-Transport und dieselbe Remote-Dateisystem-Bridge wie das
generische [SSH-Backend](/de/gateway/sandboxing#ssh-backend) und ergänzt den OpenShell-
Lebenszyklus (`sandbox create/get/delete/ssh-config`) sowie einen optionalen `mirror`-
Workspace-Synchronisierungsmodus.

## Voraussetzungen

- OpenShell-Plugin installiert (`openclaw plugins install @openclaw/openshell-sandbox`)
- `openshell`-CLI in `PATH` (oder ein benutzerdefinierter Pfad über
  `plugins.entries.openshell.config.command`)
- Ein OpenShell-Konto mit Sandbox-Zugriff
- OpenClaw Gateway wird auf dem Host ausgeführt

## Schnellstart

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

Starten Sie das Gateway neu. Beim nächsten Agent-Durchlauf erstellt OpenClaw eine OpenShell-
Sandbox und leitet die Tool-Ausführung durch sie. Überprüfen Sie dies mit:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## Workspace-Modi

Dies ist die wichtigste OpenShell-Entscheidung.

### mirror (Standard)

`plugins.entries.openshell.config.mode: "mirror"` hält den **lokalen Workspace
kanonisch**:

- Vor `exec` synchronisiert OpenClaw den lokalen Workspace in die Sandbox.
- Nach `exec` synchronisiert OpenClaw den Remote-Workspace zurück auf das lokale System.
- Datei-Tools verwenden die Sandbox-Bridge, aber das lokale System bleibt zwischen
  Durchläufen die maßgebliche Datenquelle.

Optimal für Entwicklungsabläufe: Lokale Änderungen außerhalb von OpenClaw erscheinen bei der
nächsten Ausführung, und die Sandbox verhält sich ähnlich wie das Docker-Backend.

Nachteil: Upload- und Download-Kosten bei jedem Ausführungsdurchlauf.

### remote

`mode: "remote"` macht den **OpenShell-Workspace kanonisch**:

- Bei der ersten Erstellung der Sandbox initialisiert OpenClaw den Remote-Workspace einmalig
  aus dem lokalen Workspace.
- Danach arbeiten `exec`, `read`, `write`, `edit` und `apply_patch`
  direkt im Remote-Workspace. OpenClaw synchronisiert Remote-Änderungen **nicht**
  zurück auf das lokale System.
- Medienzugriffe zur Prompt-Zeit funktionieren weiterhin (Datei-/Medien-Tools lesen über die
  Sandbox-Bridge).

Optimal für lang laufende Agents und CI: geringerer Aufwand pro Durchlauf, und hostlokale
Änderungen können den Remote-Zustand nicht unbemerkt überschreiben.

<Warning>
Änderungen an Dateien auf dem Host außerhalb von OpenClaw sind nach der anfänglichen Initialisierung für die Remote-Sandbox nicht sichtbar. Führen Sie `openclaw sandbox recreate` aus, um sie neu zu initialisieren.
</Warning>

### Modus auswählen

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **Kanonischer Workspace**  | Lokaler Host                 | Remote-OpenShell          |
| **Synchronisierungsrichtung**       | Bidirektional (bei jeder Ausführung) | Einmalige Initialisierung             |
| **Aufwand pro Durchlauf**    | Höher (Upload + Download) | Niedriger (direkte Remote-Operationen) |
| **Lokale Änderungen sichtbar?** | Ja, bei der nächsten Ausführung          | Nein, bis zur Neuerstellung        |
| **Optimal für**             | Entwicklungsabläufe      | Lang laufende Agents, CI   |

## Konfigurationsreferenz

Die gesamte OpenShell-Konfiguration befindet sich unter `plugins.entries.openshell.config`:

| Schlüssel                       | Typ                     | Standard       | Beschreibung                                                                            |
| ------------------------- | ------------------------ | ------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` oder `"remote"` | `"mirror"`    | Workspace-Synchronisierungsmodus                                                                    |
| `command`                 | `string`                 | `"openshell"` | Pfad oder Name der `openshell`-CLI                                                    |
| `from`                    | `string`                 | `"openclaw"`  | Sandbox-Quelle für die erstmalige Erstellung                                                   |
| `gateway`                 | `string`                 | nicht gesetzt         | OpenShell-Gateway-Name (`--gateway` auf oberster Ebene)                                         |
| `gatewayEndpoint`         | `string`                 | nicht gesetzt         | OpenShell-Gateway-Endpunkt (`--gateway-endpoint` auf oberster Ebene)                            |
| `policy`                  | `string`                 | nicht gesetzt         | OpenShell-Richtlinien-ID für die Sandbox-Erstellung                                               |
| `providers`               | `string[]`               | `[]`          | Bei der Sandbox-Erstellung angehängte Provider-Namen (dedupliziert, ein `--provider`-Flag pro Eintrag) |
| `gpu`                     | `boolean`                | `false`       | GPU-Ressourcen anfordern (`--gpu`)                                                        |
| `autoProviders`           | `boolean`                | `true`        | Bei der Erstellung `--auto-providers` (oder `--no-auto-providers`, wenn falsch) übergeben            |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | Primärer beschreibbarer Workspace innerhalb der Sandbox                                          |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | Einhängepfad des Agent-Workspace (schreibgeschützt, wenn der Workspace-Zugriff nicht `rw` ist)               |
| `timeoutSeconds`          | `number`                 | `120`         | Zeitüberschreitung für `openshell`-CLI-Operationen                                                 |

`remoteWorkspaceDir` und `remoteAgentWorkspaceDir` müssen absolute Pfade sein und
innerhalb der verwalteten Stammverzeichnisse `/sandbox` oder `/agent` bleiben; andere absolute Pfade werden
abgelehnt.

Einstellungen auf Sandbox-Ebene (`mode`, `scope`, `workspaceAccess`) befinden sich wie bei jedem Backend unter
`agents.defaults.sandbox`. Die vollständige Matrix finden Sie unter
[Sandboxing](/de/gateway/sandboxing).

## Beispiele

### Minimale Remote-Einrichtung

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### Mirror-Modus mit GPU

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### Agent-spezifisches OpenShell mit benutzerdefiniertem Gateway

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## Lebenszyklusverwaltung

```bash
# Alle Sandbox-Laufzeitumgebungen auflisten (Docker + OpenShell)
openclaw sandbox list

# Effektive Richtlinie prüfen
openclaw sandbox explain

# Neu erstellen (löscht den Remote-Workspace, initialisiert ihn bei der nächsten Verwendung neu)
openclaw sandbox recreate --all
```

Für den `remote`-Modus ist die Neuerstellung besonders wichtig: Sie löscht den kanonischen
Remote-Workspace für diesen Geltungsbereich, und bei der nächsten Verwendung wird ein neuer aus dem
lokalen Workspace initialisiert. Im `mirror`-Modus setzt die Neuerstellung hauptsächlich die Remote-Ausführungsumgebung
zurück, da das lokale System kanonisch bleibt.

Erstellen Sie nach Änderungen an folgenden Einstellungen neu:

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## Sicherheitshärtung

Die Dateisystem-Bridge des Mirror-Modus fixiert das lokale Workspace-Stammverzeichnis und überprüft
kanonische Pfade (über realpath) vor jedem Lesen, Schreiben, Erstellen eines Verzeichnisses, Entfernen und
Umbenennen erneut und lehnt symbolische Links innerhalb des Pfades ab. Ein Austausch eines symbolischen Links oder ein neu eingehängter Workspace
kann Dateizugriffe nicht außerhalb des gespiegelten Verzeichnisbaums umleiten.

## Aktuelle Einschränkungen

- Der Sandbox-Browser wird vom OpenShell-Backend nicht unterstützt.
- `sandbox.docker.binds` gilt nicht für OpenShell; die Sandbox-Erstellung schlägt fehl,
  wenn Bind-Mounts konfiguriert sind.
- Docker-spezifische Laufzeitoptionen unter `sandbox.docker.*` (außer `env`)
  gelten nur für das Docker-Backend.

## Funktionsweise

1. OpenClaw führt `sandbox get` für den Sandbox-Namen aus (mit allen konfigurierten
   `--gateway`/`--gateway-endpoint`); schlägt dies fehl, wird mit
   `sandbox create` eine Sandbox erstellt, wobei `--name`, `--from`, `--policy`, sofern gesetzt, `--gpu`,
   sofern aktiviert, `--auto-providers`/`--no-auto-providers` und ein
   `--provider`-Flag pro konfiguriertem Provider übergeben werden.
2. OpenClaw führt `sandbox ssh-config` für den Sandbox-Namen aus, um die SSH-
   Verbindungsdetails abzurufen.
3. Der Core schreibt die SSH-Konfiguration in eine temporäre Datei und öffnet eine SSH-Sitzung über
   dieselbe Remote-Dateisystem-Bridge wie das generische SSH-Backend.
4. Im `mirror`-Modus: vor der Ausführung lokal nach remote synchronisieren, ausführen, anschließend zurücksynchronisieren.
5. Im `remote`-Modus: bei der Erstellung einmalig initialisieren und anschließend direkt im Remote-
   Workspace arbeiten.

## Verwandte Themen

- [Sandboxing](/de/gateway/sandboxing) – Modi, Geltungsbereiche und Backend-Vergleich
- [Sandbox vs. Tool-Richtlinie vs. Erhöht](/de/gateway/sandbox-vs-tool-policy-vs-elevated) – Fehlerbehebung bei blockierten Tools
- [Multi-Agent-Sandbox und Tools](/de/tools/multi-agent-sandbox-tools) – Agent-spezifische Überschreibungen
- [Sandbox-CLI](/de/cli/sandbox) – `openclaw sandbox`-Befehle
