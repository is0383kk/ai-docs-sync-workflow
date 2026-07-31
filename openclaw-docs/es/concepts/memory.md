---
read_when:
    - Quieres entender cómo funciona la memoria
    - Quieres saber qué archivos de memoria escribir
summary: Cómo recuerda OpenClaw la información entre sesiones
title: Descripción general de la memoria
x-i18n:
    generated_at: "2026-07-26T05:10:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cdfd5276d6289a4ee38b5203eb5443312c4b040d4ea67abe4a9c579703136339
    source_path: concepts/memory.md
    workflow: 16
---

OpenClaw recuerda información escribiendo archivos Markdown sin formato en el
espacio de trabajo del agente (valor predeterminado: `~/.openclaw/workspace`). El modelo solo recuerda lo que se
guarda en el disco; no hay ningún estado oculto.

## Cómo funciona

El agente tiene tres archivos relacionados con la memoria:

- **`MEMORY.md`** — memoria a largo plazo. Datos, preferencias y
  decisiones duraderos. Se carga al inicio de una sesión.
- **`memory/YYYY-MM-DD.md`** (o `memory/YYYY-MM-DD-<slug>.md`) — notas diarias.
  Contexto y observaciones en curso. Las notas fechadas de hoy y ayer se cargan
  automáticamente con un `/new` o `/reset` básico; las variantes con identificador, como las
  escritas por el hook de memoria de sesión incluido, se recogen junto con el
  archivo que solo contiene la fecha.
- **`DREAMS.md`** (opcional) — Diario de Dreaming y resúmenes de barridos de Dreaming para
  revisión humana, incluidas entradas fundamentadas de incorporación retrospectiva histórica.

<Tip>
Si se quiere que el agente recuerde algo, basta con pedírselo: «Recuerda que
prefiero TypeScript». El agente escribe la nota en el archivo correspondiente.
</Tip>

## Qué va en cada lugar

`MEMORY.md` es la capa compacta y seleccionada: datos duraderos, preferencias, decisiones
permanentes y resúmenes breves que deben estar disponibles al inicio de una
sesión. No es una transcripción sin procesar, un registro diario ni un archivo exhaustivo.

Los archivos `memory/YYYY-MM-DD.md` son la capa de trabajo: notas diarias detalladas,
observaciones, resúmenes de sesiones y contexto sin procesar que aún puede resultar útil
más adelante. Se indexan para `memory_search` y `memory_get`, pero no se
inyectan en el prompt de arranque en cada turno.

Con el tiempo, el agente extrae material útil de las notas diarias y lo incorpora a
`MEMORY.md`, además de eliminar las entradas obsoletas de la memoria a largo plazo. Las instrucciones
generadas del espacio de trabajo y el flujo de Heartbeat realizan esta tarea periódicamente; no es necesario
editar manualmente `MEMORY.md` para cada detalle.

Si `MEMORY.md` supera el presupuesto de archivos de arranque, OpenClaw mantiene intacto el archivo en
el disco, pero trunca la copia que se inyecta en el contexto. Esto debe interpretarse como una
señal para trasladar el material detallado a `memory/*.md`, conservar solo un resumen
duradero en `MEMORY.md` o aumentar los límites de arranque si se quiere dedicar más
presupuesto del prompt. Se puede usar `/context list`, `/context detail` o `openclaw doctor` para
consultar los tamaños sin procesar e inyectados, así como el estado de truncamiento.

## Importar desde asistentes de programación

La interfaz de control puede importar memoria local existente de Codex y Claude Code.
Abra **Configuración** → **Importar memoria**, elija el agente de destino, revise los
archivos detectados y confirme la importación. OpenClaw solo copia memoria en Markdown:

- Codex: los archivos consolidados `MEMORY.md` y `memory_summary.md` dentro de
  `~/.codex/memories` (o `CODEX_HOME/memories`). Los archivos de ejecuciones y transcripciones
  sin procesar no se importan.
- Claude Code: archivos Markdown del directorio de memoria automática de cada proyecto dentro de
  `~/.claude/projects/*/memory`, además de un
  `autoMemoryDirectory` configurado por el usuario cuando esté presente. Las instrucciones del proyecto, las sesiones, la configuración
  y las credenciales no forman parte de esta acción exclusiva de memoria.

Los archivos importados permanecen separados dentro de `memory/imports/codex/` y
`memory/imports/claude-code/` en el espacio de trabajo del agente seleccionado. Se indexan
para `memory_search` y están disponibles mediante `memory_get`; no se combinan con
el archivo de arranque `MEMORY.md` del agente. Los archivos de origen no se modifican.

La vista previa marca los conflictos de destino. Active **Reemplazar importaciones existentes** para
reemplazar esos archivos; al aplicar la operación, se crea una copia de seguridad verificada previa a la importación y se conservan
copias individuales de los archivos sobrescritos en el informe de migración.

## Memorias sensibles a las acciones

La mayoría de los recuerdos son notas Markdown normales. Algunos influyen en lo que el agente debe
hacer más adelante; en esos casos, se debe registrar cuándo es seguro actuar según la nota, no solo
el dato en sí.

Registre ese límite de acción cuando una nota incluya:

- requisitos de aprobación o permiso,
- restricciones temporales,
- transferencias a otra sesión, hilo o persona,
- condiciones de expiración,
- momento seguro para actuar,
- autoridad de la fuente o del responsable,
- instrucciones para evitar una acción tentadora.

Una memoria útil sensible a las acciones deja claro:

- qué cambia el comportamiento futuro,
- cuándo o bajo qué condición se aplica,
- cuándo expira o qué habilita la acción,
- qué debe evitar hacer el agente,
- quién es la fuente o el responsable, si esto afecta a la confianza o la autoridad.

La memoria puede conservar el contexto de aprobación, pero no aplica políticas. Use
la configuración de aprobación de OpenClaw, el aislamiento y las tareas programadas como
controles operativos estrictos.

Ejemplo:

```md
La migración de la API se está diseñando en otra sesión. Los turnos futuros no deben
editar la implementación de la API desde este hilo; los hallazgos de aquí solo deben usarse como
datos de diseño hasta que se incorpore el plan de migración.
```

Otro ejemplo:

```md
Un informe de una fuente no fiable debe revisarse antes de promoverse. Los turnos futuros
deben tratarlo únicamente como evidencia; no debe almacenarse como memoria duradera hasta que un
revisor de confianza confirme el contenido.
```

Este esquema no es obligatorio para todos los recuerdos; los datos sencillos pueden ser concisos.
Use límites sensibles a las acciones cuando perder el contexto temporal, de autoridad, de expiración o
del momento seguro para actuar pueda provocar que el agente haga algo incorrecto más adelante.

Use [tareas programadas](/es/automation/cron-jobs) para recordatorios exactos, comprobaciones temporizadas
y trabajos recurrentes. La memoria puede seguir resumiendo el contexto duradero relacionado con ese
trabajo.

## Compromisos inferidos retirados

Algunos seguimientos futuros no son datos duraderos. Si se menciona una entrevista
para mañana, la memoria útil puede ser «hacer un seguimiento después de la entrevista», no «guardar
esto para siempre en `MEMORY.md`».

El experimento de compromisos inferidos se ha retirado. OpenClaw ya no extrae ni
entrega esos seguimientos. Use [tareas programadas](/es/automation/cron-jobs) para
acciones futuras; el comando heredado `openclaw commitments` sigue disponible para
inspeccionar o descartar filas almacenadas existentes.

## Herramientas de memoria

El agente dispone de dos herramientas para trabajar con la memoria:

- **`memory_search`** — busca notas relevantes mediante búsqueda semántica, incluso cuando
  la redacción difiere de la original.
- **`memory_get`** — lee un archivo de memoria o intervalo de líneas específico.

Ambas herramientas las proporciona el Plugin de memoria activo (valor predeterminado: `memory-core`).

## Búsqueda en la memoria

Cuando hay un proveedor de embeddings configurado, `memory_search` usa búsqueda híbrida:
similitud vectorial (significado semántico) combinada con coincidencia de palabras clave (términos
exactos como identificadores y símbolos de código). Funciona directamente con una clave de API
de cualquier proveedor compatible.

<Info>
OpenClaw usa embeddings de OpenAI de forma predeterminada. Configure
`memory.search.provider` explícitamente para usar Gemini, Voyage,
Mistral, Bedrock, DeepInfra, GGUF local, Ollama, LM Studio, GitHub Copilot o
un endpoint genérico compatible con OpenAI.
</Info>

Consulte [Búsqueda en la memoria](/es/concepts/memory-search) para obtener información sobre el funcionamiento de la búsqueda, las opciones
de ajuste y la configuración de proveedores.

## Backends de memoria

<CardGroup cols={3}>
<Card title="Integrado (predeterminado)" icon="database" href="/es/concepts/memory-builtin">
Basado en SQLite. Funciona directamente con búsqueda por palabras clave, similitud vectorial y
búsqueda híbrida. No requiere dependencias adicionales.
</Card>
<Card title="QMD" icon="search" href="/es/concepts/memory-qmd">
Servicio auxiliar con prioridad local, reclasificación, expansión de consultas y capacidad para indexar
directorios fuera del espacio de trabajo.
</Card>
<Card title="Honcho" icon="brain" href="/es/concepts/memory-honcho">
Memoria nativa de IA entre sesiones, con modelado de usuarios, búsqueda semántica y
conocimiento de múltiples agentes. Requiere instalar el Plugin.
</Card>
<Card title="LanceDB" icon="layers" href="/es/plugins/memory-lancedb">
Memoria basada en LanceDB con embeddings compatibles con OpenAI, recuperación automática,
captura automática y compatibilidad con embeddings locales de Ollama. Requiere instalar el Plugin.
</Card>
</CardGroup>

## Capa de wiki de conocimiento

Si se quiere que la memoria duradera se comporte más como una base de conocimiento mantenida
que como notas sin procesar, use el Plugin incluido `memory-wiki`. Compila el conocimiento
duradero en un repositorio wiki con una estructura de páginas determinista, afirmaciones
y evidencias estructuradas, seguimiento de contradicciones y vigencia, paneles
generados, compendios compilados y herramientas nativas de wiki (`wiki_status`,
`wiki_search`, `wiki_get`, `wiki_apply`, `wiki_lint`).

`memory-wiki` no sustituye al Plugin de memoria activo; este sigue
encargándose de la recuperación, la promoción y Dreaming. `memory-wiki` añade una
capa de conocimiento rica en procedencia a su lado.

<CardGroup cols={1}>
<Card title="Wiki de memoria" icon="book" href="/es/plugins/memory-wiki">
Compila la memoria duradera en un repositorio wiki rico en procedencia, con afirmaciones,
paneles, modo puente y flujos de trabajo compatibles con Obsidian.
</Card>
</CardGroup>

## Vaciado automático de memoria

Antes de que [Compaction](/es/concepts/compaction) resuma la conversación,
OpenClaw ejecuta un turno silencioso que recuerda al agente que guarde el contexto importante
en los archivos de memoria. Está activado de forma predeterminada; configure
`agents.defaults.compaction.memoryFlush.enabled: false` para desactivarlo.

Para mantener ese turno de mantenimiento en un modelo local, configure una sustitución exacta que
se aplique únicamente al turno de vaciado de memoria (no hereda la cadena de modelos
alternativos de la sesión activa):

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

<Tip>
El vaciado de memoria evita la pérdida de contexto durante Compaction. Si el agente tiene
datos importantes en la conversación que todavía no se han escrito en un archivo, se
guardan automáticamente antes de generar el resumen.
</Tip>

## Dreaming

Dreaming es un proceso opcional de consolidación de memoria en segundo plano. Recopila
señales de recuperación a corto plazo, puntúa candidatos y solo promueve los elementos
que cumplen los requisitos a la memoria a largo plazo (`MEMORY.md`):

- **Activación opcional**: desactivado de forma predeterminada.
- **Programado**: cuando está activado, `memory-core` gestiona automáticamente un trabajo Cron
  recurrente para realizar un barrido completo de Dreaming.
- **Sujeto a umbrales**: las promociones deben superar los umbrales de puntuación, frecuencia de recuperación y
  diversidad de consultas.
- **Revisable**: los resúmenes de fases y las entradas del diario se escriben en
  `DREAMS.md` para su revisión humana.

Consulte [Dreaming](/es/concepts/dreaming) para obtener información sobre el comportamiento de las fases, las señales de puntuación y
los detalles del Diario de Dreaming.

## Incorporación retrospectiva fundamentada y promoción en directo

El sistema de Dreaming tiene dos vías de revisión relacionadas:

- **Dreaming en directo** trabaja con el almacén de Dreaming a corto plazo ubicado en
  `memory/.dreams/` y es lo que utiliza la fase profunda normal para decidir qué
  se incorpora a `MEMORY.md`.
- **Incorporación retrospectiva fundamentada** lee notas históricas de `memory/YYYY-MM-DD.md` como
  archivos diarios independientes y escribe resultados de revisión estructurados en `DREAMS.md`.

La incorporación retrospectiva fundamentada resulta útil para reproducir notas antiguas e inspeccionar qué
considera duradero el sistema, sin editar manualmente `MEMORY.md`.

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

La opción `--stage-short-term` incorpora provisionalmente candidatos duraderos fundamentados en el mismo
almacén de Dreaming a corto plazo que ya utiliza la fase profunda normal; no
los promueve directamente. Por tanto:

- `DREAMS.md` sigue siendo la superficie de revisión humana.
- El almacén a corto plazo sigue siendo la superficie de clasificación orientada a la máquina.
- `MEMORY.md` solo se escribe mediante la promoción profunda.

Para deshacer una reproducción sin modificar las entradas normales del diario ni el estado
habitual de recuperación:

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # Comprobar el estado del índice y el proveedor
openclaw memory search "query"  # Buscar desde la línea de comandos
openclaw memory index --force   # Reconstruir el índice
```

## Lecturas adicionales

- [Búsqueda en memoria](/es/concepts/memory-search): pipeline de búsqueda, proveedores y optimización.
- [Motor de memoria integrado](/es/concepts/memory-builtin): backend SQLite predeterminado.
- [Motor de memoria QMD](/es/concepts/memory-qmd): proceso auxiliar avanzado con enfoque local.
- [Memoria Honcho](/es/concepts/memory-honcho): memoria nativa de IA entre sesiones.
- [Memoria LanceDB](/es/plugins/memory-lancedb): Plugin basado en LanceDB con embeddings compatibles con OpenAI.
- [Wiki de memoria](/es/plugins/memory-wiki): repositorio de conocimiento compilado y herramientas nativas de wiki.
- [Dreaming](/es/concepts/dreaming): promoción en segundo plano desde la recuperación a corto plazo hasta la memoria a largo plazo.
- [Referencia de configuración de memoria](/es/reference/memory-config): todas las opciones de configuración.
- [Compaction](/es/concepts/compaction): cómo interactúa la compactación con la memoria.
- [Active Memory](/es/concepts/active-memory): memoria de subagentes para sesiones de chat interactivas.
