---
read_when:
    - Está añadiendo un asistente de configuración a un plugin
    - Necesita comprender setup-entry.ts frente a index.ts
    - Está definiendo esquemas de configuración de plugins o metadatos de openclaw en package.json
sidebarTitle: Setup and config
summary: Asistentes de configuración, setup-entry.ts, esquemas de configuración y metadatos de package.json
title: Configuración y ajustes del Plugin
x-i18n:
    generated_at: "2026-07-26T05:22:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

Referencia para el empaquetado de plugins (metadatos de `package.json`), manifiestos (`openclaw.plugin.json`), entradas de configuración y esquemas de configuración.

<Tip>
**¿Se busca una guía paso a paso?** Las guías prácticas explican el empaquetado en contexto: [Plugins de canal](/plugins/sdk-channel-plugins#step-1-package-and-manifest) y [Plugins de proveedor](/es/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Metadatos del paquete

El `package.json` necesita un campo `openclaw` que indique al sistema de plugins qué proporciona el plugin:

<Tabs>
  <Tab title="Plugin de canal">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "Mi canal",
          "blurb": "Descripción breve del canal."
        }
      }
    }
    ```
  </Tab>
  <Tab title="Plugin de proveedor / base de referencia de ClawHub">
    ```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        "typebox": "1.1.39"
      },
      "peerDependencies": {
        "openclaw": ">=2026.3.24-beta.2"
      },
      "openclaw": {
        "extensions": ["./index.ts"],
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
  </Tab>
</Tabs>

<Note>
La publicación externa en ClawHub requiere `compat` y `build`. Los fragmentos canónicos de publicación se encuentran en `docs/snippets/plugin-publish/`.
</Note>

### Campos de `openclaw`

<ParamField path="extensions" type="string[]">
  Archivos de punto de entrada (relativos a la raíz del paquete). Entradas de código fuente válidas para el desarrollo en espacios de trabajo y checkouts de Git.
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  Archivos JavaScript compilados equivalentes para `extensions`, preferidos cuando OpenClaw carga un paquete npm instalado. Consulte [Puntos de entrada del SDK](/es/plugins/sdk-entrypoints) para conocer el orden de resolución entre código fuente y código compilado.
</ParamField>
<ParamField path="setupEntry" type="string">
  Entrada ligera exclusiva para la configuración (opcional).
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  Archivo JavaScript compilado equivalente para `setupEntry`. Requiere que también se establezca `setupEntry`.
</ParamField>
<ParamField path="plugin" type="object">
  Identidad de plugin alternativa de `{ id, label }`, utilizada cuando un plugin no tiene metadatos de canal o proveedor de los que derivar un identificador o una etiqueta.
</ParamField>
<ParamField path="channel" type="object">
  Metadatos del catálogo de canales para las superficies de configuración, selección, inicio rápido y estado.
</ParamField>
<ParamField path="install" type="object">
  Indicaciones de instalación: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`, `requiredPlatformPackages`.
</ParamField>
<ParamField path="startup" type="object">
  Indicadores de comportamiento durante el inicio.
</ParamField>
<ParamField path="compat" type="object">
  Intervalo de versiones de `pluginApi` compatible con este plugin. Obligatorio para publicaciones externas en ClawHub.
</ParamField>

<Note>
Los identificadores de proveedores (`providers: string[]`) son metadatos del manifiesto, no metadatos del paquete. Declárelos en `openclaw.plugin.json`, no aquí; consulte [Manifiesto del plugin](/es/plugins/manifest).
</Note>

### `openclaw.channel`

`openclaw.channel` son metadatos de paquete ligeros para el descubrimiento de canales y las superficies de configuración antes de cargar el entorno de ejecución.

### Campos de configuración propiedad del canal

Los plugins de canal deben definir los campos de configuración una sola vez en el código del entorno de ejecución mediante `defineChannelSetupContract(...)` y publicar la proyección serializable correspondiente en `openclaw.channel.setup.fields`. La definición del entorno de ejecución infiere el tipo de entrada local del plugin, analiza tanto los valores guiados como los no interactivos y mantiene las claves específicas del canal fuera de los tipos del núcleo. Los metadatos del paquete permiten que `openclaw channels add <channel-id> --help` y `openclaw channels add --channel <channel-id> --help` descubran únicamente las opciones del canal seleccionado sin cargar el plugin.

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "Punto de conexión del servicio" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "Propietario del transporte" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "Punto de conexión del servicio" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "Propietario del transporte" }
          }
        ]
      }
    }
  }
}
```

Los tipos de campo admitidos son `string`, `boolean`, `integer`, `string-list` y `choice`. Utilice `sensitive: true` para las credenciales. Cada clave de campo debe ser igual al nombre de atributo en camelCase de su opción larga de la CLI, incluida cualquier forma negada, como `apiToken` para `--api-token`. Los campos booleanos pueden añadir `cli.negatedFlags` cuando se necesiten tanto las formas positivas como las formas `--no-*`. `channel`, `account` y el `name` de visualización de la cuenta siguen formando la envoltura de control compartida.

El adaptador publicado `setup`/`ChannelSetupInput` continúa disponible para los plugins externos existentes. Los plugins nuevos deben exponer `setupContract`; OpenClaw siempre lo prefiere cuando ambos están presentes.

| Campo                                  | Tipo       | Significado                                                                 |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | Identificador canónico del canal.                                                         |
| `label`                                | `string`   | Etiqueta principal del canal.                                                        |
| `selectionLabel`                       | `string`   | Etiqueta de selección/configuración cuando deba diferir de `label`.                        |
| `detailLabel`                          | `string`   | Etiqueta secundaria de detalles para catálogos de canales y superficies de estado más completos.       |
| `docsPath`                             | `string`   | Ruta de la documentación para los enlaces de configuración y selección.                                      |
| `docsLabel`                            | `string`   | Etiqueta alternativa utilizada para los enlaces de documentación cuando deba diferir del identificador del canal. |
| `blurb`                                | `string`   | Descripción breve para la incorporación y el catálogo.                                         |
| `order`                                | `number`   | Orden de clasificación en los catálogos de canales.                                               |
| `aliases`                              | `string[]` | Alias de búsqueda adicionales para seleccionar el canal.                                   |
| `preferOver`                           | `string[]` | Identificadores de plugins/canales de menor prioridad a los que este canal debe preceder.                |
| `systemImage`                          | `string`   | Nombre opcional del icono o imagen del sistema para los catálogos de canales de la interfaz.                      |
| `selectionDocsPrefix`                  | `string`   | Texto de prefijo anterior a los enlaces de documentación en las superficies de selección.                          |
| `selectionDocsOmitLabel`               | `boolean`  | Muestra directamente la ruta de la documentación en lugar de un enlace etiquetado en el texto de selección. |
| `selectionExtras`                      | `string[]` | Cadenas breves adicionales anexadas al texto de selección.                               |
| `markdownCapable`                      | `boolean`  | Marca el canal como compatible con Markdown para las decisiones de formato de salida.      |
| `exposure`                             | `object`   | Controles de visibilidad del canal para la configuración, las listas configuradas y las superficies de documentación.   |
| `quickstartAllowFrom`                  | `boolean`  | Incluye este canal en el flujo de configuración estándar de inicio rápido `allowFrom`.         |
| `forceAccountBinding`                  | `boolean`  | Exige la vinculación explícita de la cuenta incluso cuando solo existe una cuenta.           |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Prefiere la búsqueda de sesiones al resolver los destinos de anuncios para este canal.       |
| `setup`                                | `object`   | Campos de configuración serializables propiedad del canal utilizados para el descubrimiento diferido de opciones de la CLI.   |

Ejemplo:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "Mi canal",
      "selectionLabel": "Mi canal (autoalojado)",
      "detailLabel": "Bot de mi canal",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Integración de chat autoalojada basada en Webhook.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Guía:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` admite:

- `configured`: incluye el canal en las superficies de listado de configuración/estado
- `setup`: incluye el canal en los selectores interactivos de configuración
- `docs`: marca el canal como visible públicamente en las superficies de documentación/navegación

### `openclaw.install`

`openclaw.install` son metadatos del paquete, no metadatos del manifiesto.

| Campo                        | Tipo                                | Qué significa                                                                     |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | Especificación canónica de ClawHub para los flujos de instalación/actualización y de instalación bajo demanda durante la incorporación. |
| `npmSpec`                    | `string`                            | Especificación canónica de npm para los flujos alternativos de instalación/actualización.                             |
| `localPath`                  | `string`                            | Ruta de desarrollo local o de instalación incluida.                                        |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | Fuente de instalación preferida cuando hay varias fuentes disponibles.                     |
| `minHostVersion`             | `string`                            | Versión mínima compatible de OpenClaw, `>=x.y.z` o `>=x.y.z-prerelease`.            |
| `expectedIntegrity`          | `string`                            | Cadena de integridad esperada de la distribución de npm, normalmente `sha512-...`, para instalaciones con versión fijada.    |
| `allowInvalidConfigRecovery` | `boolean`                           | Permite que los flujos de reinstalación de plugins incluidos se recuperen de errores específicos de configuración obsoleta.  |
| `requiredPlatformPackages`   | `string[]`                          | Alias de npm obligatorios específicos de cada plataforma que se verifican durante la instalación mediante npm.               |

<AccordionGroup>
  <Accordion title="Comportamiento de la incorporación">
    La incorporación interactiva utiliza `openclaw.install` para las superficies de instalación bajo demanda: si el plugin expone opciones de autenticación del proveedor o metadatos de configuración/catálogo del canal antes de que se cargue el entorno de ejecución, la incorporación puede solicitar la instalación desde ClawHub, npm o una fuente local, instalar o habilitar el plugin y, a continuación, continuar con el flujo seleccionado. Las opciones de ClawHub utilizan `clawhubSpec` y se prefieren cuando están presentes; las opciones de npm requieren metadatos de catálogo de confianza con un `npmSpec` de registro (las versiones exactas y `expectedIntegrity` son fijaciones opcionales que se aplican durante la instalación/actualización cuando se establecen). Mantenga «qué mostrar» en `openclaw.plugin.json` y «cómo instalarlo» en `package.json`.
  </Accordion>
  <Accordion title="Aplicación de minHostVersion">
    Si se establece `minHostVersion`, se aplica tanto durante la instalación como al cargar registros de manifiestos no incluidos. Los hosts antiguos omiten los plugins externos; se rechazan las cadenas de versión no válidas. Se presupone que los plugins de fuente incluidos tienen la misma versión que el checkout del host.
  </Accordion>
  <Accordion title="Instalaciones de npm con versión fijada">
    Para las instalaciones de npm con versión fijada, mantenga la versión exacta en `npmSpec` y añada la integridad esperada del artefacto:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="Ámbito de allowInvalidConfigRecovery">
    `allowInvalidConfigRecovery` no es una omisión general para configuraciones dañadas. Solo permite una recuperación limitada de plugins incluidos, de modo que la reinstalación/configuración pueda reparar restos conocidos de actualizaciones, como la ausencia de la ruta de un plugin incluido o una entrada `channels.<id>` obsoleta para ese mismo plugin. Si la configuración está dañada por motivos no relacionados, la instalación sigue fallando de forma cerrada e indica al operador que ejecute `openclaw doctor --fix`.
  </Accordion>
</AccordionGroup>

### Carga completa diferida

Los plugins de canal pueden optar por la carga diferida mediante:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Cuando está habilitada, OpenClaw carga únicamente `setupEntry` durante la fase de inicio previa a la escucha, incluso para los canales ya configurados. La entrada completa se carga después de que el Gateway comienza a escuchar.

<Warning>
Habilite la carga diferida únicamente cuando `setupEntry` registre todo lo que necesita el Gateway antes de comenzar a escuchar (registro del canal, rutas HTTP y métodos del Gateway). Si la entrada completa contiene capacidades de inicio obligatorias, mantenga el comportamiento predeterminado.
</Warning>

Si la entrada de configuración/completa registra métodos RPC del Gateway, manténgalos bajo un prefijo específico del plugin. Los espacios de nombres administrativos reservados del núcleo (`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`) siguen siendo propiedad del núcleo y siempre se normalizan a `operator.admin`.

## Manifiesto del plugin

Cada plugin nativo debe incluir un `openclaw.plugin.json` en la raíz del paquete. OpenClaw lo utiliza para validar la configuración sin ejecutar el código del plugin.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

Para los plugins de canal, añada `channels` (y, para los plugins de proveedor, añada `providers`):

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Incluso los plugins sin configuración deben incluir un esquema. Un esquema vacío es válido:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Consulte [Manifiesto del plugin](/es/plugins/manifest) para ver la referencia completa del esquema.

## Publicación en ClawHub

Las Skills y los paquetes de plugins utilizan comandos de publicación de ClawHub distintos. Para los paquetes de plugins, utilice el comando específico para paquetes:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` es un comando distinto para publicar una carpeta de Skills, no un paquete de plugin. Consulte [Publicación en ClawHub](/es/clawhub/publishing).
</Note>

## Entrada de configuración

`setup-entry.ts` es una alternativa ligera a `index.ts` que OpenClaw carga cuando solo necesita superficies de configuración (incorporación, reparación de la configuración e inspección de canales deshabilitados):

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Esto evita cargar código pesado del entorno de ejecución (bibliotecas criptográficas, registros de CLI y servicios en segundo plano) durante los flujos de configuración.

Los canales incluidos en el espacio de trabajo que mantienen exportaciones seguras para la configuración en módulos auxiliares pueden utilizar `defineBundledChannelSetupEntry(...)` de `openclaw/plugin-sdk/channel-entry-contract` en lugar de `defineSetupPluginEntry(...)`. Ese contrato incluido también admite una exportación opcional `runtime` para que el cableado del entorno de ejecución durante la configuración pueda seguir siendo ligero y explícito.

<AccordionGroup>
  <Accordion title="Cuándo utiliza OpenClaw setupEntry en lugar de la entrada completa">
    - El canal está deshabilitado, pero necesita superficies de configuración/incorporación.
    - El canal está habilitado, pero no está configurado.
    - La carga diferida está habilitada (`deferConfiguredChannelFullLoadUntilAfterListen`).

  </Accordion>
  <Accordion title="Qué debe registrar setupEntry">
    - El objeto del plugin de canal (mediante `defineSetupPluginEntry`).
    - Cualquier ruta HTTP necesaria antes de que el Gateway comience a escuchar.
    - Cualquier método del Gateway necesario durante el inicio.

    Esos métodos de inicio del Gateway deben seguir evitando los espacios de nombres administrativos reservados del núcleo, como `config.*` o `update.*`.

  </Accordion>
  <Accordion title="Qué NO debe incluir setupEntry">
    - Registros de CLI.
    - Servicios en segundo plano.
    - Importaciones pesadas del entorno de ejecución (criptografía, SDK).
    - Métodos del Gateway que solo son necesarios después del inicio.

  </Accordion>
</AccordionGroup>

### Importaciones limitadas de asistentes de configuración

Para las rutas críticas exclusivas de configuración, prefiera las interfaces limitadas de asistentes de configuración en lugar del módulo general `plugin-sdk/setup` cuando solo necesite una parte de la superficie de configuración:

| Ruta de importación                | Utilícela para                                                                                | Exportaciones clave                                                                                                                                                                                                                                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | asistentes del entorno de ejecución durante la configuración que siguen disponibles en `setupEntry` / el inicio diferido del canal | `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | asistentes de configuración/instalación para CLI, archivos y documentación                                                    | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                                         |

Utilice la interfaz más amplia `plugin-sdk/setup` cuando necesite el conjunto completo de herramientas compartidas de configuración, incluidos asistentes para aplicar parches a la configuración, como `moveSingleAccountChannelSectionToDefaultAccount(...)`.

Utilice `createSetupTranslator(...)` para el texto fijo del asistente de configuración. Utiliza el primer valor no vacío de `OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` y `LANG`, en ese orden, y después recurre al inglés. Establezca `OPENCLAW_LOCALE=en` para indicar una sustitución explícita en inglés. Mantenga el texto de configuración específico del plugin en código propiedad del plugin y utilice las claves del catálogo compartido únicamente para etiquetas comunes de configuración, texto de estado y texto de configuración de plugins oficiales incluidos.

Los adaptadores de parches de configuración siguen siendo seguros al importarse en rutas críticas. La consulta de la superficie del contrato de promoción de cuentas únicas incluidas es diferida, por lo que importar `plugin-sdk/setup-runtime` no carga anticipadamente el descubrimiento de superficies de contratos incluidos antes de que se utilice realmente el adaptador.

### Campos de entrada de configuración propiedad del canal

`ChannelSetupInput` es un contenedor genérico compartido por los invocadores de configuración y los
plugins de canal. Sus campos con tipado permanente son `name`, `token`, `tokenFile`,
`useEnv`, `allowFrom` y `defaultTo`. Aun pueden existir claves adicionales propiedad del plugin
en el objeto de entrada del entorno de ejecución, pero el tipo compartido no declara una
firma de índice. Cada plugin debe declarar y delimitar sus propios campos de configuración o
validarlos mediante un esquema propiedad del plugin en el límite del adaptador:

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

Los campos específicos del canal que anteriormente se declaraban directamente en
`ChannelSetupInput` permanecen tipados temporalmente para mantener la compatibilidad con fuentes externas.
Están obsoletos. Una revisión del registro del 2026-07-22 de 426 plugins de canal publicados fuera del árbol
eliminó 21 campos sin lectores y conservó 22 con lectores conocidos.
Cada campo conservado se elimina en cuanto ningún plugin publicado lo lee;
no se requiere ningún límite de versión. Los plugins nuevos e incluidos no deben depender de este
nivel; deben declarar localmente los campos que poseen.

### Promoción de cuenta única propiedad del canal

Cuando un canal pasa de una configuración de nivel superior de una sola cuenta a `channels.<id>.accounts.*`, el comportamiento compartido predeterminado mueve los valores promovidos con ámbito de cuenta a `accounts.default`.

Cada plugin de canal puede ampliar o restringir esa promoción mediante su adaptador de configuración:

- `singleAccountKeysToMove`: claves adicionales de nivel superior que deben trasladarse a la cuenta promovida
- `namedAccountPromotionKeys`: cuando ya existen cuentas con nombre, solo estas claves se trasladan a la cuenta promovida; las claves compartidas de políticas y entrega permanecen en la raíz del canal
- `resolveSingleAccountPromotionTarget(...)`: permite elegir qué cuenta existente recibe los valores promovidos

La presencia de `singleAccountKeysToMove` indica que el contrato de promoción está completo. Declare el campo aunque sea una matriz vacía para excluirse de la promoción de claves heredadas. Los adaptadores que omiten el campo conservan un nivel de promoción anterior a la declaración, respaldado por lectores, para los plugins ya publicados. La revisión del registro del 2026-07-22 eliminó 23 claves sin dependientes publicados y conservó seis claves comunes, además de la clave exclusiva de configuración `rooms`. Cada clave conservada se elimina en cuanto sus lectores publicados migran a las declaraciones; no se requiere ningún límite de versión.

Declare `openclaw.setupFeatures.configPromotion: true` en el manifiesto del paquete del plugin cuando doctor deba cargar estas declaraciones desde el artefacto ligero de configuración incluido. La superficie del plugin exclusiva de configuración y el plugin de canal completo deben exponer las mismas declaraciones.

Al llamar a `moveSingleAccountChannelSectionToDefaultAccount(...)` con un plugin ya resuelto, pase su adaptador de configuración como `setupSurface`. Las superficies de configuración proporcionadas por el llamador tienen prioridad sobre la búsqueda cargada e incluida, lo que mantiene los plugins con ámbito o exclusivos de configuración independientes del registro global.

<Note>
Matrix es el ejemplo incluido actual. Si ya existe exactamente una cuenta de Matrix con nombre, o si `defaultAccount` apunta a una clave no canónica existente, como `Ops`, la promoción conserva esa cuenta en lugar de crear una nueva entrada `accounts.default`.
</Note>

## Esquema de configuración

La configuración del plugin se valida con el esquema JSON del manifiesto. Los usuarios configuran los plugins mediante:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

El plugin recibe esta configuración como `api.pluginConfig` durante el registro.

Para la configuración específica del canal, utilice en su lugar la sección de configuración del canal:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Creación de esquemas de configuración de canales

Utilice `buildChannelConfigSchema` para convertir un esquema de Zod en el contenedor `ChannelConfigSchema` utilizado por los artefactos de configuración propiedad del plugin:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

Si ya crea el contrato como esquema JSON o TypeBox, utilice el asistente directo para que OpenClaw pueda omitir la conversión de Zod a esquema JSON en las rutas de metadatos:

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

Para los plugins de terceros, el contrato de la ruta en frío sigue siendo el manifiesto del plugin: replique el esquema JSON generado en `openclaw.plugin.json#channelConfigs` para que las superficies del esquema de configuración, de configuración y de la interfaz de usuario puedan inspeccionar `channels.<id>` sin cargar código de ejecución.

## Asistentes de configuración

Los plugins de canal pueden proporcionar asistentes de configuración interactivos para `openclaw onboard`. El asistente es un objeto `ChannelSetupWizard` en `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` también admite `textInputs`, `dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` y más. Consulte `src/setup-core.ts` del plugin de Discord para ver un ejemplo incluido completo.

<AccordionGroup>
  <Accordion title="Indicaciones allowFrom compartidas">
    Para las indicaciones de la lista de permitidos de mensajes directos que solo necesitan el flujo estándar `note -> prompt -> parse -> merge -> patch`, se recomienda utilizar los asistentes de configuración compartidos de `openclaw/plugin-sdk/setup`: `createPromptParsedAllowFromForAccount(...)` y `createTopLevelChannelParsedAllowFromPrompt(...)`.
  </Accordion>
  <Accordion title="Estado estándar de configuración del canal">
    Para los bloques de estado de configuración del canal que solo varían en las etiquetas, las puntuaciones y las líneas adicionales opcionales, se recomienda utilizar `createStandardChannelSetupStatus(...)` de `openclaw/plugin-sdk/setup` en lugar de crear manualmente el mismo objeto `status` en cada plugin.
  </Accordion>
  <Accordion title="Superficie opcional de configuración del canal">
    Para las superficies de configuración opcionales que solo deben aparecer en determinados contextos, utilice `createOptionalChannelSetupSurface` de `openclaw/plugin-sdk/channel-setup`:

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    `plugin-sdk/channel-setup` también expone los constructores de nivel inferior `createOptionalChannelSetupAdapter(...)` y `createOptionalChannelSetupWizard(...)` cuando solo se necesita una mitad de esa superficie de instalación opcional.

    El adaptador y el asistente opcionales generados se cierran de forma segura ante escrituras reales de configuración. Reutilizan un único mensaje de instalación requerida en `validateInput`, `applyAccountConfig` y `finalize`, y añaden un enlace a la documentación cuando se establece `docsPath`.

  </Accordion>
  <Accordion title="Asistentes de configuración respaldados por binarios">
    Para las interfaces de configuración respaldadas por binarios, se recomienda utilizar los asistentes delegados compartidos en lugar de copiar el mismo código de conexión de binarios y estados en cada canal:

    - `createDetectedBinaryStatus(...)` para bloques de estado que solo varían en las etiquetas, las sugerencias, las puntuaciones y la detección de binarios
    - `createCliPathTextInput(...)` para entradas de texto respaldadas por rutas
    - `createDelegatedSetupWizardProxy(...)` cuando `setupEntry` necesita reenviar de forma diferida el comportamiento de estado, preparación o finalización a un asistente completo más pesado
    - `createDelegatedTextInputShouldPrompt(...)` cuando `setupEntry` solo necesita delegar una decisión `textInputs[*].shouldPrompt`

  </Accordion>
</AccordionGroup>

## Publicación e instalación

**Plugins externos:** publíquelos en [ClawHub](/es/clawhub) y, a continuación, instálelos:

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    Las especificaciones de paquete sin prefijo se instalan desde npm durante la transición del lanzamiento, salvo que el nombre coincida con el identificador de un plugin incluido u oficial, en cuyo caso OpenClaw utiliza en su lugar esa copia local u oficial. Utilice `clawhub:`, `npm:`, `git:` o `npm-pack:` para seleccionar la fuente de forma determinista; consulte [Administrar plugins](/es/plugins/manage-plugins).

  </Tab>
  <Tab title="Solo ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="Especificación de paquete npm">
    Utilice npm cuando un paquete aún no se haya trasladado a ClawHub o cuando se necesite una
    ruta de instalación directa desde npm durante la migración:

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**Plugins del repositorio:** colóquelos bajo el árbol del espacio de trabajo de plugins incluidos; se detectan automáticamente durante la compilación.

<Info>
Para las instalaciones procedentes de npm, `openclaw plugins install` instala el paquete en un proyecto por plugin bajo `~/.openclaw/npm/projects` con los scripts del ciclo de vida desactivados (`--ignore-scripts`). Mantenga los árboles de dependencias de los plugins exclusivamente en JS/TS y evite los paquetes que requieran compilaciones `postinstall`.
</Info>

<Note>
El inicio del Gateway no instala las dependencias de los plugins. Los flujos de instalación de npm/git/ClawHub son responsables de la convergencia de dependencias; los plugins locales deben tener ya instaladas sus dependencias.
</Note>

Los metadatos de los paquetes incluidos son explícitos; no se deducen del JavaScript compilado durante el inicio del Gateway. Las dependencias de ejecución pertenecen al paquete del plugin que las posee; el inicio de OpenClaw empaquetado nunca repara ni replica las dependencias de los plugins.

## Contenido relacionado

- [Creación de plugins](/es/plugins/building-plugins) — guía de introducción paso a paso
- [Manifiesto del plugin](/es/plugins/manifest) — referencia completa del esquema del manifiesto
- [Puntos de entrada del SDK](/es/plugins/sdk-entrypoints) — `definePluginEntry` y `defineChannelPluginEntry`
