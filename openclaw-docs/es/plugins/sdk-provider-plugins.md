---
read_when:
    - Está creando un nuevo plugin de proveedor de modelos
    - Quiere añadir un proxy compatible con OpenAI o un LLM personalizado a OpenClaw
    - Debe comprender la autenticación de proveedores, los catálogos y los hooks de tiempo de ejecución
sidebarTitle: Provider plugins
summary: Guía paso a paso para crear un plugin de proveedor de modelos para OpenClaw
title: Creación de plugins de proveedores
x-i18n:
    generated_at: "2026-07-26T04:53:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

Cree un plugin de proveedor para añadir un proveedor de modelos (LLM) a OpenClaw: un
catálogo de modelos, autenticación mediante clave de API y resolución dinámica de modelos.

<Info>
  ¿Es la primera vez que usa plugins de OpenClaw? Lea primero [Primeros pasos](/es/plugins/building-plugins)
  para conocer la estructura del paquete y la configuración del manifiesto.
</Info>

<Tip>
  Los plugins de proveedor añaden modelos al bucle de inferencia normal de OpenClaw. Si el
  modelo debe ejecutarse mediante un daemon de agente nativo que gestiona hilos, Compaction
  o eventos de herramientas, combine el proveedor con un [entorno de
  agente](/es/plugins/sdk-agent-harness) en lugar de incluir los detalles del protocolo del daemon
  en el núcleo.
</Tip>

## Guía paso a paso

<Steps>
  <Step title="Paquete y manifiesto">
    ### Paso 1: Paquete y manifiesto

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Proveedor de modelos Acme AI",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Clave de API de Acme AI",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Clave de API de Acme AI"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars` permite que OpenClaw detecte credenciales sin
    cargar el entorno de ejecución del plugin. Añada `providerAuthAliases` cuando una variante del proveedor
    deba reutilizar la autenticación del identificador de otro proveedor. `modelSupport` es
    opcional y permite que OpenClaw cargue automáticamente el plugin de proveedor a partir de identificadores
    abreviados de modelos como `acme-large` antes de que existan los enlaces del entorno de ejecución. `openclaw.compat`
    y `openclaw.build` en `package.json` son obligatorios para publicar en ClawHub
    (`openclaw.compat.pluginApi` y `openclaw.build.openclawVersion`
    son los dos campos obligatorios; `minGatewayVersion` utiliza
    `openclaw.install.minHostVersion` de forma predeterminada cuando se omite).

  </Step>

  <Step title="Registrar el proveedor">
    Un proveedor de texto mínimo necesita `id`, `label`, `auth` y `catalog`.
    `catalog` es el enlace de entorno de ejecución/configuración propiedad del proveedor; puede llamar a
    las API activas del proveedor y devuelve entradas `models.providers`.

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` es la nueva superficie de catálogo del plano de control
    para las interfaces de lista, ayuda y selección, que abarca filas `text`, `voice`, `image_generation`,
    `video_generation` y `music_generation`. Mantenga las llamadas a los endpoints
    del proveedor y la asignación de respuestas en el plugin; OpenClaw gestiona la forma
    compartida de las filas, las etiquetas de origen y la representación de la ayuda.

    Con esto ya dispone de un proveedor funcional. Ahora los usuarios pueden ejecutar
    `openclaw onboard --acme-ai-api-key <key>` y seleccionar
    `acme-ai/acme-large` como modelo.

    ### Detección de modelos en tiempo real

    Si el proveedor ofrece una API `/models` compatible con OpenAI, habilite
    la detección compartida en el asistente de proveedor único:

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true` es un contrato público del SDK de Plugin con los siguientes
    comportamientos:

    | Área | Contrato |
    | --- | --- |
    | Credenciales | La detección utiliza la credencial resuelta del proveedor del catálogo y da preferencia a `discoveryApiKey` cuando la autenticación proporciona una. Los marcadores de referencia a secretos nunca se envían como tokens. La solicitud predeterminada utiliza `Authorization: Bearer <token>`; use `buildRequestHeaders` para otro esquema de autenticación del proveedor. |
    | Endpoint | La URL predeterminada es `models` con respecto a la `baseUrl` efectiva del proveedor, incluida una sustitución del operador cuando `allowExplicitBaseUrl` está habilitado. Use `endpointPath` para otra ruta relativa. Use `endpointUrl: { url, requireBaseUrl }` solo para una URL fija del proveedor; la detección se omite a menos que la URL base efectiva siga siendo igual a `requireBaseUrl`, para que la credencial de un proxy personalizado no se envíe al proveedor. |
    | Límites de red | Las solicitudes usan la protección contra SSRF de OpenClaw, un único plazo de espera de 5 segundos para toda la paginación, un límite de respuesta de 4 MiB por página y un límite de 50 páginas. Se rechazan los enlaces de paginación entre orígenes distintos; las credenciales se eliminan después de una redirección a otro origen. |
    | Caché | Los catálogos correctos y no vacíos se almacenan en caché durante 60 segundos por proveedor, endpoint y credencial resuelta. Los resultados vacíos o no utilizables no se almacenan en caché. |
    | Filtrado | Los identificadores exactos obtenidos en tiempo real conservan sus metadatos estáticos de confianza. Las filas nuevas se proyectan de forma conservadora como modelos de texto/chat. Se excluyen las filas deshabilitadas, archivadas, obsoletas, explícitamente ajenas al chat, de embeddings, reclasificación, moderación, voz, solo imágenes y solo vídeo. Use `readRows` únicamente para seleccionar filas de un contenedor de respuesta no estándar; la semántica de modelos específica del proveedor debe permanecer en un catálogo personalizado. |
    | Fallos | La detección en tiempo real es orientativa. Los fallos de autenticación, red, tiempo de espera, paginación, análisis, catálogo vacío y filtrado devuelven la base estática propiedad del proveedor en lugar de eliminarlo. |

    Para un endpoint de lista que no use Bearer o que no sea estándar, pase opciones en lugar de
    `true`:

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    No use `endpointUrl` como host alternativo incondicional. Su
    comprobación `requireBaseUrl` constituye el límite de aislamiento de credenciales para los proveedores
    cuyo host de lista de modelos difiere del host de inferencia.

    Si el proveedor necesita una semántica de modelos personalizada en lugar de la proyección
    conservadora compatible con OpenAI, mantenga esa proyección en el plugin y use
    `openclaw/plugin-sdk/provider-catalog-live-runtime` para el ciclo de vida compartido de las
    solicitudes. El asistente proporciona solicitudes HTTP protegidas, encabezados de autenticación del proveedor,
    errores HTTP estructurados, almacenamiento en caché con TTL y comportamiento de reserva estático sin
    incluir políticas específicas del proveedor en el núcleo de OpenClaw.

    Use `buildLiveModelProviderConfig` cuando la API activa solo indique qué
    filas del catálogo estático propiedad del proveedor están disponibles actualmente:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    Use `getCachedLiveProviderModelRows` cuando la API del proveedor devuelva metadatos más completos
    y el plugin necesite proyectar por sí mismo las filas en definiciones de modelos de
    OpenClaw:

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` debe permanecer condicionado por la autenticación y devolver `null` cuando no
    haya credenciales utilizables disponibles. Mantenga un `staticRun` sin conexión o un recurso alternativo estático para que la configuración, la documentación,
    las pruebas y las superficies de selección no dependan del acceso en vivo a la red. Use un TTL
    adecuado para la vigencia de la lista de modelos, evite sondear el sistema de archivos durante las solicitudes
    y proporcione un `readRows` / `readModelId` específico del proveedor solo cuando la
    respuesta del servicio de origen no tenga una estructura `{ data: [{ id, object }] }`
    compatible con OpenAI.

    Si el proveedor de origen usa tokens de control distintos de los de OpenClaw, añada una
    pequeña transformación de texto bidireccional en lugar de sustituir la ruta de transmisión:

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` reescribe el prompt final del sistema y el contenido de los mensajes de texto antes
    del transporte. `output` reescribe los deltas de texto del asistente y el texto final antes de que
    OpenClaw analice sus propios marcadores de control o realice la entrega al canal.

    Para proveedores incluidos que solo registran un proveedor de texto con autenticación
    mediante clave de API y un único entorno de ejecución respaldado por catálogo, prefiera el asistente más específico
    `defineSingleProviderPluginEntry(...)`:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Proveedor de modelos Acme AI",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Clave de API de Acme AI",
            hint: "Clave de API del panel de Acme AI",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Introduzca su clave de API de Acme AI",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` es la ruta del catálogo en vivo que se usa cuando OpenClaw puede resolver la
    autenticación real del proveedor. Puede realizar un descubrimiento específico del proveedor. Use
    `buildStaticProvider` solo para filas sin conexión que sea seguro mostrar antes de configurar
    la autenticación; no debe requerir credenciales ni realizar solicitudes de red.
    Actualmente, la visualización `models list --all` de OpenClaw ejecuta catálogos estáticos
    solo para plugins de proveedores incluidos, con una configuración vacía, un entorno vacío y sin
    rutas de agente ni de espacio de trabajo.

    Si el flujo de autenticación también necesita modificar `models.providers.*`, alias y
    el modelo predeterminado del agente durante la incorporación, use los asistentes de ajustes preestablecidos de
    `openclaw/plugin-sdk/provider-onboard`. Los asistentes más específicos son
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` y
    `createModelCatalogPresetAppliers(...)`.

    Cuando el punto de conexión nativo de un proveedor admita bloques de uso transmitidos en el
    transporte `openai-completions` normal, prefiera los asistentes de catálogo compartidos de
    `openclaw/plugin-sdk/provider-catalog-shared` en lugar de codificar de forma fija
    comprobaciones del identificador del proveedor. `supportsNativeStreamingUsageCompat(...)` y
    `applyProviderNativeStreamingUsageCompat(...)` detectan la compatibilidad mediante el
    mapa de capacidades del punto de conexión, por lo que los puntos de conexión nativos del estilo Moonshot/DashScope siguen
    habilitándose incluso cuando un plugin usa un identificador de proveedor personalizado.

    Los ejemplos de descubrimiento en vivo anteriores abarcan las API de proveedores del estilo `/models`. Mantenga
    ese descubrimiento dentro de `catalog.run`, condicionado a una autenticación utilizable, y mantenga
    `staticRun` sin acceso a la red para generar catálogos sin conexión.

  </Step>

  <Step title="Añadir resolución dinámica de modelos">
    Si el proveedor acepta identificadores de modelo arbitrarios (como un proxy o enrutador),
    añada `resolveDynamicModel`:

    ```typescript
    api.registerProvider({
      // ... identificador, etiqueta, autenticación y catálogo anteriores

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    Si la resolución requiere una llamada de red, use `prepareDynamicModel` para el
    calentamiento asíncrono; `resolveDynamicModel` vuelve a ejecutarse cuando este finaliza.

  </Step>

  <Step title="Añadir hooks de entorno de ejecución (según sea necesario)">
    La mayoría de los proveedores solo necesitan `catalog` + `resolveDynamicModel`. Añada hooks
    gradualmente a medida que el proveedor los requiera.

    Los generadores de asistentes compartidos ahora abarcan las familias más comunes de compatibilidad
    con la reproducción y las herramientas, por lo que normalmente los plugins no necesitan conectar manualmente cada hook uno por uno:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    Familias de reproducción disponibles actualmente:

    | Familia | Qué conecta | Ejemplos incluidos |
    | --- | --- | --- |
    | `openai-compatible` | Política de reproducción compartida al estilo OpenAI para transportes compatibles con OpenAI, incluida la depuración de identificadores de llamadas a herramientas, las correcciones de orden para que el asistente aparezca primero y la validación genérica de turnos de Gemini cuando el transporte la necesita | `moonshot`, `ollama`, `xai`, `zai` |
    | `anthropic-by-model` | Política de reproducción compatible con Claude elegida por `modelId`, para que los transportes de mensajes de Anthropic solo reciban la limpieza de bloques de pensamiento específica de Claude cuando el modelo resuelto sea realmente un identificador de Claude | `amazon-bedrock` |
    | `native-anthropic-by-model` | La misma política de Claude según el modelo que `anthropic-by-model`, además de la depuración de identificadores de llamadas a herramientas y la conservación de identificadores nativos de uso de herramientas de Anthropic para los transportes que deben mantener los identificadores nativos del proveedor | `anthropic-vertex`, `clawrouter` |
    | `google-gemini` | Política de reproducción nativa de Gemini más depuración de la reproducción de arranque. La familia compartida mantiene Gemini CLI con salida de texto en el razonamiento etiquetado; el proveedor directo `google` sustituye `resolveReasoningOutputMode` por `native` porque el pensamiento de la API de Gemini llega como partes de pensamiento nativas. | `google`, `google-gemini-cli` |
    | `passthrough-gemini` | Depuración de firmas de pensamiento de Gemini para modelos de Gemini que se ejecutan mediante transportes proxy compatibles con OpenAI; no habilita la validación de reproducción nativa de Gemini ni las reescrituras de arranque | `openrouter`, `kilocode`, `opencode`, `opencode-go` |
    | `hybrid-anthropic-openai` | Política híbrida para proveedores que combinan superficies de modelos de mensajes de Anthropic y compatibles con OpenAI en un mismo plugin; la eliminación opcional de bloques de pensamiento exclusiva de Claude permanece limitada al lado de Anthropic | `minimax` |

    Familias de transmisión disponibles actualmente:

    | Familia | Qué integra | Ejemplos incluidos |
    | --- | --- | --- |
    | `google-thinking` | Normalización de la carga útil de pensamiento de Gemini en la ruta de flujo compartida | `google`, `google-gemini-cli` |
    | `kilocode-thinking` | Contenedor de razonamiento de Kilo en la ruta de flujo del proxy compartido; `kilo-auto/balanced` y los identificadores de razonamiento de proxy no compatibles omiten el pensamiento inyectado | `kilocode` |
    | `moonshot-thinking` | Asignación de la carga útil binaria de pensamiento nativo de Moonshot desde la configuración y el nivel `/think` | `moonshot` |
    | `minimax-fast-mode` | Reescritura del modelo del modo rápido de MiniMax en la ruta de flujo compartida | `minimax`, `minimax-portal` |
    | `openai-responses-defaults` | Contenedores nativos compartidos de Responses de OpenAI/Codex: encabezados de atribución, `/fast`/`serviceTier`, nivel de detalle del texto, búsqueda web nativa de Codex, adaptación de la carga útil para compatibilidad con el razonamiento y gestión del contexto de Responses | `openai` |
    | `openrouter-thinking` | Contenedor de razonamiento de OpenRouter para rutas de proxy, con las omisiones por modelo no compatible/`auto` gestionadas de forma centralizada | `openrouter` |
    | `tool-stream-default-on` | Contenedor `tool_stream` activado de forma predeterminada para proveedores como Z.AI que requieren transmisión de herramientas, salvo que se desactive explícitamente | `zai` |

    <Accordion title="Puntos de integración del SDK que sustentan los constructores de familias">
      Cada constructor de familias se compone a partir de auxiliares públicos de menor nivel exportados desde el mismo paquete, a los que se puede recurrir cuando un proveedor debe apartarse del patrón común:

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`, `buildProviderReplayFamilyHooks(...)` y los constructores de reproducción sin procesar (`buildOpenAICompatibleReplayPolicy`, `buildAnthropicReplayPolicyForModel`, `buildGoogleGeminiReplayPolicy`, `buildHybridAnthropicOrOpenAIReplayPolicy`). También exporta auxiliares de reproducción de Gemini (`sanitizeGoogleGeminiReplayHistory`, `resolveTaggedReasoningOutputMode`) y auxiliares de endpoints/modelos (`resolveProviderEndpoint`, `normalizeProviderId`, `normalizeGooglePreviewModelId`).
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`, `buildProviderStreamFamilyHooks(...)`, `composeProviderStreamWrappers(...)`, además de los contenedores compartidos de OpenAI/Codex (`createOpenAIAttributionHeadersWrapper`, `createOpenAIFastModeWrapper`, `createOpenAIServiceTierWrapper`, `createOpenAIResponsesContextManagementWrapper`, `createCodexNativeWebSearchWrapper`), el contenedor compatible con OpenAI de DeepSeek V4 (`createDeepSeekV4OpenAICompatibleThinkingWrapper`), la limpieza del prellenado de pensamiento de Anthropic Messages (`createAnthropicThinkingPrefillPayloadWrapper`), la compatibilidad de llamadas de herramientas con texto sin formato (`createPlainTextToolCallCompatWrapper`) y los contenedores compartidos de proxies/proveedores (`createOpenRouterWrapper`, `createToolStreamWrapper`, `createMinimaxFastModeWrapper`).
      - `openclaw/plugin-sdk/provider-stream-shared` - contenedores ligeros de cargas útiles y eventos para rutas críticas de proveedores, incluidos `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPayloadPatchStreamWrapper`, `createPlainTextToolCallCompatWrapper`, `normalizeOpenAICompatibleReasoningPayload(...)` y `setQwenChatTemplateThinking(...)`.
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")` y auxiliares subyacentes de esquemas de proveedores.

      Para los proveedores de la familia Gemini, se debe mantener el modo de salida de razonamiento alineado con
      el transporte. Los proveedores directos de la API de Google Gemini deben usar la salida de razonamiento `native`
      para que OpenClaw consuma las partes de pensamiento nativas sin añadir
      las directivas de prompt `<think>` / `<final>`. Los backends de estilo CLI de Gemini
      exclusivamente de texto que analizan una respuesta final JSON/de texto pueden conservar el contrato etiquetado
      compartido `google-gemini`.

      Algunos auxiliares de flujo permanecen locales al proveedor de forma intencionada. `@openclaw/anthropic-provider` mantiene `wrapAnthropicProviderStream`, `resolveAnthropicBetas`, `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` y los constructores de contenedores de Anthropic de menor nivel en su propio punto de integración público `api.ts` / `contract-api.ts`, porque codifican la gestión de la versión beta de OAuth de Claude y el control de `context1m`. Del mismo modo, el plugin de xAI mantiene la adaptación nativa de Responses de xAI en su propio `wrapStreamFn` (alias `/fast`, valor predeterminado `tool_stream`, limpieza de herramientas estrictas no compatibles y eliminación de la carga útil de razonamiento específica de xAI).

      El mismo patrón de raíz del paquete también sustenta `@openclaw/openai-provider` (constructores de proveedores, auxiliares de modelos predeterminados y constructores de proveedores en tiempo real) y `@openclaw/openrouter-provider` (constructor de proveedores junto con auxiliares de incorporación/configuración).
    </Accordion>

    <Tabs>
      <Tab title="Intercambio de tokens">
        Para proveedores que requieren un intercambio de tokens antes de cada llamada de inferencia:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="Encabezados personalizados">
        Para proveedores que requieren encabezados de solicitud personalizados o modificaciones del cuerpo:

        ```typescript
        // wrapStreamFn devuelve una StreamFn derivada de ctx.streamFn
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="Identidad de transporte nativa">
        Para proveedores que requieren encabezados o metadatos nativos de solicitud/sesión en
        transportes HTTP o WebSocket genéricos:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="Uso y facturación">
        Para proveedores que exponen datos de uso/facturación:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` tiene tres resultados. Se debe devolver
        `{ token, accountId?, subscriptionType?, rateLimitTier? }` cuando el
        proveedor tenga una credencial de uso/facturación (los campos opcionales trasladan
        metadatos no secretos del plan desde el perfil resuelto hasta
        `fetchUsageSnapshot`). Se debe devolver
        `{ handled: true }` únicamente cuando el proveedor haya gestionado definitivamente la autenticación
        de uso, pero no disponga de un token de uso válido, y OpenClaw deba omitir la alternativa genérica
        de clave de API/OAuth. Se debe devolver `null` o `undefined` cuando el proveedor no haya
        gestionado la solicitud y OpenClaw deba continuar con la alternativa genérica.

        Se debe declarar el identificador del proveedor en `contracts.usageProviders`. Cuando ese contrato
        de manifiesto y **ambos** hooks están presentes, OpenClaw incluye automáticamente
        al proveedor en la recopilación de uso sin cargar plugins de proveedores
        no relacionados. No es necesario actualizar ninguna lista de permitidos del núcleo.
        `fetchUsageSnapshot` devuelve la forma compartida independiente del proveedor:

        - `plan`: etiqueta de suscripción o clave indicada por el proveedor
        - `windows`: ventanas de cuota reiniciables como porcentajes utilizados
        - `billing`: entradas con tipo `balance`, `spend` o `budget`; `unit` puede ser
          una moneda ISO o una unidad del proveedor como `credits`
        - `summary`: contexto compacto específico del proveedor que no encaja en esos
          campos estructurados

        Se debe mantener exacta la semántica de las monedas. Un crédito del proveedor no equivale a USD salvo que el
        contrato ascendente así lo indique. Un plugin que solo implemente
        `fetchUsageSnapshot` continúa disponible para llamadores explícitos/sintéticos, pero
        no se detecta automáticamente porque OpenClaw no puede resolver su credencial de uso.
      </Tab>
    </Tabs>

    <Accordion title="Hooks comunes de proveedores">
      OpenClaw llama a los hooks aproximadamente en este orden para los plugins de modelos/proveedores.
      La mayoría de los proveedores solo utilizan 2-3. Este no es el contrato
      `ProviderPlugin` completo; consulte [Aspectos internos: hooks del entorno de ejecución de
      proveedores](/es/plugins/architecture-internals#provider-runtime-hooks) para ver la lista
      completa y actualizada de hooks y las notas sobre alternativas.
      Los campos de proveedores exclusivos para compatibilidad que OpenClaw ya no invoca, como
      `ProviderPlugin.capabilities` y `suppressBuiltInModel`, no se incluyen
      aquí.

      | Hook | Cuándo utilizarlo |
      | --- | --- |
      | `catalog` | Catálogo de modelos o valores predeterminados de la URL base |
      | `applyConfigDefaults` | Valores predeterminados globales propiedad del proveedor durante la materialización de la configuración |
      | `normalizeModelId` | Limpieza de alias de identificadores de modelos heredados/preliminares antes de la búsqueda |
      | `normalizeTransport` | Limpieza de `api` / `baseUrl` de la familia del proveedor antes del ensamblaje genérico del modelo |
      | `normalizeConfig` | Normalizar la configuración de `models.providers.<id>` |
      | `applyNativeStreamingUsageCompat` | Reescrituras nativas de compatibilidad del uso transmitido para proveedores de configuración |
      | `resolveConfigApiKey` | Resolución de autenticación mediante marcadores de entorno propiedad del proveedor |
      | `resolveSyntheticAuth` | Autenticación sintética local/autohospedada o respaldada por configuración |
      | `resolveExternalAuthProfiles` | Superponer perfiles de autenticación externos propiedad del proveedor para credenciales gestionadas por la CLI/aplicación |
      | `shouldDeferSyntheticProfileAuth` | Subordinar los marcadores sintéticos de perfiles almacenados a la autenticación de entorno/configuración |
      | `resolveDynamicModel` | Aceptar identificadores arbitrarios de modelos ascendentes |
      | `prepareDynamicModel` | Obtención asíncrona de metadatos antes de la resolución |
      | `normalizeResolvedModel` | Reescrituras del transporte antes del ejecutor |
      | `normalizeToolSchemas` | Limpieza del esquema de herramientas propiedad del proveedor antes del registro |
      | `inspectToolSchemas` | Diagnósticos del esquema de herramientas propiedad del proveedor |
      | `resolveReasoningOutputMode` | Contrato de salida de razonamiento etiquetado frente a nativo |
      | `prepareExtraParams` | Parámetros predeterminados de solicitud |
      | `createStreamFn` | Transporte StreamFn totalmente personalizado |
      | `wrapStreamFn` | Contenedores personalizados de encabezados/cuerpo en la ruta de flujo normal |
      | `resolveTransportTurnState` | Encabezados/metadatos nativos por turno |
      | `resolveWebSocketSessionPolicy` | Encabezados/tiempo de espera de sesión de WS nativos |
      | `formatApiKey` | Forma personalizada del token del entorno de ejecución |
      | `refreshOAuth` | Actualización personalizada de OAuth |
      | `buildAuthDoctorHint` | Orientación para reparar la autenticación |
      | `matchesContextOverflowError` | Detección de desbordamiento propiedad del proveedor |
      | `classifyFailoverReason` | Clasificación de límites de velocidad/sobrecarga propiedad del proveedor |
      | `isCacheTtlEligible` | Control del TTL de la caché de prompts |
      | `buildMissingAuthMessage` | Indicación personalizada de autenticación ausente |
      | `augmentModelCatalog` | Filas sintéticas de compatibilidad futura (obsoleto; se prefiere `registerModelCatalogProvider`) |
      | `resolveThinkingProfile` | Conjunto de opciones `/think` específico del modelo |
      | `isBinaryThinking` | Compatibilidad para activar/desactivar el pensamiento binario (obsoleto; se prefiere `resolveThinkingProfile`) |
      | `supportsXHighThinking` | Compatibilidad con el razonamiento `xhigh` (obsoleto; se prefiere `resolveThinkingProfile`) |
      | `resolveDefaultThinkingLevel` | Compatibilidad con la política `/think` predeterminada (obsoleto; se prefiere `resolveThinkingProfile`) |
      | `isModernModelRef` | Correspondencia de modelos para pruebas en vivo/de humo |
      | `prepareRuntimeAuth` | Intercambio de tokens antes de la inferencia |
      | `resolveUsageAuth` | Análisis personalizado de credenciales de uso |
      | `fetchUsageSnapshot` | Endpoint de uso personalizado |
      | `createEmbeddingProvider` | Adaptador de embeddings propiedad del proveedor para memoria/búsqueda |
      | `buildReplayPolicy` | Política personalizada de reproducción/Compaction de transcripciones |
      | `sanitizeReplayHistory` | Reescrituras de reproducción específicas del proveedor después de la limpieza genérica |
      | `validateReplayTurns` | Validación estricta de turnos de reproducción antes del ejecutor integrado |
      | `onModelSelected` | Retrollamada posterior a la selección (p. ej., telemetría) |

      Notas sobre las alternativas del entorno de ejecución:

      - `normalizeConfig` resuelve un Plugin propietario por id de proveedor (primero los proveedores integrados y luego el Plugin de tiempo de ejecución coincidente) y llama únicamente a ese hook; no se examinan otros proveedores. El hook `normalizeConfig` propio de Google es el que normaliza las entradas de configuración `google` / `google-vertex` / `google-antigravity`; no es un mecanismo de reserva independiente del núcleo.
      - `resolveConfigApiKey` usa el hook del proveedor cuando está expuesto. Amazon Bedrock mantiene la resolución de marcadores de entorno de AWS en su Plugin de proveedor; la autenticación en tiempo de ejecución sigue usando la cadena predeterminada del SDK de AWS cuando se configura con `auth: "aws-sdk"`.
      - `resolveThinkingProfile(ctx)` recibe los elementos seleccionados `provider`, `modelId`, la indicación opcional combinada del catálogo `reasoning` y los datos opcionales combinados del modelo `compat`. Use `compat` únicamente para seleccionar la interfaz o el perfil de razonamiento del proveedor.
      - `resolveSystemPromptContribution` permite que un proveedor inserte orientación para el prompt del sistema que tenga en cuenta la caché de una familia de modelos. Se prefiere frente al hook heredado `before_prompt_build` para todo el Plugin cuando el comportamiento corresponde a una familia de proveedor o modelo y debe preservar la división estable/dinámica de la caché.

    </Accordion>

  </Step>

  <Step title="Añadir capacidades adicionales (opcional)">
    ### Paso 5: Añadir capacidades adicionales

    Un Plugin de proveedor puede registrar embeddings, voz, transcripción en tiempo real,
    voz en tiempo real, comprensión multimedia, generación de imágenes, generación de vídeo,
    obtención web y búsqueda web junto con la inferencia de texto. OpenClaw lo clasifica como un
    Plugin de **capacidades híbridas**, el patrón recomendado para los Plugins de empresas
    (un Plugin por proveedor). Consulte
    [Aspectos internos: propiedad de las capacidades](/es/plugins/architecture#capability-ownership-model).

    Registre cada capacidad dentro de `register(api)` junto con su llamada
    `api.registerProvider(...)` existente. Elija únicamente las pestañas que necesite:

    <Tabs>
      <Tab title="Voz (TTS)">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Voz de Acme",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "voz de acme",
            });
            try {
              await assertOkOrThrowProviderError(response, "Error de la API de voz de Acme");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        Use `assertOkOrThrowProviderError(...)` para los errores HTTP del proveedor, de modo que los
        Plugins compartan lecturas limitadas del cuerpo de los errores, análisis de errores JSON y
        sufijos de identificadores de solicitud.
      </Tab>
      <Tab title="Transcripción en tiempo real">
        Se prefiere `createRealtimeTranscriptionWebSocketSession(...)`: el asistente
        compartido gestiona la captura del proxy, la espera incremental de reconexión, el vaciado al cerrar, los protocolos de enlace
        de disponibilidad, la puesta en cola del audio y los diagnósticos de eventos de cierre. Su Plugin
        solo asigna los eventos del servicio de origen.

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Transcripción en tiempo real de Acme",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        Los proveedores de STT por lotes que envíen audio multipart mediante POST deben usar
        `buildAudioTranscriptionFormData(...)` de
        `openclaw/plugin-sdk/provider-http`. El asistente normaliza los nombres de archivo
        de las cargas, incluidas las cargas AAC que necesitan un nombre de archivo de tipo M4A para
        las API de transcripción compatibles.
      </Tab>
      <Tab title="Voz en tiempo real">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Voz en tiempo real de Acme",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // Establezca esto únicamente si el proveedor acepta varias respuestas de herramientas para
            // una llamada, por ejemplo, una respuesta inmediata de "procesando" seguida
            // del resultado final.
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        Declare `capabilities` para que `talk.catalog` pueda exponer los modos,
        transportes, formatos de audio e indicadores de funciones válidos a los clientes Talk
        nativos y de navegador. Implemente `handleBargeIn` cuando un transporte pueda detectar que una
        persona está interrumpiendo la reproducción del asistente y el proveedor admita
        truncar o borrar la respuesta de audio activa.
        `submitToolResult` puede devolver `void` para un envío síncrono o una
        `Promise<void>` para un límite de finalización asíncrona que pueda exponer el puente
        del proveedor. Las sesiones de retransmisión del Gateway esperan esa promesa antes de
        confirmar un resultado final o borrar la ejecución vinculada; rechácela cuando
        falle el envío.
        Establezca `supportsToolResultSuppression: false` cuando el proveedor no pueda
        respetar `options.suppressResponse`. OpenClaw evita entonces la supresión para
        los resultados internos de consulta forzada y cancelación, y rechaza las solicitudes directas
        de resultados suprimidos en lugar de iniciar silenciosamente una respuesta.
        Los consumidores de `createRealtimeVoiceBridgeSession` también pueden devolver una
        promesa desde `onToolCall`; las excepciones síncronas y los rechazos se dirigen
        a la devolución de llamada `onError` de la sesión.
        Establezca `handlesInputAudioBargeIn` únicamente cuando la VAD del proveedor confirme una
        interrupción llamando a `onClearAudio("barge-in")`. Los proveedores que omitan
        el indicador usan la detección de reserva local de audio de entrada de OpenClaw.
      </Tab>
      <Tab title="Comprensión multimedia">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "Una foto de..." }),
          transcribeAudio: async (req) => ({ text: "Transcripción..." }),
        });
        ```

        Los proveedores multimedia locales o autoalojados que deliberadamente no requieran
        credenciales pueden exponer `resolveAuth` y devolver `kind: "none"`.
        OpenClaw mantiene la comprobación de autenticación normal para los proveedores que no
        acepten explícitamente esta opción. Los proveedores existentes pueden seguir leyendo `req.apiKey`;
        los proveedores nuevos deben preferir `req.auth`.

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "Plugin local-audio sin autenticación",
          }),
          transcribeAudio: async (req) => ({ text: "Transcripción..." }),
        });
        ```
      </Tab>
      <Tab title="Embeddings">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        Declare el mismo id en `contracts.embeddingProviders`. Este es el
        contrato general de embeddings para la generación reutilizable de vectores, incluida
        la búsqueda en memoria. `registerMemoryEmbeddingProvider(...)` es una compatibilidad
        obsoleta para los adaptadores existentes específicos de memoria.
      </Tab>
      <Tab title="Generación de imágenes y vídeo">
        Las capacidades de imagen y vídeo usan una estructura **basada en modos**. Los proveedores
        de imágenes declaran los bloques de capacidades obligatorios `generate` y `edit`;
        los proveedores de vídeo declaran `generate`, `imageToVideo` y
        `videoToVideo`. Los campos agregados planos como `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` no bastan para anunciar
        correctamente la compatibilidad con el modo de transformación ni los modos deshabilitados. La generación de música
        sigue el mismo patrón `generate` / `edit`.

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Imágenes de Acme",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Vídeo de Acme",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        `capabilities` es obligatorio en ambos tipos de proveedor; `edit` y los
        bloques de transformación de vídeo (`imageToVideo`, `videoToVideo`) siempre necesitan
        un indicador `enabled` explícito.

        Use `catalogByModel` cuando los modos estáticos o las capacidades de un modelo enumerado
        difieran de los valores predeterminados del proveedor. Estos metadatos mantienen
        actualizados `video_generate action=list` y los catálogos de modelos sin
        invocar código del proveedor. La consulta y la aplicación de capacidades durante
        la solicitud siguen correspondiendo a `resolveModelCapabilities` y `generateVideo`; reutilice
        la misma constante de capacidad para ambas rutas cuando sea posible.
      </Tab>
      <Tab title="Obtención y búsqueda web">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Obtención de Acme",
          hint: "Obtenga páginas mediante el backend de renderizado de Acme.",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Obtenga una página mediante la obtención de Acme.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Búsqueda de Acme",
          hint: "Busque en la web mediante el backend de búsqueda de Acme.",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Busque en la web mediante la búsqueda de Acme.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        Ambos tipos de proveedor comparten la misma estructura de conexión de credenciales:
        `hint`, `envVars`, `placeholder`, `signupUrl`, `credentialPath`,
        `getCredentialValue`, `setCredentialValue` y `createTool` son
        obligatorios.
      </Tab>
    </Tabs>

  </Step>

  <Step title="Prueba">
    ### Paso 6: Prueba

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // Exporte el objeto de configuración del proveedor desde index.ts o un archivo dedicado
    import { acmeProvider } from "./provider.js";

    describe("proveedor acme-ai", () => {
      it("resuelve modelos dinámicos", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("devuelve el catálogo cuando la clave está disponible", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("devuelve un catálogo nulo cuando no hay clave", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## Publicación en ClawHub

Los plugins de proveedores se publican del mismo modo que cualquier otro plugin de código externo:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` es un comando diferente para publicar una carpeta de Skills,
no un paquete de plugin; no lo use aquí.

## Estructura de archivos

```
<bundled-plugin-root>/acme-ai/
├── package.json              # Metadatos de openclaw.providers
├── openclaw.plugin.json      # Manifiesto con metadatos de autenticación del proveedor
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Pruebas
    └── usage.ts              # Endpoint de uso (opcional)
```

## Referencia del orden del catálogo

`catalog.order` controla cuándo se combina el catálogo en relación con los proveedores
integrados:

| Orden     | Cuándo          | Caso de uso                                      |
| --------- | --------------- | ------------------------------------------------ |
| `simple`  | Primera pasada  | Proveedores con una clave de API simple          |
| `profile` | Después de los simples | Proveedores condicionados a perfiles de autenticación |
| `paired`  | Después del perfil | Sintetizar varias entradas relacionadas          |
| `late`    | Última pasada   | Sobrescribir proveedores existentes (prevalece en caso de colisión) |

## Próximos pasos

- [Plugins de canal](/es/plugins/sdk-channel-plugins) - si el plugin también proporciona un canal
- [Entorno de ejecución del SDK](/es/plugins/sdk-runtime) - asistentes de `api.runtime` (TTS, búsqueda, subagente)
- [Descripción general del SDK](/es/plugins/sdk-overview) - referencia completa de importaciones de subrutas
- [Funcionamiento interno de los plugins](/es/plugins/architecture-internals#provider-runtime-hooks) - detalles de los hooks y ejemplos integrados

## Relacionado

- [Configuración del SDK de plugins](/es/plugins/sdk-setup)
- [Creación de plugins](/es/plugins/building-plugins)
- [Creación de plugins de canal](/es/plugins/sdk-channel-plugins)
