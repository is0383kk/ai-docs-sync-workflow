---
read_when:
    - Se desea usar la síntesis de voz de Azure para las respuestas salientes
    - Necesita una salida nativa de notas de voz Ogg Opus de Azure Speech
summary: Texto a voz de Azure AI Speech para las respuestas de OpenClaw
title: Voz de Azure
x-i18n:
    generated_at: "2026-07-26T05:25:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfeeb9daa8d7d6aa24e497d57d64e07efa94c3c0c6b16f793343a450286ab3c1
    source_path: providers/azure-speech.md
    workflow: 16
---

Azure Speech es un proveedor de texto a voz de Azure AI Speech incluido. OpenClaw
llama directamente a la API REST de Azure Speech con SSML y sintetiza MP3 para
respuestas estándar, Ogg/Opus nativo para notas de voz y mulaw de 8 kHz para
canales de telefonía como las llamadas de voz. La solicitud envía el formato de
salida gestionado por el proveedor mediante el encabezado `X-Microsoft-OutputFormat`.

| Detalle                 | Valor                                                                                                          |
| ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| ID del proveedor        | `azure-speech` (alias: `azure`)                                                                                |
| Sitio web               | [Azure AI Speech](https://azure.microsoft.com/products/ai-services/ai-speech)                                  |
| Documentación           | [Texto a voz mediante REST de Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech) |
| Autenticación           | `AZURE_SPEECH_KEY` más `AZURE_SPEECH_REGION`                                                                  |
| Voz predeterminada      | `en-US-JennyNeural`                                                                                            |
| Archivo de salida predeterminado | `audio-24khz-48kbitrate-mono-mp3`                                                                              |
| Archivo de nota de voz predeterminado | `ogg-24khz-16bit-mono-opus`                                                                                    |

## Primeros pasos

<Steps>
  <Step title="Crear un recurso de Azure Speech">
    En el portal de Azure, cree un recurso de Speech. Copie **KEY 1** de
    Resource Management > Keys and Endpoint y copie la ubicación del recurso,
    como `eastus`.

    ```
    AZURE_SPEECH_KEY=<speech-resource-key>
    AZURE_SPEECH_REGION=eastus
    ```

  </Step>
  <Step title="Seleccionar Azure Speech en tts">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "azure-speech",
        providers: {
          "azure-speech": {
            voice: "en-US-JennyNeural",
            lang: "en-US",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Enviar un mensaje">
    Envíe una respuesta mediante cualquier canal conectado. OpenClaw sintetiza
    el audio con Azure Speech y entrega MP3 para audio estándar u Ogg/Opus
    cuando el canal espera una nota de voz.
  </Step>
</Steps>

## Opciones de configuración

Todas las opciones se encuentran en `tts.providers["azure-speech"]`.

| Opción                  | Descripción                                                                                           |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| `apiKey`                | Clave del recurso de Azure Speech. Utiliza como alternativa `AZURE_SPEECH_KEY`, `AZURE_SPEECH_API_KEY` o `SPEECH_KEY`. |
| `region`                | Región del recurso de Azure Speech. Utiliza como alternativa `AZURE_SPEECH_REGION` o `SPEECH_REGION`.                 |
| `endpoint`              | Sustitución opcional del endpoint de Azure Speech. Utiliza como alternativa el valor de confianza `AZURE_SPEECH_ENDPOINT`.               |
| `baseUrl`               | Sustitución opcional de la URL base de Azure Speech.                                                              |
| `voice`                 | ShortName de la voz de Azure (valor predeterminado: `en-US-JennyNeural`). Alias heredado: `voiceId`.                         |
| `lang`                  | Código de idioma SSML (valor predeterminado: `en-US`).                                                                 |
| `outputFormat`          | Formato de salida del archivo de audio (valor predeterminado: `audio-24khz-48kbitrate-mono-mp3`).                                 |
| `voiceNoteOutputFormat` | Formato de salida de la nota de voz (valor predeterminado: `ogg-24khz-16bit-mono-opus`).                                       |
| `timeoutMs`             | Sustitución del tiempo de espera de la solicitud en milisegundos. Utiliza como alternativa el valor global `tts.timeoutMs`.                   |

El proveedor se considera configurado cuando se establece `apiKey` junto con uno de
`region`, `endpoint` o `baseUrl`. Las variables de entorno solo se consultan como alternativa
para las claves de configuración que no estén establecidas. Los archivos `.env` del espacio de trabajo no pueden establecer
`AZURE_SPEECH_ENDPOINT`; utilice el entorno del proceso, el archivo dotenv del entorno de ejecución global
o una configuración explícita para el enrutamiento de endpoints.

## Notas

<AccordionGroup>
  <Accordion title="Autenticación">
    Azure Speech utiliza una clave de recurso de Speech, no una clave de Azure OpenAI. La clave
    se envía como `Ocp-Apim-Subscription-Key`; OpenClaw deriva
    `https://<region>.tts.speech.microsoft.com` de `region`, a menos que se
    proporcione `endpoint` o `baseUrl`.
  </Accordion>
  <Accordion title="Nombres de voz">
    Utilice el valor `ShortName` de la voz de Azure Speech, por ejemplo,
    `en-US-JennyNeural`. El proveedor incluido puede enumerar las voces mediante el
    mismo recurso de Speech y excluye las voces marcadas como obsoletas, retiradas
    o deshabilitadas.
  </Accordion>
  <Accordion title="Salidas de audio">
    Azure acepta formatos de salida como `audio-24khz-48kbitrate-mono-mp3`,
    `ogg-24khz-16bit-mono-opus` y `riff-24khz-16bit-mono-pcm`. OpenClaw
    solicita Ogg/Opus para los destinos `voice-note` a fin de que los canales puedan enviar burbujas
    de voz nativas sin una conversión adicional a MP3 y fuerza
    `raw-8khz-8bit-mono-mulaw` para los destinos de telefonía.
  </Accordion>
  <Accordion title="Alias">
    `azure` se acepta como alias del proveedor para configuraciones existentes, pero las configuraciones
    nuevas deben utilizar `azure-speech` para evitar confusiones con los proveedores
    de modelos de Azure OpenAI.
  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Texto a voz" href="/es/tools/tts" icon="waveform-lines">
    Descripción general de TTS, proveedores y configuración de `tts`.
  </Card>
  <Card title="Configuración" href="/es/gateway/configuration" icon="gear">
    Referencia completa de configuración, incluidos los ajustes de `tts`.
  </Card>
  <Card title="Proveedores" href="/es/providers" icon="grid">
    Todos los proveedores de OpenClaw incluidos.
  </Card>
  <Card title="Solución de problemas" href="/es/help/troubleshooting" icon="wrench">
    Problemas comunes y pasos de depuración.
  </Card>
</CardGroup>
