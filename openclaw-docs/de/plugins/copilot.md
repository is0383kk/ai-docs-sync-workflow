---
read_when:
    - Sie möchten das GitHub Copilot SDK-Harness für einen Agenten verwenden
    - Sie benötigen Konfigurationsbeispiele für die `copilot`-Runtime.
    - Sie verbinden einen Agenten mit einem Copilot-Abonnement (github / openclaw / copilot) und möchten, dass er über die Copilot CLI ausgeführt wird.
summary: OpenClaw-Turns eingebetteter Agenten über das externe GitHub-Copilot-SDK-Harness ausführen
title: Copilot-SDK-Harness
x-i18n:
    generated_at: "2026-07-26T19:06:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4b67959c2c72bda97a81d0b45bc32ba363373064ec40c54f9709705dd15dd9fc
    source_path: plugins/copilot.md
    workflow: 16
---

Das externe Plugin `@openclaw/copilot` führt eingebettete Agent-Durchläufe des Copilot-Abonnements
über die GitHub Copilot CLI (`@github/copilot-sdk`) statt über
das integrierte Harness von OpenClaw aus. Die Copilot-CLI-Sitzung verwaltet die untergeordnete
Agent-Schleife: native Werkzeugausführung, native Compaction (`infiniteSessions`) und
CLI-verwalteten Thread-Status unter `copilotHome`. OpenClaw verwaltet weiterhin Chat-
Kanäle, Sitzungsdateien, Modellauswahl, dynamische Werkzeuge (über eine Bridge), Genehmigungen,
Medienzustellung, die sichtbare Transkriptspiegelung, `/btw` Nebenfragen (siehe
[Nebenfragen (`/btw`)](#side-questions-btw)) und `openclaw doctor`.

Einen Überblick über die umfassendere Aufteilung zwischen Modell, Provider und Laufzeit finden Sie unter
[Agent-Laufzeiten](/de/concepts/agent-runtimes).

## Anforderungen

- OpenClaw mit installiertem Plugin `@openclaw/copilot`.
- Wenn Ihre Konfiguration `plugins.allow` verwendet, nehmen Sie `copilot` auf (die vom
  Plugin deklarierte Manifest-ID). Ein Eintrag für den npm-Paketnamen
  `@openclaw/copilot` in der Zulassungsliste stimmt nicht überein und das Plugin bleibt blockiert, selbst wenn
  `agentRuntime.id: "copilot"` festgelegt ist.
- Ein GitHub-Copilot-Abonnement, das die Copilot CLI verwenden kann, oder eine
  Umgebungsvariable `gitHubToken` bzw. ein Authentifizierungsprofileintrag für Headless- oder Cron-Ausführungen.
- Ein beschreibbares Verzeichnis `copilotHome`. Standardmäßig `<agentDir>/copilot`, wenn
  OpenClaw ein Agent-Verzeichnis bereitstellt, andernfalls
  `~/.openclaw/agents/<agentId>/copilot`.

`openclaw doctor` führt den [Doctor-Vertrag](#doctor) des Plugins für
die Zuständigkeit für den Sitzungsstatus und zukünftige Konfigurationsmigrationen aus. Die
Copilot-CLI-Umgebung wird dabei nicht geprüft.

## Installation

Die Copilot-Laufzeit wird als externes Plugin ausgeliefert, damit das zentrale Paket `openclaw`
weder `@github/copilot-sdk` noch dessen plattformspezifische
CLI-Binärdatei `@github/copilot-<platform>-<arch>` enthält (zusammen etwa 260 MB).
Installieren Sie es nur für Agenten, die diese Laufzeit ausdrücklich verwenden:

```bash
openclaw plugins install @openclaw/copilot
```

Der Einrichtungsassistent installiert das Plugin automatisch, wenn Sie erstmals
ein Modell `github-copilot/*` auswählen **und** Ihre Konfiguration dieses Modell (oder dessen
Provider) über `agentRuntime: { id: "copilot" }` an die Copilot-Laufzeit weiterleitet; siehe
[Schnellstart](#quickstart). Ohne diese Aktivierung verwendet OpenClaw seinen integrierten
GitHub-Copilot-Provider und installiert dieses Plugin nicht.

Die Laufzeit sucht das SDK in dieser Reihenfolge:

1. `import("@github/copilot-sdk")` aus dem installierten Paket `@openclaw/copilot`.
2. Das Ausweichverzeichnis `~/.openclaw/npm-runtime/copilot/` (veraltetes Ziel für die bedarfsgesteuerte
   Installation).

Bei einem fehlenden SDK wird ein einzelner Fehler mit dem Code `COPILOT_SDK_MISSING` und dem
obigen Befehl zur Neuinstallation ausgegeben.

## Schnellstart

Binden Sie ein Modell (oder einen Provider) an das Harness:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/auto",
      models: {
        "github-copilot/auto": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

Legen Sie `agentRuntime.id` für einen einzelnen Modelleintrag fest, um nur dieses Modell über
das Harness weiterzuleiten, oder für einen Provider, um jedes Modell dieses Providers weiterzuleiten.

`github-copilot/auto` ist der portable Ausgangspunkt. Benannte Copilot-Modelle hängen
von Konto- und Organisationsrichtlinien ab; vergewissern Sie sich, dass Ihre authentifizierte
Copilot CLI ein Modell tatsächlich bereitstellt, bevor Sie es fest vorgeben.

## Unterstützte Provider

Das Harness unterstützt den kanonischen Provider `github-copilot` (verwaltet von
`extensions/github-copilot`) sowie benutzerdefinierte Einträge `models.providers`, wenn das
Modell einen nicht leeren Wert für `baseUrl` und eine dieser Formen von `api` aufweist:

- `anthropic-messages`
- `azure-openai-responses`
- `ollama` (OpenAI-kompatible Vervollständigungen)
- `openai-completions`
- `openai-responses`

Native Provider-IDs (`openai`, `anthropic`, `google`, `ollama`) verbleiben in der Zuständigkeit
ihrer nativen Laufzeiten. Verwenden Sie stattdessen eine eigene benutzerdefinierte Provider-ID, um einen Endpunkt
über Copilot BYOK weiterzuleiten.

Copilot-BYOK-Endpunkte müssen öffentliche HTTPS-URLs sein. Das Harness stellt dem
Copilot SDK für jeden Versuch einen Loopback-Proxy bereit und leitet anschließend den Provider-Datenverkehr
über den geschützten Abrufpfad von OpenClaw weiter, sodass DNS-Pinning und SSRF-Richtlinien
in der Zuständigkeit von OpenClaw bleiben. Verwenden Sie die native OpenClaw-Laufzeit für lokale Ollama-,
LM-Studio- oder LAN-Modellserver.

## BYOK

Copilot BYOK verwendet den sitzungsbezogenen Vertrag des SDK für benutzerdefinierte Provider. OpenClaw
übergibt den aufgelösten Modellendpunkt, API-Schlüssel, Bearer-Token-Modus, Header, die Modell-
ID sowie Kontext- und Ausgabelimits; die Provider-Transportlogik verbleibt im SDK und nicht
im Kern.

```json5
{
  agents: {
    defaults: {
      model: "custom-proxy/llama-3.1-8b",
      models: {
        "custom-proxy/llama-3.1-8b": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "https://api.example.com/v1",
        apiKey: "${CUSTOM_PROXY_API_KEY}",
        api: "openai-responses",
        authHeader: true,
        models: [{ id: "llama-3.1-8b", name: "Llama 3.1 8B" }],
      },
    },
  },
}
```

BYOK-Sitzungen erhalten eigene Schlüssel, getrennt von Abonnementsitzungen sowie von anderen
BYOK-Endpunkten oder Anmeldedaten. Wenn der Schlüssel, die Header, das Modell oder der Endpunkt
geändert werden, startet eine neue Copilot-SDK-Sitzung, statt inkompatiblen Status fortzusetzen.

## Authentifizierung

Rangfolge, die während `runCopilotAttempt` für jeden Agenten angewendet wird:

1. **Explizites `useLoggedInUser: true`** in der Eingabe des Versuchs – verwendet den
   angemeldeten Benutzer der Copilot CLI unter `copilotHome` des Agenten.
2. **Explizites `gitHubToken`** in der Eingabe des Versuchs (erfordert `profileId` +
   `profileVersion`). Für direkte CLI-Aufrufe und Tests, die die
   Auflösung von Authentifizierungsprofilen umgehen müssen.
3. **Vertragsseitig aufgelöstes `resolvedApiKey` + `authProfileId`** – der primäre
   Produktionspfad. Der Kern löst das konfigurierte Authentifizierungsprofil `github-copilot`
   (`src/infra/provider-usage.auth.ts:resolveProviderAuths`) des Agenten auf, bevor
   das Harness aufgerufen wird, sodass ein Authentifizierungsprofil `github-copilot:<profile>`
   für Headless-, Cron- oder Mehrprofilkonfigurationen ohne Umgebungsvariablen durchgängig funktioniert.
4. **Ausweichlösung über Umgebungsvariablen**, in dieser Reihenfolge geprüft (der erste nicht leere Wert wird verwendet,
   leere Zeichenfolgen gelten als nicht vorhanden; entspricht der ausgelieferten Rangfolge des Providers `github-copilot`
   in `extensions/github-copilot/auth.ts`):
   1. `OPENCLAW_GITHUB_TOKEN` – Harness-spezifische Überschreibung; ermöglicht die feste Vorgabe eines
      Tokens für das OpenClaw-Harness, ohne die systemweite Konfiguration von `gh` /
      Copilot CLI zu verändern.
   2. `COPILOT_GITHUB_TOKEN` – standardmäßige Umgebungsvariable des Copilot SDK / der CLI.
   3. `GH_TOKEN` – standardmäßige Umgebungsvariable der CLI `gh`.
   4. `GITHUB_TOKEN` – generischer Ausweichwert für GitHub-Token.

   Die synthetisierte Pool-Profil-ID lautet `env:<NAME>`; die Profilversion ist ein
   nicht umkehrbarer SHA-256-Fingerabdruck des Tokens, sodass eine Änderung des Umgebungswerts
   den Client-Pool zuverlässig verwirft.

5. **Standardwert `useLoggedInUser`**, wenn kein Token-Signal verfügbar ist.

Jeder Agent erhält ein eigenes `copilotHome`, damit Copilot-CLI-Token, Sitzungen und
Konfigurationen niemals zwischen Agenten auf demselben Rechner übertragen werden. Standard:
`<agentDir>/copilot` (hält den SDK-Status aus demselben Verzeichnis wie
`models.json` / `auth-profiles.json` von OpenClaw heraus) oder
`~/.openclaw/agents/<agentId>/copilot`, wenn kein Agent-Verzeichnis bereitgestellt wird.
Überschreiben Sie den Wert mit `copilotHome: <path>` in der Eingabe des Versuchs, um einen benutzerdefinierten
Speicherort zu verwenden (beispielsweise ein freigegebenes Volume für eine Migration).

Live-Harness-Tests verwenden `OPENCLAW_COPILOT_AGENT_LIVE_TOKEN` für ein direktes
Token. Die gemeinsam genutzte Live-Test-Einrichtung entfernt `COPILOT_GITHUB_TOKEN`, `GH_TOKEN`
und `GITHUB_TOKEN`, nachdem echte Authentifizierungsprofile im isolierten Test-
Home-Verzeichnis bereitgestellt wurden. Dadurch verhindert ein über die dedizierte Variable übergebener Wert `gh auth token`
fälschliche Überspringvorgänge, ohne in andere Testsuiten zu gelangen.

## Konfigurationsoberfläche

Das Harness liest die Konfiguration aus der versuchsbezogenen Eingabe (`runCopilotAttempt({...})`)
und aus einer kleinen Gruppe von Umgebungsstandardwerten innerhalb von `extensions/copilot/src/`:

| Feld                     | Zweck                                                                                                                                                                                                                                                                                           |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilotHome`            | Agentenspezifisches CLI-Statusverzeichnis (Standardwerte siehe oben).                                                                                                                                                                                                                            |
| `model`                  | Zeichenfolge oder `{ provider, id, api?, baseUrl?, headers?, authHeader? }`. Weglassen, um die normale Modellauswahl des Agenten zu verwenden; das Harness prüft, ob der aufgelöste Provider unterstützt wird.                                                                                                                         |
| `reasoningEffort`        | `"low" \| "medium" \| "high" \| "xhigh"`. Wird aus der Auflösung von `ThinkLevel` / `ReasoningLevel` durch OpenClaw in `auto-reply/thinking.ts` abgeleitet.                                                                                                                                                           |
| `infiniteSessionConfig`  | Optionale Überschreibung für den vom SDK-Block `infiniteSessions`, der durch `harness.compact` gesteuert wird. Kann unverändert bleiben.                                                                                                                                                         |
| `hooksConfig`            | Optionale native Copilot-SDK-Konfiguration `SessionHooks` für Werkzeug-/MCP-, Benutzeraufforderungs-, Sitzungs- und Fehlerrückrufe. Unabhängig von den portablen Lebenszyklus-Hooks von OpenClaw.                                                                                                |
| `permissionPolicy`       | Optionale Überschreibung für den Handler `onPermissionRequest` des SDK für integrierte SDK-Werkzeugtypen (`shell`, `write`, `read`, `url`, `mcp`, `memory`, `hook`). Standardmäßig `rejectAllPolicy` als Sicherheitsnetz; unter [Berechtigungen und ask_user](#permissions-and-ask_user) wird erläutert, warum er tatsächlich nie ausgelöst wird. |
| `enableSessionTelemetry` | Optionales Telemetrie-Flag für SDK-Sitzungen.                                                                                                                                                                                                                                                    |

OpenClaw-Plugin-Hooks benötigen keine Copilot-spezifische Versuchskonfiguration. Das
Harness führt `before_prompt_build`, `llm_input`, `llm_output` und `agent_end` über die
standardmäßigen Harness-Hilfsfunktionen aus. Erfolgreiche SDK-Compactions führen außerdem
`before_compaction` und `after_compaction` aus. Über eine Bridge angebundene OpenClaw-Werkzeuge führen
`before_tool_call` aus und melden `after_tool_call`; `hooksConfig` bleibt
nativen, ausschließlich SDK-internen Rückrufen vorbehalten, für die es keine portable Entsprechung gibt.

Keine andere Komponente in OpenClaw muss diese Felder kennen. Andere Plugins,
Kanäle und der Kern sehen nur die standardmäßige Form `AgentHarnessAttemptParams` /
`AgentHarnessAttemptResult`.

## Compaction

Wenn `harness.compact` ausgeführt wird, führt das Copilot-SDK-Harness Folgendes aus:

1. Setzt die nachverfolgte SDK-Sitzung fort, ohne ausstehende Arbeit fortzuführen.
2. Ruft den sitzungsbezogenen RPC des SDK zur Komprimierung des Verlaufs auf.
3. Gibt das Ergebnis der SDK-Compaction zurück, ohne Kompatibilitätsmarkierungsdateien
   im Arbeitsbereich zu schreiben.

Die OpenClaw-seitige Transkriptspiegelung (unten) empfängt weiterhin Nachrichten nach der Compaction,
sodass der für Benutzer sichtbare Chatverlauf konsistent bleibt.

## Transkriptspiegelung

`runCopilotAttempt` schreibt die spiegelbaren Nachrichten jedes Durchlaufs doppelt in das
OpenClaw-Audit-Transkript über
`extensions/copilot/src/dual-write-transcripts.ts`. Die Spiegelung ist pro
Sitzung (`copilot:${sessionId}`) begrenzt und pro Nachricht verschlüsselt
(`${role}:${sha256_16(role,content)}`), sodass erneut ausgegebene Einträge aus vorherigen Durchläufen
mit vorhandenen Schlüsseln auf dem Datenträger kollidieren, anstatt dupliziert zu werden.

Zwei Ebenen der Fehlerbegrenzung umschließen die Spiegelung, sodass ein Fehler beim Schreiben
des Transkripts niemals den Versuch fehlschlagen lässt: ein interner Best-Effort-Wrapper sowie ein
Defense-in-Depth-`.catch(...)` auf Versuchsebene. Fehler werden protokolliert, nicht
weitergegeben.

## Nebenfragen (`/btw`)

`/btw` ist auf diesem Harness **nicht** nativ. `createCopilotAgentHarness()`
lässt `harness.runSideQuestion` bewusst undefiniert
(bestätigt in `extensions/copilot/harness.test.ts`, `describe("runSideQuestion")`),
sodass der `/btw`-Dispatcher von OpenClaw (`src/agents/btw.ts`) auf denselben
Pfad zurückfällt, den er für jede Nicht-Codex-Laufzeit verwendet: Der konfigurierte Modell-Provider
wird direkt mit einem kurzen Nebenfragen-Prompt aufgerufen, und die Antwort wird über
`streamSimple` zurückgestreamt (keine CLI-Sitzung, kein zusätzlicher Pool-Platz).

Dadurch bleiben Copilot-CLI-Sitzungen für die Hauptdurchlaufschleife des Agenten reserviert und
das Verhalten von `/btw` bleibt mit anderen Nicht-Codex-Laufzeiten identisch.

## Doctor

`extensions/copilot/doctor-contract-api.ts` wird automatisch von
`src/plugins/doctor-contract-registry.ts` geladen. Es stellt Folgendes bereit:

- Ein leeres `legacyConfigRules` (noch keine veralteten Felder).
- Ein wirkungsloses `normalizeCompatibilityConfig` (wird beibehalten, damit zukünftige Feldstilllegungen
  einen stabilen Ort im Quellbaum haben).
- Ein `sessionRouteStateOwners`-Eintrag: Provider `github-copilot`, Laufzeit
  `copilot`, CLI-Sitzungsschlüssel `copilot`, Präfix des Authentifizierungsprofils `github-copilot:`.

## Einschränkungen

- Das Harness beansprucht `github-copilot` sowie nicht zugeordnete benutzerdefinierte BYOK-Provider-IDs.
  Manifest-eigene native Provider-IDs verbleiben bei ihrer zugehörigen Laufzeit, selbst wenn
  `agentRuntime.id` auf `copilot` erzwungen wird.
- Keine TUI-Oberfläche; die TUI von PI bleibt der Fallback für Laufzeiten ohne
  gleichwertige Oberfläche.
- Der PI-Sitzungsstatus wird nicht migriert, wenn ein Agent zu `copilot` wechselt.
  Die Auswahl erfolgt pro Versuch; vorhandene PI-Sitzungen bleiben gültig.
- `ask_user` verwendet die Provider-neutrale Gateway-Fragenlaufzeit. Die Control
  UI zeigt dieselbe Fragenkarte wie bei anderen OpenClaw-Fragen, unterstützte
  Kanäle stellen Auswahlschaltflächen dar, und die nächste eingereihte Klartextnachricht
  löst diesen Gateway-Eintrag auf, bevor die SDK-Anfrage zurückkehrt.

## Berechtigungen und ask_user

Die Durchsetzung von Berechtigungen für überbrückte OpenClaw-Tools erfolgt **innerhalb des Tool-
Wrappers**, nicht über den `onPermissionRequest`-Callback des SDK. Derselbe
`wrapToolWithBeforeToolCallHook`, den PI verwendet
(`src/agents/agent-tools.before-tool-call.ts`), wird durch
`createOpenClawCodingTools` auf jedes Coding-Tool angewendet: Schleifenerkennung, Richtlinien für
vertrauenswürdige Plugins, Hooks vor Tool-Aufrufen und zweiphasige Plugin-Genehmigungen über
das Gateway (`plugin.approval.request`) durchlaufen exakt denselben Code-
Pfad wie native PI-Versuche.

Jedes vom Copilot-Tool-Bridge zurückgegebene SDK-Tool ist wie folgt gekennzeichnet:

- `overridesBuiltInTool: true` — ersetzt das integrierte Tool der Copilot CLI
  mit demselben Namen (edit, read, write, bash, ...), sodass jeder Tool-Aufruf zurück
  zu OpenClaw geleitet wird.
- `skipPermission: true` — weist das SDK an,
  `onPermissionRequest({kind: "custom-tool"})` vor dem Aufruf des Tools nicht auszulösen. Der
  umschlossene `execute()` führt bereits die umfassendere OpenClaw-Richtlinienprüfung durch; ein
  Prompt auf SDK-Ebene würde entweder die Durchsetzung von OpenClaw umgehen
  (alles zulassen) oder jeden Tool-Aufruf blockieren (alles ablehnen) — beides entspricht nicht
  der Parität mit PI.

Das Codex-Harness im Quellbaum verwendet dieselbe Aufteilung: Überbrückte OpenClaw-Tools werden
umschlossen (`extensions/codex/src/app-server/dynamic-tools.ts`), und die eigenen nativen Genehmigungsarten
des codex-app-server
(`item/commandExecution/requestApproval`, `item/fileChange/requestApproval`,
`item/permissions/requestApproval`) werden über `plugin.approval.request`
(`extensions/codex/src/app-server/approval-bridge.ts`) geleitet. Das Äquivalent im Copilot SDK
— ein Fail-Closed-`rejectAllPolicy` für jede Nicht-`custom-tool`-Art,
die jemals `onPermissionRequest` erreicht — ist dasselbe Sicherheitsnetz und wird
in der Praxis nie ausgelöst, weil `overridesBuiltInTool: true` jedes
integrierte Tool verdrängt.

Damit die Ebene der umschlossenen Tools Richtlinienentscheidungen treffen kann, die denen von PI entsprechen,
leitet das Harness den vollständigen PI-Kontext für Tools eines Versuchs an
`createOpenClawCodingTools` weiter: Identität (`senderIsOwner`, `memberRoleIds`,
`ownerOnlyToolAllowlist`, ...), Kanal/Routing (`groupId`,
`currentChannelId`, `replyToMode`, Umschalter für Nachrichtentools), Authentifizierung
(`authProfileStore`), Ausführungsidentität (`sessionKey` / `runSessionKey`, abgeleitet
aus `sandboxSessionKey`, `runId`), Modellkontext (`modelApi`,
`modelContextWindowTokens`, `modelCompat`, `modelHasVision`) und Ausführungs-Hooks
(`onToolOutcome`, `onYield`). Ohne diese Felder lehnen nur für Eigentümer geltende Zulassungslisten
standardmäßig stillschweigend ab, Richtlinien für das Vertrauen in Plugins können nicht auf den richtigen
Geltungsbereich aufgelöst werden, und `session_status: "current"` wird auf einen veralteten Sandbox-Schlüssel aufgelöst. Der
Bridge-Builder ist `extensions/copilot/src/tool-bridge.ts` und spiegelt den maßgeblichen PI-
Aufruf unter `src/agents/embedded-agent-runner/run/attempt.ts:1262`.
`runAttempt` löst den Sandbox-Kontext über die gemeinsame
`resolveSandboxContext`-Schnittstelle auf, übergibt dem SDK ein effektives Arbeitsverzeichnis
und leitet `sandbox` sowie den Arbeitsbereich für das Erzeugen von Subagenten an die Tool-
Bridge weiter. Die Bridge leitet außerdem die begrenzten Steuerelemente für die Tool-Erstellung weiter, die sie
an der SDK-Grenze durchsetzen kann: `includeCoreTools`, die Tool-
Zulassungsliste der Laufzeit und `toolConstructionPlan`.

Die Bridge verwendet für die Parität mit PI außerdem den gemeinsamen Harness-Helfer für Tool-Oberflächen aus
`openclaw/plugin-sdk/agent-harness-tool-runtime`. Wenn
die Tool-Suche aktiviert ist, sieht das SDK kompakte Steuerungstools sowie einen ausgeblendeten
Katalog-Executor anstelle jedes OpenClaw-Tool-Schemas. Wenn der Code-Modus
aktiviert ist, erstellt der Helfer dieselbe Steuerungsoberfläche und denselben Katalog-
Lebenszyklus für den Code-Modus, die von anderen Agenten-Harnesses verwendet werden. Schlanke Standardwerte
für lokale Modelle, laufzeitkompatible Schemafilterung, Verzeichnishydrierung und Katalog-
Bereinigung verbleiben vollständig im gemeinsamen Helfer, damit Copilot und Codex-nahe
Harnesses nicht auseinanderdriften.

### GitHub-Token auf Sitzungsebene

Der Vertrag des Copilot SDK unterscheidet das GitHub-Token auf **Client-Ebene**
(`CopilotClientOptions.gitHubToken`, authentifiziert den CLI-Prozess selbst)
vom Token auf **Sitzungsebene** (`SessionConfig.gitHubToken`, bestimmt
Inhaltsausschluss, Modell-Routing und Kontingent für diese Sitzung; wird sowohl bei
`createSession` als auch bei `resumeSession` berücksichtigt). Das Harness löst die Authentifizierung einmal über
`resolveCopilotAuth` auf und setzt beide Felder, wenn der Authentifizierungsmodus `gitHubToken` ist
(ein explizites `auth.gitHubToken` oder ein vertragsgemäß aufgelöstes `resolvedApiKey` aus
einem konfigurierten `github-copilot`-Authentifizierungsprofil). Wenn der aufgelöste Modus
`useLoggedInUser` ist, wird das Feld auf Sitzungsebene weggelassen, sodass das SDK
die Identität weiterhin von der angemeldeten Identität ableitet.

`ask_user` verwendet `SessionConfig.onUserInputRequest`. Die Bridge registriert SDK-
Auswahlmöglichkeiten oder optionenlose Freitext-Prompts als Gateway-Fragen, akzeptiert Auswahl-
indizes oder Beschriftungen für Anfragen mit fester Auswahl und akzeptiert Freitextantworten,
wenn die SDK-Anfrage sie zulässt. Das Abbrechen des OpenClaw-Versuchs bricht den
Gateway-Eintrag ab und gibt eine leere SDK-Antwort zurück.

## Verwandte Themen

- [Agentenlaufzeiten](/de/concepts/agent-runtimes)
- [Codex-Harness](/de/plugins/codex-harness)
- [Agenten-Harness-Plugins (SDK-Referenz)](/de/plugins/sdk-agent-harness)
