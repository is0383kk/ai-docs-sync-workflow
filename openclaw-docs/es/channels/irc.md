---
read_when:
    - Quieres conectar OpenClaw a canales de IRC o mensajes directos
    - Estás configurando listas de permitidos de IRC, políticas de grupo o restricciones de menciones.
summary: Configuración del plugin de IRC, controles de acceso y solución de problemas
title: IRC
x-i18n:
    generated_at: "2026-07-26T05:06:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

Usa IRC cuando quieras utilizar OpenClaw en canales clásicos (`#room`) y mensajes directos.
Instala el plugin oficial de IRC y configúralo en `channels.irc`.

## Inicio rápido

1. Instala el plugin:

```bash
openclaw plugins install @openclaw/irc
```

2. Configura al menos el host, el apodo y los canales a los que se debe unir en `~/.openclaw/openclaw.json`:

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. Inicia o reinicia el Gateway:

```bash
openclaw gateway run
```

Es preferible usar un servidor IRC privado para la coordinación de bots. Si se utiliza deliberadamente una red IRC pública, algunas opciones habituales son Libera.Chat, OFTC y Snoonet. Evita canales públicos predecibles para el tráfico de comunicación interna de bots o enjambres.

## Durabilidad de la entrada

OpenClaw escribe cada `PRIVMSG` de IRC aceptado en su cola de entrada duradera antes de las comprobaciones normales de políticas y del envío al agente. Los mensajes pendientes o reintentables sobreviven a un reinicio del Gateway y permanecen serializados por canal o interlocutor de mensajes directos.

IRC no proporciona un ID de entrega reproducible ni reenvía los mensajes perdidos por un cliente desconectado. Por ello, OpenClaw asigna un ID local que solo es estable dentro de la conexión TCP actual. La cola protege el intervalo local entre la aceptación y el envío; no puede recuperar un mensaje que nunca llegó a OpenClaw ni deduplicar un reenvío del servidor entre conexiones.

## Ajustes de conexión

| Clave                         | Valor predeterminado          | Notas                                                       |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | ninguno (obligatorio)         | Nombre de host del servidor IRC                             |
| `port`                        | `6697` con TLS, `6667` sin cifrar | 1-65535                                                     |
| `tls`                         | `true`                        | Establece `false` solo para usar texto sin cifrar deliberadamente |
| `nick`                        | ninguno (obligatorio)         | Apodo del bot                                               |
| `username`                    | apodo; de lo contrario, `openclaw` | Nombre de usuario de IRC                                    |
| `realname`                    | `OpenClaw`                    | Campo de nombre real/GECOS                                  |
| `password` / `passwordFile`   | ninguno                       | Contraseña del servidor; el archivo debe ser un archivo normal |
| `channels`                    | ninguno                       | Canales a los que unirse (`["#openclaw"]`)               |
| `accounts` / `defaultAccount` | ninguno                       | Configuración multicuenta; las variables de entorno solo completan la cuenta predeterminada |

## Valores de seguridad predeterminados

- IRC utiliza sockets TCP/TLS sin procesar fuera del enrutamiento mediante el proxy de reenvío administrado por el operador de OpenClaw. En implementaciones que requieran que todo el tráfico saliente pase por ese proxy de reenvío, establece `channels.irc.enabled=false`, salvo que se haya aprobado explícitamente el tráfico IRC saliente directo.
- `channels.irc.dmPolicy` tiene como valor predeterminado `"pairing"`: los remitentes desconocidos de mensajes directos reciben un código de emparejamiento que se aprueba con `openclaw pairing approve irc <code>`.
- `channels.irc.groupPolicy` tiene como valor predeterminado `"allowlist"`.
- Con `groupPolicy="allowlist"`, establece `channels.irc.groups` para definir los canales permitidos.
- Usa TLS (`channels.irc.tls=true`), salvo que se acepte deliberadamente el transporte de texto sin cifrar.

## Control de acceso

Existen dos «barreras» independientes para los canales IRC:

1. **Acceso al canal** (`groupPolicy` + `groups`): determina si el bot acepta mensajes de un canal.
2. **Acceso del remitente** (`groupAllowFrom` / `groups["#channel"].allowFrom` por canal): determina quién puede activar el bot dentro de ese canal.

Claves de configuración:

- Lista de permitidos de mensajes directos (acceso del remitente de mensajes directos): `channels.irc.allowFrom`
- Lista de permitidos de remitentes de grupos (acceso de remitentes del canal): `channels.irc.groupAllowFrom`
- Controles por canal (reglas de canal, remitente y mención): `channels.irc.groups["#channel"]` con `requireMention`, `allowFrom`, `enabled`, `tools`, `toolsBySender`, `skills` y `systemPrompt`
- `channels.irc.groupPolicy="open"` permite canales sin configurar (**siguen requiriendo una mención de forma predeterminada**)

Las entradas de la lista de permitidos deben usar identidades estables de remitentes (`nick!user@host`).
La coincidencia solo por apodo es mutable y únicamente se habilita cuando `channels.irc.dangerouslyAllowNameMatching: true`.

### Error habitual: `allowFrom` es para mensajes directos, no para canales

Si aparecen registros como:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

...significa que el remitente no tenía permiso para enviar mensajes de **grupo/canal**. Se puede corregir de cualquiera de estas formas:

- estableciendo `channels.irc.groupAllowFrom` (global para todos los canales), o
- estableciendo listas de permitidos de remitentes por canal: `channels.irc.groups["#channel"].allowFrom`

Ejemplo (permitir que cualquier persona de `#openclaw` hable con el bot):

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## Activación de respuestas (menciones)

Aunque un canal esté permitido (mediante `groupPolicy` + `groups`) y el remitente tenga permiso, OpenClaw exige **menciones** de forma predeterminada en contextos de grupo. Se considera que se ha mencionado al bot cuando el mensaje contiene el apodo del bot conectado o coincide con los patrones de mención configurados.

Esto significa que pueden aparecer registros como `drop channel … (missing-mention)`, salvo que el mensaje incluya un patrón de mención que coincida con el bot.

Para hacer que el bot responda en un canal IRC **sin necesidad de una mención**, desactiva el requisito de mención para ese canal:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

También se pueden permitir **todos** los canales IRC (sin una lista de permitidos por canal) y seguir respondiendo sin menciones:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## Nota de seguridad (recomendado para canales públicos)

Si se permite `allowFrom: ["*"]` en un canal público, cualquiera puede enviar instrucciones al bot.
Para reducir el riesgo, restringe las herramientas de ese canal.

### Las mismas herramientas para todos los participantes del canal

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### Herramientas diferentes por remitente (el propietario obtiene más privilegios)

Usa `toolsBySender` para aplicar una política más estricta a `"*"` y una más permisiva al apodo del propietario:

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

Notas:

- Las claves de `toolsBySender` deben usar prefijos explícitos (`channel:`, `id:`, `e164:`, `username:`, `name:`). Para IRC, usa `id:` con el valor de identidad del remitente: `id:alice` o `id:alice!~alice@203.0.113.7` para obtener una coincidencia más sólida.
- Las claves heredadas sin prefijo todavía se aceptan, solo se comparan como `id:` y generan una advertencia de obsolescencia.
- Se aplica la primera política de remitente que coincida; `"*"` es la alternativa comodín.

Para obtener más información sobre el acceso de grupos frente al requisito de mención (y cómo interactúan), consulta: [/channels/groups](/es/channels/groups).

## NickServ

Para identificarse con NickServ después de conectarse:

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

La identificación con NickServ se ejecuta de forma predeterminada siempre que haya una contraseña configurada (`enabled` solo debe ser `false` para inhabilitarla). El valor predeterminado de `service` es `NickServ`; `passwordFile` es una alternativa a `password` insertado directamente.

Registro único opcional al conectarse (`register: true` requiere `registerEmail`):

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

Deshabilita `register` después de registrar el apodo para evitar intentos repetidos de REGISTER.

## Variables de entorno

La cuenta predeterminada admite:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS` (separados por comas)
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

`IRC_HOST` no se puede establecer desde un `.env` del espacio de trabajo; consulta [Archivos `.env` del espacio de trabajo](/es/gateway/security).

## Solución de problemas

- Si el bot se conecta, pero nunca responde en los canales, verifica `channels.irc.groups` **y** si el requisito de mención está descartando mensajes (`missing-mention`). Para que responda sin menciones, establece `requireMention:false` para el canal.
- Si falla el inicio de sesión, verifica la disponibilidad del apodo y la contraseña del servidor.
- Si TLS falla en una red personalizada, verifica el host, el puerto y la configuración del certificado.

## Contenido relacionado

- [Descripción general de los canales](/es/channels) — todos los canales compatibles
- [Emparejamiento](/es/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/es/channels/groups) — comportamiento de los chats grupales y requisito de mención
- [Enrutamiento de canales](/es/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) — modelo de acceso y protección
