---
read_when:
    - Quieres hacer una pregunta rápida aparte sobre la sesión actual
    - Está implementando o depurando el comportamiento de BTW en distintos clientes
summary: Preguntas secundarias efímeras con /btw
title: Por cierto, preguntas secundarias
x-i18n:
    generated_at: "2026-07-26T05:23:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 338a54d0e15ec90aebaeeaee551559a26f1437f7b6dcdde4a4b1e63347ad0759
    source_path: tools/btw.md
    workflow: 16
---

`/btw` (alias `/side`) formula una pregunta secundaria rápida sobre la **sesión
actual** sin añadirla al historial de la conversación. Se basa en
`/btw` de Claude Code, adaptado al Gateway y a la arquitectura
multicanal de OpenClaw.

```text
/btw ¿qué ha cambiado?
/side ¿qué significa este error?
```

## Qué hace

1. Captura una instantánea de la sesión actual como contexto de referencia (incluido cualquier
   prompt de la ejecución principal en curso).
2. Ejecuta una consulta secundaria independiente y de una sola ejecución, e indica al modelo que responda únicamente
   a la pregunta secundaria y que no reanude ni dirija la tarea principal.
3. Entrega la respuesta como un resultado secundario en directo, no como un mensaje normal del asistente.
4. Nunca escribe la pregunta ni la respuesta en el historial de la sesión ni en `chat.history`.

La ejecución principal, si hay una activa, no se modifica.

En las sesiones del arnés de Codex, BTW bifurca el hilo activo del servidor de aplicaciones de Codex en
un hilo secundario efímero, en lugar de realizar una llamada independiente al proveedor. Esto
mantiene intactos OAuth de Codex y el comportamiento nativo de las herramientas y los hilos, y el hilo
bifurcado conserva la política de aprobación, el entorno aislado y la superficie de herramientas
nativas actuales del hilo principal. El hilo bifurcado recibe un prompt delimitador que indica al modelo que
todo lo anterior es contexto de referencia heredado, no instrucciones activas,
y que solo están activos los mensajes posteriores al límite. `/btw` requiere un
hilo de Codex existente; primero debe enviarse un mensaje normal.

Para los alias de tiempo de ejecución de la CLI, BTW invoca el backend de CLI propietario en modo
de pregunta secundaria de una sola ejecución: introduce contexto de conversación saneado en una nueva
invocación de la CLI con la agrupación de herramientas y el estado reutilizable de la sesión desactivados, y añade
cualquier indicador de no reanudación o de ausencia de herramientas que admita el backend. Los tiempos de ejecución directos
(que no usan la CLI) realizan en su lugar una llamada directa al proveedor de una sola ejecución.

## Qué no hace

`/btw` no crea una sesión duradera, no continúa la tarea principal inacabada,
no conserva los datos de la pregunta y la respuesta en el historial de la transcripción ni sobrevive a una recarga.

## Modelo de entrega

El chat normal del asistente usa el evento `chat` del Gateway. BTW usa un evento
`chat.side_result` independiente para que los clientes no puedan confundirlo con el historial
normal de la conversación. Como no se vuelve a reproducir desde `chat.history`,
desaparece después de una recarga.

## Comportamiento de las interfaces

| Interfaz          | Comportamiento                                                                                                                                                                                                                                                                            |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TUI               | Se muestra en línea en el registro del chat, claramente diferenciado de una respuesta normal y descartable con `Enter` o `Esc`.                                                                                                                                                                           |
| Canales externos  | Se entrega como una respuesta de una sola vez claramente etiquetada (Telegram, WhatsApp y Discord no tienen una superposición efímera local).                                                                                                                                                                         |
| Interfaz de control / web | Se muestra como un panel flotante «Chat secundario» fijado al hilo. Las respuestas se acumulan como turnos y un campo «Seguimiento» permite formular la siguiente pregunta secundaria. Al cerrarlo (`Esc` o la X), se conserva la conversación y vuelve a abrirse con la siguiente respuesta; el botón de la papelera la descarta y detiene cualquier ejecución pendiente. |

## Ventana emergente de selección (Interfaz de control)

Al resaltar texto dentro de un mensaje de chat en la Interfaz de control, se abre una pequeña
ventana emergente de selección con dos acciones:

- **Más detalles** envía inmediatamente una pregunta implícita `/btw` que solicita al
  modelo que explique el texto resaltado en el contexto de la sesión
  actual. La respuesta aparece en el panel flotante de chat secundario.
- **Preguntar en el chat secundario** rellena previamente el editor con un borrador `/btw` que cita el
  texto resaltado para que se pueda escribir una pregunta propia al respecto.

Ambas acciones siguen la semántica normal de `/btw`: la pregunta y la respuesta quedan fuera
del historial de la sesión y la ejecución principal no se modifica.

## Cuándo usarlo

Use `/btw` para obtener una aclaración rápida, una respuesta factual secundaria mientras una ejecución larga
sigue en curso o una respuesta temporal que no deba incorporarse al contexto futuro
de la sesión.

```text
/btw ¿qué archivo estamos editando?
/btw resume la tarea actual en una frase
/btw ¿cuánto es 17 * 19?
```

Para cualquier información que deba formar parte del contexto de trabajo futuro
de la sesión, formule la pregunta normalmente en la sesión principal.

## Temas relacionados

<CardGroup cols={2}>
  <Card title="Comandos con barra diagonal" href="/es/tools/slash-commands" icon="terminal">
    Catálogo de comandos nativos y directivas de chat.
  </Card>
  <Card title="Niveles de razonamiento" href="/es/tools/thinking" icon="brain">
    Niveles de esfuerzo de razonamiento para la llamada al modelo de la pregunta secundaria.
  </Card>
  <Card title="Sesión" href="/es/concepts/session" icon="comments">
    Claves de sesión, historial y semántica de persistencia.
  </Card>
  <Card title="Comando de dirección" href="/es/tools/steer" icon="arrow-right">
    Inyecta un mensaje de dirección en la ejecución activa sin finalizarla.
  </Card>
</CardGroup>
