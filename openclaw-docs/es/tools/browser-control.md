---
read_when:
    - Automatización o depuración del navegador del agente mediante la API de control local
    - Buscando la referencia de la CLI `openclaw browser`
    - Adición de automatización personalizada del navegador con instantáneas y referencias
summary: API de control del navegador de OpenClaw, referencia de la CLI y acciones de scripting
title: API de control del navegador
x-i18n:
    generated_at: "2026-07-26T05:30:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

Para la instalación, la configuración y la solución de problemas, consulte [Navegador](/es/tools/browser).
Esta página es la referencia de la API HTTP de control local, la CLI `openclaw browser`
y los patrones de scripting (instantáneas, referencias, esperas y flujos de depuración).

## API de control (opcional)

Solo para integraciones locales, el Gateway expone una pequeña API HTTP de bucle invertido.
Este servidor independiente es opcional: establezca la variable de entorno
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1` en el entorno del servicio del Gateway
y reinicie el Gateway para habilitar los endpoints HTTP. Sin
esta variable, el entorno de ejecución de control del navegador sigue funcionando mediante la CLI y
las herramientas del agente, pero nada escucha en el puerto de control de bucle invertido.

- Estado/inicio/detención: `GET /`, `GET /doctor`, `POST /start`, `POST /stop`, `POST /reset-profile`
- Perfiles: `GET /profiles`, `POST /profiles/create`, `DELETE /profiles/:name`
- Pestañas: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`, `POST /tabs/action`
- Instantánea/captura de pantalla: `GET /snapshot`, `POST /screenshot`
- Acciones: `POST /navigate`, `POST /act`
- Hooks: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Descargas: `POST /download`, `POST /wait/download`
- Permisos: `POST /permissions/grant`
- Depuración: `GET /console`, `POST /pdf`
- Depuración: `GET /errors`, `GET /requests`, `GET /dialogs`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Red: `POST /response/body`
- Estado: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- Estado: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Configuración: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

`POST /tabs/action` es la forma por lotes que la CLI utiliza internamente para
los subcomandos `browser tab` (`{"action":"new"|"label"|"select"|"close"|"list", ...}`);
para scripts directos, se recomienda usar las rutas de pestaña de propósito específico anteriores.

Todos los endpoints aceptan `?profile=<name>`. `POST /start?headless=true` solicita un
inicio puntual sin interfaz gráfica para perfiles locales administrados sin cambiar la configuración
persistente del navegador; los perfiles de solo conexión, CDP remoto y sesión existente rechazan
esa sustitución porque OpenClaw no inicia esos procesos del navegador.

Para los endpoints de pestañas, `targetId` es el nombre del campo de compatibilidad. Se recomienda pasar
`suggestedTargetId` desde `GET /tabs` o `POST /tabs/open`; también se aceptan las etiquetas y los
identificadores `tabId`, como `t1`. Los identificadores de destino CDP sin procesar y los prefijos
únicos de identificadores de destino sin procesar siguen funcionando, pero son identificadores de diagnóstico volátiles.

Si se configura la autenticación del Gateway mediante secreto compartido, las rutas HTTP del navegador también requieren autenticación:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` o autenticación HTTP Basic con esa contraseña

Notas:

- Esta API independiente del navegador mediante bucle invertido **no** utiliza encabezados de identidad de proxy de confianza ni de
  Tailscale Serve.
- Si `gateway.auth.mode` es `none` o `trusted-proxy`, estas rutas del navegador mediante bucle invertido
  no heredan esos modos que proporcionan identidad; manténgalas restringidas al bucle invertido.

### Contrato de errores de `/act`

`POST /act` utiliza una respuesta de error estructurada para los fallos de validación y
de políticas en el nivel de ruta:

```json
{ "error": "<message>", "code": "ACT_*" }
```

Valores actuales de `code`:

- `ACT_KIND_REQUIRED` (HTTP 400): falta `kind` o no se reconoce.
- `ACT_INVALID_REQUEST` (HTTP 400): la carga útil de la acción no superó la normalización o la validación.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): se utilizó `selector` con un tipo de acción no compatible.
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (o `wait --fn`) está deshabilitado por la configuración.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): el `targetId` de nivel superior o por lotes entra en conflicto con el destino de la solicitud.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): la acción no es compatible con perfiles de sesión existente.

Otros fallos del entorno de ejecución aún pueden devolver `{ "error": "<message>" }` sin un
campo `code`.

### Requisito de Playwright

Algunas funciones (navegación/acción/instantánea de IA/instantánea por rol, capturas de elementos
y PDF) requieren Playwright. Si Playwright no está instalado, esos endpoints devuelven
un error 501 claro.

Qué sigue funcionando sin Playwright:

- Instantáneas de ARIA
- Instantáneas de accesibilidad basadas en roles (`--interactive`, `--compact`,
  `--depth`, `--efficient`) cuando hay un WebSocket CDP disponible por pestaña. Esta es
  una alternativa para la inspección y el descubrimiento de referencias; Playwright sigue siendo el motor
  principal de acciones.
- Capturas de página para el navegador administrado `openclaw` cuando hay un WebSocket
  CDP disponible por pestaña
- Capturas de página para perfiles `existing-session` / Chrome MCP
- Capturas de pantalla basadas en referencias `existing-session` (`--ref`) a partir de la salida de instantáneas

Qué sigue requiriendo Playwright:

- `navigate`
- `act`
- Instantáneas de IA que dependen del formato nativo de instantáneas de IA de Playwright
- Capturas de elementos mediante selectores CSS (`--element`)
- Exportación completa del navegador a PDF

Las capturas de elementos también rechazan `--full-page`; la ruta devuelve `fullPage is
not supported for element screenshots`.

Si aparece `Playwright is not available in this gateway build`, al Gateway empaquetado
le falta la dependencia principal del entorno de ejecución del navegador. Reinstale o actualice
OpenClaw y, a continuación, reinicie el Gateway. Para Docker, instale también los binarios
del navegador Chromium como se muestra a continuación.

#### Instalación de Playwright en Docker

Si el Gateway se ejecuta en Docker, evite `npx playwright` (conflictos de sustitución de npm).
Para imágenes personalizadas, incluya Chromium en la imagen:

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

Para una imagen existente, realice la instalación mediante la CLI incluida:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

Para conservar las descargas del navegador, establezca `PLAYWRIGHT_BROWSERS_PATH` (por ejemplo,
`/home/node/.cache/ms-playwright`) y asegúrese de conservar `/home/node` mediante
`OPENCLAW_HOME_VOLUME` o un montaje enlazado. OpenClaw detecta automáticamente el
Chromium conservado en Linux. Consulte [Docker](/es/install/docker).

## Funcionamiento interno

Un pequeño servidor de control de bucle invertido acepta solicitudes HTTP y se conecta a navegadores basados en Chromium mediante CDP. Las acciones avanzadas (hacer clic/escribir/instantánea/PDF) se procesan con Playwright sobre CDP; cuando falta Playwright, solo están disponibles las operaciones que no dependen de Playwright. El agente ve una única interfaz estable mientras los navegadores y perfiles locales o remotos se intercambian libremente de forma subyacente.

## Referencia rápida de la CLI

Todos los comandos aceptan `--browser-profile <name>` para seleccionar un perfil específico y `--json` para obtener una salida legible por máquinas.

<AccordionGroup>

<Accordion title="Conceptos básicos: estado, pestañas, abrir/enfocar/cerrar">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # añade una prueba de instantánea en vivo
openclaw browser start
openclaw browser start --headless # inicio puntual local administrado sin interfaz gráfica
openclaw browser stop            # también borra la emulación en CDP remoto/de solo conexión
openclaw browser reset-profile   # mueve los datos del navegador del perfil a la Papelera
openclaw browser tabs
openclaw browser tab             # acceso directo a la pestaña actual
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="Perfiles: enumerar, crear, eliminar">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="Inspección: captura de pantalla, instantánea, consola, errores, solicitudes">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # o --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="Acciones: navegar, hacer clic, escribir, arrastrar, esperar, evaluar">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # o e12 para referencias de rol
openclaw browser click-coords 120 340        # coordenadas de la ventana gráfica
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="Estado: cookies, almacenamiento, modo sin conexión, encabezados, ubicación, dispositivo">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # --clear para eliminar
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

Notas:

- La herramienta `browser` orientada al agente expone `action=download` (`ref` y
  `path` obligatorios) y `action=waitfordownload` (`path` opcional). Ambas devuelven la URL
  de descarga guardada, el nombre de archivo sugerido y la ruta local protegida. La interceptación
  explícita de descargas está disponible para los perfiles administrados de Playwright; los perfiles
  de sesión existente devuelven un error de operación no admitida.
- Se recomienda usar cargas atómicas mediante el selector de archivos: pase el desencadenador `--ref` junto con la carga para que OpenClaw se prepare y haga clic en una sola solicitud. `upload` solo con rutas sigue siendo compatible cuando se pretende usar un desencadenador posterior. Use `--input-ref` o `--element` para establecer directamente una entrada de archivo. `dialog` es una llamada de preparación; ejecútela antes del clic o la pulsación que desencadena el cuadro de diálogo. Si una acción abre un cuadro modal, la respuesta de la acción incluye `blockedByDialog` y `browserState.dialogs.pending`; pase ese `dialogId` para responder directamente. Los cuadros de diálogo gestionados fuera de OpenClaw aparecen en `browserState.dialogs.recent`.
- `click`/`type`/etc. requieren un `ref` de `snapshot` (`12` numérico, referencia de rol `e12` o referencia ARIA procesable `ax12`). Los selectores CSS no se admiten deliberadamente para las acciones. Use `click-coords` cuando la posición visible en el área de visualización sea el único objetivo fiable.
- Las rutas de descarga y seguimiento están restringidas a las raíces temporales de OpenClaw: `/tmp/openclaw{,/downloads}` (alternativa: `${os.tmpdir()}/openclaw/...`).
- `upload` acepta archivos de la raíz temporal de cargas de OpenClaw y
  contenido multimedia entrante administrado por OpenClaw. Se puede hacer referencia al contenido multimedia entrante administrado mediante
  `media://inbound/<id>`, `media/inbound/<id>` relativo al entorno aislado o una
  ruta resuelta dentro del directorio de contenido multimedia entrante administrado. Se siguen rechazando las referencias
  anidadas a contenido multimedia, el recorrido de directorios, los enlaces simbólicos, los enlaces físicos y las rutas locales arbitrarias.
- `upload` también puede establecer directamente entradas de archivo mediante `--input-ref` o `--element`.

Los identificadores y las etiquetas estables de las pestañas sobreviven al reemplazo de objetivos sin procesar de Chromium cuando OpenClaw
puede demostrar cuál es la pestaña de reemplazo, como un único par anterior/nuevo para la misma URL o
una sola pestaña anterior que se convierte en una sola pestaña nueva después de enviar un formulario. Los reemplazos
ambiguos con URL duplicadas reciben nuevos identificadores. Los identificadores de objetivos sin procesar siguen siendo
volátiles; se recomienda usar `suggestedTargetId` de `tabs` en los scripts.

Resumen de las opciones de instantánea:

- `--format ai` (predeterminado con Playwright): instantánea de IA con referencias numéricas (`aria-ref="<n>"`).
- `--format aria`: árbol de accesibilidad con referencias `axN`. Cuando Playwright está disponible, OpenClaw vincula las referencias mediante identificadores DOM del backend a la página activa para que las acciones posteriores puedan utilizarlas; de lo contrario, la salida debe considerarse solo para inspección.
- `--efficient` (o `--mode efficient`): configuración preestablecida de instantánea compacta de roles. Establezca `browser.snapshotDefaults.mode: "efficient"` para convertirla en la opción predeterminada (consulte [Configuración del Gateway](/es/gateway/configuration-reference#browser)).
- `--interactive`, `--compact`, `--depth` y `--selector` fuerzan una instantánea de roles con referencias `ref=e12`. `--frame "<iframe>"` limita las instantáneas de roles a un iframe.
- Con Playwright, `--labels` añade una captura de pantalla con etiquetas de referencia superpuestas
  (muestra `MEDIA:<path>`) y un arreglo `annotations` con el cuadro delimitador
  de cada referencia. En `screenshot`, las etiquetas respaldadas por Playwright funcionan con `--full-page`,
  `--ref` y `--element`; en `snapshot`, la captura de pantalla adjunta sigue
  limitada al área de visualización. Los perfiles de sesión existente/chrome-mcp representan etiquetas superpuestas en
  las capturas de pantalla de la página, pero no devuelven `annotations` ni usan el asistente de proyección
  de página completa/referencia/elemento de Playwright. Sin Playwright ni chrome-mcp,
  las capturas de pantalla con etiquetas no están disponibles.
- `--urls` añade los destinos de enlaces detectados a las instantáneas de IA.

## Instantáneas y referencias

OpenClaw admite dos estilos de "instantánea":

- **Instantánea de IA (referencias numéricas)**: `openclaw browser snapshot` (predeterminado; `--format ai`)
  - Salida: una instantánea de texto que incluye referencias numéricas.
  - Acciones: `openclaw browser click 12`, `openclaw browser type 23 "hello"`.
  - Internamente, la referencia se resuelve mediante `aria-ref` de Playwright.

- **Instantánea de roles (referencias de rol como `e12`)**: `openclaw browser snapshot --interactive` (o `--compact`, `--depth`, `--selector`, `--frame`)
  - Salida: una lista o árbol basado en roles con `[ref=e12]` (y `[nth=1]` opcional).
  - Acciones: `openclaw browser click e12`, `openclaw browser highlight e12`.
  - Internamente, la referencia se resuelve mediante `getByRole(...)` (más `nth()` para duplicados).
  - Añada `--labels` para incluir una captura de pantalla con etiquetas `e12` superpuestas. En
    los perfiles respaldados por Playwright, esto también devuelve metadatos del cuadro delimitador de cada referencia
    (`annotations[]`).
  - Añada `--urls` cuando el texto del enlace sea ambiguo y el agente necesite objetivos
    de navegación concretos.

- **Instantánea ARIA (referencias ARIA como `ax12`)**: `openclaw browser snapshot --format aria`
  - Salida: el árbol de accesibilidad como nodos estructurados.
  - Acciones: `openclaw browser click ax12` funciona cuando la ruta de la instantánea puede vincular
    la referencia mediante Playwright y los identificadores DOM del backend de Chrome.
- Si Playwright no está disponible, las instantáneas ARIA aún pueden ser útiles para
  la inspección, pero es posible que las referencias no sean procesables. Vuelva a generar la instantánea con `--format ai`
  o `--interactive` cuando necesite referencias para acciones.
- Prueba con Docker para la ruta alternativa de CDP sin procesar: `pnpm test:docker:browser-cdp-snapshot`
  inicia Chromium con CDP, ejecuta `browser doctor --deep` y verifica que las instantáneas de roles
  incluyan las URL de los enlaces, los elementos en los que se puede hacer clic detectados mediante el cursor y los metadatos de iframe.

Comportamiento de las referencias:

- Las referencias **no son estables entre navegaciones**; si algo falla, vuelva a ejecutar `snapshot` y use una referencia nueva.
- `/act` devuelve el `targetId` sin procesar actual después de un reemplazo provocado por una acción
  cuando puede demostrar cuál es la pestaña de reemplazo. Siga usando identificadores y etiquetas estables de pestañas para
  los comandos posteriores.
- Si la instantánea de roles se tomó con `--frame`, las referencias de rol quedan limitadas a ese iframe hasta la siguiente instantánea de roles.
- Las referencias `axN` desconocidas u obsoletas fallan inmediatamente en lugar de recurrir al
  selector `aria-ref` de Playwright. Cuando esto ocurra, genere una instantánea nueva en la misma pestaña.

## CLI de lotes del navegador

`openclaw browser batch` ejecuta un arreglo de acciones `/act` anidadas en una sola llamada a `/act`
(el mismo entorno de ejecución `kind="batch"` al que se accede mediante la herramienta del agente), de modo que los usuarios
de la CLI y los scripts puedan combinar acciones como `wait`, `click`, `type` y
`evaluate` en un único plan reproducible sin recorridos de ida y vuelta por acción. Cada
entrada de `actions[]` es un `BrowserActRequest`: la unión cerrada que acepta la ruta `/act`
(`click`, `clickCoords`, `type`, `press`, `hover`,
`scrollIntoView`, `drag`, `select`, `fill`, `resize`, `wait`, `evaluate`,
`close`, `batch`), no subcomandos `openclaw browser` arbitrarios. `batch`
no es compatible con `profile="user"` ni con otros perfiles de sesión existente (chrome-mcp);
envíe allí las acciones individualmente.

- CLI: `openclaw browser batch --actions '<json>'`, `openclaw browser batch
--actions-file plan.json` o `openclaw browser batch --actions-file -` para
  leer el arreglo JSON desde la entrada estándar. `--continue` establece `stopOnError=false`; de forma
  predeterminada, la ejecución se detiene ante el primer error. `--target-id` limita todo el lote a
  una pestaña.
- Ciclo de vida de las referencias: las referencias proceden de una ejecución de `snapshot` anterior al lote (la instantánea
  no es una acción anidada). Una acción anidada que cambie el estado de la página, como un
  `click` que desencadene una navegación o un `evaluate` que modifique el DOM, puede
  invalidar las referencias anteriores durante el resto del lote. Coloque primero las acciones que cambian el estado
  o divida el proceso en un lote posterior tras volver a generar la instantánea. La navegación y
  la nueva generación de instantáneas se realizan fuera del lote (`openclaw browser navigate` /
  `snapshot`), ya que `open`, `navigate` y `snapshot` no son tipos `/act`.
- Conflictos de identificadores de destino: una acción anidada puede omitir `targetId` o repetir el
  `targetId` del nivel de solicitud; un `targetId` anidado explícito que se resuelva en una
  pestaña diferente se rechaza con `ACT_TARGET_ID_MISMATCH` antes de ejecutar cualquier acción.
  Por diseño, las acciones por lotes comparten la pestaña de la solicitud.
- Resumen de errores: la respuesta es `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }`, con una entrada por acción y en orden. Cuando
  `stopOnError` es el valor predeterminado, el arreglo termina en el primer fallo; con
  `--continue`, abarca todas las acciones. Cualquier entrada fallida hace que la CLI termine
  con un código distinto de cero; pase `--json` para conservar la respuesta completa y ordenada para los scripts.

## Capacidades avanzadas de espera

Se puede esperar algo más que tiempo o texto:

- Esperar una URL (Playwright admite patrones glob):
  - `openclaw browser wait --url "**/dash"`
- Esperar un estado de carga:
  - `openclaw browser wait --load networkidle`
  - Compatible con perfiles `openclaw` administrados y perfiles CDP sin procesar/remotos. Los perfiles que usan el controlador `existing-session` (incluido el perfil `user` predeterminado) rechazan `networkidle`; use allí esperas `--url`, `--text`, un selector o `--fn`.
- Esperar un predicado de JS:
  - `openclaw browser wait --fn "window.ready===true"`
- Esperar a que un selector sea visible:
  - `openclaw browser wait "#main"`

Estas opciones se pueden combinar:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## Flujos de trabajo de depuración

Cuando una acción falla (por ejemplo, "no visible", "infracción del modo estricto", "cubierto"):

1. `openclaw browser snapshot --interactive`
2. Use `click <ref>` / `type <ref>` (se recomiendan las referencias de rol en el modo interactivo)
3. Si sigue fallando: `openclaw browser highlight <ref>` para ver a qué apunta Playwright
4. Si la página se comporta de manera extraña:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. Para una depuración en profundidad, grabe un seguimiento:
   - `openclaw browser trace start`
   - reproduzca el problema
   - `openclaw browser trace stop` (muestra `TRACE:<path>`)

## Salida JSON

`--json` está destinado a scripts y herramientas estructuradas.

Ejemplos:

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

Las instantáneas de roles en JSON incluyen `refs` y un pequeño bloque `stats` (líneas/caracteres/referencias/interactivo) para que las herramientas puedan analizar el tamaño y la densidad de la carga útil.

## Opciones de estado y entorno

Son útiles para flujos de trabajo del tipo "hacer que el sitio se comporte como X":

- Cookies: `cookies`, `cookies set`, `cookies clear`
- Almacenamiento: `storage local|session get|set|clear`
- Sin conexión: `set offline on|off`
- Encabezados: `set headers --headers-json '{"X-Debug":"1"}'` (o la forma posicional `set headers '{"X-Debug":"1"}'`)
- Autenticación básica HTTP: `set credentials user pass` (o `--clear`)
- Geolocalización: `set geo <lat> <lon> --origin "https://example.com"` (o `--clear`)
- Medios: `set media dark|light|no-preference|none`
- Zona horaria / configuración regional: `set timezone ...`, `set locale ...`
- Dispositivo / área de visualización:
  - `set device "iPhone 14"` (configuraciones preestablecidas de dispositivos de Playwright)
  - `set viewport 1280 720`

## Seguridad y privacidad

- El perfil del navegador de OpenClaw puede contener sesiones iniciadas; trátelo como información confidencial.
- `browser act kind=evaluate` / `openclaw browser evaluate` y `wait --fn`
  ejecutan JavaScript arbitrario en el contexto de la página. La inyección de
  prompts puede manipular este comportamiento. Desactívelo con `browser.evaluateEnabled=false` si no lo necesita.
- `openclaw browser evaluate --fn` acepta el código fuente de una función, una expresión o
  el cuerpo de una instrucción. Los cuerpos de instrucciones se encapsulan como funciones asíncronas, así que use
  `return` para el valor que desea obtener. Use `--timeout-ms <ms>` cuando la
  función del lado de la página pueda necesitar más tiempo que el tiempo de espera predeterminado de evaluación.
- Para obtener información sobre inicios de sesión y medidas contra bots (X/Twitter, etc.), consulte [Inicio de sesión en el navegador y publicación en X/Twitter](/es/tools/browser-login).
- Mantenga privado el host del Gateway/Node (solo loopback o tailnet).
- Los endpoints CDP remotos son potentes; canalícelos mediante un túnel y protéjalos.

Ejemplo de modo estricto (bloquea de forma predeterminada los destinos privados o internos):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // permiso exacto opcional
    },
  },
}
```

## Contenido relacionado

- [Navegador](/es/tools/browser) - descripción general, configuración, perfiles y seguridad
- [Inicio de sesión en el navegador](/es/tools/browser-login) - inicio de sesión en sitios
- [Solución de problemas del navegador en Linux](/es/tools/browser-linux-troubleshooting)
- [Solución de problemas del navegador en WSL2](/es/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
