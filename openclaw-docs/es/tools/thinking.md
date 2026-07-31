---
read_when:
    - Ajuste del análisis de las directivas o los valores predeterminados de pensamiento, modo rápido o modo detallado
summary: Sintaxis de las directivas /think, /fast, /verbose, /trace y visibilidad del razonamiento
title: Niveles de razonamiento
x-i18n:
    generated_at: "2026-07-26T04:57:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80968ce58f642090ba0f807874e43eea1206cd31d919414c690b7537dc523658
    source_path: tools/thinking.md
    workflow: 16
---

## Qué hace

- Directiva insertada en cualquier cuerpo entrante: `/t <level>`, `/think:<level>` o `/thinking <level>`.
- Niveles (alias): `off | minimal | low | medium | high | xhigh | adaptive | max | ultra`, que reflejan aproximadamente la clásica escala de palabras mágicas de Anthropic «think» < «think hard» < «think harder» < «ultrathink»:
  - minimal ~ «pensar»
  - low ~ «pensar intensamente»
  - medium ~ «pensar con mayor intensidad»
  - high ~ «ultrathink» (presupuesto máximo)
  - xhigh ~ «ultrathink+» (modelos GPT-5.2+ y Codex, además del esfuerzo de Anthropic Claude Opus 4.7+)
  - adaptive → pensamiento adaptativo administrado por el proveedor (compatible con Claude 4.6 en Anthropic/Bedrock, Anthropic Claude Opus 4.7+ y el pensamiento dinámico de Google Gemini)
  - max → razonamiento máximo del proveedor (Anthropic Claude Opus 4.7+; Ollama lo asigna a su máximo esfuerzo nativo `think`)
  - ultra → razonamiento máximo del proveedor más orquestación proactiva de subagentes cuando el modelo o entorno de ejecución seleccionado lo admite
  - `x-high`, `x_high`, `extra-high`, `extra high` y `extra_high` se asignan a `xhigh`.
  - `highest` se asigna a `high`.
- Notas sobre proveedores:
  - Los menús y selectores de pensamiento dependen del perfil del proveedor. Los plugins de proveedor declaran el conjunto exacto de niveles del modelo seleccionado, incluidas etiquetas como la binaria `on`.
  - `adaptive`, `xhigh`, `max` y `ultra` solo se anuncian para perfiles de proveedor, modelo o entorno de ejecución compatibles. Las directivas escritas para niveles no compatibles se rechazan e indican las opciones válidas para ese modelo.
  - Los niveles no compatibles almacenados anteriormente se reasignan según la clasificación del perfil del proveedor. `adaptive` recurre a `medium` en modelos no adaptativos, mientras que `xhigh` y `max` recurren al mayor nivel compatible distinto de desactivado para el modelo seleccionado.
  - Los modelos Anthropic Claude 4.6 usan `adaptive` de forma predeterminada cuando no se establece ningún nivel de pensamiento explícito.
  - Anthropic Claude Opus 4.8 y Opus 4.7 mantienen el pensamiento desactivado a menos que se establezca explícitamente un nivel de pensamiento. El esfuerzo predeterminado propiedad del proveedor de Opus 4.8 es `high` después de habilitar el pensamiento adaptativo.
  - Anthropic Claude Opus 4.7+ asigna `/think xhigh` al pensamiento adaptativo más `output_config.effort: "xhigh"`, porque `/think` es una directiva de pensamiento y `xhigh` es la configuración de esfuerzo de Opus.
  - Anthropic Claude Opus 4.7+ también ofrece `/think max`; se asigna a la misma ruta de esfuerzo máximo propiedad del proveedor.
  - Los modelos DeepSeek V4 directos ofrecen `/think xhigh|max`; ambos se asignan a `reasoning_effort: "max"` de DeepSeek, mientras que los niveles inferiores distintos de desactivado se asignan a `high`.
  - Los modelos DeepSeek V4 enrutados mediante OpenRouter ofrecen `/think xhigh` y envían valores `reasoning.effort` compatibles con OpenRouter en lugar del valor `reasoning_effort` de nivel superior nativo de DeepSeek. Los niveles inferiores distintos de desactivado se asignan a `high` y las anulaciones `max` almacenadas recurren a `xhigh`.
  - Los modelos de Ollama con capacidad de pensamiento ofrecen `/think low|medium|high|max`; `max` se asigna al valor nativo `think: "high"` porque la API nativa de Ollama acepta las cadenas de esfuerzo `low`, `medium` y `high`.
  - Los modelos GPT de OpenAI asignan `/think` mediante la compatibilidad con el esfuerzo específica de cada modelo de la API Responses. `/think off` solo envía `reasoning.effort: "none"` cuando el modelo de destino lo admite; de lo contrario, OpenClaw omite la carga útil de razonamiento desactivado en lugar de enviar un valor no compatible.
  - GPT-5.6 Sol y Terra ofrecen `/think ultra` nativo mediante el entorno de ejecución de Codex. GPT-5.6 Luna ofrece niveles hasta `max` porque su catálogo de Codex no anuncia Ultra.
  - El entorno de ejecución integrado de OpenClaw ofrece el valor lógico `/think ultra` para GPT-5.6 Sol, Terra y Luna. Envía el esfuerzo máximo del proveedor y añade instrucciones de orquestación proactiva de subagentes con ámbito de ejecución.
  - Las entradas de catálogo personalizadas compatibles con OpenAI pueden habilitar `/think xhigh` estableciendo `models.providers.<provider>.models[].compat.supportedReasoningEfforts` para que incluya `"xhigh"`. Esto utiliza los mismos metadatos de compatibilidad que asignan las cargas útiles salientes del esfuerzo de razonamiento de OpenAI, para que los menús, la validación de sesiones, la CLI del agente y `llm-task` concuerden con el comportamiento del transporte.
  - Las referencias configuradas obsoletas de OpenRouter Hunter Alpha omiten la inyección de razonamiento del proxy porque esa ruta retirada podía devolver el texto de la respuesta final mediante campos de razonamiento.
  - Google Gemini asigna `/think adaptive` al pensamiento dinámico propiedad del proveedor de Gemini. Las solicitudes de Gemini 3 omiten un valor `thinkingLevel` fijo, mientras que las solicitudes de Gemini 2.5 envían `thinkingBudget: -1`; los niveles fijos siguen asignándose al valor `thinkingLevel` o presupuesto de Gemini más cercano para esa familia de modelos.
  - MiniMax M2.x (`minimax/MiniMax-M2*`) en la ruta de transmisión compatible con Anthropic usa `thinking: { type: "disabled" }` de forma predeterminada, a menos que se establezca explícitamente el pensamiento en los parámetros del modelo o de la solicitud. Esto evita la filtración de deltas `reasoning_content` desde el formato de transmisión Anthropic no nativo de M2.x. MiniMax-M3 (y M3.x) está exento: M3 emite bloques de pensamiento Anthropic correctos y devuelve contenido vacío cuando el pensamiento está desactivado, por lo que OpenClaw mantiene M3 en la ruta de pensamiento omitido/adaptativo del proveedor.
  - Z.AI (`zai/*`) es binario (`on`/`off`) para la mayoría de los modelos GLM. GLM-5.2 es la excepción: ofrece `/think off|low|high|max`, asigna `low` y `high` a `reasoning_effort: "high"` de Z.AI y asigna `max` a `reasoning_effort: "max"`.
  - Kimi K3 de la API de Moonshot (`moonshot/kimi-k3`) siempre piensa con `max`, envía `reasoning_effort: "max"`, omite el campo `thinking` de K2 y las anulaciones de muestreo fijas, y conserva las opciones de herramientas compatibles con K3. Kimi Code K3 (`kimi/k3` y `kimi/k3[1m]`) ofrece `/think off|max`: el modo desactivado envía `thinking.type: "disabled"`, mientras que el máximo envía pensamiento adaptativo con esfuerzo máximo. Las referencias actuales de Kimi Code también incluyen `kimi/kimi-for-coding` y `kimi/kimi-for-coding-highspeed`. Kimi K2.7 Code (`moonshot/kimi-k2.7-code` y `moonshot/kimi-k2.7-code-highspeed`) siempre piensa, solo ofrece `on` y omite tanto `thinking` como `reasoning_effort` en la salida. Otros modelos `moonshot/*` asignan `/think off` a `thinking: { type: "disabled" }` y cualquier nivel distinto de `off` a `thinking: { type: "enabled" }`. Cuando el pensamiento de K2 está habilitado, Moonshot solo acepta `tool_choice` `auto|none`; OpenClaw normaliza los valores incompatibles a `auto`.

## Orden de resolución

1. Directiva insertada en el mensaje (se aplica solo a ese mensaje).
2. Anulación de sesión (se establece enviando un mensaje que solo contenga una directiva).
3. Valor predeterminado por agente (`agents.entries.*.thinkingDefault` en la configuración).
4. Valor predeterminado global (`agents.defaults.thinkingDefault` en la configuración).
5. Alternativa: el valor predeterminado declarado por el proveedor cuando esté disponible; de lo contrario, los modelos con capacidad de razonamiento se resuelven en `medium` o en el nivel compatible distinto de `off` más cercano para ese modelo, y los modelos sin capacidad de razonamiento permanecen en `off`.

## Establecer un valor predeterminado para la sesión

- Envíe un mensaje que contenga **únicamente** la directiva (se permiten espacios en blanco), por ejemplo, `/think:medium` o `/t high`.
- Esto se mantiene durante la sesión actual (de forma predeterminada, por remitente). Use `/think default` para borrar la anulación de sesión y heredar el valor predeterminado configurado o del proveedor; los alias incluyen `inherit`, `clear`, `reset` y `unpin`.
- `/think off` almacena una anulación explícita de desactivación. Deshabilita el pensamiento hasta que se cambie o se borre la anulación de sesión.
- Se envía una respuesta de confirmación (`Thinking level set to high.` / `Thinking disabled.`). Si el nivel no es válido (por ejemplo, `/thinking big`), el comando se rechaza con una indicación y el estado de la sesión permanece sin cambios.
- Envíe `/think` (o `/think:`) sin argumentos para ver el nivel de pensamiento actual.

## Aplicación por agente

- **OpenClaw integrado**: el nivel resuelto se pasa al entorno de ejecución del agente OpenClaw en proceso.
- **Backend de la CLI de Claude**: los niveles concretos distintos de desactivado se pasan a Claude Code como `--effort` cuando se usa `claude-cli`; `adaptive` elimina los indicadores de esfuerzo configurados y delega el esfuerzo efectivo al entorno, la configuración y los valores predeterminados del modelo de Claude Code. Consulte [backends de la CLI](/es/gateway/cli-backends).

## Modo rápido (/fast)

- Niveles: `auto|on|off|default`.
- Un mensaje que solo contiene la directiva alterna una anulación del modo rápido de la sesión y responde con `Fast mode set to auto.`, `Fast mode enabled.` o `Fast mode disabled.`. Use `/fast default` para borrar la anulación de sesión y heredar el valor predeterminado configurado; los alias incluyen `inherit`, `clear`, `reset` y `unpin`.
- Envíe `/fast` (o `/fast status`) sin indicar un modo para ver el estado efectivo actual del modo rápido.
- OpenClaw resuelve el modo rápido en este orden:
  1. Anulación insertada o mediante una directiva exclusiva `/fast auto|on|off` (`/fast default` borra esta capa)
  2. Anulación de sesión
  3. Valor predeterminado por agente (`agents.entries.*.fastModeDefault`)
  4. Configuración por modelo: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Alternativa: `off`
- `auto` mantiene el modo de sesión o configuración como automático, pero resuelve cada nueva llamada al modelo de forma independiente. Las llamadas que se inician antes del límite automático tienen habilitado el modo rápido; las llamadas posteriores de reintento, alternativa, resultado de herramienta o continuación se inician con el modo rápido deshabilitado. El límite predeterminado es de 60 segundos; establezca `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` en el modelo activo para cambiarlo.
- Para `openai/*`, el modo rápido se asigna al procesamiento prioritario de OpenAI mediante el envío de `service_tier=priority` en las solicitudes Responses compatibles.
- Para los modelos `openai/*` / `openai-codex/*` respaldados por Codex, el modo rápido envía el mismo indicador `service_tier=priority` en las respuestas de Codex. Los turnos nativos del servidor de aplicaciones de Codex reciben el nivel únicamente en `turn/start` o al iniciar o reanudar el hilo, por lo que `auto` no puede cambiar el nivel de un turno del servidor de aplicaciones que ya está en ejecución; se aplica al siguiente turno del modelo que inicia OpenClaw.
- Para las solicitudes públicas directas `anthropic/*`, incluido el tráfico autenticado mediante OAuth enviado a `api.anthropic.com`, el modo rápido se asigna a los niveles de servicio de Anthropic: `/fast on` establece `service_tier=auto` y `/fast off` establece `service_tier=standard_only`.
- Para `minimax/*` en la ruta compatible con Anthropic, `/fast on` (o `params.fastMode: true`) cambia `MiniMax-M2.7` por `MiniMax-M2.7-highspeed`.
- Los parámetros explícitos del modelo `serviceTier` / `service_tier` de Anthropic anulan el valor predeterminado del modo rápido cuando ambos están establecidos. OpenClaw sigue omitiendo la inyección del nivel de servicio de Anthropic para las URL base de proxies que no son de Anthropic.
- `/status` muestra `Fast` cuando el modo rápido está habilitado y `Fast:auto` cuando el modo configurado es automático.

## Directivas de información detallada (/verbose o /v)

- Niveles: `on` (mínimo) | `full` | `off` (predeterminado).
- Un mensaje que solo contiene la directiva cambia el modo detallado de la sesión y responde `Verbose logging enabled.` / `Verbose logging disabled.`; los niveles no válidos devuelven una sugerencia sin cambiar el estado.
- `/verbose off` almacena una anulación explícita para la sesión; se puede borrar desde la interfaz de usuario de sesiones seleccionando `inherit`.
- Los remitentes autorizados de canales externos pueden conservar la anulación del modo detallado de la sesión. Los clientes internos del gateway/chat web necesitan `operator.admin` para conservarla.
- La directiva insertada afecta solo a ese mensaje; en caso contrario, se aplican los valores predeterminados de la sesión/globales.
- Envíe `/verbose` (o `/verbose:`) sin argumentos para consultar el nivel de detalle actual.
- Cuando el modo detallado está activado, los agentes que emiten resultados estructurados de herramientas devuelven cada llamada a herramienta como un mensaje independiente que contiene solo metadatos, con el prefijo `<emoji> <tool-name>: <arg>` cuando está disponible. Estos resúmenes de herramientas se envían en cuanto se inicia cada herramienta (en burbujas separadas), no como deltas transmitidos.
- Los resúmenes de errores de herramientas permanecen visibles en el modo normal, pero los sufijos con detalles del error sin procesar se ocultan a menos que el modo detallado sea `full`.
- Cuando el modo detallado es `full`, las salidas de las herramientas también se reenvían tras completarse (en una burbuja separada y truncadas a una longitud segura). Si se cambia `/verbose on|full|off` mientras hay una ejecución en curso, las burbujas de herramientas posteriores respetan la nueva configuración.
- `agents.defaults.toolProgressDetail` controla el formato de los resúmenes de herramientas `/verbose` y de las líneas de herramientas en borradores de progreso. Use `"explain"` (predeterminado) para etiquetas humanas compactas como `🛠️ Exec: checking JS syntax`; use `"raw"` cuando también se quiera añadir el comando o detalle sin procesar para la depuración. El valor `agents.entries.*.toolProgressDetail` de cada agente anula el predeterminado.
  - `explain`: `🛠️ Exec: check JS syntax for /tmp/app.js`
  - `raw`: `🛠️ Exec: check JS syntax for /tmp/app.js, node --check /tmp/app.js`

## Directivas de seguimiento de Plugin (/trace)

- Niveles: `on` | `off` (predeterminado).
- Un mensaje que solo contiene la directiva cambia la salida de seguimiento de Plugin de la sesión y responde `Plugin trace enabled.` / `Plugin trace disabled.`.
- La directiva insertada afecta solo a ese mensaje; en caso contrario, se aplican los valores predeterminados de la sesión/globales.
- Envíe `/trace` (o `/trace:`) sin argumentos para consultar el nivel de seguimiento actual.
- `/trace` tiene un alcance más limitado que `/verbose`: solo expone líneas de seguimiento/depuración propias de los plugins, como los resúmenes de depuración de Active Memory.
- Las líneas de seguimiento pueden aparecer en `/status` y como un mensaje de diagnóstico posterior a la respuesta normal del asistente.

## Visibilidad del razonamiento (/reasoning)

- Niveles: `on|off|stream`.
- Un mensaje que solo contiene la directiva cambia si los bloques de pensamiento se muestran en las respuestas.
- Cuando está activado, el razonamiento se envía como un **mensaje independiente** con el prefijo `Thinking`.
- `stream`: transmite el razonamiento mientras se genera la respuesta cuando el canal activo admite vistas previas del razonamiento y, después, envía la respuesta final sin el razonamiento.
- Alias: `/reason`.
- Envíe `/reasoning` (o `/reasoning:`) sin argumentos para consultar el nivel de razonamiento actual.
- Orden de resolución: directiva insertada, luego anulación de la sesión, valor predeterminado de cada agente (`agents.entries.*.reasoningDefault`), valor predeterminado global (`agents.defaults.reasoningDefault`) y, por último, valor alternativo (`off`).

Las etiquetas de razonamiento mal formadas de los modelos locales se gestionan de forma conservadora. Los bloques `<think>...</think>` cerrados permanecen ocultos en las respuestas normales, y el razonamiento sin cerrar que aparece después de texto ya visible también se oculta. Si una respuesta está completamente envuelta en una única etiqueta de apertura sin cerrar y, de otro modo, se entregaría como texto vacío, OpenClaw elimina la etiqueta de apertura mal formada y entrega el texto restante.

## Contenido relacionado

- La documentación del modo elevado se encuentra en [Modo elevado](/es/tools/elevated).

## Heartbeats

- El cuerpo de la comprobación de Heartbeat es el prompt de Heartbeat configurado (valor predeterminado: `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Las directivas insertadas en un mensaje de Heartbeat se aplican de la forma habitual (pero se debe evitar cambiar los valores predeterminados de la sesión desde los Heartbeats).
- De forma predeterminada, la entrega de Heartbeat incluye solo la carga útil final. Para enviar también el mensaje independiente `Thinking` (cuando esté disponible), establezca `agents.defaults.heartbeat.includeReasoning: true` o el valor `agents.entries.*.heartbeat.includeReasoning: true` de cada agente.

## Interfaz de usuario del chat web

- El selector de pensamiento del chat web refleja el nivel almacenado de la sesión procedente del almacén/configuración de la sesión entrante cuando se carga la página.
- Al seleccionar otro nivel, la anulación de la sesión se escribe inmediatamente mediante `sessions.patch`; no se espera al siguiente envío y no se trata de una anulación `thinkingOnce` de un solo uso.
- Si se envía mientras todavía se están aplicando cambios en los selectores de modelo, razonamiento o velocidad, se espera a que finalicen todos los parches pendientes de los selectores; si un cambio falla, el mensaje permanece sin enviar para su revisión.
- La primera opción siempre permite borrar la anulación. Muestra `Inherited: <resolved level>`, incluido `Inherited: Off` cuando el pensamiento heredado está desactivado.
- Las opciones explícitas del selector usan directamente sus etiquetas de nivel y conservan las etiquetas del proveedor cuando están presentes (por ejemplo, `Maximum` para una opción `max` etiquetada por el proveedor).
- El selector usa `thinkingLevels`, devuelto por la fila/los valores predeterminados de sesión del Gateway, mientras que `thinkingOptions` se conserva como una lista de etiquetas heredada. La interfaz de usuario del navegador no mantiene su propia lista de expresiones regulares de proveedores; los plugins son responsables de los conjuntos de niveles específicos de cada modelo.
- `/think:<level>` sigue funcionando y actualiza el mismo nivel almacenado de la sesión, por lo que las directivas del chat y el selector permanecen sincronizados.

## Perfiles de proveedores

- Los plugins de proveedores pueden exponer `resolveThinkingProfile(ctx)` para definir los niveles admitidos y el valor predeterminado del modelo.
- Los plugins de proveedores que actúen como proxy de modelos Claude deben reutilizar `resolveClaudeThinkingProfile(modelId)` de `openclaw/plugin-sdk/provider-model-shared` para mantener alineados los catálogos directos de Anthropic y los de proxy.
- Cada nivel de perfil tiene un `id` canónico almacenado (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` o `ultra`) y puede incluir un `label` de visualización. Los proveedores binarios usan `{ id: "low", label: "on" }`.
- Los hooks de perfil reciben los datos combinados del catálogo cuando están disponibles, incluidos `reasoning`, `compat.thinkingFormat` y `compat.supportedReasoningEfforts`. Use esos datos para exponer perfiles binarios o personalizados solo cuando el contrato de solicitud configurado admita la carga útil correspondiente.
- Los plugins de herramientas que necesiten validar una anulación explícita del pensamiento deben usar `api.runtime.agent.resolveThinkingPolicy({ provider, model, agentRuntime })` junto con `api.runtime.agent.normalizeThinkingLevel(...)`; no deben mantener sus propias listas de niveles de proveedores/modelos. Pase `agentRuntime` cuando la herramienta sea responsable de la ruta de ejecución, como en una ejecución siempre integrada.
- Los plugins de herramientas con acceso a metadatos configurados de modelos personalizados pueden pasar `catalog` a `resolveThinkingPolicy` para que las adhesiones voluntarias a `compat.supportedReasoningEfforts` se reflejen en la validación del lado del Plugin.
- Los hooks heredados publicados (`supportsXHighThinking`, `isBinaryThinking` y `resolveDefaultThinkingLevel`) permanecen como adaptadores de compatibilidad, pero los nuevos conjuntos de niveles personalizados deben usar `resolveThinkingProfile`.
- Las filas/los valores predeterminados del Gateway exponen `thinkingLevels`, `thinkingOptions` y `thinkingDefault` para que los clientes ACP/chat representen los mismos identificadores y etiquetas de perfil que utiliza la validación en tiempo de ejecución.
