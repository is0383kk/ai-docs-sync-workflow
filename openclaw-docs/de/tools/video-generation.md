---
read_when:
    - Videos über den Agenten generieren
    - Konfigurieren von Providern und Modellen für die Videogenerierung
    - Die Parameter des Tools video_generate verstehen
sidebarTitle: Video generation
summary: Videos über video_generate aus Text-, Bild- oder Videoreferenzen mit 16 Provider-Backends generieren
title: Videogenerierung
x-i18n:
    generated_at: "2026-07-26T19:17:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4afc9338fdfdc269b50b949b6d1a186e3a2064ed4ee40a41722efea40ae791aa
    source_path: tools/video-generation.md
    workflow: 16
---

OpenClaw-Agenten generieren Videos aus Text-Prompts, Referenzbildern oder
vorhandenen Videos über `video_generate`. Sechzehn Provider-Backends werden
unterstützt; der Agent wählt anhand der Konfiguration und der verfügbaren
API-Schlüssel automatisch das passende aus.

<Note>
`video_generate` wird nur angezeigt, wenn mindestens ein Provider für die
Videogenerierung verfügbar ist. Wenn es in Ihren Agenten-Tools fehlt, legen Sie
einen Provider-API-Schlüssel fest oder konfigurieren Sie `agents.defaults.mediaModels.video`.
</Note>

`video_generate` verfügt über drei Laufzeitmodi, die anhand der
Referenzeingaben im Aufruf bestimmt werden:

- `generate` – keine Referenzmedien (Text-zu-Video).
- `imageToVideo` – ein oder mehrere Referenzbilder.
- `videoToVideo` – ein oder mehrere Referenzvideos.

Provider können eine beliebige Teilmenge dieser Modi unterstützen. Das Tool
validiert den aktiven Modus vor der Übermittlung und meldet die unterstützten
Modi in `action=list`.

## Schnellstart

<Steps>
  <Step title="Authentifizierung konfigurieren">
    Legen Sie einen API-Schlüssel für einen beliebigen unterstützten Provider fest:

    ```bash
    export GEMINI_API_KEY="your-key"
    ```

  </Step>
  <Step title="Standardmodell auswählen (optional)">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "google/veo-3.1-fast-generate-preview"
    ```
  </Step>
  <Step title="Agenten auffordern">
    > Generieren Sie ein 5-sekündiges filmisches Video von einem freundlichen Lobster, der bei Sonnenuntergang surft.

    Der Agent ruft `video_generate` automatisch auf. Eine Aufnahme des Tools
    in eine Zulassungsliste ist nicht erforderlich.

  </Step>
</Steps>

## Funktionsweise der asynchronen Generierung

Die Videogenerierung erfolgt asynchron:

1. OpenClaw übermittelt die Anfrage an den Provider und gibt sofort eine Aufgaben-ID zurück.
2. Der Provider verarbeitet den Auftrag im Hintergrund (je nach Provider und Auflösung normalerweise 30 Sekunden bis mehrere Minuten; langsame warteschlangenbasierte Provider können bis zum konfigurierten Timeout laufen).
3. Sobald das Video bereit ist, aktiviert OpenClaw dieselbe Sitzung mit einem internen Abschlussereignis.
4. Der Agent meldet es über den normalen Modus der Sitzung für sichtbare Antworten:
   als automatische abschließende Antwort oder über `message(action="send")`, wenn die Sitzung
   das Nachrichten-Tool erfordert. Wenn die anfragende Sitzung inaktiv ist oder ihre Aktivierung
   fehlschlägt und die generierten Medien weiterhin in der Abschlussantwort fehlen, sendet OpenClaw
   einen idempotenten direkten Fallback mit den Medien.

Während ein Auftrag ausgeführt wird, geben doppelte Aufrufe von
`video_generate` in derselben Sitzung den aktuellen Aufgabenstatus zurück,
anstatt eine weitere Generierung zu starten. Verwenden Sie `action: "status"`,
um den Status zu prüfen, ohne eine neue Generierung auszulösen, oder
`openclaw tasks list` / `openclaw tasks show <lookup>` über die CLI (siehe
[Hintergrundaufgaben](/de/automation/tasks)).

Außerhalb sitzungsgebundener Agentenläufe (beispielsweise bei direkten
Tool-Aufrufen) wechselt das Tool zur Inline-Generierung und gibt den endgültigen
Medienpfad im selben Durchlauf zurück.

Generierte Videodateien werden im von OpenClaw verwalteten Medienspeicher
gespeichert, wenn der Provider Bytes zurückgibt. Die Standardobergrenze beträgt
16MB (das gemeinsame Medienlimit für Videos); `agents.defaults.mediaMaxMb` erhöht sie
für größere Renderings. Wenn ein Provider außerdem eine gehostete Ausgabe-URL
zurückgibt, stellt OpenClaw diese URL bereit, anstatt die Aufgabe fehlschlagen
zu lassen, falls die lokale Speicherung eine zu große Datei ablehnt.

### Aufgabenlebenszyklus

| Status      | Bedeutung                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------ |
| `queued`    | Aufgabe erstellt; wartet darauf, dass der Provider sie annimmt.                                        |
| `running`   | Der Provider verarbeitet die Aufgabe (je nach Provider und Auflösung normalerweise 30 Sekunden bis mehrere Minuten). |
| `succeeded` | Video bereit; der Agent wird aktiviert und veröffentlicht es in der Unterhaltung.                      |
| `failed`    | Provider-Fehler oder Timeout; der Agent wird mit Fehlerdetails aktiviert.                               |

Status über die CLI prüfen:

```bash
openclaw tasks list
openclaw tasks show <lookup>
openclaw tasks cancel <lookup>
```

## Unterstützte Provider

| Provider              | Standardmodell                  | Text | Bildreferenz                                          | Videoreferenz                                   | Authentifizierung                        |
| --------------------- | ------------------------------- | :--: | ---------------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| Alibaba               | `wan2.6-t2v`                    |  ✓   | Ja (Remote-URL)                                      | Ja (Remote-URL)                                 | `MODELSTUDIO_API_KEY`                    |
| BytePlus (gebündelt)  | `seedance-1-0-pro-250528`       |  ✓   | Bis zu 2 Bilder (erstes + letztes Einzelbild)        | -                                               | `BYTEPLUS_API_KEY`                       |
| BytePlus-1.5-Plugin   | `seedance-1-5-pro-251215`       |  ✓   | Bis zu 2 Bilder (erstes + letztes Einzelbild über Rolle) | -                                           | `BYTEPLUS_API_KEY`                       |
| BytePlus Seedance 2.0 | `dreamina-seedance-2-0-260128`  |  ✓   | Bis zu 9 Referenzbilder                              | Bis zu 3 Videos                                 | `BYTEPLUS_API_KEY`                       |
| ComfyUI               | `workflow`                      |  ✓   | 1 Bild                                               | -                                               | `COMFY_API_KEY` oder `COMFY_CLOUD_API_KEY` |
| DeepInfra             | `Pixverse/Pixverse-T2V`         |  ✓   | -                                                    | -                                               | `DEEPINFRA_API_KEY`                      |
| fal                   | `fal-ai/minimax/video-01-live`  |  ✓   | 1 Bild; bis zu 9 mit Seedance Referenz-zu-Video      | Bis zu 3 Videos mit Seedance Referenz-zu-Video  | `FAL_KEY`                                |
| Google                | `veo-3.1-fast-generate-preview` |  ✓   | 1 Bild                                               | 1 Video                                         | `GEMINI_API_KEY`                         |
| MiniMax               | `MiniMax-Hailuo-2.3`            |  ✓   | 1 Bild                                               | -                                               | `MINIMAX_API_KEY` oder MiniMax OAuth       |
| OpenAI                | `sora-2`                        |  ✓   | 1 Bild                                               | 1 Video                                         | `OPENAI_API_KEY`                         |
| OpenRouter            | `google/veo-3.1-fast`           |  ✓   | Bis zu 4 Bilder (erstes/letztes Einzelbild oder Referenzen) | -                                         | `OPENROUTER_API_KEY`                     |
| Qwen                  | `wan2.6-t2v`                    |  ✓   | Ja (Remote-URL)                                      | Ja (Remote-URL)                                 | `QWEN_API_KEY`                           |
| Runway                | `gen4.5`                        |  ✓   | 1 Bild                                               | 1 Video                                         | `RUNWAYML_API_SECRET`                    |
| Together              | `Wan-AI/Wan2.2-T2V-A14B`        |  ✓   | Nur `Wan-AI/Wan2.2-I2V-A14B`                        | -                                               | `TOGETHER_API_KEY`                       |
| Vydra                 | `veo3`                          |  ✓   | 1 Bild (`kling`)                                    | -                                               | `VYDRA_API_KEY`                          |
| xAI                   | `grok-imagine-video`            |  ✓   | Classic: 1 erstes Einzelbild oder 7 Referenzen; 1.5: 1 Einzelbild | Classic: 1 Video                          | `XAI_API_KEY`                            |

Einige Provider akzeptieren zusätzliche oder alternative Umgebungsvariablen
für API-Schlüssel. Einzelheiten finden Sie auf den jeweiligen
[Provider-Seiten](#related).

Führen Sie `video_generate action=list` aus, um die zur Laufzeit verfügbaren Provider,
Modelle und Laufzeitmodi zu prüfen.

### Funktionsmatrix

Der explizite Modusvertrag, der von `video_generate`, Vertragstests und
dem gemeinsamen Live-Durchlauf verwendet wird:

| Provider   | `generate` | `imageToVideo` | `videoToVideo` | Heutige gemeinsame Live-Testpfade                                                                                                       |
| ---------- | :--------: | :------------: | :------------: | --------------------------------------------------------------------------------------------------------------------------------------- |
| Alibaba    |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` wird übersprungen, da dieser Provider Remote-Video-URLs vom Typ `http(s)` benötigt |
| BytePlus   |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                  |
| ComfyUI    |     ✓      |       ✓        |       -        | Nicht im gemeinsamen Durchlauf; Workflow-spezifische Abdeckung erfolgt durch Comfy-Tests                                                |
| DeepInfra  |     ✓      |       -        |       -        | `generate`; native DeepInfra-Videoschemas sind im Plugin-Vertrag auf Text-zu-Video ausgelegt                                   |
| fal        |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` nur bei Verwendung von Seedance Referenz-zu-Video                           |
| Google     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; der gemeinsame Pfad `videoToVideo` wird übersprungen, da der aktuelle puffergestützte Gemini/Veo-Durchlauf diese Eingabe nicht akzeptiert |
| MiniMax    |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                  |
| OpenAI     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; der gemeinsame Pfad `videoToVideo` wird übersprungen, da dieser Organisations-/Eingabepfad derzeit providerseitigen Zugriff auf die Videobearbeitung benötigt |
| OpenRouter |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                  |
| Qwen       |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` wird übersprungen, da dieser Provider Remote-Video-URLs vom Typ `http(s)` benötigt |
| Runway     |     ✓      |       ✓        |       ✓        | `generate`, `imageToVideo`; `videoToVideo` wird nur ausgeführt, wenn das ausgewählte Modell `runway/gen4_aleph` ist      |
| Together   |     ✓      |       ✓        |       -        | `generate`, `imageToVideo`                                                                                                  |
| Vydra      |     ✓      |       ✓        |       -        | `generate`; der gemeinsame Pfad `imageToVideo` wird übersprungen, da das gebündelte `veo3` nur Text unterstützt und das gebündelte `kling` eine Remote-Bild-URL erfordert |
| xAI        |     ✓      |       ✓        |       ✓        | Classic unterstützt alle Modi; Video 1.5 unterstützt nur Bild-zu-Video; die Remote-MP4-Eingabe schließt `videoToVideo` aus dem gemeinsamen Durchlauf aus |

## Tool-Parameter

### Erforderlich

<ParamField path="prompt" type="string" required>
  Textbeschreibung des zu generierenden Videos. Erforderlich für `action: "generate"`.
</ParamField>

### Inhaltseingaben

<ParamField path="image" type="string">Einzelnes Referenzbild (Pfad oder URL).</ParamField>
<ParamField path="images" type="string[]">Mehrere Referenzbilder (bis zu 9).</ParamField>
<ParamField path="imageRoles" type="string[]">
Optionale positionsbezogene Rollenhinweise parallel zur kombinierten Bilderliste.
Kanonische Werte: `first_frame`, `last_frame`, `reference_image`.
</ParamField>
<ParamField path="video" type="string">Einzelnes Referenzvideo (Pfad oder URL).</ParamField>
<ParamField path="videos" type="string[]">Mehrere Referenzvideos (bis zu 4).</ParamField>
<ParamField path="videoRoles" type="string[]">
Optionale positionsbezogene Rollenhinweise parallel zur kombinierten Videoliste.
Kanonischer Wert: `reference_video`.
</ParamField>
<ParamField path="audioRef" type="string">
Einzelne Referenzaudiodatei (Pfad oder URL). Wird für Hintergrundmusik oder als
Stimmreferenz verwendet, wenn der Provider Audioeingaben unterstützt.
</ParamField>
<ParamField path="audioRefs" type="string[]">Mehrere Referenzaudiodateien (bis zu 3).</ParamField>
<ParamField path="audioRoles" type="string[]">
Optionale positionsbezogene Rollenhinweise parallel zur kombinierten Audioliste.
Kanonischer Wert: `reference_audio`.
</ParamField>

<Note>
Rollenhinweise werden unverändert an den Provider weitergeleitet. Kanonische Werte
stammen aus der Union `VideoGenerationAssetRole`, Provider können jedoch zusätzliche
Rollenzeichenfolgen akzeptieren. `*Roles`-Arrays dürfen nicht mehr Einträge
als die entsprechende Referenzliste enthalten; Abweichungen um eins führen zu einer
eindeutigen Fehlermeldung. Verwenden Sie eine leere Zeichenfolge, um einen Platz
nicht festzulegen. Legen Sie für xAI jede Bildrolle auf `reference_image` fest, um
dessen Generierungsmodus `reference_images` zu verwenden; lassen Sie die Rolle weg
oder verwenden Sie `first_frame` für die Bild-zu-Video-Generierung mit einem
einzelnen Bild.
</Note>

### Stilsteuerung

<ParamField path="aspectRatio" type="string">
  Hinweis zum Seitenverhältnis wie `1:1`, `16:9`, `9:16`, `adaptive` oder ein providerspezifischer Wert. OpenClaw normalisiert oder ignoriert nicht unterstützte Werte je nach Provider.
</ParamField>
<ParamField path="resolution" type="string">Auflösungshinweis wie `360P`, `480P`, `540P`, `720P`, `768P`, `1080P`, `4K` oder ein providerspezifischer Wert. OpenClaw normalisiert oder ignoriert nicht unterstützte Werte je nach Provider.</ParamField>
<ParamField path="durationSeconds" type="number">
  Zieldauer in Sekunden (auf den nächsten vom Provider unterstützten Wert gerundet).
</ParamField>
<ParamField path="size" type="string">Größenhinweis, sofern vom Provider unterstützt.</ParamField>
<ParamField path="audio" type="boolean">
  Aktiviert generiertes Audio in der Ausgabe, sofern unterstützt. Nicht zu verwechseln mit `audioRef*` (Eingaben).
</ParamField>
<ParamField path="watermark" type="boolean">Aktiviert oder deaktiviert Wasserzeichen des Providers, sofern unterstützt.</ParamField>

`adaptive` ist ein providerspezifischer Sentinelwert: Er wird unverändert an
Provider weitergeleitet, die `adaptive` in ihren Fähigkeiten deklarieren
(z. B. verwendet BytePlus Seedance ihn, um das Seitenverhältnis automatisch anhand
der Abmessungen des Eingabebilds zu erkennen). Provider, die ihn nicht deklarieren,
weisen den Wert über `details.ignoredOverrides` im Werkzeugergebnis aus, sodass das Verwerfen
sichtbar ist.

### Erweitert

<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` gibt die aktuelle Aufgabe der Sitzung zurück; `"list"` prüft Provider.
</ParamField>
<ParamField path="model" type="string">Überschreibung von Provider/Modell (z. B. `runway/gen4.5`).</ParamField>
<ParamField path="filename" type="string">Hinweis zum Ausgabedateinamen.</ParamField>
<ParamField path="timeoutMs" type="number">Optionale Zeitüberschreitung für den Provider-Vorgang in Millisekunden. Wenn sie weggelassen wird, verwendet OpenClaw `agents.defaults.mediaModels.video.timeoutMs`, sofern konfiguriert, andernfalls den vom Plugin festgelegten Standardwert des Providers, sofern vorhanden.</ParamField>
<ParamField path="providerOptions" type="object">
  Providerspezifische Optionen als JSON-Objekt (z. B. `{"seed": 42, "draft": true}`).
  Provider, die ein typisiertes Schema deklarieren, validieren Schlüssel und Typen;
  unbekannte Schlüssel oder Abweichungen führen dazu, dass der Kandidat während des
  Fallbacks übersprungen wird. Provider ohne deklariertes Schema erhalten die Optionen
  unverändert. Führen Sie `video_generate action=list` aus, um zu sehen, was die einzelnen
  Provider akzeptieren.
</ParamField>

<Note>
Nicht alle Provider unterstützen alle Parameter. OpenClaw normalisiert die Dauer
auf den nächstgelegenen vom Provider unterstützten Wert und ordnet übersetzte
Geometriehinweise wie Größe-zu-Seitenverhältnis neu zu, wenn ein Fallback-Provider
eine andere Steuerungsoberfläche bereitstellt. Tatsächlich nicht unterstützte
Überschreibungen werden nach dem Best-Effort-Prinzip ignoriert und als Warnungen
im Werkzeugergebnis gemeldet. Harte Fähigkeitsgrenzen (etwa zu viele
Referenzeingaben) führen vor der Übermittlung zu einem Fehler. Werkzeugergebnisse
melden die angewendeten Einstellungen; `details.normalization` erfasst jede Übersetzung
vom angeforderten zum angewendeten Wert.
</Note>

Referenzeingaben bestimmen den Laufzeitmodus:

- Keine Referenzmedien -> `generate`
- Beliebige Bildreferenz -> `imageToVideo`
- Beliebige Videoreferenz -> `videoToVideo`
- Referenzaudioeingaben ändern den aufgelösten Modus **nicht**; sie werden
  zusätzlich zu dem Modus angewendet, den die Bild-/Videoreferenzen bestimmen,
  und funktionieren nur mit Providern, die `maxInputAudios` deklarieren.

Gemischte Bild- und Videoreferenzen sind keine stabile, gemeinsam unterstützte
Fähigkeitsoberfläche. Verwenden Sie vorzugsweise nur einen Referenztyp pro Anfrage.

#### Fallback und typisierte Optionen

Einige Fähigkeitsprüfungen erfolgen auf der Fallback-Ebene statt an der
Werkzeuggrenze. Daher kann eine Anfrage, welche die Grenzen des primären Providers
überschreitet, weiterhin bei einem geeigneten Fallback ausgeführt werden:

- Ein aktiver Kandidat, der kein `maxInputAudios` (oder `0`)
  deklariert, wird übersprungen, wenn die Anfrage Audioreferenzen enthält; der
  nächste Kandidat wird versucht. Dieselbe Prüfung gilt für die Anzahl der Bild-
  und Videoreferenzen gegenüber `maxInputImages`/`maxInputVideos`.
- Liegt `maxDurationSeconds` des aktiven Kandidaten unter dem angeforderten
  `durationSeconds` und ist keine `supportedDurationSeconds`-Liste deklariert, wird er
  übersprungen.
- Enthält die Anfrage `providerOptions` und deklariert der aktive Kandidat
  ausdrücklich ein typisiertes `providerOptions`-Schema, wird er übersprungen,
  wenn die angegebenen Schlüssel nicht im Schema enthalten sind oder die Werttypen
  nicht übereinstimmen. Provider ohne deklariertes Schema erhalten die Optionen
  unverändert (abwärtskompatible Durchleitung). Ein Provider kann sämtliche
  Provideroptionen ablehnen, indem er ein leeres Schema (`capabilities.providerOptions: {}`)
  deklariert; dies führt zum gleichen Überspringen wie eine Typabweichung.

Der erste Grund für das Überspringen in einer Anfrage wird unter `warn`
protokolliert, damit Betreiber erkennen können, wann ihr primärer Provider übergangen
wurde; nachfolgende Gründe werden unter `debug` protokolliert, damit lange
Fallback-Ketten keine unnötigen Meldungen erzeugen. Wenn jeder Kandidat übersprungen
wird, enthält der zusammengefasste Fehler den jeweiligen Grund für jeden Kandidaten.

## Aktionen

| Aktion     | Funktion                                                                                                 |
| ---------- | -------------------------------------------------------------------------------------------------------- |
| `generate` | Standard. Erstellt aus der angegebenen Eingabeaufforderung und optionalen Referenzeingaben ein Video.    |
| `status`   | Prüft den Status der laufenden Videoaufgabe für die aktuelle Sitzung, ohne eine weitere Generierung zu starten. |
| `list`     | Zeigt verfügbare Provider, Modelle und deren Fähigkeiten an.                                             |

## Modellauswahl

OpenClaw löst das Modell in dieser Reihenfolge auf:

1. **Werkzeugparameter `model`** – falls der Agent einen im Aufruf angibt.
2. **`videoGenerationModel.primary`** aus der Konfiguration.
3. **`videoGenerationModel.fallbacks`** der Reihe nach.
4. **Automatische Erkennung** – Provider mit gültiger Authentifizierung, beginnend
   mit dem aktuellen Standard-Provider, danach die übrigen Provider in
   alphabetischer Reihenfolge.

Wenn ein Provider fehlschlägt, wird automatisch der nächste Kandidat versucht.
Wenn alle Kandidaten fehlschlagen, enthält der Fehler Details zu jedem Versuch.

Der automatische Fallback zwischen authentifizierten Providern ist immer aktiviert.
Ein aufrufbezogenes `model` bleibt maßgeblich.

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
        fallbacks: ["runway/gen4.5", "qwen/wan2.6-t2v"],
        timeoutMs: 180000, // optionale auf das Werkzeug bezogene Überschreibung der Zeitüberschreitung für Provider-Anfragen
      },
    },
  },
}
```

## Hinweise zu Providern

<AccordionGroup>
  <Accordion title="Alibaba">
    Verwendet den asynchronen Endpunkt von DashScope / Model Studio. Referenzbilder
    und -videos müssen entfernte `http(s)`-URLs sein.
  </Accordion>
  <Accordion title="BytePlus (gebündelt)">
    Provider-ID: `byteplus`.

    Modelle: `seedance-1-0-pro-250528` (Standard),
    `seedance-1-5-pro-251215`.

    Verwendet die einheitliche `content[]`-API. Unterstützt bis zu 2
    Eingabebilder (`first_frame` + `last_frame`). Übergeben Sie Bilder
    positionsbezogen oder legen Sie `role` jedes Bilds ausdrücklich fest.

    Unterstützte `providerOptions`-Schlüssel: `seed` (Zahl),
    `draft` (boolescher Wert – erzwingt 480p), `camera_fixed`
    (boolescher Wert).

  </Accordion>
  <Accordion title="BytePlus Seedance 1.5 plugin">
    Erfordert das Plugin [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    (extern, nicht gebündelt). Provider-ID: `byteplus-seedance15`. Modell:
    `seedance-1-5-pro-251215`.

    Verwendet die einheitliche `content[]`-API. Unterstützt höchstens 2
    Eingabebilder (`first_frame` + `last_frame`). Alle Eingaben müssen
    entfernte `https://`-URLs sein. Legen Sie `role: "first_frame"` /
    `"last_frame"` für jedes Bild fest oder übergeben Sie Bilder positionsbezogen.

    `aspectRatio: "adaptive"` erkennt das Seitenverhältnis automatisch anhand des
    Eingabebilds. `audio: true` wird `generate_audio` zugeordnet.
    `providerOptions.seed` (Zahl) wird weitergeleitet.

  </Accordion>
  <Accordion title="BytePlus Seedance 2.0">
    Erfordert das Plugin [`@openclaw/byteplus-modelark`](https://www.npmjs.com/package/@openclaw/byteplus-modelark)
    (extern, nicht gebündelt). Provider-ID: `byteplus-seedance2`. Modelle:
    `dreamina-seedance-2-0-260128`,
    `dreamina-seedance-2-0-fast-260128`.

    Verwendet die einheitliche `content[]`-API. Unterstützt bis zu 9
    Referenzbilder, 3 Referenzvideos und 3 Referenzaudiodateien. Alle Eingaben
    müssen entfernte `https://`-URLs sein. Legen Sie `role`
    für jedes Asset fest – unterstützte Werte:
    `"first_frame"`, `"last_frame"`, `"reference_image"`,
    `"reference_video"`, `"reference_audio"`.

    `aspectRatio: "adaptive"` erkennt das Seitenverhältnis automatisch anhand des
    Eingabebilds. `audio: true` wird `generate_audio` zugeordnet.
    `providerOptions.seed` (Zahl) wird weitergeleitet.

  </Accordion>
  <Accordion title="ComfyUI">
    Workflow-gesteuerte lokale oder Cloud-Ausführung. Unterstützt Text-zu-Video und
    Bild-zu-Video über den konfigurierten Graphen.
  </Accordion>
  <Accordion title="fal">
    Verwendet einen warteschlangengestützten Ablauf für lang laufende Aufträge. OpenClaw wartet standardmäßig bis zu 20
    Minuten, bevor ein noch laufender fal-Warteschlangenauftrag als
    Zeitüberschreitung behandelt wird. Die meisten fal-Videomodelle
    akzeptieren eine einzelne Bildreferenz. Seedance-2.0-Referenz-zu-Video-
    Modelle akzeptieren bis zu 9 Bilder, 3 Videos und 3 Audioreferenzen mit
    insgesamt höchstens 12 Referenzdateien.
  </Accordion>
  <Accordion title="Google (Gemini / Veo)">
    Unterstützt eine Bild- oder eine Videoreferenz. Anfragen für generiertes Audio werden
    im Gemini-API-Pfad mit einer Warnung ignoriert, da diese API den Parameter
    `generateAudio` für die aktuelle Veo-Videogenerierung ablehnt.
  </Accordion>
  <Accordion title="MiniMax">
    Nur eine einzelne Bildreferenz. MiniMax akzeptiert die Auflösungen `768P` und `1080P`;
    Anfragen wie `720P` werden vor dem Senden auf den nächstgelegenen
    unterstützten Wert normalisiert.
  </Accordion>
  <Accordion title="OpenAI">
    Nur die Überschreibung `size` wird weitergeleitet. Andere Stilüberschreibungen
    (`aspectRatio`, `resolution`, `audio`, `watermark`) werden mit
    einer Warnung ignoriert.
  </Accordion>
  <Accordion title="OpenRouter">
    Verwendet die asynchrone `/videos`-API von OpenRouter. OpenClaw übermittelt den
    Auftrag, fragt `polling_url` ab und lädt entweder `unsigned_urls` oder den
    dokumentierten Inhaltsendpunkt des Auftrags herunter. Der gebündelte Standard `google/veo-3.1-fast`
    weist Dauern von 4/6/8 Sekunden, die Auflösungen `720P`/`1080P` und
    die Seitenverhältnisse `16:9`/`9:16` aus.
  </Accordion>
  <Accordion title="Qwen">
    Dasselbe DashScope-Backend wie Alibaba. Referenzeingaben müssen entfernte
    `http(s)`-URLs sein; lokale Dateien werden vorab abgelehnt.
  </Accordion>
  <Accordion title="Runway">
    Unterstützt lokale Dateien über Daten-URIs. Video-zu-Video erfordert
    `runway/gen4_aleph`. Reine Textausführungen bieten die Seitenverhältnisse `16:9` und `9:16`
    an.
  </Accordion>
  <Accordion title="Together">
    Nur eine einzelne Bildreferenz.
  </Accordion>
  <Accordion title="Vydra">
    Verwendet `https://www.vydra.ai/api/v1` direkt, um Weiterleitungen zu vermeiden, bei denen
    die Authentifizierung verloren geht. `veo3` ist ausschließlich für Text-zu-Video gebündelt; `kling` erfordert
    eine entfernte Bild-URL.
  </Accordion>
  <Accordion title="xAI">
    Das Standardmodell `grok-imagine-video` unterstützt Text-zu-Video,
    Bild-zu-Video mit einem einzelnen ersten Frame, bis zu 7 `reference_image`-Eingaben über xAI
    `reference_images` sowie Abläufe zum Bearbeiten und Erweitern entfernter Videos. Die Generierung verwendet standardmäßig
    `480P`; Bild-zu-Video mit einem einzelnen Bild übernimmt das Seitenverhältnis der Quelle, wenn
    `aspectRatio` ausgelassen wird. Videobearbeitung und -erweiterung übernehmen die Geometrie der Eingabe und
    akzeptieren keine Überschreibungen für Seitenverhältnis oder Auflösung. Die Erweiterung akzeptiert 2-10
    Sekunden.

    `grok-imagine-video-1.5` unterstützt ausschließlich Bild-zu-Video: Geben Sie genau ein Bild an.
    Es unterstützt 1-15 Sekunden und `480P`, `720P` oder `1080P`, standardmäßig
    `480P`; lassen Sie `aspectRatio` aus, um das Seitenverhältnis des Quellbilds zu übernehmen. Die Vorschau-
    und datierten 1.5-Bezeichner werden derselben Validierung unterzogen und unverändert
    weitergeleitet.

  </Accordion>
</AccordionGroup>

## Provider-Fähigkeitsmodi

Der gemeinsame Vertrag zur Videogenerierung unterstützt modusspezifische Fähigkeiten
anstelle ausschließlich flacher Gesamtgrenzwerte. Neue Provider-Implementierungen
sollten explizite Modusblöcke bevorzugen:

```typescript
capabilities: {
  generate: {
    maxVideos: 1,
    maxDurationSeconds: 10,
    supportsResolution: true,
  },
  imageToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputImages: 1,
    maxInputImagesByModel: { "provider/reference-to-video": 9 },
    maxDurationSeconds: 5,
  },
  videoToVideo: {
    enabled: true,
    maxVideos: 1,
    maxInputVideos: 1,
    maxDurationSeconds: 5,
  },
}
```

Flache Gesamtfelder wie `maxInputImages` und `maxInputVideos` reichen
**nicht** aus, um die Unterstützung von Transformationsmodi auszuweisen. Provider sollten
`generate`, `imageToVideo` und `videoToVideo` explizit deklarieren, damit Live-
Tests, Vertragstests und das gemeinsame Tool `video_generate` die
Modusunterstützung deterministisch validieren können.

Wenn ein Modell eines Providers eine breitere Unterstützung für Referenzeingaben als die
übrigen bietet, verwenden Sie `maxInputImagesByModel`, `maxInputVideosByModel` oder
`maxInputAudiosByModel`, anstatt den Grenzwert für den gesamten Modus anzuheben.

## Live-Tests

Optional aktivierbare Live-Abdeckung für die gemeinsam gebündelten Provider:

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts
```

Repo-Wrapper:

```bash
pnpm test:live:media video
```

Diese Live-Datei verwendet standardmäßig bereits exportierte Provider-Umgebungsvariablen vor gespeicherten
Authentifizierungsprofilen und führt standardmäßig einen release-sicheren Smoke-Test aus:

- `generate` für jeden Nicht-FAL-Provider im Durchlauf.
- Einsekündiger Lobster-Prompt.
- Grenzwert für Vorgänge pro Provider aus
  `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (standardmäßig `180000`).

FAL muss ausdrücklich aktiviert werden, da die providerseitige Warteschlangenlatenz die Release-
Dauer dominieren kann:

```bash
pnpm test:live:media video --video-providers fal
```

Setzen Sie `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1`, um zusätzlich deklarierte
Transformationsmodi auszuführen, die der gemeinsame Durchlauf sicher mit lokalen Medien testen kann:

- `imageToVideo`, wenn `capabilities.imageToVideo.enabled`.
- `videoToVideo`, wenn `capabilities.videoToVideo.enabled` und der
  Provider bzw. das Modell im gemeinsamen Durchlauf puffergestützte lokale Videoeingaben akzeptiert.

Derzeit deckt die gemeinsame Live-Teststrecke `videoToVideo` nur dann `runway` ab, wenn Sie
`runway/gen4_aleph` auswählen.

## Konfiguration

Legen Sie das Standardmodell für die Videogenerierung in Ihrer OpenClaw-Konfiguration fest:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "qwen/wan2.6-t2v",
        fallbacks: ["qwen/wan2.6-r2v-flash"],
      },
    },
  },
}
```

Oder über die CLI:

```bash
openclaw config set agents.defaults.mediaModels.video.primary "qwen/wan2.6-t2v"
```

## Verwandte Themen

- [Alibaba Model Studio](/de/providers/alibaba)
- [Hintergrundaufgaben](/de/automation/tasks) - Aufgabenverfolgung für asynchrone Videogenerierung
- [BytePlus](/de/concepts/model-providers#byteplus-international)
- [ComfyUI](/de/providers/comfy)
- [Konfigurationsreferenz](/de/gateway/config-agents#agent-defaults)
- [fal](/de/providers/fal)
- [Google (Gemini)](/de/providers/google)
- [MiniMax](/de/providers/minimax)
- [Modelle](/de/concepts/models)
- [OpenAI](/de/providers/openai)
- [Qwen](/de/providers/qwen)
- [Runway](/de/providers/runway)
- [Together AI](/de/providers/together)
- [Tool-Übersicht](/de/tools)
- [Vydra](/de/providers/vydra)
- [xAI](/de/providers/xai)
