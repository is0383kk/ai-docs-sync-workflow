---
read_when:
    - Configuración de la misma lista de permitidos en varios canales de mensajes
    - Compartir reglas de acceso de remitentes para mensajes directos y grupos
    - Revisión del control de acceso de los canales de mensajería
summary: Listas reutilizables de remitentes permitidos para canales de mensajería
title: Grupos de acceso
x-i18n:
    generated_at: "2026-07-26T05:01:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 099abc95e90d9a7b7006d19062c46b4ffdb2aecb1e8e714454a3182131a786d0
    source_path: channels/access-groups.md
    workflow: 16
---

Los grupos de acceso son listas de remitentes con nombre que se definen una vez en `accessGroups` y se referencian desde las listas de permitidos de los canales mediante `accessGroup:<name>`.

Úselos cuando se deba permitir a las mismas personas en varios canales de mensajes o cuando un conjunto de confianza deba aplicarse tanto a los mensajes directos como a la autorización de remitentes de grupos.

Un grupo no concede nada por sí mismo. Solo tiene efecto cuando un campo de lista de permitidos lo referencia.

## Grupos estáticos de remitentes de mensajes

Los grupos estáticos de remitentes usan `type: "message.senders"`. `members` se organiza por id. de canal de mensajes, además de `"*"` para las entradas compartidas por todos los canales:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
}
```

| Clave                        | Significado                                                                     |
| -------------------------- | --------------------------------------------------------------------------- |
| `"*"`                      | Entradas compartidas que se comprueban para cada canal de mensajes que referencia el grupo. |
| `discord`, `telegram`, ... | Entradas que se comprueban únicamente para la coincidencia con la lista de permitidos de ese canal.                 |

Las entradas se comparan mediante las reglas normales de `allowFrom` del canal de destino. OpenClaw no convierte los id. de remitente entre canales: si Alice tiene un id. de Telegram y otro de Discord, incluya ambos id. bajo las claves de canal correspondientes.

## Referenciar grupos desde listas de permitidos

Referencie un grupo mediante `accessGroup:<name>` en cualquier lugar donde la ruta del canal de mensajes admita listas de remitentes permitidos.

Ejemplo de lista de permitidos de mensajes directos:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

Ejemplo de lista de remitentes permitidos de grupos:

```json5
{
  accessGroups: {
    oncall: {
      type: "message.senders",
      members: {
        whatsapp: ["+15551234567"],
        googlechat: ["users/1234567890"],
      },
    },
  },
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["accessGroup:oncall"],
    },
    googlechat: {
      groups: {
        "spaces/AAA": {
          users: ["accessGroup:oncall"],
        },
      },
    },
  },
}
```

Se pueden combinar grupos y entradas directas:

```json5
{
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators", "discord:123456789012345678"],
    },
  },
}
```

## Rutas de canales de mensajes compatibles

Los grupos de acceso funcionan en las rutas compartidas de autorización de canales de mensajes:

- Listas de remitentes permitidos de mensajes directos, como `channels.<channel>.allowFrom`
- Listas de remitentes permitidos de grupos, como `channels.<channel>.groupAllowFrom`
- Listas de remitentes permitidos por sala específicas de cada canal que usan las mismas reglas de coincidencia de remitentes (por ejemplo, `groups.<space>.users` de Google Chat)
- Rutas de autorización de comandos que reutilizan las listas de remitentes permitidos de los canales de mensajes

La compatibilidad de cada canal depende de si está conectado mediante los asistentes compartidos de autorización de remitentes de OpenClaw. La compatibilidad incluida actualmente abarca ClickClack, Discord, Feishu, Google Chat, iMessage, IRC, LINE, Mattermost, Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Signal, Slack, SMS, Telegram, WhatsApp, Zalo y Zalo Personal. Los grupos estáticos `message.senders` son independientes del canal, por lo que los nuevos canales de mensajes los obtienen al usar los asistentes de entrada compartidos del SDK de plugins en lugar de una expansión personalizada de la lista de permitidos.

## Audiencias de canales de Discord

Discord también admite un tipo de grupo de acceso dinámico:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

`discord.channelAudience` significa «permitir remitentes de mensajes directos de Discord que actualmente puedan ver este canal del servidor». OpenClaw resuelve el remitente mediante Discord en el momento de la autorización y aplica las reglas de permisos `ViewChannel` de Discord. `membership` es opcional y su valor predeterminado es `canViewChannel`.

Úselo cuando un canal de Discord ya sea la fuente de verdad de un equipo, como `#maintainers` o `#on-call`.

Requisitos y comportamiento ante fallos:

- El bot necesita acceso al servidor y al canal.
- El bot necesita **Server Members Intent** en Discord Developer Portal.
- El grupo de acceso deniega el acceso de forma predeterminada cuando Discord devuelve `Missing Access`, el remitente no se puede resolver como miembro del servidor o el canal pertenece a otro servidor.

Más ejemplos específicos de Discord: [Control de acceso de Discord](/es/channels/discord#access-control-and-routing)

## Diagnósticos de plugins

Los autores de plugins pueden inspeccionar el estado estructurado de los grupos de acceso sin volver a expandirlo en una lista de permitidos plana:

```typescript
import { resolveAccessGroupAllowFromState } from "openclaw/plugin-sdk/access-groups";

const state = await resolveAccessGroupAllowFromState({
  accessGroups: cfg.accessGroups,
  allowFrom: channelConfig.allowFrom,
  channel: "my-channel",
  accountId: "default",
  senderId,
  isSenderAllowed,
});
```

El resultado informa sobre los grupos referenciados, coincidentes, ausentes, no compatibles y fallidos. Úselo para diagnósticos o pruebas de conformidad. Use `expandAllowFromWithAccessGroups(...)` únicamente para rutas de compatibilidad que todavía esperan un array `allowFrom` plano.

## Notas de seguridad

- Los grupos de acceso son alias de listas de permitidos, no roles. No crean propietarios, aprueban solicitudes de emparejamiento ni conceden permisos de herramientas por sí mismos.
- `dmPolicy: "open"` sigue requiriendo `"*"` en la lista efectiva de mensajes directos permitidos. Referenciar un grupo de acceso no equivale a permitir el acceso público.
- Los nombres de grupo ausentes deniegan el acceso de forma predeterminada. Si `allowFrom` contiene `accessGroup:operators` y `accessGroups.operators` no existe, esa entrada no autoriza a nadie.
- Mantenga estables los id. de canal. Prefiera los id. numéricos o de usuario a los nombres para mostrar cuando el canal admita ambos.

## Solución de problemas

Si un remitente debería coincidir, pero está bloqueado:

1. Confirme que el campo de la lista de permitidos contiene la referencia `accessGroup:<name>` exacta.
2. Confirme que `accessGroups.<name>.type` sea correcto.
3. Confirme que el id. del remitente figure bajo la clave de canal correspondiente o bajo `"*"`.
4. Confirme que la entrada use la sintaxis normal de la lista de permitidos de ese canal.
5. Para las audiencias de canales de Discord, confirme que el bot pueda ver el canal del servidor y que Server Members Intent esté habilitado.

Ejecute `openclaw doctor` después de editar la configuración del control de acceso. Detecta muchas combinaciones no válidas de listas de permitidos y políticas antes de la ejecución.
