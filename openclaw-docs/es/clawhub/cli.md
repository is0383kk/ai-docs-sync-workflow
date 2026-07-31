---
read_when:
    - Uso de la CLI de ClawHub
    - Depuración de la instalación, la actualización o la publicación
summary: 'Referencia de la CLI: comandos, indicadores, configuración y comportamiento del archivo de bloqueo.'
x-i18n:
    generated_at: "2026-07-26T05:04:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

Paquete de CLI: `clawhub`, binario: `clawhub`.

Instálelo globalmente con npm o pnpm:

```bash
npm i -g clawhub
# o
pnpm add -g clawhub
```

A continuación, verifíquelo:

```bash
clawhub --help
clawhub login
clawhub whoami
```

## Opciones globales

- `--workdir <dir>`: directorio de trabajo (predeterminado: cwd; recurre al espacio de trabajo de Clawdbot si está configurado)
- `--dir <dir>`: directorio de instalación dentro del directorio de trabajo (predeterminado: `skills`)
- `--site <url>`: URL base para iniciar sesión en el navegador (predeterminada: `https://clawhub.ai`)
- `--registry <url>`: URL base de la API (predeterminada: detectada; en caso contrario, `https://clawhub.ai`)
- `--no-input`: desactiva las solicitudes interactivas

Equivalentes de entorno:

- `CLAWHUB_SITE` (heredado: `CLAWDHUB_SITE`)
- `CLAWHUB_REGISTRY` (heredado: `CLAWDHUB_REGISTRY`)
- `CLAWHUB_WORKDIR` (heredado: `CLAWDHUB_WORKDIR`)

### Proxy HTTP

La CLI respeta las variables de entorno estándar de proxy HTTP en sistemas situados detrás de
proxies corporativos o redes restringidas:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

Cuando se establece cualquiera de estas variables, la CLI dirige las solicitudes salientes a través
del proxy especificado. `HTTPS_PROXY` se utiliza para las solicitudes HTTPS y `HTTP_PROXY`
para HTTP sin cifrar. Se respeta `NO_PROXY` / `no_proxy` para omitir el proxy en
hosts o dominios específicos.

Esto es necesario en sistemas donde las conexiones salientes directas están bloqueadas
(p. ej., contenedores Docker, VPS de Hetzner con acceso a Internet exclusivamente mediante proxy
o cortafuegos corporativos).

Ejemplo:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "mi consulta"
```

Cuando no se establece ninguna variable de proxy, el comportamiento no cambia (conexiones directas).

## Archivo de configuración

Almacena el token de API y la URL del registro en caché.

- macOS: `~/Library/Application Support/clawhub/config.json`
- Linux/XDG: `$XDG_CONFIG_HOME/clawhub/config.json` o `~/.config/clawhub/config.json`
- Windows: `%APPDATA%\\clawhub\\config.json`
- Alternativa heredada: si `clawhub/config.json` todavía no existe, pero `clawdhub/config.json` sí, la CLI reutiliza la ruta heredada
- anulación: `CLAWHUB_CONFIG_PATH` (heredado: `CLAWDHUB_CONFIG_PATH`)

## Comandos

### `login` / `auth login`

- Predeterminado: abre el navegador en `<site>/cli/auth` y completa el proceso mediante una devolución de llamada de bucle invertido.
- Sin interfaz gráfica: `clawhub login --token clh_...`
- Interactivo remoto/sin interfaz gráfica: `clawhub login --device` muestra un código y espera mientras se autoriza en `<site>/cli/device`.

### `whoami`

- Verifica el token almacenado mediante `/api/v1/whoami`.

### `token`

- Imprime el token de API almacenado en stdout.
- Resulta útil para canalizar un token de inicio de sesión local a comandos de configuración de secretos de CI.

### `star <skill>` / `unstar <skill>`

- Añade o elimina una skill de los marcadores. Los nombres de los comandos siguen siendo `star` y
  `unstar` por compatibilidad.
- Llama a `POST /api/v1/stars/<slug>` y `DELETE /api/v1/stars/<slug>`.
- `--yes` omite la confirmación.

### `search <query...>`

- Llama a `/api/v1/search?q=...`.
- La salida incluye el slug de la skill, el identificador del propietario, el nombre para mostrar y la puntuación de relevancia.
- La búsqueda favorece las coincidencias exactas de tokens del slug o nombre frente a la popularidad de descargas. Un token de slug independiente como `map` coincide con `personal-map` con mayor fuerza que con la subcadena dentro de `amap`.
- La popularidad es un factor previo de clasificación de poca importancia, no una garantía de ocupar las primeras posiciones.
- Si una skill debería aparecer, pero no lo hace, ejecute `clawhub inspect @owner/slug` mientras mantiene la sesión iniciada para comprobar los diagnósticos de moderación visibles para el propietario antes de cambiar el nombre de los metadatos.

### `explore`

- Enumera las skills más recientes mediante `/api/v1/skills?limit=...&sort=createdAt` (ordenadas por `createdAt` de forma descendente).
- Opciones:
  - `--limit <n>` (1-200, predeterminado: 25)
  - `--sort newest|updated|rating|downloads|trending` (predeterminado: más recientes). Los alias heredados de ordenación de instalaciones siguen funcionando por compatibilidad.
  - `--json` (salida legible por máquina)
- Salida: `<slug>  v<version>  <age>  <summary>` (resumen truncado a 50 caracteres).

### `inspect @owner/slug`

- Obtiene los metadatos y los archivos de versión de una skill sin instalarla.
- `--version <version>`: inspecciona una versión específica (predeterminada: la más reciente).
- `--tag <tag>`: inspecciona una versión etiquetada (p. ej., `latest`).
- `--versions`: enumera el historial de versiones (primera página).
- `--limit <n>`: número máximo de versiones que se enumerarán (1-200).
- `--files`: enumera los archivos de la versión seleccionada.
- `--file <path>`: obtiene los bytes sin procesar del archivo (límite de 10MB).
- `--json`: salida legible por máquina; `--file` incluye los bytes exactos como base64 y texto UTF-8 cuando está disponible.

### `install @owner/slug`

- Resuelve la versión más reciente para el propietario y la skill indicados.
- Descarga el archivo ZIP mediante `/api/v1/download`.
- Lo extrae en `<workdir>/<dir>/<slug>`.
- Se niega a sobrescribir las skills fijadas; ejecute primero `clawhub unpin <skill>`.
- Escribe:
  - `<workdir>/.clawhub/lock.json` (heredado: `.clawdhub`)
  - `<skill>/.clawhub/origin.json` (heredado: `.clawdhub`)

### `uninstall <skill>`

- Elimina `<workdir>/<dir>/<slug>` y borra la entrada del archivo de bloqueo.
- Envía telemetría según el mejor esfuerzo mientras la sesión está iniciada para que los recuentos actuales de instalaciones puedan
  desactivarse.
- Interactivo: solicita confirmación.
- No interactivo (`--no-input`): requiere `--yes`.

### `list`

- Lee `<workdir>/.clawhub/lock.json` (heredado: `.clawdhub`).
- Muestra `pinned` junto a las skills inmovilizadas con `clawhub pin`, incluido el motivo opcional.

### `pin <skill>`

- Marca una skill instalada como fijada en el archivo de bloqueo.
- `--reason <text>` registra por qué está inmovilizada la skill.
- Las skills fijadas se omiten en `update --all` y las rechaza directamente `update <skill>`.
- Las skills fijadas también rechazan `install --force` para impedir que los bytes locales se sustituyan accidentalmente.

### `unpin <skill>`

- Elimina del archivo de bloqueo la fijación de una skill instalada para que las actualizaciones futuras puedan modificarla.

### `update [@owner/slug]` / `update --all`

- Calcula la huella digital a partir de los archivos locales.
- Si la huella digital coincide con una versión conocida: no se solicita confirmación.
- Si la huella digital no coincide:
  - se niega de forma predeterminada
  - sobrescribe con `--force` (o solicita confirmación si es interactivo)
- Las skills fijadas nunca se actualizan mediante `--force`.
- `update <skill>` falla de inmediato para las skills fijadas e indica que se ejecute primero `clawhub unpin <skill>`.
- `update --all` omite los slugs fijados y muestra un resumen de lo que permaneció inmovilizado.

### `skill publish <path>`

- Compara la huella digital del paquete local con ClawHub y finaliza correctamente cuando
  el contenido ya está publicado.
- Las skills nuevas usan de forma predeterminada `1.0.0`; las skills modificadas usan de forma predeterminada la siguiente
  versión de parche.
- `--version <version>` selecciona explícitamente una versión y la publica incluso cuando el
  contenido coincide con una versión existente.
- `--dry-run` resuelve la publicación sin cargar contenido; `--json` muestra un
  resultado legible por máquina.
- `--owner <handle>` publica con el identificador de editor de una organización o usuario cuando el
  actor dispone de acceso de editor.
- `--migrate-owner` traslada una skill existente a `--owner` mientras publica una
  versión nueva. Requiere acceso de administrador o propietario en ambos editores.
- El comportamiento de los propietarios y las revisiones se explica en `docs/publishing.md`.
- Publicar una skill significa que se distribuye en ClawHub bajo `MIT-0`.
- Las skills publicadas pueden utilizarse, modificarse y redistribuirse libremente sin atribución.
- ClawHub no admite skills de pago ni precios por skill.
- Alias heredado: `publish <path>`.

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

El flujo de trabajo reutilizable
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
de ClawHub llama a `skill publish` para una `skill_path`, o para cada carpeta de skill
inmediata dentro de `root` (predeterminado: `skills`). Omite las skills sin cambios y utiliza el
mismo comportamiento automático de versión de parche.

Establezca `dry_run: true` para obtener una vista previa sin token. Las publicaciones reales requieren el
secreto `clawhub_token`.

### `sync`

- Analiza el directorio de trabajo actual, el directorio de skills configurado y cualquier
  carpeta `--root <dir>` para buscar carpetas locales de skills que contengan `SKILL.md` o
  `skill.md`.
- Compara la huella digital de cada skill local con ClawHub y publica solo las skills nuevas o
  modificadas.
- Las skills nuevas se publican como `1.0.0`; las skills modificadas publican de forma predeterminada la siguiente versión
  de parche. Utilice `--bump minor|major` para lotes de actualización que deban avanzar mediante un
  incremento mayor de semver.
- `--dry-run` muestra el plan de publicación sin cargar contenido; `--json` muestra un
  plan legible por máquina.
- `--all` publica todas las skills nuevas o modificadas sin solicitar confirmación. Sin
  `--all`, los terminales interactivos permiten seleccionar las skills que se publicarán.
- `--owner <handle>` publica con el identificador de editor de una organización o usuario cuando el
  actor dispone de acceso de editor.
- `sync` solo publica en una dirección. No instala, actualiza ni descarga, ni
  informa de telemetría de instalaciones o descargas.

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- Requiere `clawhub login`.
- Ejecuta ClawScan de ClawHub mediante `POST /api/v1/skills/-/scan` y después consulta periódicamente hasta que el análisis llega a un estado terminal.
- Los análisis son asíncronos y pueden tardar en completarse. Mientras están en cola, el indicador giratorio del terminal muestra la posición actual del análisis en la cola priorizada y cuántos análisis hay por delante.
- Los análisis publicados requieren acceso de propietario o de administración del editor. Los moderadores y administradores pueden utilizar el mismo backend mediante `clawhub-admin`.
- `--update` solo es válido con `--slug`; escribe los resultados correctos del análisis publicado en la versión seleccionada.
- `--output <file.zip>` descarga el archivo completo del informe con `manifest.json`, `clawscan.json`, `skillspector.json`, `static-analysis.json`, `virustotal.json` y `README.md`.
- `--json` muestra la respuesta completa de la consulta periódica para su automatización.
- Ya no se admiten los análisis de rutas locales. Cargue una versión nueva y después utilice `scan download` para recuperar los resultados de análisis almacenados de esa versión enviada.

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- Requiere `clawhub login`.
- Descarga el archivo ZIP del informe de análisis almacenado para una versión enviada de una Skill o un Plugin, incluidas las versiones bloqueadas u ocultadas por las comprobaciones de seguridad de ClawHub.
- Las descargas de Skills usan el slug de la Skill y, de forma predeterminada, `--kind skill`.
- Las descargas de Plugins usan el nombre del paquete y requieren `--kind plugin`.
- `--version` es obligatorio para que los autores inspeccionen la versión exacta enviada que ClawHub bloqueó.
- `--output <file.zip>` elige la ruta de destino.

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub incluye un flujo de trabajo reutilizable oficial en
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)
para repositorios de Skills y repositorios de catálogos.

Configuración típica de un catálogo:

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Notas:

- `root` usa `skills` de forma predeterminada para los repositorios de catálogos.
- Pase `skill_path: skills/review-helper` para procesar una carpeta de Skill.
- `owner` corresponde a la opción `--owner` de la CLI; omítala para publicar como el usuario autenticado.
- La publicación de Skills V1 usa `clawhub_token`; por ahora, la publicación de confianza mediante OIDC de GitHub solo está disponible para paquetes.

### `delete <skill>`

- Sin `--version`, elimina de forma lógica una Skill (propietario, moderador o administrador).
- Llama a `DELETE /api/v1/skills/{slug}`.
- Las eliminaciones lógicas iniciadas por el propietario reservan el slug durante 30 días; el comando muestra la hora de vencimiento.
- `--version <version>` retira una versión propia que no sea la más reciente mediante una ruta de cierre seguro
  específica de la versión. El número de versión permanece reservado y no puede volver a publicarse con
  contenido diferente. Publique una versión de reemplazo antes de eliminar la versión más reciente actual. El personal
  de la plataforma no puede eludir la propiedad en este flujo exclusivo para versiones.
- `--reason <text>` registra una nota de moderación en la eliminación lógica de toda la Skill y en el registro de auditoría.
- `--note <text>` es un alias de `--reason`.
- `--yes` omite la confirmación.

### `undelete <skill>`

- Restaura una Skill oculta (propietario, moderador o administrador).
- Llama a `POST /api/v1/skills/{slug}/undelete`.
- `--version <version>` restaura únicamente el artefacto exacto conservado que retiró previamente el mismo
  propietario. No convierte la versión restaurada en la más reciente ni vuelve a crear las etiquetas eliminadas.
- La restauración de una versión llama a `POST /api/v1/skills/{slug}/versions/{version}/restore`.
- `--reason <text>` registra una nota de moderación en la Skill y en el registro de auditoría.
- `--note <text>` es un alias de `--reason`.
- `--yes` omite la confirmación.

### `hide <skill>`

- Oculta una Skill (propietario, moderador o administrador).
- Alias de `delete`.

### `unhide <skill>`

- Deja de ocultar una Skill (propietario, moderador o administrador).
- Alias de `undelete`.

### `skill rename <skill> <new-name>`

- Cambia el nombre de una Skill propia y conserva el slug anterior como alias de redirección.
- Llama a `POST /api/v1/skills/{slug}/rename`.
- `--yes` omite la confirmación.

### `skill merge <source> <target>`

- Fusiona una Skill propia con otra Skill propia.
- El slug de origen deja de aparecer públicamente y se convierte en un alias de redirección al destino.
- Llama a `POST /api/v1/skills/{sourceSlug}/merge`.
- `--yes` omite la confirmación.

### `transfer`

- Flujo de trabajo de transferencia de propiedad.
- Las transferencias a identificadores de usuario crean una solicitud pendiente que acepta el destinatario.
- Las transferencias a identificadores de organizaciones o publicadores se aplican inmediatamente solo cuando el actor tiene
  acceso de administrador tanto al propietario actual como al publicador de destino.
- Subcomandos:
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- Endpoints:
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- Explora o busca en el catálogo unificado de paquetes mediante `GET /api/v1/packages` y `GET /api/v1/packages/search`.
- Use esta opción para Plugins y otras entradas de familias de paquetes; `search` en el nivel superior sigue siendo la interfaz de búsqueda de Skills.
- Opciones:
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`, `--os <os>`, `--arch <arch>`, `--libc <libc>`
  - `--requires-browser`, `--requires-desktop`, `--requires-native-deps`
  - `--requires-external-service`, `--external-service <name>`
  - `--binary <name>`, `--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>` (1-100, valor predeterminado: 25)
  - `--json`

Ejemplos:

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- Obtiene los metadatos del paquete sin instalarlo.
- Use esta opción para inspeccionar los metadatos, la compatibilidad, la verificación, el origen y las versiones o archivos de un Plugin.
- `--version <version>`: inspecciona una versión específica (valor predeterminado: la más reciente).
- `--tag <tag>`: inspecciona una versión etiquetada (p. ej., `latest`).
- `--versions`: enumera el historial de versiones (primera página).
- `--limit <n>`: número máximo de versiones que se enumerarán (1-100).
- `--files`: enumera los archivos de la versión seleccionada.
- `--file <path>`: obtiene una vista previa limitada de texto UTF-8 (límite de 200KB).
- `--json`: salida legible por máquinas.

### `package download <name>`

- Resuelve una versión de paquete mediante
  `GET /api/v1/packages/{name}/versions/{version}/artifact`.
- Descarga el artefacto desde `downloadUrl` del resolutor.
- Verifica el SHA-256 de ClawHub para todos los artefactos.
- Para los artefactos npm-pack de ClawPack, también verifica la integridad `sha512` de npm,
  el shasum de npm y el nombre o la versión `package.json` del archivo tar.
- Las versiones ZIP heredadas se descargan mediante la ruta ZIP heredada.
- Opciones:
  - `--version <version>`: descarga una versión específica.
  - `--tag <tag>`: descarga una versión etiquetada (valor predeterminado: `latest`).
  - `-o, --output <path>`: archivo o directorio de salida.
  - `--force`: sobrescribe un archivo de salida existente.
  - `--json`: salida legible por máquinas.

Ejemplos:

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- Calcula el SHA-256 de ClawHub, la integridad `sha512` de npm y el shasum de npm para un
  artefacto local.
- Con `--package`, resuelve los metadatos esperados desde ClawHub y compara el
  archivo local con los metadatos del artefacto publicado.
- Con opciones de resumen directas, verifica sin realizar una consulta de red.
- Opciones:
  - `--package <name>`: nombre del paquete para resolver los metadatos esperados del artefacto.
  - `--version <version>` o `--tag <tag>`: versión esperada del paquete.
  - `--sha256 <hex>`: SHA-256 esperado de ClawHub.
  - `--npm-integrity <sri>`: integridad esperada de npm.
  - `--npm-shasum <sha1>`: shasum esperado de npm.
  - `--json`: salida legible por máquinas.

Ejemplos:

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- Ejecuta el Inspector de Plugins incluido en la CLI de ClawHub sobre una carpeta local de
  paquetes de Plugins.
- De forma predeterminada, realiza una validación estática sin conexión, sin localizar ni importar un checkout
  local de OpenClaw.
- Los errores graves de compatibilidad producen una salida distinta de cero. Los hallazgos que solo son advertencias se muestran, pero
  producen una salida de cero.
- Opciones:
  - `--out <dir>`: escribe los informes del Inspector de Plugins en este directorio.
  - `--openclaw <path>`: inspecciona con respecto a un checkout local explícito de OpenClaw.
  - `--runtime`: habilita la captura en tiempo de ejecución; importa el código del Plugin.
  - `--allow-execute`: permite la captura en tiempo de ejecución en un espacio de trabajo aislado.
  - `--no-mock-sdk`: deshabilita el SDK simulado de OpenClaw durante la captura en tiempo de ejecución.
  - `--json`: salida legible por máquinas.

Ejemplo:

```bash
clawhub package validate ./example-plugin
```

Si la validación informa de un hallazgo relacionado con el paquete, el manifiesto, la importación del SDK o el artefacto, consulte
[Correcciones de validación de Plugins](/es/clawhub/plugin-validation-fixes) y vuelva a ejecutar el comando.

### `package delete <name>`

- Sin `--version`, elimina de forma lógica un paquete y todas sus versiones.
- `--version <version>` retira una versión propia que no sea la más reciente mediante una ruta de cierre seguro
  específica de la versión. El número de versión permanece reservado y no puede volver a publicarse con
  contenido diferente. Publique una versión de reemplazo antes de eliminar la versión más reciente actual. Este
  flujo exclusivo para versiones requiere ser propietario del paquete o administrador de un publicador de la organización; el personal de la plataforma no puede
  eludir la propiedad del paquete.
- La eliminación lógica del paquete completo requiere ser propietario del paquete, propietario o administrador de un publicador de la organización, moderador
  de la plataforma o administrador de la plataforma.
- Opciones:
  - `--version <version>`: retira una versión que no sea la más reciente.
  - `--yes`: omite la confirmación.
  - `--json`: salida legible por máquinas.

Ejemplo:

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- Restaura un paquete eliminado de forma lógica y sus versiones.
- Requiere ser propietario del paquete, propietario o administrador de un publicador de la organización, moderador
  de la plataforma o administrador de la plataforma.
- Llama a `POST /api/v1/packages/{name}/undelete`.
- `--version <version>` restaura únicamente la versión exacta conservada que retiró previamente el mismo
  propietario. No convierte la versión restaurada en la más reciente ni vuelve a crear las etiquetas o dist-tags eliminadas del paquete.
- La restauración de una versión llama a `POST /api/v1/packages/{name}/versions/{version}/restore`.
- Opciones:
  - `--version <version>`: restaura una versión retirada por el propietario.
  - `--yes`: omite la confirmación.
  - `--json`: salida legible por máquinas.

Ejemplo:

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- Transfiere un paquete a otro publicador.
- Requiere acceso de administrador tanto al propietario actual del paquete como al publicador
  de destino, salvo que lo realice un administrador de la plataforma.
- Los nombres de paquetes con ámbito deben transferirse al propietario del ámbito correspondiente.
- Llama a `POST /api/v1/packages/{name}/transfer`.
- Opciones:
  - `--to <owner>`: identificador del publicador de destino.
  - `--reason <text>`: motivo de auditoría opcional.
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- Comando autenticado para denunciar un paquete a los moderadores.
- Llama a `POST /api/v1/packages/{name}/report`.
- Las denuncias se aplican al paquete, pueden vincularse opcionalmente a una versión y pasan a ser visibles
  para que los moderadores las revisen.
- Las denuncias no ocultan automáticamente los paquetes ni bloquean por sí solas las descargas.
- Opciones:
  - `--version <version>`: versión opcional del paquete que se adjuntará a la denuncia.
  - `--reason <text>`: motivo obligatorio de la denuncia.
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "carga nativa sospechosa"
```

### `package moderation-status`

- Comando del propietario para comprobar la visibilidad de moderación del paquete.
- Llama a `GET /api/v1/packages/{name}/moderation`.
- Muestra el estado actual del análisis del paquete, el número de denuncias abiertas, el estado de moderación
  manual de la versión más reciente, el estado de bloqueo de descargas y los motivos de moderación.
- Opciones:
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- Comprueba si un paquete está preparado para su uso futuro por parte de OpenClaw.
- Llama a `GET /api/v1/packages/{name}/readiness`.
- Informa de los impedimentos relacionados con el estado oficial, la disponibilidad de ClawPack, el resumen del artefacto,
  la procedencia del código fuente, la compatibilidad con OpenClaw, los destinos de host, los metadatos del entorno
  y el estado del análisis.
- Opciones:
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- Muestra el estado de migración orientado a operadores de un paquete que puede sustituir a un
  plugin incluido con OpenClaw.
- Llama al mismo endpoint de preparación calculada que `package readiness`, pero muestra
  el estado centrado en la migración, la versión más reciente, el estado de paquete oficial, las comprobaciones y
  los impedimentos.
- Opciones:
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- Crea un publicador de organización propiedad del usuario autenticado.
- El identificador se normaliza a minúsculas y puede proporcionarse con o sin `@`.
- Los publicadores de organización recién creados no son de confianza ni oficiales de forma predeterminada.
- Falla si el identificador ya lo utiliza un publicador o usuario existente, o una ruta reservada.

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- Publica un plugin de código o un plugin de paquete mediante `POST /api/v1/packages`.
- `<source>` acepta:
  - Ruta de carpeta local: `./my-plugin`
  - Tarball local de ClawPack generado por npm-pack: `./my-plugin-1.2.3.tgz`
  - Repositorio de GitHub: `owner/repo` o `owner/repo@ref`
  - URL de GitHub: `https://github.com/owner/repo`
- Los metadatos se detectan automáticamente a partir de `package.json`, `openclaw.plugin.json` y
  marcadores reales de paquetes de OpenClaw, como `.codex-plugin/plugin.json`,
  `.claude-plugin/plugin.json` y `.cursor-plugin/plugin.json`.
- Las fuentes `.tgz` se tratan como ClawPack. La CLI carga los bytes exactos de npm-pack
  y utiliza el contenido extraído de `package/` únicamente para la validación y
  el rellenado previo de metadatos.
- Las carpetas de plugins de código se empaquetan en un tarball npm de ClawPack antes de cargarlas para que
  las instalaciones de OpenClaw puedan verificar el artefacto exacto. Las carpetas de plugins de paquete siguen
  utilizando la ruta de publicación de archivos extraídos.
- Para las fuentes de GitHub, la atribución del código fuente se rellena automáticamente a partir del repositorio, el commit resuelto, la referencia y la subruta.
- Para las carpetas locales, la atribución del código fuente se detecta automáticamente mediante el repositorio Git local cuando el remoto de origen apunta a GitHub.
- Los plugins de código externos deben declarar explícitamente `openclaw.compat.pluginApi` y
  `openclaw.build.openclawVersion`.
  El campo `package.json.version` de nivel superior no se utiliza como alternativa para la validación de publicación.
- `--dry-run` ofrece una vista previa de la carga útil de publicación resuelta sin cargarla.
- `--json` emite una salida legible por máquina para la CI.
- `--owner <handle>` publica bajo el identificador de un publicador de usuario u organización cuando el actor tiene acceso al publicador.
- Los nombres de paquetes con ámbito deben coincidir con el propietario seleccionado. Consulte `docs/publishing.md`.
- Las opciones existentes (`--family`, `--name`, `--version`, `--source-repo`, `--source-commit`, `--source-ref`, `--source-path`) siguen funcionando como anulaciones.
- Los repositorios privados de GitHub requieren `GITHUB_TOKEN`.

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### Flujo local recomendado

Utilice primero `--dry-run` para confirmar los metadatos resueltos del paquete y
la atribución del código fuente antes de crear una versión activa:

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### Flujo de carpeta local

Para los plugins de código, la publicación de carpetas crea y carga un artefacto ClawPack desde
la carpeta del paquete:

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### `package.json` mínimo para `--family code-plugin`

Los plugins de código externos necesitan una pequeña cantidad de metadatos de OpenClaw en
`package.json`. Este manifiesto mínimo basta para publicar correctamente:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

Campos obligatorios:

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

Notas:

- `package.json.version` es la versión de publicación del paquete, pero no se utiliza como
  alternativa para la validación de compatibilidad o compilación de OpenClaw.
- `openclaw.hostTargets` y `openclaw.environment` son metadatos opcionales.
  ClawHub puede mostrarlos cuando estén presentes, pero no son obligatorios para publicar.
- `openclaw.compat.minGatewayVersion` y
  `openclaw.build.pluginSdkVersion` son elementos adicionales opcionales si se quieren publicar
  metadatos de compatibilidad más detallados.
- Si se utiliza una versión anterior de la CLI `clawhub`, debe actualizarse antes de publicar para que
  las comprobaciones preliminares locales se ejecuten antes de la carga.
- Si la validación informa de un código de corrección, consulte
  [Correcciones de validación de plugins](/es/clawhub/plugin-validation-fixes).

#### GitHub Actions

ClawHub también proporciona un flujo de trabajo reutilizable oficial en
[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)
para repositorios de plugins.

Configuración habitual del invocador:

```yaml
name: Package Publish

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

Notas:

- El flujo de trabajo reutilizable establece de forma predeterminada `source` en el repositorio invocador.
- En monorepos, proporcione `source_path` para que el flujo de trabajo publique la carpeta
  del paquete del plugin, por ejemplo, `source_path: extensions/codex`.
- Fije el flujo de trabajo reutilizable a una etiqueta estable o a un SHA de commit completo. No ejecute la publicación de versiones desde `@main`.
- `pull_request` debe utilizar `dry_run: true` para que la CI no genere efectos secundarios.
- Las publicaciones reales deben limitarse a eventos de confianza, como `workflow_dispatch`, o a envíos de etiquetas.
- La publicación de confianza sin secreto solo funciona en `workflow_dispatch`; los envíos de etiquetas siguen necesitando `clawhub_token`.
- Mantenga `clawhub_token` disponible para la primera publicación, los paquetes que no sean de confianza o las publicaciones de emergencia.
- El flujo de trabajo carga el resultado JSON como artefacto y lo expone como salidas del flujo de trabajo.

### `package trusted-publisher get <name>`

- Muestra la configuración del publicador de confianza de GitHub Actions para un paquete.
- Utilícelo después de establecer la configuración para confirmar el repositorio, el nombre de archivo del flujo de trabajo
  y la fijación opcional del entorno.
- Opciones:
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- Adjunta o sustituye la configuración del publicador de confianza de GitHub Actions para un
  paquete existente.
- El paquete debe crearse primero mediante la publicación normal manual o autenticada con token
  `clawhub package publish`.
- Una vez establecida la configuración, las publicaciones futuras compatibles desde GitHub Actions pueden utilizar
  OIDC o la publicación de confianza sin un token de ClawHub de larga duración.
- `--repository <repo>` debe ser `owner/repo`.
- `--workflow-filename <file>` debe coincidir con el nombre del archivo del flujo de trabajo en
  `.github/workflows/`.
- `--environment <name>` es opcional. Cuando se configura, el entorno de GitHub Actions
  en la declaración OIDC debe coincidir exactamente.
- ClawHub verifica el repositorio de GitHub configurado al ejecutar este comando.
  Los repositorios públicos pueden verificarse mediante los metadatos públicos de GitHub. Los repositorios
  privados requieren que ClawHub tenga acceso de GitHub a ese repositorio, por
  ejemplo, mediante una futura instalación de la aplicación de GitHub de ClawHub u otra
  integración autorizada de GitHub.
- Opciones:
  - `--repository <repo>`: repositorio de GitHub, por ejemplo, `openclaw/example-plugin`.
  - `--workflow-filename <file>`: nombre del archivo del flujo de trabajo, por ejemplo, `package-publish.yml`.
  - `--environment <name>`: entorno opcional de GitHub Actions con coincidencia exacta.
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- Elimina la configuración del publicador de confianza de un paquete.
- Utilícelo como reversión si es necesario desactivar o volver a crear el flujo de trabajo, el repositorio
  o la fijación del entorno.
- Las publicaciones reales posteriores deben utilizar la publicación autenticada normal hasta que se
  vuelva a establecer la configuración.
- Opciones:
  - `--json`: salida legible por máquina.

Ejemplo:

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### Telemetría de instalación

- Se envía después de `clawhub install <slug>` cuando se ha iniciado sesión, salvo que
  se haya establecido `CLAWHUB_DISABLE_TELEMETRY=1`.
- Los informes se realizan en la medida de lo posible. Los comandos de instalación no fallan si la telemetría
  no está disponible.
- Detalles: `docs/telemetry.md`.
