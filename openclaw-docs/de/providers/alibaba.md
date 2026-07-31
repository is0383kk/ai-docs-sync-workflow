---
read_when:
    - Sie möchten die Videogenerierung von Alibaba Wan in OpenClaw verwenden
    - Für die Videogenerierung müssen Sie einen Model-Studio- oder DashScope-API-Schlüssel einrichten.
summary: Alibaba Model Studio Wan-Videogenerierung in OpenClaw
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-07-26T18:04:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb74e2361500ccfbc5d3c4f2d08c3b62aacba8c79c704570952e2181abacf9fb
    source_path: providers/alibaba.md
    workflow: 16
---

Das mitgelieferte `alibaba`-Plugin registriert einen Provider zur Videogenerierung für Wan-Modelle in Alibaba Model Studio (der internationale Name für DashScope). Es ist standardmäßig aktiviert; lediglich ein API-Schlüssel ist erforderlich.

| Eigenschaft          | Wert                                                                            |
| -------------------- | ------------------------------------------------------------------------------- |
| Provider-ID          | `alibaba`                                                              |
| Plugin               | mitgeliefert, `enabledByDefault: true`                                                |
| Auth.-Umgebungsvar.  | `MODELSTUDIO_API_KEY` → `DASHSCOPE_API_KEY` → `QWEN_API_KEY` (der erste Treffer wird verwendet) |
| Onboarding-Flag      | `--auth-choice alibaba-model-studio-api-key`                                                              |
| Direktes CLI-Flag    | `--alibaba-model-studio-api-key <key>`                                                              |
| Standardmodell       | `alibaba/wan2.6-t2v`                                                              |
| Standard-Basis-URL   | `https://dashscope-intl.aliyuncs.com`                                                              |

## Erste Schritte

<Steps>
  <Step title="API-Schlüssel festlegen">
    Speichern Sie den Schlüssel über das Onboarding für den Provider `alibaba`:

    ```bash
    openclaw onboard --auth-choice alibaba-model-studio-api-key
    ```

    Alternativ können Sie den Schlüssel direkt übergeben:

    ```bash
    openclaw onboard --alibaba-model-studio-api-key <your-key>
    ```

    Oder exportieren Sie vor dem Start des Gateways eine der akzeptierten Umgebungsvariablen:

    ```bash
    export MODELSTUDIO_API_KEY=sk-...
    # oder DASHSCOPE_API_KEY=...
    # oder QWEN_API_KEY=...
    ```

  </Step>
  <Step title="Standardmodell für Videos festlegen">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "alibaba/wan2.6-t2v",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Konfiguration des Providers überprüfen">
    ```bash
    openclaw models list --provider alibaba
    ```

    Die Liste enthält alle fünf mitgelieferten Wan-Modelle. Wenn `MODELSTUDIO_API_KEY` nicht aufgelöst werden kann, meldet `openclaw models status --json` die fehlenden Anmeldedaten unter `auth.unusableProfiles`.

  </Step>
</Steps>

<Note>
  Sowohl das Alibaba-Plugin als auch das [Qwen-Plugin](/de/providers/qwen) authentifizieren sich bei DashScope und akzeptieren teilweise dieselben Umgebungsvariablen. Verwenden Sie `alibaba/...`-Modell-IDs für die dedizierte Wan-Videooberfläche; verwenden Sie `qwen/...`-IDs für Qwen-Chat, Einbettungen oder Medienverständnis.
</Note>

## Integrierte Wan-Modelle

| Modellreferenz             | Modus                            |
| -------------------------- | -------------------------------- |
| `alibaba/wan2.6-t2v`         | Text-zu-Video (Standard)         |
| `alibaba/wan2.6-i2v`         | Bild-zu-Video                    |
| `alibaba/wan2.6-r2v`         | Referenz-zu-Video                |
| `alibaba/wan2.6-r2v-flash`         | Referenz-zu-Video (schnell)      |
| `alibaba/wan2.7-r2v`         | Referenz-zu-Video                |

## Funktionen und Beschränkungen

Für alle drei Modi gelten dieselbe Anzahl von Videos pro Anfrage und dieselbe maximale Dauer; lediglich die Eingabeform unterscheidet sich.

| Modus              | Max. Ausgabevideos | Max. Eingabebilder | Max. Eingabevideos | Max. Dauer | Unterstützte Steuerelemente                               |
| ------------------ | ------------------- | ------------------ | ------------------- | ---------- | --------------------------------------------------------- |
| Text-zu-Video      | 1                   | k. A.              | k. A.               | 10 s       | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Bild-zu-Video      | 1                   | 1                  | k. A.               | 10 s       | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Referenz-zu-Video  | 1                   | k. A.              | 4                    | 10 s       | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |

Bei einer Anfrage ohne `durationSeconds` wird der von DashScope akzeptierte Standardwert von **5 Sekunden** verwendet. Legen Sie `durationSeconds` im [Tool zur Videogenerierung](/de/tools/video-generation) explizit fest, um die Dauer auf bis zu 10 s zu verlängern.

<Warning>
  Eingaben mit Referenzbildern und -videos müssen Remote-`http(s)`-URLs sein; die Referenzmodi von DashScope lehnen lokale Dateipfade ab. Laden Sie die Dateien zuerst in einen Objektspeicher hoch oder verwenden Sie den Ablauf des [Medien-Tools](/de/tools/media-overview), der bereits eine öffentliche URL erzeugt.
</Warning>

## Erweiterte Konfiguration

<AccordionGroup>
  <Accordion title="DashScope-Basis-URL überschreiben">
    Der Provider verwendet standardmäßig den internationalen DashScope-Endpunkt. So verwenden Sie stattdessen den Endpunkt für die Region China:

    ```json5
    {
      models: {
        providers: {
          alibaba: {
            baseUrl: "https://dashscope.aliyuncs.com",
          },
        },
      },
    }
    ```

    Der Provider entfernt abschließende Schrägstriche, bevor er die URLs für AIGC-Aufgaben erstellt.

  </Accordion>

  <Accordion title="Priorität der Auth.-Umgebungsvariablen">
    OpenClaw ermittelt den Alibaba-API-Schlüssel in dieser Reihenfolge aus den Umgebungsvariablen und verwendet den ersten nicht leeren Wert:

    1. `MODELSTUDIO_API_KEY`
    2. `DASHSCOPE_API_KEY`
    3. `QWEN_API_KEY`

    Konfigurierte `auth.profiles`-Einträge (festgelegt über `openclaw models auth login`) haben Vorrang vor der Auflösung über Umgebungsvariablen. Informationen zur Profilrotation, Abklingzeit und zu Überschreibungsmechanismen finden Sie unter [Authentifizierungsprofile in den häufig gestellten Fragen zu Modellen](/de/help/faq-models#auth-profiles-what-they-are-and-how-to-manage-them).

  </Accordion>

  <Accordion title="Beziehung zum Qwen-Plugin">
    Beide mitgelieferten Plugins kommunizieren mit DashScope und akzeptieren teilweise dieselben API-Schlüssel. Verwenden Sie:

    - `alibaba/wan*.*`-IDs für den auf dieser Seite dokumentierten dedizierten Wan-Video-Provider.
    - `qwen/*`-IDs für Qwen-Chat, Einbettungen und Medienverständnis (siehe [Qwen](/de/providers/qwen)).

    Wenn Sie `MODELSTUDIO_API_KEY` einmal festlegen, werden beide Plugins authentifiziert, da sich die Listen der Auth.-Umgebungsvariablen absichtlich überschneiden; ein separates Onboarding für jedes Plugin ist nicht erforderlich.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Videogenerierung" href="/de/tools/video-generation" icon="video">
    Gemeinsame Parameter des Video-Tools und Auswahl des Providers.
  </Card>
  <Card title="Qwen" href="/de/providers/qwen" icon="microchip">
    Einrichtung von Qwen-Chat, Einbettungen und Medienverständnis mit derselben DashScope-Authentifizierung.
  </Card>
  <Card title="Konfigurationsreferenz" href="/de/gateway/config-agents#agent-defaults" icon="gear">
    Agentenstandards und Modellkonfiguration.
  </Card>
  <Card title="Häufig gestellte Fragen zu Modellen" href="/de/help/faq-models" icon="circle-question">
    Authentifizierungsprofile, Modellwechsel und Behebung von „Kein Profil“-Fehlern.
  </Card>
</CardGroup>
