---
read_when:
    - Diseño o implementación del aprovisionamiento de workers en la nube, el modo worker o la transferencia de sesiones
    - Cambiar environments.*, el protocolo de trabajadores, la ingesta de transcripciones o las RPC del proxy de inferencia
    - Revisión de la postura de seguridad de la ejecución remota de agentes
summary: Ejecuta sesiones de agentes en máquinas efímeras accesibles mediante SSH, con inferencia a través del proxy del Gateway y transmisión en directo en la barra lateral.
title: Plan de trabajadores en la nube
x-i18n:
    generated_at: "2026-07-26T05:45:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 134c3f6e486837607225d95d12a3153525b14237b362b9f9957313d9bc379dc4
    source_path: plan/cloud-workers.md
    workflow: 16
---

## Estado

Propuesta, revisión 3. No implementada. Dirección acordada en 2026-07; la revisión 2 incorporó los hallazgos de la revisión adversarial (protocolo dedicado para workers, máquinas de estados de ubicación/entorno, sincronización entrante compatible con git, traspaso unidireccional en v1 y terminología de seguridad de egreso controlado). La revisión 3 establece el modelo de propiedad de la sincronización (el worker crea los commits, el gateway los adopta y publica), añade un modo de sincronización simple sin git, corrige la ejecución del worker para que sea completa dentro de la máquina, traslada la política de internet al momento del aprovisionamiento y restablece el despacho del agente en el hito 3.

## Problema

Las sesiones de agentes de OpenClaw ejecutan su bucle, sus herramientas y la inferencia dentro del proceso del gateway en una sola máquina. La capacidad de cómputo está limitada por esa máquina, las tareas largas la ocupan y el trabajo en paralelo compite por sus recursos. Los productos alojados (agentes en la nube de Cursor, Claude Code en la web, Codex cloud) resuelven esto con entornos aislados efímeros en la nube para cada tarea, pero requieren infraestructura y confianza en el proveedor.

Los operadores que ya poseen máquinas libres (o pueden alquilarlas a bajo coste) no tienen forma de indicar: ejecuta esta sesión allí, muéstrala en mi barra lateral como cualquier otra sesión y elimina la máquina después.

## Objetivos

- Ejecutar una sesión de agente completa (bucle + herramientas) en una máquina remota efímera («worker en la nube»), mientras la sesión aparece y se transmite en la interfaz de control exactamente igual que una sesión local.
- Sin credenciales permanentes en el worker (sin autenticación de proveedores ni tokens del servicio de alojamiento de repositorios) y sin egreso directo de red; la máquina solo necesita un sshd accesible.
- Aprovisionar, sincronizar, ejecutar, recopilar y destruir de forma totalmente automatizada y compatible con distintos proveedores (primer proveedor: CLI de alquiler al estilo de Crabbox).
- Despachar trabajo en ejecución desde el gateway a un worker en el límite entre turnos sin perder la transcripción, la identidad de la sesión ni, cuando los bytes de la solicitud sigan siendo equivalentes, la afinidad con la caché del proveedor; recuperar los resultados de forma segura.
- Permitir que tanto las personas (interfaz de usuario) como los agentes (herramienta) despachen trabajo a un worker en la nube.
- Admitir sesiones de varios días; la duración es una política, no un límite codificado de forma rígida.

## Fuera de alcance (v1)

- No se admiten entornos externos de programación (Claude Code, Codex CLI) en los workers. Las sesiones de los workers solo ejecutan el runner integrado de OpenClaw. La compatibilidad con estos entornos será opcional en v2 porque realizan su propia inferencia con sus propias credenciales.
- No se admite la selección del mejor de N intentos ni la expansión en paralelo de intentos.
- Sin dependencia de VPN ni tailnet. El transporte se realiza únicamente mediante SSH.
- Sin un nuevo entorno de ejecución aislado. La máquina del worker constituye el límite de aislamiento; posteriormente se podrá añadir aislamiento del sistema operativo dentro de la máquina.
- Sin migración simétrica en vivo en v1: el despacho es local → worker; el retorno de worker → local requiere una sesión detenida y que se haya completado la conciliación del espacio de trabajo. El traspaso bidireccional en vivo se basará posteriormente en el mismo mecanismo de barreras.
- Sin estado auxiliar JSON en el gateway; el estado del entorno, la ubicación, el cursor y las concesiones reside en SQLite.

## Antecedentes (qué copiamos y qué invertimos)

- Agentes en la nube de Cursor: el bucle del agente se ejecuta en su nube; la máquina virtual es el destino de ejecución de las herramientas; el almacén de conversaciones es de solo anexado y se transmite a todos los clientes; inicio en caliente mediante una instantánea posterior a la instalación; los workers autoalojados son procesos de worker que solo establecen conexiones salientes. Copiamos los modelos en los que «la fuente de verdad de la conversación permanece en el orquestador» y de transmisión; invertimos la ubicación del bucle (véase la decisión más adelante).
- Codex cloud: entorno de ejecución en dos fases: una fase de configuración con red y, después, una fase del agente sin conexión y sin secretos; caché del estado del contenedor para acelerar las continuaciones. Copiamos la división en fases como estrategia de egreso y la idea de la caché para imágenes en caliente de v2.
- Claude Code en la web: máquina virtual por sesión; proxy de git que aísla las credenciales (los tokens reales nunca entran en el entorno aislado y el envío está restringido a la rama de la sesión); instantánea del sistema de archivos después de la configuración; traspaso por teletransporte = rama enviada + historial reproducido. Copiamos el aislamiento de credenciales y el planteamiento del traspaso, pero la sincronización saliente se realiza mediante rsync desde el gateway para que funcionen los árboles de trabajo con cambios sin confirmar y no exista ningún token del servicio de alojamiento de repositorios cerca de la máquina.
- Agente de programación de Copilot: egreso denegado de forma predeterminada con una lista de permitidos para registros de paquetes. Nuestro valor predeterminado durante el funcionamiento estable es más estricto (ningún egreso directo) porque la inferencia y la búsqueda web llegan a través del túnel SSH; sin embargo, véase Seguridad para entender por qué se trata de «egreso controlado» y no de «egreso cero».

## Decisión arquitectónica: bucle en el worker, inferencia a través del gateway

Se consideraron tres ubicaciones:

1. El bucle permanece en el gateway y el worker ejecuta las herramientas (modelo de Cursor). Es el dominio de fallos más seguro (la transcripción, la inferencia, las aprobaciones y la recuperación tras reinicios permanecen en el entorno local) y el primer hito preferido por los revisores. Se descartó como arquitectura del producto: las herramientas de OpenClaw distintas de exec son operaciones del sistema de archivos dentro del proceso, por lo que cada lectura, edición o búsqueda grep de archivos se convierte en un recorrido de ida y vuelta por la red o requiere una amplia refactorización de la superficie de herramientas para convertirla en RPC generales del espacio de trabajo; el comportamiento del entorno de ejecución genera mucho tráfico y está limitado por la latencia. Reutilizamos su filosofía donde ya está implementada (descarga de exec a los nodos), pero no desarrollamos la capa de ejecución remota de herramientas.
2. Tanto el bucle como la inferencia se ejecutan en el worker. Es el dominio de fallos más sencillo, pero las credenciales del modelo (incluidos los perfiles de OAuth) deben enviarse a máquinas desechables, el gateway pierde el control de las políticas, el enrutamiento y la auditoría, y la migración cambia la identidad que invoca al proveedor, lo que invalida las cachés del proveedor.
3. El bucle y las herramientas se ejecutan en el worker, y las llamadas al modelo se canalizan a través del gateway. Opción elegida. Un recorrido de ida y vuelta por cada turno del modelo en lugar de uno por cada llamada a una herramienta; las herramientas se ejecutan junto al código; el gateway continúa siendo el único propietario de los perfiles de autenticación, el enrutamiento de proveedores y las políticas; el worker no contiene secretos.

El coste de la opción 3 es una dependencia síncrona del gateway durante cada turno del modelo, por lo que sus reglas de durabilidad forman parte de la decisión y no son una consideración posterior:

- La pérdida del gateway durante un turno provoca un fallo en la llamada activa al proveedor. El turno se marca como fallido y se reintenta como un turno nuevo después de restablecer la conexión; no se reproduce de forma transparente un flujo del proveedor en curso, debido al riesgo de facturación y llamadas a herramientas duplicadas.
- Cada operación entre el worker y el gateway incorpora una identidad duradera (véase Protocolo del worker), por lo que las reconexiones reanudan las operaciones u obtienen resultados terminales almacenados en caché en lugar de dejarlas suspendidas.
- El gateway es un componente con capacidad administrada: los límites de workers simultáneos, el control de flujo y la reducción de carga forman parte del alcance de v1 (véase Capacidad).

Como el gateway almacena la transcripción y origina todo el tráfico hacia los proveedores, la sesión es independiente de su ubicación: trasladar el bucle entre el gateway y el worker no cambia nada del lado del proveedor ni en la ruta de datos de la interfaz de usuario. Esto permite que el despacho y la recuperación sean económicos.

## Componentes

### 1. Máquina de estados del entorno + contrato del proveedor

`environments.*` en el protocolo del gateway actualmente solo es una proyección de estado. El núcleo duradero es un registro de entorno y una máquina de estados propiedad de SQLite, diseñados antes que las formas de los RPC:

`requested → provisioning → bootstrapping → ready → (attached|idle) → draining → destroying → destroyed | failed | orphaned`

- El aprovisionamiento es seguro frente a fallos: la fila de intención se conserva antes de llamar al proveedor, con un identificador de operación determinista, para que un reinicio del gateway pueda adoptar un alquiler en curso en lugar de aprovisionar dos veces o dejar huérfana una máquina de pago.
- La conciliación tras reinicios y un proceso de limpieza de recursos huérfanos (`inspect` del proveedor frente a los registros locales) son requisitos de v1, no medidas de endurecimiento.

Contrato del proveedor (implementado mediante un Plugin; sin nombres de proveedores ni políticas en el núcleo):

```ts
type WorkerProvider = {
  id: string;
  provision(profile: WorkerProfile, opId: string): Promise<WorkerLease>; // → ssh host/port/user/key material
  inspect(lease: { leaseId: string; profile: WorkerProfile }): Promise<LeaseStatus>; // adopt/health/orphan sweep
  renew?(leaseId: string): Promise<void>; // long-lived sessions vs provider TTLs
  destroy(lease: { leaseId: string; profile: WorkerProfile }): Promise<void>; // idempotent, returns only on proof of teardown
};
```

RPC: `environments.create`, `environments.destroy`, `environments.list/status` ampliado (proveedor, identificador del alquiler, estado, antigüedad, tiempo de inactividad, sesiones asociadas). Primeros proveedores: un envoltorio de CLI de alquiler con la forma de Crabbox (ruta del producto) y un proveedor de hosts SSH estáticos marcado solo para desarrollo; un worker en un host compartido puede leer datos ajenos del host, por lo que los hosts estáticos se destinan al desarrollo de la funcionalidad y no constituyen la estrategia predeterminada.

### 2. Arranque del worker: instalación de OpenClaw en la máquina

Sin artefactos específicos para el worker ni dependencia de la disponibilidad de npm:

- Instalación canónica para todos los modos: un paquete del worker generado por el gateway y con hash de contenido (la salida de compilación del propio gateway empaquetada como tarball), enviado mediante SSH e instalado en la máquina. Por diseño, esto abarca las compilaciones de desarrollo y los commits aún no publicados.
- `npm i -g openclaw@<exact gateway version>` es una optimización cuando el gateway ejecuta una versión publicada; nunca `latest`.
- El arranque es idempotente; un alquiler en caliente con un hash del paquete coincidente omite la instalación. Las máquinas sin preparar pueden necesitar una fase de herramientas con acceso a la red (entorno de ejecución de Node), que forma parte de la fase de configuración y se cierra posteriormente.
- El protocolo de enlace verifica el hash de compilación del worker, el conjunto de funcionalidades del protocolo y la compatibilidad del entorno de ejecución. Las comprobaciones existentes de versión y protocolo del gateway no bastan para esto (los nodos canalizados mediante SSH están exentos del rechazo por versión no coincidente), por lo que la admisión de workers realiza su propia comprobación de compilación exacta.

El modo worker (`openclaw worker`) es un punto de entrada, no una bifurcación: gestión de conexiones más el runner de agentes integrado, con persistencia de sesiones y llamadas al modelo respaldadas por RPC del gateway. No debe iniciar superficies del gateway: sin canales, sin inicio automático de plugins más allá del conjunto de herramientas de la sesión, con un directorio de estado desechable y sin perfiles locales de autenticación.

### 3. Transporte: todo mediante SSH

El gateway es propietario de la conectividad; el worker solo requiere sshd:

- El gateway abre una conexión SSH con el worker (con las credenciales del alquiler del proveedor y la clave del host fijada a partir de la salida del aprovisionamiento, sin `StrictHostKeyChecking=no`) y establece un túnel inverso que reenvía un socket local del worker al punto de conexión WS del gateway.
- El tráfico de control/modelo y la transferencia del espacio de trabajo utilizan conexiones SSH independientes con el mismo material de confianza fijado, para que rsync no bloquee los flujos de tokens por precedencia en la cola.
- El ciclo de vida del túnel (mantenimiento de conexión y reconexión con espera incremental) pertenece al entorno de ejecución del entorno en el gateway. Una interrupción breve del túnel es invisible en el nivel de la sesión: el estado duradero del protocolo (descrito más adelante) permite que el worker vuelva a conectarse y reanude la operación.

### 4. Protocolo del worker (dedicado; distinto del protocolo de nodos)

La revisión adversarial de las interfaces actuales de los nodos descartó su reutilización directa: las invocaciones pendientes de los nodos son promesas locales del proceso que desaparecen con la conexión, las claves de idempotencia de los nodos se analizan pero no se deduplican y, de forma decisiva, un nodo conectado puede emitir eventos ordinarios de nodo (incluidas solicitudes de ejecución de agentes), por lo que «tipo de nodo + límite de capacidades» no constituye un límite de seguridad de entrada. Por tanto, los workers reciben un rol `worker` autenticado con una lista cerrada y versionada de RPC y eventos permitidos; las conexiones de workers no pueden acceder a ningún controlador de eventos de nodos heredado.

Identidad y credenciales: el aprovisionamiento genera una credencial de corta duración para el worker vinculada al identificador del entorno, la clave del worker, el hash del paquete, la única sesión permitida, el conjunto de RPC permitido y una fecha de vencimiento. El emparejamiento verificado mediante SSH sigue aplicándose (aprovisionamos la máquina y poseemos la clave), pero la autorización procede de la credencial generada, no de la superficie de nodo declarada.

Semántica de operaciones duraderas (forma tomada del entorno de ejecución ACP existente y su registro de eventos: identificadores estables, serialización por sesión y reproducción duradera de `(session, seq)`):

- Cada operación tiene un ámbito `(sessionId, lifecycleRevision, runId, ownerEpoch, streamKind, seq)`.
- Las épocas de propiedad bloquean a los workers obsoletos: un worker de reemplazo avanza la época; los resultados tardíos de la época anterior se rechazan de forma determinista.
- Entrega al menos una vez con cursores ACK persistidos y resultados terminales almacenados en caché en SQLite; la deduplicación es determinista. No se promete una entrega exactamente una vez.
- Tramas explícitas para cancelar, cerrar, reanudar y devolver resultados terminales; control de flujo basado en créditos/ventanas en los flujos.
- La negociación de características del protocolo es independiente de la versión general del protocolo de Node.

### 5. RPC del backend de sesiones

Dos contratos distintos: el código base actual separa las mutaciones duraderas de la transcripción (propiedad del gestor de sesiones, árbol JSONL con estado de padre/hoja) de los eventos activos locales del proceso (deltas de streaming, ciclo de vida de herramientas, aprobaciones), y el protocolo del worker debe preservar esa separación:

- Confirmaciones duraderas de la transcripción: el worker envía lotes de anexión semántica con `runEpoch` + comparación e intercambio de la hoja base; el gestor de sesiones del Gateway genera los identificadores de entrada y los identificadores de padre. El worker nunca puede proporcionar filas de transcripción de confianza, identificadores de entrada, identificadores de padre ni identificadores de sesiones externas.
- Eventos activos reproducibles: una unión tipada de eventos con números de secuencia del worker, ACK del Gateway, retención acotada y bloqueo de eventos tardíos, que alimenta la distribución existente de eventos del agente para que la vista del chat, las filas de herramientas y la lógica de elementos no leídos/estado se comporten de forma idéntica a las sesiones locales.

Proxy de inferencia: reutiliza el vocabulario de eventos del cliente de flujo del proxy de ejecución existente (`src/agents/runtime/proxy.ts`), pero desplaza el límite de confianza. El worker solo envía la identidad de la sesión/ejecución, una referencia de modelo aprobada, contexto y opciones de generación restringidas; el Gateway resuelve el proveedor, el endpoint, la autenticación, las cabeceras, el enrutamiento y la política de costes desde su propio catálogo. Se rechaza un objeto de modelo proporcionado por el worker (p. ej., `baseUrl` controlado por un atacante). Se aplican límites de tamaño de solicitud, cancelación, auditoría y reproducción del resultado terminal. Las herramientas residentes en el Gateway (websearch) se ejecutan en el Gateway y devuelven los resultados por el mismo canal.

### 6. Sincronización del espacio de trabajo

El ancla de sincronización es un espacio de trabajo local del Gateway con propiedad exclusiva de la ubicación: para espacios de trabajo de git, un worktree administrado dedicado (los metadatos existentes del worktree administrado —rama, base, propiedad de la instantánea— constituyen la base); para espacios de trabajo sin git, un directorio de destino propiedad del Gateway. Nunca el checkout activo del usuario. La propiedad exclusiva mientras la sesión está ubicada de forma remota es lo que hace que la sincronización entrante no tenga conflictos por construcción.

División de responsabilidades: commit frente a publicación:

- El agente del lado del worker crea commits normalmente en su copia (`git commit` es una operación local sin credenciales; la identidad del autor se proyecta desde la configuración del Gateway). Esos commits son objetos inertes hasta que el Gateway los adopta.
- El Gateway realiza todo lo que requiere confianza: verificar que los commits entrantes se basen en la base registrada, hacer avance rápido del worktree local, push, creación de PR y firma o nueva firma opcionales, todo con credenciales locales del Gateway. El worker nunca contiene credenciales de git ni de la plataforma de alojamiento y nunca accede a un remoto.

Dos modos de sincronización, seleccionados según si el espacio de trabajo es un repositorio git:

- Modo git. Salida: sincroniza mediante rsync el worktree (incluidos los archivos sin commit y los archivos sin seguimiento aptos; inclusión/exclusión al estilo de crabbox, respetando `.worktreeinclude`) sobre la identidad SSH del túnel, registrada como un manifiesto base inmutable (hashes de contenido + commit base). Entrada: los nuevos commits se devuelven como un paquete git o una referencia temporal respecto de la base registrada; los artefactos sin seguimiento se devuelven mediante un manifiesto explícito con comprobaciones de tamaño/tipo/contención de enlaces simbólicos. La adopción verifica la ascendencia de la base y se detiene si hay divergencia; nada sobrescribe silenciosamente ninguno de los lados. Las eliminaciones, los cambios de nombre, los submódulos y los escapes mediante enlaces simbólicos se gestionan mediante las reglas del manifiesto, no mediante heurísticas de rsync.
- Modo simple (sin git; p. ej., al crear un proyecto desde cero en la máquina). La salida utiliza el mismo rsync + manifiesto base. La entrada es un reflejo basado en diferencias de manifiesto hacia el directorio de destino propiedad del Gateway, con propagación de eliminaciones. Es seguro por la misma razón que el modo git: la propiedad exclusiva implica que no existen ediciones locales simultáneas con las que entrar en conflicto; el manifiesto base sigue detectando desviaciones locales inesperadas y se detiene en lugar de sobrescribir.

Los puntos de control protegen las sesiones que duran varios días frente a la pérdida del arrendamiento: puntos de control entrantes periódicos (commits de la rama de sesión en modo git, instantáneas del manifiesto en modo simple); la cadencia es una política del perfil (por turnos de forma predeterminada).

### 7. Máquina de estados de ubicación, sesiones e interfaz de usuario

La ubicación en tiempo de ejecución es una máquina de estados propiedad de SQLite y vinculada a la sesión, no un par de campos de fila independientes:

`local → requested → provisioning → syncing → starting → active(worker) → draining → reconciling → local | reclaimed | failed`

Persiste el identificador del entorno, la generación de transición, la época del propietario activo, el manifiesto base del espacio de trabajo, el hash del paquete del worker y los últimos cursores ACK. La admisión de turnos reclama atómicamente la ubicación antes de que cualquiera de los bucles inicie un turno, por lo que un mensaje local admitido con una instantánea obsoleta nunca puede competir con un turno del worker: exactamente un bucle es propietario de la sesión en todo momento.

Interfaz de usuario:

- Una sesión de worker es una fila de sesión normal más metadatos de ubicación. Reside en el almacén normal, se enumera mediante `sessions.list` y transmite mediante las suscripciones existentes; la barra lateral y el chat no necesitan una nueva ruta de datos, solo presentación: una insignia de worker y el estado de ubicación/entorno (`provisioning / syncing / running / idle / reconciling / reclaimed`).
- Experiencia de creación: la barra de destino de la sesión (rediseño de la barra lateral de sesiones) incorpora un destino de worker en la nube junto al Gateway y Node. Requiere un perfil de proveedor configurado; la característica no se muestra hasta que se configura.
- Envío del agente: una herramienta de sesión permite que un agente transfiera trabajo a un worker en la nube del mismo modo que lo hace una persona (subsesión respaldada por un worker, al estilo de un subagente). Se publica en el mismo hito que el envío humano, sujeto a la misma configuración opcional del proveedor. La recursión está limitada estructuralmente (las sesiones de worker no pueden enviar por sí mismas otros workers en v1); el control del gasto se realiza mediante contabilidad/auditoría por entorno, no mediante mecanismos de cuotas.

## Envío y transferencia

v1 es deliberadamente asimétrica:

- Local → worker (envío): supera la barrera de migración descrita a continuación, aprovisiona o reutiliza un worker, sincroniza, cambia la ubicación y el siguiente turno se ejecuta de forma remota.
- Worker → local (recuperación): detiene la sesión (drena el worker según la misma barrera), completa la conciliación entrante y cambia la ubicación a local. No es una migración en vivo.
- La transferencia simétrica en vivo (mover en ambas direcciones una sesión que está trabajando activamente sin detenerla) reutiliza la misma barrera y los mismos mecanismos de conciliación, y se publica después de que las pruebas de inyección de fallos validen la barrera.

Barrera de migración («límite de turno» por sí solo es insuficiente: las aprobaciones, los procesos en segundo plano y las fusiones de transcripciones con bloqueo liberado pueden atravesarlo):

1. Detener la admisión de nuevos turnos (reclamación de ubicación).
2. Cancelar o drenar las ejecuciones activas.
3. Revocar las aprobaciones de ejecución y las concesiones de ejecución pendientes.
4. Drenar las escrituras laterales de la transcripción y los ACK de eventos activos.
5. Finalizar los procesos secundarios del worker.
6. Bloquear al propietario anterior avanzando la época del propietario.
7. Conciliar el espacio de trabajo (entrante, teniendo en cuenta los conflictos).
8. Activar al nuevo propietario.

Afinidad de caché: dado que las solicitudes al proveedor se originan en el Gateway en ambas ubicaciones, la afinidad de caché se conserva cuando la solicitud serializada al proveedor permanece equivalente: mismo orden de herramientas, instrucciones del sistema, envoltorios del proveedor y metadatos de caché (que permanecen en el Gateway). Esta es una propiedad comprobable, no una suposición: las pruebas de equivalencia de bytes entre la ubicación local y la del worker para cada transporte de proveedor compatible forman parte del hito que introduce el bucle del worker.

## Modelo de seguridad

En términos precisos: el worker no tiene salida de red directa ni credenciales permanentes del proveedor o de la plataforma de alojamiento. No es «salida cero»: la inferencia y las herramientas ejecutadas por el Gateway son canales de salida controlados (un worker afectado por una inyección de prompt aún puede incluir bytes del espacio de trabajo en el contexto del modelo o en consultas de websearch). Por consiguiente:

- Contabilidad de salida controlada: auditoría por entorno y contabilidad visible para el operador en el proxy de inferencia y las herramientas del Gateway. Existen límites de velocidad/bytes como control de flujo del protocolo (capacidad), no como mecanismos de cuotas de gasto.
- La entrada del worker al Gateway es la lista cerrada de elementos permitidos del protocolo del worker; las escrituras de transcripciones están restringidas estructuralmente (identificadores generados por el Gateway, una única sesión vinculada).
- La ejecución del worker dispone de permisos completos dentro de la máquina. La máquina es desechable y no contiene credenciales, por lo que la aprobación por comando añade fricción sin proteger nada; el límite protegido es la conciliación entrante y la auditoría. La ejecución nunca atraviesa la ruta de aprobación de Node del Gateway.
- La política de Internet es una decisión del proveedor en el momento del aprovisionamiento: el perfil del entorno decide al crear la máquina (firewall/grupo de seguridad/red sin salida), opcionalmente con una fase de configuración conectada a la red que el proveedor cierra antes de la fase del agente. El núcleo no implementa un interruptor de red en tiempo de ejecución.
- Higiene de la máquina en el momento del aprovisionamiento: endpoint de metadatos de la nube bloqueado o cuya ausencia se ha verificado, sin perfil de instancia, sin agente SSH heredado, sin socket de Docker, entorno/directorio personal limpios. Las claves de host SSH se fijan a partir de la salida del aprovisionamiento.
- Las aprobaciones y políticas para cualquier operación del lado del Gateway (push, PR, llamadas al proveedor) siguen ejecutándose en el Gateway.

Radio de impacto de una sesión de worker comprometida: la copia sincronizada del espacio de trabajo más lo que permitan los canales de proxy auditados; sin credenciales, sin red directa y sin superficie del Gateway más allá de la lista de elementos permitidos.

## Capacidad

El Gateway retransmite cada prompt y flujo de tokens para N workers, por lo que v1 establece un modelo de capacidad en lugar de descubrirlo en producción: límites de workers simultáneos por Gateway, ventanas de créditos por flujo (la cola actual del flujo de eventos no está acotada y el límite del búfer del socket de Node fuerza el cierre para consumidores lentos; ambos son inadecuados sin modificaciones), almacenamiento temporal en disco acotado para ráfagas y descarga de carga con estados visibles de contrapresión en la interfaz de usuario. La transferencia del espacio de trabajo permanece en su propio canal SSH.

## Ciclo de vida

- La detención automática por inactividad y el TTL son políticas del perfil del proveedor, no constantes fijas. Los valores predeterminados son generosos con mantenimiento de actividad explícito; el trabajo de varios días es de primera clase (existe `renew` del proveedor para backends basados en arrendamientos); nunca se recupera una sesión con un turno en curso o actividad reciente.
- Cuando el worker muere o se recupera: la ubicación pasa a `reclaimed`, la fila de sesión permanece y el siguiente mensaje aprovisiona un worker nuevo y vuelve a sincronizar desde el último punto de control. La conversación nunca se pierde (almacén del lado del Gateway); se pierden los cambios del espacio de trabajo posteriores al último punto de control y la interfaz de usuario lo indica.
- Reutilización de arrendamientos activos desde el primer día (para proveedores compatibles); la instantánea de imagen posterior al arranque es la ruta de inicio rápido de v2.

## Superficie de configuración

Mínima y opcional: un bloque de perfil del proveedor (identificador del proveedor, referencia de credenciales/CLI, reglas de sincronización, política de duración, presupuestos, fase de configuración opcional) más la selección de ubicación por sesión. Sin nuevas variables de entorno. Las instalaciones sin configurar no muestran nada.

## Hitos

La implementación se incorpora como PR pequeños que pueden fusionarse de forma independiente; cada hito descrito a continuación es una serie de PR, no un único cambio.

1. Fundamentos: máquina de estados del entorno + contrato del proveedor + proveedor con estructura de Crabbox (SSH estático como arnés de desarrollo), arranque del paquete del trabajador + protocolo de enlace de admisión, túnel SSH + fijación de clave de host, instantánea del árbol de trabajo gestionado + sincronización saliente (modos Git + sin formato). Barrido de huérfanos + adopción tras reinicio.
2. Protocolo del trabajador + bucle del trabajador: rol de trabajador autenticado, operaciones/épocas/cursores de ACK duraderos, confirmación de la transcripción + contratos de eventos en vivo, proxy de inferencia con modelos resueltos por el Gateway, control de flujo. Un proveedor, despacho humano únicamente de sesiones nuevas, sin traspaso. Las pruebas de inyección de fallos (partición del túnel, reinicio del Gateway, interrupción del trabajador) condicionan la salida.
3. Despacho + recuperación + despacho de agentes: barrera de migración, máquina de estados de ubicación conectada a la barra de destino de la interfaz de usuario, conciliación entrante + puntos de control, auditoría por entorno, límites de capacidad, herramienta de despacho de agentes (las sesiones de trabajadores no pueden recurrir). Pruebas de equivalencia byte a byte de la caché de prompts.
4. Traspaso simétrico en vivo, después de la prueba de inyección de fallos del hito 3.

Más adelante: arneses de ACP en los trabajadores como opción de hidratación de credenciales por entorno; inicio rápido mediante instantáneas/imágenes precalentadas; distribución en abanico (N concesiones, mismo prompt); aislamiento del SO dentro de la instancia; captura más completa de artefactos mediante el esquema de artefactos.

## Preguntas abiertas

- Disponibilidad de plugins/Skills en los trabajadores: las Skills incluidas en el repositorio se sincronizan gratuitamente con el espacio de trabajo; las Skills/plugins de agentes configurados en el Gateway requieren una decisión explícita de sincronización o exclusión (el manifiesto de herramientas/plugins forma parte del protocolo de enlace de admisión en cualquier caso).
- Cadencia predeterminada de los puntos de control: basada en turnos frente a basada en tiempo para sesiones muy activas.
- Cómo interactúan los perfiles de entorno con el enrutamiento multiagente (perfiles predeterminados por agente frente a selección únicamente por sesión).
