---
doc-schema-version: 1
read_when:
    - Quieres que OpenClaw mantenga un objetivo visible durante una sesión prolongada
    - Necesita pausar, reanudar, bloquear, completar o borrar el objetivo de una sesión
    - Quieres comprender las herramientas get_goal, create_goal y update_goal
    - Quieres ver cómo aparecen los objetivos en la TUI
summary: 'Objetivos de sesión: objetivos duraderos por sesión, controles de /goal, herramientas de objetivos del modelo, presupuestos de tokens y estado de la TUI'
title: Objetivo
x-i18n:
    generated_at: "2026-07-26T05:32:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8bfe25eb9901394b32b61729fbcb6a7bd711ed859d284fa39b637000ed7f0a18
    source_path: tools/goal.md
    workflow: 16
---

# Objetivo

Un **objetivo** es un propósito duradero asociado a la sesión actual de OpenClaw.
Proporciona al agente y al operador una meta compartida para trabajos de larga duración,
sin convertirla en una tarea en segundo plano, un recordatorio, un trabajo Cron ni una
orden permanente.

Los objetivos forman parte del estado de la sesión: se trasladan con la clave de sesión, sobreviven a los
reinicios del proceso y aparecen en `/goal`, en las herramientas de objetivos disponibles para el modelo y en el pie de página de la TUI.

Las finalizaciones de comandos desacoplados regresan al hilo visible para el usuario que los originó, por lo que
el siguiente turno continúa viendo el mismo objetivo aunque la ejecución del comando haya utilizado
una sesión independiente con una política de entorno aislado.

## Inicio rápido

```text
/goal start conseguir que la CI esté en verde para el PR 87469 y enviar la corrección
/goal
/goal edit conseguir que la CI esté en verde para el PR 87469, enviar la corrección y actualizar la documentación
/goal pause esperando a la CI
/goal resume
/goal complete enviado y verificado
/goal clear
```

`start` es opcional: `/goal get CI green for PR 87469` también crea un objetivo,
ya que cualquier texto después de `/goal` que no sea una palabra de acción conocida se trata como un
nuevo propósito.

## Para qué sirven los objetivos

Utilice un objetivo cuando una sesión tenga un resultado concreto que deba permanecer visible
durante muchos turnos:

- Cierre de un PR: corregir, verificar, ejecutar la revisión automática, enviar y abrir o actualizar el PR.
- Una sesión de depuración: reproducir el error, identificar la superficie propietaria, aplicar un parche y
  demostrar la corrección.
- Una revisión de documentación: leer la documentación pertinente, redactar la página nueva, añadir enlaces cruzados y
  verificar la compilación de la documentación.
- Una tarea de mantenimiento: inspeccionar el estado actual, realizar cambios acotados, ejecutar las
  comprobaciones adecuadas e informar de lo que cambió.

Un objetivo no es una cola de tareas. Utilice [Task Flow](/es/automation/taskflow),
[tareas](/es/automation/tasks), [trabajos Cron](/es/automation/cron-jobs) u
[órdenes permanentes](/es/automation/standing-orders) cuando el trabajo deba ejecutarse de forma desacoplada,
repetirse según una programación, distribuirse en subtareas gestionadas o persistir como una política.

## Referencia de comandos

`/goal` sin argumentos muestra el resumen del objetivo actual:

```text
Objetivo
Estado: activo
Propósito: conseguir que la CI esté en verde para el PR 87469 y enviar la corrección
Tokens utilizados: 12k
Presupuesto de tokens: 12k/50k

Comandos: /goal edit <objective>, /goal pause, /goal complete, /goal clear
```

| Comando                                             | Efecto                                                                   |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| `/goal` o `/goal status`                           | Muestra el objetivo actual.                                                   |
| `/goal start <objective>`                           | Crea un nuevo objetivo para la sesión actual.                               |
| `/goal set <objective>`, `/goal create <objective>` | Alias de `start`.                                                     |
| `/goal <objective>`                                 | También crea un nuevo objetivo (cualquier texto que no sea una palabra de acción reconocida). |
| `/goal edit <objective>`                            | Reformula el propósito actual; el estado y la contabilidad de tokens no cambian.      |
| `/goal pause [note]`                                | Pausa un objetivo activo.                                                    |
| `/goal resume [note]`                               | Reanuda un objetivo pausado, bloqueado, limitado por uso o limitado por presupuesto.         |
| `/goal complete [note]`                             | Marca el objetivo como alcanzado.                                                  |
| `/goal done [note]`                                 | Alias de `complete`.                                                    |
| `/goal block [note]`                                | Marca el objetivo como bloqueado.                                                   |
| `/goal blocked [note]`                              | Alias de `block`.                                                       |
| `/goal clear`                                       | Elimina el objetivo de la sesión.                                        |

Solo puede existir un objetivo por sesión a la vez. Intentar iniciar un segundo objetivo falla
con `Goal error: goal already exists` hasta que se borre el actual.

`/goal start` no acepta una opción de presupuesto de tokens; el presupuesto solo se puede establecer
mediante la herramienta `create_goal` disponible para el modelo.

## Estados

- `active`: la sesión está persiguiendo el objetivo.
- `paused`: el operador pausó el objetivo; `/goal resume` vuelve a activarlo.
- `blocked`: el agente o el operador informó de un bloqueo real; `/goal resume`
  vuelve a activarlo cuando hay información o un estado nuevos disponibles.
- `budget_limited`: se alcanzó el presupuesto de tokens configurado; `/goal resume`
  reinicia la consecución desde el mismo propósito con una nueva ventana de presupuesto.
- `usage_limited`: reservado para un futuro estado de detención por límite de uso; `/goal
resume` reinicia la consecución del mismo modo.
- `complete`: se alcanzó el objetivo. Los objetivos completados son terminales; utilice `/goal
clear` antes de iniciar otro objetivo.

`/new` y `/reset` borran el objetivo de la sesión actual, ya que inician
deliberadamente un contexto de sesión nuevo.

## Presupuestos de tokens

Los objetivos pueden tener un presupuesto de tokens positivo opcional, establecido mediante el
parámetro `token_budget` de la herramienta `create_goal`. El presupuesto se mide a partir del
recuento de tokens actualizado de la sesión en el momento de crear el objetivo. Si la sesión solo tiene una
instantánea de tokens obsoleta o desconocida cuando se inicia el objetivo, OpenClaw espera a la
siguiente instantánea actualizada y la utiliza como referencia, por lo que los tokens consumidos antes de que
existiera el objetivo no se contabilizan en él.

Cuando el uso alcanza el presupuesto, el objetivo pasa a `budget_limited`. Esto no
elimina el objetivo ni borra el propósito; indica al operador y al
agente que el objetivo ya no se está persiguiendo activamente hasta que se reanude o
se borre. Al reanudarlo, se inicia una nueva ventana de presupuesto a partir del recuento de tokens
actualizado.

Los presupuestos de tokens son una medida de protección para el objetivo de la sesión, no un límite de facturación. La
cuota del proveedor, los informes de costes y el comportamiento de la ventana de contexto siguen utilizando los
controles habituales de uso y modelo de OpenClaw.

## Herramientas del modelo

OpenClaw expone tres herramientas de objetivos a los entornos de agentes:

| Herramienta          | Finalidad                                                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `get_goal`    | Lee el objetivo de la sesión actual: estado, propósito, uso de tokens y presupuesto de tokens.                                         |
| `create_goal` | Crea un objetivo solo cuando el usuario o las instrucciones del sistema lo solicitan explícitamente. Falla si la sesión ya tiene un objetivo. |
| `update_goal` | Marca el objetivo como `complete` o `blocked`.                                                                                   |

El modelo no puede pausar, reanudar, borrar ni reemplazar silenciosamente un objetivo. Estas acciones siguen siendo
controles del operador o de la sesión mediante `/goal` y los comandos de reinicio, de modo que el agente
pueda informar de la consecución o de un bloqueo genuino sin cambiar silenciosamente la
meta.

`update_goal` debe marcar un objetivo como `complete` solo cuando el propósito se haya
alcanzado realmente. Debe marcar un objetivo como `blocked` solo después de que la misma
condición de bloqueo se repita durante al menos tres turnos consecutivos del objetivo, no por
dificultades habituales ni por falta de perfeccionamiento.

## Contexto del objetivo en cada turno

Cada turno de usuario o chat con un objetivo activo incluye esta línea de contexto con el rol de usuario:

```text
Objetivo activo: <objective> — aváncelo o actualice su estado (get_goal/update_goal).
```

OpenClaw mantiene la línea compacta truncando los propósitos largos. Los objetivos pausados,
bloqueados, limitados por presupuesto, limitados por uso y completados no se insertan,
por lo que una detención del operador permanece vigente hasta que se reanude el objetivo.

## Interfaz de control

La interfaz web de control muestra el objetivo como una píldora compacta sobre el redactor del chat:
un icono de estado, la etiqueta del estado (por ejemplo, `Pursuing goal`), el
propósito truncado y un temporizador en directo del tiempo transcurrido.

La píldora contiene controles en línea:

- **Lápiz** rellena previamente el redactor con `/goal edit <objective>` para que el
  propósito se pueda reformular y enviar.
- **Pausar/reanudar** alterna entre `/goal pause` y `/goal resume` según
  el estado actual.
- **Papelera** envía `/goal clear`.
- **Chevrón** expande la píldora para mostrar el propósito completo, la nota de estado
  más reciente, el uso de tokens y el tiempo transcurrido.

Los botones de acción están ocultos mientras el redactor no puede enviar mensajes (por ejemplo,
cuando la conexión con el Gateway está inactiva); el chevrón de expansión sigue funcionando.

## TUI

El pie de página de la TUI mantiene visible el objetivo de la sesión activa junto a los campos del agente,
la sesión y el modelo, antes de los indicadores de tokens y modo.

Ejemplos del pie de página:

- `Pursuing goal (12k/50k)` para un objetivo activo con un presupuesto de tokens.
- `Goal paused (/goal resume)` para un objetivo pausado.
- `Goal blocked (/goal resume)` para un objetivo bloqueado.
- `Goal hit usage limits (/goal resume)` para un objetivo limitado por uso.
- `Goal unmet (50k/50k)` para un objetivo limitado por presupuesto.
- `Goal achieved (42k)` para un objetivo completado.

El pie de página es deliberadamente compacto. Utilice `/goal` para consultar el propósito
completo, la nota, el presupuesto de tokens y los comandos disponibles.

## Comportamiento en los canales

`/goal` funciona en las sesiones de OpenClaw que admiten comandos, incluidas la TUI y
las superficies de chat que permiten comandos de texto. El estado del objetivo está asociado a la
clave de sesión, no al transporte, por lo que dos superficies que compartan una clave de sesión ven el
mismo objetivo.

El estado del objetivo no es una directiva de entrega: no fuerza las respuestas a través de un
canal, no cambia el comportamiento de la cola, no aprueba herramientas ni programa trabajos.

## Solución de problemas

| Mensaje                                | Significado                                                                                                                                      |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `Goal error: goal already exists`      | La sesión ya tiene un objetivo. Utilice `/goal` para inspeccionarlo, `/goal complete` si ha terminado o `/goal clear` antes de iniciar un propósito diferente. |
| `Goal error: goal not found`           | La sesión aún no tiene ningún objetivo. Inicie uno con `/goal start <objective>`.                                                                       |
| `Goal error: goal is already complete` | El objetivo es terminal. Bórrelo antes de iniciar o reanudar otro propósito.                                                                |

Si el uso de tokens muestra `0` o parece obsoleto, es posible que la sesión activa aún no tenga una
instantánea de tokens actualizada. El uso se actualiza a medida que OpenClaw registra el uso de la sesión
y los totales derivados de la transcripción.

## Relacionado

- [Comandos de barra](/es/tools/slash-commands)
- [TUI](/es/web/tui)
- [Herramienta de sesión](/es/concepts/session-tool)
- [Compaction](/es/concepts/compaction)
- [Task Flow](/es/automation/taskflow)
- [Órdenes permanentes](/es/automation/standing-orders)
