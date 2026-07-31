---
read_when:
    - Planificación de una migración de BlueBubbles al plugin de iMessage incluido
    - Traducción de las claves de configuración de BlueBubbles a sus equivalentes de iMessage
    - Verificación de imsg antes de habilitar el plugin de iMessage
summary: 'Migra las configuraciones antiguas de BlueBubbles al plugin incluido de iMessage: asignación de claves, controles de listas de permitidos para grupos y verificación de la transición.'
title: Procedente de BlueBubbles
x-i18n:
    generated_at: "2026-07-26T04:30:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

Se eliminó la compatibilidad con BlueBubbles. OpenClaw admite iMessage únicamente mediante el plugin `imessage` incluido, que controla [`steipete/imsg`](https://github.com/steipete/imsg) a través de JSON-RPC y accede a la misma superficie de API privada que utilizaba BlueBubbles (`react`, `edit`, `unsend`, `reply`, `sendWithEffect`, encuestas nativas, gestión de grupos y archivos adjuntos). Un único binario de CLI sustituye el servidor de BlueBubbles, la aplicación cliente y la infraestructura de webhooks: no hay ningún endpoint REST ni autenticación de webhooks.

Esta guía permite migrar las configuraciones antiguas de `channels.bluebubbles` a `channels.imessage`. No existe ninguna otra ruta de migración compatible. En la versión actual de OpenClaw, cualquier bloque `channels.bluebubbles` restante queda inerte: ningún componente del entorno de ejecución lo lee.

<Note>
Para consultar el anuncio breve y el resumen para operadores, véase [Eliminación de BlueBubbles y la ruta de iMessage mediante imsg](/es/announcements/bluebubbles-imessage).
</Note>

## Lista de comprobación para la migración

La ruta segura más corta si ya se conoce la configuración antigua de BlueBubbles:

1. Verificar `imsg` directamente en el Mac que ejecuta Messages.app (`imsg chats`, `imsg history`, `imsg send`, `imsg rpc --help`).
2. Copiar las claves de comportamiento de `channels.bluebubbles` a `channels.imessage`: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit` y `actions`.
3. Eliminar las claves de transporte que ya no existen: `serverUrl`, `password`, las URL de webhooks y la configuración del servidor de BlueBubbles.
4. Si el Gateway no se ejecuta en el Mac de Messages, establecer `channels.imessage.cliPath` en un contenedor SSH y configurar `remoteHost` para la obtención remota de archivos adjuntos.
5. Habilitar `channels.imessage`, reiniciar el Gateway y, a continuación, ejecutar `openclaw channels status --probe --channel imessage`.
6. Probar un mensaje directo, un grupo permitido, los archivos adjuntos si están habilitados y todas las acciones de la API privada que se espere que utilice el agente.
7. Eliminar el servidor de BlueBubbles y la configuración antigua de `channels.bluebubbles` después de verificar la ruta de iMessage.

## Qué hace imsg

`imsg` es una CLI local de macOS para Messages. OpenClaw inicia `imsg rpc` como proceso secundario y se comunica mediante JSON-RPC a través de stdin/stdout. No hay ningún servidor HTTP, URL de webhook, daemon en segundo plano, agente de inicio ni puerto que exponer.

- Las lecturas proceden de `~/Library/Messages/chat.db` mediante un identificador de SQLite de solo lectura.
- Los mensajes entrantes en tiempo real proceden de `imsg watch` / `watch.subscribe`, que sigue los eventos del sistema de archivos de `chat.db` con un mecanismo alternativo de sondeo.
- Los envíos utilizan la automatización de Messages.app para enviar texto normal y archivos.
- Las acciones avanzadas utilizan `imsg launch` para inyectar el asistente `imsg` en Messages.app. Esto habilita las confirmaciones de lectura, los indicadores de escritura, los envíos enriquecidos, la edición, la anulación de envíos, las respuestas en hilos, las reacciones, las encuestas y la gestión de grupos.
- Las compilaciones de Linux pueden inspeccionar una copia de `chat.db`, pero no pueden enviar mensajes, observar la base de datos activa del Mac ni controlar Messages.app. Para utilizar iMessage con OpenClaw, se debe ejecutar `imsg` en el Mac con la sesión iniciada o mediante un contenedor SSH que se conecte a ese Mac.

## Antes de comenzar

1. Instalar `imsg` en el Mac que ejecuta Messages.app:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   Para la configuración local habitual, la configuración de OpenClaw puede ofrecer una instalación o actualización de Homebrew de `imsg` confirmada por el usuario en el Mac con la sesión de Messages iniciada. La configuración manual y las topologías con contenedores SSH siguen estando administradas por el operador: se debe repetir la actualización de Homebrew en el mismo contexto de usuario local o remoto que ejecutará `imsg`. Si `imsg chats` falla con `unable to open database file`, no produce ninguna salida o muestra `authorization denied`, se debe conceder acceso total al disco al terminal, editor, proceso de Node, servicio Gateway o proceso principal de SSH que inicia `imsg` y, a continuación, volver a abrir ese proceso principal.

2. Verificar las superficies de lectura, observación, envío y RPC antes de cambiar la configuración de OpenClaw:

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   Sustituir `42` por un identificador de chat real obtenido de `imsg chats`. El envío requiere permiso de automatización para Messages.app. Si OpenClaw se ejecutará mediante SSH, se deben ejecutar estos comandos a través del mismo contenedor SSH o contexto de usuario que utilizará OpenClaw. Si las lecturas funcionan, pero los envíos fallan con el error `-1743` de AppleEvents, se debe comprobar si el permiso de automatización se concedió a `/usr/libexec/sshd-keygen-wrapper`; véase [Los envíos mediante el contenedor SSH fallan con el error -1743 de AppleEvents](/es/channels/imessage#requirements-and-permissions-macos).

3. Habilitar el puente de la API privada. Se recomienda encarecidamente para iMessage con OpenClaw, ya que las respuestas, las reacciones, los efectos, las encuestas, las respuestas a archivos adjuntos y las acciones de grupo dependen de él:

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch` requiere que SIP esté deshabilitado (y, en las versiones modernas de macOS, que la validación de bibliotecas esté flexibilizada; véase [Habilitación de la API privada de imsg](/es/channels/imessage#enabling-the-imsg-private-api)). El envío básico, el historial y la observación funcionan sin `imsg launch`; la superficie completa de acciones de iMessage de OpenClaw no.

4. Después de habilitar `channels.imessage` e iniciar el Gateway, verificar el puente mediante OpenClaw:

   ```bash
   openclaw channels status --probe
   ```

   La cuenta de iMessage debería indicar `works`; con `--json`, la carga útil de la comprobación incluye `privateApi.available: true`. Si indica `false`, se debe corregir primero; véase [Detección de capacidades](/es/channels/imessage#private-api-actions). La comprobación requiere un Gateway accesible (de lo contrario, la CLI recurre a una salida basada únicamente en la configuración) y solo comprueba las cuentas configuradas y habilitadas.

5. Crear una copia de la configuración:

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## Traducción de la configuración

iMessage y BlueBubbles comparten la mayoría de las claves de comportamiento del canal. Lo que cambia es el transporte (servidor REST frente a CLI local) y el formato de las claves del registro de grupos.

| BlueBubbles                                                | iMessage incluido                          | Notas                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | Misma semántica (valor predeterminado: `true` una vez que existe el bloque).                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _(eliminado)_                               | Sin servidor REST: el plugin inicia `imsg rpc` mediante stdio.                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _(eliminado)_                               | No se necesita autenticación de Webhook.                                                                                                                                                                                                                                                |
| _(implícito)_                                               | `channels.imessage.cliPath`               | Ruta a `imsg` (valor predeterminado: `imsg`); use un script contenedor para SSH.                                                                                                                                                                                                                   |
| _(implícito)_                                               | `channels.imessage.dbPath`                | Sustitución opcional de `chat.db` de Messages.app; se detecta automáticamente si se omite.                                                                                                                                                                                                            |
| _(implícito)_                                               | `channels.imessage.remoteHost`            | `host` o `user@host`; solo se necesita cuando `cliPath` es un contenedor SSH y se desea obtener archivos adjuntos mediante SCP.                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | Mismos valores (`pairing` / `allowlist` / `open` / `disabled`); valor predeterminado: `pairing`.                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | Mismos formatos de identificador (`+15555550123`, `user@example.com`). Las aprobaciones del almacén de emparejamiento no se transfieren; consulte más adelante.                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | Mismos valores (`allowlist` / `open` / `disabled`); valor predeterminado: `allowlist`.                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | Igual. Cuando no se establece, iMessage recurre a `allowFrom`; un valor `groupAllowFrom: []` explícitamente vacío bloquea todos los grupos con `groupPolicy: "allowlist"`.                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | Copie literalmente la entrada comodín `"*"`; cambie las claves de las entradas de cada grupo para usar el `chat_id` numérico de iMessage; consulte «Riesgo del registro de grupos». `requireMention`, `tools`, `toolsBySender` y `systemPrompt` se conservan.                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | Valor predeterminado: `true`. Con el plugin incluido, esto solo se activa cuando la sonda de la API privada está operativa.                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | Misma estructura y desactivado de forma predeterminada. Si los archivos adjuntos se transmitían en BlueBubbles, establezca esto explícitamente: las fotos y los archivos multimedia entrantes se descartan de forma silenciosa (sin una línea de registro `Inbound message`) hasta que lo haga.                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | Raíces locales; mismas reglas de comodines.                                                                                                                                                                                                                                                |
| _(N/D)_                                                    | `channels.imessage.remoteAttachmentRoots` | Solo se usa cuando se establece `remoteHost` para las obtenciones mediante SCP.                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | Valor predeterminado de 16 MB en iMessage (el valor predeterminado de BlueBubbles era 8 MB). Establézcalo explícitamente para conservar el límite inferior.                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | Valor predeterminado de 4000 en ambos.                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _(eliminado)_                               | No migre esta clave. `imsg` 0.13.1 y versiones posteriores fusionan los envíos divididos de vistas previas de URL de Apple antes de que OpenClaw los reciba; `openclaw doctor --fix` elimina una clave obsoleta de iMessage.                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _(N/D)_                                   | `imsg` ya muestra los nombres visibles de los remitentes desde `chat.db`.                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | Mismos conmutadores por acción (`reactions`, `edit`, `unsend`, `reply`, `sendWithEffect`, `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, `sendAttachment`) más el nuevo `polls`. Todos están activados de forma predeterminada; las acciones de la API privada siguen requiriendo el puente. |

Las configuraciones de varias cuentas (`channels.bluebubbles.accounts.*`) se traducen individualmente a `channels.imessage.accounts.*`.

## Riesgo del registro de grupos

El plugin de iMessage incluido ejecuta dos filtros de grupo consecutivos. Un mensaje de grupo debe superar ambos para llegar al agente:

1. **Lista de remitentes u objetivos de chat permitidos** (`channels.imessage.groupAllowFrom`): coincide con el identificador del remitente o con el objetivo del chat (entradas `chat_id:`, `chat_guid:`, `chat_identifier:`). Cuando `groupAllowFrom` no está establecido, este filtro recurre a `allowFrom`; un `groupAllowFrom: []` explícito desactiva ese mecanismo alternativo y descarta todos los mensajes de grupo con `groupPolicy: "allowlist"`.
2. **Registro de grupos** (`channels.imessage.groups`): usa como clave el `chat_id` numérico de iMessage:
   - Sin bloque `groups` (o con uno vacío): los grupos superan este filtro siempre que el filtro 1 tenga una lista efectiva de remitentes permitidos que no esté vacía; el filtrado de remitentes controla el acceso y no se activa ninguna advertencia de descarte total durante el inicio.
   - `groups` con entradas pero sin `"*"`: solo se aceptan las claves `chat_id` indicadas. Incluir cualquier grupo convierte el registro en una lista de permitidos, incluso con `groupPolicy: "open"`.
   - `groups: { "*": { ... } }`: todos los grupos superan este filtro.

El riesgo de la migración: BlueBubbles usaba el GUID o identificador del chat como clave para las entradas `groups`, mientras que el registro de iMessage usa el `chat_id` numérico. Copiar literalmente las entradas de cada grupo crea un registro no vacío cuyas claves nunca coinciden, por lo que todos los mensajes de grupo se descartan en el filtro 2. Copie literalmente el comodín `"*"`; cambie las claves de las entradas de grupos específicos para usar los valores `chat_id` de `imsg chats`.

Ambas rutas de descarte son visibles con el nivel de registro predeterminado mediante líneas `warn`:

- Una vez por cuenta durante el inicio, cuando se establece `groupPolicy: "allowlist"` y la lista efectiva de remitentes de grupo permitidos está vacía: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`. Establezca `groupAllowFrom` (o `allowFrom`) para admitir remitentes; añadir solo `groups` no satisface el filtro de remitentes.
- Una vez por `chat_id` durante la ejecución, cuando el registro descarta un grupo: `imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`, indicando la clave exacta que debe añadirse.

Los mensajes directos siguen funcionando en cualquier caso: utilizan una ruta de código diferente, por lo que su funcionamiento no demuestra que el enrutamiento de grupos sea correcto.

La configuración mínima limitada por remitente con `groupPolicy: "allowlist"`:

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

Esto admite a los remitentes configurados en cualquier grupo. Añada entradas `groups` para limitar los chats permitidos o establecer opciones por chat, como `requireMention`; copie literalmente la entrada `"*"` de BlueBubbles, pero cambie las claves de las entradas específicas para usar los valores `chat_id` numéricos de iMessage.

## Paso a paso

1. Traduzca la configuración. Mantenga el nuevo bloque desactivado mientras lo edita; el bloque antiguo `channels.bluebubbles` se ignora en la versión actual de OpenClaw y puede conservarse junto al nuevo como referencia:

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // cambie a true cuando esté listo para realizar la transición
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // copie desde bluebubbles.allowFrom
         groupPolicy: "allowlist",
         groupAllowFrom: [], // copie desde bluebubbles.groupAllowFrom
         groups: { "*": { requireMention: true } }, // el comodín se copia literalmente; cambie las claves de las entradas por chat para usar chat_id
         // las acciones están activadas de forma predeterminada; establezca conmutadores individuales en false para desactivarlas
       },
     },
   }
   ```

2. **Realice la transición y compruebe el funcionamiento.** Establezca `channels.imessage.enabled: true`, reinicie el Gateway y confirme que el canal indique que funciona correctamente:

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # se espera "works"; --json muestra privateApi.available: true
   ```

   La comprobación requiere un Gateway accesible y solo comprueba las cuentas configuradas y habilitadas. Use los comandos directos `imsg` de [Antes de empezar](#before-you-start) para validar el propio Mac.

3. **Verifique los mensajes directos.** Envíe un mensaje directo al agente y confirme que llega la respuesta.

4. **Verifique los grupos por separado.** Los mensajes directos y los grupos siguen rutas de código diferentes: que los mensajes directos funcionen no demuestra que los grupos se enruten correctamente. Envíe un mensaje en un chat grupal permitido y confirme que llega la respuesta. Si el grupo queda en silencio (sin respuesta del agente ni error), compruebe en el registro del Gateway las dos líneas `warn` de «Group registry footgun» más arriba. La advertencia de inicio significa que la lista efectiva de remitentes permitidos está vacía; una advertencia por `chat_id` significa que un registro `groups` con entradas no contiene ese chat.

5. **Verifique las acciones disponibles.** Desde un mensaje directo emparejado, pida al agente que añada una reacción, edite, anule el envío, responda, envíe una foto y, en un grupo, cambie el nombre del grupo o añada o elimine un participante. Cada acción debe reflejarse de forma nativa en Messages.app. Si alguna acción genera `iMessage <action> requires the imsg private API bridge`, vuelva a ejecutar `imsg launch` y actualice con `openclaw channels status --probe`.

6. **Elimine el servidor BlueBubbles y el bloque `channels.bluebubbles`** una vez verificados los mensajes directos, los grupos y las acciones de iMessage. OpenClaw no lee `channels.bluebubbles`.

## Comparación rápida de acciones

| Acción                                              | BlueBubbles heredado | iMessage incluido                                                               |
| --------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------- |
| Enviar texto / alternativa por SMS                  | ✅                    | ✅                                                                              |
| Enviar contenido multimedia (foto, vídeo, archivo, voz) | ✅                | ✅                                                                              |
| Respuesta en hilo (`reply_to_guid`)              | ✅                    | ✅ (resuelve [#51892](https://github.com/openclaw/openclaw/issues/51892))        |
| Tapback (`react`)                        | ✅                    | ✅                                                                              |
| Editar / anular envío (destinatarios con macOS 13+) | ✅                    | ✅                                                                              |
| Enviar con efecto de pantalla                       | ✅                    | ✅ (resuelve parte de [#9394](https://github.com/openclaw/openclaw/issues/9394)) |
| Texto enriquecido en negrita / cursiva / subrayado / tachado | ✅          | ✅ (formato de tramos tipados mediante attributedBody)                          |
| Encuestas nativas de Messages (crear y votar)       | ❌                    | ✅ (`actions.polls`; los destinatarios necesitan iOS/macOS 26+ para la representación nativa) |
| Cambiar el nombre del grupo / establecer el icono del grupo | ✅          | ✅                                                                              |
| Añadir / eliminar participante, abandonar el grupo  | ✅                    | ✅                                                                              |
| Confirmaciones de lectura e indicador de escritura  | ✅                    | ✅ (condicionado a la comprobación de la API privada)                           |
| Fusión del envío dividido de vistas previas de URL de Apple | ✅           | ✅ (gestionado en el componente previo por `imsg` 0.13.1 y versiones posteriores; no hay ningún ajuste de OpenClaw) |
| Recuperación de mensajes entrantes tras un reinicio | ✅                    | ✅ (automática: repetición de `since_rowid` + deduplicación por GUID; ventana más amplia en local) |

iMessage recupera los mensajes que no se recibieron mientras el Gateway estaba inactivo: al iniciarse, los reproduce desde el último rowid enviado mediante `imsg watch.subscribe` `since_rowid`, los deduplica por GUID y un límite de antigüedad de los mensajes pendientes obsoletos impide la «bomba de mensajes pendientes» del vaciado de Push. Esto se ejecuta mediante la conexión RPC `imsg`, por lo que también funciona en configuraciones remotas de `cliPath` mediante SSH; las configuraciones locales disponen de una ventana de recuperación más amplia porque pueden leer `chat.db`. Consulte [Recuperación de mensajes entrantes tras reiniciar un puente o el Gateway](/es/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart).

## Emparejamiento, sesiones y vinculaciones ACP

- **Las listas de permitidos se conservan por identificador.** `channels.imessage.allowFrom` reconoce las mismas cadenas `+15555550123` / `user@example.com` que utilizaba BlueBubbles; cópielas literalmente.
- **Las aprobaciones del almacén de emparejamiento no se transfieren.** El almacén de emparejamiento es específico de cada canal y nada migra el antiguo almacén de BlueBubbles. Los remitentes aprobados únicamente mediante emparejamiento deben volver a emparejarse una vez en iMessage, o bien se deben añadir sus identificadores a `allowFrom`.
- **Las sesiones** siguen estando delimitadas por agente y chat. Los mensajes directos se agrupan en la sesión principal del agente con el valor predeterminado `session.dmScope=main`; las sesiones grupales permanecen aisladas por `chat_id` (`agent:<agentId>:imessage:group:<chat_id>`). El historial de conversaciones anterior asociado a claves de sesión de BlueBubbles no se transfiere a las sesiones de iMessage.
- **Las vinculaciones ACP** que hagan referencia a `match.channel: "bluebubbles"` deben cambiarse a `"imessage"`. Los formatos `match.peer.id` (`chat_id:`, `chat_guid:`, `chat_identifier:`, identificador sin formato) son idénticos.

## No existe un canal de reversión

No existe un entorno de ejecución compatible de BlueBubbles al que volver. Si falla la verificación de iMessage, establezca `channels.imessage.enabled: false`, reinicie el Gateway, resuelva el bloqueo de `imsg` y vuelva a intentar la migración.

La caché de respuestas reside en el estado SQLite del Plugin. `openclaw doctor --fix` importa y archiva el antiguo archivo auxiliar `imessage/reply-cache.jsonl` cuando está presente.

## Contenido relacionado

- [Eliminación de BlueBubbles y la ruta de iMessage mediante imsg](/es/announcements/bluebubbles-imessage) — anuncio breve y resumen para operadores.
- [iMessage](/es/channels/imessage) — referencia completa del canal iMessage, incluida la configuración de `imsg launch` y la detección de capacidades.
- `/channels/bluebubbles` — URL heredada que redirige a esta guía de migración.
- [Emparejamiento](/es/channels/pairing) — autenticación de mensajes directos y flujo de emparejamiento.
- [Enrutamiento de canales](/es/channels/channel-routing) — cómo el Gateway selecciona un canal para las respuestas salientes.
