---
doc-schema-version: 1
read_when:
    - Instalación o configuración de plugins
    - Comprender las reglas de detección y carga de plugins
    - Trabajar con paquetes de plugins compatibles con Codex/Claude
sidebarTitle: Getting Started
summary: Instalar, configurar y gestionar plugins de OpenClaw
title: Plugins
x-i18n:
    generated_at: "2026-07-26T04:56:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f210dccab059527192eeb0aa2e780dcea243959273938ffaacc867ec96f5085e
    source_path: tools/plugin.md
    workflow: 16
---

Los Plugins amplían OpenClaw con canales, proveedores de modelos, entornos de agentes, herramientas,
Skills, voz, transcripción en tiempo real, comprensión de voz y contenido multimedia, generación,
obtención web, búsqueda web y otras capacidades de tiempo de ejecución.

Utilice esta página para instalar un Plugin, reiniciar el Gateway, verificar que el entorno de ejecución
lo haya cargado y resolver errores habituales de configuración. Para consultar ejemplos solo de comandos, véase
[Gestionar Plugins](/es/plugins/manage-plugins). Para consultar el inventario generado de
Plugins incluidos, externos oficiales y disponibles solo como código fuente, véase
[Inventario de Plugins](/es/plugins/plugin-inventory).

## Requisitos

- un checkout o una instalación de OpenClaw con la CLI `openclaw` disponible
- acceso de red a la fuente seleccionada (ClawHub, npm o un host de git)
- cualquier credencial, clave de configuración o herramienta del sistema operativo específica del Plugin indicada en la
  documentación de configuración de dicho Plugin
- permiso para que el Gateway que proporciona servicio a los canales se recargue o reinicie

## Inicio rápido

<Steps>
  <Step title="Encontrar el Plugin">
    Busque paquetes públicos de Plugins en [ClawHub](/es/clawhub):

    ```bash
    openclaw plugins search "calendar"
    ```

    ClawHub es la interfaz principal para descubrir Plugins de la comunidad. Durante la
    transición del lanzamiento, las especificaciones ordinarias de paquetes sin prefijo siguen instalándose desde npm, salvo que
    coincidan con el id de un Plugin oficial. Las especificaciones `@openclaw/*` sin procesar que coincidan con un
    Plugin incluido se resuelven a esa copia incluida. Utilice un prefijo de fuente explícito
    cuando necesite específicamente una fuente determinada.

  </Step>

  <Step title="Instalar el Plugin">
    ```bash
    # Desde ClawHub.
    openclaw plugins install clawhub:<package>

    # Desde npm.
    openclaw plugins install npm:<package>

    # Desde git.
    openclaw plugins install git:github.com/<owner>/<repo>@<ref>

    # Desde un checkout de desarrollo local.
    openclaw plugins install ./my-plugin
    openclaw plugins install --link ./my-plugin
    ```

    Trate las instalaciones de Plugins como la ejecución de código. Prefiera versiones fijadas para
    obtener instalaciones de producción reproducibles. Los paquetes de ClawHub y el catálogo
    incluido/oficial de OpenClaw son fuentes de confianza. Las nuevas fuentes arbitrarias de npm, git,
    rutas/archivos locales, `npm-pack:` o marketplace requieren
    `--force` en instalaciones no interactivas después de
    revisar la fuente y confiar en ella.

  </Step>

  <Step title="Configurar y habilitar el Plugin">
    Defina la configuración específica del Plugin en `plugins.entries.<id>.config`.
    Habilite el Plugin si aún no lo está:

    ```bash
    openclaw plugins enable <plugin-id>
    ```

    Si se establece `plugins.allow`, el id del Plugin instalado debe estar en esa lista
    para que el Plugin pueda cargarse. `openclaw plugins install` añade el
    id instalado a una lista `plugins.allow` existente y elimina ese mismo id de
    `plugins.deny`, de modo que la instalación explícita pueda cargarse tras el reinicio.

  </Step>

  <Step title="Permitir que el Gateway se recargue">
    Instalar, actualizar o desinstalar código de un Plugin requiere reiniciar el Gateway.
    Un Gateway gestionado con la recarga de configuración habilitada detecta el registro modificado
    de instalación del Plugin y se reinicia automáticamente. De lo contrario, reinícielo
    manualmente:

    ```bash
    openclaw gateway restart
    ```

    La activación o desactivación actualiza la configuración y el registro sin conexión. Una inspección del entorno de ejecución
    sigue siendo la prueba más clara de las superficies activas del entorno de ejecución.

  </Step>

  <Step title="Verificar el registro en el entorno de ejecución">
    ```bash
    openclaw plugins inspect <plugin-id> --runtime --json
    ```

    Utilice `--runtime` para comprobar las herramientas, los hooks, los servicios y los métodos del Gateway
    registrados, o los comandos de la CLI propiedad del Plugin. `inspect` sin opciones es únicamente una comprobación
    del manifiesto sin conexión y del registro.

  </Step>
</Steps>

## Configuración

### Elegir una fuente de instalación

| Fuente      | Utilícela cuando                                                               | Ejemplo                                                        |
| ----------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------- |
| ClawHub     | Se quiera usar el descubrimiento nativo de OpenClaw, los análisis, los metadatos de versiones y las indicaciones de instalación | `openclaw plugins install clawhub:<package>`                   |
| npm         | Se necesiten flujos de trabajo directos con el registro npm o etiquetas de distribución | `openclaw plugins install npm:<package>`                       |
| git         | Se necesite una rama, etiqueta o confirmación de un repositorio                | `openclaw plugins install git:github.com/<owner>/<repo>@<ref>` |
| ruta local  | Se esté desarrollando o probando un Plugin en la misma máquina                 | `openclaw plugins install --link ./my-plugin`                  |
| marketplace | Se esté instalando un Plugin de marketplace compatible con Claude              | `openclaw plugins install <plugin> --marketplace <source>`     |

Las especificaciones de paquetes sin prefijo tienen un comportamiento especial de compatibilidad: un nombre sin prefijo que
coincida con el id de un Plugin incluido utiliza esa fuente incluida; un nombre sin prefijo que coincida
con el id de un Plugin externo oficial utiliza el catálogo oficial de paquetes; cualquier otra
especificación sin prefijo se instala mediante npm durante la transición del lanzamiento. Las especificaciones `@openclaw/*`
sin procesar que coincidan con Plugins incluidos también se resuelven a la copia incluida antes de recurrir
a npm. Utilice `npm:@openclaw/<plugin>@<version>` para instalar deliberadamente el
paquete npm externo en lugar de la copia incluida. Utilice `clawhub:`, `npm:`,
`git:` o `npm-pack:` para seleccionar la fuente de forma determinista. Véase
[`openclaw plugins`](/es/cli/plugins#install) para consultar el contrato completo del comando.

Para instalaciones desde npm, las especificaciones sin versión fijada y `@latest` seleccionan el paquete
estable más reciente que anuncie compatibilidad con esta compilación de OpenClaw. Si la
versión más reciente actual de npm declara un `openclaw.compat.pluginApi` o
`openclaw.install.minHostVersion` más reciente de lo que admite esta compilación, OpenClaw examina
versiones estables anteriores e instala la más reciente que sea compatible. Las versiones exactas
y las etiquetas de canal explícitas, como `@beta`, permanecen fijadas al paquete seleccionado
y fallan cuando son incompatibles.

### Política de instalación del operador

Configure `security.installPolicy` para ejecutar un comando de política local de confianza
antes de que continúe la instalación o actualización de un Plugin. La política recibe metadatos junto con
la ruta de la fuente preparada y puede permitir o bloquear la instalación. Abarca tanto las rutas de
instalación/actualización mediante la CLI como las respaldadas por el Gateway. Los hooks `before_install` del Plugin se ejecutan
más tarde y solo en procesos de OpenClaw donde estén cargados los hooks de Plugins, por lo que debe utilizarse
`security.installPolicy` para las decisiones de instalación propiedad del operador. La
opción obsoleta `--dangerously-force-unsafe-install` se acepta por
compatibilidad, pero no realiza ninguna operación: no omite la política de instalación ni la lista de dependencias de Plugins
denegadas integrada en OpenClaw.

Véase [Configuración de Skills](/es/tools/skills-config#operator-install-policy-securityinstallpolicy)
para consultar el esquema de ejecución compartido `security.installPolicy` que utilizan tanto Skills como
Plugins.

### Configurar la política de Plugins

La estructura habitual de configuración de Plugins es:

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: ["untrusted-plugin"],
    load: { paths: ["~/Projects/oss/voice-call-plugin"] },
    slots: { memory: "memory-core" },
    entries: {
      "voice-call": { enabled: true, config: { provider: "twilio" } },
    },
  },
}
```

Reglas principales de la política:

- `plugins.enabled: false` deshabilita todos los Plugins y omite el trabajo de descubrimiento/carga.
  Las referencias obsoletas a Plugins permanecen inactivas mientras esta opción esté activa; vuelva a habilitar
  los Plugins antes de ejecutar la limpieza de doctor si desea eliminar los ids obsoletos.
- `plugins.deny` prevalece sobre la lista de permitidos y la habilitación individual de cada Plugin.
- `plugins.allow` es una lista exclusiva de permitidos. Las herramientas propiedad de Plugins que no estén en la
  lista de permitidos permanecen indisponibles incluso cuando `tools.allow` incluye `"*"`.
- `plugins.entries.<id>.enabled: false` deshabilita un Plugin sin eliminar su
  configuración.
- `plugins.load.paths` añade archivos o directorios locales explícitos de Plugins.
  Las rutas locales gestionadas mediante `plugins install` deben ser directorios o
  archivos de Plugins; utilice `plugins.load.paths` para archivos independientes de Plugins.
- Los Plugins procedentes del espacio de trabajo están deshabilitados de forma predeterminada; habilítelos explícitamente o
  añádalos a la lista de permitidos antes de utilizar código local del espacio de trabajo.
- Los Plugins incluidos siguen sus metadatos integrados de activación o desactivación predeterminada,
  salvo que la configuración los anule explícitamente.
- `plugins.slots.<slot>` (`memory` o `contextEngine`) selecciona un Plugin para una
  categoría exclusiva. La selección de un slot cuenta como activación explícita y
  fuerza la habilitación del Plugin seleccionado para ese slot, aunque de otro modo
  fuera opcional. `plugins.deny` y `plugins.entries.<id>.enabled: false` siguen
  bloqueándolo.
- Los Plugins incluidos opcionales pueden activarse automáticamente cuando la configuración menciona una de las
  superficies que poseen, como una referencia de proveedor/modelo, la configuración de un canal, un backend de la CLI
  o el entorno de ejecución de un agente.
- El enrutamiento de Codex de la familia OpenAI mantiene separados los límites del proveedor y del Plugin de tiempo de ejecución:
  las referencias de modelos Codex heredadas son configuración heredada que doctor repara,
  mientras que el Plugin incluido `codex` posee el entorno de ejecución del servidor de aplicaciones de Codex para
  referencias canónicas de agentes `openai/*`, `agentRuntime.id: "codex"` explícitas y
  referencias heredadas `codex/*`.

Cuando `plugins.allow` no está establecido y se descubren automáticamente Plugins no incluidos desde
el espacio de trabajo o las raíces globales de Plugins, el inicio registra
`plugins.allow is empty; discovered non-bundled plugins may auto-load: ...`
con los ids de los Plugins descubiertos y, en el caso de listas breves, un fragmento mínimo de `plugins.allow`.
Ejecute [`openclaw plugins list --enabled --verbose`](/es/cli/plugins#list)
o [`openclaw plugins inspect <id>`](/es/cli/plugins#inspect) con el id del
Plugin indicado antes de copiar Plugins de confianza en `openclaw.json`. La misma
fijación de confianza se aplica cuando los diagnósticos indican que un Plugin se cargó
`without install/load-path provenance`: inspeccione el id de ese Plugin y fíjelo después en
`plugins.allow`, o vuelva a instalarlo desde una fuente de confianza para que OpenClaw registre la
procedencia de la instalación.

Ejecute `openclaw doctor` o `openclaw doctor --fix` cuando la validación de la configuración
informe de ids obsoletos de Plugins, discrepancias entre la lista de permitidos y las herramientas o rutas heredadas de Plugins
incluidos.

## Comprender los formatos de Plugins

OpenClaw reconoce dos formatos de Plugins:

| Formato                  | Cómo se carga                                                                 | Utilícelo cuando                                                        |
| ------------------------ | ----------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Plugin nativo de OpenClaw | `openclaw.plugin.json` junto con un módulo de tiempo de ejecución cargado en el proceso | Se estén instalando o creando capacidades de tiempo de ejecución específicas de OpenClaw |
| Paquete compatible       | Diseño de Plugins de Codex, Claude o Cursor asignado al inventario de Plugins de OpenClaw | Se estén reutilizando Skills, comandos, hooks o metadatos de paquetes compatibles |

Ambos formatos aparecen en `openclaw plugins list`, `openclaw plugins inspect`,
`openclaw plugins enable` y `openclaw plugins disable`. Véase
[Paquetes de Plugins](/es/plugins/bundles) para consultar el límite de compatibilidad de paquetes y
[Creación de Plugins](/es/plugins/building-plugins) para crear Plugins nativos.

## Hooks de Plugins

Los Plugins pueden registrar hooks en tiempo de ejecución mediante dos API diferentes:

- `api.on(...)` hooks tipados para eventos del ciclo de vida del entorno de ejecución. Esta es la
  superficie preferida para middleware, políticas, reescritura de mensajes, definición de
  prompts y control de herramientas.
- `api.registerHook(...)` para el sistema interno de hooks descrito en
  [Hooks](/es/automation/hooks). Se utiliza principalmente para efectos secundarios generales de comandos o del ciclo de vida
  y para la compatibilidad con automatizaciones existentes de tipo HOOK.

Regla rápida: si el controlador necesita prioridad, semántica de combinación o
comportamiento de bloqueo/cancelación, utilice hooks tipados. Si solo reacciona a `command:new`,
`command:reset`, `message:sent` o eventos generales similares, `api.registerHook`
es adecuado.

Los hooks internos gestionados por Plugins aparecen en `openclaw hooks list` con
`plugin:<id>`. No se pueden habilitar ni deshabilitar mediante `openclaw hooks`;
habilite o deshabilite el Plugin en su lugar.

## Verificar el Gateway activo

`openclaw plugins list` y `openclaw plugins inspect` sin formato leen la configuración en frío y el estado
del manifiesto y del registro. No demuestran que un Gateway que ya está en ejecución
haya importado el mismo código del plugin.

Cuando un plugin aparece instalado, pero el tráfico de chat en vivo no lo utiliza:

```bash
openclaw gateway status --deep --require-rpc
openclaw plugins inspect <plugin-id> --runtime --json
openclaw gateway restart
```

Los Gateways administrados se reinician automáticamente después de instalar, actualizar y
desinstalar cambios que alteren el código fuente del plugin. En instalaciones en VPS o contenedores, hay que
asegurarse de que cualquier reinicio manual se dirija al proceso secundario `openclaw gateway run` real que
presta servicio a los canales, y no solo a un contenedor o supervisor.

## Solución de problemas

| Síntoma                                                        | Comprobación                                                                                                                                      | Solución                                                                                                     |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| El plugin aparece en `plugins list`, pero los hooks de tiempo de ejecución no se ejecutan  | Usar `openclaw plugins inspect <id> --runtime --json` y confirmar el Gateway activo con `gateway status --deep --require-rpc`             | Reiniciar el Gateway en vivo después de cambios de instalación, actualización, configuración o código fuente                               |
| Aparecen diagnósticos de propiedad duplicada de canales o herramientas         | Ejecutar `openclaw plugins list --enabled --verbose`, inspeccionar cada plugin sospechoso con `--runtime --json` y comparar la propiedad de canales/herramientas | Deshabilitar un propietario, eliminar instalaciones obsoletas o usar `preferOver` del manifiesto para un reemplazo intencional      |
| La configuración indica que falta un plugin                                | Consultar [Inventario de plugins](/es/plugins/plugin-inventory) para determinar si está incluido, es externo oficial o solo está disponible como código fuente                           | Instalar el paquete externo, habilitar el plugin incluido o eliminar la configuración obsoleta                         |
| La configuración no es válida durante la instalación                               | Leer el mensaje de validación y ejecutar `openclaw doctor --fix` si señala un estado obsoleto del plugin                                             | Doctor puede poner en cuarentena la configuración no válida del plugin deshabilitando la entrada y eliminando la carga útil no válida     |
| La ruta del plugin está bloqueada por una propiedad o permisos sospechosos | Inspeccionar el diagnóstico anterior al error de configuración                                                                                             | Corregir la propiedad o los permisos del sistema de archivos y, después, ejecutar `openclaw plugins registry --refresh`                    |
| `OPENCLAW_NIX_MODE=1` bloquea los comandos del ciclo de vida                | Confirmar que Nix administra la instalación                                                                                                      | Cambiar la selección del plugin en el código fuente de Nix en lugar de usar comandos de modificación de plugins                      |
| La importación de dependencias falla en tiempo de ejecución                             | Comprobar si el plugin se instaló mediante npm/git/ClawHub o se cargó desde una ruta local                                                 | Ejecutar `openclaw plugins update <id>`, reinstalar el código fuente o instalar manualmente las dependencias locales del plugin |

Cuando un plugin administrado habilitado no supera la verificación de la carga útil durante el inicio
del Gateway, OpenClaw pone en cuarentena esa raíz exacta del plugin instalado durante el arranque y
continúa prestando servicio a los demás plugins. `openclaw status --all`, `openclaw health`
y `openclaw doctor` lo notifican como `configured-unavailable`. Corregir o reinstalar
el plugin y, después, reiniciar el Gateway. Una sustitución explícita y correcta de `plugins.load.paths`
con el mismo id de plugin no queda en cuarentena por una instalación obsoleta y defectuosa.

Cuando la configuración obsoleta del plugin sigue mencionando un plugin de canal que ya no se puede detectar,
la validación de la configuración reduce esa clave del canal a una advertencia en lugar de provocar un
error crítico, de modo que el inicio del Gateway aún puede prestar servicio a todos los demás canales. Ejecutar
`openclaw doctor --fix` para eliminar las entradas obsoletas de plugins y canales. Las claves de canal
desconocidas sin indicios de plugins obsoletos siguen provocando un error de validación para que los errores
tipográficos permanezcan visibles.

Para reemplazar intencionalmente un canal, el plugin preferido debe declarar
`channelConfigs.<channel-id>.preferOver` con el id del plugin antiguo o de menor prioridad.
Si ambos plugins se habilitan explícitamente, OpenClaw conserva esa solicitud
y notifica diagnósticos de canales o herramientas duplicados en lugar de elegir
silenciosamente un propietario.

Si un paquete instalado informa que `requires compiled runtime output for
TypeScript entry ...`, el paquete se publicó sin los archivos JavaScript
que OpenClaw necesita en tiempo de ejecución. Actualizarlo o reinstalarlo después de que el editor publique
el JavaScript compilado, o deshabilitar/desinstalar el plugin hasta entonces.

### Propiedad bloqueada de la ruta del plugin

Si los diagnósticos indican
`blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)`
y la validación continúa con `plugin present but blocked`, OpenClaw ha encontrado
archivos de plugins que pertenecen a un usuario de Unix diferente del proceso que los carga.
Mantener la configuración del plugin; corregir la propiedad del sistema de archivos o ejecutar OpenClaw
con el mismo usuario propietario del directorio de estado.

En instalaciones con Docker, la imagen oficial se ejecuta como `node` (uid `1000`), por lo que los
directorios de configuración y espacio de trabajo de OpenClaw montados mediante enlace desde el host normalmente deben
pertenecer al uid `1000`:

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

Si OpenClaw se ejecuta intencionalmente como root, hay que reparar la raíz del plugin administrado para que
pertenezca a root:

```bash
sudo chown -R root:root /path/to/openclaw-config/npm
```

Después de corregir la propiedad, volver a ejecutar `openclaw doctor --fix` o
`openclaw plugins registry --refresh` para que el registro persistente de plugins
coincida con los archivos reparados.

### Configuración lenta de las herramientas del plugin

Si los turnos del agente parecen bloquearse mientras se preparan las herramientas, habilitar el registro de rastreo
y buscar las líneas de tiempo de las fábricas de herramientas del plugin:

```bash
openclaw config set logging.level trace
openclaw logs --follow
```

Buscar:

```text
[trace:plugin-tools] tiempos de las fábricas ...
```

El resumen muestra el tiempo total de las fábricas y las fábricas de herramientas de plugins más lentas,
incluidos el id del plugin, los nombres declarados de las herramientas, la forma del resultado y si la herramienta
es opcional. Las líneas lentas se convierten en advertencias cuando una sola fábrica tarda
al menos 1s o la preparación total de las fábricas de herramientas de plugins tarda al menos 5s.

OpenClaw almacena en caché los resultados correctos de las fábricas de herramientas de plugins para resoluciones
repetidas con el mismo contexto efectivo de solicitud. La clave de caché incluye
la configuración efectiva del tiempo de ejecución, el espacio de trabajo y el id del agente, la política del sandbox, la configuración
del navegador, el contexto de entrega, la identidad del solicitante y el estado de propiedad, por lo que
las fábricas que dependen de esos campos de confianza vuelven a ejecutarse cuando cambia el contexto.
Si los tiempos siguen siendo altos, es posible que el plugin esté realizando tareas costosas antes
de devolver las definiciones de sus herramientas.

Si un plugin domina los tiempos, hay que inspeccionar sus registros en tiempo de ejecución:

```bash
openclaw plugins inspect <plugin-id> --runtime --json
```

Después, actualizar, reinstalar o deshabilitar ese plugin. Los autores de plugins deben trasladar
la carga costosa de dependencias a la ruta de ejecución de la herramienta en lugar de realizarla
dentro de la fábrica de herramientas.

Para obtener información sobre las raíces de dependencias, la validación de metadatos de paquetes, los registros
del registro, el comportamiento de recarga al inicio y la limpieza de elementos heredados, consultar
[Resolución de dependencias de plugins](/es/plugins/dependency-resolution).

## Contenido relacionado

- [Administrar plugins](/es/plugins/manage-plugins) - ejemplos de comandos para enumerar, instalar, actualizar, desinstalar y publicar
- [`openclaw plugins`](/es/cli/plugins) - referencia completa de la CLI
- [Inventario de plugins](/es/plugins/plugin-inventory) - lista generada de plugins incluidos y externos
- [Referencia de plugins](/es/plugins/reference) - páginas de referencia generadas para cada plugin
- [Plugins de la comunidad](/es/plugins/community) - política de descubrimiento en ClawHub y pull requests de documentación
- [Resolución de dependencias de plugins](/es/plugins/dependency-resolution) - raíces de instalación, registros del registro y límites del tiempo de ejecución
- [Creación de plugins](/es/plugins/building-plugins) - guía para la creación nativa de plugins
- [Descripción general del SDK de plugins](/es/plugins/sdk-overview) - registro del tiempo de ejecución, hooks y campos de la API
- [Manifiesto del plugin](/es/plugins/manifest) - manifiesto y metadatos del paquete
