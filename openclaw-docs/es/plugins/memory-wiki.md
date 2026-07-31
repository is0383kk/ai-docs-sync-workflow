---
read_when:
    - Se desea disponer de conocimiento persistente más allá de simples notas en MEMORY.md
    - Está configurando el plugin memory-wiki incluido
    - Necesita bóvedas wiki separadas para los agentes en un mismo Gateway
    - Quieres entender wiki_search, wiki_get o el modo puente
summary: 'memory-wiki: repositorio de conocimiento compilado con procedencia, afirmaciones, paneles y modo puente'
title: Wiki de memoria
x-i18n:
    generated_at: "2026-07-26T05:13:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fda3c801ae39b529a3f1fcaf8791b6dcb1d8116ba2e73e99cca62dca6c64140a
    source_path: plugins/memory-wiki.md
    workflow: 16
---

`memory-wiki` es un plugin incluido que compila conocimiento duradero en una
wiki navegable: páginas deterministas, afirmaciones estructuradas con evidencias,
procedencia, paneles y resúmenes legibles por máquinas.

No sustituye al plugin de Active Memory. La recuperación, la promoción, la indexación y
Dreaming siguen siendo responsabilidad del backend de memoria que esté configurado
(`memory-core`, QMD, Honcho, etc.). `memory-wiki` se sitúa a su lado y compila
el conocimiento en una capa de wiki mantenida.

Active el plugin antes de usar su CLI, sus herramientas o su integración en tiempo de ejecución:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

| Capa                    | Responsabilidad                                                                      |
| ----------------------- | ------------------------------------------------------------------------------------ |
| Plugin de Active Memory | Recuperación, búsqueda semántica, promoción, Dreaming y tiempo de ejecución de memoria |
| `memory-wiki`      | Páginas de wiki compiladas, síntesis ricas en procedencia, paneles y búsqueda/obtención/aplicación en la wiki |

Regla práctica:

- `memory_search` para una única recuperación amplia en todos los corpus configurados
- `wiki_search` / `wiki_get` cuando se necesiten clasificación específica de la wiki, procedencia o una estructura de creencias a nivel de página
- `memory_search corpus=all` para abarcar ambas capas en una sola llamada, cuando el plugin de Active Memory admita la selección de corpus

Una configuración habitual que prioriza el entorno local: QMD como backend de Active Memory para la recuperación y
`memory-wiki` en modo `bridge` para páginas sintetizadas duraderas. Consulte el
ejemplo de QMD + modo puente en [Configuración](#configuration).

Si el modo puente informa de cero artefactos exportados, el plugin de Active Memory
no está exponiendo actualmente entradas públicas del puente. Ejecute primero `openclaw wiki doctor`
y, a continuación, confirme que el plugin de Active Memory admite artefactos públicos.

## Modos del almacén

- `isolated` (predeterminado): almacén propio, fuentes propias, sin dependencia del plugin de Active Memory. Úselo para un almacén de conocimiento seleccionado y autocontenido.
- `bridge`: lee artefactos públicos de memoria y registros de eventos del plugin de Active Memory mediante interfaces públicas del SDK de plugins. Úselo para compilar los artefactos exportados por el plugin de memoria sin acceder a sus componentes internos privados.
- `unsafe-local`: vía de escape explícita en la misma máquina para rutas locales privadas. Es intencionadamente experimental y no portátil; úselo solo cuando comprenda el límite de confianza y necesite específicamente acceso al sistema de archivos local que el modo puente no puede proporcionar.

El modo y el ámbito del almacén son opciones independientes:

- `vaultMode` elige de dónde proceden las entradas de la wiki.
- `vault.scope` elige si todos los agentes usan un almacén o si cada agente recibe un almacén secundario.

`vault.scope: "global"` es el valor predeterminado y conserva el comportamiento existente de un único almacén.
Use `vault.scope: "agent"` con el modo `isolated` o `bridge` cuando
los agentes no deban compartir páginas de la wiki, resúmenes compilados, resultados de búsqueda ni escrituras.
El ámbito de agente no puede combinarse con el modo `unsafe-local` porque esas rutas
privadas configuradas no son entradas propiedad del agente. La validación de la configuración rechaza esta
combinación.

El modo puente puede indexar, según el selector de configuración `bridge.*`:

- artefactos de memoria exportados (`indexMemoryRoot`)
- notas diarias (`indexDailyNotes`)
- informes de sueños (`indexDreamReports`)
- registros de eventos de memoria (`followMemoryEvents`)

Cuando el modo puente está activo y `bridge.readMemoryArtifacts` está habilitado,
`openclaw wiki status`, `openclaw wiki doctor` y `openclaw wiki bridge
import` se enrutan a través del Gateway en ejecución para que vean el mismo contexto del plugin de Active Memory
que la memoria del agente o del tiempo de ejecución. Si el puente está deshabilitado o las lecturas de
artefactos están desactivadas, esos comandos mantienen su comportamiento local/sin conexión.

## Diseño del almacén

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

El contenido gestionado permanece dentro de los bloques generados; los bloques de notas humanas se
conservan entre regeneraciones.

- `sources/`: material sin procesar importado y páginas respaldadas por el puente o por el modo local no seguro
- `entities/`: elementos duraderos, personas, sistemas, proyectos y objetos
- `concepts/`: ideas, abstracciones, patrones y políticas (también el destino de las importaciones de OKF)
- `syntheses/`: resúmenes compilados y recopilaciones mantenidas
- `reports/`: paneles generados

## Importaciones de Open Knowledge Format

```bash
openclaw wiki okf import ./bundles/ga4
```

Importe un paquete de Open Knowledge Format descomprimido en páginas de conceptos de la wiki. Es una buena
opción cuando un catálogo de datos, un rastreador de documentación o un agente de enriquecimiento ya
produce OKF: conserve OKF como artefacto portátil de intercambio y permita que `memory-wiki`
lo convierta en páginas de conceptos nativas de OpenClaw y resúmenes compilados.

- los archivos `.md` no reservados son documentos de conceptos
- cada concepto importado requiere un campo de frontmatter `type` no vacío; si falta `type`, se genera una advertencia `missing-type` y se omite el archivo
- los valores `type` desconocidos se aceptan como conceptos genéricos
- `index.md` y `log.md` están reservados y nunca se importan como conceptos
- los enlaces Markdown rotos o externos se dejan sin cambios

Las páginas importadas se aplanan bajo `concepts/` para que los flujos existentes de compilación, búsqueda, obtención y
paneles puedan verlas sin un segundo árbol de wiki. Cada página conserva el
ID original del concepto OKF, la ruta de origen, `type`, `resource`, `tags`, la marca temporal
y todo el frontmatter del productor. Los enlaces internos de OKF se reescriben para que apunten a las páginas
de conceptos generadas de la wiki y también emiten entradas estructuradas `relationships` con
`kind: okf-link`.

## Afirmaciones estructuradas y evidencias

Las páginas contienen frontmatter estructurado `claims`, no solo texto de formato libre. Cada
afirmación puede incluir `id`, `text`, `status`, `confidence`, `evidence[]` y
`updatedAt`. Cada entrada de evidencia puede incluir `kind`, `sourceId`, `path`,
`lines`, `weight`, `confidence`, `privacyTier`, `note` y `updatedAt`.

Esto hace que la wiki se comporte como una capa de creencias, no como un depósito pasivo de notas.
Las afirmaciones pueden rastrearse, puntuarse, cuestionarse y resolverse consultando las fuentes.

## Metadatos de entidades orientados a agentes

Las páginas de entidades contienen metadatos genéricos de enrutamiento que pueden usarse para personas, equipos,
sistemas, proyectos o cualquier otro tipo de entidad:

- `entityType`: por ejemplo, `person`, `team`, `system`, `project`
- `canonicalId`: clave de identidad estable entre alias e importaciones
- `aliases`: nombres, identificadores o etiquetas que se resuelven en la misma página
- `privacyTier`: cadena de formato libre; `public` se trata como sin revisión y cualquier otro valor (por ejemplo, `local-private`, `sensitive`, `confirm-before-use`) se marca en `reports/privacy-review.md`
- `bestUsedFor` / `notEnoughFor`: indicaciones compactas de enrutamiento
- `lastRefreshedAt`: marca temporal de actualización de la fuente, independiente de la hora de edición de la página
- `personCard`: tarjeta opcional de enrutamiento específica de una persona (identificadores, redes sociales, correos electrónicos, zona horaria, área, temas de consulta, temas que se deben evitar consultar, confianza y nivel de privacidad)
- `relationships`: aristas tipadas hacia páginas relacionadas (destino, tipo, peso, confianza, tipo de evidencia, nivel de privacidad y nota)

Para una wiki de personas, comience con `reports/person-agent-directory.md` y, a continuación, abra
la página de la persona con `wiki_get` antes de usar datos de contacto o hechos
inferidos.

<Accordion title="Ejemplo de página de entidad">
```yaml
pageType: entity
entityType: person
id: entity.example-person
canonicalId: maintainer.example-person
aliases:
  - Alex
  - example-handle
privacyTier: local-private
bestUsedFor:
  - Enrutamiento del ecosistema de ejemplo
notEnoughFor:
  - aprobación legal
lastRefreshedAt: "2026-04-29T00:00:00.000Z"
personCard:
  handles:
    - "@example-handle"
  socials:
    - "https://x.example/example-handle"
  emails:
    - alex@example.com
  timezone: America/Chicago
  lane: Ecosistema de ejemplo
  askFor:
    - Preguntas sobre el despliegue de ejemplo
  avoidAskingFor:
    - decisiones de facturación no relacionadas
  confidence: 0.8
  privacyTier: confirm-before-use
relationships:
  - targetId: entity.other-person
    targetTitle: Otra persona
    kind: collaborates-with
    confidence: 0.7
    evidenceKind: discrawl-stat
claims:
  - id: claim.example.routing
    text: Alex resulta útil para el enrutamiento del ecosistema de ejemplo.
    status: supported
    confidence: 0.9
    evidence:
      - kind: maintainer-whois
        sourceId: source.maintainers
        privacyTier: local-private
```
</Accordion>

## Pipeline de compilación

La compilación lee las páginas de la wiki, normaliza los resúmenes y conserva una instantánea orientada a máquinas
en el estado SQLite compartido del plugin de OpenClaw. El código de tiempo de ejecución usa la
instantánea del propietario gestionada por el ciclo de vida para cargar SQLite durante la preparación asíncrona de instrucciones;
el ensamblaje síncrono de instrucciones nunca extrae información de Markdown ni lee archivos de caché.
El resultado compilado también proporciona la indexación inicial de la wiki para búsquedas y obtenciones, la resolución de identificadores de
afirmaciones hasta las páginas propietarias, complementos compactos para instrucciones y la generación de
informes.

Las modificaciones de las fuentes y las restauraciones del almacén solo pasan a estar disponibles para las máquinas después de la siguiente
compilación. Al reiniciar o actualizar el ciclo de vida del plugin, se compara la publicación de compilación
encadenada causalmente del almacén con SQLite y se rechaza una instantánea de un
estado más reciente que se haya revertido. Un compilador iniciado antes de la reversión no puede
publicar sobre el predecesor restaurado. La preparación de instrucciones no consulta periódicamente el
almacén ni instala observadores de archivos.
Tras la cuarentena de una reversión, una compilación en el proceso en ejecución borra de inmediato el estado del propietario;
un proceso de compilación independiente requiere actualizar el ciclo de vida del plugin para que
el daemon pueda confirmar la nueva publicación duradera.
Las cachés compiladas pueden reconstruirse: las filas de caché anteriores a las épocas de publicación se
tratan como fallos de caché y se sustituyen en la siguiente compilación; no se migran.

## Paneles e informes de estado

Cuando `render.createDashboards` está habilitado, la compilación mantiene paneles en
`reports/`:

| Informe                             | Seguimiento                                              |
| ----------------------------------- | -------------------------------------------------------- |
| `reports/open-questions.md`                  | páginas con preguntas sin resolver                       |
| `reports/contradictions.md`                  | grupos de notas contradictorias                          |
| `reports/low-confidence.md`                  | páginas y afirmaciones de baja confianza                 |
| `reports/claim-health.md`                  | afirmaciones sin evidencias estructuradas                |
| `reports/stale-pages.md`                  | vigencia obsoleta o desconocida                          |
| `reports/person-agent-directory.md`                  | tarjetas de enrutamiento de personas/entidades           |
| `reports/relationship-graph.md`                  | aristas de relaciones estructuradas                      |
| `reports/provenance-coverage.md`                  | cobertura de clases de evidencias                        |
| `reports/privacy-review.md`                  | niveles de privacidad no públicos que requieren revisión antes de usarse |

## Búsqueda y recuperación

Dos backends de búsqueda:

- `shared`: usa el flujo compartido de búsqueda en memoria cuando está disponible
- `local`: busca localmente en la wiki

Tres corpus: `wiki`, `memory`, `all`.

- `wiki_search` / `wiki_get` usan resúmenes compilados como primera pasada cuando es posible
- los identificadores de afirmaciones se resuelven en la página propietaria
- las afirmaciones cuestionadas, obsoletas o vigentes influyen en la clasificación
- las etiquetas de procedencia se conservan en los resultados

Modos de búsqueda (parámetro `--mode` / `mode` de la herramienta):

| Modo              | Potencia                                                         |
| ----------------- | -------------------------------------------------------------- |
| `auto`            | valor predeterminado equilibrado                                               |
| `find-person`     | entidades similares a personas, alias, nombres de usuario, redes sociales, identificadores canónicos |
| `route-question`  | fichas de agentes, indicaciones sobre para qué preguntar o cuándo usarlos, contexto de relaciones |
| `source-evidence` | páginas de origen y metadatos de evidencia estructurados                  |
| `raw-claim`       | coincidencia de afirmaciones estructuradas; devuelve metadatos de afirmaciones y evidencias    |

Cuando un resultado coincide con una afirmación estructurada, `wiki_search` devuelve
`matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`,
`evidenceKinds` y `evidenceSourceIds` en la carga útil de detalles. La salida de texto
incluye líneas compactas de `Claim:` y `Evidence:` cuando están disponibles.

## Herramientas del agente

| Herramienta          | Finalidad                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wiki_status` | modo y ámbito actuales del almacén, agente resuelto, estado, disponibilidad de la CLI de Obsidian                                                                               |
| `wiki_search` | busca páginas de la wiki y, cuando está configurado, el corpus de memoria compartida; acepta `mode` para buscar personas, enrutar preguntas, obtener evidencia de origen o examinar afirmaciones sin procesar |
| `wiki_get`    | lee una página de la wiki por identificador/ruta y recurre al corpus de memoria compartida cuando la búsqueda compartida está activada y la consulta no encuentra resultados                                     |
| `wiki_apply`  | mutaciones específicas de síntesis/metadatos sin modificar páginas libremente                                                                                             |
| `wiki_lint`   | comprobaciones estructurales, lagunas de procedencia, contradicciones, preguntas abiertas                                                                                            |

El Plugin también registra un complemento no exclusivo del corpus de memoria, por lo que las
funciones compartidas `memory_search` y `memory_get` pueden acceder a la wiki cuando el Plugin de Active Memory
admite la selección de corpus.

## Comportamiento del prompt y del contexto

Cuando `context.includeCompiledDigestPrompt` está activado, las secciones de memoria del prompt
añaden una instantánea compilada y compacta del estado del Plugin: solo las páginas
principales, solo las afirmaciones principales, el número de contradicciones, el número de preguntas y
calificadores de confianza/actualidad. Es opcional porque cambia la estructura del prompt; es relevante
principalmente para motores de contexto o el ensamblado de prompts que consumen explícitamente
complementos de memoria.

## Configuración

Coloque la configuración en `plugins.entries.memory-wiki.config`:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            scope: "global",
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          unsafeLocal: {
            allowPrivateMemoryCoreAccess: false,
            paths: [],
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

Opciones clave:

| Clave                                        | Valores / valor predeterminado                               | Notas                                                                         |
| ------------------------------------------ | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `vaultMode`                                | `isolated` (predeterminado), `bridge`, `unsafe-local` | elige el comportamiento de entrada e integración                                        |
| `vault.scope`                              | `global` (predeterminado), `agent`                    | un almacén compartido o un almacén secundario por agente                                 |
| `vault.path`                               | valor predeterminado global `~/.openclaw/wiki/main`         | almacén exacto globalmente; el directorio principal del ámbito de agente tiene como valor predeterminado `~/.openclaw/wiki`       |
| `vault.renderMode`                         | `native` (predeterminado), `obsidian`                 |                                                                               |
| `bridge.readMemoryArtifacts`               | valor predeterminado `true`                                 | importa artefactos públicos del Plugin de Active Memory activo                                  |
| `bridge.followMemoryEvents`                | valor predeterminado `true`                                 | incluye registros de eventos en el modo puente                                             |
| `unsafeLocal.allowPrivateMemoryCoreAccess` | valor predeterminado `false`                                | necesario para ejecutar importaciones de `unsafe-local`                                        |
| `unsafeLocal.paths`                        | valor predeterminado `[]`                                   | rutas locales explícitas que se importarán en el modo `unsafe-local`                         |
| `search.backend`                           | `shared` (predeterminado), `local`                    |                                                                               |
| `search.corpus`                            | `wiki` (predeterminado), `memory`, `all`              |                                                                               |
| `context.includeCompiledDigestPrompt`      | valor predeterminado `false`                                | añade la instantánea compacta del resumen del agente seleccionado a las secciones de memoria del prompt |
| `render.createBacklinks`                   | valor predeterminado `true`                                 | genera bloques relacionados deterministas                                         |
| `render.createDashboards`                  | valor predeterminado `true`                                 | genera páginas de panel                                                      |

### Almacenes por agente

Establezca `vault.scope` en `agent` para proporcionar una wiki independiente a cada agente configurado.
En este ámbito, `vault.path` es un directorio principal y OpenClaw añade el
identificador normalizado del agente:

```json5
{
  agents: {
    list: [{ id: "support" }, { id: "marketing" }],
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
          },
        },
      },
    },
  },
}
```

Esto se resuelve como `~/.openclaw/wiki/support` y
`~/.openclaw/wiki/marketing`. Si se omite `vault.path` en el ámbito del agente, el
directorio principal tiene como valor predeterminado `~/.openclaw/wiki`. Por tanto, el agente `main` predeterminado conserva
la ruta `~/.openclaw/wiki/main` existente.

Las herramientas del agente, los resúmenes compilados del prompt y el complemento de la wiki expuesto mediante
`memory_search` / `memory_get` resuelven el almacén a partir del contexto del agente activo.
Para las llamadas de la CLI y el Gateway en una configuración con varios agentes configurados, proporcione
el agente explícitamente mediante `openclaw wiki --agent <agentId> ...` o el campo `agentId` de la
solicitud del Gateway. Un único agente configurado sigue siendo el predeterminado cuando no se
proporciona ningún identificador.

En el modo puente, las importaciones con ámbito de agente aceptan un artefacto público de memoria solo cuando
su `agentIds` incluye al agente seleccionado. Se omiten los artefactos pertenecientes a otro agente,
sin metadatos de propiedad o con un propietario desconocido. El ámbito global
mantiene el comportamiento existente de artefactos compartidos.

<Warning>
Cambiar `vault.scope` no copia ni divide un almacén existente. En el ámbito de agente,
un `vault.path` configurado explícitamente se convierte en un directorio principal, por lo que se deben mover o
importar deliberadamente las páginas existentes antes de cambiar los agentes de producción. Haga primero una copia de seguridad
del almacén.

Los almacenes por agente constituyen un límite de conocimiento dentro del mismo proceso, no un límite de seguridad
del sistema operativo. Los Plugins y las herramientas sin aislamiento que tengan acceso al sistema de archivos del host
pueden seguir leyendo el directorio de otro agente. Use el [aislamiento](/es/gateway/sandboxing) o
[perfiles de Gateway independientes](/es/gateway/multiple-gateways) cuando los agentes no confíen
entre sí.
</Warning>

### Ejemplo: QMD + modo puente

Use esta configuración cuando desee QMD para la recuperación y `memory-wiki` como capa de
conocimiento mantenida. Cada capa conserva un enfoque específico: QMD mantiene disponibles para búsqueda las notas sin procesar, las exportaciones
de sesiones y las colecciones adicionales, mientras que `memory-wiki` compila
entidades estables, afirmaciones, paneles y páginas de origen.

```json5
{
  memory: {
    backend: "qmd",
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          search: {
            backend: "shared",
            corpus: "all",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
        },
      },
    },
  },
}
```

Esto mantiene QMD a cargo de la recuperación de Active Memory, `memory-wiki` centrado en
páginas y paneles compilados, y la estructura del prompt sin cambios hasta que se
activen intencionadamente los resúmenes compilados del prompt.

## CLI

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

Consulte [CLI: wiki](/es/cli/wiki) para obtener la referencia completa de comandos, incluidos
`wiki okf import`, `wiki apply metadata`, `wiki unsafe-local import`,
`wiki chatgpt import` / `wiki chatgpt rollback` y el conjunto completo de subcomandos
`wiki obsidian`.

## Compatibilidad con Obsidian

Cuando `vault.renderMode` es `obsidian`, el Plugin escribe Markdown compatible con Obsidian
y puede usar opcionalmente la CLI oficial de `obsidian` para consultar el estado,
buscar en el almacén, abrir una página, invocar un comando y acceder a la
nota diaria. Esto es opcional; la wiki sigue funcionando en modo nativo sin
Obsidian.

Los almacenes con ámbito de agente pueden seguir usando Markdown compatible con Obsidian, pero la validación de la
configuración rechaza `obsidian.useOfficialCli: true` con `vault.scope: "agent"`.
La configuración actual de `obsidian.vaultName` es global y no puede seleccionar un almacén de
Obsidian distinto para cada agente. En su lugar, use las herramientas de la wiki y las operaciones de la CLI,
o mantenga una wiki operada mediante Obsidian en el ámbito global.

## Flujo de trabajo recomendado

<Steps>
<Step title="Mantener el plugin de memoria activa para la recuperación">
La recuperación, la promoción y Dreaming siguen siendo responsabilidad del backend de memoria configurado.
</Step>
<Step title="Activar memory-wiki">
Comience con el modo `isolated`, a menos que quiera explícitamente el modo puente.
</Step>
<Step title="Usar wiki_search / wiki_get cuando la procedencia sea importante">
Prefiera estas opciones en lugar de `memory_search` cuando quiera una clasificación específica de la wiki o una estructura de creencias a nivel de página.
</Step>
<Step title="Usar wiki_apply para síntesis acotadas o actualizaciones de metadatos">
Evite editar manualmente los bloques generados y administrados.
</Step>
<Step title="Ejecutar wiki_lint después de cambios significativos">
Detecta contradicciones, preguntas abiertas y lagunas de procedencia.
</Step>
<Step title="Activar los paneles para mostrar información obsoleta o contradicciones">
Establezca `render.createDashboards: true` (valor predeterminado).
</Step>
</Steps>

## Documentación relacionada

- [Descripción general de la memoria](/es/concepts/memory)
- [CLI: memoria](/es/cli/memory)
- [CLI: wiki](/es/cli/wiki)
- [Descripción general del SDK de plugins](/es/plugins/sdk-overview)
