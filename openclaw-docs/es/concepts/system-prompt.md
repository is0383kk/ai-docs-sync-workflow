---
read_when:
    - Edición del texto del prompt del sistema, la lista de herramientas o las secciones de tiempo/Heartbeat
    - Cambiar el comportamiento de inicialización del espacio de trabajo o de inyección de Skills
summary: Qué contiene el prompt del sistema de OpenClaw y cómo se ensambla
title: Prompt del sistema
x-i18n:
    generated_at: "2026-07-26T04:36:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 669fbc6f21a82a2c3c067d2ff3a6365acb3316460a85f2db165b7ad49ce79f70
    source_path: concepts/system-prompt.md
    workflow: 16
---

OpenClaw crea su propio prompt del sistema para cada ejecución del agente; no existe un prompt predeterminado en tiempo de ejecución.

El ensamblaje consta de tres capas:

- `buildAgentSystemPrompt` renderiza el prompt a partir de entradas explícitas. Se mantiene como un renderizador puro y no lee directamente la configuración global.
- `resolveAgentSystemPromptConfig` resuelve los ajustes del prompt respaldados por la configuración (visualización del propietario, indicaciones de TTS, alias de modelos, modo de citas de memoria, modo de delegación en subagentes) para un agente específico.
- Los adaptadores de tiempo de ejecución (integrados, CLI, vistas previas de comandos/exportaciones, Compaction) recopilan datos actuales (herramientas, estado del entorno aislado, capacidades del canal, archivos de contexto, contribuciones al prompt del proveedor) y llaman a la fachada de prompt configurada.

Esto mantiene las superficies de prompt exportadas y de depuración alineadas con las ejecuciones en vivo sin convertir cada detalle del tiempo de ejecución en un único constructor monolítico.

Los plugins de proveedores pueden aportar orientación consciente de la caché sin reemplazar el prompt propiedad de OpenClaw. Un tiempo de ejecución de proveedor puede:

- reemplazar una de las tres secciones principales con nombre: `interaction_style`, `tool_call_style`, `execution_bias`
- inyectar un **prefijo estable** por encima del límite de la caché del prompt
- inyectar un **sufijo dinámico** por debajo del límite de la caché del prompt

Use contribuciones propiedad del proveedor para ajustes específicos de familias de modelos. Reserve el enlace heredado `before_prompt_build` para compatibilidad o cambios de prompt verdaderamente globales.

La capa incluida para la familia GPT-5 de OpenAI/Codex (`resolveGpt5SystemPromptContribution`) usa este mecanismo: un contrato de comportamiento `stablePrefix` (política de ejecución, disciplina de herramientas, contrato de salida, contrato de finalización) más una sustitución opcional `interaction_style` para un tono más cordial. Se aplica a cualquier identificador de modelo `gpt-5*` enrutado a través de los plugins de OpenAI o Codex, controlado por `agents.defaults.promptOverlays.gpt5.personality` (`"friendly"`/`"on"` o `"off"`).

## Estructura

El prompt es compacto y contiene secciones fijas:

- **Herramientas**: recordatorio de que las herramientas estructuradas son la fuente de verdad, además de orientación sobre su uso en tiempo de ejecución. Cuando la herramienta experimental `update_plan` está habilitada (`tools.experimental.planTool`), su propia descripción añade: usarla solo para trabajos no triviales de varios pasos, mantener como máximo un paso `in_progress` y omitirla para trabajos sencillos de un solo paso.
- **Preferencia de ejecución**: actuar en el turno ante solicitudes ejecutables, continuar hasta terminar o quedar bloqueado, recuperarse de resultados deficientes de las herramientas, comprobar en vivo el estado mutable y verificar antes de finalizar.
- **Seguridad**: breve recordatorio de protección contra conductas orientadas a acumular poder o eludir la supervisión.
- **Skills** (cuando estén disponibles): indica al modelo cómo cargar bajo demanda las instrucciones de Skills.
- **Control de OpenClaw**: preferir la herramienta `gateway` para tareas de configuración o reinicio; no inventar comandos de la CLI.
- **Actualización automática de OpenClaw**: inspeccionar la configuración de forma segura con `config.schema.lookup`, aplicar parches con `config.patch`, reemplazar toda la configuración con `config.apply` y ejecutar `update.run` solo por solicitud explícita del usuario. La herramienta `gateway` orientada al agente se niega a reescribir `tools.exec.mode`.
- **Espacio de trabajo**: directorio de trabajo (`agents.defaults.workspace`).
- **Documentación**: ruta local de la documentación o el código fuente y cuándo consultarlos.
- **Archivos del espacio de trabajo (inyectados)**: señala que los archivos de arranque se incluyen a continuación.
- **Entorno aislado** (cuando esté habilitado): tiempo de ejecución aislado, rutas del entorno aislado y disponibilidad de ejecución con privilegios elevados.
- **Fecha y hora actuales**: solo la zona horaria (estable para la caché; el reloj en vivo procede de `session_status`).
- **Directivas de salida del asistente**: sintaxis compacta para archivos adjuntos, notas de voz y etiquetas de respuesta.
- **Heartbeats**: prompt de Heartbeat y comportamiento de confirmación, cuando los Heartbeats están habilitados para el agente predeterminado.
- **Tiempo de ejecución**: host, sistema operativo, Node, modelo, raíz del repositorio (cuando se detecte) y nivel de razonamiento (una línea).
- **Razonamiento**: nivel de visibilidad actual y sugerencia del conmutador `/reasoning`.

El contenido estable de gran tamaño (incluido el **contexto del proyecto**) permanece por encima del límite interno de la caché del prompt. Las secciones volátiles de cada turno (orientación integrada de la interfaz de control, **mensajería**, **voz**, **contexto del chat grupal**, **reacciones**, **Heartbeats**, **tiempo de ejecución**) se añaden por debajo de ese límite para que los backends locales con cachés de prefijos puedan reutilizar el prefijo estable del espacio de trabajo entre turnos de canal. Las descripciones de herramientas deben evitar incorporar nombres de canales actuales cuando el esquema aceptado ya contiene ese detalle del tiempo de ejecución.

La sección de herramientas también contiene orientación para trabajos de larga duración:

- usar Cron para el seguimiento futuro (`check back later`, recordatorios, trabajo recurrente) en lugar de bucles de suspensión de `exec`, trucos de retraso de `yieldMs` o consultas repetidas de `process`
- usar `exec` / `process` solo para comandos que comienzan ahora y continúan en segundo plano
- cuando esté habilitada la reactivación automática al completarse, iniciar el comando una vez y confiar en la ruta de reactivación basada en notificaciones
- usar `process` para registros, estado, entrada o intervención en un comando en ejecución
- para tareas de mayor envergadura, preferir `sessions_spawn`; la finalización del subagente se basa en notificaciones y se anuncia automáticamente al solicitante
- no consultar `subagents list` / `sessions_list` en un bucle únicamente para esperar a que finalice

`agents.defaults.subagents.delegationMode` (valor predeterminado: `"suggest"`) puede reforzar esto. `"prefer"` añade una sección específica de **delegación en subagentes** que indica al agente principal que actúe como coordinador receptivo y canalice mediante `sessions_spawn` todo lo que sea más complejo que una respuesta directa. Esto solo afecta al prompt; la política de herramientas sigue controlando si `sessions_spawn` está disponible.

Las medidas de seguridad del prompt del sistema son orientativas, no coercitivas. Use la política de herramientas, las aprobaciones de ejecución, el aislamiento y las listas de canales permitidos para aplicar restricciones estrictas; por diseño, los operadores pueden deshabilitar las medidas de seguridad del prompt.

En canales con tarjetas o botones de aprobación nativos, el prompt indica al agente que se apoye primero en esa interfaz y que incluya un comando manual `/approve` solo cuando el resultado de la herramienta indique que las aprobaciones por chat no están disponibles o que la aprobación manual es la única vía.

## Modos del prompt

OpenClaw renderiza prompts del sistema más pequeños para los subagentes. El tiempo de ejecución establece un `promptMode` por ejecución (no es una configuración visible para el usuario):

- `full` (predeterminado): todas las secciones anteriores.
- `minimal`: se usa para subagentes; omite la sección del prompt de memoria (incluida como **recuperación de memoria**), **actualización automática de OpenClaw**, **alias de modelos**, **identidad del usuario**, **directivas de salida del asistente**, **mensajería**, **respuestas silenciosas** y **Heartbeats**. Las herramientas, la **seguridad**, las **Skills** (cuando se proporcionen), el espacio de trabajo, el entorno aislado, la fecha y hora actuales (cuando se conozcan), el tiempo de ejecución y el contexto inyectado siguen disponibles.
- `none`: devuelve únicamente la línea de identidad base.

Con `promptMode=minimal`, los prompts adicionales inyectados se etiquetan como **contexto del subagente** en lugar de **contexto del chat grupal**.

Para ejecuciones de respuesta automática en canales, OpenClaw omite la sección genérica de **respuestas silenciosas** cuando el contexto directo, grupal o exclusivo de la herramienta de mensajería ya controla el contrato de respuesta visible. Solo el modo automático heredado de grupo o canal muestra `NO_REPLY`; los chats directos y las respuestas exclusivas de la herramienta de mensajería omiten la orientación sobre tokens silenciosos.

## Instantáneas de prompts

OpenClaw mantiene instantáneas de prompts confirmadas para el flujo principal del tiempo de ejecución de Codex en `test/fixtures/agents/prompt-snapshots/codex-runtime-happy-path/`. Renderizan parámetros seleccionados de hilo y turno del servidor de aplicaciones, además de una pila reconstruida de capas de prompts vinculadas al modelo para turnos directos de Telegram, grupales de Discord y de Heartbeat: una muestra fijada del prompt del modelo Codex `gpt-5.5`, el texto de desarrollador de permisos del flujo principal de Codex, las instrucciones de desarrollador de OpenClaw, las instrucciones del modo de colaboración limitadas al turno cuando OpenClaw las proporciona, la entrada del turno del usuario y referencias a especificaciones dinámicas de herramientas.

Actualice la muestra fijada del prompt del modelo Codex con `pnpm prompt:snapshots:sync-codex-model`. De forma predeterminada, busca `$CODEX_HOME/models_cache.json`, después `~/.codex/models_cache.json` y, a continuación, la convención de checkout del mantenedor `~/code/codex/codex-rs/models-manager/models.json`; si ninguno existe, finaliza sin modificar la muestra confirmada. Pase `--catalog <path>` para actualizar desde un archivo `models_cache.json` o `models.json` específico.

Estas instantáneas no son una captura sin procesar byte por byte de una solicitud de OpenAI. Codex puede añadir contexto del espacio de trabajo propiedad del tiempo de ejecución (`AGENTS.md`, contexto del entorno, memorias, instrucciones de aplicaciones o plugins e instrucciones integradas del modo de colaboración predeterminado) después de que OpenClaw envíe los parámetros del hilo y del turno.

Vuelva a generarlas con `pnpm prompt:snapshots:gen`; compruebe las divergencias con `pnpm prompt:snapshots:check`. La CI ejecuta la comprobación de divergencias junto con las particiones de límites adicionales, de modo que los cambios de prompts y las actualizaciones de instantáneas se incorporen en el mismo PR.

## Inyección del arranque del espacio de trabajo

Los archivos de arranque se resuelven desde el espacio de trabajo activo y se dirigen a la superficie del prompt correspondiente a su ciclo de vida:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md` (solo en espacios de trabajo recién creados)
- `MEMORY.md` cuando esté presente

En el entorno nativo de Codex, OpenClaw evita repetir archivos estables del espacio de trabajo en cada turno del usuario. Codex carga `AGENTS.md` mediante su propio mecanismo de detección de documentación del proyecto. `TOOLS.md` se reenvía como instrucciones de desarrollador heredadas de Codex. `SOUL.md`, `IDENTITY.md` y `USER.md` se reenvían como instrucciones de desarrollador de colaboración limitadas al turno, para que los subagentes nativos de Codex no las hereden. El contenido de `HEARTBEAT.md` no se inyecta directamente; los turnos de Heartbeat reciben una nota del modo de colaboración que apunta al archivo cuando existe y no está vacío. El contenido de `MEMORY.md` tampoco se pega en cada turno nativo de Codex: cuando las herramientas de memoria están disponibles para el espacio de trabajo, los turnos de Codex reciben una pequeña nota de memoria del espacio de trabajo que dirige el modelo a `memory_search` o `memory_get`. Si las herramientas están deshabilitadas, la búsqueda en memoria no está disponible o el espacio de trabajo activo difiere del espacio de trabajo de memoria del agente, `MEMORY.md` recurre a la ruta normal y acotada del contexto del turno. `BOOTSTRAP.md` conserva la función normal de contexto del turno.

En entornos que no son de Codex, los archivos de arranque se incorporan al prompt de OpenClaw conforme a sus condiciones existentes. `HEARTBEAT.md` se omite en las ejecuciones normales cuando los Heartbeats están deshabilitados para el agente predeterminado o `agents.defaults.heartbeat.includeSystemPromptSection` es falso. Mantenga concisos los archivos inyectados, especialmente el `MEMORY.md` que no es de Codex: debe seguir siendo un resumen seleccionado a largo plazo, con notas diarias detalladas en `memory/*.md` que puedan recuperarse bajo demanda mediante `memory_search` / `memory_get`. Los archivos `MEMORY.md` de gran tamaño que no son de Codex aumentan el uso del prompt y pueden inyectarse parcialmente según los límites de archivos de arranque indicados a continuación.

<Note>
Los archivos diarios `memory/*.md` **no** forman parte del contexto del proyecto de arranque normal. En los turnos ordinarios se accede a ellos bajo demanda mediante `memory_search` / `memory_get`, por lo que no cuentan para la ventana de contexto salvo que el modelo los lea explícitamente. Los turnos simples de `/new` y `/reset` son la excepción: el tiempo de ejecución puede anteponer la memoria diaria reciente como un bloque de contexto de inicio de un solo uso para ese primer turno.
</Note>

Los archivos grandes se truncan con un marcador:

| Límite                                        | Clave de configuración                              | Valor predeterminado |
| --------------------------------------------- | --------------------------------------------------- | -------------------- |
| Máximo de caracteres por archivo              | `agents.defaults.bootstrapMaxChars`                                  | 20000                |
| Total entre todos los archivos                | `agents.defaults.bootstrapTotalMaxChars`                                  | 60000                |
| Advertencia de truncamiento (`off`\|`once`\|`always`) | `agents.defaults.bootstrapPromptTruncationWarning` | `always` |

Los archivos que faltan insertan un marcador breve de archivo ausente. Los recuentos detallados de contenido sin procesar/inyectado permanecen en diagnósticos como `/context`, `/status`, doctor y los registros.

En el caso de los archivos de memoria, el truncamiento no supone pérdida de datos: el archivo permanece intacto en el disco. En Codex nativo, `MEMORY.md` se lee bajo demanda mediante las herramientas de memoria cuando están disponibles; de lo contrario, se utiliza como alternativa una versión limitada en el prompt. En otros entornos de ejecución, el modelo solo ve la copia inyectada abreviada hasta que lee o busca directamente en la memoria. Si `MEMORY.md` se trunca repetidamente, condénselo en un resumen duradero más breve, traslade el historial detallado a `memory/*.md` o aumente intencionadamente los límites de arranque.

Las sesiones de subagentes solo inyectan `AGENTS.md` y `TOOLS.md` (los demás archivos de arranque se filtran para mantener reducido el contexto de los subagentes).

Los hooks internos pueden interceptar este paso mediante el evento `agent:bootstrap` para modificar o sustituir los archivos de arranque inyectados (por ejemplo, sustituir `SOUL.md` por una personalidad alternativa).

Para que el tono resulte menos genérico, comience con la [guía de personalidad de SOUL.md](/es/concepts/soul).

Para inspeccionar cuánto aporta cada archivo inyectado (contenido sin procesar frente a inyectado, truncamiento y sobrecarga del esquema de herramientas), utilice `/context list` o `/context detail`. Consulte [Contexto](/es/concepts/context).

## Gestión del tiempo

La sección **Fecha y hora actuales** solo aparece cuando se conoce la zona horaria del usuario y únicamente incluye la **zona horaria** (sin reloj dinámico ni formato de hora) para mantener estable la caché del prompt.

Utilice `session_status` cuando el agente necesite la hora actual; su tarjeta de estado incluye una línea con la marca de tiempo. La misma herramienta puede establecer opcionalmente una sustitución del modelo por sesión (`model=default` la elimina).

Configuración:

- `agents.defaults.userTimezone`
- `agents.defaults.timeFormat` (`auto` | `12` | `24`)

Consulte [Zonas horarias](/es/concepts/timezone) y [Fecha y hora](/es/date-time) para obtener todos los detalles sobre el comportamiento.

## Skills

Cuando existen Skills aptas, OpenClaw inyecta una lista compacta `<available_skills>` (`formatSkillsForPrompt`) con la **ruta del archivo** y un marcador `<version>sha256:...</version>` derivado del contenido para cada Skill. El prompt indica al modelo que utilice `read` para cargar el archivo SKILL.md desde la ubicación indicada (espacio de trabajo, administrada o incluida) y que vuelva a leer una Skill cuando su `<version>` difiera del de un turno anterior. Si no hay Skills aptas, se omite la sección Skills.

Los turnos de Codex nativo reciben esta lista como instrucciones de desarrollador de colaboración limitadas al turno, en lugar de como entrada del usuario en cada turno, salvo los turnos ligeros de cron que conservan el prompt programado exacto. Los demás entornos de ejecución mantienen la sección normal del prompt.

La ubicación puede apuntar a una Skill anidada, como `skills/personal/foo/SKILL.md`. El anidamiento solo tiene fines organizativos; el prompt utiliza el nombre plano de la Skill procedente del frontmatter `SKILL.md`.

La aptitud incluye las condiciones de metadatos de la Skill, las comprobaciones del entorno/configuración de ejecución y la lista efectiva de Skills permitidas para el agente cuando se configura `agents.defaults.skills` o `agents.entries.*.skills`. Las Skills incluidas con un Plugin solo son aptas cuando su Plugin propietario está habilitado, lo que permite que los Plugins de herramientas expongan guías operativas más detalladas sin incorporar toda esa orientación en la descripción de cada herramienta.

```xml
<available_skills>
  <skill>
    <name>...</name>
    <description>...</description>
    <location>...</location>
    <version>sha256:...</version>
  </skill>
</available_skills>
```

Esto mantiene reducido el prompt base y, al mismo tiempo, permite el uso específico de Skills. El dimensionamiento pertenece al subsistema de Skills, separado del dimensionamiento genérico de lectura/inyección durante la ejecución:

| Ámbito     | Presupuesto del prompt de Skills                     | Presupuesto de extractos de ejecución |
| ---------- | ---------------------------------------------------- | ------------------------------------- |
| Global     | `skills.limits.maxSkillsPromptChars`                                   | `agents.defaults.contextLimits.*`                    |
| Por agente | `agents.entries.*.skillsLimits.maxSkillsPromptChars`                                   | `agents.entries.*.contextLimits.*`                    |

El presupuesto de extractos de ejecución abarca `memory_get`, los resultados de herramientas en tiempo real y las actualizaciones de `AGENTS.md` posteriores a la Compaction.

## Documentación

La sección **Documentación** apunta a la documentación local cuando está disponible (`docs/` en un checkout de Git o la documentación incluida en el paquete npm) y, en caso contrario, recurre a [https://docs.openclaw.ai](https://docs.openclaw.ai). También indica la ubicación del código fuente de OpenClaw: los checkouts de Git muestran la raíz local del código fuente, mientras que las instalaciones de paquetes proporcionan la URL del código fuente en GitHub con instrucciones para revisarlo allí cuando la documentación esté incompleta o desactualizada.

El prompt presenta la documentación como la fuente autorizada para el conocimiento de OpenClaw antes de que el modelo comprenda cómo funciona OpenClaw (memoria/notas diarias, sesiones, herramientas, Gateway, configuración, comandos y contexto del proyecto), e indica al modelo que trate `AGENTS.md`, el contexto del proyecto, las notas del espacio de trabajo/perfil/memoria y `memory_search` como contexto de instrucciones o memoria del usuario, y no como conocimiento sobre el diseño o la implementación de OpenClaw. Si la documentación no contiene información o está desactualizada, el modelo debe indicarlo e inspeccionar el código fuente. También indica al modelo que ejecute `openclaw status` por sí mismo cuando sea posible y que solo se lo solicite al usuario cuando no tenga acceso.

En concreto, para la configuración, dirige a los agentes a la acción `config.schema.lookup` de la herramienta `gateway` para consultar la documentación y las restricciones exactas de cada campo, y después a `docs/gateway/configuration.md` y `docs/gateway/configuration-reference.md` para obtener orientación más general.

## Temas relacionados

- [Ejecución del agente](/es/concepts/agent)
- [Espacio de trabajo del agente](/es/concepts/agent-workspace)
- [Motor de contexto](/es/concepts/context-engine)
