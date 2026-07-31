---
read_when:
    - Estás creando un plugin de OpenClaw
    - Necesita publicar un esquema de configuración de Plugin o depurar errores de validación de Plugin
summary: Requisitos del manifiesto del Plugin y del esquema JSON (validación estricta de la configuración)
title: Manifiesto del Plugin
x-i18n:
    generated_at: "2026-07-26T04:46:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 244e5c8265ff79b0ff6e8f4b60c9635cccc3ba66093cecab458676beb9578264
    source_path: plugins/manifest.md
    workflow: 16
---

Esta página trata sobre el **manifiesto nativo de plugins de OpenClaw**, `openclaw.plugin.json`. Para conocer las estructuras de paquetes compatibles (Codex, Claude, Cursor), consulte [Paquetes de plugins](/es/plugins/bundles).

Los formatos de paquetes compatibles utilizan sus propios archivos de manifiesto:

- Paquete de Codex: `.codex-plugin/plugin.json`
- Paquete de Claude: `.claude-plugin/plugin.json`, o la estructura predeterminada de componentes de Claude sin manifiesto
- Paquete de Cursor: `.cursor-plugin/plugin.json`

OpenClaw detecta automáticamente estas estructuras, pero no las valida con el esquema `openclaw.plugin.json` que aparece a continuación. En los paquetes compatibles, OpenClaw lee los metadatos del paquete, las raíces de Skills declaradas, las raíces de comandos de Claude, los valores predeterminados de `settings.json` de Claude, los valores predeterminados de LSP de Claude y los paquetes de hooks compatibles, cuando la estructura coincide con las expectativas del entorno de ejecución de OpenClaw.

Cada plugin nativo de OpenClaw **debe** incluir `openclaw.plugin.json` en la **raíz del plugin**. OpenClaw lo lee para validar la configuración **sin ejecutar el código del plugin**. La ausencia de un manifiesto o un manifiesto no válido bloquea la validación de la configuración y se trata como un error del plugin.

Consulte [Plugins](/es/tools/plugin) para obtener la guía completa del sistema de plugins y [Modelo de capacidades](/es/plugins/architecture#public-capability-model) para conocer el modelo de capacidades nativo y las directrices actuales de compatibilidad externa.

## Función de este archivo

`openclaw.plugin.json` contiene metadatos que OpenClaw lee **antes de cargar el código del plugin**. Todo su contenido debe poder inspeccionarse con un coste suficientemente bajo sin iniciar el entorno de ejecución del plugin.

**Se utiliza para:**

- identidad del plugin, validación de la configuración e indicaciones para la interfaz de configuración
- metadatos de autenticación, incorporación y configuración (alias, activación automática, variables de entorno del proveedor y opciones de autenticación)
- indicaciones de activación para las superficies del plano de control
- propiedad abreviada de familias de modelos
- instantáneas estáticas de propiedad de capacidades (`contracts`)
- vinculaciones de datos y verbos de acción de los widgets del panel
- servidores MCP estáticos que deben existir mientras el plugin esté habilitado
- metadatos del ejecutor de control de calidad que puede inspeccionar el host compartido `openclaw qa`
- metadatos de configuración específicos del canal que se combinan en las superficies de catálogo y validación

**No se utiliza para:** registrar hooks nativos del entorno de ejecución, declarar puntos de entrada del código del plugin ni especificar metadatos de instalación de npm. Estos elementos corresponden al código del plugin y a `package.json`.

## Ejemplo mínimo

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Ejemplo completo

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "Plugin de proveedor OpenRouter",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "setup": {
    "providers": [
      {
        "id": "openrouter",
        "envVars": ["OPENROUTER_API_KEY"]
      }
    ]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "Clave de API de OpenRouter",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "Clave de API de OpenRouter",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "Clave de API",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Referencia de campos de nivel superior

| Campo                                | Obligatorio | Tipo                         | Qué significa                                                                                                                                                                                                                                                                                  |
| ------------------------------------ | ----------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                                 | Sí      | `string`                     | Id canónico del plugin. Este es el id utilizado en `plugins.entries.<id>`.                                                                                                                                                                                                                            |
| `configSchema`                       | Sí      | `object`                     | Esquema JSON en línea para la configuración de este plugin.                                                                                                                                                                                                                                                   |
| `requiresPlugins`                    | No       | `string[]`                   | Ids de plugins que también deben estar instalados para que este plugin surta efecto. El descubrimiento mantiene el plugin disponible para su carga, pero advierte cuando falta algún plugin obligatorio.                                                                                                                                   |
| `enabledByDefault`                   | No       | `true`                       | Marca un plugin incluido como habilitado de forma predeterminada. Se debe omitir o establecer en cualquier valor distinto de `true` para dejar el plugin deshabilitado de forma predeterminada.                                                                                                                                                                   |
| `enabledByDefaultOnPlatforms`        | No       | `string[]`                   | Marca un plugin incluido como habilitado de forma predeterminada solo en las plataformas Node.js indicadas, por ejemplo, `["darwin"]`. La configuración explícita sigue teniendo prioridad.                                                                                                                                                       |
| `legacyPluginIds`                    | No       | `string[]`                   | Ids heredados que se normalizan a este id canónico del plugin.                                                                                                                                                                                                                                         |
| `autoEnableWhenConfiguredProviders`  | No       | `string[]`                   | Ids de proveedores que deben habilitar automáticamente este plugin cuando se mencionen en referencias de autenticación, configuración o modelos.                                                                                                                                                                                                |
| `kind`                               | No       | `PluginKind \| PluginKind[]` | Declara uno o más tipos de plugin exclusivos (`"memory"`, `"context-engine"`) utilizados por `plugins.slots.*`. Un plugin que posee ambos espacios declara ambos tipos en un único arreglo.                                                                                                                        |
| `channels`                           | No       | `string[]`                   | Ids de canales que pertenecen a este plugin. Se utilizan para el descubrimiento y la validación de la configuración.                                                                                                                                                                                                                    |
| `providers`                          | No       | `string[]`                   | Ids de proveedores que pertenecen a este plugin.                                                                                                                                                                                                                                                             |
| `providerCatalogEntry`               | No       | `string`                     | Ruta del módulo ligero del catálogo de proveedores, relativa a la raíz del plugin, para los metadatos del catálogo de proveedores limitados al manifiesto que pueden cargarse sin activar el entorno de ejecución completo del plugin.                                                                                                            |
| `modelSupport`                       | No       | `object`                     | Metadatos abreviados de la familia de modelos, pertenecientes al manifiesto, que se utilizan para cargar automáticamente el plugin antes del entorno de ejecución.                                                                                                                                                                                                    |
| `modelCatalog`                       | No       | `object`                     | Metadatos declarativos del catálogo de modelos para los proveedores que pertenecen a este plugin. Este es el contrato del plano de control para futuros listados de solo lectura, incorporación, selectores de modelos, alias y supresión sin cargar el entorno de ejecución del plugin.                                                                    |
| `modelPricing`                       | No       | `object`                     | Política de consulta de precios externos perteneciente al proveedor. Se utiliza para excluir a los proveedores locales o autoalojados de los catálogos de precios remotos o asignar referencias de proveedores a ids de catálogos de OpenRouter/LiteLLM sin codificar de forma fija los ids de proveedores en el núcleo.                                                                        |
| `modelIdNormalization`               | No       | `object`                     | Limpieza de alias o prefijos de ids de modelos perteneciente al proveedor que debe ejecutarse antes de que se cargue el entorno de ejecución del proveedor.                                                                                                                                                                                                      |
| `providerEndpoints`                  | No       | `object[]`                   | Metadatos de host/baseUrl del punto de conexión, pertenecientes al manifiesto, para las rutas de proveedores que el núcleo debe clasificar antes de que se cargue el entorno de ejecución del proveedor.                                                                                                                                                                       |
| `providerRequest`                    | No       | `object`                     | Metadatos ligeros de familia de proveedores y compatibilidad de solicitudes que utiliza la política genérica de solicitudes antes de que se cargue el entorno de ejecución del proveedor.                                                                                                                                                                         |
| `secretProviderIntegrations`         | No       | `Record<string, object>`     | Preajustes declarativos de proveedores de ejecución SecretRef que las interfaces de configuración o instalación pueden ofrecer sin codificar de forma fija integraciones específicas de proveedores en el núcleo.                                                                                                                                                |
| `cliBackends`                        | No       | `string[]`                   | Ids de backends de inferencia de la CLI que pertenecen a este plugin. Se utilizan para la activación automática al inicio a partir de referencias de configuración explícitas.                                                                                                                                                                                    |
| `syntheticAuthRefs`                  | No       | `string[]`                   | Referencias de proveedores o backends de la CLI cuyo enlace de autenticación sintética, perteneciente al plugin, debe sondearse durante el descubrimiento de modelos en frío antes de que se cargue el entorno de ejecución.                                                                                                                                                         |
| `nonSecretAuthMarkers`               | No       | `string[]`                   | Valores de marcador de posición de claves de API pertenecientes a plugins incluidos que representan el estado no secreto de credenciales locales, OAuth o del entorno.                                                                                                                                                                           |
| `commandAliases`                     | No       | `object[]`                   | Nombres de comandos que pertenecen a este plugin y que deben generar diagnósticos de configuración y de la CLI compatibles con el plugin antes de que se cargue el entorno de ejecución.                                                                                                                                                                           |
| `providerUsageAuthEnvVars`           | No       | `Record<string, string[]>`   | Credenciales de proveedores solo para uso y facturación. OpenClaw utiliza estos nombres para el descubrimiento de uso y la eliminación de secretos, pero nunca para la autenticación de inferencia.                                                                                                                                                      |
| `providerAuthAliases`                | No       | `Record<string, string>`     | Ids de proveedores que deben reutilizar otro id de proveedor para la consulta de autenticación; por ejemplo, un proveedor de programación que comparte la clave de API y los perfiles de autenticación del proveedor base.                                                                                                                                     |
| `providerAuthChoices`                | No       | `object[]`                   | Metadatos ligeros de opciones de autenticación para los selectores de incorporación, la resolución del proveedor preferido y la vinculación sencilla de indicadores de la CLI.                                                                                                                                                                                  |
| `activation`                         | No       | `object`                     | Metadatos ligeros del planificador de activación para la carga activada por el inicio, el proveedor, el comando, el canal, la ruta y la capacidad. Solo metadatos; el entorno de ejecución del plugin sigue siendo responsable del comportamiento real.                                                                                                                  |
| `setup`                              | No       | `object`                     | Descriptores ligeros de configuración e incorporación que las interfaces de descubrimiento y configuración pueden inspeccionar sin cargar el entorno de ejecución del plugin.                                                                                                                                                                               |
| `qaRunners`                          | No       | `object[]`                   | Descriptores ligeros del ejecutor de control de calidad utilizados por el host compartido `openclaw qa` antes de que se cargue el entorno de ejecución del plugin.                                                                                                                                                                                                 |
| `dashboard`                          | No       | `object`                     | Enlaces de datos y verbos de acción de los widgets del panel. Cada entrada se valida con un método del Gateway registrado por este plugin con el ámbito de lectura o escritura requerido. Consulte la [referencia del panel](#dashboard-reference).                                                                            |
| `mcpServers`                         | No       | `Record<string, object>`     | Definiciones estáticas de servidores MCP aportadas mientras este plugin está habilitado. Los argumentos de comandos y los directorios de trabajo relativos se resuelven desde la raíz del plugin. Las entradas `mcp.servers` del operador sobrescriben o deshabilitan las definiciones con el mismo nombre. Consulte la [referencia de servidores MCP](#mcp-server-reference). |
| `contracts`                          | No       | `object`                     | Instantánea estática de la propiedad de capacidades para hooks de autenticación externos, embeddings, voz, transcripción en tiempo real, voz en tiempo real, comprensión multimedia, generación de imágenes, vídeo y música, obtención web, búsqueda web, proveedores de workers, extracción de documentos y contenido web, y propiedad de herramientas.                     |
| `configContracts`                    | No       | `object`                     | Comportamiento de configuración propiedad del manifiesto y utilizado por los auxiliares genéricos del núcleo: detección de indicadores peligrosos, destinos de migración de SecretRef y restricción de rutas de configuración heredadas. Consulte la [referencia de configContracts](#configcontracts-reference).                                                                         |
| `mediaUnderstandingProviderMetadata` | No       | `Record<string, object>`     | Valores predeterminados económicos de comprensión multimedia para los identificadores de proveedor declarados en `contracts.mediaUnderstandingProviders`.                                                                                                                                                                                       |
| `imageGenerationProviderMetadata`    | No       | `Record<string, object>`     | Metadatos económicos de autenticación para la generación de imágenes correspondientes a los identificadores de proveedor declarados en `contracts.imageGenerationProviders`, incluidos los alias de autenticación propiedad del proveedor y las protecciones de URL base.                                                                                                                             |
| `videoGenerationProviderMetadata`    | No       | `Record<string, object>`     | Metadatos económicos de autenticación para la generación de vídeo correspondientes a los identificadores de proveedor declarados en `contracts.videoGenerationProviders`, incluidos los alias de autenticación propiedad del proveedor y las protecciones de URL base.                                                                                                                             |
| `musicGenerationProviderMetadata`    | No       | `Record<string, object>`     | Metadatos económicos de autenticación para la generación de música correspondientes a los identificadores de proveedor declarados en `contracts.musicGenerationProviders`, incluidos los alias de autenticación propiedad del proveedor y las protecciones de URL base.                                                                                                                             |
| `toolMetadata`                       | No       | `Record<string, object>`     | Metadatos económicos de disponibilidad para las herramientas propiedad del plugin declaradas en `contracts.tools`. Úselos cuando una herramienta no deba cargar el entorno de ejecución a menos que existan indicios de configuración, entorno o autenticación.                                                                                                                      |
| `channelConfigs`                     | No       | `Record<string, object>`     | Metadatos de configuración de canales propiedad del manifiesto que se combinan con las superficies de detección y validación antes de que se cargue el entorno de ejecución.                                                                                                                                                                                     |
| `skills`                             | No       | `string[]`                   | Directorios de Skills que se cargarán, relativos a la raíz del plugin.                                                                                                                                                                                                                                        |
| `name`                               | No       | `string`                     | Nombre del plugin legible para las personas.                                                                                                                                                                                                                                                                    |
| `description`                        | No       | `string`                     | Resumen breve que se muestra en las superficies del plugin.                                                                                                                                                                                                                                                        |
| `catalog`                            | No       | `object`                     | Indicaciones opcionales de presentación para las superficies del catálogo de plugins. Estos metadatos no instalan ni habilitan un plugin, ni le conceden confianza.                                                                                                                                                                   |
| `icon`                               | No       | `string`                     | URL HTTPS de la imagen para las tarjetas del marketplace o catálogo. ClawHub acepta cualquier URL `https://` válida y recurre al icono predeterminado del plugin cuando se omite o no es válida.                                                                                                                             |
| `version`                            | No       | `string`                     | Versión informativa del plugin.                                                                                                                                                                                                                                                                  |
| `uiHints`                            | No       | `Record<string, object>`     | Etiquetas de la interfaz de usuario, textos de marcador de posición e indicaciones de sensibilidad para los campos de configuración.                                                                                                                                                                                                                              |

## Referencia del servidor MCP

`mcpServers` permite que un plugin nativo incluya un servidor MCP, incluida una aplicación MCP, sin exigir que los operadores dupliquen su definición estática de proceso en `openclaw.json`:

```json
{
  "mcpServers": {
    "example": {
      "transport": "stdio",
      "command": "node",
      "args": ["./mcp-server.js"]
    }
  }
}
```

OpenClaw incluye estos servidores únicamente mientras el plugin propietario está habilitado. Las rutas relativas `command`, `args`, `cwd` y `workingDirectory` se resuelven desde la raíz del plugin. La configuración del usuario sigue siendo vinculante: `mcp.servers.<name>` puede reemplazar un valor predeterminado del plugin o establecer `enabled: false` para omitirlo. La representación de aplicaciones MCP y las llamadas a herramientas del servidor siguen requiriendo la configuración habitual de aplicaciones MCP y la política de herramientas efectiva; declarar un servidor no elude ninguno de estos límites.

## Referencia del panel

`dashboard` permite que un plugin habilitado exponga RPC existentes del Gateway a widgets del panel con los permisos correspondientes, sin añadir políticas del plugin al núcleo. Los enlaces de datos deben indicar un método que el mismo plugin registre con `operator.read`; los verbos de acción deben indicar un método que registre con `operator.write`. Una discrepancia provoca el rechazo del plugin durante el registro.

```json
{
  "dashboard": {
    "dataBindings": [
      {
        "id": "items.list",
        "method": "example.items.list",
        "description": "List example items."
      }
    ],
    "actionVerbs": [
      {
        "id": "refresh",
        "method": "example.items.refresh",
        "description": "Refresh example items.",
        "paramShape": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "force": { "type": "boolean" }
          }
        }
      }
    ]
  }
}
```

Los identificadores del manifiesto son locales al plugin. Los permisos de los widgets usan `<plugin-id>.<id>`, como `example.items.list` y `example.refresh`. Para mantener inequívoco el espacio de nombres de permisos persistentes, OpenClaw convierte `%` y `.` del segmento del identificador del plugin en `%25` y `%2E`; los identificadores de plugin habituales conservan la forma natural. `paramShape` es un esquema JSON opcional que se aplica al objeto de parámetros de la acción antes de que OpenClaw invoque el RPC del plugin.

## Referencia del catálogo

`catalog` proporciona indicaciones de visualización opcionales a los exploradores de plugins. Los hosts pueden ignorar estas indicaciones. Nunca instalan ni habilitan el plugin, y no modifican su comportamiento en tiempo de ejecución ni su nivel de confianza.

```json
{
  "catalog": {
    "featured": true,
    "order": 10
  }
}
```

| Campo      | Tipo      | Significado                                                                |
| ---------- | --------- | -------------------------------------------------------------------------- |
| `featured` | `boolean` | Si las superficies del catálogo deben destacar este plugin.                |
| `order`    | `number`  | Indicación de orden ascendente entre los plugins seleccionados; los valores menores aparecen antes. |

## Referencia de metadatos de proveedores de generación

Los campos de metadatos de proveedores de generación describen señales estáticas de autenticación para los proveedores declarados en la lista `contracts.*GenerationProviders` correspondiente. OpenClaw lee estos campos antes de cargar el entorno de ejecución del proveedor, de modo que las herramientas del núcleo puedan determinar si un proveedor de generación está disponible sin importar todos los plugins de proveedores.

Estos campos deben usarse únicamente para datos declarativos de bajo coste. El transporte, las transformaciones de solicitudes, la renovación de tokens, la validación de credenciales y el comportamiento real de generación permanecen en el entorno de ejecución del plugin.

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

Cada entrada de metadatos admite:

| Campo                  | Obligatorio | Tipo       | Significado                                                                                                                                         |
| ---------------------- | ----------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`              | No          | `string[]` | Identificadores adicionales de proveedores que deben contar como alias estáticos de autenticación para el proveedor de generación.                  |
| `authProviders`        | No          | `string[]` | Identificadores de proveedores cuyos perfiles de autenticación configurados deben contar como autenticación para este proveedor de generación.      |
| `configSignals`        | No          | `object[]` | Señales de disponibilidad de bajo coste basadas únicamente en la configuración para proveedores locales o autoalojados que puedan configurarse sin perfiles de autenticación ni variables de entorno. |
| `authSignals`          | No          | `object[]` | Señales explícitas de autenticación. Cuando están presentes, reemplazan el conjunto predeterminado de señales del identificador del proveedor, `aliases` y `authProviders`. |
| `referenceAudioInputs` | No          | `boolean`  | Solo para generación de vídeo. Se establece en `true` cuando el proveedor acepta recursos de audio de referencia; de lo contrario, `video_generate` oculta los parámetros de referencia de audio. |

Cada entrada `configSignals` admite:

| Campo            | Obligatorio | Tipo       | Significado                                                                                                                                                                               |
| ---------------- | ----------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`       | Sí          | `string`   | Ruta con puntos al objeto de configuración propiedad del plugin que se debe inspeccionar, por ejemplo, `plugins.entries.example.config`.                                               |
| `overlayPath`    | No          | `string`   | Ruta con puntos dentro de la configuración raíz cuyo objeto debe superponerse al objeto raíz antes de evaluar la señal. Se usa para configuraciones específicas de una capacidad, como `image`, `video` o `music`. |
| `overlayMapPath` | No          | `string`   | Ruta con puntos dentro de la configuración raíz cuyos valores de objeto deben superponerse individualmente al objeto raíz. Se usa para mapas de cuentas con nombre, como `accounts`, donde cualquier cuenta configurada debe cumplir los requisitos. |
| `required`       | No          | `string[]` | Rutas con puntos dentro de la configuración efectiva que deben tener valores configurados. Las cadenas no deben estar vacías; los objetos y las matrices tampoco deben estar vacíos. |
| `requiredAny`    | No          | `string[]` | Rutas con puntos dentro de la configuración efectiva en las que al menos una debe tener un valor configurado.                                                                              |
| `mode`           | No          | `object`   | Restricción opcional de modo de cadena dentro de la configuración efectiva. Se usa cuando la disponibilidad basada únicamente en la configuración se aplica solo a un modo.                |

Cada restricción `mode` admite:

| Campo        | Obligatorio | Tipo       | Significado                                                                        |
| ------------ | ----------- | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | No          | `string`   | Ruta con puntos dentro de la configuración efectiva. El valor predeterminado es `mode`. |
| `default`    | No          | `string`   | Valor de modo que se debe usar cuando la configuración omite la ruta.               |
| `allowed`    | No          | `string[]` | Si está presente, la señal solo se acepta cuando el modo efectivo es uno de estos valores. |
| `disallowed` | No          | `string[]` | Si está presente, la señal falla cuando el modo efectivo es uno de estos valores.   |

Cada entrada `authSignals` admite:

| Campo             | Obligatorio | Tipo     | Significado                                                                                                                                                                   |
| ----------------- | ----------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Sí          | `string` | Identificador del proveedor que se debe comprobar en los perfiles de autenticación configurados.                                                                              |
| `providerBaseUrl` | No          | `object` | Restricción opcional que hace que la señal solo cuente cuando el proveedor configurado al que se hace referencia utiliza una URL base permitida. Se usa cuando un alias de autenticación solo es válido para determinadas API. |

Cada restricción `providerBaseUrl` admite:

| Campo             | Obligatorio | Tipo       | Significado                                                                                                                                          |
| ----------------- | ----------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Sí          | `string`   | Identificador de configuración del proveedor cuyo `baseUrl` debe comprobarse.                                                               |
| `defaultBaseUrl`  | No          | `string`   | URL base que se debe suponer cuando la configuración del proveedor omite `baseUrl`.                                                          |
| `allowedBaseUrls` | Sí          | `string[]` | URL base permitidas para esta señal de autenticación. La señal se ignora cuando la URL base configurada o predeterminada no coincide con uno de estos valores normalizados. |

## Referencia de metadatos de herramientas

`toolMetadata` utiliza las mismas estructuras `configSignals` y `authSignals` que los metadatos de proveedores de generación, indexadas por nombre de herramienta. `contracts.tools` declara la propiedad. `toolMetadata` declara evidencias de disponibilidad de bajo coste para que OpenClaw pueda evitar importar el entorno de ejecución de un plugin solo para que su fábrica de herramientas devuelva `null`.

```json
{
  "setup": {
    "providers": [
      {
        "id": "example",
        "envVars": ["EXAMPLE_API_KEY"]
      }
    ]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

Las entradas `toolMetadata` también aceptan `optional` (marca la herramienta como no obligatoria para la activación del plugin) y `replaySafe` (marca la ejecución de la herramienta como segura para repetirla tras un turno incompleto del modelo), además de los campos compartidos `configSignals`/`authSignals` anteriores.

Si una herramienta no tiene `toolMetadata`, OpenClaw conserva el comportamiento existente y carga el plugin propietario cuando el contrato de la herramienta coincide con la política. Para las herramientas de rutas críticas cuya fábrica depende de la autenticación/configuración, los autores de plugins deben declarar `toolMetadata` en lugar de hacer que el núcleo importe el entorno de ejecución para consultarlo.

## Referencia de providerAuthChoices

Cada entrada `providerAuthChoices` describe una opción de incorporación o autenticación. OpenClaw lee esta información antes de cargar el entorno de ejecución del proveedor. Las listas de configuración de proveedores utilizan estas opciones del manifiesto, las opciones de configuración derivadas de descriptores y los metadatos del catálogo de instalación sin cargar el entorno de ejecución del proveedor.

| Campo                 | Obligatorio | Tipo                                                                  | Significado                                                                                             |
| --------------------- | -------- | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `provider`            | Sí      | `string`                                                              | Id. del proveedor al que pertenece esta opción.                                                                       |
| `method`              | Sí      | `string`                                                              | Id. del método de autenticación al que se delegará.                                                                            |
| `choiceId`            | Sí      | `string`                                                              | Id. estable de la opción de autenticación utilizado por los flujos de incorporación y de la CLI.                                                   |
| `choiceLabel`         | No       | `string`                                                              | Etiqueta visible para el usuario. Si se omite, OpenClaw recurre a `choiceId`.                                         |
| `choiceHint`          | No       | `string`                                                              | Texto de ayuda breve para el selector.                                                                         |
| `icon`                | No       | URL HTTPS                                                             | Imagen mostrada junto a esta opción en los clientes de incorporación compatibles.                                         |
| `website`             | No       | URL HTTPS                                                             | Página del producto, de inicio de sesión o de instalación que muestran los clientes de incorporación compatibles.                             |
| `assistantPriority`   | No       | `number`                                                              | Los valores menores aparecen antes en los selectores interactivos controlados por el asistente.                                        |
| `assistantVisibility` | No       | `"visible"` \| `"manual-only"`                                        | Oculta la opción en los selectores del asistente, pero permite seleccionarla manualmente mediante la CLI.                         |
| `deprecatedChoiceIds` | No       | `string[]`                                                            | Id. de opciones heredadas que deben redirigir a los usuarios a esta opción de reemplazo.                                  |
| `groupId`             | No       | `string`                                                              | Id. de grupo opcional para agrupar opciones relacionadas.                                                           |
| `groupLabel`          | No       | `string`                                                              | Etiqueta visible para el usuario de ese grupo.                                                                         |
| `groupHint`           | No       | `string`                                                              | Texto de ayuda breve para el grupo.                                                                          |
| `onboardingFeatured`  | No       | `boolean`                                                             | Muestra este grupo en el nivel destacado del selector interactivo de incorporación, antes de la entrada "Más...". |
| `optionKey`           | No       | `string`                                                              | Clave de opción interna para flujos de autenticación simples con una sola bandera.                                                       |
| `cliFlag`             | No       | `string`                                                              | Nombre de la bandera de la CLI, como `--openrouter-api-key`.                                                            |
| `cliOption`           | No       | `string`                                                              | Forma completa de la opción de la CLI, como `--openrouter-api-key <key>`.                                              |
| `cliDescription`      | No       | `string`                                                              | Descripción utilizada en la ayuda de la CLI.                                                                             |
| `appGuidedSecret`     | No       | `boolean`                                                             | Un secreto pegado junto con los valores predeterminados del proveedor basta para la configuración guiada por la aplicación.                              |
| `appGuidedDiscovery`  | No       | `boolean`                                                             | El método de autenticación correspondiente del entorno de ejecución es responsable de la detección local de solo lectura mediante `appGuidedSetup`.                 |
| `appGuidedAuth`       | No       | `"oauth"` \| `"device-code"`                                          | Inicio de sesión interactivo administrado por el proveedor que los clientes de configuración nativos pueden representar de forma genérica.                        |
| `onboardingScopes`    | No       | `Array<"text-inference" \| "image-generation" \| "music-generation">` | Superficies de incorporación en las que debe aparecer esta opción. Si se omite, el valor predeterminado es `["text-inference"]`.  |

Cuando `appGuidedDiscovery` es verdadero, el método de autenticación del proveedor correspondiente debe exponer
`appGuidedSetup.detect` y `appGuidedSetup.prepare`. La detección debe ser
de solo lectura: no debe iniciar sesión, obtener modelos, descargar ni escribir la configuración. La preparación vuelve a comprobar
el modelo exacto seleccionado y devuelve una propuesta de configuración; OpenClaw prueba en vivo esa
propuesta de forma aislada y solo la confirma después de que tenga éxito.

## Referencia de commandAliases

Utilice `commandAliases` cuando un plugin sea propietario de un nombre de comando del entorno de ejecución que los usuarios puedan poner por error en `plugins.allow` o intentar ejecutar como comando raíz de la CLI. OpenClaw utiliza estos metadatos para los diagnósticos sin importar el código del entorno de ejecución del plugin.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Campo        | Obligatorio | Tipo              | Significado                                                           |
| ------------ | -------- | ----------------- | ----------------------------------------------------------------------- |
| `name`       | Sí      | `string`          | Nombre del comando que pertenece a este plugin.                               |
| `kind`       | No       | `"runtime-slash"` | Marca el alias como un comando de barra del chat en lugar de un comando raíz de la CLI. |
| `cliCommand` | No       | `string`          | Comando raíz relacionado de la CLI que se debe sugerir para las operaciones de la CLI, si existe alguno.  |

## Referencia de activation

Utilice `activation` cuando el plugin pueda declarar con poco coste qué eventos del plano de control deben incluirlo en un plan de activación/carga.

Este bloque contiene metadatos del planificador, no es una API de ciclo de vida. No registra el comportamiento del entorno de ejecución, no reemplaza a `register(...)` ni garantiza que el código del plugin ya se haya ejecutado. El planificador de activación utiliza estos campos para reducir los plugins candidatos antes de recurrir a los metadatos existentes de propiedad del manifiesto, como `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` y los hooks.

Utilice preferentemente los metadatos más específicos que ya describan la propiedad. Utilice `providers`, `channels`, `commandAliases`, los descriptores de configuración o `contracts` cuando esos campos expresen la relación. Utilice `activation` para indicaciones adicionales del planificador que no puedan representarse mediante esos campos de propiedad. Utilice `cliBackends` de nivel superior para los alias del entorno de ejecución de la CLI, como `claude-cli`, `my-cli` o `google-gemini-cli`; `activation.onAgentHarnesses` solo se utiliza para los identificadores del arnés de agente incorporado que aún no tengan un campo de propiedad.

Cada plugin debe configurar `activation.onStartup` intencionadamente. Establézcalo en `true` solo cuando el plugin deba ejecutarse durante el inicio del Gateway. Establézcalo en `false` cuando el plugin esté inactivo durante el inicio y solo deba cargarse mediante activadores más específicos. Omitir `onStartup` ya no carga implícitamente el plugin durante el inicio; utilice metadatos de activación explícitos para el inicio, el canal, la configuración, el arnés de agente, la memoria u otros activadores de activación más específicos.

```json
{
  "activation": {
    "onStartup": false,
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onConfigPaths": ["browser"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Campo              | Obligatorio | Tipo                                                 | Qué significa                                                                                                                                                                               |
| ------------------ | -------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStartup`        | No       | `boolean`                                            | Activación explícita durante el inicio del Gateway. Todos los plugins deben establecerla. `true` importa el plugin durante el inicio; `false` mantiene su carga diferida durante el inicio, salvo que otro desencadenador coincidente requiera cargarlo. |
| `onProviders`      | No       | `string[]`                                           | Id. de proveedores que deben incluir este plugin en los planes de activación/carga.                                                                                                                      |
| `onAgentHarnesses` | No       | `string[]`                                           | Id. de entornos de ejecución del arnés de agente integrado que deben incluir este plugin en los planes de activación/carga. Use `cliBackends` de nivel superior para los alias de backend de la CLI.                                           |
| `onCommands`       | No       | `string[]`                                           | Id. de comandos que deben incluir este plugin en los planes de activación/carga.                                                                                                                       |
| `onChannels`       | No       | `string[]`                                           | Id. de canales que deben incluir este plugin en los planes de activación/carga.                                                                                                                       |
| `onRoutes`         | No       | `string[]`                                           | Tipos de rutas que deben incluir este plugin en los planes de activación/carga.                                                                                                                       |
| `onConfigPaths`    | No       | `string[]`                                           | Rutas de configuración relativas a la raíz que deben incluir este plugin en los planes de inicio/carga cuando la ruta esté presente y no se haya deshabilitado explícitamente.                                                      |
| `onCapabilities`   | No       | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Indicaciones generales de capacidades utilizadas por la planificación de activación del plano de control. Cuando sea posible, prefiera campos más específicos.                                                                                     |

Consumidores activos actuales:

- La planificación del inicio del Gateway usa `activation.onStartup` para la importación explícita durante el inicio.
- La planificación de la CLI desencadenada por comandos recurre a los valores heredados `commandAliases[].cliCommand` o `commandAliases[].name`.
- La planificación del inicio del entorno de ejecución del agente usa `activation.onAgentHarnesses` para los arneses integrados y `cliBackends[]` de nivel superior para los alias de entorno de ejecución de la CLI.
- La planificación de la configuración/canal desencadenada por canales recurre a la propiedad heredada de `channels[]` cuando faltan metadatos explícitos de activación de canales.
- La planificación de plugins durante el inicio usa `activation.onConfigPaths` para superficies de configuración raíz que no sean de canales, como el bloque `browser` del plugin de navegador incluido.
- La planificación de la configuración/entorno de ejecución desencadenada por proveedores recurre a la propiedad heredada de `providers[]` y `cliBackends[]` de nivel superior cuando faltan metadatos explícitos de activación de proveedores.

Los diagnósticos del planificador pueden distinguir las indicaciones explícitas de activación del recurso a la propiedad del manifiesto. Por ejemplo, `activation-command-hint` significa que `activation.onCommands` coincidió, mientras que `manifest-command-alias` significa que el planificador utilizó en su lugar la propiedad de `commandAliases`. Estas etiquetas de motivos están destinadas a los diagnósticos y las pruebas del host; los autores de plugins deben seguir declarando los metadatos que mejor describan la propiedad.

## Referencia de qaRunners

Use `qaRunners` cuando un plugin aporte uno o varios ejecutores de transporte bajo
la raíz compartida `openclaw qa`. Mantenga estos metadatos ligeros y estáticos; el entorno de ejecución
del plugin sigue siendo responsable del registro real en la CLI mediante una superficie ligera
`runtime-api.ts` que exporta elementos `qaRunnerCliRegistrations` coincidentes. Un
`adapterFactory` opcional expone el transporte a escenarios de control de calidad compartidos sin
cambiar el ejecutor del comando registrado.

```json
{
  "qaRunners": [
    {
      "commandName": "matrix",
      "description": "Ejecutar el carril de control de calidad en vivo de Matrix respaldado por Docker contra un servidor doméstico desechable"
    }
  ]
}
```

| Campo         | Obligatorio | Tipo     | Qué significa                                                      |
| ------------- | -------- | -------- | ------------------------------------------------------------------ |
| `commandName` | Sí      | `string` | Subcomando montado bajo `openclaw qa`; por ejemplo, `matrix`.    |
| `description` | No       | `string` | Texto de ayuda alternativo utilizado cuando el host compartido necesita un comando provisional. |

El id. `adapterFactory` debe coincidir con `commandName`. No exporte registros
para comandos ausentes del manifiesto.

## Referencia de setup

Use `setup` cuando las superficies de configuración e incorporación necesiten metadatos ligeros propiedad del plugin antes de que se cargue el entorno de ejecución.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"],
        "authEvidence": [
          {
            "type": "local-file-with-env",
            "fileEnvVar": "OPENAI_CREDENTIALS_FILE",
            "requiresAllEnv": ["OPENAI_PROJECT"],
            "credentialMarker": "openai-local-credentials",
            "source": "credenciales locales de openai"
          }
        ]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

`cliBackends` de nivel superior sigue siendo válido y continúa describiendo los backends de inferencia de la CLI. `setup.cliBackends` es la superficie de descriptores específica de la configuración para los flujos del plano de control/configuración que deben limitarse a los metadatos.

Cuando están presentes, `setup.providers` y `setup.cliBackends` constituyen la superficie preferida de búsqueda basada primero en descriptores para el descubrimiento de la configuración. Si el descriptor solo acota el plugin candidato y la configuración aún necesita enlaces más completos del entorno de ejecución durante la configuración, establezca `requiresRuntime: true` y mantenga `setup-api` como ruta de ejecución alternativa.

OpenClaw incluye `setup.providers[].envVars` en las búsquedas genéricas de autenticación de proveedores y variables de entorno. Coloque allí los metadatos de entorno de configuración y estado.

Use `providerUsageAuthEnvVars` cuando una credencial de facturación o de nivel organizativo deba activar `resolveUsageAuth` sin convertirse en una credencial de inferencia. Estos nombres se incorporan al bloqueo de dotenv del espacio de trabajo, la eliminación en procesos secundarios de ACP, el filtrado de secretos del entorno aislado y la eliminación general de secretos. El entorno de ejecución del proveedor sigue leyendo y clasificando el valor dentro de `resolveUsageAuth`.

OpenClaw también puede derivar opciones sencillas de configuración a partir de `setup.providers[].authMethods` cuando no hay ninguna entrada de configuración disponible o cuando `setup.requiresRuntime: false` declara innecesario el entorno de ejecución de configuración. Las entradas explícitas de `providerAuthChoices` siguen siendo preferibles para etiquetas personalizadas, indicadores de la CLI, el ámbito de incorporación y los metadatos del asistente.

Establezca `requiresRuntime: false` solo cuando esos descriptores sean suficientes para la superficie de configuración. OpenClaw trata un `false` explícito como un contrato basado únicamente en descriptores y no ejecutará `setup-api` ni `openclaw.setupEntry` para la búsqueda de configuración. Si un plugin basado únicamente en descriptores todavía incluye una de esas entradas del entorno de ejecución de configuración, OpenClaw genera un diagnóstico adicional y continúa ignorándola. Si se omite `requiresRuntime`, se conserva el comportamiento alternativo heredado para no interrumpir los plugins existentes que añadieron descriptores sin el indicador.

Dado que la búsqueda de configuración puede ejecutar código `setup-api` propiedad del plugin, los valores normalizados de `setup.providers[].id` y `setup.cliBackends[]` deben ser únicos entre los plugins descubiertos. En caso de propiedad ambigua, el proceso se cierra de forma segura en lugar de elegir un ganador según el orden de descubrimiento.

Cuando se ejecuta el entorno de configuración, los diagnósticos del registro de configuración informan de divergencias respecto de los descriptores si `setup-api` registra un proveedor o un backend de la CLI que los descriptores del manifiesto no declaran, o si un descriptor no tiene un registro correspondiente en el entorno de ejecución. Estos diagnósticos son adicionales y no rechazan los plugins heredados.

### Referencia de setup.providers

| Campo          | Obligatorio | Tipo       | Qué significa                                                                                    |
| -------------- | -------- | ---------- | ------------------------------------------------------------------------------------------------ |
| `id`           | Sí      | `string`   | Id. del proveedor expuesto durante la configuración o la incorporación. Mantenga los id. normalizados globalmente únicos.             |
| `authMethods`  | No       | `string[]` | Id. de métodos de configuración/autenticación que admite este proveedor sin cargar todo el entorno de ejecución.                       |
| `envVars`      | No       | `string[]` | Variables de entorno que las superficies genéricas de configuración/estado pueden comprobar antes de que se cargue el entorno de ejecución del plugin.               |
| `authEvidence` | No       | `object[]` | Comprobaciones ligeras de indicios de autenticación local para proveedores que pueden autenticarse mediante marcadores no secretos. |

`authEvidence` se utiliza para marcadores de credenciales locales propiedad del proveedor que pueden verificarse sin cargar código del entorno de ejecución. Estas comprobaciones deben mantenerse ligeras y locales: sin llamadas de red, sin lecturas del llavero o del gestor de secretos, sin comandos de shell y sin sondeos de la API del proveedor.

Entradas de indicios compatibles:

| Campo              | Obligatorio | Tipo       | Qué significa                                                                                                  |
| ------------------ | -------- | ---------- | -------------------------------------------------------------------------------------------------------------- |
| `type`             | Sí      | `string`   | Actualmente `local-file-with-env`.                                                                               |
| `fileEnvVar`       | No       | `string`   | Variable de entorno que contiene una ruta explícita al archivo de credenciales.                                                           |
| `fallbackPaths`    | No       | `string[]` | Rutas de archivos de credenciales locales que se comprueban cuando `fileEnvVar` está ausente o vacío. Admite `${HOME}` y `${APPDATA}`. |
| `requiresAnyEnv`   | No       | `string[]` | Al menos una de las variables de entorno enumeradas debe tener un valor no vacío para que el indicio sea válido.                                    |
| `requiresAllEnv`   | No       | `string[]` | Todas las variables de entorno enumeradas deben tener un valor no vacío para que el indicio sea válido.                                           |
| `credentialMarker` | Sí      | `string`   | Marcador no secreto devuelto cuando el indicio está presente.                                                       |
| `source`           | No       | `string`   | Etiqueta de origen visible para el usuario en la salida de autenticación/estado.                                                               |

### Campos de setup

| Campo              | Obligatorio | Tipo       | Significado                                                                                       |
| ------------------ | -------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `providers`        | No       | `object[]` | Descriptores de configuración del proveedor expuestos durante la configuración y la incorporación.                                     |
| `cliBackends`      | No       | `string[]` | Identificadores de backend usados durante la configuración para la búsqueda que prioriza los descriptores. Mantenga los identificadores normalizados globalmente únicos. |
| `configMigrations` | No       | `string[]` | Identificadores de migración de configuración que pertenecen a la superficie de configuración de este plugin.                                          |
| `requiresRuntime`  | No       | `boolean`  | Indica si la configuración todavía requiere ejecutar `setup-api` después de buscar el descriptor.                            |

## Referencia de uiHints

`uiHints` es un mapa de nombres de campos de configuración a pequeñas indicaciones de representación. Las claves pueden usar puntos para campos de configuración anidados, pero ningún segmento de ruta puede ser `__proto__`, `constructor` ni `prototype`; la configuración rechaza esos nombres.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "Clave de API",
      "help": "Se usa para las solicitudes de OpenRouter",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Cada indicación de campo puede incluir:

| Campo          | Tipo             | Significado                                                                                                     |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| `label`        | `string`         | Etiqueta de campo visible para el usuario.                                                                                          |
| `help`         | `string`         | Texto de ayuda breve.                                                                                                |
| `tags`         | `string[]`       | Etiquetas opcionales de la interfaz de usuario.                                                                                                 |
| `advanced`     | `boolean`        | Marca el campo como avanzado.                                                                                      |
| `sensitive`    | `boolean`        | Marca el campo como secreto o confidencial.                                                                           |
| `placeholder`  | `string`         | Texto de marcador de posición para entradas de formulario.                                                                                 |
| `presentation` | `"phone-number"` | Formato telefónico localizado solo para visualización de valores internacionales analizables (`+...`); los valores sin procesar permanecen sin cambios. |

## Referencia de contracts

Use `contracts` únicamente para metadatos estáticos de pertenencia de capacidades que OpenClaw pueda leer sin importar el entorno de ejecución del plugin.

```json
{
  "contracts": {
    "agentToolResultMiddleware": ["openclaw", "codex"],
    "trustedToolPolicies": ["workflow-budget"],
    "externalAuthProviders": ["acme-ai"],
    "embeddingProviders": ["openai-compatible"],
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "memoryEmbeddingProviders": ["local"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "musicGenerationProviders": ["stability-audio"],
    "documentExtractors": ["example-docs"],
    "webContentExtractors": ["firecrawl"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "workerProviders": ["example-worker"],
    "usageProviders": ["acme-ai"],
    "migrationProviders": ["hermes"],
    "gatewayMethodDispatch": ["authenticated-request"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Cada lista es opcional:

| Campo                            | Tipo       | Significado                                                                                                                        |
| -------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `embeddedExtensionFactories`     | `string[]` | Identificadores de fábricas de extensiones del servidor de aplicaciones de Codex, actualmente `codex-app-server`.                                                                |
| `agentToolResultMiddleware`      | `string[]` | Identificadores de entornos de ejecución para los que este plugin puede registrar middleware de resultados de herramientas.                                                                     |
| `trustedToolPolicies`            | `string[]` | Identificadores locales del plugin de políticas de confianza previas a las herramientas que puede registrar un plugin instalado. Los plugins incluidos pueden registrar políticas sin este campo. |
| `externalAuthProviders`          | `string[]` | Identificadores de proveedores cuyo hook de perfiles de autenticación externa pertenece a este plugin.                                                                      |
| `embeddingProviders`             | `string[]` | Identificadores de proveedores generales de embeddings que pertenecen a este plugin para el uso reutilizable de embeddings vectoriales, incluida la memoria.                                 |
| `speechProviders`                | `string[]` | Identificadores de proveedores de voz que pertenecen a este plugin.                                                                                                |
| `realtimeTranscriptionProviders` | `string[]` | Identificadores de proveedores de transcripción en tiempo real que pertenecen a este plugin.                                                                                |
| `realtimeVoiceProviders`         | `string[]` | Identificadores de proveedores de voz en tiempo real que pertenecen a este plugin.                                                                                        |
| `memoryEmbeddingProviders`       | `string[]` | Identificadores obsoletos de proveedores de embeddings específicos de memoria que pertenecen a este plugin.                                                                  |
| `mediaUnderstandingProviders`    | `string[]` | Identificadores de proveedores de comprensión de contenido multimedia que pertenecen a este plugin.                                                                                   |
| `transcriptSourceProviders`      | `string[]` | Identificadores de proveedores de origen de transcripciones que pertenecen a este plugin.                                                                                     |
| `documentExtractors`             | `string[]` | Identificadores de proveedores de extracción de documentos (por ejemplo, PDF) que pertenecen a este plugin.                                                                  |
| `imageGenerationProviders`       | `string[]` | Identificadores de proveedores de generación de imágenes que pertenecen a este plugin.                                                                                      |
| `videoGenerationProviders`       | `string[]` | Identificadores de proveedores de generación de vídeo que pertenecen a este plugin.                                                                                      |
| `musicGenerationProviders`       | `string[]` | Identificadores de proveedores de generación de música que pertenecen a este plugin.                                                                                      |
| `webContentExtractors`           | `string[]` | Identificadores de proveedores de extracción de contenido de páginas web que pertenecen a este plugin.                                                                           |
| `webFetchProviders`              | `string[]` | Identificadores de proveedores de obtención web que pertenecen a este plugin.                                                                                             |
| `webSearchProviders`             | `string[]` | Identificadores de proveedores de búsqueda web que pertenecen a este plugin.                                                                                            |
| `workerProviders`                | `string[]` | Identificadores de proveedores de trabajadores en la nube que pertenecen a este plugin para el aprovisionamiento y el ciclo de vida de arrendamientos respaldados por perfiles.                                      |
| `usageProviders`                 | `string[]` | Identificadores de proveedores cuyos hooks de autenticación de uso e instantáneas de uso pertenecen a este plugin.                                                             |
| `migrationProviders`             | `string[]` | Identificadores de proveedores de importación que pertenecen a este plugin para `openclaw migrate`.                                                                         |
| `gatewayMethodDispatch`          | `string[]` | Derecho reservado para rutas HTTP autenticadas de plugins que despachan métodos del Gateway dentro del proceso.                                  |
| `tools`                          | `string[]` | Nombres de herramientas del agente que pertenecen a este plugin.                                                                                                   |

`contracts.embeddedExtensionFactories` se conserva para las fábricas de extensiones incluidas que son exclusivas del servidor de aplicaciones de Codex. Las transformaciones incluidas de resultados de herramientas deben declarar `contracts.agentToolResultMiddleware` y registrarse con `api.registerAgentToolResultMiddleware(...)`. Los plugins instalados pueden usar el mismo punto de integración de middleware únicamente cuando esté habilitado explícitamente y solo para los entornos de ejecución que declaren en `contracts.agentToolResultMiddleware`.

Los plugins instalados que necesiten el nivel de políticas previas a las herramientas en el que confía el host deben declarar cada identificador local registrado en `contracts.trustedToolPolicies` y habilitarse explícitamente. Los plugins incluidos conservan la ruta existente de políticas de confianza, pero los plugins instalados con identificadores de políticas no declarados se rechazan antes del registro. Los identificadores de políticas tienen como ámbito el plugin que los registra, por lo que dos plugins pueden declarar y registrar `workflow-budget`; un mismo plugin no puede registrar dos veces el mismo identificador local.

Los registros de `api.registerTool(...)` en tiempo de ejecución deben coincidir con `contracts.tools`. El descubrimiento de herramientas usa esta lista para cargar únicamente los entornos de ejecución de plugins que pueden poseer las herramientas solicitadas.

Los plugins de proveedores que implementen `resolveExternalAuthProfiles` deben declarar `contracts.externalAuthProviders`; se ignoran los hooks de autenticación externa no declarados.

Los plugins de proveedores que implementen tanto `resolveUsageAuth` como `fetchUsageSnapshot` deben declarar cada identificador de proveedor detectado automáticamente en `contracts.usageProviders`. El descubrimiento de uso lee este contrato antes de cargar el código de ejecución y, después de cargar únicamente a los propietarios declarados, verifica ambos hooks.

Los proveedores generales de embeddings deben declarar `contracts.embeddingProviders` para cada adaptador registrado con `api.registerEmbeddingProvider(...)`. Use el contrato general para la generación reutilizable de vectores, incluidos los proveedores que consume la búsqueda en memoria. `contracts.memoryEmbeddingProviders` es una compatibilidad obsoleta específica de memoria y se conserva únicamente mientras los proveedores existentes migran al punto de integración genérico para proveedores de embeddings.

Los proveedores de trabajadores deben declarar cada identificador `api.registerWorkerProvider(...)` en `contracts.workerProviders`. El núcleo conserva la intención duradera antes de llamar a `provision`; los proveedores validan su configuración antes de la asignación externa y las llamadas repetidas con el mismo identificador de operación deben adoptar el mismo arrendamiento. El núcleo también conserva esa instantánea de la configuración validada y la pasa con `leaseId` a `inspect({ leaseId, profile })` y `destroy({ leaseId, profile })`, incluso después de que se modifique o elimine el perfil indicado. La destrucción es idempotente, la inspección devuelve la unión cerrada de estados `active` / `destroyed` / `unknown`, y el material de la clave privada SSH solo se referencia mediante `SecretRef`. Los extremos SSH aprovisionados también deben incluir un `hostKey` público procedente de una salida de aprovisionamiento de confianza exactamente como `algorithm base64`, sin nombre de host ni comentario, para que el núcleo pueda fijar el host antes de conectarse. Los proveedores que generen referencias de identidad dinámicas pueden implementar el `resolveSshIdentity({ leaseId, profile, keyRef })` autoritativo; los proveedores que no lo hagan usan el solucionador genérico de secretos del núcleo. Un `unknown` autoritativo deja huérfano un registro local activo; después de una solicitud de destrucción conservada, confirma el desmontaje.

`contracts.gatewayMethodDispatch` actualmente acepta `"authenticated-request"`. Es una barrera de higiene de API para rutas HTTP de plugins nativos que despachan intencionadamente métodos del plano de control del Gateway dentro del proceso, no un entorno aislado contra plugins nativos maliciosos. Úselo únicamente para superficies integradas o del operador sometidas a una revisión rigurosa que ya requieran autenticación HTTP del Gateway. Una ruta autorizada sigue siendo accesible mientras la admisión de trabajo raíz del Gateway está cerrada solo cuando también declara `auth: "gateway"` y el `gatewayRuntimeScopeSurface: "trusted-operator"` específico de la ruta; las rutas hermanas ordinarias del mismo plugin permanecen detrás del límite de admisión. Esto mantiene accesibles el estado de suspensión y la reanudación sin conceder a todo el plugin una omisión de la admisión. Mantenga el análisis y la conformación de respuestas acotados fuera del despacho; el trabajo sustancial o que produzca modificaciones debe pasar por el despacho de métodos del Gateway, que es responsable de aplicar la admisión y el ámbito.

## Referencia de configContracts

Use `configContracts` para el comportamiento de configuración propiedad del manifiesto que necesitan los ayudantes genéricos del núcleo sin importar el entorno de ejecución del plugin: detección de indicadores peligrosos, destinos de migración de SecretRef y delimitación de rutas de configuración heredadas.

```json
{
  "configContracts": {
    "compatibilityMigrationPaths": ["legacyProvider"],
    "compatibilityRuntimePaths": ["legacyProvider.webhook"],
    "dangerousFlags": [
      {
        "path": "accounts.*.allowUnverifiedSenders",
        "equals": true
      }
    ],
    "secretInputs": {
      "bundledDefaultEnabled": false,
      "paths": [
        {
          "path": "routes.*.secret",
          "expected": "string",
          "ownerKind": "route"
        }
      ]
    }
  }
}
```

| Campo                         | Obligatorio | Tipo       | Significado                                                                                                                                                                                                                          |
| ----------------------------- | ----------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `compatibilityMigrationPaths` | No       | `string[]` | Rutas de configuración relativas a la raíz que indican que podrían aplicarse las migraciones de compatibilidad de este plugin durante la configuración. Permite que las lecturas genéricas de configuración en tiempo de ejecución omitan todas las superficies de configuración del plugin cuando la configuración nunca hace referencia al plugin. |
| `compatibilityRuntimePaths`   | No       | `string[]` | Rutas de compatibilidad relativas a la raíz que este plugin puede atender durante la ejecución antes de que el código del plugin se active por completo. Úselo para superficies heredadas que deban reducir los conjuntos de candidatos integrados sin importar el entorno de ejecución de cada plugin compatible. |
| `dangerousFlags`              | No       | `object[]` | Literales de configuración que `openclaw doctor` debe señalar como inseguros o peligrosos cuando están habilitados. Véase a continuación. |
| `secretInputs`                | No       | `object`   | Rutas de configuración bajo `plugins.entries.<id>.config` para la migración, auditoría y materialización durante el inicio de SecretRef, así como para el aislamiento opcional del propietario durante la ejecución. Véase a continuación. |

Cada entrada de `dangerousFlags` admite:

| Campo    | Obligatorio | Tipo                                  | Significado                                                                                                       |
| -------- | ----------- | ------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `path`   | Sí      | `string`                              | Ruta de configuración separada por puntos relativa a `plugins.entries.<id>.config`. Admite comodines `*` para segmentos de mapas o matrices. |
| `equals` | Sí      | `string \| number \| boolean \| null` | Literal exacto que marca este valor de configuración como peligroso. |

`secretInputs` admite:

| Campo                   | Obligatorio | Tipo       | Significado                                                                                                                                                                                                                                                                                                                                              |
| ----------------------- | ----------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bundledDefaultEnabled` | No       | `boolean`  | Sustituye la habilitación predeterminada del plugin integrado al decidir si esta superficie de SecretRef está activa. Úselo cuando el plugin esté integrado, pero la superficie deba permanecer inactiva hasta que se habilite explícitamente en la configuración. |
| `paths`                 | Sí      | `object[]` | Rutas de configuración con forma de secreto, cada una con `path` (separada por puntos, relativa a `plugins.entries.<id>.config`, admite comodines `*`), un `expected` opcional (actualmente solo `"string"`) y un `ownerKind` opcional (actualmente solo `"route"`). Un propietario declarado aísla únicamente esa ruta coincidente exacta cuando falla la resolución; su identificador de propietario es la ruta de configuración completa. |

## Referencia de mediaUnderstandingProviderMetadata

Use `mediaUnderstandingProviderMetadata` cuando un proveedor de comprensión multimedia tenga modelos predeterminados, una prioridad de reserva para la autenticación automática o compatibilidad nativa con documentos que los ayudantes genéricos del núcleo necesiten antes de cargar el entorno de ejecución. Las claves también deben declararse en `contracts.mediaUnderstandingProviders`.

```json
{
  "contracts": {
    "mediaUnderstandingProviders": ["example"]
  },
  "mediaUnderstandingProviderMetadata": {
    "example": {
      "capabilities": ["image", "audio"],
      "defaultModels": {
        "image": "example-vision-latest",
        "audio": "example-transcribe-latest"
      },
      "autoPriority": {
        "image": 40
      },
      "nativeDocumentInputs": ["pdf"],
      "documentModels": {
        "pdf": {
          "textExtraction": "example-doc-text-latest",
          "image": "example-doc-vision-latest"
        }
      }
    }
  }
}
```

Cada entrada de proveedor puede incluir:

| Campo                  | Tipo                                                             | Significado                                                                                                   |
| ---------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `capabilities`         | `("image" \| "audio" \| "video")[]`                              | Capacidades multimedia expuestas por este proveedor.                                                                    |
| `defaultModels`        | `Record<string, string>`                                         | Valores predeterminados de capacidad a modelo utilizados cuando la configuración no especifica un modelo.                                         |
| `autoPriority`         | `Record<string, number>`                                         | Los números más bajos se ordenan antes para la reserva automática de proveedores basada en credenciales.                                    |
| `nativeDocumentInputs` | `"pdf"[]`                                                        | Entradas de documentos nativas admitidas por el proveedor.                                                               |
| `documentModels`       | `{ pdf?: { textExtraction?: string; image?: string \| false } }` | Sustituciones de modelos por tipo de documento. Establezca `image: false` para deshabilitar la extracción basada en imágenes para ese tipo de documento. |

## Referencia de channelConfigs

Use `channelConfigs` cuando un plugin de canal necesite metadatos de configuración de bajo coste antes de cargar el entorno de ejecución. La detección de configuración o estado de canales en modo de solo lectura puede utilizar estos metadatos directamente para canales externos configurados cuando no haya disponible una entrada de configuración o cuando `setup.requiresRuntime: false` declare innecesario el entorno de ejecución de configuración.

`channelConfigs` son metadatos del manifiesto del plugin, no una nueva sección de configuración de usuario de nivel superior. Los usuarios siguen configurando instancias de canal bajo `channels.<channel-id>`. OpenClaw lee los metadatos del manifiesto para determinar qué plugin posee ese canal configurado antes de que se ejecute el código del entorno de ejecución del plugin.

Para un plugin de canal, `configSchema` y `channelConfigs` describen rutas diferentes:

- `configSchema` valida `plugins.entries.<plugin-id>.config`
- `channelConfigs.<channel-id>.schema` valida `channels.<channel-id>`

Los plugins no integrados que declaren `channels[]` también deben declarar entradas `channelConfigs` coincidentes. Sin ellas, OpenClaw aún puede cargar el plugin, pero las superficies del esquema de configuración de la ruta fría, de configuración y de la interfaz de control no pueden conocer la forma de las opciones propiedad del canal ni las indicaciones de interfaz destinadas únicamente a la visualización hasta que se ejecute el entorno de ejecución del plugin.

`channelConfigs.<channel-id>.commands.nativeCommandsAutoEnabled` y `nativeSkillsAutoEnabled` pueden declarar valores predeterminados estáticos de `auto` para las comprobaciones de configuración de comandos que se ejecutan antes de cargar el entorno de ejecución del canal. Los canales integrados también pueden publicar los mismos valores predeterminados mediante `package.json#openclaw.channel.commands`, junto con los demás metadatos de catálogo de canales propiedad del paquete.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "URL del servidor doméstico",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Conexión con el servidor doméstico de Matrix",
      "commands": {
        "nativeCommandsAutoEnabled": true,
        "nativeSkillsAutoEnabled": true
      },
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Cada entrada de canal puede incluir:

| Campo         | Tipo                     | Significado                                                                                                    |
| ------------- | ------------------------ | -------------------------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | Esquema JSON para `channels.<id>`. Obligatorio para cada entrada declarada de configuración de canal.                                |
| `uiHints`     | `Record<string, object>` | Etiquetas, marcadores de posición, sensibilidad e indicaciones opcionales de presentación destinadas únicamente a la visualización para esa sección de configuración del canal. |
| `label`       | `string`                 | Etiqueta del canal que se incorpora a las superficies de selección e inspección cuando los metadatos del entorno de ejecución no están listos. |
| `description` | `string`                 | Descripción breve del canal para las superficies de inspección y catálogo.                                                      |
| `commands`    | `object`                 | Valores predeterminados estáticos de activación automática de comandos nativos y Skills nativas para las comprobaciones de configuración previas al entorno de ejecución. |
| `preferOver`  | `string[]`               | Identificadores de plugins heredados o de menor prioridad que este canal debe superar en las superficies de selección.                           |

### Sustitución de otro plugin de canal

Use `preferOver` cuando su plugin sea el propietario preferido de un identificador de canal que también pueda proporcionar otro plugin. Los casos habituales son un identificador de plugin renombrado, un plugin independiente que sustituye a uno integrado o una bifurcación mantenida que conserva el mismo identificador de canal por compatibilidad con la configuración.

```json
{
  "id": "acme-chat",
  "channels": ["chat"],
  "channelConfigs": {
    "chat": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "webhookUrl": { "type": "string" }
        }
      },
      "preferOver": ["chat"]
    }
  }
}
```

Cuando se configura `channels.chat`, OpenClaw tiene en cuenta tanto el id del canal como el id del plugin preferido. Si el plugin de menor prioridad solo se seleccionó porque está incluido o habilitado de forma predeterminada, OpenClaw lo deshabilita en la configuración efectiva del entorno de ejecución para que un único plugin sea propietario del canal y de sus herramientas. La selección explícita del usuario sigue teniendo prioridad: si el usuario habilita explícitamente ambos plugins (mediante `plugins.allow` o una configuración `plugins.entries` sustancial), OpenClaw conserva esa elección y notifica diagnósticos de duplicación de canales o herramientas, en lugar de cambiar silenciosamente el conjunto de plugins solicitado.

Limite `preferOver` a los ids de plugins que realmente puedan proporcionar el mismo canal. No es un campo de prioridad general ni cambia el nombre de las claves de configuración del usuario.

## Referencia de modelSupport

Utilice `modelSupport` cuando OpenClaw deba inferir el plugin del proveedor a partir de ids abreviados de modelos como `gpt-5.6-sol` o `claude-sonnet-4.6` antes de que se cargue el entorno de ejecución del plugin.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw aplica esta precedencia:

- las referencias `provider/model` explícitas utilizan los metadatos del manifiesto `providers` propietario
- `modelPatterns` tienen prioridad sobre `modelPrefixes`
- si coinciden un plugin no incluido y uno incluido, tiene prioridad el plugin no incluido
- la ambigüedad restante se ignora hasta que el usuario o la configuración especifiquen un proveedor

Campos:

| Campo           | Tipo       | Significado                                                                   |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Prefijos que se comparan mediante `startsWith` con ids abreviados de modelos.                 |
| `modelPatterns` | `string[]` | Fuentes de expresiones regulares que se comparan con ids abreviados de modelos después de eliminar el sufijo del perfil. |

Las entradas `modelPatterns` se compilan mediante `compileSafeRegex`, que rechaza los patrones que contienen repeticiones anidadas (por ejemplo, `(a+)+$`). Los patrones que no superan la comprobación de seguridad se omiten silenciosamente, al igual que las expresiones regulares sintácticamente no válidas. Mantenga los patrones sencillos y evite los cuantificadores anidados.

## Referencia de modelCatalog

Utilice `modelCatalog` cuando OpenClaw deba conocer los metadatos de los modelos del proveedor antes de cargar el entorno de ejecución del plugin. Esta es la fuente propiedad del manifiesto para las filas fijas del catálogo, los alias de proveedores, las reglas de supresión y el modo de detección. La actualización en tiempo de ejecución sigue correspondiendo al código del entorno de ejecución del proveedor, pero el manifiesto indica al núcleo cuándo se requiere dicho entorno.

```json
{
  "providers": ["openai"],
  "modelCatalog": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "input": ["text", "image"],
            "reasoning": true,
            "contextWindow": 256000,
            "maxTokens": 128000,
            "cost": {
              "input": 1.25,
              "output": 10,
              "cacheRead": 0.125
            },
            "status": "available",
            "tags": ["default"]
          }
        ]
      }
    },
    "aliases": {
      "azure-openai-responses": {
        "provider": "openai",
        "api": "azure-openai-responses"
      }
    },
    "suppressions": [
      {
        "provider": "azure-openai-responses",
        "model": "gpt-5.3-codex-spark",
        "reason": "not available on Azure OpenAI Responses"
      }
    ],
    "discovery": {
      "openai": "static"
    }
  }
}
```

Campos de nivel superior:

| Campo            | Tipo                                                     | Significado                                                                                               |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `providers`      | `Record<string, object>`                                 | Filas del catálogo para los ids de proveedores que pertenecen a este plugin. Las claves también deben aparecer en `providers` en el nivel superior.       |
| `aliases`        | `Record<string, object>`                                 | Alias de proveedores que deben resolverse como un proveedor propio para planificar el catálogo o las supresiones.              |
| `suppressions`   | `object[]`                                               | Filas de modelos de otra fuente que este plugin suprime por un motivo específico del proveedor.                  |
| `discovery`      | `Record<string, "static" \| "refreshable" \| "runtime">` | Indica si el catálogo del proveedor puede leerse de los metadatos del manifiesto, actualizarse en la caché o requiere el entorno de ejecución. |
| `runtimeAugment` | `boolean`                                                | Se establece en `true` solo cuando el entorno de ejecución del proveedor debe añadir filas del catálogo después de planificar el manifiesto o la configuración.       |

`aliases` participa en la búsqueda de propiedad del proveedor para planificar el catálogo de modelos. Los destinos de los alias deben ser proveedores de nivel superior que pertenezcan al mismo plugin. Cuando una lista filtrada por proveedor utiliza un alias, OpenClaw puede leer el manifiesto propietario y aplicar las sustituciones de API o URL base del alias sin cargar el entorno de ejecución del proveedor. Los alias no amplían las listas de catálogos sin filtrar; las listas generales solo emiten las filas del proveedor canónico propietario.

`suppressions` sustituye el antiguo enlace `suppressBuiltInModel` del entorno de ejecución del proveedor. Las entradas de supresión solo se respetan cuando el proveedor pertenece al plugin o se declara como una clave `modelCatalog.aliases` que apunta a un proveedor propio. Los enlaces de supresión del entorno de ejecución ya no se invocan durante la resolución de modelos.

Campos del proveedor:

| Campo                 | Tipo                     | Significado                                                                                                                                                                                                     |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`             | `string`                 | URL base predeterminada opcional para los modelos de este catálogo de proveedor.                                                                                                                                                    |
| `api`                 | `ModelApi`               | Adaptador de API predeterminado opcional para los modelos de este catálogo de proveedor.                                                                                                                                                 |
| `headers`             | `Record<string, string>` | Encabezados estáticos opcionales que se aplican a este catálogo de proveedor.                                                                                                                                                      |
| `defaultUtilityModel` | `string`                 | Id opcional de un modelo pequeño recomendado por el proveedor para tareas internas breves (títulos, narración del progreso). Se utiliza cuando `agents.defaults.utilityModel` no está definido y este proveedor sirve el modelo principal del agente. |
| `models`              | `object[]`               | Filas de modelos obligatorias. Se ignoran las filas sin un `id`.                                                                                                                                                            |

Campos del modelo:

| Campo              | Tipo                                                           | Significado                                                               |
| ------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `id`               | `string`                                                       | Id del modelo local del proveedor, sin el prefijo `provider/`.                    |
| `name`             | `string`                                                       | Nombre para mostrar opcional.                                                      |
| `api`              | `ModelApi`                                                     | Sustitución opcional de la API por modelo.                                            |
| `baseUrl`          | `string`                                                       | Sustitución opcional de la URL base por modelo.                                       |
| `headers`          | `Record<string, string>`                                       | Encabezados estáticos opcionales por modelo.                                          |
| `input`            | `Array<"text" \| "image" \| "document">`                       | Modalidades que acepta el modelo. Los demás valores se descartan silenciosamente.            |
| `reasoning`        | `boolean`                                                      | Indica si el modelo ofrece capacidades de razonamiento.                               |
| `contextWindow`    | `number`                                                       | Ventana de contexto nativa del proveedor.                                             |
| `contextTokens`    | `number`                                                       | Límite de contexto efectivo opcional del entorno de ejecución cuando difiere de `contextWindow`. |
| `maxTokens`        | `number`                                                       | Número máximo de tokens de salida, cuando se conoce.                                           |
| `thinkingLevelMap` | `Record<string, string \| null>`                               | Sustituciones opcionales del id del modelo o de parámetros para cada nivel de razonamiento.                    |
| `cost`             | `object`                                                       | Precio opcional en USD por millón de tokens, incluido el valor opcional `tieredPricing`. |
| `compat`           | `object`                                                       | Indicadores de compatibilidad opcionales que coinciden con la compatibilidad de configuración de modelos de OpenClaw.  |
| `mediaInput`       | `object`                                                       | Configuración de entrada opcional por modalidad, actualmente solo para imágenes.                   |
| `status`           | `"available"` \| `"preview"` \| `"deprecated"` \| `"disabled"` | Estado en el listado. Suprima la fila solo cuando no deba aparecer en absoluto.          |
| `statusReason`     | `string`                                                       | Motivo opcional que se muestra cuando el estado no es disponible.                            |
| `replaces`         | `string[]`                                                     | Ids antiguos de modelos locales del proveedor a los que sustituye este modelo.                       |
| `replacedBy`       | `string`                                                       | Id del modelo local del proveedor de reemplazo para las filas obsoletas.                    |
| `tags`             | `string[]`                                                     | Etiquetas estables utilizadas por los selectores y filtros.                                    |

Campos de supresión:

| Campo                      | Tipo       | Qué significa                                                                                             |
| -------------------------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`   | Id. del proveedor de la fila ascendente que se debe suprimir. Debe pertenecer a este plugin o declararse como un alias propio. |
| `model`                    | `string`   | Id. de modelo local del proveedor que se debe suprimir.                                                                      |
| `reason`                   | `string`   | Mensaje opcional que se muestra cuando se solicita directamente la fila suprimida.                                     |
| `when.baseUrlHosts`        | `string[]` | Lista opcional de hosts de URL base efectivas del proveedor necesarios para que se aplique la supresión.               |
| `when.providerConfigApiIn` | `string[]` | Lista opcional de valores exactos `api` de configuración del proveedor necesarios para que se aplique la supresión.              |

No coloque datos exclusivos del entorno de ejecución en `modelCatalog`. Use `static` solo cuando las filas del manifiesto sean lo bastante completas como para que las superficies de lista y selección filtradas por proveedor omitan la detección del registro y del entorno de ejecución. Use `refreshable` cuando las filas del manifiesto sean datos iniciales enumerables o complementos útiles, pero una actualización o caché pueda añadir más filas posteriormente; las filas actualizables no son autoritativas por sí solas. Use `runtime` cuando OpenClaw deba cargar el entorno de ejecución del proveedor para conocer la lista.

## Referencia de modelIdNormalization

Use `modelIdNormalization` para la normalización sencilla de los id. de modelo propiedad del proveedor que debe realizarse antes de cargar el entorno de ejecución del proveedor. Esto mantiene en el manifiesto del plugin propietario los alias, como nombres cortos de modelos, id. heredados locales del proveedor y reglas de prefijos de proxy, en lugar de incluirlos en las tablas principales de selección de modelos.

```json
{
  "providers": ["anthropic", "openrouter"],
  "modelIdNormalization": {
    "providers": {
      "anthropic": {
        "aliases": {
          "sonnet-4.6": "claude-sonnet-4-6"
        }
      },
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  }
}
```

Campos del proveedor:

| Campo                                | Tipo                    | Qué significa                                                                             |
| ------------------------------------ | ----------------------- | ----------------------------------------------------------------------------------------- |
| `aliases`                            | `Record<string,string>` | Alias exactos de id. de modelo sin distinción entre mayúsculas y minúsculas. Los valores se devuelven tal como están escritos.                  |
| `stripPrefixes`                      | `string[]`              | Prefijos que se deben eliminar antes de buscar alias, útiles para la duplicación heredada de proveedor/modelo.     |
| `prefixWhenBare`                     | `string`                | Prefijo que se debe añadir cuando el id. de modelo normalizado aún no contiene `/`.                  |
| `prefixWhenBareAfterAliasStartsWith` | `object[]`              | Reglas condicionales de prefijo para id. simples después de buscar alias, indexadas por `modelPrefix` y `prefix`. |

## Referencia de providerEndpoints

Use `providerEndpoints` para la clasificación de puntos de conexión que la política genérica de solicitudes debe conocer antes de cargar el entorno de ejecución del proveedor. El núcleo sigue siendo propietario del significado de cada `endpointClass`; los manifiestos de los plugins son propietarios de los metadatos del host y de la URL base.

Los plugins de proveedores externalizados oficialmente se excluyen de la distribución principal, por lo que
sus manifiestos no son visibles hasta que se instalan. Sus `providerEndpoints` también deben
reflejarse en `scripts/lib/official-external-provider-catalog.json` para que
la clasificación de puntos de conexión siga funcionando sin el plugin; una prueba de contrato
garantiza esta correspondencia.

Campos del punto de conexión:

| Campo                          | Tipo       | Qué significa                                                                                  |
| ------------------------------ | ---------- | ---------------------------------------------------------------------------------------------- |
| `endpointClass`                | `string`   | Clase de punto de conexión conocida por el núcleo, como `openrouter`, `moonshot-native` o `google-vertex`.        |
| `hosts`                        | `string[]` | Nombres de host exactos que se asignan a la clase de punto de conexión.                                                |
| `hostSuffixes`                 | `string[]` | Sufijos de host que se asignan a la clase de punto de conexión. Anteponer `.` para que la coincidencia se limite a sufijos de dominio. |
| `baseUrls`                     | `string[]` | URL base HTTP(S) normalizadas exactas que se asignan a la clase de punto de conexión.                             |
| `googleVertexRegion`           | `string`   | Región estática de Google Vertex para hosts globales exactos.                                            |
| `googleVertexRegionHostSuffix` | `string`   | Sufijo que se debe eliminar de los hosts coincidentes para exponer el prefijo de región de Google Vertex.                 |

## Referencia de providerRequest

Use `providerRequest` para los metadatos sencillos de compatibilidad de solicitudes que necesita la política genérica de solicitudes sin cargar el entorno de ejecución del proveedor. Mantenga la reescritura de cargas útiles específica del comportamiento en los hooks del entorno de ejecución del proveedor o en los asistentes compartidos de la familia de proveedores.

```json
{
  "providerRequest": {
    "providers": {
      "vllm": {
        "family": "vllm",
        "openAICompletions": {
          "supportsStreamingUsage": true
        }
      }
    }
  }
}
```

Campos del proveedor:

| Campo                 | Tipo         | Qué significa                                                                          |
| --------------------- | ------------ | -------------------------------------------------------------------------------------- |
| `family`              | `string`     | Etiqueta de la familia del proveedor utilizada por las decisiones genéricas de compatibilidad de solicitudes y los diagnósticos. |
| `compatibilityFamily` | `"moonshot"` | Categoría opcional de compatibilidad de la familia de proveedores para asistentes de solicitudes compartidos.              |
| `openAICompletions`   | `object`     | Indicadores de solicitudes de finalización compatibles con OpenAI, actualmente `supportsStreamingUsage`.       |

## Referencia de secretProviderIntegrations

Use `secretProviderIntegrations` cuando un plugin pueda publicar un preajuste reutilizable de proveedor de ejecución SecretRef. OpenClaw lee estos metadatos antes de cargar el entorno de ejecución del plugin, almacena la propiedad del plugin en `secrets.providers.<alias>.pluginIntegration` y deja la resolución real de secretos al entorno de ejecución de SecretRef. Los preajustes solo se exponen para plugins incluidos e instalados que se detectan en las raíces administradas de instalación de plugins, como las instalaciones desde git y ClawHub.

```json
{
  "secretProviderIntegrations": {
    "secret-store": {
      "providerAlias": "team-secrets",
      "displayName": "Team secrets",
      "source": "exec",
      "command": "${node}",
      "args": ["./bin/resolve-secrets.mjs"]
    }
  }
}
```

La clave del mapa es el id. de integración. Si se omite `providerAlias`, OpenClaw usa el id. de integración como alias del proveedor SecretRef. Los alias de proveedores deben coincidir con el patrón normal de alias de proveedores SecretRef, por ejemplo, `team-secrets` o `onepassword-work`.

Cuando un operador selecciona el preajuste, OpenClaw escribe una referencia de proveedor como la siguiente:

```json
{
  "secrets": {
    "providers": {
      "team-secrets": {
        "source": "exec",
        "pluginIntegration": {
          "pluginId": "acme-secrets",
          "integrationId": "secret-store"
        }
      }
    }
  }
}
```

Durante el inicio o la recarga, OpenClaw resuelve ese proveedor cargando los metadatos actuales del manifiesto del plugin, comprobando que el plugin propietario esté instalado y activo, y materializando el comando de ejecución a partir del manifiesto. Deshabilitar o eliminar el plugin revoca el proveedor para las SecretRefs activas. Los operadores que quieran una configuración de ejecución independiente pueden seguir escribiendo directamente proveedores manuales `command`/`args`.

Actualmente solo se admiten preajustes `source: "exec"`. `command` debe ser `${node}` y `args[0]` debe ser un script de resolución `./` relativo a la raíz del plugin. OpenClaw lo materializa durante el inicio o la recarga como el ejecutable actual de Node y la ruta absoluta del script dentro del plugin. Las opciones de Node como `--require`, `--import`, `--loader`, `--env-file`, `--eval` y `--print` no forman parte del contrato de preajustes del manifiesto. Los operadores que necesiten comandos que no sean de Node pueden configurar directamente proveedores de ejecución manuales independientes.

OpenClaw obtiene `trustedDirs` para los preajustes del manifiesto a partir de la raíz del plugin y, para los preajustes `${node}`, del directorio actual del ejecutable de Node. Los `trustedDirs` definidos en el manifiesto se ignoran. Otras opciones del proveedor de ejecución, como `timeoutMs`, `noOutputTimeoutMs`, `maxOutputBytes`, `jsonOnly`, `env`, `passEnv` y `allowInsecurePath`, se transfieren a la configuración normal del proveedor de ejecución SecretRef.

## Referencia de modelPricing

Use `modelPricing` cuando un proveedor necesite controlar el comportamiento de precios del plano de control antes de cargar el entorno de ejecución. La caché de precios del Gateway lee estos metadatos sin importar el código del entorno de ejecución del proveedor.

```json
{
  "providers": ["ollama", "openrouter"],
  "modelPricing": {
    "providers": {
      "ollama": {
        "external": false
      },
      "openrouter": {
        "openRouter": {
          "passthroughProviderModel": true
        },
        "liteLLM": false
      }
    }
  }
}
```

Campos del proveedor:

| Campo        | Tipo              | Qué significa                                                                                      |
| ------------ | ----------------- | -------------------------------------------------------------------------------------------------- |
| `external`   | `boolean`         | Establezca `false` para proveedores locales o autoalojados que nunca deban obtener precios de OpenRouter ni LiteLLM. |
| `openRouter` | `false \| object` | Asignación de búsqueda de precios de OpenRouter. `false` deshabilita la búsqueda en OpenRouter para este proveedor.           |
| `liteLLM`    | `false \| object` | Asignación de búsqueda de precios de LiteLLM. `false` deshabilita la búsqueda en LiteLLM para este proveedor.                 |

Campos de origen:

| Campo                      | Tipo               | Qué significa                                                                                                        |
| -------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`           | Id. del proveedor del catálogo externo cuando difiere del id. de proveedor de OpenClaw, por ejemplo, `z-ai` para un proveedor `zai`. |
| `passthroughProviderModel` | `boolean`          | Tratar los id. de modelo que contienen barras como referencias anidadas de proveedor/modelo, lo que resulta útil para proveedores proxy como OpenRouter.       |
| `modelIdTransforms`        | `"version-dots"[]` | Variantes adicionales de id. de modelo del catálogo externo. `version-dots` prueba id. de versión con puntos como `claude-opus-4.6`.            |

### Índice de proveedores de OpenClaw

El Índice de proveedores de OpenClaw son metadatos de vista previa propiedad de OpenClaw para proveedores cuyos plugins quizá aún no estén instalados. No forma parte de un manifiesto de plugin. Los manifiestos de los plugins siguen siendo la autoridad para los plugins instalados. El Índice de proveedores es el contrato de respaldo interno que utilizarán las futuras superficies de proveedores instalables y de selección de modelos previa a la instalación cuando no esté instalado un plugin de proveedor.

Orden de autoridad del catálogo:

1. Configuración del usuario.
2. `modelCatalog` del manifiesto del plugin instalado.
3. Caché del catálogo de modelos procedente de una actualización explícita.
4. Filas de vista previa del Índice de proveedores de OpenClaw.

El índice de proveedores no debe contener secretos, estado habilitado, hooks de runtime ni datos activos de modelos específicos de una cuenta. Sus catálogos de vista previa usan la misma forma de fila de proveedor `modelCatalog` que los manifiestos de plugins, pero deben limitarse a metadatos de visualización estables, salvo que campos del adaptador de runtime como `api`, `baseUrl`, precios o indicadores de compatibilidad se mantengan intencionadamente alineados con el manifiesto del plugin instalado. Los proveedores con detección activa de `/models` deben escribir las filas actualizadas mediante la ruta explícita de caché del catálogo de modelos, en lugar de hacer que el listado normal o la incorporación llamen a las API del proveedor.

Las entradas del índice de proveedores también pueden incluir metadatos de plugins instalables para proveedores cuyo plugin se haya trasladado fuera del núcleo o que aún no esté instalado por algún otro motivo. Estos metadatos siguen el patrón del catálogo de canales: el nombre del paquete, la especificación de instalación de npm, la integridad esperada y etiquetas sencillas de opciones de autenticación bastan para mostrar una opción de configuración instalable. Una vez instalado el plugin, prevalece su manifiesto y se ignora la entrada del índice de proveedores correspondiente a ese proveedor.

`openclaw doctor --fix` migra un conjunto pequeño y cerrado de claves de capacidades heredadas del nivel superior del manifiesto a `contracts.*`: `speechProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders` y `tools`. Ninguna de ellas (ni ninguna otra lista de capacidades) se lee ya como campo del nivel superior del manifiesto; la carga normal de manifiestos solo las reconoce bajo `contracts`.

## Manifiesto frente a package.json

Los dos archivos cumplen funciones diferentes:

| Archivo                   | Se utiliza para                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Detección, validación de configuración, metadatos de opciones de autenticación e indicaciones de la interfaz de usuario que deben existir antes de ejecutar el código del plugin                         |
| `package.json`         | Metadatos de npm, instalación de dependencias y el bloque `openclaw` utilizado para puntos de entrada, restricciones de instalación, configuración o metadatos del catálogo |

Si no se sabe con certeza dónde corresponde un elemento de metadatos, debe aplicarse esta regla:

- si OpenClaw debe conocerlo antes de cargar el código del plugin, debe colocarse en `openclaw.plugin.json`
- si está relacionado con el empaquetado, los archivos de entrada o el comportamiento de instalación de npm, debe colocarse en `package.json`

### Campos de package.json que afectan a la detección

Algunos metadatos de plugins anteriores al runtime se encuentran intencionadamente en `package.json`, bajo el bloque `openclaw`, en lugar de en `openclaw.plugin.json`. `openclaw.bundle` y `openclaw.bundle.json` no son contratos de plugins de OpenClaw; los plugins nativos deben usar `openclaw.plugin.json` junto con los campos `package.json#openclaw` compatibles que se indican a continuación.

Ejemplos importantes:

| Campo                                                                                      | Significado                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                                                      | Declara puntos de entrada de plugins nativos. Deben permanecer dentro del directorio del paquete del plugin.                                                                                                        |
| `openclaw.runtimeExtensions`                                                               | Declara puntos de entrada compilados del runtime de JavaScript para los paquetes instalados. Deben permanecer dentro del directorio del paquete del plugin.                                                                      |
| `openclaw.setupEntry`                                                                      | Punto de entrada ligero exclusivo para la configuración, utilizado durante la incorporación, el inicio diferido de canales y la detección de estados de canales o SecretRef de solo lectura. Debe permanecer dentro del directorio del paquete del plugin.      |
| `openclaw.runtimeSetupEntry`                                                               | Declara el punto de entrada de configuración de JavaScript compilado para los paquetes instalados. Requiere `setupEntry`, debe existir y debe permanecer dentro del directorio del paquete del plugin.                              |
| `openclaw.channel`                                                                         | Metadatos sencillos del catálogo de canales, como etiquetas, rutas de documentación, alias y texto de selección.                                                                                                      |
| `openclaw.channel.approvalFlags`                                                           | Indicadores cerrados del comportamiento de aprobación disponibles antes de cargar el runtime. `native` significa que el canal controla la interfaz de usuario nativa de aprobación y la resolución en el mismo turno.                                                |
| `openclaw.channel.commands`                                                                | Metadatos estáticos de valores predeterminados automáticos para comandos y Skills nativas, utilizados por las superficies de configuración, auditoría y listas de comandos antes de cargar el runtime del canal.                                               |
| `openclaw.channel.cliAddOptions`                                                           | Opciones de `openclaw channels add` controladas por el plugin. Cada entrada declara `flags`, `description`, un `defaultValue` opcional y un `valueType` opcional (`int` o `list`) para la conversión genérica de entradas. |
| `openclaw.channel.configuredState`                                                         | Metadatos ligeros del comprobador de estado configurado que pueden responder «¿ya existe una configuración basada únicamente en variables de entorno?» sin cargar el runtime completo del canal.                                              |
| `openclaw.channel.persistedAuthState`                                                      | Metadatos ligeros del comprobador de autenticación persistida que pueden responder «¿ya hay alguna sesión iniciada?» sin cargar el runtime completo del canal.                                                    |
| `openclaw.install.clawhubSpec` / `openclaw.install.npmSpec` / `openclaw.install.localPath` | Indicaciones de instalación y actualización para plugins incluidos y publicados externamente.                                                                                                                        |
| `openclaw.install.defaultChoice`                                                           | Ruta de instalación preferida cuando hay varias fuentes de instalación disponibles.                                                                                                                       |
| `openclaw.install.minHostVersion`                                                          | Versión mínima compatible del host OpenClaw, mediante un límite inferior de semver como `>=2026.3.22` o `>=2026.5.1-beta.1`.                                                                                  |
| `openclaw.compat.pluginApi`                                                                | Intervalo mínimo requerido de la API de plugins de OpenClaw para este paquete, mediante un límite inferior de semver como `>=2026.5.27`.                                                                                      |
| `openclaw.install.expectedIntegrity`                                                       | Cadena esperada de integridad de la distribución de npm, como `sha512-...`; los flujos de instalación y actualización verifican con ella el artefacto obtenido.                                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                                              | Permite una ruta limitada de recuperación mediante reinstalación de un plugin incluido cuando la configuración no es válida.                                                                                                            |
| `openclaw.install.requiredPlatformPackages`                                                | Alias de paquetes npm que deben materializarse cuando las restricciones de plataforma de su archivo de bloqueo coincidan con el host actual.                                                                                |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`                          | Permite cargar las superficies de canal del runtime de configuración antes de la escucha y, posteriormente, aplaza el plugin completo del canal configurado hasta su activación posterior al inicio de la escucha.                                                      |

Los metadatos del manifiesto determinan qué opciones de proveedor, canal y configuración aparecen durante la incorporación antes de cargar el runtime. `package.json#openclaw.install` indica a la incorporación cómo obtener o habilitar ese plugin cuando se selecciona una de esas opciones. No deben trasladarse las indicaciones de instalación a `openclaw.plugin.json`.

Para `openclaw.channel.cliAddOptions`, debe usarse la sintaxis de opciones largas de Commander, como `--initial-sync-limit <n>`. Debe establecerse `valueType: "int"` para analizar un entero no negativo o `valueType: "list"` para dividir una entrada delimitada por comas, puntos y comas o saltos de línea en cadenas antes de que la reciba el adaptador de configuración del plugin. Debe omitirse `valueType` para transmitir sin cambios el valor analizado por Commander.

`openclaw.install.minHostVersion` se aplica durante la instalación y la carga del registro de manifiestos para fuentes de plugins no incluidos. Los valores no válidos se rechazan; los valores válidos pero más recientes hacen que se omitan los plugins externos en hosts anteriores. Se presupone que los plugins de origen incluidos tienen la misma versión que el checkout del host.

`openclaw.install.requiredPlatformPackages` está destinado a paquetes npm que exponen los binarios nativos necesarios mediante alias opcionales específicos de cada plataforma. Debe indicarse el nombre simple del paquete npm para cada alias de plataforma compatible. Durante la instalación de npm, OpenClaw solo verifica el alias declarado cuyas restricciones del archivo de bloqueo coincidan con el host actual. Si npm informa de que la operación se completó correctamente, pero omite ese alias, OpenClaw vuelve a intentarlo una vez con una caché nueva y revierte la instalación si el alias continúa ausente.

`openclaw.compat.pluginApi` se aplica durante la instalación de paquetes para fuentes de plugins no incluidos. Debe usarse para indicar el límite inferior de la API del SDK/runtime de plugins de OpenClaw con la que se compiló el paquete. Puede ser más estricto que `minHostVersion` cuando un paquete de plugin necesita una API más reciente, pero mantiene una indicación de instalación inferior para otros flujos. De forma predeterminada, la sincronización de versiones oficiales de OpenClaw eleva los límites inferiores existentes de la API de los plugins oficiales a la versión de OpenClaw, pero las versiones exclusivas de plugins pueden mantener un límite inferior cuando el paquete admite intencionadamente hosts anteriores. No debe usarse únicamente la versión del paquete como contrato de compatibilidad. `peerDependencies.openclaw` sigue siendo un metadato de paquete npm; OpenClaw utiliza el contrato `openclaw.compat.pluginApi` para tomar decisiones de compatibilidad de instalación.

Los metadatos oficiales de instalación bajo demanda deben usar `clawhubSpec` cuando el plugin esté publicado en ClawHub; la incorporación lo considera la fuente remota preferida y registra los datos del artefacto de ClawHub después de la instalación. `npmSpec` sigue siendo la alternativa de compatibilidad para los paquetes que todavía no se han trasladado a ClawHub.

La fijación exacta de versiones de npm ya se encuentra en `npmSpec`; por ejemplo, `"npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3"`. Las entradas oficiales de catálogos externos deben asociar especificaciones exactas con `expectedIntegrity` para que los flujos de actualización fallen de forma segura si el artefacto de npm obtenido ya no coincide con la versión fijada. Para mantener la compatibilidad, la incorporación interactiva sigue ofreciendo especificaciones de npm procedentes de registros de confianza, incluidos nombres de paquetes simples y etiquetas de distribución. Los diagnósticos del catálogo pueden distinguir entre fuentes exactas, variables, fijadas por integridad, sin integridad, con discrepancia del nombre del paquete y con una opción predeterminada no válida. También advierten cuando `expectedIntegrity` está presente, pero no existe una fuente de npm válida que pueda fijar. Cuando `expectedIntegrity` está presente, los flujos de instalación y actualización lo aplican; cuando se omite, la resolución del registro se registra sin una fijación de integridad.

Los plugins de canales deben proporcionar `openclaw.setupEntry` cuando las exploraciones de estado, lista de canales o SecretRef necesiten identificar cuentas configuradas sin cargar el runtime completo. La entrada de configuración debe exponer los metadatos del canal, además de adaptadores de configuración, estado y secretos seguros para la configuración; los clientes de red, los listeners del Gateway y los runtimes de transporte deben mantenerse en el punto de entrada principal de la extensión.

Los campos del punto de entrada del entorno de ejecución no anulan las comprobaciones de los límites del paquete para los campos del punto de entrada de origen. Por ejemplo, `openclaw.runtimeExtensions` no puede hacer que se pueda cargar una ruta `openclaw.extensions` que salga del paquete.

`openclaw.install.allowInvalidConfigRecovery` es intencionadamente limitado. No permite instalar configuraciones dañadas arbitrarias. Actualmente, solo permite que los flujos de instalación se recuperen de determinados fallos de actualización obsoletos de plugins incluidos, como la ausencia de una ruta de un plugin incluido o una entrada `channels.<id>` obsoleta para ese mismo plugin incluido. Los errores de configuración no relacionados siguen bloqueando la instalación y remiten a los operadores a `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` son metadatos de paquete para un pequeño módulo de comprobación:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Se usa cuando los flujos de configuración, Doctor, estado o comprobación de presencia de solo lectura necesitan una consulta rápida de autenticación de tipo sí/no antes de que se cargue el plugin completo del canal. El estado de autenticación persistente no es el estado configurado del canal: no se deben usar estos metadatos para activar plugins automáticamente, reparar dependencias del entorno de ejecución ni decidir si debe cargarse el entorno de ejecución de un canal. La exportación de destino debe ser una función pequeña que solo lea el estado persistente; no se debe encaminar a través del barrel completo del entorno de ejecución del canal.

`openclaw.channel.configuredState` admite comprobaciones rápidas de configuración. Se deben preferir los metadatos declarativos de entorno cuando las variables de entorno sean suficientes:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "env": {
          "allOf": ["TELEGRAM_BOT_TOKEN"]
        }
      }
    }
  }
}
```

Se usa `env.allOf` cuando se requieren todas las variables enumeradas y `env.anyOf` cuando basta con cualquier variable no vacía. Si una pequeña comprobación ajena al entorno de ejecución necesita más que metadatos de entorno, se usan `specifier` y `exportName`, como se muestra para `persistedAuthState`; cuando está presente `env`, OpenClaw lo usa sin cargar ese módulo. Si la comprobación necesita la resolución completa de la configuración o el entorno de ejecución real del canal, esa lógica debe mantenerse en el hook `config.hasConfiguredState` del plugin.

## Precedencia de descubrimiento (identificadores de plugin duplicados)

OpenClaw descubre plugins en tres raíces, comprobadas en este orden: los plugins incluidos distribuidos con OpenClaw, la raíz de instalación global (`~/.openclaw/extensions`) y la raíz del espacio de trabajo actual (`<workspace>/.openclaw/extensions`), además de cualquier entrada `plugins.load.paths` explícita.

Si dos descubrimientos comparten el mismo `id`, solo se conserva el manifiesto con la **mayor precedencia**; los duplicados con menor precedencia se descartan en lugar de cargarse junto a él. Precedencia, de mayor a menor:

1. **Seleccionado por la configuración** — una ruta fijada explícitamente en `plugins.entries.<id>`
2. **Instalación global que coincide con un registro de instalación rastreado** — un plugin instalado mediante `openclaw plugin install`/`openclaw plugin update` que el seguimiento de instalaciones de OpenClaw reconoce para ese mismo identificador, incluso cuando el identificador también pertenece a un plugin incluido
3. **Incluido** — plugins distribuidos con OpenClaw
4. **Espacio de trabajo** — plugins descubiertos en relación con el espacio de trabajo actual
5. Cualquier otro candidato descubierto

Implicaciones:

- Una bifurcación o copia obsoleta de un plugin incluido que se encuentre sin rastrear en el espacio de trabajo o en la raíz global no sustituirá a la compilación incluida.
- Para sustituir un plugin incluido, se debe ejecutar `openclaw plugin install` para ese identificador, de modo que la instalación global rastreada tenga prioridad sobre la copia incluida, o fijar una ruta específica mediante `plugins.entries.<id>` para que prevalezca por la precedencia de selección mediante configuración.
- Los descartes de duplicados se registran para que Doctor y los diagnósticos de inicio puedan señalar la copia descartada.
- Las sustituciones de duplicados seleccionadas por la configuración se describen como sustituciones explícitas en los diagnósticos, pero aun así generan una advertencia para que las bifurcaciones obsoletas y las sustituciones accidentales sigan siendo visibles.

## Requisitos de JSON Schema

- **Todos los plugins deben incluir un JSON Schema**, aunque no acepten ninguna configuración.
- Se acepta un esquema vacío (por ejemplo, `{ "type": "object", "additionalProperties": false }`).
- Los esquemas se validan al leer o escribir la configuración, no durante la ejecución.
- Al ampliar o bifurcar un plugin incluido con nuevas claves de configuración, se debe actualizar al mismo tiempo el `openclaw.plugin.json` `configSchema` de ese plugin. Los esquemas de los plugins incluidos son estrictos, por lo que añadir `plugins.entries.<id>.config.myNewKey` a la configuración del usuario sin añadir `myNewKey` a `configSchema.properties` se rechazará antes de que se cargue el entorno de ejecución del plugin.

Ejemplo de ampliación del esquema:

```json
{
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myNewKey": {
        "type": "string"
      }
    }
  }
}
```

## Comportamiento de la validación

- Las claves `channels.*` desconocidas son **errores**, salvo que el identificador del canal esté declarado en el manifiesto de un plugin. Si el mismo identificador también aparece en `plugins.allow`, `plugins.entries` o `plugins.installs` (un plugin al que se hace referencia pero que no se puede descubrir actualmente), OpenClaw lo rebaja a una **advertencia**.
- Las referencias de `plugins.entries.<id>`, `plugins.allow` y `plugins.deny` a identificadores de plugins desconocidos son **advertencias** ("se ignora una entrada de configuración obsoleta"), no errores, para que las actualizaciones y los plugins eliminados o renombrados no bloqueen el inicio del Gateway.
- La referencia de `plugins.slots.memory` a un identificador de plugin desconocido es un **error**, excepto en el caso del plugin externo oficial conocido `memory-lancedb`, que genera una advertencia.
- Si un plugin está instalado, pero tiene un manifiesto o esquema dañado o ausente, la validación falla y Doctor informa del error del plugin.
- Si existe configuración del plugin, pero este está **desactivado**, la configuración se conserva y se muestra una **advertencia** en Doctor y en los registros.

Consulte la [referencia de configuración](/es/gateway/configuration) para ver el esquema completo de `plugins.*`.

## Notas

- El manifiesto es **obligatorio para los plugins nativos de OpenClaw**, incluidas las cargas desde el sistema de archivos local. El entorno de ejecución sigue cargando el módulo del plugin por separado; el manifiesto solo se usa para el descubrimiento y la validación.
- Los manifiestos nativos se analizan con JSON5, por lo que se aceptan comentarios, comas finales y claves sin comillas, siempre que el valor final siga siendo un objeto.
- El cargador de manifiestos solo lee los campos de manifiesto documentados. Se deben evitar las claves personalizadas de nivel superior.
- `channels`, `providers`, `cliBackends` y `skills` pueden omitirse cuando un plugin no los necesita.
- `providerCatalogEntry` debe seguir siendo ligero y no debe importar grandes cantidades de código del entorno de ejecución; se debe usar para metadatos estáticos del catálogo de proveedores o descriptores de descubrimiento específicos, no para la ejecución durante las solicitudes.
- Los tipos de plugins exclusivos se seleccionan mediante `plugins.slots.*`: `kind: "memory"` mediante `plugins.slots.memory` (valor predeterminado: `memory-core`), y `kind: "context-engine"` mediante `plugins.slots.contextEngine` (valor predeterminado: `legacy`).
- El tipo de plugin exclusivo se declara en este manifiesto. El `OpenClawPluginDefinition.kind` del punto de entrada del entorno de ejecución está obsoleto y se conserva únicamente como alternativa de compatibilidad para plugins antiguos.
- Los metadatos de variables de entorno de `setup.providers[].envVars` son únicamente declarativos. El estado, la auditoría, la validación de entregas de Cron y otras superficies de solo lectura siguen aplicando la confianza del plugin y la política de activación efectiva antes de considerar configurada una variable de entorno.
- Para consultar los metadatos del asistente del entorno de ejecución que requieren código del proveedor, consulte los [hooks del entorno de ejecución del proveedor](/es/plugins/architecture-internals#provider-runtime-hooks).
- Si el plugin depende de módulos nativos, se deben documentar los pasos de compilación y los requisitos de listas de permitidos del gestor de paquetes (por ejemplo, pnpm `allow-build-scripts` + `pnpm rebuild <package>`).

## Contenido relacionado

<CardGroup cols={3}>
  <Card title="Creación de plugins" href="/es/plugins/building-plugins" icon="rocket">
    Primeros pasos con los plugins.
  </Card>
  <Card title="Arquitectura de plugins" href="/es/plugins/architecture" icon="diagram-project">
    Arquitectura interna y modelo de capacidades.
  </Card>
  <Card title="Descripción general del SDK" href="/es/plugins/sdk-overview" icon="book">
    Referencia del SDK de plugins e importaciones de subrutas.
  </Card>
</CardGroup>
