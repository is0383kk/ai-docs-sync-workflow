---
read_when:
    - Está depurando las instalaciones de paquetes de plugins
    - Está cambiando el comportamiento de inicio del plugin, de doctor o de instalación del gestor de paquetes
    - Está realizando el mantenimiento de instalaciones empaquetadas de OpenClaw o de manifiestos de plugins incluidos.
sidebarTitle: Dependencies
summary: Cómo OpenClaw instala paquetes de plugins y resuelve las dependencias de los plugins
title: Resolución de dependencias de Plugins
x-i18n:
    generated_at: "2026-07-26T04:45:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae24a82568e275399cb7b68729d2805956792852612f84d6918850305f0eb243
    source_path: plugins/dependency-resolution.md
    workflow: 16
---

OpenClaw gestiona las dependencias de los plugins únicamente durante la instalación o actualización. La carga en
tiempo de ejecución nunca ejecuta un gestor de paquetes, repara un árbol de dependencias ni modifica
el directorio del paquete de OpenClaw.

## División de responsabilidades

Los paquetes de plugins son responsables de su grafo de dependencias:

- Las dependencias de tiempo de ejecución se encuentran en el `dependencies` o
  `optionalDependencies` del paquete del plugin.
- Las importaciones del SDK o del núcleo son importaciones pares o proporcionadas por OpenClaw.
- Los plugins de desarrollo local proporcionan sus propias dependencias ya instaladas.
- Los plugins de npm y git se instalan en raíces de paquetes propiedad de OpenClaw.

OpenClaw solo es responsable del ciclo de vida de los plugins:

- Detectar el origen del plugin.
- Instalar o actualizar el paquete cuando se solicite explícitamente.
- Registrar los metadatos de instalación.
- Cargar el punto de entrada del plugin.
- Generar un error que indique cómo actuar cuando falten dependencias.

## Raíces de instalación

OpenClaw utiliza raíces estables para cada origen:

- Los paquetes de npm se instalan en proyectos individuales por plugin bajo
  `~/.openclaw/npm/projects/<encoded-package>`.
- Los paquetes de git se clonan bajo `~/.openclaw/git`.
- Las instalaciones locales, desde rutas o archivos se copian o referencian sin reparar
  dependencias.

Las instalaciones de npm se ejecutan en la raíz de ese proyecto individual por plugin con:

```bash
cd ~/.openclaw/npm/projects/<encoded-package>
npm install --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts --no-audit --no-fund
```

`openclaw plugins install npm-pack:<path.tgz>` utiliza la misma raíz de proyecto npm individual por
plugin para un archivo tar local generado por npm pack: OpenClaw lee los metadatos de npm
del archivo tar, lo añade al proyecto gestionado como una dependencia `file:` copiada, ejecuta
la instalación normal de npm anterior y, a continuación, verifica los metadatos del archivo de bloqueo instalado
antes de confiar en el plugin. Esta ruta existe para la aceptación de paquetes y
la prueba de candidatos de lanzamiento, en las que un artefacto de empaquetado local debe comportarse como el
artefacto del registro que simula.

Utilice `npm-pack:` al probar paquetes de plugins oficiales o externos antes de
publicarlos. Una instalación desde un archivo sin procesar o una ruta resulta útil para la depuración local, pero
no demuestra la misma ruta de dependencias que un paquete de npm o ClawHub
instalado. `npm-pack:` demuestra la estructura de instalación del paquete gestionado; por
sí solo, no demuestra que el plugin sea contenido oficial vinculado al catálogo.

Cuando el comportamiento dependa del estado de plugin incluido o plugin oficial de confianza,
combine la prueba del paquete local con una instalación oficial respaldada por el catálogo o una
ruta de paquete publicado que registre la confianza oficial. El acceso a asistentes privilegiados
y la gestión del ámbito oficial de confianza deben validarse en esa ruta de instalación
de confianza, no inferirse a partir de la instalación de un archivo tar local.

Si un plugin falla durante el tiempo de ejecución debido a una importación ausente, corrija el manifiesto del paquete
en lugar de reparar manualmente el proyecto gestionado. Las importaciones de tiempo de ejecución deben estar en
el `dependencies` o `optionalDependencies` del paquete del plugin; `devDependencies`
no se instalan en los proyectos de tiempo de ejecución gestionados. Un `npm install` local dentro de
`~/.openclaw/npm/projects/<encoded-package>` puede desbloquear un diagnóstico
temporal, pero no constituye una prueba de aceptación del paquete porque la siguiente instalación o
actualización vuelve a crear el proyecto a partir de los metadatos del paquete.

npm puede elevar dependencias transitivas al
`node_modules` del proyecto individual por plugin junto al paquete del plugin. OpenClaw analiza la raíz del proyecto gestionado
antes de confiar en la instalación y elimina ese proyecto al desinstalarlo, por lo que
las dependencias de tiempo de ejecución elevadas permanecen dentro del límite de limpieza de ese plugin.

Los paquetes de plugins de npm publicados pueden incluir `npm-shrinkwrap.json`; npm utiliza ese
archivo de bloqueo publicable durante la instalación, y la raíz del proyecto npm gestionado de OpenClaw
lo admite mediante la ruta de instalación normal. Los paquetes de plugins publicables
propiedad de OpenClaw deben incluir un shrinkwrap local del paquete generado a partir del
grafo de dependencias publicado de ese paquete:

```bash
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check
```

El generador elimina los `devDependencies` del plugin, aplica la política de sustitución
del espacio de trabajo y escribe `extensions/<id>/npm-shrinkwrap.json` para cada plugin con
`openclaw.release.publishToNpm: true`. Los paquetes de plugins de terceros también pueden
incluir un shrinkwrap; OpenClaw no lo exige para los paquetes de la comunidad, pero
npm lo respeta cuando está presente.

Antes de considerar un paquete local como prueba de candidato de lanzamiento, inspeccione el
archivo tar que se instalará:

```bash
npm pack --pack-destination /tmp
tar -xOf /tmp/<plugin-package>.tgz package/package.json
tar -tf /tmp/<plugin-package>.tgz | grep '^package/dist/'
```

Para cambios en las dependencias, compruebe también que una instalación de producción pueda resolver los
paquetes de tiempo de ejecución sin dependencias de desarrollo:

```bash
tmpdir=$(mktemp -d)
(
  cd "$tmpdir"
  npm init -y >/dev/null
  npm install --package-lock-only --omit=dev --omit=peer --legacy-peer-deps --ignore-scripts /tmp/<plugin-package>.tgz
)
rm -rf "$tmpdir"
```

Los paquetes de plugins de npm propiedad de OpenClaw también pueden publicarse con
`bundledDependencies` explícito. La ruta de publicación de npm superpone la lista de nombres de dependencias
de tiempo de ejecución, elimina del manifiesto publicado los metadatos del espacio de trabajo exclusivos para desarrollo,
ejecuta una instalación de npm sin scripts para las dependencias de tiempo de ejecución locales del paquete
y, a continuación, empaqueta o publica el archivo tar del plugin con esos archivos de dependencias
incluidos. Los paquetes con muchos componentes nativos (Codex, ACPX, Copilot, llama.cpp,
memory-lancedb, Tlon) quedan excluidos mediante
`openclaw.release.bundleRuntimeDependencies: false`; siguen incluyendo un
shrinkwrap, pero npm resuelve las dependencias de tiempo de ejecución durante la instalación en lugar de
incorporar todos los binarios de plataforma en el archivo tar del plugin. El paquete raíz `openclaw`
no incluye todo su árbol de dependencias.

Los plugins que importan `openclaw/plugin-sdk/*` declaran `openclaw` como dependencia
par. OpenClaw no permite que npm instale una copia independiente del paquete
host desde el registro en un proyecto gestionado, porque un paquete host obsoleto puede afectar
a la resolución de dependencias pares de npm dentro de ese plugin. Las instalaciones de npm gestionadas omiten la
resolución y materialización de dependencias pares de npm, y OpenClaw restablece los enlaces
`node_modules/openclaw` locales del plugin para los paquetes instalados que declaran el host
como dependencia par, después de la instalación o actualización.

Las instalaciones de git clonan o actualizan el repositorio y, a continuación, ejecutan:

```bash
npm install --omit=dev --ignore-scripts --no-audit --no-fund
```

El plugin instalado se carga entonces desde el directorio de ese paquete, por lo que
la resolución de `node_modules` locales del paquete y superiores funciona igual que
en un paquete normal de Node.

## Plugins locales

Los plugins locales son directorios controlados por los desarrolladores. OpenClaw nunca ejecuta
`npm install`, `pnpm install` ni la reparación de dependencias en ellos; si un plugin
local tiene dependencias, instálelas en ese plugin antes de cargarlo.

Los plugins locales de TypeScript de terceros se cargan mediante Jiti como ruta de emergencia.
Los plugins JavaScript empaquetados y los plugins internos incluidos se cargan mediante
import/require nativos.

## Inicio y recarga

El inicio del Gateway y la recarga de la configuración nunca instalan dependencias de plugins. Estos procesos
leen los registros de instalación del plugin, calculan el punto de entrada y lo cargan.

Una dependencia ausente durante el tiempo de ejecución provoca un error de carga del plugin que indica
al operador una corrección explícita:

```bash
openclaw plugins update <id>
openclaw plugins install <source>
openclaw doctor --fix
```

`doctor --fix` limpia el estado de dependencias heredado generado por OpenClaw y puede
recuperar plugins descargables que faltan en los registros de instalación locales cuando
la configuración aún hace referencia a ellos. Doctor no repara las dependencias de un
plugin local ya instalado.

## Plugins incluidos

Los plugins incluidos ligeros y esenciales para el núcleo se distribuyen como parte de OpenClaw. Deben
carecer de un árbol pesado de dependencias de tiempo de ejecución o trasladarse a un
paquete descargable en ClawHub/npm.

Para consultar la lista generada actual de plugins incluidos en el paquete del núcleo,
instalados externamente o disponibles solo como código fuente, consulte
[Inventario de plugins](/es/plugins/plugin-inventory).

Los manifiestos de plugins incluidos no deben solicitar la preparación de dependencias. Las funciones
de plugins grandes u opcionales deben empaquetarse como un plugin normal e
instalarse mediante la misma ruta npm/git/ClawHub que los plugins de terceros.

En los checkouts del código fuente, OpenClaw trata el repositorio como un monorepo de pnpm.
Después de `pnpm install`, los plugins incluidos se cargan desde `extensions/<id>` para que
las dependencias locales de los paquetes del espacio de trabajo estén disponibles y los cambios se apliquen
directamente. El desarrollo desde un checkout del código fuente utiliza exclusivamente pnpm; ejecutar `npm install` sin más en
la raíz del repositorio no prepara las dependencias de los plugins incluidos.

| Estructura de instalación                    | Ubicación del plugin incluido               | Responsable de las dependencias                                                     |
| -------------------------------- | ------------------------------------- | -------------------------------------------------------------------- |
| `npm install -g openclaw`        | Árbol de tiempo de ejecución compilado dentro del paquete | Paquete de OpenClaw y flujos explícitos de instalación, actualización y Doctor de plugins     |
| Checkout de git más `pnpm install` | Paquetes del espacio de trabajo `extensions/<id>`  | El espacio de trabajo de pnpm, incluidas las dependencias propias de cada paquete de plugin |
| `openclaw plugins install ...`   | Raíz gestionada de proyecto npm, git o ClawHub  | El flujo de instalación o actualización del plugin                                       |

## Limpieza heredada

Las versiones anteriores de OpenClaw generaban raíces de dependencias de plugins incluidos durante el inicio
o la reparación con Doctor. La limpieza actual de Doctor elimina esos
directorios y enlaces simbólicos obsoletos mediante `--fix`, incluidas las raíces `plugin-runtime-deps`
antiguas, los enlaces simbólicos de paquetes del prefijo global de Node que apuntan a destinos
`plugin-runtime-deps` eliminados, los manifiestos `.openclaw-runtime-deps*`, los `node_modules`
generados de los plugins, los directorios de preparación de instalación y los almacenes de pnpm
locales de los paquetes. El script postinstall empaquetado también elimina esos enlaces simbólicos globales antes de
eliminar las raíces de destino heredadas, para que las actualizaciones no dejen importaciones de paquetes ESM
colgantes.

Las instalaciones antiguas de npm también utilizaban una raíz `~/.openclaw/npm/node_modules` compartida.
Los flujos actuales de instalación, actualización, desinstalación y Doctor siguen reconociendo esa
raíz plana heredada únicamente para recuperación y limpieza. Las nuevas instalaciones de npm crean
raíces de proyectos individuales por plugin.
