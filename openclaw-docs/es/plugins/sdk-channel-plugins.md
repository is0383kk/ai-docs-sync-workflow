---
read_when:
    - Está creando un nuevo plugin de canal de mensajería
    - Quieres conectar OpenClaw a una plataforma de mensajería
    - Necesita comprender la interfaz del adaptador ChannelPlugin
sidebarTitle: Channel Plugins
summary: Guía paso a paso para crear un plugin de canal de mensajería para OpenClaw
title: Creación de plugins de canales
x-i18n:
    generated_at: "2026-07-26T06:00:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

Esta guía crea un plugin de canal que conecta OpenClaw con una plataforma de
mensajería: seguridad de mensajes directos, emparejamiento, respuestas en hilos y mensajería saliente.

<Info>
  ¿Es la primera vez que trabaja con plugins de OpenClaw? Lea primero [Primeros pasos](/es/plugins/building-plugins)
  para conocer la estructura de los paquetes y la configuración del manifiesto.
</Info>

## Responsabilidades del plugin

Los plugins de canal no implementan herramientas para enviar, editar ni reaccionar; el núcleo proporciona una
herramienta compartida `message`. El plugin se encarga de:

- **Configuración** - resolución de cuentas y asistente de configuración
- **Seguridad** - política de mensajes directos y listas de permitidos
- **Emparejamiento** - flujo de aprobación de mensajes directos
- **Gramática de sesión** - cómo se asignan los identificadores de conversación específicos del proveedor a chats
  base, identificadores de hilo y alternativas principales
- **Salida** - envío de texto, contenido multimedia y encuestas a la plataforma
- **Hilos** - cómo se organizan las respuestas en hilos
- **Indicador de escritura de Heartbeat** - señales opcionales de escritura/ocupado para los destinos de entrega de Heartbeat

El núcleo se encarga de la herramienta compartida de mensajes, la conexión con el prompt, la forma externa de la clave de sesión,
la contabilidad genérica de `:thread:` y el despacho.

## Adaptador de mensajes

Exponga un adaptador `message` con `defineChannelMessageAdapter` desde
`openclaw/plugin-sdk/channel-outbound`. Declare únicamente las capacidades duraderas de envío final
que admita realmente el transporte nativo, respaldadas por una prueba de contrato
que demuestre el efecto secundario nativo y el acuse devuelto. Dirija los envíos de texto y contenido multimedia
a las mismas funciones de transporte que utiliza el adaptador `outbound` heredado. Para consultar
el contrato completo de la API, la matriz de capacidades, las reglas de acuse, la finalización
de la vista previa en directo, la política de acuse de recepción, las pruebas y la tabla de migración, consulte
[API de salida de canales](/es/plugins/sdk-channel-outbound).

Si el adaptador `outbound` existente ya cuenta con los métodos de envío y
los metadatos de capacidades adecuados, derive el adaptador `message` con
`createChannelMessageAdapterFromOutbound(...)` en lugar de escribir manualmente otro
puente. Los envíos del adaptador devuelven valores `MessageReceipt`. Para identificadores heredados, derívelos
con `listMessageReceiptPlatformIds(...)` o
`resolveMessageReceiptPrimaryId(...)` en lugar de mantener campos `messageIds`
paralelos.

Declare con precisión las capacidades en directo y de finalización: el núcleo las utiliza para decidir
qué puede hacer un canal, y cualquier divergencia entre el comportamiento declarado y el real provoca un
error en la prueba de contrato:

| Superficie                            | Valores                                                                                          |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`, `previewFinalization`, `progressUpdates`, `nativeStreaming`, `quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`, `normalFallback`, `discardPending`, `previewReceipt`, `retainOnAmbiguousFailure`    |

Los canales que finalizan una vista previa del borrador sin reemplazarla deben dirigir la lógica del entorno de ejecución
a través de `defineFinalizableLivePreviewAdapter(...)` junto con
`deliverWithFinalizableLivePreviewAdapter(...)`, y mantener las capacidades declaradas
respaldadas por pruebas de `verifyChannelMessageLiveCapabilityAdapterProofs(...)`
y `verifyChannelMessageLiveFinalizerProofs(...)`, para evitar divergencias silenciosas en el comportamiento
nativo de la vista previa, el progreso, la edición, la alternativa/retención, la limpieza y el acuse.

Los receptores entrantes que posponen los acuses de la plataforma deben declarar
`message.receive.defaultAckPolicy` y `supportedAckPolicies` en lugar de ocultar
el momento del acuse en el estado local del monitor. Cubra cada política declarada con
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)`.

Los auxiliares de respuesta heredados, como `dispatchInboundReplyWithBase` y
`recordInboundSessionAndDispatchReply`, siguen disponibles para los despachadores
de compatibilidad. No los utilice para código de canal nuevo; comience con el adaptador `message`,
los acuses y los auxiliares del ciclo de vida de recepción/envío de
`openclaw/plugin-sdk/channel-outbound`.

### Entrada entrante (experimental)

Los canales que migren la autorización entrante pueden utilizar la subruta experimental
`openclaw/plugin-sdk/channel-ingress-runtime` desde las rutas de recepción del entorno de ejecución.
Acepta datos de la plataforma, listas de permitidos sin procesar, descriptores de rutas, datos de comandos
y la configuración de grupos de acceso, y devuelve proyecciones de remitente/ruta/comando/activación,
además del grafo de entrada ordenado, mientras que la consulta de la plataforma y los efectos
secundarios permanecen en el plugin. Mantenga la normalización de identidad del plugin en el
descriptor que se proporciona al resolutor; no serialice los valores de coincidencia sin procesar del
estado o la decisión resueltos. Consulte
[API de entrada de canales](/es/plugins/sdk-channel-ingress) para conocer el diseño de la API,
los límites de responsabilidad y las expectativas de las pruebas.

### Entrada duradera y deduplicación de repeticiones

Los canales que adopten la entrada duradera deben utilizar `createChannelIngressMonitor`
desde `openclaw/plugin-sdk/channel-outbound`, salvo que necesiten un contrato de
admisión o procesamiento sustancialmente diferente. Ponga en cola el sobre de transporte sin procesar en un
único punto de control de recepción (sin normalización durante la recepción), condicione el
acuse del transporte a la anexión duradera para los transportes mediante Webhook, derive una
vía serializada por conversación y marque el evento como completado cuando lo adopte el despacho.
La clave principal de la cola es `(queue_name, event_id)`, y la finalización
convierte la fila en una lápida en lugar de eliminarla, por lo que una reentrega tardía de la plataforma
del mismo `event_id` se rechaza de forma duradera durante el período de retención de la lápida.
Consulte [API de salida de canales](/es/plugins/sdk-channel-outbound#durable-ingress-monitors)
para conocer la API del monitor y el contrato de apagado.

Esa lápida constituye la regla de capas para las protecciones contra repeticiones
(`openclaw/plugin-sdk/persistent-dedupe`): un canal procesado conserva una protección
independiente contra repeticiones solo cuando la identidad o retención de la protección supera a la de la cola:
una clave lógica de mensaje que difiere del identificador de entrega del transporte (Telegram
deduplica `chat_id:message_id` porque las fusiones antirrebote pueden hacer reaparecer un mensaje
con un `update_id` nuevo), o un período más largo que la retención de lápidas
del canal. Si la clave de protección fuese igual al `event_id` del procesamiento, elimine la
protección al adoptar el procesamiento y dimensione `completedTtlMs`/`completedMaxEntries`
para que cubran en su lugar el período anterior de la protección. Las protecciones ajenas a la deduplicación, como los
límites de antigüedad, no guardan relación con esta regla. Los identificadores estables de mensajes salientes utilizan el registro
compartido de eco saliente de `openclaw/plugin-sdk/channel-outbound` en lugar de una
caché TTL local del canal.

#### Clases de transporte y retención

Clasifique un transporte según la garantía de recuperación en su límite de recepción:

- **Entrega de eventos o mediante Webhook condicionada al acuse:** envíe el acuse o devuelva éxito únicamente
  después de la anexión duradera. Un error de anexión debe mantener la entrega apta
  para reintento o hacer que falle el límite de recepción. Esta clase incluye Slack, SMS, Zalo,
  Microsoft Teams, Google Chat, LINE y Synology Chat.
- **Entrega mediante sondeo o flujo con espera:** avance el cursor remoto o envíe el
  acuse del transporte únicamente después de la anexión. Cuando no exista un cursor explícito, mantenga la
  devolución de llamada de recepción serializada y con espera para impedir que un error de anexión permita que el
  bucle de recepción se adelante. El sondeo de Telegram, Signal y Tlon utilizan esta clase;
  la entrega de Telegram mediante Webhook sigue la regla condicionada al acuse indicada anteriormente.
- **Sockets sin repetición:** IRC, Mattermost, Twitch y Zalo Personal no pueden solicitar
  a la plataforma que vuelva a entregar un evento aceptado. Su cola duradera protege el
  intervalo de fallo del proceso y permite la recuperación local tras un reinicio; las lápidas
  de finalización son prácticamente inertes frente a repeticiones de la plataforma.

Utilice 30 días como convención de TTL de las lápidas para toda la flota, no como valor predeterminado del SDK. Una
ventana de reentrega de gran volumen utiliza normalmente un límite de 20,000 entradas completadas;
los transportes con espera y sin repetición de menor volumen utilizan normalmente 1,000-2,000.
Las excepciones actuales incluyen los límites de 4,096 entradas de LINE, el TTL de 24 horas para entradas completadas
de SMS y la retención de entradas completadas basada únicamente en el límite de Tlon. Los límites de filas fallidas también pueden ser inferiores
a los de filas completadas. Tanto el TTL como el límite depuran filas, por lo que la retención efectiva termina
cuando se alcanza el primer límite. Desvíese únicamente por un horizonte de reintentos documentado de la plataforma,
un período conservado y publicado de protección contra repeticiones, el volumen esperado o el presupuesto de disco,
o un transporte sin repetición, y cubra el contrato de retención con pruebas.

#### Efectos secundarios al menos una vez

El procesamiento del despacho ejecuta los efectos secundarios de los comandos antes de que la fila de entrada alcance su
lápida de finalización. Un fallo del proceso entre esos pasos repite la fila y
puede volver a ejecutar el efecto secundario. Este intervalo de fallo con semántica de al menos una vez es el
contrato predeterminado. Para trabajos no idempotentes, como escrituras de configuración, limpiezas
de almacenamiento o acuses visibles fuera de la vía de respuesta, utilice
`createIngressEffectOnce(...)` desde
`openclaw/plugin-sdk/ingress-effect-once`. Proporcione a cada llamada el `eventId` de entrada estable
junto con un nombre de efecto. Cree un auxiliar por cada cola/cuenta de entrada y
utilice un `namespacePrefix` estable y único para ese ámbito, ya que los identificadores de eventos del transporte
pueden ser locales de la cola. El auxiliar confirma su reclamación duradera únicamente después de que el
efecto se complete correctamente; un efecto que genera una excepción libera la reclamación para que un reintento de procesamiento pueda
volver a ejecutarlo, mientras que las llamadas simultáneas esperan a la reclamación activa. Los
errores de estado duradero llaman a `onDiskError` cuando se proporciona y se rechazan en lugar de recurrir
a la memoria del proceso.

Establezca el `ttlMs` del auxiliar como mínimo en la retención de lápidas de entrada del canal,
más la demora máxima entre la confirmación del efecto y la finalización de la fila, incluidos
el tiempo de inactividad acotado y los reintentos de procesamiento. El TTL del registro de efectos comienza al confirmarse,
mientras que la retención de la lápida comienza posteriormente, al finalizar; si la vida útil de la fila pendiente
no tiene límite, ningún TTL finito cubre un tiempo de inactividad arbitrario. Cuando la lápida ya no
puede repetir la fila, los registros de efectos anteriores se convierten en un peso muerto. Dimensione
`stateMaxEntries` para cada clave distinta de evento/efecto que pueda existir en esa
ventana de retención, teniendo en cuenta el límite de entradas completadas de la cola y el
número máximo de efectos por evento. Un límite inferior expulsa el registro más antiguo antes de que venza su TTL
y permite que ese efecto vuelva a ejecutarse. Persisten intervalos residuales de al menos una vez
si el proceso termina o la persistencia falla después de que el efecto se complete correctamente, pero antes de que
se confirme la reclamación, o si el registro vence mientras su fila de entrada sigue
pendiente.

#### Contrato de reinicio por cuenta

Los cambios de configuración del canal reinician de forma predeterminada todo el canal. Un canal con varias cuentas
puede establecer `reload.accountScopedRestart: true` únicamente cuando la resolución
de la configuración lee los campos compartidos de todo el canal junto con la cuenta seleccionada, nunca una
cuenta hermana, y el Gateway puede detener e iniciar el entorno de ejecución `(channel, accountId)`
de una cuenta sin reemplazar los entornos de ejecución hermanos.

La ruta con ámbito limitado se aplica únicamente a los cambios bajo
`channels.<channel>.accounts.<non-default-id>.*`. Los cambios en campos compartidos del canal,
`accounts.default`, cuentas eliminadas o que no puedan resolverse, y cambios combinados
que puedan afectar a la herencia se promueven a un reinicio de todo el canal. Los plugins
que no habiliten esta opción siempre utilizan la ruta de todo el canal.

Para los canales que utilizan el procesamiento de entrada duradera, la ruta de detención del monitor de la cuenta
debe primero completar todas las admisiones de transporte aceptadas y, después, liberar y esperar
su procesamiento. Al iniciar la cuenta, se abre la misma cola identificada por la cuenta, cuyo
procesamiento inicial recupera las filas duraderas no despachadas. No añada una segunda
pasada de repetición específica de recarga; la recuperación de la cola es la ruta canónica de reinicio.

Trate este indicador como una declaración de capacidad, no como una preferencia de rendimiento. Las pruebas
de contrato deben demostrar que añadir y editar una cuenta con nombre no modifica la configuración
resuelta de una cuenta hermana, que detener una cuenta completa únicamente el monitor y el procesamiento
de esa cuenta, y que un monitor nuevo recupera las filas de esa cuenta exactamente
una vez. Si no puede demostrarse alguna garantía, omita el indicador.

### Indicadores de escritura

Si el canal admite indicadores de escritura fuera de las respuestas entrantes, exponga
`heartbeat.sendTyping(...)` en el plugin del canal. El núcleo lo llama con el
destino resuelto de entrega de Heartbeat antes de que comience la ejecución del modelo de Heartbeat y
utiliza el ciclo de vida compartido de mantenimiento y limpieza del indicador de escritura. Añada
`heartbeat.clearTyping(...)` cuando la plataforma requiera una señal de detención explícita.

### Parámetros de origen multimedia

Si el canal añade parámetros de la herramienta de mensajes que contienen orígenes multimedia, exponga
los nombres de esos parámetros mediante `plugin.actions.describeMessageTool(...).mediaSourceParams`.
El núcleo utiliza esa lista explícita para normalizar las rutas del entorno aislado y aplicar la política de
acceso al contenido multimedia saliente, de modo que los plugins no necesiten casos especiales en el núcleo compartido para
parámetros específicos del proveedor relacionados con avatares, archivos adjuntos o imágenes de portada.

Conviene usar un mapa indexado por acción, como `{ "set-profile": ["avatarUrl", "avatarPath"] }`,
para que las acciones no relacionadas no hereden los argumentos multimedia de otra acción. Un arreglo plano
sigue funcionando para los parámetros que se comparten intencionadamente entre todas las acciones expuestas.

Los canales que deben exponer una URL pública temporal para que una plataforma
obtenga contenido multimedia pueden usar `createHostedOutboundMediaStore(...)` de
`openclaw/plugin-sdk/outbound-media` con almacenes de estado del plugin. El análisis de rutas
de la plataforma y la aplicación de tokens deben permanecer en el plugin del canal; el asistente compartido
solo gestiona la carga de contenido multimedia, los metadatos de caducidad, las filas de fragmentos y la limpieza.

Los archivos adjuntos entrantes usan datos ordenados, no campos `Media*` paralelos. Normalice
los registros del canal con `toInboundMediaFacts(...)` de
`openclaw/plugin-sdk/channel-inbound` y páselos como `media` al crear el
contexto entrante. Cuando un plugin deba autorizar lecturas de contenido multimedia local, importe
`getAgentScopedMediaLocalRoots(...)` o
`getAgentScopedMediaLocalRootsForSources(...)` desde la subruta específica
`openclaw/plugin-sdk/media-local-roots`. El antiguo
constructor/fachada raíz `agent-media-payload` es compatibilidad obsoleta.

### Estructuración de cargas útiles nativas

Si el canal necesita una estructuración específica del proveedor para `message(action="send")`,
conviene usar `actions.prepareSendPayload(...)`. Coloque tarjetas nativas, bloques, elementos insertados u
otros datos persistentes en `payload.channelData.<channel>` y deje que el núcleo realice el envío
mediante el adaptador de salida/mensajes. Use `actions.handleAction(...)` para el envío
solo como mecanismo de compatibilidad alternativo para cargas útiles que no se puedan serializar y
reintentar.

### Gramática de conversaciones de sesión

Si la plataforma almacena un ámbito adicional dentro de los identificadores de conversación, mantenga ese análisis
en el plugin mediante `messaging.resolveSessionConversation(...)`. Ese es el
enlace canónico para asignar `rawId` al identificador de conversación base, al identificador
de hilo opcional, al `baseConversationId` explícito y a cualquier
`parentConversationCandidates`. Al devolver `parentConversationCandidates`,
ordénelos desde el elemento principal más específico hasta la conversación más amplia/base.

`messaging.resolveParentConversationCandidates(...)` es un mecanismo de compatibilidad
alternativo obsoleto para plugins que solo necesitan alternativas principales además del
identificador genérico/sin procesar. Si existen ambos enlaces, el núcleo usa primero
`resolveSessionConversation(...).parentConversationCandidates` y solo recurre
a `resolveParentConversationCandidates(...)` cuando el enlace canónico
los omite.

Los plugins incluidos que necesiten el mismo análisis antes de que se inicie el registro de canales
pueden exponer un archivo `session-key-api.ts` de nivel superior con una exportación
`resolveSessionConversation(...)` correspondiente (consulte los plugins de Feishu y Telegram).
El núcleo usa esa superficie segura para el arranque solo cuando el registro de plugins
en tiempo de ejecución todavía no está disponible.

Use `openclaw/plugin-sdk/channel-route` cuando el código del plugin necesite normalizar
campos similares a rutas, comparar un hilo secundario con su ruta principal o crear una
clave estable de eliminación de duplicados a partir de `{ channel, to, accountId, threadId }`. El asistente
normaliza los identificadores numéricos de hilos del mismo modo que el núcleo, por lo que conviene usarlo en lugar de
comparaciones `String(threadId)` improvisadas. Los plugins con una gramática de destinos específica
del proveedor deben exponer `messaging.resolveOutboundSessionRoute(...)` para que el núcleo obtenga
la identidad de sesión e hilo nativa del proveedor sin adaptadores de análisis.

### Compatibilidad con vinculaciones de conversaciones por cuenta

Establezca `conversationBindings.supportsCurrentConversationBinding` cuando el canal
admita vinculaciones genéricas de la conversación actual. `createChatChannelPlugin(...)`
establece esta capacidad estática en `true` de forma predeterminada.

Si la compatibilidad varía según la cuenta configurada, implemente también
`conversationBindings.isCurrentConversationBindingSupported({ accountId })`.
El núcleo evalúa este enlace síncrono solo después de habilitar la capacidad estática.
Devolver `false` hace que las operaciones genéricas de capacidad, vinculación,
búsqueda, enumeración, actualización y desvinculación de la conversación actual no estén disponibles para esa cuenta.
Si se omite el enlace, la capacidad estática se aplica a todas las cuentas.

Resuelva la respuesta a partir de la configuración de cuenta o del estado de ejecución ya cargados. Este
enlace controla únicamente las vinculaciones genéricas de la conversación actual; no sustituye
las reglas de vinculación configuradas ni el enrutamiento de sesiones propiedad del plugin. Las pruebas de contrato
deben abarcar al menos una cuenta compatible y otra incompatible mediante el contrato
`ChannelPlugin["conversationBindings"]` exportado por
`openclaw/plugin-sdk/channel-core`.

## Aprobaciones y capacidades de los canales

La mayoría de los plugins de canal no necesitan código específico para aprobaciones. El núcleo gestiona
`/approve` en el mismo chat, las cargas útiles compartidas de los botones de aprobación y la entrega alternativa genérica.
`ChannelPlugin.approvals` se eliminó; coloque los datos de entrega, integración nativa, renderizado y autorización
de aprobaciones en un único objeto `approvalCapability`. `plugin.auth` sirve
solo para iniciar y cerrar sesión: el núcleo ya no lee enlaces de autorización de aprobaciones de ese objeto.

Use `approvalCapability.delivery` únicamente para el enrutamiento nativo de aprobaciones o la
supresión de alternativas, y `approvalCapability.render` solo cuando un canal necesite realmente
cargas útiles de aprobación personalizadas en lugar del renderizador compartido.

### Autorización de aprobaciones

- `approvalCapability.authorizeActorAction` y
  `approvalCapability.getActionAvailabilityState` son la
  interfaz canónica de autorización de aprobaciones.
- Use `getActionAvailabilityState` para comprobar la disponibilidad de la autorización de aprobaciones en el mismo chat.
  Mantenga disponibles los aprobadores configurados para `/approve` aunque la entrega nativa
  esté deshabilitada; para las instrucciones de entrega/configuración, use en su lugar el estado nativo
  de la superficie iniciadora.
- Si el canal expone aprobaciones nativas de ejecución, use
  `approvalCapability.getExecInitiatingSurfaceState` para el estado
  de la superficie iniciadora/cliente nativo cuando difiera de la autorización de aprobaciones
  en el mismo chat. El núcleo usa ese enlace específico de ejecución para distinguir `enabled` de
  `disabled`, decidir si el canal iniciador admite aprobaciones nativas de ejecución
  e incluir el canal en las instrucciones de alternativa del cliente nativo.
  `createApproverRestrictedNativeApprovalCapability(...)` lo completa para
  el caso habitual.
- Si un canal puede deducir identidades estables de mensajes directos similares a las del propietario a partir de la configuración existente,
  use `createResolvedApproverActionAuthAdapter` de
  `openclaw/plugin-sdk/approval-runtime` para restringir `/approve` en el mismo chat
  sin añadir lógica específica de aprobaciones al núcleo.
- Si la autorización de aprobaciones personalizada permite intencionadamente solo la alternativa en el mismo chat, devuelva
  `markImplicitSameChatApprovalAuthorization({ authorized: true })` desde
  `openclaw/plugin-sdk/approval-auth-runtime`; de lo contrario, el núcleo considera el
  resultado como una autorización explícita del aprobador.
- Si una devolución de llamada nativa propiedad del canal resuelve las aprobaciones directamente, use
  `isImplicitSameChatApprovalAuthorization(...)` antes de resolverlas para que la alternativa
  implícita siga pasando por la autorización de actores normal del canal.

### Ciclo de vida de la carga útil e instrucciones de configuración

- Use `outbound.shouldSuppressLocalPayloadPrompt` o
  `outbound.beforeDeliverPayload` para comportamientos del ciclo de vida de la carga útil
  específicos del canal, como ocultar solicitudes locales de aprobación duplicadas o enviar indicadores
  de escritura antes de la entrega.
- Use `approvalCapability.describeExecApprovalSetup` cuando el canal quiera
  que la respuesta de la ruta deshabilitada explique las opciones de configuración exactas necesarias para habilitar
  las aprobaciones nativas de ejecución. El enlace recibe `{ channel, channelLabel, accountId }`;
  los canales con cuentas con nombre deben representar rutas específicas de cuenta, como
  `channels.<channel>.accounts.<id>.execApprovals.*`, en lugar de valores
  predeterminados de nivel superior.
- Use `approvalCapability.describePluginApprovalSetup` cuando sea seguro mostrar
  instrucciones sobre fallos de aprobación del plugin en casos de falta de ruta y agotamiento del tiempo de espera
  de aprobaciones del plugin. `createApproverRestrictedNativeApprovalCapability(...)` no
  lo deduce de `describeExecApprovalSetup`; pase explícitamente el mismo asistente
  solo cuando las aprobaciones de plugins y de ejecución usen realmente la misma configuración nativa.

### Entrega nativa de aprobaciones

Si un canal necesita entrega nativa de aprobaciones, mantenga el código del canal centrado en
la normalización del destino y los datos de transporte/presentación. Use
`createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`,
`createChannelApproverDmTargetResolver` y
`createApproverRestrictedNativeApprovalCapability` de
`openclaw/plugin-sdk/approval-runtime`. Coloque los datos específicos del canal detrás de
`approvalCapability.nativeRuntime`, preferiblemente mediante
`createChannelApprovalNativeRuntimeAdapter(...)` o
`createLazyChannelApprovalNativeRuntimeAdapter(...)`, para que el núcleo pueda componer el
controlador y gestionar el filtrado de solicitudes, el enrutamiento, la eliminación de duplicados, la caducidad, la suscripción
al Gateway y los avisos de enrutamiento a otra ubicación.

`nativeRuntime` se divide en varias interfaces más pequeñas:

- `availability` - determina si la cuenta está configurada y si se debe
  gestionar una solicitud
- `presentation` - asigna el modelo de vista de aprobación compartido a
  cargas útiles nativas pendientes/resueltas/caducadas o acciones finales
- `transport` - prepara destinos y envía/actualiza/elimina mensajes nativos
  de aprobación
- `interactions` - enlaces opcionales de vinculación/desvinculación/eliminación de acciones para botones
  o reacciones nativos, además de un enlace `cancelDelivered` opcional. Implemente
  `cancelDelivered` cuando `deliverPending` registre un estado en proceso o persistente
  (como un almacén de destinos de reacciones) para que dicho estado pueda liberarse si la
  detención de un controlador cancela la entrega antes de que se ejecute `bindPending`, o cuando
  `bindPending` no devuelva ningún identificador
- `observe` - enlaces opcionales de diagnóstico de entrega

Otros asistentes de aprobación:

- Use `createNativeApprovalChannelRouteGates` de
  `openclaw/plugin-sdk/approval-native-runtime` cuando un canal admita tanto
  la entrega nativa desde el origen de la sesión como destinos explícitos de reenvío de aprobaciones. El
  asistente centraliza la selección de configuración de aprobaciones, la gestión de `mode`, los filtros
  de agentes/sesiones, la vinculación de cuentas, la coincidencia con el destino de la sesión y la coincidencia con listas de destinos,
  mientras que los llamadores siguen gestionando el identificador del canal, el modo de reenvío predeterminado, la búsqueda
  de cuentas, la comprobación de que el transporte esté habilitado, la normalización de destinos y la resolución del destino
  de origen del turno. No lo use para crear valores predeterminados de políticas de canal propiedad del núcleo;
  pase explícitamente el modo predeterminado documentado del canal.
- `createChannelNativeOriginTargetResolver` usa de forma predeterminada el comparador compartido de rutas
  de canal para destinos `{ to, accountId, threadId }`. Pase
  `targetsMatch` solo cuando un canal tenga reglas de equivalencia específicas del proveedor,
  como la coincidencia de prefijos de marcas de tiempo de Slack. Pase `normalizeTargetForMatch` cuando
  el canal necesite convertir identificadores del proveedor a una forma canónica antes de que se ejecute el comparador de rutas
  predeterminado o una devolución de llamada `targetsMatch` personalizada, conservando a la vez el
  destino original para la entrega. Use `normalizeTarget` solo cuando deba convertirse a una forma canónica
  el propio destino de entrega resuelto.
- Si el canal necesita objetos administrados por el entorno de ejecución, como un cliente, un token, una aplicación
  Bolt o un receptor de webhooks, regístrelos mediante
  `openclaw/plugin-sdk/channel-runtime-context`. El registro genérico de contexto
  de ejecución permite que el núcleo inicie controladores basados en capacidades a partir del estado de inicio
  del canal sin añadir código adaptador específico de aprobaciones.
- Recurra a `createChannelApprovalHandler` o
  `createChannelNativeApprovalRuntime`, de menor nivel, solo cuando la interfaz basada en capacidades aún
  no sea suficientemente expresiva.
- Los canales con aprobaciones nativas deben enrutar tanto `accountId` como `approvalKind`
  mediante esos asistentes. `accountId` mantiene la política de aprobaciones multicuenta
  limitada a la cuenta de bot correcta, y `approvalKind` mantiene disponible para el canal el comportamiento
  de aprobaciones de ejecución frente a las de plugins sin ramas codificadas de forma fija en
  el núcleo.
- El núcleo también gestiona los avisos de redireccionamiento de aprobaciones. Los plugins de canal no deben enviar
  sus propios mensajes de seguimiento del tipo «la aprobación se envió a mensajes directos/otro canal» desde
  `createChannelNativeApprovalRuntime`; en su lugar, deben exponer un enrutamiento preciso desde el origen
  y hacia los mensajes directos del aprobador mediante los asistentes compartidos de capacidades de aprobación y permitir que
  el núcleo agregue las entregas reales antes de publicar cualquier aviso en el chat
  iniciador.
- Conserve de principio a fin el tipo de identificador de aprobación entregado. Los clientes nativos no deben
  deducir ni reescribir el enrutamiento de aprobaciones de ejecución frente a las de plugins a partir del estado
  local del canal.
- Pase ese `approvalKind` explícito a `resolveApprovalOverGateway`. Esto usa
  el servicio canónico `approval.resolve` y devuelve el ganador registrado cuando
  otra superficie responde primero. La entrada explícita `resolveMethod` anterior
  se conserva para controles respaldados por comandos; las acciones nativas nuevas no deben usarla ni
  deducir el tipo a partir de un identificador.
- Los distintos tipos de aprobación pueden exponer intencionadamente superficies nativas
  diferentes. Ejemplos incluidos actuales: Matrix conserva el mismo enrutamiento nativo de mensajes directos/canales
  y la experiencia de usuario de reacciones para aprobaciones de ejecución y de plugins, aunque permite que
  la autorización difiera según el tipo de aprobación; Slack mantiene disponible el enrutamiento nativo de aprobaciones
  tanto para identificadores de ejecución como de plugins.
- `createApproverRestrictedNativeApprovalAdapter` sigue existiendo como
  adaptador de compatibilidad, pero el código nuevo debe usar preferentemente el constructor de capacidades
  y exponer `approvalCapability` en el plugin.

### Subrutas más específicas del entorno de ejecución de aprobaciones

Para puntos de entrada de canal de uso frecuente, conviene usar estas subrutas más específicas en lugar del módulo
`approval-runtime` más amplio cuando solo se necesite una parte de esa familia:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Del mismo modo, se deben preferir `openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` y
`openclaw/plugin-sdk/reply-chunking` frente a superficies generales más amplias cuando no
se necesiten todas.

### Subrutas de configuración

- `openclaw/plugin-sdk/setup-runtime` abarca los auxiliares de configuración seguros para el entorno de ejecución:
  `createSetupTranslator`, adaptadores de parches de configuración seguros para importar
  (`createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), salida de notas de consulta,
  `promptResolvedAllowFrom`, `splitSetupEntries` y los constructores delegados
  de proxies de configuración.
- `openclaw/plugin-sdk/channel-setup` abarca los constructores de configuración
  para instalaciones opcionales, además de algunas primitivas seguras para la configuración: `createOptionalChannelSetupSurface`,
  `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`,
  `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`,
  `setSetupChannelEnabled` y `splitSetupEntries`.
- Use la superficie más amplia `openclaw/plugin-sdk/setup` solo cuando también necesite
  los auxiliares compartidos de configuración más pesados, como
  `moveSingleAccountChannelSectionToDefaultAccount(...)`.

Si el canal solo pretende anunciar «instale primero este plugin» en las
superficies de configuración, se debe preferir `createOptionalChannelSetupSurface(...)`. El
adaptador/asistente generado adopta un comportamiento de cierre seguro en las escrituras de configuración y la finalización, y reutiliza
el mismo mensaje de instalación obligatoria en la validación, la finalización y el texto
del enlace a la documentación.

Si el canal admite configuración o autenticación mediante variables de entorno, debe exponerse a través del
esquema de configuración del canal y los descriptores de configuración. Mantenga `envVars` del entorno de ejecución del canal o
las constantes locales únicamente para el texto dirigido al operador.

Si el canal puede aparecer en `status`, `channels list`, `channels status` o
los análisis de SecretRef antes de que se inicie el entorno de ejecución del plugin, añada `openclaw.setupEntry` en
`package.json`. Ese punto de entrada debe poder importarse de forma segura en rutas de comandos
de solo lectura y debe devolver los metadatos del canal, el adaptador de configuración
seguro para la configuración, el adaptador de estado y los metadatos de destino de secretos del canal necesarios para esos
resúmenes. No inicie clientes, escuchadores ni entornos de ejecución de transporte desde la
entrada de configuración.

Mantenga también acotada la ruta de importación de la entrada principal del canal. El descubrimiento puede evaluar
la entrada y el módulo del plugin del canal para registrar capacidades sin
activar el canal. Los archivos como `channel-plugin-api.ts` deben exportar
el objeto del plugin del canal sin importar asistentes de configuración, clientes
de transporte, escuchadores de sockets, lanzadores de subprocesos ni módulos de inicio de servicios.
Coloque esos componentes del entorno de ejecución en módulos cargados desde `registerFull(...)`, establecedores
del entorno de ejecución o adaptadores de capacidades con carga diferida.

### Otras subrutas acotadas del canal

Para otras rutas de acceso frecuente del canal, se deben preferir los auxiliares acotados frente a superficies
heredadas más amplias:

- `openclaw/plugin-sdk/account-core`, `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` y
  `openclaw/plugin-sdk/account-helpers` para la configuración de varias cuentas y
  la alternativa de cuenta predeterminada
- `openclaw/plugin-sdk/inbound-envelope` y
  `openclaw/plugin-sdk/channel-inbound` para el enrutamiento/sobre de entrada y
  el cableado de registro y despacho
- `openclaw/plugin-sdk/channel-targets` para los auxiliares de análisis de destinos
- `openclaw/plugin-sdk/channel-outbound` para los delegados de identidad/envío de salida
  y la planificación tipada de cargas útiles
- `buildThreadAwareOutboundSessionRoute(...)` de
  `openclaw/plugin-sdk/channel-core` cuando una ruta de salida deba conservar
  un `replyToId`/`threadId` explícito o recuperar la sesión `:thread:`
  actual después de que la clave de sesión base siga coincidiendo. Los plugins de proveedores pueden
  reemplazar la precedencia, el comportamiento de los sufijos y la normalización del identificador de hilo cuando
  su plataforma tenga semántica nativa de entrega en hilos.
- `openclaw/plugin-sdk/thread-bindings-runtime` para el ciclo de vida de la vinculación de hilos
  y el registro de adaptadores

Por lo general, los canales exclusivamente de autenticación pueden limitarse a la ruta predeterminada: el núcleo gestiona
las aprobaciones y el plugin solo expone capacidades de salida/autenticación. Los canales
de aprobación nativa, como Matrix, Slack, Telegram y los transportes de chat personalizados,
deben utilizar los auxiliares nativos compartidos en lugar de implementar su propio ciclo de vida
de aprobación.

## Política de menciones de entrada

Mantenga el tratamiento de las menciones de entrada dividido en dos capas:

- recopilación de indicios propiedad del plugin
- evaluación de políticas compartida

Use `openclaw/plugin-sdk/channel-mention-gating` para las decisiones de la política de menciones.
Use `openclaw/plugin-sdk/channel-inbound` solo cuando necesite el módulo de exportaciones
más amplio de auxiliares de entrada.

Adecuado para la lógica local del plugin:

- detección de respuestas al bot
- detección de citas del bot
- comprobaciones de participación en hilos
- exclusiones de mensajes de servicio/sistema
- cachés nativas de la plataforma necesarias para demostrar la participación del bot

Adecuado para el auxiliar compartido:

- `requireMention`
- resultado de mención explícita
- lista de permitidos para menciones implícitas
- omisión para comandos
- decisión final de omisión

Flujo recomendado:

1. Calcule los hechos locales sobre menciones.
2. Pase esos hechos a `resolveInboundMentionDecision({ facts, policy })`.
3. Use `decision.effectiveWasMentioned`, `decision.shouldBypassMention` y
   `decision.shouldSkip` en la puerta de entrada.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` devuelve un booleano. `hasAnyMention`,
`isExplicitlyMentioned` y `canResolveExplicit` proceden de los metadatos de menciones
nativos del propio canal (entidades de mensajes, indicadores de respuesta al bot y similares);
proporcione valores `false`/`undefined` cuando la plataforma no pueda detectarlos.

`api.runtime.channel.mentions` expone los mismos auxiliares compartidos de menciones para
los plugins de canales incluidos que ya dependen de la inyección del entorno de ejecución:
`buildMentionRegexes`, `matchesMentionPatterns`, `matchesMentionWithExplicit`,
`implicitMentionKindWhen`, `resolveInboundMentionDecision`.

Si solo necesita `implicitMentionKindWhen` y `resolveInboundMentionDecision`,
importe desde `openclaw/plugin-sdk/channel-mention-gating` para evitar cargar
auxiliares del entorno de ejecución de entrada que no estén relacionados.

## Guía paso a paso

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paquete y manifiesto">
    Cree los archivos estándar del plugin. El campo `channels` de
    `openclaw.plugin.json` (no un campo `kind`) es el que marca un manifiesto como
    propietario de un canal. Para consultar toda la superficie de metadatos del paquete, véase
    [Configuración del plugin](/es/plugins/sdk-setup#openclaw-channel):

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "Conecte OpenClaw con Acme Chat."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Plugin del canal Acme Chat",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "Token del bot",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema` valida `plugins.entries.acme-chat.config`. Úselo para
    la configuración propiedad del plugin que no forme parte de la configuración de la cuenta del canal.
    `channelConfigs.acme-chat.schema` valida `channels.acme-chat` y es la
    fuente de la ruta fría que utilizan el esquema de configuración, la configuración y las superficies de la interfaz de usuario antes de que
    se cargue el entorno de ejecución del plugin. Consulte [Manifiesto del plugin](/es/plugins/manifest) para ver la referencia
    completa de los campos de nivel superior.

  </Step>

  <Step title="Crear el objeto del plugin del canal">
    La interfaz `ChannelPlugin` tiene muchas superficies de adaptadores opcionales. Comience con
    el mínimo —`id`, `config` y `setup`— y añada adaptadores a medida que
    los necesite.

    Cree `src/channel.ts`:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    Para los canales que aceptan tanto claves canónicas de nivel superior de mensajes directos como claves anidadas heredadas, utilice las funciones auxiliares de `plugin-sdk/channel-config-helpers`: `resolveChannelDmAccess`, `resolveChannelDmPolicy`, `resolveChannelDmAllowFrom` y `normalizeChannelDmPolicy` mantienen los valores locales de la cuenta por delante de los valores raíz heredados. Vincule el mismo resolutor con la reparación de doctor mediante `normalizeLegacyDmAliases` para que el entorno de ejecución y la migración lean el mismo contrato.

    <Accordion title="Qué hace createChatChannelPlugin por usted">
      En lugar de implementar manualmente interfaces de adaptador de bajo nivel, se proporcionan
      opciones declarativas y el constructor las combina:

      | Opción | Qué conecta |
      | --- | --- |
      | `security.dm` | Resolutor de seguridad de mensajes directos con ámbito a partir de campos de configuración |
      | `pairing.text` | Flujo de vinculación de mensajes directos basado en texto con intercambio de códigos |
      | `threading` | Resolutor del modo de respuesta (fijo, con ámbito de cuenta o personalizado) |
      | `outbound.attachedResults` | Funciones de envío que devuelven metadatos del resultado (identificadores de mensajes); requiere un identificador `channel` hermano para que el núcleo pueda registrar el resultado de entrega devuelto |

      También se pueden proporcionar objetos de adaptador sin procesar en lugar de las opciones declarativas
      si se necesita un control total.

      Los adaptadores de salida sin procesar pueden definir una función `chunker(text, limit, ctx)`.
      El valor opcional `ctx.formatting` contiene decisiones de formato en el momento de la entrega,
      como `maxLinesPerMessage`; aplíquelo antes del envío para que la estructuración de respuestas
      y los límites de fragmentos se resuelvan una sola vez mediante la entrega de salida compartida.
      Los contextos de envío también incluyen `replyToIdSource` (`implicit` o `explicit`)
      cuando se ha resuelto un destino de respuesta nativo, para que las funciones auxiliares de carga útil puedan conservar
      etiquetas de respuesta explícitas sin consumir una ranura de respuesta implícita de un solo uso.
    </Accordion>

    ### Adaptadores de políticas de herramientas de grupo

    Un canal que implemente `group.resolveToolPolicy` y admita
    `toolsBySender` debe reenviar el `ChannelGroupContext` completo a su
    solucionador de políticas compartido. En particular, debe respetar `senderPolicyMode: "never"`
    omitiendo las superposiciones específicas del remitente tanto en el ámbito del grupo coincidente como en el del comodín,
    sin dejar de aplicar la política base `tools`.

    OpenClaw establece este modo únicamente para ejecuciones de confianza que no sean de entrada y cuya autoridad del remitente
    ya se haya capturado en un sobre propiedad del servidor, como una
    ejecución programada con límites explícitos. Los Plugins no deben derivar el modo de
    los metadatos entrantes, conservarlo como estado del canal ni exponerlo como configuración. Añada
    una prueba del adaptador que demuestre que el modo omite una entrada comodín `toolsBySender`
    sin eliminar la restricción base `tools` coincidente.

  </Step>

  <Step title="Conectar el punto de entrada">
    Cree `index.ts`:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Plugin de canal de Acme Chat",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Administración de Acme Chat");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Administración de Acme Chat",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Coloque los descriptores de la CLI propiedad del canal en `registerCliMetadata(...)` para que OpenClaw
    pueda mostrarlos en la ayuda raíz sin activar el entorno de ejecución completo del canal,
    mientras que las cargas completas normales sigan recogiendo los mismos descriptores para el registro real de
    comandos. Reserve `registerFull(...)` para tareas exclusivas del entorno de ejecución.
    `defineChannelPluginEntry` gestiona automáticamente la división por modo de registro.
    Si `registerFull(...)` registra métodos RPC del Gateway, utilice un
    prefijo específico del Plugin. Los espacios de nombres administrativos del núcleo (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) permanecen reservados y siempre
    se resuelven como `operator.admin`. Consulte
    [Puntos de entrada](/es/plugins/sdk-entrypoints#definechannelpluginentry) para ver todas las
    opciones.

  </Step>

  <Step title="Añadir una entrada de configuración">
    Cree `setup-entry.ts` para una carga ligera durante la incorporación:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw carga esto en lugar del punto de entrada completo cuando el canal está deshabilitado
    o no está configurado. Evita cargar código de ejecución pesado durante los flujos de configuración.
    Consulte [Configuración inicial y configuración](/es/plugins/sdk-setup#setup-entry) para obtener más información.

    Los canales incluidos en el espacio de trabajo que separan las exportaciones seguras para la configuración en módulos
    complementarios pueden usar `defineBundledChannelSetupEntry(...)` de
    `openclaw/plugin-sdk/channel-entry-contract` cuando también necesitan un
    definidor explícito del entorno de ejecución durante la configuración.

  </Step>

  <Step title="Gestionar los mensajes entrantes">
    El plugin necesita recibir mensajes de la plataforma y reenviarlos a
    OpenClaw. El patrón habitual es un webhook que verifica la solicitud y
    la despacha mediante el controlador de mensajes entrantes del canal:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // autenticación gestionada por el plugin (verifique las firmas por su cuenta)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // El controlador de mensajes entrantes despacha el mensaje a OpenClaw.
          // La integración exacta depende del SDK de la plataforma:
          // consulte un ejemplo real en el paquete del plugin incluido de Microsoft Teams o Google Chat.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      La gestión de mensajes entrantes es específica de cada canal. Cada plugin de canal posee
      su propio pipeline de entrada. Consulte los plugins de canal incluidos
      (por ejemplo, el paquete del plugin de Microsoft Teams o Google Chat) para ver patrones reales.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Probar">
Escriba pruebas ubicadas junto al código en `src/channel.test.ts`:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("plugin acme-chat", () => {
      it("resuelve la cuenta desde la configuración", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspecciona la cuenta sin materializar secretos", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("informa de la ausencia de configuración", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    Para consultar los auxiliares de prueba compartidos, consulte [Pruebas](/es/plugins/sdk-testing).

</Step>
</Steps>

## Estructura de archivos

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # metadatos de openclaw.channel
├── openclaw.plugin.json      # Manifiesto con el esquema de configuración
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Exportaciones públicas (opcional)
├── runtime-api.ts            # Exportaciones internas del entorno de ejecución (opcional)
└── src/
    ├── channel.ts            # ChannelPlugin mediante createChatChannelPlugin
    ├── channel.test.ts       # Pruebas
    ├── client.ts             # Cliente de la API de la plataforma
    └── runtime.ts            # Almacén del entorno de ejecución (si es necesario)
```

## Temas avanzados

<CardGroup cols={2}>
  <Card title="Opciones de hilos" icon="git-branch" href="/es/plugins/sdk-entrypoints#registration-mode">
    Modos de respuesta fijos, personalizados o con ámbito de cuenta
  </Card>
  <Card title="Integración con la herramienta de mensajes" icon="puzzle" href="/es/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool y detección de acciones
  </Card>
  <Card title="Resolución de destinos" icon="crosshair" href="/es/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType, looksLikeId, reservedLiterals, resolveTarget
  </Card>
  <Card title="Utilidades de runtime" icon="settings" href="/es/plugins/sdk-runtime">
    TTS, STT, contenido multimedia y subagente mediante api.runtime
  </Card>
  <Card title="API de entrada de canales" icon="bolt" href="/es/plugins/sdk-channel-inbound">
    Ciclo de vida compartido de los eventos de entrada: ingesta, resolución, registro, despacho y finalización
  </Card>
</CardGroup>

<Note>
Todavía existen algunos puntos de integración de utilidades incluidas para el mantenimiento y la
compatibilidad de los plugins incluidos. No son el patrón recomendado para los nuevos plugins de canal;
se deben preferir las subrutas genéricas de canal/configuración/respuesta/runtime de la superficie común del SDK,
salvo que se mantenga directamente esa familia de plugins incluidos.
</Note>

## Siguientes pasos

- [Plugins de proveedor](/es/plugins/sdk-provider-plugins) - si el plugin también proporciona modelos
- [Descripción general del SDK](/es/plugins/sdk-overview) - referencia completa de importación de subrutas
- [Pruebas del SDK](/es/plugins/sdk-testing) - utilidades de prueba y pruebas de contrato
- [Manifiesto del plugin](/es/plugins/manifest) - esquema completo del manifiesto

## Contenido relacionado

- [Configuración del SDK de plugins](/es/plugins/sdk-setup)
- [Creación de plugins](/es/plugins/building-plugins)
- [Plugins del arnés de agentes](/es/plugins/sdk-agent-harness)
