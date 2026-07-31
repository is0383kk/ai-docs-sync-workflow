---
read_when:
    - Quieres servir modelos desde tu propio equipo con GPU
    - Está configurando LM Studio o un proxy compatible con OpenAI
    - Necesita orientación sobre el modelo local más seguro
summary: Ejecuta OpenClaw con LLM locales (LM Studio, vLLM, LiteLLM, endpoints personalizados de OpenAI)
title: Modelos locales
x-i18n:
    generated_at: "2026-07-26T04:40:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: af76c9e97bd1d3c9665c347944511b4f466f0b620bb8af7b5f95b1e9145aadec
    source_path: gateway/local-models.md
    workflow: 16
---

Los modelos locales funcionan, pero elevan los requisitos de hardware, tamaño de contexto y defensa contra la inyección de prompts: los modelos pequeños o cuantizados de forma agresiva truncan el contexto y omiten los filtros de seguridad del proveedor. Esta página abarca pilas locales de gama alta y servidores personalizados compatibles con OpenAI. Para seguir la vía más sencilla, se recomienda comenzar con [LM Studio](/es/providers/lmstudio) u [Ollama](/es/providers/ollama) y `openclaw onboard`.

Para servidores locales que solo deban iniciarse cuando los necesite un modelo seleccionado, consulte [Servicios de modelos locales](/es/gateway/local-model-services).

## Requisitos mínimos de hardware

Se recomienda contar con **2 o más Mac Studio con la configuración máxima o un equipo con GPU equivalente (~$30k+)** para disponer de un bucle de agente cómodo. Una sola GPU de **24 GB** únicamente admite prompts más ligeros con una latencia mayor. Ejecute siempre la **variante más grande o de tamaño completo que pueda alojar**; los puntos de control pequeños o con una cuantización intensa aumentan el riesgo de inyección de prompts (consulte [Seguridad](/es/gateway/security)).

## Elección de un backend

| Backend                                              | Cuándo utilizarlo                                                                                         |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| [ds4](/es/providers/ds4)                                | DeepSeek V4 Flash local en Metal de macOS con llamadas a herramientas compatibles con OpenAI              |
| [LM Studio](/es/providers/lmstudio)                     | Primera configuración local, cargador con interfaz gráfica y API Responses nativa                         |
| LiteLLM / OAI-proxy / proxy personalizado compatible con OpenAI | Cuando se utiliza otra API de modelos como frontend y se necesita que OpenClaw la trate como OpenAI |
| MLX / vLLM / SGLang                                  | Servicio autoalojado de alto rendimiento con un endpoint HTTP compatible con OpenAI                       |
| [Ollama](/es/providers/ollama)                          | Flujo de trabajo mediante CLI, biblioteca de modelos y servicio systemd sin intervención                  |

Utilice `api: "openai-responses"` cuando el backend lo admita (LM Studio lo hace). De lo contrario, utilice `api: "openai-completions"`. Si se omite `api` en un proveedor personalizado con un `baseUrl`, OpenClaw utiliza `openai-completions` de forma predeterminada.

<Warning>
**WSL2 + Ollama + NVIDIA/CUDA:** el instalador oficial de Ollama para Linux habilita un servicio systemd con `Restart=always`. En configuraciones de GPU con WSL2, el inicio automático puede volver a cargar el último modelo durante el arranque y retener la memoria del host, lo que provoca reinicios repetidos de la máquina virtual. Consulte [Bucle de fallos de WSL2](/es/providers/ollama#troubleshooting).
</Warning>

## LM Studio + modelo local grande (API Responses)

Esta es actualmente la mejor pila local. Cargue un modelo grande en LM Studio (una compilación de tamaño completo de Qwen, DeepSeek o Llama), habilite el servidor local (valor predeterminado: `http://127.0.0.1:1234`) y utilice la API Responses para mantener el razonamiento separado del texto final.

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "lmstudio/my-local-model": { alias: "Local" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Lista de comprobación para la configuración:

- Instale LM Studio: [https://lmstudio.ai](https://lmstudio.ai)
- Descargue la **compilación de modelo más grande disponible** (evite las variantes «small» o con una cuantización intensa), inicie el servidor y confirme que `http://127.0.0.1:1234/v1/models` lo incluya.
- Sustituya `my-local-model` por el ID real del modelo que se muestra en LM Studio.
- Mantenga el modelo cargado; una carga en frío aumenta la latencia de inicio.
- Ajuste `contextWindow`/`maxTokens` si su compilación de LM Studio es diferente.
- Para WhatsApp, utilice la API Responses para que solo se envíe el texto final.
- Mantenga `models.mode: "merge"` para que los modelos alojados sigan disponibles como alternativas.

### Configuración híbrida: modelo alojado principal y modelo local alternativo

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Para dar prioridad al modelo local y disponer de un modelo alojado como respaldo, intercambie el orden de `primary`/`fallbacks` y mantenga el mismo bloque `providers` y `models.mode: "merge"`.

### Alojamiento regional y enrutamiento de datos

También existen variantes alojadas de MiniMax/Kimi/GLM en OpenRouter con endpoints asignados a regiones específicas (por ejemplo, alojados en Estados Unidos). Elija la variante regional para mantener el tráfico en la jurisdicción seleccionada y conserve `models.mode: "merge"` para las alternativas de Anthropic/OpenAI. El uso exclusivamente local sigue siendo la opción con mayor privacidad; el enrutamiento regional alojado constituye el término medio cuando se necesitan las funciones del proveedor, pero se desea mantener el control sobre el flujo de datos.

## Otros proxies locales compatibles con OpenAI

MLX (`mlx_lm.server`), vLLM, SGLang, LiteLLM, OAI-proxy o cualquier Gateway personalizado funcionan si exponen un endpoint `/v1/chat/completions` de estilo OpenAI. Utilice `openai-completions`, salvo que la documentación del backend indique explícitamente que admite `/v1/responses`.

```json5
{
  agents: {
    defaults: {
      model: { primary: "local/my-local-model" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

Las entradas de proveedores personalizados o locales consideran de confianza el origen `baseUrl` configurado de forma exacta para las solicitudes de modelos protegidas, incluidos los hosts de bucle invertido, LAN, tailnet y DNS privado. Los orígenes de metadatos o de vínculo local se bloquean siempre. Las solicitudes a otros orígenes privados siguen necesitando `models.providers.<id>.request.allowPrivateNetwork: true`; establezca la marca de confianza en `false` para desactivar la confianza en el origen exacto.

`models.providers.<id>.models[].id` es local al proveedor; no incluya el prefijo del proveedor. Para un servidor MLX iniciado con `mlx_lm.server --model mlx-community/Qwen3-30B-A3B-6bit`:

- `models.providers.mlx.models[].id: "mlx-community/Qwen3-30B-A3B-6bit"`
- `agents.defaults.model.primary: "mlx/mlx-community/Qwen3-30B-A3B-6bit"`

Establezca `input: ["text", "image"]` en los modelos de visión locales o con proxy para que los archivos adjuntos de imágenes se inserten en los turnos del agente. La incorporación interactiva de proveedores personalizados infiere los ID habituales de modelos de visión y solo pregunta por los nombres desconocidos; la incorporación no interactiva utiliza la misma inferencia, con `--custom-image-input` / `--custom-text-input` para anularla.

Utilice `models.providers.<id>.timeoutSeconds` para servidores de modelos locales o remotos lentos antes de aumentar `agents.defaults.timeoutSeconds`. El tiempo de espera del proveedor abarca la conexión, las cabeceras, la transmisión del cuerpo y la cancelación total de la obtención protegida únicamente para las solicitudes HTTP del modelo. Si el tiempo de espera del agente o de la ejecución es inferior, auméntelo también, ya que el tiempo de espera del proveedor no puede prolongar toda la ejecución.

<Note>
Para los proveedores personalizados compatibles con OpenAI, se acepta un marcador local no secreto como `apiKey: "ollama-local"` cuando `baseUrl` se resuelve en un bucle invertido, una LAN privada, `.local` o un nombre de host sin calificar; OpenClaw lo trata como una credencial local válida en lugar de informar que falta una clave. Utilice un valor real para cualquier proveedor que acepte un nombre de host público.
</Note>

Notas de comportamiento para backends `/v1` locales o con proxy:

- OpenClaw los trata como rutas de proxy compatibles con OpenAI, no como endpoints nativos de OpenAI.
- No se aplica la conformación de solicitudes exclusiva de OpenAI nativo: no se incluye `service_tier`, ni `store` de Responses, ni conformación de cargas útiles para la compatibilidad del razonamiento de OpenAI, ni indicaciones para la caché de prompts.
- Las cabeceras ocultas de atribución de OpenClaw (`originator`, `version`, `User-Agent`) no se insertan en las URL de proxies personalizados.

Las declaraciones de compatibilidad solo se aplican al endpoint personalizado descrito por esta fila del proveedor. En su lugar, las rutas conocidas por el catálogo utilizan capacidades propias del proveedor; consulte la [guía de capacidades de proveedores personalizados](/es/gateway/config-tools#custom-provider-capability-declarations).

Anulaciones de compatibilidad para backends compatibles con OpenAI más estrictos:

- **Contenido solo de cadena:** algunos servidores solo aceptan `messages[].content` como cadena, no matrices estructuradas de partes de contenido. Establezca `models.providers.<provider>.models[].compat.requiresStringContent: true`.
- **Claves de mensaje estrictas:** si el servidor rechaza entradas de mensajes con más claves que `role`/`content`, establezca `compat.strictMessageKeys: true`.
- **Texto de herramientas entre corchetes:** algunos modelos locales emiten solicitudes de herramientas independientes entre corchetes como texto, por ejemplo, `[tool_name]` seguido de JSON y `[END_TOOL_REQUEST]`. OpenClaw las convierte en llamadas reales a herramientas solo cuando el nombre coincide exactamente con una herramienta registrada para el turno; de lo contrario, permanecen como texto oculto no admitido.
- **Texto no estructurado con aspecto de llamada a herramienta:** si un modelo emite texto de estilo JSON/XML/ReAct que parece una llamada a herramienta, pero no era una invocación estructurada, OpenClaw lo conserva como texto y registra una advertencia con el ID de ejecución, el proveedor/modelo, el patrón detectado y el nombre de la herramienta cuando esté disponible. Esto constituye una incompatibilidad del proveedor/modelo, no una ejecución de herramienta completada.
- **Forzar el uso de herramientas:** si las herramientas aparecen como texto del asistente (JSON/XML/ReAct sin procesar o una matriz `tool_calls` vacía), confirme primero que la plantilla o el analizador de chat del servidor admita llamadas a herramientas. Si el analizador solo funciona cuando se fuerza el uso de herramientas, anule el valor predeterminado del proxy `tool_choice: "auto"` para cada modelo:

  ```json5
  {
    agents: {
      defaults: {
        models: {
          "local/my-local-model": {
            params: {
              extra_body: {
                tool_choice: "required",
              },
            },
          },
        },
      },
    },
  }
  ```

  Utilice esta opción únicamente cuando todos los turnos normales deban llamar a una herramienta. Sustituya `local/my-local-model` por la referencia exacta de `openclaw models list`, o establézcala mediante la CLI:

  ```bash
  openclaw config set agents.defaults.models '{"local/my-local-model":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
  ```

- **Niveles adicionales de esfuerzo de razonamiento:** si un modelo personalizado compatible con OpenAI acepta niveles de esfuerzo de razonamiento de OpenAI adicionales a los del perfil integrado, declárelos en el bloque de compatibilidad del modelo. Añadir `"xhigh"` lo expone para esa referencia de modelo en `/think xhigh`, los selectores de sesión, la validación del Gateway y la validación de `llm-task`:

  ```json5
  {
    models: {
      providers: {
        local: {
          baseUrl: "http://127.0.0.1:8000/v1",
          apiKey: "sk-local",
          api: "openai-responses",
          models: [
            {
              id: "gpt-5.4",
              name: "GPT 5.4 mediante proxy local",
              reasoning: true,
              input: ["text"],
              cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
              contextWindow: 196608,
              maxTokens: 8192,
              compat: {
                supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
                reasoningEffortMap: { xhigh: "xhigh" },
              },
            },
          ],
        },
      },
    },
  }
  ```

## Backends más pequeños o estrictos

Si el modelo se carga correctamente, pero los turnos completos del agente no funcionan bien, proceda de arriba abajo: confirme primero el transporte y, después, reduzca la superficie.

1. **Confirme que el modelo local responde**: sin herramientas ni contexto del agente:

   ```bash
   openclaw infer model run --local --model <provider/model> --prompt "Responde exactamente: pong" --json
   ```

2. **Confirme el enrutamiento del Gateway**: envía solo el prompt, omitiendo la transcripción, la inicialización de AGENTS, el ensamblaje del motor de contexto, las herramientas y los servidores MCP incluidos, pero sigue poniendo a prueba el enrutamiento del Gateway, la autenticación y la selección del proveedor:

   ```bash
   openclaw infer model run --gateway --model <provider/model> --prompt "Responde exactamente: pong" --json
   ```

3. **Pruebe el modo ligero** si ambas pruebas tienen éxito, pero los turnos reales del agente fallan con llamadas a herramientas mal formadas o prompts demasiado grandes: establezca `agents.defaults.experimental.localModelLean: true`. Este modo descarta las herramientas pesadas de navegador, cron, mensajería, generación de contenido multimedia, voz y PDF, salvo que se requieran explícitamente, y coloca de forma predeterminada los catálogos de herramientas más grandes detrás de controles estructurados de búsqueda de herramientas, mientras mantiene `exec` visible directamente. Consulte [Funciones experimentales -> Modo ligero de modelos locales](/es/concepts/experimental-features#local-model-lean-mode) para obtener más información y saber cómo confirmar que está activado.

4. **Desactive por completo las herramientas como último recurso** estableciendo `models.providers.<provider>.models[].compat.supportsTools: false` para ese modelo; el agente se ejecutará entonces sin llamadas a herramientas.

5. **Más allá de eso, el cuello de botella está en el componente anterior.** Si el backend sigue fallando solo en ejecuciones más grandes de OpenClaw después de activar el modo ligero y `supportsTools: false`, el problema restante suele estar en el propio modelo o servidor —la ventana de contexto, la memoria de la GPU, el desalojo de la caché KV o un error del backend—, no en la capa de transporte de OpenClaw.

## Solución de problemas

- **¿El Gateway no puede acceder al proxy?** `curl http://127.0.0.1:1234/v1/models`.
- **¿Se descargó el modelo de LM Studio?** Vuelva a cargarlo; un inicio en frío es una causa habitual de que parezca «bloqueado».
- **¿El servidor local indica `terminated`, `ECONNRESET` o cierra el flujo a mitad del turno?** OpenClaw registra en los diagnósticos un `model.call.error.failureKind` de baja cardinalidad junto con una instantánea de la RSS y el heap del proceso de OpenClaw. En caso de presión de memoria de LM Studio/Ollama, compare esa marca de tiempo con el registro del servidor o con un registro de bloqueo/jetsam de macOS para confirmar si se terminó el proceso del servidor del modelo.
- **¿Hay errores de contexto?** OpenClaw deriva los umbrales de comprobación previa de la ventana de contexto a partir de la ventana detectada del modelo (o de la ventana limitada cuando `agents.defaults.contextTokens` la reduce): muestra una advertencia por debajo del 20% con un mínimo de **8k** y bloquea por completo por debajo del 10% con un mínimo de **4k** (limitado a la ventana de contexto efectiva para que unos metadatos sobredimensionados del modelo no rechacen un límite válido del usuario). Reduzca `contextWindow` o aumente el límite de contexto del servidor/modelo.
- **¿`messages[].content ... expected a string`?** Añada `compat.requiresStringContent: true` a la entrada de ese modelo.
- **¿`validation.keys`, o «las entradas de mensajes solo permiten `role` y `content`»?** Añada `compat.strictMessageKeys: true` a la entrada de ese modelo.
- **¿Las llamadas directas a `/v1/chat/completions` funcionan, pero `openclaw infer model run --local` falla con Gemma u otro modelo local?** Compruebe primero la URL del proveedor, la referencia del modelo, el marcador de autenticación y los registros del servidor; `model run` omite por completo las herramientas del agente. Si `model run` tiene éxito, pero los turnos más grandes del agente fallan, reduzca la superficie de herramientas con `localModelLean` o `compat.supportsTools: false`.
- **¿Las llamadas a herramientas aparecen como texto JSON/XML/ReAct sin procesar, o el proveedor devuelve un array `tool_calls` vacío?** No añada un proxy que convierta indiscriminadamente el texto del asistente en ejecuciones de herramientas; corrija primero la plantilla o el analizador de chat del servidor. Si el modelo solo funciona cuando se fuerza el uso de herramientas, añada la sobrescritura `params.extra_body.tool_choice: "required"` anterior y use esa entrada del modelo únicamente en sesiones donde se espere una llamada a herramientas en cada turno.
- **Seguridad**: los modelos locales omiten los filtros del proveedor. Mantenga los agentes con un alcance reducido y Compaction activado para limitar el alcance de las inyecciones de prompts.

## Contenido relacionado

- [Referencia de configuración](/es/gateway/configuration-reference)
- [Conmutación por error de modelos](/es/concepts/model-failover)
