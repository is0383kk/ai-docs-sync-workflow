---
read_when:
    - Trabajar con reacciones en cualquier canal
    - Comprender cómo difieren las reacciones con emojis entre plataformas
summary: Semántica de la herramienta de reacciones en todos los canales compatibles
title: Reacciones
x-i18n:
    generated_at: "2026-07-26T05:01:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e148a93edbcfbe997075f6e9e191667ec257f76fa48162688fd1f333479661f0
    source_path: tools/reactions.md
    workflow: 16
---

El agente añade y elimina reacciones de emoji con la acción `react`
de la herramienta `message`. El comportamiento varía según el canal.

## Cómo funciona

```json
{
  "action": "react",
  "messageId": "msg-123",
  "emoji": "thumbsup"
}
```

- `emoji` es obligatorio al añadir una reacción.
- Establezca `emoji` en una cadena vacía (`""`) para eliminar las reacciones del bot en
  los canales que lo admitan.
- Establezca `remove: true` para eliminar un emoji específico (requiere que
  `emoji` no esté vacío).
- En los canales con reacciones de estado, `trackToolCalls: true` en una reacción permite
  que el entorno de ejecución reutilice ese mensaje con reacción para las reacciones posteriores
  de progreso de la herramienta durante el mismo turno.

## Comportamiento por canal

<AccordionGroup>
  <Accordion title="Discord y Slack">
    - Un valor vacío de `emoji` elimina todas las reacciones del bot en el mensaje.
    - `remove: true` elimina solo el emoji especificado.

  </Accordion>

  <Accordion title="Nextcloud Talk">
    - Solo se pueden añadir reacciones: `emoji` es obligatorio y no debe estar vacío.
    - La eliminación de reacciones aún no está conectada a una llamada de eliminación; `remove: true` se rechaza con un error explícito en lugar de no realizar ninguna operación silenciosamente.
    - Requiere que el bot de Talk esté registrado con la función `reaction` (consulte la [documentación del canal Nextcloud Talk](/es/channels/nextcloud-talk)).

  </Accordion>

  <Accordion title="Telegram">
    - Un valor vacío de `emoji` elimina las reacciones del bot.
    - `remove: true` también elimina las reacciones, pero sigue requiriendo que `emoji` no esté vacío para validar la herramienta.

  </Accordion>

  <Accordion title="WhatsApp">
    - Un valor vacío de `emoji` elimina la reacción del bot.
    - `remove: true` se asigna internamente a un emoji vacío (sigue requiriendo `emoji` en la llamada a la herramienta).
    - WhatsApp tiene un espacio para una reacción del bot por mensaje; enviar una reacción nueva sustituye la anterior en lugar de acumular varios emojis.

  </Accordion>

  <Accordion title="Zalo Personal (zalouser)">
    - Requiere que `emoji` no esté vacío tanto para añadir como para eliminar.
    - `remove: true` elimina esa reacción de emoji específica.

  </Accordion>

  <Accordion title="Feishu/Lark">
    - Utiliza la misma acción `react` que los demás canales (añadir, eliminar y enumerar mediante identificadores de reacciones a mensajes), no una herramienta independiente.
    - Para añadir se requiere que `emoji` no esté vacío (se asigna a un `emoji_type` de Feishu, por ejemplo, `SMILE`, `THUMBSUP`, `HEART`).
    - `remove: true` requiere que `emoji` no esté vacío y elimina la reacción propia del bot que coincida con ese tipo de emoji.
    - Un valor vacío de `emoji` con `clearAll: true` elimina todas las reacciones del bot en el mensaje.

  </Accordion>

  <Accordion title="Signal">
    - Las notificaciones de reacciones entrantes se controlan mediante `channels.signal.reactionNotifications`: `"off"` las desactiva, `"own"` (valor predeterminado) emite eventos cuando los usuarios reaccionan a los mensajes del bot, `"all"` emite eventos para todas las reacciones y `"allowlist"` emite eventos solo para los remitentes incluidos en `channels.signal.reactionAllowlist`.

  </Accordion>

  <Accordion title="iMessage">
    - Las reacciones salientes son tapbacks de iMessage (`love`, `like`, `dislike`, `laugh`, `emphasize` y `question`); `emoji` debe corresponder a uno de estos tipos para añadir una reacción.
    - `remove: true` sin un tipo de tapback reconocido elimina todos los tipos de tapback; con un tipo reconocido, elimina solo ese.

  </Accordion>
</AccordionGroup>

## Nivel de reacción

El valor `reactionLevel` de cada canal limita la frecuencia con la que el agente envía sus propias
reacciones. Valores: `off`, `ack`, `minimal` o `extensive`.

- [Notificaciones de reacciones de Telegram](/es/channels/telegram#feature-reference) - `channels.telegram.reactionLevel` (valor predeterminado: `minimal`)
- [Nivel de reacción de WhatsApp](/es/channels/whatsapp#reaction-level) - `channels.whatsapp.reactionLevel` (valor predeterminado: `minimal`)
- [Reacciones de Signal](/es/channels/signal#reactions-message-tool) - `channels.signal.reactionLevel` (valor predeterminado: `minimal`)

## Contenido relacionado

- [Envío del agente](/es/tools/agent-send) - la herramienta `message` que incluye `react`
- [Canales](/es/channels) - configuración específica de cada canal
