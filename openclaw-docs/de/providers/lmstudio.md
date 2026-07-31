---
read_when:
    - Sie möchten OpenClaw mit Open-Source-Modellen über LM Studio ausführen
    - Sie möchten LM Studio einrichten und konfigurieren
summary: OpenClaw mit LM Studio ausführen
title: LM Studio
x-i18n:
    generated_at: "2026-07-26T18:03:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f43b4d04aad6e5edfdf224747083834ebd441aa7f91ccbf2d61de990443fc414
    source_path: providers/lmstudio.md
    workflow: 16
---

LM Studio führt llama.cpp- (GGUF) oder MLX-Modelle lokal aus, entweder als GUI-App oder als monitorlosen `llmster`-Daemon.
Installations- und Produktdokumentation finden Sie unter [lmstudio.ai](https://lmstudio.ai/).

## Schnellstart

<Steps>
  <Step title="Server installieren und starten">
    Installieren Sie LM Studio (Desktop) oder `llmster` (monitorlos) und starten Sie anschließend den Server:

    ```bash
    lms server start --port 1234
    ```

    Oder führen Sie den monitorlosen Daemon aus:

    ```bash
    lms daemon up
    ```

    Wenn Sie die Desktop-App verwenden, aktivieren Sie JIT, um Modelle reibungslos zu laden; weitere Informationen finden Sie im
    [Leitfaden zu JIT und TTL in LM Studio](https://lmstudio.ai/docs/developer/core/ttl-and-auto-evict).

  </Step>
  <Step title="API-Schlüssel festlegen, wenn die Authentifizierung aktiviert ist">
    ```bash
    export LM_API_TOKEN="your-lm-studio-api-token"
    ```

    Wenn die LM-Studio-Authentifizierung deaktiviert ist, lassen Sie den API-Schlüssel während der Einrichtung leer. Weitere Informationen finden Sie unter
    [LM-Studio-Authentifizierung](https://lmstudio.ai/docs/developer/core/authentication).

  </Step>
  <Step title="Onboarding ausführen">
    ```bash
    openclaw onboard
    ```

    Wählen Sie `LM Studio` und anschließend bei der Eingabeaufforderung `Default model` ein Modell aus.

    Bei einer neuen geführten Einrichtung fragt OpenClaw zunächst `/api/v1/models` auf dem
    standardmäßigen oder konfigurierten LM-Studio-Host ab. Ein vorhandenes LLM wird nur dann automatisch
    angeboten, wenn LM Studio Tool-Training und einen effektiven Kontext von mindestens 16K meldet.
    Bei geladenen Modellen hat der Kontext der geladenen Instanz Vorrang vor dem
    größeren angegebenen Maximum. Dieselbe Einrichtungsabfolge für CLI/macOS überprüft die
    Route mit einer echten Vervollständigung, bevor sie gespeichert wird. Die automatische Prüfung lädt niemals
    ein Modell herunter und ignoriert Katalogeinträge, die ausschließlich für Embeddings vorgesehen sind.

  </Step>
</Steps>

Ändern Sie das Standardmodell später:

```bash
openclaw models set lmstudio/qwen/qwen3.5-9b
```

LM-Studio-Modellschlüssel verwenden das Format `author/model-name` (z. B. `qwen/qwen3.5-9b`); OpenClaw-Modellreferenzen
stellen den Provider voran: `lmstudio/qwen/qwen3.5-9b`. Den genauen Schlüssel eines Modells finden Sie, indem Sie den
folgenden Befehl ausführen und das Feld `key` prüfen:

```bash
curl http://localhost:1234/api/v1/models
```

## Nicht interaktives Onboarding

```bash
openclaw onboard --non-interactive --accept-risk --auth-choice lmstudio
```

Alternativ können Sie Basis-URL, Modell und API-Schlüssel explizit angeben:

```bash
openclaw onboard \
  --non-interactive \
  --accept-risk \
  --auth-choice lmstudio \
  --custom-base-url http://localhost:1234/v1 \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --custom-model-id qwen/qwen3.5-9b
```

`--custom-model-id` übernimmt den von LM Studio zurückgegebenen Modellschlüssel (z. B. `qwen/qwen3.5-9b`) ohne
das Provider-Präfix `lmstudio/`. Übergeben Sie `--lmstudio-api-key` (oder legen Sie `LM_API_TOKEN` fest) für authentifizierte
Server; lassen Sie die Angabe bei nicht authentifizierten Servern weg. OpenClaw speichert stattdessen eine lokale, nicht geheime Markierung.
`--custom-api-key` wird aus Kompatibilitätsgründen weiterhin akzeptiert, `--lmstudio-api-key` wird jedoch bevorzugt.

Dadurch wird `models.providers.lmstudio` geschrieben und das Standardmodell auf `lmstudio/<custom-model-id>` festgelegt.
Wenn Sie einen API-Schlüssel angeben, wird außerdem das Authentifizierungsprofil `lmstudio:default` geschrieben.

Bei der interaktiven Einrichtung kann zusätzlich eine bevorzugte Ladekontextlänge abgefragt werden, die dann auf alle
erkannten Modelle angewendet wird, die in der Konfiguration gespeichert werden.

## Konfiguration

### Kompatibilität der Streaming-Nutzungsdaten

LM Studio gibt bei gestreamten Antworten nicht immer ein OpenAI-konformes `usage`-Objekt aus. OpenClaw
ermittelt die Token-Anzahlen stattdessen aus Metadaten im llama.cpp-Stil (`timings.prompt_n` / `timings.predicted_n`).
Jeder OpenAI-kompatible Endpunkt, der als lokaler Endpunkt aufgelöst wird (Loopback-Host), verwendet denselben
Fallback. Dies gilt auch für andere lokale Backends wie vLLM, SGLang, llama.cpp, LocalAI, Jan, TabbyAPI
und text-generation-webui.

### Kompatibilität des Denkmodus

Wenn die `/api/v1/models`-Erkennung von LM Studio modellspezifische Reasoning-Optionen meldet, stellt OpenClaw
entsprechende `reasoning_effort`-Werte (`none`, `minimal`, `low`, `medium`, `high`, `xhigh`) in den
Modellkompatibilitätsmetadaten bereit. Einige LM-Studio-Builds geben eine binäre UI-Option (`allowed_options: ["off",
"on"]`) an, lehnen diese
Literalwerte bei `/v1/chat/completions` jedoch ab; OpenClaw normalisiert diese binäre Form
vor dem Senden von Anfragen auf die sechsstufige Skala. Dies gilt auch für ältere gespeicherte Konfigurationen, die
noch `off`-/`on`-Reasoning-Zuordnungen enthalten.

### Explizite Konfiguration

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        models: [
          {
            id: "qwen/qwen3-coder-next",
            name: "Qwen 3 Coder Next",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### Vorabladen deaktivieren

LM Studio unterstützt das Just-in-Time-Laden (JIT) von Modellen, bei dem Modelle bei der ersten Anfrage geladen werden. OpenClaw
lädt Modelle standardmäßig über den nativen Ladeendpunkt von LM Studio vor, was hilfreich ist, wenn JIT
deaktiviert ist. Damit stattdessen JIT, die Leerlauf-TTL und das automatische Entfernen von LM Studio den Modelllebenszyklus
steuern, deaktivieren Sie den Vorabladeschritt von OpenClaw:

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        api: "openai-completions",
        params: { preload: false },
        models: [{ id: "qwen/qwen3.5-9b" }],
      },
    },
  },
}
```

### LAN- oder Tailnet-Host

Verwenden Sie die erreichbare Adresse des LM-Studio-Hosts, behalten Sie `/v1` bei und stellen Sie sicher, dass LM Studio auf diesem
Computer nicht nur an die Loopback-Schnittstelle gebunden ist:

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://gpu-box.local:1234/v1",
        apiKey: "lmstudio",
        api: "openai-completions",
        models: [{ id: "qwen/qwen3.5-9b" }],
      },
    },
  },
}
```

`lmstudio` vertraut seinem konfigurierten Endpunkt automatisch bei Modellanfragen, einschließlich Loopback-,
LAN- und Tailnet-Hosts (mit Ausnahme von Metadaten-/Link-Local-Ursprüngen). Jeder benutzerdefinierte/lokale OpenAI-kompatible
Provider-Eintrag erhält dasselbe Vertrauen in den exakten Ursprung. Anfragen an einen anderen privaten Host oder Port
erfordern weiterhin `models.providers.<id>.request.allowPrivateNetwork: true`; setzen Sie den Wert auf `false`, um
das standardmäßige Vertrauen zu deaktivieren.

## Fehlerbehebung

### LM Studio wird nicht erkannt

Stellen Sie sicher, dass LM Studio ausgeführt wird:

```bash
lms server start --port 1234
```

Wenn die Authentifizierung aktiviert ist, legen Sie außerdem `LM_API_TOKEN` fest. Prüfen Sie, ob die API erreichbar ist:

```bash
curl http://localhost:1234/api/v1/models
```

### Authentifizierungsfehler (HTTP 401)

- Prüfen Sie, ob `LM_API_TOKEN` mit dem in LM Studio konfigurierten Schlüssel übereinstimmt.
- Weitere Informationen finden Sie unter [LM-Studio-Authentifizierung](https://lmstudio.ai/docs/developer/core/authentication).
- Wenn der Server keine Authentifizierung erfordert, lassen Sie den Schlüssel während der Einrichtung leer.

## Verwandte Themen

- [Modellauswahl](/de/concepts/model-providers)
- [Ollama](/de/providers/ollama)
- [Lokale Modelle](/de/gateway/local-models)
