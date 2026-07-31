---
read_when:
    - Configuración de actualizaciones de progreso visibles para turnos de chat de larga duración
    - Elección entre los modos de streaming parcial, por bloques y de progreso
    - Explicación de cómo OpenClaw actualiza un mensaje de canal mientras el trabajo está en curso
    - Borradores de progreso de resolución de problemas, mensajes de progreso independientes o alternativa de finalización
summary: 'Borradores de progreso: un mensaje visible de trabajo en curso que se actualiza mientras se ejecuta un agente'
title: Borradores de progreso
x-i18n:
    generated_at: "2026-07-26T05:06:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ef66dd4d7a31c753f5faa0b88b83ec3760beecf3118cf8aae84f5e57652e809
    source_path: concepts/progress-drafts.md
    workflow: 16
---

Los borradores de progreso convierten un mensaje del canal en una línea de estado en tiempo real mientras un
agente trabaja, en lugar de crear una pila de respuestas temporales de «todavía trabajando». Configure
`channels.<channel>.streaming.mode: "progress"` y OpenClaw creará el
mensaje cuando comience el trabajo real, lo editará mientras el agente lee, planifica, llama a
herramientas o espera aprobación y, después, lo convertirá en la respuesta final.

```text
Trabajando...
📖 de docs/concepts/progress-drafts.md
🔎 Búsqueda web: de "discord edit message"
🛠️ Bash: ejecutar pruebas
```

<Note>
  Discord ya usa de forma predeterminada `streaming.mode: "progress"` cuando
  `channels.discord.streaming` no está configurado, por lo que los borradores de progreso
  aparecen allí sin ninguna configuración. Los demás canales usan de forma predeterminada `partial`
  o `off`; consulte [Transmisión y fragmentación](/es/concepts/streaming#channel-mapping)
  para ver la tabla completa de valores predeterminados de cada canal.
</Note>

## Inicio rápido

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
      },
    },
  },
}
```

Valores predeterminados a partir de aquí: un retraso inicial de 5 segundos, líneas de progreso compactas mientras
se realiza trabajo útil y la supresión de los antiguos mensajes de progreso independientes
para ese turno. Los borradores de líneas de herramientas sin procesar usan
una etiqueta automática de una palabra; un encabezado de estado omite ese título redundante
a menos que se configure uno explícitamente.

Esta página abarca la experiencia de los borradores de progreso y sus opciones de configuración. Para consultar
la matriz completa de modos de transmisión, las notas de ejecución de cada canal y la migración
de claves heredadas, consulte [Transmisión y fragmentación](/es/concepts/streaming).

## Qué ven los usuarios

| Parte                | Propósito                                                                                                               |
| -------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Encabezado de estado | En Discord y Telegram, el preámbulo del modelo; Discord añade un texto auxiliar.                                        |
| Etiqueta             | Línea inicial o de estado opcional, como `Working`.                                                            |
| Líneas de progreso   | Actualizaciones compactas de ejecución que usan los mismos iconos de herramientas y el mismo formateador que `/verbose`. |

Para el progreso de herramientas sin procesar, la etiqueta aparece cuando el agente comienza un trabajo significativo
y permanece ocupado durante el retraso inicial.
Se sitúa en la parte superior de la lista continua de líneas de progreso, por lo que desaparece al desplazarse cuando
aparecen suficientes líneas de trabajo concretas. Un encabezado de estado solo muestra el estado
del agente en lenguaje natural, salvo que se configure explícitamente una etiqueta. Las respuestas
que solo contienen texto nunca muestran un borrador de progreso; una línea aparece únicamente para actualizaciones de trabajo real,
por ejemplo `🛠️ Bash: run tests`, `🔎 Web Search: for "discord edit message"`
o `✍️ Write: to /tmp/file`.

La respuesta final sustituye al borrador en el mismo lugar cuando el canal puede hacerlo de forma segura;
de lo contrario, OpenClaw envía la respuesta final mediante la entrega normal y
elimina el borrador o deja de actualizarlo (consulte [Finalización](#finalization)).

## Elegir un modo

`channels.<channel>.streaming.mode` controla el comportamiento visible durante el progreso:

| Modo                   | Adecuado para                             | Qué aparece en el chat                                           |
| ---------------------- | ----------------------------------------- | ----------------------------------------------------------------- |
| `off`     | Canales silenciosos                       | Solo la respuesta final.                                         |
| `partial`     | Ver cómo aparece el texto de la respuesta | Un borrador editado con el texto más reciente de la respuesta.    |
| `block`     | Fragmentos mayores de vista previa        | Una vista previa actualizada o ampliada en fragmentos más grandes. |
| `progress`     | Turnos largos o con muchas herramientas   | Un borrador de estado y, después, la respuesta final.             |

Elija `progress` cuando a los usuarios les importe más «qué está sucediendo» que ver
el texto de la respuesta transmitirse token por token; `partial` cuando el propio texto de la respuesta sea
la señal de progreso; `block` para fragmentos mayores de vista previa. En Discord y
Telegram, `streaming.mode: "block"` sigue siendo transmisión de vista previa, no entrega normal
de respuestas en bloques; use `streaming.block.enabled` para ello.

## Configurar etiquetas

Las etiquetas de progreso se encuentran en `channels.<channel>.streaming.progress`. La etiqueta predeterminada
para líneas de herramientas sin procesar es `"auto"`, que usa la etiqueta sencilla integrada `Working`.
Un encabezado de estado oculta esa etiqueta implícita; configure
`label: "auto"` explícitamente si también desea una etiqueta encima:

```text
Trabajando
```

Use una etiqueta fija:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "Investigating",
        },
      },
    },
  },
}
```

Use su propio conjunto de etiquetas (se siguen eligiendo al azar o por semilla cuando `label: "auto"`):

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto",
          labels: ["Checking", "Reading", "Testing", "Finishing"],
        },
      },
    },
  },
}
```

Oculte la etiqueta y muestre únicamente las líneas de progreso:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: false,
        },
      },
    },
  },
}
```

## Controlar las líneas de progreso

Las líneas de progreso proceden de eventos reales de ejecución: inicio de herramientas, actualizaciones de elementos, planes
de tareas, aprobaciones, salida de comandos, resúmenes de parches y actividades similares del agente.
Están habilitadas de forma predeterminada (`progress.toolProgress`, valor predeterminado `true`).

Las herramientas también pueden emitir progreso tipado mientras una sola llamada sigue ejecutándose. Así,
una obtención o búsqueda lenta actualiza el borrador visible antes de que la herramienta
devuelva su resultado final. La actualización de progreso es un resultado parcial de la herramienta con
contenido vacío para el modelo y metadatos públicos explícitos del canal:

```json
{
  "content": [],
  "progress": {
    "text": "Fetching page content...",
    "visibility": "channel",
    "privacy": "public",
    "id": "web_fetch:fetching"
  }
}
```

OpenClaw representa únicamente `progress.text` en la interfaz de progreso del canal. El resultado normal
de la herramienta llega después como `content`/`details` y es la única parte
que se devuelve al modelo.

Al añadir progreso a una herramienta, emita un mensaje breve y genérico y retráselo
hasta que la operación lleve pendiente el tiempo suficiente para resultar útil. `web_fetch`
hace exactamente esto con un retraso de 5 segundos:

```typescript
const clearProgressTimer = scheduleToolProgress(
  onUpdate,
  { text: "Fetching page content...", id: "web_fetch:fetching" },
  5_000,
  { signal },
);

try {
  return await runToolWork();
} finally {
  clearProgressTimer();
}
```

Las llamadas rápidas no muestran ninguna línea de progreso; las llamadas largas muestran una mientras siguen pendientes;
las llamadas canceladas borran el temporizador antes de que pueda aparecer progreso obsoleto. El texto de progreso
es un canal lateral público de la interfaz, por lo que nunca debe incluir secretos, argumentos sin procesar,
contenido obtenido, salida de comandos ni texto de páginas.

### Modo de detalle

OpenClaw usa el mismo formateador para los borradores de progreso y `/verbose`:

```json5
{
  agents: {
    defaults: {
      toolProgressDetail: "explain", // explain | raw
    },
  },
}
```

`"explain"` es el valor predeterminado y mantiene estables los borradores con etiquetas concisas.
`"raw"` añade el comando subyacente cuando está disponible, lo que resulta útil durante
la depuración, pero genera más ruido en el chat. Por ejemplo, una llamada a `node --check /tmp/app.js`
se representa de forma diferente según el modo:

| Modo                   | Línea de progreso                |
| ---------------------- | -------------------------------- |
| `explain`     | `🛠️ check js syntax for /tmp/app.js`               |
| `raw`     | `🛠️ check js syntax for /tmp/app.js · node --check /tmp/app.js`               |

### Texto de comandos/exec

`streaming.progress.commandText` (valor predeterminado `"raw"`) controla cuánto detalle del comando
se muestra junto a las líneas de progreso de exec/bash, con independencia del modo de detalle
anterior. Establézcalo en `"status"` para mantener visible una línea de progreso de la herramienta y ocultar
por completo el texto del comando:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          commandText: "status",
        },
      },
    },
  },
}
```

### Canal de comentarios

`streaming.progress.commentary` (valor predeterminado `false`) intercala la narración
de comentarios o preámbulo previa a las herramientas del modelo (💬, por ejemplo, «Comprobaré... y después
...») con las líneas de herramientas del borrador. Consulte
[Transmisión y fragmentación](/es/concepts/streaming#commentary-progress-lane) para ver la
estructura de configuración compartida entre canales.

Con el canal de comentarios habilitado, los preámbulos se representan únicamente como esas líneas 💬
intercaladas; el encabezado de estado siguiente no aparece, de modo que el canal conserva su
estructura documentada.

### Encabezado de estado

En Discord y Telegram, en el modo de progreso, el preámbulo tipado previo a las herramientas
del modelo se convierte en el encabezado de estado del borrador siempre que esté disponible. Los demás
canales en modo de progreso mantienen su comportamiento de estado existente. El encabezado está
habilitado de forma predeterminada y no elude el filtro normal de actividad para turnos cortos;
al habilitar `streaming.progress.commentary`, los preámbulos se envían al canal
de comentarios intercalados.

En Discord, cuando se resuelve un modelo auxiliar para el agente —un
[`utilityModel`](/es/gateway/config-agents#utilitymodel) explícito o el modelo pequeño
predeterminado declarado por el proveedor principal (OpenAI → `gpt-5.6-luna`,
Anthropic → `claude-haiku-4-5`)—, proporciona un breve texto auxiliar en lenguaje natural
cuando el modelo no emite ningún preámbulo o lleva unos 20 segundos en silencio
(el encabezado de Telegram actualmente solo usa preámbulos):

```text
Actualizando el modelo predeterminado en la configuración y reiniciando después el Gateway para
aplicarlo. Una llamada para enumerar agentes falló y se está reintentando.
```

La narración auxiliar está habilitada de forma predeterminada (`streaming.progress.narration`, valor predeterminado
`true`) y nunca recurre al modelo principal: solo se ejecuta con un
`utilityModel` explícito o con un valor predeterminado declarado por el proveedor principal
del agente. Configure `utilityModel: ""` para deshabilitar por completo el enrutamiento auxiliar. Las líneas de herramientas
siguen acumulándose debajo y vuelven a aparecer si ambas fuentes de estado se detienen. Las ediciones
del borrador siguen esperando al filtro normal de actividad y a un cambio real
del texto, lo que evita parpadeos en turnos rápidos y reduce las ediciones reiteradas en canales
con mucha actividad. Configure `narration: false` para deshabilitar únicamente el texto auxiliar
del modelo auxiliar; los encabezados de preámbulo del modelo permanecen habilitados:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          narration: false,
        },
      },
    },
  },
}
```

La entrada de la narración está limitada y censurada: el modelo auxiliar recibe el texto
de la solicitud entrante junto con los mismos resúmenes compactos y censurados de herramientas que representaría
el borrador, nunca la salida de comandos ni los resultados de herramientas sin procesar. Con
`commandText: "status"`, la entrada de la narración también omite el texto de comandos exec/bash,
de acuerdo con lo que muestra el borrador.

### Límites de líneas

Limite cuántas líneas permanecen visibles (valor predeterminado: 8):

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 4,
        },
      },
    },
  },
}
```

Las líneas de progreso se compactan automáticamente para reducir el reajuste de las burbujas del chat mientras
se edita el borrador, y OpenClaw trunca las líneas largas para que las ediciones repetidas del borrador
no se ajusten de forma diferente con cada actualización. El límite predeterminado por línea es de 120
caracteres; la prosa se corta en un límite de palabra, mientras que los detalles largos, como rutas o
comandos sin procesar, se acortan con puntos suspensivos en el centro para mantener visible el sufijo.

Ajuste el límite por línea:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLineChars: 160,
        },
      },
    },
  },
}
```

### Representación enriquecida (Slack)

Slack puede representar las líneas de progreso como campos estructurados de Block Kit en lugar de
texto sin formato:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          render: "rich",
        },
      },
    },
  },
}
```

La representación enriquecida siempre envía el mismo cuerpo de texto sin formato junto con los campos
de Block Kit, por lo que los clientes que no pueden representar la estructura enriquecida siguen mostrando
el texto de progreso compacto.

### Ocultar líneas de herramientas y tareas

Mantenga el único borrador de progreso, pero oculte las líneas de herramientas y tareas:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          toolProgress: false,
        },
      },
    },
  },
}
```

Con `toolProgress: false`, OpenClaw sigue suprimiendo los mensajes independientes
anteriores de progreso de herramientas para ese turno; el canal permanece visualmente
silencioso hasta la respuesta final, excepto por la etiqueta si hay una configurada.

## Comportamiento del canal

| Canal           | Transporte del progreso                         | Notas                                                                                                                                                                                                    |
| --------------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | Envía un mensaje y luego lo edita.              | El modo predeterminado es `progress`; la respuesta final incluye un recibo de actividad `-#` y el borrador de estado se elimina después de que se entrega la respuesta.              |
| Matrix          | Envía un evento y luego lo edita.               | La configuración de transmisión a nivel de cuenta controla los borradores a nivel de cuenta.                                                                                                             |
| Microsoft Teams | Transmisión nativa de Teams en chats personales. | `streaming.mode: "block"` se asigna en su lugar a la entrega por bloques de Teams.                                                                                                                               |
| Slack           | Transmisión nativa o publicación de borrador editable. | Requiere un hilo de respuesta de destino; los mensajes directos de nivel superior sin uno siguen recibiendo publicaciones de vista previa del borrador y sus ediciones.                                  |
| Telegram        | Envía un mensaje y luego lo edita.              | Si llega un mensaje entre el borrador de progreso y la respuesta, el borrador se vuelve a publicar debajo (primero se publica el nuevo y luego se elimina el anterior) en vez de desplazar bruscamente el cliente. |
| Mattermost      | Publicación de borrador editable.               | El modo `block` alterna entre publicaciones de texto completado y de actividad de herramientas; los demás modos integran la actividad de las herramientas en la misma publicación tipo borrador. |

Los canales sin compatibilidad segura con la edición recurren a indicadores
de escritura o a la entrega únicamente de la respuesta final. Consulte
[Transmisión y fragmentación](/es/concepts/streaming) para obtener el desglose
completo del comportamiento en tiempo de ejecución por canal.

## Finalización

Cuando la respuesta final está lista, OpenClaw intenta mantener limpio el chat:

- En el modo `progress` de Discord, la respuesta final se envía como un mensaje nuevo
  con un pequeño recibo de actividad `-#` adjunto (por ejemplo,
  `-# 🧠 2 thoughts · 🛠️ 5 tool calls · ⏱️ 12s`), y el borrador de estado se
  elimina una vez entregada la respuesta. Los canales con mucha actividad no mantienen ningún registro
  huérfano de herramientas encima de la respuesta; las respuestas finales con errores conservan el borrador como registro visible del
  turno fallido.
- Si el borrador puede convertirse de forma segura en la respuesta final (modos `partial`/`block`),
  OpenClaw lo edita en el mismo lugar.
- Si el canal utiliza transmisión nativa del progreso, OpenClaw finaliza esa
  transmisión cuando el transporte nativo acepta el texto final.
- En caso contrario (contenido multimedia, una solicitud de aprobación, un destino de respuesta explícito, demasiados
  fragmentos o un error de edición o envío), OpenClaw envía la respuesta final mediante la
  ruta normal de entrega del canal en lugar de sobrescribir el borrador.

Este mecanismo alternativo es intencional: enviar una respuesta final nueva es
preferible a perder texto, asociar una respuesta al hilo equivocado o sobrescribir
un borrador con una carga que el canal no puede representar de forma segura.

## Solución de problemas

**Solo veo la respuesta final.**

Compruebe que `channels.<channel>.streaming.mode` sea `progress` en la cuenta
o el canal que gestionó el mensaje. Algunas rutas de grupos o respuestas con
cita desactivan las vistas previas del borrador durante un turno cuando el canal
no puede editar de forma segura el mensaje correcto.

**Veo la etiqueta, pero ninguna línea de herramientas.**

Compruebe `streaming.progress.toolProgress`. Si es `false`, OpenClaw mantiene
el comportamiento de borrador único, pero oculta las líneas de progreso de
herramientas y tareas.

**Veo un mensaje final nuevo en lugar de un borrador editado.**

Ese es el mecanismo alternativo de seguridad descrito en
[Finalización](#finalization). Puede ocurrir con respuestas multimedia,
respuestas largas, destinos de respuesta explícitos, borradores antiguos de
Telegram, destinos de hilo ausentes en Slack, mensajes de vista previa
eliminados o errores al finalizar una transmisión nativa.

**Sigo viendo mensajes independientes de progreso.**

El modo de progreso suprime los mensajes independientes predeterminados de
progreso de herramientas cuando hay un borrador activo. Si siguen apareciendo
mensajes independientes, confirme que el turno esté utilizando realmente el
modo `progress` y no `streaming.mode: "off"`, ni una ruta del canal que no
pueda crear un borrador para ese mensaje.

**Teams se comporta de forma diferente a Discord o Telegram.**

Microsoft Teams utiliza una transmisión nativa en chats personales en lugar del
transporte genérico de vista previa basado en enviar y editar, y asigna
`streaming.mode: "block"` a la entrega por bloques de Teams porque no dispone de un
modo de bloques de vista previa del borrador como Discord y Telegram.

## Temas relacionados

- [Transmisión y fragmentación](/es/concepts/streaming)
- [Mensajes](/es/concepts/messages)
- [Configuración de canales](/es/gateway/config-channels)
- [Discord](/es/channels/discord)
- [Matrix](/es/channels/matrix)
- [Microsoft Teams](/es/channels/msteams)
- [Slack](/es/channels/slack)
- [Telegram](/es/channels/telegram)
- [Mattermost](/es/channels/mattermost)
