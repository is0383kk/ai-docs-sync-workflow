---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: 'Funktionsweise der OpenClaw-Sandbox: Modi, Geltungsbereiche, Workspace-Zugriff und Images'
title: Sandboxing
x-i18n:
    generated_at: "2026-07-26T18:23:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw kann die Werkzeugausführung in einem Sandbox-Backend ausführen, um den möglichen Schadensradius zu reduzieren. Sandboxing ist standardmäßig deaktiviert und wird durch `agents.defaults.sandbox` (global) oder `agents.entries.*.sandbox` (pro Agent) gesteuert. Der Gateway-Prozess verbleibt immer auf dem Host; nur die Werkzeugausführung wird bei Aktivierung in die Sandbox verlagert.

<Note>
Dies ist keine perfekte Sicherheitsgrenze, schränkt jedoch den Dateisystem- und Prozesszugriff erheblich ein, wenn das Modell etwas Unsinniges tut.
</Note>

## Was in der Sandbox ausgeführt wird

- Werkzeugausführung: `exec`, `read`, `write`, `edit`, `apply_patch`, `process` usw.
- Der optionale Browser in der Sandbox (`agents.defaults.sandbox.browser`).

Nicht in der Sandbox ausgeführt:

- Der Gateway-Prozess selbst.
- Jedes Werkzeug, das über `tools.elevated` ausdrücklich für die Ausführung außerhalb der Sandbox zugelassen ist. Die Ausführung mit erhöhten Rechten umgeht das Sandboxing und erfolgt über den konfigurierten Ausweichpfad (standardmäßig `gateway` oder `node`, wenn das Ausführungsziel `node` ist). Wenn Sandboxing deaktiviert ist, ändert `tools.elevated` nichts, da die Ausführung bereits auf dem Host erfolgt. Siehe [Modus mit erhöhten Rechten](/de/tools/elevated).

## Modi, Geltungsbereich und Backend

Drei voneinander unabhängige Einstellungen steuern das Sandbox-Verhalten:

| Einstellung | Schlüssel                          | Werte                        | Standard |
| ----------- | --------------------------------- | ---------------------------- | -------- |
| Modus       | `agents.defaults.sandbox.mode`    | `off`, `non-main`, `all`     | `off`    |
| Geltungsbereich | `agents.defaults.sandbox.scope`   | `agent`, `session`, `shared` | `agent`  |
| Backend     | `agents.defaults.sandbox.backend` | `docker`, `ssh`, `openshell` | `docker` |

Der **Modus** steuert, wann Sandboxing angewendet wird:

- `off`: kein Sandboxing.
- `non-main`: Jede Sitzung mit Ausnahme der Hauptsitzung des Agenten wird in einer Sandbox ausgeführt. Der Schlüssel der Hauptsitzung ist immer `agent:<agentId>:main` (oder `global`, wenn `session.scope` den Wert `"global"` hat); er ist nicht konfigurierbar. Gruppen-/Kanalsitzungen verwenden eigene Schlüssel, gelten daher immer als Nicht-Hauptsitzungen und werden in einer Sandbox ausgeführt.
- `all`: Jede Sitzung wird in einer Sandbox ausgeführt.

Der **Geltungsbereich** steuert, wie viele Container/Umgebungen erstellt werden:

- `agent`: ein Container pro Agent.
- `session`: ein Container pro Sitzung.
- `shared`: ein von allen Sandbox-Sitzungen gemeinsam genutzter Container (agentenspezifische Überschreibungen von `docker`/`ssh`/`browser` werden in diesem Geltungsbereich ignoriert).

Das **Backend** steuert, welche Laufzeitumgebung die Sandbox-Werkzeuge ausführt. SSH-spezifische Konfiguration befindet sich unter `agents.defaults.sandbox.ssh`; OpenShell-spezifische Konfiguration befindet sich unter `plugins.entries.openshell.config`.

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **Ausführungsort**  | Lokaler Container                | Jeder per SSH erreichbare Host | Von OpenShell verwaltete Sandbox                    |
| **Einrichtung**     | `scripts/sandbox-setup.sh`       | SSH-Schlüssel + Zielhost       | OpenShell-Plugin aktiviert                          |
| **Workspace-Modell** | Bind-Mount oder Kopie           | Remote-kanonisch (einmal initialisieren) | `mirror` oder `remote`                                |
| **Netzwerksteuerung** | `docker.network` (Standard: keines) | Hängt vom Remote-Host ab       | Hängt von OpenShell ab                              |
| **Browser-Sandbox** | Unterstützt                      | Nicht unterstützt              | Noch nicht unterstützt                              |
| **Bind-Mounts**     | `docker.binds`                   | Nicht zutreffend               | Nicht zutreffend                                    |
| **Am besten geeignet für** | Lokale Entwicklung, vollständige Isolation | Auslagerung auf einen Remote-Rechner | Verwaltete Remote-Sandboxes mit optionaler bidirektionaler Synchronisierung |

## Docker-Backend

Docker ist das standardmäßige Backend, sobald Sandboxing aktiviert ist. Es führt Werkzeuge und Sandbox-Browser lokal über den Socket des Docker-Daemons (`/var/run/docker.sock`) aus; die Isolation erfolgt durch Docker-Namespaces.

Standardwerte: `network: "none"` (kein ausgehender Datenverkehr), `readOnlyRoot: true`, `capDrop: ["ALL"]`, Image `openclaw-sandbox:bookworm-slim`.

Um Host-GPUs bereitzustellen, setzen Sie `agents.defaults.sandbox.docker.gpus` (oder die agentenspezifische Überschreibung) auf einen Wert wie `"all"` oder `"device=GPU-uuid"`. Dieser Wert wird an das Docker-Flag `--gpus` übergeben und erfordert eine kompatible Host-Laufzeitumgebung wie das NVIDIA Container Toolkit.

<Warning>
**Einschränkungen von Docker-out-of-Docker (DooD)**

Wenn Sie den OpenClaw Gateway selbst als Docker-Container bereitstellen, orchestriert er über den Docker-Socket des Hosts gleichgeordnete Sandbox-Container (DooD). Dadurch entsteht eine Einschränkung für die Pfadzuordnung:

- **Konfiguration erfordert Host-Pfade**: `openclaw.json` `workspace` muss den **absoluten Pfad des Hosts** enthalten (z. B. `/home/user/.openclaw/workspaces`), nicht den internen Pfad des Gateway-Containers. Der Docker-Daemon wertet Pfade relativ zum Namespace des Hostbetriebssystems aus, nicht relativ zum eigenen Namespace des Gateways.
- **Übereinstimmende Volume-Zuordnung erforderlich**: Der Gateway-Prozess schreibt außerdem Heartbeat- und Bridge-Dateien in diesen `workspace`-Pfad. Weisen Sie dem Gateway-Container eine identische Volume-Zuordnung (`-v /home/user/.openclaw:/home/user/.openclaw`) zu, damit derselbe Host-Pfad auch innerhalb des Gateway-Containers korrekt aufgelöst wird. Abweichende Zuordnungen äußern sich als `EACCES`, wenn das Gateway versucht, seinen Heartbeat zu schreiben.
- **Codex-Code-Modus**: Wenn eine OpenClaw-Sandbox aktiv ist, deaktiviert OpenClaw für diesen Durchlauf den nativen Code-Modus des Codex-App-Servers, benutzerdefinierte MCP-Server und die Ausführung App-gestützter Plugins (diese werden vom App-Server-Prozess auf dem Gateway-Host ausgeführt, nicht vom OpenClaw-Sandbox-Backend), sofern die Werkzeugrichtlinie der Sandbox nicht die erforderlichen Werkzeuge bereitstellt und Sie den experimentellen Sandbox-Exec-Server-Pfad aktivieren. Der Shell-Zugriff wird dann über OpenClaw-Werkzeuge mit Sandbox-Backend wie `sandbox_exec` und `sandbox_process` geleitet. Binden Sie den Docker-Socket des Hosts weder in Agenten-Sandbox-Container noch in benutzerdefinierte Codex-Sandboxes ein. Das vollständige Verhalten finden Sie unter [Codex-Harness](/de/plugins/codex-harness).

Auf Ubuntu-/AppArmor-Hosts mit aktiviertem Docker-Sandbox-Modus benötigt die Shell-Ausführung über `workspace-write` des Codex-App-Servers unprivilegierte Benutzer-Namespaces innerhalb des Sandbox-Containers. Sie kann bereits vor dem Start der Shell fehlschlagen, wenn der Dienstbenutzer diese nicht erstellen kann. Wenn ausgehender Docker-Sandbox-Datenverkehr deaktiviert ist (`network: "none"`, der Standardwert), wird außerdem ein unprivilegierter Netzwerk-Namespace benötigt. Häufige Symptome: `bwrap: setting up uid map: Permission denied` und `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. Führen Sie `openclaw doctor` aus. Wenn dabei ein Fehler bei der Codex-bwrap-Namespace-Prüfung gemeldet wird, verwenden Sie vorzugsweise ein AppArmor-Profil, das dem OpenClaw-Dienstprozess die erforderlichen Namespaces gewährt. `kernel.apparmor_restrict_unprivileged_userns=0` ist eine hostweite Ausweichlösung mit Sicherheitskompromissen; verwenden Sie sie nur, wenn diese Sicherheitskonfiguration für den Host akzeptabel ist.
</Warning>

### Browser in der Sandbox

- Der Sandbox-Browser wird automatisch gestartet (wodurch die Erreichbarkeit von CDP sichergestellt wird), wenn das Browser-Werkzeug ihn benötigt. Die Konfiguration erfolgt über `agents.defaults.sandbox.browser.autoStart` (Standardwert `true`) und `autoStartTimeoutMs` (Standardwert 12s).
- Sandbox-Browser-Container verwenden anstelle des globalen Netzwerks `bridge` ein dediziertes Docker-Netzwerk (`openclaw-sandbox-browser`). Die Konfiguration erfolgt über `agents.defaults.sandbox.browser.network`.
- `agents.defaults.sandbox.browser.cdpSourceRange` beschränkt den CDP-Zugriff am Container-Rand mit einer CIDR-Zulassungsliste (zum Beispiel `172.21.0.1/32`).
- Der noVNC-Beobachterzugriff ist standardmäßig passwortgeschützt. OpenClaw gibt eine kurzlebige Token-URL aus, die eine lokale Bootstrap-Seite bereitstellt und noVNC mit dem Passwort im URL-Fragment öffnet (nicht in der Abfragezeichenfolge oder in Header-Protokollen).
- `agents.defaults.sandbox.browser.allowHostControl` (Standardwert `false`) ermöglicht Sandbox-Sitzungen, ausdrücklich den Host-Browser als Ziel zu verwenden.
- Optionale Zulassungslisten beschränken `target: "custom"`: `allowedControlUrls`, `allowedControlHosts`, `allowedControlPorts`.

## SSH-Backend

Verwenden Sie `backend: "ssh"`, um `exec`, Dateiwerkzeuge und das Lesen von Medien auf einem beliebigen per SSH erreichbaren Rechner in einer Sandbox auszuführen.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // Oder verwenden Sie SecretRefs/Inline-Inhalte anstelle lokaler Dateien:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Standardwerte: `command: "ssh"`, `workspaceRoot: "/tmp/openclaw-sandboxes"`, `strictHostKeyChecking: true`, `updateHostKeys: true`.

- **Lebenszyklus**: OpenClaw erstellt unter `sandbox.ssh.workspaceRoot` ein Remote-Stammverzeichnis pro Geltungsbereich. Bei der ersten Verwendung nach dem Erstellen oder Neuerstellen wird dieser Remote-Workspace einmalig mit dem lokalen Workspace initialisiert. Danach werden `exec`, `read`, `write`, `edit`, `apply_patch`, das Lesen von Prompt-Medien und die Bereitstellung eingehender Medien über SSH direkt im Remote-Workspace ausgeführt. OpenClaw synchronisiert Remote-Änderungen nicht automatisch zurück in den lokalen Workspace.
- **Authentifizierungsmaterial**: `identityFile`/`certificateFile`/`knownHostsFile` verweisen auf vorhandene lokale Dateien. `identityData`/`certificateData`/`knownHostsData` akzeptieren Inline-Zeichenfolgen oder SecretRefs, die über den normalen Laufzeit-Snapshot für Geheimnisse aufgelöst, mit dem Modus `0600` in temporäre Dateien geschrieben und beim Ende der SSH-Sitzung gelöscht werden. Wenn für dasselbe Element sowohl eine `*File`- als auch eine `*Data`-Variante festgelegt ist, hat `*Data` für diese Sitzung Vorrang.
- **Folgen des Remote-kanonischen Modells**: Nach der anfänglichen Initialisierung wird der Remote-SSH-Workspace zum tatsächlichen Sandbox-Zustand. Nach dem Initialisierungsschritt außerhalb von OpenClaw vorgenommene lokale Host-Änderungen sind remote erst sichtbar, wenn Sie die Sandbox neu erstellen. `openclaw sandbox recreate` löscht das Remote-Stammverzeichnis des jeweiligen Geltungsbereichs und initialisiert es bei der nächsten Verwendung erneut aus der lokalen Umgebung. Browser-Sandboxing wird von diesem Backend nicht unterstützt und die Einstellungen unter `sandbox.docker.*` gelten dafür nicht.

## OpenShell-Backend

Verwenden Sie `backend: "openshell"`, um Werkzeuge in einer von OpenShell verwalteten Remote-Umgebung in einer Sandbox auszuführen. OpenShell verwendet denselben SSH-Transport und dieselbe Remote-Dateisystem-Bridge wie das generische SSH-Backend und ergänzt den OpenShell-Lebenszyklus (`sandbox create/get/delete/ssh-config`) sowie einen optionalen Modus zur Workspace-Synchronisierung über `mirror`.

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
          mode: "remote", // spiegeln | remote
        },
      },
    },
  },
}
```

`mode: "mirror"` (Standard) hält den lokalen Arbeitsbereich kanonisch: OpenClaw synchronisiert den lokalen Arbeitsbereich vor `exec` in die Sandbox und danach zurück. `mode: "remote"` befüllt den Remote-Arbeitsbereich einmalig aus dem lokalen Arbeitsbereich und führt anschließend `exec`/`read`/`write`/`edit`/`apply_patch` direkt im Remote-Arbeitsbereich aus, ohne zurückzusynchronisieren; lokale Änderungen nach der initialen Befüllung bleiben unsichtbar, bis Sie `openclaw sandbox recreate`. Unter `scope: "agent"` oder `scope: "shared"` wird dieser Remote-Arbeitsbereich im selben Geltungsbereich gemeinsam genutzt. Aktuelle Einschränkungen: Der Sandbox-Browser wird noch nicht unterstützt, und `sandbox.docker.binds` gilt nicht für dieses Backend.

`openclaw sandbox list`/`recreate`/prune behandeln OpenShell-Laufzeitumgebungen genauso wie Docker-Laufzeitumgebungen; die Bereinigungslogik berücksichtigt das Backend.

Die vollständigen Voraussetzungen, die Konfigurationsreferenz, den Vergleich der Arbeitsbereichsmodi und Details zum Lebenszyklus finden Sie unter [OpenShell](/de/gateway/openshell).

## Zugriff auf den Arbeitsbereich

`agents.defaults.sandbox.workspaceAccess` steuert, was die Sandbox sehen kann:

| Wert            | Verhalten                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none` (Standard) | Tools sehen einen isolierten Sandbox-Arbeitsbereich unter `~/.openclaw/sandboxes`.                    |
| `ro`             | Bindet den Agenten-Arbeitsbereich schreibgeschützt unter `/agent` ein (deaktiviert `write`/`edit`/`apply_patch`). |
| `rw`             | Bindet den Agenten-Arbeitsbereich mit Lese-/Schreibzugriff unter `/workspace` ein.                                    |

Mit dem OpenShell-Backend verwendet der Modus `mirror` weiterhin den lokalen Arbeitsbereich als kanonische Quelle zwischen Ausführungen, der Modus `remote` verwendet nach der initialen Befüllung den Remote-OpenShell-Arbeitsbereich als kanonische Quelle, und `workspaceAccess: "ro"`/`"none"` beschränken das Schreibverhalten weiterhin auf dieselbe Weise.

Eingehende Medien werden in den aktiven Sandbox-Arbeitsbereich kopiert (`media/inbound/*`).

<Note>
**Skills**: Das Tool `read` ist in der Sandbox verwurzelt. Mit `workspaceAccess: "none"` spiegelt OpenClaw geeignete Skills in den Sandbox-Arbeitsbereich (`.../skills`), damit sie gelesen werden können. Mit `"rw"` sind Arbeitsbereichs-Skills unter `/workspace/skills` lesbar, und geeignete verwaltete, gebündelte oder Plugin-Skills werden im generierten schreibgeschützten Pfad `/workspace/.openclaw/sandbox-skills/skills` bereitgestellt.
</Note>

## Mehrere Ordner für einen Agenten

Verwenden Sie Docker-Bind-Mounts, wenn ein in einer Sandbox ausgeführter Agent mehr als seinen primären Arbeitsbereich benötigt. Jeder Eintrag ordnet einen Host-Ordner mit einem expliziten Zugriffsmodus einem Container-Pfad zu:

```text
host-directory:container-directory:ro
host-directory:container-directory:rw
```

- `ro` macht den eingebundenen Ordner innerhalb der Sandbox schreibgeschützt.
- `rw` erlaubt Tools und Prozessen in der Sandbox, den Host-Ordner zu ändern.
- Der Container-Pfad ist der Pfad, den der Agent verwendet. Host-Pfade werden nicht automatisch offengelegt.

Dieses Beispiel gibt dem Agenten `research` einen beschreibbaren primären Arbeitsbereich, schreibgeschütztes Referenzmaterial unter `/reference` und einen separaten beschreibbaren Ausgabeordner unter `/drafts`:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // Erforderlich, da sich diese Quellen außerhalb des Agenten-Arbeitsbereichs befinden.
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` und Bind-Modi sind unabhängig voneinander:

| Einstellung                          | Steuert                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | Verwendet einen isolierten Sandbox-Arbeitsbereich; legt den Agenten-Arbeitsbereich nicht offen.    |
| `workspaceAccess: "ro"`          | Bindet den Agenten-Arbeitsbereich schreibgeschützt unter `/agent` ein.                           |
| `workspaceAccess: "rw"`          | Bindet den Agenten-Arbeitsbereich mit Lese-/Schreibzugriff unter `/workspace` ein.                      |
| `docker.binds`-Eintrag `:ro`/`:rw` | Steuert nur diesen zusätzlichen Host-Ordner an seinem konfigurierten Container-Pfad. |

Das Ändern von `workspaceAccess` ändert eine zusätzliche Bind-Einbindung nicht von `ro` zu `rw` oder umgekehrt. Globale und agentenspezifische `docker.binds` werden zusammengeführt. Behalten Sie `scope: "agent"` oder `"session"` für agentenspezifische Bind-Einbindungen bei; `scope: "shared"` ignoriert alle agentenspezifischen Docker-Überschreibungen und verwendet ausschließlich globale Bind-Einbindungen.

Bind-Mounts sind die unterstützte Grenze für mehrere Ordner, da Docker die Dateisystemansicht des Containers mit Mount-Isolierung erstellt und der Modus `ro`/`rw` für jeden Prozess in der Sandbox gilt. Diese Grenze umfasst `exec`, Dateisystem-Tools, untergeordnete Prozesse und Bibliotheken, ohne Pfadautorisierungsprüfungen in jedem OpenClaw-Codepfad zu duplizieren. Eine hostseitige Pfad-Zulassungsliste kann nicht dieselbe vollständige Grenze bieten, wenn eine zugelassene Shell oder Abhängigkeit direkt auf Dateien zugreifen kann.

Die optionale Einstellung `dangerouslyAllowExternalBindSources` erlaubt nur Quellen außerhalb der Arbeitsbereichswurzeln. Sie deaktiviert nicht die Prüfungen von OpenClaw auf blockierte Systempfade, Anmeldedaten, Docker-Sockets, Symlink-Elternpfade oder reservierte Ziele. Bevorzugen Sie den kleinstmöglichen Ordner, verwenden Sie `ro`, sofern kein Schreibzugriff erforderlich ist, und erstellen Sie die Sandbox nach Änderungen an den Einbindungen neu:

```bash
openclaw sandbox recreate --agent research
```

### Weiteres Verhalten von Bind-Einbindungen

`agents.defaults.sandbox.docker.binds` konfiguriert globale Einbindungen. Das Format entspricht derselben Form `host:container:mode` (zum Beispiel `"/home/user/source:/source:rw"`).

`agents.defaults.sandbox.browser.binds` bindet zusätzliche Host-Verzeichnisse ausschließlich in den Container des **Sandbox-Browsers** ein. Wenn diese Einstellung gesetzt ist (einschließlich `[]`), ersetzt sie `docker.binds` für den Browser-Container; wenn sie nicht gesetzt ist, greift der Browser-Container auf `docker.binds` zurück.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**Sicherheit von Bind-Einbindungen**

- Bind-Einbindungen umgehen das Sandbox-Dateisystem: Sie legen Host-Pfade mit dem von Ihnen festgelegten Modus (`:ro` oder `:rw`) offen.
- OpenClaw blockiert gefährliche Bind-Quellen standardmäßig: Systempfade (`/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/boot`), Docker-Socket-Verzeichnisse (`/run`, `/var/run` und ihre `docker.sock`-Varianten) sowie gängige Stammverzeichnisse für Anmeldedaten in Benutzerverzeichnissen (`~/.aws`, `~/.cargo`, `~/.config`, `~/.docker`, `~/.gnupg`, `~/.netrc`, `~/.npm`, `~/.ssh`).
- Die Validierung normalisiert den Quellpfad und löst ihn anschließend über den tiefsten vorhandenen Vorfahren erneut auf, bevor blockierte Pfade und zulässige Wurzeln erneut geprüft werden. Dadurch werden Ausbrüche über Symlink-Elternpfade sicher abgelehnt, selbst wenn das letzte Pfadelement noch nicht existiert (beispielsweise wird `/workspace/run-link/new-file` weiterhin als `/var/run/...` aufgelöst, wenn `run-link` dorthin verweist).
- Bind-Ziele, die die reservierten Container-Einhängepunkte (`/workspace`, `/agent`) überlagern, werden ebenfalls standardmäßig blockiert; überschreiben Sie dies mit `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true`.
- Bind-Quellen außerhalb der zugelassenen Wurzeln des Arbeitsbereichs beziehungsweise Agenten-Arbeitsbereichs werden standardmäßig blockiert; überschreiben Sie dies mit `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true`. Zulässige Wurzeln werden auf dieselbe Weise kanonisiert. Daher wird ein Pfad, der nur vor der Symlink-Auflösung innerhalb der Zulassungsliste zu liegen scheint, weiterhin als außerhalb der zulässigen Wurzeln abgelehnt.
- Sensible Einbindungen (Geheimnisse, SSH-Schlüssel, Dienstanmeldedaten) sollten `:ro` sein, sofern nicht unbedingt etwas anderes erforderlich ist.
- Kombinieren Sie dies mit `workspaceAccess: "ro"`, wenn Sie nur Lesezugriff auf den Arbeitsbereich benötigen; die Bind-Modi bleiben unabhängig.
- Unter [Sandbox vs. Tool-Richtlinie vs. erhöhte Rechte](/de/gateway/sandbox-vs-tool-policy-vs-elevated) erfahren Sie, wie Bind-Einbindungen mit Tool-Richtlinien und Ausführungen mit erhöhten Rechten interagieren.

</Warning>

## Images und Einrichtung

Standardmäßiges Docker-Image: `openclaw-sandbox:bookworm-slim`

<Note>
**Quellcode-Checkout im Vergleich zur npm-Installation**

Die Hilfsskripte `scripts/sandbox-setup.sh`, `scripts/sandbox-common-setup.sh` und `scripts/sandbox-browser-setup.sh` sind nur bei der Ausführung aus einem [Quellcode-Checkout](https://github.com/openclaw/openclaw) verfügbar. Sie sind nicht im npm-Paket enthalten.

Wenn Sie OpenClaw über `npm install -g openclaw` installiert haben, verwenden Sie stattdessen die unten gezeigten Inline-Befehle für `docker build`.
</Note>

<Steps>
  <Step title="Standard-Image erstellen">
    Aus einem Quellcode-Checkout:

    ```bash
    scripts/sandbox-setup.sh
    ```

    Aus einer npm-Installation (kein Quellcode-Checkout erforderlich):

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    Das Standard-Image enthält **kein** Node. Wenn ein Skill Node (oder andere Laufzeitumgebungen) benötigt, erstellen Sie entweder ein benutzerdefiniertes Image oder installieren Sie es über `sandbox.docker.setupCommand` (erfordert ausgehenden Netzwerkzugriff, eine beschreibbare Root-Dateisystemebene und den Root-Benutzer).

    OpenClaw ersetzt fehlendes `openclaw-sandbox:bookworm-slim` nicht stillschweigend durch einfaches `debian:bookworm-slim`. Sandbox-Ausführungen, die auf das Standard-Image abzielen, schlagen frühzeitig mit einer Erstellungsanweisung fehl, bis Sie es erstellt haben, da das gebündelte Image `python3` für die Schreib- und Bearbeitungshilfen der Sandbox enthält.

  </Step>
  <Step title="Optional: Allgemeines Image erstellen">
    Für ein funktionsreicheres Sandbox-Image mit gängigen Tools (zum Beispiel `curl`, `jq`, Node 24, pnpm, `python3` und `git`):

    Aus einem Quellcode-Checkout:

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    Erstellen Sie bei einer npm-Installation zunächst das Standard-Image (siehe oben) und anschließend das allgemeine Image darauf aufbauend mithilfe von [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common) aus dem Repository.

    Setzen Sie anschließend `agents.defaults.sandbox.docker.image` auf `openclaw-sandbox-common:bookworm-slim`.

  </Step>
  <Step title="Optional: Image für den Sandbox-Browser erstellen">
    Aus einem Quellcode-Checkout:

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    Erstellen Sie es bei einer npm-Installation mithilfe von [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) aus dem Repository.

  </Step>
</Steps>

Standardmäßig werden Docker-Sandbox-Container **ohne Netzwerk** ausgeführt. Überschreiben Sie dies mit `agents.defaults.sandbox.docker.network`.

<AccordionGroup>
  <Accordion title="Chromium-Standardeinstellungen des Sandbox-Browsers">
    Das gebündelte Sandbox-Browser-Image verwendet konservative Chromium-Startoptionen für containerisierte Arbeitslasten:

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - `--headless=new`, wenn `browser.headless` aktiviert ist.
    - `--no-sandbox --disable-setuid-sandbox`, wenn `browser.noSandbox` aktiviert ist.
    - `--disable-3d-apis`, `--disable-gpu`, `--disable-software-rasterizer` standardmäßig; diese Flags zur Härtung der Grafikausgabe unterstützen Container ohne GPU-Unterstützung. Legen Sie `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` fest, wenn Ihre Arbeitslast WebGL oder andere 3D-Funktionen benötigt.
    - `--disable-extensions` standardmäßig; legen Sie für Abläufe, die Erweiterungen benötigen, `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` fest.
    - `--renderer-process-limit=2` standardmäßig; gesteuert durch `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`, wobei `0` den Standardwert von Chromium beibehält.

    Wenn Sie ein anderes Laufzeitprofil benötigen, verwenden Sie ein benutzerdefiniertes Browser-Image und stellen Sie einen eigenen Einstiegspunkt bereit. Verwenden Sie für lokale Chromium-Profile (außerhalb von Containern) `browser.extraArgs`, um zusätzliche Start-Flags anzuhängen.

  </Accordion>
  <Accordion title="Standardeinstellungen für die Netzwerksicherheit">
    - `network: "host"` ist blockiert.
    - `network: "container:<id>"` ist standardmäßig blockiert (Risiko einer Umgehung durch Beitritt zum Namespace).
    - Notfallübersteuerung: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

  </Accordion>
</AccordionGroup>

Docker-Installationen und das containerisierte Gateway finden Sie hier: [Docker](/de/install/docker)

Bei Docker-Gateway-Bereitstellungen kann `scripts/docker/setup.sh` die Sandbox-Konfiguration initialisieren. Legen Sie `OPENCLAW_SANDBOX=1` (oder `true`/`yes`/`on`) fest, um diesen Pfad zu aktivieren. Überschreiben Sie den Speicherort des Sockets mit `OPENCLAW_DOCKER_SOCKET`. Vollständige Einrichtung und Umgebungsvariablenreferenz: [Docker](/de/install/docker#agent-sandbox).

## setupCommand (einmalige Container-Einrichtung)

`setupCommand` wird **einmal** ausgeführt, nachdem der Sandbox-Container erstellt wurde (nicht bei jeder Ausführung). Der Befehl wird innerhalb des Containers über `sh -lc` ausgeführt.

Pfade:

- Global: `agents.defaults.sandbox.docker.setupCommand`
- Pro Agent: `agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="Häufige Fallstricke">
    - Der Standardwert von `docker.network` ist `"none"` (kein ausgehender Netzwerkverkehr), daher schlagen Paketinstallationen fehl.
    - `docker.network: "container:<id>"` erfordert `dangerouslyAllowContainerNamespaceJoin: true` und ist nur für Notfälle vorgesehen.
    - `readOnlyRoot: true` verhindert Schreibvorgänge; legen Sie `readOnlyRoot: false` fest oder erstellen Sie ein benutzerdefiniertes Image.
    - `user` muss für Paketinstallationen root sein (lassen Sie `user` weg oder legen Sie `user: "0:0"` fest).
    - Die Sandbox-Ausführung übernimmt `process.env` des Hosts **nicht**. Verwenden Sie `agents.defaults.sandbox.docker.env` (oder ein benutzerdefiniertes Image) für API-Schlüssel von Skills.
    - Werte in `agents.defaults.sandbox.docker.env` werden als explizite Umgebungsvariablen des Docker-Containers übergeben. Jeder mit Zugriff auf den Docker-Daemon kann sie mit Docker-Metadatenbefehlen wie `docker inspect` einsehen. Verwenden Sie ein benutzerdefiniertes Image, eine eingebundene Secret-Datei oder einen anderen Übermittlungsweg für Secrets, wenn diese Offenlegung über Metadaten nicht akzeptabel ist.

  </Accordion>
</AccordionGroup>

## Tool-Richtlinien und Ausweichmöglichkeiten

Zulassungs-/Ablehnungsrichtlinien für Tools gelten weiterhin vor den Sandbox-Regeln. Wenn ein Tool global oder für einen Agenten abgelehnt wird, stellt die Sandbox es nicht wieder bereit.

`tools.elevated` ist eine explizite Ausweichmöglichkeit, die `exec` außerhalb der Sandbox ausführt (standardmäßig `gateway` oder `node`, wenn das Ausführungsziel `node` ist). `/exec`-Direktiven gelten nur für autorisierte Absender und bleiben sitzungsbezogen bestehen; um `exec` vollständig zu deaktivieren, verwenden Sie die Ablehnungsrichtlinie für Tools (siehe [Sandbox im Vergleich zu Tool-Richtlinien und erhöhten Berechtigungen](/de/gateway/sandbox-vs-tool-policy-vs-elevated)).

Fehlerdiagnose:

- `openclaw sandbox list` zeigt Sandbox-Container, Status, Image-Übereinstimmung, Alter, Leerlaufzeit und die zugehörige Sitzung bzw. den zugehörigen Agenten an.
- `openclaw sandbox explain [--session <key>] [--agent <id>]` prüft den effektiven Sandbox-Modus, den Host-Arbeitsbereich, das Laufzeit-Arbeitsverzeichnis, Docker-Einbindungen, Tool-Richtlinien und Konfigurationsschlüssel zur Problembehebung. Das Feld `workspaceRoot` enthält weiterhin das konfigurierte Sandbox-Stammverzeichnis; `effectiveHostWorkspaceRoot` zeigt, wo sich der aktive Arbeitsbereich tatsächlich befindet.
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]` entfernt Container/Umgebungen, damit sie bei der nächsten Verwendung mit der aktuellen Konfiguration neu erstellt werden.
- Unter [Sandbox im Vergleich zu Tool-Richtlinien und erhöhten Berechtigungen](/de/gateway/sandbox-vs-tool-policy-vs-elevated) finden Sie ein Denkmodell für die Frage „Warum ist dies blockiert?“.

## Überschreibungen für mehrere Agenten

Jeder Agent kann Sandbox und Tools überschreiben: `agents.entries.*.sandbox` und `agents.entries.*.tools` (sowie `agents.entries.*.tools.sandbox.tools` für die Sandbox-Tool-Richtlinie). Informationen zur Rangfolge finden Sie unter [Sandbox und Tools für mehrere Agenten](/de/tools/multi-agent-sandbox-tools).

## Minimales Aktivierungsbeispiel

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## Verwandte Themen

- [Sandbox und Tools für mehrere Agenten](/de/tools/multi-agent-sandbox-tools) -- Überschreibungen pro Agent und Rangfolge
- [OpenShell](/de/gateway/openshell) -- Einrichtung des verwalteten Sandbox-Backends, Arbeitsbereichsmodi und Konfigurationsreferenz
- [Sandbox-Konfiguration](/de/gateway/config-agents#agentsdefaultssandbox)
- [Sandbox im Vergleich zu Tool-Richtlinien und erhöhten Berechtigungen](/de/gateway/sandbox-vs-tool-policy-vs-elevated) -- Fehlerdiagnose für „Warum ist dies blockiert?“
- [Sicherheit](/de/gateway/security)
