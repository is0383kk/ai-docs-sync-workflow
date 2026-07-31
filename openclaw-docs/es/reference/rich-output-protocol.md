---
read_when:
    - Cambiar la representación de la salida del asistente en la interfaz de control
    - Depuración de `[embed ...]`, medios estructurados, respuestas o directivas de presentación de audio
summary: Protocolo de salida enriquecida para contenido multimedia estructurado, elementos insertados, indicaciones de audio y respuestas
title: Protocolo de salida enriquecida
x-i18n:
    generated_at: "2026-07-26T04:58:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbfe68f38c871f5f6d2811eb52b18d0143606f30283023ae96db64543eed95a1
    source_path: reference/rich-output-protocol.md
    workflow: 16
---

La salida del asistente transporta directivas de entrega/renderizado a través de unos cuantos canales dedicados:

- Campos estructurados `mediaUrl` / `mediaUrls` para la entrega de archivos adjuntos.
- `[[audio_as_voice]]` para indicaciones de presentación de audio.
- `[[reply_to_current]]` / `[[reply_to:<id>]]` para metadatos de respuesta.
- `[embed ...]` para el renderizado enriquecido de la interfaz de control.

Los campos multimedia estructurados y las etiquetas `[[...]]` son metadatos de entrega. `[embed ...]` es la ruta independiente de renderizado enriquecido exclusiva de la web; no es un alias multimedia.

## Archivos multimedia adjuntos

Los archivos adjuntos remotos deben ser URL `https:` públicas. `http:`, los nombres de host de bucle invertido, de enlace local, privados e internos se rechazan como directivas de archivos adjuntos; los recuperadores multimedia del lado del servidor aplican además sus propias protecciones de red.

Los archivos adjuntos locales aceptan rutas absolutas, rutas relativas al espacio de trabajo o rutas `~/` relativas al directorio de inicio. Aun así, antes de la entrega se someten a la política de lectura de archivos del agente y a las comprobaciones del tipo de contenido multimedia.

<Warning>
No se deben emitir comandos de texto para archivos adjuntos desde herramientas, plugins, bloques de streaming, la salida del navegador ni acciones de mensajes. Se deben usar en su lugar campos multimedia estructurados:

```json
{ "message": "Aquí está su imagen.", "mediaUrl": "/workspace/image.png" }
```

El texto heredado de la respuesta final aún puede normalizarse por compatibilidad, pero este no es un protocolo general para plugins ni herramientas.
</Warning>

La sintaxis de imagen de Markdown sin formato (`![alt](url)`) permanece como texto de forma predeterminada. Los canales que quieran tratar las imágenes de Markdown como respuestas multimedia deben habilitarlo en su adaptador de salida; Telegram lo hace, por lo que `![alt](url)` se convierte en un archivo multimedia adjunto.

Cuando el streaming por bloques está habilitado, el contenido multimedia debe transportarse en campos de carga estructurados. Si la misma URL multimedia aparece en un bloque transmitido y de nuevo en la carga final del asistente, OpenClaw la entrega una sola vez y elimina el duplicado de la carga final.

## `[embed ...]`

`[embed ...]` es la única sintaxis de renderizado enriquecido orientada al agente para la interfaz de control. Ejemplo de cierre automático:

```text
[embed ref="cv_123" title="Status" /]
```

Reglas:

- `[view ...]` ya no es válido para salidas nuevas.
- Los códigos cortos de contenido incrustado solo se renderizan en la superficie de mensajes del asistente.
- Solo se renderiza contenido incrustado respaldado por URL; se debe usar `ref="..."` o `url="..."`.
- Los códigos cortos de contenido HTML en línea incrustado con formato de bloque no se renderizan.
- La interfaz web elimina el código corto del texto visible y renderiza el contenido incrustado en línea.

## Estructura de renderizado almacenada

El bloque normalizado/almacenado de contenido del asistente es un elemento estructurado `canvas`:

```json
{
  "type": "canvas",
  "preview": {
    "kind": "canvas",
    "surface": "assistant_message",
    "render": "url",
    "viewId": "cv_123",
    "url": "/__openclaw__/canvas/documents/cv_123/index.html",
    "title": "Status",
    "preferredHeight": 320
  }
}
```

`present_view` no se reconoce; los bloques enriquecidos almacenados/renderizados siempre usan esta estructura `canvas`.

## Relacionado

- [Adaptadores RPC](/es/reference/rpc)
- [Typebox](/es/concepts/typebox)
