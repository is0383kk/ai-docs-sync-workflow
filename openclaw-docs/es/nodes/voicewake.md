---
read_when:
    - Cambiar el comportamiento o los valores predeterminados de las palabras de activación por voz
    - Adición de nuevas plataformas Node que necesitan sincronización de la palabra de activación
summary: Palabras de activación por voz globales (gestionadas por el Gateway) y cómo se sincronizan entre nodos
title: Activación por voz
x-i18n:
    generated_at: "2026-07-26T05:11:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aef2a5bba664ce10fb6ab457bb6d202639dcc6c0a9df61567e7cb402c290bbec
    source_path: nodes/voicewake.md
    workflow: 16
---

Las palabras de activación son **una única lista global propiedad del Gateway**; no hay listas personalizadas por Node. Cualquier Node o interfaz de usuario de una aplicación puede editar la lista; el Gateway conserva el cambio y lo difunde a todos los clientes conectados.

- **macOS**: control local para activar o desactivar Voice Wake. Requiere macOS 26+; consulte [Activación por voz (macOS)](/es/platforms/mac/voicewake) para obtener detalles sobre el entorno de ejecución/PTT.
- **iOS**: control local para activar o desactivar Voice Wake en Settings.
- **Android**: control local para activar o desactivar Voice Wake y editor de palabras de activación en Settings → Voice. Requiere el reconocimiento de voz integrado en el dispositivo Android.

## Almacenamiento

Las palabras de activación y las reglas de enrutamiento se encuentran en la base de datos de estado del Gateway, `~/.openclaw/state/openclaw.sqlite` de forma predeterminada (se puede sustituir con `OPENCLAW_STATE_DIR`), en las tablas `voicewake_triggers`, `voicewake_routing_config` y `voicewake_routing_routes`. Los archivos heredados `settings/voicewake.json` y `settings/voicewake-routing.json` son únicamente entradas de migración de `openclaw doctor --fix`; el entorno de ejecución nunca los lee.

## Protocolo

### Lista de activadores

| Método          | Parámetros               | Resultado                |
| --------------- | ------------------------ | ------------------------ |
| `voicewake.get` | ninguno                  | `{ triggers: string[] }` |
| `voicewake.set` | `{ triggers: string[] }` | `{ triggers: string[] }` |

`voicewake.set` normaliza la entrada: elimina los espacios en blanco iniciales y finales, descarta las entradas vacías, conserva un máximo de 32 activadores y trunca cada uno a 64 unidades de código UTF-16 sin dividir pares suplentes. Si el resultado está vacío, se usan los valores predeterminados integrados (`openclaw`, `claude`, `computer`).

### Enrutamiento (del activador al destino)

| Método                  | Parámetros                           | Resultado                            |
| ----------------------- | ------------------------------------ | ------------------------------------ |
| `voicewake.routing.get` | ninguno                              | `{ config: VoiceWakeRoutingConfig }` |
| `voicewake.routing.set` | `{ config: VoiceWakeRoutingConfig }` | `{ config: VoiceWakeRoutingConfig }` |

```json
{
  "version": 1,
  "defaultTarget": { "mode": "current" },
  "routes": [{ "trigger": "activación del robot", "target": { "sessionKey": "agent:main:main" } }],
  "updatedAtMs": 1730000000000
}
```

Cada `target` de ruta admite exactamente una de las siguientes opciones:

- `{ "mode": "current" }`
- `{ "agentId": "main" }`
- `{ "sessionKey": "agent:main:main" }`

Límites: un máximo de 32 rutas y un texto de activación de 64 caracteres como máximo. Los activadores de ruta se normalizan para detectar coincidencias y duplicados convirtiéndolos a minúsculas, eliminando la puntuación inicial y final de cada palabra y contrayendo los espacios en blanco (`"Hey, Bot!!"` y `"hey bot"` coinciden y se consideran duplicados); esta normalización es más estricta que la simple eliminación de espacios iniciales y finales utilizada en la lista global de activadores anterior.

### Eventos

| Evento                      | Carga útil                           |
| --------------------------- | ------------------------------------ |
| `voicewake.changed`         | `{ triggers: string[] }`             |
| `voicewake.routing.changed` | `{ config: VoiceWakeRoutingConfig }` |

Ambos se difunden a todos los clientes WebSocket con permiso de lectura (la aplicación de macOS, WebChat y similares) y a todos los Node conectados. Un Node también recibe ambos como envío de instantánea inicial justo después de conectarse.

## Comportamiento de los clientes

- **macOS**: llama a `voicewake.set`/`voicewake.get` y escucha `voicewake.changed` para mantenerse sincronizado con otros clientes.
- **iOS**: llama a `voicewake.set`/`voicewake.get` y escucha `voicewake.changed` para mantener receptiva la detección local de palabras de activación.
- **Android**: llama a `voicewake.set`/`voicewake.get`, escucha `voicewake.changed` y anuncia `voiceWake` mientras está activado. El reconocimiento permanece en el dispositivo y solo funciona en primer plano; se pausa mientras Talk, el dictado manual, la captura de notas de voz o la reproducción de mensajes por voz controlan el audio.

## Contenido relacionado

- [Modo Talk](/es/nodes/talk)
- [Audio y notas de voz](/es/nodes/audio)
- [Comprensión multimedia](/es/nodes/media-understanding)
