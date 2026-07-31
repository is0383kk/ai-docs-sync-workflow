---
read_when:
    - Quieres saber si reiniciar el Gateway hace que se pierda el trabajo del agente en curso.
    - Una ejecución del agente se interrumpió debido a un reinicio, un fallo o una recarga de la configuración.
    - Está depurando la recuperación automática de la sesión después de que el gateway vuelve a estar operativo
summary: 'Qué sobrevive a un reinicio o fallo del Gateway: los turnos interrumpidos del agente se reanudan automáticamente, los subagentes y las tareas en segundo plano se recuperan, y las entregas en cola se procesan.'
title: Recuperación tras reinicio
x-i18n:
    generated_at: "2026-07-26T04:38:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdea30f3a90697951f4f63a06897d2c1d936e5145138b47fed7d8ebd8b7187ad
    source_path: gateway/restart-recovery.md
    workflow: 16
---

Reiniciar el Gateway no provoca la pérdida del estado de los agentes. Las conversaciones, las transcripciones,
los trabajos programados, los registros de tareas en segundo plano y los mensajes salientes en cola se almacenan
en disco, y el trabajo que se interrumpió durante un turno se detecta y se reanuda
automáticamente cuando el Gateway vuelve a estar activo. La recuperación está siempre habilitada y
normalmente no requiere intervención manual. La recuperación que falla repetidamente está limitada
y puede poner una sesión en cuarentena hasta que se inspeccione o sustituya.

Esta página describe qué persiste tras un reinicio, cómo se detecta el trabajo interrumpido
y cómo funciona la reanudación automática.

## Qué persiste tras un reinicio

| Estado                        | Almacenamiento                              | Comportamiento tras el reinicio                                        |
| ----------------------------- | ------------------------------------------- | ----------------------------------------------------------------------- |
| Historial de conversaciones   | Base de datos SQLite por agente             | Sin cambios; las sesiones continúan desde la transcripción almacenada   |
| Turno interrumpido de la sesión principal | Fila de sesión y transcripción SQLite por agente | Se reanuda o concilia automáticamente unos segundos después del inicio |
| Ejecuciones de subagentes     | SQLite (base de datos de estado compartida) | El registro se restaura al arrancar; las ejecuciones interrumpidas se reanudan |
| Tareas en segundo plano       | SQLite (base de datos de estado compartida) | Se concilian al arrancar; las ejecuciones huérfanas se recuperan o se marcan como perdidas |
| Entregas salientes en cola    | Cola de entregas SQLite                     | Se procesan tras el reinicio; las respuestas no entregadas se reintentan |
| Trabajos programados (Cron)   | Almacén Cron de SQLite                      | Las programaciones persisten; el programador se reactiva al arrancar    |
| Continuación tras el reinicio | Centinela de reinicio SQLite                | Se envía una continuación de una sola ejecución a la sesión que solicitó el reinicio |

## Los reinicios controlados primero esperan a que termine el trabajo

Un reinicio solicitado (`openclaw gateway restart`, un cambio de configuración que requiere
un reinicio o una actualización del Gateway) no termina inmediatamente el trabajo en curso. El
Gateway deja de aceptar trabajo nuevo y espera a que finalicen los turnos de agente activos y
las tareas en segundo plano, hasta agotar un margen de espera (5 minutos de forma predeterminada). Por tanto, la mayoría
de los reinicios no interrumpen ningún trabajo.

Solo se cancela el trabajo que no puede finalizar dentro del margen de espera (o cualquier ejecución interrumpida
por un reinicio forzado o un fallo), y antes de que esto ocurra, cada
sesión afectada se marca para su recuperación.

## Cómo se detecta el trabajo interrumpido

Tres mecanismos complementarios marcan las sesiones cuyo turno no finalizó:

- **Al admitir el turno:** para un turno de texto normal en una sesión principal existente,
  el Gateway añade el mensaje del usuario, marca la sesión como en ejecución y registra
  su solicitud de entrega para recuperación en una única transacción SQLite antes de ejecutar el modelo o
  el hook `before_agent_reply`. La interfaz de control hace esto antes de devolver la
  confirmación `started`; el envío del canal lo hace cuando el turno preparado
  adopta la ejecución del agente.
  Los comandos, los archivos adjuntos, las anulaciones por turno, las entregas pendientes, las indicaciones de cancelaciones
  anteriores, las sesiones propiedad de plugins y los turnos con hooks de ejecución mantienen sus
  rutas de admisión especializadas.
  Si hay instalado un hook `before_agent_reply`, la admisión también registra su fase.
  La recuperación nunca vuelve a ejecutar un hook interrumpido durante una llamada. Una vez que finaliza un hook
  no gestionado, su punto de control registra ese resultado, pero la recuperación sigue cerrándose
  ante errores mientras dicho hook permanezca activo: un punto de control no puede demostrar que el mismo
  código y configuración del plugin se cargaron después del reinicio. Los resultados de texto gestionados y
  los resultados silenciosos se registran en puntos de control separados para una resolución determinista.
  Las solicitudes de recuperación persistentes escritas por versiones anteriores no tienen un marcador
  de propiedad del origen, por lo que reciben la misma comprobación de cierre ante errores del hook durante una actualización.
- **Durante el apagado:** mientras se espera a que termine el trabajo antes del reinicio, cada sesión con una ejecución activa
  recibe un marcador de recuperación en el almacén de sesiones antes de que se
  cancele la ejecución.
- **Durante el inicio:** el Gateway examina los almacenes de sesiones en busca de sesiones que aún
  afirman estar en ejecución, pero que no tienen un propietario activo en el nuevo proceso. Esto detecta
  fallos críticos y terminaciones en los que no se ejecutó ningún código de apagado. Los archivos de bloqueo
  obsoletos de las transcripciones se eliminan al mismo tiempo.

## Reanudación automática

Unos segundos después del inicio, el Gateway vuelve a enviar cada sesión marcada
con un mensaje de sistema sintético que informa al agente de que su turno anterior se
interrumpió debido a un reinicio y le indica que continúe desde la transcripción existente. Si ya
se había generado una respuesta final, pero no se había entregado, su texto se incluye
para que el agente pueda entregarla en lugar de repetir el trabajo.

La conciliación durante el inicio reintenta los errores transitorios hasta tres veces con
retroceso exponencial. Por separado, cada ciclo interrumpido de la sesión principal dispone de un
límite persistente de tres intentos contabilizados de envío automático, conservado entre
reinicios del Gateway. OpenClaw contabiliza un intento antes del envío, lo reembolsa cuando
el Gateway rechaza explícitamente la solicitud antes de aceptarla y mantiene el
cargo cuando el resultado posterior al envío es incierto para evitar volver a ejecutar el trabajo.
El trabajo en primer plano que ya posee la sesión impide la recuperación automática
hasta que dicho trabajo termine.

Cuando se agota el límite persistente, la sesión se marca con una lápida en lugar de
continuar en un bucle indefinido. Inspeccione la sesión fallida y utilice `/new` o `/reset` para iniciar una
sesión de sustitución. `openclaw doctor --fix` puede reparar una marca de cancelación obsoleta que
entre en conflicto con una lápida, pero no vuelve a habilitar ese ciclo de recuperación.

Cada reintento reutiliza un único identificador persistente de envío, por lo que un fallo de conexión
ambiguo no puede iniciar dos veces la misma recuperación. Los turnos completados y no reanudables de la interfaz de
control también conservan lápidas de idempotencia persistentes y limitadas, lo que permite que
una bandeja de salida que se vuelve a conectar los retire sin volver a ejecutar la solicitud.

Las respuestas que solo utilizan la herramienta de mensajes emplean una segunda correlación persistente. Antes de que un envío
terminal a la misma conversación llegue al canal, el Gateway registra una intención de
entrega sin resolver para la sesión y el turno de origen exactos. Un éxito confirmado del proveedor
la convierte en un recibo persistente de entrega; un fallo confirmado la elimina.
La recuperación completa un recibo entregado sin volver a ejecutar herramientas. Si un fallo
deja sin determinar el resultado del proveedor, la recuperación se cierra ante errores en lugar de volver a ejecutar
un efecto externo.

La respuesta entregada también se refleja en la transcripción con el ID de su mensaje
de origen. Los reflejos terminales utilizan una clave de recibo distinta, por lo que un envío de progreso con
la misma clave de idempotencia del proveedor no puede ocultar el marcador terminal. Los envíos
de progreso y los recibos de turnos anteriores no pueden completar el turno actual. Solo
las solicitudes persistentes de entrada del canal pueden restaurar la autoridad para realizar acciones de mensajes. Una ejecución reanudada
conserva el modo de entrega de origen y la correlación de origen originales, incluida
la identidad del solicitante y cualquier restricción del mismo canal o hilo, por lo que el mismo recibo
sigue siendo autoritativo aunque se produzca otro reinicio durante la recuperación. Un
turno que solo utiliza la herramienta de mensajes y carece de una autoridad de canal reconstruible se cierra
ante errores y recibe el aviso único para volver a enviar la solicitud.

Antes de reanudar, el Gateway comprueba que sea seguro continuar desde el final de
la transcripción. Si no lo es (por ejemplo, si el turno terminó con una aprobación pendiente
obsoleta), la sesión no se vuelve a ejecutar a ciegas; en su lugar, el agente publica un breve
aviso que solicita al usuario volver a enviar la última solicitud. En WebChat, ese aviso se
escribe directamente en el historial de la sesión para que siga visible después de volver a conectarse.

OpenClaw también puede reconstruir trabajo interrumpido de solo lectura del [Modo de código](/es/tools/code-mode).
El Modo de código marca estas ejecuciones como seguras ante reinicios y rechaza las herramientas del
catálogo con efectos secundarios o los espacios de nombres de plugins antes de que se ejecuten. Si se produce un reinicio en
el control `wait`, el nuevo Gateway reconstruye el turno desde su transcripción
y obliga a que la ejecución reconstruida siga siendo segura ante reinicios, aunque el
modelo omita o elimine esa marca. El host limita todo el turno reconstruido
a herramientas principales de solo lectura auditadas y herramientas de plugins explícitamente seguras para su repetición,
incluso si el Modo de código se deshabilita después del reinicio. El trabajo con efectos secundarios
sigue protegido por el aviso para volver a enviar la solicitud, en lugar de arriesgarse a duplicar una escritura.

### Subagentes

Las ejecuciones de subagentes se conservan en la base de datos de estado SQLite compartida, por lo que el
registro de subagentes persiste entre procesos. Al arrancar, el registro se restaura y
las sesiones de subagentes interrumpidas se reanudan con el contexto de su tarea original.
Se aplican dos mecanismos de seguridad:

- Las ejecuciones interrumpidas hace más de 2 horas se finalizan en lugar de reanudarse, para que
  un Gateway que haya estado inactivo durante la noche no reactive trabajo obsoleto.
- Una sesión que falla repetidamente al recuperarse se marca con una lápida como bloqueada, para que
  la recuperación no pueda continuar en un bucle indefinido.

### Tareas en segundo plano

El [registro de tareas en segundo plano](/es/automation/tasks) utiliza SQLite y
se concilia al arrancar y a intervalos periódicos: se recuperan los resultados persistentes registrados por
las ejecuciones finalizadas, y las ejecuciones cuyo proceso propietario desapareció se
marcan como perdidas después de un periodo de gracia, en lugar de quedar bloqueadas indefinidamente.

### Reinicios solicitados por el agente

Cuando el propio agente activa un reinicio (al aplicar un cambio de configuración, actualizar
el Gateway o solicitar explícitamente un reinicio), se escribe un centinela de reinicio en
SQLite antes de que finalice el proceso. Después del arranque, el Gateway publica el resultado en
el chat de origen y envía un turno de continuación de una sola ejecución para que el
agente retome el trabajo exactamente donde lo dejó, en el mismo canal e hilo.

Las columnas SQLite tipadas del centinela son la fuente autoritativa para gestionar el reinicio;
su valor `payload_json` es solo una copia secundaria para repetición y depuración. El entorno de ejecución lee, escribe
y elimina el estado de SQLite sin recurrir a archivos. Durante la transición del almacenamiento, se
ejecuta una migración de estado limitada durante el inicio y mediante Doctor para conservar un
`restart-sentinel.json` validado que el proceso anterior dejó después de una actualización.
La migración verifica la fila tipada y elimina el archivo de origen antes de que continúe
la gestión normal del reinicio.

## Mecanismos de seguridad y observabilidad

- **Interruptor de bucles de fallos:** 3 arranques no controlados en un periodo de 5 minutos activan un interruptor que
  impide el inicio automático de servicios auxiliares durante el siguiente arranque, para que un Gateway que falla
  no amplifique el problema. Se recupera cuando finaliza el periodo de arranques no controlados.
- **Límite de intentos de la sesión principal:** tres intentos contabilizados de envío automático
  por ciclo interrumpido; al agotarse, la sesión se marca con una lápida hasta que se
  inspeccione y sustituya.
- **Métricas:** la actividad de recuperación se exporta mediante
  [Prometheus](/es/gateway/prometheus) como `openclaw_session_recovery_total` y
  `openclaw_session_recovery_age_seconds`.
- **Registros:** las decisiones de recuperación se registran en los
  subsistemas `main-session-restart-recovery` y `subagent-interrupted-resume`.

## Qué no se reanuda

- Sesiones excluidas de la recuperación de la sesión principal porque ya las gestiona otro
  propietario: sesiones de subagentes (recuperación de subagentes), sesiones de Cron (el
  programador vuelve a ejecutarlas según la programación) y sesiones gestionadas por ACP (el IDE
  o cliente conectado controla la reanudación).
- Sesiones cuyo final de transcripción no permite continuar de forma segura; estas reciben el
  aviso para volver a enviar la solicitud descrito anteriormente, en lugar de volver a ejecutarse silenciosamente.
- Trabajo que nunca se admitió: los mensajes que llegan durante el periodo de espera previo al reinicio
  se rechazan con un error explícito de reinicio, en lugar de añadirse silenciosamente a la cola de un
  proceso que está finalizando.
- Los turnos integrados independientes no pueden tomar el control de una sesión principal con una recuperación
  pendiente tras un reinicio porque no comparten el propietario del ciclo de vida del Gateway.
  Ejecute el turno mediante el Gateway o restablézcalo allí con `/new` o `/reset`.
