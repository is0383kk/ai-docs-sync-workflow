---
read_when:
    - Quiere conectar un bot de Yuanbao
    - Está configurando el canal Yuanbao
summary: Descripción general, funciones y configuración del bot Yuanbao
title: Yuanbao
x-i18n:
    generated_at: "2026-07-26T05:01:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 43488834f588530206b290cb0fb185fd1fe2e1f214ab4a4ccccc49b9b549b6ac
    source_path: channels/yuanbao.md
    workflow: 16
---

Tencent Yuanbao es la plataforma de asistente de IA de Tencent. El plugin `openclaw-plugin-yuanbao`, mantenido por la comunidad, conecta los bots de Yuanbao con OpenClaw mediante WebSocket para mensajes directos y chats de grupo.

**Estado:** listo para producción para MD con bots y chats de grupo. WebSocket es el único modo de conexión compatible. Este plugin lo mantiene el equipo de Tencent Yuanbao como una entrada de catálogo externa, no el núcleo de OpenClaw; los detalles de configuración y comportamiento que aparecen a continuación (salvo la instalación y la superficie genérica de la CLI) proceden de la documentación del propio plugin y no se han verificado con el código fuente del núcleo de OpenClaw.

## Inicio rápido

Requiere OpenClaw 2026.4.10 o posterior. Compruébelo con `openclaw --version`; actualice con `openclaw update`.

<Steps>
  <Step title="Añadir el canal de Yuanbao con sus credenciales">
  ```bash
  openclaw channels add --channel yuanbao --token "appKey:appSecret"
  ```
  `--token` utiliza `appKey:appSecret` separados por dos puntos. Obténgalos de la aplicación Yuanbao creando un bot en la configuración de su aplicación.
  </Step>

  <Step title="Reiniciar el Gateway para aplicar el cambio">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

### Configuración interactiva (alternativa)

```bash
openclaw channels login --channel yuanbao
```

Siga las indicaciones para introducir su App ID y App Secret.

## Control de acceso

### Mensajes directos

`channels.yuanbao.dm.policy`:

| Valor            | Comportamiento                                          |
| ---------------- | ------------------------------------------------- |
| `open` (predeterminado) | Permitir a todos los usuarios                                   |
| `pairing`        | Los usuarios desconocidos reciben un código de vinculación; se aprueba mediante la CLI |
| `allowlist`      | Solo pueden chatear los usuarios incluidos en `allowFrom`                |
| `disabled`       | Desactivar todos los MD                                   |

Aprobar una solicitud de vinculación:

```bash
openclaw pairing list yuanbao
openclaw pairing approve yuanbao <CODE>
```

### Chats de grupo

`channels.yuanbao.requireMention` (valor predeterminado: `true`): requiere una @mención antes de que el bot responda en un grupo. Responder al propio mensaje del bot se considera una mención implícita.

## Ejemplos de configuración

Configuración básica con una política de MD abierta:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "open",
      },
    },
  },
}
```

Restringir los MD a usuarios específicos:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "allowlist",
        allowFrom: ["user_id_1", "user_id_2"],
      },
    },
  },
}
```

Desactivar el requisito de @mención en los grupos:

```json5
{
  channels: {
    yuanbao: {
      requireMention: false,
    },
  },
}
```

Ajuste de la entrega saliente:

```json5
{
  channels: {
    yuanbao: {
      outboundQueueStrategy: "merge-text",
      minChars: 2800, // almacenar en búfer hasta alcanzar esta cantidad de caracteres
      maxChars: 3000, // forzar la división por encima de este límite
      idleMs: 5000, // vaciar automáticamente tras el tiempo de espera de inactividad (ms)
    },
  },
}
```

Establezca `outboundQueueStrategy: "immediate"` para enviar cada fragmento sin almacenarlo en búfer.

## Comandos habituales

| Comando    | Descripción                 |
| ---------- | --------------------------- |
| `/help`    | Mostrar los comandos disponibles     |
| `/status`  | Mostrar el estado del bot             |
| `/new`     | Iniciar una sesión nueva         |
| `/stop`    | Detener la ejecución actual        |
| `/restart` | Reiniciar OpenClaw            |
| `/compact` | Compactar el contexto de la sesión |

Yuanbao admite menús nativos de comandos con barra; los comandos se sincronizan automáticamente con la plataforma cuando se inicia el Gateway.

## Solución de problemas

**El bot no responde en los chats de grupo:**

1. Confirme que el bot se haya añadido al grupo
2. Confirme que @menciona al bot (obligatorio de forma predeterminada)
3. Compruebe los registros: `openclaw logs --follow`

**El bot no recibe mensajes:**

1. Confirme que el bot se haya creado y aprobado en la aplicación Yuanbao
2. Confirme que `appKey` y `appSecret` estén configurados correctamente
3. Confirme que el Gateway esté en ejecución: `openclaw gateway status`
4. Compruebe los registros: `openclaw logs --follow`

**El bot envía respuestas vacías o de reserva:**

1. Compruebe si el modelo de IA devuelve contenido válido
2. Respuesta de reserva predeterminada: "Por ahora no puedo responder; puede hacerme otra pregunta"
3. Personalícela con `channels.yuanbao.fallbackReply`

**Se ha filtrado el App Secret:**

1. Restablezca el App Secret en la aplicación Yuanbao
2. Actualice el valor en su configuración
3. Reinicie el Gateway: `openclaw gateway restart`

## Configuración avanzada

### Varias cuentas

```json5
{
  channels: {
    yuanbao: {
      defaultAccount: "main",
      accounts: {
        main: {
          appKey: "key_xxx",
          appSecret: "secret_xxx",
          name: "Bot principal",
        },
        backup: {
          appKey: "key_yyy",
          appSecret: "secret_yyy",
          name: "Bot de respaldo",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` controla qué cuenta se utiliza cuando las API salientes no especifican un `accountId`.

### Límites de mensajes

- `maxChars`: número máximo de caracteres de un solo mensaje (valor predeterminado: `3000`)
- `mediaMaxMb`: límite de carga y descarga de contenido multimedia (valor predeterminado: `20` MB)
- `overflowPolicy`: comportamiento cuando un mensaje supera el límite, `"split"` (predeterminado) o `"stop"`

### Transmisión

Yuanbao admite salida en streaming a nivel de bloque; el bot envía el texto en fragmentos a medida que lo genera.

```json5
{
  channels: {
    yuanbao: {
      disableBlockStreaming: false, // streaming por bloques activado (valor predeterminado)
    },
  },
}
```

Establezca `disableBlockStreaming: true` para enviar la respuesta completa en un solo mensaje.

### Contexto del historial de chats de grupo

```json5
{
  channels: {
    yuanbao: {
      historyLimit: 100, // valor predeterminado: 100; establecer en 0 para desactivar
    },
  },
}
```

Controla cuántos mensajes históricos se incluyen en el contexto de IA para los chats de grupo.

### Modo de respuesta

```json5
{
  channels: {
    yuanbao: {
      replyToMode: "first", // "off" | "first" | "all" (valor predeterminado: "first")
    },
  },
}
```

| Valor   | Comportamiento                                                 |
| ------- | -------------------------------------------------------- |
| `off`   | Sin respuesta con cita                                           |
| `first` | Citar solo la primera respuesta por mensaje entrante (valor predeterminado) |
| `all`   | Citar todas las respuestas                                        |

### Inyección de indicaciones de Markdown

De forma predeterminada, el bot inyecta una instrucción en el prompt del sistema para evitar que el modelo envuelva toda la respuesta en un bloque de código Markdown.

```json5
{
  channels: {
    yuanbao: {
      markdownHintEnabled: true, // valor predeterminado: true
    },
  },
}
```

### Modo de depuración

```json5
{
  channels: {
    yuanbao: {
      debugBotIds: ["bot_user_id_1", "bot_user_id_2"],
    },
  },
}
```

Activa la salida de registros sin sanear para los ID de bot enumerados.

### Enrutamiento multiagente

Utilice `bindings` para dirigir los MD o grupos de Yuanbao a distintos agentes:

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "yuanbao",
        peer: { kind: "direct", id: "user_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "yuanbao",
        peer: { kind: "group", id: "group_zzz" },
      },
    },
  ],
}
```

- `match.channel`: `"yuanbao"`
- `match.peer.kind`: `"direct"` (MD) o `"group"` (chat de grupo)
- `match.peer.id`: ID de usuario o código de grupo

## Referencia de configuración

Configuración completa: [Configuración del Gateway](/es/gateway/configuration)

| Ajuste                                    | Descripción                                       | Valor predeterminado                                |
| ------------------------------------------ | ------------------------------------------------- | -------------------------------------- |
| `channels.yuanbao.enabled`                 | Activar o desactivar el canal                        | `true`                                 |
| `channels.yuanbao.defaultAccount`          | Cuenta predeterminada para el enrutamiento saliente              | `default`                              |
| `channels.yuanbao.accounts.<id>.appKey`    | App Key (firma y generación de tickets)             | -                                      |
| `channels.yuanbao.accounts.<id>.appSecret` | App Secret (firma)                              | -                                      |
| `channels.yuanbao.accounts.<id>.token`     | Token firmado previamente (omite la firma automática de tickets) | -                                      |
| `channels.yuanbao.accounts.<id>.name`      | Nombre visible de la cuenta                              | -                                      |
| `channels.yuanbao.accounts.<id>.enabled`   | Activar o desactivar una cuenta específica                 | `true`                                 |
| `channels.yuanbao.dm.policy`               | Política de MD                                         | `open`                                 |
| `channels.yuanbao.dm.allowFrom`            | Lista de permitidos para MD (lista de ID de usuario)                       | -                                      |
| `channels.yuanbao.requireMention`          | Requerir una @mención en los grupos                        | `true`                                 |
| `channels.yuanbao.overflowPolicy`          | Gestión de mensajes largos (`split` o `stop`)         | `split`                                |
| `channels.yuanbao.replyToMode`             | Estrategia de respuesta a mensajes en grupos (`off`, `first`, `all`)   | `first`                                |
| `channels.yuanbao.outboundQueueStrategy`   | Estrategia saliente (`merge-text` o `immediate`)   | `merge-text`                           |
| `channels.yuanbao.minChars`                | Fusión de texto: caracteres mínimos para activar el envío             | `2800`                                 |
| `channels.yuanbao.maxChars`                | Fusión de texto: caracteres máximos por mensaje                 | `3000`                                 |
| `channels.yuanbao.idleMs`                  | Fusión de texto: tiempo de espera de inactividad antes del vaciado automático (ms)   | `5000`                                 |
| `channels.yuanbao.mediaMaxMb`              | Límite de tamaño del contenido multimedia (MB)                             | `20`                                   |
| `channels.yuanbao.historyLimit`            | Entradas del contexto del historial de chats de grupo                | `100`                                  |
| `channels.yuanbao.disableBlockStreaming`   | Desactivar la salida en streaming a nivel de bloque              | `false`                                |
| `channels.yuanbao.fallbackReply`           | Respuesta de reserva cuando el modelo no devuelve contenido  | `暂时无法解答，你可以换个问题问问我哦` |
| `channels.yuanbao.markdownHintEnabled`     | Inyectar instrucciones para evitar el envoltorio en Markdown        | `true`                                 |
| `channels.yuanbao.debugBotIds`             | ID de bot de la lista de permitidos de depuración (registros sin sanear)        | `[]`                                   |

## Tipos de mensajes compatibles

**Recepción:** texto, imágenes, archivos, audio/voz, vídeo, stickers/emojis personalizados y elementos personalizados (tarjetas de enlaces).

**Envío:** texto (Markdown), imágenes, archivos, audio, vídeo y stickers.

**Hilos y respuestas:** respuestas con cita (configurables mediante `replyToMode`); la plataforma no admite respuestas en hilos.

## Relacionado

- [Descripción general de los canales](/es/channels) - todos los canales compatibles
- [Vinculación](/es/channels/pairing) - autenticación por mensaje directo y flujo de vinculación
- [Grupos](/es/channels/groups) - comportamiento del chat grupal y control mediante menciones
- [Enrutamiento de canales](/es/channels/channel-routing) - enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) - modelo de acceso y refuerzo de la seguridad
