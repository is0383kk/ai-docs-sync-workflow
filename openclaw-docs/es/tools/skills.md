---
read_when:
    - Añadir o modificar Skills
    - Cambio de las restricciones, las listas de permitidos o las reglas de carga de Skills
    - Comprender la precedencia de Skills y el comportamiento de las instantáneas
sidebarTitle: Skills
summary: Las Skills enseñan a su agente a usar herramientas. Descubra cómo se cargan, cómo funciona la precedencia y cómo configurar controles de acceso, listas de permitidos y la inyección de variables de entorno.
title: Skills
x-i18n:
    generated_at: "2026-07-26T05:02:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills son archivos de instrucciones en Markdown que enseñan al agente cómo y cuándo usar
herramientas. Cada skill reside en un directorio que contiene un archivo `SKILL.md` con frontmatter
YAML y un cuerpo en Markdown. OpenClaw carga las skills incluidas junto con cualquier
sobrescritura local y las filtra durante la carga según el entorno, la configuración y
la presencia de binarios.

<CardGroup cols={2}>
  <Card title="Crear skills" href="/es/tools/creating-skills" icon="hammer">
    Cree y pruebe una skill personalizada desde cero.
  </Card>
  <Card title="Taller de skills" href="/es/tools/skill-workshop" icon="flask">
    Revise y apruebe propuestas de skills redactadas por el agente.
  </Card>
  <Card title="Configuración de skills" href="/es/tools/skills-config" icon="gear">
    Esquema de configuración completo de `skills.*` y listas de permitidas del agente.
  </Card>
  <Card title="ClawHub" href="/es/clawhub" icon="cloud">
    Explore e instale skills de la comunidad.
  </Card>
</CardGroup>

## Orden de carga

OpenClaw carga desde estas fuentes, con la **precedencia más alta primero**. Cuando el mismo
nombre de skill aparece en varios lugares, prevalece la fuente con mayor prioridad.

| Prioridad      | Fuente                         | Ruta                                    |
| -------------- | ------------------------------ | --------------------------------------- |
| 1 — máxima     | Skills del espacio de trabajo  | `<workspace>/skills`                    |
| 2              | Skills del agente del proyecto | `<workspace>/.agents/skills`            |
| 3              | Skills personales del agente   | `~/.agents/skills`                      |
| 4              | Skills gestionadas/locales      | `~/.openclaw/skills`                    |
| 5              | Skills incluidas                | incluidas con la instalación             |
| 6 — mínima     | Directorios adicionales        | `skills.load.extraDirs` + skills de plugins |

Las raíces de skills admiten estructuras agrupadas. OpenClaw detecta una skill siempre que
`SKILL.md` aparezca en cualquier lugar bajo una raíz configurada (hasta 6 niveles de profundidad):

```text
<workspace>/skills/research/SKILL.md          ✓ encontrada como "research"
<workspace>/skills/personal/research/SKILL.md ✓ también encontrada como "research"
```

La ruta de la carpeta solo sirve para la organización. El nombre de la skill y el comando con barra
provienen del campo de frontmatter `name` (o del nombre del directorio cuando falta
`name`). Las listas de permitidas del agente (más abajo) también se comparan con este
`name`.

<Note>
  El directorio nativo `$CODEX_HOME/skills` de Codex CLI **no** es una raíz de
  skills de OpenClaw. Use `openclaw migrate plan codex` para inventariar esas skills y, después,
  `openclaw migrate codex` para copiarlas en el espacio de trabajo de OpenClaw.
</Note>

## Skills alojadas en Node

Un Node sin interfaz conectado puede publicar las skills instaladas en su directorio activo de
skills de OpenClaw (`~/.openclaw/skills` de forma predeterminada; se aplican las sobrescrituras
del entorno del perfil). Aparecen en la lista normal de skills del agente mientras el Node está conectado
y desaparecen cuando se desconecta. Una skill local o del Gateway conserva su nombre en caso de
colisión; la skill del Node recibe un nombre determinista con prefijo del Node.
La versión v1 alojada en Node requiere que el nombre del directorio coincida con el campo de frontmatter
`name` de la skill.

La entrada de la skill incluye el localizador del Node. Sus archivos, referencias relativas y
binarios residen en el Node, por lo que debe cargarse y ejecutarse con
`exec host=node node=<node-id>`. Reinicie el host del Node después de cambiar sus archivos de
skills. Consulte [Nodos](/es/nodes#node-hosted-skills) para obtener información sobre el emparejamiento y los mecanismos de desactivación.

## Skills por agente frente a compartidas

En configuraciones con varios agentes, cada agente tiene su propio espacio de trabajo. Use la ruta que
corresponda a la visibilidad deseada:

| Ámbito              | Ruta                         | Visible para                         |
| ------------------- | ---------------------------- | ------------------------------------ |
| Por agente          | `<workspace>/skills`         | Solo ese agente                      |
| Agente del proyecto | `<workspace>/.agents/skills` | Solo el agente de ese espacio de trabajo |
| Agente personal     | `~/.agents/skills`           | Todos los agentes de esta máquina    |
| Gestionadas compartidas | `~/.openclaw/skills`      | Todos los agentes de esta máquina    |
| Directorios adicionales | `skills.load.extraDirs`     | Todos los agentes de esta máquina    |

## Listas de skills permitidas por agente

La **ubicación** de la skill (precedencia) y su **visibilidad** (qué agente puede
usarla) son controles independientes. Use listas de permitidas para restringir qué skills ve un agente,
independientemente de dónde se carguen.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // referencia compartida
    },
    list: [
      { id: "writer" }, // hereda github, weather
      { id: "docs", skills: ["docs-search"] }, // sustituye por completo los valores predeterminados
      { id: "locked-down", skills: [] }, // sin skills
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="Reglas de las listas de permitidas">
    - Omita `agents.defaults.skills` para dejar todas las skills sin restricciones de forma predeterminada.
    - Omita `agents.entries.*.skills` para heredar `agents.defaults.skills`.
    - Establezca `agents.entries.*.skills: []` para no exponer ninguna skill a ese agente.
    - Una lista no vacía de `agents.entries.*.skills` es el conjunto **definitivo**; no se
      combina con los valores predeterminados.
    - La lista de permitidas efectiva se aplica a la creación de prompts, la detección de comandos
      con barra, la sincronización del entorno aislado y las instantáneas de skills.
    - Esto no constituye un límite de autorización del shell del host. Si el mismo agente puede
      usar `exec`, restrinja ese shell por separado mediante aislamiento, separación
      por usuario del sistema operativo, listas de ejecución denegada/permitida y credenciales por recurso.
  </Accordion>
</AccordionGroup>

## Plugins y skills

Los plugins pueden incluir sus propias skills mediante la enumeración de directorios `skills` en
`openclaw.plugin.json` (rutas relativas a la raíz del plugin). Las skills del plugin se cargan
cuando el plugin está habilitado; por ejemplo, el plugin del navegador incluye una
skill `browser-automation` para el control del navegador en varios pasos.

Los directorios de skills de plugins se combinan en el mismo nivel de precedencia baja que
`skills.load.extraDirs`, por lo que una skill incluida, gestionada, de agente o de espacio de trabajo
con el mismo nombre los sobrescribe. Controle la elegibilidad de una skill del plugin mediante
`metadata.openclaw.requires` en su frontmatter, como con cualquier otra skill.

Consulte [Plugins](/es/tools/plugin) y [Herramientas](/es/tools) para conocer el sistema completo de plugins.

## Taller de skills

El [Taller de skills](/es/tools/skill-workshop) es una cola de propuestas entre el agente
y los archivos de skills activos. Cuando el agente detecta trabajo reutilizable, redacta una
propuesta en lugar de escribir directamente en `SKILL.md`. Debe revisarla y aprobarla
antes de que se produzca cualquier cambio.

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Consulte [Taller de skills](/es/tools/skill-workshop) para conocer el ciclo de vida completo, la referencia de la
CLI y la configuración.

## Instalación desde ClawHub

[ClawHub](https://clawhub.ai) es el registro público de skills. Use los comandos
`openclaw skills` para instalar y actualizar, o la CLI `clawhub` para
publicar y sincronizar.

| Acción                                        | Comando                                                |
| --------------------------------------------- | ------------------------------------------------------ |
| Instalar una skill en el espacio de trabajo   | `openclaw skills install @owner/<slug>`                |
| Instalar desde un repositorio Git             | `openclaw skills install git:owner/repo@ref`           |
| Instalar un directorio local de skills        | `openclaw skills install ./path/to/skill --as my-tool` |
| Instalar para todos los agentes locales       | `openclaw skills install @owner/<slug> --global`       |
| Actualizar todas las skills del espacio de trabajo | `openclaw skills update --all`                    |
| Actualizar una skill gestionada compartida    | `openclaw skills update @owner/<slug> --global`        |
| Actualizar todas las skills gestionadas compartidas | `openclaw skills update --all --global`               |
| Verificar el perímetro de confianza de una skill | `openclaw skills verify @owner/<slug>`                 |
| Mostrar la tarjeta de skill generada          | `openclaw skills verify @owner/<slug> --card`          |
| Publicar/sincronizar mediante la CLI de ClawHub | `clawhub sync --all`                                   |

<AccordionGroup>
  <Accordion title="Detalles de la instalación">
    `openclaw skills install` instala de forma predeterminada en el directorio `skills/`
    del espacio de trabajo activo. Añada `--global` para instalar en el directorio compartido
    `~/.openclaw/skills`, visible para todos los agentes locales salvo que las listas de
    permitidas de los agentes lo restrinjan.

    Las instalaciones desde Git y locales requieren `SKILL.md` en la raíz del origen. El identificador legible proviene
    del `name` del frontmatter `SKILL.md` cuando es válido; de lo contrario, se usa el
    nombre del directorio o repositorio. Use `--as <slug>` para sobrescribirlo.
    `openclaw skills update` solo realiza el seguimiento de instalaciones de ClawHub; reinstale los orígenes
    Git o locales para actualizarlos.

  </Accordion>
  <Accordion title="Verificación y análisis de seguridad">
    `openclaw skills verify @owner/<slug>` solicita a ClawHub el perímetro de confianza
    `clawhub.skill.verify.v1` de la skill. Las skills de ClawHub instaladas se verifican
    con la versión y el registro guardados en `.clawhub/origin.json`.
    Se siguen aceptando identificadores sin propietario para skills ya instaladas o no ambiguas, pero
    las referencias calificadas por propietario evitan ambigüedades sobre el publicador.

    Las páginas de skills de ClawHub muestran el estado del análisis de seguridad más reciente antes de la instalación,
    con páginas detalladas para VirusTotal, ClawScan y el análisis estático. El
    comando finaliza con un código distinto de cero cuando ClawHub marca la verificación como fallida. Los publicadores
    pueden resolver falsos positivos mediante el panel de ClawHub o
    `clawhub skill rescan @owner/<slug>`.

  </Accordion>
  <Accordion title="Instalaciones desde archivos privados">
    Los clientes del Gateway que necesiten un método de entrega distinto de ClawHub pueden preparar un archivo ZIP de una skill
    con `skills.upload.begin`, `skills.upload.chunk` y `skills.upload.commit`,
    y después instalarlo con `skills.install({ source: "upload", ... })`. Esta ruta está
    desactivada de forma predeterminada y requiere `skills.install.allowUploadedArchives: true` en
    `openclaw.json`. Las instalaciones normales desde ClawHub nunca necesitan esa opción.
  </Accordion>
</AccordionGroup>

## Seguridad

<Warning>
  Trate las skills de terceros como **código no confiable**. Léalas antes de habilitarlas.
  Es preferible realizar ejecuciones aisladas para entradas no confiables y herramientas de riesgo. Consulte
  [Aislamiento](/es/gateway/sandboxing) para conocer los controles del lado del agente.
</Warning>

<AccordionGroup>
  <Accordion title="Contención de rutas">
    La detección de skills del espacio de trabajo, del agente del proyecto y de directorios adicionales solo acepta
    raíces de skills cuya ruta real resuelta permanezca dentro de la raíz configurada, salvo que
    `skills.load.allowSymlinkTargets` confíe explícitamente en una raíz de destino.
    El Taller de skills solo escribe a través de esos destinos de confianza cuando
    `skills.workshop.allowSymlinkTargetWrites` está habilitado.
    Los directorios gestionados `~/.openclaw/skills` y personales `~/.agents/skills` pueden contener
    carpetas de skills con enlaces simbólicos, pero la ruta real de cada `SKILL.md` debe permanecer
    dentro del directorio resuelto de la skill.
  </Accordion>
  <Accordion title="Política de instalación del operador">
    Configure `security.installPolicy` para ejecutar un comando de política local de confianza
    antes de que continúen las instalaciones de skills. La política recibe metadatos y la ruta del
    origen preparado, se aplica a las rutas de ClawHub, carga, Git, local, actualización e
    instalador de dependencias, y aplica un cierre seguro cuando el comando no puede devolver
    una decisión válida.
  </Accordion>
  <Accordion title="Ámbito de inyección de secretos">
    `skills.entries.*.env` y `skills.entries.*.apiKey` inyectan secretos en el proceso
    **host** solo durante ese turno del agente, no en el entorno aislado. Mantenga los
    secretos fuera de los prompts y registros.
  </Accordion>
</AccordionGroup>

Para consultar el modelo de amenazas más amplio y las listas de comprobación de seguridad, consulte
[Seguridad](/es/gateway/security).

## Formato de SKILL.md

Cada skill necesita como mínimo un `name` y un `description` en el frontmatter:

```markdown
---
name: image-lab
description: Generar o editar imágenes mediante un flujo de trabajo de imágenes respaldado por un proveedor
---

Cuando el usuario solicite generar una imagen, use la herramienta `image_generate`...
```

<Note>
  OpenClaw sigue la especificación [AgentSkills](https://agentskills.io). El frontmatter
  se analiza primero como YAML; si esto falla, se recurre a un analizador que solo admite
  una línea. Los bloques `metadata` anidados (incluidas las asignaciones YAML de varias líneas) se
  convierten en una cadena JSON y se vuelven a analizar como JSON5, por lo que funciona el formato de bloque mostrado
  en [Control de elegibilidad](#gating). Use `{baseDir}` en el cuerpo para hacer referencia a la
  ruta de la carpeta de la skill.
</Note>

### Claves opcionales del frontmatter

<ParamField path="homepage" type="string">
  URL que aparece como "Website" en la interfaz de Skills de macOS. También se admite mediante
  `metadata.openclaw.homepage`.
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  Cuando `true`, la skill se expone como un comando de barra diagonal invocable por el usuario.
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  Cuando `true`, OpenClaw mantiene las instrucciones de la skill fuera del prompt
  normal del agente. La skill sigue estando disponible como comando de barra diagonal cuando `user-invocable`
  también es `true`.
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  Cuando se establece en `tool`, el comando de barra diagonal omite el modelo y se despacha
  directamente a una herramienta registrada.
</ParamField>

<ParamField path="command-tool" type="string">
  Nombre de la herramienta que se invocará cuando se establezca `command-dispatch: tool`.
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  Para el despacho a herramientas, reenvía la cadena de argumentos sin procesar a la herramienta sin
  análisis del núcleo. La herramienta recibe
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`.
</ParamField>

## Restricciones

OpenClaw filtra las skills durante la carga mediante `metadata.openclaw` (objeto JSON5
incrustado en el frontmatter; consulte la nota sobre el análisis anterior). Una skill sin un bloque
`metadata.openclaw` siempre es apta, salvo que se deshabilite explícitamente.

```markdown
---
name: image-lab
description: Generar o editar imágenes mediante un flujo de trabajo de imágenes respaldado por un proveedor
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  Cuando `true`, incluye siempre la skill y omite todas las demás restricciones.
</ParamField>

<ParamField path="emoji" type="string">
  Emoji opcional que se muestra en la interfaz de Skills de macOS.
</ParamField>

<ParamField path="homepage" type="string">
  URL opcional que se muestra como "Website" en la interfaz de Skills de macOS.
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  Filtro de plataforma. Cuando se establece, la skill solo es apta en uno de los sistemas operativos indicados.
</ParamField>

<ParamField path="requires.bins" type="string[]">
  Cada binario debe existir en `PATH`.
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  Al menos un binario debe existir en `PATH`.
</ParamField>

<ParamField path="requires.env" type="string[]">
  Cada variable de entorno debe existir en el proceso o proporcionarse mediante la configuración.
</ParamField>

<ParamField path="requires.config" type="string[]">
  Cada ruta `openclaw.json` debe evaluarse como verdadera.
</ParamField>

<ParamField path="primaryEnv" type="string">
  Nombre de la variable de entorno asociada con `skills.entries.<name>.apiKey`.
</ParamField>

<ParamField path="install" type="object[]">
  Especificaciones opcionales del instalador que utiliza la interfaz de Skills de macOS (brew / node / go / uv / download).
</ParamField>

<Note>
  Los bloques heredados `metadata.clawdbot` se siguen aceptando cuando
  `metadata.openclaw` está ausente, de modo que las skills instaladas más antiguas conservan sus
  restricciones de dependencias y sugerencias del instalador. Las skills nuevas deben usar
  `metadata.openclaw`.
</Note>

### Especificaciones del instalador

Las especificaciones del instalador indican a la interfaz de Skills de macOS cómo instalar una dependencia:

```markdown
---
name: gemini
description: Usar la CLI de Gemini para obtener asistencia de programación y realizar búsquedas en Google.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Instalar la CLI de Gemini (brew)",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="Reglas de selección del instalador">
    - Cuando se enumeran varios instaladores, el Gateway elige una opción
      preferida (brew cuando está disponible; de lo contrario, node).
    - Si todos los instaladores son `download`, OpenClaw enumera cada entrada para que se puedan
      ver todos los artefactos disponibles.
    - Las especificaciones pueden incluir `os: ["darwin"|"linux"|"win32"]` para filtrar por plataforma.
    - Las instalaciones de Node respetan `skills.install.nodeManager` en `openclaw.json`
      (valor predeterminado: npm; opciones: npm / pnpm / yarn / bun). Esto solo afecta a las
      instalaciones de skills; el entorno de ejecución del Gateway debe seguir siendo Node.
    - Preferencia de instaladores del Gateway: Homebrew → uv → gestor de node configurado →
      go → descarga.
  </Accordion>
  <Accordion title="Detalles por instalador">
    - **Homebrew:** OpenClaw no instala Homebrew automáticamente ni traduce las
      fórmulas de brew a comandos de paquetes del sistema. En contenedores Linux sin
      `brew`, los instaladores que solo usan brew se ocultan; utilice una imagen personalizada o instale
      la dependencia manualmente.
    - **Go:** OpenClaw requiere Go 1.21 o una versión posterior para las instalaciones automáticas de skills.
      Si falta `go` y Homebrew está disponible, OpenClaw instala primero Go mediante
      Homebrew; en Linux sin Homebrew, puede usar en su lugar `apt-get`
      como root o mediante `sudo` sin contraseña cuando el candidato actualizado de `golang-go`
      cumple la versión mínima. El `go install` real de la
      dependencia siempre apunta a un directorio de binarios dedicado y administrado por OpenClaw
      (`bin` de Homebrew en una instalación nueva; de lo contrario, `~/.local/bin`), en lugar de
      su `GOBIN` configurado; sus propias variables de entorno `GOBIN`, `GOPATH` y `GOTOOLCHAIN`
      se leen, pero nunca se sobrescriben.
    - **Descarga:** `url` (obligatorio), `archive` (`tar.gz` | `tar.bz2` | `zip`),
      `extract` (valor predeterminado: automático cuando se detecta un archivo), `stripComponents`,
      `targetDir` (valor predeterminado: `~/.openclaw/tools/<skillKey>`).
  </Accordion>
  <Accordion title="Notas sobre el aislamiento">
    `requires.bins` se comprueba en el **host** durante la carga de la skill. Si un agente
    se ejecuta en un entorno aislado, el binario también debe existir **dentro del contenedor**.
    Instálelo mediante `agents.defaults.sandbox.docker.setupCommand` o una imagen
    personalizada. `setupCommand` se ejecuta una vez después de crear el contenedor y requiere
    acceso de salida a la red, un sistema de archivos raíz con permisos de escritura y un usuario root en el entorno aislado.
  </Accordion>
</AccordionGroup>

## Anulaciones de configuración

Active y configure las skills incluidas o administradas en `skills.entries` dentro de
`~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false` deshabilita la skill incluso cuando está incluida o instalada. La skill incluida
  `coding-agent` es opcional: establezca `skills.entries.coding-agent.enabled: true`
  y asegúrese de que `claude`, `codex`, `opencode` u otra CLI compatible
  esté instalada y autenticada.
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  Campo práctico para las skills que declaran `metadata.openclaw.primaryEnv`.
  Admite una cadena de texto sin formato o un objeto SecretRef.
</ParamField>

<ParamField path="env" type="Record<string, string>">
  Variables de entorno inyectadas para la ejecución del agente. Solo se inyectan cuando la
  variable aún no está establecida en el proceso.
</ParamField>

<ParamField path="config" type="object">
  Contenedor opcional para campos de configuración personalizados por skill.
</ParamField>

<ParamField path="allowBundled" type="string[]">
  Lista de permitidas opcional solo para las skills **incluidas**. Cuando se establece, únicamente son
  aptas las skills incluidas que aparecen en la lista. Las skills administradas y del espacio de trabajo no se ven afectadas.
</ParamField>

<Note>
  De forma predeterminada, las claves de configuración coinciden con el **nombre de la skill**. Si una skill define
  `metadata.openclaw.skillKey`, utilice esa clave en `skills.entries`.
  Escriba entre comillas los nombres con guiones: JSON5 permite claves entre comillas.
</Note>

## Inyección del entorno

Cuando se inicia una ejecución del agente, OpenClaw:

<Steps>
  <Step title="Lee los metadatos de las skills">
    OpenClaw resuelve la lista efectiva de skills del agente y aplica reglas de
    restricción, listas de permitidas y anulaciones de configuración.
  </Step>
  <Step title="Inyecta variables de entorno y claves de API">
    `skills.entries.<key>.env` y `skills.entries.<key>.apiKey` se aplican a
    `process.env` mientras dura la ejecución.
  </Step>
  <Step title="Crea el prompt del sistema">
    Las skills aptas se compilan en un bloque XML compacto y se inyectan en el
    prompt del sistema.
  </Step>
  <Step title="Restaura el entorno">
    Cuando finaliza la ejecución, se restaura el entorno original.
  </Step>
</Steps>

<Warning>
  La inyección de variables de entorno se limita a la ejecución del agente en el **host**, no al entorno aislado. Dentro de un
  entorno aislado, `env` y `apiKey` no tienen efecto. Consulte
  [Configuración de Skills](/es/tools/skills-config#sandboxed-skills-and-env-vars) para saber cómo
  pasar secretos a ejecuciones aisladas.
</Warning>

Para el backend incluido `claude-cli`, OpenClaw también materializa la misma
instantánea de skills aptas como Plugin temporal de Claude Code y la pasa mediante
`--plugin-dir`. Los demás backends de CLI solo utilizan el catálogo del prompt.

## Instantáneas y actualización

OpenClaw crea una instantánea de las skills aptas **cuando se inicia una sesión** y reutiliza esa
lista en todos los turnos posteriores de la sesión. Los cambios en las skills o en la configuración surten
efecto en la siguiente sesión nueva.

Las skills se actualizan durante una sesión en dos casos:

- El observador de skills detecta un cambio en `SKILL.md`.
- Se conecta un nuevo nodo remoto apto.

La lista actualizada se utiliza en el siguiente turno del agente. Si cambia la lista de permitidas efectiva
del agente, OpenClaw actualiza la instantánea para mantener alineadas las skills
visibles.

<AccordionGroup>
  <Accordion title="Observador de Skills">
    De forma predeterminada, OpenClaw observa las carpetas de skills y actualiza la instantánea cuando
    cambian los archivos `SKILL.md`. Configúrelo en `skills.load`:

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // valor predeterminado
        },
      },
    }
    ```

    Los eventos del observador utilizan una estabilización integrada de 250 ms. Utilice `allowSymlinkTargets`
    para estructuras intencionales con enlaces simbólicos en las que un enlace simbólico de la
    raíz de una skill apunta fuera de la raíz configurada, por ejemplo,
    `<workspace>/skills/manager -> ~/Projects/manager/skills`.
    Habilite `skills.workshop.allowSymlinkTargetWrites` solo cuando Skill Workshop
    también deba aplicar propuestas mediante esas rutas de enlaces simbólicos de confianza.

  </Accordion>
  <Accordion title="Nodos macOS remotos (Gateway Linux)">
    Si el Gateway se ejecuta en Linux, pero hay conectado un **nodo macOS** con
    `system.run` permitido, OpenClaw puede considerar aptas las skills exclusivas de macOS cuando
    los binarios necesarios están presentes en ese nodo. El agente debe ejecutar esas
    skills mediante la herramienta `exec` con `host=node`.

    Los nodos sin conexión **no** hacen visibles las skills exclusivas de acceso remoto. Si un nodo deja de
    responder a los sondeos de binarios, OpenClaw borra sus coincidencias de binarios almacenadas en caché.

  </Accordion>
</AccordionGroup>

## Impacto en los tokens

Cuando hay skills aptas, OpenClaw inyecta un bloque XML compacto en el prompt
del sistema. El coste es determinista y aumenta linealmente por skill:

- **Sobrecarga base** (solo cuando hay 1 o más skills aptas): un bloque fijo de texto
  introductorio más el contenedor `<available_skills>`.
- **Por skill:** ~97 caracteres + las longitudes de los campos `name`, `description` y `location`.
- El escape de XML expande `& < > " '` en entidades, lo que añade algunos caracteres por
  aparición.
- Con ~4 caracteres/token, 97 caracteres ≈ 24 tokens por skill antes de las longitudes de los campos.

Si el bloque renderizado superara el presupuesto configurado del prompt
(`skills.limits.maxSkillsPromptChars`), OpenClaw conserva primero tantas identidades de Skills
(nombre, ubicación y versión) como permita el formato compacto sin
descripciones. Después, utiliza el presupuesto restante para descripciones abreviadas. Si no
queda presupuesto para descripciones, estas se omiten. El prompt incluye una
nota que remite a `openclaw skills check` siempre que se requiera el formato compacto o el
truncamiento de la lista.

Mantenga las descripciones breves y descriptivas para minimizar la sobrecarga del prompt.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Creación de Skills" href="/es/tools/creating-skills" icon="hammer">
    Guía paso a paso para crear una Skill personalizada.
  </Card>
  <Card title="Taller de Skills" href="/es/tools/skill-workshop" icon="flask">
    Cola de propuestas para Skills redactadas por agentes.
  </Card>
  <Card title="Configuración de Skills" href="/es/tools/skills-config" icon="gear">
    Esquema completo de configuración de `skills.*` y listas de agentes permitidos.
  </Card>
  <Card title="Comandos de barra diagonal" href="/es/tools/slash-commands" icon="terminal">
    Cómo se registran y enrutan los comandos de barra diagonal de las Skills.
  </Card>
  <Card title="ClawHub" href="/es/clawhub" icon="cloud">
    Explore y publique Skills en el registro público.
  </Card>
  <Card title="Plugins" href="/es/tools/plugin" icon="plug">
    Los Plugins pueden incluir Skills junto con las herramientas que documentan.
  </Card>
</CardGroup>
