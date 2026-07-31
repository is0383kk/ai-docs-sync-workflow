---
read_when:
    - Usabas el antiguo canal BlueBubbles y necesitas migrar a iMessage
    - Está eligiendo la configuración compatible de iMessage para OpenClaw
    - Necesitas una breve explicación de la eliminación de BlueBubbles
summary: Se eliminó la compatibilidad con BlueBubbles de OpenClaw. Use el plugin de iMessage incluido con imsg para las configuraciones de iMessage nuevas y migradas.
title: Eliminación de BlueBubbles y la ruta de iMessage mediante imsg
x-i18n:
    generated_at: "2026-07-26T04:30:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dec7d3f27e0df6431494d864b0c7ae7457574797e199f9a2cb6931d28feacd0
    source_path: announcements/bluebubbles-imessage.md
    workflow: 16
---

# Eliminación de BlueBubbles y la vía de iMessage con imsg

OpenClaw ya no incluye el canal BlueBubbles. La compatibilidad con iMessage funciona mediante el plugin `imessage` incluido: el Gateway inicia [`imsg`](https://github.com/steipete/imsg) como proceso secundario, localmente o mediante un contenedor SSH, y se comunica por JSON-RPC a través de stdin/stdout. Sin servidor, sin webhook, sin puerto.

Si la configuración aún contiene `channels.bluebubbles`, migre a `channels.imessage`. La antigua URL de documentación `/channels/bluebubbles` redirige a [Migración desde BlueBubbles](/es/channels/imessage-from-bluebubbles), que contiene la tabla completa de conversión de la configuración y la lista de comprobación para la transición.

## Qué ha cambiado

- La vía de iMessage compatible no tiene servidor HTTP de BlueBubbles, ruta de webhook, contraseña REST ni entorno de ejecución del plugin BlueBubbles.
- OpenClaw lee y supervisa Mensajes mediante `imsg` en el Mac donde se ha iniciado sesión en Messages.app.
- El envío, la recepción, el historial y los archivos multimedia básicos utilizan las superficies normales de `imsg` y los permisos de macOS.
- Las acciones avanzadas (respuestas en hilos, tapbacks, edición, anulación de envío, efectos, confirmaciones de lectura, indicadores de escritura y gestión de grupos) necesitan el puente de API privada: ejecute `imsg launch`, que requiere desactivar SIP.
- Los gateways de Linux y Windows pueden seguir usando iMessage si `channels.imessage.cliPath` apunta a un contenedor SSH que ejecute `imsg` en el Mac con la sesión iniciada.

## Qué hacer

1. Instale y verifique `imsg` en el Mac con Mensajes:

   ```bash
   brew install steipete/tap/imsg
   imsg --version
   imsg chats --limit 3
   imsg rpc --help
   ```

2. Conceda permisos de acceso total al disco y automatización al contexto del proceso que ejecuta `imsg` y OpenClaw.

3. Convierta la configuración anterior:

   ```json5
   {
     channels: {
       imessage: {
         enabled: true,
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"],
         groupPolicy: "allowlist",
         groupAllowFrom: ["+15555550123"],
         groups: {
           "*": { requireMention: true },
         },
         includeAttachments: true,
       },
     },
   }
   ```

4. Reinicie el Gateway y verifique:

   ```bash
   openclaw channels status --probe
   ```

5. Pruebe los mensajes directos, los grupos, los archivos adjuntos y cualquier acción de API privada de la que dependa antes de eliminar el antiguo servidor BlueBubbles.

## Notas de migración

- `channels.bluebubbles.serverUrl` y `channels.bluebubbles.password` no tienen equivalente en iMessage; no hay ningún servidor al que conectarse o en el que autenticarse.
- `allowFrom`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit` y `actions.*` conservan su significado en `channels.imessage`.
- `channels.imessage.includeAttachments` sigue desactivado de forma predeterminada. Configúrelo explícitamente si espera que las fotos, notas de voz, vídeos o archivos entrantes lleguen al agente.
- Con `groupPolicy: "allowlist"`, copie el bloque `groups` anterior, incluida cualquier entrada comodín `"*"`. Las listas de remitentes permitidos de los grupos y el registro de grupos son controles independientes; un bloque `groups` con entradas pero sin ningún `chat_id` coincidente (o sin `"*"`) descarta el mensaje durante la ejecución, y un bloque `groups` vacío registra una advertencia al iniciarse, aunque el filtrado de remitentes siga permitiendo el paso de los mensajes.
- Las vinculaciones ACP con `match.channel: "bluebubbles"` deben cambiar a `"imessage"`.
- Las antiguas claves de sesión de BlueBubbles no se convierten en claves de sesión de iMessage. Las aprobaciones de emparejamiento se basan en los identificadores de los remitentes, por lo que las entradas `allowFrom` copiadas siguen funcionando, pero el historial de conversaciones asociado a las claves de sesión de BlueBubbles no se transfiere.

## Véase también

- [Migración desde BlueBubbles](/es/channels/imessage-from-bluebubbles)
- [iMessage](/es/channels/imessage)
- [Referencia de configuración: iMessage](/es/gateway/config-channels#imessage)
