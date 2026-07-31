---
read_when:
    - Creación de clientes de Matrix que renderizan respuestas enriquecidas de OpenClaw
    - Depuración del contenido de eventos de com.openclaw.presentation
summary: Metadatos de MessagePresentation de Matrix para clientes compatibles con OpenClaw
title: Metadatos de presentación de Matrix
x-i18n:
    generated_at: "2026-07-26T04:31:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0de4d13c6cefc6f91dcc7a4b0edeea6bf001f3bd71f52c9f0498ad422783d8a
    source_path: channels/matrix-presentation.md
    workflow: 16
---

OpenClaw adjunta metadatos normalizados de `MessagePresentation` a los eventos salientes de Matrix `m.room.message` bajo la clave de contenido `com.openclaw.presentation`.

Los clientes estándar de Matrix siguen mostrando el texto sin formato `body`. Los clientes compatibles con OpenClaw pueden leer los metadatos estructurados y mostrar una interfaz de usuario nativa, como botones, selectores, filas de contexto y divisores.

## Contenido del evento

```json
{
  "msgtype": "m.text",
  "body": "Seleccionar modelo\n\nElija un modelo:\n- DeepSeek",
  "com.openclaw.presentation": {
    "version": 1,
    "type": "message.presentation",
    "title": "Seleccionar modelo",
    "tone": "info",
    "blocks": [
      {
        "type": "select",
        "placeholder": "Elija un modelo",
        "options": [
          {
            "label": "DeepSeek",
            "value": "/model deepseek/deepseek-chat"
          }
        ]
      }
    ]
  }
}
```

- `version` es la versión del esquema de metadatos; la versión actual es `1`. `type` es un discriminador estable, siempre `"message.presentation"`. El adaptador de Matrix solo emite cargas útiles con exactamente esta versión y este tipo; del mismo modo, los clientes deben ignorar las versiones desconocidas que no puedan interpretar de forma segura, los valores desconocidos de `type` y los tipos de bloque desconocidos.
- `title` y `tone` (`info`, `success`, `warning`, `danger`, `neutral`) son indicaciones opcionales.
- Los botones y las opciones de selección pueden incluir un `action` tipado (`{ "type": "command", "command": "/..." }` o `{ "type": "callback", "value": "..." }`) junto con la cadena heredada `value`. Se debe dar preferencia a `action` cuando ambos estén presentes.

## Comportamiento alternativo

OpenClaw siempre genera una alternativa legible en texto sin formato en `body`. Los metadatos estructurados son complementarios y no deben ser necesarios para la interoperabilidad básica con Matrix.

Reglas de representación alternativa:

- El contenido de `title`, `text` y `context` se representa como líneas de texto sin formato.
- Los botones con una acción `command` se representan como ``label: `/command` `` para que el comando se pueda copiar. Los botones con una acción `callback` o únicamente con un `value` heredado se representan solo con la etiqueta para que los valores opacos de devolución de llamada permanezcan privados; los botones deshabilitados siempre se representan solo con la etiqueta. Los botones de URL y de aplicaciones web se representan como `label: URL`.
- Los bloques de selección representan el marcador de posición (o `Options:`) como encabezado, seguido de líneas de opciones que solo contienen las etiquetas.
- Si no se representa nada, por ejemplo, una presentación que solo contiene divisores, el cuerpo recurre a `---`.

Los clientes no compatibles siguen mostrando el texto alternativo. Los clientes compatibles con OpenClaw pueden dar preferencia a los metadatos estructurados para la visualización y conservar el texto alternativo para copiar, buscar, mostrar notificaciones y facilitar la accesibilidad.

## Bloques compatibles

El adaptador de salida de Matrix anuncia compatibilidad nativa con:

- `buttons`
- `select`
- `context`
- `divider`

Los bloques `text` siempre son compatibles mediante el cuerpo alternativo. Todos los bloques deben tratarse como indicaciones de presentación basadas en el mejor esfuerzo; se deben ignorar los campos y tipos de bloque desconocidos en lugar de hacer que falle el mensaje completo.

## Interacciones

Estos metadatos no añaden semántica de devolución de llamada a Matrix. Los valores de los botones y las selecciones son cargas útiles de interacción alternativas, normalmente comandos con barra diagonal o comandos de texto. Un cliente de Matrix que quiera admitir la interacción resuelve el valor del control (`action.command`, después `action.value` y, por último, `value`) y lo devuelve a la sala como un mensaje normal.

Por ejemplo, un botón con el valor `/model deepseek/deepseek-chat` puede gestionarse enviando ese valor como un mensaje de texto cifrado de Matrix en la misma sala.

## Relación con los metadatos de aprobación

`com.openclaw.presentation` se utiliza para la presentación general de mensajes enriquecidos.

Las solicitudes de aprobación utilizan los metadatos específicos `com.openclaw.approval`, ya que las aprobaciones incluyen estados y decisiones críticos para la seguridad, así como detalles de ejecución y de plugins. Si ambas claves de metadatos están presentes en el mismo evento, los clientes deben dar preferencia al representador específico de aprobaciones.

## Mensajes multimedia

Cuando una respuesta contiene varias URL de contenido multimedia, OpenClaw envía un evento de Matrix por cada URL. El texto del pie y los metadatos de presentación se adjuntan únicamente al primer evento, de modo que los clientes reciban una sola carga útil estructurada y estable sin representadores duplicados. La misma regla se aplica cuando un texto largo se divide entre varios eventos: los metadatos solo se incluyen en el primero.

Los metadatos de presentación deben mantenerse compactos. El texto visible para el usuario que sea extenso debe permanecer en `body` y utilizar la ruta normal de división de texto de Matrix.
