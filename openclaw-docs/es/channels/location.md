---
read_when:
    - Añadir o modificar el análisis de ubicaciones de canales
    - Uso de campos de contexto de ubicación en prompts o herramientas del agente
summary: Análisis de ubicaciones de canales y cargas útiles de ubicación salientes portátiles
title: Análisis de la ubicación del canal
x-i18n:
    generated_at: "2026-07-26T04:30:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c7e5647d02643ad6d95024b362228377690d7fdff66441fae367f0f5307217fb
    source_path: channels/location.md
    workflow: 16
---

OpenClaw normaliza las ubicaciones compartidas desde los canales de chat en:

- texto conciso de coordenadas añadido al cuerpo entrante, y
- campos estructurados en la carga útil de contexto de respuesta automática. Las etiquetas, direcciones y leyendas/comentarios proporcionados por el canal se representan en el prompt mediante el bloque JSON compartido de metadatos no fiables, no en línea en el cuerpo del usuario.

Compatibilidad actual:

- **LINE** (mensajes de ubicación con título/dirección)
- **Matrix** (`m.location` con `geo_uri`)
- **Telegram** (marcadores de ubicación + lugares + ubicaciones en directo)
- **WhatsApp** (`locationMessage` + `liveLocationMessage`)

## Formato del texto

Las ubicaciones se representan como líneas legibles sin corchetes. Las coordenadas usan seis decimales; la precisión se redondea a metros enteros:

- Marcador:
  - `📍 48.858844, 2.294351 ±12m`
- Lugar con nombre (en la misma línea; el nombre y la dirección solo se incluyen en el bloque de metadatos):
  - `📍 48.858844, 2.294351 ±12m`
- Ubicación compartida en directo:
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

Si el canal incluye una etiqueta, una dirección o una leyenda/comentario, se conserva en la carga útil de contexto y aparece en el prompt como JSON no fiable delimitado (los campos se omiten cuando no están presentes):

````text
Ubicación (metadatos no fiables):
```json
{
  "latitude": 48.858844,
  "longitude": 2.294351,
  "accuracy_m": 12,
  "source": "place",
  "name": "Torre Eiffel",
  "address": "Campo de Marte, París",
  "caption": "Nos vemos aquí"
}
```
````

## Campos de contexto

Cuando hay una ubicación, estos campos se añaden a `ctx`:

- `LocationLat` (número)
- `LocationLon` (número)
- `LocationAccuracy` (número, metros; opcional)
- `LocationName` (cadena; opcional)
- `LocationAddress` (cadena; opcional)
- `LocationSource` (`pin | place | live`)
- `LocationIsLive` (booleano)
- `LocationCaption` (cadena; opcional)

Cuando el canal no establece un origen explícito, OpenClaw lo infiere: las ubicaciones compartidas en directo se convierten en `live`, las ubicaciones con nombre o dirección se convierten en `place` y todas las demás son `pin`.

El renderizador de prompts trata `LocationName`, `LocationAddress` y `LocationCaption` como metadatos no fiables y los serializa mediante la misma ruta JSON acotada que se utiliza para otros contextos de canal.

## Cargas útiles salientes

La herramienta de mensajes y el SDK de Plugin utilizan la misma estructura `NormalizedLocation` para ubicaciones salientes portátiles. Una carga útil que solo contiene coordenadas representa un marcador. Los canales con compatibilidad nativa con lugares pueden asignar `name` más `address` a una tarjeta de lugar.

Actualmente, Telegram expone esta funcionalidad mediante `message(action="send")`. Su primera implementación es deliberadamente independiente: las cargas útiles de ubicación no pueden combinarse con texto ni contenido multimedia, y los pares de lugar incompletos generan un error en lugar de descartar silenciosamente un nombre o una dirección. Los canales no compatibles no anuncian el parámetro de ubicación.

## Notas sobre los canales

- **LINE**: los campos `title`/`address` del mensaje de ubicación se asignan a `LocationName`/`LocationAddress`; no admite ubicaciones en directo.
- **Matrix**: `geo_uri` se analiza como una ubicación de marcador; el parámetro `u` (incertidumbre) se asigna a `LocationAccuracy`, el cuerpo del evento rellena `LocationCaption`, la altitud se ignora y `LocationIsLive` siempre es falso.
- **Telegram**: los lugares se asignan a `LocationName`/`LocationAddress`; las ubicaciones en directo se detectan mediante `live_period`.
- **WhatsApp**: `locationMessage.comment` y `liveLocationMessage.caption` rellenan `LocationCaption`.

## Contenido relacionado

- [Comando de ubicación (nodos)](/es/nodes/location-command)
- [Captura de cámara](/es/nodes/camera)
- [Comprensión de contenido multimedia](/es/nodes/media-understanding)
