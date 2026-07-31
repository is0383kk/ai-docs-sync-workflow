---
read_when:
    - Necesita llamar a funciones auxiliares del núcleo desde un plugin (TTS, STT, generación de imágenes, búsqueda web, Gateway, subagente, nodos)
    - Quieres entender qué expone api.runtime
    - Estás accediendo a ayudantes de configuración, agentes o medios desde el código del plugin
sidebarTitle: Runtime helpers
summary: api.runtime -- los auxiliares de runtime inyectados disponibles para los plugins
title: Funciones auxiliares del entorno de ejecución de Plugin
x-i18n:
    generated_at: "2026-07-26T05:15:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

Referencia del objeto `api.runtime` inyectado en cada plugin durante el registro. Utilice estos asistentes en lugar de importar directamente los componentes internos del host.

<CardGroup cols={2}>
  <Card title="Plugins de canal" href="/es/plugins/sdk-channel-plugins">
    Guía paso a paso que utiliza estos asistentes en contexto para los plugins de canal.
  </Card>
  <Card title="Plugins de proveedor" href="/es/plugins/sdk-provider-plugins">
    Guía paso a paso que utiliza estos asistentes en contexto para los plugins de proveedor.
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` es la versión actual del producto OpenClaw, obtenida del solucionador de versiones compartido para que los plugins vean el mismo valor que muestra la CLI.

## Carga y escritura de la configuración

Es preferible usar la configuración que ya se haya pasado a la ruta de llamada activa; por ejemplo, `api.config` durante el registro o un argumento `cfg` en las devoluciones de llamada de canales o proveedores. Así, una única instantánea del proceso fluye por el trabajo en lugar de volver a analizar la configuración en rutas críticas.

Utilice `api.runtime.config.current()` solo cuando un controlador de larga duración necesite la instantánea actual del proceso y no se haya pasado ninguna configuración a esa función. El valor devuelto es de solo lectura; clónelo o utilice un asistente de mutación antes de editarlo.

Las fábricas de herramientas reciben `ctx.runtimeConfig` junto con `ctx.getRuntimeConfig()`. Utilice el captador dentro de la devolución de llamada `execute` de una herramienta de larga duración cuando la configuración pueda cambiar después de crear la definición de la herramienta.

Conserve los cambios con `api.runtime.config.mutateConfigFile(...)` o `api.runtime.config.replaceConfigFile(...)`. Cada escritura debe elegir una política `afterWrite` explícita:

- `afterWrite: { mode: "auto" }` permite que el planificador de recarga del Gateway decida.
- `afterWrite: { mode: "restart", reason: "..." }` fuerza un reinicio limpio cuando el escritor sabe que la recarga en caliente no es segura.
- `afterWrite: { mode: "none", reason: "..." }` impide la recarga o el reinicio automáticos solo cuando el llamador se encarga del seguimiento.

Los asistentes de mutación devuelven `afterWrite` junto con un resumen `followUp` tipado para que los llamadores puedan registrar o comprobar si solicitaron un reinicio. El Gateway sigue controlando cuándo se produce realmente dicho reinicio.

Utilice `current()`, un `cfg` pasado, `mutateConfigFile(...)` o
`replaceConfigFile(...)` para acceder a la configuración en tiempo de ejecución y escribirla.

Para las importaciones directas del SDK, es preferible usar las subrutas de configuración específicas en lugar del barrel de compatibilidad general `openclaw/plugin-sdk/config-runtime`: `config-contracts` para los tipos, `runtime-config-snapshot` para las instantáneas actuales del proceso y `config-mutation` para las escrituras. Lea los valores específicos de la entrada desde `api.pluginConfig`; utilice un contexto de herramienta proporcionado solo para su instantánea de configuración de todo el entorno de ejecución y mantenga la combinación específica del plugin en ese límite. Las pruebas de plugins incluidos deben simular directamente estas subrutas específicas en lugar de simular el barrel de compatibilidad general.

El código interno del entorno de ejecución de OpenClaw sigue el mismo enfoque: cargar la configuración una vez en el límite de la CLI, del Gateway o del proceso y, después, pasar ese valor. Las escrituras de mutación correctas actualizan la instantánea del entorno de ejecución del proceso e incrementan su revisión interna; las cachés de larga duración deben usar como clave la clave de caché propiedad del entorno de ejecución, en lugar de serializar la configuración localmente. Los módulos del entorno de ejecución de larga duración cuentan con un analizador de tolerancia cero para las llamadas ambientales a `loadConfig()`; utilice un `cfg` pasado, un `context.getRuntimeConfig()` de la solicitud o `getRuntimeConfig()` en un límite explícito del proceso.

Las rutas de ejecución de proveedores y canales deben utilizar la instantánea activa de la configuración del entorno de ejecución, no una instantánea del archivo devuelta para consultar o editar la configuración. Las instantáneas de archivos conservan valores de origen, como los marcadores SecretRef, para la interfaz de usuario y las escrituras; las devoluciones de llamada de proveedores necesitan la vista resuelta del entorno de ejecución. Cuando se pueda llamar a un asistente con la instantánea activa del origen o con la instantánea activa del entorno de ejecución, enrute mediante `selectApplicableRuntimeConfig()` antes de leer las credenciales.

## Utilidades reutilizables del entorno de ejecución

Utilice los datos `botLoopProtection` entrantes para los mensajes entrantes creados por bots. El núcleo aplica la protección compartida de ventana deslizante en memoria antes del registro y el despacho de la sesión, sin vincular la política a un único canal. La protección realiza un seguimiento de las claves `(scopeId, conversationId, participant pair)`, cuenta conjuntamente ambas direcciones de un par, aplica un período de espera cuando se supera el límite de la ventana y elimina de forma oportunista las entradas inactivas.

Los plugins de canal que expongan este comportamiento a los operadores deben usar preferentemente la estructura compartida `channels.defaults.botLoopProtection` para los límites de referencia y, después, superponer las anulaciones específicas del canal o proveedor. La configuración compartida utiliza segundos porque está orientada al usuario:

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

Pase los datos normalizados del par de bots con el turno resuelto. El núcleo resuelve los valores predeterminados, la conversión de unidades y la semántica de `enabled`:

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

Utilice `openclaw/plugin-sdk/pair-loop-guard-runtime` directamente solo para bucles de eventos personalizados
entre dos partes que no pasen por el ejecutor compartido de respuestas entrantes.

## Espacios de nombres del entorno de ejecución

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    Identidad del agente, directorios y gestión de sesiones.

    ```typescript
    // Resolver el directorio de trabajo del agente (agentId es obligatorio)
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // Resolver el espacio de trabajo del agente
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // Obtener la identidad del agente
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // Obtener el nivel de razonamiento predeterminado
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // Validar un nivel de razonamiento proporcionado por el usuario respecto al perfil del proveedor activo
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // pasar el nivel a una ejecución integrada
    }

    // Obtener el tiempo de espera del agente
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // Asegurarse de que exista el espacio de trabajo
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // Ejecutar un turno de agente integrado
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "Resumir los cambios más recientes",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` es el asistente neutral para iniciar un turno normal de un agente de OpenClaw desde el código de un plugin. Utiliza la misma resolución de proveedor y modelo, así como la misma selección del arnés del agente, que las respuestas activadas por canales.

    `runEmbeddedPiAgent(...)` se mantiene como alias de compatibilidad obsoleto para los plugins existentes. El código nuevo debe utilizar `runEmbeddedAgent(...)`.

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` comparte la decisión de despacho al backend de la CLI del ejecutor integrado (la ruta, la capacidad `subscriptionAuthDispatch` declarada por el backend y el modo de credenciales almacenado —respetando un `authProfileId` fijado explícitamente—) con los llamadores que incorporan las ejecuciones integradas a `cliBackendDispatch: "subscription-auth"`. Devuelve `{ provider }` cuando la ejecución se realizaría mediante el backend de la CLI y `undefined` cuando permanece en el paso directo, para que los llamadores puedan asignar los tiempos de espera de la ejecución que realmente tendrá lugar.

    `resolveThinkingPolicy(...)` devuelve los niveles de razonamiento admitidos por el proveedor o modelo y el valor predeterminado opcional. Los plugins de proveedor controlan el perfil específico del modelo mediante sus hooks de razonamiento, por lo que los plugins de herramientas deben llamar a este asistente del entorno de ejecución en lugar de importar o duplicar listas de proveedores.

    `normalizeThinkingLevel(...)` convierte texto del usuario como `on`, `x-high` o `extra high` al nivel canónico almacenado antes de compararlo con la política resuelta.

    Los **asistentes del almacén de sesiones** se encuentran en `api.runtime.agent.session`:

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // Iterar por las filas de sesiones sin depender de la estructura heredada de sessions.json.
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // Crear o actualizar la sesión y, después, pasar signal a la ejecución admitida del agente.
      },
    );
    ```

    Utilice preferentemente `getSessionEntry(...)`, `listSessionEntries(...)`, `patchSessionEntry(...)` o `upsertSessionEntry(...)` para los flujos de trabajo de sesiones. Estos asistentes identifican las sesiones mediante la identidad del agente y de la sesión para que los plugins no dependan de la estructura de almacenamiento heredada `sessions.json`. Utilice `preserveActivity: true` para modificaciones exclusivamente de metadatos que no deban actualizar la actividad de la sesión, y `replaceEntry: true` solo cuando la devolución de llamada devuelva una entrada completa y los campos eliminados deban permanecer eliminados. Las rutas de Doctor y migración pueden combinar `fallbackEntry`, `skipMaintenance` y `requireWriteSuccess` para realizar una única reparación atómica del almacén canónico.

    `createSessionEntry(...)` crea una nueva fila de sesión canónica y su transcripción. Su superficie de confianza `initialEntry` es deliberadamente limitada: un `agentHarnessId` no vacío, un `modelSelectionLocked: true` opcional y un `pluginExtensions` opcional. El entorno de ejecución inyectado solo acepta identificadores de arnés que pertenezcan al plugin llamador mediante `registerAgentHarness(...)`; se trata de una invariante de propiedad, no de un entorno aislado entre plugins del mismo proceso. Rechaza una fila existente; `label` y `spawnedCwd` son campos de creación independientes en lugar de modificaciones de entradas de confianza.

    La creación mantiene el cerrojo de mutación del ciclo de vida de la sesión mediante `afterCreate`, por lo que el trabajo nuevo espera a que finalice la inicialización propiedad del plugin, y el trabajo admitido previamente hace que falle la creación. La devolución de llamada recibe un clon del estado creado. Si devuelve una modificación, esta solo puede contener `pluginExtensions`, y su valor constituye el campo `pluginExtensions` final completo. Un fallo de la devolución de llamada o de la persistencia final revierte la nueva fila sin cambios y la transcripción; la reversión protegida conserva cualquier fila modificada o reclamada simultáneamente. `recoverMatchingInitialEntry: true` sirve únicamente para reintentar una inicialización interrumpida cuando los campos de confianza conservados coinciden exactamente, y la recuperación exige que `afterCreate` devuelva una modificación final.

    Utilice `runWithWorkAdmission(...)` cuando un plugin inicie trabajo en una sesión persistente. La devolución de llamada rechaza las sesiones archivadas o reemplazadas simultáneamente, mantiene coordinadas hasta su finalización las mutaciones de archivado, restablecimiento y eliminación, y recibe un `AbortSignal` que debe reenviarse a la ejecución del agente. Un arnés puede designar explícitamente delegados de ejecución de confianza mediante su campo de registro experimental `delegatedExecutionPluginIds`. Los delegados solo pueden admitir y ejecutar una sesión existente exacta con el modelo bloqueado; todas las mutaciones de la sesión siguen restringidas al propietario del arnés. Consulte [Plugins de arnés de agente](/es/plugins/sdk-agent-harness#delegated-execution).

    Los plugins de mantenimiento y reparación pueden usar `deleteSessionEntry(...)` para una entrada de sesión con ámbito específico, `cleanupSessionLifecycleArtifacts(...)` para sesiones temporales administradas por el ciclo de vida y `resolveSessionStoreBackupPaths(...)` antes de modificar un almacén. Pase `expectedSessionId` y `expectedUpdatedAt` cuando la eliminación no deba entrar en conflicto con una actualización simultánea de la sesión; use `expectedSessionId: null` cuando la instantánea anterior no tuviera un id de sesión. Estos asistentes son superficies específicas de reparación y ciclo de vida, no una API general de eliminación de almacenes.

    `resolveStorePath(...)` y `updateSessionStoreEntry(...)` completan los asistentes de sesión: `resolveStorePath` resuelve la ruta del almacén de sesiones para un ámbito determinado, y `updateSessionStoreEntry({ storePath, sessionKey, update })` modifica directamente una entrada mediante la ruta del almacén cuando el llamador ya la conoce.

    `loadTranscriptEventsSync(...)` está disponible para las rutas síncronas de diagnóstico y reparación que no pueden usar el entorno de ejecución asíncrono de transcripciones. Devuelve registros `SessionStoreTranscriptEvent` sin procesar. El código normal del entorno de ejecución de plugins debe usar preferentemente `openclaw/plugin-sdk/session-transcript-runtime`.

    `formatSqliteSessionFileMarker(...)`, `parseSqliteSessionFileMarker(...)` y `sqliteSessionFileMarkerMatchesSession(...)` son asistentes de transición para código que aún recibe un campo heredado llamado `sessionFile`. Un marcador de SQLite analizado identifica un destino activo de transcripción de SQLite; no es una ruta del sistema de archivos. Las API nuevas deben transportar una identidad de sesión tipada en lugar de cadenas de marcadores.

    Para leer y escribir transcripciones, importe `openclaw/plugin-sdk/session-transcript-runtime` y use `resolveSessionTranscriptIdentity(...)`, `resolveSessionTranscriptTarget(...)`, `readSessionTranscriptEvents(...)`, `readSessionTranscriptRawDelta(...)`, `readSessionTranscriptVisibleMessageDelta(...)`, `readVisibleSessionTranscriptMessageEntries(...)`, `appendSessionTranscriptMessageByIdentity(...)`, `publishSessionTranscriptUpdateByIdentity(...)` o `withSessionTranscriptWriteLock(...)` con `{ agentId, sessionKey, sessionId }`. Estas API permiten que los plugins identifiquen una transcripción, lean eventos sin procesar o entradas de mensajes visibles seguras para las ramas, anexen mensajes, publiquen actualizaciones y ejecuten operaciones relacionadas bajo el mismo bloqueo de escritura de la transcripción sin depender de rutas de archivos de transcripciones activas. `readVisibleSessionTranscriptMessageEntries(...)` devuelve metadatos de lectura ordenados; su campo `seq` no es un cursor reanudable.

    `appendSessionTranscriptMessageByIdentity(...)` es una operación de anexado de bajo nivel de un mensaje que ya es canónico. Los plugins no deben sintetizar filas de usuario con contenido multimedia mediante `MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType` o `MediaTypes` en el nivel superior. La entrada del canal debe pasar los datos ordenados mediante `MsgContext.media` y permitir que el host administre la persistencia del turno del usuario. Un mensaje de usuario persistido y preparado por el host contiene datos ordenados canónicos en `message.__openclaw.media`; la API genérica de anexado no infiere ni repara matrices paralelas heredadas.

    `readSessionTranscriptRawDelta(...)` devuelve un resultado limitado `page`, `reset` o `missing`. Pase el valor opaco `page.cursor` a la siguiente llamada. Los anexados puros conservan el cursor, mientras que la sustitución de la transcripción devuelve `reset` con un nuevo cursor de arranque. Las páginas tienen valores predeterminados de 1,000 eventos y 1,000,000 de bytes serializados; los llamadores pueden solicitar hasta 10,000 eventos y 64 MiB. Cuando solo el siguiente evento supera `maxBytes`, la página está vacía e informa de `requiredBytes`; vuelva a intentarlo con al menos ese límite de bytes cuando no sea superior a 64 MiB. Los eventos individuales más grandes requieren la API de lectura completa. Un cursor solo identifica una posición y nunca concede acceso a otra sesión.

    `readSessionTranscriptVisibleMessageDelta(...)` proporciona la misma estructura limitada de arranque y reanudación sobre la proyección activa de mensajes administrada por el host. Devuelve los mensajes del más antiguo al más reciente, para que los motores de contexto puedan consumir el historial inicial y conservar el cursor opaco como su marca de agua. Almacene y devuelva el cursor sin modificarlo; es una indicación de continuación, no una credencial de autorización. Los anexados lineales se reanudan después del último mensaje devuelto. La sustitución de la transcripción, un cursor cuyo anclaje salió de la rama activa o se desplazó dentro de ella, los cursores con formato incorrecto y los cursores entre sesiones devuelven `reset` con un nuevo cursor de arranque. Los valores predeterminados y los límites de cantidad y bytes coinciden con los de la API de deltas sin procesar. Mientras la proyección activa se reconstruye tras un cambio de rama, el resultado es `unavailable` con el motivo `projection_rebuilding`; vuelva a intentarlo más tarde en lugar de recurrir a un archivo de transcripción activo.

    Los asistentes heredados para el almacén completo y el archivo de transcripción activo ya no se exportan desde el SDK de plugins. Use los asistentes de entradas con ámbito específico para los metadatos de sesión y los asistentes de identidad de transcripciones para las operaciones de transcripciones activas. Los flujos de trabajo de archivado y asistencia que necesiten artefactos de archivo deben usar sus superficies específicas de archivado en lugar de las API del entorno de ejecución de sesiones activas.

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    Constantes predeterminadas de modelo y proveedor:

    ```typescript
    const model = api.runtime.agent.defaults.model; // p. ej., "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // p. ej., "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    Ejecute una finalización de texto administrada por el host sin importar elementos internos del proveedor ni
    duplicar la preparación del modelo, la autenticación y la URL base de OpenClaw.

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "Resume esta transcripción." }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    La orquestación del proveedor también puede adquirir el ciclo de vida
    configurado del servicio local antes de emitir una solicitud HTTP:

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // Envíe y consuma por completo la solicitud del proveedor.
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` es un contrato estable y genérico del SDK de servicios
    de proveedores. El host resuelve la configuración del proceso desde
    `models.providers.<providerId>.localService`; los llamadores no pueden proporcionar un
    comando, argumentos, entorno ni política de ciclo de vida. La creación de procesos,
    la disponibilidad, los diagnósticos y la política de detención por inactividad siguen siendo internos del host.

    Pase el id exacto del proveedor configurado y la URL base resuelta de la solicitud. No
    sustituya los alias por un id de adaptador: distintos alias pueden apuntar a distintos
    hosts de GPU locales. El host rechaza los puntos de conexión que no coincidan con la URL base
    configurada del proveedor, salvo la normalización `/v1` utilizada por los adaptadores de Ollama y LM
    Studio. El host administra la serialización del inicio, las comprobaciones de disponibilidad,
    las concesiones de solicitudes, la gestión de cancelaciones y el apagado por inactividad.

    El asistente usa la misma ruta de preparación de finalizaciones simples que el
    entorno de ejecución integrado de OpenClaw y la instantánea de configuración del entorno de ejecución administrada por el host. Los motores de contexto
    reciben una capacidad `llm.complete` vinculada a la sesión, por lo que las llamadas al modelo usan el
    agente de la sesión activa y no recurren silenciosamente al agente predeterminado. El
    resultado incluye la atribución de proveedor, modelo y agente, además del uso normalizado de tokens,
    caché y coste estimado cuando está disponible.

    Establezca `reasoning` para solicitar un esfuerzo de razonamiento para el modelo seleccionado. El
    host normaliza los niveles canónicos de razonamiento (`off`, `minimal`, `low`,
    `medium`, `high`, `xhigh`, `adaptive`, `max` y `ultra`) para el
    proveedor y el modelo seleccionados antes de enviar la finalización. `adaptive` se convierte en
    `medium`; `max` y `ultra` se convierten en `max` cuando se admiten; de lo contrario, en `xhigh`.

    <Warning>
    Las sustituciones de modelos requieren la aceptación explícita del operador mediante `plugins.entries.<id>.llm.allowModelOverride: true` en la configuración. Use `plugins.entries.<id>.llm.allowedModels` para restringir los plugins de confianza a destinos canónicos `provider/model` específicos. Las finalizaciones entre agentes requieren `plugins.entries.<id>.llm.allowAgentIdOverride: true`.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    Llame a otro método del Gateway dentro del proceso y conserve la identidad de confianza del entorno de ejecución
    del plugin actual. Esto está pensado para plugins oficiales integrados o de confianza que componen capacidades
    del Gateway administradas por plugins sin abrir una conexión WebSocket de bucle invertido.

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    Las solicitudes usan el ámbito `operator.write` y no conceden el ámbito de administración. Se rechazan las llamadas de plugins
    externos arbitrarios. Los métodos fallidos generan una excepción `GatewayClientRequestError` que conserva `details` estructurado,
    los metadatos de reintento y el código de error del Gateway para los flujos de recuperación. Use `isAvailable()`
    antes de elegir esta ruta desde herramientas que también pueden ejecutarse en procesos de agentes independientes.

  </Accordion>
  <Accordion title="api.runtime.subagent">
    Inicie y administre ejecuciones de subagentes en segundo plano.

    ```typescript
    // Iniciar una ejecución de subagente
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "Amplía esta consulta para convertirla en búsquedas de seguimiento específicas.",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // sustitución opcional
      model: "gpt-5.6-sol", // sustitución opcional
      deliver: false,
    });

    // Esperar a que finalice
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // Leer los mensajes de la sesión
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // Eliminar una sesión
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    Las sustituciones de modelos (`provider`/`model`) requieren la aceptación explícita del operador mediante `plugins.entries.<id>.subagent.allowModelOverride: true` en la configuración. Los plugins que no sean de confianza pueden seguir ejecutando subagentes, pero las solicitudes de sustitución se rechazan.
    </Warning>

    `toolsAlsoAllow` añade a la superficie normal de herramientas del trabajador herramientas exactas y con propietario único registradas por el plugin llamador. El entorno de ejecución rechaza las herramientas del núcleo y los nombres compartidos con otro plugin. Los perfiles y las políticas de herramientas del operador siguen aplicándose, incluidas las listas explícitas de elementos permitidos y las denegaciones.

    `deleteSession(...)` puede eliminar sesiones creadas por el mismo plugin mediante `api.runtime.subagent.run(...)`. Eliminar sesiones arbitrarias de usuarios u operadores sigue requiriendo una solicitud del Gateway con ámbito de administración.

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    Inspeccione la autoridad efectiva del espacio de trabajo aislado para una sesión de agente.

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    El resultado indica si esta sesión está aislada, si su espacio de trabajo
    no está disponible, es de solo lectura o permite escritura, y proporciona un valor opcional `confinementError`
    cuando la política efectiva de Docker, herramientas, sesión, navegador o privilegios elevados puede
    escapar de ese espacio de trabajo. Use este resultado para las decisiones de delegación administradas por el host que
    no deben conceder a un trabajador más autoridad que la de su llamador. Es un asistente de certificación,
    no un sustituto de la comprobación de la autorización propia del llamador.

    `prepareWorkspaceAuthority(...)` realiza la misma comprobación de políticas y también
    prepara el entorno aislado de Docker para `workspaceDir`. Rechaza un contenedor activo
    cuyo hash de configuración en ejecución no coincida con los montajes o la política solicitados. Pase
    únicamente los nombres exactos de las herramientas cuyas implementaciones registradas restringe el plugin
    llamador; los prefijos comodín no demuestran la propiedad de la herramienta.

  </Accordion>
  <Accordion title="api.runtime.nodes">
    Enumere los nodos conectados e invoque un comando del host de un nodo desde el código de un plugin cargado por el Gateway o desde comandos de la CLI del plugin. Use esta opción cuando un plugin administre trabajo local en un dispositivo emparejado, por ejemplo, un puente de navegador o audio en otro Mac.

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` incluye los descriptores `nodePluginTools` anunciados
    por cada Node conectado cuando ese Node expone al agente herramientas
    respaldadas por plugins o MCP. Esos descriptores representan el estado de
    conexión en vivo: el Gateway los elimina cuando el Node se desconecta, y un
    Node puede reemplazarlos por `node.pluginTools.update` después de que cambie el
    inventario local de plugins/MCP.

    Dentro del Gateway, este entorno de ejecución se ejecuta en el mismo proceso. En los comandos de la CLI de los plugins, llama al Gateway configurado mediante RPC, por lo que comandos como `openclaw googlemeet recover-tab` pueden inspeccionar los Nodes emparejados desde el terminal. Los comandos de Node siguen pasando por el emparejamiento normal de Nodes del Gateway, las listas de comandos permitidos, las políticas de invocación de Nodes de los plugins y el procesamiento local de comandos del Node.

    Los plugins que exponen herramientas de agente alojadas en Nodes pueden establecer `agentTool.defaultPlatforms` para comandos no peligrosos que deban incluirse de forma predeterminada en la lista de permitidos. Omítalo cuando los operadores deban habilitarlos expresamente con `gateway.nodes.commands.allow`. Los comandos peligrosos del host de Node deben registrar una política de invocación de Nodes con `api.registerNodeInvokePolicy(...)`; la política se ejecuta en el Gateway después de comprobar la lista de comandos permitidos y antes de reenviar el comando al Node, por lo que las llamadas directas a `node.invoke`, las herramientas de plugins alojadas en Nodes y las herramientas de plugins de nivel superior comparten la misma ruta de aplicación.

    <Warning>
    El campo opcional `scopes` solicita ámbitos de operador del Gateway para la invocación. OpenClaw solo lo admite para plugins incluidos y para instalaciones de plugins oficiales de confianza; las solicitudes de otros plugins no elevan los privilegios de la llamada. Úselo únicamente cuando un plugin de confianza deba invocar un comando de Node con un ámbito del Gateway más estricto, como `operator.admin`.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    Vincula el estado de Task Flow y Task Run a una clave de sesión existente de OpenClaw o a un contexto de herramienta de confianza.

    - `api.runtime.tasks.managedFlows` permite realizar mutaciones: crear, avanzar y cancelar Task Flows.
    - `api.runtime.tasks.flows` y `api.runtime.tasks.runs` son vistas DTO de solo lectura para enumeraciones y consultas de estado; ambas exponen `bindSession(...)` / `fromToolContext(...)`, además de `get`, `list`, `findLatest` y `resolve`.

    Task Flow realiza el seguimiento del estado persistente de los flujos de trabajo de varios pasos. No es un programador:
    use Cron o `api.session.workflow.scheduleSessionTurn(...)` para activaciones
    futuras y, a continuación, use `managedFlows` desde el turno programado cuando ese trabajo
    necesite el estado del flujo, tareas secundarias, esperas o cancelación.

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "Review new pull requests",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "Review PR #123",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    Use `bindSession({ sessionKey, requesterOrigin })` cuando ya disponga de una clave de sesión de confianza de OpenClaw procedente de su propia capa de vinculación. No realice la vinculación a partir de entradas de usuario sin procesar.

  </Accordion>
  <Accordion title="api.runtime.tts">
    Síntesis de texto a voz.

    ```typescript
    // TTS estándar
    const clip = await api.runtime.tts.textToSpeech({
      text: "Hola desde OpenClaw",
      cfg: api.config,
    });

    // TTS optimizado para telefonía
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "Hola desde OpenClaw",
      cfg: api.config,
    });

    // Enumerar las voces disponibles
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    Utiliza la configuración principal `tts` y la selección de proveedor. Devuelve un búfer de audio PCM y la frecuencia de muestreo. `textToSpeechStream` también está disponible para la síntesis en streaming.

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    Análisis de imágenes, audio y vídeo.

    ```typescript
    // Describir una imagen
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // Transcribir audio
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // opcional, para cuando no se pueda inferir el MIME
    });

    // Describir un vídeo
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // Análisis genérico de archivos
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // Extracción estructurada de imágenes mediante un proveedor/modelo específico.
    // Incluya al menos una imagen; las entradas de texto son contexto complementario.
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "Dé prioridad al total impreso sobre las notas manuscritas." },
      ],
      instructions: "Extraiga el proveedor, el total y las etiquetas de búsqueda.",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    Devuelve `{ text: undefined }` cuando no se genera ninguna salida (por ejemplo, si se omite la entrada).

    `describeImageFileWithModel(...)` describe una imagen ya conocida mediante un proveedor/modelo específico, omitiendo la resolución predeterminada del modelo activo que utiliza `describeImageFile(...)`.

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    Generación de imágenes.

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "Un robot pintando una puesta de sol",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    Generación de vídeo, con la misma estructura que la generación de imágenes.

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "Una toma de dron sobrevolando una costa al amanecer",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    Generación de música, con la misma estructura que la generación de imágenes.

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "Una pista lo-fi animada para una sesión de programación",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    Búsqueda web.

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "SDK de plugins de OpenClaw", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    Utilidades multimedia de bajo nivel.

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "imagen"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    Instantánea de la configuración actual del entorno de ejecución y escrituras transaccionales de configuración. Dé prioridad
    a la configuración que ya se haya pasado a la ruta de llamada activa; use
    `current()` solo cuando el controlador necesite directamente la instantánea del proceso.

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` y `replaceConfigFile(...)` devuelven un valor `followUp`,
    por ejemplo `{ mode: "restart", requiresRestart: true, reason }`,
    que registra la intención del escritor sin quitarle al Gateway el control del reinicio.

  </Accordion>
  <Accordion title="api.runtime.system">
    Utilidades a nivel de sistema.

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // Alias de compatibilidad obsoleto.
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` ejecuta inmediatamente un único ciclo de Heartbeat, omitiendo el temporizador normal de agrupación. Pase `{ heartbeat: { target: "last" } }` para forzar la entrega al último canal activo en lugar de la supresión predeterminada `target: "none"`.

    `runCommandWithTimeout(...)` devuelve los valores capturados de `stdout` y `stderr`, recuentos opcionales
    de truncamiento, `code`, `signal`, `killed`, `termination` y
    `noOutputTimedOut`. Los resultados de tiempo de espera y de tiempo de espera sin salida informan de `code: 124`
    cuando el proceso secundario no proporciona un código de salida distinto de cero. Las salidas por señal
    que no se deban a un tiempo de espera aún pueden devolver `code: null`, por lo que deben usarse `termination` y
    `noOutputTimedOut` para distinguir los motivos del tiempo de espera.

  </Accordion>
  <Accordion title="api.runtime.events">
    Suscripciones a eventos.

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    Registro.

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    Resolución de autenticación de modelos y proveedores.

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // Request-ready auth, including provider runtime exchanges (e.g. OAuth refresh)
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    Resolución del directorio de estado y almacenamiento con claves respaldado por SQLite.

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binary or text payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    Los almacenes con claves sobreviven a los reinicios y están aislados mediante el id del plugin vinculado al entorno de ejecución. Utilice `registerIfAbsent(...)` para las reclamaciones atómicas de deduplicación: devuelve `true` cuando la clave no existía o había caducado y se registró, o `false` cuando ya existe un valor vigente, sin sobrescribir su valor, hora de creación ni TTL. Utilice `deleteIf(...)` cuando la limpieza deba eliminar únicamente el valor observado anteriormente; su predicado síncrono y la eliminación se ejecutan en una sola transacción de SQLite. Límites: `maxEntries` por espacio de nombres, 50,000 filas vigentes por plugin, valores JSON inferiores a 64KB y caducidad TTL opcional. De forma predeterminada, una escritura que alcance cualquiera de los límites de filas descarta las filas vigentes más antiguas del espacio de nombres en el que se escribe; los espacios de nombres relacionados no se desalojan por esa escritura, y esta sigue fallando si el espacio de nombres no puede liberar suficientes filas. Establezca `overflowPolicy: "reject-new"` para los registros de propiedad duraderos que nunca deban desalojarse: las claves nuevas fallan al alcanzar cualquiera de los límites, mientras que las claves existentes siguen pudiendo actualizarse.

    `openSyncKeyedStore<T>(...)` devuelve la misma estructura de almacén con métodos síncronos (`register`, `registerIfAbsent`, `deleteIf`, `lookup`, `consume` y `clear` devuelven los valores directamente en lugar de promesas) para los llamadores que no pueden esperar.

    `openBlobStore<TMetadata>(...)` almacena cargas binarias acotadas en SQLite compartido sin base64 ni archivos auxiliares. Requiere límites de bytes por entrada y por espacio de nombres, además de límites de filas; copia las matrices de bytes en el límite de la API; y enumera los metadatos sin cargar cada BLOB. `register(...)` es una operación upsert explícita, incluso para claves caducadas. `registerIfAbsent(...)` permite una creación segura frente a colisiones: una clave caducada sigue ocupada hasta que su propietario la reclama con `deleteExpiredKey(key)` o `deleteExpired()`, lo que conserva los metadatos necesarios para eliminar los artefactos relacionados con nombre después de confirmar la transacción de SQLite. Cualquier fila con TTL es transitoria y se excluye de las copias de seguridad y la restauración incluso antes de caducar; omita el TTL para el estado duradero y restaurable. Los límites del host restringen cada BLOB a 100 MiB, cada plugin a 512 MiB de BLOB almacenados físicamente y cada plugin a 50,000 filas almacenadas físicamente, incluidas las filas caducadas pendientes de limpieza por parte del propietario. Utilice `registerIfAbsent(...)` con `overflowPolicy: "reject-new"` cuando las materializaciones externas no deban quedar huérfanas silenciosamente debido a una sustitución o un desalojo.

    `openChannelIngressQueue<TPayload>(...)` abre una cola de entrada persistente limitada al plugin que realiza la llamada, para almacenar en búfer eventos entrantes que requieren procesamiento al menos una vez entre reinicios. Cuando la recuperación de reclamaciones obsoletas utilice `shouldRecover`, proporcione también `shouldRecoverCorrupt` si las cargas reclamadas dañadas deben ponerse en cuarentena: su identidad de reclamación independiente de la carga permite que el plugin conserve la política vigente del propietario y del carril antes de que la cola convierta la fila en una lápida.

    `withLease(...)` serializa el trabajo cooperativo de los plugins entre procesos de OpenClaw. Elija `database: { scope: "shared" }` para un único propietario global o `{ scope: "agent", agentId }` para una propiedad independiente por agente. Reenvíe el `AbortSignal` de la devolución de llamada a cada operación susceptible de fallar. `assertOwned()` es un punto de control en un instante concreto antes de iniciar otro paso importante; el host también verifica la propiedad después de la devolución de llamada. La pérdida del arrendamiento o la cancelación por parte del llamador interrumpe la señal. Las esperas de adquisición y los Heartbeat se realizan fuera de las breves transacciones síncronas de SQLite; los plugins nunca reciben rutas ni identificadores de bases de datos. Se trata de cancelación cooperativa, no de un token de delimitación ni de autorización para escrituras externas sin delimitación.

    `openChannelIngressDrain(...)` abre sobre esa cola el proceso de trabajo principal independiente del canal (o crea una cola si no se proporciona ninguna). El vaciado se encarga de la recuperación de reclamaciones obsoletas, la serialización de reclamaciones por carril, la finalización al adoptar o al regresar el envío, la disposición de reintentos o mensajes fallidos, la sustitución opcional previa a la adopción y el tiempo de espera por bloqueo entre reclamación y adopción. Conecte la propiedad de la reclamación con la generación de respuestas mediante `turnAdoptionLifecycle` (a través de `bindIngressLifecycleToReplyOptions` desde `plugin-sdk/channel-outbound`). Los plugins de canal conservan la puesta en cola del lado de aceptación, la derivación de carriles, la clasificación de elementos no reintentables y cualquier política de autorización de sustitución.

    <Warning>
    `openBlobStore`, `openKeyedStore`, `openSyncKeyedStore`, `withLease`, `openChannelIngressQueue` y `openChannelIngressDrain` solo están disponibles para los plugins incluidos y las instalaciones de plugins oficiales de confianza en esta versión.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    Ayudantes del entorno de ejecución específicos del canal (disponibles cuando se carga un plugin de canal). Agrupados por área:

    | Grupo | Finalidad |
    | --- | --- |
    | `text` | Fragmentación (`chunkText`, `chunkMarkdownText`, `resolveChunkMode`), detección de comandos de control y conversión de tablas Markdown. |
    | `reply` | Envío de respuestas en bloques almacenados en búfer, formato de envoltorios y resolución de la configuración efectiva de mensajes y retrasos humanos. |
    | `routing` | `buildAgentSessionKey`, `resolveAgentRoute`. |
    | `pairing` | `buildPairingReply`, lecturas y eliminaciones de listas de permitidos, operaciones upsert de solicitudes de vinculación y entradas de aprobación derivadas de solicitudes. |
    | `media` | Descarga y almacenamiento de contenido multimedia remoto (consulte más adelante). |
    | `activity` | Registrar y leer la última actividad del canal. |
    | `session` | Metadatos de sesión procedentes de eventos entrantes y actualizaciones de la última ruta. |
    | `mentions` | Ayudantes de políticas de menciones (consulte más adelante). |
    | `reactions` | Identificadores de reacciones de confirmación para indicadores de procesamiento en curso. |
    | `groups` | Resolución de políticas de grupo y del requisito de mención. |
    | `debounce` | Antirrebote de mensajes entrantes. |
    | `commands` | Autorización de comandos y control de acceso a comandos de texto. |
    | `outbound` | Cargar el adaptador saliente de un canal. |
    | `inbound` | Crear el contexto de eventos entrantes y ejecutar el núcleo compartido de eventos entrantes y respuestas. |
    | `threadBindings` | Ajustar el tiempo de espera por inactividad y la antigüedad máxima de los hilos de sesión vinculados. |
    | `runtimeContexts` | Registrar, leer y observar el contexto local del proceso por canal, cuenta y capacidad. |

    `api.runtime.channel.media` es la superficie recomendada para descargar y almacenar contenido multimedia de canales:

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    Utilice `saveRemoteMedia(...)` cuando una URL remota deba convertirse en contenido multimedia de OpenClaw. Utilice `saveResponseMedia(...)` cuando el plugin ya haya obtenido un `Response` con autenticación, redirecciones o gestión de listas de permitidos propias del plugin. Utilice `readRemoteMediaBuffer(...)` únicamente cuando el plugin necesite los bytes sin procesar para inspeccionarlos, transformarlos, descifrarlos o volver a cargarlos. `fetchRemoteMedia(...)` sigue siendo un alias de compatibilidad obsoleto de `readRemoteMediaBuffer(...)`.

    `api.runtime.channel.mentions` es la superficie compartida de políticas de menciones entrantes para los plugins de canal incluidos que utilizan inyección del entorno de ejecución:

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    Ayudantes de menciones disponibles:

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    Utilice la ruta normalizada `{ facts, policy }` para las decisiones sobre menciones.

    Varios campos de `reply`, `session` y `inbound` incluyen notas `@deprecated` por campo que apuntan al núcleo actual del turno del canal o a los adaptadores de salida del canal; consulte el JSDoc en línea del ayudante específico antes de crear código nuevo basado en él.

  </Accordion>
</AccordionGroup>

## Almacenamiento de referencias del entorno de ejecución

Utilice `createPluginRuntimeStore` para almacenar la referencia del entorno de ejecución y usarla fuera de la devolución de llamada `register`:

<Steps>
  <Step title="Crear el almacén">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="Conectarlo al punto de entrada">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="Acceder desde otros archivos">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // throws if not initialized
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // returns null if not initialized
    }
    ```

  </Step>
</Steps>

<Note>
Se recomienda `pluginId` para la identidad del almacén del entorno de ejecución. La forma de nivel inferior `key` está destinada a casos poco habituales en los que un plugin necesita intencionadamente más de una ranura del entorno de ejecución.
</Note>

## Otros campos de nivel superior de `api`

Además de `api.runtime`, el objeto de la API también proporciona:

<ParamField path="api.id" type="string">
  Id. del Plugin.
</ParamField>
<ParamField path="api.name" type="string">
  Nombre para mostrar del Plugin.
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  Instantánea de la configuración actual (instantánea activa del entorno de ejecución en memoria cuando esté disponible).
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  Configuración específica del Plugin procedente de `plugins.entries.<id>.config`.
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  Registrador con ámbito (`debug`, `info`, `warn`, `error`).
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  Modo de carga actual: `"full"` (activación en vivo), `"discovery"` / `"tool-discovery"` (detección de capacidades de solo lectura), `"setup-only"` (entrada de configuración ligera), `"setup-runtime"` (flujo de configuración que también necesita la entrada del canal del entorno de ejecución) o `"cli-metadata"` (recopilación de metadatos de comandos de la CLI).
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  Resuelve una ruta relativa a la raíz del Plugin.
</ParamField>

## Contenido relacionado

- [Funcionamiento interno del Plugin](/es/plugins/architecture) — modelo de capacidades y registro
- [Puntos de entrada del SDK](/es/plugins/sdk-entrypoints) — opciones de `definePluginEntry`
- [Descripción general del SDK](/es/plugins/sdk-overview) — referencia de subrutas
