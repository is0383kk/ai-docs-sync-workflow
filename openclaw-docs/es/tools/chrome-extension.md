---
read_when:
    - Quieres que un agente controle desde tu teléfono tu sesión real iniciada en Chrome
    - Te sigue apareciendo el aviso de Chrome «Allow remote debugging?» cuando no hay nadie frente al equipo
    - Quiere comprender el modelo de seguridad de la toma de control del navegador mediante la extensión
summary: 'Extensión de Chrome: permite que OpenClaw controle tu sesión iniciada de Chrome sin mostrar el aviso de depuración remota'
title: Extensión de Chrome
x-i18n:
    generated_at: "2026-07-26T05:31:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d974f62bb5697a23dd6a6852137ce6af5a8a4a2a8ff738eec0098f259e8faa0
    source_path: tools/chrome-extension.md
    workflow: 16
---

# Extensión de Chrome

La extensión de Chrome de OpenClaw permite que un agente controle las **pestañas
de Chrome con sesión iniciada** sin abrir un navegador administrado independiente
y **sin** el aviso bloqueante de Chrome «Allow remote debugging?».

Esto es importante cuando se controla OpenClaw desde un teléfono (Telegram,
WhatsApp, etc.): el [perfil `user`](/es/tools/browser#profiles-openclaw-user-chrome)
se conecta mediante el puerto de depuración remota de Chrome, lo que muestra un
cuadro de diálogo de consentimiento en el escritorio en el que nadie puede hacer
clic cuando no se está presente. En su lugar, la extensión utiliza la API
`chrome.debugger`, por lo que la única indicación dentro de la página es el
aviso descartable de Chrome «OpenClaw started debugging this browser».

Es el mismo enfoque que utilizan las extensiones de Chrome de Claude de Anthropic
y Codex de OpenAI.

## Cómo funciona

Consta de tres partes:

- **Servicio de control del navegador** (Gateway o host del nodo): la API a la que llama la
  herramienta `browser`.
- **Relé de la extensión** (WebSocket de bucle invertido): un pequeño servidor que el servicio de
  control inicia en `127.0.0.1`. Presenta un endpoint del protocolo Chrome
  DevTools a OpenClaw y se comunica con la extensión. Ambos extremos se autentican
  con un token local del host (véase más adelante).
- **Extensión de Chrome de OpenClaw** (MV3): se conecta a las pestañas con
  `chrome.debugger`, reenvía el tráfico CDP y administra el **grupo de pestañas de
  OpenClaw**.

OpenClaw solo ve y controla las pestañas que están en el **grupo de pestañas de
OpenClaw**. El grupo es el límite del consentimiento: arrastre una pestaña al
grupo para compartirla y sáquela de él (o haga clic en el botón de la barra de
herramientas) para revocar el acceso al instante.

## Instalación y vinculación

1. Muestre la ruta de la extensión sin empaquetar:

   ```bash
   openclaw browser extension path
   ```

2. Abra `chrome://extensions`, active **Developer mode**, haga clic en **Load
   unpacked** y seleccione el directorio mostrado.

3. Muestre la cadena de vinculación:

   ```bash
   openclaw browser extension pair
   ```

4. Haga clic en el icono de OpenClaw de la barra de herramientas y pegue la cadena de
   vinculación en la ventana emergente. La insignia cambia a **ON** cuando la
   extensión se conecta al relé.

El token de vinculación es un **secreto local del host** creado durante el primer
uso y almacenado en `credentials/` dentro del directorio de estado (modo
`0600`). Cada máquina que ejecuta un navegador —el host del Gateway y
cada host de nodo de navegador— posee su propio token, por lo que ninguna
credencial tiene que desplazarse entre máquinas. Para rotarlo, elimine el archivo
`browser-extension-relay.secret` y vuelva a realizar la vinculación.

## Uso

Seleccione el perfil integrado `chrome` en una llamada a la herramienta
`browser` o establézcalo como predeterminado:

```bash
openclaw config set browser.defaultProfile chrome
```

```json5
{
  browser: {
    profiles: {
      chrome: { driver: "extension", color: "#FF4500" },
    },
  },
}
```

- Comparta una pestaña: haga clic en el botón de OpenClaw de la barra de herramientas
  de esa pestaña (se unirá al grupo de pestañas de OpenClaw) o arrastre cualquier
  pestaña al grupo.
- El agente también puede abrir nuevas pestañas, que se incorporan automáticamente al
  grupo.
- Revoque el acceso: vuelva a hacer clic en el botón, saque la pestaña del grupo o
  descarte el aviso de depuración de Chrome. El agente pierde inmediatamente el
  acceso a esa pestaña.

### Panel lateral del copiloto de pestaña

Después de vincular la extensión, haga clic en **Open tab copilot** en la ventana
emergente de su barra de herramientas. OpenClaw configura `sidepanel.html` para
esa pestaña concreta de Chrome; el manifiesto no tiene una ruta global del panel
lateral. Por tanto, cada pestaña obtiene un documento de panel, una sesión del
Gateway, una suscripción de mensajes y un enlace tipado de la herramienta del
navegador independientes.

El panel no incluye la URL, el título, el DOM ni el texto visible de la página en
el mensaje. Solo envía el texto que se escribe. Las acciones del navegador
incorporan un enlace independiente autenticado por el Gateway que contiene la
pestaña de Chrome y el destino CDP, y la herramienta del navegador rechaza los
intentos de sustituir ese destino o utilizar acciones que abarquen todo el
navegador. Las respuestas permanecen en el panel (`deliver: false`); no heredan
una ruta de Telegram, Discord ni de ningún otro canal.

El copiloto es un dispositivo dedicado vinculado al Gateway con los ámbitos
`operator.read` y `operator.write`. Durante el primer uso, inspeccione y
apruebe su solicitud:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

La extensión conserva esa identidad del dispositivo y el token de dispositivo
emitido por el Gateway, restringidos al endpoint canónico del Gateway que los
emitió. La vinculación con un Gateway diferente crea una identidad, un token y
una custodia de sesión independientes; las credenciales y las sesiones nunca se
reutilizan entre endpoints. La extensión no conserva el secreto compartido del
Gateway. Un panel solo puede suscribirse a sus propias sesiones de pestaña, y el
Gateway filtra esos eventos antes de entregarlos.

Si la conexión con el Gateway se interrumpe durante una ejecución, la extensión
mantiene la custodia persistente del ID de esa ejecución. Al volver a conectarse,
cancela la ejecución sin resolver antes de volver a habilitar cualquier panel y,
a continuación, recarga el historial de la transcripción. Este paso de cierre
seguro evita que las acciones del navegador continúen sin ser visibles durante
una interrupción de la entrega.

Al cerrar una pestaña se elimina inmediatamente su suscripción activa, se cancela
cualquier ejecución visible y se marca como archivada la sesión de esa pestaña.
Si el Gateway está temporalmente sin conexión, la extensión conserva el archivado
pendiente y solo lo vuelve a intentar cuando se reconecta ese mismo endpoint del
Gateway; nunca envía una solicitud de archivado a un Gateway diferente. Después
de un fallo del navegador, el siguiente inicio archiva las sesiones dejadas por
la instancia anterior del navegador. Las sesiones archivadas rechazan nuevos
trabajos, mientras que sus transcripciones siguen disponibles en el historial de
sesiones. Las claves del copiloto del navegador son sesiones de hilo, por lo que
el mantenimiento normal por antigüedad y número de entradas las conserva. Se
sigue aplicando el presupuesto de disco de sesiones por agente (valor
predeterminado: `2gb`) y, bajo presión, puede desalojar las sesiones
más antiguas; consulte [mantenimiento de sesiones](/es/reference/session-management-compaction#store-maintenance-and-disk-controls).

Actualmente, el panel lateral requiere un relé de extensión alojado en el Gateway
o un relé remoto directo del Gateway. Un relé de bucle invertido en un nodo de
navegador todavía no puede proporcionar la ruta del nodo que exige el enlace
tipado de la pestaña, por lo que el panel rechaza esa topología en lugar de
recurrir al enrutamiento de todo el navegador.

## Enviar una página a OpenClaw

Utilice **Send page to OpenClaw** en la ventana emergente de la barra de
herramientas para compartir el texto legible de la página con la sesión principal
de OpenClaw. Se puede añadir una nota opcional, utilizar el menú contextual de la
página o de la selección, o pulsar `Alt+Shift+S`. OpenClaw da prioridad a la
selección actual cuando existe, pone el contenido compartido en cola como evento
del sistema y activa inmediatamente la sesión principal.

No es necesario que la pestaña esté en el grupo de pestañas de OpenClaw. Se trata
de un contenido compartido explícito y de una sola vez: no se expone ningún otro
elemento de la página ni se concede acceso continuo. Google Docs se exporta como
texto sin formato mediante la sesión iniciada del navegador, sin configurar la
API de Google. Los hilos de X y Twitter se extraen sin la interfaz circundante.

El texto de la página queda envuelto en el límite de seguridad para contenido
externo de OpenClaw. La nota opcional permanece fuera de ese límite como
instrucción propia. El texto de las páginas y las selecciones tienen un límite de
aproximadamente 120,000 caracteres e incluyen un marcador de truncamiento cuando
se acortan.

El uso compartido de páginas funciona cuando el relé de la extensión está alojado
en el Gateway, mediante la vinculación en el mismo host o la vinculación directa
con el Gateway `wss://`. Por ahora, los relés alojados en nodos
devuelven un error claro. Para reasignar el método abreviado de teclado, abra
`chrome://extensions/shortcuts`.

## Uso remoto o entre máquinas

Chrome no tiene que ejecutarse en el host del Gateway. Se admiten tres topologías:

- **Mismo host** (Gateway y Chrome en una máquina): realice la vinculación en esa
  máquina con `openclaw browser extension pair`. El relé solo utiliza el bucle invertido.
  Si el Gateway local utiliza TLS, indique explícitamente el nombre de host de su
  certificado mediante `--gateway-url wss://gateway-host.example`; la vinculación nunca lo sustituye por
  una IP de bucle invertido.
- **Conexión directa a un Gateway remoto** (Chrome en el portátil, el Gateway en un
  VPS y **nada más en el portátil**): ejecute `openclaw browser extension pair --gateway-url wss://your-gateway.example.com` en el Gateway.
  Se muestra una cadena `wss://…/browser/extension#<secret>`; cargue y vincule la extensión en el
  portátil. La extensión se conecta **directamente al Gateway** mediante
  `wss://`: no se requiere instalar OpenClaw, Node ni la CLI, ni abrir
  ningún puerto de entrada en el portátil. Esta es la opción para alojamiento
  administrado.
- **Mediante un host de nodo de navegador** (Chrome en una máquina que ya ejecuta un
  nodo de OpenClaw): ejecute `pair` en el nodo y realice la
  vinculación localmente; el Gateway redirige las acciones del navegador al nodo
  mediante su enlace de nodo autenticado existente.

El secreto de vinculación es específico de cada host (el del Gateway en el caso
directo) y se valida mediante la ruta `/browser/extension` del Gateway. Para la ruta
directa, sirva el Gateway mediante TLS (`wss://`) para cifrar el secreto
de vinculación y el tráfico CDP. El secreto permanece en el fragmento de URL de
la cadena de vinculación y se presenta durante el protocolo de enlace WebSocket
como credencial de subprotocolo, por lo que los registros normales de acceso del
proxy no lo reciben en la URL de la solicitud. Asegúrese de que cualquier proxy
inverso conserve el encabezado estándar `Sec-WebSocket-Protocol`.

## Diagnóstico

```bash
openclaw browser status --browser-profile chrome
openclaw browser doctor --browser-profile chrome
```

`doctor` indica que la comprobación del **relé de la extensión de Chrome**
falla hasta que la ventana emergente de la extensión muestra **Connected**.

## Modelo de seguridad

- El relé solo se enlaza al bucle invertido; ambos extremos de WebSocket se
  autentican con el token derivado y se comprueba que el origen de la extensión
  sea `chrome-extension://`.
- La vinculación directa con el Gateway no acepta el token del relé en la URL de
  la solicitud; en su lugar, la extensión incluida lo incorpora en la lista de
  subprotocolos de WebSocket.
- El agente solo puede ver y controlar las pestañas del **grupo de pestañas de
  OpenClaw**. Las demás pestañas permanecen privadas.
- Las ejecuciones del panel lateral tienen una doble restricción: la entrega del
  Gateway utiliza una lista de permitidos por sesión y las herramientas del
  navegador hacen cumplir el enlace con la pestaña o el destino de Chrome que se
  transmite fuera del prompt.
- En comparación con el perfil `user` (Chrome MCP), que expone todo el
  navegador con sesión iniciada una vez aprobado el aviso de depuración remota,
  la extensión mantiene la superficie compartida restringida a un grupo de
  pestañas que se puede controlar de un vistazo.

Consulte también [Navegador](/es/tools/browser) para conocer el modelo completo de
perfiles y los perfiles administrados `openclaw` y `user` de
Chrome MCP.
