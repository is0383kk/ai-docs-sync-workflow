---
read_when:
    - Está conectando las interfaces de uso y cuota del proveedor
    - Es necesario explicar el comportamiento del seguimiento del uso o los requisitos de autenticación.
summary: Superficies de seguimiento del uso y requisitos de credenciales
title: Seguimiento del uso
x-i18n:
    generated_at: "2026-07-26T05:11:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a1bc9aeb95cd80a48ab57a18fcd24894fdd6fb71e10e8bea8bae67a8688b78e
    source_path: concepts/usage-tracking.md
    workflow: 16
---

## Qué es

- Obtiene el uso y la cuota del proveedor directamente desde el endpoint de uso de cada proveedor. No se estima la facturación del proveedor; solo se incluyen los nombres de planes, períodos de cuota, saldos, gastos, presupuestos, historial de costes diarios, atribución de tokens/modelos o resúmenes del estado de la cuenta comunicados por el proveedor.
- La salida legible de los períodos de cuota se normaliza a `X% left`, incluso cuando un proveedor informa de la cuota consumida, la cuota restante o solo recuentos sin procesar. Los proveedores sin períodos de cuota restablecibles muestran en su lugar el texto de resumen del proveedor (por ejemplo, un saldo).
- El `/status` de nivel de sesión y la herramienta `session_status` recurren al registro de transcripción de la sesión cuando la instantánea de la sesión activa no contiene datos de tokens/modelos. Ese mecanismo alternativo completa los contadores de tokens/caché que falten, puede recuperar la etiqueta del modelo de ejecución activo y prefiere el total más alto orientado al prompt cuando faltan los metadatos de la sesión o son inferiores (`totalTokensFresh !== true`, cero o por debajo del valor derivado de la transcripción). Los valores activos distintos de cero siempre prevalecen sobre el mecanismo alternativo.

## Dónde aparece

- `/status` en los chats: tarjeta de estado con los tokens de la sesión y el coste estimado (solo modelos con clave de API). Cuando está disponible, se muestra el uso del **proveedor del modelo actual**, como un período `X% left` normalizado o como texto de resumen del proveedor.
- `/usage off|tokens|full` en los chats: pie de uso por respuesta.
- `/usage cost` en los chats: resumen de costes locales agregado a partir de los registros de sesión de OpenClaw.
- CLI: `openclaw status --usage` imprime un desglose completo del uso y la cuota por proveedor.
- CLI: `openclaw models status` enumera los perfiles de autenticación OAuth/token y muestra un resumen de los períodos de uso junto a cada proveedor que disponga de uno.
- Interfaz de control: **Uso** muestra tarjetas del plan y de facturación del proveedor sobre el análisis de tokens y costes estimados derivado de las sesiones de OpenClaw. Las credenciales de la API de administración de Anthropic y OpenAI añaden el gasto comunicado por el proveedor de hoy, de 7 días y de 30 días, tendencias diarias, totales de tokens, modelos principales y categorías de costes.
- Interfaz de control: la ventana emergente del anillo de contexto del redactor de chats muestra el **uso del plan** para los proveedores de suscripción: barras por período (5 horas, semanal y específico del modelo) con horas de restablecimiento, el plan del proveedor cuando se conoce (por ejemplo, `Max (20x)`) y créditos de uso adicional. Las sesiones facturadas mediante un plan ocultan las estimaciones monetarias por token; las sesiones facturadas mediante API conservan `Est. cost` y el desglose de costes por tipo. Las configuraciones de la CLI de Claude Code (`claude-cli`) reutilizan el mismo uso de la suscripción de Anthropic.
- Barra de menús de macOS: aparece una sección raíz «Uso» debajo de Contexto cuando hay disponibles instantáneas de uso del proveedor. Consulte [Barra de menús](/es/platforms/mac/menu-bar).

`openclaw channels list` ya no imprime el uso del proveedor; en su lugar, dirige a los usuarios a `openclaw status` o `openclaw models list`.

## Historial de costes de Anthropic y OpenAI

La cuota de suscripción y la facturación de la API son superficies distintas del proveedor:

- Las credenciales de suscripción/configuración de Anthropic siguen mostrando los períodos de cuota de Claude y los presupuestos opcionales de uso adicional. Configure `ANTHROPIC_ADMIN_KEY` o `ANTHROPIC_ADMIN_API_KEY` para mostrar en su lugar el historial de las API de uso y costes de la organización. Las credenciales de proveedor de Anthropic que comiencen por `sk-ant-admin` se detectan automáticamente.
- OAuth de OpenAI ChatGPT/Codex sigue mostrando el plan, los períodos de cuota y el saldo de crédito. Configure `OPENAI_ADMIN_KEY` para mostrar en su lugar el historial de costes y uso de finalizaciones de la organización; opcionalmente, configure `OPENAI_PROJECT_ID` para limitarlo a un proyecto. OpenClaw nunca envía credenciales de inferencia de `OPENAI_API_KEY`, de la configuración del proveedor ni de los perfiles de autenticación a las API de la organización, ya que esas claves pueden pertenecer a endpoints personalizados.

Las credenciales de administración tienen prioridad porque proporcionan la facturación real de la organización. OpenClaw no combina estos totales comunicados por el proveedor con sus estimaciones de sesión locales; las dos secciones responden deliberadamente a preguntas diferentes.

## Modo predeterminado del pie de uso

`/usage off|tokens|full` establece el pie de una sesión y se recuerda durante esa
sesión. `messages.responseUsage` inicializa ese modo para las sesiones que no han
elegido uno, de modo que el pie pueda estar activado de forma predeterminada sin escribir `/usage` cada vez.

Configure un modo para todos los canales o un mapa por canal con un valor alternativo `default`:

```jsonc
{
  "messages": {
    "responseUsage": "tokens",
    // o bien: { "default": "off", "discord": "full" }
  },
}
```

Valores aceptados: `"off"`, `"tokens"`, `"full"` y el alias heredado `"on"` (tratado como `"tokens"`).

### Tres estados de sesión distintos

El campo `responseUsage` de una sesión tiene tres estados representables, cada uno con
una semántica diferente:

| Estado                         | Valor almacenado                                | Modo efectivo                                                                 |
| ------------------------------ | ----------------------------------------------- | ----------------------------------------------------------------------------- |
| **Sin definir / heredado**     | `undefined` (ausente)                    | Recurre al valor predeterminado de configuración `messages.responseUsage` y después a `off`. |
| **Desactivado explícitamente** | `"off"` (almacenado)                 | Siempre desactivado; un valor predeterminado de configuración distinto de desactivado no puede volver a activar el pie. |
| **Activado explícitamente**    | `"tokens"` o `"full"` (almacenado) | Ese modo, independientemente del valor predeterminado de configuración.       |

### Precedencia

Modo efectivo = anulación de la sesión → entrada de configuración del canal → `default` → `off`.

Un valor `/usage off` explícito se **conserva** como el valor literal `"off"` en la
sesión; no equivale a «sin definir». Un valor predeterminado `messages.responseUsage`
distinto de desactivado no puede volver a activar el pie después de que el usuario lo haya desactivado explícitamente.

### Restablecer frente a desactivar

- `/usage off` fuerza la desactivación del pie y conserva esa elección. Un valor predeterminado
  configurado distinto de desactivado no puede anularla.
- `/usage reset` (alias: `default`, `inherit`, `inherited`, `clear`, `unpin`) borra la anulación de la sesión.
  La sesión pasa entonces a **heredar** el valor predeterminado efectivo de la configuración
  (`messages.responseUsage`). Si no hay ningún valor predeterminado configurado, el pie permanece desactivado.
- Un restablecimiento completo de la sesión (`/reset` o `/new`) o una rotación de sesión **conserva**
  la preferencia explícita del modo de uso, de modo que la opción de visualización del usuario sobreviva
  a las rotaciones de sesión. Solo `/usage reset` (y sus alias) borra la anulación.

### Comportamiento de alternancia

`/usage` sin argumentos recorre: desactivado → tokens → completo → desactivado. El punto de partida
del ciclo es el modo actual **efectivo** (la anulación de la sesión recurre
al valor predeterminado de configuración cuando no está definida), por lo que el ciclo siempre coincide con lo que
el usuario ve actualmente en el pie.

### Configuración

Sin configuración, se conserva el comportamiento anterior (pie desactivado hasta `/usage`). Utilice
`/usage reset` para borrar una anulación de sesión y volver a heredar el valor predeterminado configurado.

## Pie `/usage full` personalizado

`/usage tokens` siempre representa una línea `Usage: X in / Y out` sencilla (además de los sufijos de caché y
coste estimado cuando están disponibles). Solo `/usage full` representa el pie más completo
descrito a continuación.

`/usage full` muestra un pie compacto integrado con el modelo, el razonamiento, el modo rápido/lento,
la ventana de contexto y el coste cuando esos campos están disponibles. No se requiere ningún archivo de plantilla
para el pie integrado.

`messages.usageTemplate` está destinado únicamente a diseños personalizados avanzados. El valor es una
ruta de archivo JSON (admite `~`) o un objeto en línea, y sustituye al pie integrado
cuando es válido. La ruta de archivo se supervisa y se vuelve a cargar en directo cuando cambia.

```json
{
  "messages": {
    "usageTemplate": "~/.openclaw/usage-footer.json"
  }
}
```

Las plantillas ausentes o vacías recurren silenciosamente al pie integrado. Las plantillas
configuradas que no puedan leerse o no sean válidas (JSON incorrecto o una estructura sin
elementos de salida representables) también recurren al pie integrado y emiten una advertencia para el operador.

Parta de la estructura integrada para crear plantillas personalizadas y, a continuación, edite las partes que desee
cambiar:

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": {
    "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿",
    "block": "░▏▎▍▌▋▊▉█",
    "shade": "░▒▓█",
    "moon": "🌑🌘🌗🌖🌕",
    "level": "▁▂▃▄▅▆▇█",
    "weather": ["🥶", "☁️", "🌥", "⛅️", "🌤", "☀️"],
    "plants": ["🪾", "🍂", "🌱", "☘️", "🍀", "🌿"],
    "moons6": ["🌑", "🌚", "🌘", "🌗", "🌖", "🌝"],
  },
  "aliases": {
    "models": {
      "claude-opus-4-6": "opus46",
      "claude-opus-4-8": "opus48",
      "claude-sonnet-4-6": "sonnet46",
      "claude-haiku-4-5": "haiku45",
      "gpt-5.5": "gpt5.5",
    },
    "reasoning": {
      "off": "🌑",
      "minimal": "🌚",
      "low": "🌘",
      "medium": "🌗",
      "high": "🌕",
      "xhigh": "🌝",
    },
  },
  "output": {
    "sep": "",
    "default": [
      { "text": "{model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
      { "map": "model.is_fallback", "cases": { "true": "🔄" } },
      { "map": "model.is_override", "cases": { "true": "📌" } },
      { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
      { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
      {
        "when": "context.max_tokens",
        "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
      },
      { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
    ],
    "surfaces": {
      "discord": [
        { "text": "-# -\n" },
        { "text": "-# {model.provider}{identity.emoji|🤖}{model.display_name|alias:models}" },
        { "map": "model.is_fallback", "cases": { "true": "🔄" } },
        { "map": "model.is_override", "cases": { "true": "📌" } },
        { "when": "model.reasoning", "text": "{model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": "⚡️", "false": "🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚[{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
        { "when": "cost.turn_usd", "text": " 💰{cost.turn_usd|fixed:4}" },
      ],
    },
  },
}
```

### Estructura

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "<name>": "glifos de menor a mayor" }, // cadena (1 glifo/carácter) o matriz
  "aliases": { "<table>": { "<value>": "<label>" } },
  "output": {
    "sep": "", // une los elementos restantes
    "default": [/* pieces */], // alternativa para cualquier superficie
    "surfaces": {
      "discord": [/* pieces */],
      "telegram": [/* pieces */],
    },
  },
}
```

Cada superficie es una lista ordenada de **elementos**; el motor representa cada uno, descarta
los vacíos y une los restantes con `sep`. Una superficie sin entrada utiliza
`output.default`.

### Rutas del contrato

Un elemento lee valores del contrato de cada turno mediante una ruta de puntos. Los valores ausentes están
vacíos (por lo que una protección `when` o un `|fallback` mantiene limpio el elemento).

| Ruta                                                                                | Significado                                                                                              |
| ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `surface`                                                                           | id del canal (`discord`/`telegram`/etc.)                                                               |
| `agentId` / `chat_type`                                                             | id del agente propietario / tipo de superficie de chat                                                                  |
| `model.id` / `model.display_name` / `model.provider`                                | id del modelo / nombre para mostrar / id del proveedor                                                                |
| `model.actual`, `model.resolved_ref`                                                | referencia de proveedor/modelo utilizada realmente para el turno                                                        |
| `model.requested`                                                                   | referencia de proveedor/modelo solicitada (antes de la alternativa)                                                       |
| `model.reasoning`                                                                   | esfuerzo (`off` a `xhigh`)                                                                       |
| `model.is_fallback` / `model.is_override`                                           | booleano: se usó una alternativa / modelo fijado                                                                   |
| `model.override_source` / `model.auth_mode`                                         | etiqueta de origen de la anulación / modo de credenciales (`oauth`, `api-key`, `token`, `mixed`, `aws-sdk`, `unknown`) |
| `state.fast_mode`                                                                   | booleano: rápido frente a lento                                                                                   |
| `state.compactions`                                                                 | recuento de Compaction de la sesión                                                                     |
| `context.max_tokens` / `context.used_tokens` / `context.pct_used`                   | presupuesto de ventana / tokens ocupados / porcentaje usado de 0 a 100                                                         |
| `usage.input_tokens` / `usage.output_tokens` / `usage.total_tokens`                 | agregado del turno                                                                                       |
| `usage.cache_read_tokens` / `usage.cache_write_tokens`                              | tokens de lectura y escritura de caché del turno                                                       |
| `usage.has_tokens` / `usage.has_split_tokens` / `usage.has_total_only_tokens`       | protecciones de visualización de tokens                                                                                 |
| `usage.cache_hit_pct`                                                               | proporción de lecturas de caché respecto al total de tokens del prompt                                                              |
| `usage.last.input_tokens` / `usage.last.output_tokens` / `usage.last.cache_hit_pct` | solo la llamada final al modelo (también contiene `cache_read_tokens`, `cache_write_tokens`, `total_tokens`)           |
| `cost.turn_usd` / `cost.available`                                                  | coste estimado del turno / si se resolvió una tabla de costes                                                  |
| `timing.duration_ms`                                                                | duración del turno según el tiempo transcurrido real                                                                             |
| `identity.name` / `identity.emoji` / `identity.avatar`                              | nombre de identidad del agente / emoji / avatar                                                                 |
| `session.id`                                                                        | id de la sesión                                                                                           |

(Las ventanas de límite de frecuencia del proveedor **no** forman parte de este contrato; actualmente no hay ninguna ruta con valor de matriz, por lo que una pieza `each` no tiene nada sobre lo que iterar).

### Verbos

Pase un valor por los verbos de izquierda a derecha; un segmento que no sea un verbo es la alternativa.

| Verbo            | Efecto                                | Ejemplo                           |
| --------------- | ------------------------------------- | --------------------------------- |
| `num`           | recuento compacto                         | `272000 -> 272k`                  |
| `fixed:N`       | N decimales (`0..100`, 2 de forma predeterminada)      | `0.0377`                          |
| `dur`           | segundos a duración                   | `14820 -> 4h07m`                  |
| `pct`           | añadir `%`                            | `96 -> 96%`                       |
| `inv`           | `100 - x`                             | para convertir usado en restante             |
| `alias:TABLE`   | buscar en `aliases`, repetir si no aparece | `medium -> 🌗`                    |
| `meter:W:SCALE` | barra de glifos de W celdas sobre un valor de 0 a 100   | `[⣿⣿⠐⠐⠐]` (`meter:1` = un glifo) |

`fixed:N` solo acepta un entero decimal completo de 0 a 100. Los argumentos de
precisión no válidos hacen que esa interpolación quede vacía.

`meter:W:SCALE` solo acepta un ancho entero decimal completo de 1 a 100. Deje el ancho en blanco para usar el valor predeterminado 5 (`meter::braille`); los
anchos no válidos hacen que esa interpolación quede vacía.

### Formas de las piezas

- `{ "text": "📚 {context.max_tokens|num}" }`: literal + interpolación.
- `{ "when": "<path>", "text": "..." }`: renderizar solo si la ruta es verdadera.
- `{ "map": "<path>", "cases": { "true": "⚡", "false": "🐌" } }`: valor a glifo (un caso `_default` abarca los valores sin coincidencia).
- `{ "each": "<array-path>", "item": "{label}" }`: iterar una ruta con valor de matriz (ninguna ruta del contrato actual es una matriz).

### Ejemplo

```jsonc
{
  "schema": "openclaw.usageBar.v1",
  "scales": { "braille": "⠐⡀⡄⡆⡇⣇⣧⣷⣿" },
  "aliases": { "reasoning": { "medium": "🌗", "high": "🌕" } },
  "output": {
    "surfaces": {
      "discord": [
        { "text": "{model.display_name}" },
        { "when": "model.reasoning", "text": " {model.reasoning|alias:reasoning}" },
        { "map": "state.fast_mode", "cases": { "true": " ⚡", "false": " 🐌" } },
        {
          "when": "context.max_tokens",
          "text": " | 📚 [{context.pct_used|meter:5:braille}]{context.max_tokens|num}",
        },
      ],
    },
  },
}
```

se renderiza, por ejemplo, como `claude-sonnet-4-6 🌗 🐌 | 📚 [⣿⣿⣿⣿⣧]272k`.

## Proveedores y credenciales

El uso se oculta cuando no se puede resolver ninguna autenticación de uso del proveedor que sea válida. OpenClaw
descubre automáticamente los plugins de proveedor habilitados que declaran
`contracts.usageProviders` e implementan tanto `resolveUsageAuth` como
`fetchUsageSnapshot`; no hay una lista de proveedores permitidos separada en el núcleo. El contrato
estático mantiene acotado el descubrimiento sin importar todos los plugins de proveedor. Cada
plugin es responsable de su endpoint ascendente y de la asignación de respuestas. La instantánea
compartida mantiene los nombres de planes, las ventanas de cuota, los saldos, el gasto y los presupuestos
independientes del proveedor para los consumidores de la CLI, la aplicación y la interfaz de control.

- **Anthropic (Claude)**: tokens OAuth en los perfiles de autenticación. Si el token OAuth carece del
  ámbito `user:profile`, recurre a una sesión web `claude.ai` (`CLAUDE_AI_SESSION_KEY`,
  `CLAUDE_WEB_SESSION_KEY` o una cookie `sessionKey=` en `CLAUDE_WEB_COOKIE`) cuando está configurada.
  Se incluyen los límites específicos del modelo y los gastos/presupuestos mensuales habilitados de uso adicional
  cuando Anthropic los notifica. En su lugar, una clave explícita de la API de administración de Anthropic, o un
  perfil de proveedor `sk-ant-admin...` detectado automáticamente, muestra el coste de la organización de los últimos 30 días
  y el historial de la API de mensajes.
- **ClawRouter**: clave de API (`CLAWROUTER_API_KEY`). Muestra una ventana de presupuesto mensual
  y un presupuesto tipado en USD cuando está configurado; de lo contrario, muestra el gasto agregado y un
  resumen de solicitudes/tokens/costes.
- **DeepSeek**: clave de API mediante entorno/configuración/almacén de autenticación (`DEEPSEEK_API_KEY`).
  Muestra cada saldo de divisa notificado por el proveedor.
- **GitHub Copilot**: tokens OAuth en los perfiles de autenticación.
- **Gemini CLI**: tokens OAuth en los perfiles de autenticación.
- **MiniMax**: clave de API o perfil de autenticación OAuth de MiniMax. OpenClaw considera
  `minimax`, `minimax-cn` y `minimax-portal` como la misma superficie de cuota de MiniMax,
  prefiere el OAuth de MiniMax almacenado cuando está disponible y, de lo contrario, recurre
  a `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY` o `MINIMAX_API_KEY`.
  El sondeo de uso obtiene el host de Coding Plan de `models.providers.minimax-portal.baseUrl`
  o `models.providers.minimax.baseUrl` cuando están configurados y, de lo contrario, usa el
  host de MiniMax para China.
  Los campos sin procesar `usage_percent` / `usagePercent` de MiniMax representan la cuota
  **restante**, por lo que OpenClaw los invierte antes de mostrarlos; los campos basados en recuentos tienen prioridad
  cuando están presentes.
  - Las etiquetas de ventana provienen de los campos de horas/minutos del proveedor cuando están presentes y, después,
    recurren al intervalo `start_time` / `end_time`.
  - Si el endpoint del plan de codificación devuelve `model_remains`, OpenClaw prefiere la
    entrada del modelo de chat, obtiene la etiqueta de la ventana de las marcas de tiempo cuando no existen campos
    explícitos `window_hours` / `window_minutes` e incluye el nombre del modelo
    en la etiqueta del plan.
- **OpenAI (plan Codex/ChatGPT)**: tokens OAuth en los perfiles de autenticación (se envía el encabezado `ChatGPT-Account-Id`
  cuando hay un id de cuenta). Muestra el plan de ChatGPT, las ventanas
  restablecibles de Codex y un saldo de créditos cuando se notifican. Los créditos siguen siendo créditos
  del proveedor; OpenClaw no los etiqueta como dólares. `OPENAI_ADMIN_KEY` añade
  el coste de la organización de los últimos 30 días y el historial de uso de completados cuando la clave tiene acceso
  al panel de uso. Las credenciales de inferencia nunca se reenvían a las API de la organización.
- **OpenRouter**: clave de API o clave de API respaldada por OAuth (`OPENROUTER_API_KEY` o un perfil
  de autenticación). Combina el endpoint de créditos de la cuenta con el endpoint de cuota de la clave,
  de modo que aparecen el saldo/gasto de la cuenta, el presupuesto de la clave y el uso diario/semanal/mensual
  cuando la credencial puede acceder a ellos. Cualquiera de los endpoints puede enriquecer la instantánea
  de forma independiente.
- **Venice**: clave de API mediante entorno/configuración/almacén de autenticación (`VENICE_API_KEY`). Muestra los saldos
  en USD y DIEM, además del uso de la asignación por época de DIEM cuando se notifica.
- **Xiaomi MiMo**: dos superficies de uso independientes. El pago por uso utiliza una clave de API
  (`XIAOMI_API_KEY`); Token Plan utiliza una clave independiente (`XIAOMI_TOKEN_PLAN_API_KEY`).
  Actualmente, ninguna de las dos informa de ventanas de cuota.
- **z.ai**: clave de API mediante entorno/configuración/almacén de autenticación (`ZAI_API_KEY` o `Z_AI_API_KEY`).

## Relacionado

- [Uso y costes de tokens](/es/reference/token-use)
- [Uso y costes de la API](/es/reference/api-usage-costs)
- [Almacenamiento en caché de prompts](/es/reference/prompt-caching)
- [Barra de menús](/es/platforms/mac/menu-bar)
