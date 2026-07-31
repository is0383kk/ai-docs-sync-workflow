---
read_when:
    - Es necesario saber desde qué subruta del SDK importar.
    - Se necesita una referencia de todos los métodos de registro de OpenClawPluginApi
    - Está buscando una exportación específica del SDK
sidebarTitle: Plugin SDK overview
summary: Mapa de importaciones, referencia de la API de registro y arquitectura del SDK
title: Descripción general del SDK de Plugins
x-i18n:
    generated_at: "2026-07-26T04:47:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f490aa8670c57cfc1a635fb1f5d9950fa1cabdb3d45abbc2295da796edcd52e
    source_path: plugins/sdk-overview.md
    workflow: 16
---

El SDK de plugins es el contrato tipado entre los plugins y el núcleo. Esta página es la
referencia de **qué importar** y **qué se puede registrar**.

<Note>
  Esta página está destinada a autores de plugins que usan `openclaw/plugin-sdk/*` dentro de
  OpenClaw. Para aplicaciones externas, scripts, paneles, trabajos de CI y extensiones de IDE
  que quieran ejecutar agentes mediante el Gateway, use en su lugar
  [Integraciones del Gateway para aplicaciones externas](/es/gateway/external-apps).
</Note>

<Tip>
¿Busca una guía práctica? Comience con [Creación de plugins](/es/plugins/building-plugins). Use [Plugins de canal](/es/plugins/sdk-channel-plugins) para canales, [Plugins de proveedor](/es/plugins/sdk-provider-plugins) para proveedores de modelos, [Plugins de backend de CLI](/es/plugins/cli-backend-plugins) para backends locales de CLI de IA, [Plugins de infraestructura de agentes](/es/plugins/sdk-agent-harness) para ejecutores nativos de agentes y [Hooks de plugins](/es/plugins/hooks) para hooks de herramientas o del ciclo de vida.
</Tip>

## Convención de importación

Importe siempre desde una subruta específica:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

Cada subruta es un módulo pequeño y autónomo. Esto agiliza el inicio y
evita problemas de dependencias circulares. Para los ayudantes de entrada y compilación
específicos de canales, prefiera `openclaw/plugin-sdk/channel-core`; reserve `openclaw/plugin-sdk/core` para
la superficie general más amplia y los ayudantes compartidos, como
`buildChannelConfigSchema`.

Para la configuración de canales, publique el esquema JSON propiedad del canal mediante
`openclaw.plugin.json#channelConfigs`. La subruta `plugin-sdk/channel-config-schema`
está destinada a primitivas de esquema compartidas y al constructor genérico. Los
plugins incluidos de OpenClaw usan `plugin-sdk/bundled-channel-config-schema` para los esquemas
conservados de los canales incluidos. Esa subruta de esquemas incluidos no es un patrón para nuevos
plugins.

<Warning>
  No importe interfaces auxiliares de conveniencia con marcas de proveedores o canales (por ejemplo,
  `openclaw/plugin-sdk/slack`, `.../discord`, `.../signal`, `.../whatsapp`).
  Los plugins incluidos componen subrutas genéricas del SDK dentro de sus propios barrels `api.ts` /
  `runtime-api.ts`; los consumidores del núcleo deben usar esos barrels locales del plugin
  o añadir un contrato genérico y limitado del SDK cuando una necesidad sea realmente
  común a varios canales.

Un pequeño conjunto de interfaces auxiliares para plugins incluidos sigue apareciendo en el mapa de exportaciones
generado cuando tiene un uso registrado por parte de sus propietarios. Solo existe para el mantenimiento de
plugins incluidos y no se recomienda como ruta de importación para nuevos plugins de
terceros.

`openclaw/plugin-sdk/discord` y `openclaw/plugin-sdk/telegram-account` también
se conservan como fachadas de compatibilidad obsoletas para el uso registrado por parte de sus propietarios. No
copie esas rutas de importación en plugins nuevos; use en su lugar ayudantes del entorno de ejecución
inyectados y subrutas genéricas del SDK de canales.
</Warning>

## Referencia de subrutas

El SDK de plugins se expone como un conjunto de subrutas específicas agrupadas por área (entrada del
plugin, canal, proveedor, autenticación, entorno de ejecución, capacidad, memoria y ayudantes
reservados para plugins incluidos). Para consultar el catálogo completo, agrupado y con enlaces, consulte
[Subrutas del SDK de plugins](/es/plugins/sdk-subpaths).

El inventario de puntos de entrada del compilador se encuentra en
`scripts/lib/plugin-sdk-entrypoints.json`; las exportaciones públicas tipadas excluyen las
subrutas internas enumeradas en
`scripts/lib/plugin-sdk-private-local-only-subpaths.json`. Las entradas de producción
de esa lista conservan exportaciones de JavaScript únicamente del entorno de ejecución del host para plugins oficiales
publicados por separado, mientras que las entradas exclusivas para pruebas permanecen sin exportar. Ejecute
`pnpm plugin-sdk:surface` para auditar el recuento de exportaciones públicas. Las subrutas públicas
obsoletas con suficiente antigüedad y que no utiliza el código de producción de extensiones incluidas
se registran en `scripts/lib/plugin-sdk-deprecated-public-subpaths.json`; los barrels amplios
de reexportaciones obsoletas se registran en
`scripts/lib/plugin-sdk-deprecated-barrel-subpaths.json`.

## API de registro

La función de retorno `register(api)` recibe un objeto `OpenClawPluginApi` con estos
métodos:

Los plugins que proporcionan una superficie externa de chat de equipo para una sesión pueden registrar
el único proveedor para todo el proceso exportado por
`openclaw/plugin-sdk/session-discussion`. Su método `info({ sessionKey })`
indica si una conversación no está disponible, está lista para abrirse o ya está abierta;
`open({ sessionKey })` crea o resuelve la conversación y devuelve sus URL de inserción
y externas. Registrar otro proveedor sustituye al proveedor actual.

### Registro de capacidades

| Método                                           | Qué registra                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerProvider(...)`                      | Inferencia de texto (LLM)                                                                                                                      |
| `api.registerWorkerProvider(...)`                | Concesiones de ciclo de vida para trabajadores en la nube                                                                                                             |
| `api.registerModelCatalogProvider(...)`          | Filas del catálogo de modelos para la generación de texto y contenido multimedia                                                                                          |
| `api.registerAgentHarness(...)`                  | Ejecutor nativo de agentes [experimental](/es/plugins/sdk-agent-harness) (Codex, Copilot)                                                         |
| `api.registerCliBackend(...)`                    | Backend local de inferencia mediante CLI                                                                                                               |
| `api.registerChannel(...)`                       | Canal de mensajería                                                                                                                         |
| `api.registerEmbeddingProvider(...)`             | Proveedor reutilizable de incrustaciones vectoriales                                                                                                        |
| `api.registerSpeechProvider(...)`                | Síntesis de texto a voz / STT                                                                                                            |
| `api.registerRealtimeTranscriptionProvider(...)` | Transcripción en tiempo real por streaming                                                                                                          |
| `api.registerRealtimeVoiceProvider(...)`         | Sesiones de voz bidireccionales en tiempo real                                                                                                            |
| `api.registerMediaUnderstandingProvider(...)`    | Análisis de imágenes, audio y vídeo                                                                                                                |
| `api.registerTranscriptSourceProvider(...)`      | Fuente de transcripciones de reuniones en directo o importadas; los plugins de reuniones pueden usar `createMeetingTranscriptSourceProvider` de `plugin-sdk/transcripts` |
| `api.registerImageGenerationProvider(...)`       | Generación de imágenes                                                                                                                          |
| `api.registerMusicGenerationProvider(...)`       | Generación de música                                                                                                                          |
| `api.registerVideoGenerationProvider(...)`       | Generación de vídeo                                                                                                                          |
| `api.registerWebFetchProvider(...)`              | Proveedor de obtención / extracción web                                                                                                               |
| `api.registerWebSearchProvider(...)`             | Búsqueda web                                                                                                                                |
| `api.registerCompactionProvider(...)`            | Backend conectable de compactación de transcripciones                                                                                                   |

Los proveedores de trabajadores también deben declarar su identificador en `contracts.workerProviders`.
El núcleo conserva la intención duradera antes de `provision(profile, operationId)`. Los proveedores validan la configuración antes de la asignación externa y lanzan `WorkerProviderError` para el rechazo permanente de un perfil. `provision` debe adoptar la misma concesión cuando se repita el identificador de la operación.
El núcleo conserva la configuración validada del perfil junto con la concesión y proporciona esa instantánea a `destroy({ leaseId, profile })`, que debe ser idempotente, y a `inspect({ leaseId, profile })`, que devuelve `active`, `destroyed` o `unknown`. Esto permite a los proveedores enrutar llamadas del ciclo de vida después de reiniciar el Gateway o eliminar un perfil con nombre. Los endpoints SSH usan un `SecretRef` para `keyRef`, nunca material de claves insertado directamente, e incluyen un `hostKey` procedente de la salida de aprovisionamiento de confianza exactamente como `algorithm base64`, sin nombre de host ni comentario. El núcleo fija `hostKey` y nunca confía en una clave de la primera conexión. Un proveedor que emita dinámicamente un `keyRef` puede implementar `resolveSshIdentity({ leaseId, profile, keyRef })`; cuando está presente, ese mecanismo de resolución es la fuente de autoridad, mientras que los proveedores que no lo tienen usan el mecanismo genérico configurado de resolución de secretos.
Los proveedores con concesiones renovables también pueden implementar `renew(leaseId)`.
`inspect` debe lanzar una excepción ante fallos transitorios o indeterminados; devuelva `unknown` solo cuando la ausencia sea concluyente. El núcleo marca un registro local activo como huérfano o considera la ausencia como la finalización de la eliminación después de una solicitud de destrucción persistida.

Los proveedores de incrustaciones registrados con `api.registerEmbeddingProvider(...)` también deben
figurar en `contracts.embeddingProviders` en el manifiesto del plugin. Esta
es la superficie genérica de incrustaciones para generar vectores reutilizables. La búsqueda en
memoria puede consumir esta superficie genérica de proveedores. La interfaz anterior
`api.registerMemoryEmbeddingProvider(...)` y
`contracts.memoryEmbeddingProviders` ofrece compatibilidad obsoleta mientras
se migran los proveedores existentes específicos de memoria.

Los proveedores específicos de memoria que aún exponen un `batchEmbed(...)` del entorno de ejecución permanecen en
el contrato existente de procesamiento por lotes por archivo, salvo que su entorno de ejecución establezca explícitamente
`sourceWideBatchEmbed: true`. Esta activación permite al host de memoria enviar fragmentos de
varios archivos de memoria modificados y fuentes habilitadas en una llamada a `batchEmbed(...)`
hasta los límites de lote del host. Los adaptadores por lotes que cargan archivos de solicitud JSONL deben
dividir los trabajos del proveedor antes de alcanzar tanto el límite del tamaño de carga como el límite de la cantidad
de solicitudes. El proveedor debe devolver una incrustación por cada fragmento de entrada en el mismo orden que
`batch.chunks`; omita la marca cuando el proveedor espere lotes locales de archivos o
no pueda conservar el orden de entrada en un trabajo más amplio para toda la fuente.

### Herramientas y comandos

Use [`defineToolPlugin`](/es/plugins/tool-plugins) para plugins sencillos que solo aporten herramientas
con nombres de herramientas fijos. Use `api.registerTool(...)` directamente para plugins mixtos
o para el registro de herramientas totalmente dinámicas.

| Método                                 | Qué registra                                                                                                                        |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerTool(tool, opts?)`        | Herramienta del agente (obligatoria o `{ optional: true }`)                                                                                            |
| `api.registerCommand(def)`             | Comando personalizado (omite el LLM)                                                                                                        |
| `api.registerNodeHostCommand(command)` | Comando gestionado por `openclaw node run`; los metadatos opcionales `agentTool` pueden exponerlo como una herramienta visible para el agente mientras el nodo está conectado |

Los comandos de plugins pueden establecer `agentPromptGuidance` cuando el agente necesita una indicación breve
de enrutamiento propiedad del comando. Mantenga ese texto centrado en el propio comando; no añada
políticas específicas del proveedor o del plugin a los constructores de prompts del núcleo.

Las entradas de orientación pueden ser cadenas heredadas, que se aplican a todas las superficies de prompts, o
entradas estructuradas:

```ts
agentPromptGuidance: [
  "Indicación global del comando.",
  { text: "Mostrar esto solo en el prompt principal de OpenClaw.", surfaces: ["openclaw_main"] },
];
```

El contenido estructurado `surfaces` puede incluir `openclaw_main`, `codex_app_server`,
`cli_backend`, `acp_backend` o `subagent`. `pi_main` sigue siendo un alias obsoleto
de `openclaw_main`. Omita `surfaces` para proporcionar orientación intencionada en todas las superficies. No
pase un array `surfaces` vacío; se rechaza para evitar que una pérdida accidental de alcance
convierta el contenido en texto de prompt global.

Las instrucciones nativas para desarrolladores del servidor de aplicaciones de Codex son más estrictas que las de otras
superficies de prompts: solo la orientación cuyo alcance se haya definido explícitamente como `codex_app_server` se promueve
a ese nivel de mayor prioridad. La orientación heredada en forma de cadena y la orientación estructurada
sin alcance definido siguen disponibles para las superficies de prompts que no son de Codex por compatibilidad.

Los comandos del host Node se ejecutan en el host Node conectado, no dentro del proceso
del Gateway. Si `agentTool` está presente, el Node publica un descriptor después de
conectarse correctamente al Gateway; el Gateway lo expone a las ejecuciones del agente únicamente mientras ese
Node esté conectado y solo si el `command` del descriptor pertenece a la superficie de comandos
aprobada del Node. Establezca `agentTool.defaultPlatforms` para incluir un
comando no peligroso en la lista de permitidos predeterminada de comandos del Node; de lo contrario, exija
un `gateway.nodes.commands.allow` explícito o una política de invocación del Node. `agentTool.name`
debe ser seguro para el proveedor: debe comenzar por una letra, contener únicamente letras, dígitos,
guiones bajos o guiones, y no superar los 64 caracteres. Las herramientas del Node respaldadas por MCP
pueden establecer metadatos `agentTool.mcp` para que las superficies de catálogo y búsqueda de herramientas muestren
la identidad del servidor o la herramienta MCP remotos, pero la ejecución sigue realizándose mediante el
comando del Node anunciado.

### Infraestructura

| Método                                          | Qué registra                                                            |
| ----------------------------------------------- | ------------------------------------------------------------------------ |
| `api.registerHook(events, handler, opts?)`      | Hook de evento                                                           |
| `api.registerHttpRoute(params)`                 | Endpoint HTTP del Gateway                                                |
| `api.registerGatewayMethod(name, handler)`      | Método RPC del Gateway                                                   |
| `api.registerGatewayDiscoveryService(service)`  | Anunciante de detección local del Gateway                                |
| `api.registerCli(registrar, opts?)`             | Subcomando de la CLI                                                     |
| `api.registerNodeCliFeature(registrar, opts?)`  | CLI de funciones del Node bajo `openclaw nodes`                         |
| `api.registerService(service)`                  | Servicio en segundo plano                                                |
| `api.registerInteractiveHandler(registration)`  | Controlador interactivo                                                  |
| `api.registerAgentToolResultMiddleware(...)`    | Middleware de resultados de herramientas en tiempo de ejecución          |
| `api.registerMemoryPromptSupplement(builder)`   | Sección de prompt aditiva adyacente a la memoria                         |
| `api.registerMemoryPromptPreparation(prepare)`  | Preparación asíncrona de una sección de prompt adyacente a la memoria    |
| `api.registerMemoryCorpusSupplement(adapter)`   | Corpus aditivo de búsqueda y lectura de memoria                          |
| `api.registerHostedMediaResolver(resolver)`     | Solucionador de URL de contenido multimedia alojado de estilo navegador |
| `api.registerMcpServerConnectionResolver(...)`  | Transporte MCP por solicitante (`url`/`headers`) para un nombre de servidor estático |
| `api.registerTextTransforms(transforms)`        | Reescrituras de texto de compatibilidad de prompts y mensajes propiedad del Plugin |
| `api.registerConfigMigration(migrate)`          | Migración ligera de configuración ejecutada antes de cargar el entorno de ejecución del Plugin |
| `api.registerMigrationProvider(provider)`       | Importador de `openclaw migrate`                                         |
| `api.registerAutoEnableProbe(probe)`            | Comprobación de configuración que puede habilitar automáticamente este Plugin |
| `api.registerReload(registration)`              | Política de prefijos de configuración reinicio/en caliente/sin operación para gestionar recargas |
| `api.registerNodeHostCommand(command)`          | Controlador de comandos expuesto a Nodes emparejados                     |
| `api.registerNodeInvokePolicy(policy)`          | Política de lista de permitidos/aprobación para comandos invocados por Nodes |
| `api.registerSecurityAuditCollector(collector)` | Recopilador de hallazgos para `openclaw security audit`                          |

#### Trabajo posterior a la confirmación del Webhook

Las rutas de Webhook que confirman una solicitud antes de que finalice el procesamiento deben trasladar
ese trabajo desvinculado a su propia raíz de admisión supervisada:

```typescript
import { runDetachedWebhookWork } from "openclaw/plugin-sdk/webhook-request-guards";

void runDetachedWebhookWork(() => processWebhookEvent(event)).catch((error) => {
  runtime.error?.(`falló el despacho del webhook: ${String(error)}`);
});
```

Llame a `runDetachedWebhookWork(...)` de forma síncrona mientras la solicitud HTTP siga
admitida. El asistente reserva inmediatamente una raíz independiente y, después, inicia el
callback en la siguiente microtarea para que el controlador de solicitudes pueda escribir primero su
confirmación. La promesa devuelta adopta el resultado del callback; quienes realizan la llamada
siguen siendo responsables de gestionar los rechazos. Esto mantiene aceptado el trabajo de la cola posterior a la confirmación y hace
que los drenajes por reinicio o suspensión esperen a que finalice. Los controladores que esperan a que termine todo el procesamiento
antes de devolver el resultado no necesitan este asistente.

#### Conexiones MCP con alcance por solicitante

Mantenga estática la **identidad** del servidor MCP (nombre y filtro de herramientas) en `mcp.servers`, en
el campo de manifiesto `mcpServers` de un Plugin nativo o en el manifiesto de un paquete. Opcionalmente, registre un solucionador de conexiones para que cada
solicitante de mensajes de confianza obtenga su propio transporte:

```ts
api.registerMcpServerConnectionResolver({
  serverName: "user-email",
  resolve: async (ctx) => {
    // ctx.requesterSenderId es una identidad de confianza del host; nunca invente aquí la identidad del remitente.
    const token = await lookupUserToken(ctx.requesterSenderId);
    if (!token) {
      return null; // omitir este servidor para la ejecución actual
    }
    return {
      url: "https://mcp.example.com/email",
      headers: { Authorization: `Bearer ${token}` },
    };
  },
});
```

Notas del contrato:

- El contexto del solucionador solo contiene identidades de confianza del host (`requesterSenderId`,
  con `agentAccountId` / `messageChannel` opcionales). En el futuro se podrán añadir de forma aditiva
  campos de confianza (por ejemplo, el contexto de usuario de Cron o del subagente).
- Un Plugin es propietario de un nombre de servidor: si otro
  Plugin registra un `registerMcpServerConnectionResolver` duplicado para el mismo `serverName`,
  se rechaza con un diagnóstico de error (prevalece el primer registro), por lo que
  la propiedad de la conexión nunca depende del orden de carga de los plugins.
- Los nombres de las herramientas se derivan del conjunto completo de servidores declarados, de modo que una resolución parcial
  nunca cambia los nombres seguros de los servidores entre solicitantes o turnos. El núcleo no
  verifica que los endpoints de distintos solicitantes sirvan esquemas de herramientas idénticos; un
  solucionador debe dirigir a todos los solicitantes al mismo servicio lógico, o los esquemas
  de las herramientas (y la estabilidad de la caché de prompts) divergirán según el solicitante.
- Las ejecuciones sin un `requesterSenderId` de confianza (Cron, subagente, Heartbeat, Gateway
  público) nunca materializan servidores con alcance por solicitante. No existe ninguna
  conexión alternativa compartida.
- `resolve` tiene un límite de 10 segundos por servidor; si se agota el tiempo o se produce una excepción, se omite ese
  servidor en la ejecución sin provocar un fallo del MCP estático.
- Las conexiones resueltas se vuelven a validar como máximo cada 5 minutos por solicitante:
  la rotación reconstruye el transporte con credenciales nuevas y un resultado `null`
  lo revoca (el entorno de ejecución almacenado en caché se elimina incluso en mitad de la sesión). Por tanto, una credencial
  revocada o rotada puede seguir utilizándose durante un máximo de 5 minutos.
- Los `headers` resueltos nunca se registran ni persisten; el núcleo conserva únicamente un resumen
  efímero indexado en memoria (HMAC local del proceso) para detectar la rotación de credenciales y
  registra los valores de las credenciales resueltas de encabezados/URL en el registro de ocultación
  de capturas de depuración y registros.
- Los servidores con alcance por solicitante no generan vistas de aplicaciones MCP: una vista perdura más que la
  ejecución autenticada del solicitante y el límite de vistas del Gateway no dispone de identidad del solicitante,
  por lo que las vistas previas de las aplicaciones permanecen cerradas de forma segura para estos servidores. Los resultados de las herramientas
  no se ven afectados.
- Los servidores estáticos sin solucionador conservan el ciclo de vida existente con alcance de sesión.
- **Regla de entrega del arnés:** los servidores con alcance por solicitante nunca se incorporan a la
  configuración del cliente MCP nativo del arnés (`mcp_servers` del hilo de Codex, `-c mcp_servers=…` de la CLI ni
  ninguna otra proyección MCP compartida por la sesión). En su lugar, los arneses los entregan como herramientas
  con alcance de ejecución:
  - Ejecutor integrado: entorno de ejecución MCP de la sesión + herramientas del paquete (estáticas + con alcance).
  - Servidor de aplicaciones de Codex: herramientas dinámicas mediante
    `materializeRequesterScopedMcpToolsForHarnessRun` (solo con alcance; los servidores
    estáticos permanecen en el cliente MCP nativo de Codex).
- Las **especificaciones** de las herramientas con alcance permanecen estables durante la sesión después de la primera resolución correcta
  en esa sesión, de modo que los arneses con hilos compartidos (Codex) no roten los hilos cuando
  cambien los remitentes. Antes de que algún solicitante se resuelva, no se anuncia ninguna especificación con alcance.
- Los solicitantes no autenticados de un arnés con hilos compartidos siguen viendo las herramientas
  con alcance anunciadas; al llamar a una, se devuelve un error claro de herramienta no conectada para ese
  solicitante. OpenClaw nunca recurre a las credenciales de otro solicitante.

Los generadores de complementos de prompts de memoria reciben contexto opcional `agentId`,
`agentSessionKey` y `sandboxed`. Las llamadas `search`
y `get` de complementos del corpus de memoria reciben contexto opcional `agentId` y `sandboxed`. Los plugins con
almacenamiento propiedad del agente deben resolver ese almacenamiento en cada llamada en lugar de
capturar una única ruta global durante el registro. Si se necesita un id de agente, pero
falta en una operación multiagente, cierre de forma segura en lugar de elegir un
agente arbitrario.

Use `registerMemoryPromptPreparation(...)` cuando el texto del prompt dependa del estado asíncrono
del Plugin. El callback se ejecuta una vez antes de cada prompt completo del agente y recibe
el mismo contexto de herramientas, agente, sesión y sandbox que los generadores síncronos de prompts de
memoria. Valide la instancia actual propietaria del almacenamiento antes de cargar el estado persistente
y devuelva únicamente las líneas correspondientes a esa ejecución. OpenClaw inmoviliza esas líneas y
entrega el resultado inmutable al ensamblado síncrono del prompt. Mantenga la persistencia,
el reemplazo atómico y la eliminación al retirar al propietario dentro del Plugin propietario; no
sondee ni lea archivos desde un generador de prompts.

Los controladores interactivos de Telegram pueden devolver `{ submitText }` para encaminar el texto por
la ruta normal de entrada al agente de Telegram una vez que el controlador se complete correctamente. OpenClaw conserva
el botón de callback cuando la política de entrada omite el texto o falla el procesamiento, para que
el usuario pueda volver a intentarlo cuando cambie la condición que lo bloquea. Este campo de resultado es
específico de Telegram; los demás canales mantienen sus propios contratos de resultados interactivos.

### Hooks del host para plugins de flujo de trabajo

Los hooks del host son las interfaces del SDK para los plugins que necesitan participar en el ciclo de vida
del host en lugar de limitarse a añadir un proveedor, canal o herramienta. Son
contratos genéricos; el Modo Plan puede utilizarlos, pero también pueden hacerlo los flujos de trabajo de aprobación,
los controles de políticas del espacio de trabajo, los monitores en segundo plano, los asistentes de configuración y los plugins
complementarios de la interfaz de usuario.

| Método                                                                               | Contrato del que es responsable                                                                                                                            |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.session.state.registerSessionExtension(...)`                                    | Estado de sesión propiedad del Plugin, compatible con JSON y proyectado mediante sesiones del Gateway                                                       |
| `api.session.workflow.enqueueNextTurnInjection(...)`                                 | Contexto duradero de ejecución exactamente una vez, inyectado en el siguiente turno del agente para una sesión                                              |
| `api.registerTrustedToolPolicy(...)`                                                 | Política de herramientas de confianza, previa al Plugin y condicionada por el manifiesto, que puede bloquear o reescribir parámetros de herramientas        |
| `api.registerToolMetadata(...)`                                                      | Metadatos de presentación del catálogo de herramientas sin cambiar la implementación de la herramienta                                                     |
| `api.registerCommand(...)`                                                           | Comandos de Plugin con ámbito definido; los resultados de comandos pueden establecer `continueAgent: true` o `suppressReply: true`; los comandos nativos de Discord admiten `descriptionLocalizations` |
| `api.session.controls.registerControlUiDescriptor(...)`                              | Descriptores de contribución a la interfaz de control para superficies de sesión, herramienta, ejecución, configuración o pestaña                          |
| `api.lifecycle.registerRuntimeLifecycle(...)`                                        | Funciones de devolución de llamada de limpieza para recursos de ejecución propiedad del Plugin en rutas de restablecimiento, eliminación o recarga          |
| `api.agent.events.registerAgentEventSubscription(...)`                               | Suscripciones a eventos saneados para el estado y los monitores de flujos de trabajo                                                                         |
| `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`  | Estado temporal del Plugin por ejecución, borrado durante el ciclo de vida terminal de la ejecución                                                         |
| `api.session.workflow.registerSessionSchedulerJob(...)`                              | Metadatos de limpieza para trabajos del planificador propiedad del Plugin; no programa trabajo ni crea registros de tareas                                  |
| `api.session.workflow.sendSessionAttachment(...)`                                    | Entrega de archivos adjuntos mediada por el host, solo para Plugins incluidos, a la ruta activa de la sesión de salida directa                              |
| `api.session.workflow.scheduleSessionTurn(...)` / `unscheduleSessionTurnsByTag(...)` | Turnos de sesión programados respaldados por Cron, solo para Plugins incluidos, más limpieza basada en etiquetas                                            |
| `api.session.controls.registerSessionAction(...)`                                    | Acciones de sesión tipadas que los clientes pueden enviar mediante el Gateway                                                                               |

Un descriptor `surface: "tab"` añade una pestaña a la barra lateral de la interfaz de control. Los descriptores de pestaña de los
Plugins activos se anuncian a los clientes del panel en el saludo del Gateway
(`controlUiTabs`), por lo que la pestaña solo aparece mientras el Plugin está habilitado.
Los Plugins incluidos pueden proporcionar una vista de panel de primera clase para su pestaña; otros
Plugins pueden establecer `path` en una ruta HTTP del Plugin (consulte
`api.registerHttpRoute(...)`) que el panel representa en un marco aislado.
`icon` es una sugerencia de nombre de icono para el panel, `group` selecciona la sección de la barra lateral
(`control` o `agent`), `order` determina el orden entre las pestañas de Plugins y `requiredScopes`
oculta la pestaña para las conexiones que carecen de esos ámbitos de operador:

Para una pestaña externa protegida por el Gateway, registre el descriptor `path` bajo una
ruta HTTP `auth: "gateway"` del mismo Plugin. Tras el arranque autenticado, el navegador obtiene una
concesión HttpOnly de corta duración, limitada a ese Plugin y a la raíz de la ruta, para que el
marco aislado pueda cargarse sin copiar el token de portador del Gateway en su URL
ni en JavaScript. El elemento principal autenticado renueva la concesión mientras la pestaña externa
está activa y antes de montarla tras una navegación o al reanudar el navegador. También
comprueba la concesión desde el mismo entorno aislado opaco antes del montaje, de modo que los
modos de privacidad del navegador que bloquean la cookie fallen de forma segura mostrando un panel no disponible.
La concesión del marco solo acepta `GET` y `HEAD`, y siempre incluye
`operator.read`; `requiredScopes` controla la visibilidad de la pestaña, pero nunca amplía la
concesión de la cookie. Las mutaciones permanecen en superficies principales autenticadas explícitamente por el Gateway o
en superficies de portador. Las pestañas externas requieren HTTPS/Tailscale Serve o un
origen de bucle invertido de confianza para el navegador; HTTP sin cifrar en un host de LAN muestra el
error de contexto seguro en lugar de montar un panel que no puede autenticarse.
El bloqueo total de cookies de terceros también hace que las pestañas protegidas por el Gateway no estén disponibles.
Como ocurre con todas las superficies nativas de Plugins, el marco permanece dentro del límite de confianza
del Plugin instalado; OpenClaw no trata los Plugins instalados como principales de seguridad del navegador
aislados entre sí.
Las concesiones de cookies usan el límite del nombre de host del navegador, no el límite de su puerto. No
aloje conjuntamente servicios que no sean de confianza mutua en el nombre de host del Gateway, ni siquiera en otros
puertos.
Las pestañas respaldadas por autenticación administrada por el Plugin conservan su comportamiento directo de iframe y no
solicitan ni requieren esta concesión del Gateway.

```typescript
api.session.controls.registerControlUiDescriptor({
  surface: "tab",
  id: "logbook",
  label: "Registro",
  description: "Su día como una línea temporal, creada a partir de capturas de pantalla.",
  icon: "sun",
  group: "control",
  requiredScopes: ["operator.write"],
});
```

Use los espacios de nombres agrupados para el código nuevo de Plugins:

- `api.session.state.registerSessionExtension(...)`
- `api.session.workflow.enqueueNextTurnInjection(...)`
- `api.session.workflow.registerSessionSchedulerJob(...)`
- `api.session.workflow.sendSessionAttachment(...)`
- `api.session.workflow.scheduleSessionTurn(...)`
- `api.session.workflow.unscheduleSessionTurnsByTag(...)`
- `api.session.controls.registerSessionAction(...)`
- `api.session.controls.registerControlUiDescriptor(...)`
- `api.agent.events.registerAgentEventSubscription(...)`
- `api.agent.events.emitAgentEvent(...)`
- `api.runContext.setRunContext(...)` / `getRunContext(...)` / `clearRunContext(...)`
- `api.lifecycle.registerRuntimeLifecycle(...)`

Los métodos planos equivalentes siguen disponibles como alias de compatibilidad
obsoletos para los Plugins existentes. No añada código nuevo de Plugins que llame directamente a
`api.registerSessionExtension`, `api.enqueueNextTurnInjection`,
`api.registerControlUiDescriptor`, `api.registerRuntimeLifecycle`,
`api.registerAgentEventSubscription`, `api.emitAgentEvent`,
`api.setRunContext`, `api.getRunContext`, `api.clearRunContext`,
`api.registerSessionSchedulerJob`, `api.registerSessionAction`,
`api.sendSessionAttachment`, `api.scheduleSessionTurn` o
`api.unscheduleSessionTurnsByTag`.

`scheduleSessionTurn(...)` es una utilidad de conveniencia con ámbito de sesión sobre el
planificador Cron del Gateway. Cron controla la temporización y crea el registro de tarea en segundo plano cuando se
ejecuta el turno; el SDK del Plugin solo restringe la sesión de destino, la
nomenclatura propiedad del Plugin y la limpieza. Use `api.runtime.tasks.managedFlows` dentro del turno
programado cuando el trabajo necesite un estado duradero de Task Flow de varios pasos.

Los contratos dividen deliberadamente la autoridad:

- Los Plugins externos pueden controlar extensiones de sesión, descriptores de interfaz, comandos, metadatos de
  herramientas, inyecciones para el siguiente turno y hooks normales.
- Las políticas de herramientas de confianza se ejecutan antes que los hooks `before_tool_call`
  normales y son de confianza para el host. Las políticas incluidas se ejecutan primero; las políticas de Plugins
  instalados requieren habilitación explícita y que sus identificadores locales estén en
  `contracts.trustedToolPolicies`, y se ejecutan después en el orden de carga de los Plugins. Los identificadores de las políticas
  tienen el ámbito del Plugin que las registra.
- La propiedad de comandos reservados es exclusiva de los Plugins incluidos. Los Plugins externos deben usar sus
  propios nombres de comando o alias.
- `allowPromptInjection=false` deshabilita los hooks que modifican el prompt, incluidos
  `agent_turn_prepare`, `before_prompt_build`, `heartbeat_prompt_contribution`
  y `enqueueNextTurnInjection`.

Ejemplos de consumidores que no son Plan:

| Arquetipo de Plugin                  | Hooks utilizados                                                                                                                              |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Flujo de trabajo de aprobación       | Extensión de sesión, continuación de comandos, inyección para el siguiente turno, descriptor de interfaz                                      |
| Puerta de política de presupuesto/espacio de trabajo | Política de herramientas de confianza, metadatos de herramientas, proyección de sesión                                                       |
| Monitor del ciclo de vida en segundo plano | Limpieza del ciclo de vida de ejecución, suscripción a eventos del agente, propiedad/limpieza del planificador de sesiones, contribución al prompt de Heartbeat, descriptor de interfaz |
| Asistente de configuración o incorporación | Extensión de sesión, comandos con ámbito definido, descriptor de la interfaz de control                                                       |

<Note>
  Los espacios de nombres administrativos reservados del núcleo (`config.*`, `exec.approvals.*`, `wizard.*`,
  `update.*`) siempre permanecen como `operator.admin`, incluso si un Plugin intenta asignar un
  ámbito más limitado al método del Gateway. Se recomienda usar prefijos específicos del Plugin para los
  métodos propiedad del Plugin.
</Note>

<Accordion title="Cuándo usar middleware de resultados de herramientas">
  Los Plugins incluidos y los Plugins instalados habilitados explícitamente con contratos
  de manifiesto coincidentes pueden usar `api.registerAgentToolResultMiddleware(...)` cuando
  necesiten reescribir el resultado de una herramienta después de su ejecución y antes de que el entorno de ejecución
  devuelva ese resultado al modelo. Este es el punto de integración de confianza e independiente del entorno de ejecución
  para reductores de salida asíncronos como tokenjuice.

Los Plugins deben declarar `contracts.agentToolResultMiddleware` para cada entorno de ejecución
de destino, por ejemplo, `["openclaw", "codex"]`. Los Plugins instalados sin ese
contrato, o sin habilitación explícita, no pueden registrar este middleware; mantenga
los hooks normales de Plugins de OpenClaw para trabajos que no necesiten ejecutarse entre el resultado de la herramienta
y su entrega al modelo. Se ha eliminado la antigua
ruta de registro de fábricas de extensiones exclusiva del ejecutor integrado.
</Accordion>

### Registro de descubrimiento del Gateway

`api.registerGatewayDiscoveryService(...)` permite que un Plugin anuncie el
Gateway activo en un transporte de descubrimiento local como mDNS/Bonjour. OpenClaw llama al
servicio durante el inicio del Gateway cuando el descubrimiento local está habilitado, le pasa los
puertos actuales del Gateway y los datos orientativos TXT no secretos, y llama al controlador
`stop` devuelto durante el apagado del Gateway.

```typescript
api.registerGatewayDiscoveryService({
  id: "my-discovery",
  async advertise(ctx) {
    const handle = await startMyAdvertiser({
      gatewayPort: ctx.gatewayPort,
      tls: ctx.gatewayTlsEnabled,
      displayName: ctx.machineDisplayName,
    });
    return { stop: () => handle.stop() };
  },
});
```

Los Plugins de descubrimiento del Gateway no deben tratar los valores TXT anunciados como secretos ni como
autenticación. El descubrimiento es una indicación de enrutamiento; la autenticación del Gateway y la fijación de TLS siguen
controlando la confianza.

### Metadatos de registro de la CLI

`api.registerCli(registrar, opts?)` acepta dos tipos de metadatos de comandos:

- `commands`: nombres explícitos de comandos propiedad del registrador
- `descriptors`: descriptores de comandos durante el análisis, usados para la ayuda de la CLI,
  el enrutamiento y el registro diferido de la CLI del Plugin
- `parentPath`: ruta opcional del comando principal para grupos de comandos anidados, como
  `["nodes"]`

Para las funciones de nodos emparejados, se recomienda
`api.registerNodeCliFeature(registrar, opts?)`. Es un pequeño envoltorio de
`api.registerCli(..., { parentPath: ["nodes"] })` y hace que comandos como
`openclaw nodes canvas` sean funciones de nodo explícitamente propiedad del Plugin.

Para que un comando de Plugin permanezca con carga diferida en la ruta normal de la CLI raíz,
proporcione `descriptors` que cubran todas las raíces de comandos de nivel superior expuestas por ese
registrador.

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerMatrixCli } = await import("./src/cli.js");
    registerMatrixCli({ program });
  },
  {
    descriptors: [
      {
        name: "matrix",
        description: "Gestionar cuentas, verificación, dispositivos y estado del perfil de Matrix",
        hasSubcommands: true,
      },
    ],
  },
);
```

Los comandos anidados reciben el comando principal resuelto como `program`:

```typescript
api.registerCli(
  async ({ program }) => {
    const { registerNodesCanvasCommands } = await import("./src/cli.js");
    registerNodesCanvasCommands(program);
  },
  {
    parentPath: ["nodes"],
    descriptors: [
      {
        name: "canvas",
        description: "Capturar o renderizar contenido del lienzo desde un nodo emparejado",
        hasSubcommands: true,
      },
    ],
  },
);
```

Use `commands` por sí solo únicamente cuando no necesite el registro diferido de la CLI raíz.
Esa ruta de compatibilidad inmediata sigue siendo compatible, pero no instala
marcadores de posición respaldados por descriptores para la carga diferida durante el análisis.

### Registro del backend de la CLI

`api.registerCliBackend(...)` permite que un plugin controle la configuración predeterminada de un backend
local de CLI de IA, como `claude-cli` o `my-cli`.

- El `id` del backend se convierte en el prefijo del proveedor en referencias de modelo como `my-cli/gpt-5`.
- El `config` del backend es el adaptador de comandos autoritativo: el comportamiento de argv, entorno,
  analizador, sesión, imágenes y fiabilidad reside en el código del plugin.
- Los usuarios seleccionan el backend mediante referencias de modelo o `agentRuntime.id` con ámbito de modelo;
  `openclaw.json` no reescribe el adaptador.
- Use `normalizeConfig` cuando los campos estáticos registrados necesiten una fase de
  normalización que tenga en cuenta el entorno de ejecución.
- Use `resolveExecutionArgs` para reescrituras de argv con ámbito de solicitud que pertenezcan
  al dialecto de la CLI, como asignar los niveles de razonamiento de OpenClaw a una marca nativa
  de esfuerzo. El hook recibe `ctx.executionMode`; use `"side-question"` para añadir
  marcas de aislamiento nativas del backend a llamadas efímeras de `/btw`. Si esas marcas
  desactivan de forma fiable las herramientas nativas de una CLI que, de otro modo, siempre las tendría activas, declare
  también `sideQuestionToolMode: "disabled"`.
- Use `prepareExecution` para el entorno de inicio controlado por el backend o puentes
  temporales de autenticación/configuración. Su `ctx.contextTokenBudget` es el límite efectivo
  de tokens seleccionado para la ejecución, de modo que los backends con Compaction nativa puedan alinear su
  propio umbral sin ramas del núcleo específicas del proveedor. También recibe el
  `ctx.env` preparado por el núcleo cuando la preparación del backend debe ampliar la configuración de MCP incluida.
- Los backends que puedan desactivar todas las herramientas nativas para una ejecución concreta pueden declarar
  `nativeToolMode: "selectable"`. Las llamadas restringidas pasan una lista exacta
  de `ctx.toolAvailability.native` junto con nombres canónicos
  de `ctx.toolAvailability.openClaw`. Declare
  `toolAvailabilityEnforcement: "execution-args"` y aplique el contrato en el
  argv final nuevo o reanudado, o declare `"prepare-execution"`, aplíquelo en
  la política preparada y devuelva `toolAvailabilityEnforced: true`. OpenClaw desactiva
  las herramientas nativas para límites del entorno de ejecución, como `toolsAllow` de Cron, y adopta una política de fallo cerrado cuando
  la ruta de aplicación declarada está incompleta.

Para consultar una guía completa de creación, consulte
[Plugins de backend de la CLI](/es/plugins/cli-backend-plugins).

### Ranuras exclusivas

| Método                                     | Qué registra                                                                                                                                                                                  |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `api.registerContextEngine(id, factory)`   | Motor de contexto (solo uno activo a la vez). Las funciones de retorno del ciclo de vida reciben `runtimeSettings` cuando el host puede proporcionar diagnósticos de modelo/proveedor/modo; los motores estrictos antiguos se vuelven a intentar sin esa clave. |
| `api.registerMemoryCapability(capability)` | Capacidad de memoria unificada                                                                                                                                                                          |

### Adaptadores obsoletos de incrustaciones de memoria

| Método                                         | Qué registra                              |
| ---------------------------------------------- | ---------------------------------------------- |
| `api.registerMemoryEmbeddingProvider(adapter)` | Adaptador de incrustaciones de memoria para el plugin activo |

- `registerMemoryCapability` es la API exclusiva del plugin de memoria.
- `registerMemoryCapability` también puede exponer `publicArtifacts.listArtifacts(...)`
  para exportaciones gestionadas por el host. Los plugins complementarios que enumeran esos
  artefactos declarados siguen usando `listActiveMemoryPublicArtifacts(...)` de la fachada
  `openclaw/plugin-sdk/memory-host-core` conservada hasta que exista una API pública específica
  para consumidores; no deben acceder a la estructura privada de otro plugin.
- `MemoryFlushPlan.model` puede fijar el turno de vaciado a una referencia exacta de `provider/model`,
  como `ollama/qwen3:8b`, sin heredar la cadena de respaldo activa.
- `registerMemoryEmbeddingProvider` está obsoleto. Los nuevos proveedores de incrustaciones
  deben usar `api.registerEmbeddingProvider(...)` y
  `contracts.embeddingProviders`.
- Los proveedores existentes específicos de memoria siguen funcionando durante el período
  de migración, pero la inspección de plugins informa de ello como deuda de compatibilidad para
  los plugins no incluidos.

### Eventos y ciclo de vida

| Método                                       | Qué hace                  |
| -------------------------------------------- | ----------------------------- |
| `api.on(hookName, handler, opts?)`           | Hook tipado del ciclo de vida          |
| `api.onConversationBindingResolved(handler)` | Función de retorno de vinculación de conversaciones |

Consulte [Hooks de plugins](/es/plugins/hooks) para ver ejemplos, nombres habituales de hooks y semántica
de las protecciones.

### Semántica de decisión de los hooks

`before_install` es un hook del ciclo de vida del entorno de ejecución de plugins, no la superficie de políticas
de instalación del operador. Use `security.installPolicy` cuando una decisión de permitir o bloquear deba
abarcar rutas de instalación o actualización mediante la CLI y el Gateway.

- `before_tool_call`: devolver `{ block: true }` es definitivo. Cuando un controlador lo establece, se omiten los controladores de menor prioridad.
- `before_tool_call`: devolver `{ block: false }` se considera que no hay decisión (igual que omitir `block`), no una sobrescritura.
- `before_install`: devolver `{ block: true }` es definitivo. Cuando un controlador lo establece, se omiten los controladores de menor prioridad.
- `before_install`: devolver `{ block: false }` se considera que no hay decisión (igual que omitir `block`), no una sobrescritura.
- `reply_dispatch`: devolver `{ handled: true, ... }` es definitivo. Cuando un controlador reclama el envío, se omiten los controladores de menor prioridad y la ruta predeterminada de envío al modelo.
- `message_sending`: devolver `{ cancel: true }` es definitivo. Cuando un controlador lo establece, se omiten los controladores de menor prioridad.
- `message_sending`: devolver `{ cancel: false }` se considera que no hay decisión (igual que omitir `cancel`), no una sobrescritura.
- `message_received`: use el campo tipado `threadId` cuando necesite el enrutamiento de hilos/temas entrantes. Reserve `metadata` para datos adicionales específicos del canal.
- `message_sending`: use los campos de enrutamiento tipados `replyToId` / `threadId` antes de recurrir a `metadata`, específico del canal.
- `gateway_start`: use `ctx.config`, `ctx.workspaceDir` y `ctx.getCron?.()` para el estado de inicio controlado por el Gateway, en lugar de depender de hooks internos de `gateway:startup`. Cron aún puede estar cargándose en este punto.
- `cron_reconciled`: reconstruya una proyección externa completa de Cron tras el inicio o la recarga del planificador. Incluye `reason` y el estado efectivo de `enabled`, incluido `enabled: false`, mientras que `ctx.getCron?.()` devuelve el planificador exacto reconciliado. Pase `ctx.abortSignal` al trabajo de proyección persistente; se cancela cuando esa instantánea del planificador queda reemplazada o se cierra el Gateway.
- `cron_changed`: observe los cambios del ciclo de vida de Cron controlados por el Gateway. Los eventos `scheduled` y `removed` son indicios de reconciliación posteriores a la confirmación, no un registro ordenado de cambios. El `event.nextRunAtMs` de un evento programado está ausente cuando el trabajo no tiene un próximo despertar; un evento eliminado sigue incluyendo la instantánea del trabajo eliminado.

Los planificadores externos de activación deben aplicar antirrebote o combinar los eventos `cron_changed`,
y después volver a leer la vista persistente completa desde el último planificador capturado por
`cron_reconciled`. No adopte el planificador de un contexto de `cron_changed`: un
indicio desvinculado de un planificador anterior puede solaparse con una recarga posterior.

Use `cron_reconciled` como desencadenador de instantánea completa para el estado persistente cargado durante
el inicio del Gateway o la sustitución del planificador. No se reproduce al recargar únicamente un plugin
en caliente. Los controladores de observación se ejecutan en paralelo y los
envíos sin espera de respuesta pueden solaparse, por lo que los consumidores no deben depender del orden de finalización de los eventos.
Mantenga OpenClaw como fuente de verdad para las comprobaciones de vencimiento y la ejecución.

Para consultar un adaptador de ejecución única con sustitución persistente, reintentos/espera incremental y cierre
correcto, consulte [Proyección externa segura de Cron](/es/plugins/hooks#safe-external-cron-projection).

### Campos del objeto de API

| Campo                    | Tipo                      | Descripción                                                                                 |
| ------------------------ | ------------------------- | ------------------------------------------------------------------------------------------- |
| `api.id`                 | `string`                  | Id. del plugin                                                                                   |
| `api.name`               | `string`                  | Nombre para mostrar                                                                                |
| `api.version`            | `string?`                 | Versión del plugin (opcional)                                                                   |
| `api.description`        | `string?`                 | Descripción del plugin (opcional)                                                               |
| `api.source`             | `string`                  | Ruta de origen del plugin                                                                          |
| `api.rootDir`            | `string?`                 | Directorio raíz del plugin (opcional)                                                            |
| `api.config`             | `OpenClawConfig`          | Instantánea actual de la configuración (instantánea activa en memoria del entorno de ejecución, cuando está disponible)                  |
| `api.pluginConfig`       | `Record<string, unknown>` | Configuración específica del plugin de `plugins.entries.<id>.config`                                   |
| `api.runtime`            | `PluginRuntime`           | [Funciones auxiliares del entorno de ejecución](/es/plugins/sdk-runtime)                                                     |
| `api.logger`             | `PluginLogger`            | Registrador con ámbito (`debug`, `info`, `warn`, `error`)                                            |
| `api.registrationMode`   | `PluginRegistrationMode`  | Modo de carga actual; `"setup-runtime"` es la ventana ligera de inicio/configuración anterior a la entrada completa |
| `api.resolvePath(input)` | `(string) => string`      | Resolver una ruta relativa a la raíz del plugin                                                        |

## Convención de módulos internos

Dentro del plugin, use archivos de barril locales para las importaciones internas:

```text
my-plugin/
  api.ts            # Exportaciones públicas para consumidores externos
  runtime-api.ts    # Exportaciones internas exclusivas del entorno de ejecución
  index.ts          # Punto de entrada del plugin
  setup-entry.ts    # Entrada ligera exclusiva para la configuración (opcional)
```

<Warning>
  Nunca importe su propio plugin mediante `openclaw/plugin-sdk/<your-plugin>`
  desde el código de producción. Enrute las importaciones internas mediante `./api.ts` o
  `./runtime-api.ts`. La ruta del SDK es únicamente el contrato externo.
</Warning>

Las superficies públicas de plugins integrados cargadas mediante fachadas (`api.ts`, `runtime-api.ts`,
`index.ts`, `setup-entry.ts` y archivos de entrada públicos similares) prefieren la
instantánea activa de la configuración del entorno de ejecución cuando OpenClaw ya se está ejecutando. Si todavía no existe
ninguna instantánea del entorno de ejecución, recurren al archivo de configuración resuelto en el disco.
Las fachadas de plugins integrados empaquetados deben cargarse mediante los cargadores de fachadas
de plugins de OpenClaw; las importaciones directas desde `dist/extensions/...` omiten las comprobaciones
del manifiesto y del componente auxiliar del entorno de ejecución que las instalaciones empaquetadas utilizan para el código propiedad del plugin.

Los plugins de proveedores pueden exponer un punto de exportación de contrato local al plugin y limitado cuando un
asistente es deliberadamente específico del proveedor y todavía no corresponde incluirlo en una
subruta genérica del SDK. Ejemplos integrados:

- **Anthropic**: interfaz pública `api.ts` / `contract-api.ts` para los
  asistentes de cabeceras beta y de flujo `service_tier` de Claude.
- **`@openclaw/openai-provider`**: `api.ts` exporta constructores de proveedores,
  asistentes de modelos predeterminados y constructores de proveedores en tiempo real.
- **`@openclaw/openrouter-provider`**: `api.ts` exporta el constructor de proveedores
  junto con asistentes de incorporación y configuración.

<Warning>
  El código de producción de las extensiones también debe evitar las importaciones de `openclaw/plugin-sdk/<other-plugin>`.
  Si un asistente es realmente compartido, promuévalo a una subruta neutral del SDK,
  como `openclaw/plugin-sdk/speech`, `.../provider-model-shared` u otra
  superficie orientada a capacidades, en lugar de acoplar dos plugins entre sí.
</Warning>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Puntos de entrada" icon="door-open" href="/es/plugins/sdk-entrypoints">
    Opciones de `definePluginEntry` y `defineChannelPluginEntry`.
  </Card>
  <Card title="Asistentes del entorno de ejecución" icon="gears" href="/es/plugins/sdk-runtime">
    Referencia completa del espacio de nombres `api.runtime`.
  </Card>
  <Card title="Configuración inicial y configuración" icon="sliders" href="/es/plugins/sdk-setup">
    Empaquetado, manifiestos y esquemas de configuración.
  </Card>
  <Card title="Pruebas" icon="vial" href="/es/plugins/sdk-testing">
    Utilidades de prueba y reglas de lint.
  </Card>
  <Card title="Migración del SDK" icon="arrows-turn-right" href="/es/plugins/sdk-migration">
    Migración desde superficies obsoletas.
  </Card>
  <Card title="Aspectos internos de los plugins" icon="diagram-project" href="/es/plugins/architecture">
    Arquitectura detallada y modelo de capacidades.
  </Card>
</CardGroup>
