---
read_when:
    - Sie möchten die Kosten für Prompt-Token durch Cache-Aufbewahrung reduzieren
    - Sie benötigen in Multi-Agent-Setups ein Cache-Verhalten pro Agent.
    - Sie stimmen Heartbeat und Cache-TTL-Bereinigung aufeinander ab
summary: Optionen für Prompt-Caching, Zusammenführungsreihenfolge, Provider-Verhalten und Optimierungsmuster
title: Prompt-Caching
x-i18n:
    generated_at: "2026-07-26T19:13:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99dfd3d226d37014110adf16818051236114dcb0277e9b4d13eaced0f1fc03aa
    source_path: reference/prompt-caching.md
    workflow: 16
---

Prompt-Caching ermöglicht es einem Modell-Provider, ein unverändertes Prompt-Präfix (System-/Entwickleranweisungen, Tool-Definitionen und anderen stabilen Kontext) über mehrere Durchläufe hinweg wiederzuverwenden, statt es bei jeder Anfrage erneut zu verarbeiten. Dies reduziert Token-Kosten und Latenz bei lang laufenden Sitzungen mit wiederkehrendem Kontext.

OpenClaw normalisiert die Providernutzung in `cacheRead` und `cacheWrite`, sofern die Upstream-API diese Zähler bereitstellt. Nutzungszusammenfassungen (`/status` und ähnliche) greifen auf den letzten Nutzungseintrag im Transkript zurück, wenn der Snapshot der laufenden Sitzung keine Cache-Zähler enthält; ein von null verschiedener Live-Wert hat immer Vorrang vor dem Rückfallwert.

Provider-Referenzen:

- [Anthropic-Prompt-Caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [OpenAI-Prompt-Caching](https://developers.openai.com/api/docs/guides/prompt-caching)

## Primäre Einstellungen

### `cacheRetention`

Werte: `"none" | "short" | "long"`. Konfigurierbar als globaler Standardwert, pro Modell und pro Agent.
`"standard"` ist kein Alias; verwenden Sie `"short"` für das standardmäßige Cache-Zeitfenster des Providers. Ungültige Werte werden mit einer Warnung ignoriert.

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # überschreibt den globalen Standardwert für dieses Modell
  list:
    - id: "alerts"
      params:
        cacheRetention: "none" # überschreibt beide Standardwerte für diesen Agent
```

Zusammenführungsreihenfolge (spätere Werte haben Vorrang):

1. `agents.defaults.params` - globaler Standardwert für alle Modelle
2. `agents.defaults.models["provider/model"].params` - Überschreibung pro Modell
3. `agents.entries.*.params` - Überschreibung pro Agent, abgeglichen anhand der Agent-ID

Quelle: `src/agents/embedded-agent-runner/extra-params.ts` (`resolveExtraParams`).

### `contextPruning.mode: "cache-ttl"`

Entfernt alten Tool-Ergebniskontext nach Ablauf des Cache-TTL-Zeitfensters, damit eine Anfrage nach einer Leerlaufphase keinen übermäßig großen Verlauf erneut zwischenspeichert.

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

Das vollständige Verhalten wird unter [Sitzungsbereinigung](/de/concepts/session-pruning) beschrieben.

### Warmhalten durch Heartbeat

Heartbeat kann Cache-Zeitfenster warmhalten und wiederholte Cache-Schreibvorgänge nach Leerlaufphasen reduzieren. Global (`agents.defaults.heartbeat`) oder pro Agent (`agents.entries.*.heartbeat`) konfigurierbar.

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

## Provider-Verhalten

### Anthropic (direkte API und Vertex AI)

- `cacheRetention` wird für die Provider `anthropic` und `anthropic-vertex` sowie für Claude-Modelle auf `amazon-bedrock` und benutzerdefinierten, mit `anthropic-messages` kompatiblen Endpunkten unterstützt, wenn `cacheRetention` explizit festgelegt ist.
- Wenn kein Wert festgelegt ist, setzt OpenClaw `cacheRetention: "short"` für direktes Anthropic ein (nur für die Provider `anthropic` und `anthropic-vertex`; andere Routen der Anthropic-Familie erfordern einen expliziten Wert).
- Native Antworten von Anthropic Messages stellen `cache_read_input_tokens` und `cache_creation_input_tokens` bereit, die auf `cacheRead` und `cacheWrite` abgebildet werden.
- `cacheRetention: "short"` wird dem standardmäßigen flüchtigen 5-Minuten-Cache zugeordnet. `cacheRetention: "long"` fordert bei expliziter Festlegung die 1-Stunden-TTL (`cache_control: { type: "ephemeral", ttl: "1h" }`) an. Eine implizite bzw. durch eine Umgebungsvariable gesteuerte lange Aufbewahrung (`OPENCLAW_CACHE_RETENTION=long` ohne explizites `cacheRetention`) wird nur auf `api.anthropic.com`- oder Vertex-AI-Hosts (`aiplatform.googleapis.com` / `*-aiplatform.googleapis.com`) auf die 1-Stunden-TTL hochgestuft; andere Hosts behalten den 5-Minuten-Cache bei.

Quelle: `packages/ai/src/transports/anthropic-payload-policy.ts` (`resolveAnthropicEphemeralCacheControl`, `isLongTtlEligibleEndpoint`).

### OpenAI (direkte API)

- Prompt-Caching erfolgt bei unterstützten aktuellen Modellen automatisch; OpenClaw fügt keine Cache-Markierungen auf Blockebene ein.
- OpenClaw sendet `prompt_cache_key`, um das Cache-Routing über mehrere Durchläufe hinweg stabil zu halten. Direkte `api.openai.com`-Hosts erhalten dies automatisch. OpenAI-kompatible Proxys (oMLX, llama.cpp, benutzerdefinierte Endpunkte) müssen zur Aktivierung `compat.supportsPromptCacheKey: true` in der Modellkonfiguration festlegen – bei einem Proxy wird dies niemals automatisch erkannt.
- `prompt_cache_retention: "24h"` wird nur hinzugefügt, wenn `cacheRetention: "long"` ausgewählt ist und der aufgelöste Endpunkt sowohl den Cache-Schlüssel als auch die lange Aufbewahrung unterstützt (`compat.supportsLongCacheRetention`, standardmäßig wahr; Kompatibilitätsprofile von Together AI und Cloudflare deaktivieren dies). `cacheRetention: "none"` unterdrückt beide Felder.
- Cache-Treffer werden über `usage.prompt_tokens_details.cached_tokens` (Chat Completions) oder `input_tokens_details.cached_tokens` (Responses API) bereitgestellt und auf `cacheRead` abgebildet.
- Nutzdaten der Responses API können außerdem `input_tokens_details.cache_write_tokens` bereitstellen, das auf `cacheWrite` abgebildet und zum Cache-Schreibpreis des Modells abgerechnet wird; bei Responses-Nutzdaten ohne dieses Feld bleibt `cacheWrite` auf `0`. Die Chat Completions API von OpenAI dokumentiert und liefert keinen `cache_write_tokens`-Zähler, OpenClaw liest dort jedoch weiterhin `prompt_tokens_details.cache_write_tokens` für OpenRouter-kompatible und DeepSeek-artige Proxys, die eine separate Schreibanzahl melden.
- In der Praxis verhält sich OpenAI eher wie ein Cache für das anfängliche Präfix als wie Anthrophics Wiederverwendung des fortlaufenden vollständigen Verlaufs – siehe unten [Erwartungen an OpenAI im Live-Betrieb](#openai-live-expectations).

### Amazon Bedrock

- Anthropic-Claude-Modellreferenzen (`amazon-bedrock/*anthropic.claude*` sowie die AWS-System-Inferenzprofilpräfixe `us.`/`eu.`/`global.anthropic.claude*`) unterstützen die explizite Weitergabe von `cacheRetention`.
- Bedrock-Modelle, die nicht von Anthropic stammen (beispielsweise `amazon.nova-*`), werden zur Laufzeit unabhängig von einem konfigurierten `cacheRetention`-Wert ohne Cache-Aufbewahrung aufgelöst.
- Undurchsichtige ARNs von Bedrock-Anwendungsinferenzprofilen (Profil-IDs, die `claude` nicht enthalten) werden ebenfalls ohne Cache-Aufbewahrung aufgelöst, sofern `cacheRetention` nicht explizit festgelegt ist, da die Modellfamilie nicht allein aus dem ARN abgeleitet werden kann.

### OpenRouter

Für `openrouter/anthropic/*`-Modellreferenzen fügt OpenClaw Anthropic-`cache_control`-Markierungen in System-/Entwickler-Prompt-Blöcke ein, jedoch nur, wenn die Anfrage weiterhin an eine verifizierte OpenRouter-Route gerichtet ist (`openrouter` an ihrem Standardendpunkt oder ein beliebiger Provider bzw. eine beliebige Basis-URL, die zu `openrouter.ai` aufgelöst wird). Wird das Modell auf eine beliebige OpenAI-kompatible Proxy-URL umgeleitet, werden diese Markierungen nicht mehr eingefügt.

`contextPruning.mode: "cache-ttl"` ist für die Modellreferenzen `openrouter/anthropic/*`, `openrouter/deepseek/*`, `openrouter/moonshot/*`, `openrouter/moonshotai/*` und `openrouter/zai/*` zulässig, da diese Routen das providerseitige Prompt-Caching ohne von OpenClaw eingefügte Markierungen verarbeiten.

Quelle: `extensions/openrouter/index.ts` (`OPENROUTER_CACHE_TTL_MODEL_PREFIXES`).

Der Aufbau des DeepSeek-Caches auf OpenRouter erfolgt nach bestem Bemühen und kann einige Sekunden dauern; eine unmittelbar folgende Anfrage kann weiterhin `cached_tokens: 0` anzeigen. Überprüfen Sie dies nach einer kurzen Verzögerung mit einer wiederholten Anfrage mit demselben Präfix und verwenden Sie `usage.prompt_tokens_details.cached_tokens` als Signal für einen Cache-Treffer.

### Google Gemini (direkte API)

- Der direkte Gemini-Transport (`api: "google-generative-ai"`) meldet Cache-Treffer über das Upstream-Feld `cachedContentTokenCount`, das auf `cacheRead` abgebildet wird.
- Unterstützte Modellfamilien: `gemini-2.5*` und `gemini-3*` (Live-/Vorschauvarianten außerhalb dieses Präfixabgleichs sind ausgeschlossen, beispielsweise `gemini-live-2.5-flash-preview`).
- Wenn `cacheRetention` für ein unterstütztes Modell festgelegt ist, erstellt, verwendet und aktualisiert OpenClaw automatisch eine `cachedContents`-Ressource für den System-Prompt – ein manuelles Handle für zwischengespeicherte Inhalte ist nicht erforderlich. Die TTL beträgt `300s` für `cacheRetention: "short"` und `3600s` für `"long"`.
- Ein bereits vorhandenes Gemini-Handle für zwischengespeicherte Inhalte kann weiterhin als `params.cachedContent` (oder als veraltetes `params.cached_content`) übergeben werden; bei einem expliziten Handle wird der automatische Cache-Verwaltungspfad vollständig übersprungen.
- Dies ist vom Prompt-Präfix-Caching von Anthropic/OpenAI getrennt: OpenClaw verwaltet für Gemini eine providernative `cachedContents`-Ressource, statt Inline-Cache-Markierungen einzufügen.

Quelle: `src/agents/embedded-agent-runner/google-prompt-cache.ts`.

### CLI-Harness-Provider (Claude Code, Gemini CLI)

CLI-Backends, die JSONL-Nutzungsereignisse (`jsonlDialect: "claude-stream-json"` oder `"gemini-stream-json"`) ausgeben, durchlaufen einen gemeinsamen Nutzungsparser, der mehrere Varianten von Feldnamen erkennt, darunter einen einfachen `cached`-Zähler, der auf `cacheRead` abgebildet wird. Wenn die JSON-Nutzdaten der CLI kein direktes Feld für Eingabe-Token enthalten, leitet OpenClaw es als `input_tokens - cached` ab. Dies dient ausschließlich der Nutzungsnormalisierung – es werden keine Prompt-Cache-Markierungen im Stil von Anthropic/OpenAI für diese CLI-gesteuerten Modelle erstellt.

Quelle: `src/agents/cli-output.ts` (`toCliUsage`).

### Andere Provider

Wenn ein Provider keinen der oben genannten Cache-Modi unterstützt, hat `cacheRetention` keine Wirkung.

## Cache-Grenze des System-Prompts

OpenClaw teilt den System-Prompt an einer internen Cache-Präfixgrenze in ein **stabiles Präfix** und ein **veränderliches Suffix**. Inhalte oberhalb der Grenze (Tool-Definitionen, Skills-Metadaten, Workspace-Dateien) werden so angeordnet, dass sie über mehrere Durchläufe hinweg byteidentisch bleiben. Inhalte unterhalb der Grenze (beispielsweise `HEARTBEAT.md`, Laufzeit-Zeitstempel und andere Metadaten pro Durchlauf) können sich ändern, ohne das zwischengespeicherte Präfix ungültig zu machen.

Wesentliche Designentscheidungen:

- Stabile Projektkontextdateien des Workspace werden vor `HEARTBEAT.md` angeordnet, damit Heartbeat-Änderungen das stabile Präfix nicht ungültig machen.
- Die Grenze gilt für die Transportaufbereitung der Anthropic-, OpenAI- und Google-Familien sowie der CLI, sodass alle unterstützten Provider von derselben Präfixstabilität profitieren.
- Codex-Responses- und Anthropic-Vertex-Anfragen werden durch eine grenzbewusste Cache-Aufbereitung geleitet, sodass die Cache-Wiederverwendung mit den tatsächlich bei den Providern eingehenden Daten übereinstimmt.
- Fingerabdrücke von System-Prompts werden normalisiert (Leerraum, Zeilenenden, durch Hooks hinzugefügter Kontext und Reihenfolge der Laufzeitfähigkeiten), sodass semantisch unveränderte Prompts über mehrere Durchläufe hinweg denselben Cache verwenden.

Wenn nach einer Konfigurations- oder Workspace-Änderung unerwartete Spitzen bei `cacheWrite` auftreten, prüfen Sie, ob die Änderung oberhalb oder unterhalb der Cache-Grenze liegt. Das Verschieben veränderlicher Inhalte unter die Grenze (oder deren Stabilisierung) behebt das Problem in der Regel.

## OpenClaw-Schutzmechanismen für Cache-Stabilität

- Gebündelte MCP-Tool-Kataloge werden vor der Tool-Registrierung deterministisch sortiert (zuerst nach Servername, dann nach Tool-Name), sodass Änderungen an der Reihenfolge von `listTools()` den Tool-Block nicht verändern und die Präfixe des Prompt-Caches nicht ungültig machen.
- Bei älteren Sitzungen mit gespeicherten Bildblöcken bleiben die **3 neuesten abgeschlossenen Durchläufe** vollständig erhalten (gezählt werden alle abgeschlossenen Durchläufe, nicht nur solche mit Bildern). Ältere, bereits verarbeitete Bildblöcke werden durch eine Textmarkierung ersetzt, damit bildintensive Folgeanfragen nicht wiederholt große, veraltete Nutzdaten senden.

## Abstimmungsmuster

### Gemischter Datenverkehr (empfohlener Standardwert)

Behalten Sie eine langlebige Basiskonfiguration für Ihren Haupt-Agent bei und deaktivieren Sie das Caching für Agents mit stoßweisem Benachrichtigungsverkehr:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### Kostenorientierte Basiskonfiguration

- Legen Sie `cacheRetention: "short"` als Basiskonfiguration fest.
- Aktivieren Sie `contextPruning.mode: "cache-ttl"`.
- Halten Sie den Heartbeat nur bei Agents, die von warmen Caches profitieren, unterhalb Ihrer TTL.

## Live-Regressionstests

OpenClaw führt ein kombiniertes Live-Regressionstor für Caches aus, das wiederholte Präfixe, Tool-Durchläufe, Bilddurchläufe, Tool-Transkripte im MCP-Stil und eine Anthropic-Kontrolle ohne Cache abdeckt.

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-runner.ts`
- `src/agents/live-cache-regression-baseline.ts`

Führen Sie es wie folgt aus:

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

Die Baseline-Datei speichert die zuletzt beobachteten Live-Werte sowie die providerspezifischen Regressionsuntergrenzen, gegen die der Test prüft. Jeder Durchlauf verwendet neue sitzungsspezifische IDs und Prompt-Namespaces, damit ein vorheriger Cache-Zustand die aktuelle Stichprobe nicht verfälscht. Anthropic und OpenAI verwenden unterschiedliche Durchsetzungsregeln: Wird eine Anthropic-Untergrenze unterschritten, gilt dies als harte Regression (der Test schlägt fehl), während das Unterschreiten einer OpenAI-Untergrenze nur überwacht wird (als Warnung protokolliert, ohne dass der Durchlauf fehlschlägt). Sie verwenden keinen gemeinsamen providerübergreifenden Schwellenwert.

### Live-Erwartungen für Anthropic

- Explizite Warmup-Schreibvorgänge über `cacheWrite` werden erwartet.
- Bei wiederholten Durchläufen wird eine nahezu vollständige Wiederverwendung des Verlaufs erwartet, da die Cache-Steuerung von Anthropic den Cache-Haltepunkt im Verlauf der Unterhaltung verschiebt.
- Baseline-Untergrenzen für stabile, Tool-, Bild- und MCP-artige Abläufe sind harte Regressionsprüfungen.

### Live-Erwartungen für OpenAI

- Es wird nur `cacheRead` erwartet; `cacheWrite` bleibt bei Chat Completions `0`.
- Die Cache-Wiederverwendung bei wiederholten Durchläufen ist als providerspezifisches Plateau zu behandeln, nicht als Anthropic-artige fortschreitende Wiederverwendung des vollständigen Verlaufs.
- Untergrenzen dienen nur der Überwachung (eine Unterschreitung wird als Warnung protokolliert und führt nicht zum Fehlschlagen des Tests) und wurden aus dem beobachteten Live-Verhalten bei `gpt-5.4-mini` abgeleitet:

| Szenario                  | Untergrenze für `cacheRead` | Untergrenze für Trefferquote |
| ------------------------- | ---------------------------------: | ---------------------------: |
| Stabiles Präfix           |                              4,608 |                         0.90 |
| Tool-Transkript           |                              4,096 |                         0.85 |
| Bildtranskript            |                              3,840 |                         0.82 |
| MCP-artiges Transkript    |                              4,096 |                         0.85 |

Die zuletzt beobachteten Baseline-Werte (aus `live-cache-regression-baseline.ts`) lagen bei: stabiles Präfix `cacheRead=4864`, Trefferquote `0.966`; Tool-Transkript `cacheRead=4608`, Trefferquote `0.896`; Bildtranskript `cacheRead=4864`, Trefferquote `0.954`; MCP-artiges Transkript `cacheRead=4608`, Trefferquote `0.891`.

Warum sich die Prüfungen unterscheiden: Anthropic stellt explizite Cache-Haltepunkte und eine fortschreitende Wiederverwendung des Unterhaltungsverlaufs bereit, während das effektiv wiederverwendbare Präfix von OpenAI im Live-Datenverkehr bereits vor dem vollständigen Prompt ein Plateau erreichen kann. Der Vergleich beider Provider anhand eines einzigen providerübergreifenden prozentualen Schwellenwerts erzeugt falsche Regressionen.

## `diagnostics.cacheTrace`-Konfiguration

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # optional
    includeMessages: false # standardmäßig true
    includePrompt: false # standardmäßig true
    includeSystem: false # standardmäßig true
```

Standardwerte:

| Schlüssel         | Standardwert                                  |
| ----------------- | -------------------------------------------- |
| `filePath`        | `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl` |
| `includeMessages` | `true`                                       |
| `includePrompt`   | `true`                                       |
| `includeSystem`   | `true`                                       |

### Umgebungsvariablen (einmalige Fehlerdiagnose)

| Variable                             | Wirkung                                      |
| ------------------------------------ | -------------------------------------------- |
| `OPENCLAW_CACHE_TRACE=1`             | Aktiviert die Cache-Ablaufverfolgung         |
| `OPENCLAW_CACHE_TRACE_FILE=path`     | Überschreibt den Ausgabepfad                  |
| `OPENCLAW_CACHE_TRACE_MESSAGES=0\|1` | Schaltet die Erfassung vollständiger Nachrichten-Payloads um |
| `OPENCLAW_CACHE_TRACE_PROMPT=0\|1`   | Schaltet die Erfassung des Prompt-Texts um    |
| `OPENCLAW_CACHE_TRACE_SYSTEM=0\|1`   | Schaltet die Erfassung des System-Prompts um  |

### Zu prüfende Informationen

- Cache-Ablaufverfolgungsereignisse liegen im JSONL-Format mit stufenweisen Snapshots wie `session:loaded`, `prompt:before`, `stream:context` und `session:after` vor.
- Die Auswirkung des Caches auf die Token pro Durchlauf ist in den normalen Nutzungsoberflächen sichtbar: `cacheRead` und `cacheWrite` erscheinen in `/usage tokens`, `/status`, Sitzungsnutzungszusammenfassungen und benutzerdefinierten `messages.usageTemplate`-Layouts.
- Bei Anthropic werden bei aktivem Caching sowohl `cacheRead` als auch `cacheWrite` erwartet.
- Bei OpenAI wird bei Cache-Treffern `cacheRead` erwartet; `cacheWrite` wird nur bei Payloads der Responses API ausgefüllt, die diesen Wert enthalten (siehe [OpenAI](#openai-direct-api) weiter oben).
- OpenAI gibt außerdem Header für Ablaufverfolgung und Ratenbegrenzung wie `x-request-id`, `openai-processing-ms` und `x-ratelimit-*` zurück; verwenden Sie diese zur Anfrageverfolgung. Die Erfassung von Cache-Treffern sollte jedoch weiterhin aus dem Nutzungs-Payload und nicht aus Headern stammen.

## Schnelle Fehlerbehebung

- **Hoher Wert für `cacheWrite` bei den meisten Durchläufen**: Prüfen Sie auf veränderliche Eingaben im System-Prompt und stellen Sie sicher, dass das Modell beziehungsweise der Provider Ihre Cache-Einstellungen unterstützt.
- **Hoher Wert für `cacheWrite` bei Anthropic**: Dies bedeutet häufig, dass der Cache-Haltepunkt auf Inhalten liegt, die sich bei jeder Anfrage ändern.
- **Niedriger OpenAI-Wert für `cacheRead`**: Stellen Sie sicher, dass das stabile Präfix am Anfang steht, das wiederholte Präfix mindestens 1024 Token umfasst und derselbe Wert für `prompt_cache_key` bei Durchläufen wiederverwendet wird, die sich einen Cache teilen sollen.
- **Keine Wirkung von `cacheRetention`**: Stellen Sie sicher, dass der Modellschlüssel mit `agents.defaults.models["provider/model"]` übereinstimmt.
- **Bedrock-Nova-Anfragen mit Cache-Einstellungen**: Erwartetes Verhalten – diese werden zur Laufzeit ohne Cache-Aufbewahrung aufgelöst.

Zugehörige Dokumentation:

- [Anthropic](/de/providers/anthropic)
- [Token-Nutzung und Kosten](/de/reference/token-use)
- [Sitzungsbereinigung](/de/concepts/session-pruning)
- [Referenz zur Gateway-Konfiguration](/de/gateway/configuration-reference)

## Verwandte Themen

- [Token-Nutzung und Kosten](/de/reference/token-use)
- [API-Nutzung und Kosten](/de/reference/api-usage-costs)
