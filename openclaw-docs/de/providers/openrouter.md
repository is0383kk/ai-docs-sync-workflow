---
read_when:
    - Sie möchten einen einzigen API-Schlüssel für viele LLMs
    - Sie möchten Modelle über OpenRouter in OpenClaw ausführen
    - Sie möchten OpenRouter für die Bilderzeugung verwenden
    - Sie möchten OpenRouter für die Musikgenerierung verwenden
    - Sie möchten OpenRouter für die Videogenerierung verwenden
summary: Verwenden Sie die einheitliche API von OpenRouter, um in OpenClaw auf zahlreiche Modelle zuzugreifen
title: OpenRouter
x-i18n:
    generated_at: "2026-07-26T18:04:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter leitet Anfragen über eine API und einen Schlüssel an viele Modelle weiter. Es ist
OpenAI-kompatibel, sodass OpenClaw über denselben Transport im
`openai-completions`-Stil mit ihm kommuniziert, der auch für andere Proxy-Provider verwendet wird.

## Erste Schritte

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="OAuth-Onboarding ausführen">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw öffnet den browserbasierten Anmeldeablauf von OpenRouter (PKCE), tauscht den
        Code gegen einen OpenRouter-API-Schlüssel aus und speichert ihn im standardmäßigen
        OpenRouter-Authentifizierungsprofil. Auf entfernten/headless Hosts gibt OpenClaw die
        Anmelde-URL aus und fordert Sie auf, nach der Anmeldung die Weiterleitungs-URL einzufügen.
      </Step>
      <Step title="(Optional) Zu einem bestimmten Modell wechseln">
        Beim Onboarding wird standardmäßig `openrouter/auto` verwendet. Wählen Sie später ein konkretes Modell:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="API-Schlüssel">
    <Steps>
      <Step title="API-Schlüssel abrufen">
        Erstellen Sie unter [openrouter.ai/keys](https://openrouter.ai/keys) einen API-Schlüssel.
      </Step>
      <Step title="API-Schlüssel-Onboarding ausführen">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="(Optional) Zu einem bestimmten Modell wechseln">
        Beim Onboarding wird standardmäßig `openrouter/auto` verwendet. Wählen Sie später ein konkretes Modell:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## Konfigurationsbeispiel

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## Modellreferenzen

<Note>
Modellreferenzen folgen dem Muster `openrouter/<provider>/<model>`. Die vollständige Liste der
verfügbaren Provider und Modelle finden Sie unter [/concepts/model-providers](/de/concepts/model-providers).
</Note>

Mitgelieferte Fallback-Modelle, die verwendet werden, wenn die Live-Katalogerkennung nicht verfügbar ist:

| Modellreferenz                    | Hinweise                     |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | Automatisches OpenRouter-Routing |
| `openrouter/moonshotai/kimi-k2.6` | Kimi K2.6 über MoonshotAI     |
| `openrouter/moonshotai/kimi-k2.5` | Kimi K2.5 über MoonshotAI     |

Jede andere `openrouter/<provider>/<model>`-Referenz, einschließlich
`openrouter/openrouter/fusion` (siehe [Fusion-Router](#fusion-router)), wird
dynamisch anhand des Live-Modellkatalogs von OpenRouter aufgelöst.

## Bilderzeugung

OpenRouter kann das Tool `image_generate` bereitstellen. Legen Sie ein OpenRouter-Bildmodell
unter `agents.defaults.mediaModels.image` fest:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw sendet Bildanfragen mit `modalities: ["image", "text"]` an die Chat-Completions-Bild-API
von OpenRouter. Gemini-Bildmodelle erhalten zusätzlich die Hinweise
`aspectRatio` und `resolution` über `image_config` von OpenRouter; andere
Bildmodelle erhalten sie nicht. Verwenden Sie `agents.defaults.mediaModels.image.timeoutMs` für
langsamere Modelle; der pro Aufruf gesetzte Wert `timeoutMs` des Tools `image_generate` hat weiterhin Vorrang.

## Videoerzeugung

OpenRouter kann das Tool `video_generate` über seine asynchrone
`/videos`-API bereitstellen. Legen Sie ein OpenRouter-Videomodell unter
`agents.defaults.mediaModels.video` fest:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw übermittelt Text-zu-Video- und Bild-zu-Video-Aufträge, fragt den zurückgegebenen
`polling_url` ab und lädt das fertige Video von `unsigned_urls` von OpenRouter
oder vom Inhaltsendpunkt des Auftrags herunter. Referenzbilder werden standardmäßig als
Bilder des ersten/letzten Frames verwendet; mit `reference_image` markierte Bilder werden stattdessen als
Eingabereferenzen gesendet. Der mitgelieferte Standard `google/veo-3.1-fast` unterstützt Dauern von 4/6/8
Sekunden, Auflösungen von `720P`/`1080P` und Seitenverhältnisse von `16:9`/`9:16`.
Video-zu-Video wird nicht unterstützt: Die vorgelagerte API akzeptiert nur Text- und
Bildreferenzen.

## Musikerzeugung

OpenRouter kann das Tool `music_generate` über die Audioausgabe von
Chat-Completions bereitstellen. Legen Sie ein OpenRouter-Audiomodell unter
`agents.defaults.mediaModels.music` fest:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

Der mitgelieferte OpenRouter-Musik-Provider verwendet standardmäßig `google/lyria-3-pro-preview`
und stellt außerdem `google/lyria-3-clip-preview` bereit. OpenClaw sendet `modalities:
["text", "audio"]`, streamt die Antwort, sammelt die Audiosegmente und speichert
das Ergebnis als erzeugtes Medium für die Kanalauslieferung. Lyria-Modelle akzeptieren ein
Referenzbild über den gemeinsamen Parameter `music_generate image=...`.
Streaming-Audio, die Aufbewahrung von Transkripten und die abgeleitete SSE-Ereignishülle werden
durch `agents.defaults.mediaMaxMb` begrenzt (das standardmäßige Audiolimit beträgt 16 MB).

## Text-to-Speech

OpenRouter kann über seinen OpenAI-kompatiblen
`/audio/speech`-Endpunkt als TTS-Provider fungieren.

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

Wenn `tts.providers.openrouter.apiKey` weggelassen wird, greift TTS zunächst auf
`models.providers.openrouter.apiKey` und anschließend auf `OPENROUTER_API_KEY` zurück.

## Speech-to-Text (eingehendes Audio)

OpenRouter kann eingehende Sprach-/Audioanhänge über den gemeinsamen
Pfad `tools.media.audio` mithilfe seines STT-Endpunkts (`/audio/transcriptions`) transkribieren.
Dies gilt für jedes Kanal-Plugin, das eingehende Sprache/Audio an die
Vorprüfung für das Medienverständnis weiterleitet.

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw sendet OpenRouter-STT-Anfragen gemäß dem STT-Vertrag von OpenRouter als JSON mit
Base64-Audio unter `input_audio` und nicht als mehrteilige OpenAI-Formular-
Uploads.

## Fusion-Router

OpenRouter Fusion sendet eine OpenClaw-Modellreferenz parallel an mehrere OpenRouter-Modelle,
lässt OpenRouter deren Antworten bewerten und gibt über den normalen OpenRouter-Endpunkt
eine endgültige Antwort zurück. Der vorgelagerte Modell-Slug lautet
`openrouter/fusion`, daher enthält die OpenClaw-Modellreferenz sowohl das OpenClaw-
Provider-Präfix als auch den vorgelagerten OpenRouter-Namensraum:

```bash
openclaw models set openrouter/openrouter/fusion
```

Konfigurieren Sie das Panel und das Bewertungsmodell von Fusion über `params.extraBody` des Modells;
diese Felder werden direkt an den Textkörper der Chat-Completions-Anfrage von OpenRouter
weitergeleitet. Fusion funktioniert sowohl mit OAuth- als auch mit API-Schlüssel-Onboarding. Wenn Sie OAuth verwenden,
lassen Sie die folgende Zeile `env.OPENROUTER_API_KEY` weg.

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` ist das parallele Panel; `model` in der Konfiguration des Fusion-Plugins
ist das Bewertungsmodell. Setzen Sie `tool_choice` auf oberster Ebene bei normalen
Agenten-/Chat-Durchläufen nicht auf `"required"`, um Fusion zu erzwingen: OpenClaw-Durchläufe können
eigene Tooldefinitionen enthalten, und eine auf oberster Ebene vorgeschriebene Toolauswahl könnte eines davon
anstelle des Fusion-Routers auswählen. Wenn diese Fusion-Plugin-Konfiguration vorhanden ist,
fügt OpenClaw einen bereinigten System-Prompt-Hinweis hinzu, der die konfigurierten Analysemodelle
und das Bewertungsmodell auflistet, sodass der Agent Fragen zu seinem eigenen Fusion-
Panel beantworten kann. Andere `extraBody`-Felder werden nicht in den Prompt kopiert.

Fusion ist absichtlich langsamer: OpenRouter verteilt den Prompt auf mehrere
Analysemodelle und führt anschließend einen Bewertungs-/Syntheseschritt aus, sodass die Latenz höher ist als
bei einer direkten Anfrage an ein einzelnes Modell. Verwenden Sie Fusion für wohlüberlegte, hochwertige Antworten oder
Eskalationspfade und nicht als latenzempfindlichen Standard. Halten Sie das Panel klein und
wählen Sie für schnellere Antworten schnellere Analyse-/Bewertungsmodelle.

Testen Sie eine konfigurierte Referenz mit einem einmaligen lokalen Aufruf:

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "Reply with exactly: FUSION_OK" \
  --json
```

## Authentifizierung und Header

OpenRouter verwendet ein Bearer-Token aus Ihrem API-Schlüssel. OpenRouter OAuth ist ein PKCE-
Anmeldeablauf, der einen OpenRouter-API-Schlüssel ausstellt. Daher speichert OpenClaw das Ergebnis im
selben API-Schlüssel-Authentifizierungsprofil `openrouter:default`, das bei der manuellen Einrichtung eines API-Schlüssels
verwendet wird.

So melden Sie sich bei einer vorhandenen Installation an oder wechseln den gespeicherten Schlüssel, ohne das
vollständige Onboarding erneut auszuführen:

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

Bei verifizierten OpenRouter-Anfragen (`https://openrouter.ai/api/v1`) fügt OpenClaw
die dokumentierten App-Zuordnungs-Header von OpenRouter hinzu:

| Header                    | Wert                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`            | `https://openclaw.ai`                                                                                  |
| `X-OpenRouter-Title`      | `OpenClaw`                                                                                             |
| `X-OpenRouter-Categories` | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent` |

<Warning>
Wenn Sie den OpenRouter-Provider auf einen anderen Proxy oder eine andere Basis-URL umstellen, fügt OpenClaw
diese OpenRouter-spezifischen Header oder Anthropic-Cache-Markierungen **nicht** ein.
</Warning>

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Antwort-Caching">
    Das Antwort-Caching von OpenRouter ist optional. Aktivieren Sie es pro Modell:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw sendet `X-OpenRouter-Cache: true` und, sofern konfiguriert,
    `X-OpenRouter-Cache-TTL`. `responseCacheClear: true` erzwingt eine Aktualisierung für
    die aktuelle Anfrage und speichert die Ersatzantwort. Snake-Case-
    Aliasse (`response_cache`, `response_cache_ttl_seconds`,
    `response_cache_clear`) werden ebenso akzeptiert wie `responseCacheTtl` /
    `response_cache_ttl` ohne das Suffix `Seconds`.

    Dies ist vom Prompt-Caching des Providers und von den Anthropic-
    Markierungen `cache_control` von OpenRouter getrennt. Es gilt nur für verifizierte
    `openrouter.ai`-Routen, nicht für benutzerdefinierte Proxy-Basis-URLs.

  </Accordion>

  <Accordion title="Anthropic-Cache-Markierungen">
    Auf verifizierten OpenRouter-Routen behalten Anthropic-Modellreferenzen die Anthropic-
    Markierungen `cache_control` von OpenRouter bei, um die Wiederverwendung des Prompt-Caches für
    System-/Entwickler-Prompt-Blöcke zu verbessern.
  </Accordion>

  <Accordion title="Anthropic-Reasoning-Prefill">
    Auf verifizierten OpenRouter-Routen entfernen Anthropic-Modellreferenzen mit aktiviertem Reasoning
    abschließende Assistant-Prefill-Durchläufe, bevor die Anfrage
    OpenRouter erreicht. Dies entspricht der Anforderung von Anthropic, dass Reasoning-Konversationen
    mit einem Benutzerdurchlauf enden.
  </Accordion>

  <Accordion title="Einfügung von Denken / Schlussfolgern">
    Auf unterstützten Routen, die nicht `auto` sind, ordnet OpenClaw die ausgewählte Denkstufe
    den Reasoning-Payloads des OpenRouter-Proxys zu. `openrouter/auto` und nicht unterstützte
    Modellhinweise überspringen diese Einfügung. Veraltete `openrouter/hunter-alpha`-Referenzen
    überspringen sie ebenfalls, da OpenRouter auf dieser eingestellten Route den Text der endgültigen Antwort
    in Reasoning-Feldern zurückgeben könnte.
  </Accordion>

  <Accordion title="Wiedergabe des DeepSeek-V4-Reasonings">
    Auf verifizierten OpenRouter-Routen ergänzen `openrouter/deepseek/deepseek-v4-flash` und
    `openrouter/deepseek/deepseek-v4-pro` fehlende `reasoning_content` bei
    wiedergegebenen Assistenten-Turns, sodass Denk-/Tool-Unterhaltungen die für nachfolgende
    Nachrichten erforderliche Form von DeepSeek V4 beibehalten. OpenClaw sendet für diese Routen
    von OpenRouter unterstützte `reasoning.effort`-Werte: `xhigh`/`max` werden `xhigh` zugeordnet,
    jede andere nicht deaktivierte Stufe wird `high` zugeordnet.
  </Accordion>

  <Accordion title="Nur für OpenAI geltende Anfrageaufbereitung">
    OpenRouter wird über den Proxy-ähnlichen, OpenAI-kompatiblen Pfad ausgeführt, daher werden native,
    ausschließlich für OpenAI geltende Anfrageaufbereitungen wie `serviceTier`, Responses `store`,
    OpenAI-Payloads zur Reasoning-Kompatibilität und Hinweise für den Prompt-Cache nicht weitergeleitet.
  </Accordion>

  <Accordion title="Gemini-gestützte Routen">
    Gemini-gestützte OpenRouter-Referenzen verbleiben auf dem Proxy-Gemini-Pfad: OpenClaw behält
    dort die Bereinigung der Gemini-Denksignaturen bei, aktiviert jedoch weder die native
    Gemini-Wiedergabevalidierung noch Bootstrap-Umschreibungen.
  </Accordion>

  <Accordion title="Metadaten für das Provider-Routing">
    OpenRouter unterstützt ein `provider`-Anfrageobjekt für das Routing des zugrunde liegenden Providers.
    Konfigurieren Sie mit `models.providers.openrouter.params.provider` eine Standardrichtlinie für alle
    OpenRouter-Textmodellanfragen:

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw leitet dieses Objekt als Anfrage-Payload `provider`
    an OpenRouter weiter. Verwenden Sie die dokumentierten snake_case-Felder von OpenRouter, darunter `sort`,
    `only`, `ignore`, `order`, `allow_fallbacks`, `require_parameters`,
    `data_collection`, `quantizations`, `max_price`, `preferred_max_latency`,
    `preferred_min_throughput`, `zdr` und `enforce_distillable_text`.

    Modellspezifische Parameter überschreiben das Provider-weite Routingobjekt:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    Dies gilt nur für Chat-Completions-Routen von OpenRouter. Direkte Routen für Anthropic,
    Google, OpenAI oder benutzerdefinierte Provider ignorieren die Routingparameter von OpenRouter.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Modellauswahl" href="/de/concepts/model-providers" icon="layers">
    Auswahl von Providern, Modellreferenzen und Failover-Verhalten.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/configuration-reference" icon="gear">
    Vollständige Konfigurationsreferenz für Agenten, Modelle und Provider.
  </Card>
</CardGroup>
