---
read_when:
    - Cambiar la transcripción de audio o el manejo de contenido multimedia
summary: Cómo se descargan, transcriben e incorporan en las respuestas los audios y las notas de voz entrantes
title: Audio y notas de voz
x-i18n:
    generated_at: "2026-07-26T04:44:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4076e3e55eb5c6dcc94cfdd842619697c8d756b924956d7b266d18446b4dd9be
    source_path: nodes/audio.md
    workflow: 16
---

## Qué hace

Cuando la comprensión de audio está habilitada (o se detecta automáticamente), OpenClaw:

1. Localiza el primer archivo adjunto de audio (ruta local o URL) y lo descarga si es necesario.
2. Aplica `maxBytes` antes de enviarlo a cada entrada de modelo.
3. Ejecuta en orden la primera entrada de modelo apta (proveedor o CLI); si una entrada falla o se omite (tamaño/tiempo de espera), se prueba la siguiente.
4. Si se completa correctamente, sustituye `Body` por un bloque `[Audio]` y establece `{{Transcript}}`.

Cuando la transcripción se completa correctamente, `CommandBody`/`RawBody` también se establecen en la transcripción para que los comandos con barra sigan funcionando. Con `--verbose`, los registros muestran cuándo se ejecuta la transcripción y cuándo sustituye el cuerpo.

## Detección automática (predeterminada)

Si no se han configurado modelos y `tools.media.audio.enabled` no es `false`, OpenClaw realiza la detección automática en este orden y se detiene en la primera opción que funcione:

1. **Modelo de respuesta activo**, cuando su proveedor admite la comprensión de audio.
2. **Autenticación de proveedor configurada**: cualquier entrada `models.providers.*` con autenticación disponible para un proveedor que admita la transcripción de audio. Esto se comprueba antes que las CLI locales, por lo que una clave de API configurada siempre tiene prioridad sobre un binario local en `PATH`.
   Prioridad de proveedores cuando hay varios configurados: Groq, OpenAI, xAI, Deepgram, Google, SenseAudio, ElevenLabs, Mistral.
3. **CLI locales** (solo si no se resolvió ninguna autenticación de proveedor). OpenClaw crea una lista ordenada de alternativas:
   - `whisper-cli`, antes que las opciones predeterminadas de CPU solo cuando una invocación anterior de un modelo en el proceso actual detectó Metal o CUDA
   - `sherpa-onnx-offline` en su proveedor de CPU predeterminado (requiere `SHERPA_ONNX_MODEL_DIR` con `tokens.txt`, `encoder.onnx`, `decoder.onnx` y `joiner.onnx`)
   - `whisper-cli` cuando Metal/CUDA solo está disponible para la compilación o, de otro modo, no se ha detectado el backend seleccionado
   - `parakeet-mlx` en Apple Silicon (compatible con MLX; el uso del dispositivo sigue sin observarse)
   - `whisper` (CLI de Python; descarga los modelos automáticamente)

La procedencia de la instalación o el enlace constituye evidencia de capacidad, no de ejecución. Por sí sola, nunca antepone un candidato a sherpa en CPU. OpenClaw no carga un modelo durante la configuración ni las comprobaciones de estado únicamente para sondear un backend.
La detección automática de whisper.cpp mantiene habilitados sus registros normales de ejecución del modelo para que OpenClaw pueda registrar la línea `using … backend` del componente de origen. Las entradas de CLI explícitas conservan sus indicadores de salida configurados.

La detección automática de Gemini CLI para la comprensión multimedia se sustituyó por una alternativa de Antigravity CLI aislada (`agy`) para imágenes y vídeo; el audio no usa ninguna alternativa de CLI aparte de los binarios locales anteriores.

Para deshabilitar la detección automática, establezca `tools.media.audio.enabled: false`. Para personalizarla, añada entradas con etiquetas de capacidad a `tools.media.models`.

<Note>
La detección de binarios se realiza con el máximo esfuerzo posible en macOS/Linux/Windows. Asegúrese de que la CLI esté en `PATH` (`~` se expande), o establezca un modelo de CLI explícito con la ruta completa del comando.
</Note>

Inspeccione la selección local sin transcribir audio:

```bash
openclaw capability audio providers
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

El inventario de proveedores informa del ganador de la alternativa local por separado de la selección global de proveedores, además de los campos de backend disponible, solicitado y detectado. Después de ejecutar la transcripción, `/status` informa del backend solicitado o detectado en la línea multimedia. Las entradas de CLI `tools.media.models` explícitas compatibles con audio siguen omitiendo la selección automática; use sus indicadores específicos del backend, como `--provider=cuda` de sherpa o `--no-gpu`/`--device` de whisper.cpp.

## Ejemplos de configuración

### Proveedor + alternativa de CLI (OpenAI + Whisper CLI)

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          timeoutSeconds: 45,
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-transcribe" },
    },
  },
}
```

### Solo proveedor (Deepgram)

```json5
{
  tools: {
    media: {
      models: [{ provider: "deepgram", model: "nova-3", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Solo proveedor (Mistral Voxtral)

```json5
{
  tools: {
    media: {
      models: [{ provider: "mistral", model: "voxtral-mini-latest", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Solo proveedor (SenseAudio)

```json5
{
  tools: {
    media: {
      models: [
        {
          provider: "senseaudio",
          model: "senseaudio-asr-pro-1.5-260319",
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true },
    },
  },
}
```

### Repetir la transcripción en el chat (opcional)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
    },
  },
}
```

## Notas y límites

- La autenticación del proveedor sigue el orden estándar de autenticación de modelos (perfiles de autenticación, variables de entorno, `models.providers.*.apiKey`).
- Detalles de configuración de Groq: [Groq](/es/providers/groq).
- Deepgram utiliza `DEEPGRAM_API_KEY` cuando se usa `provider: "deepgram"`. Detalles de configuración: [Deepgram](/es/providers/deepgram).
- Detalles de configuración de Mistral: [Mistral](/es/providers/mistral).
- SenseAudio utiliza `SENSEAUDIO_API_KEY` cuando se usa `provider: "senseaudio"`. Detalles de configuración: [SenseAudio](/es/providers/senseaudio).
- Los proveedores de audio pueden usar los valores predeterminados de `tools.media.audio` o sustituir `baseUrl`, `headers`, `providerOptions` y los límites en su entrada `tools.media.models[]`.
- El límite de tamaño de audio integrado es de 20MB. Una sustitución de `maxBytes` en el nivel de entrada puede cambiarlo; el audio que supere el tamaño se omite para ese modelo y se prueba la siguiente entrada.
- Los archivos de audio de menos de 1024 bytes se omiten antes de la transcripción mediante un proveedor o una CLI.
- El valor predeterminado de `maxChars` para audio está **sin establecer** (transcripción completa). Establezca `tools.media.audio.maxChars` o `maxChars` por entrada para recortar la salida.
- El valor predeterminado de la detección automática de OpenAI es `gpt-4o-transcribe`; establezca `model: "gpt-4o-mini-transcribe"` para obtener una opción más económica y rápida.
- La transcripción está disponible para las plantillas como `{{Transcript}}`.
- `tools.media.audio.echoTranscript` está desactivado de forma predeterminada; `echoFormat` acepta un marcador de posición `{transcript}`.
- La salida estándar de la CLI está limitada a 5MB; mantenga concisa la salida de la CLI.
- `args` de la CLI debe usar `{{AttachmentPath}}` para la ruta del archivo de audio local. Ejecute `openclaw doctor --fix` para migrar los marcadores de posición `{input}` obsoletos de configuraciones `audio.transcription.command` anteriores (clave retirada: `audio.transcription`, sustituida por `tools.media.models`). `{{MediaPath}}` sigue siendo un alias de compatibilidad obsoleto.
- `tools.media.concurrency` limita las tareas multimedia; no es un planificador de GPU.

### STT local residente

El STT local detectado automáticamente sigue ejecutándose en un proceso por solicitud. Actualmente, OpenClaw no administra un servidor whisper.cpp residente porque el paquete estándar `whisper-cpp` de Homebrew deshabilita ese servidor, mientras que el ejemplo del proyecto de origen no tiene configurada una cola de admisión limitada. Para poder habilitar de forma segura un ciclo de vida residente propiedad de un Plugin, se necesita un worker empaquetado y mantenido con comprobación de estado/inicio, residencia del modelo, colas limitadas, cancelación/tiempo de espera, funcionamiento sin autenticación limitado a la interfaz de bucle invertido y sin alternativa en la nube.

### Compatibilidad con variables de entorno de proxy

La transcripción de audio basada en proveedores respeta las variables de entorno estándar de proxy de salida, de acuerdo con la semántica de `EnvHttpProxyAgent` de undici:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `ALL_PROXY` / `all_proxy`

Las variables en minúsculas tienen prioridad sobre las de mayúsculas; las entradas `NO_PROXY`/`no_proxy` (nombres de host, `*.suffix` o `host:port`) omiten el proxy. Si no se establece ninguna variable de entorno de proxy, se usa una salida directa. Si falla la configuración del proxy (URL con formato incorrecto), OpenClaw registra una advertencia y recurre a la solicitud directa.

## Detección de menciones en grupos

En los canales que admiten una comprobación previa de audio, OpenClaw transcribe el audio **antes** de comprobar las menciones cuando `requireMention: true` está establecido para un chat grupal. Esto permite que una nota de voz sin texto supere el control de menciones cuando su transcripción contiene un patrón de mención configurado. La documentación específica de cada canal describe los transportes que requieren una mención escrita.

**Cómo funciona:**

1. Si un mensaje de voz no tiene cuerpo de texto y el grupo requiere menciones, OpenClaw realiza una transcripción previa del primer archivo adjunto de audio.
2. Se comprueba si la transcripción contiene patrones de mención (por ejemplo, `@BotName` o activadores con emojis).
3. Si se encuentra una mención, el mensaje continúa por el Pipeline de respuesta completo.

**Comportamiento alternativo:** si falla la transcripción previa (tiempo de espera, error de API, etc.), el mensaje recurre a la detección de menciones únicamente en el texto, de modo que los mensajes mixtos (texto + audio) nunca se descarten.

**Desactivación por grupo/tema de Telegram:**

- Establezca `channels.telegram.groups.<chatId>.disableAudioPreflight: true` para omitir las comprobaciones previas de menciones en la transcripción para ese grupo.
- Establezca `channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight` para sustituir el valor por tema (`true` para omitirlo, `false` para habilitarlo de manera forzada).
- El valor predeterminado es `false` (la comprobación previa se habilita cuando se cumplen las condiciones del control de menciones).

**Ejemplo:** una persona envía una nota de voz que dice «Oye, @Claude, ¿qué tiempo hace?» en un grupo de Telegram con `requireMention: true`. La nota de voz se transcribe, se detecta la mención y el agente responde.

## Aspectos importantes

- Las reglas de ámbito usan la primera coincidencia; `chatType` se normaliza como `direct`, `group` o `channel`.
- Asegúrese de que la CLI termine con el código 0 e imprima texto sin formato; la salida JSON debe adaptarse mediante `jq -r .text`.
- Los modos conocidos de salida a archivos son autoritativos: si falta el archivo de transcripción inferido o está vacío, no se genera ninguna transcripción en lugar de recurrir a la salida de progreso de la CLI.
- Para `parakeet-mlx`, use `--output-format txt` (o `all`) con `--output-dir` y la plantilla de salida predeterminada `{filename}`. También se respetan las variables de entorno `PARAKEET_OUTPUT_FORMAT` y `PARAKEET_OUTPUT_TEMPLATE` del proyecto de origen. OpenClaw lee `<output-dir>/<media-basename>.txt`; el formato predeterminado `srt`, los demás formatos y las plantillas de salida personalizadas siguen usando la salida estándar.
- Mantenga tiempos de espera razonables (`timeoutSeconds`, 60s de forma predeterminada) para evitar bloquear la cola de respuestas.
- La transcripción previa solo procesa el **primer** archivo adjunto de audio para detectar menciones. Los archivos adjuntos de audio adicionales se procesan durante la fase principal de comprensión multimedia.

## Contenido relacionado

- [Comprensión multimedia](/es/nodes/media-understanding)
- [Modo de conversación](/es/nodes/talk)
- [Activación por voz](/es/nodes/voicewake)
