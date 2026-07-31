---
doc-schema-version: 1
read_when:
    - Desea explorar, instalar, habilitar o deshabilitar plugins en la interfaz de control
    - Quiere ejemplos rápidos para listar, instalar, actualizar, inspeccionar o desinstalar plugins
    - Quiere elegir una fuente de instalación del plugin
    - Quieres la referencia adecuada para publicar paquetes de plugins
sidebarTitle: Manage plugins
summary: Gestiona los plugins de OpenClaw desde la interfaz de control o la CLI
title: Gestionar plugins
x-i18n:
    generated_at: "2026-07-26T05:48:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9101d5c3630b618a043f1e71fdf5fa083698cc23694ccdc773d295a37c4c1ef3
    source_path: plugins/manage-plugins.md
    workflow: 16
---

La interfaz de control abarca el flujo de trabajo habitual de descubrimiento,
instalación, activación y desactivación. La CLI añade actualización,
desinstalación, configuración avanzada y controles explícitos de la fuente de
instalación. Para consultar el contrato completo de comandos, las opciones, las
reglas de selección de fuentes y los casos extremos, véase [`openclaw plugins`](/es/cli/plugins).

Flujo de trabajo típico de la CLI: buscar un paquete, instalarlo desde ClawHub,
npm, git o una ruta local, permitir que el Gateway administrado se reinicie
automáticamente (o reiniciarlo manualmente) y, a continuación, verificar los
registros del entorno de ejecución del plugin.

## Usar la interfaz de control

Abra **Plugins** en la interfaz de control o use `/settings/plugins` en relación
con la ruta base configurada de la interfaz de control. Por ejemplo, una ruta
base `/openclaw` usa `/openclaw/settings/plugins`. La página tiene dos pestañas:

- **Instalados** muestra el inventario local completo agrupado por categoría
(canales, proveedores de modelos, memoria y herramientas). Cada fila abre una
vista de detalles; su menú de desbordamiento (`…`) activa o
desactiva el plugin y, en el caso de los plugins instalados externamente, ofrece
**Eliminar**. La pestaña también muestra los [servidores MCP](/es/cli/mcp)
configurados con las mismas acciones de activación, desactivación y eliminación
mediante menús, que modifican `mcp.servers` en la configuración del Gateway.
- **Descubrir** es la tienda: plugins destacados incluidos con OpenClaw,
plugins externos oficiales y una selección de conectores. Las tarjetas de
conectores añaden un servidor MCP alojado con un solo clic (GitHub, Notion,
Linear, Sentry, Home Assistant) o abren una búsqueda de ClawHub previamente
rellenada. Al escribir en el cuadro de búsqueda, se consulta
[ClawHub](https://clawhub.ai/plugins) en la propia página y se añade una sección
**Desde ClawHub** con recuentos de descargas e insignias de verificación de la
fuente.

Los plugins incluidos no necesitan la instalación de un paquete. La acción de
su menú es **Activar** o **Desactivar**. Workboard, por ejemplo, está incluido
con OpenClaw y desactivado de forma predeterminada, por lo que debe seleccionarse
**Activar** para habilitarlo. Los plugins integrados no se pueden eliminar, solo
desactivar.

El acceso al catálogo y a la búsqueda requiere `operator.read`. Los cambios
de instalación, activación, desactivación, eliminación y servidores MCP
requieren `operator.admin`. El Gateway realiza las instalaciones desde
ClawHub y mantiene sus comprobaciones de confianza, integridad y políticas de
instalación de plugins. Cuando un administrador activa un plugin instalado,
también registra esa confianza explícita añadiendo el plugin seleccionado a una
lista restrictiva existente `plugins.allow`. Una entrada explícita
`plugins.deny` sigue teniendo prioridad y debe eliminarse antes de activar
el plugin.

La instalación o eliminación del código de un plugin requiere reiniciar el
Gateway. Los cambios de activación pueden aplicarse sin reiniciar cuando el
plugin instalado y el entorno de ejecución actual del Gateway lo permiten; de
lo contrario, la interfaz indica que se requiere un reinicio. Los conectores
MCP respaldados por OAuth siguen necesitando una ejecución única de
`openclaw mcp login <name>` desde la CLI después de añadirlos.

La interfaz de control no permite instalar desde fuentes arbitrarias de npm,
git o rutas locales, actualizar plugins ni acceder a una configuración avanzada
de plugins. Use los siguientes flujos de trabajo de la CLI para esas
operaciones.

## Mostrar y buscar plugins

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
openclaw plugins search "calendar"
```

`--json` para scripts:

```bash
openclaw plugins list --json \
  | jq '.plugins[] | {id, enabled, format, source, dependencyStatus}'
```

`plugins list` es una comprobación del inventario en frío: muestra lo que
OpenClaw puede detectar mediante la configuración, los manifiestos y el
registro persistente de plugins. No demuestra que un Gateway ya en ejecución
haya importado el entorno de ejecución del plugin. La salida JSON incluye
diagnósticos del registro y el `dependencyStatus` de cada plugin (si los
`dependencies`/`optionalDependencies` declarados se resuelven en el disco).

`plugins search` consulta ClawHub en busca de paquetes de plugins instalables
e imprime una sugerencia de instalación (`openclaw plugins install clawhub:<package>`) para cada
resultado.

## Activar y desactivar plugins

```bash
openclaw plugins enable <plugin-id>
openclaw plugins disable <plugin-id>
```

Alterna la entrada de configuración de un plugin sin modificar los archivos
instalados. Algunos plugins integrados (proveedores integrados de modelos o voz
y el plugin integrado del navegador) están activados de forma predeterminada;
otros requieren `enable` después de la instalación.

## Instalar plugins

```bash
# Buscar paquetes de plugins en ClawHub.
openclaw plugins search "calendar"

# Instalar desde ClawHub.
openclaw plugins install clawhub:<package>
openclaw plugins install clawhub:<package>@1.2.3
openclaw plugins install clawhub:<package>@beta

# Instalar desde npm.
openclaw plugins install npm:<package>
openclaw plugins install npm:@scope/openclaw-plugin@1.2.3
openclaw plugins install npm:@openclaw/codex

# Instalar desde un artefacto local de npm pack.
openclaw plugins install npm-pack:<path.tgz>

# Instalar desde git o una copia de desarrollo local.
openclaw plugins install git:github.com/acme/openclaw-plugin@v1.0.0
openclaw plugins install ./my-plugin
openclaw plugins install --link ./my-plugin
```

Las especificaciones de paquetes sin prefijo se instalan desde npm durante la
transición de lanzamiento, salvo que el nombre coincida con el identificador de
un plugin integrado u oficial; en ese caso, OpenClaw usa esa copia local u
oficial. Use `clawhub:`, `npm:`, `git:` o
`npm-pack:` para seleccionar la fuente de forma determinista. Los
paquetes integrados y oficiales del catálogo de OpenClaw se consideran de
confianza junto con los paquetes de ClawHub. Las fuentes nuevas y arbitrarias
de npm, git, rutas o archivos locales, `npm-pack:` o mercados requieren
`--force` en las instalaciones no interactivas después de revisar la
fuente y determinar que es de confianza.

`--force` confirma una fuente que no sea ClawHub sin solicitar
confirmación y sobrescribe un destino de instalación existente cuando es
necesario. Para las actualizaciones habituales de una instalación registrada
de npm, ClawHub o hook-pack, use `openclaw plugins update`. Con
`--link`, `--force` solo confirma la fuente; el directorio
enlazado no se copia ni se sobrescribe.

Si un plugin recién instalado requiere una configuración que aún no existe,
OpenClaw registra la instalación, pero deja el plugin desactivado. Configure
`plugins.entries.<id>.config` y, a continuación, ejecute `openclaw plugins enable <id>`. Si existe una
entrada de configuración, pero no es válida, la instalación falla sin
reescribirla.

## Reiniciar e inspeccionar

Un Gateway administrado en ejecución que tenga habilitada la recarga de la
configuración se reinicia automáticamente después de instalar, actualizar o
desinstalar el código de un plugin. Si el Gateway no está administrado o la
recarga está desactivada, reinícielo manualmente antes de comprobar las
superficies activas del entorno de ejecución:

```bash
openclaw gateway restart
openclaw plugins inspect <plugin-id> --runtime --json
```

`inspect --runtime` carga el módulo del plugin y demuestra que ha registrado
superficies del entorno de ejecución (herramientas, hooks, servicios, métodos
del Gateway, rutas HTTP y comandos de la CLI propiedad del plugin).
`inspect` y `list` sin más opciones solo realizan
comprobaciones en frío del manifiesto, la configuración y el registro.

## Actualizar plugins

```bash
openclaw plugins update <plugin-id>
openclaw plugins update <npm-package-or-spec>
openclaw plugins update --all
openclaw plugins update <plugin-id> --dry-run
```

Al proporcionar el identificador de un plugin, se reutiliza su especificación
de instalación registrada: las etiquetas de distribución almacenadas
(`@beta`) y las versiones exactas fijadas se conservan en las
ejecuciones posteriores de `update <plugin-id>`.

`openclaw plugins update --all` es la ruta de mantenimiento masivo. Sigue respetando las
especificaciones de instalación registradas habituales, pero los registros de
plugins oficiales de confianza de OpenClaw se sincronizan con el destino actual
del catálogo oficial en vez de permanecer fijados a un paquete oficial exacto
y obsoleto; cuando `update.channel` es `beta`, esa sincronización
prefiere la línea de versiones beta. Use una ejecución dirigida de
`update <plugin-id>` para mantener intacta una especificación oficial exacta o
etiquetada.

En las instalaciones desde npm, proporcione una especificación explícita del
paquete para cambiar el registro:

```bash
openclaw plugins update @scope/openclaw-plugin@beta
openclaw plugins update @scope/openclaw-plugin
```

El segundo comando devuelve un plugin a la línea de versiones predeterminada
del registro cuando anteriormente estaba fijado a una versión exacta o una
etiqueta.

Consulte [`openclaw plugins`](/es/cli/plugins#update) para conocer las reglas
exactas de alternativa y fijación de versiones.

## Desinstalar plugins

```bash
openclaw plugins uninstall <plugin-id> --dry-run
openclaw plugins uninstall <plugin-id>
openclaw plugins uninstall <plugin-id> --keep-files
```

La desinstalación elimina la entrada de configuración del plugin, el registro
persistente del índice de plugins, las entradas de las listas de permisos y
denegaciones y las entradas enlazadas de `plugins.load.paths` cuando corresponda.
El directorio de instalación administrado se elimina salvo que se proporcione
`--keep-files`. Un Gateway administrado en ejecución se reinicia
automáticamente cuando la desinstalación cambia la fuente del plugin.

En el modo Nix (`OPENCLAW_NIX_MODE=1`), la instalación, actualización,
desinstalación, activación y desactivación de plugins están deshabilitadas;
gestione esas opciones en la fuente Nix de la instalación.

## Elegir una fuente

| Fuente       | Cuándo usarla                                                                 | Ejemplo                                                        |
| ------------ | ----------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ClawHub      | Cuando se desea detección nativa de OpenClaw, resúmenes de análisis, versiones y sugerencias | `openclaw plugins install clawhub:<package>`                   |
| git          | Cuando se desea una rama, etiqueta o confirmación de un repositorio            | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| ruta local   | Cuando se desarrolla o prueba un plugin en la misma máquina                    | `openclaw plugins install --link ./my-plugin`                  |
| mercado      | Cuando se instala un plugin de mercado compatible con Claude                   | `openclaw plugins install <plugin> --marketplace <source>`     |
| npm pack     | Cuando se valida un artefacto de paquete local mediante la semántica de instalación de npm | `openclaw plugins install npm-pack:<path.tgz>`                 |
| npmjs.com    | Cuando ya se distribuyen paquetes de JavaScript o se necesitan etiquetas de distribución de npm o un registro privado | `openclaw plugins install npm:@acme/openclaw-plugin`           |

Las instalaciones administradas desde rutas locales deben ser directorios o
archivos de plugins. Coloque los archivos de plugins independientes en
`plugins.load.paths` en lugar de instalarlos con `plugins install`.

## Publicar plugins

ClawHub es la principal superficie pública de descubrimiento de plugins de
OpenClaw. Publique allí cuando desee que los usuarios encuentren los metadatos
del plugin, el historial de versiones, los resultados del análisis del registro
y las sugerencias de instalación antes de instalarlo.

```bash
npm i -g clawhub
clawhub login
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
clawhub package publish your-org/your-plugin@v1.0.0
```

Los plugins nativos de npm deben incluir un manifiesto del plugin
(`openclaw.plugin.json`) y metadatos `package.json` antes de publicarse:

```json package.json
{
  "name": "@acme/openclaw-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

```bash
npm publish --access public
openclaw plugins install npm:@acme/openclaw-plugin
openclaw plugins install npm:@acme/openclaw-plugin@beta
openclaw plugins install npm:@acme/openclaw-plugin@1.0.0
```

Use estas páginas para consultar el contrato completo de publicación en lugar
de considerar esta página como la referencia de publicación:

- [Publicación en ClawHub](/es/clawhub/publishing) explica los propietarios,
los ámbitos, las versiones, la revisión, la validación de paquetes y la
transferencia de paquetes.
- [Creación de plugins](/es/plugins/building-plugins) muestra la estructura
completa de los paquetes de plugins (incluido `openclaw.plugin.json`) y el flujo de
trabajo de la primera publicación.
- [Manifiesto del plugin](/es/plugins/manifest) define los campos del
manifiesto nativo de un plugin.

Si el mismo paquete está disponible tanto en ClawHub como en npm, use el prefijo
explícito `clawhub:` o `npm:` para forzar una fuente.

## Contenido relacionado

- [Plugins](/es/tools/plugin) - instalar, configurar, reiniciar y solucionar problemas
- [`openclaw plugins`](/es/cli/plugins) - referencia completa de la CLI
- [Plugins de la comunidad](/es/plugins/community) - descubrimiento público y publicación en ClawHub
- [ClawHub](/es/clawhub/cli) - operaciones de la CLI del registro
- [Creación de plugins](/es/plugins/building-plugins) - crear un paquete de plugin
- [Manifiesto de plugin](/es/plugins/manifest) - manifiesto y metadatos del paquete
