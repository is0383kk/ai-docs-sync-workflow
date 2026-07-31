---
read_when:
    - Adición o modificación de la representación de tarjetas de mensaje, gráficos, tablas, botones o selectores
    - Creación de un plugin de canal compatible con mensajes salientes enriquecidos
    - Cambio de la presentación de la herramienta de mensajes o de las capacidades de entrega
    - Depuración de regresiones de renderizado de tarjetas, bloques y componentes específicas del proveedor
summary: Tarjetas de mensajes semánticas, gráficos, tablas, controles, texto alternativo e indicaciones de entrega para plugins de canal
title: Presentación de mensajes
x-i18n:
    generated_at: "2026-07-26T05:20:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

La presentación de mensajes es el contrato compartido de OpenClaw para interfaces de chat salientes enriquecidas.
Permite que los agentes, los comandos de la CLI, los flujos de aprobación y los plugins describan una sola vez
la intención del mensaje, mientras cada plugin de canal representa la mejor forma nativa que puede.

Utilice la presentación para interfaces de mensajes portátiles: secciones de texto, texto contextual o de pie
breve, divisores, gráficos, tablas, botones, menús de selección y título o tono de tarjetas.

No añada al instrumento de mensajes compartido nuevos campos nativos del proveedor, como Discord `components`, Slack
`blocks`, Telegram `buttons`, Teams `card` o Feishu `card`. Son salidas del renderizador
que pertenecen al plugin de canal.

## Contrato

Los autores de plugins importan el contrato público desde:

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Estructura:

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** Valor de devolución de llamada heredado. Para controles nuevos, prefiera action. */
  value?: string;
  /** @deprecated Utilice una acción con el tipo "url". */
  url?: string;
  /** @deprecated Utilice una acción con el tipo "web-app". */
  webApp?: { url: string };
  /** @deprecated Utilice una acción con el tipo "web-app". */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** Valor de devolución de llamada heredado. Para controles nuevos, prefiera action. */
  value?: string;
};

type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

Semántica de los botones:

- `action.type: "command"` ejecuta un comando de barra nativo mediante la ruta de comandos
  del núcleo. Utilícelo para botones y menús de comandos integrados.
- `action.type: "callback"` transporta datos opacos del plugin mediante la ruta de
  interacción del canal. Los plugins de canal no deben reinterpretar los datos de devolución de llamada como
  comandos de barra.
- `action.type: "approval"` identifica una aprobación duradera del operador, su tipo
  explícito `exec` o `plugin` y la decisión solicitada. Los plugins de canal
  codifican esa acción en una devolución de llamada privada del transporte y la resuelven mediante
  el servicio de aprobación; no deben analizar el texto del comando `/approve` ni deducir
  el tipo a partir del ID.
- `action.type: "question"` identifica una opción para una pregunta `ask_user` activa
  creada por el entorno de ejecución. Al igual que `approval`, esta es una acción del entorno de ejecución de OpenClaw;
  los agentes y plugins no deben generar identificadores de preguntas. Telegram, Discord y
  Slack la asignan a devoluciones de llamada nativas privadas del transporte y resuelven la opción
  mediante el Gateway. Cuando la pregunta se responde, caduca o
  se cancela, esos canales editan el mensaje entregado, eliminan sus acciones
  y añaden el estado final. WhatsApp, Signal e iMessage representan hasta
  cuatro opciones de selección única como reacciones de `1️⃣` a `4️⃣`. Las demás formas de pregunta
  se degradan a texto de etiqueta y se puede responder con una
  respuesta de texto sin formato.
- `action.type: "url"` abre un enlace normal.
- `action.type: "web-app"` inicia una aplicación web nativa del canal. Establezca `url` para una
  aplicación respaldada por una URL o `widgetId` para un widget alojado por OpenClaw cuyo mecanismo
  de inicio pertenece al canal; se requiere al menos uno. Cuando ambos están
  presentes, un canal puede preferir el inicio de su widget alojado nativo y utilizar la URL
  cuando ese mecanismo no esté disponible.
- `value` es el valor opaco heredado de devolución de llamada. Los controles nuevos deben utilizar `action`
  para que los plugins de canal puedan asignar comandos y devoluciones de llamada sin deducirlos del texto.
- `url`, `webApp` y `web_app` siguen aceptándose como entradas de límite obsoletas.
  Los normalizadores conservan estos campos para que los renderizadores puedan distinguir la semántica heredada
  publicada de las acciones tipadas explícitas. Los productores nuevos deben utilizar `action`.
- `label` es obligatorio y también se utiliza en la alternativa de texto.
- `style` es orientativo. Los renderizadores deben asignar los estilos no compatibles a un valor
  predeterminado seguro, no hacer que falle el envío.
- `priority` es opcional. Cuando un canal anuncia límites de acciones y es necesario
  descartar controles, el núcleo conserva primero los botones con mayor prioridad y mantiene
  el orden original entre los botones de igual prioridad. Cuando caben todos los controles, se conserva
  el orden definido por el autor.
- `disabled` es opcional. Los canales deben habilitarlo expresamente con `supportsDisabled`; de lo contrario,
  el núcleo degrada el control deshabilitado a texto alternativo no interactivo. Un
  botón deshabilitado siempre se representa únicamente mediante su etiqueta en el texto alternativo, incluso cuando
  contiene una acción `command`.
- `reusable` es opcional. Los canales que admiten devoluciones de llamada nativas reutilizables pueden
  mantener la acción disponible después de una interacción correcta. Utilícelo para
  acciones repetibles o idempotentes, como actualizar, inspeccionar o mostrar más detalles;
  déjelo sin establecer para las aprobaciones normales de un solo uso y las acciones destructivas.

Semántica de la selección:

- `options[].action` solo acepta `command` o `callback`; las acciones de aprobación y enlace son exclusivas de los botones.
- `options[].value` es el valor de aplicación seleccionado heredado.
- `placeholder` es orientativo y los canales sin compatibilidad nativa con selecciones pueden
  ignorarlo.
- Si un canal no admite selecciones, el texto alternativo enumera las etiquetas.

Semántica de los gráficos:

- `pie` requiere valores de segmento positivos.
- `bar`, `area` y `line` utilizan una matriz `categories` ordenada. Cada serie
  proporciona exactamente un valor finito por categoría, en el mismo orden.
- Las etiquetas de las categorías y los nombres de las series deben ser únicos. Los bloques de gráficos
  no válidos o incompletos se descartan durante la normalización en lugar de modificar silenciosamente los datos.
- La representación nativa de gráficos se habilita expresamente mediante `presentationCapabilities.charts`.
  Los demás canales reciben el título del gráfico, los ejes, las categorías, las series y los valores
  como texto determinista. Esta es también la alternativa de accesibilidad.

Semántica de las tablas:

- `caption` es un encabezado breve obligatorio. `headers` debe contener al menos una
  etiqueta de columna única y no vacía.
- `rows` debe contener al menos una fila. Cada fila debe tener exactamente una celda por
  encabezado y cada celda debe ser una cadena no vacía o un número finito.
- `rowHeaderColumnIndex` es un índice opcional de base cero que identifica la columna
  cuyas celdas deben exponerse como encabezados de fila mediante los renderizadores nativos.
- La normalización de tablas es atómica. Un título, encabezado, ancho de fila, celda
  o índice de encabezado de fila no válido provoca que se descarte el bloque de tabla, en lugar de truncar o reparar
  sus datos.
- La representación nativa de tablas se habilita expresamente mediante `presentationCapabilities.tables`.
  Los demás canales reciben el título y cada fila como texto lineal
  determinista, con los espacios internos contraídos:

  ```text
  Pipeline abierto (tabla)
  - Cuenta: Acme; Etapa: Ganada; ARR: 125000
  - Cuenta: Globex; Etapa: Revisión; ARR: 82000
  ```

No existe un discriminador `report` independiente. Componga un informe a partir de `title`,
`tone`, `text`, `context`, `chart`, `table` y bloques de acciones. Esto permite que cada
bloque se pueda representar de forma independiente y proporciona al informe completo la misma
alternativa de texto determinista.

## Ejemplos de productores

Tarjeta sencilla:

```json
{
  "title": "Aprobación del despliegue",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "La versión canary está lista para promoverse." },
    { "type": "context", "text": "Compilación 1234; la fase de staging se completó correctamente." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Aprobar",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "Rechazar",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

Botón de enlace únicamente con URL:

```json
{
  "blocks": [
    { "type": "text", "text": "Las notas de la versión están listas." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Abrir notas",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Botón de Mini App de Telegram:

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Iniciar",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

Menú de selección:

```json
{
  "title": "Elegir entorno",
  "blocks": [
    {
      "type": "select",
      "placeholder": "Entorno",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "Producción", "value": "env:prod" }
      ]
    }
  ]
}
```

Gráfico:

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "Ingresos trimestrales",
      "categories": ["T1", "T2", "T3"],
      "series": [
        { "name": "Producto", "values": [120, 145, 138] },
        { "name": "Servicios", "values": [80, 95, 104] }
      ],
      "xLabel": "Trimestre",
      "yLabel": "Ingresos"
    }
  ]
}
```

Informe tabular:

```json
{
  "title": "Informe del pipeline",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "Oportunidades actuales por etapa." },
    {
      "type": "table",
      "caption": "Pipeline abierto",
      "headers": ["Cuenta", "Etapa", "ARR"],
      "rows": [
        ["Acme", "Ganada", 125000],
        ["Globex", "Revisión", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "Actualizado a partir de la instantánea del CRM." }
  ]
}
```

Envío mediante la CLI:

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "Aprobación del despliegue" \
  --presentation '{"title":"Aprobación del despliegue","tone":"warning","blocks":[{"type":"text","text":"La versión canary está lista."},{"type":"buttons","buttons":[{"label":"Aprobar","value":"deploy:approve","style":"success"},{"label":"Rechazar","value":"deploy:decline","style":"danger"}]}]}'
```

Entrega fijada:

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "Tema abierto" \
  --pin
```

Entrega fijada con JSON explícito:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## Contrato del renderizador

Los plugins de canal declaran la compatibilidad de renderizado en su adaptador de salida:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

Los valores booleanos de capacidad describen qué elementos puede hacer interactivos el renderizador. Los valores opcionales
`limits` describen el contenedor genérico que el núcleo puede adaptar antes de llamar al
renderizador:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};
```

El núcleo aplica límites genéricos a los controles semánticos antes del renderizado. Los renderizadores
siguen siendo responsables de la validación final específica del proveedor y del recorte del número de bloques nativos,
el tamaño de las tarjetas, los límites de URL y las particularidades del proveedor que no se pueden expresar en
el contrato genérico. Si los límites eliminan todos los controles de un bloque, el núcleo conserva
las etiquetas como texto de contexto no interactivo para que el mensaje entregado siga teniendo una
alternativa visible.

## Flujo de renderizado del núcleo

En la ruta de salida canónica utilizada por la CLI y las acciones de mensaje estándar, el núcleo:

1. Normaliza la carga útil de presentación.
2. Resuelve el adaptador de salida del canal de destino.
3. Lee `presentationCapabilities`.
4. Aplica límites genéricos de capacidad, como el número de acciones, la longitud de las etiquetas y
   el número de opciones de selección, cuando el adaptador los anuncia. Los bloques de gráficos y tablas
   se convierten en texto determinista, salvo que el adaptador anuncie explícitamente
   `charts: true` o `tables: true`, respectivamente.
5. Llama a `renderPresentation` cuando el adaptador puede renderizar la carga útil.
6. Recurre a texto conservador cuando el adaptador no existe o no puede renderizar.
7. Envía la carga útil resultante mediante la ruta normal de entrega del canal.
8. Aplica metadatos de entrega como `delivery.pin` después del primer mensaje
   enviado correctamente.

Los flujos locales del canal de respuesta o vista previa que consumen `ReplyPayload` directamente
deben entrar en esa ruta canónica o materializar la misma alternativa de presentación
antes de proyectar la carga útil como texto sin formato o contenido multimedia.

El núcleo controla el comportamiento alternativo para que los productores puedan ser independientes del canal. Los
plugins de canal controlan el renderizado nativo y la gestión de interacciones.

## Reglas de degradación

La presentación debe poder enviarse de forma segura en canales limitados.

El texto alternativo incluye:

- `title` como primera línea
- bloques `text` como párrafos normales
- bloques `context` como líneas de contexto compactas
- bloques `divider` como separador visual
- etiquetas de botones, incluidas las URL de los botones de enlace
- etiquetas de opciones de selección
- título, tipo, ejes, categorías, series y valores del gráfico
- leyenda, encabezados y valores de todas las filas de la tabla

### Visibilidad alternativa de los valores de botones

Cuando un canal no puede renderizar controles interactivos, los valores de botones y selecciones
se convierten en texto sin formato. El comportamiento alternativo mantiene la usabilidad y
conserva la privacidad de los datos opacos de devolución de llamada:

- Las **acciones de tipo `command`** se renderizan como `` label: `command` `` para que los usuarios puedan
  copiar el comando y ejecutarlo manualmente en la entrada del canal.
- Las **acciones de tipo `callback`** y los campos **`value`** heredados se renderizan
  solo con la etiqueta. El valor opaco de devolución de llamada no se expone en el texto alternativo.
- Las **acciones de tipo `approval`** se renderizan solo con la etiqueta. Los identificadores y las decisiones de aprobación son
  datos de transporte y no se exponen mediante ayudantes escalares genéricos ni texto
  alternativo.
- Las **acciones `url`**, las **acciones `web-app` respaldadas por URL** y las entradas obsoletas **`url` /
  `webApp` / `web_app`** renderizan el texto de la URL junto a la etiqueta del botón,
  ya que la URL es visible para el usuario. Las acciones exclusivas de widgets alojados se renderizan solo con la etiqueta en
  canales sin inicio nativo de widgets.
- Las **opciones de selección** se renderizan solo con la etiqueta. El valor subyacente de la opción no se
  expone en el texto alternativo.

Los adaptadores de canal que añadan instrucciones para comandos manuales en su interfaz de usuario alternativa (por ejemplo,
las instrucciones de comentarios de documentos de Feishu) deben derivar la comprobación de presencia de comandos
de los mismos bloques de presentación que utiliza el renderizador alternativo, para que el
texto de orientación solo aparezca cuando realmente se muestre un comando manual.

Los controles nativos no compatibles deben degradarse en lugar de provocar el fallo de todo el envío.
Ejemplos:

- Telegram con los botones insertados deshabilitados envía texto alternativo.
- Un canal sin compatibilidad con selecciones muestra las opciones de selección como texto.
- Un canal sin compatibilidad nativa con gráficos muestra los datos del gráfico como texto.
- Un canal sin compatibilidad nativa con tablas muestra todas las filas de la tabla como texto.
- Un botón que solo contiene una URL se convierte en un botón de enlace nativo o en una línea de URL alternativa.
- Los fallos opcionales de fijación no provocan el fallo del mensaje entregado.

La principal excepción es `delivery.pin.required: true`; si la fijación se solicita como
obligatoria y el canal no puede fijar el mensaje enviado, la entrega informa de un fallo.

## Asignación de proveedores

Renderizadores incluidos actualmente:

| Canal           | Destino de renderizado nativo             | Notas                                                                                                                                                                                                             |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | Componentes y contenedores de componentes | Conserva `channelData.discord.components` heredado para los productores de cargas útiles nativas del proveedor existentes, pero los nuevos envíos compartidos deben utilizar `presentation`.                                                                 |
| Feishu          | Tarjetas interactivas                     | El encabezado de la tarjeta puede utilizar `title`; el cuerpo evita duplicar ese título.                                                                                                                                                  |
| Matrix          | Texto alternativo más campo de evento estructurado | Los botones y las selecciones se anuncian como compatibles, pero actualmente todos los bloques se renderizan como salida `renderMessagePresentationFallbackText` incluida en un campo de evento `com.openclaw.presentation`, no como widgets interactivos nativos. |
| Mattermost      | Texto más propiedades interactivas        | Las selecciones y los divisores no son compatibles; esos bloques se degradan a texto.                                                                                                                                             |
| Microsoft Teams | Adaptive Cards                            | El texto `message` sin formato se incluye con la tarjeta cuando se proporcionan ambos. Las selecciones, los estilos y el estado deshabilitado no son compatibles.                                                                                     |
| Slack           | Block Kit                                 | Renderiza `chart` como `data_visualization` nativo y `table` como `data_table` nativo; conserva `channelData.slack.blocks` heredado, pero los nuevos envíos compartidos deben utilizar `presentation`.                                   |
| Telegram        | Texto más teclados insertados             | Los botones y las selecciones requieren la capacidad de botones insertados en la superficie de destino; de lo contrario, se utiliza texto alternativo.                                                                                                         |
| Canales simples | Texto alternativo                         | Los canales sin renderizador siguen recibiendo una salida legible.                                                                                                                                                            |

La compatibilidad con cargas útiles nativas del proveedor es un recurso de transición para los
productores de respuestas existentes. No es una razón para añadir nuevos campos nativos compartidos.

## Presentación frente a InteractiveReply

`InteractiveReply` es el subconjunto interno anterior utilizado por los ayudantes de aprobación e interacción.
Admite:

- texto
- botones
- selecciones

`MessagePresentation` es el contrato canónico de envío compartido. Añade:

- título
- tono
- contexto
- divisor
- gráfico
- tabla
- botones que solo contienen URL
- metadatos genéricos de entrega mediante `ReplyPayload.delivery`

Utilice los ayudantes de `openclaw/plugin-sdk/interactive-runtime` al conectar código
anterior:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

El código nuevo debe aceptar o producir `MessagePresentation` directamente. Las cargas útiles
`interactive` existentes son un subconjunto obsoleto de `presentation`; se mantiene la compatibilidad en tiempo de ejecución
para productores anteriores.

Ayudantes no obsoletos que conviene conocer:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  validan y convierten una carga sin tipar (por ejemplo, JSON de la opción
  `--presentation` de la CLI) en `MessagePresentation`.
- `isMessagePresentationInteractiveBlock(block)` restringe un bloque a la unión
  `buttons` | `select`.
- `resolveMessagePresentationButtonAction(button)` y
  `resolveMessagePresentationOptionAction(option)` devuelven la acción tipada canónica
  a la vez que aceptan campos de límite obsoletos. Un valor `action` explícito
  siempre tiene prioridad.
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` solo leen valores escalares
  de comandos/devoluciones de llamada. Una acción canónica no escalar nunca pasa a un
  valor `value` heredado en segundo plano, por lo que los identificadores de aprobación y los destinos de los enlaces permanecen tipados.
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` representan un bloque de datos
  estructurados como texto determinista para las rutas de respaldo específicas del canal.

Los tipos `InteractiveReply*` heredados y los auxiliares de conversión están marcados como
`@deprecated` en el SDK:

- `InteractiveReply`, `InteractiveReplyBlock`, `InteractiveReplyButton` y
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` y
`presentationToInteractiveControlsReply(...)` siguen disponibles como puentes de representación
para implementaciones de canales heredadas. El nuevo código productor no debe llamarlos;
debe enviar `presentation` y permitir que la adaptación del núcleo/canal gestione la representación.

Los auxiliares de aprobación también tienen sustitutos que priorizan la presentación:

- use `buildApprovalPresentation(...)` en lugar de
  `buildApprovalInteractiveReply(...)`
- use `buildExecApprovalPresentation(...)` en lugar de
  `buildExecApprovalInteractiveReply(...)`

Esos constructores publicados siguen estando respaldados por comandos para mantener la compatibilidad de los plugins. El Gateway
y el código de canales incluido que posea un tipo de aprobación persistente deben usar
`buildTypedApprovalPresentation(...)`,
`buildTypedExecApprovalPendingReplyPayload(...)` o
`buildTypedPluginApprovalPendingReplyPayload(...)` para que los transportes reciban una
acción `approval` explícita en lugar de inferir la semántica del texto `/approve`.

`renderMessagePresentationFallbackText(...)` devuelve una cadena vacía para
los bloques de presentación que no tienen texto de respaldo, como una presentación
que solo contiene un separador. Los transportes que requieran un cuerpo de envío no vacío pueden pasar
`emptyFallback` para habilitar un cuerpo mínimo sin cambiar el contrato de respaldo
predeterminado.

## Fijación en la entrega

La fijación es un comportamiento de entrega, no de presentación. Use `delivery.pin` en lugar de
campos nativos del proveedor como `channelData.telegram.pin`.

Semántica:

- `pin: true` fija el primer mensaje entregado correctamente.
- `pin.notify` tiene como valor predeterminado `false`.
- `pin.required` tiene como valor predeterminado `false`.
- Los fallos de fijación opcional se degradan y dejan intacto el mensaje enviado.
- Los fallos de fijación obligatoria hacen que la entrega falle.
- Los mensajes fragmentados fijan el primer fragmento entregado, no el último.

Las acciones de mensaje manuales `pin`, `unpin` y `pins` siguen existiendo para los
mensajes existentes cuando el proveedor admite esas operaciones.

## Lista de comprobación para autores de plugins

- Declare `presentation` desde `describeMessageTool(...)` cuando el canal pueda
  representar o degradar de forma segura la presentación semántica.
- Añada `presentationCapabilities` al adaptador de salida en tiempo de ejecución.
- Implemente `renderPresentation` en el código de tiempo de ejecución, no en el código de
  configuración del plugin del plano de control.
- Mantenga las bibliotecas de interfaz de usuario nativas fuera de las rutas críticas de configuración/catálogo.
- Declare límites genéricos de capacidad en `presentationCapabilities.limits` cuando
  se conozcan.
- Conserve los límites finales de la plataforma en el representador y las pruebas.
- Añada pruebas de respaldo para gráficos, tablas, botones, selectores y botones de URL
  no compatibles, duplicación de título/texto y envíos mixtos de `message` más `presentation`.
- Añada compatibilidad con la fijación en la entrega mediante `deliveryCapabilities.pin` y
  `pinDeliveredMessage` solo cuando el proveedor pueda fijar el identificador del mensaje enviado.
- No exponga nuevos campos de tarjetas/bloques/componentes/botones nativos del proveedor mediante
  el esquema compartido de acciones de mensaje.

## Documentación relacionada

- [CLI de mensajes](/es/cli/message)
- [Descripción general del SDK de plugins](/es/plugins/sdk-overview)
- [Arquitectura de plugins](/es/plugins/architecture-internals#message-tool-schemas)
- [Plan de refactorización de la presentación de canales](/es/plan/ui-channels)
