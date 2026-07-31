---
read_when:
    - Modificación del pipeline de medios o de los archivos adjuntos
summary: Reglas de gestión de imágenes y contenido multimedia para envíos, el Gateway y las respuestas del agente
title: Compatibilidad con imágenes y contenido multimedia
x-i18n:
    generated_at: "2026-07-26T05:10:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

El canal de WhatsApp se ejecuta sobre Baileys Web. Esta página abarca las reglas de gestión de contenido multimedia para los envíos, el Gateway y las respuestas del agente.

## Objetivos

- Enviar contenido multimedia con una leyenda opcional mediante `openclaw message send --media`.
- Permitir que las respuestas automáticas de la bandeja de entrada web incluyan contenido multimedia junto con texto.
- Mantener límites razonables y predecibles para cada tipo.

## Superficie de la CLI

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — adjunta contenido multimedia (imagen/audio/vídeo/documento); acepta rutas locales o URL. Es opcional; la leyenda puede estar vacía para los envíos que solo contienen contenido multimedia.
- `--gif-playback` — trata el contenido multimedia de vídeo como una reproducción GIF (solo WhatsApp).
- `--force-document` — envía el contenido multimedia como documento para evitar la compresión del canal (Telegram, WhatsApp); se aplica a imágenes, GIF y vídeos.
- `--reply-to <id>`, `--thread-id <id>`, `--pin`, `--silent` — opciones de entrega y organización en hilos compartidas con los envíos de solo texto.
- `--dry-run` — muestra la carga útil resuelta y omite el envío.
- `--json` — muestra el resultado como JSON: `{ action, channel, dryRun, handledBy, messageId?, payload }` (`payload` contiene el resultado del envío específico del canal, incluida cualquier referencia al contenido multimedia).

## Comportamiento del canal web de WhatsApp

- Entrada: ruta de archivo local **o** URL HTTP(S).
- Flujo: se carga en un búfer, se detecta el tipo de contenido multimedia y, después, se crea la carga útil saliente correspondiente a cada tipo:
  - **Imágenes:** se optimizan para que no superen `channels.whatsapp.mediaMaxMb` (valor predeterminado: 50MB). Las imágenes opacas se vuelven a comprimir como JPEG (la secuencia predeterminada de dimensiones comienza en 2048px y disminuye cuando se incumple reiteradamente el límite de tamaño); las imágenes con transparencia se conservan como PNG. Si el origen ya es un archivo JPEG/PNG/WebP aceptable que respeta los límites de tamaño y longitud de los lados, los bytes originales se conservan sin cambios en lugar de volver a comprimirse. Los GIF animados nunca se vuelven a codificar; solo se comprueba su tamaño.
  - **Audio/voz:** salvo que ya sea audio de voz nativo (`.ogg`/`.opus` o `audio/ogg`/`audio/opus`), el audio saliente se transcodifica mediante `ffmpeg` a Opus/OGG (48kHz mono, 64kbps, limitado a 20 minutos) antes de enviarse como nota de voz (`ptt: true`).
  - **Vídeo:** se transmite sin cambios hasta 16MB.
  - **Documentos:** cualquier otro contenido, hasta 100MB, conservando el nombre del archivo cuando esté disponible.
- Reproducción al estilo GIF de WhatsApp: envía un MP4 con `gifPlayback: true` (CLI: `--gif-playback`) para que los clientes móviles lo reproduzcan en bucle en línea.
- La detección MIME da prioridad a los bytes mágicos detectados, después a la extensión del archivo y, por último, a los encabezados de respuesta; un contenedor genérico detectado (`application/octet-stream`, `zip`) nunca prevalece sobre una asignación de extensión más específica (por ejemplo, XLSX frente a ZIP).
- La leyenda procede de `--message` o `reply.text`; se permite una leyenda vacía.
- Registro: el modo no detallado muestra `↩️`/`✅`; el modo detallado incluye el tamaño y la ruta/URL de origen.

<Note>
Las cifras anteriores de 16MB para audio/vídeo y 100MB para documentos son los valores predeterminados compartidos por tipo de contenido multimedia cuando no se proporciona un límite explícito de bytes. Los envíos de WhatsApp establecen un límite explícito mediante `channels.whatsapp.mediaMaxMb` (valor predeterminado: 50MB), que se aplica de manera uniforme a todos los tipos de esa cuenta.
</Note>

## Pipeline de respuestas automáticas

- `getReplyFromConfig` devuelve una carga útil de respuesta (o una matriz de cargas útiles) con `text?`, `mediaUrl?` y `mediaUrls?`, entre otros campos.
- Cuando hay contenido multimedia, el remitente web resuelve las rutas locales o las URL mediante el mismo Pipeline que `openclaw message send`.
- Si se proporcionan varias entradas de contenido multimedia, se envían de forma secuencial.

## Contenido multimedia entrante en comandos

- Cuando los mensajes web entrantes incluyen contenido multimedia, OpenClaw lo descarga en un archivo temporal y expone variables de plantilla:
  - `{{AttachmentUrl}}` — URL original o referencia del proveedor correspondiente al archivo adjunto actual.
  - `{{AttachmentPath}}` — ruta temporal local escrita antes de ejecutar el comando.
  - `{{AttachmentContentType}}` — tipo de contenido MIME.
  - `{{AttachmentDir}}` — directorio que contiene la ruta local.
  - `{{AttachmentIndex}}` — índice de origen basado en cero.
- Cuando se habilita un entorno aislado de Docker por sesión, el contenido multimedia entrante se copia en el espacio de trabajo del entorno aislado y la ruta/referencia del archivo adjunto se reescribe como una ruta relativa al entorno aislado, como `media/inbound/<filename>`.
- `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` y `{{MediaDir}}` siguen siendo alias de compatibilidad obsoletos durante el período de migración del SDK de plugins.
- La comprensión de contenido multimedia (configurada mediante `tools.media.*` o el elemento compartido `tools.media.models`) se ejecuta antes de aplicar las plantillas y puede insertar bloques `[Image]`, `[Audio]` y `[Video]` en `Body`.
  - El audio establece `{{Transcript}}` y utiliza la transcripción para analizar los comandos, de modo que los comandos de barra diagonal sigan funcionando.
  - Las descripciones de vídeos e imágenes conservan cualquier texto de la leyenda para analizar los comandos.
  - Si el modelo principal activo ya admite visión de forma nativa, OpenClaw omite el bloque de resumen `[Image]` y, en su lugar, pasa la imagen original al modelo.
- De forma predeterminada, solo se procesa el primer archivo adjunto coincidente de imagen/audio/vídeo; utilice `tools.media.<capability>.attachments` para seleccionar varios archivos adjuntos.

## Límites y errores

**Límites de envío saliente (envío web de WhatsApp)**

- Imágenes: hasta `channels.whatsapp.mediaMaxMb` (valor predeterminado: 50MB) después de la optimización.
- Audio/vídeo: límite de 16MB (valor predeterminado compartido; se reemplaza por `mediaMaxMb` al enviar mediante WhatsApp).
- Documentos: límite de 100MB (valor predeterminado compartido; se reemplaza por `mediaMaxMb` al enviar mediante WhatsApp).
- El contenido multimedia demasiado grande o ilegible genera un error claro en los registros y se omite la respuesta.

**Límites de comprensión de contenido multimedia (transcripción/descripción)**

- Valor predeterminado para imágenes: 10MB (se puede reemplazar con `tools.media.image.maxBytes` o, para cada
  entrada `tools.media.models[]`, con `maxBytes`).
- Valor predeterminado para audio: 20MB (se puede reemplazar con `tools.media.audio.maxBytes` o por entrada).
- Valor predeterminado para vídeo: 50MB (se puede reemplazar con `tools.media.video.maxBytes` o por entrada).
- El contenido multimedia demasiado grande omite la fase de comprensión, pero la respuesta sigue procesándose con el cuerpo original.

## Notas para las pruebas

- Cubrir los flujos de envío y respuesta para los casos de imagen, audio y documento.
- Validar los límites de tamaño después de la optimización de imágenes y el indicador de nota de voz para el audio.
- Garantizar que las respuestas con varios elementos multimedia se distribuyan como envíos secuenciales.

## Contenido relacionado

- [Captura de cámara](/es/nodes/camera)
- [Comprensión de contenido multimedia](/es/nodes/media-understanding)
- [Audio y notas de voz](/es/nodes/audio)
