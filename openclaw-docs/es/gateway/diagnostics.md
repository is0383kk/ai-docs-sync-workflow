---
read_when:
    - Preparación de un informe de error o una solicitud de asistencia
    - Depuración de cierres inesperados y reinicios del Gateway, presión de memoria o cargas útiles demasiado grandes
    - Revisión de qué datos de diagnóstico se registran o se censuran
summary: Crear paquetes de diagnóstico del Gateway que se puedan compartir para informes de errores
title: Exportación de diagnósticos
x-i18n:
    generated_at: "2026-07-26T05:10:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw puede crear un `.zip` de diagnóstico local para informes de errores: estado
y salud del Gateway, registros, estructura de configuración y eventos recientes de estabilidad sin cargas útiles, todo ello sanitizado.

Trate los paquetes de diagnóstico como secretos hasta que se revisen. Las cargas útiles y las credenciales
se ocultan por diseño, pero el paquete sigue resumiendo los registros locales del Gateway y
el estado del entorno de ejecución a nivel del host.

## Inicio rápido

```bash
openclaw gateway diagnostics export
```

Imprime la ruta del archivo zip generado. Para elegir una ruta de salida:

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

Para automatización:

```bash
openclaw gateway diagnostics export --json
```

## Comando de chat

Los propietarios pueden ejecutar `/diagnostics [note]` en cualquier conversación para solicitar una exportación
local del Gateway como un único informe de soporte que se puede copiar y pegar:

1. Envíe `/diagnostics`, opcionalmente con una nota breve (`/diagnostics bad tool choice`).
2. OpenClaw envía un preámbulo y solicita una única aprobación explícita de ejecución, que ejecuta
   `openclaw gateway diagnostics export --json`. No apruebe los diagnósticos mediante
   una regla de autorización total.
3. Tras la aprobación, OpenClaw responde con la ruta del paquete local, un resumen
   del manifiesto, notas de privacidad e identificadores de sesión relevantes.

En los chats grupales, un propietario puede seguir ejecutando `/diagnostics`, pero OpenClaw envía de forma
privada al propietario el resultado de la exportación, las solicitudes de aprobación y el desglose de sesiones/hilos
de Codex. El grupo solo ve un aviso breve de que los diagnósticos se enviaron
de forma privada. Si no existe una ruta privada hacia el propietario, el comando falla de forma segura y solicita
al propietario que lo ejecute desde un mensaje directo.

Cuando la sesión activa utiliza el arnés nativo de OpenAI Codex, la misma aprobación de
ejecución también cubre el envío de comentarios a OpenAI para los hilos de Codex que OpenClaw
conoce. Ese envío es independiente del archivo zip local del Gateway y solo
se produce en sesiones del arnés de Codex. La solicitud de aprobación indica que la aprobación
también envía comentarios de Codex, sin enumerar los identificadores de sesión ni de hilo de Codex. Tras
la aprobación, la respuesta enumera los canales, los identificadores de sesión de OpenClaw, los identificadores de hilo de Codex y
los comandos locales para reanudar los hilos que se enviaron a OpenAI. Rechazar o
ignorar la aprobación omite la exportación, el envío de comentarios de Codex y la
lista de identificadores de Codex.

Esto acorta el ciclo de depuración de Codex: detecte un comportamiento incorrecto en un canal,
ejecute `/diagnostics`, apruebe una vez, comparta el informe y, a continuación, ejecute localmente el comando
`codex resume <thread-id>` impreso si desea inspeccionar el hilo
personalmente. Consulte [arnés de Codex](/es/plugins/codex-harness#inspect-codex-threads-locally).

## Contenido de la exportación

- `summary.md`: resumen legible para el equipo de soporte.
- `diagnostics.json`: resumen procesable por máquinas de la configuración, los registros, el estado, la salud
  y los datos de estabilidad.
- `manifest.json`: metadatos de la exportación y lista de archivos.
- Estructura de configuración sanitizada y detalles de configuración no secretos.
- Resúmenes de registros sanitizados y líneas de registro recientes con datos ocultos.
- Instantáneas del estado y la salud del Gateway obtenidas con el mejor esfuerzo posible.
- `stability/latest.json`: paquete de estabilidad persistente más reciente, cuando esté disponible.

La exportación sigue siendo útil cuando el Gateway no funciona correctamente: si fallan las solicitudes
de estado o salud, se siguen recopilando los registros locales, la estructura de configuración y el paquete de estabilidad
más reciente cuando están disponibles.

## Modelo de privacidad

Se conservan: nombres de subsistemas, identificadores de plugins, identificadores de proveedores, identificadores de canales, modos
configurados, códigos de estado, duraciones, recuentos de bytes, estado de las colas, lecturas de memoria,
metadatos de registros sanitizados, mensajes operativos con datos ocultos, estructura de configuración y
ajustes de funciones no secretos.

Se omiten o se ocultan: texto de chats, prompts, instrucciones, cuerpos de webhooks, salidas de
herramientas, credenciales, claves de API, tokens, cookies, valores secretos, cuerpos sin procesar de
solicitudes/respuestas, identificadores de cuentas, identificadores de mensajes, identificadores de sesión sin procesar,
nombres de host y nombres de usuario locales.

Cuando un mensaje de registro parece contener texto de una carga útil de usuario, chat, prompt o herramienta, la
exportación solo conserva que se omitió un mensaje y su recuento de bytes.

## Registrador de estabilidad

De forma predeterminada, el Gateway registra un flujo de estabilidad acotado y sin cargas útiles cuando
los diagnósticos están habilitados. Captura datos operativos, no contenido.

El mismo Heartbeat también muestrea la capacidad de respuesta cuando el bucle de eventos o la CPU parecen
saturados y emite eventos `diagnostic.liveness.warning` con el retraso del bucle de eventos,
la utilización del bucle de eventos, la proporción de núcleos de CPU, los recuentos de sesiones activas/en espera/en cola,
la fase actual de inicio/ejecución (cuando se conoce), los intervalos de fases recientes y
etiquetas de trabajo acotadas. Estos se convierten en líneas de registro del Gateway de nivel `warn` solo cuando
hay trabajo en espera o en cola, o cuando el trabajo activo coincide con un retraso sostenido del bucle de eventos;
de lo contrario, se registran en `debug`. Las muestras de capacidad de respuesta en reposo se siguen registrando
como eventos de diagnóstico, pero nunca se elevan por sí mismas al nivel de advertencia.

Las fases de inicio emiten eventos `diagnostic.phase.completed` con tiempos de reloj de pared y
de CPU. Los diagnósticos de ejecuciones integradas bloqueadas marcan `terminalProgressStale=true`
cuando el último progreso del puente parecía terminal (por ejemplo, un elemento de respuesta
sin procesar o un evento de finalización de respuesta), pero el Gateway todavía considera activa la
ejecución integrada.

Para inspeccionar el registrador en directo:

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

Para inspeccionar el paquete persistente más reciente después de una salida fatal, un tiempo de espera agotado durante el apagado o
un fallo de inicio tras un reinicio:

```bash
openclaw gateway stability --bundle latest
```

Para crear un archivo zip de diagnóstico a partir del paquete persistente más reciente:

```bash
openclaw gateway stability --bundle latest --export
```

Los paquetes persistentes se almacenan en `~/.openclaw/logs/stability/` cuando existen eventos.

## Opciones útiles

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| Indicador               | Valor predeterminado                                                            | Descripción                                                   |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | Escribe en una ruta específica de archivo zip (o directorio). |
| `--log-lines <count>`   | `5000`                                                                        | Número máximo de líneas de registro sanitizadas que se incluirán. |
| `--log-bytes <bytes>`   | `1000000`                                                                     | Número máximo de bytes de registro que se inspeccionarán.     |
| `--url <url>`           | -                                                                             | URL de WebSocket del Gateway para instantáneas de estado/salud. |
| `--token <token>`       | -                                                                             | Token del Gateway para instantáneas de estado/salud.          |
| `--password <password>` | -                                                                             | Contraseña del Gateway para instantáneas de estado/salud.     |
| `--timeout <ms>`        | `3000`                                                                        | Tiempo de espera de la instantánea de estado/salud.            |
| `--no-stability-bundle` | desactivado                                                                  | Omite la búsqueda de paquetes de estabilidad persistentes.    |
| `--json`                | desactivado                                                                  | Imprime metadatos de exportación procesables por máquinas.     |

## Desactivar los diagnósticos

Los diagnósticos están habilitados de forma predeterminada. Para desactivar el registrador de estabilidad y
la recopilación de eventos de diagnóstico:

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

Desactivar los diagnósticos reduce el nivel de detalle de los informes de errores, pero no afecta al registro
normal del Gateway.

Los eventos de presión de memoria registran datos de RSS, heap, umbral y crecimiento
(`rss_threshold`, `heap_threshold`, `rss_growth`) sin realizar un análisis
del sistema de archivos ni escribir una instantánea previa a una falta de memoria.

## Contenido relacionado

- [Comprobaciones de estado](/es/gateway/health)
- [CLI del Gateway](/es/cli/gateway#gateway-diagnostics-export)
- [Protocolo del Gateway](/es/gateway/protocol#rpc-method-families)
- [Registro](/es/logging)
- [Exportación de OpenTelemetry](/es/gateway/opentelemetry) - flujo independiente para transmitir diagnósticos a un recopilador
