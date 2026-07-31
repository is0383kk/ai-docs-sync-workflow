---
read_when:
    - Usas `openclaw browser` y quieres ejemplos de tareas comunes
    - Se desea controlar un navegador que se ejecuta en otra máquina mediante un host de Node
    - Quieres conectarte a tu Chrome local con la sesión iniciada mediante Chrome MCP
summary: Referencia de la CLI para `openclaw browser` (ciclo de vida, perfiles, pestañas, acciones, estado y depuración)
title: Navegador
x-i18n:
    generated_at: "2026-07-26T05:08:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 62eb41248cda87cef96be7b0dfe3e0d36a9d3e1ee55c165bd8e3efd68d1e9a5e
    source_path: cli/browser.md
    workflow: 16
---

# `openclaw browser`

Gestiona la superficie de control del navegador de OpenClaw y ejecuta acciones del navegador: ciclo de vida, perfiles, pestañas, instantáneas, capturas de pantalla, navegación, entrada, emulación de estado y depuración.

Relacionado: [Herramienta de navegador](/es/tools/browser)

## Opciones comunes

- `--url <gatewayWsUrl>`: URL de WebSocket del Gateway (de forma predeterminada, usa la configuración).
- `--token <token>`: token del Gateway (si es necesario).
- `--timeout <ms>`: tiempo de espera de la solicitud en ms (valor predeterminado: `30000`).
- `--expect-final`: espera una respuesta final del Gateway.
- `--browser-profile <name>`: elige un perfil de navegador (valor predeterminado: `openclaw` o `browser.defaultProfile`).
- `--json`: salida legible por máquinas (cuando sea compatible). Esta es una opción del nivel del navegador, por lo que
  debe colocarse antes del subcomando para obtener una forma inequívoca, como
  `openclaw browser --json status`. También funciona colocarla al final, como en
  `openclaw browser status --json`, cuando el comando secundario seleccionado no
  define su propia opción `--json`.

## Inicio rápido (local)

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

Los agentes pueden ejecutar la misma comprobación de disponibilidad con `browser({ action: "doctor" })`.

## Solución rápida de problemas

Si `start` falla con `not reachable after start`, primero deben solucionarse los problemas de disponibilidad de CDP. Si `start` y `tabs` se ejecutan correctamente, pero `open` o `navigate` fallan, el plano de control del navegador funciona correctamente y el fallo suele deberse a un bloqueo de la política SSRF de navegación.

Secuencia mínima:

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

Guía detallada: [Solución de problemas del navegador](/es/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## Ciclo de vida

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

- `doctor --deep` añade una comprobación de instantánea en vivo: resulta útil cuando la disponibilidad básica de CDP es correcta, pero se necesita demostrar que se puede inspeccionar la pestaña actual.
- Para un perfil local administrado en ejecución, `status` y `doctor` muestran diagnósticos
  gráficos almacenados en caché de Chrome: clasificación de hardware/software, renderizador,
  backend, dispositivo/controlador, detalles de las funciones y de su estado de desactivación, y capacidades
  de vídeo acelerado. `openclaw browser --json status` devuelve la carga útil estructurada completa.
  El estado pasivo nunca inicia Chrome únicamente para recopilar estos datos.
- `stop` cierra la sesión de control activa y elimina las anulaciones temporales de emulación incluso para `attachOnly` y perfiles CDP remotos en los que OpenClaw no inició el proceso del navegador. En los perfiles locales administrados, `stop` también detiene el proceso del navegador iniciado.
- `start --headless` solo se aplica a esa solicitud de inicio y únicamente cuando OpenClaw inicia un navegador local administrado. No reescribe `browser.headless` ni la configuración del perfil, y no tiene efecto en un navegador que ya se esté ejecutando.
- En hosts Linux sin `DISPLAY` ni `WAYLAND_DISPLAY`, los perfiles locales administrados se ejecutan automáticamente sin interfaz gráfica, salvo que `OPENCLAW_BROWSER_HEADLESS=0`, `browser.headless=false` o `browser.profiles.<name>.headless=false` soliciten explícitamente un navegador visible.

## Si falta el comando

Si `openclaw browser` es un comando desconocido, debe comprobarse `plugins.allow` en `~/.openclaw/openclaw.json`. Cuando `plugins.allow` esté presente, debe incluirse explícitamente el plugin de navegador incluido, salvo que la configuración ya contenga un bloque raíz `browser`:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Un bloque raíz `browser` explícito (por ejemplo, `browser.enabled=true` o `browser.profiles.<name>`) también activa el plugin de navegador incluido con una lista restrictiva de plugins permitidos.

Relacionado: [Herramienta de navegador](/es/tools/browser#missing-browser-command-or-tool)

## Perfiles

Los perfiles son configuraciones con nombre para el enrutamiento del navegador:

- `openclaw` (valor predeterminado): inicia una instancia dedicada de Chrome administrada por OpenClaw o se conecta a ella (directorio de datos de usuario aislado).
- `user`: controla la sesión existente de Chrome en la que se ha iniciado sesión mediante Chrome DevTools MCP.
- perfiles CDP personalizados: apuntan a un endpoint CDP local o remoto.

```bash
openclaw browser profiles
openclaw browser system-profiles
openclaw browser system-profiles --browser brave
openclaw browser import-profile --browser chrome --system Default --into imported
openclaw browser import-profile --system "Profile 1" --into work --domains google.com,youtube.com
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

Puede utilizarse un perfil específico con `--browser-profile <name>` en cualquier subcomando, por ejemplo, `openclaw browser --browser-profile work tabs`.

En macOS, `system-profiles` enumera los perfiles reales de Chrome, Brave, Edge o Chromium disponibles en el host. `import-profile` descifra sus cookies después de una solicitud de consentimiento del Llavero de macOS/Touch ID y las inyecta en un perfil nuevo administrado por OpenClaw. Solo importa cookies; el almacenamiento local e IndexedDB no se modifican. Algunas sesiones de Google utilizan credenciales de sesión vinculadas al dispositivo (DBSC) y pueden seguir requiriendo una nueva autenticación después de la importación.

Cuando la aplicación de macOS utiliza un Gateway local, puede ofrecer esta importación una vez y establecer el perfil importado aislado como predeterminado para la navegación de los agentes. La importación siempre requiere un clic explícito; si se completa correctamente o se descarta, se suprimen las solicitudes automáticas posteriores, y **Settings → General → Browser login** sigue disponible para volver a importar.

La importación de perfiles del sistema está activada de forma predeterminada. Establezca `browser.allowSystemProfileImport=false` para desactivar tanto las importaciones mediante la CLI como las iniciadas por agentes. La importación es local al host y no puede ejecutarse mediante el proxy del Node del navegador.

## Pestañas

```bash
openclaw browser tabs
openclaw browser tab new --label docs
openclaw browser tab label t1 docs
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai --label docs
openclaw browser focus docs
openclaw browser close t1
```

`tabs` devuelve primero `suggestedTargetId`, seguido del `tabId` estable (como `t1`), la etiqueta opcional y el `targetId` sin procesar. Vuelva a pasar `suggestedTargetId` a `focus`, `close`, las instantáneas y las acciones. Asigne una etiqueta con `open --label`, `tab new --label` o `tab label`; se aceptan etiquetas, identificadores de pestaña, identificadores de destino sin procesar y prefijos únicos de identificadores de destino. El campo de solicitud sigue denominándose `targetId` por compatibilidad, pero acepta cualquiera de estas referencias de pestaña.

Los identificadores de destino sin procesar son referencias de diagnóstico volátiles, no memoria duradera del agente: cuando Chromium reemplaza el destino sin procesar subyacente durante una navegación o el envío de un formulario, OpenClaw conserva el `tabId`/la etiqueta estable asociado a la pestaña de reemplazo cuando puede demostrar la correspondencia. Se recomienda `suggestedTargetId`.

## Instantáneas, capturas de pantalla y acciones

Instantánea:

```bash
openclaw browser snapshot
openclaw browser snapshot --urls
```

Captura de pantalla:

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
openclaw browser screenshot --labels
```

- `--full-page` se utiliza únicamente para capturas de página; no puede combinarse con `--ref` ni `--element`.
- Los perfiles `existing-session` / `user` admiten capturas de pantalla de páginas y capturas de pantalla `--ref` procedentes de la salida de instantáneas, pero no capturas de pantalla CSS `--element`.
- `--labels` superpone las referencias de la instantánea actual sobre la captura de pantalla. En los perfiles basados en Playwright, funciona con `--full-page` (superposición de página completa), `--ref` (superposición de recorte de elemento mediante una referencia ARIA) y `--element` (superposición de recorte de elemento mediante un selector CSS); en los modos de recorte de elemento, las etiquetas se proyectan con respecto al elemento. La respuesta también incluye una matriz `annotations` (se omite cuando está vacía) con el cuadro delimitador de cada referencia: `ref`, `number`, `role`, `name` opcional y `box: {x, y, width, height}` en el espacio de coordenadas de la imagen capturada (ventana gráfica / página completa / relativo al elemento).
  Los perfiles `existing-session` renderizan una superposición de chrome-mcp en las capturas de pantalla de páginas, pero no utilizan el asistente de proyección de Playwright ni incluyen `annotations`; las capturas de pantalla CSS `--element` no son compatibles en ellos. Sin Playwright ni chrome-mcp, las capturas de pantalla con etiquetas no están disponibles.
- `snapshot --urls` añade los destinos de enlaces detectados a las instantáneas para IA, de modo que los agentes puedan elegir destinos de navegación directa en lugar de deducirlos únicamente a partir del texto de los enlaces.

Navegación/clic/escritura (automatización de la interfaz de usuario basada en referencias):

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser click-coords 120 340
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
```

`evaluate --fn` acepta el código fuente de una función, una expresión o el cuerpo de una instrucción. Los cuerpos de instrucciones se encapsulan como funciones asíncronas, por lo que debe utilizarse `return` para el valor que se quiera devolver. Utilice `--timeout-ms` cuando la función ejecutada en la página pueda necesitar más tiempo que el tiempo de espera predeterminado de evaluación. `browser.evaluateEnabled=false` (valor predeterminado: `true`) desactiva tanto `evaluate` como `wait --fn`.

Las respuestas de las acciones devuelven el `targetId` sin procesar actual después de que una acción provoque el reemplazo de una página, cuando OpenClaw puede demostrar cuál es la pestaña de reemplazo. Aun así, los scripts deben almacenar y pasar `suggestedTargetId`/etiquetas para los flujos de trabajo de larga duración.

Asistentes para archivos y cuadros de diálogo:

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser upload media://inbound/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
```

Los perfiles administrados de Chrome guardan las descargas normales activadas mediante un clic en el directorio de descargas de OpenClaw (`/tmp/openclaw/downloads` de forma predeterminada, o la raíz temporal configurada). Utilice `waitfordownload` o `download` cuando el agente necesite esperar un archivo específico y devolver su ruta; esos mecanismos de espera explícitos controlan la siguiente descarga. Las cargas aceptan archivos de la raíz de cargas temporales de OpenClaw y contenido multimedia entrante administrado por OpenClaw, incluidas referencias `media://inbound/<id>` y `media/inbound/<id>` relativas al entorno aislado. Se rechazan las referencias multimedia anidadas, el recorrido de directorios y las rutas locales arbitrarias.

Cuando una acción abre un cuadro de diálogo modal, la respuesta de la acción devuelve `blockedByDialog` con `browserState.dialogs.pending`; pase `--dialog-id` para responder directamente. Los cuadros de diálogo gestionados fuera de OpenClaw aparecen en `browserState.dialogs.recent`.

Acciones por lotes:

```bash
openclaw browser batch --actions '[{"kind":"wait","timeMs":500},{"kind":"click","ref":"12"},{"kind":"type","ref":"23","text":"hello"}]'
openclaw browser batch --actions-file plan.json
openclaw browser batch --actions-file - --continue
```

`openclaw browser batch` envía una solicitud `kind="batch"` `/act` con acciones `BrowserActRequest` anidadas (`wait`, `click`, `type`, `evaluate`, ...), no `open`/`navigate`/`snapshot`/`screenshot`, que son subcomandos de la CLI, no tipos de `/act`. `--continue` establece `stopOnError=false` (de forma predeterminada, se detiene tras el primer error); `--target-id` limita todo el lote a una sola pestaña. Una acción anidada fallida hace que el comando termine con un código distinto de cero; use `--json` para conservar la respuesta `results` ordenada. Consulte [CLI de lotes del navegador](/es/tools/browser-control#browser-batch-cli) para conocer el contrato completo (ciclo de vida de las referencias, conflictos de identificadores de destino y resumen de errores). `batch` no es compatible con perfiles `profile="user"` ni de sesión existente.

## Estado y almacenamiento

Ventana gráfica y emulación:

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

Cookies y almacenamiento:

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## Depuración

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## Chrome existente mediante MCP

Use el perfil `user` integrado o cree su propio perfil `existing-session`:

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser create-profile --name chrome-port --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser --browser-profile chrome-live tabs
```

La ruta predeterminada de sesión existente es la conexión automática de Chrome MCP únicamente en el host. Si el navegador ya se está ejecutando con un punto de conexión de DevTools, pase `--cdp-url` para que Chrome MCP se conecte a ese punto de conexión. Para Docker, Browserless u otras configuraciones remotas que no necesiten la semántica de Chrome MCP, use un perfil CDP.

Límites actuales de las sesiones existentes:

- Las acciones basadas en instantáneas usan referencias, no selectores CSS.
- Las solicitudes `act` compatibles usan un valor predeterminado integrado de 60000 ms cuando los invocadores omiten `timeoutMs`; el valor `timeoutMs` de cada llamada sigue teniendo prioridad.
- `click` solo admite el clic izquierdo.
- `type` no admite `slowly=true`.
- `press` no admite `delayMs`.
- `hover`, `scrollintoview`, `drag`, `select` y `fill` rechazan las anulaciones del tiempo de espera por llamada; `evaluate` acepta `--timeout-ms`.
- `select` solo admite un valor.
- `wait --load networkidle` no es compatible (funciona en perfiles administrados y perfiles CDP sin procesar/remotos).
- La carga de archivos requiere `--ref` / `--input-ref`, no admite `--element` de CSS y permite un archivo a la vez.
- Los enlaces de diálogo no admiten `--timeout`.
- Las capturas de pantalla admiten capturas de página y `--ref`, pero no `--element` de CSS.
- `responsebody`, la interceptación de descargas, la exportación a PDF y las acciones por lotes siguen requiriendo un navegador administrado o un perfil CDP sin procesar.

## Control remoto del navegador (proxy del host del Node)

Si el Gateway se ejecuta en una máquina distinta de la del navegador, ejecute un **host del Node** en la máquina que tenga Chrome/Brave/Edge/Chromium. El Gateway redirige las acciones del navegador a ese Node; no se requiere un servidor de control del navegador independiente.

Use `gateway.nodes.browser.mode` para controlar el enrutamiento automático y `gateway.nodes.browser.node` para fijar un Node específico si hay varios conectados.

Seguridad y configuración remota: [Herramienta del navegador](/es/tools/browser), [Acceso remoto](/es/gateway/remote), [Tailscale](/es/gateway/tailscale), [Seguridad](/es/gateway/security)

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Navegador](/es/tools/browser)
