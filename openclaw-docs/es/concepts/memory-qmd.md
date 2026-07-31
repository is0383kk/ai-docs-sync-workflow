---
read_when:
    - Quieres configurar QMD como backend de memoria
    - Se necesitan funciones avanzadas de memoria, como la reordenación por relevancia o rutas indexadas adicionales.
summary: Servicio auxiliar de búsqueda local primero con BM25, vectores, reranking y expansión de consultas
title: Motor de memoria QMD
x-i18n:
    generated_at: "2026-07-26T05:05:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0e54dc9a18d834036e4c79d6b7bdecb268a29976d9f30ea6e82a56ca5d71fda
    source_path: concepts/memory-qmd.md
    workflow: 16
---

[QMD](https://github.com/tobi/qmd) es un servicio auxiliar de búsqueda con enfoque local que se ejecuta
junto con OpenClaw. Combina BM25, búsqueda vectorial y reclasificación en un único
binario, y puede indexar contenido más allá de los archivos de memoria del espacio de trabajo.

## Qué aporta respecto al motor integrado

- **Reclasificación y expansión de consultas** para mejorar la exhaustividad.
- **Indexación de directorios adicionales**: documentación de proyectos, notas de equipo y cualquier contenido del disco.
- **Indexación de transcripciones de sesiones**: permite recordar conversaciones anteriores.
- **Totalmente local**: se ejecuta con el Plugin oficial del proveedor llama.cpp y
  descarga automáticamente los modelos GGUF.
- **Alternativa automática**: si QMD no está disponible, OpenClaw recurre al
  motor integrado sin interrupciones.

## Primeros pasos

### Requisitos previos

- Instale QMD: `npm install -g @tobilu/qmd` o `bun install -g @tobilu/qmd`
- Una compilación de SQLite que permita extensiones (`brew install sqlite` en macOS).
- QMD debe estar en el `PATH` del Gateway.
- macOS y Linux funcionan sin configuración adicional. En Windows, se recomienda usar WSL2.

### Activación

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw crea un directorio principal autónomo de QMD en
`~/.openclaw/agents/<agentId>/qmd/` y gestiona automáticamente el ciclo de vida
del servicio auxiliar: se encarga de las colecciones, las actualizaciones y las ejecuciones de incrustación.
Da preferencia a los formatos actuales de consultas de colecciones y MCP de QMD, pero recurre a
marcas alternativas para patrones de colecciones y nombres antiguos de herramientas MCP cuando es necesario.
La conciliación durante el inicio también vuelve a crear las colecciones gestionadas obsoletas con sus
patrones canónicos cuando sigue presente una colección antigua de QMD con el mismo nombre.

## Cómo funciona el servicio auxiliar

- OpenClaw crea colecciones a partir de los archivos de memoria del espacio de trabajo y de los
  `memory.qmd.paths` configurados. El adaptador de QMD controla las heurísticas de actualización,
  incrustación, espera antirrebote y tiempo de espera; estas no son opciones configurables por el usuario.
- QMD continúa controlando su `index.sqlite`, la configuración YAML de las colecciones y las descargas
  de modelos en el directorio principal de QMD de cada agente; estos son artefactos de una herramienta externa,
  no tablas de estado de OpenClaw. La coordinación controlada por OpenClaw reside únicamente en SQLite:
  un arrendamiento compartido limita el trabajo de incrustación entre agentes, mientras que un arrendamiento en cada
  base de datos de agente serializa las escrituras de colecciones, actualizaciones e incrustaciones de ese agente.
  El entorno de ejecución ya no crea archivos auxiliares de bloqueo de QMD. `openclaw doctor --fix`
  elimina los archivos auxiliares retirados solo después de comprobar que el proceso antiguo que los controlaba está obsoleto.
  Las actualizaciones realizan una transición limpia: detenga y reinicie todos los procesos de OpenClaw que
  compartan el directorio de estado antes de usar la nueva versión. No se admite la coexistencia de procesos de escritura
  antiguos y nuevos de QMD; el entorno de ejecución no bloquea simultáneamente de forma intencionada los archivos auxiliares
  retirados.
- La colección predeterminada del espacio de trabajo realiza el seguimiento de `MEMORY.md` y del árbol
  `memory/`. El archivo `memory.md` en minúsculas no se indexa como archivo de memoria raíz.
- El analizador de QMD ignora las rutas ocultas y los directorios habituales de dependencias y compilación,
  como `.git`, `.cache`, `node_modules`, `vendor`, `dist` y
  `build`. Al iniciarse, el Gateway mantiene QMD en modo diferido; el gestor se inicializa cuando se
  usa la memoria por primera vez.
- Las búsquedas utilizan el `searchMode` configurado (valor predeterminado: `search`; también admite
  `vsearch` y `query`). `search` solo usa BM25, por lo que OpenClaw omite las comprobaciones
  de preparación de vectores semánticos y el mantenimiento de incrustaciones en ese modo. Si un modo
  falla, OpenClaw vuelve a intentarlo con `qmd query`.
- Cuando `searchMode` sea `query`, establezca `memory.qmd.rerank` en `false` para usar
  la ruta de consulta híbrida de QMD sin el reclasificador (requiere QMD 2.1 o posterior).
  OpenClaw pasa `--no-rerank` a la ruta directa de la CLI de QMD y
  `rerank: false` a la herramienta de consultas MCP de QMD.
- Con las versiones de QMD que anuncian filtros para varias colecciones, OpenClaw agrupa
  las colecciones con la misma fuente en una sola invocación de búsqueda de QMD. Las versiones anteriores de QMD
  mantienen la alternativa compatible de una invocación por colección.
- Si QMD falla por completo, OpenClaw recurre al motor SQLite integrado.
  Los intentos repetidos durante los turnos del chat realizan una breve espera progresiva tras un fallo de apertura para que
  un binario ausente o una dependencia defectuosa del servicio auxiliar no genere una avalancha de reintentos;
  `openclaw memory status` y las comprobaciones únicas de la CLI siguen verificando QMD
  directamente.

<Info>
La primera búsqueda puede ser lenta: QMD descarga automáticamente modelos GGUF (~2 GB) para
la reclasificación y la expansión de consultas durante la primera ejecución de `qmd query`.
</Info>

## Rendimiento y compatibilidad de la búsqueda

OpenClaw mantiene la ruta de búsqueda de QMD compatible tanto con las instalaciones actuales
como con las antiguas de QMD.

Al iniciarse, OpenClaw comprueba una vez por gestor el texto de ayuda de QMD instalado. Si
el binario anuncia que admite varios filtros de colecciones, OpenClaw
busca en todas las colecciones con la misma fuente mediante un solo comando:

```bash
qmd search "router notes" --json -n 10 -c memory-root-main -c memory-dir-main
```

Esto evita iniciar un subproceso de QMD por cada colección de memoria persistente.
Las colecciones de transcripciones de sesiones permanecen en su propio grupo de fuentes, por lo que las búsquedas
combinadas de `memory` y `sessions` siguen proporcionando al diversificador resultados de
ambas fuentes.

Las compilaciones antiguas de QMD solo aceptan un filtro de colección. Cuando OpenClaw detecta una
de esas compilaciones, mantiene la ruta de compatibilidad y busca en cada colección
por separado antes de combinar y eliminar los resultados duplicados.

Para inspeccionar manualmente el contrato instalado, ejecute:

```bash
qmd --help | grep -i collection
```

La ayuda actual de QMD menciona la selección de una o más colecciones. La ayuda antigua
suele describir una sola colección.

## Sustitución de modelos

Las variables de entorno de los modelos de QMD se transfieren sin cambios desde el proceso del Gateway,
por lo que es posible ajustar QMD globalmente sin añadir nueva configuración de OpenClaw:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

Después de cambiar el modelo de incrustación, vuelva a ejecutar las incrustaciones para que el índice coincida con el
nuevo espacio vectorial.

## Indexación de rutas adicionales

Indique a QMD directorios adicionales para que se puedan buscar:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Los fragmentos de rutas adicionales aparecen como `qmd/<collection>/<relative-path>` en los
resultados de búsqueda. `memory_get` reconoce este prefijo y lee desde la
raíz correcta de la colección.

## Indexación de transcripciones de sesiones

Active la indexación de sesiones para recordar conversaciones anteriores. QMD necesita tanto la
fuente general de sesiones `memory.search` como el exportador de transcripciones de QMD:

```json5
{
  memory: {
    backend: "qmd",
    search: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"],
    },
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Las transcripciones se exportan como turnos saneados de Usuario/Asistente a una colección específica de QMD
en `~/.openclaw/agents/<id>/qmd/sessions/`. Configurar únicamente
`sources: ["sessions"]` no exporta las transcripciones a QMD; active también
`rememberAcrossConversations` o la exportación explícita de sesiones de QMD.

Las coincidencias de sesiones siguen filtrándose mediante
[`tools.sessions.visibility`](/es/gateway/config-tools#toolssessions). La
visibilidad predeterminada `tree` incluye la sesión actual, las sesiones que esta ha iniciado
y las sesiones de grupo del mismo agente supervisadas mediante el reconocimiento ambiental de grupos. Con
`session.dmScope: "main"`, los usuarios de una configuración de mensajes directos multiusuario comparten la sesión
principal y pueden recordar contenido de los grupos que esta supervisa. Use un
`dmScope` por interlocutor para aislar los mensajes directos o establezca la visibilidad en `"self"` para excluir
las lecturas ambientales de sesiones supervisadas. Las demás sesiones no relacionadas del mismo agente siguen requiriendo
la visibilidad `"agent"`.

## Ámbito de búsqueda

De forma predeterminada, los resultados de búsqueda de QMD solo se muestran en sesiones directas, no
en chats de grupo ni de canales. Configure `memory.qmd.scope` para cambiar este comportamiento:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

El fragmento anterior es la regla predeterminada real. Cuando el ámbito deniega una búsqueda,
OpenClaw registra una advertencia con el canal y el tipo de chat derivados para facilitar
la depuración de resultados vacíos.

## Citas

Cuando `memory.citations` es `auto` o `on`, se añade a los fragmentos de búsqueda un
pie `Source: <path>#L<line>` (o `#L<start>-L<end>`). En el modo `auto`,
el pie solo se añade en las sesiones de chat directo. Establezca
`memory.citations = "off"` para omitir el pie y seguir pasando internamente la ruta
al agente.

## Cuándo usarlo

Elija QMD cuando necesite:

- Reclasificación para obtener resultados de mayor calidad.
- Buscar documentación o notas de proyectos fuera del espacio de trabajo.
- Recordar conversaciones de sesiones anteriores.
- Búsqueda totalmente local sin claves de API.

Para configuraciones más sencillas, el [motor integrado](/es/concepts/memory-builtin) funciona bien
sin dependencias adicionales.

## Solución de problemas

**¿No se encuentra QMD?** Compruebe que el binario esté en el `PATH` del Gateway. Si OpenClaw
se ejecuta como servicio, cree un enlace simbólico:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

Si `qmd --version` funciona en el shell, pero OpenClaw sigue informando
`spawn qmd ENOENT`, es probable que el proceso del Gateway tenga un `PATH` diferente del
shell interactivo. Fije explícitamente el binario:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      command: "/absolute/path/to/qmd",
    },
  },
}
```

Use `command -v qmd` en el entorno donde esté instalado QMD y vuelva a comprobarlo
con `openclaw memory status --deep`.

**¿La primera búsqueda es muy lenta?** QMD descarga los modelos GGUF la primera vez que se usa. Realice una
preparación previa con `qmd query "test"` usando los mismos directorios XDG que utiliza OpenClaw.

**¿Se ejecutan muchos subprocesos de QMD durante la búsqueda?** Actualice QMD si es posible. OpenClaw
usa un solo proceso para las búsquedas en varias colecciones con la misma fuente únicamente cuando el
QMD instalado anuncia que admite varios filtros `-c`; de lo contrario,
mantiene la alternativa anterior de una invocación por colección para garantizar la corrección.

**¿QMD configurado solo para BM25 sigue intentando compilar llama.cpp?** Establezca
`memory.qmd.searchMode = "search"`. OpenClaw trata ese modo como
exclusivamente léxico, omite las comprobaciones de estado vectorial y el mantenimiento de incrustaciones de QMD, y
deja las comprobaciones de preparación semántica a las configuraciones `vsearch` o `query`.

**¿La búsqueda agota el tiempo de espera?** Aumente `memory.qmd.limits.timeoutMs` (valor predeterminado: 4000ms).
Establézcalo en un valor superior, por ejemplo `120000`, para hardware más lento. Este límite se aplica a
los propios comandos de búsqueda de QMD durante las llamadas `memory_search` del agente; la configuración, la sincronización,
la alternativa integrada y el trabajo del corpus complementario conservan sus propios plazos más breves.

**¿Se obtienen resultados vacíos en chats de grupo o de canales?** Esto es lo esperado con el
valor predeterminado `memory.qmd.scope`, que solo permite sesiones directas. Añada una
regla `allow` para los tipos de chat `group` o `channel` si desea obtener resultados de QMD
en ellos.

**¿La búsqueda en la memoria raíz se ha vuelto demasiado amplia de repente?** Reinicie el Gateway o espere
a la siguiente conciliación durante el inicio. OpenClaw vuelve a crear las colecciones gestionadas obsoletas
con los patrones canónicos `MEMORY.md` y `memory/` cuando
detecta un conflicto con el mismo nombre.

**¿Los repositorios temporales visibles desde el espacio de trabajo provocan `ENAMETOOLONG` o errores de indexación?**
El recorrido de QMD sigue el analizador subyacente de QMD en lugar de las
reglas de enlaces simbólicos del motor integrado de OpenClaw. Mantenga las copias de trabajo temporales de monorrepositorios en
directorios ocultos como `.tmp/` o fuera de las raíces indexadas por QMD hasta que QMD proporcione
un recorrido seguro frente a ciclos o controles de exclusión explícitos.

## Configuración

Para consultar toda la superficie de configuración (`memory.qmd.*`), los modos de búsqueda, los intervalos de actualización,
las reglas de ámbito y todas las demás opciones, consulte la
[referencia de configuración de memoria](/es/reference/memory-config).

## Contenido relacionado

- [Descripción general de la memoria](/es/concepts/memory)
- [Motor de memoria integrado](/es/concepts/memory-builtin)
- [Memoria de Honcho](/es/concepts/memory-honcho)
