---
read_when:
    - Refactorización de la interfaz de mensajes de canales, las cargas interactivas o los renderizadores nativos de canales
    - Cambio de las capacidades de la herramienta de mensajes, las indicaciones de entrega o los marcadores entre contextos
    - Depuración de la distribución de importaciones de Carbon de Discord o de la carga diferida del entorno de ejecución del plugin de canal
summary: Desacoplar la presentación semántica de mensajes de los renderizadores de interfaz de usuario nativos del canal.
title: Plan de refactorización de la presentación de canales
x-i18n:
    generated_at: "2026-07-26T04:46:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## Estado

Implementado para las superficies del agente compartido, la CLI, las capacidades de plugins y la entrega saliente:

- `ReplyPayload.presentation` transporta la interfaz semántica de mensajes.
- `ReplyPayload.delivery.pin` transporta solicitudes para fijar mensajes enviados.
- Las acciones de mensajes compartidas exponen `presentation`, `delivery` y `pin` en lugar de los elementos nativos del proveedor `components`, `blocks`, `buttons` o `card`.
- El núcleo renderiza o degrada automáticamente la presentación mediante las capacidades de salida declaradas por los plugins.
- Los renderizadores de Discord, Slack, Telegram, Mattermost, MS Teams y Feishu consumen el contrato genérico.
- El código del plano de control del canal de Discord ya no importa contenedores de interfaz respaldados por Carbon.

La documentación canónica ahora se encuentra en [Presentación de mensajes](/es/plugins/message-presentation).
Conserve este plan como contexto histórico de implementación; actualice la guía canónica
cuando cambie el contrato, el renderizador o el comportamiento de respaldo.

## Problema

Actualmente, la interfaz de los canales está dividida entre varias superficies incompatibles:

- El núcleo posee un hook de renderización entre contextos con la estructura de Discord mediante `buildCrossContextComponents`.
- `channel.ts` de Discord puede importar la interfaz nativa de Carbon mediante `DiscordUiContainer`, lo que incorpora dependencias de interfaz en tiempo de ejecución al plano de control del plugin del canal.
- El agente y la CLI exponen mecanismos de escape para cargas útiles nativas, como `components` de Discord, `blocks` de Slack, `buttons` de Telegram o Mattermost y `card` de Teams o Feishu.
- `ReplyPayload.channelData` transporta tanto indicaciones de transporte como envoltorios de interfaz nativos.
- El modelo genérico `interactive` existe, pero es más limitado que los diseños más completos que ya utilizan Discord, Slack, Teams, Feishu, LINE, Telegram y Mattermost.

Esto hace que el núcleo conozca las estructuras de interfaz nativas, debilita la carga diferida del entorno de ejecución de plugins y ofrece a los agentes demasiadas formas específicas de cada proveedor para expresar la misma intención de mensaje.

## Objetivos

- El núcleo decide la mejor presentación semántica para un mensaje a partir de las capacidades declaradas.
- Las extensiones declaran capacidades y renderizan la presentación semántica como cargas útiles nativas de transporte.
- La interfaz web de control permanece separada de la interfaz nativa de chat.
- Las cargas útiles nativas de los canales no se exponen mediante la superficie compartida de mensajes del agente ni de la CLI.
- Las funciones de presentación no compatibles se degradan automáticamente a la mejor representación textual.
- El comportamiento de entrega, como fijar un mensaje enviado, constituye metadatos genéricos de entrega, no de presentación.

## No objetivos

- Ninguna capa de compatibilidad con versiones anteriores para `buildCrossContextComponents`.
- Ningún mecanismo público de escape nativo para `components`, `blocks`, `buttons` o `card`.
- Ninguna importación en el núcleo de bibliotecas de interfaz nativas de los canales.
- Ninguna interfaz del SDK específica del proveedor para los canales incluidos.

## Modelo de destino

Añada un campo `presentation`, propiedad del núcleo, a `ReplyPayload`.

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

`interactive` se convierte en un subconjunto de `presentation` durante la migración:

- El bloque de texto `interactive` se asigna a `presentation.blocks[].type = "text"`.
- El bloque de botones `interactive` se asigna a `presentation.blocks[].type = "buttons"`.
- El bloque de selección `interactive` se asigna a `presentation.blocks[].type = "select"`.

Los esquemas externos del agente y de la CLI ahora utilizan `presentation`; `interactive` permanece como auxiliar interno heredado de análisis y renderización para los productores de respuestas existentes.
La API pública orientada a productores trata `interactive` como obsoleto. La compatibilidad
en tiempo de ejecución se mantiene para que los auxiliares de aprobación existentes y los plugins más antiguos sigan
funcionando mientras el código nuevo emite `presentation`.

## Metadatos de entrega

Añada un campo `delivery`, propiedad del núcleo, para el comportamiento de envío que no corresponde a la interfaz.

```ts
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

Semántica:

- `delivery.pin = true` significa fijar el primer mensaje entregado correctamente.
- `notify` tiene como valor predeterminado `false`.
- `required` tiene como valor predeterminado `false`; los canales no compatibles o los errores al fijar se degradan automáticamente y permiten que la entrega continúe.
- Las acciones manuales de mensajes `pin`, `unpin` y `list-pins` permanecen disponibles para los mensajes existentes.

La vinculación actual de temas ACP de Telegram debe trasladarse de `channelData.telegram.pin = true` a `delivery.pin = true`.

## Contrato de capacidades del entorno de ejecución

Añada hooks de renderización de presentación y entrega al adaptador de salida del entorno de ejecución, no al plugin del canal del plano de control.

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
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

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

Comportamiento del núcleo:

- Resolver el canal de destino y el adaptador del entorno de ejecución.
- Consultar las capacidades de presentación.
- Degradar los bloques no compatibles y aplicar los límites genéricos de capacidad antes de
  renderizar.
- Invocar `renderPresentation`.
- Si no existe ningún renderizador, convertir la presentación en texto de respaldo.
- Tras un envío correcto, invocar `pinDeliveredMessage` cuando se solicite `delivery.pin` y sea compatible.

## Asignación de canales

Discord:

- Renderizar `presentation` como componentes v2 y contenedores de Carbon en módulos exclusivos del entorno de ejecución.
- Mantener los auxiliares de color de realce en módulos ligeros.
- Eliminar las importaciones de `DiscordUiContainer` del código del plano de control del plugin del canal.

Slack:

- Renderizar `presentation` como Block Kit.
- Eliminar la entrada `blocks` del agente y de la CLI.

Telegram:

- Renderizar el texto, el contexto y los divisores como texto.
- Renderizar las acciones y la selección como teclados en línea cuando estén configurados y permitidos para la superficie de destino.
- Utilizar texto de respaldo cuando los botones en línea estén desactivados.
- Trasladar la fijación de temas ACP a `delivery.pin`.

Mattermost:

- Renderizar las acciones como botones interactivos cuando estén configuradas.
- Renderizar los demás bloques como texto de respaldo.

MS Teams:

- Renderizar `presentation` como Adaptive Cards.
- Mantener las acciones manuales para fijar, dejar de fijar y enumerar los elementos fijados.
- Implementar opcionalmente `pinDeliveredMessage` si la compatibilidad con Graph es fiable para la conversación de destino.

Feishu:

- Renderizar `presentation` como tarjetas interactivas.
- Mantener las acciones manuales para fijar, dejar de fijar y enumerar los elementos fijados.
- Implementar opcionalmente `pinDeliveredMessage` para fijar mensajes enviados si el comportamiento de la API es fiable.

LINE:

- Renderizar `presentation` como mensajes Flex o de plantilla cuando sea posible.
- Utilizar texto de respaldo para los bloques no compatibles.
- Eliminar las cargas útiles de interfaz de LINE de `channelData`.

Canales de texto sin formato o limitados:

- Convertir la presentación en texto con un formato conservador.

## Pasos de refactorización

1. Volver a aplicar la corrección de la versión de Discord que separa `ui-colors.ts` de la interfaz respaldada por Carbon y elimina `DiscordUiContainer` de `extensions/discord/src/channel.ts`.
2. Añadir `presentation` y `delivery` a `ReplyPayload`, la normalización de cargas útiles salientes, los resúmenes de entrega y las cargas útiles de hooks.
3. Añadir el esquema `MessagePresentation` y los auxiliares de análisis en una subruta específica del SDK o del entorno de ejecución.
4. Sustituir las capacidades de mensajes `buttons`, `cards`, `components` y `blocks` por capacidades de presentación semántica.
5. Añadir hooks al adaptador de salida del entorno de ejecución para renderizar la presentación y fijar elementos durante la entrega.
6. Sustituir la construcción de componentes entre contextos por `buildCrossContextPresentation`.
7. Eliminar `src/infra/outbound/channel-adapters.ts` y quitar `buildCrossContextComponents` de los tipos de plugins de canales.
8. Cambiar `maybeApplyCrossContextMarker` para adjuntar `presentation` en lugar de parámetros nativos.
9. Actualizar las rutas de envío del despacho de plugins para que consuman únicamente la presentación semántica y los metadatos de entrega.
10. Eliminar los parámetros de cargas útiles nativas del agente y de la CLI: `components`, `blocks`, `buttons` y `card`.
11. Eliminar los auxiliares del SDK que crean esquemas nativos de herramientas de mensajes y sustituirlos por auxiliares de esquemas de presentación.
12. Eliminar los envoltorios de interfaz o nativos de `channelData`; conservar únicamente los metadatos de transporte hasta revisar cada campo restante.
13. Migrar los renderizadores de Discord, Slack, Telegram, Mattermost, MS Teams, Feishu y LINE.
14. Actualizar la documentación de la CLI de mensajes, las páginas de canales, el SDK de plugins y el recetario de capacidades.
15. Ejecutar un análisis de la propagación de importaciones para Discord y los puntos de entrada de los canales afectados.

Los pasos 1-11 y 13-14 se implementan en esta refactorización para los contratos del agente compartido, la CLI, las capacidades de plugins y el adaptador de salida. El paso 12 queda pendiente como una fase más profunda de limpieza interna de los envoltorios de transporte `channelData` privados del proveedor. El paso 15 queda pendiente como validación posterior si se desean cifras cuantificadas de propagación de importaciones más allá de la barrera de tipos y pruebas.

## Pruebas

Añadir o actualizar:

- Pruebas de normalización de la presentación.
- Pruebas de degradación automática de la presentación para bloques no compatibles.
- Pruebas de marcadores entre contextos para el despacho de plugins y las rutas de entrega del núcleo.
- Pruebas de la matriz de renderización de canales para Discord, Slack, Telegram, Mattermost, MS Teams, Feishu, LINE y el respaldo de texto.
- Pruebas del esquema de herramientas de mensajes que demuestren que los campos nativos han desaparecido.
- Pruebas de la CLI que demuestren que las opciones nativas han desaparecido.
- Prueba de regresión de la carga diferida de importaciones del punto de entrada de Discord que abarque Carbon.
- Pruebas de fijación durante la entrega que abarquen Telegram y el respaldo genérico.

## Preguntas abiertas

- ¿Debería implementarse `delivery.pin` para Discord, Slack, MS Teams y Feishu en la primera fase, o inicialmente solo para Telegram?
- ¿Debería `delivery` incorporar finalmente campos existentes como `replyToId`, `replyToCurrent`, `silent` y `audioAsVoice`, o mantenerse centrado en los comportamientos posteriores al envío?
- ¿Debería la presentación admitir directamente imágenes o referencias a archivos, o deberían los elementos multimedia permanecer separados del diseño de la interfaz de usuario por ahora?

## Relacionado

- [Descripción general de los canales](/es/channels)
- [Presentación de mensajes](/es/plugins/message-presentation)
