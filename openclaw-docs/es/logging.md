---
read_when:
    - Necesita una introducción al registro de OpenClaw apta para principiantes
    - Se desea configurar los niveles de registro, los formatos o la censura
    - Se están solucionando problemas y es necesario encontrar los registros rápidamente
summary: Registros de archivos, salida de la consola, seguimiento de registros mediante la CLI y pestaña Registros de la interfaz de control
title: Registro
x-i18n:
    generated_at: "2026-07-26T05:10:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c9303c4bc9c0797ca9c5775a281dce95229661b61d710425b2f7bec182b2e75
    source_path: logging.md
    workflow: 16
---

OpenClaw tiene dos superficies principales de registro:

- **Registros de archivo** (líneas JSON) escritos por el Gateway.
- **Salida de consola** en el terminal que ejecuta el Gateway.

La pestaña **Registros** de la interfaz de control sigue en tiempo real el archivo de registro del Gateway. Esta página explica dónde
se encuentran los registros, cómo leerlos y cómo configurar sus niveles y formatos.

## Dónde se encuentran los registros

De forma predeterminada, el Gateway escribe un archivo de registro rotativo por día. El perfil predeterminado
conserva la ruta histórica:

`/tmp/openclaw/openclaw-YYYY-MM-DD.log`

Los perfiles con nombre usan un nombre de archivo calificado por el perfil en el mismo directorio:

`/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`

El segmento de perfil del nombre de archivo está en minúsculas y se limita a letras, números y
guiones. Los nombres sencillos en minúsculas siguen siendo legibles, por lo que la abreviatura `--dev` escribe
`openclaw-dev-YYYY-MM-DD.log`. Las mayúsculas, los guiones bajos y los guiones literales utilizan un
escape de guion reversible para que distintos nombres de perfil nunca compartan un archivo de registro.
Los valores demasiado grandes establecidos directamente mediante el entorno usan un sufijo hash acotado
para mantenerse dentro de los límites de longitud de los nombres de archivo del sistema de archivos. Un valor explícito de `logging.file` sustituye
estos valores predeterminados.

La fecha utiliza la zona horaria local del host del Gateway. Cuando `/tmp/openclaw` no es seguro
o no está disponible (y siempre en Windows), OpenClaw utiliza un directorio
`openclaw-<uid>` específico del usuario dentro del directorio temporal del sistema operativo. Los archivos de registro con fecha se
eliminan después de 24 horas.

Cada archivo rota cuando la siguiente escritura superaría `logging.maxFileBytes`
(valor predeterminado: 100 MB). OpenClaw conserva hasta cinco archivos numerados junto al
archivo activo, como `openclaw-YYYY-MM-DD.1.log` o
`openclaw-dev-YYYY-MM-DD.1.log`, y continúa escribiendo en un nuevo registro activo en lugar
de suprimir los diagnósticos.

La ruta se puede sustituir en `~/.openclaw/openclaw.json`:

```json
{
  "logging": {
    "file": "/path/to/openclaw.log"
  }
}
```

## Cómo leer los registros

### CLI: seguimiento en tiempo real (recomendado)

Siga en tiempo real el archivo de registro del Gateway mediante RPC:

```bash
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
```

El selector de perfil raíz resuelve el mismo archivo específico del perfil que utiliza el
Gateway, incluidas las lecturas alternativas de la CLI cuando la RPC local no está disponible.

Opciones:

| Indicador           | Predeterminado | Comportamiento                                                                         |
| ------------------- | -------------- | -------------------------------------------------------------------------------------- |
| `--follow`          | desactivado    | Continúa el seguimiento; vuelve a conectarse con espera progresiva al desconectarse    |
| `--limit <n>`       | `200`    | Máximo de líneas por recuperación                                                      |
| `--max-bytes <n>`   | `250000` | Máximo de bytes que se leen por recuperación                                           |
| `--interval <ms>`   | `1000`   | Intervalo de consulta durante el seguimiento                                           |
| `--json`            | desactivado    | JSON delimitado por líneas (un evento por línea)                                       |
| `--plain`           | desactivado    | Fuerza el texto sin formato en sesiones TTY                                            |
| `--no-color`        | —              | Desactiva los colores ANSI                                                             |
| `--utc`             | desactivado    | Representa las marcas de tiempo en UTC (la hora local es la predeterminada)             |
| `--local-time`      | desactivado    | Grafía de compatibilidad aceptada para el valor predeterminado de hora local; no tiene ningún otro efecto |
| `--url` / `--token` | —              | Indicadores estándar de RPC del Gateway                                                |
| `--timeout <ms>`    | `30000`  | Tiempo de espera de RPC del Gateway                                                    |
| `--expect-final`    | desactivado    | Indicador de espera de la respuesta final de RPC respaldada por un agente (se acepta aquí mediante la capa de cliente compartida) |

Modos de salida:

- **Sesiones TTY**: líneas de registro estructuradas, legibles y coloreadas.
- **Sesiones que no son TTY**: texto sin formato.

Cuando se proporciona un valor explícito de `--url`, la CLI no aplica automáticamente las credenciales de la configuración ni
del entorno; incluya `--token` manualmente o la llamada fallará con
`gateway url override requires explicit credentials`.

En modo JSON, la CLI emite objetos etiquetados con `type`:

- `meta`: metadatos del flujo (archivo, origen, tipo de origen, servicio, cursor, tamaño)
- `log`: entrada de registro analizada
- `notice`: indicaciones de truncamiento o rotación
- `raw`: línea de registro sin analizar
- `error`: fallos de conexión con el Gateway (escritos en stderr)

Si el Gateway de bucle invertido local implícito solicita emparejamiento, se cierra durante la conexión
o agota el tiempo de espera antes de que `logs.tail` responda, `openclaw logs` recurre automáticamente al
archivo de registro configurado del Gateway. Los destinos explícitos de `--url` no utilizan
esta alternativa. `openclaw logs --follow` es más estricto: en Linux utiliza el diario del Gateway
de systemd del usuario activo por PID cuando está disponible y, de lo contrario, reintenta la conexión con el
Gateway activo con espera progresiva en lugar de seguir un archivo paralelo que podría estar
obsoleto.

Si no se puede acceder al Gateway, la CLI muestra una breve indicación para ejecutar:

```bash
openclaw doctor
```

### Interfaz de control (web)

La pestaña **Registros** de la interfaz de control sigue en tiempo real el mismo archivo mediante `logs.tail`.
Consulte [Interfaz de control](/es/web/control-ui) para saber cómo abrirla.

### Registros exclusivos de canales

Para filtrar la actividad de los canales (WhatsApp/Telegram/etc.), utilice:

```bash
openclaw channels logs --channel whatsapp
```

`--channel` tiene como valor predeterminado `all`; también están disponibles `--lines <n>` (valor predeterminado: 200) y `--json`.

## Formatos de registro

### Registros de archivo (JSONL)

Cada línea del archivo de registro es un objeto JSON. La CLI y la interfaz de control analizan estas
entradas para representar una salida estructurada (hora, nivel, subsistema, mensaje).

Los registros JSONL del archivo también incluyen campos de nivel superior filtrables por máquina cuando
están disponibles:

- `hostname`: nombre del host del Gateway.
- `message`: texto aplanado del mensaje de registro para búsquedas de texto completo.
- `agent_id`: identificador del agente activo cuando la llamada de registro contiene contexto del agente.
- `session_id`: identificador o clave de la sesión activa cuando la llamada de registro contiene contexto de sesión.
- `channel`: canal activo cuando la llamada de registro contiene contexto del canal.

OpenClaw conserva los argumentos estructurados originales del registro junto con estos campos
para que sigan funcionando los analizadores existentes que leen claves numeradas de argumentos de tslog.

La actividad de conversación, voz en tiempo real y salas administradas emite registros acotados del ciclo de vida
mediante esta misma pipeline de registros de archivo. Estos registros incluyen el tipo de evento,
el modo, el transporte, el proveedor y las mediciones de tamaño y tiempo cuando están disponibles, pero omiten
el texto de la transcripción, las cargas de audio, los identificadores de turno, los identificadores de llamada y los identificadores de elementos del proveedor.

### Salida de consola

Los registros de consola **detectan TTY** y se formatean para facilitar la lectura:

- Prefijos de subsistemas (p. ej., `gateway/channels/whatsapp`)
- Colores según el nivel (información/advertencia/error)
- Modo compacto o JSON opcional

El formato de la consola se controla mediante `logging.consoleStyle`.

### Registros WebSocket del Gateway

`openclaw gateway` también ofrece registro del protocolo WebSocket para el tráfico RPC:

- modo normal: solo resultados relevantes (errores, errores de análisis, llamadas lentas)
- `--verbose`: todo el tráfico de solicitudes y respuestas
- `--ws-log auto|compact|full`: selecciona el estilo de representación detallada
- `--compact`: alias de `--ws-log compact`

Ejemplos:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## Configuración de los registros

Toda la configuración de los registros se encuentra bajo `logging` en `~/.openclaw/openclaw.json`.

```json
{
  "logging": {
    "level": "info",
    "file": "/path/to/openclaw.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### Niveles de registro

Niveles: `silent`, `fatal`, `error`, `warn`, `info`, `debug`, `trace`.

- `logging.level`: nivel de los **registros de archivo** (JSONL) (valor predeterminado: `info`).
- `logging.consoleLevel`: nivel de detalle de la **consola**.

Ambos se pueden sustituir mediante la variable de entorno **`OPENCLAW_LOG_LEVEL`** (p. ej., `OPENCLAW_LOG_LEVEL=debug`). La variable de entorno tiene prioridad sobre el archivo de configuración, por lo que se puede aumentar el nivel de detalle para una sola ejecución sin editar `openclaw.json`. También se puede proporcionar la opción global de la CLI **`--log-level <level>`** (por ejemplo, `openclaw --log-level debug gateway run`), que sustituye la variable de entorno para ese comando.

`--verbose` solo afecta a la salida de consola y al nivel de detalle del registro de WS; no cambia
los niveles de los registros de archivo.

### Diagnósticos específicos del transporte de modelos

Al depurar llamadas a proveedores, utilice indicadores de entorno específicos en lugar de elevar
todos los registros a `debug`:

```bash
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 openclaw gateway
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools OPENCLAW_DEBUG_SSE=events openclaw gateway
```

Indicadores disponibles:

- `OPENCLAW_DEBUG_MODEL_TRANSPORT=1`: emite el inicio de la solicitud, la respuesta de la recuperación, los encabezados
  del SDK, el primer evento de transmisión, la finalización del flujo y los errores de transporte con
  el nivel `info`.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=summary`: incluye un resumen acotado de la carga de la solicitud
  en los registros de solicitudes al modelo.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=tools`: incluye todos los nombres de herramientas visibles para el modelo en
  el resumen de la carga.
- `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`: incluye una instantánea JSON censurada y limitada de la
  carga. Utilícelo solo durante la depuración; los secretos se censuran, pero los prompts
  y el texto de los mensajes aún pueden estar presentes.
- `OPENCLAW_DEBUG_SSE=events`: emite los tiempos del primer evento y de finalización del flujo.
- `OPENCLAW_DEBUG_SSE=peek`: también emite las primeras cinco cargas censuradas de eventos SSE,
  limitadas por evento.
- `OPENCLAW_DEBUG_CODE_MODE=1`: emite diagnósticos de la superficie del modelo en modo de código,
  incluso cuando las herramientas nativas del proveedor están ocultas porque el modo de código controla la
  superficie de herramientas.

Estos indicadores registran mediante el sistema normal de registros de OpenClaw, por lo que `openclaw logs --follow`
y la pestaña Registros de la interfaz de control los muestran. Sin los indicadores, los mismos diagnósticos
siguen estando disponibles con el nivel `debug`.

Los metadatos de inicio y respuesta de `[model-fetch]` (proveedor, API, modelo, estado,
latencia y campos de solicitud como método, URL, tiempo de espera, proxy y política)
siempre se emiten con el nivel `info`, independientemente de
`OPENCLAW_DEBUG_MODEL_TRANSPORT`, para que la información básica sobre el transporte del modelo sea visible
sin indicadores de depuración.

### Correlación de trazas

Los registros de archivo son JSONL. Cuando una llamada de registro contiene un contexto válido de traza de diagnóstico,
OpenClaw escribe los campos de la traza como claves JSON de nivel superior (`traceId`, `spanId`,
`parentSpanId`, `traceFlags`) para que los procesadores externos de registros puedan correlacionar la línea
con los intervalos de OTEL y la propagación de `traceparent` del proveedor.

Las solicitudes HTTP del Gateway y las tramas WebSocket del Gateway establecen un ámbito interno de traza
de solicitudes. Los registros y eventos de diagnóstico emitidos dentro de ese ámbito asíncrono heredan
la traza de la solicitud cuando no proporcionan un contexto de traza explícito. Las trazas de ejecución de agentes y
de llamadas a modelos se convierten en descendientes de la traza de solicitud activa, por lo que los registros locales,
las instantáneas de diagnóstico, los intervalos de OTEL y los encabezados de confianza `traceparent` del proveedor pueden
vincularse mediante `traceId` sin registrar el contenido sin procesar de la solicitud o del modelo.

Los registros del ciclo de vida de las conversaciones también se envían a la exportación de registros de diagnostics-otel cuando
la exportación de registros de OpenTelemetry está activada, utilizando los mismos atributos acotados que los registros de
archivo. Configure `diagnostics.otel.logsExporter` para seleccionar OTLP, JSONL en stdout o
ambos destinos.

### Tamaño y tiempos de las llamadas a modelos

Los diagnósticos de las llamadas a modelos registran mediciones acotadas de solicitudes y respuestas sin
capturar el contenido sin procesar del prompt ni de la respuesta:

- `requestPayloadBytes`: tamaño en bytes UTF-8 de la carga útil final de la solicitud al modelo
- `responseStreamBytes`: tamaño en bytes UTF-8 del fragmento transmitido de la respuesta del modelo.
  Los eventos de alta frecuencia de texto, razonamiento y deltas de llamadas a herramientas
  solo cuentan los bytes incrementales de `delta` en lugar de las instantáneas completas de `partial`.
- `timeToFirstByteMs`: tiempo transcurrido antes del primer evento de respuesta transmitido
- `durationMs`: duración total de la llamada al modelo

Estos campos están disponibles para las instantáneas de diagnóstico, los hooks de Plugin de llamadas al modelo y
los spans y las métricas OTEL de llamadas al modelo cuando está habilitada la exportación de diagnósticos.

### Estilos de consola

`logging.consoleStyle`:

- `pretty`: fácil de interpretar, con colores y marcas de tiempo.
- `compact`: salida más compacta (ideal para sesiones largas).
- `json`: JSON por línea (para procesadores de registros).

### Censura

OpenClaw puede censurar tokens confidenciales antes de que lleguen a la salida de la consola, los registros de archivos,
los registros de log de OTLP, el texto persistente de la transcripción de la sesión o las cargas útiles de eventos de herramientas
de la interfaz de control (argumentos de inicio de herramientas, cargas útiles de resultados parciales/finales, salida de
ejecución derivada y resúmenes de parches):

- La censura de valores confidenciales siempre está habilitada.
- `logging.redactPatterns`: lista de cadenas de expresiones regulares que sustituye el conjunto predeterminado para la salida de registros/transcripciones. En las cargas útiles de herramientas de la interfaz de control, los patrones personalizados se aplican además de los valores predeterminados integrados, por lo que añadir un patrón nunca debilita la censura de valores que ya detectan los valores predeterminados.

Los registros de archivos y las transcripciones de sesiones se mantienen en formato JSONL, pero los valores secretos coincidentes se
enmascaran antes de que la línea o el mensaje se escriban en el disco. La censura se realiza con el máximo esfuerzo:
se aplica al contenido textual de los mensajes y a las cadenas de registro, no a todos los
identificadores ni a los campos de cargas útiles binarias.

Los valores predeterminados integrados abarcan credenciales de API comunes y nombres de campos de credenciales de pago,
como número de tarjeta, CVC/CVV, token de pago compartido y credencial de pago,
cuando aparecen como campos JSON, parámetros de URL, indicadores de CLI o asignaciones.

OpenClaw también censura las cargas útiles de límites de seguridad que se muestran a clientes de la interfaz de usuario, paquetes de
soporte, observadores de diagnósticos, solicitudes de aprobación o herramientas de agentes. Los
`logging.redactPatterns` personalizados pueden añadir patrones específicos del proyecto en esas superficies.

## Diagnósticos y OpenTelemetry

Los diagnósticos son eventos estructurados y legibles por máquinas para ejecuciones del modelo y
telemetría del flujo de mensajes (webhooks, puesta en cola, estado de la sesión). **No**
sustituyen los registros: alimentan métricas, trazas y exportadores. Los eventos se emiten
de forma predeterminada dentro del proceso (establezca `diagnostics.enabled: false` para desactivarlos);
su exportación se configura por separado.

Dos superficies adyacentes:

- **Exportación de OpenTelemetry** — envía métricas, trazas y registros mediante OTLP/HTTP a
  cualquier recopilador o backend compatible con OpenTelemetry (Datadog, Grafana,
  Honeycomb, New Relic, Tempo, etc.). La configuración completa, el catálogo de señales,
  los nombres de métricas/spans, las variables de entorno y el modelo de privacidad se encuentran en una página específica:
  [Exportación de OpenTelemetry](/es/gateway/opentelemetry).
- **Indicadores de diagnóstico** — indicadores específicos de registros de depuración que dirigen registros adicionales a
  `logging.file` sin aumentar `logging.level`. Los indicadores no distinguen entre mayúsculas y minúsculas
  y admiten comodines (`telegram.*`, `*`). Se configuran en `diagnostics.flags`
  o mediante la sustitución de entorno `OPENCLAW_DIAGNOSTICS=...`. Guía completa:
  [Indicadores de diagnóstico](/es/diagnostics/flags).

Para exportar mediante OTLP a un recopilador, consulte [Exportación de OpenTelemetry](/es/gateway/opentelemetry).

## Consejos para solucionar problemas

- **¿No se puede acceder al Gateway?** Ejecute primero `openclaw doctor`.
- **¿Los registros están vacíos?** Compruebe que el Gateway esté en ejecución y escriba en la ruta del archivo
  indicada en `logging.file`.
- **¿Se necesitan más detalles?** Establezca `logging.level` en `debug` o `trace` y vuelva a intentarlo.

## Contenido relacionado

- [Exportación de OpenTelemetry](/es/gateway/opentelemetry) — exportación mediante OTLP/HTTP, catálogo de métricas/spans y modelo de privacidad
- [Indicadores de diagnóstico](/es/diagnostics/flags) — indicadores específicos de registros de depuración
- [Aspectos internos de los registros del Gateway](/es/gateway/logging) — estilos de registros de WS, prefijos de subsistemas y captura de consola
- [Referencia de configuración](/es/gateway/configuration-reference#diagnostics) — referencia completa del campo `diagnostics.*`
