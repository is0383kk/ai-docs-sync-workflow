---
read_when:
    - Sie müssen wissen, aus welchem SDK-Unterpfad Sie importieren müssen
    - Sie möchten eine Referenz für alle Registrierungsmethoden von OpenClawPluginApi.
    - Sie suchen nach einem bestimmten SDK-Export.
sidebarTitle: Plugin SDK overview
summary: Import-Map, Referenz der Registrierungs-API und SDK-Architektur
title: Übersicht über das Plugin-SDK
x-i18n:
    generated_at: "2026-07-26T17:59:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

Das Plugin-SDK ist der typisierte Vertrag zwischen Plugins und dem Kern. Diese Seite dient als
Referenz dafür, **was importiert werden muss** und **was registriert werden kann**.

<Note>
  Diese Seite richtet sich an Plugin-Autoren, die `openclaw/plugin-sdk/*` innerhalb von
  OpenClaw verwenden. Externe Apps, Skripte, Dashboards, CI-Aufträge und IDE-Erweiterungen,
  die Agenten über das Gateway ausführen möchten, sollten stattdessen
  [Gateway-Integrationen für externe Apps](/de/gateway/external-apps) verwenden.
</Note>

<Tip>
Suchen Sie stattdessen nach einer praktischen Anleitung? Beginnen Sie mit [Plugins erstellen](/de/plugins/building-plugins). Verwenden Sie [Kanal-Plugins](/de/plugins/sdk-channel-plugins) für Kanäle, [Provider-Plugins](/de/plugins/sdk-provider-plugins) für Modell-Provider, [CLI-Backend-Plugins](/de/plugins/cli-backend-plugins) für lokale KI-CLI-Backends, [Agent-Harness-Plugins](/de/plugins/sdk-agent-harness) für native Agent-Ausführungsumgebungen und [Plugin-Hooks](/de/plugins/hooks) für Tool- oder Lebenszyklus-Hooks.
</Tip>

## Importkonvention

Importieren Sie immer aus einem bestimmten Unterpfad:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Jeder Unterpfad ist ein kleines, eigenständiges Modul. Dies beschleunigt den Start und
verhindert Probleme mit zyklischen Abhängigkeiten. Bevorzugen Sie für kanalspezifische Einstiegspunkt-/Build-Hilfsfunktionen
`openclaw/plugin-sdk/channel-core`; verwenden Sie `openclaw/plugin-sdk/core` weiterhin für
die umfassendere Sammelschnittstelle und gemeinsam genutzte Hilfsfunktionen wie
`buildChannelConfigSchema`.

Veröffentlichen Sie für die Kanalkonfiguration das kanaleigene JSON-Schema über
`openclaw.plugin.json#channelConfigs`. Der Unterpfad `plugin-sdk/channel-config-schema`
ist für gemeinsam genutzte Schemaprimitive und den generischen Builder vorgesehen. Die
mit OpenClaw gebündelten Plugins verwenden `plugin-sdk/bundled-channel-config-schema` für beibehaltene
Schemas gebündelter Kanäle. Dieser Unterpfad für gebündelte Schemas ist kein Muster für neue
Plugins.

<Warning>
  Importieren Sie keine Provider- oder kanalbezogenen Komfortschnittstellen (zum Beispiel
  `openclaw/plugin-sdk/slack`, `.../discord`, `.../signal`, `.../whatsapp`).
  Gebündelte Plugins setzen generische SDK-Unterpfade in ihren eigenen `api.ts`-/
  `runtime-api.ts`-Barrels zusammen; Nutzer des Kerns sollten entweder diese Plugin-lokalen
  Barrels verwenden oder einen eng gefassten generischen SDK-Vertrag hinzufügen, wenn ein Bedarf tatsächlich
  kanalübergreifend ist.

Ein kleiner Satz von Hilfsschnittstellen für gebündelte Plugins erscheint weiterhin in der generierten Export-
Zuordnung, wenn ihre Nutzung durch den zuständigen Eigentümer nachverfolgt wird. Sie dienen ausschließlich der Wartung
gebündelter Plugins und werden nicht als Importpfade für neue Drittanbieter-
Plugins empfohlen.

`openclaw/plugin-sdk/discord` und `openclaw/plugin-sdk/telegram-account` werden
ebenfalls als veraltete Kompatibilitätsfassaden für nachverfolgte Nutzung durch den zuständigen Eigentümer beibehalten. Übernehmen Sie
diese Importpfade nicht in neue Plugins; verwenden Sie stattdessen injizierte Laufzeit-Hilfsfunktionen und
generische Kanal-SDK-Unterpfade.
</Warning>

## Unterpfadreferenz

Das Plugin-SDK wird als Satz eng gefasster Unterpfade bereitgestellt, die nach Bereichen gruppiert sind (Plugin-
Einstiegspunkt, Kanal, Provider, Authentifizierung, Laufzeit, Fähigkeit, Speicher und reservierte
Hilfsfunktionen für gebündelte Plugins). Den vollständigen, gruppierten und verlinkten Katalog finden Sie unter
[Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths).

Das Inventar der Compiler-Einstiegspunkte befindet sich in
`scripts/lib/plugin-sdk-entrypoints.json`; typisierte öffentliche Exporte schließen die
internen Unterpfade aus, die in
`scripts/lib/plugin-sdk-private-local-only-subpaths.json` aufgeführt sind. Produktionseinstiegspunkte
in dieser Liste behalten reine JavaScript-Exporte der Host-Laufzeit für separat
veröffentlichte offizielle Plugins bei, während reine Testeinstiegspunkte nicht exportiert bleiben. Führen Sie
`pnpm plugin-sdk:surface` aus, um die Anzahl öffentlicher Exporte zu prüfen. Veraltete öffentliche
Unterpfade, die alt genug sind und nicht vom Produktionscode gebündelter Erweiterungen verwendet werden,
werden in `scripts/lib/plugin-sdk-deprecated-public-subpaths.json` nachverfolgt; umfassende
veraltete Reexport-Barrels werden in
`scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json` nachverfolgt.

## Registrierungs-API

Der Callback `register(api)` erhält ein `OpenClawPluginApi`-Objekt mit diesen
Methoden:

Plugins, die eine externe Teamchat-Oberfläche für eine Sitzung bereitstellen, können
den einzelnen prozessweiten Provider registrieren, der von
`openclaw/plugin-sdk/session-discussion` exportiert wird. Dessen Methode `info({ sessionKey })`
meldet, ob eine Diskussion nicht verfügbar, zum Öffnen bereit oder bereits geöffnet ist;
`open({ sessionKey })` erstellt oder ermittelt die Diskussion und gibt deren Einbettungs-
und externe URLs zurück. Durch die Registrierung eines anderen Providers wird der aktuelle Provider ersetzt.

### Registrierung von Fähigkeiten

| Methode                                           | Was sie registriert                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | Textinferenz (LLM)                                                                                                                      |
| `api.registerWorkerProvider(...)`                | Lebenszyklus-Leases für Cloud-Worker                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | Modellkatalogzeilen für Text- und Mediengenerierung                                                                                          |
| `api.registerAgentHarness(...)`                  | [Experimenteller](/de/plugins/sdk-agent-harness) nativer Agent-Executor (Codex, Copilot)                                                         |
| `api.registerCliBackend(...)`                    | Lokales CLI-Inferenz-Backend                                                                                                               |
| `api.registerChannel(...)`                       | Nachrichtenkanal                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | Wiederverwendbarer Provider für Vektoreinbettungen                                                                                                        |
| `api.registerSpeechProvider(...)`                | Text-zu-Sprache-/STT-Synthese                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | Echtzeit-Transkription per Streaming                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | Duplex-Echtzeit-Sprachsitzungen                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | Bild-/Audio-/Videoanalyse                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | Quelle für Live- oder importierte Besprechungstranskripte; Besprechungs-Plugins können `createMeetingTranscriptSourceProvider` aus `plugin-sdk/transcripts` verwenden |
| `api.registerImageGenerationProvider(...)`       | Bildgenerierung                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | Musikgenerierung                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | Videogenerierung                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | Provider für Webabruf/Web-Scraping                                                                                                               |
| `api.registerWebSearchProvider(...)`             | Websuche                                                                                                                                |
| `api.registerCompactionProvider(...)`            | Austauschbares Backend für die Transkript-Compaction                                                                                                   |

Worker-Provider müssen ihre ID außerdem in `contracts.workerProviders` deklarieren.
Der Kern speichert die dauerhafte Absicht vor `provision(profile, operationId)`. Provider validieren die Einstellungen vor der externen Zuweisung und lösen bei einer dauerhaften Ablehnung des Profils `WorkerProviderError` aus. `provision` muss dieselbe Lease übernehmen, wenn sich die Vorgangs-ID wiederholt.
Der Kern speichert die validierten Profileinstellungen zusammen mit der Lease und stellt diesen Snapshot `destroy({ leaseId, profile })`, das idempotent sein muss, sowie `inspect({ leaseId, profile })` bereit, das `active`, `destroyed` oder `unknown` zurückgibt. Dadurch können Provider Lebenszyklusaufrufe nach einem Neustart des Gateways oder dem Entfernen eines benannten Profils weiterleiten. SSH-Endpunkte verwenden für `keyRef` ein `SecretRef`, niemals direkt eingebettetes Schlüsselmaterial, und enthalten ein `hostKey` aus einer vertrauenswürdigen Bereitstellungsausgabe exakt als `algorithm base64`, ohne Hostnamen oder Kommentar. Der Kern fixiert `hostKey` und vertraut niemals einem Schlüssel aus der ersten Verbindung. Ein Provider, der ein dynamisches `keyRef` ausstellt, kann `resolveSshIdentity({ leaseId, profile, keyRef })` implementieren; falls vorhanden, ist dieser Resolver maßgeblich, während Provider ohne ihn den konfigurierten generischen Secret-Resolver verwenden.
Provider mit verlängerbaren Leases können außerdem `renew(leaseId)` implementieren.
`inspect` muss bei vorübergehenden oder unbestimmten Fehlern eine Ausnahme auslösen; geben Sie `unknown` nur bei verbindlich festgestellter Abwesenheit zurück. Der Kern markiert einen aktiven lokalen Datensatz als verwaist oder behandelt die Abwesenheit nach einer dauerhaft gespeicherten Löschanforderung als Abschluss des Abbaus.

Mit `api.registerEmbeddingProvider(...)` registrierte Einbettungs-Provider müssen
außerdem im Plugin-Manifest unter `contracts.embeddingProviders` aufgeführt sein. Dies
ist die generische Einbettungsschnittstelle für wiederverwendbare Vektorgenerierung. Die Speichersuche
kann diese generische Provider-Schnittstelle verwenden. Die ältere Schnittstelle
`api.registerMemoryEmbeddingProvider(...)` und
`contracts.memoryEmbeddingProviders` dient als veraltete Kompatibilität, während
bestehende speicherspezifische Provider migriert werden.

Speicherspezifische Provider, die weiterhin eine Laufzeit `batchEmbed(...)` bereitstellen, verbleiben beim
bestehenden Vertrag für dateiweise Batches, sofern ihre Laufzeit nicht ausdrücklich
`sourceWideBatchEmbed: true` festlegt. Diese Aktivierung ermöglicht es dem Speicher-Host, Chunks aus
mehreren geänderten Speicherdateien und aktivierten Quellen in einem `batchEmbed(...)`-Aufruf
bis zu den Batch-Grenzwerten des Hosts zu übermitteln. Batch-Adapter, die JSONL-Anforderungsdateien hochladen, müssen
Provider-Aufträge sowohl vor Erreichen der maximalen Upload-Größe als auch der maximalen Anzahl
von Anforderungen aufteilen. Der Provider muss für jeden Eingabe-Chunk genau eine Einbettung in derselben Reihenfolge wie
`batch.chunks` zurückgeben; lassen Sie das Flag weg, wenn der Provider dateilokale Batches erwartet oder
die Eingabereihenfolge über einen größeren, quellweiten Auftrag hinweg nicht beibehalten kann.

### Tools und Befehle

Verwenden Sie [`defineToolPlugin`](/de/plugins/tool-plugins) für einfache reine Tool-Plugins
mit festen Tool-Namen. Verwenden Sie `api.registerTool(...)` direkt für gemischte Plugins
oder eine vollständig dynamische Tool-Registrierung.

| Methode                                 | Was sie registriert                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | Agent-Tool (erforderlich oder `{ optional: true }`)                                                                                            |
| `api.registerCommand(def)`             | Benutzerdefinierter Befehl (umgeht das LLM)                                                                                                        |
| `api.registerNodeHostCommand(command)` | Von `openclaw node run` verarbeiteter Befehl; optionale `agentTool`-Metadaten können ihn als für den Agenten sichtbares Tool verfügbar machen, während die Node verbunden ist |

Plugin-Befehle können `agentPromptGuidance` festlegen, wenn der Agent einen kurzen,
befehlseigenen Routing-Hinweis benötigt. Beschränken Sie diesen Text auf den Befehl selbst; fügen Sie den Prompt-Buildern
des Kerns keine Provider- oder Plugin-spezifischen Richtlinien hinzu.

Anleitungseinträge können Legacy-Zeichenfolgen sein, die für jede Prompt-Oberfläche gelten, oder
strukturierte Einträge:

```ts
agentPromptGuidance: [
  "Globaler Befehlshinweis.",
  { text: "Dies nur im OpenClaw-Hauptprompt anzeigen.", surfaces: ["openclaw_main"] },
];
```

Strukturierte `surfaces` können `openclaw_main`, `codex_app_server`,
`cli_backend`, `acp_backend` oder `subagent` enthalten. `pi_main` bleibt ein veralteter Alias
für `openclaw_main`. Lassen Sie `surfaces` für bewusst oberflächenübergreifende Anweisungen weg. Übergeben
Sie kein leeres `surfaces`-Array; es wird abgelehnt, damit ein versehentlicher Verlust des Geltungsbereichs
nicht zu globalem Prompt-Text führt.

Native Entwickleranweisungen für den Codex-App-Server sind strenger als bei anderen Prompt-
Oberflächen: Nur Anweisungen, die ausdrücklich auf `codex_app_server` begrenzt sind, werden in
diese Lane mit höherer Priorität übernommen. Veraltete Zeichenkettenanweisungen und unstrukturierte
strukturierte Anweisungen bleiben aus Kompatibilitätsgründen für Nicht-Codex-Prompt-Oberflächen verfügbar.

Node-Host-Befehle werden auf dem verbundenen Node-Host ausgeführt, nicht innerhalb des Gateway-
Prozesses. Wenn `agentTool` vorhanden ist, veröffentlicht die Node nach einer
erfolgreichen Gateway-Verbindung einen Deskriptor; das Gateway stellt ihn Agent-Ausführungen nur bereit, solange diese
Node verbunden ist und nur, wenn `command` des Deskriptors zur
genehmigten Befehlsoberfläche der Node gehört. Setzen Sie `agentTool.defaultPlatforms`, um einen
ungefährlichen Befehl in die standardmäßige Node-Befehls-Zulassungsliste aufzunehmen; andernfalls ist
ein explizites `gateway.nodes.commands.allow` oder eine Node-Aufrufrichtlinie erforderlich. `agentTool.name`
muss Provider-sicher sein: mit einem Buchstaben beginnen, ausschließlich Buchstaben, Ziffern,
Unterstriche oder Bindestriche verwenden und höchstens 64 Zeichen lang sein. MCP-gestützte Node-Tools
können `agentTool.mcp`-Metadaten setzen, damit Katalog- und Tool-Suchoberflächen
die Identität des entfernten MCP-Servers/Tools anzeigen können; die Ausführung erfolgt jedoch weiterhin über den
angekündigten Node-Befehl.

### Infrastruktur

| Methode                                         | Was sie registriert                                                     |
| ----------------------------------------------- | ----------------------------------------------------------------------- |
| `api.registerHook(events, handler, opts?)`      | Ereignis-Hook                                                           |
| `api.registerHttpRoute(params)`                 | Gateway-HTTP-Endpunkt                                                   |
| `api.registerGatewayMethod(name, handler)`      | Gateway-RPC-Methode                                                     |
| `api.registerGatewayDiscoveryService(service)`  | Ankündigungsdienst für die lokale Gateway-Erkennung                     |
| `api.registerCli(registrar, opts?)`             | CLI-Unterbefehl                                                         |
| `api.registerNodeCliFeature(registrar, opts?)`  | CLI für Node-Funktionen unter `openclaw nodes`                          |
| `api.registerService(service)`                  | Hintergrunddienst                                                       |
| `api.registerInteractiveHandler(registration)`  | Interaktiver Handler                                                    |
| `api.registerAgentToolResultMiddleware(...)`    | Laufzeit-Middleware für Tool-Ergebnisse                                 |
| `api.registerMemoryPromptSupplement(builder)`   | Additiver Prompt-Abschnitt im Umfeld des Speichers                      |
| `api.registerMemoryPromptPreparation(prepare)`  | Asynchrone Vorbereitung eines Prompt-Abschnitts im Umfeld des Speichers |
| `api.registerMemoryCorpusSupplement(adapter)`   | Additiver Korpus zum Durchsuchen/Lesen des Speichers                    |
| `api.registerHostedMediaResolver(resolver)`     | Resolver für browserartige URLs gehosteter Medien                       |
| `api.registerMcpServerConnectionResolver(...)`  | MCP-Transport pro Anfragendem (`url`/`headers`) für einen statischen Servernamen |
| `api.registerTextTransforms(transforms)`        | Plugin-eigene Kompatibilitätsumschreibungen für Prompt-/Nachrichtentext |
| `api.registerConfigMigration(migrate)`          | Leichtgewichtige Konfigurationsmigration vor dem Laden der Plugin-Laufzeit |
| `api.registerMigrationProvider(provider)`       | Importeur für `openclaw migrate`                                        |
| `api.registerAutoEnableProbe(probe)`            | Konfigurationsprüfung, die dieses Plugin automatisch aktivieren kann   |
| `api.registerReload(registration)`              | Richtlinie für Neustart/Hot-Reload/Keine Aktion anhand von Konfigurationspräfixen |
| `api.registerNodeHostCommand(command)`          | Befehls-Handler, der gekoppelten Nodes bereitgestellt wird              |
| `api.registerNodeInvokePolicy(policy)`          | Zulassungslisten-/Genehmigungsrichtlinie für von Nodes aufgerufene Befehle |
| `api.registerSecurityAuditCollector(collector)` | Befundsammler für `openclaw security audit`                                     |

#### Webhook-Arbeit nach der Bestätigung

Webhook-Routen, die eine Anfrage bestätigen, bevor die Verarbeitung abgeschlossen ist, müssen
diese abgekoppelte Arbeit auf einen eigenen nachverfolgten Zulassungsstamm verlagern:

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`Webhook-Versand fehlgeschlagen: ${String(error)}`);
});
```

Rufen Sie `runDetachedWebhookWork(...)` synchron auf, solange die HTTP-Anfrage noch
zugelassen ist. Der Helper reserviert sofort einen unabhängigen Stamm und startet dann den
Callback im nächsten Microtask, sodass der Anfrage-Handler zuerst seine
Bestätigung schreiben kann. Das zurückgegebene Promise übernimmt das Callback-Ergebnis; die Aufrufenden
sind weiterhin für die Behandlung von Ablehnungen verantwortlich. Dadurch wird Arbeit in der Warteschlange nach der Bestätigung angenommen, und
Entleerungsvorgänge bei Neustart oder Suspendierung warten auf sie. Handler, die vor der Rückgabe die gesamte Verarbeitung
abwarten, benötigen diesen Helper nicht.

#### Auf Anfragende begrenzte MCP-Verbindungen

Halten Sie die MCP-Server-**Identität** (Name, Tool-Filter) in `mcp.servers`, im Manifestfeld `mcpServers`
eines nativen Plugins oder in einem Bundle-Manifest statisch. Optional können Sie einen Verbindungs-Resolver registrieren, damit jeder vertrauenswürdige
Nachrichtenabsender einen eigenen Transport erhält:

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId wird vom Host als vertrauenswürdig eingestuft; erfinden Sie hier niemals eine Absenderidentität.
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // diesen Server für die aktuelle Ausführung weglassen
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

Vertragshinweise:

- Der Resolver-Kontext enthält ausschließlich eine vertrauenswürdige Host-Identität (`requesterSenderId`,
  optional `agentAccountId` / `messageChannel`). Zukünftige vertrauenswürdige Felder (zum
  Beispiel Cron-/Subagent-Benutzerkontext) können additiv hinzugefügt werden.
- Ein Plugin besitzt genau einen Servernamen: Ein doppeltes
  `registerMcpServerConnectionResolver` für denselben `serverName` von einem anderen
  Plugin wird mit einer Fehlerdiagnose abgelehnt (die erste Registrierung gewinnt), sodass
  der Verbindungseigentümer niemals von der Ladereihenfolge der Plugins abhängt.
- Tool-Namen werden aus der vollständigen Menge deklarierter Server abgeleitet, sodass eine teilweise Auflösung
  sichere Servernamen niemals zwischen Anfragenden oder Durchläufen verändert. Der Core überprüft nicht,
  ob unterschiedliche Endpunkte für Anfragende identische Tool-Schemas bereitstellen; ein
  Resolver muss jeden Anfragenden auf denselben logischen Dienst verweisen, andernfalls
  unterscheiden sich Tool-Schemas (und die Stabilität des Prompt-Caches) je nach Anfragendem.
- Ausführungen ohne vertrauenswürdiges `requesterSenderId` (Cron, Subagent, Heartbeat, öffentliches
  Gateway) materialisieren niemals auf Anfragende begrenzte Server. Es gibt keine gemeinsam genutzte
  Fallback-Verbindung.
- `resolve` ist auf 10 Sekunden pro Server begrenzt; bei einer Zeitüberschreitung oder Ausnahme wird dieser
  Server für die Ausführung weggelassen, ohne dass statisches MCP fehlschlägt.
- Aufgelöste Verbindungen werden höchstens alle 5 Minuten pro Anfragendem erneut validiert:
  Bei einer Rotation wird der Transport mit neuen Anmeldedaten neu aufgebaut, und ein `null`-Ergebnis
  widerruft ihn (die zwischengespeicherte Laufzeit wird selbst mitten in einer Sitzung freigegeben). Widerrufene oder
  rotierte Anmeldedaten können daher bis zu 5 Minuten lang weiterverwendet werden.
- Aufgelöste `headers` werden niemals protokolliert oder persistiert; der Core hält nur einen flüchtigen,
  im Arbeitsspeicher abgelegten schlüsselbasierten Digest (prozesslokales HMAC) vor, um die Rotation von Anmeldedaten zu erkennen, und
  registriert aufgelöste Anmeldedatenwerte aus Headern/URLs in der Schwärzungsregistrierung
  für Protokollierung und Debug-Erfassung.
- Auf Anfragende begrenzte Server erzeugen keine MCP-App-Ansichten: Eine Ansicht überdauert die
  durch den Anfragenden authentifizierte Ausführung, und die Begrenzung der Gateway-Ansicht besitzt keine Identität des Anfragenden,
  daher bleiben App-Vorschauen für diese Server standardmäßig geschlossen. Tool-Ergebnisse
  sind davon nicht betroffen.
- Statische Server ohne Resolver behalten den bestehenden sitzungsbezogenen Lebenszyklus bei.
- **Bereitstellungsregel für Harnesses:** Auf Anfragende begrenzte Server gelangen niemals in die Harness-native
  MCP-Client-Konfiguration (Codex-Thread `mcp_servers`, CLI `-c mcp_servers=…` oder eine
  andere sitzungsübergreifend gemeinsam genutzte MCP-Projektion). Harnesses stellen sie stattdessen als ausführungsbezogene
  Tools bereit:
  - Eingebetteter Runner: Sitzungs-MCP-Laufzeit + Bundle-Tools (statisch + begrenzt).
  - Codex-App-Server: dynamische Tools über
    `materializeRequesterScopedMcpToolsForHarnessRun` (nur begrenzte Tools; statische
    Server verbleiben im nativen MCP-Client von Codex).
- Begrenzte Tool-**Spezifikationen** sind nach der ersten erfolgreichen Auflösung in
  dieser Sitzung sitzungsstabil, sodass Harnesses mit gemeinsam genutzten Threads (Codex) bei einem Wechsel
  der Absender keine Threads rotieren. Bevor ein Anfragender erfolgreich aufgelöst wurde, werden keine begrenzten Spezifikationen angekündigt.
- Nicht authentifizierte Anfragende in einem Harness mit gemeinsam genutzten Threads sehen weiterhin die angekündigten
  begrenzten Tools; der Aufruf eines solchen Tools gibt für diesen Anfragenden einen eindeutigen Tool-Fehler wegen fehlender Verbindung zurück.
  OpenClaw greift niemals auf die Anmeldedaten eines anderen Anfragenden zurück.

Builder für Speicher-Prompt-Ergänzungen erhalten optional den Kontext `agentId`,
`agentSessionKey` und `sandboxed`. Aufrufe von `search` und `get` für Ergänzungen des Speicherkorpus
erhalten optional den Kontext `agentId` und `sandboxed`. Plugins mit
Agent-eigenem Speicher sollten diesen Speicher bei jedem Aufruf auflösen, statt
bei der Registrierung einen einzelnen globalen Pfad zu erfassen. Wenn eine Agent-ID erforderlich ist, aber
bei einer Multi-Agent-Operation fehlt, muss der Vorgang standardmäßig geschlossen fehlschlagen, statt einen
beliebigen Agent auszuwählen.

Verwenden Sie `registerMemoryPromptPreparation(...)`, wenn Prompt-Text vom asynchronen
Plugin-Zustand abhängt. Der Callback wird einmal vor jedem vollständigen Agent-Prompt ausgeführt und erhält
denselben Tool-, Agent-, Sitzungs- und Sandbox-Kontext wie synchrone Builder für Speicher-Prompts.
Validieren Sie die aktuelle Speicherinhaber-Instanz, bevor Sie persistierten
Zustand laden, und geben Sie dann nur Zeilen für diese Ausführung zurück. OpenClaw friert diese Zeilen ein und
übergibt das unveränderliche Ergebnis an die synchrone Prompt-Zusammenstellung. Persistenz,
atomarer Austausch und Löschung bei Entfernung des Inhabers müssen innerhalb des zuständigen Plugins verbleiben; führen Sie in einem Prompt-Builder keine
Abfragen oder Dateilesevorgänge durch.

Interaktive Telegram-Handler können `{ submitText }` zurückgeben, um Text nach
erfolgreichem Abschluss des Handlers durch den normalen eingehenden Agent-Pfad von Telegram zu leiten. OpenClaw behält
die Callback-Schaltfläche bei, wenn die Richtlinie für eingehende Nachrichten den Text überspringt oder die Verarbeitung fehlschlägt, sodass
die Person den Vorgang wiederholen kann, nachdem sich die blockierende Bedingung geändert hat. Dieses Ergebnisfeld ist
Telegram-spezifisch; andere Kanäle behalten ihre eigenen Verträge für interaktive Ergebnisse bei.

### Host-Hooks für Workflow-Plugins

Host-Hooks sind die SDK-Schnittstellen für Plugins, die am Host-
Lebenszyklus teilnehmen müssen, statt nur einen Provider, Kanal oder ein Tool hinzuzufügen. Es handelt sich um
generische Verträge; der Planungsmodus kann sie verwenden, ebenso aber Genehmigungs-Workflows,
Arbeitsbereichs-Richtlinienprüfungen, Hintergrundüberwachungen, Einrichtungsassistenten und UI-Begleit-
Plugins.

| Methode                                                                               | Zuständiger Vertrag                                                                                                                                           |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | Plugin-eigener, JSON-kompatibler Sitzungsstatus, der über Gateway-Sitzungen projiziert wird                                                                             |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | Dauerhafter Exactly-once-Kontext, der für eine Sitzung in den nächsten Agent-Durchlauf injiziert wird                                                                             |
| `api.registerTrustedToolPolicy(...)`                                                 | Durch das Manifest kontrollierte, vertrauenswürdige Prä-Plugin-Tool-Richtlinie, die Tool-Parameter blockieren oder umschreiben kann                                                                        |
| `api.registerToolMetadata(...)`                                                      | Anzeigemetadaten des Tool-Katalogs, ohne die Tool-Implementierung zu ändern                                                                                     |
| `api.registerCommand(...)`                                                           | Bereichsgebundene Plugin-Befehle; Befehlsergebnisse können `continueAgent: true` oder `suppressReply: true` setzen; native Discord-Befehle unterstützen `descriptionLocalizations` |
| `api.session.controls.registerControlUiDescriptor(...)`                              | Beitragsdeskriptoren für die Control UI für Sitzungs-, Tool-, Ausführungs-, Einstellungs- oder Registerkartenoberflächen                                                                      |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | Bereinigungs-Callbacks für Plugin-eigene Laufzeitressourcen auf Pfaden zum Zurücksetzen, Löschen oder Neuladen                                                                          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | Bereinigte Ereignisabonnements für Workflow-Status und Überwachungen                                                                                              |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | Plugin-Arbeitsstatus pro Ausführung, der am Ende des Ausführungslebenszyklus gelöscht wird                                                                                             |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | Bereinigungsmetadaten für Plugin-eigene Scheduler-Aufträge; plant keine Arbeit und erstellt keine Aufgabeneinträge                                                            |
| `api.session.workflow.sendSessionAttachment(...)`                                    | Nur für gebündelte Plugins verfügbare, vom Host vermittelte Zustellung von Dateianhängen an die aktive Route der direkt ausgehenden Sitzung                                                            |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | Nur für gebündelte Plugins verfügbare, Cron-gestützte geplante Sitzungsdurchläufe sowie Tag-basierte Bereinigung                                                                                    |
| `api.session.controls.registerSessionAction(...)`                                    | Typisierte Sitzungsaktionen, die Clients über den Gateway auslösen können                                                                                             |

Ein `surface: "tab"`-Deskriptor fügt der Control UI eine Registerkarte in der Seitenleiste hinzu. Die Registerkartendeskriptoren aktiver
Plugins werden Dashboard-Clients in der Gateway-Begrüßung
(`controlUiTabs`) bekannt gegeben, sodass die Registerkarte nur angezeigt wird, solange das Plugin aktiviert ist.
Gebündelte Plugins können eine vollwertige Dashboard-Ansicht für ihre Registerkarte bereitstellen; andere
Plugins können `path` auf eine Plugin-HTTP-Route setzen (siehe
`api.registerHttpRoute(...)`), die das Dashboard in einem Sandbox-Frame darstellt.
`icon` ist ein Hinweis auf den Namen eines Dashboard-Symbols, `group` wählt den Seitenleistenabschnitt
(`control` oder `agent`), `order` bestimmt die Reihenfolge unter den Plugin-Registerkarten und `requiredScopes`
blendet die Registerkarte für Verbindungen aus, denen diese Operator-Berechtigungsbereiche fehlen:

Registrieren Sie für eine durch den Gateway geschützte externe Registerkarte den Deskriptor `path` unter einer
HTTP-Route `auth: "gateway"` desselben Plugins. Nach dem authentifizierten Bootstrap erhält der Browser eine
kurzlebige, auf dieses Plugin und den Routenstamm beschränkte HttpOnly-Berechtigung, damit der
Sandbox-Frame geladen werden kann, ohne das Gateway-Bearer-Token in seine URL
oder sein JavaScript zu kopieren. Das authentifizierte übergeordnete Element erneuert die Berechtigung, solange die externe Registerkarte
aktiv ist, sowie vor ihrem Einhängen nach einer Navigation oder der Wiederaufnahme des Browsers. Es
prüft die Berechtigung außerdem aus derselben undurchsichtigen Sandbox, bevor es den Frame einhängt, sodass Browser-
Datenschutzmodi, die das Cookie blockieren, sicher geschlossen mit einem nicht verfügbaren Panel fehlschlagen.
Die Frame-Berechtigung akzeptiert nur `GET` und `HEAD` und enthält immer
`operator.read`; `requiredScopes` steuert die Sichtbarkeit der Registerkarte, erweitert jedoch niemals die
Cookie-Berechtigung. Änderungen verbleiben auf explizit durch den Gateway authentifizierten übergeordneten Oberflächen oder
Bearer-Oberflächen. Externe Registerkarten erfordern HTTPS/Tailscale Serve oder einen
vom Browser als vertrauenswürdig eingestuften Loopback-Ursprung; einfaches HTTP auf einem LAN-Host zeigt den
Fehler für einen unsicheren Kontext an, statt ein Panel einzuhängen, das sich nicht authentifizieren kann.
Eine vollständige Blockierung von Drittanbieter-Cookies macht durch den Gateway geschützte Registerkarten ebenfalls nicht verfügbar.
Wie bei allen nativen Plugin-Oberflächen verbleibt der Frame innerhalb der Vertrauensgrenze des installierten
Plugins; OpenClaw behandelt installierte Plugins nicht als gegenseitig
isolierte Browser-Sicherheitsprinzipale.
Cookie-Berechtigungen verwenden die Hostnamengrenze des Browsers, nicht dessen Portgrenze. Betreiben Sie
keine gegenseitig nicht vertrauenswürdigen Dienste gemeinsam unter dem Gateway-Hostnamen, auch nicht an anderen
Ports.
Registerkarten mit Plugin-verwalteter Authentifizierung behalten ihr direktes iframe-Verhalten bei und fordern oder
benötigen diese Gateway-Berechtigung nicht.

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "Logbuch",
  description: "Ihr Tag als Zeitleiste, erstellt aus Bildschirmaufnahmen.",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

Verwenden Sie für neuen Plugin-Code die gruppierten Namensräume:

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

Die entsprechenden flachen Methoden bleiben als veraltete Kompatibilitäts-
Aliasse für vorhandene Plugins verfügbar. Fügen Sie keinen neuen Plugin-Code hinzu, der
`api.registerSessionExtension`, `api.enqueueNextTurnInjection`,
`api.registerControlUiDescriptor`, `api.registerRuntimeLifecycle`,
`api.registerAgentEventSubscription`, `api.emitAgentEvent`,
`api.setRunContext`, `api.getRunContext`, `api.clearRunContext`,
`api.registerSessionSchedulerJob`, `api.registerSessionAction`,
`api.sendSessionAttachment`, `api.scheduleSessionTurn` oder
`api.unscheduleSessionTurnsByTag` direkt aufruft.

`scheduleSessionTurn(...)` ist eine sitzungsgebundene Komfortfunktion über dem Gateway-
Cron-Scheduler. Cron ist für die Zeitplanung zuständig und erstellt den Hintergrund-Aufgabeneintrag, wenn der
Durchlauf ausgeführt wird; das Plugin SDK beschränkt lediglich die Zielsitzung, die Plugin-eigene
Benennung und die Bereinigung. Verwenden Sie `api.runtime.tasks.managedFlows` innerhalb des geplanten
Durchlaufs, wenn die Arbeit selbst dauerhaften mehrstufigen Task-Flow-Status benötigt.

Die Verträge trennen die Zuständigkeiten bewusst:

- Externe Plugins können Sitzungserweiterungen, UI-Deskriptoren, Befehle, Tool-
  Metadaten, Injektionen für den nächsten Durchlauf und normale Hooks verwalten.
- Vertrauenswürdige Tool-Richtlinien werden vor gewöhnlichen `before_tool_call`-Hooks ausgeführt und sind
  vom Host als vertrauenswürdig eingestuft. Gebündelte Richtlinien werden zuerst ausgeführt; Richtlinien installierter Plugins erfordern
  eine explizite Aktivierung sowie ihre lokalen IDs in
  `contracts.trustedToolPolicies` und werden anschließend in der Ladereihenfolge der Plugins ausgeführt. Richtlinien-IDs
  sind auf das registrierende Plugin beschränkt.
- Die Eigentümerschaft reservierter Befehle ist ausschließlich gebündelten Plugins vorbehalten. Externe Plugins sollten ihre
  eigenen Befehlsnamen oder Aliasse verwenden.
- `allowPromptInjection=false` deaktiviert Prompt-verändernde Hooks einschließlich
  `agent_turn_prepare`, `before_prompt_build`, `heartbeat_prompt_contribution`
  und `enqueueNextTurnInjection`.

Beispiele für Nicht-Plan-Nutzer:

| Plugin-Archetyp             | Verwendete Hooks                                                                                                                             |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Genehmigungsworkflow            | Sitzungserweiterung, Befehlsfortsetzung, Injektion für den nächsten Durchlauf, UI-Deskriptor                                                            |
| Richtlinien-Gate für Budget/Arbeitsbereich | Vertrauenswürdige Tool-Richtlinie, Tool-Metadaten, Sitzungsprojektion                                                                                 |
| Hintergrund-Lebenszyklusüberwachung | Bereinigung des Laufzeitlebenszyklus, Abonnement von Agent-Ereignissen, Eigentümerschaft/Bereinigung des Sitzungsschedulers, Heartbeat-Prompt-Beitrag, UI-Deskriptor |
| Einrichtungs- oder Onboarding-Assistent   | Sitzungserweiterung, bereichsgebundene Befehle, Control-UI-Deskriptor                                                                              |

<Note>
  Reservierte zentrale Admin-Namensräume (`config.*`, `exec.approvals.*`, `wizard.*`,
  `update.*`) bleiben immer `operator.admin`, selbst wenn ein Plugin versucht, einen
  engeren Gateway-Methodenbereich zuzuweisen. Verwenden Sie vorzugsweise Plugin-spezifische Präfixe für
  Plugin-eigene Methoden.
</Note>

<Accordion title="Wann Tool-Ergebnis-Middleware verwendet werden sollte">
  Gebündelte Plugins und explizit aktivierte installierte Plugins mit passenden
  Manifestverträgen können `api.registerAgentToolResultMiddleware(...)` verwenden, wenn
  sie ein Tool-Ergebnis nach der Ausführung und bevor die Laufzeit
  dieses Ergebnis an das Modell zurückgibt, umschreiben müssen. Dies ist die vertrauenswürdige, laufzeitneutrale
  Schnittstelle für asynchrone Ausgabereduzierer wie tokenjuice.

Plugins müssen `contracts.agentToolResultMiddleware` für jede vorgesehene
Laufzeit deklarieren, beispielsweise `["openclaw", "codex"]`. Installierte Plugins ohne diesen
Vertrag oder ohne explizite Aktivierung können diese Middleware nicht registrieren; verwenden Sie
normale OpenClaw-Plugin-Hooks für Arbeit, die kein Tool-Ergebnis-Timing vor dem Modell
benötigt. Der alte
Registrierungspfad für Erweiterungsfabriken, der ausschließlich für eingebettete Runner vorgesehen war, wurde entfernt.
</Accordion>

### Registrierung der Gateway-Erkennung

`api.registerGatewayDiscoveryService(...)` ermöglicht einem Plugin, den aktiven
Gateway über einen lokalen Erkennungstransport wie mDNS/Bonjour bekannt zu geben. OpenClaw ruft den
Dienst während des Gateway-Starts auf, wenn die lokale Erkennung aktiviert ist, übergibt die
aktuellen Gateway-Ports und nicht geheimen TXT-Hinweisdaten und ruft den zurückgegebenen
`stop`-Handler während des Herunterfahrens des Gateways auf.

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

Gateway-Erkennungs-Plugins dürfen bekannt gegebene TXT-Werte nicht als Geheimnisse oder
Authentifizierung behandeln. Die Erkennung ist ein Routing-Hinweis; die Gateway-Authentifizierung und TLS-Pinning bleiben
für das Vertrauen zuständig.

### CLI-Registrierungsmetadaten

`api.registerCli(registrar, opts?)` akzeptiert zwei Arten von Befehlsmetadaten:

- `commands`: explizite Befehlsnamen im Besitz des Registrierenden
- `descriptors`: Befehlsdeskriptoren für die Analysephase, die für CLI-Hilfe,
  Routing und verzögerte Plugin-CLI-Registrierung verwendet werden
- `parentPath`: optionaler Pfad des übergeordneten Befehls für verschachtelte Befehlsgruppen, beispielsweise
  `["nodes"]`

Verwenden Sie für Funktionen gekoppelter Nodes vorzugsweise
`api.registerNodeCliFeature(registrar, opts?)`. Es ist ein kleiner Wrapper um
`api.registerCli(..., { parentPath: ["nodes"] })` und weist Befehle wie
`openclaw nodes canvas` explizit als Plugin-eigene Node-Funktionen aus.

Wenn ein Plugin-Befehl im normalen Stamm-CLI-Pfad verzögert geladen bleiben soll,
geben Sie `descriptors` an, die jeden von diesem
Registrierenden bereitgestellten Befehlsstamm der obersten Ebene abdecken.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Matrix-Konten, Verifizierung, Geräte und Profilstatus verwalten",
        hasSubcommands: true,
      },
    ],
  },
);
```

Verschachtelte Befehle erhalten den aufgelösten übergeordneten Befehl als `program`:

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "Canvas-Inhalte von einem gekoppelten Node erfassen oder rendern",
        hasSubcommands: true,
      },
    ],
  },
);
```

Verwenden Sie `commands` nur dann allein, wenn Sie keine verzögerte Registrierung der Root-CLI benötigen.
Dieser sofortige Kompatibilitätspfad wird weiterhin unterstützt, installiert jedoch keine
deskriptorbasierten Platzhalter für verzögertes Laden zur Parse-Zeit.

### Registrierung von CLI-Backends

Mit `api.registerCliBackend(...)` kann ein Plugin die Standardkonfiguration für ein lokales
KI-CLI-Backend wie `claude-cli` oder `my-cli` bereitstellen.

- Die Backend-`id` wird zum Provider-Präfix in Modellreferenzen wie `my-cli/gpt-5`.
- Die Backend-`config` ist der maßgebliche Befehlsadapter: argv, Umgebung,
  Parser-, Sitzungs-, Bild- und Zuverlässigkeitsverhalten befinden sich im Plugin-Code.
- Benutzer wählen das Backend über Modellreferenzen oder modellbezogene `agentRuntime.id` aus;
  `openclaw.json` schreibt den Adapter nicht um.
- Verwenden Sie `normalizeConfig`, wenn registrierte statische Felder einen laufzeitabhängigen
  Normalisierungsdurchlauf benötigen.
- Verwenden Sie `resolveExecutionArgs` für anfragebezogene argv-Umschreibungen, die zum
  CLI-Dialekt gehören, etwa um OpenClaw-Denkstufen einem nativen Aufwands-Flag
  zuzuordnen. Der Hook erhält `ctx.executionMode`; verwenden Sie `"side-question"`, um
  Backend-native Isolations-Flags für kurzlebige `/btw`-Aufrufe hinzuzufügen. Wenn diese Flags
  native Tools für eine ansonsten stets aktive CLI zuverlässig deaktivieren,
  deklarieren Sie außerdem `sideQuestionToolMode: "disabled"`.
- Verwenden Sie `prepareExecution` für Backend-eigene Startumgebungen oder temporäre
  Authentifizierungs-/Konfigurationsbrücken. Das zugehörige `ctx.contextTokenBudget` ist das effektive
  Token-Limit, das für den Lauf ausgewählt wurde, sodass Backends mit nativer Compaction ihren
  eigenen Schwellenwert ohne providerspezifische Core-Verzweigungen abstimmen können. Es erhält außerdem die
  vom Core vorbereitete `ctx.env`, wenn das Backend-Staging gebündelte MCP-Einstellungen erweitern muss.
- Backends, die alle nativen Tools für einen bestimmten Lauf deaktivieren können, dürfen
  `nativeToolMode: "selectable"` deklarieren. Eingeschränkte Aufrufe übergeben eine exakte
  `ctx.toolAvailability.native`-Liste sowie kanonische
  `ctx.toolAvailability.openClaw`-Namen. Deklarieren Sie
  `toolAvailabilityEnforcement: "execution-args"` und setzen Sie den Vertrag in den
  endgültigen argv für neue oder fortgesetzte Sitzungen durch, oder deklarieren Sie `"prepare-execution"`, setzen Sie ihn in
  der bereitgestellten Richtlinie durch und geben Sie `toolAvailabilityEnforced: true` zurück. OpenClaw deaktiviert
  native Tools für Laufzeitbeschränkungen wie Cron-`toolsAllow` und schlägt geschlossen fehl, wenn
  der deklarierte Durchsetzungspfad unvollständig ist.

Eine vollständige Anleitung zur Erstellung finden Sie unter
[CLI-Backend-Plugins](/de/plugins/cli-backend-plugins).

### Exklusive Slots

| Methode                                    | Registrierter Inhalt                                                                                                                                                                                                    |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Kontext-Engine (jeweils eine aktiv). Lebenszyklus-Callbacks erhalten `runtimeSettings`, wenn der Host Modell-/Provider-/Modusdiagnosen bereitstellen kann; ältere strikte Engines werden ohne diesen Schlüssel erneut aufgerufen. |
| `api.registerMemoryCapability(capability)` | Einheitliche Speicherfunktion                                                                                                                                                                                           |

### Veraltete Adapter für Speicher-Embeddings

| Methode                                        | Registrierter Inhalt                            |
| ---------------------------------------------- | ----------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Speicher-Embedding-Adapter für das aktive Plugin |

- `registerMemoryCapability` ist die exklusive Speicher-Plugin-API.
- `registerMemoryCapability` kann außerdem `publicArtifacts.listArtifacts(...)`
  für vom Host verwaltete Exporte bereitstellen. Begleit-Plugins, die diese deklarierten
  Artefakte auflisten, verwenden weiterhin `listActiveMemoryPublicArtifacts(...)` aus der beibehaltenen
  `openclaw/plugin-sdk/memory-host-core`-Fassade, bis eine gezielte öffentliche Verbraucher-API
  verfügbar ist; sie dürfen nicht auf die private Struktur eines anderen Plugins zugreifen.
- `MemoryFlushPlan.model` kann den Flush-Durchlauf an eine exakte `provider/model`-
  Referenz wie `ollama/qwen3:8b` binden, ohne die aktive Fallback-Kette
  zu übernehmen.
- `registerMemoryEmbeddingProvider` ist veraltet. Neue Embedding-Provider
  sollten `api.registerEmbeddingProvider(...)` und
  `contracts.embeddingProviders` verwenden.
- Bestehende speicherspezifische Provider funktionieren während des Migrationszeitraums
  weiterhin, bei der Plugin-Inspektion wird dies für nicht gebündelte Plugins jedoch als
  Kompatibilitätsschuld ausgewiesen.

### Ereignisse und Lebenszyklus

| Methode                                      | Funktion                         |
| -------------------------------------------- | -------------------------------- |
| `api.on(hookName, handler, opts?)`           | Typisierter Lebenszyklus-Hook    |
| `api.onConversationBindingResolved(handler)` | Callback für Gesprächszuordnungen |

Beispiele, gängige Hook-Namen und Schutzsemantik finden Sie unter [Plugin-Hooks](/de/plugins/hooks).

### Entscheidungssemantik von Hooks

`before_install` ist ein Lebenszyklus-Hook der Plugin-Laufzeit und nicht die
Installationsrichtlinienoberfläche für Betreiber. Verwenden Sie `security.installPolicy`, wenn eine Zulassen-/Blockieren-Entscheidung
CLI- und Gateway-gestützte Installations- oder Aktualisierungspfade abdecken muss.

- `before_tool_call`: Die Rückgabe von `{ block: true }` ist endgültig. Sobald ein Handler diesen Wert setzt, werden Handler mit niedrigerer Priorität übersprungen.
- `before_tool_call`: Die Rückgabe von `{ block: false }` gilt als keine Entscheidung (wie das Auslassen von `block`), nicht als Überschreibung.
- `before_install`: Die Rückgabe von `{ block: true }` ist endgültig. Sobald ein Handler diesen Wert setzt, werden Handler mit niedrigerer Priorität übersprungen.
- `before_install`: Die Rückgabe von `{ block: false }` gilt als keine Entscheidung (wie das Auslassen von `block`), nicht als Überschreibung.
- `reply_dispatch`: Die Rückgabe von `{ handled: true, ... }` ist endgültig. Sobald ein Handler den Dispatch beansprucht, werden Handler mit niedrigerer Priorität und der standardmäßige Modell-Dispatch-Pfad übersprungen.
- `message_sending`: Die Rückgabe von `{ cancel: true }` ist endgültig. Sobald ein Handler diesen Wert setzt, werden Handler mit niedrigerer Priorität übersprungen.
- `message_sending`: Die Rückgabe von `{ cancel: false }` gilt als keine Entscheidung (wie das Auslassen von `cancel`), nicht als Überschreibung.
- `message_received`: Verwenden Sie das typisierte Feld `threadId`, wenn Sie eingehendes Thread-/Themen-Routing benötigen. Behalten Sie `metadata` für kanalspezifische Zusatzangaben bei.
- `message_sending`: Verwenden Sie die typisierten Routing-Felder `replyToId` / `threadId`, bevor Sie auf das kanalspezifische `metadata` zurückgreifen.
- `gateway_start`: Verwenden Sie `ctx.config`, `ctx.workspaceDir` und `ctx.getCron?.()` für den Gateway-eigenen Startstatus, anstatt sich auf interne `gateway:startup`-Hooks zu verlassen. Cron wird zu diesem Zeitpunkt möglicherweise noch geladen.
- `cron_reconciled`: Erstellen Sie nach dem Start oder dem erneuten Laden des Schedulers eine vollständige externe Cron-Projektion neu. Sie umfasst `reason` und den effektiven `enabled`-Status einschließlich `enabled: false`, während `ctx.getCron?.()` den exakt abgeglichenen Scheduler zurückgibt. Übergeben Sie `ctx.abortSignal` an dauerhafte Projektionsarbeiten; der Vorgang wird abgebrochen, wenn dieser Scheduler-Snapshot ersetzt oder das Gateway geschlossen wird.
- `cron_changed`: Beobachten Sie Änderungen am Gateway-eigenen Cron-Lebenszyklus. `scheduled`- und `removed`-Ereignisse sind Abgleichhinweise nach dem Commit und kein geordnetes Delta-Protokoll. Das `event.nextRunAtMs` eines geplanten Ereignisses fehlt, wenn der Auftrag keinen nächsten Aktivierungszeitpunkt hat; ein Entfernungsereignis enthält weiterhin den Snapshot des gelöschten Auftrags.

Externe Aktivierungs-Scheduler sollten `cron_changed`-Ereignisse entprellen oder zusammenfassen
und anschließend die vollständige dauerhafte Ansicht aus dem zuletzt von
`cron_reconciled` erfassten Scheduler erneut lesen. Übernehmen Sie den Scheduler nicht aus einem `cron_changed`-Kontext:
Ein losgelöster Hinweis eines älteren Schedulers kann sich mit einem späteren Neuladen überschneiden.

Verwenden Sie `cron_reconciled` als Auslöser für vollständige Snapshots dauerhafter Zustände, die beim
Start des Gateways oder beim Ersetzen des Schedulers geladen werden. Bei einem reinen
Hot-Reload des Plugins wird er nicht erneut wiedergegeben. Beobachtungs-Handler werden parallel ausgeführt, und
Fire-and-forget-Dispatches können sich überschneiden, daher dürfen Verbraucher nicht von der Abschlussreihenfolge der Ereignisse abhängen.
Behalten Sie OpenClaw als maßgebliche Quelle für Fälligkeitsprüfungen und Ausführung bei.

Einen Single-Flight-Adapter mit dauerhafter Ersetzung, Wiederholungsversuchen/Backoff und sauberem
Herunterfahren finden Sie unter [Sichere externe Cron-Projektion](/de/plugins/hooks#safe-external-cron-projection).

### Felder des API-Objekts

| Feld                     | Typ                       | Beschreibung                                                                                           |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `api.id`                 | `string`                  | Plugin-ID                                                                                              |
| `api.name`               | `string`                  | Anzeigename                                                                                            |
| `api.version`            | `string?`                 | Plugin-Version (optional)                                                                              |
| `api.description`        | `string?`                 | Plugin-Beschreibung (optional)                                                                         |
| `api.source`             | `string`                  | Plugin-Quellpfad                                                                                       |
| `api.rootDir`            | `string?`                 | Plugin-Stammverzeichnis (optional)                                                                     |
| `api.config`             | `OpenClawConfig`          | Aktueller Konfigurations-Snapshot (aktiver In-Memory-Laufzeit-Snapshot, sofern verfügbar)              |
| `api.pluginConfig`       | `Record<string, unknown>` | Plugin-spezifische Konfiguration aus `plugins.entries.<id>.config`                                    |
| `api.runtime`            | `PluginRuntime`           | [Laufzeit-Hilfsfunktionen](/de/plugins/sdk-runtime)                                                       |
| `api.logger`             | `PluginLogger`            | Bereichsbezogener Logger (`debug`, `info`, `warn`, `error`)                                              |
| `api.registrationMode`   | `PluginRegistrationMode`  | Aktueller Lademodus; `"setup-runtime"` ist das schlanke Start-/Einrichtungsfenster vor dem vollständigen Einstieg |
| `api.resolvePath(input)` | `(string) => string`      | Pfad relativ zum Plugin-Stammverzeichnis auflösen                                                      |

## Konvention für interne Module

Verwenden Sie innerhalb Ihres Plugins lokale Barrel-Dateien für interne Importe:

```text
my-plugin/
  api.ts            # Öffentliche Exporte für externe Verbraucher
  runtime-api.ts    # Nur interne Laufzeitexporte
  index.ts          # Plugin-Einstiegspunkt
  setup-entry.ts    # Schlanker Einstieg nur für die Einrichtung (optional)
```

<Warning>
  Importieren Sie Ihr eigenes Plugin im Produktionscode niemals über `openclaw/plugin-sdk/<your-plugin>`.
  Leiten Sie interne Importe über `./api.ts` oder
  `./runtime-api.ts`. Der SDK-Pfad dient ausschließlich als externer Vertrag.
</Warning>

Über Fassaden geladene öffentliche Schnittstellen gebündelter Plugins (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` und ähnliche öffentliche Einstiegspunktdateien) verwenden bevorzugt den
aktiven Snapshot der Laufzeitkonfiguration, wenn OpenClaw bereits ausgeführt wird. Wenn noch kein
Laufzeit-Snapshot vorhanden ist, greifen sie auf die aufgelöste Konfigurationsdatei auf dem Datenträger zurück.
Fassaden paketierter gebündelter Plugins sollten über die Plugin-Fassaden-Loader von OpenClaw
geladen werden; direkte Importe aus `dist/extensions/...` umgehen die Manifest-
und Laufzeit-Sidecar-Prüfungen, die paketierte Installationen für Plugin-eigenen Code verwenden.

Provider-Plugins können ein eng abgegrenztes, Plugin-lokales Vertrags-Barrel bereitstellen, wenn ein
Hilfsprogramm bewusst Provider-spezifisch ist und noch nicht in einen generischen SDK-
Unterpfad gehört. Gebündelte Beispiele:

- **Anthropic**: öffentliche `api.ts`- / `contract-api.ts`-Schnittstelle für Claude-
  Beta-Header- und `service_tier`-Stream-Hilfsprogramme.
- **`@openclaw/openai-provider`**: `api.ts` exportiert Provider-Builder,
  Hilfsprogramme für Standardmodelle und Echtzeit-Provider-Builder.
- **`@openclaw/openrouter-provider`**: `api.ts` exportiert den Provider-Builder
  sowie Hilfsprogramme für Onboarding und Konfiguration.

<Warning>
  Produktionscode von Erweiterungen sollte ebenfalls Importe aus `openclaw/plugin-sdk/<other-plugin>`
  vermeiden. Wenn ein Hilfsprogramm tatsächlich gemeinsam genutzt wird, verschieben Sie es in einen neutralen SDK-Unterpfad
  wie `openclaw/plugin-sdk/speech`, `.../provider-model-shared` oder eine andere
  fähigkeitsorientierte Schnittstelle, statt zwei Plugins miteinander zu koppeln.
</Warning>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Einstiegspunkte" icon="door-open" href="/de/plugins/sdk-entrypoints">
    Optionen für `definePluginEntry` und `defineChannelPluginEntry`.
  </Card>
  <Card title="Laufzeit-Hilfsprogramme" icon="gears" href="/de/plugins/sdk-runtime">
    Vollständige Referenz des `api.runtime`-Namensraums.
  </Card>
  <Card title="Einrichtung und Konfiguration" icon="sliders" href="/de/plugins/sdk-setup">
    Paketierung, Manifeste und Konfigurationsschemas.
  </Card>
  <Card title="Tests" icon="vial" href="/de/plugins/sdk-testing">
    Testhilfsprogramme und Lint-Regeln.
  </Card>
  <Card title="SDK-Migration" icon="arrows-turn-right" href="/de/plugins/sdk-migration">
    Migration von veralteten Schnittstellen.
  </Card>
  <Card title="Plugin-Interna" icon="diagram-project" href="/de/plugins/architecture">
    Detaillierte Architektur und Fähigkeitsmodell.
  </Card>
</CardGroup>
