---
read_when:
    - Necesita la firma de tipo exacta de defineToolPlugin, definePluginEntry o defineChannelPluginEntry
    - Quieres entender el modo de registro (completo, configuración o metadatos de la CLI)
    - Está consultando las opciones de punto de entrada
sidebarTitle: Entry Points
summary: Referencia de defineToolPlugin, definePluginEntry, defineChannelPluginEntry y defineSetupPluginEntry
title: Puntos de entrada de Plugin
x-i18n:
    generated_at: "2026-07-26T05:51:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e64fe1d65531fea8f266aa23b73064daf2ed2c5c43af8bb08ea57e347fe566f4
    source_path: plugins/sdk-entrypoints.md
    workflow: 16
---

Cada plugin exporta un objeto de entrada predeterminado. El SDK proporciona una función auxiliar para
cada forma de entrada: `defineToolPlugin`, `definePluginEntry`,
`defineChannelPluginEntry`, `defineSetupPluginEntry`.

<Tip>
  **¿Busca una guía paso a paso?** Consulte [Plugins de herramientas](/es/plugins/tool-plugins),
  [Plugins de canales](/es/plugins/sdk-channel-plugins) o
  [Plugins de proveedores](/es/plugins/sdk-provider-plugins) para obtener guías paso a paso.
</Tip>

## Entradas de paquetes

Los plugins instalados hacen que los campos `package.json` `openclaw` apunten tanto a las entradas
de origen como a las compiladas:

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

- `extensions` y `setupEntry` son entradas de origen, utilizadas para el desarrollo en espacios de trabajo y
  checkouts de git.
- `runtimeExtensions` y `runtimeSetupEntry` son las opciones preferidas para los paquetes
  instalados: permiten que los paquetes npm omitan la compilación de TypeScript en tiempo de ejecución.
- `runtimeExtensions`, cuando está presente, debe coincidir con `extensions` en la longitud del arreglo
  (las entradas se emparejan por posición). `runtimeSetupEntry` requiere `setupEntry`.
- Si se declara un artefacto `runtimeExtensions`/`runtimeSetupEntry` pero
  no está presente, la instalación o detección falla con un error de empaquetado; OpenClaw no
  recurre silenciosamente al código fuente. La alternativa de código fuente (descrita más abajo) solo se aplica cuando no
  se declara ninguna entrada de tiempo de ejecución.
- Si un paquete instalado declara únicamente una entrada de código fuente TypeScript, OpenClaw
  busca una entrada compilada equivalente `dist/*.js` (o `.mjs`/`.cjs`) y la utiliza;
  de lo contrario, recurre al código fuente TypeScript.
- Todas las rutas de entrada deben permanecer dentro del directorio del paquete del plugin. Las entradas de tiempo de
  ejecución y las entradas equivalentes de JS compilado inferidas no hacen válida una ruta de código fuente `extensions` o
  `setupEntry` que se escape del directorio.

## `defineToolPlugin`

**Importación:** `openclaw/plugin-sdk/tool-plugin`

Para plugins que solo añaden herramientas de agente. Mantiene el código fuente reducido, infiere los tipos de configuración
y de parámetros de herramientas a partir de esquemas TypeBox, envuelve los valores de retorno simples en
el formato de resultados de herramientas de OpenClaw y expone metadatos estáticos que
`openclaw plugins build` escribe en el manifiesto del plugin (`contracts.tools`,
`configSchema`).

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quotes.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "quote",
      label: "Quote",
      description: "Fetch a quote.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          hasKey: Type.Boolean(),
        },
        { additionalProperties: false },
      ),
      execute: async ({ symbol }, config) => ({ symbol, hasKey: Boolean(config.apiKey) }),
    }),
  ],
});
```

- `configSchema` es opcional; si se omite, se utiliza un esquema estricto de objeto vacío
  (el manifiesto generado sigue incluyendo `configSchema`).
- `execute` devuelve una cadena simple o un valor serializable como JSON; la función auxiliar
  lo envuelve como un resultado de herramienta de texto con `details` establecido en el valor de retorno
  original (sin convertir en cadena).
- `outputSchema` describe opcionalmente ese valor `details` original para el modo de código
  y la búsqueda de herramientas. Las llamadas al catálogo rechazan un esquema no válido antes de la ejecución
  y validan el valor final antes de devolverlo.
- Para resultados de herramientas personalizados, `openclaw/plugin-sdk/tool-results` exporta
  `textResult` y `jsonResult`.
- Los nombres de las herramientas son estáticos, por lo que `openclaw plugins build` deriva
  `contracts.tools` de las herramientas declaradas sin duplicar manualmente los nombres.
- La carga en tiempo de ejecución sigue siendo estricta: los plugins instalados aún necesitan
  `openclaw.plugin.json` y `package.json` `openclaw.extensions`. OpenClaw
  nunca ejecuta código de plugins para inferir datos faltantes del manifiesto.

## `definePluginEntry`

**Importación:** `openclaw/plugin-sdk/plugin-entry`

Para plugins de proveedores, plugins de herramientas avanzadas, plugins de hooks y cualquier elemento que
**no** sea un canal de mensajería.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({/* ... */});
    api.registerTool({/* ... */});
  },
});
```

| Campo                     | Tipo                                                             | Obligatorio | Valor predeterminado             |
| ------------------------- | ---------------------------------------------------------------- | ----------- | -------------------------------- |
| `id`                      | `string`                                                         | Sí          | -                                |
| `name`                    | `string`                                                         | Sí          | -                                |
| `description`             | `string`                                                         | Sí          | -                                |
| `kind`                    | `string` (obsoleto, consulte más abajo)                          | No          | -                                |
| `configSchema`            | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | No          | Esquema de objeto vacío          |
| `reload`                  | `OpenClawPluginReloadRegistration`                               | No          | -                                |
| `nodeHostCommands`        | `OpenClawPluginNodeHostCommand[]`                                | No          | -                                |
| `securityAuditCollectors` | `OpenClawPluginSecurityAuditCollector[]`                         | No          | -                                |
| `register`                | `(api: OpenClawPluginApi) => void`                               | Sí          | -                                |

- `id` debe coincidir con el manifiesto `openclaw.plugin.json`.
- Los catálogos de sesiones externas utilizan
  `openclaw/plugin-sdk/session-catalog` y
  `api.registerSessionCatalog({ id, label, list, read, continueSession?, archive? })`.
  El núcleo es propietario de los métodos del Gateway `sessions.catalog.*`; los proveedores devuelven proyecciones del host,
  de la sesión y de la transcripción normalizada sin registrar RPC. Un
  proveedor de listas debe invocar la función de devolución opcional `onHost(host)` a medida que se completa cada host;
  el arreglo de hosts devuelto sigue siendo obligatorio como instantánea final de
  compatibilidad.
- `kind` está obsoleto: declare un espacio exclusivo (`"memory"` o
  `"context-engine"`) en el campo `kind` del manifiesto `openclaw.plugin.json`
  en su lugar. La entrada de tiempo de ejecución `kind` permanece únicamente como alternativa de compatibilidad para
  plugins antiguos.
- `configSchema` puede ser una función para su evaluación diferida. OpenClaw resuelve y
  memoriza el esquema durante el primer acceso, por lo que los generadores de esquemas costosos solo se ejecutan
  una vez.
- Un descriptor `nodeHostCommands` puede definir `isAvailable({ config, env })`.
  Devolver `false` omite ese comando y su capacidad de la declaración del Gateway
  del nodo sin interfaz gráfica. OpenClaw lo evalúa con respecto a la configuración de inicio local
  del nodo; los controladores de comandos deben seguir validando la disponibilidad al
  invocarse.

## `defineChannelPluginEntry`

**Importación:** `openclaw/plugin-sdk/channel-core`

Envuelve `definePluginEntry` con conexiones específicas del canal: invoca automáticamente
`api.registerChannel({ plugin })`, expone una interfaz opcional de metadatos de la CLI
para la ayuda raíz y condiciona `registerFull` al modo de registro.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| Campo                 | Tipo                                                             | Obligatorio | Valor predeterminado    |
| --------------------- | ---------------------------------------------------------------- | ----------- | ----------------------- |
| `id`                  | `string`                                                         | Sí          | -                       |
| `name`                | `string`                                                         | Sí          | -                       |
| `description`         | `string`                                                         | Sí          | -                       |
| `plugin`              | `ChannelPlugin`                                                  | Sí          | -                       |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | No          | Esquema de objeto vacío |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | No          | -                       |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | No          | -                       |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | No          | -                       |

Las funciones de devolución se ejecutan según el modo de registro (tabla completa en
[Modo de registro](#registration-mode)):

- `setRuntime` se ejecuta en todos los modos excepto `"cli-metadata"` y
  `"tool-discovery"`. Almacene aquí la referencia al entorno de ejecución, normalmente mediante
  `createPluginRuntimeStore`.
- `registerCliMetadata` se ejecuta para `"cli-metadata"`, `"discovery"` y
  `"full"`. Utilícelo como ubicación canónica para los descriptores de la CLI propiedad del canal,
  de modo que la ayuda raíz no provoque la activación, las instantáneas de detección incluyan metadatos
  estáticos de comandos y el registro normal de la CLI siga siendo compatible con las cargas
  completas de plugins.
- `registerFull` se ejecuta únicamente para `"full"` y `"tool-discovery"`. Para
  `"tool-discovery"`, se ejecuta _en lugar del_ registro del canal: OpenClaw
  omite por completo `registerChannel`/`setRuntime` e invoca únicamente
  `registerFull`, por lo que cualquier registro de proveedores o herramientas que el canal necesite para
  la detección o ejecución independiente de herramientas debe residir allí, no detrás de la configuración
  normal del canal.
- El registro de detección no provoca la activación, pero sí puede realizar importaciones: OpenClaw puede
  evaluar la entrada del plugin de confianza y el módulo del plugin de canal para crear la
  instantánea. Mantenga las importaciones de nivel superior libres de efectos secundarios y coloque los sockets,
  clientes, procesos de trabajo y servicios detrás de rutas exclusivas de `"full"`.
- Al igual que `definePluginEntry`, `configSchema` puede ser una fábrica diferida; OpenClaw
  memoriza el esquema resuelto durante el primer acceso.

Registro de la CLI:

- Use `api.registerCli(..., { descriptors: [...] })` para los comandos raíz de la
  CLI propiedad del plugin que se quiera cargar de forma diferida sin que desaparezcan del árbol
  de análisis de la CLI raíz. Los nombres de los descriptores deben contener letras, números, guiones y
  guiones bajos, y comenzar con una letra o un número; OpenClaw rechaza otras
  formas y elimina las secuencias de control del terminal de las descripciones antes de
  mostrar la ayuda. Abarque cada raíz de comando de nivel superior que exponga el registrador.
  `commands` por sí solo permanece en la ruta de compatibilidad de carga inmediata.
- Use `api.registerNodeCliFeature(...)` para los comandos de funciones de nodos emparejados, de modo que
  queden bajo `openclaw nodes` (equivalente a
  `registerCli(registrar, { parentPath: ["nodes"], ... })`).
- Para otros comandos de plugins anidados, añada `parentPath` y registre los comandos
  en el objeto `program` pasado al registrador; OpenClaw lo resuelve como
  el comando principal antes de llamar al plugin.
- Para los plugins de canal, registre los descriptores de la CLI desde `registerCliMetadata`
  y mantenga `registerFull` centrado únicamente en el trabajo de tiempo de ejecución.
- Si `registerFull` también registra métodos RPC del Gateway, manténgalos bajo un
  prefijo específico del plugin. Los espacios de nombres administrativos reservados del núcleo (`config.*`,
  `exec.approvals.*`, `wizard.*`, `update.*`) siempre se convierten en
  `operator.admin`.

## `defineSetupPluginEntry`

**Importación:** `openclaw/plugin-sdk/channel-core`

Para el archivo ligero `setup-entry.ts`. Devuelve únicamente `{ plugin }`, sin
conexiones de tiempo de ejecución ni de la CLI.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

OpenClaw carga esto en lugar de la entrada completa cuando un canal está deshabilitado,
no está configurado o cuando la carga diferida está habilitada. Consulte
[Configuración](/es/plugins/sdk-setup#setup-entry) para saber cuándo es importante.

Combine `defineSetupPluginEntry(...)` con las familias específicas de asistentes de configuración:

| Importación                         | Uso                                                                                                                                                                                |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw/plugin-sdk/setup-runtime` | Asistentes de configuración seguros para el tiempo de ejecución: `createSetupTranslator`, adaptadores de parches de configuración seguros para la importación, salida de notas de búsqueda, `promptResolvedAllowFrom`, `splitSetupEntries`, proxies de configuración delegados |
| `openclaw/plugin-sdk/channel-setup` | Superficies de configuración de instalación opcional                                                                                                                               |
| `openclaw/plugin-sdk/setup-tools`   | Asistentes de CLI, archivos y documentación para configuración e instalación                                                                                                       |

Mantenga los SDK pesados, el registro de la CLI y los servicios de tiempo de ejecución de larga duración en la
entrada completa.

Los canales incluidos en el espacio de trabajo que dividen las superficies de configuración y tiempo de ejecución pueden usar
`defineBundledChannelSetupEntry(...)` de
`openclaw/plugin-sdk/channel-entry-contract` en su lugar. Permite que la entrada de
configuración conserve las exportaciones de plugins y secretos seguras para la configuración, a la vez que expone un definidor de
tiempo de ejecución:

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
  registerSetupRuntime(api) {
    api.registerHttpRoute({
      path: "/my-channel/events",
      auth: "plugin",
      handler: async (req, res) => {
        /* ruta segura para la configuración */
      },
    });
  },
});
```

Úselo solo cuando un flujo de configuración necesite realmente un definidor ligero de tiempo de ejecución o
una superficie del Gateway segura para la configuración antes de que se cargue la entrada completa del canal.
`registerSetupRuntime` se ejecuta solo para las cargas de `"setup-runtime"`; limítelo
a rutas o métodos exclusivos de configuración que deban existir antes de la activación
completa diferida.

## Modo de registro

`api.registrationMode` indica al plugin cómo se cargó:

| Modo               | Cuándo                                             | Qué registrar                                                                                                                     |
| ------------------ | -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `"full"`           | Inicio normal del Gateway                          | Todo                                                                                                                              |
| `"discovery"`      | Detección de capacidades de solo lectura           | Registro del canal más descriptores estáticos de la CLI; el código de entrada puede cargarse, pero omita sockets, procesos de trabajo, clientes y servicios |
| `"tool-discovery"` | Carga limitada para enumerar o ejecutar herramientas de plugins específicos | Solo registro de capacidades/herramientas; sin activación del canal                                                               |
| `"setup-only"`     | Canal deshabilitado/no configurado                 | Solo registro del canal                                                                                                           |
| `"setup-runtime"`  | Flujo de configuración con tiempo de ejecución disponible | Registro del canal más únicamente el tiempo de ejecución ligero necesario antes de que se cargue la entrada completa              |
| `"cli-metadata"`   | Ayuda raíz / captura de metadatos de la CLI        | Solo descriptores de la CLI                                                                                                       |

`defineChannelPluginEntry` gestiona esta división automáticamente. Si se usa
`definePluginEntry` directamente para un canal, compruebe el modo y recuerde que
`"tool-discovery"` omite el registro del canal:

```typescript
register(api) {
  if (
    api.registrationMode === "cli-metadata" ||
    api.registrationMode === "discovery" ||
    api.registrationMode === "full"
  ) {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  if (api.registrationMode === "tool-discovery") {
    // Registre superficies exclusivas de capacidades (proveedores/herramientas), sin canal.
    return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // Registros pesados exclusivos del tiempo de ejecución
  api.registerService(/* ... */);
}
```

Los servicios de larga duración pueden emitir pequeños eventos de invalidación o del ciclo de vida mediante
su contexto de servicio:

```typescript
api.registerService({
  id: "index-events",
  start(ctx) {
    ctx.gatewayEvents?.emit("changed", { revision: 1 }, { scope: "operator.read" });
  },
});
```

OpenClaw aplica el espacio de nombres `plugin.<plugin-id>.changed`. Los nombres de los eventos constan de un
segmento en minúsculas, las cargas útiles deben ser JSON acotado y el ámbito debe ser
`operator.read`, `operator.write` o `operator.admin`. El emisor existe únicamente
durante la vida útil del servicio y se revoca tras la detención o un inicio fallido. Prefiera
cargas útiles de versión o invalidación en lugar de registros completos para que los clientes autorizados vuelvan a leer
el estado canónico mediante los métodos de Gateway con ámbito del plugin.

El modo de detección crea una instantánea del registro sin activación. Aun así, puede
evaluar la entrada del plugin y el objeto del plugin de canal para que OpenClaw pueda
registrar las capacidades del canal y los descriptores estáticos de la CLI. Trate la evaluación
del módulo durante la detección como fiable, pero ligera: sin clientes de red,
subprocesos, escuchas, conexiones de bases de datos, procesos de trabajo en segundo plano,
lecturas de credenciales ni otros efectos secundarios activos del tiempo de ejecución en el nivel superior.

Considere `"setup-runtime"` como la ventana en la que las superficies de inicio exclusivas de la configuración deben
existir sin volver a entrar en el tiempo de ejecución completo del canal incluido. Las opciones adecuadas son
el registro del canal, las rutas HTTP seguras para la configuración, los métodos del Gateway seguros para la configuración
y los asistentes de configuración delegados. Los servicios pesados en segundo plano, los registradores de la CLI y
las inicializaciones de SDK de proveedores/clientes siguen perteneciendo a `"full"`.

## Formas de plugins

OpenClaw clasifica los plugins cargados según su comportamiento de registro:

| Forma                 | Descripción                                             |
| --------------------- | ------------------------------------------------------- |
| **plain-capability**  | Un tipo de capacidad (p. ej., solo proveedor)           |
| **hybrid-capability** | Varios tipos de capacidades (p. ej., proveedor + voz)   |
| **hook-only**         | Solo enlaces, sin capacidades                           |
| **non-capability**    | Herramientas/comandos/servicios, pero sin capacidades   |

Use `openclaw plugins inspect <id>` para ver la forma de un plugin.

## Contenido relacionado

- [Descripción general del SDK](/es/plugins/sdk-overview) - referencia de la API de registro y las subrutas
- [Asistentes de tiempo de ejecución](/es/plugins/sdk-runtime) - `api.runtime` y `createPluginRuntimeStore`
- [Configuración](/es/plugins/sdk-setup) - manifiesto, entrada de configuración y carga diferida
- [Plugins de canal](/es/plugins/sdk-channel-plugins) - creación del objeto `ChannelPlugin`
- [Plugins de proveedores](/es/plugins/sdk-provider-plugins) - registro y enlaces de proveedores
