---
read_when:
    - Sie möchten die Inworld-Sprachsynthese für ausgehende Antworten verwenden
    - Sie benötigen von Inworld eine Sprachausgabe im PCM-Telefonieformat oder Sprachnachrichten im Format OGG_OPUS
summary: Inworld-Streaming-Text-to-Speech für OpenClaw-Antworten
title: Inworld
x-i18n:
    generated_at: "2026-07-26T18:43:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09560f5beda3b40d9c67f9408d34446f28ecddb8235fc0725c4265c813302946
    source_path: providers/inworld.md
    workflow: 16
---

Inworld ist ein Provider für Streaming-Text-to-Speech (TTS). In OpenClaw synthetisiert er Audio für ausgehende Antworten (standardmäßig MP3, OGG_OPUS für Sprachnachrichten) sowie unkomprimiertes PCM-Audio für Telefoniekanäle wie Voice Call.

OpenClaw sendet Anfragen an den Streaming-TTS-Endpunkt von Inworld, fügt die zurückgegebenen Base64-Audiosegmente zu einem einzigen Puffer zusammen und übergibt das Ergebnis an die standardmäßige Pipeline für Antwortaudio.

| Eigenschaft    | Wert                                                            |
| -------------- | --------------------------------------------------------------- |
| Provider-ID    | `inworld`                                              |
| Plugin         | offizielles externes Paket (`@openclaw/inworld-speech`)                 |
| Vertrag        | `speechProviders` (nur TTS)                                    |
| Auth.-Env.-Var. | `INWORLD_API_KEY` (HTTP Basic, Base64-Dashboard-Zugangsdaten) |
| Basis-URL      | `https://api.inworld.ai`                                              |
| Standardstimme | `Sarah`                                              |
| Standardmodell | `inworld-tts-1.5-max`                                              |
| Ausgabe        | MP3 (Standard), OGG_OPUS (Sprachnachrichten), PCM 22050 Hz (Telefonie) |
| Website        | [inworld.ai](https://inworld.ai)                                |
| Dokumentation  | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)      |

## Plugin installieren

```bash
openclaw plugins install @openclaw/inworld-speech
openclaw gateway restart
```

## Erste Schritte

<Steps>
  <Step title="API-Schlüssel festlegen">
    Kopieren Sie die Zugangsdaten aus Ihrem Inworld-Dashboard (Workspace > API Keys) und legen Sie sie als Umgebungsvariable fest. Der Wert wird unverändert als HTTP-Basic-Zugangsdaten gesendet. Codieren Sie ihn daher nicht erneut mit Base64 und wandeln Sie ihn nicht in ein Bearer-Token um.

    ```bash
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

  </Step>
  <Step title="Inworld unter tts auswählen">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "inworld",
        providers: {
          inworld: {
            voiceId: "Sarah",
            modelId: "inworld-tts-1.5-max",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Nachricht senden">
    Senden Sie über einen beliebigen verbundenen Kanal eine Antwort. OpenClaw synthetisiert das Audio mit Inworld und stellt es als MP3 bereit (oder als OGG_OPUS, wenn der Kanal eine Sprachnachricht erwartet).
  </Step>
</Steps>

## Konfigurationsoptionen

| Option        | Pfad                                | Beschreibung                                                        |
| ------------- | ----------------------------------- | ------------------------------------------------------------------- |
| `apiKey`      | `tts.providers.inworld.apiKey`      | Base64-Dashboard-Zugangsdaten. Fällt auf `INWORLD_API_KEY` zurück. |
| `baseUrl`     | `tts.providers.inworld.baseUrl`     | Überschreibt die Basis-URL der Inworld-API (Standard: `https://api.inworld.ai`). |
| `voiceId`     | `tts.providers.inworld.voiceId`     | Stimmenkennung (Standard: `Sarah`). Veralteter Alias: `speakerVoiceId`. |
| `modelId`     | `tts.providers.inworld.modelId`     | TTS-Modell-ID (Standard: `inworld-tts-1.5-max`).                       |
| `temperature` | `tts.providers.inworld.temperature` | Sampling-Temperatur, `0` (exklusiv) bis `2` (optional). |

## Hinweise

<AccordionGroup>
  <Accordion title="Authentifizierung">
    Inworld verwendet HTTP-Basic-Authentifizierung mit einer einzelnen Base64-codierten Zeichenfolge für die Zugangsdaten. Kopieren Sie sie unverändert aus dem Inworld-Dashboard. Der Provider sendet sie ohne weitere Codierung als `Authorization: Basic <apiKey>`. Codieren Sie sie daher nicht selbst mit Base64 und übergeben Sie kein Token im Bearer-Stil. Den gleichen Hinweis finden Sie unter [Hinweise zur TTS-Authentifizierung](/de/tools/tts#inworld-primary).
  </Accordion>
  <Accordion title="Modelle">
    Unterstützte Modell-IDs: `inworld-tts-1.5-max` (Standard), `inworld-tts-1.5-mini`, `inworld-tts-1-max`, `inworld-tts-1`.
  </Accordion>
  <Accordion title="Audioausgaben">
    Antworten verwenden standardmäßig MP3. Wenn das Kanalziel `voice-note` ist, fordert OpenClaw bei Inworld `OGG_OPUS` an, damit das Audio als native Sprachnachricht wiedergegeben wird. Die Telefoniesynthese verwendet unkomprimiertes `PCM` mit 22050 Hz, um die Telefonie-Bridge zu versorgen.
  </Accordion>
  <Accordion title="Benutzerdefinierte Endpunkte">
    Überschreiben Sie den API-Host mit `tts.providers.inworld.baseUrl`. Abschließende Schrägstriche werden entfernt, bevor Anfragen gesendet werden.
  </Accordion>
</AccordionGroup>

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Text-to-Speech" href="/de/tools/tts" icon="waveform-lines">
    Übersicht über TTS, Provider und die `tts`-Konfiguration.
  </Card>
  <Card title="Konfiguration" href="/de/gateway/configuration" icon="gear">
    Vollständige Konfigurationsreferenz einschließlich der Einstellungen für `tts`.
  </Card>
  <Card title="Provider" href="/de/providers" icon="grid">
    Alle unterstützten OpenClaw-Provider.
  </Card>
  <Card title="Fehlerbehebung" href="/de/help/troubleshooting" icon="wrench">
    Häufige Probleme und Schritte zur Fehlerdiagnose.
  </Card>
</CardGroup>
