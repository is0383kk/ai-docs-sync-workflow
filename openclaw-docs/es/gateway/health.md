---
read_when:
    - Diagnóstico de la conectividad de los canales o del estado del Gateway
    - Descripción de los comandos y las opciones de la CLI para las comprobaciones de estado
summary: Comandos de comprobación de estado y monitorización del estado del Gateway
title: Comprobaciones de estado
x-i18n:
    generated_at: "2026-07-26T05:07:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59a7fbfb7fb86be7dbd3a03f96c7328c2bc8cc851230c0bdd1b1b750b3014be4
    source_path: gateway/health.md
    workflow: 16
---

Guía breve para verificar la conectividad de los canales sin hacer suposiciones.

## Comprobaciones rápidas

- `openclaw status` - resumen local: accesibilidad/modo del Gateway, aviso de actualización, antigüedad de la autenticación del canal vinculado, sesiones y actividad reciente.
- `openclaw status --all` - diagnóstico local completo (solo lectura, con color y seguro para pegar al depurar).
- `openclaw status --deep` - solicita al Gateway en ejecución un sondeo en vivo (`health` con `probe:true`), incluidos los sondeos de canales por cuenta cuando se admiten.
- `openclaw status --usage` - muestra instantáneas de uso/cuota del proveedor de modelos.
- `openclaw health` - solicita al Gateway en ejecución su instantánea de estado (solo WS; sin conexiones directas a los canales desde la CLI).
- `openclaw health --verbose` (alias `--debug`) - fuerza un sondeo de estado en vivo y muestra los detalles de conexión del Gateway.
- `openclaw health --json` - salida de la instantánea de estado legible por máquina.
- Envía `/status` como comando de chat independiente en cualquier canal para obtener una respuesta de estado sin invocar al agente.
- Registros: ejecuta `openclaw logs --follow` (o `openclaw --profile <profile> logs --follow`) y filtra por `web-heartbeat`, `web-reconnect`, `web-auto-reply`, `web-inbound`.

Para Discord y otros proveedores de chat, las filas de sesión no indican que una conexión esté activa.
`openclaw sessions`, el `sessions.list` del Gateway y la herramienta `sessions_list` del agente
leen el estado almacenado de las conversaciones. Un proveedor puede volver a conectarse y mostrar un estado
de canal correcto antes de que se materialice una nueva fila de sesión. Usa los comandos de estado
del canal y de estado general anteriores para comprobar la conectividad en vivo.

## Diagnóstico exhaustivo

- Credenciales en disco: `ls -l ~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (la fecha de modificación debe ser reciente).
- Almacén de sesiones: `ls -l ~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. El recuento y los destinatarios recientes se muestran mediante `status`.
- Flujo para volver a vincular: `openclaw channels logout && openclaw channels login --verbose` cuando aparezcan códigos de estado 409-515 o `loggedOut` en los registros. El flujo de inicio de sesión mediante QR se reinicia automáticamente una vez si se produce el estado 515 después del emparejamiento.
- El diagnóstico está habilitado de forma predeterminada (`diagnostics.enabled: false` lo deshabilita). Los eventos de memoria registran los recuentos de bytes de RSS/montículo y la presión por umbral/crecimiento. Las advertencias de actividad registran el retraso/uso del bucle de eventos, la proporción de núcleos de CPU y los recuentos de sesiones activas/en espera/en cola cuando el proceso está en ejecución pero saturado. Los eventos de carga útil sobredimensionada registran qué se rechazó/truncó/dividió, además de sus tamaños y límites, pero nunca el texto de los mensajes, el contenido de los archivos adjuntos, los cuerpos de Webhook, los cuerpos sin procesar de solicitudes/respuestas, los tokens, las cookies ni los valores secretos.
- El mismo Heartbeat controla el registrador de estabilidad acotado: `openclaw gateway stability` (o la RPC `diagnostics.stability` del Gateway). Las salidas fatales del Gateway, los tiempos de espera agotados durante el cierre y los fallos de inicio tras un reinicio conservan la instantánea más reciente en `~/.openclaw/logs/stability/`. Examina el paquete más reciente con `openclaw gateway stability --bundle latest`.
- Para los informes de errores, ejecuta `openclaw gateway diagnostics export` y adjunta el archivo zip generado: un resumen en Markdown, el paquete de estabilidad más reciente, metadatos de registro desinfectados, instantáneas desinfectadas del estado general y del estado del Gateway y la estructura de la configuración. El texto de los chats, los cuerpos de Webhook, las salidas de las herramientas, las credenciales, las cookies, los identificadores de cuentas/mensajes y los valores secretos se omiten o se censuran. Consulta [Exportación de diagnósticos](/es/gateway/diagnostics).

## Configuración del monitor de estado

- `channels.<provider>.healthMonitor.enabled`: deshabilita los reinicios del monitor de estado para un canal específico mientras mantiene habilitada la supervisión global.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: anulación para varias cuentas que prevalece sobre la configuración del canal.
- Estas anulaciones por canal se aplican a los canales integrados que las ofrecen actualmente: Discord, Google Chat, iMessage, IRC, Microsoft Teams, Signal, Slack, Telegram y WhatsApp.

## Supervisión del tiempo de actividad

Los servicios externos de supervisión del tiempo de actividad deben usar el endpoint específico `/health`, no `/v1/chat/completions`.

- **SÍ debe usarse:** `GET /health` - respuesta instantánea, no se crea ninguna sesión, no se llama al LLM y devuelve `{"ok":true,"status":"live"}`
- **NO debe usarse:** `/v1/chat/completions` para las comprobaciones de estado; cada solicitud crea una sesión completa del agente con una instantánea de Skills, ensamblaje del contexto y llamadas al LLM

Cuando no se proporciona el encabezado `x-openclaw-session-key` ni el campo `user`, `/v1/chat/completions` genera una nueva sesión aleatoria para cada solicitud. Los servicios de supervisión que realizan una consulta cada 15 minutos crean unas 96 sesiones/día, cada una de las cuales consume 4-22KB. Con el tiempo, esto provoca el crecimiento excesivo del almacén de sesiones y puede ocasionar que se desborde la ventana de contexto.

### Ejemplos de configuración de servicios de supervisión

- **BetterStack:** Establece la URL de comprobación de estado en `https://<your-gateway-host>:<port>/health`
- **UptimeRobot:** Añade un nuevo monitor HTTP con la URL `https://<your-gateway-host>:<port>/health`
- **Genérico:** Cualquier solicitud HTTP GET a `/health` devuelve 200 con `{"ok":true}` cuando el Gateway funciona correctamente

## Cuando algo falla

- `logged out` o estado 409-515 -> vuelve a vincular con `openclaw channels logout` y después con `openclaw channels login`.
- Gateway inaccesible -> inícialo: `openclaw gateway --port 18789` (usa `--force` si el puerto está ocupado).
- No se reciben mensajes -> confirma que el teléfono vinculado esté conectado y que el remitente esté permitido (`channels.whatsapp.allowFrom`); para los chats grupales, asegúrate de que coincidan la lista de permitidos y las reglas de menciones (`channels.whatsapp.groups`, `agents.entries.*.groupChat.mentionPatterns`).

## Comando «health» específico

`openclaw health` solicita al Gateway en ejecución su instantánea de estado (sin conexiones directas
a los canales desde la CLI). De forma predeterminada, devuelve una instantánea reciente almacenada en caché del Gateway y este
actualiza esa caché en segundo plano; `--verbose` fuerza en su lugar un sondeo en vivo.
El comando informa de la antigüedad de las credenciales/autenticación vinculadas cuando están disponibles, los resúmenes de los sondeos por canal,
el resumen del almacén de sesiones y la duración del sondeo. Finaliza con un código distinto de cero si el Gateway está
inaccesible o si el sondeo falla o agota el tiempo de espera.

Opciones:

- `--json`: salida JSON legible por máquina
- `--timeout <ms>`: anula el tiempo de espera predeterminado de 10s del sondeo
- `--verbose`: fuerza un sondeo en vivo y muestra los detalles de conexión del Gateway
- `--debug`: alias de `--verbose`

La instantánea de estado incluye: `ok` (booleano), `ts` (marca temporal), `durationMs` (duración del sondeo), estado por canal, disponibilidad del agente y resumen del almacén de sesiones.

## Contenido relacionado

- [Guía operativa del Gateway](/es/gateway)
- [Exportación de diagnósticos](/es/gateway/diagnostics)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
