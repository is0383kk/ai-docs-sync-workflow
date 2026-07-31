---
read_when:
    - Quieres reducir el crecimiento del contexto causado por las salidas de las herramientas
    - Quieres comprender la optimización de la caché de prompts de Anthropic
summary: Recorte de resultados antiguos de herramientas para mantener un contexto conciso y una caché eficiente
title: Poda de sesiones
x-i18n:
    generated_at: "2026-07-26T04:35:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd5cb4582cb8d9d7265213abe1f5b5893634882b9f8b3ce1deef746293dd07db
    source_path: concepts/session-pruning.md
    workflow: 16
---

La poda de sesiones recorta los **resultados antiguos de herramientas** del contexto antes de cada llamada al LLM. Reduce la sobrecarga del contexto causada por los resultados acumulados de herramientas (resultados de ejecución, lecturas de archivos, resultados de búsqueda) sin reescribir el texto normal de la conversación.

<Info>
La poda solo se realiza en memoria; no modifica la transcripción de la sesión almacenada en disco. El historial completo siempre se conserva.
</Info>

## Por qué es importante

Las sesiones largas acumulan resultados de herramientas que inflan la ventana de contexto. Esto aumenta el coste y puede forzar la [Compaction](/es/concepts/compaction) antes de lo necesario.

La poda es especialmente valiosa para el **almacenamiento en caché de prompts de Anthropic**. Una vez que vence el TTL de la caché, la siguiente solicitud vuelve a almacenar en caché el prompt completo. La poda reduce el tamaño de escritura en la caché, lo que disminuye directamente el coste.

## Cómo funciona

La poda se ejecuta en el modo `cache-ttl`, condicionada tanto por una comprobación de tiempo como por una comprobación del tamaño del contexto:

1. Se espera a que venza el TTL de la caché (de forma predeterminada, 5 minutos cuando se establece manualmente; consulte [Valores predeterminados inteligentes](#smart-defaults) para conocer el valor predeterminado automático de Anthropic). Antes de que transcurra el TTL, la poda se omite por completo para conservar la reutilización de la caché de prompts en turnos cercanos.
2. Una vez transcurrido el TTL, se calcula el tamaño total del contexto en relación con la ventana de contexto del modelo. Si la proporción es inferior a `softTrimRatio` (valor predeterminado: 0.3), se omite la poda y el reloj del TTL sigue avanzando.
3. Se aplica un **recorte parcial** a los resultados de herramientas sobredimensionados que superen la proporción: se conservan el principio y el final (de forma predeterminada, 1500 caracteres cada uno, con un límite combinado de 4000 caracteres) y se inserta `...` entre ambos.
4. Si la proporción sigue siendo igual o superior a `hardClearRatio` (valor predeterminado: 0.5) y quedan al menos `minPrunableToolChars` (valor predeterminado: 50,000) de contenido de herramientas que se pueda podar, esos resultados se **eliminan por completo**: su contenido se sustituye por un marcador de posición (valor predeterminado: `[Old tool result content cleared]`).
5. El reloj del TTL solo se reinicia cuando la poda modifica realmente el contexto, para que las solicitudes posteriores reutilicen la caché recién creada.

Se aplican dos reglas de seguridad independientemente de los umbrales: nunca se podan los `keepLastAssistants` turnos más recientes del asistente (valor predeterminado: 3), ni nada anterior al primer mensaje del usuario de la sesión (esto protege lecturas de inicialización como `SOUL.md`/`USER.md`).

Solo los mensajes `toolResult` son aptos; el texto normal de la conversación no se modifica. Utilice `agents.defaults.contextPruning.tools.{allow,deny}` para delimitar qué nombres de herramientas se pueden podar.

## Limpieza de imágenes heredadas

OpenClaw también crea una vista de reproducción idempotente independiente para las sesiones que conservan bloques de imágenes sin procesar o marcadores multimedia de hidratación de prompts en el historial.

- Conserva byte por byte los **3 turnos completados más recientes** para que los prefijos de la caché de prompts permanezcan estables en las interacciones posteriores recientes. Este recuento incluye todos los turnos completados, no solo los que contienen imágenes, por lo que los turnos que solo contienen texto también ocupan la ventana.
- En la vista de reproducción, los bloques de imágenes más antiguos ya procesados del historial de `user` o `toolResult` se sustituyen por `[image data removed - already processed by model]`.
- Las referencias multimedia textuales más antiguas, como `[media attached: ...]`, `[Image: source: ...]` y `media://inbound/...`, se sustituyen por `[media reference removed - already processed by model]`. Los marcadores de archivos adjuntos del turno actual permanecen intactos para que los modelos de visión aún puedan hidratar imágenes nuevas.
- La transcripción sin procesar de la sesión no se reescribe, por lo que los visores del historial todavía pueden representar las entradas de mensajes originales y sus imágenes.
- Esto es independiente de la poda normal por TTL de caché descrita anteriormente. Su finalidad es evitar que las cargas repetidas de imágenes o las referencias multimedia obsoletas invaliden las cachés de prompts en turnos posteriores.

## Valores predeterminados inteligentes

El plugin de Anthropic incluido configura automáticamente la poda y la cadencia del Heartbeat la primera vez que resuelve un perfil de autenticación de Anthropic (o de la CLI de Claude), pero solo para los campos que aún no se hayan establecido explícitamente:

| Modo de autenticación                    | `contextPruning.mode` | `contextPruning.ttl` | `heartbeat.every` |
| ---------------------------------------- | --------------------- | -------------------- | ----------------- |
| OAuth/token (incluida la reutilización de la CLI de Claude) | `cache-ttl`           | `1h`                 | `1h`              |
| Clave de API                             | `cache-ttl`           | `1h`                 | `30m`             |

Si se establece manualmente `agents.defaults.contextPruning.mode` o `agents.defaults.heartbeat.every`, OpenClaw no los sobrescribe. Este valor predeterminado automático solo se aplica a la autenticación de la familia Anthropic; los demás proveedores reciben la poda `off` salvo que se configure.

## Activar o desactivar

La poda está desactivada de forma predeterminada para los proveedores que no sean Anthropic. Para activarla:

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

Para desactivarla: establezca `mode: "off"`.

## Poda frente a Compaction

|                | Poda                           | Compaction                   |
| -------------- | ------------------------------ | ---------------------------- |
| **Qué hace**   | Recorta resultados de herramientas | Resume la conversación   |
| **¿Se guarda?** | No (por solicitud)            | Sí (en la transcripción)     |
| **Ámbito**     | Solo resultados de herramientas | Toda la conversación       |

Ambas se complementan: la poda mantiene concisos los resultados de las herramientas entre ciclos de Compaction.

## Lecturas adicionales

- [Compaction](/es/concepts/compaction): reducción del contexto basada en resúmenes
- [Configuración del Gateway](/es/gateway/configuration): todas las opciones de configuración de la poda (`contextPruning.*`)

## Temas relacionados

- [Gestión de sesiones](/es/concepts/session)
- [Herramientas de sesión](/es/concepts/session-tool)
- [Motor de contexto](/es/concepts/context-engine)
