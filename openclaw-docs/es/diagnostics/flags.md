---
read_when:
    - Necesita registros de depuración específicos sin aumentar los niveles de registro globales
    - Necesita recopilar registros específicos del subsistema para obtener asistencia.
summary: Indicadores de diagnóstico para registros de depuración específicos
title: Indicadores de diagnóstico
x-i18n:
    generated_at: "2026-07-26T04:36:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3bdab6ba1fd98ba58c99c93f9a12d31f57e2655cb0c1eb2de09e34b970f56c
    source_path: diagnostics/flags.md
    workflow: 16
---

Las marcas de diagnóstico activan registros adicionales para un subsistema sin aumentar
`logging.level` globalmente. Una marca no tiene efecto a menos que un subsistema la compruebe.

## Cómo funciona

- Las marcas son cadenas que no distinguen entre mayúsculas y minúsculas, resueltas a partir de `diagnostics.flags` en la
  configuración más la sobrescritura de entorno `OPENCLAW_DIAGNOSTICS`, sin duplicados y convertidas a minúsculas.
- `name.*` coincide con el propio `name` y con todo lo que haya bajo `name.` (por ejemplo,
  `telegram.*` coincide con `telegram.http`).
- `*` o `all` activa todas las marcas.
- Reinicie el Gateway después de cambiar `diagnostics.flags` en la configuración; no se
  recarga en caliente.

## Marcas conocidas

| Marca                 | Activa                                                    |
| --------------------- | --------------------------------------------------------- |
| `telegram.http`       | Registro de errores HTTP de la API de bots de Telegram    |
| `brave.http`          | Registro de solicitudes, respuestas y caché de Brave Search |
| `profiler`            | Perfilador de la etapa de respuesta y del servidor de aplicaciones de Codex (ambos) |
| `reply.profiler`      | Solo el perfilador de la etapa de respuesta               |
| `codex.profiler`      | Solo el perfilador del servidor de aplicaciones de Codex  |
| `health`              | Detalles de depuración de sondas de estado, cuentas y vinculaciones del Gateway |
| `ingress.timing`      | Tiempos de carga de sesiones, selección de modelos y catálogo de modelos |
| `plugin.load-profile` | Tiempos de carga síncrona de módulos de plugins            |
| `timeline`            | Artefacto de cronología JSONL estructurada (véase más adelante) |

## Activación mediante configuración

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Varias marcas:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "brave.http", "gateway.*"]
  }
}
```

## Sobrescritura mediante variable de entorno (ocasional)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,brave.http
```

Los valores se separan por comas o espacios en blanco. Valores especiales:

| Valor                       | Efecto                                   |
| --------------------------- | ---------------------------------------- |
| `0`, `false`, `off`, `none` | Desactiva todas las marcas y también sobrescribe la configuración |
| `1`, `true`, `all`, `*`     | Activa todas las marcas                  |

`OPENCLAW_DIAGNOSTICS=0` desactiva las marcas tanto del entorno como de la configuración para ese
proceso, lo que resulta útil para silenciar temporalmente una marca del perfilador que se dejó activada en la configuración
sin editar el archivo.

## Marcas del perfilador

Las marcas del perfilador controlan intervalos ligeros de medición de tiempo; no añaden sobrecarga cuando están desactivadas.

Active todos los intervalos controlados por el perfilador para una ejecución del Gateway:

```bash
OPENCLAW_DIAGNOSTICS=profiler openclaw gateway run
```

Active solo los intervalos del perfilador de despacho de respuestas:

```bash
OPENCLAW_DIAGNOSTICS=reply.profiler openclaw gateway run
```

Active solo los intervalos del perfilador de inicio, herramientas e hilos del servidor de aplicaciones de Codex:

```bash
OPENCLAW_DIAGNOSTICS=codex.profiler openclaw gateway run
```

`profiler` activa tanto el perfilador de respuestas como el perfilador de Codex; utilice los
nombres de marca con ámbito para activar solo uno.

También se puede establecer en la configuración:

```json
{
  "diagnostics": {
    "flags": ["reply.profiler", "codex.profiler"]
  }
}
```

Reinicie el Gateway después de cambiar las marcas de configuración. Para desactivar una marca del perfilador,
elimínela de `diagnostics.flags` y reinicie, o inicie el proceso con
`OPENCLAW_DIAGNOSTICS=0` para sobrescribir todas las marcas de diagnóstico durante esa ejecución.

## Artefactos de cronología

La marca `timeline` (alias: `diagnostics.timeline`) escribe los eventos estructurados de tiempo de inicio
y ejecución como JSONL para sistemas externos de control de calidad:

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

También se puede activar en la configuración:

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

La ruta de salida siempre procede de `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH`, incluso
cuando la propia marca se establece en la configuración; no existe ninguna clave de configuración para la ruta.
Cuando `timeline` se activa solo desde la configuración, faltan los primeros intervalos de carga de la configuración
porque OpenClaw aún no la ha leído; los intervalos de inicio posteriores
se capturan con normalidad.

`OPENCLAW_DIAGNOSTICS=1`, `=all` y `=*` también activan la cronología, ya que
activan todas las marcas. Utilice preferentemente la marca con ámbito `timeline` cuando solo se necesite el
artefacto JSONL y no todas las demás marcas de diagnóstico.

Las muestras de retraso del bucle de eventos en la cronología requieren una activación adicional además de
`timeline`: establezca `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` (o `on`/`true`/`yes`) además
de activar la cronología.

Los registros de cronología utilizan el contenedor `openclaw.diagnostics.v1` y pueden incluir
identificadores de procesos, nombres de fases, nombres de intervalos, duraciones, identificadores de plugins, recuentos de
dependencias, muestras de retraso del bucle de eventos, nombres de operaciones del proveedor, estado de salida
de procesos secundarios y nombres o mensajes de errores de inicio. Trate los archivos de cronología como
artefactos de diagnóstico locales; revíselos antes de compartirlos fuera de su equipo.

## Destino de los registros

Las marcas emiten registros en el archivo de registro de diagnóstico estándar. De forma predeterminada:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Los perfiles con nombre utilizan `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`; por
ejemplo, `--dev` utiliza `openclaw-dev-YYYY-MM-DD.log`.

Si establece `logging.file`, utilice esa ruta en su lugar. Los registros están en formato JSONL (un objeto JSON
por línea). La ocultación sigue aplicándose según `logging.redactSensitive`.
Consulte [Registro](/es/logging) para conocer el modelo completo de resolución de rutas, rotación y
ocultación de registros.

## Extracción de registros

Lea el archivo de registro más reciente del perfil activo:

```bash
openclaw logs --plain
# Ejemplo de perfil con nombre:
openclaw --profile work logs --plain
```

Filtre los diagnósticos HTTP de Telegram:

```bash
openclaw logs --plain --limit 5000 | rg "telegram http error"
```

Filtre los diagnósticos HTTP de Brave Search:

```bash
openclaw logs --plain --limit 5000 | rg "brave http"
```

O siga los registros mientras reproduce el problema:

```bash
openclaw logs --follow --plain | rg "telegram http error"
```

Para Gateways remotos, utilice `openclaw logs --follow` en su lugar (consulte
[/cli/logs](/es/cli/logs)).

## Notas

- Si `logging.level` se establece por encima de `warn`, los registros controlados por marcas pueden
  suprimirse. El valor predeterminado de `info` es adecuado.
- `brave.http` registra las URL y los parámetros de consulta de las solicitudes de Brave Search, el
  estado y los tiempos de respuesta, así como los eventos de acierto, fallo y escritura de la caché. No registra la clave de la API
  (enviada como encabezado de solicitud) ni los cuerpos de las respuestas, pero las consultas de búsqueda pueden ser
  confidenciales.
- Es seguro dejar las marcas activadas; solo afectan al volumen de registros del
  subsistema específico.
- Utilice [/logging](/es/logging) para cambiar los destinos, niveles y la ocultación de los registros.

## Relacionado

- [Diagnósticos del Gateway](/es/gateway/diagnostics)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
