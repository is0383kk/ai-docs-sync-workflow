---
read_when:
    - Sie möchten die Inferenz einrichten und anschließend die Einrichtung mit OpenClaw abschließen
summary: CLI-Referenz für `openclaw onboard` (interaktives Onboarding)
title: Einrichtung
x-i18n:
    generated_at: "2026-07-26T17:42:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ec5cfc564aa14041d1aa67a978a4661e6105b7119a942940f71197c695e788b
    source_path: cli/onboard.md
    workflow: 16
---

# `openclaw onboard`

Geführte Einrichtung, bei der Inferenz zuerst hergestellt wird: Sie erkennt vorhandenen KI-Zugriff,
erfordert eine erfolgreiche Live-Vervollständigung, speichert nur die funktionierende Route und startet dann
OpenClaw, um den Rest zu konfigurieren. `openclaw setup` ruft diesen Ablauf auf neuen
Systemen oder immer dann auf, wenn eine Onboarding-Option vorhanden ist, auf; konfigurierte Systeme verwenden
den bloßen Befehl `openclaw setup` für den Chat mit dem System-Agenten. `openclaw setup --baseline`
schreibt nur die Basiskonfiguration und den Workspace.

<CardGroup cols={2}>
  <Card title="CLI-Onboarding-Zentrale" href="/de/start/wizard" icon="rocket">
    Anleitung für den interaktiven CLI-Ablauf.
  </Card>
  <Card title="Onboarding-Übersicht" href="/de/start/onboarding-overview" icon="map">
    Wie die Komponenten des OpenClaw-Onboardings zusammenspielen.
  </Card>
  <Card title="Referenz zur CLI-Einrichtung" href="/de/start/wizard-cli-reference" icon="book">
    Ausgaben, interne Abläufe und Verhalten der einzelnen Schritte.
  </Card>
  <Card title="CLI-Automatisierung" href="/de/start/wizard-cli-automation" icon="terminal">
    Nicht interaktive Flags und skriptgesteuerte Einrichtungen.
  </Card>
  <Card title="Onboarding der macOS-App" href="/de/start/onboarding" icon="apple">
    Onboarding-Ablauf für die macOS-Menüleisten-App.
  </Card>
</CardGroup>

## Beispiele

```bash
openclaw onboard
openclaw onboard --tui
openclaw onboard --classic
openclaw onboard --modern
openclaw onboard --flow quickstart
openclaw onboard --flow manual
openclaw onboard --flow import
openclaw onboard --import-from hermes --import-source ~/.hermes
openclaw onboard --skip-bootstrap
openclaw onboard recommendations --json
openclaw onboard recommendations acknowledge
openclaw onboard recommendations acknowledge --retry "<failed-id>"
openclaw onboard recommendations refresh
openclaw onboard --mode remote --remote-url wss://gateway-host:18789
```

`openclaw onboard recommendations` liest ausstehende Treffer für App-Empfehlungen,
die während des Onboardings gespeichert wurden. Fügen Sie `--json` hinzu, um die maschinenlesbare Liste abzurufen, die vom
Bootstrap beim ersten Start verwendet wird. Der Befehl durchsucht weder installierte Apps erneut noch ruft er ein
Modell auf. Seine Ausgabe enthält nur validierte Installations-IDs, Quelle und Stufe; sie
lässt nicht vertrauenswürdige Marktplatztexte, Modellbegründungen und lokale App-
Bezeichnungen absichtlich aus. Nachdem das Empfehlungsangebot beantwortet wurde, gibt der Befehl
eine leere Liste zurück, und bei zukünftigen Onboarding-Durchläufen wird der Schritt vollständig übersprungen.
`openclaw onboard recommendations refresh` löscht das gespeicherte Angebot, sodass beim nächsten
Onboarding-Durchlauf installierte Apps erneut durchsucht und ein neues Angebot erstellt werden.

Bei neuen Workspaces wird die Auswahl der Empfehlungen auf die Bootstrap-Unterhaltung verschoben.
Nachdem diese Unterhaltung die Auswahl der benutzenden Person verarbeitet hat,
markiert `openclaw onboard recommendations acknowledge` das gespeicherte Angebot als beantwortet.
Die Bestätigung ist idempotent. Wenn eine ausgewählte Installation fehlschlägt, übergeben Sie jede fehlgeschlagene
undurchsichtige ID mit `--retry <id...>`; erfolgreiche und abgelehnte Treffer werden übernommen,
während fehlgeschlagene Treffer für einen späteren Onboarding-Durchlauf ausstehend bleiben. Unbekannte IDs
führen zu einem Fehler, ohne das gespeicherte Angebot zu ändern. Nach einer unterbrochenen Installation eines ClawHub-Skills
gilt ein vorhandenes Ziel nur dann als erfolgreich, wenn
`openclaw skills verify "@owner/slug"` für dieselbe
durch den Herausgeber qualifizierte Empfehlungs-ID erfolgreich ausgeführt wird und seine JSON-Ausgabe
`openclaw.resolution.source: "installed"` meldet. Die Verifizierung der Registry allein ist kein
Nachweis einer lokalen Installation. Lassen Sie diese ID andernfalls mit `--retry` ausstehend und
überschreiben Sie den vorhandenen Skill nicht.

- `--classic`: öffnet den vollständigen Assistenten mit allen Einzelschritten. Diese Option kann nicht mit
  `--non-interactive` kombiniert werden; lassen Sie `--classic` für die automatisierte Einrichtung weg.
- `--flow quickstart`: öffnet den klassischen Assistenten mit minimalen Eingabeaufforderungen, verwendet
  standardmäßig Token-Authentifizierung und generiert ein Token, wenn keine gespeicherten oder expliziten
  Anmeldedaten verwendet werden können. Explizite Flags für das lokale Gateway wie
  `--gateway-port`, `--gateway-bind`, `--gateway-auth` und `--tailscale`
  überschreiben die entsprechenden gespeicherten oder standardmäßigen Schnellstartwerte; ausgelassene
  Optionen behalten ihre aktuellen Werte bei.
- `--flow manual` (Alias `advanced`): öffnet den klassischen Assistenten mit vollständigen Eingabeaufforderungen
  für Port, Bindung und Authentifizierung.
- `--flow import`: führt einen erkannten Migrations-Provider (zum Beispiel Hermes über `--import-from hermes`) für eine neue Einrichtung aus. Nach der Bestätigung stellt das Onboarding Konfiguration, Anmeldedaten, Workspace-Dateien, Memory und Skills unter privaten temporären Zielen bereit; die importierte Inferenz muss eine Live-Vervollständigung erfolgreich durchlaufen, bevor Workspace- und Agent-Zustand übernommen und die Konfiguration festgeschrieben werden. Bei einem Fehler oder Abbruch vor der Übernahme bleibt das Live-Ziel unverändert. Externe Aktivierungsschritte, die nicht rückgängig gemacht werden können, etwa die Installation des Codex-Plugins, werden anschließend ausgeführt und können über den Migrationsbericht erneut versucht werden. Setzen Sie zuerst Konfiguration, Anmeldedaten, Sitzungen und Workspace-Zustand zurück, falls diese vorhanden sind. Verwenden Sie [`openclaw migrate`](/de/cli/migrate) für Probelaufpläne, den Überschreibmodus, verifizierte Sicherungen, Berichte und exakte Zuordnungen.
- `--remote-url` und `--remote-token`: füllen den klassischen Schritt für ein entferntes Gateway vorab aus und überschreiben für diesen Durchlauf gespeicherte Remote-Werte. Beim Ändern der URL werden gespeicherte Anmeldedaten nicht wiederverwendet, sofern Sie nicht auch ein Token übergeben. Das Token bleibt in Eingabeaufforderungen maskiert und folgt der bestehenden Auswahl des Assistenten zur Speicherung als Klartext oder SecretRef.
- `--tailscale-reset-on-exit` und `--no-tailscale-reset-on-exit`: steuern ausdrücklich, ob die Konfiguration von Tailscale Serve oder Funnel beim Beenden des Gateways zurückgesetzt wird. Werden beide Optionen ausgelassen, bleibt die aktuelle Einstellung bei nicht interaktiven erneuten Ausführungen erhalten.
- `--modern` ist ein Kompatibilitätsalias für den dialogorientierten OpenClaw-Einrichtungsassistenten.
  Er verwendet dieselbe Live-Inferenz-Prüfung wie `openclaw setup` und
  akzeptiert nur `--workspace`, `--accept-risk`,
  `--non-interactive` und `--json`. Andere Einrichtungs-Flags werden abgelehnt, statt
  stillschweigend ignoriert zu werden.

## Geführter Ablauf

Der bloße Befehl `openclaw onboard` startet den geführten Ablauf. Er zeigt den Sicherheitshinweis an
und stellt anschließend zu Beginn eine Frage: **Vollzugriff** (empfohlen – die Einrichtung sucht
automatisch nach KI-Apps, Schlüsseln und lokalen Laufzeitumgebungen) oder **zuerst fragen** (die Einrichtung fragt
einmal, bevor sie die Umgebung durchsucht, oder ermöglicht Ihnen die manuelle Konfiguration). Die
Auswahl wird als `wizard.accessMode` gespeichert. Wenn die Erkennung erlaubt ist, erkennt das Onboarding
bereits verfügbaren KI-Zugriff über konfigurierte Modelle, API-Key-
Umgebungsvariablen und unterstützte lokale CLIs und testet anschließend den empfohlenen
Kandidaten mit einer echten Vervollständigung. Wenn ein Kandidat fehlschlägt, versucht das Onboarding unauffällig
den nächsten verwendbaren Kandidaten und fasst alle Kandidaten, die nicht geantwortet haben, in einer
einzigen Zeile zusammen; die funktionierende Route wird zusammen mit einer Option angekündigt, über einen einzigen Tastendruck
stattdessen alle anderen Optionen anzuzeigen.

Wenn die automatische Erkennung ausgeschöpft ist, zeigt die Provider-Auswahl zuerst OpenAI,
Anthropic, xAI (Grok), Google und OpenRouter an. Wählen Sie **Mehr…**, um alle
anderen unterstützten Provider nach Provider gruppiert anzuzeigen; Regionen, Tarife und Authentifizierungsmethoden
erscheinen anschließend in einem zweiten Menü. Unterstützte Browser- oder Geräteanmeldungen sowie maskierte
API-Key- oder Token-Methoden verwenden denselben Live-Vervollständigungspfad. OpenClaw speichert
erst nach erfolgreichem Test ausschließlich die verifizierte Modellroute und die zugehörigen Anmeldedaten; ein
fehlgeschlagener Kandidat ersetzt weder das konfigurierte Modell noch speichert er die versuchten
Anmeldedaten. Wählen Sie **Vorerst überspringen**, um den Vorgang zu beenden, ohne OpenClaw zu starten, und
führen Sie `openclaw onboard` erneut aus, wenn Sie bereit sind. Die Einrichtung von Workspace und Gateway bleibt
unverändert, bis OpenClaw startet.

Im geführten Modus stellt `--workspace <dir>` den von OpenClaw vorgeschlagenen Workspace
und den isolierten Inferenzkontext bereit. Diese werden erst gespeichert, nachdem Sie den
OpenClaw-Einrichtungsvorschlag genehmigt haben. Beim klassischen und nicht interaktiven Onboarding wird der
Workspace über den jeweiligen normalen Einrichtungsablauf gespeichert. Bei einer erneuten Ausführung mit einer vorhandenen Agent-
Liste bewahrt das Onboarding den konfigurierten Flotten-Workspace: Der klassische
Assistent zeigt beide Pfade an und erfordert vor dem Verschieben eine ausdrückliche Bestätigung,
während die nicht interaktive Einrichtung warnt und den aktuellen Wert beibehält.

Nach erfolgreicher Inferenz sucht das Onboarding nach Memories aus unterstützten lokalen KI-
Tools: Claude Code Auto-Memory, konsolidierte Codex-Memories und Hermes-Memory-
Dateien. Wenn welche gefunden werden, bietet eine Seite an, sie zur indizierten Wiedererkennung
unter `memory/imports/` in den Agent-Workspace zu kopieren. Ohne
Bestätigung wird nichts importiert, zuvor importierte Dateien werden übersprungen, und Sie können
später jederzeit über die [Seite zum Memory-Import](/de/web/control-ui) in der Control UI importieren, die
denselben ausschließlich auf Memory beschränkten Umfang bietet. (Eine vollständige Ausführung von [`openclaw migrate`](/de/cli/migrate) ist
umfangreicher: Sie kann auch Konfiguration, Skills und Anmeldedaten importieren.) Der klassische
Assistent zeigt dieselbe Seite an, nachdem er den Workspace vorbereitet hat.

Nach erfolgreicher Inferenz (und dem Angebot zum Memory-Import) wendet das geführte Onboarding
die Standardeinrichtung automatisch an – Workspace, Gateway und Sitzungen,
denselben Plan, den der dialogorientierte Chat `openclaw setup` bei „Ja“ anwenden würde –
und bietet anschließend Plugin- und Skill-Empfehlungen auf Grundlage installierter Apps an; App-Namen
werden über Ihr konfiguriertes Modell und die ClawHub-Suche abgeglichen, und der Schritt kann
mit [`wizard.appRecommendations`](/de/gateway/configuration-reference#wizard) deaktiviert werden.
In einer macOS-, Linux- oder Windows-Desktop-Sitzung öffnet er anschließend das authentifizierte
Dashboard der Control UI und wartet bis zu 60 Sekunden, bis der Browser-Client eine
Verbindung herstellt. Unter einem Linux-System ohne grafische Oberfläche oder über SSH gibt er eine auffällige, kopierbare
Dashboard-URL aus, einschließlich eines SSH-Portweiterleitungsbefehls für ein Loopback-Gateway,
und wartet bis zu fünf Minuten. Nach einer erfolgreichen Verbindung wird der Vorgang im Browser fortgesetzt;
ein nicht erreichbares Gateway oder eine Zeitüberschreitung führt zur selben Terminal-Ausweichoption wie
zuvor. Übergeben Sie `--tui`, um die Browser-Übergabe zu überspringen und diese Terminal-Ausweichoption zu erzwingen.
Wenn das Anwenden der Einrichtung fehlschlägt, wechselt das Onboarding zum dialogorientierten OpenClaw-
Chat, um die Einrichtung interaktiv abzuschließen. Kanäle, Agenten,
Plugins und andere optionale Funktionen bleiben dem OpenClaw-Chat vorbehalten: Führen Sie
`openclaw` aus und verwenden Sie `open channel wizard for <channel>`, um die Erfassung von Kanal-
Anmeldedaten an einen maskierten Terminal-Assistenten zu übergeben. Um den Modell-
Provider oder dessen Authentifizierung zu ändern, beenden Sie OpenClaw und führen Sie `openclaw onboard` aus;
OpenClaw öffnet weder die geführten noch die klassischen Provider-Abläufe.

Wird `openclaw onboard` bei einer konfigurierten Installation erneut ausgeführt, überprüft es zuerst das aktuelle
Standardmodell, sodass derselbe Ablauf als Überprüfungs- und Reparaturdurchlauf dient –
die Einrichtung wird weder erneut angewendet noch neu installiert, und der Gateway-Dienst wird nicht neu gestartet.
Wenn diese Prüfung fehlschlägt, wird das konfigurierte Modell niemals automatisch ersetzt –
das Onboarding hält an und fragt, wie fortgefahren werden soll. Die Prüfung wird außerhalb Ihres
Workspace ausgeführt, sodass ein von einem Workspace-Plugin bereitgestelltes Modell hier fehlschlagen kann, obwohl es
im Agenten weiterhin funktioniert.
Verwenden Sie `openclaw onboard --classic` für providerspezifische Authentifizierung, Kanäle, Skills,
die Einrichtung eines entfernten Gateways, Importe oder vollständige Gateway-Steuerungsmöglichkeiten. Führen Sie für die dialogorientierte
Einrichtung und Reparatur ohne Inferenz `openclaw setup` aus; `openclaw onboard
--modern` ist ein Kompatibilitätsalias über dieselbe Inferenzprüfung. Der klassische
Assistent kann das Standardmodell optional mit einer Live-Vervollständigung überprüfen, aber
OpenClaw startet erst, nachdem seine eigene Live-Inferenzprüfung erfolgreich war.

In einem interaktiven Terminal leitet der bloße Befehl `openclaw` (ohne Unterbefehl) abhängig vom Konfigurations-
zustand weiter:

- Wenn die aktive Konfigurationsdatei fehlt oder keine selbst festgelegten Einstellungen enthält (leer oder
  nur Metadaten), startet das geführte Onboarding.
- Wenn die Konfigurationsdatei vorhanden ist, aber die Validierung fehlschlägt, startet der klassische
  Onboarding-Pfad mit Hinweisen aus `openclaw doctor`. OpenClaw benötigt eine funktionierende
  Inferenz und wird nicht zur Reparatur dieses Zustands vor der Inferenz verwendet.
- Wenn die Konfigurationsdatei gültig ist, wird die normale Agent-TUI geöffnet. Ein erreichbares
  konfiguriertes Gateway mit einem Agenten und Modell wechselt ohne
  Onboarding oder OpenClaw direkt zu dieser Benutzeroberfläche. Bei einer konfigurierten Installation erreichen Sie OpenClaw über
  `/openclaw` innerhalb der TUI oder über `openclaw setup`.

Klartext-`ws://` wird für Loopback-, private IP-Literale, `.local`- und Tailnet-`*.ts.net`-Gateway-URLs akzeptiert. Legen Sie für andere vertrauenswürdige private DNS-Namen `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` in der Prozessumgebung des Onboardings fest.

## Zurücksetzen

```bash
openclaw onboard --reset
openclaw onboard --reset --reset-scope full
```

`--reset` löscht den Zustand, bevor die Einrichtung ausgeführt wird. `--reset-scope` steuert den Umfang: `config` (nur Konfiguration), `config+creds+sessions` (Standard, wenn `--reset` ohne Bereich übergeben wird) oder `full` (setzt auch den Arbeitsbereich zurück). Der Arbeitsbereich wird nur mit `--reset-scope full` zurückgesetzt.

## Gebietsschema

Das interaktive Onboarding verwendet das Gebietsschema des CLI-Assistenten für fest vorgegebene Einrichtungstexte. Dabei wird der erste nicht leere Wert in dieser Reihenfolge verwendet:

1. `OPENCLAW_LOCALE`
2. `LC_ALL`
3. `LC_MESSAGES`
4. `LANG`
5. Englisch als Rückfalloption

Unterstützte Gebietsschemas des Assistenten sind `en`, `zh-CN` und `zh-TW`. Gebietsschemawerte können Unterstriche oder POSIX-Suffixformen wie `zh_CN.UTF-8` verwenden. Produktnamen, Befehlsnamen, Konfigurationsschlüssel, URLs, Provider-IDs, Modell-IDs sowie Plugin-/Kanalkennzeichnungen bleiben unverändert.

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # Explizite Überschreibung mit Englisch
```

## Nicht interaktive Einrichtung

`--non-interactive` erfordert `--accept-risk` (bestätigt, dass Agenten leistungsfähig sind und vollständiger Systemzugriff riskant ist). `--mode` verwendet standardmäßig `local`.

```bash
openclaw onboard --non-interactive \
  --auth-choice custom-api-key \
  --custom-base-url "https://llm.example.com/v1" \
  --custom-model-id "foo-large" \
  --custom-api-key "$CUSTOM_API_KEY" \
  --secret-input-mode plaintext \
  --custom-compatibility openai \
  --custom-image-input
```

`--custom-api-key` ist optional; wenn es weggelassen wird, prüft das Onboarding `CUSTOM_API_KEY` in der Umgebung. OpenClaw kennzeichnet gängige Vision-Modell-IDs (GPT-4o/4.1/5.x, Claude 3/4, Gemini, Qwen-VL, LLaVA, Pixtral und ähnliche) automatisch als bildfähig. Übergeben Sie `--custom-image-input` für unbekannte benutzerdefinierte Vision-IDs oder `--custom-text-input`, um reine Textmetadaten zu erzwingen. Verwenden Sie `--custom-compatibility openai-responses` für OpenAI-kompatible Endpunkte, die `/v1/responses`, aber nicht `/v1/chat/completions` unterstützen; gültige Werte sind `openai` (Standard), `openai-responses` und `anthropic`.

LM Studio verfügt außerdem über ein providerspezifisches Schlüssel-Flag:

```bash
openclaw onboard --non-interactive \
  --auth-choice lmstudio \
  --custom-base-url "http://localhost:1234/v1" \
  --custom-model-id "qwen/qwen3.5-9b" \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --accept-risk
```

Nicht interaktives Ollama:

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

`--custom-base-url` verwendet standardmäßig `http://127.0.0.1:11434`. `--custom-model-id` ist optional; wenn es weggelassen wird, verwendet das Onboarding die von Ollama vorgeschlagenen Standardwerte. Cloud-Modell-IDs wie `kimi-k2.5:cloud` funktionieren hier ebenfalls.

Provider-Schlüssel als Referenzen statt als Klartext speichern:

```bash
openclaw onboard --non-interactive \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --accept-risk
```

Mit `--secret-input-mode ref` schreibt das Onboarding umgebungsbasierte Referenzen statt Schlüsselwerte im Klartext: Für auf Authentifizierungsprofilen basierende Provider wird `keyRef: { source: "env", provider: "default", id: <envVar> }` geschrieben; für benutzerdefinierte Provider wird `models.providers.<id>.apiKey` auf dieselbe Weise geschrieben (zum Beispiel `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`). Vertrag: Setzen Sie die Provider-Umgebungsvariable in der Prozessumgebung des Onboardings (zum Beispiel `OPENAI_API_KEY`) und übergeben Sie nicht zusätzlich ein Inline-Schlüssel-Flag, sofern diese Umgebungsvariable nicht gesetzt ist – ein Flag-Wert ohne die entsprechende Umgebungsvariable schlägt sofort mit einer Anleitung fehl.

### Gateway-Authentifizierung (nicht interaktiv)

- `--gateway-auth token --gateway-token <token>` speichert ein Klartext-Token. `token` ist der standardmäßige Authentifizierungsmodus.
- `--gateway-auth token --gateway-token-ref-env <name>` speichert `gateway.auth.token` als umgebungsbasierte SecretRef. Erfordert in der Prozessumgebung des Onboardings eine nicht leere Umgebungsvariable dieses Namens.
- `--gateway-token` und `--gateway-token-ref-env` schließen sich gegenseitig aus.
- Mit `--install-daemon`: Ein durch SecretRef verwaltetes `gateway.auth.token` wird validiert, aber nicht als aufgelöster Klartext in den Umgebungsmetadaten des Supervisor-Dienstes gespeichert; wenn die Referenz nicht aufgelöst werden kann, schlägt die Installation geschlossen mit einer Anleitung zur Behebung fehl. Wenn sowohl `gateway.auth.token` als auch `gateway.auth.password` konfiguriert sind und `gateway.auth.mode` nicht gesetzt ist, wird die Installation blockiert, bis der Modus ausdrücklich festgelegt wurde.
- Das lokale Onboarding schreibt `gateway.mode="local"` in die Konfiguration. Wenn in einer späteren Konfigurationsdatei `gateway.mode` fehlt, deutet dies auf eine beschädigte Konfiguration oder eine unvollständige manuelle Bearbeitung hin, nicht auf eine gültige Abkürzung für den lokalen Modus.
- Das lokale Onboarding installiert herunterladbare Plugins, die für den gewählten Einrichtungsweg erforderlich sind (zum Beispiel ein Codex- oder Copilot-Laufzeit-Plugin für diese Authentifizierungsoptionen). Das Remote-Onboarding schreibt lediglich Verbindungsinformationen für das Remote-Gateway – es installiert niemals lokale Plugin-Pakete.
- `--allow-unconfigured` ist ein separater `openclaw gateway run`-Notausgang; dadurch kann das Onboarding `gateway.mode` nicht überspringen.

```bash
export OPENAI_API_KEY="your-provider-key"
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN \
  --accept-risk
```

### Integritätsprüfung des lokalen Gateways

- Sofern Sie nicht `--skip-health` übergeben, wartet das Onboarding auf ein erreichbares lokales Gateway, bevor es erfolgreich beendet wird.
- `--install-daemon` startet zuerst den verwalteten Gateway-Installationspfad. Ohne dieses Flag muss bereits ein lokales Gateway ausgeführt werden (zum Beispiel `openclaw gateway run`).
- `--skip-health` überspringt das Warten, wenn Sie in einer Automatisierung nur Schreibvorgänge für Konfiguration, Arbeitsbereich und Bootstrap durchführen möchten.
- `--skip-bootstrap` setzt `agents.defaults.skipBootstrap: true` und überspringt die Erstellung von `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` und `BOOTSTRAP.md`.
- Unter nativem Windows versucht `--install-daemon` zunächst, geplante Aufgaben zu verwenden, und greift auf ein benutzerspezifisches Anmeldeelement im Autostartordner zurück, wenn die Aufgabenerstellung verweigert wird.

### Interaktiver Referenzmodus

- Wählen Sie bei der Aufforderung **Geheimnisreferenz verwenden** und anschließend entweder **Umgebungsvariable** oder einen konfigurierten Geheimnis-Provider (`file` oder `exec`).
- Das Onboarding führt vor dem Speichern der Referenz eine schnelle Vorabvalidierung aus und ermöglicht bei einem Fehlschlag einen erneuten Versuch.

### Auswahlmöglichkeiten für Z.AI-Endpunkte

<Note>
`--auth-choice zai-api-key` erkennt automatisch den besten Z.AI-Endpunkt und das beste Modell für Ihren Schlüssel: Coding-Plan-Endpunkte bevorzugen `zai/glm-5.2` (mit Rückfall auf `glm-5.1`, falls nicht verfügbar); allgemeine API-Endpunkte verwenden standardmäßig `zai/glm-5.1`. Um einen Coding-Plan-Endpunkt zu erzwingen, wählen Sie direkt `zai-coding-global` oder `zai-coding-cn`.
</Note>

```bash
# Auswahl des Endpunkts ohne Eingabeaufforderung
openclaw onboard --non-interactive \
  --auth-choice zai-coding-global \
  --zai-api-key "$ZAI_API_KEY"

# Weitere Z.AI-Endpunktauswahlen: zai-coding-cn, zai-global, zai-cn
```

Mistral:

```bash
openclaw onboard --non-interactive \
  --auth-choice mistral-api-key \
  --mistral-api-key "$MISTRAL_API_KEY"
```

## Zusätzliche nicht interaktive Flags

Tokenbasierte Modellauthentifizierung (verwendet mit `--auth-choice token`):

| Flag                            | Beschreibung                                                                                                                 |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `--token-provider <id>`         | ID des Token-Providers, der das Token ausstellt                                                                                         |
| `--token <token>`               | Tokenwert für die Modellauthentifizierung                                                                                        |
| `--token-profile-id <id>`       | ID des Authentifizierungsprofils (Standard `<provider>:manual`; einige providereigene Abläufe verwenden einen eigenen Standard, etwa `anthropic:default`) |
| `--token-expires-in <duration>` | Optionale Gültigkeitsdauer des Tokens (z. B. `365d`, `12h`)                                                                         |

Cloudflare AI Gateway: `--cloudflare-ai-gateway-account-id <id>`, `--cloudflare-ai-gateway-gateway-id <id>`.

Steuerung der Daemon-Installation: `--no-install-daemon` / `--skip-daemon` (Aliasse; überspringen die Installation des Gateway-Dienstes), `--daemon-runtime <node>`.

Skills: `--node-manager <npm|pnpm|bun>` (Standard `npm`), `--skip-skills`.

Einrichtung von UI und Hooks: `--skip-ui` (Eingabeaufforderungen für Control UI/TUI überspringen), `--skip-hooks` (Webhook-/Hook-Einrichtung überspringen), `--skip-channels`, `--skip-search`.

Ausgabe: `--suppress-gateway-token-output` unterdrückt tokenhaltige Gateway-/UI-Ausgaben (Tokenhinweise, URL für die automatische Anmeldung mit eingebettetem Token und automatischer Start der Control UI) – nützlich in gemeinsam verwendeten Terminals und in CI.

<Note>
`--json` aktiviert im geführten oder klassischen Onboarding nicht automatisch den nicht interaktiven Modus.
Mit `--modern` ist JSON eine einmalige OpenClaw-Übersicht und wird nach diesem
einzelnen Ergebnis beendet. Verwenden Sie `--non-interactive` für andere Skripte.
</Note>

## Provider-Vorfilterung

Wenn eine Authentifizierungsoption einen bevorzugten Provider vorgibt, filtert das Onboarding die Auswahlfelder für Standardmodell und Zulassungsliste vorab auf die Modelle dieses Providers. Der Filter berücksichtigt außerdem andere Provider desselben Plugins, wodurch Coding-Plan-Varianten wie `volcengine`/`volcengine-plan` und `byteplus`/`byteplus-plan` abgedeckt werden. Wenn der Filter für den bevorzugten Provider keine geladenen Modelle ergibt, greift das Onboarding auf den ungefilterten Katalog zurück, statt das Auswahlfeld leer zu lassen.

## Folgeabfragen für die Websuche

Einige Websuch-Provider lösen während des Onboardings providerspezifische Folgeabfragen aus:

- **Grok** kann eine optionale Einrichtung von `x_search` mit derselben xAI-Authentifizierung und einer Modellauswahl für `x_search` anbieten.
- **Kimi** kann nach der Moonshot-API-Region (`api.moonshot.ai` gegenüber `api.moonshot.cn`) und dem standardmäßigen Kimi-Websuchmodell fragen.

## Weitere Verhaltensweisen

- Verhalten des DM-Bereichs beim lokalen Onboarding: [Referenz zur CLI-Einrichtung](/de/start/wizard-cli-reference#outputs-and-internals).
- Schnellster erster Chat: `openclaw dashboard` (Control UI, keine Kanaleinrichtung).
- Benutzerdefinierter Provider: Verbinden Sie einen beliebigen OpenAI- oder Anthropic-kompatiblen Endpunkt, einschließlich nicht aufgeführter gehosteter Provider. Verwenden Sie die Kompatibilität **Unbekannt**, um sie mittels einer Live-Prüfung automatisch zu erkennen.
- Wenn ein Hermes-Zustand erkannt wird, bietet das Onboarding einen Migrationsablauf an (siehe `--flow import` oben).

## Häufig verwendete Folgebefehle

Verwenden Sie später `openclaw configure` für gezielte Änderungen ohne Inferenz und `openclaw
channels add` für eine reine Kanaleinrichtung. Führen Sie bei Änderungen am Modell-Provider oder an der Authentifizierungsroute
stattdessen `openclaw onboard` aus.

```bash
openclaw channels add
openclaw configure
openclaw agents add <name>
```
