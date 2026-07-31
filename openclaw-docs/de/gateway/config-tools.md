---
read_when:
    - Richtlinie, Positivlisten oder experimentelle Funktionen für `tools.*` konfigurieren
    - Benutzerdefinierte Provider registrieren oder Basis-URLs überschreiben
    - OpenAI-kompatible selbst gehostete Endpunkte einrichten
sidebarTitle: Tools and custom providers
summary: Tool-Konfiguration (Richtlinien, experimentelle Schalter, Provider-gestützte Tools) und Einrichtung benutzerdefinierter Provider/Basis-URLs
title: Konfiguration — Tools und benutzerdefinierte Provider
x-i18n:
    generated_at: "2026-07-26T17:49:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

`tools.*`-Konfigurationsschlüssel und Einrichtung benutzerdefinierter Provider/Basis-URLs. Informationen zu Agenten, Kanälen und anderen Konfigurationsschlüsseln auf oberster Ebene finden Sie in der [Konfigurationsreferenz](/de/gateway/configuration-reference).

## Tools

### Tool-Profile

`tools.profile` legt vor `tools.allow`/`tools.deny` eine grundlegende Positivliste fest:

<Note>
Beim lokalen Onboarding wird für neue lokale Konfigurationen standardmäßig `tools.profile: "coding"` verwendet, wenn kein Wert festgelegt ist (vorhandene explizite Profile bleiben erhalten).
</Note>

| Profil      | Enthält                                                                                                                                                                                                                                                 |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | nur `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`, `group:runtime`, `group:web`, `group:sessions`, `group:memory`, `cron`, `get_goal`, `create_goal`, `update_goal`, `update_plan`, `ask_user`, `skill_workshop`, `image`, `image_generate`, `music_generate`, `video_generate`                |
| `messaging` | `group:messaging`, `sessions`, `sessions_list`, `sessions_history`, `sessions_search`, `conversations_list`, `conversations_send`, `conversations_turn`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`, `ask_user` |
| `full`      | Keine Einschränkung (entspricht einem nicht gesetzten Wert)                                                                                                                                                                                             |

`coding` und `messaging` erlauben implizit auch `bundle-mcp` (konfigurierte MCP-Server).

### Tool-Gruppen

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
| `group:openclaw`   | Alle oben genannten integrierten Tools außer `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` (schließt Plugin-Tools aus)                                                                                                                                  |
| `group:plugins`    | Tools, die geladenen Plugins gehören, einschließlich konfigurierter MCP-Server, die über `bundle-mcp` bereitgestellt werden                                                                                                                                                           |

Mit `spawn_task` kann ein Coding-Agent bestätigte Folgearbeiten vorschlagen, ohne sie zu starten. Die Control UI zeigt Titel und Zusammenfassung als interaktiven Chip an; eine Gateway-gestützte TUI zeigt eine entsprechende interaktive Aufforderung. Wird einer der Vorschläge angenommen, wird eine neue verwaltete Worktree-Sitzung erstellt und die vollständige Eingabe dorthin gesendet, während der aktuelle Turn fortgesetzt wird. `dismiss_task` zieht einen noch ausstehenden Vorschlag anhand der kurzlebigen `task_id` zurück, die von `spawn_task` zurückgegeben wurde.

Die Tools werden nur angeboten, wenn die initiierende Bedienoberfläche Gateway-Ereignisse für Aufgabenvorschläge empfangen und verarbeiten kann. Kanalsitzungen und lokale/eingebettete TUI-Sitzungen empfangen sie nicht; Kanaltransporte benötigen eine portable typisierte Aufgabenaktion, bevor sie diesen Ablauf sicher bereitstellen können. Vorschläge sind prozesslokal und verschwinden bei einem Neustart des Gateways. Beide Tools bleiben im Profil `coding` und in `group:sessions`, sodass die normale Richtlinienkonfiguration über `tools.allow` und `tools.deny` sie automatisch konfiguriert, wenn die Oberfläche sie unterstützt.

### MCP- und Plugin-Tools innerhalb der Sandbox-Tool-Richtlinie

Konfigurierte MCP-Server werden unter der Plugin-ID `bundle-mcp` als Plugin-eigene Tools bereitgestellt. Normale Tool-Profile können sie erlauben, aber `tools.sandbox.tools` ist eine zusätzliche Schranke für Sandbox-Sitzungen. Wenn der Sandbox-Modus `"all"` oder `"non-main"` ist, fügen Sie einen der folgenden Einträge zur Sandbox-Tool-Positivliste hinzu, wenn MCP-/Plugin-Tools sichtbar sein sollen:

- `bundle-mcp` für von OpenClaw verwaltete MCP-Server aus `mcp.servers`
- die Plugin-ID für ein bestimmtes natives Plugin
- `group:plugins` für alle geladenen Plugin-eigenen Tools
- exakte Tool-Namen von MCP-Servern oder Server-Globs wie `outlook__send_mail` oder `outlook__*`, wenn nur ein Server gewünscht ist

Server-Globs verwenden das providersichere MCP-Serverpräfix, nicht notwendigerweise den unverarbeiteten `mcp.servers`-Schlüssel. Zeichen außerhalb von `[A-Za-z0-9_-]` werden zu `-`, Namen, die nicht mit einem Buchstaben beginnen, erhalten das Präfix `mcp-`, und lange oder doppelte Präfixe können gekürzt oder mit einem Suffix versehen werden; beispielsweise verwendet `mcp.servers["Outlook Graph"]` ein Glob wie `outlook-graph__*`.

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

Ohne diesen Eintrag auf Sandbox-Ebene kann der MCP-Server weiterhin erfolgreich geladen werden, während seine Tools vor der Provider-Anfrage herausgefiltert werden. Verwenden Sie `openclaw doctor`, um dieses Muster bei von OpenClaw verwalteten Servern in `mcp.servers` zu erkennen. MCP-Server, die aus gebündelten Plugin-Manifesten oder Claude `.mcp.json` geladen werden, verwenden dieselbe Sandbox-Schranke; diese Diagnose führt diese Quellen jedoch noch nicht auf. Verwenden Sie dieselben Einträge in der Positivliste, wenn deren Tools in Sandbox-Turns verschwinden.

### `tools.codeMode`

`tools.codeMode` aktiviert die generische Code-Mode-Oberfläche von OpenClaw. Wenn sie
für einen Lauf mit Tools aktiviert ist, werden normale OpenClaw-Tools hinter die
in der Sandbox befindliche `tools.*`-Katalogbrücke verschoben, und MCP-Tools sind über den generierten
Namensraum `MCP` verfügbar. Das Modell sieht normalerweise `exec` und `wait`; Tools wie `computer`,
deren strukturierte Ergebnisse die reine JSON-Brücke nicht passieren können, bleiben direkt verfügbar.

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

Auch die Kurzform wird akzeptiert:

```json5
{
  tools: { codeMode: true },
}
```

MCP-Deklarationen werden im Code-Mode über die schreibgeschützte virtuelle API-Dateioberfläche
bereitgestellt. Gastcode kann `API.list("mcp")` und
`API.read("mcp/<server>.d.ts")` aufrufen, um Signaturen im TypeScript-Stil zu prüfen, bevor
`MCP.<server>.<tool>()` aufgerufen wird. Informationen zum Laufzeitvertrag, zu Einschränkungen und zu Debugging-Schritten finden Sie unter [Code-Mode](/de/tools/code-mode).

### `tools.allow` / `tools.deny`

Globale Positiv-/Negativlistenrichtlinie für Tools (Verbote haben Vorrang). Groß-/Kleinschreibung wird nicht berücksichtigt, `*`-Platzhalter werden unterstützt. Wird auch angewendet, wenn die Docker-Sandbox deaktiviert ist.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` und `apply_patch` sind separate Tool-IDs. `allow: ["write"]` aktiviert für kompatible Modelle auch `apply_patch`, aber `deny: ["write"]` verbietet nicht `apply_patch`. Um sämtliche Dateiänderungen zu blockieren, verbieten Sie `group:fs` oder führen Sie jedes verändernde Tool explizit auf:

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` und `alsoAllow` können nicht beide im selben Geltungsbereich (`tools`, `tools.byProvider.<id>`, `agents.entries.*.tools`) festgelegt werden — die Konfigurationsvalidierung lehnt dies ab. Führen Sie die `alsoAllow`-Einträge mit `allow` zusammen oder entfernen Sie `allow` und verwenden Sie stattdessen `profile` + `alsoAllow`.
</Note>

### `tools.byProvider`

Schränken Sie Tools für bestimmte Provider oder Modelle weiter ein. Reihenfolge: Basisprofil → Provider-Profil → erlauben/verbieten.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

Schränkt die Tools für die Person ein, von der die aktuelle Anfrage ursprünglich stammt. Dies dient als mehrschichtige Absicherung zusätzlich zur Kanalzugriffskontrolle; Absenderwerte müssen vom Kanaladapter stammen, nicht aus dem Nachrichtentext. Andere Inhalte im Modell-Prompt werden dadurch nicht authentifiziert; siehe [Anfragestellerspezifische Steuerungen und Prompt-Kontext](/de/gateway/security#requester-scoped-controls-and-prompt-context).

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

Schlüssel verwenden explizite Präfixe: `channel:<channelId>:<senderId>`, `id:<senderId>`, `e164:<phone>`, `username:<handle>`, `name:<displayName>` oder `"*"`. Kanal-IDs sind kanonische OpenClaw-IDs; Aliasse wie `teams` werden zu `msteams` normalisiert. Veraltete Schlüssel ohne Präfix werden nur als `id:` akzeptiert. Die Abgleichsreihenfolge lautet: Kanal+ID, ID, e164, Benutzername, Name, dann Platzhalter.

Das agentenspezifische `agents.entries.*.tools.toolsBySender` überschreibt bei Übereinstimmung den globalen Absenderabgleich, selbst bei einer leeren `{}`-Richtlinie.

### `tools.elevated`

Steuert den erhöhten exec-Zugriff außerhalb der Sandbox:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- Die agentenspezifische Überschreibung (`agents.entries.*.tools.elevated`) kann nur weitere Einschränkungen vornehmen.
- `/elevated on|off|ask|full` speichert den Zustand pro Sitzung; Inline-Direktiven gelten für eine einzelne Nachricht.
- Erhöhtes `exec` umgeht die Sandbox und verwendet den konfigurierten Ausbruchspfad (standardmäßig `gateway` oder `node`, wenn das exec-Ziel `node` ist).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

Die angezeigten Werte sind Standardwerte, mit Ausnahme von `applyPatch.allowModels` (standardmäßig leer/nicht gesetzt, sodass jedes kompatible Modell `apply_patch` verwenden darf). `approvalRunningNoticeMs` gibt einen Hinweis zur laufenden Ausführung aus, wenn eine genehmigungsbasierte exec-Ausführung lange dauert; `0` deaktiviert ihn.

### `tools.loopDetection`

Sicherheitsprüfungen für Tool-Schleifen sind **standardmäßig deaktiviert**. Setzen Sie `enabled: true`, um die Erkennung zu aktivieren. Einstellungen können global in `tools.loopDetection` definiert und pro Agent unter `agents.entries.*.tools.loopDetection` überschrieben werden.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // oder Umgebungsvariable BRAVE_API_KEY (Brave-Provider)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // optional; für automatische Erkennung weglassen
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

Die angezeigten Werte sind Standardwerte, mit Ausnahme von `provider` und `userAgent`. `maxResponseBytes` begrenzt den Wert auf 32000–10000000; `maxChars` begrenzt ihn auf `maxCharsCap` (erhöhen Sie `maxCharsCap`, um größere Antworten zuzulassen).

### `tools.media`

Konfiguriert die Verarbeitung eingehender Medien (Bild/Audio/Video):

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` ist die einzige konfigurierte Modellliste. Jeder Eintrag gibt die von ihm verarbeiteten Fähigkeiten an. Der optionale `preferredModel`-Selektor akzeptiert `provider/model`, eine Modell-ID, `provider:<id>` für Einträge mit Provider-Standardwerten oder `cli:command`; übereinstimmende Einträge werden an den Anfang der Fallback-Reihenfolge der jeweiligen Fähigkeit verschoben. Fähigkeitsspezifische Prompts, Grenzwerte, Anfrageeinstellungen, Geltungsbereiche, Anhangsrichtlinien und die Wiedergabe von Audiotranskripten behalten für konfigurierte und automatisch erkannte Modelle ihre Standardwerte bei; ein Modelleintrag kann modellspezifische Felder überschreiben.

<AccordionGroup>
  <Accordion title="Felder eines Medienmodelleintrags">
    **Provider-Eintrag** (`type: "provider"` oder weggelassen):

    - `provider`: API-Provider-ID (`openai`, `anthropic`, `google`/`gemini`, `groq` usw.)
    - `model`: Überschreibung der Modell-ID
    - `profile` / `preferredProfile`: Auswahl des `auth-profiles.json`-Profils

    **CLI-Eintrag** (`type: "cli"`):

    - `command`: auszuführbare Datei
    - `args`: Vorlagenargumente (unterstützt `{{AttachmentPath}}`, `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{Prompt}}`, `{{MaxChars}}` usw.; `openclaw doctor --fix` migriert veraltete `{input}`-Platzhalter zu `{{AttachmentPath}}`). Die älteren Aliasse `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` und `{{MediaDir}}` bleiben während ihres Kompatibilitätszeitraums verfügbar, sind jedoch veraltet.

    **Gemeinsame Felder:**

    - `capabilities`: Liste mit einem oder mehreren der Werte `image`, `audio` und `video`.
    - `prompt`, `maxChars`, `maxBytes`, `timeoutSeconds`, `language`: eintragsspezifische Überschreibungen.
    - Übereinstimmende `timeoutSeconds`-Einträge für Bildmodelle gelten auch, wenn der Agent das explizite `image`-Tool aufruft. Bei der Bildverarbeitung gilt dieses Zeitlimit für die Anfrage selbst und wird nicht durch vorherige Vorbereitungsarbeiten verkürzt.
    - Bei Fehlern wird auf den nächsten Eintrag zurückgegriffen.

    Die Provider-Authentifizierung folgt der Standardreihenfolge: `auth-profiles.json` → Umgebungsvariablen → `models.providers.*.apiKey`.

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

Steuert, welche Sitzungen von den Sitzungstools (`sessions_list`, `sessions_history`, `sessions_send`) angesprochen werden können.

Standard: `tree` (aktuelle Sitzung + von ihr gestartete Sitzungen, etwa Subagenten, sowie im Hintergrund
beobachtete Gruppensitzungen desselben Agenten).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Sichtbarkeitsbereiche">
    - `self`: nur der aktuelle Sitzungsschlüssel.
    - `tree`: aktuelle Sitzung + von der aktuellen Sitzung gestartete Sitzungen (Subagenten). Bei Lesevorgängen sind außerdem Gruppensitzungen desselben Agenten enthalten, welche die aktuelle Sitzung über die im Hintergrund bestehende Gruppenwahrnehmung beobachtet.
    - `agent`: jede Sitzung, die zur aktuellen Agenten-ID gehört (kann andere Benutzer einschließen, wenn Sie absenderspezifische Sitzungen unter derselben Agenten-ID ausführen).
    - `all`: jede Sitzung. Agentenübergreifende Zielauswahl erfordert weiterhin `tools.agentToAgent`.
    - Sandbox-Begrenzung: Wenn die aktuelle Sitzung in einer Sandbox ausgeführt wird und `agents.defaults.sandbox.sessionToolsVisibility="spawned"` gilt (der Standardwert), wird die Sichtbarkeit auf `tree` erzwungen, selbst wenn `tools.sessions.visibility="all"`.
    - Wenn nicht `all`, enthält `sessions_list` ein kompaktes `visibility`-Feld,
      das den wirksamen Modus beschreibt und darauf hinweist, dass einige Sitzungen
      außerhalb des aktuellen Geltungsbereichs möglicherweise nicht aufgeführt werden.

  </Accordion>
</AccordionGroup>

Mit dem Standardwert `session.dmScope: "main"` macht menschliche Aktivität in einer Gruppe diese Gruppensitzung desselben Agenten
für die Hauptsitzung des Agenten im Hintergrund sichtbar. In einer Mehrbenutzerkonfiguration teilt `"main"` außerdem
eine DM-Sitzung zwischen Benutzern, sodass jeder dorthin weitergeleitete Benutzer aus im Hintergrund beobachteten Gruppen lesen kann,
einschließlich über das Sitzungsspeicher-`memory_search`. Verwenden Sie für die DM-Isolierung ein Peer-spezifisches `dmScope` oder setzen Sie
`tools.sessions.visibility: "self"`, um das Lesen aus im Hintergrund beobachteten Sitzungen zu deaktivieren.

### `tools.sessions_spawn`

Steuert die Unterstützung von Inline-Anhängen für `sessions_spawn`.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // Opt-in: auf true setzen, um Inline-Dateianhänge zuzulassen
        maxTotalBytes: 5242880, // insgesamt 5 MB über alle Dateien hinweg
        maxFiles: 50,
        maxFileBytes: 1048576, // 1 MB pro Datei
        retainOnSessionKeep: false, // Anhänge beibehalten, wenn cleanup="keep"
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Hinweise zu Anhängen">
    - Anhänge erfordern `enabled: true`.
    - Subagentenanhänge werden im untergeordneten Arbeitsbereich unter `.openclaw/attachments/<uuid>/` mit einem `.manifest.json` materialisiert.
    - ACP-Anhänge dürfen nur Bilder enthalten und werden inline an die ACP-Laufzeit weitergeleitet, nachdem dieselben Grenzwerte für Dateianzahl, Bytes pro Datei und Gesamtbytes eingehalten wurden.
    - Anhangsinhalte werden bei der dauerhaften Speicherung des Transkripts automatisch geschwärzt.
    - Base64-Eingaben werden durch strenge Prüfungen von Alphabet und Auffüllung sowie durch eine Größenprüfung vor der Dekodierung validiert.
    - Die Dateiberechtigungen für Subagentenanhänge sind `0700` für Verzeichnisse und `0600` für Dateien.
    - Die Bereinigung von Subagenten folgt der `cleanup`-Richtlinie: `delete` entfernt Anhänge immer; `keep` behält sie nur bei, wenn `retainOnSessionKeep: true`.

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

Experimentelle Flags für integrierte Tools. Standardmäßig deaktiviert, sofern keine Regel zur automatischen Aktivierung für strikt agentenorientiertes GPT-5 gilt.

```json5
{
  tools: {
    experimental: {
      planTool: true, // experimentelles update_plan aktivieren
    },
  },
}
```

- `planTool`: aktiviert das strukturierte `update_plan`-Tool zur Nachverfolgung nicht trivialer, mehrstufiger Arbeiten.
- Standard: `false`, sofern `agents.defaults.embeddedAgent.executionContract` (oder eine agentenspezifische Überschreibung) nicht für eine Ausführung mit einem `openai`-Provider und einer Modell-ID aus der GPT-5-Familie auf `"strict-agentic"` gesetzt ist (dies umfasst auch Ausführungen der OpenAI Codex CLI, da Codex-Authentifizierung und Modellrouting unter dem `openai`-Provider angesiedelt sind). Setzen Sie `true`, um das Tool außerhalb dieses Geltungsbereichs zu erzwingen, oder `false`, um es selbst für strikt agentenorientierte GPT-5-Ausführungen deaktiviert zu lassen.
- Wenn es aktiviert ist, ergänzt der System-Prompt außerdem Nutzungshinweise, damit das Modell es nur für umfangreiche Arbeiten verwendet und höchstens einen Schritt im Zustand `in_progress` hält.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: Standardmodell für gestartete Sub-Agenten. Wenn nicht angegeben, übernehmen Sub-Agenten das Modell des Aufrufers.
- `allowAgents`: Standard-Zulassungsliste konfigurierter Ziel-Agent-IDs für `sessions_spawn`, wenn der anfordernde Agent keine eigene `subagents.allowAgents` festlegt (`["*"]` = jedes konfigurierte Ziel; Standard: nur derselbe Agent). Veraltete Einträge, deren Agentenkonfiguration gelöscht wurde, werden von `sessions_spawn` abgelehnt und in `agents_list` ausgelassen; führen Sie `openclaw doctor --fix` aus, um sie zu bereinigen.
- `maxConcurrent`: maximale Anzahl gleichzeitiger Sub-Agent-Ausführungen. Standard: `8`.
- `runTimeoutSeconds`: Zeitüberschreitung (Sekunden) für `sessions_spawn`, wenn der Aufrufer keine eigene Überschreibung übergibt. Standard: `0` (keine Zeitüberschreitung); der oben gezeigte Wert `900` ist ein häufig verwendeter optionaler Wert, nicht der integrierte Standard.
- `announceTimeoutMs`: Zeitüberschreitung pro Aufruf (Millisekunden) für Zustellversuche von Gateway-`agent`-Ankündigungen. Standard: `120000`. Vorübergehende Wiederholungsversuche können dazu führen, dass die gesamte Wartezeit für die Ankündigung länger als eine konfigurierte Zeitüberschreitung ist.
- `archiveAfterMinutes`: Minuten nach Abschluss einer Sub-Agent-Sitzung, bevor sie automatisch archiviert wird. Standard: `60`; `0` deaktiviert die automatische Archivierung.
- Werkzeugrichtlinie pro Sub-Agent: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## Benutzerdefinierte Provider und Basis-URLs

Provider-Plugins veröffentlichen ihre eigenen Modellkatalogzeilen. Fügen Sie benutzerdefinierte Provider über `models.providers` in der Konfiguration oder über `~/.openclaw/agents/<agentId>/agent/models.json` hinzu.

Die Konfiguration eines benutzerdefinierten/lokalen Providers `baseUrl` ist zugleich die eng begrenzte Netzwerkvertrauensentscheidung für Modell-HTTP-Anfragen: OpenClaw lässt genau diesen `scheme://host:port`-Ursprung über den geschützten Abrufpfad zu, ohne eine separate Konfigurationsoption hinzuzufügen oder anderen privaten Ursprüngen zu vertrauen.

```json5
{
  models: {
    mode: "merge", // zusammenführen (Standard) | ersetzen
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | usw.
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Authentifizierung und Zusammenführungspriorität">
    - Verwenden Sie `authHeader: true` + `headers` für benutzerdefinierte Authentifizierungsanforderungen.
    - Überschreiben Sie das Stammverzeichnis der Agentenkonfiguration mit `OPENCLAW_AGENT_DIR`.
    - Zusammenführungspriorität für übereinstimmende Provider-IDs:
      - Nicht leere `models.json`-`baseUrl`-Werte des Agenten haben Vorrang.
      - Nicht leere `apiKey`-Werte des Agenten haben nur Vorrang, wenn dieser Provider im aktuellen Konfigurations-/Authentifizierungsprofilkontext nicht von SecretRef verwaltet wird.
      - Von SecretRef verwaltete `apiKey`-Werte des Providers werden aus Quellmarkierungen (`ENV_VAR_NAME` für Umgebungsreferenzen, `secretref-managed` für Datei-/Ausführungsreferenzen) aktualisiert, anstatt aufgelöste Geheimnisse dauerhaft zu speichern.
      - Von SecretRef verwaltete Headerwerte des Providers werden aus Quellmarkierungen (`secretref-env:ENV_VAR_NAME` für Umgebungsreferenzen, `secretref-managed` für Datei-/Ausführungsreferenzen) aktualisiert.
      - Leere oder fehlende `apiKey`/`baseUrl` des Agenten greifen auf `models.providers` in der Konfiguration zurück.
      - Bei übereinstimmenden Modell-`contextWindow`/`maxTokens` hat der explizite Konfigurationswert Vorrang, wenn er vorhanden und gültig ist (eine positive endliche Zahl); andernfalls wird der implizite/generierte Katalogwert verwendet.
      - Übereinstimmendes Modell-`contextTokens` folgt derselben Regel „explizit vor implizit“; verwenden Sie es, um den effektiven Kontext zu begrenzen, ohne die nativen Modellmetadaten zu ändern.
      - Kataloge von Provider-Plugins werden als generierte, dem Plugin zugeordnete Katalogsegmente im Plugin-Zustand des Agenten gespeichert.
      - Verwenden Sie `models.mode: "replace"`, wenn die Konfiguration `models.json` vollständig neu schreiben und das Zusammenführen Plugin-eigener Katalogsegmente überspringen soll.
      - Die Markierungsspeicherung richtet sich maßgeblich nach der Quelle: Markierungen werden aus dem aktiven Quellkonfigurations-Snapshot (vor der Auflösung) geschrieben, nicht aus aufgelösten Laufzeit-Geheimniswerten.

  </Accordion>
</AccordionGroup>

### Details zu Provider-Feldern

<AccordionGroup>
  <Accordion title="Katalog der obersten Ebene">
    - `models.mode`: Verhalten des Provider-Katalogs (`merge` oder `replace`).
    - `models.providers`: Zuordnung benutzerdefinierter Provider, nach Provider-ID verschlüsselt.
      - Sichere Bearbeitungen: Verwenden Sie `openclaw config set models.providers.<id> '<json>' --strict-json --merge` oder `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` für additive Aktualisierungen. `config set` lehnt destruktive Ersetzungen ab, sofern Sie nicht `--replace` übergeben.

  </Accordion>
  <Accordion title="Provider-Verbindung und Authentifizierung">
    - `models.providers.*.api`: Anfrageadapter (`openai-completions`, `openai-responses`, `openai-chatgpt-responses`, `anthropic-messages`, `google-generative-ai`, `google-vertex`, `github-copilot`, `bedrock-converse-stream`, `ollama`, `azure-openai-responses`). Verwenden Sie für selbst gehostete `/v1/chat/completions`-Backends wie MLX, vLLM, SGLang und die meisten OpenAI-kompatiblen lokalen Server `openai-completions`. Ein benutzerdefinierter Provider mit `baseUrl`, aber ohne `api`, verwendet standardmäßig `openai-completions`; legen Sie `openai-responses` nur fest, wenn das Backend `/v1/responses` unterstützt.
    - `models.providers.*.apiKey`: Provider-Anmeldedaten (SecretRef-/Umgebungsersetzung bevorzugen).
    - `models.providers.*.auth`: Authentifizierungsstrategie (`api-key`, `token`, `oauth`, `aws-sdk`).
    - `models.providers.*.contextWindow`: standardmäßiges natives Kontextfenster für Modelle dieses Providers, wenn der Modelleintrag `contextWindow` nicht festlegt.
    - `models.providers.*.contextTokens`: standardmäßige effektive Laufzeit-Kontextobergrenze für Modelle dieses Providers, wenn der Modelleintrag `contextTokens` nicht festlegt.
    - `models.providers.*.maxTokens`: standardmäßige Obergrenze für Ausgabe-Token für Modelle dieses Providers, wenn der Modelleintrag `maxTokens` nicht festlegt.
    - `models.providers.*.timeoutSeconds`: optionale Zeitüberschreitung in Sekunden für Modell-HTTP-Anfragen pro Provider, einschließlich Verbindungsaufbau, Headern, Textkörper und Behandlung des Abbruchs der gesamten Anfrage.
    - `models.providers.*.injectNumCtxForOpenAICompat`: fügt für Ollama + `openai-completions` `options.num_ctx` in Anfragen ein (Standard: `true`).
    - `models.providers.*.authHeader`: erzwingt bei Bedarf die Übertragung der Anmeldedaten im `Authorization`-Header.
    - `models.providers.*.baseUrl`: Basis-URL der vorgelagerten API.
    - `models.providers.*.headers`: zusätzliche statische Header für Proxy-/Mandanten-Routing.

  </Accordion>
  <Accordion title="Überschreibungen des Anfragetransports">
    `models.providers.*.request`: Transportüberschreibungen für HTTP-Anfragen an Modell-Provider.

    - `request.headers`: zusätzliche Header (mit den Provider-Standardwerten zusammengeführt). Werte akzeptieren SecretRef.
    - `request.auth`: Überschreibung der Authentifizierungsstrategie. Modi: `"provider-default"` (integrierte Authentifizierung des Providers verwenden), `"authorization-bearer"` (mit `token`), `"header"` (mit `headerName`, `value`, optional `prefix`).
    - `request.proxy`: HTTP-Proxy-Überschreibung. Modi: `"env-proxy"` (`HTTP_PROXY`/`HTTPS_PROXY`-Umgebungsvariablen verwenden), `"explicit-proxy"` (mit `url`). Beide Modi akzeptieren ein optionales `tls`-Unterobjekt.
    - `request.tls`: TLS-Überschreibung für direkte Verbindungen. Felder: `ca`, `cert`, `key`, `passphrase` (alle akzeptieren SecretRef), `serverName`, `insecureSkipVerify`.
    - `request.allowPrivateNetwork`: Wenn `true`, sind Modell-Provider-HTTP-Anfragen an private, CGNAT- oder ähnliche Bereiche über die HTTP-Abrufsicherung des Providers zulässig. Basis-URLs benutzerdefinierter/lokaler Provider vertrauen bereits genau dem konfigurierten Ursprung, ausgenommen Metadaten-/Link-Local-Ursprünge, die ohne ausdrückliche Aktivierung weiterhin blockiert bleiben. Setzen Sie dies auf `false`, um das Vertrauen in den exakten Ursprung zu deaktivieren. WebSocket verwendet dieselbe `request` für Header/TLS, jedoch nicht diese Abruf-SSRF-Sicherung. Standard: `false`.

  </Accordion>
  <Accordion title="Modellkatalogeinträge">
    - `models.providers.*.models`: explizite Provider-Modellkatalogeinträge.
    - `models.providers.*.models.*.input`: Modelleingabemodalitäten. Verwenden Sie `["text"]` für reine Textmodelle und `["text", "image"]` für native Bild-/Vision-Modelle. Bildanhänge werden nur in Agenteninteraktionen eingefügt, wenn das ausgewählte Modell als bildfähig gekennzeichnet ist.
    - `models.providers.*.models.*.contextWindow`: Metadaten des nativen Modellkontextfensters. Dies überschreibt `contextWindow` auf Provider-Ebene für dieses Modell.
    - `models.providers.*.models.*.contextTokens`: optionale Laufzeit-Kontextobergrenze. Dies überschreibt `contextTokens` auf Provider-Ebene; verwenden Sie sie, wenn Sie ein kleineres effektives Kontextbudget als den nativen `contextWindow` des Modells wünschen; `openclaw models list` zeigt beide Werte an, wenn sie sich unterscheiden.

    #### Fähigkeitsdeklarationen benutzerdefinierter Provider

    Provider-Kataloge besitzen `compat` für gebündelte und katalogbekannte Modellrouten. Kopieren Sie diese Flags nicht in die Konfiguration: OpenClaw verwendet die Katalogzeile, wenn die konfigurierten `api` und `baseUrl` diese Route weiterhin identifizieren. `openclaw doctor --fix` entfernt übereinstimmende veraltete Überschreibungen und meldet abweichende Werte zur Überprüfung.

    Ein `compat`-Block wird weiterhin für einen wirklich benutzerdefinierten Provider, ein benutzerdefiniertes Modell oder ein an einen anderen Endpunkt weitergeleitetes Katalogmodell unterstützt. Legen Sie nur Fähigkeiten fest, die für diesen Endpunkt verifiziert wurden:

    | Schlüssel für benutzerdefinierte Route | Laufzeitvertrag |
    | --- | --- |
    | `supportsStore` | Akzeptiert das OpenAI-Anfragefeld `store`. |
    | `supportsPromptCacheKey` | Akzeptiert OpenAI-Schlüssel für Prompt-Cache/Sitzungsaffinität. |
    | `supportsDeveloperRole` | Akzeptiert `developer`-Nachrichten, anstatt `system` zu erfordern. |
    | `supportsReasoningEffort` | Akzeptiert eine Steuerung des Reasoning-Aufwands. |
    | `supportsTemperature` | Akzeptiert `temperature` für dieses Modell und diesen Adapter. |
    | `supportsUsageInStreaming` | Gibt Nutzungsmetadaten in Streaming-Antworten aus. |
    | `supportsTools` | Unterstützt strukturierte Werkzeug-/Funktionsaufrufe. Setzen Sie `false`, um Werkzeuge zu deaktivieren. |
    | `supportsStrictMode` | Akzeptiert strikte Werkzeugschemas. |
    | `requiresStringContent` | Erfordert Nachrichteninhalte für Chat Completions als einfache Zeichenfolgen. |
    | `strictMessageKeys` | Erfordert, dass ausgehende Nachrichten nur akzeptierte Schlüssel enthalten. |
    | `visibleReasoningDetailTypes` | Benennt Typen von Reasoning-Detailblöcken, die sicher in Transkripten angezeigt werden können. |
    | `supportedReasoningEfforts` | Listet die vom Endpunkt akzeptierten Reasoning-Bezeichnungen auf. |
    | `reasoningEffortMap` | Ordnet OpenClaw-Denkbezeichnungen endpunktspezifischen Bezeichnungen zu. |
    | `maxTokensField` | Wählt `max_tokens` oder `max_completion_tokens` aus. |
    | `thinkingFormat` | Wählt den Dialekt der Reasoning-Nutzdaten des Endpunkts aus. |
    | `requiresToolResultName` | Erfordert einen Werkzeugnamen in Werkzeugergebnisnachrichten. |
    | `requiresAssistantAfterToolResult` | Erfordert nach Werkzeugergebnissen eine Assistentennachricht. |
    | `requiresThinkingAsText` | Gibt Reasoning bei der Wiedergabe als Text statt als strukturierten Inhalt wieder. |
    | `requiresReasoningContentOnAssistantMessages` | Bewahrt DeepSeek-artige `reasoning_content` während der Wiedergabe. |
    | `toolSchemaProfile` | Wählt ein vom Provider definiertes Normalisierungsprofil für Werkzeugschemas aus. |
    | `unsupportedToolSchemaKeywords` | Entfernt benannte JSON-Schema-Schlüsselwörter, die vom Endpunkt abgelehnt werden. |
    | `toolCallArgumentsEncoding` | Wählt die Kodierung der Werkzeugaufrufargumente des Endpunkts aus. |
    | `requiresOpenAiAnthropicToolPayload` | Konvertiert OpenAI-förmige Werkzeugaufrufe in Nutzdaten der Anthropic-Familie. |

  </Accordion>
  <Accordion title="Amazon-Bedrock-Erkennung">
    - `plugins.entries.amazon-bedrock.config.discovery`: Stamm der Einstellungen für die automatische Bedrock-Erkennung.
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: implizite Erkennung ein-/ausschalten.
    - `plugins.entries.amazon-bedrock.config.discovery.region`: AWS-Region für die Erkennung.
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: optionaler Provider-ID-Filter für die gezielte Erkennung.
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: Abfrageintervall für die Aktualisierung der Erkennung.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: Ausweichwert für das Kontextfenster erkannter Modelle.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: Ausweichwert für die maximale Anzahl an Ausgabe-Tokens erkannter Modelle.

  </Accordion>
</AccordionGroup>

Das interaktive Onboarding für benutzerdefinierte Provider leitet die Unterstützung für Bildeingaben aus bekannten Mustern für Vision-Modell-IDs ab. Dazu gehören GPT-4o/GPT-4.1/GPT-5+, die Reasoning-Familien `o1`/`o3`/`o4`, Claude, Gemini, jede ID mit dem Suffix `-vl` (Qwen-VL und Ähnliche) sowie benannte Familien wie LLaVA, Pixtral, InternVL, Mllama, MiniCPM-V und GLM-4V. Bei bekannten reinen Textfamilien (Llama, DeepSeek, Mistral/Mixtral, Kimi/Moonshot, Codestral, Devstral, Phi, QwQ, CodeLlama und einfachen Qwen-IDs ohne vl-/vision-Suffix) wird die zusätzliche Frage übersprungen. Bei unbekannten Modell-IDs wird weiterhin nach Bildunterstützung gefragt. Das nicht interaktive Onboarding verwendet dieselbe Ableitung. Übergeben Sie `--custom-image-input`, um Metadaten für Bildunterstützung zu erzwingen, oder `--custom-text-input`, um reine Textmetadaten zu erzwingen.

### Provider-Beispiele

<AccordionGroup>
  <Accordion title="Cerebras (GLM 4.7 / GPT OSS)">
    Das offizielle externe Provider-Plugin `cerebras` kann dies über `openclaw onboard --auth-choice cerebras-api-key` konfigurieren. Verwenden Sie eine explizite Provider-Konfiguration nur, wenn Sie Standardwerte überschreiben.

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    Verwenden Sie `cerebras/zai-glm-4.7` für Cerebras und `zai/glm-4.7` für den direkten Zugriff auf Z.AI.

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    Anthropic-kompatibler integrierter Provider. Kurzbefehl: `openclaw onboard --auth-choice kimi-code-api-key`.

  </Accordion>
  <Accordion title="Lokale Modelle (LM Studio)">
    Siehe [Lokale Modelle](/de/gateway/local-models). Kurz gesagt: Führen Sie über die LM Studio Responses API auf leistungsfähiger Hardware ein großes lokales Modell aus und lassen Sie gehostete Modelle als Ausweichoption zusammengeführt.
  </Accordion>
  <Accordion title="MiniMax M3 (direkt)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    Legen Sie `MINIMAX_API_KEY` fest. Kurzbefehle: `openclaw onboard --auth-choice minimax-global-api` oder `openclaw onboard --auth-choice minimax-cn-api`. Der Modellkatalog verwendet standardmäßig M3 und enthält außerdem die M2.7-Varianten. Auf dem Anthropic-kompatiblen Streaming-Pfad deaktiviert OpenClaw das Thinking von MiniMax M2.x standardmäßig, sofern Sie `thinking` nicht selbst explizit festlegen. MiniMax-M3 (und M3.x) verbleibt standardmäßig auf dem ausgelassenen/adaptiven Thinking-Pfad des Providers. `/fast on` oder `params.fastMode: true` schreibt `MiniMax-M2.7` in `MiniMax-M2.7-highspeed` um.

  </Accordion>
  <Accordion title="Moonshot AI (Kimi)">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    Für den Endpunkt in China: `baseUrl: "https://api.moonshot.cn/v1"` oder `openclaw onboard --auth-choice moonshot-api-key-cn`.

    Native Moonshot-Endpunkte geben auf dem gemeinsam genutzten Transport `openai-completions` die Kompatibilität mit Streaming-Nutzungsdaten an. OpenClaw richtet dies nach den Fähigkeiten des Endpunkts aus, nicht allein nach der ID des integrierten Providers.

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    Legen Sie `OPENCODE_API_KEY` (oder `OPENCODE_ZEN_API_KEY`) fest. Verwenden Sie `opencode/...`-Referenzen für den Zen-Katalog oder `opencode-go/...`-Referenzen für den Go-Katalog. Kurzbefehl: `openclaw onboard --auth-choice opencode-zen` oder `openclaw onboard --auth-choice opencode-go`.

  </Accordion>
  <Accordion title="Synthetic (Anthropic-kompatibel)">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    Die Basis-URL sollte `/v1` nicht enthalten (der Anthropic-Client hängt es an). Kurzbefehl: `openclaw onboard --auth-choice synthetic-api-key`.

  </Accordion>
  <Accordion title="Z.AI (GLM-4.7)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    Legen Sie `ZAI_API_KEY` fest. Modellreferenzen verwenden die kanonische Provider-ID `zai/*`. Kurzbefehl: `openclaw onboard --auth-choice zai-api-key`.

    - Allgemeiner Endpunkt: `https://api.z.ai/api/paas/v4`
    - Coding-Endpunkt: `https://api.z.ai/api/coding/paas/v4`
    - Die standardmäßige Authentifizierungsoption `zai-api-key` prüft Ihren Schlüssel und erkennt automatisch, zu welchem Endpunkt er gehört. Ist die Erkennung nicht eindeutig, wird eine Abfrage angezeigt, deren Standardwert „Global“ ist. Für eine explizite Auswahl stehen außerdem eigene Authentifizierungsoptionen für CN und den Coding-Plan zur Verfügung.
    - Definieren Sie für den allgemeinen Endpunkt einen benutzerdefinierten Provider mit überschriebener Basis-URL.

  </Accordion>
</AccordionGroup>

---

## Verwandte Themen

- [Konfiguration — Agenten](/de/gateway/config-agents)
- [Konfiguration — Kanäle](/de/gateway/config-channels)
- [Konfigurationsreferenz](/de/gateway/configuration-reference) — weitere Schlüssel der obersten Ebene
- [Tools und Plugins](/de/tools)
