---
read_when:
    - El transporte del canal indica que está conectado, pero las respuestas fallan
    - Se necesitan comprobaciones específicas del canal antes de consultar la documentación detallada del proveedor
summary: Solución rápida de problemas a nivel de canal con patrones de error y correcciones específicos de cada canal
title: Solución de problemas de canales
x-i18n:
    generated_at: "2026-07-26T05:32:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3891595e4b5aca9de7997a6e908fa1c9246579032bfdfa1656a6992d644c3ecc
    source_path: channels/troubleshooting.md
    workflow: 16
---

Use esta página cuando un canal se conecte, pero su comportamiento sea incorrecto.

## Secuencia de comandos

Primero, ejecute estos comandos en orden:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Estado de referencia correcto:

- `Runtime: running`
- `Connectivity probe: ok`
- `Capability: read-only`, `write-capable` o `admin-capable`
- La sonda del canal muestra que el transporte está conectado y, cuando se admite, `works` o `audit ok`

## Después de una actualización

Use este procedimiento cuando Telegram, iMessage, las configuraciones de la época de BlueBubbles u otro canal de Plugin desaparezcan
después de actualizar.

```bash
openclaw status --all
openclaw doctor --fix
openclaw gateway restart
openclaw status --all
```

Busque `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` en `openclaw
status --all`. Esto significa que el canal está configurado, pero la configuración o carga del Plugin encontró un árbol
de dependencias dañado en lugar de registrar el canal. `openclaw doctor --fix` elimina los enlaces simbólicos obsoletos
de las dependencias del entorno de ejecución del Plugin y las sombras de autenticación obsoletas; después, `openclaw gateway restart` vuelve a cargar
el estado limpio.

## WhatsApp

### Indicadores de fallos de WhatsApp

| Síntoma                             | Comprobación más rápida                                       | Solución                                                                                                                              |
| ----------------------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Está conectado, pero no responde a los mensajes directos         | `openclaw pairing list whatsapp`                    | Apruebe al remitente o cambie la política o lista de permitidos de mensajes directos.                                                                                    |
| Se ignoran los mensajes de grupo              | Compruebe `requireMention` y los patrones de mención de la configuración | Mencione al bot o flexibilice la política de menciones para ese grupo.                                                                          |
| El inicio de sesión mediante QR agota el tiempo de espera con el código 408         | Compruebe las variables de entorno `HTTPS_PROXY` y `HTTP_PROXY` del Gateway      | Configure un proxy accesible; use `NO_PROXY` solo para omisiones.                                                                         |
| Bucles aleatorios de desconexión y nuevo inicio de sesión     | `openclaw channels status --probe` y los registros           | Las reconexiones recientes se marcan incluso si actualmente existe conexión; observe los registros, reinicie el Gateway y vuelva a vincular la cuenta si la inestabilidad continúa. |
| Bucle `status=408 Request Time-out`  | Sonda, registros, doctor y, después, estado del Gateway            | Corrija primero la conectividad y la temporización del host; haga una copia de seguridad de la autenticación y vuelva a vincular la cuenta si el bucle persiste.                                   |
| Las respuestas llegan con segundos o minutos de retraso | `openclaw doctor --fix`                             | Doctor detiene los clientes TUI locales obsoletos verificados cuando degradan el bucle de eventos del Gateway.                                    |

Solución de problemas completa: [Solución de problemas de WhatsApp](/es/channels/whatsapp#troubleshooting)

## Telegram

### Indicadores de fallos de Telegram

| Síntoma                              | Comprobación más rápida                                    | Solución                                                                                                                    |
| ------------------------------------ | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/start`, pero no hay un flujo de respuestas utilizable    | `openclaw pairing list telegram`                 | Apruebe el emparejamiento o cambie la política de mensajes directos.                                                                                   |
| El bot está en línea, pero el grupo permanece en silencio    | Verifique el requisito de mención y el modo de privacidad del bot  | Desactive el modo de privacidad para permitir la visibilidad en el grupo o mencione al bot.                                                              |
| Fallos de envío con errores de red    | Inspeccione los registros en busca de fallos en las llamadas a la API de Telegram      | Corrija el enrutamiento de DNS, IPv6 o proxy hacia `api.telegram.org`.                                                                      |
| El inicio informa de `getMe returned 401` | Compruebe el origen del token configurado                    | Vuelva a copiar o genere de nuevo el token de BotFather y actualice `botToken`, `tokenFile` o `TELEGRAM_BOT_TOKEN` de la cuenta predeterminada. |
| El sondeo se bloquea o tarda en reconectarse  | `openclaw logs --follow` para obtener diagnósticos del sondeo | Actualice; los bloqueos persistentes suelen indicar problemas con el proxy, DNS o IPv6.                                                            |
| `setMyCommands` se rechaza durante el inicio  | Inspeccione los registros en busca de `BOT_COMMANDS_TOO_MUCH`         | Reduzca los comandos de Plugin, Skills o personalizados de Telegram, o desactive los menús nativos.                                                  |
| Tras actualizar, la lista de permitidos le bloquea    | `openclaw security audit` y las listas de permitidos de la configuración  | Ejecute `openclaw doctor --fix` o sustituya `@username` por identificadores numéricos de remitentes.                                            |

Solución de problemas completa: [Solución de problemas de Telegram](/es/channels/telegram#troubleshooting)

## Discord

### Indicadores de fallos de Discord

| Síntoma                                   | Comprobación más rápida                                                                                                                | Solución                                                                                                                                                                                                                                                                   |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| El bot está en línea, pero no responde en el servidor           | `openclaw channels status --probe`                                                                                           | Permita el servidor o canal y verifique la intención de contenido de los mensajes.                                                                                                                                                                                                                |
| Se ignoran los mensajes de grupo                    | Compruebe en los registros si se descartan mensajes por el control de menciones                                                                                          | Mencione al bot o configure `requireMention: false` para el servidor o canal.                                                                                                                                                                                                             |
| Hay actividad de escritura o uso de tokens, pero no aparece ningún mensaje de Discord | Compruebe si se trata de un evento de sala ambiental o de una sala `message_tool` habilitada en la que el modelo omitió `message(action=send)` | Inspeccione el registro detallado del Gateway en busca de metadatos de la carga útil final suprimida, verifique `messages.groupChat.unmentionedInbound`, consulte [Eventos de salas ambientales](/es/channels/ambient-room-events) o mantenga `messages.groupChat.visibleReplies: "automatic"` para las solicitudes de grupo normales. |
| Faltan respuestas a mensajes directos                        | `openclaw pairing list discord`                                                                                              | Apruebe el emparejamiento de mensajes directos o ajuste su política.                                                                                                                                                                                                                               |

Solución de problemas completa: [Solución de problemas de Discord](/es/channels/discord#troubleshooting)

## Slack

### Indicadores de fallos de Slack

| Síntoma                                | Comprobación más rápida                             | Solución                                                                                                                                                  |
| -------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| El modo de socket está conectado, pero no hay respuestas | `openclaw channels status --probe`        | Verifique el token de la aplicación, el token del bot y los ámbitos necesarios; compruebe si aparece `botTokenStatus` o `appTokenStatus = configured_unavailable` en las configuraciones basadas en SecretRef. |
| Los mensajes directos están bloqueados                            | `openclaw pairing list slack`             | Apruebe el emparejamiento o flexibilice la política de mensajes directos.                                                                                                                  |
| Se ignora el mensaje del canal                | Compruebe `groupPolicy` y la lista de canales permitidos | Permita el canal o cambie la política a `open`.                                                                                                        |

Solución de problemas completa: [Solución de problemas de Slack](/es/channels/slack#troubleshooting)

## iMessage

### Indicadores de fallos de iMessage

| Síntoma                              | Comprobación más rápida                                           | Solución                                                                   |
| ------------------------------------ | ------------------------------------------------------- | --------------------------------------------------------------------- |
| Falta `imsg` o falla en sistemas distintos de macOS | `openclaw channels status --probe --channel imessage`   | Ejecute OpenClaw en el Mac con Mensajes o use un contenedor SSH para `cliPath`. |
| Se puede enviar, pero no recibir en macOS     | Compruebe los permisos de privacidad de macOS para la automatización de Mensajes | Vuelva a conceder los permisos de TCC y reinicie el proceso del canal.                 |
| El remitente de mensajes directos está bloqueado                    | `openclaw pairing list imessage`                        | Apruebe el emparejamiento o actualice la lista de permitidos.                                  |

Solución de problemas completa: [Solución de problemas de iMessage](/es/channels/imessage#troubleshooting)

## Signal

### Indicadores de fallos de Signal

| Síntoma                         | Comprobación más rápida                              | Solución                                                      |
| ------------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| Se puede acceder al demonio, pero el bot permanece en silencio | `openclaw channels status --probe`         | Verifique la URL o cuenta del demonio `signal-cli` y el modo de recepción. |
| Los mensajes directos están bloqueados                      | `openclaw pairing list signal`             | Apruebe al remitente o ajuste la política de mensajes directos.                      |
| Las respuestas de grupo no se activan    | Compruebe la lista de grupos permitidos y los patrones de mención | Añada al remitente o grupo, o flexibilice el control.                       |

Solución de problemas completa: [Solución de problemas de Signal](/es/channels/signal#troubleshooting)

## Bot de QQ

### Indicadores de fallos del bot de QQ

| Síntoma                         | Comprobación más rápida                               | Solución                                                             |
| ------------------------------- | ------------------------------------------- | --------------------------------------------------------------- |
| El bot responde «se ha ido a Marte»      | Verifique `appId` y `clientSecret` en la configuración | Configure las credenciales o reinicie el Gateway.                         |
| No hay mensajes entrantes             | `openclaw channels status --probe`          | Verifique las credenciales en QQ Open Platform.                     |
| La voz no se transcribe           | Compruebe la configuración del proveedor de STT                   | Configure `channels.qqbot.stt` o `tools.media.audio`.          |
| Los mensajes proactivos no llegan | Compruebe los requisitos de interacción de la plataforma QQ  | QQ puede bloquear los mensajes iniciados por el bot si no ha habido una interacción reciente. |

Solución de problemas completa: [Solución de problemas del bot de QQ](/es/channels/qqbot#troubleshooting)

## Matrix

### Indicadores de fallos de Matrix

| Síntoma                             | Comprobación más rápida                | Solución                                                                  |
| ----------------------------------- | -------------------------------------- | ------------------------------------------------------------------------- |
| Se inició sesión, pero se ignoran los mensajes de la sala | `openclaw channels status --probe`     | Compruebe `groupPolicy`, la lista de permitidos de la sala y el requisito de mención.                  |
| Los mensajes directos no se procesan | `openclaw pairing list matrix`         | Apruebe al remitente o ajuste la política de mensajes directos.                                       |
| Las salas cifradas fallan           | `openclaw matrix verify status`        | Vuelva a verificar el dispositivo y, después, compruebe `openclaw matrix verify backup status`.  |
| La restauración de la copia de seguridad está pendiente o no funciona | `openclaw matrix verify backup status` | Ejecute `openclaw matrix verify backup restore` o repita el proceso con una clave de recuperación. |
| La firma cruzada o el arranque parecen incorrectos | `openclaw matrix verify bootstrap`     | Repare de una sola vez el almacenamiento de secretos, la firma cruzada y el estado de la copia de seguridad.       |

Configuración completa: [Matrix](/es/channels/matrix)

## Temas relacionados

- [Emparejamiento](/es/channels/pairing)
- [Enrutamiento de canales](/es/channels/channel-routing)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
