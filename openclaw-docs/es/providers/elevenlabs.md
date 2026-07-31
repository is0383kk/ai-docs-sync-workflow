---
read_when:
    - Quieres usar la conversión de texto a voz de ElevenLabs en OpenClaw
    - Quiere usar la conversión de voz a texto de ElevenLabs Scribe para archivos de audio adjuntos
    - Se quiere usar la transcripción en tiempo real de ElevenLabs para Voice Call o Google Meet
summary: Usa la síntesis de voz de ElevenLabs, Scribe STT y la transcripción en tiempo real con OpenClaw
title: ElevenLabs
x-i18n:
    generated_at: "2026-07-26T05:53:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c570aab5fd3ca00e8ded8e3daa143cb199334d507461800ec0b6c1ab0b65c59
    source_path: providers/elevenlabs.md
    workflow: 16
---

OpenClaw usa ElevenLabs para la conversión de texto a voz, la conversión por lotes de voz a texto con Scribe
v2 y la conversión de voz a texto en streaming con Scribe v2 Realtime. El plugin está incluido y
habilitado de forma predeterminada; no se necesita ningún paso de `plugins install`.

| Capacidad                       | Superficie de OpenClaw                                                | Valor predeterminado      |
| ------------------------------- | -------------------------------------------------------------------- | ------------------------ |
| Conversión de texto a voz       | `tts` / `talk`                                                       | `eleven_multilingual_v2` |
| Conversión por lotes de voz a texto | `tools.media.audio`                                                  | `scribe_v2`              |
| Conversión de voz a texto en streaming | Streaming de Voice Call o `realtime.transcriptionProvider` de Google Meet | `scribe_v2_realtime`     |

## Autenticación

Configure `ELEVENLABS_API_KEY` en el entorno. También se acepta `XI_API_KEY` por
compatibilidad con las herramientas existentes de ElevenLabs.

```bash
export ELEVENLABS_API_KEY="..."
```

## Conversión de texto a voz

```json5
{
  tts: {
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        voiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

Configure `modelId` como `eleven_v3` para usar la conversión de texto a voz v3 de ElevenLabs. OpenClaw mantiene
`eleven_multilingual_v2` como valor predeterminado para las instalaciones existentes.

Los canales de voz de Discord usan el endpoint de conversión de texto a voz en streaming de ElevenLabs cuando ElevenLabs
es el proveedor de `voice.tts`/`tts` seleccionado: la reproducción comienza desde el
flujo de audio devuelto en lugar de esperar a que OpenClaw descargue primero todo el
archivo de audio. `latencyTier` se asigna al parámetro de consulta `optimize_streaming_latency`
de ElevenLabs para los modelos que lo aceptan; OpenClaw omite ese parámetro para
`eleven_v3`, que lo rechaza.

## Conversión de voz a texto

Use Scribe v2 para los archivos adjuntos de audio entrantes y los segmentos breves de voz grabados:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "elevenlabs", model: "scribe_v2" }],
      },
    },
  },
}
```

OpenClaw envía audio multiparte a `/v1/speech-to-text` de ElevenLabs con
`model_id: "scribe_v2"`. Las indicaciones de idioma se asignan a `language_code` cuando están presentes.

## Conversión de voz a texto en streaming

El plugin `elevenlabs` incluido registra Scribe v2 Realtime para la transcripción en streaming de Voice Call y
del modo agente de Google Meet.

| Ajuste              | Ruta de configuración                                                     | Valor predeterminado                               |
| ------------------- | ------------------------------------------------------------------------- | ------------------------------------------------- |
| Clave de API        | `plugins.entries.voice-call.config.streaming.providers.elevenlabs.apiKey` | Recurre a `ELEVENLABS_API_KEY` / `XI_API_KEY` |
| Modelo              | `...elevenlabs.modelId`                                                   | `scribe_v2_realtime`                              |
| Formato de audio    | `...elevenlabs.audioFormat`                                               | `ulaw_8000`                                       |
| Frecuencia de muestreo | `...elevenlabs.sampleRate`                                                | `8000`                                            |
| Estrategia de confirmación | `...elevenlabs.commitStrategy`                                            | `vad`                                             |
| Idioma              | `...elevenlabs.languageCode`                                              | (sin configurar)                                  |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "${ELEVENLABS_API_KEY}",
                audioFormat: "ulaw_8000",
                commitStrategy: "vad",
                languageCode: "en",
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
Voice Call recibe el contenido multimedia de Twilio como G.711 u-law a 8 kHz. El proveedor en tiempo real
de ElevenLabs usa `ulaw_8000` de forma predeterminada, por lo que las tramas de telefonía pueden reenviarse sin
transcodificación.
</Note>

Para el modo agente de Google Meet, configure
`plugins.entries.google-meet.config.realtime.transcriptionProvider` como
`"elevenlabs"` y configure el mismo bloque de proveedor en
`plugins.entries.google-meet.config.realtime.providers.elevenlabs`.

## Contenido relacionado

- [Conversión de texto a voz](/es/tools/tts)
- [Google Meet](/es/plugins/google-meet)
- [Selección de modelos](/es/concepts/model-providers)
