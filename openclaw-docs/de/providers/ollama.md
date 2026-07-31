---
read_when:
    - Sie möchten OpenClaw mit Cloud- oder lokalen Modellen über Ollama ausführen
    - Sie benötigen Anleitungen zur Einrichtung und Konfiguration von Ollama
    - Sie möchten Ollama-Vision-Modelle für die Bilderkennung verwenden
summary: OpenClaw mit Ollama ausführen (Cloud- und lokale Modelle)
title: Ollama
x-i18n:
    generated_at: "2026-07-26T18:35:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80ae833d006ce307406fac11fe3457809165035a38b7e0a970777baf126cc9cb
    source_path: providers/ollama.md
    workflow: 16
---

OpenClaw kommuniziert mit der nativen API von Ollama (`/api/chat`), nicht mit dem OpenAI-kompatiblen
Endpunkt `/v1`. Drei Modi werden unterstützt:

| Modus         | Verwendete Ressourcen                                                            |
| ------------- | -------------------------------------------------------------------------------- |
| Cloud + lokal | Ein erreichbarer Ollama-Host, der lokale Modelle und (falls angemeldet) `:cloud`-Modelle bereitstellt |
| Nur Cloud     | Direkt `https://ollama.com`, ohne lokalen Daemon                                  |
| Nur lokal     | Ein erreichbarer Ollama-Host, ausschließlich mit lokalen Modellen                |

Informationen zur reinen Cloud-Einrichtung mit der dedizierten Provider-ID `ollama-cloud` finden Sie unter
[Ollama Cloud](/de/providers/ollama-cloud). Verwenden Sie `ollama-cloud/<model>`-Referenzen, wenn
das Cloud-Routing von einem lokalen `ollama`-Provider getrennt bleiben soll.

<Warning>
Verwenden Sie nicht die OpenAI-kompatible URL `/v1` (`http://host:11434/v1`). Dadurch funktionieren Tool-Aufrufe nicht ordnungsgemäß, und Modelle können unverarbeitetes Tool-Aufruf-JSON als Klartext ausgeben. Verwenden Sie die native URL: `baseUrl: "http://host:11434"` (ohne `/v1`).
</Warning>

Der kanonische Konfigurationsschlüssel ist `baseUrl`. `baseURL` wird auch für
Beispiele im Stil des OpenAI-SDK akzeptiert, neue Konfigurationen sollten jedoch `baseUrl` verwenden.

## Authentifizierungsregeln

<AccordionGroup>
  <Accordion title="Lokale und LAN-Hosts">
    Ollama-URLs für Loopback, private Netzwerke, `.local` und einfache Hostnamen benötigen kein echtes Bearer-Token. OpenClaw verwendet für diese die Markierung `ollama-local`.
  </Accordion>
  <Accordion title="Remote- und Ollama-Cloud-Hosts">
    Öffentliche Remote-Hosts und `https://ollama.com` erfordern echte Anmeldedaten: `OLLAMA_API_KEY`, ein Authentifizierungsprofil oder den Wert `apiKey` des Providers. Bevorzugen Sie für die direkte gehostete Nutzung den Provider `ollama-cloud`.
  </Accordion>
  <Accordion title="Benutzerdefinierte Provider-IDs">
    Für einen benutzerdefinierten Provider mit `api: "ollama"` gelten dieselben Regeln. Beispielsweise kann ein auf einen privaten LAN-Host verweisender `ollama-remote`-Provider `apiKey: "ollama-local"` verwenden; Sub-Agenten lösen diese Markierung über den Ollama-Provider-Hook auf, anstatt sie als fehlende Anmeldedaten zu behandeln. `memory.search.provider` kann ebenfalls auf eine benutzerdefinierte Provider-ID verweisen, damit Einbettungen diesen Ollama-Endpunkt verwenden.
  </Accordion>
  <Accordion title="Authentifizierungsprofile">
    `auth-profiles.json` speichert die Anmeldedaten für eine Provider-ID; speichern Sie Endpunkteinstellungen (`baseUrl`, `api`, Modelle, Header, Zeitüberschreitungen) in `models.providers.<id>`. Ältere flache Dateien wie `{ "ollama-windows": { "apiKey": "ollama-local" } }` sind kein Laufzeitformat; `openclaw doctor --fix` schreibt sie mit einer Sicherung in ein kanonisches API-Schlüsselprofil `ollama-windows:default` um. Ein Wert `baseUrl` in dieser Legacy-Datei ist irrelevant und sollte in die Provider-Konfiguration verschoben werden.
  </Accordion>
  <Accordion title="Gültigkeitsbereich von Speicher-Einbettungen">
    Die Bearer-Authentifizierung für Ollama-Speicher-Einbettungen ist auf den Host beschränkt, für den sie deklariert wurde:

    - Ein Schlüssel auf Provider-Ebene wird nur an den Host dieses Providers gesendet.
    - `memory.search.remote.apiKey` und agentenspezifische Überschreibungen werden nur an ihren Remote-Einbettungshost gesendet.
    - Ein reiner Umgebungswert `OLLAMA_API_KEY` wird als Ollama-Cloud-Konvention behandelt und standardmäßig nicht an lokale oder selbst gehostete Hosts gesendet.

  </Accordion>
</AccordionGroup>

## Erste Schritte

<Tabs>
  <Tab title="Onboarding (empfohlen)">
    <Steps>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard
        ```

        Wählen Sie **Ollama** und anschließend einen Modus: **Cloud + lokal**, **Nur Cloud** oder **Nur lokal**.

        Bei einer neuen geführten Einrichtung prüft OpenClaw zunächst den standardmäßigen oder konfigurierten
        Ollama-Host. Ein installiertes Modell wird nur dann automatisch angeboten, wenn
        `/api/show` die Tool-Unterstützung und ein Kontextfenster von mindestens 16K bestätigt;
        fehlende oder kleinere Kontextmetadaten verbleiben im manuellen Einrichtungsablauf. Die
        gemeinsame Einrichtungsabfolge für CLI/macOS überprüft die ausgewählte Route weiterhin mit einer
        echten Vervollständigung, bevor sie gespeichert wird. Diese automatische Prüfung ruft niemals ein
        Modell ab; wenn kein geeignetes installiertes Modell vorhanden ist, wird das Onboarding mit der
        regulären Ollama-Auswahl fortgesetzt.
      </Step>
      <Step title="Modell auswählen">
        `Cloud only` fragt nach `OLLAMA_API_KEY` und schlägt gehostete Cloud-Standardwerte vor. `Cloud + Local` und `Local only` fragen nach einer Ollama-Basis-URL, ermitteln verfügbare Modelle und rufen das ausgewählte lokale Modell automatisch ab, falls es fehlt. Ein installiertes `:latest`-Tag wie `gemma4:latest` wird einmal angezeigt, anstatt `gemma4` zu duplizieren. `Cloud + Local` prüft außerdem, ob der Host für den Cloud-Zugriff angemeldet ist.
      </Step>
      <Step title="Überprüfen">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    Nicht interaktiv:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    `--custom-base-url` und `--custom-model-id` sind optional; wenn sie weggelassen werden, kommen der lokale Standardhost und das vorgeschlagene Modell `gemma4` zum Einsatz.

  </Tab>

  <Tab title="Manuelle Einrichtung">
    <Steps>
      <Step title="Ollama installieren und starten">
        Laden Sie Ollama von [ollama.com/download](https://ollama.com/download) herunter und rufen Sie anschließend ein Modell ab:

        ```bash
        ollama pull gemma4
        ```

        Führen Sie für hybriden Cloud-Zugriff `ollama signin` auf demselben Host aus.
      </Step>
      <Step title="Anmeldedaten festlegen">
        ```bash
        export OLLAMA_API_KEY="ollama-local"    # lokaler/LAN-Host, beliebiger Wert funktioniert
        export OLLAMA_API_KEY="your-real-key"   # nur https://ollama.com
        ```

        Alternativ in der Konfiguration: `openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"`.
      </Step>
      <Step title="Modell auswählen">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Alternativ in der Konfiguration:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Cloud-Modelle über einen lokalen Host

`Cloud + Local` leitet sowohl lokale als auch `:cloud`-Modelle über einen einzigen erreichbaren
Ollama-Host weiter. Dies ist der hybride Ablauf von Ollama und der Modus, den Sie bei der Einrichtung wählen sollten,
wenn Sie beides verwenden möchten.

OpenClaw fragt nach der Basis-URL, ermittelt lokale Modelle und prüft den
Status `ollama signin`. Wenn der Host angemeldet ist, werden gehostete Standardwerte vorgeschlagen
(`kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`, `glm-5.2:cloud`). Wenn
der Host nicht angemeldet ist, bleibt die Einrichtung rein lokal, bis Sie `ollama signin` ausführen.

Verwenden Sie für reinen Cloud-Zugriff ohne lokalen Daemon `openclaw onboard --auth-choice ollama-cloud` und lesen Sie [Ollama Cloud](/de/providers/ollama-cloud). Dieser Pfad benötigt weder `ollama signin` noch einen laufenden Server:

```bash
openclaw onboard --auth-choice ollama-cloud
openclaw models set ollama-cloud/kimi-k2.5:cloud
```

Die während `openclaw onboard` angezeigte Cloud-Modellliste wird live aus
`https://ollama.com/api/tags` befüllt und ist auf 500 Einträge begrenzt, sodass die Auswahl
den aktuellen gehosteten Katalog widerspiegelt. Wenn `ollama.com` zum Einrichtungszeitpunkt nicht erreichbar ist oder keine
Modelle zurückgibt, greift OpenClaw auf seine fest codierte Vorschlagsliste zurück, damit
das Onboarding dennoch abgeschlossen werden kann.

## Modellerkennung (impliziter Provider)

Wenn `OLLAMA_API_KEY` (oder ein Authentifizierungsprofil) festgelegt ist und weder
`models.providers.ollama` noch ein anderer benutzerdefinierter Provider mit `api: "ollama"`
definiert ist, ermittelt OpenClaw Modelle aus `http://127.0.0.1:11434`:

| Verhalten            | Details                                                                                                                                                                                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Katalogabfrage       | `/api/tags`                                                                                                                                                                                                                                                                            |
| Funktionserkennung   | Die bestmögliche Erkennung über `/api/show` liest `contextWindow`, `num_ctx`-Modelfile-Parameter und Funktionen (Bildverarbeitung/Tools/Denken)                                                                                                                              |
| Bildmodelle          | Eine `vision`-Funktion aus `/api/show` kennzeichnet das Modell als bildfähig (`input: ["text", "image"]`)                                                                                                                                                                            |
| Reasoning-Erkennung  | Verwendet, sofern verfügbar, die `thinking`-Funktion aus `/api/show`; andernfalls wird auf eine Namensheuristik (`r1`, `reason`, `reasoning`, `think`) zurückgegriffen, wenn Ollama keine Funktionen angibt. `glm-5.2:cloud` und `deepseek-v4-flash\|pro:cloud` werden unabhängig von den gemeldeten Funktionen immer als Reasoning behandelt. |
| Tokenlimits          | `maxTokens` verwendet standardmäßig die Obergrenze für Ollama-Tokens von OpenClaw                                                                                                                                                                                                      |
| Kosten               | Alle Kosten betragen `0`                                                                                                                                                                                                                                                       |

```bash
ollama list
openclaw models list
```

Das Festlegen von `models.providers.ollama` mit einem expliziten `models`-Array oder eines
benutzerdefinierten Providers mit `api: "ollama"` und einer nicht auf Loopback verweisenden `baseUrl` deaktiviert
die automatische Erkennung; Modelle müssen dann manuell definiert werden (siehe
[Konfiguration](#configuration)). Ein auf das gehostete `https://ollama.com` verweisender
`models.providers.ollama`-Eintrag überspringt die Erkennung ebenfalls, da Ollama-Cloud-Modelle
vom Provider verwaltet werden. Benutzerdefinierte Loopback-Provider wie
`http://127.0.0.2:11434` gelten weiterhin als lokal und behalten die automatische Erkennung bei.

Sie können eine vollständige Referenz wie `ollama/<pulled-model>:latest` ohne einen
manuell erstellten `models.json`-Eintrag verwenden; OpenClaw löst sie live auf. Bei angemeldeten
Hosts wird bei der Auswahl einer nicht aufgeführten `ollama/<model>:cloud`-Referenz genau dieses
Modell mit `/api/show` validiert und nur dann zum Laufzeitkatalog hinzugefügt, wenn Ollama
die Metadaten bestätigt. Tippfehler führen weiterhin zu einem Fehler wegen eines unbekannten Modells.

### Smoke-Tests

Für eine gezielte Textprüfung, die die vollständige Tool-Oberfläche des Agenten überspringt:

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "Antworten Sie exakt mit: pong" \
    --json
```

Fügen Sie für eine schlanke Prüfung eines Bildverarbeitungsmodells `--file` mit einem Bild hinzu (akzeptiert PNG/JPEG/WebP;
Nicht-Bilddateien werden abgelehnt, bevor Ollama aufgerufen wird – verwenden Sie
`openclaw infer audio transcribe` für Audio):

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "Beschreiben Sie dieses Bild in einem Satz." \
    --file ./photo.jpg \
    --json
```

Keiner der beiden Pfade lädt Chat-Tools, Speicher oder Sitzungskontext. Wenn die Ausführung erfolgreich ist,
während normale Agentenantworten fehlschlagen, liegt das Problem wahrscheinlich an der Tool-/Agentenfähigkeit
des Modells und nicht am Endpunkt.

Die Auswahl eines Modells mit `/model ollama/<model>` ist eine exakte Benutzerentscheidung: Wenn das
konfigurierte `baseUrl` nicht erreichbar ist, schlägt die nächste Antwort mit dem Provider-
Fehler fehl, statt stillschweigend auf ein anderes konfiguriertes Modell zurückzugreifen.

Isolierte Cron-Aufträge führen vor Beginn des Agent-Durchlaufs eine lokale Sicherheitsprüfung durch:
Wenn das ausgewählte Modell auf einen Ollama-Provider im lokalen/privaten Netzwerk/`.local`
aufgelöst wird und `/api/tags` nicht erreichbar ist, zeichnet OpenClaw diesen Durchlauf als
`skipped` auf, wobei das Modell im Fehlertext enthalten ist. Diese Endpunktprüfung wird pro
Host 5 Minuten lang zwischengespeichert, sodass wiederholte Cron-Aufträge bei einem angehaltenen Daemon
nicht alle fehlschlagende Anfragen starten.

Live-Verifizierung:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Richten Sie denselben Live-Test für Ollama Cloud auf den gehosteten Endpunkt aus (Embeddings werden
standardmäßig übersprungen; erzwingen Sie sie mit `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1`, da ein
Cloud-Schlüssel möglicherweise `/api/embed` nicht autorisiert):

```bash
export OLLAMA_API_KEY='<your-ollama-cloud-api-key>'
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud \
OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=1 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Um ein Modell hinzuzufügen, laden Sie es herunter; es wird automatisch erkannt:

```bash
ollama pull mistral
```

## Node-lokale Inferenz

Agenten können eine kurze Aufgabe an ein Ollama-Modell auf einem gekoppelten Desktop- oder
Server-Node delegieren. Prompt und Antwort werden über die bestehende authentifizierte
Gateway-/Node-Verbindung übertragen; die Anfrage wird am Ollama-Loopback-Endpunkt des
Nodes (`http://127.0.0.1:11434`) ausgeführt.

<Steps>
  <Step title="Ollama auf dem Node starten">
    ```bash
    ollama pull qwen3:0.6b
    ollama list
    ```
  </Step>
  <Step title="Node-Host verbinden">
    ```bash
    openclaw node run \
      --host <gateway-host> \
      --port 18789 \
      --display-name "Local inference"
    ```

    Genehmigen Sie das Gerät und seine Node-Befehle auf dem Gateway-Host und überprüfen Sie anschließend:

    ```bash
    openclaw devices list
    openclaw devices approve <deviceRequestId>
    openclaw nodes pending
    openclaw nodes approve <nodeRequestId>
    openclaw nodes status --connected
    ```

    Eine erste Verbindung oder ein Upgrade, das Ollama-Befehle hinzufügt, kann eine
    Genehmigung der Node-Befehle auslösen. Wenn sich der Node verbindet, ohne
    `ollama.models` und `ollama.chat` anzukündigen, prüfen Sie `openclaw nodes pending` erneut.

  </Step>
  <Step title="Von einem Agenten verwenden">
    Das mitgelieferte Ollama-Plugin stellt das Tool `node_inference` bereit. Agenten rufen
    zuerst `action: "discover"` und anschließend `action: "run"` mit einem Node und Modell aus
    diesem Ergebnis auf (`run` kann den Node auslassen, wenn genau ein geeigneter Node
    verbunden ist). Beispiel: „Ermitteln Sie die Ollama-Modelle auf meinen Nodes und verwenden
    Sie dann das schnellste geladene Modell, um diesen Text zusammenzufassen.“
  </Step>
</Steps>

Die Erkennung liest `/api/tags`, prüft die Fähigkeiten von `/api/show` und verwendet
`/api/ps`, sofern verfügbar, um bereits geladene Modelle zuerst einzustufen. Sie gibt nur
lokale Modelle zurück, die Ollama als chatfähig meldet (Fähigkeit `completion`) —
Ollama-Cloud-Einträge und reine Embedding-Modelle werden ausgeschlossen. Jeder Durchlauf deaktiviert
das Denken des Modells und begrenzt die Ausgabe standardmäßig auf 512 Token (feste Obergrenze 8192),
sofern der Tool-Aufruf kein anderes `maxTokens` anfordert; einige Modelle (beispielsweise GPT-OSS)
unterstützen das Deaktivieren des Denkens nicht und geben möglicherweise weiterhin Reasoning-Token aus.

So lassen Sie Ollama auf einem Node laufen, ohne es Agenten bereitzustellen:

```bash
openclaw config set plugins.entries.ollama.config.nodeInference.enabled false
```

Starten Sie den Node neu (`openclaw node restart`, oder beenden Sie `openclaw node run` und führen Sie es
für eine Vordergrundsitzung erneut aus). Der Node kündigt `ollama.models` und
`ollama.chat` nicht mehr an; Ollama selbst und der Ollama-Provider des Gateways bleiben davon
unberührt. Setzen Sie den Wert wieder auf `true` und starten Sie neu, um die Funktion
wieder zu aktivieren; eine geänderte Befehlsoberfläche kann nach der erneuten Verbindung wieder eine
Genehmigung von `openclaw nodes pending` erfordern.

Überprüfen Sie die Node-Befehle direkt und ohne Agent-Durchlauf:

```bash
openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.models \
  --params '{}' \
  --invoke-timeout 90000 \
  --timeout 100000

openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.chat \
  --params '{"model":"qwen3:0.6b","prompt":"Reply with exactly: pong","maxTokens":32,"timeoutMs":120000}' \
  --invoke-timeout 130000 \
  --timeout 140000
```

`--invoke-timeout` begrenzt, wie lange der Node den Befehl ausführen darf;
`--timeout` begrenzt den gesamten Gateway-Aufruf und sollte größer sein.

Node-lokale Inferenz verwendet immer den eigenen Loopback-Endpunkt des Nodes — sie
verwendet kein konfiguriertes entferntes/Cloud-`models.providers.ollama.baseUrl`. Die
Node-Befehle sind standardmäßig auf macOS-, Linux- und Windows-Node-Hosts verfügbar
und unterliegen weiterhin den üblichen Richtlinien für Node-Kopplung und -Befehle.

## Bildverarbeitung und Bildbeschreibung

Das mitgelieferte Ollama-Plugin registriert Ollama als bildfähigen
Provider für das Medienverständnis, sodass OpenClaw explizite Anfragen zur Bildbeschreibung
und konfigurierte Standardwerte für Bildmodelle über lokale oder gehostete Ollama-
Bildverarbeitungsmodelle leiten kann.

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --json
```

`--model` muss eine vollständige `<provider/model>`-Referenz sein; wenn diese festgelegt ist, versucht `infer image
describe` zuerst dieses Modell, statt die Beschreibung bei Modellen
zu überspringen, die bereits native Bildverarbeitung unterstützen. Wenn der Aufruf fehlschlägt, kann OpenClaw
mit `agents.defaults.imageModel.fallbacks` fortfahren; Fehler bei der Datei-/URL-
Vorbereitung treten auf, bevor ein Fallback versucht wird. Verwenden Sie `infer image describe` für den
Bildverständnisablauf von OpenClaw und das konfigurierte `imageModel`; verwenden Sie `infer model run
--file` für eine direkte multimodale Prüfung mit einem benutzerdefinierten Prompt.

So legen Sie Ollama als standardmäßigen Provider für das Bildverständnis eingehender Medien fest:

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

Bevorzugen Sie die vollständige `ollama/<model>`-Referenz. Eine einfache `imageModel`-Referenz wie
`qwen2.5vl:7b` wird nur dann zu `ollama/qwen2.5vl:7b` normalisiert, wenn genau dieses Modell
unter `models.providers.ollama.models` mit
`input: ["text", "image"]` aufgeführt ist und kein anderer konfigurierter Bild-Provider dieselbe
einfache ID bereitstellt; verwenden Sie andernfalls explizit das Provider-Präfix.

Langsame lokale Bildverarbeitungsmodelle können für das Bildverständnis ein längeres Zeitlimit als
Cloud-Modelle benötigen und auf Hardware mit eingeschränkten Ressourcen abstürzen, wenn Ollama versucht,
den gesamten angekündigten Bildkontext des Modells zuzuweisen. Legen Sie ein Fähigkeits-
Zeitlimit fest und begrenzen Sie `num_ctx`:

```json5
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5vl:7b",
            name: "qwen2.5vl:7b",
            input: ["text", "image"],
            params: { num_ctx: 2048, keep_alive: "1m" },
          },
        ],
      },
    },
  },
  tools: {
    media: {
      image: {
        timeoutSeconds: 180,
        models: [{ provider: "ollama", model: "qwen2.5vl:7b", timeoutSeconds: 300 }],
      },
    },
  },
}
```

Dieses Zeitlimit gilt für das Verständnis eingehender Bilder und für das explizite
Tool `image`. `models.providers.ollama.timeoutSeconds` steuert weiterhin die
zugrunde liegende Absicherung der Ollama-HTTP-Anfrage für normale Modellaufrufe.

Live-Verifizierung:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA_IMAGE=1 \
  pnpm test:live -- src/agents/tools/image-tool.ollama.live.test.ts
```

Wenn Sie `models.providers.ollama.models` manuell definieren, kennzeichnen Sie Bildverarbeitungsmodelle
explizit:

```json5
{
  id: "qwen2.5vl:7b",
  name: "qwen2.5vl:7b",
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 8192,
}
```

OpenClaw lehnt Anfragen zur Bildbeschreibung für Modelle ab, die nicht als
bildfähig gekennzeichnet sind. Bei impliziter Erkennung stammt diese Angabe aus der
Bildverarbeitungsfähigkeit von `/api/show`.

## Konfiguration

<Tabs>
  <Tab title="Grundlegend (implizite Erkennung)">
    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    Wenn `OLLAMA_API_KEY` festgelegt ist, können Sie `apiKey` im Provider-Eintrag auslassen; OpenClaw ergänzt es für Verfügbarkeitsprüfungen.
    </Tip>

  </Tab>

  <Tab title="Explizit (manuelle Modelle)">
    Verwenden Sie eine explizite Konfiguration für eine gehostete Cloud-Einrichtung, einen vom Standard
    abweichenden Host/Port, erzwungene Kontextfenster oder vollständig manuelle Modelllisten:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 128000,
                maxTokens: 8192
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="Benutzerdefinierte Basis-URL">
    Eine explizite Konfiguration deaktiviert die automatische Erkennung, daher müssen die Modelle aufgeführt werden:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // Kein /v1 – native Ollama-API-URL
            api: "ollama", // Explizit: gewährleistet natives Tool-Calling-Verhalten
            timeoutSeconds: 300, // Optional: längeres Verbindungs-/Streaming-Budget für kalte lokale Modelle
            models: [
              {
                id: "qwen3:32b",
                name: "qwen3:32b",
                params: {
                  keep_alive: "15m", // Optional: hält das Modell zwischen Durchläufen geladen
                },
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    Fügen Sie `/v1` nicht hinzu. Dieser Pfad wählt den OpenAI-kompatiblen Modus aus, in dem Tool-Aufrufe nicht zuverlässig sind.
    </Warning>

  </Tab>
</Tabs>

## Häufige Konfigurationen

Ersetzen Sie Modell-IDs durch die exakten Namen aus `ollama list` oder
`openclaw models list --provider ollama`.

<AccordionGroup>
  <Accordion title="Lokales Modell mit automatischer Erkennung">
    Ollama auf demselben Rechner wie das Gateway, automatisch erkannt:

    ```bash
    ollama serve
    ollama pull gemma4
    export OLLAMA_API_KEY="ollama-local"
    openclaw models list --provider ollama
    openclaw models set ollama/gemma4
    ```

    Fügen Sie keinen `models.providers.ollama`-Block hinzu, sofern Sie keine manuellen Modelle benötigen.

  </Accordion>

  <Accordion title="Ollama-Host im LAN mit manuellen Modellen">
    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                reasoning: true,
                input: ["text"],
                params: {
                  num_ctx: 32768,
                  thinking: false,
                  keep_alive: "15m",
                },
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/qwen3.5:9b" },
        },
      },
    }
    ```

    `contextWindow` ist das Kontextbudget von OpenClaw; `params.num_ctx` wird an
    Ollama gesendet. Halten Sie beide aufeinander abgestimmt, wenn die Hardware nicht den gesamten
    angekündigten Kontext des Modells ausführen kann.

  </Accordion>

  <Accordion title="Nur Ollama Cloud">
    Kein lokaler Daemon, direkt gehostete Modelle:

    ```bash
    export OLLAMA_API_KEY="your-ollama-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                contextWindow: 128000,
                maxTokens: 8192,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/kimi-k2.5:cloud" },
        },
      },
    }
    ```

    Informationen zur Verwendung der dedizierten Provider-ID `ollama-cloud` anstelle dieser Struktur finden Sie unter
    [Ollama Cloud](/de/providers/ollama-cloud).

  </Accordion>

  <Accordion title="Cloud und lokale Modelle über einen angemeldeten Daemon">
    ```bash
    ollama signin
    ollama pull gemma4
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            models: [
              { id: "gemma4", name: "gemma4", input: ["text"] },
              { id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text", "image"] },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama/gemma4",
            fallbacks: ["ollama/kimi-k2.5:cloud"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Mehrere Ollama-Hosts">
    Benutzerdefinierte Provider-IDs beim Betrieb mehrerer Ollama-Server; jeder erhält einen
    eigenen Host, eigene Modelle, eine eigene Authentifizierung und ein eigenes Zeitlimit.

    ```json5
    {
      models: {
        providers: {
          "ollama-fast": {
            baseUrl: "http://mini.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [{ id: "gemma4", name: "gemma4", input: ["text"] }],
          },
          "ollama-large": {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 420,
            contextWindow: 131072,
            maxTokens: 16384,
            models: [{ id: "qwen3.5:27b", name: "qwen3.5:27b", input: ["text"] }],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama-fast/gemma4",
            fallbacks: ["ollama-large/qwen3.5:27b"],
          },
        },
      },
    }
    ```

    OpenClaw entfernt vor dem Aufruf von Ollama das Präfix des aktiven Providers (mit Rückgriff auf ein einfaches
    Präfix `ollama/`), sodass `ollama-large/qwen3.5:27b`
    Ollama als `qwen3.5:27b` erreicht.

  </Accordion>

  <Accordion title="Schlankes lokales Modellprofil">
    Einige lokale Modelle verarbeiten einfache Prompts, haben jedoch Schwierigkeiten mit dem vollständigen
    Tool-Umfang des Agenten. Begrenzen Sie Tools und Kontext, bevor Sie globale
    Laufzeiteinstellungen ändern:

    ```json5
    {
      agents: {
        list: [
          {
            id: "local",
            experimental: {
              localModelLean: true,
            },
            model: { primary: "ollama/gemma4" },
          },
        ],
      },
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [
              {
                id: "gemma4",
                name: "gemma4",
                input: ["text"],
                params: { num_ctx: 32768 },
                compat: { supportsTools: false },
              },
            ],
          },
        },
      },
    }
    ```

    Verwenden Sie `compat.supportsTools: false` nur, wenn das Modell oder der Server bei
    Tool-Schemas zuverlässig fehlschlägt — dabei wird Agentenfunktionalität zugunsten der Stabilität aufgegeben.
    `localModelLean` entfernt ressourcenintensive Browser-, Cron-, Nachrichten-, Mediengenerierungs-,
    Sprach- und PDF-Tools aus dem direkten Agentenumfang, sofern sie nicht ausdrücklich erforderlich sind,
    und stellt größere Kataloge über die Tool-Suche bereit. Dies ändert weder den
    Laufzeitkontext noch den Denkmodus von Ollama. Kombinieren Sie es mit `params.num_ctx` und
    `params.thinking: false` für kleine Qwen-artige Denkmodelle, die Schleifen bilden oder
    ihr Budget für verborgenes Schlussfolgern aufwenden.

  </Accordion>
</AccordionGroup>

### Modellauswahl

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

Benutzerdefinierte Provider-IDs funktionieren genauso: Bei einer Referenz, die das Präfix des aktiven Providers
verwendet, wie etwa `ollama-spark/qwen3:32b`, entfernt OpenClaw dieses Präfix vor dem
Aufruf von Ollama und sendet `qwen3:32b`.

Bei langsamen lokalen Modellen sollten Sie zunächst providerspezifische Anpassungen vornehmen, bevor Sie das Zeitlimit
der gesamten Agentenlaufzeit erhöhen:

```json5
{
  models: {
    providers: {
      ollama: {
        timeoutSeconds: 300,
        models: [
          {
            id: "gemma4:26b",
            name: "gemma4:26b",
            params: { keep_alive: "15m" },
          },
        ],
      },
    },
  },
}
```

`timeoutSeconds` umfasst die HTTP-Anfrage an das Modell: Verbindungsaufbau, Header,
Body-Streaming und den gesamten geschützten Fetch-Abbruch. `params.keep_alive` wird
bei nativen `/api/chat`-Anfragen als `keep_alive` auf oberster Ebene weitergeleitet; legen Sie den Wert pro
Modell fest, wenn die Ladezeit beim ersten Durchlauf der Engpass ist.

### Schnellüberprüfung

```bash
# Ollama-Daemon ist für diesen Rechner erreichbar
curl http://127.0.0.1:11434/api/tags

# OpenClaw-Katalog und ausgewähltes Modell
openclaw models list --provider ollama
openclaw models status

# Direkter Modell-Schnelltest
openclaw infer model run \
  --model ollama/gemma4 \
  --prompt "Antworten Sie exakt mit: ok"
```

Ersetzen Sie bei Remote-Hosts `127.0.0.1` durch den Host `baseUrl`. Wenn `curl`
funktioniert, OpenClaw jedoch nicht, prüfen Sie, ob der Gateway auf einem anderen
Rechner, in einem Container oder unter einem anderen Dienstkonto ausgeführt wird.

## Ollama-Websuche

OpenClaw enthält die **Ollama-Websuche** als Provider `web_search`.

| Eigenschaft   | Details                                                                                                                                                    |
| ------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Host          | `models.providers.ollama.baseUrl`, wenn festgelegt, andernfalls `http://127.0.0.1:11434`; `https://ollama.com` verwendet die gehostete API direkt                          |
| Authentifizierung | Ohne Schlüssel für einen angemeldeten lokalen Host; `OLLAMA_API_KEY` oder konfigurierte Provider-Authentifizierung für die direkte Suche über `https://ollama.com` oder authentifizierungsgeschützte Hosts           |
| Voraussetzung | Lokale/selbst gehostete Hosts müssen ausgeführt werden und mit `ollama signin` angemeldet sein; die direkte gehostete Suche benötigt `baseUrl: "https://ollama.com"` sowie einen echten API-Schlüssel |

Wählen Sie ihn während `openclaw onboard` oder `openclaw configure --section web` aus, oder legen Sie Folgendes fest:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Für die direkte gehostete Suche über Ollama Cloud:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
        api: "ollama",
        models: [{ id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text"] }],
      },
    },
  },
  tools: {
    web: {
      search: { provider: "ollama" },
    },
  },
}
```

Bei einem selbst gehosteten Host versucht OpenClaw zuerst den lokalen Proxy `/api/experimental/web_search`
und greift anschließend auf den gehosteten Pfad `/api/web_search` auf demselben Host zurück; ein
angemeldeter lokaler Daemon antwortet normalerweise über den lokalen Proxy. Direkte
Aufrufe von `https://ollama.com` verwenden immer den gehosteten Endpunkt `/api/web_search`.

<Note>
Die vollständige Einrichtung und das Verhalten werden unter [Ollama-Websuche](/de/tools/ollama-search) beschrieben.
</Note>

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Veralteter OpenAI-kompatibler Modus">
    <Warning>
    **Tool-Aufrufe sind in diesem Modus nicht zuverlässig.** Verwenden Sie ihn nur, wenn ein Proxy das OpenAI-Format benötigt und Sie nicht auf native Tool-Aufrufe angewiesen sind.
    </Warning>

    Legen Sie `api: "openai-completions"` ausdrücklich für einen Proxy hinter
    `/v1/chat/completions` fest:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // Standard: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    Dieser Modus unterstützt möglicherweise nicht gleichzeitig Streaming und Tool-Aufrufe; eventuell
    benötigen Sie `params: { streaming: false }` für das Modell.

    OpenClaw fügt in diesem Modus standardmäßig `options.num_ctx` ein, damit Ollama nicht
    unbemerkt auf einen Kontext mit 4096 Token zurückfällt. Wenn Ihr Proxy
    unbekannte Felder vom Typ `options` ablehnt, deaktivieren Sie dies:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Kontextfenster">
    Bei automatisch erkannten Modellen verwendet OpenClaw das von `/api/show`
    gemeldete Kontextfenster, einschließlich größerer `PARAMETER num_ctx`-Werte aus benutzerdefinierten
    Modelfiles; andernfalls wird auf das standardmäßige Ollama-Kontextfenster von OpenClaw
    zurückgegriffen.

    `contextWindow`, `contextTokens` und `maxTokens` auf Provider-Ebene legen
    Standardwerte für jedes Modell unter diesem Provider fest und können pro
    Modell überschrieben werden. `contextWindow` ist OpenClaws eigenes Prompt-/Compaction-Budget. Native
    `/api/chat`-Anfragen lassen `options.num_ctx` ungesetzt, sofern Sie
    `params.num_ctx` nicht ausdrücklich festlegen; dadurch verwendet Ollama seinen eigenen modell-,
    `OLLAMA_CONTEXT_LENGTH`- oder VRAM-basierten Standardwert. Ungültige, nullwertige, negative
    oder nicht endliche `params.num_ctx`-Werte werden ignoriert. Wenn eine ältere Konfiguration
    ausschließlich `contextWindow`/`maxTokens` verwendete, um den Kontext nativer Anfragen zu erzwingen, führen Sie
    `openclaw doctor --fix` aus, um diese Werte nach `params.num_ctx` zu kopieren. Der
    OpenAI-kompatible Adapter fügt `options.num_ctx` weiterhin standardmäßig aus
    dem konfigurierten `params.num_ctx` oder `contextWindow` ein; deaktivieren Sie dies mit
    `injectNumCtxForOpenAICompat: false`, wenn das Upstream-System `options` ablehnt.

    Native Modelleinträge akzeptieren unter `params` außerdem gängige Ollama-Laufzeitoptionen,
    die als native `/api/chat`-`options` weitergeleitet werden: `num_keep`, `seed`,
    `num_predict`, `top_k`, `top_p`, `min_p`, `typical_p`, `repeat_last_n`,
    `temperature`, `repeat_penalty`, `presence_penalty`, `frequency_penalty`,
    `stop`, `num_batch`, `num_gpu`, `main_gpu`, `use_mmap` und `num_thread`.
    Einige Schlüssel (`format`, `keep_alive`, `truncate`, `shift`) werden als
    Anfragefelder auf oberster Ebene statt als verschachtelte `options` weitergeleitet. OpenClaw
    leitet nur diese Ollama-Anfrageschlüssel weiter, sodass reine Laufzeitparameter wie
    `streaming` niemals an Ollama gesendet werden. Verwenden Sie `params.think` (oder
    `params.thinking`), um `think` auf oberster Ebene festzulegen; `false` deaktiviert das Denken
    auf API-Ebene für Qwen-artige Denkmodelle.

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
                params: {
                  num_ctx: 32768,
                  temperature: 0.7,
                  top_p: 0.9,
                  thinking: false,
                },
              }
            ]
          }
        }
      }
    }
    ```

    Pro Modell funktioniert `agents.defaults.models["ollama/<model>"].params.num_ctx` ebenfalls;
    der explizite Provider-Modelleintrag hat Vorrang, wenn beide festgelegt sind.

  </Accordion>

  <Accordion title="Steuerung des Denkens">
    OpenClaw leitet das Denken so weiter, wie Ollama es erwartet: `think` auf oberster Ebene, nicht
    `options.think`. Automatisch erkannte Modelle, deren `/api/show` eine
    `thinking`-Fähigkeit meldet, stellen `/think low`, `/think medium`, `/think high`
    und `/think max` bereit; Modelle ohne Denkfunktion stellen nur `/think off` bereit.

    ```bash
    openclaw agent --model ollama/gemma4 --thinking off
    openclaw agent --model ollama/gemma4 --thinking low
    ```

    Alternativ können Sie einen Modellstandard festlegen:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "ollama/gemma4": {
              thinking: "low",
            },
          },
        },
      },
    }
    ```

    Mit `params.think`/`params.thinking` pro Modell lässt sich das API-Denken
    für ein bestimmtes Modell deaktivieren oder erzwingen. OpenClaw behält diese explizite Konfiguration bei,
    wenn der aktive Lauf nur den impliziten Standard `off` verwendet; ein Laufzeitbefehl,
    der nicht „off“ entspricht, wie `/think medium`, setzt sie dennoch außer Kraft. Eine aktivierte
    Denkanforderung wird niemals an ein Modell gesendet, das explizit mit
    `reasoning: false` gekennzeichnet ist; eine `think: false`-Anforderung wird unabhängig davon immer gesendet.

  </Accordion>

  <Accordion title="Reasoning-Modelle">
    Modelle namens `deepseek-r1`, `reasoning`, `reason` oder `think` werden
    standardmäßig als Reasoning-fähig behandelt — es ist keine zusätzliche Konfiguration erforderlich:

    ```bash
    ollama pull deepseek-r1:32b
    ```

  </Accordion>

  <Accordion title="Modellkosten">
    Ollama wird lokal ausgeführt und ist kostenlos, daher betragen sämtliche Modellkosten sowohl für
    automatisch erkannte als auch für manuell definierte Modelle `0`.
  </Accordion>

  <Accordion title="Speicher-Embeddings">
    Das mitgelieferte Ollama-Plugin registriert einen Provider für Speicher-Embeddings für die
    [Speichersuche](/de/concepts/memory). Er verwendet die konfigurierte Ollama-Basis-URL
    und den API-Schlüssel, ruft `/api/embed` auf und fasst nach Möglichkeit mehrere Speicherabschnitte
    in einer `input`-Anforderung zusammen.

    Bei `proxy.enabled=true` verwenden Embedding-Anforderungen an den exakten hostlokalen
    Loopback-Ursprung, der aus dem konfigurierten `baseUrl` abgeleitet wird, den abgesicherten
    direkten Pfad von OpenClaw anstelle des verwalteten Forward-Proxys. Der konfigurierte
    Hostname muss selbst `localhost` oder ein Loopback-IP-Literal sein — DNS-Namen,
    die lediglich zu Loopback aufgelöst werden, verwenden weiterhin den verwalteten Proxy-Pfad. Ollama-Hosts
    im LAN, Tailnet, privaten Netzwerk oder öffentlichen Netzwerk verbleiben stets auf dem
    verwalteten Proxy-Pfad, und Weiterleitungen zu einem anderen Host/Port übernehmen
    das Vertrauen nicht. `proxy.loopbackMode: "proxy"` leitet Loopback-Datenverkehr trotzdem durch den
    Proxy; `proxy.loopbackMode: "block"` lehnt ihn vor dem Verbindungsaufbau ab —
    siehe [Verwalteter Proxy](/de/security/network-proxy#gateway-loopback-mode).

    | Eigenschaft | Wert |
    | --- | --- |
    | Standardmodell | `nomic-embed-text` |
    | Automatischer Abruf | Ja, falls lokal nicht vorhanden |
    | Standardmäßige Inline-Parallelität | 1 (andere Provider verwenden standardmäßig höhere Werte; erhöhen Sie den Wert mit `nonBatchConcurrency`, wenn der Host dies bewältigen kann) |

    Embeddings zur Abfragezeit verwenden für Modelle, die diese erfordern oder
    empfehlen, Abrufpräfixe: `nomic-embed-text`, `qwen3-embedding` und
    `mxbai-embed-large`. Dokument-Batches bleiben unverändert, sodass vorhandene Indizes
    keine Formatmigration benötigen.

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          remote: {
            // Standard für Ollama. Auf größeren Hosts erhöhen, wenn die Neuindizierung zu langsam ist.
            nonBatchConcurrency: 1,
          },
        },
      },
    }
    ```

    Begrenzen Sie bei einem entfernten Embedding-Host die Authentifizierung auf diesen Host:

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          model: "nomic-embed-text",
          remote: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            nonBatchConcurrency: 2,
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Streaming-Konfiguration">
    Ollama verwendet standardmäßig die **native API** (`/api/chat`), die
    Streaming und Tool-Aufrufe gemeinsam unterstützt — es ist keine besondere Konfiguration erforderlich.

    Bei nativen Anforderungen wird die Steuerung des Denkens direkt weitergeleitet: `/think off`
    und `openclaw agent --thinking off` senden `think: false` auf oberster Ebene, sofern
    kein explizites `params.think`/`params.thinking` konfiguriert ist; `/think
    low|medium|high` senden die entsprechende Aufwandszeichenfolge; `/think max` wird Ollamas höchstem Aufwand `think: "high"` zugeordnet.

    <Tip>
    Informationen zur Verwendung des OpenAI-kompatiblen Endpunkts finden Sie stattdessen oben unter „Legacy OpenAI-kompatibler Modus“ — Streaming und Tool-Aufrufe funktionieren dort möglicherweise nicht gemeinsam.
    </Tip>

  </Accordion>
</AccordionGroup>

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="WSL2-Absturzschleife (wiederholte Neustarts)">
    Unter WSL2 mit NVIDIA/CUDA erstellt das offizielle Ollama-Linux-Installationsprogramm eine
    `ollama.service`-systemd-Unit mit `Restart=always`. Wenn dieser Dienst
    automatisch startet und während des WSL2-Starts ein GPU-gestütztes Modell lädt, kann Ollama beim Laden
    Hostspeicher fest belegen; die Hyper-V-Speicherrückgewinnung kann diese
    Seiten nicht immer zurückgewinnen, sodass Windows die WSL2-VM beenden kann, systemd
    Ollama neu startet und sich die Schleife wiederholt.

    Hinweise: wiederholte WSL2-Neustarts/-Beendigungen, hohe CPU-Auslastung in `app.slice` oder
    `ollama.service` direkt nach dem WSL2-Start und SIGTERM von systemd statt
    durch den Linux-OOM-Killer.

    OpenClaw protokolliert beim Start eine Warnung, wenn es WSL2, aktiviertes `ollama.service`
    mit `Restart=always` und sichtbare CUDA-Markierungen erkennt.

    Abhilfe:

    ```bash
    sudo systemctl disable ollama
    ```

    Fügen Sie auf der Windows-Seite Folgendes zu `%USERPROFILE%\.wslconfig` hinzu und führen Sie anschließend
    `wsl --shutdown` aus:

    ```ini
    [experimental]
    autoMemoryReclaim=disabled
    ```

    Alternativ können Sie die Keep-Alive-Zeit verkürzen bzw. Ollama nur bei Bedarf manuell starten:

    ```bash
    export OLLAMA_KEEP_ALIVE=5m
    ollama serve
    ```

    Siehe [ollama/ollama#11317](https://github.com/ollama/ollama/issues/11317).

  </Accordion>

  <Accordion title="Ollama wird nicht erkannt">
    Vergewissern Sie sich, dass Ollama ausgeführt wird, `OLLAMA_API_KEY` (oder ein Authentifizierungsprofil) festgelegt ist
    und `models.providers.ollama` **nicht** explizit definiert ist:

    ```bash
    ollama serve
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Keine Modelle verfügbar">
    Rufen Sie das Modell lokal ab oder definieren Sie es explizit in
    `models.providers.ollama`:

    ```bash
    ollama list  # Anzeigen, was installiert ist
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Oder ein anderes Modell
    ```

  </Accordion>

  <Accordion title="Verbindung abgelehnt">
    ```bash
    # Prüfen, ob Ollama ausgeführt wird
    ps aux | grep ollama

    # Oder Ollama neu starten
    ollama serve
    ```

  </Accordion>

  <Accordion title="Entfernter Host funktioniert mit curl, aber nicht mit OpenClaw">
    Überprüfen Sie dies auf demselben Rechner und in derselben Laufzeitumgebung, in der das Gateway ausgeführt wird:

    ```bash
    openclaw gateway status --deep
    curl http://ollama-host:11434/api/tags
    ```

    Häufige Ursachen:

    - `baseUrl` verweist auf `localhost`, aber das Gateway wird in Docker oder auf einem anderen Host ausgeführt.
    - Die URL verwendet `/v1`, wodurch OpenAI-kompatibles Verhalten anstelle des nativen Ollama-Verhaltens ausgewählt wird.
    - Der entfernte Host erfordert Änderungen an der Firewall oder LAN-Bindung.
    - Das Modell befindet sich im Daemon Ihres Laptops, jedoch nicht im entfernten Daemon.

  </Accordion>

  <Accordion title="Modell gibt Tool-JSON als Text aus">
    Üblicherweise befindet sich der Provider im OpenAI-kompatiblen Modus oder das Modell kann
    Tool-Schemas nicht verarbeiten. Bevorzugen Sie den nativen Modus:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            api: "ollama",
          },
        },
      },
    }
    ```

    Wenn ein kleines lokales Modell bei Tool-Schemas weiterhin fehlschlägt, legen Sie
    `compat.supportsTools: false` für diesen Modelleintrag fest und testen Sie erneut.

  </Accordion>

  <Accordion title="Kimi oder GLM gibt unleserliche Symbole zurück">
    Gehostete Kimi-/GLM-Antworten, die aus langen Folgen nichtsprachlicher Symbole bestehen, werden
    als fehlgeschlagener Provider-Aufruf und nicht als erfolgreiche Antwort behandelt, sodass
    die normale Wiederholungs-/Fallback-/Fehlerbehandlung übernimmt, statt beschädigten
    Text dauerhaft in der Sitzung zu speichern.

    Wenn das Problem erneut auftritt, erfassen Sie den Modellnamen, die aktuelle Sitzungsdatei und
    ob der Lauf `Cloud + Local` oder `Cloud only` verwendet hat. Versuchen Sie anschließend eine neue
    Sitzung und ein Fallback-Modell:

    ```bash
    openclaw infer model run --model ollama/kimi-k2.5:cloud --prompt "Antworten Sie exakt mit: ok" --json
    openclaw models set ollama/gemma4
    ```

  </Accordion>

  <Accordion title="Zeitüberschreitung bei kaltem lokalem Modell">
    Große lokale Modelle können beim ersten Laden viel Zeit benötigen. Begrenzen Sie die Zeitüberschreitung auf den
    Ollama-Provider und lassen Sie das Modell optional zwischen Interaktionen geladen:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            timeoutSeconds: 300,
            models: [
              {
                id: "gemma4:26b",
                name: "gemma4:26b",
                params: { keep_alive: "15m" },
              },
            ],
          },
        },
      },
    }
    ```

    Wenn der Host selbst Verbindungen nur langsam annimmt, verlängert `timeoutSeconds` außerdem
    die abgesicherte Verbindungszeitüberschreitung für diesen Provider.

  </Accordion>

  <Accordion title="Modell mit großem Kontext ist zu langsam oder der Arbeitsspeicher reicht nicht aus">
    Viele Modelle geben Kontextgrößen an, die auf Ihrer Hardware nicht
    problemlos ausgeführt werden können. Das native Ollama verwendet seinen eigenen Laufzeitstandard, sofern
    `params.num_ctx` nicht festgelegt ist. Begrenzen Sie sowohl das Budget von OpenClaw als auch den Anforderungskontext
    von Ollama, um eine vorhersagbare Latenz bis zum ersten Token zu erreichen:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                params: { num_ctx: 32768, thinking: false },
              },
            ],
          },
        },
      },
    }
    ```

    Verringern Sie `contextWindow`, wenn OpenClaw einen zu großen Prompt sendet. Verringern Sie
    `params.num_ctx`, wenn der Laufzeitkontext von Ollama für den Rechner zu groß ist.
    Verringern Sie `maxTokens`, wenn die Generierung zu lange dauert.

  </Accordion>
</AccordionGroup>

<Note>
Weitere Hilfe: [Fehlerbehebung](/de/help/troubleshooting) und [FAQ](/de/help/faq).
</Note>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Ollama Cloud" href="/de/providers/ollama-cloud" icon="cloud">
    Reine Cloud-Einrichtung mit dem dedizierten `ollama-cloud`-Provider.
  </Card>
  <Card title="Modell-Provider" href="/de/concepts/model-providers" icon="layers">
    Übersicht über alle Provider, Modellreferenzen und das Failover-Verhalten.
  </Card>
  <Card title="Modellauswahl" href="/de/concepts/models" icon="brain">
    Auswahl und Konfiguration von Modellen.
  </Card>
  <Card title="Ollama-Websuche" href="/de/tools/ollama-search" icon="magnifying-glass">
    Vollständige Einrichtungs- und Verhaltensdetails für die Ollama-gestützte Websuche.
  </Card>
  <Card title="Konfiguration" href="/de/gateway/configuration" icon="gear">
    Vollständige Konfigurationsreferenz.
  </Card>
</CardGroup>
