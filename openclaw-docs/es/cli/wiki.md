---
read_when:
    - Quiere usar la CLI memory-wiki
    - Está documentando o modificando `openclaw wiki`
summary: Referencia de la CLI para `openclaw wiki` (estado del repositorio de memory-wiki, búsqueda, compilación, análisis, aplicación, puente, importación de ChatGPT y utilidades de Obsidian)
title: Wiki
x-i18n:
    generated_at: "2026-07-26T05:36:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

Inspeccione y mantenga el almacén `memory-wiki`. Lo proporciona el Plugin opcional incluido `memory-wiki`. Habilítelo antes del primer uso:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

Relacionado: [Plugin Memory Wiki](/es/plugins/memory-wiki), [Descripción general de la memoria](/es/concepts/memory), [CLI: memoria](/es/cli/memory)

## Comandos comunes

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "who should I ask about Teams?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Summary" \
  --body "Short synthesis body" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Still active?"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## Selección del agente

Cuando `plugins.entries.memory-wiki.config.vault.scope` sea `agent`, seleccione el
almacén con la opción de nivel superior `--agent <id>`:

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "refund policy"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

En una configuración con varios agentes configurados, se requiere `--agent` para las
operaciones de la CLI, de modo que un comando no pueda leer ni escribir en un almacén predeterminado arbitrario. Si
solo hay un agente configurado, ese agente continúa siendo el predeterminado. Los identificadores de agente desconocidos
producen un error antes de que comience la operación del almacén. La opción no cambia la
ruta seleccionada cuando `vault.scope` es `global`.

Los clientes del Gateway siguen la misma regla: pase `agentId` en las solicitudes `wiki.*`
respaldadas por el almacén en una configuración multiagente con ámbito de agente. Un identificador ausente o desconocido es un
error. Los turnos del agente, las herramientas de wiki, los suplementos del corpus de memoria y los resúmenes compilados
para el prompt ya incluyen el contexto del agente de ejecución activo.

## Comandos

### `wiki status`

Muestra el modo y el ámbito del almacén, el agente resuelto, el estado y la disponibilidad de la CLI de Obsidian. Utilice esto primero para comprobar si el almacén previsto está inicializado, si el modo puente funciona correctamente o si la integración con Obsidian está disponible.

Cuando el modo puente está activo y configurado para leer artefactos de memoria, este comando consulta el Gateway en ejecución para ver el mismo contexto activo del Plugin de memoria que la memoria del agente o del entorno de ejecución.

### `wiki doctor`

Ejecuta comprobaciones del estado de la wiki e informa de correcciones aplicables. Finaliza con un código distinto de cero cuando el estado no es correcto.

Cuando el modo puente está activo y configurado para leer artefactos de memoria, este comando consulta el Gateway en ejecución antes de generar el informe. Las importaciones del puente deshabilitadas y las configuraciones del puente que no leen artefactos de memoria permanecen en modo local y sin conexión.

Problemas habituales:

- modo puente habilitado sin artefactos de memoria públicos
- diseño del almacén no válido o ausente
- falta la CLI externa de Obsidian cuando se espera el modo Obsidian

### `wiki init`

Crea el diseño del almacén de la wiki y las páginas iniciales, incluidos los índices de nivel superior y los directorios de caché.

### `wiki ingest <path>`

Importa un archivo Markdown o de texto local en la carpeta `sources/` de la wiki como página de origen. `<path>` debe ser una ruta de archivo local; actualmente no se admite la ingesta desde URL. Rechaza los archivos binarios.

Las páginas de origen importadas incluyen frontmatter de procedencia (`sourceType: local-file`, `sourcePath`, `ingestedAt`). La ingesta siempre vuelve a compilar el almacén después.

Opciones: `--title <title>` sustituye el título del origen (valor predeterminado: derivado del nombre del archivo).

### `wiki okf import <path>`

Importa un paquete de Open Knowledge Format descomprimido en páginas de conceptos de la wiki.

El importador lee todos los documentos de conceptos `.md` no reservados del árbol de directorios de OKF, requiere un campo `type` no vacío y trata los valores `type` de OKF desconocidos como conceptos genéricos. Los archivos reservados `index.md` y `log.md` de OKF no se importan como conceptos.

Las páginas importadas se aplanan bajo `concepts/` para que los flujos existentes de compilación, búsqueda, obtención, resumen y panel de la wiki las detecten de inmediato. El identificador original del concepto de OKF, `type`, `resource`, `tags`, la marca de tiempo, la ruta de origen y el frontmatter completo se conservan en el frontmatter de la página. Los enlaces Markdown internos de OKF se reescriben para dirigir a las páginas generadas de la wiki; los enlaces rotos o externos permanecen sin cambios. La importación siempre vuelve a compilar el almacén después.

Ejemplos:

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery Table" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

Reconstruye índices, bloques relacionados, paneles y la instantánea compilada de consultas y prompts. La instantánea se conserva en el estado SQLite compartido del Plugin de OpenClaw y se mantiene en memoria para la proyección síncrona de prompts; no crea archivos de caché en el almacén.

Si `render.createDashboards` está habilitado, la compilación también actualiza las páginas de informes.

### `wiki lint`

Analiza el almacén y escribe un informe que abarca:

- problemas estructurales (enlaces rotos, identificadores ausentes o duplicados, tipo de página o título ausentes, frontmatter no válido)
- carencias de procedencia (identificadores de origen ausentes, procedencia de importación ausente)
- contradicciones (contradicciones marcadas, afirmaciones incompatibles)
- preguntas abiertas
- páginas y afirmaciones de baja confianza
- páginas y afirmaciones obsoletas

Ejecute esto después de realizar actualizaciones importantes en la wiki.

### `wiki search <query>`

Busca contenido de la wiki. El comportamiento depende de la configuración:

- `search.backend`: `shared` o `local`
- `search.corpus`: `wiki`, `memory` o `all`
- `--mode`: `auto`, `find-person`, `route-question`, `source-evidence` o `raw-claim`

Utilice `wiki search` para la clasificación y la procedencia específicas de la wiki. Para una única búsqueda amplia en la memoria compartida, utilice preferentemente `openclaw memory search` cuando el Plugin de memoria activo exponga la búsqueda compartida.

Modos de búsqueda:

- `find-person`: alias, identificadores, redes sociales, identificadores canónicos y páginas de personas
- `route-question`: indicaciones sobre a quién preguntar o para qué se utiliza mejor y contexto de relaciones
- `source-evidence`: páginas de origen y campos de evidencia estructurados
- `raw-claim`: texto de afirmaciones estructurado con metadatos de afirmación y evidencia

Ejemplos:

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "who knows Teams rollout?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "strong route Teams" --mode raw-claim --json
```

La salida de texto incluye líneas `Claim:` y `Evidence:` cuando un resultado coincide con una afirmación estructurada. La salida JSON también expone `matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`, `evidenceKinds` y `evidenceSourceIds` para el análisis detallado por parte del agente.

### `wiki get <lookup>`

Lee una página de la wiki por identificador o ruta relativa.

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

Aplica modificaciones específicas sin manipular libremente la página:

- `apply synthesis <title>`: crea o actualiza una página de síntesis con un cuerpo de resumen gestionado
- `apply metadata <lookup>`: actualiza los metadatos de una página existente

Ambas aceptan `--source-id`, `--contradiction`, `--question` (cada una se puede repetir), `--confidence <n>` (0-1) y `--status <status>`. `apply metadata` también acepta `--clear-confidence` para eliminar un valor de confianza almacenado. Esta es la forma admitida de desarrollar las páginas de la wiki para que los bloques generados y gestionados permanezcan intactos.

### `wiki bridge import`

Importa artefactos de memoria públicos del Plugin de memoria activo en páginas de origen respaldadas por el puente. Utilice esto en el modo `bridge` para incorporar al almacén de la wiki los artefactos de memoria exportados más recientes.

Para las lecturas activas de artefactos del puente, la CLI dirige la importación a través de RPC del Gateway para utilizar el contexto del Plugin de memoria del entorno de ejecución. Si las importaciones del puente están deshabilitadas o las lecturas de artefactos están desactivadas, el comando mantiene el comportamiento local y sin conexión de cero importaciones. La actualización del índice posterior a la importación está condicionada por `ingest.autoCompile`.

### `wiki unsafe-local import`

Importa desde rutas locales configuradas explícitamente (`unsafeLocal.paths`) en el modo `unsafe-local`. Es deliberadamente experimental y solo para el mismo equipo. La actualización del índice posterior a la importación está condicionada por `ingest.autoCompile`.

### `wiki chatgpt import`

Importa una exportación de ChatGPT en páginas de origen de borrador de la wiki.

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| Opción             | Valor predeterminado | Descripción                                                         |
| ------------------ | --------------------- | ------------------------------------------------------------------- |
| `--export <path>` | (obligatorio)         | Directorio de exportación de ChatGPT o ruta `conversations.json`.     |
| `--dry-run` | `false`    | Previsualiza las cantidades creadas, actualizadas y omitidas sin escribir páginas. |

Una importación que no sea de prueba y que cambie alguna página registra un identificador de ejecución de importación, impreso en el resumen y necesario para la reversión.

### `wiki chatgpt rollback <run-id>`

Revierte una ejecución de importación de ChatGPT aplicada anteriormente, elimina las páginas que creó y restaura las páginas que sobrescribió. No realiza ninguna operación (e informa de `alreadyRolledBack`) si la ejecución ya se había revertido.

### `wiki obsidian ...`

Comandos auxiliares de Obsidian para almacenes que se ejecutan en un modo compatible con Obsidian: `status`, `search`, `open`, `command`, `daily`. Estos requieren la CLI oficial `obsidian` en `PATH` cuando `obsidian.useOfficialCli` está habilitado.

La validación de la configuración rechaza `obsidian.useOfficialCli: true` cuando
`vault.scope` es `agent` porque `obsidian.vaultName` es una configuración global,
no una asignación por agente. La representación de Markdown compatible con Obsidian continúa
disponible.

## Orientación de uso práctico

- Utilice `wiki search` + `wiki get` cuando la procedencia y la identidad de la página sean importantes.
- Utilice `wiki apply` en lugar de editar manualmente las secciones generadas y gestionadas.
- Utilice `wiki lint` antes de confiar en contenido contradictorio o de baja confianza.
- Utilice `wiki compile` después de importaciones masivas o cambios en los orígenes cuando necesite inmediatamente paneles actualizados y resúmenes compilados.
- Utilice `wiki okf import` cuando un catálogo de datos, una exportación de documentación o un Pipeline de enriquecimiento de agentes ya genere paquetes Markdown de OKF.
- Utilice `wiki bridge import` cuando el modo puente dependa de artefactos de memoria recién exportados.

## Opciones de configuración relacionadas

El comportamiento de `openclaw wiki` está determinado por:

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

Consulte [Plugin Memory Wiki](/es/plugins/memory-wiki) para conocer el modelo de configuración completo.

## Temas relacionados

- [Referencia de la CLI](/es/cli)
- [Wiki de memoria](/es/plugins/memory-wiki)
