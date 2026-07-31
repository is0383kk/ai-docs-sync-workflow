---
read_when:
    - Enruta los chats grupales a agentes dedicados
    - Quieres trabajar en paralelo sin que una tarea larga bloquee todos los chats
    - Está diseñando una configuración de operaciones multiagente
sidebarTitle: Specialist lanes
status: active
summary: Ejecuta agentes especializados en paralelo sin saturar la capacidad compartida del modelo y las herramientas
title: Líneas de especialistas en paralelo
x-i18n:
    generated_at: "2026-07-26T04:38:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09852b6cf5a790e98fb5e0805b0df57b2f3719b1387ecfacfb4973bb6841abb4
    source_path: concepts/parallel-specialist-lanes.md
    workflow: 16
---

Los carriles de especialistas en paralelo permiten que un Gateway dirija distintos chats o salas a
distintos agentes, a la vez que mantiene una experiencia de usuario rápida. Trate el paralelismo como
un problema de diseño con recursos escasos, no solo como «más agentes».

## Principios fundamentales

Un carril de especialista solo mejora el rendimiento cuando reduce la contención en los
cuellos de botella reales:

- **Bloqueos de sesión**: solo una ejecución debe modificar una sesión determinada a la vez.
- **Capacidad global del modelo**: todas las ejecuciones de chat visibles siguen compartiendo los límites del proveedor.
- **Capacidad de las herramientas**: el trabajo con el shell, el navegador, la red y el repositorio puede ser más lento
  que el propio turno del modelo.
- **Presupuesto de contexto**: las transcripciones largas hacen que cada turno futuro sea más lento y esté menos
  enfocado.
- **Ambigüedad de responsabilidad**: los agentes duplicados que realizan el mismo trabajo desperdician capacidad.

OpenClaw ya serializa las ejecuciones por sesión y limita el paralelismo global
mediante la [cola de comandos](/es/concepts/queue). Los carriles de especialistas añaden una política
adicional: qué agente se encarga de cada trabajo, qué permanece en el chat y qué se convierte en
trabajo en segundo plano.

## Despliegue recomendado

### Fase 1: contratos de carril + trabajo pesado en segundo plano

Proporcione a cada carril un contrato escrito en su espacio de trabajo y en el prompt del sistema:

- **Propósito**: el trabajo del que se encarga este carril.
- **Objetivos excluidos**: el trabajo que debe transferir en lugar de intentar realizar.
- **Presupuesto de chat**: las respuestas rápidas permanecen en el chat; las tareas largas se confirman brevemente
  y después se ejecutan en un subagente o una tarea en segundo plano.
- **Regla de transferencia**: cuando otro carril se encargue del trabajo, indique adónde debe dirigirse y
  proporcione un resumen de transferencia conciso.
- **Regla de riesgo de herramientas**: prefiera la superficie de herramientas más pequeña que pueda realizar el trabajo.

Esta es la fase menos costosa y resuelve la mayoría de las congestiones: un trabajo de programación ya no
convierte el carril de investigación en un proceso extremadamente lento, y cada chat mantiene limpio su propio contexto.

### Fase 2: controles de prioridad y concurrencia

Ajuste la capacidad de la cola y del modelo según el valor empresarial de cada carril:

```json5
{
  agents: {
    defaults: {
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8, delegationMode: "prefer" },
    },
  },
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,
      cap: 20,
      drop: "summarize",
    },
  },
}
```

Use chats directos o personales y agentes de operaciones de producción para el trabajo de alta prioridad. Permita que
la investigación, la redacción y la programación por lotes pasen a tareas en segundo plano cuando el sistema esté
ocupado.

### Fase 3: coordinador/controlador de tráfico

Añada un patrón de coordinación sencillo cuando haya varios carriles activos:

- Realice un seguimiento de las tareas activas de los carriles y sus responsables.
- Detecte solicitudes duplicadas entre grupos.
- Transfiera resúmenes de traspaso entre carriles.
- Muestre únicamente los bloqueos, los resultados completados y las decisiones que debe tomar la persona.

No empiece por aquí. Un coordinador sin contratos de carril solo coordina el caos.

## Plantilla mínima de contrato de carril

```md
# Contrato del carril

## Se encarga de

- <job this lane is responsible for>

## No se encarga de

- <work to hand off>

## Presupuesto de chat

- Responda directamente a las preguntas rápidas.
- Para trabajos de varios pasos, lentos o que requieran muchas herramientas: confirme brevemente, genere o ejecute en segundo plano
  el trabajo y, después, devuelva el resultado cuando se complete.

## Transferencia

Si otro carril se encarga de la solicitud, responda con:

- carril de destino
- objetivo
- contexto relevante
- siguiente acción exacta

## Enfoque de herramientas

Use la superficie de herramientas más pequeña que pueda completar la tarea. Evite el trabajo amplio con el shell o
la red, salvo que este carril se encargue explícitamente de ello.
```

## Contenido relacionado

- [Enrutamiento multiagente](/es/concepts/multi-agent)
- [Cola de comandos](/es/concepts/queue)
- [Subagentes](/es/tools/subagents)
