---
read_when:
    - Estás creando un Plugin de backend de CLI de IA local
    - Quieres registrar un backend para referencias de modelos como acme-cli/model
    - Debes integrar una CLI de terceros en el ejecutor alternativo de texto de OpenClaw
sidebarTitle: CLI backend plugins
summary: Crea un plugin que registre un backend local de CLI de IA
title: Creación de plugins de backend para la CLI
x-i18n:
    generated_at: "2026-07-26T05:19:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

Los plugins de backend de CLI permiten que OpenClaw invoque una CLI de IA local como backend de inferencia
de texto. El backend aparece como prefijo del proveedor en las referencias de modelo:

```text
acme-cli/acme-large
```

Use un backend de CLI cuando la integración ascendente ya esté expuesta como un comando
local, cuando la CLI gestione el estado de inicio de sesión local o como alternativa cuando los
proveedores de API no estén disponibles.

<Info>
  Si el servicio ascendente expone una API HTTP de modelos normal, cree en su lugar un
  [plugin de proveedor](/es/plugins/sdk-provider-plugins). Si el entorno de ejecución ascendente
  gestiona sesiones completas del agente, eventos de herramientas, Compaction o el estado de
  tareas en segundo plano, use un [arnés de agente](/es/plugins/sdk-agent-harness).
</Info>

## Responsabilidades del plugin

Un plugin de backend de CLI tiene tres contratos:

| Contrato             | Archivo                   | Propósito                                                   |
| -------------------- | ------------------------- | ----------------------------------------------------------- |
| Entrada del paquete  | `package.json`        | Dirige OpenClaw al módulo de entorno de ejecución del plugin |
| Propiedad del manifiesto | `openclaw.plugin.json` | Declara el id. del backend antes de cargar el entorno de ejecución |
| Registro en tiempo de ejecución | `index.ts` | Invoca `api.registerCliBackend(...)` con los valores predeterminados del comando |

El manifiesto contiene metadatos de detección: no ejecuta la CLI ni registra
el comportamiento en tiempo de ejecución. El comportamiento en tiempo de ejecución comienza cuando la entrada del plugin invoca
`api.registerCliBackend(...)`.

## Plugin de backend mínimo

<Steps>
  <Step title="Crear los metadatos del paquete">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
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
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    Los paquetes publicados deben incluir archivos JavaScript compilados para el entorno de ejecución. Si la entrada
    del código fuente es `./src/index.ts`, añada `openclaw.runtimeExtensions` que apunte al archivo JavaScript
    compilado correspondiente. Consulte [Puntos de entrada](/es/plugins/sdk-entrypoints).

  </Step>

  <Step title="Declarar la propiedad del backend">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Run Acme's local AI CLI through OpenClaw",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` es la lista de propiedad en tiempo de ejecución; permite que OpenClaw cargue automáticamente el
    plugin cuando la selección del modelo o `agentRuntime.id` mencione `acme-cli`.

    `setup.cliBackends` es la superficie de configuración basada primero en descriptores. Añádala cuando
    la detección de modelos, la incorporación o el estado deban reconocer el backend
    sin cargar el entorno de ejecución del plugin. Use `requiresRuntime: false` únicamente cuando
    esos descriptores estáticos sean suficientes para la configuración.

  </Step>

  <Step title="Registrar el backend">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    El id. del backend debe coincidir con la entrada `cliBackends` del manifiesto. El adaptador
    registrado es código autoritativo del plugin; la configuración de OpenClaw selecciona el backend,
    pero no reescribe su contrato de comandos.

  </Step>
</Steps>

## Estructura de configuración

`CliBackendConfig` describe cómo debe OpenClaw iniciar y analizar la CLI. El
ejemplo práctico anterior utiliza intencionadamente los mismos campos de comando, reanudación, JSONL,
alias de modelo, sesión, imagen y watchdog que el adaptador incluido
`google-gemini-cli`:

| Campo                                                     | Uso                                                                               |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                        | Nombre del binario o ruta absoluta del comando                                    |
| `args`                                        | argv base para ejecuciones nuevas                                                 |
| `resumeArgs`                                        | argv alternativo para sesiones reanudadas; admite `{sessionId}`              |
| `output` / `resumeOutput`                   | Analizador: `json`, `jsonl` o `text`           |
| `jsonlDialect`                                        | Dialecto de eventos JSONL: `claude-stream-json` o `gemini-stream-json`                |
| `liveSession`                                        | Modo de proceso de CLI de larga duración (`claude-stdio`)                     |
| `input`                                        | Transporte del prompt: `arg` o `stdin`                    |
| `maxPromptArgChars`                                        | Longitud máxima del prompt para el modo `arg` antes de recurrir a stdin |
| `env` / `clearEnv`                   | Variables de entorno adicionales que se inyectan o nombres que se eliminan antes del inicio |
| `modelArg`                                        | Indicador utilizado antes del id. del modelo                                      |
| `modelAliases`                                        | Asigna los id. de modelo de OpenClaw a los id. nativos de la CLI                  |
| `sessionArgs`                                        | Cómo pasar un id. de sesión mediante `{sessionId}`                            |
| `sessionMode`                                        | `always`, `existing` o `none`                       |
| `sessionIdFields`                                        | Campos JSON que OpenClaw lee de la salida de la CLI                               |
| `systemPromptArg` / `systemPromptFileArg`                   | Transporte del prompt del sistema                                                 |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey`                   | Transporte de anulación de configuración para un archivo de prompt del sistema (por ejemplo, `-c`) |
| `systemPromptMode`                                        | `append` o `replace`                                           |
| `systemPromptWhen`                                        | `first`, `always` o `never`                       |
| `imageArg` / `imageMode`                   | Indicador de ruta de imagen y cómo pasar varias imágenes (`repeat` o `list`) |
| `imagePathScope`                                        | Ubicación de los archivos de imagen preparados antes de la entrega: `temp` o `workspace` |
| `serialize`                                        | Mantiene ordenadas las ejecuciones del mismo backend                              |
| `reseedFromRawTranscriptWhenUncompacted`                                        | Habilita la reinicialización limitada de la transcripción sin procesar antes de Compaction para restablecer sesiones de forma segura |
| `reliability.watchdog`                                        | Ajuste del tiempo de espera sin salida, por separado para ejecuciones nuevas y reanudadas |

Prefiera la configuración estática más pequeña que se ajuste a la CLI. Añada devoluciones de llamada del plugin
solo para comportamientos que realmente pertenezcan al backend.

## Hooks avanzados del backend

`CliBackendPlugin` también puede definir:

| Hook                               | Uso                                                                         |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)`                 | Normaliza el adaptador estático registrado con el contexto del entorno de ejecución |
| `resolveExecutionArgs(ctx)`                 | Añade indicadores con ámbito de solicitud, como el esfuerzo de razonamiento o el aislamiento de preguntas secundarias |
| `prepareExecution(ctx)`                 | Crea puentes temporales de autenticación, configuración o entorno antes del inicio |
| `transformSystemPrompt(ctx)`                 | Aplica una transformación final del prompt del sistema específica de la CLI |
| `textTransforms`                 | Sustituciones bidireccionales de prompt/salida                               |
| `defaultAuthProfileId`                 | Da preferencia a un perfil de autenticación específico de OpenClaw           |
| `authEpochMode`                 | Decide cómo los cambios de autenticación invalidan las sesiones almacenadas de la CLI |
| `nativeToolMode`                 | Declara si las herramientas nativas están ausentes, siempre activas o pueden seleccionarse en el host |
| `toolAvailabilityEnforcement`                 | Declara si los límites exactos de herramientas se aplican en argv o durante la preparación de la ejecución |
| `sideQuestionToolMode`                 | Declara las herramientas nativas deshabilitadas para preguntas secundarias de `/btw` |
| `bundleMcp` / `bundleMcpMode` | Habilita el puente de herramientas MCP de bucle invertido de OpenClaw        |
| `ownsNativeCompaction`                 | El backend gestiona su propia Compaction; OpenClaw la delega                 |
| `subscriptionAuthDispatch`                 | Las ejecuciones integradas habilitadas con credenciales de suscripción se realizan mediante este backend |
| `runtimeArtifact`                 | Vincula un iniciador de scripts a todo el árbol del paquete incluido         |

Mantenga estos hooks bajo la responsabilidad del proveedor. No añada ramas específicas de la CLI al núcleo cuando
un hook del backend pueda expresar el comportamiento.

`prepareExecution(ctx)` recibe `ctx.contextTokenBudget`, el límite efectivo de tokens
seleccionado para la ejecución. Los backends que gestionan la Compaction nativa pueden asignar ese
presupuesto a su contrato de inicio específico de la CLI.

`runtimeArtifact` pertenece al plugin. Se consulta
solo cuando un turno de inferencia activo crea o revalida una autoridad de configuración verificada;
las ejecuciones normales de la CLI no la requieren. Un backend sin esta declaración no puede
crear una autoridad de configuración de la CLI verificada. Una declaración `bundled-package-tree` identifica
al propietario exacto de `package.json` y requiere que el punto de entrada del paquete sea el
comando. OpenClaw calcula el hash del árbol completo y acotado del paquete instalado, incluidas
las dependencias anidadas, y adopta un cierre seguro ante enlaces simbólicos que redirigen,
iniciadores externos al paquete declarado, declaraciones de dependencias externas
requeridas, árboles sobredimensionados y scripts desconocidos. Declare esto solo cuando dicho
árbol contenga la implementación de inferencia completa; las integraciones opcionales de herramientas
no hacen que un grafo de implementación externo sea seguro.

Si el mismo backend también distribuye un ejecutable nativo autocontenido, indique sus
nombres base canónicos en `nativeExecutableNames`. Los demás comandos nativos permanecen
sin verificar.

`ctx.executionMode` es `"agent"` para los turnos normales y `"side-question"` para
las llamadas efímeras de `/btw`. Úselo cuando la CLI necesite marcas distintas para una sola ejecución,
como desactivar las herramientas nativas, la persistencia de sesiones o el comportamiento de reanudación para
BTW. Si un backend normalmente tiene `nativeToolMode: "always-on"`, pero los argumentos argv
de sus preguntas secundarias desactivan esas herramientas de forma fiable, establezca también
`sideQuestionToolMode: "disabled"`; de lo contrario, OpenClaw adopta un cierre seguro cuando BTW
requiere una ejecución de la CLI sin herramientas.

Establezca `nativeToolMode: "selectable"` solo cuando el backend pueda desactivar todas las
herramientas nativas del backend para una ejecución individual. Las ejecuciones restringidas reciben un contrato
canónico: `ctx.toolAvailability.native` es la lista exacta nativa del backend y
`ctx.toolAvailability.openClaw` es la lista exacta de nombres de herramientas de OpenClaw. El
host limita de forma independiente la configuración de MCP generada y la concesión a esa
lista de OpenClaw; los plugins no deben traducirla en el núcleo ni añadir prefijos de transporte.

Declare cómo aplica el backend ese contrato:

- `toolAvailabilityEnforcement: "execution-args"` requiere
  `resolveExecutionArgs`. El hook debe sustituir las marcas de herramientas en conflicto, desactivar
  las superficies de personalización que puedan ejecutarse fuera de las herramientas seleccionadas y
  devolver argumentos argv que apliquen las restricciones tanto para ejecuciones nuevas como reanudadas.
- `toolAvailabilityEnforcement: "prepare-execution"` requiere
  `prepareExecution`. El hook debe preparar una política exacta por ejecución y devolver
  `toolAvailabilityEnforced: true`; si falta la confirmación, se adopta un cierre seguro y
  OpenClaw limpia los recursos preparados antes del inicio.

OpenClaw normaliza y expande por grupos los límites de ejecución, como `toolsAllow` de Cron,
antes de crear este contrato. Las herramientas nativas se desactivan y un
backend sin una ruta de aplicación declarada completa falla antes de la ejecución.

Los plugins creados con versiones desde `v2026.7.2-beta.1` hasta `v2026.7.2-beta.3` aún pueden
leer la proyección obsoleta de nombres de transporte `ctx.toolAvailability.mcp` y
pueden omitir `toolAvailabilityEnforcement` cuando un backend seleccionable implementa
`resolveExecutionArgs`. OpenClaw reconoce esa ruta beta distribuida a partir de los
metadatos `openclaw.build.openclawVersion` requeridos del paquete del plugin y
la conserva durante la línea `2026.8.x`. Los plugins nuevos y actualizados deben usar nombres
canónicos `ctx.toolAvailability.openClaw` y declarar
`toolAvailabilityEnforcement: "execution-args"` explícitamente; está previsto eliminar la
ruta de compatibilidad beta después de ese período.

### `ownsNativeCompaction`: exclusión voluntaria de la Compaction de OpenClaw

Si el backend ejecuta un agente que compacta su **propia** transcripción, establezca
`ownsNativeCompaction: true` para que el resumidor de protección de OpenClaw nunca se ejecute
sobre sus sesiones: el ciclo de vida de Compaction de la CLI no realiza ninguna operación y el
turno continúa. `claude-cli` lo declara porque Claude Code compacta
internamente sin un endpoint del arnés. En cambio, las sesiones con arnés nativo, como Codex,
siguen dirigiéndose a su endpoint de Compaction del arnés.

**Declárelo únicamente cuando se cumpla todo lo siguiente**; de lo contrario, una sesión diferida
que supere el presupuesto puede permanecer por encima de este o quedar obsoleta (OpenClaw deja de
rescatarla):

- el backend compacta o limita de forma fiable su propia transcripción al acercarse a su
  ventana;
- conserva una sesión reanudable para que el estado compactado persista entre turnos
  (por ejemplo, `--resume` / `--session-id`);
- no es una sesión de Compaction con arnés nativo; las sesiones que coinciden con `agentHarnessId`
  se dirigen al endpoint del arnés.

## Puente de herramientas MCP

Los backends de la CLI no reciben herramientas de OpenClaw de forma predeterminada. Si la CLI puede consumir
una configuración de MCP, habilítela explícitamente:

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

Modos de puente compatibles:

| Modo                     | Uso                                                               |
| ------------------------ | ----------------------------------------------------------------- |
| `claude-config-file`     | CLI que aceptan un archivo de configuración de MCP                |
| `codex-config-overrides` | CLI que aceptan sustituciones de configuración mediante argv      |
| `gemini-system-settings` | CLI que leen la configuración de MCP desde su directorio de configuración del sistema |

Habilite el puente solo cuando la CLI pueda consumirlo realmente. Si la CLI tiene
su propia capa de herramientas integrada que no se puede desactivar, establezca `nativeToolMode:
"always-on"` para que OpenClaw pueda adoptar un cierre seguro cuando una llamada requiera que no haya herramientas
nativas. Si puede desactivar todas las herramientas nativas en cada ejecución, use `"selectable"` con el
contrato `resolveExecutionArgs` anterior.

## Selección del backend

Los usuarios seleccionan un backend independiente mediante el prefijo de su referencia de modelo. Un backend que
declare un `modelProvider` canónico puede seleccionarse, en cambio, mediante el
`agentRuntime.id` de ese modelo de proveedor. La mecánica del adaptador permanece en el plugin:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

Coloque las credenciales en los perfiles de autenticación de OpenClaw o en una configuración propiedad del plugin. Asegúrese de que el
comando registrado se encuentre en el `PATH` del servicio Gateway; las implementaciones que necesiten una
ruta o argumentos argv diferentes deben modificar o envolver el registro del plugin.

## Verificación

Para los plugins incluidos, añada una prueba específica para el constructor y el registro de
configuración; después, ejecute el conjunto de pruebas específico del plugin:

```bash
pnpm test extensions/acme-cli
```

Para los plugins locales o instalados, verifique el descubrimiento y una ejecución real del modelo:

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "responde exactamente: backend ok" --model acme-cli/acme-large
```

Si el backend admite imágenes o MCP, añada una prueba de humo en vivo que demuestre esas
rutas con la CLI real. No dependa de la inspección estática para comprobar el comportamiento de los prompts, las imágenes,
MCP o la reanudación de sesiones.

## Lista de comprobación

<Check>`package.json` tiene `openclaw.extensions` y entradas de ejecución compiladas para los paquetes publicados</Check>
<Check>`openclaw.plugin.json` declara `cliBackends` y un `activation.onStartup` intencional</Check>
<Check>`setup.cliBackends` está presente cuando la configuración o el descubrimiento de modelos deben detectar el backend en frío</Check>
<Check>`api.registerCliBackend(...)` usa el mismo id de backend que el manifiesto</Check>
<Check>El prefijo del modelo del backend o el `agentRuntime.id` limitado al modelo selecciona el registro</Check>
<Check>La configuración de la sesión, el prompt del sistema, las imágenes y el analizador de salida coincide con el contrato real de la CLI</Check>
<Check>Las pruebas específicas y al menos una prueba de humo de la CLI en vivo demuestran la ruta del backend</Check>

## Contenido relacionado

- [Backends de la CLI](/es/gateway/cli-backends) - selección y comportamiento en tiempo de ejecución
- [Creación de plugins](/es/plugins/building-plugins) - conceptos básicos de paquetes y manifiestos
- [Descripción general del SDK de plugins](/es/plugins/sdk-overview) - referencia de la API de registro
- [Manifiesto del plugin](/es/plugins/manifest) - `cliBackends` y descriptores de configuración
- [Arnés de agentes](/es/plugins/sdk-agent-harness) - entornos de ejecución completos para agentes externos
