---
read_when:
    - Sie müssen wissen, welche Umgebungsvariablen in welcher Reihenfolge geladen werden.
    - Sie beheben fehlende API-Schlüssel im Gateway.
    - Sie dokumentieren die Provider-Authentifizierung oder Bereitstellungsumgebungen
summary: Woher OpenClaw Umgebungsvariablen lädt und welche Rangfolge gilt
title: Umgebungsvariablen
x-i18n:
    generated_at: "2026-07-26T18:25:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw bezieht Umgebungsvariablen aus mehreren Quellen. Dabei gilt die Regel: **Vorhandene Werte niemals überschreiben**.
Workspace-`.env`-Dateien sind eine weniger vertrauenswürdige Quelle: OpenClaw ignoriert Provider-Zugangsdaten und geschützte Laufzeitsteuerungen aus Workspace-`.env`, bevor die Rangfolge angewendet wird.

## Rangfolge (höchste bis niedrigste)

1. **Prozessumgebung** (was der Gateway-Prozess bereits von der übergeordneten Shell bzw. dem übergeordneten Daemon übernommen hat).
2. **`.env` im aktuellen Arbeitsverzeichnis** (dotenv-Standard; überschreibt nicht; Provider-Zugangsdaten und geschützte Laufzeitsteuerungen werden ignoriert).
3. **Globale `.env`** unter `~/.openclaw/.env` (auch `$OPENCLAW_STATE_DIR/.env` genannt; für Provider-API-Schlüssel empfohlen; überschreibt nicht).
4. **Konfigurationsblock `env`** in `~/.openclaw/openclaw.json` (wird nur angewendet, wenn der Wert fehlt).
5. **Optionaler Import aus der Anmelde-Shell** (`env.shellEnv.enabled` oder `OPENCLAW_LOAD_SHELL_ENV=1`), wird nur auf fehlende erwartete Schlüssel angewendet.

Bei neuen Ubuntu-Installationen, die das standardmäßige Zustandsverzeichnis verwenden, behandelt OpenClaw außerdem `~/.config/openclaw/gateway.env` als Kompatibilitäts-Fallback nach der globalen `.env`. Wenn beide Dateien vorhanden sind und sich widersprechen, behält OpenClaw `~/.openclaw/.env` bei und gibt eine Warnung aus.

Wenn die Konfigurationsdatei vollständig fehlt, wird Schritt 4 übersprungen; der Shell-Import wird weiterhin ausgeführt, sofern er aktiviert ist.

## Unterstützte Variablen für den Betrieb

Die folgenden Variablen bilden den unterstützten Umgebungsvertrag für den Betrieb. Nicht dokumentierte `OPENCLAW_*`-Variablen sind interne Implementierungsdetails und können ohne Vorankündigung entfallen.

### Pfade und Instanzen

| Variable                 | Zweck                                                              |
| ------------------------ | ------------------------------------------------------------------ |
| `OPENCLAW_HOME`          | Überschreibt das für OpenClaw-Pfadstandards verwendete Home-Verzeichnis. |
| `OPENCLAW_STATE_DIR`     | Überschreibt das veränderliche Zustandsverzeichnis.                |
| `OPENCLAW_CONFIG_PATH`   | Überschreibt den Pfad der aktiven Konfigurationsdatei.             |
| `OPENCLAW_WORKSPACE_DIR` | Überschreibt den standardmäßigen Agent-Workspace.                  |
| `OPENCLAW_PROFILE`       | Wählt ein benanntes Profil und dessen isolierte Standardwerte aus. |
| `OPENCLAW_GIT_DIR`       | Überschreibt den Quellcode-Checkout für Aktualisierungen des Entwicklungskanals. |
| `OPENCLAW_INCLUDE_ROOTS` | Ermöglicht die Auflösung von `$include` aus zusätzlichen Stammverzeichnissen. |

### Gateway und Authentifizierung

| Variable                    | Zweck                                                             |
| --------------------------- | ----------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | Überschreibt die von Clients verwendete Remote-Gateway-URL.       |
| `OPENCLAW_GATEWAY_PORT`     | Überschreibt den lokalen Gateway-Port.                             |
| `OPENCLAW_GATEWAY_TOKEN`    | Stellt Token-Authentifizierung für Gateway-Server und -Clients bereit. |
| `OPENCLAW_GATEWAY_PASSWORD` | Stellt Passwortauthentifizierung für Gateway-Server und -Clients bereit. |

### Provider-Zugangsdaten

Core und gebündelte Provider-Plugins erkennen die folgenden Variablen für Zugangsdaten und die Provider-Auswahl. Verwenden Sie vorzugsweise die Konfigurations- oder SecretRef-Felder des jeweiligen Providers, wenn Sie abgegrenzte Zugangsdaten statt eines einzigen prozessweiten Werts benötigen.

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY` und `Z_AI_API_KEY`.

Installierte Drittanbieter-Plugins können in ihren Plugin-Manifesten zusätzliche Zugangsdatenvariablen deklarieren; diese Variablen sind Verträge des Plugins, das sie deklariert, und keine OpenClaw-Core-Variablen.

### Protokollierung und Diagnose

| Variable                             | Zweck                                                          |
| ------------------------------------ | -------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | Überschreibt die Protokollstufen für Datei und Konsole.        |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | Aktiviert Zeitdiagnosen für den Modelltransport.               |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | Wählt Diagnosen mit redigierten Modellnutzdaten aus.           |
| `OPENCLAW_DEBUG_SSE`                 | Wählt SSE-Zeitdiagnosen oder Diagnosen zur Ereignisvorschau aus. |
| `OPENCLAW_DEBUG_CODE_MODE`           | Aktiviert Oberflächendiagnosen für den Codemodus.              |
| `OPENCLAW_DIAGNOSTICS`               | Aktiviert benannte Diagnose-Flags oder deaktiviert mit `0` alle Flags. |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | Wählt den JSONL-Pfad für Zeitverlaufsdiagnosen aus.             |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | Fügt den Zeitverlaufsdiagnosen Ereignisschleifen-Stichproben hinzu. |

### Funktions- und Laufzeitumschalter

| Variable                             | Zweck                                                                         |
| ------------------------------------ | ----------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | Importiert fehlende erwartete Variablen aus der Anmelde-Shell.                |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | Legt das Zeitlimit für den Import aus der Anmelde-Shell fest.                 |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | Deaktiviert Exec-Shell-Snapshots mit `0`.                      |
| `OPENCLAW_OFFLINE`                   | Verhindert Downloads angehefteter Agent-Hilfsbinärdateien.                    |
| `OPENCLAW_BROWSER_HEADLESS`          | Erzwingt verwaltete Browserstarts mit Benutzeroberfläche (`0`) oder ohne Benutzeroberfläche (`1`). |
| `OPENCLAW_DISABLE_BONJOUR`           | Erzwingt die Bonjour-Ankündigung als aktiviert (`0`) oder deaktiviert (`1`). |
| `OPENCLAW_NO_AUTO_UPDATE`            | Deaktiviert die automatische Anwendung von Aktualisierungen.                 |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | Erlaubt als Notfallausnahme vertrauenswürdige Private-DNS-Verbindungen zu `ws://`. |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | Erlaubt mehrere Gateway-Prozesse unter Beibehaltung zustandsspezifischer Eigentümersperren. |
| `OPENCLAW_SKIP_CHANNELS`             | Startet das Gateway zur Fehlerbehebung ohne Kanaltransporte.                  |
| `OPENCLAW_THEME`                     | Erzwingt für die TUI-Palette `light` oder `dark`.      |

## Provider-Zugangsdaten und Workspace-`.env`

Speichern Sie Provider-API-Schlüssel nicht ausschließlich in einer Workspace-`.env`. OpenClaw blockiert eine große Menge von Schlüsseln für Provider-Zugangsdaten und Endpunktumleitungen aus Workspace-`.env`-Dateien, darunter jede bekannte Umgebungsvariable für die Provider-Authentifizierung (zum Beispiel `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `XAI_API_KEY`, `MISTRAL_API_KEY`, `GROQ_API_KEY`, `DEEPSEEK_API_KEY`, `PERPLEXITY_API_KEY`, `BRAVE_API_KEY`, `TAVILY_API_KEY`, `EXA_API_KEY`, `FIRECRAWL_API_KEY`), außerdem alle Schlüssel, die auf `_API_HOST`, `_BASE_URL`, `_ENDPOINT` oder `_HOMESERVER` enden, sowie die gesamten Namensräume `OPENCLAW_*`, `CLAWHUB_*`, `ANTHROPIC_API_KEY_*` und `OPENAI_API_KEY_*`.

Verwenden Sie stattdessen eine dieser vertrauenswürdigen Quellen für Provider-Zugangsdaten:

- Die Umgebung des Gateway-Prozesses, etwa eine Shell, eine launchd-/systemd-Unit, ein Container-Secret oder ein CI-Secret.
- Die globale dotenv-Laufzeitdatei unter `~/.openclaw/.env` oder `$OPENCLAW_STATE_DIR/.env`.
- Den Konfigurationsblock `env` in `~/.openclaw/openclaw.json`.
- Den optionalen Import aus der Anmelde-Shell, wenn `env.shellEnv.enabled` oder `OPENCLAW_LOAD_SHELL_ENV=1` aktiviert ist.

Wenn Sie Provider-Schlüssel oder Werte für die Endpunktweiterleitung zuvor ausschließlich in einer Workspace-`.env` gespeichert haben, verschieben Sie sie in eine der oben genannten vertrauenswürdigen Quellen. Workspace-`.env` kann weiterhin gewöhnliche Projektvariablen bereitstellen, bei denen es sich nicht um Zugangsdaten, Endpunktumleitungen, Host-Überschreibungen oder `OPENCLAW_*`-Laufzeitsteuerungen handelt.

Die Sicherheitsbegründung finden Sie unter [Workspace-`.env`-Dateien](/de/gateway/security#workspace-env-files).

## Konfigurationsblock `env`

Es gibt zwei gleichwertige Möglichkeiten, Inline-Umgebungsvariablen festzulegen (beide überschreiben keine vorhandenen Werte):

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

Der Konfigurationsblock `env` akzeptiert ausschließlich literale Zeichenfolgenwerte. Er expandiert keine
`file:...`-Werte; beispielsweise wird `XAI_API_KEY: "file:secrets/xai-api-key.txt"`
als exakt diese Zeichenfolge an Provider übergeben.

Verwenden Sie für dateibasierte Provider-Schlüssel eine SecretRef im Zugangsdatenfeld, das
sie unterstützt:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

Informationen zu unterstützten Feldern finden Sie unter [Secret-Verwaltung](/de/gateway/secrets) und der
[SecretRef-Zugangsdatenoberfläche](/de/reference/secretref-credential-surface).

## Import der Shell-Umgebung

`env.shellEnv` führt Ihre Anmelde-Shell aus und importiert nur **fehlende** erwartete Schlüssel:

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

Entsprechende Umgebungsvariablen:

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000` (Standardwert `15000`)

## Exec-Shell-Snapshots

Auf Gateway-Hosts ohne Windows verwenden bash- und zsh-`exec`-Befehle standardmäßig einen Start-Snapshot.
Setzen Sie `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` in der Umgebung des Gateway-Prozesses, um diesen Pfad zu deaktivieren.
Die Werte `false`, `no` und `off` deaktivieren ihn ebenfalls. Pro Aufruf gesetzte `exec.env`-Werte können
Snapshots weder umschalten noch den Snapshot-Cache umleiten.

## Zur Laufzeit injizierte Umgebungsvariablen

OpenClaw injiziert außerdem Kontextmarkierungen in gestartete untergeordnete Prozesse:

- `OPENCLAW_SHELL=exec`: wird für Befehle gesetzt, die über das Tool `exec` ausgeführt werden.
- `OPENCLAW_SHELL=acp-client`: wird für `openclaw acp client` gesetzt, wenn es den ACP-Bridge-Prozess startet.
- `OPENCLAW_SHELL=tui-local`: wird für lokale TUI-Shell-Befehle von `!` gesetzt.
- `OPENCLAW_CLI=1`: wird für untergeordnete Prozesse gesetzt, die vom CLI-Einstiegspunkt gestartet werden.

Dies sind Laufzeitmarkierungen (keine erforderliche Benutzerkonfiguration). Sie können in der Shell-/Profillogik verwendet werden,
um kontextspezifische Regeln anzuwenden.

## UI-Umgebungsvariablen

- `OPENCLAW_THEME=light`: erzwingt die helle TUI-Palette, wenn Ihr Terminal einen hellen Hintergrund hat.
- `OPENCLAW_THEME=dark`: erzwingt die dunkle TUI-Palette.
- `COLORFGBG`: wenn Ihr Terminal diese Variable exportiert, verwendet OpenClaw den Hinweis zur Hintergrundfarbe, um die TUI-Palette automatisch auszuwählen.

## Ersetzung von Umgebungsvariablen in der Konfiguration

Sie können mit der Syntax `${VAR_NAME}` direkt auf Umgebungsvariablen in Zeichenfolgenwerten der Konfiguration verweisen:

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

Ausführliche Informationen finden Sie unter [Konfiguration: Ersetzung von Umgebungsvariablen](/de/gateway/configuration-reference#env-var-substitution).

## Secret-Referenzen im Vergleich zu `${ENV}`-Zeichenfolgen

OpenClaw unterstützt zwei umgebungsgesteuerte Muster:

- Ersetzung von `${VAR}`-Zeichenfolgen in Konfigurationswerten.
- SecretRef-Objekte (`{ source: "env", provider: "default", id: "VAR" }`) für Felder, die Secret-Referenzen unterstützen.

Beide werden zum Aktivierungszeitpunkt aus der Prozessumgebung aufgelöst. Details zu SecretRef sind unter [Secret-Verwaltung](/de/gateway/secrets) dokumentiert.
Der Konfigurationsblock `env` selbst löst weder SecretRefs noch
Kurzschreibweisen mit `file:...` auf.

## Pfadbezogene Umgebungsvariablen

| Variable                 | Zweck                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | Überschreibt das Home-Verzeichnis, das für interne OpenClaw-Pfadstandardwerte verwendet wird (`~/.openclaw/`, Agent-Verzeichnisse, Sitzungen, Anmeldedaten, Installer-Onboarding und der standardmäßige Entwicklungs-Checkout). Nützlich, wenn OpenClaw unter einem dedizierten Dienstbenutzer ausgeführt wird. |
| `OPENCLAW_STATE_DIR`     | Überschreibt das Zustandsverzeichnis (Standard: `~/.openclaw`).                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | Überschreibt den Pfad der Konfigurationsdatei (Standard: `~/.openclaw/openclaw.json`).                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | Pfadliste mit Verzeichnissen, in denen `$include`-Direktiven Dateien außerhalb des Konfigurationsverzeichnisses auflösen dürfen (Standard: keine – `$include` ist auf das Konfigurationsverzeichnis beschränkt). Tilden werden expandiert.                                                         |

## Downloads von Agent-Hilfswerkzeugen

Setzen Sie `OPENCLAW_OFFLINE=1`, um zu verhindern, dass OpenClaw seine festgelegten Hilfsbinärdateien
`fd` und `ripgrep` herunterlädt. Vorhandene Hilfsprogramme im
OpenClaw-Werkzeugverzeichnis und funktionsfähige Systembinärdateien können weiterhin verwendet werden; ein fehlendes Hilfsprogramm bleibt
nicht verfügbar, anstatt eine Netzwerkanfrage auszulösen.

## Protokollierung

| Variable                         | Zweck                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | Überschreibt die Protokollstufe für Datei und Konsole (z. B. `debug`, `trace`). Hat Vorrang vor `logging.level` und `logging.consoleLevel` in der Konfiguration. Ungültige Werte werden mit einer Warnung ignoriert. |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | Gibt gezielte Zeitdiagnosen für Modellanfragen und -antworten auf der Stufe `info` aus, ohne globale Debug-Protokolle zu aktivieren.                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | Modellnutzlastdiagnose: `summary`, `tools` oder `full-redacted`. `full-redacted` ist begrenzt und bereinigt, kann jedoch Prompt-/Nachrichtentext enthalten.                                               |
| `OPENCLAW_DEBUG_SSE`             | Streaming-Diagnose: `events` für die Zeitmessung des ersten und abgeschlossenen Ereignisses, `peek`, um die ersten fünf bereinigten SSE-Ereignisse einzuschließen.                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | Diagnose der Modelloberfläche im Codemodus, einschließlich des Ausblendens von Provider-Tools und der kompakten direkten Durchsetzung von Steueranweisungen.                                                                                  |

### `OPENCLAW_HOME`

Wenn gesetzt, ersetzt `OPENCLAW_HOME` das Home-Verzeichnis des Systems (`$HOME` / `os.homedir()`) für interne OpenClaw-Pfadstandardwerte. Dazu gehören das standardmäßige Zustandsverzeichnis, der Konfigurationspfad, Agent-Verzeichnisse, Anmeldedaten, der Onboarding-Arbeitsbereich des Installers und der von `openclaw update --channel dev` verwendete standardmäßige Entwicklungs-Checkout.

**Rangfolge:** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > Termux-Home-Fallback `PREFIX` unter Android > `os.homedir()`

**Beispiel** (macOS LaunchDaemon):

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` kann auch auf einen Pfad mit Tilde gesetzt werden (z. B. `~/svc`), der vor der Verwendung anhand derselben Fallback-Kette für das Betriebssystem-Home-Verzeichnis expandiert wird.

Explizite Pfadvariablen wie `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH` und `OPENCLAW_GIT_DIR` haben weiterhin Vorrang. Aufgaben im Zusammenhang mit dem Betriebssystemkonto, etwa das Erkennen von Shell-Startdateien, die Einrichtung des Paketmanagers und die hostseitige Expansion von `~`, können weiterhin das tatsächliche System-Home-Verzeichnis verwenden.

## nvm-Benutzer: TLS-Fehler bei web_fetch

Wenn Node.js über **nvm** (nicht über den Systempaketmanager) installiert wurde, verwendet das integrierte `fetch()`
den mit nvm gebündelten CA-Speicher, in dem möglicherweise moderne Stammzertifizierungsstellen fehlen (ISRG Root X1/X2 für Let's Encrypt,
DigiCert Global Root G2 usw.). Dies führt dazu, dass `web_fetch` auf den meisten HTTPS-Websites mit `"fetch failed"` fehlschlägt.

Unter Linux erkennt OpenClaw nvm automatisch und wendet die Korrektur in der tatsächlichen Startumgebung an:

- `openclaw gateway install` schreibt `NODE_EXTRA_CA_CERTS` in die Umgebung des systemd-Dienstes
- der CLI-Einstiegspunkt `openclaw` startet sich selbst erneut, wobei `NODE_EXTRA_CA_CERTS` vor dem Start von Node gesetzt wird

**Manuelle Korrektur (für ältere Versionen oder direkte Starts von `node ...`):**

Exportieren Sie die Variable, bevor Sie OpenClaw starten:

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

Verlassen Sie sich bei dieser Variable nicht darauf, sie nur in `~/.openclaw/.env` zu schreiben; Node liest
`NODE_EXTRA_CA_CERTS` beim Prozessstart.

## Veraltete Umgebungsvariablen

OpenClaw liest nur `OPENCLAW_*`-Umgebungsvariablen. Die veralteten Präfixe
`CLAWDBOT_*` und `MOLTBOT_*` aus früheren Versionen werden stillschweigend
ignoriert.

Falls beim Start noch welche im Gateway-Prozess gesetzt sind, gibt OpenClaw eine
einmalige Node-Veraltungswarnung (`OPENCLAW_LEGACY_ENV_VARS`) aus, die die
erkannten Präfixe und die Gesamtanzahl auflistet. Benennen Sie jeden Wert um, indem Sie das
veraltete Präfix durch `OPENCLAW_` ersetzen (beispielsweise `CLAWDBOT_GATEWAY_TOKEN` durch
`OPENCLAW_GATEWAY_TOKEN`); die alten Namen haben keine Wirkung.

## Verwandte Themen

- [Gateway-Konfiguration](/de/gateway/configuration)
- [FAQ: Umgebungsvariablen und Laden von .env](/de/help/faq#env-vars-and-env-loading)
- [Modellübersicht](/de/concepts/models)
