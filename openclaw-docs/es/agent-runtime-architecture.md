---
summary: 'Cómo estructura OpenClaw el entorno de ejecución integrado del agente: organización del código, límites, manifiestos de recursos y selección del entorno de ejecución.'
title: Arquitectura del entorno de ejecución del agente
x-i18n:
    generated_at: "2026-07-26T04:29:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e09ff21b4369a7c102db51e4458ad3ba1e86c9fe43a3a8bff72eef1713d2d51
    source_path: agent-runtime-architecture.md
    workflow: 16
---

OpenClaw es propietario del entorno de ejecución de agentes integrado. El código del entorno de ejecución se encuentra en `src/agents/`, el transporte de modelos/proveedores se encuentra en `src/llm/` y los contratos orientados a plugins se exponen mediante los barrels de `openclaw/plugin-sdk/*`.

## Estructura del entorno de ejecución

| Ruta                                | Responsabilidad                                                                                                                                                                                                                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agents/embedded-agent-runner/` | Bucle de intentos integrado (`run.ts`, `run/`), selección de modelos y normalización de proveedores (`model*.ts`), parámetros de solicitud por proveedor (`extra-params.*`), Compaction y conexión de transcripciones y sesiones.                            |
| `src/agents/sessions/`              | Persistencia de sesiones (`session-manager.ts`), detección de recursos (`package-manager.ts`, `resource-loader.ts`), carga de `extensions` en la sesión, plantillas de prompts, Skills, temas y renderizadores de herramientas basados en TUI (`tools/`). |
| `packages/agent-core/`              | Núcleo de agente reutilizable (`@openclaw/agent-core`): bucle del agente, tipos del arnés, mensajes, auxiliares de Compaction, plantillas de prompts, Skills y contratos de almacenamiento de sesiones.                                                           |
| `src/agents/runtime/`               | Fachada de OpenClaw que conecta `@openclaw/agent-core` con el entorno de ejecución de LLM del SDK de plugins y lo reexporta junto con utilidades de proxy locales.                                                                                             |
| `src/agents/agent-tools*.ts`        | Definiciones de herramientas propiedad de OpenClaw, esquemas de parámetros, políticas de herramientas, adaptadores anteriores y posteriores a las llamadas de herramientas, y herramientas de edición del host y del entorno aislado.                                                                                            |
| `src/agents/agent-hooks/`           | Hooks integrados del entorno de ejecución: protección de Compaction, instrucciones de Compaction y poda de contexto.                                                                                                                                   |
| `src/agents/harness/`               | Registro, política de selección y ciclo de vida de los arneses integrados y registrados por plugins.                                                                                                                       |
| `src/llm/`                          | Registro de modelos/proveedores, auxiliares de transporte e implementaciones de flujo específicas del proveedor (`src/llm/providers/`).                                                                                                          |

## Límites

El núcleo llama al entorno de ejecución integrado mediante módulos de OpenClaw y barrels del SDK; ya no queda ningún paquete externo de marcos de agentes. Los plugins utilizan los puntos de entrada documentados de `openclaw/plugin-sdk/*` y no importan elementos internos de `src/**`.

`@earendil-works/pi-tui` sigue siendo una dependencia de terceros: un kit de herramientas de componentes de terminal utilizado por la TUI local y los renderizadores de herramientas de sesión. Su internalización requeriría un esfuerzo de incorporación de dependencias independiente.

## Manifiestos

Los paquetes de recursos declaran recursos de OpenClaw en los metadatos de `package.json`. Las entradas son rutas de archivos o patrones glob relativos a la raíz del paquete:

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

Los tipos de recursos que no figuran en un manifiesto recurren a la detección de los directorios convencionales `extensions/`, `skills/`, `prompts/` y `themes/`.

## Selección del entorno de ejecución

- El identificador del entorno de ejecución integrado es `openclaw`. El alias heredado `pi` se normaliza como `openclaw`; `codex-app-server` se normaliza como `codex`.
- Los arneses de plugins registran identificadores adicionales de entornos de ejecución (por ejemplo, `codex`).
- La política del entorno de ejecución es una configuración `agentRuntime.id` con alcance de modelo/proveedor (la entrada del modelo prevalece sobre la del proveedor). Un valor no establecido o `default` se resuelve como `auto`.
- `auto` selecciona un arnés de Plugin registrado que admita la ruta efectiva del proveedor; de lo contrario, selecciona el entorno de ejecución integrado de OpenClaw. Un prefijo de proveedor o modelo por sí solo nunca selecciona un arnés.
- OpenAI puede seleccionar implícitamente `codex` solo para una ruta oficial HTTPS exacta de Platform Responses o ChatGPT Responses sin ninguna sobrescritura de solicitud definida. Los adaptadores de Completions, los endpoints personalizados y las rutas con comportamiento de solicitud definido permanecen en `openclaw`; los endpoints HTTP oficiales sin cifrar se rechazan. Consulte [Entorno de ejecución de agentes implícito de OpenAI](/es/providers/openai#implicit-agent-runtime).

## Generaciones del entorno de ejecución de modelos

El inicio del Gateway y la publicación de configuración, plugins o autenticación crean una generación preparada del entorno de ejecución de modelos por cada agente configurado. Cada generación contiene la plantilla de autenticación detectada, el registro de modelos y el catálogo de modelos proyectado como una única instantánea atómica. Las ejecuciones de agentes bifurcan almacenes mutables de autenticación y registro a partir de esa instantánea; las rutas de exploración, estado, Cron, diagnóstico, TUI, PDF e imágenes leen el catálogo publicado en lugar de repetir la detección del sistema de archivos.

Los entornos de ejecución integrados independientes publican la misma estructura de instantánea en su límite de activación. Una generación fallida u obsoleta nunca se sirve junto con una generación parcial más reciente; el propietario del ciclo de vida debe publicar primero un reemplazo completo.

## Temas relacionados

- [Flujo de trabajo del entorno de ejecución de agentes de OpenClaw](/es/openclaw-agent-runtime)
- [Entornos de ejecución de agentes](/es/concepts/agent-runtimes)
