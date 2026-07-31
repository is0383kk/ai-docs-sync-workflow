---
read_when:
    - Configuración de la compatibilidad con Signal
    - Depuración del envío y la recepción de Signal
summary: Compatibilidad con Signal mediante signal-cli (daemon nativo o contenedor de bbernhard), rutas de configuración y modelo de números
title: Signal
x-i18n:
    generated_at: "2026-07-26T04:31:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal es un plugin de canal descargable (`@openclaw/signal`). El gateway se comunica con `signal-cli` mediante HTTP: ya sea el daemon nativo (JSON-RPC + SSE) o el contenedor [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) (REST + WebSocket). OpenClaw no incorpora libsignal.

## El modelo de números (leer primero)

- El gateway se conecta a un **dispositivo Signal**: la cuenta `signal-cli`.
- Ejecutar el bot en **su cuenta personal de Signal** hace que ignore sus propios mensajes (protección contra bucles).
- Para «envío un mensaje al bot y este responde», use un **número de bot independiente**.

## Instalación

```bash
openclaw plugins install @openclaw/signal
```

Las especificaciones de plugins sin calificar prueban primero ClawHub y, después, recurren a npm. Fuerce un origen con `openclaw plugins install clawhub:@openclaw/signal` o `npm:@openclaw/signal`. `plugins install` registra y habilita el plugin; no se necesita un paso independiente de `enable`. Consulte [Plugins](/es/tools/plugin) para conocer las reglas generales de instalación.

## Configuración rápida

<Steps>
  <Step title="Elegir un número">
    Use un **número de Signal independiente** para el bot (recomendado).
  </Step>
  <Step title="Instalar el plugin">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="Ejecutar la configuración guiada">
    ```bash
    openclaw channels add
    ```
    El asistente detecta si `signal-cli` está en `PATH` y, si falta, ofrece instalarlo: descarga la compilación nativa oficial de GraalVM en Linux x86-64 o lo instala mediante Homebrew en macOS y otras arquitecturas. Después, solicita el número del bot y la ruta de `signal-cli`.

    Para una configuración no interactiva, `openclaw channels add --channel signal` también acepta `--signal-number <e164>` para el número de teléfono del bot, además de `--http-host <host>` y `--http-port <port>` para el endpoint del daemon de Signal (valor predeterminado: `127.0.0.1:8080`).

  </Step>
  <Step title="Vincular o registrar la cuenta">
    - **Vinculación mediante QR (la más rápida):** `signal-cli link -n "OpenClaw"` y, después, escanee el código con Signal. Consulte la [ruta A](#setup-path-a-link-existing-signal-account-qr).
    - **Registro mediante SMS:** número dedicado con captcha + verificación por SMS. Consulte la [ruta B](#setup-path-b-register-dedicated-bot-number-sms-linux).

  </Step>
  <Step title="Verificar y emparejar">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    Envíe un primer MD y apruebe el emparejamiento: `openclaw pairing approve signal <CODE>`.
  </Step>
</Steps>

Configuración mínima:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| Campo       | Descripción                                       |
| ----------- | ------------------------------------------------- |
| `account`   | Número de teléfono del bot en formato E.164 (`+15551234567`) |
| `transport` | Conexión de Signal propiedad de la cuenta y modo de proceso  |
| `dmPolicy`  | Política de acceso a MD (se recomienda `pairing`)          |
| `allowFrom` | Números de teléfono o valores `uuid:<id>` autorizados a enviar MD |

Compatibilidad con varias cuentas: use `channels.signal.accounts` con configuración por cuenta y `name` opcional. Cada cuenta con nombre posee su propio `transport`; no hereda el transporte de nivel superior. El transporte de nivel superior pertenece únicamente a la cuenta `default` implícita. Consulte [Canales con varias cuentas](/es/gateway/config-channels#multi-account-all-channels) para conocer el patrón compartido.

## Qué es

- Enrutamiento determinista: las respuestas siempre regresan a Signal.
- Los MD comparten la sesión principal del agente; los grupos están aislados (`agent:<agentId>:signal:group:<groupId>`).
- De forma predeterminada, Signal puede escribir actualizaciones de configuración activadas por `/config set|unset` (requiere `commands.config: true`). Deshabilítelo con `channels.signal.configWrites: false`.

## Ruta de configuración A: vincular una cuenta de Signal existente (QR)

1. Instale `signal-cli` (JVM o compilación nativa), o permita que `openclaw channels add` lo instale.
2. Vincule una cuenta de bot: `signal-cli link -n "OpenClaw"` y, después, escanee el código QR en Signal.
3. Configure Signal e inicie el gateway.

## Ruta de configuración B: registrar un número de bot dedicado (SMS, Linux)

Use esta opción para un número de bot dedicado en lugar de vincular la cuenta de una aplicación Signal existente. El flujo siguiente se ha probado en Ubuntu 24.

1. Obtenga un número que pueda recibir SMS (o una verificación por voz en el caso de líneas fijas). Un número de bot dedicado evita conflictos de cuentas o sesiones.
2. Instale `signal-cli` en el host del gateway:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

Si usa la compilación JVM (`signal-cli-${VERSION}.tar.gz`), instale primero un JRE. Mantenga `signal-cli` actualizado; el proyecto original advierte que las versiones antiguas pueden dejar de funcionar a medida que cambian las API del servidor de Signal.

3. Registre y verifique el número:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

Si se requiere un captcha (se necesita acceso a un navegador para completar este paso):

1. Abra `https://signalcaptchas.org/registration/generate.html`.
2. Complete el captcha y copie el destino del enlace `signalcaptcha://...` de "Open Signal".
3. Cuando sea posible, ejecute el comando desde la misma IP externa que la sesión del navegador (los tokens de captcha caducan rápidamente).
4. Registre y verifique inmediatamente:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. Configure OpenClaw, reinicie el gateway y verifique el canal:

```bash
# Si ejecuta el gateway como un servicio systemd de usuario:
systemctl --user restart openclaw-gateway.service

# Después, verifique:
openclaw doctor
openclaw channels status --probe
```

5. Empareje el remitente de MD:
   - Envíe cualquier mensaje al número del bot.
   - Apruébelo en el servidor: `openclaw pairing approve signal <PAIRING_CODE>`.
   - Guarde el número del bot como contacto en su teléfono para evitar "Unknown contact".

<Warning>
Registrar una cuenta de número de teléfono con `signal-cli` puede desautenticar la sesión de la aplicación principal de Signal correspondiente a ese número. Es preferible usar un número de bot dedicado o el modo de vinculación mediante QR para conservar la configuración actual de la aplicación del teléfono.
</Warning>

Referencias del proyecto original:

- README de `signal-cli`: `https://github.com/AsamK/signal-cli`
- Flujo de captcha: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- Flujo de vinculación: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## Modo de daemon nativo externo

Para gestionar `signal-cli` de forma independiente (inicios en frío lentos de JVM, inicialización del contenedor, CPU compartidas), ejecute el daemon por separado y dirija OpenClaw hacia él:

Para una configuración no interactiva, seleccione explícitamente el tipo de endpoint cuando sea necesario:

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

Esto omite el inicio automático y la espera de arranque de OpenClaw. Para un daemon gestionado con un inicio lento, establezca `channels.signal.transport.startupTimeoutMs`.

## Modo de contenedor (bbernhard/signal-cli-rest-api)

En lugar de ejecutar `signal-cli` de forma nativa, use el contenedor Docker [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api), que encapsula `signal-cli` tras una interfaz REST + WebSocket.

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

Requisitos:

- El contenedor **debe** ejecutarse con `MODE=json-rpc` para recibir mensajes en tiempo real.
- Registre o vincule su cuenta de Signal dentro del contenedor antes de conectar OpenClaw.

Ejemplo de servicio `docker-compose.yml`:

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

Configuración de OpenClaw:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind` controla qué protocolo y ciclo de vida del proceso utiliza OpenClaw:

| Valor               | Comportamiento                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | Inicia signal-cli nativo y usa JSON-RPC en `/api/v1/rpc` junto con SSE en `/api/v1/events`; `url` puede seleccionar un endpoint de conexión distinto de la dirección de enlace del daemon |
| `"external-native"` | Se conecta a un daemon nativo signal-cli que ya está en ejecución                                                                                                       |
| `"container"`       | Se conecta al REST de bbernhard en `/v2/send` y al WebSocket en `/v1/receive/{account}`                                                                             |

La configuración y `openclaw doctor --fix` pueden sondear una vez un endpoint existente para identificar su tipo concreto. Las operaciones en tiempo de ejecución no detectan ni cambian los protocolos automáticamente.

El modo de contenedor admite las mismas operaciones de Signal que el modo nativo cuando el contenedor expone API equivalentes: envíos, recepciones, archivos adjuntos, indicadores de escritura, confirmaciones de lectura y visualización, reacciones, grupos y texto con estilo. OpenClaw traduce las llamadas RPC nativas de Signal a las cargas útiles REST del contenedor, incluidos los identificadores de grupo `group.{base64(internal_id)}` y `text_mode: "styled"` para texto con formato.

Notas operativas:

- Use `MODE=json-rpc` para recibir. `MODE=normal` puede hacer que `/v1/about` parezca funcionar correctamente, pero `/v1/receive/{account}` no realizará la actualización a WebSocket, por lo que la transmisión de recepción del contenedor no superará el sondeo.
- Establezca `kind: "container"` para la API REST de bbernhard y `kind: "external-native"` para JSON-RPC/SSE de `signal-cli` nativo.
- Las descargas de archivos adjuntos del contenedor respetan los mismos límites de bytes multimedia que el modo nativo. Las respuestas demasiado grandes se rechazan antes de almacenarse por completo en el búfer cuando el servidor envía `Content-Length`, y durante la transmisión en caso contrario.

## Control de acceso (MD + grupos)

MD:

- Valor predeterminado: `channels.signal.dmPolicy = "pairing"`.
- Los remitentes desconocidos reciben un código de emparejamiento; los mensajes se ignoran hasta su aprobación (los códigos caducan después de 1 hora).
- Apruebe mediante `openclaw pairing list signal` y `openclaw pairing approve signal <CODE>`.
- El emparejamiento es el intercambio de tokens predeterminado para los MD de Signal. Detalles: [Emparejamiento](/es/channels/pairing)
- Los remitentes que solo tienen UUID (de `sourceUuid`) se almacenan como `uuid:<id>` en `channels.signal.allowFrom`.

Grupos:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- `channels.signal.groupAllowFrom` controla qué grupos o remitentes pueden activar respuestas de grupo cuando se establece `allowlist`; las entradas pueden ser identificadores de grupo de Signal (sin formato, `group:<id>` o `signal:group:<id>`), números de teléfono de remitentes, valores de `uuid:<id>` o `*`.
- `channels.signal.groups["<group-id>" | "*"]` puede anular el comportamiento de los grupos mediante `requireMention`, `tools` y `toolsBySender`.
- Use `channels.signal.accounts.<id>.groups` para aplicar anulaciones por cuenta en configuraciones con varias cuentas.
- Añadir un grupo de Signal a la lista de permitidos mediante `groupAllowFrom` no desactiva por sí solo el requisito de mención. Una entrada `channels.signal.groups["<group-id>"]` configurada específicamente procesa todos los mensajes del grupo, salvo que se establezca `requireMention=true`.
- Con `requireMention=true`, las @menciones nativas de Signal se comparan, mediante los metadatos estructurados de las menciones, con el teléfono o el `accountUuid` de la cuenta del bot. Los `mentionPatterns` configurados siguen sirviendo como alternativa de texto sin formato.
- Nota sobre el entorno de ejecución: si `channels.signal` falta por completo, el entorno de ejecución recurre a `groupPolicy="allowlist"` para las comprobaciones de grupos (aunque se haya establecido `channels.defaults.groupPolicy`).

Grupo sujeto a mención con contexto limitado:

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

Los mensajes permitidos del grupo que no mencionan al bot permanecen sin respuesta y se conservan únicamente en la ventana limitada del historial pendiente. Cuando una @mención nativa posterior o una mención de texto alternativa activa el bot, OpenClaw incluye ese contexto reciente y responde al mismo grupo. Los cuerpos de los archivos adjuntos omitidos no se descargan; solo pueden aparecer como marcadores compactos de contenido multimedia en el contexto pendiente.

## Cómo funciona (comportamiento)

- Modo nativo: `signal-cli` se ejecuta como demonio; el Gateway lee los eventos mediante SSE.
- Modo de contenedor: el Gateway envía mediante la API REST y recibe mediante WebSocket.
- Los mensajes entrantes se normalizan en el sobre de canal compartido.
- Las respuestas siempre se dirigen de nuevo al mismo número o grupo.
- Las respuestas a mensajes entrantes incluyen metadatos de cita nativos de Signal cuando el backend acepta la marca de tiempo y el autor del mensaje entrante; si faltan los metadatos de cita o se rechazan, OpenClaw envía la respuesta como un mensaje normal.
- Configure el uso de citas nativas mediante `channels.signal.replyToMode = off | first | all | batched`, o `channels.signal.replyToModeByChatType.direct/group` para aplicar anulaciones por tipo de chat. Los valores del nivel de cuenta definidos en `channels.signal.accounts.<id>` tienen prioridad.

## Contenido multimedia y límites

- El texto saliente se divide en fragmentos según `channels.signal.textChunkLimit` (valor predeterminado: 4000).
- División opcional por saltos de línea: establezca `channels.signal.streaming.chunkMode="newline"` para dividir por líneas en blanco (límites de párrafo) antes de dividir por longitud.
- Se admiten archivos adjuntos (codificados en base64 y obtenidos desde `signal-cli`).
- Los archivos adjuntos de notas de voz utilizan el nombre de archivo `signal-cli` como alternativa para el tipo MIME cuando falta `contentType`, de modo que la transcripción de audio pueda seguir clasificando las notas de voz AAC.
- Límite predeterminado de contenido multimedia: `channels.signal.mediaMaxMb` (valor predeterminado: 8).
- Use `channels.signal.ignoreAttachments` para omitir la descarga de contenido multimedia en cualquier transporte.
- El contexto del historial de grupos utiliza `channels.signal.historyLimit` (o `channels.signal.accounts.*.historyLimit`) y, como alternativa, `messages.groupChat.historyLimit`. Establezca `0` para desactivarlo (valor predeterminado: 50).

## Indicadores de escritura y confirmaciones de lectura

- **Indicadores de escritura**: OpenClaw envía señales de escritura mediante `signal-cli sendTyping` y las actualiza mientras se genera una respuesta.
- **Confirmaciones de lectura**: cuando `channels.signal.sendReadReceipts` es verdadero, OpenClaw reenvía confirmaciones de lectura para los mensajes directos permitidos.
- `signal-cli` no expone confirmaciones de lectura para los grupos.

## Reacciones de estado del ciclo de vida

Establezca `messages.statusReactions.enabled: true` para permitir que Signal muestre el ciclo de vida compartido de reacciones de en cola, razonamiento, herramienta, Compaction, finalización y error en los turnos entrantes. Signal utiliza la marca de tiempo del mensaje entrante como objetivo de la reacción; las reacciones de grupo se envían con el identificador del grupo de Signal y el remitente original como autor objetivo.

Las reacciones de estado también requieren una reacción de confirmación y un `messages.ackReactionScope` coincidente (`direct`, `group-all`, `group-mentions` o `all`). Establezca `channels.signal.reactionLevel: "off"` para desactivar las reacciones de estado de Signal.

Signal restaura la reacción de confirmación inicial después del estado final de finalización o error.

## Reacciones (herramienta de mensajes)

Use `message action=react` con `channel=signal`.

- Objetivos: número E.164 o UUID del remitente (use `uuid:<id>` de la salida del emparejamiento; también funciona un UUID sin prefijo).
- `messageId` es la marca de tiempo de Signal del mensaje al que se reacciona.
- Las reacciones de grupo requieren `targetAuthor` o `targetAuthorUuid`.

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

Configuración:

- `channels.signal.actions.reactions`: activa o desactiva las acciones de reacción (valor predeterminado: verdadero).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (valor predeterminado: `minimal`).
  - `off`/`ack` desactiva las reacciones del agente (la herramienta de mensajes `react` genera errores).
  - `minimal`/`extensive` activa las reacciones del agente y establece el nivel de orientación.
- Anulaciones por cuenta: `channels.signal.accounts.<id>.actions.reactions`, `channels.signal.accounts.<id>.reactionLevel`.

## Reacciones de aprobación

Las solicitudes de aprobación de ejecución y de plugins de Signal utilizan los bloques de enrutamiento de nivel superior `approvals.exec` y `approvals.plugin`. Signal no tiene ningún bloque `channels.signal.execApprovals`.

- `👍` concede una aprobación única.
- `👎` rechaza.
- Use `/approve <id> allow-always` cuando una solicitud ofrezca aprobación persistente.

La resolución de reacciones de aprobación requiere aprobadores explícitos de Signal procedentes de `channels.signal.allowFrom`, `channels.signal.defaultTo` o los campos correspondientes del nivel de cuenta. Las solicitudes directas de aprobación de ejecución en el mismo chat aún pueden ocultar la alternativa local duplicada `/approve` sin aprobadores explícitos; en las aprobaciones de grupo sin aprobadores, la alternativa local permanece visible.

## Reacciones a preguntas

Para una solicitud `ask_user` con una pregunta no secreta de selección única y entre una y cuatro opciones, Signal muestra desde `1️⃣` hasta `4️⃣` junto a las etiquetas de las opciones. Para responder, reaccione a la solicitud entregada con el número correspondiente. OpenClaw comprueba que la reacción tenga como objetivo el mensaje creado por el bot y, a continuación, asigna el número a la opción canónica mediante el Gateway. Los toques obsoletos o duplicados se ignoran. Las solicitudes con varias preguntas, selección múltiple o texto libre siguen admitiendo únicamente respuestas de texto; las reglas normales de admisión de mensajes directos y grupos de Signal autorizan al remitente.

## Destinos de entrega (CLI/Cron)

- Mensajes directos: `signal:+15551234567` (o E.164 sin prefijo).
- Mensajes directos por UUID: `uuid:<id>` (o UUID sin prefijo).
- Grupos: `signal:group:<groupId>`.
- Nombres de usuario: `username:<name>` (si la cuenta de Signal los admite).

## Alias

Configure alias para usar nombres estables con destinos recurrentes de Signal. Los alias solo forman parte de la configuración de OpenClaw; no crean ni editan contactos de Signal.

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

Use alias en cualquier lugar donde se acepten destinos de entrega de Signal:

```bash
openclaw message send --channel signal --target signal:ops --message "Deployment is complete"
```

Los alias por cuenta heredan los alias de nivel superior y pueden añadir o anular nombres:

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` y `openclaw directory groups list --channel signal` muestran los alias configurados. El directorio de Signal se basa en la configuración; no consulta en tiempo real los contactos de Signal ni modifica la cuenta de Signal.

## Solución de problemas

Ejecute primero esta secuencia:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Después, confirme el estado de emparejamiento de los mensajes directos si es necesario:

```bash
openclaw pairing list signal
```

Errores habituales:

- El demonio está accesible, pero no hay respuestas: compruebe `account`, `transport.kind`, la URL de transporte y el modo de recepción.
- Se ignoran los mensajes directos: el remitente está pendiente de aprobación de emparejamiento.
- Se ignoran los mensajes de grupo: los requisitos de remitente o mención del grupo bloquean la entrega.
- Errores de validación de la configuración después de editarla: ejecute `openclaw doctor --fix`.
- Signal no aparece en los diagnósticos: confirme `channels.signal.enabled: true`.

Comprobaciones adicionales:

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

Para consultar el flujo de triaje: [Solución de problemas de canales](/es/channels/troubleshooting).

## Notas de seguridad

- `signal-cli` almacena localmente las claves de la cuenta (normalmente en `~/.local/share/signal-cli/data/`).
- Realice una copia de seguridad del estado de la cuenta de Signal antes de migrar o reconstruir el servidor.
- Mantenga `channels.signal.dmPolicy: "pairing"`, salvo que se desee explícitamente un acceso más amplio a los mensajes directos.
- La verificación por SMS solo es necesaria para los flujos de registro o recuperación, pero perder el control del número o de la cuenta puede complicar un nuevo registro.

## Referencia de configuración (Signal)

Configuración completa: [Configuración](/es/gateway/configuration)

Opciones del proveedor:

- `channels.signal.enabled`: habilita/deshabilita el inicio del canal.
- `channels.signal.account`: E.164 para la cuenta del bot.
- `channels.signal.accountUuid`: UUID opcional de la cuenta del bot para la detección nativa de @menciones y la protección contra bucles.
- `channels.signal.transport`: transporte propiedad de la cuenta. Omítalo para usar los valores predeterminados nativos gestionados.
- `channels.signal.transport.kind`: `managed-native | external-native | container`.
- `channels.signal.transport.url`: obligatorio para `external-native` y `container`; opcional para `managed-native` cuando su endpoint de conexión difiere de la vinculación del daemon.
- `channels.signal.transport.cliPath`: ruta nativa gestionada a `signal-cli`.
- `channels.signal.transport.configPath`: directorio `signal-cli --config` nativo gestionado opcional.
- `channels.signal.transport.httpHost`, `channels.signal.transport.httpPort`: vinculación del daemon nativo gestionado (valor predeterminado: `127.0.0.1:8080`).
- `channels.signal.transport.startupTimeoutMs`: espera de inicio del componente nativo gestionado en ms (mín. 1000, máx. 120000; valor predeterminado: 30000).
- `channels.signal.transport.receiveMode`: `on-start | manual` nativo gestionado.
- `channels.signal.ignoreAttachments`: omite las descargas de archivos adjuntos entrantes para esta cuenta.
- `channels.signal.transport.ignoreStories`: control de historias nativo gestionado.
- `channels.signal.sendReadReceipts`: reenvía las confirmaciones de lectura.
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled` (valor predeterminado: emparejamiento).
- `channels.signal.allowFrom`: lista de permitidos para mensajes directos (E.164 o `uuid:<id>`). `open` requiere `"*"`. Signal no tiene nombres de usuario; use identificadores de teléfono/UUID.
- `channels.signal.aliases`: alias del lado de OpenClaw para destinos de entrega de mensajes directos o grupos.
- `channels.signal.groupPolicy`: `open | allowlist | disabled` (valor predeterminado: lista de permitidos).
- `channels.signal.groupAllowFrom`: lista de permitidos para grupos; acepta identificadores de grupos de Signal (sin procesar, `group:<id>` o `signal:group:<id>`), números E.164 de remitentes o valores `uuid:<id>`.
- `channels.signal.groups`: anulaciones por grupo indexadas mediante el identificador de grupo de Signal (o `"*"`). Campos admitidos: `requireMention`, `tools`, `toolsBySender`.
- `channels.signal.accounts.<id>.groups`: versión por cuenta de `channels.signal.groups` para configuraciones con varias cuentas.
- `channels.signal.accounts.<id>.aliases`: alias por cuenta, combinados con los alias de nivel superior.
- `channels.signal.replyToMode`: modo nativo de cita de respuesta, `off | first | all | batched` (valor predeterminado: `all`).
- `channels.signal.replyToModeByChatType.direct`, `channels.signal.replyToModeByChatType.group`: anulaciones nativas de citas de respuesta por tipo de chat.
- `channels.signal.accounts.<id>.replyToMode`, `channels.signal.accounts.<id>.replyToModeByChatType.direct`, `channels.signal.accounts.<id>.replyToModeByChatType.group`: anulaciones de citas de respuesta por cuenta.
- `channels.signal.historyLimit`: número máximo de mensajes de grupo que se incluirán como contexto (0 lo deshabilita).
- `channels.signal.dmHistoryLimit`: límite del historial de mensajes directos en turnos del usuario. Anulaciones por usuario: `channels.signal.dms["<phone_or_uuid>"].historyLimit`.
- `channels.signal.textChunkLimit`: tamaño de los fragmentos salientes en caracteres (valor predeterminado: 4000).
- `channels.signal.streaming.chunkMode`: `length` (valor predeterminado) o `newline` para dividir por líneas en blanco (límites de párrafo) antes de fragmentar por longitud.
- `channels.signal.mediaMaxMb`: límite de contenido multimedia entrante/saliente en MB (valor predeterminado: 8).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (valor predeterminado: `minimal`). Consulte [Reacciones](#reactions-message-tool).
- `channels.signal.reactionNotifications`: `off | own | all | allowlist` (valor predeterminado: `own`) - cuándo se notifica al agente sobre reacciones entrantes de otras personas.
- `channels.signal.reactionAllowlist`: remitentes cuyas reacciones notifican al agente cuando `reactionNotifications: "allowlist"`.
- `channels.signal.streaming.block.enabled`, `channels.signal.streaming.block.coalesce`: controles de streaming en modo de bloques compartidos entre canales. Consulte [Streaming](/es/concepts/streaming).

Opciones globales relacionadas:

- `agents.entries.*.groupChat.mentionPatterns` (alternativa de texto sin formato; las @menciones nativas de Signal se detectan a partir de metadatos estructurados cuando se configura la identidad de la cuenta del bot).
- `messages.groupChat.mentionPatterns` (alternativa global).
- `channels.signal.responsePrefix` o un `responsePrefix` en el nivel de la cuenta.

## Temas relacionados

- [Descripción general de los canales](/es/channels) - todos los canales compatibles
- [Emparejamiento](/es/channels/pairing) - autenticación de mensajes directos y flujo de emparejamiento
- [Grupos](/es/channels/groups) - comportamiento de los chats grupales y control mediante menciones
- [Enrutamiento de canales](/es/channels/channel-routing) - enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) - modelo de acceso y refuerzo de seguridad
