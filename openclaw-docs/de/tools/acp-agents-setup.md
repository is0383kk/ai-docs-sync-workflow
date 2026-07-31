---
read_when:
    - Installieren oder Konfigurieren des acpx-Harness für Claude Code / Codex / Gemini CLI
    - Aktivieren der MCP-Bridge für plugin-tools oder OpenClaw-tools
    - ACP-Berechtigungsmodi konfigurieren
summary: 'ACP-Agenten einrichten: acpx-Harness-Konfiguration, Plugin-Einrichtung, Berechtigungen'
title: ACP-Agenten — Einrichtung
x-i18n:
    generated_at: "2026-07-26T19:15:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae3750092175b44252dd080717a1af176995df43c653f245f82d7e556cfd25eb
    source_path: tools/acp-agents-setup.md
    workflow: 16
---

Eine Übersicht, das Operator-Runbook und die Konzepte finden Sie unter [ACP-Agenten](/de/tools/acp-agents).

Diese Seite behandelt die acpx-Harness-Konfiguration, die Plugin-Einrichtung für die MCP-Bridges und die Berechtigungskonfiguration.

Verwenden Sie diese Seite nur, wenn Sie die ACP/acpx-Route einrichten. Informationen zur nativen Laufzeitkonfiguration des Codex-
App-Servers finden Sie unter [Codex-Harness](/de/plugins/codex-harness). Informationen zu
OpenAI-API-Schlüsseln oder zur Modell-Provider-Konfiguration für Codex OAuth finden Sie unter
[OpenAI](/de/providers/openai).

Codex bietet zwei OpenClaw-Routen:

| Route                       | Konfiguration/Befehl                                   | Einrichtungsseite                         |
| --------------------------- | ------------------------------------------------------ | ----------------------------------------- |
| Nativer Codex-App-Server    | `/codex ...`, `openai/gpt-*`-Agent-Referenzen | [Codex-Harness](/de/plugins/codex-harness)   |
| Expliziter Codex-ACP-Adapter | `/acp spawn codex`, `runtime: "acp", agentId: "codex"` | Diese Seite                               |

Bevorzugen Sie die native Route, sofern Sie nicht ausdrücklich das Verhalten von ACP/acpx benötigen.

## Unterstützung durch das acpx-Harness (aktuell)

Integrierte acpx-Harness-Aliasse (aus der angehefteten Abhängigkeit `acpx`):

| Alias        | Umschließt                                                                                                      |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| `claude`     | [Claude Code](https://claude.ai/code)                                                                           |
| `codex`      | [Codex CLI](https://codex.openai.com)                                                                           |
| `copilot`    | [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/copilot-chat/use-copilot-chat-in-the-command-line) |
| `cursor`     | [Cursor CLI](https://cursor.com/docs/cli/acp) (`cursor-agent acp`)                                              |
| `droid`      | [Factory Droid](https://www.factory.ai)                                                                         |
| `fast-agent` | [fast-agent](https://fast-agent.ai)                                                                             |
| `gemini`     | [Gemini CLI](https://github.com/google/gemini-cli)                                                              |
| `iflow`      | [iFlow CLI](https://github.com/iflow-ai/iflow-cli)                                                              |
| `kilocode`   | [Kilocode](https://kilocode.ai)                                                                                 |
| `kimi`       | [Kimi CLI](https://github.com/MoonshotAI/kimi-cli)                                                              |
| `kiro`       | [Kiro CLI](https://kiro.dev)                                                                                    |
| `mux`        | [Mux](https://mux.coder.com)                                                                                    |
| `opencode`   | [OpenCode](https://opencode.ai)                                                                                 |
| `openclaw`   | OpenClaw-ACP-Bridge (nativ `openclaw acp`)                                                                  |
| `pi`         | [Pi Coding Agent](https://github.com/mariozechner/pi)                                                           |
| `qoder`      | [Qoder CLI](https://docs.qoder.com/cli/acp)                                                                     |
| `qwen`       | [Qwen Code](https://github.com/QwenLM/qwen-code)                                                                |
| `trae`       | [Trae CLI](https://docs.trae.cn/cli)                                                                            |

`factory-droid` und `factorydroid` werden ebenfalls zum integrierten Adapter `droid` aufgelöst.

Wenn OpenClaw das acpx-Backend verwendet, bevorzugen Sie diese Werte für `agentId`, sofern Ihre acpx-Konfiguration keine benutzerdefinierten Agent-Aliasse definiert.
Falls Ihre lokale Cursor-Installation ACP weiterhin als `agent acp` bereitstellt, überschreiben Sie den Agent-Befehl `cursor` in Ihrer acpx-Konfiguration, anstatt den integrierten Standardwert zu ändern.

Bei der direkten Verwendung der acpx-CLI können über `--agent <command>` auch beliebige Adapter angesprochen werden. Dieser rohe Ausweichmechanismus ist jedoch eine Funktion der acpx-CLI und nicht der normale OpenClaw-Pfad `agentId`.

Die Modellsteuerung hängt von den Fähigkeiten des Adapters ab. Codex-ACP-Modellreferenzen werden
vor dem Start durch OpenClaw normalisiert. Andere Harnesses benötigen ACP-Unterstützung für `models` sowie
`session/set_model`. Wenn ein Harness weder diese ACP-Fähigkeit
noch ein eigenes Modell-Flag für den Start bereitstellt, können OpenClaw/acpx keine Modellauswahl erzwingen.

## Erforderliche Konfiguration

ACP-Basiskonfiguration des Kerns:

```json5
{
  acp: {
    enabled: true,
    // Optional. Der Standardwert ist true; setzen Sie ihn auf false, um die ACP-Weiterleitung zu pausieren und die /acp-Steuerelemente beizubehalten.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "qwen",
    ],
    stream: {
      deliveryMode: "live",
    },
  },
}
```

Die Konfiguration der Thread-Bindung wird von den unterstützten Kanaladaptern gemeinsam verwendet:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
    },
  },
}
```

Wenn das Thread-gebundene Starten von ACP nicht funktioniert, prüfen Sie zuerst das Funktions-Flag des Adapters:

- Discord: `session.threadBindings.spawnSessions=true`

Bindungen an die aktuelle Konversation erfordern keine Erstellung eines untergeordneten Threads. Sie erfordern einen aktiven Konversationskontext und einen Kanaladapter, der ACP-Konversationsbindungen bereitstellt.

Siehe [Konfigurationsreferenz](/de/gateway/configuration-reference).

## Plugin-Einrichtung für das acpx-Backend

Paketierte Installationen verwenden das offizielle Laufzeit-Plugin `@openclaw/acpx` für ACP.
Installieren und aktivieren Sie es, bevor Sie ACP-Harness-Sitzungen verwenden:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Quellcode-Checkouts können nach `pnpm install` auch das lokale Workspace-Plugin verwenden.

Beginnen Sie mit:

```text
/acp doctor
```

Wenn Sie `acpx` deaktiviert, es über `plugins.allow` / `plugins.deny` verweigert haben oder
zum paketierten Plugin zurückwechseln möchten, verwenden Sie den expliziten Paketpfad:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Lokale Workspace-Installation während der Entwicklung:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

Prüfen Sie anschließend den Zustand des Backends:

```text
/acp doctor
```

### Startprüfung der acpx-Laufzeit

Das Plugin `acpx` bettet die ACP-Laufzeit direkt ein (keine separate ausführbare Datei `acpx` und
keine zu konfigurierende Version). Standardmäßig registriert es das eingebettete Backend während
des Gateway-Starts und wartet vor dem Gateway-Signal `ready` auf eine Startprüfung.
Setzen Sie `OPENCLAW_ACPX_RUNTIME_STARTUP_PROBE=0` oder
`OPENCLAW_SKIP_ACPX_RUNTIME_PROBE=1` nur für Skripte oder Umgebungen,
in denen die Startprüfung absichtlich deaktiviert bleiben soll. Führen Sie `/acp doctor` für eine explizite
Prüfung bei Bedarf aus.

Überschreiben Sie den Befehl eines einzelnen ACP-Agenten mit strukturierten Argumenten, wenn ein Pfad
oder Flag-Wert als einzelnes argv-Token erhalten bleiben soll:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "agents": {
            "claude": {
              "command": "node",
              "args": ["/path/to/custom adapter.mjs", "--verbose"]
            }
          }
        }
      }
    }
  }
}
```

- `agents.<id>.command` ist die ausführbare Datei oder die vorhandene Befehlszeichenfolge für diesen ACP-Agenten.
- `agents.<id>.args` ist optional. Jedes Array-Element wird für die Shell in Anführungszeichen gesetzt, bevor OpenClaw es über die aktuelle acpx-Registrierung für Befehlszeichenfolgen weitergibt.

Siehe [Plugins](/de/tools/plugin).

### Automatischer Adapter-Download

`acpx` lädt ACP-Adapter (beispielsweise die ACP-Bridges für Claude und Codex)
bei der ersten Verwendung über `npx` automatisch herunter. Sie müssen Adapterpakete nicht
manuell installieren, und für OpenClaw selbst gibt es keinen separaten Postinstallationsschritt. Wenn ein
Adapter-Download oder -Start fehlschlägt, meldet `/acp doctor` den Fehler.

### MCP-Bridge für Plugin-Tools

Standardmäßig stellen ACPX-Sitzungen dem ACP-Harness **keine** von OpenClaw-Plugins registrierten Tools
zur Verfügung.

Wenn ACP-Agenten wie Codex oder Claude Code installierte
OpenClaw-Plugin-Tools wie das Abrufen/Speichern von Erinnerungen aufrufen können sollen, aktivieren Sie die dedizierte Bridge:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

Funktionsweise:

- Fügt beim Starten der ACPX-Sitzung einen integrierten MCP-Server namens `openclaw-plugin-tools`
  ein.
- Stellt Plugin-Tools bereit, die bereits von installierten und aktivierten OpenClaw-
  Plugins registriert wurden.
- Übergibt die Identität der aktiven ACP-Sitzung an die Plugin-Tool-Factories, sodass
  Agent-spezifische Tools im Namespace dieses Agenten verbleiben.
- Hält die Funktion explizit und standardmäßig deaktiviert.

Hinweise zu Sicherheit und Vertrauen:

- Dies erweitert die Tool-Oberfläche des ACP-Harnesses.
- ACP-Agenten erhalten nur Zugriff auf Plugin-Tools, die bereits im Gateway aktiv sind.
- Behandeln Sie dies als dieselbe Vertrauensgrenze, die gilt, wenn Sie diesen Plugins die Ausführung
  in OpenClaw selbst gestatten.
- Prüfen Sie die installierten Plugins, bevor Sie die Funktion aktivieren.

Benutzerdefinierte `mcpServers` funktionieren weiterhin wie zuvor. Die integrierte Plugin-Tools-Bridge ist eine
zusätzliche optionale Komfortfunktion und kein Ersatz für die generische MCP-Server-Konfiguration.

### MCP-Bridge für OpenClaw-Tools

Standardmäßig stellen ACPX-Sitzungen integrierte OpenClaw-Tools ebenfalls **nicht** über
MCP bereit. Aktivieren Sie die separate Bridge für Kern-Tools, wenn ein ACP-Agent ausgewählte
integrierte Tools wie `cron` benötigt:

```bash
openclaw config set plugins.entries.acpx.config.openClawToolsMcpBridge true
```

Funktionsweise:

- Fügt beim Starten der ACPX-Sitzung einen integrierten MCP-Server namens `openclaw-tools`
  ein.
- Stellt ausgewählte integrierte OpenClaw-Tools bereit. Der anfängliche Server stellt `cron` bereit.
- Hält die Bereitstellung von Kern-Tools explizit und standardmäßig deaktiviert.

### Konfiguration des Zeitlimits für Laufzeitoperationen

Das Plugin `acpx` gewährt Start- und Steuerungsoperationen der eingebetteten Laufzeit standardmäßig 120
Sekunden. Dadurch erhalten langsamere Harnesses wie Gemini CLI genügend Zeit,
um den ACP-Start und die Initialisierung abzuschließen. Überschreiben Sie den Wert, wenn Ihr Host ein
anderes Operationslimit benötigt:

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

Laufzeit-Turns verwenden die Zeitlimits für Agenten/Ausführungen von OpenClaw, einschließlich `/acp timeout`.
`sessions_spawn` akzeptiert keine Zeitlimitüberschreibungen pro Aufruf; der Operatorpfad
ist `agents.defaults.subagents.runTimeoutSeconds`. Starten Sie das Gateway neu, nachdem Sie
`timeoutSeconds` geändert haben.

### Konfiguration des Agenten für Zustandsprüfungen

Wenn `/acp doctor` oder die Startprüfung das Backend prüft, testet das gebündelte Plugin `acpx`
einen Harness-Agenten. Wenn `acp.allowedAgents` gesetzt ist, wird standardmäßig
der erste zulässige Agent verwendet; andernfalls ist der Standardwert `codex`. Wenn Ihre Bereitstellung
einen anderen ACP-Agenten für Zustandsprüfungen benötigt, legen Sie den Prüf-Agenten ausdrücklich fest:

```bash
openclaw config set plugins.entries.acpx.config.probeAgent claude
```

Starten Sie das Gateway neu, nachdem Sie diesen Wert geändert haben.

## Berechtigungskonfiguration

ACP-Sitzungen werden nicht interaktiv ausgeführt – es gibt kein TTY, über das Berechtigungsaufforderungen für Dateischreibvorgänge und Shell-Ausführungen genehmigt oder abgelehnt werden können. Das acpx-Plugin stellt zwei Konfigurationsschlüssel bereit, die steuern, wie Berechtigungen behandelt werden:

Diese ACPX-Harness-Berechtigungen sind von OpenClaw-Exec-Genehmigungen und von Umgehungs-Flags der CLI-Backend-Anbieter wie Claude CLI `--permission-mode bypassPermissions` getrennt. ACPX `approve-all` ist der Break-Glass-Schalter auf Harness-Ebene für ACP-Sitzungen.

Einen umfassenderen Vergleich zwischen OpenClaw `tools.exec.mode`, Codex-Guardian-Genehmigungen
und ACPX-Harness-Berechtigungen finden Sie unter
[Berechtigungsmodi](/de/tools/permission-modes).

### `permissionMode`

Steuert, welche Vorgänge der Harness-Agent ohne Rückfrage ausführen kann.

| Wert            | Verhalten                                                               |
| --------------- | ----------------------------------------------------------------------- |
| `approve-all`   | Alle Dateischreibvorgänge und Shell-Befehle automatisch genehmigen.     |
| `approve-reads` | Nur Lesevorgänge automatisch genehmigen; Schreib- und Exec-Vorgänge erfordern Rückfragen. |
| `deny-all`      | Alle Berechtigungsanfragen ablehnen.                                    |

### `nonInteractivePermissions`

Steuert, was geschieht, wenn eine Berechtigungsabfrage angezeigt werden müsste, aber kein interaktives TTY verfügbar ist (was bei ACP-Sitzungen immer der Fall ist).

| Wert   | Verhalten                                                                 |
| ------ | ------------------------------------------------------------------------- |
| `fail` | Sitzung mit `PermissionPromptUnavailableError` abbrechen. **(Standard)** |
| `deny` | Berechtigung ohne Meldung ablehnen und fortfahren (kontrollierte Beeinträchtigung). |

### Konfiguration

Über die Plugin-Konfiguration festlegen:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

Starten Sie das Gateway nach dem Ändern dieser Werte neu.

<Warning>
OpenClaw verwendet standardmäßig `permissionMode=approve-reads` und `nonInteractivePermissions=fail`. In nicht interaktiven ACP-Sitzungen können Schreib- oder Exec-Vorgänge, die eine Berechtigungsabfrage auslösen, mit `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` fehlschlagen.

Wenn Sie Berechtigungen einschränken müssen, setzen Sie `nonInteractivePermissions` auf `deny`, damit Sitzungen kontrolliert weiterlaufen, statt abzustürzen.
</Warning>

## Verwandte Themen

- [ACP-Agenten](/de/tools/acp-agents) — Übersicht, Betriebshandbuch, Konzepte
- [Sub-Agenten](/de/tools/subagents)
- [Multi-Agent-Routing](/de/concepts/multi-agent)
