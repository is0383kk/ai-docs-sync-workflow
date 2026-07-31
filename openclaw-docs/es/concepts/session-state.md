---
read_when:
    - Se desea que los agentes detecten cuando personas u otros agentes cambian una sesión sin su conocimiento.
    - Está depurando avisos de cambio de estado, cursores de observación o cambios de session_status changesSince
    - Quiere comprender cómo los agentes principales se mantienen sincronizados con las sesiones secundarias
sidebarTitle: Session state awareness
summary: 'Registro de señales del estado de sesión persistente: versiones del estado, observadores, avisos de estado obsoleto y reconciliación'
title: Conocimiento del estado de la sesión
x-i18n:
    generated_at: "2026-07-26T05:06:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb4126a0802e1ca4418f225c792490493a78886089b81c3b4567f72090ce34f4
    source_path: concepts/session-state.md
    workflow: 16
---

Cuando varias sesiones trabajan en el mismo problema —un gestor que delega en sesiones secundarias, una persona que interviene directamente en una sesión de trabajo, dos agentes que se coordinan mediante [`sessions_send`](/es/concepts/session-tool)—, cada sesión genera suposiciones sobre las demás. Esas suposiciones quedan obsoletas en cuanto interviene otro actor. El reconocimiento del estado de las sesiones es el mecanismo que detecta la intervención, avisa una vez a la sesión afectada y le proporciona una forma económica de ponerse al día antes de actuar.

Tres elementos trabajan conjuntamente:

1. Un **registro de señales duradero** registra determinados cambios de estado de cada sesión.
2. Los **observadores** mantienen cursores por destino y reciben un único aviso consolidado de estado obsoleto.
3. La **reconciliación** obtiene el delta exacto mediante `session_status` con `changesSince`.

## El registro de señales

OpenClaw añade un evento tipado a la base de datos de estado compartida (`session_state_events`) cuando una sesión observada cambia de forma significativa. Los eventos incluyen metadatos y un resumen de una línea, pero nunca el contenido de los mensajes.

| Tipo                   | Cuándo se registra                                            | Notifica a los observadores |
| ---------------------- | -------------------------------------------------------- | ----------------- |
| `human_direct_message` | Una persona envía un turno directamente a una sesión observada       | Sí               |
| `upstream_missing`     | Desaparece la fuente ascendente de una sesión adoptada          | Sí               |
| `goal_changed`         | Se crea, actualiza o borra el estado del objetivo de la sesión | Sí               |
| `child_spawned`        | Se crea una sesión secundaria de un subagente o ACP              | No (inicializa el cursor) |
| `run_completed`        | Una ejecución secundaria finaliza correctamente                            | No (solo registro)     |
| `run_failed`           | Una ejecución secundaria falla, agota el tiempo de espera o se cancela            | No (solo registro)     |
| `compacted`            | Se compacta el historial de la sesión                       | No (solo registro)     |
| `adopted`              | Se adopta en OpenClaw una sesión del catálogo               | No (solo registro)     |

Cada evento identifica a su actor (`human`, `agent` o `system`). Las ejecuciones secundarias canceladas o que agotan el tiempo de espera se registran como fallos, conservando el resultado exacto (`cancelled`, `timeout` o `error`) en la carga útil del evento.

La **versión de estado** de una sesión es simplemente el número de secuencia más alto de su registro, que se conserva en una cabecera duradera por sesión que sobrevive a la depuración. Las filas de `sessions_list` incluyen `stateVersion` cuando una sesión ha registrado cambios; `session_status` siempre lo comunica.

Los tipos de solo registro existen para el historial de reconciliación, no para las notificaciones: la entrega normal de la finalización de ejecuciones secundarias sigue siendo responsabilidad de los [anuncios de subagentes](/es/tools/subagents), y el registro de señales nunca la duplica.

## Observadores

Un observador es una sesión que mantiene un cursor (`session_watch_cursors`) sobre un destino. Los cursores proceden de dos lugares:

- **Implícitos (relaciones de creación).** Cuando una sesión crea un subagente o una sesión secundaria de ACP, el cursor de la sesión superior se inicializa automáticamente en la versión de creación de la sesión secundaria. Las sesiones superiores nunca se suscriben manualmente.
- **Explícitos (`sessions_send watch: true`).** Cualquier coordinador puede observar un destino que no haya creado: pase `watch: true` a `sessions_send` y, después de que el envío se despache correctamente, el remitente se registrará como observador de la sesión que haya recibido realmente el mensaje. El registro comienza en la versión de estado actual del destino; el historial anterior nunca genera avisos. El resultado de la herramienta comunica `watched: true|false` cuando se ha establecido el parámetro.

La identidad del observador debe ser una clave de sesión cualificada por agente. En `session.scope="global"`, la clave compartida `global` es ambigua entre agentes, por lo que dichas sesiones reciben el registro duradero y `changesSince`, pero no avisos proactivos.

Las observaciones se limpian automáticamente: las filas de cursores caducan con la retención del registro de señales, se eliminan cuando se restablece la sesión observadora y se borran junto con cualquiera de las sesiones. En v1 no existe ninguna acción para dejar de observar.

Las sesiones observadas adoptadas desde un catálogo de sesiones se comprueban con una cadencia fija para detectar actividad humana directa en la fuente ascendente. La actividad detectada entra en el mismo registro de señales y flujo de observadores que los demás turnos humanos directos.

Si la fuente ascendente de una sesión adoptada se elimina externamente, tres comprobaciones consecutivas sin encontrarla (aproximadamente tres ciclos de supervisión) generan una señal `upstream_missing` para sus observadores y eliminan el vínculo ascendente. Volver a continuar la sesión del catálogo crea un vínculo nuevo.

## Avisos: uno, no muchos

Cuando se registra un evento que admite notificaciones y el cursor de un observador está atrasado, el observador recibe un aviso del sistema en su siguiente turno:

```
La sesión "agent:main:subagent:child" ha cambiado (otro actor). Reconcilie antes de actuar: session_status sessionKey "agent:main:subagent:child" changesSince 12.
```

Los observadores de sesiones principales también se activan inmediatamente mediante una activación de Heartbeat; los observadores de subagentes anidados reciben el aviso en su siguiente turno.

El protocolo está diseñado expresamente para evitar mensajes repetitivos:

- **Un aviso pendiente por cada par de observador y destino.** El texto del aviso permanece idéntico byte por byte mientras está pendiente y la cola de eventos del sistema lo desduplica, por lo que veinte cambios rápidos en el mismo destino siguen produciendo una sola línea en el prompt del observador.
- **Marca de agua fija.** El cursor congela la posición notificada cuando se pone un aviso en cola. Los eventos significativos posteriores solo hacen avanzar la marca de agua significativa; no generan otra notificación.
- **Confirmación al extraer; reapertura solo si hay trabajo intercalado.** Cuando el turno del observador consume el aviso, el cursor avanza. Si llegaron más eventos significativos entre la puesta en cola y la extracción, se abre exactamente un aviso nuevo para los eventos restantes.
- **Supresión de eventos propios.** Un observador nunca recibe notificaciones sobre eventos que él mismo haya provocado.
- **Recuperación tras reinicios.** Los avisos pendientes residen en una cola en memoria; un barrido al iniciar los vuelve a materializar a partir de los cursores duraderos tras reiniciar el Gateway.

## Reconciliación

El aviso indica al observador exactamente qué debe hacer. `session_status` con `changesSince: <version>` devuelve los eventos tipados posteriores a esa versión (hasta 200), sin hacer avanzar ningún cursor:

```json
{
  "stateVersion": 19,
  "stateChanges": {
    "events": [
      {
        "sequence": 14,
        "kind": "human_direct_message",
        "actorType": "human",
        "summary": "mensaje humano mediante telegram"
      },
      { "sequence": 19, "kind": "goal_changed", "actorType": "human", "summary": "objetivo actualizado" }
    ],
    "historyGap": false
  }
}
```

`historyGap: true` significa que la versión solicitada es anterior al historial conservado; actualice todo el estado de la sesión (`sessions_history`, `session_status`) en lugar de tratar la respuesta como un delta exacto. La señal de discontinuidad es exacta: procede de una marca de agua depurada por sesión, no se deduce mediante aritmética de secuencias.

## Almacenamiento y límites

El historial reside en la base de datos de estado compartida y está limitado a 30 días y 50,000 filas; las cabeceras por sesión siguen siendo monotónicas después de la depuración. El registro se realiza según el mejor esfuerzo: si una adición falla, el error se registra y nunca provoca el fallo del turno que la originó. Por tanto, `stateVersion` es una cabecera del registro de señales, no una versión transaccional de captura de cambios de datos.

Límites actuales:

- La entrega de avisos presupone que un proceso del Gateway controla la base de datos de estado compartida. Varios Gateways comparten el registro duradero y `changesSince`, pero v1 no envía avisos entre procesos.
- Los eventos de Compaction abarcan los propietarios de Compaction del entorno de ejecución integrado; la Compaction exclusiva del arnés nativo no se registra por completo.
- Actualmente, las ejecuciones secundarias de ACP generan los detalles de la carga útil de los resultados cancelados; las cancelaciones de subagentes nativos aparecen como fallos genéricos.
- La detección de eco propio ascendente compara texto normalizado de usuario. Un prompt externo que coincida con uno de los 10 mensajes de usuario más recientes del lado de OpenClaw se considera un eco propio.
- Una sola fila JSONL local de Claude que supere el límite de análisis de 1 MiB por ciclo bloquea el cursor de esa sesión en v1; nunca se omiten bytes sin clasificar.
- Las comprobaciones de Claude en nodos emparejados clasifican los últimos 50 elementos de la transcripción por ciclo. Las ráfagas mayores pueden quedar fuera de la ventana de análisis de v1.
- Las lecturas del historial de Claude en nodos emparejados no exponen un resultado definitivo de hilo no encontrado, por lo que las eliminaciones remotas de Claude no se clasifican como `upstream_missing` en v1.
- Las sesiones del catálogo que no se hayan adoptado permanecen fuera de la capa de reconocimiento en v1.
- Las sesiones adoptadas antes de esta función no tienen ningún vínculo ascendente; continúelas una vez desde el catálogo para iniciar la supervisión ascendente.
- Los vínculos ascendentes presuponen que cada clave de sesión adoptada corresponde a un único agente propietario (la adopción utiliza el agente predeterminado del almacén). La adopción por varios agentes del mismo hilo externo no se supervisa en v1.

## Temas relacionados

- [Herramientas de sesión](/es/concepts/session-tool) — `sessions_send`, `session_status`, `sessions_list`
- [Subagentes](/es/tools/subagents) — relaciones de creación y anuncios de finalización
- [Heartbeat](/es/gateway/heartbeat) — cómo los avisos en cola activan las sesiones principales
- [Gestión de sesiones](/es/concepts/session) — claves, ámbitos y ciclo de vida de las sesiones
