---
read_when:
    - Sie möchten die Runway-Videogenerierung in OpenClaw verwenden
    - Sie müssen den Runway-API-Schlüssel bzw. die Umgebungsvariable einrichten
    - Sie möchten Runway zum standardmäßigen Video-Provider machen
summary: Einrichtung der Runway-Videogenerierung in OpenClaw
title: Runway
x-i18n:
    generated_at: "2026-07-26T18:44:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a56e768893e327b56d70e8b8c2d426123a861b3cf05c0107d98104e2cee856c
    source_path: providers/runway.md
    workflow: 16
---

OpenClaw enthält einen gebündelten `runway`-Provider für gehostete Videogenerierung, der standardmäßig aktiviert und für den `videoGenerationProviders`-Vertrag registriert ist.

| Eigenschaft        | Wert                                                             |
| --------------- | ----------------------------------------------------------------- |
| Provider-ID     | `runway`                                                          |
| Plugin          | gebündelt, `enabledByDefault: true`                                 |
| Umgebungsvariablen für die Authentifizierung   | `RUNWAYML_API_SECRET` (kanonisch) oder `RUNWAY_API_KEY`             |
| Onboarding-Flag | `--auth-choice runway-api-key`                                    |
| Direktes CLI-Flag | `--runway-api-key <key>`                                          |
| API             | Aufgabenbasierte Videogenerierung von Runway (`GET /v1/tasks/{id}`-Polling) |
| Standardmodell   | `runway/gen4.5`                                                   |

## Erste Schritte

<Steps>
  <Step title="API-Schlüssel festlegen">
    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```
  </Step>
  <Step title="Runway als Standard-Provider für Videos festlegen">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "runway/gen4.5"
    ```
  </Step>
  <Step title="Ein Video generieren">
    Bitten Sie den Agenten, ein Video zu generieren. Runway wird automatisch verwendet.
  </Step>
</Steps>

## Unterstützte Modi und Modelle

Der Provider stellt sieben Runway-Modelle in drei Modi bereit. Dieselbe Modell-ID kann für mehrere Modi verwendet werden (beispielsweise funktioniert `gen4.5` sowohl für Text-zu-Video als auch für Bild-zu-Video).

| Modus           | Modelle                                                                 | Referenzeingabe         |
| -------------- | ---------------------------------------------------------------------- | ----------------------- |
| Text-zu-Video  | `gen4.5` (Standard), `veo3.1`, `veo3.1_fast`, `veo3`                    | Keine                    |
| Bild-zu-Video | `gen4.5`, `gen4_turbo`, `gen3a_turbo`, `veo3.1`, `veo3.1_fast`, `veo3` | 1 lokales oder entferntes Bild |
| Video-zu-Video | `gen4_aleph`                                                           | 1 lokales oder entferntes Video |

Lokale Bild- und Videoreferenzen werden über Daten-URIs unterstützt.

| Seitenverhältnisse         | Zulässige Werte                              |
| --------------------- | ------------------------------------------- |
| Text-zu-Video         | `16:9`, `9:16`                              |
| Bild- und Videobearbeitung | `1:1`, `16:9`, `9:16`, `3:4`, `4:3`, `21:9` |

<Warning>
  Video-zu-Video erfordert derzeit `runway/gen4_aleph`. Andere Runway-Modell-IDs lehnen Videoreferenzeingaben ab.
</Warning>

<Note>
  Wenn eine Runway-Modell-ID aus der falschen Spalte ausgewählt wird, wird ein eindeutiger Fehler ausgegeben, bevor die API-Anfrage OpenClaw verlässt. Der Provider validiert `model` anhand der Positivliste des Modus (`TEXT_ONLY_MODELS`, `IMAGE_MODELS`, `VIDEO_MODELS`) in `extensions/runway/video-generation-provider.ts`.
</Note>

## Konfiguration

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="Aliasse für Umgebungsvariablen">
    OpenClaw erkennt sowohl `RUNWAYML_API_SECRET` (kanonisch) als auch `RUNWAY_API_KEY`.
    Beide Variablen authentifizieren den Runway-Provider.
  </Accordion>

  <Accordion title="Aufgaben-Polling">
    Runway verwendet eine aufgabenbasierte API. Nach dem Absenden einer Generierungsanfrage fragt OpenClaw
    `GET /v1/tasks/{id}` ab, bis das Video bereit ist. Für das Polling-Verhalten ist keine zusätzliche
    Konfiguration erforderlich.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Videogenerierung" href="/de/tools/video-generation" icon="video">
    Gemeinsame Tool-Parameter, Provider-Auswahl und asynchrones Verhalten.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/config-agents#agent-defaults" icon="gear">
    Standardeinstellungen des Agenten einschließlich des Videogenerierungsmodells.
  </Card>
</CardGroup>
