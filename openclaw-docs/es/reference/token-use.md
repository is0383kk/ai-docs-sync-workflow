---
read_when:
    - Explicación del uso de tokens, los costes o las ventanas de contexto
    - Depuración del crecimiento del contexto o del comportamiento de Compaction
summary: Cómo crea OpenClaw el contexto del prompt e informa del uso de tokens y los costes
title: Uso de tokens y costes
x-i18n:
    generated_at: "2026-07-26T04:53:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw contabiliza **tokens**, no caracteres. Los tokens son específicos de cada modelo, pero la mayoría de los modelos
al estilo de OpenAI promedian ~4 caracteres por token en textos en inglés.

## Cómo se crea el prompt del sistema

OpenClaw ensambla su propio prompt del sistema en cada ejecución. Incluye:

- Lista de herramientas + descripciones breves
- Lista de Skills (solo metadatos; las instrucciones se cargan bajo demanda con `read`). Los turnos
  nativos de Codex reciben el bloque compacto de Skills como instrucciones de
  desarrollador para la colaboración limitadas al turno; otros entornos de ejecución lo reciben en la superficie normal del prompt.
  Limitado por `skills.limits.maxSkillsPromptChars`, con una anulación opcional por agente
  en `agents.entries.*.skillsLimits.maxSkillsPromptChars`.
- Instrucciones de actualización automática
- Espacio de trabajo + archivos de arranque (`AGENTS.md`, `SOUL.md`, `TOOLS.md`,
  `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` cuando es nuevo, además de
  `MEMORY.md` cuando está presente). Los archivos grandes inyectados se truncan mediante
  `agents.defaults.bootstrapMaxChars` (valor predeterminado: `20000`); la inyección
  total de arranque está limitada por `agents.defaults.bootstrapTotalMaxChars` (valor predeterminado:
  `60000`).
  - Los turnos nativos de Codex no pegan `MEMORY.md` sin procesar cuando las herramientas de memoria están
    disponibles para ese espacio de trabajo; en su lugar, reciben un pequeño puntero de memoria en
    las instrucciones de desarrollador para la colaboración limitadas al turno y usan las herramientas de memoria
    bajo demanda. Si las herramientas están deshabilitadas, la búsqueda en memoria no está disponible o
    el espacio de trabajo activo difiere del espacio de trabajo de memoria del agente, `MEMORY.md`
    recurre a la ruta normal de contexto de turno limitado.
  - El archivo raíz `memory.md` en minúsculas nunca se inyecta. Es una entrada de reparación heredada
    para `openclaw doctor --fix`, que la migra a `MEMORY.md`.
  - Los archivos diarios `memory/*.md` no forman parte del prompt de arranque normal;
    permanecen disponibles bajo demanda mediante las herramientas de memoria en los turnos ordinarios. Las ejecuciones
    del modelo durante el restablecimiento o el inicio pueden anteponer un bloque de contexto de inicio de un solo uso con memoria
    diaria reciente para ese primer turno, controlado por
    `agents.defaults.startupContext`. Los mensajes de chat simples `/new` y `/reset` se
    confirman sin invocar el modelo.
  - Los extractos posteriores a la Compaction de `AGENTS.md` requieren la
    activación explícita mediante `agents.defaults.compaction.postCompactionSections`; los plugins pueden añadir
    otro contexto mediante `before_prompt_build`.
- Hora (UTC + zona horaria del usuario)
- Etiquetas de respuesta + comportamiento de Heartbeat
- Metadatos del entorno de ejecución (host/SO/modelo/razonamiento)

Consulte el desglose completo en [Prompt del sistema](/es/concepts/system-prompt).

Al documentar credenciales o fragmentos de autenticación, utilice las
[Convenciones de marcadores de posición para secretos](/es/reference/secret-placeholder-conventions) para
evitar falsos positivos del detector de secretos en cambios exclusivos de la documentación.

## Qué se contabiliza en la ventana de contexto

Todo lo que recibe el modelo se contabiliza para el límite de contexto:

- Prompt del sistema (todas las secciones anteriores)
- Historial de la conversación (mensajes del usuario + del asistente)
- Llamadas a herramientas y resultados de herramientas
- Archivos adjuntos/transcripciones (imágenes, audio, archivos)
- Resúmenes de Compaction y artefactos de poda
- Envoltorios del proveedor o encabezados de seguridad (no visibles, pero igualmente contabilizados)

Las superficies con uso intensivo del entorno de ejecución tienen sus propios límites explícitos en
`agents.defaults.contextLimits` (anulaciones por agente en
`agents.entries.*.contextLimits`):

| Clave                    | Propósito                                                                |
| ------------------------ | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | Máximo de caracteres que devuelve `memory_get` antes del truncamiento. |
| `postCompactionMaxChars` | Máximo de caracteres conservados de `AGENTS.md` durante la actualización posterior a la Compaction. |

Estos son extractos limitados del entorno de ejecución y bloques inyectados que son propiedad de este,
independientes de los límites de arranque, los límites del contexto de inicio y los límites
del prompt de Skills.

OpenClaw deriva el límite activo de los resultados de herramientas a partir de la ventana de contexto
efectiva del modelo: `16000` caracteres por debajo de
100K tokens, `32000` caracteres con 100K+ tokens y `64000` caracteres con 200K+ tokens.
La protección de proporción del contexto del entorno de ejecución también limita un único resultado de herramienta al 30% de la
ventana de contexto.

Las ventanas grandes de los proveedores no se habilitan automáticamente cuando alteran considerablemente
el coste o la latencia. Por ejemplo, los modelos directos OpenAI GPT-5.5 y GPT-5.6
publican una ventana total de `1050000` tokens, pero OpenClaw establece de forma predeterminada su presupuesto activo
del entorno de ejecución en `272000` tokens. El presupuesto de entrada opcional `922000` reserva la
asignación completa de salida de `128000`, y OpenAI aplica precios superiores por contexto largo
a toda la solicitud una vez que la entrada supera los `272000` tokens. Consulte
[Valores predeterminados de la ventana de contexto de OpenAI](/es/providers/openai#context-window-defaults-and-long-context-opt-in).

Para las imágenes, OpenClaw reduce la resolución de las cargas de imágenes de transcripciones y herramientas antes de
las llamadas al proveedor. Ajústelo con `agents.defaults.imageMaxDimensionPx` (valor predeterminado:
`1200`):

- Los valores más bajos reducen el uso de tokens de visión y el tamaño de la carga.
- Los valores más altos conservan más detalle visual en capturas de pantalla con abundante OCR o elementos de interfaz.

Para obtener un desglose práctico (por archivo inyectado, herramientas, Skills y tamaño del
prompt del sistema), utilice `/context list` o `/context detail`. Consulte
[Contexto](/es/concepts/context).

## Cómo consultar el uso actual de tokens

En el chat:

- `/status` -> tarjeta de estado con abundantes emojis que muestra el modelo de la sesión, el uso del contexto,
  los tokens de entrada/salida de la última respuesta y el coste estimado cuando los precios locales están
  configurados para el modelo activo.
- `/usage off|tokens|full` -> añade un pie de uso por respuesta a cada
  respuesta. Se conserva por sesión (almacenado como `responseUsage`).
  - `/usage reset` (alias: `inherit`, `clear`, `default`) borra la
    anulación de la sesión para que vuelva a heredar el valor predeterminado configurado.
  - `/usage tokens` muestra los detalles de tokens/caché del turno.
  - `/usage full` muestra detalles compactos del modelo/contexto/coste; el coste estimado
    solo aparece cuando OpenClaw dispone de metadatos de uso y precios locales para el
    modelo activo. Los diseños personalizados de `messages.usageTemplate` pueden incluir
    campos de tokens/caché.
- `/usage cost` -> resumen local de costes a partir de los registros de sesión de OpenClaw.

Otras superficies:

- **TUI/TUI web:** se admiten `/status` y `/usage`.
- **CLI:** `openclaw status --usage` y `openclaw channels list` muestran
  ventanas normalizadas de cuota del proveedor (`X% left`, no costes por respuesta).
  Proveedores actuales con ventanas de uso: Claude (Anthropic), ClawRouter, Copilot
  (GitHub), DeepSeek, Gemini (Google Gemini CLI), MiniMax, OpenAI, Xiaomi,
  Xiaomi Token Plan y z.ai.

Las superficies de uso normalizan los alias comunes de los campos nativos de los proveedores antes de
mostrarlos. Para el tráfico de Responses de la familia OpenAI, esto incluye tanto
`input_tokens`/`output_tokens` como `prompt_tokens`/`completion_tokens`, por lo que
los nombres de campo específicos del transporte no modifican `/status`, `/usage` ni los resúmenes
de sesión. También se normaliza el uso de Gemini CLI: el analizador predeterminado `stream-json`
lee los eventos `message` del asistente, y `stats.cached` se asigna a
`cacheRead`, con `stats.input_tokens - stats.cached` cuando la CLI omite
un campo `stats.input` explícito. Las anulaciones JSON heredadas siguen leyendo el texto de la respuesta
desde `response`.

Para el tráfico nativo de Responses de la familia OpenAI, los alias de uso de WebSocket/SSE se
normalizan del mismo modo, y los totales recurren a la suma normalizada de entrada + salida
cuando `total_tokens` falta o es `0`.

Cuando la instantánea de la sesión actual contiene pocos datos, `/status` y `session_status`
pueden recuperar los contadores de tokens/caché y la etiqueta del modelo activo del entorno de ejecución a partir del
registro de uso de transcripción más reciente. Los valores activos distintos de cero siguen
teniendo prioridad sobre los valores alternativos de la transcripción, y los totales de la transcripción
orientados al prompt que sean mayores pueden prevalecer cuando los totales almacenados faltan o son inferiores.

La autenticación de uso para las ventanas de cuota del proveedor procede primero de los hooks
específicos del proveedor; si un proveedor no dispone de un hook (o el hook no resuelve un token),
OpenClaw recurre a las credenciales OAuth/clave de API coincidentes de los perfiles de
autenticación, el entorno o la configuración.

Las entradas de transcripción del asistente conservan la misma estructura de uso normalizada,
incluido `usage.cost` cuando el modelo activo tiene precios configurados y el
proveedor devuelve metadatos de uso. Esto proporciona a `/usage cost` y
al estado de sesión basado en transcripciones una fuente estable incluso después de que desaparezca el estado
activo del entorno de ejecución.

OpenClaw mantiene la contabilidad de uso del proveedor separada de la instantánea actual del
contexto. El `usage.total` del proveedor puede incluir entrada en caché, salida y
varias llamadas al modelo dentro del bucle de herramientas, por lo que resulta útil para costes y telemetría, pero
puede sobreestimar la ventana de contexto activa. Las vistas y los diagnósticos del contexto utilizan
la instantánea del prompt más reciente (`promptTokens`, o la última llamada al modelo cuando no hay
ninguna instantánea de prompt disponible) para `context.used`.

## Estimación de costes (cuando se muestra)

Los costes se estiman a partir de la configuración de precios del modelo:

```text
models.providers.<provider>.models[].cost
```

Estos son valores en **USD por 1M de tokens** para `input`, `output`, `cacheRead` y
`cacheWrite`. Si faltan los precios, `/usage full` omite el coste; utilice
`/usage tokens` o un `messages.usageTemplate` personalizado cuando necesite
detalles de tokens/caché en cada respuesta. La visualización del coste no se limita a la
autenticación mediante clave de API: los proveedores que no usan claves de API, como `aws-sdk`, pueden mostrar el coste estimado cuando
la entrada de su modelo configurado incluye precios locales y el proveedor
devuelve metadatos de uso.

Después de que los procesos auxiliares y los canales alcancen la ruta de disponibilidad del Gateway, OpenClaw inicia un
proceso opcional en segundo plano para obtener precios de las referencias de modelos configuradas que aún no
dispongan de precios locales. Este proceso obtiene los catálogos remotos de precios de OpenRouter y
LiteLLM. Configure `models.pricing.enabled: false` para omitir la obtención de esos
catálogos en redes sin conexión o restringidas; las entradas explícitas de
`models.providers.*.models[].cost` siguen determinando las estimaciones locales de costes.

## Impacto del TTL de caché y la poda

El almacenamiento en caché del prompt del proveedor solo se aplica dentro de la ventana de TTL de la caché. OpenClaw
puede ejecutar opcionalmente la **poda por TTL de caché**: poda la sesión una vez que
el TTL de la caché ha caducado y luego restablece la ventana de caché para que las solicitudes posteriores
reutilicen el contexto recién almacenado en caché en lugar de volver a almacenar en caché todo el historial.
Esto reduce los costes de escritura en caché cuando una sesión permanece inactiva más allá del TTL.

Configúrelo en [Configuración del Gateway](/es/gateway/configuration) y consulte los
detalles del comportamiento en [Poda de sesiones](/es/concepts/session-pruning).

Heartbeat puede mantener la caché **activa** durante periodos de inactividad. Si el TTL de caché
del modelo es `1h`, configurar el intervalo de Heartbeat justo por debajo de ese valor (por ejemplo, `55m`) puede
evitar volver a almacenar en caché todo el prompt, lo que reduce los costes de escritura en caché.

En configuraciones con varios agentes, puede mantener una configuración de modelo compartida y ajustar el comportamiento de
la caché por agente mediante `agents.entries.*.params.cacheRetention`.

Para obtener una guía completa de cada opción, consulte [Almacenamiento en caché de prompts](/es/reference/prompt-caching).

En los precios de la API de Anthropic, las lecturas de caché son considerablemente más económicas que los tokens
de entrada, mientras que las escrituras en caché se facturan con un multiplicador superior. Consulte los precios
de almacenamiento en caché de prompts de Anthropic para conocer las tarifas y los multiplicadores de TTL más recientes:
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Ejemplo: mantener activa una caché de 1h con Heartbeat

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Ejemplo: tráfico mixto con estrategia de caché por agente

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # línea base predeterminada para la mayoría de los agentes
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # mantener activa la caché de larga duración para sesiones profundas
    - id: "alerts"
      params:
        cacheRetention: "none" # evitar escrituras en caché para notificaciones en ráfagas
```

`agents.entries.*.params` se combina sobre `params` del modelo seleccionado, por lo que se
puede sobrescribir únicamente `cacheRetention` y heredar los demás valores predeterminados del modelo
sin cambios.

### Contexto de 1M de Anthropic

OpenClaw configura los modelos Claude 4.x con disponibilidad general, como Opus 4.8, Opus 4.7, Opus
4.6 y Sonnet 4.6, con la ventana de contexto de 1M de Anthropic. No se necesita
`params.context1m: true` para esos modelos.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

Las configuraciones antiguas pueden conservar `context1m: true`, pero OpenClaw ya no envía
el encabezado beta retirado `context-1m-2025-08-07` de Anthropic para este ajuste ni
amplía a 1M los modelos Claude antiguos no compatibles.

Requisito: la credencial debe ser apta para el uso de contexto largo. De lo contrario,
Anthropic responde con un error de límite de tasa del proveedor para esa solicitud.

Si la autenticación con Anthropic se realiza mediante tokens de OAuth/suscripción
(`sk-ant-oat-*`), OpenClaw conserva los encabezados beta de Anthropic requeridos por OAuth
y elimina el beta retirado `context-1m-*` si aún permanece en una
configuración antigua.

## Consejos para reducir la presión de tokens

- Usar `/compact` para resumir sesiones largas.
- Recortar las salidas extensas de las herramientas en los flujos de trabajo.
- Reducir `agents.defaults.imageMaxDimensionPx` para sesiones con muchas capturas de pantalla.
- Mantener breves las descripciones de las Skills (la lista de Skills se inserta en el prompt).
- Preferir modelos más pequeños para trabajos detallados y exploratorios.

Consultar [Skills](/es/tools/skills) para conocer la fórmula exacta de la sobrecarga de la lista de Skills.

## Temas relacionados

- [Uso y costes de la API](/es/reference/api-usage-costs)
- [Almacenamiento en caché de prompts](/es/reference/prompt-caching)
- [Seguimiento del uso](/es/concepts/usage-tracking)
