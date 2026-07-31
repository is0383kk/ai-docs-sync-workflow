---
read_when:
    - Sie möchten, dass OpenClaw einen lokalen Modellserver nur dann startet, wenn dessen Modell- oder Embedding-Provider ausgewählt ist
    - Sie betreiben ds4, inferrs, vLLM, llama.cpp, MLX oder einen anderen OpenAI-kompatiblen lokalen Server
    - Sie müssen Kaltstart, Bereitschaft und Leerlaufabschaltung für lokale Provider steuern.
summary: Lokale Modellserver bei Bedarf vor Modell- und Embedding-Anfragen von OpenClaw starten
title: Lokale Modelldienste
x-i18n:
    generated_at: "2026-07-26T18:57:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a761113dd591fed0394379b2bad173165efc5e284565c652493e73d1e724529d
    source_path: gateway/local-model-services.md
    workflow: 16
---

`models.providers.<id>.localService` startet bei Bedarf einen Provider-eigenen lokalen Modellserver. Wenn eine Modell- oder Embedding-Anfrage diesen Provider auswählt, prüft OpenClaw den Zustandsendpunkt, startet den Prozess, falls er nicht ausgeführt wird, wartet auf die Betriebsbereitschaft und sendet dann die Anfrage. Verwenden Sie diese Funktion, damit teure lokale Server nicht den ganzen Tag ausgeführt werden müssen.

## Funktionsweise

1. Eine Modell- oder Embedding-Anfrage wird einem konfigurierten Provider zugeordnet.
2. Wenn dieser Provider über `localService` verfügt, prüft OpenClaw `healthUrl`.
3. Bei einer erfolgreichen Prüfung verwendet OpenClaw den bereits ausgeführten Server.
4. Bei einer fehlgeschlagenen Prüfung startet OpenClaw `command` mit `args`.
5. OpenClaw fragt den Zustandsendpunkt ab, bis `readyTimeoutMs` abläuft.
6. Die Anfrage durchläuft den normalen Modell- oder Embedding-Transport.
7. Wenn OpenClaw den Prozess gestartet hat und `idleStopMs` festgelegt ist, beendet es den Prozess, nachdem seit der letzten laufenden Anfrage für diese Dauer keine Aktivität mehr vorlag.

OpenClaw installiert hierfür weder launchd noch systemd, Docker oder einen anderen Daemon. Der Server ist ein gewöhnlicher untergeordneter Prozess des OpenClaw-Prozesses, der ihn zuerst benötigt hat.

Der Start wird pro konfiguriertem Provider und Befehls-/Argument-/Umgebungsvariablensatz serialisiert, sodass gleichzeitige Chat- und Embedding-Anfragen für denselben Dienst keine doppelten Serverprozesse starten. Jede Anfrage behält ihre eigene Lease, bis die Verarbeitung der Antwort abgeschlossen ist. Daher wartet die Leerlaufabschaltung auf sämtliche laufenden Modell- und Embedding-Anfragen. Konfigurierte Provider-Aliasse bleiben voneinander getrennt: Zwei Aliasse können auf unterschiedliche GPU-Hosts verweisen, ohne derselben Ollama-, LM-Studio- oder OpenAI-kompatiblen Adapter-ID zugeordnet zu werden.

Wenn bereits ein anderer OpenClaw-Prozess unter derselben `healthUrl` über einen funktionsfähigen Server verfügt, verwendet dieser Prozess ihn erneut, ohne ihn zu übernehmen (jeder Prozess verwaltet nur den untergeordneten Prozess, den er selbst gestartet hat). Start- und Beendigungsprotokolle enthalten begrenzte, redigierte Auszüge der Ausgabe des untergeordneten Prozesses sowie Zeit- und Beendigungsdetails; konfigurierte Umgebungswerte werden niemals ausgegeben.

## Konfigurationsstruktur

```json5
{
  models: {
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "local-model",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/absolute/path/to/server",
          args: ["--host", "127.0.0.1", "--port", "8000"],
          cwd: "/absolute/path/to/working-dir",
          env: { LOCAL_MODEL_CACHE: "/absolute/path/to/cache" },
          healthUrl: "http://127.0.0.1:8000/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "my-local-model",
            name: "My Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Legen Sie `timeoutSeconds` im Provider-Eintrag fest (nicht `localService`), damit langsame Kaltstarts und lange Generierungen nicht das standardmäßige Zeitlimit für Modellanfragen erreichen. Legen Sie ausdrücklich `healthUrl` fest, wenn Ihr Server die Betriebsbereitschaft an einer anderen Stelle als `/models` der Basis-URL bereitstellt.

## Felder

| Feld            | Erforderlich | Beschreibung                                                                                                                          |
| ---------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `command`        | ja      | Absoluter Pfad zur ausführbaren Datei. Keine Suche über den Shell-PATH.                                                                                      |
| `args`           | nein       | Prozessargumente. Keine Shell-Erweiterung, Pipes, Glob-Muster oder Anführungszeichenverarbeitung.                                                                  |
| `cwd`            | nein       | Arbeitsverzeichnis des Prozesses.                                                                                                   |
| `env`            | nein       | Umgebungsvariablen, die mit Vorrang vor der Umgebung des OpenClaw-Prozesses zusammengeführt werden.                                                                  |
| `healthUrl`      | nein       | URL für die Betriebsbereitschaft. Standardmäßig `baseUrl` mit angehängtem `/models` (`http://127.0.0.1:8000/v1` wird zu `http://127.0.0.1:8000/v1/models`). |
| `readyTimeoutMs` | nein       | Frist für die Betriebsbereitschaft beim Start. Standard: `120000`.                                                                                       |
| `idleStopMs`     | nein       | Verzögerung der Leerlaufabschaltung für einen von OpenClaw gestarteten Prozess. `0` oder das Weglassen des Feldes hält ihn aktiv, bis OpenClaw beendet wird.                             |

## Inferrs-Beispiel

Inferrs ist ein benutzerdefiniertes OpenAI-kompatibles `/v1`-Backend. Daher funktioniert dieselbe `localService`-API mit einem `inferrs`-Provider-Eintrag:

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: { requiresStringContent: true },
          },
        ],
      },
    },
  },
}
```

Ersetzen Sie `command` durch das Ergebnis von `which inferrs` auf dem Computer, auf dem OpenClaw ausgeführt wird. Vollständige Einrichtung von Inferrs: [Inferrs](/de/providers/inferrs).

## ds4-Beispiel

```json5
{
  models: {
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "<DS4_DIR>/ds4-server",
          args: [
            "--model",
            "<DS4_DIR>/ds4flash.gguf",
            "--host",
            "127.0.0.1",
            "--port",
            "18000",
            "--ctx",
            "32768",
            "--tokens",
            "128",
          ],
          cwd: "<DS4_DIR>",
          healthUrl: "http://127.0.0.1:18000/v1/models",
          readyTimeoutMs: 300000,
          idleStopMs: 0,
        },
        models: [],
      },
    },
  },
}
```

Vollständige Einrichtung, Kontextdimensionierung und Überprüfungsbefehle: [ds4](/de/providers/ds4).

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Lokale Modelle" href="/de/gateway/local-models" icon="server">
    Einrichtung lokaler Modelle, Provider-Auswahl und Sicherheitshinweise.
  </Card>
  <Card title="Inferrs" href="/de/providers/inferrs" icon="cpu">
    Führen Sie OpenClaw über den lokalen OpenAI-kompatiblen Server von Inferrs aus.
  </Card>
</CardGroup>
