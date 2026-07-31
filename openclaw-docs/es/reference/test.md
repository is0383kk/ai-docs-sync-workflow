---
read_when:
    - Ejecutar o corregir pruebas
summary: Cómo ejecutar pruebas localmente (vitest) y cuándo usar los modos de ejecución forzada/cobertura
title: Pruebas
x-i18n:
    generated_at: "2026-07-26T05:55:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 391185703e853bb523e1396eb22da4693d10d47b1644d3b2a51707d329f67dae
    source_path: reference/test.md
    workflow: 16
---

- Kit completo de pruebas (suites, en vivo, Docker): [Pruebas](/es/help/testing)
- Validación de actualizaciones y paquetes de plugins: [Pruebas de actualizaciones y plugins](/es/help/testing-updates-plugins)

## Configuración predeterminada del agente

Las sesiones de agente ejecutan localmente una o unas pocas pruebas específicas y comprobaciones estáticas de bajo coste solo
para código fuente de confianza y cuando la instalación de dependencias existente está lista. Nunca
se deben ejecutar localmente herramientas de repositorios que no sean de confianza. Las suites más grandes, las puertas de cambios con
distribución de comprobaciones de tipos/lint, las compilaciones, Docker, los flujos de paquetes, E2E, las pruebas en vivo y
la validación multiplataforma se ejecutan de forma remota mediante Crabbox. Las
pruebas intensivas de mantenedores de confianza utilizan Blacksmith Testbox de forma predeterminada. El flujo de trabajo de Testbox configurado
carga credenciales, por lo que el código de colaboradores o bifurcaciones que no sea de confianza debe usar
la Pipeline de CI sin secretos de la bifurcación o una instancia directa y saneada de AWS Crabbox.

No se debe realizar un calentamiento previo para trabajos previstos. Se debe adquirir el backend de forma diferida cuando
el primer comando intensivo esté listo, reutilizar el id `tbx_...` devuelto para los comandos intensivos
posteriores, sincronizar el checkout actual en cada ejecución y detenerlo antes de la entrega.

Después de la primera reutilización correcta, el contenedor registra la base del arrendamiento,
las dependencias y la huella digital del flujo de trabajo de Testbox en `.crabbox/testbox-leases/`.
Las ediciones exclusivas del código fuente siguen reutilizando la máquina preparada. Un cambio en la base de fusión, el archivo de bloqueo,
la entrada del gestor de paquetes, el contenedor o el flujo de trabajo de Testbox provoca un cierre seguro y requiere un
arrendamiento nuevo. Cada ejecución sigue sincronizando el checkout actual.
`OPENCLAW_TESTBOX_ALLOW_STALE=1` se utiliza únicamente para diagnósticos intencionados, no
para pruebas de versiones.

Los comandos de pruebas locales que aparecen a continuación están destinados a flujos de trabajo humanos y pruebas acotadas del agente.
La falta de disponibilidad del proveedor remoto debe notificarse; no autoriza a
ejecutar silenciosamente una puerta local amplia.

Para pruebas intensivas que no sean de confianza, se debe preparar de forma diferida con `--provider aws`. Cada ejecución debe establecer
`CRABBOX_ENV_ALLOW=CI`, pasar `--provider aws --no-hydrate` y utilizar
un `HOME` remoto temporal nuevo antes de instalar dependencias o ejecutar
pruebas. Se debe usar un arrendamiento recién preparado y dedicado a ese código fuente que no sea de confianza; nunca se debe reutilizar
un arrendamiento de confianza o previamente cargado con credenciales. Se debe iniciar un binario Crabbox
de confianza instalado desde un checkout `main` limpio y de confianza, y obtener únicamente el pull request remoto con
`--fresh-pr`; nunca se debe ejecutar localmente el contenedor ni la configuración del checkout que no sea de confianza.
Se debe desactivar `CRABBOX_AWS_INSTANCE_PROFILE` y aplicar un cierre seguro salvo que el valor resuelto de
`aws.instanceProfile` esté vacío. Antes de cualquier instalación o prueba, se deben usar herramientas de
ruta absoluta de confianza para exigir un token IMDSv2, demostrar que el endpoint de credenciales de IAM
devuelve 404 y verificar que el `git rev-parse HEAD` remoto sea igual al SHA completo
de la cabecera del pull request revisado. Se debe vincular el arrendamiento a ese SHA y detenerlo y volverlo a preparar cuando cambie la cabecera.
Se debe cargar el archivo de confianza `scripts/crabbox-untrusted-bootstrap.sh` desde un
`main` limpio junto con `--fresh-pr`; este instala versiones fijadas de Node/pnpm, verifica el SHA
y la versión fijada del gestor de paquetes, aísla `HOME`, instala las dependencias y, a continuación, ejecuta
la prueba solicitada. Si el bróker no puede demostrar que no existe ningún rol o ningún pull request remoto,
se debe usar la Pipeline de CI sin secretos de la bifurcación. No se deben usar `hydrate-github`, `--no-sync` ni un
flujo de trabajo de Testbox cargado con credenciales.
Se deben desactivar todas las anulaciones `CRABBOX_TAILSCALE*`, forzar `--network public
--tailscale=false`, borrar las marcas de nodo de salida/LAN y exigir que `crabbox inspect`
informe de una red pública sin estado de Tailscale antes de cargar cualquier script.

## Orden local habitual

1. `pnpm test:changed` para pruebas Vitest del ámbito modificado.
2. `pnpm test <path-or-filter>` para un archivo, directorio u objetivo explícito.
3. `pnpm test` únicamente cuando se necesite intencionadamente la suite Vitest local completa.

En un árbol de trabajo de Codex o un checkout vinculado o disperso, los agentes evitan ejecutar directamente de forma local
`pnpm test*` / `pnpm check*` / `pnpm crabbox:run`:

- Prueba específica acotada con las dependencias listas:
  `node scripts/run-vitest.mjs <path-or-filter>`.
- Comprobación de cambios con clasificación previa: `node scripts/check-changed.mjs`; los planes exclusivos de documentación,
  sin cambios y de metadatos pequeños permanecen en local cuando las dependencias están listas,
  mientras que los planes intensivos o con dependencias ausentes se delegan a Testbox.
- Prueba amplia explícita con arrendamiento conservado: `node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox ... -- env OPENCLAW_CHECK_CHANGED_REMOTE_CHILD=1 OPENCLAW_CHANGED_LANES_RAW_SYNC=1 corepack pnpm check:changed`, para que pnpm se ejecute dentro de Testbox.
- El `exitCode` final del contenedor y el JSON de tiempos constituyen el resultado del comando. Una ejecución delegada de GitHub Actions de Blacksmith puede mostrar `cancelled` después de un comando SSH correcto porque Testbox se detiene desde fuera de la acción de mantenimiento activo; se deben comprobar el resumen del contenedor y la salida del comando antes de considerarlo un fallo.
- `OPENCLAW_HEAVY_CHECK_LOCK_SCOPE=worktree <local-heavy-check command>`: mantiene la serialización de comprobaciones intensivas dentro del árbol de trabajo actual en lugar del directorio común de Git para comandos como `pnpm check:changed` y `pnpm test ...` dirigidos. Se debe usar únicamente en hosts locales de alta capacidad cuando se ejecuten intencionadamente comprobaciones independientes en distintos árboles de trabajo vinculados.

## Comandos principales

Las ejecuciones del contenedor de pruebas terminan con un breve resumen `[test] passed|failed|skipped ... in ...`; la línea de duración propia de Vitest sigue siendo el detalle de cada fragmento.

| Comando                                           | Qué hace                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                                       | Los objetivos explícitos de archivos o directorios se canalizan mediante flujos Vitest con ámbito definido. Las ejecuciones sin objetivos constituyen pruebas de la suite completa: los grupos fijos de fragmentos se expanden a configuraciones finales para su ejecución local en paralelo y la distribución de fragmentos prevista se muestra antes de comenzar. El grupo de extensiones siempre se expande a configuraciones de fragmentos por extensión en lugar de un único proceso gigante del proyecto raíz.           |
| `pnpm test:changed`                               | Ejecución inteligente y económica de pruebas modificadas: objetivos precisos procedentes de ediciones directas de pruebas, archivos `*.test.ts` hermanos, asignaciones explícitas de código fuente y el grafo de importaciones local. Los cambios amplios de configuración o paquetes se omiten salvo que se asignen a pruebas precisas.                                                                                                                               |
| `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` | Ejecución amplia explícita de pruebas modificadas; se utiliza cuando una edición del arnés de pruebas, la configuración o un paquete debe recurrir al comportamiento más amplio de pruebas modificadas de Vitest.                                                                                                                                                                                                                        |
| `pnpm test:force`                                 | Libera el puerto configurado del Gateway de OpenClaw (valor predeterminado: `18789`) y, a continuación, ejecuta la suite completa con un puerto de Gateway aislado para que las pruebas del servidor no colisionen con una instancia en ejecución.                                                                                                                                                                                    |
| `pnpm test:coverage`                              | Emite un informe informativo de cobertura V8 para el flujo unitario predeterminado (`vitest.unit.config.ts`); no se aplican umbrales de cobertura.                                                                                                                                                                                                                             |
| `pnpm test:coverage:changed`                      | Cobertura unitaria únicamente para los archivos modificados desde `origin/main`.                                                                                                                                                                                                                                                                                                       |
| `pnpm changed:lanes`                              | Muestra los flujos arquitectónicos activados por las diferencias con respecto a `origin/main`.                                                                                                                                                                                                                                                                                      |
| `pnpm check:changed`                              | Clasifica los flujos modificados antes de elegir la ejecución. Los planes exclusivos de documentación, sin cambios y de metadatos pequeños permanecen en local cuando las dependencias están listas; los planes con distribución de comprobaciones de tipos/lint, otros flujos intensivos o dependencias locales ausentes se delegan a Crabbox/Testbox fuera de la Pipeline de CI. No ejecuta Vitest; se debe usar `pnpm test:changed` o `pnpm test <target>` para las pruebas. |

## Estado de pruebas compartido y auxiliares de procesos

- `src/test-utils/openclaw-test-state.ts`: se utiliza desde Vitest cuando una prueba necesita un `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, fixture de configuración, espacio de trabajo, directorio de agente o almacén de perfiles de autenticación aislados.
- `pnpm test:env-mutations:report`: informe no bloqueante de pruebas y arneses que modifican directamente `HOME`, `OPENCLAW_STATE_DIR`, `OPENCLAW_CONFIG_PATH`, `OPENCLAW_WORKSPACE_DIR` o claves de entorno relacionadas. Se utiliza para encontrar candidatos de migración al auxiliar de estado de pruebas compartido.
- `test/helpers/openclaw-test-instance.ts`: pruebas E2E a nivel de proceso que necesitan un Gateway en ejecución, el entorno de la CLI, captura de registros y limpieza en un único lugar.
- Los flujos E2E de Docker/Bash que cargan `scripts/lib/docker-e2e-image.sh` pueden pasar `docker_e2e_test_state_shell_b64 <label> <scenario>` al contenedor y decodificarlo con `scripts/lib/openclaw-e2e-instance.sh`; los scripts con varios directorios de inicio pueden pasar `docker_e2e_test_state_function_b64` y llamar a `openclaw_test_state_create <label> <scenario>` en cada flujo. `node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` escribe un archivo de entorno del host que puede cargarse (el `--` antes de `create` evita que las versiones más recientes de Node traten `--env-file` como una marca de Node). Los flujos que inician un Gateway pueden cargar `scripts/lib/openclaw-e2e-instance.sh` para la resolución del punto de entrada, el inicio simulado de OpenAI, la ejecución en primer o segundo plano, las sondas de disponibilidad, la exportación del entorno de estado, los volcados de registros y la limpieza de procesos.

## Flujos de la interfaz de control, la TUI y las extensiones

- **E2E simulado de la interfaz de control:** `pnpm test:ui:e2e` ejecuta la vía de Vitest + Playwright que inicia la interfaz de control de Vite y controla una página real de Chromium contra un WebSocket simulado del Gateway. Las pruebas se encuentran en `ui/src/**/*.e2e.test.ts`; las simulaciones y los controles compartidos se encuentran en `ui/src/test-helpers/control-ui-e2e.ts`. `pnpm test:e2e` incluye esta vía. Las ejecuciones de agentes usan Testbox/Crabbox de forma predeterminada, incluida la verificación específica; use `node scripts/run-vitest.mjs run --config test/vitest/vitest.ui-e2e.config.ts --configLoader runner ui/src/ui/e2e/chat-flow.e2e.test.ts` solo como alternativa local explícita.
- **Pruebas de PTY de la TUI:** `node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts` ejecuta la vía rápida de PTY con un backend falso. `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` o `pnpm tui:pty:test:watch --mode local` ejecuta la prueba de humo más lenta de `tui --local`, que solo simula el endpoint externo del modelo. Compruebe texto visible estable o llamadas a fixtures, no instantáneas ANSI sin procesar.
- `pnpm test:extensions` y `pnpm test extensions` ejecutan todos los shards de extensiones/plugins. Los plugins de canales pesados, el plugin del navegador y OpenAI se ejecutan como shards dedicados; los demás grupos de plugins permanecen agrupados. `pnpm test extensions/<id>` ejecuta una vía de un plugin incluido.
- Los archivos fuente con pruebas hermanas se asignan a esas pruebas antes de recurrir a globs de directorio más amplios. Las modificaciones de auxiliares en `src/channels/plugins/contracts/test-helpers`, `src/plugin-sdk/test-helpers` y `src/plugins/contracts` usan un grafo de importaciones local para ejecutar las pruebas que los importan, en lugar de ejecutar ampliamente todos los shards cuando la ruta de dependencia es precisa.
- Los destinos de directorios de contratos se distribuyen entre sus vías de contratos: `pnpm test src/channels/plugins/contracts` ejecuta las cuatro configuraciones de contratos de canales y `pnpm test src/plugins/contracts` ejecuta la configuración de contratos de plugins, ya que los proyectos genéricos `channels`/`plugins` excluyen `contracts/**`.
- `auto-reply` se divide en tres configuraciones dedicadas (`core`, `top-level`, `reply`) para que el arnés de respuestas no domine las pruebas más ligeras de estado, tokens y auxiliares de nivel superior.
- Los archivos de prueba seleccionados de `plugin-sdk` y `commands` se dirigen a través de vías ligeras dedicadas que conservan solo `test/setup.ts`, mientras los casos con mayor carga de ejecución permanecen en sus vías existentes.
- La configuración base de Vitest usa de forma predeterminada `pool: "threads"` y `isolate: false`, con el ejecutor compartido no aislado habilitado en todas las configuraciones del repositorio.
- `pnpm test:channels` ejecuta `vitest.channels.config.ts`.

## Gateway y E2E

- La integración del Gateway es opcional: `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` o `pnpm test:gateway`.
- `pnpm test:e2e`: agregado E2E del repositorio = `pnpm test:e2e:gateway && pnpm test:ui:e2e`.
- `pnpm test:e2e:gateway`: pruebas de humo de extremo a extremo del Gateway (emparejamiento de WS/HTTP/Node con varias instancias). Usa de forma predeterminada `threads` + `isolate: false` con workers adaptativos en `vitest.e2e.config.ts`; ajústelos con `OPENCLAW_E2E_WORKERS=<n>` y habilite registros detallados con `OPENCLAW_E2E_VERBOSE=1`.
- `pnpm test:live`: pruebas en vivo de proveedores (Claude/Minimax/DeepSeek/z.ai/etc., controladas por `*.live.test.ts`). Requieren claves de API y `LIVE=1` (o `OPENCLAW_LIVE_TEST=1`) para dejar de omitirse; habilite la salida detallada con `OPENCLAW_LIVE_TEST_QUIET=0`.

## Suite completa de Docker (`pnpm test:docker:all`)

Compila la imagen compartida de pruebas en vivo, empaqueta OpenClaw una vez como tarball de npm, compila o reutiliza una imagen básica de ejecución con Node/Git y una imagen funcional que instala ese tarball en `/app`, y después ejecuta las vías de humo de Docker mediante un planificador ponderado. `scripts/package-openclaw-for-docker.mjs` es el único empaquetador local/de CI y valida el tarball y `dist/postinstall-inventory.json` antes de que Docker lo utilice.

- Imagen básica (`OPENCLAW_DOCKER_E2E_BARE_IMAGE`): vías del instalador, las actualizaciones y las dependencias de plugins; monta el tarball precompilado en lugar de fuentes copiadas del repositorio.
- Imagen funcional (`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`): vías normales de funcionalidad de la aplicación compilada.
- Definiciones de las vías: `scripts/lib/docker-e2e-scenarios.mjs`. Planificador: `scripts/lib/docker-e2e-plan.mjs`. Ejecutor: `scripts/test-docker-all.mjs`.
- `node scripts/test-docker-all.mjs --plan-json` emite el plan de CI gestionado por el planificador (vías, tipos de imagen, necesidades de paquetes/imágenes en vivo, escenarios de estado y comprobaciones de credenciales) sin compilar ni ejecutar Docker.

Parámetros de planificación (variables de entorno, valores predeterminados entre paréntesis):

| Variable de entorno                                                                                             | Valor predeterminado | Propósito                                                                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`                                                                               | 10                   | Espacios de procesos.                                                                                                                                                                                                                                                                      |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM`                                                                          | 10                   | Grupo final sensible al proveedor.                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`                                                                                | 9                    | Límite de vías pesadas de proveedores en vivo.                                                                                                                                                                                                                                             |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`                                                                                 | 5                    | Límite de vías que usan recursos de npm.                                                                                                                                                                                                                                                   |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`                                                                             | 7                    | Límite de vías que usan recursos de servicios.                                                                                                                                                                                                                                             |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` / `_CODEX_LIMIT` / `_GEMINI_LIMIT` / `_DROID_LIMIT` / `_OPENCODE_LIMIT` | 4                    | Límites de vías pesadas por proveedor.                                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_LIVE_OPENAI_LIMIT` / `_TELEGRAM_LIMIT`                                                     | 1                    | Límites más restrictivos por proveedor.                                                                                                                                                                                                                                                    |
| `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` / `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`                                         | -                    | Sustitución para hosts de mayor capacidad.                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS`                                                                          | 2000                 | Retardo entre los inicios de las vías; evita avalanchas de creación en el daemon local de Docker.                                                                                                                                                                                          |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`                                                                           | 7,200,000 (120 min)  | Tiempo de espera alternativo por vía; las vías en vivo/finales seleccionadas usan límites más estrictos.                                                                                                                                                                                   |
| `OPENCLAW_DOCKER_ALL_LIVE_RETRIES`                                                                              | 1                    | Reintentos ante fallos transitorios de proveedores en vivo.                                                                                                                                                                                                                                |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`                                                                                   | desactivado          | Imprime el manifiesto de vías sin ejecutar Docker.                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`                                                                        | 30000                | Intervalo de impresión del estado de las vías activas.                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_TIMINGS`                                                                                   | activado             | Reutiliza `.artifacts/docker-tests/lane-timings.json` para ordenar primero las de mayor duración; establezca `0` para deshabilitarlo.                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_LIVE_MODE`                                                                                 | -                    | `skip` solo para vías deterministas/locales, `only` solo para vías de proveedores en vivo. Alias: `pnpm test:docker:local:all`, `pnpm test:docker:live:all`. El modo solo en vivo combina las vías principales y finales en vivo en un único grupo ordenado de mayor a menor duración para que los grupos de proveedores agrupen el trabajo de Claude/Codex/Gemini. |
| `OPENCLAW_LIVE_CLI_BACKEND_SETUP_TIMEOUT_SECONDS`                                                               | 180                  | Tiempo de espera de configuración de Docker para el backend de la CLI.                                                                                                                                                                                                                     |

El patrón de variables de entorno para los límites de recursos es `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` (nombre del recurso en mayúsculas, con los caracteres no alfanuméricos sustituidos por `_`).

Otro comportamiento: el ejecutor realiza una comprobación previa de Docker de forma predeterminada, limpia los contenedores E2E obsoletos de OpenClaw, comparte las cachés de herramientas de la CLI del proveedor entre carriles compatibles y deja de programar nuevos carriles agrupados después del primer fallo, salvo que se establezca `OPENCLAW_DOCKER_ALL_FAIL_FAST=0`. Si un carril supera el límite efectivo de peso/recursos en un host con poco paralelismo, aun así puede iniciarse desde un grupo vacío y ejecutarse en solitario hasta que libere capacidad. Los registros por carril, `summary.json`, `failures.json` y los tiempos de las fases se escriben en `.artifacts/docker-tests/<run-id>/`; use `pnpm test:docker:timings <summary.json>` para inspeccionar los carriles lentos y `pnpm test:docker:rerun <run-id|summary.json|failures.json>` para mostrar comandos económicos de repetición dirigida.

### Carriles de Docker destacados

| Comando                                                                     | Verifica                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test:docker:browser-cdp-snapshot`                                     | Contenedor E2E de código fuente respaldado por Chromium con CDP sin procesar + Gateway aislado; las instantáneas de roles de CDP de `browser doctor --deep` incluyen las URL de los enlaces, los elementos en los que se puede hacer clic promovidos por el cursor, las referencias de iframe y los metadatos de los marcos.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `pnpm test:docker:skill-install`                                            | Instala el archivo tar empaquetado en un ejecutor Docker básico con `skills.install.allowUploadedArchives: false`, resuelve un slug de skill actual mediante una búsqueda en vivo de ClawHub, lo instala mediante `openclaw skills install` y verifica `SKILL.md`, `.clawhub/origin.json`, `.clawhub/lock.json` y `skills info --json`.                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `pnpm test:docker:live-cli-backend:claude`, `:claude:resume`, `:claude:mcp` | Sondeos en vivo específicos del backend de la CLI; Gemini cuenta con los alias equivalentes `:resume` y `:mcp`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `pnpm test:docker:openwebui`                                                | OpenClaw + Open WebUI en Docker: inicia sesión, comprueba `/api/models` y ejecuta un chat real mediante proxy a través de `/api/chat/completions`. Requiere una clave utilizable de un modelo en vivo y descarga una imagen externa; no se espera que tenga la misma estabilidad en CI que las suites unitarias/E2E.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `pnpm test:docker:mcp-channels`                                             | Contenedor de Gateway precargado más un contenedor cliente que inicia `openclaw mcp serve`: detección de conversaciones enrutadas, lectura de transcripciones, metadatos de archivos adjuntos, comportamiento de la cola de eventos en vivo, enrutamiento de envíos salientes y notificaciones de canal + permisos al estilo de Claude a través del puente stdio real (la aserción lee directamente los marcos MCP de stdio sin procesar).                                                                                                                                                                                                                                                                                                                                                                                                               |
| `pnpm test:docker:upgrade-survivor`                                         | Instala el archivo tar empaquetado sobre un fixture modificado de un usuario antiguo, ejecuta la actualización del paquete y el doctor no interactivo sin claves de proveedor/canal en vivo, inicia un Gateway de bucle invertido y comprueba que se conserven los agentes, la configuración de canales, las listas de permitidos de plugins, los archivos del espacio de trabajo y de sesión, el estado obsoleto de las dependencias de plugins heredados, el inicio y el estado de RPC.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `pnpm test:docker:published-upgrade-survivor`                               | Instala `openclaw@latest` de forma predeterminada, precarga archivos realistas de usuarios existentes, configura mediante una receta `openclaw config set` incorporada, actualiza al archivo tar empaquetado, ejecuta el doctor no interactivo, escribe `.artifacts/upgrade-survivor/summary.json` y comprueba `/healthz`, `/readyz` y el estado de RPC. Sustituya el valor con `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, amplíe una matriz con `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` o añada fixtures de escenarios con `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` (incluye `configured-plugin-installs` y `stale-source-plugin-shadow`). Package Acceptance los expone como `published_upgrade_survivor_baseline(s)` / `_scenarios` y resuelve metatokens como `last-stable-4` o `all-since-2026.4.23`. |
| `pnpm test:docker:update-migration`                                         | Arnés de supervivencia a actualizaciones publicadas en el escenario `plugin-deps-cleanup`, que comienza en `openclaw@2026.4.23` de forma predeterminada. El flujo de trabajo `Update Migration` lo amplía con `baselines=all-since-2026.4.23` para demostrar la limpieza de dependencias de plugins configurados fuera de la CI de versión completa.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `pnpm test:docker:plugins`                                                  | Prueba de humo de instalación/actualización para una ruta local, `file:`, paquetes del registro npm con dependencias elevadas, referencias móviles de git, fixtures de ClawHub, actualizaciones del marketplace y habilitación/inspección del paquete de Claude.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

## Puerta local para PR

Para las comprobaciones locales de puerta/integración de PR, ejecute:

- `pnpm check:changed`
- `pnpm check`
- `pnpm check:test-types`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

Si `pnpm test` presenta fallos intermitentes en un host con carga, vuelva a ejecutarlo una vez antes de considerarlo una regresión y, después, aíslelo con `pnpm test <path/to/test>`. Para hosts con memoria limitada:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## Herramientas de rendimiento de pruebas

- `pnpm test:perf:imports`: habilita los informes de duración y desglose de importaciones de Vitest, pero sigue usando el enrutamiento de carriles delimitado para objetivos explícitos de archivos/directorios. `pnpm test:perf:imports:changed` limita el mismo perfilado a los archivos modificados desde `origin/main`.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` compara el rendimiento de la ruta enrutada del modo de cambios con la ejecución nativa del proyecto raíz para la misma diferencia de git confirmada; `pnpm test:perf:changed:bench -- --worktree` evalúa el rendimiento del conjunto de cambios actual del árbol de trabajo sin confirmarlo primero.
- `pnpm test:perf:profile:main` escribe un perfil de CPU para el hilo principal de Vitest (`.artifacts/vitest-main-profile`); `pnpm test:perf:profile:runner` escribe perfiles de CPU + heap para el ejecutor unitario (`.artifacts/vitest-runner-profile`).
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`: ejecuta en serie cada configuración hoja de Vitest de la suite completa y escribe datos de duración agrupados, además de artefactos JSON/registros por configuración. Los informes de la suite completa aíslan los archivos de forma predeterminada para que los grafos de módulos retenidos y las pausas de GC de archivos anteriores no se atribuyan a aserciones posteriores; pase `-- --no-isolate` solo cuando se perfile intencionadamente la acumulación de trabajadores compartidos. El agente de rendimiento de pruebas utiliza esto como referencia antes de intentar corregir pruebas lentas. `pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` compara informes agrupados después de un cambio centrado en el rendimiento.
- Las ejecuciones fragmentadas de la suite completa, extensiones y patrones de inclusión actualizan los datos locales de tiempos en `.artifacts/vitest-shard-timings.json`; las ejecuciones posteriores de configuraciones completas usan esos tiempos para equilibrar los fragmentos lentos y rápidos. Los fragmentos de CI con patrones de inclusión añaden el nombre del fragmento a la clave de tiempos, lo que mantiene visibles los tiempos de los fragmentos filtrados sin sustituir los datos de tiempos de la configuración completa. Establezca `OPENCLAW_TEST_PROJECTS_TIMINGS=0` para ignorar el artefacto local de tiempos.

## Pruebas de rendimiento

<Accordion title="Latencia del modelo (scripts/bench-model.ts)">

```bash
pnpm tsx scripts/bench-model.ts --runs 10
```

Variables de entorno opcionales: `MINIMAX_API_KEY`, `MINIMAX_BASE_URL`, `MINIMAX_MODEL`, `ANTHROPIC_API_KEY`. Prompt predeterminado: «Responde con una sola palabra: ok. Sin puntuación ni texto adicional».

</Accordion>

<Accordion title="Inicio de la CLI (scripts/bench-cli-startup.ts)">

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
pnpm tsx scripts/bench-cli-startup.ts --runs 12
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3
pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all
```

Preajustes:

- `startup`: `--version`, `--help`, `health`, `health --json`, `status --json`, `status`
- `real`: `health`, `status`, `status --json`, `sessions`, `sessions --json`, `tasks --json`, `tasks list --json`, `tasks audit --json`, `agents list --json`, `gateway status`, `gateway status --json`, `gateway health --json`, `config get gateway.port`
- `all`: ambos preajustes combinados

La salida incluye `sampleCount`, promedio, p50, p95, mínimo/máximo, distribución de códigos de salida/señales y RSS máximo por comando. `--cpu-prof-dir` / `--heap-prof-dir` escriben perfiles de V8 por ejecución.

Salida guardada: `pnpm test:startup:bench:smoke` escribe `.artifacts/cli-startup-bench-smoke.json`; `pnpm test:startup:bench:save` escribe `.artifacts/cli-startup-bench-all.json` (`runs=5 warmup=1`). Fixture incluida en el repositorio: `test/fixtures/cli-startup-bench.json`, actualizada mediante `pnpm test:startup:bench:update` y comparada mediante `pnpm test:startup:bench:check`.

</Accordion>

<Accordion title="Inicio del Gateway (scripts/bench-gateway-startup.ts)">

De forma predeterminada, usa el punto de entrada de la CLI compilada en `dist/entry.js`; ejecute primero `pnpm build`. Pase `--entry scripts/run-node.mjs` para medir en su lugar el ejecutor del código fuente y mantenga esos resultados separados de las líneas base del punto de entrada compilado.

```bash
pnpm test:startup:gateway -- --runs 5 --warmup 1
pnpm test:startup:gateway -- --case skipChannels --case fiftyPlugins --runs 5
node --import tsx scripts/bench-gateway-startup.ts --case default --runs 5 --output .artifacts/gateway-startup.json
```

Identificadores de casos: `default`, `skipChannels` (se omite el inicio de los canales), `oneInternalHook`, `allInternalHooks`, `fiftyPlugins` (50 plugins de manifiesto), `fiftyStartupLazyPlugins` (50 plugins de manifiesto con inicio diferido).

La salida incluye la primera salida del proceso, `/healthz`, `/readyz`, el tiempo del registro de escucha HTTP, el tiempo del registro de disponibilidad del Gateway, el tiempo de CPU, la proporción de núcleos de CPU, el RSS máximo, el heap, las métricas de seguimiento del inicio, el retraso del bucle de eventos y métricas detalladas de la tabla de búsqueda de plugins. El script establece `OPENCLAW_GATEWAY_STARTUP_TRACE=1` en el entorno del Gateway secundario.

`/healthz` indica actividad (el servidor HTTP puede responder). `/readyz` indica disponibilidad operativa (los procesos auxiliares de plugins de inicio, los canales y el trabajo posterior a la conexión crítico para la disponibilidad han terminado). Los hooks de inicio se despachan de forma asíncrona y no forman parte de la garantía de disponibilidad. El tiempo del registro de disponibilidad es la marca de tiempo interna del Gateway, útil para la atribución en el proceso, pero no sustituye a la sonda externa `/readyz`.

Use la salida JSON o `--output` al comparar cambios. Use `--cpu-prof-dir` únicamente después de que la salida de seguimiento señale trabajo de importación, compilación o limitado por la CPU que los tiempos de las fases por sí solos no puedan explicar.

</Accordion>

<Accordion title="Reinicio del Gateway (scripts/bench-gateway-restart.ts)">

Solo macOS y Linux (usa SIGUSR1 para los reinicios dentro del proceso; falla inmediatamente en Windows). Se aplican el mismo punto de entrada compilado predeterminado y la misma sustitución `--entry scripts/run-node.mjs` que en el inicio del Gateway anterior.

```bash
pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5
pnpm test:restart:gateway -- --case default --runs 3 --restarts 3 --warmup 1
```

Identificadores de casos: `skipChannels`, `skipChannelsAcpxProbe` (sonda de inicio ACPX activada), `skipChannelsNoAcpxProbe` (sonda desactivada), `default`, `fiftyPlugins`.

La salida incluye el siguiente `/healthz`, el siguiente `/readyz`, el tiempo de inactividad, el tiempo de disponibilidad tras el reinicio, la CPU, el RSS, las métricas de seguimiento del inicio del proceso de sustitución y las métricas de seguimiento del reinicio correspondientes a la gestión de señales, el drenaje del trabajo activo, las fases de cierre, el siguiente inicio, el tiempo de disponibilidad y las instantáneas de memoria. El script establece `OPENCLAW_GATEWAY_STARTUP_TRACE=1` y `OPENCLAW_GATEWAY_RESTART_TRACE=1`.

Use este benchmark cuando un cambio afecte a las señales de reinicio, los controladores de cierre, el inicio posterior al reinicio, el cierre de procesos auxiliares, la transferencia del servicio o la disponibilidad después del reinicio. Comience con `skipChannels` para aislar la mecánica del Gateway del inicio de los canales; use `default` o casos con muchos plugins únicamente después de que el caso acotado explique la ruta de reinicio. Las métricas de seguimiento son indicios para la atribución, no veredictos: evalúe un cambio de reinicio a partir de varias muestras, el intervalo correspondiente del propietario, el comportamiento de `/healthz`/`/readyz` y el contrato de reinicio visible para el usuario.

</Accordion>

## E2E de incorporación (Docker)

Opcional; solo es necesario para las pruebas de humo de incorporación en contenedores. Flujo completo de arranque en frío en un contenedor Linux limpio:

```bash
scripts/e2e/onboard-docker.sh
```

Controla el asistente interactivo mediante una pseudoterminal, verifica los archivos de configuración, espacio de trabajo y sesión, inicia después el Gateway y ejecuta `openclaw health`.

## Prueba de humo de importación de QR (Docker)

Garantiza que el helper mantenido del entorno de ejecución de QR se cargue en los entornos de ejecución de Node compatibles con Docker (Node 24 de forma predeterminada, compatible con Node 22):

```bash
pnpm test:docker:qr
```

## Contenido relacionado

- [Pruebas](/es/help/testing)
- [Pruebas en vivo](/es/help/testing-live)
- [Pruebas de actualizaciones y plugins](/es/help/testing-updates-plugins)
