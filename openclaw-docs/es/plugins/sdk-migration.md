---
read_when:
    - Aparece la advertencia OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED
    - Se muestra la advertencia OPENCLAW_EXTENSION_API_DEPRECATED
    - Usaste `api.registerEmbeddedExtensionFactory` antes de OpenClaw 2026.4.25
    - Estás actualizando un plugin a la arquitectura moderna de plugins
    - Mantiene un plugin externo de OpenClaw
sidebarTitle: Migrate to SDK
summary: Migra de la capa heredada de compatibilidad con versiones anteriores al SDK moderno de plugins
title: Migración del SDK de plugins
x-i18n:
    generated_at: "2026-07-26T04:49:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw sustituyó una amplia capa de compatibilidad con versiones anteriores por una arquitectura de plugins
moderna construida a partir de importaciones pequeñas y específicas. Si el plugin es anterior a ese
cambio, esta guía permite adaptarlo a los contratos actuales.

## Qué cambió

Anteriormente, varias superficies de importación muy abiertas permitían que los plugins accedieran a casi cualquier cosa
desde un único punto de entrada:

- **`openclaw/plugin-sdk`** y **`openclaw/plugin-sdk/compat`**: reexportaban
  decenas de auxiliares mientras se desarrollaba el SDK específico. Ambas raíces se han
  eliminado; importe en su lugar una subruta documentada.
- **`openclaw/plugin-sdk/infra-runtime`**: un barrel amplio que mezclaba eventos del sistema,
  estado de Heartbeat, colas de entrega, auxiliares de obtención/proxy, auxiliares de archivos,
  tipos de aprobación y utilidades no relacionadas.
- **`openclaw/plugin-sdk/config-runtime`**: un barrel de configuración amplio conservado
  únicamente durante su ventana de compatibilidad posterior; se han eliminado los auxiliares directos
  de carga/escritura en tiempo de ejecución.
- **`openclaw/extension-api`**: un puente eliminado que proporcionaba a los plugins acceso directo
  a auxiliares del lado del host, como el ejecutor de agentes integrado.
- **`api.registerEmbeddedExtensionFactory(...)`**: un hook eliminado exclusivo del ejecutor integrado
  que observaba eventos de dicho ejecutor, como `tool_result`. Use en su lugar middleware
  de resultados de herramientas del agente (consulte [Migrar extensiones de resultados de herramientas
  integradas a middleware](#how-to-migrate)).

Se han eliminado el SDK raíz, el barrel de compatibilidad, el puente de extensiones y la fábrica de extensiones integradas.
`infra-runtime` y `config-runtime` se mantienen únicamente durante sus
ventanas posteriores registradas por separado; los plugins nuevos deben usar subrutas específicas.

<Warning>
  Los plugins que importan las superficies raíz, de compatibilidad o de extensión eliminadas ya no
  se cargan. Siga las correspondencias indicadas a continuación antes de actualizar.
</Warning>

OpenClaw no elimina ni reinterpreta el comportamiento documentado de los plugins en el mismo
cambio que introduce un reemplazo. Los cambios incompatibles en los contratos pasan primero por
un adaptador de compatibilidad, diagnósticos, documentación y un periodo de obsolescencia. Esto
se aplica a las importaciones del SDK, los campos del manifiesto, las API de configuración, los hooks y el comportamiento
de registro en tiempo de ejecución.

### Motivos

- **Inicio lento**: importar un auxiliar cargaba decenas de módulos no relacionados.
- **Dependencias circulares**: las reexportaciones amplias facilitaban la creación de
  ciclos de importación.
- **Superficie de API poco clara**: no había forma de distinguir las exportaciones estables de las internas.

Ahora, cada `openclaw/plugin-sdk/<subpath>` es un módulo pequeño y autónomo con
un contrato documentado.

También se han eliminado las antiguas interfaces auxiliares de proveedores para los canales incluidos:
los atajos auxiliares específicos de cada canal eran recursos privados del monorrepositorio, no
contratos estables para plugins. Use en su lugar subrutas genéricas y específicas del SDK. Dentro del
espacio de trabajo de plugins incluidos, mantenga los auxiliares propiedad del proveedor en los
`api.ts` o `runtime-api.ts` del propio plugin:

- Anthropic mantiene los auxiliares de flujos específicos de Claude en su propia interfaz `api.ts` /
  `contract-api.ts`.
- OpenAI mantiene los constructores de proveedores, los auxiliares de modelos predeterminados y los constructores
  de proveedores en tiempo real en su propio `api.ts`.
- OpenRouter mantiene el constructor de proveedores y los auxiliares de incorporación/configuración en su propio
  `api.ts`.

## Política de compatibilidad

El trabajo de compatibilidad de plugins externos sigue este orden:

1. Añadir el nuevo contrato.
2. Mantener el comportamiento anterior conectado mediante un adaptador de compatibilidad.
3. Emitir un diagnóstico o una advertencia que indique la ruta anterior y su reemplazo.
4. Cubrir ambas rutas en las pruebas.
5. Documentar la obsolescencia y la ruta de migración.
6. Eliminar únicamente después del periodo de migración anunciado, normalmente en una versión
   principal.

Si todavía se acepta un campo del manifiesto, siga utilizándolo hasta que la documentación y
los diagnósticos indiquen lo contrario. El código nuevo debe preferir el reemplazo documentado;
los plugins existentes no deben dejar de funcionar durante actualizaciones menores ordinarias.

### Compatibilidad de la configuración de canales publicados

Los paquetes de Slack, Discord, Signal y Microsoft Teams publicados mediante
`2026.7.1` importan esquemas de configuración específicos del canal desde
`openclaw/plugin-sdk/bundled-channel-config-schema`. Los paquetes publicados de Slack y
Discord también importan `createLegacyCompatChannelDmPolicy` y
`promptLegacyChannelAllowFromForAccount` desde
`openclaw/plugin-sdk/setup-runtime`.

Esas exportaciones siguen disponibles como adaptadores de compatibilidad en tiempo de ejecución obsoletos.
Los plugins nuevos y republicados deben gestionar localmente sus propios esquemas de configuración y su política
de configuración mediante primitivas genéricas de `channel-config-schema` y
`setup-runtime`. Las exportaciones de compatibilidad solo pueden eliminarse cuando las
versiones mínimas admitidas de los paquetes publicados ya no las importen.

### Compatibilidad de campos de entrada de configuración de canales

`ChannelSetupInput` ahora mantiene tipado permanentemente solo el contenedor de configuración común
a todos los canales. Los campos específicos de cada canal permanecen tipados en un nivel de compatibilidad
obsoleto para que los plugins externos existentes sigan compilándose mientras sus autores trasladan esos
campos a tipos de entrada de configuración locales del plugin.

OpenClaw no publica versiones principales. Un análisis del registro realizado el 2026-07-22 inspeccionó
426 plugins de canal externos publicados y eliminó 21 campos sin lectores.
Cada uno de los 22 campos conservados tiene un lector publicado conocido. Cada campo adicional se
elimina en cuanto ningún plugin publicado lo lee; el conjunto conservado se reduce a medida que
los autores de plugins migran a tipos de entrada de configuración locales del plugin.

El mismo análisis eliminó 23 claves heredadas de promoción de adaptadores no declaradas sin
dependientes publicados. Permanecen seis claves comunes y la clave exclusiva de configuración `rooms`.
Ese conjunto también se reduce a medida que los plugins publicados declaran `singleAccountKeysToMove`.

El tipo compartido no tiene firma de índice. Las claves propiedad del plugin pueden seguir presentes
en los objetos de entrada en tiempo de ejecución; declárelas en una intersección local del plugin o redúzcalas
mediante el esquema de configuración del plugin propietario.

| `code`                                  | `owner`   | `replacement`                                                                                    | Condición de eliminación                                               |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | Intersecar `ChannelSetupInput` con un tipo local del plugin que declare los campos del canal propietario | Eliminar un campo cuando el análisis del registro de plugins publicados no encuentre ningún lector |

El nivel heredado de promoción de adaptadores no declarados sigue la misma política basada en lectores.
Declare `singleAccountKeysToMove`, incluido un array vacío cuando el
plugin no necesite claves de promoción adicionales, para que el recurso compartido alternativo pueda retirarse
clave por clave.

#### Verificación de lectores

1. Recorra por páginas `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` con cada `nextCursor` y conserve los paquetes cuyos `categories` incluyan `channels`.
2. Añada candidatos de npm desde `npm search --json --searchlimit=1000 "openclaw channel plugin"`. Añada candidatos disponibles solo como código fuente a partir de búsquedas de código de GitHub para `openclaw/plugin-sdk/channel-setup`, `openclaw/plugin-sdk/setup` y `openclaw/plugin-sdk/core`.
3. Determine la última versión publicada de cada candidato. Ejecute `npm pack <package>@<version> --json --pack-destination <temp-dir>`, desempaquétela e inspeccione el JavaScript y las declaraciones distribuidos en `dist` para detectar lecturas directas o desestructuradas de campos. Descargue el artefacto de ClawHub cuando un paquete no tenga una versión en npm.
4. Registre el paquete, la versión, el campo o la clave de promoción y el archivo coincidente. Un campo o una clave solo puede eliminarse cuando ningún artefacto publicado de un plugin lo lea. Mantenga sincronizados con el análisis los nombres de los lectores en los comentarios del código junto a las listas de campos y claves conservados.

Este es únicamente un registro de compatibilidad de código fuente/tipos. No tiene ningún adaptador en tiempo de ejecución ni
entrada en el registro de compatibilidad porque los objetos de entrada de configuración en tiempo de ejecución y el comportamiento
de configuración no han cambiado.

Audite la cola de migración actual con `pnpm plugins:boundary-report`:

| Opción                                                  | Efecto                                                                         |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary` (o `pnpm plugins:boundary-report:summary`) | Recuentos compactos en lugar de detalles completos.                            |
| `--json`                                                | Informe legible por máquinas.                                                  |
| `--owner <id>`                                          | Filtrar por un plugin o propietario de compatibilidad.                         |
| `--fail-on-cross-owner`                                 | Salir con un código distinto de cero ante importaciones reservadas del SDK entre propietarios. |
| `--fail-on-eligible-compat`                             | Salir con un código distinto de cero cuando haya pasado la fecha `removeAfter` de un registro de compatibilidad obsoleto. |
| `--fail-on-unclassified-unused-reserved`                | Salir con un código distinto de cero ante shims reservados del SDK sin usar.    |

`pnpm plugins:boundary-report:ci` se ejecuta con las tres opciones de error. Los registros
obsoletos suelen tener una fecha `removeAfter` explícita en lugar de una imprecisa «próxima
versión principal». Un registro cuyo propietario no haya aprobado una fecha deja
`removeAfter` ausente, aparece como `no-date` y nunca puede eliminarse.
El informe agrupa los registros obsoletos por fecha, cuenta las referencias locales de código/documentación,
muestra las importaciones reservadas del SDK entre propietarios y resume el puente privado
del SDK del host de memoria. Las subrutas reservadas del SDK deben tener un uso registrado por parte del propietario;
las exportaciones reservadas sin usar deben eliminarse del SDK público.

### Proyección heredada de medios

El registro de compatibilidad `media-legacy-projection` abarca los antiguos campos
paralelos de medios, los constructores de payloads, los alias de metadatos de hooks y los nombres
de plantillas de medios. Su fecha `removeAfter` aprobada es **2026-10-01** (dos ciclos de publicación
después de que se distribuyeran los reemplazos basados primero en hechos). Además, su eliminación requiere
en ese momento un análisis limpio de los artefactos de plugins publicados; migre antes de la fecha.

Para la entrada de canales, sustituya los valores singular/plural `MediaPath`, `MediaUrl`,
`MediaType`, `MediaPaths`, `MediaUrls`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` y `MediaStaged` por hechos
ordenados:

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

Use `event.media` en los hooks `inbound_claim` y `message_received`. Si los medios remotos
no se han preparado localmente, use `event.originalMedia` para identidad/diagnósticos
y espere a `event.media`; `event.mediaStagingPending` distingue ese
estado. No lea las propiedades singular/plural obsoletas de
`event.metadata`.

Para los modelos multimedia de la CLI, sustituya `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`
y `{{MediaDir}}` por `{{AttachmentPath}}`, `{{AttachmentUrl}}`,
`{{AttachmentContentType}}` y `{{AttachmentDir}}`. Use
`{{AttachmentIndex}}` cuando la posición de los archivos adjuntos sea importante.

Para la política de lectura de medios locales, importe `getAgentScopedMediaLocalRoots(...)` o
`getAgentScopedMediaLocalRootsForSources(...)` desde
`openclaw/plugin-sdk/media-local-roots`. La fachada
`openclaw/plugin-sdk/agent-media-payload` y su
proyección `buildAgentMediaPayload(...)` están obsoletas.

## Cómo migrar

<Steps>
  <Step title="Migrar los auxiliares de carga/escritura de configuración en tiempo de ejecución">
    Los plugins incluidos deben dejar de llamar directamente a `api.runtime.config.loadConfig()` y
    `api.runtime.config.writeConfigFile(...)`. Es preferible usar la configuración ya
    pasada a la ruta de llamada activa. Los controladores de larga duración que necesiten la
    instantánea actual del proceso pueden usar `api.runtime.config.current()`. Las herramientas
    de agente de larga duración deben leer `ctx.getRuntimeConfig()` dentro de `execute` para que una herramienta
    creada antes de una escritura de configuración siga viendo la configuración actualizada.

    Las escrituras de configuración pasan por el auxiliar transaccional con una política explícita
    posterior a la escritura:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    Use `afterWrite: { mode: "restart", reason: "..." }` cuando el cambio requiera
    un reinicio limpio del Gateway, y `afterWrite: { mode: "none", reason: "..." }`
    solo cuando el llamador sea responsable del seguimiento y suprima deliberadamente el
    planificador de recarga. Los resultados de las mutaciones incluyen un resumen tipado `followUp` para
    pruebas y registros; el Gateway sigue siendo responsable de aplicar o
    programar el reinicio.

    `loadConfig` y `writeConfigFile` se han eliminado del entorno de ejecución
    de plugins. Los plugins incluidos y el código del entorno de ejecución del repositorio están protegidos por
    `pnpm check:deprecated-api-usage` y
    `pnpm check:no-runtime-action-load-config`: el nuevo uso en producción de plugins
    falla por completo, las escrituras directas de configuración fallan, los métodos del servidor Gateway deben usar
    la instantánea del entorno de ejecución de la solicitud, los asistentes de envío/acción/cliente de canales del entorno de ejecución
    deben recibir la configuración desde su límite, y los módulos de larga duración del entorno de ejecución
    no permiten ninguna llamada ambiental a `loadConfig()`.

    El código nuevo de plugins debe evitar el barrel amplio `openclaw/plugin-sdk/config-runtime`.
    Use la subruta específica para la tarea:

    | Necesidad | Importación |
    | --- | --- |
    | Tipos de configuración como `OpenClawConfig` | `openclaw/plugin-sdk/config-contracts` |
    | Búsqueda de configuración en la entrada del plugin | `api.pluginConfig` |
    | Combinación de configuraciones | Lógica local del plugin en el límite de configuración |
    | Lecturas de la instantánea actual del entorno de ejecución | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | Escrituras de configuración | `openclaw/plugin-sdk/config-mutation` |
    | Asistentes del almacén de sesiones | `openclaw/plugin-sdk/session-store-runtime` |
    | Configuración de tablas Markdown | `openclaw/plugin-sdk/markdown-table-runtime` |
    | Asistentes del entorno de ejecución de políticas de grupo | `openclaw/plugin-sdk/runtime-group-policy` |
    | Resolución de entrada de secretos | `openclaw/plugin-sdk/secret-input-runtime` |
    | Sustituciones de modelo/sesión | `openclaw/plugin-sdk/model-session-runtime` |

    Los plugins incluidos y sus pruebas están protegidos mediante análisis contra el barrel
    amplio para que las importaciones y los mocks permanezcan limitados al comportamiento que necesitan. El
    barrel sigue existiendo por compatibilidad externa, pero el código nuevo no debe
    depender de él.

  </Step>

  <Step title="Migrar las extensiones de resultados de herramientas integradas a middleware">
    Los plugins incluidos deben reemplazar los controladores de resultados de herramientas
    `api.registerEmbeddedExtensionFactory(...)`, exclusivos del ejecutor integrado, por
    middleware independiente del entorno de ejecución:

    ```typescript
    // Herramientas del entorno de ejecución de OpenClaw y herramientas dinámicas del entorno de ejecución de Codex (el resultado puede
    // transformarse). Los resultados de herramientas nativas de Codex también se retransmiten para su observación,
    // pero su salida transformada nunca llega al modelo: el contrato del hook
    // PostToolUse de Codex no puede reemplazar una respuesta de herramienta nativa.
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    Actualice al mismo tiempo el manifiesto del plugin:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    Los plugins instalados también pueden registrar middleware de resultados de herramientas cuando esté habilitado
    explícitamente y cada entorno de ejecución de destino esté declarado en
    `contracts.agentToolResultMiddleware`. Los registros de middleware instalado
    no declarado se rechazan.

  </Step>

  <Step title="Migrar los controladores nativos de aprobación a hechos de capacidad">
    Los plugins de canal con capacidad de aprobación exponen el comportamiento nativo de aprobación mediante
    `approvalCapability.nativeRuntime` junto con el registro compartido de contexto
    del entorno de ejecución:

    - Reemplace `approvalCapability.handler.loadRuntime(...)` por
      `approvalCapability.nativeRuntime`.
    - Traslade la autenticación/entrega específica de aprobaciones fuera del cableado heredado de `plugin.auth` /
      `plugin.approvals` y pásela a `approvalCapability`.
    - `ChannelPlugin.approvals` se ha eliminado del contrato público
      de plugins de canal; traslade los campos de entrega/nativos/renderizado a
      `approvalCapability`.
    - `plugin.auth` se mantiene únicamente para los flujos de inicio/cierre de sesión del canal; el núcleo ya no
      lee allí los hooks de autenticación de aprobaciones.
    - Registre los objetos del entorno de ejecución propiedad del canal (clientes, tokens, aplicaciones Bolt)
      mediante `openclaw/plugin-sdk/channel-runtime-context`.
    - No envíe avisos de redireccionamiento propiedad del plugin desde los controladores nativos de aprobación;
      el núcleo controla los avisos de redireccionamiento a otro lugar a partir de los resultados reales de entrega.
    - Al pasar `channelRuntime` a `createChannelManager(...)`, proporcione una
      superficie `createPluginRuntime().channel` real; los stubs parciales se
      rechazan.

    Consulte [Plugins de canal](/es/plugins/sdk-channel-plugins) para conocer la disposición actual
    de la capacidad de aprobación.

  </Step>

  <Step title="Auditar el comportamiento de respaldo de los wrappers de Windows">
    Si el plugin usa `openclaw/plugin-sdk/windows-spawn`, los wrappers de Windows
    `.cmd`/`.bat` no resueltos ahora fallan de forma cerrada, salvo que se pase explícitamente
    `allowShellFallback: true`:

    ```typescript
    // Antes
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Después
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Establezca esto únicamente para llamadores de compatibilidad de confianza que acepten
      // deliberadamente el respaldo mediado por el shell.
      allowShellFallback: true,
    });
    ```

    Si el llamador no depende intencionadamente del respaldo del shell, no establezca
    `allowShellFallback` y gestione en su lugar el error generado.

  </Step>

  <Step title="Buscar importaciones obsoletas">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="Reemplazar por importaciones específicas">
    Cada exportación de la superficie anterior se corresponde con una ruta de importación moderna específica:

    ```typescript
    // Antes (capa obsoleta de retrocompatibilidad)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Después (importaciones modernas específicas)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Para los asistentes del lado del host, use el entorno de ejecución del plugin inyectado en lugar de
    realizar una importación directa:

    ```typescript
    // Antes (puente extension-api obsoleto)
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // Después (entorno de ejecución inyectado)
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    El mismo patrón se aplica a otros asistentes de puente heredados:

    | Importación anterior | Equivalente moderno |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | asistentes del almacén de sesiones | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Reemplazar las importaciones amplias de infra-runtime">
    `openclaw/plugin-sdk/infra-runtime` sigue existiendo por compatibilidad
    externa, pero el código nuevo debe importar la superficie específica que realmente
    necesita:

    | Necesidad | Importación |
    | --- | --- |
    | Asistentes de la cola de eventos del sistema | `openclaw/plugin-sdk/system-event-runtime` |
    | Asistentes de activación, eventos y visibilidad de Heartbeat | `openclaw/plugin-sdk/heartbeat-runtime` |
    | Vaciado de la cola de entregas pendientes | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | Telemetría de actividad del canal | `openclaw/plugin-sdk/channel-activity-runtime` |
    | Cachés de desduplicación en memoria y respaldadas por almacenamiento persistente | `openclaw/plugin-sdk/dedupe-runtime` |
    | Asistentes seguros para rutas de archivos locales y medios | `openclaw/plugin-sdk/file-access-runtime` |
    | Solicitud consciente del despachador | `openclaw/plugin-sdk/runtime-fetch` |
    | Asistentes de solicitudes mediante proxy y protegidas | `openclaw/plugin-sdk/fetch-runtime` |
    | Tipos de políticas del despachador contra SSRF | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | Tipos de solicitud/resolución de aprobación | `openclaw/plugin-sdk/approval-runtime` |
    | Asistentes de comandos y cargas útiles de respuesta de aprobación | `openclaw/plugin-sdk/approval-reply-runtime` |
    | Asistentes de formato de errores | `openclaw/plugin-sdk/error-runtime` |
    | Esperas de disponibilidad del transporte | `openclaw/plugin-sdk/transport-ready-runtime` |
    | Asistentes seguros para tokens | `openclaw/plugin-sdk/secure-random-runtime` |
    | Concurrencia limitada de tareas asíncronas | `openclaw/plugin-sdk/concurrency-runtime` |
    | Aserciones de valores obligatorios para invariantes demostrables | `openclaw/plugin-sdk/expect-runtime` |
    | Coerción numérica | `openclaw/plugin-sdk/number-runtime` |
    | Bloqueo asíncrono local del proceso | `openclaw/plugin-sdk/async-lock-runtime` |
    | Bloqueos de archivos | `openclaw/plugin-sdk/file-lock` |

    Los plugins incluidos están protegidos mediante análisis contra `infra-runtime`, por lo que el código del repositorio
    no puede volver a usar el barrel amplio.

  </Step>

  <Step title="Migrar los asistentes de rutas de canal">
    El código nuevo de rutas de canal usa `openclaw/plugin-sdk/channel-route`. Los nombres anteriores
    de claves de ruta se mantienen como alias de compatibilidad:

    | Asistente anterior | Asistente moderno |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    Los asistentes modernos de rutas normalizan `{ channel, to, accountId, threadId }`
    de forma coherente entre las aprobaciones nativas, la supresión de respuestas, la desduplicación
    entrante, la entrega de Cron y el enrutamiento de sesiones.

    No añada usos nuevos de `ChannelMessagingAdapter.parseExplicitTarget` ni
    `resolveChannelRouteTargetWithParser(...)` desde
    `plugin-sdk/channel-route`; están obsoletos y se mantienen únicamente para plugins
    anteriores. Los plugins de canal nuevos deben usar
    `messaging.targetResolver.resolveTarget(...)` para la normalización del identificador de destino
    y el respaldo cuando no se encuentra el directorio,
    `messaging.inferTargetChatType(...)` cuando el núcleo necesite conocer de forma anticipada el tipo de par,
    y `messaging.resolveOutboundSessionRoute(...)` para la identidad de sesión
    e hilo nativa del proveedor.

  </Step>

  <Step title="Compilar y probar">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## Referencia de rutas de importación

El mapa de exportaciones del paquete público es la fuente de verdad de las subrutas
importables del SDK. Use las guías temáticas del SDK enlazadas desde la [descripción general del SDK](/es/plugins/sdk-overview)
y prefiera la subruta pública documentada más específica. El inventario del compilador en
`scripts/lib/plugin-sdk-entrypoints.json` también contiene entradas locales privadas utilizadas
para compilar plugins incluidos; su presencia allí no las convierte en exportaciones públicas del paquete.

Esta tabla es el subconjunto común de migración, no toda la superficie del SDK. El
inventario de puntos de entrada del compilador se encuentra en `scripts/lib/plugin-sdk-entrypoints.json`;
las exportaciones del paquete se generan a partir del subconjunto público.

Las interfaces auxiliares reservadas para plugins incluidos se han retirado del mapa de exportaciones
del SDK público, salvo las fachadas de compatibilidad documentadas explícitamente, como el
shim obsoleto `plugin-sdk/discord`, conservado para plugins externos que todavía
importan directamente el paquete publicado `@openclaw/discord`. Los asistentes específicos
de cada propietario residen dentro del paquete del plugin correspondiente; el comportamiento compartido del host se
canaliza mediante contratos genéricos del SDK, como `plugin-sdk/gateway-runtime`,
`plugin-sdk/security-runtime` y la API del plugin inyectada.

Use la importación más específica que corresponda a la tarea. Si no encuentra una exportación,
consulte el código fuente en `src/plugin-sdk/` o pregunte a los responsables qué contrato
genérico debe asumirla.

## Superficies de compatibilidad eliminadas

La revisión de julio de 2026 eliminó los barrels raíz y de compatibilidad del SDK, el puente de la API
de extensiones, los alias caducados de subrutas del SDK, las subrutas del SDK sin uso y las exportaciones
públicas de módulos del SDK exclusivos de plugins incluidos. Los módulos exclusivos de plugins incluidos
siguen disponibles para sus propietarios del repositorio mediante asignaciones de compilación locales privadas;
no pueden importarse desde el paquete publicado.

### Publicación global del proceso de proveedores de API

`registerApiProvider(...)` y `unregisterApiProviders(...)` se eliminaron de
`openclaw/plugin-sdk/llm`. Publicaban transportes de API en el estado global del
proceso, que los entornos de ejecución de modelos administrados por el ciclo de vida debían copiar después en cada registro
preparado.

Los plugins de proveedores deben registrar los proveedores de inferencia de texto mediante
`api.registerProvider(...)`. El código y las pruebas propiedad del host que construyan un
`ApiRegistry` deben registrarse directamente en ese registro para que la propiedad
del proveedor y su desmontaje permanezcan limitados al entorno de ejecución preparado.

### Barrel privado de pruebas

`openclaw/plugin-sdk/testing` era local del repositorio y estaba excluido de los artefactos
del paquete distribuido, por lo que se eliminó antes de su fecha `removeAfter` de 2026-07-28. Las pruebas del repositorio
usan subrutas específicas como `plugin-sdk/plugin-test-runtime`,
`plugin-sdk/channel-test-helpers`, `plugin-sdk/channel-target-testing`,
`plugin-sdk/test-env` y `plugin-sdk/test-fixtures`.

## Referencia de migración

  Estas correspondencias abarcan tanto las superficies eliminadas en julio de 2026 como las
  desaprobaciones activas de periodos posteriores. Una correspondencia es una guía de migración,
  no una prueba de que la superficie anterior siga disponible; consulte el registro de
  compatibilidad y el calendario de eliminación para conocer el estado actual.

  <AccordionGroup>
  <Accordion title="Generadores de ayuda de command-auth -> command-status">
    **Anterior (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`,
    `buildCommandsMessagePaginated`, `buildHelpMessage`.

    **Nuevo (`openclaw/plugin-sdk/command-status`)**: las mismas firmas, importadas
    desde la subruta más específica. Se han eliminado las reexportaciones de compatibilidad
    de `command-auth`.

    ```typescript
    // Antes
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // Después
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="Funciones auxiliares de control de menciones -> resolveInboundMentionDecision">
    **Anterior**: `resolveMentionGating(params)` y
    `resolveMentionGatingWithBypass(params)` de
    `openclaw/plugin-sdk/channel-inbound` o
    `openclaw/plugin-sdk/channel-mention-gating`.

    **Nuevo**: `resolveInboundMentionDecision({ facts, policy })`: un único objeto
    de decisión en lugar de dos formas de llamada separadas.

    Adoptado en Discord, iMessage, Matrix, MS Teams, QQBot, Signal,
    Telegram, WhatsApp y Zalo. El modelo de eventos `app_mention` propio de Slack
    no utiliza esta función auxiliar.

  </Accordion>

  <Accordion title="Capa de compatibilidad del entorno de ejecución del canal y funciones auxiliares de acciones del canal">
    Se ha eliminado `openclaw/plugin-sdk/channel-runtime`. Utilice
    `openclaw/plugin-sdk/channel-runtime-context` para registrar objetos del entorno de ejecución.

    Las funciones auxiliares del esquema de mensajes nativos de `openclaw/plugin-sdk/channel-actions`
    se eliminaron junto con las exportaciones de acciones sin procesar del canal. En su lugar,
    exponga las capacidades mediante la superficie semántica `presentation`: los plugins
    de canal declaran qué representan (tarjetas, botones y selectores), en vez de los nombres
    de acciones sin procesar que aceptan.

  </Accordion>

  <Accordion title="Función auxiliar tool() del proveedor de búsqueda web -> createTool() en el plugin">
    **Anterior**: fábrica `tool()` de `openclaw/plugin-sdk/provider-web-search`.

    **Nuevo**: implemente `createTool(...)` directamente en el plugin del proveedor.
    OpenClaw ya no necesita la función auxiliar del SDK para registrar el contenedor de la herramienta.

  </Accordion>

  <Accordion title="Envoltorios de canal de texto sin formato -> BodyForAgent">
    **Anterior**: `api.runtime.channel.reply.formatInboundEnvelope(...)` (y el
    campo `channelEnvelope` de los objetos de mensajes entrantes) para crear un
    envoltorio plano de solicitud en texto sin formato a partir de mensajes entrantes del canal.

    **Nuevo**: `BodyForAgent` más bloques estructurados de contexto del usuario. Los plugins
    de canal adjuntan los metadatos de enrutamiento (hilo, tema, respuesta y reacciones) como
    campos tipados, en lugar de concatenarlos en una cadena de solicitud. La función auxiliar
    `formatAgentEnvelope(...)` sigue siendo compatible con los envoltorios sintetizados
    dirigidos al asistente, pero los envoltorios entrantes de texto sin formato están en proceso
    de eliminación.

    Áreas afectadas: `inbound_claim`, `message_received` y cualquier plugin
    de canal personalizado que procesara posteriormente el texto del envoltorio anterior.

  </Accordion>

  <Accordion title="Enlace deactivate -> gateway_stop">
    **Anterior**: `api.on("deactivate", handler)`.

    **Nuevo**: `api.on("gateway_stop", handler)`. El mismo contrato de limpieza
    durante el apagado; solo cambia el nombre del enlace.

    ```typescript
    // Antes
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // Después
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` continúa conectado como alias de compatibilidad desaprobado hasta su
    eliminación después del 2026-08-16.

  </Accordion>

  <Accordion title="Enlace subagent_spawning -> vinculación de hilos del núcleo">
    **Anterior**: `api.on("subagent_spawning", handler)` devolvía
    `threadBindingReady` o `deliveryOrigin`.

    **Nuevo**: permita que el núcleo prepare las vinculaciones de subagentes `thread: true`
    mediante el adaptador de vinculación de sesiones del canal. Utilice `api.on("subagent_spawned", handler)`
    únicamente para la observación posterior al inicio.

    ```typescript
    // Antes
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // Después
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`, `PluginHookSubagentSpawningEvent`,
    `PluginHookSubagentSpawningResult` y
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` se mantienen únicamente como
    superficies de compatibilidad desaprobadas mientras migran los plugins externos y se
    eliminarán después del 2026-08-30.

  </Accordion>

  <Accordion title="Tipos de descubrimiento de proveedores -> tipos del catálogo de proveedores">
    Cuatro alias de tipos de descubrimiento son ahora contenedores ligeros de los tipos
    de la era del catálogo:

    | Alias anterior            | Tipo nuevo                |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    Se han eliminado los alias y el contenedor estático heredado `ProviderCapabilities`.
    Los plugins de proveedores deben utilizar enlaces explícitos del proveedor, como
    `buildReplayPolicy`, `normalizeToolSchemas` y `wrapStreamFn`, en lugar de un objeto estático.

  </Accordion>

  <Accordion title="Enlaces de políticas de razonamiento -> resolveThinkingProfile">
    **Anterior** (tres enlaces separados en `ProviderThinkingPolicy`):
    `isBinaryThinking(ctx)`, `supportsXHighThinking(ctx)` y
    `resolveDefaultThinkingLevel(ctx)`.

    **Nuevo**: un único `resolveThinkingProfile(ctx)` que devuelve un
    `ProviderThinkingProfile` con el `id` canónico, un `label` opcional y una
    lista ordenada de niveles. OpenClaw reduce automáticamente los valores almacenados obsoletos
    según la clasificación del perfil.

    El contexto incluye `provider`, `modelId`, un `reasoning` combinado opcional
    y datos combinados opcionales del modelo `compat`. Los plugins de proveedores pueden utilizar esos
    datos del catálogo para exponer un perfil específico del modelo únicamente cuando el contrato
    de solicitud configurado lo admita.

    Implemente un enlace en lugar de tres. Se han eliminado los enlaces heredados.

  </Accordion>

  <Accordion title="Proveedores de autenticación externos -> contracts.externalAuthProviders">
    **Anterior**: implementar enlaces de autenticación externa sin declarar el proveedor
    en el manifiesto del plugin.

    **Nuevo**: declare `contracts.externalAuthProviders` en el manifiesto del plugin
    **e** implemente `resolveExternalAuthProfiles(...)`.

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="Búsqueda de variables de entorno del proveedor -> setup.providers[].envVars">
    Campo **anterior** del manifiesto: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`.

    **Nuevo**: replique la misma búsqueda de variables de entorno en `setup.providers[].envVars`
    dentro del manifiesto. Esto consolida los metadatos de variables de entorno de configuración
    y estado en un solo lugar, y evita iniciar el entorno de ejecución del plugin únicamente
    para resolver búsquedas de variables de entorno.

    `providerAuthEnvVars` ya no se acepta.

  </Accordion>

  <Accordion title="Registro del plugin de memoria -> registerMemoryCapability">
    **Anterior**: tres llamadas separadas: `api.registerMemoryPromptSection(...)`,
    `api.registerMemoryFlushPlan(...)`, `api.registerMemoryRuntime(...)`.

    **Nuevo**: una llamada en la API de estado de memoria:
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`.

    Las mismas ranuras, una única llamada de registro. Las funciones auxiliares aditivas de solicitud
    y corpus (`registerMemoryPromptSupplement`, `registerMemoryCorpusSupplement`) no se ven
    afectadas.

  </Accordion>

  <Accordion title="API del proveedor de incrustaciones de memoria">
    **Anterior**: `api.registerMemoryEmbeddingProvider(...)` más
    `contracts.memoryEmbeddingProviders`.

    **Nuevo**: `api.registerEmbeddingProvider(...)` más
    `contracts.embeddingProviders`.

    El contrato genérico del proveedor de incrustaciones se puede reutilizar fuera de la memoria
    y es la vía compatible para los proveedores nuevos. La API de registro específica de memoria
    continúa conectada como compatibilidad desaprobada mientras migran los proveedores existentes.
    La inspección de plugins informa del uso fuera del paquete como deuda de compatibilidad.

  </Accordion>

  <Accordion title="Resultados sin procesar de envíos de canal -> OutboundDeliveryResult">
    **Anterior**: devolvía `{ ok, messageId, error }` mediante
    `ChannelSendRawResult` y lo normalizaba con
    `createRawChannelSendResultAdapter(...)`.

    **Nuevo**: devuelva los campos `OutboundDeliveryResult` y adjunte el canal con
    `createAttachedChannelResultAdapter(...)`. Los envíos fallidos deben generar una excepción
    en lugar de devolver una cadena de error. El tipo de resultado sin procesar seguirá disponible
    hasta la próxima versión principal del SDK de plugins.

  </Accordion>

  <Accordion title="Cambio de nombre de los tipos de mensajes de sesión de subagentes">
    Dos alias de tipos heredados que aún se exportan desde `src/plugins/runtime/types.ts`:

    | Anterior                      | Nuevo                           |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    El método del entorno de ejecución `readSession` está desaprobado en favor de
    `getSessionMessages`. La misma firma; el método anterior delega en el nuevo.

  </Accordion>

  <Accordion title="API eliminadas de archivos de sesiones y transcripciones">
    El cambio de sesiones y transcripciones a SQLite elimina o desaprueba las API orientadas a plugins
    que exponían almacenes `sessions.json` activos, rutas de transcripciones JSONL o listas
    de archivos de sesiones. Los plugins del entorno de ejecución deben utilizar la identidad de sesión
    y las funciones auxiliares del entorno de ejecución del SDK, en lugar de resolver o modificar archivos activos.

    | Superficie que se migra | Sustitución |
    | ----------------- | ----------- |
    | `loadSessionStore(...)`, `updateSessionStore(...)` y `resolveSessionStoreEntry(...)` desaprobados | `getSessionEntry(...)`, `listSessionEntries(...)` y modificaciones de sesiones a nivel de fila. |
    | `resolveSessionFilePath(...)` desaprobado | Identidad de sesión (`sessionKey`, `sessionId` y funciones auxiliares de destino del entorno de ejecución del SDK), además de métodos del Gateway que operan en la sesión actual. |
    | `saveSessionStore(...)` eliminado | API del entorno de ejecución de sesiones propiedad del Gateway; el código del plugin debe solicitar o modificar el estado de la sesión mediante funciones auxiliares documentadas del entorno de ejecución o del contexto, en lugar de escribir en el archivo activo del almacén. |
    | `resolveSessionTranscriptPathInDir(...)` y `resolveAndPersistSessionFile(...)` eliminados | Identidad de sesión y métodos del Gateway que operan en la sesión actual. |
    | `readLatestAssistantTextFromSessionTranscript(...)` | Lectores de transcripciones respaldados por la identidad y expuestos por el contexto actual del entorno de ejecución, o métodos de historial y sesión del Gateway cuando el plugin se encuentra fuera de la ruta del propietario de la transcripción. |
    | `SessionTranscriptUpdate.sessionFile` | `SessionTranscriptUpdate.target` con `agentId`, `sessionKey` y `sessionId`. |
    | Entradas de sincronización de memoria como `sessionFiles` | Fuentes de transcripciones y sesiones respaldadas por la identidad y proporcionadas por el host; no recorra archivos JSONL activos de sesiones en curso. |
    | Opciones del entorno de ejecución denominadas `transcriptPath` o `sessionFile` para sesiones activas | Objetos `sessionTarget` o de destino del entorno de ejecución que transportan una identidad de sesión independiente del almacenamiento. |

    Los archivos de transcripciones JSONL heredados siguen siendo válidos como artefactos de importación,
    archivo, exportación y soporte. Ya no son el contrato permanente del entorno de ejecución para
    las sesiones activas.

    Los plugins oficiales publicados con `v2026.7.1-beta.5` importaban las cuatro
    funciones auxiliares desaprobadas anteriores. `openclaw/plugin-sdk/session-store-runtime` mantiene
    exactamente ese puente hasta el 2026-10-12; los plugins nuevos deben utilizar las sustituciones.
    `resolveStorePath(...)` sigue siendo una función auxiliar compatible del SDK y no forma parte de
    esta desaprobación.

    `openclaw plugins inspect --all --runtime` informa de los plugins fuera del paquete cuyos
    errores de carga o diagnósticos todavía hacen referencia a estas API de archivos eliminadas. El
    análisis de avisos `@openclaw/plugin-inspector` debe utilizar la versión `0.3.17` o
    una posterior, para que los análisis de paquetes externos también detecten las funciones auxiliares
    del almacén completo de sesiones, las funciones auxiliares de rutas de archivos de sesiones, los destinos
    de archivos de transcripciones heredados y las funciones auxiliares de bajo nivel para transcripciones
    antes de la publicación.

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **Anterior**: `runtime.tasks.flow` (singular) devolvía un descriptor activo
    de flujo de tareas.

    **Nuevo**: `runtime.tasks.managedFlows` conserva el entorno de ejecución administrado de modificación de
    TaskFlow para los plugins que crean, actualizan, cancelan o ejecutan tareas secundarias desde un
    flujo. Utilice `runtime.tasks.flows` cuando el plugin solo necesite
    lecturas basadas en DTO.

    ```typescript
    // Antes
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // Después
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    Los alias heredados se eliminaron en julio de 2026.

  </Accordion>

  <Accordion title="Fábricas de extensiones integradas -> middleware de resultados de herramientas del agente">
    Se explica en [Cómo migrar](#how-to-migrate) más arriba. Se incluye aquí para
    completar la información: la ruta eliminada exclusiva del ejecutor integrado
    `api.registerEmbeddedExtensionFactory(...)` se sustituye por
    `api.registerAgentToolResultMiddleware(...)` con una lista explícita de entornos de ejecución
    en `contracts.agentToolResultMiddleware`.
  </Accordion>

  <Accordion title="Alias OpenClawSchemaType -> OpenClawConfig">
    Se eliminó el alias `OpenClawSchemaType` del SDK raíz. Utilice el nombre
    canónico `OpenClawConfig`.

    ```typescript
    // Antes
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // Después
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
Las obsolescencias a nivel de extensión (dentro de los plugins de canal/proveedor
incluidos en `extensions/`) se registran en sus propios barrels
`api.ts` y `runtime-api.ts`. No afectan a los contratos de plugins de
terceros y no se enumeran aquí. Si se consume directamente el barrel local de
un plugin incluido, deben leerse los comentarios de obsolescencia de ese barrel
antes de actualizar.
</Note>

## Migración de Talk y voz en tiempo real

El código de voz en tiempo real, telefonía, reuniones y Talk en el navegador
comparte un único controlador de sesiones de Talk exportado por
`openclaw/plugin-sdk/realtime-voice`. El controlador gestiona el sobre común de eventos de Talk,
el estado del turno activo, el estado de captura, el estado del audio de salida,
el historial de eventos recientes y el rechazo de turnos obsoletos. Los plugins
de proveedor gestionan las sesiones en tiempo real específicas del proveedor.
Los plugins de reuniones en el navegador utilizan `openclaw/plugin-sdk/meeting-runtime` para los
mecanismos de sesión, navegador, audio, host de Node, consulta con el agente y
llamada de voz, y luego implementan `MeetingPlatformAdapter` para las reglas de URL,
los scripts del DOM, la asignación de acciones manuales, los subtítulos, la
creación y los planes de acceso telefónico. Las API REST de la plataforma,
OAuth, los artefactos, los selectores y los nombres del protocolo permanecen en
el plugin. Los planes de permisos del navegador reciben la URL de reunión
solicitada para que cada plataforma pueda conceder únicamente sus orígenes
compatibles exactos. Los entornos de ejecución de sesión también deben
normalizar el estado de funcionamiento en directo específico de la plataforma
tras confirmarse la salida del navegador; los campos históricos de la
transcripción pueden permanecer, pero la disponibilidad de subtítulos y audio
no debe seguir activa después de salir.

Todas las superficies incluidas se ejecutan en el controlador compartido: relay
del navegador, transferencia de sala gestionada, llamada de voz en tiempo real,
STT de transmisión de llamadas de voz, Google Meet en tiempo real y
pulsar para hablar nativo. El Gateway anuncia un único canal de eventos de Talk
en directo en `hello-ok.features.events`: `talk.event`.

El código nuevo no debe llamar directamente a `createTalkEventSequencer(...)`, salvo que
implemente un adaptador de bajo nivel o un fixture de prueba. Utilice el
controlador compartido para que los eventos limitados a un turno no puedan
emitirse sin un identificador de turno, las llamadas obsoletas
`turnEnd` / `turnCancel` no puedan borrar un turno activo más
reciente y los eventos del ciclo de vida del audio de salida permanezcan
coherentes entre la telefonía, las reuniones, el relay del navegador, la
transferencia de sala gestionada y los clientes nativos de Talk.

La forma de la API pública:

```typescript
// API de sesiones de Talk gestionada por el Gateway.
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// API de sesiones de proveedor gestionada por el cliente.
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

Las sesiones de WebRTC/websocket de proveedor gestionadas por el navegador
utilizan `talk.client.create`, porque el navegador gestiona la negociación con el
proveedor y el transporte multimedia, mientras que el Gateway gestiona las
credenciales, las instrucciones y la política de herramientas.
`talk.session.*` es la superficie común gestionada por el Gateway para las
sesiones en tiempo real mediante relay del Gateway, la transcripción mediante
relay del Gateway y las sesiones nativas de STT/TTS de salas gestionadas.

Las configuraciones heredadas que sitúan selectores de tiempo real junto a
`talk.provider` / `talk.providers` deben repararse con
`openclaw doctor --fix`; Talk en tiempo de ejecución no reinterpreta la configuración
del proveedor de voz/TTS como configuración del proveedor en tiempo real.

Las combinaciones compatibles de `talk.session.create` son intencionadamente
reducidas:

| Modo            | Transporte       | Cerebro           | Responsable              | Notas                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | Audio bidireccional del proveedor transmitido mediante el Gateway; las llamadas a herramientas se encaminan mediante la herramienta de consulta con el agente.           |
| `transcription` | `gateway-relay` | `none`          | Gateway            | Solo STT de transmisión; los llamantes envían audio de entrada y reciben eventos de transcripción.                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | Sala nativa/del cliente | Salas de estilo pulsar para hablar y walkie-talkie en las que el cliente gestiona la captura/reproducción y el Gateway gestiona el estado del turno. |
| `stt-tts`       | `managed-room`  | `direct-tools`  | Sala nativa/del cliente | Modo de sala exclusivo para administradores destinado a superficies propias de confianza que ejecutan directamente acciones de herramientas del Gateway.                  |

Mapa de métodos para quienes migren desde las familias anteriores
`talk.realtime.*` / `talk.transcription.*` / `talk.handoff.*` (todas
eliminadas):

| Anterior                         | Nuevo                                                    |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` o `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

El vocabulario de control unificado también es deliberadamente reducido:

| Método                          | Se aplica a                                              | Contrato                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`, `transcription/gateway-relay` | Anexa un fragmento de audio PCM en base64 a la sesión del proveedor gestionada por la misma conexión del Gateway.                                                                                                                             |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | Inicia un turno de usuario en una sala gestionada.                                                                                                                                                                                           |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | Finaliza el turno activo tras validar que no sea obsoleto.                                                                                                                                                                          |
| `talk.session.cancelTurn`       | todas las sesiones gestionadas por el Gateway                              | Cancela el trabajo activo de captura/proveedor/agente/TTS de un turno.                                                                                                                                                                 |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | Detiene la salida de audio del asistente sin finalizar necesariamente el turno del usuario.                                                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | Completa una llamada a una herramienta del proveedor después de cualquier finalización asíncrona expuesta por su puente; pasa `options.willContinue` para la salida provisional o, cuando sea compatible, `options.suppressResponse` para evitar otra respuesta del asistente. |
| `talk.session.steer`            | sesiones de Talk respaldadas por agentes                              | Envía el control hablado `status`, `steer`, `cancel` o `followup` a la ejecución integrada activa resuelta a partir de la sesión de Talk.                                                                                                 |
| `talk.session.close`            | todas las sesiones unificadas                                    | Detiene las sesiones de relay o revoca el estado de la sala gestionada y, después, olvida el identificador de sesión unificado.                                                                                                                                     |

No introduzca casos especiales de proveedores o plataformas en el núcleo para
que esto funcione. El núcleo gestiona la semántica de las sesiones de Talk. Los
plugins de proveedor gestionan la configuración de las sesiones del proveedor.
Las llamadas de voz y Google Meet gestionan los adaptadores de
telefonía/reuniones. Las aplicaciones web y nativas gestionan la experiencia de
usuario de captura/reproducción del dispositivo.

## Cronología de eliminación

| Cuándo                                        | Qué sucede                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Ahora**                                     | Las superficies obsoletas capaces de generar advertencias emiten advertencias en tiempo de ejecución; las protecciones del repositorio rechazan las importaciones obsoletas del SDK desde el núcleo y los plugins incluidos. |
| **Decisión del propietario pendiente**                  | Los registros sin fecha permanecen obsoletos y no pueden eliminarse hasta que su propietario publique una fecha `removeAfter`.                          |
| **Fecha `removeAfter` de cada registro de compatibilidad** | Esa superficie específica puede eliminarse; `pnpm plugins:boundary-report --fail-on-eligible-compat` hace que la CI falle una vez pasada la fecha.    |
| **Próxima versión principal**                      | Las superficies con fecha solo pueden eliminarse después de su fecha `removeAfter`; los registros sin fecha siguen requiriendo la aprobación del propietario y una fecha publicada.   |

Las subrutas públicas restantes del SDK que aparecen a continuación tienen periodos de eliminación respaldados por el registro.
Las filas del 30 de julio se eliminaron tras su revisión anticipada autorizada por los mantenedores:
se eliminaron las subrutas no utilizadas y los alias de compatibilidad anteriores, y
los módulos exclusivos de los plugins incluidos se degradaron a asignaciones de compilación locales privadas.

| `removeAfter` | Nivel                               | Subrutas del SDK                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | Obsolescencias de compatibilidad anteriores | `agent-config-primitives`, `channel-logging`, `channel-secret-runtime`, `channel-streaming`, `group-access`, `inbound-reply-dispatch`, `matrix`, `text-runtime`, `zod`              |
| `2026-09-01`  | Obsolescencias de compatibilidad anteriores | `channel-lifecycle`, `channel-message`, `channel-reply-pipeline`, `config-runtime`, `infra-runtime`                                                                                 |
| `2026-10-01`  | Proyección multimedia heredada            | `agent-media-payload`, además de los campos `MsgContext Media*` que no son subrutas, los generadores de cargas multimedia entrantes de canales, `buildMediaPayload`, los alias multimedia de hooks y las plantillas `{{Media*}}` |

Todos los plugins del núcleo ya han migrado. Los plugins externos deben migrar
antes de la próxima versión principal. Ejecute `pnpm plugins:boundary-report` para ver qué
registros de compatibilidad tienen las fechas más próximas para las superficies que utiliza su plugin.

## Supresión temporal de las advertencias

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Esta es una vía de escape temporal, no una solución permanente.

## Contenido relacionado

- [Primeros pasos](/es/plugins/building-plugins) - cree su primer plugin
- [Descripción general del SDK](/es/plugins/sdk-overview) - referencia completa de importación de subrutas
- [Plugins de canal](/es/plugins/sdk-channel-plugins) - creación de plugins de canal
- [Plugins de proveedor](/es/plugins/sdk-provider-plugins) - creación de plugins de proveedor
- [Aspectos internos de los plugins](/es/plugins/architecture) - análisis detallado de la arquitectura
- [Manifiesto del plugin](/es/plugins/manifest) - referencia del esquema del manifiesto
