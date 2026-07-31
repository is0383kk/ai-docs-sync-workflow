---
read_when:
    - Cambiar la salida o los formatos de registro
    - Depuración de la salida de la CLI o del Gateway
summary: Superficies de registro, registros en archivos, estilos de registro de WS y formato de la consola
title: Registro del Gateway
x-i18n:
    generated_at: "2026-07-26T04:37:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0b11a68611032c29c31091b2411982487e7f5df3ecf4f1e3b586e7d21e543d3
    source_path: gateway/logging.md
    workflow: 16
---

# Registro

Para obtener una descripción general orientada al usuario (CLI + interfaz de control + configuración), consulte [/logging](/es/logging).

OpenClaw tiene dos superficies de registro:

- **Salida de consola** - lo que se ve en el terminal o en la interfaz de depuración.
- **Registros de archivo** - líneas JSON escritas por el registrador del Gateway.

Al iniciarse, el Gateway registra el modelo de agente predeterminado resuelto, además de los valores predeterminados de modo que afectan a las sesiones nuevas:

```text
modelo de agente: openai/gpt-5.6-sol (razonamiento=medio, rápido=activado)
```

`thinking` proviene del agente predeterminado, de los parámetros del modelo o del valor predeterminado global del agente; cuando no está configurado, muestra `medium`. `fast` proviene del agente predeterminado o de los parámetros `fastMode` del modelo.

## Registrador basado en archivos

- Los archivos de registro rotativos predeterminados se encuentran en `/tmp/openclaw/` (un archivo por día), con fechas según la zona horaria local del host del Gateway. El perfil predeterminado utiliza `openclaw-YYYY-MM-DD.log`; los perfiles con nombre utilizan `openclaw-<profile>-YYYY-MM-DD.log` (por ejemplo, `openclaw-dev-YYYY-MM-DD.log`). Si ese directorio no es seguro o no permite escritura (propietario incorrecto, escritura habilitada para todos o enlace simbólico), OpenClaw utiliza en su lugar una ruta `os.tmpdir()/openclaw-<uid>` específica del usuario; en Windows siempre utiliza esa alternativa basada en el directorio temporal del sistema operativo.
- Los archivos de registro activos rotan al alcanzar `logging.maxFileBytes` (valor predeterminado: 100 MB), conservan hasta cinco archivos numerados (`.1` a `.5`) y continúan escribiendo en un archivo activo nuevo.
- Configure la ruta y el nivel del archivo de registro mediante `~/.openclaw/openclaw.json`: `logging.file`, `logging.level`.
- El formato del archivo es un objeto JSON por línea.

Las rutas de código de conversación, voz en tiempo real y salas administradas utilizan el registrador de archivos compartido para registros limitados del ciclo de vida destinados a la depuración operativa y a la exportación de registros OTLP. El texto de las transcripciones, las cargas de audio, los identificadores de turno, los identificadores de llamada y los identificadores de elementos del proveedor nunca se copian en el registro.

La pestaña Registros de la interfaz de control sigue este archivo mediante el Gateway (`logs.tail`). La CLI hace lo mismo:

```bash
openclaw logs --follow
```

### Modo detallado frente a niveles de registro

- Los **registros de archivo** se controlan exclusivamente mediante `logging.level`.
- `--verbose` solo afecta al **nivel de detalle de la consola** (y al estilo de registro de WS); **no** aumenta el nivel de los registros de archivo.
- Para capturar en los registros de archivo detalles disponibles solo en el modo detallado, configure `logging.level` como `debug` o `trace`.
- El registro de seguimiento también incluye resúmenes de tiempos de diagnóstico para determinadas rutas críticas, como la preparación de la fábrica de herramientas de plugins. Consulte [/tools/plugin#slow-plugin-tool-setup](/es/tools/plugin#slow-plugin-tool-setup).

## Captura de consola

La CLI captura `console.log/info/warn/error/debug/trace`, los escribe en los registros de archivo y continúa imprimiéndolos en stdout/stderr.

Ajuste de forma independiente el nivel de detalle de la consola:

- `logging.consoleLevel` (valor predeterminado: `info`)
- `logging.consoleStyle` (`pretty` | `compact` | `json`; el valor predeterminado es `pretty` en una TTY y `compact` en caso contrario)

## Ocultación

OpenClaw oculta los tokens confidenciales antes de que la salida del registro o de la transcripción salga del proceso. Esta política de ocultación se aplica a los destinos de texto de la consola, los registros de archivo, los registros OTLP y las transcripciones de sesión, por lo que los valores secretos coincidentes se ocultan antes de escribir líneas JSONL o mensajes en el disco.

- La ocultación de valores confidenciales está siempre activada.
- `logging.redactPatterns`: matriz de cadenas de expresiones regulares (sustituye los valores predeterminados)
  - Utilice cadenas de expresiones regulares sin procesar (`gi` automático) o `/pattern/flags` para indicadores personalizados.
  - Las coincidencias se ocultan conservando los primeros 6 y los últimos 4 caracteres (valores de >= 18 caracteres); los valores más cortos se convierten en `***`.
  - Los valores predeterminados abarcan asignaciones habituales de claves, indicadores de la CLI, campos JSON, encabezados de portador, bloques PEM, prefijos de tokens de proveedores populares y nombres de campos de credenciales de pago (número de tarjeta, CVC/CVV, token de pago compartido y credencial de pago).

Los límites de seguridad, como los eventos de llamada a herramientas de la interfaz de control, la salida de `sessions_history`, las exportaciones de diagnóstico, los errores del proveedor, la visualización de aprobaciones de ejecución y los registros de WebSocket del Gateway siempre aplican la ocultación. `logging.redactPatterns` añade patrones específicos de la implementación.

## Registros de WebSocket del Gateway

El Gateway imprime los registros del protocolo WebSocket en dos modos:

- **Modo normal (sin `--verbose`)**: solo se imprimen los resultados RPC «relevantes»: errores (`ok=false`), llamadas lentas (umbral predeterminado: `>= 50ms`) y errores de análisis.
- **Modo detallado (`--verbose`)**: imprime todo el tráfico de solicitudes y respuestas de WS.

### Estilo de registro de WS

`openclaw gateway` admite un selector de estilo por Gateway:

- `--ws-log auto` (valor predeterminado): el modo normal está optimizado; el modo detallado utiliza una salida compacta.
- `--ws-log compact`: salida compacta (solicitud y respuesta emparejadas) en el modo detallado.
- `--ws-log full`: salida completa por trama en el modo detallado.
- `--compact`: alias de `--ws-log compact`.

```bash
# optimizado (solo errores/operaciones lentas)
openclaw gateway

# mostrar todo el tráfico de WS (emparejado)
openclaw gateway --verbose --ws-log compact

# mostrar todo el tráfico de WS (metadatos completos)
openclaw gateway --verbose --ws-log full
```

## Formato de consola (registro por subsistema)

El formateador de consola **detecta la TTY** e imprime líneas coherentes con prefijos. Los registradores de subsistemas mantienen la salida agrupada y fácil de examinar:

- **Prefijos de subsistema** en cada línea (p. ej., `[gateway]`, `[canvas]`, `[tailscale]`).
- **Colores de subsistema** (estables por subsistema, derivados mediante hash del nombre), además de colores por nivel.
- **Color cuando la salida es una TTY** o el entorno parece un terminal enriquecido (`TERM`/`COLORTERM`/`TERM_PROGRAM`); respeta `NO_COLOR` y `FORCE_COLOR`.
- **Prefijos de subsistema abreviados**: elimina un segmento inicial `gateway/`, `channels/` o `providers/` y conserva como máximo los últimos 2 segmentos restantes (p. ej., `channels/turn/kernel` se muestra como `turn/kernel`). Los subsistemas de canales conocidos (`telegram`, `whatsapp`, `slack`, etc.) siempre se reducen únicamente al nombre del canal.
- **Registradores secundarios por subsistema** (prefijo automático + campo estructurado `{ subsystem }`).
- **`logRaw()`** para la salida de QR/UX (sin prefijo ni formato).
- **Estilos de consola**: `pretty` | `compact` | `json`.
- El **nivel de registro de la consola** es independiente del nivel de registro del archivo (el archivo conserva todos los detalles cuando `logging.level` es `debug`/`trace`).
- Los **cuerpos de los mensajes de WhatsApp** se registran en `debug` (utilice `--verbose` para verlos).

Esto mantiene estables los registros de archivo y facilita el examen de la salida interactiva.

## Contenido relacionado

- [Registro](/es/logging)
- [Exportación de OpenTelemetry](/es/gateway/opentelemetry)
- [Exportación de diagnósticos](/es/gateway/diagnostics)
