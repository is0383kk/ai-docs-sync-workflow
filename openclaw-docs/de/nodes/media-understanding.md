---
read_when:
    - Medienverständnis entwerfen oder refaktorieren
    - Optimierung der Vorverarbeitung eingehender Audio-, Video- und Bilddaten
sidebarTitle: Media understanding
summary: Verarbeitung eingehender Bilder, Audio- und Videodaten (optional) mit Provider- und CLI-Fallbacks
title: Medienverständnis
x-i18n:
    generated_at: "2026-07-26T19:02:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 38e9a0f89607bb9c4af85689ef0fbd3df9234b36e06d86c129e0d823d6e05143
    source_path: nodes/media-understanding.md
    workflow: 16
---

OpenClaw kann eingehende Medien (Bild/Audio/Video) zusammenfassen, bevor die Antwort-Pipeline ausgeführt wird, sodass Befehlsanalyse und Routing mit kurzem Text statt mit Rohbytes arbeiten. Understanding erkennt lokale Tools oder Provider-Schlüssel automatisch, alternativ können Sie explizite Modelle konfigurieren. Die Originalmedien werden dem Modell wie gewohnt immer bereitgestellt. Wenn Understanding fehlschlägt oder deaktiviert ist, wird der Antwortablauf unverändert fortgesetzt.

Provider-Plugins registrieren Fähigkeitsmetadaten (welcher Provider welchen Medientyp unterstützt, Standardmodell, Priorität). Der OpenClaw-Kern verwaltet die gemeinsame `tools.media`-Konfiguration, die Fallback-Reihenfolge und die Integration in die Antwort-Pipeline.

## Funktionsweise

<Steps>
  <Step title="Anhänge erfassen">
    Geordnete Fakten zu eingehenden Medien erfassen (`path`, `url`, `contentType` und `kind`).
  </Step>
  <Step title="Nach Fähigkeit auswählen">
    Für jede aktivierte Fähigkeit (Bild/Audio/Video) Anhänge gemäß der `attachments`-Richtlinie auswählen (Standard: nur der erste Anhang).
  </Step>
  <Step title="Modell auswählen">
    Den ersten geeigneten Modelleintrag auswählen (Größe + Fähigkeit + verfügbare Authentifizierung).
  </Step>
  <Step title="Bei Fehler auf Fallback zurückgreifen">
    Wenn ein Modell einen Fehler meldet, das Zeitlimit überschreitet oder das Medium `maxBytes` überschreitet, den nächsten Eintrag versuchen.
  </Step>
  <Step title="Bei Erfolg anwenden">
    `Body` wird zu einem `[Image]`-, `[Audio]`- oder `[Video]`-Block. Audio setzt außerdem `{{Transcript}}`; für die Befehlsanalyse wird, sofern vorhanden, der Untertiteltext verwendet, andernfalls das Transkript. Untertitel bleiben innerhalb des Blocks als `User text:` erhalten.
  </Step>
</Steps>

## Konfiguration

`tools.media` enthält eine einzige nach Fähigkeiten gekennzeichnete Modellliste sowie kompakte Steuerungsoptionen pro Fähigkeit:

```json5
{
  tools: {
    media: {
      concurrency: 2, // maximale Anzahl gleichzeitiger Fähigkeitsausführungen (Standard)
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["image", "video"] },
      ],
      image: { preferredModel: "google/gemini-3-flash-preview" },
      audio: { enabled: true },
      video: { enabled: true },
    },
  },
}
```

Schlüssel pro Fähigkeit (`image`/`audio`/`video`):

| Schlüssel         | Typ       | Standard                               | Hinweise                                                             |
| ---------------- | --------- | -------------------------------------- | -------------------------------------------------------------------- |
| `enabled`        | `boolean` | automatisch (`false` deaktiviert)                | `false` festlegen, um die automatische Erkennung für diese Fähigkeit auszuschalten |
| `preferredModel` | `string`  | erster kompatibler Eintrag             | `provider/model`, Modell-ID, `provider:<id>` oder `cli:command` bevorzugen |
| `prompt`         | `string`  | Standard der Fähigkeit                 | Standard-Prompt, wenn ein Eintrag ihn nicht überschreibt             |
| `maxChars`       | `number`  | `500` für Bild/Video, für Audio nicht festgelegt | Standardmäßiges Ausgabelimit                                         |
| `maxBytes`       | `number`  | 10MB Bild, 20MB Audio, 50MB Video      | Standardmäßiges Eingabelimit                                         |
| `timeoutSeconds` | `number`  | `60` für Bild/Audio, `120` für Video | Standardmäßiges Anfragezeitlimit                                     |
| `language`       | `string`  | nicht festgelegt                       | Hinweis zur Audiotranskription                                       |
| `scope`          | object    | nicht festgelegt                       | Nach Kanal-/Chattyp-/Quellschlüssel einschränken                     |
| `attachments`    | object    | `{ mode: "first", maxAttachments: 1 }` | Auswählen, welche passenden Anhänge verarbeitet werden               |
| `echoTranscript` | `boolean` | `false`                     | Nur Audio: das Transkript vor der Agent-Verarbeitung ausgeben        |
| `echoFormat`     | `string`  | `'📝 "{transcript}"'`                  | Nur Audio: Format für das ausgegebene Transkript                     |

Prompts, Limits, Sprachhinweise, Anfrageüberschreibungen und Provider-Optionen können als Standardwerte einer Fähigkeit festgelegt oder in einzelnen `tools.media.models[]`-Einträgen überschrieben werden. Die Standardwerte einer Fähigkeit gelten auch für automatisch erkannte Provider, wenn kein explizites Modell konfiguriert ist.

### Modelleinträge

Jeder `models[]`-Eintrag ist ein **Provider**-Eintrag (Standard) oder ein **CLI**-Eintrag:

<Tabs>
  <Tab title="Provider-Eintrag">
    ```json5
    {
      type: "provider", // Standard, wenn nicht angegeben
      provider: "openai",
      model: "gpt-5.6-sol",
      prompt: "Beschreiben Sie das Bild in <= 500 Zeichen.",
      maxChars: 500,
      maxBytes: 10485760,
      timeoutSeconds: 60,
      capabilities: ["image"],
      profile: "vision-profile",
      preferredProfile: "vision-fallback",
    }
    ```
  </Tab>
  <Tab title="CLI-Eintrag">
    ```json5
    {
      type: "cli",
      command: "gemini",
      args: [
        "-m",
        "gemini-3-flash",
        "--allowed-tools",
        "read_file",
        "Lesen Sie das Medium unter {{AttachmentPath}} und beschreiben Sie es in <= {{MaxChars}} Zeichen.",
      ],
      maxChars: 500,
      maxBytes: 52428800,
      timeoutSeconds: 120,
      capabilities: ["video", "image"],
    }
    ```

    CLI-Vorlagen können außerdem `{{AttachmentUrl}}`, `{{AttachmentContentType}}`, `{{AttachmentDir}}`, `{{AttachmentIndex}}`, `{{OutputDir}}` (für diese Ausführung erstelltes temporäres Verzeichnis) und `{{OutputBase}}` (Basispfad der temporären Datei ohne Erweiterung) verwenden. Die älteren Namen `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` und `{{MediaDir}}` bleiben als veraltete Kompatibilitätsaliase erhalten.

  </Tab>
</Tabs>

### Provider-Anmeldedaten

Das Medienverständnis über Provider verwendet dieselbe Authentifizierungsauflösung wie normale Modellaufrufe: Authentifizierungsprofile, Umgebungsvariablen und anschließend `models.providers.<providerId>.apiKey`. `tools.media.models[]`-Einträge akzeptieren kein eingebettetes `apiKey`-Feld.

```json5
{
  models: {
    providers: {
      openai: { apiKey: "<OPENAI_API_KEY>" },
      moonshot: { apiKey: "<MOONSHOT_API_KEY>" },
    },
  },
}
```

Informationen zu Profilen, Umgebungsvariablen und benutzerdefinierten Basis-URLs finden Sie unter [Tools und benutzerdefinierte Provider](/de/gateway/config-tools).

## Regeln und Verhalten

- Medien, die `maxBytes` überschreiten, werden für dieses Modell übersprungen; anschließend wird das nächste versucht.
- Audiodateien mit weniger als 1024 Bytes werden als leer/beschädigt behandelt und vor der Transkription übersprungen; der Agent erhält stattdessen ein deterministisches Platzhaltertranskript.
- Wenn das aktive primäre Bildmodell Vision bereits nativ unterstützt, überspringt OpenClaw den `[Image]`-Zusammenfassungsblock und übergibt das Originalbild direkt an das Modell. MiniMax bildet eine Ausnahme: `minimax`, `minimax-cn`, `minimax-portal` und `minimax-portal-cn` leiten das Bildverständnis immer über den Plugin-eigenen Medien-Provider `MiniMax-VL-01`, selbst wenn ältere MiniMax-M2.x-Chatmetadaten Bildeingaben angeben (nur `MiniMax-M3` und neuere Versionen gelten als nativ visionfähig).
- Wenn ein primäres Gateway-/WebChat-Modell nur Text unterstützt, bleiben Bildanhänge als ausgelagerte `media://inbound/*`-Referenzen erhalten, damit Bild-/PDF-Tools oder ein konfiguriertes Bildmodell sie weiterhin untersuchen können, statt den Anhang zu verlieren.
- Ein explizites `openclaw infer image describe --file <path> --model <provider/model>` (Alias: `openclaw capability image describe`) führt diesen bildfähigen Provider bzw. dieses Modell direkt aus, einschließlich Ollama-Referenzen wie `ollama/qwen2.5vl:7b`, wenn unter `models.providers.ollama.models[]` ein passendes bildfähiges Modell konfiguriert ist.
- Wenn `<capability>.enabled` nicht `false` ist, aber keine Modelle konfiguriert sind, versucht OpenClaw das aktive Antwortmodell, sofern dessen Provider die Fähigkeit unterstützt.

### Automatische Erkennung (Standard)

Wenn `tools.media.<capability>.enabled` nicht `false` ist und keine Modelle konfiguriert sind, versucht OpenClaw die folgenden Optionen der Reihe nach und beendet die Suche bei der ersten funktionierenden Option:

<Steps>
  <Step title="Konfiguriertes Bildmodell (nur Bild)">
    Primäre/Fallback-Referenzen aus `agents.defaults.imageModel`, sofern das aktive Antwortmodell Vision nicht bereits nativ unterstützt. `provider/model`-Referenzen werden bevorzugt; nicht qualifizierte Referenzen werden nur anhand konfigurierter bildfähiger Provider-Modelleinträge qualifiziert, wenn die Übereinstimmung eindeutig ist.
  </Step>
  <Step title="Aktives Antwortmodell">
    Das aktive Antwortmodell, sofern dessen Provider die Fähigkeit unterstützt.
  </Step>
  <Step title="Provider-Authentifizierung (nur Audio, vor lokalen CLIs)">
    Konfigurierte `models.providers.*`-Einträge mit Audiounterstützung werden vor lokalen CLIs versucht. Gebündelte Provider-Prioritätsreihenfolge (Gleichstände werden alphabetisch nach Provider-ID aufgelöst): Groq/OpenAI &rarr; xAI &rarr; Deepgram &rarr; OpenRouter &rarr; Google/SenseAudio &rarr; Deepinfra/ElevenLabs &rarr; Mistral.
  </Step>
  <Step title="Lokale CLIs (nur Audio)">
    Verfügbare lokale Binärdateien bilden eine geordnete Fallback-Liste:
    - `whisper-cli` nur dann zuerst, wenn ein früherer Modellaufruf im aktuellen Prozess Metal oder CUDA erkannt hat
    - Standardmäßig CPU-basiertes `sherpa-onnx-offline` (erfordert `SHERPA_ONNX_MODEL_DIR` mit `tokens.txt`/`encoder.onnx`/`decoder.onnx`/`joiner.onnx`)
    - `whisper-cli`, wenn Beschleunigung lediglich beim Build unterstützt wird oder noch nicht erkannt wurde
    - `parakeet-mlx` auf Apple Silicon (MLX-fähig, Gerätenutzung nicht erkannt)
    - `whisper` (Python-CLI; verwendet standardmäßig das Modell `turbo` und lädt es automatisch herunter)

    Die Überprüfung der Backend-Fähigkeiten wird zwischengespeichert und lädt kein Modell. Build-Fähigkeit, angeforderte Backend-Flags und das bei einem tatsächlichen Aufruf erkannte Backend bleiben getrennt. Automatisch erkanntes whisper.cpp lässt Protokolle der Modellausführung aktiviert, damit die vorgelagerte Zeile zum ausgewählten Backend aufgezeichnet werden kann. Explizite CLI-Einträge behalten ihre konfigurierte Reihenfolge sowie ihre Backend- und Ausgabe-Flags bei.

  </Step>
  <Step title="Provider-Authentifizierung (Bild/Video)">
    Konfigurierte `models.providers.*`-Einträge, die die Fähigkeit unterstützen, werden vor der gebündelten Fallback-Reihenfolge versucht. Nur für Bilder konfigurierte Provider mit einem bildfähigen Modell werden automatisch für das Medienverständnis registriert, auch wenn sie kein gebündeltes Provider-Plugin sind.

    Gebündelte Provider-Prioritätsreihenfolge (Gleichstände werden alphabetisch nach Provider-ID aufgelöst):
    - Bild: Anthropic/OpenAI &rarr; Google &rarr; MiniMax &rarr; Deepinfra &rarr; MiniMax Portal &rarr; Z.AI
    - Video: Google &rarr; Qwen &rarr; Moonshot

  </Step>
  <Step title="Antigravity-CLI (nur Bild/Video)">
    Die erste installierte Binärdatei `agy` oder `antigravity` (mit `OPENCLAW_ANTIGRAVITY_CLI` überschreibbar), in einer Sandbox auf das Verzeichnis des Mediums beschränkt.
  </Step>
</Steps>

So deaktivieren Sie die automatische Erkennung für eine Fähigkeit:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: false,
      },
    },
  },
}
```

<Note>
Die Binärdateierkennung erfolgt unter macOS/Linux/Windows nach bestem Bemühen. Stellen Sie sicher, dass sich die CLI unter `PATH` befindet (`~` wird expandiert), oder legen Sie einen expliziten CLI-Modelleintrag mit einem vollständigen Befehlspfad fest.
</Note>

### Proxy-Unterstützung (Provider-Aufrufe für Audio/Video)

Provider-basiertes **Audio**- und **Video**-Understanding berücksichtigt die üblichen Umgebungsvariablen für ausgehende Proxys einschließlich der Umgehungsregeln `NO_PROXY`/`no_proxy`: `HTTPS_PROXY`, `HTTP_PROXY`, `ALL_PROXY`, `https_proxy`, `http_proxy`, `all_proxy`. Variablen in Kleinbuchstaben haben Vorrang vor solchen in Großbuchstaben. Wenn keine festgelegt sind, verwendet das Medienverständnis einen direkten ausgehenden Zugriff. Ist der Proxy-Wert fehlerhaft, protokolliert OpenClaw eine Warnung und greift auf einen direkten Abruf zurück. Das Bildverständnis verwendet diesen Proxy-Pfad nicht.

## Fähigkeiten

Legen Sie `capabilities` in einem `models[]`-Eintrag fest, um ihn auf bestimmte Medientypen zu beschränken. Für gemeinsam genutzte Listen leitet OpenClaw die Standardwerte pro gebündeltem Provider ab:

| Provider                                                                 | Funktionen            |
| ------------------------------------------------------------------------ | --------------------- |
| `openai`, `anthropic`, `minimax`                                         | Bild                  |
| `minimax-portal`                                                         | Bild                  |
| `moonshot`                                                               | Bild + Video          |
| `openrouter`                                                             | Bild + Audio          |
| `google` (Gemini-API)                                                    | Bild + Audio + Video  |
| `qwen`                                                                   | Bild + Video          |
| `deepinfra`                                                              | Bild + Audio          |
| `mistral`                                                                | Audio                 |
| `zai`                                                                    | Bild                  |
| `groq`, `xai`, `deepgram`, `senseaudio`                                  | Audio                 |
| Jeder `models.providers.<id>.models[]`-Katalog mit einem bildfähigen Modell | Bild                  |

Legen Sie für CLI-Einträge `capabilities` explizit fest, um unerwartete Übereinstimmungen zu vermeiden; wird die Angabe ausgelassen, kommt der Eintrag für jede Funktionsliste infrage, in der er erscheint.

## Provider-Unterstützungsmatrix

| Funktion | Provider                                                                                                                                                | Hinweise                                                                                                                                                                                                 |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bild     | Anthropic, Codex app-server, Deepinfra, Google, MiniMax, MiniMax Portal, Moonshot, OpenAI, OpenAI Codex OAuth, OpenRouter, Qwen, Z.AI, Konfigurations-Provider | Provider-Plugins registrieren Bildunterstützung; `openai/*` kann API-Schlüssel- oder Codex-OAuth-Routing verwenden; `codex/*` verwendet einen begrenzten Codex-app-server-Durchlauf; bildfähige Konfigurations-Provider werden automatisch registriert. |
| Audio    | Deepgram, Deepinfra, ElevenLabs, Google, Groq, Mistral, OpenAI, OpenRouter, SenseAudio, xAI                                                             | Transkription durch den Provider (Whisper/Groq/xAI/Deepgram/OpenRouter STT/Gemini/SenseAudio/Scribe/Voxtral).                                                                                            |
| Video    | Google, Moonshot, Qwen                                                                                                                                  | Videoverständnis durch den Provider über Hersteller-Plugins; das Videoverständnis von Qwen verwendet die standardmäßigen DashScope-Endpunkte.                                                           |

<Note>
**Hinweis zu MiniMax**: Das Bildverständnis für `minimax`, `minimax-cn`, `minimax-portal` und `minimax-portal-cn` stammt immer vom Plugin-eigenen Medien-Provider `MiniMax-VL-01`, selbst wenn veraltete Chat-Metadaten von MiniMax M2.x Bildeingaben ausweisen.
</Note>

## Hinweise zur Modellauswahl

- Bevorzugen Sie für jede Medienfunktion das leistungsstärkste Modell der aktuellen Generation, wenn Qualität und Sicherheit wichtig sind.
- Vermeiden Sie bei Agenten mit Werkzeugzugriff, die nicht vertrauenswürdige Eingaben verarbeiten, ältere oder schwächere Medienmodelle.
- Halten Sie für die Verfügbarkeit mindestens ein Ausweichmodell pro Funktion bereit (Qualitätsmodell + schnelleres/günstigeres Modell).
- CLI-Ausweichlösungen (`whisper-cli`, `whisper`, `gemini`) helfen, wenn Provider-APIs nicht verfügbar sind.
- Bekannte Dateiausgabemodi sind maßgeblich: Eine leere oder fehlende abgeleitete Transkriptdatei erzeugt kein Transkript, statt auf die CLI-Fortschrittsausgabe zurückzugreifen.
- `parakeet-mlx`: Verwenden Sie `--output-format txt` (oder `all`) mit `--output-dir` und der standardmäßigen Ausgabevorlage `{filename}`. Die vorgelagerten Umgebungsvariablen `PARAKEET_OUTPUT_FORMAT` und `PARAKEET_OUTPUT_TEMPLATE` werden ebenfalls berücksichtigt. OpenClaw liest `<output-dir>/<media-basename>.txt`; das standardmäßige Format `srt`, andere Formate und benutzerdefinierte Ausgabevorlagen verwenden weiterhin stdout.

## Richtlinie für Anhänge

Die funktionsspezifische Einstellung `attachments` steuert, welche Anhänge verarbeitet werden:

<ParamField path="mode" type='"first" | "all"' default="first">
  Verarbeitet nur den ersten ausgewählten Anhang oder alle ausgewählten Anhänge.
</ParamField>
<ParamField path="maxAttachments" type="number" default="1">
  Begrenzt die Anzahl der verarbeiteten Anhänge.
</ParamField>
<ParamField path="prefer" type='"first" | "last" | "path" | "url"'>
  Auswahlpräferenz unter den infrage kommenden Anhängen.
</ParamField>

Bei `mode: "all"` werden Ausgaben mit `[Image 1/2]`, `[Audio 2/2]` usw. gekennzeichnet.

### Extraktion aus Dateianhängen

- Extrahierter Dateitext wird als nicht vertrauenswürdiger externer Inhalt umschlossen, bevor er an den Medien-Prompt angehängt wird. Dabei werden Begrenzungsmarkierungen wie `<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>` sowie eine Metadatenzeile `Source: External` verwendet.
- Dieser Pfad lässt das lange Banner `SECURITY NOTICE:` absichtlich aus, um den Medien-Prompt kurz zu halten; die Begrenzungsmarkierungen und Metadaten gelten weiterhin.
- Eine Datei ohne extrahierbaren Text erhält `[No extractable text]`.
- Wenn bei einer PDF-Datei ersatzweise gerenderte Seitenbilder verwendet werden, leitet OpenClaw diese Bilder an antwortende Modelle mit Bildverarbeitung weiter und behält den Platzhalter `[PDF content rendered to images]` im Dateiblock bei.

## Konfigurationsbeispiele

<Tabs>
  <Tab title="Gemeinsame Modelle + Überschreibungen">
    ```json5
    {
      tools: {
        media: {
          models: [
            { provider: "openai", model: "gpt-5.6-sol", capabilities: ["image"] },
            {
              provider: "google",
              model: "gemini-3-flash-preview",
              capabilities: ["image", "audio", "video"],
            },
            {
              type: "cli",
              command: "gemini",
              args: [
                "-m",
                "gemini-3-flash",
                "--allowed-tools",
                "read_file",
                "Lies die Mediendatei unter {{AttachmentPath}} und beschreibe sie in höchstens {{MaxChars}} Zeichen.",
              ],
              capabilities: ["image", "video"],
            },
          ],
          audio: {
            attachments: { mode: "all", maxAttachments: 2 },
          },
          video: {
            maxChars: 500,
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Nur Audio + Video">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [
              { provider: "openai", model: "gpt-4o-mini-transcribe" },
              {
                type: "cli",
                command: "whisper",
                args: ["--model", "base", "{{AttachmentPath}}"],
              },
            ],
          },
          video: {
            enabled: true,
            maxChars: 500,
            models: [
              { provider: "google", model: "gemini-3-flash-preview" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Lies die Mediendatei unter {{AttachmentPath}} und beschreibe sie in höchstens {{MaxChars}} Zeichen.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Nur Bild">
    ```json5
    {
      tools: {
        media: {
          image: {
            enabled: true,
            maxBytes: 10485760,
            maxChars: 500,
            models: [
              { provider: "openai", model: "gpt-5.6-sol" },
              { provider: "anthropic", model: "claude-opus-5" },
              {
                type: "cli",
                command: "gemini",
                args: [
                  "-m",
                  "gemini-3-flash",
                  "--allowed-tools",
                  "read_file",
                  "Lies die Mediendatei unter {{AttachmentPath}} und beschreibe sie in höchstens {{MaxChars}} Zeichen.",
                ],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Einzelner multimodaler Eintrag">
    ```json5
    {
      tools: {
        media: {
          image: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          audio: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
          video: {
            models: [
              {
                provider: "google",
                model: "gemini-3.1-pro-preview",
                capabilities: ["image", "video", "audio"],
              },
            ],
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Statusausgabe

Wenn die Medienanalyse ausgeführt wird, enthält `/status` eine Zusammenfassungszeile pro Funktion:

```
📎 Medien: Bild ok (openai/gpt-5.6-sol) · Audio ok (whisper-cli beobachtet=metal)
```

Führen Sie für die Vorab-Bestandsaufnahme `openclaw capability audio providers` aus. Lokale Zeilen zeigen die lokale Ausweichlösung getrennt von der globalen Provider-Auswahl, der Bereitschaft und den separaten Feldern für fähiges/angefordertes/beobachtetes Backend. Dieselbe lokale Auswahl ist als informative Doctor-Feststellung verfügbar:

```bash
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

## Hinweise

- Die Analyse erfolgt nach bestem Bemühen. Fehler blockieren keine Antworten.
- Anhänge werden auch dann an Modelle übergeben, wenn die Analyse deaktiviert ist.
- Verwenden Sie `scope`, um einzuschränken, wo die Analyse ausgeführt wird (beispielsweise nur in Direktnachrichten).

## Verwandte Themen

- [Konfiguration](/de/gateway/configuration)
- [Unterstützung für Bilder und Medien](/de/nodes/images)
