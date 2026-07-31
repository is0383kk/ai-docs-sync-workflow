---
read_when:
    - Sie möchten Google-Gemini-Modelle mit OpenClaw verwenden
    - Sie benötigen den API-Schlüssel oder den OAuth-Authentifizierungsablauf
summary: Google-Gemini-Einrichtung (API-Schlüssel + OAuth, Bildgenerierung, Medienverständnis, TTS, Websuche)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-07-26T18:43:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf8db70bcebd425238e5f02ca12bdbcd75fa1c03d285ea127d4e3863892b3aa
    source_path: providers/google.md
    workflow: 16
---

Das Google-Plugin bietet Zugriff auf Gemini-Modelle über Google AI Studio sowie Bilderzeugung, Medienverständnis (Bild/Audio/Video), Text-to-Speech und Websuche über Gemini Grounding.

- Provider: `google`
- Authentifizierung: `GEMINI_API_KEY` oder `GOOGLE_API_KEY`
- API: Google Gemini API
- Laufzeitoption: `agentRuntime.id: "google-gemini-cli"` verwendet Gemini CLI OAuth wieder, während Modellreferenzen weiterhin kanonisch als `google/*` angegeben werden.

## Erste Schritte

Wählen Sie Ihre bevorzugte Authentifizierungsmethode und führen Sie die Einrichtungsschritte aus.

<Tabs>
  <Tab title="API-Schlüssel">
    **Am besten geeignet für:** standardmäßigen Zugriff auf die Gemini API über Google AI Studio.

    <Steps>
      <Step title="API-Schlüssel abrufen">
        Erstellen Sie in [Google AI Studio](https://aistudio.google.com/apikey) einen kostenlosen Schlüssel.
      </Step>
      <Step title="Onboarding ausführen">
        ```bash
        openclaw onboard --auth-choice gemini-api-key
        ```

        Alternativ können Sie den Schlüssel direkt übergeben:

        ```bash
        openclaw onboard --non-interactive \
          --mode local \
          --auth-choice gemini-api-key \
          --gemini-api-key "$GEMINI_API_KEY"
        ```
      </Step>
      <Step title="Standardmodell festlegen">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "google/gemini-3.1-pro-preview" },
            },
          },
        }
        ```
      </Step>
      <Step title="Verfügbarkeit des Modells überprüfen">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    <Tip>
    `GEMINI_API_KEY` und `GOOGLE_API_KEY` werden beide akzeptiert. Verwenden Sie die bereits konfigurierte Variante.
    </Tip>

    Mit einem konfigurierten API-Schlüssel aktualisiert OpenClaw den Textmodellkatalog
    von Google AI Studio über die Gemini-API `models.list`. Neu veröffentlichte Varianten von Gemini 3 Pro, Flash
    und Flash-Lite erscheinen daher in
    `openclaw models list --provider google`, ohne dass auf eine OpenClaw-Version
    gewartet werden muss. Wenn die Erkennung nicht verfügbar ist, behält OpenClaw den mitgelieferten Ausweichkatalog
    bei.

  </Tab>

  <Tab title="Gemini CLI (OAuth)">
    **Am besten geeignet für:** die Anmeldung mit Ihrem Google-Konto über Gemini CLI OAuth, anstatt einen separaten API-Schlüssel zu verwenden.

    <Warning>
    Der Provider `google-gemini-cli` ist eine inoffizielle Integration. Einige Benutzer
    berichten von Kontoeinschränkungen bei dieser Art der OAuth-Nutzung. Die Verwendung erfolgt auf eigenes Risiko.
    </Warning>

    <Steps>
      <Step title="Gemini CLI installieren">
        Der lokale Befehl `gemini` muss unter `PATH` verfügbar sein.

        ```bash
        # Homebrew
        brew install gemini-cli

        # or npm
        npm install -g @google/gemini-cli
        ```

        OpenClaw unterstützt sowohl Homebrew-Installationen als auch globale npm-Installationen, einschließlich
        gängiger Windows-/npm-Verzeichnisstrukturen.
      </Step>
      <Step title="Über OAuth anmelden">
        ```bash
        openclaw models auth login --provider google-gemini-cli --set-default
        ```
      </Step>
      <Step title="Verfügbarkeit des Modells überprüfen">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    - Standardmodell: `google/gemini-3.1-pro-preview`
    - Laufzeit: `google-gemini-cli`
    - Alias: `gemini-cli`

    Die Gemini-API-Modell-ID von Gemini 3.1 Pro lautet `gemini-3.1-pro-preview`. OpenClaw akzeptiert die kürzere Form `google/gemini-3.1-pro` als praktischen Alias und normalisiert sie vor Provider-Aufrufen.

    **Umgebungsvariablen:**

    - `OPENCLAW_GEMINI_OAUTH_CLIENT_ID` / `GEMINI_CLI_OAUTH_CLIENT_ID`
    - `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET` / `GEMINI_CLI_OAUTH_CLIENT_SECRET`

    <Note>
    Wenn Gemini CLI OAuth-Anfragen nach der Anmeldung fehlschlagen, legen Sie `GOOGLE_CLOUD_PROJECT` oder
    `GOOGLE_CLOUD_PROJECT_ID` auf dem Gateway-Host fest und versuchen Sie es erneut.
    </Note>

    <Note>
    Wenn die Anmeldung fehlschlägt, bevor der Browserablauf beginnt, stellen Sie sicher, dass der lokale Befehl `gemini`
    installiert und unter `PATH` verfügbar ist.
    </Note>

    Die automatische Onboarding-Erkennung führt eine vorhandene Gemini-CLI-Anmeldung auf, testet sie jedoch
    nie automatisch, da Gemini CLI keine werkzeugfreie Prüfung bietet. Wählen Sie Gemini CLI
    OAuth oder einen Gemini-API-Schlüssel aus, um fortzufahren.

    Modellreferenzen vom Typ `google-gemini-cli/*` sind Aliasse für die Abwärtskompatibilität. Neue
    Konfigurationen sollten Modellreferenzen vom Typ `google/*` zusammen mit der Laufzeit `google-gemini-cli`
    verwenden, wenn sie Gemini CLI lokal ausführen möchten.

  </Tab>
</Tabs>

<Note>
`google/gemini-3-pro-preview` wurde am 2026-03-09 eingestellt; verwenden Sie stattdessen `google/gemini-3.1-pro-preview`. Durch erneutes Ausführen der Einrichtung des Gemini-API-Schlüssels (`openclaw onboard --auth-choice gemini-api-key` oder `openclaw models auth login --provider google`) wird ein veraltetes konfiguriertes Standardmodell auf das aktuelle Modell umgestellt.
</Note>

## Funktionen

| Funktion                 | Unterstützt                  |
| ------------------------ | ---------------------------- |
| Chat-Vervollständigungen | Ja                           |
| Bilderzeugung            | Ja                           |
| Musikerzeugung           | Ja                           |
| Text-to-Speech           | Ja                           |
| Echtzeit-Sprache         | Ja (Google Live API)         |
| Bildverständnis          | Ja                           |
| Audiotranskription       | Ja                           |
| Videoverständnis         | Ja                           |
| Websuche (Grounding)     | Ja                           |
| Denken/Schlussfolgern    | Ja (Gemini 2.5+ / Gemini 3+) |
| Gemma-4-Modelle          | Ja                           |

## Websuche

Der mitgelieferte Websuch-Provider `gemini` verwendet Gemini Google Search Grounding.
Konfigurieren Sie unter `plugins.entries.google.config.webSearch` einen eigenen Suchschlüssel,
oder lassen Sie ihn nach `GEMINI_API_KEY` den Wert `models.providers.google.apiKey` wiederverwenden:

```json5
{
  plugins: {
    entries: {
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // optional if GEMINI_API_KEY or models.providers.google.apiKey is set
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // falls back to models.providers.google.baseUrl
            model: "gemini-2.5-flash",
          },
        },
      },
    },
  },
}
```

Bei den Anmeldedaten hat zunächst der dedizierte Wert `webSearch.apiKey` Vorrang, gefolgt von `GEMINI_API_KEY`
und anschließend `models.providers.google.apiKey`. `webSearch.baseUrl` ist optional und
für Betreiber-Proxys oder kompatible Gemini-API-Endpunkte vorgesehen; wenn der Wert nicht angegeben wird,
verwendet die Gemini-Websuche `models.providers.google.baseUrl` wieder. Informationen zum providerspezifischen Werkzeugverhalten finden Sie unter
[Gemini-Suche](/de/tools/gemini-search).

<Tip>
Gemini-3-Modelle verwenden `thinkingLevel` anstelle von `thinkingBudget`. OpenClaw ordnet
die Steuerung des Schlussfolgerns für Gemini 3, Gemini 3.1 und den Alias `gemini-*-latest`
`thinkingLevel` zu, damit standardmäßige Ausführungen und Ausführungen mit niedriger Latenz keine deaktivierten
Werte für `thinkingBudget` senden.

`/think adaptive` behält Googles dynamische Denksemantik bei, anstatt
eine feste OpenClaw-Stufe auszuwählen. Gemini 3 und Gemini 3.1 lassen einen festen Wert für `thinkingLevel` aus, damit
Google die Stufe auswählen kann; Gemini 2.5 sendet Googles dynamischen Sentinel-Wert
`thinkingBudget: -1`.

Gemma-4-Modelle (beispielsweise `gemma-4-26b-a4b-it`) unterstützen den Denkmodus. OpenClaw
schreibt `thinkingBudget` für Gemma 4 in einen unterstützten Google-Wert für `thinkingLevel` um.
Wenn das Denken auf `off` festgelegt wird, bleibt es deaktiviert, anstatt auf
`MINIMAL` abgebildet zu werden.

Gemini 2.5 Pro funktioniert nur im Denkmodus und lehnt einen expliziten Wert
für `thinkingBudget: 0` ab; OpenClaw entfernt diesen Wert bei Anfragen an Gemini 2.5 Pro,
anstatt ihn zu senden.
</Tip>

## Bilderzeugung

Der mitgelieferte Bilderzeugungs-Provider `google` verwendet standardmäßig
`google/gemini-3.1-flash-image`.

- Unterstützt außerdem `google/gemini-3-pro-image`
- Erzeugung: bis zu 4 Bilder pro Anfrage
- Bearbeitungsmodus: aktiviert, bis zu 5 Eingabebilder
- Geometriesteuerung: `size`, `aspectRatio` und `resolution`

So verwenden Sie Google als standardmäßigen Bilderzeugungs-Provider:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image",
      },
    },
  },
}
```

<Note>
Unter [Bilderzeugung](/de/tools/image-generation) finden Sie Informationen zu gemeinsamen Werkzeugparametern, zur Provider-Auswahl und zum Failover-Verhalten.
</Note>

## Videoerzeugung

Das mitgelieferte Plugin `google` registriert außerdem die Videoerzeugung über das gemeinsame
Werkzeug `video_generate`.

- Standard-Videomodell: `google/veo-3.1-fast-generate-preview`
- Modi: Text-zu-Video, Bild-zu-Video und Abläufe mit einer einzelnen Videoreferenz
- Unterstützt `aspectRatio` (`16:9`, `9:16`) und `resolution` (`720P`, `1080P`); die Audioausgabe wird derzeit von Veo nicht unterstützt
- Unterstützte Dauern: **4, 6 oder 8 Sekunden** (andere Werte werden auf den nächstgelegenen zulässigen Wert gesetzt)

So verwenden Sie Google als standardmäßigen Video-Provider:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

<Note>
Unter [Videoerzeugung](/de/tools/video-generation) finden Sie Informationen zu gemeinsamen Werkzeugparametern, zur Provider-Auswahl und zum Failover-Verhalten.
</Note>

## Musikerzeugung

Das mitgelieferte Plugin `google` registriert außerdem die Musikerzeugung über das gemeinsame
Werkzeug `music_generate`.

- Standard-Musikmodell: `google/lyria-3-clip-preview`
- Unterstützt außerdem `google/lyria-3-pro-preview`
- Prompt-Steuerung: `lyrics` und `instrumental`
- Ausgabeformat: standardmäßig `mp3`, zusätzlich `wav` unter `google/lyria-3-pro-preview`
- Referenzeingaben: bis zu 10 Bilder
- Sitzungsgestützte Ausführungen werden über den gemeinsamen Aufgaben-/Statusablauf abgekoppelt, einschließlich `action: "status"`

So verwenden Sie Google als standardmäßigen Musik-Provider:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

<Note>
Unter [Musikerzeugung](/de/tools/music-generation) finden Sie Informationen zu gemeinsamen Werkzeugparametern, zur Provider-Auswahl und zum Failover-Verhalten.
</Note>

## Text-to-Speech

Der mitgelieferte Sprach-Provider `google` verwendet den TTS-Pfad der Gemini API mit
`gemini-3.1-flash-tts-preview`.

- Standardstimme: `Kore`
- Authentifizierung: `tts.providers.google.apiKey`, `models.providers.google.apiKey`, `GEMINI_API_KEY` oder `GOOGLE_API_KEY`
- Ausgabe: WAV für reguläre TTS-Anhänge, Opus für Sprachnachrichtenziele, PCM für Talk/Telefonie
- Sprachnachrichtenausgabe: Google PCM wird als WAV verpackt und mit `ffmpeg` in Opus mit 48 kHz transkodiert

Googles Batch-Pfad für Gemini TTS gibt das erzeugte Audio in der abgeschlossenen
Antwort `generateContent` zurück. Verwenden Sie für gesprochene Unterhaltungen mit niedrigster Latenz den
Echtzeit-Sprach-Provider von Google auf Basis der Gemini Live API anstelle von Batch-
TTS.

So verwenden Sie Google als standardmäßigen TTS-Provider:

```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        audioProfile: "Speak professionally with a calm tone.",
      },
    },
  },
}
```

Gemini API TTS verwendet natürlichsprachliche Prompts zur Stilsteuerung. Legen Sie
`audioProfile` fest, um dem gesprochenen Text einen wiederverwendbaren Stil-Prompt voranzustellen. Legen Sie
`speakerName` fest, wenn sich Ihr Prompt-Text auf einen benannten Sprecher bezieht.

Gemini API TTS akzeptiert im Text außerdem ausdrucksbezogene, in eckige Klammern gesetzte Audio-Tags,
beispielsweise `[whispers]` oder `[laughs]`. Um Tags aus der sichtbaren Chat-Antwort
herauszuhalten und dennoch an TTS zu senden, platzieren Sie sie in einem
Block vom Typ `[[tts:text]]...[[/tts:text]]`:

```text
Hier ist der bereinigte Antworttext.

[[tts:text]][whispers] Hier ist die gesprochene Version.[[/tts:text]]
```

<Note>
Ein auf die Gemini API beschränkter API-Schlüssel aus der Google Cloud Console ist für diesen
Provider gültig. Dies ist nicht der separate API-Pfad von Cloud Text-to-Speech.
</Note>

## Echtzeit-Sprache

Das mitgelieferte Plugin `google` registriert einen Echtzeit-Sprach-Provider auf Basis der
Gemini Live API für backendseitige Audiobrücken wie Voice Call und Google Meet.

| Einstellung                  | Konfigurationspfad                                                   | Standardwert                                                                                 |
| ---------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Modell                       | `plugins.entries.voice-call.config.realtime.providers.google.model` | `gemini-3.1-flash-live-preview`                                                          |
| Stimme                       | `...google.voice`                                                   | `Kore`                                                                           |
| Temperatur                   | `...google.temperature`                                                   | (nicht festgelegt)                                                                           |
| VAD-Startempfindlichkeit     | `...google.startSensitivity`                                                   | (nicht festgelegt)                                                                           |
| VAD-Endempfindlichkeit       | `...google.endSensitivity`                                                   | (nicht festgelegt)                                                                           |
| Stilledauer                  | `...google.silenceDurationMs`                                                   | (nicht festgelegt)                                                                           |
| Aktivitätsverarbeitung       | `...google.activityHandling`                                                   | Google-Standardwert, `start-of-activity-interrupts`                                                       |
| Turn-Abdeckung               | `...google.turnCoverage`                                                   | Google-Standardwert, `audio-activity-and-all-video`                                                       |
| Automatische VAD deaktivieren | `...google.automaticActivityDetectionDisabled`                                                  | `false`                                                                           |
| Sitzungsfortsetzung          | `...google.sessionResumption`                                                   | `true`                                                                           |
| Kontextkomprimierung         | `...google.contextWindowCompression`                                                   | `true`                                                                           |
| API-Schlüssel                | `...google.apiKey`                                                   | Fällt auf `models.providers.google.apiKey`, `GEMINI_API_KEY` oder `GOOGLE_API_KEY` zurück |

Beispielkonfiguration für Voice Call in Echtzeit:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          realtime: {
            enabled: true,
            provider: "google",
            providers: {
              google: {
                model: "gemini-3.1-flash-live-preview",
                speakerVoice: "Kore",
                activityHandling: "start-of-activity-interrupts",
                turnCoverage: "audio-activity-and-all-video",
              },
            },
          },
        },
      },
    },
  },
}
```

<Note>
Die Google Live API verwendet bidirektionales Audio und Funktionsaufrufe über einen WebSocket.
OpenClaw passt das Audio der Telefonie-/Meet-Bridge an den PCM-Live-API-Stream von Gemini an und
belässt Tool-Aufrufe im gemeinsamen Echtzeit-Sprachvertrag. Lassen Sie `temperature`
nicht festgelegt, sofern Sie keine Änderungen am Sampling benötigen. OpenClaw lässt nicht positive Werte
weg, da Google Live für `temperature: 0` Transkripte ohne Audio zurückgeben kann.
Die Transkription der Gemini API wird ohne `languageCodes` aktiviert; das aktuelle Google
SDK lehnt Hinweise zum Sprachcode auf diesem API-Pfad ab.
</Note>

<Note>
Gemini 3.1 Live akzeptiert Konversationstext über Echtzeiteingaben und verwendet
sequenzielle Funktionsaufrufe. OpenClaw lässt für dieses Modell die älteren Felder
`NON_BLOCKING`, die Zeitplanung von Funktionsantworten und affektive Dialogfelder weg. Bevorzugen Sie
`thinkingLevel`; konfigurierte positive Werte für `thinkingBudget` werden der
nächstgelegenen unterstützten Stufe zugeordnet, während `-1` den Google-Standardwert beibehält. Siehe den
[Vergleich der Funktionen von Gemini Live](https://ai.google.dev/gemini-api/docs/live-api/capabilities).
</Note>

<Note>
Control UI Talk unterstützt Google-Live-Browsersitzungen mit eingeschränkten Einmal-
Tokens. In Video Talk sendet der Browser begrenzte JPEG-Frames direkt an
Google Live, mit dem Maximum des Providers von einem Frame pro Sekunde. Die Funktion
`describe_view` meldet, ob dieser Kamerastream aktiv ist.
Kameraframes durchlaufen nicht den Gateway. Nur im Backend ausgeführte Echtzeit-Sprach-
Provider können auch über den generischen Gateway-Relay-Transport ausgeführt werden, der
die Provider-Anmeldedaten auf dem Gateway hält.
</Note>

Führen Sie für die Live-Verifizierung durch Maintainer
`OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` aus.
Der Smoke-Test deckt außerdem die OpenAI-Backend-/WebRTC-Pfade ab; der Google-Teil stellt denselben
eingeschränkten Live-API-Token-Typ aus, den Control UI Talk verwendet, öffnet den Browser-
WebSocket-Endpunkt, sendet die anfängliche Setup-Nutzlast zusammen mit einem JPEG-Frame und
überprüft eine Textantwort sowie den Funktions-Roundtrip von `describe_view`.

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Direkte Wiederverwendung des Gemini-Caches">
    Bei direkten Gemini-API-Ausführungen (`api: "google-generative-ai"`) übergibt OpenClaw
    ein konfiguriertes `cachedContent`-Handle an Gemini-Anfragen.

    - Konfigurieren Sie modellspezifische oder globale Parameter entweder mit
      `cachedContent` oder dem veralteten `cached_content`
    - Parameter aus einem spezifischeren Gültigkeitsbereich (Modellebene vor globaler Ebene) haben stets Vorrang.
      Wenn beide Schlüssel innerhalb desselben Gültigkeitsbereichs festgelegt sind, hat `cached_content` Vorrang.
      Verwenden Sie nur einen Schlüssel pro Gültigkeitsbereich, um Überraschungen zu vermeiden.
    - Beispielwert: `cachedContents/prebuilt-context`
    - Die Nutzung bei einem Gemini-Cache-Treffer wird aus dem vorgelagerten
      `cachedContentTokenCount` in OpenClaw `cacheRead` normalisiert

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "google/gemini-2.5-pro": {
              params: {
                cachedContent: "cachedContents/prebuilt-context",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Hinweise zur Verwendung der Gemini CLI">
    Bei Verwendung des OAuth-Providers `google-gemini-cli` verwendet OpenClaw standardmäßig die
    Ausgabe `stream-json` der Gemini CLI und normalisiert die Nutzung aus der abschließenden
    `stats`-Nutzlast. Veraltete Überschreibungen von `--output-format json` verwenden weiterhin den
    JSON-Parser.

    - Der gestreamte Antworttext stammt aus den `message`-Ereignissen des Assistenten.
    - Bei veralteter JSON-Ausgabe stammt der Antworttext aus dem Feld `response` des CLI-JSON.
    - Die Nutzung fällt auf `stats` zurück, wenn die CLI `usage` leer lässt.
    - `stats.cached` wird in OpenClaw `cacheRead` normalisiert.
    - Wenn `stats.input` fehlt, leitet OpenClaw die Eingabe-Tokens aus
      `stats.input_tokens - stats.cached` ab.

  </Accordion>

  <Accordion title="Einrichtung von Umgebung und Daemon">
    Wenn der Gateway als Daemon (launchd/systemd) ausgeführt wird, stellen Sie sicher, dass `GEMINI_API_KEY`
    für diesen Prozess verfügbar ist (beispielsweise in `~/.openclaw/.env` oder über
    `env.shellEnv`).
  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="Bilderzeugung" href="/de/tools/image-generation" icon="image">
    Gemeinsame Parameter des Bild-Tools und Provider-Auswahl.
  </Card>
  <Card title="Videoerzeugung" href="/de/tools/video-generation" icon="video">
    Gemeinsame Parameter des Video-Tools und Provider-Auswahl.
  </Card>
  <Card title="Musikerzeugung" href="/de/tools/music-generation" icon="music">
    Gemeinsame Parameter des Musik-Tools und Provider-Auswahl.
  </Card>
</CardGroup>
