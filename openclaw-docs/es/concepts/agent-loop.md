---
read_when:
    - Necesita una guía paso a paso exacta del bucle del agente o de los eventos del ciclo de vida.
    - Estás cambiando el comportamiento de la cola de sesiones, las escrituras de transcripciones o el bloqueo de escritura de sesiones
summary: Ciclo de vida del bucle del agente, flujos y semántica de espera
title: Bucle del agente
x-i18n:
    generated_at: "2026-07-26T05:04:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1d0102ffb6ebf572ea0201470db138775be33b0f0b655d9d08742177be5f3f31
    source_path: concepts/agent-loop.md
    workflow: 16
---

El bucle del agente es la ejecución serializada por sesión que convierte un mensaje en
acciones y una respuesta: recepción, ensamblaje del contexto, inferencia del modelo, ejecución
de herramientas, transmisión y persistencia.

## Puntos de entrada

- RPC del Gateway: `agent` y `agent.wait`.
- CLI: `openclaw agent`.

## Secuencia de ejecución

1. La RPC `agent` valida los parámetros, resuelve la sesión (`sessionKey`/`sessionId`), conserva los metadatos de la sesión y devuelve `{ runId, acceptedAt }` inmediatamente.
2. `agentCommand` ejecuta el turno: resuelve los valores predeterminados del modelo y de pensamiento/detalle/traza, carga la instantánea de Skills, llama a `runEmbeddedAgent` y emite un evento alternativo de **fin/error del ciclo de vida** si el bucle integrado aún no ha emitido ninguno.
3. `runEmbeddedAgent`: serializa las ejecuciones mediante colas por sesión y globales, resuelve el modelo y el perfil de autenticación, crea la sesión de OpenClaw, se suscribe a los eventos del entorno de ejecución, transmite los incrementos del asistente y de las herramientas, aplica el tiempo límite de ejecución (abortándola al vencer) y devuelve las cargas útiles junto con los metadatos de uso. En los turnos del servidor de aplicaciones de Codex, también aborta un turno aceptado que deja de producir progreso del servidor de aplicaciones antes de un evento terminal.
4. `subscribeEmbeddedAgentSession` conecta los eventos del entorno de ejecución con el flujo `agent`: los eventos de herramientas con `stream: "tool"`, los incrementos del asistente con `stream: "assistant"` y los eventos del ciclo de vida con `stream: "lifecycle"` (`phase: "start" | "end" | "error"`).
5. `agent.wait` (`waitForAgentRun`) espera un **fin/error del ciclo de vida** en un `runId` y devuelve `{ status: ok|error|timeout, startedAt, endedAt, error? }`.

## Gestión de colas y concurrencia

Las ejecuciones se serializan por clave de sesión (canal de sesión) y, opcionalmente, mediante un canal global, lo que evita condiciones de carrera entre herramientas y sesiones. Los canales de mensajería eligen un modo de cola (direccionamiento/seguimiento/recopilación/interrupción) que alimenta este sistema de canales; consulte [Cola de comandos](/es/concepts/queue).

Las escrituras de la transcripción también están protegidas mediante un bloqueo de escritura de sesión en el archivo de sesión. El bloqueo tiene en cuenta los procesos y se basa en archivos, por lo que detecta los escritores que eluden la cola interna del proceso o que proceden de otro proceso. De manera predeterminada, los escritores esperan hasta 60 segundos (variable de entorno para anular este valor: `OPENCLAW_SESSION_WRITE_LOCK_ACQUIRE_TIMEOUT_MS`) antes de indicar que la sesión está ocupada.

Los bloqueos de escritura de sesión no son reentrantes de manera predeterminada. Un auxiliar que anide intencionadamente la adquisición del mismo bloqueo mientras conserva un único escritor lógico debe habilitar esta posibilidad mediante `allowReentrant: true`.

## Preparación de la sesión y del espacio de trabajo

- El espacio de trabajo se resuelve y se crea; las ejecuciones en entorno aislado pueden redirigirse a la raíz de un espacio de trabajo aislado.
- Las Skills se cargan (o se reutilizan desde una instantánea) y se inyectan en el entorno y en el prompt.
- Los archivos de arranque/contexto se resuelven y se inyectan en el prompt del sistema.
- Se adquiere un bloqueo de escritura de sesión y se prepara el destino de la transcripción de la sesión antes de iniciar la transmisión. Cualquier ruta posterior de reescritura, Compaction o truncamiento de la transcripción debe adquirir el mismo bloqueo antes de modificar las filas de la transcripción en SQLite.

## Ensamblaje del prompt

El prompt del sistema se crea a partir del prompt base de OpenClaw, el prompt de Skills, el contexto de arranque y las anulaciones específicas de cada ejecución. Se aplican los límites específicos del modelo y los tokens de reserva de Compaction. Consulte [Prompt del sistema](/es/concepts/system-prompt) para conocer lo que ve el modelo.

## Hooks

OpenClaw tiene dos sistemas de hooks:

- **Hooks internos** (hooks del Gateway): scripts controlados por eventos para comandos y eventos del ciclo de vida.
- **Hooks de Plugin**: puntos de extensión dentro del ciclo de vida del agente y las herramientas, y del Pipeline del Gateway.

### Hooks internos (hooks del Gateway)

- **`agent:bootstrap`**: se ejecuta mientras se crean los archivos de arranque, antes de finalizar el prompt del sistema. Se utiliza para añadir o eliminar archivos de contexto de arranque.
- **Hooks de comandos**: `/new`, `/reset`, `/stop` y otros eventos de comandos (consulte la documentación de Hooks).

Consulte [Hooks](/es/automation/hooks) para obtener información sobre la configuración y ejemplos.

### Hooks de Plugin

Estos se ejecutan dentro del bucle del agente o del Pipeline del Gateway:

| Hook                                                    | Se ejecuta                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `before_model_resolve`                                  | Antes de la sesión (sin `messages`), para anular de forma determinista el proveedor/modelo antes de la resolución.                                                                                                                                                                       |
| `before_prompt_build`                                   | Después de cargar la sesión (con `messages`), para inyectar `prependContext`, `systemPrompt`, `prependSystemContext` o `appendSystemContext` antes del envío. Utilice `prependContext` para texto dinámico por turno y los campos de contexto del sistema para las instrucciones estables que pertenecen al espacio del prompt del sistema. |
| `before_agent_reply`                                    | Después de las acciones en línea y antes de la llamada al LLM. Permite que un Plugin reclame el turno y devuelva una respuesta sintética o lo silencie por completo.                                                                                                                        |
| `agent_end`                                             | Después de finalizar, con la lista final de mensajes y los metadatos de la ejecución.                                                                                                                                                                                                        |
| `before_compaction` / `after_compaction`                | Observa o anota los ciclos de Compaction.                                                                                                                                                                                                                                                     |
| `before_tool_call` / `after_tool_call`                  | Intercepta los parámetros/resultados de las herramientas.                                                                                                                                                                                                                                     |
| `before_install`                                        | Después de aplicar la política de instalación del operador, sobre el material preparado para instalar Skills/Plugin, cuando los hooks de Plugin están cargados en el proceso actual.                                                                                                         |
| `tool_result_persist`                                   | Transforma de forma síncrona los resultados de las herramientas antes de escribirlos en una transcripción de sesión propiedad de OpenClaw.                                                                                                                                                   |
| `message_received` / `message_sending` / `message_sent` | Hooks de mensajes entrantes y salientes.                                                                                                                                                                                                                                                      |
| `session_start` / `session_end`                         | Límites del ciclo de vida de la sesión.                                                                                                                                                                                                                                                       |
| `gateway_start` / `gateway_stop`                        | Eventos del ciclo de vida del Gateway.                                                                                                                                                                                                                                                        |

Reglas de decisión de los hooks para los controles de salida/herramientas:

- `before_tool_call`: `{ block: true }` es terminal y detiene los controladores de menor prioridad. `{ block: false }` no realiza ninguna operación y no elimina un bloqueo anterior.
- `before_install`: tiene la misma semántica terminal/sin operación indicada anteriormente. Utilice `security.installPolicy`, no `before_install`, para las decisiones de permitir/bloquear instalaciones que pertenezcan al operador y deban abarcar las rutas de instalación y actualización de la CLI.
- `message_sending`: `{ cancel: true }` es terminal y detiene los controladores de menor prioridad. `{ cancel: false }` no realiza ninguna operación y no elimina una cancelación anterior.

Consulte [Hooks de Plugin](/es/plugins/hooks) para conocer la API de hooks y los detalles de registro.

Los arneses pueden adaptar estos hooks. El arnés del servidor de aplicaciones de Codex mantiene los hooks de Plugin de OpenClaw como contrato de compatibilidad para las superficies reflejadas documentadas; los hooks nativos de Codex son un mecanismo de Codex independiente y de menor nivel.

## Transmisión

- Los incrementos del asistente se transmiten desde el entorno de ejecución del agente como eventos `assistant`.
- La transmisión por bloques puede emitir respuestas parciales en `text_end` o `message_end`.
- La transmisión del razonamiento puede ser un flujo independiente o respuestas por bloques.
- Consulte [Transmisión](/es/concepts/streaming) para obtener información sobre la fragmentación y el comportamiento de las respuestas por bloques.

## Ejecución de herramientas

- Los eventos de inicio/actualización/fin de las herramientas se emiten en el flujo `tool`.
- Los resultados de las herramientas se depuran según el tamaño y las cargas útiles de imágenes antes de registrarlos o emitirlos.
- Se realiza un seguimiento de los envíos de las herramientas de mensajería para suprimir las confirmaciones duplicadas del asistente.

## Conformación de respuestas

Las cargas útiles finales se ensamblan a partir del texto del asistente (más el razonamiento opcional), los resúmenes de herramientas en línea (cuando se permite el modo detallado) y el texto de error del asistente cuando se produce un error en el modelo.

- El token silencioso exacto `NO_REPLY` se filtra de las cargas útiles salientes.
- Los duplicados de las herramientas de mensajería se eliminan de la lista final de cargas útiles.
- Si no quedan cargas útiles que se puedan representar y se produjo un error en una herramienta, se emite una respuesta alternativa de error de la herramienta, a menos que una herramienta de mensajería ya haya enviado una respuesta visible para el usuario.

## Compaction y reintentos

La Compaction automática emite eventos de flujo `compaction` y puede activar un reintento. Durante el reintento, los búferes en memoria y los resúmenes de herramientas se restablecen para evitar resultados duplicados. Consulte [Compaction](/es/concepts/compaction).

## Flujos de eventos

- `lifecycle`: emitido por `subscribeEmbeddedAgentSession` (y, como alternativa, por `agentCommand`).
- `assistant`: incrementos transmitidos desde el entorno de ejecución del agente.
- `tool`: eventos de herramientas transmitidos desde el entorno de ejecución del agente.

El Gateway proyecta los eventos del ciclo de vida y los eventos de inicio/terminales de las herramientas en el [registro de auditoría](/es/cli/audit) acotado,
que solo contiene metadatos. Esta proyección registra la procedencia y
los códigos de resultado sin copiar prompts, mensajes, argumentos de herramientas, resultados de herramientas
ni errores sin procesar fuera de la ruta de la transcripción o del entorno de ejecución.

## Gestión del canal de chat

Los incrementos del asistente se almacenan en búfer en mensajes `delta` del chat. Se emite un `final` de chat al producirse un **fin/error del ciclo de vida**.

## Tiempos límite

| Tiempo de espera                                   | Valor predeterminado                   | Notas                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------------------------------------------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent.wait`                                 | 30s                                    | Solo espera; el parámetro `timeoutMs` lo reemplaza. No detiene la ejecución subyacente.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Tiempo de ejecución del agente (`agents.defaults.timeoutSeconds`) | 172800s (48h)                          | Se aplica mediante el temporizador de cancelación de `runEmbeddedAgent`. Establezca `0` para disponer de un presupuesto de ejecución ilimitado; los mecanismos de vigilancia de actividad del flujo del modelo siguen aplicándose.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Mecanismo de vigilancia sin salida del backend de la CLI | calculado para cada ejecución nueva/reanudada de la CLI | Es independiente del tiempo de ejecución del agente y pertenece al plugin de backend registrado. Una tarea en segundo plano interna de la CLI comparte el subproceso principal y no sobrevive al tiempo de espera general del agente.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Turno de agente aislado de Cron                    | gestionado por Cron                     | El planificador inicia su propio temporizador cuando comienza la ejecución, cancela la ejecución al alcanzar el plazo configurado y, a continuación, realiza una limpieza acotada antes de registrar el tiempo de espera, para que una sesión secundaria obsoleta no pueda mantener bloqueada la vía de ejecución.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Tiempo de espera por inactividad del modelo       | Nube 120s; alojamiento propio 300s     | OpenClaw cancela una solicitud al modelo cuando no llega ningún fragmento de respuesta antes de que finalice el intervalo de inactividad. `models.providers.<id>.timeoutSeconds` amplía este mecanismo de vigilancia de inactividad para proveedores locales o de alojamiento propio lentos, pero sigue limitado por cualquier `agents.defaults.timeoutSeconds` finito inferior o tiempo de espera específico de la ejecución, ya que estos rigen toda la ejecución del agente. Los presupuestos de ejecución ilimitados siguen manteniendo el mecanismo de vigilancia de inactividad correspondiente a la clase de proveedor. Las ejecuciones de modelos en la nube activadas por Cron sin un tiempo de espera explícito del modelo o del agente usan el mismo valor predeterminado; con un tiempo de espera explícito de ejecución de Cron, los bloqueos del flujo del modelo en la nube se limitan a 60s para que los modelos alternativos configurados aún puedan ejecutarse antes del plazo externo de Cron. Las ejecuciones activadas por Cron en endpoints realmente locales (baseUrl de bucle invertido/privada) mantienen la exclusión del tiempo de espera por inactividad local; los proveedores de alojamiento propio con baseUrls de red reciben el mecanismo de vigilancia implícito de 300s. Con un tiempo de espera explícito de ejecución de Cron, los bloqueos locales o de alojamiento propio se limitan a dicho tiempo de espera. Establezca `models.providers.<id>.timeoutSeconds` para proveedores locales lentos. |
| Tiempo de espera de solicitud HTTP del proveedor  | `models.providers.<id>.timeoutSeconds`                     | Abarca la conexión, los encabezados, el cuerpo, el tiempo de espera de solicitudes del SDK, la gestión de cancelación de guarded-fetch y el mecanismo de vigilancia de inactividad del flujo del modelo para ese proveedor. Úselo para proveedores locales o de alojamiento propio lentos (por ejemplo, Ollama) antes de aumentar el tiempo de espera de ejecución de todo el agente; mantenga el tiempo de espera del agente o de ejecución al menos igual de alto cuando la solicitud al modelo necesite ejecutarse durante más tiempo.                                                                                                                                                                                                                                                                                                                                                                                                 |

### Diagnóstico de sesiones bloqueadas

Con el diagnóstico habilitado, un umbral integrado de dos minutos clasifica las sesiones `processing` prolongadas en las que no se ha observado progreso de respuesta, herramienta, estado, bloqueo ni ACP:

- Las ejecuciones integradas, las llamadas al modelo y las llamadas a herramientas activas se notifican como `session.long_running`. Las llamadas silenciosas al modelo con propietario permanecen como `session.long_running` hasta el umbral de cancelación, para que los proveedores lentos o sin transmisión no se marquen como bloqueados demasiado pronto.
- El trabajo activo sin progreso reciente se notifica como `session.stalled`. Las llamadas al modelo con propietario cambian a `session.stalled` al alcanzar o superar el umbral de cancelación; la actividad obsoleta de modelos o herramientas sin propietario no se oculta como una ejecución prolongada.
- `session.stuck` se reserva para los registros obsoletos recuperables de las sesiones, incluidas las sesiones inactivas en cola con actividad obsoleta de modelos o herramientas sin propietario.

El umbral de cancelación es de al menos 5 minutos y 3 veces el umbral de advertencia. Los registros obsoletos de sesión liberan la vía de ejecución de la sesión afectada inmediatamente después de superar las comprobaciones de recuperación; las ejecuciones integradas bloqueadas solo se cancelan y vacían después del umbral de cancelación, por lo que el trabajo en cola se reanuda sin interrumpir ejecuciones meramente lentas. La recuperación emite resultados estructurados de solicitud y finalización; el estado de diagnóstico se marca como inactivo solo si la misma generación de procesamiento sigue vigente, y los diagnósticos `session.stuck` repetidos aumentan progresivamente su intervalo mientras la sesión permanece sin cambios.

## Dónde puede finalizar antes de tiempo

- Tiempo de espera del agente (cancelación)
- AbortSignal (cancelación)
- Desconexión del Gateway o tiempo de espera de RPC
- Tiempo de espera de `agent.wait` (solo espera, no detiene al agente)

## Temas relacionados

- [Herramientas](/es/tools) - herramientas disponibles para el agente
- [Hooks](/es/automation/hooks) - scripts controlados por eventos que se activan mediante eventos del ciclo de vida del agente
- [Compaction](/es/concepts/compaction) - cómo se resumen las conversaciones largas
- [Aprobaciones de ejecución](/es/tools/exec-approvals) - controles de aprobación para comandos de shell
- [Pensamiento](/es/tools/thinking) - configuración del nivel de pensamiento/razonamiento
