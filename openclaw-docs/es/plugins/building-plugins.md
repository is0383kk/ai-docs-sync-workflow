---
doc-schema-version: 1
read_when:
    - Quiere crear un nuevo plugin de OpenClaw
    - Necesita una guía de inicio rápido para el desarrollo de plugins
    - Está eligiendo entre la documentación de canales, proveedores, backends de CLI, herramientas o hooks
sidebarTitle: Getting Started
summary: Crea tu primer plugin de OpenClaw en minutos
title: Creación de plugins
x-i18n:
    generated_at: "2026-07-26T04:48:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

Los plugins amplían OpenClaw sin modificar el núcleo. Un plugin puede añadir un
canal de mensajería, un proveedor de modelos, un backend de CLI local, una herramienta de agente, un hook, un proveedor de medios
u otra capacidad propiedad del plugin.

No es necesario añadir un plugin externo al repositorio de OpenClaw. Publique
el paquete en [ClawHub](/es/clawhub) y los usuarios podrán instalarlo con:

```bash
openclaw plugins install clawhub:<package-name>
```

Las especificaciones de paquetes sin prefijo todavía se instalan desde npm durante la transición del lanzamiento. Utilice el
prefijo `clawhub:` cuando quiera usar la resolución de ClawHub.

## Requisitos

- Node 22.22.3+, Node 24.15+ o Node 25.9+, y `npm` o `pnpm`.
- Módulos ESM de TypeScript.
- Para trabajar en plugins incluidos en el repositorio, clone el repositorio y ejecute `pnpm install`.
  El desarrollo de plugins desde una copia del código fuente solo admite pnpm porque OpenClaw descubre
  los plugins incluidos a partir de los paquetes del espacio de trabajo `extensions/*`.

## Elegir la estructura del plugin

<CardGroup cols={2}>
  <Card title="Plugin de canal" icon="messages-square" href="/es/plugins/sdk-channel-plugins">
    Conecte OpenClaw a una plataforma de mensajería.
  </Card>
  <Card title="Plugin de proveedor" icon="cpu" href="/es/plugins/sdk-provider-plugins">
    Añada un proveedor de modelos, medios, búsqueda, obtención, voz o tiempo real.
  </Card>
  <Card title="Plugin de backend de CLI" icon="terminal" href="/es/plugins/cli-backend-plugins">
    Ejecute una CLI de IA local mediante la alternativa de modelos de OpenClaw.
  </Card>
  <Card title="Plugin de herramientas" icon="wrench" href="/es/plugins/tool-plugins">
    Registre herramientas de agente.
  </Card>
</CardGroup>

## Inicio rápido

Cree un plugin de herramientas mínimo registrando una herramienta de agente obligatoria. Esta es la
estructura de plugin útil más breve y abarca el paquete, el manifiesto, el punto de entrada y
la verificación local.

<Steps>
  <Step title="Crear los metadatos del paquete">
    <CodeGroup>

```json package.json
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

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    Los plugins externos publicados deben hacer que las entradas de ejecución apunten a archivos JavaScript
    compilados. Consulte [Puntos de entrada del SDK](/es/plugins/sdk-entrypoints) para conocer el contrato completo
    de los puntos de entrada.

    Cada plugin necesita un manifiesto, incluso si no tiene configuración. Las herramientas de ejecución deben
    aparecer en `contracts.tools` para que OpenClaw pueda descubrir su propiedad sin
    cargar de forma anticipada el entorno de ejecución de cada plugin. Defina `activation.onStartup`
    deliberadamente; este ejemplo se carga al iniciar el Gateway.

    Las superficies de plugins de confianza para el host también están controladas por el manifiesto y requieren una
    declaración explícita para los plugins instalados: `api.registerAgentToolResultMiddleware(...)`
    requiere que cada entorno de ejecución de destino figure en `contracts.agentToolResultMiddleware`,
    y `api.registerTrustedToolPolicy(...)` requiere cada identificador de política en
    `contracts.trustedToolPolicies`. Estas declaraciones mantienen alineadas la
    inspección durante la instalación y el registro en tiempo de ejecución.

    Para conocer todos los campos del manifiesto, consulte [Manifiesto del plugin](/es/plugins/manifest).

  </Step>

  <Step title="Registrar la herramienta">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Echo one input value",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Got: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    Utilice `definePluginEntry` para los plugins que no sean de canal. Los plugins de canal utilizan
    en su lugar `defineChannelPluginEntry` de `openclaw/plugin-sdk/core`.

  </Step>

  <Step title="Probar el entorno de ejecución">
    Para un plugin instalado o externo, inspeccione el entorno de ejecución cargado:

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    Si el plugin registra un comando de CLI, ejecute también ese comando y confirme
    la salida; por ejemplo, `openclaw demo-plugin ping`.

    Para un plugin incluido en este repositorio, OpenClaw descubre los paquetes de plugins
    de la copia del código fuente a partir del espacio de trabajo `extensions/*`. Ejecute la prueba específica
    más cercana:

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="Probar la instalación del paquete">
    Antes de publicar un plugin listo para empaquetar, pruebe la misma forma de instalación que
    recibirán los usuarios. Primero añada un paso de compilación, haga que las entradas de ejecución como
    `openclaw.extensions` apunten a JavaScript compilado como `./dist/index.js` y asegúrese de
    que `npm pack` incluya esa salida `dist/`. Las entradas de código fuente TypeScript son
    solo para copias del código fuente y rutas de desarrollo local.

    A continuación, empaquete el plugin e instale el archivo tar con `npm-pack:`:

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:` utiliza el proyecto npm administrado por OpenClaw para cada plugin, por lo que detecta
    errores en las dependencias de ejecución que las pruebas desde una copia del código fuente pueden ocultar. Demuestra
    la estructura del paquete y de sus dependencias, no la confianza oficial vinculada al catálogo.
    Las importaciones del entorno de ejecución deben estar en `dependencies` o `optionalDependencies`;
    las dependencias que solo figuren en `devDependencies` no se instalarán para el
    proyecto de entorno de ejecución administrado.

    No utilice una instalación directa desde un archivo o una ruta como verificación final del comportamiento
    oficial o privilegiado de un plugin. El código fuente directo resulta útil para la depuración local, pero
    no demuestra la misma ruta de dependencias que las instalaciones desde npm o ClawHub. Si
    su plugin depende del estado de plugin oficial de confianza, añada una segunda verificación
    mediante una instalación oficial respaldada por el catálogo o una ruta de paquete publicado que
    registre la confianza oficial. Consulte
    [Resolución de dependencias de plugins](/es/plugins/dependency-resolution) para obtener
    detalles sobre la raíz de instalación y la propiedad de las dependencias.

  </Step>

  <Step title="Publicar">
    Valide el paquete antes de publicarlo:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    Los fragmentos canónicos de paquetes de ClawHub se encuentran en `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="Instalar">
    Instale el paquete publicado mediante ClawHub:

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## Registrar herramientas

Las herramientas pueden ser obligatorias u opcionales. Las herramientas obligatorias están siempre disponibles cuando el
plugin está habilitado. Las herramientas opcionales requieren la aceptación explícita del usuario antes de que OpenClaw
cargue el entorno de ejecución del plugin propietario.

Las fábricas de herramientas reciben un contexto de ejecución de confianza, incluidos `deliveryContext`,
`nativeChannelId` para la conversación activa de la plataforma cuando está disponible y
`requesterSenderId`.

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` es opcional. Describe el valor estructurado `details` utilizado por
[Modo de código](/es/tools/code-mode) y [Búsqueda de herramientas](/es/tools/tool-search). Las llamadas al catálogo
rechazan los esquemas no válidos antes de la ejecución y validan el valor final después de
los hooks de herramientas. Omítalo para las herramientas que no tengan un resultado JSON estable. Consulte
[Plugins de herramientas](/es/plugins/tool-plugins#output-contracts) para conocer el contrato completo.

Cada herramienta registrada con `api.registerTool(...)` también debe declararse en el
manifiesto del plugin:

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

Los usuarios aceptan su uso mediante `tools.allow`:

```json5
{
  tools: { allow: ["workflow_tool"] }, // or ["my-plugin"] for every tool from one plugin
}
```

Las herramientas opcionales controlan si una herramienta se expone al modelo. Utilice
[solicitudes de permisos de plugins](/es/plugins/plugin-permission-requests) cuando una herramienta
o un hook deba solicitar aprobación después de que el modelo lo seleccione y antes de que se
ejecute la acción.

Utilice herramientas opcionales para efectos secundarios, binarios poco habituales o capacidades que
no deban exponerse de forma predeterminada. Los nombres de las herramientas no deben entrar en conflicto con los nombres de las herramientas
del núcleo; los conflictos se omiten y se notifican en los diagnósticos de plugins. Los
registros con formato incorrecto se omiten y se notifican de la misma manera: un
`name` no vacío ausente, un `execute` que no sea una función o un descriptor de herramienta sin un objeto `parameters`.

Las fábricas de herramientas reciben un objeto de contexto proporcionado por el entorno de ejecución. Utilice `ctx.activeModel`
cuando una herramienta necesite registrar, mostrar o adaptarse al modelo activo del turno
actual; puede incluir `provider`, `modelId` y `modelRef`. Trátelo como
metadatos informativos del entorno de ejecución, no como un límite de seguridad frente al operador
local, el código de plugins instalado o un entorno de ejecución de OpenClaw modificado. Las
herramientas locales sensibles deben seguir requiriendo la aceptación explícita del plugin o del operador y
rechazar la ejecución cuando los metadatos del modelo activo falten o no sean adecuados.

El manifiesto declara la propiedad y el descubrimiento; la ejecución sigue invocando la
implementación de la herramienta registrada en vivo. Mantenga `toolMetadata.<tool>.optional: true`
alineado con `api.registerTool(..., { optional: true })` para que OpenClaw pueda evitar
cargar el entorno de ejecución de ese plugin hasta que la herramienta se incluya explícitamente en la lista de permitidas.

## Convenciones de importación

Importe desde subrutas específicas del SDK:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

Dentro del paquete del plugin, utilice archivos de barril locales como `api.ts` y
`runtime-api.ts` para las importaciones internas. No importe su propio plugin mediante una
ruta del SDK. Los auxiliares específicos del proveedor deben permanecer en el paquete del proveedor, salvo que
el punto de integración sea verdaderamente genérico.

Los métodos RPC personalizados del Gateway son un punto de entrada avanzado. Manténgalos en un
prefijo específico del plugin; los espacios de nombres administrativos del núcleo como `config.*`,
`exec.approvals.*`, `operator.admin.*`, `wizard.*` y `update.*` permanecen reservados
y se resuelven como `operator.admin`. El
puente `openclaw/plugin-sdk/gateway-method-runtime` está reservado para las rutas HTTP de plugins
que declaran `contracts.gatewayMethodDispatch: ["authenticated-request"]`.

Para consultar el mapa completo de importaciones, consulte [Descripción general del SDK de plugins](/es/plugins/sdk-overview).

Los campos de compatibilidad del SDK de OpenClaw contienen anotaciones `@deprecated` de TypeScript,
que los editores muestran como advertencias de migración. Para aplicarlas durante la compilación,
habilite una regla que tenga en cuenta los tipos, como
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/).
Oxlint no tiene en cuenta los tipos, por lo que no puede aplicar estas anotaciones.

## Lista de comprobación previa al envío

<Check>**package.json** tiene los metadatos `openclaw` correctos</Check>
<Check>El manifiesto **openclaw.plugin.json** está presente y es válido</Check>
<Check>El punto de entrada usa `defineChannelPluginEntry` o `definePluginEntry`</Check>
<Check>Todas las importaciones usan rutas `plugin-sdk/<subpath>` específicas</Check>
<Check>Las importaciones internas usan módulos locales, no autoimportaciones del SDK</Check>
<Check>Las pruebas pasan (`pnpm test <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` pasa (plugins del repositorio)</Check>

## Pruebas con versiones beta

1. Siga los lanzamientos de [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) (`Watch` > `Releases`). Las etiquetas beta tienen un formato similar a `v2026.3.N-beta.1`. También puede seguir a [@openclaw](https://x.com/openclaw) en X para recibir anuncios de lanzamientos.
2. Pruebe su plugin con la etiqueta beta en cuanto aparezca. El plazo antes de la versión estable suele ser de solo unas horas.
3. Después de realizar las pruebas, publique en el hilo de su plugin en el canal de Discord `plugin-forum` ([discord.gg/clawd](https://discord.gg/clawd)) e indique `all good` o qué dejó de funcionar. Cree un hilo si todavía no tiene uno.
4. Si algo deja de funcionar, abra o actualice una incidencia titulada `Beta blocker: <plugin-name> - <summary>` y aplique la etiqueta `beta-blocker`. Enlace la incidencia en su hilo.
5. Abra un PR para `main` titulado `fix(<plugin-id>): beta blocker - <summary>` y enlace la incidencia tanto en el PR como en su hilo de Discord. Los colaboradores no pueden etiquetar los PR, por lo que el título sirve como señal del PR para los mantenedores y la automatización. Los bloqueos con un PR se fusionan; los bloqueos sin uno podrían publicarse de todos modos.
6. El silencio significa que todo está correcto. Si se pierde el plazo, la corrección suele incorporarse en el siguiente ciclo.

## Siguientes pasos

<CardGroup cols={2}>
  <Card title="Plugins de canales" icon="messages-square" href="/es/plugins/sdk-channel-plugins">
    Cree un plugin de canal de mensajería
  </Card>
  <Card title="Plugins de proveedores" icon="cpu" href="/es/plugins/sdk-provider-plugins">
    Cree un plugin de proveedor de modelos
  </Card>
  <Card title="Plugins de backend de CLI" icon="terminal" href="/es/plugins/cli-backend-plugins">
    Registre un backend local de CLI de IA
  </Card>
  <Card title="Descripción general del SDK" icon="book-open" href="/es/plugins/sdk-overview">
    Referencia del mapa de importaciones y de la API de registro
  </Card>
  <Card title="Ayudantes de entorno de ejecución" icon="settings" href="/es/plugins/sdk-runtime">
    TTS, búsqueda y subagente mediante api.runtime
  </Card>
  <Card title="Pruebas" icon="test-tubes" href="/es/plugins/sdk-testing">
    Utilidades y patrones de prueba
  </Card>
  <Card title="Manifiesto del plugin" icon="file-json" href="/es/plugins/manifest">
    Referencia completa del esquema del manifiesto
  </Card>
</CardGroup>

## Contenido relacionado

- [Hooks de plugins](/es/plugins/hooks)
- [Arquitectura de plugins](/es/plugins/architecture)
