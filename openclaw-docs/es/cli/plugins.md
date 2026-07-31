---
read_when:
    - Se desea instalar o gestionar plugins del Gateway o paquetes compatibles
    - Quieres crear la estructura inicial o validar un Plugin de herramientas sencillo
    - Quiere depurar los fallos de carga de plugins
sidebarTitle: Plugins
summary: Referencia de la CLI para `openclaw plugins` (inicializar, compilar, validar, listar, instalar, marketplace, desinstalar, habilitar/deshabilitar, diagnosticar)
title: Plugins
x-i18n:
    generated_at: "2026-07-26T05:35:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a1acba76fb1bc0ddae75e51fe573d3c2ac8f694607836e0c072ec7ca8fc0e262
    source_path: cli/plugins.md
    workflow: 16
---

Gestiona plugins de Gateway, paquetes de hooks y bundles compatibles.

<CardGroup cols={2}>
  <Card title="Sistema de plugins" href="/es/tools/plugin">
    Guía para usuarios finales sobre cómo instalar, habilitar y solucionar problemas de plugins.
  </Card>
  <Card title="Gestionar plugins" href="/es/plugins/manage-plugins">
    Ejemplos rápidos para instalar, enumerar, actualizar, desinstalar y publicar.
  </Card>
  <Card title="Bundles de plugins" href="/es/plugins/bundles">
    Modelo de compatibilidad de bundles.
  </Card>
  <Card title="Manifiesto de plugins" href="/es/plugins/manifest">
    Campos del manifiesto y esquema de configuración.
  </Card>
  <Card title="Seguridad" href="/es/gateway/security">
    Refuerzo de la seguridad para las instalaciones de plugins.
  </Card>
</CardGroup>

## Comandos

```bash
openclaw plugins list [--enabled] [--verbose] [--json]
openclaw plugins search <query> [--limit <n>] [--json]
openclaw plugins install <path-or-spec> [--link] [--force] [--pin] [--marketplace <source>]
openclaw plugins inspect <id> [--runtime] [--json]
openclaw plugins inspect --all [--runtime] [--json]
openclaw plugins info <id>                    # alias de inspect
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id> [--dry-run] [--keep-files] [--force]
openclaw plugins update <id-or-npm-spec> | --all [--dry-run]
openclaw plugins registry [--refresh] [--json]
openclaw plugins doctor
openclaw plugins init <id> [--name <name>] [--type tool|provider] [--directory <path>]
openclaw plugins build [--entry <path>] [--check]
openclaw plugins validate [--entry <path>]
openclaw plugins marketplace entries [--offline] [--feed-profile <name>] [--json]
openclaw plugins marketplace list <source> [--json]
openclaw plugins marketplace refresh [--feed-profile <name>] [--expected-sha256 <sha256>] [--json]
```

Para investigar instalaciones, inspecciones, desinstalaciones o actualizaciones del registro lentas, ejecuta el
comando con `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1`. La traza escribe los tiempos de las fases
en stderr y mantiene analizable la salida JSON. Consulta [Depuración](/es/help/debugging#plugin-lifecycle-trace).

<Note>
En el modo Nix (`OPENCLAW_NIX_MODE=1`), `openclaw.json` es inmutable. `install`, `update`, `uninstall`, `enable` y `disable` se niegan a ejecutarse. En su lugar, edita la fuente de Nix de esta instalación (`programs.openclaw.config` o `instances.<name>.config` para nix-openclaw) y, a continuación, vuelve a compilar. Consulta el [Inicio rápido](https://github.com/openclaw/nix-openclaw#quick-start) orientado a agentes.
</Note>

<Note>
Los plugins incluidos se distribuyen con OpenClaw. Algunos están habilitados de forma predeterminada (por ejemplo, los proveedores de modelos incluidos, los proveedores de voz incluidos y el plugin de navegador incluido); otros requieren `plugins enable`.

Los plugins nativos de OpenClaw incluyen `openclaw.plugin.json` con un JSON Schema en línea (`configSchema`, aunque esté vacío). Los bundles compatibles utilizan sus propios manifiestos de bundle.

`plugins list` muestra `Format: openclaw` o `Format: bundle`. La salida detallada de lista/información también muestra el subtipo de bundle (`codex`, `claude` o `cursor`) junto con las capacidades detectadas del bundle.
</Note>

## Creación

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm run plugin:build
npm run plugin:validate
```

`plugins init` crea de forma predeterminada un plugin de herramientas mínimo en TypeScript. El primer
argumento es el id del plugin; `--name` establece el nombre para mostrar. OpenClaw utiliza el
id para el directorio de salida predeterminado y el nombre del paquete. Las plantillas de herramientas utilizan
`defineToolPlugin` y generan los scripts `package.json` `plugin:build` y
`plugin:validate`, que compilan y después llaman a `openclaw plugins build`/`validate`.

`plugins build` importa el punto de entrada compilado, lee los metadatos estáticos de sus herramientas, escribe
`openclaw.plugin.json` y mantiene alineado el `openclaw.extensions` de `package.json`.
`plugins validate` comprueba que el manifiesto generado, los metadatos del paquete y
la exportación actual del punto de entrada sigan coincidiendo. Consulta [Plugins de herramientas](/es/plugins/tool-plugins) para conocer
el flujo de creación completo.

La plantilla escribe el código fuente de TypeScript, pero genera los metadatos a partir del punto de entrada
`./dist/index.js` compilado, por lo que el flujo también funciona con la CLI publicada. Utiliza
`--entry <path>` cuando el punto de entrada no sea el punto de entrada predeterminado del paquete. Utiliza
`plugins build --check` en la Pipeline de CI para que se produzca un error cuando los metadatos generados estén obsoletos sin
reescribir los archivos.

### Plantilla de proveedor

```bash
openclaw plugins init acme-models --name "Acme Models" --type provider
cd acme-models
npm install
npm run build
npm test
npm run validate
```

Las plantillas de proveedores crean un plugin genérico de proveedor de modelos compatible con OpenAI
con la infraestructura de autenticación mediante clave de API, un script `npm run validate` que ejecuta
`clawhub package validate`, metadatos de paquete de ClawHub y un flujo de trabajo de GitHub Actions
que se ejecuta manualmente para permitir futuras publicaciones de confianza mediante OIDC de GitHub.
Las plantillas de proveedores no generan Skills ni utilizan
`openclaw plugins build`/`validate`; esos comandos son para la ruta de metadatos generados
de la plantilla de herramientas.

Antes de publicar, sustituye la URL base de la API de marcador de posición, el catálogo de modelos, la ruta de la documentación,
el texto de las credenciales y el contenido del README por los datos reales del proveedor. Utiliza el
README generado para la primera publicación en ClawHub y la configuración del publicador de confianza.

## Instalación

```bash
openclaw plugins search "calendar"                      # buscar plugins de ClawHub
openclaw plugins install @openclaw/<package>            # catálogo oficial de confianza
openclaw plugins install <package>                       # paquete npm arbitrario
openclaw plugins install clawhub:<package>                # solo ClawHub
openclaw plugins install npm:<package>                    # solo npm
openclaw plugins install npm-pack:<path.tgz>               # archivo tar local de npm-pack
openclaw plugins install git:github.com/<owner>/<repo>     # repositorio git
openclaw plugins install git:github.com/<owner>/<repo>@<ref>
openclaw plugins install <path>                            # ruta o archivo local
openclaw plugins install -l <path>                         # enlazar en lugar de copiar
openclaw plugins install <plugin>@<marketplace>             # forma abreviada del marketplace
openclaw plugins install <plugin> --marketplace <name>      # marketplace (explícito)
openclaw plugins install <package> --force                  # confirmar fuente/sobrescribir existente
openclaw plugins install <package> --pin                    # fijar la versión de npm resuelta
openclaw plugins install clawhub:<package> --acknowledge-clawhub-risk
openclaw plugins install <package> --dangerously-force-unsafe-install
```

Los mantenedores que prueben instalaciones durante la configuración pueden sustituir las fuentes automáticas
de instalación de plugins mediante variables de entorno protegidas. Consulta
[Anulaciones de instalación de plugins](/es/plugins/install-overrides).

<Warning>
Durante la transición de lanzamiento, los nombres de paquete sin prefijo se instalan de forma predeterminada desde npm, salvo que coincidan con el id de un plugin incluido u oficial, en cuyo caso OpenClaw utiliza esa copia local/oficial en lugar de acceder al registro de npm. Utiliza `npm:<package>` cuando se quiera expresamente un paquete npm externo. Utiliza `clawhub:<package>` para ClawHub. Trata las instalaciones de plugins como ejecución de código; prefiere versiones fijadas.
</Warning>

<Warning>
Los paquetes de ClawHub y el catálogo incluido/oficial de OpenClaw son fuentes de instalación
de confianza. Una nueva fuente arbitraria de npm, `npm-pack:`, git, ruta/archivo local o
marketplace muestra una advertencia y solicita confirmación antes de continuar. Las instalaciones arbitrarias
no interactivas deben proporcionar `--force` después de revisar la fuente y confiar en ella. La misma
opción sobrescribe un destino de instalación existente cuando es necesario. Las actualizaciones normales de una
instalación ya registrada no la requieren. Esta confirmación es independiente de
`--acknowledge-clawhub-risk`, que solo se aplica a las advertencias de confianza de versiones arriesgadas de
ClawHub. `--force` no omite `security.installPolicy` ni las demás
comprobaciones de seguridad de la instalación.
</Warning>

`plugins search` consulta en ClawHub los paquetes instalables `code-plugin` y
`bundle-plugin` (no Skills; utiliza `openclaw skills search` para ellas).
El valor predeterminado de `--limit` es 20, con un máximo de 100. Solo lee el catálogo remoto: no
inspecciona el estado local, modifica la configuración, instala paquetes ni carga el entorno de ejecución
de plugins. Los resultados incluyen el nombre del paquete de ClawHub, la familia, el canal, la versión,
el resumen y una indicación de instalación como `openclaw plugins install clawhub:<package>`.

<Note>
ClawHub es la principal superficie de distribución y descubrimiento para la mayoría de los plugins. Npm
sigue siendo una alternativa compatible y una ruta de instalación directa. Los paquetes de plugins
`@openclaw/*` propiedad de OpenClaw vuelven a publicarse en npm; consulta la lista actual
en [npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) o el
[inventario de plugins](/es/plugins/plugin-inventory). Las instalaciones estables utilizan `latest`.
Las instalaciones y actualizaciones del canal beta prefieren la etiqueta de distribución `beta` de npm cuando está disponible,
con `latest` como alternativa. En el canal estable extendido, los plugins oficiales de npm
con intención sin prefijo/predeterminada o `latest` se resuelven a la versión principal
instalada exacta. Las versiones fijadas exactas y las etiquetas explícitas distintas de `latest`, los paquetes de terceros y
las fuentes distintas de npm no se reescriben.
</Note>

<AccordionGroup>
  <Accordion title="Inclusiones de configuración y reparación de configuraciones no válidas">
    Si la sección `plugins` está respaldada por una inclusión `$include` de un solo archivo, `plugins install/update/enable/disable/uninstall` escribe directamente en ese archivo incluido y deja `openclaw.json` intacto. Las inclusiones raíz, las matrices de inclusiones y las inclusiones con anulaciones hermanas se cierran de forma segura en lugar de aplanarse. Consulta [Inclusiones de configuración](/es/gateway/configuration) para conocer las formas compatibles.

    Si la configuración no es válida antes de la instalación, `plugins install` normalmente se cierra de forma segura y solicita ejecutar primero `openclaw doctor --fix`. Durante el inicio y la recarga en caliente de Gateway, una configuración de plugin no válida se cierra de forma segura como cualquier otra configuración no válida; `openclaw doctor --fix` puede poner en cuarentena la entrada de plugin no válida. La única excepción para una configuración preexistente es una ruta limitada de recuperación de plugins incluidos para los plugins que habilitan explícitamente `openclaw.install.allowInvalidConfigRecovery`.

    Cuando la configuración existente del host es válida, pero no existe la configuración propia del plugin recién instalado, OpenClaw registra la instalación como deshabilitada en lugar de escribir una entrada habilitada no válida. Configura `plugins.entries.<id>.config` y, a continuación, ejecuta `openclaw plugins enable <id>`. Si ya existe una entrada de configuración del plugin, pero no es válida, la instalación falla sin reescribirla.

  </Accordion>
  <Accordion title="Confirmación con --force y reinstalación frente a actualización">
    `--force` confirma una fuente que no sea de ClawHub sin solicitar confirmación. No omite `security.installPolicy` ni las demás comprobaciones de seguridad de la instalación. Cuando el plugin o el paquete de hooks ya está instalado, también reutiliza el destino existente y lo sobrescribe en el mismo lugar. Utilízalo después de revisar una fuente arbitraria de npm, local, de archivo, git o marketplace, o cuando se reinstale intencionadamente el mismo id. Para las actualizaciones rutinarias de un plugin de npm ya registrado, prefiere `openclaw plugins update <id-or-npm-spec>`.

    Si se ejecuta `plugins install` para el id de un plugin que ya está instalado, OpenClaw se detiene y dirige a `plugins update <id-or-npm-spec>` para una actualización normal, o a `plugins install <package> --force` cuando realmente se quiera sobrescribir la instalación actual desde una fuente diferente. Las fuentes arbitrarias siguen mostrando la advertencia interactiva sobre la procedencia; las instalaciones no interactivas deben proporcionar `--force` después de la revisión. Las fuentes de confianza de ClawHub y del catálogo de OpenClaw no la necesitan. Con `--link`, `--force` confirma la fuente, pero no cambia el modo de instalación mediante ruta enlazada.

  </Accordion>
  <Accordion title="ámbito de --pin">
    `--pin` se aplica únicamente a instalaciones de npm y registra la `<name>@<version>` exacta resuelta. No es compatible con instalaciones `git:` (en su lugar, fije la referencia en la especificación, por ejemplo, `git:github.com/acme/plugin@v1.2.3`) ni con `--marketplace` (las instalaciones del marketplace conservan los metadatos de origen del marketplace en lugar de una especificación de npm).
  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` está obsoleto y ahora no realiza ninguna operación. OpenClaw ya no ejecuta el bloqueo integrado de código peligroso durante la instalación de plugins.

    Utilice la superficie `security.installPolicy`, controlada por el operador, cuando se requiera una política de instalación específica del host. Los hooks `before_install` del plugin son hooks del ciclo de vida del entorno de ejecución del plugin, no el límite de política principal para las instalaciones mediante la CLI.

    Si un plugin que publicó en ClawHub está oculto o bloqueado por un análisis del registro, siga los pasos para editores de [Publicación en ClawHub](/es/clawhub/publishing). `--dangerously-force-unsafe-install` no solicita a ClawHub que vuelva a analizar el plugin ni que haga pública una versión bloqueada.

  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk">
    Las instalaciones comunitarias de ClawHub comprueban el registro de confianza de la versión seleccionada antes de descargarla. Si ClawHub deshabilita la descarga de la versión, informa de hallazgos maliciosos en el análisis o coloca la versión en un estado de moderación bloqueante (en cuarentena, revocada), OpenClaw la rechaza de forma incondicional, independientemente de esta opción. Para estados de análisis riesgosos no bloqueantes o estados de moderación no bloqueantes, OpenClaw muestra los detalles de confianza y solicita confirmación antes de continuar.

    Utilice `--acknowledge-clawhub-risk` únicamente después de revisar la advertencia de ClawHub y decidir continuar sin una solicitud interactiva. Los resultados de análisis pendientes u obsoletos (aún no limpios) generan una advertencia, pero no requieren confirmación. Los paquetes oficiales de ClawHub y las fuentes de plugins incluidas con OpenClaw omiten por completo esta comprobación de confianza de la versión.

  </Accordion>
  <Accordion title="Paquetes de hooks y especificaciones de npm">
    `plugins install` también es la superficie de instalación para los paquetes de hooks que exponen `openclaw.hooks` en `package.json`. Utilice `openclaw hooks` para la visibilidad filtrada de hooks y la habilitación individual de cada hook, no para la instalación de paquetes.

    Las especificaciones de npm son **exclusivas del registro** (nombre del paquete más una **versión exacta** o **dist-tag** opcional). Se rechazan las especificaciones de Git/URL/archivo y los rangos de semver. Por seguridad, las instalaciones de dependencias se ejecutan en un proyecto npm administrado por plugin con `--ignore-scripts`, incluso cuando el shell tiene configuraciones globales de instalación de npm. Los proyectos npm administrados de los plugins heredan el `overrides` de npm a nivel de paquete de OpenClaw, por lo que las fijaciones de seguridad del host también se aplican a las dependencias elevadas de los plugins.

    Utilice `npm:<package>` para hacer explícita la resolución de npm. Durante la transición del lanzamiento, las especificaciones de paquetes sin prefijo también se instalan directamente desde npm, salvo que coincidan con un id de plugin oficial.

    Las especificaciones `@openclaw/*` sin procesar que coincidan con plugins incluidos se resuelven a la copia incluida propiedad de la imagen antes de recurrir a npm. Por ejemplo, `openclaw plugins install @openclaw/discord@2026.5.20 --pin` utiliza el plugin de Discord incluido en la compilación actual de OpenClaw en lugar de crear una sustitución administrada de npm. Para forzar el paquete npm externo, utilice `openclaw plugins install npm:@openclaw/discord@2026.5.20 --pin`.

    Las especificaciones sin prefijo y `@latest` permanecen en el canal estable. Las versiones correctivas de OpenClaw con fecha, como `2026.5.3-1`, se consideran estables para esta comprobación. Si npm resuelve cualquiera de las dos formas a una versión preliminar, OpenClaw se detiene y solicita habilitarla explícitamente mediante una etiqueta de versión preliminar (`@beta`/`@rc`) o una versión preliminar exacta (`@1.2.3-beta.4`).

    Para las instalaciones de npm sin una versión exacta (`npm:<package>` o `npm:<package>@latest`), OpenClaw comprueba los metadatos del paquete resuelto antes de instalarlo. Si el paquete estable más reciente requiere una API de plugins de OpenClaw más nueva o una versión mínima del host superior, OpenClaw examina versiones estables anteriores e instala en su lugar la versión compatible más reciente. Las versiones exactas y los dist-tags explícitos siguen siendo estrictos: una selección incompatible falla y solicita actualizar OpenClaw o elegir una versión compatible.

    Si una especificación de instalación sin prefijo coincide con un id de plugin oficial (por ejemplo, `diffs`), OpenClaw instala directamente la entrada del catálogo. Para instalar un paquete npm con el mismo nombre, utilice una especificación con ámbito explícito (por ejemplo, `@scope/diffs`).

  </Accordion>
  <Accordion title="Repositorios Git">
    Utilice `git:<repo>` para instalar directamente desde un repositorio Git. Formas compatibles: `git:github.com/owner/repo`, `git:owner/repo`, `https://` completo, `ssh://`, `git://`, `file://` y URL de clonación `git@host:owner/repo.git`. Añada `@<ref>` o `#<ref>` para extraer una rama, etiqueta o commit antes de la instalación.

    Las instalaciones desde Git clonan el contenido en un directorio temporal, extraen la referencia solicitada cuando está presente y luego utilizan el instalador normal de directorios de plugins; por tanto, la validación del manifiesto, la política de instalación del operador, el trabajo de instalación del gestor de paquetes y los registros de instalación se comportan igual que en las instalaciones de npm. Las instalaciones desde Git registradas incluyen la URL/referencia de origen y el commit resuelto para que `openclaw plugins update` pueda volver a resolver el origen posteriormente.

    Después de instalar desde Git, utilice `openclaw plugins inspect <id> --runtime --json` para verificar los registros del entorno de ejecución, como los métodos del Gateway y los comandos de la CLI. Si el plugin registró una raíz de la CLI con `api.registerCli`, ejecute ese comando directamente mediante la CLI raíz de OpenClaw, por ejemplo, `openclaw demo-plugin ping`.

  </Accordion>
  <Accordion title="Archivos comprimidos">
    Archivos compatibles: `.zip`, `.tgz`, `.tar.gz`, `.tar`. Los archivos de plugins nativos de OpenClaw deben contener un `openclaw.plugin.json` válido en la raíz extraída del plugin; los archivos que solo contienen `package.json` se rechazan antes de que OpenClaw escriba los registros de instalación.

    Utilice `npm-pack:<path.tgz>` cuando el archivo sea un tarball de npm-pack y se desee
    usar la misma ruta de proyecto npm administrado por plugin que emplean las instalaciones desde el registro,
    incluida la verificación de `package-lock.json`, el análisis de dependencias elevadas
    y los registros de instalación de npm. Las rutas de archivos simples siguen instalándose como
    archivos locales bajo la raíz de extensiones de plugins.

    También se admiten las instalaciones desde el marketplace de Claude.

  </Accordion>
</AccordionGroup>

Las instalaciones de ClawHub utilizan un localizador `clawhub:<package>` explícito:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

Durante la transición del lanzamiento, las especificaciones de plugins sin prefijo que sean válidas para npm se instalan desde npm de forma predeterminada, salvo que coincidan con un id de plugin oficial:

```bash
openclaw plugins install openclaw-codex-app-server
```

Utilice `npm:` para hacer explícita la resolución exclusiva mediante npm:

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@openclaw/discord@2026.5.20
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw comprueba la compatibilidad anunciada con la API de plugins y la versión mínima del Gateway antes de la instalación. Cuando la versión seleccionada de ClawHub publica un artefacto ClawPack, OpenClaw descarga el `.tgz` de npm-pack versionado, verifica la cabecera de resumen de ClawHub y el resumen del artefacto y, a continuación, lo instala mediante la ruta normal para archivos. Las versiones anteriores de ClawHub sin metadatos de ClawPack siguen instalándose mediante la ruta heredada de verificación de archivos de paquetes. Las instalaciones registradas conservan sus metadatos de origen de ClawHub, el tipo de artefacto, la integridad de npm, el shasum de npm, el nombre del tarball y los datos del resumen de ClawPack para actualizaciones posteriores.
Las instalaciones de ClawHub sin versión conservan una especificación registrada sin versión para que `openclaw plugins update` pueda seguir las versiones más recientes de ClawHub; los selectores explícitos de versión o etiqueta, como `clawhub:pkg@1.2.3` y `clawhub:pkg@beta`, permanecen fijados a ese selector.

### Forma abreviada del marketplace

Utilice la forma abreviada `plugin@marketplace` cuando el nombre del marketplace exista en la caché local del registro de Claude en `~/.claude/plugins/known_marketplaces.json`:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

Utilice `--marketplace` para proporcionar explícitamente el origen del marketplace:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

<Tabs>
  <Tab title="Orígenes del marketplace">
    - un nombre de marketplace conocido por Claude de `~/.claude/plugins/known_marketplaces.json`
    - una raíz de marketplace local o una ruta `marketplace.json`
    - una forma abreviada de repositorio de GitHub, como `owner/repo`
    - una URL de repositorio de GitHub, como `https://github.com/owner/repo`
    - una URL de Git

  </Tab>
  <Tab title="Reglas de marketplaces remotos">
    Para los marketplaces remotos cargados desde GitHub o Git, las entradas de plugins deben permanecer dentro del repositorio clonado del marketplace. OpenClaw acepta orígenes de rutas relativas de ese repositorio y rechaza los orígenes de plugins HTTP(S), de rutas absolutas, Git, GitHub y otros orígenes que no sean rutas procedentes de manifiestos remotos.
  </Tab>
</Tabs>

Para rutas y archivos locales, OpenClaw detecta automáticamente:

- plugins nativos de OpenClaw (`openclaw.plugin.json`)
- paquetes compatibles con Codex (`.codex-plugin/plugin.json`)
- paquetes compatibles con Claude (`.claude-plugin/plugin.json`, o la disposición predeterminada de componentes de Claude cuando ese archivo de manifiesto no está presente)
- paquetes compatibles con Cursor (`.cursor-plugin/plugin.json`)

Las instalaciones locales administradas deben ser directorios o archivos de plugins. Los archivos de plugins independientes `.js`,
`.mjs`, `.cjs` y `.ts` no se copian en la raíz administrada de plugins
mediante `plugins install`, ni se cargan al colocarlos directamente en
`~/.openclaw/extensions` o `<workspace>/.openclaw/extensions`; esas
raíces detectadas automáticamente cargan directorios de paquetes o paquetes compatibles de plugins y omiten
los archivos de script de nivel superior por considerarlos auxiliares locales. En su lugar, enumere explícitamente los archivos independientes en
`plugins.load.paths`.

<Note>
Los paquetes compatibles se instalan en la raíz normal de plugins y participan en el mismo flujo de enumeración, información, habilitación y deshabilitación. Actualmente se admiten las Skills de los paquetes, las Skills de comandos de Claude, los valores predeterminados de `settings.json` de Claude, los valores predeterminados de `.lsp.json` de Claude y de `lspServers` declarados en el manifiesto, las Skills de comandos de Cursor y los directorios de hooks compatibles con Codex; otras capacidades detectadas de los paquetes se muestran en los diagnósticos y en la información, pero aún no están conectadas a la ejecución en el entorno de ejecución.
</Note>

Utilice `-l`/`--link` para apuntar a un directorio local de plugins sin copiarlo (se añade
a `plugins.load.paths`):

```bash
openclaw plugins install -l ./my-plugin
```

`--link` no es compatible con instalaciones `--marketplace` ni `git:`, y
requiere una ruta local que ya exista. Para crear un enlace local de forma no interactiva,
proporcione `--force` después de revisar el origen; esto confirma la procedencia, pero no
copia ni sobrescribe el directorio enlazado.

<Note>
Los plugins originados en un espacio de trabajo y detectados desde una raíz de extensiones del espacio de trabajo no se
importan ni ejecutan hasta que se habilitan explícitamente. Para el desarrollo local,
ejecute `openclaw plugins enable <plugin-id>` o configure
`plugins.entries.<plugin-id>.enabled: true`; si la configuración utiliza
`plugins.allow`, incluya también allí el mismo id de plugin. Esta regla de cierre seguro
también se aplica cuando la configuración de un canal apunta explícitamente a un plugin originado en el espacio de trabajo para
cargarlo únicamente durante la configuración, por lo que el código de configuración del plugin de canal local no se ejecutará mientras ese
plugin del espacio de trabajo permanezca deshabilitado o excluido de la lista de permitidos. Las instalaciones enlazadas
y las entradas explícitas de `plugins.load.paths` siguen la política normal correspondiente a su
origen de plugin resuelto. Consulte
[Configurar la política de plugins](/es/tools/plugin#configure-plugin-policy)
y la [Referencia de configuración](/es/gateway/configuration-reference#plugins).

Utilice `--pin` en las instalaciones de npm para guardar la especificación exacta resuelta (`name@version`) en el índice administrado de plugins, a la vez que se mantiene sin fijar el comportamiento predeterminado.
</Note>

## Enumerar

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

<ParamField path="--enabled" type="boolean">
  Mostrar solo los plugins habilitados.
</ParamField>
<ParamField path="--verbose" type="boolean">
  Cambiar de la vista de tabla a líneas de detalles por plugin con metadatos de formato, fuente, origen, versión y activación.
</ParamField>
<ParamField path="--json" type="boolean">
  Inventario legible por máquina, junto con diagnósticos del registro y el estado de instalación de las dependencias de paquetes.
</ParamField>

<Note>
`plugins list` lee primero el registro local persistente de plugins, con una alternativa derivada únicamente del manifiesto cuando el registro falta o no es válido. Resulta útil para comprobar si un plugin está instalado, habilitado y visible para la planificación del inicio en frío, pero no es una comprobación en vivo del tiempo de ejecución de un proceso de Gateway que ya está en ejecución. Después de cambiar el código, la habilitación o la política de hooks de un plugin, o `plugins.load.paths`, reinicie el Gateway que sirve el canal antes de esperar que se ejecuten el código o los hooks nuevos de `register(api)`. En implementaciones remotas o en contenedores, verifique que se esté reiniciando el proceso secundario `openclaw gateway run` real, no solo un proceso contenedor.

`plugins list --json` incluye el valor `dependencyStatus` de cada plugin procedente de `package.json`
`dependencies` y `optionalDependencies`. OpenClaw comprueba si esos nombres de
paquetes están presentes en la ruta de búsqueda normal de Node `node_modules` del plugin; no
importa el código de tiempo de ejecución del plugin, ejecuta un gestor de paquetes ni repara las
dependencias que faltan.
</Note>

Si el inicio registra `plugins.allow is empty; discovered non-bundled plugins may auto-load: ...`,
ejecute `openclaw plugins list --enabled --verbose` o
`openclaw plugins inspect <id>` con uno de los identificadores de plugin enumerados para confirmar los
identificadores de plugin y copie los identificadores de confianza en `plugins.allow` dentro de `openclaw.json`. Cuando la
advertencia pueda enumerar todos los plugins detectados, mostrará un fragmento
`plugins.allow` listo para pegar que ya incluye esos identificadores. Si un plugin se carga
sin procedencia de instalación o ruta de carga, inspeccione el identificador del plugin y, a continuación, fije
el identificador de confianza en `plugins.allow` o reinstale el plugin desde una fuente de confianza
para que OpenClaw registre la procedencia de la instalación.

Para trabajar con plugins incluidos dentro de una imagen de Docker empaquetada, monte mediante enlace el directorio
de origen del plugin sobre la ruta de origen empaquetada correspondiente, como
`/app/extensions/synology-chat`. OpenClaw detecta esa superposición de origen montada
antes de `/app/dist/extensions/synology-chat`; un directorio de origen simplemente copiado
permanece inactivo, por lo que las instalaciones empaquetadas normales siguen utilizando el código compilado de dist.

Para depurar hooks durante el tiempo de ejecución:

- `openclaw plugins inspect <id> --runtime --json` muestra los hooks registrados y los diagnósticos de una pasada de inspección con el módulo cargado. La inspección del tiempo de ejecución nunca instala dependencias; use `openclaw doctor --fix` para limpiar el estado de dependencias heredado o recuperar plugins descargables que falten y estén referenciados por la configuración.
- `openclaw gateway status --deep --require-rpc` confirma la URL o el perfil accesible del Gateway, las indicaciones del servicio o proceso, la ruta de configuración y el estado de la RPC.
- Los hooks de conversación no incluidos (`llm_input`, `llm_output`, `before_model_resolve`, `before_agent_reply`, `before_agent_run`, `before_agent_finalize`, `agent_end`) requieren `plugins.entries.<id>.hooks.allowConversationAccess=true`.

### Índice de plugins

Los metadatos de instalación de plugins son un estado gestionado por la máquina, no una configuración de usuario. Las instalaciones y actualizaciones los escriben en la base de datos de estado SQLite compartida, bajo el directorio de estado activo de OpenClaw. La fila `installed_plugin_index` almacena metadatos duraderos de `installRecords`, incluidos registros de manifiestos de plugins dañados o ausentes, además de una caché del registro en frío derivada de los manifiestos que utilizan `openclaw plugins update`, la desinstalación, los diagnósticos y el registro de plugins en frío.

`plugins.installs` es una superficie de configuración manual retirada. Los comandos de tiempo de ejecución y actualización solo leen el índice de plugins instalados de SQLite. Ejecute `openclaw doctor --fix` para importar los registros de configuración heredados en el índice y eliminar la clave retirada antes del uso normal en tiempo de ejecución.

## Desinstalación

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
openclaw plugins uninstall <id> --force
```

`uninstall` elimina los registros del plugin de `plugins.entries`, el índice persistente de plugins, las entradas de las listas de plugins permitidos y denegados y, cuando corresponde, las entradas vinculadas de `plugins.load.paths`. A menos que se establezca `--keep-files`, la desinstalación también elimina el directorio de instalación gestionado registrado, pero solo cuando este se resuelve dentro de la raíz de extensiones de plugins de OpenClaw. Si el plugin ocupa actualmente la ranura `memory` o `contextEngine`, esa ranura se restablece a su valor predeterminado (`memory-core` para la memoria y `legacy` para el motor de contexto).

`uninstall` muestra una vista previa de lo que se eliminará y, a continuación, solicita `Uninstall plugin "<id>"?` antes de realizar cambios. Pase `--force` para omitir la solicitud de confirmación (útil para scripts y ejecuciones no interactivas); sin esta opción, la desinstalación requiere una TTY interactiva. `--dry-run` muestra la misma vista previa y finaliza sin solicitar confirmación ni cambiar nada.

<Note>
`--keep-config` se admite como alias obsoleto de `--keep-files`.
</Note>

## Actualización

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call
openclaw plugins update @acme/demo
openclaw plugins update openclaw-codex-app-server --acknowledge-clawhub-risk
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

Las actualizaciones se aplican a las instalaciones de plugins registradas en el índice gestionado de plugins y a las instalaciones registradas de paquetes de hooks en el estado SQLite compartido. Reutilizan la fuente que el usuario ya eligió al instalar el plugin, por lo que no requieren una segunda confirmación de la fuente.

<AccordionGroup>
  <Accordion title="Resolución del identificador del plugin frente a la especificación de npm">
    Cuando se pasa un identificador de plugin, OpenClaw reutiliza la especificación de instalación registrada para ese plugin. Esto significa que las etiquetas de distribución almacenadas anteriormente, como `@beta`, y las versiones exactas fijadas siguen utilizándose en ejecuciones posteriores de `update <id>`.

    Durante `update <id> --dry-run`, las instalaciones de npm fijadas a una versión exacta permanecen fijadas. Si OpenClaw también puede resolver la línea predeterminada del registro del paquete y esta es más reciente que la versión fijada instalada, la simulación informa de la fijación y muestra el comando explícito de actualización del paquete `@latest` para seguir la línea predeterminada del registro.

    Esa regla de actualización dirigida difiere de la ruta de mantenimiento masivo `openclaw plugins update --all`. Las actualizaciones masivas siguen respetando las especificaciones de instalación registradas habituales, pero los registros de plugins oficiales de OpenClaw de confianza pueden sincronizarse con el destino actual del catálogo oficial en lugar de permanecer en un paquete oficial exacto obsoleto. Use la actualización dirigida `update <id>` cuando se quiera mantener intacta de forma intencionada una especificación oficial exacta o etiquetada.

    Para las instalaciones de npm, también se puede pasar una especificación explícita de paquete de npm con una etiqueta de distribución o una versión exacta. OpenClaw resuelve ese nombre de paquete al registro del plugin correspondiente, actualiza ese plugin instalado y registra la nueva especificación de npm para futuras actualizaciones basadas en el identificador.

    Pasar el nombre del paquete de npm sin una versión ni etiqueta también lo resuelve al registro del plugin correspondiente. Use esta opción cuando un plugin esté fijado a una versión exacta y se quiera devolver a la línea de versiones predeterminada del registro.

  </Accordion>
  <Accordion title="Actualizaciones del canal beta">
    La actualización dirigida `openclaw plugins update <id-or-npm-spec>` reutiliza la especificación registrada del plugin, a menos que se pase una especificación nueva. La actualización masiva `openclaw plugins update --all` utiliza el valor configurado de `update.channel` cuando sincroniza los registros de plugins oficiales de confianza con el destino del catálogo oficial, de modo que las instalaciones del canal beta pueden permanecer en la línea de versiones beta en lugar de normalizarse silenciosamente a stable/latest.

    `openclaw update` también conoce el canal de actualización activo de OpenClaw: en el canal beta, los registros de plugins de npm y ClawHub de la línea predeterminada prueban primero `@beta`. Utilizan como alternativa la especificación default/latest registrada si no existe una versión beta del plugin; los plugins de npm también recurren a la alternativa cuando el paquete beta existe, pero no supera la validación de instalación. Esa alternativa se notifica como advertencia y no hace que falle la actualización del núcleo. Las versiones exactas y las etiquetas explícitas permanecen fijadas a ese selector para las actualizaciones dirigidas.

  </Accordion>
  <Accordion title="Comprobaciones de versión y desviación de integridad">
    Antes de una actualización en vivo de npm, OpenClaw comprueba la versión del paquete instalado con los metadatos del registro de npm. Si la versión instalada y la identidad registrada del artefacto ya coinciden con el destino resuelto, la actualización se omite sin descargar, reinstalar ni reescribir `openclaw.json`.

    Cuando existe un hash de integridad almacenado y cambia el hash del artefacto obtenido, OpenClaw lo trata como una desviación del artefacto de npm. El comando interactivo `openclaw plugins update` muestra los hashes esperado y real y solicita confirmación antes de continuar. Los asistentes de actualización no interactivos se cierran de forma segura, a menos que el invocador proporcione una política explícita de continuación.

  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install en la actualización">
    `--dangerously-force-unsafe-install` también se acepta en `plugins update` por compatibilidad, pero está obsoleto y ya no modifica el comportamiento de actualización de los plugins. El valor `security.installPolicy` del operador aún puede bloquear las actualizaciones; los hooks `before_install` de los plugins solo se aplican en procesos donde estén cargados los hooks de plugins.
  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk en la actualización">
    Las actualizaciones de plugins de la comunidad respaldados por ClawHub ejecutan la misma comprobación de confianza de la versión exacta que las instalaciones antes de descargar el paquete de sustitución. Use `--acknowledge-clawhub-risk` para automatizaciones revisadas que deban continuar cuando la versión seleccionada de ClawHub presente una advertencia de confianza de riesgo. Los paquetes oficiales de ClawHub y las fuentes de plugins incluidas de OpenClaw omiten esta solicitud de confianza de la versión.
  </Accordion>
</AccordionGroup>

## Inspección

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
```

La inspección muestra la identidad, el estado de carga, la fuente, las capacidades del manifiesto, los indicadores de políticas, los diagnósticos, los metadatos de instalación, las capacidades del paquete y cualquier compatibilidad detectada con servidores MCP o LSP, sin importar de forma predeterminada el código de tiempo de ejecución del plugin. La salida JSON incluye los contratos del manifiesto del plugin, como `contracts.agentToolResultMiddleware` y `contracts.trustedToolPolicies`, para que los operadores puedan auditar las declaraciones de superficies de confianza antes de habilitar o reiniciar un plugin. Añada `--runtime` para cargar el módulo del plugin e incluir los hooks, las herramientas, los comandos, los servicios, los métodos del Gateway y las rutas HTTP registrados. La inspección del tiempo de ejecución informa directamente de las dependencias de plugins ausentes; las instalaciones y reparaciones permanecen en `openclaw plugins install`, `openclaw plugins update` y `openclaw doctor --fix`.

Los comandos de la CLI pertenecientes a plugins suelen instalarse como grupos de comandos raíz `openclaw`, pero los plugins también pueden registrar comandos anidados bajo un elemento principal del núcleo, como `openclaw nodes`. Después de que `inspect --runtime` muestre un comando bajo `cliCommands`, ejecútelo en la ruta indicada; por ejemplo, un plugin que registre `demo-git` puede verificarse con `openclaw demo-git ping`.

Cada plugin se clasifica según lo que registra realmente durante el tiempo de ejecución:

| Forma               | Significado                                                           |
| ------------------- | ----------------------------------------------------------------- |
| `plain-capability`  | exactamente un tipo de capacidad (p. ej., un plugin exclusivo de proveedor)         |
| `hybrid-capability` | más de un tipo de capacidad (p. ej., texto + voz + imágenes)       |
| `hook-only`         | solo hooks, sin capacidades, herramientas, comandos, servicios ni rutas |
| `non-capability`    | herramientas, comandos o servicios, pero sin capacidades                       |

Consulte [Formas de plugins](/es/plugins/architecture#plugin-shapes) para obtener más información sobre el modelo de capacidades.

<Note>
El indicador `--json` genera un informe legible por máquina adecuado para scripts y auditorías. `inspect --all` representa una tabla de toda la flota con columnas de forma, tipos de capacidades, avisos de compatibilidad, capacidades del paquete y resumen de hooks. `info` es un alias de `inspect`.
</Note>

## Diagnóstico

```bash
openclaw plugins doctor
```

`doctor` informa de errores de carga de plugins, diagnósticos de manifiesto/detección, avisos de compatibilidad y referencias obsoletas de configuración de plugins, como ranuras de plugins ausentes. Cuando el árbol de instalación y la configuración de plugins están limpios, muestra `No plugin issues detected.` Si queda configuración obsoleta, pero el árbol de instalación está en buen estado en los demás aspectos, el resumen lo indica en lugar de dar a entender que todos los plugins están en buen estado.

Si un plugin configurado está presente en el disco, pero lo bloquean las comprobaciones de seguridad de rutas del cargador, la validación de la configuración conserva la entrada del plugin e informa de ella como `present but blocked`. Corrija el diagnóstico anterior del plugin bloqueado, como la propiedad de la ruta o los permisos de escritura para todo el mundo, en lugar de eliminar la configuración `plugins.entries.<id>` o `plugins.allow`.

Para fallos de estructura de módulos, como la ausencia de exportaciones `register`/`activate`, vuelva a ejecutar con `OPENCLAW_PLUGIN_LOAD_DEBUG=1` para incluir un resumen compacto de la estructura de exportaciones en la salida de diagnóstico.

## Registro

```bash
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins registry --json
```

El registro local de plugins es el modelo persistente de lectura en frío de OpenClaw para la identidad de los plugins instalados, su habilitación, los metadatos de origen y la propiedad de las contribuciones. El inicio normal, la búsqueda del propietario del proveedor, la clasificación de la configuración de canales y el inventario de plugins pueden leerlo sin importar módulos de tiempo de ejecución de plugins.

Use `plugins registry` para comprobar si el registro persistente está presente, actualizado u obsoleto. Use `--refresh` para reconstruirlo a partir del índice persistente de plugins, la política de configuración y los metadatos del manifiesto/paquete. Esta es una ruta de reparación, no una ruta de activación en tiempo de ejecución.

`openclaw doctor --fix` también repara las divergencias de npm administrado adyacentes al registro. Si un paquete `@openclaw/*` huérfano o recuperado en un proyecto npm de plugins administrados o en la raíz plana heredada de npm administrado oculta un plugin incluido, Doctor elimina ese paquete obsoleto y reconstruye el registro para que el inicio valide el manifiesto incluido. Cuando un registro de instalación autoritativo selecciona una generación administrada, pero permanecen directorios planos o de generaciones anteriores, Doctor retira esos árboles obsoletos para su poda después de que se reinicie el Gateway. Doctor también vuelve a enlazar el paquete `openclaw` del host en los plugins npm administrados que declaran `peerDependencies.openclaw`, para que las importaciones de tiempo de ejecución locales del paquete, como `openclaw/plugin-sdk/*`, se resuelvan después de actualizaciones o reparaciones de npm.

## Mercado

```bash
openclaw plugins marketplace entries
openclaw plugins marketplace entries --offline
openclaw plugins marketplace entries --json
openclaw plugins marketplace entries --feed-profile <name>
openclaw plugins marketplace entries --feed-url <url>
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
openclaw plugins marketplace refresh
openclaw plugins marketplace refresh --feed-profile <name>
openclaw plugins marketplace refresh --feed-url <url>
openclaw plugins marketplace refresh --expected-sha256 <sha256> --json
```

`plugins marketplace entries` enumera las entradas del canal configurado del mercado de OpenClaw. De forma predeterminada, intenta usar el canal alojado y recurre a la instantánea aceptada más reciente o a los datos incluidos. Use `--feed-profile <name>` para leer un perfil configurado específico, `--feed-url <url>` para leer una URL explícita de un canal alojado y `--offline` para leer la instantánea aceptada más reciente sin obtener el canal.

`plugins marketplace refresh` actualiza la instantánea del canal alojado configurado e indica si OpenClaw aceptó datos alojados, una instantánea alojada o datos alternativos incluidos. Use `--expected-sha256` cuando quien invoque el comando necesite que este falle salvo que una carga útil alojada nueva coincida con una suma de comprobación fijada.

`list` del mercado acepta una ruta local del mercado, una ruta `marketplace.json`, una abreviatura de GitHub como `owner/repo`, una URL de repositorio de GitHub o una URL de git. `--json` muestra la etiqueta de origen resuelta, junto con el manifiesto analizado del mercado y las entradas de plugins.

La actualización del mercado carga un canal alojado del mercado de OpenClaw y conserva la
respuesta validada como instantánea local del canal alojado. Sin opciones, usa
el perfil de canal predeterminado configurado. Use `--feed-profile <name>` para actualizar un
perfil configurado específico, `--feed-url <url>` para actualizar una URL explícita de un
canal alojado, `--expected-sha256 <sha256>` para exigir una suma de comprobación coincidente de la carga útil
(`sha256:<hex>` o un resumen hexadecimal simple de 64 caracteres) y `--json` para obtener
una salida legible por máquinas. Las URL explícitas de canales alojados no deben incluir
credenciales, cadenas de consulta ni fragmentos. Las actualizaciones no fijadas pueden informar de una
instantánea alojada o de un resultado alternativo incluido sin que falle el comando. Las actualizaciones
fijadas fallan salvo que acepten una carga útil alojada nueva, y las actualizaciones alojadas
correctas fallan si OpenClaw no puede conservar la instantánea validada.

El perfil integrado `clawhub-public` espera la identidad de carga útil
`clawhub-official`. OpenClaw incluirá la clave pública de producción de ClawHub después de que
ClawHub genere y entregue esa clave. Hasta entonces, el perfil integrado no
otorga autoridad de instalación mediante un canal firmado. Las claves públicas deben proceder de una
versión de confianza o de un canal del operador, no de un punto de conexión de claves en el host del canal.

OpenClaw verifica el sobre DSSE y, cuando un perfil declara `feedId`,
exige que el ID de la carga útil decodificada coincida. El perfil integrado `clawhub-public`
siempre declara su identidad, lo que impide que un documento válido para otro
canal se reproduzca mediante ese perfil.

Durante el despliegue por etapas, los perfiles firmados personalizados existentes que omitan `feedId`
conservan la verificación de firmas sin vinculación de identidad de la carga útil. Los nuevos
perfiles personalizados deben declarar `feedId`. La superficie de configuración de perfiles de canal se
incorporará por separado con los metadatos de presentación que necesita Control UI; su
diagnóstico de Doctor debe solicitar al operador que proporcione una identidad ausente y no debe
inferirla a partir de la URL del canal. Esta vinculación de confianza no restaura la clave raíz
retirada `marketplaces`.

## Temas relacionados

- [Creación de plugins](/es/plugins/building-plugins)
- [Referencia de la CLI](/es/cli)
- [ClawHub](/es/clawhub)
