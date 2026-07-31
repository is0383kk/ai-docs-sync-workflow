---
read_when:
    - Sie möchten einen verwalteten Schlüssel für mehrere Modell-Provider verwenden
    - Sie benötigen die ClawRouter-Modellerkennung oder Kontingentberichte in OpenClaw
summary: Modelle mit Anmeldedatenbereich über ClawRouter weiterleiten und verwaltete Kontingente anzeigen
title: ClawRouter
x-i18n:
    generated_at: "2026-07-26T18:41:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 929a93e8d1d003e21f792d0fdab9542553ffab374f59d4d0505819b0f719591f
    source_path: providers/clawrouter.md
    workflow: 16
---

ClawRouter stellt OpenClaw einen richtliniengebundenen Schlüssel für mehrere vorgelagerte Modell-
Provider bereit. Das mitgelieferte Plugin `clawrouter` erkennt nur die für
diesen Schlüssel zugelassenen Modelle, leitet jedes Modell über sein deklariertes Protokoll weiter und meldet
das Budget des Schlüssels sowie die aggregierte Nutzung auf den OpenClaw-Nutzungsoberflächen.

Vorgelagerte Zugangsdaten und providerspezifische Weiterleitungen verbleiben in ClawRouter, sodass
Sie niemals die Plugins der einzelnen vorgelagerten Provider auf dem
OpenClaw-Host installieren oder authentifizieren müssen. Das Plugin wird mit OpenClaw mitgeliefert (`enabledByDefault: true`);
Sie benötigen lediglich ausgestellte ClawRouter-Zugangsdaten.

| Eigenschaft   | Wert                                     |
| ------------- | ---------------------------------------- |
| Provider      | `clawrouter`                       |
| Plugin        | mitgeliefert (in OpenClaw enthalten)     |
| Authentifizierung | `CLAWROUTER_API_KEY`                  |
| Standard-URL  | `https://clawrouter.openclaw.ai`                       |
| Modellkatalog | Zugangsdatengebunden über `/v1/catalog` |
| Kontingente   | Monatsbudget und Nutzung über `/v1/usage` |

## Erste Schritte

<Steps>
  <Step title="Richtliniengebundene Zugangsdaten anfordern">
    Bitten Sie Ihre ClawRouter-Administration um Zugangsdaten, deren Richtlinie
    die Provider, Modelle und das Monatsbudget umfasst, die Sie verwenden sollen. Zugangsdaten werden
    bei der Ausstellung einmalig angezeigt.
  </Step>
  <Step title="OpenClaw konfigurieren">
    ```bash
    export CLAWROUTER_API_KEY="..."
    openclaw onboard --auth-choice clawrouter-api-key
    openclaw plugins enable clawrouter
    ```

    `clawrouter` wird mitgeliefert und ist standardmäßig aktiviert. Falls Ihre Konfiguration
    `plugins.allow` festlegt, fügen Sie `clawrouter` dieser Liste hinzu, bevor Sie es aktivieren. Legen Sie bei einer
    benutzerdefinierten Bereitstellung `models.providers.clawrouter.baseUrl` auf den
    ClawRouter-Ursprung fest; der Standardwert ist `https://clawrouter.openclaw.ai`.

  </Step>
  <Step title="Freigegebene Modelle auflisten">
    ```bash
    openclaw models list --all --provider clawrouter
    ```

    Verwenden Sie die zurückgegebenen Modellreferenzen genau wie angezeigt. Sie behalten den vorgelagerten
    Namensraum bei, beispielsweise `clawrouter/openai/gpt-5.5`,
    `clawrouter/anthropic/claude-sonnet-4-6` oder
    `clawrouter/google/gemini-3.5-flash`. Falls `agents.defaults.modelPolicy.allow`
    konfiguriert ist, fügen Sie ihm jede ausgewählte ClawRouter-Referenz hinzu.

  </Step>
  <Step title="Modell auswählen">
    ```bash
    openclaw models set clawrouter/<provider>/<model>
    ```

    Sie können ein zurückgegebenes Modell auch für einen einzelnen Lauf mit
    `openclaw agent --model clawrouter/<provider>/<model> --message "..."` auswählen.

  </Step>
</Steps>

## Verwaltete nicht interaktive Bereitstellung

Bewahren Sie den Proxy-Schlüssel in der Secret-Einspeisung der Workload auf und speichern Sie in
`openclaw.json` ausschließlich eine SecretRef. Die kanonischen verwalteten Felder sind:

| Zweck         | Konfigurations- oder Umgebungsfeld                                        |
| ------------- | ------------------------------------------------------------------------ |
| Router-Ursprung | `models.providers.clawrouter.baseUrl`                                                     |
| Zugangsdaten  | `models.providers.clawrouter.apiKey` -> Umgebungs-SecretRef                                 |
| Secret-Wert   | `CLAWROUTER_API_KEY` in der Prozessumgebung des Gateways                    |
| Standardmodell | `agents.defaults.model.primary` -> `clawrouter/<provider>/<model>`                                |
| Workload-Tag  | `models.providers.clawrouter.headers.X-ClawRouter-Project-Id` (optional)                                             |

Beispielsweise kann ein Bereitstellungscontroller diesen JSON5-Patch verwalten:

```json5
{
  plugins: {
    entries: { clawrouter: { enabled: true } },
  },
  models: {
    providers: {
      clawrouter: {
        baseUrl: "https://clawrouter.internal.example",
        apiKey: {
          source: "env",
          provider: "default",
          id: "CLAWROUTER_API_KEY",
        },
        headers: {
          "X-ClawRouter-Project-Id": "fakeco",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "clawrouter/openai/gpt-5.5" },
    },
  },
}
```

Falls die Bereitstellung `plugins.allow` festlegt, behalten Sie die vorhandenen Einträge bei und fügen Sie
`clawrouter` hinzu. Validieren und übernehmen Sie den Patch ohne interaktiven Assistenten:

```bash
openclaw config patch --file ./clawrouter.patch.json5 --dry-run --json
openclaw config patch --file ./clawrouter.patch.json5
```

Der Probelauf löst die SecretRef auf, gibt ihren Wert jedoch niemals aus. Um die
Zugangsdaten zu rotieren, aktualisieren Sie das externe Secret, das `CLAWROUTER_API_KEY` bereitstellt, und
starten Sie die Gateway-Workload neu, damit die neue Prozessumgebung geladen wird. Die
Konfigurationsdatei und die Modellreferenz ändern sich nicht.

Bei einem aus dem Quellcode erstellten eigenständigen Docker-Gateway ist ClawRouter bereits in
der Root-Laufzeit enthalten. Wählen Sie nur das Kanal-Plugin aus, das separat paketiert werden muss,
beispielsweise `OPENCLAW_EXTENSIONS=clickclack`, `slack` oder `msteams`; siehe
[aus dem Quellcode erstellte Images mit ausgewählten Plugins](/de/install/docker#source-built-images-with-selected-plugins).
Archiv-/Appliance-Bereitstellungen müssen denselben übernommenen Quellcode über ihre
eigene Artefakt-Pipeline paketieren, statt das OCI-Image zu verwenden.

## Bereitschaft und Live-Nachweis

Diese Prüfungen weisen unterschiedliche Grenzen nach; ersetzen Sie keine durch eine andere:

```bash
# Nur Prozesszustand von ClawRouter; Zugangsdaten und vorgelagerte Modelle werden nicht geprüft.
curl -fsS https://clawrouter.internal.example/v1/health

# Nur Startbereitschaft des OpenClaw-Gateways; es wird kein Modellaufruf durchgeführt.
curl -fsS http://127.0.0.1:18789/readyz

# Zugangsdatengebundene Katalogerkennung.
openclaw models list --all --provider clawrouter --json

# Minimale echte Inferenzprüfung über den konfigurierten ClawRouter-Provider.
openclaw models status --probe --probe-provider clawrouter --probe-max-tokens 8 --json

# Workload-Canary mit einer exakt freigegebenen Modellreferenz.
openclaw agent --agent main \
  --model clawrouter/openai/gpt-5.5 \
  --message "Antworten Sie exakt: CLAWROUTER_CANARY_OK" \
  --json
```

Verwenden Sie ein vom richtliniengebundenen Katalog zurückgegebenes Modell, statt das Beispielmodell
blind zu übernehmen. Eine erfolgreiche `/readyz`-Antwort bedeutet, dass das Gateway Anfragen bedienen
kann; sie besagt nicht, dass ClawRouter, seine Zugangsdaten oder ein vorgelagerter
Provider bereit sind. Die Modellprüfung und der Agent-Canary sind die Inferenznachweise.

Führen Sie zur Live-Diagnose den Canary aus und prüfen Sie die Standardprotokolle des Gateways.
Die vorhandenen, ausschließlich metadatenbasierten Modelltransportdiagnosen erzeugen Zeilen wie:

```text
[model-fetch] Start provider=clawrouter api=openai-responses model=openai/gpt-5.5 method=POST url=https://clawrouter.internal.example/v1/responses
[model-fetch] Antwort provider=clawrouter api=openai-responses model=openai/gpt-5.5 status=200
```

Das Plugin sendet begrenzte Header `X-ClawRouter-Client`, `X-ClawRouter-Agent-Id` und
`X-ClawRouter-Session-Id`, wenn diese Kennungen verfügbar sind. Es ordnet außerdem
die diagnostische `callId` (`<run-id>:model:<n>`) des Modellaufrufs
`X-Request-ID` zu, sodass ein OpenClaw-Modellaufrufereignis mit dem
ausschließlich metadatenbasierten Audit-Trail von ClawRouter verknüpft werden kann. Werte innerhalb des
128-Zeichen-Budgets für Anforderungs-IDs sind identisch. Bei längeren Werten bleiben das Suffix
`:model:<n>` und ein deterministischer Hash erhalten, sodass verschiedene Aufrufe begrenzt und
verknüpfbar bleiben. Statische Bereitstellungsmetadaten wie `X-ClawRouter-Project-Id` können in der
Provider-Zuordnung `headers` festgelegt werden. Header zur Agent- und
Sitzungszuordnung behalten ihr separates Limit von 256 Zeichen. Automatische Anforderungs-IDs,
die Zeichen außerhalb des ASCII-Kennungssatzes von ClawRouter enthalten, verwenden dieselbe
deterministische begrenzte Form.
Explizit konfigurierte Header, einschließlich jeder Groß-/Kleinschreibungsvariante von `X-Request-ID`, haben
Vorrang vor automatischen Werten. Die Transportdiagnose zeichnet Routing- und
Antwortmetadaten auf; sie protokolliert weder Zugangsdaten noch Anforderungs-IDs, Prompts oder Ausgaben.
Das eigene Audit-Ereignis von ClawRouter liefert den ausgewählten vorgelagerten Provider und
den Zustand der Inhaltsaufbewahrung.

## Modellerkennung

`GET /v1/catalog` gibt `{ providers: [...] }` zurück, wobei jeder Provider-Eintrag
seine eigenen `models[]` (mit vorgelagerter ID, Funktionen und Preisen) sowie seine
unterstützten Anforderungsrouten auflistet. OpenClaw liefert keine zweite, feste Liste von
ClawRouter-Modellen aus. Ein Katalogmodell wird als OpenClaw-Modell angeboten, wenn:

- die Richtlinie der Zugangsdaten seinen Provider freigibt;
- das Katalogmodell eine unterstützte LLM-Funktion angibt (`llm.responses`,
  `llm.chat`, `llm.messages` oder `llm.stream` mit einer passenden Streaming-
  Route); und
- der Provider eine passende Route für einen der nachstehenden Transporte bereitstellt.

Das Hinzufügen eines Modells zu einem unterstützten ClawRouter-Provider erfordert keine OpenClaw-Veröffentlichung:
Bei der nächsten Katalogaktualisierung (60 Sekunden pro Zugangsdatenbereich zwischengespeichert) wird
es erkannt. Ein Modell, das ein neues Übertragungsprotokoll benötigt, erfordert zunächst Unterstützung durch das Plugin.

## Protokoll- und Provider-Plugins

ClawRouter verwaltet die vorgelagerten Zugangsdaten; sein Katalog teilt OpenClaw mit, welcher
Transport verwendet werden soll, sodass Sie nicht das Authentifizierungs-Plugin jedes vorgelagerten Unternehmens installieren müssen.

| Katalogfunktion/-route                                  | OpenClaw-Transport      |
| -------------------------------------------------------- | ---------------------- |
| `llm.responses` (OpenAI-kompatibler Provider)         | `openai-responses`     |
| `llm.chat` (OpenAI-kompatibler Provider)         | `openai-completions`     |
| `llm.messages` + `anthropic.messages`-Route            | `anthropic-messages`     |
| `llm.stream` + Streaming-Route `google.generate_content`  | `google-generative-ai`     |

Das Plugin wendet außerdem die passenden Richtlinien für Replay und Werkzeugschemas für diese
Familien an (Werkzeugschema-Kompatibilität für OpenAI/DeepSeek/Gemini/Perplexity; native
Replay-Richtlinien von Anthropic und Google Gemini). Perplexity-Modelle erhalten eine strikte
Schemaumschreibung: `patternProperties` und `additionalProperties` werden entfernt und
jedes Objektschema deklariert `properties`, da Perplexity Werkzeugschemas
ohne diese Angaben ablehnt. Ein Katalog-Provider, der ausschließlich ein
nicht unterstütztes Anfrageformat bereitstellt, wird absichtlich nicht als OpenClaw-
Textmodell angeboten. Normalisieren Sie diese Provider in ClawRouter auf einen der unterstützten
Verträge, statt eine inkompatible Nutzlast zu senden.

## Kontingente und Nutzung

Die Antwort `/v1/usage` von ClawRouter speist die normalen OpenClaw-Oberflächen zur
Provider-Nutzung: Summen für Anfragen, Token und Ausgaben sowie ein monatliches Budgetfenster, wenn
der Schlüssel ein Limit besitzt. Schlüssel ohne Mengenbegrenzung zeigen weiterhin die aggregierte Nutzung ohne
Prozentfenster an.

Die Kontingentabfrage verwendet denselben richtliniengebundenen Schlüssel wie die Modellerkennung. Eine fehlgeschlagene
Kontingentabfrage blockiert die Modellausführung nicht.

Prüfen Sie den Live-Snapshot mit:

```bash
openclaw status --usage
openclaw models status
```

Derselbe Provider-Snapshot ist für `/status` im Chat und in der
Nutzungsoberfläche von OpenClaw verfügbar. Das Budget gilt für die gesamte Richtlinie; daher können Anfragen eines anderen Clients,
der dieselbe ClawRouter-Richtlinie verwendet, den verbleibenden Prozentsatz ändern.

## Fehlerbehebung

| Symptom                                  | Prüfung                                                                                                                                        |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Keine ClawRouter-Modelle                 | Prüfen Sie, ob das Plugin aktiviert und durch `plugins.allow` zugelassen ist, und vergewissern Sie sich anschließend, dass die Zugangsdaten aktiv sind und mindestens einen bereiten Provider freigeben. |
| Ein konfiguriertes ClawRouter-Modell fehlt | Prüfen Sie seine `/v1/catalog`-Funktion und die Routenunterstützung. Nicht unterstützte Transportverträge werden absichtlich herausgefiltert. |
| Modellüberschreibung von Richtlinie abgelehnt | Fügen Sie die exakte Katalogreferenz oder `clawrouter/*` zu `agents.defaults.modelPolicy.allow` hinzu. |
| `401` oder `403` von Katalog oder Nutzung | Stellen Sie die ClawRouter-Zugangsdaten neu aus oder ändern Sie ihren Geltungsbereich; OpenClaw greift nicht auf Schlüssel vorgelagerter Provider zurück. |
| Modellaufruf schlägt nach Erkennung fehl | Prüfen Sie die Provider-Verbindung und den Zustand des vorgelagerten Dienstes in ClawRouter und versuchen Sie es erneut, sobald dessen Bereitschaft wiederhergestellt ist. |
| Nutzung enthält Summen, aber keinen Prozentsatz | Die Richtlinie ist nicht mengenbegrenzt; fügen Sie in ClawRouter ein Monatsbudget hinzu, um ein Prozentfenster bereitzustellen. |

## Sicherheitsverhalten

- Die Katalogermittlung ist auf den konfigurierten Proxy-Schlüssel beschränkt und wird pro Anmeldedatenbereich (Agent-Verzeichnis, Workspace-Verzeichnis, Authentifizierungsprofil-ID und Basis-URL) zwischengespeichert.
- Der Proxy-Schlüssel wird erst beim Senden der Anfrage angefügt; er wird nicht in den Modellmetadaten gespeichert.
- Werte für die automatische Zuordnung und Anfragekorrelation werden vor dem Senden gekürzt und bei enthaltenen Steuerzeichen abgelehnt. Zuordnungswerte sind auf 256 Zeichen begrenzt; Anfrage-IDs auf 128.
- Diagnosedaten zum Modelltransport enthalten ausschließlich Metadaten und niemals den Proxy-Schlüssel oder Modellinhalte.
- Native Anthropic- und Gemini-Modell-IDs werden erst beim Senden in ihre Upstream-IDs umgeschrieben.
- Nicht unterstützte oder nicht freigegebene Katalogeinträge werden standardmäßig abgelehnt und können nicht ausgewählt werden.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modell-Provider" href="/de/concepts/model-providers" icon="layers">
    Provider-Konfiguration und Modellauswahl.
  </Card>
  <Card title="Nutzungsverfolgung" href="/de/concepts/usage-tracking" icon="chart-line">
    OpenClaw-Oberflächen für Nutzung und Status.
  </Card>
</CardGroup>
