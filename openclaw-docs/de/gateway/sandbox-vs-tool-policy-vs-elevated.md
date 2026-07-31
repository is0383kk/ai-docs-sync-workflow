---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'Warum ein Tool blockiert ist: Sandbox-Laufzeit, Richtlinie zum Zulassen/Ablehnen von Tools und Freigaben für die Ausführung mit erhöhten Berechtigungen'
title: Sandbox vs. Tool-Richtlinie vs. erhöhte Berechtigungen
x-i18n:
    generated_at: "2026-07-26T17:50:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4da521215fe55bf2774008a53d896d5c00b8babcbca2005dc4593ebfebc5343
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 16
---

OpenClaw verfügt über drei miteinander verbundene, aber unterschiedliche Steuerungsmechanismen:

1. **Sandbox** (`agents.defaults.sandbox.*` / `agents.entries.*.sandbox.*`) bestimmt, **wo Tools ausgeführt werden** (Sandbox-Backend oder Host).
2. **Tool-Richtlinie** (`tools.*`, `tools.sandbox.tools.*`, `agents.entries.*.tools.*`) bestimmt, **welche Tools verfügbar/zulässig sind**.
3. **Elevated** (`tools.elevated.*`, `agents.entries.*.tools.elevated.*`) ist ein **ausschließlich für die Ausführung vorgesehener Ausweg**, um außerhalb der Sandbox auszuführen, wenn Sie sich in einer Sandbox befinden (standardmäßig `gateway` oder `node`, wenn das Ausführungsziel als `node` konfiguriert ist).

## Schnelle Fehlerdiagnose

Verwenden Sie den Inspektor, um zu sehen, was OpenClaw _tatsächlich_ tut:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

Er gibt Folgendes aus:

- effektiver Sandbox-Modus/-Geltungsbereich/-Workspace-Zugriff
- ob sich die Sitzung derzeit in einer Sandbox befindet (Haupt- oder Nicht-Hauptsitzung)
- effektive Zulassungs-/Ablehnungsregeln für Sandbox-Tools (und ob sie vom Agenten, global oder aus der Standardeinstellung stammen)
- Elevated-Schranken und Schlüsselpfade zur Problembehebung

## Sandbox: wo Tools ausgeführt werden

Die Sandbox-Nutzung wird durch `agents.defaults.sandbox.mode` gesteuert:

- `"off"`: Alles wird auf dem Host ausgeführt.
- `"non-main"`: Nur Nicht-Hauptsitzungen werden in einer Sandbox ausgeführt (häufige „Überraschung“ bei Gruppen/Kanälen).
- `"all"`: Alles wird in einer Sandbox ausgeführt.

`agents.defaults.sandbox.workspaceAccess` steuert, was die Sandbox sehen kann: `"none"`, `"ro"` oder `"rw"`.

Die vollständige Matrix (Geltungsbereich, Workspace-Einhängungen, Images) finden Sie unter [Sandbox-Nutzung](/de/gateway/sandboxing).

### Bind-Mounts (schnelle Sicherheitsprüfung)

- `docker.binds` _durchbricht_ das Sandbox-Dateisystem: Alles, was Sie einhängen, ist innerhalb des Containers mit dem von Ihnen festgelegten Modus (`:ro` oder `:rw`) sichtbar.
- Wenn Sie den Modus weglassen, ist der Standard Lese-/Schreibzugriff; bevorzugen Sie `:ro` für Quellcode/Secrets.
- `scope: "shared"` ignoriert agentenspezifische Bind-Mounts (nur globale Bind-Mounts gelten).
- OpenClaw validiert Bind-Quellen zweimal: zuerst anhand des normalisierten Quellpfads und anschließend erneut nach der Auflösung über den tiefsten vorhandenen übergeordneten Pfad. Ausbrüche über symbolisch verknüpfte übergeordnete Pfade umgehen die Prüfungen auf gesperrte Pfade oder zulässige Stammverzeichnisse nicht.
- Auch nicht vorhandene Blattpfade werden sicher geprüft. Wenn `/workspace/alias-out/new-file` über einen symbolisch verknüpften übergeordneten Pfad in einen gesperrten Pfad oder außerhalb der konfigurierten zulässigen Stammverzeichnisse aufgelöst wird, wird der Bind-Mount abgelehnt.
- Das Einbinden von `/var/run/docker.sock` übergibt der Sandbox faktisch die Kontrolle über den Host; tun Sie dies nur absichtlich.
- Der Workspace-Zugriff (`workspaceAccess`) ist von den Bind-Modi unabhängig.

Eine agentenspezifische Konfiguration mit mehreren Host-Ordnern, Zugriffsmodi und der expliziten Sicherheitsfreigabe für externe Quellen finden Sie unter [Mehrere Ordner für einen Agenten](/de/gateway/sandboxing#multiple-folders-for-one-agent).

## Tool-Richtlinie: welche Tools vorhanden/aufrufbar sind

Mehrere Ebenen sind relevant:

- **Tool-Profil**: `tools.profile` und `agents.entries.*.tools.profile` (Basis-Zulassungsliste)
- **Provider-Tool-Profil**: `tools.byProvider[provider].profile` und `agents.entries.*.tools.byProvider[provider].profile`
- **Globale/agentenspezifische Tool-Richtlinie**: `tools.allow`/`tools.deny` und `agents.entries.*.tools.allow`/`agents.entries.*.tools.deny`
- **Provider-Tool-Richtlinie**: `tools.byProvider[provider].allow/deny` und `agents.entries.*.tools.byProvider[provider].allow/deny`
- **Sandbox-Tool-Richtlinie** (gilt nur innerhalb einer Sandbox): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` und `agents.entries.*.tools.sandbox.tools.*`

Faustregeln:

- `deny` hat immer Vorrang.
- Wenn `allow` nicht leer ist, gilt alles andere als gesperrt.
- Die Tool-Richtlinie ist die unumgängliche Grenze: `/exec` kann ein abgelehntes `exec`-Tool nicht außer Kraft setzen.
- Die Tool-Richtlinie filtert die Tool-Verfügbarkeit nach Namen; sie prüft keine Nebeneffekte innerhalb von `exec`. Wenn `exec` zulässig ist, werden Shell-Befehle durch das Ablehnen von `write`, `edit` oder `apply_patch` nicht schreibgeschützt.
- `/exec` ändert lediglich die Sitzungsvorgaben für autorisierte Absender; es gewährt keinen Tool-Zugriff.
- Provider-Tool-Schlüssel akzeptieren entweder `provider` (z. B. `google-antigravity`) oder `provider/model` (z. B. `openai/gpt-5.4`).
- Gateway-Protokolle enthalten `agents/tool-policy`-Überwachungseinträge, wenn ein Schritt der Tool-Richtlinie Tools entfernt oder eine Sandbox-Tool-Richtlinie einen Aufruf blockiert. Verwenden Sie `openclaw logs`, um die Regelbezeichnung, den Konfigurationsschlüssel und die betroffenen Tool-Namen anzuzeigen.

### Tool-Gruppen (Kurzformen)

Tool-Richtlinien (global, Agent, Sandbox) unterstützen `group:*`-Einträge, die zu mehreren Tools erweitert werden:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

Verfügbare Gruppen:

| Gruppe             | Tools                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`, `process`, `code_execution` (`bash` wird als Alias für `exec` akzeptiert)                                                                                                                                                                        |
| `group:fs`         | `read`, `write`, `edit`, `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `spawn_task`, `dismiss_task` |
| `group:memory`     | `memory_search`, `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`, `x_search`, `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`, `screen`, `terminal`, `canvas`, `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`, `cron`, `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`, `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`, `image_generate`, `music_generate`, `video_generate`, `tts`                                                                                                                                                                                   |
| `group:openclaw`   | die meisten integrierten OpenClaw-Tools (ausgenommen die Datei- und Laufzeitprimitive `read`/`write`/`edit`/`apply_patch`/`exec`/`process`, `canvas` und Provider-Plugins)                                                                                             |
| `group:plugins`    | alle geladenen Plugin-eigenen Tools, einschließlich konfigurierter MCP-Server, die über `bundle-mcp` bereitgestellt werden                                                                                                                                                           |

Lehnen Sie für schreibgeschützte Agenten neben dateisystemverändernden Tools auch `group:runtime` ab, sofern nicht die Sandbox-Dateisystemrichtlinie oder eine separate Host-Grenze die Schreibschutzbeschränkung durchsetzt.

Bei MCP-Servern in einer Sandbox ist die Sandbox-Tool-Richtlinie eine zweite Zulassungsschranke. Wenn `mcp.servers` konfiguriert ist, Sandbox-Durchläufe aber nur integrierte Tools anzeigen, fügen Sie `bundle-mcp`, `group:plugins` oder einen mit dem Serverpräfix versehenen MCP-Tool-Namen bzw. ein entsprechendes Glob-Muster wie `outlook__send_mail` oder `outlook__*` zu `tools.sandbox.tools.alsoAllow` hinzu. Starten Sie anschließend das Gateway neu bzw. laden Sie es neu und erfassen Sie die Tool-Liste erneut. Server-Glob-Muster verwenden das Provider-sichere MCP-Serverpräfix: Zeichen, die nicht `[A-Za-z0-9_-]` entsprechen, werden zu `-`; Namen, die nicht mit einem Buchstaben beginnen, erhalten das Präfix `mcp-`; lange oder doppelte Präfixe können gekürzt oder mit einem Suffix versehen werden.

`openclaw doctor` prüft derzeit diese Struktur für von OpenClaw verwaltete Server in `mcp.servers`. MCP-Server, die aus gebündelten Plugin-Manifesten oder Claude `.mcp.json` geladen werden, verwenden dieselbe Sandbox-Schranke, diese Diagnose führt diese Quellen jedoch noch nicht auf. Verwenden Sie dieselben Einträge in der Zulassungsliste, wenn deren Tools in Sandbox-Durchläufen verschwinden.

## Elevated: ausschließlich für die Ausführung „auf dem Host ausführen“

Elevated gewährt **keine** zusätzlichen Tools; es wirkt sich nur auf `exec` aus.

- Wenn Sie sich in einer Sandbox befinden, wird `/elevated on` (oder `exec` mit `elevated: true`) außerhalb der Sandbox ausgeführt (Genehmigungen können weiterhin erforderlich sein).
- Verwenden Sie `/elevated full`, um Ausführungsgenehmigungen für die Sitzung zu überspringen.
- Wenn die Ausführung bereits direkt erfolgt, ist Elevated praktisch wirkungslos (unterliegt aber weiterhin den Schranken).
- Elevated ist **nicht** auf Skills beschränkt und setzt die Tool-Zulassungs-/Ablehnungsregeln **nicht** außer Kraft.
- Elevated gewährt keine beliebigen hostübergreifenden Außerkraftsetzungen von `host=auto`; es folgt den normalen Regeln für Ausführungsziele und behält `node` nur bei, wenn das konfigurierte bzw. sitzungsspezifische Ziel bereits `node` ist.
- `/exec` ist von Elevated getrennt. Es passt lediglich die sitzungsspezifischen Ausführungsvorgaben für autorisierte Absender an.

Schranken:

- Aktivierung: `tools.elevated.enabled` (und optional `agents.entries.*.tools.elevated.enabled`)
- Absender-Zulassungslisten: `tools.elevated.allowFrom.<provider>` (und optional `agents.entries.*.tools.elevated.allowFrom.<provider>`)

Siehe [Elevated-Modus](/de/tools/elevated).

## Häufige Lösungen für „Sandbox-Gefängnis“-Probleme

### „Tool X durch Sandbox-Tool-Richtlinie blockiert“

Schlüssel zur Problembehebung (wählen Sie einen aus):

- Sandbox deaktivieren: `agents.defaults.sandbox.mode=off` (oder agentenspezifisch `agents.entries.*.sandbox.mode=off`)
- Das Tool innerhalb der Sandbox zulassen:
  - Entfernen Sie es aus `tools.sandbox.tools.deny` (oder agentenspezifisch aus `agents.entries.*.tools.sandbox.tools.deny`)
  - oder fügen Sie es zu `tools.sandbox.tools.allow` hinzu (oder agentenspezifisch zur Zulassungsliste)
- Prüfen Sie `openclaw logs` auf den Eintrag `agents/tool-policy`. Er zeichnet den Sandbox-Modus auf und gibt an, ob die Zulassungs- oder Verweigerungsregel das Tool blockiert hat.

### „Ich dachte, dies wäre die Hauptsitzung. Warum wird sie in einer Sandbox ausgeführt?“

Im Modus `"non-main"` sind Gruppen-/Kanalschlüssel _nicht_ die Hauptsitzung. Verwenden Sie den Schlüssel der Hauptsitzung (angezeigt durch `sandbox explain`) oder wechseln Sie in den Modus `"off"`.

## Verwandte Themen

- [Sandboxing](/de/gateway/sandboxing) -- vollständige Sandbox-Referenz (Modi, Geltungsbereiche, Backends, Images)
- [Multi-Agent-Sandbox und -Tools](/de/tools/multi-agent-sandbox-tools) -- agentenspezifische Überschreibungen und Rangfolge
- [Erweiterter Modus](/de/tools/elevated)
