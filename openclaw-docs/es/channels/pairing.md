---
read_when:
    - Configuración del control de acceso a mensajes directos
    - Emparejamiento de un nuevo Node de iOS/Android
    - Revisión de la postura de seguridad de OpenClaw
summary: 'Descripción general del emparejamiento: aprueba quién puede enviarte mensajes directos y qué nodos pueden unirse'
title: Emparejamiento
x-i18n:
    generated_at: "2026-07-26T04:31:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc874d660509f59bc26795c8b3ce13f5d238cd61154c717637f5d545b995fb08
    source_path: channels/pairing.md
    workflow: 16
---

El «emparejamiento» es el paso explícito de aprobación de acceso de OpenClaw.
Se utiliza en dos lugares:

1. **Emparejamiento de mensajes directos** (quién puede comunicarse con el bot)
2. **Emparejamiento de Node** (qué dispositivos o Nodes pueden unirse a la red del Gateway)

Contexto de seguridad: [Seguridad](/es/gateway/security)

## 1) Emparejamiento de mensajes directos (acceso al chat entrante)

Cuando un canal está configurado con la política de mensajes directos `pairing`, los remitentes desconocidos reciben un código corto y su mensaje **no se procesa** hasta que se aprueba.

Las políticas predeterminadas de mensajes directos se documentan en: [Seguridad](/es/gateway/security)

`dmPolicy: "open"` es público solo cuando la lista efectiva de remitentes permitidos para mensajes directos incluye `"*"`.
La configuración y la validación requieren ese comodín para las configuraciones públicas abiertas. Si el estado
existente contiene `open` con entradas `allowFrom` concretas, el entorno de ejecución sigue admitiendo
solo a esos remitentes, y las aprobaciones del almacén de emparejamiento no amplían el acceso a `open`.

Códigos de emparejamiento:

- 8 caracteres, en mayúsculas y sin caracteres ambiguos (`0O1I`).
- **Caducan después de 1 hora**. El bot solo envía el mensaje de emparejamiento cuando se crea una solicitud nueva (aproximadamente una vez por hora y remitente).
- Las solicitudes pendientes de emparejamiento de mensajes directos están limitadas a **3 por cuenta de canal**; las solicitudes adicionales se ignoran hasta que una caduque o se apruebe.

### Aprobación desde la interfaz de control

Abra **Configuración → Canales → Solicitudes de acceso a mensajes directos**. La cola combina las solicitudes
pendientes de todas las cuentas de canal configuradas cuya política de mensajes directos sea `pairing`.
Filtre por canal o cuenta, revise el ID y los metadatos del remitente y, a continuación, seleccione
**Aprobar**.

La aprobación concede únicamente acceso mediante mensajes directos. No concede acceso a grupos. El
cuadro de diálogo de aprobación también ofrece estas opciones explícitas cuando son compatibles:

- **Notificar al solicitante después de la aprobación**
- **Convertir también a este remitente en el primer propietario de comandos**, que solo se muestra cuando no existe ningún propietario
  de comandos y la sesión de la interfaz de control tiene `operator.admin`

Seleccione **Descartar** para eliminar una solicitud pendiente sin aprobarla. El descarte
no constituye un bloqueo permanente; el remitente puede volver a solicitar acceso más adelante.

### Aprobación desde la CLI

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

Añada `--notify` para avisar al solicitante en el mismo canal. Los canales con varias cuentas
aceptan `--account <id>`.

A diferencia de la casilla explícita de la interfaz de control, la CLI inicializa automáticamente
`commands.ownerAllowFrom` cuando no hay ningún propietario de comandos configurado, mediante una entrada
como `telegram:123456789`. Esto proporciona a las configuraciones iniciales un propietario explícito para
los comandos privilegiados y las solicitudes de aprobación de ejecución. Una vez que existe un propietario, las
aprobaciones de emparejamiento posteriores solo conceden acceso a mensajes directos; no añaden más propietarios.

<Note>
El QR de inicio de sesión de WhatsApp vincula una cuenta de WhatsApp con OpenClaw. Las solicitudes de acceso a mensajes directos
aprueban a las personas que envían mensajes a esa cuenta. Son flujos independientes.
</Note>

Canales compatibles (cualquier Plugin de canal instalado que declare emparejamiento; los Plugins externos como `openclaw-weixin` pueden añadir más): `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `sms`, `synology-chat`, `telegram`, `twitch`, `whatsapp`, `zalo`, `zalouser`.

### Grupos de remitentes reutilizables

Utilice `accessGroups` en el nivel superior cuando el mismo conjunto de remitentes de confianza deba aplicarse a
varios canales de mensajería o tanto a las listas de permitidos de mensajes directos como a las de grupos.

Los grupos estáticos utilizan `type: "message.senders"` y se hace referencia a ellos mediante
`accessGroup:<name>` desde las listas de permitidos de los canales:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
  channels: {
    telegram: { dmPolicy: "allowlist", allowFrom: ["accessGroup:operators"] },
    whatsapp: { groupPolicy: "allowlist", groupAllowFrom: ["accessGroup:operators"] },
  },
}
```

Los grupos de acceso se documentan en detalle aquí: [Grupos de acceso](/es/channels/access-groups)

### Ubicación del estado

Se almacena en la base de datos de estado SQLite compartida en
`~/.openclaw/state/openclaw.sqlite`:

- solicitudes pendientes en `channel_pairing_requests`
- remitentes aprobados en `channel_pairing_allow_entries`

Comportamiento del ámbito de las cuentas:

- cada solicitud y remitente aprobado se identifica mediante el canal y la cuenta
- el entorno de ejecución solo lee las filas canónicas de SQLite; no combina archivos heredados

Los Gateways anteriores escribían `<channel>-pairing.json` y
`<channel>-<accountId>-allowFrom.json` en `~/.openclaw/credentials/`.
La migración al iniciar y `openclaw doctor --fix` importan esos archivos en SQLite y
eliminan cada origen tras una importación correcta. Trate la base de datos SQLite como
información confidencial, ya que estas filas controlan el acceso al asistente.

<Note>
El almacén de listas de permitidos de emparejamiento sirve para el acceso mediante mensajes directos. La autorización de grupos es independiente.
Aprobar un código de emparejamiento de mensajes directos no permite automáticamente que ese remitente ejecute comandos de grupo
ni controle el bot en grupos. La inicialización del primer propietario es un estado de configuración independiente
en `commands.ownerAllowFrom`, y la entrega de mensajes en chats grupales sigue respetando las
listas de permitidos de grupos del canal (por ejemplo, `groupAllowFrom`, `groups` o las anulaciones por grupo
o por tema, según el canal).
</Note>

## 2) Emparejamiento de dispositivos Node (Nodes iOS/Android/macOS/sin interfaz)

Los Nodes se conectan al Gateway como **dispositivos** con `role: node`. El Gateway
crea una solicitud de emparejamiento de dispositivo que debe aprobarse.

### Emparejamiento desde la interfaz de control (recomendado)

Utilice una sesión de la interfaz de control ya conectada con acceso `operator.admin`:

1. Abra la interfaz de control y vaya a **Configuración → Dispositivos**.
2. En la página **Dispositivos**, haga clic en **Emparejar dispositivo móvil**.
3. Mantenga **Acceso completo (recomendado)** o seleccione **Acceso limitado** para omitir
   los controles administrativos del Gateway.
4. Haga clic en **Crear código de configuración**.
5. En el teléfono, abra la aplicación OpenClaw → **Configuración** → **Gateway**.
6. Escanee el código QR o pegue el código de configuración y, a continuación, conéctese.

Las aplicaciones oficiales de OpenClaw para iOS y Android se aprueban automáticamente cuando los
metadatos de su código de configuración coinciden. Si **Aprobación pendiente** muestra una solicitud (por
ejemplo, de un cliente no oficial o con metadatos que no coinciden), revise su rol y
sus ámbitos antes de aprobarla.

El botón está deshabilitado cuando la sesión actual de la interfaz de control no tiene
acceso de administrador. En ese caso, utilice el siguiente flujo de aprobación mediante la CLI desde el host del Gateway.

### Emparejamiento mediante Telegram

Si utiliza el Plugin `device-pair`, puede realizar el emparejamiento inicial del dispositivo íntegramente desde Telegram:

1. En Telegram, envíe este mensaje al bot: `/pair`
2. El bot responde con dos mensajes: uno con instrucciones y otro independiente con el **código de configuración** (fácil de copiar y pegar en Telegram).
3. En el teléfono, abra la aplicación OpenClaw para iOS → Configuración → Gateway.
4. Escanee el código QR (`/pair qr`) o pegue el código de configuración y conéctese.
5. La aplicación móvil oficial se conecta automáticamente. Si `/pair pending` muestra una
   solicitud, revise su rol y sus ámbitos antes de aprobarla.

El código de configuración es una carga JSON codificada en base64 que contiene:

- `url`: la URL de WebSocket del Gateway (`ws://...` o `wss://...`)
- `urls`: cuando estén disponibles, las rutas LAN/Tailnet ordenadas que puede probar la aplicación móvil
- `bootstrapToken`: un token de inicialización de un solo uso para el intercambio inicial de emparejamiento; el Gateway lo hace caducar después de 10 minutos

Ejecute `/pair cleanup` para invalidar los códigos de configuración no utilizados una vez finalizado el emparejamiento.

Ese token de inicialización incluye el perfil integrado de inicialización de emparejamiento:

- una configuración segura de `wss://` (o de bucle invertido en el mismo host) utiliza de forma predeterminada `node` más acceso
  completo a `operator` para dispositivos móviles nativos
- el token `node` transferido permanece como `scopes: []`
- el token `operator` transferido de forma predeterminada incluye `operator.admin`,
  `operator.approvals`, `operator.read`, `operator.talk.secrets` y
  `operator.write`
- El **Acceso limitado** de la interfaz de control y `openclaw qr --limited` omiten
  `operator.admin`, pero mantienen los demás ámbitos de operador
- la configuración LAN de texto sin formato `ws://` utiliza automáticamente el mismo perfil limitado;
  configure `wss://` o Tailscale Serve y genere un código nuevo para obtener acceso completo
- la posterior rotación o revocación del token sigue estando limitada tanto por el contrato de rol aprobado
  del dispositivo como por los ámbitos de operador de la sesión que realiza la llamada

Trate el código de configuración como una contraseña mientras sea válido.

Las páginas **Configuración → Gateway** de iOS y Android muestran acceso **Completo** o **Limitado**.
Para ampliar el acceso de un teléfono limitado, configure primero una ruta segura `wss://` o
Tailscale Serve; a continuación, genere un nuevo código de configuración con acceso completo, escanéelo o péguelo
en esa página de configuración y vuelva a conectarse.

Para el emparejamiento móvil mediante Tailscale, una red pública u otra conexión remota, utilice Tailscale Serve/Funnel
u otra URL `wss://` del Gateway. Los códigos de configuración `ws://` en texto sin formato solo se aceptan
para el bucle invertido, direcciones LAN privadas, hosts Bonjour `.local` y el host del
emulador de Android. Las rutas de texto sin formato que no sean de bucle invertido reciben acceso limitado. Las direcciones
CGNAT de Tailnet, los nombres `.ts.net` y los hosts públicos siguen rechazándose de forma segura antes
de emitir el QR o el código de configuración.

Para las URL de configuración `gateway.bind=lan`, OpenClaw detecta las raíces HTTPS persistentes de Tailscale Serve
que actúan como proxy del puerto de bucle invertido del Gateway activo y las anuncia
junto con la ruta LAN. El comando de configuración añade esta alternativa solo
para `lan`; `custom` y `tailnet` conservan sus rutas anunciadas explícitamente. La
aplicación para iOS prueba las rutas anunciadas en orden y guarda el primer
punto de conexión accesible.

### Aprobación de un dispositivo Node

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Cuando se deniega una aprobación explícita porque la sesión del dispositivo emparejado que aprueba
se abrió con un ámbito exclusivo de emparejamiento, la CLI vuelve a intentar la misma solicitud con
`operator.admin`. Esto permite que un dispositivo emparejado existente con capacidad administrativa recupere un nuevo
emparejamiento de la interfaz de control o del navegador sin editar manualmente el almacén de emparejamiento. El
Gateway sigue validando la conexión reintentada; los tokens que no puedan autenticarse
con `operator.admin` permanecen bloqueados.

Si el mismo dispositivo vuelve a intentarlo con datos de autenticación diferentes (por ejemplo, un
rol, ámbitos o clave pública distintos), la solicitud pendiente anterior queda reemplazada y se crea un nuevo
`requestId`.

<Note>
Un dispositivo ya emparejado no obtiene silenciosamente un acceso más amplio. Si vuelve a conectarse solicitando más ámbitos o un rol más amplio, OpenClaw mantiene la aprobación existente sin cambios y crea una nueva solicitud pendiente de ampliación. Utilice `openclaw devices list` para comparar el acceso aprobado actualmente con el nuevo acceso solicitado antes de aprobarlo.
</Note>

### Aprobación automática opcional de Nodes mediante CIDR de confianza

El emparejamiento de dispositivos sigue siendo manual de forma predeterminada. Para redes de Nodes estrictamente controladas,
se puede habilitar la aprobación automática inicial de Nodes mediante direcciones IP exactas o CIDR explícitos:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Esto solo se aplica a solicitudes de emparejamiento `role: node` nuevas que no soliciten
ámbitos. Los clientes de operador, navegador, interfaz de control y WebChat siguen requiriendo aprobación
manual. Los cambios de rol, ámbito, metadatos y clave pública también siguen requiriendo aprobación
manual.

### Almacenamiento del estado de emparejamiento de Nodes

Se almacena en la base de datos de estado SQLite compartida en `~/.openclaw/state/openclaw.sqlite`:

- solicitudes pendientes de emparejamiento de dispositivos (de corta duración; caducan después de 5 minutos)
- dispositivos emparejados y tokens

Los gateways anteriores conservaban este estado en `~/.openclaw/devices/*.json`; esos archivos se
importan en SQLite al iniciar el gateway y se archivan con el sufijo `.migrated`.

### Notas

- La API `node.pair.*` (CLI: `openclaw nodes pending|approve|reject|remove|rename`) gestiona
  las aprobaciones de capacidades de los nodos almacenadas en los mismos registros de dispositivos emparejados. Los nodos WS
  siguen requiriendo el emparejamiento de dispositivos; consulte [Emparejamiento de nodos](/es/gateway/pairing).
- El registro de emparejamiento es la fuente de verdad persistente para los roles aprobados. Los tokens de
  dispositivo activos permanecen limitados a ese conjunto de roles aprobados; una entrada de token aislada
  fuera de los roles aprobados no crea nuevos accesos.

## Documentación relacionada

- Modelo de seguridad e inyección de prompts: [Seguridad](/es/gateway/security)
- Actualización segura (ejecute doctor): [Actualización](/es/install/updating)
- Configuraciones de canales:
  - Telegram: [Telegram](/es/channels/telegram)
  - WhatsApp: [WhatsApp](/es/channels/whatsapp)
  - Signal: [Signal](/es/channels/signal)
  - iMessage: [iMessage](/es/channels/imessage)
  - Discord: [Discord](/es/channels/discord)
  - Slack: [Slack](/es/channels/slack)
