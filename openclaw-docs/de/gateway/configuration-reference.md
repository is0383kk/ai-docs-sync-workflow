---
read_when:
    - Sie benötigen genaue Konfigurationssemantik oder Standardwerte auf Feldebene
    - Sie validieren Konfigurationsblöcke für Kanäle, Modelle, Gateways oder Tools
summary: Gateway-Konfigurationsreferenz für zentrale OpenClaw-Schlüssel, Standardwerte und Links zu Referenzen der jeweiligen Subsysteme
title: Konfigurationsreferenz
x-i18n:
    generated_at: "2026-07-26T17:47:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

Referenz auf Feldebene für `~/.openclaw/openclaw.json`: Schlüssel, Standardwerte und Links zu ausführlicheren Seiten der Subsysteme. Aufgabenorientierte Einrichtungsanleitungen finden Sie unter [Konfiguration](/de/gateway/configuration). Befehlsverzeichnisse im Besitz von Channels und Plugins sowie detaillierte Speicher-/QMD-Optionen befinden sich auf den jeweiligen eigenen Seiten, nicht hier.

Das Konfigurationsformat ist **JSON5** (Kommentare und nachgestellte Kommas sind zulässig). Alle Felder sind optional; OpenClaw verwendet bei Auslassung sichere Standardwerte.

Der Code ist maßgeblicher als diese Seite:

- `openclaw config schema` gibt das für die Validierung und Control UI verwendete aktuelle JSON-Schema aus, in das Metadaten von gebündelten Komponenten, Plugins und Channels zusammengeführt sind.
- Agents sollten vor dem Bearbeiten der Konfiguration die Tool-Aktion `config.schema.lookup` des Tools `gateway` für genau einen pfadbezogenen Schemaknoten aufrufen.
- `pnpm config:docs:check` / `pnpm config:docs:gen` validieren den Baseline-Hash dieser Dokumentation gegen die aktuelle Schemaoberfläche.

Schema-`uiHints` enthalten außerdem für jeden Pfad einen aufgelösten booleschen Wert `advanced`.
Control UI verwendet ihn, um häufig verwendete Felder zuerst anzuzeigen und erweiterte Felder pro
Abschnitt einzuklappen; die Suche umfasst weiterhin beide Ebenen. Die Ebenenmetadaten dienen nur der Darstellung.
Wenn Sie einen Schlüssel hinzufügen, deklarieren Sie seine Ebene am Blatt oder lassen Sie ihn die Deklaration des nächsten
Vorfahren erben. Ein Pfad ohne deklarierten Vorfahren ist standardmäßig erweitert.

Eigene ausführliche Referenzen:

- [Referenz zur Speicherkonfiguration](/de/reference/memory-config) für `memory.search.*`, `memory.qmd.*`, `memory.citations` und die Dreaming-Konfiguration unter `plugins.entries.memory-core.config.dreaming`.
- [Slash-Befehle](/de/tools/slash-commands) für das aktuelle Verzeichnis integrierter und gebündelter Befehle.
- Die zuständigen Channel-/Plugin-Seiten für channelspezifische Befehlsoberflächen.

---

## Channels

Konfigurationsschlüssel für einzelne Channels finden Sie unter [Konfiguration – Channels](/de/gateway/config-channels): `channels.*` für Slack, Discord, Telegram, WhatsApp, Matrix, iMessage und andere gebündelte Channels (Authentifizierung, Zugriffskontrolle, mehrere Konten, Erwähnungssteuerung).

## Agent-Standardwerte, mehrere Agents, Sitzungen und Nachrichten

Unter [Konfiguration – Agents](/de/gateway/config-agents) finden Sie:

- `agents.defaults.*` (Arbeitsbereich, Modell, Denken, Heartbeat, Speicher, Medien, Skills, Sandbox)
- `multiAgent.*` (Routing und Bindungen für mehrere Agents)
- `session.*` (Sitzungslebenszyklus, Compaction, Bereinigung)
- `messages.*` (Nachrichtenzustellung, TTS, Markdown-Darstellung)
- `talk.*` (Talk-Modus)
  - `talk.consultThinkingLevel`: Überschreibung der Denkstufe für den vollständigen OpenClaw-Agent-Lauf hinter Echtzeitberatungen von Control UI Talk
  - `talk.consultFastMode`: einmalige Überschreibung für den Schnellmodus bei Echtzeitberatungen von Control UI Talk
  - `talk.speechLocale`: optionale BCP-47-Gebietsschema-ID für die Spracherkennung von Talk unter Android, iOS und macOS
  - `talk.silenceTimeoutMs`: wenn nicht festgelegt, behält Talk das Standard-Pausenfenster der Plattform bei, bevor das Transkript gesendet wird (`700 ms on macOS and Android, 900 ms on iOS`)
  - `talk.realtime.consultRouting`: Gateway-Relay-Fallback für abgeschlossene Echtzeittranskripte von Talk, die `openclaw_agent_consult` überspringen

## Tools und benutzerdefinierte Provider

Tool-Richtlinien, experimentelle Umschalter, die Konfiguration von Provider-gestützten Tools sowie die Einrichtung benutzerdefinierter
Provider/Basis-URLs finden Sie unter
[Konfiguration – Tools und benutzerdefinierte Provider](/de/gateway/config-tools).

## Modelle

Provider-Definitionen, Modell-Zulassungslisten und die Einrichtung benutzerdefinierter Provider finden Sie unter
[Konfiguration – Tools und benutzerdefinierte Provider](/de/gateway/config-tools#custom-providers-and-base-urls).
Die Wurzel `models` steuert außerdem das globale Verhalten des Modellkatalogs.

```json5
{
  models: {
    // Optional. Standardwert: true. Erfordert bei einer Änderung einen Neustart des Gateways.
    pricing: { enabled: false },
  },
}
```

- `models.mode`: Verhalten des Provider-Katalogs (`merge` oder `replace`).
- `models.providers`: benutzerdefinierte Provider-Zuordnung, deren Schlüssel die Provider-ID ist.
- `models.providers.*.localService`: optionaler bedarfsgesteuerter Prozessmanager für
  lokale Modellserver. OpenClaw prüft den konfigurierten Zustandsendpunkt, startet bei
  Bedarf den absoluten Pfad `command`, wartet auf die Betriebsbereitschaft und sendet dann die Modellanfrage.
  Siehe [Lokale Modelldienste](/de/gateway/local-model-services).
- `models.pricing.enabled`: steuert die im Hintergrund ausgeführte Initialisierung der Preisinformationen, die
  beginnt, nachdem Sidecars und Channels den Bereitschaftspfad des Gateways erreicht haben. Bei `false`
  überspringt das Gateway das Abrufen der Preiskataloge von OpenRouter und LiteLLM; konfigurierte
  Werte für `models.providers.*.models[].cost` funktionieren weiterhin für lokale Kostenschätzungen.

## MCP

Von OpenClaw verwaltete MCP-Serverdefinitionen befinden sich unter `mcp.servers` und werden
vom eingebetteten OpenClaw und anderen Laufzeitadaptern verwendet. Die Befehle `openclaw mcp list`,
`show`, `set` und `unset` verwalten diesen Block, ohne bei Konfigurationsänderungen eine Verbindung zum
Zielserver herzustellen.

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // Optionale Projektionssteuerung für den Codex-App-Server.
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`: benannte stdio- oder entfernte MCP-Serverdefinitionen für Laufzeiten, die
  konfigurierte MCP-Tools bereitstellen.
  Entfernte Einträge verwenden `transport: "streamable-http"` oder `transport: "sse"`;
  `type: "http"` ist ein CLI-nativer Alias, den `openclaw mcp set` und
  `openclaw doctor --fix` in das kanonische Feld `transport` normalisieren.
- `mcp.servers.<name>.enabled`: Legen Sie `false` fest, um eine gespeicherte Serverdefinition
  beizubehalten, sie jedoch von der MCP-Erkennung und Tool-Projektion des eingebetteten OpenClaw auszuschließen.
- `mcp.servers.<name>.requestTimeoutMs`: MCP-Anfragezeitlimit pro Server in Millisekunden.
- `mcp.servers.<name>.connectionTimeoutMs`: Verbindungszeitlimit pro Server in Millisekunden.
- `mcp.servers.<name>.supportsParallelToolCalls`: optionaler Nebenläufigkeitshinweis für
  Adapter, die entscheiden können, ob sie parallele MCP-Tool-Aufrufe ausführen.
- `mcp.servers.<name>.auth`: Legen Sie `"oauth"` für HTTP-MCP-Server fest, die
  OAuth erfordern. Führen Sie `openclaw mcp login <name>` aus, um Tokens im OpenClaw-Zustand zu speichern.
- `mcp.servers.<name>.oauth`: optionale Überschreibungen für OAuth-Berechtigungsumfang, Weiterleitungs-URL und URL
  der Client-Metadaten.
- `mcp.servers.<name>.sslVerify`, `clientCert`, `clientKey`: HTTP-TLS-Steuerung
  für private Endpunkte und gegenseitiges TLS.
- `mcp.servers.<name>.toolFilter`: optionale Tool-Auswahl pro Server. `include`
  begrenzt die erkannten MCP-Tools auf übereinstimmende Namen; `exclude` blendet übereinstimmende
  Namen aus. Einträge sind exakte MCP-Tool-Namen oder einfache `*`-Glob-Muster. Server mit
  Ressourcen oder Prompts erzeugen außerdem Namen von Hilfstools (`resources_list`,
  `resources_read`, `prompts_list`, `prompts_get`); für diese Namen gilt derselbe
  Filter.
- `mcp.servers.<name>.codex`: optionale Projektionssteuerung für den Codex-App-Server.
  Dieser Block enthält ausschließlich OpenClaw-Metadaten für Codex-App-Server-Threads; er wirkt sich nicht
  auf ACP-Sitzungen, die generische Codex-Harness-Konfiguration oder andere Laufzeitadapter aus.
  Ein nicht leeres `codex.agents` beschränkt den Server auf die aufgeführten OpenClaw-Agent-IDs.
  Leere oder ungültige bereichsbezogene Agent-Listen werden von der Konfigurationsvalidierung abgelehnt
  und vom Laufzeit-Projektionspfad ausgelassen, statt global zu werden.
  `codex.defaultToolsApprovalMode` gibt das native
  `default_tools_approval_mode` von Codex für diesen Server aus. OpenClaw entfernt den Block `codex`,
  bevor die native Konfiguration `mcp_servers` an Codex übergeben wird. Lassen Sie den Block aus, um
  den Server für jeden Codex-App-Server-Agent mit dem standardmäßigen
  MCP-Genehmigungsverhalten von Codex zu projizieren.
- Sitzungsbezogene gebündelte MCP-Laufzeiten verwenden eine integrierte Leerlauf-TTL von 10 Minuten.
  Einmalige eingebettete Läufe fordern eine Bereinigung am Laufende an; die TTL dient als Absicherung für langlebige Sitzungen und zukünftige Aufrufer.
- Änderungen unter `mcp.*` werden direkt angewendet, indem zwischengespeicherte Sitzungs-MCP-Laufzeiten verworfen werden.
  Bei der nächsten Tool-Erkennung/-Verwendung werden sie anhand der neuen Konfiguration neu erstellt, sodass entfernte
  `mcp.servers`-Einträge sofort bereinigt werden, statt auf die Leerlauf-TTL zu warten.
- Die Laufzeiterkennung berücksichtigt außerdem Benachrichtigungen über Änderungen an MCP-Tool-Listen, indem sie
  den zwischengespeicherten Katalog für diese Sitzung verwirft. Server, die Ressourcen oder
  Prompts anbieten, erhalten Hilfstools zum Auflisten/Lesen von Ressourcen sowie zum Auflisten/Abrufen von
  Prompts. Wiederholte Fehler bei Tool-Aufrufen pausieren den betroffenen Server kurz, bevor
  ein weiterer Aufruf versucht wird.

Informationen zum Laufzeitverhalten finden Sie unter [MCP](/de/cli/mcp#openclaw-as-an-mcp-client-registry) und
[CLI-Backends](/de/gateway/cli-backends#bundle-mcp-overlays).

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // oder Klartextzeichenfolge
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: optionale Zulassungsliste nur für gebündelte Skills (verwaltete Skills und Arbeitsbereich-Skills sind nicht betroffen).
- `load.extraDirs`: zusätzliche gemeinsam genutzte Skill-Stammverzeichnisse (niedrigste Priorität).
- `load.allowSymlinkTargets`: vertrauenswürdige reale Zielstammverzeichnisse, in die Skill-Symlinks
  aufgelöst werden dürfen, wenn sich der Link außerhalb seines konfigurierten Quellstammverzeichnisses befindet.
- `workshop.allowSymlinkTargetWrites`: erlaubt Skill Workshop beim Anwenden, über
  bereits vertrauenswürdige Symlink-Ziele zu schreiben (Standardwert: false).
- `install.preferBrew`: wenn true, werden Homebrew-Installationsprogramme bevorzugt, wenn `brew`
  verfügbar ist, bevor auf andere Installationsarten zurückgegriffen wird.
- `install.nodeManager`: bevorzugtes Node-Installationsprogramm für `metadata.openclaw.install`-Spezifikationen
  (`npm` | `pnpm` | `yarn` | `bun`).
- `install.allowUploadedArchives`: erlaubt vertrauenswürdigen `operator.admin`-Gateway-
  Clients die Installation privater ZIP-Archive, die über `skills.upload.*` bereitgestellt wurden
  (Standardwert: false). Dies aktiviert nur den Pfad für hochgeladene Archive; normale ClawHub-
  Installationen benötigen ihn nicht.
- `entries.<skillKey>.enabled: false` deaktiviert einen Skill, selbst wenn er gebündelt/installiert ist.
- `entries.<skillKey>.apiKey`: Komfortoption für Skills, die eine primäre Umgebungsvariable deklarieren (Klartextzeichenfolge oder SecretRef-Objekt).
- `limits.maxCandidatesPerRoot`, `limits.maxSkillsLoadedPerSource`, `limits.maxSkillsInPrompt`, `limits.maxSkillsPromptChars`, `limits.maxSkillFileBytes`: begrenzen die Skill-Erkennung und den modellseitigen Skills-Prompt.
- Autonomie-/Genehmigungseinstellungen von Skill Workshop (`workshop.autonomous.enabled`, `workshop.approvalPolicy`, `workshop.maxPending`, `workshop.maxSkillBytes`) sind unter [Skills-Konfiguration](/de/tools/skills-config) dokumentiert.

---

## Plugins

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- Wird aus Paket- oder Bundle-Verzeichnissen unter `~/.openclaw/extensions` und `<workspace>/.openclaw/extensions` sowie aus den in `plugins.load.paths` aufgeführten Dateien oder Verzeichnissen geladen.
- Legen Sie eigenständige Plugin-Dateien in `plugins.load.paths` ab; automatisch erkannte Erweiterungsstammverzeichnisse ignorieren `.js`-, `.mjs`- und `.ts`-Dateien auf oberster Ebene, damit Hilfsskripte in diesen Stammverzeichnissen den Start nicht blockieren.
- Die Erkennung akzeptiert native OpenClaw-Plugins sowie kompatible Codex-Bundles und Claude-Bundles, einschließlich manifestloser Claude-Bundles mit Standardlayout.
- **Konfigurationsänderungen erfordern einen Neustart des Gateways.**
- `allow`: optionale Zulassungsliste (nur aufgeführte Plugins werden geladen). `deny` hat Vorrang.
- `plugins.entries.<id>.apiKey`: komfortables Feld für den API-Schlüssel auf Plugin-Ebene (sofern vom Plugin unterstützt).
- `plugins.entries.<id>.env`: Plugin-spezifische Zuordnung von Umgebungsvariablen.
- `plugins.entries.<id>.hooks.allowPromptInjection`: Wenn `false`, blockiert der Kern Prompt-verändernde Hooks wie `before_prompt_build`. Dies gilt für native Plugin-Hooks und unterstützte, von Bundles bereitgestellte Hook-Verzeichnisse.
- `plugins.entries.<id>.hooks.allowConversationAccess`: Wenn `true`, dürfen vertrauenswürdige, nicht gebündelte Plugins rohe Gesprächsinhalte aus typisierten Hooks wie `llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize` und `agent_end` lesen.
- `plugins.entries.<id>.subagent.allowModelOverride`: Diesem Plugin ausdrücklich vertrauen, damit es pro Ausführung Überschreibungen für `provider` und `model` bei Subagent-Hintergrundausführungen anfordern darf.
- `plugins.entries.<id>.subagent.allowedModels`: optionale Zulassungsliste kanonischer `provider/model`-Ziele für vertrauenswürdige Subagent-Überschreibungen. Verwenden Sie `"*"` nur, wenn Sie absichtlich jedes Modell zulassen möchten.
- `plugins.entries.<id>.llm.allowModelOverride`: Diesem Plugin ausdrücklich vertrauen, damit es Modellüberschreibungen für `api.runtime.llm.complete` anfordern darf.
- `plugins.entries.<id>.llm.allowedModels`: optionale Zulassungsliste kanonischer `provider/model`-Ziele für vertrauenswürdige Überschreibungen von Plugin-LLM-Vervollständigungen. Verwenden Sie `"*"` nur, wenn Sie absichtlich jedes Modell zulassen möchten.
- `plugins.entries.<id>.llm.allowAgentIdOverride`: Diesem Plugin ausdrücklich vertrauen, damit es `api.runtime.llm.complete` für eine nicht standardmäßige Agenten-ID ausführen darf.
- `plugins.entries.<id>.config`: vom Plugin definiertes Konfigurationsobjekt (wird anhand des nativen OpenClaw-Plugin-Schemas validiert, sofern verfügbar).
- Konto- und Laufzeiteinstellungen für Kanal-Plugins befinden sich unter `channels.<id>` und sollten durch die `channelConfigs`-Metadaten im Manifest des zuständigen Plugins beschrieben werden, nicht durch eine zentrale OpenClaw-Optionsregistrierung.

### Konfiguration des Codex-Harness-Plugins

Das gebündelte Plugin `codex` verwaltet die Einstellungen des nativen Codex-App-Server-Harness unter
`plugins.entries.codex.config`. Die vollständige Konfigurationsoberfläche finden Sie in der
[Codex-Harness-Referenz](/de/plugins/codex-harness-reference), das Laufzeitmodell unter
[Codex-Harness](/de/plugins/codex-harness).

`codexPlugins` gilt nur für Sitzungen, die den nativen Codex-Harness auswählen.
Es aktiviert keine Codex-Plugins für OpenClaw-Provider-Ausführungen, ACP-
Gesprächsbindungen oder andere Harnesses als Codex.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: aktiviert die native Unterstützung für Codex-
  Plugins/Apps im Codex-Harness. Standard: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: stellt jede
  aktuell zugängliche App, die mit dem authentifizierten Codex-Konto verbunden ist, in
  jedem neuen nativen Codex-Thread bereit. Standard: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  Standardrichtlinie für destruktive Aktionen bei konfigurierten Plugin-App-Abfragen.
  Verwenden Sie `true`, um sichere Codex-Genehmigungsschemas ohne Rückfrage zu akzeptieren, `false`,
  um sie abzulehnen, `"auto"`, um von Codex verlangte Genehmigungen über OpenClaw-
  Plugin-Genehmigungen zu leiten, oder `"ask"`, um bei jeder schreibenden/destruktiven
  Plugin-Aktion ohne dauerhafte Genehmigung nachzufragen. Der Modus `"ask"` löscht dauerhafte Codex-
  Genehmigungsüberschreibungen pro Tool für die betroffene App und wählt den menschlichen
  Genehmigungsprüfer für diese App aus, bevor der Codex-Thread beginnt.
  Standard: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: aktiviert einen
  konfigurierten Plugin-Eintrag, wenn auch das globale `codexPlugins.enabled` wahr ist.
  Standard: `true` für explizite Einträge.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  stabile Marketplace-Identität, die zusammen mit `pluginName` für jeden aufgelösten
  Eintrag erforderlich ist. Unterstützt `"openai-curated"` und `"workspace-directory"`. Einträge,
  denen eines der beiden Identitätsfelder fehlt, werden ignoriert.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: stabile
  Codex-Plugin-Identität, die zusammen mit `marketplaceName` erforderlich ist. Ein
  `workspace-directory`-Eintrag muss das exakte, Marketplace-qualifizierte
  `summary.id` verwenden, das von `plugin/list` zurückgegeben wird, zum Beispiel
  `"example-plugin@workspace-directory"`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  Plugin-spezifische Überschreibung für destruktive Aktionen. Wenn sie fehlt, wird der globale
  Wert `allow_destructive_actions` verwendet. Der Plugin-spezifische Wert akzeptiert dieselben
  Richtlinien `true`, `false`, `"auto"` oder `"ask"`.

Jede zugelassene Plugin-App, die `"ask"` verwendet, leitet die Genehmigungsanfragen dieser App
an den menschlichen Prüfer weiter. Andere Apps und Genehmigungen für Threads, die nicht zu Apps gehören, behalten ihren
konfigurierten Prüfer, sodass gemischte Plugin-Richtlinien das Verhalten von `"ask"` nicht übernehmen.

`codexPlugins.enabled` ist die globale Aktivierungsanweisung. Explizite Plugin-
Einträge, die durch eine Migration geschrieben wurden, bilden die dauerhafte kuratierte Menge für Installations- und Reparaturberechtigungen.
Manuell konfigurierte `workspace-directory`-Einträge müssen bereits
installiert und aktiviert sein, und die zugehörigen Apps müssen zugänglich sein; OpenClaw
installiert oder authentifiziert sie nicht. Wenn Codex die explizite Kataloganfrage des Arbeitsbereichs
ablehnt, schlagen aktivierte Arbeitsbereichseinträge geschlossen mit
`marketplace_missing` fehl, während kuratierte Einträge aus dem Standardkatalog weiterhin
verfügbar bleiben. `plugins["*"]` wird nicht unterstützt, es gibt keinen `install`-Schalter, und
lokale `marketplacePath`-Werte sind absichtlich keine Konfigurationsfelder, da sie
hostspezifisch sind. Anforderungen an App-Server-Version und
Bereitschaft finden Sie unter [Native Codex-Plugins](/de/plugins/codex-native-plugins).

Bereitschaftsprüfungen für `app/list` werden eine Stunde lang zwischengespeichert und bei Veraltung
asynchron aktualisiert. Die App-Konfiguration für Codex-Threads wird beim Aufbau der Codex-Harness-
Sitzung berechnet, nicht bei jedem Durchlauf; verwenden Sie nach Änderungen an der nativen Plugin-Konfiguration `/new`, `/reset` oder einen
Neustart des Gateways.

`codexPlugins.allow_all_plugins` fügt jeder neuen nativen Codex-Konversation eine Momentaufnahme jeder
aktuell zugänglichen Konto-App hinzu. Es installiert keine Plugins oder Apps, und
nicht zugängliche Apps bleiben ausgeschlossen. Konto-Apps verwenden die globale
Richtlinie `codexPlugins.allow_destructive_actions`. Explizite Plugin-Einträge haben
Vorrang, wenn dieselbe App über beide Pfade vorhanden ist. Wenn `app/list` nicht
gelesen werden kann, schlägt die kontoweite Bereitstellung geschlossen fehl.

- `plugins.entries.firecrawl.config.webFetch`: Einstellungen für den Firecrawl-Webabruf-Provider.
  - `apiKey`: Optionaler Firecrawl-API-Schlüssel für höhere Limits (akzeptiert SecretRef). Fällt auf die Umgebungsvariable `plugins.entries.firecrawl.config.webSearch.apiKey` oder `FIRECRAWL_API_KEY` zurück.
  - `baseUrl`: Firecrawl-API-Basis-URL (Standard: `https://api.firecrawl.dev`; selbst gehostete Überschreibungen müssen auf private/interne Endpunkte verweisen).
  - `onlyMainContent`: extrahiert nur den Hauptinhalt von Seiten (Standard: `true`).
  - `maxAgeMs`: maximales Cache-Alter in Millisekunden (Standard: `172800000` / 2 Tage).
  - `timeoutSeconds`: Zeitüberschreitung für Scraping-Anfragen in Sekunden (Standard: `60`).
- `plugins.entries.xai.config.xSearch`: Einstellungen für xAI X Search (Grok-Websuche).
  - `enabled`: aktiviert den X-Search-Provider.
  - `model`: für die Suche zu verwendendes Grok-Modell (z. B. `"grok-4.3"`).
- `plugins.entries.memory-core.config.dreaming`: Einstellungen für Memory Dreaming. Phasen und Schwellenwerte finden Sie unter [Dreaming](/de/concepts/dreaming).
  - `enabled`: Hauptschalter für Dreaming (Standard: `false`).
  - `frequency`: Cron-Intervall für jeden vollständigen Dreaming-Durchlauf (standardmäßig `"0 3 * * *"`).
  - `model`: optionale Überschreibung des Subagent-Modells für Dream Diary. Erfordert `plugins.entries.memory-core.subagent.allowModelOverride: true`; kombinieren Sie dies mit `allowedModels`, um Ziele einzuschränken. Fehler aufgrund eines nicht verfügbaren Modells führen zu einem erneuten Versuch mit dem Standardmodell der Sitzung; bei Fehlern der Vertrauensprüfung oder Zulassungsliste erfolgt kein stiller Rückfall.
  - Phasenrichtlinie und Schwellenwerte sind Implementierungsdetails (keine für Benutzer sichtbaren Konfigurationsschlüssel).
- Die vollständige Memory-Konfiguration finden Sie in der [Referenz zur Memory-Konfiguration](/de/reference/memory-config):
  - `memory.search.*`
  - `agents.entries.*.memory.search.*` für Agenten-spezifische Überschreibungen
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- Aktivierte Claude-Bundle-Plugins können außerdem eingebettete OpenClaw-Standardeinstellungen aus `settings.json` beitragen; OpenClaw wendet diese als bereinigte Agenteneinstellungen an, nicht als rohe OpenClaw-Konfigurations-Patches.
- `plugins.slots.memory`: wählt die ID des aktiven Memory-Plugins oder `"none"`, um Memory-Plugins zu deaktivieren.
- `plugins.slots.contextEngine`: wählt die ID des aktiven Kontext-Engine-Plugins; standardmäßig `"legacy"`, sofern Sie keine andere Engine installieren und auswählen.

Siehe [Plugins](/de/tools/plugin).

---

## Browser

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // nur für vertrauenswürdigen Zugriff auf private Netzwerke aktivieren
      // allowPrivateNetwork: true, // veralteter Alias
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false` deaktiviert `act:evaluate` und `wait --fn`.
- `tabCleanup` steuert die nach bestem Bemühen regelmäßig durchgeführte Bereinigung nachverfolgter Tabs des primären Agenten
  nach einer Leerlaufzeit oder wenn eine Sitzung ihr Limit überschreitet. Die Nachverfolgung gilt nur
  für Tabs, die vom Browser-Tool `action: "open"` erstellt wurden; vom Benutzer geöffnete Tabs oder
  Tabs mit unbekannter Eigentümerschaft werden niemals übernommen. Das Deaktivieren von `tabCleanup` deaktiviert nicht die explizite Bereinigung des Sitzungslebenszyklus.
- Host-lokale Öffnungen mit einem stabilen nativen CDP-Ziel und einer Browseridentität werden
  im gemeinsamen SQLite-Zustand gespeichert und bleiben über Gateway-Neustarts hinweg für
  `/new` und die Bereinigung des Sitzungslebenszyklus berechtigt. Native, für Tools bestimmte CDP-Ziele
  bleiben auch nach einem Neustart für die Leerlauf- und Limitbereinigung berechtigt. Chrome MCP verwendet
  prozesslokale Ziel-Handles, daher warten kalte Datensätze bestehender Sitzungen auf die
  Lebenszyklusbereinigung, statt eine Leerlaufbereinigung zu riskieren, die nicht zuordenbare
  Aktivitäten nach einem Neustart betrifft. OpenClaw überprüft das Profil und die Browserinstanz,
  bevor sie geschlossen wird. Die automatische Verbindung von Chrome MCP, eine fehlende
  `/json/version`-Browseridentität und nicht aufgelöste native Ziele bleiben vollständig prozesslokal,
  sodass sie nach einem Neustart nicht automatisch geschlossen werden. Ältere, nicht nachverfolgte Tabs
  müssen manuell geschlossen werden. Vorübergehende Fehler bleiben für einen späteren erneuten Versuch
  ausstehend. Siehe [Eigentümerschaft bei der Tab-Bereinigung](/de/tools/browser#tab-cleanup-ownership).
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` ist deaktiviert, wenn kein Wert festgelegt ist, sodass die Browsernavigation standardmäßig strikt bleibt.
- Legen Sie `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` nur fest, wenn Sie der Browsernavigation in privaten Netzwerken ausdrücklich vertrauen.
- Im strikten Modus unterliegen Endpunkte entfernter CDP-Profile (`profiles.*.cdpUrl`) bei Erreichbarkeits- und Erkennungsprüfungen derselben Blockierung privater Netzwerke.
- `ssrfPolicy.allowPrivateNetwork` wird weiterhin als veralteter Alias unterstützt.
- Verwenden Sie im strikten Modus `ssrfPolicy.hostnameAllowlist` und `ssrfPolicy.allowedHostnames` für explizite Ausnahmen.
- Entfernte Profile können nur angehängt werden (Starten/Stoppen/Zurücksetzen deaktiviert).
- `profiles.*.cdpUrl` akzeptiert `http://`, `https://`, `ws://` und `wss://`.
  Verwenden Sie HTTP(S), wenn OpenClaw `/json/version` ermitteln soll; verwenden Sie WS(S),
  wenn Ihr Provider Ihnen eine direkte DevTools-WebSocket-URL bereitstellt.
- Wenn ein extern verwalteter CDP-Dienst über Loopback erreichbar ist, legen Sie
  für dieses Profil `attachOnly: true` fest; andernfalls behandelt OpenClaw den Loopback-Port als
  lokal verwaltetes Browserprofil und meldet möglicherweise Fehler zur lokalen Port-Eigentümerschaft.
- `existing-session`-Profile verwenden Chrome MCP anstelle von CDP und können auf
  dem ausgewählten Host oder über einen verbundenen Browser-Node angehängt werden.
- `existing-session`-Profile können `userDataDir` festlegen, um ein bestimmtes
  Chromium-basiertes Browserprofil wie Brave oder Edge zu verwenden.
- `existing-session`-Profile können `cdpUrl` festlegen, wenn Chrome bereits
  hinter einem DevTools-HTTP(S)-Erkennungsendpunkt oder einem direkten WS(S)-Endpunkt ausgeführt wird. In diesem
  Modus übergibt OpenClaw den Endpunkt an Chrome MCP, anstatt die automatische Verbindung zu verwenden;
  `userDataDir` wird für die Startargumente von Chrome MCP ignoriert.
- `existing-session`-Profile behalten die aktuellen Einschränkungen der Chrome-MCP-Route bei:
  Snapshot-/Referenz-gesteuerte Aktionen anstelle der Zielauswahl über CSS-Selektoren, Upload-Hooks
  für einzelne Dateien, keine Überschreibungen des Dialog-Timeouts, kein `wait --load networkidle` und keine
  `responsebody`, kein PDF-Export, kein Abfangen von Downloads und keine Stapelaktionen.
- Lokal verwaltete `openclaw`-Profile weisen `cdpPort` und `cdpUrl` automatisch zu; legen Sie
  `cdpUrl` nur für entfernte CDP-Profile oder zum Anhängen an Endpunkte bestehender Sitzungen
  explizit fest.
- Lokal verwaltete Profile können `executablePath` festlegen, um das globale
  `browser.executablePath` für dieses Profil zu überschreiben. Verwenden Sie dies, um ein Profil in
  Chrome und ein anderes in Brave auszuführen.
- Reihenfolge der automatischen Erkennung: Standardbrowser, falls Chromium-basiert → Chrome → Brave → Edge → Chromium → Chrome Canary.
- `browser.executablePath` und `browser.profiles.<name>.executablePath`
  akzeptieren vor dem Start von Chromium sowohl `~` als auch `~/...` für das Home-Verzeichnis Ihres Betriebssystems.
  Das profilspezifische `userDataDir` bei `existing-session`-Profilen wird ebenfalls mit Tilde-Erweiterung verarbeitet.
- Steuerungsdienst: nur Loopback (Port wird aus `gateway.port` abgeleitet, Standardwert `18791`).
- `extraArgs` hängt zusätzliche Start-Flags an den lokalen Chromium-Start an (zum Beispiel
  `--disable-gpu`, Fenstergrößen oder Debug-Flags).

---

## Benutzeroberfläche

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // Emoji, kurzer Text, Bild-URL oder Daten-URI
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // Behält Kommentare nach Ausführungen in der Control UI bei; übermittelt sie nicht an Kanäle
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue; weglassen, um den Warteschlangenmodus des Servers zu verwenden
      showAdvancedSettings: false, // Erweitert jede Gruppe „Advanced“ in den Einstellungen
    },
  },
}
```

- `seamColor`: Akzentfarbe für die Benutzeroberflächenelemente nativer Apps (Farbton der Sprechmodus-Blase usw.).
- `assistant`: Überschreibung der Control-UI-Identität. Fällt auf die Identität des aktiven Agenten zurück.
- `prefs`: geräteübergreifende Bedienereinstellungen. Dies ist der kanonische Speicherort, damit Agenten
  sie über das Genehmigungs-Gate ändern können und alle Control-UI-Clients synchron
  bleiben; Browser spiegeln die Werte für einen sofortigen Start in den lokalen Speicher und behalten
  eine gerätelokale Kopie, wenn sie die Konfiguration nicht schreiben können (Betrachterbereich, offline).
  `chatPersistCommentary` verwendet standardmäßig `true`. Wenn der Wert auf `false` gesetzt wird, bleiben Live-
  Kommentare während einer Ausführung sichtbar, werden jedoch bei deren Abschluss entfernt, und neue
  Codex-Kommentare gelangen nicht in die dauerhafte Transkriptspiegelung. Die Übermittlung an
  Nachrichtenkanäle bleibt davon getrennt und unverändert.
  `showAdvancedSettings` verwendet standardmäßig `false`; die Einstellungssuche kann vorübergehend
  eine passende erweiterte Gruppe öffnen, ohne diese Einstellung zu ändern.
  Reine Darstellungseinstellungen wie Textskalierung, Chatbreite und Live-
  Aktivitäten in der Seitenleiste bleiben browserlokal und werden in den Einstellungen konfiguriert.
  Verbundene Clients übernehmen serverseitige Änderungen live: Das Gateway sendet nach jedem
  persistenten Konfigurationsschreibvorgang ein reines Hash-Ereignis `config.changed`, und
  Clients aktualisieren ihren Snapshot (wird übersprungen, solange ein lokaler Einstellungsentwurf
  ungespeicherte Änderungen enthält). Clients gleichen ihren Zustand beim erneuten Verbinden ab.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // oder OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // für mode=trusted-proxy; siehe /gateway/trusted-proxy-auth
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // optionale KI-Zwecktitel für Tool-Aufrufe (verbraucht Tokens des Utility-Modells)
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // gefährlich: absolute externe http(s)-Einbettungs-URLs zulassen
      // allowedOrigins: ["https://control.example.com"], // für eine Control UI außerhalb von Loopback erforderlich
      // dangerouslyAllowHostHeaderOriginFallback: false, // gefährlicher Fallback-Modus für den Ursprung anhand des Host-Headers
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // Optional. Standardmäßig false.
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // Optional. Standardmäßig nicht festgelegt/deaktiviert.
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // SSH-verifizierte automatische Genehmigung. Standardmäßig aktiviert (true).
        // Auf false setzen, um nur die SSH-Verifizierung zu deaktivieren; dies wirkt sich nicht auf
        // autoApproveCidrs oben aus. Für eine ausschließlich manuelle Node-Kopplung false setzen UND
        // autoApproveCidrs nicht festlegen. Zur Feinabstimmung ein Objekt übergeben: { user, identity,
        // timeoutMs, cidrs }.
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // Zusätzliche HTTP-Ablehnungen für /tools/invoke
      deny: ["browser"],
      // Tools für Eigentümer-/Administratoraufrufer aus der standardmäßigen HTTP-Ablehnungsliste entfernen
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="Details zu Gateway-Feldern">

- `mode`: `local` (Gateway ausführen) oder `remote` (Verbindung mit Remote-Gateway herstellen). Der Gateway verweigert den Start, sofern nicht `local`.
- `port`: einzelner multiplexierter Port für WS + HTTP. Rangfolge: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`, `loopback` (Standard), `lan` (`0.0.0.0`), `tailnet` (Tailscale-IPv4, sofern verfügbar, andernfalls Loopback) oder `custom` (eine IPv4-Adresse). Eine aufgelöste `tailnet`-Adresse und jede `custom`-Adresse außer `127.0.0.1` oder `0.0.0.0` erfordern für Clients auf demselben Host `127.0.0.1` am selben Port; der Start schlägt fehl, wenn einer der Listener keine Bindung herstellen kann. Die Exposition außerhalb des Loopbacks bleibt auf die ausgewählte Schnittstelle beschränkt.
- **Veraltete Bind-Aliasse**: Verwenden Sie Bind-Moduswerte in `gateway.bind` (`auto`, `loopback`, `lan`, `tailnet`, `custom`), keine Host-Aliasse (`0.0.0.0`, `127.0.0.1`, `localhost`, `::`, `::1`).
- **Docker-Hinweis**: Die standardmäßige `loopback`-Bindung lauscht innerhalb des Containers auf `127.0.0.1`. Bei Docker-Bridge-Netzwerken (`-p 18789:18789`) trifft Datenverkehr auf `eth0` ein, sodass der Gateway nicht erreichbar ist. Verwenden Sie `--network host` oder legen Sie `bind: "lan"` (oder `bind: "custom"` mit `customBindHost: "0.0.0.0"`) fest, damit auf allen Schnittstellen gelauscht wird.
- **Authentifizierung**: standardmäßig erforderlich. Bindungen außerhalb des Loopbacks erfordern eine Gateway-Authentifizierung. In der Praxis bedeutet dies ein gemeinsames Token/Passwort oder einen identitätsbewussten Reverse-Proxy mit `gateway.auth.mode: "trusted-proxy"`. Der Onboarding-Assistent generiert standardmäßig ein Token.
- Wenn sowohl `gateway.auth.token` als auch `gateway.auth.password` konfiguriert sind (einschließlich SecretRefs), setzen Sie `gateway.auth.mode` ausdrücklich auf `token` oder `password`. Start sowie Dienstinstallations- und Reparaturabläufe schlagen fehl, wenn beide konfiguriert sind und kein Modus festgelegt ist.
- `gateway.auth.mode: "none"`: expliziter Modus ohne Authentifizierung. Verwenden Sie ihn nur für vertrauenswürdige lokale Loopback-Konfigurationen; er wird in Onboarding-Eingabeaufforderungen absichtlich nicht angeboten.
- `gateway.auth.mode: "trusted-proxy"`: Delegiert die Browser-/Benutzerauthentifizierung an einen identitätsbewussten Reverse-Proxy und vertraut Identitäts-Headern von `gateway.trustedProxies` (siehe [Authentifizierung über vertrauenswürdige Proxys](/de/gateway/trusted-proxy-auth)). Dieser Modus erwartet standardmäßig eine Proxy-Quelle **außerhalb des Loopbacks**; Reverse-Proxys auf demselben Host über Loopback erfordern ausdrücklich `gateway.auth.trustedProxy.allowLoopback = true`. Interne Aufrufer auf demselben Host können `gateway.auth.password` als lokalen direkten Fallback verwenden; `gateway.auth.token` schließt den Trusted-Proxy-Modus weiterhin gegenseitig aus.
- `gateway.auth.allowTailscale`: Wenn `true`, können Identitäts-Header von Tailscale Serve die Control-UI-/WebSocket-Authentifizierung erfüllen (über `tailscale whois` verifiziert). HTTP-API-Endpunkte verwenden diese Tailscale-Header-Authentifizierung **nicht**; sie folgen stattdessen dem normalen HTTP-Authentifizierungsmodus des Gateways. Dieser tokenlose Ablauf setzt voraus, dass der Gateway-Host vertrauenswürdig ist. Ist standardmäßig `true`, wenn `tailscale.mode = "serve"`.
- `gateway.auth.rateLimit`: optionaler Begrenzer für fehlgeschlagene Authentifizierungsversuche. Gilt pro Client-IP und pro Authentifizierungsbereich (gemeinsames Geheimnis und Geräte-Token werden unabhängig voneinander erfasst). Blockierte Versuche geben `429` + `Retry-After` zurück.
  - Im asynchronen Control-UI-Pfad von Tailscale Serve werden fehlgeschlagene Versuche für denselben `{scope, clientIp}` vor dem Schreiben des Fehlers serialisiert. Gleichzeitige fehlerhafte Versuche desselben Clients können daher den Begrenzer bei der zweiten Anfrage auslösen, statt beide aufgrund eines Wettlaufs als einfache Abweichungen passieren zu lassen.
  - `gateway.auth.rateLimit.exemptLoopback` ist standardmäßig `true`; setzen Sie `false`, wenn Sie absichtlich auch den Datenverkehr von localhost begrenzen möchten (für Testkonfigurationen oder strikte Proxy-Bereitstellungen).
- WS-Authentifizierungsversuche mit Browser-Ursprung werden immer gedrosselt, wobei die Loopback-Ausnahme deaktiviert ist (mehrschichtiger Schutz vor Browser-basierten Brute-Force-Angriffen auf localhost).
- Auf dem Loopback werden diese Sperren für Browser-Ursprünge pro normalisiertem `Origin`
  -Wert isoliert, sodass wiederholte Fehler von einem localhost-Ursprung nicht automatisch
  einen anderen Ursprung sperren.
- `tailscale.mode`: `serve` (nur Tailnet, Loopback-Bindung) oder `funnel` (öffentlich, erfordert Authentifizierung).
- `tailscale.serviceName`: optionaler Tailscale-Dienstname für den Serve-Modus, beispielsweise
  `svc:openclaw`. Wenn festgelegt, übergibt OpenClaw ihn an `tailscale serve
--service`, sodass die Control UI über einen benannten Dienst statt
  über den Geräte-Hostnamen bereitgestellt werden kann. Der Wert muss das `svc:<dns-label>`
  -Format für Dienstnamen von Tailscale verwenden; beim Start wird die abgeleitete Dienst-URL ausgegeben.
- `tailscale.preserveFunnel`: Wenn `true` und `tailscale.mode = "serve"`, prüft OpenClaw
  vor dem erneuten Anwenden von Serve beim Start `tailscale funnel status` und überspringt
  es, wenn eine extern konfigurierte Funnel-Route den Gateway-Port bereits abdeckt.
  Standardwert: `false`.
- `controlUi.allowedOrigins`: explizite Zulassungsliste für Browser-Ursprünge bei Gateway-WebSocket-Verbindungen. Erforderlich für öffentliche Browser-Ursprünge außerhalb des Loopbacks. Private UI-Ladevorgänge gleichen Ursprungs im LAN/Tailnet von Loopback-, RFC1918-/Link-Local-, `.local`-, `.ts.net`- oder Tailscale-CGNAT-Hosts werden akzeptiert, ohne den Host-Header-Fallback zu aktivieren.
- `controlUi.toolTitles`: Aktiviert KI-generierte Zweckbezeichnungen für Tool-Aufrufe im Control-UI-Chat. Standard: `false` (die Tool-Darstellung bleibt vollständig deterministisch, ohne Modellaufrufe im Hintergrund). Wenn aktiviert, kennzeichnet die `chat.toolTitles`-Methode komplexe Aufrufe über das standardmäßige Routing für Hilfsmodelle – über `utilityModel` des Agenten (eine Betreiberentscheidung, die wie jede Hilfsaufgabe begrenzte Tool-Argumente an den gewählten Provider senden kann) oder den deklarierten Standard für kleine Modelle des Session-Providers (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`) – und speichert Ergebnisse in der agentenspezifischen Zustandsdatenbank zwischen, sodass wiederholte Ansichten nie erneut abgerechnet werden. `utilityModel: \"\"` deaktiviert Bezeichnungen wie bei jeder anderen Hilfsaufgabe; Bezeichnungen greifen nie auf das primäre Modell zurück.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: gefährlicher Modus, der den Host-Header-Ursprungs-Fallback für Bereitstellungen aktiviert, die absichtlich auf einer Host-Header-Ursprungsrichtlinie beruhen.
- `terminal.enabled`: Aktiviert das auf Administratoren beschränkte Betreiberterminal. Standard: `false`. Das Terminal startet ein Host-PTY im ausgewählten Agenten-Arbeitsbereich, übernimmt die Umgebung des Gateway-Prozesses und wird für Agenten mit `sandbox.mode: "all"` verweigert. Aktivieren Sie es nur für vertrauenswürdige Betreiberbereitstellungen; eine Änderung startet den Gateway neu und aktualisiert die Content-Security-Policy der Control UI.
- `terminal.shell`: optionale ausführbare Shell-Datei. Wenn nicht festgelegt, verwendet OpenClaw unter Unix `$SHELL` und unter Windows `%ComSpec%`.
- `terminal.detachedSessionTimeoutSeconds`: Zeitraum, den eine Terminalsitzung nach dem Verbindungsabbruch (Neuladen der Seite, Ruhezustand des Laptops) bestehen bleibt und über `terminal.attach` erneut verbunden werden kann, wobei ihre jüngsten Ausgaben wiedergegeben werden. Standard: `300`. Setzen Sie `0`, um Sitzungen sofort zu beenden, sobald ihre Verbindung abbricht. Getrennte Sitzungen führen ihre Befehle weiter aus; verkürzen Sie diesen Zeitraum daher auf gemeinsam genutzten oder exponierten Hosts.
- `remote.transport`: `ssh` (Standard) oder `direct` (ws/wss). Für `direct` muss `remote.url` bei öffentlichen Hosts `wss://` sein; unverschlüsseltes `ws://` wird nur für Loopback-, LAN-, Link-Local-, `.local`-, `.ts.net`- und Tailscale-CGNAT-Hosts akzeptiert.
- `remote.remotePort`: Gateway-Port auf dem entfernten SSH-Host. Standardmäßig `18789`; verwenden Sie dies, wenn sich der lokale Tunnel-Port vom Port des entfernten Gateways unterscheidet.
- `remote.tlsFingerprint`: erwarteter SHA-256-Zertifikatfingerabdruck für einen entfernten `wss://`-Gateway. Die macOS-App wendet ihn sowohl auf Betreiber-/Steuerungsverbindungen als auch auf Verbindungen zu Companion-Nodes an. Ohne einen expliziten Wert zeichnet macOS eine TOFU-Pinierung erst auf, nachdem die normale Systemvertrauensprüfung erfolgreich war.
- `remote.sshHostKeyPolicy`: Richtlinie für SSH-Tunnel-Hostschlüssel unter macOS. `strict` ist der Standard und erfordert einen bereits vertrauenswürdigen Schlüssel. `openssh` ist eine explizite Zustimmung zur wirksamen OpenSSH-Konfiguration für verwaltete Aliasse; überprüfen Sie vor der Verwendung die entsprechenden SSH-Einstellungen des Benutzers und des Systems. Die macOS-App und `configure-remote` setzen diese Richtlinie beim Wechseln von Zielen auf `strict` zurück, sofern nicht erneut ausdrücklich zugestimmt wird.
- `gateway.remote.token` / `.password` sind Anmeldedatenfelder für Remote-Clients. Sie konfigurieren die Gateway-Authentifizierung nicht selbstständig.
- `gateway.push.apns.relay.baseUrl`: HTTPS-Basis-URL für das externe APNs-Relay, das verwendet wird, nachdem Relay-gestützte iOS-Builds Registrierungen am Gateway veröffentlicht haben. Öffentliche App-Store-Builds verwenden das gehostete OpenClaw-Relay. Benutzerdefinierte Relay-URLs müssen zu einem bewusst separaten iOS-Build-/Bereitstellungspfad passen, dessen Relay-URL auf dieses Relay verweist.
- `gateway.push.apns.relay.timeoutMs`: Zeitüberschreitung für das Senden vom Gateway zum Relay in Millisekunden. Standardmäßig `10000`.
- Relay-gestützte Registrierungen werden an eine bestimmte Gateway-Identität delegiert. Die gekoppelte iOS-App ruft `gateway.identity.get` ab, schließt diese Identität in die Relay-Registrierung ein und leitet eine registrierungsbezogene Sendeberechtigung an den Gateway weiter. Ein anderer Gateway kann diese gespeicherte Registrierung nicht wiederverwenden.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: temporäre Umgebungsüberschreibungen für die oben genannte Relay-Konfiguration.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: ausschließlich für die Entwicklung vorgesehener Ausweg für Loopback-HTTP-Relay-URLs. Produktions-Relay-URLs sollten HTTPS verwenden.
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: optionale Umgebungsüberschreibung für die integrierte Zeitüberschreitung des Gateway-WebSocket-Handshakes vor der Authentifizierung.
- `channels.<provider>.healthMonitor.enabled`: kanalspezifische Deaktivierung von Neustarts durch die Zustandsüberwachung bei weiterhin aktivierter globaler Überwachung.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: kontospezifische Überschreibung für Kanäle mit mehreren Konten. Wenn festgelegt, hat sie Vorrang vor der Überschreibung auf Kanalebene.
- Lokale Gateway-Aufrufpfade können `gateway.remote.*` nur dann als Fallback verwenden, wenn `gateway.auth.*` nicht festgelegt ist.
- Wenn `gateway.auth.token` / `gateway.auth.password` ausdrücklich über SecretRef konfiguriert und nicht aufgelöst ist, schlägt die Auflösung geschlossen fehl (keine Verschleierung durch einen Remote-Fallback).
- `trustedProxies`: IP-Adressen von Reverse-Proxys, die TLS terminieren oder Header für weitergeleitete Clients einfügen. Führen Sie nur Proxys auf, die Sie kontrollieren. Loopback-Einträge sind für Proxy-/Lokalerkennungskonfigurationen auf demselben Host weiterhin gültig (beispielsweise Tailscale Serve oder ein lokaler Reverse-Proxy), machen Loopback-Anfragen jedoch **nicht** für `gateway.auth.mode: "trusted-proxy"` zulässig.
- `allowRealIpFallback`: Wenn `true`, akzeptiert der Gateway `X-Real-IP`, falls `X-Forwarded-For` fehlt. Standardmäßig `false` für ein geschlossenes Fehlerverhalten.
- `gateway.nodes.pairing.autoApproveCidrs`: optionale CIDR-/IP-Zulassungsliste zur automatischen Genehmigung der erstmaligen Kopplung eines Node-Geräts ohne angeforderte Bereiche. Sie ist deaktiviert, wenn sie nicht festgelegt ist. Dadurch werden weder Betreiber-/Browser-/Control-UI-/WebChat-Kopplungen noch Rollen-, Bereichs-, Metadaten- oder Public-Key-Upgrades automatisch genehmigt.
- `gateway.nodes.pairing.sshVerify`: SSH-verifizierte automatische Genehmigung für die erstmalige Kopplung eines Node-Geräts (Standard: aktiviert). Der Gateway stellt per SSH eine Rückverbindung zum Kopplungs-Host her (BatchMode, strikte Hostschlüssel) und genehmigt nur bei einer exakten Übereinstimmung des `openclaw node identity`-Geräteschlüssels. Es gilt dieselbe Mindestvoraussetzung wie für `autoApproveCidrs`; Prüfungen sind auf private/CGNAT-Quelladressen beschränkt, sofern `cidrs` sie nicht überschreibt. Setzen Sie `false` zum Deaktivieren oder `{ user, identity, timeoutMs, cidrs }` zur Feinabstimmung. Siehe [Node-Kopplung](/de/gateway/pairing#ssh-verified-device-auto-approval-default).
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: globale Zulässigkeits-/Sperrsteuerung für deklarierte Node-Befehle nach der Kopplung und Auswertung der Plattform-Zulassungsliste. Verwenden Sie `commands.allow`, um gefährliche Node-Befehle wie `camera.snap`, `camera.clip`, `screen.record`, `health.summary`, `sms.search` und `sms.send` ausdrücklich zuzulassen; `commands.deny` entfernt einen Befehl, selbst wenn er andernfalls durch eine Plattformvorgabe oder eine ausdrückliche Zulassung aufgenommen würde. Die iOS-Health-Berechtigung, die Android-SMS-Berechtigung und die Gateway-Befehlsautorisierung sind voneinander unabhängig. Nachdem ein Node seine Liste deklarierter Befehle geändert hat, müssen Sie die Gerätekopplung ablehnen und erneut genehmigen, damit das Gateway den aktualisierten Befehlsschnappschuss speichert.
- `gateway.tools.deny`: zusätzliche Tool-Namen, die für HTTP `POST /tools/invoke` gesperrt sind (erweitert die standardmäßige Sperrliste).
- `gateway.tools.allow`: entfernt Tool-Namen aus der standardmäßigen HTTP-Sperrliste für
  Aufrufer mit Eigentümer-/Administratorrechten. Dadurch erhalten identitätstragende `operator.write`-
  Aufrufer keinen Eigentümer-/Administratorzugriff; `cron`, `gateway` und `nodes` bleiben
  für Aufrufer ohne Eigentümerrechte auch dann nicht verfügbar, wenn sie auf der Zulassungsliste stehen.

</Accordion>

### OpenAI-kompatible Endpunkte

- Admin-HTTP-RPC: standardmäßig deaktiviert, ebenso wie das `admin-http-rpc`-Plugin. Aktivieren Sie das Plugin, um `POST /api/v1/admin/rpc` zu registrieren. Siehe [Admin-HTTP-RPC](/de/plugins/admin-http-rpc).
- Chat Completions: standardmäßig deaktiviert. Aktivieren Sie sie mit `gateway.http.endpoints.chatCompletions.enabled: true`.
- Responses API: `gateway.http.endpoints.responses.enabled`.
- Absicherung der URL-Eingabe für Responses:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    Leere Zulassungslisten werden als nicht festgelegt behandelt; verwenden Sie `gateway.http.endpoints.responses.files.allowUrl=false`
    und/oder `gateway.http.endpoints.responses.images.allowUrl=false`, um den URL-Abruf zu deaktivieren.
- Optionaler Header zur Absicherung von Antworten:
  - `gateway.http.securityHeaders.strictTransportSecurity` (nur für von Ihnen kontrollierte HTTPS-Ursprünge festlegen; siehe [Authentifizierung über vertrauenswürdige Proxys](/de/gateway/trusted-proxy-auth#tls-termination-and-hsts))

### Isolierung mehrerer Instanzen

Führen Sie mehrere Gateways auf einem Host mit eindeutigen Ports und Zustandsverzeichnissen aus:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

Komfortoptionen: `--dev` (verwendet `~/.openclaw-dev` + Port `19001`), `--profile <name>` (verwendet `~/.openclaw-<name>`).

Siehe [Mehrere Gateways](/de/gateway/multiple-gateways).

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: aktiviert die TLS-Terminierung am Gateway-Listener (HTTPS/WSS) (Standard: `false`).
- `autoGenerate`: generiert automatisch ein lokales selbstsigniertes Zertifikat/Schlüsselpaar, wenn keine expliziten Dateien konfiguriert sind; nur für lokale Entwicklungsumgebungen.
- `certPath`: Dateisystempfad zur TLS-Zertifikatsdatei.
- `keyPath`: Dateisystempfad zur privaten TLS-Schlüsseldatei; beschränken Sie die Zugriffsberechtigungen.
- `caPath`: optionaler Pfad zum CA-Bundle für die Clientüberprüfung oder benutzerdefinierte Vertrauensketten.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: steuert, wie Konfigurationsänderungen zur Laufzeit angewendet werden.
  - `"off"`: ignoriert Live-Änderungen; Änderungen erfordern einen expliziten Neustart.
  - `"restart"`: startet den Gateway-Prozess bei einer Konfigurationsänderung immer neu.
  - `"hot"`: wendet Änderungen prozessintern ohne Neustart an.
  - `"hybrid"` (Standard): versucht zuerst einen Hot Reload; greift bei Bedarf auf einen Neustart zurück.
- `debounceMs`: Entprellzeitfenster in ms, bevor Konfigurationsänderungen angewendet werden (nicht negative Ganzzahl; Standard: `300`).
- `deferralTimeoutMs`: optionale maximale Wartezeit in ms für laufende Vorgänge, bevor ein Neustart oder ein Hot Reload des Kanals erzwungen wird. Lassen Sie den Wert weg, um die standardmäßige begrenzte Wartezeit (`300000`) zu verwenden; setzen Sie `0`, um unbegrenzt zu warten und regelmäßig Warnungen über noch ausstehende Vorgänge zu protokollieren.

---

## Cloud-Worker-Umgebungen

Cloud-Worker sind optional. Wenn `cloudWorkers` fehlt oder `profiles` leer ist, akzeptiert OpenClaw keine Erstellung neuer Worker. Zuvor erstellte dauerhafte Datensätze werden weiterhin abgeglichen und bleiben sichtbar; die bestehende Gateway-/Node-Projektion bleibt unverändert.

Jeder Worker-Provider muss einen SSH-`hostKey` aus einer vertrauenswürdigen Bereitstellungsausgabe exakt als `algorithm base64` zurückgeben, ohne Hostnamen oder Kommentar. Der Bootstrap schreibt diesen Schlüssel in eine isolierte `known_hosts`-Datei, verwendet `StrictHostKeyChecking=yes` und schlägt vor dem Aufbau einer Verbindung fehl, wenn der Provider ihn nicht bereitstellt. Es gibt keinen Fallback nach dem Prinzip „Trust on First Use“.

Die Einrichtung des Tunnels erfolgt bei Bedarf und nicht als Teil der Bereitstellung. Nach dem Start leitet das Gateway einen Worker-lokalen Unix-Socket rückwärts an seinen Loopback-WebSocket-Endpunkt weiter. Der Socket befindet sich in einem zufällig zugewiesenen, ausschließlich für den Eigentümer zugänglichen Remote-Verzeichnis; anders als ein Loopback-TCP-Port ist er für andere Konten auf einem Mehrbenutzer-Worker nicht erreichbar und kann nicht mit dem Port einer anderen Umgebung kollidieren. SSH-Keepalives und ein begrenzter exponentieller Backoff für erneute Verbindungen werden nur ausgeführt, solange der Eigentümer des Tunnels aktuell bleibt. Beim Stoppen des Tunnels werden erneute Verbindungen gesperrt, bevor der SSH-Prozess geschlossen wird.

Steuerdatenverkehr und Workspace-Übertragung verwenden separate SSH-Verbindungen. Beide verwenden dieselbe aufgelöste Identität und isolierte angeheftete `known_hosts`-Datei erneut, aber die Workspace-Übertragung teilt sich das SSH-Verbindungs-Multiplexing nicht mit dem langlebigen Tunnel, sodass rsync den Steuerdatenverkehr nicht blockieren kann.

### Crabbox-Profil

Der gebündelte `crabbox`-Provider stellt über die lokale Crabbox-CLI eine SSH-fähige Lease bereit. Der innere `settings.provider` wählt das Crabbox-Backend aus; er ist von der äußeren OpenClaw-Provider-ID getrennt.

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // Standard; verwenden Sie "npm" nur für eine veröffentlichte Gateway-Version.
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // Optionaler absoluter Pfad. Standard: benachbartes ../crabbox/bin/crabbox, dann PATH.
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider` (erforderlich): Crabbox-Backend, das über `--provider` übergeben wird. Verwenden Sie ein Backend, dessen Inspektionsausgabe einen SSH-Endpunkt enthält; `aws` wählt das direkte AWS-Backend aus.
- `settings.class` (erforderlich): Crabbox-Maschinenklasse, die an `--class` übergeben wird.
- `settings.ttl` und `settings.idleTimeout` (erforderlich): positive Go-Zeitdauerzeichenfolgen, die an `--ttl` und `--idle-timeout` übergeben werden. Diese Provider-seitigen Ausfallsicherungen unterscheiden sich von der unten gespeicherten `lifetime`-Richtlinie von OpenClaw.
- `settings.binary`: optionaler absoluter Pfad zur ausführbaren Crabbox-Datei. Ohne diesen prüft OpenClaw zunächst den benachbarten Crabbox-Checkout, dann ausführbare Einträge in `PATH` und ruft schließlich `crabbox` auf, damit eine fehlende CLI als sichtbarer Provider-Fehler erhalten bleibt.

Unbekannte Einstellungen werden abgelehnt. Crabbox-Anmeldedaten und Backend-spezifische Kontokonfigurationen verbleiben im Besitz von Crabbox; legen Sie sie nicht in `settings` ab. OpenClaw ruft ausschließlich die lokale CLI auf und führt von diesem Plugin aus keine Provider-Netzwerkaufrufe durch. Die Bereitstellung übergibt immer `--keep=true`; OpenClaw ist für den externen Lebenszyklus zuständig und zerstört die Lease mit `crabbox stop`.

<Note>
  OpenClaw löst den Lease-lokalen `sshKey`-Pfad von Crabbox über die Provider-eigene Geheimnisauflösung auf und heftet den maßgeblichen `sshHostKey` an, den `crabbox inspect --json` zurückgibt. Die AWS-Zulassung erfordert außerdem `providerMetadata.instanceProfileAttached`. Installieren Sie Crabbox 0.38.1 oder neuer für diesen geschlossenen Inspektionsvertrag.
</Note>

### Statisches SSH-Entwicklungsprofil

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: benannte Worker-Profile mit nicht leeren, von Leerraum bereinigten IDs. Jedes Profil wählt einen durch ein Plugin registrierten Provider aus.
- `provider`: nicht leere Worker-Provider-ID. Die Beispiele verwenden den gebündelten `crabbox`-Provider und den QA-Lab-Provider `static-ssh`.
- `install`: Installationsmethode des Workers. `"bundle"` (Standard) überträgt ein inhaltsgehashtes Bundle des installierten Gateway-Builds und unterstützt veröffentlichte, Entwicklungs- und unveröffentlichte Versionen. `"npm"` ist eine optionale Optimierung für ein unverändertes paketiertes Release; sie installiert `openclaw@<exact gateway version>` aus der öffentlichen npm-Registry und installiert niemals `latest`.
- Gebündelte Provider-Plugins werden bei entsprechender Konfiguration automatisch ausgewählt, explizite Deaktivierungen und `plugins.allow` gelten jedoch weiterhin. Nehmen Sie die Provider-ID (zum Beispiel `crabbox`) auf, wenn eine Zulassungsliste konfiguriert ist. Externe Provider-Plugins müssen ebenfalls installiert und explizit aktiviert werden.
- `settings`: Provider-eigenes begrenztes JSON. Das ausgewählte Plugin definiert und validiert seine Schlüssel; verwenden Sie [SecretRef-Objekte](/de/gateway/secrets) für Werte, die Geheimnisse enthalten. Der statische SSH-Provider erfordert `host`, `user`, `hostKey` und `keyRef`; `port` verwendet standardmäßig `22`. `hostKey` muss eine einzelne Zeile eines öffentlichen OpenSSH-Hostschlüssels (`algorithm base64`) sein, die vom bekannten Host oder über einen anderen vertrauenswürdigen Kanal bezogen wurde, ohne vorangestellte Optionen.
- `lifetime.idleTimeoutMinutes`: positive ganzzahlige Minuten, die für eine spätere Richtlinie zur Rückgewinnung bei Inaktivität gespeichert werden.
- `lifetime.maxLifetimeMinutes`: positive ganzzahlige Minuten, die für eine spätere Lebenszyklusrichtlinie gespeichert werden.

Eine unterstützte Node-Laufzeitumgebung (22.22.3+, 24.15+ oder 25.9+) mit WAL-Reset-sicherem SQLite muss bereits auf dem Worker installiert sein. Die optionale `"npm"`-Methode erfordert außerdem `npm` und ausgehenden HTTPS-Zugriff auf die öffentliche npm-Registry. Die Einrichtung vernetzter Toolchains ist eine Provider-Richtlinie; der Bootstrap meldet einen umsetzbaren Fehler, statt Toolchains selbst zu installieren.

Diese Grundlage installiert und überprüft den Gateway-Build und stellt den Lebenszyklus zum Starten und Stoppen des Tunnels bereit, startet jedoch nicht die allgemeine OpenClaw-CLI. Der eigenständige Worker-Einstiegspunkt und die Schleife folgen im nächsten Cloud-Worker-Meilenstein.

Jeder dauerhafte Umgebungsdatensatz behält seine validierten Provider-Einstellungen, die aufgelöste Installationsmethode und die Lebenszyklusrichtlinie in einem bei der Erstellung angelegten Profilsnapshot. Das Ändern oder Entfernen eines benannten Profils wirkt sich auf neue Erstellungen aus; bestehende Datensätze setzen den Lebenszyklusabgleich mit diesem Snapshot fort, sofern das zuständige Plugin verfügbar bleibt.

Lebensdauerwerte sind im ersten Cloud-Worker-Release lediglich Daten; die automatische Durchsetzung folgt mit späteren Lebenszyklusarbeiten. Profiländerungen erfordern einen Neustart des Gateways.

<Warning>
  Der `static-ssh`-Provider ist ein QA-Lab-Entwicklungsharness für den Quellbaum und ist von paketierten Distributionen ausgeschlossen. Ein Worker, der auf seinem gemeinsam genutzten Host ausgeführt wird, kann nicht zugehörige Hostdaten lesen; verwenden Sie diesen Provider daher nicht als Isolationsgrenze für Produktionsumgebungen.
  Der Betreiber muss den erwarteten `hostKey` bereitstellen; OpenClaw erlernt oder akzeptiert bei der ersten Verbindung keinen Schlüssel.
  Das Zerstören seiner Lease gibt nur den logischen Datensatz von OpenClaw frei; der Host wird dadurch weder gestoppt noch bereinigt.
</Warning>

---

## Hooks

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "Von: {{messages[0].from}}\nBetreff: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

Authentifizierung: `Authorization: Bearer <token>` oder `x-openclaw-token: <token>`.
Hook-Token in Abfragezeichenfolgen werden abgelehnt.

Hinweise zu Validierung und Sicherheit:

- `hooks.enabled=true` erfordert einen nicht leeren `hooks.token`.
- `hooks.token` sollte sich von der aktiven Shared-Secret-Authentifizierung des Gateways (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` oder `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) unterscheiden; beim Start wird eine nicht schwerwiegende Sicherheitswarnung protokolliert, wenn eine Wiederverwendung erkannt wird.
- `openclaw security audit` kennzeichnet die Wiederverwendung der Hook-/Gateway-Authentifizierung als kritischen Befund, einschließlich einer Gateway-Passwortauthentifizierung, die nur zum Zeitpunkt des Audits angegeben wird (`--auth password --password <password>`). Führen Sie `openclaw doctor --fix` aus, um einen dauerhaft gespeicherten, wiederverwendeten `hooks.token` zu rotieren, und aktualisieren Sie anschließend externe Hook-Absender, damit sie das neue Hook-Token verwenden.
- `hooks.path` darf nicht `/` sein; verwenden Sie einen dedizierten Unterpfad wie `/hooks`.
- Falls `hooks.allowRequestSessionKey=true`, beschränken Sie `hooks.allowedSessionKeyPrefixes` (zum Beispiel `["hook:"]`).
- Wenn eine Zuordnung oder Voreinstellung einen vorlagenbasierten `sessionKey` verwendet, legen Sie `hooks.allowedSessionKeyPrefixes` und `hooks.allowRequestSessionKey=true` fest. Statische Zuordnungsschlüssel erfordern diese ausdrückliche Aktivierung nicht.

**Endpunkte:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - `sessionKey` aus der Anfragenutzlast wird nur akzeptiert, wenn `hooks.allowRequestSessionKey=true` (Standard: `false`).
- `POST /hooks/<name>` → wird über `hooks.mappings` aufgelöst
  - Durch Vorlagen gerenderte `sessionKey`-Werte der Zuordnung werden als extern bereitgestellt behandelt und erfordern ebenfalls `hooks.allowRequestSessionKey=true`.

<Accordion title="Zuordnungsdetails">

- `match.path` gleicht den Unterpfad nach `/hooks` ab (z. B. `/hooks/gmail` → `gmail`).
- `match.source` gleicht bei generischen Pfaden ein Nutzlastfeld ab.
- Vorlagen wie `{{messages[0].subject}}` lesen aus der Nutzlast.
- `transform` kann auf ein JS-/TS-Modul verweisen, das eine Hook-Aktion zurückgibt.
  - `transform.module` muss ein relativer Pfad sein und innerhalb von `hooks.transformsDir` bleiben (absolute Pfade und Verzeichnisdurchquerung werden abgelehnt).
  - Speichern Sie `hooks.transformsDir` unter `~/.openclaw/hooks/transforms`; Skill-Verzeichnisse im Arbeitsbereich werden abgelehnt. Wenn `openclaw doctor` diesen Pfad als ungültig meldet, verschieben Sie das Transformationsmodul in das Transformationsverzeichnis der Hooks oder entfernen Sie `hooks.transformsDir`.
- `agentId` leitet an einen bestimmten Agenten weiter; unbekannte IDs greifen auf den Standardagenten zurück.
- `allowedAgentIds`: beschränkt das effektive Agenten-Routing, einschließlich des Standardagentenpfads, wenn `agentId` nicht angegeben ist (`*` oder nicht angegeben = alle zulassen, `[]` = alle ablehnen).
- `defaultSessionKey`: optionaler fester Sitzungsschlüssel für Hook-Agentenläufe ohne expliziten `sessionKey`.
- `allowRequestSessionKey`: erlaubt `/hooks/agent`-Aufrufern und vorlagengesteuerten Sitzungsschlüsseln der Zuordnung, `sessionKey` festzulegen (Standard: `false`).
- `allowedSessionKeyPrefixes`: optionale Präfix-Zulassungsliste für explizite `sessionKey`-Werte (Anfrage + Zuordnung), z. B. `["hook:"]`. Sie wird erforderlich, sobald eine Zuordnung oder Voreinstellung einen vorlagenbasierten `sessionKey` verwendet.
- `deliver: true` sendet die endgültige Antwort an einen Kanal; `channel` verwendet standardmäßig `last`.
- `model` überschreibt das LLM für diesen Hook-Lauf (muss zulässig sein, wenn der Modellkatalog festgelegt ist).

</Accordion>

### Gmail-Integration

- Die integrierte Gmail-Voreinstellung verwendet `sessionKey: "hook:gmail:{{messages[0].id}}"`.
- Dieser nachrichtenspezifische Schlüssel isoliert den Gesprächskontext, nicht die Tools oder den Zugriff auf den Arbeitsbereich. Ohne eine benutzerdefinierte Zuordnung, die `agentId` festlegt, verwendet die Voreinstellung den Standardagenten.
- Leiten Sie Gmail bei nicht vertrauenswürdigen Posteingängen an einen dedizierten Leseagenten weiter und beschränken Sie diesen Agenten mit einer [agentenspezifischen Sandbox- und Tool-Richtlinie](/de/tools/multi-agent-sandbox-tools). Wenn der Leseagent den Hauptagenten benachrichtigen muss, beschränken Sie die Übergabe mit [`tools.agentToAgent`](/de/gateway/config-tools#toolsagenttoagent). Das empfohlene Bedrohungsmodell und die Modellstufe finden Sie unter [Prompt-Injection](/de/gateway/security#prompt-injection).
- Wenn Sie dieses nachrichtenspezifische Routing beibehalten, legen Sie `hooks.allowRequestSessionKey: true` fest und beschränken Sie `hooks.allowedSessionKeyPrefixes` so, dass es dem Gmail-Namensraum entspricht, zum Beispiel `["hook:", "hook:gmail:"]`.
- Wenn Sie `hooks.allowRequestSessionKey: false` benötigen, überschreiben Sie die Voreinstellung mit einem statischen `sessionKey` anstelle des vorlagenbasierten Standards.

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- Das Gateway startet `gog gmail watch serve` beim Systemstart automatisch, wenn es konfiguriert ist. Legen Sie `OPENCLAW_SKIP_GMAIL_WATCHER=1` fest, um dies zu deaktivieren.
- Führen Sie nicht parallel zum Gateway einen separaten `gog gmail watch serve` aus.

---

## Host des Canvas-Plugins

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // oder OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- Stellt vom Agenten bearbeitbares HTML/CSS/JS und A2UI über HTTP am Gateway-Port bereit:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- Nur lokal: Behalten Sie `gateway.bind: "loopback"` bei (Standard).
- Bei Bindungen außerhalb der Loopback-Schnittstelle erfordern Canvas-Routen eine Gateway-Authentifizierung (Token/Passwort/vertrauenswürdiger Proxy), genau wie andere HTTP-Oberflächen des Gateways.
- Node-WebViews senden normalerweise keine Authentifizierungs-Header; nachdem eine Node gekoppelt und verbunden wurde, stellt das Gateway für den Zugriff auf Canvas/A2UI funktionsspezifische URLs mit Node-Gültigkeitsbereich bereit.
- Funktionsspezifische URLs sind an die aktive WS-Sitzung der Node gebunden und laufen schnell ab. Ein IP-basierter Fallback wird nicht verwendet.
- Fügt einen Live-Reload-Client in das bereitgestellte HTML ein.
- Erstellt bei leerem Verzeichnis automatisch eine anfängliche `index.html`.
- Stellt A2UI außerdem unter `/__openclaw__/a2ui/` bereit.
- Änderungen erfordern einen Neustart des Gateways.
- Deaktivieren Sie Live Reload bei großen Verzeichnissen oder `EMFILE`-Fehlern.

---

## Erkennung

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (Standard): `cliPath` + `sshPort` aus TXT-Einträgen auslassen.
- `full`: `cliPath` + `sshPort` einbeziehen; für Multicast-Ankündigungen im LAN muss das gebündelte `bonjour`-Plugin weiterhin aktiviert sein.
- `off`: unterdrückt Multicast-Ankündigungen im LAN, ohne die Plugin-Aktivierung zu ändern.
- Das gebündelte `bonjour`-Plugin startet auf macOS-Hosts automatisch und muss unter Linux, Windows sowie bei containerisierten Gateway-Bereitstellungen ausdrücklich aktiviert werden.
- Der Hostname verwendet standardmäßig den System-Hostnamen, wenn dieser eine gültige DNS-Bezeichnung ist, andernfalls `openclaw`. Überschreiben Sie ihn mit `OPENCLAW_MDNS_HOSTNAME`.
- `OPENCLAW_DISABLE_BONJOUR=1` deaktiviert mDNS-Ankündigungen vollständig und überschreibt `discovery.mdns.mode`.

### Netzwerkübergreifend (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

Schreibt eine Unicast-DNS-SD-Zone unter `~/.openclaw/dns/`. Kombinieren Sie dies für die netzwerkübergreifende Erkennung mit einem DNS-Server (CoreDNS empfohlen) und Tailscale-Split-DNS.

Einrichtung: `openclaw dns setup --apply`.

---

## Umgebung

### `env` (Inline-Umgebungsvariablen)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- Inline-Umgebungsvariablen werden nur angewendet, wenn der Schlüssel in der Prozessumgebung fehlt.
- `.env`-Dateien: `.env` im aktuellen Arbeitsverzeichnis + `~/.openclaw/.env` (keine der beiden überschreibt vorhandene Variablen).
- `shellEnv`: importiert fehlende erwartete Schlüssel aus dem Profil Ihrer Anmelde-Shell.
- Die vollständige Rangfolge finden Sie unter [Umgebung](/de/help/environment).

### Ersetzung von Umgebungsvariablen

Verweisen Sie in einer beliebigen Konfigurationszeichenfolge mit `${VAR_NAME}` auf Umgebungsvariablen:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- Es werden nur großgeschriebene Namen abgeglichen: `[A-Z_][A-Z0-9_]*`.
- Fehlende/leere Variablen lösen beim Laden der Konfiguration einen Fehler aus.
- Maskieren Sie mit `$${VAR}`, um ein wörtliches `${VAR}` zu erhalten.
- Funktioniert mit `$include`.

---

## Geheimnisse

Geheimnisreferenzen sind additiv: Klartextwerte funktionieren weiterhin.

### `SecretRef`

Verwenden Sie eine der folgenden Objektstrukturen:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

Validierung:

- `provider`-Muster: `^[a-z][a-z0-9_-]{0,63}$`
- `source: "env"`-ID-Muster: `^[A-Z][A-Z0-9_]{0,127}$`
- `source: "file"`-ID: absoluter JSON-Zeiger (zum Beispiel `"/providers/openai/apiKey"`)
- `source: "exec"`-ID-Muster: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (unterstützt AWS-ähnliche `secret#json_key`-Selektoren)
- `source: "exec"`-IDs dürfen keine durch Schrägstriche getrennten Pfadsegmente `.` oder `..` enthalten (zum Beispiel wird `a/../b` abgelehnt)

### Unterstützte Anmeldedatenoberfläche

- Kanonische Matrix: [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface)
- `secrets apply` zielt auf unterstützte `openclaw.json`-Anmeldedatenpfade ab.
- `auth-profiles.json`-Referenzen werden in die Laufzeitauflösung und Audit-Abdeckung einbezogen.

### Konfiguration der Geheimnis-Provider

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // optionaler expliziter Umgebungs-Provider
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

Hinweise:

- `file`-Provider unterstützt `mode: "json"` und `mode: "singleValue"` (`id` muss im singleValue-Modus `"value"` sein).
- Pfade von Datei- und Exec-Providern schlagen sicher fehl, wenn die Überprüfung von Windows-ACLs nicht verfügbar ist. Legen Sie `allowInsecurePath: true` nur für vertrauenswürdige Pfade fest, die nicht überprüft werden können.
- `exec`-Provider erfordert einen absoluten `command`-Pfad und verwendet Protokollnutzlasten über stdin/stdout.
- Standardmäßig werden symbolische Links als Befehlspfade abgelehnt. Legen Sie `allowSymlinkCommand: true` fest, um Pfade mit symbolischen Links zuzulassen und gleichzeitig den aufgelösten Zielpfad zu validieren.
- Wenn `trustedDirs` konfiguriert ist, wird die Prüfung des vertrauenswürdigen Verzeichnisses auf den aufgelösten Zielpfad angewendet.
- Die untergeordnete Umgebung von `exec` ist standardmäßig minimal; übergeben Sie erforderliche Variablen explizit mit `passEnv`.
- Geheimnisreferenzen werden zum Aktivierungszeitpunkt in einen In-Memory-Snapshot aufgelöst; anschließend lesen Anfragepfade ausschließlich diesen Snapshot.
- Während der Aktivierung wird nach aktiven Oberflächen gefiltert: Nicht aufgelöste Referenzen auf aktivierten Oberflächen lassen den Start bzw. das Neuladen fehlschlagen, während inaktive Oberflächen mit Diagnosemeldungen übersprungen werden.

---

## Authentifizierungsspeicher

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- Agent-spezifische Profile werden unter `<agentDir>/auth-profiles.json` gespeichert.
- `auth-profiles.json` unterstützt für statische Anmeldedatenmodi Referenzen auf Wertebene (`keyRef` für `api_key`, `tokenRef` für `token`).
- Veraltete flache `auth-profiles.json`-Zuordnungen wie `{ "provider": { "apiKey": "..." } }` sind kein Laufzeitformat; `openclaw doctor --fix` schreibt sie als kanonische `provider:default`-API-Schlüsselprofile mit einer `.legacy-flat.*.bak`-Sicherung neu.
- Profile im OAuth-Modus (`auth.profiles.<id>.mode = "oauth"`) unterstützen keine SecretRef-basierten Anmeldedaten für Authentifizierungsprofile.
- Statische Laufzeitanmeldedaten stammen aus im Arbeitsspeicher aufgelösten Momentaufnahmen; veraltete statische `auth.json`-Einträge werden beim Auffinden bereinigt.
- Veraltete OAuth-Importe aus `~/.openclaw/credentials/oauth.json`.
- Siehe [OAuth](/de/concepts/oauth).
- Laufzeitverhalten von Geheimnissen und `audit/configure/apply`-Werkzeuge: [Verwaltung von Geheimnissen](/de/gateway/secrets).

---

## Auditierung

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

Der Gateway zeichnet Auditereignisse **ausschließlich mit Metadaten** für Agent-Ausführungen und Werkzeugaktionen in der gemeinsamen Zustandsdatenbank auf. Metadaten zum Nachrichtenlebenszyklus müssen separat aktiviert werden. Das Protokoll speichert Identität, Zeitangaben, Werkzeugnamen und normalisierte Ergebnisse, jedoch niemals Prompts, Nachrichteninhalte, Werkzeugargumente, Ergebnisse oder unbearbeiteten Fehlertext. Nachrichtenzeilen speichern keine unverarbeiteten Plattformkonto-, Konversations-, Nachrichten- und Ziel-IDs. Sitzungskennungen für Ausführungen und Werkzeuge bleiben zur Korrelation verfügbar und können selbst Plattformkonto- oder Peer-IDs enthalten. Datensätze laufen nach 30 Tagen ab und das Protokoll ist auf 100.000 Zeilen begrenzt. Fragen Sie sie mit [`openclaw audit`](/de/cli/audit) oder dem Gateway-RPC [`audit.activity.list`](/de/gateway/protocol#audit-ledger-rpc) ab. Unter [Auditverlauf](/de/gateway/audit) finden Sie das vollständige Datenmodell, die Datenschutzsemantik und die Abdeckungsgrenzen.

- `enabled`: neue Auditereignisse aufzeichnen (Standard: `true`). Das Protokoll ist standardmäßig aktiviert, da ein erst nach einem Vorfall aktivierter Audit-Trail den Vorfall nicht erklären kann. Das Festlegen von `false` beendet das Einfügen neuer Ereignisse nach dem Neustart des Gateways; vorhandene Datensätze bleiben lesbar, bis sie ablaufen. Bei erneuter Aktivierung wird die Aufzeichnung ab diesem Zeitpunkt fortgesetzt — die Lücke wird nicht nachträglich gefüllt.
- `messages`: Umfang der Nachrichtenmetadaten (Standard: `"off"`). `"direct"` zeichnet nur bekannte direkte Konversationen auf. `"all"` zeichnet zusätzlich Gruppen, Kanäle und unbekannte Konversationsarten auf. Beide Modi bleiben inhaltsfrei und ersetzen unverarbeitete Kennungen durch installationslokale, schlüsselbasierte Pseudonyme, sofern Korrelation verfügbar ist. Diese dienen als Korrelationshilfen und nicht der Anonymisierung; die Zustandsdatenbank speichert den Ableitungsschlüssel, RPC- und CLI-Exporte jedoch nicht.

Der laufende Gateway erfasst `audit.enabled` und `audit.messages` beim Start; starten Sie ihn neu, nachdem Sie eine der beiden Einstellungen geändert haben. Die Nachrichtenabdeckung umfasst derzeit akzeptierte eingehende Nachrichten, die die zentrale Weiterleitung erreichen, sowie eine abschließende Zeile pro ursprünglicher logischer Nutzlast einer ausgehenden Antwort, die die gemeinsame dauerhafte Zustellung erreicht. Plugin-lokale und direkte Sendepfade, die diese gemeinsamen Grenzen umgehen, sind noch nicht abgedeckt. Der begrenzte Hintergrundschreiber arbeitet nach bestem Bemühen und ist kein verlustfreies Compliance-Archiv.

---

## Protokollierung

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- Standardprotokolldatei: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`; benannte Profile verwenden `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`.
- Legen Sie `logging.file` für einen stabilen Pfad fest.
- `consoleLevel` wird auf `debug` angehoben, wenn `--verbose`.
- `maxFileBytes`: maximale Größe der aktiven Protokolldatei in Byte vor der Rotation (positive Ganzzahl; Standard: `104857600` = 100 MB). OpenClaw bewahrt neben der aktiven Datei bis zu fünf nummerierte Archive auf.
- `redactSensitive` / `redactPatterns`: Maskierung nach bestem Bemühen für Konsolenausgaben, Dateiprotokolle, OTLP-Protokolldatensätze und dauerhaft gespeicherten Text aus Sitzungstranskripten. `redactSensitive: "off"` deaktiviert nur diese allgemeine Richtlinie für Protokolle und Transkripte; Sicherheitsoberflächen der Benutzeroberfläche, Werkzeuge und Diagnose entfernen Geheimnisse weiterhin vor der Ausgabe.

---

## Diagnose

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: Hauptschalter für die Instrumentierungsausgabe (Standard: `true`).
- `flags`: Array aus Flag-Zeichenfolgen zum Aktivieren gezielter Protokollausgaben (unterstützt Platzhalter wie `"telegram.*"` oder `"*"`).
- `otel.enabled`: aktiviert die OpenTelemetry-Exportpipeline (Standard: `false`). Die vollständige Konfiguration, den Signalkatalog und das Datenschutzmodell finden Sie unter [OpenTelemetry-Export](/de/gateway/opentelemetry).
- `otel.endpoint`: Collector-URL für den OTel-Export.
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: optionale signalspezifische OTLP-Endpunkte. Wenn sie festgelegt sind, überschreiben sie `otel.endpoint` nur für das jeweilige Signal.
- `otel.protocol`: `"http/protobuf"` (Standard) oder `"grpc"`.
- `otel.headers`: zusätzliche HTTP-/gRPC-Metadatenheader, die mit OTel-Exportanfragen gesendet werden.
- `otel.serviceName`: Dienstname für Ressourcenattribute.
- `otel.traces` / `otel.metrics` / `otel.logs`: Export von Traces, Metriken oder Protokollen aktivieren.
- `otel.logsExporter`: Ziel für den Protokollexport: `"otlp"` (Standard), `"stdout"` für ein JSON-Objekt pro stdout-Zeile oder `"both"`.
- `otel.sampleRate`: Trace-Abtastrate `0`–`1`.
- `otel.flushIntervalMs`: regelmäßiges Intervall zum Leeren der Telemetriedaten in ms.
- `otel.captureContent`: optionale Erfassung unverarbeiteter Inhalte für OTEL-Span-Attribute. Standardmäßig deaktiviert. Der boolesche Wert `true` erfasst Nachrichten- und Werkzeuginhalte außerhalb des Systems; mit der Objektform können Sie `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `systemPrompt` und `toolDefinitions` explizit aktivieren.
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: Umgebungsschalter für die neueste experimentelle Form von GenAI-Inferenz-Spans, einschließlich `{gen_ai.operation.name} {gen_ai.request.model}`-Span-Namen, der Span-Art `CLIENT` und `gen_ai.provider.name` anstelle des veralteten `gen_ai.system`. Standardmäßig behalten Spans aus Kompatibilitätsgründen `openclaw.model.call` und `gen_ai.system` bei; GenAI-Metriken verwenden begrenzte semantische Attribute.
- `OPENCLAW_OTEL_PRELOADED=1`: Umgebungsschalter für Hosts, die bereits ein globales OpenTelemetry-SDK registriert haben. OpenClaw überspringt dann den Start und das Herunterfahren des Plugin-eigenen SDK, während die Diagnose-Listener aktiv bleiben.
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` und `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: signalspezifische Endpunkt-Umgebungsvariablen, die verwendet werden, wenn der entsprechende Konfigurationsschlüssel nicht festgelegt ist.
- `cacheTrace.enabled`: Cache-Trace-Momentaufnahmen für eingebettete Ausführungen protokollieren (Standard: `false`).
- `cacheTrace.filePath`: Ausgabepfad für Cache-Trace-JSONL (Standard: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: steuern, was in der Cache-Trace-Ausgabe enthalten ist (alle standardmäßig: `true`).

---

## Aktualisierung

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: Veröffentlichungskanal – `"stable"`, `"extended-stable"`, `"beta"` oder `"dev"`. Extended-Stable ist ausschließlich paketbasiert: Vordergrundbefehle verwalten die Installation, während der Gateway schreibgeschützte Aktualisierungshinweise ausgeben kann.
- `checkOnStart`: beim Start des Gateways nach npm-Aktualisierungen suchen (Standard: `true`). Gespeicherte Extended-Stable-Auswahlen verwenden denselben schreibgeschützten Hinweis und denselben 24-Stunden-Zeitplan für Hinweise.
- `auto.enabled`: automatische Hintergrundaktualisierung für Stable- und Beta-Paketinstallationen aktivieren (Standard: `false`). Extended-Stable wird niemals automatisch angewendet.

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: globale ACP-Funktionsfreigabe (Standard: `true`; legen Sie `false` fest, um ACP-Weiterleitung und Optionen zum Erzeugen auszublenden).
- `dispatch.enabled`: unabhängige Freigabe für die Weiterleitung von ACP-Sitzungsdurchläufen (Standard: `true`). Legen Sie `false` fest, damit ACP-Befehle verfügbar bleiben, die Ausführung jedoch blockiert wird.
- `backend`: Standard-ID des ACP-Laufzeit-Backends (muss mit einem registrierten ACP-Laufzeit-Plugin übereinstimmen).
  Installieren Sie zuerst das Backend-Plugin. Falls `plugins.allow` festgelegt ist, nehmen Sie außerdem die Backend-Plugin-ID auf (zum Beispiel `acpx`), andernfalls wird das ACP-Backend nicht geladen.
- `fallbacks`: geordnete Liste alternativer ACP-Backend-IDs, die ausprobiert werden, wenn das primäre Backend frühzeitig mit einem vermutlich vorübergehenden Fehler ausfällt (nicht verfügbar, ratenbegrenzt, Kontingent ausgeschöpft oder überlastet), bevor es eine Ausgabe erzeugt hat. Jeder Eintrag muss mit dem Backend eines registrierten ACP-Laufzeit-Plugins übereinstimmen.
- `defaultAgent`: ID des alternativen ACP-Ziel-Agenten, wenn beim Erzeugen kein explizites Ziel angegeben wird.
- `allowedAgents`: Zulassungsliste der Agent-IDs, die für ACP-Laufzeitsitzungen erlaubt sind; leer bedeutet keine zusätzliche Einschränkung.
- `stream.repeatSuppression`: wiederholte Status-/Werkzeugzeilen pro Durchlauf unterdrücken (Standard: `true`).
- `stream.deliveryMode`: `"live"` streamt schrittweise; `"final_only"` puffert bis zu den abschließenden Ereignissen des Durchlaufs.
- `stream.tagVisibility`: Datensatz aus Tag-Namen und booleschen Sichtbarkeitsüberschreibungen für gestreamte Ereignisse.
- `runtime.installCommand`: optionaler Installationsbefehl, der beim Einrichten einer ACP-Laufzeitumgebung ausgeführt wird.

---

## Assistent

Verhalten und Metadaten für geführte CLI-Einrichtungsabläufe (`onboard`, `configure`, `doctor`):

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: zu Beginn des geführten Onboardings ausgewählte Zustimmung zur Erkennung. `"full"` (empfohlen) ermöglicht dem Einrichtungsprozess, automatisch nach KI-Anwendungen, Schlüsseln und lokalen Laufzeitumgebungen zu suchen; bei `"guarded"` fragt der Einrichtungsprozess einmal nach, bevor er die Umgebung durchsucht, und bietet stattdessen eine manuelle Konfiguration an.

- `wizard.appRecommendations` ist standardmäßig auf `true` gesetzt. Setzen Sie den Wert auf `false`, um Empfehlungen für installierte Anwendungen während des geführten oder klassischen Onboardings zu deaktivieren und den Gateway-Zugriff über `device.apps` zu sperren. Node-Hosts benötigen weiterhin ihr separates, standardmäßig deaktiviertes Freigabe-Flag für installierte Anwendungen, bevor sie den Befehl bekannt geben.

---

## Identität

Siehe die Identitätsfelder unter `agents.entries` in den [Agent-Standardeinstellungen](/de/gateway/config-agents#agent-defaults).

---

## Bridge (veraltet, entfernt)

Aktuelle Builds enthalten die TCP-Bridge nicht mehr. Nodes stellen die Verbindung über den Gateway-WebSocket her. `bridge.*`-Schlüssel sind nicht mehr Teil des Konfigurationsschemas (die Validierung schlägt fehl, bis sie entfernt wurden; `openclaw doctor --fix` kann unbekannte Schlüssel entfernen).

<Accordion title="Veraltete Bridge-Konfiguration (historische Referenz)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // deprecated fallback for stored notify:true jobs
    webhookToken: "replace-with-dedicated-token", // optional bearer token for outbound webhook auth
    sessionRetention: "24h", // duration string or false
  },
}
```

- `sessionRetention`: wie lange abgeschlossene isolierte Cron-Ausführungssitzungen aufbewahrt werden, bevor SQLite-Sitzungszeilen bereinigt werden. Steuert außerdem die Bereinigung archivierter Transkripte gelöschter Cron-Jobs. Standard: `24h`; legen Sie `false` fest, um dies zu deaktivieren.
- Der Ausführungsverlauf bewahrt automatisch die neuesten 2000 Abschlusszeilen pro Job auf. Verlorene Zeilen behalten ihr 24-stündiges Bereinigungsfenster.
- `webhookToken`: Bearer-Token für die POST-Zustellung von Cron-Webhooks (`delivery.mode = "webhook"`); wenn nicht angegeben, wird kein Authentifizierungs-Header gesendet.
- `webhook`: veraltete Legacy-Fallback-Webhook-URL (http/https), die von `openclaw doctor --fix` verwendet wird, um gespeicherte Jobs zu migrieren, die noch `notify: true` enthalten; die Laufzeitzustellung verwendet `delivery.mode="webhook"` pro Job zusammen mit `delivery.to` oder `delivery.completionDestination`, wenn die Ankündigungszustellung beibehalten wird.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: Fehlerwarnungen für Cron-Jobs aktivieren (Standard: `false`).
- `after`: Anzahl aufeinanderfolgender Fehler, bevor eine Warnung ausgelöst wird (positive Ganzzahl, Minimum: `1`).
- `cooldownMs`: Mindestanzahl an Millisekunden zwischen wiederholten Warnungen für denselben Job (nicht negative Ganzzahl).
- `includeSkipped`: aufeinanderfolgende übersprungene Ausführungen für den Warnschwellenwert mitzählen (Standard: `false`). Übersprungene Ausführungen werden separat erfasst und wirken sich nicht auf das Backoff bei Ausführungsfehlern aus.
- `mode`: Zustellungsmodus – `"announce"` sendet über eine Kanalnachricht; `"webhook"` sendet einen POST an den konfigurierten Webhook.
- `accountId`: optionale Konto- oder Kanal-ID zur Eingrenzung der Warnungszustellung.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- Standardziel für Cron-Fehlerbenachrichtigungen über alle Jobs hinweg.
- `mode`: `"announce"` oder `"webhook"`; standardmäßig `"announce"`, wenn genügend Zieldaten vorhanden sind.
- `channel`: Kanalüberschreibung für die Ankündigungszustellung. `"last"` verwendet den zuletzt bekannten Zustellungskanal erneut.
- `to`: explizites Ankündigungsziel oder explizite Webhook-URL. Für den Webhook-Modus erforderlich.
- `accountId`: optionale Kontoüberschreibung für die Zustellung.
- `delivery.failureDestination` pro Job überschreibt diesen globalen Standardwert.
- Wenn weder ein globales noch ein jobspezifisches Fehlerziel festgelegt ist, greifen Jobs, die bereits über `announce` zustellen, im Fehlerfall auf dieses primäre Ankündigungsziel zurück.
- `delivery.failureDestination` wird nur für `sessionTarget="isolated"`-Jobs unterstützt, sofern die primäre `delivery.mode` des Jobs nicht `"webhook"` ist.

Siehe [Cron-Jobs](/de/automation/cron-jobs). Isolierte Cron-Ausführungen werden als [Hintergrundaufgaben](/de/automation/tasks) erfasst.

## Vorlagenvariablen für Medienmodelle

In `tools.media.models[].args` expandierte Vorlagenplatzhalter:

| Variable                    | Beschreibung                                      |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | Vollständiger Text der eingehenden Nachricht      |
| `{{RawBody}}`               | Unverarbeiteter Text (ohne Verlaufs-/Absender-Wrapper) |
| `{{BodyStripped}}`          | Text ohne Gruppenerwähnungen                      |
| `{{From}}`                  | Absenderkennung                                   |
| `{{To}}`                    | Zielkennung                                       |
| `{{MessageSid}}`            | Kanalnachrichten-ID                               |
| `{{SessionId}}`             | UUID der aktuellen Sitzung                       |
| `{{IsNewSession}}`          | `"true"`, wenn eine neue Sitzung erstellt wurde |
| `{{AttachmentUrl}}`         | URL des aktuellen Anhangs oder Provider-Referenz  |
| `{{AttachmentPath}}`        | Lokaler Pfad des aktuellen Anhangs                |
| `{{AttachmentContentType}}` | MIME-Inhaltstyp des aktuellen Anhangs             |
| `{{AttachmentDir}}`         | Verzeichnis, das `AttachmentPath` enthält       |
| `{{AttachmentIndex}}`       | Nullbasierter Index des Quellenfakts               |
| `{{Transcript}}`            | Audiotranskript                                   |
| `{{Prompt}}`                | Aufgelöster Medien-Prompt für CLI-Einträge        |
| `{{MaxChars}}`              | Aufgelöste maximale Anzahl an Ausgabezeichen für CLI-Einträge |
| `{{ChatType}}`              | `"direct"` oder `"group"`        |
| `{{GroupSubject}}`          | Gruppenbetreff (Best-Effort)                      |
| `{{GroupMembers}}`          | Vorschau der Gruppenmitglieder (Best-Effort)      |
| `{{SenderName}}`            | Anzeigename des Absenders (Best-Effort)           |
| `{{SenderE164}}`            | Telefonnummer des Absenders (Best-Effort)         |
| `{{Provider}}`              | Provider-Hinweis (whatsapp, telegram, discord usw.) |

Die Legacy-Namen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` und `{{MediaDir}}`
bleiben während des Kompatibilitätszeitraums des Plugin-SDK verfügbar, sind jedoch
veraltet. Neue Konfigurationen sollten die `Attachment*`-Variablen verwenden.

---

## Konfigurationseinbindungen (`$include`)

Konfiguration auf mehrere Dateien verteilen:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**Zusammenführungsverhalten:**

- Einzelne Datei: ersetzt das umschließende Objekt.
- Datei-Array: wird der Reihe nach tief zusammengeführt (spätere Werte überschreiben frühere).
- Schwesterschlüssel: werden nach den Einbindungen zusammengeführt (überschreiben eingebundene Werte).
- Verschachtelte Einbindungen: bis zu 10 Ebenen tief.
- Pfade: werden relativ zur einbindenden Datei aufgelöst, müssen jedoch innerhalb des obersten Konfigurationsverzeichnisses bleiben (`dirname` von `openclaw.json`). Absolute/`../`-Formen sind nur zulässig, wenn sie weiterhin innerhalb dieser Grenze aufgelöst werden. Legen Sie `OPENCLAW_INCLUDE_ROOTS` (absolute Pfade) fest, um zusätzliche Stammverzeichnisse außerhalb des Konfigurationsverzeichnisses zuzulassen.
- Grenzwerte: Pfade dürfen keine Nullbytes enthalten und müssen vor und nach der Auflösung strikt kürzer als 4096 Zeichen sein; jede eingebundene Datei ist auf 2 MB begrenzt.
- Von OpenClaw ausgeführte Schreibvorgänge, die nur einen obersten Abschnitt ändern, der durch eine Einbindung einer einzelnen Datei hinterlegt ist, schreiben direkt in diese eingebundene Datei. Beispielsweise aktualisiert `plugins install` den Wert `plugins: { $include: "./plugins.json5" }` in `plugins.json5` und lässt `openclaw.json` unverändert.
- Stammeinbindungen, Einbindungs-Arrays und Einbindungen mit Schwesterschlüssel-Überschreibungen sind für von OpenClaw ausgeführte Schreibvorgänge schreibgeschützt; diese Schreibvorgänge schlagen sicher fehl, statt die Konfiguration zu verflachen.
- Fehler: eindeutige Meldungen für fehlende Dateien, Parsing-Fehler, zirkuläre Einbindungen, ungültige Pfadformate und übermäßige Länge.

---

## Verwandte Themen

- [Konfiguration](/de/gateway/configuration)
- [Konfigurationsbeispiele](/de/gateway/configuration-examples)
- [Doctor](/de/gateway/doctor)
