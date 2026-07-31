---
read_when:
    - Trabajo en el comportamiento del canal de WhatsApp/web o en el enrutamiento de la bandeja de entrada
summary: Compatibilidad con el canal de WhatsApp, controles de acceso, comportamiento de entrega y operaciones
title: WhatsApp
x-i18n:
    generated_at: "2026-07-26T04:32:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7489b37f91775868d0694daea8a0958ee000d1411674d1800bb1e77df5961e68
    source_path: channels/whatsapp.md
    workflow: 16
---

Estado: listo para producción mediante WhatsApp Web (Baileys). El Gateway es responsable de las sesiones vinculadas; no hay ningún canal de WhatsApp de Twilio independiente.

## Instalación

`openclaw onboard` y `openclaw channels add --channel whatsapp` solicitan instalar el plugin la primera vez que se selecciona; `openclaw channels login --channel whatsapp` ofrece el mismo flujo de instalación si falta el plugin. Los checkouts de desarrollo utilizan la ruta local del plugin; las instalaciones estables/beta instalan primero `@openclaw/whatsapp` desde ClawHub y recurren a npm si falla. El entorno de ejecución de WhatsApp se distribuye fuera del paquete npm principal de OpenClaw, por lo que sus dependencias de ejecución permanecen en el plugin externo. Instalación manual:

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

Utilice el paquete npm sin prefijo (`@openclaw/whatsapp`) únicamente como alternativa del registro; fije una versión exacta solo para obtener una instalación reproducible.

<CardGroup cols={3}>
  <Card title="Vinculación" icon="link" href="/es/channels/pairing">
    La política predeterminada de mensajes directos para remitentes desconocidos es la vinculación.
  </Card>
  <Card title="Solución de problemas de canales" icon="wrench" href="/es/channels/troubleshooting">
    Diagnósticos entre canales y procedimientos de reparación.
  </Card>
  <Card title="Configuración del Gateway" icon="settings" href="/es/gateway/configuration">
    Patrones y ejemplos completos de configuración de canales.
  </Card>
</CardGroup>

## Configuración rápida

<Steps>
  <Step title="Configurar la política de acceso">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="Vincular WhatsApp (QR)">

```bash
openclaw channels login --channel whatsapp
```

    El inicio de sesión solo se realiza mediante QR. En hosts remotos o sin interfaz gráfica, disponga de un medio fiable para entregar el QR activo al teléfono antes de iniciar la sesión; los QR representados en el terminal, las capturas de pantalla o los archivos adjuntos de chat pueden caducar durante el envío.

    Para una cuenta específica:

```bash
openclaw channels login --channel whatsapp --account work
```

    Para adjuntar un directorio de autenticación existente o personalizado antes de iniciar sesión:

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="Iniciar el Gateway">

```bash
openclaw gateway
```

  </Step>

  <Step title="Aprobar la primera solicitud de acceso por mensaje directo (modo de vinculación)">

    Abra **Settings → Channels → DM access requests**, busque la cuenta de WhatsApp
    y apruebe al remitente. Si prefiere la CLI:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    Las solicitudes de acceso por mensaje directo caducan después de 1 hora; las solicitudes pendientes están limitadas a 3 por
    cuenta. Esta aprobación es independiente del QR de inicio de sesión de WhatsApp utilizado para vincular la
    cuenta.

  </Step>
</Steps>

<Note>
Se recomienda utilizar un número de WhatsApp independiente (la configuración y los metadatos están optimizados para ello), pero se admiten por completo las configuraciones con un número personal o chat con uno mismo.
</Note>

## Patrones de despliegue

<AccordionGroup>
  <Accordion title="Número dedicado (recomendado)">
    - identidad de WhatsApp independiente para OpenClaw
    - listas de permitidos de mensajes directos y límites de enrutamiento más claros
    - menor probabilidad de confusión con el chat con uno mismo

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Alternativa con número personal">
    La incorporación admite el modo de número personal y escribe una configuración de referencia adecuada para el chat con uno mismo: `dmPolicy: "allowlist"`, `allowFrom` incluido su propio número, `selfChatMode: true`. Las protecciones del entorno de ejecución para el chat con uno mismo se basan en el número propio vinculado junto con `allowFrom`.
  </Accordion>
</AccordionGroup>

## Modelo de ejecución

- El Gateway es responsable del socket de WhatsApp y del bucle de reconexión.
- Un monitor supervisa dos señales de forma independiente: la actividad de transporte sin procesar de WhatsApp Web y la actividad de mensajes de la aplicación. Una sesión inactiva pero conectada no se reinicia solo porque no se haya recibido ningún mensaje recientemente; solo fuerza la reconexión cuando dejan de llegar tramas de transporte durante un intervalo interno fijo (no configurable por el usuario) o los mensajes de la aplicación permanecen inactivos durante más de 4 veces el tiempo de espera normal de mensajes. Inmediatamente después de una reconexión de una sesión activa recientemente, ese primer intervalo utiliza el tiempo de espera normal de mensajes, más corto, en lugar del intervalo de 4 veces. OpenClaw puede responder automáticamente a los mensajes sin conexión que Baileys entrega al principio de esa reconexión, dentro del límite de la duración de la desduplicación de identificadores de mensajes entrantes; el inicio inicial mantiene la protección breve contra el historial obsoleto.
- Los envíos salientes requieren un listener de WhatsApp activo para la cuenta de destino; de lo contrario, fallan de inmediato.
- Los envíos a grupos adjuntan metadatos nativos de menciones para los tokens `@+<digits>` y `@<digits>` (en el texto y en los pies de contenido multimedia) cuando el token coincide con los metadatos actuales de un participante, incluidos los grupos respaldados por LID.
- Se ignoran los chats de estado y difusión (`@status`, `@broadcast`).
- Los chats directos utilizan las reglas de sesión de mensajes directos (`session.dmScope`; el valor predeterminado `main` agrupa los mensajes directos en la sesión principal del agente). Las sesiones de grupo se aíslan por JID (`agent:<agentId>:whatsapp:group:<jid>`).
- Los Canales/Boletines de WhatsApp pueden ser destinos salientes explícitos mediante su JID `@newsletter` nativo, utilizando metadatos de sesión de canal (`agent:<agentId>:whatsapp:channel:<jid>`) en lugar de la semántica de mensajes directos.
- El transporte de WhatsApp Web respeta las variables de entorno de proxy estándar en el host del Gateway (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, y sus variantes en minúsculas). Se recomienda la configuración del proxy en el host en lugar de ajustes por canal.

## Llamar al solicitante actual con MeowCaller (experimental)

El plugin puede exponer `whatsapp_call` en los turnos del agente originados en WhatsApp. Utiliza [MeowCaller](https://github.com/purpshell/meowcaller) para realizar una llamada de voz de WhatsApp al solicitante autorizado actual y reproducir un mensaje TTS de OpenClaw cuando responda. La herramienta no tiene ningún parámetro de número de destino, por lo que una instrucción no puede redirigir la llamada. Está desactivada de forma predeterminada.

<Warning>
MeowCaller es experimental, no tiene ninguna versión etiquetada y utiliza una sesión de dispositivo vinculado whatsmeow emparejada por separado; no puede reutilizar las credenciales de Baileys del plugin. La vinculación añade otro dispositivo vinculado a la misma cuenta de WhatsApp; escanee el código con la identidad que utiliza OpenClaw. El modo de número personal o chat con uno mismo no puede llamarse a sí mismo; utilice un número dedicado de OpenClaw para llamar a su número personal.
</Warning>

<Steps>
  <Step title="Activar las llamadas experimentales">

    Añada `actions.calls: true` a la configuración del canal de WhatsApp y reinicie el Gateway:

```json
{
  "channels": {
    "whatsapp": {
      "actions": {
        "calls": true
      }
    }
  }
}
```

    Cuando está ausente o es `false`, OpenClaw no expone la herramienta `whatsapp_call`.

  </Step>

  <Step title="Instalar la CLI revisada de MeowCaller">

    El adaptador espera un ejecutable `meowcaller` en el `PATH` del host del Gateway. Hasta que se fusione [el pull request #7 de MeowCaller](https://github.com/purpshell/meowcaller/pull/7), compile la rama revisada:

```bash
git clone --branch feat/send-only-notify https://github.com/steipete/meowcaller.git
cd meowcaller
git checkout 752050471fc2bf7a8cdfbf7dbd3cd4e865d85d3f
mkdir -p "$HOME/.local/bin"
go build -o "$HOME/.local/bin/meowcaller" ./cmd/meowcaller
```

    Asegúrese de que `$HOME/.local/bin` esté en el `PATH` del servicio del Gateway. Esta revisión incluye comandos explícitos `pair` y `notify` de solo envío; `notify` no abre ningún micrófono, altavoz, dispositivo de vídeo ni captura de diagnóstico. No lo sustituya por el comando `play` de la CLI de ejemplo del proyecto original.

  </Step>

  <Step title="Vincular el dispositivo enlazado de MeowCaller">

    Solicite al agente de WhatsApp que compruebe la configuración de las llamadas (la acción de estado `whatsapp_call` informa del directorio de estado específico de la cuenta y del comando de vinculación). Para la cuenta predeterminada:

```bash
state_dir="$HOME/.openclaw/credentials/whatsapp-calls/default"
mkdir -p "$state_dir"
chmod 700 "$state_dir"
meowcaller pair --store "$state_dir/wa-voip.db"
```

    Ejecute este comando de forma interactiva, escanee el QR desde **WhatsApp > Linked devices** y espere a `MeowCaller linked device ready`. Mantenga `wa-voip.db` en privado: es la sesión de MeowCaller. Las cuentas no predeterminadas obtienen su propia ruta de almacenamiento mediante la acción de estado; en Windows, ejecute su comando de PowerShell.

  </Step>

  <Step title="Configurar TTS y llamar desde WhatsApp">

    Configure un [proveedor de TTS](/es/tools/tts) compatible con telefonía, reinicie el Gateway y envíe una solicitud como `Call me and say the build finished.` La herramienta obtiene el remitente a partir del contexto entrante de confianza, sintetiza un archivo WAV privado temporal, ejecuta MeowCaller durante un intervalo de llamada limitado y elimina después el archivo de audio. OpenClaw pasa explícitamente el almacenamiento de la cuenta, espera un estado de salida cero después de responder, reproducir y colgar, y considera que un tiempo de espera agotado o un estado de salida distinto de cero constituyen una llamada fallida de la herramienta.

  </Step>
</Steps>

Límites: solo llamadas de audio salientes individuales, sin números de destino arbitrarios, sin autenticación compartida con la conexión de chat, sin llamadas a uno mismo desde el modo de número personal o chat con uno mismo, audio sintetizado limitado a 60 segundos, sin confirmación de audibilidad en el teléfono más allá de que MeowCaller complete las fases de respuesta, reproducción y finalización, y OpenClaw detiene el proceso complementario después de un intervalo limitado de 115-175 segundos (que abarca las fases de conexión, respuesta, reproducción y cierre de MeowCaller).

## Solicitudes de aprobación

WhatsApp puede representar las solicitudes de aprobación de ejecución y plugins como reacciones `👍`/`👎`, controladas mediante la configuración de reenvío de aprobaciones de nivel superior:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",
    },
    plugin: {
      enabled: true,
      mode: "targets",
      targets: [{ channel: "whatsapp", to: "+15551234567" }],
    },
  },
}
```

`approvals.exec` y `approvals.plugin` son independientes; activar WhatsApp como canal solo vincula el transporte y no envía nada a menos que la familia de aprobaciones correspondiente esté activada y enrutada allí. El modo de sesión entrega aprobaciones de emojis nativas únicamente para las aprobaciones que se originan en WhatsApp. El modo de destino utiliza el pipeline de reenvío compartido para destinos explícitos y no crea una distribución independiente a mensajes directos de aprobadores.

Las reacciones de aprobación de WhatsApp requieren aprobadores explícitos en `allowFrom` (o `"*"`). `defaultTo` establece destinos predeterminados para mensajes normales, no una lista de aprobadores. Los comandos manuales `/approve` siguen pasando por la ruta normal de autorización de remitentes de WhatsApp antes de resolver la aprobación.

## Reacciones a preguntas

Para una solicitud `ask_user` con una pregunta no secreta de selección única y entre una y cuatro opciones, WhatsApp muestra desde `1️⃣` hasta `4️⃣` junto a las etiquetas de las opciones. Reaccione a la solicitud entregada con el número correspondiente para responder. OpenClaw asigna el número a la opción canónica mediante el Gateway; se ignoran las pulsaciones obsoletas o duplicadas. Las solicitudes con varias preguntas, selección múltiple o texto libre siguen admitiendo únicamente respuestas de texto. Las reglas normales de admisión de mensajes directos y grupos de WhatsApp autorizan al remitente que reacciona.

## Hooks de plugins y privacidad

Los mensajes entrantes de WhatsApp pueden contener información personal, números de teléfono, identificadores de grupos, nombres de remitentes y campos de correlación de sesiones. WhatsApp no difunde a los plugins las cargas útiles entrantes del hook `message_received` salvo que se habilite explícitamente:

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

Limite la habilitación a una cuenta mediante `channels.whatsapp.accounts.<id>.pluginHooks.messageReceived`. Active esta opción únicamente para plugins en los que confíe para gestionar el contenido y los identificadores entrantes de WhatsApp.

## Control de acceso y activación

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.whatsapp.dmPolicy`:

    | Valor | Comportamiento |
    | --- | --- |
    | `pairing` (predeterminado) | Los remitentes desconocidos solicitan vinculación; el propietario los aprueba |
    | `allowlist` | Solo se admiten los remitentes de `allowFrom` |
    | `open` | Requiere que `allowFrom` incluya `"*"` |
    | `disabled` | Bloquear todos los mensajes directos |

    `allowFrom` acepta números con formato E.164 (normalizados internamente). Es únicamente una lista de control de acceso de remitentes de mensajes directos; no restringe los envíos salientes explícitos a JID de grupos ni a JID de canales de `@newsletter`.

    Anulación para varias cuentas: `channels.whatsapp.accounts.<id>.dmPolicy` (y `.allowFrom`) tienen prioridad sobre los valores predeterminados del canal para esa cuenta.

    Notas sobre el entorno de ejecución:

    - los emparejamientos persisten en el almacén de permitidos del canal y se combinan con la configuración de `allowFrom`
    - la automatización programada y la selección alternativa de destinatarios de Heartbeat usan destinos de entrega explícitos o la configuración de `allowFrom`; las aprobaciones de emparejamiento de mensajes directos no se convierten implícitamente en destinatarios de Cron/Heartbeat
    - si no se configura ninguna lista de permitidos, el número propio vinculado se permite de forma predeterminada
    - OpenClaw nunca empareja automáticamente los mensajes directos salientes de `fromMe` (mensajes que se envían al propio usuario desde el dispositivo vinculado)

  </Tab>

  <Tab title="Política de grupos y listas de permitidos">
    El acceso a grupos tiene dos niveles:

    1. **Lista de miembros de grupos permitidos** (`channels.whatsapp.groups`): si se omite `groups`, todos los grupos son aptos; si está presente, actúa como lista de grupos permitidos (`"*"` admite todos).
    2. **Política de remitentes de grupos** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`): `open` omite la lista de remitentes permitidos, `allowlist` requiere una coincidencia con `groupAllowFrom` (o `*`) y `disabled` bloquea todos los mensajes entrantes de grupos.

    Si no se establece `groupAllowFrom`, las comprobaciones de remitentes recurren a `allowFrom` cuando contiene entradas. Las listas de remitentes permitidos se evalúan antes de la activación mediante mención o respuesta.

    Si no existe ningún bloque `channels.whatsapp`, el entorno de ejecución recurre a `groupPolicy: "allowlist"` (con una advertencia en el registro), aunque `channels.defaults.groupPolicy` tenga otro valor.

    <Note>
    La resolución de pertenencia a grupos tiene una protección para cuentas únicas: si solo hay una cuenta de WhatsApp configurada y su `accounts.<id>.groups` es un objeto vacío explícito (`{}`), se considera «no establecido» y se recurre al mapa raíz `channels.whatsapp.groups`, en lugar de bloquear silenciosamente todos los grupos. Con 2 o más cuentas configuradas, un mapa de cuenta vacío explícito permanece vacío y no recurre al mapa raíz; esto permite que una cuenta deshabilite intencionadamente todos los grupos sin afectar a las demás.
    </Note>

  </Tab>

  <Tab title="Menciones y /activation">
    Las respuestas en grupos requieren una mención de forma predeterminada. La detección de menciones incluye:

    - menciones explícitas de WhatsApp a la identidad del bot
    - patrones de expresiones regulares de menciones configurados (`agents.entries.*.groupChat.mentionPatterns`, con `messages.groupChat.mentionPatterns` como alternativa)
    - transcripciones de notas de voz entrantes para mensajes de grupos autorizados
    - detección implícita de respuesta al bot (el remitente de la respuesta coincide con la identidad del bot)

    Seguridad: citar o responder solo satisface el requisito de mención; **no** concede autorización al remitente. Con `groupPolicy: "allowlist"`, los remitentes que no están en la lista de permitidos siguen bloqueados aunque respondan al mensaje de un usuario permitido.

    Comando de activación a nivel de sesión: `/activation mention` o `/activation always`. Esto actualiza el estado de la sesión (no la configuración global) y está restringido al propietario.

  </Tab>
</Tabs>

## Enlaces ACP configurados

WhatsApp admite enlaces ACP persistentes mediante `bindings[]` en el nivel superior:

```json5
{
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "direct", id: "+15555550123" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "group", id: "120363424282127706@g.us" },
      },
    },
  ],
}
```

Los chats directos coinciden con números E.164; los grupos coinciden con JID de grupos de WhatsApp. Las listas de grupos permitidos, la política de remitentes y el requisito de mención/activación se ejecutan antes de que OpenClaw garantice la existencia de la sesión ACP vinculada. Un enlace coincidente controla la ruta: los grupos de difusión no distribuyen ese turno a las sesiones ordinarias de WhatsApp.

## Comportamiento del número personal y del chat propio

Cuando el número propio vinculado también está presente en `allowFrom`, se activan las protecciones para el chat propio: se omiten las confirmaciones de lectura para los turnos del chat propio, se ignora el comportamiento de activación automática mediante el JID de mención que provocaría una mención al propio usuario y las respuestas utilizan de forma predeterminada `[{identity.name}]` (o `[openclaw]`) cuando no se establece `responsePrefix` para el canal o la cuenta.

## Normalización de mensajes y contexto

<AccordionGroup>
  <Accordion title="Envoltorio de entrada y contexto de respuesta">
    Los mensajes entrantes se encapsulan en el envoltorio de entrada compartido. Una respuesta que cita otro mensaje añade contexto con este formato:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    Los metadatos de respuesta (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, JID/E.164 del remitente) se rellenan cuando están disponibles. Si el destino citado es contenido multimedia descargable, OpenClaw lo guarda mediante el almacén habitual de contenido multimedia entrante y expone `MediaPath`/`MediaType` para que el agente pueda inspeccionarlo directamente, en lugar de ver únicamente `<media:image>`.

  </Accordion>

  <Accordion title="Marcadores de contenido multimedia y extracción de ubicaciones/contactos">
    Los mensajes que solo contienen contenido multimedia se normalizan como marcadores: `<media:image>`, `<media:video>`, `<media:audio>`, `<media:document>`, `<media:sticker>`.

    Las notas de voz de grupos autorizados se transcriben antes de comprobar las menciones cuando el cuerpo solo contiene `<media:audio>`, por lo que mencionar al bot en la nota de voz puede activar la respuesta. Si la transcripción sigue sin mencionar al bot, permanece en el historial pendiente del grupo en lugar de conservar el marcador sin procesar.

    Los cuerpos de ubicación se representan como texto conciso de coordenadas. Las etiquetas o comentarios de ubicación y los detalles de contactos/vCard se representan como metadatos no confiables delimitados, no como texto insertado directamente en el prompt.

  </Accordion>

  <Accordion title="Inyección del historial pendiente de grupos">
    Los mensajes de grupos no procesados se almacenan temporalmente y se insertan como contexto cuando finalmente se activa el bot.

    - límite predeterminado: `50`
    - configuración: `channels.whatsapp.historyLimit`, con `messages.groupChat.historyLimit` como alternativa
    - `0` deshabilita esta función

    Marcadores de inyección: `[Chat messages since your last reply - for context]` y `[Current message - respond to this]`.

  </Accordion>

  <Accordion title="Confirmaciones de lectura">
    Están habilitadas de forma predeterminada para los mensajes entrantes aceptados. Para deshabilitarlas globalmente:

    ```json5
    { channels: { whatsapp: { sendReadReceipts: false } } }
    ```

    Anulación por cuenta: `channels.whatsapp.accounts.<id>.sendReadReceipts`. Los turnos del chat propio omiten las confirmaciones de lectura aunque estén habilitadas globalmente.

  </Accordion>
</AccordionGroup>

## Entrega, fragmentación y contenido multimedia

<AccordionGroup>
  <Accordion title="Fragmentación de texto">
    - límite predeterminado de fragmentos: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.streaming.chunkMode = "length" | "newline"`; `newline` prioriza los límites entre párrafos (líneas en blanco) y después recurre a una fragmentación segura según la longitud

  </Accordion>

  <Accordion title="Comportamiento del contenido multimedia saliente">
    - admite cargas de imágenes, vídeos, audio (nota de voz PTT) y documentos
    - el audio se envía como carga `audio` de Baileys con `ptt: true`, y se representa como una nota de voz de pulsar para hablar; `audioAsVoice` se conserva en las cargas de respuesta para que la salida de notas de voz de TTS permanezca en esta ruta independientemente del formato de origen del proveedor
    - el audio Ogg/Opus nativo se envía como `audio/ogg; codecs=opus`; cualquier otro formato (incluida la salida MP3/WebM de TTS de Microsoft Edge) se transcodifica mediante `ffmpeg` a Ogg/Opus mono de 48 kHz antes de la entrega mediante PTT
    - `/tts latest` envía la respuesta más reciente del asistente como una sola nota de voz y evita los envíos repetidos de la misma respuesta; `/tts chat on|off|default` controla el TTS automático del chat actual
    - habilitar `gifPlayback: true` en vídeos permite la reproducción como GIF animado
    - `forceDocument`/`asDocument` dirige las imágenes, los GIF y los vídeos salientes a través de la carga de documentos de Baileys para evitar la compresión multimedia de WhatsApp y conservar el nombre de archivo y el tipo MIME resueltos
    - los pies de contenido se aplican al primer elemento multimedia de una respuesta con varios elementos, excepto en las notas de voz PTT: el audio se envía primero sin pie y después el pie se envía como un mensaje de texto independiente (los clientes de WhatsApp no representan de forma coherente los pies de las notas de voz)
    - el origen del contenido multimedia puede ser HTTP(S), `file://` o una ruta local

  </Accordion>

  <Accordion title="Límites de tamaño del contenido multimedia y comportamiento alternativo">
    - límite de guardado entrante y de envío saliente: `channels.whatsapp.mediaMaxMb` (valor predeterminado: `50`)
    - anulación por cuenta: `channels.whatsapp.accounts.<id>.mediaMaxMb`
    - las imágenes se optimizan automáticamente (ajuste de tamaño/calidad) para cumplir los límites, salvo que `forceDocument`/`asDocument` solicite la entrega como documento
    - si falla el envío de contenido multimedia, la alternativa para el primer elemento envía una advertencia de texto en lugar de descartar silenciosamente la respuesta

  </Accordion>
</AccordionGroup>

## Citas en respuestas

`channels.whatsapp.replyToMode` controla las citas nativas en respuestas (las respuestas salientes citan visiblemente el mensaje entrante):

| Valor             | Comportamiento                                                       |
| ----------------- | -------------------------------------------------------------- |
| `"off"` (predeterminado) | No citar nunca; enviar como mensaje sin formato                           |
| `"first"`         | Citar solo el primer fragmento de respuesta saliente                      |
| `"all"`           | Citar cada fragmento de respuesta saliente                               |
| `"batched"`       | Citar las respuestas agrupadas en cola; dejar sin cita las respuestas inmediatas |

Anulación por cuenta: `channels.whatsapp.accounts.<id>.replyToMode`.

```json5
{ channels: { whatsapp: { replyToMode: "first" } } }
```

## Nivel de reacciones

`channels.whatsapp.reactionLevel` controla con qué amplitud utiliza el agente las reacciones con emojis:

| Nivel                 | Reacciones de confirmación | Reacciones iniciadas por el agente  |
| --------------------- | ------------- | -------------------------- |
| `"off"`               | No            | No                         |
| `"ack"`               | Sí           | No                         |
| `"minimal"` (predeterminado) | Sí           | Sí, directrices conservadoras |
| `"extensive"`         | Sí           | Sí, directrices que las fomentan   |

Anulación por cuenta: `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{ channels: { whatsapp: { reactionLevel: "ack" } } }
```

## Reacciones de confirmación

`channels.whatsapp.ackReaction` envía una reacción inmediata al recibir un mensaje entrante, condicionada por `reactionLevel` (se suprime cuando `"off"`):

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // siempre | menciones | nunca
      },
    },
  },
}
```

Notas: se envía inmediatamente después de aceptar el mensaje entrante (antes de responder); si `ackReaction` está presente sin `emoji`, WhatsApp utiliza el emoji de identidad del agente al que se dirige el mensaje y recurre a "👀" como alternativa (omita `ackReaction` o establezca `emoji: ""` para no enviar confirmación); los errores se registran, pero no bloquean la entrega de la respuesta; el modo de grupo `mentions` solo reacciona en los turnos activados mediante una mención, mientras que la activación de grupo `always` omite esa comprobación; WhatsApp solo utiliza `channels.whatsapp.ackReaction` (el `messages.ackReaction` heredado no se aplica aquí).

## Reacciones de estado del ciclo de vida

Establezca `messages.statusReactions.enabled: true` para permitir que WhatsApp sustituya la reacción de confirmación durante un turno en lugar de dejar un emoji de recepción estático, pasando por estados como en cola, pensando, actividad de herramientas, Compaction, finalizado y error:

```json5
{
  messages: {
    statusReactions: {
      enabled: true,
    },
  },
}
```

Notas: `channels.whatsapp.ackReaction` sigue controlando la aptitud para mensajes directos y grupos; el estado en cola utiliza el mismo emoji efectivo que las reacciones de confirmación simples; WhatsApp dispone de un espacio de reacción del bot por mensaje, por lo que las actualizaciones del ciclo de vida sustituyen la reacción actual y restauran la confirmación después del estado final de finalización o error.

## Varias cuentas y credenciales

<AccordionGroup>
  <Accordion title="Selección de cuentas y valores predeterminados">
    Los identificadores de cuenta provienen de `channels.whatsapp.accounts`. La selección de cuenta predeterminada es `default` si está presente; de lo contrario, se usa el primer identificador de cuenta configurado (ordenado alfabéticamente). Los identificadores de cuenta se normalizan internamente para su búsqueda.
  </Accordion>

  <Accordion title="Rutas de credenciales y compatibilidad heredada">
    - ruta de autenticación actual: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (copia de seguridad: `creds.json.bak`)
    - la autenticación predeterminada heredada en `~/.openclaw/credentials/` todavía se reconoce y migra para los flujos de la cuenta predeterminada

  </Accordion>

  <Accordion title="Comportamiento al cerrar sesión">
    `openclaw channels logout --channel whatsapp [--account <id>]` borra el estado de autenticación de WhatsApp para esa cuenta. Cuando se puede acceder a un Gateway, el cierre de sesión detiene primero el receptor activo de esa cuenta, de modo que la sesión vinculada deja de recibir mensajes antes del siguiente reinicio. `openclaw channels remove --channel whatsapp` también detiene el receptor activo antes de deshabilitar o eliminar la configuración de la cuenta.

    En los directorios de autenticación heredados, se conserva `oauth.json` mientras se eliminan los archivos de autenticación de Baileys.

  </Accordion>
</AccordionGroup>

## Herramientas, acciones y escrituras de configuración

- La compatibilidad con herramientas del agente incluye la acción de reacción de WhatsApp (`react`).
- Controles de acciones: `channels.whatsapp.actions.reactions`, `channels.whatsapp.actions.polls` (el valor predeterminado de las acciones existentes es `true`), `channels.whatsapp.actions.calls` (valor predeterminado: `false`; consulte MeowCaller más arriba).
- Las escrituras de configuración iniciadas por el canal están habilitadas de forma predeterminada; deshabilítelas mediante `channels.whatsapp.configWrites: false`.

## Solución de problemas

<AccordionGroup>
  <Accordion title="Sin vincular (se requiere un código QR)">
    Síntoma: el estado del canal indica que no está vinculado.

```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

  </Accordion>

  <Accordion title="Vinculado, pero desconectado o en un bucle de reconexión">
    Síntoma: una cuenta vinculada presenta desconexiones o intentos de reconexión repetidos.

    Las cuentas inactivas pueden permanecer conectadas más allá del tiempo de espera normal de los mensajes; el supervisor solo reinicia cuando se detiene la actividad del transporte de WhatsApp Web, se cierra el socket o la actividad de la aplicación permanece inactiva más allá del intervalo de seguridad más largo (consulte Modelo de ejecución más arriba).

    Solución:

    ```bash
    openclaw channels status --probe
    openclaw doctor
    openclaw logs --follow
    openclaw gateway status
    ```

    Si el bucle persiste después de corregir la conectividad y los tiempos del host, haga una copia de seguridad del directorio de autenticación de la cuenta y vuelva a vincularla:

    ```bash
    cp -a ~/.openclaw/credentials/whatsapp/<accountId> \
      ~/.openclaw/credentials/whatsapp/<accountId>.bak
    openclaw channels logout --channel whatsapp --account <accountId>
    openclaw channels login --channel whatsapp --account <accountId>
    ```

    Si `~/.openclaw/logs/whatsapp-health.log` indica `Gateway inactive`, pero tanto `openclaw gateway status` como `openclaw channels status --probe` muestran un estado correcto, ejecute `openclaw doctor`. En Linux, doctor advierte sobre entradas heredadas de crontab que invocan el script retirado `~/.openclaw/bin/ensure-whatsapp.sh`; elimine esas entradas con `crontab -e`. Cron puede carecer del entorno del bus de usuario de systemd y hacer que ese script antiguo informe incorrectamente del estado del Gateway.

  </Accordion>

  <Accordion title="El inicio de sesión mediante QR agota el tiempo de espera detrás de un proxy">
    Síntoma: `openclaw channels login --channel whatsapp` falla antes de mostrar un código QR utilizable con `status=408 Request Time-out` o con una desconexión del socket TLS.

    El inicio de sesión de WhatsApp Web utiliza el entorno de proxy estándar del host del Gateway (`HTTPS_PROXY`, `HTTP_PROXY`, variantes en minúsculas, `NO_PROXY`). Verifique que el proceso del Gateway herede el entorno del proxy y que `NO_PROXY` no coincida con `mmg.whatsapp.net`.

  </Accordion>

  <Accordion title="No hay ningún receptor activo al enviar">
    Los envíos salientes fallan de inmediato cuando no existe ningún receptor activo del Gateway para la cuenta de destino. Confirme que el Gateway esté en ejecución y que la cuenta esté vinculada.
  </Accordion>

  <Accordion title="La respuesta aparece en la transcripción, pero no en WhatsApp">
    Las filas de la transcripción registran lo que generó el agente; la entrega mediante WhatsApp se comprueba por separado. OpenClaw solo considera que se envió una respuesta automática después de que Baileys devuelve un identificador de mensaje saliente para al menos un envío visible de texto o contenido multimedia.

    Las reacciones de confirmación son acuses de recibo independientes y anteriores a la respuesta: una reacción correcta no demuestra que la respuesta posterior de texto o contenido multimedia se haya aceptado. Compruebe los registros del Gateway en busca de `auto-reply delivery failed` o `auto-reply was not accepted by WhatsApp provider`.

  </Accordion>

  <Accordion title="Los mensajes de grupo se ignoran inesperadamente">
    Compruebe, en este orden: `groupPolicy`, `groupAllowFrom`/`allowFrom`, las entradas de la lista de permitidos de `groups`, el control de menciones (`requireMention` + patrones de mención) y las claves duplicadas en `openclaw.json` (las entradas posteriores de JSON5 sobrescriben las anteriores; mantenga un único `groupPolicy` por ámbito).

    Si `channels.whatsapp.groups` está presente, WhatsApp aún puede observar mensajes de otros grupos, pero OpenClaw los descarta antes del enrutamiento de sesiones. Añada el JID del grupo a `channels.whatsapp.groups` o añada `groups["*"]` para admitir todos los grupos y mantener la autorización de remitentes en `groupPolicy`/`groupAllowFrom`.

  </Accordion>

  <Accordion title="Advertencia sobre el entorno de ejecución Bun">
    Los Gateways de OpenClaw requieren Node. Bun no proporciona la API `node:sqlite` utilizada por el almacén de estado canónico, y doctor migra los servicios heredados de Bun a Node.
  </Accordion>
</AccordionGroup>

## Prompts del sistema

WhatsApp admite prompts del sistema al estilo de Telegram para grupos y chats directos mediante los mapas `groups` y `direct`.

Resolución para mensajes de grupo: primero se determina el mapa `groups` efectivo; si la cuenta define su propia clave `groups`, esta sustituye por completo el mapa raíz `groups` (sin combinación profunda). La búsqueda del prompt se realiza después en ese único mapa resultante:

1. **Prompt específico del grupo** (`groups["<groupId>"].systemPrompt`): se utiliza cuando existe la entrada del grupo **y** está definida su clave `systemPrompt`. Una cadena vacía (`""`) suprime el comodín y no aplica ningún prompt.
2. **Prompt comodín de grupo** (`groups["*"].systemPrompt`): se utiliza cuando la entrada específica del grupo no existe o existe sin una clave `systemPrompt`.

La resolución para mensajes directos sigue el mismo patrón con el mapa `direct` y `direct["*"]`.

<Note>
`dms` sigue siendo el contenedor ligero de reemplazos del historial por mensaje directo (`dms.<id>.historyLimit`). Los reemplazos de prompts se encuentran en `direct`.
</Note>

<Note>
Este comportamiento en el que la cuenta sustituye la raíz para resolver prompts es un reemplazo superficial simple: cualquier clave `groups`/`direct` de la cuenta, incluido un objeto vacío explícito, sustituye el mapa raíz. Es diferente de la comprobación de la lista de permitidos para pertenencia a grupos descrita más arriba, que dispone de una protección para cuentas únicas en caso de que `groups: {}` quede vacío accidentalmente.
</Note>

**Diferencia con Telegram:** Telegram suprime el valor raíz `groups` para todas las cuentas de una configuración con varias cuentas (incluso para las cuentas que no tienen su propio `groups`) para evitar que un bot reciba mensajes de grupos a los que no pertenece. WhatsApp no aplica esa protección: cualquier cuenta sin un reemplazo propio hereda los valores raíz `groups`/`direct`, independientemente del número de cuentas. En una configuración de WhatsApp con varias cuentas, defina explícitamente el mapa completo en cada cuenta si desea prompts por cuenta.

Comportamiento importante:

- `channels.whatsapp.groups` es tanto un mapa de configuración por grupo como la lista de permitidos de grupos a nivel de chat. Tanto en el ámbito raíz como en el de la cuenta, `groups["*"]` significa «se admiten todos los grupos» para ese ámbito.
- Añada un comodín `systemPrompt` solo cuando ya desee que ese ámbito admita todos los grupos. Para mantener como elegibles únicamente un conjunto fijo de identificadores de grupo, repita el prompt en cada entrada incluida explícitamente en la lista de permitidos en lugar de utilizar `groups["*"]`.
- La admisión de grupos y la autorización de remitentes son comprobaciones independientes. `groups["*"]` amplía los grupos que llegan al procesamiento de grupos; no autoriza a todos los remitentes de esos grupos. Esto sigue controlado por `groupPolicy`/`groupAllowFrom`.
- `channels.whatsapp.direct` no tiene ningún efecto secundario equivalente para los mensajes directos: `direct["*"]` solo proporciona una configuración predeterminada después de que un mensaje directo ya se haya admitido mediante `dmPolicy` junto con `allowFrom` o las reglas del almacén de emparejamientos.

Ejemplo:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        // Utilizar solo si deben admitirse todos los grupos en el ámbito raíz.
        // Se aplica a todas las cuentas que no definen su propio mapa groups.
        "*": { systemPrompt: "Prompt predeterminado para todos los grupos." },
      },
      direct: {
        // Se aplica a todas las cuentas que no definen su propio mapa direct.
        "*": { systemPrompt: "Prompt predeterminado para todos los chats directos." },
      },
      accounts: {
        work: {
          groups: {
            // Esta cuenta define sus propios grupos, por lo que los grupos raíz se
            // sustituyen por completo. Para conservar un comodín, defina "*" explícitamente también aquí.
            "120363406415684625@g.us": {
              requireMention: false,
              systemPrompt: "Centrarse en la gestión de proyectos.",
            },
            // Utilizar solo si deben admitirse todos los grupos en esta cuenta.
            "*": { systemPrompt: "Prompt predeterminado para los grupos de trabajo." },
          },
          direct: {
            // Esta cuenta define su propio mapa direct, por lo que las entradas direct raíz se
            // sustituyen por completo. Para conservar un comodín, defina "*" explícitamente también aquí.
            "+15551234567": { systemPrompt: "Prompt para un chat directo de trabajo específico." },
            "*": { systemPrompt: "Prompt predeterminado para los chats directos de trabajo." },
          },
        },
      },
    },
  },
}
```

## Referencias de configuración

Referencia principal: [Referencia de configuración: WhatsApp](/es/gateway/config-channels#whatsapp)

| Área                       | Campos                                                                                                         |
| -------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Acceso                     | `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`                                             |
| Entrega                    | `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `sendReadReceipts`, `ackReaction`, `reactionLevel`      |
| Varias cuentas             | `accounts.<id>.enabled`, `accounts.<id>.authDir` y otros reemplazos por cuenta                              |
| Operaciones                | `configWrites`, `enabled`                                                                                      |
| Agrupación de entradas     | `messages.inbound.debounceMs`, `messages.inbound.byChannel.whatsapp`                                           |
| Comportamiento de sesión   | `session.dmScope`, `historyLimit`, `dmHistoryLimit`, `dms.<id>.historyLimit`                                   |
| Prompts                    | `groups.<id>.systemPrompt`, `groups["*"].systemPrompt`, `direct.<id>.systemPrompt`, `direct["*"].systemPrompt` |

## Temas relacionados

- [Emparejamiento](/es/channels/pairing)
- [Grupos](/es/channels/groups)
- [Seguridad](/es/gateway/security)
- [Enrutamiento de canales](/es/channels/channel-routing)
- [Enrutamiento multiagente](/es/concepts/multi-agent)
- [Solución de problemas](/es/channels/troubleshooting)
