---
read_when:
    - Uso de /steer o /tell mientras un agente ya está en ejecución
    - Comparación de los modos /steer y /queue
    - Decidir si dirigir la ejecución actual o una sesión de ACP
sidebarTitle: Steer
summary: Dirige una ejecución activa sin cambiar el modo de cola
title: Dirigir
x-i18n:
    generated_at: "2026-07-26T05:34:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d420e14982d52520e415103ffa6d86923fad6f13c43ff7741ebbd8dde0d0073f
    source_path: tools/steer.md
    workflow: 16
---

`/steer` primero intenta enviar indicaciones a una ejecución que ya está activa. Está pensado para
situaciones en las que se desea «ajustar esta ejecución mientras sigue trabajando». Si el entorno de ejecución actual
no puede aceptar indicaciones, OpenClaw envía el mensaje como un prompt normal en lugar
de descartarlo.

## Sesión actual

Use `/steer` en el nivel superior para dirigirse a la ejecución activa de la sesión actual:

```text
/steer prioriza el parche más pequeño y mantén las pruebas enfocadas
/tell resume antes de realizar la siguiente llamada a una herramienta
```

Comportamiento:

- Se dirige únicamente a la ejecución activa de la sesión actual.
- Funciona independientemente del modo `/queue` de la sesión.
- Inicia un turno normal con el mismo mensaje cuando la sesión está inactiva o la
  ejecución activa no puede aceptar indicaciones.
- Usa la ruta de indicaciones del entorno de ejecución activo, por lo que el modelo recibe las indicaciones en
  el siguiente límite compatible del entorno de ejecución.

## Indicar frente a poner en cola

`/queue steer` hace que los mensajes entrantes normales intenten proporcionar indicaciones a la ejecución activa cuando
llegan mientras hay una ejecución activa. `/steer <message>` es un comando explícito
que intenta insertar el mensaje de dicho comando en la ejecución activa en el siguiente
límite compatible del entorno de ejecución, independientemente del ajuste `/queue` almacenado. Cuando
esa inserción no está disponible, se elimina el prefijo del comando y `<message>`
continúa como un prompt normal.

El comando explícito `/steer` (y `/tell`) está respaldado por el Gateway. En
`openclaw chat` o `openclaw tui --local`, seleccione `/queue steer` y envíe las
indicaciones como un mensaje normal; el entorno de ejecución integrado aplica la misma política de indicaciones
sin reenviar un comando del Gateway.

Use:

- `/steer <message>` cuando se quiera orientar la ejecución activa de inmediato.
- `/queue steer` cuando se quiera que los futuros mensajes normales proporcionen indicaciones a las ejecuciones activas de
  forma predeterminada.
- `/queue collect` o `/queue followup` cuando los futuros mensajes normales deban esperar
  a un turno posterior en lugar de proporcionar indicaciones a la ejecución activa.
- `/queue interrupt` cuando el mensaje más reciente deba reemplazar la ejecución activa
  en lugar de proporcionarle indicaciones.

Para obtener información sobre los modos de cola y los límites de las indicaciones, consulte [Cola de comandos](/es/concepts/queue) y
[Cola de indicaciones](/es/concepts/queue-steering).

## Subagentes

`/steer` en el nivel superior se dirige a la ejecución activa de la sesión actual. Los subagentes informan
a su sesión superior o solicitante; `/subagents` solo proporciona visibilidad.

## Sesiones ACP

Use `/acp steer` cuando el destino sea una sesión del entorno ACP:

```text
/acp steer --session agent:main:acp:codex precisa la reproducción
```

Consulte [Agentes ACP](/es/tools/acp-agents) para obtener información sobre la selección de sesiones ACP y el comportamiento del entorno
de ejecución.

## Temas relacionados

- [Comandos con barra](/es/tools/slash-commands)
- [Cola de comandos](/es/concepts/queue)
- [Cola de indicaciones](/es/concepts/queue-steering)
- [Subagentes](/es/tools/subagents)
