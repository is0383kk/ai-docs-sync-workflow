---
read_when:
    - Creación o ejecución de control de calidad visual en vivo para errores de OpenClaw
    - Añadir verificación antes y después para un pull request
    - Adición de escenarios de transporte en vivo para Discord, Slack, WhatsApp u otros servicios
    - Ejecución de una prueba enfocada en navegador de la interfaz de control para una referencia candidata
    - Depuración de ejecuciones de control de calidad que requieren capturas de pantalla, automatización del navegador o acceso VNC
summary: Mantis captura evidencia visual integral para comparaciones de transporte en vivo y pruebas de navegador específicas centradas únicamente en candidatos, y luego adjunta los artefactos a los PR.
title: Mantis
x-i18n:
    generated_at: "2026-07-26T05:36:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 48a1b306e37aba7e8c67139df61f3680a9aec066361aa196d88c81270337bc1b
    source_path: concepts/mantis.md
    workflow: 16
---

Mantis publica evidencia visual de CI y un comentario en el PR sobre el comportamiento de OpenClaw.
Los escenarios de transporte en vivo comparan una referencia base que se sabe que es defectuosa con una referencia candidata;
en cambio, los flujos de navegador específicos pueden demostrar un candidato frente a un transporte simulado
determinista. Discord se lanzó primero con autenticación real de bot, canales de servidor,
reacciones, hilos y un testigo en el navegador. También existen flujos de chat de Slack, Telegram y específicos de la
UI de Control; WhatsApp y Matrix no están implementados.

## Responsabilidad

- OpenClaw (`extensions/qa-lab/src/mantis/*`): entorno de ejecución de escenarios, CLI `pnpm openclaw qa mantis <command>`, esquema de evidencias.
- Laboratorio de QA (`extensions/qa-lab/src/live-transports/*`): entorno de pruebas de transporte en vivo, bots controlador/SUT, generadores de informes/evidencias.
- Crabbox (`openclaw/crabbox`): máquinas Linux preparadas, concesiones, VNC, `crabbox media preview`.
- GitHub Actions (`.github/workflows/mantis-*.yml`): puntos de entrada remotos, conservación de artefactos.
- ClawSweeper: analiza comandos de mantenedores en PR, ejecuta flujos de trabajo y publica el comentario final en el PR.

## Comandos de la CLI

Todos los comandos son `pnpm openclaw qa mantis <command>`, definidos en
`extensions/qa-lab/src/mantis/cli.ts`. Requiere `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1`
durante la compilación/ejecución (los flujos de trabajo incluidos establecen `OPENCLAW_BUILD_PRIVATE_QA=1` y
`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` antes de compilar).

| Comando                         | Propósito                                                                                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discord-smoke`                 | Verificar que el bot de Discord de Mantis pueda ver el servidor/canal, publicar y reaccionar.                                                                                 |
| `run`                           | Ejecutar un escenario de antes/después con las referencias base y candidata (solo Discord).                                                                           |
| `desktop-browser-smoke`         | Conceder/reutilizar un escritorio Crabbox, abrir un navegador visible y capturar una imagen de pantalla + vídeo.                                                                        |
| `slack-desktop-smoke`           | Conceder/reutilizar un escritorio Crabbox, ejecutar QA de Slack en él, abrir Slack Web y capturar evidencias.                                                                  |
| `telegram-desktop-builder`      | Conceder/reutilizar un escritorio Crabbox, instalar Telegram Desktop y, opcionalmente, configurar un Gateway de OpenClaw.                                                        |
| `visual-task` / `visual-driver` | Captura genérica del escritorio Crabbox con aserciones opcionales de comprensión de imágenes; `visual-driver` es la parte del controlador iniciada mediante `crabbox record --while`. |

Todos los comandos aceptan `--repo-root <path>` y `--output-dir <path>`; los comandos de Crabbox
también aceptan `--crabbox-bin`, `--provider`, `--machine-class`/`--class`,
`--lease-id`, `--idle-timeout`, `--ttl` y `--keep-lease`. Los valores predeterminados de la CLI local
para proveedor/clase son `hetzner`/`beast`, salvo que se indique lo contrario; los flujos de trabajo de CI
suelen sustituir ambos.

### `discord-smoke`

```bash
pnpm openclaw qa mantis discord-smoke \
  --output-dir .artifacts/qa-e2e/mantis/discord-smoke
```

Llama a la API REST de Discord (`https://discord.com/api/v10`) para obtener el usuario
del bot, el servidor, los canales del servidor y el canal de destino; comprueba que el
canal pertenezca al servidor y, salvo que se use `--skip-post`, publica un mensaje y
añade una reacción `👀`. Escribe `mantis-discord-smoke-summary.json` y
`mantis-discord-smoke-report.md`.

Orden de resolución del token: valor de `--token-file`, después `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
(sustituible con `--token-env`) y, por último, un archivo indicado por `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN_FILE`
(sustituible con `--token-file-env`). Los identificadores de servidor/canal proceden de
`OPENCLAW_QA_DISCORD_GUILD_ID` / `OPENCLAW_QA_DISCORD_CHANNEL_ID` (sustituibles con
`--guild-id` / `--channel-id`) y deben ser snowflakes de Discord de 17-20 dígitos. Establezca
`OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` para sustituir los identificadores
y nombres del bot/servidor/canal/mensaje por `<redacted>` en el resumen y el informe publicados.

### `run`

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-status-reactions-tool-only \
  --baseline origin/main \
  --candidate HEAD \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-status-reactions
```

Actualmente, `--transport` solo acepta `discord`. `--scenario` es uno de dos
identificadores integrados, cada uno con su propia referencia base predeterminada y las etiquetas esperadas de antes/después
(`extensions/qa-lab/src/mantis/run.runtime.ts`):

| Escenario                                   | Referencia base predeterminada                           | Resultado esperado de la referencia base                         | Resultado esperado del candidato            |
| ------------------------------------------ | ------------------------------------------ | ---------------------------------------- | ---------------------------- |
| `discord-status-reactions-tool-only`       | `0bf06e953fdda290799fc9fb9244a8f67fdae593` | `queued-only`                            | `queued -> thinking -> done` |
| `discord-thread-reply-filepath-attachment` | `81349cdc2a9d5143fd0991ed858b739e7d96e05c` | la respuesta del hilo omite el archivo adjunto `filePath` | la respuesta del hilo lo incluye     |

El valor predeterminado de `--candidate` es `HEAD`. Otras opciones: `--credential-source`
(valor predeterminado `convex`), `--credential-role` (valor predeterminado `ci`), `--provider-mode`
(valor predeterminado `live-frontier`), `--fast` (activado de forma predeterminada), `--skip-install`, `--skip-build`.

El ejecutor crea copias de trabajo `git worktree` desacopladas para la referencia base y
la candidata en `<output-dir>/worktrees/`, ejecuta `pnpm install`/`pnpm build` en
cada una (salvo que se omita) y después ejecuta
`pnpm openclaw qa discord --scenario <id> --model openai/gpt-5.4 --alt-model openai/gpt-5.4 --allow-failures`
en cada copia de trabajo. Cada flujo escribe `discord-qa-reaction-timelines.json`
junto con un par `<scenario-id>-timeline.html`/`.png`; el ejecutor vuelve a copiar esta
evidencia en `baseline/`/`candidate/`, escribe `comparison.json`,
`mantis-report.md` y `mantis-evidence.json` en el directorio de salida, y
finaliza con un código distinto de cero si la comparación no se supera (referencia base `fail` y candidata
`pass`).

El segundo escenario de Discord (`discord-thread-reply-filepath-attachment`) publica
un mensaje principal con el bot controlador, crea un hilo real, llama a la acción
`message.thread-reply` del SUT con un `filePath` local del repositorio y, a continuación, consulta
periódicamente el hilo para obtener la respuesta y el nombre de archivo del adjunto. Espera un adjunto
denominado `mantis-thread-report.md`.

### `desktop-browser-smoke`

```bash
pnpm openclaw qa mantis desktop-browser-smoke \
  --output-dir .artifacts/qa-e2e/mantis/desktop-browser
```

Concede o reutiliza un escritorio Crabbox, inicia un navegador dentro de la sesión VNC
dirigido a `--browser-url` (valor predeterminado `https://openclaw.ai`) o a un
`--html-file` renderizado, espera, toma una captura de pantalla con `scrot`, graba opcionalmente un MP4 con
`ffmpeg` y sincroniza mediante rsync `desktop-browser-smoke.png` / `.mp4` / `remote-metadata.json`
de vuelta a `--output-dir`.

Opciones:

- `--lease-id <cbx_...>` reutiliza un escritorio preparado en lugar de crear uno.
- `--browser-profile-dir <remote-path>` reutiliza un directorio remoto de datos de usuario de Chrome para que un escritorio persistente mantenga la sesión iniciada entre ejecuciones (se utiliza para un perfil de visor de Discord Web de larga duración).
- `--browser-profile-archive-env <name>` restaura antes del inicio un archivo de perfil de Chrome `.tgz` en base64 desde esa variable de entorno (valor predeterminado `OPENCLAW_MANTIS_BROWSER_PROFILE_TGZ_B64`); se utiliza para testigos con sesión iniciada como Discord Web.
- `--video-duration <seconds>` controla la duración de la captura MP4 (valor predeterminado 10s).
- `--keep-lease` (o `OPENCLAW_MANTIS_KEEP_VM=1`) mantiene abierta para inspección mediante VNC una concesión creada durante esta ejecución; las ejecuciones fallidas que hayan creado una concesión también la mantienen de forma predeterminada.

Para las evidencias de Discord Web, Mantis utiliza una cuenta de visor dedicada, no un token
de bot. El oráculo REST de Discord (mediante `qa discord`) sigue siendo la fuente autoritativa; cuando
se establece `OPENCLAW_QA_DISCORD_CAPTURE_UI_METADATA=1`, el escenario también escribe un
artefacto con una URL de Discord Web y `OPENCLAW_QA_DISCORD_KEEP_THREADS=1` mantiene el
hilo abierto el tiempo suficiente para que el navegador lo abra.

El flujo de trabajo de GitHub prefiere un perfil de visor persistente mediante
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` (los archivos de perfil completos pueden superar
el límite de tamaño de secretos de GitHub); para perfiles pequeños/iniciales, puede restaurar en su lugar un
`.tgz` en base64 desde `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64`. Si no se configura
ninguna de las dos fuentes, el flujo de trabajo sigue publicando las capturas de pantalla deterministas
de la referencia base y la candidata, y registra que se omitió el testigo con sesión iniciada.

### `slack-desktop-smoke`

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --output-dir .artifacts/qa-e2e/mantis/slack-desktop \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Concede o reutiliza un escritorio Crabbox, sincroniza la copia de trabajo con la máquina virtual, ejecuta
`pnpm openclaw qa slack` en ella, abre Slack Web en el navegador VNC,
captura el escritorio y copia localmente tanto los artefactos de QA de Slack (`slack-qa/`) como
la captura de pantalla/vídeo de VNC. Esta es la única configuración de Mantis en la que el
Gateway del SUT y el navegador se ejecutan en la misma máquina virtual.

Con `--gateway-setup`, el comando crea un directorio de inicio persistente y desechable de OpenClaw
en `$HOME/.openclaw-mantis/slack-openclaw` dentro de la máquina virtual, modifica la configuración de
Socket Mode de Slack para el canal de destino, inicia
`openclaw gateway run --dev --allow-unconfigured --port 38973` y deja
Chrome ejecutándose en la sesión VNC; si se omite `--gateway-setup`, se ejecuta en su lugar el flujo normal
de QA de Slack de bot a bot.

Variables de entorno obligatorias para `--credential-source env` (el valor predeterminado local es `env`; el valor
predeterminado del rol es `maintainer`):

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`
- `OPENCLAW_LIVE_OPENAI_KEY` para el flujo de modelo remoto (si solo se establece `OPENAI_API_KEY`
  localmente, Mantis lo copia en `OPENCLAW_LIVE_OPENAI_KEY` antes de
  invocar Crabbox)

Con `--credential-source convex`, Mantis obtiene una concesión de la credencial del SUT de Slack desde
el grupo compartido antes de crear la máquina virtual y reenvía el identificador del canal, el token de la aplicación y
el token del bot a la máquina virtual como variables de entorno `OPENCLAW_MANTIS_SLACK_*`, de modo que los flujos de trabajo de GitHub
solo necesitan el secreto del intermediario Convex, no los tokens de Slack sin procesar.

Otras opciones: `--slack-url <url>` abre una URL específica (de lo contrario, Mantis obtiene
`https://app.slack.com/client/<team>/<channel>` a partir de `auth.test`);
`--slack-channel-id <id>` establece el canal de la lista de permitidos del Gateway;
`OPENCLAW_MANTIS_SLACK_BROWSER_PROFILE_DIR` controla el perfil persistente de Chrome
dentro de la máquina virtual (valor predeterminado `$HOME/.config/openclaw-mantis/slack-chrome-profile`);
`--approval-checkpoints` ejecuta los escenarios nativos de aprobación de Slack
(`slack-approval-exec-native`, `slack-approval-plugin-native`) y renderiza
capturas de pantalla de puntos de control pendientes/resueltos en lugar de configurar el Gateway (es
mutuamente excluyente con `--gateway-setup`); `--hydrate-mode source|prehydrated`,
`--provider-mode`, `--model`, `--alt-model` y `--fast` se transfieren al
flujo en vivo de Slack.

Las capturas de pantalla de los puntos de control de aprobación se renderizan a partir del mensaje de la API de Slack que
observó el escenario, no de la UI de Slack en vivo; `slack-desktop-smoke.png` solo es
una prueba de Slack Web cuando el perfil del navegador de la concesión ya tenía la sesión
iniciada.

### `telegram-desktop-builder`

```bash
pnpm openclaw qa mantis telegram-desktop-builder \
  --credential-source convex \
  --credential-role maintainer \
  --keep-lease
```

Concede o reutiliza un escritorio Crabbox, instala la aplicación nativa Telegram Desktop para Linux,
restaura opcionalmente un archivo de sesión de usuario, configura OpenClaw con el
token de bot del SUT de Telegram concedido, inicia
`openclaw gateway run --dev --allow-unconfigured --port 38974`, publica un
mensaje de disponibilidad del bot controlador en el grupo privado concedido y, a continuación, captura una
imagen de pantalla y un MP4. Un token de bot solo configura OpenClaw; nunca inicia
sesión en Telegram Desktop. El visor de escritorio es una sesión de usuario de Telegram independiente,
restaurada desde `--telegram-profile-archive-env <name>` o iniciada manualmente
mediante VNC y mantenida activa con `--keep-lease`.

Opciones: `--lease-id <cbx_...>` vuelve a ejecutar el proceso en una máquina virtual que ya tiene una sesión iniciada en
Telegram Desktop; `--telegram-profile-archive-env <name>` restaura antes del inicio un archivo de perfil
`.tgz` en base64; `--telegram-profile-dir <remote-path>`
establece el directorio remoto del perfil (valor predeterminado `$HOME/.local/share/TelegramDesktop`);
`--no-gateway-setup` únicamente instala y abre Telegram Desktop;
los valores predeterminados de `--credential-source`/`--credential-role` son `convex`/`maintainer`.

## Manifiesto de evidencias

Cada escenario que publica en un PR escribe `mantis-evidence.json` junto a
su informe:

```json
{
  "schemaVersion": 1,
  "id": "discord-status-reactions",
  "title": "QA de reacciones de estado de Discord de Mantis",
  "summary": "Resumen principal legible para personas para el comentario del PR.",
  "scenario": "discord-status-reactions-tool-only",
  "comparison": {
    "baseline": { "sha": "...", "status": "fail", "expected": "queued-only" },
    "candidate": { "sha": "...", "status": "pass", "expected": "queued -> thinking -> done" },
    "pass": true
  },
  "artifacts": [
    {
      "kind": "timeline",
      "lane": "baseline",
      "label": "Referencia solo en cola",
      "path": "baseline/timeline.png",
      "targetPath": "baseline.png",
      "alt": "Cronología de referencia de Discord",
      "width": 420
    }
  ]
}
```

El `path` del artefacto es relativo al directorio del manifiesto; `targetPath` es
relativo al prefijo de artefactos R2/S3 configurado. `scripts/mantis/publish-pr-evidence.mjs`
rechaza el recorrido de rutas y omite las entradas con `"required": false` cuando
falta el archivo.

Tipos de artefactos: `timeline` (captura de pantalla determinista del antes/después),
`desktopScreenshot` (captura de pantalla de VNC/navegador), `motionPreview` (GIF animado
en línea de la grabación), `motionClip` (MP4 recortado según el movimiento), `fullVideo` (grabación
completa), `metadata` (archivo complementario JSON/registro), `report` (informe Markdown).

Disposición de artefactos de una ejecución en disco:

```text
.artifacts/qa-e2e/mantis/<run-id>/
  mantis-report.md
  mantis-evidence.json
  baseline/
  candidate/
  comparison.json
```

Las capturas de pantalla son pruebas, no secretos, pero aun así requieren medidas de
ocultación: pueden aparecer nombres de canales privados, nombres de usuario o contenido de
mensajes. Establezca `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` para las cargas públicas de artefactos; está
habilitado de forma predeterminada en los flujos de trabajo de GitHub de Discord/Slack/Telegram.

## Automatización de GitHub

`scripts/mantis/publish-pr-evidence.mjs` es el publicador reutilizable. Los flujos de trabajo
lo invocan con el manifiesto, el PR de destino, la raíz de destino de los artefactos, el marcador del comentario,
la URL de los artefactos, la URL de la ejecución y el origen de la solicitud. Carga los artefactos declarados en
el bucket R2 de Mantis, crea un comentario de PR que prioriza el resumen con
imágenes/vistas previas en línea y vídeos enlazados y, a continuación, actualiza el comentario con el marcador existente o
crea uno nuevo. Variables de entorno requeridas:

- `MANTIS_ARTIFACT_R2_ACCESS_KEY_ID`
- `MANTIS_ARTIFACT_R2_SECRET_ACCESS_KEY`
- `MANTIS_ARTIFACT_R2_BUCKET` (los flujos de trabajo establecen `openclaw-crabbox-artifacts`)
- `MANTIS_ARTIFACT_R2_ENDPOINT`
- `MANTIS_ARTIFACT_R2_REGION` (los flujos de trabajo establecen `auto`)
- `MANTIS_ARTIFACT_R2_PUBLIC_BASE_URL` (los flujos de trabajo establecen `https://artifacts.openclaw.ai`)

Los comentarios se publican mediante la aplicación de GitHub de Mantis (`MANTIS_GITHUB_APP_ID` /
`MANTIS_GITHUB_APP_PRIVATE_KEY`), no mediante `github-actions[bot]`, y utilizan un comentario
marcador oculto como clave de inserción o actualización.

| Flujo de trabajo                   | Desencadenador                                                                              | Qué hace                                                                                                                                                                                                                                                                                                                    |
| --------------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Mantis Discord Smoke`            | ejecución manual                                                                            | Ejecuta `discord-smoke` en una referencia elegida.                                                                                                                                                                                                                                                                       |
| `Mantis Discord Status Reactions` | comentario de PR o ejecución manual                                                              | Crea árboles de trabajo separados para la referencia y el candidato, ejecuta `discord-status-reactions-tool-only` en cada uno, representa la cronología de cada vía en un navegador de escritorio de Crabbox, genera vistas previas GIF/MP4 recortadas según el movimiento con `crabbox media preview`, carga los artefactos y publica pruebas en línea en el PR.                                 |
| `Mantis Scenario`                 | ejecución manual                                                                            | Despachador genérico: recibe `scenario_id` (`discord-status-reactions-tool-only`, `discord-thread-reply-filepath-attachment`, `slack-desktop-smoke`, `telegram-live`, `telegram-desktop-proof`, `web-ui-chat-proof`), `baseline_ref`, `candidate_ref`, `pr_number` y los reenvía al flujo de trabajo del escenario correspondiente. |
| `Mantis Slack Desktop Smoke`      | ejecución manual                                                                            | Reserva un escritorio Linux de Crabbox (de forma predeterminada, `aws`, con la opción de `hetzner`), ejecuta `slack-desktop-smoke --gateway-setup` en el candidato, graba el escritorio, genera una vista previa de movimiento, carga los artefactos y publica pruebas en el PR cuando se proporciona un número de PR.                                                      |
| `Mantis Telegram Live`            | comentario de PR o ejecución manual                                                              | Ejecuta la vía de QA en vivo de Telegram mediante la API de bot (`openclaw qa telegram`), escribe `mantis-evidence.json` a partir del resumen de QA, representa el HTML de las pruebas ocultadas mediante un navegador de escritorio de Crabbox, genera un GIF de movimiento y publica pruebas en el PR. Esta vía no requiere iniciar sesión en Telegram Web.                               |
| `Mantis Telegram Desktop Proof`   | etiqueta de PR del mantenedor (`mantis: telegram-visible-proof`) más comentario de PR, o ejecución manual | Prueba agéntica nativa del antes/después en Telegram Desktop. Entrega el PR, las referencias de referencia/candidato y las instrucciones del mantenedor a Codex, que ejecuta la vía de pruebas de Telegram Desktop con un usuario real en Crabbox para ambas referencias y publica una tabla de pruebas de 2 columnas en el PR.                                                              |
| `Mantis Web UI Chat Proof`        | comentario de PR o ejecución manual                                                              | Ejecuta la prueba específica de chat de la interfaz de control de OpenClaw con Playwright en el candidato, verifica que el navegador envíe mediante el Gateway simulado, captura artefactos de captura de pantalla/vídeo y publica pruebas en el PR. Esta vía solo prueba el chat web, no WinUI/aplicaciones nativas ni pruebas visuales arbitrarias.                           |

Tanto `Mantis Discord Status Reactions` como `Mantis Telegram Live` aceptan
`baseline_ref`/`candidate_ref` (o `baseline=`/`candidate=` en un comentario de PR)
y validan que el SHA resuelto sea un ancestro de `origin/main`, una
etiqueta de versión (`v*`) o la cabecera de un PR abierto antes de ejecutarse con
credenciales que contienen secretos.

Desencadenadores mediante comentarios desde un PR con acceso de escritura/mantenimiento/administración:

```text
@openclaw-mantis discord status reactions
@openclaw-mantis discord status reactions baseline=origin/main candidate=HEAD
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
@openclaw-mantis web ui chat
@openclaw-mantis web-ui-chat candidate=HEAD
```

Los desencadenadores mediante comentarios de Telegram utilizan de forma predeterminada el SHA de la cabecera del PR como candidato y
`telegram-status-command` como escenario; aceptan `provider=aws|hetzner` y
`lease=<cbx_...>` para seleccionar un proveedor específico de Crabbox o un
escritorio precalentado. `Mantis Telegram Desktop Proof` solo responde a un comentario de PR cuando
el PR ya tiene la etiqueta `mantis: telegram-visible-proof`.

Los desencadenadores mediante comentarios del chat de la interfaz web utilizan de forma predeterminada el SHA de la cabecera del PR como candidato. Ejecutan
la prueba de chat de la interfaz de control con un Gateway simulado y publican artefactos del navegador; para
otras páginas web y superficies de aplicaciones nativas, utilice pruebas normales con Playwright/navegador,
capturas de pantalla del mantenedor, Crabbox o artefactos locales.

ClawSweeper también puede ejecutar un escenario directamente:

```text
@clawsweeper mantis discord discord-status-reactions-tool-only
```

## Máquinas y secretos

Los valores predeterminados de Crabbox en la CLI local son `--provider hetzner --class beast`; sustitúyalos
con `--provider`, `--class`/`--machine-class` o
`OPENCLAW_MANTIS_CRABBOX_PROVIDER` / `OPENCLAW_MANTIS_CRABBOX_CLASS`. Los flujos de trabajo de
GitHub suelen sustituir ambos (por ejemplo, `--class standard` y la entrada de elección del proveedor
`aws`/`hetzner` del flujo de trabajo de Slack). Si un proveedor es demasiado
lento o no está disponible, añádalo detrás de la misma interfaz de Crabbox en lugar de
codificar una alternativa.

Configuración básica de la máquina virtual: Linux con Chrome/Chromium compatible con escritorio, acceso CDP, VNC/
noVNC, Node 22.22.3+, 24.15+ o 25.9+ y pnpm, un checkout de OpenClaw y
acceso saliente al transporte de destino, GitHub, proveedores de modelos y el
intermediario de credenciales.

Nombres de credenciales y variables de entorno utilizados en los comandos y flujos de trabajo de Mantis:

- `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- El `qa mantis run --credential-source env` local también requiere
  `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`, `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
  y `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID`. Los flujos de trabajo de GitHub normalmente utilizan
  `--credential-source convex` y las credenciales del intermediario indicadas a continuación, en lugar de tokens
  sin procesar del bot de Discord.
- `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` para cargas públicas de artefactos
- `OPENCLAW_QA_CONVEX_SITE_URL`, `OPENCLAW_QA_CONVEX_SECRET_CI`
- `OPENAI_API_KEY` (o el valor específico de las pruebas de Telegram Desktop
  `OPENCLAW_MANTIS_AGENT_OPENAI_API_KEY`)
- `CRABBOX_COORDINATOR` / `CRABBOX_COORDINATOR_TOKEN` (los flujos de trabajo también aceptan
  `OPENCLAW_QA_MANTIS_CRABBOX_COORDINATOR` / `_TOKEN` como alternativa y los asignan
  a los nombres simples antes de invocar Crabbox)
- `CRABBOX_ACCESS_CLIENT_ID`, `CRABBOX_ACCESS_CLIENT_SECRET`
- `MANTIS_GITHUB_APP_ID`, `MANTIS_GITHUB_APP_PRIVATE_KEY`

El ejecutor de Mantis nunca debe mostrar tokens de bot de Discord/Slack/Telegram,
claves de API de proveedores, cookies del navegador, contenido de perfiles de autenticación, contraseñas de VNC ni
cargas útiles de credenciales sin procesar. Si se filtra un token en una incidencia, PR, chat o registro,
rótelo después de almacenar el secreto de sustitución.

## Resultados de la ejecución

Los escenarios de transporte del antes/después distinguen estos resultados para que un entorno
inestable no se interprete como una regresión del producto:

- **Error reproducido**: la referencia falló de la forma que esperaba el escenario.
- **Fallo del entorno de pruebas**: la configuración del entorno, las credenciales, la API de transporte, el navegador
  o el proveedor fallaron antes de que el oráculo fuera significativo.

Las pruebas del navegador solo para el candidato indican si el candidato superó las aserciones del
Gateway simulado y de la interfaz de usuario visible; no afirman haber reproducido el comportamiento de referencia.

## Adición de un escenario

Los escenarios de transporte en vivo se definen en TypeScript para cada transporte (consulte
`MANTIS_SCENARIO_CONFIGS` en `extensions/qa-lab/src/mantis/run.runtime.ts` para
la forma del antes/después de Discord), no mediante un formato de archivo declarativo independiente.
Cada escenario necesita: id y título, transporte, credenciales requeridas, política de referencia
de referencia, política de referencia del candidato, parche de configuración de OpenClaw, pasos de configuración/estímulo,
oráculo esperado para la referencia y el candidato, destinos de captura visual, presupuesto de
tiempo de espera y pasos de limpieza.

Las pruebas específicas del navegador solo para el candidato pueden utilizar una prueba E2E determinista dedicada
y un flujo de trabajo. Mantenga explícito su alcance, valide la referencia del candidato antes de
la ejecución, aísle la publicación respaldada por secretos y emita el mismo contrato de
manifiesto de pruebas.

Prefiera oráculos pequeños y tipados en lugar de comprobaciones visuales: estado de reacciones de Discord o
referencias de mensajes, estado de la API de `ts`/reacciones de hilos de Slack, identificadores
y cabeceras de mensajes de correo electrónico. Utilice capturas de pantalla del navegador cuando la interfaz de usuario sea el único elemento observable fiable
y mantenga las comprobaciones visuales como complemento de un oráculo de la API de la plataforma cuando exista.

Después de Discord, Slack y Telegram, la misma forma de ejecutor se extiende a WhatsApp
(inicio de sesión mediante QR, reidentificación, entrega, contenido multimedia y reacciones) y Matrix
(salas cifradas, relaciones de hilos/respuestas y reanudación tras reinicio); ninguno está
implementado todavía.

## Preguntas abiertas

- ¿Qué bot de Discord debe actuar como controlador y cuál como SUT cuando se reutiliza el bot
  Mantis existente?
- ¿Durante cuánto tiempo debe conservar GitHub los artefactos de Mantis para los PR?
- ¿Cuándo debe ClawSweeper recomendar automáticamente un escenario de Mantis en lugar de
  esperar una orden de un responsable de mantenimiento?
- ¿Deben ocultarse los datos sensibles de las capturas de pantalla o recortarse antes de subirlas a PR públicos?
