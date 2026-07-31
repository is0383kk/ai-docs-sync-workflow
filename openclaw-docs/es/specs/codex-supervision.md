---
read_when:
    - Diseño del comportamiento de descubrimiento, continuación o archivado de sesiones de Codex
    - Cambiar la interfaz de usuario nativa del catálogo de sesiones o los RPC del Gateway
    - Ampliación de la supervisión de Codex entre nodos emparejados
summary: Arquitectura y límites del producto para supervisar sesiones nativas de Codex desde OpenClaw.
title: Supervisión de Codex
x-i18n:
    generated_at: "2026-07-26T05:29:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e259badc8f7fdec6fa093785a1dd04394e12287ae61f00474bcd45e7b95352d
    source_path: specs/codex-supervision.md
    workflow: 16
---

# Supervisión de Codex

## Objetivo

La supervisión de Codex permite que un operador de OpenClaw descubra sesiones nativas de Codex y,
cuando sea seguro, cree una rama local mediante la interfaz habitual de Chat de OpenClaw.
Codex App Server sigue siendo el propietario del hilo y del bucle del modelo. OpenClaw proporciona el
catálogo de la flota, la interfaz de operador autenticada, la vinculación de sesiones y la entrega por canales.

La función pertenece al plugin oficial `codex`. No existe un
plugin Supervisor independiente ni una segunda implementación del protocolo Codex.

## Límite del producto

El catálogo se registra siempre que el plugin Codex está activo, salvo que el descubrimiento
de sesiones nativas se deshabilite explícitamente con:

```text
plugins.entries.codex.config.sessionCatalog.enabled = false
```

Habilite las herramientas de supervisión orientadas a agentes con:

```text
plugins.entries.codex.config.supervision.enabled = true
```

El producto inicial activo es intencionadamente más reducido que el plan de flota
a largo plazo:

- Mostrar solo los hilos de Codex no archivados.
- Agrupar las filas locales y de nodos emparejados que hayan aceptado participar según la identidad estable del host.
- Crear una rama de Chat normal y bloqueada al modelo a partir de un hilo almacenado o inactivo local
  al Gateway, iniciar su hilo completo del entorno de Codex en el primer turno o abrir el Chat
  creado para una rama anterior.
- Archivar un hilo almacenado o inactivo local al Gateway solo tras la confirmación explícita
  de que no hay otro ejecutor.
- Mostrar las fuentes locales activas sin controles de nueva rama ni archivado, pero permitir
  que se abra un Chat supervisado existente.
- Mostrar las filas más recientes de cada host en la barra lateral principal, mantener el catálogo completo en
  la página de sesiones y proporcionar lecturas de transcripciones acotadas y paginadas mediante cursor para
  las filas locales y de nodos emparejados.
- Aislar por host los fallos del catálogo.

El catálogo es la colección de elementos no archivados. Una fila de este todavía puede tener un
estado de turno inactivo, activo, `notLoaded` o de error.

La supervisión orientada a agentes sigue siendo opcional. La incorporación guiada intenta instalarla y habilitarla
después de que la detección de la instalación nativa de Codex se complete correctamente y el backend de inferencia
seleccionado supere su comprobación en vivo, independientemente del backend principal que seleccione el usuario.
La supervisión se activa únicamente cuando esa configuración oportunista del plugin
se completa correctamente. Un plugin deshabilitado explícitamente, un bloqueo de política o
`supervision.enabled: false` siguen teniendo autoridad sobre las herramientas de supervisión, pero
no deshabilitan el catálogo de sesiones del operador. `sessionCatalog.enabled: false`
deshabilita el descubrimiento del operador y los comandos de catálogo de nodos emparejados; el
proveedor y el entorno de Codex permanecen activos.

## Propiedad

El plugin `codex` es propietario de todo el comportamiento de Codex App Server:

- descubrimiento de endpoints y ciclo de vida de la conexión
- inicialización del protocolo y comprobaciones de versión
- listado, lectura, reanudación, archivado y gestión de eventos de hilos
- puentes de aprobación y entrada del usuario
- vinculaciones de hilos nativos con sesiones de OpenClaw
- aplicación exclusiva del modelo y el entorno de Codex tras la continuación

La interfaz de control y el Gateway consumen ese servicio propiedad del plugin. No leen
directamente los archivos de despliegue de Codex ni implementan otro cliente de App Server.

La topología local predeterminada es:

```text
Codex Desktop -> App Server stdio privado -> directorio principal de Codex del usuario
                                             ^
Plugin Codex de OpenClaw -> conexión de App Server para supervisión
  (de forma predeterminada, stdio administrado del directorio principal del usuario; se respetan los ajustes explícitos de appServer)
  -> catálogo pasivo de fuentes y lectura
  -> fijación de instantánea -> rama canónica de origen appServer
  -> inyección del historial visible y todos los turnos posteriores del Chat supervisado

Sesiones ordinarias de Codex de OpenClaw -> stdio administrado del directorio principal del agente de forma predeterminada
  -> hilos completos ordinarios del entorno -> Chat de OpenClaw y entrega por canales
```

Habilitar la supervisión no cambia el entorno ordinario de Codex: de forma predeterminada, sigue
teniendo el ámbito del agente. La conexión de supervisión independiente utiliza de forma predeterminada
stdio administrado del directorio principal del usuario, por lo que sus operaciones de catálogo e instantáneas ven los hilos nativos
almacenados. Se respetan los ajustes explícitos de conexión `appServer`. Cuando
`homeScope` no está definido, la conexión de supervisión lo resuelve como `"user"` para stdio
o Unix y como `"agent"` para WebSocket. Defina `appServer.homeScope: "user"`
explícitamente solo cuando el entorno ordinario también deba compartir el directorio principal nativo de
Codex. Un Chat adoptado desde el grupo de Codex de la barra lateral es la excepción: su vinculación privada
de supervisión mantiene las lecturas de origen, la creación de la rama canónica y los turnos
posteriores en la conexión de supervisión. El estado y la propiedad en vivo siguen siendo
locales al proceso; un hilo desconocido para el proceso de supervisión de OpenClaw es `notLoaded`
incluso cuando Codex Desktop lo está ejecutando activamente.

Codex dispone de un daemon local canónico experimental con un contrato de arranque independiente
administrado por el instalador. Esta función no debe arrancar, atribuirse ni
dar por supuesto implícitamente ese daemon.

## Flujo del catálogo

El método genérico del Gateway `sessions.catalog.list` despacha al proveedor de catálogo `codex`,
que siempre solicita `archived: false` y permite que App Server
aplique su valor predeterminado de fuentes interactivas: `cli`, `vscode`, Atlas y ChatGPT. Combina:

1. Los resultados `thread/list` locales al Gateway del App Server de supervisión,
   que utiliza de forma predeterminada stdio administrado del directorio principal del usuario.
2. Los resultados `codex.appServer.threads.list.v1` de cada nodo conectado que haya aceptado participar.

La selección de transcripciones utiliza `thread/turns/list` con `itemsView: "full"` localmente o
el comando versionado `codex.appServer.thread.turns.list.v1` en el nodo
seleccionado. Cada respuesta contiene como máximo 20 turnos persistentes, además de cursores opacos
hacia delante y hacia atrás. La interfaz de control solicita primero las páginas más recientes, representa cada página en
orden cronológico y antepone las páginas más antiguas. Nunca recurre a un
`thread/read` sin límites. OpenClaw también rechaza cualquier página de elementos serializada de más de
20 MiB antes de que pueda atravesar el transporte del nodo o del Gateway.

La implementación nativa de nodos emparejados de macOS solo admite un valor no definido/predeterminado o
`appServer.transport: "stdio"` explícito con un ámbito de supervisión no definido/predeterminado o
`appServer.homeScope: "user"` explícito. Transmite `command`, `args`
configurados y `clearEnv` normalizado al proceso secundario. Con `"unix"`, `"websocket"`
o `homeScope: "agent"` explícito, no anuncia ni la capacidad
ni el comando del catálogo; la invocación directa también falla de forma segura. Nunca debe exponer el directorio principal
de Codex del usuario para una configuración con ámbito de agente ni sustituir un endpoint explícito por stdio
local.

La proyección del catálogo normaliza los identificadores, el título, cwd, el estado, los indicadores de espera
activa, las marcas de tiempo, la fuente, el proveedor del modelo, la versión de Codex y la rama de Git. No
devuelve vistas previas de transcripciones, turnos, rutas de despliegue, rutas del directorio principal de Codex,
remotos de Git, SHA de commits, endpoints sin procesar ni errores sin procesar de App Server. Las respuestas de
transcripciones solo contienen la página de elementos de App Server solicitada explícitamente y sus
cursores opacos.

Los fallos de los hosts permanecen aislados en el resultado de cada host. Un nodo sin conexión o un
App Server local no disponible no eliminan de la página los hosts que funcionan correctamente. La conectividad es una
propiedad del host, no un estado del hilo: el resultado de un host con errores no contiene filas
de sesiones recientes ni proyecta `offline` en los hilos nativos.

La interfaz de control solicita actualizaciones progresivas del catálogo. Cada host local o emparejado
aparece cuando finaliza el listado de su propio App Server; la respuesta agregada sigue siendo
la instantánea de compatibilidad y recuperación. La página visible se reconcilia después de
cambios de conectividad, al recibir el foco y, como máximo, cada 30 segundos, con una pasada más rápida
después de los cambios. Por tanto, las sesiones nativas de Codex creadas en otro cliente
terminan por descubrirse sin importarlas al almacenamiento de OpenClaw.

El descubrimiento del catálogo es pasivo. El listado o la lectura de metadatos no deben llamar a
`thread/resume`, suscribir el cliente de OpenClaw a solicitudes de hilos en vivo ni
responder a una aprobación.

La búsqueda se realiza solo por título y no distingue entre mayúsculas y minúsculas. Para cada página del catálogo devuelta, el
Gateway y el Mac emparejado examinan un número acotado de páginas nativas sin pasar
la consulta a App Server, ya que la búsqueda nativa también puede encontrar coincidencias en vistas previas de
transcripciones. El cursor nativo devuelto permite que los llamadores continúen el examen.

## Límite de la CLI del operador

El plugin registra tres comandos de shell respaldados por el Gateway:

```text
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [gateway-options]
openclaw codex continue <thread-id> [--json] [gateway-options]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [gateway-options]
```

`[gateway-options]` es `--url <url>`, `--token <token>`, `--timeout <ms>` y
la opción heredada `--expect-final`. El listado de sesiones tiene un valor predeterminado de 75,000 ms;
la continuación y el archivado tienen un valor predeterminado de 30,000 ms;
`--expect-final` no tiene ningún efecto adicional en estas RPC unarias. La búsqueda de sesiones
se realiza solo por título y no distingue entre mayúsculas y minúsculas; cada respuesta examina una cadena acotada de páginas
nativas y `--cursor` continúa con resultados más antiguos. El límite predeterminado es de 50 por host
y acepta valores de 1 a 100; un cursor requiere un único destino `--host`
estable. Ningún comando acepta
una opción de archivados/incluir archivados. Solo `sessions` puede dirigirse a hosts emparejados;
`continue` y `archive` siempre envían `hostId: "gateway:local"`, y el archivado
requiere el indicador de confirmación explícita.

El espacio de nombres del shell no es el espacio de nombres de ejecución `/codex` dentro del Chat. En
particular, `/codex sessions --host <node>` enumera los archivos de sesiones de la CLI de Codex en un
nodo, `/codex threads` enumera los hilos de App Server para la conexión de la conversación
actual y `/codex resume` o `/codex bind` modifican la vinculación de esa conversación.
Esos comandos no sustituyen a `sessions.catalog.continue`, y no existe ningún comando de ejecución
`/codex continue` ni `/codex archive`.

## Continuación local

Para una fila almacenada o inactiva local al Gateway, la interfaz llama a
`sessions.catalog.continue` con `catalogId: "codex"`, además de los identificadores del host y del
hilo. El plugin:

1. Reutiliza el Chat supervisado existente cuando la fuente ya tiene uno.
2. De lo contrario, proyecta un historial acotado del usuario y del asistente hasta el último
   turno terminal persistente de la fuente (completado, interrumpido o fallido) en un nuevo
   Chat de OpenClaw y registra una rama pendiente del entorno.
3. Almacena la política pendiente de bloqueo exclusivo a modelos de Codex, no una selección concreta de
   modelo o proveedor, además del ámbito de la conexión privada de supervisión, y
   devuelve el `sessionKey` de OpenClaw.

La proyección del historial selecciona el tramo final más reciente de mensajes visibles del usuario y del asistente,
con límites estrictos de 200 mensajes, 512 KiB de texto UTF-8 en total y
64 KiB por mensaje. Sustituye las entradas de imagen y de imagen local por
`[Image attachment]`, nunca copia cargas útiles ni rutas de imágenes y omite el razonamiento,
las llamadas a herramientas y los resultados de herramientas.

La interfaz navega al Chat normal con esa clave de sesión. Todavía no existe ningún hilo canónico
del entorno. En el primer turno normal del Chat, el entorno instala los controladores reales
de aprobación, obtención de información, eventos y entrega de Codex y, a continuación:

1. Utiliza la conexión de supervisión para llamar a `thread/fork` nativo sin una invalidación del modelo
   ni del proveedor y fija la instantánea persistente de la fuente. El estado actual
   `ConfigManager` de Codex selecciona el modelo y el proveedor, y la respuesta de la bifurcación
   informa del par real. Si el modelo difiere del último modelo registrado
   en la fuente, Codex emite su advertencia habitual de diferencia de modelo.
2. En esa misma conexión, inicia el hilo canónico completo del entorno de Codex con
   `threadSource: "appServer"`, el cwd, la política, la configuración y el entorno de OpenClaw,
   toda la superficie de herramientas del entorno de OpenClaw y exactamente el modelo y el proveedor
   devueltos por la bifurcación para este inicio inicial.
3. Inyecta el historial acotado y visible del usuario y del asistente mediante esa
   conexión, confirma la vinculación canónica sin descartar su ámbito de supervisión,
   ejecuta el turno y archiva la bifurcación temporal.

Antes del primer turno, el Chat es una rama pendiente bloqueada con un espejo
visible del historial; después, cada turno del modelo se ejecuta mediante el hilo
canónico del arnés de Codex en la conexión de supervisión. La rama no es un clon
completo del rollout nativo: se omiten deliberadamente el razonamiento de origen,
las llamadas a herramientas y los resultados de las herramientas. Si falla la
fijación de la instantánea o la creación del hilo canónico, la rama pendiente
sigue permitiendo reintentos. Una carrera de vinculación, la supervisión
deshabilitada o una conexión de supervisión no disponible o incompatible hacen
que se produzca un cierre seguro antes de ejecutar el turno, en lugar de recurrir
al arnés ordinario del directorio principal del agente.

Esto garantiza la selección gestionada por Codex, no la conservación del modelo
histórico del origen. El par devuelto por el fork se utiliza para iniciar el hilo
canónico, y Codex conserva el modelo y el proveedor nativos de ese hilo. Las
reanudaciones posteriores omiten las anulaciones de modelo y proveedor de
OpenClaw, por lo que Codex restaura el par conservado. Si un control nativo de
Codex independiente cambia el hilo canónico, OpenClaw acepta esa selección nativa
conservada. El modelo externo de OpenClaw y la cadena de respaldo nunca lo
sustituyen.

Los cambios de modelo, la eliminación de sesión y las operaciones de
restablecimiento o creación de sesión producen un cierre seguro para el Chat
supervisado con el modelo bloqueado. Modificar `/codex model <model>`, `/codex
bind`, `/codex resume` (incluido el Node `--bind here`) y `/codex detach` o
`/codex unbind` también produce un cierre seguro porque estas operaciones
reemplazan o eliminan la vinculación. La consulta `/codex model` y
`/codex fast`, `/codex permissions` y `/codex
threads` siguen disponibles. La herramienta de agente `codex_threads` no puede adjuntar un
fork nuevo ni archivar el hilo nativo vinculado. La lectura de listas y solo
metadatos sigue disponible; los campos de transcripción requieren
`supervision.allowRawTranscripts`, mientras que el cambio de nombre, la desarchivación, el fork
desvinculado y el archivado de un hilo no relacionado requieren
`supervision.allowWriteControls`. Ninguna opción puede reemplazar la vinculación bloqueada.
De lo contrario, eliminar o restablecer la entrada de OpenClaw descartaría la
vinculación nativa y crearía o permitiría un hilo genérico tras una sesión que
parecería de Codex. Por tanto, el mantenimiento de retención conserva las
entradas con el modelo bloqueado incluso cuando superan los límites ordinarios
de antigüedad, cantidad o presupuesto de disco. Deshabilitar o desinstalar el
Plugin propietario también conserva el bloqueo y el marcador de propiedad del
Plugin. El Chat permanece no disponible y produce un cierre seguro hasta que se
vuelve a habilitar el mismo Plugin; la limpieza nunca lo convierte en una sesión
de modelo ordinaria.

Esta acción nunca reanuda ni modifica el origen. El fork temporal fija una
instantánea; no es el hilo de continuación duradero. Iniciar un hilo distinto
del arnés canónico en el primer turno impide que OpenClaw se convierta en un
escritor de origen competidor simplemente porque el estado local del proceso no
detectó un turno gestionado por Desktop. El espejo visible del historial y la
instantánea fijada pueden omitir trabajo que aún no haya finalizado en un origen
activo. El origen original de CLI, VS Code, Atlas o ChatGPT sigue siendo apto
tanto para los catálogos nativos como para los de OpenClaw. La rama canónica
sigue siendo un hilo nativo de Codex en el almacén de supervisión, pero los
clientes nativos pueden filtrar su tipo de origen `appServer`, por lo que
la visibilidad en Codex Desktop no forma parte del contrato.

## Comportamiento del archivado

Para una fila almacenada o inactiva local del Gateway, `sessions.catalog.archive` con
`catalogId: "codex"` requiere
`confirmNoOtherRunner: true` explícito, vuelve a leer el estado local actual del proceso,
solo continúa con `idle` o `notLoaded`, llama a `thread/archive` nativo
y devuelve éxito únicamente después de que Codex acepte la operación. Después,
la fila deja de aparecer en el catálogo no archivado.

Un estado activo o de error en la nueva lectura rechaza el archivado. También lo
hace una rama supervisada en inicialización o pendiente del origen: el primer
turno del Chat debe materializar su rama canónica antes de que pueda archivarse
el origen. Un propietario conocido de una vinculación activa de OpenClaw para el
destino exacto o cualquier descendiente generado no archivado también rechaza
el archivado. OpenClaw pagina la relación experimental
`thread/list ancestorThreadId` de Codex y produce un cierre seguro ante errores de
solicitud o respuesta, ciclos de cursores o hilos y agotamiento del límite de
seguridad. El archivado nativo puede detener el trabajo cargado del elemento
principal y sus descendientes, por lo que el archivado no es un atajo para
interrumpir. Las llamadas de lectura, enumeración de descendientes y archivado
no son atómicas. Un cliente independiente todavía puede poseer o iniciar trabajo
en una fila que parezca inactiva o `notLoaded` localmente. La confirmación de
que no hay otro ejecutor abarca los clientes desconocidos y esa carrera hasta
que Codex disponga de un archivado condicional o un arrendamiento entre procesos.
Se prohíbe el archivado mediante Node emparejado.

No hay una vista archivada en el catálogo de Codex. Un hilo restaurado con
`thread/unarchive` en otra superficie de Codex autorizada por el propietario
vuelve a ser apto para el catálogo no archivado.

## Seguridad de los hilos activos

Codex serializa las modificaciones de un hilo entre los clientes de un único App
Server, pero no expone un arrendamiento exclusivo de ejecutor entre procesos ni
de propietario de aprobaciones. App Servers stdio independientes pueden añadir
contenido al mismo rollout, mientras cada uno solo ve su propio estado en
memoria. Las solicitudes de aprobación también pueden llegar a todos los
suscriptores de un servidor, y la primera respuesta válida completa la solicitud.

Por tanto:

- los clientes pasivos del catálogo no se suscriben a las aprobaciones ni las rechazan automáticamente
- las filas indicadas actualmente como activas no exponen ni una rama nueva ni Archive
- un origen sin asignar se convierte en una rama con historial visible cuyo hilo
  del arnés canónico nunca reanuda el origen
- `notLoaded` se muestra como actividad desconocida y solo puede archivarse tras
  confirmar de manera informada que no hay otro ejecutor
- el archivado local requiere esa confirmación más una nueva lectura de `idle` o `notLoaded`,
  a la vez que se reconoce la carrera del protocolo entre la lectura y el archivado

La interrupción y la transferencia entre varios clientes son decisiones futuras
del producto. No quedan implícitas por mostrar una fila activa.

## Límite del Node emparejado

Actualmente, la invocación de Node solo admite solicitud y respuesta. Puede
devolver de forma segura metadatos acotados del catálogo y páginas de turnos de
transcripción, pero no puede transportar el flujo de eventos de larga duración,
las solicitudes de aprobación, las llamadas a herramientas, la cancelación y
los deltas del asistente que requiere una ejecución del arnés de Codex.

Por tanto, el contrato del Node admite listas y páginas de turnos de
transcripción. Las filas remotas siguen siendo legibles, pero **Continue** y
**Archive** no están disponibles, independientemente del estado inactivo. Una
continuación remota real requiere un ejecutor en el Node y un puente de
streaming que conserven las mismas invariantes de aprobación y vinculación que el
arnés local.

## Permisos

Cada equipo otorga su consentimiento localmente. Habilitar el Gateway no
autoriza a otro Node a leer sus metadatos de Codex. La capacidad del Node debe
superar el emparejamiento normal y la aprobación de la política de comandos.

La enumeración de la flota y la visualización de transcripciones utilizan el
ámbito `operator.write` del Gateway porque invocan Nodes emparejados. La
continuación y el archivado locales son acciones autenticadas del operador y
siguen sujetos a comprobaciones del host y del estado.

El acceso autónomo del agente y el acceso MCP independiente son cuestiones
separadas. Los contratos de herramientas distribuidos
`codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` y `codex_session_interrupt` siguen perteneciendo
al Plugin `codex`. Con la supervisión habilitada, las lecturas de
transcripciones sin procesar de `codex_threads` y los campos de lista derivados
de transcripciones también requieren `supervision.allowRawTranscripts`; cada fork,
cambio de nombre, archivado o desarchivación de `codex_threads`
requiere `supervision.allowWriteControls`. Ambas políticas están deshabilitadas de
forma predeterminada.

## Compatibilidad

`openclaw doctor --fix` migra la configuración distribuida de
`plugins.entries.codex-supervisor`, incluidos los endpoints y las políticas de transcripción y
escritura, además de las referencias de permiso o denegación de Plugins, a
`plugins.entries.codex.config.supervision`. Los valores canónicos explícitos del destino
prevalecen en caso de conflicto. El código de ejecución solo utiliza la forma
canónica del Plugin `codex` después de la migración.

El Plugin oficial conserva exactamente cinco herramientas de compatibilidad de
Supervisor:
`codex_endpoint_probe`, `codex_sessions_list`, `codex_session_read`,
`codex_session_send` y `codex_session_interrupt`. De forma predeterminada, la lista de
sesiones solo incluye las cargadas; no hay ningún parámetro `loaded_only`.
`include_stored: true` añade filas no archivadas de la base de datos de estado,
limitadas por endpoint mediante `max_stored_sessions` (valor predeterminado 200,
intervalo aceptado de 1 a 1,000); ese ajuste no limita las filas cargadas. Los
campos derivados de transcripciones y las lecturas siguen estando restringidos
por `allowRawTranscripts`; el envío y la interrupción siguen estando restringidos
por `allowWriteControls`.

El envío de compatibilidad nunca inicia ni reanuda un hilo inactivo.
`mode: "start"` siempre se rechaza; `"auto"` y
`"steer"` solo dirigen un turno activo legible. Del mismo modo, la
interrupción requiere un turno activo legible. La continuación inactiva se
dirige al catálogo nativo de Codex para que el arnés completo gestione las
aprobaciones, las herramientas y la vinculación. El adaptador MCP heredado
independiente resuelve estas mismas herramientas desde el Plugin oficial y es la
única vía que respeta las variables de entorno conservadas de la política
heredada.

La interfaz de usuario del catálogo de julio, el método del Gateway, la capacidad
del Node y el registro de la CLI no se habían distribuido con el identificador
antiguo del Plugin. Pasan directamente a ser propiedad de `codex` sin
una segunda fachada de ejecución.

## Trabajo futuro

- ejecutor de streaming en el Node y puente de eventos para la continuación remota
- arrendamientos explícitos del ejecutor y del propietario de aprobaciones para la transferencia simultánea entre clientes
- archivado remoto cuando exista un arrendamiento de propiedad del ejecutor o un mecanismo de aislamiento equivalente
- interrupción y observación más completa de sesiones activas
- transferencia auditada entre Codex Desktop, CLI y OpenClaw

La exploración de elementos archivados no forma parte de la barra lateral de
supervisión prevista. Las superficies nativas de Codex siguen siendo la vía de
recuperación para los hilos archivados.

## Pruebas de aceptación

- Al habilitar la supervisión, se enumeran las sesiones locales no archivadas.
- Las sesiones archivadas nunca aparecen en la respuesta del catálogo ni en la interfaz de usuario.
- Los hosts en buen estado permanecen visibles cuando falla otro host; un host no disponible
  no devuelve filas recientes en lugar de inventar un estado de sesión sin conexión.
- Una fila local almacenada o inactiva crea un reflejo de Chat con un bloqueo de
  modelo/entorno de ejecución exclusivo de Codex; el primer turno fija una instantánea temporal e inicia el
  hilo canónico del arnés completo, y al repetir Continue se abre el Chat existente.
- El primer turno omite las sustituciones de modelo/proveedor en la bifurcación de la instantánea y fija
  el inicio canónico al par exacto devuelto por Codex, incluso cuando Codex advierte
  que su modelo actual difiere del último modelo registrado del origen.
- Las vinculaciones supervisadas pendientes y confirmadas usan la conexión de supervisión para
  acceder al origen, crear la rama canónica y realizar todos los turnos posteriores; las sesiones
  ordinarias de Codex permanecen limitadas al agente.
- Las reanudaciones posteriores omiten las sustituciones de modelo/proveedor de OpenClaw, conservan la
  selección persistida canónica de Codex, aceptan cambios nativos independientes en ese hilo
  y nunca sustituyen el modelo externo de OpenClaw ni la cadena de respaldo.
- Al deshabilitar la supervisión o perder el ciclo de vida de la vinculación/conexión, se produce un fallo
  seguro en lugar de trasladar el Chat al arnés ordinario del directorio principal del agente.
- Un Chat supervisado con el modelo bloqueado no puede eliminarse mientras proteja la vinculación
  nativa.
- El Chat refleja como máximo 200 mensajes del usuario y del asistente, 512 KiB en total y
  64 KiB por mensaje. Las imágenes se convierten en marcadores de posición; no se clonan el razonamiento
  del origen, las llamadas a herramientas, los resultados de herramientas, las cargas de imágenes ni las rutas locales.
- El flujo de la rama nunca reanuda el hilo de origen.
- El origen original sigue siendo apto para ambos catálogos. La rama nativa
  canónica usa el tipo de origen `appServer` y no se garantiza que aparezca en
  Codex Desktop.
- Los orígenes locales activos no pueden crear una rama ni archivarse; aun así, se puede abrir
  un Chat supervisado existente.
- Las filas con actividad desconocida pueden crear una rama sin confirmación; para archivarlas se requiere
  una confirmación explícita de que no hay otro ejecutor.
- Un origen con una rama supervisada en inicialización o pendiente no puede archivarse
  hasta que el primer turno de Chat materialice la rama canónica.
- Un propietario de vinculación activo conocido para el destino exacto o cualquier descendiente generado
  no archivado impide el archivado; los fallos al enumerar descendientes producen un fallo seguro, y
  la confirmación explícita sigue siendo responsable de los clientes desconocidos y de la
  condición de carrera entre la comprobación del estado y el archivado.
- El archivado local confirmado de una sesión almacenada o inactiva elimina la fila tras el éxito nativo.
- Las filas de nodos emparejados permanecen visibles sin Continue ni Archive.
- La enumeración pasiva nunca se suscribe a las aprobaciones del hilo ni responde a ellas.
- La configuración heredada de Supervisor migra a la estructura de configuración canónica de Codex.
- La lista heredada solo se carga de forma predeterminada, la enumeración almacenada respeta su límite
  por endpoint y el envío de compatibilidad nunca inicia ni reanuda un hilo inactivo.
