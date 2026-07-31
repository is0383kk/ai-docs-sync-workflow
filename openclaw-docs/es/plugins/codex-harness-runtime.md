---
read_when:
    - Necesita el contrato de compatibilidad del entorno de ejecución del arnés de Codex
    - Estás depurando herramientas nativas de Codex, hooks, Compaction o la carga de comentarios
    - Está cambiando el comportamiento del plugin en los turnos del entorno de pruebas de OpenClaw y Codex
summary: Límites del entorno de ejecución, hooks, herramientas, permisos y diagnósticos para el arnés de Codex
title: Entorno de ejecución del arnés de Codex
x-i18n:
    generated_at: "2026-07-26T04:45:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d18d42683df0d827b776547f7b45f60f572cf39410d00533f53f8fdcdccb0d2
    source_path: plugins/codex-harness-runtime.md
    workflow: 16
---

Contrato de runtime para los turnos del arnés de Codex. Para la configuración y el enrutamiento, consulte
[Arnés de Codex](/es/plugins/codex-harness). Para los campos de configuración, consulte
[Referencia del arnés de Codex](/es/plugins/codex-harness-reference).

## Descripción general

Codex controla el bucle nativo del modelo, la reanudación nativa de hilos, la
continuación nativa de herramientas y la Compaction nativa. OpenClaw controla el
enrutamiento de canales, los archivos de sesión, la entrega de mensajes visibles,
las herramientas dinámicas de OpenClaw, las aprobaciones, la entrega de contenido
multimedia y un reflejo de la transcripción en torno a ese límite.

El enrutamiento del prompt sigue el runtime seleccionado, no solo la cadena del
proveedor. Un turno nativo de Codex recibe las instrucciones de desarrollador del
servidor de aplicaciones de Codex; una ruta explícita de compatibilidad de OpenClaw
mantiene el prompt del sistema habitual de OpenClaw incluso cuando utiliza
autenticación o transporte de OpenAI con características de Codex.

OpenClaw inicia y reanuda hilos nativos de Codex con la personalidad integrada de
Codex desactivada (`personality: "none"`) para que los archivos de personalidad del
espacio de trabajo y la identidad del agente de OpenClaw sigan siendo la autoridad.
Por lo demás, Codex nativo mantiene las instrucciones base y del modelo controladas
por Codex, así como la carga de la documentación del proyecto. Las ejecuciones
ligeras de OpenClaw (por ejemplo, cron) siguen suprimiendo la carga de la
documentación del proyecto.

Las instrucciones de desarrollador de OpenClaw abarcan aspectos del runtime de
OpenClaw: la entrega al canal de origen, las herramientas dinámicas de OpenClaw,
la delegación ACP, el contexto del adaptador y los archivos de perfil activos del
espacio de trabajo del agente. Los catálogos de Skills y los punteros
`MEMORY.md` enrutados mediante herramientas se proyectan como instrucciones
de desarrollador para la colaboración limitadas al turno. Cuando las herramientas
de memoria no están disponibles, el contenido activo de `BOOTSTRAP.md` y todo
`MEMORY.md` se incorporan en su lugar como contexto de entrada de texto
sin formato para el turno.

La mayoría de las herramientas dinámicas de OpenClaw utilizan el espacio de nombres
consultable `openclaw`. Las herramientas marcadas con
`catalogMode: "direct-only"` utilizan `openclaw_direct`, que Codex mantiene directamente
visible para el modelo como `DirectModelOnly`, en lugar de exponerlo a la ejecución
anidada del modo de código.

## Vinculaciones de hilos y cambios de modelo

Cuando una sesión de OpenClaw está asociada a un hilo existente de Codex, el
siguiente turno vuelve a enviar al servidor de aplicaciones el modelo seleccionado
actualmente, la política de aprobación, el entorno aislado, el revisor de
aprobaciones y el nivel de servicio. Al cambiar de `openai/gpt-5.5` a
`openai/gpt-5.2`, se mantiene la vinculación del hilo, pero se solicita a Codex
que continúe con el modelo recién seleccionado.

Las vinculaciones supervisadas son la excepción. El selector de modelos de OpenClaw
permanece bloqueado y las reanudaciones omiten las anulaciones del modelo y el
proveedor para que Codex restaure el modelo y el proveedor persistentes del hilo
canónico. Un control nativo independiente de Codex puede cambiar ese par persistente,
y la instantánea inicial puede generar la advertencia habitual de Codex sobre la
diferencia de modelo; el modelo externo de OpenClaw y la cadena de reserva nunca
sustituyen a ninguno de los dos.

## Supervisión y continuación segura

La supervisión de Codex es una capacidad opcional del mismo Plugin
`codex`. Descubre hilos nativos mediante una conexión independiente y
solo proyecta las sesiones no archivadas en el catálogo del Gateway. Sin una
configuración de conexión explícita de `appServer`, esa conexión utiliza
stdio administrado en el directorio principal del usuario, mientras que el arnés
ordinario sigue limitado al agente. Las operaciones de listado y lectura de
metadatos son pasivas: no reanudan ningún hilo, no suscriben OpenClaw a sus eventos
en vivo ni responden a sus solicitudes de aprobación.

Para una sesión almacenada o inactiva en el equipo del Gateway, **Continuar como rama**
crea un Chat normal con el modelo bloqueado y refleja un historial limitado de
mensajes del usuario y del asistente hasta el último turno terminal persistente
del origen. El primer turno normal de Chat instala los controladores de aprobación
reales y utiliza una bifurcación nativa temporal para fijar la instantánea sin
anular el modelo ni el proveedor. Codex App Server utiliza su configuración nativa
actual y devuelve el par seleccionado; emite su advertencia habitual si ese modelo
difiere del último modelo registrado del origen. En la misma conexión de
supervisión, OpenClaw inicia el hilo canónico del arnés de Codex de origen
`appServer` conforme a su directorio de trabajo y su política de runtime,
con exactamente el modelo y el proveedor devueltos para ese inicio inicial, inyecta
el historial visible limitado y archiva la bifurcación temporal. El origen nunca
se reanuda. El hilo canónico dispone de toda la superficie de herramientas del
arnés de OpenClaw; el razonamiento, las llamadas a herramientas y los resultados
de herramientas del origen no se clonan en él. El ámbito de la conexión privada
se conserva durante los estados de vinculación pendiente y confirmada, por lo que
cada turno posterior permanece en esa conexión con la autenticación nativa y la
configuración del proveedor. Si la supervisión está desactivada o se produce una
divergencia en la vinculación o la conexión, se aplica un cierre seguro en lugar
de cambiar al arnés ordinario del directorio principal del agente.

El origen original de la CLI, VS Code, Atlas o ChatGPT sigue siendo apto para ambos
catálogos. La rama canónica es un hilo nativo de Codex, pero su tipo de origen es
`appServer`; los clientes nativos pueden filtrar ese tipo de origen, por lo
que no se garantiza que aparezca en Codex Desktop.

Los orígenes activos no pueden iniciar una rama nueva ni archivarse; sí se puede
abrir un Chat supervisado existente. `notLoaded` significa que la actividad
es desconocida, no que esté inactivo; OpenClaw solo permite archivar una fila local
`idle` o `notLoaded` tras una confirmación explícita de que no hay
otro ejecutor y una lectura reciente del estado local del proceso. Codex serializa
las mutaciones de hilos dentro de un proceso de App Server, pero no proporciona un
arrendamiento exclusivo entre procesos para el ejecutor ni para el responsable de
las aprobaciones, por lo que esa lectura no puede demostrar que otro proceso no
esté usando el hilo. OpenClaw bloquea a un propietario de vinculación que se sabe
activo para el destino exacto o para cualquier descendiente generado y no archivado
que devuelva la consulta paginada de descendientes de Codex. Los errores de
enumeración, los ciclos y el agotamiento del límite de seguridad provocan un cierre
seguro. El archivado nativo todavía puede entrar en conflicto con un turno nuevo
de otro proceso, por lo que la confirmación abarca los clientes desconocidos y el
intervalo entre la lectura del estado y el archivado. Un Chat supervisado con el
modelo bloqueado no puede eliminarse mientras proteja la vinculación nativa.

Los catálogos de nodos emparejados se limitan a metadatos en la versión inicial.
El límite actual de invocación del Node es de solicitud/respuesta y no puede
transportar los eventos de turno de larga duración, las solicitudes de aprobación
ni la salida en streaming que requiere una vinculación real del arnés de Codex.
Por tanto, las acciones remotas **Continuar** y **Archivar** siguen sin estar
disponibles incluso cuando la fila está inactiva.

Consulte [Supervisión de Codex](/es/plugins/codex-supervision) para conocer la
configuración del operador y el comportamiento visible de la interfaz de control.

## Respuestas visibles y Heartbeat

Los turnos de chat directo o de origen mediante el arnés de Codex utilizan de forma
predeterminada la entrega final automática del asistente para las superficies
internas de WebChat, de acuerdo con el contrato del arnés de Pi: el agente responde
con normalidad y OpenClaw publica el texto final en la conversación de origen.
Establezca `messages.visibleReplies: "message_tool"` para mantener privado el texto final del asistente,
salvo que el agente llame a `message(action="send")`.

Los turnos de Heartbeat de Codex incluyen `heartbeat_respond` de forma predeterminada
en el catálogo consultable de herramientas de OpenClaw para que el agente pueda
registrar si la activación debe permanecer silenciosa o generar una notificación.
Las directrices sobre la iniciativa de Heartbeat se envían como una instrucción de
desarrollador del modo de colaboración de Codex limitada al turno de Heartbeat; los
turnos de chat ordinarios permanecen en el modo predeterminado de Codex. Cuando
`HEARTBEAT.md` no está vacío, las instrucciones de Heartbeat remiten a Codex al
archivo en lugar de incluir su contenido directamente.

## Límites de los hooks

| Capa                                  | Propietario              | Finalidad                                                                    |
| ------------------------------------- | ------------------------ | --------------------------------------------------------------------------- |
| Hooks de plugins de OpenClaw          | OpenClaw                 | Compatibilidad de productos/plugins entre OpenClaw y los arneses de Codex.  |
| Middleware de extensión de Codex App Server | Plugins incluidos de OpenClaw | Comportamiento del adaptador por turno en torno a las herramientas dinámicas de OpenClaw. |
| Hooks nativos de Codex                | Codex                    | Ciclo de vida de bajo nivel de Codex y política de herramientas nativas procedente de la configuración de Codex. |

OpenClaw no utiliza archivos `hooks.json` globales ni del proyecto de Codex
para enrutar el comportamiento de los plugins. Para el puente de herramientas y
permisos nativos, OpenClaw inyecta la configuración de Codex por hilo para
`PreToolUse`, `PostToolUse`, `PermissionRequest` y
`Stop`.

Cuando las aprobaciones de Codex App Server están habilitadas
(`approvalPolicy` no es `"never"`), la configuración predeterminada
inyectada de hooks nativos omite `PermissionRequest` para que el revisor de App
Server de Codex y el puente de aprobación de OpenClaw gestionen las escaladas reales
después de la revisión. Añada `permission_request` a `nativeHookRelay.events` para forzar
de todos modos el relé de compatibilidad. Otros hooks de Codex, como
`SessionStart` y `UserPromptSubmit`, siguen siendo controles del nivel de
Codex; no se exponen como hooks de plugins de OpenClaw en el contrato v1.

En el caso de las herramientas dinámicas de OpenClaw, OpenClaw ejecuta la
herramienta después de que Codex solicita la llamada, por lo que el comportamiento
del plugin y del middleware se ejecuta en el adaptador del arnés. El modo de código
de Codex recibe los resultados dinámicos genéricos como texto y serializa las
llamadas dinámicas anidadas; los llamadores deben analizar los resultados que
parezcan JSON y no pueden depender de `Promise.all` para el envío simultáneo.
En el caso de las herramientas nativas de Codex, Codex controla el registro
canónico de la herramienta; OpenClaw puede reflejar eventos seleccionados, pero no
puede reescribir el hilo nativo salvo que Codex lo exponga mediante App Server o
devoluciones de llamada de hooks nativos.

Los eventos `PreToolUse` del modo de informe de Codex App Server aplazan la
aprobación del plugin hasta la aprobación correspondiente de App Server. Si un
hook `before_tool_call` de OpenClaw devuelve `requireApproval` mientras que la
carga nativa establece `openclaw_approval_mode:
"report"`, el relé del hook nativo registra el
requisito de aprobación del plugin y no devuelve ninguna decisión nativa. Cuando
Codex envía posteriormente la solicitud de aprobación de App Server para el mismo
uso de la herramienta, OpenClaw abre la solicitud de aprobación del plugin y
asigna la decisión de vuelta a Codex. Los eventos `PermissionRequest` de Codex
constituyen una ruta de aprobación independiente y aún pueden enrutarse mediante
las aprobaciones de OpenClaw cuando se configuran para ese puente.

Las notificaciones de elementos de Codex App Server también proporcionan
observaciones asíncronas de `after_tool_call` para las finalizaciones de
herramientas nativas que aún no estén cubiertas por el relé nativo
`PostToolUse`. Estas observaciones solo sirven para telemetría y
compatibilidad; no pueden bloquear, retrasar ni modificar la llamada a la
herramienta nativa.

Las proyecciones de Compaction y del ciclo de vida del LLM proceden de las
notificaciones de Codex App Server y del estado del adaptador de OpenClaw, no de
los comandos de hooks nativos de Codex. `before_compaction`,
`after_compaction`, `llm_input` y `llm_output` son observaciones
del nivel del adaptador, no capturas byte por byte de las cargas internas de
solicitud o Compaction de Codex.

Las notificaciones nativas `hook/started` y `hook/completed` de Codex App
Server se proyectan como eventos de agente `codex_app_server.hook` para el análisis de
trayectorias y la depuración. No invocan hooks de plugins de OpenClaw.

## Contrato de compatibilidad de V1

Compatible con el runtime v1 de Codex:

| Superficie                                       | Compatibilidad                                                                          | Motivo                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Bucle de modelos de OpenAI mediante Codex               | Compatible                                                                        | El app-server de Codex gestiona el turno de OpenAI, la reanudación nativa de hilos y la continuación nativa de herramientas.                                                                                                                                                                                                                                                                                                                                                                                          |
| Enrutamiento y entrega de canales de OpenClaw         | Compatible                                                                        | Telegram, Discord, Slack, WhatsApp, iMessage y otros canales permanecen fuera del entorno de ejecución del modelo.                                                                                                                                                                                                                                                                                                                                                                                    |
| Herramientas dinámicas de OpenClaw                        | Compatible                                                                        | Codex solicita a OpenClaw que ejecute estas herramientas, por lo que OpenClaw permanece en la ruta de ejecución.                                                                                                                                                                                                                                                                                                                                                                                                |
| Plugins de prompt y contexto                    | Compatible                                                                        | OpenClaw proyecta el prompt y el contexto específicos de OpenClaw en el turno de Codex, mientras mantiene los prompts base, del modelo y de documentación del proyecto configurados, que son propiedad de Codex, en la vía nativa de Codex. OpenClaw desactiva la personalidad integrada de Codex para los hilos nativos, de modo que los archivos de personalidad del espacio de trabajo del agente sigan siendo la fuente autoritativa. Las instrucciones nativas de desarrollador de Codex solo aceptan orientación sobre comandos cuyo ámbito se limite explícitamente a `codex_app_server`; las indicaciones globales heredadas sobre comandos se mantienen para superficies de prompts ajenas a Codex. |
| Ciclo de vida del motor de contexto                      | Compatible                                                                        | El ensamblaje, la ingesta y el mantenimiento posterior al turno se ejecutan en torno a los turnos de Codex. Los motores de contexto no sustituyen la Compaction nativa de Codex.                                                                                                                                                                                                                                                                                                                                                        |
| Hooks de herramientas dinámicas                            | Compatible                                                                        | `before_tool_call`, `after_tool_call` y el middleware de resultados de herramientas se ejecutan en torno a las herramientas dinámicas propiedad de OpenClaw.                                                                                                                                                                                                                                                                                                                                                                          |
| Hooks del ciclo de vida                               | Compatible como observaciones del adaptador                                                | `llm_input`, `llm_output`, `agent_end`, `before_compaction` y `after_compaction` se activan con cargas útiles fieles al modo Codex.                                                                                                                                                                                                                                                                                                                                                           |
| Puerta de revisión de la respuesta final                    | Compatible mediante la retransmisión de hooks nativos                                              | El `Stop` de Codex se retransmite a `before_agent_finalize`; `revise` solicita a Codex una pasada más del modelo antes de finalizar.                                                                                                                                                                                                                                                                                                                                                                |
| Bloqueo u observación nativos del shell, los parches y MCP | Compatible mediante la retransmisión de hooks nativos                                              | Los `PreToolUse` y `PostToolUse` de Codex se retransmiten para las superficies de herramientas nativas confirmadas, incluidas las cargas útiles de MCP en la versión `0.142.0` o posterior del app-server de Codex. Se admite el bloqueo; no se admite la reescritura de argumentos.                                                                                                                                                                                                                                                                               |
| Política de permisos nativos                      | Compatible mediante las aprobaciones del app-server de Codex y la retransmisión compatible de hooks nativos | Las solicitudes de aprobación del app-server de Codex se enrutan a través de OpenClaw después de la revisión de Codex. La retransmisión del hook nativo `PermissionRequest` es opcional para los modos de aprobación nativos porque Codex lo emite antes de la revisión del guardián.                                                                                                                                                                                                                                                                          |
| Captura de la trayectoria del app-server                 | Compatible                                                                        | OpenClaw registra la solicitud que envió al app-server y las notificaciones que recibe de este.                                                                                                                                                                                                                                                                                                                                                                                    |

No compatible con el entorno de ejecución Codex v1:

| Superficie                                             | Límite de V1                                                                                                                                     | Ruta futura                                                                               |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Mutación de argumentos de herramientas nativas                       | Los hooks nativos previos a las herramientas de Codex pueden bloquear, pero OpenClaw no reescribe los argumentos de las herramientas nativas de Codex.                                               | Requiere compatibilidad del hook o esquema de Codex para sustituir la entrada de la herramienta.                            |
| Historial editable de transcripciones nativas de Codex            | Codex gestiona el historial canónico de hilos nativos. OpenClaw gestiona un reflejo y puede proyectar contexto futuro, pero no debe modificar componentes internos no compatibles. | Añadir API explícitas del app-server de Codex si se necesita intervenir en los hilos nativos.                    |
| `tool_result_persist` para registros de herramientas nativas de Codex | Ese hook transforma las escrituras de transcripciones gestionadas por OpenClaw, no los registros de herramientas nativas de Codex.                                                           | Se podrían reflejar los registros transformados, pero la reescritura canónica requiere compatibilidad de Codex.              |
| Metadatos enriquecidos de Compaction nativa                     | OpenClaw puede solicitar la Compaction nativa, pero no recibe una lista estable de elementos conservados o descartados, la diferencia de tokens, un resumen de finalización ni la carga útil del resumen.   | Requiere eventos de Compaction más completos de Codex.                                                     |
| Intervención en la Compaction                             | OpenClaw no permite que los plugins ni los motores de contexto veten, reescriban o sustituyan la Compaction nativa de Codex.                                             | Añadir hooks de Codex anteriores y posteriores a la Compaction si los plugins necesitan vetar o reescribir la Compaction nativa. |
| Captura byte por byte de solicitudes a la API del modelo             | OpenClaw puede capturar las solicitudes y notificaciones del app-server, pero el núcleo de Codex genera internamente la solicitud final a la API de OpenAI.                      | Requiere un evento de seguimiento de solicitudes del modelo o una API de depuración de Codex.                                   |

## Permisos nativos y solicitudes de información de MCP

Para `PermissionRequest`, OpenClaw solo devuelve decisiones explícitas de
permitir o denegar cuando la política toma una decisión. Un resultado sin
decisión no equivale a permitir: Codex lo trata como si el hook no hubiera
tomado una decisión y recurre a su propio guardián o a la ruta de aprobación
del usuario.

Los modos de aprobación del app-server de Codex omiten este hook nativo de
forma predeterminada. Esto se aplica salvo que `permission_request` se incluya
explícitamente en `nativeHookRelay.events` o que un entorno de ejecución de
compatibilidad lo instale.

Cuando un operador elige `allow-always` para una solicitud de permiso
nativo de Codex, OpenClaw recuerda esa huella digital exacta de
proveedor/sesión/entrada de herramienta/cwd durante un intervalo de sesión
limitado. La decisión recordada solo se aplica intencionadamente a coincidencias
exactas: un cambio en el comando, los argumentos, la carga útil de la
herramienta o el cwd genera una nueva aprobación.

Las solicitudes de aprobación de herramientas MCP de Codex se enrutan a través
del flujo de aprobación de plugins de OpenClaw cuando Codex marca
`_meta.codex_approval_kind` como `"mcp_tool_call"`. El `request_user_input` de Codex
registra una pregunta del Gateway independiente del proveedor para la sesión de
origen. La interfaz de control muestra la tarjeta de pregunta del Gateway, y
una única opción no secreta utiliza botones de canal tipados cuando el canal es
compatible. Las pulsaciones de botones, las respuestas de la interfaz de
control y la siguiente respuesta de texto sin formato en cola resuelven el
mismo registro del Gateway antes de que OpenClaw devuelva la respuesta del
app-server. La resolución automática de Codex y las cancelaciones de intentos
limitan la espera y cancelan el registro. Las preguntas secretas permanecen
íntegramente en la ruta advertida de respuesta por texto. Las demás solicitudes
de información de MCP se rechazan de forma segura.

Para consultar el flujo general de aprobación de plugins que transporta estas
solicitudes, véase [Solicitudes de permisos de plugins](/es/plugins/plugin-permission-requests).

## Dirección de la cola

La dirección de la cola de ejecuciones activas se asigna a `turn/steer` del servidor de aplicaciones de Codex. Con el
valor predeterminado `messages.queue.mode: "steer"`, OpenClaw agrupa los mensajes de chat
en modo de dirección durante el intervalo de inactividad configurado y los envía como una sola solicitud `turn/steer`
en orden de llegada.

Los turnos de revisión y Compaction manual de Codex pueden rechazar la dirección en el mismo turno. En
ese caso, OpenClaw espera a que termine la ejecución activa antes de iniciar el
prompt. Use `/queue followup` o `/queue collect` cuando los mensajes deban ponerse en cola
de forma predeterminada en lugar de dirigir la ejecución. Consulte [Cola de dirección](/es/concepts/queue-steering).

## Carga de comentarios de Codex

Cuando se aprueba `/diagnostics [note]` para una sesión en el entorno de ejecución nativo de Codex,
OpenClaw también llama a `feedback/upload` del servidor de aplicaciones de Codex para los hilos
de Codex pertinentes, incluidos los registros de cada hilo enumerado y los subhilos
de Codex generados cuando están disponibles.

La carga pasa por la ruta normal de comentarios de Codex hacia los servidores de OpenAI. Si
los comentarios de Codex están deshabilitados en ese servidor de aplicaciones, el comando devuelve el
error del servidor de aplicaciones. La respuesta de diagnóstico completada enumera los canales,
los identificadores de sesión de OpenClaw, los identificadores de hilo de Codex y los comandos locales `codex resume <thread-id>`
de los hilos enviados.

Si se deniega o ignora la aprobación, OpenClaw no muestra esos identificadores de Codex
ni envía comentarios de Codex. La carga no sustituye la exportación local
de diagnósticos del Gateway. Consulte [Exportación de diagnósticos](/es/gateway/diagnostics) para conocer
el comportamiento de la aprobación, la privacidad, el paquete local y el chat grupal.

Use `/codex diagnostics [note]` solo cuando desee cargar los comentarios de Codex
del hilo conectado actualmente sin el paquete completo de diagnósticos del Gateway.

## Compaction y réplica de la transcripción

Cuando el modelo seleccionado utiliza el entorno de ejecución de Codex, la Compaction nativa del hilo
corresponde al servidor de aplicaciones de Codex. OpenClaw no ejecuta una Compaction preliminar en
los turnos de Codex, no sustituye la Compaction de Codex por la del motor de contexto ni
recurre a la generación de resúmenes de OpenClaw o de la API pública de OpenAI cuando no se puede
iniciar la Compaction nativa. OpenClaw mantiene una réplica de la transcripción para el historial del canal, la búsqueda,
`/new`, `/reset` y futuros cambios de modelo o entorno de ejecución.

Las solicitudes explícitas de Compaction, como `/compact` o una operación compact manual
solicitada por un Plugin, inician la Compaction nativa de Codex con `thread/compact/start`.
OpenClaw mantiene abiertos la solicitud y el arrendamiento del cliente compartido hasta que Codex emite el
elemento de finalización `contextCompaction` correspondiente y, a continuación, informa que el turno de
Compaction se ha completado. Si ese turno terminal supera el tiempo de espera de Compaction
configurado, OpenClaw solicita una interrupción nativa del turno. El arrendamiento y el bloqueo de
Compaction por hilo se mantienen hasta que Codex informa del estado terminal o confirma
la RPC de interrupción. Si Codex no la confirma durante el período de gracia de la
interrupción, OpenClaw retira la conexión antes de liberar el bloqueo. Las conexiones
remotas también desconectan la vinculación del hilo correspondiente para que el trabajo posterior no pueda
solaparse con un turno remoto sin confirmar. Los demás turnos de una conexión retirada fallan
y pueden volver a intentarse con un cliente nuevo. El cierre del cliente, la cancelación de la solicitud o un
turno de Compaction fallido devuelven una operación fallida. La Compaction automática por presión
del contexto es responsabilidad de Codex; OpenClaw solo inicia la Compaction nativa para los desencadenadores
solicitados manualmente.

Cuando un motor de contexto solicita la proyección de inicialización de un hilo de Codex, OpenClaw
proyecta los nombres e identificadores de las llamadas a herramientas, las estructuras de entrada y el contenido
censurado de los resultados de las herramientas en el nuevo hilo de Codex. No copia los valores sin procesar de los argumentos
de las llamadas a herramientas en esa proyección.

La réplica incluye el prompt del usuario, el texto final del asistente y registros ligeros
del razonamiento o del plan de Codex cuando el servidor de aplicaciones los emite. OpenClaw
registra el inicio y el estado terminal de la Compaction nativa, pero no
expone un resumen de la Compaction legible para personas ni una lista auditable de las
entradas que Codex conservó después de la Compaction.

Como Codex controla el hilo nativo canónico, `tool_result_persist` no
reescribe los registros de resultados de herramientas nativos de Codex. Solo se aplica cuando OpenClaw
escribe el resultado de una herramienta en una transcripción de sesión controlada por OpenClaw.

## Contenido multimedia y entrega

OpenClaw sigue controlando la entrega de contenido multimedia y la selección del proveedor multimedia. Las imágenes,
los vídeos, la música, los PDF, la TTS y la comprensión de contenido multimedia utilizan los ajustes correspondientes
del proveedor o modelo, como `agents.defaults.mediaModels.image`,
`agents.defaults.mediaModels.video`, `pdfModel` y `tts`.

El texto, las imágenes, el vídeo, la música, la TTS, las aprobaciones y la salida de las herramientas de mensajería continúan
por la ruta normal de entrega de OpenClaw; la generación de contenido multimedia no requiere
el entorno de ejecución heredado. Cuando Codex emite un elemento nativo de generación de imágenes con un
`savedPath`, OpenClaw reenvía ese archivo exacto mediante la ruta normal de contenido multimedia
de respuesta aunque el turno de Codex no contenga texto del asistente.

## Temas relacionados

- [Entorno de ejecución de Codex](/es/plugins/codex-harness)
- [Referencia del entorno de ejecución de Codex](/es/plugins/codex-harness-reference)
- [Supervisión de Codex](/es/plugins/codex-supervision)
- [Plugins nativos de Codex](/es/plugins/codex-native-plugins)
- [Hooks de Plugins](/es/plugins/hooks)
- [Plugins de entornos de ejecución de agentes](/es/plugins/sdk-agent-harness)
- [Exportación de diagnósticos](/es/gateway/diagnostics)
- [Exportación de trayectorias](/es/tools/trajectory)
