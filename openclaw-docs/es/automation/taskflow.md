---
read_when:
    - Quieres entender cómo se relaciona Task Flow con las tareas en segundo plano
    - Encuentra Task Flow o el flujo de tareas de OpenClaw en las notas de la versión o en la documentación
    - Se desea inspeccionar o gestionar el estado persistente del flujo
summary: Capa de orquestación de Task Flow sobre las tareas en segundo plano
title: Flujo de tareas
x-i18n:
    generated_at: "2026-07-26T05:30:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5ccc6acf58b4b44c2989e3061bff08dabce8ef385706102360c756a1286ddd1b
    source_path: automation/taskflow.md
    workflow: 16
---

Task Flow es la capa de orquestación situada sobre las [tareas en segundo plano](/es/automation/tasks). Un flujo es un registro duradero de trabajo de varios pasos con su propio estado, estado JSON, contador de revisiones y registros de tareas vinculados. Los flujos sobreviven a los reinicios del Gateway; las tareas individuales siguen siendo la unidad de trabajo independiente.

## Cuándo usar Task Flow

| Escenario                                         | Uso                                              |
| ------------------------------------------------- | ------------------------------------------------ |
| Trabajo único en segundo plano                    | Tarea simple                                     |
| Pipeline de varios pasos controlado por un plugin | Task Flow (gestionado)                           |
| Inicio independiente de ACP o subagente           | Task Flow (reflejado, creado automáticamente)    |
| Recordatorio puntual                              | Trabajo de Cron                                  |

## Modos de sincronización

### Modo gestionado

Un flujo gestionado tiene un controlador: código de plugin que crea el flujo mediante la API de Task Flow del entorno de ejecución del plugin con un objetivo y un id. de controlador obligatorio y, a continuación, lo controla explícitamente.

- Cada paso se ejecuta como una tarea en segundo plano creada dentro del flujo; la clave del propietario y el origen del solicitante del flujo se transfieren a las tareas secundarias.
- El controlador hace avanzar el flujo entre `running`, `waiting` y los estados terminales, y almacena un estado de paso JSON arbitrario en el registro del flujo.
- Cada mutación incluye la revisión esperada del flujo. Una escritura obsoleta se rechaza como conflicto de revisión en lugar de sobrescribir el estado más reciente.
- Una vez solicitada la cancelación, se rechazan las nuevas tareas secundarias y el flujo finaliza como `cancelled` cuando no queda ninguna tarea secundaria activa.

Ejemplo: un flujo de informe semanal que (1) recopila datos, (2) genera el informe y (3) lo entrega, con una tarea en segundo plano por paso:

```
Flujo: weekly-report
  Paso 1: gather-data     → tarea creada → completada correctamente
  Paso 2: generate-report → tarea creada → completada correctamente
  Paso 3: deliver         → tarea creada → en ejecución
```

### Modo reflejado

OpenClaw crea automáticamente un flujo reflejado de una sola tarea cuando se inicia una ejecución independiente de ACP o de un subagente (tareas con ámbito de sesión y finalización entregable). El registro del flujo refleja su única tarea subyacente —estado, objetivo y tiempos— para que los inicios independientes dispongan de un identificador de flujo estable en las superficies de estado y reintento sin necesidad de un controlador. Los flujos reflejados muestran el modo de sincronización `task_mirrored` en la CLI.

## Estados de los flujos

| Estado      | Significado                                                                 |
| ----------- | --------------------------------------------------------------------------- |
| `queued`    | Creado, aún no está avanzando                                               |
| `running`   | El flujo está avanzando activamente                                         |
| `waiting`   | El flujo gestionado está detenido según metadatos de espera (temporizador, evento externo) |
| `blocked`   | Un paso terminó sin un resultado utilizable; `blockedTaskId`/resumen indican cuál |
| `succeeded` | Completado correctamente                                                    |
| `failed`    | Completado con un error                                                     |
| `cancelled` | Cancelación solicitada y todas las tareas secundarias resueltas             |
| `lost`      | El flujo perdió su estado subyacente autoritativo                           |

## Estado duradero y seguimiento de revisiones

Los registros de flujos persisten en la base de datos SQLite de estado compartido (tabla `~/.openclaw/state/openclaw.sqlite`, `flow_runs`) junto con los registros de tareas, por lo que el progreso sobrevive a los reinicios del Gateway. Cada escritura incrementa el `revision` del flujo; los escritores simultáneos que proporcionan una revisión esperada obsoleta reciben un conflicto y deben volver a leer. El crecimiento de WAL está limitado mediante los puntos de control automáticos de SQLite y puntos de control pasivos periódicos, con puntos de control de truncamiento durante el apagado. `openclaw doctor` importa el archivo auxiliar heredado `flows/registry.sqlite` de instalaciones anteriores.

## Comportamiento de cancelación

`openclaw tasks flow cancel` establece una intención de cancelación persistente en el flujo, cancela sus tareas secundarias activas y rechaza nuevas tareas secundarias gestionadas. Cuando ya no queda ninguna tarea secundaria activa, el flujo finaliza como `cancelled`, de inmediato o mediante el barrido de mantenimiento si las tareas secundarias tardan más en resolverse. La intención se conserva, por lo que un flujo cancelado sigue cancelado aunque el Gateway se reinicie antes de que todas las tareas secundarias hayan terminado.

## Comandos de la CLI

```bash
# Enumerar los flujos activos y recientes
openclaw tasks flow list [--status <status>] [--json]

# Mostrar los detalles de un flujo específico
openclaw tasks flow show <lookup> [--json]

# Cancelar un flujo en ejecución y sus tareas activas
openclaw tasks flow cancel <lookup>
```

| Comando                           | Descripción                                                             |
| --------------------------------- | ----------------------------------------------------------------------- |
| `openclaw tasks flow list`        | Flujos registrados con modo de sincronización, estado, revisión, controlador y recuentos de tareas |
| `openclaw tasks flow show <id>`   | Inspecciona un flujo por id. de flujo o clave de propietario, incluidas las tareas vinculadas |
| `openclaw tasks flow cancel <id>` | Cancela un flujo en ejecución y sus tareas activas                      |

Los flujos también están cubiertos por `openclaw tasks audit` (hallazgos de flujos obsoletos o dañados) y `openclaw tasks maintenance` (finaliza cancelaciones bloqueadas y elimina flujos terminales después de 7 días).

## Patrón fiable de flujo de trabajo programado

Para flujos de trabajo recurrentes, como informes de inteligencia de mercado, las comprobaciones de programación, orquestación y fiabilidad deben tratarse como capas independientes:

1. Usar [Tareas programadas](/es/automation/cron-jobs) para la temporización.
2. Usar una sesión de Cron persistente cuando el flujo de trabajo deba aprovechar el contexto anterior.
3. Usar [Lobster](/es/tools/lobster) para pasos deterministas, puertas de aprobación y tokens de reanudación.
4. Usar Task Flow para realizar el seguimiento de la ejecución de varios pasos entre tareas secundarias, esperas, reintentos y reinicios del Gateway.

Ejemplo de estructura de Cron:

```bash
openclaw cron add \
  --name "Resumen de inteligencia de mercado" \
  --cron "0 7 * * 1-5" \
  --tz "America/New_York" \
  --session session:market-intel \
  --message "Ejecuta el flujo de trabajo de Lobster market-intel. Verifica la vigencia de las fuentes antes de resumir." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

Usar `--session session:<id>` en lugar de `isolated` cuando el flujo de trabajo recurrente necesite un historial deliberado, resúmenes de ejecuciones anteriores o contexto permanente. Usar `isolated` cuando cada ejecución deba comenzar desde cero y todo el estado necesario esté explícito en el flujo de trabajo.

Dentro del flujo de trabajo, colocar las comprobaciones de fiabilidad antes del paso de resumen del LLM:

```yaml
name: market-intel-brief
steps:
  - id: preflight
    command: market-intel check --json
  - id: collect
    command: market-intel collect --json
    stdin: $preflight.json
  - id: summarize
    command: market-intel summarize --json
    stdin: $collect.json
  - id: approve
    command: market-intel deliver --preview
    stdin: $summarize.json
    approval: required
  - id: deliver
    command: market-intel deliver --execute
    stdin: $summarize.json
    condition: $approve.approved
```

Comprobaciones preliminares recomendadas:

- Disponibilidad del navegador y elección del perfil, por ejemplo, `openclaw` para el estado gestionado o `user` cuando se requiera una sesión de Chrome con la sesión iniciada. Consultar [Navegador](/es/tools/browser).
- Credenciales y cuota de la API para cada fuente.
- Accesibilidad de red de los endpoints necesarios.
- Herramientas necesarias habilitadas para el agente, como `lobster`, `browser` y `llm-task`.
- Destino de errores configurado para Cron de modo que los errores de las comprobaciones preliminares sean visibles. Consultar [Tareas programadas](/es/automation/cron-jobs#delivery-and-output).

Campos de procedencia de datos recomendados para cada elemento recopilado:

```json
{
  "sourceUrl": "https://example.com/report",
  "retrievedAt": "2026-04-24T12:00:00Z",
  "asOf": "2026-04-24",
  "title": "Informe de ejemplo",
  "content": "..."
}
```

Configurar el flujo de trabajo para que rechace o marque como obsoletos los elementos antes de resumirlos. El paso del LLM solo debe recibir JSON estructurado y se le debe solicitar que conserve `sourceUrl`, `retrievedAt` y `asOf` en su salida. Usar [Tarea de LLM](/es/tools/llm-task) cuando se necesite un paso de modelo validado mediante un esquema dentro del flujo de trabajo.

Para flujos de trabajo reutilizables de equipos o comunidades, empaquetar la CLI, los archivos `.lobster` y las notas de configuración como una skill o un plugin y publicarlos mediante [ClawHub](/es/clawhub). Mantener las protecciones específicas del flujo de trabajo en ese paquete, salvo que la API del plugin carezca de alguna capacidad genérica necesaria.

## Relación entre los flujos y las tareas

Los flujos coordinan las tareas, no las sustituyen. Un único flujo puede controlar varias tareas en segundo plano a lo largo de su ciclo de vida. Usar `openclaw tasks` para inspeccionar los registros de tareas individuales y `openclaw tasks flow` para inspeccionar el flujo de orquestación.

## Temas relacionados

- [Tareas en segundo plano](/es/automation/tasks) - el registro de trabajo independiente que coordinan los flujos
- [CLI: tareas](/es/cli/tasks) - referencia de los comandos de la CLI para `openclaw tasks flow`
- [Descripción general de la automatización](/es/automation) - todos los mecanismos de automatización de un vistazo
- [Trabajos de Cron](/es/automation/cron-jobs) - trabajos programados que pueden alimentar los flujos
