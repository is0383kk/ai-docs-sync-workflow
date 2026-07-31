---
read_when:
    - Sie möchten Vydra-Mediengenerierung in OpenClaw verwenden
    - Sie benötigen eine Anleitung zum Einrichten des Vydra-API-Schlüssels
summary: Vydra-Bild-, -Video- und -Sprachfunktionen in OpenClaw verwenden
title: Vydra
x-i18n:
    generated_at: "2026-07-26T18:07:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cc3856c2dd740e87d70d7eedefd9eae7905ab547aa0d68a1c479a305c59b2982
    source_path: providers/vydra.md
    workflow: 16
---

Das mitgelieferte Vydra-Plugin fügt Folgendes hinzu:

- Bildgenerierung über `vydra/grok-imagine`
- Videogenerierung über `vydra/veo3` (Text-zu-Video) und `vydra/kling` (Bild-zu-Video)
- Sprachsynthese über Vydras ElevenLabs-gestützte TTS-Route

OpenClaw verwendet für alle drei Funktionen denselben `VYDRA_API_KEY`.

| Eigenschaft     | Wert                                                                      |
| --------------- | ------------------------------------------------------------------------- |
| Provider-ID     | `vydra`                                                        |
| Plugin          | mitgeliefert, `enabledByDefault: true`                                         |
| Auth-Umgebungsvariable | `VYDRA_API_KEY`                                                  |
| Onboarding-Flag | `--auth-choice vydra-api-key`                                                        |
| Direktes CLI-Flag | `--vydra-api-key <key>`                                                      |
| Verträge        | `imageGenerationProviders`, `videoGenerationProviders`, `speechProviders`               |
| Basis-URL       | `https://www.vydra.ai/api/v1` (den Host `www` verwenden)                |

<Warning>
Verwenden Sie `https://www.vydra.ai/api/v1` als Basis-URL. Vydras Apex-Host (`https://vydra.ai/api/v1`) leitet derzeit zu `www` weiter. Einige HTTP-Clients verwerfen bei dieser hostübergreifenden Weiterleitung `Authorization`, wodurch ein gültiger API-Schlüssel zu einem irreführenden Authentifizierungsfehler führt. Das mitgelieferte Plugin normalisiert jede konfigurierte `vydra.ai`-Basis-URL zu `www.vydra.ai`, um dies zu vermeiden.
</Warning>

## Einrichtung

<Steps>
  <Step title="Interaktives Onboarding ausführen">
    ```bash
    openclaw onboard --auth-choice vydra-api-key
    ```

    Alternativ können Sie die Umgebungsvariable direkt festlegen:

    ```bash
    export VYDRA_API_KEY="vydra_live_..."
    ```

  </Step>
  <Step title="Eine Standardfunktion auswählen">
    Wählen Sie mindestens eine der folgenden Funktionen (Bild, Video oder Sprache) und wenden Sie die entsprechende Konfiguration an.
  </Step>
</Steps>

## Funktionen

<AccordionGroup>
  <Accordion title="Bildgenerierung">
    Standardmäßiges und einziges mitgeliefertes Bildmodell:

    - `vydra/grok-imagine`

    Legen Sie es als standardmäßigen Bild-Provider fest:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "vydra/grok-imagine",
          },
        },
      },
    }
    ```

    Die mitgelieferte Unterstützung umfasst ausschließlich Text-zu-Bild und höchstens ein Bild pro Anfrage. Vydras gehostete Bearbeitungsrouten erwarten Remote-Bild-URLs, und das mitgelieferte Plugin fügt keine Vydra-spezifische Upload-Brücke hinzu.

    <Note>
    Unter [Bildgenerierung](/de/tools/image-generation) finden Sie Informationen zu gemeinsamen Tool-Parametern, zur Provider-Auswahl und zum Failover-Verhalten.
    </Note>

  </Accordion>

  <Accordion title="Videogenerierung">
    Registrierte Videomodelle:

    - `vydra/veo3` für Text-zu-Video (lehnt Bildreferenzeingaben ab)
    - `vydra/kling` für Bild-zu-Video (erfordert genau eine Remote-Bild-URL)

    Legen Sie Vydra als standardmäßigen Video-Provider fest:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "vydra/veo3",
          },
        },
      },
    }
    ```

    Hinweise:

    - `vydra/kling` lehnt lokale Datei-Uploads von vornherein ab; es funktioniert nur eine Referenz auf eine Remote-Bild-URL.
    - Vydras `kling`-HTTP-Route war uneinheitlich darin, ob sie `image_url` oder `video_url` erfordert; der mitgelieferte Provider sendet dieselbe Remote-Bild-URL in beiden Feldern.
    - Das mitgelieferte Plugin bleibt konservativ und leitet undokumentierte Stiloptionen wie Seitenverhältnis, Auflösung, Wasserzeichen oder generiertes Audio nicht weiter.

    <Note>
    Unter [Videogenerierung](/de/tools/video-generation) finden Sie Informationen zu gemeinsamen Tool-Parametern, zur Provider-Auswahl und zum Failover-Verhalten.
    </Note>

  </Accordion>

  <Accordion title="Video-Live-Tests">
    Provider-spezifische Live-Abdeckung:

    ```bash
    OPENCLAW_LIVE_TEST=1 \
    OPENCLAW_LIVE_VYDRA_VIDEO=1 \
    pnpm test:live -- extensions/vydra/vydra.live.test.ts
    ```

    Die mitgelieferte Vydra-Live-Datei deckt Folgendes ab:

    - `vydra/veo3` Text-zu-Video
    - `vydra/kling` Bild-zu-Video unter Verwendung einer Remote-Bild-URL

    Überschreiben Sie bei Bedarf die Remote-Bild-Testressource:

    ```bash
    export OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL="https://example.com/reference.png"
    ```

  </Accordion>

  <Accordion title="Sprachsynthese">
    Legen Sie Vydra als Sprach-Provider fest:

    ```json5
    {
      tts: {
        provider: "vydra",
        providers: {
          vydra: {
            apiKey: "${VYDRA_API_KEY}",
            voiceId: "21m00Tcm4TlvDq8ikWAM",
          },
        },
      },
    }
    ```

    Standardwerte:

    - Modell: `elevenlabs/tts`
    - Stimmen-ID: `21m00Tcm4TlvDq8ikWAM` ("Rachel")

    Das mitgelieferte Plugin stellt diese eine bewährte Standardstimme bereit und gibt MP3-Audiodateien zurück.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Provider-Verzeichnis" href="/de/providers/index" icon="list">
    Durchsuchen Sie alle verfügbaren Provider.
  </Card>
  <Card title="Bildgenerierung" href="/de/tools/image-generation" icon="image">
    Gemeinsame Parameter des Bild-Tools und Provider-Auswahl.
  </Card>
  <Card title="Videogenerierung" href="/de/tools/video-generation" icon="video">
    Gemeinsame Parameter des Video-Tools und Provider-Auswahl.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/config-agents#agent-defaults" icon="gear">
    Agent-Standardwerte und Modellkonfiguration.
  </Card>
</CardGroup>
