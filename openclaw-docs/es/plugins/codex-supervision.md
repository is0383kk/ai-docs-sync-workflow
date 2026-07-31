---
read_when:
    - Quieres que las sesiones de Codex Desktop o de la CLI aparezcan en OpenClaw
    - Necesita crear una rama a partir de una sesión local de Codex almacenada o inactiva, o archivarla.
    - Está exponiendo sesiones de Codex y el historial de transcripciones de nodos emparejados
sidebarTitle: Codex supervision
summary: Explora sesiones nativas de Codex no archivadas y transcripciones paginadas en los nodos de OpenClaw
title: Supervisar sesiones de Codex
x-i18n:
    generated_at: "2026-07-26T05:12:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f365e3207dff092c3dfd8f7588d60d70a16f0cce484991eb4ab3fc0bd15f8051
    source_path: plugins/codex-supervision.md
    workflow: 16
---

La supervisión de Codex es una capacidad opcional del plugin oficial `codex`. Muestra
las sesiones de origen no archivadas de Codex CLI, VS Code, Atlas y ChatGPT del
equipo del Gateway y de los equipos emparejados que hayan habilitado esta opción
en la barra lateral normal de sesiones y en el panel de Chat.

La versión inicial mantiene deliberadamente un ámbito de propiedad limitado:

- Una sesión local almacenada o inactiva puede crear un Chat de OpenClaw con el modelo bloqueado a partir
  de su historial persistente y limitado de mensajes del usuario y del asistente. El primer mensaje inicia una
  bifurcación de instantánea nativa y, a continuación, inicia el hilo completo del entorno de Codex exactamente con
  el modelo y el proveedor que Codex App Server seleccionó para esa bifurcación. Los turnos posteriores
  restauran el par persistente del hilo nativo canónico, mientras que la
  vinculación supervisada impide que OpenClaw lo sustituya por otro entorno de ejecución,
  modelo o mecanismo alternativo. Un control nativo de Codex independiente aún puede cambiar ese
  par persistente. Una rama ya creada abre su Chat existente.
- Una sesión almacenada que se haya detectado desde otro proceso de Codex tiene una actividad en curso
  desconocida. Puede bifurcarse o archivarse únicamente después de que el operador
  confirme que ningún otro cliente de Codex la está utilizando.
- Un origen activo permanece visible, pero no puede crear una rama ni archivarse hasta que
  finalice su turno actual. Si ya tiene un Chat supervisado, **Abrir Chat**
  permanece disponible.
- Una sesión en un nodo emparejado expone su transcripción persistente mediante lecturas limitadas
  y paginadas por cursor de App Server. La continuación remota
  requiere un futuro puente de nodo con streaming; el archivado remoto requiere además
  un arrendamiento de propiedad del ejecutor o un mecanismo de exclusión equivalente.
- Las sesiones archivadas no se muestran. Una sesión local almacenada o inactiva solo puede
  archivarse después de que el operador confirme que ningún otro cliente de Codex la está utilizando.

## Antes de comenzar

- Instale el plugin oficial `@openclaw/codex` en el Gateway. La aplicación de OpenClaw para
  macOS puede instalarlo al habilitar las funciones de Codex; las instalaciones mediante CLI pueden
  ejecutar `openclaw plugins install @openclaw/codex`.
- Instale e inicie sesión en Codex Desktop o Codex CLI en cada equipo cuyas
  sesiones desee mostrar.
- Empareje los equipos remotos como nodos de OpenClaw. Cada equipo debe habilitar la opción localmente;
  habilitar la supervisión solo en el Gateway no autoriza a otro nodo.
- Utilice un Gateway controlado por el propietario. Los títulos de las sesiones, los directorios de trabajo y las ramas de Git
  pueden revelar información confidencial del proyecto.

## Habilitar la supervisión

La configuración guiada de `openclaw onboard` y la configuración inicial de macOS intentan instalar y
habilitar la supervisión de Codex después de detectar una instalación nativa de Codex y
activar correctamente el backend de inferencia seleccionado. Codex no necesita ser el
backend principal. La supervisión queda disponible cuando la activación oportunista
del plugin se completa correctamente. La disponibilidad de App Server se comprueba cuando
la supervisión se conecta por primera vez. La desactivación explícita del plugin de Codex o un bloqueo por
política impiden la activación oportunista, y un valor explícito existente de
`supervision.enabled: false` deshabilita las herramientas de supervisión orientadas al agente; el
catálogo del operador permanece registrado siempre que el plugin de Codex esté activo, salvo que
`sessionCatalog.enabled: false` lo deshabilite. Este interruptor independiente mantiene sin cambios
el proveedor de Codex, el entorno y la política de supervisión orientada al agente, a la vez que
elimina de este host los comandos de listado y lectura del catálogo de nodos emparejados.
Las instalaciones existentes pueden habilitar manualmente la misma capacidad:

Habilite el plugin `codex` y su capacidad de supervisión en `openclaw.json`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          supervision: {
            enabled: true,
          },
        },
      },
    },
  },
}
```

Si `plugins.allow` está presente, incluya `codex`. Reinicie el Gateway después de
cambiar la activación del plugin.

Sin una configuración de conexión explícita de `appServer`, la supervisión utiliza una conexión de
supervisión stdio administrada e independiente con el directorio personal nativo de Codex del usuario. El
entorno ordinario de Codex permanece limitado al agente de forma predeterminada. Esto hace que las sesiones
nativas sean visibles en ambas aplicaciones sin que los turnos ordinarios de OpenClaw compartan
el estado nativo de Codex. Establezca `appServer.homeScope: "user"` explícitamente si el entorno
también debe compartir ese estado. La supervisión respeta la configuración de conexión explícita de
`appServer` en lugar de sustituirla por su valor predeterminado del directorio personal del usuario local.

Un Chat adoptado desde el grupo **Codex** de la barra lateral no es una sesión ordinaria del entorno.
Su vinculación privada de supervisión utiliza la conexión de supervisión para las lecturas del origen,
la creación de ramas canónicas, la inyección del historial y todos los turnos posteriores. Con
la conexión local predeterminada, esto conserva el directorio personal nativo de Codex del usuario, la autenticación
y la configuración del proveedor sin cambiar el valor predeterminado de otras sesiones.
Los Chats adoptados y observados también participan en el [conocimiento del estado de la sesión](/es/concepts/session-state).

En la conexión de supervisión local predeterminada, el almacén se comparte con los clientes
nativos de Codex. OpenClaw no presupone que otro cliente comparta el mismo proceso activo de
App Server, y la propiedad del estado nativo es local al proceso. Por tanto,
trata un hilo que su App Server de supervisión indique como `notLoaded` como
**Almacenado / actividad desconocida**, no como inactivo.

Aplique la misma activación opcional en cada host de nodo sin interfaz cuyas sesiones deban aparecer.
La aplicación nativa de OpenClaw para macOS lee la misma configuración local cuando anuncia
su catálogo de Codex al Gateway emparejado. Ese catálogo de un Mac nativo emparejado admite
únicamente el valor predeterminado o un `appServer.transport: "stdio"` explícito con
`appServer.homeScope: "user"` sin establecer o explícito. `command`, `args` y `clearEnv` se
respetan para ese proceso stdio. Si la configuración del Mac selecciona `"unix"`,
`"websocket"` o `homeScope: "agent"`, la aplicación no anuncia la capacidad ni el comando
del catálogo, y una invocación directa obsoleta falla en lugar de exponer
el directorio personal de Codex del usuario o iniciar otro App Server stdio local.

Un comando de nodo anunciado recientemente modifica la superficie de comandos aprobados del nodo.
Apruebe la actualización desde el host del Gateway:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Las sesiones de Codex no archivadas también aparecen en la barra lateral principal de la interfaz de control, agrupadas
por host. Seleccione una para leer su transcripción persistente. El visor utiliza la API más reciente
de Codex `thread/turns/list` con `itemsView: "full"` y carga como máximo 20 turnos
por solicitud; **Cargar elementos anteriores de la transcripción** sigue el cursor opaco de App Server desde la página más reciente.
Las páginas cargadas se representan en orden cronológico. El visor nunca carga un historial
`thread/read` ilimitado. Una página que supere el límite de seguridad de transporte de 20 MiB se cierra
de forma segura en lugar de poner en riesgo la conexión del nodo o del Gateway.

Abra el grupo **Codex** en la barra lateral normal de sesiones. Este muestra las mismas sesiones
agrupadas por host. **Cargar más sesiones** añade la página siguiente de cada host que
tenga filas anteriores, y esas filas añadidas se conservan durante las actualizaciones periódicas de la barra lateral.
Cada host aparece en cuanto finaliza su propio listado nativo. La página visible
se concilia tras los cambios de conectividad de los nodos, cuando recupera el foco y, como máximo,
cada 30 segundos; un resultado modificado provoca una pasada de seguimiento más rápida. Por tanto, las sesiones creadas
en Codex Desktop, la CLI u otro cliente nativo aparecen sin necesidad de
recargar toda la página. La primera página sigue el orden de actualización más reciente del propio Codex,
por lo que una sesión nativa recién creada puede aparecer inmediatamente.
Cada página de búsqueda devuelta examina un número limitado de páginas nativas por host en lugar
de enviar la consulta a App Server, porque la búsqueda nativa también puede encontrar coincidencias
en las vistas previas de las transcripciones.

La disponibilidad del host y el estado del hilo son independientes. **Sin conexión** o **No disponible**
describen una actualización del host; un host no disponible no devuelve filas de sesiones nuevas ni
cambia el estado nativo de un hilo a `offline`. Las filas de sesiones utilizan estados de Codex
como `idle`, `active`, `notLoaded` o error. El fallo de un host no
oculta los resultados de los hosts en buen estado.

La advertencia de la barra lateral incluye el código de error del catálogo y el error subyacente seguro
del Gateway. Abra **Settings > Automation > Plugins > Codex > Native Session
Discovery** para deshabilitar la detección sin deshabilitar Codex. Para
`NODE_LIST_FAILED`, compare `openclaw nodes list` con **Settings > Devices**;
la causa detallada identifica el fallo del almacén de emparejamiento, el registro de nodos, los permisos o
el ciclo de vida del Gateway que debe corregirse.

## Utilizar la CLI del operador

La CLI del terminal expone el mismo catálogo de elementos no archivados y las acciones locales del Gateway
para crear ramas y archivar:

```bash
openclaw codex sessions [--search <text>] [--host <id>] [--limit <count>] [--cursor <cursor>] [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex continue <thread-id> [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
openclaw codex archive <thread-id> --confirm-no-other-runner [--json] [--url <url>] [--token <token>] [--timeout <ms>] [--expect-final]
```

Opciones de `openclaw codex sessions`:

- `--search <text>` busca títulos de sesiones sin distinguir entre mayúsculas y minúsculas.
- `--host <id>` limita la respuesta a un único host estable del catálogo, como
  `gateway:local` o `node:<node-id>`.
- `--limit <count>` establece entre 1 y 100 filas por host; el valor predeterminado es 50.
- `--cursor <cursor>` continúa una página de un host y, por tanto, requiere `--host`.
- `--json` imprime la respuesta estructurada del Gateway.

Los tres comandos heredan `--url`, `--token` y `--timeout <ms>` del
cliente del Gateway. El listado de sesiones utiliza de forma predeterminada 75,000 ms para que los catálogos
de nodos emparejados que se inician en frío puedan completarse; la continuación y el archivado utilizan de forma predeterminada 30,000 ms. También exponen el
interruptor compartido `--expect-final`, que no modifica estas RPC de supervisión unarias.
Cada comando requiere el ámbito `operator.write` del Gateway.
La salida estándar `-h, --help` está disponible en cada subcomando.
No existe ninguna opción para elementos archivados ni para incluir los archivados. `sessions` puede mostrar los hosts
emparejados, pero `continue` y `archive` siempre se dirigen a `gateway:local`; las filas emparejadas
son solo de lectura. El archivado siempre requiere `--confirm-no-other-runner`.

Estos comandos del shell son distintos de los comandos de entorno `/codex` dentro del Chat.
`/codex threads [filter]` muestra los hilos de App Server disponibles para la conexión de la
conversación actual. `/codex sessions --host <node>` muestra los archivos de sesión reanudables de Codex
CLI en un nodo, no el catálogo de la flota de supervisión. `/codex
resume` y `/codex bind` vinculan la conversación actual en lugar de crear una
rama supervisada segura, y un Chat supervisado con el modelo bloqueado rechaza esas
modificaciones de vinculación. No existe ningún comando de entorno `/codex continue` ni `/codex archive`.

## Crear una rama desde una sesión local

Elija **Continuar como rama** en una fila almacenada o inactiva del equipo del Gateway.
OpenClaw crea una entrada normal de Chat, replica el historial limitado del usuario y del asistente
hasta el último turno terminal persistente del origen (completado, interrumpido o
fallido), registra una rama pendiente del entorno y abre el Chat. El selector genérico de modelos
queda bloqueado, pero todavía no se ha seleccionado ningún modelo ni proveedor concreto. El
origen no se reanuda y el hilo canónico del entorno todavía no se inicia.
Repetir la acción abre el Chat existente en lugar de crear otra
rama.

La réplica conserva el tramo visible más reciente que cumpla los tres límites: como máximo 200
mensajes del usuario o del asistente, 512 KiB de texto UTF-8 en total y 64 KiB por
mensaje. Los mensajes demasiado grandes se truncan con un marcador y los mensajes anteriores se
omiten cuando se alcanza un límite. Una entrada de imagen o imagen local se convierte en el marcador de posición literal
`[Image attachment]`; los datos de imagen y las rutas locales no se copian.

Envía el primer mensaje normal de Chat para comenzar el trabajo. El entorno de Codex instala los
controladores reales de aprobación, obtención de información, eventos y entrega. Utiliza una bifurcación
nativa temporal en la conexión de supervisión para fijar la instantánea de origen sin
proporcionar una anulación de modelo ni de proveedor. Codex App Server selecciona ambos a partir de su
configuración nativa actual y devuelve la selección real. En esa misma
conexión, OpenClaw inicia el hilo canónico del entorno completo con origen `appServer`
en su cwd y conforme a su política de ejecución exactamente con ese par devuelto, inyecta el
historial visible acotado y archiva la bifurcación temporal. El hilo canónico
dispone de toda la superficie de herramientas del entorno de OpenClaw. Esta es una bifurcación del historial visible, no
un clon completo de la ejecución nativa: se omiten el razonamiento de origen, las llamadas a herramientas
y los resultados de las herramientas. Este turno y todos los posteriores permanecen en la conexión
supervisada de Codex, en lugar de usar otro entorno de ejecución de modelos de OpenClaw o el entorno
normal del directorio principal del agente.

La selección devuelta no demuestra cuál era el modelo histórico del origen. Si la
configuración nativa actual difiere del modelo registrado para el último turno
del origen, Codex emite su advertencia normal de diferencia de modelo. OpenClaw utiliza el
par devuelto para iniciar el hilo canónico. Codex conserva el modelo y el
proveedor nativos de ese hilo canónico, y las reanudaciones posteriores los preservan porque
OpenClaw omite las anulaciones de modelo y proveedor. Si el hilo canónico se modifica
mediante un control nativo independiente de Codex, OpenClaw acepta la selección
persistida de Codex. OpenClaw nunca la sustituye por su modelo externo ni por la cadena de alternativas.

El Chat supervisado con modelo bloqueado no se puede eliminar, cambiar de modelo, usar `/new`
ni `/reset`, invocar la acción de restablecimiento de sesión del Gateway ni usar la acción genérica
**Bifurcar sesión**. También se rechaza modificar `/codex model <model>`, `/codex
bind`, `/codex resume` (incluida una sesión de nodo con `--bind here`) y
`/codex detach` o `/codex unbind`, porque estas acciones reemplazarían
o borrarían la vinculación nativa bloqueada. La consulta `/codex model` y `/codex fast`,
`/codex permissions` y `/codex threads` siguen disponibles. Inicia otra
sesión normal cuando se necesite un modelo diferente o un hilo nuevo.

Mantén habilitada la supervisión para este Chat. Si se deshabilita la supervisión o su
vinculación de conexión almacenada deja de estar disponible o se vuelve incoherente, el turno falla
de forma cerrada en lugar de pasar a una sesión normal del directorio principal del agente.

Deshabilitar o desinstalar el plugin `codex` no libera esa propiedad ni
hace que el Chat pueda utilizar otro modelo. El Chat bloqueado se conserva, pero
queda inaccesible; reinstala o vuelve a habilitar el mismo plugin y reinicia el Gateway para
reanudarlo. Este comportamiento deliberadamente cerrado ante fallos evita que la limpieza de retención o una
interrupción temporal del plugin deje huérfana silenciosamente la vinculación nativa.

La herramienta de agente `codex_threads` respeta el mismo límite. No puede adjuntar una
bifurcación diferente ni archivar el hilo nativo vinculado al Chat. Las operaciones de listado y lectura
solo de metadatos siguen disponibles. Las lecturas de transcripciones sin procesar requieren `allowRawTranscripts`.
Cuando el acceso sin procesar está deshabilitado, `codex_threads` también rechaza la búsqueda en listas porque
la búsqueda nativa incluye vistas previas de las transcripciones; la interfaz de control y la CLI del operador
siguen proporcionando una búsqueda acotada solo por título. Cambiar el nombre, desarchivar, realizar una bifurcación independiente y
archivar un hilo ajeno no relacionado requieren
`allowWriteControls`. Ninguna opción elude la vinculación bloqueada.

OpenClaw no se suscribe a solicitudes de aprobación ni las responde cuando simplemente enumera
el hilo de origen o muestra el Chat pendiente. Iniciar un hilo canónico
independiente del entorno en el primer turno permite que otro proceso de Codex siga siendo propietario del
origen sin crear escritores de ejecución en conflicto.

El origen original de la CLI, VS Code, Atlas o ChatGPT sigue siendo visible para los clientes
nativos y el catálogo de OpenClaw. La bifurcación canónica se almacena como un hilo nativo
de Codex, pero su tipo de origen es `appServer`; Codex Desktop u otro
cliente nativo puede filtrar ese tipo de origen, por lo que no se garantiza que la bifurcación
aparezca en todas las vistas del historial nativo.

Una fila activa indicada por App Server de OpenClaw no puede iniciar una bifurcación nueva. Espera
a que termine el turno actual y actualiza el catálogo. Codex App Server
serializa las modificaciones dentro de un mismo proceso, pero no proporciona un ejecutor exclusivo
entre procesos ni un arrendamiento de propietario de aprobaciones.

Para una fila **Almacenada / actividad desconocida**, el reflejo del Chat y la fijación de la instantánea
del primer turno utilizan el estado de Codex hasta el último turno terminal persistido. El hilo
de origen no se reanuda, interrumpe ni archiva. Si otro proceso tiene un
turno en curso, es posible que su trabajo en curso más reciente no esté presente en la bifurcación.

## Archivar una sesión local

Selecciona **Archivar** en una fila local del Gateway almacenada o inactiva y, a continuación, confirma que ningún
otro cliente de Codex ni ejecutor de OpenClaw esté utilizando ese hilo ni sus
descendientes generados. OpenClaw vuelve a leer el estado local del proceso, continúa solo si es
`idle` o `notLoaded`, llama a la operación nativa de archivado de Codex y elimina la
sesión de la lista de elementos no archivados. Codex nativo también intenta archivar los
descendientes generados del hilo.

El archivado no está disponible cuando la lectura reciente indica que la sesión está activa o en
estado de error, cuando pertenece a un nodo emparejado o mientras un Chat supervisado
recién creado aún tiene una bifurcación pendiente de ese origen. Envía el primer mensaje del Chat
para materializar su bifurcación canónica antes de archivar el origen.
El archivado también se bloquea cuando OpenClaw sabe que una vinculación activa posee el
hilo de destino exacto o cualquier descendiente generado no archivado. OpenClaw recorre la
consulta experimental de descendientes de Codex en todas las páginas; una respuesta no válida,
un error de solicitud, un cursor o hilo repetido o el agotamiento del límite de seguridad hacen que se rechace
el archivado.

Las solicitudes de lectura, enumeración de descendientes y archivado no constituyen una única operación
condicional, por lo que aún puede iniciarse un turno entre ellas. El estado de App Server tampoco
se comparte entre procesos independientes. Por lo tanto, la confirmación es el
límite de seguridad frente a clientes desconocidos y esa condición de carrera: cierra o verifica de otro modo
todos los demás clientes antes de confirmar. Restaura un hilo archivado con Codex
Desktop, la CLI de Codex o un flujo nativo de administración de hilos autorizado por el propietario;
volverá a aparecer después de desarchivarlo.

```bash
codex unarchive <thread-id>
```

## Comprender los límites de los nodos emparejados

Los nodos emparejados exponen los comandos de solo lectura y con versión
`codex.appServer.threads.list.v1` y
`codex.appServer.thread.turns.list.v1`. Los hosts de nodos nativos que tienen disponible la
CLI de Codex también exponen el comando permitido `codex.terminal.resume.v1`.
El Gateway recibe metadatos normalizados
y páginas acotadas de transcripciones solicitadas explícitamente, nunca endpoints sin procesar de App Server.
Al abrir una fila en el terminal del operador, se ejecuta `codex resume <thread-id>`
en el host propietario y se retransmite el PTY de ese comando; no se expone un shell general
ni argumentos argv suministrados por el Gateway.

La retransmisión del terminal no proporciona los contratos de continuación del entorno ni de propiedad
del archivado. Por tanto, las filas remotas siguen siendo visibles, pero no ofrecen **Continuar** ni
**Archivar**, incluso cuando el hilo remoto está inactivo. Utiliza Codex en ese equipo
mediante **Abrir en terminal**, o utiliza un flujo de continuación futuro con un límite seguro
de propiedad del ejecutor.

## Metadatos y permisos

Las filas del catálogo pueden incluir:

- identificadores de hilo y sesión
- título y directorio de trabajo
- estado actual e indicadores de espera activa
- marcas de tiempo de creación, actualización y actividad
- origen, proveedor del modelo, versión de la CLI de Codex y rama de Git

La proyección del catálogo excluye las vistas previas de transcripciones, los turnos, las rutas de ejecución,
la ruta del directorio principal de Codex, los remotos de Git, los SHA de commits y los errores sin procesar de App Server. El acceso
al catálogo y las lecturas de transcripciones de la interfaz de control requieren el ámbito del Gateway
`operator.write`, porque la agregación de la flota utiliza la ruta estándar `node.invoke`, aunque
ambos comandos de nodo sean de solo lectura.

`supervision.allowRawTranscripts` y `supervision.allowWriteControls` controlan
las herramientas autónomas del agente y las herramientas MCP independientes. El valor predeterminado de ambos es `false`. Con
la supervisión habilitada, `codex_threads` elimina las vistas previas de transcripciones y los turnos de
los resultados de listas y lecturas solo de metadatos, salvo que se permitan las transcripciones sin procesar; una
lectura que incluya turnos falla de forma cerrada. Cada bifurcación, cambio de nombre, archivado y desarchivado
requiere controles de escritura. Estas opciones no restringen la visualización autenticada de transcripciones
en la interfaz de control ni eluden las comprobaciones de vinculación, host, estado o confirmación.

### Herramientas de compatibilidad

El plugin oficial `codex` conserva los cinco nombres publicados de herramientas de Supervisor para
los clientes existentes del agente y de MCP independientes:

- `codex_endpoint_probe`
- `codex_sessions_list`
- `codex_session_read`
- `codex_session_send`
- `codex_session_interrupt`

De forma predeterminada, `codex_sessions_list` solo incluye elementos cargados; no existe ningún parámetro `loaded_only`.
Establece `include_stored: true` para leer también filas almacenadas no archivadas de
la base de datos de estado de Codex. El límite opcional `max_stored_sessions` tiene un valor predeterminado de 200
y acepta de 1 a 1,000 filas por endpoint. No limita las filas cargadas.
Sin permiso para transcripciones sin procesar, los resultados de las listas omiten los nombres derivados de transcripciones,
las vistas previas y los errores detallados de endpoints.
`codex_session_read` requiere `allowRawTranscripts`; `include_turns: true`
también solicita turnos a Codex.

`codex_session_send` y `codex_session_interrupt` requieren
`allowWriteControls`. El envío acepta `mode: "auto" | "start" | "steer"`, pero
`"start"` siempre se rechaza y tanto `"auto"` como `"steer"` solo pueden orientar un
turno activo legible. Un hilo inactivo se rechaza y se indica que se utilice **Sesiones de
Codex**, donde el entorno completo instala los controladores de aprobación y herramientas antes
de la continuación. La interrupción también requiere un turno activo legible. Estas herramientas
no reanudan ni inician un hilo de origen inactivo.

`openclaw doctor --fix` mueve una entrada retirada `codex-supervisor`, sus campos de endpoint
y permisos, y las referencias de las políticas de permisos y denegaciones del plugin al plugin oficial
`codex`, sin sobrescribir la configuración canónica explícita. El adaptador MCP independiente
de compatibilidad sigue cargando las mismas cinco herramientas desde ese
plugin; las variables de entorno de políticas heredadas solo se aplican dentro de ese adaptador
de confianza.

Para consultar todos los campos de configuración de supervisión, véase
[Referencia del entorno de Codex](/es/plugins/codex-harness-reference#supervision).

## Solución de problemas

**No aparece ninguna sesión:** verifica que `@openclaw/codex` esté instalado, que tanto el
plugin como `supervision.enabled` sean verdaderos, que la lista de permisos actual de plugins permita
`codex` y que las sesiones no estén archivadas. Reinicia el Gateway o el nodo después
de cambiar la activación.

**Continuar está deshabilitado:** una fila sin asignar está activa, pertenece a un nodo emparejado,
su host está desconectado o hay otra acción pendiente. Las filas locales del Gateway almacenadas e inactivas
ofrecen **Continuar como bifurcación** en lugar de una apropiación insegura del hilo exacto. Una fila
que ya tenga un Chat supervisado ofrece **Abrir Chat**.

**Archivar está deshabilitado:** el archivado está disponible para las filas locales del Gateway
almacenadas/con actividad desconocida e inactivas después de confirmar que no hay otro ejecutor. Las filas activas, con errores,
desconectadas, de nodos emparejados, con una bifurcación pendiente o con un propietario conocido de la vinculación exacta siguen
siendo de solo lectura para el archivado.

**Una sesión archivada desapareció:** es lo esperado. La página de supervisión no tiene
una vista de elementos archivados. Ejecuta `codex unarchive <thread-id>` o utiliza Codex Desktop para mostrarla
de nuevo.

**La configuración antigua de `codex-supervisor` permanece:** ejecuta `openclaw doctor --fix`. Doctor
mueve la entrada retirada del plugin y las referencias relacionadas de políticas de plugins a
`plugins.entries.codex.config.supervision` sin sobrescribir la configuración explícita de Codex.

## Relacionado

- [Entorno de Codex](/es/plugins/codex-harness)
- [Referencia del entorno de Codex](/es/plugins/codex-harness-reference)
- [Entorno de ejecución del entorno de Codex](/es/plugins/codex-harness-runtime)
- [Arquitectura de supervisión de Codex](/es/specs/codex-supervision)
- [Nodos](/es/nodes)
- [Seguridad del Gateway](/es/gateway/security)
