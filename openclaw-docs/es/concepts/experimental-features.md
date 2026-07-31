---
read_when:
    - Ves una clave de configuración `.experimental` y quieres saber si es estable
    - Quieres probar las funciones experimentales del entorno de ejecución sin confundirlas con los valores predeterminados normales
    - Se busca un único lugar donde encontrar las opciones experimentales documentadas actualmente
summary: Qué significan las opciones experimentales en OpenClaw y cuáles están documentadas actualmente
title: Funciones experimentales
x-i18n:
    generated_at: "2026-07-26T04:35:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6c14b74bbafce77c0d1e1358ad94053675c4aad9e26be78719f58e78f455c3a2
    source_path: concepts/experimental-features.md
    workflow: 16
---

Las funciones experimentales son superficies preliminares disponibles mediante indicadores explícitos. Necesitan más experiencia de uso en entornos reales antes de adoptar un valor predeterminado estable o un contrato duradero.

- Desactivadas de forma predeterminada, salvo que un documento describa una regla específica de configuración automática.
- La forma y el comportamiento pueden cambiar más rápido que la configuración estable.
- Es preferible usar una vía estable cuando ya exista una.
- Impleméntelas de forma generalizada solo después de probarlas primero en un entorno más pequeño.

## Indicadores documentados actualmente

| Superficie                    | Clave                                                                                         | Cuándo usarla                                                                                                                           | Más información                                                                                  |
| ----------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Runtime de modelos locales    | `agents.defaults.experimental.localModelLean`, `agents.entries.*.experimental.localModelLean` | Un backend local más pequeño o estricto no puede procesar toda la superficie de herramientas predeterminada de OpenClaw                | [Modelos locales](/es/gateway/local-models)                                                         |
| Arnés de Codex                | `plugins.entries.codex.config.appServer.experimental.sandboxExecServer`                       | Se desea que el servidor de aplicaciones nativo de Codex 0.132.0 o posterior use un servidor de ejecución respaldado por un sandbox de OpenClaw en lugar de desactivar el Modo Código | [Referencia del arnés de Codex](/es/plugins/codex-harness-reference#sandboxed-native-execution) |
| Herramienta de planificación estructurada | `tools.experimental.planTool`                                                                 | Se desea exponer la herramienta estructurada `update_plan` para realizar el seguimiento de trabajos de varios pasos en runtimes e interfaces de usuario compatibles | [Referencia de configuración del Gateway](/es/gateway/config-tools#toolsexperimental) |
| Modo Código                   | `tools.codeMode.enabled`                                                                      | Se desea acceder mediante código orquestado y compacto a un catálogo oculto de herramientas de OpenClaw                                | [Modo Código](/es/tools/code-mode)                                                                 |
| Enjambre                      | `tools.swarm.enabled`                                                                         | Se desea que los scripts del Modo Código orquesten en paralelo grupos acotados de subagentes                                           | [Enjambre](/es/tools/swarm)                                                                        |

## Laboratorios de la interfaz de control

Abra **Configuración → Agentes y herramientas → Laboratorios** para gestionar los experimentos que disponen de un
interruptor en la interfaz de control. Al activar o desactivar un laboratorio, se modifica inmediatamente la configuración
canónica del Gateway; la página solo muestra una indicación para reiniciar cuando una función
lo requiere.

Modo Código y Enjambre son las entradas de Laboratorios disponibles actualmente. Ambos interruptores
escriben claves de configuración existentes y validadas, y normalmente se aplican a futuras
ejecuciones de agentes sin reiniciar el Gateway.

## Modo ligero para modelos locales

`agents.defaults.experimental.localModelLean: true` elimina en cada turno las herramientas opcionales de gran tamaño de la superficie directa del agente: `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` y `pdf`. Las herramientas permitidas explícitamente o necesarias para la entrega siguen disponibles, aunque la búsqueda de herramientas puede catalogarlas en lugar de exponerlas directamente. El modo ligero también configura de forma predeterminada los catálogos de plugins/MCP/clientes para la búsqueda estructurada de herramientas (`tool_search`, `tool_describe`, `tool_call`) cuando `tools.toolSearch` aún no está establecido. Use `agents.entries.*.experimental.localModelLean` para limitarlo a un agente.

Durante la incorporación, una ruta de inferencia `ollama` o `lmstudio` verificada establece automáticamente `agents.defaults.experimental.localModelLean: true` cuando ese valor no está presente. OpenClaw registra que el ajuste procede de la incorporación, por lo que una ruta no local verificada posteriormente solo elimina el ajuste automático. Se conserva cualquier valor de `true` o `false` configurado explícitamente. Los demás proveedores autoalojados y compatibles con OpenAI no se infieren a partir de nombres de modelos ni de URL.

Si ya se ajusta globalmente la búsqueda de herramientas, OpenClaw no modifica esa configuración. Establezca `tools.toolSearch: false` para no usar el valor predeterminado de búsqueda de herramientas del modo ligero.

En el modo estructurado `tools`, las ejecuciones ligeras mantienen `exec` visible directamente junto a los controles de búsqueda de herramientas, de modo que los modelos locales optimizados para programación puedan seguir eligiendo su vía de shell habitual. Esto solo cambia la visibilidad del esquema: se siguen aplicando la política normal de herramientas, el aislamiento mediante sandbox y las aprobaciones de ejecución. Los modos explícitos `code` y `directory` conservan su comportamiento normal de Compaction.

### Motivos para elegir estas herramientas

Estas herramientas tienen las descripciones más extensas, las estructuras de parámetros más amplias o la mayor probabilidad de distraer a un modelo pequeño de la vía normal de programación y conversación. En un backend compatible con OpenAI que tenga un contexto pequeño o sea más estricto, esto marca la diferencia entre:

- Que los esquemas de herramientas quepan en el prompt o desplacen el historial de conversación.
- Que el modelo elija la herramienta correcta o emita llamadas de herramientas mal formadas debido a la existencia de demasiados esquemas similares.
- Que el adaptador de finalizaciones de chat se mantenga dentro de los límites de salida estructurada o se produzca un error 400 por el tamaño de la carga útil de las llamadas de herramientas.

Eliminarlas solo acorta la lista directa de herramientas. El modelo sigue teniendo `read`, `write`, `edit`, `exec`, `apply_patch`, comprensión de imágenes, búsqueda/obtención web (cuando estén configuradas), memoria y herramientas de sesión/agente. Los catálogos adicionales siguen siendo accesibles mediante la búsqueda de herramientas, salvo que se establezca `tools.toolSearch: false`; los permisos explícitos de herramientas pueden volver a incluir un agente ligero en un flujo de trabajo reducido.

### Cuándo activarlo

Active el modo ligero cuando se haya comprobado que el modelo puede comunicarse con el Gateway, pero los turnos completos del agente no funcionen correctamente:

1. `openclaw infer model run --gateway --model <ref> --prompt "Reply with exactly: pong"` se ejecuta correctamente.
2. Un turno normal del agente falla por llamadas de herramientas mal formadas, prompts demasiado grandes o porque el modelo ignora sus herramientas.
3. Cambiar `localModelLean: true` elimina el fallo.

### Cuándo dejarlo desactivado

Si el backend gestiona correctamente todo el runtime predeterminado, deje esta opción desactivada. Es una solución alternativa para pilas locales que necesitan una superficie de herramientas más pequeña, no un valor predeterminado para modelos alojados ni para equipos locales con recursos suficientes.

El modo ligero no sustituye a `tools.profile`, `tools.allow`/`tools.deny` ni a la vía de escape `compat.supportsTools: false` del modelo. Para disponer de forma permanente de una superficie de herramientas más limitada en un agente específico, es preferible usar esos controles estables.

### Activación

```json5
{
  agents: {
    defaults: {
      experimental: {
        localModelLean: true,
      },
    },
  },
}
```

Solo para un agente:

```json5
{
  agents: {
    list: [
      {
        id: "local",
        model: "lmstudio/gemma-4-e4b-it",
        experimental: {
          localModelLean: true,
        },
      },
    ],
  },
}
```

Reinicie el Gateway después de cambiar el indicador. El filtrado ligero elimina `browser`, `cron`, `message`, `image_generate`, `music_generate`, `video_generate`, `tts` y `pdf`, salvo que se conserven explícitamente mediante `tools.allow` o `tools.alsoAllow`; la búsqueda de herramientas aún puede catalogar las herramientas conservadas en lugar de exponerlas directamente.

## Experimental no significa oculto

Una función experimental debe indicarlo claramente en la documentación y en la propia ruta de configuración, en lugar de ocultarse tras un control predeterminado que parezca estable.

## Contenido relacionado

- [Funciones](/es/concepts/features)
- [Canales de publicación](/es/install/development-channels)
